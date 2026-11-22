# 2.24. simd：可移植向量

`core::simd` 包含 `Simd<T, N>` 和 `Mask<T, N>`：前者是由同一数值类型 `T` 的 `N` 个通道（lane）组成的向量，后者是比较结果的各通道。作用于向量的运算符同时作用于每个通道。若目标具有适用于这些通道的向量单元，代码生成器会把整向量操作变为向量指令，否则变为对各通道的循环；在每个目标上结果都相同。

```nabi
use core::simd::{f32x4, Simd};

fn dot(a: List<f32>, b: List<f32>) -> f32 {
    let acc = f32x4::splat(0.0);
    let i = 0;
    while i + 4 <= a.len() {
        acc += f32x4::load(a, i) * f32x4::load(b, i);
        i += 4;
    }
    let s = acc.reduce_sum();
    while i < a.len() {
        s += a[i] * b[i];
        i += 1;
    }
    s
}
```

## 2.24.1. 类型

```nabi
pub trait SimdElement: PartialOrd + Debug { /* lane operations */ }
pub struct Simd<T, const N: usize> { /* private: [T; N] */ }
pub struct Mask<T, const N: usize> { /* private: [T; N] */ }
```

- `SimdElement` 由 `i8`、`i16`、`i32`、`i64`、`u8`、`u16`、`u32`、`u64`、`f32` 和 `f64` 实现。它的方法（`lane_add`、`lane_mask` 等）是编写向量类型所用的通道操作。
- `Simd<T, N>` 是值类型，与 `[T; N]` 一样在赋值时复制，通道按顺序排列：通道 0 在前。任何 `N` 都可以，包括 `0`；当 `N * size_of::<T>()` 是 16 字节（使用 AVX2 时为 32 字节）的倍数时使用向量指令。
- `Mask<T, N>` 是 `Simd<T, N>` 的掩码：每个通道要么置位（所有位为 1），要么清零。它带有向量的通道类型，所以 `Mask<f32, 4>` 与 `f32x4` 搭配。
- 在 `extern "C"` 签名中，8 或 16 字节的 `Simd<T, N>` 在 AAPCS64 和 x86-64 上是目标上该大小的 C 向量（`float32x4_t`、`__m128`、`__attribute__((vector_size(16)))`），与 C 一样通过向量寄存器传递（[refs-lang 8.6.3](../../refs-lang/008-abi/806-calling-convention.md#863-struct-实参--返回小-struct-规则)）。其他大小、其他目标、按值传递的结构体中的 `Simd` 以及 `Mask` 都是编译错误，错误信息会提示改为传递指向各通道的指针。在内存中，`Simd` 按其通道对齐，而不是像 C 的向量那样按 16 字节对齐。
- 别名为常见形状命名：`f32x2`、`f32x4`、`f32x8`、`f32x16`、`f64x2`、`f64x4`、`f64x8`、`i8x16`、`i8x32`、`i16x8`、`i16x16`、`i32x4`、`i32x8`、`i64x2`、`i64x4`、`u8x16`、`u8x32`、`u16x8`、`u16x16`、`u32x4`、`u32x8`、`u64x2`、`u64x4`（`f32x4` 即 `Simd<f32, 4>`）。

`Simd` 和 `Mask` 实现了 `PartialEq`（每个通道都相等；NaN 通道不等于任何值）和 `Debug`（通道以数组形式显示：`[1.0, 2.0, 3.0, 4.0]`，掩码显示为 `[true, false, ..]`），`Simd` 还实现了 `Default`（每个通道为零）和 `Index<usize>`（`v[i]`，超过最后一个通道时触发 panic）。

## 2.24.2. 构造、加载与存储

| 函数 | 含义 |
|---|---|
| `Simd::splat(x)` | 每个通道都是 `x` |
| `Simd::from_array(a)` / `v.to_array()` | 从 `[T; N]` 得到各通道 / 把各通道转为 `[T; N]` |
| `Simd::from_slice(s)` | `Slice<T>` 的前 `N` 个元素；元素不足时触发 panic |
| `Simd::load(xs, at)` | `List<T>` 的 `xs[at..at + N]`；越过末尾时触发 panic |
| `v.store(xs, at)` | 把 `xs[at..at + N]` 设为各通道；越过末尾时触发 panic |
| `v.copy_to(s)` | 设置 `SliceMut<T>` 的前 `N` 个元素；元素不足时触发 panic |
| `v.get(i)` | 通道 `i`，越过最后一个时为 `none` |
| `v.with_lane(i, x)` | 把通道 `i` 设为 `x` 的副本；越过最后一个时触发 panic |
| `Simd::<T, N>::LANES` | `N` |

`from_slice`、`load`、`store` 和 `copy_to` 的 panic 消息会写出调用、索引和长度（`Simd::load: 4 lanes from index 6 need 10 elements, but there are 8`）。

## 2.24.3. 逐通道运算符

| 形式 | 通道 | 含义 |
|---|---|---|
| `a + b`、`a - b`、`a * b` | 全部 | 整数回绕（同 `wrapping_add`），浮点数按标量运算符舍入 |
| `a / b`、`a % b` | 全部 | 与标量运算符相同：整数通道的除数为零时触发 panic |
| `-a` | 全部 | 整数回绕（`0 - a`） |
| `a & b`、`a \| b`、`a ^ b`、`!a` | 整数 | 按位运算 |
| `a << n`、`a >> n` | 整数 | 每个通道按 `u32` 类型的 `n` 移位，与标量运算符相同 |
| `a += b`、`-=`、`*=`、`/=` | 全部 | `a = a op b` |
| `a + x`、`a - x`、`a * x`、`a / x` | 全部 | `x: T` 表示 `Simd::splat(x)` |

向量上的整数 `+ - *` 在溢出时回绕而不是陷入（trap）：向量操作是一条向量指令，没有溢出检查。要检查每个通道，写 `a.map(..)` 并配合 `checked_add`。

## 2.24.4. 比较与掩码

点比较逐通道比较并给出 `Mask<T, N>`：`a .== b`、`a .!= b`、`a .< b`、`a .<= b`、`a .> b`、`a .>= b`（[2.6](206-broadcast.md)）。浮点通道与标量运算符的比较方式相同：涉及 NaN 时只有 `.!=` 置位。`a == b` 是针对整个向量的一个 `bool`。

| 函数 | 含义 |
|---|---|
| `Mask::splat(on)`、`Mask::from_array(bools)`、`m.to_array()` | 构造掩码，或以 `[bool; N]` 读取其各通道 |
| `m.test(i)` | 通道 `i` 是否置位；越过最后一个时触发 panic |
| `m.any()`、`m.all()`、`m.count()` | 是否有通道 / 是否每个通道都置位，以及置位的个数 |
| `m.select(yes, no)` | `m` 的通道置位处取 `yes` 的通道，否则取 `no` 的 |
| `m & n`、`m \| n`、`m ^ n`、`!m` | 逐通道逻辑运算 |

```nabi
let x = f32x4::from_array([1.0, -2.0, 3.0, -4.0]);
let pos = x .> f32x4::splat(0.0);
let relu = pos.select(x, f32x4::splat(0.0));    // [1.0, 0.0, 3.0, 0.0]
```

## 2.24.5. 通道数学与归约

| 函数 | 含义 |
|---|---|
| `a.min(b)`、`a.max(b)`、`a.clamp(lo, hi)` | 逐通道；浮点通道仅在 `b` 的通道更小（更大）时取 `b` 的，所以 `b` 中的 NaN 会保留 `a` 的通道 |
| `a.abs()` | 每个通道的绝对值；整数在其最小值处回绕 |
| `a.sqrt()`、`a.mul_add(b, c)` | `f32` / `f64` 通道：平方根，以及只舍入一次的 `a * b + c` |
| `a.map(f)`、`a.zip_with(b, f)` | 对每个通道（或每对通道）调用 `f`，得到 `Simd<U, N>`：`a.map(\|x: i32\| -> f32 { x as f32 })` 用于转换 |
| `a.reduce_sum()`、`a.reduce_product()` | 从第一个到最后一个依次合并各通道（整数回绕）；零个通道的和为零 |
| `a.reduce_min()`、`a.reduce_max()` | 最小 / 最大的通道，跳过 NaN 通道（全部为 NaN 时除外）；零个通道时触发 panic |

浮点归约按通道顺序相加，所以其结果在每个目标上都相同；它可能与按其他顺序求得的和不同。

## 2.24.6. 重排

| 函数 | 含义 |
|---|---|
| `a.reverse()` | 通道顺序反转 |
| `a.rotate_left(k)`、`a.rotate_right(k)` | `rotate_left(k)` 的通道 `i` 是通道 `(i + k) % N` |
| `a.swizzle(idx)` | 通道 `i` 是 `a` 的通道 `idx[i]`，`idx: [usize; N]`；索引越过最后一个通道时触发 panic |
| `a.interleave(b)` | `(lo, hi)`：`a0 b0 a1 b1 ..`，前 `N` 个通道在 `lo` 中 |
| `a.deinterleave(b)` | 逆操作：`a` 然后 `b` 的偶数通道，以及奇数通道 |

## 2.24.7. 代码生成

当目标的向量单元对该通道类型具有相应操作时，`+`、`-`、`*`、浮点 `/`、按位运算符、`-a`、`!a`、比较、整数 `min` / `max`、浮点 `abs` 和 `select` 会生成向量代码：`x86_64`（SSE2；对 32 字节向量配合 `-C target-feature=+avx2` 使用 AVX2）、`i686`（SSE2）、`aarch64`（NEON）、`armv7`（NEON，仅整数通道）、`powerpc64le`（VMX 和 VSX）、带 SIMD128 的 `wasm32`、带 `vector` 的 `s390x` 以及带 `v` 的 `riscv64`（[0.6.4](../../refs-lang/000-meta/006-platform-targets.md#064-simd)），在每个优化级别上都如此，包括 `-O0`。在其他情况下以及对其他操作，操作就是其定义给出的对各通道的循环。两者给出相同的通道：所选的向量指令计算的正是标量运算符所计算的结果，目标缺少的整数乘法（SSE2 上的 `i64` 通道）是循环，浮点 `min` / `max` 是循环，因为相应指令对 NaN 的处理不同。加载、存储、重排和归约是对各通道的循环。运行时不做任何检测。
