# 7.4. 屏障与根跟踪

本文定义 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 的标记起点（根），以及老 → 年轻引用的记录（写屏障）。

> [!NOTE]
> 本节是资料性的：它描述参考运行时的 GC，可能在任何版本中改变。只有在 [8.1](../008-abi/801-object-header-abi.md) 中作为 ABI 固定的对象布局是规范性的。

## 7.4.1. 根跟踪：编译器直接生成的影子栈

### 选择影子栈

最常见的两种根跟踪方式是：

| 方式                        | 优点                          | 缺点                                |
|-----------------------------|-------------------------------|-------------------------------------|
| 栈映射                      | 精确，每次调用无额外工作      | 每个目标都需要一种映射格式和一次栈遍历；WASM 无法遍历自己的栈 |
| 影子栈（直接生成）          | 每个目标上的代码相同          | 额外的栈内存以及 push/pop           |

受支持的[平台目标](../000-meta/006-platform-targets.md)包括 32 位 ARM（`armv7`）、32 位 x86（`i686`）和 WASM。WASM 代码无法检查自己的栈，因此唯一适用于所有这些目标的方式是[影子栈](../000-meta/007-glossary.md#shadow-stack)，每个线程一条链。

> [!IMPORTANT]
> Nabi 的根跟踪是影子栈，其帧的 push / pop 由编译器直接生成。链头是 `nabi_tls` 偏移 0 处的槽（`nabi_tls_shadow_stack_top`），每个线程一个（[8.3.8](../008-abi/803-tls-slots.md#838-编译器生成的形式)）。它在每个受支持的目标上（包括 32 位 ARM / x86 / WASM）都以相同方式工作。

### 影子栈的结构

每个操作系统线程都有自己的影子栈：由放在各函数机器栈上的帧通过 `prev` 连成的链表。

```
nabi_tls_shadow_stack_top ──► frame N      (current function)
                             ├── prev ──► frame N-1 (caller) ──► ... ──► null
                             ├── count = 2
                             ├── slots[0]  ──► &word (local x)      ──► GC object
                             └── slots[1]  ──► &word (spill slot)   ──► GC object
```

```
ShadowFrame {
    prev:  ptr          // caller's frame (null if none)
    count: usize        // number of slots (pointer width)
    slots: [ptr; count] // addresses of words holding GC references
}
```

在 `-O0` 以上，一个 `(n << 1) | 1` 槽后跟一个地址，表示从该地址开始的 `n` 个相邻字；`count` 为 0 的帧用它唯一的槽指向一个静态描述符，该描述符以 16 位条目保存列表，每个地址都是相对于帧的偏移（[8.9.12](../008-abi/809-lowering-control.md#8912-影子栈帧)）。

`slots` 存放的不是 GC 引用值，而是**存放 GC 引用的字的地址**。这样的字是局部变量的栈槽或溢写槽；数组元素或 `struct` 字段中的字也以同样的方式指向。因此给变量赋值不需要另外更新帧。

### 编译器生成规则

只有持有 GC 引用并且可能到达安全点（[8.4.2](../008-abi/804-safepoint.md#842-生成位置)）的函数才需要帧。对永不返回的函数（panic）的调用，以及对既不分配也不阻塞的运行时函数（[8.5.2](../008-abi/805-extern-symbols.md#不是安全点的函数)）的调用不算在内。对这样的函数，编译器直接生成以下代码：

只有跨越安全点仍然存活的局部变量才需要根。

1. 进入时，把 GC 引用字清零。
2. 在机器栈上预留一个 `ShadowFrame`。在函数每条路径上的第一个安全点之前，把这些字的地址记录到 `slots` 中，把当前的 `nabi_tls_shadow_stack_top` 保存到 `prev`，再把链头设为该帧（链接）。链接可以更早进行，最早到函数入口；在路径汇合处，要么所有路径都已链接，要么都没有。不到达任何安全点的路径无需链接。
3. 在链接了该帧的路径上，紧接在每个 `return` 之前，把链头恢复为 `prev`（解除链接）。错误返回也是普通的 `return`，因此走同一条路径。panic 会终止进程，因此无需解除链接。
4. 仅跨越回边轮询（[8.4.8](../008-abi/804-safepoint.md#848-循环回边轮询)）、不跨越其他任何安全点而存活的局部变量，可以改为只在轮询的慢速路径上注册：放在第二个帧中，慢速路径在链接该帧并调用运行时之前写入其字（局部变量已死的地方写 null），并在调用之后立即解除链接。这样的字不在其他任何地方写入。规则 2 中在该路径上尚未链接的帧随它一起链接和解除链接。这依赖于一条规则：线程只会在编译器计入的安全点（调用、分配、入口轮询）或回边轮询处为回收而停下。因此，不是安全点的运行时函数（[8.5.2](../008-abi/805-extern-symbols.md#不是安全点的函数)）永远不得等待停止：它既不分配，也不阻塞，也不轮询。

当某个变量（根据活跃性）不再使用时，其根槽指向的字可以清为 null。只要该字不再被读取，这不会改变语义，并且能防止帧让死对象保持存活。

标记时，GC 从每个线程的链头开始，遍历各帧，把每个 `slots[i]` 指向的字的值作为根标记。

### 根指向的对象不会被移动

根（影子栈、`static`、已注册的线程根）直接指向的对象在该次回收中不会被移动。因此，传给阻塞的运行时调用或 `extern "C"` 函数的 GC 指针，**只要该实参在调用结束前一直位于根中**，就会保持原位。如果 C 代码在调用结束后保留该指针，就必须用 `nabi_gc_pin` 固定它（[8.5.1](../008-abi/805-extern-symbols.md#851-gc-区域)）。

协程帧是 GC 堆对象（[7.8](708-coroutine.md)），因此其中的变量通过帧对象的扫描表跟踪，而不是通过影子栈。

### 与回收器的约定

根（本节的影子栈）与写屏障（[7.4.2](#742-写屏障对象记忆)）按此处的规定生成。回收器对编译器输出没有其他要求。

### 与错误传播共存

[错误控制流](../003-expressions/314-error-control-flow.md)中的错误通过抛出函数的返回值传递（[8.6.5](../008-abi/806-calling-convention.md#865-抛出函数的返回基于值的错误传播)）。错误传播时，每个帧仍然通过普通的返回路径退出，并弹出自己的影子栈条目，因此影子栈始终与实际调用栈一致，无需为错误处理做任何特殊调整。

## 7.4.2. 写屏障：对象记忆

### 为什么需要写屏障

minor 回收不进入老对象（[7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-分代回收)）。当一个老对象开始指向一个年轻对象时，写屏障会通知回收器。否则那个年轻对象会被回收。

### 对象记忆（object-remembering）

屏障**只检查目标对象 `p` 的头**。存活下来的（老）对象在设置 `STICKY` 的同时设置了 `UNLOGGED` 位（[8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags偏移-4u32)）；对该对象的第一次写入时，整个对象被记录到记忆集中。

```
*p = q;
                                // ↓ barrier (may be before the store)
if (p.gc_flags & UNLOGGED) {
    nabi_gc_wb_slow(p, &p.f, len);  // clear UNLOGGED and put p in the remembered set
}
```

- 存入的值 `q` 的头（`q.sticky` 等）**不会被读取**。`q` 可能是 `any` 槽中的内联整数，也可能是 `str` 的内部指针，因此把它当作对象头读取是不安全的。
- 屏障可以放在存储之前。
- 慢速路径 `nabi_gc_wb_slow(obj, slot, len)` 接收对象、将要写入的槽的地址以及存入值的大小。它用原子 `fetch_and` 清除 `UNLOGGED`。在并发标记期间，它还会把该对象当前的引用记录到 SATB 缓冲区（[7.5.3](705-concurrent-collection.md#753-satb-屏障)）；对于大对象，只记录 `slot..slot + len` 所跨越的卡中的引用。因此即使多个线程并发写入，也只有一个线程记录该对象。慢速路径不得是安全点（回收不得在其中开始）。
- 年轻代 GC 把记忆集中的对象当作根，标记它们指向的年轻对象。回收之后，已记录的对象会被重新设置 `UNLOGGED`。

### 卡标记

8 KiB 或更大的老对象按 512 字节的**卡**记录，而不是作为整个对象记录。这样的对象设置的是 `CARDED` 位（[8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags偏移-4u32)），而不是 `UNLOGGED`。

- 这样的对象有一张卡表（每 512 B 1 字节），分配在对象之后。指向该卡表的指针位于紧挨头之前的字中。
- 内联的快速路径一次性检查 `p` 头中的 `UNLOGGED | CARDED`（一次加载，一次分支）。
- 顺序：如果有 `UNLOGGED`，先调用慢速路径；然后，如果有 `CARDED`，标记卡。在并发标记（[7.5](705-concurrent-collection.md)）期间，老的 LOS 对象可能同时有这两个位。
- 如果有 `CARDED`，屏障以内联方式、不经调用地把被写入的槽所跨越的卡（即从 `(slot - p) >> 9` 到 `(slot + len - 1 - p) >> 9`）标记为脏。因此屏障接收槽地址和存入值的大小（`len`）。
- 如果有 `UNLOGGED`，它如上所述调用 `nabi_gc_wb_slow(p, slot, len)`（不变）。
- minor 回收只扫描 `CARDED` 对象的脏卡，并清理这些卡。

不需要屏障的存储：

| 存储                                | 原因                                                |
|-------------------------------------|-----------------------------------------------------|
| 初始化刚分配的对象                  | 分配与存储之间没有 GC，因此仍是年轻对象             |
| 写入 `static`                       | 静态变量是根                                        |
| 原始 `Ptr<T>` 内存                  | 不受 GC 跟踪                                        |
| 原子操作（非引用值）                | 不是 GC 引用                                        |

### 记忆集

每个线程有一个 512 字的日志块（64 位目标上为 4 KiB，32 位上为 2 KiB）（TLS 槽 `nabi_tls_wb_log_cursor` / `nabi_tls_wb_log_limit`）。写满的块，以及在回收时或线程退出时的所有块，会被压入一个无锁栈。回收器排空这个栈，并把记录的对象用作根。

### 内联开销

快速路径是对 `p` 头的一次加载和一次分支（`UNLOGGED | CARDED`）；通常两个位都未设置（年轻对象或已记录的对象），因此直接通过。

### 编译器生成

编译器以内联方式生成快速路径；只有慢速路径调用运行时函数（[8.5](../008-abi/805-extern-symbols.md)）。

```
extern "C" {
    fn nabi_gc_wb_slow(obj: Ptr<u8>, slot: ConstPtr<u8>, len: usize);   // clear UNLOGGED and add to the remembered set (slow path, not a safepoint)
}
```

## 7.4.3. 安全点轮询

回收在所有 mutator 线程都停在安全点时进行。编译器在每个函数和循环中生成安全点轮询（8.4.2 中的例外除外）（[8.4](../008-abi/804-safepoint.md)）。

```
poll:
    if tls.yield_flag {
        call nabi_runtime_yield()       // signal arrival at a GC-safe point
    }
```

`nabi_tls_yield_flag` 通常为 0。当回收器设置它时，每个线程在其下一次轮询时停下。

## 7.4.4. 遍历影子栈

GC 开始标记时执行以下操作：

```
for each thread T:
    frame = T.nabi_tls_shadow_stack_top
    while frame != null:
        for i in 0..frame.count:
            ref = *frame.slots[i]          // slot is the address of a word
            if ref != null:
                mark(ref)
        frame = frame.prev
```

它遍历每个线程的影子栈，把每个存活的 GC 对象变量加入标记队列。

之后，标记算法（[GC 算法](703-gc-algorithm.md)）从这些对象开始，跟随其子对象。

## 7.4.5. 关键 TLS 槽

每个线程都有以下 TLS 槽。这些槽是运行时导出的每线程块 `nabi_tls` 中固定偏移处的字段（[8.3.1](../008-abi/803-tls-slots.md#831-tls-槽结构)）。

| 槽                          | 含义                                                                                      |
|-----------------------------|-------------------------------------------------------------------------------------------|
| `nabi_tls_yield_flag`        | 是否在安全点 `yield`                                                                      |
| `nabi_tls_tlab_cursor`       | [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 中的下一个分配位置 |
| `nabi_tls_tlab_limit`        | TLAB 的末端                                                                               |
| `nabi_tls_shadow_stack_top`  | 影子栈链头（最内层帧）                                                                    |
| `nabi_tls_wb_log_cursor`     | 写屏障日志中的位置                                                                        |
| `nabi_tls_mutator_state`     | 当前 mutator 状态（运行中 / 配合 GC）                                                     |

TLS 槽的详细定义见 [TLS 槽](../008-abi/803-tls-slots.md)。
