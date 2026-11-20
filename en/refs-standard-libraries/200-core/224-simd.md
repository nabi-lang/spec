# 2.24. simd - portable vectors

`core::simd` holds `Simd<T, N>`, a vector of `N` lanes of one number type `T`, and `Mask<T, N>`, the lanes of a comparison. An operator on vectors works on every lane at once. The code generator turns the whole-vector operations into vector instructions where the target has a vector unit for the lanes, and into a loop over the lanes elsewhere; the result is the same on every target.

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

## 2.24.1. Types

```nabi
pub trait SimdElement: PartialOrd + Debug { /* lane operations */ }
pub struct Simd<T, const N: usize> { /* private: [T; N] */ }
pub struct Mask<T, const N: usize> { /* private: [T; N] */ }
```

- `SimdElement` is implemented by `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32` and `f64`. Its methods (`lane_add`, `lane_mask`, ...) are the lane operations the vector types are written with.
- `Simd<T, N>` is a value type, copied on assignment like `[T; N]`, with the lanes in order: lane 0 first. Any `N` works, `0` included; vector instructions are used for `N * size_of::<T>()` a multiple of 16 bytes (or 32 bytes with AVX2).
- `Mask<T, N>` is the mask of a `Simd<T, N>`: each lane is set (all bits one) or clear. It has the vector's lane type, so `Mask<f32, 4>` goes with `f32x4`.
- In an `extern "C"` signature a `Simd<T, N>` of 8 or 16 bytes is the target's C vector of that size (`float32x4_t`, `__m128`, `__attribute__((vector_size(16)))`) on AAPCS64 and x86-64, passed in vector registers as C passes it ([refs-lang 8.6.3](../../refs-lang/008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)). Other sizes, other targets, a `Simd` inside a struct passed by value and `Mask` are compile errors that say to pass a pointer to the lanes. In memory a `Simd` is aligned as its lanes, not to 16 bytes as C's vectors are.
- The aliases name the common shapes: `f32x2`, `f32x4`, `f32x8`, `f32x16`, `f64x2`, `f64x4`, `f64x8`, `i8x16`, `i8x32`, `i16x8`, `i16x16`, `i32x4`, `i32x8`, `i64x2`, `i64x4`, `u8x16`, `u8x32`, `u16x8`, `u16x16`, `u32x4`, `u32x8`, `u64x2`, `u64x4` (`f32x4` is `Simd<f32, 4>`).

`Simd` and `Mask` implement `PartialEq` (every lane equal; a NaN lane equals nothing), `Debug` (the lanes as an array: `[1.0, 2.0, 3.0, 4.0]`, a mask as `[true, false, ..]`), and `Simd` implements `Default` (every lane zero) and `Index<usize>` (`v[i]`, which panics past the last lane).

## 2.24.2. Building, loading and storing

| Function | Meaning |
|---|---|
| `Simd::splat(x)` | every lane `x` |
| `Simd::from_array(a)` / `v.to_array()` | the lanes from / as a `[T; N]` |
| `Simd::from_slice(s)` | the first `N` elements of a `Slice<T>`; panics when it has fewer |
| `Simd::load(xs, at)` | `xs[at..at + N]` of a `List<T>`; panics past the end |
| `v.store(xs, at)` | sets `xs[at..at + N]` to the lanes; panics past the end |
| `v.copy_to(s)` | sets the first `N` elements of a `SliceMut<T>`; panics when it has fewer |
| `v.get(i)` | lane `i`, or `none` past the last |
| `v.with_lane(i, x)` | a copy with lane `i` set to `x`; panics past the last |
| `Simd::<T, N>::LANES` | `N` |

The panics of `from_slice`, `load`, `store` and `copy_to` name the call, the index and the length (`Simd::load: 4 lanes from index 6 need 10 elements, but there are 8`).

## 2.24.3. Lane-wise operators

| Form | Lanes | Meaning |
|---|---|---|
| `a + b`, `a - b`, `a * b` | all | integers wrap (as `wrapping_add`), floats round as the scalar operator |
| `a / b`, `a % b` | all | as the scalar operator: an integer lane panics on a zero divisor |
| `-a` | all | integers wrap (`0 - a`) |
| `a & b`, `a \| b`, `a ^ b`, `!a` | integers | bitwise |
| `a << n`, `a >> n` | integers | each lane shifted by the `u32` `n`, as the scalar operator |
| `a += b`, `-=`, `*=`, `/=` | all | `a = a op b` |
| `a + x`, `a - x`, `a * x`, `a / x` | all | `x: T` stands for `Simd::splat(x)` |

An integer `+ - *` on vectors wraps rather than trapping on overflow: a vector operation is a vector instruction, which has no overflow check. Write `a.map(..)` with `checked_add` to check each lane.

## 2.24.4. Comparisons and masks

The dot comparisons compare lane by lane and give a `Mask<T, N>`: `a .== b`, `a .!= b`, `a .< b`, `a .<= b`, `a .> b`, `a .>= b` ([2.6](206-broadcast.md)). A float lane compares as the scalar operator does: with a NaN only `.!=` is set. `a == b` is one `bool` for the whole vector.

| Function | Meaning |
|---|---|
| `Mask::splat(on)`, `Mask::from_array(bools)`, `m.to_array()` | build a mask, or read its lanes as `[bool; N]` |
| `m.test(i)` | whether lane `i` is set; panics past the last |
| `m.any()`, `m.all()`, `m.count()` | whether some / every lane is set, how many are |
| `m.select(yes, no)` | `yes`'s lane where `m`'s is set, else `no`'s |
| `m & n`, `m \| n`, `m ^ n`, `!m` | lane-wise logic |

```nabi
let x = f32x4::from_array([1.0, -2.0, 3.0, -4.0]);
let pos = x .> f32x4::splat(0.0);
let relu = pos.select(x, f32x4::splat(0.0));    // [1.0, 0.0, 3.0, 0.0]
```

## 2.24.5. Lane math and reductions

| Function | Meaning |
|---|---|
| `a.min(b)`, `a.max(b)`, `a.clamp(lo, hi)` | lane-wise; a float lane takes `b`'s only where it is less (greater), so a NaN in `b` keeps `a`'s lane |
| `a.abs()` | each lane's magnitude; an integer wraps at its least value |
| `a.sqrt()`, `a.mul_add(b, c)` | `f32` / `f64` lanes: the square root, `a * b + c` rounded once |
| `a.map(f)`, `a.zip_with(b, f)` | `f` of each lane (or pair), into a `Simd<U, N>`: `a.map(\|x: i32\| -> f32 { x as f32 })` converts |
| `a.reduce_sum()`, `a.reduce_product()` | the lanes combined first to last (integers wrap); the sum of no lanes is zero |
| `a.reduce_min()`, `a.reduce_max()` | the least / greatest lane, NaN lanes skipped unless all are NaN; panics on zero lanes |

A float reduction adds in lane order, so its result is the same on every target; it may differ from a sum in another order.

## 2.24.6. Shuffles

| Function | Meaning |
|---|---|
| `a.reverse()` | the lanes in reverse order |
| `a.rotate_left(k)`, `a.rotate_right(k)` | lane `i` of `rotate_left(k)` is lane `(i + k) % N` |
| `a.swizzle(idx)` | lane `i` is lane `idx[i]` of `a`, `idx: [usize; N]`; panics on an index past the last lane |
| `a.interleave(b)` | `(lo, hi)`: `a0 b0 a1 b1 ..`, the first `N` lanes in `lo` |
| `a.deinterleave(b)` | the inverse: the even lanes of `a` then `b`, and the odd ones |

## 2.24.7. Code generation

`+`, `-`, `*`, a float `/`, the bitwise operators, `-a`, `!a`, the comparisons, integer `min` / `max`, float `abs` and `select` are vector code where the target's vector unit has the operation for the lane type: `x86_64` (SSE2; AVX2 for 32-byte vectors with `-C target-feature=+avx2`), `i686` (SSE2), `aarch64` (NEON), `armv7` (NEON, integer lanes only), `powerpc64le` (VMX and VSX), `wasm32` with SIMD128, `s390x` with `vector` and `riscv64` with `v` ([0.6.4](../../refs-lang/000-meta/006-platform-targets.md#064-simd)), at every optimization level, `-O0` included. Elsewhere, and for the other operations, the operation is the loop over the lanes that its definition gives. Both give the same lanes: the vector instructions are chosen to compute what the scalar operators compute, an integer multiply that a target lacks (`i64` lanes on SSE2) is a loop, and a float `min` / `max` is a loop since the instructions treat NaN otherwise. Loads, stores, shuffles and reductions are loops over the lanes. Nothing is detected at run time.
