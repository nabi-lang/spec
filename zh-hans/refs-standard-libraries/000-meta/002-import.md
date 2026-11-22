# 0.2. 导入策略：prelude `core::prelude`

> 本文档定义标准库的隐式导入策略。每个用户文件无需 `use` 即可看到 `core::prelude` 这一个模块中的名称；`core` 中的其他内容，以及 `bare` 和 `std` 的全部内容，都要通过路径指称或显式导入。

Nabi 会在每个用户源文件开头**隐式**加上下面这一行。

```nabi
use core::prelude::*;
```

`core::prelude` 是一个由 `pub use` 重新导出组成的普通模块（[refs-lang 5.8](../../refs-lang/005-modules/508-re-export.md)）。它的列表在下文（0.2.3）中固定，并且是规范性的：只有本文档列出的名称才在 prelude 中。

prelude 不包含枚举变体。单独的变体名根据该位置所期望的类型来解析（[refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-bare-variants-from-the-expected-type)）：`let o: Ordering = Less;`、`x.load(SeqCst)`、`return Ok(v);`。

## 0.2.1. 核心设计决策

| 模型                                          | 语言                  | Nabi 是否采用  |
|-----------------------------------------------|-----------------------|----------------|
| 只用显式 use                                  | C / C++               | （冗长）       |
| 核心库的所有公开名称                          | Swift、Python builtins | 不采用        |
| **简短且经过筛选的 prelude 模块**             | Rust、Gleam、Elm      | **采用**       |

### 为什么是筛选过的列表

如果对整个 `core` 做隐式 glob，每个公开的 core 项都会在每个文件中成为单独的名称：约 160 个类型、100 个 trait、90 个函数，以及所有枚举变体。其中有几个会相互冲突（`iter` 和 `stream` 中的 `Map`、`Filter`、`Zip`；`task` 和 `par` 中的 `join`；`Ordering` 和 `fmt::Alignment` 中的变体 `Equal`），而且每新增一个 core 项都会改变用户代码中的名称解析。

prelude 保留的是几乎每个文件都会写到的内容：集合类型和核心类型、写在 `impl` 头部和约束中的 trait、打印和 panic 函数，以及 core 的顶层模块，这样 `task::sleep` 和 `par::map` 依然简短。其中每个名称在其命名空间内都是唯一的（库测试在每个目标上检查这一点），所以单独的 prelude 名称永远不会有歧义。由于隐式导入的优先级最低（0.2.4），向 prelude 添加名称永远不会破坏定义或导入了同名项的程序。

## 0.2.2. `**` 递归 glob

```nabi
use foo::bar;            // single item
use foo::{a, b};         // multiple
use foo::*;              // single-level glob (only foo's direct children)
use foo::**;             // recursive glob (foo + all submodules)
```

`**` 语法的规范性定义见 [refs-lang 5.5 use](../../refs-lang/005-modules/505-use.md)。

## 0.2.3. prelude 列表

下面的名称正好就是 `core::prelude` 中的名称。与 Nabi 的其他地方一样，模块、类型和 trait 共用一个命名空间，函数使用另一个，所以 `channel`、`panic` 和 `print` 各自既是模块又是函数。

**模块（29 个）。** `core` 的顶层模块，`prelude` 本身和 `hash`（用 `use` 导入，[2.5.11](../200-core/205-cmp.md)）除外：

`array` `bench` `channel` `cmp` `collections` `convert` `crypto` `encoding` `error` `fmt` `future` `hint` `index` `io` `iter` `marker` `num` `ops` `panic` `par` `print` `range` `reflect` `str` `stream` `sync` `task` `time` `tuple`

**类型（36 个）。**

| 分组 | 名称 |
|---|---|
| 集合类型 | `List` `Dict` `Set` `Deque` `BinaryHeap` `Slice` `SliceMut` `StrBuf` `FrozenList` `FrozenDict` `FrozenSet` |
| 数值与区间 | `Range` `RangeInclusive` `Int` `Float` |
| 通用 | `Result` `Ordering` `Duration` `Instant` |
| 同步与通道 | `Mutex` `RwLock` `Atomic` `Cell` `Sender` `Receiver` `Task` |
| 格式化 | `Formatter` `FmtError` |
| 核心错误 | `ParseError` `ValueError` `OverflowError` `ConversionError` `TypeError` `StackOverflowError` `OutOfMemoryError` `InternalError` |

**trait（31 个）。**

| 分组 | 名称 |
|---|---|
| 标记 | `Clone` `Copy` `Drop` `Default` `Send` `Sync` |
| 比较 | `PartialEq` `Eq` `PartialOrd` `Ord` `Hash` |
| 格式化与错误 | `Debug` `Display` `RichDisplay` `Error` |
| 迭代与异步 | `Iterator` `IntoIterator` `FromIterator` `TryIterator` `Future` `IntoFuture` |
| I/O | `Read` `Write` `AsyncRead` `AsyncWrite` |
| 转换 | `From` `Into` `TryFrom` `TryInto` `FromStr` |
| 反射 | `Inspect`（也包括它的 derive，[refs-lang 6.8.11](../../refs-lang/006-meta-system/608-meta-blocks.md)） |

运算符 trait 不在 prelude 中：写 `ops::Add`，或者 `use core::ops::Add;`。运算符本身不需要名称，在任何地方都可用（[refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-visibility-of-mappings)），每个 `core` trait 的方法也是如此（[refs-lang 3.5](../../refs-lang/003-expressions/305-member-access-and-method-call.md)）。

**操作系统扩展 trait。** 在具备它们的目标上，prelude 还会指称 `PermissionsExt`、`CommandExt` 和 `ExitStatusExt`（unix），以及 `ChildExt` 和 `TcpStreamExt`（linux），所以在这些目标上使用它们的方法不需要 `use`（[3.10.1](../300-std/310-os.md#3101-ext-traits-in-the-prelude)）。它们不计入上面的数目。

**函数（13 个）。**

`print` `println` `eprint` `eprintln` `dbg` `display` `panic` `assert` `assert_eq` `assert_ne` `unreachable` `channel` `channel_bounded`

**变体。** 无。见 [refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-bare-variants-from-the-expected-type)。

```nabi
// user code
fn main() {
    let xs: List<i32> = [1, 2, 3];        // prelude: List
    let total: i32 = xs.iter().sum();      // methods of core traits: no import
    println(f"{total}");                   // prelude: println

    let m = Mutex::new(0);                 // prelude: Mutex
    task::spawn(async { /* ... */ });      // prelude module task
    task::sleep(Duration::ms(10));         // prelude: Duration

    let n = Atomic::new(0);
    n.store(1, SeqCst);                    // variant from the parameter type
}
```

`core` 中的其他内容都通过 prelude 中的模块来指称，或者导入：

```nabi
let p: future::Poll<i32> = future::Poll::Pending;
impl ops::Add for V { ... }
use core::sync::LoadOrdering;
let h = encoding::hex::encode("nabi");
```

不在作用域中的名称会得到一条错误，告诉你应该怎么写：可行的最短路径（`task::sleep`、`ops::Add`、`Ordering::Less`），以及导入它的 `use`。

## 0.2.4. 名称冲突策略

隐式导入的优先级最低：查找时先看模块自身的项和它的 `use`（[refs-lang 5.7.1](../../refs-lang/005-modules/507-paths.md#571-starting-identifier-of-a-path)）。与 prelude 名称同名的用户项会在该模块中遮蔽它，而 core 的项仍然可以通过路径访问。

```nabi
struct List<T> { ... }                  // OK: shadows the prelude's List here
let xs: core::collections::List<i32> = [1, 2];   // the core List, by path
```

`use` 路径可以以 prelude 名称开头：`use task::sleep;` 导入 `core::task::sleep`。

## 0.2.5. **不**隐式生效的层级

只有 prelude 是隐式的。unsafe 和 std **只能显式导入**。

```nabi
use bare::ffi::{CStr, CString};      // unsafe: danger signal
use bare::ptr::{Ptr, ConstPtr};
use std::fs;                         // std: target-dependency signal
use std::time::SystemTime;
```

该策略与 [0.1 层级模型](001-tier-model.md)中的下表一致。

| 层级            | 隐式导入                | 原因                            |
|-----------------|-------------------------|---------------------------------|
| `core`     | ✓ prelude（0.2.3）      | 所有目标 / 安全                 |
| `bare`     | ✗                       | 危险信号，只能显式导入          |
| `std`      | ✗                       | 依赖目标，只能显式导入          |

这样，用户进入了哪些范围，从文件顶部就能一眼看出。

## 0.2.6. 退出隐式导入：`#![no_implicit_nabi]`

在特殊情况下，可以关闭隐式导入本身。

```nabi
#![no_implicit_nabi]   // top of the file

// no implicit import in this file
use core::prelude::*;   // the same import, written out
```

字面量由编译器定义，仍然可用；运算符和 core trait 的方法也同样可用（[refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-visibility-of-mappings)）；失去的只是单独的名称。

它适用于教学或演示代码，可以一眼看出每个项来自哪里。

标准库自身的模块不导入 prelude：每个层级都能以末段名称看到 `core` 的每个公开项（内部的 `use core::**;`），用户代码则没有这种待遇。

## 0.2.7. 可见性

prelude 只重新导出 `pub` 项。内部实现（`pub(pkg)` / 私有）绝不会泄露给用户：`core::collections::raw` 是 `pub(pkg)`，所以 `GcBuf` 和 `buf_*` 函数只能通过显式 `use` 以 `bare::mem::..` 的形式访问。

列表之外的名称不会被展平；模块路径保持原样。

```nabi
let m = Mutex::new(0);              // prelude: Mutex
let s = task::spawn(async { /* */ });      // prelude module task; spawn is task::spawn

// both mean the same thing.
core::sync::Mutex::new(0);
Mutex::new(0);
```

## 0.2.8. 与其他语言的比较

| 语言     | 隐式作用域                                         | 变体                                     |
|----------|----------------------------------------------------|------------------------------------------|
| Rust     | `std::prelude::rust_20xx`，显式列表                | 只有 `Some` `None` `Ok` `Err`            |
| Go       | universe block：约 20 个内置函数和类型             | 需限定（`time.Second`）                  |
| Swift    | `Swift` 的所有公开声明                             | 根据上下文类型写 `.less`                 |
| Kotlin   | 默认包导入                                         | 上下文相关解析（2.2）                    |
| **Nabi**  | `core::prelude`，显式列表（0.2.3）            | 单独写，根据期望类型（任何枚举）         |
