# 8.2. ScanTable 序列化格式

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 的标记阶段必须沿着每个对象的子指针前进，因此需要一份说明子指针位于哪些偏移处的数据。这份数据称为 ScanTable；本文档定义它的序列化格式。

## 8.2.1. 用途

GC 标记循环的工作方式如下。

```
GC mark loop:
  for each live object O:
    type_id = O.type_id
    scan_table = lookup(type_id)
    for each (offset, kind) in scan_table:
      child = *(O + offset)
      mark(child)
```

其中 `lookup(type_id)` 使用的是编译器在进程启动时通过[对象头 ABI](801-object-header-abi.md) 中的 `nabi_gc_register_type` 注册的内容。

栈（影子栈）和 `static` 根也按同样基于字节偏移的方式遍历扫描。单个根槽被当作一个 ScanTable 条目处理，`any` 根还会覆盖其载荷字。

## 8.2.2. 注册调用

注册通过调用以下外部函数完成。

```
extern "C" {
    fn nabi_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
}
```

- `type_id`：被注册类型的 ID。
- `scan_table`：指向序列化 ScanTable 起点的指针。

编译器把 ScanTable 内存生成到 .rodata 中，因此它在进程的整个生命周期内都有效。运行时只读取这一区域。

## 8.2.3. 序列化格式

ScanTable 由一个 8 字节的头和其后的条目数组组成。

```
ScanTable Header (8 bytes, 4-byte aligned):
  offset  0:  total_size       u32         // total object size (16 + payload)
  offset  4:  version          u16  = 1
  offset  6:  num_entries      u16         // number of child pointer entries

Entry array (8 bytes each):
  offset  8 + 8*i + 0:  field_offset  u32  // offset within the object
  offset  8 + 8*i + 4:  field_kind    u32  // kind of child pointer
```

总大小为 8 + 8 × num_entries，每个字段都自然对齐。

### 头

头的各字段如下。

| 字段          | 类型 | 含义                                             |
|---------------|------|--------------------------------------------------|
| `version`     | u16  | 格式版本（位 0 ~ 14，当前 = 1）。位 15 = 数组表  |
| `total_size`  | u32  | 对象总大小（16 字节对象头 + 对齐后的载荷，至少 24，[8.1](801-object-header-abi.md)） |
| `num_entries` | u16  | 子指针条目的数量                                 |

如果 `version` 的位 15 为 1，该表是一个**数组表**。此时 `total_size` 是元素步长，每个条目的 `field_offset` 相对于元素起点，这些条目在整个载荷范围内对每个元素重复应用。编译器以这种形式写出 GC 缓冲区的元素表。

### 条目

每个条目指定对象中的一个子指针槽。

| 字段           | 类型 | 含义                             |
|----------------|------|----------------------------------|
| `field_offset` | u32  | 槽的偏移（从对象起点算起）       |
| `field_kind`   | u32  | 槽的种类                         |

### `field_kind` 的取值

`field_kind` 可以取的值及其含义如下。

| 值  | 含义                                                             |
|-----|------------------------------------------------------------------|
| 0   | `STRONG_REF`：普通 GC 对象指针（标记时跟随）                     |
| 1   | `WEAK_REF`：弱引用（标记时不跟随，回收时清除）                   |
| 2   | `INTERIOR_REF`：指向对象内部（转发时调整）                       |
| 3   | `ANY_SLOT`：`tag + payload` 形式的动态槽。检查标签后再标记       |
| 4   | `SUM_PAYLOAD`：和类型的标签以及各变体的引用（8.2.6）。位 8..31 存放描述符的位置 |
| 5   | `AMBIGUOUS`：可能是也可能不是引用的字。所指对象保持存活，但不会被移动（类型 id `2` 的缓冲区、无标签的 `union`） |
| 6 ~ | 保留 / 附加种类                                                  |

每种种类实际如何标记由 GC 实现决定；本规范只定义存在哪些种类及其编码。

## 8.2.4. 编码示例：`struct Point { x: f64; y: f64; }`

这个结构体没有子指针（所有字段都是基本浮点类型）。

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 header + 8 + 8
  num_entries  = 0
}
```

因此该 ScanTable 为 8 字节，只有头。

## 8.2.5. 编码示例：`struct Pair { a: List<i32>; b: List<f64>; }`

这个结构体有两个子指针。

```
ScanTable {
  version      = 1
  total_size   = 32        // 16 header + 8 (List ptr) + 8
  num_entries  = 2

  entry[0]: field_offset = 16, field_kind = STRONG_REF (0)
  entry[1]: field_offset = 24, field_kind = STRONG_REF (0)
}
```

总大小为 8 + 16 = 24 字节。

## 8.2.6. 编码示例：`enum` 的变体

```
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

存放该类型的对象由 16 字节的对象头、标签（4 字节）、填充（4 字节）和最大变体的载荷（16 字节，一个 `str` 指针和长度）组成，共 40 字节。

各变体中引用所在的位置不同，因此使用一个 `SUM_PAYLOAD` 条目及其描述符。

- 条目的 `field_offset` 是标签（u32）的偏移。
- `field_kind` 的低字节为 `4`；位 8..31 是描述符的位置（以从表起点算起的 u32 字为单位）。
- 描述符先存放 `n`，然后为每个含引用的变体存放 `{tag, num_entries, entries_index}`。
- 变体的条目是 `{相对于标签的偏移, field_kind}`。嵌套的和类型同样写作 `SUM_PAYLOAD`。
- 标签不在描述符中的变体不含引用。

```
ScanTable {
  version = 1, total_size = 40, num_entries = 1
  entry[0]: field_offset = 16, field_kind = 4 | (desc << 8)    // tag at offset 16
  desc:     n = 1
            { tag = 1 (Err), num_entries = 1, entries_index = e }
  e:        { offset = 8, field_kind = INTERIOR_REF }           // str ptr of Err (into a payload)
}
```

枚举 / 和类型载荷中的引用是精确的，因此经由载荷到达的对象也可以被疏散移动。`AMBIGUOUS` 只用于类型 id `2` 的缓冲区以及用户 `union` 的存储，后者没有标签，无法解释。

## 8.2.7. ANY 槽

`any` 槽（[`any` 槽](../002-types/202-any-slot.md)）可以以 `tag + payload` 的形式存放任意类型的值。

```
ANY slot layout.
  offset  0:            tag       u32
  offset  ptr-aligned:  payload   usize (numbers/bool/char up to pointer width stored directly, everything else as a box ptr)
```

对应的 ScanTable 条目如下。

```
field_offset = (offset of the slot)
field_kind = ANY_SLOT (3)
```

载荷是否为装箱引用由标签携带。

- 如果标签的**位 31** 为 1，载荷是装箱引用；为 0 时，载荷就是值本身。
- 对标签做类型比较时，使用屏蔽掉位 31 后的值。

因此 GC 无需任何额外的表就能精确扫描载荷，并改写被移动的装箱的地址。分别编译的包之间也无需合并表。

## 8.2.8. WEAK_REF

弱引用不会让对象保持存活。

- 标记阶段不跟随弱 `ref`。
- 清除之后，如果弱 `ref` 所指的对象已被回收，该弱 `ref` 槽被清为 0。

这一机制适用于缓存、观察者模式等场景；其库接口是 `Weak<T>`（[标准库 1.6.9](../../refs-standard-libraries/100-bare/106-gc-pin.md#169-weakt弱引用)）。

## 8.2.9. INTERIOR_REF

INTERIOR_REF 指向对象内部的某个位置（而不是对象起点）。

- 它来自切片、部分视图等。
- 转发时，按新旧对象之间的相对偏移进行调整。

具体处理方式由 GC 实现决定。

## 8.2.10. 序列化的字节序

该表是按目标字节序存放的 `u32` 字数组，运行时以本机字读取。内置的[平台目标](../000-meta/006-platform-targets.md)都是小端，只有 `s390x-unknown-linux-gnu`（进行中）例外，其字为大端。

## 8.2.11. 变更的兼容性

`version` 字段标识格式。对格式的变更遵循[概述](README.md#稳定性)中的稳定性策略。预构建的包只与构建它的那个 `nabic` 一起使用，因此程序注册的表与读取这些表的运行时总是来自同一工具链。

## 8.2.12. 运行时保留的类型 id

以下三个类型 id 由运行时直接定义；编译器不为它们注册 ScanTable（[8.1.2](801-object-header-abi.md#812-type_id偏移-0u32)）。

| 类型 id | 含义              | 扫描                                                    |
|---------|-------------------|---------------------------------------------------------|
| `1`     | 原始数据          | 载荷不含引用，不扫描：GC 字符串、字节缓冲区，以及 `T` 不含 GC 引用（数值、`bool`、`char`、`Pod` 结构体）的所有 `GcBuf<T>` |
| `2`     | 按字追踪的缓冲区  | 载荷的每个字都作为含糊字（`AMBIGUOUS`）扫描             |
| `3`     | 弱单元（`Weak<T>`） | 一个 `WEAK_REF` 字（8.2.8）；由 `nabi_gc_weak_new` 创建 |

id `1` 和 `2` 通过以类型 id `1` 或 `2` 调用 `nabi_gc_alloc_slow` 来分配（[8.5.1](805-extern-symbols.md#851-gc-区域)）。标准库的 [`GcBuf<T>`](../../refs-standard-libraries/200-core/209-collections.md#2912-gcbuft原始-gc-缓冲区) 建立在其之上：编译器以 id `1` 分配不含引用的 `T` 的缓冲区，因此无论数值数据多大，回收器都不必为其花费时间（一个存活的 256 MiB `List<f64>` 不会给完整回收带来可测量的开销；同样的字节若作为类型 `2` 缓冲区，每次回收约需 140 ms），其中的任何数值也不会让垃圾保持存活。`T` 含引用的缓冲区会得到 `T` 的数组 `ScanTable`（`version` 的位 15，8.2.3），只扫描引用槽。类型 id `2` 仅用于运行时的字缓冲区。

## 8.2.13. 注册时机

注册在进程启动时批量完成。

```
process_start:
  for each registered type:
    nabi_gc_register_type(type_id, &scan_table)
  ↓
  user main()
```

编译器生成一个汇集所有注册调用的函数 `nabi.register_types`。在 `-O0` 以上，当包以连续 id 注册四个或更多表时，改为用一次调用注册：`nabi_gc_register_types(base, tables, n)` 把指针数组 `tables` 中的 `n` 个表注册到 id `base`、`base + 1`、.. 之下（[8.5.1](805-extern-symbols.md#nabi_gc_register_type--nabi_gc_register_types)）。编译器生成的 C ABI `main` 在 `nabi_runtime_init` 之后立即调用该函数，然后进入用户入口点。

```
main(argc, argv):                // compiler-generated, C ABI
  nabi_runtime_init()
  nabi.register_types()           // collection of nabi_gc_register_type calls
  → user entry point (4.16)
```

GC 对象分配的慢速路径是 `nabi_gc_alloc_slow(size, align, type_id)`（[7.2.3](../007-runtime/702-allocation.md#723-慢速路径nabi_gc_alloc_slow)），编译器分配的类型 id 从 `0x10000` 开始（[8.1.2](801-object-header-abi.md#812-type_id偏移-0u32)）。

一个表的字节在程序中只出现一次：每个包的 `nabi.register_types` 引用一个以表内容命名的隐藏定义，链接时只保留一份，即使多个包以各自的 id 注册相同的布局也是如此。
