# 2.11. 转换 trait

`core::convert` 模块定义了一组把值从一种类型转换为另一种类型的 trait。Nabi 转换策略的核心原则是拒绝静默的信息丢失。因此 `.into()` 只允许无损转换；对于可能丢失信息的转换，用户必须显式选择策略。

[refs-lang 2.14 转换](../../refs-lang/002-types/214-cast-and-conversion.md)的语法分派到这些 trait。

## 2.11.1. 转换策略：四种转换

Nabi 按**意图**把转换分为四种。用户每次都必须写明适用哪一种。

| 种类         | 方法                                | 含义                                       | 失败时                      |
|--------------|-------------------------------------|--------------------------------------------|-----------------------------|
| **无损**     | `.into::<T>()`                      | 零信息丢失；总是成功                       | 编译错误（没有 `From` 实现）|
| **可失败**   | `.try_into::<T>()`                  | 可能丢失信息；由调用者处理                 | 抛出目标类型的 `TryFrom::Error` |
| **饱和**     | `.saturating_into::<T>()`           | 超出范围时钳制到 i32::MAX / MIN            | 显式饱和（用户的选择）      |
| **位**       | `.wrapping_into::<T>()` 或 `as T`   | 位重新解释：模运算 / 位模式                | 显式回绕（用户的选择）      |

每种转换的方法名本身就表明了意图。写 `.into()` 表示**无损**意图；写 `.saturating_into()` 表示**饱和**意图。编译器不猜测含义。

### 设计依据

静默饱和或静默回绕会在用户不知情的情况下改变值，产生难以调试的 bug。Nabi 同时吸取了 JS 引入 Number → BigInt 时的教训，以及 NumPy 从回绕改为 OverflowError 时的教训。

出于同样的原因，定宽算术（`+ - *`、一元 `-`）在所有构建配置中溢出时都会 panic，与 Swift 相同。这些检查代价很小：在 Apple M 系列上以 `-O2` 用 `bench/` 测量（退役指令数，取 3 次最佳），`fannkuch`（整数列表）+5.6%，`jsontokenizer`（字节扫描）+9.2%，`nbody`（f64）在噪声范围内（-2.8%）。需要最后几个百分点的程序可以用 `-C overflow-checks=no` 或 `nabi.nabion` 构建配置（`profile: { "release": { overflow_checks: false } }`，[refs-lang 9.10.3](../../refs-lang/009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）关闭它们，此时溢出会回绕。

## 2.11.2. `From<U>` trait：无损转换

```nabi
trait From<U> {
    fn from(value: U) -> Self;
}
```

`From::from(u)` 是总是成功且不丢失信息的转换。

### 示例

```nabi
let n: i32 = 42;
let big: i64 = i64::from(n);     // i32 → i64 (always lossless)

let x: int = int::from(42i32);   // i32 → int (lossless)
```

### 导入

`From` 在 [prelude](../000-meta/002-import.md) 中，所以不需要单独的 `use` 声明。

## 2.11.3. `Into` trait：`From` 的镜像

```nabi
trait Into {
    fn into<T: From<Self>>(self) -> T;
}

impl<S> Into for S {
    fn into<T: From<S>>(self) -> T { T::from(self) }
}
```

`Into` 是一个没有类型参数、只有一个泛型方法 `into` 的 trait。一个通用实现（blanket impl）给每个类型都提供了 `into`，所以 `x.into()` 就是 `T::from(x)`。目标类型 `T` 由期望类型（`let` 标注、实参位置等）或 `x.into::<T>()` 决定。用户不实现 `Into`，只编写 `From`。

可转换性约束写在 `From` 一侧。没有 `T: Into<U>` 这样的约束。

```nabi
fn widen<T, U: From<T>>(x: T) -> U { x.into() }      // OK (U: From<T>)
fn widen2<T: Into<U>, U>(x: T) -> U { x.into() }     // ERROR: Into takes no type argument
```

### 示例

```nabi
let n: i32 = 42;
let big: i64 = n.into();         // i32::into::<i64>() automatically, using From<i32> for i64
let big_typed: i64 = (42i32).into();
```

### 优先使用 `From`：设计模式

显式转换可以写成以下两种形式之一。

```nabi
// form 1: From directly
let big = i64::from(42i32);

// form 2: into method
let big: i64 = (42i32).into();
```

两种形式在语义上等价。用 `from` 时目标类型在调用处可见；用 `into` 时由**标注**决定目标类型。根据可读性或模式选择合适的一种。

### 无损性的编译时检查

尝试使用未实现 `From<U> for T` 的组合是编译错误。

```nabi
let big: i64 = 1000;
let small: i32 = big.into();   // compile error: no From<i64> for i32
                                //   (not lossless)
```

这种情况下必须显式选择 `try_into`、`saturating_into` 或 `as` 之一。

```nabi
let small: i32 = try big.try_into();       // throws, caller handles it
let small: i32 = big.saturating_into();    // explicit saturate
let small: i32 = big as i32;               // explicit wrap
```

## 2.11.4. `TryFrom<U>` trait：可失败转换

```nabi
trait TryFrom<U> {
    type Error = ConversionError;
    fn try_from(value: U) throws (Self::Error) -> Self;
}
```

`try_from` 在签名中表明转换可能失败。错误类型是关联类型 `Error`，其默认值是 [`ConversionError`](214-error.md)（省略它的实现获得默认值，[refs-lang 2.8.3](../../refs-lang/002-types/208-trait.md)）。数值转换的实现（整数 ↔ 整数、浮点数 → 整数、`int` / `float` 转换、`TryFrom<u32> for char`）声明 `type Error = OverflowError;`。由于签名带 `throws`，用户用 `try` 标记调用以传播错误，用 `catch` 处理，或用 `try?` 得到 `T | none`。

```nabi
let big: i64 = 9999999999;
let small_result: i32 = try i32::try_from(big);  // propagate with try
let small_catch: i32 = try i32::try_from(big) catch e: OverflowError { default };
let small_opt: i32 | none = try? i32::try_from(big);
```

### 自动的 `TryInto` 镜像

适用与 `From` ↔ `Into` 相同的模式。

```nabi
trait TryInto {
    fn try_into<T: TryFrom<Self>>(self) throws (T::Error) -> T;
}
impl<S> TryInto for S { ... }            // = try T::try_from(self)
```

数值 `try_from` 抛出的 `OverflowError`（[2.14](214-error.md)）中，`operation()` 是 `"try_from"`，`type_name()` 是目标类型的名称（`"i32"` 等）。

```nabi
let small: i32 = try big.try_into();     // = try i32::try_from(big)
```

## 2.11.5. `saturating_into` / `wrapping_into`：显式策略

对于可能丢失信息的转换，由用户显式写明改变值的策略。为此提供了两个方法。

```nabi
trait SaturatingFrom<U> {
    fn saturating_from(value: U) -> Self;    // out of range: nearest end
}
trait SaturatingInto {
    fn saturating_into<T: SaturatingFrom<Self>>(self) -> T;
}

trait WrappingFrom<U> {
    fn wrapping_from(value: U) -> Self;      // keeps the low bits (= as)
}
trait WrappingInto {
    fn wrapping_into<T: WrappingFrom<Self>>(self) -> T;
}
```

形状与 `From` / `Into` 一致。转换本身是 `SaturatingFrom` / `WrappingFrom` 的实现；`saturating_into` / `wrapping_into` 是由通用实现提供的泛型方法。

### `saturating_into`：钳制到范围两端

```nabi
let big: i64 = 9999999999;
let small: i32 = big.saturating_into();    //→ i32::MAX (= 2147483647)

let big_neg: i64 = -9999999999;
let small_neg: i32 = big_neg.saturating_into();  //→ i32::MIN
```

这种策略适用于 DSP、信号处理、UI 钳制等领域。

### `wrapping_into`：模运算

```nabi
let big: i64 = 0x100000001;
let small: i32 = big.wrapping_into();      //→ 1 (low 32 bits)
```

它与 `as i32` 含义相同，但以显式方法调用表达意图。这种策略适合哈希和位掩码领域。

## 2.11.6. `as T`：位重新解释 / 数值强制转换

`as` 关键字（[refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md)）是使数值 / 位转换**显式化**的手段。

```nabi
let n: i32 = 256;
let b: u8 = n as u8;             // low 8 bits = 0
let b2: u8 = 257 as u8;          // 1
let b_neg: u8 = -1i32 as u8;     // 255

let f: f64 = 1.7;
let i: i32 = f as i32;            // 1 (truncation, IEEE 754 → integer)
let big: i32 = 1e10 as i32;       // i32::MAX (saturates)
let nan: i32 = f64::NAN as i32;   // 0
let f2: f32 = 3.14f64 as f32;     // 3.14 → f32 (IEEE 754 narrowing)
```

`as` 的行为如下。

| 转换                                               | 含义                                               |
|----------------------------------------------------|----------------------------------------------------|
| 整数 → 同宽度的整数（有符号/无符号）               | 位模式相同；只改变符号的解释                       |
| 窄整数 → 宽整数                                    | 符号扩展（有符号）或零扩展（无符号）               |
| 宽整数 → 窄整数                                    | 截断（只保留低位）                                 |
| 整数 → 浮点数                                      | 转为 IEEE 754 表示（可能损失精度）                 |
| 浮点数 → 整数                                      | 向零截断；超出范围时饱和到 MIN/MAX，NaN → 0（所有优化级别） |
| 浮点数 → 相同/其他宽度的浮点数                     | IEEE 754 舍入                                      |
| `u8` → `char`                                      | 对应的码位（U+0000..=U+00FF）                      |
| `char` → 整数                                      | 码位（对窄整数截断）                               |

转换到 `char` 的 `as` 只能从 `u8` 出发。其他整数 / 浮点数 / `bool` / `c_char` 是类型错误；更宽的值用 `char::from_u32(n)`（`char | none`）转换（[2.15.4](215-string.md)）。转换到 `bool` 的 `as` 只能从 `bool` 出发；其他情况改用比较（`x != 0`、`x != 0.0`）。

### `as` 的限制

`as` 只限于数值和位转换。任意类型之间的转换会被拒绝。

```nabi
struct Vec3 { x: f64; y: f64; z: f64; }
let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let s: str = v as str;     // compile error: `as` rejects user structs
```

用户类型之间的转换必须用 `From` / `Into` trait 写明。

## 2.11.7. 字面量上下文的自动 From：编译时范围检查

对于带标注的字面量，编译时范围检查和无损转换一起适用。

```nabi
let x: i32 = 42;             // OK: 42 is in i32 range
let y: i32 = 9999999999;     // compile error: literal outside i32 range
let z: f32 = 3.14;           // OK: round to nearest f32
let w: u8 = 256;             // compile error: 256 ∉ u8 0..=255
```

从字面量选择实现的规则（如 `T::from(1)` / `1.into()`）遵循 [refs-lang 3.1.3](../../refs-lang/003-expressions/301-operator-dispatch.md#selecting-an-impl)。这一机制之所以可行，是因为字面量是编译时已知的值。转换运行时的值必须显式写出 `from`、`into`、`try_into` 或 `as` 之一。

## 2.11.8. `From` 实现矩阵：原始类型之间

标准库提供的 `From<U> for T` 实现列表如下（仅限无损转换）。

### 整数 → 整数（有符号）

| 源 \ 目标 | i8 | i16 | i32 | i64           | isize         | int |
|-----------|----|-----|-----|---------------|---------------|-----|
| `i8`      |    | ✓   | ✓   | ✓             | ✓             | ✓   |
| `i16`     |    |     | ✓   | ✓             | ✓             | ✓   |
| `i32`     |    |     |     | ✓             | ✓（目标≥32）  | ✓   |
| `i64`     |    |     |     |               |               | ✓   |
| `isize`   |    |     |     | ✓（目标≤64）  |               | ✓   |

### 整数 → 整数（无符号）

| 源 \ 目标 | u8 | u16 | u32 | u64           | usize         | int（总是无损） |
|-----------|----|-----|-----|---------------|---------------|-----------------------|
| `u8`      |    | ✓   | ✓   | ✓             | ✓             | ✓                     |
| `u16`     |    |     | ✓   | ✓             | ✓             | ✓                     |
| `u32`     |    |     |     | ✓             | ✓（目标≥32）  | ✓                     |
| `u64`     |    |     |     |               |               | ✓                     |
| `usize`   |    |     |     | ✓（目标≤64）  |               | ✓                     |

`u64` → `usize` 和 `i64` → `isize` 没有 `From`：它们在 32 位目标上是收窄转换，而在一个目标上能构建的转换必须在所有目标上都能构建。请使用 `TryFrom<u64> for usize` / `TryFrom<i64> for isize`（`usize::try_from(n)`），它们在所有目标上都存在，并且在 64 位目标上从不抛出。

### 无符号 → 有符号（仅限无损）

| 源 \ 目标 | i8 | i16 | i32 | i64 | isize | int |
|-----------|----|-----|-----|-----|-------|-----|
| `u8`      |    | ✓   | ✓   | ✓   | ✓     | ✓   |
| `u16`     |    |     | ✓   | ✓   | ✓     | ✓   |
| `u32`     |    |     |     | ✓   |       | ✓   |
| `u64`     |    |     |     |     |       | ✓   |

从无符号类型转换到相同或更小宽度的有符号类型（例如 u32 → i32）不保证无损，所以必须使用 `try_into` 或 `as`。`isize` 和 `usize` 在所有支持的目标上至少有 32 位，所以 `u8` / `u16` → `isize` 处处无损并具有 `From`，`i8` / `i16` / `i32` → `isize` 和 `u8` / `u16` / `u32` → `usize` 也是如此。`u32` → `isize` 只在 64 位目标上无损，所以与 `u64` → `usize` 一样只有 `TryFrom`（`isize::try_from(n)`）。具有 `From` 的类型对没有单独的 `TryFrom`。

### 整数 → 浮点数

| 源 \ 目标 | f32 | f64                         |
|-----------|-----|-----------------------------|
| `i8`      | ✓   | ✓                           |
| `i16`     | ✓   | ✓                           |
| `i32`     |     | ✓                           |
| `i64`     |     | （没有 `From`：损失精度）   |
| `u8`      | ✓   | ✓                           |
| `u16`     | ✓   | ✓                           |
| `u32`     |     | ✓                           |
| `u64`     |     | （没有 `From`：损失精度）   |

f32 的尾数为 24 位，f64 的尾数为 53 位，所以只有在该位宽之内才能保证无损。更宽的整数用 `as` 转换，结果舍入到最接近的值。

### 浮点类型之间

| 源 \ 目标 | f32 | f64 | float |
|-----------|-----|-----|-------|
| `f32`     |     | ✓   | ✓     |
| `f64`     |     |     | ✓     |

f64 → f32 是收窄转换，所以必须使用 `try_into`、`saturating_into` 或 `as` 之一。

### int / float 与定宽原始类型

| 源 \ 目标 | i32                   | i64             | f64                                             |
|-----------|-----------------------|-----------------|-------------------------------------------------|
| `int`     | 仅 `try_into`         | 仅 `try_into`   | `to_f64`、`try_into`、`saturating_into`（舍入） |
| `float`   | `try_into`（截断）    | `try_into`      | `to_f64`、`try_into`、`saturating_into`（舍入） |

`From` 无一例外地保证无损，所以 `int` 和 `float` 没有到 `f64` 的 `From`，也没有到 `f64` / `f32` 的 `WrappingFrom`（回绕保留低位，而浮点数没有低位）：`x as f64` 是编译错误，错误信息会指出 `to_f64`。三种形式都舍入到最接近的 `f64`（平局时取偶数），区别只在超出 `±f64::MAX` 时：`to_f64()` 给出 `±inf`，`try_into` 抛出 `OverflowError`（与 `TryFrom<f64> for f32` 的规则相同），`saturating_into` 给出 `±f64::MAX`。对 `f64` 来说过小的 `float` 在三种形式中都变为次正规数或 `±0`。

`int` → 定宽类型的转换不保证无损（`int` 可能超出定宽范围），所以必须使用 `try_into` 或 `saturating_into`。

反过来，定宽类型 → `int` 总是无损的（`From<i32> for int` 等全部已实现）。

## 2.11.9. `From` 实现：用户类型

用户类型之间的转换通过直接编写 `impl From<U> for T` 来定义。

```nabi
struct Celsius { degree: f64; }
struct Fahrenheit { degree: f64; }

impl From<Celsius> for Fahrenheit {
    fn from(c: Celsius) -> Fahrenheit {
        Fahrenheit { degree: c.degree * 9.0 / 5.0 + 32.0 }
    }
}

let c = Celsius { degree: 100.0 };
let f: Fahrenheit = c.into();   //→ Fahrenheit { degree: 212.0 }
```

通用的 `Into` 把 `c.into()` 路由到 `Fahrenheit::from(c)`，所以用户不需要编写 `Into`。

### `try_from` 遵循同样的模式

```nabi
impl TryFrom<str> for i32 {
    type Error = ParseError;
    fn try_from(s: str) throws (Self::Error) -> i32 {
        // i32::from_str(s): an integer string within i32 range, else ParseError
        ...
    }
}

let n: i32 = try "42".try_into();
```

每个定宽整数类型都以这种方式在其 `FromStr` 之上实现 `TryFrom<str>`：它就是 `FromStr::from_str`，所以空文本、无效数字或超出范围的值都抛出相同的 `ParseError`（0.5.9：由输入文本导致的失败是 `ParseError`）。浮点类型、`bool` 和 `char` 只提供 `FromStr`。

## 2.11.10. 算术结果的自动转换：没有

算术结果的自动 From（[refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md)）只在**字面量上下文**中适用。对值做算术的结果不会自动转换。

```nabi
let a: i32 = 5;
let b: i32 = 3;
let c: i32 = a + b;          // OK: i32 + i32 = i32

let big: i32 = a + b * 1000000;
// → OK if a + b * 1000000 is within i32 range; otherwise a panic (a wrap with overflow checks off)
// no silent saturate / silent promote

let auto: i64 = a + b;       // compile error: no silent promotion of an i32 result to i64
let auto: i64 = (a + b) as i64;  // explicit conversion OK
let auto: i64 = (a + b).into();  // explicit From<i32> for i64 (lossless)
```

## 2.11.11. 与其他语言的比较：设计依据

| 语言     | 转换模型                                                 | 与 Nabi 的区别                                                                    |
|----------|----------------------------------------------------------|-----------------------------------------------------------------------------------|
| Rust     | `From`/`Into` 无损，`as` 回绕，`try_into` 返回 Result    | 最接近 Nabi；Nabi 增加了显式的 `saturating_into`/`wrapping_into` 方法             |
| Swift    | 所有整数转换默认 trap，`.init(clamping:)` 需显式选择     | 定宽算术同样 trap（在所有构建配置中 panic）；转换用显式方法而不是会 trap 的初始化器 |
| Julia    | `Int32(big)` 会 trap，`convert` 函数族                   | Nabi 统一使用 trait 方法（`from`/`into`）                                         |
| Python   | `int(x)`/`float(x)` 函数，总是任意精度                   | Nabi 的 `int` / `float` 精神相同；定宽原始类型是另一个世界                        |
| C        | 隐式转换（`int` → `long` 自动进行）                      | Nabi 明确拒绝，阻止静默的类型混用                                                 |

从这一比较可以看出 Nabi 的关键设计决定：拒绝静默的信息丢失（受 Rust 影响）；通过显式区分四种转换（`into`、`try_into`、`saturating_into`、`as`）强制表达意图；只在字面量上下文中允许自动 From，对运行时的值要求显式转换；把 `int` / `float` 的自动提升与定宽原始类型分开，这一决定受 Python 和 Smalltalk 影响。

## 2.11.12. 模块对应

| Trait               | lang 槽                | 标准库位置                |
|---------------------|------------------------|---------------------------|
| `From<U>`           | （无）                 | `core::convert`     |
| `Into`              | （通用实现）           | `core::convert`     |
| `TryFrom<U>`        | （无）                 | `core::convert`     |
| `TryInto`           | （通用实现）           | `core::convert`     |
| `SaturatingFrom<U>` | （无）                 | `core::convert`     |
| `SaturatingInto`    | （通用实现）           | `core::convert`     |
| `WrappingFrom<U>`   | （无）                 | `core::convert`     |
| `WrappingInto`      | （通用实现）           | `core::convert`     |
| `AsBytes`           | （无）                 | `core::convert`     |

原始类型之间的每个无损转换实现都带有 `#[intrinsic("nabi.cast.FROM.TO")]`。

```nabi
impl From<i32> for i64 {
    #[intrinsic("nabi.cast.i32.i64")]
    fn from(value: i32) -> i64 { ... }
}
```

`as T` 的数值强制转换使用相同的内建函数；由编译器在无损分支和回绕分支之间选择。

## 2.11.13. `AsBytes` trait：字节视图

`core::convert::AsBytes` 是标准库中每个字节实参所接受的类型：`str::from_utf8`（[2.15.6](215-string.md#2156-utf-8-不变式进程内的每个字符串)）、`hex` 和 `base64`（[2.22](222-encoding.md)）、`crypto`（[2.21](221-crypto.md)）、`CString::new`（[1.7.2](../100-bare/107-ffi.md)）、`fs` 和 `net` 的写入函数（[2.23.3](223-io.md)）以及 `Slice<u8>::find`（[2.9.7](209-collections.md)）。这样的参数写作 `<B: AsBytes>`，每个字节实参一个类型参数，所以密钥和消息不必是相同的类型。

```nabi
trait AsBytes {
    fn byte_slice(self) -> Slice<u8>;      // the bytes as a view, no copy; need not be UTF-8
}
impl AsBytes for str;                      // its UTF-8 bytes
impl AsBytes for List<u8>;
impl AsBytes for Slice<u8>;
impl<const N: usize> AsBytes for [u8; N];  // a view of a copy (b"...", include_bytes!)
impl AsBytes for crypto::Secret;           // the bytes in place (2.21)
```

程序自己的类型只要实现了 `AsBytes`，就能在任何接受字节的地方传入：

```nabi
struct Frame { body: List<u8>; }
impl AsBytes for Frame {
    fn byte_slice(self) -> Slice<u8> { self.body[:] }
}

let tag = crypto::Hmac::mac(crypto::HashAlgorithm::Sha256, key, frame);
let text = encoding::hex::encode(frame);
```

`AsBytes` 只有这一条路径；`core::io` 不重新导出它。
