# 3. Std 层级

依赖目标的那部分标准库。它涵盖文件系统、网络、时间、操作系统线程、环境变量、进程、路径、NABION 格式、操作系统随机源、操作系统特有的项、插件、panic 钩子和热重载，可用性随用户环境（操作系统 / 平台）而不同。

## 位置

规范正文位于 `spec/zh-hans/refs-standard-libraries/300-std/` 下。

## 依赖

向上，它依赖 `core` 和 `bare`。向下，用户代码处于依赖方的位置，但只有显式导入时才会引入 std。

## 用户导入策略

隐式 prelude 不包含 std 的项；`std::*` 需要另行显式导入。由于可用性因目标而异，这样的设计让用户**显式**表达对平台的依赖。

```nabi
use std::fs;
use std::time::SystemTime;
use std::thread::Builder;
```

## 模块索引

本层级由以下十三个模块组成。

- [3.1. `fs::`：文件系统（原子操作、Dir 能力、reflink 复制）](301-fs.md)
- [3.2. `net::`：TCP / UDP / Unix + TLS 1.3](302-net.md)
- [3.3. `time::`：时钟、日历日期和时区（`Instant` / `Duration` 来自 `core::time`）](303-time.md)
- [3.4. `thread::`：操作系统线程（1:1，与 `task::` 分开）](304-thread.md)
- [3.5. `env::`：环境变量 / args / current_dir / home_dir](305-env.md)
- [3.6. `process::`：`Command` / `Child` / pidfd / kill_on_drop](306-process.md)
- [3.7. `path::`：UTF-8 `Path`（Python 风格）+ `RawPath` 逃生口](307-path.md)
- [3.8. `nabion::`：NABION 数据格式（`.nabion`）](308-nabion.md)
- [3.9. `random::`：随机字节、生成器和分布](309-random.md)
- [3.10. `os::`：只有部分操作系统才有的内容：`unix` / `linux` / `windows`](310-os.md)
- [3.11. `plugin::`：把插件加载到程序中](311-plugin.md)
- [3.12. `panic::`：panic 钩子（`set_hook` / `take_hook`）](312-panic.md)
- [3.13. `reload::`：热重载钩子和状态（`point` / `on_reload` / `State`）](313-reload.md)
- `wit::`：生成的 WIT 绑定所调用的组件模型异步 ABI（[1.8](../100-bare/108-wit.md)）。仅供绑定内部使用：它是公开的，以便生成的代码能够访问，但不应直接调用，本规范也不对它作规定。

## 参数约定

本层级的所有签名都遵循两条简写。

- 写作 `Path` 的路径参数代表类型参数 `P: AsPath`，因此 `str` 和 `Path` 都可以传入（[3.7.3a](307-path.md#373a-aspath路径参数)）。两个路径参数彼此独立（`P: AsPath, Q: AsPath`）。
- 写入的数据（`Write::write` / `write_all`、`send`、`send_to`、`send_with_fd`、它们的 `_async` 形式，以及 `fs` 的写入函数）是 `B: AsBytes`，因此可以传入 `str`、`List<u8>`、`Slice<u8>` 和 `[u8; N]`（[2.23.3](../200-core/223-io.md#2233-write-与-close)）。`write_zerocopy` 接受 `List<u8>`。

读取缓冲区是 `SliceMut<u8>`：`read(buf)` 填充 `buf[0..n]` 并返回 `n`，到达流末尾时返回 `0`。`List<u8>` 作为整个列表的视图传入，`buf[a:b]` 作为其中一部分传入。

## 核心设计原则

### 1. 阻塞 API + `*_async`：两套 trait

std 的 I/O 函数是同步的，会阻塞调用它的操作系统线程。要在任务中等待时让出工作线程，就 `await` 同一操作的 `*_async` 方法（[3.2.0](302-net.md#320-通用规则)）。在无栈协程下，普通函数无法挂起任务，所以同步调用永远不会自行变成异步调用。

```nabi
// blocking: on an OS thread (thread::spawn) or outside a task
fn fetch(addr: str) throws (IoError) -> List<u8> {
    using conn = try net::tcp_connect(addr) {
        try conn.read_to_end()
    }
}

// async: suspends only the task
async fn fetch_async(addr: str) throws (IoError) -> List<u8> {
    using conn = try await net::tcp_connect_async(addr) {
        try await conn.read_to_end_async()
    }
}
```

流 I/O 通过 `core::io` 的 trait 进行（[2.23](../200-core/223-io.md)）：`Read` / `Write` 会阻塞，`AsyncRead` / `AsyncWrite` 会挂起任务，方法名保留 `_async` 后缀，因此两者可以同时处于作用域中。不属于流 I/O 的操作（`connect_async`、`accept_async`、`open_async` 等）是各类型的固有方法和函数。套接字等待 I/O 驱动报告就绪（[3.2.0](302-net.md#320-通用规则)）；普通文件则在另一个操作系统线程上执行阻塞调用（[3.1.4a](301-fs.md#314a-异步文件-io_async)）。

### 2. I/O 后端：运行时细节

等待就绪是**运行时的职责**（[refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)）。用户代码无需知道运行的是哪个后端。

| 平台            | 后端                        |
|-----------------|-----------------------------|
| Linux           | epoll（边沿触发）           |
| macOS / FreeBSD | kqueue（`EV_CLEAR`）        |
| Windows         | IOCP                        |
| WASI            | `wasi:io/poll`（`wasm32-wasip2`）、`poll_oneoff`（`wasm32-wasip1-threads`） |

### 3. TLS 1.3 在 std 中：HTTP 在外

TLS 1.3 **攸关安全**。如果每个应用都自己实现 TLS，安全漏洞就会成倍增加，因此它被集成到 std 中。

```nabi
use std::net::tls;

let conn = try net::tcp_connect("example.com:443");
let tls_conn = try tls::Client::new(conn, "example.com");
let req = b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n";
try tls_conn.write_all(req);
let resp = try tls_conn.read_to_end();
try tls_conn.close();
```

这套协议栈是 Nabi 自己的，基于 `core::crypto`（[2.21](../200-core/221-crypto.md)）编写；不使用操作系统或第三方的 TLS 库（[3.2.5](302-net.md#325-tls-13nettls)）。

而 HTTP/1、HTTP/2 和 HTTP/3（QUIC）则位于 **std 之外**的独立包中（`http` 涵盖 HTTP/1.1 和 HTTP/2）。这是从 Go 的 `net/http` **冻结在 1.0** 中得到的教训。

### 4. 基于能力的 fs：Dir 句柄

沿用 WASI Preview 2 的预打开目录句柄模型。所有 fs 操作都**通过 Dir** 进行。

```nabi
use std::fs::{Dir, File};

let root: Dir = Dir::root();                  // capability, at process start
let data_dir: Dir = try root.open_dir("data"); // sub-cap

let f: File = try data_dir.open("config.toml");
let bytes = try f.read_to_end();
try f.close();
```

这个模型有三个优点。第一，它避免了 TOCTOU：在 `open_dir` 之后 `open` 不存在竞争。第二，沙箱化自然而然：只把一个**范围狭窄的 Dir** 交给子任务。第三，它与 WASI 兼容：同一份代码在原生平台和 WASI 上都能运行。

为方便起见，也允许 `fs::open(absolute_path)`。它在内部展开为 `Dir::root().open(...)`。

### 5. UTF-8 Path（Python 风格）

这一决定是为了避免 Rust 的 `Path` / `PathBuf` + `OsStr` 带来的**使用负担**。

```nabi
use std::path::Path;

let p: Path = Path::new("data/config.toml");
let parent: Path = p.parent()?;
let with_ext: Path = p.with_extension("bak");

// `/` operator (Python pathlib style)
let full = Path::new("/etc") / "nginx" / "nginx.conf";
```

`Path` 是一个 **UTF-8 newtype**。非 UTF-8 的文件名（在 Unix 上偶尔存在）另用 `RawPath`（字节）处理。单靠 `Path` 应当能覆盖 99% 的代码。

详细的比较和理由见 [3.7 path](307-path.md)。

### 6. 所有 I/O 都 `throws (IoError)`

几乎不会发生 panic（OOM 除外）。文件不存在、权限被拒和网络故障都以 `throws (IoError)` 签名抛出，并通过在调用处标上 `try` 来传播。错误类型之间没有自动转换；要把一种错误转成另一种错误类型，就用 `catch` 包裹后重新抛出。

```nabi
fn load_config() throws (ConfigError) -> Config {
    let bytes = try fs::read("config.toml") catch e: IoError {
        throw ConfigError::Io(e);              // IoError → ConfigError (explicit conversion)
    };
    try parse(bytes)
}
```

## 目标可用性

各平台上的模块可用性：

| 模块                | linux/macos/windows | wasm32-wasip2   | wasm32-wasip1-threads | wasm32（浏览器） |
|---------------------|---------------------|---------------|---------------------|------------------|
| `std::fs`      | ✓                   | ✓（基于能力） | ✓（基于能力）       | ✗（每次调用都 `Unsupported`） |
| `std::net`     | ✓（+ TLS 1.3）      | ✓（`wasi:sockets`） | 由主机传入监听器 | ✗（使用 `fetch`） |
| `std::time`    | ✓                   | ✓             | ✓                   | ✓（Date.now）    |
| `std::thread`  | ✓                   | 单线程（`Builder::spawn` 为 `Unsupported`） | ✓（wasi-threads） | 单线程（`Builder::spawn` 为 `Unsupported`） |
| `std::env`     | ✓                   | ✓             | ✓                   | ✗                |
| `std::process` | ✓                   | ✗             | ✗                   | ✗                |
| `std::path`    | ✓                   | ✓             | ✓                   | ✓                |
| `std::nabion`   | ✓                   | ✓             | ✓                   | ✓                |
| `std::random`  | ✓                   | ✓（`random_get`） | ✓（`random_get`） | ✓（`crypto.getRandomValues`，3.9） |
| `std::wit`     | 空                  | ✓             | ✓                   | 空               |
| `std::os`      | 仅限各自的操作系统（3.10） | 不存在   | 不存在              | 不存在           |
| `std::panic`   | ✓                   | ✓             | ✓                   | ✓                |

✗ 表示该模块可以构建，但其操作在运行时以 `IoErrorKind::Unsupported` 失败；这并不是编译时不存在。`os::unix`、`os::linux` 和 `os::windows` 是例外：它们只存在于各自的目标上。`wit` 在所有目标上都存在，但只在 WASI 目标上有项（“空”）。在浏览器中，stdout / stderr 输出到主机控制台，stdin 处于 EOF。各模块规范的可用性一节会更详细地给出各平台上的行为。
