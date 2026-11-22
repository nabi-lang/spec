# 9.11. nabi-mir 檔案格式

本節定義 `<unit>.nabimir`，即 `nabic` 把一個 codegen 單元最佳化後的 nabi-mir 交給另外安裝之程式碼產生器（[8.11.13](../008-abi/811-backend-contract.md#81113-separately-installed-code-generators)）所用的檔案。它容納程式碼產生器契約（[8.11](../008-abi/811-backend-contract.md)）留給 `nabic` 決定的一切：符號及其連結性、型別配置、GC 型別、安全點資訊以及目標。

> [!NOTE]
> 0.2 的草案。0.1 的 `nabic` 不會寫出或讀取這個格式；下文是一份設計，在某個版本實作它之前都會改變。`--emit=mir` 仍會把 nabi-mir 印成供人閱讀的文字（[9.9.1](909-debug-tools.md#991-intermediate-output---emit)）。

## 9.11.1. 二進位，而非文字

這個格式是二進位的。

- 它的讀取者是程式碼產生器，不是人：`--emit=mir` 的文字是給人讀的，省略了程式碼產生器所需的東西（配置、符號、GC 型別、偵錯資訊）。文字形式會要求每個程式碼產生器都有一套文法與剖析器，卻沒有任何讀取者因此受益。
- 實例化了大量標準函式庫之單元的最佳化後 nabi-mir 很大；以 LEB128 表示整數，並讓每個字串只出現一次，可以讓每次建置的每個單元的檔案都保持小巧、寫出快速。
- LEB128 是 LLVM 與其他程式碼產生器已經會讀取的（`decodeULEB128`），也是 `nabic` 已經為其增量快取寫出的格式。
- NABIHIR（[9.4](904-nabihir-bytecode.md)）是原始碼文字，因為它的讀取者是 `nabic` 的剖析器；這個檔案的讀取者是程式碼產生器，它沒有 Nabi 的剖析器。

## 9.11.2. 編碼

| 記號 | 編碼 |
|----------|----------|
| 標頭與區段表中的 `u8`、`u16`、`u32`、`u64` | 固定寬度，小端序 |
| `u` | 無號 LEB128（最多 10 個位元組，值最多 64 位元） |
| `s` | 有號整數，先經 zigzag 對應（`(v << 1) ^ (v >> 63)`），再以 `u` 表示 |
| `w` | 128 位元整數：其低 64 位元以 `u` 表示，然後其高 64 位元以 `u` 表示 |
| `b` | 一個位元組：`0` 或 `1` |
| `str` | 字串表（9.11.4）中的 `u` 索引；`0` 是空字串 |
| `opt<T>` | `b`，為 `1` 時接著 `T` |
| `list<T>` | `u` 數量，然後是那麼多個 `T` |
| `ty` | 型別表（9.11.6）中的 `u` 索引 |
| `fn` | 函式表（9.11.9）中的 `u` 索引 |
| `static` | static 表（9.11.8）中的 `u` 索引 |
| `local`、`block` | 一個 `u`：區域變數或區塊在其函式內的編號 |
| `pos` | 原始碼位置：`u` 檔案（9.11.4；`0`：無）、`u` 行、`u` 欄（兩者都從 1 起算；`0`：未知） |

位置是 panic 或偵錯資訊指名它的地方：檔案是 `path:line:col` 所顯示的路徑，在 `nabic` 已把展開後程式碼中的位置對應回其來源文字之後。檔案中的值只有在它們是目標記憶體的位元組時（常數位元組陣列，9.11.7）才採用目標的位元組順序；其他每個數字都依上述方式編碼。

## 9.11.3. 配置

```
header (16 bytes)
  magic          4 bytes   "NABM"
  major          u16
  minor          u16
  flags          u32       0
  sections       u32       number of section table entries
section table    sections x 24 bytes
  kind           u32
  reserved       u32       0
  offset         u64       from the start of the file
  size           u64
sections         at their offsets, in any order
```

每種區段最多出現一次。種類如下：

| 種類 | 區段 | 內容 |
|------|---------|---------|
| 1 | strings | 9.11.4 |
| 2 | files | 9.11.4 |
| 3 | unit | 9.11.5 |
| 4 | target | 9.11.5 |
| 5 | structs | 9.11.6 |
| 6 | types | 9.11.6 |
| 7 | GC types | 9.11.7 |
| 8 | statics | 9.11.8 |
| 9 | functions | 9.11.9 |
| 10 | debug | 9.11.11 |

讀取者會略過它不認識之種類的區段。區段 1 到 9 存在於每個檔案中；10 只在有 `-g` 時存在。

## 9.11.4. 字串與檔案

**strings**：`u` 數量，然後每個字串以 `u` 長度以及那麼多個 UTF-8 位元組表示。字串 0 是空字串。每個不同的字串只儲存一次。

**files**：`list<str>`，即位置所指名之原始碼檔案的路徑，與 panic 印出它們的方式相同。檔案索引 `i`（從 1 起算）是第 `i` 個項目。

## 9.11.5. 單元與目標

**unit**：

| 欄位 | 編碼 | 意義 |
|-------|----------|---------|
| producer | `str` | `nabic <version> (<compiler id>)`，與 NABIHIR 相同（[9.4.4](904-nabihir-bytecode.md#944-header-and-producer)） |
| name | `str` | 單元的名稱：其所屬模組（套件根為 `<root>`），或對函式庫實例的分桶而言為 `<module>#k`（[9.2.16](902-aot-pipeline.md#codegen-units)） |
| opt | `u8` | `0` `-O0`、`1` `-O1`、`2` `-O2`、`3` `-O3`、`4` `-Os`、`5` `-Oz` |
| flags | `u` | 位元 0：開啟溢位檢查；位元 1：`-g`；位元 2：依程式碼範圍描述變數（`-O0` 以上的 `-g`，wasm32 除外） |
| source directory | `str` | `files` 中的相對路徑所相對的目錄（偵錯資訊） |

**target**：以程式碼產生器所讀取之欄位表示的目標（[0.6.5](../000-meta/006-platform-targets.md#065-target-specifications)），然後是完整的規格：

| 欄位 | 編碼 |
|-------|----------|
| name | `str`（內建名稱或規格檔的主檔名） |
| arch、os、env、vendor、family | 各為 `str`，與 `cfg` 的命名相同 |
| endian | `u8`：`0` 小端序、`1` 大端序 |
| pointer width、`wide_align`、max atomic width | 各為 `u`（位元、位元組、位元） |
| C ABI | `u8`：`c_abi`（`Aarch64` 0、`X86_64SysV` 1、`X86_64Win` 2、`X86SysV` 3、`X86Win` 4、`ArmHf` 5、`Wasm32` 6、`Riscv64` 7、`Riscv32` 8、`Ppc64` 9、`S390x` 10、`Other` 11） |
| object format | `u8`：`0` Mach-O、`1` ELF、`2` COFF、`3` wasm |
| CPU、features | 各為 `str` |
| C symbol prefix | `str` |
| stack probes | `u8`：`0` 行內、`1` 呼叫、`2` 無 |
| threads | `b` |
| specification | `str`：`nabic --print target-spec` 所印出的 `TargetSpec { .. }` 文字 |

## 9.11.6. 型別

**structs**：結構紀錄的 `list`，依序從 0 開始編號；型別以這個編號指名結構。

| 欄位 | 編碼 | 意義 |
|-------|----------|---------|
| name | `str` | 供偵錯資訊使用（未命名時為空） |
| key | `str` | 結構與建置無關的名稱 |
| packed | `b` | `#[repr(packed)]` |
| align | `u` | `#[repr(align(N))]`；沒有時為 `0` |
| field names | `list<str>` | 供偵錯資訊使用 |

**types**：型別紀錄的 `list`。紀錄只參照它之前的型別，因此可以一次讀完整個表；相等的型別只有一筆紀錄。

| 標籤（`u8`） | 型別 | 接著 |
|-----|------|------|
| 0 | `()` | |
| 1 | 基本型別 | `u8` 基本型別（見下文） |
| 2 | 未解析的字組 | |
| 3 | 元組 | `list<ty>` 欄位 |
| 4 | 結構 | `u` 結構編號、`list<ty>` 欄位 |
| 5 | `any Tr` 值 | （[8.8.9](../008-abi/808-lowering-types.md#889-representation-of-any-slots) 的 `(u32 tag, payload)` 組合） |
| 6 | 陣列 | `ty` 元素、`u` 長度 |
| 7 | 函式指標 | `list<ty>` 參數、`ty` 結果 |
| 8 | GC 參照 | |
| 9 | union | `list<(u32 tag as u, ty)>` 變體、`u` 字組數、`u8` 種類：`0` 和型別的酬載、`1` 一般 |

基本型別：`0` `i8`、`1` `i16`、`2` `i32`、`3` `i64`、`4` `isize`、`5` `u8`、`6` `u16`、`7` `u32`、`8` `u64`、`9` `usize`、`10` `f32`、`11` `f64`、`12` `bool`、`13` `c_char`、`14` `char`、`15` `str`。

在其標籤與運算元之後，每筆紀錄都有其配置，即 `nabic` 為目標所計算的配置（[8.11.5](../008-abi/811-backend-contract.md#8115-data-layout)）：

| 欄位 | 編碼 | 意義 |
|-------|----------|---------|
| size、align | `u`、`u` | 以位元組計 |
| offsets | `list<u>` | 元組與結構：每個欄位的偏移；其他型別為空 |
| roots | `list<u>` | 影子堆疊框架為這個型別之區域變數所註冊之字組的偏移，遞增排列（[8.11.6](../008-abi/811-backend-contract.md#8116-gc-allocation-roots-and-barriers)）；型別不容納 GC 參照時為空 |
| barrier | `b` | 把這個型別存入 GC 物件需要寫入屏障 |

函式值是函式指標與 GC 參照的元組 `(code, env)`；會擲回錯誤之函式的承載結構是元組 `(is_err, payload)`；和型別是種類 0 的元組 `(i32, union)`。型別表沒有其他種類。

## 9.11.7. GC 型別與常數

**GC types**：單元之 GC 型別的 `list`，每個：

| 欄位 | 編碼 | 意義 |
|-------|----------|---------|
| id | `u` | 型別 id，當程式碼可以把它當作常數使用時；否則為 `0` |
| symbol | `str` | 在單元建置中，容納該 id 的 `_Y<pkg>_gctype.<hash>` 符號（[8.11.3](../008-abi/811-backend-contract.md#8113-symbols-and-linkage)）；否則為空 |
| table | `list<u>` | ScanTable 的 `u32` 字組（[8.2.3](../008-abi/802-scan-table.md#823-serialization-format)），僅供讀取：由共用單元發出並註冊它 |

進行配置的 rvalue 以其在這個清單中的索引指名其 GC 型別。

**常數**（`const`），依標籤 `u8`：

| 標籤 | 常數 | 接著 |
|-----|----------|------|
| 0 | `()` | |
| 1 | `bool` | `b` |
| 2 | 有號整數 | `w` 值、`u8` 基本型別 |
| 3 | 無號整數 | `w` 值、`u8` 基本型別 |
| 4 | 浮點數 | `u` 位元（`f32` 的位於低 32 位元）、`u8` 基本型別 |
| 5 | `char` | `u` 碼位 |
| 6 | 字串 | `str` |
| 7 | 位元組 | `u` 長度、那麼多個位元組 |
| 8 | none | |

常數取得它被使用之位置的型別：作為浮點數使用的整數常數就是轉換後的該數字，而 `()` 與 none 是該型別的零值。

## 9.11.8. static

**statics**：static 紀錄的 `list`：

| 欄位 | 編碼 | 意義 |
|-------|----------|---------|
| name | `str` | 其項目路徑 |
| symbol | `str` | 其連結器符號（[8.11.3](../008-abi/811-backend-contract.md#8113-symbols-and-linkage)） |
| type | `ty` | |
| flags | `u` | 位元 0：`mut`；位元 1：定義在程式之外（`extern`）；位元 2：經由 `__imp_<symbol>` 取用（DLL 的，Windows）；位元 3：延遲（[8.11.10](../008-abi/811-backend-contract.md#81110-statics)）；位元 4：定義在這個程式自己的目的檔中（否則在格式有 GOT 時經由 GOT） |
| init | `opt<const>` | 初始值，僅供讀取：由共用單元定義儲存空間 |

## 9.11.9. 函式

**functions**：函式紀錄的 `list`。單元所定義的函式有主體；單元所呼叫或取得其位址的其他任何函式都是宣告（沒有區塊）。

| 欄位 | 編碼 | 意義 |
|-------|----------|---------|
| name | `str` | nabi-mir 名稱 |
| path | `str` | 項目路徑（泛型實例的：其泛型的路徑） |
| symbol | `str` | 主體的符號 |
| C entry | `str` | `extern "C"` 定義：C 進入點的符號；否則為空 |
| thunk | `str` | 當單元取得函式的位址且它接受 `nabi_tls` 時，函式值 thunk 的符號（`<symbol>.v`）；否則為空 |
| define | `u8` | `0` 宣告、`1` 區域、`2` 全域、`3` weak 全域 |
| flags | `u` | 見下文 |
| ABI | `u8`，然後 | `0` Nabi；`1` C：`b` 可變參數、`b` `extern "system"` |
| C classification | `opt<cabi>` | 供需要它的 C ABI 函式與 `extern "nabi"` 宣告使用（[8.11.4](../008-abi/811-backend-contract.md#8114-calling-conventions)） |
| wasm import | `opt<(str, str)>` | wasm32 匯入的模組與名稱 |
| position | `pos` | 宣告 |
| parameters | `list<local>` | |
| parameter types、result | `list<ty>`、`ty` | |
| locals | `list<(local, str name, ty)>` | 每個區域變數，包括參數；偵錯資訊省略者名稱為空 |
| pins | `list<local>` | 在其 `StorageDead` 之前保持註冊的區域變數 |
| blocks | `list<block record>` | 第一個是進入區塊 |
| debug | `opt<debug record>` | 9.11.11；有 `-g` 時存在 |

函式旗標：

| 位元 | 意義 |
|-----|---------|
| 0 | 以隱藏的最後一個引數接受 `nabi_tls` 位址 |
| 1 | 永不到達安全點：沒有框架、沒有輪詢，對它的呼叫不是安全點（[8.11.7](../008-abi/811-backend-contract.md#8117-safepoints-and-polls)） |
| 2 | 可重入：在其第一個安全點之前輪詢 |
| 3 | 以直接的 Nabi 呼叫以外的方式進入：在進入處輪詢 |
| 4 | `#[track_caller]`：最後一個參數是呼叫者的位置（`str`） |
| 5 | `#[no_safepoint]` 執行環境宣告 |
| 6 | `#[cold]` |
| 7 | `#[inline(never)]` |
| 8 | 使用者的 `main`（由共用單元的 C `main` 呼叫它） |
| 9 | `any` 分派器 |

C 分類 `cabi`：`u8` 結果（`0` 直接、`1` 強制轉換，接著一個 `ctype`；`2` 經由 `sret`）、引數的 `list`（`0` 直接、`1` 強制轉換，接著一個 `ctype`；`2` 以參照，接著 `b` byval）、`b` 可變參數、`b` `str` 作為 `(ptr, len)` 組合、`b` stdcall。`ctype` 是 `u8` `0` 純量，接著 `cscalar`；`1` `u` 數量與 `cscalar`（連續的暫存器）；`2` `list<cscalar>`（欄位自然對齊的結構）；`3` `list<(u offset, cscalar)>`。`cscalar` 是 `u8` `0` 整數，接著 `u` 位元數；`1` `f32`；`2` `f64`；`3` 一個暫存器中的兩個 `f32`；`4` 指標。

**區塊紀錄**：`block` 編號、`list<local>` 參數、`u` 旗標（位元 0：一個被輪詢的回邊離開這個區塊，因此在其終結指令之前會執行輪詢）、`list<statement>`、終結指令。

**運算元**：`u8` `0` 接著 `const`，或 `1` 接著 `local`。

**陳述式**，依標籤 `u8`：

| 標籤 | 陳述式 | 接著 |
|-----|-----------|------|
| 0 | `local = rvalue` | `local`、rvalue |
| 1 | 陣列元素寫入 | `local` 陣列、`u` 位元組偏移、運算元索引、運算元值、`ty` 元素、`u` 長度 |
| 2 | storage dead | `local` |
| 3 | 緩衝區元素存入 | 運算元緩衝區、運算元索引、運算元值、`ty` 元素 |
| 4 | 位置 | `pos`、`u` 範圍、`u` 父範圍 |
| 5 | 原始存入 | 運算元位址、運算元值、`ty`、`b` volatile |
| 6 | 堆積欄位存入 | 運算元物件、`ty` 酬載（元組）、`u` 欄位、運算元值 |
| 7 | static 存入 | `static`、運算元值、`ty` |

**Rvalue**，依標籤 `u8`：

| 標籤 | Rvalue | 接著 |
|-----|--------|------|
| 0 | 使用 | 運算元 |
| 1 | 二元 | `u8` 運算子、`ty` 運算元型別、運算元、運算元 |
| 2 | 一元 | `u8` 運算子、`ty` 運算元型別、運算元 |
| 3 | 呼叫 | `fn`、`list<operand>`、`ty` 結果 |
| 4 | 結構值 | `ty`、`list<operand>` 欄位 |
| 5 | 欄位讀取 | 運算元、`u` 欄位、`ty` 欄位型別 |
| 6 | 轉型 | 運算元、`ty` 來源、`ty` 目標 |
| 7 | 位元轉型 | 運算元、`ty` 來源、`ty` 目標 |
| 8 | 替換欄位 | 運算元、`ty` 結構、`u` 欄位、運算元值 |
| 9 | 陣列值 | `ty` 元素、`list<operand>` |
| 10 | 陣列元素讀取 | `local` 陣列、`u` 位元組偏移、運算元索引、`ty` 元素、`u` 長度 |
| 11 | 函式位址 | `fn`、`ty` |
| 12 | static 讀取 | `static`、`ty` |
| 13 | 裝箱 | 運算元、`ty`、`u` GC 型別 + 1（`0`：值就是酬載字組，不配置任何東西） |
| 14 | 緩衝區配置 | `ty` 元素、運算元長度、`u` GC 型別 |
| 15 | 原始載入 | 運算元位址、`ty`、`b` volatile |
| 16 | 區域變數位址 | `local` |
| 17 | static 位址 | `static` |
| 18 | 大小或對齊 | `ty`、`b` 對齊 |
| 19 | 緩衝區元素讀取 | 運算元緩衝區、運算元索引、`ty` 元素 |
| 20 | 緩衝區原子操作 | `u8` 運算、運算元緩衝區、運算元索引、`list<operand>`、`ty` 元素、`u8` 順序、`u8` 失敗時順序 |
| 21 | box 讀取 | 運算元、`ty` |
| 22 | 堆積配置 | `ty` 酬載、`u` GC 型別 |
| 23 | 堆積欄位讀取 | 運算元物件、`ty` 酬載、`u` 欄位 |
| 24 | 間接呼叫 | 運算元被呼叫者、`list<operand>`、`list<ty>` 參數、`ty` 結果、與函式紀錄相同的 ABI、`opt<cabi>` |

運算子（`u8`）：`0` `+`、`1` `-`、`2` `*`、`3` `/`、`4` `%`、`6` 取負、`7` `!`、`8` `&`、`9` `|`、`10` `^`、`11` `<<`、`12` `>>`、`13` `==`、`14` `!=`、`15` `<`、`16` `<=`、`17` `>`、`18` `>=`、`19` `&&`、`20` `||`、`21` 邏輯非、`22` 平方根、`23` 回繞 `+`、`24` 回繞 `-`、`25` 回繞 `*`、`26` 1 的個數、`27` 前導零、`28` 尾隨零、`29` 位元組交換。`5` 保留（向下取整的 `%%`，它絕不會到達程式碼產生器）。比較的結果型別就是其目的地的型別。

原子運算：`0` 載入、`1` 存入、`2` 比較交換（引數：預期值、新值）、`3` 交換、`4` 加、`5` 減、`6` and、`7` or、`8` xor。順序：`0` relaxed、`1` acquire、`2` release、`3` acq_rel、`4` seq_cst。

**終結指令**，依標籤 `u8`：`0` 跳躍（`block`、為其參數提供的 `list<operand>` 引數）、`1` 分支（運算元條件、為真時的 `block` 與 `list<operand>`、為假時的 `block` 與 `list<operand>`）、`2` 回傳（`opt<operand>`）、`3` unreachable。

## 9.11.10. 意義

每個陳述式、rvalue 與終結指令的意義就是它在 nabi-mir 中的意義；[8.11](../008-abi/811-backend-contract.md) 說明程式碼產生器要在其上加入什麼（檢查、屏障、根、輪詢、配置）。nabi-mir 值所參照、但不在檔案中的資料，屬於共用單元或另一個目的檔，經由符號取用。

## 9.11.11. 偵錯資訊

函式的偵錯紀錄（有 `-g` 時）：

| 欄位 | 編碼 | 意義 |
|-------|----------|---------|
| local types | `list<(local, dty)>` | 對於只憑型別無法得知的具名區域變數，偵錯工具要顯示什麼 |
| bindings | `list` | 指派被最佳化器移除的具名區域變數：`local`、`opt<operand>` 在它們可見之處所容納的值、`list<opt<operand>>` 就地建立之結構的欄位、`u` 行、`u` 範圍、`u` 父範圍 |
| inlined | `list` | 被內嵌的主體：`u` 範圍、`u` 父範圍、`str` 被呼叫者路徑、`pos` 宣告、`pos` 呼叫、`list<local>` 其參數的複本 |
| results | `list` | 被內嵌之主體的結果所指派到的區域變數：`local`、`u` 行、`u` 範圍、`u` 父範圍 |

debug 區段容納 `list<(u struct, list<dty>)>`，即依結構排列的欄位偵錯型別，以及 `list<(u tag, str name, dty)>`，即依標籤排列、存放在 `any` 值中的型別。`dty` 是 `u8` `0` 接著 `ty`（依型別所示）；`1` 和型別：`str` 名稱、`ty`、`list<(u tag, str name, dty)>`；`2` GC 緩衝區：`str` 名稱、`dty` 元素；`3` `any`；`4` 裝箱的值：`str` 名稱、`dty`；`5` 被擷取之變數的儲存格：`dty`；`6` 函式值：`str` 名稱、`ty`、`dty` 環境。

## 9.11.12. 版本與相容性

標頭中的 `major` 與 `minor` 是格式的版本。

- 讀取者接受其 `major` 是它能讀取的檔案。在同一個 `major` 內，較高的 `minor` 只會新增區段種類，較舊的讀取者會略過它們；既有區段的編碼，因而每個陳述式、rvalue、型別與常數標籤，都不會改變。
- 對既有區段的任何改變都是新的 `major`：新的陳述式、rvalue 或型別種類、新的欄位、改變的意義。讀取者會拒絕它無法讀取的 `major`，並指出兩個版本（[8.11.13](../008-abi/811-backend-contract.md#81113-separately-installed-code-generators) 的結束狀態 2）。
- 當 `major` 是 `0` 時（本草案），讀取者只接受它建置時所針對的確切 `major.minor`。
- 元件的版本就是其工具鏈的版本（[8.11.13](../008-abi/811-backend-contract.md#81113-separately-installed-code-generators)），因此工具鏈的 `nabic` 與其元件一律一致；版本規則對在工具鏈之外建置的元件才有意義。

寫出是確定性的：相同的單元產生相同的位元組。字串、型別與函式依寫出者第一次遇到它們的順序編號，依 nabi-mir 順序走訪單元的函式；對應表依鍵排序寫出。
