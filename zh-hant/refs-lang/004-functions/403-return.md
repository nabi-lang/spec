# 4.3. 回傳

本節定義函式主體產生其結果值的規則。

## 4.3.1. 核心規則：`return` 不是必要的

函式主體是[區塊運算式](../003-expressions/309-block-expressions.md)中定義的區塊，主體最後一個運算式的值成為函式的回傳值。

```nabi
fn add(x: i32, y: i32) -> i32 {
    x + y                         // returned automatically
}
```

`return` 關鍵字只用於提前離開；在正常路徑上回傳單一值，慣用的寫法是使用最後一個運算式。

## 4.3.2. 明確的 `return`

明確回傳的文法如下：

```
return_expr     ::= "return" expr?
```

```nabi
return expr;
return;
```

`return expr;` 回傳 `expr` 的值並離開函式；`return;` 不帶值離開（也就是回傳 unit）。`return` 運算式是發散的：求值在那一點離開函式。

```nabi
fn early(x: i32) -> i32 {
    if x < 0 { return 0; }        // early exit
    x * 2
}

fn validate(s: str) -> bool {
    if s.len() == 0 { return false; }
    if s.starts_with("#") { return false; }
    true
}
```

## 4.3.3. 回傳型別與主體的一致

[函式宣告](401-function-declaration.md)中定義的回傳型別，必須在主體的每條回傳路徑上一致。回傳路徑包括主體最後一個運算式的型別，以及每個 `return expr;` 中 `expr` 的型別。

宣告的回傳型別是每條路徑的接收型別，與[運算式概觀](../003-expressions/README.md#分支的結果型別)的分支結果規則相同：每條路徑都必須與之相容，而當宣告的型別是能接受每一條路徑的和型別時，型別不同的路徑也沒有問題。

```nabi
fn maybe_int(cond: bool) -> i32 | str {
    if cond { return 42i32; }
    "no"                          // each path fits the declared i32 | str
}
```

## 4.3.4. 省略回傳型別

如[函式宣告](401-function-declaration.md)所示，回傳型別可以省略，此時由主體推論。在 `pub` 或 `pub(pkg)` 函式上，缺少 `->` 則表示 `()`（[4.1.3](401-function-declaration.md#413-省略回傳型別)）。

```nabi
fn double(x: i32) {
    x * 2                          // inferred: i32
}
```

推論出的型別是所有值路徑（`return expr` 與最後的運算式）的型別，它們必須像分支一樣一致：某條路徑的和型別若容納其他每條路徑的型別，就是推論出的型別，無論哪條路徑在前（在 `x: i32 | none` 之下，`return none` 與結尾的 `x` 得到 `i32 | none`）；但不會為型別不同的路徑憑空造出和型別，因此它們是編譯錯誤（E0041），其說明會要求寫出回傳型別（`-> i32 | str`）。若值路徑與不回傳值的路徑混用，或回傳型別相依於自身，也是編譯錯誤（E0042）：遞迴函式需要 `-> T`。若沒有任何路徑到達主體末端或 `return`，推論出的型別是發散型別（4.3.7）。

## 4.3.5. 多個回傳值：元組

要回傳多個值，請使用元組。

```nabi
fn divmod(n: i32, d: i32) -> (i32, i32) {
    let q = n / d;
    let r = n % d;
    (q, r)
}

let (quotient, remainder) = divmod(10i32, 3i32);
```

元組回傳型別遵循[型別運算式](../002-types/201-type-expressions.md)中的元組型別，接收者則使用[解構指派](../003-expressions/315-destructuring.md)的元組解構。

不引入另外的多重回傳關鍵字或語法；元組就足夠了。

## 4.3.6. 捨棄結果的呼叫

不使用函式結果時，寫法如下：

```nabi
compute();                        // result discarded (expression + ;)
_ = compute();                    // intentional discard (explicit)
```

兩種形式都會捨棄結果，但 `_ = ...` 表示使用者刻意忽略它。在工具或 linter 對被忽略的結果發出警告之處，`_ = ...` 形式就是讓警告安靜的確認。

## 4.3.7. 發散的函式

有些函式永遠不會正常完成：控制一律經由 `return`、`throw`、無限迴圈等離開。

```nabi
fn fail_always() throws {
    throw ValueError::new("fatal");
}

fn loop_forever() {
    loop { }
}
```

這種函式的回傳型別不能容納任何值。它寫成 `!`（never），例如 `fn fail() -> ! { ... }`。

對發散函式的呼叫與任何型別相容，就像[區塊運算式](../003-expressions/309-block-expressions.md)中的發散區塊一樣。

```nabi
let x: i32 = if cond {
    42i32
} else {
    try fail_always()             // divergent: OK in an i32 position
};
```

## 4.3.8. `return` 的位置

`return` 運算式可以出現在函式主體的任何地方：區塊、迴圈、`match` 分支等之內。

```nabi
fn find(xs: List<i32>, target: i32) -> i32 | none {
    for x in xs {
        if x == target { return x; }   // returns from the function itself inside the loop
    }
    none
}
```

巢狀在迴圈、`match` 等之內的 `return` 會離開外層函式本身。若只要離開迴圈，請使用[迴圈](../003-expressions/313-loop-expressions.md)的 `break`。

## 4.3.9. 回傳值何時求值

`return expr;` 的 `expr` 在 `return` 運算式求值時完整求值，然後函式離開。

```nabi
fn f() -> i32 {
    return compute_a() + compute_b();   // both evaluated, then the sum is returned
}
```

因此若 `return` 內有帶副作用的運算式，副作用會在回傳之前發生。

## 4.3.10. `return` 與 try/catch

`try` 主體內的 `return` 同樣會離開外層函式。不過，若有[錯誤控制流程](../003-expressions/314-error-control-flow.md)的 `finally` 主體，它在離開的途中一律會求值。

```nabi
fn cleanup_aware() -> i32 {
    try {
        return 42i32;             // finally runs first
    } finally {
        cleanup();
    }
}
```

這個函式先呼叫 `cleanup()`，然後回傳 `42i32`。

## 4.3.11. 閉包與巢狀函式中的 `return`

閉包或巢狀函式中的 `return` 離開的是該閉包或函式本身，而不是外層函式。

```nabi
fn outer() -> i32 {
    let f = |x: i32| -> i32 {
        if x < 0 { return 0; }   // exits only this closure
        x * 2
    };
    f(-5i32);                     // f returns 0 from this call
    100i32                         // outer itself returns 100
}
```

在作為陳述式之呼叫的[尾隨閉包](408-trailing-closure.md)中，單獨的 `return;` 讀起來像是從函式回傳，但它只會結束閉包的那一次呼叫，因此是編譯錯誤（E0020）。要結束閉包的這一次呼叫，請寫帶標籤的 `return@each`（4.3.12）；要離開函式，請以 `for` 迴圈取代該呼叫。說明會附帶修正建議 `return@each`。

```nabi
fn main() {
    each(xs) |x| {
        if x == 2 { return; }     // ERROR: would end only this call
        println(f"{x}");
    }
    for x in xs {
        if x == 2 { return; }     // leaves main
        println(f"{x}");
    }
}
```

在以括號傳入的閉包、呼叫不是陳述式的尾隨閉包（`let n = first(xs) |x| { .. }`），以及巢狀在尾隨閉包內的閉包中，`return` 仍是允許的：它讀作閉包自己的 `return`。

定義巢狀函式與閉包的細節見[閉包](405-closure.md)。

## 4.3.12. 帶標籤的回傳

當最內層的閉包是對 `name` 之呼叫的尾隨閉包時，`return@name` 結束該閉包的一次呼叫。標籤是被呼叫者的最後一個路徑片段或方法的名稱：`each(xs) |x| { .. }` 與 `m::each(xs) |x| { .. }` 取 `@each`，`xs.visit() |x| { .. }` 取 `@visit`。`@` 緊接在 `return` 之後，不帶空格。值的寫法與 `return` 相同：`return@first true`。

```
labeled_return ::= "return" "@" identifier expression?
```

```nabi
fn main() {
    each(xs) |x| {
        if x == 2 { return@each; }    // ends this call; 3 still prints
        println(f"{x}");
    }
    let n = first(xs) |x| { if x > 1 { return@first true; } false };
}
```

沒有指稱最內層閉包的標籤是編譯錯誤（E0020）：閉包不能離開包圍它的另一個閉包，而巢狀閉包、`async` / `yield` 區塊與巢狀函式各自開始自己的 `return` 範圍。

```nabi
each(xs) |x| {
    let g = || { return@each; };      // ERROR: the innermost closure is g
}
```
