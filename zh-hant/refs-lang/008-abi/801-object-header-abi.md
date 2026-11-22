# 8.1. 物件標頭 ABI

定義[物件配置與記憶體空間](../007-runtime/701-object-layout.md)中 16 位元組標頭的位元配置。標頭大小、`gc_flags` 位元、LOS 門檻、型別 id 的起始值以及執行環境緩衝區型別 id 等 GC 常數，都來自編譯器與執行環境共用的單一定義（`nabir_abi::gc`）。

## 8.1.1. 確切的配置

標頭的形式如下。

```
+---------+---------+-----------------+----------------+
| offset  |  size   |  field          |  type          |
+---------+---------+-----------------+----------------+
|    0    |    4    |  type_id        |  u32           |
|    4    |    4    |  gc_flags       |  u32           |
|    8    |    8    |  gc_word        |  u64 / ptr     |
|   16    |    var  |  payload        |  per type_id   |
+---------+---------+-----------------+----------------+
```

> [!IMPORTANT]
> 標頭的總大小是 16 位元組，8 位元組對齊，在 32 位元與 64 位元目標上相同。

每個物件至少 24 位元組（標頭加上一個酬載字組，即 `nabir_abi::gc` 中的 `MIN_OBJECT_SIZE`），因此編譯後的程式碼可以載入它持有參照之任何物件的第一個酬載字組。

## 8.1.2. `type_id`（偏移 0，u32）

`type_id` 是型別的唯一識別碼。它是目標位元組順序的 32 位元無號值（除 `s390x` 之外的每個目標都是小端序，[8.2.10](802-scan-table.md#8210-endianness-of-the-serialization)）；值 `0` 是保留的，表示 null 型別或無效的物件。在行程啟動時，編譯器呼叫 `nabi_gc_register_type` 註冊每個 type_id。

### type_id 的指派

在一個建置中，同一個型別有同一個 id。id 在不同建置之間不穩定，因此不用於序列化。

### 保留範圍

保留的 id 範圍如下。

| 範圍           | 用途                        |
|----------------|-----------------------------|
| `0`            | 無效 / null                 |
| `1`            | 執行環境（GC 字串，不追蹤） |
| `2`            | 執行環境（以字組追蹤的緩衝區） |
| `3`            | 執行環境（weak 儲存格，`Weak<T>`） |
| `4` ~ `0xFFFF` | 保留給執行環境              |
| `0x10000` ~    | 每個編譯後的型別（標準函式庫與使用者套件） |

編譯器為它所發出的每個型別（包括標準函式庫的型別）指派 `0x10000` 以上的 id。執行環境忽略低於 `0x10000` 的型別註冊。

## 8.1.3. `gc_flags`（偏移 4，u32）

`gc_flags` 欄位存放 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 狀態位元的確切位置，配置如下。

```
bit  0  : MARK       (mark for the current GC cycle)
bit  1  : STICKY     (sticky mark-bit, old generation)
bit  2  : PINNED     (must not move, FFI)
bit  3  : FORWARDED  (after evacuation; gc_word is the new location)
bit  4  : LARGE      (LOS object; gc_word is the size)
bit  5  : reserved   (unused)
bit  6 ~ 31 : reserved / GC implementation-defined
```

下列位元取自保留給實作的範圍（[7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-object-remembering)）。

| 位元 | 名稱       | 意義                                                                                       |
|------|------------|--------------------------------------------------------------------------------------------|
| 6    | `UNLOGGED` | 在回收中存活的物件上與 `STICKY` 一起設定。在寫入屏障第一次把物件記錄到記憶集時清除 |
| 7    | `CARDED`   | 在 8 KiB 以上的老物件上取代 `UNLOGGED` 設定。屏障把 512 B 的卡片標記為髒。卡片表指標是緊接在標頭之前的字組 |
| 8    | `NOMOVE`   | 位址被當作數字使用的物件。無論生命週期為何，絕不會被移動。由 `nabi_task_key` 設定（[8.5](805-extern-symbols.md)） |
| 9    | `NOEVAC`   | 回收器內部使用，為期一次完整回收：模糊字組（根，或執行環境字組緩衝區的字組，型別 id `2`）參照該物件，因此該次回收的疏散讓它留在原位（[7.3.1](../007-runtime/703-gc-algorithm.md#opportunistic-evacuation)）。在疏散之前對候選區塊中的這類物件設定，在疏散走訪它們時清除 |
| 10   | `SATB`     | 在並行標記期間，屏障記錄這個物件的參照時設定。競爭的存入會等到它被清除（[7.5.3](../007-runtime/705-concurrent-collection.md#753-satb-barrier)） |
| 11   | `SCANNED`  | 這個物件已在並行標記中被掃描 |
| 12   | `WIPE`     | 機密緩衝區（`nabi_gc_secret_new`，[8.5.1](805-extern-symbols.md#nabi_gc_secret_new--nabi_gc_secret_wipe)）。釋放它的清除階段會先把其酬載清零。絕不會被清除 |
| 13 ~ 28 | `VISIT_STAMP` | 供 `par` 走訪使用（[stdlib 2.19.1](../../refs-standard-libraries/200-core/219-par.md#2191-core-api)）：到達該物件之最後一次走訪的 16 位元時期編號。回收器保留這些位元 |
| 29   | `VISIT_OTHERS` | `par` 走訪：被不只一個項目到達，每個都是透過切片 |
| 30   | `VISIT_SLICED` | `par` 走訪：到目前為止只透過切片被到達 |
| 31   | `VISIT_CUR` | `par` 走訪：被正在走訪的項目到達 |

### 原子性要求

在 STW 回收中，`MARK` / `STICKY` / `FORWARDED` 是非原子的。在並行標記期間由標記器與 mutator 一起存取的 `MARK` / `SATB` / `SCANNED`，以及 mutator 之間會競爭的 `UNLOGGED` 清除，是原子的（[7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-object-remembering)）。

## 8.1.4. `gc_word`（偏移 8，u64 或 ptr）

`gc_word` 是物件的總大小（包括 16 位元組的標頭，為 8 的倍數）。它適用於 Immix 物件與 LOS 物件，並由配置的快速路徑以行內方式寫入（[8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path)）。

只有在完整回收的疏散期間，被移動之物件的 `FORWARDED` 位元才會被設定，而 `gc_word` 指向新的位址。所有的參照都在同一次回收中修正（[7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)），之後舊的位置會被捨棄，因此 mutator 永遠不會觀察到轉發。

區塊內的物件絕不會有 `type_id` 0。回收器以 0 標記可用空間。

### 回收器私有的資訊

回收器可以把自己的簿記資訊保存在標頭之前（位址低於物件位址之處），或另外的側表中。編譯器與其他程式碼只看得到從物件位址開始的標頭與酬載；其外的一切都屬於回收器的實作。

## 8.1.5. `payload`（偏移 16 以上）

編譯器依為該 `type_id` 註冊的欄位配置決定酬載區域。格式如下。

- 對 [`struct`](../002-types/203-struct.md) / [`enum`](../002-types/204-enum.md) / [`union`](../002-types/205-union.md)，配置由編譯器決定。
- 有 `#[repr(C)]` 時，遵循 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)。
- 有 `#[repr(packed)]` 時，忽略對齊。

在 GC 的標記階段，子指標的偏移透過 `type_id → ScanTable` 對應找出（格式定義於 [ScanTable 序列化格式](802-scan-table.md)）。

## 8.1.6. 對齊要求

各位置的對齊要求如下。

| 位置                     | 對齊                                |
|--------------------------|-------------------------------------|
| 物件開頭（標頭）         | 8 位元組                            |
| payload 開頭（偏移 16）  | 8 位元組（由於標頭是 16 位元組，自動成立） |
| 每個欄位                 | 其型別的對齊                        |

若 `payload` 內的欄位需要大於 8 位元組的對齊（例如 `#[repr(align(16))]`），酬載開頭就對齊到該值：標頭位於某個對齊位址之前 16 位元組處，而其前方的空隙屬於回收器（[8.1.4](#814-gc_word-offset-8-u64-or-ptr)）。

## 8.1.7. 標頭 + 酬載的總大小

標頭加上酬載的總大小計算如下。

```
total_size = 16 + align_up(payload_size, max(8, payload_align))
```

其中 `align_up(x, a) = (x + a - 1) & ~(a - 1)`。

這個大小就是 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) bump 指標的增量（見[記憶體配置](../007-runtime/702-allocation.md)）。

## 8.1.8. 範例：i32 物件

容納一個 i32 之 GC 物件的配置如下。

```
offset  0:  type_id = ID_I32     (4 bytes)
offset  4:  gc_flags = 0          (4 bytes)
offset  8:  gc_word = 24          (total size, 8 bytes)
offset 16:  i32 value             (4 bytes)
offset 20:  padding               (4 bytes, for 8-byte alignment)
total: 24 bytes
```

單一個 i32 總共占 24 位元組：16 位元組的標頭、4 位元組的值與 4 位元組的填充。

這個額外負擔對小物件而言看起來很大，但大多數物件的酬載更大（複合結構），因此比例會縮小。

## 8.1.9. 範例：大型物件（List<i64> 的緩衝區）

一個酬載為 16 KB 的串列緩衝區範例：

```
offset    0:  type_id = 1              (4 bytes, raw data: no reference)
offset    4:  gc_flags = LARGE          (4 bytes)
offset    8:  gc_word = 16400            (total size, 8 bytes)
offset   16:  i64[2048]                 (16384 bytes)
total: 16 + 16384 = 16400 bytes
```

`LARGE` 位元已設定，因此這是 LOS 物件。`gc_word` 是總大小（8.1.4）。元素不容納任何參照，因此該緩衝區具有執行環境型別 id `1`，不會被掃描（[8.2.12](802-scan-table.md#8212-runtime-reserved-type-ids)）。

## 8.1.10. 標頭改變時的相容性

這個標頭配置是 ABI 最核心的部分。它只在可能破壞 ABI 的版本中改變（[概觀](README.md#stability)）：1.0 之前是 0.x.0 版本，從 1.0 起是主版本。0.x.y 版本（從 1.0 起為次版本或修訂版本）會保持它。

由於這種穩定性，由同一個工具鏈建置的使用者套件、標準函式庫與執行環境全都相容。
