# 3.9. 块表达式

用 `{ ... }` 括起的一组语句和表达式，是 Nabi 中最基本的表达式单元。本文档定义块的词法形式及其值的求值规则。

## 3.9.1. 形式

块的语法如下：

```
block       ::= label? "{" stmt* tail_expr? "}"
label       ::= label_token ":"              // label_token: 1.2
stmt        ::= expr ";"
              | block_like_expr ";"?
              | let_stmt
              | item_decl
item_decl   ::= fn_decl | const_decl | use_stmt | struct_decl | enum_decl
              | union_decl | trait_decl | impl_block | type_alias | static_decl
type_alias  ::= "type" identifier generics? where_clause? "=" type_expr ";"
block_like_expr ::= block | if_expr | match_expr
              | loop_expr       // loop, while, for (3.13)
              | unsafe_block    // 4.15
tail_expr   ::= expr            // no trailing ";"
```

块中可以声明 `fn`（[4.10](../004-functions/410-nested-functions.md)）、`const`（[5.2.2](../005-modules/502-const.md#522-允许的位置)）、`use`、`struct`、`enum`、`union`、`trait`、`impl`、`type` 和 `static` 项。`mod` 和 `extern` 项只能在模块层级声明（[5.4.10](../005-modules/504-modules.md#5410-模块声明可以出现的位置)、[5.9.8](../005-modules/509-extern-block.md#598-extern-块本身的位置)）；在块中声明它们是编译错误（E0017）。

块中声明的项遵循以下规则：

- **作用域**。项的名字在整个块中可见，包括其声明之前和之后，以及该块中声明的各项内部。在块内，它遮蔽块外声明的同名、同命名空间的项；内层块中的 `use` 又会遮蔽它。在块外不能指称它。同一块中两个同名、同命名空间的项，或一个项与其名字的 `use`，是编译错误（E0026 和 E0032，与模块层级相同）。
- **它能看到什么**。块中的项能看到其模块的项，以及其所在块和外围各块的项与 `use` 声明。它不能使用外围代码的局部变量或参数（`let`、函数或闭包参数、模式绑定）或泛型参数；这样做是编译错误（E0039）。值通过字段和参数传给它；类型通过它自己的泛型参数传给它。块中的 glob `use p::*` 会被该块中声明的各项的头部（字段类型、签名、`const` / `static` 初始化器、别名、变体）看到，与其主体一样，优先级与函数体中相同：模块自身的项优先于 glob，内层块的 glob 优先于外层块的，而泛型参数优先于两者。
- **一个项**。块中的项无论其所在块运行多少次，都是同一个项：泛型函数中的项不随函数的参数泛型化；块中的 `static` 是一个槽，只初始化一次；块中的 `const` 只有一个值。
- **一致性**。块中的 `impl` 与同一模块的模块层级 `impl` 一样参与一致性和孤儿规则（[2.9.5](../002-types/209-impl.md#295-孤儿规则)）；它的方法和 trait 实现在任何地方都成立。它可以为外部类型实现外部 trait。
- **可见性**。块中的项没有自己的可见性：在其上写 `pub` 是编译错误（E0016）。块中声明的类型的字段和方法在其模块内遵循通常规则。
- **名字**。`type_name`、`Debug` 和调试信息按书写形式命名块中的项（`S`）；诊断也如此，并在另有同名项时加上声明它的函数（`f::S`）。

```nabi
struct S { a: i32; }

fn f() -> i32 {
    use core::fmt::Debug;
    const SCALE: i32 = 2;
    #[derive(Debug)]
    struct S { x: i32; }               // hides the module's `S` in this block
    impl S { fn scaled(self) -> i32 { self.x * SCALE } }   // OK: `SCALE` is an item
    S { x: 1 }.scaled()
}

fn g(n: i32) {
    struct P { v: i32 = n; }           // ERROR: `n` is a local of `g`
}
```

示意如下：

```nabi
{
    statement1;
    statement2;
    ...
    last_expr     // no terminator, becomes the block's value
}
```

主体可以包含零个或多个语句，以及零个或一个最后表达式。如果最后表达式不带终止符，它的值就是块的值；如果带终止符（`expr;`），块的值为单元值。

```nabi
let a = {
    let x = 1;
    let y = 2;
    x + y               // block value = 3
};

let b = {
    let x = 1;
    x + 2;              // terminator present → result discarded
                        // block value = unit
};

let c = {
    let x = 1;
    return x;           // return exits the function itself (block value never reached)
};
```

## 3.9.2. 用途

块可以出现在任何表达式可以出现的地方。

```nabi
let v = { ... };                  // variable initializer
fn_call({ ... }, other);          // function argument
if { ... > 0 } { ... }            // condition (rare but allowed)
```

其他控制结构（`if`、`match`、`loop`、函数体）的主体也是块。

## 3.9.3. 作用域

块创建一个新的作用域。块内用 `let` 声明的变量只在该块中可见，并在退出块时被丢弃。外层变量在块内仍然可见，除非被内层同名的 `let` 遮蔽。

内层同名的 `let` 是一个独立的绑定，不影响外层绑定。确切规则定义于[变量声明](../005-modules/501-variable-declaration.md)。

## 3.9.4. 空块

```nabi
{}
```

空块的值为单元值。

## 3.9.5. 块中的最后表达式与终止符

以下两种输入的求值结果不同。

```nabi
let a = { x };          // a is the value of x
let b = { x; };         // b is unit
```

不带终止符结束的表达式成为块的值，而带终止符结束的表达式的结果被丢弃，块的值为单元值。这一区别与[语句终止符](../001-lexical/103-statement-terminator.md)的规则一致。

- 块中间不带 `;` 的类块语句的类型必须为 `()`。
- `}` 之前的最后一条语句可以省略 `;`，含义与写上它相同（[1.3.4](../001-lexical/103-statement-terminator.md#134-类块表达式语句)）：`{ x += 1 }` 和 `{ let y = x }` 的值为单元值，与 `{ x += 1; }` 和 `{ let y = x; }` 相同。
- `;;` 是空语句。
- 类块表达式之后的成员访问（`.`）和 `?` 会延续该表达式（`match v { .. }.len();`）。在语句位置，其 `}` 之后的 `(` 或 `[` 开始一个新表达式（[1.3.4](../001-lexical/103-statement-terminator.md#134-类块表达式语句)）；在更大的表达式内部，如 `let f = if c { g } else { h }(x);`，它是调用或索引。

### 等价形式：最后表达式与显式 `return` / `break`

不带终止符的最后表达式，等价于与其上下文相应的显式退出语句。

```nabi
// function body: last expression == return value
fn f() -> i32 { 42 }
fn g() -> i32 { return 42; }      // same meaning

// labeled block: last expression == break 'label value
let x = 'b: { 42 };
let y = 'b: { break 'b 42; };     // same meaning

// loop: a loop without break is divergent. There is no last-expression position.
```

最后表达式的位置是值到达的出口。在函数体中，它表示函数的返回；在带标签的块中，它表示跳出到该标签。无论写成哪种形式，编译器产生的结果都相同。

## 3.9.6. 与函数体的关系

函数体也是块，其最后表达式的值就是函数的返回值。因此 `return` 关键字不是必需的。

```nabi
fn add(x: i32, y: i32) -> i32 {
    x + y               // no terminator, returned automatically
}

fn other(x: i32) -> i32 {
    let r = x + 1;
    return r;           // explicit return also allowed, same meaning as above
}

fn early(x: i32) -> i32 {
    if x < 0 { return 0; }   // explicit return for early exit
    x * 2                    // last expression of normal flow, returned automatically
}
```

三个函数都能工作。使用哪种形式是风格问题；规范完全支持两者。

### 建议的用法

使用最后表达式（不写 `return`）适合正常流程的单一返回点。显式 `return` 适合提前退出、从函数中间返回，以及为可读性而作标记。

函数体、`return` 关键字和返回类型规则详细定义于 [4. 函数](../004-functions/README.md)。

## 3.9.7. 块的类型

块的类型是其最后表达式的类型（没有最后表达式时为单元类型）。

```
{ 1i32 }                // i32
{ "hi" }                 // str
{ 1i32; }                // unit (expression ends with a terminator → result discarded)
{ }                      // unit (empty block)
```

这一类型确定与[表达式概述](README.md)中的分支结果类型规则相结合。如果同一位置的多个块表达式产生不同的类型，接收方必须有显式类型。

## 3.9.8. 发散块

有些块因 `return`、`break`、`continue`、`throw`、`loop { /* no break */ }` 等而不产生值。

```nabi
let x = {
    return 0;       // exits the function
};
```

这样的块被归类为发散的（永不完成）。发散块的类型与任何类型兼容，并能通过分支结果类型检查。

```nabi
let x: i32 = if cond {
    42i32
} else {
    return -1i32;       // divergent, does not affect branch agreement
};
```

检测发散的确切规则定义于函数和控制流规范。

## 3.9.9. 块的求值顺序

块中的语句和最后表达式按书写顺序（从上到下）求值，副作用也遵循这一顺序。

如果求值过程中发生 `return`、`break`、`continue`、抛出的错误等，块的求值在该处停止。

## 3.9.10. 带标签的块：`'label: { ... }`

带标签的块可以用在任何块表达式可以出现的地方：`let` 初始化器、函数体、`if` / `else` 分支、`match` 分支主体、另一个块的末尾表达式等。没有额外的位置限制；标签只指定该块的退出点。

在块前加上 `'name:` 形式的标签，就可以在块内用 `break 'name value;` 提前退出，同时设定块的值。

```nabi
let result = 'b: {
    if early_check_fails() { break 'b -1; }
    if shortcut_taken()    { break 'b  0; }
    let x = compute();
    let y = refine(x);
    x + y                    // last expression of normal flow: the value of 'b
};
```

标签词法单元的形式（`'identifier:`）与 [3.13.4 标签](313-loop-expressions.md#3134-标签控制嵌套循环)中相同。`break 'name value` 以给定的值退出该带标签的块。如果 `continue 'name` 的 `'name` 指向一个**块**标签，则是编译错误（E0059），因为 `continue` 只适用于循环标签。此外，不带标签的 `break` / `continue` 不能以块为目标（它们只以最内层的**循环**为目标）。

无需为提前退出而编写小的辅助函数，带标签的块就能清晰地表达带着一个值提前离开块。

## 3.9.11. `unsafe { ... }` 块

在块前放置 `unsafe` 关键字，使它成为进入不安全作用域的块。与普通块一样，最后表达式的值就是块的值，并且可以执行安全抽象保证之外的操作（裸指针解引用、调用 `unsafe fn` 等）。

```nabi
let n: usize = unsafe { strlen(cs.as_ptr()) };
```

详细含义和责任定义于 [4.15 unsafe](../004-functions/415-unsafe.md)。

## 3.9.12. 作用域结束时的释放顺序

大多数值无需释放：GC 会回收它们，没有确定的顺序，也不运行代码。类型带有 `Drop`（[标准库 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop释放时的清理)）的值在确定的时间点释放。以下规则汇集自定义它们的各节：

1. 值在其所有者离开作用域处释放，覆盖每种退出方式：正常完成、`return`、`break` / `continue`、`?`、`throw` 以及错误传播。移交给新所有者（调用实参、字段、通道）的值改由该所有者释放。
2. 在同一作用域内，所有者按声明的逆序离开（最后声明的最先释放）；内层块的所有者先于外层块的离开。
3. `using` 在其块结束时释放其绑定，最后一个绑定最先释放（[3.17.4](317-using-expression.md#3174-多重资源)）。
4. 释放一个值时，先运行它自己的 `drop`，然后按声明的逆序运行其字段的 drop glue，按逆序运行元组元素的，以及枚举或和类型当前所持有载荷的。
5. 重新声明名字的 `let`（[5.1.4](../005-modules/501-variable-declaration.md#514-在同一作用域中重新声明)）不释放任何东西：旧值在旧绑定本应离开作用域的地方释放。
6. 关闭可能失败的值（`io::Close`）从不替你关闭：让它在未关闭的状态下离开作用域是编译错误，没有所有者的此类类型的临时值也是（E0134）；应用 `using` 绑定它。
7. panic 会结束进程，不运行任何 `drop`。
