# 6. 元系统概述

本部分定义 Nabi 的元系统：属性、魔法和元[块](../000-meta/007-glossary.md#block块)。元系统是实现[语言与标准库的边界](../000-meta/004-language-vs-standard-libraries.md)这一模型的核心机制：语言提供一个小巧的核心，由标准库填充其含义。

## 目录

- [6.1. 属性语法](601-attribute-syntax.md)
- [6.2. 内置属性](602-built-in-attributes.md)
- [6.3. `#[cfg]` 条件编译](603-cfg.md)
- [6.4. `#[lang(...)]` 魔法](604-lang-magic.md)
- [6.5. `#[literal(...)]` 魔法](605-literal-magic.md)
- [6.6. `#[keyword(...)]` 魔法](606-keyword-magic.md)
- [6.7. 运算符魔法](607-operator-magic.md)
- [6.8. 用户定义的元定义（元块与宏）](608-meta-blocks.md)
- [6.9. 模板记号与展开](609-template-tokens.md)
- [6.10. 反射](610-reflection.md)
- [6.11. 编译期文件嵌入](611-include.md)

## 三层结构

元系统分为三个层级：

```
attribute
  ├── plain attribute: hints / metadata for compiler / tools
  └── magic: channel where the library binds to language meaning

meta definition: user-defined code generator, one template engine (6.8)
  ├── derive / attribute / per-field  (triggered by #[..] on a type or item)
  └── macro                           (triggered by a call, name!(..))
      └── template token (@Self, @field, ..) + matcher (typed fragments)
```

### 属性

属性的样子如下：

```nabi
#[derive(Debug, Clone)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }

#[cfg(target_os = "linux")]
fn linux_only() { ... }
```

属性以 `#[...]` 或 `#![...]` 的形式附加在项上。编译器识别的内置属性（例如 `#[derive]`、`#[repr]`、`#[cfg]`）与用户定义的属性形式相同。详细形式定义在[属性语法](601-attribute-syntax.md)和[内置属性](602-built-in-attributes.md)中。

### 魔法

魔法是指连接语言与库的特殊属性。

```nabi
#[lang("slot_name")]            // connects a standard library item ↔ a compiler slot
#[literal("...")]                // registers literal syntax
#[keyword("name")]               // registers a new keyword
```

此外还有运算符魔法这一族。

```nabi
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]
```

这些魔法把 `1 + 2` 这样的语言词法单元连接到库 trait 的方法调用。它们在 [`#[literal]` 魔法](605-literal-magic.md)到[运算符魔法](607-operator-magic.md)各节中定义。

### 元块

元定义是代码生成器。其中三种形式让用户定义自己的 derive 或属性（取代 Rust 的 derive 过程宏和属性过程宏）；第四种是[宏](608-meta-blocks.md#6812-宏形式)，由调用 `name!(..)` 触发，根据调用方所写的内容生成代码（对应类函数宏的情形）。这四种形式共用同一个模板引擎。

```nabi
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {
        @for_fields {
            // per-field code generation
        }
    }
}
```

共有四种形式：由属性触发的 `#[impl_derive]`、`#[impl_attribute]` 和 `#[impl_derive_attribute]`，以及由调用触发的 `#[impl_macro(name)] @macro { .. }`（以 `name!(..)` 调用）。它们的写法都相同：`#[impl_<kind>(name)]`、可见性、一个 `@` 引导符和一个块。它们的主体用 `@Self`、`@field` 和 `@for_fields` 等模板记号编写代码生成规则，宏还增加了由带类型片段组成的匹配器（`@x: expr`）。不提供过程宏风格的词法单元流操作；仅凭模板记号就能获得足够的表达力。完整定义见[元定义](608-meta-blocks.md)。

## 核心原则

元系统建立在四条核心原则之上。

### 1. 不操作词法单元流

Nabi 没有类似 Rust 的 `syn` 或 `quote` 那样的、过程式地操作词法单元的宏。所有用户代码生成，无论是由属性触发的元块还是由调用触发的宏（[6.8.12](608-meta-blocks.md#6812-宏形式)），都只通过模式和模板进行：匹配器绑定输入的各个部分，模板转写它们。没有任何步骤会运行任意代码或改写原始的词法单元流。

这样做是为了让生成的代码保持在编译器、格式化工具和语言服务器能够追踪的状态。允许任意的词法单元变换会带来种种代价：不可预测、难以调试，以及工具需要应对的范围急剧膨胀。宏在遵守这一规则的前提下，增加了调用处的代码生成（这是其他形式做不到的）。

### 2. 魔法由标准库使用

`#[lang]`、`#[literal]`、`#[keyword]` 和运算符魔法主要由标准库使用；用户代码很少直接使用它们。

这些魔法是标准库与语言绑定从而产生含义的机制；[语言与标准库的边界](../000-meta/004-language-vs-standard-libraries.md)正是通过这一元系统实现的。

### 3. 宏的卫生性

元块中的代码生成是卫生的。模板所写的名字在定义处、即元块所在的模块中解析；由使用处传入的名字在使用处解析；模板的局部变量会被重命名，因此永远不会捕获或遮蔽用户的名字。展开所生成的项放在使用处的模块中，其名字在那里可见（[6.9.9](609-template-tokens.md#展开中的名字)）。

这防止了用户代码与元块之间不可预测的冲突。

### 4. 仅限编译期

所有元处理都在编译期完成。不支持在运行时注册新的 derive 或注入新的魔法。
