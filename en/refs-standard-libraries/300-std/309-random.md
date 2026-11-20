# 3.9. `random::` - random bytes, generators and distributions

## 3.9.1. `random`

```nabi
use std::random;

// std::random
fn fill(buf: SliceMut<u8>);      // overwrite every byte with random bytes
fn bytes(n: usize) -> List<u8>;  // n random bytes
fn secret(n: usize) -> Secret;   // n random bytes in secret storage (keys, seeds): no other copy
```

`fill` is the low-level form: it writes into a buffer the caller owns, any writable view of one ([2.9.7](../200-core/209-collections.md)): `fill(buf)` fills a whole `List<u8>`, `fill(buf[4:20])` only those bytes. `bytes` and `secret` are the convenient forms that allocate the buffer.

Bytes come from the operating system's cryptographically secure generator, read on every call (no user-space pool, so a forked process or a restored snapshot does not repeat output that `random` buffered):

| Target | Source |
|---|---|
| Linux (gnu, musl; x86-64, AArch64, i686, ARMv7) | `getrandom(2)` with no flags: blocks until the kernel pool is initialized, then never blocks |
| macOS | `getentropy(2)`, at most 256 bytes per call |
| Windows | `BCryptGenRandom` with `BCRYPT_USE_SYSTEM_PREFERRED_RNG` |
| wasm32-wasip2, wasm32-wasip1-threads | WASI `random_get` (the host's CSPRNG) |
| wasm32-unknown-unknown | the JavaScript host's `crypto.getRandomValues` (through the `random_get` import the host provides) |

An interrupted call is retried. A failing source cannot be recovered from: `fill`, `bytes` and `secret` panic rather than return predictable bytes (as Go's `crypto/rand`). The message names the called function and the OS error: `random::fill: the OS random source failed (error N)` (`random::bytes: ..`, `random::secret: ..`). The algorithms are `core::crypto` (2.21), which take their random inputs (keys, seeds, nonces, salts) as arguments; `net::tls` (3.2.5) draws all of its from here.

## 3.9.2. Generators - `Rng`

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

The derived methods are fixed by this section so that a seeded generator gives the same values on every target and release.

- `below(n)` draws `next_u64()` until it is at least `(2^64 - n) mod n` and returns it `mod n`.
- `range(r)` is `r.outcome(below(r.outcomes()))`, or `r.outcome(next_u64())` for a range of all 2^64 values: `a + below(b - a)` for `a..b` and `a + below(b - a + 1)` for `a..=b`, in two's complement width. `range(1..=6)`, `range(0u8..=255)` and `range(0usize..n)` all work; an empty range panics, naming it (`Rng::range: 5..5 is empty; the end must be above the start`). A float range is `uniform(lo, hi)`.
- `normal` is the ziggurat of Marsaglia and Tsang with 256 layers in the form of Doornik (2005): one `next_u64()` gives the layer (its low 8 bits) and a signed uniform (its top 53 bits); the tables are those of `r = 3.654152885361009`. Nearly every deviate takes one draw.
- `exponential(rate)` is `-ln(u) / rate` with `u = ((next_u64() >> 12) + 0.5) * 2^-52` in (0, 1).
- `gamma(shape, scale)` is Marsaglia and Tsang (2000) (for `shape < 1`, the deviate of `shape + 1` times `u^(1 / shape)`), times `scale`; `beta(a, b)` is `x / (x + y)` of two gamma deviates.
- `poisson(lam)` searches the CDF from 0 when `lam < 10`, else uses PTRS (Hormann 1993), as NumPy; `binomial(n, p)` splits by beta-distributed order statistics while `n min(p, 1 - p) >= 30`, then inverts the CDF (Devroye 1986).
- `weighted(ws)` draws `t = f64() * sum` and returns the first index whose running sum passes `t`.
- `shuffle` is Fisher-Yates from the back (position `i` swaps with `below(i + 1)`); `choice` is `xs[below(len)]`; `sample` swaps position `i` with `i + below(len - i)` for `i` in `0..k` and takes the items in that order.

A parameter outside the distribution's domain panics with the method's name and what is allowed: a rate, shape or scale that is not above 0, a Poisson mean that is negative or not finite, `n < 0` or `p` outside [0, 1] for `binomial`, a negative or non-finite weight, or weights that sum to 0.

The batch forms fill many values with one call: `fill_f64` and `fill_normal` write a writable view (a `List<f64>` passes as a view of all of it), and `random_array`, `normal_array` and `uniform_array` return a fresh row-major `Array` of the shape, the values in row-major order (as one `f64()` / `normal()` per element would give them).

**`Xoshiro256PlusPlus`** (Blackman and Vigna; Julia's default) implements `Rng`. `Xoshiro256PlusPlus::seed(n: u64)` sets the four state words to four successive SplitMix64 outputs from `n`; the stream of a seed never changes, and a new algorithm gets a new type. `from_os()` seeds it from the OS source. `split(ref self) -> Xoshiro256PlusPlus` returns a copy of the generator and jumps the original 2^128 steps ahead (Vigna's `jump`), giving non-overlapping streams for parallel work that depend only on the seed and the order of `split` calls. It is not for secrets.

**`OsRng`** implements `Rng` over the OS source (`next_u64` reads 8 bytes, big-endian), never reproducible.

## 3.9.3. One-call forms

```nabi
fn rng() -> Xoshiro256PlusPlus;                          // Xoshiro256PlusPlus::from_os()
fn f64() -> f64;
fn range<R: SampleRange>(r: R) -> R::Item;
fn normal(mean: f64 = 0.0, std: f64 = 1.0) -> f64;
fn shuffle<T>(xs: List<T>);
fn choice<T>(xs: List<T>) -> T | none;
```

The one-call forms draw from one `Xoshiro256PlusPlus` for the process, seeded from the OS source on the first call and kept behind a lock, so a loop of `random::f64()` costs no system call per value (Python's `random.random()`, Go's `math/rand/v2`). They are fast but not for secrets: keys, tokens and nonces come from `bytes`, `secret` or `OsRng`. For a stream that repeats, seed a generator: `Xoshiro256PlusPlus::seed(n)`.

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
