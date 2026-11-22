# 3.11. 模式

模式依形狀檢查一個值，同時把其中的部分繫結到變數。模式出現在多個位置：[`match` 運算式](312-match-expression.md)的分支、[`if` 運算式](310-if-expression.md)的 `if let`、[解構指派](315-destructuring.md)的 `let` 左側，以及其他位置。

```
pattern         ::= or_pattern
or_pattern      ::= unary_pattern ("|" unary_pattern)*
unary_pattern   ::= literal_pattern
                  | identifier_pattern
                  | "_"                                  // wildcard
                  | type_check_pattern
                  | tuple_pattern
                  | slice_pattern
                  | struct_pattern
                  | enum_variant_pattern
                  | range_pattern
                  | binding_pattern
                  | "(" pattern ")"                      // grouping
binding_pattern ::= identifier "@" unary_pattern
type_check_pattern  ::= identifier ":" type_expr
tuple_pattern   ::= "(" (pattern ("," pattern)* ","?)? ")"
slice_pattern   ::= "[" (slice_item ("," slice_item)* ","?)? "]"
slice_item      ::= pattern | ".." identifier?                  // ".." at most once
struct_pattern  ::= path "{" (field_pattern ("," field_pattern)* ","?)? "}"
field_pattern   ::= identifier (":" pattern)?
enum_variant_pattern ::= path ("(" (pattern | "..") ("," (pattern | ".."))* ","? ")")?   // ".." at most once
range_pattern   ::= literal_pattern (".." | "..=") literal_pattern
```

## 3.11.1. 模式的種類

### 字面值模式

字面值模式把值與常數比較。

```nabi
0
3.14f64
"hello"
true
none
'A'
```

從[數值字面值](../001-lexical/106-numeric-literals.md)到[布林值與哨兵值字面值](../001-lexical/109-other-literals.md)的每一種字面值形式都可以出現在模式位置；比對使用相等比較（`==`）。因此浮點數字面值模式 `0.0` 也會比對 `-0.0`，而沒有任何字面值能比對 NaN：請在守衛中測試它（`x if x.is_nan() =>`）。在和型別的受檢值上，字面值或範圍模式比對它所適合的成員型別（`1` 比對 `i32 | str` 的 `i32` 成員）。

### 識別字模式（繫結）

識別字模式在比對時引入一個新變數。

```nabi
x
count
name
```

無論值為何，它一律比對成功，並把值繫結到該識別字。它不得與[關鍵字](../001-lexical/105-keywords.md)的保留字衝突。若單獨的名稱解析為以 `use` 匯入的 `enum` 變體，它就不是繫結，而是該變體的 `enum` 模式（[5.5.7](../005-modules/505-use.md#importing-enum-variants)）。作為受檢值之 `enum` 型別之變體的單獨名稱，無論是否匯入，也是該變體（[2.4.8](../002-types/204-enum.md#248-bare-variants-from-the-expected-type)）；帶有酬載的變體以其酬載模式比對（`V(..)`）。

解析為 `const` 或 `static` 的名稱（無論大小寫：`LIMIT`、`limit`），或與受檢值之 `enum` 型別的某個變體相差兩次編輯以內的大寫開頭名稱（`Gren` 之於 `Green`），是編譯錯誤：作為繫結，它會比對每一個值。要與常數比較，請寫守衛：`x if x == LIMIT`；要比對變體，請寫出其路徑。其他大寫開頭的繫結（`Foo =>`）是一般的繫結，會觸發 `non_snake_case` lint。

```nabi
const LIMIT: i32 = 5;
match n {
    LIMIT => ..,              // ERROR (E0103): `LIMIT` here would be a binding that matches every value
    x if x == LIMIT => ..,    // compares with the const
    _ => ..,
}
```

### 萬用字元模式：`_`

```nabi
_
```

萬用字元模式也一律比對成功，但會捨棄值而不繫結它。用它來明確地忽略某個部分。

帶型別的萬用字元 `_: T` 只在值為 `T` 時比對成功，與型別檢查模式相同，而且不繫結任何東西。

```nabi
match v {
    _: i32 => "int",          // matches if i32, no binding
    _ => "other",
}
```

### 型別檢查模式：` name: Type`

```nabi
n: i32
v: User
```

型別檢查模式只在左側動態槽位或和型別的值是給定型別時比對成功。比對成功時，收窄為該型別的值被繫結到該識別字。

這個模式以模式的形式表達 [`any` 槽位](../002-types/202-any-slot.md)與[和型別](../002-types/206-sum-types.md)的型別收窄機制。

`:` 之後的型別是完整的 `type_expr`，因此包括 `|`。`n: i32 | i64` 是把值收窄為和型別 `i32 | i64` 的單一型別檢查模式。型別檢查模式的或模式要加上括號：`(n: i32) | (m: i64)`。在普通型別 `A` 的值上，`v: A | B` 一律比對成功；`v` 是作為 `A | B` 的該值。

```nabi
match v {
    n: i32 => use_int(n),       // matches if v is i32
    s: str => use_str(s),       // matches if v is str
}
```

### 元組模式

```nabi
(a, b)
(_, second)
(first, ..rest)
(..init, last)
(head, ..mid, tail)
(..)
```

`(p)` 是分組模式，與 `p` 相同，而不是元組。單一元素的元組以尾隨逗號撰寫：`(p,)`。元組型別（`(T,)`）與元組運算式（`(e,)`）遵循相同的規則。

元組模式以子模式比對元組的每個元素。`..` 是剩餘模式，用來忽略元組的一部分，或把它繫結到單一變數。`..` 在一個模式中最多只能出現一次，而長度必須與值相符（有 `..` 時長度可以自由相符）。

### 切片模式

```nabi
[]
[x]
[first, ..rest]
[.., last]
[first, .., last]
[0, n]
[[a, b], ..]
```

切片模式依元素比對 `List<T>`、`Slice<T>`、`SliceMut<T>` 或固定長度陣列 `[T; N]`，對於恰好有一個這類成員的和型別，則比對該成員。沒有 `..` 時，它比對元素數量恰好與其項目數相同的值；有 `..`（最多一次）時，比對至少有其周圍項目數量的值，中間的元素可以是任何東西。`..name` 繫結那些元素：值的切片 `xs[a:b]`（[3.6](306-indexing-and-slicing.md)），即共用這些元素的 `Slice<T>`。每個項目比對其位置上的元素，在 `..` 之前從前面算起，在其之後從後面算起，讀取方式與 `xs[i]` 相同：元素的複本，就像元組模式繫結其部分的複本一樣。

在固定長度陣列上，長度是已知的：項目數與之不符的模式是編譯錯誤（E0101），而 `let [a, b, c] = arr;` 或 `let [first, ..rest] = arr;` 會解構它。串列或切片的長度只在執行時才知道，因此 `let` 要以 `let .. else`（[3.16](316-let-else.md)）拆開它。

窮盡性會計算長度：`[]`、`[x]` 與 `[x, y, ..]` 涵蓋每一個串列，而遺漏某個長度的比對會指出它（`` `[_, _, ..]` not covered ``）。

### `enum` 模式

```nabi
Circle(r)
Rectangle(w, h)
Pending
Click { x, y }
Click { x, .. }
Rectangle(..)
Rectangle(w, ..)
```

`enum` 模式以變體名稱開頭。元組變體使用位置子模式（`Circle(r)`），具名變體使用欄位名稱與子模式（`Click { x, y }`），unit 變體只使用名稱（`Pending`）。元組變體模式可以使用一次剩餘的 `..`（`V(..)`、`V(a, ..)`、`V(.., z)`）。

> [!NOTE]
> 其他語言以 `Option<T>` 表達的東西，在 Nabi 中以[和型別](../002-types/206-sum-types.md)的 `T | none` 表達。因此沒有 `Some(v)` 或 `None` 這樣的 `enum` 變體模式。對 `T | none` 的比對使用型別檢查模式（`v: T`）與字面值模式（`none`）。

### `struct` 模式

```nabi
Point { x, y }
Point { x, .. }
Point { x: px, y: py }
Point { x, y: 0.0 }      // y matches a literal, x is bound
```

`struct` 模式由 `struct` 名稱與 `{ ... }` 內的欄位子模式組成。當欄位名稱與繫結的變數名稱相同時可以簡寫：`Point { x }` 與 `Point { x: x }` 相同。`..` 忽略其餘欄位。

### 型別模式（簡單）

```nabi
i32
f64
bool
```

若受檢值是和型別或 `any`，單獨的型別名稱（`f64 =>`、`i32 | str =>`）是只檢查值是否具有該型別、不繫結任何東西的型別模式。對其他任何受檢值，同一個名稱是識別字模式（繫結）。

### 範圍模式

```nabi
1..5
1..=5
'a'..='z'
```

[範圍運算式](307-range-expressions.md)的語彙單元原樣用在模式位置，若值位於範圍之內，模式就比對成功。它們可以用於整數與 `char`（`'a'..='z'`）。

### 或模式

```nabi
1 | 2 | 3
"yes" | "y" | "Y"
Pending | Circle(0.0f64)
```

或模式以 `|` 組合多個子模式；只要任一子模式比對成功，整體就比對成功。各子模式繫結的變數必須全都有相同的名稱與型別。

```nabi
match v {
    Ok(x) | OkAlt(x) => consume(x), // both variants bind the same name x with the same type
}
```

### 繫結模式：`name @ pat`

```nabi
n @ 1..=9
p @ Point { x: 0, .. }
```

若子模式 `pat` 比對成功，`name @ pat` 就把整個值繫結到 `name`。`pat` 內的繫結也會建立。或模式要以括號包起來，例如 `n @ (1 | 2)`。繫結模式不只可以用在 `match` 中，也可以用在 `let` 與 `for` 的模式中。

```nabi
match age {
    n @ 0..=17 => f"minor {n}",
    n          => f"adult {n}",     // ranges are open sets, so a catch-all is needed
}
```

## 3.11.2. 守衛

```nabi
pattern if condition
```

守衛是在模式比對成功之後才檢查的 `bool` 條件；若條件為 `false`，該分支就算作比對失敗。

```nabi
match (x, y) {
    (a, b) if a == b => "equal",
    (a, b) if a > b  => "first bigger",
    _                => "second bigger",
}
```

守衛是一個 `bool` 運算式，意義與 [`if` 運算式](310-if-expression.md)的條件相同。

## 3.11.3. 窮盡性檢查

窮盡性檢查驗證一組模式涵蓋了所有可能的輸入；確切規則定義於 [`match` 運算式](312-match-expression.md)。

簡言之，對封閉和型別或 `enum` 的比對必須涵蓋每個變體，或以萬用字元、識別字模式或型別模式涵蓋其餘部分。切片模式涵蓋的是長度：直到最長項目數為止的每個長度，而 `..` 模式涵蓋從其項目數起的長度。

## 3.11.4. 或模式中變數繫結的一致性

```nabi
(v: i32) | (_: i32) => v    // ERROR (E0102): one side binds, the other ignores
(n: i32) | (n: i64) => ...  // ERROR (E0102): n has different types; the diagnostic suggests n: i32 | i64
n: i32 | i64 => ...         // OK: n: i32 | i64
A(x) | B(y) => ...          // ERROR (E0102): x / y are asymmetric
```

或模式的每個子模式都必須以相同的型別繫結相同名稱的變數。一個名稱在所有選項中只有單一型別。

## 3.11.5. 模式位置一覽

| 位置                        | 形式                                          |
|-----------------------------|-----------------------------------------------|
| `match` 分支                | `match v { pattern => expr, ... }`            |
| `if let`                    | `if let pattern = expr { ... } else { ... }`  |
| `let-else`                  | `let pattern = expr else { ... };`            |
| `let` 解構                  | `let pattern = expr;`                         |
| `for` 迴圈                  | `for pattern in iter { ... }`                 |

各位置允許哪些模式，定義於該位置的規格。有些位置允許可反駁的模式；其他位置只允許不可反駁的模式。

## 3.11.6. 可反駁與不可反駁

可反駁的模式是比對可能失敗的模式：字面值、`enum` 變體、型別檢查、範圍等。不可反駁的模式一律比對成功：識別字、萬用字元，以及其解構形狀由型別保證的元組或 `struct`。

`let` 解構只允許不可反駁的模式，而 `match`、`if let` 與 `let-else` 兩者都允許。
