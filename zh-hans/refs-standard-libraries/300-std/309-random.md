# 3.9. `random::`：随机字节、生成器与分布

## 3.9.1. `random`

```nabi
use std::random;

// std::random
fn fill(buf: SliceMut<u8>);      // overwrite every byte with random bytes
fn bytes(n: usize) -> List<u8>;  // n random bytes
fn secret(n: usize) -> Secret;   // n random bytes in secret storage (keys, seeds): no other copy
```

`fill` 是底层形式：它写入调用者拥有的缓冲区，即缓冲区的任意可写视图（[2.9.7](../200-core/209-collections.md)）：`fill(buf)` 填满整个 `List<u8>`，`fill(buf[4:20])` 只填那些字节。`bytes` 和 `secret` 是替你分配缓冲区的便捷形式。

字节来自操作系统的密码学安全生成器，每次调用都重新读取（没有用户空间的池，所以派生出的进程或恢复的快照不会重复 `random` 缓冲过的输出）：

| 目标 | 来源 |
|---|---|
| Linux（gnu、musl；x86-64、AArch64、i686、ARMv7） | 不带标志的 `getrandom(2)`：在内核池初始化之前阻塞，之后从不阻塞 |
| macOS | `getentropy(2)`，每次调用最多 256 字节 |
| Windows | 带 `BCRYPT_USE_SYSTEM_PREFERRED_RNG` 的 `BCryptGenRandom` |
| wasm32-wasip2、wasm32-wasip1-threads | WASI `random_get`（宿主的 CSPRNG） |
| wasm32-unknown-unknown | JavaScript 宿主的 `crypto.getRandomValues`（通过宿主提供的 `random_get` 导入） |

被中断的调用会重试。来源失败无法恢复：`fill`、`bytes` 和 `secret` 会触发 panic，而不是返回可预测的字节（与 Go 的 `crypto/rand` 相同）。消息会写出被调用的函数和操作系统错误：`random::fill: the OS random source failed (error N)`（`random::bytes: ..`、`random::secret: ..`）。算法在 `core::crypto`（2.21）中，它们以参数形式接收随机输入（密钥、种子、nonce、盐）；`net::tls`（3.2.5）的所有随机输入都从这里取得。

## 3.9.2. 生成器：`Rng`

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

派生方法由本节固定，使带种子的生成器在每个目标和每个版本上都给出相同的值。

- `below(n)` 反复抽取 `next_u64()`，直到它不小于 `(2^64 - n) mod n`，然后返回它 `mod n` 的值。
- `range(r)` 是 `r.outcome(below(r.outcomes()))`，对于包含全部 2^64 个值的区间则是 `r.outcome(next_u64())`：对 `a..b` 是 `a + below(b - a)`，对 `a..=b` 是 `a + below(b - a + 1)`，按补码位宽计算。`range(1..=6)`、`range(0u8..=255)` 和 `range(0usize..n)` 都可以使用；空区间会触发 panic，并写出该区间（`Rng::range: 5..5 is empty; the end must be above the start`）。浮点区间用 `uniform(lo, hi)`。
- `normal` 是 Marsaglia 与 Tsang 的 ziggurat 算法，采用 Doornik（2005）的 256 层形式：一次 `next_u64()` 给出层（低 8 位）和一个有符号均匀值（高 53 位）；表格对应 `r = 3.654152885361009`。几乎每个随机变量都只需一次抽取。
- `exponential(rate)` 是 `-ln(u) / rate`，其中 `u = ((next_u64() >> 12) + 0.5) * 2^-52`，位于 (0, 1) 内。
- `gamma(shape, scale)` 是 Marsaglia 与 Tsang（2000）的算法（`shape < 1` 时，取 `shape + 1` 的随机变量乘以 `u^(1 / shape)`），再乘以 `scale`；`beta(a, b)` 是两个 gamma 随机变量的 `x / (x + y)`。
- `poisson(lam)` 在 `lam < 10` 时从 0 开始搜索 CDF，否则使用 PTRS（Hormann 1993），与 NumPy 相同；`binomial(n, p)` 在 `n min(p, 1 - p) >= 30` 期间按服从 beta 分布的顺序统计量拆分，之后对 CDF 求逆（Devroye 1986）。
- `weighted(ws)` 抽取 `t = f64() * sum`，返回累加和第一个超过 `t` 的索引。
- `shuffle` 是从后往前的 Fisher-Yates（位置 `i` 与 `below(i + 1)` 交换）；`choice` 是 `xs[below(len)]`；`sample` 对 `0..k` 中的每个 `i` 把位置 `i` 与 `i + below(len - i)` 交换，并按该顺序取出元素。

超出分布定义域的参数会触发 panic，消息写出方法名和允许的取值：不大于 0 的 rate、shape 或 scale，负的或非有限的 Poisson 均值，`binomial` 的 `n < 0` 或 [0, 1] 之外的 `p`，负的或非有限的权重，以及总和为 0 的权重。

批量形式一次调用填充许多值：`fill_f64` 和 `fill_normal` 写入一个可写视图（`List<f64>` 作为其全部内容的视图传入），`random_array`、`normal_array` 和 `uniform_array` 返回给定形状的新的行优先 `Array`，其中的值按行优先顺序排列（与逐元素调用一次 `f64()` / `normal()` 得到的值相同）。

**`Xoshiro256PlusPlus`**（Blackman 与 Vigna；Julia 的默认生成器）实现了 `Rng`。`Xoshiro256PlusPlus::seed(n: u64)` 把四个状态字设为由 `n` 起始的 SplitMix64 的四个连续输出；一个种子对应的序列永远不变，新算法会使用新类型。`from_os()` 用操作系统来源为它设置种子。`split(ref self) -> Xoshiro256PlusPlus` 返回生成器的一个副本，并让原生成器向前跳 2^128 步（Vigna 的 `jump`），从而为并行工作提供互不重叠、只取决于种子和 `split` 调用顺序的序列。它不适用于机密数据。

**`OsRng`** 基于操作系统来源实现 `Rng`（`next_u64` 读取 8 字节，大端），永远不可复现。

## 3.9.3. 单次调用形式

```nabi
fn rng() -> Xoshiro256PlusPlus;                          // Xoshiro256PlusPlus::from_os()
fn f64() -> f64;
fn range<R: SampleRange>(r: R) -> R::Item;
fn normal(mean: f64 = 0.0, std: f64 = 1.0) -> f64;
fn shuffle<T>(xs: List<T>);
fn choice<T>(xs: List<T>) -> T | none;
```

单次调用形式从进程共用的一个 `Xoshiro256PlusPlus` 中抽取：它在第一次调用时用操作系统来源设置种子，并放在锁后面，所以循环调用 `random::f64()` 不会为每个值付出一次系统调用（类似 Python 的 `random.random()`、Go 的 `math/rand/v2`）。它们速度快，但不适用于机密数据：密钥、令牌和 nonce 应当来自 `bytes`、`secret` 或 `OsRng`。需要可重复的序列时，为生成器设置种子：`Xoshiro256PlusPlus::seed(n)`。

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
