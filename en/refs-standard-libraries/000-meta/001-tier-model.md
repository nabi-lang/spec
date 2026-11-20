# 0.1. The 3-tier model

The Nabi standard libraries are divided into three tiers. Each tier may depend on a different
area, and the import rules in user code differ as well.

```
bare   (base, depends only on the runtime ABI extern "nabi")
  ↓
core     (foundation, safe, all targets; uses bare internally)
  ↓
std   (target-dep; uses foundation + bare)
```

Each tier is a path root of its own: `bare::..`, `core::..`, `std::..` ([refs-lang 5.7.4](../../refs-lang/005-modules/507-paths.md#574-std-core-bare-the-standard-library-tiers)).

The reasons for the three-way split are clear. Raw operations that can break memory safety are
easier to audit when confined to a narrow area, and core abstractions that must behave identically
on every platform can guarantee portability only when they are not mixed with OS-dependent code.
Features whose availability varies by platform must expose that fact in their signatures and import
sites so that users can handle wasm builds and native builds consistently. Each of the three tiers
is responsible for one of these three requirements.

## 0.1.1. Responsibilities per tier

### `bare`

The bottom tier, `bare`, depends only on the runtime's `extern "nabi"` symbols.
It exposes raw tools such as `Ptr<T>`, `ConstPtr<T>`, `transmute`, raw allocation, pin/unpin and
volatile, so using them always requires an explicit `use`. Being explicit is the danger signal:
it is a deliberate device so that dangerous areas are visible at a glance during code review.

```nabi
use bare::ptr::CPtr;
use bare::alloc::{alloc, Layout};
let layout = try Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // null on failure (see 1.4 alloc)
```

### `core` (foundation)

The middle, foundation tier, `core`, holds modules such as `core::collections`, `core::iter` and `core::ops`, and depends on `bare` only internally. Nearly every abstraction needed for everyday use of the language (numeric types, arithmetic traits,
iter, collections, fmt, conversion, sync, task, future, byte-stream I/O, etc.) is gathered in this tier. In user code,
**its prelude, `use core::prelude::*;`, is applied implicitly and automatically** (see [0.2 Import policy](002-import.md)
for the detailed policy), so it can be used directly without any import.

```nabi
let xs: List<i32> = [1, 2, 3];        // List used directly, brought in by the prelude
let r = a + b;                         // Add trait automatic
let m = Mutex::new(0);                 // sync::Mutex automatic too
task::spawn(async { /* ... */ });              // task module automatic too
```

The prelude is a fixed list of names ([0.2.3](002-import.md)); the rest of `core` is named
through a prelude module (`task::sleep`, `ops::Add`) or imported.

### `std`

The top tier, `std`, uses both the foundation and `bare` as needed. This tier holds
platform-dependent items such as `fs`, `net`, `time` (the wall and boot clocks), `thread` (OS
threads), `env`, `process`, `path` and `random`. Because availability varies by target, users must always import them explicitly, making
clear which platform features they are stepping into.

```nabi
use std::fs;
use std::time::SystemTime;
```

## 0.1.2. Dependency rules (enforced)

The following dependencies are allowed.

| Dependent   | May depend on                                             |
|-------------|-----------------------------------------------------------|
| `bare`  | the runtime ABI `extern "nabi"`                           |
| `core`  | `bare`                                              |
| `std`  | `core`, `bare`                                  |
| User code   | `core` (implicit), `std` (explicit), `bare` (explicit) |

Dependencies in the following directions are **forbidden**. The foundation (`core`) must not
import `std`: the foundation must build on every target, and portability collapses the moment
it pulls in platform-dependent code. `bare` must not import the foundation or `std`
either: once a low-level tier starts depending on a high-level one, the layering itself loses its
meaning. Finally, circular dependencies among individual `std::*` modules are forbidden, so
each std module remains a unit that can be built independently.

The rule governs what a tier writes: its `use`s and paths. Each tier's internal `use core::**` still supplies it the language's vocabulary that bare's own signatures and impls use (1.6, 1.7): the 0.4 slot items, `List`, `Slice`, `Result`, `panic`, and the traits bare's types implement, `Debug` / `Formatter` / `FmtError`, `Hash` / `Hasher`, `Error`, `AsBytes`, with `Utf8Error` and `str::from_utf8`.

The prelude is the one place `core` names `std` items: on their targets `core::prelude` lists the `std::os` Ext traits (0.2.3), so their methods need no `use`. That is the implicit import (0.2), not a dependency of core's own code, which still builds without `std`.

## 0.1.3. Build units

Each of the three tiers is processed as a separate build unit (a separate archive) and can be cached per compilation.
The link order runs `std` → `core` → `bare` → runtime, with the user package on top
of everything. The compiler builds them on first use into its cache and links them automatically, so users do
not need to add them to dependency declarations (see [0.5.4 Distribution form and linking mechanism](../../refs-lang/000-meta/005-standard-libraries-tiers.md#054-distribution-form-and-linking-mechanism)).

## 0.1.4. Target availability

`bare` and the foundation (`core`) build on every [supported target](../../refs-lang/000-meta/006-platform-targets.md),
including wasm32. `std`, by contrast, has availability that varies by target. For example,
`std::thread` cannot start threads on `wasm32-wasip2` or in the browser, which run one thread
only (`wasm32-wasip1-threads` has threads). `std::fs` is likewise unavailable on wasm32-unknown-unknown and is provided only on
the WASI targets (`wasm32-wasip2`, `wasm32-wasip1-threads`). Such availability is stated in the availability section of each `std::*` module spec.
The API still builds on every target: an operation a target lacks fails at run time with
`IoErrorKind::Unsupported`. In the browser, stdout / stderr go to the host console and stdin is at EOF.
The exception is what belongs to one OS family: it lives in `std::os::<family>` (`unix`, `linux`,
`windows`), which exists only on that family's targets, so using it elsewhere is a compile-time error
that names the `#[cfg]` it needs ([3.10](../300-std/310-os.md)).

## 0.1.5. Evolution policy

The three tiers also differ in stability policy. The `bare` API is not stable and
evolves together with the compiler and runtime, so user code is advised to pin the version.
The foundation (`core`) API aims for SemVer stability, and an incompatible change waits for a breaking
release (`0.x.0` before 1.0). The `std` API is as volatile as its per-target availability differences, but the
signatures themselves remain stable.

Within a release series an item is never removed or changed incompatibly. An item that is replaced
keeps working and is marked `#[deprecated(since = "..", note = "..")]`
([6.2.7](../../refs-lang/006-meta-system/602-built-in-attributes.md#627-deprecated-discouraged-use-marker)),
the note naming what to use instead. A safe function that must become `unsafe` is marked
`#[deprecated_safe]` ([6.2.14](../../refs-lang/006-meta-system/602-built-in-attributes.md#6214-deprecated_safe-safe-today-unsafe-later)).
The next breaking release removes the item or makes it `unsafe`.

## 0.1.6. One path per item

Every item has one path. A module splits into public and private submodules the way a namespace
does: a public submodule is a namespace users name (`core::num::niche`,
`core::crypto::ed25519`, `std::net::tls`, `std::os::unix::fs`), while a
submodule that only holds part of its parent's code is private (`mod buffer;` in
`core::io`) and its parent re-exports its items with `pub use`. The path of an item is its
defining module's when every module on the way is public, else the parent's that re-exports it:
`BufReader` is `core::io::BufReader`, `TcpStream` is `std::net::TcpStream`, `sleep` is
`core::task::sleep`. A second path to the same item exists only in two shapes: a parent
module that flattens its public child modules (`core::collections::List` for
`core::collections::list::List`, `core::crypto::Secret`, the `bare` root casts),
and the implicit import (0.2), which also names the `std::os` Ext traits on their targets
(0.1.2). No module re-exports its parent's items, a sibling's or another tier's: `Duration` is
`core::time::Duration` and nothing else, `SeekFrom` is `core::io::SeekFrom`, `size_of`
is `bare::mem::size_of`. Where documentation and diagnostics show an item's path, it is the
shortest public one, never through a private module (`std::net::WriteHalf`). The standard library's own
tests reject any other `pub use`.
