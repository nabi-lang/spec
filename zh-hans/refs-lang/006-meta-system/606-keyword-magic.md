# 6.6. `#[keyword(...)]` 魔法

`#[keyword("name")]` 属性是标准库注册新关键字的机制。

## 6.6.1. 动机

[关键字](../001-lexical/105-keywords.md)中的保留关键字列表是由语言本身定义的固定集合。难道每要增加一个关键字，都必须修改语言规范吗？

标准库有时需要引入看起来像关键字的词法单元，例如库 DSL 中的领域关键字。`#[keyword]` 魔法就是在这类情况下，库注册语言级关键字的正式渠道。

## 6.6.2. 形式

```
keyword_attr    ::= "#" "[" "keyword" "(" string_lit ")" "]"
```

```nabi
#[keyword("yield_priority")]
fn yield_priority_handler(...) { ... }

#[keyword("with")]
fn with_handler(...) { ... }
```

基本形式是 `#[keyword("name")]`，以关键字名（标识符形状的字符串字面量）为实参。处理函数是一个**自由函数**，不能附加在方法或 trait 项上。

`struct` 上的 `#[keyword("name")]` 注册的是**类型关键字**：该名字在类型位置表示这个结构体（`int` → `core::num::bigint::Int`，`float` → `core::num::bigfloat::Float`，[标准库 2.1.2](../../refs-standard-libraries/200-core/201-numbers.md#212-int任意精度整数)）。只有标准库可以注册类型关键字。把它附加在既不是函数也不是结构体的项上是错误。

## 6.6.3. 注册的效果

某处注册了 `#[keyword("name")]` 后，会产生以下效果。

1. 在该编译单元中，标识符 `name` 被切分为关键字。
2. 关键字的使用脱糖为对处理函数的调用。`name(args)` 是经由处理函数绝对路径的调用，其后跟随的块成为[尾随闭包](../004-functions/408-trailing-closure.md)实参。

```nabi
#[keyword("with")]
fn with_handler(m: Mutex<i32>, body: fn()) { ... }

with(m) { critical(); }
// ⇒ pkg::with_handler(m) { critical(); }     (handler + trailing closure)
```

注册之前，`name` 可以用作普通标识符；注册之后，它是关键字词法单元。因此，把 `name` 用作声明或成员的名字是编译错误。这些位置是 `let name`、`fn name`、`struct name`、`enum name`、`trait name`、`mod name`、`for name in`、`.name`、`::name`、`name:`（字段名或实参名）以及 `name::`。同一规则也适用于 f-string 的 `{..}` 内部。原始标识符 `r#name`（[1.4.4](../001-lexical/104-identifiers.md#原始标识符)）始终是标识符，因此可以在这些位置保留该名字（`struct S { r#f16: str; }`、`s.r#f16`）。

## 6.6.4. 冲突检查

```nabi
#[keyword("if")]                  // ERROR: 'if' is a reserved keyword
fn ... { ... }
```

[关键字](../001-lexical/105-keywords.md)中的保留关键字、只在固定位置才是关键字的名字（`type`、`any`、`in`、`where`、`throws`、`rethrows`、`catch`、`finally`、`const`、`static`、`extern`、`mut`、`union`）以及路径根（`pkg`、`sup`、`nabi`）都不能注册。它们在注册时立即被拒绝。

在同一次构建中重复注册同一个关键字是错误，无论是在同一个包内还是在不同的库之间。

```
package_a registers #[keyword("with")]
package_b also registers #[keyword("with")]    → ERROR if both packages are used in one build
```

## 6.6.5. 适用范围

`#[keyword]` 只在包含该注册的编译单元（通常是一个包）内生效。在其他包中，同一个名字作为普通标识符使用。例外是标准库的类型关键字（`int`、`float`、`f16`、`bf16`），它们适用于每个包。

这种局部性减少了库之间的冲突，也让用户能够有意识地决定由哪个包引入哪个关键字。

## 6.6.6. 使用模式

`#[keyword]` 很少使用，主要用于以下情况：

- 领域特定语言（DSL）：当领域词汇（表格数据、网格、电路等）在标识符位置表现得像关键字更为自然时。
- 难以用运算符魔法表达的、块形式或主体形式的新语法。

在大多数情况下，函数与尾随闭包的组合就足以构造类似关键字的模式（[尾随闭包](../004-functions/408-trailing-closure.md)）。能用这种形式时，应优先使用它。

## 6.6.7. 与扫描预处理的配合

`#[keyword]` 注册在何时被识别，由编译器的扫描预处理（scanner pre-pass）负责。

1. 在解析之前，编译器快速扫描源文件，收集 `#[keyword(...)]` 注册。
2. 注册表被传给词法分析器。
3. 在完整的词法分析过程中，已注册的名字被切分为关键字词法单元。
4. 解析器把它们脱糖为已注册的处理函数。

借助这一预处理，无论注册出现在哪里（哪个文件、哪一行），编译单元内都能保持一致。用户无需关心注册顺序。

## 6.6.8. 关键字处理函数的签名

编译器不单独检查处理函数的签名。关键字的使用变为处理函数调用（6.6.3）之后，该调用作为普通函数调用进行类型检查。因此，处理函数的参数决定了关键字的使用形式（实参、尾随闭包）。

## 6.6.9. 已注册的关键字永远不是上下文关键字

[1.5.4](../001-lexical/105-keywords.md#154-上下文关键字) 中的上下文关键字（`type`、`in`、`static` 等）是语言定义的固定集合。已注册的关键字永远不是上下文关键字。

一旦注册了 `#[keyword("with")]`，`with` 在编译单元内的任何位置都被切分为关键字。注册不能让一个词只在某些位置成为关键字。

这种一致性有助于保持词法分析器和解析器的简单。

## 6.6.10. 注册的可见性

带有 `#[keyword]` 的项的可见性（[可见性](../005-modules/506-visibility.md)）只适用于对该项的访问，与关键字注册的效果相互独立。

```nabi
pub(pkg) fn keyword_handler(...) { ... }

#[keyword("with")]
fn another_handler(...) { ... }     // keyword registration: applies to the whole package
```

处理函数能否从外部调用由可见性决定，但关键字注册本身影响整个编译单元。

## 6.6.11. `#[keyword]` 与用户定义宏的区别

与其他语言的宏系统相比：

- Rust 过程宏（proc macro）：执行任意的词法单元变换，用户可以自由生成代码。卫生性是另一回事。
- Lisp 宏：动态宏，也可以在运行时注册。
- Nabi `#[keyword]`：编译期的关键字注册，由处理函数决定脱糖方式。无法进行自由的词法单元变换。

Nabi 有意采用受限的宏系统，在表达力与工具友好性的权衡中偏向工具友好性。

## 6.6.12. 已注册的关键字与函数调用

```nabi
// keyword form
with(mutex) { critical(); }

// function call form (no registration)
with_lock(mutex) { critical(); }     // function + trailing closure
```

大多数类似 DSL 的模式都可以通过函数与尾随闭包的组合来实现。应把 `#[keyword]` 注册限制在那种形式显得不自然的情况。

## 6.6.13. 何时求值

关键字注册在扫描预处理阶段处理，早于其他元处理（cfg、元块展开等）。

由于这一顺序，cfg 可以有条件地作用于关键字处理函数本身，关键字也可以在元块主体中使用。元块或关键字处理函数的 `#[cfg]` 按定义它的包的 cfg 求值（使用该包自己的特性，[9.10.4c](../009-tools/910-nabi-frontend.md#9104c-特性)）。

## 6.6.14. 谨慎使用

`#[keyword]` 应当很少、谨慎地使用。每一次注册都会扩大语言的表面，并让其他用户承担猜测新关键字含义的负担。

对于会被普遍采用的语言关键字，把它直接加入[关键字](../001-lexical/105-keywords.md)中的保留列表（也就是作为语言规范的变更）可能更合适。
