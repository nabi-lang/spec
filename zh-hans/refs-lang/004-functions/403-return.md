# 4.3. 返回

本节定义函数主体产生结果值的规则。

## 4.3.1. 核心规则：不要求写 `return`

函数主体是[块表达式](../003-expressions/309-block-expressions.md)中定义的块，主体最后一个表达式的值即为函数的返回值。

```nabi
fn add(x: i32, y: i32) -> i32 {
    x + y                         // returned automatically
}
```

`return` 关键字只用于提前退出；在正常路径上返回单个值，惯用写法是写成最后一个表达式。

## 4.3.2. 显式 `return`

显式返回的语法如下：

```
return_expr     ::= "return" expr?
```

```nabi
return expr;
return;
```

`return expr;` 退出函数并返回 `expr` 的值；`return;` 不带值退出（即返回 unit）。`return` 表达式是发散的：求值在该处离开函数。

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

## 4.3.3. 返回类型与主体一致

[函数声明](401-function-declaration.md)中定义的返回类型必须在主体的每条返回路径上一致。返回路径是指主体最后一个表达式的类型，以及每个 `return expr;` 中 `expr` 的类型。

声明的返回类型是每条路径的接收类型，与[表达式概述](../003-expressions/README.md#分支结果类型)中分支结果的规则相同：每条路径都必须与之兼容；当声明的类型是能容纳各条路径类型的和类型时，路径类型可以不同。

```nabi
fn maybe_int(cond: bool) -> i32 | str {
    if cond { return 42i32; }
    "no"                          // each path fits the declared i32 | str
}
```

## 4.3.4. 省略返回类型

如[函数声明](401-function-declaration.md)所示，返回类型可以省略，此时从主体推断。在 `pub` 或 `pub(pkg)` 函数上，缺少 `->` 则表示 `()`（[4.1.3](401-function-declaration.md#413-省略返回类型)）。

```nabi
fn double(x: i32) {
    x * 2                          // inferred: i32
}
```

推断出的类型是所有值路径（`return expr` 和最终表达式）的类型，它们必须像分支一样一致：如果某条路径的和类型容纳了其他每条路径的类型，它就是推断出的类型，无论哪条路径在前（`x: i32 | none` 时，尾部 `x` 与 `return none` 并存得到 `i32 | none`）；但不会为类型不同的路径凭空构造和类型，所以这种情况是编译错误（E0041），其帮助信息要求写明返回类型（`-> i32 | str`）。值路径与不返回值的路径混用，或者返回类型依赖于自身，也是编译错误（E0042）：递归函数需要写 `-> T`。如果没有路径到达主体末尾或 `return`，推断出的类型是发散类型（4.3.7）。

## 4.3.5. 多个返回值：元组

要返回多个值，使用元组。

```nabi
fn divmod(n: i32, d: i32) -> (i32, i32) {
    let q = n / d;
    let r = n % d;
    (q, r)
}

let (quotient, remainder) = divmod(10i32, 3i32);
```

元组返回类型遵循[类型表达式](../002-types/201-type-expressions.md)中的元组类型，接收方使用[解构赋值](../003-expressions/315-destructuring.md)中的元组解构。

不引入单独的多返回值关键字或语法；元组就足够了。

## 4.3.6. 丢弃结果的调用

不使用函数结果时，写法如下：

```nabi
compute();                        // result discarded (expression + ;)
_ = compute();                    // intentional discard (explicit)
```

两种形式都会丢弃结果，但 `_ = ...` 表示用户有意忽略它。当工具或 lint 工具对被忽略的结果发出警告时，`_ = ...` 形式起到确认作用，可以抑制该警告。

## 4.3.7. 发散函数

有些函数永远不会正常完成：控制总是通过 `return`、`throw`、无限循环等离开。

```nabi
fn fail_always() throws {
    throw ValueError::new("fatal");
}

fn loop_forever() {
    loop { }
}
```

这类函数的返回类型不能容纳任何值。它写作 `!`（never），如 `fn fail() -> ! { ... }`。

对发散函数的调用与任何类型兼容，与[块表达式](../003-expressions/309-block-expressions.md)中的发散块相同。

```nabi
let x: i32 = if cond {
    42i32
} else {
    try fail_always()             // divergent: OK in an i32 position
};
```

## 4.3.8. `return` 的位置

`return` 表达式可以出现在函数主体的任何位置：块、循环、`match` 分支等内部。

```nabi
fn find(xs: List<i32>, target: i32) -> i32 | none {
    for x in xs {
        if x == target { return x; }   // returns from the function itself inside the loop
    }
    none
}
```

嵌套在循环、`match` 等内部的 `return` 退出外层函数本身。只想退出循环时，使用[循环](../003-expressions/313-loop-expressions.md)中的 `break`。

## 4.3.9. 返回值的求值时机

`return expr;` 的 `expr` 在求值 `return` 表达式时被完整求值，随后函数退出。

```nabi
fn f() -> i32 {
    return compute_a() + compute_b();   // both evaluated, then the sum is returned
}
```

因此，如果 `return` 中有带副作用的表达式，副作用恰好在返回之前发生。

## 4.3.10. `return` 与 try/catch

`try` 主体中的 `return` 同样退出外层函数。不过，如果有[错误控制流](../003-expressions/314-error-control-flow.md)中的 `finally` 主体，它在退出途中总会被求值。

```nabi
fn cleanup_aware() -> i32 {
    try {
        return 42i32;             // finally runs first
    } finally {
        cleanup();
    }
}
```

这个函数先调用 `cleanup()`，再返回 `42i32`。

## 4.3.11. 闭包与嵌套函数中的 `return`

闭包或嵌套函数中的 `return` 退出该闭包或函数本身，而不是外层函数。

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

在作为语句的调用的[尾随闭包](408-trailing-closure.md)中，单独的 `return;` 读起来像是从函数返回，但它只会结束闭包的这一次调用，所以是编译错误（E0020）。要结束闭包的这一次调用，写带标签的返回 `return@each`（4.3.12）；要离开函数，用 `for` 循环代替这个调用。帮助信息提供 `return@each`，并附带自动修复。

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

在括号中传入的闭包里、在调用不是语句的尾随闭包里（`let n = first(xs) |x| { .. }`），以及在嵌套于尾随闭包内的闭包里，`return` 仍然允许：它读作闭包自己的返回。

嵌套函数和闭包的定义详见[闭包](405-closure.md)。

## 4.3.12. 带标签的返回

当最内层闭包是对 `name` 的调用的尾随闭包时，`return@name` 结束该闭包的一次调用。标签是被调用者路径的最后一段或方法名：`each(xs) |x| { .. }` 和 `m::each(xs) |x| { .. }` 用 `@each`，`xs.visit() |x| { .. }` 用 `@visit`。`@` 紧跟在 `return` 之后，中间没有空格。值的写法与 `return` 相同：`return@first true`。

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

标签指向的不是最内层闭包时，是编译错误（E0020）：闭包不能离开包围它的另一个闭包，而嵌套闭包、`async` / `yield` 块和嵌套函数各自开始 `return` 的新作用域。

```nabi
each(xs) |x| {
    let g = || { return@each; };      // ERROR: the innermost closure is g
}
```
