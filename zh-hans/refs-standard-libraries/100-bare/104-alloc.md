# 1.4. 裸分配：`alloc` / `dealloc` / `realloc`

`bare::alloc` 模块提供在 GC 管辖之外的区域直接分配和释放内存的 API。普通的 Nabi 代码由 GC 管理，但在 GC 不合适的地方会用到本模块：跨 FFI 边界把内存交给 C 一侧、大缓冲区、mmap 等。

## 1.4.1. API

```nabi
struct LayoutError { .. }                      // private fields; only the library builds one (0.5.10)
impl LayoutError {
    fn message(self) -> str;
}
// + Display, Debug, Error, PartialEq, Eq, Clone

struct Layout { /* private: size, align */ }

impl Layout {
    fn new<T>() -> Layout;                     // size_of<T>(), align_of<T>()
    fn array<T>(n: usize) throws (LayoutError) -> Layout;
    fn from_size_align(size: usize, align: usize) throws (LayoutError) -> Layout;
                                               // align must be a power of two
    fn size(self) -> usize;
    fn align(self) -> usize;
}

unsafe fn alloc(layout: Layout) -> CPtr<u8>;
unsafe fn alloc_zeroed(layout: Layout) -> CPtr<u8>;
unsafe fn dealloc(ptr: Ptr<u8>, layout: Layout);
unsafe fn realloc(ptr: Ptr<u8>, old_layout: Layout, new_size: usize) -> CPtr<u8>;

// convenience functions (typed)
unsafe fn alloc_one<T>() -> CPtr<T>;
unsafe fn alloc_array<T>(n: usize) -> CPtr<T>;
unsafe fn dealloc_one<T>(ptr: Ptr<T>);
unsafe fn dealloc_array<T>(ptr: Ptr<T>, n: usize);
```

`alloc` 失败或大小为 0 时返回 `CPtr::null()`；它不会中止，所以调用者必须检查并处理结果。`n` 个元素溢出 `usize` 时，`alloc_array` 也返回 `CPtr::null()`。`Layout` 的字段是私有的：用 `size()` / `align()` 读取。`n` 个元素溢出 `usize` 时 `array` 抛出 `LayoutError`，`align` 不是 2 的幂时 `from_size_align` 抛出 `LayoutError`；`message()` 说明是哪种情况。

## 1.4.2. 示例

```nabi
let layout = try Layout::array::<u32>(1000);
let p: CPtr<u8> = unsafe { bare::alloc::alloc(layout) };

match p.as_ptr() {
    raw: Ptr<u8> => {
        let typed: Ptr<u32> = bare::ptr_cast::<u32>(raw);
        // use
        unsafe {
            for i in 0u32..1000u32 { typed.add(i as usize).write(i); }
        }
        unsafe { bare::alloc::dealloc(raw, layout) };
    }
    none => panic("alloc failed"),
}
```

用 `using` 块时，清理采用 RAII 形式。

```nabi
using buf = try bare::alloc::Allocation::<u32>::new(1000) {
    // buf.as_ptr() is usable
}     // ← automatic dealloc
```

这里的 `Allocation<T>` 是本模块包含的 RAII 包装类型。

```nabi
struct Allocation<T> { /* private: pointer + length */ }

impl<T> Allocation<T> {
    fn new(n: usize) throws (LayoutError) -> Allocation<T>;   // n zeroed Ts
    fn as_ptr(self) -> Ptr<T>;
    fn as_many(self) -> ManyPtr<T>;
    fn len(self) -> usize;
}

impl<T> Drop for Allocation<T> { /* frees the block */ }
```

大小溢出 `usize` 或系统没有内存时，`new` 抛出 `LayoutError`（`message()` 为 `"out of memory"`）。`Allocation` 实现了 `Drop`，所以它是仿射的（[2.13.4](../200-core/213-marker.md#2134-drop---cleanup-on-release)）：`let b = a` 会移动它，它的内存块在最后一个所有者释放它的位置被释放（`using` 块结束，或所有者离开作用域）。

## 1.4.3. 与 GC 的关系

| 区域                     | 分配 / 释放                 | 由 GC 管理？       |
|--------------------------|-----------------------------|--------------------|
| 普通对象（`List<i32>`）  | GC TLAB 指针碰撞（bump）    | ✓                  |
| `bare::alloc::alloc` | 系统 malloc                 | ✗（手动）          |
| FFI `malloc`             | C `malloc`                  | ✗                  |
| 大对象（≥8KB）           | LOS（不移动的标记-清除）    | ✓（但不移动）      |

如果需要由 GC 管理的裸缓冲区，请使用 `bare::mem::GcBuf<T>`（[2.9.12](../200-core/209-collections.md#2912-gcbuft---raw-gc-buffer)），而不是本模块。

GC 之外的区域不被 GC 追踪，所以清理由用户负责。不过那里的对象不会移动，因此 `Ptr<T>` 无需单独的固定守卫也一直有效。

## 1.4.4. 运行时链接

`bare::alloc::alloc` 的主体降级为 Nabi 运行时在系统 malloc 之上提供的包装函数，最终调用 `libc::malloc` / `libc::free` / `libc::realloc`。
