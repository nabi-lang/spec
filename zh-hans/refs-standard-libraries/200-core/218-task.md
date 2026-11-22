# 2.18. task：M:N 调度器任务 API

`core::task` 模块提供运行在 M:N 调度器上的协作式任务 API。它借鉴了 Rust Tokio 与 Go goroutine，任务实现为工作窃取（work-stealing）调度器上的**无栈协程**。

模块 `task` 和 `Task<T>` 在 [prelude](../000-meta/002-import.md) 中，所以无需导入即可使用 `task::spawn`、`task::race`、`Task<T>` 等；模块前缀 `task::` 仍然保留。代码需要显式标明异步时，推荐写 `use core::task::*;`，让异步流程一眼可辨。

## 2.18.1. 核心 API

核心 API 的签名如下：

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

`sleep` 和 `timeout` 从第一次轮询开始计时。手写的 `Future` 通过它的 `Waker` 等待（[2.20](220-future.md)）：它保存 `ctx.waker()`，返回 `Pending`，改变状态的一方对它调用 `wake()`。

这些函数返回的 future 和错误：

```nabi
struct SelectFuture<F> { /* ... */ }
impl<F: Future> Future for SelectFuture<F> { type Output = (usize, F::Output); ... }

struct Join<T> { /* ... */ }                        // Task::join, awaited with try await
impl<T> Join<T> {
    fn poll(ref self, ref ctx: Context) throws (CancellationError) -> Poll<T>;
}

struct TimeoutError;                                // task::timeout ran out; message() and Display "timed out"
```

### 示例

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

`spawn` 接受 future 而不是闭包，所以要传入 `async` 块或 `async fn` 调用的结果，例如 `task::spawn(async { ... })`。不是 `Send` 的 future 通过 `spawn_local` 在当前工作线程（worker）上运行。`join` / `race` / `timeout` 需要 `await` 才能得到结果。

`join`、`race` 和 `select` 接受任务或 future 的任意 `IntoIterator`，包括列表字面量。

`race` 和 `timeout` 会取消落败者。`race` 一旦看到第一个任务完成（按列表顺序，因此同时完成时先列出的任务胜出），就对它收到的其他每个任务调用 `cancel`（[2.18.4](#2184-协作式取消taskcancel)）；`timeout` 的时间一到，就取消它的任务并抛出 `TimeoutError`。落败者就成为一个如同被 `cancel` 取消的任务：

- 如果它经过了取消点，`is_cancelled()` 为 true，`try await t.join()` 抛出 `CancellationError`，而普通的 `await t` 会触发 panic，消息中指明原因：``awaited a task cancelled by `task::race`; use `try await t.join()` ``（或 `task::timeout`）。
- 如果它已经完成，或者没有再经过取消点就完成了，`await t` 和 `join` 返回它自己的值。落败者永远不会产生 `race` 的结果。

两者都不等待落败者的清理：与 `select` 一样，请求取消后就返回。需要等落败者清理完成的代码写 `try await loser.join()`，它在清理之后才完成。取消正在等待 `race` 或 `timeout` 的任务，会取消交给它们的任务；在别处持有的该 `Task` 的副本能看到这次取消。对零个任务调用 `race` 会触发 panic。

### `spawn_blocking`：阻塞工作

`spawn_blocking(f)` 在阻塞线程池的一个操作系统线程上运行 `f`，而不是在执行器的工作线程上运行，并返回一个以其结果完成的 `Task<T>`，与 `spawn` 返回的句柄相同（2.18.2）。它用于在任务内等待无法按就绪状态 `await` 的阻塞系统调用（普通文件 I/O 等，[stdlib 3.1.4a](../300-std/301-fs.md#314a-异步文件-io_async)）。

- `f` 在**调用时立即**开始，而不是在第一次轮询时（线程池已满时先排队）。
- 每个进程有一个阻塞线程池。它按需增加线程（有上限），并结束空闲一段时间的线程。作业按到达顺序（FIFO）处理。上限为 512 个线程；达到上限后作业排队等待。空闲 10 秒的线程结束。每个作业交给一个空闲线程。
- `f` 没有取消点。在线程池中的线程取走作业之前调用 `cancel()`，`f` 就永远不会运行，`join` 抛出 `CancellationError`；`f` 一旦开始运行就会运行完毕，`await` / `join` 返回其结果。
- 如果正在 `await` 的任务被取消，它会像对任何 `Task` 一样离开 `await`；阻塞任务不会被取消，其结果留在句柄中。

```nabi
let n = await task::spawn_blocking(|| heavy_checksum(path));
```

`f` 与创建它的代码共享其普通捕获，所以这些捕获必须是 `Send + Sync`（[refs-lang 4.6.9](../../refs-lang/004-functions/406-capture-semantics.md#469-additional-constraints-in-parallel-and-concurrent-contexts)）。只有阻塞工作需要的值（一个集合、一个 `Command`）以下面两种形式之一交给它：

- 在 `f` 的捕获列表中写 `[move x]`，把 `x` 移动到闭包中（[refs-lang 4.6.11](../../refs-lang/004-functions/406-capture-semantics.md#4611-move-capture-the-move-x-capture-list)）。
- `spawn_blocking_with_arg(arg, f)` 像通道发送那样交出 `arg`，并在线程池线程上调用 `f(arg)`：该线程得到 `arg` 所能到达的每个集合的副本，之后再使用 `arg` 是编译错误（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）。`arg` 只需要 `Send`。

无论哪种方式，启动方的代码都不能再使用该值。拥有被移动捕获的闭包也会一并交出：在 `spawn_blocking(f)` 之后再次调用它是编译错误。

```nabi
let rows: List<str> = load();
let n = await task::spawn_blocking([move rows] || write_all(rows));
let m = await task::spawn_blocking_with_arg(load(), |rows: List<str>| write_all(rows));
```

### `spawn` 的 `Send` 检查

`task::spawn` 也要求结果类型是 `Send`（`where F::Output: Send`），并拒绝**帧**（frame）不是 `Send` 的 future（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）。无论 future 来自何处都是如此：不仅检查 `async` 块和 `async fn` 调用，也检查参数、结构体 / 元组 / 和类型的字段、辅助函数的结果、列表元素、闭包持有的 future，以及其他协程帧中的 future。future 的帧由以下内容组成：

- 捕获的值；
- 跨越 `await` 仍然存活的局部变量和临时值；
- 帧持有的其他 future（递归地包括它们的帧）。

```nabi
let c = Cell::new(0);                      // Cell: !Send
task::spawn(async { c.set(1); });          // ERROR: !Send capture
task::spawn_local(async { c.set(1); });    // OK: pinned to the current worker

fn make() -> AsyncFuture<()> { let c = Cell::new(0); async { c.set(1); } }
task::spawn(make());                       // ERROR: the frame of a future made by a helper is checked too
```

凡是把闭包发送到另一个线程的地方都适用同样的检查：`thread::spawn`、`thread::Builder::spawn`、`par` 的 `Scope::spawn` 以及 `task::spawn_blocking`。如果闭包持有帧不是 `Send` 的 future，就会报错 ``` `X` would move a future to another thread: ... ```。

该检查可以保守地近似。实现可以拒绝实际上是 `Send` 的 future，但绝不能接受会把非 `Send` 值传给另一个线程的 future。检查不会深入以下值：

- 类型无法持有函数值的值（它们无法藏有 future 的帧）；
- 类型带有显式 `unsafe impl Send` 的值（例如 `Task<T>`）；由该类型负责安全性。

`async fn` 的结果类型 `AsyncFuture<T>` 在 `T: Send` 时是 `Send`；会抛出错误的对应类型 `TryAsyncFuture<T, E>` 在 `T` 和 `E` 都是 `Send` 时是 `Send`。捕获和帧由上面的检查分别考察。捕获是 `Send` 但不是 `Sync` 的帧（`async fn` 接受的列表，`async` 块捕获的列表或 future）是可以接受的：`task::spawn` 像通道发送那样交出 future（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）。任务得到一个自己的帧，其中持有 future 所捕获内容的副本，这些副本在一次交接中生成，所以捕获之间原本共享的东西（同一个列表出现两次，一个列表和基于它的 future）仍然共享；与发送之后一样，spawn 之后再使用该 future 或它捕获的内容是编译错误（`E0105`）。只有尚未被轮询过的 future 才能交出：交出已经开始（被 `select`、用户的 `Future` 或手动轮询过一次）或已经完成的 future，或者其帧或所捕获 future 的帧中有 `[ref x]` 捕获的 future，会触发 panic，且不共享任何内容。通过通道发送、放入 `Mutex` 或 `Frozen` 的协程 future 也以同样方式交出，每次都得到自己的帧。对一个 `Task` 被转交出去（`Frozen::get`、通道、锁、`par` 元素）的任务，每次 `await` 都得到结果的独立副本；同一线程上的 `Task` 副本共享结果。

### `select`：最先完成的 future

```nabi
const fn select<I: IntoIterator<Item = F>, F: Future>(iter: I) -> SelectFuture<F>;   // Output = (usize, F::Output)

let (i, v) = await task::select([fetch_a(), fetch_b()]);
```

与 `race` 不同，它不启动这些 future，而是在等待的任务**内部**运行它们。

- future 按列表顺序轮询；如果两个或更多在同一次轮询中完成，索引较小的胜出。
- 一个完成时，其他所有已经开始的 future 都被取消。它们会在已取消状态下再被轮询一次，以便运行 `finally` / `using`（2.18.4）。清理中含有 `await` 的落败者会作为独立任务在后台完成清理；`select` 不等待它。从未被轮询的 future 永远不会运行。
- 落败 future 所做的所有唤醒注册（定时器、停驻）都会释放。
- 如果等待的任务被取消，子 future 也被取消。
- 没有 future 时触发 panic。
- 这些 future 必须是同一类型。结果类型不同时，把它们包装起来：`async { Msg::A(await a) }`。

也可以用 `select` 和 `sleep` 构造超时。

```nabi
enum Ev { Line(str), Timeout }

let (_, ev) = await task::select([
    async { Ev::Line(await read_line()) },
    async { await task::sleep(Duration::secs(5)); Ev::Timeout },
]);
```

### `TaskId`

`task::id()` 是正在运行的任务的 id；在任务之外调用会触发 panic，消息为 ``task::id: called outside a task; use `task::current_id()`, which is `none` there``。`Task::id()` 是该任务的 id。id 来自进程范围的计数器，永不复用。

## 2.18.2. `Task<T>` 类型

`Task<T>` 类型定义如下。

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

### `IntoFuture` 集成

`Task<T>` 实现了 `Future<Output = T>`，所以可以用 `await t` 等待；它也被 [2.20 future](220-future.md) 的 `IntoFuture` 通用实现（blanket impl）覆盖，因此与 `thread::JoinHandle`（它有自己的 `IntoFuture` 实现，3.4）以同样方式被等待。`IntoFuture` trait 和 lang 槽 `future_trait` 的权威定义在 [2.20 future](220-future.md) 中；本节原样遵循。

## 2.18.3. 调度器模型：工作窃取

一个进程有一个执行器。它在首次使用时启动，由与 CPU 数量（`cpu_count()`）相同数目的工作线程、一条阻塞通道（blocking lane）和一个 I/O 驱动组成。`block_on` 把调用线程作为客座工作线程接入该执行器，并把 `f` 运行到完成。`block_on` 返回时仍在运行的任务会继续运行。`block_on` 要等固定在其线程上的 `spawn_local` 任务也完成后才返回；这些任务无法转移到其他线程。`par::*` 和 `spawn_blocking` 使用同一个执行器。

用 `spawn` 创建的任务可以在任何工作线程上运行，所以它必须是 `Future + Send`，其结果也必须是 `Send`。来自 `spawn_local` 的任务固定在创建它的工作线程上。它固定在工作线程上，而不是操作系统线程上：工作线程可能在另一个操作系统线程上继续运行（[refs-lang 7.6.7](../../refs-lang/007-runtime/706-scheduler.md#767-automatic-detection-of-blocking-syscalls)），所以 `spawn_local` 任务不应依赖由哪个操作系统线程运行它，也不应依赖该线程的线程局部状态。（参考运行时从不转交带有存活 `spawn_local` 任务的工作线程。）在 `block_on` 线程轮询但并不拥有的任务（它在等待时取来的 `spawn` 任务）中调用 `spawn_local`，会改为固定到线程池的某个工作线程，所以 `block_on` 只等待它自己任务的局部任务。同一次这样的轮询启动的所有局部任务都交给同一个线程池工作线程，并且只在该次轮询结束后才开始，与在本该运行它们的工作线程上一样。

调度器的骨架如下。

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

该结构的关键性质：本地队列是无锁的 Chase-Lev 双端队列，拥有它的工作线程和窃取者都先取最旧的任务（因此不断重新唤醒自己的任务不会让其他任务饿死，[refs-lang 7.6.2](../../refs-lang/007-runtime/706-scheduler.md#762-work-stealing-structure)）。忙碌的工作线程每处理 32 个任务就处理一次到期的定时器。I/O 驱动方面，每个进程有一个反应器（reactor）线程以边沿触发方式等待 epoll / kqueue（io_uring / IOCP 是后续工作，[refs-lang 7.7](../../refs-lang/007-runtime/707-io-and-timer.md)）；定时器结构由实现定义（二叉堆或时间轮，[refs-lang 7.7.4](../../refs-lang/007-runtime/707-io-and-timer.md#774-timer-structure)）。

标准库的权威范围仅限于 API 签名；算法本体属于运行时和标准库实现，不在规范之内。

执行器自身的类型和函数（`TaskRef`、`Pool`、`Worker`、`wake_task`、`cancel_requested`、`unwinding`、`park`、`unpark`）在 `core` 中是 `pub(pkg)`：其他包不会用到这些名称。`await` 调用的钩子（`cancel_point`、`cancel_left`、`cleanup_running`、`cleanup_done`）也是 `pub(pkg)`；降级后的代码通过它们的 lang 槽访问它们（[0.4.4](../000-meta/004-lang-slot-table.md)）。`readable` / `writable`（套接字 `*_async` 方法所等待的就绪 future，[refs-lang 7.7.2](../../refs-lang/007-runtime/707-io-and-timer.md#772-io-wait-pattern)）、`wake_io`（在关闭或 shutdown 之后唤醒某个描述符的读写等待者）以及 `host_enter` / `host_leave` / `host_run`（WebAssembly 组件的事件循环入口）是 `#[internal]`：由 `std` 调用，其他包使用它们是错误（[0.4](../000-meta/004-lang-slot-table.md)）。在 `host_enter` 之外调用 `host_leave` 会触发 panic 并说明原因。

## 2.18.4. 协作式取消：`Task::cancel`

协作式取消的用法如下。

```nabi
let t = task::spawn(async { long_running() });
// ... other work ...
t.cancel();              // cooperative signal; the task ends at a cancellation point
```

`cancel()` 只是**请求**取消。规则如下（[refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)）：

- 每个 `await` 都是取消点。已取消的任务在下一个 `await` 处如同执行 `return` 一样离开函数体，并按由内到外的顺序运行途经的 `finally` 和 `using` 清理。
- 这些清理中的 `await` 不是取消点：它会完成，任务会一直被轮询直到清理结束。永不结束的清理会让已取消的任务一直存活；用 `task::timeout` 加以限制。
- 取消不是可以在任务内 `catch` 的错误。不会向任务内抛出 `CancellationError`。
- 阻塞系统调用不是取消点。任务在调用结束后的下一个 `await` 处离开。
- 在套接字 I/O（`*_async`）、`sleep` / `timeout`、`lock_async` 或 `recv_async` / `send_async` 中等待的任务，被取消时会被唤醒，并在下一个 `await` 处离开。
- 任务结束时，它的所有唤醒注册（I/O 就绪、定时器、停驻的等待）都会释放。
- 在第一次轮询之前被取消的任务永远不会运行：函数体的任何部分都不运行，任务没有结果地结束。
- 如果在等待 `spawn_blocking` 或基于它的 `fs::*_async` 时被取消，已经开始的工作会在后台运行到完成。写入等副作用确实会发生。在任务再次轮询之前完成的结果由那次轮询保留，见下文。
- 如果在某次轮询中被等待的 future 交出了一个值（一条消息、一个锁守卫），该轮询会保留这个值。取消检查看的是“被等待的协程是否因取消而离开”；值保留下来，任务在下一个 `await` 处离开（[refs-lang 7.7.9](../../refs-lang/007-runtime/707-io-and-timer.md#779-cancellation)）。
- 经过最后一个取消点之后才被取消的任务会正常完成并有结果。`await t` / `join` 返回该结果。
- `CancellationError` 由等待一个没有结果就结束的任务的一方收到。用 `try await t.join()` 等待这样的任务会抛出 `CancellationError`；用 `await t` 等待会触发 panic。

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

## 2.18.5. `thread::*`：1:1 操作系统线程

独立的 `thread` 模块直接启动**操作系统线程**，而不使用 M:N 模型。

```nabi
use std::thread;

let t = thread::spawn { os_level_work() };
let r = try t.join();          // OS-level synchronous wait
```

两种模型的比较如下。

| 方面                    | `task::spawn`              | `thread::spawn`         |
|-------------------------|----------------------------|-------------------------|
| 模型                    | M:N（用户态调度器）        | 1:1（操作系统线程）     |
| 开销                    | 轻（约 1KB 帧）            | 重（约 MB 级栈）        |
| 协作式取消              | ✓                          | ✗（需要操作系统级 kill） |
| async 集成              | ✓                          | △（只有 join 的一方 await） |
| 用途                    | 一般并发（默认）           | CPU 密集、阻塞 I/O      |

默认使用 `task::spawn`，只有明确需要操作系统线程时才选择 `thread::spawn`。

## 2.18.6. `Duration` / `Instant`

任务函数接受的时间类型是 `core::time::Duration`（有符号 `i64` 纳秒时长）和 `core::time::Instant`（单调时钟读数），定义在 [3.3](../300-std/303-time.md)（3.3.2、3.3.5）中。它们位于 `task` 旁边，而不在其中：`sleep`、`sleep_until` 和 `timeout` 之所以留在 `task` 中，是因为它们会挂起任务。

`await task::sleep(Duration::ms(100))` 或 `await task::sleep_until(t)` 这样的调用保证使用准确的单调时间。

## 2.18.7. 模块映射

各项所在的模块：

| 项                   | 模块                                  | 行为                     |
|----------------------|---------------------------------------|--------------------------|
| `task::spawn`        | `core::task` + 调度器运行时      | 推入调度器               |
| `task::race`         | `core::task`                     | 选出最先完成者           |
| `task::join`         | `core::task`                     | 等待全部                 |
| `task::timeout`      | `core::task`                     | race + 定时器            |
| `Task<T>`            | `core::task`                     | 协程帧引用               |
| `Duration`/`Instant` | `core::time`                     | i64 ns + 单调时钟        |
| `Future`（lang 槽）  | `core::future`                   | 基于轮询                 |
| `IntoFuture`         | `core::future`                   | await 入口点             |
