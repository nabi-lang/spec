# 2.1. Numeric types

The `core::num` module defines every numeric type of nabi. Nabi keeps numbers from two
worlds side by side, and the user is free to choose either.

```
int / float    (auto-promote, arbitrary precision, opt-in)
fixed primitive (i8..i64, u8..u64, isize, usize, f32, f64; hardware-friendly, default)
```

The two worlds **coexist**. An unsuffixed literal follows the expected type; if there is no expected type it is `i64` / `f64` ([refs-lang 1.6.3](../../refs-lang/001-lexical/106-numeric-literals.md#163-unsuffixed-literals)). `int` / `float` are used only where the type is written out.

## 2.1.1. Two worlds of numbers - why they are separate

### `int` / `float` - the auto-promote world

This world aims at arbitrary precision. Overflow never happens; computation ends with OOM
only when the memory limit is reached. Small values go through the small-int optimization
and are stored in hardware fixed-width form; as a value grows it is automatically promoted
to a heap BigInt. `int` / `float` values are immutable, and a heap BigInt is never modified in place, only shared (which is why they are `Copy`). The user is thereby guaranteed arithmetic correctness: `I32_MAX + 1` never
silently wraps into an absurd result.

### Fixed primitives - the hardware-friendly world

Fixed primitives, by contrast, are the world close to the hardware. They comprise 15 types
such as `i32` / `f64`; their syntax is defined by [refs-lang 2.1 Type expressions](../../refs-lang/002-types/201-type-expressions.md).
They suit situations that need precise control of memory and performance (FFI interfaces, SIMD,
large arrays, hot loops), and their overflow policy is explicit (see [2.3 overflow behavior](203-ops.md#overflow-behavior)).

### Which one is the default

An unsuffixed literal (`42`, `3.14`) is **`i64` / `f64`** when nothing fixes its type (a `let`-bound literal takes its type from later uses, [1.6.3](../../refs-lang/001-lexical/106-numeric-literals.md#163-unsuffixed-literals)). The auto-promote world is used only where `int` / `float` is written out.

```nabi
let total: int = 0;               // int (auto-promote), explicit
for x in xs { total = total + x; }
// total is auto-promoted even past the i64 range; no silent overflow

let n = 0;                        // i64 (nothing fixes it)
let small: i32 = 0;               // i32: follows the expected type
```

## 2.1.2. `int` - arbitrary-precision integer

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

These follow the fixed-width integers' names and panics. Bit counts and indices are `u64`, like `bit_length`: a value can have more than 2^32 bits (the fixed-width rule is below). The width-bound bit methods (`leading_zeros`, `rotate_left`, `swap_bytes`, ...) have no meaning without a width and are left out. `count_ones` counts `|self|` (Python's `bit_count`), since a negative number has infinitely many one bits. The representation is in the name: `to_le_bytes` / `from_le_bytes` use two's complement, which agrees with the fixed-width integers' bytes, and the `_unsigned` forms use the magnitude, which panics on a negative value (`Int::to_le_bytes_unsigned: negative value; use to_le_bytes for two's complement or call .abs() first`). The bytes are as short as the value allows: `128` is `[128, 0]` and `[128]` unsigned, `-1` is `[255]`, zero is `[0]`. The `write_*` forms fill a fixed width instead, every byte of `out` (a `List<u8>` or a view of one): `-2` in four bytes is `[254, 255, 255, 255]` little-endian, and a value that needs more bytes than `out` has panics (`Int::write_le_bytes: the value needs 2 bytes but `out` has 1; pass a view of at least 2 bytes`). The `from_*` forms take any bytes ([`AsBytes`](211-conversion.md)): a `List<u8>`, a `Slice<u8>`, a `[u8; N]` or a `str`. `pow` takes a `u32` exponent: a larger one gives at least 2^32 bits for any base but 0 and ±1. `modpow` takes an `int` exponent, since its result stays below `|m|`; the result has the sign of `m`, like [`%%`](203-ops.md#modulo---floored-modulo-). A negative exponent is an error (no modular inverse). Zero is `Int::default()` (or `Int::zero()`, see `Zero` in 2.1.5); `Int` has no `new`. Dividing by zero panics with `attempt to divide an int by zero`, from the operators `/`, `%` and `%%` and from `div_rem`, `div_euclid` and `rem_euclid` alike. The checked `to_i32` / `to_i64` / `to_u64` sit beside `TryFrom` as inherent methods ([0.5.11](../000-meta/005-notation.md)).

`Int` is defined in `core::num::bigint::Int` (`Float` in
`core::num::bigfloat::Float`). `#[keyword("int")]` registers the identifier `int` as a type keyword,
and this registration applies to every package. User code therefore writes the type as lowercase
`int`, not PascalCase `Int`
([refs-lang 6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) is
authoritative). The compiler finds the `int` type through `#[lang("int_type")]`.

The key properties of `int` are as follows. In memory, small
values go through the small-int optimization and are stored directly in a 64-bit word, while large values are
automatically promoted to a heap BigInt representation; all of this is
transparent to the user. The representation is always canonical: a value within the i64 range is in small form, otherwise it is a BigInt, and when an arithmetic result returns to the i64 range it returns to small form. Equal values have equal representations. The **lang slot** used is `int_type` ([0.4 lang slot
table](../000-meta/004-lang-slot-table.md)). An unsuffixed integer literal is `int` only when the expected type is `int`.

### Arithmetic

Arithmetic between `int`s never wraps or saturates.

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

`int` implements the arithmetic traits ([2.3 ops](203-ops.md)) `Add`, `Sub`, `Mul`, `Div`, `Rem`, `Modulo` and `Neg`. `Div` is integer division (truncating), so
`7 / 2 = 3` and `-7 / 2 = -3`. The bitwise traits `BitAnd`, `BitOr`, `BitXor` and `BitNot` work on two's complement with the sign bit repeated forever (`~x == -x - 1`). The shifts are `Shl<u32>` and `Shr<u32>`: the right side is a `u32`, not `int`; `x << n == x * 2^n` and `x >> n == floor(x / 2^n)` (an arithmetic shift). The comparison traits ([2.5
cmp](205-cmp.md)) `PartialEq`, `Eq`, `PartialOrd`, `Ord` and `Hash` are implemented, as are `Clone`, `Copy`, `Default` (zero), `Zero`, `One`, `Display`, `Debug` and `FromStr`.

### Main uses

`int` serves in many situations. As the arithmetic default in a REPL or Jupyter,
it guarantees correctness without typing annotations every time; accumulated sums or
products in numeric simulations carry no overflow concern. Cryptographic operations and
large-integer processing work with the standard library alone, without depending on a separate BigInt library.

### Limitations

`int` has limitations. In memory, BigInt promotion requires heap
allocation, so fixed primitives are recommended in hot loops and large arrays.
In performance, it is close to fixed primitives within the small-int range, but necessarily slower after BigInt
promotion. For FFI, `extern "C"`
functions accept only fixed primitives, so an `int` must be explicitly converted to `i64` or similar before
being passed.

## 2.1.3. `float` - floating-point with unbounded exponent

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

`#[keyword("float")]` registers the `float` keyword and routes it to struct `Float`.
User code therefore sees only lowercase `float`.

A `float` value has a 53-bit significand (the same precision as f64) and an unbounded (i64) exponent.
It normally uses the f64 representation; when a finite result exceeds the f64 range (overflow) or becomes
too small toward 0 (underflow), it is promoted to a BigFloat representation instead of becoming `inf` / `0`.

- `inf` / `NaN` arise only from division by zero or from `inf` / `NaN` operands.
- `%` is an exact fmod.
- `Display` writes a value in the f64 range as f64 does; a promoted value is written with an exponent, as in `1.5e400` (`{:g}` gives `1.5e+400`).

### Stated limitations

The limitations of `float` must be stated clearly. Its precision is 53 bits, not unbounded,
so it has the same round-off as IEEE 754 f64. For example, `0.1 + 0.2` not being
`0.3` happens in Nabi as well. `Eq` is also not
implemented: `float` does not implement the `Eq` marker trait and implements only `PartialEq`, to handle NaN.
See [2.5 cmp](205-cmp.md) for the full hierarchy.

### lang slot

The compiler finds the `float` type through `float_type` ([0.4 lang slot table](../000-meta/004-lang-slot-table.md)). An unsuffixed floating-point literal is `float` only when the expected type is `float`.

## 2.1.4. Fixed-width primitives

[refs-lang 2.1 Type expressions](../../refs-lang/002-types/201-type-expressions.md) defines the **names** of the 15 primitives.
This section defines the standard library methods and behavior on top of that.

### Integers (10 types)

| Type    | Bit width                       | Range          |
|---------|---------------------------------|----------------|
| `i8`    | 8                               | -128 ~ 127     |
| `i16`   | 16                              | -32768 ~ 32767 |
| `i32`   | 32                              | -2³¹ ~ 2³¹-1   |
| `i64`   | 64                              | -2⁶³ ~ 2⁶³-1   |
| `isize` | target pointer width (32 or 64) | target-dep     |
| `u8`    | 8                               | 0 ~ 255        |
| `u16`   | 16                              | 0 ~ 65535      |
| `u32`   | 32                              | 0 ~ 2³²-1      |
| `u64`   | 64                              | 0 ~ 2⁶⁴-1      |
| `usize` | target pointer width            | target-dep     |

### Floating-point (2 types)

| Type  | Bit width | Standard                             |
|-------|-----------|--------------------------------------|
| `f32` | 32        | IEEE 754 binary32 (single precision) |
| `f64` | 64        | IEEE 754 binary64 (double precision) |

The 16-bit floats `f16` and `bf16` are standard library additions for storage ([below](#f16-and-bf16---16-bit-storage-floats)).

### Floating-point semantics

`f32` and `f64` behave the same on every target:

- **Rounding.** `+ - * /`, `%` (exact, C's `fmod`), `√` / `sqrt`, `mul_add`, every `as` conversion into a float, every float literal and `parse` round to nearest, ties to even. A literal or a parsed string is rounded once from its decimal value: `0.1f32` is the `f32` nearest to 0.1, not `0.1f64` rounded again. There is no other rounding mode and no way to change it.
- **No contraction.** `a * b + c` is a product rounded, then a sum rounded. Only `mul_add` fuses the two into one rounding. Neither the optimizer nor any backend contracts, reassociates or reorders float operations: a loop summing floats is not vectorized (lanes would change the order), and constant evaluation gives the bits the program would compute.
- **Subnormals are kept.** No target flushes subnormal inputs or results to zero. The runtime leaves the floating-point control state (MXCSR, FPCR, FPSCR) at its IEEE default on every target, and Nabi code cannot change it.
- **NaN.** Every comparison with a NaN is false except `!=`. Which NaN an operation returns is unspecified: its payload and its sign may differ between targets (x86 makes `0.0 / 0.0` negative). `-x`, `abs` and `copysign` change only the sign bit, of a NaN too, and `to_bits` / `from_bits` keep every bit. `f32::NAN` and `f64::NAN` are fixed quiet NaNs with the sign bit clear.
- **Signed zero.** `-0.0 == 0.0`, and `1.0 / -0.0` is `-inf`.
- **`min` / `max`** are IEEE 754 `minimumNumber` / `maximumNumber`: a NaN operand gives the other one (NaN only when both are), and `-0.0` orders below `+0.0`, so `(0.0).min(-0.0)` is `-0.0` and `(-0.0).max(0.0)` is `0.0` on every target. `total_cmp` is the IEEE total order ([2.1.11](#2111-numeric-traits-float-ordering-ranges-complex-numbers)).
- **Float to integer** `as` truncates toward zero and saturates at the integer type's bounds; NaN gives 0. Integer to float rounds to nearest, ties to even.
- **Library functions.** `sin`, `exp`, `ln`, `pow` and the other transcendental functions are the target C library's: not correctly rounded, and the last bit may differ between targets. Constant evaluation does not fold them.

`float` (2.1.3) follows the same rules inside `f64`'s range.

### `f16` and `bf16` - 16-bit storage floats

```nabi
#[lang("f16_type")]
#[keyword("f16")]
pub struct F16 { bits: u16; }      // IEEE 754 binary16

#[lang("bf16_type")]
#[keyword("bf16")]
pub struct BF16 { bits: u16; }     // bfloat16
```

| Type   | Significand bits | Exponent bits | Largest   | Smallest normal | `EPSILON` |
|--------|------------------|---------------|-----------|-----------------|-----------|
| `f16`  | 11 (10 stored)   | 5             | 65504     | 2^-14           | 2^-10     |
| `bf16` | 8 (7 stored)     | 8 (`f32`'s)   | about 3.39e38 | 2^-126      | 2^-7      |

`f16` and `bf16` (`core::num::half`) are storage types: they halve the memory of large arrays (model weights, images, sensor data) and compute through `f32`. Each is 2 bytes with alignment 2, a `List<f16>` or `Array<bf16, 2>` stores 2 bytes per element, and `Array::zeros` / `ones` work over them.

- **Literals.** An unsuffixed literal in an `f16` / `bf16` slot, or one with the suffix `f16` / `bf16` (`0.1f16`, `2bf16`), is the value rounded once from its decimal. A literal that rounds to infinity is a compile error (`let h: f16 = 70000.0;`); an integer literal must be exact (`let h: f16 = 2049;` is an error, as for `f32`).
- **`as`** converts to and from every integer and float primitive, and between `f16` and `bf16`. Into a 16-bit float it rounds to nearest, ties to even, straight from the source: an `f64` or an `i64` is never rounded to `f32` first. Out of one it is exact into `f32` / `f64`, and into an integer it is the `f32` value's `as`. `int`, `float`, `bool` and `char` do not convert with `as` (write `x.to_f64() as f16`).
- **Arithmetic.** `+ - * / % %%`, their compound assignments, unary `-` and `√` are computed in `f32` and rounded once to the type. For `+ - * / %` and `√` that is exactly the correctly rounded IEEE result of the 16-bit format (`f32` has more than twice its significand bits plus two). Mixed operands are rejected as for the primitives (2.1.6). There are no transcendental methods and no `FloatingPoint` impl: convert to `f32` (`h.to_f32().sin()`).
- **Comparison** (`==`, `<`, ...) is that of the `f32` values: IEEE, NaN unequal to everything, `-0.0 == 0.0`. Neither type is `Eq`, `Ord` or `Hash`; sort with `xs.sort_by(f16::total_cmp)`. A literal pattern does not match them: compare with `==` in a guard.
- **Methods and constants.** `from_bits` / `to_bits` (the `u16` pattern), `from_f32`, `from_f64`, `from_i64`, `from_u64`, `to_f32`, `to_f64`, `is_nan`, `is_infinite`, `is_finite`, `is_normal`, `is_sign_positive`, `is_sign_negative`, `abs`, `copysign`, `min`, `max` (as `f32`'s), `total_cmp`; `EPSILON`, `MAX`, `MIN`, `MIN_POSITIVE`, `NAN`, `INFINITY`, `NEG_INFINITY`.
- **Traits.** `Clone`, `Copy`, `Default` (`0.0`), `Zero`, `One`, `PartialEq`, `PartialOrd`, the operator traits above, `Display`, `Debug`, `FromStr` (rounded once from the decimal), and `From<f16>` / `From<bf16>` for `f32` and `f64` (exact).
- **Text.** `Display` writes the shortest digits that read back as the same value (`0.1`, `65500`), never an exponent, `inf` / `NaN`; `Debug` adds `.0` to a whole number. A format spec (`{h:.3}`, `{h:e}`) formats the exact value as `f64` does.
- **Hardware.** `f16` <-> `f32` uses the CPU where it converts: F16C on x86-64 when the target features include `f16c` (`-C target-cpu=x86-64-v3`), `fcvt` on arm64. Elsewhere it is a runtime call, with the same results. `bf16` conversions are a few integer operations, inline.
- **C.** In an `extern "C"` signature `f16` is C's `_Float16` and `bf16` is `__bf16`, passed in floating-point registers as C passes them ([refs-lang 8.6.3](../../refs-lang/008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)). Where the target's C has no such type (powerpc64le and wasm32; `__bf16` on s390x; a half result on i686) the signature is an error: pass `h.to_bits()` as a `uint16_t` instead.

### Boolean / character (3 types)

| Type     | Meaning                                                                                |
|----------|----------------------------------------------------------------------------------------|
| `bool`   | `true` / `false`                                                                       |
| `char`   | Unicode scalar value (4 bytes), compiler primitive. Methods in [2.15](215-string.md)  |
| `c_char` | C ABI compatible char (1 byte, sign target-dependent, compares as its byte `c as u8`); FFI only |

### Two groups - language built-ins vs standard library additions

The numeric / boolean / character types of this chapter fall into **two groups**.

| Group                                | Members                                                                                               | Registration                                       |
|--------------------------------------|-------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| **16 language built-ins**            | `i8`/`i16`/`i32`/`i64`, `u8`/`u16`/`u32`/`u64`, `isize`/`usize`, `f32`/`f64`, `bool`, `char`, `c_char`, `str` | Recognized by the language; no `#[keyword]` needed  |
| **4 standard library additions**     | `int`, `float`, `f16`, `bf16`                                                                 | Registered with `#[keyword(...)]`                  |

Language built-ins are tokens the compiler (lexer/parser) recognizes directly, so the standard library
does not register them as keywords. This automatically guarantees that users cannot
redefine these names.

The standard library additions (`int` / `float` / `f16` / `bf16`), in contrast, are registered with `#[keyword(...)]`.
The `#[keyword]` attribute is attached to the `Int` / `Float` / `F16` / `BF16` definitions in this module ([refs-lang
6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) is authoritative).

### Standard library form of the built-in primitives

```nabi
impl i32 { ... }              // inherent constants and methods (below)
impl Add for i32 { ... }      // trait impls: Add, PartialOrd, Display, Clone, ...
impl PartialEq for c_char { ... }
```

The standard library declares no struct for a built-in primitive. It only attaches `impl` blocks
for inherent methods and traits (`Add`, `Display`, `Clone`, ...). Recognizing the type itself is the
compiler's responsibility.

### Arithmetic / comparison trait impls

Each primitive implements the traits of [2.3 ops](203-ops.md), [2.4 ops-assign](204-ops-assign.md) and
[2.5 cmp](205-cmp.md). Every operator method and every `PartialEq` / `PartialOrd` method, `char` and `c_char` included, carries
`#[intrinsic("nabi.OP.TYPE")]`, which binds it directly to a compiler backend PrimOp. The one exception is `bool`'s `PartialOrd`, a plain body (`false < true`). `c_char` compares as its byte on every target (`c as u8`, 2.15.5): `255u8 as c_char > 1u8 as c_char`.

### Methods (per primitive)

Each integer / floating-point type exposes the following methods (the example is `i32`).

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

`i8`, `i16`, `i64` and `isize` have the same set with their own width. The unsigned types
(`u8`..`u64`, `usize`) have the same set except the sign-related methods (`abs`, `unsigned_abs`,
`signum`, `is_positive`, `is_negative`, `checked_isqrt`, `checked_abs`, `saturating_neg`,
`saturating_abs`, `wrapping_abs`); `abs_diff` returns the same type, and `checked_neg` is `none`
for every value but 0. They add powers of two.

A bit count or bit index has the type of `BITS` and of shift amounts: `u32` for every fixed-width
integer (`count_ones`, `leading_zeros`, `trailing_zeros`, `ilog2`, `rotate_left(n: u32)`,
`pow(exp: u32)`, `wrapping_shl(n: u32)`), whatever the integer's own width; `trailing_zeros(0)` is `BITS`. `int` uses
`u64` instead, since a value can exceed 2^32 bits, and `trailing_zeros` of 0 is `none`. The `<<` / `>>` operators differ: on a fixed-width integer the right side has the left side's type (`Shl for T`, so `i64 << i64`), on `int` it is `u32` (`Shl<u32>`).

```nabi
impl u32 {
    fn is_power_of_two(self) -> bool;
    fn next_power_of_two(self) -> u32;                 // smallest power of two >= self; panics on overflow
    fn checked_next_power_of_two(self) -> u32 | none;
}
```

The minimum / maximum of an integer type are the associated constants `i32::MIN` / `i32::MAX`, not functions. An argument-less inherent method `min()` / `max()` would shadow `Ord::min` / `Ord::max` (`a.min(b)`) of the same name.

Floating-point types (`f32`, `f64`) provide these constants and methods (`f32` has the same set
with `f32` in place of `f64` and `u32` bits).

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

## 2.1.5. Literal → which type?

The rules of [refs-lang 1.6 Numeric literals](../../refs-lang/001-lexical/106-numeric-literals.md) are:

| Form      | Resulting type                                          |
|-----------|---------------------------------------------------------|
| `42`      | Expected type (integer / floating-point type), else `i64` |
| `3.14`    | Expected type (floating-point type), else `f64`         |
| `42i32`   | `i32`                                                   |
| `42u64`   | `u64`                                                   |
| `3.14f32` | `f32`                                                   |
| `3.14f64` | `f64`                                                   |
| `0.5f16`  | `f16`                                                   |
| `0.5bf16` | `bf16`                                                  |
| `'A'`     | `char`                                                  |
| `c'A'`    | `c_char` (primitive)                                    |
| `0xFF`    | Same as decimal: expected type, else `i64`              |
| `0xFFi32` | `i32`                                                   |

### Expected type and range checking

Ranges are checked at compile time.

```nabi
let x: i32 = 42;            // i32
let y: i32 = 9999999999;    // compile error (outside the i32 range)
let z: f32 = 3.14;          // f32
let f: f64 = 1;             // f64: an integer literal fits a floating-point slot
let w: u8 = 256;            // compile error (256 ∉ u8 range 0..=255)
let i: i32 = 1.5;           // compile error: a floating-point literal cannot fill an integer slot
let big: int = 9999999999;  // int
```

The full conversion matrix is in [2.11 Conversion](211-conversion.md).

### `Zero` / `One`

```nabi
trait Zero { fn zero() -> Self; }       // additive identity
trait One  { fn one() -> Self; }        // multiplicative identity
```

Every numeric primitive and `int` / `float` implement both traits. `Iterator::sum` / `product` use them ([2.7.1](207-iter.md#271-iterator---stepping)).

## 2.1.6. Arithmetic result types

### `int + int = int`

Because auto-promote applies, the result is always `int` itself. No fixed-width inference
takes place.

```nabi
let a: int = 1000;
let b: int = 2000;
let c = a + b;       // int (3000)
```

### `i32 + i32 = i32`

Fixed width is preserved. A user who wants a wider type must use an explicit
conversion.

```nabi
let a: i32 = 5;
let b: i32 = 3;
let c: i32 = a + b;   // i32

let wider: i64 = (a as i64) + (b as i64);   // i64
```

### `int + i32` (mixed - rejected)

```nabi
let a: int = 5;       // int
let b: i32 = 3;       // i32
let c = a + b;        // compile error: type mismatch
```

An explicit conversion is required here.

```nabi
let c1 = a + int::from_i32(b);    // int (a + int(3))
let c2 = (a.to_i32()? + b);       // i32 (OK if a is within the i32 range)
```

This policy blocks silent type mixing. Unlike Julia, Nabi does not auto-promote;
the user states which world to use.

The compiler's help on mixed numbers converts the narrower side to the wider
type, so the suggested cast loses no value: an integer goes to a float, any
primitive goes to `int` or `float`, and of two signed or two unsigned integers
the narrower goes to the wider. A signed type takes an unsigned one; when both
have the same width below 64 bits, both go to the signed type one width up.

```nabi
let a: i32 = 1;
let b: i64 = 5000000000;
let c = a + b;   // help: convert one side: `(a as i64) + b`
let u: u32 = 3;
let d = a + u;   // help: convert both sides: `(a as i64) + (u as i64)`
```

## 2.1.7. Fixed primitive ↔ int conversion

Conversion between fixed primitives and `int` is explicit only.

| Conversion               | Method                                | On failure                   |
|--------------------------|---------------------------------------|------------------------------|
| `i32` → `int`            | `.into::<int>()` or `int::from_i32`   | Always succeeds (lossless)   |
| `int` → `i32`            | `.try_into::<i32>()`                  | Throws `OverflowError`       |
| `int` → `i32` (saturate) | `.saturating_into::<i32>()`           | i32::MIN or MAX              |
| `int` → `i32` (wrap)     | `.wrapping_into::<i32>()`             | Modular arithmetic           |
| `int` → `f64`            | `int::to_f64`                         | Rounds to the nearest value; ±inf past `f64::MAX` |
| `int` → `f64` (checked)  | `.try_into::<f64>()`                  | Rounds; throws `OverflowError` past `f64::MAX` |
| `int` → `f64` (saturate) | `.saturating_into::<f64>()`           | Rounds; ±`f64::MAX` past the range |

`float` → `f64` has the same three forms. Neither has `From` (a rounding conversion is not lossless, [2.11.8](211-conversion.md)) nor `as` (`as` keeps the low bits, and a float has none).

## 2.1.8. Niche optimization (brief)

[refs-lang 2.7
niche](../../refs-lang/002-types/207-niche-optimization.md) defines the mechanism.
The primitives that get an automatic niche are:

`bool | none` is 1 byte (niche value `2`), and `Ptr<T> | none` / `ConstPtr<T> | none`
are pointer-sized (null is none) ([refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-automatic-niches)). `i8`..`i64`,
`u8`..`u64`, `f32` and `f64` themselves get no automatic niche, because every bit
pattern is a valid value.

When an opt-in niche is needed, use the
`NonZero<T>` / `NonNan<F>` / `Finite<F>` / `Positive<T>` wrappers of [2.2 non-zero](202-non-zero.md).

## 2.1.9. lang slot mapping

The 16 language built-ins (`i8`..`i64`, `u8`..`u64`, `isize`, `usize`, `f32`, `f64`, `bool`,
`c_char`, `char`, `str`) are recognized by the compiler by name. No standard library item binds a
lang slot for them and no struct is declared for them; the library adds only `impl` blocks
(2.1.4), so no `#[keyword]` or `#[lang]` is needed.

The 4 types added by the standard library are PascalCase structs registered with `#[keyword(...)]` and bound to their lang slots with `#[lang(...)]`.

| primitive | lang slot    | struct name | Registration          |
|-----------|--------------|-------------|-----------------------|
| `int`     | `int_type`   | `Int`       | `#[keyword("int")]`   |
| `float`   | `float_type` | `Float`     | `#[keyword("float")]` |
| `f16`     | `f16_type`   | `F16`       | `#[keyword("f16")]`   |
| `bf16`    | `bf16_type`  | `BF16`      | `#[keyword("bf16")]`  |

`Range<T>` (`a..b`) is defined by [2.7 iter](207-iter.md).

## 2.1.10. Comparison with other languages - design rationale

Nabi's two-world model (auto-promote `int` and fixed primitives) draws on several
languages. Smalltalk's SmallInteger ↔ LargeInteger is the prototype of auto-promote,
and the numeric tower of Lisp and Scheme (fixnum / bignum /
rational / real) is another important reference. **Python int** makes arbitrary precision
the default but has no fixed primitives, while **Julia** makes fixed
primitives the default. In Nabi too, literals default to fixed (`i64` / `f64`), and `int` is opt-in.

The differences from each language: **Python** offers only int; lacking
fixed width, it depends on NumPy for FFI and performance-critical
work. **Julia**, like Nabi, defaults to fixed and requires an explicit `BigInt` when arbitrary precision is
needed. **Rust** is fixed only, so the user must always
write the type. **Nabi** lets the two worlds coexist:
literals follow the expected type (`i64` / `f64` if none), and `int` / `float` are written where arbitrary precision is needed.

## 2.1.11. Numeric traits, float ordering, ranges, complex numbers

**`Num`.** `trait Num: Copy + Zero + One + Add<Output = Self> + Sub<Output = Self> + Mul<Output = Self> + Div<Output = Self> + PartialEq + PartialOrd {}`, implemented by the twelve fixed primitives: write `T: Num` for code over any of them. Mixed operands stay rejected (2.1.6).

**`FloatingPoint`.** `trait FloatingPoint: Num + Neg<Output = Self>` is implemented by `f32` and `f64` (not by `float`, whose type is named `Float`, hence the trait's Swift name). It carries the constants `NAN`, `INFINITY`, `NEG_INFINITY`, `EPSILON`, `PI`, `E` and the methods `abs sqrt cbrt exp ln log2 log10 powi pow sin cos tan asin acos atan atan2 sinh cosh tanh hypot floor ceil round trunc signum mul_add min max is_nan is_infinite is_finite total_cmp`, each the type's own method of that name, and `to_f64`, `from_f64` (round to nearest) and `from_usize`.

**`total_cmp`.** `f32::total_cmp` / `f64::total_cmp(self, other) -> Ordering` is the IEEE 754 `totalOrder`: `-NaN < -inf < .. < -0.0 < +0.0 < .. < inf < +NaN`. `xs.sort_by(f64::total_cmp)` sorts any float list; `f64` stays without `Ord` and `==` stays IEEE. `f64::NAN` and `f32::NAN` (`0x7fc0_0000`) have the sign bit clear on every target, so they order after `inf`; a NaN that arithmetic makes (`0.0 / 0.0`) may have either sign (x86 sets it), and a negative one orders first.

**`linspace` / `arange`** (`core::num`, generic over `T: FloatingPoint`, returning `List<T>`):

```nabi
linspace(0.0, 1.0, 5)     //→ [0.0, 0.25, 0.5, 0.75, 1.0]: n values, both ends included
arange(0.0, 1.0, 0.25)    //→ [0.0, 0.25, 0.5, 0.75]: start + i*step while before stop
```

`linspace(start, stop, n)` gives `start + i * step` with `step = (stop - start) / (n - 1)` and the last value exactly `stop` (just `start` for `n == 1`, nothing for `0`). `arange(start, stop, step)` gives `ceil((stop - start) / step)` values `start + i * step`, each computed from `i` so no error accumulates; a zero or NaN step panics. A `Range<f64>` is not an iterator ([2.7.3](207-iter.md#273-step---the-one-step-abstraction)).

**`gcd` / `lcm`.** Every fixed-width integer type has `gcd(other)`, the greatest common divisor, never negative (`0` for two zeros; `12i32.gcd(-18) == 6`), and `lcm(other)`, the least common multiple, never negative (`0` when either is 0). A result that does not fit panics, saying to compute it as `int` (`i64::MIN.gcd(0)`, an overflowing `lcm`); `int` has its own `gcd` / `lcm` (2.1.2).

**`is_close`.** `f64::is_close(self, other, rel: f64 = 1e-9, abs: f64 = 0.0, equal_nan: bool = false) -> bool` is Python's `math.isclose`: `true` when the two are equal, or both finite and `|a - b| <= max(rel * max(|a|, |b|), abs)`. It is symmetric in the two values. The default `rel` asks for about nine equal digits; near zero only `abs` helps (`1e-12.is_close(0.0)` is `false`, `1e-12.is_close(0.0, abs: 1e-9)` is `true`). An infinity is close only to itself, and NaN to nothing unless `equal_nan` and both are NaN. `f32::is_close` has the same form with a default `rel` of 2^-20 (eight units in the last place; 1e-9 is below `f32`'s precision). The array forms are `Array::is_close` / `all_close` ([2.9.14](209-collections.md#2914-arrayt-const-n-usize---n-dimensional-array)).

**`comb` / `perm` / `fsum`** (`core::num`):

```nabi
fn comb(n: u64, k: u64) -> int;                     // n! / (k! (n - k)!), 0 when k > n; Python math.comb
fn perm(n: u64, k: u64) -> int;                     // n! / (n - k)!, 0 when k > n; Python math.perm
fn fsum<I: IntoIterator<Item = f64>>(xs: I) -> f64; // correctly rounded sum; Python math.fsum
```

`comb` and `perm` are exact (`comb(100, 50) == 100891344545564193334812497256`). `fsum` keeps the exact sum as a list of non-overlapping partial sums (Shewchuk 1997) and rounds it once, to nearest with ties to even, so the result does not depend on the order or the magnitudes (`fsum([0.1; 10]) == 1.0`, `fsum([1e100, 1.0, -1e100]) == 1.0`). An infinity or NaN among the values gives the plain IEEE result (`inf - inf` is NaN), and so does a running sum past the range of `f64` (`[1e308, 1e308, -1e308]` gives `inf`). It costs about three times a plain loop; `Array::sum` is pairwise and fast.

**`div_floor` / `rem_floor`.** Every integer type has `div_floor(rhs)`, the quotient rounded toward negative infinity (Python's `//`), and `rem_floor(rhs)`, the remainder with the divisor's sign (Python's `%`, Nabi's `%%`): `(-7).div_floor(2) == -4`, `(-7).rem_floor(2) == 1`. For unsigned types they are `/` and `%`.

**`Complex<T>`** (`core::num::Complex`) is `struct Complex<T> { pub re: T; pub im: T; }` with `new`, `i()`, `from_polar`, `abs`, `norm_sqr`, `arg`, `conj`, `recip`, `exp`, `ln`, `sqrt` (the principal root, each part to full relative precision, a tiny imaginary part included), `powi`, `pow`, `is_nan` for `T: FloatingPoint`; `+ - * /` and unary `-` between complex numbers and with a real `T` on either side (`2.0 * z`, `z / 2.0`; a real on the left for `f32` and `f64`), `Zero`, `One`, `PartialEq`, `Debug` (`Complex { re: 1.0, im: 2.0 }`) and `Display` (`1+2i`, `1-2i`). The numeric suffix `i` is registered in core by `#[literal(number "i", priority = -1)]` and makes the imaginary `Complex<f64>`: `3.0 + 4.0i`, `1i`. A package's own `i` handler outranks it ([refs-lang 6.5.5](../../refs-lang/006-meta-system/605-literal-magic.md#655-multiple-handlers-ambiguity)).
