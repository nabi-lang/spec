# 9.12. 插件

*插件*是构建为共享库的库包，由 Nabi 程序在运行时加载（[3.11 `std::plugin`](../../refs-standard-libraries/300-std/311-plugin.md)）。与 `cdylib`（9.3.3）不同，插件没有自己的运行时：它运行在加载它的程序的运行时上，因此它的值就是程序的值（一个 GC 堆、一个调度器、每个静态变量一份），任何 `pub` Nabi 函数都可以携带任何 Nabi 值跨越边界调用。

## 9.12.1. 模型

一个进程只有一个运行时，位于*宿主*中，即加载插件的程序。插件不链接运行时，也不链接它从宿主获取的任何包。插件携带一个模块描述符（[8.12](../008-abi/812-module-abi.md)），其导入表列出它用到的每个运行时入口、标准库静态变量和依赖符号；加载插件时，Nabi 模块加载器按名称把每一项绑定到宿主（或先前加载的插件）。操作系统的加载器只负责映射文件和解析 C 符号（C 库和 `#[link]` 库）。

程序只要用到了 `std::plugin` 就是宿主，无需任何声明。宿主保留完整的运行时，并在导出表中列出其 Nabi 符号；对于不加载插件的程序，这不产生任何开销，因为只有宿主才有导出表。

| | cdylib | 插件 |
|-|--------|--------|
| 加载者 | 任何 C 宿主 | Nabi 程序 |
| 运行时 | 自己的一份，加载时启动 | 宿主的运行时 |
| 接口 | `pub extern "C" fn`、`#[repr(C)]` 类型 | 每个 `pub fn`、任何 Nabi 值 |
| GC 堆、调度器、静态变量 | 私有 | 与宿主共享 |

## 9.12.2. 构建

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

- `lib: { kind: "plugin" }` 使 `nabi build` 传入 `--crate-type=plugin`（[9.1](901-nabic-cli.md)）。输出为 `target/<profile>/libenemies.dylib`（macOS）、`libenemies.so`（Linux）或 `enemies.dll`（Windows），与任何库一样，旁边还有 `libenemies.a` 和 `enemies.nabihir`。`std::plugin::file("enemies")` 给出目标平台上的文件名。
- 插件包有 `src/lib.nabi`，没有 `src/main.nabi`（`` a plugin is loaded by a program, so move main.nabi to a package of its own that loads this one with `std::plugin::load` ``）。对插件包运行 `nabi run` 是错误。
- 依赖默认从宿主获取：插件针对它编译，它的静态变量是宿主的，描述符在 `needs` 下列出它。`bundle: true` 则把依赖链接进插件：它的静态变量归插件自己所有，并在 `provides` 下列出。对非插件包的依赖使用 `bundle` 是错误。
- 程序的依赖 `Dep::path("../physics", plugin: true)` 会作为插件构建到程序的目标目录中（因而位于程序旁边，使用程序的构建配置和目标），且不被链接进程序；程序用 `plugin::load(plugin::beside("physics"))` 加载它。在同一个依赖上同时使用 `plugin: true` 和 `bundle: true` 是错误。
- 插件用到的标准库代码被编译进插件（与程序一样只编译用到的部分，[9.3.6](903-package-archive.md#936-标准库)）；其静态变量是宿主的。重复的代码无害，因为一个进程中的每个模块都由同一个工具链构建（9.12.6）。
- 插件的泛型实例被编译进需要它们的模块：`List<Orc>` 位于定义 `Orc` 的插件中。泛型代码没有自己的静态变量，因此它的各个副本可以互换。
- 对插件和宿主关闭代码生成单元（[9.2.16](902-aot-pipeline.md#9216-增量构建)）：它们的描述符保存真实的包哈希。

## 9.12.3. 同一性

当一个类型的*类型键*相同时，它在两个模块中就是同一个类型。类型键由包的内容哈希、路径以及泛型实参的键组成，例如 `Y3f8a1c2b:engine::Body`（[8.13](../008-abi/813-type-descriptor.md)）。包哈希是内容哈希（[9.5](905-symbol-mangling.md)），因此针对与宿主不同的 `engine` 源代码构建的插件具有另一个键，加载会因缺少包而失败，而不会混用两种 `Body` 布局。

- 符号按修饰后的名称绑定。插件对 `_Y3f8a1c2b_engine_spawn_..` 的导入只会绑定到提供包 `3f8a1c2b` 的模块。
- `any` 标签按进程分配：宿主登记其编译期标签，插件的代码从其描述符的标签表中读取标签，该表由加载器填写（8.12.6）。进程已经见过的键保留其标签，因此插件存入 `any` 的 `engine::Body` 就是宿主的 `engine::Body`。
- GC 类型 id 是结构性的，并且按模块分配；加载器为每个插件分配一个区间。
- 两个模块都描述的类型（`Inspect` 类型、泛型实例）必须具有相同的布局哈希。若不同（通常是某个依赖以不同的特性构建），则拒绝加载：`` engine::Body is laid out differently in the program and the plugin; build both with the same features of its package ``。

## 9.12.4. 加载与调用

`plugin::load(path)` 映射文件，并在所有线程的一次短暂停顿中登记其描述符：它检查工具链和目标，检查 `needs` 下的每个包都有提供者（所有缺失的包在一个错误中一并报告），绑定导入，分配 GC 类型 id 和 `any` 标签，登记根、类型描述符和 `any Tr` 方法体，然后发布该插件。插件没有初始化函数；与其他地方一样，它的静态变量在首次使用时初始化。

`p.get::<F>("enemies::spawn")` 按路径查找一个 `pub` 非泛型函数，并检查其签名恰好是 `F`：双方都对以类型键书写的签名文本（`fn({engine::World},i32)->List<..>`）求哈希，因此不匹配会得到 `WrongType`，而不是崩溃。结果是一个普通的函数值。对泛型函数调用 `get` 找不到任何东西（`NoSuchItem`，其消息提示导出一个具体的 `pub fn`）。`p.get_static::<T>(path)` 读取一个静态变量。

跨越边界时：

- `str`、`List`、`Dict`、闭包以及其他所有 GC 值都是指向同一个堆的引用，原样双向传递。
- 闭包会使创建它的插件保持映射，直到闭包被回收（9.12.5）。
- `any` 和 `any Tr` 原样传递。`any Tr` 方法的分派器知道其所在模块的类型；遇到不认识的标签时，它询问进程（`nabi_any_dispatch`，8.12.6），由进程调用存入该值的模块的分派器。不加载插件的程序保留 [2.2.10](../002-types/202-any-slot.md) 中的封闭分派器。
- 和类型、枚举和 `throws` 错误是带结构性标签的值，原样跨越边界。插件中的 panic 与其他任何地方一样会结束进程。

## 9.12.5. 卸载

`p.unload()` 是延迟执行的：

1. 如果另一个已加载的插件从 `p` 导入，它返回 `Pending`，并附上使用 `p` 的对象。如果 `p` 交出过一个指向其自身代码的 C 回调（`NO_UNLOAD`），它返回 `Kept`：这样的插件保持映射。
2. 否则，`p` 立即退出查找（`get` 和 `plugin::loaded()` 不再看到它），并进入*退役*状态。
3. 之后的一次完整回收检查退役插件是否仍然存活：是否存在其 GC 类型之一的对象、代码位于其中的函数值或协程、栈上属于它的栈帧。当这些都不存在时，丢弃它的根、类型和方法体，并取消文件映射；如果调用 `unload` 时已经如此，则返回 `Done`。

`plugin::wait_unloaded(p, ms)` 持续进行回收，直到插件消失或超时（用于测试和工具）。

## 9.12.6. 稳定性与错误

不存在跨工具链构建稳定的插件 ABI。一个进程中的每个模块都必须来自同一个工具链构建：描述符的 `toolchain` 是编译器 id，进行精确比较（`` the plugin was built by nabic 0.1.3 (a1b2c3), this program by nabic 0.1.4 (d4e5f6); rebuild the plugin with this program's toolchain ``）。

| `PluginError` | 何时 | 消息提示的做法 |
|---|---|---|
| `NotFound` | 路径处没有文件 | 给出 `nabi build` 写出的文件 |
| `NotAPlugin` | 文件没有 Nabi 描述符 | 用 `lib: { kind: "plugin" }` 构建该包，或用 `bare::ffi` 加载 C 库 |
| `Toolchain` / `Target` | 编译器 id / 目标不同 | 重新构建插件 |
| `MissingPackages` | 进程缺少 `needs` 下的某个包 | 把它添加到程序中，或在插件中使用 `bundle: true` |
| `MissingSymbols` | 某个导入没有提供者 | 针对程序的包构建插件 |
| `Layout` | 同一个键，两种布局 | 用相同的特性构建两者 |
| `NoSuchItem` | 路径处没有 `pub fn` / 静态变量 | 与之相近的名称；导出一个具体函数 |
| `WrongType` | 签名不同 | 应当书写的类型 |
| `NotLoaded` | 插件已被卸载 | 重新加载它 |
| `Os` | 系统加载器失败 | 系统加载器的消息 |

## 9.12.7. 目标

| 目标 | 文件 | 状态 |
|---|---|---|
| aarch64-apple-darwin | `MH_DYLIB`，install name 为 `@rpath/lib<name>.dylib`，ad hoc 签名，只导出 `_nabi_module` | 已支持；启用 hardened runtime 的宿主需要 `disable-library-validation` entitlement 才能加载他人签名的插件 |
| x86_64-unknown-linux-gnu、aarch64-unknown-linux-gnu | `ET_DYN`，只导出 `nabi_module` | 已支持 |
| x86_64-pc-windows-msvc、aarch64-pc-windows-msvc | 导出 `nabi_module` 的 DLL，带有自己的入口；用 `LoadLibraryExW` 加载 | 已支持；已加载 DLL 的文件被锁定 |
| *-linux-musl | | 错误：C 库静态链接，无法加载代码（`` build plugins and the program for `x86_64-unknown-linux-gnu`, or build one program ``） |
| wasm32-* | | 错误：`` `--crate-type=plugin` needs a native target ``；用到 `std::plugin` 的 wasm 程序会以同样的方式链接失败（`` on wasm, build one program: make the plugin's package an ordinary dependency ``） |
| 其他目标 | | 尚未实现链接 |

`NABI_CC` 无法链接插件或宿主：它们需要 Nabi 自己的链接器，由它生成导入桩和各种表。
