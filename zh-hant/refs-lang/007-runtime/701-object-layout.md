# 7.1. 物件配置與記憶體空間

每個由 [GC](../000-meta/007-glossary.md#gc垃圾回收器) 管理的物件都以固定的標頭開始。本文件定義標頭格式，以及物件可能所在的三個記憶體空間。

> [!NOTE]
> 本節僅供參考，但在 [8.1](../008-abi/801-object-header-abi.md) 中固定為 ABI 的 16 位元組標頭除外。記憶體空間描述的是參考執行環境，可能在任何版本中改變。

## 7.1.1. 16 位元組標頭

每個 GC 物件的前 16 個位元組是標頭。

```
offset  0 :  type_id    u32       // type identifier
offset  4 :  gc_flags   u32       // GC state bits
offset  8 :  gc_word    u64       // total object size (new address during evacuation)
offset 16 :  payload ...           // the object's actual fields
```

標頭與酬載都以 8 位元組對齊。配置在 32 位元與 64 位元上相同，因此在 32 位元目標上標頭也是 16 位元組。

### `type_id`（偏移 0，4 位元組）

型別的唯一識別碼，由編譯器為每個型別指派。

它有幾種用途。在動態分派中，它作為 [`any` 槽位](../002-types/202-any-slot.md)的標籤；在 GC 的欄位掃描中，它透過 `type_id → field layout` 對應找出欄位；在診斷中，它用來顯示對使用者友善的型別名稱。

`type_id` 在行程的生命週期內是穩定的；同一個型別一律有同一個 id。

### `gc_flags`（偏移 4，4 位元組）

表示 GC 狀態的位元。

| 位元        | 意義                                                                                    |
|-------------|-----------------------------------------------------------------------------------------|
| `mark`      | 在目前的 GC 週期中是否已到達                                                            |
| `sticky`    | [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit)（世代追蹤）              |
| `pinned`    | 標示為不可移動，例如供 [FFI](../000-meta/007-glossary.md#ffi外部函式介面) 使用 |
| `forwarded` | 已經被疏散到新位置（gc_word 是轉發指標）                                                |
| `large`     | [LOS](../000-meta/007-glossary.md#los大型物件空間)（Large Object Space）物件     |
| ...         | 其他位元由 GC 實作決定                                                                  |

確切的位元配置由 GC 實作決定；表格只列出語意上必需的位元種類。

### `gc_word`（偏移 8，8 位元組）

物件的總大小（包括標頭）。對於在疏散期間被移動的物件，`forwarded` 位元會被設定，而這個槽位指向新的位置（[8.1.4](../008-abi/801-object-header-abi.md#814-gc_word偏移-8u64-或-ptr)）。

### `payload`（偏移 16 以上）

存放物件欄位的區域。欄位順序與對齊由編譯器決定；這對 [`enum`](../002-types/204-enum.md) 與 [`union`](../002-types/205-union.md) 也一樣。[`struct`](../002-types/203-struct.md) 值本身不是堆積物件；存放在堆積物件（串列緩衝區、閉包環境、協程框架）中時，它以行內方式配置在這裡（[8.8.3](../008-abi/808-lowering-types.md#883-struct-的表示方式)）。

若附加了 `#[repr(C)]` 之類的屬性，就適用對應的表示規則（[內建屬性](../006-meta-system/602-built-in-attributes.md)）。

## 7.1.2. 記憶體空間

GC 把每個物件放在下列三個空間之一。字面值與 `const` / `static` 資料不是堆積物件：它們位於程式的資料區段中，在這些空間之外，GC 既不標記也不移動它們（容納 GC 參照的 `static` 是根，[7.4.1](704-barriers-and-roots.md)）。

### 1. Immix

大多數一般物件所在的空間，使用 Immix mark-region 演算法。

它被劃分為 32 KiB 的[區塊](../000-meta/007-glossary.md#區塊block)與 128 B 的行。預設是標記-清除，完整回收可能會把物件移出碎片化的區塊（機會式疏散）。世代以 sticky mark-bit 區分（[7.3](703-gc-algorithm.md)）。

### 2. 大型物件空間（LOS）

8 KiB 以上，或需要大於 8 位元組之對齊的物件，以及機密緩衝區（`nabi_gc_secret_new`，[8.5.1](../008-abi/805-extern-symbols.md#nabi_gc_secret_new--nabi_gc_secret_wipe)），在 LOS 中個別配置，並設定 `gc_flags` 的 `LARGE` 位元。LOS 以標記-清除回收，不會移動物件。已成為老物件的 LOS 物件有卡片表（[7.4.2](704-barriers-and-roots.md#742-寫入屏障物件記錄)）。機密緩衝區有 `WIPE` 位元（[8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags偏移-4u32)）：釋放它的清除階段會先以零覆寫其酬載。

### 3. 釘選的物件（FFI 釘選）

傳給 [`extern` 區塊](../005-modules/509-extern-block.md)之 FFI 函式的 GC 指標，在呼叫結束之前不會被移動，因為該引數位於根中（[7.4.1](704-barriers-and-roots.md#被根指向的物件不會被移動)）。若外部程式碼需要在呼叫之後保留該位址，請以 `nabi_gc_pin` 釘選它。釘選的物件設定了 `gc_flags` 的 `PINNED` 位元，並保持在原位。`nabi_gc_unpin` 解除釘選。

釘選的物件仍會以標記-清除回收，但絕不會被疏散。

## 7.1.3. 空間之間的移動

```
TLAB (allocation) → Immix
Immix → Immix         (evacuation in full collection)
Immix / LOS ↔ pinned  (pin / unpin, in place)
```

物件不會在空間之間移動。只有疏散會改變位址。

## 7.1.4. 標頭寬度的動機

16 位元組的標頭是時間 / 空間取捨的結果。

對於小物件，例如 4 位元組的 i32，16 位元組的標頭會損害記憶體效率。但 16 位元組能同時容納 type_id、旗標與轉發資訊，簡化了演算法，而且配置在 32 位元與 64 位元上保持相同，也簡化了跨平台的程式碼。

在大多數物件都是 4 位元組以上的假設下，這個取捨是合理的選擇。

## 7.1.5. 註冊 type_id：`nabi_gc_register_type`

每個型別的欄位配置都必須向 GC 註冊。

```
nabi_gc_register_type(type_id, scan_table)
```

`type_id` 是型別的識別碼，`scan_table` 存放指向 GC 物件之指標欄位的偏移，以及額外的中繼資料。

標記物件時，GC 使用其 `type_id` 找到 `scan_table`，並透過它追蹤子指標。

編譯器在行程啟動時透過 `extern "C"` 呼叫進行這項註冊；註冊的 ABI 定義於 [Extern "C" 符號](../008-abi/805-extern-symbols.md)。

## 7.1.6. 計算物件的總大小

```
size = 16  (header) + payload_size  (aligned size of the fields)
```

配置器把這個總大小存放在每個物件之標頭的 `gc_word` 中，Immix 與 LOS 皆然（[8.1.4](../008-abi/801-object-header-abi.md#814-gc_word偏移-8u64-或-ptr)），因此回收器從那裡讀取它。
