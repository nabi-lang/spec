# 2.24. simd - 이식 가능한 벡터

`core::simd`에는 수 타입 `T` 하나의 레인 `N`개로 된 벡터 `Simd<T, N>`과, 비교 결과의 레인인 `Mask<T, N>`이 있습니다. 벡터에 쓴 연산자는 모든 레인에 한꺼번에 적용됩니다. 코드 생성기는 타깃에 그 레인을 다루는 벡터 장치가 있으면 벡터 전체 연산을 벡터 명령으로, 없으면 레인을 도는 루프로 만듭니다. 결과는 모든 타깃에서 같습니다.

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

## 2.24.1. 타입

```nabi
pub trait SimdElement: PartialOrd + Debug { /* 레인 연산 */ }
pub struct Simd<T, const N: usize> { /* 비공개: [T; N] */ }
pub struct Mask<T, const N: usize> { /* 비공개: [T; N] */ }
```

- `SimdElement`는 `i8`, `i16`, `i32`, `i64`, `u8`, `u16`, `u32`, `u64`, `f32`, `f64`가 구현합니다. 그 메서드(`lane_add`, `lane_mask`, ...)는 벡터 타입을 이루는 레인 연산입니다.
- `Simd<T, N>`은 `[T; N]`처럼 대입하면 복사되는 값 타입이고, 레인은 순서대로(레인 0이 처음) 놓입니다. `N`은 `0`을 포함해 무엇이든 됩니다. `N * size_of::<T>()`가 16바이트(AVX2에서는 32바이트)의 배수이면 벡터 명령을 씁니다.
- `Mask<T, N>`은 `Simd<T, N>`의 마스크입니다. 각 레인은 켜져 있거나(모든 비트 1) 꺼져 있습니다. 벡터와 레인 타입이 같으므로 `f32x4`에는 `Mask<f32, 4>`가 짝입니다.
- `extern "C"` 시그니처에서 8바이트나 16바이트인 `Simd<T, N>`은 AAPCS64와 x86-64에서 같은 크기의 C 벡터(`float32x4_t`, `__m128`, `__attribute__((vector_size(16)))`)이고, C가 넘기는 대로 벡터 레지스터로 넘깁니다([refs-lang 8.6.3](../../refs-lang/008-abi/806-calling-convention.md#863-struct-인자--반환-작은-struct-규칙)). 다른 크기, 다른 타깃, 값으로 넘기는 struct 안의 `Simd`, `Mask`는 레인을 가리키는 포인터를 넘기라고 알려 주는 컴파일 에러입니다. 메모리에서 `Simd`는 C 벡터처럼 16바이트가 아니라 레인만큼 정렬됩니다.
- 별칭은 흔한 모양에 이름을 붙입니다: `f32x2`, `f32x4`, `f32x8`, `f32x16`, `f64x2`, `f64x4`, `f64x8`, `i8x16`, `i8x32`, `i16x8`, `i16x16`, `i32x4`, `i32x8`, `i64x2`, `i64x4`, `u8x16`, `u8x32`, `u16x8`, `u16x16`, `u32x4`, `u32x8`, `u64x2`, `u64x4`(`f32x4`는 `Simd<f32, 4>`입니다).

`Simd`와 `Mask`는 `PartialEq`(모든 레인이 같음, NaN 레인은 무엇과도 같지 않음)와 `Debug`(레인을 배열로: `[1.0, 2.0, 3.0, 4.0]`, 마스크는 `[true, false, ..]`)를 구현하고, `Simd`는 `Default`(모든 레인 0)와 `Index<usize>`(`v[i]`, 마지막 레인을 넘으면 panic)를 구현합니다.

## 2.24.2. 만들기, 읽기, 쓰기

| 함수 | 뜻 |
|---|---|
| `Simd::splat(x)` | 모든 레인이 `x` |
| `Simd::from_array(a)` / `v.to_array()` | `[T; N]`에서 레인을 만들거나 레인을 `[T; N]`으로 |
| `Simd::from_slice(s)` | `Slice<T>`의 처음 `N`개 원소. 원소가 모자라면 panic |
| `Simd::load(xs, at)` | `List<T>`의 `xs[at..at + N]`. 끝을 넘으면 panic |
| `v.store(xs, at)` | `xs[at..at + N]`을 레인으로 설정. 끝을 넘으면 panic |
| `v.copy_to(s)` | `SliceMut<T>`의 처음 `N`개 원소를 설정. 원소가 모자라면 panic |
| `v.get(i)` | 레인 `i`, 마지막 레인을 넘으면 `none` |
| `v.with_lane(i, x)` | 레인 `i`를 `x`로 바꾼 복사본. 마지막 레인을 넘으면 panic |
| `Simd::<T, N>::LANES` | `N` |

`from_slice`, `load`, `store`, `copy_to`의 panic은 호출, 인덱스, 길이를 알려 줍니다(`Simd::load: 4 lanes from index 6 need 10 elements, but there are 8`).

## 2.24.3. 레인별 연산자

| 형태 | 레인 | 뜻 |
|---|---|---|
| `a + b`, `a - b`, `a * b` | 모두 | 정수는 감쌉니다(`wrapping_add`처럼). 부동소수는 스칼라 연산자처럼 반올림합니다 |
| `a / b`, `a % b` | 모두 | 스칼라 연산자와 같습니다. 정수 레인은 나누는 수가 0이면 panic |
| `-a` | 모두 | 정수는 감쌉니다(`0 - a`) |
| `a & b`, `a \| b`, `a ^ b`, `!a` | 정수 | 비트 연산 |
| `a << n`, `a >> n` | 정수 | 각 레인을 `u32` `n`만큼 스칼라 연산자처럼 자리 이동 |
| `a += b`, `-=`, `*=`, `/=` | 모두 | `a = a op b` |
| `a + x`, `a - x`, `a * x`, `a / x` | 모두 | `x: T`는 `Simd::splat(x)`를 뜻합니다 |

벡터 정수 `+ - *`는 넘침에서 트랩하지 않고 감쌉니다. 벡터 연산은 넘침 검사가 없는 벡터 명령 하나이기 때문입니다. 레인마다 검사하려면 `checked_add`로 `a.map(..)`을 씁니다.

## 2.24.4. 비교와 마스크

점 비교는 레인끼리 비교해 `Mask<T, N>`을 줍니다: `a .== b`, `a .!= b`, `a .< b`, `a .<= b`, `a .> b`, `a .>= b`([2.6](206-broadcast.md)). 부동소수 레인은 스칼라 연산자처럼 비교하므로 NaN과는 `.!=`만 켜집니다. `a == b`는 벡터 전체에 대한 `bool` 하나입니다.

| 함수 | 뜻 |
|---|---|
| `Mask::splat(on)`, `Mask::from_array(bools)`, `m.to_array()` | 마스크를 만들거나 레인을 `[bool; N]`으로 읽기 |
| `m.test(i)` | 레인 `i`가 켜졌는지. 마지막 레인을 넘으면 panic |
| `m.any()`, `m.all()`, `m.count()` | 켜진 레인이 있는지 / 모두 켜졌는지 / 몇 개인지 |
| `m.select(yes, no)` | `m`의 레인이 켜진 곳은 `yes`의 레인, 아니면 `no`의 레인 |
| `m & n`, `m \| n`, `m ^ n`, `!m` | 레인별 논리 |

```nabi
let x = f32x4::from_array([1.0, -2.0, 3.0, -4.0]);
let pos = x .> f32x4::splat(0.0);
let relu = pos.select(x, f32x4::splat(0.0));    // [1.0, 0.0, 3.0, 0.0]
```

## 2.24.5. 레인 수학과 리덕션

| 함수 | 뜻 |
|---|---|
| `a.min(b)`, `a.max(b)`, `a.clamp(lo, hi)` | 레인별. 부동소수 레인은 `b`가 더 작을(클) 때만 `b`의 레인을 택하므로 `b`의 NaN은 `a`의 레인을 남깁니다 |
| `a.abs()` | 각 레인의 크기. 정수는 최솟값에서 감쌉니다 |
| `a.sqrt()`, `a.mul_add(b, c)` | `f32` / `f64` 레인: 제곱근, 한 번만 반올림한 `a * b + c` |
| `a.map(f)`, `a.zip_with(b, f)` | 각 레인(또는 쌍)에 `f`를 적용한 `Simd<U, N>`: `a.map(\|x: i32\| -> f32 { x as f32 })`로 변환합니다 |
| `a.reduce_sum()`, `a.reduce_product()` | 레인을 처음부터 끝까지 합침(정수는 감쌈). 레인이 없으면 합은 0 |
| `a.reduce_min()`, `a.reduce_max()` | 가장 작은 / 큰 레인. 모두 NaN이 아니면 NaN 레인은 건너뜁니다. 레인이 없으면 panic |

부동소수 리덕션은 레인 순서대로 더하므로 결과가 모든 타깃에서 같습니다. 다른 순서로 더한 합과는 다를 수 있습니다.

## 2.24.6. 셔플

| 함수 | 뜻 |
|---|---|
| `a.reverse()` | 레인을 거꾸로 |
| `a.rotate_left(k)`, `a.rotate_right(k)` | `rotate_left(k)`의 레인 `i`는 레인 `(i + k) % N` |
| `a.swizzle(idx)` | 레인 `i`는 `a`의 레인 `idx[i]`, `idx: [usize; N]`. 마지막 레인을 넘는 인덱스는 panic |
| `a.interleave(b)` | `(lo, hi)`: `a0 b0 a1 b1 ..`, 처음 `N`개 레인은 `lo`에 |
| `a.deinterleave(b)` | 그 역: `a`와 `b`의 짝수 레인, 그리고 홀수 레인 |

## 2.24.7. 코드 생성

`+`, `-`, `*`, 부동소수 `/`, 비트 연산자, `-a`, `!a`, 비교, 정수 `min` / `max`, 부동소수 `abs`, `select`는 타깃의 벡터 장치가 그 레인 타입의 연산을 가질 때 벡터 코드입니다: `x86_64`(SSE2, `-C target-feature=+avx2`이면 32바이트 벡터에 AVX2), `i686`(SSE2), `aarch64`(NEON), `armv7`(NEON, 정수 레인만), `powerpc64le`(VMX와 VSX), SIMD128이 있는 `wasm32`, `vector`가 있는 `s390x`, `v`가 있는 `riscv64`([0.6.4](../../refs-lang/000-meta/006-platform-targets.md#064-simd)). `-O0`을 포함한 모든 최적화 수준에서 그렇습니다. 그 밖의 타깃과 다른 연산은 정의가 주는 레인 루프입니다. 둘은 같은 레인을 줍니다. 벡터 명령은 스칼라 연산자가 계산하는 값을 계산하도록 고르며, 타깃에 없는 정수 곱셈(SSE2의 `i64` 레인)은 루프이고, 부동소수 `min` / `max`는 명령이 NaN을 다르게 다루므로 루프입니다. 읽기, 쓰기, 셔플, 리덕션은 레인을 도는 루프입니다. 실행 중에 무엇도 감지하지 않습니다.
