# 7.2. 分配

本文介绍[对象布局](701-object-layout.md)中定义的对象如何放入内存。分配算法分两个阶段定义：快速路径和慢速路径。

> [!NOTE]
> 本节是资料性的：它描述参考运行时的 GC，可能在任何版本中改变。只有在 [8.1](../008-abi/801-object-header-abi.md) 中作为 ABI 固定的对象布局是规范性的。

## 7.2.1. TLAB（线程局部分配缓冲区）

每个调度器工作线程（= 操作系统线程）都有自己的 TLAB。TLAB 由从 [Immix](../000-meta/007-glossary.md#immix) 空间获得的一个[块](../000-meta/007-glossary.md#block)（或其一部分）构成，通过 bump 指针快速分配。

```
TLAB structure:
  cursor : ptr      // next allocation position
  limit  : ptr      // end of the TLAB
  block  : ptr      // start (for reuse)
```

只要 cursor 低于 limit，空间就足够。如果 cursor 加上 size 超过 limit，[TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 就耗尽了，分配转入慢速路径。

## 7.2.2. 快速路径：4 条内联指令

> [!IMPORTANT]
> 大多数分配由内联代码处理。它们在 4 到 5 条机器指令内完成，同步开销为零。

```
1. tmp = cursor
2. new_cursor = tmp + size
3. if new_cursor > limit → goto slow
4. cursor = new_cursor
   (return tmp)
```

这条路径在 4 到 5 条机器指令内完成。分支几乎总是 `false`，走快速路径；而且 TLAB 本身是线程局部的，所以同步开销为零。

快速路径是编译器在每个分配点生成的内联代码；运行时不把它作为函数调用暴露。

## 7.2.3. 慢速路径：`nabi_gc_alloc_slow`

在 TLAB 耗尽或对象较大（[LOS](../000-meta/007-glossary.md#los-large-object-space)）时使用。

```
extern "C" {
    fn nabi_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> Ptr<u8>;
}
```

这是运行时提供的 `extern "C"` 函数；快速路径失败时，编译器调用它。

慢速路径执行以下操作：

1. 如果对象为 8 KiB 或更大，或其对齐大于 8 字节，就在 LOS 中分配（7.2.9）。
2. 大于一行、又放不进 TLAB 剩余空间的对象，从线程的中等区间 bump 分配。中等区间是第二段连续的空闲行，像 TLAB 一样补充，这样 TLAB 的尾部仍然可用。
3. 否则，补充 TLAB。如有需要，先执行一次 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 回收（7.2.8）。
4. 初始化头的 `type_id` 和 `gc_flags`。

## 7.2.4. 对象头初始化

对象头在慢速路径中初始化；在快速路径上，编译器也会在使用前内联填写头。

```
*(ptr + 0) = type_id
*(ptr + 4) = 0u32          // initial gc_flags
*(ptr + 8) = size          // gc_word: total object size (8.1.4)
```

载荷已经为零，由用户代码填入字段值。回收器可以把自己的簿记信息保存在头之前或旁表中。

## 7.2.5. 对齐

每次分配都保证 8 字节对齐。TLAB 的 cursor 保持为 8 字节的倍数，对象大小向上取整为 8 字节的倍数。需要更大对齐（`#[repr(align(N))]`）的对象在慢速路径上处理。

## 7.2.6. TLAB

TLAB 是 Immix 块内一段连续的空闲行。补充时，这段区间在交出前会被清零，因此快速路径拿到的是已填零的内存（刚从操作系统获得的块无需清零）。

线程拥有整块。每次回收结束时，回收器列出空闲块，以及含有空闲行的部分使用块；需要块的线程通过一个原子游标取列表的下一项（空闲块优先），并用该块的空闲段补充自己的 TLAB 和中等区间。补充不加锁：只有扩展堆和发起回收才会获取全局锁（world lock）。线程未用完的行会在下一次回收时回到列表中。

## 7.2.7. TLAB 与工作窃取

[调度器](706-scheduler.md)的工作窃取机制允许一个工作线程窃取另一个工作线程的任务。

被窃取的任务从窃取方工作线程的 TLAB 中分配，与原工作线程的 TLAB 无关。

这一模型之所以可行，是因为每个 TLAB 都以线程局部的方式绑定到其操作系统线程。

## 7.2.8. GC 触发

在以下情况下，慢速路径会发起回收。条件以及在 minor 与 full 之间的选择遵循 [7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-generational-collection)。

- 无法交出新的 TLAB 或块时。
- 自上次回收以来的分配量超过下一周期的预算时。块在被线程取走时就按整块计入。年轻代预算为上一周期中每个进行过分配的线程 8 MiB，最少 32 MiB，最多 128 MiB；较大的堆获得与老年代字节数相同的年轻代字节数，最多 128 MiB。`NABIR_GC_YOUNG` 可以精确设置它（[9.9.8](../009-tools/909-debug-tools.md#998-memory--gc-tools)）；设置了停顿目标时，它遵循该目标（[bare 1.6.12](../../refs-standard-libraries/100-bare/106-gc-pin.md#1612-set_pause_goal---a-pause-goal)）。
- 堆上限会限制预算（[bare 1.6.14](../../refs-standard-libraries/100-bare/106-gc-pin.md#1614-set_heap_limit---a-heap-limit)）。在不回收区间打开期间，任何分配都不会触发回收：超出预算时改为扩展堆（[bare 1.6.13](../../refs-standard-libraries/100-bare/106-gc-pin.md#1613-no_collect---a-region-without-collections)）。

`nabi_gc_collect` 显式执行一次完整回收，`nabi_gc_collect_young` 执行一次 minor 回收。

## 7.2.9. 大对象的 LOS 分配

8 KiB 或更大的对象，或需要大于 8 字节对齐的对象，绕过 TLAB，在 LOS 中逐个分配。把大对象放进 TLAB 会浪费剩余空间；而且 LOS 对象不会被移动，bump 分配也带不来好处。

对每个 8 KiB 或更大的对象，有两项保证：

- **对齐**。其载荷（紧跟在 16 字节头之后的地址）按 64 字节（`LOS_ALIGN`）对齐，即一条缓存行，足以满足所有 SIMD 宽度。要求更大对齐（`align > 64`）的缓冲区获得它自己要求的对齐。
- **地址稳定**。回收器永远不会移动它：只要对象可达，其载荷地址就保持不变。从它取得的指针（`Array::as_ptr`、`List::as_ptr`，[标准库 2.9.12](../../refs-standard-libraries/200-core/209-collections.md#2912-gcbuft---raw-gc-buffer)）可以不经固定就交给向量代码或 C（BLAS、LAPACK），前提是所有者在使用期间保持可达。较小的对象可能被移动（疏散，7.3.1）；当它的地址必须固定时，固定它（[7.2.10](#7210-pinned-objects)）。

## 7.2.10. 固定对象

在接收 GC 指针的 `extern "C"` 调用期间，无需单独固定（[7.4.1](704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)）。如果 C 在调用之后保留该指针，就用 `nabi_gc_pin` 固定它。固定只设置 `PINNED` 位，不复制对象。`nabi_gc_unpin` 解除固定。

## 7.2.11. 分配时间与堆使用量

快速路径是常数时间（约 4 条指令）。平均分配开销由快速路径决定。

## 7.2.12. 分配消除

如果某个分配的对象从不逃逸，并且在其存储之后从不被读取，优化器可以移除这个分配；这样的分配不是安全点，也不计入回收器的统计。大小无法表示的分配（`len * size + header` 溢出）总会到达运行时的“allocation too large”。
