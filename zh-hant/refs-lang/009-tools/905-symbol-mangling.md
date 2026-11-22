# 9.5. 符號修飾方案

由於 [AOT 建置管線](902-aot-pipeline.md)中的 [specialize](../000-meta/007-glossary.md#特化specialize)，一個函式會依每個具體型別分成個別的函式。本節定義賦予這些函式唯一符號名稱的規則。

## 9.5.1. 形式

符號的整體文法如下：

```
mangled_symbol  ::= "_Y" pkg_hash "_" mod_path "_" item_name "_" type_hash
pkg_hash        ::= [0-9a-f]{8}            // package SHA-256 prefix
mod_path        ::= identifier ("_" identifier)*
item_name       ::= identifier
type_hash       ::= [0-9a-f]{16}           // generic args SHA-256 prefix
                  | ""                       // concrete (non-generic) item
```

寫成一行：

```
_Y <pkg_hash_8> _ <path> _ <name> _ <typehash_16>
```

實際的例子：

```
_Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
   |       |       |    |
   |       |       |    +-- typehash (16 chars, hash of generic args)
   |       |       +-- name (push)
   |       +-- path (std_vec)
   +-- pkg hash (8 chars)
```

## 9.5.2. 各部分的意義

### `_Y`：Nabi 符號前綴

每個 Nabi 符號都以 `_Y` 開頭。以底線 + 大寫字母開頭的名稱在 C 中保留給實作使用，因此它們不會與一般的 C 符號衝突；選擇 `_Y` 也是為了不與廣泛使用的修飾方案重疊（C++ Itanium 的 `_Z`、Rust v0 的 `_R`、D 的 `_D`、Swift 的 `$s`）。它也讓 Nabi 所發出的符號在 `nm` 或連結器錯誤輸出中容易辨識。前綴之後緊接著小寫十六進位的 `<pkg_hash_8>`，因此只憑第一個字元就能與後面接大寫字母的保留前綴（`_YK` 等，[9.5.5](#955-保留前綴)）區分。

### `<pkg_hash_8>`

[原始碼單元](../000-meta/003-source-units.md)的套件雜湊：以內容為基礎之 SHA-256 雜湊的前 8 個字元。

```
3f8a1c2b
```

內容相同的套件無論在哪裡建置都會產生相同的雜湊。因此，不同套件中的同名函式絕不會衝突。

### `<path>`：模組路徑

[路徑](../005-modules/507-paths.md)中定義的模組路徑，把 `::` 換成 `_`。標準函式庫的路徑保留其層級（`std::vec` → `std_vec`），使用者路徑則去掉 `pkg::`。函式庫的相依者經由函式庫的名稱指稱其項目，因此在函式庫中，路徑以該名稱開頭（函式庫 `app` 的 `pkg::geom::f` → `app_geom_f`）。

```
std::vec      → std_vec
pkg::geom   → geom               (pkg:: is already identified by the package hash)
foo::inner::sub → foo_inner_sub
```

### `<name>`：項目名稱

[識別字](../001-lexical/104-identifiers.md)中定義的識別字。`<path>` 的每個片段也依相同規則跳脫。

由於 `_` 是分隔符號，識別字中 `[A-Za-z0-9]` 以外的字元依下列方式跳脫。

| 原始 | 編碼 |
|---|---|
| `[A-Za-z0-9]` | 不變 |
| `_` | `_0` |
| 其他任何字元（包括 Unicode） | `_1<小寫十六進位碼位>_` |

識別字不以數字開頭，因此 `0` / `1` 絕不會接在分隔符號 `_` 之後。所以跳脫與分隔符號一律可以區分。

```
mod a { fn b }   → _Y<pkg>_a_b_
fn a__b          → _Y<pkg>__a_0_0b_
fn α             → _Y<pkg>___13b1__
```

### 方法名稱

方法以 `<type>.<method>` 作為其名稱、以定義該型別的模組作為其路徑進行修飾。若多個 trait 為同一個型別提供同名的方法，則附加 `__by_<trait>`。相依套件（[9.2.3](902-aot-pipeline.md#923-相依套件解析)）的方法也以這個穩定的路徑 `<module>::<type>.<method>[__by_<trait>]` 修飾，因此分開建置的套件與程式會產生相同的符號。`.` 與 `_` 依上表跳脫。

### 巢狀函式

巢狀函式（[4.10](../004-functions/410-nested-functions.md)）在其外圍項目的模組中，以 `<outer>/<name>` 作為其名稱進行修飾，其中 `<outer>` 是外圍函式的名稱（方法為 `<type>.<method>`，trait 預設主體為 `<trait>.<method>`，巢狀函式則遞迴地為 `<outer>/<name>`）。同一個原始碼檔案中有相同 `<outer>/<name>` 的兩個巢狀函式，依原始碼順序得到 `/2`、`/3`……。`/` 不會出現在其他任何項目名稱中，因此該符號與模組路徑及方法都不同。`/` 依表跳脫（`_12f_`）。

```
mod a { fn outer() { fn helper() } }   → _Y<pkg>_a_outer_12f_helper_
```

### `<typehash_16>`：型別組合雜湊

[泛型](../002-types/210-generics.md)的具體型別組合以 SHA-256 雜湊，並使用前 16 個字元。

```
fn id<T>(x: T) -> T

call id(42i32)   → typehash = SHA-256("(i32)")[0..16]
call id(3.14f64) → typehash = SHA-256("(f64)")[0..16]
```

### 具體 `fn` 的 typehash

對不接受泛型引數的函式，typehash 為空，因此符號以分隔符號 `_` 結尾。

```
_Y3f8a1c2b_geom_compute_            (empty typehash)
```

泛型實例一律有其 16 個字元，因此空的 typehash 明確表示沒有泛型引數。

## 9.5.3. 比較：Rust 的 `_ZN`

Rust 的 Itanium 風格修飾看起來像這樣。

```
_ZN3std3vec10push_inner17h2d4c3e9d8f1a2b3cE
```

- 每個組成部分都帶有長度前綴（`3std3vec`）。
- 包含一個 [ABI](../000-meta/007-glossary.md#abi應用程式二進位介面) 識別雜湊（17h...）。
- 以 `E` 結尾。

它有 Itanium 相容的優點，但難以閱讀，而且長度前綴的編碼效率不佳。

相較之下，Nabi 的 `_Y...` 容易閱讀，也沒有長度前綴（組成部分以 `_` 分隔）。解修飾也保持簡單。

## 9.5.4. demangle 工具

把符號解碼為人類易讀的形式：

```
nabi demangle <symbol>
```

輸出看起來像這樣：

```
$ nabi demangle _Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
package:  3f8a1c2b
path:     std::vec
name:     push
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.5.5. 保留前綴

使用中與保留的前綴如下：

| 前綴     | 意義                                                                           |
|----------|--------------------------------------------------------------------------------|
| `_Y...`  | 一般的 Nabi 符號                                                                |
| `nabi_*`  | `extern` "C" 執行環境函式（[Extern "C" 符號](../008-abi/805-extern-symbols.md)） |
| `_YK...` | （保留）關鍵字處理器                                                           |
| `_YL...` | （保留）字面值處理器                                                           |
| `_YI...` | （保留）中綴 / 前綴 / 後綴魔法                                                 |
| `_YM...` | （保留）meta 區塊產生的程式碼（impl_derive 等）                                |

`_YK` / `_YL` / `_YI` / `_YM` 只保留命名空間；所有一般的符號都以 `_Y` 前綴發出。

## 9.5.6. 符號長度

大多數 Nabi 符號的長度分布如下。

```
_Y + 8 + _ + path (tens) + _ + name (a few to tens) + _ + 16
≈ 30 ~ 80 characters
```

非常長的路徑加上深層的[泛型](../000-meta/007-glossary.md#泛型generic)可能超過 100 個字元。偵錯工具與其他工具都能毫無問題地處理這樣的長度（一般建議在 2KB 以下）。

## 9.5.7. 避免衝突：typehash 的穩定性

泛型引數的型別組合先以確定性的方式序列化，再以 SHA-256 雜湊。

```
type tuple → canonical serialization → SHA-256 → first 16 characters
```

無論建置重做多少次，相同的型別都產生相同的雜湊。不同建置中的相同特化也得到相同的符號名稱。nabic 給閉包與其他產生之函式的名稱衍生自它們所寫在的項目（對實例而言，還包括其型別引數），因此它們在每次建置中也都相同。

這種穩定性讓分開建置、增量建置以及相依套件之間的符號比對成為可能。

### 標準序列化規則

型別組合的序列化是確定性的。

- 基本型別寫成簡短的識別字（`i32` / `f64` 等）。
- 使用者型別採用 `{pkg_hash}**{path}**{name}` 加上其泛型引數的形式。
- 和型別的成員先正規化順序再串接（遵循[和型別（`A | B`）](../002-types/206-sum-types.md)的集合語意）。
- `struct` 由其欄位型別組成（欄位名稱無關）。

確切的正規化演算法由 `nabic` 選擇。本規格只要求它是確定性的。

## 9.5.8. `extern` "C" 函式：不修飾

[`extern` 區塊](../005-modules/509-extern-block.md)中宣告的外部函式原樣使用該 ABI 的標準命名慣例。不套用 Nabi 修飾。

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;     // symbol: "malloc" (as is)
}
```

在另一個方向上，`#[export_name("name")]` 在對外公開 `extern "C"` 函式時直接指定名稱，而 `#[export]` 以其自己的名稱匯出它（見 [`extern` 區塊（[FFI](../000-meta/007-glossary.md#ffi外部函式介面) 宣告）](../005-modules/509-extern-block.md)）。這樣的項目是消除無法到達之函式時的根，一律會被發出（[9.2.8](902-aot-pipeline.md#928-specialize單態化)）。

```nabi
#[export_name("nabi_compute")]
pub extern "C" fn compute(x: i32, y: i32) -> i32 { x + y }
// symbol: "nabi_compute" (not mangled)
```

這兩種機制提供 Nabi 與 C 之間的整合。

## 9.5.9. main 函式的符號

[原始碼單元](../000-meta/003-source-units.md)的進入點函式原樣使用平台一般的 main 符號。

```
main                                (Linux / macOS)
mainCRTStartup                      (Windows console)
WinMainCRTStartup + WinMain         (Windows GUI, -C subsystem=windows)
_start                              (no-OS)
__main_void                         (wasm32-wasip*; called by wasi-libc's _start)
_initialize, then main              (wasm32-unknown-unknown; reactor exports called by the host)
```

在 `wasm32-unknown-unknown` 上，模組是一個 reactor：主機呼叫 `_initialize`，然後呼叫匯出的 `main`，之後執行環境保持運作（不關閉）。`#[export_name("x")]` 函式是名為 `x` 的 wasm 匯出。在每個 wasm32 目標上，執行環境在 `__main_void`、`main` 或匯出之呼叫中最先發生者時啟動。

在 Windows 上，Nabi 自己的啟動目的檔提供 `mainCRTStartup`（主控台）或 `WinMainCRTStartup` 與 `WinMain`（GUI），它們呼叫 nabic 產生的包裝函式 `nabi_main`；Windows 執行檔沒有 `main` 符號。在偵錯資訊（DWARF 與 CodeView）中，該包裝函式在每個目標上都是 `nabi_main`，在 C 函式庫呼叫它之處其符號仍是 `main`，因此在回溯中只有程式自己的 `main` 叫做 `main`。

平台進入點在 nabic 產生的包裝函式內呼叫使用者定義的 main 函式。包裝函式依序呼叫 `nabi_runtime_init` 與型別註冊函式 `nabi.register_types`（[8.2.13](../008-abi/802-scan-table.md#8213-註冊時機)），然後呼叫進入函式，依 [4.16.4](../004-functions/416-entry-point.md#4164-結束碼與未處理的例外) 的規則決定結束碼，並經由 `nabi_runtime_shutdown` 回傳。

當 `main.nabi` 由頂層陳述式組成時（[4.16.2](../004-functions/416-entry-point.md#4162-頂層陳述式的意義隱含-main)），編譯器合成的隱含 `main` 就是進入函式。隱含的 `main` 以與一般函式相同的規則修飾，但無法從使用者程式碼以名稱參照；只有包裝函式會呼叫它。

## 9.5.10. 偵錯資訊

每個修飾後的符號在 DWARF 與 PDB 偵錯資訊中都帶有其原始名稱。

```
debugger output.
  fn distance<f64> at geom.nabi:42
    (mangled: _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190)
```

這讓 gdb、lldb 與 VS Code 等工具能以原始名稱顯示 Nabi 函式。協程的恢復函式命名為 `<creator>::{coroutine#k}`，其中 `k` 把該建立者的閉包與協程一起計數。

## 9.5.11. 穩定性

符號修飾方案也是 ABI 的一部分。因此變更必須遵循[概觀](../008-abi/README.md)中的穩定性方針。

- 在 1.0 之前，方案可以在每個 0.x.0 版本中改變，並在一個 0.x 系列內保持不變。
- 從 1.0 起，在同一個主版本內維持相容；跨主版本時會破壞相容，ABI 雜湊也會改變。
