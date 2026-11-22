# 5.7. 路径

由 `::` 词法单元（[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)）连接的标识符序列是一条路径，指示模块树中的一个位置。本文档把路径定义为对模块和项的引用。

[类型路径](../002-types/213-type-paths.md)讨论类型位置的路径；本文档讨论指向值和模块的路径。两者形式相同。

```
path            ::= path_root ("::" path_segment)*
path_root       ::= "pkg" | "sup" | "::" identifier | identifier
path_segment    ::= identifier
                  | identifier "::" "<" type_arg_list ">"      // turbofish
type_arg_list   ::= type_expr ("," type_expr)* ","?
```

上面的语法只定义值位置和模块位置路径的 turbofish（`name::<T>`）。类型位置中写作 `name<T>` 的泛型由[类型路径](../002-types/213-type-paths.md)中的 `type_path` 定义。

## 5.7.1. 路径的起始标识符

路径的第一个标识符是下列之一：

| 起始词法单元          | 含义                                                  |
|----------------------|----------------------------------------------------------|
| `pkg`                | 从当前包的根开始                     |
| `sup`                | 从父模块开始                                   |
| `std`、`core`、`bare` | 标准库层级，除非有局部名称占用了它    |
| `::`                 | 包：一个层级或该包的某个依赖，越过任何局部名称 |
| 普通标识符  | 当前可见的名称（子模块、已 `use` 的名称）     |

`pkg` 与可见性关键字 `pub(pkg)` 同词干，`sup` 是可见性 `pub(sub)` 的**镜像**方向（sub 指后代，sup 指祖先）。两者只有紧跟 `::` 时才是路径根；在其他位置它们是普通名称（[1.5.4 上下文关键字](../001-lexical/105-keywords.md)）。层级名称 `std`、`core` 和 `bare` 是普通标识符（[5.7.4](#574-std-core-bare-the-standard-library-tiers)）。

`self::` 不是路径起点。同一模块的子模块和项可以通过普通标识符访问（`child::item`）。

名称解析不会隐式查找祖先模块（例外：`extern` 函数，[5.9.4](509-extern-block.md#name-space-of-extern-functions)）。普通标识符只在当前模块的项、子模块和用 `use` 导入的名称（包括隐式的 `use core::prelude::*`）中查找。父模块或更高层模块的项必须用 `sup::` 指名或用 `use` 导入。

查找顺序如下；第一个匹配者胜出。

1. 当前模块的项和子模块，以及包自身依赖的名称（[5.7.14](#5714-paths-between-packages)；同时属于两者的名称是错误（ERROR））
2. 标准库层级（`std::fs`），除非当前模块的某个具名 `use` 占用了该名称
3. 当前模块的 `use`。具名导入优先于 glob（`*`、`**`）。
4. 隐式的 `use core::prelude::*`（标准库自身的模块：`use core::**`）
5. 对于期望枚举类型的位置上的单独名称，该枚举的一个变体（[2.4.8](../002-types/204-enum.md#248-bare-variants-from-the-expected-type)）

函数无论是否泛型，都共享同一个值名称空间。因此最近的声明胜出，与种类无关（当前模块中的非泛型 `max` 会隐藏 core 的泛型 `max`）。

```nabi
// inside pkg::a::b
helper();              // ERROR: no implicit search even if helper is in pkg::a
sup::helper();         // OK
```

## 5.7.2. `pkg::`：从包根开始

```nabi
pkg::geom::Point
pkg::config::DEFAULT_TIMEOUT
pkg::main_module::run
```

`pkg::` 从包的根模块（`src/main.nabi` 或 `src/lib.nabi`）开始，在包内任何位置写法都相同。它是大型代码库中进行安全绝对引用的标准形式。

## 5.7.3. `sup::`：从父模块开始

```nabi
sup::sibling_function
sup::parent_const
```

`sup::` 从当前模块的父模块开始，因此适合引用同级模块和父模块的项。要向上多级，链式写作 `sup::sup::...`。

```nabi
sup::sup::ancestor_item
```

本项目的风格是只在内联的 `mod name { }` 中写 `sup::`（如测试模块中的 `use sup::*;`），其他地方都写从包根开始的路径（`pkg::`）：警告 lint `sup_path`（[6.2](../006-meta-system/602-built-in-attributes.md)）会报告其余的 `sup::` 路径和 `pub(sub)`，其附注给出应写的 `pkg::` 路径。

## 5.7.4. `std::`、`core::`、`bare::`：标准库层级

```nabi
core::collections::List
core::iter::Iterator
bare::ptr::Ptr
std::fs::File
```

标准库是三个包，每个层级一个（[0.5 标准库层级](../000-meta/005-standard-libraries-tiers.md)），每个层级的名称是其路径的根。

| 路径           | 含义                              |
|----------------|--------------------------------------|
| `bare::*` | 裸、不安全区域                    |
| `core::*` | 基础、常用 trait    |
| `std::*`  | 操作系统集成、I/O 等            |

这三个包随工具链一起提供，编译器自动构建并链接它们（[0.5.4](../000-meta/005-standard-libraries-tiers.md#054-distribution-form-and-linking-mechanism)）；它们不是用户包的依赖，也不需要 `--extern`。其他任何包都不能命名为 `std`、`core` 或 `bare`（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）。

除此之外，层级名称是普通名称。模块、项或依赖别名都可以叫 `std`、`core` 或 `bare`；在声明或导入它的模块中，该名称指的是局部的那个，就像任何遮蔽其他名称的名称一样。前导 `::` 总是指名工具链的库（或某个依赖）：`::std::fs::File` 越过局部的 `std` 到达该层级。

```nabi
mod std { pub fn f() -> i32 { 1 } }

fn main() {
    let a = std::f();                    // the module above
    let p = ::std::path::Path::empty();  // the library's std
}
```

`bare` 和 `std` 不会自动导入，需要显式导入。相比之下，`core` 的 prelude 通过 [5.5 use](505-use.md) 中定义的隐式 `use core::prelude::*;` 自动可见。

## 5.7.5. 以普通标识符开始

```nabi
foo::bar::baz                   // OK if foo is a currently visible name
geom::Point                      // if the geom module is a child or imported
Point                            // a single name, resolved automatically
```

如果第一个标识符是当前模块中可见的名称，就可以直接使用。这里的可见名称指子模块或用 `use` 导入的项。

## 5.7.6. 路径各段的含义

```nabi
pkg::geom::shapes::Circle
└──┘  └──┘  └────┘   └────┘
root  mod   module   item
```

每一段是一个模块，或者（对最后一段而言）一个项。最后一个标识符是最终指示的项；前面的各段是路径上经过的模块或类型。

## 5.7.7. 各类项的路径

| 指示对象                                   | 形式                  |
|----------------------------------------------|-----------------------|
| 函数                                     | `path::function_name` |
| 类型（`struct` / `enum` / `union` / `trait`） | `path::TypeName`      |
| `const`                                      | `path::CONST_NAME`    |
| `static`                                     | `path::STATIC_NAME`   |
| 模块本身                                | `path::module_name`   |
| 静态方法                                | `Type::method_name`   |
| 关联 `const`                           | `Type::CONST`         |
| 关联类型                              | `T::AssocType`        |

这些与[类型路径](../002-types/213-type-paths.md)中访问关联项的形式相同。

## 5.7.8. 表达式位置与类型位置

同一种路径形式既可以用在表达式位置（指示值），也可以用在类型位置（指示类型）。

```nabi
// expression position
let f = pkg::geom::distance;          // function value
let p = pkg::geom::Point::origin();   // static method call
let v = pkg::config::DEFAULT;           // const value

// type position
let p: pkg::geom::Point = ...;
fn make() -> pkg::geom::Point { ... }
```

两种位置的区别在于泛型实参是否用 turbofish（`::<...>`）书写。表达式位置使用 turbofish；类型位置使用普通的 `<...>`。详情请见[类型路径](../002-types/213-type-paths.md)。

## 5.7.9. 路径与 glob 的组合

与 [`use`](505-use.md) 的导入形式结合，可以使用多种导入写法。

```nabi
use pkg::geom::{Point, Circle, distance};
use sup::{a, b as alias};
use foo::*;
```

`*`（glob）和 `**`（递归 glob）的行为如 [`use`（导入）](505-use.md)中所定义。

## 5.7.10. 空路径与前导 `::`

空路径（只有 `::` 而没有标识符）是编译错误。

前导 `::` 从包开始：它后面的标识符是一个标准库层级或该包的一个依赖（`::std::fs`、`::json::parse`），无论是否存在同样拼写的局部名称。它后面的任何其他名称都是错误（ERROR）（`::A`）；本包的项写作 `pkg::A`。

## 5.7.11. 用 `use` 缩短

用 `use` 导入的名称可以在当前模块中以短名称使用。

```nabi
use pkg::geom::Point;

fn f() {
    let p = Point { ... };          // short name
    let q: Point = ...;
}
```

在模块顶部用 `use` 汇集常用的项，可以让代码主体保持简洁。

## 5.7.12. 路径何时解析

路径解析发生在编译期的名称解析阶段，对运行时行为没有影响（不过，动态槽上的方法分派另外在运行时进行；请见 [`any` 槽](../002-types/202-any-slot.md)）。

## 5.7.13. 动态引用模块（不提供）

```nabi
let m = "foo";
m::bar();                            // ERROR: no dynamic module references
```

路径的模块部分总是静态的，在编译期确定。不提供在运行时以变量接收模块并调用其中函数的功能。

这与[模块](504-modules.md)中模块树是静态结构的原则一致。

## 5.7.14. 包之间的路径

另一个包（依赖）的项通过以该包名开始的路径引用。

```nabi
use parser::Token;
let t = parser::lex(src);
```

- 依赖包的名称是一个顶层路径。该名称是 `nabi.nabion` 中的 `deps` 键，其中的 `-` 替换为 `_`（`nabi-syntax` → `nabi_syntax::..`），构建工具用 `nabic --extern name=..` 注册它（[9.2.3](../009-tools/902-aot-pipeline.md#923-dependency-resolution)、[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）。不需要 Rust 的 `extern crate` 那样的单独声明。
- 一个包只能看到它所依赖的包，并且使用它自己的 `deps` 给它们起的名称。构建中其他包的依赖不在作用域内：以 `"csvio": Dep::..(package: "csv")` 依赖 `csv` 的包写作 `csvio::..`，在那里 `csv::..` 是它自己的模块 `csv`，否则是错误（ERROR），其帮助信息提示把 `csv` 添加到该包的 `deps` 中。
- 既是包的依赖、又是当前模块子模块的名称是有歧义的：以它开始的路径是错误（ERROR），并指出两者。其帮助信息说明应写什么：依赖写 `::name::..`，模块写 `pkg::..::name::..`，或者在 `nabi.nabion` 中给依赖另起一个名称（用带 `package:` 的键指名该包）。
- 依赖包内的 `pkg::` 是该依赖自己的根。
- 正在构建的包的根上的模块，不能与构建中某个包同名（各包挂载在同一棵模块树的顶端）：这是错误，提示重命名该模块。
- 解析为正在构建的包本身的依赖是错误；包用 `pkg::` 指名自己的项。

### 包边界

下列每一项都是一个独立的包。可见性 `pub(pkg)` / `pub(sub)`（[5.6](506-visibility.md)）不跨越这些边界。

- 正在构建的包
- 每个依赖包
- 每个标准库层级（`bare`、`core`、`std`）

因此，依赖的私有、`pub(sub)`、`pub(pkg)` 项不能在其外部使用；只有 `pub` 项可以。
