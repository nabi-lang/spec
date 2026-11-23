# 5.4. 模組

子模組以 `mod` 關鍵字（[關鍵字](../001-lexical/105-keywords.md)）宣告。本文件定義模組宣告與檔案系統配置之間的對應。

```
mod_decl        ::= "mod" identifier ";"
                  | "mod" identifier "{" item* "}"      // inline module
```

## 5.4.1. 模組宣告

```nabi
mod foo;
mod parsing;
```

模組宣告由 `mod` 關鍵字、模組名稱與 `;` 組成。模組名稱遵循[識別字](../001-lexical/104-identifiers.md)的形式，光靠宣告就會自動對應到同名的檔案或資料夾。

## 5.4.2. 對應形式

出現 `mod foo;` 時，編譯器在與父模組主體相同的目錄中尋找下列兩種形式之一。這與 Rust 2018+ 的規則相同。

### 形式 1：單一檔案（`foo.nabi`）

沒有子模組的模組只需要一個檔案。

```
parent/
├── parent.nabi                <- where `mod foo;` appears
└── foo.nabi                   <- foo module body, no submodules
```

`foo.nabi` 的內容就是模組 foo 的主體。

### 形式 2：資料夾（`foo.nabi` + `foo/`）

有子模組的模組必須同時有主體檔案（`foo.nabi`）與同名資料夾（`foo/`）。主體檔案作為模組的主體，資料夾存放其子模組。

```
parent/
├── parent.nabi                <- where `mod foo;` appears
├── foo.nabi                   <- foo module body
└── foo/                      <- foo's submodules
    ├── bar.nabi                   declared as `mod bar;` inside foo.nabi
    └── baz.nabi                   declared as `mod baz;` inside foo.nabi
```

這裡的 `foo.nabi` 是模組 foo 的主體，包含一般項目、`mod bar;` 之類的子模組宣告、可見性等。`foo/` 目錄中的每個 `.nabi` 檔案都是 foo 的子模組，必須在主體中以 `mod name;` 明確宣告。若只有 `foo/` 而沒有 `foo.nabi`，就是錯誤。主體檔案是必要的。

這個模型與 Rust 2018+ 的 `foo.rs` + `foo/` 結構完全相同；沒有 `mod.nabi` 這種另外的主體檔名。

### 同名檔案與資料夾的意義

當同名的 `foo.nabi` 與 `foo/` 一起位於父目錄中時，它們會被自動辨識為資料夾形式；這不是衝突。

```
parent/
├── parent.nabi
├── foo.nabi                   <- foo body
└── foo/                      <- foo's submodules
    └── ...
```

總之：只有 `foo.nabi` 是單一檔案模組；`foo.nabi` 加上 `foo/` 是資料夾模組；只有 `foo/` 是錯誤。

## 5.4.3. 子模組

只有資料夾形式的模組（`foo.nabi` + `foo/`）可以有子模組。在主體（`foo.nabi`）中寫 `mod name;`，會把 `foo/name.nabi` 或 `foo/name.nabi` + `foo/name/` 對應為該子模組。

```nabi
// contents of foo.nabi
mod bar;
mod baz;

pub fn foo_helper() { ... }
```

該主體旁邊必須存在下列檔案：

```
foo.nabi                       <- module body (the code above)
foo/
├── bar.nabi                   <- foo::bar submodule
└── baz.nabi                   <- foo::baz submodule
```

子模組本身也可以採用單一檔案或資料夾形式。

```
foo.nabi
foo/
├── bar.nabi                   <- foo::bar (single-file)
├── baz.nabi                   <- foo::baz body
└── baz/                      <- foo::baz submodules
    ├── x.nabi                     foo::baz::x
    └── y.nabi                     foo::baz::y
```

這種模式可遞迴地適用於任意深度，每個模組各自獨立地選擇單一檔案或資料夾形式。

### 單一檔案模組沒有**對應到檔案的**子模組

```nabi
// foo.nabi (single-file module, no foo/)
mod bar;            // ERROR: there is no foo/, so nowhere to map
```

若需要對應到檔案的子模組，請建立同名資料夾（`foo/`），並把子模組檔案放進去。把單一檔案模組升級為資料夾模組，只需要建立資料夾並移動子模組檔案。

另一方面，行內子模組（`mod bar { ... }`）不需要檔案對應，因此即使在單一檔案模組中也可以自由使用（[5.4.5 行內模組](#545-行內模組在主體內定義)）。

## 5.4.4. 套件根

[原始碼單元](../000-meta/003-source-units.md)中定義的 `src/main.nabi` 或 `src/lib.nabi` 是套件根模組的主體檔案，而套件的原始碼目錄 `src/` 作為根模組的資料夾。

根模組不需要 `mod` 宣告就會被自動辨識，其他每個模組都是從根開始之樹中的一個節點。

```
my_pkg/
├── nabi.nabion
└── src/
    ├── lib.nabi                   <- root module body (declares `mod geom; mod parsing;`)
    ├── geom.nabi                  <- geom module    (single-file: geom.nabi)
    ├── parsing.nabi               <- parsing module body
    └── parsing/                  <- parsing's submodules (declared inside parsing.nabi)
        ├── lexer.nabi                 parsing::lexer
        └── parser.nabi                parsing::parser
```

根模組一律是資料夾形式：`src/` 是其資料夾，`lib.nabi`（函式庫）或 `main.nabi`（執行檔）是其主體檔案。根中的 `mod foo;` 對應到 `src/foo.nabi`；位於 `src/` 旁邊的 `foo.nabi` 絕不是模組。`#[path(..)]`（5.4.6）從子模組檔案通常所在的資料夾解析：根中的 `mod` 是 `src/`，`parsing.nabi` 內的 `mod` 是 `src/parsing/`。

## 5.4.5. 行內模組：在主體內定義

在 `mod` 宣告之後直接放上區塊主體，就能不用另外的檔案定義一個行內模組。

```nabi
mod inner {
    pub fn helper() -> i32 { 42i32 }
    const X: i32 = 100i32;
}

// use
let r = inner::helper();
```

行內模組寫成 `mod name { ... }`，其主體可以包含任何項目：函式、型別、trait、`const`、`static`，以及更多的 `mod`。旁邊不得存在同名的檔案或資料夾；衝突是錯誤。

## 5.4.6. 自訂路徑：`#[path("...")]`

```
path_attr       ::= "#" "[" "path" "(" string_lit ")" "]"
```

`#[path("...")]` 屬性覆寫預設的對應。

```nabi
#[path("custom_geom.nabi")]
mod geom;

#[path("../shared/shared.nabi")]
mod shared;
```

給定路徑上的檔案被對應為模組主體。路徑相對於通常會找到 `name.nabi` 的目錄。該模組的子模組放在該檔案旁邊、以其主檔名命名的資料夾中：對 `#[path("f.nabi")]` 而言是 `f/`。目標檔案不得命名為 `mod.nabi`，因為 Nabi 沒有 `mod.nabi` 主體形式（[5.4.4](#544-套件根)）；那是錯誤。`#[path(..)]` 是唯一的形式。它在一般使用中很少出現，用於建置系統與產生的程式碼等特殊情況。

## 5.4.7. 模組內的項目

每個模組都可以包含[概觀](README.md)中介紹的項目。

```nabi
mod geom {
    struct Point { ... }
    fn distance(a: Point, b: Point) -> f64 { ... }
    const PI: f64 = 3.14159f64;
    static CACHE: Mutex<Dict<str, f64>> = ...;
    impl Point { ... }
    trait Shape { ... }
    mod inner { ... }            // nested module again
}
```

模組只能包含項目；可執行的陳述式（頂層的 `let`、運算式陳述式）只允許出現在進入模組 `main.nabi` 中（[4.16.6](../004-functions/416-entry-point.md#4166-模組只含定義)）。所有這些項目都是該模組的一部分，從外部參照它們需要[可見性](506-visibility.md)關鍵字與[路徑](507-paths.md)兩者。在函式主體中宣告的項目（[3.9.1](../003-expressions/309-block-expressions.md#391-形式)）就一致性而言屬於該模組，但只能在其區塊內指稱。

## 5.4.8. 模組的可見性

模組本身可以被賦予可見性。

```nabi
pub mod public_api;            // usable from external packages
pub(pkg) mod internal;          // within the same package only

mod _private;                   // module-private: only from the same parent module
```

詳細意義定義於[可見性](506-visibility.md)。

## 5.4.9. 在模組內參照父 / 兄弟 / 子模組

參照其他模組之項目的路徑記法定義於[路徑](507-paths.md)。

```nabi
pkg::geom::Point             // from the package root
sup::sibling_mod::item        // through the parent module to a sibling
child_mod::item                // within the same module (an identifier suffices)
```

## 5.4.10. 模組宣告可以出現的位置

`mod` 宣告只能出現在模組層級，也就是另一個 `mod` 的主體中或套件根。在函式主體內宣告 `mod` 是錯誤（[3.9.1](../003-expressions/309-block-expressions.md#391-形式) 列出了區塊可以宣告的項目）。

```nabi
fn f() {
    mod inner { ... }          // ERROR: no module declarations inside a function
}
```

這來自「模組樹是在編譯期決定的靜態結構」的原則。

## 5.4.11. 把一個模組分散到多個檔案

要把主體分散到多個檔案，請使用資料夾形式，並在主體（`foo.nabi`）中透過子模組重新匯出項目。

```nabi
// foo.nabi
mod core;
mod ext;

pub use core::*;
pub use ext::*;
```

```
foo.nabi                       <- module body (re-exports only)
foo/
├── core.nabi                  <- core items
└── ext.nabi                   <- extension items
```

依這種模式，使用者以 `foo::item` 存取每個項目，主體檔案保持小巧，實際的程式碼則分散在各子模組中。這與 Rust 的 `pub use` 重新匯出模式相同。

## 5.4.12. 同一父模組中不得有重複的模組

在同一個父模組中兩次宣告同名的模組是錯誤。

```nabi
mod foo;
mod foo;                        // ERROR: duplicate
```
