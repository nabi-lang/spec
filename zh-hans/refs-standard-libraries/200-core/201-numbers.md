# 2.1. 数值类型

`core::num` 模块定义了 nabi 的所有数值类型。Nabi 让来自两个世界的数并存，用户可以自由选择其中之一。

```
int / float    (auto-promote, arbitrary precision, opt-in)
fixed primitive (i8..i64, u8..u64, isize, usize, f32, f64; hardware-friendly, default)
```

两个世界**并存**。无后缀字面量遵循期望类型；没有期望类型时为 `i64` / `f64`（[refs-lang 1.6.3](../../refs-lang/001-lexical/106-numeric-literals.md#163-无后缀字面量)）。`int` / `float` 只在写明类型的地方使用。

## 2.1.1. 数的两个世界：为何分开

### `int` / `float`：自动提升的世界

这个世界以任意精度为目标。溢出永远不会发生；只有达到内存上限时，计算才会以 OOM 结束。小的值经过小整数优化（small-int optimization），以硬件定宽形式存储；值变大时会自动提升为堆上的 BigInt。`int` / `float` 值是不可变的，堆上的 BigInt 从不就地修改，只会被共享（这也是它们是 `Copy` 的原因）。由此用户得到算术正确性的保证：`I32_MAX + 1` 永远不会悄悄回绕成荒谬的结果。

### 定宽原始类型：贴近硬件的世界

与之相对，定宽原始类型是贴近硬件的世界。它们包括 `i32` / `f64` 等 15 种类型，其语法由 [refs-lang 2.1 类型表达式](../../refs-lang/002-types/201-type-expressions.md)定义。它们适合需要精确控制内存和性能的场景（FFI 接口、SIMD、大型数组、热循环），其溢出策略是明确的（见 [2.3 溢出行为](203-ops.md#溢出行为)）。

### 哪一个是默认

无后缀字面量（`42`、`3.14`）在没有任何东西确定其类型时是 **`i64` / `f64`**（用 `let` 绑定的字面量从后续使用中取得类型，[1.6.3](../../refs-lang/001-lexical/106-numeric-literals.md#163-无后缀字面量)）。只有写明 `int` / `float` 的地方才使用自动提升的世界。

```nabi
let total: int = 0;               // int (auto-promote), explicit
for x in xs { total = total + x; }
// total is auto-promoted even past the i64 range; no silent overflow

let n = 0;                        // i64 (nothing fixes it)
let small: i32 = 0;               // i32: follows the expected type
```

## 2.1.2. `int`：任意精度整数

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

这些方法沿用定宽整数的名字和 panic 行为。位数和位索引是 `u64`，与 `bit_length` 一样：一个值可以超过 2^32 位（定宽的规则见下文）。与位宽绑定的位方法（`leading_zeros`、`rotate_left`、`swap_bytes` 等）在没有位宽时没有意义，因此不提供。`count_ones` 统计的是 `|self|`（即 Python 的 `bit_count`），因为负数有无穷多个 1 位。表示方式体现在名字上：`to_le_bytes` / `from_le_bytes` 使用二进制补码，与定宽整数的字节一致；`_unsigned` 形式使用绝对值，遇到负值会触发 panic（`Int::to_le_bytes_unsigned: negative value; use to_le_bytes for two's complement or call .abs() first`）。字节数取值所允许的最短长度：`128` 是 `[128, 0]`，无符号时是 `[128]`；`-1` 是 `[255]`；零是 `[0]`。`write_*` 形式则填满固定宽度，即 `out`（`List<u8>` 或其视图）的每个字节：`-2` 写成四个字节，小端为 `[254, 255, 255, 255]`；值所需的字节多于 `out` 时触发 panic（`Int::write_le_bytes: the value needs 2 bytes but `out` has 1; pass a view of at least 2 bytes`）。`from_*` 形式接受任意字节（[`AsBytes`](211-conversion.md)）：`List<u8>`、`Slice<u8>`、`[u8; N]` 或 `str`。`pow` 接受 `u32` 指数：除 0 和 ±1 之外，任何底数配上更大的指数都会得到至少 2^32 位。`modpow` 接受 `int` 指数，因为其结果总小于 `|m|`；结果的符号与 `m` 相同，与 [`%%`](203-ops.md#modulo向下取整取模) 一样。负指数是错误（没有模逆元）。零是 `Int::default()`（或 `Int::zero()`，见 2.1.5 的 `Zero`）；`Int` 没有 `new`。除以零会以 `attempt to divide an int by zero` 触发 panic，运算符 `/`、`%`、`%%` 和 `div_rem`、`div_euclid`、`rem_euclid` 都是如此。检查式的 `to_i32` / `to_i64` / `to_u64` 作为固有方法与 `TryFrom` 并存（[0.5.11](../000-meta/005-notation.md)）。

`Int` 定义在 `core::num::bigint::Int` 中（`Float` 定义在 `core::num::bigfloat::Float` 中）。`#[keyword("int")]` 把标识符 `int` 注册为类型关键字，这一注册对所有包生效。因此用户代码把这个类型写成小写的 `int`，而不是 PascalCase 的 `Int`（以 [refs-lang 6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) 为准）。编译器通过 `#[lang("int_type")]` 找到 `int` 类型。

`int` 的关键性质如下。在内存中，小的值经过小整数优化，直接存放在一个 64 位字中，大的值自动提升为堆上的 BigInt 表示；这一切对用户都是透明的。表示总是规范的：处于 i64 范围内的值是小整数形式，否则是 BigInt；算术结果回到 i64 范围时，也回到小整数形式。相等的值有相同的表示。所用的 **lang 槽**是 `int_type`（[0.4 lang 槽表](../000-meta/004-lang-slot-table.md)）。只有期望类型为 `int` 时，无后缀整数字面量才是 `int`。

### 算术

`int` 之间的算术从不回绕，也不饱和。

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

`int` 实现了算术 trait（[2.3 ops](203-ops.md)）`Add`、`Sub`、`Mul`、`Div`、`Rem`、`Modulo` 和 `Neg`。`Div` 是整数除法（截断），因此 `7 / 2 = 3`，`-7 / 2 = -3`。位运算 trait `BitAnd`、`BitOr`、`BitXor` 和 `BitNot` 按符号位无限重复的二进制补码运算（`~x == -x - 1`）。移位是 `Shl<u32>` 和 `Shr<u32>`：右侧是 `u32` 而不是 `int`；`x << n == x * 2^n`，`x >> n == floor(x / 2^n)`（算术右移）。比较 trait（[2.5 cmp](205-cmp.md)）`PartialEq`、`Eq`、`PartialOrd`、`Ord` 和 `Hash` 都已实现，`Clone`、`Copy`、`Default`（零）、`Zero`、`One`、`Display`、`Debug` 和 `FromStr` 也是如此。

### 主要用途

`int` 适用于许多场景。作为 REPL 或 Jupyter 中的算术默认类型，它保证正确性，而不必每次都写类型标注；数值模拟中累加的和或积不必担心溢出。密码运算和大整数处理只用标准库即可完成，不必依赖单独的 BigInt 库。

### 局限

`int` 有其局限。在内存方面，提升为 BigInt 需要堆分配，因此在热循环和大型数组中推荐使用定宽原始类型。在性能方面，它在小整数范围内接近定宽原始类型，但提升为 BigInt 之后必然更慢。在 FFI 方面，`extern "C"` 函数只接受定宽原始类型，因此 `int` 必须先显式转换为 `i64` 等类型再传递。

## 2.1.3. `float`：指数不受限的浮点数

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

`#[keyword("float")]` 注册 `float` 关键字，并把它指向结构体 `Float`。因此用户代码只看到小写的 `float`。

`float` 值有 53 位有效数字（与 f64 精度相同）和不受限的（i64）指数。它通常使用 f64 表示；当有限结果超出 f64 范围（上溢）或过于接近 0（下溢）时，它会提升为 BigFloat 表示，而不是变成 `inf` / `0`。

- `inf` / `NaN` 只来自除以零，或来自 `inf` / `NaN` 操作数。
- `%` 是精确的 fmod。
- `Display` 对 f64 范围内的值按 f64 的方式输出；提升后的值带指数输出，如 `1.5e400`（`{:g}` 得到 `1.5e+400`）。

### 明确的局限

`float` 的局限必须明确说明。它的精度是 53 位而非无限，因此与 IEEE 754 f64 有相同的舍入误差。例如，`0.1 + 0.2` 不等于 `0.3` 的情况在 Nabi 中同样会发生。`Eq` 也没有实现：为了处理 NaN，`float` 不实现 `Eq` 标记 trait，只实现 `PartialEq`。完整的层次结构见 [2.5 cmp](205-cmp.md)。

### lang 槽

编译器通过 `float_type` 找到 `float` 类型（[0.4 lang 槽表](../000-meta/004-lang-slot-table.md)）。只有期望类型为 `float` 时，无后缀浮点字面量才是 `float`。

## 2.1.4. 定宽原始类型

[refs-lang 2.1 类型表达式](../../refs-lang/002-types/201-type-expressions.md)定义了 15 种原始类型的**名字**。本节在此基础上定义标准库的方法和行为。

### 整数（10 种）

| 类型    | 位宽                            | 范围           |
|---------|---------------------------------|----------------|
| `i8`    | 8                               | -128 ~ 127     |
| `i16`   | 16                              | -32768 ~ 32767 |
| `i32`   | 32                              | -2³¹ ~ 2³¹-1   |
| `i64`   | 64                              | -2⁶³ ~ 2⁶³-1   |
| `isize` | 目标指针宽度（32 或 64）        | 取决于目标     |
| `u8`    | 8                               | 0 ~ 255        |
| `u16`   | 16                              | 0 ~ 65535      |
| `u32`   | 32                              | 0 ~ 2³²-1      |
| `u64`   | 64                              | 0 ~ 2⁶⁴-1      |
| `usize` | 目标指针宽度                    | 取决于目标     |

### 浮点数（2 种）

| 类型  | 位宽      | 标准                                 |
|-------|-----------|--------------------------------------|
| `f32` | 32        | IEEE 754 binary32（单精度）          |
| `f64` | 64        | IEEE 754 binary64（双精度）          |

16 位浮点数 `f16` 和 `bf16` 是标准库为存储而添加的类型（[见下文](#f16-和-bf1616-位存储用浮点数)）。

### 浮点语义

`f32` 和 `f64` 在所有目标上行为相同：

- **舍入**。`+ - * /`、`%`（精确，即 C 的 `fmod`）、`√` / `sqrt`、`mul_add`、所有转换为浮点数的 `as`、所有浮点字面量以及 `parse` 都舍入到最近值，平局时取偶数。字面量或解析的字符串从其十进制值舍入一次：`0.1f32` 是最接近 0.1 的 `f32`，而不是对 `0.1f64` 再舍入一次。没有其他舍入模式，也无法更改。
- **不做缩并**。`a * b + c` 是先对积舍入，再对和舍入。只有 `mul_add` 把两者融合为一次舍入。优化器和任何后端都不会对浮点运算做缩并、重结合或重排序：对浮点数求和的循环不会被向量化（各通道会改变顺序），常量求值得到的位与程序计算出的位相同。
- **保留次正规数**。任何目标都不会把次正规数输入或结果冲刷为零。运行时在所有目标上都让浮点控制状态（MXCSR、FPCR、FPSCR）保持 IEEE 默认值，Nabi 代码无法更改它。
- **NaN**。与 NaN 的任何比较都为假，`!=` 除外。运算返回哪个 NaN 是未指定的：其载荷和符号可能因目标而异（x86 让 `0.0 / 0.0` 为负）。`-x`、`abs` 和 `copysign` 只改变符号位，对 NaN 也是如此，`to_bits` / `from_bits` 保留每一位。`f32::NAN` 和 `f64::NAN` 是符号位清零的固定静默 NaN。
- **有符号零**。`-0.0 == 0.0`，`1.0 / -0.0` 是 `-inf`。
- **`min` / `max`** 是 IEEE 754 的 `minimumNumber` / `maximumNumber`：一个操作数为 NaN 时得到另一个（两者都是 NaN 时才得到 NaN），并且 `-0.0` 排在 `+0.0` 之前，因此在所有目标上 `(0.0).min(-0.0)` 都是 `-0.0`，`(-0.0).max(0.0)` 都是 `0.0`。`total_cmp` 是 IEEE 全序（[2.1.11](#2111-数值-trait浮点排序区间复数)）。
- **浮点数到整数**的 `as` 向零截断，并在整数类型的边界处饱和；NaN 得到 0。整数到浮点数舍入到最近值，平局时取偶数。
- **库函数**。`sin`、`exp`、`ln`、`pow` 及其他超越函数来自目标的 C 库：不保证正确舍入，最后一位可能因目标而异。常量求值不会折叠它们。

`float`（2.1.3）在 `f64` 的范围内遵循相同的规则。

### `f16` 和 `bf16`：16 位存储用浮点数

```nabi
#[lang("f16_type")]
#[keyword("f16")]
pub struct F16 { bits: u16; }      // IEEE 754 binary16

#[lang("bf16_type")]
#[keyword("bf16")]
pub struct BF16 { bits: u16; }     // bfloat16
```

| 类型   | 有效数字位       | 指数位        | 最大值    | 最小正规数      | `EPSILON` |
|--------|------------------|---------------|-----------|-----------------|-----------|
| `f16`  | 11（存储 10）    | 5             | 65504     | 2^-14           | 2^-10     |
| `bf16` | 8（存储 7）      | 8（与 `f32` 相同） | 约 3.39e38 | 2^-126     | 2^-7      |

`f16` 和 `bf16`（`core::num::half`）是存储类型：它们把大型数组（模型权重、图像、传感器数据）的内存减半，并通过 `f32` 进行计算。每个都是 2 字节、对齐为 2，`List<f16>` 或 `Array<bf16, 2>` 每个元素存储 2 字节，`Array::zeros` / `ones` 也适用于它们。

- **字面量**。位于 `f16` / `bf16` 位置的无后缀字面量，或带后缀 `f16` / `bf16` 的字面量（`0.1f16`、`2bf16`），其值从十进制舍入一次得到。舍入为无穷大的字面量是编译错误（`let h: f16 = 70000.0;`）；整数字面量必须精确（`let h: f16 = 2049;` 是错误，与 `f32` 相同）。
- **`as`** 可以与所有整数和浮点原始类型相互转换，也可以在 `f16` 和 `bf16` 之间转换。转换为 16 位浮点数时直接从源值舍入到最近值，平局时取偶数：`f64` 或 `i64` 绝不会先舍入为 `f32`。从 16 位浮点数转换为 `f32` / `f64` 是精确的，转换为整数时与其 `f32` 值的 `as` 相同。`int`、`float`、`bool` 和 `char` 不能用 `as` 转换（写成 `x.to_f64() as f16`）。
- **算术**。`+ - * / % %%`、它们的复合赋值、一元 `-` 和 `√` 在 `f32` 中计算，再舍入到该类型一次。对于 `+ - * / %` 和 `√`，这恰好就是 16 位格式的正确舍入 IEEE 结果（`f32` 的有效数字位多于它的两倍加二）。混合操作数与原始类型一样被拒绝（2.1.6）。没有超越函数方法，也没有 `FloatingPoint` 实现：请转换为 `f32`（`h.to_f32().sin()`）。
- **比较**（`==`、`<` 等）与其 `f32` 值的比较相同：遵循 IEEE，NaN 与任何值都不相等，`-0.0 == 0.0`。两种类型都不是 `Eq`、`Ord` 或 `Hash`；排序用 `xs.sort_by(f16::total_cmp)`。字面量模式不匹配它们：请在守卫中用 `==` 比较。
- **方法和常量**。`from_bits` / `to_bits`（`u16` 位模式）、`from_f32`、`from_f64`、`from_i64`、`from_u64`、`to_f32`、`to_f64`、`is_nan`、`is_infinite`、`is_finite`、`is_normal`、`is_sign_positive`、`is_sign_negative`、`abs`、`copysign`、`min`、`max`（与 `f32` 的相同）、`total_cmp`；`EPSILON`、`MAX`、`MIN`、`MIN_POSITIVE`、`NAN`、`INFINITY`、`NEG_INFINITY`。
- **trait**。`Clone`、`Copy`、`Default`（`0.0`）、`Zero`、`One`、`PartialEq`、`PartialOrd`、上述运算符 trait、`Display`、`Debug`、`FromStr`（从十进制舍入一次），以及 `f32` 和 `f64` 的 `From<f16>` / `From<bf16>`（精确）。
- **文本**。`Display` 输出能读回同一值的最短数字（`0.1`、`65500`），从不使用指数，以及 `inf` / `NaN`；`Debug` 为整数值加上 `.0`。格式说明符（`{h:.3}`、`{h:e}`）按 `f64` 的方式格式化精确值。
- **硬件**。`f16` <-> `f32` 在 CPU 支持转换的地方使用 CPU：x86-64 上目标特性包含 `f16c` 时用 F16C（`-C target-cpu=x86-64-v3`），arm64 上用 `fcvt`。其他情况下是运行时调用，结果相同。`bf16` 的转换是内联的几条整数运算。
- **C**。在 `extern "C"` 签名中，`f16` 是 C 的 `_Float16`，`bf16` 是 `__bf16`，与 C 一样通过浮点寄存器传递（[refs-lang 8.6.3](../../refs-lang/008-abi/806-calling-convention.md#863-struct-实参--返回小-struct-规则)）。目标的 C 没有这种类型时（powerpc64le 和 wasm32；s390x 上的 `__bf16`；i686 上的半精度结果），该签名是错误：请改为把 `h.to_bits()` 作为 `uint16_t` 传递。

### 布尔 / 字符（3 种）

| 类型     | 含义                                                                                   |
|----------|----------------------------------------------------------------------------------------|
| `bool`   | `true` / `false`                                                                       |
| `char`   | Unicode 标量值（4 字节），编译器原始类型。方法见 [2.15](215-string.md)                |
| `c_char` | 与 C ABI 兼容的 char（1 字节，符号取决于目标，按其字节 `c as u8` 比较）；仅用于 FFI   |

### 两组：语言内置与标准库添加

本章的数值 / 布尔 / 字符类型分为**两组**。

| 组                                   | 成员                                                                                                  | 注册方式                                           |
|--------------------------------------|-------------------------------------------------------------------------------------------------------|----------------------------------------------------|
| **16 种语言内置**                    | `i8`/`i16`/`i32`/`i64`, `u8`/`u16`/`u32`/`u64`, `isize`/`usize`, `f32`/`f64`, `bool`, `char`, `c_char`, `str` | 由语言识别；不需要 `#[keyword]`                    |
| **4 种标准库添加**                   | `int`, `float`, `f16`, `bf16`                                                                 | 用 `#[keyword(...)]` 注册                          |

语言内置类型是编译器（词法分析器 / 解析器）直接识别的词法单元，因此标准库不把它们注册为关键字。这自动保证了用户无法重新定义这些名字。

与之相对，标准库添加的类型（`int` / `float` / `f16` / `bf16`）用 `#[keyword(...)]` 注册。`#[keyword]` 属性附加在本模块的 `Int` / `Float` / `F16` / `BF16` 定义上（以 [refs-lang 6.6](../../refs-lang/006-meta-system/606-keyword-magic.md) 为准）。

### 内置原始类型在标准库中的形式

```nabi
impl i32 { ... }              // inherent constants and methods (below)
impl Add for i32 { ... }      // trait impls: Add, PartialOrd, Display, Clone, ...
impl PartialEq for c_char { ... }
```

标准库不为内置原始类型声明结构体。它只为固有方法和 trait（`Add`、`Display`、`Clone` 等）附加 `impl` 块。识别类型本身是编译器的职责。

### 算术 / 比较 trait 实现

每个原始类型都实现 [2.3 ops](203-ops.md)、[2.4 ops-assign](204-ops-assign.md) 和 [2.5 cmp](205-cmp.md) 的 trait。每个运算符方法以及每个 `PartialEq` / `PartialOrd` 方法（包括 `char` 和 `c_char`）都带有 `#[intrinsic("nabi.OP.TYPE")]`，直接绑定到编译器后端的 PrimOp。唯一的例外是 `bool` 的 `PartialOrd`，它是普通的函数体（`false < true`）。`c_char` 在所有目标上都按其字节比较（`c as u8`，2.15.5）：`255u8 as c_char > 1u8 as c_char`。

### 方法（按原始类型）

每个整数 / 浮点类型都提供以下方法（示例为 `i32`）。

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

`i8`、`i16`、`i64` 和 `isize` 拥有相同的一组方法，各按自己的位宽。无符号类型（`u8`..`u64`、`usize`）拥有相同的一组，但不含与符号相关的方法（`abs`、`unsigned_abs`、`signum`、`is_positive`、`is_negative`、`checked_isqrt`、`checked_abs`、`saturating_neg`、`saturating_abs`、`wrapping_abs`）；`abs_diff` 返回同一类型，`checked_neg` 除 0 之外对所有值都是 `none`。它们另外提供与 2 的幂有关的方法。

位数或位索引的类型与 `BITS` 和移位量相同：对每个定宽整数都是 `u32`（`count_ones`、`leading_zeros`、`trailing_zeros`、`ilog2`、`rotate_left(n: u32)`、`pow(exp: u32)`、`wrapping_shl(n: u32)`），与整数自身的位宽无关；`trailing_zeros(0)` 为 `BITS`。`int` 则使用 `u64`，因为一个值可以超过 2^32 位，并且 0 的 `trailing_zeros` 为 `none`。`<<` / `>>` 运算符不同：对定宽整数，右侧的类型与左侧相同（`Shl for T`，因此是 `i64 << i64`）；对 `int`，右侧为 `u32`（`Shl<u32>`）。

```nabi
impl u32 {
    fn is_power_of_two(self) -> bool;
    fn next_power_of_two(self) -> u32;                 // smallest power of two >= self; panics on overflow
    fn checked_next_power_of_two(self) -> u32 | none;
}
```

整数类型的最小值 / 最大值是关联常量 `i32::MIN` / `i32::MAX`，而不是函数。无参数的固有方法 `min()` / `max()` 会遮蔽同名的 `Ord::min` / `Ord::max`（`a.min(b)`）。

浮点类型（`f32`、`f64`）提供以下常量和方法（`f32` 拥有相同的一组，以 `f32` 代替 `f64`，位模式为 `u32`）。

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

## 2.1.5. 字面量 → 哪种类型

[refs-lang 1.6 数值字面量](../../refs-lang/001-lexical/106-numeric-literals.md)的规则如下：

| 形式      | 得到的类型                                              |
|-----------|---------------------------------------------------------|
| `42`      | 期望类型（整数 / 浮点类型），否则为 `i64`               |
| `3.14`    | 期望类型（浮点类型），否则为 `f64`                      |
| `42i32`   | `i32`                                                   |
| `42u64`   | `u64`                                                   |
| `3.14f32` | `f32`                                                   |
| `3.14f64` | `f64`                                                   |
| `0.5f16`  | `f16`                                                   |
| `0.5bf16` | `bf16`                                                  |
| `'A'`     | `char`                                                  |
| `c'A'`    | `c_char`（原始类型）                                    |
| `0xFF`    | 与十进制相同：期望类型，否则为 `i64`                    |
| `0xFFi32` | `i32`                                                   |

### 期望类型与范围检查

范围在编译期检查。

```nabi
let x: i32 = 42;            // i32
let y: i32 = 9999999999;    // compile error (outside the i32 range)
let z: f32 = 3.14;          // f32
let f: f64 = 1;             // f64: an integer literal fits a floating-point slot
let w: u8 = 256;            // compile error (256 ∉ u8 range 0..=255)
let i: i32 = 1.5;           // compile error: a floating-point literal cannot fill an integer slot
let big: int = 9999999999;  // int
```

完整的转换矩阵见 [2.11 转换](211-conversion.md)。

### `Zero` / `One`

```nabi
trait Zero { fn zero() -> Self; }       // additive identity
trait One  { fn one() -> Self; }        // multiplicative identity
```

每个数值原始类型以及 `int` / `float` 都实现这两个 trait。`Iterator::sum` / `product` 使用它们（[2.7.1](207-iter.md#271-iterator步进)）。

## 2.1.6. 算术结果类型

### `int + int = int`

由于适用自动提升，结果总是 `int` 本身。不会推断出定宽类型。

```nabi
let a: int = 1000;
let b: int = 2000;
let c = a + b;       // int (3000)
```

### `i32 + i32 = i32`

保持定宽。想要更宽类型的用户必须使用显式转换。

```nabi
let a: i32 = 5;
let b: i32 = 3;
let c: i32 = a + b;   // i32

let wider: i64 = (a as i64) + (b as i64);   // i64
```

### `int + i32`（混合：拒绝）

```nabi
let a: int = 5;       // int
let b: i32 = 3;       // i32
let c = a + b;        // compile error: type mismatch
```

这里需要显式转换。

```nabi
let c1 = a + int::from_i32(b);    // int (a + int(3))
let c2 = (a.to_i32()? + b);       // i32 (OK if a is within the i32 range)
```

这一策略阻止了类型的悄然混用。与 Julia 不同，Nabi 不做自动提升；由用户说明使用哪个世界。

编译器对混合数值给出的提示会把较窄的一侧转换为较宽的类型，因此建议的转换不会丢失值：整数转为浮点数，任何原始类型转为 `int` 或 `float`，两个有符号或两个无符号整数中较窄的转为较宽的。有符号类型可以容纳无符号类型；两者位宽相同且低于 64 位时，两者都转为宽一级的有符号类型。

```nabi
let a: i32 = 1;
let b: i64 = 5000000000;
let c = a + b;   // help: convert one side: `(a as i64) + b`
let u: u32 = 3;
let d = a + u;   // help: convert both sides: `(a as i64) + (u as i64)`
```

## 2.1.7. 定宽原始类型 ↔ int 转换

定宽原始类型与 `int` 之间只能显式转换。

| 转换                     | 方法                                  | 失败时                       |
|--------------------------|---------------------------------------|------------------------------|
| `i32` → `int`            | `.into::<int>()` 或 `int::from_i32`   | 总是成功（无损）             |
| `int` → `i32`            | `.try_into::<i32>()`                  | 抛出 `OverflowError`         |
| `int` → `i32`（饱和）    | `.saturating_into::<i32>()`           | i32::MIN 或 MAX              |
| `int` → `i32`（回绕）    | `.wrapping_into::<i32>()`             | 模运算                       |
| `int` → `f64`            | `int::to_f64`                         | 舍入到最近值；超过 `f64::MAX` 时为 ±inf |
| `int` → `f64`（检查）    | `.try_into::<f64>()`                  | 舍入；超过 `f64::MAX` 时抛出 `OverflowError` |
| `int` → `f64`（饱和）    | `.saturating_into::<f64>()`           | 舍入；超出范围时为 ±`f64::MAX` |

`float` → `f64` 有同样的三种形式。两者都没有 `From`（舍入转换不是无损的，[2.11.8](211-conversion.md)），也没有 `as`（`as` 保留低位，而浮点数没有低位）。

## 2.1.8. niche 优化（简述）

[refs-lang 2.7 niche](../../refs-lang/002-types/207-niche-optimization.md) 定义了这一机制。获得自动 niche 的原始类型如下：

`bool | none` 占 1 字节（niche 值为 `2`），`Ptr<T> | none` / `ConstPtr<T> | none` 与指针同宽（null 即 none）（[refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-自动-niche)）。`i8`..`i64`、`u8`..`u64`、`f32` 和 `f64` 本身没有自动 niche，因为每种位模式都是有效值。

需要选择加入的 niche 时，请使用 [2.2 non-zero](202-non-zero.md) 的 `NonZero<T>` / `NonNan<F>` / `Finite<F>` / `Positive<T>` 包装类型。

## 2.1.9. lang 槽映射

16 种语言内置类型（`i8`..`i64`、`u8`..`u64`、`isize`、`usize`、`f32`、`f64`、`bool`、`c_char`、`char`、`str`）由编译器按名字识别。没有标准库项为它们绑定 lang 槽，也没有为它们声明结构体；库只添加 `impl` 块（2.1.4），因此不需要 `#[keyword]` 或 `#[lang]`。

标准库添加的 4 种类型是用 `#[keyword(...)]` 注册、并用 `#[lang(...)]` 绑定到各自 lang 槽的 PascalCase 结构体。

| 原始类型  | lang 槽      | 结构体名    | 注册方式              |
|-----------|--------------|-------------|-----------------------|
| `int`     | `int_type`   | `Int`       | `#[keyword("int")]`   |
| `float`   | `float_type` | `Float`     | `#[keyword("float")]` |
| `f16`     | `f16_type`   | `F16`       | `#[keyword("f16")]`   |
| `bf16`    | `bf16_type`  | `BF16`      | `#[keyword("bf16")]`  |

`Range<T>`（`a..b`）由 [2.7 iter](207-iter.md) 定义。

## 2.1.10. 与其他语言比较：设计理由

Nabi 的双世界模型（自动提升的 `int` 与定宽原始类型）借鉴了多种语言。Smalltalk 的 SmallInteger ↔ LargeInteger 是自动提升的原型，Lisp 和 Scheme 的数值塔（fixnum / bignum / rational / real）是另一个重要参考。**Python int** 以任意精度为默认，但没有定宽原始类型；**Julia** 则以定宽原始类型为默认。在 Nabi 中，字面量同样默认是定宽的（`i64` / `f64`），`int` 需要选择加入。

与各语言的区别：**Python** 只提供 int；由于缺少定宽类型，FFI 和性能关键的工作要依赖 NumPy**。Julia** 与 Nabi 一样默认定宽，需要任意精度时必须显式使用 `BigInt`**。Rust** 只有定宽类型，因此用户必须始终写出类型**。Nabi** 让两个世界并存：字面量遵循期望类型（没有时为 `i64` / `f64`），需要任意精度的地方写 `int` / `float`。

## 2.1.11. 数值 trait、浮点排序、区间、复数

**`Num`**。`trait Num: Copy + Zero + One + Add<Output = Self> + Sub<Output = Self> + Mul<Output = Self> + Div<Output = Self> + PartialEq + PartialOrd {}`，由十二种定宽原始类型实现：针对其中任一类型编写代码时，写 `T: Num`。混合操作数仍被拒绝（2.1.6）。

**`FloatingPoint`**。`trait FloatingPoint: Num + Neg<Output = Self>` 由 `f32` 和 `f64` 实现（`float` 不实现；`float` 的类型名为 `Float`，因此这个 trait 采用 Swift 的名字）。它带有常量 `NAN`、`INFINITY`、`NEG_INFINITY`、`EPSILON`、`PI`、`E`，以及方法 `abs sqrt cbrt exp ln log2 log10 powi pow sin cos tan asin acos atan atan2 sinh cosh tanh hypot floor ceil round trunc signum mul_add min max is_nan is_infinite is_finite total_cmp`（每个都是该类型自己的同名方法），还有 `to_f64`、`from_f64`（舍入到最近值）和 `from_usize`。

**`total_cmp`**。`f32::total_cmp` / `f64::total_cmp(self, other) -> Ordering` 是 IEEE 754 的 `totalOrder`：`-NaN < -inf < .. < -0.0 < +0.0 < .. < inf < +NaN`。`xs.sort_by(f64::total_cmp)` 可以对任何浮点列表排序；`f64` 仍然没有 `Ord`，`==` 仍遵循 IEEE。`f64::NAN` 和 `f32::NAN`（`0x7fc0_0000`）在所有目标上符号位都清零，因此排在 `inf` 之后；算术产生的 NaN（`0.0 / 0.0`）可能带任一符号（x86 会设置符号位），负的 NaN 排在最前。

**`linspace` / `arange`**（`core::num`，对 `T: FloatingPoint` 泛型，返回 `List<T>`）：

```nabi
linspace(0.0, 1.0, 5)     //→ [0.0, 0.25, 0.5, 0.75, 1.0]: n values, both ends included
arange(0.0, 1.0, 0.25)    //→ [0.0, 0.25, 0.5, 0.75]: start + i*step while before stop
```

`linspace(start, stop, n)` 给出 `start + i * step`，其中 `step = (stop - start) / (n - 1)`，最后一个值恰好为 `stop`（`n == 1` 时只有 `start`，`0` 时为空）。`arange(start, stop, step)` 给出 `ceil((stop - start) / step)` 个值 `start + i * step`，每个值都从 `i` 计算，因此误差不会累积；步长为零或 NaN 时触发 panic。`Range<f64>` 不是迭代器（[2.7.3](207-iter.md#273-step单步抽象)）。

**`gcd` / `lcm`**。每个定宽整数类型都有 `gcd(other)`，即最大公约数，从不为负（两个零时为 `0`；`12i32.gcd(-18) == 6`）；以及 `lcm(other)`，即最小公倍数，从不为负（任一为 0 时为 `0`）。结果放不下时触发 panic，并提示改用 `int` 计算（`i64::MIN.gcd(0)`、溢出的 `lcm`）；`int` 有自己的 `gcd` / `lcm`（2.1.2）。

**`is_close`**。`f64::is_close(self, other, rel: f64 = 1e-9, abs: f64 = 0.0, equal_nan: bool = false) -> bool` 即 Python 的 `math.isclose`：两者相等，或两者都有限且 `|a - b| <= max(rel * max(|a|, |b|), abs)` 时为 `true`。它对两个值是对称的。默认的 `rel` 要求大约九位相同的数字；在零附近只有 `abs` 起作用（`1e-12.is_close(0.0)` 为 `false`，`1e-12.is_close(0.0, abs: 1e-9)` 为 `true`）。无穷大只与自身接近，NaN 与任何值都不接近，除非设置了 `equal_nan` 且两者都是 NaN。`f32::is_close` 形式相同，默认 `rel` 为 2^-20（末位的八个单位；1e-9 低于 `f32` 的精度）。数组形式是 `Array::is_close` / `all_close`（[2.9.14](209-collections.md#2914-arrayt-const-n-usizen-维数组)）。

**`comb` / `perm` / `fsum`**（`core::num`）：

```nabi
fn comb(n: u64, k: u64) -> int;                     // n! / (k! (n - k)!), 0 when k > n; Python math.comb
fn perm(n: u64, k: u64) -> int;                     // n! / (n - k)!, 0 when k > n; Python math.perm
fn fsum<I: IntoIterator<Item = f64>>(xs: I) -> f64; // correctly rounded sum; Python math.fsum
```

`comb` 和 `perm` 是精确的（`comb(100, 50) == 100891344545564193334812497256`）。`fsum` 把精确的和保存为一组互不重叠的部分和（Shewchuk 1997），最后舍入一次（舍入到最近值，平局时取偶数），因此结果与顺序和数量级无关（`fsum([0.1; 10]) == 1.0`，`fsum([1e100, 1.0, -1e100]) == 1.0`）。值中有无穷大或 NaN 时得到普通的 IEEE 结果（`inf - inf` 为 NaN），累加和超出 `f64` 范围时也是如此（`[1e308, 1e308, -1e308]` 得到 `inf`）。它的开销约为普通循环的三倍；`Array::sum` 采用成对求和，速度快。

**`div_floor` / `rem_floor`**。每个整数类型都有 `div_floor(rhs)`，即向负无穷舍入的商（Python 的 `//`），以及 `rem_floor(rhs)`，即符号与除数相同的余数（Python 的 `%`、Nabi 的 `%%`）：`(-7).div_floor(2) == -4`，`(-7).rem_floor(2) == 1`。对无符号类型，它们就是 `/` 和 `%`。

**`Complex<T>`**（`core::num::Complex`）是 `struct Complex<T> { pub re: T; pub im: T; }`，在 `T: FloatingPoint` 时提供 `new`、`i()`、`from_polar`、`abs`、`norm_sqr`、`arg`、`conj`、`recip`、`exp`、`ln`、`sqrt`（主平方根，每个部分都有完整的相对精度，包括极小的虚部）、`powi`、`pow`、`is_nan`；复数之间以及复数与任一侧的实数 `T` 之间的 `+ - * /` 和一元 `-`（`2.0 * z`、`z / 2.0`；实数在左侧仅限 `f32` 和 `f64`）、`Zero`、`One`、`PartialEq`、`Debug`（`Complex { re: 1.0, im: 2.0 }`）和 `Display`（`1+2i`、`1-2i`）。数值后缀 `i` 由 `#[literal(number "i", priority = -1)]` 在 core 中注册，生成虚数 `Complex<f64>`：`3.0 + 4.0i`、`1i`。包自己的 `i` 处理器优先于它（[refs-lang 6.5.5](../../refs-lang/006-meta-system/605-literal-magic.md#655-多个处理函数歧义)）。
