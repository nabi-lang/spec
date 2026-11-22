# 5.6. 可见性

为每个项（函数、类型、trait、模块、`const`、`static`）指定可见性，可以控制它暴露的范围。

```
visibility      ::= "pub" ("(" reach ")")?
reach           ::= "pkg" | "sub"
```

## 5.6.1. 四个级别

可见性有四个级别：

| 关键字    | 含义                                                   |
|------------|-----------------------------------------------------------|
| （无）     | 模块私有。仅在同一模块内。              |
| `pub(sub)` | 仅在同一模块树的子模块内。           |
| `pub(pkg)` | 仅在同一包内。                             |
| `pub`      | 外部包也可以访问。                              |

最窄的范围是默认的私有；最宽的是 `pub`。工具链自己的包之间还共享一些标记为 `#[internal]` 的 `pub` 项，其他包不能使用它们（[6.2.16](../006-meta-system/602-built-in-attributes.md#6216-internal-shared-by-the-toolchains-packages)）；这一标记不是可见性级别。

`pkg` 和 `sub` 是上下文关键字，只在 `pub(...)` 的括号内充当限定符；在其他位置它们是普通标识符（请见 [1.5.4 可见性限定符](../001-lexical/105-keywords.md#visibility-qualifiers-pkg--sub)）。

## 5.6.2. 默认：无可见性关键字

没有可见性关键字时，项是模块私有的，只在同一模块内可见。

```nabi
mod m {
    fn helper() -> i32 { 42i32 }   // module-private (default)
    pub fn api() -> i32 { helper() }
}

let r = m::api();                   // OK
let h = m::helper();                // ERROR: module-private
```

私有的 `mod` 对声明它的父模块以及该父模块的子模块可见。

## 5.6.3. `pub`：最宽的暴露

要获得最宽的暴露，加上 `pub`。

```nabi
pub fn calculate(x: i32) -> i32 { ... }
pub struct Point { pub x: f64; pub y: f64; }
pub trait Display { ... }
```

这样声明的项可以用 `use` 导入，并在其他包中使用。

## 5.6.4. `pub(pkg)`：包内

`pub(pkg)` 表示在包内公开。

```nabi
pub(pkg) fn internal_helper() -> i32 { ... }
```

它对同一包中的任何模块可见，但对外部包不可见。依赖包和每个标准库层级也都是独立的包（[5.7.14](507-paths.md#5714-paths-between-packages)）。

这一级别用于包内部的库工具：那些不是公开 API、但必须在模块之间共享的函数和类型。

## 5.6.5. `pub(sub)`：子模块

`pub(sub)` 用于只在模块树的一个分支内共享。

```nabi
mod parent {
    pub(sub) fn shared_within_subtree() -> i32 { ... }

    pub mod child {
        pub fn use_it() -> i32 { sup::shared_within_subtree() }   // OK
    }
}
```

它只在 `parent` 及其所有后代模块中可见，在 `parent` 的祖先或同级模块中不可见。

这一级别用于一组模块内部的工具，只在模块树的一个分支内共享。

## 5.6.6. 级别之间的包含关系

可见性构成一个单调递增的格。

```
pub  ⊇  pub(pkg)  ⊇  pub(sub)  ⊇  private
```

`pub(sub)` 的包含关系只在相关模块之间（即定义模块及其后代之间）才有意义。

- `pub` 对每个调用方可见。
- `pub(pkg)` 只在同一包内可见。
- `pub(sub)` 只在同一模块树内可见。
- 私有只在同一模块内可见。

## 5.6.7. glob 导入与可见性

glob 导入（`*`、`**`，包括隐式的 `use core::prelude::*`）只导入从导入方模块可见的项。只有可达范围包含该模块的项才会被引入，不可见的项既不隐藏名称，也不引起冲突。没有按项阻止 glob 的标记；要收窄对外的接口面，创建一个单独的 `prelude` 模块，并用 `pub use` 精选其内容。

## 5.6.8. `struct` 字段的可见性

每个字段都可以有自己的可见性。

```nabi
struct Account {
    pub id: i64;                    // readable externally
    pub balance: f64;               // readable externally
        secret: str;                // module-private
}
```

字段的可见性独立于 `struct` 本身的可见性。

```nabi
pub struct Counter {                // the struct itself is visible externally
        count: i32 = 0i32;          // the field is module-private (default)
}

let c = Counter { count: 0i32 };    // ERROR: private field, cannot construct directly from outside
let c = Counter::new();             // OK: factory method (if any)
```

这是暴露类型、同时阻止直接实例化的标准模式（newtype、不透明类型）。

结构体字面量检查它所写字段的可见性。只有当使用处看不到的字段具有默认值（[2.3.3](../002-types/203-struct.md#233-field-default-values)）时，字面量才可以省略该字段；没有默认值的私有字段会使其作用范围之外的每个字面量都成为错误，此时要用其所在模块的函数构造该值。

```nabi
pub struct Opts {
    pub depth: i32 = 4;
        seed: i64 = 7;              // private, with a default
}
pub struct Keyed {
    pub label: str = "k";
        key: i64;                   // private, no default
}

let o = Opts { depth: 1 };          // OK: `seed` takes its default
let k = Keyed { label: "x" };       // ERROR outside the module: `key` has no default
```

### 元组结构体的字段

元组结构体的每个字段也都有自己的可见性，默认是私有的。

```nabi
pub struct S(pub i32, i32);         // field 0 is pub, field 1 is private
```

- 用 `S(a, b)` 构造要求每个字段在使用处都可见。只要有一个字段是私有的，在其可见范围之外就要使用构造函数（`NonZero::new`）。
- 读取或赋值 `x.N` 要求字段 N 可见。
- 在模式 `S(p0, p1, ..)` 中，子模式不是 `_` 的每个位置都必须可见。具名字段的结构体模式同样如此（只检查涉及的字段）。
- 紧跟在 `pub` 之后的 `(` 被读作可见性限制（`pub(pkg)` 等）。要公开元组类型的字段，再加一对括号：`struct S(pub ((i32, i32)));`。

## 5.6.9. `enum` 变体的可见性

变体没有单独的可见性；它们遵循 `enum` 的可见性。在变体前写可见性关键字是编译错误。

```nabi
pub enum Shape {
    Circle(f64),                    // pub (follows Shape)
    Square(f64),
}

pub enum Bad {
    pub A,                          // ERROR: no visibility on variants
}
```

变体的载荷字段也没有逐字段的可见性；它们遵循 `enum`。与 `struct` 字段不同，隐藏单个变体会使外部 `match` 无法做到穷尽，因此有这一规则。按照这一规则，标准库的 `pub enum Result<T, E> { Ok(T), Err(E) }` 使 `Ok` / `Err` 公开。

## 5.6.10. `impl` 方法的可见性

固有 `impl`（[`impl`](../002-types/209-impl.md)）的方法默认是模块私有的；要暴露它们，显式标记为 `pub`。

```nabi
impl Account {
    pub fn balance(self) -> f64 { self.amount }
        fn _internal_check(self) { ... }
}
```

trait `impl` 的方法遵循 trait 的可见性，不能单独标记。

## 5.6.11. `use` 的可见性

通过 [`use`](505-use.md) 的导入也有可见性。

```nabi
pub use foo::bar;                   // re-export: visible externally too
pub(pkg) use foo::bar;              // within the package only

use foo::bar;                       // module-private (default)
```

重新导出与 [`use`](505-use.md) 的可见性一节结合，决定对外的接口面。

## 5.6.12. 可见性的位置

可见性关键字作为项声明的第一个词法单元出现。

```nabi
pub fn f() { ... }
pub(pkg) fn g() { ... }
pub struct S { ... }
pub mod m { ... }
pub const X: i32 = 0;
pub static Y: i32 = 0;
pub use foo::bar;
pub trait T { ... }
```

它可以加在 `fn`、`struct`、`enum`、`union`、`trait`、`impl`（不加在 `impl` 本身，而加在每个方法上）、`mod`、`const`、`static`、`use` 和 `type`（别名）的每种声明形式上。

## 5.6.13. 可见性与名称解析

与 [`use`](505-use.md) 结合时，只有同时满足下列两点，项才在当前模块中可见。

1. 项的可见性能到达当前模块。路径中的每个模块段也必须可见（`a::b::f` 要求 `a`、`b` 和 `f` 都可见）。
2. 项是直接可见的（子模块或用 `use` 导入的名称）。

两点都满足时，就可以用短名称使用该项。

对于可见性不足的项，编译器诊断会报告它不可见的原因，并指出能到达该使用处的可见性：在其包中任何位置使用需要 `pub(pkg)`，在该项下层的模块中使用也是如此（私有项对子模块不可见），在另一个包中使用需要 `pub`。

元模板或编译器生成的代码，从其定义所在的模块进行检查，而不是从展开处检查（[6.9.9](../006-meta-system/609-template-tokens.md#names-in-an-expansion)）：模板可以调用它自己模块的私有项和它所在包的 `pub(pkg)` 项，而使用处传入的内容在使用处检查。

```text
error[E0034]: function `foo::secret` is private to module `foo` and cannot be used from `<root>` (add `pub(pkg)` to expose it)
```
