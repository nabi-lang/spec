# 2.7. niche 最佳化

niche 是在型別的有效位元模式中永遠不會出現的位元模式。把和型別的標籤存放在這樣的位元模式中，就不需要另外的標籤槽位，從而得到記憶體額外負擔為零的和型別。

這個最佳化之所以可行，是因為[和型別](206-sum-types.md)的記憶體表示交給了編譯器。

## 2.7.1. 動機

```nabi
let p: Ptr<u8> | none;     // 64-bit pointer + none
```

這個型別必須能以 8 位元組表示，而不是 16 位元組（指標 8 + 標籤 1 + 填充 7）。指標的 `null` 位元模式（0x0000_0000_0000_0000）在有效指標中永遠不會出現，因此可以用該位置表示 `none`。

這種節省是科學與資料程式碼記憶體效率的核心（受 Julia 的 isbits `union` 啟發）。

## 2.7.2. 自動 niche

`none` 的位元由實作定義。由一項加上 `none` 組成的和型別 `T | none`，若 `T` 有 niche，其大小就是 `T` 的大小，而 `none` 以該 niche 值表示。niche 值以全零位元儲存：當它不是 0 時，該項以其 niche 純量與 niche 值 XOR 後的結果儲存，而那個結果永遠不是零。因此對每一個 niche 和型別而言，以零填滿的儲存空間讀出來都是 `none`。

| 型別                                       | niche 值                              |
|--------------------------------------------|---------------------------------------|
| `bool`                                     | `2`                                   |
| `char`（編譯器基本型別）                   | 代理碼位 `0xD800`                     |
| 沒有欄位的 `enum`、酬載中沒有 GC 參照的 `enum` | 第一個未使用的標籤                 |
| `extern "C" fn(..) -> R`                   | null                                  |
| `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>`    | null（`#[niche(0)]`）                 |
| 單一欄位的 `struct` / 元組                 | 該欄位的 niche                        |
| 多欄位的 `struct` / 元組                   | 第一個有 niche 之欄位的 niche，略過 `bool` 與浮點數的 niche |

- 酬載中存放 GC 參照的 `enum` 沒有 niche。
- `bool` 或浮點數純量上的非零 niche（`bool`，或 `#[niche(NAN)]` 浮點數包裝）只會透過單一欄位的包裝傳遞。多欄位的聚合型別會略過這樣的欄位，因此 `(i64, bool) | none` 有標籤，而 `(i64, char) | none` 使用 `char` 的 niche。
- 當 niche 值為 0（null 指標、`extern "C" fn`、`NonZero`）時，該項的位元不變，因此這樣的 `T | none` 可以原樣作為可為 null 的值跨越 C ABI 傳遞（[5.9.4](../005-modules/509-extern-block.md#可為-null-的-c-函式指標extern-c-fn---r--none)）。
- 除 `none` 之外有兩項以上的和型別有標籤。

```nabi
Ptr<u8> | none           // 8 bytes, null is none
bool | none              // 1 byte
```

## 2.7.3. 沒有自動 niche

整數（`i8` .. `i64`、`u8` .. `u64`、`usize`、`isize`）與浮點數型別（`f32`、`f64`）沒有 niche，因為每一種位元模式都是有效的。需要 niche 時，請使用包裝（2.7.6）或 `#[niche]`（2.7.4）。

## 2.7.4. 選用的 niche：`#[niche(..)]`

```
niche_attr ::= "#" "[" "niche" "(" niche_arg ("," niche_arg)* ","? ")" "]"
niche_arg  ::= int_lit | "-" int_lit | const_path | range
range      ::= niche_val "..=" niche_val | ".." niche_val | niche_val ".."
const_path ::= "MIN" | "MAX" | "NAN" | "INFINITY" | "NEG_INFINITY" | type "::" const_name
```

```nabi
#[niche(0)]
struct UserId(u64);

#[niche(i32::MIN..=-1)]
struct NonNegative(i32);

#[niche(NAN, INFINITY, NEG_INFINITY)]
struct Real { value: f64; }
```

- 引數可以是整數、`MIN` / `MAX`（`T::MIN`）、`NAN` / `INFINITY` / `NEG_INFINITY`（`F::NAN`）、範圍 `a..=b` / `..b` / `a..`，以及其中多個的組合。它們依實例化後的欄位型別解讀。
- 列出的值是一項承諾：它們不是該型別的有效值。編譯器不檢查這一點；若出現這樣的值，和型別的判別就會出錯。

## 2.7.5. 可以附加的位置

`#[niche(..)]` 附加在單一欄位的 `struct` 上（元組 newtype 或一個具名欄位）。附加在 `enum`、`union` 或多欄位的 `struct` 上是錯誤。

## 2.7.6. 標準函式庫的包裝

| 名稱          | 排除的值                              | 定義                          |
|---------------|---------------------------------------|-------------------------------|
| `NonZero<T>`  | 0（僅限整數 T）                       | [stdlib 2.2.2](../../refs-standard-libraries/200-core/202-non-zero.md#222-nonzerot禁止零) |
| `NonNan<F>`   | NaN                                   | [stdlib 2.2.3](../../refs-standard-libraries/200-core/202-non-zero.md#223-nonnanf禁止-nan) |
| `Finite<F>`   | NaN、±∞                               | [stdlib 2.2.4](../../refs-standard-libraries/200-core/202-non-zero.md#224-finitef禁止-nan-與-) |
| `Positive<T>` | 負數（包含 0）。無號的 `T` 沒有 niche（`Positive<u32> \| none` 是 8 位元組） | [stdlib 2.2.5](../../refs-standard-libraries/200-core/202-non-zero.md#225-positivet禁止負值) |
