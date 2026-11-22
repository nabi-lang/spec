# 6.2. 內建屬性

本節定義由編譯器直接處理的內建屬性。`#[cfg]` 另見[條件式編譯](603-cfg.md)，魔法屬性與元區塊則在 [`#[lang(...)]` 魔法](604-lang-magic.md)起的各節中說明。

```
derive_attr     ::= "#" "[" "derive" "(" trait_name ("," trait_name)* ","? ")" "]"
trait_name      ::= identifier
repr_attr       ::= "#" "[" "repr" "(" repr_arg ("," repr_arg)* ")" "]"
repr_arg        ::= "C" | "transparent" | "packed" | "align" "(" int_lit ")"
                  | identifier                          // primitive int repr
inline_attr     ::= "#" "[" "inline" ("(" ("always" | "never") ")")? "]"
deprecated_attr ::= "#" "[" "deprecated" ("(" deprecated_args ")" | "=" string_lit)? "]"
deprecated_args ::= string_lit | deprecated_kv ("," deprecated_kv)*
deprecated_kv   ::= ("since" | "note") "=" string_lit
deprecated_safe ::= "#" "[" "deprecated_safe" ("(" string_lit ")" | "=" string_lit)? "]"
sealed_attr     ::= "#" "[" "sealed" "]"
internal_attr   ::= "#" "[" "internal" "]"
discardable_attr ::= "#" "[" "discardable" "]"
```

## 6.2.1. `#[derive(...)]` 自動 trait 實作

```nabi
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
struct Point { x: i64; y: i64; }
```

`#[derive(...)]` 的引數是以逗號分隔的 trait 名稱。對每個列出的 trait，會產生適合該型別的自動實作。

### 內建可衍生的 trait

只有 `Clone`、`Copy`、`Debug`、`Default`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash` 與 `Error` 可以衍生。各 trait 的產生規則與要求定義於 [stdlib 2.13.11](../../refs-standard-libraries/200-core/213-marker.md#21311-可衍生的-trait)。

- `Display` 不能衍生。顯示格式由型別的作者決定，以 `impl Display` 或 `#[display("..")]` 撰寫（[stdlib 2.10.14](../../refs-standard-libraries/200-core/210-fmt.md#21014-衍生政策摘要)）。
- `Error` 以其預設的 `source()` 實作 `Error`；`Display` 與 `Debug` 來自它們自己的 derive、屬性或 impl（[stdlib 2.14](../../refs-standard-libraries/200-core/214-error.md#deriveerror)）。
- 衍生 `Eq` / `Ord` 而沒有其父 trait（`PartialEq` / `PartialOrd`、`Eq`）是編譯錯誤。`Copy` 需要 `Clone`，且不能與 `Drop` 結合。
- 在 `union` 上使用 `#[derive]` 是編譯錯誤。

### 與使用者定義的 derive 結合

使用者定義的 trait 也可以衍生，例如 `#[derive(MyTrait)]`。這需要 `MyTrait` 由 `#[impl_derive(MyTrait)]` 元區塊定義（[6.8](608-meta-blocks.md)）。上述內建 derive 名稱不能由元區塊重新定義。

內建與使用者定義的 derive 可以在一個 `#[derive(...)]` 中自由混用。

```nabi
#[derive(PartialEq, Eq, Clone, MyCustomTrait)]
struct Foo { ... }
```

## 6.2.2. `#[repr(...)]` 明確的記憶體表示

```nabi
#[repr(C)]
struct CCompatible { x: i32; y: i32; }

#[repr(transparent)]
struct Wrapped(InnerType);

#[repr(packed)]
struct Tight { a: u8; b: u32; }

#[repr(align(16))]
struct Aligned { ... }

#[repr(u32)]
enum Status { ... }
```

| 引數                        | 意義                                                                                                 |
|-----------------------------|------------------------------------------------------------------------------------------------------|
| `C`                         | C [ABI](../000-meta/007-glossary.md#abi應用程式二進位介面) 配置：宣告順序、自然對齊。與預設的 `struct` 配置相同 |
| `transparent`               | 恰好一個欄位，且沒有其他 `repr` 引數。與該欄位表示相同，並以它的方式跨越 C ABI |
| `packed`                    | 忽略對齊而緊密封裝（僅限 [FFI](../000-meta/007-glossary.md#ffi外部函式介面)）。不能容納 GC 參照；與 `align(N)` 互斥 |
| `align(N)`                  | 強制最小對齊為 N 位元組                                                                              |
| `u8` 到 `u64` / `i8` 到 `i64` | `enum` 的標籤型別。32 位元以外的寬度只用於無欄位的列舉；帶資料的列舉接受 `i32` / `u32` |

沒有 `#[repr]` 時，[`struct`](../002-types/203-struct.md) 使用宣告順序與自然對齊（與 `repr(C)` 相同）；[`enum`](../002-types/204-enum.md) 與和型別的配置由編譯器決定（[2.7](../002-types/207-niche-optimization.md)）。

## 6.2.3. `#[link(...)]` 指名外部函式庫

```nabi
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
}

#[link(name = "openssl", kind = "static")]
extern "C" { ... }
```

`#[link(...)]` 指名 [`extern` 區塊](../005-modules/509-extern-block.md)所連結的外部函式庫；它附加在 `extern` 區塊上。

| 引數形式                    | 意義                        |
|-----------------------------|-----------------------------|
| `"libname"`                 | 單純的函式庫名稱            |
| `name = "..."`              | 函式庫名稱（鍵值）          |
| `kind = "static" / "dylib" / "framework"` | 連結種類（`framework`：Apple 框架） |
| `wasm_import_module = "m"` | wasm32：區塊的函式從核心模組 `m` 匯入；在其他地方忽略 |

### `#[link_name = "sym"]`

extern 函式上的 `#[link_name = "sym"]` 給出它所連結的符號，或在 `wasm_import_module` 區塊內給出其匯入名稱。

```nabi
#[link(wasm_import_module = "env")]
extern "C" {
    #[link_name = "log_line"]
    fn host_log(ptr: ConstPtr<u8>, len: usize);
}
```

## 6.2.4. `#[inline]` 內嵌提示

```nabi
#[inline]
fn small_helper(x: i32) -> i32 { x + 1 }

#[inline(always)]
fn very_hot() { ... }

#[inline(never)]
fn never_inline() { ... }
```

| 引數     | 意義                                          |
|----------|-----------------------------------------------|
| （無）   | 建議內嵌（由編譯器決定）                      |
| `always` | 強制內嵌                                      |
| `never`  | 禁止內嵌                                      |

`#[inline]` 只是最佳化提示；它不影響程式的意義。

## 6.2.5. `#[test]` 測試函式標記

```nabi
#[test]
fn test_addition() {
    assert_eq(1i32 + 1i32, 2i32);
}
```

`assert_eq` 見 [stdlib 2.14.5](../../refs-standard-libraries/200-core/214-error.md#assert--assert_eq--assert_ne)。函式上的 `#[test]` 把它標示為測試函式。`nabic test` 把這類函式收集到測試框架中，測試執行器（`nabi test` 等）會自動執行它們。在一般建置中它們被排除，就像也附加了 `#[cfg(test)]` 一樣。無論其自身與外層模組的可見性為何，測試函式都會被執行；`#[test]` 不會為其他程式碼改變可見性。

有兩個屬性與 `#[test]` 搭配，說明通過的意義：

```nabi
#[test]
#[should_panic(expected = "empty input")]
fn rejects_empty() {
    parse_config("");
}

#[test]
#[ignore = "needs the network"]
fn downloads() { }
```

- `#[should_panic]` 只在測試 panic 時通過；`#[should_panic(expected = "text")]` 還要求 panic 的輸出包含 `text`。回傳了或以未捕捉之錯誤結束的測試會失敗。
- `#[ignore]`（或 `#[ignore = "reason"]`）讓測試不在一般執行中執行；它會被回報為已忽略，並附上理由。`nabi test --ignored` 只執行這類測試，`--include-ignored` 也會執行它們（[9.10.1](../009-tools/910-nabi-frontend.md#9101-子指令與全域選項)）。

在沒有 `#[test]` 的函式上使用其中任一者是錯誤，錯誤訊息會提示加上 `#[test]`；其他任何引數形式都是錯誤，錯誤訊息會顯示這兩種形式。

`#[bench]` 標示基準測試函式，並與 `#[test]` 一樣隱含 `#[cfg(test)]`。`nabic bench` 收集它們並建置測試框架，`nabi bench` 執行它（[9.1.2](../009-tools/901-nabic-cli.md#nabic-bench效能測試)）。無論其自身與外層模組的可見性為何，基準測試函式都會被執行；`#[bench]` 不會為其他程式碼改變可見性。

## 6.2.6. `#[cold]` 很少被呼叫的函式

```nabi
#[cold]
fn handle_error() { ... }
```

`#[cold]` 告訴編譯器該函式很少被呼叫。程式碼產生器於是把該函式放在熱程式碼路徑之外，以改善快取友善性。

## 6.2.7. `#[deprecated]` 不建議使用的標記

```nabi
#[deprecated]
fn old_api() { ... }

#[deprecated = "use new_api instead"]
fn old_api2() { ... }

impl Config {
    #[deprecated(since = "1.2.0", note = "use `Config::load`")]
    pub fn new() -> Config { ... }
}
```

使用標示為 `#[deprecated]` 的項目會產生警告（lint `deprecated`），而不是錯誤。以 `= "..."`、`("...")` 或 `note = "..."` 給出的訊息會與警告一起顯示；它應當說明應改寫成什麼。`since = "x.y.z"` 指出棄用該項目的版本。

警告涵蓋下列項目，無論它們宣告於何處（套件本身、相依套件、標準函式庫）：

- 函式與方法，無論被呼叫或作為值使用；trait 方法的標記涵蓋其每個 impl；
- 欄位，在被讀取、指派、在結構字面值中設定或在模式中指名時；
- 結構、列舉、聯集、型別別名與 trait，在型別中或作為路徑開頭被指名時（`Old::new`）；
- 常數與 static。

只有正在建置之套件中的使用會發出警告（[6.2.9](#629-allow--deny--warn-lint-控制)）。型別自己的 `impl` 不會因該型別而發出警告，結構自己的方法也不會因其欄位而發出警告。型別以其完整路徑辨識，或以沒有其他型別共用的最後一個片段辨識。這個屬性在任何項目上都被接受；在其他項目（變體、impl、模組）上它沒有效果。

`since` 僅供參考：任何版本的棄用都屬於 `deprecated` 與 `warnings`。語言伺服器會把該使用標記為已棄用，因此編輯器會以刪除線顯示它。

## 6.2.8. `#[doc = "..."]` 文件註解形式

```nabi
#[doc = "Returns the sum of two integers."]
fn add(x: i32, y: i32) -> i32 { x + y }
```

[語彙結構](../001-lexical/102-lexical-structure.md#文件註解)中的 `///` 文件註解就是這個屬性。

```nabi
/// This function ...
fn f() { ... }

// same meaning
#[doc = "This function ..."]
fn f() { ... }
```

內部形式 `#![doc]` 同樣與 `//!` 效果相同。

### `#[doc(hidden)]`

```nabi
/// What a dict spread checks (lowered code calls it).
#[doc(hidden)]
pub const fn spread_misfit<V>(d: Dict<str, V>, key: str, why: str) { .. }
```

`#[doc(hidden)]` 讓公開的項目不出現在 API 文件（`nabi doc`）與語言伺服器的自動完成中。它用於只是為了讓編譯器產生的程式碼能呼叫才設為 `pub` 的項目；該項目仍然是公開的，編譯與解析照舊。用在模組上時，它隱藏其中的一切。它可以與項目的文件註解並存，後者仍用於滑鼠懸停提示。其他任何 `#[doc(..)]` 形式都是錯誤：請寫 `#[doc = ".."]` 或 `#[doc(hidden)]`。工具鏈的套件之間共用、但使用者不得呼叫的項目是 `#[internal]`（[6.2.16](#6216-internal-由工具鏈的套件共用)），而不是 `#[doc(hidden)]`。

## 6.2.9. `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` lint 控制

```nabi
#[allow(unused)]
fn _placeholder() { }

#[deny(deprecated)]
fn strict_zone() { ... }
```

這些屬性在其所附加之項目的範圍內改變 lint 診斷的等級。lint 在每個編譯套件的指令中執行：`nabic build`、`test`、`bench`、`emit` 與 `check`（[9.1.2](../009-tools/901-nabic-cli.md#912-核心子指令)）、`nabi run`、`nabi lint`（[9.10.8](../009-tools/910-nabi-frontend.md#nabi-lintlinter)）、REPL 以及語言伺服器（`doc` 除外）。它們檢查正在建置的套件，而不檢查其相依套件。`warn` lint 會被印出，不會讓建置失敗；`deny` lint 是錯誤，建置以結束碼 1 失敗。`allow` 忽略該診斷，`warn` 以警告發出它，`deny` 以錯誤發出它。`#![allow(..)]` 之類的內部形式適用於外層模組（位於檔案開頭時為整個檔案）。引數是 lint 名稱或群組名稱。項目的 `#[allow]` 也涵蓋其變體、欄位與方法。

等級依下列順序決定，後者覆寫前者：每個 lint 的預設等級 < `nabi.nabion` 的 `lints` 欄位（經過其 `lints_from` 鏈之後）（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）< 命令列的 `-A` / `-W` / `-D` < 原始碼中的屬性（最內層者勝出）。`lints` 被視為放在命令列旗標之前，因此寫在它之後的命令列旗標會覆寫它。lint 診斷的錯誤碼是 `E0131`。

每個 lint 的預設等級都是 `warn`，但 `nan_comparison` 與 `self_assignment` 一律是錯誤寫法，預設為 `deny`，而 `missing_docs` 是 `allow`。Since 欄是加入該 lint 的版本。

| lint                     | Since   | 檢查                                                                |
|--------------------------|---------|---------------------------------------------------------------------|
| `unused_variables`       | 0.1.0   | 從未被讀取的區域變數或參數。被指派或以 `ref` 擷取（`[ref n]`）的 `ref` 參數算作已使用：寫入會到達呼叫者 |
| `unused_imports`         | 0.1.0   | 模組從未使用其名稱的 `use`                                          |
| `dead_code`              | 0.1.0   | 沒有任何人呼叫的非 `pub` 函式                                       |
| `unreachable_code`       | 0.1.0   | 位於 `return` / `break` / `continue` / `throw` / `panic` 之後的陳述式 |
| `unused_must_use`        | 0.1.0   | 捨棄 `#[must_use]` 函式的結果，或 `#[must_use]` 型別的值（[6.2.11](#6211-未使用的結果unused_resultdiscardablemust_use)） |
| `unused_result`          | 0.1.0   | 值不是 `()`、`none` 或 `Never` 的運算式陳述式，除非它呼叫 `#[discardable]` 函式；要刻意捨棄值，請寫 `_ = expr;`（[6.2.11](#6211-未使用的結果unused_resultdiscardablemust_use)） |
| `unused_doc_comments`    | 0.1.0   | 位於不會被撰寫文件之處（陳述式、match 分支、結構字面值或元組欄位、運算式）的文件註解；一般註解請用 `//` |
| `ignored_error`          | 0.1.0   | 默默捨棄錯誤的 `try? f();` 陳述式；`let _ = try? f();` 標示刻意盡力而為的呼叫，不會被回報 |
| `nan_comparison`         | 0.1.0   | 與 NaN 比較（一律為 false；`!=` 一律為 true）；請用 `.is_nan()`（deny） |
| `bool_comparison`        | 0.1.0   | 把 `bool` 與 `true` / `false` 比較                                  |
| `self_assignment`        | 0.1.0   | `x = x;`（deny）                                                    |
| `while_true`             | 0.1.0   | 以 `while true` 取代 `loop`                                         |
| `closed_type_switch`     | 0.1.0   | 後備路徑會 panic 的 `if T is ..` 鏈：封閉的型別集合適合用 trait     |
| `deprecated`             | 0.1.0   | 使用 `#[deprecated]` 的函式、方法、欄位、型別、常數或 static（[6.2.7](#627-deprecated-不建議使用的標記)） |
| `deprecated_safe`        | 0.1.0   | 在 `unsafe` 之外呼叫 `#[deprecated_safe]` 函式，或使用這種 static（[6.2.14](#6214-deprecated_safe-現在安全之後-unsafe)） |
| `redundant_cast`         | 0.1.0   | `x` 已經是 `T` 時的 `x as T`                                        |
| `redundant_clone`        | 0.1.0   | 對基本型別的值呼叫 `.clone()`                                       |
| `non_snake_case`         | 0.1.0   | 不是 snake_case 的函式、區域變數、參數或模組名稱                    |
| `non_camel_case_types`   | 0.1.0   | 不是 UpperCamelCase 的型別、trait 或變體名稱：不得以小寫字母開頭、不得有 `__`，`_` 只能位於沒有大小寫之分的字元之間（`V1_0`） |
| `non_upper_case_globals` | 0.1.0   | 不是 UPPER_SNAKE_CASE 的 `const` / `static` 名稱                    |
| `sup_path`               | 0.1.0   | 在行內 `mod name { }` 之外的 `sup::` 路徑或 `pub(sub)`；請寫從套件根開始的路徑（`pkg::a::b`，附註會指出它）或 `pub(pkg)`（[5.7](../005-modules/507-paths.md)） |
| `local_use`              | 0.1.0   | 函式主體內的 `use`；請把它移到檔案頂端，與其他 `use` 項目放在一起（[5.5.8](../005-modules/505-use.md)） |
| `confusable_idents`      | 0.1.0   | 彼此相似的識別字（[1.4](../001-lexical/104-identifiers.md)）        |
| `hidden_chars`           | 0.1.0   | 直接寫在字串、字元字面值或註解中的不可見或未指派字元；請把它寫成 `\u{..}` 跳脫序列（[1.1.2](../001-lexical/101-source-files.md#112-字元編碼)） |
| `missing_f_prefix`       | 0.1.0   | 含有 `{name}` 的一般字串字面值，而 `name` 是作用域內的區域變數：缺少 `f` 前綴 |
| `literal_spread_mismatch` | 0.1.0   | 展開到固定參數卻無法填滿它們的 `List` 或 Dict 字面值：長度不同，或鍵沒有指稱剩下的參數（[4.2](../004-functions/402-parameters.md)）；該呼叫在執行時會 panic |
| `assoc_type_shadowing`   | 0.1.0   | 與外部型別同名的關聯型別，它在其 trait 或 `impl` 內隱藏該外部型別（2.8.3） |
| `plain_module_doc`       | 0.1.0   | 函式庫的根檔案，或公開模組（其路徑上每個模組都是 `pub`）的檔案，沒有 `//!` 文件卻以 `//` 行開頭：它們讀起來像模組的文件，但並不是（[9.10.8](../009-tools/910-nabi-frontend.md#nabi-doc產生文件)）；說明會顯示應寫的 `//!` 行 |
| `missing_docs`           | 0.1.0   | 沒有 `///` 文件註解的 `pub` 項目、固有 impl 的 `pub` 方法、`pub` trait 的方法或行內 `pub mod`；`#[doc(hidden)]` 算作已撰寫文件，trait impl 的方法不受檢查（allow） |
| `resize_while_iterating` | 0.1.0   | 走訪 `List` 的 `for x in xs`，其主體改變 `xs` 的長度（`push`、`remove`……），而同一區塊中其後沒有 `break`、`return` 或 `throw`：迴圈會看到新的長度（[3.13.3](../003-expressions/313-loop-expressions.md#3133-for以迭代器為基礎的重複)） |

| 群組                | 包含                                                                                         |
|---------------------|----------------------------------------------------------------------------------------------|
| `unused`            | `unused_variables`、`unused_imports`、`dead_code`、`unreachable_code`、`unused_must_use`、`unused_result`、`unused_doc_comments` |
| `nonstandard_style` | `non_snake_case`、`non_camel_case_types`、`non_upper_case_globals`、`sup_path`、`local_use` |
| `warnings`          | 每個等級為 warn 的 lint，包括之後版本加入的                                                  |
| `all`               | 每個 lint，包括之後版本加入的                                                                |

`warnings` 採用 rustc 的意義：deny 它（在 `lints` 中寫 `"warnings": "deny"`、`-D warnings` 或 `#![deny(warnings)]`）會把每個警告都變成錯誤，包括之後版本加入的 lint 與棄用。必須在各版本之間持續能建置的建置，要固定其工具鏈（`nabi.nabion` 中的 `toolchain`，[9.10.6](../009-tools/910-nabi-frontend.md)）。每個 lint 的 Since 欄僅供參考。

有些警告來自編譯器本身而非 lint 階段，但仍屬於某個 lint：易混淆的識別字是 `confusable_idents`，字串或註解中的不可見字元是 `hidden_chars`，位置錯誤的文件註解是 `unused_doc_comments`。它們的等級與其他任何 lint 一樣，以該 lint 的名稱設定。

每個 lint 診斷都會說明其等級來自何處：`` `#[warn(unused_variables)]` on by default ``、`` `-D unused` on the command line ``、來自清單之等級的 `` `"unused_variables": "deny"` in `lints` of nabi.nabion ``，或在設定它的屬性上的標籤。未知的 lint 或群組名稱是警告，警告會指出最接近的已知名稱，並指向 `nabi lint --list`。在屬性、`-A` / `-W` / `-D` 或 `lints` 中指名已移除的 lint（大多已變成錯誤）會被接受，但沒有任何作用；即使在 `-D warnings` 之下也不會被回報。

`nabi lint --list` 印出各 lint 及其預設等級與加入它的版本，然後印出各群組。

有些發現附帶能解決它的修改（`while true` -> `loop`、刪除未使用的 `use`）；編輯器把它當作快速修正提供，而 `nabi fix`（[9.10.8](../009-tools/910-nabi-frontend.md#nabi-fix套用修正)）會在整個套件中套用它們。

`nabic check --check-library`（[9.1.2](../009-tools/901-nabic-cli.md#nabic-check診斷)）與 `nabi lint --check-library`（[9.10.8](../009-tools/910-nabi-frontend.md#nabi-lintlinter)）也會以標準函式庫的等級對其自身的程式碼執行 lint：每個預設等級，外加 `missing_docs` 為 `deny`：標準函式庫的每個公開項目都有文件註解（[stdlib 0.5.14](../../refs-standard-libraries/000-meta/005-notation.md#0514-文件與屬性)）。

## 6.2.10. `#[non_exhaustive]` 退出外部的窮盡性

```nabi
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    Unauthorized,
}
```

`#[non_exhaustive]` 可以附加在 `enum` 或 `struct` 上。在 `enum` 上，從外部套件對該型別進行的 `match`，即使處理了每個變體，也需要額外的全收分支（`_ => ...`）。這讓函式庫作者之後能新增變體而不破壞 SemVer 相容性。

```nabi
#[non_exhaustive]
pub struct ClientOptions {
    pub retries: i32 = 3;
    pub verbose: bool = false;
}

let o = ClientOptions { retries: 5 };           // OK in another package
let p = ClientOptions { verbose: true, ..o };   // OK
let ClientOptions { retries, .. } = p;          // `..` required in another package
```

在 `struct` 上，它標示一個其套件可能新增欄位的紀錄：

- 從外部套件對它進行的結構模式以 `..` 結尾（`let`、`match`、`if let`）；沒有 `..` 是錯誤，錯誤訊息會顯示加上 `..` 的模式。定義它的套件可以列出每個欄位。
- 結構第一個版本之後新增的每個欄位都有預設值（[2.3.3](../002-types/203-struct.md#233-欄位預設值)），因此省略它的字面值可以繼續編譯。編譯器看不到結構的歷史；這條規則是作者的責任，之後的 semver 檢查會強制它。
- 與 Rust 不同，其他套件中的結構字面值與 `..base` 展開（[2.3.4](../002-types/203-struct.md#234-欄位展開)）仍是允許的：字面值省略有預設值的欄位並指名其餘欄位。沒有合理預設值的欄位在每個字面值中都是必要的。

## 6.2.11. 未使用的結果：`unused_result`、`#[discardable]`、`#[must_use]`

```nabi
fn calculate() -> i32 { ... }

#[discardable]
fn insert(self, k: K, v: V) -> V | none { ... }

#[must_use("thread is never joined")]
pub struct JoinHandle<T> { ... }
```

值不是 `()`、`none` 或 `Never` 的運算式陳述式會以 lint `unused_result` 發出警告：`calculate();` 捨棄了一個 `i32`。經過 `try`、`try!` 與 `await` 時，值是該呼叫的值。`try? f();` 陳述式改由 `ignored_error` 回報，而類區塊陳述式（`if`、`match`、區塊、迴圈）不受檢查：其值是其分支的值。要刻意捨棄值，請寫 `_ = expr;`（[4.2](../004-functions/402-parameters.md)）或 `let _ = expr;`。

函式或方法上的 `#[discardable]` 表示其結果可以被捨棄：呼叫它的陳述式不會發出警告。它不接受引數。在 trait 方法上，它涵蓋每個 impl。標準函式庫標示了回傳其取代之值或附帶回報之值的方法：`Dict::insert` / `remove`、`Set::insert` / `remove`、`List::pop` / `remove` / `drain`、`Deque::pop_front` / `pop_back`、`BinaryHeap::pop`、`Atomic::swap` / `fetch_*`、`Cell::replace` / `take`、`marker::replace` / `take`、`mem::replace`、`Dir::copy`、`fs::copy`、`io::copy` 與 `io::copy_async`。短寫入（`Write::write -> usize`）、`checked_*` 以及 `str::trim` 之類的純方法不是 discardable 的。

`#[must_use]` 可以附加在函式或方法、`struct` 或 `enum` 上，並可帶有訊息（`#[must_use("..")]`）。在函式上，它加入訊息：捨棄其結果會以 `unused_must_use` 而非 `unused_result` 發出警告。在型別上它更強：值具有該型別的運算式陳述式，或從未被讀取的該型別區域變數，會以屬性的訊息產生 `unused_must_use` 警告，即使它是由 `#[discardable]` 函式回傳的也一樣。`async fn` 呼叫或 `async` 區塊的值（從未被 await 的 future 永遠不會執行）以及 `thread::JoinHandle`（[stdlib 3.4.2](../../refs-standard-libraries/300-std/304-thread.md#342-api)；從未被 join 的執行緒可能在程式結束前還沒完成；`detach` 表示它自行執行）是 `#[must_use]`；[stdlib 0.5.14](../../refs-standard-libraries/000-meta/005-notation.md#0514-文件與屬性) 列出其餘的。`#[must_use]` 型別就是屬性所在的那個型別：另一個套件中同名的型別不是。

型別的訊息寫成接在「this」之後：警告讀作 `this <message>`。第一個 `": "` 之後的文字成為 `help:` 行，其中 `{}` 代表被捨棄之運算式的原始碼（或區域變數的名稱）：`#[must_use("future is never awaited, so its body never runs: write `await {}`")]` 會警告 ``this future is never awaited, so its body never runs``，並附帶 ``help: write `await fetch()` ``。沒有訊息時，警告會指出型別：``this value of type `T` is never used``。

## 6.2.12. `#[track_caller]` 傳遞呼叫處

```nabi
#[track_caller]
fn expect_positive(n: i32) -> i32 {
    if n <= 0 { panic("not positive") }      // location: where expect_positive was called
    n
}
```

- 函式以一個隱藏的尾端 `str` 引數接收呼叫處。其中的 panic 會回報該位置（[4.14.13](../004-functions/414-throws.md#41413-panic-與致命錯誤)）。
- 當受追蹤的函式呼叫另一個受追蹤的函式時，它會傳遞它所收到的位置。
- 受追蹤函式的函式值回報建立該值的位置。
- 在 `async fn` 上，只有呼叫受追蹤，協程主體不受追蹤。
- 在 `extern`（執行環境）宣告上，符號會多接受一個位置引數（[8.5.7](../008-abi/805-extern-symbols.md#857-診斷區域)）。
- 它不能附加在 `main`、`extern "C" fn` 或 `#[export_name]` 函式上。

對無效輸入會 panic 的標準函式庫函式（`panic`、`unreachable`、`assert*`、索引、切片、`unwrap`、`List::insert` / `remove` 等）都是 `#[track_caller]`。

## 6.2.13. 總結

上述屬性是由編譯器與工具直接辨識的內建屬性；它們不需要使用者透過元區塊定義。

`#[lang]` 與 `#[literal]` 等魔法屬性與語言本身的意義繫結，定義於另外的文件。

### 其他內建屬性

編譯器也辨識下列屬性。每一個都在其功能所在之處規定。

| 屬性                           | 位置                          | 意義 |
|--------------------------------|-------------------------------|---------|
| `#[path("file.nabi")]`          | `mod name;`                   | 從該檔案而不是 `name.nabi` 載入模組（[5.4.6](../005-modules/504-modules.md#546-自訂路徑path)） |
| `#![no_implicit_nabi]`          | 檔案（內部）                  | 為該檔案關閉隱含的 `use core::prelude::*;`（[stdlib 0.2.6](../../refs-standard-libraries/000-meta/002-import.md#026-停用no_implicit_nabi)） |
| `#[export_name("sym")]`        | `extern "C"` / `extern "system"` fn | 以未修飾的符號 `sym` 匯出函式（在 wasm32 上是同名的 wasm 匯出）；是無法到達函式消除的根（[9.5.8](../009-tools/905-symbol-mangling.md)）。在一般函式上是錯誤（[5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-定義對外公開)） |
| `#[export]`                    | `extern "C"` / `extern "system"` fn | 以函式自己的名稱進行的 `#[export_name]`（[5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-定義對外公開)） |
| `#[on_panic(return = v)]`、`#[on_panic]` | `extern "C"` / `extern "system"` fn | 函式中的 panic 會被回報，然後函式回傳 `v`（整數、浮點數或 `bool` 字面值，null 指標為 `0`；沒有結果時不回傳任何東西），而不是中止行程（[5.9.10](../005-modules/509-extern-block.md#on_panic給-c-一個值而不是中止)） |
| `#[errno]`                     | `extern "C"` / `extern "system"` 區塊或宣告 | 每次呼叫在回傳之後立即儲存 errno（在 Windows 上還有 `GetLastError`），供 `bare::ffi::errno()` 使用（[stdlib 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errnoerrno-與-getlasterror)）；在 `extern "nabi"` 區塊與函式上是錯誤 |
| `#[leaf]`                      | `extern "C"` / `extern "system"` 區塊或宣告 | 呼叫略過其周圍的停駐：成本較低，但函式既不得阻塞也不得回呼（[stdlib 1.7.1](../../refs-standard-libraries/100-bare/107-ffi.md#leaf短呼叫)）；在 `extern "nabi"` 區塊上是錯誤 |
| `#[no_mangle]`                 | 函式                          | 讓函式免於命名與未使用函式的 lint。它不改變符號；那要用 `#[export_name]` |
| `#[wit::export]`               | 函式、`pub struct`            | 實作 WIT 匯出：函式或資源（[stdlib 1.8](../../refs-standard-libraries/100-bare/108-wit.md)） |
| `#[niche(..)]`                 | newtype 結構                  | 宣告該型別永不容納的值，用作 `T \| none` 的 niche（[2.7.4](../002-types/207-niche-optimization.md#274-選用的-nicheniche)） |
| `#[display("text")]`          | `struct`、`enum`、列舉變體    | 以寫出 `f"text"` 實作 `Display`（[stdlib 2.10.14](../../refs-standard-libraries/200-core/210-fmt.md#21014-衍生政策摘要)） |
| `#[intrinsic("nabi.OP.TYPE")]`  | 僅限標準函式庫                | 把函式或方法繫結到編譯器運算；在使用者程式碼中是錯誤（[stdlib 0.3](../../refs-standard-libraries/000-meta/003-intrinsics.md)） |
| `#[no_safepoint]`              | `extern "nabi"` 區塊           | 工具鏈內部：區塊的函式既不配置記憶體也不阻塞，因此對它們的呼叫不是安全點；在其他區塊上是錯誤（[5.9.2](../005-modules/509-extern-block.md#592-abi-指定符)） |
| `#[sealed]`                    | `trait`                       | 只有定義它的套件能實作該 trait（[6.2.15](#6215-sealed-只在定義套件中實作)） |
| `#[internal]`                  | 函式、方法、`const`、`static`、欄位 | 只由 `bare`、`core` 與 `std` 共用（[6.2.16](#6216-internal-由工具鏈的套件共用)） |
| `#[runtime_attribute(name)]`   | `struct`                      | 宣告執行時屬性 `#[name(..)]`，保留在 `Inspect` 描述中（[6.10.8](610-reflection.md#6108-執行時屬性)） |
| `#[inspect(skip)]`             | 欄位                          | 讓該欄位不出現在其型別的 `Inspect` 描述中（[6.10.7](610-reflection.md#6107-typeinfo型別的描述)） |
| `#[renamed_from("old")]`       | 欄位                          | 在其描述中記錄該欄位先前的名稱（[6.10.8](610-reflection.md#6108-執行時屬性)） |
| `#[available(macos = "13.0")]` | 函式、方法、`extern "objc"` 區塊的方法 | 該 API 需要的 macOS 版本：除非部署目標或外圍的 `if objc::available(macos: "13.0")` 涵蓋它，否則呼叫是錯誤（[stdlib 1.10.13](../../refs-standard-libraries/100-bare/110-objc.md#11013-可用性與部署目標)） |
| `#[objc::class(..)]`、`#[objc::protocol(..)]`、`#[objc::category(..)]`、`#[selector("..")]`、`#[returns_retained]`、`#[returns_not_retained]` | `extern "objc"` 區塊及其方法 | Objective-C 類別及其方法（[stdlib 1.10.2](../../refs-standard-libraries/100-bare/110-objc.md#1102-extern-objc-區塊)） |

## 6.2.14. `#[deprecated_safe]` 現在安全，之後 `unsafe`

```nabi
#[deprecated_safe("`ptr` must point at a NUL-terminated string")]
pub fn from_ptr(ptr: Ptr<u8>) -> CStr { ... }
```

`#[deprecated_safe("note")]` 標示之後版本會改為 `unsafe` 的函式庫函式、方法、extern 函式或 static。在 `unsafe` 區塊或 `unsafe fn` 之外呼叫它或使用該 static，會以 lint `deprecated_safe` 及附註發出警告：把呼叫包在 `unsafe { .. }` 中，現在就能編譯，而在該項目變成 `unsafe` 之後也仍然正確。在 `unsafe` 內不會有警告。

它是給標準函式庫與函式庫套件使用的。用在正在建置之程式的函式上是錯誤：請改把該函式設為 `unsafe fn`。用在函式、方法、extern 函式或 static 以外的項目上是錯誤。

## 6.2.15. `#[sealed]` 只在定義套件中實作

```nabi
/// The element types `Atomic<T>` supports.
#[sealed]
pub trait AtomicPrimitive {}
```

trait 上的 `#[sealed]` 表示只有定義該 trait 的套件能實作它。在其他任何套件中對該 trait 的 `impl` 是錯誤，錯誤訊息會指出該 trait 是 sealed 的以及擁有它的套件。其他一切都與任何 trait 相同：其他套件可以在約束中使用它（`T: AtomicPrimitive`）、呼叫其方法，並在 `use` 中指名它。

當函式庫依賴於知道實作它的每一個型別時，就把 trait 設為 sealed：出於健全性（原子運算只為那些型別存在），或因為其他 impl 會被忽略。標準函式庫把 `AtomicPrimitive` 設為 sealed（[stdlib 2.16](../../refs-standard-libraries/200-core/216-sync.md)）。

工具鏈的每個層級都是自己的套件（[5.6.4](../005-modules/506-visibility.md#564-pubpkg在套件內)）：`core` 設為 sealed 的 trait，其 impl 在 `core` 中。`#[sealed]` 不接受引數；用在 trait 以外的任何東西上都是錯誤。

## 6.2.16. `#[internal]` 由工具鏈的套件共用

```nabi
/// Wake every task waiting on `fd`'s readiness.
#[internal]
pub fn wake_io(fd: i32) { .. }
```

`#[internal]` 標示一個 `pub` 項目，它由工具鏈自己的套件 `bare`、`core` 與 `std` 彼此共用，但不屬於它們的 API。它可以附加在函式、方法、`const`、`static` 或欄位上。

- 從其他任何套件的使用（呼叫、函式值、讀取）是錯誤：「internal to the Nabi toolchain」。該項目仍會被解析，因此錯誤訊息會指出它。
- `#[internal("..")]` 指出應改寫成什麼，錯誤訊息會顯示它（`Int::from_literal` 指出 `s.parse::<Int>()`）。沒有附註時，錯誤訊息會說明該項目只供編譯器使用。
- 它不會出現在 API 文件（`nabi doc`）、語言伺服器的自動完成與懸停清單，以及「did you mean」建議中，與 `#[doc(hidden)]` 項目相同。
- 它不是新的可見性等級（[5.6.1](../005-modules/506-visibility.md#561-四個等級)）：該項目是 `pub`，而這項檢查與可見性分開。
- 只有工具鏈的套件可以寫它。用在其他任何套件的項目上是錯誤；要讓項目留在套件內，請把它設為 `pub(pkg)`。

帶有它的項目列於 [stdlib 0.4.4](../../refs-standard-libraries/000-meta/004-lang-slot-table.md)。只在一個層級內使用的項目是 `pub(pkg)`，而編譯器從降階後的程式碼呼叫的函式是透過 lang 槽位存取的，因此兩者都不需要 `#[internal]`。
