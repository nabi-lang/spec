# 5.7. 路徑

以 `::` 語彙單元（[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)）連接的一串識別字，是指定模組樹中某個位置的路徑。本文件把路徑定義為對模組與項目的參照。

[型別路徑](../002-types/213-type-paths.md)涵蓋型別位置的路徑；本文件涵蓋指稱值與模組的路徑。兩者形式相同。

```
path            ::= path_root ("::" path_segment)*
path_root       ::= "pkg" | "sup" | "::" identifier | identifier
path_segment    ::= identifier
                  | identifier "::" "<" type_arg_list ">"      // turbofish
type_arg_list   ::= type_expr ("," type_expr)* ","?
```

上面的文法只定義值與模組位置之路徑的 turbofish（`name::<T>`）。在型別位置寫成 `name<T>` 的泛型由[型別路徑](../002-types/213-type-paths.md)中的 `type_path` 定義。

## 5.7.1. 路徑的起始識別字

路徑的第一個識別字是下列之一：

| 起始語彙單元         | 意義                                                     |
|----------------------|----------------------------------------------------------|
| `pkg`                | 從目前套件的根開始                                       |
| `sup`                | 從父模組開始                                             |
| `std`、`core`、`bare` | 標準函式庫層級，除非某個區域名稱取用了它                 |
| `::`                 | 套件：某個層級或該套件的某個相依套件，越過任何區域名稱   |
| 一般識別字           | 目前可見的名稱（子模組、`use` 的名稱）                   |

`pkg` 與可見性關鍵字 `pub(pkg)` 同源，而 `sup` 是可見性 `pub(sub)` 的**鏡像**方向（sub 表示後代，sup 表示祖先）。兩者都只在後面緊接著 `::` 時才是路徑根；在其他位置它們是一般名稱（[1.5.4 上下文關鍵字](../001-lexical/105-keywords.md)）。層級名稱 `std`、`core` 與 `bare` 是一般識別字（[5.7.4](#574-std-core-bare-the-standard-library-tiers)）。

`self::` 不是路徑的起點。同一個模組的子項與項目可以用一般識別字存取（`child::item`）。

名稱解析不會隱含地搜尋祖先模組（例外：`extern` 函式，[5.9.4](509-extern-block.md#name-space-of-extern-functions)）。一般識別字只在目前模組的項目、子模組，以及以 `use` 匯入的名稱（包括隱含的 `use core::prelude::*`）中查找。父模組或更上層模組的項目必須以 `sup::` 指稱，或以 `use` 匯入。

查找順序如下；第一個相符者勝出。

1. 目前模組的項目與子模組，以及套件自身相依套件的名稱（[5.7.14](#5714-paths-between-packages)；兩者皆是的名稱是錯誤）
2. 標準函式庫層級（`std::fs`），除非目前模組的具名 `use` 取用了該名稱
3. 目前模組的 `use`。具名匯入優先於 glob（`*`、`**`）。
4. 隱含的 `use core::prelude::*`（標準函式庫自身的模組：`use core::**`）
5. 對於在預期列舉型別之位置的單獨名稱，該列舉的變體（[2.4.8](../002-types/204-enum.md#248-bare-variants-from-the-expected-type)）

函式無論是否為泛型，都共用單一的值命名空間。因此無論種類為何，最近的宣告勝出（目前模組中非泛型的 `max` 會隱藏 core 中泛型的 `max`）。

```nabi
// inside pkg::a::b
helper();              // ERROR: no implicit search even if helper is in pkg::a
sup::helper();         // OK
```

## 5.7.2. `pkg::`：從套件根開始

```nabi
pkg::geom::Point
pkg::config::DEFAULT_TIMEOUT
pkg::main_module::run
```

`pkg::` 從套件的根模組（`src/main.nabi` 或 `src/lib.nabi`）開始，在套件中的任何地方都以相同方式撰寫。它是大型程式碼庫中安全的絕對參照的標準形式。

## 5.7.3. `sup::`：從父模組開始

```nabi
sup::sibling_function
sup::parent_const
```

`sup::` 從目前模組的父模組開始，因此適合參照兄弟模組與父模組的項目。要往上多層，請串接為 `sup::sup::...`。

```nabi
sup::sup::ancestor_item
```

慣用風格只在行內 `mod name { }` 內（例如測試模組中的 `use sup::*;`）寫 `sup::`，其他地方一律寫從套件根開始的路徑（`pkg::`）：警告 lint `sup_path`（[6.2](../006-meta-system/602-built-in-attributes.md)）會回報其他的 `sup::` 路徑以及 `pub(sub)`，其附註會指出應寫的 `pkg::` 路徑。

## 5.7.4. `std::`、`core::`、`bare::`：標準函式庫層級

```nabi
core::collections::List
core::iter::Iterator
bare::ptr::Ptr
std::fs::File
```

標準函式庫是三個套件，每個層級一個（[0.5 標準函式庫層級](../000-meta/005-standard-libraries-tiers.md)），而每個層級的名稱就是其路徑的根。

| 路徑           | 意義                                 |
|----------------|--------------------------------------|
| `bare::*` | 原始 / 不安全領域                    |
| `core::*` | 基礎 / 常用的 trait                  |
| `std::*`  | 作業系統整合 / I/O 等                |

這三個套件隨工具鏈一起提供，由編譯器自動建置與連結（[0.5.4](../000-meta/005-standard-libraries-tiers.md#054-distribution-form-and-linking-mechanism)）；它們不是使用者套件的相依套件，也不需要 `--extern`。其他任何套件都不得命名為 `std`、`core` 或 `bare`（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）。

除此之外，層級的名稱是一般名稱。模組、項目或相依套件別名可以叫做 `std`、`core` 或 `bare`；在宣告或匯入它的模組中，該名稱就是區域的那一個，就像任何遮蔽另一個名稱的名稱一樣。開頭的 `::` 一律指稱工具鏈的函式庫（或某個相依套件）：`::std::fs::File` 越過區域的 `std` 到達該層級。

```nabi
mod std { pub fn f() -> i32 { 1 } }

fn main() {
    let a = std::f();                    // the module above
    let p = ::std::path::Path::empty();  // the library's std
}
```

`bare` 與 `std` 不會自動匯入，需要明確匯入。相對地，`core` 的 prelude 透過 [5.5 use](505-use.md) 中定義的隱含 `use core::prelude::*;` 自動可見。

## 5.7.5. 以一般識別字開頭

```nabi
foo::bar::baz                   // OK if foo is a currently visible name
geom::Point                      // if the geom module is a child or imported
Point                            // a single name, resolved automatically
```

若第一個識別字是目前模組中可見的名稱，就可以原樣使用。這裡的可見名稱指子模組或以 `use` 匯入的項目。

## 5.7.6. 各路徑片段的意義

```nabi
pkg::geom::shapes::Circle
└──┘  └──┘  └────┘   └────┘
root  mod   module   item
```

每個片段都是一個模組，或（最後一個）是一個項目。最後一個識別字是最終被指定的項目；前面的片段是路徑上的模組或型別。

## 5.7.7. 依項目種類的路徑

| 指定的東西                                   | 形式                  |
|----------------------------------------------|-----------------------|
| 函式                                         | `path::function_name` |
| 型別（`struct` / `enum` / `union` / `trait`）| `path::TypeName`      |
| `const`                                      | `path::CONST_NAME`    |
| `static`                                     | `path::STATIC_NAME`   |
| 模組本身                                     | `path::module_name`   |
| 靜態方法                                     | `Type::method_name`   |
| 關聯 `const`                                 | `Type::CONST`         |
| 關聯型別                                     | `T::AssocType`        |

這些與[型別路徑](../002-types/213-type-paths.md)的關聯項目存取形式相同。

## 5.7.8. 運算式位置與型別位置

相同的路徑形式可以同時用在運算式位置（指定值）與型別位置（指定型別）。

```nabi
// expression position
let f = pkg::geom::distance;          // function value
let p = pkg::geom::Point::origin();   // static method call
let v = pkg::config::DEFAULT;           // const value

// type position
let p: pkg::geom::Point = ...;
fn make() -> pkg::geom::Point { ... }
```

兩種位置的差別在於泛型引數是否以 turbofish（`::<...>`）撰寫。運算式位置使用 turbofish；型別位置使用一般的 `<...>`。詳見[型別路徑](../002-types/213-type-paths.md)。

## 5.7.9. 結合路徑與 glob

與 [`use`](505-use.md) 的匯入形式結合，可以使用各種匯入記法。

```nabi
use pkg::geom::{Point, Circle, distance};
use sup::{a, b as alias};
use foo::*;
```

`*`（glob）與 `**`（遞迴 glob）的行為依 [`use`（匯入）](505-use.md)的定義。

## 5.7.10. 空路徑 / 開頭的 `::`

空路徑（只有 `::` 而沒有識別字）是編譯錯誤。

開頭的 `::` 從套件開始：其後的識別字是標準函式庫層級或該套件的相依套件（`::std::fs`、`::json::parse`），無論同拼法的區域名稱是什麼。其後是其他任何名稱都是錯誤（`::A`）；本套件的項目要寫成 `pkg::A`。

## 5.7.11. 以 `use` 縮短

以 `use` 匯入的名稱可以在目前模組中以其簡短名稱使用。

```nabi
use pkg::geom::Point;

fn f() {
    let p = Point { ... };          // short name
    let q: Point = ...;
}
```

在模組頂端以 `use` 收集常用的項目，能讓程式碼主體保持簡潔。

## 5.7.12. 路徑何時解析

路徑解析在編譯期的名稱解析階段進行，對執行時的行為沒有影響（不過，動態槽位上的方法分派另外在執行時進行；見 [`any` 槽位](../002-types/202-any-slot.md)）。

## 5.7.13. 動態參照模組（無）

```nabi
let m = "foo";
m::bar();                            // ERROR: no dynamic module references
```

路徑的模組部分一律是靜態的，在編譯期決定。不提供在執行時把模組當作變數接收並呼叫其中函式的功能。

這與[模組](504-modules.md)中「模組樹是靜態結構」的原則一致。

## 5.7.14. 套件之間的路徑

另一個套件（相依套件）的項目，以從該套件名稱開始的路徑參照。

```nabi
use parser::Token;
let t = parser::lex(src);
```

- 相依套件的名稱是頂層路徑。名稱是 `nabi.nabion` 中 `deps` 的鍵，並把 `-` 換成 `_`（`nabi-syntax` → `nabi_syntax::..`），建置工具以 `nabic --extern name=..` 註冊它（[9.2.3](../009-tools/902-aot-pipeline.md#923-dependency-resolution)、[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）。不需要像 Rust 的 `extern crate` 那樣另外宣告。
- 套件只看得到它所相依的套件，名稱就是它自己的 `deps` 給它們的名稱。建置中其他套件的相依套件不在作用域內：以 `"csvio": Dep::..(package: "csv")` 相依於 `csv` 的套件寫的是 `csvio::..`，而在那裡 `csv::..` 是它自己的模組 `csv`，否則是錯誤，其說明會提示把 `csv` 加入該套件的 `deps`。
- 同時是套件相依套件與目前模組子模組的名稱有歧義：以它開頭的路徑是錯誤，錯誤訊息會指出兩者。其說明會指出應寫的形式：相依套件寫 `::name::..`，模組寫 `pkg::..::name::..`，或在 `nabi.nabion` 中為相依套件換一個名稱（以 `package:` 指名該套件的鍵）。
- 相依套件內的 `pkg::` 是該相依套件自己的根。
- 正在建置之套件根部的模組，不得與建置中某個套件同名（這些套件掛載在同一棵模組樹的頂端）：那是錯誤，錯誤訊息會提示為該模組改名。
- 解析到正在建置之套件本身的相依套件是錯誤；套件以 `pkg::` 指稱自己的項目。

### 套件邊界

下列每一項都是獨立的套件。可見性 `pub(pkg)` / `pub(sub)`（[5.6](506-visibility.md)）不會跨越這些邊界。

- 正在建置的套件
- 每個相依套件
- 標準函式庫的每個層級（`bare`、`core`、`std`）

因此相依套件的私有 / `pub(sub)` / `pub(pkg)` 項目不能從其外部使用；只有 `pub` 項目可以。
