# 2.20. future：`Future` / `await` 集成

`core::future` 模块提供异步机制的权威 trait。`await expr` 的脱糖降级为对本章所定义的 `Future` trait 的分派。本章还定义了 lang 槽之一 `future_trait`（[0.4](../000-meta/004-lang-slot-table.md)）。

## 2.20.1. `Future`：异步结果

`Future` 是表示异步计算结果的核心 trait；结果类型是关联类型 `Output`。

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

`poll(ctx)` 有两种结果。`Ready(v)` 表示结果已就绪，`v` 原样返回；`Pending` 表示还没有结果，当 `ctx` 中的唤醒器唤醒该 future 时会再次轮询它。

这个模型与 Rust 的 `Future` 和 `Poll`（Nabi 的是 `core::future::Poll`）同构，并编译为**无栈协程**（编译器生成的状态机，[refs-lang 7.8](../../refs-lang/007-runtime/708-coroutine.md)）。

## 2.20.2. `Context` / `Waker`

`Context` 和 `Waker` 构成轮询与调度器之间的通信渠道。

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

`poll` 返回 `Pending` 时，会在内部注册一个 Waker；之后发生事件（I/O 或定时器）时调用该唤醒器，任务被重新调度。大多数用户从不直接处理 `Waker`，因为 `await` 会自动处理这些。

`waker` 字段是库私有的：手写的 `poll` 用 `ctx.waker()` 读取它，保存一个克隆，并在能够推进时调用 `wake`。`Context::from_waker` 用已有的唤醒器构造上下文，以便在同一个唤醒器下手动轮询内部 future。`Waker` 本身只由执行器创建。

## 2.20.3. `await expr` 脱糖

最常见的用法：

```nabi
let v = await some_future;
```

概念上它脱糖为：

```nabi
let __f = IntoFuture::into_future(some_future);
let v = loop {
    match __f.poll(ref CURRENT_CONTEXT) {
        Ready(v)  => break v,
        Pending   => yield_to_scheduler(),
    }
};
```

编译器把它变成状态机，每个 `await` 成为一个**挂起点**。只有帧变量会被保存，存放在位于 GC 堆上的协程状态中。

## 2.20.4. `async fn` / `async { }` / `async || { }`

三种形式都返回 **`AsyncFuture<T>`**（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#result-types-of-coroutine-functions)）。

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

这**三种形式**的权威定义是 [refs-lang 4.12 async](../../refs-lang/004-functions/412-async.md)。

## 2.20.5. `IntoFuture`：可等待的值

`IntoFuture` 是 await 的统一入口。不仅是 `Future`，像 `JoinHandle<T>`（[3.4](../300-std/304-thread.md)）这样本身不是 `Future` 的句柄也能通过这个 trait 被 `await`。

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

`JoinHandle<T>` 的实现在阻塞线程池中 join 该线程（`task::spawn_blocking`，[2.18](218-task.md)），所以只有正在等待的任务会等待。只要该类型不实现 `Future`，这个实现就是合法的，从而不会与通用实现重叠（[refs-lang 2.9.5](../../refs-lang/002-types/209-impl.md#coherence)）。

`await expr` 先调用 `IntoFuture::into_future(expr)`，然后对结果反复调用 `Future::poll`（2.20.3）。这与 [2.8 stream](208-stream.md) 中 `for await` 处理 `IntoStream` 的方式相同。

只有 `future_trait` 注册为 lang 槽；`IntoFuture` 是普通 trait。编译器在 `await` 脱糖中直接调用 `IntoFuture::into_future`，所以与 `IntoStream` 一样，不需要单独的 lang 槽。

## 2.20.6. async / await 的“函数颜色”

JavaScript 和 Python 有著名的**函数颜色**问题：async 函数与同步函数不能自由混用（只能在 async 内 await，同步函数中无法进行 async 调用）。

Nabi 的方针：

- `async fn` 和 `await` 让颜色显式可见
- `for` 和 `for await` 让颜色显式可见
- 不尝试统一颜色。JS、Python、C# 和 Swift 都遵循同样的模型

区分颜色是为了保证**调度器协作**。只有显式声明“此处可以挂起”，编译器才能构建协程状态机。

```nabi
// await inside a sync fn: compile error
fn sync_caller() {
    let v = await some_future;       // ERROR: await is allowed only inside an async function
}

async fn async_caller() {
    let v = await some_future;       // OK
}
```

### 在同步上下文中获取 future 的结果

要在同步上下文中获取 future 的结果，使用 `block_on`。

```nabi
let v = task::block_on(some_future);    // the current thread attaches to the executor and runs it to completion
```

`block_on` 只能在顶层（main 或同步调用方）使用。在嵌套的异步代码中使用 `block_on` 有死锁的风险。

## 2.20.7. `for await`：消费 Stream

Stream 用 `for await` 形式消费。

```nabi
for await event in stream {            // consumes Stream<Item = T> (async)
    handle(event);
}

for x in iterator {                    // consumes Iterator<Item = T> (sync)
    process(x);
}
```

`for await` 的权威定义是 [2.8 stream](208-stream.md) 中的 `Stream` trait。

## 2.20.8. `async yield`：生成 Stream

反过来，Stream 用 `async yield` 生成。

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

`async yield` 结合了 async 与 yield，生成一个 Stream。语法的权威定义是 [refs-lang 4.13 stream](../../refs-lang/004-functions/413-stream.md)。

## 2.20.8a. `AsyncFuture<T>` / `TryAsyncFuture<T, E>`：异步函数的值

它们是 `async fn` 调用和 `async { .. }` 块的值（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#result-types-of-coroutine-functions)）。

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

`TryAsyncFuture` 不实现 `Future`，因为它的 `poll` 会抛出错误；它用 `try await` 等待。私有字段不作规定。副本共享同一个协程帧，因此共享推进位置。

## 2.20.9. 模块映射

各项所在模块及其 lang 槽 / 作用；每一项都只有这一条路径（[0.1.6](../000-meta/001-tier-model.md#016-每个项只有一条路径)）：

| 项               | 模块                | 作用                        |
|------------------|---------------------|-----------------------------|
| `Future`         | `core::future` | `#[lang("future_trait")]`   |
| `Poll<T>`        | `core::future` | 枚举 Ready/Pending          |
| `Context`        | `core::future` | 持有唤醒器                  |
| `Waker`          | `core::future` | 调度器回调                  |
| `IntoFuture`     | `core::future` | 普通 trait（无 lang 槽）    |
| `AsyncFuture<T>` | `core::future` | `async fn` / `async { }` 的值 |
| `TryAsyncFuture<T, E>` | `core::future` | 会抛出错误的 `async fn` 的值 |
| `task::block_on` | `core::task`   | 同步入口点                  |

## 2.20.10. 与其他语言的比较

各语言的异步模型比较如下。

| 方面                 | Rust                    | C#                 | Swift                  | Nabi                              |
|----------------------|-------------------------|--------------------|------------------------|-----------------------------------|
| 模型                 | 基于轮询                | 回调（状态）       | 续体（continuation）   | 基于轮询（与 Rust 同构）          |
| `async fn`           | ✓                       | ✓                  | ✓                      | ✓                                 |
| `await`              | 后缀 `.await`           | 前缀 `await`       | 前缀 `await`           | 前缀 `await`                      |
| 协程实现             | 编译器生成              | 编译器生成         | 编译器生成             | 编译器状态机                      |
| 无栈                 | ✓                       | ✓                  | ✓                      | ✓                                 |
| 颜色统一             | ✗（同步/异步分离）      | ✗                  | ✗                      | ✗（采用业界标准）                 |
| await 泛化           | `Future` + 外部库       | 统一的 `Task<T>`   | `Task` + AsyncSequence | 通过 `IntoFuture` 泛化            |

Nabi 主要有两点不同。

1. **前缀 `await`**：与 C#、Swift 和 JS 相同；权威定义是 [refs-lang 4.12](../../refs-lang/004-functions/412-async.md)。要在结果上链式调用，用括号包起来（`(await x).method()`）。
2. 普通 trait `IntoFuture` 让 `Task` 和 `JoinHandle` 能以统一方式被等待。这个思路与 Swift 的 `Task` 统一类似，但不同之处在于保留各自不同的类型，并通过 trait 实现泛化。
