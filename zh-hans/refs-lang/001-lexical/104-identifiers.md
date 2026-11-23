# 1.4. 标识符

标识符是为变量、函数、类型、模块等命名的词法单元。

## 1.4.1. 格式

标识符采用 Unicode 标准附录 #31（UAX #31）的默认定义（`XID_Start` / `XID_Continue`），并增加以下内容：第一，下划线 `_`；第二，数学记法配置；第三，撇号；第四，emoji。

```
identifier          ::= identifier_start identifier_continue*
identifier_start    ::= xid_start | "_" | math_start | emoji | flag
identifier_continue ::= xid_continue | math_continue | prime | emoji | emoji_component
```

> [!IMPORTANT]
> **只要第一个字符不是数字，变量名就可以用人类语言和数学符号书写**。

### Unicode 版本

属性 `XID_Start`、`XID_Continue`、`ID_Compat_Math_Start`、`ID_Compat_Math_Continue` 和 `Extended_Pictographic` 遵循 **Unicode 18.0.0**（UCD 18.0.0）的表。

### 数学记法配置

这遵循 UAX #31 §7.1 “Mathematical Compatibility Notation Profile”。`math_start` 是 `ID_Compat_Math_Start` 加上 n 元运算符符号 `∑ ∏ ∫`，`math_continue` 是 `ID_Compat_Math_Continue` 加上同样三个符号。下表是资料性示例（截至 UCD 18.0）；以属性为准。

| 集合            | 码位                                                                                                        |
|-----------------|-------------------------------------------------------------------------------------------------------------|
| `math_start`    | `∂` U+2202、`∇` U+2207、`∞` U+221E，它们的数学样式变体 U+1D6C1、U+1D6DB、U+1D6FB、U+1D715、U+1D735、U+1D74F、U+1D76F、U+1D789、U+1D7A9、U+1D7C3（`𝛁 𝛛 𝛻 𝜕 𝜵 𝝏 𝝯 𝞉 𝞩 𝟃`），以及 `∑` U+2211、`∏` U+220F、`∫` U+222B |
| `math_continue` | `math_start` 的全部，上标 `¹ ² ³ ⁰ ⁴`-`⁹` `⁺ ⁻ ⁼ ⁽ ⁾`（U+00B2、U+00B3、U+00B9、U+2070、U+2074-207E），下标 `₀`-`₉` `₊ ₋ ₌ ₍ ₎`（U+2080-208E） |

`∑ ∏ ∫` 是 Nabi 为了能把它们用作函数名而加入的；因此它们不能作为运算符词法单元。

### 撇号

`′ ″ ‴ ⁗`（U+2032-2034、U+2057）只能出现在 `identifier_continue` 中（`f′`、`x″`）。

### Emoji

- `emoji`：具有 `Extended_Pictographic` 属性的码位，不包括 U+00FF 及以下的码位（`©`、`®` 等）。它们可以出现在开头位置和后续位置。
- `emoji_component`：只能出现在后续位置。
  - ZWJ（U+200D）：只能位于两个象形字符之间
  - 变体选择符 U+FE0E / U+FE0F：在规范化（1.4.2）中被去除
  - 肤色修饰符 U+1F3FB-1F3FF
  - 区域指示符 U+1F1E6-1F1FF（旗帜）
  - 标签 U+E0020-E007F

- `flag`：由两个区域指示符组成的旗帜（`🇰🇷`）也可以出现在标识符开头。

键帽序列（`1️⃣`）以数字开头，因此不是标识符。

### `x²` 是一个名称

`x²` 不是 `x` 的平方，而是单个名称 `x²`。平方写作 `x ** 2`、`x * x` 或 `x.pow(2)`。

### 合法示例

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

### 非法示例

```nabi
2x         // first character is a digit
my-var     // '-' is not an identifier character
my var     // whitespace
′x         // a prime cannot be the first character
1️⃣         // keycap: starts with a digit
```

## 1.4.2. 规范化

标识符在经过 NFC（Normalization Form C）规范化并去除变体选择符 U+FE0E / U+FE0F 之后再进行比较。因此预组合的 `é` 与组合形式 `e + ◌́` 是同一个标识符，`❤️` 与 `❤` 也是。编译器在切分出标识符之后立即应用这一规范化。

这一决定保证了可搜索性以及诊断消息的一致性。

### 不可见字符与孤立的组合符号

标识符不包含不可见字符（[1.1.2](101-source-files.md#112-字符编码)）：`co` + U+200B + `unt` 不是另一个名称，而是错误，诊断会指出 `count` 并建议改用它；只由韩文填充符（U+3164）构成的名称也是错误。ZWNJ 只有位于使用它的文字（波斯文 `می‌خواهم`）的字母之后才属于名称，ZWJ 只有位于两个象形字符之间才属于名称。组合符号可以延续名称（`e` + U+0301 即 `é`），但不能开始一个词法单元：前面没有字母的组合符号，无论位于词法单元开头还是数字之后，都是错误，诊断会指出该符号。前面没有可修饰字符的变体选择符同样是错误。

### 易混淆警告

编译器在以下情况下发出**警告**（不是错误）。

- 同一个包内（不论在哪个文件）不同的名称具有相同的骨架时。骨架是 NFKC 规范化加上易混淆字符折叠；实现可以只使用 UTS #39 易混淆字符表的一个子集。例如：`σ²` 与 `σ2`；拉丁字母 `a` 与西里尔字母 `а`。
- 一个标识符内混用西里尔字母与拉丁字母或希腊字母时。当每个西里尔字母都有对应的拉丁字母（`а` U+0430 对应 `a`）时，警告会指出对应的拉丁字母并给出拉丁拼写（`pay`）。拉丁字母与希腊字母混用（`Δt`、`σx`）在数学名称中很常见，不会被警告；韩文、汉字或假名与拉丁字母混用也不会被警告。

## 1.4.3. 大小写

标识符区分大小写。`count` 与 `Count` 是不同的词法单元。

推荐的命名约定 `snake_case`、`PascalCase` 和 `SCREAMING_SNAKE` 属于风格指南，规范不强制执行。

## 1.4.4. 与关键字冲突

与[关键字](105-keywords.md)中定义的保留字完全一致的词法单元被归类为关键字，而不是标识符。上下文关键字在其指定位置之外是标识符；在 `.` 之后、字段名声明中以及具名参数位置上，即使是保留字也是标识符（[1.5.5](105-keywords.md#155-名称位置上的关键字)）。

### 原始标识符

`r#` 后紧跟一个标识符构成原始标识符：它就是那个标识符，绝不是关键字，无论保留关键字还是上下文关键字。`r#match` 是名称 `match`，`r#type` 即使位于项的开头也是名称 `type`。

```
raw_identifier ::= "r#" identifier
```

```nabi
fn r#match(x: i32) -> i32 { x + 1 }   // a function named match
struct Tok { r#type: i32; }            // the same field as type: i32
let r#move = r#match(1);               // move, a reserved keyword (1.5.6)
let k = tok.type + tok.r#type;         // the same field twice
```

原始标识符与普通拼写指称同一事物：`t.r#type` 与 `t.type` 是同一个字段，声明为 `move: i32` 的字段既可读作 `s.move`，也可读作 `s.r#move`。`r#self`、`r#Self` 和 `r#_` 是编译错误：`self` 和 `Self` 不能作为名称（[1.5.5](105-keywords.md#155-名称位置上的关键字)），而 `_` 本来就是名称。`r#"` 和 `r##"` 仍然开始一个原始字符串（[1.7](107-string-literals.md)）。

原始标识符让代码可以保留一个本身是关键字的名称，例如为以后使用而保留的词之一（[1.5.6](105-keywords.md#156-为以后使用而保留的关键字)）：`r#move`。

## 1.4.5. 长度限制

本规范不对标识符长度设上限，遵循[源文件](101-source-files.md)中的一般规则。

## 1.4.6. 单个下划线

单个下划线（`_`）可以作为标识符，但在模式匹配和丢弃值时有特殊含义。该含义在模式规范中定义。在词法阶段，它被切分为一个单字符的标识符。
