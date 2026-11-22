# 8.13. 类型描述符 ABI

*类型描述符*（`TypeDesc`）是一个类型的只读记录。反射读取它（[6.10](../006-meta-system/610-reflection.md)），模块加载器用它检查两个模块对某个类型的理解是否一致（[8.12](812-module-abi.md)）。布局只在 `nabir_abi::typedesc` 中定义一次；[8.12.2](812-module-abi.md#8122-布局规则) 的布局规则同样适用。

## 8.13.1. 描述符何时存在

只有当程序请求某个类型的描述符时，编译器才会生成它：对 `Inspect` 类型调用 `T::type_info()` 或 `core::reflect::info::<T>()`，并递归地生成其字段、变体和类型实参所指类型的描述符。从不请求的程序没有任何描述符，其对象与没有反射时逐字节相同。

每个描述符都是只读数据中一个隐藏的弱符号，因此同一程序的两个目标文件生成的副本会合并为一个。当程序有描述符时，它的 `nabi.register_types` 会在启动时把它们连同编译期 `any` 标签一起传给 `nabi_types_register`。

## 8.13.2. 键

类型的*键*在各模块间为类型命名：

- 结构体、枚举或联合体：`Y<pkg8>:` 加上其完整路径，如 `Y3f8a1c2b:game::Player`，其中 `pkg8` 是定义它的包的哈希，与 [9.5](../009-tools/905-symbol-mangling.md) 中相同；
- 泛型实例：基础键后接 `<..>` 中的各实参键，如 `Y..:core::collections::List<Y..:game::Player>`；
- 基本类型、`str`、`any` 和 `none`：其名字；
- 元组、和类型、数组和函数类型：按源代码中的写法由各组成部分的键写成，和类型的成员按规范顺序排列。

`key_hash` 是键的 SHA-256 的前 8 个字节，按小端解释。两个描述符属于同一类型，当且仅当它们的键相等。

## 8.13.3. `TypeDesc`

| 字段 | 类型 | 含义 |
|-------|------|---------|
| `magic` | `u32` | 以小端 `u32` 表示的 `"NABT"` |
| `version` | `u16` | 1 |
| `kind` | `u16` | `STRUCT` 1、`TUPLE_STRUCT` 2、`UNIT` 3、`ENUM` 4、`UNION` 5、`SUM` 6、`PRIM` 7、`STR` 8、`LIST` 9、`DICT` 10、`ARRAY` 11、`TUPLE` 12、`FN` 13、`ANY` 14、`OPAQUE` 15 |
| `flags` | `u32` | `INSPECT` 1（已填充字段、属性和 `ops`）、`LAYOUT_ONLY` 2、`INSTANCE` 4、`HAS_DEFAULT` 8 |
| `tag_index` | `u32` | 模块 `Tags` 中的索引（[8.12.5](812-module-abi.md#8125-标签)）；没有时为 `u32::MAX` |
| `key_hash` | `u64` | 8.13.2 |
| `layout_hash` | `u64` | 大小、对齐、字段偏移与类型键、变体标签以及 GC 字的哈希 |
| `size` | `u64` | 同 `T::size()` |
| `align` | `u32` | 同 `T::align()` |
| `reserved0` | `u32` | 0 |
| `key` | `Str` | 8.13.2 |
| `name` | `Str` | 同 `T::type_name()`：`Player`、`List<f64>` |
| `path` | `Str` | `game::Player`；没有路径的类型为空 |
| `fields` | `Slice<FieldDesc>` | 结构体按声明顺序排列的字段；枚举为空 |
| `variants` | `Slice<VariantDesc>` | 枚举按声明顺序排列的变体 |
| `generics` | `Slice<TypeRef>` | 实例的类型实参；和类型的成员；元组的各部分；列表或数组的元素；字典的键，然后是值 |
| `attrs` | `Slice<AttrDesc>` | 类型上的运行时属性（[6.10.8](../006-meta-system/610-reflection.md#6108-运行时属性)） |
| `ops` | `*const TypeOps` | 动态访问（8.13.5），或 null |
| `reserved1` | `usize` | 0 |

## 8.13.4. 组成部分

| 结构体 | 字段 |
|-----------|--------|
| `TypeRef` | `name: Str`、`key_hash: u64`、`desc: *const TypeDesc`（未生成时为 null）、`kind: usize` |
| `FieldDesc` | `name: Str`（元组结构体或变体中为 `"0"`、`"1"`、..）、`renamed_from: Str`（除非有 `#[renamed_from(..)]`，否则为空）、`ty: TypeRef`、`attrs: Slice<AttrDesc>`、`offset: u64`（相对于值的起始处；字段没有固定位置时为 `u64::MAX`）、`flags: u32`（`PUB` 1、`HAS_DEFAULT` 2、`SKIP` 4）、`index: u32`、`init_default: FnValue`（仅热构建） |
| `VariantDesc` | `name: Str`、`fields: Slice<FieldDesc>`、`tag: u32`（存储的标签）、`index: u32`、`attrs: Slice<AttrDesc>` |
| `AttrDesc` | `ty: TypeRef`（属性结构体）、`value: *const u8`（按该结构体布局存放的常量）、`reserved: usize` |
| `FnValue` | `code: usize`、`env: usize`：一个 Nabi 函数值（[8.8](808-lowering-types.md)）；`env` 为 0 |

## 8.13.5. `TypeOps`

`TypeOps` 存放 Nabi 函数值（类型没有对应操作时为 `{0, 0}`），以 Nabi 调用约定调用：

| 字段 | 签名 | 含义 |
|-------|-----------|---------|
| `get` | `fn(owner: any, field: usize) -> any \| none` | 字段的值；`owner` 不是该类型时为 `none`。枚举的字段编号为 `variant << 16 \| index`，对其他变体返回 `none`；`usize::MAX` 返回 `owner` 所持变体的索引，类型为 `usize`。 |
| `set` | `fn(owner: any, field: usize, value: any) -> any \| none` | 设置了该字段的 `owner` 副本；`owner` 或 `value` 不匹配时为 `none` |
| `build` | `fn(variant: usize, fields: Dict<str, any>) -> (any \| none, str)` | 由具名字段构造的新值（枚举用 `variant`，结构体为 0），缺失的字段取其默认值；或者返回 `none` 和一条指出问题所在的消息 |
| `new_default` | `fn() -> any \| none` | 保留（`{0, 0}`）；不带字段调用 `build` 即可构造每个字段都取默认值的值 |
| `list` | `*const ListOps` | 列表和数组：`len: fn(any) -> usize`、`get: fn(any, usize) -> any \| none`、`items: fn(any) -> List<any> \| none`、`from: fn(List<any>) -> any \| none` |
| `dict` | `*const DictOps` | 字典：`len: fn(any) -> usize`、`items: fn(any) -> List<(any, any)> \| none`、`from: fn(List<(any, any)>) -> any \| none`、`reserved` |
