# 5.4. Modules

Submodules are declared with the `mod` keyword ([keywords](../001-lexical/105-keywords.md)). This document defines the mapping between module declarations and the file system layout.

```
mod_decl        ::= "mod" identifier ";"
                  | "mod" identifier "{" item* "}"      // inline module
```

## 5.4.1. Module declarations

```nabi
mod foo;
mod parsing;
```

A module declaration is the `mod` keyword, a module name, and `;`. The module name follows the [identifier](../001-lexical/104-identifiers.md) form, and the declaration alone maps automatically to the file or folder of the same name.

## 5.4.2. Mapping forms

When `mod foo;` appears, the compiler looks for one of the following two forms in the same directory as the parent module body. These are the same rules as Rust 2018+.

### Form 1: single file (`foo.nabi`)

A module without submodules needs only a single file.

```
parent/
├── parent.nabi                <- where `mod foo;` appears
└── foo.nabi                   <- foo module body, no submodules
```

The contents of `foo.nabi` are the body of module foo.

### Form 2: folder (`foo.nabi` + `foo/`)

A module with submodules must have both a body file (`foo.nabi`) and a folder of the same name (`foo/`). The body file serves as the module's body, and the folder holds its submodules.

```
parent/
├── parent.nabi                <- where `mod foo;` appears
├── foo.nabi                   <- foo module body
└── foo/                      <- foo's submodules
    ├── bar.nabi                   declared as `mod bar;` inside foo.nabi
    └── baz.nabi                   declared as `mod baz;` inside foo.nabi
```

Here `foo.nabi` is the body of module foo and contains ordinary items, submodule declarations such as `mod bar;`, visibility, and so on. Each `.nabi` file in the `foo/` directory is a submodule of foo and must be explicitly declared in the body as `mod name;`. If only `foo/` exists without `foo.nabi`, it is an ERROR. The body file is mandatory.

This model is identical to Rust 2018+'s `foo.rs` + `foo/` structure; there is no separate body file name such as `mod.nabi`.

### Meaning of a file and folder with the same name

When `foo.nabi` and `foo/` of the same name sit together in the parent directory, they are recognized as the folder form automatically; this is not a conflict.

```
parent/
├── parent.nabi
├── foo.nabi                   <- foo body
└── foo/                      <- foo's submodules
    └── ...
```

In summary: only `foo.nabi` is a single-file module; `foo.nabi` together with `foo/` is a folder module; only `foo/` is an ERROR.

## 5.4.3. Submodules

Only folder-form modules (`foo.nabi` + `foo/`) may have submodules. Writing `mod name;` in the body (`foo.nabi`) maps `foo/name.nabi` or `foo/name.nabi` + `foo/name/` as that submodule.

```nabi
// contents of foo.nabi
mod bar;
mod baz;

pub fn foo_helper() { ... }
```

The following files must exist alongside that body:

```
foo.nabi                       <- module body (the code above)
foo/
├── bar.nabi                   <- foo::bar submodule
└── baz.nabi                   <- foo::baz submodule
```

A submodule may itself take either the single-file or the folder form.

```
foo.nabi
foo/
├── bar.nabi                   <- foo::bar (single-file)
├── baz.nabi                   <- foo::baz body
└── baz/                      <- foo::baz submodules
    ├── x.nabi                     foo::baz::x
    └── y.nabi                     foo::baz::y
```

This pattern applies recursively to any depth, and each module independently chooses the single-file or folder form.

### Single-file modules have no **file-mapped** submodules

```nabi
// foo.nabi (single-file module, no foo/)
mod bar;            // ERROR: there is no foo/, so nowhere to map
```

If file-mapped submodules become necessary, create a folder of the same name (`foo/`) and put the submodule files in it. Promoting a single-file module to a folder module costs only creating the folder and moving submodule files.

Inline submodules (`mod bar { ... }`), on the other hand, may be used freely even within a single-file module, since they need no file mapping ([5.4.5 Inline modules](#545-inline-modules-defined-within-the-body)).

## 5.4.4. Package root

The `src/main.nabi` or `src/lib.nabi` defined in [source units](../000-meta/003-source-units.md) is the body file of the package root module, and the package's source directory `src/` serves as the root module's folder.

The root module is recognized automatically without a `mod` declaration, and every other module is a node in the tree starting from the root.

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

The root module is always in folder form: `src/` is its folder, and `lib.nabi` (library) or `main.nabi` (executable) is its body file. `mod foo;` in the root maps to `src/foo.nabi`; a `foo.nabi` beside `src/` is never a module. `#[path(..)]` (5.4.6) resolves from the folder where the submodule's file would normally be: `src/` for a `mod` in the root, `src/parsing/` for a `mod` inside `parsing.nabi`.

## 5.4.5. Inline modules: defined within the body

Placing a block body directly after a `mod` declaration defines an inline module without a separate file.

```nabi
mod inner {
    pub fn helper() -> i32 { 42i32 }
    const X: i32 = 100i32;
}

// use
let r = inner::helper();
```

An inline module is written `mod name { ... }`, and its body may contain any items: functions, types, traits, `const`, `static`, and further `mod`s. A file or folder of the same name must not exist alongside it; a conflict is an ERROR.

## 5.4.6. Custom paths: `#[path("...")]`

```
path_attr       ::= "#" "[" "path" "(" string_lit ")" "]"
```

The `#[path("...")]` attribute overrides the default mapping.

```nabi
#[path("custom_geom.nabi")]
mod geom;

#[path("../shared/shared.nabi")]
mod shared;
```

The file at the given path is mapped as the module body. The path is relative to the directory where `name.nabi` would normally be found. Submodules of that module go in the folder next to the file with its stem: for `#[path("f.nabi")]`, in `f/`. The target file may not be named `mod.nabi`, since Nabi has no `mod.nabi` body form ([5.4.4](#544-package-root)); it is an error. `#[path(..)]` is the only form. It rarely appears in ordinary use and serves special cases such as build systems and generated code.

## 5.4.7. Items within a module

Each module may contain the items introduced in the [overview](README.md).

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

A module may contain only items; executable statements (top-level `let`, expression statements) are allowed only in the entry module `main.nabi` ([4.16.6](../004-functions/416-entry-point.md#4166-modules-are-definition-only)). All such items are part of that module, and referencing them from outside requires both a [visibility](506-visibility.md) keyword and a [path](507-paths.md). An item declared in a function body ([3.9.1](../003-expressions/309-block-expressions.md#391-form)) belongs to the module for coherence but is named only inside its block.

## 5.4.8. Module visibility

A module itself may be given a visibility.

```nabi
pub mod public_api;            // usable from external packages
pub(pkg) mod internal;          // within the same package only

mod _private;                   // module-private: only from the same parent module
```

The detailed meaning is defined in [visibility](506-visibility.md).

## 5.4.9. Referring to parent / sibling / child within a module

The path notation for referring to items of other modules is defined in [paths](507-paths.md).

```nabi
pkg::geom::Point             // from the package root
sup::sibling_mod::item        // through the parent module to a sibling
child_mod::item                // within the same module (an identifier suffices)
```

## 5.4.10. Where module declarations may appear

A `mod` declaration may appear only at module level, that is, in the body of another `mod` or at the package root. Declaring a `mod` inside a function body is an ERROR ([3.9.1](../003-expressions/309-block-expressions.md#391-form) lists the items a block may declare).

```nabi
fn f() {
    mod inner { ... }          // ERROR: no module declarations inside a function
}
```

This follows from the principle that the module tree is a static structure determined at compile time.

## 5.4.11. Splitting one module across several files

To split a body across several files, use the folder form and re-export items through submodules from the body (`foo.nabi`).

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

With this pattern users access every item as `foo::item`, the body file stays small, and the actual code is distributed across submodules. It is the same as Rust's `pub use` re-export pattern.

## 5.4.12. No duplicate modules in the same parent

Declaring a module of the same name twice in the same parent module is an ERROR.

```nabi
mod foo;
mod foo;                        // ERROR: duplicate
```
