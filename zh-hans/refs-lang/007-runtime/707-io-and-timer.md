# 7.7. I/O 驱动与定时器

[调度器](706-scheduler.md)的工作线程使用两种机制，高效处理等待 I/O 和等待定时器的任务。

## 7.7.1. I/O 多路复用

每个进程有**一个 reactor 线程**，它在操作系统的就绪多路复用器上等待，并唤醒等待已就绪 fd 的任务。

| 操作系统 / 环境       | 机制                                |
|-----------------------|-------------------------------------|
| Linux                 | epoll（`EPOLLET`，边沿触发）        |
| BSD（FreeBSD / macOS） | kqueue（`EV_CLEAR`，边沿触发）     |
| Windows               | IOCP + AFD poll（模拟就绪模型）     |
| WASI（`wasm32-wasip2`） | `wasi:io/poll`                    |
| WASI（`wasm32-wasip1-threads`） | `poll_oneoff`             |

目标对应的驱动在构建时通过[条件编译](../006-meta-system/603-cfg.md)的 `target_os` 分支编译进来。

reactor 线程在第一次异步 I/O 等待时启动。

在 `wasm32-wasip2`（preview 2 组件）上，驱动是 `wasi:io/poll`；当宿主授予网络访问权限时，`std::net` 通过 `wasi:sockets` 工作。在 `wasm32-wasip1-threads`（preview 1）上，`poll_oneoff` 监视宿主传入的套接字。

## 7.7.2. I/O 等待模式

一次 I/O 等待按如下方式进行：

```
1. The task tries read on a non-blocking socket → WouldBlock.
2. The task awaits fd's readiness: (re)registers fd with the reactor and parks.
3. The worker runs another task.
4. When the reactor receives a readiness edge for fd, it unparks the task parked on that fd.
5. Some worker takes that task and retries the read.
```

这一流程与 `async` `fn` 中的 `await` 结合（见[异步](../004-functions/412-async.md)）。注册是边沿触发的，因此每次等待都要重新注册。标准库的就绪 future 在驻留之后会再检查一次 fd，以免错过期间经过的边沿。被唤醒的任务可能发现 fd 尚未就绪，因此由调用方重试操作。这些 future（`task::readable` / `task::writable`）是 `#[internal]` 的（[标准库 2.18](../../refs-standard-libraries/200-core/218-task.md)）：手写的 future 通过 `nabi_io_register` 等待 fd（[8.5.5](../008-abi/805-extern-symbols.md#855-io--定时器区域)）。

阻塞调用（`TcpStream::read` 等）不使用这条路径；它们阻塞调用方的操作系统线程。挂起任务的 I/O 由标准库的 `*_async` 方法提供（[标准库 3.2.0](../../refs-standard-libraries/300-std/302-net.md#320-通用规则)）。

普通文件无法用就绪模型等待（在 epoll / kqueue 下它们总是就绪的）。因此，普通文件上的异步 I/O（`fs::*_async`，[标准库 3.1.4a](../../refs-standard-libraries/300-std/301-fs.md#314a-异步文件-io_async)）在另一个操作系统线程上执行阻塞的系统调用（`task::spawn_blocking`），并在此期间挂起任务。

## 7.7.3. I/O 驱动抽象

每个驱动都实现一个公共接口。

```
trait IODriver {
    fn register(ref self, fd: Fd, interest: IoInterest) throws (IoError);   // edge-triggered
    fn poll(ref self, timeout: Duration) -> List<Event>;
}
```

这个接口是概念性的；确切的签名由运行时实现决定。驱动不持有唤醒器。要唤醒的任务通过由 fd 和方向（读 / 写）决定的驻留键，在 `core::task` 的驻留表中查找。运行时符号见 [8.5.5](../008-abi/805-extern-symbols.md#855-io--定时器区域)。

## 7.7.4. 定时器结构

`task::sleep(d)` / `task::timeout(d, ..)` 的等待保存在按截止时间排序的二叉堆中。截止时间一过，任务就被唤醒。已取消任务的定时器留在堆中，在到期或清理时被丢弃。在处理定时器时截止时间已过的睡眠按截止时间顺序唤醒，即使有多个线程同时处理定时器也是如此；在第一次轮询时截止时间已过的睡眠当即完成。最早的截止时间还保存在一个原子字中，因此没有到期定时器的工作线程不会锁定堆，并且只有成为最早截止时间的定时器才会唤醒正在睡眠的工作线程（其他工作线程本来就会睡到更早的截止时间）。

插入和到期都是 `O(log n)`。

## 7.7.5. 定时器精度

没有固定的时钟节拍。截止时间以纳秒保存，无事可做的工作线程在操作系统的定时等待中睡到下一个截止时间，因此定时器触发的精度与该等待唤醒的精度相同。

## 7.7.6. 定时器与 I/O 的整合

工作线程的主 `loop` 如下。对于 I/O 就绪，reactor 线程通过驻留表把任务放回队列，因此工作线程不直接轮询驱动。

```nabi
loop {
    // 1. Process ready tasks from the queue (including tasks woken by the reactor)
    while let task: Task = queue.next_ready() {        // stops on none
        execute(task);
    }

    // 2. Return tasks of expired timers to the queue
    timers.fire_due();

    // 3. If there is nothing to do, park until the next timer (woken by unpark)
    park_until(timers.next_deadline());
}
```

reactor 线程的 `loop` 是：

```nabi
loop {
    let events = io_driver.poll();                     // blocks; a native section for the GC
    for event in events {
        task::unpark(parking_key(event.fd, event.direction));
    }
}
```

## 7.7.7. 驻留 / 取消驻留

当工作线程的队列完全为空且 I/O 空闲时，该工作线程在操作系统睡眠中驻留（park）。

- 驻留使用 `pthread_cond_wait` 或操作系统特定的睡眠机制。
- 取消驻留（unpark）由另一个工作线程或 I/O 事件触发。

驻留期间 CPU 使用率为零，因此大量空闲的工作线程不会带来额外开销。

## 7.7.8. 动态调整工作线程数

默认情况下，N 固定为 CPU 核心数。

启用后（`NABIR_DYNAMIC_WORKERS=1`），过于空闲的工作线程会驻留，推送任务时优先唤醒其他工作线程；当所有活动的工作线程都繁忙时，它会被重新启用。在调用中阻塞的工作线程在所有模式下都会被替换（[7.6.7](706-scheduler.md#767-自动检测阻塞系统调用)）。

## 7.7.9. 取消

`Task::cancel` **请求**取消（[标准库 2.18.4](../../refs-standard-libraries/200-core/218-task.md#2184-协作式取消taskcancel)）。运行时设置一个标志并唤醒该任务。

- 每个 `await` 都是取消点。`await` 在进入时以及即将挂起之前检查该标志。在任务被轮询期间到达的取消，会在其下一次轮询时处理，这次轮询由取消引起的唤醒带来：被等待的 future 最先看到它，因此嵌套的协程以及 `task::select` 中每个已启动的子项都会运行其清理。
- 被等待的 future 返回 `Ready` 之后的检查不是“标志是否已设置”，而是“**被等待的协程是否因取消而离开**”。因此 future 交出的值（收到的消息、获得的锁守卫）会被保留，任务在下一个 `await` 处离开。
- 被取消的任务在下一个 `await` 处像执行 `return` 一样离开其主体。经过的 `finally` 块和 `using` 的 `drop` 从最内层向外运行。嵌套的 `async fn` 调用从最内层的协程开始清理。
- 当协程运行其取消的清理时，无论在它内部还是在它等待的对象中，都没有 `await` 检查该标志：清理会完整执行。运行时持续轮询该任务直到清理结束；外层协程在内层协程完成清理后离开。
- 这不是能在任务内部 `catch` 的错误。不抛出错误的异步主体也经由同一路径结束。结果被丢弃。
- 在最后一个取消点之后被取消的任务会正常完成并具有结果（`await` / `join` 返回该结果）。`CancellationError` 只适用于没有结果就结束的任务。
- 阻塞的系统调用（`TcpStream::read` 等）不是取消点。任务在调用结束后的下一个 `await` 处离开。
- 在套接字 I/O、`sleep` / `timeout`、`lock_async` 或 `recv_async` / `send_async` 中等待的任务，被取消时会被唤醒，并在下一个 `await` 处离开。
- 任务结束时，它的所有唤醒注册（I/O 就绪等待、定时器、驻留表中的唤醒器）都会被释放。运行中池里的注册数量可以用诊断函数 `task::registrations()` 查看；该值的确切含义由实现定义。

在 `task::timeout` 到期或 `task::race` 决出胜者之后，落败的任务会继续运行。要停止它们，调用 `cancel`。`task::select` 不生成任务，因此它自己取消落败的 future（[标准库 2.18.1](../../refs-standard-libraries/200-core/218-task.md#select最先完成的-future)）；清理中含有 `await` 的落败者作为独立的任务完成清理。

## 7.7.10. 与 reactor 模式的比较

Nabi 采用传统的 reactor 结构：一个单独的 reactor 线程等待 I/O，只通过驻留表与工作线程交汇。代价是 I/O 唤醒要跨线程（一次 unpark）。
