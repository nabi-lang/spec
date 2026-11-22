# 2.1. 类型表达式

类型表达式是在源代码中表示类型的写法。类型表达式出现在许多位置：变量声明的 `:` 之后、函数参数的 `:` 之后、表示返回类型的 `->` 之后、作为泛型实参，以及转换中的 `as` 之后，等等。

本文档只定义形式；哪个名称表示哪个类型由标准库决定。

## 2.1.1. 基本形式

类型表达式的基本语法如下：

```
type_expr   ::= sum_type
sum_type    ::= unary_type ("|" unary_type)*
unary_type  ::= base_type "?"*                  // T?: optional (2.1.5a), `??` is two `?`
base_type   ::= path_type
              | fn_type
              | tuple_type
              | array_type
              | any_type
              | self_type
              | qualified_type
              | macro_type
              | "none"                          // the none type (2.6)
              | "!"                             // never (4.3.7)
              | "(" type_expr ")"               // grouping
self_type   ::= "Self" ("::" identifier ("::" identifier)* type_args?)?   // Self, Self::Item
qualified_type ::= "<" type_expr "as" trait_ref ">" "::" identifier       // 2.13.4
macro_type  ::= identifier "!" ( "(" token* ")" | "[" token* "]" | "{" token* "}" )   // 6.8.12
any_type    ::= "any" ( trait_ref | "(" trait_ref ("+" trait_ref)* ")" )?
trait_ref   ::= identifier ("::" identifier)* trait_args?
trait_args  ::= "<" trait_arg ("," trait_arg)* ","? ">"
trait_arg   ::= type_expr | identifier "=" type_expr | const_arg
path_type   ::= identifier ("::" identifier)* type_args?
type_args   ::= "<" trait_arg ("," trait_arg)* ","? ">"
const_arg   ::= literal                         // int, char, true / false
              | "-" literal
              | "{" expr "}"                    // 2.11.4
fn_type     ::= "unsafe"? ("extern" string_lit)? "fn" "( (type_expr ("," type_expr)*)? ")" ("throws" ("(" type_expr ")")?)? ("->" unary_type)? fn_bound*
fn_bound    ::= "+" ("Send" | "Sync")
tuple_type  ::= "(" ")"
              | "(" type_expr "," ")"
              | "(" type_expr ("," type_expr)+ ","? ")"
array_type  ::= "[" type_expr ";" const_expr "]"
```

`extern "C" fn(A, B) -> R` 是 C 函数指针类型（[5.9.4](../005-modules/509-extern-block.md#c-function-pointer-types)）。ABI 字符串只能是 `"C"` 或 `"system"`（[5.9](../005-modules/509-extern-block.md)）；其他任何值都是错误。

各种形式汇总如下。

| 形式              | 示例                                        |
|-------------------|---------------------------------------------|
| 简单名称          | `i32`、`bool`、`Point`                      |
| 路径              | `core::time::Duration`                 |
| 泛型应用          | `List<i32>`、`Dict<str, f64>`、`Pair<A, B>` |
| 和类型（`\|`）    | `i32 \| f32`、`T \| none`                     |
| 可选（`?`）       | `i32?`、`(i32 \| str)?`、`List<T>?`           |
| 函数类型          | `fn(i32, i32) -> i32`、`fn() -> T + Send`   |
| 元组类型          | `(i32, str)`、`(i32,)`                      |
| 数组类型          | `[T; N]`（定长）                            |
| 动态槽            | `any`、`any Error`、`any (Display + Hash)`  |
| `none` / never    | `none`、`!`                                 |
| `Self` 及其项     | `Self`、`Self::Item`                     |
| 限定路径          | `<T as Iterator>::Item`                     |
| 宏调用            | `name!(..)`，展开为一个类型（[6.8.12](../006-meta-system/608-meta-blocks.md#6812-the-macro-form)） |
| 常量实参          | `Buf<16>`、`Buf<-1>`、`Buf<{ N + 1 }>`      |

下文依次定义每种形式的确切含义。

形如 `T::Assoc` 的关联类型投影（`I::Item`、`F::Output`）也是类型。它属于路径形式（`path_type`）之一，当 `T` 是受某个 trait 约束、且该 trait 具有这个关联类型的类型参数时有效（[类型路径](213-type-paths.md)）。

```nabi
fn zip<I: Iterator, J: Iterator>(a: I, b: J) -> List<(I::Item, J::Item)> { ... }
```

## 2.1.2. 简单名称

简单名称由单个标识符组成，遵循[标识符](../001-lexical/104-identifiers.md)的形式。

```nabi
i32
bool
Point
변수타입
```

名称指向哪一项在名称解析阶段决定。候选包括当前模块中定义的项和用 `use` 导入的项。

## 2.1.3. 路径

用 `::` 连接的多个标识符也可以用作类型表达式。

```nabi
core::time::Duration
core::collections::List
pkg::geom::Point
```

这里的 `::` 是[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)中定义的路径分隔符。路径的最后一个词法单元是类型名；前面的部分是模块或项的路径，说明在哪里找到该类型。

路径的含义与解析规则在[类型路径](213-type-paths.md)中定义；本文档只说明路径可以作为类型表达式的一种形式出现。

## 2.1.4. 泛型应用

在类型名后追加 `<...>` 可以给出类型实参。

```nabi
List<i32>
Dict<str, f64>
Pair<A, B>
Result<T, E>
Matrix<f64, 3, 4>      // const generic (value arguments, 2.11.1)
```

规则如下：实参之间用逗号分隔。实参可以是另一个类型表达式；对于 `const` 泛型，也允许 `const` 表达式。不允许空的 `<>`；没有实参时，根本不写 `<>`。允许尾随逗号（`<A, B,>`）。

值实参（例如 `Matrix<f64, 3, 4>`）的确切含义在 [`const` 泛型](211-const-generics.md)中定义。

### 表达式位置中的 turbofish

在表达式内部给出的泛型实参使用 `::<...>` 形式，由表达式规范定义。本文档只涉及类型表达式位置中的 `<...>`。

## 2.1.5. 和类型（`|`）

值为若干类型之一的和类型用 `|` 书写。

```nabi
i32 | f32
T | none
A | B | C | D
```

以 `|` 分隔的每一项都是一个独立的类型表达式，得到的和类型持有其中某一项的值。其含义与内存表示在[和类型](206-sum-types.md)中定义。

`|` 的结合比其他类型组合符更松。因此 `i32 | List<f32>` 分组为 `i32 | (List<f32>)`，因为泛型应用比和类型结合得更紧。

## 2.1.5a. 可选类型（`T?`）

后缀 `?` 构成可选类型。`T?` 的行为与 `Maybe<T>`（`type Maybe<T> = T | none;`）完全相同。也就是说，`?` 的操作数是一个泛型项，适用[和类型的分层规则](206-sum-types.md#264-set-semantics)。

```nabi
i32?                // i32 | none
(i32 | str)?        // i32 | str | none (no overlap, flattened)
(i32 | none)?       // two layers: outer none ≠ inner none
i32??               // two layers: same as (i32 | none)?
i32 | none          // written sum: always flat i32 | none
```

- 后缀 `?` 比 `|` 结合得更紧，也比函数类型的 `->` 结合得更紧（2.1.10）。`i32 | str?` 是 `i32 | (str?)`。
- `fn() -> i32?` 是返回可选值的函数；`(fn() -> i32)?` 是可选的函数。
- 在类型位置中，`??` 词法单元是两个 `?`（`i32??`）。
- 在表达式 `x as T?` / `x is T?` 中，末尾的 `?` 属于类型。要对结果应用 `?` 运算符（[3.14](../003-expressions/314-error-control-flow.md)），需加括号：`(x as T)?`。
- 不过，`as` / `is` 的目标之后的 `??` 永远不是类型的一部分：类型在它之前结束，`??` 是空值合并运算符。要在那里写两层可选类型，需加括号：`x is (i32??)`。

## 2.1.6. 函数类型

表示函数本身的类型写作：

```nabi
fn(i32, i32) -> i32
fn() -> str
fn(T) -> T
fn(i32)                  // return type may be omitted (unit type)
```

函数类型以 `fn` 词法单元开头；括号内只有参数类型，没有参数名。返回类型跟在 `->` 之后；省略它表示没有返回值（确切含义由函数规范定义）。

## 2.1.7. 元组类型

把若干类型组合在一起的元组类型写作：

```nabi
(i32, str)
(f64, f64, f64)
(i32,)                  // a one-element tuple requires a trailing comma
()                      // empty tuple (unit type)
```

规则：有两个或更多元素时，用逗号分隔。单元素元组必须带尾随逗号（`(T,)`）；没有逗号时，它被解析为对 `T` 的分组。没有元素的元组（`()`）表示单元类型。

## 2.1.8. 数组类型

定长数组类型用方括号书写。

```nabi
[T; N]
[i32; 16]
[f64; 1024]
[u8; SIZE]
```

数组类型在方括号内依次写元素类型、分号和长度。长度是 `const` 表达式，必须在编译期确定。允许长度为 0（`[T; 0]`，即空数组类型）。

## 2.1.9. 动态槽

动态类型槽用 `any` 关键字书写。

```nabi
any
any Error
any (Display + Hash)
any Iterator<Item = i32>
any Future<Output = User>
```

`any` 是[关键字](../001-lexical/105-keywords.md)中保留的类型名槽关键字；其含义在 [`any` 槽](202-any-slot.md)中定义。在 `any` 之后跟一个 trait 路径，或跟用括号括起、以 `+` 连接的多个 trait 路径，就构成受约束的动态槽，它只持有实现了这些 trait 的值（[2.2.10](202-any-slot.md#2210-constrained-any-any-trait)）。在 trait 路径的 `<...>` 中，除类型实参外，还可以出现固定关联类型的绑定，例如 `Item = T`（`trait_arg`）。两个或更多 trait 路径必须加括号；`any` 只绑定紧随其后的那一个 trait，所以 `any Error | none` 是 `(any Error) | none`。trait 路径只能在 `any` 之后出现在类型位置中（[trait 不能作类型](212-trait-as-type-banned.md)）。

## 2.1.10. 优先级与结合

类型表达式中混用组合符时，其优先级（紧 ↔ 松）如下：

```
strong  ─→  weak

[T; N]   (...)   <...>   ::   .  →  T?  →  fn(...) -> ...   →   |
```

也就是说，路径（`::`）、泛型应用（`<...>`）、数组（`[...]`）和分组（`(...)`）结合得最紧。其次是后缀可选（`?`），然后是函数类型（`fn(...) -> ...`），和类型（`|`）结合得最松。

因此，`i32 | List<f32>` 分组为 `i32 | (List<f32>)`，`fn(A | B) -> C` 分组为 `fn((A|B)) -> C`。

需要显式分组时，使用括号。

```nabi
fn(i32) -> (i32 | str)         // return type is a sum type
(fn(i32) -> i32) | (fn(f64) -> f64)   // sum type with two function types as terms
```
