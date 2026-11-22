# 3.14. 错误控制流

`throw`、`try`、`catch` 和 `finally` 是用于抛出错误、标记错误可能流经之处、以及接收和处理错误的控制流。后缀 `?` 把 `T | none` 值中的 `none` 提前返回给调用方。

```
throw_stmt     ::= "throw" expr ";"
try_expr       ::= ("try" | "try?" | "try!") expr
catch_expr     ::= expr catch_clause+ finally_clause?
                 | expr finally_clause
catch_clause   ::= "catch" pattern? block            // struct patterns also allowed: catch E { f, .. } { .. }
finally_clause ::= "finally" block
question_expr  ::= expr "?"
```

`catch` 和 `finally` 是上下文关键字，只在表达式或块之后的后缀子句位置是关键字（[1.5.4](../001-lexical/105-keywords.md#154-上下文关键字)）。可能抛出错误的函数的签名（`throws`、`rethrows`）在 [`throws` 签名](../004-functions/414-throws.md)中定义。本文档讨论表达式和语句层面的控制流。

## 3.14.1. `throw`：抛出错误

```nabi
throw err;
```

由 `throw` 关键字后跟任意表达式构成的语句；表达式的结果值作为错误值被抛出。被抛出的错误由包围该语句的最近的 `catch` 表达式处理；如果未被处理，它就成为外层函数的错误结果。此时外层函数的 `throws` 签名必须接受该错误类型（[4.14.5](../004-functions/414-throws.md#4145-throws-与函数体的一致)）。

`throw` 语句之后控制流不再继续，因此以 `throw` 语句结尾的块是发散的。

被抛出的 `str` 会被包装：当 `msg: str` 时，`throw msg` 抛出 `ValueError::new(msg)`（[标准库 2.14](../../refs-standard-libraries/200-core/214-error.md)）。外层函数必须像接受其他被抛出的错误一样接受 `ValueError`：无类型的 `throws` 接受它，带类型的 `throws (E)` 必须列出它（`throws (E | ValueError)`），否则就是 E0062，并附带说明这一点的帮助信息。在 `str` 作为 `str` 被接收之处，它原样被抛出：即函数声明了 `throws (str)`（或含有 `str` 的和类型），或者外层有 `catch e: str` 分支接收它。

```nabi
fn check(age: i32) throws -> i32 {
    if age < 0 { throw f"age {age} is negative"; }   // a ValueError
    age
}
```

```nabi
let x = if cond { 42i32 } else { throw ValueError::new("not allowed"); };
// x: i32 (else branch is divergent)
```

## 3.14.2. `try`：标记调用点

```nabi
let n = try parse(s);
```

`try` 是前缀表达式运算符。对可能抛出错误的函数（带 `throws` 签名的函数）的调用必须位于被 `try` 覆盖的表达式之内；否则是编译错误（E0062）。这条规则让错误可能流经的每一处在代码中都清晰可见。

`try e` 的值和类型与 `e` 相同。如果 `e` 内部的某个可能抛出错误的调用抛出了错误，由最近的 `catch` 处理；如果没有 `catch` 或没有匹配的 `catch`，错误传播到外层函数。

### 覆盖范围

`try` 覆盖其右侧的整个表达式。一个 `try` 可以同时标记多个调用。

```nabi
try a(b());                 // covers both a and b
try f().g().h();            // covers the whole chain
try {                       // a block is also an expression, so every call in it is covered
    let cfg = load(path);
    apply(cfg);
};
```

`try` 的操作数向右延伸到表达式结束之处，在 `catch` / `finally` 子句、语句终止符 `;`、右括号或逗号之前停止。因此 `try a + b()` 是 `try (a + b())`，而 `try parse(s) catch _ { 0 }` 中的 `catch` 附着于整个 `try parse(s)`。确切的结合规则在[优先级与结合性](302-precedence-and-associativity.md)中定义。

当 `try` 作为二元运算符的右操作数或出现在函数实参位置时，它只覆盖该操作数。

唯一的例外是 `for` 操作数：`try for pat in xs { body }` 覆盖可迭代值 `xs` 和循环的每一步，而不覆盖 `body`（[3.13.3](313-loop-expressions.md#3133-for基于迭代器的重复)）。主体中可能抛出错误的调用需要自己的 `try`。

```nabi
let total = base + try fetch_extra();
log(try describe(item));
```

### 与 `await` 一起使用

`async` 且可能抛出错误的函数的错误在 `await` 点交付，因此 `await` 表达式由 `try` 覆盖（[4.14.10](../004-functions/414-throws.md#41410-throws-对和类型签名的影响)）。生成器的错误同样在 `next` 点接收。

```nabi
let data = try await fetch(url);
```

## 3.14.3. `try?` 与 `try!`

`try` 的这两种变体不把错误传播到外层函数，而是就地把它转换为另一种形式。

| 形式     | `e: T` 时的结果类型      | 出错时                            |
|----------|--------------------------|-----------------------------------|
| `try e`  | `T`                      | 最近的 `catch`，或者传播          |
| `try? e` | `T?`                     | 丢弃错误并产生 `none`             |
| `try! e` | `T`                      | 触发 panic（3.14.8）              |

```nabi
let port = try? env::var("PORT") ?? "8080";      // (try? env::var("PORT")) ?? "8080"
let cfg = try! load_builtin_config();      // failure is a program bug
```

`try? e` 的类型是以 `T` 为泛型项的 `T?`（[2.6.4 泛型和类型的层](../002-types/206-sum-types.md#泛型和类型的分层)）。如果 `T` 不含 `none`，那就是扁平的 `T | none`。如果 `T` 已经含有 `none`，结果是嵌套的 `(T | none)?`，因此被抛出的错误（外层的 `none`）与被返回的 `none`（内层的那个）保持区分。`??`、`?`、`is none` 和模式只作用于外层。

```nabi
fn lookup(k: str) throws (IoError) -> i32 | none { .. }

let r = try? lookup("a");             // (i32 | none)?
match r {
    none => print("lookup failed"),   // outer: an error was thrown
    v => print(f"{v:?}"),             // inner: i32 | none from lookup
}
let v = try? lookup("a") ?? none;     // i32 | none: error and absence merged
```

`try?` / `try!` 中的 `?` / `!` 必须紧跟在 `try` 之后。`try !x` 是 `try (!x)`。由于 `try?` 和 `try!` 不传播错误，外层函数不必是 `throws`。它们的覆盖范围与 `try` 相同，只是在 `??` 之前结束：`try? f() ?? d` 是 `(try? f()) ?? d`，出错时得到 `d`，这正是书写默认值时想要的读法。要在 `f() ?? d` 出错时得到 `none`，应加括号分组：`try? (f() ?? d)`。对于 `T | none` 结果，`??` 仍然只作用于外层，如上例所示（`try? lookup("a") ?? none` 把错误与缺失的值合并）。

## 3.14.4. `catch`：后缀处理表达式

```nabi
let n = try parse(s) catch e: ParseError { 0 } catch _ { -1 };
```

`catch` 子句是附着于其左侧表达式的后缀子句。如果对左侧表达式求值时产生错误，就自上而下检查 `catch` 子句，并对第一个匹配子句的块求值。如果左侧表达式正常完成，不运行任何 `catch` 块。左侧表达式的错误是由用 `try` 标记的可能抛出错误的调用以及 `throw` 语句产生的错误。

如果没有子句匹配，错误会再次传播，此时外层函数必须 `throws` 该错误类型。

### 块形式

`try` 覆盖一个块、`catch` / `finally` 附着于该表达式的形式，按同样的规则解释。

```nabi
try {
    body
} catch e {
    handler
}
```

### 模式与类型匹配

```nabi
try fetch(url)
catch e: TimeoutError { retry() }
catch HttpError { code, msg } { fail(code, msg) }
catch e: any Error { log(f"{e}"); fallback() }
```

`catch` 之后可以跟[模式](311-patterns.md)中的任何模式形式。`e: TimeoutError` 是类型检查模式，在匹配的分支内 `e` 被收窄为该类型。`catch e: A | B` 把捕获的错误绑定为和类型 `A | B`（错误和类型的一个子集）。无类型 `throws` 的错误是 [`any Error`](../002-types/202-any-slot.md) 值，因此要用一个名字接收所有错误，使用 `catch e: any Error` 或 `catch e`。按照 [trait 不能作类型](../002-types/212-trait-as-type-banned.md)的规则，写裸 trait 名 `catch e: Error` 是编译错误（E0071）。

省略模式的 `catch { ... }` 与 `catch _ { ... }` 相同。

### 穷尽性

如果 `catch` 子句覆盖了左侧表达式可能抛出的每种错误类型，`catch` 表达式就不再抛出错误。对于带类型 `throws (A | B)` 的调用，覆盖 `A` 和 `B` 即可；对于无类型的 `throws`，必须有一个接受所有值的子句，例如 `catch e`、`catch _` 或 `catch e: any Error`。

```nabi
fn parse_or_zero(s: str) -> i32 {          // non-throwing
    try parse(s) catch _ { 0 }             // covers every error
}
```

### 不可达的 `try` 与 `catch` 的警告

在以下情况下，编译器发出警告（E0096）：

- 没有覆盖任何可能抛出错误的调用的 `try`
- 任何错误都无法到达的 `catch` 子句或分支

触发警告的子句仍会进行类型检查。

## 3.14.5. `finally`

```nabi
try {
    body
} finally {
    cleanup
}
```

它也可以与 `catch` 组合：

```nabi
try {
    body
} catch e: IoError {
    handler
} finally {
    cleanup
}
```

在离开左侧表达式的每一种情况下，`finally` 块都会运行：正常完成、由 `catch` 处理、未处理错误的传播，以及通过 `return` / `break` / `continue` 退出。因此它适合用于释放资源、释放锁和恢复临时状态。如果 `catch` 块抛出新错误，或者 `finally` 块内抛出错误，新错误优先。

离开 `finally` 块本身的 `return`、`break` 或 `continue` 是编译错误（E0059）：它会丢弃正在传递的错误。应把跳转移到 `try` 表达式之后。跳转到 `finally` 块内部的循环或带标签的块，或者其中闭包里的 `return`，仍停留在块内，是允许的。

panic 和致命错误（3.14.8）会终止进程，因此 `finally` 不会运行。

## 3.14.6. 表达式的值与类型

`try`、`catch` 和 `finally` 都是表达式。

```nabi
let x = try parse(input) catch _ { default };
```

`catch` 表达式的值，在左侧表达式正常完成时是左侧表达式的值，在捕获到错误时是处理块的值。左侧表达式和每个处理块必须类型相同（[表达式概述](README.md)中的分支结果规则）；以 `throw` 或 `return` 结尾的处理块是发散的，不参与类型检查。`finally` 块不产生值，也不影响表达式的结果。

## 3.14.7. 被抛出的值与函数签名

`throw err` 中的 `err` 是任意表达式。带类型 `throws (E)` 的函数可以抛出对应于 `E` 各成员的值；无类型 `throws` 的函数是 `throws (any Error)` 的简写，可以抛出实现了标准库 `Error` trait 的值。被抛出的 `str` 是 `ValueError`（3.14.1）。

```nabi
fn parse(s: str) throws -> i32 { ... }              // untyped, any Error
fn parse(s: str) throws (TypeError) -> i32 { ... }  // single type
fn parse(s: str) throws (A | B | C) -> i32 { ... }  // sum type
```

签名中没有 `throws` 的函数不会有任何错误离开它。当 `throws (T)` 中的 `T` 是和类型时，它就是该函数可能抛出的错误的封闭集合，调用方用它来编写 `catch` 子句并检查穷尽性。

函数签名和 `throws` 的含义在 [`throws` 签名](../004-functions/414-throws.md)中详细定义。

## 3.14.8. panic 与致命错误

与 `throws` 错误不同，以下情况不作为值交付，也不能用 `catch` 捕获。

- panic。包括对标准库 `panic` 的调用、`try!` 失败、除以零、整数溢出、边界检查失败等。
- 致命错误。无法继续执行的状态，例如 `StackOverflowError`、`OutOfMemoryError` 和 `InternalError`。

两种情况下，运行时都会向标准错误（stderr）打印一条消息，并以退出码 `101` 终止进程。哪些情况属于 panic 或致命错误，其确切列表由标准库定义。

失败的 `try!` 在 `try!` 所在位置触发 panic。其消息以未捕获错误从 `main` 传出时的方式显示该错误（[4.16.4](../004-functions/416-entry-point.md)：显示其 `Display`，没有则显示其 `Debug`，再没有则显示其类型名），然后显示抛出它的 `throw` 的位置，以及它到达 `try!` 所经过的位置：

```text
thread 'main' panicked at src/main.nabi:12:13:
try! failed: no such key `port`
  raised at src/config.nabi:30:9
  through src/main.nabi:12:18
```

当错误就是在该位置本身抛出时，或者 `throw` 未知时，省略 `through` 行：此时 `raised at` 给出错误到达 `try!` 所经过的位置。`catch` 内的 `throw` 会重新抛出错误，因此显示的是它的位置。由子句不匹配的 `catch` 继续传递的错误保留其 `throw` 的位置。标准库抛出的错误，或者作为值传给另一个线程或任务的错误，不指明任何 `throw`；`try_block_on`（因而也包括 `async` main）保留其主体中 `throw` 的位置。

## 3.14.9. 求值顺序

`catch` 表达式和 `finally` 按以下顺序求值：

1. 对左侧表达式求值。
2. 如果左侧表达式产生错误，找到匹配的 `catch` 子句并对其块求值。
3. 如果有 `finally` 块，无论以何种方式离开左侧表达式，都对它求值。
4. 如果没有 `catch` 子句匹配，先运行 `finally`，再把错误向外传播。

对 `throw` 语句求值时，先对其表达式求值，然后抛出错误。

## 3.14.10. `?` 运算符：提前返回 `none`

`?` 是后缀一元运算符，它解开 `T | none` 值，如果值为 `none`，立即从外层函数返回 `none`。

```nabi
fn first_word_len(text: str | none) -> usize | none {
    let t = text?;                      // none → return none immediately
    let w = t.split(" ").next()?;
    w.len()
}
```

当 `e: T | none` 时，`e?` 表达式的值为 `T`。如果 `e` 为 `none`，外层函数执行 `return none`。因此外层函数的返回类型必须包含 `none` 成员（`_ | none`）；否则是编译错误（E0064）。由于是后缀形式，它可以自然地放在方法链末尾。

### 脱糖规则

`?` 由编译器直接展开，不经过 trait。

```
expr?
```

展开为：

```nabi
match expr {
    none => return none,
    v => v,
}
```

### 适用范围

`?` 只适用于 `T | none`。它不是传播错误的手段，不能用于标准库的 `Result` 值或可能抛出错误的调用。错误传播用 `throws` 签名和 `try` 标记。

```nabi
fn load() -> Result<Config, IoError> {
    let raw = read_result(path)?;     // ERROR (E0064): `?` is only for `T | none`.
                                      //   Use `throws` and `try` for error propagation.
    ...
}
```

闭包主体内的 `?` 作用于闭包的返回类型，与外层函数的返回类型无关。

```nabi
let lens = users.iter().map(|u| -> usize | none { let nick = u.nickname?; nick.len() });
```

要用替代值填补 `none`，或者让成员访问穿过 `none`，使用[可选链与空值合并](308-optional-chaining-and-coalescing.md)中的 `?.` 和 `??`。
