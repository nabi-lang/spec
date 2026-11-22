# 6.6. `#[keyword(...)]` 魔法

`#[keyword("name")]` 屬性是標準函式庫註冊新關鍵字的機制。

## 6.6.1. 動機

[關鍵字](../001-lexical/105-keywords.md)中的保留關鍵字清單是由語言本身定義的固定集合。每次要再加入一個關鍵字時，都必須修改語言規格嗎？

標準函式庫有時想引入看起來像關鍵字的語彙單元，例如函式庫 DSL 的領域關鍵字。`#[keyword]` 魔法是函式庫在這種情況下能註冊語言層級關鍵字的正式管道。

## 6.6.2. 形式

```
keyword_attr    ::= "#" "[" "keyword" "(" string_lit ")" "]"
```

```nabi
#[keyword("yield_priority")]
fn yield_priority_handler(...) { ... }

#[keyword("with")]
fn with_handler(...) { ... }
```

基本形式是 `#[keyword("name")]`，以關鍵字名稱（識別字形狀的字串字面值）作為引數。處理器是一個**自由函式**；它不能附加在方法或 trait 項目上。

`struct` 上的 `#[keyword("name")]` 註冊一個**型別關鍵字**：該名稱在型別位置指稱該結構（`int` → `core::num::bigint::Int`、`float` → `core::num::bigfloat::Float`，[stdlib 2.1.2](../../refs-standard-libraries/200-core/201-numbers.md#212-int---arbitrary-precision-integer)）。只有標準函式庫可以註冊型別關鍵字。附加在既不是函式也不是結構的項目上是錯誤。

## 6.6.3. 註冊的效果

當某處註冊了 `#[keyword("name")]` 時，下列事項生效。

1. 在該編譯單元中，識別字 `name` 被切分為關鍵字。
2. 該關鍵字的使用被去糖為處理器呼叫。`name(args)` 是透過處理器之絕對路徑的呼叫，而其後的區塊成為[尾隨閉包](../004-functions/408-trailing-closure.md)引數。

```nabi
#[keyword("with")]
fn with_handler(m: Mutex<i32>, body: fn()) { ... }

with(m) { critical(); }
// ⇒ pkg::with_handler(m) { critical(); }     (handler + trailing closure)
```

在註冊之前，`name` 可以作為一般識別字使用；註冊之後它是關鍵字語彙單元。因此把 `name` 用作宣告或成員的名稱是編譯錯誤。這些位置是 `let name`、`fn name`、`struct name`、`enum name`、`trait name`、`mod name`、`for name in`、`.name`、`::name`、`name:`（欄位 / 引數名稱）以及 `name::`。同樣的規則也適用於 f-string 的 `{..}` 內。原始識別字 `r#name`（[1.4.4](../001-lexical/104-identifiers.md#原始識別字)）一律是識別字，因此能在這些位置保留該名稱（`struct S { r#f16: str; }`、`s.r#f16`）。

## 6.6.4. 衝突檢查

```nabi
#[keyword("if")]                  // ERROR: 'if' is a reserved keyword
fn ... { ... }
```

[關鍵字](../001-lexical/105-keywords.md)中的保留關鍵字、只在固定位置是關鍵字的名稱（`type`、`any`、`in`、`where`、`throws`、`rethrows`、`catch`、`finally`、`const`、`static`、`extern`、`mut`、`union`），以及路徑根（`pkg`、`sup`、`nabi`）都不能被註冊。它們在註冊時就會立即被拒絕。

在一個建置中兩次註冊同一個關鍵字是錯誤，無論是在同一個套件內還是跨不同的函式庫。

```
package_a registers #[keyword("with")]
package_b also registers #[keyword("with")]    → ERROR if both packages are used in one build
```

## 6.6.5. 適用範圍

`#[keyword]` 只在包含該註冊的編譯單元（通常是一個套件）內生效。在其他套件中，同一個名稱作為一般識別字運作。例外是標準函式庫的型別關鍵字（`int`、`float`、`f16`、`bf16`），它們適用於每個套件。

這種區域性減少了函式庫之間的衝突，並讓使用者能有意識地決定哪個套件引入哪個關鍵字。

## 6.6.6. 使用模式

`#[keyword]` 很少使用，主要用於下列情況：

- 領域特定語言（DSL）：當領域詞彙（表格資料、網格、電路等）在識別字位置表現得像關鍵字很自然時。
- 難以用運算子魔法表達的區塊或主體形式的新語法。

在大多數情況下，函式與尾隨閉包的組合就足以建立類似關鍵字的模式（[尾隨閉包](../004-functions/408-trailing-closure.md)）。能用那種形式時，請優先使用它。

## 6.6.7. 與掃描預處理的結合

`#[keyword]` 註冊何時被辨識，由編譯器的掃描預處理階段處理。

1. 在剖析之前，編譯器快速掃描原始檔並收集 `#[keyword(...)]` 註冊。
2. 註冊表被傳給語彙分析器。
3. 在完整的語彙分析期間，已註冊的名稱被切分為關鍵字語彙單元。
4. 剖析器把它們去糖為已註冊的處理器。

由於這個預處理，無論註冊出現在哪裡（哪個檔案、哪一行），編譯單元內的一致性都能維持。使用者不需要在意註冊的順序。

## 6.6.8. 關鍵字處理器的簽章

編譯器不另外檢查處理器的簽章。在關鍵字的使用變成處理器呼叫之後（6.6.3），該呼叫以一般函式呼叫的方式進行型別檢查。因此處理器的參數決定了關鍵字的使用形式（引數、尾隨閉包）。

## 6.6.9. 註冊的關鍵字絕不是上下文關鍵字

[1.5.4](../001-lexical/105-keywords.md#154-上下文關鍵字) 的上下文關鍵字（`type`、`in`、`static` 等）是語言定義的固定集合。註冊的關鍵字絕不是上下文關鍵字。

一旦註冊了 `#[keyword("with")]`，`with` 在編譯單元中的任何地方都被切分為關鍵字。註冊無法讓一個字詞只在某些位置是關鍵字。

這種一致性有助於讓語彙分析器與剖析器保持簡單。

## 6.6.10. 註冊的可見性

帶有 `#[keyword]` 之項目的可見性（[可見性](../005-modules/506-visibility.md)）只適用於存取該項目，與關鍵字註冊的效果是分開的。

```nabi
pub(pkg) fn keyword_handler(...) { ... }

#[keyword("with")]
fn another_handler(...) { ... }     // keyword registration: applies to the whole package
```

處理器能否從外部呼叫由可見性決定，但關鍵字註冊本身影響整個編譯單元。

## 6.6.11. `#[keyword]` 與使用者定義巨集的差異

與其他語言的巨集系統相比：

- Rust proc macro：進行任意的語彙單元轉換；使用者自由地產生程式碼。衛生是另一回事。
- Lisp macro：也能在執行時註冊的動態巨集。
- Nabi `#[keyword]`：編譯期的關鍵字註冊；由處理器函式決定去糖。不可能進行自由的語彙單元轉換。

Nabi 刻意採用受限的巨集系統，在表達力與工具友善性的取捨中偏向工具友善性。

## 6.6.12. 註冊的關鍵字與函式呼叫

```nabi
// keyword form
with(mutex) { critical(); }

// function call form (no registration)
with_lock(mutex) { critical(); }     // function + trailing closure
```

大多數類似 DSL 的模式，都可以透過結合函式與尾隨閉包來達成。請把 `#[keyword]` 註冊限制在那種形式會不自然的情況。

## 6.6.13. 何時求值

關鍵字註冊在掃描預處理階段處理，先於其他 meta 處理（cfg、元區塊展開等）。

由於這個順序，cfg 可以有條件地套用於關鍵字處理器本身，而關鍵字也可以用在元區塊主體內。元區塊或關鍵字處理器的 `#[cfg]` 以定義它之套件的 cfg 求值（其自身的功能特性，[9.10.4c](../009-tools/910-nabi-frontend.md#9104c-feature)）。

## 6.6.14. 謹慎使用

請少用並謹慎使用 `#[keyword]`。每一次註冊都會擴大語言的表面，並讓其他使用者負擔猜測新關鍵字意義的工作。

對於會被普遍採用的語言關鍵字，直接把它加入[關鍵字](../001-lexical/105-keywords.md)中的保留清單（也就是作為語言規格的變更）可能更恰當。
