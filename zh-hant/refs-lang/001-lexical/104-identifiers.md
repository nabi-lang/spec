# 1.4. 識別字

識別字是為變數、函式、型別、模組等命名的語彙單元。

## 1.4.1. 格式

識別字採用 Unicode Standard Annex #31（UAX #31）的預設定義（`XID_Start` / `XID_Continue`），再加上下列各項：第一，底線 `_`；第二，數學記號設定檔；第三，撇號；第四，emoji。

```
identifier          ::= identifier_start identifier_continue*
identifier_start    ::= xid_start | "_" | math_start | emoji | flag
identifier_continue ::= xid_continue | math_continue | prime | emoji | emoji_component
```

> [!IMPORTANT]
> **只要第一個字元不是數字，變數名稱就可以用人類語言與數學符號撰寫。**

### Unicode 版本

`XID_Start`、`XID_Continue`、`ID_Compat_Math_Start`、`ID_Compat_Math_Continue` 與 `Extended_Pictographic` 這些屬性遵循 **Unicode 18.0.0**（UCD 18.0.0）的表格。

### 數學記號設定檔

這遵循 UAX #31 §7.1「Mathematical Compatibility Notation Profile」。`math_start` 是 `ID_Compat_Math_Start` 加上 n 元運算子符號 `∑ ∏ ∫`，`math_continue` 是 `ID_Compat_Math_Continue` 加上同樣這三個符號。下表是參考性的範例（以 UCD 18.0 為準）；具權威性的是屬性本身。

| 集合            | 碼位                                                                                                        |
|-----------------|-------------------------------------------------------------------------------------------------------------|
| `math_start`    | `∂` U+2202、`∇` U+2207、`∞` U+221E，它們的數學樣式變體 U+1D6C1、U+1D6DB、U+1D6FB、U+1D715、U+1D735、U+1D74F、U+1D76F、U+1D789、U+1D7A9、U+1D7C3（`𝛁 𝛛 𝛻 𝜕 𝜵 𝝏 𝝯 𝞉 𝞩 𝟃`），以及 `∑` U+2211、`∏` U+220F、`∫` U+222B |
| `math_continue` | `math_start` 的全部、上標 `¹ ² ³ ⁰ ⁴`-`⁹` `⁺ ⁻ ⁼ ⁽ ⁾`（U+00B2、U+00B3、U+00B9、U+2070、U+2074-207E）、下標 `₀`-`₉` `₊ ₋ ₌ ₍ ₎`（U+2080-208E） |

`∑ ∏ ∫` 是 Nabi 加入的，以便能用作函式名稱；因此它們不能作為運算子語彙單元。

### 撇號

`′ ″ ‴ ⁗`（U+2032-2034、U+2057）只能出現在 `identifier_continue` 中（`f′`、`x″`）。

### Emoji

- `emoji`：具有 `Extended_Pictographic` 屬性的碼位，但不含 U+00FF 以下者（`©`、`®` 等）。它們可以出現在開頭與後續位置。
- `emoji_component`：只能出現在後續位置。
  - ZWJ（U+200D）：只能位於兩個圖形符號之間
  - 異體字選擇器 U+FE0E / U+FE0F：由正規化（1.4.2）移除
  - 膚色修飾符 U+1F3FB-1F3FF
  - 區域指示符 U+1F1E6-1F1FF（旗幟）
  - 標籤 U+E0020-E007F

- `flag`：由兩個區域指示符組成的旗幟（`🇰🇷`）也可以出現在識別字開頭。

鍵帽序列（`1️⃣`）以數字開頭，因此不是識別字。

### `x²` 是一個名稱

`x²` 不是 `x` 的平方，而是一個單一名稱 `x²`。平方寫成 `x ** 2`、`x * x` 或 `x.pow(2)`。

### 有效的範例

```nabi
count
my_var
α
β2
Δt
σ²
x₀
f′
∂u
∇φ
∑
변수1
データ
🚀count
_internal
```

### 無效的範例

```nabi
2x         // first character is a digit
my-var     // '-' is not an identifier character
my var     // whitespace
′x         // a prime cannot be the first character
1️⃣         // keycap: starts with a digit
```

## 1.4.2. 正規化

識別字在經過 NFC（Normalization Form C）正規化並移除異體字選擇器 U+FE0E / U+FE0F 之後才進行比較。因此預組合的 `é` 與組合形式 `e + ◌́` 是同一個識別字，`❤️` 與 `❤` 也是。編譯器在切分出識別字之後立即套用這個正規化。

這個決定確保了可搜尋性以及診斷訊息的一致性。

### 不可見字元與孤立的附加符號

識別字不包含不可見字元（[1.1.2](101-source-files.md#112-character-encoding)）：`co` + U+200B + `unt` 不是另一個名稱，而是一個錯誤，錯誤訊息會指出並建議 `count`；只由韓文填充字元（U+3164）組成的名稱也是錯誤。ZWNJ 只有在會用到它的文字的字母之後（波斯文 `می‌خواهم`）才是名稱的一部分，ZWJ 則只能位於兩個圖形符號之間。組合用附加符號可以延續名稱（`e` + U+0301 是 `é`），但不能開始一個語彙單元：前面沒有字母的附加符號，無論位於語彙單元開頭或數字之後，都是錯誤，錯誤訊息會指出該附加符號。前面沒有可修飾字元的異體字選擇器也是錯誤。

### 易混淆警告

在下列情況下，編譯器發出**警告**（而非錯誤）。

- 同一個套件內（不論檔案）不同的名稱具有相同的骨架時。骨架是 NFKC 正規化加上易混淆字元折疊；實作可以只使用 UTS #39 易混淆字元表的子集。例如：`σ²` 與 `σ2`；拉丁字母 `a` 與西里爾字母 `а`。
- 一個識別字內混用西里爾字母與拉丁或希臘字母時。若每個西里爾字母都有拉丁字母的對應（`а` U+0430 對應 `a`），警告會指出它並建議拉丁拼法（`pay`）。拉丁與希臘字母混用（`Δt`、`σx`）在數學名稱中很常見，不會發出警告；韓文、漢字或假名與拉丁字母混用也不會。

## 1.4.3. 大小寫

識別字區分大小寫。`count` 與 `Count` 是不同的語彙單元。

建議的命名慣例 `snake_case`、`PascalCase` 與 `SCREAMING_SNAKE` 屬於風格指南，規格不強制。

## 1.4.4. 與關鍵字的衝突

與[關鍵字](105-keywords.md)中定義的保留字完全相同的語彙單元，會被歸類為關鍵字而非識別字。上下文關鍵字在其指定位置之外是識別字；在 `.` 之後、欄位名稱宣告中以及具名引數位置，連保留字也是識別字（[1.5.5](105-keywords.md#155-keywords-in-name-positions)）。

### 原始識別字

`r#` 後面直接接一個識別字，就是原始識別字：它就是那個識別字，絕不是關鍵字，無論保留關鍵字或上下文關鍵字。`r#match` 是名稱 `match`，`r#type` 即使位於項目開頭也是名稱 `type`。

```
raw_identifier ::= "r#" identifier
```

```nabi
fn r#match(x: i32) -> i32 { x + 1 }   // a function named match
struct Tok { r#type: i32; }            // the same field as type: i32
let r#move = r#match(1);               // move, a reserved keyword (1.5.6)
let k = tok.type + tok.r#type;         // the same field twice
```

原始識別字與一般拼法指稱同一個東西：`t.r#type` 與 `t.type` 是同一個欄位，宣告為 `move: i32` 的欄位可以讀作 `s.move` 或 `s.r#move`。`r#self`、`r#Self` 與 `r#_` 是編譯錯誤：`self` 與 `Self` 不能作為名稱（[1.5.5](105-keywords.md#155-keywords-in-name-positions)），而 `_` 本來就是名稱。`r#"` 與 `r##"` 仍然開啟原始字串（[1.7](107-string-literals.md)）。

原始識別字讓程式碼能保留是關鍵字的名稱，例如保留給日後使用的字詞之一（[1.5.6](105-keywords.md#156-keywords-reserved-for-later-use)）：`r#move`。

## 1.4.5. 長度限制

本規格對識別字長度不設上限，遵循[原始檔](101-source-files.md)的一般規則。

## 1.4.6. 單一底線

單一底線（`_`）被接受為識別字，但在模式比對與捨棄值時有特殊意義。該意義定義於模式規格。在語彙階段，它被切分為一個單字元的識別字。
