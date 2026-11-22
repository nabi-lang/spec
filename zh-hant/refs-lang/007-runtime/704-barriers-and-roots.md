# 7.4. 屏障與根追蹤

本文件定義 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 的標記起點（根），以及老 → 年輕參照的記錄（寫入屏障）。

> [!NOTE]
> 本節僅供參考：它描述的是參考執行環境的 GC，可能在任何版本中改變。只有在 [8.1](../008-abi/801-object-header-abi.md) 中固定為 ABI 的物件配置具規範性。

## 7.4.1. 根追蹤：由編譯器直接發出的影子堆疊

### 選擇影子堆疊

最常見的兩種根追蹤方法如下：

| 方法                        | 優點                          | 缺點                                |
|-----------------------------|-------------------------------|-------------------------------------|
| 堆疊對應表                  | 精確，每次呼叫不需額外工作    | 每個目標都需要一種對應表格式與堆疊走訪；WASM 無法走訪其堆疊 |
| 影子堆疊（直接發出）        | 每個目標都是相同的程式碼      | 額外的堆疊記憶體與 push/pop         |

支援的[平台目標](../000-meta/006-platform-targets.md)包括 32 位元 ARM（`armv7`）、32 位元 x86（`i686`）與 WASM。WASM 程式碼無法檢視自己的堆疊，因此唯一能適用於所有這些目標的方法是[影子堆疊](../000-meta/007-glossary.md#shadow-stack)，每個執行緒一條鏈。

> [!IMPORTANT]
> Nabi 的根追蹤是由編譯器直接發出框架 push / pop 的影子堆疊。鏈的開頭是 `nabi_tls` 偏移 0 的槽位（`nabi_tls_shadow_stack_top`），每個執行緒一個（[8.3.8](../008-abi/803-tls-slots.md#838-form-of-compiler-emission)）。它在每個支援的目標（包括 32 位元 ARM / x86 / WASM）上的運作方式都相同。

### 影子堆疊的結構

每個作業系統執行緒都有自己的影子堆疊：一條透過 `prev` 串接、放在各函式機器堆疊上之框架的鏈結串列。

```
nabi_tls_shadow_stack_top ──► frame N      (current function)
                             ├── prev ──► frame N-1 (caller) ──► ... ──► null
                             ├── count = 2
                             ├── slots[0]  ──► &word (local x)      ──► GC object
                             └── slots[1]  ──► &word (spill slot)   ──► GC object
```

```
ShadowFrame {
    prev:  ptr          // caller's frame (null if none)
    count: usize        // number of slots (pointer width)
    slots: [ptr; count] // addresses of words holding GC references
}
```

在 `-O0` 以上，後面接著一個位址的槽位 `(n << 1) | 1` 代表從該位址起的 `n` 個相鄰字組，而 `count` 為 0 的框架以其唯一的槽位指向一個靜態描述子，該描述子以 16 位元的項目存放清單，每個位址都是相對於框架的偏移（[8.9.12](../008-abi/809-lowering-control.md#8912-shadow-stack-frame)）。

`slots` 存放的不是 GC 參照值，而是**容納 GC 參照之字組的位址**。這樣的字組是區域變數的堆疊槽位或溢出槽位；陣列元素或 `struct` 欄位內的字組也以相同方式被指向。因此指派給變數不需要另外更新框架。

### 編譯器的發出規則

函式只有在容納 GC 參照、而且可能到達安全點時（[8.4.2](../008-abi/804-safepoint.md#842-emission-sites)）才需要框架。對永不回傳之函式（panic）以及既不配置記憶體也不阻塞之執行環境函式（[8.5.2](../008-abi/805-extern-symbols.md#functions-that-are-not-safepoints)）的呼叫不算在內。對這樣的函式，編譯器直接發出下列程式碼：

只有跨越安全點而存活的區域變數才需要根。

1. 進入時，把 GC 參照字組清零。
2. 在機器堆疊上保留一個 `ShadowFrame`。在函式中每條路徑的第一個安全點之前，把那些字組的位址記錄在 `slots` 中，把目前的 `nabi_tls_shadow_stack_top` 存入 `prev`，然後把開頭設為這個框架（連上）。連上可以更早進行，最早到函式的進入處；在路徑匯合之處，要麼所有路徑都已連上，要麼都沒有。不會到達任何安全點的路徑不需要連上。
3. 在已連上框架之路徑上的每個 `return` 之前，立即把開頭還原為 `prev`（解開）。錯誤回傳也是一般的 `return`，因此走相同的路徑。panic 會終止行程，因此不需要解開。
4. 只跨越回邊輪詢（[8.4.8](../008-abi/804-safepoint.md#848-loop-back-edge-poll)）而存活、不跨越其他任何安全點的區域變數，可以改為只在輪詢的慢速路徑上註冊：放在第二個框架中，其字組由慢速路徑在連上該框架並呼叫執行環境之前寫入（區域變數已死亡之處寫 null），並在之後立即解開。這樣的字組不會在其他任何地方被寫入。在那條路徑上尚未連上的規則 2 框架，會與它一起連上與解開。這建立在一條規則之上：執行緒只會在編譯器所計算的安全點（呼叫、配置、進入輪詢）或回邊輪詢處為回收而停下。因此不是安全點的執行環境函式（[8.5.2](../008-abi/805-extern-symbols.md#functions-that-are-not-safepoints)）絕不能等待停止：它既不配置記憶體、不阻塞，也不輪詢。

當變數不再被使用時（依存活性），其根槽位所指向的字組可以被清為 null。只要那個字組不會再被讀取，這就不會改變意義，而且能防止框架讓死亡的物件繼續存活。

標記時，GC 從每個執行緒的鏈開頭開始走訪框架，並把每個 `slots[i]` 所指向之字組的值當作根標記。

### 被根指向的物件不會被移動

被根直接指向的物件（影子堆疊、`static`、註冊的執行緒根）在該次回收中不會被移動。因此傳給阻塞之執行環境呼叫或 `extern "C"` 函式的 GC 指標，**只要該引數在呼叫結束之前都位於根中**，就會保持在原位。若 C 程式碼在呼叫結束之後保留該指標，就必須以 `nabi_gc_pin` 釘選它（[8.5.1](../008-abi/805-extern-symbols.md#851-gc-area)）。

協程框架是 GC 堆積物件（[7.8](708-coroutine.md)），因此其中的變數透過框架物件的掃描表追蹤，而不是透過影子堆疊。

### 與回收器的契約

根（本節的影子堆疊）與寫入屏障（[7.4.2](#742-write-barrier-object-remembering)）依此處的規定發出。回收器不對編譯器的輸出要求其他任何東西。

### 與錯誤傳播並存

[錯誤控制流程](../003-expressions/314-error-control-flow.md)的錯誤透過會擲回錯誤之函式的回傳值交付（[8.6.5](../008-abi/806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)）。錯誤傳播時，每個框架仍經由一般的回傳路徑離開，並彈出其影子堆疊項目，因此影子堆疊始終與實際的呼叫堆疊一致，不需要為錯誤處理做任何特別的調整。

## 7.4.2. 寫入屏障：物件記錄

### 為什麼需要寫入屏障

minor 回收不會進入老物件（[7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-generational-collection)）。當老物件開始指向年輕物件時，寫入屏障會通知回收器。否則那個年輕物件就會被回收。

### 物件記錄

屏障**只檢查目的物件 `p` 的標頭**。存活下來的（老）物件除了 `STICKY` 之外還設定了 `UNLOGGED` 位元（[8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)）；在第一次寫入該物件時，整個物件會被記錄到記憶集中。

```
*p = q;
                                // ↓ barrier (may be before the store)
if (p.gc_flags & UNLOGGED) {
    nabi_gc_wb_slow(p, &p.f, len);  // clear UNLOGGED and put p in the remembered set
}
```

- 不會讀取被存入之值 `q` 的標頭（`q.sticky` 等）。`q` 可能是 `any` 槽位中的行內整數，或 `str` 的內部指標，因此把它當作物件標頭讀取是不安全的。
- 屏障可以放在存入之前。
- 慢速路徑 `nabi_gc_wb_slow(obj, slot, len)` 接受物件、即將被寫入之槽位的位址，以及被存入之值的大小。它以原子的 `fetch_and` 清除 `UNLOGGED`。在並行標記期間，它也會把物件目前的參照記錄到 SATB 緩衝區（[7.5.3](705-concurrent-collection.md#753-satb-barrier)）；對大型物件而言，只記錄 `slot..slot + len` 所跨越之卡片中的參照。因此即使多個執行緒同時寫入，也只有一個執行緒會記錄該物件。慢速路徑不得是安全點（回收不得在其中開始）。
- 年輕代 GC 把記憶集中的物件當作根，並標記它們所指向的年輕物件。回收之後，被記錄的物件會再次設定 `UNLOGGED`。

### 卡片標記

8 KiB 以上的老物件以每 512 位元組一張**卡片**的方式記錄，而不是記錄整個物件。這樣的物件設定的是 `CARDED` 位元（[8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)）而不是 `UNLOGGED`。

- 這樣的物件有一張卡片表（每 512 B 1 位元組），配置在物件之後。指向該卡片表的指標位於緊接在標頭之前的字組中。
- 行內快速路徑一次檢查 `p` 標頭中的 `UNLOGGED | CARDED`（一次載入、一個分支）。
- 順序：若是 `UNLOGGED`，先呼叫慢速路徑；然後若是 `CARDED`，標記卡片。在並行標記期間（[7.5](705-concurrent-collection.md)），老的 LOS 物件可能同時有兩個位元。
- 若是 `CARDED`，屏障會以行內方式、不經呼叫地，把被寫入之槽位所跨越的卡片標記為髒，即從 `(slot - p) >> 9` 到 `(slot + len - 1 - p) >> 9`。因此屏障需要槽位位址以及被存入之值的大小（`len`）。
- 若是 `UNLOGGED`，它如上所述呼叫 `nabi_gc_wb_slow(p, slot, len)`（不變）。
- minor 回收只掃描 `CARDED` 物件的髒卡片，並清理那些卡片。

不需要屏障的存入：

| 存入                                | 理由                                                |
|-------------------------------------|-----------------------------------------------------|
| 初始化剛配置的物件                  | 配置與存入之間沒有 GC，因此仍是年輕的               |
| 寫入 `static`                       | static 是根                                         |
| 原始的 `Ptr<T>` 記憶體              | 不受 GC 追蹤                                        |
| atomics（非參照值）                 | 不是 GC 參照                                        |

### 記憶集

每個執行緒有一個 512 字組的記錄區塊（在 64 位元目標上為 4 KiB，在 32 位元上為 2 KiB）（TLS 槽位 `nabi_tls_wb_log_cursor` / `nabi_tls_wb_log_limit`）。已滿的區塊，以及在回收時或執行緒結束時的所有區塊，會被推入一個無鎖堆疊。回收器取出該堆疊的內容，並把記錄的物件當作根使用。

### 行內成本

快速路徑是對 `p` 標頭的一次載入與一個分支（`UNLOGGED | CARDED`）；通常兩個位元都是清除的（年輕物件或已記錄），因此會直接通過。

### 編譯器的發出

編譯器以行內方式發出快速路徑；只有慢速路徑會呼叫執行環境函式（[8.5](../008-abi/805-extern-symbols.md)）。

```
extern "C" {
    fn nabi_gc_wb_slow(obj: Ptr<u8>, slot: ConstPtr<u8>, len: usize);   // clear UNLOGGED and add to the remembered set (slow path, not a safepoint)
}
```

## 7.4.3. 安全點輪詢

回收在所有 mutator 執行緒都停在安全點時進行。編譯器在每個函式與迴圈中發出安全點輪詢（8.4.2 的例外除外）（[8.4](../008-abi/804-safepoint.md)）。

```
poll:
    if tls.yield_flag {
        call nabi_runtime_yield()       // signal arrival at a GC-safe point
    }
```

`nabi_tls_yield_flag` 通常是 0。當回收器設定它時，每個執行緒會在下一次輪詢時停下。

## 7.4.4. 走訪影子堆疊

GC 開始標記時，會做下列事情：

```
for each thread T:
    frame = T.nabi_tls_shadow_stack_top
    while frame != null:
        for i in 0..frame.count:
            ref = *frame.slots[i]          // slot is the address of a word
            if ref != null:
                mark(ref)
        frame = frame.prev
```

它走訪每個執行緒的影子堆疊，並把每個存活的 GC 物件變數加入標記佇列。

之後，標記演算法（[GC 演算法](703-gc-algorithm.md)）從那些物件開始並追蹤其子項。

## 7.4.5. 主要的 TLS 槽位

每個執行緒都有下列 TLS 槽位。這些槽位是由執行環境匯出、每個執行緒一份的區塊 `nabi_tls` 中固定偏移的欄位（[8.3.1](../008-abi/803-tls-slots.md#831-tls-slot-structure)）。

| 槽位                        | 意義                                                                                      |
|-----------------------------|-------------------------------------------------------------------------------------------|
| `nabi_tls_yield_flag`        | 是否要在安全點 `yield`                                                                    |
| `nabi_tls_tlab_cursor`       | [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 中下一個配置位置 |
| `nabi_tls_tlab_limit`        | TLAB 的末端                                                                               |
| `nabi_tls_shadow_stack_top`  | 影子堆疊鏈的開頭（最內層的框架）                                                          |
| `nabi_tls_wb_log_cursor`     | 在寫入屏障記錄中的位置                                                                    |
| `nabi_tls_mutator_state`     | 目前的 mutator 狀態（執行中 / 與 GC 協作中）                                              |

TLS 槽位的詳細定義見 [TLS 槽位](../008-abi/803-tls-slots.md)。
