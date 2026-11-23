# 4.1. 函式宣告

本節定義以 `fn` 關鍵字（[關鍵字](../001-lexical/105-keywords.md)）開頭之具名函式的宣告形式。

## 4.1.1. 形式

函式宣告的文法如下：

```
fn_decl       ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
fn_modifiers  ::= "unsafe"? "async"? "yield"?
return_type   ::= "->" type_expr
param_list    ::= param ("," param)* ","?
```

這裡的 `visibility?` 是[可見性](../005-modules/506-visibility.md)中的 `pub`、`pub(pkg)`、`pub(sub)` 之一，或省略（模組私有）。`fn_modifiers` 是三個修飾詞 [unsafe](415-unsafe.md)、[async](412-async.md) 與 [yield（產生器）](411-generators.md)的位置，每個都**依固定順序寫零次或一次**（詳細規則見 [4.1.11 fn 修飾詞的順序與組合](#4111-fn-修飾詞的順序與組合)）。`throws_clause?` 的意義定義於 [throws](414-throws.md)；本文件不涵蓋可見性、修飾詞或 throws 關鍵字本身的意義。

非正式地說，整體形狀如下：

```nabi
fn name(parameters) -> ReturnType {
    body
}
```

各組成部分的意義：

- 以 `fn` 關鍵字開頭。
- 函式名稱遵循[識別字](../001-lexical/104-identifiers.md)的形式。
- 接著是括號中的參數清單；詳細規則定義於[參數](402-parameters.md)。
- 回傳型別是選用的：位於 `->` 語彙單元（[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)）之後的[型別運算式](../002-types/201-type-expressions.md)，或不透明的 `some Tr`（[4.1.13](#4113-不透明回傳型別some-trait)）。
- 主體是[區塊運算式](../003-expressions/309-block-expressions.md)。

## 4.1.2. 最簡單的範例

最簡單的函式宣告如下：

```nabi
fn greet() {
    print("hello");
}

fn add(x: i32, y: i32) -> i32 {
    x + y
}
```

## 4.1.3. 省略回傳型別

回傳型別的 `->` 部分可以省略。其意義取決於函式的[可見性](../005-modules/506-visibility.md)。

在模組私有或 `pub(sub)` 函式上：

- 回傳型別由主體推論；主體最後一個運算式的型別成為回傳型別。
- 若無法推論或有歧義，就是編譯錯誤。

在下面的範例中，回傳型別被省略，但從最後一個運算式的型別推論出 `i32`。

```nabi
fn double(x: i32) {              // return type omitted
    x * 2                         // inferred: i32
}
// the above is equivalent to fn double(x: i32) -> i32
```

若主體不產生值（最後一個運算式以終止符結束），回傳型別是 unit（`()`）。

```nabi
fn print_msg(s: str) {
    print(s);                     // terminator at the end: returns unit
}
```

在 `pub` 或 `pub(pkg)` 函式上，其他模組會呼叫它，因此其簽章要寫出來，而修改主體不能改變簽章。那裡缺少 `->` 表示 `()`，無論主體做了什麼，即使主體永不回傳也一樣。若主體給出一個值（其最後一個運算式或某個 `return v` 的型別不是 `()`），就是編譯錯誤（E0138），錯誤訊息會指出應寫的型別。

```nabi
pub fn log(s: str) {              // returns ()
    print(s);
}

pub fn double(x: i32) {           // ERROR: the body gives i32; write `-> i32`
    x * 2
}

pub fn double2(x: i32) -> i32 {   // OK
    x * 2
}
```

固有 `impl` 的方法依其自身的可見性遵循相同的規則。trait `impl` 的方法依 trait 宣告的寫法寫出其 `->`，因為回傳型別必須與之相符（[2.9.1](../002-types/209-impl.md)）；只有其未定型的參數從 trait 取得型別（[4.2.2](402-parameters.md#422-省略型別)）。

若寫出了回傳型別，但函式主體與之不符，就是編譯錯誤。

```nabi
fn bad() -> i32 {
    "hello"                       // ERROR: str is not i32
}
```

## 4.1.4. 省略參數型別

在模組私有或 `pub(sub)` 函式上，沒有型別的參數被視為隱含泛型（[`any` 槽位](../002-types/202-any-slot.md)、[泛型](../002-types/210-generics.md)）。

```nabi
fn id(x) {                        // x: T (generic)
    x
}
// monomorphized by the type at the call site
let a = id(42i32);                // id<i32>
let b = id("hi");                 // id<str>
```

`pub` 或 `pub(pkg)` 函式要寫出每個參數的型別；未定型的參數是編譯錯誤（[4.2.2](402-parameters.md#422-省略型別)）。

若參數意在成為動態槽位，請明確寫出 `any`。

```nabi
fn dyn_id(x: any) {               // x is an any slot (runtime dispatch)
    x
}
```

## 4.1.5. 主體

函式主體遵循[區塊運算式](../003-expressions/309-block-expressions.md)的規則，並具有下列性質：

- 它包含零或多個陳述式。
- 它可以以零或一個最後的運算式（不帶終止符）結束，其值成為函式的回傳值。
- 它可以在末尾以 `return`（[`return`](403-return.md)）明確回傳。

## 4.1.6. 空的主體

主體為空的函式回傳 unit。

```nabi
fn nothing() { }                  // empty body: returns unit
```

## 4.1.7. 回傳型別與主體的最後一個運算式

在有明確回傳型別的函式中，不帶終止符的最後一個運算式成為回傳值。

```nabi
fn f() -> i32 {
    42i32                         // last expression is the return value
}
```

在上面的範例中，`42i32` 沒有終止符，因此它成為函式的回傳值。若寫成帶終止符的 `42i32;`，值會被捨棄而主體回傳 unit，這與宣告的 `-> i32` 衝突，是編譯錯誤。

```nabi
fn f() -> i32 {
    42i32;                        // ERROR: terminator discards the value
                                  // (does not match return type i32)
}
```

## 4.1.8. 函式主體中明確的 `return`

在主體內，`return` 關鍵字明確地回傳；詳細內容定義於 [`return`](403-return.md)。

```nabi
fn early(x: i32) -> i32 {
    if x < 0 { return -1; }
    x * 2
}
```

## 4.1.9. 可見性

函式上的可見性關鍵字（`pub`、`pub(pkg)`、`pub(sub)`）控制它公開到什麼程度。

```nabi
pub fn area(r: f64) -> f64 { ... }              // callable from external packages
pub(pkg) fn helper(x: i32) -> i32 { ... }       // only within the same package
fn _internal(s: str) { ... }                     // module-private (default)
```

各可見性等級的確切意義定義於[可見性](../005-modules/506-visibility.md)。

## 4.1.10. 同一模組內不得重複定義

在同一個模組中宣告兩個同名的函式是編譯錯誤（E0026）。Nabi 沒有多載。

```nabi
fn add(x: i32, y: i32) -> i32 { x + y }
fn add(x: f64, y: f64) -> f64 { x + y }   // ERROR: duplicate
```

要依型別改變行為，請結合 trait 與泛型（[`trait`](../002-types/208-trait.md)）。

模組的函式、`const` 與 `static` 共用一個值命名空間：在同一個模組中與其中另一個同名的 `const`、`static` 或函式也是編譯錯誤。元組結構或 unit 結構的建構子也在這個命名空間中（[2.3.7](../002-types/203-struct.md#237-同一模組內不得重複定義)）：`struct Point(i32, i32);` 與 `fn Point`、`const Point` 或 `static Point` 並存是重複定義。不同的參數數量、`async` 或型別參數都不能讓兩個函式區分開來。要接受不同數量的引數，請為一個函式的尾端參數提供預設值（[4.2.3](402-parameters.md#423-預設值)）。

## 4.1.11. fn 修飾詞的順序與組合

函式修飾詞（`unsafe`、`async`、`yield`）**必須依此順序撰寫**：

```
unsafe → async → yield
```

違反這個順序是語法錯誤（E0014）。在文法層級也不可能把同一個修飾詞寫兩次（`unsafe?` 中的 `?` 只允許零或一次）。

```nabi
unsafe async yield fn ok() -> i32 { ... }   // OK: prescribed order
async unsafe fn bad1() { ... }           // ERROR: async before unsafe
yield async fn bad2() { ... }            // ERROR: yield before async
unsafe unsafe fn bad3() { ... }          // ERROR: duplicate unsafe
```

這條規則保證意義相同的函式一律以相同方式撰寫，簡化了格式化工具、linter 與閱讀。

### 8 種可能的組合

三個修飾詞都是選用的，因此有 8 種可能的組合，定義如下。

| 組合                    | 呼叫結果                               | 意義                 | 定義於                             |
|-------------------------|----------------------------------------|----------------------|------------------------------------|
| `fn`                    | `R`                                    | 一般函式             | [4.1](401-function-declaration.md) |
| `unsafe fn`             | `R`                                    | 不安全函式           | [4.15](415-unsafe.md)              |
| `async fn`              | `AsyncFuture<R>`                   | 非同步函式           | [4.12](412-async.md)               |
| `yield fn`              | `GenIter<R>`                       | 產生器               | [4.11](411-generators.md)          |
| `async yield fn`        | `AsyncStream<R>`                   | 非同步串流           | [4.13](413-stream.md)              |
| `unsafe async fn`       | `AsyncFuture<R>`                   | 不安全的非同步函式   | （各修飾詞之和）                   |
| `unsafe yield fn`       | `GenIter<R>`                       | 不安全的產生器       | （各修飾詞之和）                   |
| `unsafe async yield fn` | `AsyncStream<R>`                   | 不安全的非同步串流   | （各修飾詞之和）                   |

帶有 `unsafe` 的組合很簡單：整個函式主體成為不安全作用域（如 4.15 所定義），而結果型別與協程行為遵循其他修飾詞（`async` / `yield`）的定義。也就是說，`unsafe` 只是在主體內加入 `unsafe` 區塊所允許的操作（[4.15.2](415-unsafe.md#4152-unsafe---區塊運算式)）：呼叫 `unsafe fn`、解參照原始指標、讀取 `union` 欄位，以及在值被交出之後使用它。

### 協程函式的結果型別

呼叫標示了 `async` / `yield` 的函式（或區塊、閉包）會得到一個標準函式庫型別。宣告的 `-> R` 給出元素 / 結果型別 `T`。

| 形式             | 結果型別          | 實作的 trait           | 帶 `throws (E)` 時        |
|------------------|-------------------|------------------------|---------------------------|
| `yield fn`       | `GenIter<T>`      | `Iterator<Item = T>`   | `TryGenIter<T, E>`        |
| `async fn`       | `AsyncFuture<T>`  | `Future<Output = T>`   | `TryAsyncFuture<T, E>`    |
| `async yield fn` | `AsyncStream<T>`  | `Stream<Item = T>`     | `TryAsyncStream<T, E>`    |

這些型別位於 `iter`（`GenIter`、`TryGenIter`）、`future`（`AsyncFuture`、`TryAsyncFuture`）與 `stream`（`AsyncStream`、`TryAsyncStream`），不在 prelude 中：請透過其模組（`future::AsyncFuture<T>`）或 `use` 指稱它們。`Try*` 變體在下一個推進點擲回錯誤。例如 `TryGenIter<T, E>` 的 `next` 是 `next(ref self) throws (E) -> T | none`，而 `TryAsyncFuture` 在 `try await` 處擲回錯誤。這些變體不實作一般的 `Iterator` / `Future` / `Stream` trait（[4.14.10](414-throws.md#41410-throws-對和型別簽章的影響)）；`TryGenIter` 實作 `TryIterator`，`TryAsyncStream` 實作 `TryStream`（[stdlib 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator可能失敗的迭代)、[2.8.11](../../refs-standard-libraries/200-core/208-stream.md#2811-trystream可能失敗的串流)）。

```nabi
async fn fetch_user(id: i64) -> User { ... }
let f: future::AsyncFuture<User> = fetch_user(42i64);

yield fn lines(path: str) throws (IoError) -> str { ... }
let it: iter::TryGenIter<str, IoError> = lines("a.txt");
```

### 閉包與區塊形式

相同的順序規則也適用於 `async` 與 `yield` 的閉包 / 區塊形式。例如，產生 Stream 的閉包一律寫成 `async yield ||`；`yield async ||` 不是閉包形式：它讀作對 `async ||` 閉包套用 `yield`：在產生器之外是錯誤（E0059，在產生器之外的 `yield`），在產生器之內則產出該閉包。閉包 / 區塊形式見各修飾詞的文件。

## 4.1.12. 參數、回傳與主體的求值順序

呼叫的求值順序如下：

1. 對引數運算式求值（由左至右）。
2. 配對參數並開始對函式主體求值。
3. 對主體求值；完成時，最後一個運算式或 `return` 的值就是結果。
4. 把結果交付給呼叫處。

詳細的求值順序定義於[參數](402-parameters.md)的參數求值順序一節。

## 4.1.13. 不透明回傳型別：`some Trait`

```
opaque_ret ::= "some" (bound | "(" bound ("+" bound)* ")")
bound      ::= type_path | fn_type
```

函式的回傳型別（在 `->` 之後，也包括 `throws (E) ->` 之後）可以是 `some Tr`：一個具有 `Tr` 的具體型別，由主體選定並對呼叫者隱藏。`Tr` 是 trait，可以像在約束中那樣帶有關聯型別繫結（`some Iterator<Item = str>`），或是函式型別（`some fn(i32) -> i32`）。多個約束放在括號中：`some (Iterator<Item = i32> + Clone)`。

```nabi
pub fn evens(n: i32) -> some Iterator<Item = i32> {
    (0..n).filter(|x| x % 2 == 0)
}

fn adder(k: i32) -> some fn(i32) -> i32 {
    |x| x + k                     // x: i32, from the bound
}

let total = evens(10).map(|x| x * x).sum();
```

- 具體型別是主體值的型別，依 [4.1.3](#413-省略回傳型別) 推論，在 `pub` 函式上也是如此。呼叫是靜態的：沒有標籤，沒有裝箱，與[泛型](../002-types/210-generics.md)相同。
- 每個函式只有一個具體型別：型別不同的回傳路徑是錯誤（`` the return paths of `f` have different types .. ``），說明 `` to return several types, write `-> any Tr` ``。
- 具體型別必須具有每個約束：否則在約束處報錯 `` `f` returns `ListIter<i32>`, which is not `Iterator<Item = str>` ``。
- 呼叫者只看得到約束：可以呼叫約束（及其父 trait）的方法、在約束是迭代器時迭代它、把它傳到預期帶有這些約束之泛型參數的位置，以及在約束是函式型別時呼叫它。指稱或使用具體型別是錯誤：`` no method `contains` on `some Iterator<Item = i32>`: callers see only its bounds ``；`` `let b` expects `Range<i32>`, found `some Iterator<Item = i32>` ``。
- 同一個函式一律回傳相同的型別，因此兩次呼叫的結果是同一個型別。不同函式的結果是不同的型別。
- `some` 是函式的回傳型別、參數的型別（[4.1.14](#4114-some-參數)），或具名不透明型別的右側（[4.1.15](#4115-具名不透明型別type-x--some-trait)）。在其他位置（`let` 標註、欄位、型別引數、閉包參數）是錯誤，錯誤訊息會說明應寫什麼：變數不寫標註，欄位或型別引數寫泛型 `<T: Tr>`，多種型別的值寫 `any Tr`。
- 跨套件時，函式的主體隨套件的介面傳遞，因此相依的建置知道具體型別；其使用者仍只看得到約束。
- 沒有 `Fn` trait：函式約束就是函式型別 `fn(A) -> R`。

## 4.1.14. `some` 參數

參數的型別可以是 `some Tr`（約束如 [4.1.13](#4113-不透明回傳型別some-trait)）：一個匿名的型別參數。`fn show(x: some Display)` 就是 `T` 沒有名稱的 `fn show<T: Display>(x: T)`。

```nabi
fn show(x: some Display) -> str { f"<{x}>" }
fn both(a: some Display, b: some Display) -> str { f"{a}{b}" }
fn count(it: some Iterator<Item = i32>) -> usize { it.count() }
fn twice(f: some fn(i32) -> i32, x: i32) -> i32 { f(f(x)) }

let s = both(1, 2.5);            // a: i64, b: f64
let n = count([1, 2, 3].iter());
let t = twice(|x| x * 3, 2);     // x: i32, from the bound
```

- 每次呼叫中，該參數有一個靜態型別，即引數的型別，與[泛型](../002-types/210-generics.md)相同：沒有標籤，沒有裝箱。引數必須具有每個約束。
- 每個 `some` 參數都是自己的型別參數：`both(1, 2.5)` 讓 `a` 與 `b` 有不同的型別。
- 主體只看得到約束，與具名參數 `T: Tr` 相同。
- `some fn(A) -> R` 接受函式值（函式或閉包），而閉包的參數從它取得型別。
- 自由函式、方法與 trait 方法都可以有 `some` 參數，並與具名的型別參數並存。
- 該型別沒有名稱：呼叫處的 `::<..>` 只給出具名的型別參數。寫出更多是錯誤，說明會指出 `some` 參數的型別就是引數的型別：請傳入引數，或宣告 `<T: Tr>` 以便用 `::<..>` 選擇 `T`。

`any Tr` 與 `some Tr` 是一對。`any Tr` 是動態槽位：它容納任何具有 `Tr` 之型別的值，每次可以是不同的型別，而呼叫在執行時分派（[2.2](../002-types/202-any-slot.md)）。`some Tr` 是一個被隱藏的靜態型別：被呼叫者的主體（對回傳而言）或呼叫的引數（對參數而言）固定它，而呼叫靜態分派。

## 4.1.15. 具名不透明型別：`type X = some Trait`

```
opaque_alias ::= visibility? "type" identifier "=" opaque_ret ";"
```

`type X = some Tr;` 為一個具有 `Tr` 的具體型別命名（約束如 [4.1.13](#4113-不透明回傳型別some-trait)）。與 `-> some Tr` 不同，這個型別有名稱，因此可以寫在任何能放型別之處：欄位、`let` 標註、型別引數（`List<X>`）、參數與回傳型別。

```nabi
mod nums {
    pub type Evens = some Iterator<Item = i64>;

    pub fn evens(n: i64) -> Evens {        // defines Evens
        (0..n).filter(|x| x % 2 == 0)
    }

    pub fn first(e: Evens) -> i64 | none { e.next() }
}

struct Shelf { evens: nums::Evens; more: List<nums::Evens>; }

fn total(e: nums::Evens) -> i64 { e.sum() }

let s = Shelf { evens: nums::evens(10), more: [nums::evens(4)] };
println(total(s.evens));                  // 20
```

- 定義函式是 X 所屬模組中回傳型別為 `X`（寫成 `X` 或 `self::X`）的自由函式。每個定義函式的具體型別是其主體值的型別，依 `-> some Tr` 的方式推論，並且必須具有每個約束。必須至少有一個，而且它們必須全都給出相同的型別：一個都沒有時是 `` `X` has no defining function ``（E0145），說明 ``a function in this module that returns `X` gives its type``；有兩個不同時是 `` `g` returns `B`, but `X` is `A` (from `f`) ``（E0145），說明會提示回傳相同的型別或呼叫 `f`。
- 定義函式沒有型別參數，`X` 也沒有：泛型的定義函式是錯誤（E0145），其說明會寫出 `fn f<T>(..) -> some Tr`，即每次呼叫各有一個隱藏型別（[4.1.13](#4113-不透明回傳型別some-trait)）。
- 方法，以及回傳 `List<X>` 或 `X | none` 的函式，不定義 `X`；它們使用它。
- 在 X 所屬模組內（不含其子模組），`X` 就是其具體型別：模組的函式可以呼叫它的方法、讀取它的欄位，並像建立該型別的任何值一樣建立它。
- 在 X 所屬模組之外，`X` 是一個只具有其約束的獨立型別：值以 `X` 的身分被存放、傳遞、回傳以及保存在 `List<X>` 中，並透過約束（及其父 trait）的方法使用，與 `-> some Tr` 的結果相同。具體型別是隱藏的：約束沒有提供的方法或欄位會得到 `` no method `len` on `Evens`: outside `nums`, only its bounds are visible ``；在預期另一個型別之處放 `X` 的值，或在預期 `X` 之處放另一個型別的值（模組外的函式從自己的主體回傳 `X`），會得到 `` `let n` expects `i64`, found `X`: outside `m`, `X` is opaque ``（E0146），說明會提示把值保持為 `X`、從 X 的定義函式取得一個值，或把程式碼移進 X 的模組。
- 以函式型別為約束時（`type Step = some fn(i64) -> i64;`），值只能被呼叫，因此該型別在任何地方都保持為其具體型別。
- 呼叫與儲存都是靜態的：型別為 `X` 的欄位以行內方式容納具體型別，沒有標籤也沒有裝箱。
- `pub type X = some Tr;` 匯出該名稱。跨套件時，定義函式的主體隨套件的介面傳遞（[4.1.13](#4113-不透明回傳型別some-trait)），因此相依的建置能配置 `X`；相依者只看得到約束。
- `type X = any Tr;` 是動態槽位 `any Tr` 的一般型別別名（[2.6.9](../002-types/206-sum-types.md#269-與型別別名結合)）：`X` 的每個值可以有不同的型別，而呼叫在執行時分派。`type X = some Tr;` 則是由其模組固定的一個靜態型別。
