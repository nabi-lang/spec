# 1. bare 层级

bare 层级（`bare`）定义标准库中底层或危险的项。裸指针、transmute、裸分配、GC 固定和 FFI 原语等在安全抽象之外工作的工具都集中在这里。

## 依赖

向上，它只依赖运行时通过运行时 ABI `extern "nabi"` 暴露的符号。向下，`core`、`std` 和用户代码依赖它，但只能通过显式导入。

## 用户导入策略

```nabi
use bare::ptr::CPtr;                             // explicit import required (danger signal)
use bare::alloc::{alloc, Layout};

let layout = try! Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // null on failure
```

隐式应用的 prelude 不包含不安全的项。因此 `bare::*` 需要单独显式导入，这样用户进入危险区域的位置在代码中一目了然。

## 模块目录

### 指针家族（4 种）

- [1.1. `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>` / `CPtr<T>`：Zig 式数量编码](101-ptr.md)
- [1.2. 强制转换操作：`ptr_cast`、`align_cast`、`const_cast`、`reinterpret`](102-cast.md)

### 内存 / 转换

- [1.3. `MaybeUninit<T>` / `transmute_unchecked`：位级转换](103-mem.md)
- [1.4. 裸分配（`alloc`、`dealloc`、`realloc`）](104-alloc.md)
- [1.5. volatile 内存访问：`volatile::read` / `volatile::write`](105-volatile.md)

### GC 集成

- [1.6. `Pinned<T>` / `gc::pin`：阻止 GC 移动对象](106-gc-pin.md)

### FFI

- [1.7. extern "C" / `CStr` / `CString` / `CPtr`：C ABI 兼容；C 类型、`Handle`、errno、宽字符串、`Library`](107-ffi.md)
- [1.8. WASM 组件模型接口（`ffi::wit`）](108-wit.md)
- [1.10. Objective-C（`extern "objc"`、`ffi::objc`）](110-objc.md)

没有 1.9 节。这个编号空缺，Objective-C 保留 1.10，以免指向它的链接失效。

## 核心设计原则

### 1. Strict provenance：采用 Rust 2024 模型

`Ptr<T>` 是（地址，来源（provenance））对，而不是普通整数。因此 `usize ↔ Ptr<T>` 往返会丢失来源，只能通过显式的逃生口进行。

```nabi
let p: Ptr<i32> = ...;
let addr: usize = p.addr();              // safe: address only
let q: Ptr<i32> = p.with_addr(addr + 4); // safe: provenance kept, only the address changes
let r: Ptr<i32> = unsafe { bare::ptr::from_exposed_provenance(addr) };  // unsafe: fresh provenance
```

`as` 强制转换永远不能用于指针。这一方向的依据是 [Ralf Jung “Pointers Are Complicated”](https://www.ralfj.de/blog/2022/04/11/provenance-exposed.html)，它与 Rust 1.84（2024 年 12 月）稳定的模型同构。

### 2. 数量编码在类型中：Zig 模型

| 类型          | 含义                                     | C 对应          |
|---------------|------------------------------------------|-----------------|
| `Ptr<T>`      | 恰好**一个** T                           | `T*`（单个）    |
| `ConstPtr<T>` | 恰好**一个** const T                     | `const T*`      |
| `ManyPtr<T>`  | **N 个连续的** T（长度未知，兼容 C）     | `T*`（数组）    |
| `Slice<T>`    | **N 个连续的** T（包含长度）             | (ptr, len)      |
| `CPtr<T>`     | 兼容 C 的**可为 null** 的指针            | `T*`（可为 null） |

`Slice<T>` 是 [2.9 集合类型](../200-core/209-collections.md)中定义的安全视图，不属于 bare 层级。本章只介绍 `Slice` 与指针之间的裸路径：`ManyPtr::slice`，以及固定列表的数据指针。

### 3. GC 与裸指针：分开的两个类型层次

| 类型       | 含义                                       | GC 移动对象时                |
|------------|--------------------------------------------|------------------------------|
| `GcPtr<T>` | 指向 GC 堆上对象的引用（通常写作 `T`）     | **自动更新**                 |
| `Ptr<T>`   | 裸指针（非托管或已固定）                   | **失效**                     |

这一区分沿用 Go 1.21 的 `runtime.Pinner` 模型。裸 `Ptr<T>` 只在 GC 固定期间有效，因此在固定解除后使用它是未定义行为（UB）。

```nabi
let obj: List<i32> = [1, 2, 3];                // GcPtr (automatic)
using pin = bare::gc::pin(obj) {          // pin begins
    let raw: Ptr<i32> = pin.as_data_ptr();
    ffi_function(raw, obj.len());
}                                              // pin released, raw invalid
```

### 4. 强制转换只能是显式命名的操作

C 和 Rust 中对指针做 `as` 强制转换的陷阱是最大的危险。因此 Nabi 只以显式命名的函数暴露每一种不安全的强制转换。

| C/Rust 写法       | Nabi 写法                                | 含义                           |
|-------------------|------------------------------------------|--------------------------------|
| `ptr as *U`       | `bare::ptr_cast<U>(p)`               | 只改变类型（保留来源）         |
| `ptr as *Aligned` | `bare::align_cast(p)`                | 对齐向上转换（未对齐时为 `none`） |
| `*const → *mut`   | `bare::const_cast(p)`                | 去掉 const                     |
| `transmute<A,B>`  | `bare::reinterpret<A:Pod, B:Pod>(a)` | 位重新解释（强制 Pod）         |

### 5. 用 `using` 块实现 RAII：不依赖终结器

每一种 GC 语言都已经认识到，在 GC 环境中终结器（finalizer）并不可靠。因此文件、套接字、互斥锁等外部资源一律确定性地释放，绝不交给 GC：`Drop` 类型是仿射的，编译器在它最后一个所有者离开作用域的位置（每一条退出路径上）或在它的 `using` 块结束时释放它，关闭可能失败的资源必须显式关闭。GC 从不调用 `drop`，也没有终结器（[stdlib 2.13.4](../200-core/213-marker.md#2134-drop释放时的清理)）。

```nabi
use bare::ptr::{ConstPtr, CPtr};

extern "C" {
    fn fopen(path: ConstPtr<c_char>, mode: ConstPtr<c_char>) -> CPtr<u8>;
    fn fclose(f: CPtr<u8>) -> i32;
}

// a user-written wrapper: Drop closes the C handle
struct File { raw: CPtr<u8>; }

impl Drop for File {
    fn drop(ref self) { unsafe { fclose(self.raw) }; }
}

using f = File { raw: opened } {   // opened: non-null CPtr<u8> from fopen
    // ... use f.raw ...
}     // ← _guaranteed_ fclose at end of scope (RAII via Drop)

// ordinary GC objects need no using; the GC reclaims them
let xs: List<i32> = [1, 2, 3];     // GC automatic
```

依据是 [2.13 marker](../200-core/213-marker.md) 中的 `Drop` trait；`using` 块的语法依据是 [refs-lang 3.17 using 表达式](../../refs-lang/003-expressions/317-using-expression.md)。
