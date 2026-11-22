# 6.1. 屬性語法

本節定義屬性的語彙與語法形式及其附加規則。屬性遵循下列文法：

```
attribute       ::= outer_attr | inner_attr
outer_attr      ::= "#" "[" attr_body "]"
inner_attr      ::= "#" "!" "[" attr_body "]"
attr_body       ::= path attr_args?
attr_args       ::= "(" attr_arg ("," attr_arg)* ","? ")"
                  | "=" literal
attr_arg        ::= literal | identifier | attr_body | identifier "=" literal
```

有些內建屬性接受自己文法的引數，見其各自的章節：`#[niche]` 接受負值、範圍以及 `i32::MIN` 之類的常數（[2.7](../002-types/207-niche-optimization.md)），`#[literal]` 接受語彙單元模式（[6.5](605-literal-magic.md)）。

## 6.1.1. 引導符

屬性由[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)中定義的兩個語彙單元引導。

### `#[...]` 外部屬性

外部屬性附加在緊接其後的項目上。例如：

```nabi
#[derive(Debug)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }
```

`#[...]` 附加在其後的項目上。

### `#![...]` 內部屬性

內部屬性附加在包圍它的項目或模組上：

```nabi
#![cfg(target_os = "linux")]      // applies to this whole module / file

mod m {
    #![allow(unused)]              // applies to module m itself
    fn helper() { ... }
}
```

內部屬性只出現在檔案或 `mod { .. }` 主體的開頭。在函式主體中不接受它們；請改在函式上放外部屬性。外部 / 內部的區分遵循與[語彙結構](../001-lexical/102-lexical-structure.md)中 `///`（外部文件註解）與 `//!`（內部文件註解）相同的模式。

## 6.1.2. 主體形式

屬性主體有三種形式：`#[name]`、`#[name(args)]` 與 `#[name = value]`。

### 單獨的名稱

最簡單的形式只使用名稱：

```nabi
#[test]
#[inline]
#[cold]
```

單獨名稱的形式用作布林旗標或標記。

### 帶引數

帶引數的形式也很常見：

```nabi
#[derive(Debug, Clone, PartialEq)]
#[repr(C)]
#[link("m")]
#[inline(always)]
#[cfg(target_os = "linux")]
#[infix("+")]
```

這是 `name(args)` 形式，引數以逗號分隔。每個引數可以是識別字、字面值、鍵值（`like = "+"`），或巢狀的屬性形式（`derive(...)` 內的 `Debug`）。

### 鍵值

最後是鍵值形式：

```nabi
#[doc = "This function ..."]
#[link_name = "external_symbol"]
```

這是 `name = value` 形式，其中的值是字面值。它看起來與引數形式中的鍵值記法相同，只是整個主體是單一的鍵值。

## 6.1.3. 可以帶屬性的項目

屬性可以附加在下列東西上：

| 項目                                | 範例                                    |
|-------------------------------------|-----------------------------------------|
| 函式 / 方法                         | `#[inline] fn f() { ... }`              |
| `struct` / `enum` / `union`         | `#[derive(PartialEq, Eq)] struct S { ... }`（在 `union` 上 derive 是錯誤） |
| `trait` / `impl`                    | `#[deprecated] trait T { ... }`         |
| `const` / `static`                  | `#[deprecated] const X: i32 = 0;`       |
| 模組                                | `#[cfg(test)] mod tests { ... }`        |
| 項目內的欄位 / 變體（包括元組變體的欄位） | `struct S { #[deprecated] x: i32; }`、`enum E { V(#[cfg(x)] i32) }` |
| trait 方法 / 關聯項目               | `#[infix("+")] fn add(...);`            |
| `use` / `extern`                    | `#[link("m")] extern "C" { ... }`       |
| 陳述式（僅 `#[cfg]`）               | `#[cfg(test)] my_test_call();`          |
| match 分支（僅 `#[cfg]`）           | `#[cfg(debug_assertions)] Op::Trace => trace(),` |
| 結構字面值欄位（僅 `#[cfg]`）       | `S { #[cfg(x)] extra: 1, .. }`        |
| 函式參數（僅 `#[cfg]`）             | `fn f(#[cfg(x)] extra: i32) { ... }`    |

哪些屬性在各位置有意義，取決於各屬性的定義；例如 `#[derive]` 只適用於型別，`#[inline]` 只適用於函式。

## 6.1.4. 多重附加

同一個項目可以附加多個屬性。

```nabi
#[derive(Debug, Clone)]
#[repr(C)]
#[deprecated]
struct OldPoint { x: f64; y: f64; }
```

每個 `#[...]` 可以各占一行，或為了可讀性把多個放在同一行。

```nabi
#[inline] #[cold] fn rarely_called() { ... }
```

## 6.1.5. 屬性何時生效

屬性何時生效因屬性而異。

- 編譯器辨識的屬性（例如 `#[inline]`、`#[derive]`）由編譯器在編譯期直接處理。
- 魔法屬性（例如 `#[lang]`、`#[literal]`、`#[infix]`）由編譯器結合語言意義處理。
- 使用者定義的屬性（來自元區塊）在樣板展開階段產生程式碼。

所有這些處理都在剖析之後的 meta 展開階段完成。詳細流程定義於[樣板記號與展開](609-template-tokens.md)。

## 6.1.6. 未知的屬性

編譯器可能遇到它不認得的屬性：

```nabi
#[mystery_attribute]
fn f() { ... }
```

既不是內建、也沒有被任何元區塊定義或測試（包括 `@field.has_attr(x)`）的屬性是編譯錯誤，因此打錯字絕不會被默默忽略。內建屬性會檢查其位置與引數形狀。

## 6.1.7. 屬性與衛生

當附加在項目上的屬性導致程式碼產生時，產生的程式碼中的識別字遵循 [6. 元系統](README.md)的衛生規則，因此不會與使用者程式碼的命名空間衝突。詳細的衛生規則定義於[樣板記號與展開](609-template-tokens.md)。

## 6.1.8. 同一語彙單元的多重對應

同一個方法可以附加多個屬性：

```nabi
#[infix("⊕", like = "+")]
#[infix("⊞", like = "+")]
fn join(self, rhs: Self) -> Self;
```

此時兩個語彙單元（`⊕`、`⊞`）都分派到同一個方法。`≤` 之類的固定 Unicode 拼法不需要別名：語彙分析器把它們讀作其 ASCII 語彙單元，而註冊其中之一（`#[infix("≤")]`）是編譯錯誤（[6.7.4](607-operator-magic.md#674-unicode-與多重別名)）。

## 6.1.9. 屬性與可見性及其他關鍵字的互動

屬性可以與可見性關鍵字（`pub`、`pub(pkg)` 等）及函式限定詞（`async`、`yield` 等）以任何順序組合。

```nabi
#[inline]
pub fn f() { ... }

pub
#[inline]
fn g() { ... }                     // same meaning

pub async fn h() { ... }
#[inline] pub async fn h() { ... }
pub #[inline] async fn h() { ... }
```

慣用的順序是 屬性 → 可見性 → 限定詞（`#[inline] pub async fn ...`），但規格不強制順序。
