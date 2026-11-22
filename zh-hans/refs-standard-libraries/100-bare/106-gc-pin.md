# 1.6. `Pinned<T>`：GC 固定

`bare::gc` 模块定义一种机制，把 GC 堆上的对象暂时固定，使它不能移动。当裸 `Ptr<T>` 指向 GC 对象而对象发生移动时，指针就会失效；本章介绍如何解决这一冲突。

这是 Nabi 不安全层级中最重要的一章：它正面说明指针与 GC 之间的冲突如何调和。

## 1.6.1. 动机：为什么需要固定

[refs-lang 7.3 GC 算法](../../refs-lang/007-runtime/703-gc-algorithm.md)中的 Immix 标记-区域 GC 会进行**疏散**（evacuation，通过复制来移动对象）。因此对象的地址在两次 GC 周期之间可以自由改变。

```
  GC cycle start:    obj at 0x1000
  GC evacuation:     obj at 0x2000  (copied, then forwarded)
  GC cycle end:      obj at 0x2000
```

对托管代码（`GcPtr<T>`）而言，GC 追踪根并自动更新每一个指针。但裸 `Ptr<T>` 对 GC 不可见，所以不会被更新，成为**过期指针**（stale ptr）。

```nabi
use bare::gc;
use bare::ptr::Ptr;

let xs: List<i32> = [1, 2, 3];                 // GcPtr (updated automatically)
let raw: Ptr<i32> = Ptr::dangling();
using pin = gc::pin(xs) {
    raw = pin.as_data_ptr();                    // raw ptr, valid inside the pin
}                                               // unpinned here
gc::collect();                                  // xs may move
unsafe { raw.read() };                          // UB: raw is stale
```

解决办法是**固定**（pinning）：显式告诉 GC“不要移动这个对象”。

## 1.6.2. `Pinned<T>` 守卫

```nabi
struct Pinned<T> {
    obj: T;
    // internal: registered in the GC's pin set
}

#[sealed]
trait Pinnable {
    // the GC object holding the value's data; none when nothing can move
    #[internal("write `pin(x)` to keep a value in place")]
    fn gc_object(self) -> GcBuf<u8> | none;
}
impl<T> Pinnable for List<T> { /* the list's buffer; none when empty */ }
impl<T> Pinnable for Slice<T> { /* the buffer it views; none when empty */ }
impl<T, const N: usize> Pinnable for Array<T, N> { /* the element buffer; none when empty (2.9.14) */ }
impl Pinnable for str { /* none (2.15.2) */ }
impl Pinnable for StrBuf { /* the byte buffer; none while it has no room (2.15.2) */ }
impl<T> Pinnable for GcPtr<T> { /* the cell */ }

fn pin<T: Pinnable>(obj: T) -> Pinned<T>;
fn collect();                           // a full collection now; may move every unpinned object
fn stats() -> GcStats;                  // the collector's counters (1.6.10)
fn collect_young();                     // a minor collection now (1.6.11)
fn collect_young_within(ms: f64) -> bool;
fn set_pause_goal(ms: f64);             // 1.6.12
fn no_collect<R>(f: fn() -> R) -> R;    // 1.6.13
fn set_heap_limit(bytes: u64);          // 1.6.14
struct Arena<T>;                        // 1.6.15

impl<T> Pinned<T> {
    fn obj(self) -> T;
}

impl<T> Drop for Pinned<T> {
    fn drop(ref self) {
        // removed from the GC pin set
    }
}
```

它是一个守卫。只要 `pin()` 返回的 `Pinned<T>` 还活着，GC 就不会移动该对象。`Pinned<T>` 是 `Drop` 值，只释放一次（[2.13.4](../200-core/213-marker.md#2134-drop释放时的清理)）：在守卫的所有者离开作用域的位置，或在绑定它的 `using` 块结束时，`Pinned::drop` 运行并解除对象的固定。固定是计数的：只有经过同样次数的解除固定后，对象才会再次移动。`pin` 固定 `gc_object` 所指出的对象，`gc_object` 返回 `none` 时什么也不固定。`GcBuf<u8>` 是 [1.3.9](103-mem.md#139-gcbuft-与-buf_-原语) 中擦除了类型的 GC 缓冲区。`Pinnable` 是封闭的（sealed），`gc_object` 是工具链内部的（[6.2.16](../../refs-lang/006-meta-system/602-built-in-attributes.md#6216-internal-shared-by-the-toolchains-packages)）：程序用 `pin(x)` 固定，不调用 `gc_object`。

### 示例：FFI 调用

```nabi
let buf: List<u8> = read_input();

using pin = bare::gc::pin(buf) {
    let raw: Ptr<u8> = pin.as_data_ptr();
    unsafe {
        ffi::process(raw, buf.len());          // FFI uses buf's data
    }
}     // ← automatic unpin at end of scope
```

这里的 `using` 块遵循 [refs-lang 3.17 using 表达式](../../refs-lang/003-expressions/317-using-expression.md)定义的 RAII 形式。

## 1.6.3. `GcPtr<T>` 与 `Ptr<T>`：类型层面的分离

```nabi
struct GcPtr<T> { /* compiler-managed; roots traced automatically */ }
struct Ptr<T>   { /* raw; no root tracking */ }
```

| 方面       | `GcPtr<T>`                    | `Ptr<T>`                                         |
|------------|-------------------------------|--------------------------------------------------|
| GC 根      | **自动追踪**                  | **不追踪**                                       |
| 移动时     | **自动更新**                  | **过期**                                         |
| 使用场合   | 普通代码（`List`/`Dict`/...） | 仅限 unsafe + FFI                                |
| 创建方式   | GC 分配（`List::new()` 等）   | `gc::pin().as_data_ptr()` 或 `alloc::alloc(layout)`   |
| 有效期     | 只要对象可达                  | 仅在固定守卫的作用域内                           |

由于类型系统把两者分开，用户代码通常只与 `GcPtr` 打交道（它被不可见地包装在 `List<T>` 等类型中）。只有进入 unsafe 之后，`Ptr<T>` 才会显式出现。

### 操作

```nabi
impl<T> GcPtr<T> {
    fn new(value: T) -> GcPtr<T>;           // a new GC cell holding value
    fn get(self) -> T;
    fn set(self, value: T);                 // every copy of the GcPtr sees it
    fn same(self, other: GcPtr<T>) -> bool; // identity: the same cell
}
impl<T> !Sync for GcPtr<T> {}
impl<T> Pinnable for GcPtr<T> {}
```

### 内部实现：GcPtr 的形态

```nabi
struct GcPtr<T> {
    // internal: usually a plain `*T`; the GC traces it externally via the root set
    // for a local variable, kept in a shadow stack frame slot emitted by the compiler
}
```

Nabi 的 GC 通过编译器直接生成的**每线程影子栈**（shadow stack）追踪根集合（[refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)）。

## 1.6.4. pin / unpin 在 GC 一侧的机制

```
GC pin set structure.
  - hash table: Ptr<Object> → pin count
  - during evacuation the GC _skips_ objects with pin count > 0

pin(obj):
  1. check the pin bit in obj's header (or register in the pin set)
  2. count++
  3. return Pinned<T>

drop(Pinned):
  1. count--
  2. if count == 0, clear the pin bit
```

这一结构实际上与 Go 1.21 的 `runtime.Pinner` 同构。详情见 [pkg.go.dev/runtime#Pinner](https://pkg.go.dev/runtime#Pinner)。

### 开销

开销如下：

- 一次固定只是一次 O(1) 的哈希插入
- 一次解除固定同样是一次 O(1) 的哈希删除
- 在 GC 周期内会增加一次固定集合检查，但开销很小

在大多数情况下这并不在热路径上，因为 FFI 调用本身没有那么频繁。

## 1.6.5. 何时需要固定

每一次 `extern "C"` 调用都会让调用线程进入停驻（park）状态（[1.7.1](107-ffi.md#171-extern-c-块)）：C 运行期间，回收器不会等待该线程，所以在任何 C 调用期间，无论多短，都可能发生一次回收并移动对象。C 在调用期间使用的、指向 GC 对象内部的指针，只在以下情况下有效：

- 对象在整个调用期间都被固定（`gc::pin`，1.6.2），并且在调用之后，只要 C 还保留着该指针，就一直被固定；或者
- 指针指向的正是某个根所指出的对象本身：`str` 实参，调用者的帧会把它作为根保持到调用返回（[refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)），以及在 [2.15.2](../200-core/215-string.md#2152-str不可变的-utf-8-字符串) 规则下的 `str::as_ptr` / `CStr::as_ptr`。

其他所有指向 GC 对象内部的指针，即列表的缓冲区（`as_data_ptr`）、切片的元素、`StrBuf` 的字节、`GcPtr` 的值，都需要固定。无论是什么调用，编译器都不会自行省略固定：固定由程序来写。`#[leaf]`（[1.7.1](107-ffi.md#171-extern-c-块)）让调用更便宜，但不改变这条规则。C 在多次调用之间保留的值放在 `Handle`（[1.7.10](107-ffi.md#1710-handletc-持有的-nabi-值)）中，它会一直固定该值直到被释放。

不持有裸指针的代码不需要固定：

| 场合             | 原因                                    |
|------------------|-----------------------------------------|
| 普通方法调用     | GcPtr 自动更新；不会创建裸指针          |
| GcPtr 之间的操作 | 全部由 GC 追踪                          |
| 传给 `extern "C"` 函数的 `str` | 实参的根在调用期间让字符串保持原位 |

### 被固定的 `List` 或 `StrBuf` 不能增长

列表或 `StrBuf` 增长超过其容量时会替换缓冲区，而 C 可能持有旧缓冲区的地址。缓冲区被固定期间，增长（`reserve`，以及经由它的 `push`、`push_str`、`extend` 等）会改为触发 panic：

```text
cannot grow a list while it is pinned (C may hold its address): call `xs.reserve(n)` before `gc::pin(xs)`
cannot grow a `StrBuf` while it is pinned (C may hold its address): call `b.reserve(n)` before `gc::pin(b)`
```

固定之前预留的空间可以在固定期间填充。

```nabi
use bare::gc::pin;

let ys: List<i64> = [1];
ys.reserve(7);                     // room for 8
let py = pin(ys);
for i in 0..7 { ys.push(i); }      // fits: the buffer stays
ys.push(9);                        // panics: the buffer would have to grow
```

## 1.6.6. 提取方法

```nabi
impl<T> Pinned<List<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // start of the List's buffer
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl<T> Pinned<Slice<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // the slice's first element; dangling when empty
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl<T, const N: usize> Pinned<Array<T, N>> {
    fn as_data_ptr(self) -> Ptr<T>;          // the first element; steps are strides(), in elements
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl<T> Pinned<GcPtr<T>> {
    fn as_value_ptr(self) -> Ptr<T>;         // the cell's value
}

impl Pinned<str> {
    fn as_bytes_ptr(self) -> ConstPtr<u8>;   // UTF-8 bytes
    fn as_c_ptr(self) -> ConstPtr<c_char>;   // null termination not guaranteed; convert separately
}

impl Pinned<StrBuf> {
    fn as_bytes_ptr(self) -> Ptr<u8>;        // the first byte (no NUL promised); dangling while there is no room
}
```

这些容器方法就是提取 API，在持有 `Pinned` 期间有效。（`str::as_ptr` 和 `CStr::as_ptr` 在 [2.15.2](../200-core/215-string.md#2152-str不可变的-utf-8-字符串) 更窄的规则下无需固定就给出地址。）没有泛型的 `as_ptr`；`as_value_ptr` 的命名有意与 `List` 的 `as_data_ptr` 区分开。每种容器都有专门的提取方法，因此可以按各容器的布局取出数据，例如缓冲区起始地址或以 null 结尾的形式。

## 1.6.7. GC 运行时链接

各项与 GC 运行时调用的对应关系如下。

| 项               | 运行时调用                     |
|------------------|--------------------------------|
| `Pinned<T>`      | `nabi_gc_pin` / `nabi_gc_unpin`  |
| `pin<T>()`       | （同上）                       |
| `collect()`      | `nabi_gc_collect`               |
| `stats()`        | `nabi_gc_stats`（1.6.10）       |
| `collect_young()` / `collect_young_within(ms)` | `nabi_gc_collect_young` / `nabi_gc_collect_young_within`（1.6.11） |
| `set_pause_goal(ms)` | `nabi_gc_set_pause_goal`（1.6.12） |
| `no_collect(f)`  | `nabi_gc_no_collect`（1.6.13）  |
| `set_heap_limit(bytes)` | `nabi_gc_set_heap_limit`（1.6.14） |
| `Arena<T>`       | `nabi_gc_arena_epoch`（1.6.15） |
| `GcPtr<T>`       | 影子栈帧槽（由编译器生成）     |
| `as_data_ptr` 等 | 取决于各容器的布局             |
| `Weak<T>`        | `nabi_gc_weak_new` / `nabi_gc_weak_get`（1.6.9） |
| 被固定缓冲区的增长检查（1.6.5） | `nabi_gc_pinned` |

## 1.6.8. 与其他语言的比较

| 方面             | Java JNI                       | .NET fixed             | Go 1.21 Pinner           | Nabi                       |
|------------------|--------------------------------|------------------------|--------------------------|---------------------------|
| 固定 API         | `Get*Critical`                 | `fixed (T* p = ...)`   | `runtime.Pinner`         | `bare::gc::pin(obj)`  |
| RAII 守卫        | 显式 `Release`                 | 块作用域               | `defer pin.Unpin()`      | `using` 块（自动 drop）   |
| 类型分离         | **无**（全是 `jobject`）       | **无**                 | **无**（所有指针一样）   | `GcPtr<T>` 与 `Ptr<T>`   |
| 固定后的失效检查 | **无**                         | 编译器（块作用域）     | **无**（手动）           | 类型系统 + 作用域         |
| 多重固定         | 可重叠                         | **受限**               | 可重叠                   | 可重叠（计数）            |

Nabi 与其他语言主要在两点上不同。

1. 由类型系统强制。`GcPtr<T>` 和 `Ptr<T>` 是不同的类型，拒绝自动转换；唯一的跨越途径是固定守卫。
2. 释放基于 `using` 块。`Pinned<T>` 在离开 `using` 块时被 drop，所以永远不会漏掉解除固定。这比 Go 手动 `defer` 的方式更安全。

> 这一策略是 Nabi 指针模型的骨架：unsafe 和 GC 这两个领域在这里安全地交汇。

## 1.6.9. `Weak<T>`：弱引用

```nabi
// bare::gc
struct Weak<T> { /* opaque */ }

impl<T> Weak<T> {
    fn new(target: GcPtr<T>) -> Weak<T>;
    fn get(self) -> GcPtr<T> | none;        // none once the target is freed
}
impl<T> !Sync for Weak<T> {}
impl<T> !Send for Weak<T> {}    // a hand-off cannot copy what it names (2.13.9)
```

弱引用不会让它的对象保持存活。一旦某次回收释放了对象，指向它的每一个 `Weak` 都读出 `none`；移动对象的回收会更新 `Weak`。这是 `WEAK_REF` 扫描类别的库接口（[refs-lang 8.2.8](../../refs-lang/008-abi/802-scan-table.md#828-weak_ref)）。

在运行时，`Weak` 是类型 id 为 `3` 的 GC 单元（[refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-runtime-reserved-type-ids)），持有一个 `WEAK_REF` 字，由 `nabi_gc_weak_new` 创建，由 `nabi_gc_weak_get` 读取（[refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-auxiliary-symbols-appendix-non-normative)）。

## 1.6.10. `stats`：回收器计数器

```nabi
// bare::gc
struct GcStats {
    pub collections: u64;      // every kind
    pub minor: u64;            // minor collections
    pub full: u64;             // full collections that stopped the world throughout
    pub concurrent: u64;       // full collections marked concurrently (7.5)
    pub pause_total_ns: u64;   // time the program stood stopped, all pauses together
    pub pause_max_ns: u64;     // the longest pause
    pub live_bytes: u64;       // alive at the last collection, plus allocated since
    pub heap_bytes: u64;       // held from the operating system (blocks and large objects)
    pub moved_bytes: u64;      // moved by evacuation
    pub assists: u64;          // times an allocating thread marked for a concurrent mark (7.5.4)
    pub grown_bytes: u64;      // bytes no_collect regions grew the heap by (1.6.13)
    pub limit_collections: u64; // collections the heap limit brought forward (1.6.14)
    pub young_budget: u64;     // bytes allocated between two minor collections now
}

fn stats() -> GcStats;
```

`stats()` 读取程序启动以来的计数器，用于指标或帧预算。它会短暂获取回收器的锁，所以适合偶尔读取（一次导出、每帧一次），而不是在热循环中读取。停顿从发出停止请求时开始计算，所以包含到达安全点的时间（[refs-lang 9.9.8](../../refs-lang/009-tools/909-debug-tools.md#998-memory--gc-tools)）。

`collect()` 在整个完整回收期间停止所有线程，比运行时自行发起的回收（一次次要回收，或一次并发标记的完整回收）更长。要让停顿保持短暂，请把回收交给运行时，或在帧与帧之间调用 `collect_young`（1.6.11）。

## 1.6.11. `collect_young`：按需进行次要回收

```nabi
// bare::gc
fn collect_young();
fn collect_young_within(ms: f64) -> bool;
```

`collect_young()` 立即运行一次次要回收（minor collection）：它只检查上次回收以来分配的对象，所以开销大约等于其中存活下来的部分，而不是堆的大小。两帧之间或两个请求之间是回收的廉价时机。并发标记运行期间（[refs-lang 7.5](../../refs-lang/007-runtime/705-concurrent-collection.md)），它什么也不做：标记会自行完成，不会产生长停顿。

`collect_young_within(ms)` 只在最近几次次要回收表明它最多耗时 `ms` 毫秒（帧剩余的时间）时才运行，并返回它是否运行了。估算值是最近次要回收每个年轻字节的开销乘以当前的年轻字节数，再加上一个固定部分；尚未观察到任何次要回收时，它会运行。

## 1.6.12. `set_pause_goal`：停顿目标

```nabi
// bare::gc
fn set_pause_goal(ms: f64);   // 0: no goal (the default)
```

设定目标后，回收器会切分自己的工作，使每次停顿接近 `ms` 毫秒：

- 年轻代预算跟随目标：取最近次要回收表明能放进目标的量的四分之三，范围从 1 MiB 到默认预算（[refs-lang 7.2.8](../../refs-lang/007-runtime/702-allocation.md#728-gc-trigger)）。`GcStats::young_budget` 显示当前值。
- 每一次完整回收都并发标记，无论堆有多大，堆也永远不会自行疏散：只有 `collect()` 会压缩它。
- 并发标记之后的清除以及标记协助本来就在停顿之外运行（[refs-lang 7.5.2](../../refs-lang/007-runtime/705-concurrent-collection.md#752-phases)）。

这是一个目标，而不是承诺：停顿仍可能更长，例如在存活堆快速增长时，或在一次分配量很大的并发标记刚结束之后。

## 1.6.13. `no_collect`：不发生回收的区域

```nabi
// bare::gc
fn no_collect<R>(f: fn() -> R) -> R;
```

`no_collect(f)` 调用 `f`，在它运行期间，任何线程上的分配都不会发起回收：没有次要回收，没有完整回收，也没有压力测试回收（`NABIR_GC_STRESS`）。区域在 `f` 返回的位置结束，无论它以何种方式离开。区域可以嵌套，也可以同时在多个线程上打开；最后一个区域关闭时，回收才恢复。

- 区域内年轻代预算耗尽时，堆会改为增长，每次至少 1 MiB；`GcStats::grown_bytes` 对此计数。回收发生在区域之后，即下一次需要回收的分配时。
- 正在运行的并发标记在区域期间继续标记，并在区域之后完成（它的最终停顿）。
- 堆上限（1.6.14）仍然有效：会超过上限的分配会触发回收，在区域内也是如此。
- 在区域内调用的 `collect()` 和 `collect_young()` 仍然会回收。

## 1.6.14. `set_heap_limit`：堆上限

```nabi
// bare::gc
fn set_heap_limit(bytes: u64);   // 0: no limit (the default)
```

每个周期的预算以 `bytes` 为上限：堆越接近上限，回收来得越早，因上限而提前的回收是一次完整回收（计入 `GcStats::limit_collections`）。上限计算的是 GC 对象的字节数，包括已分发的年轻字节，而不是进程的全部内存。存活数据超过上限并不是错误：此时上限变为存活字节数再加八分之一（至少 4 MiB），所以堆会一点一点地超过上限增长，每一步都进行完整回收。会超过上限的并发标记会在一次停顿中完成。

## 1.6.15. `Arena<T>`：一帧的值

```nabi
// bare::gc
struct Arena<T> { /* shared by copies */ }
struct ArenaRef<T> { /* plain data: an index and a generation */ }
impl<T> Copy for ArenaRef<T> {}

impl<T> Arena<T> {
    fn new() -> Arena<T>;
    fn with_capacity(n: usize) -> Arena<T>;
    fn alloc(self, value: T) -> ArenaRef<T>;   // append in place
    fn get(self, r: ArenaRef<T>) -> T;         // panics for a stale reference
    fn try_get(self, r: ArenaRef<T>) -> T | none;
    fn set(self, r: ArenaRef<T>, value: T);    // panics for a stale reference
    fn holds(self, r: ArenaRef<T>) -> bool;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;
    fn reset(self);                            // empty at once; every ArenaRef so far is stale
    fn values(self) -> List<T>;                // a new list, in allocation order
}
impl<T> !Sync for Arena<T> {}
```

`Arena<T>` 把一帧（或一个请求）的值放在一个缓冲区中：`alloc` 原地追加一个值，所以回收器既不为每个值分配对象，也不追踪每个值的对象；`reset` 一次清空 arena，同时保留缓冲区供下一帧使用。`Arena` 的副本共享它。

内存安全依赖两条规则：

- 缓冲区是普通的 GC 对象。值在 arena 中期间，它们引用的东西保持存活，arena 中的任何东西都不会悬垂。（`reset` 之后，持有 GC 引用的类型的值会被清除；不持有 GC 引用的类型的值会被后续的 `alloc` 覆盖。）
- `ArenaRef` 是普通数据，即一个索引和一个代号（generation）。每次 `reset` 都取一个本程序中任何 arena 都未用过的代号（[refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-auxiliary-symbols-appendix-non-normative)），所以 reset 之前的引用或来自另一个 arena 的引用无法通过 `get` 和 `set` 中的检查，并触发 panic（`try_get` 给出 `none`，`holds` 返回 `false`）。它可以保存在任何地方，包括 GC 对象中：过晚使用它是一个有定义的 panic，绝不会读到下一帧的值。

因此 arena 不需要生命周期、不需要带标记的作用域，也不需要限制 `T` 可以持有什么；代价是每次访问一次整数比较。

```nabi
use bare::gc;
use bare::gc::Arena;

struct Particle { x: f64; y: f64; }

let particles: Arena<Particle> = Arena::with_capacity(4096);
loop {
    let p = particles.alloc(Particle { x: 0.0, y: 0.0 });
    update(particles, p);
    draw(particles);
    particles.reset();                         // the frame's particles are gone
    gc::collect_young_within(frame_left_ms());
}
```
