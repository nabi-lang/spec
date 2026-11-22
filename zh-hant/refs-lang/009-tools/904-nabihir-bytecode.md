# 9.4. NABIHIR 格式

本節定義預先建置之函式庫套件的介面檔 `<name>.nabihir`。[NABIHIR](../000-meta/007-glossary.md#nabihirnabi-hir-位元組碼) 檔案位於該套件的 `lib<name>.a`（[套件封存檔](903-package-archive.md)）旁邊，不是封存檔的成員。

## 9.4.1. 動機

[AOT 建置管線](902-aot-pipeline.md)中的單態化依呼叫處的具體型別進行[特化](../000-meta/007-glossary.md#特化specialize)。因此當另一個套件呼叫某個套件的[泛型](../000-meta/007-glossary.md#泛型generic) `fn` 時，特化必須在該呼叫處進行。只有[具體 fn](../000-meta/007-glossary.md#具體函式--泛型函式concrete-fn--generic-fn) 的目的碼是不夠的，所以另外提供一個介面供下游讀取。

單態化的輸入是 AST。因此 NABIHIR 的主體是**去除過的套件原始碼**，由下游的剖析器讀取。

## 9.4.2. 不使用外部函式庫

依零相依原則，也不使用任何序列化函式庫；NABIHIR 是從零開始設計的。

## 9.4.3. 整體配置

```
NABIHIR file (little-endian):
+-----------------------------------+
|  Header                           |  (16 bytes fixed)
|    magic       "NABH" (4 bytes)   |
|    version     u32  (= 1)         |
|    flags       u32                |
|    body_offset u32                |
+-----------------------------------+
|  Producer string        str       |  "nabic <version> (<compiler id>)"
+-----------------------------------+
|  Meta                             |
|    name                 str       |  mount name (--extern <name>=)
|    hash                 str       |  package hash (0.3.3)
|    source dir           str       |
|    gc type offset       varint    |
|    gc type count        varint    |
|    features  varint n, n × str    |
+-----------------------------------+
|  Body (at body_offset)            |
|    file count           varint    |
|    file × n: path str, text str   |
+-----------------------------------+
|  Summaries                        |
|    count                varint    |
|    n × (function str, summary str) |
+-----------------------------------+
|  Call summaries                   |
|    count                varint    |
|    n × (function str,             |
|         flags varint,             |
|         keeps_none varint,        |
|         returns varint)           |
+-----------------------------------+
```

`str` 是前面帶有 PrefixVarInt 長度（9.4.5）的 UTF-8 位元組。沒有字串 / 型別 / 項目區段。

## 9.4.4. 標頭與產生者

| 欄位          | 大小 | 意義                                                         |
|---------------|------|--------------------------------------------------------------|
| `magic`       | 4    | `"NABH"`                                                     |
| `version`     | u32  | 格式版本（`1`）。配置或主體內容改變時遞增                    |
| `flags`       | u32  | 保留（0）                                                    |
| `body_offset` | u32  | 主體的起始偏移                                               |

產生者字串是 `nabic <version> (<compiler id>)`。編譯器 id 是該 `nabic` 執行檔的戳記，每當它被寫入或取代時就會改變：其大小、修改時間、inode 與變更時間（`<size>-<mtime>-<inode>-<ctime>.<ctime_nsec>`，十六進位；在 Windows 上以檔案 id 與建立時間取代 inode 與變更時間）。

## 9.4.5. VarInt 編碼

PrefixVarInt 是 LEB128 的一種變體。第一個位元組開頭的 1 位元數量就是後續位元組的數量，而值以大端序占據其餘的位元。

```
1-byte int  : 0xxxxxxx                    (0 ~ 127)
2-byte int  : 10xxxxxx xxxxxxxx           (0 ~ 16383)
3-byte int  : 110xxxxx xxxxxxxx xxxxxxxx  (0 ~ 2097151)
...
```

## 9.4.6. meta

| 欄位             | 意義                                                                              |
|------------------|-----------------------------------------------------------------------------------|
| name             | 套件的掛載名稱。它出現在符號中，因此必須等於 `--extern <name>=`                   |
| hash             | 套件雜湊（[0.3.3](../000-meta/003-source-units.md#033-套件識別碼)）。用於符號修飾的值。對標準函式庫套件 `nabi` 而言，它是一個 `pkg=hash;...` 清單，每個層級一個項目 |
| source dir       | 建置時容納套件原始碼的目錄。主體中檔案路徑的基準；下游讀取主體，不讀取磁碟       |
| gc type offset / count | 這個套件的 GC 型別 id 範圍 `[offset, offset + count)`（9.4.8）               |
| features         | 建置套件時所使用的 `feature` cfg 值。下游以建置的其他 cfg 與這些 feature 剖析主體 |

## 9.4.7. 主體

主體以（路徑，文字）組合容納套件的**每個原始碼檔案**。每個檔案依下列方式去除。

- 具體函式的主體被刪減成 `;`。換行會保留，因此診斷的行號與原始檔案相符。這些函式作為封存檔的符號連結。
- 下游必須單態化或內嵌的主體會保留：泛型函式、`async` / `yield` 函式、`#[inline]` 函式、`extern "C"` 定義、trait 的預設方法，以及泛型 `impl` 的方法。
- 簽章、型別、trait、`impl` 與 `use` 等宣告保持原樣。
- 保留之文字中的 `include_str!` / `include_bytes!` 被替換成它們所讀取的字面值（保留換行），因此下游不讀取任何被引入的檔案，移動過的函式庫也仍能運作。

原始碼檔案之後是套件之 `ffi::wit` 項目所讀取的 `.wit` 檔案（[stdlib 1.8](../../refs-standard-libraries/100-bare/108-wit.md)），連同其 `deps/` 完整收錄。它們的路徑相對於 source dir，並可以經由 `..` 離開它。下游從主體讀取 WIT，不讀取磁碟。

主體之後是套件函式的交出摘要（[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync執行緒邊界標記)）。摘要由以 tab 分隔欄位的行組成：

- `g <root> <mutex|rwlock|channel> <line> <function> <file>`：交出 `<root>`；
- `s <root> <root>`：同一個物件；
- `i <root> <root>`：第一個消失時會一併帶走第二個；
- `t <guard parameter> <root>`：綁定到該 guard。
- `x <tag|*> [<root>]`：在回傳值帶有該標籤的出口上（`*`：未知；`bool` 的 0 / 1、和型別或 enum 的變體、承載結構的 1 表示錯誤、0 表示值、2 + 值的標籤），`<root>` 屬於被交出者；沒有 `x` 行時，每個出口都交出所有 `g` 根。

根是一個參數索引或 `r`（回傳值），然後是以 `.` 分隔的欄位路徑（元組索引會設定位元 31）。名為 `#fn`（`#fne`：閉包，環境在前）、其後接以 U+001F 分隔之參數型別鍵的項目，合併以那些參數型別作為值使用之函式的摘要。

最後是呼叫摘要：呼叫一個其程式碼位於封存檔中的函式會做什麼，好讓最佳化的下游建置不必把它當作可能做任何事。項目依函式名稱（其項目路徑，與下游宣告它的方式相同）排序；沒有任何已知資訊的函式沒有項目，泛型實例（下游會自己產生）與 `any` 分派器（程式的分派器會取代它）也沒有。每項事實都針對封存檔的程式碼成立：

| 欄位         | 意義 |
|--------------|---------|
| `flags`      | 位元 0 `readonly`：不寫入其呼叫者能讀取的任何記憶體（它仍可以配置記憶體、到達安全點與 panic）；位元 1 `quiet`：永不到達 GC 安全點（沒有配置、輪詢或阻塞等待），因此呼叫不會移動物件；位元 2 `cold`：`#[cold]`，或永不回傳 |
| `keeps_none` | 位元 `i`：參數 `i` 的值不會被儲存、擷取或傳到可能被保留之處，而且只有在 `returns` 這麼說時，才會在呼叫之後存活於結果中 |
| `returns`    | 位元 `i`：結果可能是參數 `i` 的值或由它衍生 |

下游只在 `-O0` 以上讀取它們。`readonly` 呼叫不會破壞任何載入，因此在它之前載入的值在它之後會被重複使用，載入也可以跨越它移動；`quiet` 呼叫對其呼叫者而言不是安全點；`cold` 呼叫把其區塊標示為很少執行；傳入 `keeps_none` 參數的堆疊位址不會逃逸。panic 不會被移動：呼叫留在原處。

## 9.4.8. 分開編譯的規則

預先建置的函式庫無法一次看到整個程式，因此各套件依下列規則協調。

- **`any` 標籤。** `any` 槽位的標籤是型別名稱的 31 位元雜湊，因此分開建置的套件對同一個型別使用相同的標籤。
- **`any` 種子。** 為了讓程式知道函式庫放進 `any` 的型別，函式庫會在主體結尾附加一個使用那些型別的小型 Nabi 函式。程式的封閉世界 `any` 分派器也包括這些型別。
- **分派器符號。** `any` 分派器是符號 `nabi.any.<hash>`，在函式庫中定義為 weak，在程式中定義為 strong。在連結時，程式的定義勝出。
- **和型別成員順序。** 和型別的各項依結構鍵排序，而不是依內部型別編號排序（[2.6.4](../002-types/206-sum-types.md#標準順序)），因此各套件的標籤一致。
- **GC 型別 id。** 每個套件都有自己的 GC 型別 id 範圍（9.4.6）。同一個程式中兩個套件的範圍重疊是錯誤。
- **方法符號。** 相依套件的方法以穩定的路徑 `<module>::<type>.<method>[__by_<trait>]` 修飾（[9.5](905-symbol-mangling.md)）。
- **呼叫摘要。** 下游建置的程式碼取決於它所呼叫之函式的呼叫摘要（9.4.7）。增量建置的 codegen 單元以其位於其他單元與套件中之被呼叫者的摘要作為鍵，因此摘要改變之重新建置的相依套件，只會重新編譯呼叫了已改變之函式的單元。

## 9.4.9. 讀取與拒絕

`--extern name=<name>.nabihir` 或 `--extern name=lib<name>.a` 都會找到另一個檔案（[9.2.3](902-aot-pipeline.md#923-相依套件解析)）。下游把主體當作原始碼覆蓋層剖析以取得宣告，把被去除的函式宣告為預先建置的符號，就地單態化泛型，然後連結封存檔。

若下列任何一項不同，NABIHIR 會以「重新建置」錯誤被拒絕。

- `version`
- 產生者（由不同的 `nabic` 產生的 NABIHIR）
- name（`--extern` 名稱與 NABIHIR 的掛載名稱）

## 9.4.10. 檢視 NABIHIR

`nabi inspect --dump <file>` 印出 NABIHIR 的版本、產生者、meta（包括 `features:`）、介面檔案的路徑與文字、摘要，以及呼叫摘要（`--- calls`：每個函式一行，包括其名稱與已設定的事實）（[9.9.2](909-debug-tools.md#992-檢視-nabihir)）。`nabi inspect`（[9.9.14](909-debug-tools.md#9914-檢視套件)）把摘要與封存檔一起顯示。

## 9.4.11. 穩定性

只會讀取由同一個 `nabic` 產生的 NABIHIR（9.4.9）。格式改變時，`version` 會遞增，而不同的版本需要重新建置。適用[概觀](../008-abi/README.md)的穩定性方針。

標準函式庫不以 NABIHIR 散布（[9.3.6](903-package-archive.md#936-標準函式庫)）。
