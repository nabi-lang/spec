# 7.6. 调度器

Nabi 的调度器是 M:N [工作窃取](../000-meta/007-glossary.md#work-stealing)调度器；本文定义其结构。

## 7.6.1. 统一的工作线程模型

一个进程有一个执行器，在首次使用时启动。工作线程是与 CPU 数量相同的操作系统线程，外加一条阻塞通道和一个 I/O 驱动。`task::block_on` 把调用线程作为访客工作线程接入；它返回后，剩余的任务继续运行。`par::*` 和 `spawn_blocking` 使用同一个执行器（[标准库 2.18.3](../../refs-standard-libraries/200-core/218-task.md#2183-scheduler-model---work-stealing)）。每个工作线程兼具以下角色：

```
worker = {
    scheduler   // task queue (work-stealing)
    mutator     // TLAB + shadow stack (701 / 702 / 704)
}
```

工作线程不兼任 I/O 驱动；每个进程由单个 reactor 线程负责 I/O（[7.7.1](707-io-and-timer.md#771-io-multiplexing)）。STW 回收由发起回收的线程执行，并发标记由标记线程执行（7.6.10）。

## 7.6.2. 工作窃取结构

每个工作线程有一个本地双端队列（无锁的 Chase-Lev 双端队列，[8.5.4 `nabi_deque_*`](../008-abi/805-extern-symbols.md#nabi_deque_---work-stealing-deque)）。

```
Worker:
  ├── local_deque : growable ring    (this worker's tasks, GC-traced)
  └── ...
```

新唤醒或新生成的任务进入该工作线程双端队列的底部。**所属的工作线程和窃取者都先取最旧的任务（顶部）**。如果所属工作线程先取最新的任务，那么一个不断重新唤醒自己的任务（比如在 `await task::yield_now()` 上循环的任务）可能让排在它后面的任务饿死。

在这种工作窃取模型中，工作线程通常运行自己生成的任务，从而保持缓存局部性；空闲的工作线程窃取工作，从而实现负载均衡。实现可以加入 LIFO 槽之类的“最近任务优先”优化，只要上述饥饿不会发生。

## 7.6.3. 全局注入队列

除每个工作线程的本地队列外，还有一个单独的全局注入队列。

```
Global injector:
  - receives tasks spawned from outside (e.g. extern callback).
  - when a worker's local queue is empty, pull from global.
```

这个队列是从工作线程池之外到来的任务的入口。

## 7.6.4. 窃取算法

当工作线程的本地双端队列为空时，它按以下顺序进行：

```
1. Try to pull from the global injector.
2. Try to steal from the top of another worker's local deque. Victims start at a per-worker pseudo-random index, then go round-robin.
3. If all fail, sleep / park (blocking sync).
```

窃取最多尝试 N-1 个工作线程；如果都为空，该工作线程切换到驻留（parked）状态。

## 7.6.5. 关键字与库函数

遵循[关键字](../001-lexical/105-keywords.md)中尽量减少保留关键字的原则，调度器功能全部以库函数提供。

| 库函数                  | 含义                                                                     |
|-------------------------|--------------------------------------------------------------------------|
| `task::spawn(fut)`      | 把 future 作为新任务压入队列。结果为 `Task<T>`。                         |
| `task::yield_now()`     | 主动让出（`await`）。让其他就绪任务运行一次。                            |
| `task::sleep(d)`        | 移入定时器队列。                                                         |
| `task::race([...])`     | 最先完成的任务；其余任务被取消（await）。                                |
| `task::join([...])`     | 直到全部完成（`List<Task<T>>`，await）。                                 |
| `task::timeout(d, t)`   | 时间限制。超时则在 `try await` 处得到 `TimeoutError`。                   |
| `thread::spawn(f)`      | 创建新的操作系统线程（1:1）。                                            |
| `par::for_each(xs, f)`  | 并行 for-each（在 join 上[阻塞](../000-meta/007-glossary.md#block)）。   |
| `par::scope { ... }`    | 并行作用域（尾随闭包）。                                                 |

由于它们不是关键字，用户代码中可以自由使用 `spawn` 或 `race` 这样的标识符。

## 7.6.6. 1:1 操作系统线程：`thread::spawn`

大多数任务按调度器的映射以 M:N 方式运行，但对于阻塞工作或外部库调用，专用的操作系统线程有时更自然。这时，`thread::spawn` 创建一个 1:1 映射的真实操作系统线程。

在内部，它直接调用 Unix 的 `pthread_create`、Windows 的 `CreateThread` 等，并在与普通任务分开的池中运行。结果以 `JoinHandle<T>` 返回，这是一个不同于 Future 的类型。

## 7.6.7. 自动检测阻塞系统调用

工作窃取模型有一个问题：当任务调用阻塞的系统调用时，工作线程本身会停滞。这一问题通过自动卸载到阻塞池来解决。

编译器和运行时会自动把已知的阻塞系统调用交给阻塞池的工作线程，它是单独的操作系统线程，因此不会妨碍其他任务的进展。具体机制留给调度器实现决定。

> [!NOTE]
> 在参考运行时中，如果池线程的任务一直停留在阻塞调用中，该线程会把它的工作线程身份交给一个新线程。用 `spawn_local` 生成的任务会固定其工作线程，该工作线程永远不会被移交。

## 7.6.8. 每核一线程选项

默认有 N 个工作线程，任务在线程之间自由移动。启用每核一线程（thread-per-core）选项后，每个任务固定在一个工作线程上，工作线程之间不互相窃取；缓存局部性最大化，适合 NUMA 环境和数据中心工作负载。

它通过环境变量 `NABIR_THREAD_PER_CORE=1` 开启；`NABIR_WORKERS=<n>` 设置 N（默认：在线 CPU 数）。

> [!NOTE]
> 任务固定在工作线程上；任务阻塞的工作线程可能在另一个操作系统线程上继续（[7.6.7](#767-automatic-detection-of-blocking-syscalls)），因此对于固定的任务，线程身份并不稳定。

## 7.6.9. 优先级 / 公平性

默认是无优先级的 FIFO + 工作窃取。所属工作线程和窃取者都先处理双端队列中最旧的任务（7.6.2）。

繁忙的工作线程也不会推迟定时器。每执行 32 个任务，它就处理已到期的定时器，并把对应任务放回队列。因此即使在总有工作的工作线程上，`task::sleep` / `task::timeout` 也会按时唤醒。

任务没有优先级：运行时和标准库都不提供设置优先级的方法。应当让步的任务调用 `task::yield_now`（[标准库 2.18](../../refs-standard-libraries/200-core/218-task.md)），而对延迟敏感、不得排在其他任务之后等待的工作，在自己的线程上运行（`thread::spawn`，[标准库 3.4](../../refs-standard-libraries/300-std/304-thread.md)）。

## 7.6.10. 集成的 GC 工作角色

STW 回收由通过分配或 `nabi_gc_collect` 发起它的线程执行，其他工作线程在安全点停下并等待（[7.4.3](704-barriers-and-roots.md#743-safepoint-polling)）。回收之后，工作线程恢复运行任务。并发标记由单独的一个标记线程完成（[7.5](705-concurrent-collection.md)）。

## 7.6.11. 调度器与协程

调度器使用[协程](708-coroutine.md)中的无栈协程；每个任务的状态机存放在协程帧中。

当任务遇到 `await` 或 `yield` 时，只有协程帧回到队列中，工作线程继续执行其他任务；任务恢复时，可能由另一个工作线程取出该帧并继续执行。这一模型让数百万个任务可以在 N 个工作线程上高效处理。

## 7.6.12. 生成开销

`task::spawn` 只需分配一个无栈协程帧即可完成。帧的大小取决于任务的局部变量，大约 64 到 256 字节，并在 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 快速路径上分配，因此开销接近零。

因此，即使有数百万个并发任务，也比操作系统线程轻得多，无法相提并论。

## 7.6.13. 整体流程

```
user code:    task::spawn(async { compute() })
  ↓
compiler:     create coroutine frame + push to queue
  ↓
worker:       pop task from its queues and execute
  ↓ (on await / yield)
worker:       requeue frame, run another task
  ↓ (some worker)
resume:       pop frame and continue execution
  ↓ (on normal termination)
result:       delivered as Output of Task<T> (Task<T>: Future<Output = T>)
```
