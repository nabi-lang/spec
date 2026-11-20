# 3.11. `plugin::` - loading plugins

`std::plugin` loads a plugin, a library package built with `lib: { kind: "plugin" }`, into the running program and gets its functions as function values ([9.12](../../refs-lang/009-tools/912-plugins.md)). A plugin runs on the program's runtime, so what it takes and returns are ordinary values of the program.

```nabi
use std::plugin;

let p = try plugin::load(plugin::beside("enemies"));
let spawn = try p.get::<fn(i32) -> List<str>>("enemies::spawn");
println(spawn(3));
match p.unload() {
    plugin::Unload::Done => {},
    plugin::Unload::Pending(why) => println(f"later: {why}"),
    plugin::Unload::Kept(why) => println(f"kept: {why}"),
}
```

## 3.11.1. Functions

```nabi
fn load(path: str) throws (PluginError) -> Plugin;  // load the file (a bare name: in the current directory); a loaded one gives that plugin
fn file(name: str) -> str;                          // "libname.dylib" / "libname.so" / "name.dll"
fn beside(name: str) -> str;                        // file(name) in the running executable's directory
fn loaded() -> List<Plugin>;                        // in load order, unloaded ones left out
fn wait_unloaded(p: Plugin, ms: u64) -> bool;       // collect until p is gone, at most ms milliseconds
```

`file` names the file `nabi build` writes for a plugin package on this target, and `beside` where a `plugin: true` dependency is put (9.10.3).

## 3.11.2. `Plugin`

```nabi
struct Plugin;                       // Copy; copies name one plugin
fn name(self) -> str;                // its package name
fn version(self) -> str;             // its package version ("" when unknown)
fn path(self) -> str;                // the file it was loaded from
fn get<F>(self, path: str) throws (PluginError) -> F;
fn get_static<T>(self, path: str) throws (PluginError) -> T;
fn types(self) -> List<TypeInfo>;    // its type descriptors (core::reflect)
fn unload(self) -> Unload;
```

`get::<F>(path)` finds a `pub` non-generic function of the plugin's package by its path and returns it as a value of `F`, which must be the function's exact type: the signature is compared by a hash of its text written with type keys, so another type is `WrongType`, whose `have` is the function's declared type. `F` must be a function type; anything else is a compile error that shows how to write one or to use `get_static`. A generic function is not exported: export a concrete `pub fn` that calls it. `get_static::<T>(path)` reads static `path`.

`types()` gives the descriptions ([2.12.7](../200-core/212-reflect.md#2127-typeinfo)) of the types the plugin's code asks `core::reflect` about and the types they are made of. They point into the plugin: do not use one after it is unloaded.

`unload` is deferred (9.12.5):

```nabi
enum Unload {
    Done,            // unloaded: nothing of it was alive
    Pending(str),    // no longer found; it goes once nothing of it is alive (what still holds it)
    Kept(str),       // stays loaded (why)
}
```

## 3.11.3. `PluginError`

```nabi
#[non_exhaustive]
enum PluginError {
    NotFound { path: str; },
    NotAPlugin { path: str; },
    Toolchain { plugin: str; program: str; },
    Target { plugin: str; program: str; },
    MissingPackages { names: List<str>; },
    MissingSymbols { names: List<str>; },
    Layout { type_path: str; },
    NoSuchItem { path: str; near: List<str>; },
    WrongType { path: str; want: str; have: str; },
    NotLoaded { name: str; },
    Os { message: str; },
}
fn message(self) -> str;   // also its Display
```

Each message says what to do (the table of [9.12.6](../../refs-lang/009-tools/912-plugins.md#9126-stability-and-errors)).

## 3.11.4. Targets

Plugins load on the targets of [9.12.7](../../refs-lang/009-tools/912-plugins.md#9127-targets). On wasm and on statically linked musl targets a program that reaches `std::plugin` does not link, with an error that says to build one program.
