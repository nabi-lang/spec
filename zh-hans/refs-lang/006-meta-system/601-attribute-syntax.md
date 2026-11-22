# 6.1. 属性语法

本节定义属性的词法形式、语法形式及其附着规则。属性遵循以下语法：

```
attribute       ::= outer_attr | inner_attr
outer_attr      ::= "#" "[" attr_body "]"
inner_attr      ::= "#" "!" "[" attr_body "]"
attr_body       ::= path attr_args?
attr_args       ::= "(" attr_arg ("," attr_arg)* ","? ")"
                  | "=" literal
attr_arg        ::= literal | identifier | attr_body | identifier "=" literal
```

某些内置属性的实参有各自的语法，在对应的节中给出：`#[niche]` 接受负值、区间以及 `i32::MIN` 这样的常量（[2.7](../002-types/207-niche-optimization.md)），`#[literal]` 接受词法单元模式（[6.5](605-literal-magic.md)）。

## 6.1.1. 引导符

属性由[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)中定义的两种词法单元引导。

### `#[...]` 外部属性

外部属性附着在紧随其后的项上。例如：

```nabi
#[derive(Debug)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }
```

`#[...]` 附着在其后的项上。

### `#![...]` 内部属性

内部属性附着在包含它的项或模块上：

```nabi
#![cfg(target_os = "linux")]      // applies to this whole module / file

mod m {
    #![allow(unused)]              // applies to module m itself
    fn helper() { ... }
}
```

内部属性只出现在文件开头或 `mod { .. }` 主体的开头。函数体中不接受内部属性，应改为在函数上放外部属性。外部与内部的区分沿用[词法结构](../001-lexical/102-lexical-structure.md)中 `///`（外部文档注释）与 `//!`（内部文档注释）的模式。

## 6.1.2. 主体形式

属性主体有三种形式：`#[name]`、`#[name(args)]` 和 `#[name = value]`。

### 仅名称

最简单的形式只使用一个名称：

```nabi
#[test]
#[inline]
#[cold]
```

仅名称形式用作布尔标志或标记。

### 带实参

带实参的形式也很常见：

```nabi
#[derive(Debug, Clone, PartialEq)]
#[repr(C)]
#[link("m")]
#[inline(always)]
#[cfg(target_os = "linux")]
#[infix("+")]
```

这是 `name(args)` 形式，实参之间用逗号分隔。每个实参可以是标识符、字面量、键值对（`like = "+"`）或嵌套的属性形式（`derive(...)` 中的 `Debug`）。

### 键值

最后是键值形式：

```nabi
#[doc = "This function ..."]
#[link_name = "external_symbol"]
```

这是 `name = value` 形式，其中值是一个字面量。它看起来与实参形式中的键值写法相同，区别在于整个主体就是一个键值对。

## 6.1.3. 可以带属性的项

属性可以附着在以下位置：

| 项                                  | 示例                                    |
|-------------------------------------|-----------------------------------------|
| 函数 / 方法                         | `#[inline] fn f() { ... }`              |
| `struct` / `enum` / `union`         | `#[derive(PartialEq, Eq)] struct S { ... }`（在 `union` 上 derive 是错误） |
| `trait` / `impl`                    | `#[deprecated] trait T { ... }`         |
| `const` / `static`                  | `#[deprecated] const X: i32 = 0;`       |
| 模块                                | `#[cfg(test)] mod tests { ... }`        |
| 项内的字段 / 变体（包括元组变体的字段） | `struct S { #[deprecated] x: i32; }`、`enum E { V(#[cfg(x)] i32) }` |
| trait 方法 / 关联项                 | `#[infix("+")] fn add(...);`            |
| `use` / `extern`                    | `#[link("m")] extern "C" { ... }`       |
| 语句（仅 `#[cfg]`）                 | `#[cfg(test)] my_test_call();`          |
| match 分支（仅 `#[cfg]`）           | `#[cfg(debug_assertions)] Op::Trace => trace(),` |
| 结构体字面量的字段（仅 `#[cfg]`）   | `S { #[cfg(x)] extra: 1, .. }`        |
| 函数参数（仅 `#[cfg]`）             | `fn f(#[cfg(x)] extra: i32) { ... }`    |

各位置上哪些属性有意义，取决于每个属性的定义；例如 `#[derive]` 只适用于类型，`#[inline]` 只适用于函数。

## 6.1.4. 多个属性

同一个项上可以附着多个属性。

```nabi
#[derive(Debug, Clone)]
#[repr(C)]
#[deprecated]
struct OldPoint { x: f64; y: f64; }
```

每个 `#[...]` 可以各占一行，也可以为了可读性把多个写在同一行。

```nabi
#[inline] #[cold] fn rarely_called() { ... }
```

## 6.1.5. 属性何时生效

属性何时生效因属性而异。

- 编译器识别的属性（如 `#[inline]`、`#[derive]`）由编译器在编译期直接处理。
- 魔法属性（如 `#[lang]`、`#[literal]`、`#[infix]`）由编译器结合语言含义进行处理。
- 用户定义的属性（来自元块）在模板展开阶段生成代码。

所有这些处理都在解析之后紧接着的元展开阶段完成。具体流程在[模板记号与展开](609-template-tokens.md)中定义。

## 6.1.6. 未知属性

编译器可能遇到它不认识的属性：

```nabi
#[mystery_attribute]
fn f() { ... }
```

既不是内置属性、也没有被任何元块定义或检测（包括 `@field.has_attr(x)`）的属性是编译错误，因此拼写错误永远不会被悄悄忽略。内置属性会检查其放置位置和实参形状。

## 6.1.7. 属性与卫生性

当附着在项上的属性导致代码生成时，生成代码中的标识符遵循 [6. 元系统](README.md)的卫生性规则，因此不会与用户代码的命名空间冲突。详细的卫生性规则在[模板记号与展开](609-template-tokens.md)中定义。

## 6.1.8. 同一词法单元的多重映射

同一个方法上可以附着多个属性：

```nabi
#[infix("⊕", like = "+")]
#[infix("⊞", like = "+")]
fn join(self, rhs: Self) -> Self;
```

这样两个词法单元（`⊕`、`⊞`）都分派到同一个方法。`≤` 这样固定的 Unicode 拼写不需要别名：词法分析器把它们读作对应的 ASCII 词法单元，注册这样的拼写（`#[infix("≤")]`）是编译错误（[6.7.4](607-operator-magic.md#674-unicode-与多个别名)）。

## 6.1.9. 属性与可见性及其他关键字的交互

属性可以与可见性关键字（`pub`、`pub(pkg)` 等）和函数限定符（`async`、`yield` 等）以任意顺序组合。

```nabi
#[inline]
pub fn f() { ... }

pub
#[inline]
fn g() { ... }                     // same meaning

pub async fn h() { ... }
#[inline] pub async fn h() { ... }
pub #[inline] async fn h() { ... }
```

惯用的顺序是属性 → 可见性 → 限定符（`#[inline] pub async fn ...`），但规范不强制顺序。
