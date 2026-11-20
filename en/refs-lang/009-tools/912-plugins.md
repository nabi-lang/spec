# 9.12. Plugins

A *plugin* is a library package built as a shared library that a Nabi program loads while it runs ([3.11 `std::plugin`](../../refs-standard-libraries/300-std/311-plugin.md)). Unlike a `cdylib` (9.3.3), a plugin has no runtime of its own: it runs on the loading program's runtime, so its values are the program's values (one GC heap, one scheduler, one copy of each static), and any `pub` Nabi function can be called across the boundary with any Nabi value.

## 9.12.1. Model

A process has one runtime, in the *host*: the program that loads plugins. A plugin links no runtime and none of the packages it takes from the host. Instead it carries a module descriptor ([8.12](../008-abi/812-module-abi.md)) whose import table names every runtime entry, standard library static and dependency symbol it uses; the Nabi module loader binds each one by name to the host (or to a plugin loaded before) when the plugin is loaded. The operating system's loader only maps the file and resolves C symbols (the C library and `#[link]` libraries).

A program is a host when it reaches `std::plugin`; nothing has to be declared. A host keeps its whole runtime and lists its Nabi symbols in an export table, which costs nothing in a program that loads no plugin, because only a host gets one.

| | cdylib | plugin |
|-|--------|--------|
| Loaded by | any C host | a Nabi program |
| Runtime | its own copy, started when loaded | the host's |
| Interface | `pub extern "C" fn`, `#[repr(C)]` types | every `pub fn`, any Nabi value |
| GC heap, scheduler, statics | private | shared with the host |

## 9.12.2. Building

```nabion
Package {
    name: "enemies",
    version: "0.1.0",
    lib: { kind: "plugin" },
    deps: {
        "engine": "^0.3",                             // taken from the host
        "noise": Dep::version("^1.2", bundle: true),  // linked into the plugin
    },
}
```

- `lib: { kind: "plugin" }` makes `nabi build` pass `--crate-type=plugin` ([9.1](901-nabic-cli.md)). The output is `target/<profile>/libenemies.dylib` (macOS), `libenemies.so` (Linux) or `enemies.dll` (Windows), beside `libenemies.a` and `enemies.nabihir` as for any library. `std::plugin::file("enemies")` gives the file name for the target.
- A plugin package has `src/lib.nabi` and no `src/main.nabi` (`` a plugin is loaded by a program, so move main.nabi to a package of its own that loads this one with `std::plugin::load` ``). `nabi run` on it is an error.
- A dependency is taken from the host by default: the plugin compiles against it, its statics are the host's, and the descriptor lists it under `needs`. `bundle: true` links the dependency into the plugin instead: its statics are the plugin's own and it is listed under `provides`. `bundle` on a dependency of a package that is not a plugin is an error.
- A program's dependency `Dep::path("../physics", plugin: true)` is built as a plugin into the program's target directory (so beside the program, with the program's profile and target) and is not linked in; the program loads it with `plugin::load(plugin::beside("physics"))`. `plugin: true` and `bundle: true` on one dependency is an error.
- The standard library's code a plugin reaches is compiled into it (reach-only, as for a program, [9.3.6](903-package-archive.md#936-standard-library)); its statics are the host's. Duplicate code is harmless because every module of a process is built by the same toolchain (9.12.6).
- A plugin's generic instances are compiled into the module that needs them: `List<Orc>` lives in the plugin that defines `Orc`. Generic code has no statics of its own, so its copies are interchangeable.
- Codegen units ([9.2.16](902-aot-pipeline.md#9216-incremental-build)) are off for plugins and hosts: their descriptors hold the real package hashes.

## 9.12.3. Identity

A type is the same type in two modules when its *type key* is the same: the package's content hash, the path and the generic arguments' keys, `Y3f8a1c2b:engine::Body` ([8.13](../008-abi/813-type-descriptor.md)). The package hash is a content hash ([9.5](905-symbol-mangling.md)), so a plugin built against other sources of `engine` than the host's has another key, and the load fails with a missing package rather than mixing two `Body` layouts.

- Symbols bind by mangled name. A plugin's import of `_Y3f8a1c2b_engine_spawn_..` binds only to a module that provides package `3f8a1c2b`.
- `any` tags are given per process: the host registers its compile-time tags, and a plugin's code reads its tags from its descriptor's tag table, which the loader fills (8.12.6). A key the process has seen keeps its tag, so an `engine::Body` stored into `any` by the plugin is the host's `engine::Body`.
- GC type ids are structural and per module; the loader gives each plugin a range.
- A type that both modules describe (an `Inspect` type, a generic instance) must have the same layout hash. A difference, typically a dependency built with other features, refuses the load: `` engine::Body is laid out differently in the program and the plugin; build both with the same features of its package ``.

## 9.12.4. Loading and calling

`plugin::load(path)` maps the file and registers its descriptor inside a short stop of every thread: it checks the toolchain and target, that every package under `needs` is provided (all missing ones are reported in one error), binds the imports, gives the GC type ids and `any` tags, registers the roots, type descriptors and `any Tr` method bodies, and publishes the plugin. A plugin has no initialization function; its statics initialize on first use, as everywhere.

`p.get::<F>("enemies::spawn")` looks up a `pub` non-generic function by path and checks that its signature is exactly `F`: both sides hash the signature's text written with type keys (`fn({engine::World},i32)->List<..>`), so a mismatch is `WrongType`, not a crash. The result is an ordinary function value. `get` of a generic function finds nothing (`NoSuchItem`, whose message says to export a concrete `pub fn`). `p.get_static::<T>(path)` reads a static.

Across the boundary:

- `str`, `List`, `Dict`, closures and every other GC value are references into the one heap and pass both ways as they are.
- A closure keeps the plugin that made it mapped until the closure is collected (9.12.5).
- `any` and `any Tr` pass as they are. A dispatcher of an `any Tr` method knows the types of its own module; for a tag it does not know it asks the process (`nabi_any_dispatch`, 8.12.6), which calls the dispatcher of the module that stored the value. A program that loads no plugin keeps the closed dispatcher of [2.2.10](../002-types/202-any-slot.md).
- Sums, enums and `throws` errors are values with structural tags and cross unchanged. A panic in a plugin ends the process as anywhere else.

## 9.12.5. Unloading

`p.unload()` is deferred:

1. If another loaded plugin imports from `p`, it returns `Pending` with what uses it. If `p` gave out a C callback of its own code (`NO_UNLOAD`), it returns `Kept`: such a plugin stays mapped.
2. Otherwise `p` leaves lookup at once (`get` and `plugin::loaded()` no longer see it) and is *retired*.
3. A later full collection checks whether a retired plugin is still alive: an object of one of its GC types, a function value or coroutine whose code is in it, a frame of it on a stack. When none is left, its roots, types and method bodies are dropped and the file is unmapped; `unload` returns `Done` when that is already so.

`plugin::wait_unloaded(p, ms)` collects until the plugin is gone or the time is up (tests and tools).

## 9.12.6. Stability and errors

There is no plugin ABI stable across toolchain builds. Every module of a process must come from the same toolchain build: the descriptor's `toolchain` is the compiler id, compared exactly (`` the plugin was built by nabic 0.1.3 (a1b2c3), this program by nabic 0.1.4 (d4e5f6); rebuild the plugin with this program's toolchain ``).

| `PluginError` | When | What the message says to do |
|---|---|---|
| `NotFound` | no file at the path | give the file `nabi build` wrote |
| `NotAPlugin` | a file without a Nabi descriptor | build the package with `lib: { kind: "plugin" }`, or load a C library with `bare::ffi` |
| `Toolchain` / `Target` | another compiler id / target | rebuild the plugin |
| `MissingPackages` | a package under `needs` the process lacks | add it to the program, or `bundle: true` in the plugin |
| `MissingSymbols` | an import nothing provides | build the plugin against the program's packages |
| `Layout` | one key, two layouts | build both with the same features |
| `NoSuchItem` | no `pub fn` / static at the path | names close to it; export a concrete function |
| `WrongType` | another signature | the type to write |
| `NotLoaded` | the plugin was unloaded | load it again |
| `Os` | the system loader failed | its message |

## 9.12.7. Targets

| Target | File | Status |
|---|---|---|
| aarch64-apple-darwin | `MH_DYLIB`, install name `@rpath/lib<name>.dylib`, ad hoc signed, exports only `_nabi_module` | supported; a hardened-runtime host needs the `disable-library-validation` entitlement to load plugins others signed |
| x86_64-unknown-linux-gnu, aarch64-unknown-linux-gnu | `ET_DYN`, exports only `nabi_module` | supported |
| x86_64-pc-windows-msvc, aarch64-pc-windows-msvc | DLL exporting `nabi_module`, with its own entry; loaded with `LoadLibraryExW` | supported; a loaded DLL's file is locked |
| *-linux-musl | | an error: the C library is linked statically and cannot load code (`` build plugins and the program for `x86_64-unknown-linux-gnu`, or build one program ``) |
| wasm32-* | | an error: `` `--crate-type=plugin` needs a native target ``; a wasm program that reaches `std::plugin` fails to link the same way (`` on wasm, build one program: make the plugin's package an ordinary dependency ``) |
| other targets | | not linked yet |

`NABI_CC` cannot link plugins or hosts: they need Nabi's own linker, which makes the import stubs and tables.
