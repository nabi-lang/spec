# 5.7. Paths

A sequence of identifiers joined by the `::` token ([punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)) is a path that designates a location in the module tree. This document defines paths as references to modules and items.

[Type paths](../002-types/213-type-paths.md) covers paths in type position; this document covers paths referring to values and modules. Both have the same form.

```
path            ::= path_root ("::" path_segment)*
path_root       ::= "pkg" | "sup" | "::" identifier | identifier
path_segment    ::= identifier
                  | identifier "::" "<" type_arg_list ">"      // turbofish
type_arg_list   ::= type_expr ("," type_expr)* ","?
```

The grammar above defines only the turbofish (`name::<T>`) of value and module position paths. Generics written as `name<T>` in type position are defined by `type_path` in [type paths](../002-types/213-type-paths.md).

## 5.7.1. Starting identifier of a path

The first identifier of a path is one of:

| Start token          | Meaning                                                  |
|----------------------|----------------------------------------------------------|
| `pkg`                | From the root of the current package                     |
| `sup`                | From the parent module                                   |
| `std`, `core`, `bare` | A standard library tier, unless a local name takes it    |
| `::`                 | The packages: a tier or one of the package's dependencies, past any local name |
| Ordinary identifier  | A currently visible name (child module, `use`d name)     |

`pkg` shares its stem with the visibility keyword `pub(pkg)`, and `sup` is the **mirror** direction of the visibility `pub(sub)` (sub means descendants, sup means ancestors). Both are path roots only when immediately followed by `::`; elsewhere they are ordinary names ([1.5.4 Contextual keywords](../001-lexical/105-keywords.md)). The tier names `std`, `core` and `bare` are ordinary identifiers ([5.7.4](#574-std-core-bare-the-standard-library-tiers)).

`self::` is not a path starting point. Children and items of the same module are reachable by ordinary identifiers (`child::item`).

Name resolution does not implicitly search ancestor modules (exception: `extern` functions, [5.9.4](509-extern-block.md#name-space-of-extern-functions)). An ordinary identifier is looked up only among the current module's items, child modules, and names imported with `use` (including the implicit `use core::prelude::*`). Items of the parent or higher modules must be named with `sup::` or imported with `use`.

Lookup order is as follows; the first match wins.

1. Items and child modules of the current module, and the names of the package's own dependencies ([5.7.14](#5714-paths-between-packages); a name that is both is an ERROR)
2. A standard library tier (`std::fs`), unless a named `use` of the current module takes that name
3. `use`s of the current module. Named imports take precedence over globs (`*`, `**`).
4. The implicit `use core::prelude::*` (the standard library's own modules: `use core::**`)
5. For a bare name in a position that expects an enum type, a variant of that enum ([2.4.8](../002-types/204-enum.md#248-bare-variants-from-the-expected-type))

Functions, generic or not, share a single value name space. The nearest declaration therefore wins regardless of kind (a non-generic `max` in the current module hides the generic `max` of core).

```nabi
// inside pkg::a::b
helper();              // ERROR: no implicit search even if helper is in pkg::a
sup::helper();         // OK
```

## 5.7.2. `pkg::`: from the package root

```nabi
pkg::geom::Point
pkg::config::DEFAULT_TIMEOUT
pkg::main_module::run
```

`pkg::` starts from the package's root module (`src/main.nabi` or `src/lib.nabi`) and is written the same way anywhere in the package. It is the standard form for safe absolute references in large codebases.

## 5.7.3. `sup::`: from the parent module

```nabi
sup::sibling_function
sup::parent_const
```

`sup::` starts from the current module's parent module, so it suits references to items of sibling and parent modules. To go up several levels, chain it as `sup::sup::...`.

```nabi
sup::sup::ancestor_item
```

The house style writes `sup::` only inside an inline `mod name { }` (such as `use sup::*;` in a test module) and a path from the package root (`pkg::`) everywhere else: the warning lint `sup_path` ([6.2](../006-meta-system/602-built-in-attributes.md)) reports the other `sup::` paths and `pub(sub)`, and its note names the `pkg::` path to write.

## 5.7.4. `std::`, `core::`, `bare::`: the standard library tiers

```nabi
core::collections::List
core::iter::Iterator
bare::ptr::Ptr
std::fs::File
```

The standard library is three packages, one per tier ([0.5 Standard library tiers](../000-meta/005-standard-libraries-tiers.md)), and each tier's name is the root of its paths.

| Path           | Meaning                              |
|----------------|--------------------------------------|
| `bare::*` | raw / unsafe area                    |
| `core::*` | foundation / commonly used traits    |
| `std::*`  | OS integration / I/O etc.            |

The three packages ship with the toolchain, and the compiler builds and links them automatically ([0.5.4](../000-meta/005-standard-libraries-tiers.md#054-distribution-form-and-linking-mechanism)); they are not user package dependencies and need no `--extern`. No other package may be named `std`, `core` or `bare` ([9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)).

Otherwise a tier's name is an ordinary name. A module, an item or a dependency alias may be called `std`, `core` or `bare`; in the module that declares or imports it, that name is the local one, like any name that shadows another. A leading `::` always names the toolchain's library (or a dependency): `::std::fs::File` reaches the tier past a local `std`.

```nabi
mod std { pub fn f() -> i32 { 1 } }

fn main() {
    let a = std::f();                    // the module above
    let p = ::std::path::Path::empty();  // the library's std
}
```

`bare` and `std` are not automatically imported and require an explicit import. The `core` prelude, by contrast, is visible automatically through the implicit `use core::prelude::*;` defined in [5.5 use](505-use.md).

## 5.7.5. Starting with an ordinary identifier

```nabi
foo::bar::baz                   // OK if foo is a currently visible name
geom::Point                      // if the geom module is a child or imported
Point                            // a single name, resolved automatically
```

If the first identifier is a name visible in the current module, it can be used as is. A visible name here means a child module or an item imported with `use`.

## 5.7.6. Meaning of each path segment

```nabi
pkg::geom::shapes::Circle
└──┘  └──┘  └────┘   └────┘
root  mod   module   item
```

Each segment is a module or (for the last one) an item. The last identifier is the final item being designated; the preceding segments are modules or types along the path.

## 5.7.7. Paths by item kind

| Designates                                   | Form                  |
|----------------------------------------------|-----------------------|
| Function                                     | `path::function_name` |
| Type (`struct` / `enum` / `union` / `trait`) | `path::TypeName`      |
| `const`                                      | `path::CONST_NAME`    |
| `static`                                     | `path::STATIC_NAME`   |
| Module itself                                | `path::module_name`   |
| Static method                                | `Type::method_name`   |
| Associated `const`                           | `Type::CONST`         |
| Associated type                              | `T::AssocType`        |

These are the same as the associated item access forms of [type paths](../002-types/213-type-paths.md).

## 5.7.8. Expression position and type position

The same path form can be used both in expression position (designating a value) and in type position (designating a type).

```nabi
// expression position
let f = pkg::geom::distance;          // function value
let p = pkg::geom::Point::origin();   // static method call
let v = pkg::config::DEFAULT;           // const value

// type position
let p: pkg::geom::Point = ...;
fn make() -> pkg::geom::Point { ... }
```

The difference between the two positions is whether generic arguments are written with the turbofish (`::<...>`). Expression position uses the turbofish; type position uses plain `<...>`. Details are in [type paths](../002-types/213-type-paths.md).

## 5.7.9. Combining paths and globs

Combined with the import forms of [`use`](505-use.md), various import notations are available.

```nabi
use pkg::geom::{Point, Circle, distance};
use sup::{a, b as alias};
use foo::*;
```

`*` (glob) and `**` (recursive glob) behave as defined in [`use` (imports)](505-use.md).

## 5.7.10. Empty paths / a leading `::`

An empty path (only `::` with no identifier) is a compile error.

A leading `::` starts at the packages: the identifier after it is a standard library tier or a dependency of the package (`::std::fs`, `::json::parse`), whatever a local name of that spelling is. Any other name after it is an ERROR (`::A`); an item of this package is written `pkg::A`.

## 5.7.11. Shortening with `use`

A name imported with `use` can be used by its short name in the current module.

```nabi
use pkg::geom::Point;

fn f() {
    let p = Point { ... };          // short name
    let q: Point = ...;
}
```

Gathering frequently used items with `use` at the top of a module keeps code bodies concise.

## 5.7.12. When paths are resolved

Path resolution happens in the compile-time name resolution phase and has no effect on runtime behavior (method dispatch on dynamic slots, however, happens separately at runtime; see [`any` slot](../002-types/202-any-slot.md)).

## 5.7.13. Referring to modules dynamically (none)

```nabi
let m = "foo";
m::bar();                            // ERROR: no dynamic module references
```

The module part of a path is always static and determined at compile time. Receiving a module as a variable at runtime and calling functions in it is not provided.

This is consistent with the principle in [modules](504-modules.md) that the module tree is a static structure.

## 5.7.14. Paths between packages

Items of another package (a dependency) are referenced by paths starting with that package's name.

```nabi
use parser::Token;
let t = parser::lex(src);
```

- The name of a dependency package is a top-level path. The name is the `deps` key in `nabi.nabion` with `-` replaced by `_` (`nabi-syntax` → `nabi_syntax::..`), and the build tool registers it with `nabic --extern name=..` ([9.2.3](../009-tools/902-aot-pipeline.md#923-dependency-resolution), [9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)). No separate declaration such as Rust's `extern crate` is needed.
- A package sees only the packages it depends on, under the names its own `deps` give them. The dependencies of another package in the build are not in scope: a package that depends on `csv` as `"csvio": Dep::..(package: "csv")` writes `csvio::..`, and there `csv::..` is its own module `csv`, or an ERROR whose help says to add `csv` to that package's `deps`.
- A name that is both a dependency of the package and a child module of the current module is ambiguous: a path starting with it is an ERROR that names both. Its help says what to write: `::name::..` for the dependency, `pkg::..::name::..` for the module, or another name for the dependency in `nabi.nabion` (a key with `package:` naming the package).
- `pkg::` inside a dependency package is that dependency's own root.
- A module at the root of the package being built may not have the name of a package in the build (the packages are mounted at the top of one module tree): it is an error that says to rename the module.
- A dependency that resolves to the package being built is an error; a package names its own items with `pkg::`.

### Package boundaries

Each of the following is a separate package. The visibilities `pub(pkg)` / `pub(sub)` ([5.6](506-visibility.md)) do not cross these boundaries.

- The package being built
- Each dependency package
- Each standard library tier (`bare`, `core`, `std`)

Thus private / `pub(sub)` / `pub(pkg)` items of a dependency cannot be used from outside it; only `pub` items can.
