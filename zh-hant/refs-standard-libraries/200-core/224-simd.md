# 2.24. simd：可攜式向量

`core::simd` 收容 `Simd<T, N>`，即由單一數值型別 `T` 的 `N` 個線道組成的向量，以及 `Mask<T, N>`，即比較結果的各線道。向量上的運算子同時作用於每個線道。若目標具備可處理這些線道的向量單元，程式碼產生器會把整個向量的操作轉成向量指令，否則轉成逐線道的迴圈；結果在每個目標上都相同。

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

## 2.24.1. 型別

```nabi
pub trait SimdElement: PartialOrd + Debug { /* lane operations */ }
pub struct Simd<T, const N: usize> { /* private: [T; N] */ }
pub struct Mask<T, const N: usize> { /* private: [T; N] */ }
```

- `SimdElement` 由 `i8`、`i16`、`i32`、`i64`、`u8`、`u16`、`u32`、`u64`、`f32` 與 `f64` 實作。它的方法（`lane_add`、`lane_mask`、...）是撰寫向量型別所用的線道操作。
- `Simd<T, N>` 是值型別，與 `[T; N]` 一樣在指派時複製，線道依序排列：線道 0 在前。任何 `N` 都可以，包括 `0`；當 `N * size_of::<T>()` 是 16 位元組的倍數（使用 AVX2 時為 32 位元組）時才會使用向量指令。
- `Mask<T, N>` 是 `Simd<T, N>` 的遮罩：每個線道為設定（所有位元為一）或清除。它具有向量的線道型別，因此 `Mask<f32, 4>` 與 `f32x4` 搭配。
- 在 `extern "C"` 簽章中，8 或 16 位元組的 `Simd<T, N>` 在 AAPCS64 與 x86-64 上是目標中該大小的 C 向量（`float32x4_t`、`__m128`、`__attribute__((vector_size(16)))`），並如 C 一樣以向量暫存器傳遞（[refs-lang 8.6.3](../../refs-lang/008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)）。其他大小、其他目標、以傳值方式傳遞之結構內的 `Simd`，以及 `Mask`，都是編譯錯誤，錯誤訊息會指示改為傳遞指向各線道的指標。在記憶體中，`Simd` 依其線道對齊，而不像 C 的向量那樣對齊到 16 位元組。
- 別名為常見的形狀命名：`f32x2`、`f32x4`、`f32x8`、`f32x16`、`f64x2`、`f64x4`、`f64x8`、`i8x16`、`i8x32`、`i16x8`、`i16x16`、`i32x4`、`i32x8`、`i64x2`、`i64x4`、`u8x16`、`u8x32`、`u16x8`、`u16x16`、`u32x4`、`u32x8`、`u64x2`、`u64x4`（`f32x4` 即 `Simd<f32, 4>`）。

`Simd` 與 `Mask` 實作 `PartialEq`（每個線道都相等；NaN 線道與任何值都不相等）與 `Debug`（線道以陣列表示：`[1.0, 2.0, 3.0, 4.0]`，遮罩則為 `[true, false, ..]`），`Simd` 另外實作 `Default`（每個線道為零）與 `Index<usize>`（`v[i]`，超過最後一個線道時觸發 panic）。

## 2.24.2. 建立、載入與儲存

| 函式 | 意義 |
|---|---|
| `Simd::splat(x)` | 每個線道都是 `x` |
| `Simd::from_array(a)` / `v.to_array()` | 從 `[T; N]` 取得線道 / 以 `[T; N]` 表示線道 |
| `Simd::from_slice(s)` | `Slice<T>` 的前 `N` 個元素；元素較少時觸發 panic |
| `Simd::load(xs, at)` | `List<T>` 的 `xs[at..at + N]`；超過結尾時觸發 panic |
| `v.store(xs, at)` | 將 `xs[at..at + N]` 設為各線道；超過結尾時觸發 panic |
| `v.copy_to(s)` | 設定 `SliceMut<T>` 的前 `N` 個元素；元素較少時觸發 panic |
| `v.get(i)` | 線道 `i`，超過最後一個時為 `none` |
| `v.with_lane(i, x)` | 線道 `i` 設為 `x` 的副本；超過最後一個時觸發 panic |
| `Simd::<T, N>::LANES` | `N` |

`from_slice`、`load`、`store` 與 `copy_to` 的 panic 會指出呼叫、索引與長度（`Simd::load: 4 lanes from index 6 need 10 elements, but there are 8`）。

## 2.24.3. 逐線道運算子

| 形式 | 線道 | 意義 |
|---|---|---|
| `a + b`、`a - b`、`a * b` | 全部 | 整數環繞（如 `wrapping_add`），浮點數依純量運算子捨入 |
| `a / b`、`a % b` | 全部 | 同純量運算子：整數線道遇到除數為零時觸發 panic |
| `-a` | 全部 | 整數環繞（`0 - a`） |
| `a & b`、`a \| b`、`a ^ b`、`!a` | 整數 | 位元運算 |
| `a << n`、`a >> n` | 整數 | 每個線道位移 `u32` 的 `n`，同純量運算子 |
| `a += b`、`-=`、`*=`、`/=` | 全部 | `a = a op b` |
| `a + x`、`a - x`、`a * x`、`a / x` | 全部 | `x: T` 代表 `Simd::splat(x)` |

向量上的整數 `+ - *` 在溢位時環繞而非中止：向量操作就是一條向量指令，而向量指令沒有溢位檢查。若要檢查每個線道，請以 `checked_add` 撰寫 `a.map(..)`。

## 2.24.4. 比較與遮罩

點比較運算子逐線道比較，並產生 `Mask<T, N>`：`a .== b`、`a .!= b`、`a .< b`、`a .<= b`、`a .> b`、`a .>= b`（[2.6](206-broadcast.md)）。浮點線道的比較方式與純量運算子相同：遇到 NaN 時只有 `.!=` 被設定。`a == b` 是代表整個向量的單一 `bool`。

| 函式 | 意義 |
|---|---|
| `Mask::splat(on)`、`Mask::from_array(bools)`、`m.to_array()` | 建立遮罩，或以 `[bool; N]` 讀取其線道 |
| `m.test(i)` | 線道 `i` 是否已設定；超過最後一個時觸發 panic |
| `m.any()`、`m.all()`、`m.count()` | 是否有某些 / 每個線道已設定、已設定的數量 |
| `m.select(yes, no)` | `m` 的線道已設定處取 `yes` 的線道，否則取 `no` 的線道 |
| `m & n`、`m \| n`、`m ^ n`、`!m` | 逐線道邏輯運算 |

```nabi
let x = f32x4::from_array([1.0, -2.0, 3.0, -4.0]);
let pos = x .> f32x4::splat(0.0);
let relu = pos.select(x, f32x4::splat(0.0));    // [1.0, 0.0, 3.0, 0.0]
```

## 2.24.5. 線道數學與歸約

| 函式 | 意義 |
|---|---|
| `a.min(b)`、`a.max(b)`、`a.clamp(lo, hi)` | 逐線道；浮點線道只在 `b` 較小（較大）處才取 `b` 的值，因此 `b` 中的 NaN 會保留 `a` 的線道 |
| `a.abs()` | 每個線道的絕對值；整數在其最小值處環繞 |
| `a.sqrt()`、`a.mul_add(b, c)` | `f32` / `f64` 線道：平方根、只捨入一次的 `a * b + c` |
| `a.map(f)`、`a.zip_with(b, f)` | 對每個線道（或每對線道）套用 `f`，產生 `Simd<U, N>`：`a.map(\|x: i32\| -> f32 { x as f32 })` 進行轉換 |
| `a.reduce_sum()`、`a.reduce_product()` | 從第一個到最後一個合併各線道（整數環繞）；零個線道的和為零 |
| `a.reduce_min()`、`a.reduce_max()` | 最小 / 最大的線道，略過 NaN 線道，除非全部都是 NaN；零個線道時觸發 panic |

浮點歸約依線道順序相加，因此結果在每個目標上都相同；它可能與以其他順序計算的和不同。

## 2.24.6. 重排

| 函式 | 意義 |
|---|---|
| `a.reverse()` | 反序的各線道 |
| `a.rotate_left(k)`、`a.rotate_right(k)` | `rotate_left(k)` 的線道 `i` 是線道 `(i + k) % N` |
| `a.swizzle(idx)` | 線道 `i` 是 `a` 的線道 `idx[i]`，`idx: [usize; N]`；索引超過最後一個線道時觸發 panic |
| `a.interleave(b)` | `(lo, hi)`：`a0 b0 a1 b1 ..`，前 `N` 個線道在 `lo` |
| `a.deinterleave(b)` | 反向操作：`a` 再接 `b` 的偶數線道，以及奇數線道 |

## 2.24.7. 程式碼產生

`+`、`-`、`*`、浮點 `/`、位元運算子、`-a`、`!a`、比較、整數 `min` / `max`、浮點 `abs` 與 `select`，在目標的向量單元具備該線道型別之操作時，會產生向量程式碼：`x86_64`（SSE2；以 `-C target-feature=+avx2` 時 32 位元組向量使用 AVX2）、`i686`（SSE2）、`aarch64`（NEON）、`armv7`（NEON，僅限整數線道）、`powerpc64le`（VMX 與 VSX）、具 SIMD128 的 `wasm32`、具 `vector` 的 `s390x` 以及具 `v` 的 `riscv64`（[0.6.4](../../refs-lang/000-meta/006-platform-targets.md#064-simd)），在每個最佳化等級都是如此，包括 `-O0`。其他情況以及其他操作，則為該操作定義所給出的逐線道迴圈。兩者給出相同的線道：所選的向量指令計算的正是純量運算子所計算的結果；目標缺少的整數乘法（SSE2 上的 `i64` 線道）是迴圈；浮點 `min` / `max` 是迴圈，因為這些指令對 NaN 的處理方式不同。載入、儲存、重排與歸約都是逐線道的迴圈。執行時不做任何偵測。
