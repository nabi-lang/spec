# 3. 運算式概觀

本章定義運算式與陳述式的形式，以及讓每個控制結構都成為運算式所帶來的結果。各種運算式及其語意在後續文件中說明。

## 目錄

- [3.1. 運算子分派](301-operator-dispatch.md)
- [3.2. 優先順序與結合性](302-precedence-and-associativity.md)
- [3.3. 廣播運算子](303-broadcast-operators.md)
- [3.4. 指派與複合指派](304-assignment.md)
- [3.5. 成員存取與方法呼叫](305-member-access-and-method-call.md)
- [3.6. 索引與切片](306-indexing-and-slicing.md)
- [3.7. 範圍運算式](307-range-expressions.md)
- [3.8. 可選鏈結與空值合併](308-optional-chaining-and-coalescing.md)
- [3.9. 區塊運算式](309-block-expressions.md)
- [3.10. `if` 運算式](310-if-expression.md)
- [3.11. 模式](311-patterns.md)
- [3.12. `match` 運算式](312-match-expression.md)
- [3.13. 迴圈運算式](313-loop-expressions.md)
- [3.14. 錯誤控制流程](314-error-control-flow.md)
- [3.15. 解構指派](315-destructuring.md)
- [3.16. `let-else`](316-let-else.md)
- [3.17. `using` 運算式](317-using-expression.md)
- [3.18. 管線運算式](318-pipe-expression.md)

## 運算式與陳述式

運算式是產生值的結構：對它求值會得到某個型別的值。**陳述式**是不產生值、以 `;` [陳述式終止符](../001-lexical/103-statement-terminator.md)結束的程式碼單位。

Nabi 以下列規則連結兩者。

```
<statement> = <expression> ';'        // a statement that discards the expression's result
```

陳述式是附加了終止符的運算式。幾乎每一種程式碼結構都是產生值的運算式；捨棄那個值，它就成為陳述式。

## 控制結構是運算式

每個控制結構（`if`、`match`、`loop`、`while`、`for`、`try`）也都是運算式；`try` 運算式的值見 [3.14.6](314-error-control-flow.md#3146-value-and-type-of-the-expression)。下面的範例展示控制結構直接產生值。

```nabi
let a = if cond { x } else { y };

let b = match v {
    1 => "one",
    2 => "two",
    _ => "other",
};

let c = loop {
    let v = compute();
    if v.converged { break v; }
};

let d = { let a = 1; let b = 2; a + b };
```

這個決定帶來兩個結果。第一，控制結構可以出現在任何運算式位置：函式引數、變數初始化式、其他運算式的一部分。第二，不需要另外的三元形式：沒有 `cond ? x : y` 這種語彙單元；由 `if cond { x } else { y }` 擔任那個角色。

各控制結構回傳之值的確切規則，定義於對應的規格文件。

## 分支的結果型別

運算式的結果型別與接收其值之槽位的型別分別決定。

### 運算式的結果型別

若控制結構的每個分支都產生相同型別的值，該型別就是運算式的結果型別。型別本身已是和型別的分支，以其成員計：若其他每個分支的型別都是該和型別或其成員之一，該和型別就是結果型別，無論哪個分支在前。不會形成新的和型別。型別來自其槽位的分支，例如元素型別仍開放的空 `[]` 或 `Array::zeros(..)`，會取其他分支的型別，無論在它們之前或之後（在 `xs: List<f64>` 之下，`if c { xs } else { [] }` 是 `List<f64>`）。否則，若各分支產生不同型別的值，接收的一方必須提供型別（槽位的明確型別、參數型別或回傳型別），而每個分支都必須與之相容。若接收的一方沒有型別，就是編譯錯誤。自動推論和型別以及自動提升為共同父型別（例如把 `i32` 隱含提升為 `i64` 或 `f64`）都是不允許的。

```
if cond { 1i32 } else { 2i32 }          // result type of the expression: i32
if cond { 1i32 } else { 2.0f64 }        // the receiving side must specify i32 | f64 or similar
if cond { none } else { x }             // x: i32 | none → result type: i32 | none
if cond { "a" } else { v }              // v: i32 | str → result type: i32 | str
if cond { none } else { 1i32 }          // the receiving side must specify i32 | none
if cond { xs } else { [] }              // xs: List<f64> → result type: List<f64>
```

### 和型別要明確寫出；不做提升

這兩種情況被拒絕的理由不同。不自動推論和型別，是為了避免某個分支中的錯誤默默地產生和型別，然後擴散到之後所有的程式碼。想要和型別時，請在接收的一方寫出，例如 `let d: i32 | str = if ...`。相對地，自動提升為父型別會把一種型別轉換成另一種（`i32` → `f64`），可能遺失精度或表示能力，因此即使接收的一方有型別也不允許。

這種區分與[語言與標準函式庫的邊界](../000-meta/004-language-vs-standard-libraries.md)的明確性原則一致。任何需要的轉換都必須以[轉換](../002-types/214-cast-and-conversion.md)的三種形式之一明確寫出：`as`、`.into()`、`.try_into()`。

### 以槽位接收運算式的結果

當變數槽位接收運算式的結果時，槽位的型別遵循[變數宣告](../005-modules/501-variable-declaration.md#517-type-annotations-and-type-inference)的規則。若給出型別，它就成為接收的型別；若省略，槽位的型別由運算式的結果型別推論。

```nabi
// no type annotation → slot is inferred from the expression's result type
let a = if cond { 1i32 } else { 2i32 };          // a: i32
let b = if cond { 1i32 } else { 2.0f64 };        // ERROR: branch type mismatch; type annotation required

// any slot → accepts any combination of branches
let g: any = if cond { 1i32 } else { 2.0f64 };   // OK

// sum type slot → the expression's result must be within the slot's member set
let c: i32 | f64 = if cond { 1i32 } else { 2.0f64 };   // OK
let d: i32 | f64 = if cond { 1i32 } else { "hi" };     // ERROR: str is not in the slot

// single concrete type slot → the expression's result must be exactly that type
let e: i32 = if cond { 1i32 } else { 2i32 };           // OK
let f: i32 = if cond { 1i32 } else { 2.0f64 };         // ERROR: f64 branch does not fit i32
```

### 槽位相容性表

運算式結果型別與接收槽位之間的相容性整理如下。

| 接收槽位                            | 運算式結果型別                              | 是否允許                   |
|-------------------------------------|---------------------------------------------|----------------------------|
| `any`（明確的 `: any`）             | 任何型別                                    | 一律允許                   |
| 和型別 `T1 \| T2 \| ...`              | 結果的成員都在槽位的集合之內                | 允許                       |
| 單一具體型別 `T`                    | 結果恰好是 `T`                              | 允許                       |
| 單一具體型別 `T`                    | 結果有 `T` 以外的成員                       | 拒絕                       |
| 省略型別標註                        | 所有分支的型別相同                          | 允許（推論為該型別）       |
| 省略型別標註                        | 某個分支的和型別容納每個分支的型別          | 允許（推論為該型別）       |
| 省略型別標註                        | 分支型別以其他方式不同                      | 拒絕                       |

### 漸進式與靜態

在這些相容性規則之上，使用者可以在三種模式中選擇。在漸進式模式（`let x: any = ...`）中，槽位是 `any`，可以自由接受任何分支組合；分派由執行時標籤檢查完成，適合快速探索。在靜態和型別模式（`let x: T1 | T2 = ...`）中，槽位是明確的和型別，分支結果必須落在其成員集合之內；可以使用型別收窄與模式比對的窮盡性檢查，適合穩定性與效能。在靜態單一型別模式（`let x: T = ...`，或省略型別的 `let x = ...`）中，槽位是單一型別，每個分支都必須是該型別，能達到最高效能並進行完整的靜態分析。

這三種模式可以在同一份程式碼中自由混用；使用者依變數逐一選擇。

## 區分運算式與陳述式

大多數程式碼行採取下列兩種形式之一。

```nabi
<declaration>;           // a statement: a definition of let / fn / struct / ...
<control-or-expr>;       // a statement that discards the expression's result
```

以 `}` 結尾的運算式，例如 `if` / `match` / 迴圈 / 區塊，在陳述式位置可以省略 `;`（[1.3.4](../001-lexical/103-statement-terminator.md#134-block-like-expression-statements)）。

區塊中的最後一個運算式是例外。

```nabi
{
    let a = 1;
    let b = 2;
    a + b           // no terminator: the value of this block
}
```

只有區塊的最後一個運算式可以不帶終止符，此時它成為整個區塊的值。詳細規則定義於[區塊運算式](309-block-expressions.md)。

## 副作用

運算式在求值期間可能產生副作用，例如 I/O、變數重新指派或外部呼叫。求值順序由運算子優先順序與函式呼叫的求值順序決定，定義於後續文件。

當副作用在單一運算式內形成相依時，該相依遵循明確的順序：由左至右、深度優先。編譯器不會重新排序。指派會先對其值求值，再對其位置求值（[3.4.7](304-assignment.md#347-evaluation-order)）。
