# 1.5. 关键字

关键字是外形像标识符、但由语言保留的词法单元。当关键字出现在标识符位置时，编译器产生的是关键字词法单元，而不是标识符。

本文档只定义语言直接保留的关键字列表。每个关键字的确切含义在规范后面的部分介绍；本文档只说明它们的形式和分类。

## 1.5.1. 保留关键字列表

关键字分为两类。其原则是下面这一条规则。

> [!IMPORTANT]
> 开始一条语句或一个表达式、且可能与“标识符 + 其后内容”相混淆的词是**保留关键字**；出现位置固定的词是**上下文关键字**（1.5.4）。

保留关键字无论出现在哪里都被切分为关键字，不能用作标识符。唯一的例外是 1.5.5 中的名称位置。

```
let       fn        if        else      match     while     loop      for
return    break     continue  try       throw
async     await     yield     unsafe    using
struct    enum      trait     impl      pub       mod       use
as        is        ref
true      false     none      self      Self
macro     defer     dyn       move      gen       do
```

> [!NOTE]
> `try?` 和 `try!` 不是独立的关键字，而是关键字 `try` 与标点 `?` / `!` 组成的词法单元序列（[标点与运算符词法单元](110-punctuation-and-operator-tokens.md#点--问号)）。

`self`（小写）指称实例标识符（方法的接收者），`Self`（大写）指称当前 `impl` 所针对的类型的名称。`using` 是引入 RAII 式资源管理块的关键字（[`using` 表达式](../003-expressions/317-using-expression.md)）。

## 1.5.2. 含义在后文定义

上述词法单元的含义分布在整个规范中定义：变量、可见性和模块在第 5 章；类型和容器在第 2 章；控制流在第 3 章；函数、闭包和协程在第 4 章；元系统和魔法在第 6 章。

第 1 章只涵盖这样一条分类规则：在词法阶段，这些词法单元被区分为关键字而不是标识符。

## 1.5.3. 注册关键字（扩展机制）

除上面的保留列表外，还有一种机制可以让标准库注册额外的关键字。它是元系统的一部分。注册机制和确切的处理规则在 [`#[keyword(...)]` 魔法](../006-meta-system/606-keyword-magic.md)中定义。

在本文档的范围内，只承诺以下几点。除 1.5.5 的名称位置外，保留关键字不是标识符，而被区分为关键字。注册的关键字只在包含其注册的编译单元内生效。如果两个集合发生冲突，注册会被拒绝；保留关键字不能被覆盖。

## 1.5.4. 上下文关键字

以下词只在指定位置上是关键字，在其他位置是普通标识符。

| 词                                 | 作为关键字的位置                                                       | 定义 |
|------------------------------------|------------------------------------------------------------------------|------|
| `type`                             | 位于项的开头，且后跟一个标识符，再跟 `=` 或 `<` 时（类型别名）         | [2.6](../002-types/206-sum-types.md) |
| `any`                              | 类型位置（`any`、`any Tr`）                                            | [2.2](../002-types/202-any-slot.md) |
| `some`                             | 函数的返回类型或参数类型中，或类型别名的右侧，位于约束之前（`-> some Tr`、`x: some Tr`、`type X = some Tr;`） | [4.1.13](../004-functions/401-function-declaration.md#4113-不透明返回类型some-trait), [4.1.15](../004-functions/401-function-declaration.md#4115-具名不透明类型type-x--some-trait) |
| `in`                               | `for pat in` 中的 `in`；在表达式中是成员运算符 `in` / `!in`（`∈` / `∉`） | [3.13.3](../003-expressions/313-loop-expressions.md#3133-for基于迭代器的重复), [3.2.3](../003-expressions/302-precedence-and-associativity.md#323-优先级与结合性表) |
| `where`                            | 函数签名、`impl` 头部，或 `struct` / `enum` / `union` / 类型别名头部之后 | [2.10.4](../002-types/210-generics.md#2104-where-子句) |
| `throws`, `rethrows`               | 函数签名的参数 `)` 之后                                                | [4.14](../004-functions/414-throws.md) |
| `catch`, `finally`                 | 表达式或块之后（后缀子句）                                             | [3.14](../003-expressions/314-error-control-flow.md) |
| `const`                            | 位于项的开头，在 `impl` / `trait` 体中也是（`const` 项、`const fn`），以及在泛型参数列表中（`<const N: usize>`） | [5.2](../005-modules/502-const.md), [2.11](../002-types/211-const-generics.md) |
| `static`, `extern`, `mut`          | 位于项的开头（包括 `extern "C"`、`static mut`）                        | [5.3](../005-modules/503-static.md), [5.9](../005-modules/509-extern-block.md) |
| `union`                            | 位于项的开头，且紧跟一个标识符时                                       | [2.5](../002-types/205-union.md) |
| `pkg`, `sub`                       | 紧跟在 `pub(` 之后                                                     | [5.6](../005-modules/506-visibility.md) |
| `pkg`, `sup`                       | 作为路径的第一个词法单元，且紧跟 `::` 时                               | [5.7](../005-modules/507-paths.md) |

```nabi
let type = 1;                       // identifier
type Pair = (i32, i32);             // type alias (item start + identifier + =)
let in = 3;  for x in xs { }        // identifier / the in of for
fn catch(e: str) { }                // identifier: not after an expression
```

声明名为 `any` 的类型（`struct any { }`）是编译错误，因为它无法与类型位置上的 `any` 区分。

### 可见性限定符：`pkg` / `sub`

`pkg` 和 `sub` 在 `pub(pkg)` 和 `pub(sub)` 的括号内用作限定符，在其他位置是普通标识符。

```nabi
pub(sub) fn helper() { ... }   // inside pub( ): visibility qualifier
trait Sub { fn sub(self, rhs: Self) -> Self; }  // in ordinary position: identifier
```

### 路径根：`pkg` / `sup`

这两个词作为路径的第一个词法单元且紧跟 `::` 时，总是路径根（[5.7 路径](../005-modules/507-paths.md)）。在其他位置它们是普通标识符。

- `pkg::xxx`：从当前包的根开始
- `sup::xxx`：从父模块开始

标准库层级 `std`、`core` 和 `bare`（[0.5 标准库层级](../000-meta/005-standard-libraries-tiers.md)）以及词 `nabi` 都不是关键字：局部名称可以使用它们（[5.7.4](../005-modules/507-paths.md#574-stdcorebare标准库层级)）。

由于 `pkg::name` 总是被解释为根，把 `pkg` 或 `sup` 用作模块或类型（`struct`、`enum`、`union`、`trait`、类型别名、`mod`）的名称是编译错误。它们可以用作变量、函数、字段和方法的名称。

```nabi
let pkg = 42;                   // variable name OK
fn sup() { }                    // function name OK
self.pkg                        // field / member name OK
mod sup { ... }                 // ERROR: cannot be a module name
struct pkg { }                  // ERROR: cannot be a type name
::core::iter::Iterator     // the library past a local `core`
```

## 1.5.5. 名称位置上的关键字

在以下位置，所有关键字（包括保留关键字）都用作标识符，因为周围的词法单元已清楚表明这里需要的是名称。

- 紧跟在 `.` 之后（成员访问、方法调用）：`x.type`、`stream.match(p)`
- `struct` / `enum` 的字段名声明：`struct Token { type: Kind; }`
- `struct` 字面量和 `struct` 模式中的字段名：`Ev { match: 1 }`、`Ev { match: m } => ...`
- 具名参数的名称：`f(type: 1)`。具名参数指称一个形参，而形参名不能是保留关键字，所以实际上这里只会出现上下文关键字。

`self` 是例外，即使在这些位置也不能作为标识符。在其他位置，要把关键字用作名称，请把它写成原始标识符 `r#match`（[1.4.4](104-identifiers.md#原始标识符)）。

```nabi
struct Token { type: Kind; for: str; }   // OK: field name
let k = tok.type;                         // OK: after .
make(type: Kind::Ident);                  // OK: named argument (contextual keyword)
struct S { self: i32; }                   // ERROR: self is the exception
```

## 1.5.6. 为以后使用而保留的关键字

1.5.1 的最后一行 `macro`、`defer`、`dyn`、`move`、`gen` 和 `do` 是目前尚无任何语法使用的保留关键字，只有闭包捕获列表中的 `move`（`[move x]`，[4.6.11](../004-functions/406-capture-semantics.md#4611-移动捕获move-x-捕获列表)）除外。保留它们，是为了让以后的版本能够在不破坏代码的前提下为它们赋予语法，也是为了让从其他语言带过来的写法得到一个说明 Nabi 应当怎样写的错误：

| 关键字  | 保留用途     | Nabi 目前的写法        |
|---------|--------------|------------------------|
| `macro` | 宏项         | 元块的 `@macro`（[6.8](../006-meta-system/608-meta-blocks.md)） |
| `defer` | 在作用域结束时运行的语句 | `using`（[3.17](../003-expressions/317-using-expression.md)）或 `finally`（[3.14](../003-expressions/314-error-control-flow.md)） |
| `dyn`   | 无：保留它是为了让 Rust 的 `dyn Trait` 得到错误 | `any Trait`（[2.2](../002-types/202-any-slot.md)、[2.12](../002-types/212-trait-as-type-banned.md)）；`dyn Trait`、`&dyn Trait` 和 `Box<dyn Trait>` 是编译错误，其帮助信息写出 `any Trait` |
| `move`  | 捕获列表之外无：保留它是为了让 Rust 的 `move \|x\| ..` 得到错误 | `[move x]` 捕获列表（[4.6.11](../004-functions/406-capture-semantics.md#4611-移动捕获move-x-捕获列表)）；`move \|..\|` 是说明这一点的编译错误 |
| `gen`   | 生成器形式   | `yield fn`、`yield { .. }`、`yield \|\| ..`（[4.11](../004-functions/411-generators.md)） |
| `do`    | 在循环体之后检测条件的循环 | `loop { ..; if !cond { break } }`（[3.13](../003-expressions/313-loop-expressions.md)） |

与所有保留关键字一样，它们在 1.5.5 的名称位置之外不能作为名称；拼写与之相同的标识符是编译错误，其帮助信息会给出原始标识符（[1.4.4](104-identifiers.md#原始标识符)）。

```nabi
fn gen(n: i32) -> i32 { n }     // ERROR: `gen` is a reserved keyword, not a name
fn r#gen(n: i32) -> i32 { n }   // OK
let s = Step { move: 1 };       // OK: a field name (1.5.5)
let m = s.move;                 // OK: after .
```

元块的 `@macro`（[6.8](../006-meta-system/608-meta-blocks.md)）是元语法，不是对该关键字的使用。
