# 4.10. 嵌套函数

函数可以定义在另一个函数的函数体内。

## 4.10.1. 形式

```nabi
fn outer(x: i32) -> i32 {
    fn inner(y: i32) -> i32 {
        y + 1
    }

    inner(x) + inner(x * 2)
}
```

把 `fn` 关键字放在外层函数的函数体内。定义和使用的规则与[函数声明](401-function-declaration.md)中模块级函数的规则相同。

## 4.10.2. 作用域

嵌套函数只在定义它的块作用域内可见。

```nabi
fn outer() {
    fn helper() { ... }
    helper();                     // OK
}

helper();                          // ERROR (E0027): not visible outside outer
```

一旦离开该块，嵌套函数的名字就不再可见。

## 4.10.3. 捕获外层变量（不捕获）

嵌套函数不捕获外层函数的变量。

```nabi
fn outer(n: i32) {
    fn inner() {
        let x = n;                // ERROR (E0027): n is not visible
    }
    inner();
}
```

这表明嵌套函数是一个干净的函数单元。需要依赖外层变量的有状态函数时，请使用[闭包](405-closure.md)。

```nabi
fn outer(n: i32) {
    let inner = |x| x + n;        // OK: closures can capture
    inner(5);
}
```

由此得到以下划分：

- 嵌套函数看不到外层变量；除了作用域受限之外，它与模块级函数相同。
- 闭包可以捕获（见[捕获语义](406-capture-semantics.md)）。

嵌套函数可以看到它所在的块及外围各块中的项和 `use` 声明，包括 `const`（[3.9.1](../003-expressions/309-block-expressions.md#391-form)）；只有这些块的局部变量、参数和泛型参数是它无法触及的。

## 4.10.4. 调用时的行为

调用嵌套函数遵循与调用模块级函数相同的规则。它通过函数地址直接调用，因此没有捕获，也没有开销。

## 4.10.5. 定义顺序

在同一个块内，嵌套函数可以在定义之前或之后调用。两种形式都允许：

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

提升（hoisting）使它在整个块内可见，因此可以从任何位置调用。

## 4.10.6. 在嵌套函数与闭包之间选择

| 属性                             | 嵌套函数               | 闭包                     |
|----------------------------------|------------------------|--------------------------|
| 定义形式                         | `fn name(...) { ... }` | `\| ... \| body`         |
| 捕获外层变量                     | 否                     | 是                       |
| 名字                             | 有                     | 无                       |
| 作为实参传给函数                 | 可以（按名字）         | 可以                     |
| 无状态辅助函数                   | 合适                   | 可以但多余               |
| 有状态回调                       | 不合适                 | 合适                     |

不需要外层变量且名字有助于可读性时，选择嵌套函数；需要外层变量或匿名就足够时，使用闭包。

## 4.10.7. 嵌套函数的可见性

嵌套函数只在外层函数的函数体内可见，因此 `pub` 等可见性关键字没有意义。

```nabi
fn outer() {
    pub fn helper() { ... }       // ERROR (E0016): pub is meaningless on a nested function
}
```

可见性只适用于模块级的项。在块中声明的每一个项都是如此（[3.9.1](../003-expressions/309-block-expressions.md#391-form)）。

## 4.10.8. 同名的嵌套函数与外层变量

```nabi
fn outer() {
    let helper = 42;
    fn helper() { ... }            // ERROR (E0038): variable and function with the same name
}
```

同一个块中同名的变量和函数相互冲突，这是编译错误。

## 4.10.9. 多层嵌套

函数可以嵌套到任意深度。

```nabi
fn outer() {
    fn middle() {
        fn inner() { ... }
        inner();
    }
    middle();
}
```

嵌套函数在它的整个块中可见（被提升）。它可以看到自身所在块以及每一个外围块中的嵌套函数、其他项和 `use` 声明，但看不到其中任何块的局部变量。块中的 `use` 会遮蔽外层同名的嵌套函数。

## 4.10.10. 嵌套函数适用的场景

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

它们很适合作为大型函数的内部辅助函数。对于不必暴露给其他模块、且与外层函数上下文绑定的辅助逻辑，嵌套函数非常合适。
