# 0.5. Standard library tiers

The standard libraries are divided into three package tiers, `bare`, `core` and `std`, each a path root of its own (`core::List`, `std::fs`). They are distinguished by the environments in which they are available and by the risk of the behavior they expose.

## 0.5.1. Three tiers

The overall structure is as follows.

```
  bare   ←   no safety guarantees. Raw memory, FFI, transmutable bit casts.
     ↑
  core   ←   safe foundation available on every target. Uses bare internally.
     ↑
  std   ←   target-dependent tier. File system, network, OS threads, etc.
```

The arrows show the direction of dependency. `core` may use `bare` in its internal implementation, and `std` may use both `core` and `bare`. Dependencies in the opposite direction are forbidden.

Each tier's name starts its paths ([5.7.4](../005-modules/507-paths.md#574-std-core-bare-the-standard-library-tiers)). The names are not keywords: a user module or dependency alias may take one, and `::std::..` still reaches the library past it.

## 0.5.2. Responsibilities of each tier

### `bare`

`bare` is the low-level, dangerous area. Behavior whose safety the compiler cannot guarantee gathers here; representative items include raw pointer forms (the raw pointer abstraction) and arbitrary bit casts (`transmute`-like). Pinning, volatile memory access, and the primitives of C [ABI](007-glossary.md#abi-application-binary-interface)-compatible memory allocation are also part of `bare`.

The caller of a function in this tier is responsible for safety, and the path itself makes that fact clear.

### `core` (foundation)

`core` is the most frequently used safe library; every module of the foundation lives here (e.g. `core::collections`, `core::iter`, `core::ops`). It must be available identically on every build target this specification guarantees. Its contents are very broad: numeric, character and string types and their methods, and collections (lists, maps, sets, tuples). It has the trait definitions for arithmetic, comparison, logical and bitwise operators, and also provides iterators, streams and futures. Concurrency primitives (atomic, mutex, channel) and the API surface for tasks and parallel execution also belong to this tier. Finally, it covers formatting, display, debug, reflection and inspection.

Portable SIMD vectors are `core::simd`; acceleration domains such as tensors / automatic differentiation / GPU are not part of `core`; external libraries implement them and reach the compiler's backend through the backend interface, planned for a later release. The standard libraries are responsible only for the surface semantics of ordinary code and leave the acceleration domain to decisions at the library layer.

The common items of `core` are made visible by the compiler implicitly applying `use core::prelude::*;` at the start of every Nabi source. That is, `core::collections::List` can be written as `List`, and `core::task::sleep` as `task::sleep`. The prelude's list is fixed in [stdlib 0.2.3](../../refs-standard-libraries/000-meta/002-import.md).

Only the prelude's items are visible without a `use`. Every other `pub` item of `core` is reached through its module (`ops::Add`, `num::Zero`) or a `use`; items of `core` without `pub` are private to it, as in any package (5.6).

### `std`

`std` is the area that is meaningful only in particular environments (mainly on top of an OS). It contains the file system and networking, time and clocks, standard I/O, environment variables, processes, and direct control of OS threads.

Functions of `std` may not exist on some targets. In environments where some areas are meaningless (for example, a notebook kernel running inside a browser), the corresponding modules are not built.

## 0.5.3. Visibility conventions

The visibility rules are as follows.

| Area            | Implicit import | Explicit import required | Notes                                         |
|-----------------|-----------------|--------------------------|-----------------------------------------------|
| `core`     | ✓ prelude items only | ✓ for other items   | `use core::prelude::*;` applied automatically    |
| `bare::**` | ✗               | ✓ required               | Makes the danger signal visible in code       |
| `std::**` | ✗               | ✓ required               | Makes target dependency visible in code       |
| User packages   | ✗               | ✓ required               | Items of other packages                       |

The concrete syntax for "bringing in" (`use`, `import`, etc.) is defined in the module system specification.

## 0.5.4. Distribution form and linking mechanism

The three tiers ship with the compiler as library packages in source form:

```
lib/nabi/bare/src/lib.nabi
lib/nabi/core/src/lib.nabi
lib/nabi/std/src/lib.nabi
```

The compiler builds them on first use into its cache and links them automatically ([9.2.19](../009-tools/902-aot-pipeline.md#9219-installation-layout)). This path is separate from the ordinary dependencies of user packages, so a user package that happens to have the same name (e.g. `mod bare`, `mod std`) does not collide.

Like the compiler itself, the three tiers follow the zero-dependency principle. Building a user package links only the tiers it needs, and the resulting artifact requires no additional external libraries.

## 0.5.5. Why three tiers

With only two tiers (safe, dangerous), "safe anywhere" could not be distinguished from "safe only when an OS is present", and the split would break down in environments lacking some OS features (browser notebooks, etc.). Conversely, four or more tiers increase the rules users must memorize and diminish the value of module boundaries.

Three tiers is a split well proven in practice, and the simplest one that is still meaningful.

Naming each tier by its own root keeps paths short and lowers the learning cost: `core` is in scope everywhere through the prelude, `std::fs` reads as what it is, and the division of responsibility among `bare` / `core` / `std` is learned gradually.
