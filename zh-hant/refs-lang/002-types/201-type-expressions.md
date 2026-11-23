# 2.1. 型別運算式

型別運算式是原始碼中表示型別的記法。型別運算式出現在許多位置：變數宣告的 `:` 之後、函式參數的 `:` 之後、回傳型別的 `->` 之後、作為泛型引數、轉換中的 `as` 之後等。

本文件只定義形式；哪個名稱表示哪個型別由標準函式庫決定。

## 2.1.1. 基本形式

型別運算式的基本文法如下：

```
type_expr   ::= sum_type
sum_type    ::= unary_type ("|" unary_type)*
unary_type  ::= base_type "?"*                  // T?: optional (2.1.5a), `??` is two `?`
base_type   ::= path_type
              | fn_type
              | tuple_type
              | array_type
              | any_type
              | self_type
              | qualified_type
              | macro_type
              | "none"                          // the none type (2.6)
              | "!"                             // never (4.3.7)
              | "(" type_expr ")"               // grouping
self_type   ::= "Self" ("::" identifier ("::" identifier)* type_args?)?   // Self, Self::Item
qualified_type ::= "<" type_expr "as" trait_ref ">" "::" identifier       // 2.13.4
macro_type  ::= identifier "!" ( "(" token* ")" | "[" token* "]" | "{" token* "}" )   // 6.8.12
any_type    ::= "any" ( trait_ref | "(" trait_ref ("+" trait_ref)* ")" )?
trait_ref   ::= identifier ("::" identifier)* trait_args?
trait_args  ::= "<" trait_arg ("," trait_arg)* ","? ">"
trait_arg   ::= type_expr | identifier "=" type_expr | const_arg
path_type   ::= identifier ("::" identifier)* type_args?
type_args   ::= "<" trait_arg ("," trait_arg)* ","? ">"
const_arg   ::= literal                         // int, char, true / false
              | "-" literal
              | "{" expr "}"                    // 2.11.4
fn_type     ::= "unsafe"? ("extern" string_lit)? "fn" "( (type_expr ("," type_expr)*)? ")" ("throws" ("(" type_expr ")")?)? ("->" unary_type)? fn_bound*
fn_bound    ::= "+" ("Send" | "Sync")
tuple_type  ::= "(" ")"
              | "(" type_expr "," ")"
              | "(" type_expr ("," type_expr)+ ","? ")"
array_type  ::= "[" type_expr ";" const_expr "]"   // const_expr: 5.2
```

`extern "C" fn(A, B) -> R` 是 C 函式指標型別（[5.9.4](../005-modules/509-extern-block.md#c-函式指標型別)）。ABI 字串只能是 `"C"` 或 `"system"`（[5.9](../005-modules/509-extern-block.md)）；其他任何字串都是錯誤。

各形式整理如下。

| 形式              | 範例                                        |
|-------------------|---------------------------------------------|
| 簡單名稱          | `i32`、`bool`、`Point`                      |
| 路徑              | `core::time::Duration`                 |
| 泛型套用          | `List<i32>`、`Dict<str, f64>`、`Pair<A, B>` |
| 和型別（`\|`）    | `i32 \| f32`、`T \| none`                     |
| 可選（`?`）       | `i32?`、`(i32 \| str)?`、`List<T>?`           |
| 函式型別          | `fn(i32, i32) -> i32`、`fn() -> T + Send`   |
| 元組型別          | `(i32, str)`、`(i32,)`                      |
| 陣列型別          | `[T; N]`（固定長度）                        |
| 動態槽位          | `any`、`any Error`、`any (Display + Hash)`  |
| `none` / never    | `none`、`!`                                 |
| `Self` 及其項目   | `Self`、`Self::Item`                        |
| 限定路徑          | `<T as Iterator>::Item`                     |
| 巨集呼叫          | `name!(..)`，展開為型別（[6.8.12](../006-meta-system/608-meta-blocks.md#6812-巨集形式)） |
| 常數引數          | `Buf<16>`、`Buf<-1>`、`Buf<{ N + 1 }>`      |

各形式的確切意義依序定義於下文。

`T::Assoc` 形式的關聯型別投影（`I::Item`、`F::Output`）也是型別。它是路徑形式（`path_type`）之一，當 `T` 是受到擁有該關聯型別之 trait 約束的型別參數時有效（[型別路徑](213-type-paths.md)）。

```nabi
fn zip<I: Iterator, J: Iterator>(a: I, b: J) -> List<(I::Item, J::Item)> { ... }
```

## 2.1.2. 簡單名稱

簡單名稱由單一識別字組成，遵循[識別字](../001-lexical/104-identifiers.md)的形式。

```nabi
i32
bool
Point
변수타입
```

名稱指向哪個項目在名稱解析時決定。候選包括目前模組中定義的項目以及以 `use` 匯入的項目。

## 2.1.3. 路徑

以 `::` 連接的多個識別字也可以用作型別運算式。

```nabi
core::time::Duration
core::collections::List
pkg::geom::Point
```

這裡的 `::` 語彙單元是[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)中定義的路徑分隔符。路徑的最後一個語彙單元是型別名稱；前面的部分是說明到哪裡找該型別的模組或項目路徑。

路徑的意義與解析規則定義於[型別路徑](213-type-paths.md)；本文件只說明路徑可以作為型別運算式的一種形式出現。

## 2.1.4. 泛型套用

在型別名稱之後附加 `<...>` 可以給出型別引數。

```nabi
List<i32>
Dict<str, f64>
Pair<A, B>
Result<T, E>
Matrix<f64, 3, 4>      // const generic (value arguments, 2.11.1)
```

規則如下：引數以逗號分隔。引數可以是另一個型別運算式，對 `const` 泛型而言也允許 `const` 運算式。不允許空的 `<>`；沒有引數時，根本不寫 `<>`。允許尾隨逗號（`<A, B,>`）。

值引數的確切意義（例如 `Matrix<f64, 3, 4>`）定義於 [`const` 泛型](211-const-generics.md)。

### 運算式位置的 turbofish

在運算式內給出的泛型引數使用 `::<...>` 形式，定義於運算式規格。本文件只涵蓋型別運算式位置的 `<...>`。

## 2.1.5. 和型別（`|`）

值為多種型別之一的和型別以 `|` 撰寫。

```nabi
i32 | f32
T | none
A | B | C | D
```

以 `|` 分隔的每一項都是一個獨立的型別運算式，產生的和型別容納其中某一項的值。其意義與記憶體表示定義於[和型別](206-sum-types.md)。

`|` 語彙單元的結合比其他型別組合子鬆。因此 `i32 | List<f32>` 分組為 `i32 | (List<f32>)`，因為泛型套用的結合比和型別緊。

## 2.1.5a. 可選型別（`T?`）

後綴 `?` 構成可選型別。`T?` 的行為與 `Maybe<T>`（`type Maybe<T> = T | none;`）完全相同。也就是說，`?` 的運算元是一個泛型項，並適用[和型別的分層規則](206-sum-types.md#264-集合語意)。

```nabi
i32?                // i32 | none
(i32 | str)?        // i32 | str | none (no overlap, flattened)
(i32 | none)?       // two layers: outer none ≠ inner none
i32??               // two layers: same as (i32 | none)?
i32 | none          // written sum: always flat i32 | none
```

- 後綴 `?` 的結合比 `|` 緊，也比函式型別的 `->`（2.1.10）緊。`i32 | str?` 是 `i32 | (str?)`。
- `fn() -> i32?` 是回傳可選值的函式；`(fn() -> i32)?` 是可選的函式。
- 在型別位置，`??` 語彙單元是兩個 `?`（`i32??`）。
- 在運算式 `x as T?` / `x is T?` 中，結尾的 `?` 屬於型別。要對結果套用 `?` 運算子（[3.14](../003-expressions/314-error-control-flow.md)），請加括號：`(x as T)?`。
- 不過，`as` / `is` 目標之後的 `??` 絕不是型別的一部分：型別在它之前結束，而 `??` 是空值合併運算子。要在那裡指定兩層的可選型別，請加括號：`x is (i32??)`。

## 2.1.6. 函式型別

表示函式本身的型別寫法如下：

```nabi
fn(i32, i32) -> i32
fn() -> str
fn(T) -> T
fn(i32)                  // return type may be omitted (unit type)
```

函式型別以 `fn` 語彙單元開頭；括號內只放參數型別，不放名稱。回傳型別接在 `->` 之後；省略它表示沒有回傳值（確切意義定義於函式規格）。

## 2.1.7. 元組型別

把多個型別組在一起的元組型別寫法如下：

```nabi
(i32, str)
(f64, f64, f64)
(i32,)                  // a one-element tuple requires a trailing comma
()                      // empty tuple (unit type)
```

規則：有兩個以上的元素時，以逗號分隔。單一元素的元組必須有尾隨逗號（`(T,)`）；沒有的話，它會被剖析為 `T` 的分組。沒有元素的元組（`()`）表示 unit 型別。

## 2.1.8. 陣列型別

固定長度的陣列型別以方括號撰寫。

```nabi
[T; N]
[i32; 16]
[f64; 1024]
[u8; SIZE]
```

陣列型別在方括號內依序寫出元素型別、分號與長度。長度是 `const` 運算式，必須在編譯期確定。允許長度為 0（`[T; 0]`，空陣列型別）。

## 2.1.9. 動態槽位

動態型別槽位以 `any` 關鍵字撰寫。

```nabi
any
any Error
any (Display + Hash)
any Iterator<Item = i32>
any Future<Output = User>
```

`any` 是[關鍵字](../001-lexical/105-keywords.md)中保留的型別名稱槽位關鍵字；其意義定義於 [`any` 槽位](202-any-slot.md)。在 `any` 之後接一個 trait 路徑，或接以 `+` 連接、加上括號的多個 trait 路徑，就構成只容納實作了這些 trait 之值的受約束動態槽位（[2.2.10](202-any-slot.md#2210-受約束的-anyany-trait)）。在 trait 路徑的 `<...>` 內，除了型別引數之外，也可以出現固定關聯型別的繫結，例如 `Item = T`（`trait_arg`）。兩個以上的 trait 路徑需要括號；`any` 只繫結緊接在它之後的單一 trait，因此 `any Error | none` 是 `(any Error) | none`。trait 路徑只能在 `any` 之後出現在型別位置（[trait 不能作為型別](212-trait-as-type-banned.md)）。

## 2.1.10. 優先順序 / 結合

型別運算式中混用組合子時，它們的優先順序（緊 ↔ 鬆）如下：

```
strong  ─→  weak

[T; N]   (...)   <...>   ::   .  →  T?  →  fn(...) -> ...   →   |
```

也就是說，路徑（`::`）、泛型套用（`<...>`）、陣列（`[...]`）與分組（`(...)`）結合最緊。其次是後綴可選（`?`），再來是函式型別（`fn(...) -> ...`），和型別（`|`）結合最鬆。

結果是，`i32 | List<f32>` 分組為 `i32 | (List<f32>)`，`fn(A | B) -> C` 分組為 `fn((A|B)) -> C`。

需要明確分組之處，請使用括號。

```nabi
fn(i32) -> (i32 | str)         // return type is a sum type
(fn(i32) -> i32) | (fn(f64) -> f64)   // sum type with two function types as terms
```
