# 7.2. 記憶體配置

本文件涵蓋[物件配置](701-object-layout.md)中定義的物件如何放入記憶體。配置演算法分兩個階段定義：快速路徑與慢速路徑。

> [!NOTE]
> 本節僅供參考：它描述的是參考執行環境的 GC，可能在任何版本中改變。只有在 [8.1](../008-abi/801-object-header-abi.md) 中固定為 ABI 的物件配置具規範性。

## 7.2.1. TLAB（執行緒區域配置緩衝區）

每個排程器工作者（= 作業系統執行緒）都有自己的 TLAB。TLAB 由從 [Immix](../000-meta/007-glossary.md#immix) 空間取得的一個[區塊](../000-meta/007-glossary.md#區塊block)（或其一部分）組成，以 bump 指標快速配置。

```
TLAB structure:
  cursor : ptr      // next allocation position
  limit  : ptr      // end of the TLAB
  block  : ptr      // start (for reuse)
```

當 cursor 低於 limit 時，空間就足夠。若 cursor 加上 size 超過 limit，[TLAB](../000-meta/007-glossary.md#tlab執行緒區域配置緩衝區) 就已用盡，配置會落入慢速路徑。

## 7.2.2. 快速路徑：4 道行內指令

> [!IMPORTANT]
> 大多數配置由行內程式碼處理。它們在 4-5 道機器指令內完成，同步成本為零。

```
1. tmp = cursor
2. new_cursor = tmp + size
3. if new_cursor > limit → goto slow
4. cursor = new_cursor
   (return tmp)
```

這條路徑在 4-5 道機器指令內完成。分支幾乎總是 `false`，走快速路徑，而由於 TLAB 本身是執行緒區域的，同步成本為零。

快速路徑是編譯器在每個配置處產生的行內程式碼；執行環境不把它公開為函式呼叫。

## 7.2.3. 慢速路徑：`nabi_gc_alloc_slow`

用於 TLAB 已用盡或物件很大（[LOS](../000-meta/007-glossary.md#los大型物件空間)）時。

```
extern "C" {
    fn nabi_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

這是執行環境提供的 `extern "C"` 函式；快速路徑失敗時由編譯器呼叫它。

慢速路徑做下列事情：

1. 若物件為 8 KiB 以上，或其對齊大於 8 位元組，就在 LOS 中配置它（7.2.9）。
2. 大於一行、又放不進 TLAB 剩餘空間的物件，從該執行緒的中型範圍以 bump 方式配置；中型範圍是像 TLAB 一樣補充的第二段可用行，因此 TLAB 的尾端仍然可用。
3. 否則補充 TLAB。必要時先執行一次 [GC](../000-meta/007-glossary.md#gc垃圾回收器) 回收（7.2.8）。
4. 初始化標頭的 `type_id` 與 `gc_flags`。

## 7.2.4. 物件標頭的初始化

物件標頭在慢速路徑中初始化；在快速路徑上，編譯器也會在使用前以行內方式填入標頭。

```
*(ptr + 0) = type_id
*(ptr + 4) = 0u32          // initial gc_flags
*(ptr + 8) = size          // gc_word: total object size (8.1.4)
```

酬載已經是零，由使用者程式碼填入欄位值。回收器可以把自己的簿記資訊保存在標頭之前或另一張表中。

## 7.2.5. 對齊

每次配置都保證 8 位元組對齊。TLAB 的 cursor 保持在 8 位元組的倍數上，物件大小則向上取整為 8 位元組的倍數。需要更大對齊的物件（`#[repr(align(N))]`）在慢速路徑上處理。

## 7.2.6. TLAB

TLAB 是 Immix 區塊內一段連續的可用行範圍。補充時，該範圍在交出之前會被清零，因此快速路徑得到的是以零填滿的記憶體（剛從作業系統取得的區塊不需要清零）。

執行緒擁有整個區塊。在每次回收結束時，回收器列出可用的區塊，以及有可用行的部分使用區塊；需要區塊的執行緒透過原子游標取得清單的下一個項目（可用區塊優先），並從該區塊的可用段補充其 TLAB 與中型範圍。補充不需要任何鎖：只有擴大堆積與啟動回收才會取得 world 鎖。執行緒沒有用到的行，會在下一次回收時回到清單中。

## 7.2.7. TLAB 與工作竊取

[排程器](706-scheduler.md)的工作竊取機制讓一個工作者竊取另一個工作者的任務。

被竊取的任務從竊取者的 TLAB 配置，與原工作者的 TLAB 無關。

這個模型之所以可行，是因為每個 TLAB 都以執行緒區域的方式繫結到其作業系統執行緒。

## 7.2.8. GC 觸發

慢速路徑在下列情況下啟動回收。條件以及在 minor 與 full 之間的選擇遵循 [7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-分代回收)。

- 無法交出新的 TLAB 或區塊時。
- 自上次回收以來配置的量超過下一個週期的預算時。區塊在執行緒取得時以整塊計算。年輕代預算是上一個週期中每個有配置的執行緒 8 MiB，至少 32 MiB、最多 128 MiB；較大的堆積會得到與老年代一樣多的年輕代位元組，最多 128 MiB。`NABIR_GC_YOUNG` 可以精確設定它（[9.9.8](../009-tools/909-debug-tools.md#998-記憶體--gc-工具)）；有暫停目標時，它跟隨該目標（[bare 1.6.12](../../refs-standard-libraries/100-bare/106-gc-pin.md#1612-set_pause_goal---a-pause-goal)）。
- 堆積上限會限制預算（[bare 1.6.14](../../refs-standard-libraries/100-bare/106-gc-pin.md#1614-set_heap_limit---a-heap-limit)）。在不回收區域開啟期間，任何配置都不會回收：超過預算時改為擴大堆積（[bare 1.6.13](../../refs-standard-libraries/100-bare/106-gc-pin.md#1613-no_collect---a-region-without-collections)）。

`nabi_gc_collect` 明確地執行完整回收，`nabi_gc_collect_young` 執行 minor 回收。

## 7.2.9. 大型物件的 LOS 配置

8 KiB 以上，或需要大於 8 位元組之對齊的物件，略過 TLAB，在 LOS 中個別配置。把大型物件放在 TLAB 中會浪費剩餘的空間，而且由於 LOS 物件不會被移動，bump 配置也沒有好處。

對每個 8 KiB 以上的物件，有兩項保證：

- **對齊。**其酬載（緊接在 16 位元組標頭之後的位址）以 64 位元組（`LOS_ALIGN`）對齊，即一條快取線，足以應付每一種 SIMD 寬度。要求更多的緩衝區（`align > 64`）會得到它自己的對齊。
- **位址穩定。**回收器絕不移動它：只要該物件可到達，其酬載位址就保持不變。從它取得的指標（`Array::as_ptr`、`List::as_ptr`，[stdlib 2.9.12](../../refs-standard-libraries/200-core/209-collections.md#2912-gcbuft---raw-gc-buffer)）可以不經釘選就交給向量程式碼或 C（BLAS、LAPACK），前提是其擁有者在使用期間保持可到達。較小的物件可能會移動（疏散，7.3.1）；當其位址必須保持固定時，請釘選它（[7.2.10](#7210-釘選的物件)）。

## 7.2.10. 釘選的物件

在接收 GC 指標的 `extern "C"` 呼叫期間，不需要另外釘選（[7.4.1](704-barriers-and-roots.md#被根指向的物件不會被移動)）。若 C 在呼叫之後保留該指標，請以 `nabi_gc_pin` 釘選它。釘選只設定 `PINNED` 位元；它不會複製物件。`nabi_gc_unpin` 解除釘選。

## 7.2.11. 配置時間與堆積用量

快速路徑是常數時間（約 4 道指令）。平均配置成本由快速路徑決定。

## 7.2.12. 配置消除

最佳化器可以移除其物件從未逃逸、且在寫入之後從未被讀取的配置；這樣的配置不是安全點，也不計入回收器的統計。大小無法表示的配置（`len * size + header` 溢位）一律會到達執行環境的「allocation too large」。
