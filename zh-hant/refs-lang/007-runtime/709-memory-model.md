# 7.9. 記憶體模型

本節定義一個執行緒能看到其他執行緒所寫入之記憶體的什麼。Nabi 採用 C++20（ISO/IEC 14882:2020，[intro.multithread] 與 [atomics.order]）的記憶體模型，Rust 也使用這個模型；下列術語具有其在 C++20 中的意義。在其之上，本節說明哪些函式庫操作會同步，以及安全的 Nabi 保證了什麼。與 7.1 到 7.5 不同，本節具規範性。

這裡的「執行緒」指作業系統執行緒。任務（[stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)）在工作執行緒上執行，並可能在任何暫停點之後於另一個工作者上恢復；排程器會為任務自身的各步驟排序，因此同一個任務中的程式碼無論在哪裡恢復，都看得到它自己先前的寫入。兩個任務之間適用下列規則，就像兩個執行緒之間一樣。

## 7.9.1. 關係

- **Sequenced before**：在一個執行緒內，依求值順序（[3. 運算式](../003-expressions/README.md)）排在另一個求值之前的求值。
- **Modification order**：每個原子物件對其所有修改都有單一的全序，每個執行緒都一致地觀察到這個順序（C++20 的一致性規則）。
- **Synchronizes with**：對原子物件的 release 操作，與對同一物件、讀取它所寫入之值（或其 release 序列中之後的值）的 acquire 操作同步。7.9.3 的函式庫操作也依該處的說明同步。
- **Happens before**：sequenced before 與 synchronizes with 的遞移閉包（即 C++20 的 simply-happens-before；Nabi 沒有 `consume` 順序，因此這兩個概念一致）。

非原子的讀取會看到對該位置、happens before 它的最後一次寫入，前提是沒有其他對它的寫入與該讀取之間沒有順序關係。

## 7.9.2. 原子順序

`Atomic<T>`（[stdlib 2.16.2](../../refs-standard-libraries/200-core/216-sync.md#2162-atomict---lock-free-atomics)）接受 C++20 的五種順序，意義與 C++20 相同：

| 順序      | C++20                        | 意義 |
|-----------|------------------------------|---------|
| `Relaxed` | `memory_order_relaxed`       | 原子的，並位於修改順序中；沒有同步。 |
| `Acquire` | `memory_order_acquire`       | 與它所讀取之 release 同步的載入（或讀取-修改-寫入的載入部分）。 |
| `Release` | `memory_order_release`       | 作為 release 序列開頭的存入（或讀取-修改-寫入的存入部分）。 |
| `AcqRel`  | `memory_order_acq_rel`       | 兩者兼具的讀取-修改-寫入。 |
| `SeqCst`  | `memory_order_seq_cst`       | 對載入是 acquire，對存入是 release，並且屬於所有 `SeqCst` 操作的單一全序。預設值。 |

失敗的比較交換是以其失敗順序進行的載入。每個操作的順序型別只接受它能採用的順序（2.16.2），因此 `consume` 與 acquire 存入並不存在。對同一個物件的原子操作，無論其順序為何，都絕不會撕裂，彼此之間也絕不會形成資料競爭。

Nabi 沒有獨立的記憶體柵欄：每個順序都附加在某個原子操作上。`volatile` 存取（[stdlib 1.5](../../refs-standard-libraries/100-bare/105-volatile.md)）不是原子的，不會在執行緒之間排序任何東西。

## 7.9.3. 會同步的操作

下列每一項都讓較早的操作與較晚的操作同步，因此 sequenced before 前者的一切，都 happens before sequenced after 後者的一切。

| 較早 | 較晚 |
|---------|-------|
| `Mutex` 解鎖（守衛的釋放） | 該 `Mutex` 下一次成功的上鎖（`lock`、`try_lock`、`lock_async`） |
| `RwLock` 寫入解鎖 | 下一次 `read` / `write` 取得 |
| `RwLock` 讀取解鎖 | 下一次 `write` 取得 |
| `Condvar::wait` 釋放其鎖 | 視為 `Mutex` 解鎖；其重新上鎖視為 `Mutex` 上鎖 |
| `Once::call_once` 的 `f` 回傳 | 該 `Once` 的每次 `call_once` 回傳，以及回傳 `true` 的 `is_completed` |
| 延遲初始化之 `static` 的初始化式完成 | 該 `static` 的每次存取（[5.3.9](../005-modules/503-static.md#539-初始化式編譯期或第一次使用)） |
| 在通道上 `send` / `try_send` / `send_async` 一個值 | 接收到該值的 `recv`（[stdlib 2.17](../../refs-standard-libraries/200-core/217-channel.md)） |
| `thread::spawn` / `Builder::spawn` | 新執行緒的第一步 |
| 執行緒的最後一步 | 回傳其結果的 `join`（或對其 `JoinHandle` 的 `await`）（[stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)） |
| `task::spawn` / `spawn_blocking` | 任務的第一步 |
| 任務的最後一步 | 回傳結果的、對其 `Task` 的 `await`（[stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)） |
| `par` 呼叫或 `par::scope` 的開始 | 其每項工作的第一步 |
| 每項 `par` 工作的最後一步 | `par` 呼叫的回傳，或作用域的結束（[stdlib 2.19](../../refs-standard-libraries/200-core/219-par.md)） |

在參考函式庫中，鎖、`Once` 與通道都建立在 `SeqCst` atomics 之上，因此上述順序也屬於 `SeqCst` 全序的一部分。`Atomic<i32>::wait` / `wake` 只會阻塞與喚醒：它們本身不同步任何東西，而 `wait` 所比較的值以該 atomic 自己的順序讀取。

回收器、其寫入屏障（[7.4.2](704-barriers-and-roots.md#742-寫入屏障物件記錄)）以及 stop-the-world 暫停只與回收器同步。它們不為使用者程式碼提供任何 happens-before 關係，即使它們恰好在硬體層面形成了柵欄。

## 7.9.4. 資料競爭

當對同一記憶體位置的兩次存取中至少有一次是寫入，而且兩者並非都是原子的時候，它們就**衝突**。當兩個衝突的存取發生在不同的執行緒中，而且兩者都不 happens before 另一者時，程式就有**資料競爭**。資料競爭是未定義行為（[4.15.8](../004-functions/415-unsafe.md#4158-未定義行為)）。

資料競爭只能透過 `unsafe` 所放行的程式碼發生。安全的 Nabi 沒有資料競爭，理由如下：

- `static` 可以從每個執行緒存取，因此其型別必須是 `Sync`（[5.3.1](../005-modules/503-static.md#531-基本形式)）；`static mut` 只存在於 `extern "C"` 區塊中，存取它需要 `unsafe`。
- 集合型別（`List`、`Dict`、`StrBuf`……）是 `!Sync`，`Cell<T>`、原始指標與守衛也是（[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)）。共享的可變狀態只能放在 `Mutex`、`RwLock`、`Atomic` 或通道內跨越執行緒。凍結的集合（`xs.freeze()`，[stdlib 2.9.7b](../../refs-standard-libraries/200-core/209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s---read-only-collections)）是 `Sync`，因為沒有任何東西寫入它：沒有任何可變更的控制代碼能到達其儲存空間。
- 把值交給另一個擁有者（`Mutex::new`、`send`、`task::spawn`、`[move x]`、執行緒的結果……）會讓新的擁有者得到該值所能到達之每個集合的自己的複本，而之後使用傳送者的值是編譯錯誤（2.13.9）。被產生之閉包的一般擷取必須是 `Send + Sync`。
- `par` 在執行時檢查兩個項目不會到達同一個不是 `Sync` 的集合，並一次執行一個這樣的項目（[stdlib 2.19.1](../../refs-standard-libraries/200-core/219-par.md#2191-core-api)）。項目只有在其他項目的視圖或切片都不與自己寫入的部分重疊之處，才能透過視圖（`chunks_mut`）寫入同一個串列。

使資料競爭成為可能、因而需要作者小心的東西：

- 透過 `Ptr<T>`、`ManyPtr<T>`、`CPtr<T>` 與 `GcBuf<T>` 的讀取與寫入（[stdlib 1.1](../../refs-standard-libraries/100-bare/101-ptr.md)、[stdlib 1.3](../../refs-standard-libraries/100-bare/103-mem.md)）；
- 在其共享實際上並未受保護的型別上的 `unsafe impl Send` 或 `unsafe impl Sync`（[4.15.4](../004-functions/415-unsafe.md#4154-在-trait-中)）；
- 對其他東西仍然能到達的值使用 `Mutex::new_unshared`（[stdlib 2.16.3](../../refs-standard-libraries/200-core/216-sync.md#2163-mutext---exclusive-lock)）；
- 在 `unsafe` 內略過的交接檢查（2.13.9）；
- `extern static mut` 以及透過 FFI 存取的外部程式碼（[5.9](../005-modules/509-extern-block.md)），包括在 `env::set_var` 執行期間呼叫 `getenv` / `setenv` 的 C 程式碼（[stdlib 3.5](../../refs-standard-libraries/300-std/305-env.md)）。

## 7.9.5. GC 參照

GC 參照是一個對齊的機器字組，以一次存取讀取與寫入。回收器依賴這一點：它絕不會遇到寫到一半的參照，並把它所掃描的每個參照字組都視為一個物件或 null。這並不讓競爭的參照存取成為有定義的。對參照的資料競爭與其他任何資料競爭一樣是未定義行為，而 `str`（指標與長度，兩個字組）或結構在競爭時可能撕裂。
