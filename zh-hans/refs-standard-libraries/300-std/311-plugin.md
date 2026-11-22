# 3.11. `plugin::`：加载插件

`std::plugin` 把插件（用 `lib: { kind: "plugin" }` 构建的库包）加载到正在运行的程序中，并把插件的函数取为函数值（[9.12](../../refs-lang/009-tools/912-plugins.md)）。插件运行在程序的运行时上，所以它接收和返回的都是程序中的普通值。

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

## 3.11.1. 函数

```nabi
fn load(path: str) throws (PluginError) -> Plugin;  // load the file (a bare name: in the current directory); a loaded one gives that plugin
fn file(name: str) -> str;                          // "libname.dylib" / "libname.so" / "name.dll"
fn beside(name: str) -> str;                        // file(name) in the running executable's directory
fn loaded() -> List<Plugin>;                        // in load order, unloaded ones left out
fn wait_unloaded(p: Plugin, ms: u64) -> bool;       // collect until p is gone, at most ms milliseconds
```

`file` 给出 `nabi build` 在当前目标上为插件包写出的文件名，`beside` 给出 `plugin: true` 依赖被放置的位置（9.10.3）。

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

`get::<F>(path)` 按路径查找插件包中一个 `pub` 的非泛型函数，并把它作为 `F` 类型的值返回。`F` 必须正好是该函数的类型：签名通过“用类型键写出的签名文本”的哈希来比较，所以其他类型会得到 `WrongType`，其 `have` 是函数声明的类型。`F` 必须是函数类型；其他类型是编译错误，错误会说明如何写出函数类型或改用 `get_static`。泛型函数不会被导出：请导出一个调用它的具体 `pub fn`。`get_static::<T>(path)` 读取静态变量 `path`。

`types()` 给出插件代码向 `core::reflect` 查询的类型及组成这些类型的类型的描述（[2.12.7](../200-core/212-reflect.md#2127-typeinfo)）。这些描述指向插件内部：插件卸载后不得再使用。

`unload` 是延迟的（9.12.5）：

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

每条消息都说明应当怎么做（见 [9.12.6](../../refs-lang/009-tools/912-plugins.md#9126-稳定性与错误) 的表格）。

## 3.11.4. 目标

插件可以在 [9.12.7](../../refs-lang/009-tools/912-plugins.md#9127-目标) 所列的目标上加载。在 wasm 和静态链接的 musl 目标上，用到 `std::plugin` 的程序无法链接，错误会提示改为构建成一个程序。
