# 7. 執行環境概觀

這一部分定義 Nabi 執行環境系統（[GC](../000-meta/007-glossary.md#gc垃圾回收器)、任務排程器、協程）的實作規格。這個系統在[執行引擎與使用者介面](../000-meta/001-execution-modes.md)的兩個引擎（[JIT](../000-meta/007-glossary.md#jit即時編譯) / [AOT](../000-meta/007-glossary.md#aot預先編譯)）下運作方式完全相同。

## 目錄

本章的各節如下：

- [7.1. 物件配置與記憶體空間](701-object-layout.md)
- [7.2. 記憶體配置](702-allocation.md)
- [7.3. GC 演算法](703-gc-algorithm.md)
- [7.4. 屏障與根追蹤](704-barriers-and-roots.md)
- [7.5. 並行標記](705-concurrent-collection.md)
- [7.6. 排程器](706-scheduler.md)
- [7.7. I/O 驅動程式與計時器](707-io-and-timer.md)
- [7.8. 協程](708-coroutine.md)
- [7.9. 記憶體模型](709-memory-model.md)

## 關鍵決定

執行環境核心元件的決定整理如下：

```
GC          in-house Immix + Sticky Mark-Bit generations
Scheduler   M:N work-stealing
Coroutine   compiler-generated state machine (stackless)
```

三者都是自行實作的，對外部函式庫的相依為零。

## 規範性地位

GC 各節，即 7.1 到 7.5，僅供參考：它們描述參考執行環境目前如何配置與回收，之後的執行環境可以使用其他演算法、空間或屏障。具規範性的是在 [8.1](../008-abi/801-object-header-abi.md) 中固定為 ABI 的物件配置：每個 GC 物件的酬載之前有一個 16 位元組的標頭。函式庫程式碼依賴它：`Pinned::as_data_ptr` 與 `as_value_ptr`（[stdlib 1.6.6](../../refs-standard-libraries/100-bare/106-gc-pin.md#166-extraction-methods)）以物件位址加 16 計算酬載位址。[7.9](709-memory-model.md) 的記憶體模型也具規範性：它是程式與每個後端所依賴的東西。

## 零相依原則

執行環境不相依於外部函式庫。具體而言，它不使用：

- 外部的 GC 函式庫（Boehm GC、MMTk 等）；
- 外部的非同步執行環境；
- 外部的協程函式庫。

平台的系統介面（`libc`、作業系統標頭檔等）不算作相依，因為它們是作業系統介面的必要部分。

## M:N 模型

排程器遵循 M:N 混合模型。

```
OS Thread (worker) x N  ──── nabi task / coroutine x M
                                   (typically M >> N)
```

預設的 N 是 CPU 核心數，可以透過執行環境設定調整（`NABIR_WORKERS`，[7.6.8](706-scheduler.md#768-thread-per-core-選項)）。每個工作執行緒既是排程器的工作者，也是 mutator，而 stop-the-world 回收在啟動它的執行緒上執行（[7.6.10](706-scheduler.md#7610-整合的-gc-工作者角色)）。I/O 驅動程式是一個獨立的 reactor 執行緒（[7.7](707-io-and-timer.md)），並行標記在獨立的標記執行緒上執行（[7.5](705-concurrent-collection.md)）。

## 單一 .a 輸出

執行環境以單一的 staticlib `libnabir.a` 散布。建置使用者套件時會連結 `libnabir.a`，產生沒有其他外部相依的可執行二進位檔。

## 執行時錯誤：堆疊溢位

當執行緒的堆疊溢位時，執行環境會（盡可能）清空 stdout，把下列內容印到 stderr，然後中止（SIGABRT）。與 panic 不同，沒有展開，`finally` 也不會執行。

```
thread '<name>' has overflowed its stack
fatal runtime error: stack overflow, aborting
```

`<name>` 是執行緒的名稱：以 `thread::Builder::name`（[stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)）給出的名稱，它也被設為作業系統的執行緒名稱。主執行緒是 `main`，未命名的執行緒是 `<unnamed>`。這適用於主執行緒、`thread::spawn` 執行緒以及執行器工作者（包括很深的 `await` 鏈）。偵測機制（保護範圍與 `sigaltstack`）與 TLS 槽位見 [8.5.7](../008-abi/805-extern-symbols.md#堆疊溢位)。在 wasm 上，模組無法捕捉這個 trap；由主機回報它（隨附的 JS 主機會印出上述訊息）。

## 與編譯器的介面

執行環境透過固定的 [ABI](../000-meta/007-glossary.md#abi應用程式二進位介面) 與編譯器產生的程式碼溝通。介面由三個元素組成：

- 物件配置（16 位元組標頭 + 酬載）。
- `extern "C"` 函式，例如 `nabi_gc_alloc_slow`、`nabi_runtime_yield`、`nabi_thread_spawn`。
- 協程是編譯器產生的狀態機（[7.8](708-coroutine.md)）。對於 GC 根，編譯器直接發出影子堆疊框架（[7.4.1](704-barriers-and-roots.md#741-根追蹤由編譯器直接發出的影子堆疊)）。

這個 ABI 詳細定義於 [8. 執行環境 ABI / 降階](../008-abi/README.md)；本章只涵蓋內部行為。
