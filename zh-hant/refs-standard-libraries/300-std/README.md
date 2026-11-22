# 3. Std 層級

與目標相依領域的標準函式庫。它涵蓋檔案系統、網路、時間、作業系統執行緒、環境變數、行程、路徑、NABION 格式、作業系統亂數來源、特定作業系統的項目、外掛、panic 鉤子與熱重載，可用性隨使用者的環境（作業系統 / 平台）而異。

## 位置

規格本文位於 `spec/en/refs-standard-libraries/300-std/` 之下。

## 相依關係

向上，它相依於 `core` 與 `bare`。向下，使用者程式碼處於相依的位置，但 std 只會經由明確的匯入引入。

## 使用者匯入政策

隱含的 prelude 不包含 std 項目；`std::*` 需要另外明確匯入。由於可用性因目標而異，這項設計讓使用者**明確地**表達平台相依性。

```nabi
use std::fs;
use std::time::SystemTime;
use std::thread::Builder;
```

## 模組索引

此層級由以下十三個模組組成。

- [3.1. `fs::`：檔案系統（原子操作、Dir 能力、reflink 複製）](301-fs.md)
- [3.2. `net::`：TCP / UDP / Unix + TLS 1.3](302-net.md)
- [3.3. `time::`：時鐘、曆法日期與時區（搭配 `core::time` 的 `Instant` / `Duration`）](303-time.md)
- [3.4. `thread::`：作業系統執行緒（1:1，與 `task::` 分開）](304-thread.md)
- [3.5. `env::`：環境變數 / args / current_dir / home_dir](305-env.md)
- [3.6. `process::`：`Command` / `Child` / pidfd / kill_on_drop](306-process.md)
- [3.7. `path::`：UTF-8 `Path`（Python 風格）+ `RawPath` 逃生門](307-path.md)
- [3.8. `nabion::`：NABION 資料格式（`.nabion`）](308-nabion.md)
- [3.9. `random::`：亂數位元組、產生器與分布](309-random.md)
- [3.10. `os::`：只有部分作業系統才有的功能：`unix` / `linux` / `windows`](310-os.md)
- [3.11. `plugin::`：將外掛載入程式](311-plugin.md)
- [3.12. `panic::`：panic 鉤子（`set_hook` / `take_hook`）](312-panic.md)
- [3.13. `reload::`：熱重載鉤子與狀態（`point` / `on_reload` / `State`）](313-reload.md)
- `wit::`：產生的 WIT 繫結所呼叫的元件模型非同步 ABI（[1.8](../100-bare/108-wit.md)）。屬於繫結內部：設為公開是為了讓產生的程式碼能取用它，並非供直接呼叫，本規格也不加以規定。

## 引數慣例

此層級的每個簽章都適用兩個簡寫。

- 寫成 `Path` 的路徑參數代表型別參數 `P: AsPath`，因此同時接受 `str` 與 `Path`（[3.7.3a](307-path.md#373a-aspath路徑引數)）。兩個路徑參數彼此獨立（`P: AsPath, Q: AsPath`）。
- 寫入的資料（`Write::write` / `write_all`、`send`、`send_to`、`send_with_fd`、它們的 `_async` 形式、`fs` 的寫入函式）是 `B: AsBytes`，因此可以傳入 `str`、`List<u8>`、`Slice<u8>` 與 `[u8; N]`（[2.23.3](../200-core/223-io.md#2233-write-and-close)）。`write_zerocopy` 接受 `List<u8>`。

讀取緩衝區是 `SliceMut<u8>`：`read(buf)` 填入 `buf[0..n]` 並回傳 `n`，串流結束時為 `0`。`List<u8>` 以整個串列的檢視傳入，`buf[a:b]` 則傳入其中一部分。

## 核心設計原則

### 1. 阻塞 API + `*_async`：兩組 trait

std 的 I/O 函式是同步的，會阻塞呼叫它的作業系統執行緒。若要在任務中等待時讓出工作者，請 await 同一操作的 `*_async` 方法（[3.2.0](302-net.md#320-共通規則)）。在無堆疊協程下，普通函式無法暫停任務，因此同步呼叫絕不會自行變成非同步。

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

串流 I/O 透過 `core::io` 的 trait 進行（[2.23](../200-core/223-io.md)）：`Read` / `Write` 會阻塞，`AsyncRead` / `AsyncWrite` 會暫停任務，且方法名稱保留 `_async` 後綴，因此兩者可以同時在作用域內。不屬於串流 I/O 的操作（`connect_async`、`accept_async`、`open_async`、..）是型別的固有方法與函式。通訊端等待 I/O 驅動程式的就緒通知（[3.2.0](302-net.md#320-共通規則)）；一般檔案則在另一個作業系統執行緒上執行阻塞呼叫（[3.1.4a](301-fs.md#314a-非同步檔案-io_async)）。

### 2. I/O 後端：執行環境的細節

等待就緒是**執行環境的責任**（[refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)）。使用者程式碼不需要知道執行的是哪個後端。

| 平台            | 後端                        |
|-----------------|-----------------------------|
| Linux           | epoll（邊緣觸發）           |
| macOS / FreeBSD | kqueue（`EV_CLEAR`）        |
| Windows         | IOCP                        |
| WASI            | `wasi:io/poll`（`wasm32-wasip2`）、`poll_oneoff`（`wasm32-wasip1-threads`） |

### 3. TLS 1.3 在 std 內，HTTP 在外

TLS 1.3 **攸關安全**。若每個應用程式都自行實作 TLS，安全漏洞將會倍增，因此它被整合進 std。

```nabi
use std::net::tls;

let conn = try net::tcp_connect("example.com:443");
let tls_conn = try tls::Client::new(conn, "example.com");
let req = b"GET / HTTP/1.1\r\nHost: example.com\r\n\r\n";
try tls_conn.write_all(req);
let resp = try tls_conn.read_to_end();
try tls_conn.close();
```

這個協定堆疊是 Nabi 自己的，建立在 `core::crypto` 之上（[2.21](../200-core/221-crypto.md)）；不使用作業系統或第三方的 TLS 函式庫（[3.2.5](302-net.md#325-tls-13nettls)）。

另一方面，HTTP/1、HTTP/2 與 HTTP/3（QUIC）位於 **std 之外**，在獨立的套件中（`http` 涵蓋 HTTP/1.1 與 HTTP/2）。這是從 Go 的 `net/http` **凍結於 1.0** 所得到的教訓。

### 4. 以能力為基礎的 fs：Dir 控制代碼

遵循 WASI Preview 2 預先開啟目錄控制代碼的模型。每個 fs 操作都**經由 Dir** 進行。

```nabi
use std::fs::{Dir, File};

let root: Dir = Dir::root();                  // capability, at process start
let data_dir: Dir = try root.open_dir("data"); // sub-cap

let f: File = try data_dir.open("config.toml");
let bytes = try f.read_to_end();
try f.close();
```

此模型有三項優點。第一，它避免 TOCTOU：在 `open_dir` 之後進行的 `open` 沒有競爭。第二，沙箱化自然而然：只把**範圍狹窄的 Dir** 傳給子任務。第三，它與 WASI 相容：同一份程式碼可在原生環境與 WASI 上運作。

為了方便，也允許 `fs::open(absolute_path)`。它在內部展開為 `Dir::root().open(...)`。

### 5. UTF-8 Path（Python 風格）

這項決定是為了避免 Rust 的 `Path` / `PathBuf` + `OsStr` 帶來的**使用負擔**。

```nabi
use std::path::Path;

let p: Path = Path::new("data/config.toml");
let parent: Path = p.parent()?;
let with_ext: Path = p.with_extension("bak");

// `/` operator (Python pathlib style)
let full = Path::new("/etc") / "nginx" / "nginx.conf";
```

`Path` 是 **UTF-8 newtype**。非 UTF-8 的檔名（在 Unix 上偶爾會出現）另以 `RawPath`（位元組）處理。單靠 `Path` 應當就能涵蓋 99% 的程式碼。

詳細的比較與理由見 [3.7 path](307-path.md)。

### 6. 所有 I/O 都 `throws (IoError)`

panic 幾乎不會發生（OOM 除外）。找不到檔案、權限遭拒與網路失敗，都以 `throws (IoError)` 簽章擲回，並透過以 `try` 標示呼叫來傳播。錯誤型別之間沒有自動轉換；若要轉成其他錯誤型別，請以 `catch` 包住並重新擲回。

```nabi
fn load_config() throws (ConfigError) -> Config {
    let bytes = try fs::read("config.toml") catch e: IoError {
        throw ConfigError::Io(e);              // IoError → ConfigError (explicit conversion)
    };
    try parse(bytes)
}
```

## 目標可用性

各平台的模組可用性：

| 模組                | linux/macos/windows | wasm32-wasip2   | wasm32-wasip1-threads | wasm32（瀏覽器） |
|---------------------|---------------------|---------------|---------------------|------------------|
| `std::fs`      | ✓                   | ✓（以能力為基礎） | ✓（以能力為基礎）  | ✗（每個呼叫皆為 `Unsupported`） |
| `std::net`     | ✓（+ TLS 1.3）      | ✓（`wasi:sockets`） | 由主機傳入的監聽器 | ✗（請用 `fetch`） |
| `std::time`    | ✓                   | ✓             | ✓                   | ✓（Date.now）    |
| `std::thread`  | ✓                   | 單一執行緒（`Builder::spawn` 為 `Unsupported`） | ✓（wasi-threads） | 單一執行緒（`Builder::spawn` 為 `Unsupported`） |
| `std::env`     | ✓                   | ✓             | ✓                   | ✗                |
| `std::process` | ✓                   | ✗             | ✗                   | ✗                |
| `std::path`    | ✓                   | ✓             | ✓                   | ✓                |
| `std::nabion`   | ✓                   | ✓             | ✓                   | ✓                |
| `std::random`  | ✓                   | ✓（`random_get`） | ✓（`random_get`） | ✓（`crypto.getRandomValues`，3.9） |
| `std::wit`     | 空                  | ✓             | ✓                   | 空               |
| `std::os`      | 僅限其作業系統（3.10） | 不存在     | 不存在              | 不存在           |
| `std::panic`   | ✓                   | ✓             | ✓                   | ✓                |

✗ 表示模組可以建置，但其操作在執行時以 `IoErrorKind::Unsupported` 失敗；並非在編譯時不存在。`os::unix`、`os::linux` 與 `os::windows` 是例外：它們只存在於各自的目標上。`wit` 在任何地方都存在，但只有在 WASI 目標上才有項目（「空」）。在瀏覽器中，stdout / stderr 送往主機的主控台，stdin 則處於 EOF。各模組規格的可用性一節會更詳細地說明各平台的行為。
