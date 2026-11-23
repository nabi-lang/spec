# 5.4. 模块

子模块用 `mod` 关键字（[关键字](../001-lexical/105-keywords.md)）声明。本文档定义模块声明与文件系统布局之间的映射。

```
mod_decl        ::= "mod" identifier ";"
                  | "mod" identifier "{" item* "}"      // inline module
```

## 5.4.1. 模块声明

```nabi
mod foo;
mod parsing;
```

模块声明由 `mod` 关键字、模块名和 `;` 组成。模块名遵循[标识符](../001-lexical/104-identifiers.md)的形式，仅凭这一声明就会自动映射到同名的文件或文件夹。

## 5.4.2. 映射形式

出现 `mod foo;` 时，编译器在父模块主体所在的目录中查找下列两种形式之一。这些规则与 Rust 2018+ 相同。

### 形式 1：单个文件（`foo.nabi`）

没有子模块的模块只需要一个文件。

```
parent/
├── parent.nabi                <- where `mod foo;` appears
└── foo.nabi                   <- foo module body, no submodules
```

`foo.nabi` 的内容就是模块 foo 的主体。

### 形式 2：文件夹（`foo.nabi` + `foo/`）

有子模块的模块必须同时有主体文件（`foo.nabi`）和同名文件夹（`foo/`）。主体文件充当模块的主体，文件夹存放它的子模块。

```
parent/
├── parent.nabi                <- where `mod foo;` appears
├── foo.nabi                   <- foo module body
└── foo/                      <- foo's submodules
    ├── bar.nabi                   declared as `mod bar;` inside foo.nabi
    └── baz.nabi                   declared as `mod baz;` inside foo.nabi
```

这里 `foo.nabi` 是模块 foo 的主体，包含普通的项、`mod bar;` 这样的子模块声明、可见性等。`foo/` 目录中的每个 `.nabi` 文件都是 foo 的子模块，必须在主体中以 `mod name;` 显式声明。只有 `foo/` 而没有 `foo.nabi` 是错误。主体文件是必需的。

这一模型与 Rust 2018+ 的 `foo.rs` + `foo/` 结构完全相同；没有 `mod.nabi` 这样单独的主体文件名。

### 同名文件与文件夹的含义

父目录中同时存在同名的 `foo.nabi` 和 `foo/` 时，它们会被自动识别为文件夹形式；这不是冲突。

```
parent/
├── parent.nabi
├── foo.nabi                   <- foo body
└── foo/                      <- foo's submodules
    └── ...
```

总结：只有 `foo.nabi` 是单文件模块；`foo.nabi` 加 `foo/` 是文件夹模块；只有 `foo/` 是错误。

## 5.4.3. 子模块

只有文件夹形式的模块（`foo.nabi` + `foo/`）可以有子模块。在主体（`foo.nabi`）中写 `mod name;`，会把 `foo/name.nabi` 或 `foo/name.nabi` + `foo/name/` 映射为该子模块。

```nabi
// contents of foo.nabi
mod bar;
mod baz;

pub fn foo_helper() { ... }
```

该主体旁边必须存在下列文件：

```
foo.nabi                       <- module body (the code above)
foo/
├── bar.nabi                   <- foo::bar submodule
└── baz.nabi                   <- foo::baz submodule
```

子模块本身也可以采用单文件或文件夹形式。

```
foo.nabi
foo/
├── bar.nabi                   <- foo::bar (single-file)
├── baz.nabi                   <- foo::baz body
└── baz/                      <- foo::baz submodules
    ├── x.nabi                     foo::baz::x
    └── y.nabi                     foo::baz::y
```

这一模式可以递归适用于任意深度，每个模块独立选择单文件或文件夹形式。

### 单文件模块没有**映射到文件的**子模块

```nabi
// foo.nabi (single-file module, no foo/)
mod bar;            // ERROR: there is no foo/, so nowhere to map
```

如果需要映射到文件的子模块，创建同名文件夹（`foo/`），把子模块文件放进去。把单文件模块升级为文件夹模块，只需创建文件夹并移动子模块文件。

另一方面，内联子模块（`mod bar { ... }`）不需要文件映射，所以即使在单文件模块中也可以自由使用（[5.4.5 内联模块](#545-内联模块在主体中定义)）。

## 5.4.4. 包根

[源单元](../000-meta/003-source-units.md)中定义的 `src/main.nabi` 或 `src/lib.nabi` 是包根模块的主体文件，包的源目录 `src/` 充当根模块的文件夹。

根模块无需 `mod` 声明即可自动识别，其他每个模块都是从根开始的树中的一个节点。

```
my_pkg/
├── nabi.nabion
└── src/
    ├── lib.nabi                   <- root module body (declares `mod geom; mod parsing;`)
    ├── geom.nabi                  <- geom module    (single-file: geom.nabi)
    ├── parsing.nabi               <- parsing module body
    └── parsing/                  <- parsing's submodules (declared inside parsing.nabi)
        ├── lexer.nabi                 parsing::lexer
        └── parser.nabi                parsing::parser
```

根模块总是文件夹形式：`src/` 是它的文件夹，`lib.nabi`（库）或 `main.nabi`（可执行文件）是它的主体文件。根中的 `mod foo;` 映射到 `src/foo.nabi`；位于 `src/` 旁边的 `foo.nabi` 永远不是模块。`#[path(..)]`（5.4.6）从子模块文件通常所在的文件夹开始解析：对根中的 `mod` 是 `src/`，对 `parsing.nabi` 中的 `mod` 是 `src/parsing/`。

## 5.4.5. 内联模块：在主体中定义

在 `mod` 声明之后直接放上块主体，就定义了一个不需要单独文件的内联模块。

```nabi
mod inner {
    pub fn helper() -> i32 { 42i32 }
    const X: i32 = 100i32;
}

// use
let r = inner::helper();
```

内联模块写作 `mod name { ... }`，其主体可以包含任何项：函数、类型、trait、`const`、`static`，以及更多的 `mod`。它旁边不得存在同名的文件或文件夹；冲突是错误。

## 5.4.6. 自定义路径：`#[path("...")]`

```
path_attr       ::= "#" "[" "path" "(" string_lit ")" "]"
```

`#[path("...")]` 属性覆盖默认映射。

```nabi
#[path("custom_geom.nabi")]
mod geom;

#[path("../shared/shared.nabi")]
mod shared;
```

给定路径处的文件被映射为模块主体。路径相对于通常能找到 `name.nabi` 的目录。该模块的子模块放在文件旁边、以文件主名命名的文件夹中：对 `#[path("f.nabi")]` 是 `f/`。目标文件不能命名为 `mod.nabi`，因为 Nabi 没有 `mod.nabi` 主体形式（[5.4.4](#544-包根)）；这是错误。`#[path(..)]` 是唯一的形式。它在日常使用中很少出现，用于构建系统和生成代码等特殊情况。

## 5.4.7. 模块内的项

每个模块都可以包含[概述](README.md)中介绍的项。

```nabi
mod geom {
    struct Point { ... }
    fn distance(a: Point, b: Point) -> f64 { ... }
    const PI: f64 = 3.14159f64;
    static CACHE: Mutex<Dict<str, f64>> = ...;
    impl Point { ... }
    trait Shape { ... }
    mod inner { ... }            // nested module again
}
```

模块只能包含项；可执行语句（顶层 `let`、表达式语句）只允许出现在入口模块 `main.nabi` 中（[4.16.6](../004-functions/416-entry-point.md#4166-模块只包含定义)）。所有这些项都是该模块的一部分，从外部引用它们既需要[可见性](506-visibility.md)关键字，也需要[路径](507-paths.md)。在函数体中声明的项（[3.9.1](../003-expressions/309-block-expressions.md#391-形式)）就一致性规则（coherence）而言属于该模块，但只能在其所在块内通过名称引用。

## 5.4.8. 模块的可见性

模块本身也可以指定可见性。

```nabi
pub mod public_api;            // usable from external packages
pub(pkg) mod internal;          // within the same package only

mod _private;                   // module-private: only from the same parent module
```

详细含义在[可见性](506-visibility.md)中定义。

## 5.4.9. 在模块内引用父模块、同级模块与子模块

引用其他模块中项的路径写法在[路径](507-paths.md)中定义。

```nabi
pkg::geom::Point             // from the package root
sup::sibling_mod::item        // through the parent module to a sibling
child_mod::item                // within the same module (an identifier suffices)
```

## 5.4.10. 模块声明可以出现的位置

`mod` 声明只能出现在模块级，即另一个 `mod` 的主体中或包根处。在函数体中声明 `mod` 是错误（[3.9.1](../003-expressions/309-block-expressions.md#391-形式) 列出了块可以声明的项）。

```nabi
fn f() {
    mod inner { ... }          // ERROR: no module declarations inside a function
}
```

这源于一个原则：模块树是在编译期确定的静态结构。

## 5.4.11. 把一个模块拆分到多个文件

要把一个主体拆分到多个文件，使用文件夹形式，并在主体（`foo.nabi`）中通过子模块重新导出各项。

```nabi
// foo.nabi
mod core;
mod ext;

pub use core::*;
pub use ext::*;
```

```
foo.nabi                       <- module body (re-exports only)
foo/
├── core.nabi                  <- core items
└── ext.nabi                   <- extension items
```

采用这一模式，用户以 `foo::item` 访问每个项，主体文件保持精简，实际代码分布在各子模块中。这与 Rust 的 `pub use` 重新导出模式相同。

## 5.4.12. 同一父模块中不能有重复模块

在同一个父模块中两次声明同名模块是错误。

```nabi
mod foo;
mod foo;                        // ERROR: duplicate
```
