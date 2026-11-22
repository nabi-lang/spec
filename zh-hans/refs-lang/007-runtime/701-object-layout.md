# 7.1. 对象布局与内存空间

每个由 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 管理的对象都以一个固定的头开始。本文定义头的格式，以及对象可以驻留的三种内存空间。

> [!NOTE]
> 本节是资料性的，但 16 字节的头除外，它在 [8.1](../008-abi/801-object-header-abi.md) 中作为 ABI 固定。内存空间描述的是参考运行时，可能在任何版本中改变。

## 7.1.1. 16 字节头

每个 GC 对象的前 16 个字节是头。

```
offset  0 :  type_id    u32       // type identifier
offset  4 :  gc_flags   u32       // GC state bits
offset  8 :  gc_word    u64       // total object size (new address during evacuation)
offset 16 :  payload ...           // the object's actual fields
```

头和载荷都按 8 字节对齐。该布局在 32 位和 64 位上相同，因此在 32 位目标上头也是 16 字节。

### `type_id`（偏移 0，4 字节）

类型的唯一标识符，由编译器为每个类型分配。

它有多种用途。在动态分派中，它充当 [`any` 槽](../002-types/202-any-slot.md)的标签；在 GC 字段扫描中，它通过 `type_id → field layout` 映射定位字段；在诊断中，它用于显示对用户友好的类型名。

`type_id` 在进程的整个生命周期内保持稳定；同一类型始终具有相同的 id。

### `gc_flags`（偏移 4，4 字节）

表示 GC 状态的位。

| 位          | 含义                                                                                    |
|-------------|-----------------------------------------------------------------------------------------|
| `mark`      | 在当前 GC 周期中是否已到达                                                              |
| `sticky`    | [sticky mark-bit](../000-meta/007-glossary.md#sticky-mark-bit)（分代跟踪）              |
| `pinned`    | 标记为不可移动，例如用于 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) |
| `forwarded` | 已被疏散到新位置（gc_word 是转发指针）                                                  |
| `large`     | [LOS](../000-meta/007-glossary.md#los-large-object-space)（大对象空间）对象             |
| ...         | 其他位由 GC 实现决定                                                                    |

具体的位排布由 GC 实现决定；此表只列出语义上必需的位的种类。

### `gc_word`（偏移 8，8 字节）

对象的总大小（包括头）。对于在疏散中被移动的对象，会设置 `forwarded` 位，此槽指向新位置（[8.1.4](../008-abi/801-object-header-abi.md#814-gc_word-offset-8-u64-or-ptr)）。

### `payload`（偏移 16 起）

存放对象字段的区域。字段顺序和对齐由编译器决定；这同样适用于 [`enum`](../002-types/204-enum.md) 和 [`union`](../002-types/205-union.md)。[`struct`](../002-types/203-struct.md) 值本身不是堆对象；当它存放在某个堆对象（列表缓冲区、闭包环境、协程帧）中时，会内联布局在这里（[8.8.3](../008-abi/808-lowering-types.md#883-representation-of-struct)）。

如果附加了 `#[repr(C)]` 这样的属性，则适用相应的表示规则（[内置属性](../006-meta-system/602-built-in-attributes.md)）。

## 7.1.2. 内存空间

GC 把每个对象放在以下三种空间之一。字面量以及 `const` / `static` 数据不是堆对象：它们位于程序的数据节中，在这些空间之外，GC 既不标记也不移动它们（持有 GC 引用的 `static` 是根，[7.4.1](704-barriers-and-roots.md)）。

### 1. Immix

大多数普通对象所在的空间，使用 Immix mark-region 算法。

它被划分为 32 KiB 的[块](../000-meta/007-glossary.md#block)和 128 B 的行。默认采用标记-清除，完整回收可以把对象移出碎片化的块（机会式疏散）。代由 sticky mark-bit 区分（[7.3](703-gc-algorithm.md)）。

### 2. 大对象空间（LOS）

8 KiB 或更大的对象、需要大于 8 字节对齐的对象，以及机密缓冲区（`nabi_gc_secret_new`，[8.5.1](../008-abi/805-extern-symbols.md#nabi_gc_secret_new--nabi_gc_secret_wipe)），在 LOS 中逐个分配，并设置 `gc_flags` 的 `LARGE` 位。LOS 用标记-清除回收，不移动对象。老的 LOS 对象带有卡表（[7.4.2](704-barriers-and-roots.md#742-write-barrier-object-remembering)）。机密缓冲区带有 `WIPE` 位（[8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)）：释放它的清除阶段会先用零覆写其载荷。

### 3. 固定对象（FFI 固定）

传给 [`extern` 块](../005-modules/509-extern-block.md)中 FFI 函数的 GC 指针在调用结束前不会被移动，因为该实参位于根中（[7.4.1](704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)）。如果外部代码需要在调用之后保留该地址，就用 `nabi_gc_pin` 固定它。固定的对象会设置 `gc_flags` 的 `PINNED` 位并保持原位。`nabi_gc_unpin` 解除固定。

固定对象仍由标记-清除回收，但永远不会被疏散。

## 7.1.3. 空间之间的移动

```
TLAB (allocation) → Immix
Immix → Immix         (evacuation in full collection)
Immix / LOS ↔ pinned  (pin / unpin, in place)
```

对象不会在空间之间移动。只有疏散会改变地址。

## 7.1.4. 头宽度的动机

16 字节的头是时间与空间权衡的结果。

对于小对象，比如 4 字节的 i32，16 字节的头会损害内存效率。但 16 字节可以同时容纳 type_id、标志和转发信息，从而简化算法；而且布局在 32 位和 64 位上保持一致，也简化了跨平台代码。

在大多数对象为 4 字节或更大的假设下，这一权衡是合理的选择。

## 7.1.5. 注册 type_id：`nabi_gc_register_type`

每个类型的字段布局必须向 GC 注册。

```
nabi_gc_register_type(type_id, scan_table)
```

`type_id` 是类型的标识符，`scan_table` 存放指向 GC 对象的指针字段的偏移以及附加元数据。

标记对象时，GC 用其 `type_id` 找到 `scan_table`，并通过它跟随子指针。

编译器在进程启动时通过 `extern "C"` 调用完成这一注册；注册 ABI 定义在 [Extern "C" 符号](../008-abi/805-extern-symbols.md)中。

## 7.1.6. 计算对象总大小

```
size = 16  (header) + payload_size  (aligned size of the fields)
```

分配器为每个对象（无论是 Immix 还是 LOS）把这个总大小存入头的 `gc_word`（[8.1.4](../008-abi/801-object-header-abi.md#814-gc_word-offset-8-u64-or-ptr)），回收器从那里读取它。
