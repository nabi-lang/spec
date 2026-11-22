# 3.4. `thread::`：操作系统线程（1:1 模型）

`std::thread` 模块是操作系统级线程的入口。正如 [2.18 task](../200-core/218-task.md) 所述，它与运行在 M:N 协作式调度器上的任务是两种不同的抽象；本模块提供直接启动原生线程的途径。

## 3.4.1. 何时用 thread，何时用 task

下表说明在什么情况下选择哪一种。

| 情况                           | 建议                                        |
|--------------------------------|---------------------------------------------|
| 一般的并发 / 异步 I/O          | `task::spawn`（轻量）                       |
| CPU 密集型计算                 | `task::spawn` 或 `par::*`（工作窃取）       |
| 阻塞的 C 库调用                | `thread::spawn`（避免阻塞任务调度器）       |
| 需要操作系统线程 API 的地方    | `thread::spawn`（TLS、信号掩码等）          |
| 嵌入式 / 无任务                | `thread::spawn`                             |

简而言之，凡是任务足够轻量、表达力足够的地方都优先使用任务；只有必须直接处理操作系统资源的地方才下沉到线程。

## 3.4.2. API

基本类型和函数签名如下。

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

`Thread::unpark` 设置该线程的 park 令牌并唤醒它。一个令牌由一次 `park` 消耗。

`thread::spawn` 就是套在 `try!` 后面、使用默认设置的 `Builder::spawn`：当 `Builder::spawn` 抛出错误时（目标上没有线程，或操作系统拒绝创建新线程），`thread::spawn` 会触发 panic。要处理该错误，就使用 `Builder::new().spawn(f)`。

`await handle`（[2.20.5](../200-core/220-future.md#2205-intofuture---awaitable-values)）在任务内部等待线程：join 在执行器的阻塞通道（[`task::spawn_blocking`](../200-core/218-task.md#2181-core-api)）上运行，因此只有该任务在等待。它产出 `T`；操作系统层面的 join 失败（`JoinError`）会触发 panic，所以要处理它就调用 `join`。

### Panic 与 `JoinError`

任何线程中的 panic 都会以退出码 101 终止整个进程。因此 `join` 永远不会收到另一个线程的 panic，而 `thread::panicking()` 只对进程结束期间在发生 panic 的线程上运行的代码（嵌入方的退出钩子）为 true。`JoinError` 表示操作系统层面的 join 失败（`JoinErrorKind::Os`），或者对以失败的 REPL 单元结束的线程表示为 `JoinErrorKind::Lost`（[refs-lang 9.7.0](../../refs-lang/009-tools/907-jit-cells.md#970-rules-of-the-cell-session)），显示为“thread join failed: the thread ended with a failed REPL cell”。

`JoinErrorKind::Taken`：该句柄或它的副本已经被 join 或 detach 过。第二次 `join` 会抛出它；第二次 `detach` 什么也不做。

## 3.4.3. 示例

从最简单的 spawn-join 模式到构建器和多次启动：

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

## 3.4.4. 线程局部存储

当每个线程都需要自己独立的存储时，使用 `thread::local`。

```nabi
let counter = thread::local(|| 0);

try thread::spawn(|| {
    counter.set(counter.get() + 1);         // only this thread's value changes
}).join();
```

其形式沿用 Rust 的 `LocalKey`。`with(f)` 把当前线程的值借给 `f`，并返回 `f` 的返回值。该值是一个限定作用域的视图，像锁守卫的值一样被追踪（[stdlib 2.16.3](../200-core/216-sync.md#2163-mutext---exclusive-lock)）：把它从 `f` 中返回、在调用之后继续保存它（捕获、字段、`Cell`）、发送它，或者把它传给会保留它的被调用方，都是编译错误 E0136；给 `with` 传入 `async` 闭包同样如此，因为该闭包的 future 会跨越 `await` 持有这个值，并可能在另一个工作线程上恢复。应当改为保留 `clone()` 或 `f` 计算出的结果。

```nabi
static SEEN: ThreadLocal<List<str>> = thread::local(|| -> List<str> { [] });

SEEN.with(|l| l.push("a"));             // changes this thread's list in place
let n = SEEN.with(|l| l.len());          // what f computes comes out
let copy = SEEN.with(|l| l.clone());     // so does a copy
let all = SEEN.take();                   // the list itself, leaving []
```

带所有权的形式把值移入和移出：`set(v)` 存入 `v`，`replace(v)` 存入 `v` 并返回旧值，`take()` 返回该值并留下 `T::default()`。`get()` 返回一个副本，要求 `T: Copy`；对其他任何 `T` 都是编译错误（请使用 `with`、`replace` 或 `take`）。`unsafe fn as_ptr()` 指向当前线程的值；该槽位在线程结束前一直保持固定，通过该指针进行的读写由调用方负责。

`with` 接受的闭包不能 `await`，因此线程的值永远不会跨越 `await` 被持有，而在 `await` 处任务可能会在另一个线程上恢复。

## 3.4.5. impl / 目标

各平台的支持情况：

| 项              | linux/macos/windows   | wasm32-wasip2 / 浏览器    | wasm32-wasip1-threads |
|-----------------|-----------------------|---------------------------|---------------------|
| `spawn`         | ✓                     | 触发 panic（`Builder::spawn` 返回 `Unsupported`） | ✓    |
| `Builder::spawn` | ✓                    | 抛出 `Unsupported`        | ✓                   |
| `local`         | ✓                     | ✓（单线程）               | ✓                   |
| `park`/`unpark` | futex / WaitOnAddress | ✓                         | ✓                   |

在 `wasm32-wasip2` 和浏览器中只有一个线程：`thread::current().name()` 为 `none`。在 `wasm32-wasip1-threads` 上，`nabi_cpu_count()` 和可用并行度都是 4，因为 WASI 无法报告核心数。
