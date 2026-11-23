# 2.8. stream：非同步迭代器

`core::stream` 模組定義 `Stream`，即非同步步進的 trait（其元素型別是關聯型別 `Item`）。[2.7 iter](207-iter.md) 的 `Iterator` 是**同步**的，而 `Stream` 是**非同步**的：每次 `next()` 呼叫都**可以 await**。

本模組也是 `for await x in stream` 去語法糖（desugaring）的權威來源，該語法分派到本章定義的 trait。

## 2.8.1. `Stream`：非同步步進

此 trait 定義如下。

```nabi
trait Stream {
    type Item;

    fn poll_next(ref self, ref ctx: Context) -> Poll<Item | none>;
}
```

這是 `Iterator::next` 的非同步版本。就回傳值而言，`Poll::Ready(v)`（`v: Item`）表示下一個元素 `v` 已就緒；`Poll::Ready(none)` 表示串流結束；`Poll::Pending` 表示尚無就緒的值，待 Waker 喚醒串流時會再次輪詢它。

簡言之，它與 `Iterator` 的 `next() -> T | none` 意義相同，決定性的差異在於它**可以 await**。

## 2.8.2. `for await` 去語法糖

考慮下列程式碼。

```nabi
for await x in stream { body }
```

概念上它會去語法糖為：

```nabi
loop {
    match await stream.poll_next(ref CURRENT_CONTEXT) {
        v: Item => { let x = v; body; }
        none    => break,
    }
}
```

這與 [2.7.2 for 去語法糖](207-iter.md)中的同步版本結構相同，只多了 `await`。

## 2.8.3. `IntoStream`：非同步可迭代物件

接下來是表示非同步可迭代物件的 trait。

```nabi
trait IntoStream {
    type Item;
    type IntoStream: Stream<Item = Item>;

    fn into_stream(self) -> IntoStream;
}

// every stream is its own IntoStream (as IntoIterator, 2.7)
impl<S: Stream> IntoStream for S {
    type Item = S::Item;
    type IntoStream = S;
    fn into_stream(self) -> S { self }
}
```

`for await` 開始時會呼叫 `IntoStream::into_stream`，然後反覆呼叫 `Stream::poll_next`。具體的轉換如下：

```nabi
for await event in event_source { ... }
// = let __s = event_source.into_stream();
//   loop {
//       match await __s.poll_next(ref CTX) {
//           e: Item => { let event = e; body; }
//           none    => break,
//       }
//   }
```

## 2.8.4. `async yield`：建立 Stream

建立 Stream 最直接的方式是撰寫 `async yield` 函式。

```nabi
async yield fn ticks(interval: Duration) -> Instant {
    loop {
        await task::sleep(interval);
        yield Instant::now();
    }
}
```

`-> Instant` 是元素型別，呼叫的結果是 `AsyncStream<Instant>`（實作 `Stream<Item = Instant>`）（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#協程函式的結果型別)）。編譯器會把此函式轉成狀態機（[refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)）。每個 `yield` 都是**暫停點**，每個 `await` 也是。

語法的權威來源是 [refs-lang 4.13 stream（`async yield`）](../../refs-lang/004-functions/413-stream.md)。

## 2.8.5. Stream 配接器

與 `Iterator` 一樣，一系列配接器以 `Stream` trait 的預設方法提供，因此每個實作 `Stream` 的型別（`AsyncStream<T>`、`Receiver<T>`、使用者型別）都擁有相同的配接器。

```nabi
trait Stream {
    type Item;
    fn poll_next(ref self, ref ctx: Context) -> Poll<Item | none>;

    // default methods
    fn map<U>(self, f: fn(Item) -> U) -> Map<Self, U>;
    fn filter(self, pred: fn(Item) -> bool) -> Filter<Self>;
    fn take(self, n: usize) -> Take<Self>;
    fn skip(self, n: usize) -> Skip<Self>;

    async fn collect<C: FromStream<Item> = List<Item>>(self) -> C;   // List<Item> unless the slot decides
    async fn for_each(self, f: fn(Item));

    // async-specific
    fn buffer_unordered(self, n: usize) -> BufferUnordered<Self> where Item: Future;
    fn buffered(self, n: usize) -> Buffered<Self> where Item: Future;
    fn throttle(self, dur: Duration) -> Throttle<Self>;
}
```

與 [2.7 iter](207-iter.md) 的配接器一樣，回傳型別是具名的具體結構，位於 `core::stream` 模組（`stream::Map<S, U>`、`stream::Filter<S>`……）。`Map<Self, U>` 實作 `Stream<Item = U>`，`Buffered<Self>` / `BufferUnordered<Self>` 實作 `Stream<Item = Item::Output>`（它們產出 future 的輸出，而不是 future 本身）；其餘實作 `Stream<Item = Item>`。

> [!NOTE]
> `stream::Map` / `Filter` / `Take` / `Skip` / `Zip` 與 `iter::Map` 等同名。兩者都不在 [prelude](../000-meta/002-import.md) 中，因此需要寫出型別時，請使用 `stream::Map` 這類路徑，或以明確的 `use` 匯入。

其中，`buffer_unordered(n)` 與 `buffered(n)` 用於並行控制，可同時處理 N 個 future。

## 2.8.6. `FromStream<T>`：`collect` 的分派

負責 `collect` 分派的 trait 是 `FromStream<T>`。它是 [2.7 iter](207-iter.md) 中 `FromIterator<T>` 的非同步對應物；其簽章形式相同，只是 `from_stream` 可以 await。

```nabi
trait FromStream<T> {
    async fn from_stream<S: IntoStream<Item = T>>(stream: S) -> Self;
}

impl<T> FromStream<T> for List<T> { ... }
impl<T: Hash> FromStream<T> for Set<T> { ... }
impl<K: Hash, V> FromStream<(K, V)> for Dict<K, V> { ... }
```

`collect` 的目標由接收的槽位決定；若槽位未決定，則為 `List<Item>`。

呼叫 `Stream::collect` 時，會呼叫目標集合型別的 `from_stream`，串流的每個元素都會被 await 並累積到該集合中。

```nabi
async yield fn nums() -> i32 { for x in 0..5 { yield x; } }

let v: List<i32> = await nums().map(|x| async { x * 2 }).buffered(3).collect();
//   = await List::from_stream(...), [0, 2, 4, 6, 8]
```

`List`、`Dict` 與 `Set` 同時實作 `FromIterator` 與 `FromStream`，在同步與非同步程式碼中遵循相同的模式。

## 2.8.7. `Channel` ↔ `Stream`

`Receiver<T>`（通道）實作 `Stream<Item = T>`，因此可以把通道接收直接當作串流處理。

```nabi
let (tx, rx) = channel::<Event>();
task::spawn(async { try! tx.send(event1); try! tx.send(event2); });

for await event in rx {
    handle(event);
}
```

也就是說，[2.17 channel](217-channel.md) 的接收端被視為**自動串流**，與 `for await` 相容。

## 2.8.8. `merge` / `zip` 等

也提供組合多個串流的組合子。

```nabi
fn merge<S1: Stream, S2: Stream>(a: S1, b: S2) -> Merge<S1, S2>;    // Stream<Item = S1::Item | S2::Item>
fn zip<S1: Stream, S2: Stream>(a: S1, b: S2) -> Zip<S1, S2>;        // Stream<Item = (S1::Item, S2::Item)>
fn select<S1: Stream, S2: Stream>(a: S1, b: S2) -> Select<S1, S2>;  // Stream<Item = S1::Item | S2::Item>, ends when either ends
```

`merge` 在兩個串流的項目一就緒時就產出它們，並在兩者都結束時結束。`select` 是任一串流結束就立即結束的 `merge`。`zip` 把兩個串流配對。`merge` 與 `select` 的每次輪詢都先嘗試第一個串流，再嘗試第二個，因此若第一個串流始終就緒，第二個串流就會陷入飢餓。

> [!NOTE]
> `select` 與 `task::select` 同名；兩者都不在 [prelude](../000-meta/002-import.md) 中：請寫成 `stream::select` 或匯入它。

## 2.8.8a. `AsyncStream<T>` / `TryAsyncStream<T, E>`：非同步串流的值

這兩者是 `async yield fn` 呼叫與 `async yield { .. }` 區塊的值（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#協程函式的結果型別)）。

```nabi
struct AsyncStream<T> { /* private */ }
impl<T> Stream for AsyncStream<T> {
    type Item = T;
    fn poll_next(ref self, ref ctx: Context) -> Poll<T | none>;
}

struct TryAsyncStream<T, E> { /* private */ }    // async yield fn .. throws (E)
impl<T, E> TryStream for TryAsyncStream<T, E> {
    type Item = T;
    type Error = E;
    fn poll_next(ref self, ref ctx: Context) throws (E) -> Poll<T | none>;
}
```

`AsyncStream<T>` 的配接器（`map`、`filter`、`collect` 等）是 `Stream` 的預設方法（2.8.5）。`TryAsyncStream` 是 `TryStream`（[2.8.11](#2811-trystream可能失敗的串流)），而不是 `Stream`，因為它的 `poll_next` 會擲回錯誤；它以 `try for await` 或 `TryStream` 配接器消費。私有欄位不作規定。複本共用同一個協程框架，因此共用進度位置。

## 2.8.9. 模組對應

每個項目都定義在 `core::stream`（`Receiver<T>` 則在 `core::channel`），這是它唯一的路徑（[0.1.6](../000-meta/001-tier-model.md#016-每個項目只有一條路徑)）；沒有子模組。

| 項目                          | 模組               | 形式                         |
|-------------------------------|----------------------|------------------------------|
| `Stream`                      | `core::stream`  | 基於輪詢                   |
| `IntoStream`                  | `core::stream`  |                              |
| `Receiver<T>`                 | `core::channel` | 實作 `Stream<Item = T>` |
| `Map`/`Filter`/`Take`/`Skip`/`Throttle`/`Buffered`/`BufferUnordered` | `core::stream` | 配接器結構（實作 `Stream`） |
| `merge`/`zip`/`select`        | `core::stream`  | `Merge`/`Zip`/`Select` 結構 |
| `TryStream`、`from_results`、`TryMap`/`TryFilter`/`MapErr`/`TryTake`/`TrySkip`/`Results`/`FromResults` | `core::stream` | 可能失敗的串流（2.8.11） |

## 2.8.10. `Stream` 是否有 lang 槽位

本章不在 `Stream::poll_next` 上附加 `#[lang(...)]`；`Stream` 是一般的 trait。

理由：`for await` 的去語法糖在編譯器中寫死為**直接**呼叫 `Stream::poll_next`，因此不需要 lang 槽位即可辨識。此外，lang 槽位是為多個分派進入點而設的機制，而 Stream 只有單一進入點，因此不需要。

## 2.8.11. `TryStream`：可能失敗的串流

`TryIterator`（[2.7.13](207-iter.md#2713-tryiterator可能失敗的迭代)）的非同步對應物：`poll_next` 給出一個項目、在結束時給出 `none`，或擲回 `Error`。

```nabi
trait TryStream {
    type Item;
    type Error;
    fn poll_next(ref self, ref ctx: Context) throws (Error) -> Poll<Item | none>;

    fn map<U>(self, f: fn(Item) throws (Error) -> U) -> TryMap<Self, U>;
    fn filter(self, pred: fn(Item) throws (Error) -> bool) -> TryFilter<Self>;
    fn map_err<E2>(self, f: fn(Error) -> E2) -> MapErr<Self, E2>;
    fn take(self, n: usize) -> TryTake<Self>;
    fn skip(self, n: usize) -> TrySkip<Self>;
    fn results(self) -> Results<Self>;              // Stream<Item = Result<Item, Error>>

    async fn collect<C: FromStream<Item> = List<Item>>(self) throws (Error) -> C;
    async fn for_each(self, f: fn(Item) throws (Error)) throws (Error);
    async fn fold<B>(self, init: B, f: fn(B, Item) throws (Error) -> B) throws (Error) -> B;
}

fn from_results<S: Stream<Item = Result<T, E>>, T, E>(s: S) -> FromResults<S, T, E>;   // TryStream
```

2.7.13 的規則同樣適用：配接器的函式可以擲回 `Error`，只有 `map_err` 會改變它，`collect` 在第一個錯誤處停止並擲回該錯誤，`results()` 在錯誤之後仍會繼續輪詢。非同步方法與 `Stream::collect` 一樣接受 `self`。`TryStream` 不在 prelude 中（`Stream` 也不在）；請寫成 `stream::TryStream`。

`try for await x in s { .. }` 步進一個 `TryStream`；`try` 涵蓋串流運算式與每次 `poll_next`，而不涵蓋迴圈主體（[refs-lang 3.13.3](../../refs-lang/003-expressions/313-loop-expressions.md#3133-for以迭代器為基礎的重複)）。

```nabi
async fn sizes(urls: List<str>) throws (HttpError) -> List<usize> {
    try await fetch_all(urls).map(|body| body.len()).collect()
}
```
