# 5.8. 重新导出（`pub use`）

把 `use` 与可见性关键字结合，就构成重新导出：导入的项从当前模块再次暴露出去。语法如下：

```
re_export       ::= visibility "use" use_tree ";"
```

## 5.8.1. 形式

重新导出由可见性关键字、`use`、路径和分号组成：

```nabi
pub use foo::bar;
pub(pkg) use pkg::internal::helper;
pub(sub) use sup::shared;
```

把 `pub` 或其他可见性关键字与 `use`、路径和 `;` 一起写，就会以该可见性从当前模块再次暴露导入的项。

## 5.8.2. 含义

```nabi
// root
pub use deep::Item;             // Item is pub at the root too

// pkg::deep::Item
mod deep {
    pub struct Item { ... }
}
```

于是外部包可以通过两条路径访问同一个项。

```nabi
use my_pkg::deep::Item;          // original path: OK
use my_pkg::Item;                // re-export path: OK
```

同一个项通过两条路径可见。

## 5.8.3. 用途

### 外观模式

用于隐藏内部结构，对外提供扁平的 API。

```nabi
// root: flat for external users
pub use core::geom::Point;
pub use core::geom::distance;

// pkg::core::geom::Point
mod core {
    pub mod geom {
        pub struct Point { ... }
    }
}
```

外部用户访问 `my_pkg::Point`，无需了解内部结构（`core::geom::`）。

### 构建 prelude

把常用的项汇集在一处的 prelude 模块，也以同样的方式构建。

```nabi
// pkg::prelude
pub use pkg::types::{Point, Circle, Polygon};
pub use pkg::ops::{distance, area};
```

用户只需一行 `use my_pkg::prelude::*;`，就能一次引入常用的项。

### 版本兼容

如果内部项移动了位置，但保留了重新导出路径，旧路径对外部用户仍然可用。

```nabi
// v1: pkg::Item
pub struct Item { ... }

// v2: moved
pub mod core { pub struct Item { ... } }
pub use core::Item;             // keeps the v1 path
```

## 5.8.4. 可见性级别的组合

与可见性关键字结合的重新导出：

```
pub use         → exposed to external packages
pub(pkg) use    → exposed within the package
pub(sub) use    → exposed to submodules
use             → module-private (not a re-export)
```

可见性的顺序为私有 < `pub(sub)` < `pub(pkg)` < `pub`。原项的可见性必须 ≥ 重新导出的可见性；比原项更宽的 `pub use` 是错误。

```nabi
pub use inner::private_fn;       // ERROR: cannot expose a private item as pub

mod inner {
    fn private_fn() { ... }      // private
}
```

glob 重新导出（`pub use mod::*`、`pub use mod::**`）的行为不同：原可见性窄于重新导出可见性的项会被悄悄过滤掉，这不是错误（[5.8.12 glob 重新导出](#5812-glob-重新导出与可见性)）。

## 5.8.5. 多项重新导出

可以一次重新导出多个项，也可以指定别名。

```nabi
pub use foo::{a, b, c};
pub use foo::{
    Point,
    Circle,
    distance as dist,
};
```

[`use`](505-use.md) 的多项导入和别名形式原样适用。

## 5.8.6. glob 重新导出

要一次重新导出整个模块或子树：

```nabi
pub use foo::*;
pub use foo::**;
```

整个模块或其下的子树会一次性暴露出去。

> [!WARNING]
> 名称冲突在重新导出处就是错误。

## 5.8.7. 与别名结合

别名可以缩短对外可见的名称。

```nabi
pub use foo::very_long_name as short_name;
```

对外只有短名称（`short_name`）可见；要使用原名称（`very_long_name`），需要单独导入。

## 5.8.8. 重新导出模块本身

不仅可以重新导出项，也可以重新导出模块本身。

```nabi
pub use foo::geom;               // re-exports the module itself
```

于是外部用户可以访问 `my_pkg::geom::Point`。经由重新导出的模块的路径，与经由该模块本身的路径解析方式相同。

## 5.8.9. `prelude` 模式

按照惯例，库作者会构建一个汇集常用项的 `prelude` 模块，以精选对外的接口面。编译器不会自动引入它；用户必须显式导入。

```nabi
// pkg::prelude
pub use pkg::collections::{List, Dict, Set};
pub use pkg::ops::{Add, Sub, Mul};
pub use pkg::iter::Iterator;
```

在用户一侧，用一行导入它：

```nabi
// user side
use my_pkg::prelude::*;
```

编译器不赋予 `prelude` 这个名称任何特殊含义；它被当作普通模块。

## 5.8.10. 循环重新导出（不允许）

不允许重新导出之间互相指向形成循环。

```nabi
mod a { pub use pkg::b::Item; }
mod b { pub use pkg::a::Item; }
// ERROR: circular dependency
```

这样的循环是错误，由编译器检测。

## 5.8.11. 实现

重新导出只在名称表中添加一个条目。实际的代码和内存只存在于原项处；没有包装函数或副本，所以开销为零。任意长度的重新导出链都解析到原项；没有步数限制。

## 5.8.12. glob 重新导出与可见性

glob 重新导出是否暴露某个项，取决于原项的可见性。

```nabi
pub use source::*;               // a is exposed externally; b is pub(pkg), so visible only within the package

mod source {
    pub fn a() { ... }
    pub(pkg) fn b() { ... }
}
```

`pub use ...::*` 只导入这样的项：重新导出的可见性能到达调用方，且原可见性允许该重新导出的可见性。可达范围窄的项不能通过重新导出放宽（[5.8.4](#584-可见性级别的组合)）。

## 5.8.13. 小结

`pub use` 是把内部结构与对外 API 分开的关键工具。用户只需要知道公开的位置，库作者则可以自由地重新组织内部位置。
