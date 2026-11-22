# 2.18. task：M:N 排程器任務 API

`core::task` 模組提供在 M:N 排程器上執行的協作式任務 API。受 Rust Tokio 與 Go goroutine 啟發，任務以**無堆疊協程**的形式實作於工作竊取（work-stealing）排程器之上。

模組 `task` 與 `Task<T>` 位於 [prelude](../000-meta/002-import.md) 中，因此 `task::spawn`、`task::race`、`Task<T>` 等不需匯入即可使用；模組前綴 `task::` 仍然保留。若程式碼應明確標示為非同步，建議使用 `use core::task::*;`，讓非同步流程一眼可辨。

## 2.18.1. 核心 API

核心 API 的簽章：

```nabi
fn block_on<F: Future>(f: F) -> F::Output;              // sync entry point: attaches to the executor and runs f to completion
fn try_block_on<T, E>(f: TryAsyncFuture<T, E>) throws (E) -> T;   // block_on for a throwing async body; its error is thrown here
fn spawn<F: Future + Send>(f: F) -> Task<F::Output> where F::Output: Send;
fn spawn_local<F: Future>(f: F) -> Task<F::Output>;     // non-Send future, pinned to the current worker (not an OS thread)
fn spawn_blocking<T: Send>(f: fn() -> T + Send) -> Task<T>;   // blocking work on an OS thread outside the executor
fn spawn_blocking_with_arg<V: Send, T: Send>(arg: V, f: fn(V) -> T + Send) -> Task<T>;   // spawn_blocking that hands arg to f
fn worker_count() -> usize;                               // pool size inside the executor, 1 outside
fn cpu_count() -> usize;                                // number of online CPUs (the pool size)
async fn race<I: IntoIterator<Item = Task<T>>, T>(tasks: I) -> T;   // result of the first task to complete; cancels the others
const fn select<I: IntoIterator<Item = F>, F: Future>(iter: I) -> SelectFuture<F>;   // await: (index, result), does not spawn
fn id() -> TaskId;                                      // id of the current task; panics outside a task
fn current_id() -> TaskId | none;                       // id of the current task; none outside a task
async fn join<I: IntoIterator<Item = Task<T>>, T>(tasks: I) -> List<T>;   // after all complete
async fn timeout<T>(dur: Duration, task: Task<T>) throws (TimeoutError) -> T;   // throws in try await; cancels the task
async fn sleep(dur: Duration);
async fn sleep_until(when: Instant);
async fn yield_now();                                   // await: yields once to other ready tasks
fn registrations() -> usize;                            // diagnostic: parked wakers + pending timers of the running pool (implementation-defined)
```

`sleep` 與 `timeout` 從第一次輪詢起計算時間長度。手寫的 `Future` 透過其 `Waker` 等待（[2.20](220-future.md)）：它保留 `ctx.waker()`、回傳 `Pending`，而改變狀態的一方在其上呼叫 `wake()`。

這些函式回傳的 future 與錯誤：

```nabi
struct SelectFuture<F> { /* ... */ }
impl<F: Future> Future for SelectFuture<F> { type Output = (usize, F::Output); ... }

struct Join<T> { /* ... */ }                        // Task::join, awaited with try await
impl<T> Join<T> {
    fn poll(ref self, ref ctx: Context) throws (CancellationError) -> Poll<T>;
}

struct TimeoutError;                                // task::timeout ran out; message() and Display "timed out"
```

### 範例

```nabi
use core::task::*;

let t = task::spawn(async { compute_heavy() });
let result = await t;

let results = await task::join([
    task::spawn(async { fetch("a") }),
    task::spawn(async { fetch("b") }),
    task::spawn(async { fetch("c") }),
]);

let first = await task::race([
    task::spawn(async { source_a() }),
    task::spawn(async { source_b() }),
]);

await task::sleep(Duration::ms(100));
await task::yield_now();
```

`spawn` 接受的是 future 而不是閉包，因此要傳入 `async` 區塊或 `async fn` 呼叫的結果，如 `task::spawn(async { ... })`。不是 `Send` 的 future 透過 `spawn_local` 在目前的 worker 上執行。`join` / `race` / `timeout` 要 `await` 才能取得結果。

`join`、`race` 和 `select` 接受其任務或 future 的任意 `IntoIterator`，包括串列字面值。

`race` 與 `timeout` 會取消落敗者。`race` 一旦看到第一個任務完成（依串列順序，因此平手時由較前面的任務勝出），就會對它收到的每個其他任務呼叫 `cancel`（[2.18.4](#2184-協作式取消taskcancel)）；`timeout` 的時間一旦用完，就會取消其任務並擲回 `TimeoutError`。落敗者即是如同被 `cancel` 取消的任務：

- 若它通過了取消點，`is_cancelled()` 為 true，`try await t.join()` 擲回 `CancellationError`，而單純的 `await t` 會觸發 panic，其訊息指出原因：``awaited a task cancelled by `task::race`; use `try await t.join()` ``（或 `task::timeout`）。
- 若它已經完成，或在未通過另一個取消點的情況下完成，`await t` 與 `join` 回傳它自己的值。落敗者永遠不會產出 `race` 的結果。

兩者都不等待落敗者的清理：一旦請求取消就回傳，與 `select` 相同。需要落敗者清理完成的程式碼請寫 `try await loser.join()`，它會在清理之後完成。取消正在 `await` `race` 或 `timeout` 的任務，會取消交給它的任務；在其他地方持有的此類 `Task` 的複本會看到該取消。對零個任務呼叫 `race` 會觸發 panic。

### `spawn_blocking`：阻塞工作

`spawn_blocking(f)` 在阻塞池的 OS 執行緒上而非執行器 worker 上執行 `f`，並回傳一個以其結果完成的 `Task<T>`，與 `spawn` 回傳的控制代碼相同（2.18.2）。它用於在任務中等待無法以就緒狀態 await 的阻塞系統呼叫（一般檔案 I/O 等，[stdlib 3.1.4a](../300-std/301-fs.md#314a-非同步檔案-io_async)）。

- `f` 在**呼叫時立即**開始，而不是在第一次輪詢時（若池已滿，則先排入佇列）。
- 每個行程有一個阻塞池。它會依需要增加執行緒（有上限），並結束閒置一段時間的執行緒。工作依到達順序處理（FIFO）。上限為 512 個執行緒；達到上限後，工作會排隊。閒置 10 秒的執行緒會結束。每個工作交給一個閒置的執行緒。
- `f` 沒有取消點。在池執行緒取得該工作之前呼叫 `cancel()`，表示 `f` 永遠不會執行，且 `join` 擲回 `CancellationError`；一旦 `f` 開始執行就會執行完畢，`await` / `join` 回傳其結果。
- 若正在 `await` 的任務被取消，它會像對任何 `Task` 一樣離開其 `await`；阻塞任務不會被取消，其結果留在控制代碼中。

```nabi
let n = await task::spawn_blocking(|| heavy_checksum(path));
```

`f` 與建立它的程式碼共用其一般擷取，因此這些擷取必須是 `Send + Sync`（[refs-lang 4.6.9](../../refs-lang/004-functions/406-capture-semantics.md#469-平行與並行上下文中的額外限制)）。只有阻塞工作需要的值（集合型別、`Command`）要交給它，有兩種形式：

- `f` 擷取清單中的 `[move x]` 會把 `x` 移動到閉包中（[refs-lang 4.6.11](../../refs-lang/004-functions/406-capture-semantics.md#4611-移動擷取move-x-擷取清單)）。
- `spawn_blocking_with_arg(arg, f)` 會像通道傳送一樣交出 `arg`，並在池執行緒上呼叫 `f(arg)`：該執行緒取得 `arg` 所觸及之每個集合型別的複本，之後使用 `arg` 是編譯錯誤（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）。`arg` 只需要 `Send`。

無論哪種方式，啟動端的程式碼都不能再使用該值。擁有被移動擷取的閉包也會被交出：在 `spawn_blocking(f)` 之後再次呼叫它是編譯錯誤。

```nabi
let rows: List<str> = load();
let n = await task::spawn_blocking([move rows] || write_all(rows));
let m = await task::spawn_blocking_with_arg(load(), |rows: List<str>| write_all(rows));
```

### `spawn` 的 `Send` 檢查

`task::spawn` 也要求結果型別是 `Send`（`where F::Output: Send`），並拒絕其**框架**不是 `Send` 的 future（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）。無論 future 從何而來都是如此：不只 `async` 區塊與 `async fn` 呼叫，參數、結構 / 元組 / 和型別的欄位、輔助函式的結果、串列元素、閉包持有的 future，以及其他協程框架中的 future 都會被檢查。future 的框架由以下組成：

- 擷取的值；
- 跨越 `await` 仍存活的區域變數與暫存值；
- 框架所持有的其他 future（遞迴地包括它們的框架）。

```nabi
let c = Cell::new(0);                      // Cell: !Send
task::spawn(async { c.set(1); });          // ERROR: !Send capture
task::spawn_local(async { c.set(1); });    // OK: pinned to the current worker

fn make() -> AsyncFuture<()> { let c = Cell::new(0); async { c.set(1); } }
task::spawn(make());                       // ERROR: the frame of a future made by a helper is checked too
```

凡是把閉包送到另一個執行緒之處，都適用相同的檢查：`thread::spawn`、`thread::Builder::spawn`、`par` 的 `Scope::spawn` 和 `task::spawn_blocking`。若閉包持有框架不是 `Send` 的 future，就會產生錯誤 ``` `X` would move a future to another thread: ... ```。

此檢查可以保守地近似。實作可以拒絕實際上是 `Send` 的 future，但絕不得接受會把非 `Send` 值傳到另一個執行緒的 future。此檢查不會深入：

- 型別無法持有函式值的值（它們無法隱藏 future 的框架）；
- 具有明確 `unsafe impl Send` 的型別的值（例如 `Task<T>`）；由該型別負責安全性。

`async fn` 的結果型別 `AsyncFuture<T>` 在 `T: Send` 時是 `Send`；其會擲回錯誤的對應型別 `TryAsyncFuture<T, E>` 在 `T` 與 `E` 皆為 `Send` 時是 `Send`。擷取與框架由上述檢查分別檢視。擷取為 `Send` 但非 `Sync` 的框架（`async fn` 接收的串列、`async` 區塊擷取的串列或 future）會被接受：`task::spawn` 會像通道傳送一樣交出該 future（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）。任務會取得自己的框架，其中持有 future 所擷取內容的複本，在一次交接中建立，因此擷取原本共用的內容（同一個串列出現兩次、一個串列與基於它的 future）仍然共用；在啟動之後使用該 future 或其擷取的內容是編譯錯誤（`E0105`），與傳送之後相同。只有從未被輪詢過的 future 才能交出：交出已開始（被 `select`、使用者的 `Future` 或手動輪詢過一次）或已完成的 future，或其框架或被擷取之 future 的框架含有 `[ref x]` 擷取的 future，會觸發 panic，且不會共用任何內容。經由通道傳送、放入 `Mutex` 或 `Frozen` 的協程 future 也以相同方式交出，每次都有自己的框架。若任務的 `Task` 被轉交出去（`Frozen::get`、通道、鎖、`par` 項目），每次對它的 `await` 都會取得結果的各自複本；同一執行緒上的 `Task` 複本則共用結果。

### `select`：最先完成的 future

```nabi
const fn select<I: IntoIterator<Item = F>, F: Future>(iter: I) -> SelectFuture<F>;   // Output = (usize, F::Output)

let (i, v) = await task::select([fetch_a(), fetch_b()]);
```

與 `race` 不同，它不會啟動這些 future；而是在等待中的任務**內部**執行它們。

- future 依串列順序被輪詢；若有兩個以上在同一次輪詢中完成，較小的索引勝出。
- 當其中一個完成時，其他所有已開始的 future 都會被取消。它們會在取消狀態下再被輪詢一次，讓 `finally` / `using` 得以執行（2.18.4）。清理中含有 `await` 的落敗者會在背景以自己的任務完成清理；`select` 不等待它。從未被輪詢的 future 永遠不會執行。
- 落敗 future 所做的所有喚醒註冊（計時器、停駐）都會被釋放。
- 若等待中的任務被取消，子 future 也會被取消。
- 沒有 future 時會觸發 panic。
- future 必須是單一型別。若結果型別不同，請包裝它們：`async { Msg::A(await a) }`。

逾時也可以由 `select` 與 `sleep` 建構。

```nabi
enum Ev { Line(str), Timeout }

let (_, ev) = await task::select([
    async { Ev::Line(await read_line()) },
    async { await task::sleep(Duration::secs(5)); Ev::Timeout },
]);
```

### `TaskId`

`task::id()` 是正在執行之任務的 ID；在任務外呼叫它會觸發 panic，訊息為 ``task::id: called outside a task; use `task::current_id()`, which is `none` there``。`Task::id()` 是該任務的 ID。ID 來自全行程的計數器，永不重複使用。

## 2.18.2. `Task<T>` 型別

`Task<T>` 型別定義如下。

```nabi
struct Task<T> {
    // internal: coroutine frame on the GC heap + result slot
}

impl<T> Task<T> {
    fn cancel(self);                      // requests cancellation; ends at the next await (2.18.4)
    fn is_cancelled(self) -> bool;
    fn is_finished(self) -> bool;
    fn join(self) -> Join<T>;             // try await t.join(): CancellationError if cancelled without a result
    fn id(self) -> TaskId;
}

struct TaskId { /* u64 */ }
impl TaskId {
    fn as_u64(self) -> u64;
}
// TaskId: Clone / Copy / PartialEq / Eq / Hash / Display / Debug

struct CancellationError;           // message() and Display "task cancelled"

impl<T> Future for Task<T> {              // `await t`: suspend until completion
    type Output = T;
    ...
}
```

### `IntoFuture` 整合

`Task<T>` 實作了 `Future<Output = T>`，因此可以用 `await t` 等待；它也涵蓋於 [2.20 future](220-future.md) 的 `IntoFuture` 全面實作（blanket impl）之中，因此等待方式與 `thread::JoinHandle`（它有自己的 `IntoFuture` 實作，3.4）相同。`IntoFuture` trait 與 lang 槽位 `future_trait` 的權威定義在 [2.20 future](220-future.md)；本節照樣遵循它們。

## 2.18.3. 排程器模型：工作竊取

一個行程有一個執行器。它在首次使用時啟動，由與 CPU 數量（`cpu_count()`）相同的 worker 執行緒、一條阻塞線道與一個 I/O 驅動器組成。`block_on` 將呼叫它的執行緒作為訪客 worker 附加到此執行器，並執行 `f` 直到完成。`block_on` 回傳時仍在執行的任務會繼續執行。`block_on` 只有在固定於其執行緒上的 `spawn_local` 任務也都完成後才會回傳；這些任務不能移到其他執行緒。`par::*` 與 `spawn_blocking` 使用同一個執行器。

以 `spawn` 建立的任務可以在任何 worker 上執行，因此它必須是 `Future + Send`，其結果也必須是 `Send`。來自 `spawn_local` 的任務固定於建立它的 worker。它固定的是 worker，而不是 OS 執行緒：worker 可能在另一個 OS 執行緒上繼續（[refs-lang 7.6.7](../../refs-lang/007-runtime/706-scheduler.md#767-自動偵測阻塞的系統呼叫)），因此 `spawn_local` 任務不應當依賴由哪個 OS 執行緒執行它，或該執行緒的執行緒區域狀態。（參考執行環境絕不會轉交仍有存活 `spawn_local` 任務的 worker。）若某任務由 `block_on` 執行緒輪詢但不歸其所有（它在等待時撿起的 `spawn` 任務），從該任務發出的 `spawn_local` 會改為固定於池中的 worker，因此 `block_on` 只等待它自己任務的區域任務。一次這樣的輪詢所啟動的所有區域任務都交給同一個池 worker，且只在該次輪詢結束後才開始，就如同在原本會執行它們的 worker 上一樣。

排程器的骨架如下。

```
Worker 0 ─┐      Worker 1 ─┐    ...    Worker N-1 ─┐
 Local     │      Local     │            Local      │
 deque     │      deque     │            deque      │
 (Chase-Lev)      (Chase-Lev)            (Chase-Lev)│
           └─ Global Injector Queue ──────┘
           └─ I/O Driver (portable) ──────┘
           └─ Timer (heap)  ───────────────┘
           └─ Parked Worker Stack   ──────┘
```

此結構的關鍵特性：區域佇列是無鎖的 Chase-Lev 雙端佇列，擁有它的 worker 與竊取者都先取最舊的任務（因此不斷重新喚醒自己的任務不會使其他任務飢餓，[refs-lang 7.6.2](../../refs-lang/007-runtime/706-scheduler.md#762-工作竊取結構)）。忙碌的 worker 每執行 32 個任務就處理一次到期的計時器。I/O 驅動器方面，每個行程有一個 reactor 執行緒以邊緣觸發方式等待 epoll / kqueue（io_uring / IOCP 是未來的工作，[refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)）；計時器結構由實作定義（二元堆積或時間輪，[refs-lang 7.7.4](../../refs-lang/007-runtime/707-io-and-timer.md#774-計時器結構)）。

標準函式庫的權威僅限於 API 簽章；演算法主體屬於執行環境與標準函式庫本體，不在規格範圍內。

執行器自身的型別與函式（`TaskRef`、`Pool`、`Worker`、`wake_task`、`cancel_requested`、`unwinding`、`park`、`unpark`）在 `core` 中是 `pub(pkg)`：其他套件不會指名它們。`await` 所呼叫的鉤子（`cancel_point`、`cancel_left`、`cleanup_running`、`cleanup_done`）也是 `pub(pkg)`；降階後的程式碼透過它們的 lang 槽位觸及它們（[0.4.4](../000-meta/004-lang-slot-table.md)）。`readable` / `writable`（通訊端 `*_async` 方法所 await 的就緒 future，[refs-lang 7.7.2](../../refs-lang/007-runtime/707-io-and-timer.md#772-io-等待模式)）、`wake_io`（在關閉或 shutdown 後喚醒描述元的可讀與可寫等待者）以及 `host_enter` / `host_leave` / `host_run`（WebAssembly 元件的事件迴圈入口）是 `#[internal]`：由 `std` 呼叫，從其他任何套件使用都是錯誤（[0.4](../000-meta/004-lang-slot-table.md)）。在 `host_enter` 之外呼叫 `host_leave` 會觸發 panic 並說明原因。

## 2.18.4. 協作式取消：`Task::cancel`

協作式取消的用法如下。

```nabi
let t = task::spawn(async { long_running() });
// ... other work ...
t.cancel();              // cooperative signal; the task ends at a cancellation point
```

`cancel()` **請求**取消。規則如下（[refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-取消)）：

- 每個 `await` 都是取消點。被取消的任務會在下一個 `await` 處如同 `return` 一般離開其主體，並由內而外執行它經過的 `finally` 與 `using` 清理。
- 這些清理中的 `await` 不是取消點：它會完成，任務會持續被輪詢直到清理結束。永不結束的清理會讓被取消的任務一直存活；請以 `task::timeout` 加以限制。
- 取消不是可以在任務內 `catch` 的錯誤。不會有 `CancellationError` 擲入任務。
- 阻塞系統呼叫不是取消點。任務會在該呼叫結束後的下一個 `await` 處離開。
- 等待通訊端 I/O（`*_async`）、`sleep` / `timeout`、`lock_async` 或 `recv_async` / `send_async` 的任務被取消時會醒來，並在下一個 `await` 處離開。
- 任務結束時，它所有的喚醒註冊（I/O 就緒、計時器、停駐等待）都會被釋放。
- 在第一次輪詢前就被取消的任務永遠不會執行：其主體的任何部分都不會執行，並在沒有結果的情況下結束。
- 若在等待 `spawn_blocking` 或建構於其上的 `fs::*_async` 時被取消，已開始的工作會在背景執行到完成。寫入等副作用確實會發生。在任務再次被輪詢前就已完成的結果，會由該次輪詢保留，如下所述。
- 被等待的 future 交出了值（訊息、鎖守衛）的那次輪詢會保留該值。取消檢查看的是「被等待的協程是否因取消而離開」；值會保留，任務在下一個 `await` 處離開（[refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-取消)）。
- 在通過最後一個取消點之後才被取消的任務會正常完成並有結果。`await t` / `join` 回傳該結果。
- `CancellationError` 由等待一個沒有結果就結束之任務的一方收到。以 `try await t.join()` 等待這樣的任務會擲回 `CancellationError`；以 `await t` 等待則會觸發 panic。

```nabi
let t = task::spawn(async {
    using log = open_log();                      // dropped here on cancellation
    loop {
        await task::sleep(Duration::secs(1));     // cancellation point
        tick(log);
    }
});
t.cancel();
```

```nabi
let r = try await t.join() catch _: CancellationError { fallback() };
```

## 2.18.5. `thread::*`：1:1 OS 執行緒

獨立的 `thread` 模組直接啟動 **OS 執行緒**，而不使用 M:N 模型。

```nabi
use std::thread;

let t = thread::spawn { os_level_work() };
let r = try t.join();          // OS-level synchronous wait
```

兩種模型的比較如下。

| 面向                  | `task::spawn`              | `thread::spawn`         |
|-------------------------|----------------------------|-------------------------|
| 模型                   | M:N（使用者空間排程器） | 1:1（OS 執行緒）         |
| 成本                    | 輕（約 1KB 框架）         | 重（約 MB 級堆疊）       |
| 協作式取消 | ✓                         | ✗（需要 OS 層級的終止） |
| 非同步整合       | ✓                          | △（只有 join 的一方 await） |
| 用途                | 一般並行（預設） | CPU 密集、阻塞 I/O |

預設請使用 `task::spawn`，只有在明確需要 OS 執行緒時才選擇 `thread::spawn`。

## 2.18.6. `Duration` / `Instant`

任務函式所接受的時間型別是 `core::time::Duration`（有號 `i64` 奈秒時間長度）與 `core::time::Instant`（單調時鐘讀數），定義於 [3.3](../300-std/303-time.md)（3.3.2、3.3.5）。它們位於 `task` 旁邊，而不在其中：`sleep`、`sleep_until` 和 `timeout` 之所以留在 `task`，是因為它們會暫停任務。

`await task::sleep(Duration::ms(100))` 或 `await task::sleep_until(t)` 這類呼叫保證使用精確的單調時間。

## 2.18.7. 模組對應

各項目所在的模組：

| 項目                 | 模組                                | 行為                 |
|----------------------|---------------------------------------|--------------------------|
| `task::spawn`        | `core::task` + 排程器執行環境 | 推入排程器           |
| `task::race`         | `core::task`                     | 最先完成者選擇器 |
| `task::join`         | `core::task`                     | 等待全部完成                 |
| `task::timeout`      | `core::task`                     | race + 計時器             |
| `Task<T>`            | `core::task`                     | 協程框架參考      |
| `Duration`/`Instant` | `core::time`                     | i64 ns + 單調時鐘 |
| `Future`（lang 槽位） | `core::future`                   | 基於輪詢               |
| `IntoFuture`         | `core::future`                   | await 進入點        |
