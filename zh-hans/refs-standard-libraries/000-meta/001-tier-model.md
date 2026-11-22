# 0.1. 三层级模型

Nabi 标准库分为三个层级。每个层级可以依赖的范围不同，用户代码中的导入规则也不同。

```
bare   (base, depends only on the runtime ABI extern "nabi")
  ↓
core     (foundation, safe, all targets; uses bare internally)
  ↓
std   (target-dep; uses foundation + bare)
```

每个层级各自是一个路径根：`bare::..`、`core::..`、`std::..`（[refs-lang 5.7.4](../../refs-lang/005-modules/507-paths.md#574-std-core-bare-the-standard-library-tiers)）。

分成三层的理由很明确。可能破坏内存安全的原始操作，限制在狭小的范围内才更容易审计；必须在每个平台上表现一致的核心抽象，只有不与依赖操作系统的代码混在一起，才能保证可移植性。可用性因平台而异的功能，必须在签名和导入处暴露这一事实，让用户能一致地处理 wasm 构建和原生构建。三个层级各自负责其中一项要求。

## 0.1.1. 各层级的职责

### `bare`

最底层的 `bare` 只依赖运行时的 `extern "nabi"` 符号。它暴露 `Ptr<T>`、`ConstPtr<T>`、`transmute`、原始分配、pin/unpin 和 volatile 等原始工具，所以使用它们总是需要显式的 `use`。显式本身就是危险信号：这是一种有意的设计，让危险的范围在代码审查时一眼可见。

```nabi
use bare::ptr::CPtr;
use bare::alloc::{alloc, Layout};
let layout = try Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // null on failure (see 1.4 alloc)
```

### `core`（基础）

中间的基础层级 `core` 包含 `core::collections`、`core::iter`、`core::ops` 等模块，只在内部依赖 `bare`。日常使用语言所需的几乎所有抽象（数值类型、算术 trait、iter、集合类型、fmt、转换、sync、task、future、字节流 I/O 等）都集中在这一层。在用户代码中，**其 prelude `use core::prelude::*;` 会隐式地自动生效**（详细策略见 [0.2 导入策略](002-import.md)），因此无需任何导入即可直接使用。

```nabi
let xs: List<i32> = [1, 2, 3];        // List used directly, brought in by the prelude
let r = a + b;                         // Add trait automatic
let m = Mutex::new(0);                 // sync::Mutex automatic too
task::spawn(async { /* ... */ });              // task module automatic too
```

prelude 是一个固定的名称列表（[0.2.3](002-import.md)）；`core` 的其余部分通过 prelude 中的模块来指称（`task::sleep`、`ops::Add`），或者导入。

### `std`

最上层的 `std` 按需同时使用基础层级和 `bare`。这一层包含 `fs`、`net`、`time`（挂钟时间和启动时钟）、`thread`（操作系统线程）、`env`、`process`、`path`、`random` 等依赖平台的项。由于可用性因目标而异，用户必须始终显式导入它们，表明自己正在使用哪些平台功能。

```nabi
use std::fs;
use std::time::SystemTime;
```

## 0.1.2. 依赖规则（强制）

允许以下依赖。

| 依赖方      | 可以依赖                                                  |
|-------------|-----------------------------------------------------------|
| `bare`  | 运行时 ABI `extern "nabi"`                                |
| `core`  | `bare`                                              |
| `std`  | `core`、`bare`                                  |
| 用户代码    | `core`（隐式）、`std`（显式）、`bare`（显式） |

以下方向的依赖**被禁止**。基础层级（`core`）不得导入 `std`：基础层级必须能在每个目标上构建，一旦引入依赖平台的代码，可移植性就会崩溃。`bare` 也不得导入基础层级或 `std`：底层一旦开始依赖上层，分层本身就失去了意义。最后，各个 `std::*` 模块之间禁止循环依赖，使每个 std 模块都保持为可以独立构建的单元。

该规则约束的是一个层级所写的内容：它的 `use` 和路径。每个层级内部的 `use core::**` 仍然为它提供 bare 自身的签名和实现所用的语言词汇（1.6、1.7）：0.4 中的槽项、`List`、`Slice`、`Result`、`panic`，以及 bare 的类型所实现的 trait：`Debug` / `Formatter` / `FmtError`、`Hash` / `Hasher`、`Error`、`AsBytes`，还有 `Utf8Error` 和 `str::from_utf8`。

prelude 是 `core` 唯一指称 `std` 项的地方：在相应目标上，`core::prelude` 列出 `std::os` 的 Ext trait（0.2.3），因此使用它们的方法不需要 `use`。这属于隐式导入（0.2），而不是 core 自身代码的依赖；core 自身的代码在没有 `std` 时仍然可以构建。

## 0.1.3. 构建单元

三个层级各自作为独立的构建单元（独立的归档文件）处理，并且可以按编译缓存。链接顺序是 `std` → `core` → `bare` → 运行时，用户包位于最上层。编译器在首次使用时把它们构建到缓存中并自动链接，所以用户不需要把它们加入依赖声明（见 [0.5.4 分发形式与链接机制](../../refs-lang/000-meta/005-standard-libraries-tiers.md#054-distribution-form-and-linking-mechanism)）。

## 0.1.4. 目标可用性

`bare` 和基础层级（`core`）可以在每个[受支持的目标](../../refs-lang/000-meta/006-platform-targets.md)上构建，包括 wasm32。而 `std` 的可用性因目标而异。例如，`std::thread` 在只运行一个线程的 `wasm32-wasip2` 和浏览器中无法启动线程（`wasm32-wasip1-threads` 有线程）。同样，`std::fs` 在 wasm32-unknown-unknown 上不可用，只在 WASI 目标（`wasm32-wasip2`、`wasm32-wasip1-threads`）上提供。这类可用性写在每个 `std::*` 模块规范的可用性一节中。API 仍然可以在每个目标上构建：目标不具备的操作在运行时以 `IoErrorKind::Unsupported` 失败。在浏览器中，stdout / stderr 输出到宿主控制台，stdin 处于 EOF。例外是只属于某一操作系统家族的功能：它们位于 `std::os::<family>`（`unix`、`linux`、`windows`）中，只存在于该家族的目标上，所以在其他地方使用它是编译期错误，错误会指出所需的 `#[cfg]`（[3.10](../300-std/310-os.md)）。

## 0.1.5. 演进策略

三个层级的稳定性策略也不同。`bare` API 不稳定，与编译器和运行时一同演进，因此建议用户代码固定版本。基础层级（`core`）API 以 SemVer 稳定性为目标，不兼容的变更要等到破坏性版本（1.0 之前为 `0.x.0`）。`std` API 的变动程度与其各目标可用性的差异相当，但签名本身保持稳定。

在一个版本系列内，项不会被移除，也不会发生不兼容的变更。被取代的项继续可用，并标记为 `#[deprecated(since = "..", note = "..")]`（[6.2.7](../../refs-lang/006-meta-system/602-built-in-attributes.md#627-deprecated-discouraged-use-marker)），note 指出应改用什么。必须变为 `unsafe` 的安全函数标记为 `#[deprecated_safe]`（[6.2.14](../../refs-lang/006-meta-system/602-built-in-attributes.md#6214-deprecated_safe-safe-today-unsafe-later)）。下一个破坏性版本会移除该项，或把它改为 `unsafe`。

## 0.1.6. 每个项只有一条路径

每个项只有一条路径。模块像命名空间一样分为公开和私有子模块：公开子模块是用户会指称的命名空间（`core::num::niche`、`core::crypto::ed25519`、`std::net::tls`、`std::os::unix::fs`），而只承载父模块部分代码的子模块是私有的（`core::io` 中的 `mod buffer;`），其父模块用 `pub use` 重新导出它的项。如果途经的每个模块都是公开的，项的路径就是其定义模块的路径；否则是重新导出它的父模块的路径：`BufReader` 是 `core::io::BufReader`，`TcpStream` 是 `std::net::TcpStream`，`sleep` 是 `core::task::sleep`。同一个项的第二条路径只以两种形式存在：一是把公开子模块展平的父模块（`core::collections::List` 对应 `core::collections::list::List`，以及 `core::crypto::Secret`、`bare` 根下的转换函数）；二是隐式导入（0.2），它在相应目标上还会指称 `std::os` 的 Ext trait（0.1.2）。没有任何模块重新导出其父模块、兄弟模块或其他层级的项：`Duration` 就是 `core::time::Duration`，没有别的路径；`SeekFrom` 是 `core::io::SeekFrom`；`size_of` 是 `bare::mem::size_of`。文档和诊断中显示项的路径时，使用最短的公开路径，绝不经过私有模块（`std::net::WriteHalf`）。标准库自身的测试会拒绝其他任何 `pub use`。
