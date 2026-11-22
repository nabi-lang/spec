# 1.5. 關鍵字

關鍵字是看起來像識別字、但被語言保留的語彙單元。當關鍵字出現在識別字位置時，編譯器產生的是關鍵字語彙單元而非識別字。

本文件只定義語言直接保留的關鍵字清單。各關鍵字的確切意義在規格後面的部分介紹；本文件只說明它們的形式與分類。

## 1.5.1. 保留關鍵字清單

關鍵字分為兩類。原則是下面這一條規則。

> [!IMPORTANT]
> 開始陳述式或運算式、且可能與「識別字 + 其後內容」混淆的字詞是**保留關鍵字**；出現位置固定的字詞是**上下文關鍵字**（1.5.4）。

保留關鍵字無論出現在哪裡都被切分為關鍵字，不能用作識別字。唯一的例外是 1.5.5 的名稱位置。

```
let       fn        if        else      match     while     loop      for
return    break     continue  try       throw
async     await     yield     unsafe    using
struct    enum      trait     impl      pub       mod       use
as        is        ref
true      false     none      self      Self
macro     defer     dyn       move      gen       do
```

> [!NOTE]
> `try?` 與 `try!` 不是另外的關鍵字，而是關鍵字 `try` 與標點 `?` / `!` 組成的語彙單元序列（[標點與運算子語彙單元](110-punctuation-and-operator-tokens.md#dot--question-mark)）。

`self`（小寫）指實例識別字（方法接收者），`Self`（大寫）指目前 `impl` 所針對之型別的名稱。`using` 是引入 RAII 風格資源管理區塊的關鍵字（[`using` 運算式](../003-expressions/317-using-expression.md)）。

## 1.5.2. 意義於後文定義

上述語彙單元的意義分散定義於整份規格：變數、可見性與模組在第 5 章；型別與容器在第 2 章；控制流程在第 3 章；函式、閉包與協程在第 4 章；meta 與魔法在第 6 章。

本第 1 章只涵蓋分類規則：在語彙階段，這些語彙單元被區分為關鍵字而非識別字。

## 1.5.3. 註冊的關鍵字（擴充機制）

除了上述保留清單之外，還有一種機制讓標準函式庫能註冊額外的關鍵字。它是元系統的一部分。註冊機制與確切的處理規則定義於 [`#[keyword(...)]` 魔法](../006-meta-system/606-keyword-magic.md)。

在本文件的範圍內，只承諾下列事項。保留關鍵字除了在 1.5.5 的名稱位置之外都不是識別字，並被區分為關鍵字。註冊的關鍵字只在包含其註冊的編譯單元內生效。若兩個集合衝突，註冊會被拒絕；保留關鍵字不能被覆寫。

## 1.5.4. 上下文關鍵字

下列字詞只在指定位置是關鍵字，在其他位置是一般識別字。

| 字詞                               | 作為關鍵字的位置                                                       | 定義 |
|------------------------------------|------------------------------------------------------------------------|------|
| `type`                             | 在項目開頭，後面接著識別字，再接著 `=` 或 `<` 時（型別別名）            | [2.6](../002-types/206-sum-types.md) |
| `any`                              | 型別位置（`any`、`any Tr`）                                            | [2.2](../002-types/202-any-slot.md) |
| `some`                             | 函式的回傳型別或參數型別，或型別別名的右側，位於約束之前（`-> some Tr`、`x: some Tr`、`type X = some Tr;`） | [4.1.13](../004-functions/401-function-declaration.md#4113-opaque-return-types-some-trait)、[4.1.15](../004-functions/401-function-declaration.md#4115-named-opaque-types-type-x--some-trait) |
| `in`                               | `for pat in` 的 `in`；在運算式中，成員運算子 `in` / `!in`（`∈` / `∉`） | [3.13.3](../003-expressions/313-loop-expressions.md#3133-for-iterator-based-repetition)、[3.2.3](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table) |
| `where`                            | 在函式簽章、`impl` 標頭，或 `struct` / `enum` / `union` / 型別別名標頭之後 | [2.10.4](../002-types/210-generics.md#2104-where-clauses) |
| `throws`、`rethrows`               | 在函式簽章的參數 `)` 之後                                              | [4.14](../004-functions/414-throws.md) |
| `catch`、`finally`                 | 在運算式或區塊之後（後綴子句）                                         | [3.14](../003-expressions/314-error-control-flow.md) |
| `const`                            | 在項目開頭，包括 `impl` / `trait` 主體中（`const` 項目、`const fn`），以及泛型參數清單中（`<const N: usize>`） | [5.2](../005-modules/502-const.md)、[2.11](../002-types/211-const-generics.md) |
| `static`、`extern`、`mut`          | 在項目開頭（包括 `extern "C"`、`static mut`）                          | [5.3](../005-modules/503-static.md)、[5.9](../005-modules/509-extern-block.md) |
| `union`                            | 在項目開頭，後面緊接著識別字時                                         | [2.5](../002-types/205-union.md) |
| `pkg`、`sub`                       | 緊接在 `pub(` 之後                                                     | [5.6](../005-modules/506-visibility.md) |
| `pkg`、`sup`                       | 作為路徑的第一個語彙單元，且後面緊接著 `::` 時                         | [5.7](../005-modules/507-paths.md) |

```nabi
let type = 1;                       // identifier
type Pair = (i32, i32);             // type alias (item start + identifier + =)
let in = 3;  for x in xs { }        // identifier / the in of for
fn catch(e: str) { }                // identifier: not after an expression
```

宣告名為 `any` 的型別是編譯錯誤（`struct any { }`），因為它無法與型別位置的 `any` 區分。

### 可見性限定詞：`pkg` / `sub`

`pkg` 與 `sub` 在 `pub(pkg)` 與 `pub(sub)` 的括號內作為限定詞使用，在其他位置是一般識別字。

```nabi
pub(sub) fn helper() { ... }   // inside pub( ): visibility qualifier
trait Sub { fn sub(self, rhs: Self) -> Self; }  // in ordinary position: identifier
```

### 路徑根：`pkg` / `sup`

這兩個字詞作為路徑的第一個語彙單元，且後面緊接著 `::` 時，一律是路徑根（[5.7 路徑](../005-modules/507-paths.md)）。在其他位置它們是一般識別字。

- `pkg::xxx`：從目前套件的根開始
- `sup::xxx`：從父模組開始

標準函式庫層級 `std`、`core` 與 `bare`（[0.5 標準函式庫層級](../000-meta/005-standard-libraries-tiers.md)）以及字詞 `nabi` 都不是關鍵字：區域名稱可以取用它們（[5.7.4](../005-modules/507-paths.md#574-std-core-bare-the-standard-library-tiers)）。

由於 `pkg::name` 一律被解讀為根，把 `pkg` 或 `sup` 用作模組或型別（`struct`、`enum`、`union`、`trait`、型別別名、`mod`）的名稱是編譯錯誤。它們可以用作變數、函式、欄位與方法的名稱。

```nabi
let pkg = 42;                   // variable name OK
fn sup() { }                    // function name OK
self.pkg                        // field / member name OK
mod sup { ... }                 // ERROR: cannot be a module name
struct pkg { }                  // ERROR: cannot be a type name
::core::iter::Iterator     // the library past a local `core`
```

## 1.5.5. 名稱位置中的關鍵字

在下列位置，所有關鍵字（包括保留關鍵字）都作為識別字使用，因為周圍的語彙單元已清楚表明這裡要的是名稱。

- 緊接在 `.` 之後（成員存取、方法呼叫）：`x.type`、`stream.match(p)`
- `struct` / `enum` 欄位名稱宣告：`struct Token { type: Kind; }`
- `struct` 字面值與 `struct` 模式中的欄位名稱：`Ev { match: 1 }`、`Ev { match: m } => ...`
- 具名引數的名稱：`f(type: 1)`。具名引數指稱一個參數，而參數名稱不能是保留關鍵字，因此實際上這裡只會出現上下文關鍵字。

`self` 是例外，即使在這些位置也不能作為識別字。在其他位置，要把關鍵字用作名稱，就把它寫成原始識別字 `r#match`（[1.4.4](104-identifiers.md#raw-identifiers)）。

```nabi
struct Token { type: Kind; for: str; }   // OK: field name
let k = tok.type;                         // OK: after .
make(type: Kind::Ident);                  // OK: named argument (contextual keyword)
struct S { self: i32; }                   // ERROR: self is the exception
```

## 1.5.6. 保留給日後使用的關鍵字

1.5.1 的最後一列 `macro`、`defer`、`dyn`、`move`、`gen` 與 `do`，是尚無任何語法使用的保留關鍵字，唯一的例外是閉包擷取清單中的 `move`（`[move x]`，[4.6.11](../004-functions/406-capture-semantics.md#4611-move-capture-the-move-x-capture-list)）。保留它們，是為了讓之後的版本能在不破壞程式碼的前提下賦予它們語法，也讓從其他語言帶過來的寫法得到一個說明 Nabi 改寫成什麼的錯誤：

| 關鍵字  | 保留用途     | Nabi 目前的寫法        |
|---------|--------------|------------------------|
| `macro` | 巨集項目     | 元區塊的 `@macro`（[6.8](../006-meta-system/608-meta-blocks.md)） |
| `defer` | 在作用域結束時執行的陳述式 | `using`（[3.17](../003-expressions/317-using-expression.md)）或 `finally`（[3.14](../003-expressions/314-error-control-flow.md)） |
| `dyn`   | 無：保留它是為了讓 Rust 的 `dyn Trait` 得到錯誤 | `any Trait`（[2.2](../002-types/202-any-slot.md)、[2.12](../002-types/212-trait-as-type-banned.md)）；`dyn Trait`、`&dyn Trait` 與 `Box<dyn Trait>` 是編譯錯誤，其說明會寫出 `any Trait` |
| `move`  | 在擷取清單之外，無：保留它是為了讓 Rust 的 `move \|x\| ..` 得到錯誤 | `[move x]` 擷取清單（[4.6.11](../004-functions/406-capture-semantics.md#4611-move-capture-the-move-x-capture-list)）；`move \|..\|` 是說明這一點的編譯錯誤 |
| `gen`   | 產生器形式   | `yield fn`、`yield { .. }`、`yield \|\| ..`（[4.11](../004-functions/411-generators.md)） |
| `do`    | 在主體之後才測試條件的迴圈 | `loop { ..; if !cond { break } }`（[3.13](../003-expressions/313-loop-expressions.md)） |

與每個保留關鍵字一樣，它們在 1.5.5 的名稱位置之外不能作為名稱；拼寫成其中之一的識別字是編譯錯誤，其說明會指出原始識別字（[1.4.4](104-identifiers.md#raw-identifiers)）。

```nabi
fn gen(n: i32) -> i32 { n }     // ERROR: `gen` is a reserved keyword, not a name
fn r#gen(n: i32) -> i32 { n }   // OK
let s = Step { move: 1 };       // OK: a field name (1.5.5)
let m = s.move;                 // OK: after .
```

元區塊的 `@macro`（[6.8](../006-meta-system/608-meta-blocks.md)）是 meta 語法，而不是對該關鍵字的使用。
