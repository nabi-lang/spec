# 2.2. Niche 包裝型別：`NonZero` / `NonNan` / `Finite` / `Positive`

`core::num::niche` 提供把值限制在某個子集內的包裝型別。每個包裝型別都是帶有 `#[niche(..)]`（[refs-lang 2.7.4](../../refs-lang/002-types/207-niche-optimization.md#274-選用的-nicheniche)）的 newtype，被排除的值成為 `T | none` 的 niche。

這些包裝型別不在 prelude 中：要透過 `num::niche::` 或 `use core::num::niche::NonZero;` 指名。每個包裝型別都實作 `Clone`、`Copy`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash`、`Display` 和 `Debug`（`NonZero(42)`），並以 `From` 無損地轉換回其 `T`。

## 2.2.1. 大小

帶有 niche 的 `T | none` 與 `T` 大小相同（[refs-lang 2.7.2](../../refs-lang/002-types/207-niche-optimization.md#272-自動-niche)）。`none` 以 niche 值表示；實際的位元由實作選定。

| 型別                     | `T \| none` 大小 | `none` 的 niche 值 |
|--------------------------|-----------------|-----------------------|
| `i32 \| none`             | 8 B（標籤）       | 無                  |
| `NonZero<i32> \| none`    | 4 B             | 0                     |
| `f64 \| none`             | 16 B（標籤）      | 無                  |
| `NonNan<f64> \| none`     | 8 B             | NaN                   |
| `Finite<f64> \| none`     | 8 B             | NaN、±∞               |
| `Positive<i32> \| none`   | 4 B             | 負值       |
| `Positive<u32> \| none`   | 8 B（標籤）       | 無                  |

當 niche 值為 0（`NonZero`）時，酬載位元不變，因此 `NonZero<T> | none` 以整數形式跨越 C ABI，其中 0 代表 `none`。

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

只接受整數型別。浮點數型別請使用 `NonNan` / `Finite`。

| 運算                   | 結果                                            |
|-----------------------------|---------------------------------------------------|
| `NonZero<T> * NonZero<T>`   | `NonZero<T>`。若乘積環繞為 0 則觸發 panic（`NonZero::mul: the product wrapped to zero; use `checked_mul` on the values`，`#[track_caller]`） |
| `NonZero<T> + - / NonZero<T>` | `T`                                             |
| `T / NonZero<T>`            | `T`。不會除以零                          |

`TryFrom<T>` 遇到 0 時擲回 `ZeroError`。`ZeroError` 是定義在此模組中的單元結構（`core::num::niche::ZeroError`），並實作 `Error`（[2.14.4](214-error.md#2144-內建錯誤型別)）。`From<NonZero<T>> for T` 是無損的。

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

- 實作 `Eq`、`Ord`、`Hash`。`-0.0 == 0.0`，且兩者的雜湊值相等。
- `+ - * /` 的結果是 `F`（∞ − ∞ 之類的結果可能是 NaN）。
- `TryFrom<F>` 遇到 NaN 時擲回 `ConversionError`。

## 2.2.4. `Finite<F>`：禁止 NaN 與 ±∞

```nabi
#[niche(F::NAN, F::INFINITY, F::NEG_INFINITY)]
struct Finite<F>(F);

impl<F: NicheFloat> Finite<F> {
    fn new(value: F) -> Finite<F> | none;
    unsafe fn new_unchecked(value: F) -> Finite<F>;    // UB if NaN / ±∞
    fn get(self) -> F;
}
```

- 實作 `Eq`、`Ord`、`Hash`。`-0.0 == 0.0`。
- `+ - *` 產生 `Result<Finite<F>, OverflowError>`（結果為 ±∞ 時為 `Err`）：運算子不能擲回錯誤（[2.3](203-ops.md)），所以可能失敗的運算子回傳 `Result`。
- 不提供 `/`：除以零會得到 NaN 或 ±∞，因此請把 `get()` 的值相除，再以 `Finite::new` 檢查。
- `TryFrom<F>` 遇到 NaN / ±∞ 時擲回 `ConversionError`。

## 2.2.5. `Positive<T>`：禁止負值

```nabi
#[niche(T::MIN..=-1)]
struct Positive<T>(T);

impl<T: NicheInt> Positive<T> {
    fn new(value: T) -> Positive<T> | none;            // none if negative; 0 is included
    unsafe fn new_unchecked(value: T) -> Positive<T>;
    fn get(self) -> T;
}
```

值為 0 或以上。無號的 `T` 沒有負值，因此沒有 niche。`TryFrom<T>` 遇到負值時擲回 `ConversionError`，而 `From<Positive<T>> for T` 是無損的。

## 2.2.6. `new_unchecked`

`new_unchecked` 是 `unsafe fn`。違反不變式是未定義行為，且 `T | none` 可能把該值讀作 `none`。

```nabi
use core::num::niche::NonZero;

let n: i32 = compute();                                // the caller guarantees it is not 0
let nz: NonZero<i32> = unsafe { NonZero::new_unchecked(n) };
```

## 2.2.7. 驗證

- 在 const 位置，`NonZero::new(0)` 之類的運算式在編譯時求值為 `none`。若附加 `?? panic(..)`，則為編譯錯誤（[refs-lang 5.2.4](../../refs-lang/005-modules/502-const.md#524-常數運算式與求值器)）。
- 在執行時，驗證使用 `new`（`T | none`）和 `try_from`（擲回錯誤）。

## 2.2.8. 定義摘要

| 型別          | 約束        | `#[niche]`                                 |
|---------------|--------------|--------------------------------------------|
| `NonZero<T>`  | `NicheInt`   | `0`                                        |
| `Positive<T>` | `NicheInt`   | `T::MIN..=-1`                              |
| `NonNan<F>`   | `NicheFloat` | `F::NAN`                                   |
| `Finite<F>`   | `NicheFloat` | `F::NAN, F::INFINITY, F::NEG_INFINITY`     |

## 2.2.9. 與其他語言的比較

| 面向       | Rust                          | Nabi                                   |
|--------------|-------------------------------|---------------------------------------|
| `NonZero<T>` | `NonZero<T>`（整數）       | `NonZero<T: NicheInt>`                |
| `NonNan` / `Finite` | 無（外部 crate）  | 內建                              |
| 自動 niche | `Option<Box<T>>` = `Box<T>` | `T \| none`                            |
| 使用者 niche   | 不穩定的屬性            | `#[niche(..)]`                         |
