# 3.8. `nabion::`：NABION 数据格式

NABION（Nabi Object Notation）是用 Nabi 字面量语法书写值的数据格式。文件扩展名是 `.nabion`。它的作用与 Rust 的 RON 和 Zig 的 ZON 相同；包清单 `nabi.nabion` 和锁文件 `nabi.lock`（[refs-lang 9.10.3](../../refs-lang/009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）都使用这种格式。

## 3.8.1. 格式

NABION 是 Nabi 表达式语法的一个子集。库用自己针对该子集的词法分析器和递归下降解析器读取它；运行时不使用编译器的解析器。一个文件保存一个值。

| 形式        | 示例                                                  | 说明                                                  |
|-------------|-------------------------------------------------------|-------------------------------------------------------|
| none        | `none`                                                |                                                       |
| bool        | `true`、`false`                                       |                                                       |
| 整数        | `42`、`-7`、`0xff`、`0b1010`、`0o17`、`1_000_000`      | 无后缀                                                |
| 浮点数      | `3.14`、`-1.5e-3`、`2E10`                              | 无后缀                                                |
| char        | `'a'`、`'\n'`                                         | 不允许 `c'..'`                                        |
| 字符串      | `"text"`、原始字符串、多行字符串                      | 不允许 f-string                                       |
| 列表        | `[a, b, c]`                                           |                                                       |
| dict        | `{ "k": v, 1: w }`                                    | 键是值（字符串、数字等）                              |
| 元组        | `(a, b)`、`(a,)`                                      | `(x)` 是分组，不是元组                                |
| struct      | `Point { x: 1, y: 2 }`、`{ x: 1, y: 2 }`、`Pair(1, 2)`、`Unit {}` | 名称可以省略。字段名是标识符。元组结构体写作 `Name(a, b)` |
| enum 变体   | `Shape::Unit`、`Shape::Circle(1.0, at: p)`、`Shape::Rect { w: 2, h: 3 }` | 用完整路径书写                       |

- 允许尾随逗号。注释为 `//` 和 `/* */`。
- `{}` 既是空结构体也是空 dict（由目标类型决定）。一个 `{ .. }` 中可以全是字段（标识符键）或全是键（值键），但不能两者兼有；混用是错误。
- 不允许：裸标识符、运算符（只允许数字前的一元 `-`）、方法调用、`..`、块、f-string、Set 字面量（`{1, 2}`；Set 要写成列表）、`c'..'` 字面量、泛型路径（`List<i32>::..`），以及值之后剩余的词法单元。这些都是错误，错误会显示 `line:col`。
- NaN 和无穷大没有 NABION 写法。输入中的 `nan` / `inf` 会作为裸标识符被拒绝。
- NABION 浮点数是 `f64`。超出 `f64` 范围的浮点字面量（`1e999`）是错误，错误会写出范围：``float `1e999` is out of range (NABION floats are f64, up to about 1.8e308)``。`nabion::parse` 和工具链的读取器（`nabi.nabion`、`nabi.lock`、对 `.nabion` 文件运行 `nabi fmt`）遵循同一规则。

> [!NOTE]
> 已知限制：不支持第一个键为 `none` 的 dict（`{ none: 1 }`），因为读取器会把 `none:` 当作字段名，把花括号读成结构体。

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

## 3.8.2. 与 Nabi 类型的对应

| NABION         | Nabi 类型                                                                    |
|----------------|------------------------------------------------------------------------------|
| `none`         | `T \| none` 的 `none`                                                        |
| `true` / `false` | `bool`                                                                     |
| 整数           | 所有整数类型、`int`。在期望浮点数的地方（`f32` / `f64` / `float`）也接受     |
| 浮点数         | `f32`、`f64`、`float`                                                         |
| char           | `char`                                                                        |
| 字符串         | `str`                                                                         |
| 列表           | `List<T>`、`[T; N]`、`Set<T>`                                                 |
| dict           | `Dict<K, V>`                                                                  |
| 元组           | 1 ~ 6 个元素的元组                                                            |
| struct         | `struct`（`#[derive(Nabion)]`）                                               |
| enum 变体      | `enum`（`#[derive(Nabion)]`）                                                 |
| 任意值         | `Value`                                                                       |

## 3.8.3. 读取

`parse` 把文本读成 `Value`。

| 输入                         | `Value`                                                        |
|------------------------------|----------------------------------------------------------------|
| `{ ident: v, .. }`           | 无名 `Struct`                                                  |
| `{ "k": v, .. }`、`{}`       | `Dict`                                                         |
| `Name { f: v }`              | 具名 `Struct`                                                  |
| `A::B { f: v }`              | `Variant`（值在 `named` 中）                                   |
| `A::B(a, k: b)`、`A::B`      | `Variant`（`args`、`named`）                                   |
| `Name(a, b)`                 | 单段路径的 `Variant`（元组结构体）                             |
| 整数                         | `Int`（精确的 `int`）                                          |

在带类型的读取（`from_str`、`from_value`）中，由目标类型决定如何解释。

- 结构体目标接受 `Struct`、`{}` 和 `A::B { .. }`，所以带模块路径的结构体名（`geom::Point { .. }`）也被接受。
- 写出的名称必须与目标匹配。结构体比较路径的最后一段；enum 变体比较最后两段。
- 类型中没有的字段会被跳过，所以格式可以增加字段而不破坏旧的读取方。类型上带有 `#[nabion(deny_unknown_fields)]` 时，这是在该字段名位置报告的错误。这适用于结构体和具名变体。
- 缺失的字段如果有 `= default` 就取该默认值，类型为 `T | none` 时取 `none`，否则是错误（`FromNabion::from_missing`）。
- 超出目标整数类型范围的整数值是错误。
- 具名变体也可以写成 `E::V(a, k: b)` 来读取。位置参数按顺序填充字段，之后是具名参数。

## 3.8.4. 规范形式

`nabion::to_str` 和 `nabi`（编辑清单或写出 `nabi.lock` 时）输出的规范形式如下。规范形式同时也是可读形式，所以没有单独的美化函数。

- 缩进为 4 个空格。
- 值能在 100 列以内写下时写成一行；否则写成多行，每个元素后带逗号。
- 浮点数使用能读回同一值的最短写法（`1.0`、`1.0e300`）。没有 NABION 形式的浮点数会作为带路径的 `NabionError` 抛出：NaN / 无穷大（“NaN/infinity has no NABION form”），以及超出 `f64` 范围的 `float`（“float 1.2e600 is out of range (NABION floats are f64, up to about 1.8e308)”）。
- 只有具名值的变体只有一种规范形式 `E::V { .. }`（`Value::Variant` 不保存形式标志）。
- 字符串和 char 中的双向控制字符（U+202A-U+202E、U+2066-U+2069）以转义形式写出（`\u{202e}`）。
- `nabion::to_str_flat` 不论宽度如何，都把同样的文本写成一行。
- 把规范形式读回会得到同一个值（往返）。

`nabi fmt` 检查 `.nabion` 文件是否为 NABION，并且只重新缩进。它不会把文件改写成规范形式，所以注释和排版得以保留。

## 3.8.5. 库：`std::nabion`

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

- 有位置时，`NabionError` 的 `Display` 是 `line:col: msg`，否则是 `at <path>: msg`。带类型读取的错误带有 `.servers[1].port` 或 `.limits["cpu"]` 这样的路径，`from_str` 会把该路径转换为输入中的 `line:col`。
- `Value` 是 `#[non_exhaustive]` 的：标准库之外的 `match` 需要一个 `_` 分支。
- `Value::kind_name` 是错误消息中使用的种类名称，依次为 `"none"`、`"bool"`、`"integer"`、`"float"`、`"char"`、`"string"`、`"list"`、`"dict"`、`"tuple"`、`"struct"`，`Variant` 则为 ``"variant `<path>`"``。
- `Value` 上的 `==` 和 `hash` 是结构性的。dict 条目和结构体字段的比较与顺序无关，浮点值按同一性比较（NaN == NaN，-0.0 != 0.0）。无名空结构体与空 dict 相等。
- `Value` 上的 `Display`、`Debug`、`==` 和 `hash` 从不触发 panic。`Display` 不做检查地写出规范形式（NaN 显示为 `nan`）；`Debug` 把它写成一行。
- 标准实现：所有整数类型、`f32` / `f64`、`int` / `float`、`bool` / `char` / `str`、`Value`、`T | none`、`List`、`Set`、`Dict`、1 ~ 6 个元素的元组、`[T; N]`。

`@derive` 为 `#[derive(Nabion)]` 生成的代码调用 `Fields`、`struct_args`、`variant_of`、`variant_args`、`unknown_variant` 和 `from_value_at`。它们是 `pub(pkg)` 的：模板写出的名称在模板所在的模块中检查（[refs-lang 6.9.9](../../refs-lang/006-meta-system/609-template-tokens.md#展开中的名字)），所以 derive 在任何包中都能工作，而在 `std` 之外直接使用它们是错误。读取器和写入器是标准库私有的。

### `#[derive(Nabion)]`

为结构体和枚举（包括泛型的）同时实现 `ToNabion` 和 `FromNabion`。

| 定义                 | 书写形式               |
|----------------------|------------------------|
| 带具名字段的结构体   | `Name { f: v }`        |
| 元组结构体           | `Name(a, b)`           |
| 单元结构体           | `Name {}`              |
| 单元变体             | `E::V`                 |
| 元组变体             | `E::V(a)`              |
| 具名变体             | `E::V { f: v }`        |

字段的 `= default` 用于缺失的字段（3.8.3）。类型上的 `#[nabion(deny_unknown_fields)]` 使类型中没有的字段成为错误（3.8.3）；`#[nabion(..)]` 不接受其他键，拼错的键是错误，错误会列出有效的键。derive 用元记号 `@Self.name`、`@Self.is_unit` / `is_tuple` / `is_named`、`@field.has_default` / `@field.default`、`@Self.has_derive_key` 构建（[refs-lang 6.9](../../refs-lang/006-meta-system/609-template-tokens.md)）。

```nabi
#[derive(Nabion)]
struct Server { host: str; port: u16 = 80; tls: bool | none; }

#[derive(Nabion)]
struct Config { name: str; servers: List<Server>; limits: Dict<str, i32>; }

let cfg: Config = try nabion::from_str(text);   // errors are "line:col: msg"
print(try nabion::to_str(cfg));
```

### 限制

- `Dict` / `Set` 按插入顺序写出，与迭代顺序相同（2.9）。
- 无法读取 `[T; 0]`。
- 元组最多 6 个元素。
- 无法读取第一个键为 `none` 的 dict（3.8.1）。

## 3.8.6. 与其他格式的比较

| 格式   | struct 写法                 | 说明                              |
|--------|-----------------------------|-----------------------------------|
| RON    | `Name(field: v)`            | 与 Rust 不同的专用语法            |
| ZON    | `.{ .name = "x" }`          | Zig 的匿名结构体字面量            |
| NABION  | `Name { field: v }`         | 原样使用 Nabi 自己的字面量语法    |

NABION 不增加单独的语法，直接使用 Nabi 字面量，所以写在源代码中的值和写在文件中的值看起来一样。
