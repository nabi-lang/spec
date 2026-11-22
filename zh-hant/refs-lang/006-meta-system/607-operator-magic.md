# 6.7. 運算子魔法

`#[infix]`、`#[prefix]` 與 `#[postfix]` 這三個屬性只把運算子語彙單元連接到 trait 方法。優先順序與結合性由 [3.2.3](../003-expressions/302-precedence-and-associativity.md#323-優先順序與結合性表) 的表格在語言層級固定，這些屬性無法設定或改變它們。它們是讓[運算子分派](../003-expressions/301-operator-dispatch.md)的去糖機制運作所必需的核心魔法。

```
infix_attr      ::= "#" "[" "infix" "(" string_lit ("," like_arg)? ("," "swap")? ")" "]"
prefix_attr     ::= "#" "[" "prefix" "(" string_lit ("," like_arg)? ")" "]"
postfix_attr    ::= "#" "[" "postfix" "(" string_lit ("," like_arg)? ")" "]"
like_arg        ::= "like" "=" string_lit
```

## 6.7.1. 三個屬性

首先，是三個屬性用法的範例。

```nabi
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}

trait Degrees {
    #[postfix("°", like = "?")]    // extension token, same level as postfix ?
    fn degrees(self) -> f64;
}
```

各屬性附加的位置以及它所表達的運算形式：

| 屬性                  | 位置          | 形式                     |
|-----------------------|---------------|--------------------------|
| `#[infix(sym)]`       | trait 方法    | 二元（`a SYM b`）        |
| `#[prefix(sym)]`      | trait 方法    | 前綴一元（`SYM a`）      |
| `#[postfix(sym)]`     | trait 方法    | 後綴一元（`a SYM`）      |

## 6.7.2. 引數

各屬性接受的引數：

| 引數     | 意義                                                               | 是否必要？                                   |
|----------|--------------------------------------------------------------------|----------------------------------------------|
| `sym`    | 運算子語彙單元（字串字面值）                                       | 必要                                         |
| `like`   | 新語彙單元所跟隨的既有語彙單元（其等級與結合性）                   | 必要，僅限不在 3.2 表中的語彙單元            |
| `swap`   | 以交換運算元的方式連接（僅限 infix）                               | 選用                                         |

`sym` 與 `like` 和其他字串字面值一樣：語彙單元是字串的值，因此 `"\\"` 與 `r"\"` 都指稱單字元的語彙單元 `\`（`a \ b`）。

`swap` 以交換兩個運算元的方式連接二元運算。帶有 `#[infix("∈", swap)]` 的 `contains` 把 `a ∈ b` 去糖為 `b.contains(a)`。它是為了容器不在左側的運算子（`∈ ∉`）而存在的。

對 3.2 表中的語彙單元不能寫 `like`。沒有指定優先順序或結合性的引數（`prec`、`left` / `right` / `none` 等）；寫出這樣的引數是編譯錯誤。

```nabi
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]      // new token, same level and associativity as +
```

## 6.7.3. 連接機制

當 `#[infix("+")]` 附加在 trait 方法 `add` 上時，編譯器的行為如下：

首先，`+` 語彙單元被繫結到該 trait，作為其連接點。接著編譯器把運算式 `a + b` 去糖為 `<typeof(a) as the_trait<typeof(b)>>::add(a, b)`。優先順序與結合性是[優先順序與結合性](../003-expressions/302-precedence-and-associativity.md)的表格中為 `+` 固定的值（9，左結合）。

三個屬性都遵循相同的模式；被繫結的只有語彙單元、位置與 trait 方法。

運算元以該呼叫之引數的方式定型。字面值的右運算元（`[1.0, 2.0]`、`{..}`、數字）取為左運算元所選之 impl 的參數型別，因此當該 impl 的方法接受 `Vector<f64>` 時，`t ⊲ [1.0, 2.0]` 傳入的是 `Vector<f64>`。閉包運算元會滿足其 `fn` 型別上的 impl，就像它滿足約束一樣（[4.4.2](../004-functions/404-function-type.md#442-函式值與閉包值的型別)），而 blanket `impl<F: Tr> Op for F` 也適用於它。對沒有型別參數的 trait 而言，方法自身的參數型別就是右運算元的型別。

接受 `ref self` 的方法以方法呼叫接受其接收者的方式接受其左（或唯一的）運算元：當運算元是變數或變數的欄位時，方法所做的變更會寫回它，因此在 `fn draw(ref self, d: D)` 之下，`g ∼ d` 會像 `g.draw(d)` 一樣推進 `g`。閉包中依值擷取的變數，會像方法呼叫一樣被拒絕（[4.6.7](../004-functions/406-capture-semantics.md#467-擷取結構與參照控制代碼)）。

運算子不會擲回錯誤（[4.14](../004-functions/414-throws.md)），因此該方法必須宣告為不帶 `throws`；在會擲回錯誤的方法上使用運算子屬性，在宣告處就是錯誤。可能失敗的運算把錯誤以值的形式回傳，`Result<T, E>` 或 `T | E`，由呼叫者處理（`try (a ⧵ b).get()`、`match`），或保持為不帶該屬性的具名方法（`try a.solve(b)`）。

```nabi
trait Solve<Rhs = Self> {
    type Output;
    #[infix("⧵", like = "/")]     // ERROR: `solve` throws; operators do not throw
    fn solve(self, rhs: Rhs) throws (str) -> Output;
}
```

## 6.7.4. Unicode 與多重別名

把多個 `#[infix]` 附加在同一個方法上，會把多個語彙單元對應到該方法。

```nabi
trait Join<Rhs = Self> {
    type Output;
    #[infix("⊕", like = "+")]
    #[infix("⊞", like = "+")]
    fn join(self, rhs: Rhs) -> Output;
}
```

兩個語彙單元都呼叫 `join`。`÷ ≡ ≠ ≤ ≥` 不需要別名：語彙分析器把它們讀作 `/ == != <= >=`（[1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-固定語彙單元表)），因此它們到達與 ASCII 語彙單元相同的方法。實作 ASCII 語彙單元的 trait（`<=` 的 `PartialOrd`）就能讓兩種拼法都運作。

註冊這些拼法之一是編譯錯誤：語彙分析器永遠不會產生語彙單元 `≤`，因此對它的繫結永遠不會觸發。錯誤訊息會指出 ASCII 語彙單元以及應實作的 trait。Unicode 運算子之前的點（`.≤`、`.×`）也不是語彙單元：語彙分析器把 `.` 與該運算子分開讀取，而只有 1.10.3 的 ASCII 運算子才有廣播形式。

```nabi
trait Before {
    #[infix("≤")]     // ERROR: `≤` is the fixed operator `<=`; implement `ops::PartialOrd`
    fn before(self, rhs: Self) -> bool;
}
```

## 6.7.5. 衝突檢查

若同一組（語彙單元、位置）附加了不同的 trait，就是註冊時的錯誤。

```nabi
trait MyAdd { #[infix("+")] fn add(...) ... }
// another library / module.
trait Add { #[infix("+")] fn add(...) ... }
// → two traits for the binary '+' position. ERROR.
```

每一組（語彙單元、位置）恰好只接受一個註冊。

## 6.7.6. 同一語彙單元的多個位置

可以同時用在前綴與二元位置的語彙單元，例如 `-`，被當作分開的註冊處理。

```nabi
trait Sub<Rhs = Self> {
    type Output;
    #[infix("-")]
    fn sub(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}
```

例如在 `-a + b` 中，前綴的 `-` 結合得比二元的 `+` 緊，因此剖析為 `(-a) + b`。

二元的 `-` 與前綴的 `-` 是分開的對應，因此把它們附加在不同的 trait 上沒有問題。

## 6.7.7. 帶點前綴（廣播）的對應

[廣播運算子](../003-expressions/303-broadcast-operators.md)的帶點前綴語彙單元透過相同的魔法註冊。

```nabi
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+")]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*")]
    fn dot_mul(self, rhs: Rhs) -> Output;
}
```

只有語彙單元不同；註冊機制完全相同。

## 6.7.8. 對應的可見性

透過 `#[infix]` 等屬性的註冊，只在該 trait 可見之處生效。可見是指在 [5.6](../005-modules/506-visibility.md) 之下可以存取：能以路徑到達的 `pub` trait 就足夠；不需要 `use`。在該 trait 不可見之處，該運算子也不起作用。

這條規則符合函式庫的隔離：在某個套件中定義的使用者定義運算子 trait，只在匯入該套件的程式碼中起作用。`pub` 運算子 trait 的語彙單元也到達同樣的程式碼：語彙分析器在相依於註冊套件的每個套件的檔案中切出它們，因此來自相依套件的 `<~` 或 `<-` 在那裡的切分方式與在其自身套件中相同（[1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-擴充語彙單元)）。`#[literal]` 註冊對 `pub` 處理器遵循相同的規則，作用於直接相依於其套件的套件（[6.5.2](605-literal-magic.md#652-形式)）；其他的則留在其套件中。

標準函式庫的運算子 trait（`Add`、`Sub`、`Mul`……）位於 `core::ops`。運算子不需要 trait 名稱在作用域內：`core` trait 的對應適用於每份原始碼，因此算術與比較運算式不需要另外匯入就能運作。指名 trait 本身，例如 `impl ops::Add for V`，要透過 prelude 中的模組 `ops` 或 `use`（[stdlib 0.2.3](../../refs-standard-libraries/000-meta/002-import.md)）。

## 6.7.9. 使用者定義的運算子

使用者也可以在自己的 trait 上附加 `#[infix]` 等屬性來定義運算子對應。

```nabi
trait CustomCombine<Rhs = Self> {
    type Output;
    #[infix("⊕")]
    fn combine(self, rhs: Rhs) -> Output;
}

impl CustomCombine for MyType { ... }

let r = a ⊕ b;            // user token
```

不在 3.2 表中的語彙單元，例如 `⊕`，以 `like = "+"` 指名一個既有的語彙單元，並跟隨該語彙單元的等級與結合性。不能建立新的等級。該語彙單元必須滿足[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md#1104-擴充語彙單元)的擴充語彙單元條件：每個字元都必須是 Unicode `Pattern_Syntax`，而若它與既有的固定語彙單元相同就是錯誤。

語彙單元可以像固定語彙單元那樣開頭：`#[infix("<-", like = "+")]` 讓 `c <- 3` 成為一個運算子，因為語彙分析器採用最長匹配。這樣的語彙單元，即以固定語彙單元開頭字元開頭的語彙單元，在註冊它的套件的檔案中起作用，而若來自 `pub` trait，也在相依於它的套件中起作用；在其他地方，`a<-1` 仍是 `a < -1`（[1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-擴充語彙單元)）。

二元的 `@` 也可以用 `like` 繫結：`#[infix("@", like = "*")]` 讓 `a @ b` 成為矩陣乘積，與 NumPy 相同。`@` 仍是固定語彙單元，而文法中沒有其他二元的 `@`：語言只在其前沒有運算元結束之處寫它，即在 meta 語法之前（`@derive`、`@Self`）、模式中繫結名稱之後（`n @ (a, b)`），以及黏在 `return` 上（`return@f`）。因此完整運算元之後的 `@`，在其 trait 可見之處就是該運算子，而那些其他用法在同一個檔案中仍保有其意義。`@` 不能作為前綴或後綴運算子繫結，因為那些位置是 meta 語法。

二元的 `~` 同樣是空著的：語言只把 `~` 讀作前綴的位元 not（`~x`），絕不出現在運算元之後，因此 `#[infix("~", like = "==")]` 讓 `x ~ d`（「服從某分布」，或模型公式 `y ~ x`）成為運算子，而 `~x` 保有其意義。其他任何固定語彙單元只能在 3.2 表為它給出形式之處繫結：後綴的 `%` 是錯誤，錯誤訊息會指向數字後綴（`95pct`，[6.5.7](605-literal-magic.md#657-使用者定義的字面值)），而 `->` 或 `|>` 是語法。後綴的 `!`（階乘 `n!`）也是錯誤：在名稱之後 `!` 開啟巨集呼叫（`name!(..)`），在運算元之前它是 `not`；請寫函式或方法（`factorial(n)`、`n.factorial()`）。

語彙單元不含識別字字元：`′`（`f′` 是名稱）與 `ℱ`（字母）不能作為運算子。請寫方法（`a.t()`）或函式（`fourier(x)`），或選用 `†` 之類的符號（[1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-擴充語彙單元)）。

## 6.7.10. 優先順序與結合性等級

運算子等級的唯一權威位置是 [3.2.3 優先順序表](../003-expressions/302-precedence-and-associativity.md#323-優先順序與結合性表)。標準函式庫與使用者程式碼都不能設定或改變等級；新的語彙單元透過 `like` 附著到既有的某個等級。

## 6.7.11. 何時求值

`#[infix]` 等註冊在剖析之後的 meta 展開階段處理。之後所有的運算式剖析都會查閱註冊表。

因此在同一個編譯單元內，註冊與使用的順序無關緊要，因為掃描預處理會事先收集它們。3.2 表中語彙單元的優先順序從一開始就固定，與註冊無關。

## 6.7.12. 不同等級的別名（無）

附加在同一個 trait 方法上的所有別名都必須是同一等級的語彙單元。

```nabi
trait Before {
    #[infix("<")]
    #[infix("⊲", like = "+")]    // ERROR: < is level 4, + is level 9
    fn lt(self, rhs: Self) -> bool;
}
```

這項限制源自別名共用相同意義的假設。

## 6.7.13. 只能附加在 trait 方法上

`#[infix]`、`#[prefix]` 與 `#[postfix]` 只能附加在 trait 的方法簽章上。它們不能附加在自由函式、`impl` 方法或 trait 之外的一般方法上。

```nabi
fn my_add(a: i32, b: i32) -> i32 { a + b }
#[infix("+++")]                              // ERROR: cannot attach to a free function
fn my_concat(a: str, b: str) -> str { ... }
```

理由是把運算子對應繫結到 trait 系統，並保證一致的分派。

## 6.7.14. 呼叫值：`ops::Call`

被呼叫者是值而非函式的呼叫 `v(a)`，若其型別實作了 `core::ops::Call`（[stdlib 2.3](../../refs-standard-libraries/200-core/203-ops.md)），就是 `v.call(a)`。引數以一個值傳入：單一引數就是它本身，多個引數是元組（`p(x, y)` 是 `p.call((x, y))`），沒有引數則是 `()`。因此內插器、多項式或分布的密度讀起來就像它所代表的函式：`s(1.5)`、`xs |> s`，而 `s.(xs)` 像 `fn` 一樣把它套用到每個元素。

```nabi
struct Spline { a: f64; }

impl ops::Call<f64> for Spline {
    type Output = f64;
    fn call(self, x: f64) -> f64 { self.a * x }
}

let s = Spline { a: 2.0 };
s(1.5);                          // 3: s.call(1.5)
```

`Args` 不同的多個 impl 依引數選擇，與其他多載的 trait 方法相同（[3.1.3](../003-expressions/301-operator-dispatch.md#313-不經-vtable-的分派)）：具體的 `Call<f64>` 與泛型的 `impl<const N: usize> Call<Array<f64, N>>` 並存時，能對任何秩的陣列使用 `s(1.5)` 與 `s(xs)`，`s(1)` 採用 `f64` 的 impl，`s([1.0, 2.0])` 採用秩 1 的 impl。容納可呼叫值的欄位以 `(r.f)(x)` 呼叫；`r.f(x)` 指稱方法（[4.9.6](../004-functions/409-method.md#496-自由函式與方法的呼叫形式)）。約束 `F: Call<f64, Output = f64>` 透過它呼叫 `f(x)`。引數只能是位置引數。`fn` 值與閉包直接被呼叫，不實作任何東西。呼叫其他任何型別的值是錯誤，其說明會顯示應寫的 impl。

## 6.7.15. 總結

運算子魔法與 `#[lang]` 一起，是在[語言與標準函式庫的邊界](../000-meta/004-language-vs-standard-libraries.md)中實現語言語彙單元與函式庫 trait 繫結的核心機制。

標準函式庫的運算子規格透過這個魔法決定每個語彙單元的意義。
