# 8.12. 模块描述符 ABI

*模块*是进程中一个可加载的 Nabi 代码单元：程序本身（*宿主*）或一个插件。每个模块都用只读数据中的一个*模块描述符*向运行时描述自己。加载器读取它，以检查该模块是否适合当前进程、绑定它导入的符号，并注册它的 GC 类型、`any` 标签、根和类型描述符（[8.13](813-type-descriptor.md)）。布局只在 `nabir_abi::module` 中定义一次，编译器和运行时都构建它。

## 8.12.1. 符号

| 符号 | 定义者 | 含义 |
|--------|-----------|------------|
| `nabi_module`（Mach-O 上为 `_nabi_module`） | 插件 | 插件的描述符，也是插件唯一导出的符号 |
| `nabi_host_module` | 宿主 | 宿主的描述符，由运行时直接访问 |

## 8.12.2. 布局规则

所有指针都是绝对地址，由操作系统的重定位确定。`Str` 是指向 UTF-8 字节的 `{ptr, len: usize}`，`Slice<T>` 是指向 `len` 个元素的 `{ptr, len: usize}`。每个结构体在两种指针宽度下都没有隐式填充：`u64` 位于 8 的倍数处，指针大小的字成对出现。因此 `u64` 对齐为 4（i686）和 8（wasm32）的 32 位目标在每个偏移上都一致。字段只会追加；`version` 说明读取者可以使用哪些字段。

## 8.12.3. `ModuleDesc`

| 字段 | 类型 | 含义 |
|-------|------|---------|
| `magic` | `u32` | 以小端 `u32` 表示的 `"NABM"` |
| `version` | `u16` | 1 |
| `flags` | `u16` | `HOST` 1、`PLUGIN` 2、`HOT` 4（热重载构建）、`NO_UNLOAD` 8（把自身代码的 C 回调交了出去） |
| `hot_version` | `u64` | 除非设置了 `HOT`，否则为 0 |
| `text_lo`、`text_hi` | `usize` | 模块的代码为 `[text_lo, text_hi)` |
| `toolchain` | `Str` | 编译器 id；同一进程中的每个模块都相同 |
| `target` | `Str` | 目标三元组 |
| `name` | `Str` | 插件的包名，或宿主的程序名 |
| `provides` | `Slice<PkgDesc>` | 代码和静态变量位于本模块中的包 |
| `needs` | `Slice<PkgDesc>` | 本模块从进程中导入的包 |
| `imports` | `Slice<Import>` | 本模块从进程中获取的符号 |
| `exports` | `Slice<Export>` | 本模块提供的符号，按 `sym` 排序 |
| `gc_types` | `GcTypes` | 本模块的 ScanTable（[8.2](802-scan-table.md)）以及加载器为它们分配的 id |
| `tags` | `Tags` | 本模块代码使用的 `any` 标签的类型键，以及加载器为它们分配的标签 |
| `roots` | `Slice<Root>` | 本模块的静态根 |
| `types` | `Slice<*const TypeDesc>` | 本模块的类型描述符（[8.13](813-type-descriptor.md)） |
| `dispatch` | `Slice<Dispatch>` | 本模块的 `any Tr` 方法体，按方法和标签索引 |
| `functions` | `Slice<FnRange>` | 仅 `HOT`：每个函数的范围和帧信息 |

各组成部分：

| 结构体 | 字段 |
|-----------|--------|
| `PkgDesc` | `name: Str`、`version: Str`、`pkg8: [u8; 8]`（该包在其 `_Y` 符号中的包哈希的十六进制数字，[9.5](../009-tools/905-symbol-mangling.md)）、`sha256: [u8; 32]`（包的完整内容哈希） |
| `Import` | `sym: Str`（修饰后的名字）、`sig: u64`、`slot: *mut usize`、`kind: usize`（`FN` 0、`DATA` 1、`HOT_FN` 2） |
| `Export` | `sym: Str`、`path: Str`（对 `pub` 非泛型函数或静态变量为 `enemies::spawn` 这样的路径，否则为空）、`sig: u64`、`addr: usize`、`kind: usize`（同 `Import`，另加 `FN_VALUE` 3：函数的值 thunk）、`sig_text: Str`（`sig` 所哈希的类型，按 `type_name` 的写法，如 `fn(i32) -> List<str>`，用于错误消息；`sig` 为 0 时为空） |
| `GcTypes` | `tables: *const *const u8`、`ids: *mut u32`、`n: u32`、`reserved: u32` |
| `TagKey` | `key: Str`（类型键，8.13.2）、`hash: u64`（`key` 的 FNV-1a 64） |
| `Tags` | `keys: *const TagKey`、`tags: *mut u32`、`n: u32`、`reserved: u32` |
| `Root` | `word: *const usize`、`any: usize`（1：`any` 的载荷字，其标签位于前一个字） |
| `Dispatch` | `method: u64`、`code: usize`、`tag_index: usize`（`tags` 中的索引） |
| `FnRange` | `start: u32`（相对于 `text_lo`）、`len: u32`、`frame: *const u8`、`reserved: usize` |

`sig` 是用类型键写成的规范签名文本（如 `fn(Y3f8a1c2b:game::Player, i32) -> str`）的 SHA-256 的前 64 位。对数据而言，它是该静态变量类型键的哈希。`FN` 导入通过一个经 `slot` 跳转的桩来调用；`DATA` 导入的 `slot` 存放数据的地址。

## 8.12.4. 运行时入口

这些入口扩展了 [8.5](805-extern-symbols.md)。它们全部是 `thread-safe`。

| 入口 | 含义 |
|-------|---------|
| `nabi_types_register(descs: *const *const TypeDesc, n: usize, tags: *const TagKey, out_tags: *mut u32, nt: usize) -> i32` | 注册 `n` 个类型描述符并解析 `nt` 个标签（8.12.5）。返回 0；若 `descs[i]` 的键已经以另一个布局哈希注册，则返回 `1 + i`，此时不注册任何内容。 |
| `nabi_types_by_tag(tag: u32) -> *const TypeDesc` | 某个 `any` 标签的描述符（忽略其 `ANY_BOXED` 位），或 null |
| `nabi_types_by_path(p: *const u8, len: usize) -> *const TypeDesc` | `game::Player` 这类路径最新注册的描述符，或 null |
| `nabi_types_unregister(descs: *const *const TypeDesc, n: usize)` | 在模块被释放时撤销注册 |
| `nabi_module_register(m: *const ModuleDesc) -> i32` | 在一次短暂的全局停顿（stop-the-world）中检查并绑定模块、注册其各表；成功时返回 0 |
| `nabi_any_dispatch(method: u64, tag: u32) -> usize` | 已加载模块中针对 `tag` 值的 `method` 代码，或 0 |
| `nabi_tls_get() -> *mut u8` | 调用线程的 `nabi_tls` 块（[8.3](803-tls-slots.md)） |

类型注册表的读取者不加锁，也不分配任何内存。

## 8.12.5. 标签

模块的代码从 `Tags.tags` 中读取它所用每个类型的 `any` 标签。对每个键，若 `out_tags` 中已有标签（不是 `u32::MAX`），`nabi_types_register` 会保留它：宿主和没有插件的程序就以这种方式传入编译期标签。之前注册过的键得到相同的标签，这正是让同一类型在各模块间对应同一标签的机制。新键得到该键 FNV-1a 哈希的低 31 位，并移过所有已在使用的标签。

## 8.12.6. 编译器生成的内容

插件的胶水代码（[8.7](807-code-generation.md)）生成 `nabi_module` 及其指向的各表；宿主的胶水代码生成 `nabi_host_module`。既不涉及 `std::plugin` 也不使用 `--crate-type=plugin` 的程序两者都不生成，其代码与没有插件时完全相同。

| 字段 | 插件 | 宿主 |
|---|---|---|
| `flags` | `PLUGIN` | `HOST` |
| `text_lo`、`text_hi` | 指向 `nabi.module.text_lo` / `nabi.module.text_hi` 的重定位，这两个符号由链接器定义；加载器从映射的映像中获取代码范围 | 同左 |
| `provides` | 它自己的包以及它捆绑的包（[9.12.2](../009-tools/912-plugins.md#9122-building)） | 它链接的除标准库外的每个包 |
| `needs` | 构建中除标准库外的其他所有包 | 空 |
| `imports` | 指向 `nabi.module.imports` 的重定位，该表由链接器根据未定义的符号（每个 `nabi_*` 入口和 `_Y` 符号）构建；链接器把其长度写入描述符；`sig` 为 0 | 空 |
| `exports` | 按 `sym` 排序：其包中每个函数的 `FN`（`pub` 非泛型函数设置 `path`）、它们的值 thunk 的 `FN_VALUE`（设置 `path`、`sig` 和 `sig_text`）、其静态变量的 `DATA`（设置 `path`），以及它从进程中获取的静态变量的自有副本（见下文） | 指向 `nabi.module.exports` 的重定位，该表由链接器构建（每个已定义的 `nabi_*` 入口和 `_Y` 符号，`sig` 为 0）；链接器写入其长度 |
| `gc_types` | 其代码用到的每个 ScanTable；`ids` 初始为 0 | 空：宿主像任何程序一样用 `nabi_gc_register_type` 注册其表 |
| `tags` | 其代码用到的每个标签的键；`tags` 初始为 `u32::MAX` | 各键及其编译期标签 |
| `roots` | 它定义的每个静态变量的每个引用字（`any` 为 0） | 空：像任何程序的根一样注册 |
| `types` | 其类型描述符（[8.13](813-type-descriptor.md)），其中 `tag_index` 索引其 `tags` | 空：它像任何程序一样用 `nabi_types_register` 注册描述符 |
| `dispatch` | 每个 `any Tr` 分派器及其分支所接受的每个标签各一项：方法为 `<trait path>.<method>` 的 FNV-1a 64 哈希，代码为分派器的值代码 | 宿主的分派器同理 |

宿主的 `nabi.register_types`（[8.5](805-extern-symbols.md)）以 `nabi_module_register(&nabi_host_module)` 结尾，因此它的标签、分派项和导出表在 `main` 之前、在任何插件之前就已进入进程。

插件代码与程序代码的区别：

- **GC 类型 id**。分配会读取 `nabi.module.gc_ids[0]` 并加上表的索引：加载器给模块分配一段连续区间，因此 `ids[i] = ids[0] + i`。
- **`any` 标签**。存入 `any` 以及测试 `any` 的标签时，从 `nabi.module.tags` 加载标签（`tags[i]`，`i` 为该类型在模块标签列表中的索引），而不是使用常量；`ANY_BOXED` 在运行时按位或进去。
- **TLS**。原本要取 `nabi_tls`（[8.3](803-tls-slots.md)）、函数值 thunk 或 C 入口地址的代码，改为调用导入的 `nabi_tls_get()`。Nabi 到 Nabi 的调用和其他地方一样，把该块作为隐藏的最后一个参数传递，所以大部分代码从不需要询问。
- **从进程中获取的静态变量**。插件既不拥有也不捆绑的包（包括标准库）中的静态变量，通过目标平台的 GOT 形式访问，链接器让它指向导入的槽。插件还会把自己的副本定义为 `<sym>.own`（对惰性初始化的静态变量还有 `<sym>.state.own`、`<sym>.init.own`），并以 `<sym>` 的名字导出。加载器把导入绑定到宿主的静态变量；若没有，则绑定到之前加载的插件的静态变量；再没有，则绑定到这个副本。因此每个静态变量在进程中恰好有一个实例。
- **分派器**。分派器中针对未知标签的分支会调用 `nabi_any_dispatch(method, tag)`，并把结果当作分派器自身类型的函数值来调用。在不是宿主的程序中，该分支仍是 [8.8](808-lowering-types.md) 中的 trap。
- **签名**。导出的 `sig`，以及 `Plugin::get::<F>` 传入的值，是用类型键写成的签名文本（`fn(Y3f8a1c2b:engine::World,i32)->str`）的 SHA-256 的前 8 个字节（小端）。双方在编译期根据同一文本计算它，因此哈希相等就意味着类型相同。
