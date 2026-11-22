# 2.8. stream：异步迭代器

`core::stream` 模块定义 `Stream`，即异步逐步取值的 trait（其元素类型为关联类型 `Item`）。[2.7 iter](207-iter.md) 的 `Iterator` 是**同步**的，而 `Stream` 是**异步**的：每次 `next()` 调用都**可以 await**。

本模块也是 `for await x in stream` 脱糖（desugaring）的权威定义，该语法分派到本章定义的 trait。

## 2.8.1. `Stream`：异步逐步取值

该 trait 定义如下。

```nabi
trait Stream {
    type Item;

    fn poll_next(ref self, ref ctx: Context) -> Poll<Item | none>;
}
```

这是 `Iterator::next` 的异步版本。返回值中，`Poll::Ready(v)`（`v: Item`）表示下一个元素 `v` 已就绪；`Poll::Ready(none)` 表示流结束；`Poll::Pending` 表示还没有就绪的值，唤醒器（Waker）唤醒该流时会再次轮询它。

简而言之，它与 `Iterator` 的 `next() -> T | none` 含义相同，决定性的区别在于它**可以 await**。

## 2.8.2. `for await` 脱糖

考虑下面的代码。

```nabi
for await x in stream { body }
```

从概念上讲，它脱糖为：

```nabi
loop {
    match await stream.poll_next(ref CURRENT_CONTEXT) {
        v: Item => { let x = v; body; }
        none    => break,
    }
}
```

它与 [2.7.2 for 脱糖](207-iter.md) 中的同步版本形状相同，只是加了 `await`。

## 2.8.3. `IntoStream`：异步可迭代对象

下面是表示异步可迭代对象的 trait。

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

`for await` 开始时调用 `IntoStream::into_stream`，然后重复调用 `Stream::poll_next`。具体的变换如下：

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

## 2.8.4. `async yield`：创建 Stream

创建 Stream 最直接的方式是编写 `async yield` 函数。

```nabi
async yield fn ticks(interval: Duration) -> Instant {
    loop {
        await task::sleep(interval);
        yield Instant::now();
    }
}
```

`-> Instant` 是元素类型，调用的结果是 `AsyncStream<Instant>`（实现 `Stream<Item = Instant>`）（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#result-types-of-coroutine-functions)）。编译器把这个函数变换为状态机（[refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)）。每个 `yield` 都是一个**挂起点**，每个 `await` 也是。

语法的权威定义是 [refs-lang 4.13 stream（`async yield`）](../../refs-lang/004-functions/413-stream.md)。

## 2.8.5. Stream 适配器

与 `Iterator` 一样，一系列适配器作为 `Stream` trait 的默认方法提供，因此每个实现 `Stream` 的类型（`AsyncStream<T>`、`Receiver<T>`、用户类型）都有相同的适配器。

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

与 [2.7 iter](207-iter.md) 的适配器一样，返回类型是具名的具体结构体，位于 `core::stream` 模块（`stream::Map<S, U>`、`stream::Filter<S>` 等）。`Map<Self, U>` 实现 `Stream<Item = U>`，`Buffered<Self>` / `BufferUnordered<Self>` 实现 `Stream<Item = Item::Output>`（产出的是 future 的输出，而不是 future 本身）；其余实现 `Stream<Item = Item>`。

> [!NOTE]
> `stream::Map` / `Filter` / `Take` / `Skip` / `Zip` 与 `iter::Map` 等同名。两者都不在 [prelude](../000-meta/002-import.md) 中，因此需要写出类型时，使用 `stream::Map` 这样的路径，或用显式的 `use` 导入。

其中，`buffer_unordered(n)` 和 `buffered(n)` 用于并发控制，同时处理 N 个 future。

## 2.8.6. `FromStream<T>`：`collect` 的分派

负责 `collect` 分派的 trait 是 `FromStream<T>`。它是 [2.7 iter](207-iter.md) 中 `FromIterator<T>` 的异步对应物，签名形式相同，只是 `from_stream` 可以 await。

```nabi
trait FromStream<T> {
    async fn from_stream<S: IntoStream<Item = T>>(stream: S) -> Self;
}

impl<T> FromStream<T> for List<T> { ... }
impl<T: Hash> FromStream<T> for Set<T> { ... }
impl<K: Hash, V> FromStream<(K, V)> for Dict<K, V> { ... }
```

`collect` 的目标由接收它的槽决定；槽不能决定时，目标为 `List<Item>`。

调用 `Stream::collect` 时，会调用目标集合类型的 `from_stream`，流的每个元素都被 await 并累积到该集合中。

```nabi
async yield fn nums() -> i32 { for x in 0..5 { yield x; } }

let v: List<i32> = await nums().map(|x| async { x * 2 }).buffered(3).collect();
//   = await List::from_stream(...), [0, 2, 4, 6, 8]
```

`List`、`Dict` 和 `Set` 同时实现 `FromIterator` 和 `FromStream`，同步代码和异步代码遵循相同的模式。

## 2.8.7. `Channel` ↔ `Stream`

`Receiver<T>`（通道）实现 `Stream<Item = T>`，因此通道的接收可以直接当作流处理。

```nabi
let (tx, rx) = channel::<Event>();
task::spawn(async { try! tx.send(event1); try! tx.send(event2); });

for await event in rx {
    handle(event);
}
```

也就是说，[2.17 channel](217-channel.md) 的接收端被视为**自动流**，可以与 `for await` 配合使用。

## 2.8.8. `merge` / `zip` 等

本模块也提供组合多个流的组合子。

```nabi
fn merge<S1: Stream, S2: Stream>(a: S1, b: S2) -> Merge<S1, S2>;    // Stream<Item = S1::Item | S2::Item>
fn zip<S1: Stream, S2: Stream>(a: S1, b: S2) -> Zip<S1, S2>;        // Stream<Item = (S1::Item, S2::Item)>
fn select<S1: Stream, S2: Stream>(a: S1, b: S2) -> Select<S1, S2>;  // Stream<Item = S1::Item | S2::Item>, ends when either ends
```

`merge` 按就绪顺序产出两个流的项，两个流都结束时才结束。`select` 是任一流结束就立即结束的 `merge`。`zip` 把两个流配对。`merge` 和 `select` 的每次轮询都先尝试第一个流，再尝试第二个流，因此一个始终就绪的第一个流会使第二个流饥饿。

> [!NOTE]
> `select` 与 `task::select` 同名；两者都不在 [prelude](../000-meta/002-import.md) 中：写 `stream::select` 或将其导入。

## 2.8.8a. `AsyncStream<T>` / `TryAsyncStream<T, E>`：异步流的值

它们是 `async yield fn` 调用和 `async yield { .. }` 块的值（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#result-types-of-coroutine-functions)）。

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

`AsyncStream<T>` 的适配器（`map`、`filter`、`collect` 等）是 `Stream` 的默认方法（2.8.5）。`TryAsyncStream` 是 `TryStream`（[2.8.11](#2811-trystream可失败的流)），而不是 `Stream`，因为它的 `poll_next` 会抛出错误；它通过 `try for await` 或 `TryStream` 适配器消费。私有字段不作规定。副本共享同一个协程帧，因此也共享前进位置。

## 2.8.9. 模块对应关系

每个项都定义在 `core::stream` 中（`Receiver<T>` 定义在 `core::channel` 中），这是它唯一的路径（[0.1.6](../000-meta/001-tier-model.md#016-每个项只有一条路径)）；没有子模块。

| 项                            | 模块                 | 形式                         |
|-------------------------------|----------------------|------------------------------|
| `Stream`                      | `core::stream`  | 基于轮询                     |
| `IntoStream`                  | `core::stream`  |                              |
| `Receiver<T>`                 | `core::channel` | 实现 `Stream<Item = T>` |
| `Map`/`Filter`/`Take`/`Skip`/`Throttle`/`Buffered`/`BufferUnordered` | `core::stream` | 适配器结构体（实现 `Stream`） |
| `merge`/`zip`/`select`        | `core::stream`  | `Merge`/`Zip`/`Select` 结构体 |
| `TryStream`、`from_results`、`TryMap`/`TryFilter`/`MapErr`/`TryTake`/`TrySkip`/`Results`/`FromResults` | `core::stream` | 可失败的流（2.8.11） |

## 2.8.10. `Stream` 是否有 lang 槽

本章不给 `Stream::poll_next` 附加任何 `#[lang(...)]`；`Stream` 是普通的 trait。

原因是：`for await` 脱糖在编译器中写死为**直接**调用 `Stream::poll_next`，因此无需 lang 槽即可识别。此外，lang 槽是为多个分派入口点准备的机制，而 Stream 只有一个入口点，所以不需要。

## 2.8.11. `TryStream`：可失败的流

它是 `TryIterator`（[2.7.13](207-iter.md#2713-tryiterator可能失败的迭代)）的异步对应物：`poll_next` 给出一个项，在结束时给出 `none`，或者抛出 `Error`。

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

2.7.13 的规则同样适用：适配器的函数可以抛出 `Error`，只有 `map_err` 会改变它，`collect` 在第一个错误处停止并抛出该错误，`results()` 在出错后继续轮询。异步方法与 `Stream::collect` 一样接收 `self`。`TryStream` 不在 prelude 中（`Stream` 也不在）；写 `stream::TryStream`。

`try for await x in s { .. }` 逐步遍历 `TryStream`；`try` 覆盖流表达式和每次 `poll_next`，不覆盖循环体（[refs-lang 3.13.3](../../refs-lang/003-expressions/313-loop-expressions.md#3133-for-iterator-based-repetition)）。

```nabi
async fn sizes(urls: List<str>) throws (HttpError) -> List<usize> {
    try await fetch_all(urls).map(|body| body.len()).collect()
}
```
