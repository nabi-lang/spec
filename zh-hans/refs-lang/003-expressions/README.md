# 3. 表达式概述

本章定义表达式和语句的形式，以及让每种控制结构都成为表达式所带来的影响。各类表达式及其语义在随后的文档中讨论。

## 目录

- [3.1. 运算符分派](301-operator-dispatch.md)
- [3.2. 优先级与结合性](302-precedence-and-associativity.md)
- [3.3. 广播运算符](303-broadcast-operators.md)
- [3.4. 赋值与复合赋值](304-assignment.md)
- [3.5. 成员访问与方法调用](305-member-access-and-method-call.md)
- [3.6. 索引与切片](306-indexing-and-slicing.md)
- [3.7. 区间表达式](307-range-expressions.md)
- [3.8. 可选链与空值合并](308-optional-chaining-and-coalescing.md)
- [3.9. 块表达式](309-block-expressions.md)
- [3.10. `if` 表达式](310-if-expression.md)
- [3.11. 模式](311-patterns.md)
- [3.12. `match` 表达式](312-match-expression.md)
- [3.13. 循环表达式](313-loop-expressions.md)
- [3.14. 错误控制流](314-error-control-flow.md)
- [3.15. 解构赋值](315-destructuring.md)
- [3.16. `let-else`](316-let-else.md)
- [3.17. `using` 表达式](317-using-expression.md)
- [3.18. 管道表达式](318-pipe-expression.md)

## 表达式与语句

表达式是产生值的构造：对它求值会得到某种类型的值。**语句**是不产生值、以 `;` [语句终止符](../001-lexical/103-statement-terminator.md)结尾的代码单位。

Nabi 用以下规则把两者联系起来。

```
<statement> = <expression> ';'        // a statement that discards the expression's result
```

语句就是附加了终止符的表达式。几乎每种代码构造都是产生值的表达式；丢弃该值，它就成为语句。

## 控制结构是表达式

每种控制结构（`if`、`match`、`loop`、`while`、`for`、`try`）也都是表达式；`try` 表达式的值见 [3.14.6](314-error-control-flow.md#3146-表达式的值与类型)。下面的示例展示了直接产生值的控制结构。

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

这一决定带来两个后果。第一，控制结构可以出现在任何表达式位置：函数实参、变量初始化式、另一个表达式的一部分。第二，不需要单独的三元形式：不存在 `cond ? x : y` 词法单元；`if cond { x } else { y }` 承担这一角色。

各控制结构所返回的值的确切规则，在相应的规范文档中定义。

## 分支结果类型

表达式的结果类型与接收其值的槽的类型是分别确定的。

### 表达式的结果类型

如果控制结构的每个分支都产生同一类型的值，该类型就是表达式的结果类型。类型本身已是和类型的分支按其成员计算：如果其他每个分支的类型都是该和类型或其成员之一，该和类型就是结果类型，与哪个分支在前无关。不会形成新的和类型。类型来自其槽的分支，例如元素类型尚未确定的空 `[]` 或 `Array::zeros(..)`，采用其他分支的类型，无论在它们之前还是之后（当 `xs: List<f64>` 时，`if c { xs } else { [] }` 是 `List<f64>`）。否则，如果各分支产生不同类型的值，接收方必须提供类型（槽的显式类型、参数类型或返回类型），且每个分支都必须与之兼容。如果接收方没有类型，就是编译错误。和类型的自动推断和向公共父类型的自动提升（例如把 `i32` 隐式提升为 `i64` 或 `f64`）都是不允许的。

```
if cond { 1i32 } else { 2i32 }          // result type of the expression: i32
if cond { 1i32 } else { 2.0f64 }        // the receiving side must specify i32 | f64 or similar
if cond { none } else { x }             // x: i32 | none → result type: i32 | none
if cond { "a" } else { v }              // v: i32 | str → result type: i32 | str
if cond { none } else { 1i32 }          // the receiving side must specify i32 | none
if cond { xs } else { [] }              // xs: List<f64> → result type: List<f64>
```

### 和类型是显式的；不做提升

这两种情况被拒绝的理由不同。不自动推断和类型，是为了让一个分支中的错误不会悄无声息地产生一个和类型，并随后扩散到所有后续代码中。需要和类型时，应在接收方写明，如 `let d: i32 | str = if ...`。而自动的父类型提升会把一种类型转换为另一种类型（`i32` → `f64`），可能丢失精度或表示，因此即使接收方有类型也不允许。

这一区分与[语言与标准库的边界](../000-meta/004-language-vs-standard-libraries.md)中的显式性原则一致。任何需要的转换都必须用[转换](../002-types/214-cast-and-conversion.md)的三种形式之一显式书写：`as`、`.into()`、`.try_into()`。

### 用槽接收表达式结果

当变量槽接收表达式的结果时，槽的类型遵循[变量声明](../005-modules/501-variable-declaration.md#517-类型标注与类型推断)的规则。给出类型时，它就是接收类型；省略时，槽的类型由表达式的结果类型推断。

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

### 槽兼容性表

表达式结果类型与接收槽之间的兼容性总结如下。

| 接收槽                              | 表达式结果类型                              | 是否允许                   |
|-------------------------------------|---------------------------------------------|----------------------------|
| `any`（显式 `: any`）               | 任意                                        | 总是允许                   |
| 和类型 `T1 \| T2 \| ...`              | 结果成员都在槽的集合之内                    | 允许                       |
| 单一具体类型 `T`                    | 结果恰好是 `T`                              | 允许                       |
| 单一具体类型 `T`                    | 结果含有 `T` 以外的成员                     | 拒绝                       |
| 省略类型标注                        | 所有分支类型相同                            | 允许（推断为该类型）       |
| 省略类型标注                        | 某个分支的和类型包含所有分支的类型          | 允许（推断为该类型）       |
| 省略类型标注                        | 除此之外分支类型不同                        | 拒绝                       |

### 渐进与静态

在这些兼容性规则之上，用户可以在三种模式中选择。在渐进模式（`let x: any = ...`）下，槽为 `any`，自由接受任意分支组合；分派通过运行时标签检查进行，适合快速探索。在静态和类型模式（`let x: T1 | T2 = ...`）下，槽是显式的和类型，分支结果必须落在其成员集合之内；可以使用类型收窄和模式匹配的穷尽检查，适合追求稳定性和性能。在静态单一类型模式（`let x: T = ...`，或省略类型的 `let x = ...`）下，槽是单一类型，每个分支都必须是该类型，从而获得最高性能和完整的静态分析。

三种模式可以在同一代码中自由混用；用户按变量选择。

## 区分表达式与语句

大多数代码行采用以下两种形式之一。

```nabi
<declaration>;           // a statement: a definition of let / fn / struct / ...
<control-or-expr>;       // a statement that discards the expression's result
```

以 `}` 结尾的表达式，例如 `if` / `match` / 循环 / 块，在语句位置可以省略 `;`（[1.3.4](../001-lexical/103-statement-terminator.md#134-类块表达式语句)）。

块中的最后一个表达式是例外。

```nabi
{
    let a = 1;
    let b = 2;
    a + b           // no terminator: the value of this block
}
```

只有块的最后一个表达式可以不带终止符，此时它成为整个块的值。详细规则在[块表达式](309-block-expressions.md)中定义。

## 副作用

表达式在求值过程中可能产生副作用，例如 I/O、变量重新赋值或外部调用。求值顺序由运算符优先级和函数调用的求值顺序决定，这些在后续文档中定义。

当副作用在单个表达式内形成依赖时，该依赖遵循明确的顺序：从左到右、深度优先。编译器不会重新排列它。赋值先对其值求值，再对其位置求值（[3.4.7](304-assignment.md#347-求值顺序)）。
