# 3.8. `nabion::`：NABION 資料格式

NABION（Nabi Object Notation）是以 Nabi 字面值語法寫出值的資料格式。副檔名為 `.nabion`。它的角色與 Rust 的 RON、Zig 的 ZON 相同；套件清單 `nabi.nabion` 與鎖定檔 `nabi.lock`（[refs-lang 9.10.3](../../refs-lang/009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）都使用這個格式。

## 3.8.1. 格式

NABION 是 Nabi 運算式語法的子集。函式庫以自己的詞法分析器與遞迴下降剖析器讀取這個子集；執行時不使用編譯器的剖析器。一個檔案只保存一個值。

| 形式        | 範例                                               | 說明                                                 |
|-------------|-------------------------------------------------------|-------------------------------------------------------|
| none        | `none`                                                |                                                       |
| bool        | `true`、`false`                                       |                                                       |
| 整數     | `42`、`-7`、`0xff`、`0b1010`、`0o17`、`1_000_000`      | 不加後綴                                             |
| 浮點數       | `3.14`、`-1.5e-3`、`2E10`                              | 不加後綴                                             |
| char        | `'a'`、`'\n'`                                         | 不允許 `c'..'`                                   |
| 字串      | `"text"`、原始字串、多行字串             | 不允許 f-string                                 |
| 串列        | `[a, b, c]`                                           |                                                       |
| dict        | `{ "k": v, 1: w }`                                    | 鍵是值（字串、數字等）              |
| 元組       | `(a, b)`、`(a,)`                                      | `(x)` 是分組，不是元組                        |
| struct      | `Point { x: 1, y: 2 }`、`{ x: 1, y: 2 }`、`Pair(1, 2)`、`Unit {}` | 名稱可以省略。欄位名稱是識別字。元組結構寫成 `Name(a, b)` |
| enum 變體 | `Shape::Unit`、`Shape::Circle(1.0, at: p)`、`Shape::Rect { w: 2, h: 3 }` | 以完整路徑寫出            |

- 允許尾隨逗號。註解為 `//` 與 `/* */`。
- `{}` 既是空結構也是空字典（由目標型別決定）。一個 `{ .. }` 可以包含欄位（識別字鍵）或鍵（值鍵），但不能兩者都有；混用是錯誤。
- 不允許：裸識別字、運算子（只允許數字前的一元 `-`）、方法呼叫、`..`、區塊、f-string、Set 字面值（`{1, 2}`；Set 請寫成串列）、`c'..'` 字面值、泛型路徑（`List<i32>::..`），以及值之後剩下的語彙單元。這些都是會顯示 `line:col` 的錯誤。
- NaN 與無窮大沒有 NABION 表示法。輸入中的 `nan` / `inf` 會被當成裸識別字拒絕。
- NABION 的浮點數是 `f64`。超出 `f64` 範圍的浮點數字面值（`1e999`）是指出範圍的錯誤：``float `1e999` is out of range (NABION floats are f64, up to about 1.8e308)``。`nabion::parse` 與工具鏈的讀取器（`nabi.nabion`、`nabi.lock`、對 `.nabion` 檔案執行的 `nabi fmt`）遵循相同的規則。

> [!NOTE]
> 已知限制：不支援第一個鍵為 `none` 的字典（`{ none: 1 }`），因為讀取器會把 `none:` 當成欄位名稱，並把大括號讀成結構。

```nabion
// config.nabion
Config {
    name: "geom",
    retries: 3,
    ratio: 0.75,
    tags: ["a", "b"],
    limits: { "cpu": 2, "mem": 512 },
    mode: Mode::Fast,
    proxy: none,
}
```

## 3.8.2. 與 Nabi 型別的對應

| NABION          | Nabi 型別                                                                     |
|----------------|------------------------------------------------------------------------------|
| `none`         | `T \| none` 的 `none`                                                        |
| `true` / `false` | `bool`                                                                     |
| 整數        | 所有整數型別、`int`。在預期浮點數（`f32` / `f64` / `float`）的地方也接受 |
| 浮點數          | `f32`、`f64`、`float`                                                         |
| char           | `char`                                                                        |
| 字串         | `str`                                                                         |
| 串列           | `List<T>`、`[T; N]`、`Set<T>`                                                 |
| dict           | `Dict<K, V>`                                                                  |
| 元組          | 1 ~ 6 個元素的元組                                                      |
| struct         | `struct`（`#[derive(Nabion)]`）                                                 |
| enum 變體   | `enum`（`#[derive(Nabion)]`）                                                   |
| 任意值      | `Value`                                                                       |

## 3.8.3. 讀取

`parse` 把文字讀成 `Value`。

| 輸入                        | `Value`                                                        |
|------------------------------|----------------------------------------------------------------|
| `{ ident: v, .. }`           | 無名稱的 `Struct`                                               |
| `{ "k": v, .. }`、`{}`       | `Dict`                                                         |
| `Name { f: v }`              | 具名稱的 `Struct`                                                 |
| `A::B { f: v }`              | `Variant`（值在 `named` 中）                                  |
| `A::B(a, k: b)`、`A::B`      | `Variant`（`args`、`named`）                                    |
| `Name(a, b)`                 | 路徑只有一段的 `Variant`（元組結構）            |
| 整數                      | `Int`（精確的 `int`）                                            |

在具型別的讀取（`from_str`、`from_value`）中，由目標型別決定如何解讀。

- 結構目標接受 `Struct`、`{}` 與 `A::B { .. }`，所以帶模組路徑的結構名稱（`geom::Point { .. }`）也會被接受。
- 寫出的名稱必須與目標相符。結構比較路徑的最後一段；enum 變體比較最後兩段。
- 型別沒有的欄位會被略過，所以格式可以新增欄位而不破壞舊的讀取器。若型別上有 `#[nabion(deny_unknown_fields)]`，這會是位於該欄位名稱處的錯誤。這適用於結構與具名變體。
- 缺少的欄位若有 `= default` 就取該預設值，型別為 `T | none` 時取 `none`，否則是錯誤（`FromNabion::from_missing`）。
- 超出目標整數型別範圍的整數值是錯誤。
- 具名變體也可以讀成 `E::V(a, k: b)`。位置引數依序填入欄位，接著是具名引數。

## 3.8.4. 正規形式

`nabion::to_str` 與 `nabi`（編輯清單或寫出 `nabi.lock` 時）輸出的正規形式如下。正規形式也就是易讀的形式，所以沒有另外的美化輸出函式。

- 縮排為 4 個空格。
- 值若能放進 100 欄就寫成一行；否則寫成多行，每個元素後面加逗號。
- 浮點數使用能讀回相同值的最短表示法（`1.0`、`1.0e300`）。沒有 NABION 形式的浮點數會以帶路徑的 `NabionError` 擲回：NaN / 無窮大（"NaN/infinity has no NABION form"），以及超出 `f64` 範圍的 `float`（"float 1.2e600 is out of range (NABION floats are f64, up to about 1.8e308)"）。
- 只有具名值的變體只有一種正規形式，即 `E::V { .. }`（`Value::Variant` 不保存形式旗標）。
- 字串與字元中的雙向控制字元（U+202A-U+202E、U+2066-U+2069）會以跳脫形式寫出（`\u{202e}`）。
- `nabion::to_str_flat` 不論寬度，都把相同的文字寫成一行。
- 讀回正規形式會得到相同的值（往返）。

`nabi fmt` 檢查 `.nabion` 檔案是否為 NABION，並且只重新縮排。它不會把檔案改寫成正規形式，所以註解與版面會保留。

## 3.8.5. 函式庫：`std::nabion`

```nabi
use std::nabion;

#[non_exhaustive]
pub enum Value {
    None,
    Bool(bool),
    Int(int),
    Float(float),
    Char(char),
    Str(str),
    List(List<Value>),
    Dict(Dict<Value, Value>),
    Tuple(List<Value>),
    Struct { name: str | none; fields: Dict<str, Value>; },
    Variant { path: str; args: List<Value>; named: Dict<str, Value>; },
}
impl Value {
    fn to_str(self) throws (NabionError) -> str;   // canonical form
    fn kind_name(self) -> str;       // kind name for error messages (below)
}                                    // + Display, Debug, PartialEq, Eq, Hash

/// `v` as a `Value` through its type's description (`core::reflect`):
/// `Inspect` structs and enums, lists, dicts, sums and primitives.
pub fn to_value_any(v: any) throws (NabionError) -> Value;
/// A value of the type `info` describes, made from `v`; a struct field
/// `v` leaves out takes its default.
pub fn from_value_any(info: core::reflect::TypeInfo, v: Value) throws (NabionError) -> any;

pub struct NabionError { .. }         // private fields (0.5.10); + Display, Debug, Error, PartialEq, Eq, Clone
impl NabionError {
    pub fn new(message: str) -> NabionError;      // a decode error with no position
    pub fn within(self, seg: str) -> NabionError; // self one level down: seg (".name", "[3]") goes in front of the path
    pub fn message(self) -> str;     // without the position Display adds
    pub fn line(self) -> u32;        // 1-based; 0 when the value did not come from text
    pub fn column(self) -> u32;      // 1-based, in code points; 0 as for line
    pub fn path(self) -> str;        // ".servers[2].port"; empty at the root
}

pub trait ToNabion { fn to_nabion(self) -> Value; }
pub trait FromNabion {
    fn from_nabion(v: Value) throws (NabionError) -> Self;
    fn from_missing() throws (NabionError) -> Self;   // default: error. none for T | none
}

pub fn parse(s: str) throws (NabionError) -> Value;
pub fn to_str<T: ToNabion>(v: T) throws (NabionError) -> str;       // canonical form
pub fn to_str_flat<T: ToNabion>(v: T) throws (NabionError) -> str;  // canonical form on one line
pub fn from_str<T: FromNabion>(s: str) throws (NabionError) -> T;
pub fn to_value<T: ToNabion>(x: T) -> Value;
pub fn from_value<T: FromNabion>(v: Value) throws (NabionError) -> T;
```

- `NabionError` 的 `Display` 在有位置時為 `line:col: msg`，否則為 `at <path>: msg`。具型別讀取產生的錯誤帶有 `.servers[1].port` 或 `.limits["cpu"]` 之類的路徑，`from_str` 會把該路徑轉換成輸入中的 `line:col`。
- `Value` 是 `#[non_exhaustive]`：標準函式庫之外的 `match` 需要 `_` 分支。
- `Value::kind_name` 是錯誤訊息中使用的種類名稱：依序為 `"none"`、`"bool"`、`"integer"`、`"float"`、`"char"`、`"string"`、`"list"`、`"dict"`、`"tuple"`、`"struct"`，`Variant` 則為 ``"variant `<path>`"``。
- `Value` 上的 `==` 與 `hash` 是結構性的。字典項目與結構欄位的比較不論順序，浮點值依同一性比較（NaN == NaN、-0.0 != 0.0）。無名稱的空結構與空字典相等。
- `Value` 上的 `Display`、`Debug`、`==` 與 `hash` 絕不觸發 panic。`Display` 不加檢查地寫出正規形式（NaN 顯示為 `nan`）；`Debug` 把它寫成一行。
- 標準實作：所有整數型別、`f32` / `f64`、`int` / `float`、`bool` / `char` / `str`、`Value`、`T | none`、`List`、`Set`、`Dict`、1 ~ 6 個元素的元組、`[T; N]`。

`@derive` 為 `#[derive(Nabion)]` 產生的程式碼會呼叫 `Fields`、`struct_args`、`variant_of`、`variant_args`、`unknown_variant` 與 `from_value_at`。它們是 `pub(pkg)`：樣板寫出的名稱在樣板所在的模組中檢查（[refs-lang 6.9.9](../../refs-lang/006-meta-system/609-template-tokens.md#names-in-an-expansion)），所以衍生在任何套件中都能運作，而在 `std` 之外直接使用則是錯誤。讀取器與寫入器是標準函式庫私有的。

### `#[derive(Nabion)]`

為結構與 enum（包括泛型的）同時實作 `ToNabion` 與 `FromNabion`。

| 定義           | 寫出的形式           |
|----------------------|------------------------|
| 具名欄位的結構 | `Name { f: v }`    |
| 元組結構         | `Name(a, b)`           |
| 單元結構          | `Name {}`              |
| 單元變體         | `E::V`                 |
| 元組變體        | `E::V(a)`              |
| 具名變體        | `E::V { f: v }`        |

缺少的欄位使用該欄位的 `= default`（3.8.3）。型別上的 `#[nabion(deny_unknown_fields)]` 讓型別沒有的欄位成為錯誤（3.8.3）；`#[nabion(..)]` 不接受其他鍵，拼錯的鍵是列出有效鍵的錯誤。衍生是以元語彙單元 `@Self.name`、`@Self.is_unit` / `is_tuple` / `is_named`、`@field.has_default` / `@field.default`、`@Self.has_derive_key` 建構的（[refs-lang 6.9](../../refs-lang/006-meta-system/609-template-tokens.md)）。

```nabi
#[derive(Nabion)]
struct Server { host: str; port: u16 = 80; tls: bool | none; }

#[derive(Nabion)]
struct Config { name: str; servers: List<Server>; limits: Dict<str, i32>; }

let cfg: Config = try nabion::from_str(text);   // errors are "line:col: msg"
print(try nabion::to_str(cfg));
```

### 限制

- `Dict` / `Set` 依插入順序寫出，與迭代順序相同（2.9）。
- 無法讀取 `[T; 0]`。
- 元組最多 6 個元素。
- 無法讀取第一個鍵為 `none` 的字典（3.8.1）。

## 3.8.6. 與其他格式的比較

| 格式 | struct 表示法             | 說明                             |
|--------|-----------------------------|-----------------------------------|
| RON    | `Name(field: v)`            | 與 Rust 不同的專用語法 |
| ZON    | `.{ .name = "x" }`          | Zig 的匿名結構字面值    |
| NABION  | `Name { field: v }`         | 直接使用 Nabi 本身的字面值語法    |

NABION 沒有另加語法，而是直接使用 Nabi 字面值，所以寫在原始碼中的值與寫在檔案中的值看起來一樣。
