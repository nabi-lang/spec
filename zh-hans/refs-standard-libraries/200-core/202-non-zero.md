# 2.2. niche 包装类型：`NonZero` / `NonNan` / `Finite` / `Positive`

`core::num::niche` 提供把值限制在某个子集内的包装类型。每个包装类型都是带有 `#[niche(..)]` 的 newtype（[refs-lang 2.7.4](../../refs-lang/002-types/207-niche-optimization.md#274-opt-in-niches-niche)），被排除的值成为 `T | none` 的 niche。

这些包装类型不在 prelude 中：通过 `num::niche::` 或 `use core::num::niche::NonZero;` 来指名。每个包装类型都实现 `Clone`、`Copy`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash`、`Display` 和 `Debug`（`NonZero(42)`），并可以用 `From` 无损地转换回它的 `T`。

## 2.2.1. 大小

带有 niche 的 `T | none` 与 `T` 大小相同（[refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-automatic-niches)）。`none` 由 niche 值表示，实际的位由实现选择。

| 类型                     | `T \| none` 的大小 | `none` 的 niche 值 |
|--------------------------|-----------------|-----------------------|
| `i32 \| none`             | 8 B（标签）       | 无                  |
| `NonZero<i32> \| none`    | 4 B             | 0                     |
| `f64 \| none`             | 16 B（标签）      | 无                  |
| `NonNan<f64> \| none`     | 8 B             | NaN                   |
| `Finite<f64> \| none`     | 8 B             | NaN、±∞               |
| `Positive<i32> \| none`   | 4 B             | 负值       |
| `Positive<u32> \| none`   | 8 B（标签）       | 无                  |

当 niche 值为 0（`NonZero`）时，载荷的位不变，因此 `NonZero<T> | none` 以整数的形式跨越 C ABI，其中 0 表示 `none`。

## 2.2.2. `NonZero<T>`：禁止零

```nabi
#[sealed]
trait NicheInt { .. }            // implemented by the 10 integer types

#[niche(0)]
struct NonZero<T>(T);

impl<T: NicheInt> NonZero<T> {
    fn new(value: T) -> NonZero<T> | none;
    unsafe fn new_unchecked(value: T) -> NonZero<T>;   // UB if value == 0
    fn get(self) -> T;
}
```

只接受整数类型。浮点类型使用 `NonNan` / `Finite`。

| 操作                   | 结果                                            |
|-----------------------------|---------------------------------------------------|
| `NonZero<T> * NonZero<T>`   | `NonZero<T>`。乘积回绕为 0 时触发 panic（`NonZero::mul: the product wrapped to zero; use `checked_mul` on the values`，`#[track_caller]`） |
| `NonZero<T> + - / NonZero<T>` | `T`                                             |
| `T / NonZero<T>`            | `T`。不会除以零                          |

`TryFrom<T>` 遇到 0 时抛出 `ZeroError`。`ZeroError` 是本模块定义的单元结构体（`core::num::niche::ZeroError`），实现了 `Error`（[2.14.4](214-error.md#2144-built-in-error-types)）。`From<NonZero<T>> for T` 是无损的。

```nabi
use core::num::niche::NonZero;

let n: NonZero<i32> = NonZero::new(42) ?? panic("zero");
let nz: NonZero<i32> = try 7.try_into();
let v: i32 = nz.into();
```

## 2.2.3. `NonNan<F>`：禁止 NaN

```nabi
#[sealed]
trait NicheFloat { .. }          // implemented by f32, f64; its helper methods are #[internal]

#[niche(F::NAN)]
struct NonNan<F>(F);

impl<F: NicheFloat> NonNan<F> {
    fn new(value: F) -> NonNan<F> | none;
    unsafe fn new_unchecked(value: F) -> NonNan<F>;    // UB if NaN
    fn get(self) -> F;
}
```

- 实现 `Eq`、`Ord`、`Hash`。`-0.0 == 0.0`，且二者的哈希相等。
- `+ - * /` 的结果是 `F`（∞ − ∞ 等可能是 NaN）。
- `TryFrom<F>` 遇到 NaN 时抛出 `ConversionError`。

## 2.2.4. `Finite<F>`：禁止 NaN 和 ±∞

```nabi
#[niche(F::NAN, F::INFINITY, F::NEG_INFINITY)]
struct Finite<F>(F);

impl<F: NicheFloat> Finite<F> {
    fn new(value: F) -> Finite<F> | none;
    unsafe fn new_unchecked(value: F) -> Finite<F>;    // UB if NaN / ±∞
    fn get(self) -> F;
}
```

- 实现 `Eq`、`Ord`、`Hash`。`-0.0 == 0.0`。
- `+ - *` 产生 `Result<Finite<F>, OverflowError>`（结果为 ±∞ 时为 `Err`）：运算符不能抛出错误（[2.3](203-ops.md)），所以可能失败的运算符返回 `Result`。
- 不提供 `/`：除以零会得到 NaN 或 ±∞，所以应当对 `get()` 的值做除法，再用 `Finite::new` 检查。
- `TryFrom<F>` 遇到 NaN / ±∞ 时抛出 `ConversionError`。

## 2.2.5. `Positive<T>`：禁止负值

```nabi
#[niche(T::MIN..=-1)]
struct Positive<T>(T);

impl<T: NicheInt> Positive<T> {
    fn new(value: T) -> Positive<T> | none;            // none if negative; 0 is included
    unsafe fn new_unchecked(value: T) -> Positive<T>;
    fn get(self) -> T;
}
```

值为 0 或更大。无符号的 `T` 没有负值，因此没有 niche。`TryFrom<T>` 遇到负值时抛出 `ConversionError`，`From<Positive<T>> for T` 是无损的。

## 2.2.6. `new_unchecked`

`new_unchecked` 是 `unsafe fn`。违反不变式是未定义行为，`T | none` 可能把该值读作 `none`。

```nabi
use core::num::niche::NonZero;

let n: i32 = compute();                                // the caller guarantees it is not 0
let nz: NonZero<i32> = unsafe { NonZero::new_unchecked(n) };
```

## 2.2.7. 验证

- 在常量位置，`NonZero::new(0)` 等在编译期求值为 `none`。附加 `?? panic(..)` 时是编译错误（[refs-lang 5.2.4](../../refs-lang/005-modules/502-const.md#524-constant-expressions-and-the-evaluator)）。
- 在运行时，用 `new`（`T | none`）和 `try_from`（抛出错误）进行验证。

## 2.2.8. 定义汇总

| 类型          | 约束        | `#[niche]`                                 |
|---------------|--------------|--------------------------------------------|
| `NonZero<T>`  | `NicheInt`   | `0`                                        |
| `Positive<T>` | `NicheInt`   | `T::MIN..=-1`                              |
| `NonNan<F>`   | `NicheFloat` | `F::NAN`                                   |
| `Finite<F>`   | `NicheFloat` | `F::NAN, F::INFINITY, F::NEG_INFINITY`     |

## 2.2.9. 与其他语言的比较

| 方面       | Rust                          | Nabi                                   |
|--------------|-------------------------------|---------------------------------------|
| `NonZero<T>` | `NonZero<T>`（整数）       | `NonZero<T: NicheInt>`                |
| `NonNan` / `Finite` | 无（外部 crate）  | 内置                              |
| 自动 niche | `Option<Box<T>>` = `Box<T>` | `T \| none`                            |
| 用户 niche   | 不稳定属性            | `#[niche(..)]`                         |
