# 5. 变量、模块、可见性与包概述

本部分讨论 Nabi 的名称空间。目标是定义变量在哪里可见、模块如何组织、可见性级别如何起作用，以及包与包之间的边界在哪里。

## 目录

- [5.1. 变量声明](501-variable-declaration.md)
- [5.2. `const`](502-const.md)
- [5.3. `static`](503-static.md)
- [5.4. 模块](504-modules.md)
- [5.5. `use`（导入）](505-use.md)
- [5.6. 可见性](506-visibility.md)
- [5.7. 路径](507-paths.md)
- [5.8. 重新导出（`pub use`）](508-re-export.md)
- [5.9. `extern` 块（FFI 声明）](509-extern-block.md)

## 名称空间的四个层级

名称空间有四个层级：

```
package
  └── module
        └── item
              └── local variable
```

包含关系自上而下。一个包有多个模块，一个模块有多个项（函数、类型、trait、`const`、`static`），一个项（尤其是函数体）有多个局部变量。

各层级的边界规则在下列文档中定义。

## 包：最大的单位

[源单元](../000-meta/003-source-units.md)中定义的包是名称空间的最外层。每个包由基于内容的哈希标识。

同一个包中的所有项属于同一个代码权限域。这就是 [`impl`](../002-types/209-impl.md) 的孤儿规则按包适用的原因。

## 模块：包内的组织单位

一个包有多个模块。每个模块要么是单个文件（`foo.nabi`），要么是一个主体文件加上同名文件夹（`foo.nabi` + `foo/`）。这与 Rust 2018+ 的模型相同。详细规则在[模块](504-modules.md)中定义。

```
my_pkg/
└── src/
    ├── lib.nabi                  <- root module body
    ├── geom.nabi                 <- geom module    (single-file: geom.nabi)
    ├── parsing.nabi              <- parsing module body
    └── parsing/                 <- parsing's submodules
        ├── lexer.nabi                parsing::lexer
        └── parser.nabi               parsing::parser
```

## 项：模块内的具名定义

每个模块都有项：函数、类型、trait、`impl` 块、`const`、`static`，以及其他模块声明。

[函数声明](../004-functions/401-function-declaration.md)、[`struct`](../002-types/203-struct.md)、[`enum`](../002-types/204-enum.md)、[`trait`](../002-types/208-trait.md) 和 [`impl`](../002-types/209-impl.md) 的每个定义都是模块内的一个项。

## 局部变量：函数与块内的变量

在函数、块和其他表达式内部，局部变量用 `let` 关键字（[关键字](../001-lexical/105-keywords.md)）声明。

```nabi
fn area(r: f64) -> f64 {
    let pi = 3.14159f64;          // local variable
    pi * r * r
}
```

局部变量的可见范围按[块表达式](../003-expressions/309-block-expressions.md)的块确定。详细规则在[变量声明](501-variable-declaration.md)中定义。

## 可见性：暴露项

每个项都有可见性。可见性级别（[可见性](506-visibility.md)）决定该项在多大范围内可见。

```
private (default)    within the same module only
pub(sub)             within submodules only
pub(pkg)             within the same package only
pub                  also from external packages
```

`#[internal] pub` 项是工具链自己的包之间共享的 `pub` 项；其他包不能使用它们（[5.6](506-visibility.md)、[6.2.15](../006-meta-system/602-built-in-attributes.md#6215-internal-由工具链的各个包共用)）。

## 名称解析规则

确定代码中的某个名称指向哪个项，称为名称解析。

名称解析考虑以下几点。第一，当前模块中局部可见的项是候选。第二，用 `use`（[`use`](505-use.md)）导入的外部项也是候选。第三，还会查找自动导入的 prelude 和标准库层级（[标准库层级](../000-meta/005-standard-libraries-tiers.md)）。最后进行可见性检查，只有可见的项才保留为候选。

名称解析在编译期进行。动态槽上的方法分派在运行时进行，但名称本身的解析总是在编译期完成。
