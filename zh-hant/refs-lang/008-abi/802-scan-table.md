# 8.2. ScanTable 序列化格式

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 的標記階段必須追蹤每個物件的子指標，因此需要說明子指標位於哪些偏移的資料。這份資料稱為 ScanTable；本文件定義其序列化格式。

## 8.2.1. 目的

GC 的標記迴圈運作如下。

```
GC mark loop:
  for each live object O:
    type_id = O.type_id
    scan_table = lookup(type_id)
    for each (offset, kind) in scan_table:
      child = *(O + offset)
      mark(child)
```

這裡的 `lookup(type_id)` 使用編譯器在行程啟動時，透過[物件標頭 ABI](801-object-header-abi.md) 的 `nabi_gc_register_type` 所註冊的內容。

堆疊（影子堆疊）與 `static` 根以相同的、以位元組偏移為基礎的走訪方式掃描。單一的根槽位被當作一個 ScanTable 項目處理，而 `any` 根也涵蓋其酬載字組。

## 8.2.2. 註冊呼叫

註冊透過呼叫下列外部函式進行。

```
extern "C" {
    fn nabi_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

- `type_id`：要註冊之型別的 ID。
- `scan_table`：指向序列化後之 ScanTable 開頭的指標。

編譯器把 ScanTable 記憶體發出到 .rodata 中，因此它在行程的生命週期內都有效。執行環境只讀取這個區域。

## 8.2.3. 序列化格式

ScanTable 是一個 8 位元組的標頭，其後接一個項目陣列。

```
ScanTable Header (8 bytes, 4-byte aligned):
  offset  0:  total_size       u32         // total object size (16 + payload)
  offset  4:  version          u16  = 1
  offset  6:  num_entries      u16         // number of child pointer entries

Entry array (8 bytes each):
  offset  8 + 8*i + 0:  field_offset  u32  // offset within the object
  offset  8 + 8*i + 4:  field_kind    u32  // kind of child pointer
```

總大小是 8 + 8 × num_entries，每個欄位都自然對齊。

### 標頭

標頭欄位如下。

| 欄位          | 型別 | 意義                                             |
|---------------|------|--------------------------------------------------|
| `version`     | u16  | 格式版本（位元 0 ~ 14，目前 = 1）。位元 15 = 陣列表 |
| `total_size`  | u32  | 物件總大小（16 位元組標頭 + 對齊後的酬載，至少 24，[8.1](801-object-header-abi.md)） |
| `num_entries` | u16  | 子指標項目的數量                                 |

若 `version` 的位元 15 為 1，該表就是**陣列表**。此時 `total_size` 是元素的步距，每個項目的 `field_offset` 相對於元素的開頭，而這些項目會反覆套用到整個酬載中的每個元素。編譯器以這種形式為 GC 緩衝區撰寫元素表。

### 項目

每個項目指定物件中的一個子指標槽位。

| 欄位           | 型別 | 意義                             |
|----------------|------|----------------------------------|
| `field_offset` | u32  | 槽位的偏移（從物件開頭起算）     |
| `field_kind`   | u32  | 槽位的種類                       |

### `field_kind` 的值

`field_kind` 可以取的值及其意義如下。

| 值  | 意義                                                             |
|-----|------------------------------------------------------------------|
| 0   | `STRONG_REF`：一般的 GC 物件指標（由標記追蹤）                   |
| 1   | `WEAK_REF`：弱參照（標記不追蹤，在回收時清除）                   |
| 2   | `INTERIOR_REF`：指向物件內部（在轉發時調整）                     |
| 3   | `ANY_SLOT`：`tag + payload` 形式的動態槽位。檢查標籤之後標記     |
| 4   | `SUM_PAYLOAD`：和型別的標籤與各變體的參照（8.2.6）。位元 8..31 存放描述子的位置 |
| 5   | `AMBIGUOUS`：可能是也可能不是參照的字組。被指向的物件會保持存活，但不會被移動（型別 id `2` 的緩衝區、不帶標籤的 `union`） |
| 6 ~ | 保留 / 額外的種類                                                |

各種類實際上如何標記由 GC 實作決定；本規格只定義存在哪些種類及其編碼。

## 8.2.4. 編碼範例：`struct Point { x: f64; y: f64; }`

這個結構沒有子指標（所有欄位都是基本的浮點數型別）。

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 header + 8 + 8
  num_entries  = 0
}
```

因此 ScanTable 是 8 位元組，只有標頭。

## 8.2.5. 編碼範例：`struct Pair { a: List<i32>; b: List<f64>; }`

這個結構有兩個子指標。

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 header + 8 (List ptr) + 8
  num_entries  = 2

  entry[0]: field_offset = 16, field_kind = STRONG_REF (0)
  entry[1]: field_offset = 24, field_kind = STRONG_REF (0)
}
```

總大小是 8 + 16 = 24 位元組。

## 8.2.6. 編碼範例：`enum` 變體

```
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

容納這個型別的物件由 16 位元組的標頭、標籤（4 位元組）、填充（4 位元組）以及最大變體的酬載（16 位元組，一個 `str` 指標與長度）組成，總共 40 位元組。

每個變體中參照位於不同的位置，因此使用單一的 `SUM_PAYLOAD` 項目及其描述子。

- 項目的 `field_offset` 是標籤（u32）的偏移。
- `field_kind` 的低位元組是 `4`；位元 8..31 是描述子的位置（以從表開頭起算的 u32 字組為單位）。
- 描述子存放 `n`，其後為每個有參照之變體的 `{tag, num_entries, entries_index}`。
- 變體的項目是 `{offset from the tag, field_kind}`。巢狀的和型別再以 `SUM_PAYLOAD` 撰寫。
- 標籤不在描述子中的變體沒有參照。

```
ScanTable {
  version = 1, total_size = 40, num_entries = 1
  entry[0]: field_offset = 16, field_kind = 4 | (desc << 8)    // tag at offset 16
  desc:     n = 1
            { tag = 1 (Err), num_entries = 1, entries_index = e }
  e:        { offset = 8, field_kind = INTERIOR_REF }           // str ptr of Err (into a payload)
}
```

列舉 / 和型別酬載中的參照是精確的，因此透過酬載到達的物件也可能被疏散移動。`AMBIGUOUS` 只用於型別 id `2` 的緩衝區，以及沒有標籤而無法解讀的使用者 `union` 的儲存空間。

## 8.2.7. ANY 槽位

`any` 槽位（[`any` 槽位](../002-types/202-any-slot.md)）可以以 `tag + payload` 的形式容納任何型別的值。

```
ANY slot layout.
  offset  0:            tag       u32
  offset  ptr-aligned:  payload   usize (numbers/bool/char up to pointer width stored directly, everything else as a box ptr)
```

對應的 ScanTable 項目如下。

```
field_offset = (offset of the slot)
field_kind = ANY_SLOT (3)
```

酬載是否為 box 參照由標籤攜帶。

- 若標籤的**位元 31** 為 1，酬載是 box 參照；若為 0，它就是值本身。
- 對標籤的型別比較使用遮掉位元 31 之後的值。

因此 GC 可以精確地掃描酬載，並改寫被移動之 box 的位址，不需要任何額外的表。在分開編譯的套件之間也不需要合併表。

## 8.2.8. WEAK_REF

弱參照不會讓物件保持存活。

- 標記階段不追蹤弱 `ref`。
- 清除之後，若弱 `ref` 所指向的物件已被回收，弱 `ref` 槽位會被清為 0。

這個機制對快取、觀察者模式等很有用；其函式庫介面是 `Weak<T>`（[stdlib 1.6.9](../../refs-standard-libraries/100-bare/106-gc-pin.md#169-weakt---weak-references)）。

## 8.2.9. INTERIOR_REF

INTERIOR_REF 指向物件內部的某個位置（而不是其開頭）。

- 它來自切片、部分視圖等。
- 在轉發時，它依舊物件與新物件之間的相對偏移調整。

確切的處理方式由 GC 實作決定。

## 8.2.10. 序列化的位元組順序

表是以目標位元組順序表示的 `u32` 字組陣列，執行環境把它們當作原生字組讀取。內建的[平台目標](../000-meta/006-platform-targets.md)都是小端序，但 `s390x-unknown-linux-gnu`（進行中）除外，在那裡這些字組是大端序。

## 8.2.11. 變更的相容性

`version` 欄位識別格式。格式的變更遵循[概觀](README.md#stability)中的穩定性方針。預先建置的套件只與建置它的 `nabic` 一起使用，因此程式所註冊的表與讀取它們的執行環境一律來自同一個工具鏈。

## 8.2.12. 執行環境保留的型別 id

下列三個型別 id 由執行環境直接定義；編譯器不為它們註冊 ScanTable（[8.1.2](801-object-header-abi.md#812-type_id-offset-0-u32)）。

| type id | 意義              | 掃描                                                    |
|---------|-------------------|---------------------------------------------------------|
| `1`     | 原始資料          | 酬載不容納參照，不會被掃描：GC 字串、位元組緩衝區，以及每個 `T` 不容納 GC 參照（數字、`bool`、`char`、`Pod` 結構）的 `GcBuf<T>` |
| `2`     | 以字組追蹤的緩衝區 | 酬載的每個字組都被當作模糊字組（`AMBIGUOUS`）掃描       |
| `3`     | weak 儲存格（`Weak<T>`） | 一個 `WEAK_REF` 字組（8.2.8）；由 `nabi_gc_weak_new` 建立 |

id `1` 與 `2` 以 `nabi_gc_alloc_slow` 配置，型別 id 為 `1` 或 `2`（[8.5.1](805-extern-symbols.md#851-gc-area)）。標準函式庫的 [`GcBuf<T>`](../../refs-standard-libraries/200-core/209-collections.md#2912-gcbuft---raw-gc-buffer) 建立在這之上：編譯器以 id `1` 配置不含參照之 `T` 的緩衝區，因此無論數值資料多大，回收器都不必花時間在其上（存活的 256 MiB `List<f64>` 不會讓完整回收增加任何可量測的時間；同樣的位元組若是型別 `2` 的緩衝區，每次回收約需 140 ms），而且其中的任何數字都不會讓垃圾保持存活。`T` 容納參照的緩衝區會得到 `T` 的陣列 `ScanTable`（`version` 的位元 15，8.2.3），它只掃描參照槽位。型別 id `2` 只用於執行環境的字組緩衝區。

## 8.2.13. 註冊時機

註冊在行程啟動時整批進行。

```
process_start:
  for each registered type:
    nabi_gc_register_type(type_id, &scan_table)
  ↓
  user main()
```

編譯器發出一個收集所有註冊呼叫的函式 `nabi.register_types`。在 `-O0` 以上，當套件以連續的 id 註冊四個以上的表時，它改以單一呼叫註冊：`nabi_gc_register_types(base, tables, n)` 把指標陣列 `tables` 的 `n` 個表註冊在 id `base`、`base + 1`、.. 之下（[8.5.1](805-extern-symbols.md#nabi_gc_register_type--nabi_gc_register_types)）。編譯器產生的 C ABI `main` 在 `nabi_runtime_init` 之後立即呼叫這個函式，然後進入使用者的進入點。

```
main(argc, argv):                // compiler-generated, C ABI
  nabi_runtime_init()
  nabi.register_types()           // collection of nabi_gc_register_type calls
  → user entry point (4.16)
```

GC 物件配置的慢速路徑是 `nabi_gc_alloc_slow(size, align, type_id)`（[7.2.3](../007-runtime/702-allocation.md#723-slow-path-nabi_gc_alloc_slow)），而編譯器指派的型別 id 從 `0x10000` 開始（[8.1.2](801-object-header-abi.md#812-type_id-offset-0-u32)）。

表的位元組在程式中只出現一次：每個套件的 `nabi.register_types` 參照一個以表的內容命名的隱藏定義，連結時只保留一份，即使多個套件以各自的 id 註冊同一種配置也一樣。
