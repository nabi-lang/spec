# 5. 變數 / 模組 / 可見性 / 套件概觀

這一部分涵蓋 Nabi 的命名空間。其目標是定義變數在哪裡可見、模組如何分組、可見性等級如何運作，以及套件之間的邊界在哪裡。

## 目錄

- [5.1. 變數宣告](501-variable-declaration.md)
- [5.2. `const`](502-const.md)
- [5.3. `static`](503-static.md)
- [5.4. 模組](504-modules.md)
- [5.5. `use`（匯入）](505-use.md)
- [5.6. 可見性](506-visibility.md)
- [5.7. 路徑](507-paths.md)
- [5.8. 重新匯出（`pub use`）](508-re-export.md)
- [5.9. `extern` 區塊（FFI 宣告）](509-extern-block.md)

## 命名空間的四個層級

命名空間有四個層級：

```
package
  └── module
        └── item
              └── local variable
```

包含關係由上而下。一個套件有多個模組，一個模組有多個項目（函式、型別、trait、`const`、`static`），而一個項目（特別是函式主體）有多個區域變數。

各層級的邊界規則定義於下列文件。

## 套件：最大的單位

[原始碼單元](../000-meta/003-source-units.md)中定義的套件是命名空間的最外層。每個套件以基於內容的雜湊識別。

同一個套件中的所有項目屬於同一個程式碼權限範圍。這就是 [`impl`](../002-types/209-impl.md) 的孤兒規則以套件為單位適用的原因。

## 模組：套件內的分組單位

一個套件有多個模組。每個模組要麼是單一檔案（`foo.nabi`），要麼是主體檔案加上同名資料夾（`foo.nabi` + `foo/`）。這與 Rust 2018+ 的模型相同。詳細規則定義於[模組](504-modules.md)。

```
my_pkg/
└── src/
    ├── lib.nabi                  <- root module body
    ├── geom.nabi                 <- geom module    (single-file: geom.nabi)
    ├── parsing.nabi              <- parsing module body
    └── parsing/                 <- parsing's submodules
        ├── lexer.nabi                parsing::lexer
        └── parser.nabi               parsing::parser
```

## 項目：模組內的具名定義

每個模組都有項目：函式、型別、trait、`impl` 區塊、`const`、`static`，以及其他模組宣告。

[函式宣告](../004-functions/401-function-declaration.md)、[`struct`](../002-types/203-struct.md)、[`enum`](../002-types/204-enum.md)、[`trait`](../002-types/208-trait.md) 與 [`impl`](../002-types/209-impl.md) 的每個定義都是模組內的項目。

## 區域變數：函式 / 區塊內的變數

在函式、區塊與其他運算式內，以 `let` 關鍵字（[關鍵字](../001-lexical/105-keywords.md)）宣告區域變數。

```nabi
fn area(r: f64) -> f64 {
    let pi = 3.14159f64;          // local variable
    pi * r * r
}
```

區域變數的可見範圍依[區塊運算式](../003-expressions/309-block-expressions.md)的區塊決定。詳細規則定義於[變數宣告](501-variable-declaration.md)。

## 可見性：公開項目

每個項目都有一個可見性。可見性等級（[可見性](506-visibility.md)）決定該項目在多大範圍內可見。

```
private (default)    within the same module only
pub(sub)             within submodules only
pub(pkg)             within the same package only
pub                  also from external packages
```

`#[internal] pub` 項目是工具鏈自己的套件之間共用的 `pub` 項目；其他套件不能使用它們（[5.6](506-visibility.md)、[6.2.16](../006-meta-system/602-built-in-attributes.md#6216-internal-shared-by-the-toolchains-packages)）。

## 名稱解析規則

決定程式碼中的名稱指向哪個項目，稱為名稱解析。

它考慮下列事項。第一，目前模組中區域可見的項目是候選。第二，以 `use`（[`use`](505-use.md)）匯入的外部項目也是候選。第三，也會查閱自動匯入的 prelude 與標準函式庫層級（[標準函式庫層級](../000-meta/005-standard-libraries-tiers.md)）。最後進行可見性檢查，只有可見的項目會留作候選。

名稱解析在編譯期進行。動態槽位上的方法分派在執行時進行，但名稱本身的解析一律在編譯期完成。
