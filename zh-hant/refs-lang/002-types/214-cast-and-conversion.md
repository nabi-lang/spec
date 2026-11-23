# 2.14. 轉換

本節定義把值從一個型別移到另一個型別的語法。`as` 內建於語言中；`.into()` 與 `.try_into()` 則是對標準函式庫 trait 的一般方法呼叫，其 impl 構成標準函式庫的轉換矩陣。

```
cast_expr   ::= expr "as" type_expr
into_expr   ::= expr "." "into" ("::" "<" type_expr ">")? "(" ")"
try_into    ::= expr "." "try_into" ("::" "<" type_expr ">")? "(" ")"
```

## 2.14.1. 三種形式

三種形式及各自執行的內容：

| 形式               | 形狀                   | 執行                        |
|--------------------|------------------------|-----------------------------|
| `as` 強制轉型      | `expr as Type`         | 內建的強制轉型（沒有 trait）|
| `.into()` 呼叫     | `expr.into()`          | `Target::from`（`Into::into`） |
| `.try_into()` 呼叫 | `expr.try_into::<T>()` | `Target::try_from`（`TryInto::try_into`） |

沒有 `Cast` trait，使用者型別也不能新增 `as` 配對。`From` / `Into` 與 `TryFrom` / `TryInto` 的確切簽章，以及哪些型別配對實作了它們，定義於標準函式庫的轉換矩陣。

## 2.14.2. `as` 強制轉型

第一種形式是 `as` 強制轉型。

```nabi
expr as Type
```

`as` 是[關鍵字](../001-lexical/105-keywords.md)；左側是任何運算式，右側是[型別運算式](201-type-expressions.md)。編譯器直接降階這個強制轉型；不會呼叫任何 trait 方法。它用於可能不保留值的轉換，例如位元截斷、符號擴充與表示方式改變，其結果與 `.wrapping_into::<T>()` 相同（[stdlib 2.11.5](../../refs-standard-libraries/200-core/211-conversion.md)）。

```nabi
let x: i32 = 256i32;
let y: u8 = x as u8;          // 0
let z: f64 = x as f64;        // 256.0
```

允許的配對是一個固定的集合：

- 任何型別轉成它自己（無作用）。
- 整數型別、浮點數型別、`bool`、`char` 與 `c_char` 之中任兩者之間。有兩項限制。目標為 `bool`：只有 `bool` 能強制轉型為 `bool`。目標為 `char`：只允許 `u8 as char`（以及 `char as char`），因為較寬的值可能是代理碼位或超過 `U+10FFFF`。其他整數、浮點數、`bool` 與 `c_char` → `char` 的強制轉型是型別錯誤，訊息為 ``only `u8` can be cast as `char`, not `u32`; use `char::from_u32(n)` (`char | none`) for a wider value``。
- 沒有酬載的 `enum` 轉成整數型別，得到其標籤或值（[2.4.2](204-enum.md#強制轉型為整數)）。
- 轉成或轉出任意精度的 std 型別 `int` / `float`。轉成它們會執行無損的 `From` impl，因此沒有該 impl 的配對（`f64 as int`）是錯誤；從它們轉出會執行目標的 `WrappingFrom` impl。

從浮點數型別轉成整數型別時，`as` 向零截斷並飽和：超出目標範圍的值得到最接近的端點（`1e10 as i32` 是 `i32::MAX`，`-1.5 as u8` 是 `0`），NaN 得到 `0`，在每個最佳化等級都是如此。`as` 結果的完整表格見 [stdlib 2.11.6](../../refs-standard-libraries/200-core/211-conversion.md#2116-as-t位元重新解讀--數值強制轉型)。

整數、浮點數或 `char` 值 `as bool` 是編譯錯誤 ``cannot cast `i32` as `bool` ``：請改寫成比較，整數寫 `x != 0`，浮點數寫 `x != 0.0`。這個錯誤附帶該修改作為修正建議。`bool` 強制轉型為整數型別時是 `0` 與 `1`。

其他任何配對都是編譯錯誤：``cannot cast `str` as `i32` (only numeric / bool / char casts, and a payload-less enum to an integer)``。允許的 `as` 一律成功。對可能失敗的轉換，請改用 `.try_into()`。

## 2.14.3. `.into()` 呼叫

接下來是 `.into()` 呼叫。

```nabi
expr.into()
```

這會去語法糖為 `Into::into::<Target>(expr)`，即 `Target::from(expr)`，並且需要 `Target: From<typeof(expr)>` 的實作。目標型別從上下文推論，例如變數的宣告型別或參數型別。

```nabi
let x: i64 = 100i64;
let y: int = x.into();          // int::from(x)
```

無法推論目標型別時，可以用 turbofish 給出。

```nabi
let y = x.into::<int>();
```

`.into()` 只適用於標準函式庫註冊了保留值的轉換（即沒有資訊遺失的轉換）之處。它通常用於從較窄的型別轉換為較寬的型別。

## 2.14.4. `.try_into()` 呼叫

最後是 `.try_into()` 呼叫。

```nabi
expr.try_into::<Type>()
```

這會去語法糖為 `Target::try_from(expr)`（`Target: TryFrom<typeof(expr)>`）。`try_into` 是一個 [`throws`](../004-functions/414-throws.md) 方法：成功時回傳目標型別的值，失敗時擲回錯誤（目標的 `TryFrom::Error`；數值轉換為 `OverflowError`，[stdlib 2.11.4](../../refs-standard-libraries/200-core/211-conversion.md#2114-tryfromu-trait可能失敗的轉換)）。因此該呼叫必須以 `try` 標示。它用於轉換可能失敗的情況，例如把大的整數移到較小的整數，或把浮點數值轉成整數。

```nabi
let big: i64 = 10_000_000_000i64;
let n = try big.try_into::<i32>() catch e {
    ...
};
let m: i32 | none = try? big.try_into::<i32>();
```

## 2.14.5. 三種形式總結

三種形式的差異如下。

| 形式          | 可能失敗？           | 可能遺失資訊？          | trait        |
|---------------|----------------------|-------------------------|--------------|
| `as`          | 否                   | 是                      | 無（內建）   |
| `.into()`     | 否                   | 否（保留值）            | `From<S>`    |
| `.try_into()` | 是（擲回錯誤）       | 否（成功時）            | `TryFrom<S>` |

依情況選擇：要刻意改變位元或精度時使用 `as`；只需要安全、無損的放寬時使用 `.into()`；要明確處理失敗的可能性時使用 `.try_into()`。

## 2.14.6. 依上下文的自動轉換

無後綴的數值字面值不會被轉換；其型別由預期型別決定。`let x: f64 = 1` 中的 `1` 從一開始就是 `f64`。確切規則定義於[數值字面值](../001-lexical/106-numeric-literals.md#163-無後綴的字面值)。

相對地，執行時變數的自動轉換是不允許的。必須一律寫出上述三種形式之一。唯一的例外是檢視而非轉換：在預期 `SliceMut<T>` 之處的 `List<T>`，會以整個串列的可寫入檢視傳遞，而在那裡的 `xs[a:b]` 則以該範圍的檢視傳遞（[stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt可寫入檢視)）。不會複製任何東西。

```nabi
let a: i64 = 100i64;
let b: i32 = a;                 // ERROR: explicit conversion required
let b: i32 = a as i32;          // OK
let b: i32 = try! a.try_into::<i32>();     // OK: panics on failure
```

這種明確性防止了精度在不知不覺中遺失。

## 2.14.7. 轉換 trait 的位置

`From<U>` / `Into` 與 `TryFrom<U>` / `TryInto` 定義於 `core::convert`（[標準函式庫層級](../000-meta/005-standard-libraries-tiers.md)）。哪些 trait 為哪些型別配對實作的完整矩陣，定義於標準函式庫規格。`as` 沒有 trait；其配對是 [2.14.2](#2142-as-強制轉型) 中的固定集合。
