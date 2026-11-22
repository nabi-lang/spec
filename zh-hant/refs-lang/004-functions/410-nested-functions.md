# 4.10. 巢狀函式

函式可以定義在另一個函式的主體內。

## 4.10.1. 形式

```nabi
fn outer(x: i32) -> i32 {
    fn inner(y: i32) -> i32 {
        y + 1
    }

    inner(x) + inner(x * 2)
}
```

把 `fn` 關鍵字放在外層函式的主體內。定義與使用的規則與[函式宣告](401-function-declaration.md)中模組層級的函式相同。

## 4.10.2. 作用域

巢狀函式只在定義它的區塊作用域內可見。

```nabi
fn outer() {
    fn helper() { ... }
    helper();                     // OK
}

helper();                          // ERROR (E0027): not visible outside outer
```

一旦離開該區塊，巢狀函式的名稱就不再可見。

## 4.10.3. 擷取外部變數（無）

巢狀函式不擷取外層函式的變數。

```nabi
fn outer(n: i32) {
    fn inner() {
        let x = n;                // ERROR (E0027): n is not visible
    }
    inner();
}
```

這讓巢狀函式明確地成為乾淨的函式單位。需要依賴外部變數的有狀態函式時，請使用[閉包](405-closure.md)。

```nabi
fn outer(n: i32) {
    let inner = |x| x + n;        // OK: closures can capture
    inner(5);
}
```

這帶來下列區分：

- 巢狀函式看不到外部變數；除了作用域受限之外，它與模組層級的函式相同。
- 閉包可以擷取（見[擷取語意](406-capture-semantics.md)）。

巢狀函式看得到其區塊及外圍區塊的項目與 `use` 宣告，包括 `const`（[3.9.1](../003-expressions/309-block-expressions.md#391-form)）；只有它們的區域變數、參數與泛型參數是無法觸及的。

## 4.10.4. 呼叫時的行為

呼叫巢狀函式遵循與呼叫模組層級函式相同的規則。它透過函式位址直接呼叫，因此沒有擷取，也沒有額外負擔。

## 4.10.5. 定義順序

在同一個區塊內，巢狀函式可以在其定義之前或之後呼叫。兩種形式都是允許的：

```nabi
fn outer() {
    helper();                     // call before definition
    fn helper() { ... }
}

fn outer() {
    fn helper() { ... }
    helper();                     // call after definition
}
```

提升（hoisting）讓它在整個區塊中可見，因此可以從任何位置呼叫。

## 4.10.6. 在巢狀函式與閉包之間選擇

| 性質                             | 巢狀函式               | 閉包                     |
|----------------------------------|------------------------|--------------------------|
| 定義形式                         | `fn name(...) { ... }` | `\| ... \| body`         |
| 擷取外部變數                     | 否                     | 是                       |
| 名稱                             | 有                     | 無                       |
| 作為引數傳給函式                 | 可以（以名稱）         | 可以                     |
| 無狀態的輔助函式                 | 適合                   | 可以但過度               |
| 有狀態的回呼                     | 不適合                 | 適合                     |

當不需要外部變數、而名稱有助於可讀性時，選擇巢狀函式；當需要外部變數或匿名就足夠時，使用閉包。

## 4.10.7. 巢狀函式的可見性

巢狀函式只在外層函式的主體內可見，因此 `pub` 等可見性關鍵字沒有意義。

```nabi
fn outer() {
    pub fn helper() { ... }       // ERROR (E0016): pub is meaningless on a nested function
}
```

可見性只適用於模組層級的項目。在區塊中宣告的每個項目都是如此（[3.9.1](../003-expressions/309-block-expressions.md#391-form)）。

## 4.10.8. 同名的巢狀函式與外部變數

```nabi
fn outer() {
    let helper = 42;
    fn helper() { ... }            // ERROR (E0038): variable and function with the same name
}
```

同一個區塊中同名的變數與函式會衝突，是編譯錯誤。

## 4.10.9. 多層巢狀

函式可以巢狀到任意深度。

```nabi
fn outer() {
    fn middle() {
        fn inner() { ... }
        inner();
    }
    middle();
}
```

巢狀函式在其整個區塊中可見（被提升）。它看得到其自身區塊以及每個外圍區塊的巢狀函式、其他項目與 `use` 宣告，但看不到其中任何一個的區域變數。區塊中的 `use` 會隱藏外層同名的巢狀函式。

## 4.10.10. 巢狀函式適用的場合

```nabi
fn parse(input: str) -> AST {
    fn parse_expr(s: str) -> Expr { ... }
    fn parse_stmt(s: str) -> Stmt { ... }

    let result = AST::new();
    while !input.is_empty() {
        result.add(parse_stmt(input));
    }
    result
}
```

它們很適合作為大型函式的內部輔助函式。對於不需要公開給其他模組、且與外層函式的上下文緊密相關的輔助邏輯，巢狀函式很合適。
