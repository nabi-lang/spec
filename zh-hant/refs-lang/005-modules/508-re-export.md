# 5.8. 重新匯出（`pub use`）

把 `use` 與可見性關鍵字結合，就構成重新匯出：匯入的項目從目前模組再次公開。文法如下：

```
re_export       ::= visibility "use" use_tree ";"
```

## 5.8.1. 形式

重新匯出結合了可見性關鍵字、`use`、路徑與分號：

```nabi
pub use foo::bar;
pub(pkg) use pkg::internal::helper;
pub(sub) use sup::shared;
```

把 `pub` 或其他可見性關鍵字與 `use`、路徑及 `;` 一起寫出，就會以該可見性從目前模組再次公開匯入的項目。

## 5.8.2. 意義

```nabi
// root
pub use deep::Item;             // Item is pub at the root too

// pkg::deep::Item
mod deep {
    pub struct Item { ... }
}
```

外部套件於是可以透過兩條路徑存取同一個項目。

```nabi
use my_pkg::deep::Item;          // original path: OK
use my_pkg::Item;                // re-export path: OK
```

同一個項目透過兩條路徑可見。

## 5.8.3. 用途

### 門面模式

用來隱藏內部結構，並對外提供扁平的 API。

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

外部使用者存取 `my_pkg::Point`，不需要知道內部結構（`core::geom::`）。

### 建立 prelude

把常用項目集中在一處的 prelude 模組，也以相同方式建立。

```nabi
// pkg::prelude
pub use pkg::types::{Point, Circle, Polygon};
pub use pkg::ops::{distance, area};
```

使用者只用一行 `use my_pkg::prelude::*;` 就能一次引入常用的項目。

### 版本相容性

若內部項目搬移了但保留重新匯出的路徑，舊的路徑對外部使用者而言仍然可用。

```nabi
// v1: pkg::Item
pub struct Item { ... }

// v2: moved
pub mod core { pub struct Item { ... } }
pub use core::Item;             // keeps the v1 path
```

## 5.8.4. 結合可見性等級

與可見性關鍵字結合的重新匯出：

```
pub use         → exposed to external packages
pub(pkg) use    → exposed within the package
pub(sub) use    → exposed to submodules
use             → module-private (not a re-export)
```

可見性的順序是 私有 < `pub(sub)` < `pub(pkg)` < `pub`。原始項目的可見性必須 ≥ 重新匯出的可見性；比原始項目更寬的 `pub use` 是錯誤。

```nabi
pub use inner::private_fn;       // ERROR: cannot expose a private item as pub

mod inner {
    fn private_fn() { ... }      // private
}
```

glob 重新匯出（`pub use mod::*`、`pub use mod::**`）的行為不同：原始可見性比重新匯出可見性窄的項目會被默默過濾掉，而這不是錯誤（[5.8.12 glob 重新匯出](#5812-glob-重新匯出與可見性)）。

## 5.8.5. 多重重新匯出

可以一次重新匯出多個項目，也可以加上別名。

```nabi
pub use foo::{a, b, c};
pub use foo::{
    Point,
    Circle,
    distance as dist,
};
```

[`use`](505-use.md) 的多重匯入與別名形式原樣適用。

## 5.8.6. glob 重新匯出

要一次重新匯出整個模組或子樹：

```nabi
pub use foo::*;
pub use foo::**;
```

整個模組或其下的子樹一次公開。

> [!WARNING]
> 名稱衝突在重新匯出處是錯誤。

## 5.8.7. 與別名結合

別名可以縮短對外可見的名稱。

```nabi
pub use foo::very_long_name as short_name;
```

對外只看得到簡短名稱（`short_name`）；要使用原本的名稱（`very_long_name`）需要另外匯入。

## 5.8.8. 重新匯出模組本身

不只是項目，模組本身也可以被重新匯出。

```nabi
pub use foo::geom;               // re-exports the module itself
```

外部使用者於是可以存取 `my_pkg::geom::Point`。經由重新匯出之模組的路徑，與經由該模組本身一樣解析。

## 5.8.9. `prelude` 模式

依慣例，函式庫作者會建立一個集中常用項目的 `prelude` 模組，以精選對外的介面。編譯器不會自動拉進它；使用者必須明確匯入它。

```nabi
// pkg::prelude
pub use pkg::collections::{List, Dict, Set};
pub use pkg::ops::{Add, Sub, Mul};
pub use pkg::iter::Iterator;
```

使用者一方以一行匯入它：

```nabi
// user side
use my_pkg::prelude::*;
```

編譯器不賦予 `prelude` 這個名稱任何特殊意義；它被當作一般模組處理。

## 5.8.10. 循環的重新匯出（無）

不允許互相指向的重新匯出循環。

```nabi
mod a { pub use pkg::b::Item; }
mod b { pub use pkg::a::Item; }
// ERROR: circular dependency
```

這樣的循環是錯誤，由編譯器偵測。

## 5.8.11. 實作

重新匯出只是在名稱表中加入一筆條目。實際的程式碼與記憶體只存在於原始項目處；沒有包裝函式也沒有複本，因此成本為零。任何長度的重新匯出鏈都解析到原始項目；沒有步數限制。

## 5.8.12. glob 重新匯出與可見性

glob 重新匯出是否公開某個項目，取決於原始項目的可見性。

```nabi
pub use source::*;               // a is exposed externally; b is pub(pkg), so visible only within the package

mod source {
    pub fn a() { ... }
    pub(pkg) fn b() { ... }
}
```

`pub use ...::*` 只匯入重新匯出可見性能到達呼叫者、且原始可見性允許該重新匯出可見性的項目。可及範圍窄的項目無法透過重新匯出被放寬（[5.8.4](#584-結合可見性等級)）。

## 5.8.13. 總結

`pub use` 是把內部結構與對外 API 分離的關鍵工具。使用者只需要知道公開的位置，而函式庫作者可以自由地重新組織內部位置。
