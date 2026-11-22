# 8.1. 对象头 ABI

本节定义[对象布局与内存空间](../007-runtime/701-object-layout.md)中 16 字节对象头的位布局。头大小、`gc_flags` 各位、LOS 阈值、类型 id 的起始值以及运行时缓冲区的类型 id 等 GC 常量，都来自编译器与运行时共用的同一份定义（`nabir_abi::gc`）。

## 8.1.1. 精确布局

对象头的形式如下。

```
+---------+---------+-----------------+----------------+
| offset  |  size   |  field          |  type          |
+---------+---------+-----------------+----------------+
|    0    |    4    |  type_id        |  u32           |
|    4    |    4    |  gc_flags       |  u32           |
|    8    |    8    |  gc_word        |  u64 / ptr     |
|   16    |    var  |  payload        |  per type_id   |
+---------+---------+-----------------+----------------+
```

> [!IMPORTANT]
> 对象头总大小为 16 字节，按 8 字节对齐，在 32 位和 64 位目标上完全相同。

每个对象至少 24 字节（对象头加一个载荷字，即 `nabir_abi::gc` 中的 `MIN_OBJECT_SIZE`），因此编译后的代码可以读取它持有引用的任何对象的第一个载荷字。

## 8.1.2. `type_id`（偏移 0，u32）

`type_id` 是类型的唯一标识符。它是按目标字节序存放的 32 位无符号值（除 `s390x` 外的所有目标均为小端，[8.2.10](802-scan-table.md#8210-序列化的字节序)）；值 `0` 保留，表示空类型或无效对象。进程启动时，编译器调用 `nabi_gc_register_type` 注册每个 type_id。

### type_id 的分配

在一次构建内，同一类型具有同一 id。id 在不同构建之间不稳定，因此不用于序列化。

### 保留范围

保留的 id 范围如下。

| 范围           | 用途                        |
|----------------|-----------------------------|
| `0`            | 无效 / 空                   |
| `1`            | 运行时（GC 字符串，不追踪） |
| `2`            | 运行时（按字追踪的缓冲区）  |
| `3`            | 运行时（弱单元，`Weak<T>`） |
| `4` ~ `0xFFFF` | 保留给运行时                |
| `0x10000` ~    | 所有编译出的类型（标准库和用户包） |

编译器为它生成的每个类型（包括标准库中的类型）分配 `0x10000` 及以上的 id。运行时忽略低于 `0x10000` 的类型注册。

## 8.1.3. `gc_flags`（偏移 4，u32）

`gc_flags` 字段存放 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 状态位的确切位置，布局如下。

```
bit  0  : MARK       (mark for the current GC cycle)
bit  1  : STICKY     (sticky mark-bit, old generation)
bit  2  : PINNED     (must not move, FFI)
bit  3  : FORWARDED  (after evacuation; gc_word is the new location)
bit  4  : LARGE      (LOS object; gc_word is the size)
bit  5  : reserved   (unused)
bit  6 ~ 31 : reserved / GC implementation-defined
```

以下各位取自实现保留范围（[7.4.2](../007-runtime/704-barriers-and-roots.md#742-写屏障对象记忆)）。

| 位   | 名称       | 含义                                                                                       |
|------|------------|--------------------------------------------------------------------------------------------|
| 6    | `UNLOGGED` | 在一次回收中存活下来的对象上与 `STICKY` 一同置位。写屏障第一次把该对象记入记忆集时清除 |
| 7    | `CARDED`   | 在 8 KiB 及以上的老对象上代替 `UNLOGGED` 置位。屏障把一张 512 B 的卡标记为脏。卡表指针位于对象头之前紧邻的字 |
| 8    | `NOMOVE`   | 地址被当作数值使用的对象。无论生命周期如何都不会被移动。由 `nabi_task_key`（[8.5](805-extern-symbols.md)）置位 |
| 9    | `NOEVAC`   | 回收器内部使用，作用于一次完整回收：某个不确定字（一个根，或运行时字缓冲区（类型 id `2`）中的一个字）指向该对象，因此该次回收的疏散会把它留在原处（[7.3.1](../007-runtime/703-gc-algorithm.md#机会式疏散)）。在疏散前对候选块中的此类对象置位，疏散访问到它们时清除 |
| 10   | `SATB`     | 在并发标记期间，屏障记录该对象的引用时置位。相互竞争的存储会等待它被清除（[7.5.3](../007-runtime/705-concurrent-collection.md#753-satb-屏障)） |
| 11   | `SCANNED`  | 该对象已在并发标记中被扫描 |
| 12   | `WIPE`     | 机密缓冲区（`nabi_gc_secret_new`，[8.5.1](805-extern-symbols.md#nabi_gc_secret_new--nabi_gc_secret_wipe)）。释放它的清除先把其载荷清零。永不清除 |
| 13 ~ 28 | `VISIT_STAMP` | 供 `par` 访问使用（[标准库 2.19.1](../../refs-standard-libraries/200-core/219-par.md#2191-核心-api)）：最近一次到达该对象的访问的 16 位纪元。回收器保留这些位 |
| 29   | `VISIT_OTHERS` | `par` 访问：被不止一个项到达，且每个都经由切片 |
| 30   | `VISIT_SLICED` | `par` 访问：到目前为止只经由切片到达 |
| 31   | `VISIT_CUR` | `par` 访问：被当前正在访问的项到达 |

### 原子性要求

在 STW 回收中，`MARK` / `STICKY` / `FORWARDED` 是非原子的。并发标记期间标记器与 mutator 会同时触及的 `MARK` / `SATB` / `SCANNED`，以及 mutator 之间会竞争的 `UNLOGGED` 清除操作，都是原子的（[7.4.2](../007-runtime/704-barriers-and-roots.md#742-写屏障对象记忆)）。

## 8.1.4. `gc_word`（偏移 8，u64 或 ptr）

`gc_word` 是对象的总大小（包括 16 字节的对象头，是 8 的倍数）。它同时适用于 Immix 对象和 LOS 对象，由分配快速路径以内联方式写入（[8.9.9](809-lowering-control.md#899-gc-分配快速--慢速路径)）。

只有在完整回收的疏散期间，被移动对象的 `FORWARDED` 位才会置位，此时 `gc_word` 指向新地址。所有引用都在同一次回收内修正（[7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)），旧位置随后被丢弃，因此 mutator 永远观察不到转发。

块内的对象的 `type_id` 永远不为 0。回收器用 0 标记空闲空间。

### 回收器私有信息

回收器可以把自己的簿记信息放在对象头之前（低于对象地址的地址处），或放在单独的旁表中。编译器和其他代码只能看到从对象地址开始的对象头和载荷；此范围之外的一切都属于回收器实现。

## 8.1.5. `payload`（偏移 16 起）

编译器按照为该 `type_id` 注册的字段布局确定载荷区域。格式如下。

- 对于 [`struct`](../002-types/203-struct.md) / [`enum`](../002-types/204-enum.md) / [`union`](../002-types/205-union.md)，布局由编译器自行决定。
- 带 `#[repr(C)]` 时，遵循 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)。
- 带 `#[repr(packed)]` 时，忽略对齐。

在 GC 标记阶段，子指针的偏移通过 `type_id → ScanTable` 映射查得（格式定义见 [ScanTable 序列化格式](802-scan-table.md)）。

## 8.1.6. 对齐要求

各位置的对齐要求如下。

| 位置                     | 对齐                                |
|--------------------------|-------------------------------------|
| 对象起点（对象头）       | 8 字节                              |
| payload 起点（偏移 16）  | 8 字节（由于对象头为 16 字节，自动满足） |
| 每个字段                 | 其类型的对齐                        |

如果 `payload` 中某个字段要求大于 8 字节的对齐（例如 `#[repr(align(16))]`），载荷起点按该值对齐：对象头位于对齐地址之前 16 字节处，其前面的空隙属于回收器（[8.1.4](#814-gc_word偏移-8u64-或-ptr)）。

## 8.1.7. 对象头 + 载荷的总大小

对象头加载荷的总大小按如下方式计算。

```
total_size = 16 + align_up(payload_size, max(8, payload_align))
```

其中 `align_up(x, a) = (x + a - 1) & ~(a - 1)`。

这个大小就是 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) bump 指针的增量（见[分配](../007-runtime/702-allocation.md)）。

## 8.1.8. 示例：i32 对象

存放一个 i32 的 GC 对象，其布局如下。

```
offset  0:  type_id = ID_I32     (4 bytes)
offset  4:  gc_flags = 0          (4 bytes)
offset  8:  gc_word = 24          (total size, 8 bytes)
offset 16:  i32 value             (4 bytes)
offset 20:  padding               (4 bytes, for 8-byte alignment)
total: 24 bytes
```

单个 i32 共占 24 字节：16 字节的对象头、4 字节的值和 4 字节的填充。

对于小对象，这一开销看起来很大，但大多数对象的载荷更大（复合结构），所以比例会降低。

## 8.1.9. 示例：大对象（List<i64> 的缓冲区）

一个载荷为 16 KB 的列表缓冲区示例：

```
offset    0:  type_id = 1              (4 bytes, raw data: no reference)
offset    4:  gc_flags = LARGE          (4 bytes)
offset    8:  gc_word = 16400            (total size, 8 bytes)
offset   16:  i64[2048]                 (16384 bytes)
total: 16 + 16384 = 16400 bytes
```

`LARGE` 位已置位，因此这是一个 LOS 对象。`gc_word` 是总大小（8.1.4）。元素不含引用，所以该缓冲区使用运行时类型 id `1`，不会被扫描（[8.2.12](802-scan-table.md#8212-运行时保留的类型-id)）。

## 8.1.10. 对象头变更时的兼容性

这一对象头布局是 ABI 中最核心的部分。它只在允许破坏 ABI 的版本中变更（[概述](README.md#稳定性)）：1.0 之前为 0.x.0 版本，1.0 起为主版本。0.x.y 版本（1.0 起为次版本或修订版本）保持不变。

得益于这种稳定性，由同一工具链构建的用户包、标准库和运行时彼此兼容。
