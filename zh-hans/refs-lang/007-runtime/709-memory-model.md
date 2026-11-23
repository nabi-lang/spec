# 7.9. 内存模型

本节定义一个线程能看到其他线程写入的内存的哪些内容。Nabi 采用 C++20 的内存模型（ISO/IEC 14882:2020，[intro.multithread] 和 [atomics.order]），Rust 也使用这一模型；下文术语具有其 C++20 含义。在此基础上，本节规定哪些库操作会同步，以及安全的 Nabi 提供哪些保证。与 7.1 至 7.5 不同，本节是规范性的。

这里的“线程”指操作系统线程。任务（[标准库 2.18](../../refs-standard-libraries/200-core/218-task.md)）在工作线程上运行，并可能在任何挂起点之后在另一个工作线程上恢复；调度器为任务自己的各个步骤排定顺序，因此一个任务中的代码无论在哪里恢复，都能看到它自己先前的写入。两个任务之间适用下面的规则，如同在两个线程之间。

## 7.9.1. 关系

- **先序于（sequenced before）**：在一个线程内，按求值顺序（[3. 表达式](../003-expressions/README.md)）排在另一个求值之前的求值。
- **修改顺序（modification order）**：每个原子对象的所有修改都有一个唯一的全序，每个线程都一致地观察到它（C++20 的一致性规则）。
- **同步于（synchronizes with）**：原子对象上的释放操作，与同一对象上读取它所写的值（或其释放序列中更晚的值）的获取操作同步。7.9.3 中的库操作也按那里的规定同步。
- **先行发生（happens before）**：先序于与同步于的传递闭包（即 C++20 的 simply-happens-before；Nabi 没有 `consume` 内存序，因此两个概念重合）。

非原子读取看到的是先行发生于它的、对该位置的最后一次写入，前提是没有其他对该位置的写入与该读取之间无序。

## 7.9.2. 原子内存序

`Atomic<T>`（[标准库 2.16.2](../../refs-standard-libraries/200-core/216-sync.md#2162-atomict无锁原子操作)）接受 C++20 的五种内存序，含义与 C++20 相同：

| 内存序    | C++20                        | 含义 |
|-----------|------------------------------|------|
| `Relaxed` | `memory_order_relaxed`       | 原子的，并处于修改顺序中；不同步。 |
| `Acquire` | `memory_order_acquire`       | 一次加载（或读-改-写的加载部分），与它所读取的释放操作同步。 |
| `Release` | `memory_order_release`       | 一次存储（或读-改-写的存储部分），作为一个释放序列的开头。 |
| `AcqRel`  | `memory_order_acq_rel`       | 同时具备两者的读-改-写。 |
| `SeqCst`  | `memory_order_seq_cst`       | 对加载是获取，对存储是释放，并属于所有 `SeqCst` 操作的唯一全序。默认值。 |

失败的比较交换是一次带有其失败内存序的加载。每个操作的内存序类型只接受该操作可以采用的内存序（2.16.2），因此不存在 `consume` 和获取存储。同一对象上的原子操作永远不会撕裂，也永远不会彼此构成数据竞争，无论其内存序如何。

Nabi 没有独立的内存栅栏：每个内存序都附着在一个原子操作上。`volatile` 访问（[标准库 1.5](../../refs-standard-libraries/100-bare/105-volatile.md)）不是原子的，不在线程之间排定任何顺序。

## 7.9.3. 会同步的操作

以下每一项都使较早的操作同步于较晚的操作，因此先序于前者的一切，都先行发生于后者之后（按先序关系）的一切。

| 较早 | 较晚 |
|------|------|
| `Mutex` 解锁（守卫的释放） | 该 `Mutex` 下一次成功的加锁（`lock`、`try_lock`、`lock_async`） |
| `RwLock` 写解锁 | 下一次 `read` / `write` 获取 |
| `RwLock` 读解锁 | 下一次 `write` 获取 |
| `Condvar::wait` 释放其锁 | 视同 `Mutex` 解锁；其重新加锁视同 `Mutex` 加锁 |
| `Once::call_once` 的 `f` 返回 | 该 `Once` 的每次 `call_once` 返回，以及返回 `true` 的 `is_completed` |
| 惰性初始化的 `static` 的初始化表达式完成 | 对该 `static` 的每次访问（[5.3.9](../005-modules/503-static.md#539-初始化式编译期或首次使用时)） |
| 在通道上 `send` / `try_send` / `send_async` 一个值 | 接收该值的 `recv`（[标准库 2.17](../../refs-standard-libraries/200-core/217-channel.md)） |
| `thread::spawn` / `Builder::spawn` | 新线程的第一步 |
| 线程的最后一步 | 返回其结果的 `join`（或对其 `JoinHandle` 的 `await`）（[标准库 3.4](../../refs-standard-libraries/300-std/304-thread.md)） |
| `task::spawn` / `spawn_blocking` | 该任务的第一步 |
| 任务的最后一步 | 对其 `Task` 返回结果的 `await`（[标准库 2.18](../../refs-standard-libraries/200-core/218-task.md)） |
| `par` 调用或 `par::scope` 的开始 | 其每个作业的第一步 |
| 每个 `par` 作业的最后一步 | `par` 调用的返回，或作用域的结束（[标准库 2.19](../../refs-standard-libraries/200-core/219-par.md)） |

在参考库中，锁、`Once` 和通道都建立在 `SeqCst` 原子操作之上，因此上述顺序也属于 `SeqCst` 全序。`Atomic<i32>::wait` / `wake` 只负责阻塞和唤醒：它们本身不同步任何东西，`wait` 比较的值以该原子对象自身的内存序读取。

回收器、其写屏障（[7.4.2](704-barriers-and-roots.md#742-写屏障对象记忆)）以及全局停顿只与回收器同步。它们不为用户代码提供任何先行发生边，即使它们恰好在硬件层面起到了栅栏作用。

## 7.9.4. 数据竞争

对同一内存位置的两次访问，如果至少一次是写入，并且两者不都是原子的，就称为**冲突**。如果两次冲突的访问发生在不同线程中，并且两者都不先行发生于对方，程序就存在**数据竞争**。数据竞争是未定义行为（[4.15.8](../004-functions/415-unsafe.md#4158-未定义行为)）。

数据竞争只能经由 `unsafe` 放行的代码发生。安全的 Nabi 中没有数据竞争，原因如下：

- `static` 可从每个线程访问，因此其类型必须是 `Sync`（[5.3.1](../005-modules/503-static.md#531-基本形式)）；`static mut` 只存在于 `extern "C"` 块中，访问它需要 `unsafe`。
- 集合类型（`List`、`Dict`、`StrBuf` 等）是 `!Sync` 的，`Cell<T>`、裸指针和守卫也是如此（[标准库 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync线程边界标记)）。共享的可变状态只能放在 `Mutex`、`RwLock`、`Atomic` 或通道中跨越线程。冻结的集合（`xs.freeze()`，[标准库 2.9.7b](../../refs-standard-libraries/200-core/209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s只读集合类型)）是 `Sync` 的，因为没有任何东西写入它：没有可变的句柄能到达它的存储。
- 把值交给另一个所有者（`Mutex::new`、`send`、`task::spawn`、`[move x]`、线程的结果等）时，新所有者会得到该值所能到达的每个集合的独立副本，此后再使用发送方的值是编译错误（2.13.9）。传给 spawn 的闭包，其普通捕获必须是 `Send + Sync`。
- `par` 在运行时检查两个条目不会到达同一个非 `Sync` 的集合，并逐个运行这样的条目（[标准库 2.19.1](../../refs-standard-libraries/200-core/219-par.md#2191-核心-api)）。只有在其他条目的视图或切片都不与所写部分重叠时，条目才可以通过视图（`chunks_mut`）写入同一个列表。

使数据竞争成为可能、因而需要作者留意的情形：

- 通过 `Ptr<T>`、`ManyPtr<T>`、`CPtr<T>` 和 `GcBuf<T>` 进行的读写（[标准库 1.1](../../refs-standard-libraries/100-bare/101-ptr.md)、[标准库 1.3](../../refs-standard-libraries/100-bare/103-mem.md)）；
- 在其共享实际上并未受到保护的类型上的 `unsafe impl Send` 或 `unsafe impl Sync`（[4.15.4](../004-functions/415-unsafe.md#4154-在-trait-中)）；
- 对仍被其他东西引用的值使用 `Mutex::new_unshared`（[标准库 2.16.3](../../refs-standard-libraries/200-core/216-sync.md#2163-mutext互斥锁)）；
- 在 `unsafe` 内部跳过的移交检查（2.13.9）；
- `extern static mut` 以及通过 FFI 到达的外部代码（[5.9](../005-modules/509-extern-block.md)），包括在 `env::set_var` 运行期间调用 `getenv` / `setenv` 的 C 代码（[标准库 3.5](../../refs-standard-libraries/300-std/305-env.md)）。

## 7.9.5. GC 引用

GC 引用是一个对齐的机器字，用一次访问读写。回收器依赖这一点：它永远不会遇到写了一半的引用，并把它扫描到的每个引用字都当作对象或 null。这并不会使存在竞争的引用访问成为有定义的行为。引用上的数据竞争与其他数据竞争一样是未定义行为，而 `str`（指针和长度，两个字）或结构体在竞争时可能会撕裂。
