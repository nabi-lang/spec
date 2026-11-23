# 2.20. future：`Future` / `await` 整合

`core::future` 模組提供非同步機制的權威 trait。`await expr` 的去語法糖會降階為對本章定義的 `Future` trait 的分派。本章也定義 lang 槽位之一 `future_trait`（[0.4](../000-meta/004-lang-slot-table.md)）。

## 2.20.1. `Future`：非同步結果

`Future` 是表示非同步計算結果的核心 trait；結果型別是關聯型別 `Output`。

```nabi
#[lang("future_trait")]
trait Future {
    type Output;

    fn poll(ref self, ref ctx: Context) -> Poll<Output>;
}

enum Poll<T> {
    Ready(T),
    Pending,
}

impl<T> Poll<T> {
    const fn is_ready(self) -> bool;        // Ready(_)
    const fn is_pending(self) -> bool;      // Pending
}
```

`poll(ctx)` 有兩種結果。`Ready(v)` 表示結果已就緒，`v` 原樣回傳；`Pending` 表示還沒有結果，當 `ctx` 中的喚醒器喚醒它時，future 會再次被輪詢。

此模型與 Rust 的 `Future` 和 `Poll`（Nabi 的是 `core::future::Poll`）同構，並編譯為**無堆疊協程**（由編譯器產生的狀態機，[refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)）。

## 2.20.2. `Context` / `Waker`

`Context` 與 `Waker` 構成輪詢與排程器之間的溝通管道。

```nabi
struct Context { /* ... */ }         // fields are private

impl Context {
    const fn from_waker(waker: Waker) -> Context;   // a context whose waker is waker
    const fn waker(self) -> Waker;                  // the waker to register before returning Pending
}

struct Waker {
    // internal: a callback that communicates with the scheduler
}

impl Waker {
    fn wake(self);              // signal to schedule this task again
}
impl Clone for Waker;           // a copy wakes the same task
impl Debug for Waker;
```

`poll` 回傳 `Pending` 時，內部會登記一個 Waker；之後事件（I/O 或計時器）發生時，喚醒器被呼叫，任務再次被排程。大多數使用者不會直接處理 `Waker`，因為 `await` 會自動處理這些事。

`waker` 欄位是函式庫私有的：手寫的 `poll` 用 `ctx.waker()` 讀取它，保留一份複本，並在能夠繼續推進時呼叫 `wake`。`Context::from_waker` 由手上已有的喚醒器建立上下文，用於在同一個喚醒器下手動輪詢內層 future。`Waker` 本身只由執行器建立。

## 2.20.3. `await expr` 的去語法糖

最常見的使用形式：

```nabi
let v = await some_future;
```

這在概念上去語法糖為：

```nabi
let __f = IntoFuture::into_future(some_future);
let v = loop {
    match __f.poll(ref CURRENT_CONTEXT) {
        Ready(v)  => break v,
        Pending   => yield_to_scheduler(),
    }
};
```

編譯器把它轉成狀態機，每個 `await` 成為一個**暫停點**。只有框架變數會被保留，存放在位於 GC 堆積上的協程狀態中。

## 2.20.4. `async fn` / `async { }` / `async || { }`

這三種形式都回傳 **`AsyncFuture<T>`**（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#協程函式的結果型別)）。

```nabi
// (a) named function
async fn fetch(url: str) -> Response {
    let raw = await get(url);
    await parse(raw)
}
await fetch("/");       // fetch("/"): AsyncFuture<Response>

// (b) inline block (a value)
let f = async { await compute() };       // AsyncFuture<T>
await f;

// (c) closure
let g = async || { await compute() };    // a closure that, when called, returns a value implementing Future<Output = T>
await g();
```

這**三種形式**的權威定義在 [refs-lang 4.12 async](../../refs-lang/004-functions/412-async.md)。

## 2.20.5. `IntoFuture`：可 await 的值

`IntoFuture` 是 await 的統一進入點。不只 `Future`，像 `JoinHandle<T>`（[3.4](../300-std/304-thread.md)）這類本身不是 `Future` 的控制代碼，也可以透過這個 trait 被 `await`。

```nabi
trait IntoFuture {
    type Output;
    type IntoFuture: Future<Output = Output>;

    fn into_future(self) -> IntoFuture;
}

impl<F: Future> IntoFuture for F {               // Task<T>: Future<Output = T> is included here too
    type Output = F::Output;
    type IntoFuture = F;
    fn into_future(self) -> F { self }
}
impl<T: Send> IntoFuture for JoinHandle<T> { type Output = T; type IntoFuture = Task<T>; ... }
```

`JoinHandle<T>` 的 impl 在阻塞池（`task::spawn_blocking`，[2.18](218-task.md)）中 join 該執行緒，因此只有進行 await 的任務會等待。只有在該型別不實作 `Future` 時，這個 impl 才合法，以免與全面實作（blanket impl）重疊（[refs-lang 2.9.5](../../refs-lang/002-types/209-impl.md#一致性)）。

`await expr` 先呼叫 `IntoFuture::into_future(expr)`，再對結果反覆呼叫 `Future::poll`（2.20.3）。這與 [2.8 stream](208-stream.md) 中 `for await` 處理 `IntoStream` 的方式相同。

只有 `future_trait` 登記為 lang 槽位；`IntoFuture` 是普通的 trait。編譯器在 `await` 的去語法糖中直接呼叫 `IntoFuture::into_future`，因此與 `IntoStream` 一樣，不需要另外的 lang 槽位。

## 2.20.6. async / await 的「函式顏色」

JavaScript 與 Python 有著名的**函式顏色**問題：非同步函式與同步函式不能自由混用（await 只能在 async 內使用，而同步函式中無法進行 async 呼叫）。

Nabi 的方針：

- `async fn` 與 `await` 讓顏色明確
- `for` 與 `for await` 讓顏色明確
- 不嘗試統一顏色。JS、Python、C# 和 Swift 都遵循相同的模型

顏色分開是為了保證**與排程器的合作**。只有明確寫出「這個位置可以暫停」，編譯器才能建構協程狀態機。

```nabi
// await inside a sync fn: compile error
fn sync_caller() {
    let v = await some_future;       // ERROR: await is allowed only inside an async function
}

async fn async_caller() {
    let v = await some_future;       // OK
}
```

### 在同步上下文中取得 future 的結果

要在同步上下文中取得 future 的結果，使用 `block_on`。

```nabi
let v = task::block_on(some_future);    // the current thread attaches to the executor and runs it to completion
```

`block_on` 只能在最上層（main 或同步呼叫端）使用。在巢狀的非同步程式碼中使用 `block_on` 有死結的風險。

## 2.20.7. `for await`：消費 Stream

Stream 以 `for await` 形式消費。

```nabi
for await event in stream {            // consumes Stream<Item = T> (async)
    handle(event);
}

for x in iterator {                    // consumes Iterator<Item = T> (sync)
    process(x);
}
```

`for await` 的權威定義是 [2.8 stream](208-stream.md) 中的 `Stream` trait。

## 2.20.8. `async yield`：產生 Stream

反過來，Stream 以 `async yield` 產生。

```nabi
async yield fn ticks(interval: Duration) -> Instant {
    loop {
        await task::sleep(interval);
        yield Instant::now();
    }
}

for await t in ticks(Duration::ms(100)) {
    log(f"tick at {t}");
}
```

`async yield` 結合了 async 與 yield，產生一個 Stream。語法的權威定義是 [refs-lang 4.13 stream](../../refs-lang/004-functions/413-stream.md)。

## 2.20.8a. `AsyncFuture<T>` / `TryAsyncFuture<T, E>`：非同步函式的值

這些是 `async fn` 呼叫與 `async { .. }` 區塊的值（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#協程函式的結果型別)）。

```nabi
struct AsyncFuture<T> { /* private */ }
impl<T> Future for AsyncFuture<T> {
    type Output = T;
    fn poll(ref self, ref ctx: Context) -> Poll<T>;   // advances the body to the next wait point
}

struct TryAsyncFuture<T, E> { /* private */ }    // async fn .. throws (E)
impl<T, E> TryAsyncFuture<T, E> {
    fn poll(ref self, ref ctx: Context) throws (E) -> Poll<T>;
}
```

`TryAsyncFuture` 不實作 `Future`，因為它的 `poll` 會擲回錯誤；它以 `try await` 進行 await。私有欄位不予規定。複本共用同一個協程框架，因此共用推進的位置。

## 2.20.9. 模組對應

各項目的模組位置與 lang 槽位 / 角色；每個項目只有那一個路徑（[0.1.6](../000-meta/001-tier-model.md#016-每個項目只有一條路徑)）：

| 項目             | 模組              | 角色                        |
|------------------|---------------------|-----------------------------|
| `Future`         | `core::future` | `#[lang("future_trait")]`   |
| `Poll<T>`        | `core::future` | 列舉 Ready/Pending          |
| `Context`        | `core::future` | 持有喚醒器                |
| `Waker`          | `core::future` | 排程器回呼          |
| `IntoFuture`     | `core::future` | 普通 trait（無 lang 槽位） |
| `AsyncFuture<T>` | `core::future` | `async fn` / `async { }` 的值 |
| `TryAsyncFuture<T, E>` | `core::future` | 會擲回錯誤的 `async fn` 的值 |
| `task::block_on` | `core::task`   | 同步進入點            |

## 2.20.10. 與其他語言的比較

各語言的非同步模型比較如下。

| 面向               | Rust                    | C#                 | Swift                  | Nabi                               |
|----------------------|-------------------------|--------------------|------------------------|-----------------------------------|
| 模型                | 基於輪詢              | 回呼（狀態）   | continuation           | 基於輪詢（與 Rust 同構）   |
| `async fn`           | ✓                       | ✓                  | ✓                      | ✓                                 |
| `await`              | 後置 `.await`        | 前置 `await`     | 前置 `await`         | 前置 `await`                    |
| 協程實作 | 編譯器產生  | 編譯器產生 | 編譯器產生     | 編譯器狀態機            |
| 無堆疊            | ✓                       | ✓                  | ✓                      | ✓                                 |
| 顏色統一    | ✗（同步/非同步分離）    | ✗                  | ✗                      | ✗（採用業界標準）  |
| await 泛化 | `Future` + 外部函式庫 | 統一的 `Task<T>` | `Task` + AsyncSequence | 透過 `IntoFuture` 泛化      |

Nabi 的差異主要有兩點。

1. **前置 `await`**：與 C#、Swift 和 JS 相同；權威定義在 [refs-lang 4.12](../../refs-lang/004-functions/412-async.md)。要在結果上串接呼叫，用括號包起來（`(await x).method()`）。
2. 普通 trait `IntoFuture` 讓 `Task` 與 `JoinHandle` 能以一致的方式被 await。這個想法與 Swift 的 `Task` 統一相似，不同之處在於保留各自的型別，並透過 trait 泛化。
