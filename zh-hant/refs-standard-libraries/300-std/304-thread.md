# 3.4. `thread::`：作業系統執行緒（1:1 模型）

`std::thread` 模組是作業系統層級執行緒的進入點。如 [2.18 task](../200-core/218-task.md) 所述，它與在 M:N 協作式排程器上執行的任務是不同的抽象；本模組提供直接啟動原生執行緒的途徑。

## 3.4.1. 何時使用 thread，何時使用 task

下表指出在何種情況下應選擇哪一種。

| 情況                           | 建議                                        |
|--------------------------------|---------------------------------------------|
| 一般並行 / 非同步 I/O          | `task::spawn`（輕量）                       |
| CPU 密集的計算                 | `task::spawn` 或 `par::*`（工作竊取）       |
| 阻塞的 C 函式庫呼叫            | `thread::spawn`（避免阻塞任務排程器）       |
| 需要作業系統執行緒 API 之處    | `thread::spawn`（TLS、訊號遮罩等）          |
| 嵌入式 / 無任務                | `thread::spawn`                             |

簡言之，凡是任務夠輕量、表達力也足夠之處，都以任務優先；降到執行緒層級，是為了必須直接處理作業系統資源的地方。

## 3.4.2. API

基本型別與函式簽章如下。

```nabi
struct Thread { /* OS handle, not generic */ }
struct JoinHandle<T> { /* join result T */ }
struct ThreadId { /* opaque */ }       // Clone, Copy, PartialEq, Eq, Hash, Debug

struct JoinError { .. }                 // private fields (0.5.10)
impl JoinError {
    fn new(kind: JoinErrorKind) -> JoinError;
    fn kind(self) -> JoinErrorKind;
    fn os_code(self) -> i32 | none;      // the number of an Os kind
    fn message(self) -> str;             // as Display shows it
}
// implements Display, Debug, Error, PartialEq, Eq and Clone

#[non_exhaustive]
enum JoinErrorKind {
    Os(i32),                             // the OS's join error number (errno)
    Lost,                                // the thread ended with a failed REPL cell
    Taken,                               // already joined or detached
}                                        // Clone, Copy, PartialEq, Eq, Hash, Debug

fn spawn<T: Send>(f: fn() -> T + Send) -> JoinHandle<T>;   // = try! Builder::new().spawn(f)

struct Builder { /* opaque: name, stack size */ }

impl Builder {
    fn new() -> Builder;                 // no name, default stack
    fn name(self, name: str) -> Builder;
    fn stack_size(self, size: usize) -> Builder;   // bytes (default 8 MiB)
    fn spawn<T: Send>(self, f: fn() -> T + Send) throws (IoError) -> JoinHandle<T>;
}

impl<T> JoinHandle<T> {
    fn join(self) throws (JoinError) -> T;
    fn detach(self);                     // does not join; the thread runs to completion
    fn thread(self) -> Thread;
}

impl<T: Send> IntoFuture for JoinHandle<T> { ... }   // `await handle` joins and yields T

impl ThreadId {
    fn as_u64(self) -> u64;              // the runtime's id of the thread
}

impl Thread {
    fn id(self) -> ThreadId;
    fn name(self) -> str | none;        // name given with Builder::name (also set as the OS thread name), none if absent
    fn unpark(self);
}

// current thread
fn current() -> Thread;
fn yield_now();
fn sleep(dur: Duration);
fn park();                                     // blocks until unparked (may return without a reason)
fn park_timeout(dur: Duration);
fn panicking() -> bool;                        // true on a thread that panicked, while the process ends

// thread-local storage
struct ThreadLocal<T> { .. }            // Send + Sync for every T: one value per thread
fn local<T>(init: fn() -> T + Sync) -> ThreadLocal<T>;     // init is called on first use in each thread

impl<T> ThreadLocal<T> {
    fn with<R>(self, f: fn(T) -> R) -> R;    // f of this thread's value, lent to f (scoped)
    fn set(self, value: T);                  // replaces this thread's value
    fn replace(self, value: T) -> T;         // replaces it, returns the old one
    unsafe fn as_ptr(self) -> Ptr<T>;        // this thread's value, pinned until the thread ends
}
impl<T: Copy> ThreadLocal<T> {
    fn get(self) -> T;                       // a copy of this thread's value
}
impl<T: Default> ThreadLocal<T> {
    fn take(self) -> T;                      // the value, leaving T::default()
}
```

`Thread::unpark` 設定該執行緒的停駐權杖並喚醒它。一個權杖由一次 `park` 消耗。

`thread::spawn` 是以預設值呼叫並以 `try!` 包住的 `Builder::spawn`：在 `Builder::spawn` 擲回之處（目標上沒有執行緒，或作業系統拒絕建立新執行緒），`thread::spawn` 會觸發 panic。若要處理該錯誤，請使用 `Builder::new().spawn(f)`。

`await handle`（[2.20.5](../200-core/220-future.md#2205-intofuture可-await-的值)）在任務內等待執行緒：join 在執行器的阻塞線道上執行（[`task::spawn_blocking`](../200-core/218-task.md#2181-核心-api)），因此只有該任務在等待。它產生 `T`；作業系統的 join 失敗（`JoinError`）會觸發 panic，因此若要處理它，請呼叫 `join`。

### panic 與 `JoinError`

任何執行緒中的 panic 都會以結束碼 101 終止整個行程。因此 `join` 絕不會收到其他執行緒的 panic，而 `thread::panicking()` 只對在行程結束期間、於觸發 panic 的執行緒上執行的程式碼（嵌入者的結束鉤子）為 true。`JoinError` 表示作業系統層級的 join 失敗（`JoinErrorKind::Os`），或者對以失敗的 REPL 儲存格結束的執行緒表示 `JoinErrorKind::Lost`（[refs-lang 9.7.0](../../refs-lang/009-tools/907-jit-cells.md#970-儲存格工作階段的規則)），顯示為「thread join failed: the thread ended with a failed REPL cell」。

`JoinErrorKind::Taken`：該控制代碼或其副本已經 join 或 detach。第二次 `join` 會擲回它；第二次 `detach` 不做任何事。

## 3.4.3. 範例

從最簡單的啟動再 join 模式，到建構器與多次啟動：

```nabi
use std::thread;

let h = thread::spawn(|| {
    heavy_blocking_work()
});
let result = try h.join();

// builder
let h = try thread::Builder::new()
    .name("worker-1")
    .stack_size(4 * 1024 * 1024)
    .spawn(|| { ... });

// multiple
let handles: List<thread::JoinHandle<i64>> = (0..10)
    .map(|i| thread::spawn(|| { compute(i) }))
    .collect();

let results: List<i64> = handles
    .into_iter()
    .map(|h| try! h.join())
    .collect();
```

## 3.4.4. 執行緒區域儲存

當每個執行緒都需要自己獨立的儲存空間時，請使用 `thread::local`。

```nabi
let counter = thread::local(|| 0);

try thread::spawn(|| {
    counter.set(counter.get() + 1);         // only this thread's value changes
}).join();
```

其形式仿照 Rust 的 `LocalKey`。`with(f)` 將目前執行緒的值借給 `f`，並回傳 `f` 的回傳值。該值是作用域檢視，如同鎖守衛的值一樣受到追蹤（[stdlib 2.16.3](../200-core/216-sync.md#2163-mutext互斥鎖)）：將它從 `f` 回傳、保存到呼叫之外（擷取、欄位、`Cell`）、傳送它，或把它傳給會保留它的被呼叫者，都是編譯錯誤 E0136；給 `with` 一個 `async` 閉包亦然，因為其 future 會跨越 `await` 持有該值，並可能在另一個工作者執行緒上繼續執行。請改為保留 `clone()` 或 `f` 計算出的結果。

```nabi
static SEEN: ThreadLocal<List<str>> = thread::local(|| -> List<str> { [] });

SEEN.with(|l| l.push("a"));             // changes this thread's list in place
let n = SEEN.with(|l| l.len());          // what f computes comes out
let copy = SEEN.with(|l| l.clone());     // so does a copy
let all = SEEN.take();                   // the list itself, leaving []
```

擁有形式將值移入與移出：`set(v)` 儲存 `v`，`replace(v)` 儲存 `v` 並回傳舊值，`take()` 回傳該值並留下 `T::default()`。`get()` 回傳副本，需要 `T: Copy`；對其他任何 `T` 都是編譯錯誤（請使用 `with`、`replace` 或 `take`）。`unsafe fn as_ptr()` 指向目前執行緒的值；該槽位在執行緒結束前保持固定，透過該指標進行的讀寫由呼叫者負責。

`with` 接受無法 `await` 的閉包，因此執行緒的值絕不會被跨越 `await` 持有；任務在 `await` 之後可能於另一個執行緒上繼續執行。

## 3.4.5. impl / target

各平台的支援：

| 項目            | linux/macos/windows   | wasm32-wasip2 / 瀏覽器    | wasm32-wasip1-threads |
|-----------------|-----------------------|---------------------------|---------------------|
| `spawn`         | ✓                     | 觸發 panic（`Builder::spawn` 回報 `Unsupported`） | ✓    |
| `Builder::spawn` | ✓                    | 擲回 `Unsupported`        | ✓                   |
| `local`         | ✓                     | ✓（單一執行緒）           | ✓                   |
| `park`/`unpark` | futex / WaitOnAddress | ✓                         | ✓                   |

在 `wasm32-wasip2` 與瀏覽器中只有一個執行緒：`thread::current().name()` 為 `none`。在 `wasm32-wasip1-threads` 上，`nabi_cpu_count()` 與可用的平行度為 4，因為 WASI 無法回報核心數。
