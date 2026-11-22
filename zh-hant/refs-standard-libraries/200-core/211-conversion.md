# 2.11. 轉換 trait

`core::convert` 模組定義一組把值從一個型別轉移到另一個型別的 trait。Nabi 轉換政策的核心原則是拒絕無聲的資訊遺失。因此 `.into()` 只允許無損轉換；對可能遺失資訊的轉換，使用者必須明確選擇政策。

[refs-lang 2.14 轉換](../../refs-lang/002-types/214-cast-and-conversion.md)的語法分派到這些 trait。

## 2.11.1. 轉換政策：四種類別

Nabi 依**意圖**把轉換分成四種類別。使用者每次都必須指明適用哪一種。

| 類別         | 方法                              | 意義                                    | 失敗時                  |
|--------------|-------------------------------------|--------------------------------------------|-----------------------------|
| **無損** | `.into::<T>()`                      | 零資訊遺失；必定成功     | 編譯錯誤（沒有 `From` 實作） |
| **可能失敗** | `.try_into::<T>()`                  | 可能遺失資訊；由呼叫端處理    | 擲回目標的 `TryFrom::Error` |
| **飽和** | `.saturating_into::<T>()`           | 超出範圍時夾限到 i32::MAX / MIN      | 明確的飽和（使用者的選擇） |
| **位元**      | `.wrapping_into::<T>()` 或 `as T`   | 位元重新解讀：模運算 / 位元樣式  | 明確的環繞（使用者的選擇） |

每種類別的方法名稱本身就表明意圖。寫 `.into()` 表示**無損**意圖；寫 `.saturating_into()` 表示**飽和**意圖。編譯器不猜測意義。

### 設計理由

無聲的飽和或無聲的環繞會在使用者不知情的情況下改變值，產生難以偵錯的錯誤。Nabi 同時採納了 JS 在引入 Number → BigInt 時學到的教訓，以及 NumPy 從環繞改為 OverflowError 時學到的教訓。

基於相同的理由，固定寬度算術（`+ - *`、一元 `-`）在每個建置設定檔中都會在溢位時觸發 panic，與 Swift 相同。這些檢查的成本很低：在 Apple M 系列上以 `-O2` 用 `bench/` 量測（退休指令數，取 3 次中最佳），`fannkuch`（整數串列）+5.6%，`jsontokenizer`（位元組掃描）+9.2%，`nbody`（f64）在雜訊範圍內（-2.8%）。需要最後幾個百分點的程式可以用 `-C overflow-checks=no` 或 `nabi.nabion` 建置設定檔（`profile: { "release": { overflow_checks: false } }`，[refs-lang 9.10.3](../../refs-lang/009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）關閉檢查，此時溢位會環繞。

## 2.11.2. `From<U>` trait：無損轉換

```nabi
trait From<U> {
    fn from(value: U) -> Self;
}
```

`From::from(u)` 是必定成功且無資訊遺失的轉換。

### 範例

```nabi
let n: i32 = 42;
let big: i64 = i64::from(n);     // i32 → i64 (always lossless)

let x: int = int::from(42i32);   // i32 → int (lossless)
```

### 匯入

`From` 在 [prelude](../000-meta/002-import.md) 中，因此不需要另外的 `use` 宣告。

## 2.11.3. `Into` trait：`From` 的鏡像

```nabi
trait Into {
    fn into<T: From<Self>>(self) -> T;
}

impl<S> Into for S {
    fn into<T: From<S>>(self) -> T { T::from(self) }
}
```

`Into` 是沒有型別參數、只有單一泛型方法 `into` 的 trait。一個全面實作讓每個型別都擁有 `into`，因此 `x.into()` 就是 `T::from(x)`。目標型別 `T` 由預期型別（`let` 標註、引數位置等）或 `x.into::<T>()` 決定。使用者不實作 `Into`；只撰寫 `From`。

可轉換性的約束寫在 `From` 那一側。沒有 `T: Into<U>` 這種約束。

```nabi
fn widen<T, U: From<T>>(x: T) -> U { x.into() }      // OK (U: From<T>)
fn widen2<T: Into<U>, U>(x: T) -> U { x.into() }     // ERROR: Into takes no type argument
```

### 範例

```nabi
let n: i32 = 42;
let big: i64 = n.into();         // i32::into::<i64>() automatically, using From<i32> for i64
let big_typed: i64 = (42i32).into();
```

### 優先使用 `From`：設計模式

明確的轉換可以寫成下列兩種形式之一。

```nabi
// form 1: From directly
let big = i64::from(42i32);

// form 2: into method
let big: i64 = (42i32).into();
```

兩種形式在語意上等價。使用 `from` 時，目標型別在呼叫處可見；使用 `into` 時，由**標註**決定目標型別。依可讀性或模式選擇合適的一種。

### 無損性的編譯時檢查

嘗試未實作 `From<U> for T` 的組合是編譯錯誤。

```nabi
let big: i64 = 1000;
let small: i32 = big.into();   // compile error: no From<i64> for i32
                                //   (not lossless)
```

此時必須明確選擇 `try_into`、`saturating_into` 或 `as` 之一。

```nabi
let small: i32 = try big.try_into();       // throws, caller handles it
let small: i32 = big.saturating_into();    // explicit saturate
let small: i32 = big as i32;               // explicit wrap
```

## 2.11.4. `TryFrom<U>` trait：可能失敗的轉換

```nabi
trait TryFrom<U> {
    type Error = ConversionError;
    fn try_from(value: U) throws (Self::Error) -> Self;
}
```

`try_from` 在簽章中表明轉換可能失敗。錯誤型別是關聯型別 `Error`，其預設為 [`ConversionError`](214-error.md)（省略它的實作會得到預設值，[refs-lang 2.8.3](../../refs-lang/002-types/208-trait.md)）。數值轉換的實作（整數 ↔ 整數、浮點數 → 整數、`int` / `float` 的轉換、`TryFrom<u32> for char`）宣告 `type Error = OverflowError;`。由於簽章是 `throws`，使用者以 `try` 標記呼叫以傳遞錯誤、以 `catch` 處理，或以 `try?` 取得 `T | none`。

```nabi
let big: i64 = 9999999999;
let small_result: i32 = try i32::try_from(big);  // propagate with try
let small_catch: i32 = try i32::try_from(big) catch e: OverflowError { default };
let small_opt: i32 | none = try? i32::try_from(big);
```

### 自動的 `TryInto` 鏡像

適用與 `From` ↔ `Into` 相同的模式。

```nabi
trait TryInto {
    fn try_into<T: TryFrom<Self>>(self) throws (T::Error) -> T;
}
impl<S> TryInto for S { ... }            // = try T::try_from(self)
```

在數值 `try_from` 擲回的 `OverflowError`（[2.14](214-error.md)）中，`operation()` 是 `"try_from"`，`type_name()` 是目標型別的名稱（`"i32"` 等）。

```nabi
let small: i32 = try big.try_into();     // = try i32::try_from(big)
```

## 2.11.5. `saturating_into` / `wrapping_into`：明確的政策

對可能遺失資訊的轉換，使用者明確指明改變值的政策。為此提供兩個方法。

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

其形狀與 `From` / `Into` 一致。轉換本身是 `SaturatingFrom` / `WrappingFrom` 的實作；`saturating_into` / `wrapping_into` 是由全面實作提供的泛型方法。

### `saturating_into`：夾限到範圍兩端

```nabi
let big: i64 = 9999999999;
let small: i32 = big.saturating_into();    //→ i32::MAX (= 2147483647)

let big_neg: i64 = -9999999999;
let small_neg: i32 = big_neg.saturating_into();  //→ i32::MIN
```

此政策在 DSP、訊號處理、UI 夾限等領域很有用。

### `wrapping_into`：模運算

```nabi
let big: i64 = 0x100000001;
let small: i32 = big.wrapping_into();      //→ 1 (low 32 bits)
```

它的意義與 `as i32` 相同，但以明確的方法呼叫表達意圖。此政策適合雜湊與位元遮罩領域。

## 2.11.6. `as T`：位元重新解讀 / 數值強制轉型

`as` 關鍵字（[refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md)）是讓數值 / 位元轉換**明確化**的手段。

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

`as` 的行為如下。

| 轉換                                         | 意義                                            |
|----------------------------------------------------|----------------------------------------------------|
| 整數 → 相同寬度的整數（有號/無號） | 相同的位元樣式；只改變正負號的解讀方式  |
| 窄整數 → 寬整數                      | 符號延伸（有號）或零延伸（無號）     |
| 寬整數 → 窄整數                      | 截斷（只保留低位元）                         |
| 整數 → 浮點數                           | 轉為 IEEE 754 表示（可能遺失精度）    |
| 浮點數 → 整數                           | 向零截斷；超出範圍時飽和到 MIN/MAX，NaN → 0（所有最佳化等級） |
| 浮點數 → 相同/不同寬度的浮點數 | IEEE 754 捨入                                    |
| `u8` → `char`                                      | 對應的碼位（U+0000..=U+00FF）                  |
| `char` → 整數                                   | 碼位（轉為窄整數時截斷）        |

轉為 `char` 的 `as` 只能從 `u8` 進行。其他整數 / 浮點數 / `bool` / `c_char` 是型別錯誤；較寬的值以 `char::from_u32(n)`（`char | none`）轉換（[2.15.4](215-string.md)）。轉為 `bool` 的 `as` 只能從 `bool` 進行；請改用比較（`x != 0`、`x != 0.0`）。

### `as` 的限制

`as` 僅限於數值與位元轉換。任意型別之間的轉換會被拒絕。

```nabi
struct Vec3 { x: f64; y: f64; z: f64; }
let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let s: str = v as str;     // compile error: `as` rejects user structs
```

使用者型別之間的轉換必須以 `From` / `Into` trait 表明。

## 2.11.7. 字面值情境的自動 From：編譯時範圍檢查

對帶有標註的字面值，編譯時範圍檢查與無損轉換會一併適用。

```nabi
let x: i32 = 42;             // OK: 42 is in i32 range
let y: i32 = 9999999999;     // compile error: literal outside i32 range
let z: f32 = 3.14;           // OK: round to nearest f32
let w: u8 = 256;             // compile error: 256 ∉ u8 0..=255
```

從字面值選擇實作的規則，如 `T::from(1)` / `1.into()`，遵循 [refs-lang 3.1.3](../../refs-lang/003-expressions/301-operator-dispatch.md#選擇-impl)。此機制之所以可行，是因為字面值是編譯時已知的值。轉換執行時的值則必須明確寫出 `from`、`into`、`try_into` 或 `as` 之一。

## 2.11.8. `From` 實作矩陣：基本型別之間

標準函式庫提供的 `From<U> for T` 實作列表如下（僅限無損）。

### 整數 → 整數（有號）

| from \ to | i8 | i16 | i32 | i64           | isize         | int |
|-----------|----|-----|-----|---------------|---------------|-----|
| `i8`      |    | ✓   | ✓   | ✓             | ✓             | ✓   |
| `i16`     |    |     | ✓   | ✓             | ✓             | ✓   |
| `i32`     |    |     |     | ✓             | ✓（目標≥32） | ✓   |
| `i64`     |    |     |     |               |               | ✓   |
| `isize`   |    |     |     | ✓（目標≤64） |               | ✓   |

### 整數 → 整數（無號）

| from \ to | u8 | u16 | u32 | u64           | usize         | int（必定無損） |
|-----------|----|-----|-----|---------------|---------------|-----------------------|
| `u8`      |    | ✓   | ✓   | ✓             | ✓             | ✓                     |
| `u16`     |    |     | ✓   | ✓             | ✓             | ✓                     |
| `u32`     |    |     |     | ✓             | ✓（目標≥32） | ✓                     |
| `u64`     |    |     |     |               |               | ✓                     |
| `usize`   |    |     |     | ✓（目標≤64） |               | ✓                     |

`u64` → `usize` 與 `i64` → `isize` 沒有 `From`：在 32 位元目標上它們是窄化轉換，而能在一個目標上建置的轉換必須能在每個目標上建置。請使用 `TryFrom<u64> for usize` / `TryFrom<i64> for isize`（`usize::try_from(n)`），它們在所有目標上都存在，且在 64 位元目標上永遠不會擲回錯誤。

### 無號 → 有號（僅限無損）

| from \ to | i8 | i16 | i32 | i64 | isize | int |
|-----------|----|-----|-----|-----|-------|-----|
| `u8`      |    | ✓   | ✓   | ✓   | ✓     | ✓   |
| `u16`     |    |     | ✓   | ✓   | ✓     | ✓   |
| `u32`     |    |     |     | ✓   |       | ✓   |
| `u64`     |    |     |     |     |       | ✓   |

從無號型別轉為相同或更小寬度的有號型別（例如 u32 → i32）不保證無損，因此必須使用 `try_into` 或 `as`。`isize` 與 `usize` 在每個支援的目標上至少為 32 位元，因此 `u8` / `u16` → `isize` 在所有目標上都是無損的並具有 `From`，`i8` / `i16` / `i32` → `isize` 與 `u8` / `u16` / `u32` → `usize` 也是如此。`u32` → `isize` 只在 64 位元目標上無損，因此與 `u64` → `usize` 一樣只有 `TryFrom`（`isize::try_from(n)`）。具有 `From` 的組合沒有另外的 `TryFrom`。

### 整數 → 浮點數

| from \ to | f32 | f64                         |
|-----------|-----|-----------------------------|
| `i8`      | ✓   | ✓                           |
| `i16`     | ✓   | ✓                           |
| `i32`     |     | ✓                           |
| `i64`     |     | （沒有 `From`：會遺失精度） |
| `u8`      | ✓   | ✓                           |
| `u16`     | ✓   | ✓                           |
| `u32`     |     | ✓                           |
| `u64`     |     | （沒有 `From`：會遺失精度） |

f32 有 24 位元的尾數，f64 有 53 位元的尾數，因此只有在該位元寬度內才能保持無損。更寬的整數以 `as` 轉換，捨入到最接近的值。

### 浮點數型別之間

| from \ to | f32 | f64 | float |
|-----------|-----|-----|-------|
| `f32`     |     | ✓   | ✓     |
| `f64`     |     |     | ✓     |

f64 → f32 是窄化轉換，因此必須使用 `try_into`、`saturating_into` 或 `as` 之一。

### int / float 與固定基本型別

| from \ to | i32                   | i64             | f64                                             |
|-----------|-----------------------|-----------------|-------------------------------------------------|
| `int`     | 僅 `try_into`       | 僅 `try_into` | `to_f64`、`try_into`、`saturating_into`（捨入） |
| `float`   | `try_into`（截斷） | `try_into`     | `to_f64`、`try_into`、`saturating_into`（捨入） |

`From` 毫無例外地無損，因此 `int` 與 `float` 沒有轉為 `f64` 的 `From`，也沒有轉為 `f64` / `f32` 的 `WrappingFrom`（環繞保留低位元，而浮點數沒有低位元）：`x as f64` 是會指出 `to_f64` 的編譯錯誤。三種形式都捨入到最接近的 `f64`（同距時取偶數），只在超過 `±f64::MAX` 時有差異：`to_f64()` 得到 `±inf`，`try_into` 擲回 `OverflowError`（即 `TryFrom<f64> for f32` 的規則），`saturating_into` 得到 `±f64::MAX`。對 `f64` 而言過小的 `float` 在三者中都會成為次正規數或 `±0`。

`int → fixed` 不保證無損（`int` 可能超出固定寬度型別的範圍），因此必須使用 `try_into` 或 `saturating_into`。

反之，`fixed → int` 必定無損（`From<i32> for int` 等全都已實作）。

## 2.11.9. `From` 實作：使用者型別

使用者型別之間的轉換以直接撰寫 `impl From<U> for T` 來定義。

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

全面實作的 `Into` 把 `c.into()` 導向 `Fahrenheit::from(c)`，因此使用者不撰寫 `Into`。

### `try_from` 遵循相同的模式

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

每個固定寬度整數型別都以這種方式在其 `FromStr` 之上實作 `TryFrom<str>`：它就是 `FromStr::from_str`，因此空文字、無效數字或超出範圍的值都擲回相同的 `ParseError`（0.5.9：由輸入文字造成的失敗是 `ParseError`）。浮點數型別、`bool` 與 `char` 只提供 `FromStr`。

## 2.11.10. 算術結果的自動轉換：不存在

算術結果的自動 From（[refs-lang 2.14](../../refs-lang/002-types/214-cast-and-conversion.md)）只適用於**字面值情境**。對值做算術的結果不會自動轉換。

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

## 2.11.11. 與其他語言的比較：設計理由

| 語言 | 轉換模型                                         | 與 Nabi 的差異                                                               |
|----------|----------------------------------------------------------|-----------------------------------------------------------------------------------|
| Rust     | `From`/`Into` 無損，`as` 環繞，`try_into` 回傳 Result     | 最接近 Nabi；Nabi 另外加入明確的 `saturating_into`/`wrapping_into` 方法       |
| Swift    | 所有整數轉換預設 trap，`.init(clamping:)` 選擇性加入 | 固定寬度算術也會 trap（在每個建置設定檔中觸發 panic）；轉換是明確的方法，而不是會 trap 的初始化器 |
| Julia    | `Int32(big)` 會 trap，`convert` 函式系列            | Nabi 統一使用 trait 方法（`from`/`into`）                                      |
| Python   | `int(x)`/`float(x)` 函式，永遠是任意精度  | Nabi 的 `int` / `float` 精神相同；固定基本型別則是另一個世界   |
| C        | 隱含轉換（`int` → `long` 自動進行）       | Nabi 明確拒絕，杜絕無聲的型別混用                            |

由此比較可見 Nabi 的關鍵設計決策：拒絕無聲的資訊遺失（受 Rust 影響）；透過明確區分四種類別（`into`、`try_into`、`saturating_into`、`as`）強制表達意圖；只在字面值情境允許自動 From，並要求對執行時的值明確轉換；以及把 `int` / `float` 的自動提升與固定基本型別分開，這項決策受 Python 與 Smalltalk 影響。

## 2.11.12. 模組對應

| Trait               | lang 槽位              | 標準函式庫位置 |
|---------------------|------------------------|---------------------------|
| `From<U>`           | （無）                 | `core::convert`     |
| `Into`              | （全面實作）         | `core::convert`     |
| `TryFrom<U>`        | （無）                 | `core::convert`     |
| `TryInto`           | （全面實作）         | `core::convert`     |
| `SaturatingFrom<U>` | （無）                 | `core::convert`     |
| `SaturatingInto`    | （全面實作）         | `core::convert`     |
| `WrappingFrom<U>`   | （無）                 | `core::convert`     |
| `WrappingInto`      | （全面實作）         | `core::convert`     |
| `AsBytes`           | （無）                 | `core::convert`     |

基本型別之間的每個無損轉換實作都帶有 `#[intrinsic("nabi.cast.FROM.TO")]`。

```nabi
impl From<i32> for i64 {
    #[intrinsic("nabi.cast.i32.i64")]
    fn from(value: i32) -> i64 { ... }
}
```

`as T` 的數值強制轉型使用相同的內建函式；編譯器在無損分支與環繞分支之間選擇。

## 2.11.13. `AsBytes` trait：位元組作為檢視

`core::convert::AsBytes` 是標準函式庫每個位元組引數所接受的型別：`str::from_utf8`（[2.15.6](215-string.md#2156-utf-8-不變式行程內的每個字串)）、`hex` 與 `base64`（[2.22](222-encoding.md)）、`crypto`（[2.21](221-crypto.md)）、`CString::new`（[1.7.2](../100-bare/107-ffi.md)）、`fs` 與 `net` 的寫入函式（[2.23.3](223-io.md)）以及 `Slice<u8>::find`（[2.9.7](209-collections.md)）。這類參數寫成 `<B: AsBytes>`，每個位元組引數各有一個型別參數，因此金鑰與訊息不必是相同的型別。

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

程式自己的型別只要實作 `AsBytes`，就能用在任何接受位元組的地方：

```nabi
struct Frame { body: List<u8>; }
impl AsBytes for Frame {
    fn byte_slice(self) -> Slice<u8> { self.body[:] }
}

let tag = crypto::Hmac::mac(crypto::HashAlgorithm::Sha256, key, frame);
let text = encoding::hex::encode(frame);
```

`AsBytes` 只有這一個路徑；`core::io` 不重新匯出它。
