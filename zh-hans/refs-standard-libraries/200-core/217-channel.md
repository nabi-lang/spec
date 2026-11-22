# 2.17. channel：消息传递

`core::channel` 模块提供在任务和线程之间交换消息的通道原语。其设计参考了 Go 的通道和 Rust 的 `mpsc::channel`。

名称 `channel`、`Sender` 和 `Receiver` 在 [prelude](../000-meta/002-import.md) 中，无需导入即可以 `channel::<T>()` 或 `Sender<T>` 这样的短形式使用。如果希望来源清晰可见，推荐显式写出 `use core::channel::*;`。

## 2.17.1. 基本形式

通道由两个函数创建。

```nabi
const fn channel<T: Send>() -> (Sender<T>, Receiver<T>);
const fn channel_bounded<T: Send>(cap: usize) -> (Sender<T>, Receiver<T>);
```

`channel<T>()` 创建**无界**通道：其容量无限，所以实际上可以接收消息直到内存上限。`channel_bounded<T>(cap)` 创建容量固定为 `cap` 的**有界**通道；队列满时，发送方阻塞。`channel_bounded(0)` 是会合（rendezvous）通道：`send` 一直等待，直到有接收方取走该值。在这种通道上，如果没有接收方在等待，`try_send` 抛出 `TrySendError::Full`。不仅阻塞在 `recv` 中的线程，挂起在 `recv_async` / `for await` 中的任务也算作正在等待的接收方。

### 示例

最简单的用法：

```nabi
let (tx, rx) = channel::<i32>();

task::spawn(async {
    try tx.send(1);
    try tx.send(2);
    try tx.send(3);
});

while let v: i32 = rx.recv() {
    process(v);                  //→ 1, 2, 3
}
```

## 2.17.2. `Sender<T>` / `Receiver<T>`

构成通道两端的两个类型定义如下。

```nabi
struct Sender<T> { /* ... */ }
struct Receiver<T> { /* ... */ }

impl<T> Sender<T> {
    fn send(self, value: T) throws (SendError<T>);
    fn try_send(self, value: T) throws (TrySendError<T>);
    const fn send_async(self, value: T) -> SendFuture<T>;   // try await tx.send_async(v): suspends only the task, throws (SendError<T>)
    fn clone(self) -> Sender<T>;             // multiple senders
    fn close(self);                          // closes this sender
}

impl<T> Drop for Sender<T> {             // releasing a sender closes it
    fn drop(ref self);
}

impl<T> Receiver<T> {
    fn recv(self) -> T | none;               // none = every sender closed
    fn try_recv(self) -> T | Empty | Closed; // nothing queued: Empty while a sender is open, else Closed
    const fn recv_async(self) -> RecvFuture<T>;    // await rx.recv_async(): suspends only the task, Output = T | none
    fn iter(self) -> ReceiverIter<T>;         // for x in rx { ... }
}

impl<T> Stream for Receiver<T> {         // for await v in rx: suspends only the task
    type Item = T;
    ...
}

impl<T> IntoIterator for Receiver<T> {   // for x in rx: ends when every sender is closed
    type Item = T;
    type IntoIter = ReceiverIter<T>;
}

enum SendError<T> { Closed(T) }
enum TrySendError<T> { Full(T), Closed(T) }
impl<T> SendError<T> { fn message(self) -> str; }      // "sending on a closed channel"
impl<T> TrySendError<T> { fn message(self) -> str; }   // "sending on a full channel" / "... closed channel"
// both implement Display (the message), Debug (`Closed(..)`, `Full(..)`) and Error: they name the case, not the value
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
struct Empty;      // try_recv: nothing queued, a sender is still open
#[derive(Clone, Copy, PartialEq, Eq, Hash, Debug)]
struct Closed;     // try_recv: nothing queued, every sender closed

struct RecvFuture<T> { /* ... */ }
impl<T> Future for RecvFuture<T> {
    type Output = T | none;
    ...
}

struct SendFuture<T> { /* ... */ }
impl<T> SendFuture<T> {
    // driven by `try await`, like `task::Join` (2.18.1): the error is the send's
    fn poll(ref self, ref ctx: Context) throws (SendError<T>) -> Poll<()>;
}

struct ReceiverIter<T> { /* ... */ }    // Iterator<Item = T>, ends when every sender is closed
```

如果 `T: Send`，那么 `Sender<T>` 和 `Receiver<T>` 也是 `Send`，可以移动到其他任务或线程（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）。`send`、`try_send` 和 `send_async` 把该值的集合的副本放入队列（2.13.9）：接收方与发送方不共享任何集合。

`recv` 和 `send`（有界通道已满时，或在会合通道上直到值被取走为止）会阻塞线程。要在异步代码中只挂起任务，使用 `await rx.recv_async()`、`for await v in rx` 或 `try await tx.send_async(v)`。`send_async` 在有界通道已满时挂起任务，在会合通道上则挂起到有接收方取走该值为止。与 `send` 一样，如果这个发送方已经被 `close()`，它抛出携带该值的 `SendError::Closed`。被取消的 `recv_async` 不会取走值，被取消的 `send_async` 不会发送它的值（[2.18.4](218-task.md#2184-协作式取消taskcancel)）。`Mutex::lock` / `lock_async` 也做同样的区分（[2.16](216-sync.md)）。

## 2.17.3. 关闭语义

通道何时关闭、信号如何传递：一旦所有发送方都通过 `close()` 或被释放而关闭，通道就关闭。接收方随后通过 `recv()` 返回 `none` 得到关闭信号。

用 `close()` 关闭通道：

```nabi
let (tx, rx) = channel::<i32>();
try tx.send(1);
tx.close();            // explicit close

rx.recv();             //→ 1
rx.recv();             //→ none  (closed)
```

`Sender` 是 `Drop` 资源（[2.13.4](213-marker.md#2134-drop释放时的清理)）：`let tx2 = tx` 移动它，`clone()` 创建另一个拥有自己所有者的发送方，`close()` 只关闭那一个发送方。编译器在发送方的最后一个所有者离开作用域的地方释放它，任务结束时释放其 `async` 块捕获的发送方；所有发送方都关闭或释放后，通道关闭。释放发送方不会失败，所以不需要显式关闭。线程的闭包不会释放它捕获的东西：请在线程工作结束时关闭发送方。

## 2.17.4. 多个生产者与消费者

`Sender<T>` 实现 `Clone`，所以多个任务可以向同一个通道发送消息。

```nabi
let (tx, rx) = channel::<str>();
let tx2 = tx.clone();

task::spawn(async { try tx.send("hello"); });
task::spawn(async { try tx2.send("world"); });

let v1 = rx.recv();
let v2 = rx.recv();
```

`Receiver<T>` 不实现 `Clone`，但它是句柄：副本（`let rx2 = rx;`，或移动到其他任务或线程中的副本）共享同一个队列。因此通道是多生产者、多消费者（MPMC）的。任意数量的接收方副本都可以同时调用 `recv`（或 `try_recv`、`recv_async`、`for`），每个发送的值恰好交给其中一个，即最先取走它的那个；没有哪个接收方能看到所有值。所有发送方都关闭且队列为空后，每个副本都得到 `none`。要把每个值交给多个消费者，请使用一个扇出任务，把值转发到每个消费者各自的通道。

## 2.17.5. 与 `for` / `Stream` 的集成

`Receiver<T>` 实现 `IntoIterator<Item = T>`，所以可以直接用在 `for` 语句中。

```nabi
for v in rx {
    process(v);                  // repeats until the channel is closed
}
```

它也实现异步迭代器 `Stream<Item = T>`，所以 `for await` 的用法相同。

```nabi
for await v in rx {
    await handle(v);
}
```

详见 [2.7 iter](207-iter.md) 和 [2.8 stream](208-stream.md)。

## 2.17.6. 模块映射

每个类型和函数都定义在 `core::channel` 中，这是它唯一的路径（[0.1.6](../000-meta/001-tier-model.md#016-每个项只有一条路径)）：

| 类型           | 模块                 | 内部结构                 |
|----------------|----------------------|--------------------------|
| `Sender<T>`    | `core::channel` | 原子引用计数 + 队列           |
| `Receiver<T>`  | `core::channel` | 队列 + 停驻（parking）        |
| `channel<T>()`, `channel_bounded<T>(cap)` | `core::channel` | 类似 Arc 的内部状态 |
| `SendError<T>`, `TrySendError<T>` | `core::channel` | 携带未发送值的错误 |
| `Empty`, `Closed` | `core::channel` | `try_recv` 的结果 |
| `ReceiverIter<T>` | `core::channel` | `for x in rx` 的迭代器 |
| `RecvFuture<T>`, `SendFuture<T>` | `core::channel` | `recv_async` / `send_async` 的 future |

内部队列方面，有界通道使用类似 Tokio mpsc 的环形缓冲区，无界通道使用分段链表。有界通道的队列是无锁的；无界通道的分段链表可能使用锁。
