# 7.7. I/O 驅動程式與計時器

[排程器](706-scheduler.md)的工作者使用兩種機制，有效率地處理等待 I/O 與等待計時器的任務。

## 7.7.1. I/O 多工

每個行程有**一個 reactor 執行緒**，它在作業系統的就緒多工器上等待，並喚醒等待就緒 fd 的任務。

| 作業系統 / 環境       | 機制                                |
|-----------------------|-------------------------------------|
| Linux                 | epoll（`EPOLLET`，邊緣觸發）        |
| BSD（FreeBSD / macOS）| kqueue（`EV_CLEAR`，邊緣觸發）      |
| Windows               | IOCP + AFD poll（模擬就緒）         |
| WASI（`wasm32-wasip2`）  | `wasi:io/poll`                      |
| WASI（`wasm32-wasip1-threads`） | `poll_oneoff`                |

目標的驅動程式在建置時透過[條件式編譯](../006-meta-system/603-cfg.md)的 `target_os` 分支編譯進來。

reactor 執行緒在第一次非同步 I/O 等待時啟動。

在 `wasm32-wasip2`（preview 2 元件）上，驅動程式是 `wasi:io/poll`，而當主機授予網路存取權時，`std::net` 透過 `wasi:sockets` 運作。在 `wasm32-wasip1-threads`（preview 1）上，`poll_oneoff` 監看主機傳入的通訊端。

## 7.7.2. I/O 等待模式

I/O 等待依下列方式進行：

```
1. The task tries read on a non-blocking socket → WouldBlock.
2. The task awaits fd's readiness: (re)registers fd with the reactor and parks.
3. The worker runs another task.
4. When the reactor receives a readiness edge for fd, it unparks the task parked on that fd.
5. Some worker takes that task and retries the read.
```

這個流程與 `async` `fn` 中的 `await` 結合（見[非同步](../004-functions/412-async.md)）。註冊是邊緣觸發的，因此每次等待都會重新啟用。標準函式庫的就緒 future 在停駐之後會再檢查一次 fd，以免錯過期間經過的邊緣。被喚醒的任務可能發現 fd 尚未就緒，因此呼叫者會重試該操作。那些 future（`task::readable` / `task::writable`）是 `#[internal]`（[stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)）：手寫的 future 透過 `nabi_io_register` 等待 fd（[8.5.5](../008-abi/805-extern-symbols.md#855-io--計時器區域)）。

阻塞的呼叫（`TcpStream::read` 等）不使用這條路徑；它們會阻塞呼叫它們的作業系統執行緒。暫停任務的 I/O 由標準函式庫的 `*_async` 方法提供（[stdlib 3.2.0](../../refs-standard-libraries/300-std/302-net.md#320-common-rules)）。

一般檔案無法以就緒模型等待（在 epoll / kqueue 之下它們一律是就緒的）。因此一般檔案的非同步 I/O（`fs::*_async`，[stdlib 3.1.4a](../../refs-standard-libraries/300-std/301-fs.md#314a-asynchronous-file-io-_async)）在另一個作業系統執行緒上執行阻塞的系統呼叫（`task::spawn_blocking`），並在此期間暫停任務。

## 7.7.3. I/O 驅動程式的抽象

每個驅動程式都實作一個共同的介面。

```
trait IODriver {
    fn register(ref self, fd: Fd, interest: IoInterest) throws (IoError);   // edge-triggered
    fn poll(ref self, timeout: Duration) -> List<Event>;
}
```

這個介面是概念上的；確切的簽章由執行環境的實作決定。驅動程式不持有任何 waker。要喚醒的任務透過由 fd 與方向（讀 / 寫）決定的停駐鍵，在 `core::task` 的停駐表中找到。執行環境符號見 [8.5.5](../008-abi/805-extern-symbols.md#855-io--計時器區域)。

## 7.7.4. 計時器結構

`task::sleep(d)` / `task::timeout(d, ..)` 的等待保存在依截止時間排序的二元堆積中。截止時間過去時，任務會被喚醒。已取消之任務的計時器留在堆積中，在到期或清理時被捨棄。在處理計時器時截止時間已過的 sleep，會依截止時間的順序喚醒，即使多個執行緒同時處理計時器也一樣；在第一次輪詢時截止時間就已過的 sleep 會在那時完成。最早的截止時間也保存在一個原子字組中，因此沒有到期計時器的工作者不會鎖住堆積，而只有成為最早者的計時器才會喚醒睡眠中的工作者（其他的計時器無論如何都會睡到更早的截止時間）。

插入與到期處理是 `O(log n)`。

## 7.7.5. 計時器的精度

沒有固定的節拍。截止時間以奈秒保存，而沒有東西可執行的工作者會在作業系統的計時等待中睡到下一個截止時間，因此計時器觸發的精度就是該等待喚醒的精度。

## 7.7.6. 整合計時器與 I/O

工作者的主要 `loop` 如下。對 I/O 就緒而言，reactor 執行緒透過停駐表把任務放回佇列，因此工作者不直接輪詢驅動程式。

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

reactor 執行緒的 `loop` 如下：

```nabi
loop {
    let events = io_driver.poll();                     // blocks; a native section for the GC
    for event in events {
        task::unpark(parking_key(event.fd, event.direction));
    }
}
```

## 7.7.7. 停駐 / 解除停駐

當工作者的佇列完全為空且 I/O 閒置時，工作者會以作業系統的睡眠停駐。

- 停駐使用 `pthread_cond_wait` 或作業系統特有的睡眠機制。
- 解除停駐由另一個工作者或 I/O 事件觸發。

停駐期間 CPU 使用率為零，因此許多閒置的工作者不會帶來額外成本。

## 7.7.8. 工作者數量的動態調整

預設情況下，N 固定為 CPU 核心數。

啟用時（`NABIR_DYNAMIC_WORKERS=1`），過度閒置的工作者會停駐，而推入的工作會先喚醒其他工作者；當每個作用中的工作者都忙碌時，它會被重新啟用。在每種模式下，阻塞在呼叫中的工作者都會被替換（[7.6.7](706-scheduler.md#767-自動偵測阻塞的系統呼叫)）。

## 7.7.9. 取消

`Task::cancel` **請求**取消（[stdlib 2.18.4](../../refs-standard-libraries/200-core/218-task.md#2184-cooperative-cancellation---taskcancel)）。執行環境設定一個旗標並喚醒該任務。

- 每個 `await` 都是取消點。`await` 在進入時以及即將暫停之前檢查旗標。在任務被輪詢期間到達的取消，會在其下一次輪詢時處理，而那次輪詢正是取消的喚醒所帶來的：被 await 的 future 會先看到它，因此巢狀的協程，以及 `task::select` 每個已啟動的子項，都會執行其清理。
- 在被 await 的 future 回傳 `Ready` 之後的檢查，不是「旗標是否已設定」，而是「**被 await 的協程是否因取消而離開**」。因此 future 所交出的值（收到的訊息、取得的鎖守衛）會被保留，而任務在下一個 `await` 處離開。
- 被取消的任務在下一個 `await` 處像 `return` 一樣離開其主體。所經過的 `finally` 區塊與 `using` 的 `drop` 從最內層向外執行。巢狀的 `async fn` 呼叫從最內層的協程開始清理。
- 當協程正在執行其取消的清理時，其中及其所 await 之物中的任何 `await` 都不會檢查旗標：清理會完成。執行環境持續輪詢該任務直到清理結束；外層協程在內層協程完成其清理時離開。
- 這不是可以在任務內 `catch` 的錯誤。不會擲回錯誤的非同步主體也以相同的路徑結束。結果會被捨棄。
- 在其最後一個取消點之後才被取消的任務會正常完成並有結果（`await` / `join` 回傳該結果）。`CancellationError` 只適用於沒有結果就結束的任務。
- 阻塞的系統呼叫（`TcpStream::read` 等）不是取消點。任務在呼叫結束之後的下一個 `await` 處離開。
- 在通訊端 I/O、`sleep` / `timeout`、`lock_async` 或 `recv_async` / `send_async` 中等待的任務，在被取消時會被喚醒，並在下一個 `await` 處離開。
- 任務結束時，它所有的喚醒註冊（I/O 就緒等待、計時器、停駐表中的 waker）都會被釋放。執行中之池的註冊數量可以用診斷函式 `task::registrations()` 檢視；該值的確切意義由實作定義。

在 `task::timeout` 到期或 `task::race` 決定勝者之後，落敗的任務會繼續執行。要停止它們，請呼叫 `cancel`。`task::select` 不會產生任務，因此它自己取消落敗的 future（[stdlib 2.18.1](../../refs-standard-libraries/200-core/218-task.md#select---the-first-future-to-finish)）；清理中會 `await` 的落敗者，會作為它自己的任務完成清理。

## 7.7.10. 與 reactor 模式的比較

Nabi 使用傳統的 reactor 結構：單一的獨立 reactor 執行緒等待 I/O，只透過停駐表與工作者執行緒接觸。代價是 I/O 喚醒要跨越執行緒（一次解除停駐）。
