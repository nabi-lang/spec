# 2.1. 數值型別

`core::num` 模組定義 nabi 的所有數值型別。Nabi 讓兩個世界的數字並存，使用者可以自由選擇其中一個。

```
int / float    (auto-promote, arbitrary precision, opt-in)
fixed primitive (i8..i64, u8..u64, isize, usize, f32, f64; hardware-friendly, default)
```

兩個世界**並存**。沒有後綴的字面值依循預期型別；沒有預期型別時為 `i64` / `f64`（[refs-lang 1.6.3](../../refs-lang/001-lexical/106-numeric-literals.md#163-unsuffixed-literals)）。`int` / `float` 只用在明確寫出型別的地方。

## 2.1.1. 數字的兩個世界：為什麼分開

### `int` / `float`：自動提升的世界

這個世界以任意精度為目標。溢位永遠不會發生；只有在達到記憶體上限時，計算才會以 OOM 結束。小的值經過 small-int 最佳化，以硬體固定寬度的形式儲存；值變大時會自動提升為堆積上的 BigInt。`int` / `float` 的值不可變，堆積上的 BigInt 從不就地修改，只會共用（因此它們是 `Copy`）。使用者因而得到算術正確性的保證：`I32_MAX + 1` 絕不會默默環繞成荒謬的結果。

### 固定基本型別：貼近硬體的世界

相對地，固定基本型別是貼近硬體的世界。它包含 `i32` / `f64` 等 15 個型別；其語法由 [refs-lang 2.1 型別運算式](../../refs-lang/002-types/201-type-expressions.md)定義。它們適合需要精確控制記憶體與效能的情境（FFI 介面、SIMD、大型陣列、熱迴圈），而且溢位策略是明確的（請見 [2.3 溢位行為](203-ops.md#溢位行為)）。

### 哪一個是預設

沒有後綴的字面值（`42`、`3.14`）在沒有任何東西決定其型別時為 **`i64` / `f64`**（以 `let` 繫結的字面值從之後的使用取得型別，[1.6.3](../../refs-lang/001-lexical/106-numeric-literals.md#163-unsuffixed-literals)）。只有明確寫出 `int` / `float` 的地方才使用自動提升的世界。

```nabi
let total: int = 0;               // int (auto-promote), explicit
for x in xs { total = total + x; }
// total is auto-promoted even past the i64 range; no silent overflow

let n = 0;                        // i64 (nothing fixes it)
let small: i32 = 0;               // i32: follows the expected type
```

## 2.1.2. `int`：任意精度整數

```nabi
#[keyword("int")]
#[lang("int_type")]
struct Int { ... }  // internal representation: small-int or BigInt

impl Int {
    fn from_i32(n: i32) -> int;
    fn from_i64(n: i64) -> int;
    fn from_u64(n: u64) -> int;
    fn parse_radix(s: str, radix: u32) throws (ParseError) -> int;  // radix parsing (base 10 is FromStr); panics outside 2..=36
    fn to_str_radix(self, radix: u32) -> str;   // radix 2..=36, lowercase digits, `-` first; panics on another radix

    fn to_i32(self) -> i32 | none;  // the value if in range, none otherwise
    fn to_i64(self) -> i64 | none;
    fn to_u64(self) -> u64 | none;  // none if negative or past u64::MAX
    fn to_f64(self) -> f64;        // nearest f64, ties to even; ±inf past f64::MAX

    fn is_zero(self) -> bool;
    fn is_negative(self) -> bool;
    fn is_positive(self) -> bool;               // false for 0
    fn signum(self) -> int;                     // -1, 0 or 1
    fn abs(self) -> int;
    fn bit_length(self) -> u64;                 // bits of |self| without leading zeros; 0 for 0

    // bits: two's complement, the sign bit repeated forever
    fn count_ones(self) -> u64;                 // one bits of |self|
    fn trailing_zeros(self) -> u64 | none;      // none for 0
    fn bit(self, n: u64) -> bool;               // a negative number has every high bit set
    fn set_bit(ref self, n: u64, value: bool);
    fn is_power_of_two(self) -> bool;           // false for 0 and negatives
    fn next_power_of_two(self) -> int;          // smallest power of two >= self; 1 for self <= 1
    fn ilog2(self) -> u64;                      // floor(log2(self)); panics unless self > 0
    fn checked_ilog2(self) -> u64 | none;

    // bytes: the shortest two's complement; *_unsigned is the magnitude
    fn to_le_bytes(self) -> List<u8>;
    fn to_be_bytes(self) -> List<u8>;
    fn to_le_bytes_unsigned(self) -> List<u8>;    // panics if self < 0
    fn to_be_bytes_unsigned(self) -> List<u8>;
    fn byte_len(self) -> usize;                   // to_le_bytes().len()
    fn byte_len_unsigned(self) -> usize;          // panics if self < 0
    fn write_le_bytes(self, out: SliceMut<u8>);            // exactly out.len() bytes, sign-extended
    fn write_be_bytes(self, out: SliceMut<u8>);
    fn write_le_bytes_unsigned(self, out: SliceMut<u8>);   // zero-extended; panics if self < 0
    fn write_be_bytes_unsigned(self, out: SliceMut<u8>);
    fn from_le_bytes<B: AsBytes>(bytes: B) -> int;            // no bytes is 0
    fn from_be_bytes<B: AsBytes>(bytes: B) -> int;
    fn from_le_bytes_unsigned<B: AsBytes>(bytes: B) -> int;   // never negative
    fn from_be_bytes_unsigned<B: AsBytes>(bytes: B) -> int;

    fn div_rem(self, rhs: int) -> (int, int);   // (self / rhs, self % rhs), truncating; panics if rhs == 0
    fn pow(self, exp: u32) -> int;              // exp: u32 as for fixed-width integers
    fn isqrt(self) -> int;                      // floor(sqrt(self)); panics if self < 0
    fn checked_isqrt(self) -> int | none;       // none if self < 0
    fn div_euclid(self, rhs: int) -> int;       // the quotient for which rem_euclid >= 0; panics if rhs == 0
    fn rem_euclid(self, rhs: int) -> int;       // in 0..rhs.abs(); panics if rhs == 0
    fn modpow(self, exp: int, m: int) -> int;   // self^exp %% m; panics if m == 0 or exp < 0
    fn gcd(self, other: int) -> int;            // never negative; gcd(0, 0) == 0
    fn lcm(self, other: int) -> int;            // never negative; 0 if either operand is 0
}
```

這些方法沿用固定寬度整數的名稱與 panic。位元數與位元索引是 `u64`，和 `bit_length` 一樣：一個值可以有超過 2^32 個位元（固定寬度的規則見下文）。依附於寬度的位元方法（`leading_zeros`、`rotate_left`、`swap_bytes` 等）在沒有寬度時沒有意義，因此不提供。`count_ones` 計算的是 `|self|`（即 Python 的 `bit_count`），因為負數有無限多個 1 位元。表示法寫在名稱裡：`to_le_bytes` / `from_le_bytes` 使用二補數，與固定寬度整數的位元組一致；`_unsigned` 形式使用絕對值，遇到負值會 panic（`Int::to_le_bytes_unsigned: negative value; use to_le_bytes for two's complement or call .abs() first`）。位元組長度是值所允許的最短長度：`128` 是 `[128, 0]`，無號形式為 `[128]`；`-1` 是 `[255]`；零是 `[0]`。`write_*` 形式則填滿固定寬度，也就是 `out`（一個 `List<u8>` 或其檢視）的每個位元組：`-2` 以四個位元組、小端序寫出是 `[254, 255, 255, 255]`；值需要的位元組比 `out` 多時會 panic（`Int::write_le_bytes: the value needs 2 bytes but `out` has 1; pass a view of at least 2 bytes`）。`from_*` 形式接受任何位元組（[`AsBytes`](211-conversion.md)）：`List<u8>`、`Slice<u8>`、`[u8; N]` 或 `str`。`pow` 接受 `u32` 指數：更大的指數對 0 與 ±1 以外的任何底數，都會產生至少 2^32 個位元。`modpow` 接受 `int` 指數，因為其結果保持在 `|m|` 以下；結果的正負號與 `m` 相同，和 [`%%`](203-ops.md#modulo向下取整模數) 一樣。負指數是錯誤（沒有模反元素）。零是 `Int::default()`（或 `Int::zero()`，請見 2.1.5 的 `Zero`）；`Int` 沒有 `new`。除以零時，無論來自運算子 `/`、`%` 與 `%%`，或來自 `div_rem`、`div_euclid` 與 `rem_euclid`，都會以 `attempt to divide an int by zero` 觸發 panic。帶檢查的 `to_i32` / `to_i64` / `to_u64` 是與 `TryFrom` 並列的固有方法（[0.5.11](../000-meta/005-notation.md)）。

`Int` 定義在 `core::num::bigint::Int`（`Float` 在 `core::num::bigfloat::Float`）。`#[keyword("int")]` 將識別字 `int` 註冊為型別關鍵字，此註冊適用於所有套件。因此使用者程式碼將型別寫成小寫的 `int`，而不是 PascalCase 的 `Int`（以 [refs-lang 6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) 為準）。編譯器透過 `#[lang("int_type")]` 找到 `int` 型別。

`int` 的主要性質如下。在記憶體中，小的值經過 small-int 最佳化，直接存放在一個 64 位元字組中，大的值則自動提升為堆積上的 BigInt 表示；這一切對使用者都是透明的。表示法永遠是正規的：在 i64 範圍內的值為小型形式，否則為 BigInt；算術結果回到 i64 範圍時，也會回到小型形式。相等的值有相同的表示。使用的 **lang 槽位**是 `int_type`（[0.4 lang 槽位表](../000-meta/004-lang-slot-table.md)）。沒有後綴的整數字面值只有在預期型別為 `int` 時才是 `int`。

### 算術

`int` 之間的算術絕不環繞，也不飽和。

```nabi
let big: int = 1;
big = big * 1000;
big = big * 1000;
big = big * 1000;
big = big * 1000;
big = big * 1000;
big = big * 1000;
// big = 10^18, near the end of the i64 range
big = big * 1000;
// big = 10^21, outside the i64 range, promoted to BigInt
big = big * big;
// big = 10^42, a larger BigInt, exact
```

`int` 實作算術 trait（[2.3 ops](203-ops.md)）`Add`、`Sub`、`Mul`、`Div`、`Rem`、`Modulo` 與 `Neg`。`Div` 是整數除法（截斷），因此 `7 / 2 = 3`，`-7 / 2 = -3`。位元 trait `BitAnd`、`BitOr`、`BitXor` 與 `BitNot` 作用於符號位元無限重複的二補數（`~x == -x - 1`）。位移是 `Shl<u32>` 與 `Shr<u32>`：右側是 `u32`，不是 `int`；`x << n == x * 2^n`，`x >> n == floor(x / 2^n)`（算術位移）。比較 trait（[2.5 cmp](205-cmp.md)）`PartialEq`、`Eq`、`PartialOrd`、`Ord` 與 `Hash` 均有實作，`Clone`、`Copy`、`Default`（零）、`Zero`、`One`、`Display`、`Debug` 與 `FromStr` 亦然。

### 主要用途

`int` 適用於許多情境。作為 REPL 或 Jupyter 中的算術預設，它不必每次寫型別標註就能保證正確性；數值模擬中累加的和或積不必擔心溢位。密碼學運算與大整數處理只靠標準函式庫就能完成，不必相依於另外的 BigInt 函式庫。

### 限制

`int` 有其限制。在記憶體方面，提升為 BigInt 需要堆積配置，因此在熱迴圈與大型陣列中建議使用固定基本型別。在效能方面，它在 small-int 範圍內接近固定基本型別，但提升為 BigInt 之後必然較慢。在 FFI 方面，`extern "C"` 函式只接受固定基本型別，因此 `int` 必須先明確轉換為 `i64` 等型別才能傳遞。

## 2.1.3. `float`：指數無上限的浮點數

```nabi
#[keyword("float")]
#[lang("float_type")]
struct Float { ... }  // internal: f64 or BigFloat (when the exponent is outside f64)

impl Float {
    fn from_f32(x: f32) -> float;
    fn from_f64(x: f64) -> float;
    fn from_int(n: int) -> float;                  // nearest float, ties to even

    fn to_f32(self) -> f32 | none;     // none only when a finite value is outside the f32 range
    fn to_f64(self) -> f64;            // nearest f64; ±inf past f64::MAX, ±0 below the subnormals
    fn to_int(self) -> int | none;     // truncated toward zero; none for NaN / ±inf

    fn is_nan(self) -> bool;
    fn is_infinite(self) -> bool;
    fn is_finite(self) -> bool;
    fn is_zero(self) -> bool;
    fn is_sign_negative(self) -> bool;
    fn abs(self) -> float;
}
```

`#[keyword("float")]` 註冊 `float` 關鍵字，並將它導向 struct `Float`。因此使用者程式碼只會看到小寫的 `float`。

`float` 值具有 53 位元的有效數字（與 f64 精度相同）以及無上限（i64）的指數。它平常使用 f64 表示；當有限的結果超出 f64 範圍（溢位）或太接近 0（下溢）時，會提升為 BigFloat 表示，而不是變成 `inf` / `0`。

- `inf` / `NaN` 只來自除以零，或來自 `inf` / `NaN` 運算元。
- `%` 是精確的 fmod。
- `Display` 對 f64 範圍內的值與 f64 寫法相同；提升後的值以指數寫出，如 `1.5e400`（`{:g}` 得到 `1.5e+400`）。

### 明示的限制

`float` 的限制必須清楚說明。它的精度是 53 位元，不是無上限，因此與 IEEE 754 f64 有相同的捨入誤差。例如 `0.1 + 0.2` 不等於 `0.3`，在 Nabi 中也一樣。`Eq` 也沒有實作：為了處理 NaN，`float` 不實作 `Eq` 標記 trait，只實作 `PartialEq`。完整的階層請見 [2.5 cmp](205-cmp.md)。

### lang 槽位

編譯器透過 `float_type` 找到 `float` 型別（[0.4 lang 槽位表](../000-meta/004-lang-slot-table.md)）。沒有後綴的浮點數字面值只有在預期型別為 `float` 時才是 `float`。

## 2.1.4. 固定寬度基本型別

[refs-lang 2.1 型別運算式](../../refs-lang/002-types/201-type-expressions.md)定義 15 個基本型別的**名稱**。本節在此之上定義標準函式庫的方法與行為。

### 整數（10 個型別）

| 型別    | 位元寬度                        | 範圍           |
|---------|---------------------------------|----------------|
| `i8`    | 8                               | -128 ~ 127     |
| `i16`   | 16                              | -32768 ~ 32767 |
| `i32`   | 32                              | -2³¹ ~ 2³¹-1   |
| `i64`   | 64                              | -2⁶³ ~ 2⁶³-1   |
| `isize` | 目標的指標寬度（32 或 64）      | 依目標而定     |
| `u8`    | 8                               | 0 ~ 255        |
| `u16`   | 16                              | 0 ~ 65535      |
| `u32`   | 32                              | 0 ~ 2³²-1      |
| `u64`   | 64                              | 0 ~ 2⁶⁴-1      |
| `usize` | 目標的指標寬度                  | 依目標而定     |

### 浮點數（2 個型別）

| 型別  | 位元寬度  | 標準                                 |
|-------|-----------|--------------------------------------|
| `f32` | 32        | IEEE 754 binary32（單精度）          |
| `f64` | 64        | IEEE 754 binary64（雙精度）          |

16 位元浮點數 `f16` 與 `bf16` 是標準函式庫為儲存而新增的型別（[見下文](#f16-與-bf1616-位元儲存用浮點數)）。

### 浮點數語意

`f32` 與 `f64` 在每個目標上的行為都相同：

- **捨入。**`+ - * /`、`%`（精確，即 C 的 `fmod`）、`√` / `sqrt`、`mul_add`、每個轉成浮點數的 `as` 轉換、每個浮點數字面值以及 `parse`，都捨入到最近值，平手時取偶數。字面值或剖析的字串從其十進位值只捨入一次：`0.1f32` 是最接近 0.1 的 `f32`，而不是將 `0.1f64` 再捨入一次。沒有其他捨入模式，也無法變更。
- **不縮併。**`a * b + c` 是先捨入乘積，再捨入總和。只有 `mul_add` 將兩者融合為一次捨入。最佳化器與任何後端都不會縮併、重新結合或重新排列浮點數運算：加總浮點數的迴圈不會向量化（線道會改變順序），而常數求值得到的位元與程式執行時計算的相同。
- **保留次正規數。**沒有任何目標會將次正規數的輸入或結果清為零。執行環境在每個目標上都將浮點數控制狀態（MXCSR、FPCR、FPSCR）保持在 IEEE 預設值，Nabi 程式碼無法變更它。
- **NaN。**與 NaN 的每個比較都為 false，`!=` 除外。運算回傳哪一個 NaN 未指定：其酬載與正負號在不同目標上可能不同（x86 讓 `0.0 / 0.0` 為負）。`-x`、`abs` 與 `copysign` 只改變符號位元，對 NaN 亦然，而 `to_bits` / `from_bits` 保留每個位元。`f32::NAN` 與 `f64::NAN` 是符號位元清除的固定 quiet NaN。
- **有號零。**`-0.0 == 0.0`，而 `1.0 / -0.0` 是 `-inf`。
- **`min` / `max`** 是 IEEE 754 的 `minimumNumber` / `maximumNumber`：一個運算元為 NaN 時得到另一個（只有兩者都是 NaN 時才得到 NaN），且 `-0.0` 排在 `+0.0` 之前，因此在每個目標上 `(0.0).min(-0.0)` 都是 `-0.0`，`(-0.0).max(0.0)` 都是 `0.0`。`total_cmp` 是 IEEE 全序（[2.1.11](#2111-數值-trait浮點數排序範圍複數)）。
- **浮點數轉整數**的 `as` 向零截斷，並在整數型別的界限處飽和；NaN 得到 0。整數轉浮點數捨入到最近值，平手時取偶數。
- **函式庫函式。**`sin`、`exp`、`ln`、`pow` 與其他超越函式使用目標的 C 函式庫：不保證正確捨入，最後一個位元在不同目標上可能不同。常數求值不會將它們摺疊。

`float`（2.1.3）在 `f64` 的範圍內遵循相同的規則。

### `f16` 與 `bf16`：16 位元儲存用浮點數

```nabi
#[lang("f16_type")]
#[keyword("f16")]
pub struct F16 { bits: u16; }      // IEEE 754 binary16

#[lang("bf16_type")]
#[keyword("bf16")]
pub struct BF16 { bits: u16; }     // bfloat16
```

| 型別   | 有效數字位元     | 指數位元      | 最大值    | 最小正規值      | `EPSILON` |
|--------|------------------|---------------|-----------|-----------------|-----------|
| `f16`  | 11（儲存 10）    | 5             | 65504     | 2^-14           | 2^-10     |
| `bf16` | 8（儲存 7）      | 8（與 `f32` 相同） | 約 3.39e38 | 2^-126      | 2^-7      |

`f16` 與 `bf16`（`core::num::half`）是儲存用型別：它們讓大型陣列（模型權重、影像、感測器資料）的記憶體減半，並透過 `f32` 進行計算。兩者都是 2 個位元組、對齊為 2，`List<f16>` 或 `Array<bf16, 2>` 每個元素存放 2 個位元組，`Array::zeros` / `ones` 也適用於它們。

- **字面值。**位於 `f16` / `bf16` 槽位中的無後綴字面值，或帶有後綴 `f16` / `bf16` 的字面值（`0.1f16`、`2bf16`），其值是從十進位只捨入一次的結果。捨入成無窮大的字面值是編譯錯誤（`let h: f16 = 70000.0;`）；整數字面值必須精確（`let h: f16 = 2049;` 是錯誤，與 `f32` 相同）。
- **`as`** 可在 16 位元浮點數與每個整數、浮點數基本型別之間互轉，也可在 `f16` 與 `bf16` 之間互轉。轉成 16 位元浮點數時，直接從來源捨入到最近值，平手時取偶數：`f64` 或 `i64` 絕不會先捨入成 `f32`。從它轉出時，轉成 `f32` / `f64` 是精確的，轉成整數則等同其 `f32` 值的 `as`。`int`、`float`、`bool` 與 `char` 不能用 `as` 轉換（請寫 `x.to_f64() as f16`）。
- **算術。**`+ - * / % %%`、其複合指派、一元 `-` 與 `√` 都以 `f32` 計算，再對該型別捨入一次。對 `+ - * / %` 與 `√` 而言，這正好是 16 位元格式下正確捨入的 IEEE 結果（`f32` 的有效數字位元多於其兩倍再加二）。混合運算元與基本型別一樣會被拒絕（2.1.6）。沒有超越函式方法，也沒有 `FloatingPoint` 實作：請轉成 `f32`（`h.to_f32().sin()`）。
- **比較**（`==`、`<` 等）以 `f32` 值進行：IEEE 語意，NaN 與任何值都不相等，`-0.0 == 0.0`。兩個型別都不是 `Eq`、`Ord` 或 `Hash`；排序請用 `xs.sort_by(f16::total_cmp)`。字面值模式不會比對它們：請在守衛中用 `==` 比較。
- **方法與常數。**`from_bits` / `to_bits`（`u16` 位元樣式）、`from_f32`、`from_f64`、`from_i64`、`from_u64`、`to_f32`、`to_f64`、`is_nan`、`is_infinite`、`is_finite`、`is_normal`、`is_sign_positive`、`is_sign_negative`、`abs`、`copysign`、`min`、`max`（與 `f32` 的相同）、`total_cmp`；`EPSILON`、`MAX`、`MIN`、`MIN_POSITIVE`、`NAN`、`INFINITY`、`NEG_INFINITY`。
- **trait。**`Clone`、`Copy`、`Default`（`0.0`）、`Zero`、`One`、`PartialEq`、`PartialOrd`、上述運算子 trait、`Display`、`Debug`、`FromStr`（從十進位捨入一次），以及 `f32` 與 `f64` 的 `From<f16>` / `From<bf16>`（精確）。
- **文字。**`Display` 寫出能讀回相同值的最短位數（`0.1`、`65500`），絕不使用指數，也寫出 `inf` / `NaN`；`Debug` 對整數值加上 `.0`。格式規範（`{h:.3}`、`{h:e}`）與 `f64` 一樣格式化精確值。
- **硬體。**`f16` <-> `f32` 在 CPU 支援轉換的地方使用 CPU：x86-64 上當目標特性包含 `f16c` 時使用 F16C（`-C target-cpu=x86-64-v3`），arm64 上使用 `fcvt`。其他地方則是執行環境呼叫，結果相同。`bf16` 的轉換是少數幾個內嵌的整數運算。
- **C。**在 `extern "C"` 簽章中，`f16` 是 C 的 `_Float16`，`bf16` 是 `__bf16`，以 C 的方式透過浮點數暫存器傳遞（[refs-lang 8.6.3](../../refs-lang/008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)）。目標的 C 沒有這種型別時（powerpc64le 與 wasm32；s390x 上的 `__bf16`；i686 上的半精度回傳值），該簽章是錯誤：請改以 `uint16_t` 傳遞 `h.to_bits()`。

### 布林值 / 字元（3 個型別）

| 型別     | 意義                                                                                   |
|----------|----------------------------------------------------------------------------------------|
| `bool`   | `true` / `false`                                                                       |
| `char`   | Unicode 純量值（4 個位元組），編譯器基本型別。方法見 [2.15](215-string.md)            |
| `c_char` | 與 C ABI 相容的 char（1 個位元組，正負號依目標而定，以其位元組 `c as u8` 比較）；僅用於 FFI |

### 兩個群組：語言內建與標準函式庫新增

本章的數值 / 布林值 / 字元型別分為**兩個群組**。

| 群組                                 | 成員                                                                                                  | 註冊方式                                           |
|--------------------------------------|-------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| **16 個語言內建**                    | `i8`/`i16`/`i32`/`i64`、`u8`/`u16`/`u32`/`u64`、`isize`/`usize`、`f32`/`f64`、`bool`、`char`、`c_char`、`str` | 由語言辨識；不需要 `#[keyword]`                    |
| **4 個標準函式庫新增**               | `int`、`float`、`f16`、`bf16`                                                                         | 以 `#[keyword(...)]` 註冊                          |

語言內建是編譯器（詞法分析器／剖析器）直接辨識的語彙單元，因此標準函式庫不將它們註冊為關鍵字。這自動保證使用者無法重新定義這些名稱。

相對地，標準函式庫新增的型別（`int` / `float` / `f16` / `bf16`）以 `#[keyword(...)]` 註冊。`#[keyword]` 屬性附加在本模組中 `Int` / `Float` / `F16` / `BF16` 的定義上（以 [refs-lang 6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) 為準）。

### 內建基本型別在標準函式庫中的形式

```nabi
impl i32 { ... }              // inherent constants and methods (below)
impl Add for i32 { ... }      // trait impls: Add, PartialOrd, Display, Clone, ...
impl PartialEq for c_char { ... }
```

標準函式庫不為內建基本型別宣告任何 struct。它只附加用於固有方法與 trait（`Add`、`Display`、`Clone` 等）的 `impl` 區塊。辨識型別本身是編譯器的責任。

### 算術 / 比較 trait 實作

每個基本型別都實作 [2.3 ops](203-ops.md)、[2.4 ops-assign](204-ops-assign.md) 與 [2.5 cmp](205-cmp.md) 的 trait。每個運算子方法以及每個 `PartialEq` / `PartialOrd` 方法（包括 `char` 與 `c_char` 的）都帶有 `#[intrinsic("nabi.OP.TYPE")]`，將其直接繫結到編譯器後端的 PrimOp。唯一的例外是 `bool` 的 `PartialOrd`，它是一般的函式主體（`false < true`）。`c_char` 在每個目標上都以其位元組比較（`c as u8`，2.15.5）：`255u8 as c_char > 1u8 as c_char`。

### 方法（各基本型別）

每個整數 / 浮點數型別都提供下列方法（範例為 `i32`）。

```nabi
impl i32 {
    const MIN: i32 = -2147483648;
    const MAX: i32 = 2147483647;
    const BITS: u32 = 32;

    fn pow(self, exp: u32) -> i32;                // overflow follows the arithmetic (panic; wrap with overflow checks off)
    fn abs(self) -> i32;
    fn unsigned_abs(self) -> u32;                 // |self| without overflow (i32::MIN gives 2^31)
    fn abs_diff(self, rhs: i32) -> u32;
    fn signum(self) -> i32;                       // -1, 0 or 1
    fn is_positive(self) -> bool;
    fn is_negative(self) -> bool;
    fn div_euclid(self, rhs: i32) -> i32;         // the quotient for which rem_euclid >= 0
    fn rem_euclid(self, rhs: i32) -> i32;         // in 0..rhs.abs()
    fn isqrt(self) -> i32;                        // floor(sqrt(self)); panics if self < 0
    fn checked_isqrt(self) -> i32 | none;         // none if self < 0
    fn ilog2(self) -> u32;                        // floor(log2(self)); panics unless self > 0
    fn ilog10(self) -> u32;
    fn ilog(self, base: i32) -> u32;              // panics unless self > 0 and base >= 2
    fn checked_ilog2(self) -> u32 | none;
    fn checked_ilog10(self) -> u32 | none;
    fn checked_ilog(self, base: i32) -> u32 | none;

    // bits
    fn count_ones(self) -> u32;
    fn count_zeros(self) -> u32;
    fn leading_zeros(self) -> u32;
    fn trailing_zeros(self) -> u32;
    fn leading_ones(self) -> u32;
    fn trailing_ones(self) -> u32;
    fn rotate_left(self, n: u32) -> i32;
    fn rotate_right(self, n: u32) -> i32;

    // byte order
    fn swap_bytes(self) -> i32;
    fn to_be(self) -> i32;                        // big-endian from native order
    fn to_le(self) -> i32;
    fn from_be(x: i32) -> i32;
    fn from_le(x: i32) -> i32;
    fn to_le_bytes(self) -> [u8; 4];
    fn to_be_bytes(self) -> [u8; 4];
    fn to_ne_bytes(self) -> [u8; 4];              // native order
    fn from_le_bytes(bytes: [u8; 4]) -> i32;
    fn from_be_bytes(bytes: [u8; 4]) -> i32;
    fn from_ne_bytes(bytes: [u8; 4]) -> i32;

    // explicit-overflow variants
    fn checked_add(self, rhs: i32) -> i32 | none;
    fn checked_sub(self, rhs: i32) -> i32 | none;
    fn checked_mul(self, rhs: i32) -> i32 | none;
    fn checked_div(self, rhs: i32) -> i32 | none; // none for rhs == 0 and for MIN / -1
    fn checked_rem(self, rhs: i32) -> i32 | none;
    fn checked_neg(self) -> i32 | none;
    fn checked_shl(self, n: u32) -> i32 | none;   // none if n >= BITS
    fn checked_shr(self, n: u32) -> i32 | none;
    fn checked_pow(self, exp: u32) -> i32 | none;
    fn checked_abs(self) -> i32 | none;

    fn saturating_add(self, rhs: i32) -> i32;     // clamp to i32::MAX or MIN
    fn saturating_sub(self, rhs: i32) -> i32;
    fn saturating_mul(self, rhs: i32) -> i32;
    fn saturating_neg(self) -> i32;
    fn saturating_pow(self, exp: u32) -> i32;
    fn saturating_abs(self) -> i32;

    fn wrapping_add(self, rhs: i32) -> i32;       // modular arithmetic
    fn wrapping_sub(self, rhs: i32) -> i32;
    fn wrapping_mul(self, rhs: i32) -> i32;
    fn wrapping_div(self, rhs: i32) -> i32;       // MIN / -1 is MIN; rhs == 0 still panics
    fn wrapping_rem(self, rhs: i32) -> i32;       // MIN % -1 is 0
    fn wrapping_neg(self) -> i32;
    fn wrapping_shl(self, n: u32) -> i32;         // shift by n % BITS
    fn wrapping_shr(self, n: u32) -> i32;
    fn wrapping_pow(self, exp: u32) -> i32;
    fn wrapping_abs(self) -> i32;

    // the wrapped result and whether it overflowed
    fn overflowing_add(self, rhs: i32) -> (i32, bool);
    fn overflowing_sub(self, rhs: i32) -> (i32, bool);
    fn overflowing_mul(self, rhs: i32) -> (i32, bool);
    fn overflowing_div(self, rhs: i32) -> (i32, bool);
    fn overflowing_rem(self, rhs: i32) -> (i32, bool);
    fn overflowing_neg(self) -> (i32, bool);
    fn overflowing_shl(self, n: u32) -> (i32, bool);  // true if n >= BITS
    fn overflowing_shr(self, n: u32) -> (i32, bool);
    fn overflowing_pow(self, exp: u32) -> (i32, bool);
}
```

`i8`、`i16`、`i64` 與 `isize` 以各自的寬度擁有相同的方法集。無號型別（`u8`..`u64`、`usize`）擁有相同的方法集，但不包括與正負號相關的方法（`abs`、`unsigned_abs`、`signum`、`is_positive`、`is_negative`、`checked_isqrt`、`checked_abs`、`saturating_neg`、`saturating_abs`、`wrapping_abs`）；`abs_diff` 回傳相同型別，而 `checked_neg` 對 0 以外的每個值都是 `none`。它們另外提供 2 的冪相關方法。

位元數或位元索引的型別與 `BITS` 及位移量相同：每個固定寬度整數都是 `u32`（`count_ones`、`leading_zeros`、`trailing_zeros`、`ilog2`、`rotate_left(n: u32)`、`pow(exp: u32)`、`wrapping_shl(n: u32)`），與整數本身的寬度無關；`trailing_zeros(0)` 是 `BITS`。`int` 則改用 `u64`，因為一個值可以超過 2^32 個位元，且 0 的 `trailing_zeros` 是 `none`。`<<` / `>>` 運算子則不同：在固定寬度整數上，右側與左側型別相同（`Shl for T`，因此是 `i64 << i64`）；在 `int` 上則是 `u32`（`Shl<u32>`）。

```nabi
impl u32 {
    fn is_power_of_two(self) -> bool;
    fn next_power_of_two(self) -> u32;                 // smallest power of two >= self; panics on overflow
    fn checked_next_power_of_two(self) -> u32 | none;
}
```

整數型別的最小值 / 最大值是關聯常數 `i32::MIN` / `i32::MAX`，不是函式。沒有引數的固有方法 `min()` / `max()` 會遮蔽同名的 `Ord::min` / `Ord::max`（`a.min(b)`）。

浮點數型別（`f32`、`f64`）提供下列常數與方法（`f32` 擁有相同的集合，只是以 `f32` 取代 `f64`，位元為 `u32`）。

```nabi
impl f64 {
    const EPSILON: f64;          // 2^-52, the gap between 1.0 and the next f64
    const MAX: f64;              // largest finite value
    const MIN: f64;              // -MAX
    const MIN_POSITIVE: f64;     // smallest positive normal value
    const NAN: f64;              // quiet NaN, sign bit clear: from_bits(0x7ff8_0000_0000_0000)
    const INFINITY: f64;
    const NEG_INFINITY: f64;
    const PI: f64;
    const TAU: f64;              // 2π
    const E: f64;
    const FRAC_PI_2: f64;        // π/2
    const FRAC_PI_4: f64;        // π/4
    const FRAC_1_PI: f64;        // 1/π
    const SQRT_2: f64;
    const FRAC_1_SQRT_2: f64;
    const LN_2: f64;
    const LN_10: f64;
    const LOG2_E: f64;
    const LOG10_E: f64;

    fn is_nan(self) -> bool;
    fn is_infinite(self) -> bool;
    fn is_finite(self) -> bool;
    fn is_normal(self) -> bool;  // neither zero, subnormal, infinite nor NaN
    fn is_sign_positive(self) -> bool;
    fn is_sign_negative(self) -> bool;

    fn floor(self) -> f64;
    fn ceil(self) -> f64;
    fn round(self) -> f64;       // halves away from zero
    fn trunc(self) -> f64;
    fn fract(self) -> f64;       // self - self.trunc()

    fn abs(self) -> f64;
    fn signum(self) -> f64;      // 1.0 if positive, -1.0 if negative, that zero if ±0.0, NaN if NaN
    fn copysign(self, sign: f64) -> f64;
    fn recip(self) -> f64;       // 1.0 / self
    fn min(self, other: f64) -> f64;     // a NaN operand loses to a number
    fn max(self, other: f64) -> f64;
    fn clamp(self, lo: f64, hi: f64) -> f64;   // NaN stays NaN; panics if lo > hi
    fn div_euclid(self, rhs: f64) -> f64;
    fn rem_euclid(self, rhs: f64) -> f64;
    fn mul_add(self, a: f64, b: f64) -> f64;   // self * a + b with one rounding
    fn lerp(self, other: f64, t: f64) -> f64;  // self at t = 0, other at t = 1; exact at both, monotonic in t
    fn next_after(self, toward: f64) -> f64;   // one step toward `toward` (C nextafter); toward if equal
    fn next_up(self) -> f64;                   // least value > self; inf and NaN stay
    fn next_down(self) -> f64;                 // greatest value < self

    fn sqrt(self) -> f64;        // NaN for a negative value; forwards to the Sqrt trait (2.3)
    fn cbrt(self) -> f64;
    fn hypot(self, other: f64) -> f64;   // sqrt(self² + other²) without overflow
    fn pow(self, exp: f64) -> f64;
    fn powi(self, n: i32) -> f64;        // repeated multiplication: may round unlike pow
    fn exp(self) -> f64;
    fn exp2(self) -> f64;
    fn exp_m1(self) -> f64;      // e^self - 1, exact near zero
    fn ln(self) -> f64;
    fn ln_1p(self) -> f64;       // ln(1 + self), exact near zero
    fn log(self, base: f64) -> f64;
    fn log2(self) -> f64;
    fn log10(self) -> f64;
    fn sin(self) -> f64;
    fn cos(self) -> f64;
    fn tan(self) -> f64;
    fn asin(self) -> f64;
    fn acos(self) -> f64;
    fn atan(self) -> f64;
    fn atan2(self, other: f64) -> f64;   // angle of the point (other, self)
    fn sinh(self) -> f64;
    fn cosh(self) -> f64;
    fn tanh(self) -> f64;
    fn asinh(self) -> f64;
    fn acosh(self) -> f64;
    fn atanh(self) -> f64;
    fn to_degrees(self) -> f64;
    fn to_radians(self) -> f64;

    fn to_bits(self) -> u64;     // bit pattern
    fn from_bits(bits: u64) -> f64;
}
```

## 2.1.5. 字面值 → 哪個型別？

[refs-lang 1.6 數值字面值](../../refs-lang/001-lexical/106-numeric-literals.md)的規則如下：

| 形式      | 結果型別                                                |
|-----------|---------------------------------------------------------|
| `42`      | 預期型別（整數 / 浮點數型別），否則為 `i64`             |
| `3.14`    | 預期型別（浮點數型別），否則為 `f64`                    |
| `42i32`   | `i32`                                                   |
| `42u64`   | `u64`                                                   |
| `3.14f32` | `f32`                                                   |
| `3.14f64` | `f64`                                                   |
| `0.5f16`  | `f16`                                                   |
| `0.5bf16` | `bf16`                                                  |
| `'A'`     | `char`                                                  |
| `c'A'`    | `c_char`（基本型別）                                    |
| `0xFF`    | 與十進位相同：預期型別，否則為 `i64`                    |
| `0xFFi32` | `i32`                                                   |

### 預期型別與範圍檢查

範圍在編譯時檢查。

```nabi
let x: i32 = 42;            // i32
let y: i32 = 9999999999;    // compile error (outside the i32 range)
let z: f32 = 3.14;          // f32
let f: f64 = 1;             // f64: an integer literal fits a floating-point slot
let w: u8 = 256;            // compile error (256 ∉ u8 range 0..=255)
let i: i32 = 1.5;           // compile error: a floating-point literal cannot fill an integer slot
let big: int = 9999999999;  // int
```

完整的轉換矩陣請見 [2.11 轉換](211-conversion.md)。

### `Zero` / `One`

```nabi
trait Zero { fn zero() -> Self; }       // additive identity
trait One  { fn one() -> Self; }        // multiplicative identity
```

每個數值基本型別以及 `int` / `float` 都實作這兩個 trait。`Iterator::sum` / `product` 使用它們（[2.7.1](207-iter.md#271-iterator步進)）。

## 2.1.6. 算術結果型別

### `int + int = int`

由於適用自動提升，結果永遠是 `int` 本身。不會推論固定寬度。

```nabi
let a: int = 1000;
let b: int = 2000;
let c = a + b;       // int (3000)
```

### `i32 + i32 = i32`

固定寬度會保留。想要更寬型別的使用者必須使用明確轉換。

```nabi
let a: i32 = 5;
let b: i32 = 3;
let c: i32 = a + b;   // i32

let wider: i64 = (a as i64) + (b as i64);   // i64
```

### `int + i32`（混合：拒絕）

```nabi
let a: int = 5;       // int
let b: i32 = 3;       // i32
let c = a + b;        // compile error: type mismatch
```

此處需要明確轉換。

```nabi
let c1 = a + int::from_i32(b);    // int (a + int(3))
let c2 = (a.to_i32()? + b);       // i32 (OK if a is within the i32 range)
```

這項策略阻止默默混用型別。與 Julia 不同，Nabi 不會自動提升；由使用者指明要使用哪個世界。

編譯器對混合數字的提示會將較窄的一側轉成較寬的型別，因此建議的強制轉型不會遺失任何值：整數轉成浮點數，任何基本型別轉成 `int` 或 `float`，兩個有號或兩個無號整數則由較窄者轉成較寬者。有號型別可容納無號型別；當兩者寬度相同且小於 64 位元時，兩者都轉成寬度大一級的有號型別。

```nabi
let a: i32 = 1;
let b: i64 = 5000000000;
let c = a + b;   // help: convert one side: `(a as i64) + b`
let u: u32 = 3;
let d = a + u;   // help: convert both sides: `(a as i64) + (u as i64)`
```

## 2.1.7. 固定基本型別 ↔ int 轉換

固定基本型別與 `int` 之間的轉換只能明確進行。

| 轉換                     | 方法                                  | 失敗時                       |
|--------------------------|---------------------------------------|------------------------------|
| `i32` → `int`            | `.into::<int>()` 或 `int::from_i32`   | 一律成功（無損）             |
| `int` → `i32`            | `.try_into::<i32>()`                  | 擲回 `OverflowError`         |
| `int` → `i32`（飽和）    | `.saturating_into::<i32>()`           | i32::MIN 或 MAX              |
| `int` → `i32`（環繞）    | `.wrapping_into::<i32>()`             | 模運算                       |
| `int` → `f64`            | `int::to_f64`                         | 捨入到最近值；超過 `f64::MAX` 時為 ±inf |
| `int` → `f64`（檢查）    | `.try_into::<f64>()`                  | 捨入；超過 `f64::MAX` 時擲回 `OverflowError` |
| `int` → `f64`（飽和）    | `.saturating_into::<f64>()`           | 捨入；超出範圍時為 ±`f64::MAX` |

`float` → `f64` 有相同的三種形式。兩者都沒有 `From`（捨入的轉換不是無損的，[2.11.8](211-conversion.md)），也沒有 `as`（`as` 保留低位元，而浮點數沒有低位元）。

## 2.1.8. niche 最佳化（簡述）

[refs-lang 2.7 niche](../../refs-lang/002-types/207-niche-optimization.md) 定義其機制。獲得自動 niche 的基本型別如下：

`bool | none` 是 1 個位元組（niche 值為 `2`），`Ptr<T> | none` / `ConstPtr<T> | none` 與指標同大小（null 即 none）（[refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-automatic-niches)）。`i8`..`i64`、`u8`..`u64`、`f32` 與 `f64` 本身沒有自動 niche，因為每個位元樣式都是有效值。

需要選擇性加入的 niche 時，請使用 [2.2 non-zero](202-non-zero.md) 的 `NonZero<T>` / `NonNan<F>` / `Finite<F>` / `Positive<T>` 包裝型別。

## 2.1.9. lang 槽位對應

16 個語言內建（`i8`..`i64`、`u8`..`u64`、`isize`、`usize`、`f32`、`f64`、`bool`、`c_char`、`char`、`str`）由編譯器依名稱辨識。沒有任何標準函式庫項目為它們繫結 lang 槽位，也沒有為它們宣告 struct；函式庫只新增 `impl` 區塊（2.1.4），因此不需要 `#[keyword]` 或 `#[lang]`。

標準函式庫新增的 4 個型別是以 `#[keyword(...)]` 註冊、並以 `#[lang(...)]` 繫結到其 lang 槽位的 PascalCase struct。

| 基本型別  | lang 槽位    | struct 名稱 | 註冊方式              |
|-----------|--------------|-------------|-----------------------|
| `int`     | `int_type`   | `Int`       | `#[keyword("int")]`   |
| `float`   | `float_type` | `Float`     | `#[keyword("float")]` |
| `f16`     | `f16_type`   | `F16`       | `#[keyword("f16")]`   |
| `bf16`    | `bf16_type`  | `BF16`      | `#[keyword("bf16")]`  |

`Range<T>`（`a..b`）由 [2.7 iter](207-iter.md) 定義。

## 2.1.10. 與其他語言的比較：設計理由

Nabi 的雙世界模型（自動提升的 `int` 與固定基本型別）借鑑了多種語言。Smalltalk 的 SmallInteger ↔ LargeInteger 是自動提升的原型，而 Lisp 與 Scheme 的數值塔（fixnum / bignum / rational / real）是另一個重要參考。**Python int** 以任意精度為預設，但沒有固定基本型別；**Julia** 則以固定基本型別為預設。在 Nabi 中，字面值同樣預設為固定型別（`i64` / `f64`），`int` 需選擇性加入。

與各語言的差異：**Python** 只提供 int；由於缺少固定寬度，FFI 與效能關鍵的工作需要相依於 NumPy。**Julia** 與 Nabi 一樣預設為固定型別，需要任意精度時必須明確使用 `BigInt`。**Rust** 只有固定型別，因此使用者必須一律寫出型別。**Nabi** 讓兩個世界並存：字面值依循預期型別（沒有時為 `i64` / `f64`），需要任意精度的地方則寫 `int` / `float`。

## 2.1.11. 數值 trait、浮點數排序、範圍、複數

**`Num`。**`trait Num: Copy + Zero + One + Add<Output = Self> + Sub<Output = Self> + Mul<Output = Self> + Div<Output = Self> + PartialEq + PartialOrd {}`，由十二個固定基本型別實作：要撰寫適用於其中任何一個的程式碼，請寫 `T: Num`。混合運算元仍然會被拒絕（2.1.6）。

**`FloatingPoint`。**`trait FloatingPoint: Num + Neg<Output = Self>` 由 `f32` 與 `f64` 實作（`float` 不實作，其型別名為 `Float`，因此這個 trait 採用 Swift 的名稱）。它帶有常數 `NAN`、`INFINITY`、`NEG_INFINITY`、`EPSILON`、`PI`、`E`，以及方法 `abs sqrt cbrt exp ln log2 log10 powi pow sin cos tan asin acos atan atan2 sinh cosh tanh hypot floor ceil round trunc signum mul_add min max is_nan is_infinite is_finite total_cmp`（各自就是該型別同名的方法），還有 `to_f64`、`from_f64`（捨入到最近值）與 `from_usize`。

**`total_cmp`。**`f32::total_cmp` / `f64::total_cmp(self, other) -> Ordering` 是 IEEE 754 的 `totalOrder`：`-NaN < -inf < .. < -0.0 < +0.0 < .. < inf < +NaN`。`xs.sort_by(f64::total_cmp)` 可排序任何浮點數串列；`f64` 仍然沒有 `Ord`，`==` 仍然是 IEEE 語意。`f64::NAN` 與 `f32::NAN`（`0x7fc0_0000`）在每個目標上符號位元都清除，因此排在 `inf` 之後；算術產生的 NaN（`0.0 / 0.0`）可能帶有任一正負號（x86 會設定它），負的 NaN 排在最前面。

**`linspace` / `arange`**（`core::num`，對 `T: FloatingPoint` 泛型，回傳 `List<T>`）：

```nabi
linspace(0.0, 1.0, 5)     //→ [0.0, 0.25, 0.5, 0.75, 1.0]: n values, both ends included
arange(0.0, 1.0, 0.25)    //→ [0.0, 0.25, 0.5, 0.75]: start + i*step while before stop
```

`linspace(start, stop, n)` 產生 `start + i * step`，其中 `step = (stop - start) / (n - 1)`，最後一個值恰為 `stop`（`n == 1` 時只有 `start`，`0` 時什麼都沒有）。`arange(start, stop, step)` 產生 `ceil((stop - start) / step)` 個值 `start + i * step`，每個值都由 `i` 計算，因此不會累積誤差；步長為零或 NaN 時會 panic。`Range<f64>` 不是迭代器（[2.7.3](207-iter.md#273-step單步抽象)）。

**`gcd` / `lcm`。**每個固定寬度整數型別都有 `gcd(other)`，即最大公因數，絕不為負（兩個零時為 `0`；`12i32.gcd(-18) == 6`），以及 `lcm(other)`，即最小公倍數，絕不為負（任一為 0 時為 `0`）。結果放不下時會 panic，並提示改用 `int` 計算（`i64::MIN.gcd(0)`、溢位的 `lcm`）；`int` 有自己的 `gcd` / `lcm`（2.1.2）。

**`is_close`。**`f64::is_close(self, other, rel: f64 = 1e-9, abs: f64 = 0.0, equal_nan: bool = false) -> bool` 即 Python 的 `math.isclose`：當兩者相等，或兩者皆有限且 `|a - b| <= max(rel * max(|a|, |b|), abs)` 時為 `true`。它對兩個值是對稱的。預設的 `rel` 要求大約九位相同的數字；接近零時只有 `abs` 有作用（`1e-12.is_close(0.0)` 是 `false`，`1e-12.is_close(0.0, abs: 1e-9)` 是 `true`）。無窮大只與自己接近；NaN 不與任何值接近，除非指定 `equal_nan` 且兩者都是 NaN。`f32::is_close` 的形式相同，預設 `rel` 為 2^-20（最後一位的八個單位；1e-9 低於 `f32` 的精度）。陣列形式是 `Array::is_close` / `all_close`（[2.9.14](209-collections.md#2914-arrayt-const-n-usizen-維陣列)）。

**`comb` / `perm` / `fsum`**（`core::num`）：

```nabi
fn comb(n: u64, k: u64) -> int;                     // n! / (k! (n - k)!), 0 when k > n; Python math.comb
fn perm(n: u64, k: u64) -> int;                     // n! / (n - k)!, 0 when k > n; Python math.perm
fn fsum<I: IntoIterator<Item = f64>>(xs: I) -> f64; // correctly rounded sum; Python math.fsum
```

`comb` 與 `perm` 是精確的（`comb(100, 50) == 100891344545564193334812497256`）。`fsum` 將精確的總和保存為不重疊的部分和串列（Shewchuk 1997），並只捨入一次（捨入到最近值，平手時取偶數），因此結果與順序及數量級無關（`fsum([0.1; 10]) == 1.0`、`fsum([1e100, 1.0, -1e100]) == 1.0`）。值中有無窮大或 NaN 時得到一般的 IEEE 結果（`inf - inf` 是 NaN），累計和超出 `f64` 範圍時也一樣（`[1e308, 1e308, -1e308]` 得到 `inf`）。它的成本約為一般迴圈的三倍；`Array::sum` 採用成對加總，速度快。

**`div_floor` / `rem_floor`。**每個整數型別都有 `div_floor(rhs)`，即向負無窮大捨入的商（Python 的 `//`），以及 `rem_floor(rhs)`，即與除數同號的餘數（Python 的 `%`，Nabi 的 `%%`）：`(-7).div_floor(2) == -4`、`(-7).rem_floor(2) == 1`。對無號型別而言，它們就是 `/` 與 `%`。

**`Complex<T>`**（`core::num::Complex`）是 `struct Complex<T> { pub re: T; pub im: T; }`，對 `T: FloatingPoint` 提供 `new`、`i()`、`from_polar`、`abs`、`norm_sqr`、`arg`、`conj`、`recip`、`exp`、`ln`、`sqrt`（主平方根，每個部分都達到完整的相對精度，包括極小的虛部）、`powi`、`pow`、`is_nan`；複數之間以及與任一側的實數 `T` 之間的 `+ - * /` 與一元 `-`（`2.0 * z`、`z / 2.0`；實數在左側僅限 `f32` 與 `f64`）、`Zero`、`One`、`PartialEq`、`Debug`（`Complex { re: 1.0, im: 2.0 }`）與 `Display`（`1+2i`、`1-2i`）。數值後綴 `i` 由 core 以 `#[literal(number "i", priority = -1)]` 註冊，產生虛數 `Complex<f64>`：`3.0 + 4.0i`、`1i`。套件自己的 `i` 處理器優先於它（[refs-lang 6.5.5](../../refs-lang/006-meta-system/605-literal-magic.md#655-multiple-handlers-ambiguity)）。
