# 1.2. 强制转换操作：显式命名的强制转换

指针之间以及类型之间的转换只以**命名函数**的形式存在。没有 `cast` 模块：`ptr_cast`、`align_cast`、`align_cast_unchecked` 和 `const_cast` 定义在 `bare::ptr` 中，`reinterpret` 和 `transmute_unchecked` 定义在 `bare::mem` 中，`bare` 根模块重新导出这全部六个函数（`bare::ptr_cast`）以及 `Pod`（[1.2.5](#125-reinterpreta-pod-b-pod位强制转换安全)）。C 和 Rust 中的 `as` 强制转换之所以危险，是因为转换是静默发生的；为了避开这个陷阱，Nabi 要求每一次强制转换都带有**显式的名字**。

## 1.2.1. 4 个强制转换函数

四个基本函数如下：

| 函数                                     | 含义                                       | 风险                    |
|------------------------------------------|--------------------------------------------|-------------------------|
| `ptr_cast<U>(p: Ptr<T>)`                 | 只改变类型（保留来源和地址）               | 对齐由调用者负责        |
| `align_cast<T>(p: Ptr<u8>)`              | 对齐向上转换                               | 有检查（未对齐时为 `none`） |
| `const_cast<T>(p: ConstPtr<T>)`          | 去掉 const                                 | 很高（由调用者保证）    |
| `reinterpret<A: Pod, B: Pod>(a: A) -> B` | 位级强制转换（仅限 `Pod`）                 | 低（由 Pod 标记保证）   |

此外，还有一个在需要更深层绕过时使用的逃生口。

| 函数                                  | 含义                                            | 风险    |
|---------------------------------------|-------------------------------------------------|---------|
| `transmute_unchecked<A,B>(a: A) -> B` | **不做 Pod 检查**的位强制转换（由调用者负责）   | 最高    |

编译器拒绝对指针使用 `as` 关键字，所以用户只能使用命名的强制转换。

## 1.2.2. `ptr_cast<U>`：指针类型转换

```nabi
fn ptr_cast<U, T>(p: Ptr<T>) -> Ptr<U>;
```

这个函数保持地址和来源不变，只改变类型标签。**因此对齐由调用者负责**。它只接受 `Ptr`（Nabi 没有重载）；`ConstPtr` 和 `ManyPtr` 用各自的 `cast` 方法以同样方式改变类型（[1.1](101-ptr.md)）：`p.cast::<U>()`。

```nabi
let p: Ptr<u32> = ...;        // 4-byte aligned
let q: Ptr<u8> = bare::ptr_cast::<u8>(p);    // safe (down-align)
let r: Ptr<u64> = bare::ptr_cast::<u64>(p);  // dangerous: u64 needs 8-byte
                                                 // r.read() may SIGBUS (ARM)
```

在任何构建中都不会插入对齐检查：通过未对齐的结果读写是未定义行为。请先用 `align_cast`（1.2.3）或 `is_aligned`（1.1.2）检查。

### 安全的强制转换：类型等价

布局完全相同的两个结构体之间的转换是安全的。

```nabi
struct A { x: u32 }
struct B { x: u32 }       // same layout as A

let pa: Ptr<A> = ...;
let pb: Ptr<B> = bare::ptr_cast::<B>(pa);    // safe if layout matches
```

不过编译器并不验证布局是否真的一致；这一保证必须由调用者承担。

## 1.2.3. `align_cast<T>`：对齐检查

```nabi
fn align_cast<T>(p: Ptr<u8>) -> Ptr<T> | none;
```

把 `Ptr<u8>`（1 字节对齐）转换为对齐要求更大的 `Ptr<T>` 时，**只有在验证了实际对齐之后**才进行转换。

```nabi
let bytes: ManyPtr<u8> = ...;
let p: Ptr<u8> = bytes.add(offset).as_one();

if let aligned: Ptr<u32> = bare::align_cast::<u32>(p) {
    let v = unsafe { aligned.read() };       // safe (verified)
} else {
    panic("not 4-byte aligned");
}
```

`align_cast` 返回 `T | none` 而不是 `Result`，因为两种形式中前者更轻量。

### unchecked 变体

```nabi
fn align_cast_unchecked<T>(p: Ptr<u8>) -> Ptr<T>;     // without checking
```

只应在调用前刚刚验证过对齐的地方使用它。这个变体即使在调试构建中也不做任何检查。

## 1.2.4. `const_cast<T>`：去掉 const

```nabi
fn const_cast<T>(p: ConstPtr<T>) -> Ptr<T>;
```

这个函数去掉 const。危险是显而易见的：如果原始数据位于只读区域（永生常量），在那里写入会立即导致段错误。

```nabi
use bare::ptr::{Ptr, ConstPtr, const_cast};

let s: str = "hello";              // immortal const
let p: ConstPtr<u8> = s.as_ptr();
let m: Ptr<u8> = const_cast(p);
unsafe { m.write(72u8) };          // 'H': SEGFAULT (write to immortal const)
```

它与 C 的 `const_cast<T*>` 完全同构。它的用途仅限于在要求可变性的地方临时包装只读 API；普通代码**不得使用它**。

## 1.2.5. `reinterpret<A: Pod, B: Pod>`：位强制转换（安全）

```nabi
unsafe trait Pod { /* marker: Plain Old Data */ }

fn reinterpret<A: Pod, B: Pod>(a: A) -> B;
```

对于带有 `Pod`（Plain Old Data）标记的类型，可以保证**安全的 transmute**，原因如下：

- `Pod` 类型的每一种位模式都有效，所以永远不会暴露未初始化的数据。
- 只允许在大小相同的类型之间转换。`where size_of` 约束没有编译期形式，所以大小条件在 `reinterpret` 运行时检查（不一致时触发 panic）。
- 它是值转换，所以对齐自然得到保证。

```nabi
let f: f32 = 3.14;
let bits: u32 = bare::reinterpret(f);     // OK: both Pod, 4 bytes
let f2: f32 = bare::reinterpret(bits);    // round-trip OK
```

### 自动 `Pod` 派生

标准库自动为下列类型提供 `Pod`：

```nabi
unsafe impl Pod for i8, i16, i32, i64, u8, u16, u32, u64, usize, isize { }
unsafe impl Pod for f32, f64 { }
unsafe impl Pod for c_char { }
unsafe impl<T: Pod, const N: usize> Pod for [T; N] { }
unsafe impl<A: Pod> Pod for (A,) { }
unsafe impl<A: Pod, B: Pod> Pod for (A, B) { }
unsafe impl<A: Pod, B: Pod, C: Pod> Pod for (A, B, C) { }
unsafe impl<A: Pod, B: Pod, C: Pod, D: Pod> Pod for (A, B, C, D) { }
```

`bool` 和 `char` 不是 `Pod`：它们的大多数位模式都不是值（`bool` 是 0 或 1，`char` 是 Unicode 标量值）。请重新解释为 `u8` / `u32`，再带检查地转换（`b != 0u8`、`char::from_u32`）。

`Pod` 是 `unsafe trait`（[4.15.4](../../refs-lang/004-functions/415-unsafe.md#4154-在-trait-中)）：它的实现承诺每一种位模式都是值，而编译器无法检查这一点，所以手写的实现是 `unsafe impl Pod for T {}`。`#[derive(Pod)]` 会生成它。

以下类型不会自动获得它：

- `Ptr<T>` 携带单独的来源信息，无法只用位来表示。
- 用户结构体必须用 `#[derive(Pod)]` 显式选择加入。

```nabi
#[derive(Pod)]
#[repr(C)]
struct Vec3 { x: f32; y: f32; z: f32; }

let v: Vec3 = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let bytes: [u8; 12] = bare::reinterpret(v);
let v2: Vec3 = bare::reinterpret(bytes);
```

这一设计沿用 [Project Safe Transmute（Rust RFC 2835）](https://rust-lang.github.io/rfcs/2835-project-safe-transmute.html)模型和 [bytemuck crate](https://docs.rs/bytemuck) 的模式。

## 1.2.6. `transmute_unchecked<A, B>`：逃生口

```nabi
unsafe fn transmute_unchecked<A, B>(a: A) -> B;   // no size check either
```

这个函数连 `Pod` 检查都**不做**，原样搬运位。正如其名，它是最危险的强制转换，所以是 `unsafe fn`：暴露未初始化数据、大小不一致、对齐问题以及内部的 GC 引用都由调用者负责。无法分配临时块时，它以 `mem::transmute_unchecked: out of memory` 触发 panic。大小不一致不会被拒绝；值会经过一个与较大类型同样大小、已清零的临时块，因此超出源类型大小的字节读出为零。

它的用途仅限于标准库内部的**最底层**，只用于无法用其他任何工具表达的情况，例如编写 `Pod` 实现本身。普通用户代码绝不能使用它。

```nabi
// example inside the standard library
unsafe impl Pod for SomeNewtype {
    // empty marker
}

let s: SomeNewtype = ...;
let bits: u64 = unsafe { bare::transmute_unchecked(s) };   // code where SomeNewtype guarantees size 8
```

## 1.2.7. `as` 强制转换的适用范围

`as` 是数值强制转换（[refs-lang 2.14.2](../../refs-lang/002-types/214-cast-and-conversion.md#2142-as-强制转换)）：它可以用在整数类型、浮点类型、`bool`、`char` 和 `c_char` 中的任意两者之间，受 refs-lang 给出的目标限制，结果与 `.wrapping_into()` 相同。有损的组合会回绕（整数），或向零截断并饱和（浮点到整数）。只有指针会被拒绝。

| 表达式             | 结果                                                   |
|--------------------|--------------------------------------------------------|
| `i32 as i64`       | OK（无损扩宽）                                         |
| `i64 as i32`       | OK，回绕（要检查或饱和，用 `try_into` / `saturating_into`） |
| `f32 as f64`       | OK（无损）                                             |
| `i32 as f32`       | OK，舍入到最接近的 `f32`                               |
| `u32 as i32`       | OK，位相同（`0xFFFF_FFFF` 是 `-1`）                    |
| `Ptr<T> as Ptr<U>` | **编译错误**：`bare::ptr_cast::<U>`                |
| `Ptr<T> as usize`  | **编译错误**：`.addr()` 或 `expose_provenance`         |
| `usize as Ptr<T>`  | **编译错误**：`from_exposed_provenance`                |

完整的转换矩阵见 [2.11 conversion](../200-core/211-conversion.md)。

## 1.2.8. 策略小结

本章的强制转换策略归结为四点。

- `as` 强制转换只用于数值；指针永远不经过 `as`，非数值的位重新解释经过 `reinterpret`。
- `bare::ptr_cast` / `align_cast` / `const_cast` / `reinterpret` 这四个函数是显式强制转换的通道。
- `Pod` 标记 trait 提供位重新解释的安全条件。
- `transmute_unchecked` 只作为逃生口暴露。

有了这套策略，Nabi 既避开了 C 繁琐的类型转换，也避开了 Rust 在指针上的 `as` 陷阱。
