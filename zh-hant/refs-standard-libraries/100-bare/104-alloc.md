# 1.4. 原始配置：`alloc` / `dealloc` / `realloc`

`bare::alloc` 模組提供 API，讓你直接在 GC 管控之外的區域配置和釋放記憶體。一般的 Nabi 程式碼由 GC 管理，但在 GC 不適合的地方會使用這個模組：跨越 FFI 邊界把記憶體交給 C 端、大型緩衝區、mmap 等。

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

`alloc` 失敗或大小為 0 時，回傳 `CPtr::null()`；它不會中止，因此呼叫端必須檢查並處理結果。`n` 個元素使 `usize` 溢位時，`alloc_array` 也回傳 `CPtr::null()`。`Layout` 的欄位是私有的：請用 `size()` / `align()` 讀取。`n` 個元素使 `usize` 溢位時 `array` 擲回 `LayoutError`，`align` 不是 2 的冪時 `from_size_align` 擲回 `LayoutError`；`message()` 說明是哪一種。

## 1.4.2. 範例

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

使用 `using` 區塊時，清理採用 RAII 形式。

```nabi
using buf = try bare::alloc::Allocation::<u32>::new(1000) {
    // buf.as_ptr() is usable
}     // ← automatic dealloc
```

這裡的 `Allocation<T>` 是本模組內含的 RAII 包裝器。

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

大小使 `usize` 溢位或系統沒有記憶體時，`new` 擲回 `LayoutError`（`message()` 為 `"out of memory"`）。`Allocation` 實作 `Drop`，因此它是仿射的（[2.13.4](../200-core/213-marker.md#2134-drop釋放時的清理)）：`let b = a` 會移動它，而它的區塊在最後一個擁有者釋放它的位置（`using` 區塊結束或擁有者離開作用域）被釋放。

## 1.4.3. 與 GC 的關係

| 區域                     | 配置 / 釋放                 | 是否由 GC 管理？   |
|--------------------------|-----------------------------|--------------------|
| 一般物件（`List<i32>`） | GC TLAB bump          | ✓                  |
| `bare::alloc::alloc` | 系統 malloc               | ✗（手動）         |
| FFI `malloc`             | C `malloc`                  | ✗                  |
| 大型物件（≥8KB）      | LOS（不移動的 mark-sweep） | ✓（但不移動） |

如果需要由 GC 管理的原始緩衝區，請改用 `bare::mem::GcBuf<T>`（[2.9.12](../200-core/209-collections.md#2912-gcbuft原始-gc-緩衝區)），而不是這個模組。

GC 之外的區域不會被 GC 追蹤，因此清理是使用者的責任。不過，由於那裡的物件不會移動，`Ptr<T>` 不需要另外的固定守衛就能保持有效。

## 1.4.4. 執行環境連結

`bare::alloc::alloc` 的主體降階為 Nabi 執行環境在系統 malloc 之上提供的包裝器，最終呼叫 `libc::malloc` / `libc::free` / `libc::realloc`。
