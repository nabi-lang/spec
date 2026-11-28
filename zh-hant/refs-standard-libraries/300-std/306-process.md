# 3.6. `process::`：子行程

`std::process` 模組啟動子行程並處理訊號。基本骨架沿用 Rust 的 `Command` 建構器模式，再加上現代的防護：`pidfd`、`kill_on_drop`，以及 `posix_spawn` 快速路徑。

## 3.6.1. `Command` 建構器

最先接觸的是 `Command` 結構，它是子行程的藍圖。所有設定（要執行的程式、引數、環境變數、標準 I/O 的連接方式）都在啟動之前逐步建立。

路徑參數遵循整個層級共通的[引數慣例](README.md#引數慣例)（`Path` 即 `P: AsPath`）。

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

建構器方法回傳變更後的指令，被呼叫的那個指令保持不變，所以 `Command` 的各個副本彼此獨立：`let a = base.arg("x")` 之後，`base` 沒有 `x`。`Command` 不是仿射的，但其中的 `Stdio::File` 是（[2.13.4](../200-core/213-marker.md)）：那一個檔案是例外，由所有副本共用（見下文）。結果被捨棄的建構器呼叫（`base.arg("x");`）不會改變任何東西，並會發出警告（`#[must_use]`）。

`Stdio` 有四個分支：`Inherit` 傳遞父行程的 fd；`Null` 完全捨棄輸出；`Piped` 用於雙向通訊；`File` 接上一個已開啟的檔案。

移入 `Stdio::File` 的 `File` 屬於該指令，而不屬於程式：這次移動會消耗它，就像任何 `File` 的移動一樣（[2.13.4](../200-core/213-marker.md)），程式不會關閉它。執行指令（`spawn`、`output`、`status` 及其 `_async` 對應版本）會在子行程啟動後立即關閉父行程的那份副本，啟動失敗時也會關閉；子行程保有自己的副本。此時關閉失敗即為這次執行的 `IoError`：已啟動的子行程會先被終止並回收，所以不會遺失任何 `Child`。若某個指令副本的檔案已被先前的執行關閉，該副本不會執行，而是以 `InvalidInput` 失敗；請讓每次執行使用自己的檔案（`File::try_clone`）。從未執行的指令會讓它的檔案一直開著，直到程式結束：垃圾回收器釋放記憶體，但不關閉任何東西。

持有檔案的 `Stdio` 也是仿射的：接收它的建構器呼叫會把它移入指令。把同一個 `Stdio::File` 值交給兩個指令，或交給同一個指令的兩個串流（`.stdout(s).stderr(s)`），是編譯錯誤 `E0133`；請讓每一處使用自己的檔案（`Stdio::File(try f.try_clone())`）。`Stdio::Inherit`、`Null` 與 `Piped` 不持有任何東西，可以交出任意次。`Stdio` 實作 `Close`：從未交給指令的 `Stdio::File` 由程式關閉（`try s.close()`），就像它的 `File` 一樣。

### `run`

```nabi
pub fn run<P: AsPath>(program: P, args: List<str> = []) throws (IoError | ExitStatusError | Utf8Error) -> str;
```

建立在 `Command` 之上的便利形式：以 `Command::output` 的方式用 `args` 執行 `program`，並以文字回傳其 stdout。非成功的狀態會擲回 `ExitStatusError`（`status()`、以文字表示且無效 UTF-8 已被取代的 `stderr()`、`message()`）；不是 UTF-8 的 stdout 會擲回 `Utf8Error`。`let out = try process::run("git", ["log", "-1"]);`（相當於 Python 的 `subprocess.run(.., check=True, capture_output=True, text=True).stdout`）。

## 3.6.2. `Child`

`spawn` 成功時回傳 `Child`，即執行中行程的控制代碼。透過它可以存取標準 I/O 串流、等待結束或傳送訊號。

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

結束結果保存在不透明型別 `ExitStatus` 中。若 `code()` 為 `none`，表示行程在 unix 上被訊號終止，`signal()`（`ExitStatusExt`，在 unix 上位於 prelude 中）會以 `Signal` 指出是哪個訊號。

## 3.6.3. 範例

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

如最後一個範例所示，把啟用 `kill_on_drop` 的 `Child` 繫結在 `using` 中，即使遇到錯誤傳播或提前回傳之類的流程，也會清理子行程。`Drop` 在擁有者釋放 `Child` 的地方發出終止：`using` 區塊結束處，或其擁有者離開作用域處（[2.13.4](../200-core/213-marker.md)）。`ChildStdin` / `ChildStdout` / `ChildStderr` 是管道的檔案控制代碼，讀寫方式與 `File` 相同。`Child` 持有它的 `Piped` 串流，直到 `take_stdin` / `take_stdout` / `take_stderr` 交出其中一個並留下 `none`；從那之後由取走者擁有該管道，並且必須像 `File` 一樣關閉它。從未被取走的管道在 `Child` 被釋放處關閉。`wait` 會先關閉仍由 `Child` 持有的 stdin，讓讀到輸入結尾的子行程可以結束；已被取走的 stdin 應由取走者在 `wait` 之前關閉。

`Child` 是仿射的（[2.13.4](../200-core/213-marker.md)）：由一個擁有者持有它的管道並等待它。若要在擁有者等待期間從別處停止子行程（逾時、關機訊號），用 `handle()` 取得 `ChildHandle`，再把它傳給另一個執行緒或任務。控制代碼不擁有任何東西：沒有管道、不能等待、沒有要關閉的東西；它只能傳送訊號。`wait` / `try_wait` 只在透過控制代碼進行中的 `send_signal` / `kill` 呼叫回傳之後才回收子行程，所以訊號絕不會送到已被重複使用的 pid 或 pidfd；子行程已結束且正在被回收時，`send_signal` 或 `kill` 不做任何事。

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

## 3.6.4. 現代的補強：pidfd / posix_spawn / CLOEXEC

在建構器模式之上，標準函式庫加入了幾項現代的補強。各項的意義與效果如下：

| 功能                    | 意義                          | 效果                    |
|-------------------------|-------------------------------|-------------------------|
| `posix_spawn` 快速路徑 | 以一個系統呼叫取代 fork+exec | 從龐大的行程更快啟動 |
| `pidfd`（Linux 5.3+）    | 以 fd 而非 pid 追蹤    | 沒有競爭的訊號        |
| 預設 `CLOEXEC`       | 每個 fd 自動設為 close-on-exec | 防止 fd 洩漏 |
| `kill_on_drop`          | 在擁有者釋放處終止 | 防止行程洩漏  |

這些決策一致地以減少 fd 洩漏、行程洩漏與訊號競爭為目標。

使用 `Command::env` 時，程式仍在父行程的 `PATH` 上尋找，而不是子行程的 `PATH`。

## 3.6.5. 訊號處理（進階）

再深入一層，是訊號處理函式。

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

`on_signal` 的處理常式不在訊號情境（訊號處理常式內部）中執行。執行環境接收訊號後，把它交給專用的分派執行緒 `nabi-signal`，處理常式在那裡像一般函式一樣執行。因此處理常式必須是 `Send`；它會被保存並在每次訊號時呼叫一次，所以也是 `Sync`。它可以配置記憶體或取得鎖。對同一個訊號再次呼叫 `on_signal` 會取代處理常式。沒有處理常式的 `SIGTERM`、`SIGINT` 或 `SIGHUP` 在行程因其結束之前仍會寫出 stdout 緩衝區（[2.23.8](../200-core/223-io.md#2238-標準串流)）；有處理常式時，是否結束行程由處理常式決定，`exit` 會排清緩衝區。`Signal` 在每個作業系統上都能指稱訊號。`process_group(pgid)`（`CommandExt`，unix）把子行程放入該行程群組（`0` 表示以子行程的 pid 建立新群組）。

在 Windows 上，主控台事件對應到訊號：Ctrl-C → `Interrupt`（2），主控台關閉 / 登出 / 關機 → `Hangup`（1）。Ctrl-Break 沒有對應的 `Signal`；由 `os::windows::process::on_ctrl_break` 轉送（[3.10.7](310-os.md#3107-oswindowsprocessctrl-break)）。`send_signal` 只傳送 `Signal::Kill`（即 `TerminateProcess`），`raise` 只引發 `Interrupt` 與 `Hangup`；其他任何 `Signal` 都是 `IoError`（`Unsupported`），而對 `Interrupt` 或 `Hangup` 以外的訊號呼叫 `on_signal` 會得到 `SignalError`。

大多數使用者永遠不需要碰這些。只有在必須正面處理訊號的地方，例如常駐程式、伺服器與互動式 CLI，才需要這些工具。

## 3.6.6. 平台可用性

各平台的支援：

| 項目             | linux      | macos | windows         | wasi |
|------------------|------------|-------|-----------------|------|
| `Command::spawn` | ✓          | ✓     | ✓               | ✗    |
| `posix_spawn`    | ✓          | ✓     | （CreateProcess） | ✗    |
| `pidfd`          | linux 5.3+（`os::linux`） | ✗ | ✗         | ✗    |
| `kill_on_drop`   | ✓          | ✓     | ✓               | ✗    |
| 訊號處理常式   | ✓          | ✓     | （有限）       | ✗    |
| `process::id()`  | ✓          | ✓     | ✓               | 觸發 panic（wasm32） |

在 wasm32 目標上，標示 ✗ 的項目會在執行時以 `IoErrorKind::Unsupported` 失敗。
