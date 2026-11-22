# 3.9. `random::`：亂數位元組、產生器與分布

## 3.9.1. `random`

```nabi
use std::random;

// std::random
fn fill(buf: SliceMut<u8>);      // overwrite every byte with random bytes
fn bytes(n: usize) -> List<u8>;  // n random bytes
fn secret(n: usize) -> Secret;   // n random bytes in secret storage (keys, seeds): no other copy
```

`fill` 是低階形式：它寫入呼叫端擁有的緩衝區，可以是任何可寫入的檢視（[2.9.7](../200-core/209-collections.md)）：`fill(buf)` 填滿整個 `List<u8>`，`fill(buf[4:20])` 只填那幾個位元組。`bytes` 和 `secret` 是會自行配置緩衝區的便利形式。

位元組來自作業系統的密碼學安全亂數產生器，每次呼叫都會讀取（沒有使用者空間的緩衝池，因此分叉出的行程或還原的快照不會重複 `random` 先前緩衝的輸出）：

| 目標 | 來源 |
|---|---|
| Linux（gnu、musl；x86-64、AArch64、i686、ARMv7） | 不帶旗標的 `getrandom(2)`：在核心亂數池初始化之前會阻塞，之後永不阻塞 |
| macOS | `getentropy(2)`，每次呼叫最多 256 個位元組 |
| Windows | 帶 `BCRYPT_USE_SYSTEM_PREFERRED_RNG` 的 `BCryptGenRandom` |
| wasm32-wasip2、wasm32-wasip1-threads | WASI `random_get`（主機的 CSPRNG） |
| wasm32-unknown-unknown | JavaScript 主機的 `crypto.getRandomValues`（透過主機提供的 `random_get` 匯入） |

被中斷的呼叫會重試。來源失敗則無法復原：`fill`、`bytes` 和 `secret` 會觸發 panic，而不是回傳可預測的位元組（與 Go 的 `crypto/rand` 相同）。訊息會指出被呼叫的函式與作業系統錯誤：`random::fill: the OS random source failed (error N)`（`random::bytes: ..`、`random::secret: ..`）。演算法位於 `core::crypto`（2.21），它們以引數接收亂數輸入（金鑰、種子、隨機數（nonce）、salt）；`net::tls`（3.2.5）的所有亂數輸入都從這裡取得。

## 3.9.2. 產生器：`Rng`

```nabi
pub trait Rng {
    fn next_u64(ref self) -> u64;                        // the only required method
    fn f64(ref self) -> f64;                             // [0, 1): (next_u64() >> 11) * 2^-53
    fn f32(ref self) -> f32;                             // [0, 1): (next_u64() >> 40) * 2^-24
    fn below(ref self, n: u64) -> u64;                   // [0, n), unbiased; n == 0 panics
    fn range<R: SampleRange>(ref self, r: R) -> R::Item; // a..b / a..=b of any integer type; empty panics
    fn uniform(ref self, lo: f64, hi: f64) -> f64;       // lo + (hi - lo) * f64()
    fn bool(ref self, p: f64 = 0.5) -> bool;             // f64() < p
    fn normal(ref self, mean: f64 = 0.0, std: f64 = 1.0) -> f64;
    fn exponential(ref self, rate: f64 = 1.0) -> f64;
    fn gamma(ref self, shape: f64, scale: f64 = 1.0) -> f64;
    fn beta(ref self, a: f64, b: f64) -> f64;
    fn poisson(ref self, lam: f64) -> i64;
    fn binomial(ref self, n: i64, p: f64) -> i64;
    fn weighted(ref self, weights: List<f64>) -> usize;  // i with probability weights[i] / sum
    fn shuffle<T>(ref self, xs: List<T>);                // in place
    fn choice<T>(ref self, xs: List<T>) -> T | none;
    fn sample<T>(ref self, xs: List<T>, k: usize) -> List<T>;

    fn fill_f64(ref self, out: SliceMut<f64>);           // f64() each
    fn fill_normal(ref self, out: SliceMut<f64>, mean: f64 = 0.0, std: f64 = 1.0);
    fn random_array<const N: usize>(ref self, shape: [usize; N]) -> Array<f64, N>;
    fn normal_array<const N: usize>(ref self, shape: [usize; N], mean: f64 = 0.0, std: f64 = 1.0) -> Array<f64, N>;
    fn uniform_array<const N: usize>(ref self, shape: [usize; N], lo: f64, hi: f64) -> Array<f64, N>;
}

pub trait SampleRange {                                  // Range<T> and RangeInclusive<T> of every integer type
    type Item;
    fn outcomes(self) -> u64;                            // values in the range, 0 for all 2^64; empty panics
    fn outcome(self, k: u64) -> Item;                    // the value k past the start
}
```

衍生方法由本節固定，使得以種子初始化的產生器在每個目標與每個版本上都給出相同的值。

- `below(n)` 反覆抽取 `next_u64()`，直到其值至少為 `(2^64 - n) mod n`，然後回傳該值 `mod n`。
- `range(r)` 是 `r.outcome(below(r.outcomes()))`，若範圍涵蓋全部 2^64 個值則為 `r.outcome(next_u64())`：對 `a..b` 是 `a + below(b - a)`，對 `a..=b` 是 `a + below(b - a + 1)`，以二補數寬度計算。`range(1..=6)`、`range(0u8..=255)` 和 `range(0usize..n)` 都可以使用；空範圍會觸發 panic，並指出該範圍（`Rng::range: 5..5 is empty; the end must be above the start`）。浮點數範圍則是 `uniform(lo, hi)`。
- `normal` 是 Marsaglia 與 Tsang 的 ziggurat 演算法，採用 Doornik（2005）形式的 256 層：一次 `next_u64()` 同時給出層（其低 8 位元）與一個有號均勻值（其高 53 位元）；表格是 `r = 3.654152885361009` 的表格。幾乎每個隨機變數只需一次抽取。
- `exponential(rate)` 是 `-ln(u) / rate`，其中 `u = ((next_u64() >> 12) + 0.5) * 2^-52` 位於 (0, 1)。
- `gamma(shape, scale)` 是 Marsaglia 與 Tsang（2000）的方法（`shape < 1` 時，取 `shape + 1` 的隨機變數乘以 `u^(1 / shape)`），再乘以 `scale`；`beta(a, b)` 是兩個 gamma 隨機變數的 `x / (x + y)`。
- `poisson(lam)` 在 `lam < 10` 時從 0 開始搜尋 CDF，否則與 NumPy 相同，使用 PTRS（Hormann 1993）；`binomial(n, p)` 在 `n min(p, 1 - p) >= 30` 期間以 beta 分布的順序統計量進行分割，之後反轉 CDF（Devroye 1986）。
- `weighted(ws)` 抽取 `t = f64() * sum`，並回傳累進總和第一個超過 `t` 的索引。
- `shuffle` 是從尾端開始的 Fisher-Yates（位置 `i` 與 `below(i + 1)` 交換）；`choice` 是 `xs[below(len)]`；`sample` 對 `0..k` 中的每個 `i` 將位置 `i` 與 `i + below(len - i)` 交換，並依此順序取出項目。

參數超出分布的定義域時會觸發 panic，訊息包含方法名稱與允許的值：不大於 0 的 rate、shape 或 scale，為負或非有限值的 Poisson 平均數，`binomial` 的 `n < 0` 或位於 [0, 1] 之外的 `p`，為負或非有限值的權重，或總和為 0 的權重。

批次形式以一次呼叫填入多個值：`fill_f64` 和 `fill_normal` 寫入可寫入的檢視（`List<f64>` 會以涵蓋其全部的檢視傳入），而 `random_array`、`normal_array` 和 `uniform_array` 回傳該形狀的全新列優先 `Array`，其值依列優先順序排列（與逐元素各呼叫一次 `f64()` / `normal()` 所得的值相同）。

**`Xoshiro256PlusPlus`**（Blackman 與 Vigna；Julia 的預設）實作 `Rng`。`Xoshiro256PlusPlus::seed(n: u64)` 將四個狀態字組設為從 `n` 開始的四個連續 SplitMix64 輸出；一個種子的串流永遠不會改變，新的演算法會使用新的型別。`from_os()` 以作業系統來源為它設定種子。`split(ref self) -> Xoshiro256PlusPlus` 回傳產生器的副本，並讓原產生器向前跳 2^128 步（Vigna 的 `jump`），為平行工作提供互不重疊、只取決於種子與 `split` 呼叫順序的串流。它不適用於機密資料。

**`OsRng`** 在作業系統來源之上實作 `Rng`（`next_u64` 讀取 8 個位元組，大端序），永遠無法重現。

## 3.9.3. 單次呼叫形式

```nabi
fn rng() -> Xoshiro256PlusPlus;                          // Xoshiro256PlusPlus::from_os()
fn f64() -> f64;
fn range<R: SampleRange>(r: R) -> R::Item;
fn normal(mean: f64 = 0.0, std: f64 = 1.0) -> f64;
fn shuffle<T>(xs: List<T>);
fn choice<T>(xs: List<T>) -> T | none;
```

單次呼叫形式從整個行程共用的一個 `Xoshiro256PlusPlus` 抽取，它在第一次呼叫時以作業系統來源設定種子，並以鎖保護，因此 `random::f64()` 的迴圈不會每個值都產生一次系統呼叫（如 Python 的 `random.random()`、Go 的 `math/rand/v2`）。它們很快，但不適用於機密資料：金鑰、權杖和 nonce 應當來自 `bytes`、`secret` 或 `OsRng`。需要可重複的串流時，應以種子初始化產生器：`Xoshiro256PlusPlus::seed(n)`。

```nabi
use std::random;
use std::random::{Rng, Xoshiro256PlusPlus};

let rng = Xoshiro256PlusPlus::seed(42);
let u = rng.f64();
let k = rng.range(0..10);
let die = rng.range(1..=6);
let x = rng.normal(10.0, 2.0);
let w = rng.weighted([0.2, 0.5, 0.3]);
let noise = rng.normal_array([3, 4]);
let xs = [1, 2, 3, 4];
rng.shuffle(xs);
let worker = rng.split();
let quick = random::range(0..100);                       // the shared generator
```
