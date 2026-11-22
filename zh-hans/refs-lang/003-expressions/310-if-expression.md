# 3.10. `if` 表达式

`if` 和 `else` 构成一个按布尔值分支的表达式。

## 3.10.1. 形式

```
if_expr     ::= "if" expr block ("else" (if_expr | block))?
              | "if" "let" pattern "=" expr block ("else" (if_expr | block))?
```

```nabi
if cond { ... }
if cond { ... } else { ... }
if cond1 { ... } else if cond2 { ... } else { ... }
```

其中 `cond` 是 `bool` 类型的表达式，主体和 `else` 主体都是[块表达式](309-block-expressions.md)。`else if` 是 `else { if ... }` 的简写；它不是单独的词法单元，而是两者自然的组合。

## 3.10.2. 作为表达式的值

只有带 `else` 子句时，`if` 才产生 unit 以外的值。

```nabi
let a = if cond { 1i32 } else { 2i32 };          // OK, i32
let b = if cond { 1i32 };                          // OK, but b has type unit (no else)
```

没有 `else` 的 `if` 只有一个分支，因此不返回值，结果为 unit。带 `else` 的 `if` 把两个分支的值合并为它的结果。

### 分支结果类型

按照[表达式概述](README.md)中的分支结果规则，如果所有分支类型相同，该类型就是 `if` 表达式的结果类型；某个分支的和类型包含另一分支的类型时也是如此（当 `x: i32 | none` 时，`if c { none } else { x }` 为 `i32 | none`）。除此之外分支类型不同时，接收方必须有显式类型，且每个分支都必须与之兼容。和类型不会被自动推断。接收槽的类型遵循[变量声明](../005-modules/501-variable-declaration.md)的规则。

```nabi
// no type on the slot → inferred from the expression's result type
let x = if cond { 1i32 } else { 2i32 };          // expression result i32, slot x: i32
let y = if cond { 1i32 } else { 2.0f64 };        // ERROR: branch type mismatch; type annotation required

// sum type on the slot → member check
let w: i32 | f64 = if cond { 1i32 } else { 2.0f64 };   // OK

// single type on the slot → the expression's result must be that type
let z: i32 = if cond { 1i32 } else { 2.0f64 };   // ERROR: f64 branch does not fit i32
```

`else if` 链的每个分支（每个 `else if` 和最后的 `else`）都要经过这一检查。

## 3.10.3. 条件表达式的位置

```nabi
if cond { ... }
```

条件位置 `cond` 可以出现任何 `bool` 表达式：调用、方法调用、比较、逻辑组合等。

```nabi
if user.is_admin() && !user.is_locked { ... }
if items.len() > 0 { ... }
```

非 `bool` 类型是编译错误（E0040）。不存在把 0 或空字符串视为 `false` 的隐式真值转换。需要布尔值的地方，应写出显式的转换或比较。

```
if x { ... }              // ERROR if x: i32 (i32 is not bool)
if x != 0i32 { ... }      // OK
```

检测类型参数的条件（`if T is f32 { .. }`）在每个实例中都是常量；其分支把 `T` 视为被检测的类型，且一个实例只降级它所走的那个分支（[2.10.13](../002-types/210-generics.md#21013-对类型参数的类型测试)）。

### 条件位置中的结构体字面量

在以下位置，结构体字面量不能出现在表达式的顶层，因为它的 `{` 会被读作主体的开始。

- `if` / `while` / `if let` / `while let` 的条件
- `match` 的被匹配值（[3.12](312-match-expression.md)）
- `for` 的迭代目标（[3.13](313-loop-expressions.md)）
- `using` 的资源表达式（[3.17](317-using-expression.md)）

在定界符内部（`( )`、`[ ]`、调用 / 方法实参、数组、元组、索引、字典字面量、块 `{ }`）是允许的。要在顶层使用，用圆括号把它括起来。

```nabi
if S { a: 1 }.ok() { .. }       // ERROR (E0013): the `{` after `S` is the body
if (S { a: 1 }).ok() { .. }     // OK
if check(S { a: 1 }) { .. }     // OK: inside an argument
```

## 3.10.4. 发散分支

在分支结果类型检查中，[块表达式](309-block-expressions.md)的发散块与任何类型都兼容。

```nabi
let x: i32 = if cond {
    42i32
} else {
    return -1i32;          // divergent, OK even in an i32 position
};
```

这条规则让提前退出可以自然地表达。

## 3.10.5. `if let`：与模式匹配结合

`if` 的条件位置也可以与模式匹配结合。

```nabi
if let v = compute() {             // compute(): User | none; v: User
    consume(v);
} else {
    fallback();
}
```

```
if_let ::= "if" "let" pattern (":" type_expr)? "=" expr block ("else" (block | if_expr))?
```

- 不带类型时，表达式的值与 `pattern` 匹配。当值是可选的（含 `none` 的和类型）且 `pattern` 是普通名字时，该名字匹配去掉 `none` 之后的值：对于 `T | none`，`if let x = xs.pop()` 绑定 `x: T`；对于 `A | B | none`，绑定 `x: A | B`。只去掉一层：对于 `(T | none)?`（可选调用的 `try?`），`x` 为 `T | none`。元组和结构体模式本来就只匹配与其形状相符的成员。
- 带 `: T` 时，值先收窄为 `T`，再由 `pattern` 解构。此时 `pattern` 对 `T` 必须是不可反驳的（`if let (a, b): (i32, i32) = v { .. }`）。
- 匹配成功时运行主体，模式的绑定在主体内可见。否则运行 `else`。

`if let` 的模式语法在[模式](311-patterns.md)中定义。

## 3.10.6. 链式 `if let`（不采用）

不采用 Rust RFC 2497 风格、在一行中串联多个 `if let` 的写法。

```
if let A(a) = x && let B(b) = y { ... }     // not adopted
```

作为替代，使用[模式](311-patterns.md)中的元组模式或嵌套的 `if let`。

## 3.10.7. 对 `else` 主体形式的限制

`else` 之后只能跟块（`{ ... }`）或另一个 `if` 表达式。不允许直接跟普通表达式。

```
if a { 1 } else 2          // ERROR: block required after else
if a { 1 } else { 2 }      // OK
if a { 1 } else if b { 2 } else { 3 }    // OK, if after else
```

这一限制避免了有歧义的悬挂 `else`。

## 3.10.8. 求值顺序

`if` 按以下方式求值。

1. 对 `cond` 求值。
2. 如果 `cond` 为 `true`，对 `then` 主体求值；如果为 `false`，对 `else` 主体求值。

求值是短路的。`cond` 一旦确定，只对其中一侧求值，另一侧的副作用不会发生。`else if` 链自上而下检查，运行第一个条件为 `true` 的分支。
