# 3.6. `process::`：子进程

`std::process` 模块启动子进程并处理信号。它的基本骨架沿用 Rust 的 `Command` 构建器模式，并在此之上加入现代的防护措施：`pidfd`、`kill_on_drop` 和 `posix_spawn` 快速路径。

## 3.6.1. `Command` 构建器

首先遇到的是 `Command` 结构体，它是子进程的蓝图。所有设置（要运行的程序、参数、环境变量、标准 I/O 的连接方式）都在启动之前逐步构建。

路径参数遵循整个层级共用的[参数约定](README.md#参数约定)（`Path` 即 `P: AsPath`）。

```nabi
#[must_use]
struct Command { /* opaque: program, args, environment, streams, cwd, options */ }

impl Command {
    fn new(program: Path) -> Command;

    fn arg(self, arg: str) -> Command;
    fn args<I: IntoIterator<Item = str>>(self, args: I) -> Command;
    fn env(self, key: str, value: str) -> Command;
    fn envs<I: IntoIterator<Item = (str, str)>>(self, envs: I) -> Command;
    fn env_clear(self) -> Command;
    fn current_dir(self, dir: Path) -> Command;

    fn stdin(self, cfg: Stdio) -> Command;
    fn stdout(self, cfg: Stdio) -> Command;
    fn stderr(self, cfg: Stdio) -> Command;

    fn kill_on_drop(self, on: bool) -> Command;         // kill when the spawned Child is released (Drop)
    // unix: process_group, CommandExt (3.10.5)

    // execution
    fn spawn(self) throws (IoError) -> Child;
    fn output(self) throws (IoError) -> Output;          // wait + collect: stdout / stderr left at Inherit are piped; an Inherit stdin is null, a Piped one is closed at once (the child sees end of input)
    fn status(self) throws (IoError) -> ExitStatus;      // wait, no output
    async fn output_async(self) throws (IoError) -> Output;
    async fn status_async(self) throws (IoError) -> ExitStatus;
}

#[non_exhaustive]
enum Stdio {
    Inherit,                   // parent's fd as is (default)
    Null,                      // /dev/null
    Piped,                     // pipe, readable/writable from Child
    File(File),                // this file; the command owns it from here
}

struct Output {
    status: ExitStatus;
    stdout: List<u8>;
    stderr: List<u8>;
}                                        // + Clone, PartialEq, Eq, Debug
```

构建器方法返回修改后的命令，调用它的那个命令保持不变，所以 `Command` 的各个副本相互独立：执行 `let a = base.arg("x")` 之后，`base` 中没有 `x`。`Command` 不是仿射的，尽管其中的 `Stdio::File` 是仿射的（[2.13.4](../200-core/213-marker.md)）：这一个文件是例外，由所有副本共享（见下文）。结果被丢弃的构建器调用（`base.arg("x");`）不会改变任何东西，并会产生警告（`#[must_use]`）。

`Stdio` 有四个分支：`Inherit` 传递父进程的 fd；`Null` 完全丢弃输出；`Piped` 用于双向通信；`File` 接入一个已打开的文件。

移入 `Stdio::File` 的 `File` 属于命令，而不属于程序：与 `File` 的任何移动一样，这次移动会消耗它（[2.13.4](../200-core/213-marker.md)），程序不再关闭它。运行命令（`spawn`、`output`、`status` 及其 `_async` 孪生版本）会在子进程启动后立即关闭父进程的副本，启动失败时也会关闭；子进程保留它自己的副本。此处关闭失败即为这次运行的 `IoError`：已经启动的子进程会先被杀死并回收，所以不会丢失任何 `Child`。如果某个命令的文件已被之前的一次运行关闭，该命令的副本会以 `InvalidInput` 失败，而不会运行；请为每次运行提供单独的文件（`File::try_clone`）。从未运行的命令会让它的文件一直打开到程序退出：垃圾回收器释放内存，但不关闭任何东西。

持有文件的 `Stdio` 也是仿射的：接收它的构建器调用会把它移入命令。把同一个 `Stdio::File` 值交给两个命令，或交给同一个命令的两个流（`.stdout(s).stderr(s)`），是编译错误 `E0133`；请为每个流提供单独的文件（`Stdio::File(try f.try_clone())`）。`Stdio::Inherit`、`Null` 和 `Piped` 不持有任何东西，可以交出任意多次。`Stdio` 实现了 `Close`：从未交给命令的 `Stdio::File` 由程序关闭（`try s.close()`），与它的 `File` 一样。

### `run`

```nabi
pub fn run<P: AsPath>(program: P, args: List<str> = []) throws (IoError | ExitStatusError | Utf8Error) -> str;
```

基于 `Command` 的便捷形式：像 `Command::output` 一样以 `args` 运行 `program`，并以文本形式返回它的 stdout。非成功状态抛出 `ExitStatusError`（`status()`、以文本形式返回并替换了无效 UTF-8 的 `stderr()`、`message()`）；不是 UTF-8 的 stdout 抛出 `Utf8Error`。`let out = try process::run("git", ["log", "-1"]);`（相当于 Python 的 `subprocess.run(.., check=True, capture_output=True, text=True).stdout`）。

## 3.6.2. `Child`

`spawn` 成功时返回 `Child`，即正在运行的进程的句柄。通过它可以访问标准 I/O 流、等待退出或发送信号。

```nabi
struct Child { /* opaque: pid, the piped streams not yet taken */ }

impl Child {
    fn id(self) -> u32;                                  // pid
    fn take_stdin(ref self) -> ChildStdin | none;        // none: not Piped, or taken already
    fn take_stdout(ref self) -> ChildStdout | none;
    fn take_stderr(ref self) -> ChildStderr | none;
    fn kill(self) throws (IoError);
    fn wait(ref self) throws (IoError) -> ExitStatus;
    async fn wait_async(ref self) throws (IoError) -> ExitStatus;
    fn try_wait(self) throws (IoError) -> ExitStatus | none;

    // Linux: pidfd, ChildExt (3.10.5)

    fn send_signal(self, sig: Signal) throws (IoError);  // Windows: Kill only
    fn handle(self) -> ChildHandle;                      // signals the child from another thread or task
}

struct ChildHandle { /* opaque: pid */ }                 // Send + Sync; copies are free

impl ChildHandle {
    fn id(self) -> u32;                                  // pid
    fn kill(self) throws (IoError);
    fn send_signal(self, sig: Signal) throws (IoError);  // Windows: Kill only
}

impl ChildStdin {                        // write end of the child's stdin pipe; impl io::Write + io::Close, io::AsyncWrite + io::AsyncClose
    fn close(self) throws (IoError);     // the child reads end of input
}
impl ChildStdout {                       // read end of the child's stdout pipe; impl io::Read + io::Close, io::AsyncRead + io::AsyncClose
    fn close(self) throws (IoError);
}
impl ChildStderr {                       // read end of the child's stderr pipe; impl io::Read + io::Close, io::AsyncRead + io::AsyncClose
    fn close(self) throws (IoError);
}
// + AsRawFd, IntoRawFd (unix), AsRawHandle, IntoRawHandle (Windows) for the three pipes: 3.10

struct ExitStatus { /* opaque */ }

impl ExitStatus {
    fn success(self) -> bool;
    fn code(self) -> i32 | none;        // none = signal terminated (unix)
    // unix: signal, ExitStatusExt (3.10.5)
}
impl Display for ExitStatus { ... }      // "exit status: 1" / "signal: 9"
impl Debug for ExitStatus { ... }        // "ExitStatus { code: 1 }" / "ExitStatus { signal: 9 }"
// ExitStatus: + Clone, Copy, PartialEq, Eq, Hash
```

退出结果保存在不透明类型 `ExitStatus` 中。如果 `code()` 为 `none`，说明进程在 unix 上被信号终止，`signal()`（`ExitStatusExt`，在 unix 上位于 prelude 中）会以 `Signal` 指出是哪个信号。

## 3.6.3. 示例

```nabi
use std::process::{Command, Stdio};

// simple execution
let status = try Command::new("ls").arg("-la").status();
if !status.success() { panic("ls failed") };

// collecting output
let output = try Command::new("git")
    .args(["log", "--oneline", "-10"])
    .output();
let log = try str::from_utf8(output.stdout);

// I/O through pipes
let child = try Command::new("grep")
    .arg("error")
    .stdin(Stdio::Piped)
    .stdout(Stdio::Piped)
    .spawn();

let stdin = child.take_stdin() ?? panic("stdin not piped");
try stdin.write_all(input_bytes);
try stdin.close();                                       // end of input: grep can finish
let stdout = child.take_stdout() ?? panic("stdout not piped");
let result = try stdout.read_to_end();
try stdout.close();
try child.wait();

// output into a file: the command takes the file and closes our copy
let log = try File::create("ls.txt");
try Command::new("ls").stdout(Stdio::File(log)).status();

// kill_on_drop prevents process leaks
using child = try Command::new("long_running")
        .kill_on_drop(true)
        .spawn() {
    // child is guaranteed to be killed even on error propagation / early return
}    // ← on using release, Drop → kill
```

如最后一个示例所示，把启用了 `kill_on_drop` 的 `Child` 绑定在 `using` 中，即使在错误传播或提前返回这样的流程中也能清理子进程。`Drop` 在所有者释放 `Child` 的地方发出 kill：在 `using` 块结束处，或在其所有者离开作用域的地方（[2.13.4](../200-core/213-marker.md)）。`ChildStdin` / `ChildStdout` / `ChildStderr` 是管道的文件句柄，读写方式与 `File` 相同。`Child` 持有它的 `Piped` 流，直到 `take_stdin` / `take_stdout` / `take_stderr` 交出其中一个并留下 `none`；此后取走者拥有该管道，并且必须像 `File` 一样关闭它。从未被取走的管道在 `Child` 被释放的地方关闭。`wait` 会先关闭仍由 `Child` 持有的 stdin，使读取到输入结束的子进程能够结束；已被取走的 stdin 由取走者在 `wait` 之前关闭。

`Child` 是仿射的（[2.13.4](../200-core/213-marker.md)）：由一个所有者持有它的管道并等待它。要在所有者等待期间从别处停止子进程（超时、关闭信号），用 `handle()` 取得 `ChildHandle`，把它发送给另一个线程或任务。句柄不拥有任何东西：没有管道，不能等待，也没有需要关闭的东西；它只能发送信号。`wait` / `try_wait` 只有在经由句柄发出、正在进行中的 `send_signal` / `kill` 调用返回之后才回收子进程，所以信号绝不会到达已被重用的 pid 或 pidfd；子进程已经结束并正在被回收时，`send_signal` 或 `kill` 不做任何事。

```nabi
let child = try Command::new("server").spawn();
let h = child.handle();
let timer = task::spawn(async {
    await task::sleep(Duration::secs(30));
    try h.kill() catch _ {};
    0
});
let status = try await child.wait_async();
```

## 3.6.4. 现代化补强：pidfd / posix_spawn / CLOEXEC

在构建器模式之上，标准库还加入了几项现代化补强。各项的含义和效果如下：

| 特性                    | 含义                          | 效果                    |
|-------------------------|-------------------------------|-------------------------|
| `posix_spawn` 快速路径  | 用一个系统调用代替 fork+exec  | 从巨大的进程中更快地启动 |
| `pidfd`（Linux 5.3+）   | 用 fd 而不是 pid 来跟踪       | 无竞争的信号            |
| 默认 `CLOEXEC`          | 每个 fd 自动设置 close-on-exec | 防止 fd 泄漏            |
| `kill_on_drop`          | 在所有者释放它的地方杀死进程  | 防止进程泄漏            |

这些决定始终着眼于减少 fd 泄漏、进程泄漏和信号竞争。

使用 `Command::env` 时，程序仍然在父进程的 `PATH` 上查找，而不是在子进程的 `PATH` 上。

## 3.6.5. 信号处理（进阶）

再深入一层，是信号处理函数。

```nabi
#[non_exhaustive]
enum Signal {
    Hangup, Interrupt, Quit, IllegalInstruction, Trap, Abort, Bus, FloatingPoint, Kill,
    Segmentation, Pipe, Alarm, Terminate, User1, User2, Child, Continue, Stop,
    TerminalStop, TerminalInput, TerminalOutput, Urgent, CpuLimit, FileSizeLimit,
    VirtualAlarm, Profile, WindowChange, Io, BadSyscall,
}                                        // SIGHUP .. SIGSYS

impl Signal {
    fn number(self) -> i32;              // the OS's number (SIGHUP = 1, ..; Linux's, else the BSD one)
}                                        // + Clone, Copy, PartialEq, Eq, Hash, Debug

fn on_signal(sig: Signal, handler: fn() + Send + Sync) throws (SignalError);
fn send_signal(pid: u32, sig: Signal) throws (IoError);
fn raise(sig: Signal) throws (IoError);  // Windows: Interrupt and Hangup only
fn id() -> u32;                          // this process's id; panics on wasm32, which has none
fn exit(code: i32) -> !;                 // exit now via the runtime's nabi_exit; finally and Drop do not run (refs-lang 4.16)
fn abort() -> !;                         // abnormal termination (SIGABRT); nothing is cleaned up

struct SignalError { .. }                // private fields (0.5.10)
impl SignalError {
    fn signal(self) -> Signal;           // the signal being installed
    fn os_code(self) -> i32 | none;      // errno (Windows: the error code); none if the dispatcher thread could not start
    fn message(self) -> str;
}
// + Display, Debug, Error, PartialEq, Eq, Clone
```

`on_signal` 处理函数不在信号上下文（信号处理函数内部）中运行。运行时接收信号，并把它交给专用的分发线程 `nabi-signal`，处理函数在那里像普通函数一样运行。因此处理函数必须是 `Send`；它会被保存下来，每个信号调用一次，所以它也是 `Sync`。处理函数可以分配内存或获取锁。对同一个信号再次调用 `on_signal` 会替换处理函数。没有处理函数的 `SIGTERM`、`SIGINT` 或 `SIGHUP` 在进程因其终止之前仍会写出 stdout 缓冲区（[2.23.8](../200-core/223-io.md#2238-标准流)）；有处理函数时，是否结束进程由处理函数决定，`exit` 会刷新缓冲区。`Signal` 在每个操作系统上为信号命名。`process_group(pgid)`（`CommandExt`，unix）把子进程放入该进程组（`0` 表示以子进程的 pid 新建一个组）。

在 Windows 上，控制台事件映射为信号：Ctrl-C → `Interrupt`（2），关闭控制台 / 注销 / 关机 → `Hangup`（1）。Ctrl-Break 没有对应的 `Signal`；由 `os::windows::process::on_ctrl_break` 转发它（[3.10.7](310-os.md#3107-oswindowsprocessctrl-break)）。`send_signal` 只发送 `Signal::Kill`，作为 `TerminateProcess`；`raise` 只引发 `Interrupt` 和 `Hangup`；其他任何 `Signal` 都是 `IoError`（`Unsupported`），对 `Interrupt` 或 `Hangup` 以外的信号调用 `on_signal` 是 `SignalError`。

大多数用户永远不需要接触这些。只有在守护进程、服务器和交互式 CLI 这类必须正面处理信号的场合，才需要这些工具。

## 3.6.6. 平台可用性

各平台的支持情况：

| 项               | linux      | macos | windows         | wasi |
|------------------|------------|-------|-----------------|------|
| `Command::spawn` | ✓          | ✓     | ✓               | ✗    |
| `posix_spawn`    | ✓          | ✓     | （CreateProcess）| ✗    |
| `pidfd`          | linux 5.3+（`os::linux`） | ✗ | ✗         | ✗    |
| `kill_on_drop`   | ✓          | ✓     | ✓               | ✗    |
| 信号处理函数     | ✓          | ✓     | （有限）        | ✗    |
| `process::id()`  | ✓          | ✓     | ✓               | panic（wasm32） |

在 wasm32 目标上，标记为 ✗ 的项在运行时以 `IoErrorKind::Unsupported` 失败。
