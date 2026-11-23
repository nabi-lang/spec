# 9.4. NABIHIR 格式

本节定义预构建库包的接口文件 `<name>.nabihir`。[NABIHIR](../000-meta/007-glossary.md#nabihir-nabi-hir-bytecode) 文件位于该包的 `lib<name>.a`（[包归档文件](903-package-archive.md)）旁边，不是该归档文件的成员。

## 9.4.1. 动机

[AOT 构建流水线](902-aot-pipeline.md)中的单态化按调用处的具体类型进行[特化](../000-meta/007-glossary.md#specialize)。因此，当另一个包调用某个包的[泛型](../000-meta/007-glossary.md#generic泛型) `fn` 时，特化必须在该调用处进行。只有[具体 fn](../000-meta/007-glossary.md#concrete-fn--generic-fn) 的目标代码是不够的，所以另外提供一个供下游读取的接口。

单态化的输入是 AST。因此 NABIHIR 的主体是**剥离后的包源代码**，由下游的解析器读取。

## 9.4.2. 不使用外部库

根据零依赖原则，序列化库也不使用；NABIHIR 是从头设计的。

## 9.4.3. 整体布局

```
NABIHIR file (little-endian):
+-----------------------------------+
|  Header                           |  (16 bytes fixed)
|    magic       "NABH" (4 bytes)   |
|    version     u32  (= 1)         |
|    flags       u32                |
|    body_offset u32                |
+-----------------------------------+
|  Producer string        str       |  "nabic <version> (<compiler id>)"
+-----------------------------------+
|  Meta                             |
|    name                 str       |  mount name (--extern <name>=)
|    hash                 str       |  package hash (0.3.3)
|    source dir           str       |
|    gc type offset       varint    |
|    gc type count        varint    |
|    features  varint n, n × str    |
+-----------------------------------+
|  Body (at body_offset)            |
|    file count           varint    |
|    file × n: path str, text str   |
+-----------------------------------+
|  Summaries                        |
|    count                varint    |
|    n × (function str, summary str) |
+-----------------------------------+
|  Call summaries                   |
|    count                varint    |
|    n × (function str,             |
|         flags varint,             |
|         keeps_none varint,        |
|         returns varint)           |
+-----------------------------------+
```

`str` 是前面带有 PrefixVarInt 长度（9.4.5）的 UTF-8 字节。没有字符串 / 类型 / 项的节。

## 9.4.4. 头部与生成者

| 字段         | 大小 | 含义                                                      |
|---------------|------|--------------------------------------------------------------|
| `magic`       | 4    | `"NABH"`                                                     |
| `version`     | u32  | 格式版本（`1`）。布局或主体内容改变时递增 |
| `flags`       | u32  | 保留（0）                                                 |
| `body_offset` | u32  | 主体的起始偏移                                     |

生成者字符串为 `nabic <version> (<compiler id>)`。编译器 id 是该 `nabic` 可执行文件的一个戳记，每当该文件被写入或替换时都会改变：它的大小、修改时间、inode 和变更时间（`<size>-<mtime>-<inode>-<ctime>.<ctime_nsec>`，十六进制；在 Windows 上，以文件 id 和创建时间代替 inode 和变更时间）。

## 9.4.5. VarInt 编码

PrefixVarInt 是 LEB128 的一种变体。第一个字节开头 1 位的个数就是后续字节的个数，值以大端序占据其余的位。

```
1-byte int  : 0xxxxxxx                    (0 ~ 127)
2-byte int  : 10xxxxxx xxxxxxxx           (0 ~ 16383)
3-byte int  : 110xxxxx xxxxxxxx xxxxxxxx  (0 ~ 2097151)
...
```

## 9.4.6. 元数据

| 字段            | 含义                                                                           |
|------------------|-----------------------------------------------------------------------------------|
| name             | 包的挂载名。它出现在符号中，因此必须等于 `--extern <name>=` |
| hash             | 包哈希（[0.3.3](../000-meta/003-source-units.md#033-包标识符)）。用于符号修饰的值。对于标准库包 `nabi`，它是一个 `pkg=hash;...` 列表，每个层级一项 |
| source dir       | 构建时存放包源代码的目录。是主体中文件路径的基准；下游读取主体，而不是磁盘 |
| gc type offset / count | 本包的 GC 类型 id 范围 `[offset, offset + count)`（9.4.8）          |
| features         | 构建该包时使用的 `feature` cfg 值。下游用构建的其他 cfg 和这些特性解析主体 |

## 9.4.7. 主体

主体以（路径，文本）对的形式保存包的**每个源文件**。每个文件按以下方式剥离。

- 具体函数的主体被截成 `;`。换行保留，因此诊断的行号与原文件一致。这些函数作为归档文件的符号链接。
- 下游必须单态化或内联的主体保留：泛型函数、`async` / `yield` 函数、`#[inline]` 函数、`extern "C"` 定义、trait 的默认方法，以及泛型 `impl` 的方法。
- 签名、类型、trait、`impl` 和 `use` 等声明原样保留。
- 保留文本中的 `include_str!` / `include_bytes!` 被替换为它们读取到的字面量（保留换行），因此下游不读取任何被包含的文件，移动后的库仍然可以工作。

源文件之后是包的 `ffi::wit` 项所读取的 `.wit` 文件（[标准库 1.8](../../refs-standard-libraries/100-bare/108-wit.md)），连同其 `deps/` 整个保存。它们的路径相对于 source dir，可以通过 `..` 离开该目录。下游从主体读取 WIT，而不是从磁盘读取。

主体之后是包中各函数的移交摘要（[标准库 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync线程边界标记)）。摘要由若干行以制表符分隔的字段组成：

- `g <root> <mutex|rwlock|channel> <line> <function> <file>`：移交 `<root>`；
- `s <root> <root>`：同一个对象；
- `i <root> <root>`：前者离开时会带走后者；
- `t <guard parameter> <root>`：绑定到该守卫。
- `x <tag|*> [<root>]`：在返回值带有该标签的出口上（`*`：未知；也可以是 `bool` 的 0 / 1、和类型或枚举的变体，或载体的 1（错误）、0（值）、2 + 值的标签），`<root>` 属于被移交的对象；没有 `x` 行时，每个出口都移交所有 `g` 根。

根是一个参数索引或 `r`（返回值），后跟以 `.` 分隔的字段路径（元组索引设置第 31 位）。名为 `#fn`（`#fne`：闭包，环境在前）、后跟以 U+001F 分隔的参数类型键的条目，把以这些参数类型作为值使用的函数的摘要合并起来。

最后是调用摘要：说明调用代码位于归档文件中的函数会做什么，使优化的下游构建不必把它当作可能做任何事。条目按函数名（即下游声明它时的项路径）排序；什么都不知道的函数没有条目，泛型实例（下游自己生成）和 `any` 分派器（由程序的分派器取代）也没有。每条事实都对归档文件中的代码成立：

| 字段        | 含义 |
|--------------|---------|
| `flags`      | 第 0 位 `readonly`：不写入调用者可以读取的任何内存（它仍然可以分配、到达安全点和 panic）；第 1 位 `quiet`：从不到达 GC 安全点（不分配、不轮询、不阻塞等待），因此调用不会移动对象；第 2 位 `cold`：`#[cold]`，或者从不返回 |
| `keeps_none` | 第 `i` 位：参数 `i` 的值不会被存储、捕获或传到可能被保留的地方，并且只有在 `returns` 这样说明时，才会在调用之后存活于结果中 |
| `returns`    | 第 `i` 位：结果可能是参数 `i` 的值，或从它派生而来 |

下游只在高于 `-O0` 时读取它们。`readonly` 调用不会破坏任何加载，因此在它之前加载的值可以在它之后复用，加载也可以跨越它移动；`quiet` 调用对其调用者来说不是安全点；`cold` 调用把它所在的块标记为很少运行；传入 `keeps_none` 参数的栈地址不会逃逸。panic 不会被移动：调用保持在原处。

## 9.4.8. 分离编译的规则

预构建的库无法一次看到整个程序，因此各个包按以下规则协调。

- **`any` 标签**。`any` 槽的标签是类型名的 31 位哈希，因此分别构建的包对同一类型使用同一个标签。
- **`any` 种子**。为了让程序知道库放入 `any` 的类型，库会在主体末尾追加一个使用这些类型的小型 Nabi 函数。程序的封闭世界 `any` 分派器也包括这些类型。
- **分派器符号**。`any` 分派器是符号 `nabi.any.<hash>`，在库中定义为弱符号，在程序中定义为强符号。链接时程序的定义生效。
- **和类型成员的顺序**。和类型的各项按结构键排序，而不是按内部类型编号排序（[2.6.4](../002-types/206-sum-types.md#规范顺序)），因此标签在各个包之间一致。
- **GC 类型 id**。每个包都有自己的 GC 类型 id 范围（9.4.6）。同一程序中两个包的范围重叠是错误。
- **方法符号**。依赖包的方法按稳定路径 `<module>::<type>.<method>[__by_<trait>]` 修饰（[9.5](905-symbol-mangling.md)）。
- **调用摘要**。下游构建的代码依赖于它所调用函数的调用摘要（9.4.7）。增量构建的代码生成单元以其在其他单元和包中的被调用者的摘要作为键，因此摘要发生变化的依赖在重新构建后，只重新编译调用了变化函数的那些单元。

## 9.4.9. 读取与拒绝

`--extern name=<name>.nabihir` 或 `--extern name=lib<name>.a` 都会找到另一个同级文件（[9.2.3](902-aot-pipeline.md#923-依赖解析)）。下游把主体作为源代码覆盖层解析以获得声明，把被剥离的函数声明为预构建符号，就地单态化泛型，然后链接归档文件。

以下任何一项不同时，NABIHIR 会被拒绝，并报告“需要重新构建”的错误。

- `version`
- 生成者（由不同 `nabic` 生成的 NABIHIR）
- 名称（`--extern` 名称与 NABIHIR 的挂载名）

## 9.4.10. NABIHIR 检查

`nabi inspect --dump <file>` 打印 NABIHIR 的版本、生成者、元数据（包括 `features:`）、接口文件的路径和文本、摘要以及调用摘要（`--- calls`：每个函数一行，给出其名称和已设置的事实）（[9.9.2](909-debug-tools.md#992-nabihir-检查)）。`nabi inspect`（[9.9.14](909-debug-tools.md#9914-包检查)）把摘要与归档文件一起显示。

## 9.4.11. 稳定性

只读取由同一个 `nabic` 生成的 NABIHIR（9.4.9）。格式改变时，`version` 递增，不同的版本需要重新构建。适用[概述](../008-abi/README.md)中的稳定性策略。

标准库不以 NABIHIR 形式分发（[9.3.6](903-package-archive.md#936-标准库)）。
