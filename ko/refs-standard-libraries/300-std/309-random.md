# 3.9. `random::`: 난수 바이트, 생성기, 분포

## 3.9.1. `random`

```nabi
use std::random;

// std::random
fn fill(buf: SliceMut<u8>);      // 모든 바이트를 난수로 덮어씀
fn bytes(n: usize) -> List<u8>;  // n 개 난수 바이트
fn secret(n: usize) -> Secret;   // 비밀 저장소의 n 개 난수 바이트 (키, seed): 다른 사본 없음
```

`fill`은 저수준 형태입니다. 호출자가 가진 버퍼, 곧 그 버퍼의 쓰기 가능한 뷰([2.9.7](../200-core/209-collections.md))에 씁니다. `fill(buf)`는 `List<u8>` 전체를, `fill(buf[4:20])`은 그 바이트만 채웁니다. `bytes`와 `secret`은 버퍼를 직접 만들어 주는 편의 형태입니다.

바이트는 호출할 때마다 OS의 암호학적으로 안전한 생성기에서 읽습니다. 사용자 공간 풀이 없습니다. 그래서 fork로 만든 프로세스나 복원한 스냅숏이 `random`이 버퍼에 담아 둔 출력을 반복하지 않습니다.

| Target | 원천 |
|---|---|
| Linux (gnu, musl; x86-64, AArch64, i686, ARMv7) | 플래그 없는 `getrandom(2)`: 커널 풀이 초기화될 때까지 막고, 그 뒤로는 막지 않음 |
| macOS | `getentropy(2)`, 호출마다 최대 256바이트 |
| Windows | `BCRYPT_USE_SYSTEM_PREFERRED_RNG`를 준 `BCryptGenRandom` |
| wasm32-wasip2, wasm32-wasip1-threads | WASI `random_get` (호스트의 CSPRNG) |
| wasm32-unknown-unknown | JavaScript 호스트의 `crypto.getRandomValues` (호스트가 주는 `random_get` import로) |

중단된 호출은 다시 시도합니다. 원천의 실패는 복구할 수 없습니다. 이때 `fill`, `bytes`, `secret`은 예측 가능한 바이트를 돌려주지 않고 panic을 일으킵니다(Go의 `crypto/rand`와 같음). 메시지에는 호출한 함수와 OS 에러를 적습니다. `random::fill: the OS random source failed (error N)`(`random::bytes: ..`, `random::secret: ..`). 알고리듬은 `core::crypto`(2.21)에 있고, 무작위 입력(키, seed, nonce, salt)을 인자로 받습니다. `net::tls`(3.2.5)는 이 입력을 모두 여기서 얻습니다.

## 3.9.2. 생성기: `Rng`

```nabi
pub trait Rng {
    fn next_u64(ref self) -> u64;                        // 유일한 필수 메서드
    fn f64(ref self) -> f64;                             // [0, 1): (next_u64() >> 11) * 2^-53
    fn f32(ref self) -> f32;                             // [0, 1): (next_u64() >> 40) * 2^-24
    fn below(ref self, n: u64) -> u64;                   // [0, n), 치우침 없음. n == 0이면 패닉
    fn range<R: SampleRange>(ref self, r: R) -> R::Item; // 모든 정수 타입의 a..b / a..=b, 비면 패닉
    fn uniform(ref self, lo: f64, hi: f64) -> f64;       // lo + (hi - lo) * f64()
    fn bool(ref self, p: f64 = 0.5) -> bool;             // f64() < p
    fn normal(ref self, mean: f64 = 0.0, std: f64 = 1.0) -> f64;
    fn exponential(ref self, rate: f64 = 1.0) -> f64;
    fn gamma(ref self, shape: f64, scale: f64 = 1.0) -> f64;
    fn beta(ref self, a: f64, b: f64) -> f64;
    fn poisson(ref self, lam: f64) -> i64;
    fn binomial(ref self, n: i64, p: f64) -> i64;
    fn weighted(ref self, weights: List<f64>) -> usize;  // weights[i] / 합의 확률로 i
    fn shuffle<T>(ref self, xs: List<T>);                // 제자리
    fn choice<T>(ref self, xs: List<T>) -> T | none;
    fn sample<T>(ref self, xs: List<T>, k: usize) -> List<T>;

    fn fill_f64(ref self, out: SliceMut<f64>);           // 하나마다 f64()
    fn fill_normal(ref self, out: SliceMut<f64>, mean: f64 = 0.0, std: f64 = 1.0);
    fn random_array<const N: usize>(ref self, shape: [usize; N]) -> Array<f64, N>;
    fn normal_array<const N: usize>(ref self, shape: [usize; N], mean: f64 = 0.0, std: f64 = 1.0) -> Array<f64, N>;
    fn uniform_array<const N: usize>(ref self, shape: [usize; N], lo: f64, hi: f64) -> Array<f64, N>;
}

pub trait SampleRange {                                  // 모든 정수 타입의 Range<T>, RangeInclusive<T>
    type Item;
    fn outcomes(self) -> u64;                            // 범위의 값 수, 2^64개 전부면 0. 비면 패닉
    fn outcome(self, k: u64) -> Item;                    // 시작에서 k만큼 지난 값
}
```

파생 메서드는 이 절이 정하므로, 시드를 준 생성기는 모든 타깃과 릴리스에서 같은 값을 냅니다.

- `below(n)`은 `next_u64()`가 `(2^64 - n) mod n` 이상이 될 때까지 뽑고 그 값의 `mod n`을 돌려줍니다.
- `range(r)`은 `r.outcome(below(r.outcomes()))`이고, 2^64개 값 전부인 범위면 `r.outcome(next_u64())`입니다. 곧 `a..b`는 `a + below(b - a)`, `a..=b`는 `a + below(b - a + 1)`이며 2의 보수 폭으로 계산합니다. `range(1..=6)`, `range(0u8..=255)`, `range(0usize..n)`이 모두 됩니다. 빈 범위는 그 범위를 밝히며 패닉합니다(`Rng::range: 5..5 is empty; the end must be above the start`). 실수 범위는 `uniform(lo, hi)`입니다.
- `normal`은 Marsaglia와 Tsang의 ziggurat이며, 층 256개를 Doornik(2005)의 꼴로 둡니다. `next_u64()` 하나가 층(하위 8비트)과 부호 있는 균등값(상위 53비트)을 줍니다. 표는 `r = 3.654152885361009`의 것입니다. 거의 모든 편차가 한 번 뽑기로 끝납니다.
- `exponential(rate)`는 `u = ((next_u64() >> 12) + 0.5) * 2^-52`((0, 1) 안)로 `-ln(u) / rate`입니다.
- `gamma(shape, scale)`는 Marsaglia와 Tsang(2000)이고(`shape < 1`이면 `shape + 1`의 편차에 `u^(1 / shape)`를 곱함) 거기에 `scale`을 곱합니다. `beta(a, b)`는 감마 편차 두 개의 `x / (x + y)`입니다.
- `poisson(lam)`은 `lam < 10`이면 0부터 누적분포를 찾고, 그 밖에는 NumPy처럼 PTRS(Hormann 1993)를 씁니다. `binomial(n, p)`는 `n min(p, 1 - p) >= 30`인 동안 베타 분포를 따르는 순서 통계량으로 나누고, 그 뒤 누적분포를 역으로 찾습니다(Devroye 1986).
- `weighted(ws)`는 `t = f64() * 합`을 뽑고, 누적합이 `t`를 넘는 첫 인덱스를 돌려줍니다.
- `shuffle`은 뒤에서부터의 Fisher-Yates입니다(위치 `i`와 `below(i + 1)`을 바꿈). `choice`는 `xs[below(len)]`입니다. `sample`은 `0..k`의 `i`마다 위치 `i`와 `i + below(len - i)`를 바꾸고 그 순서로 항목을 가져옵니다.

분포의 정의역 밖 인수는 메서드 이름과 허용되는 값을 밝히며 패닉합니다. 0보다 크지 않은 rate, shape, scale, 음수이거나 유한하지 않은 Poisson 평균, `binomial`의 `n < 0`이나 [0, 1] 밖의 `p`, 음수이거나 유한하지 않은 가중치, 합이 0인 가중치가 그렇습니다.

일괄 형태는 한 번 호출로 값을 많이 채웁니다. `fill_f64`와 `fill_normal`은 쓰기 가능한 뷰에 쓰고(`List<f64>`는 전체의 뷰로 넘어감), `random_array`, `normal_array`, `uniform_array`는 그 모양의 새 행 우선 `Array`를 돌려줍니다. 값은 행 우선 순서이며, 원소마다 `f64()` / `normal()`을 한 번씩 부른 것과 같습니다.

**`Xoshiro256PlusPlus`**(Blackman, Vigna. Julia의 기본값)는 `Rng`를 구현합니다. `Xoshiro256PlusPlus::seed(n: u64)`는 `n`에서 시작한 SplitMix64 출력 네 개를 상태 워드로 씁니다. 한 시드의 수열은 바뀌지 않으며 새 알고리즘은 새 타입이 됩니다. `from_os()`는 OS 원천으로 시드를 정합니다. `split(ref self) -> Xoshiro256PlusPlus`는 생성기의 복사본을 돌려주고 원본을 2^128걸음 앞으로 건너뜁니다(Vigna의 `jump`). 그래서 병렬 작업용으로 겹치지 않는 수열을 주며, 수열은 시드와 `split` 호출 순서에만 달려 있습니다. 비밀 값에는 쓰지 않습니다.

**`OsRng`**는 OS 원천 위의 `Rng`입니다(`next_u64`는 8바이트를 빅 엔디언으로 읽음). 결코 재현되지 않습니다.

## 3.9.3. 한 번 호출 형태

```nabi
fn rng() -> Xoshiro256PlusPlus;                          // Xoshiro256PlusPlus::from_os()
fn f64() -> f64;
fn range<R: SampleRange>(r: R) -> R::Item;
fn normal(mean: f64 = 0.0, std: f64 = 1.0) -> f64;
fn shuffle<T>(xs: List<T>);
fn choice<T>(xs: List<T>) -> T | none;
```

한 번 호출 형태는 프로세스에 하나인 `Xoshiro256PlusPlus`에서 뽑습니다. 이 생성기는 첫 호출 때 OS 원천으로 시드를 정하고 잠금 뒤에 두므로, `random::f64()`를 반복문에서 불러도 값마다 시스템 호출이 들지 않습니다(Python의 `random.random()`, Go의 `math/rand/v2`). 빠르지만 비밀 값용은 아닙니다. 키, 토큰, nonce는 `bytes`, `secret`, `OsRng`에서 얻습니다. 되풀이되는 수열이 필요하면 생성기에 시드를 줍니다: `Xoshiro256PlusPlus::seed(n)`.

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
let quick = random::range(0..100);                       // 공유 생성기
```
