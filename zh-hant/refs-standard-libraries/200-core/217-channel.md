# 2.17. channel：訊息傳遞

`core::channel` 模組提供在任務與執行緒之間交換訊息的通道原語。其設計參考了 Go 的通道與 Rust 的 `mpsc::channel`。

`channel`、`Sender` 和 `Receiver` 這些名稱位於 [prelude](../000-meta/002-import.md) 中，不需匯入即可用 `channel::<T>()`、`Sender<T>` 這樣的簡短形式使用。需要讓來源一目了然時，建議明確寫出 `use core::channel::*;`。

## 2.17.1. 基本形式

通道由兩個函式建立。

```nabi
const fn channel<T: Send>() -> (Sender<T>, Receiver<T>);
const fn channel_bounded<T: Send>(cap: usize) -> (Sender<T>, Receiver<T>);
```

`channel<T>()` 建立**無界**通道：其容量無限，實際上可接收訊息直到記憶體上限。`channel_bounded<T>(cap)` 建立容量固定為 `cap` 的**有界**通道；佇列已滿時，傳送端會阻塞。`channel_bounded(0)` 是會合（rendezvous）通道：`send` 會等到接收端取走該值為止。在這種通道上，若沒有接收端在等待，`try_send` 會擲回 `TrySendError::Full`。不只阻塞在 `recv` 的執行緒，暫停在 `recv_async` / `for await` 的任務也算是等待中的接收端。

### 範例

最簡單的用法：

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

構成通道兩端的兩個型別定義如下。

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

若 `T: Send`，則 `Sender<T>` 與 `Receiver<T>` 也是 `Send`，可以移動到其他任務或執行緒（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）。`send`、`try_send` 和 `send_async` 會將值所含集合型別的複本放入佇列（2.13.9）：接收端不與傳送端共用任何集合型別。

`recv` 與 `send`（有界通道已滿時，或直到會合值被取走為止）會阻塞執行緒。若要在非同步程式碼中只暫停任務，請使用 `await rx.recv_async()`、`for await v in rx` 或 `try await tx.send_async(v)`。`send_async` 在有界通道已滿時暫停任務，在會合通道上則暫停到接收端取走該值為止。與 `send` 相同，若此傳送端已經 `close()`，它會擲回攜帶該值的 `SendError::Closed`。被取消的 `recv_async` 不會取走值，被取消的 `send_async` 也不會送出其值（[2.18.4](218-task.md#2184-協作式取消taskcancel)）。`Mutex::lock` / `lock_async` 也有相同的區別（[2.16](216-sync.md)）。

## 2.17.3. 關閉語意

通道何時關閉，以及訊號如何傳遞：一旦每個傳送端都已關閉（透過 `close()` 或被釋放），通道即關閉。接收端隨後以 `recv()` 回傳 `none` 的形式收到關閉訊號。

以 `close()` 關閉通道：

```nabi
let (tx, rx) = channel::<i32>();
try tx.send(1);
tx.close();            // explicit close

rx.recv();             //→ 1
rx.recv();             //→ none  (closed)
```

`Sender` 是 `Drop` 資源（[2.13.4](213-marker.md#2134-drop釋放時的清理)）：`let tx2 = tx` 會移動它，`clone()` 會建立另一個有自己擁有者的傳送端，而 `close()` 只關閉該傳送端。編譯器會在傳送端最後一個擁有者離開作用域處釋放它，任務結束時也會釋放其 `async` 區塊所擷取的傳送端；每個傳送端都已關閉或釋放後，通道即關閉。釋放傳送端不會失敗，因此不需要明確關閉。執行緒的閉包不會釋放它所擷取的內容：請在執行緒工作結束時關閉傳送端。

## 2.17.4. 多個生產者與消費者

`Sender<T>` 實作了 `Clone`，因此多個任務可以向同一個通道傳送訊息。

```nabi
let (tx, rx) = channel::<str>();
let tx2 = tx.clone();

task::spawn(async { try tx.send("hello"); });
task::spawn(async { try tx2.send("world"); });

let v1 = rx.recv();
let v2 = rx.recv();
```

`Receiver<T>` 沒有實作 `Clone`，但它是控制代碼：其複本（`let rx2 = rx;`，或移動到其他任務或執行緒中的複本）共用同一個佇列。因此通道是多生產者、多消費者（MPMC）的。任意數量的接收端複本都可以同時呼叫 `recv`（或 `try_recv`、`recv_async`、`for`），而每個送出的值只會交給其中一個，也就是最先取走它的那個；沒有任何接收端能看到每個值。一旦每個傳送端都已關閉且佇列為空，每個複本都會得到 `none`。若要把每個值交給多個消費者，請使用扇出（fan-out）任務，將值轉送到每個消費者各自的通道。

## 2.17.5. `for` / `Stream` 整合

`Receiver<T>` 實作了 `IntoIterator<Item = T>`，因此可以直接用於 `for` 陳述式。

```nabi
for v in rx {
    process(v);                  // repeats until the channel is closed
}
```

它也實作了非同步迭代器 `Stream<Item = T>`，因此 `for await` 同樣可用。

```nabi
for await v in rx {
    await handle(v);
}
```

詳見 [2.7 iter](207-iter.md) 與 [2.8 stream](208-stream.md)。

## 2.17.6. 模組對應

每個型別與函式都定義在 `core::channel` 中，這是它們唯一的路徑（[0.1.6](../000-meta/001-tier-model.md#016-每個項目只有一條路徑)）：

| 型別           | 模組               | 內部結構       |
|----------------|----------------------|--------------------------|
| `Sender<T>`    | `core::channel` | 原子參考計數 + 佇列 |
| `Receiver<T>`  | `core::channel` | 佇列 + 停駐 |
| `channel<T>()`, `channel_bounded<T>(cap)` | `core::channel` | 類似 Arc 的內部狀態 |
| `SendError<T>`, `TrySendError<T>` | `core::channel` | 攜帶未送出值的錯誤 |
| `Empty`, `Closed` | `core::channel` | `try_recv` 的結果 |
| `ReceiverIter<T>` | `core::channel` | `for x in rx` 的迭代器 |
| `RecvFuture<T>`, `SendFuture<T>` | `core::channel` | `recv_async` / `send_async` 的 future |

在內部佇列方面，有界通道使用類似 Tokio mpsc 的環形緩衝區，無界通道則使用分段鏈結串列。有界通道的佇列是無鎖的；無界通道的分段串列則可能使用鎖。
