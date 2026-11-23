# 7.6. 排程器

Nabi 的排程器是 M:N [工作竊取](../000-meta/007-glossary.md#工作竊取work-stealing)；本文件定義其結構。

## 7.6.1. 統一的工作者模型

一個行程有一個執行器，在第一次使用時啟動。工作者是與 CPU 數量一樣多的作業系統執行緒，加上一條阻塞通道與一個 I/O 驅動程式。`task::block_on` 把呼叫它的執行緒附加為訪客工作者；在它回傳之後，剩下的任務繼續執行。`par::*` 與 `spawn_blocking` 使用同一個執行器（[stdlib 2.18.3](../../refs-standard-libraries/200-core/218-task.md#2183-排程器模型工作竊取)）。每個工作者結合了下列角色：

```
worker = {
    scheduler   // task queue (work-stealing)
    mutator     // TLAB + shadow stack (701 / 702 / 704)
}
```

工作者不兼任 I/O 驅動程式；每個行程由單一的 reactor 執行緒處理它（[7.7.1](707-io-and-timer.md#771-io-多工)）。STW 回收由啟動該回收的執行緒執行，並行標記則由標記執行緒執行（7.6.10）。

## 7.6.2. 工作竊取結構

每個工作者都有一個區域雙端佇列（無鎖的 Chase-Lev 雙端佇列，[8.5.4 `nabi_deque_*`](../008-abi/805-extern-symbols.md#nabi_deque_工作竊取雙端佇列)）。

```
Worker:
  ├── local_deque : growable ring    (this worker's tasks, GC-traced)
  └── ...
```

新喚醒或新啟動的任務會放到該工作者雙端佇列的底部。**擁有它的工作者與竊取者都先取最舊的任務（頂部）**。若擁有者先取最新的任務，一個不斷重新喚醒自己的任務（例如在 `await task::yield_now()` 上迴圈的任務）就可能讓排在它後面的任務餓死。

這是一種工作竊取模型：工作者通常執行它自己啟動的任務，以保持快取區域性，而閒置的工作者竊取工作，以達成負載平衡。實作可以加入 LIFO 槽位之類「最新任務優先」的最佳化，只要不會發生上述的餓死情況。

## 7.6.3. 全域注入佇列

除了每個工作者的區域佇列之外，還有一個獨立的全域注入佇列。

```
Global injector:
  - receives tasks spawned from outside (e.g. extern callback).
  - when a worker's local queue is empty, pull from global.
```

這個佇列是來自工作者池之外之任務的進入點。

## 7.6.4. 竊取演算法

當工作者的區域雙端佇列為空時，它依下列順序進行：

```
1. Try to pull from the global injector.
2. Try to steal from the top of another worker's local deque. Victims start at a per-worker pseudo-random index, then go round-robin.
3. If all fail, sleep / park (blocking sync).
```

竊取最多嘗試 N-1 個工作者；若全都是空的，該工作者切換為停駐狀態。

## 7.6.5. 關鍵字與函式庫函式

依照[關鍵字](../001-lexical/105-keywords.md)中盡量減少保留關鍵字的原則，排程器的函式全都以函式庫函式提供。

| 函式庫函式              | 意義                                                                     |
|-------------------------|--------------------------------------------------------------------------|
| `task::spawn(fut)`      | 把 future 作為新任務推入佇列。結果為 `Task<T>`。                         |
| `task::yield_now()`     | 自願讓出（`await`）。讓其他就緒的任務執行一次。                          |
| `task::sleep(d)`        | 移到計時器佇列。                                                         |
| `task::race([...])`     | 最先完成的任務；其他的被取消（await）。                                  |
| `task::join([...])`     | 直到全部完成（`List<Task<T>>`，await）。                                 |
| `task::timeout(d, t)`   | 時間限制。若超過，就在 `try await` 處得到 `TimeoutError`。               |
| `thread::spawn(f)`      | 建立新的作業系統執行緒（1:1）。                                          |
| `par::for_each(xs, f)`  | 平行的 for-each（所有呼叫都完成後才回傳）。 |
| `par::scope { ... }`    | 平行的作用域（尾隨閉包）。                                               |

由於這些不是關鍵字，`spawn` 或 `race` 之類的識別字可以在使用者程式碼中自由使用。

## 7.6.6. 1:1 的作業系統執行緒：`thread::spawn`

大多數任務以排程器所對應的 M:N 方式執行，但對於阻塞的工作或外部函式庫呼叫，有時專用的作業系統執行緒才是自然的選擇。此時 `thread::spawn` 建立一個以 1:1 對應的真正作業系統執行緒。

在內部，它直接呼叫 Unix 的 `pthread_create`、Windows 的 `CreateThread` 等，並在與一般任務分開的池中執行。結果以 `JoinHandle<T>` 回傳，這是與 Future 不同的型別。

## 7.6.7. 自動偵測阻塞的系統呼叫

工作竊取模型有一個問題：當任務呼叫阻塞的系統呼叫時，工作者本身會停滯。這以自動卸載到阻塞池來解決。

編譯器與執行環境會自動把已知的阻塞系統呼叫送到阻塞池的工作者，那是獨立的作業系統執行緒，因此不會妨礙其他任務的進展。詳細機制交給排程器的實作。

> [!NOTE]
> 在參考執行環境中，任務停留在阻塞呼叫中的池執行緒會把其工作者交給一個新的執行緒。以 `spawn_local` 啟動的任務會固定其工作者，該工作者絕不會被交出去。

## 7.6.8. thread-per-core 選項

預設有 N 個工作者，任務在執行緒之間自由移動。啟用 thread-per-core 選項時，每個任務都固定在某個工作者上，工作者之間不互相竊取；快取區域性達到最大，適合 NUMA 環境與資料中心的工作負載。

它以環境變數 `NABIR_THREAD_PER_CORE=1` 切換；`NABIR_WORKERS=<n>` 設定 N（預設：線上的 CPU 數）。

> [!NOTE]
> 任務被固定在一個工作者上；其任務阻塞的工作者可能在另一個作業系統執行緒上繼續（[7.6.7](#767-自動偵測阻塞的系統呼叫)），因此對被固定的任務而言，執行緒的身分並不穩定。

## 7.6.9. 優先順序 / 公平性

預設是沒有優先順序的 FIFO + 工作竊取。擁有它的工作者與竊取者都先處理雙端佇列中最舊的任務（7.6.2）。

忙碌的工作者也不會延後計時器。每執行 32 個任務，它就處理已到期的計時器，並把其任務放回佇列。因此即使在一直有工作的工作者上，`task::sleep` / `task::timeout` 也會準時喚醒。

任務沒有優先順序：執行環境與標準函式庫都不提供設定優先順序的方法。應該讓路的任務呼叫 `task::yield_now`（[stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)），而不能排在其他任務之後等待、對延遲很敏感的工作，則在自己的執行緒上執行（`thread::spawn`，[stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)）。

## 7.6.10. 整合的 GC 工作者角色

STW 回收由透過配置或 `nabi_gc_collect` 啟動它的執行緒執行，其他工作者則在安全點停下等待（[7.4.3](704-barriers-and-roots.md#743-安全點輪詢)）。回收之後，工作者回去執行任務。並行標記由單一的獨立標記執行緒進行（[7.5](705-concurrent-collection.md)）。

## 7.6.11. 排程器與協程

排程器使用[協程](708-coroutine.md)的無堆疊協程；每個任務的狀態機存放在協程框架中。

當任務遇到 `await` 或 `yield` 時，只有協程框架會回到佇列中，工作者則繼續執行另一個任務；當任務繼續時，可能由另一個工作者取得該框架並繼續它。這個模型讓數百萬個任務能在 N 個工作者上有效率地處理。

## 7.6.12. 啟動任務的成本

`task::spawn` 只需配置一個無堆疊協程框架就能完成。框架的大小取決於任務的區域變數，大約 64-256 位元組，並在 [TLAB](../000-meta/007-glossary.md#tlab執行緒區域配置緩衝區) 快速路徑上配置，因此成本接近於零。

結果是，即使是數百萬個並行任務，也比作業系統執行緒輕量得多。

## 7.6.13. 整體流程

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
