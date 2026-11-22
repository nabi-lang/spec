# 6.7. 运算符魔法

`#[infix]`、`#[prefix]` 和 `#[postfix]` 这三个属性只负责把运算符词法单元连接到 trait 方法。优先级和结合性由 [3.2.3](../003-expressions/302-precedence-and-associativity.md#323-优先级与结合性表) 中的表格在语言层面固定，这些属性既不能设置也不能修改它们。它们是[运算符分派](../003-expressions/301-operator-dispatch.md)的脱糖机制得以运作所必需的核心魔法。

```
infix_attr      ::= "#" "[" "infix" "(" string_lit ("," like_arg)? ("," "swap")? ")" "]"
prefix_attr     ::= "#" "[" "prefix" "(" string_lit ("," like_arg)? ")" "]"
postfix_attr    ::= "#" "[" "postfix" "(" string_lit ("," like_arg)? ")" "]"
like_arg        ::= "like" "=" string_lit
```

## 6.7.1. 三个属性

先看这三个属性的用法示例。

```nabi
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}

trait Degrees {
    #[postfix("°", like = "?")]    // extension token, same level as postfix ?
    fn degrees(self) -> f64;
}
```

各属性附加的位置以及它所表达的运算形式：

| 属性                  | 位置          | 形式                     |
|-----------------------|---------------|--------------------------|
| `#[infix(sym)]`       | trait 方法    | 二元（`a SYM b`）        |
| `#[prefix(sym)]`      | trait 方法    | 前缀一元（`SYM a`）      |
| `#[postfix(sym)]`     | trait 方法    | 后缀一元（`a SYM`）      |

## 6.7.2. 参数

各属性接受的参数：

| 参数     | 含义                                                               | 是否必需                                     |
|----------|--------------------------------------------------------------------|----------------------------------------------|
| `sym`    | 运算符词法单元（字符串字面量）                                     | 必需                                         |
| `like`   | 新词法单元所跟随的已有词法单元（沿用其级别和结合性）               | 必需，仅用于不在 3.2 表中的词法单元          |
| `swap`   | 交换操作数后再连接（仅 infix）                                     | 可选                                         |

`sym` 和 `like` 与其他字符串字面量无异：词法单元就是字符串的值，因此 `"\\"` 和 `r"\"` 都表示单字符词法单元 `\`（`a \ b`）。

`swap` 把二元运算的两个操作数交换后再连接。带有 `#[infix("∈", swap)]` 的 `contains` 会把 `a ∈ b` 脱糖为 `b.contains(a)`。它用于容器不在左侧的运算符（`∈ ∉`）。

对于 3.2 表中的词法单元，不能写 `like`。不存在指定优先级或结合性的参数（`prec`、`left` / `right` / `none` 等），写出这类参数是编译错误。

```nabi
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]      // new token, same level and associativity as +
```

## 6.7.3. 连接机制

当 `#[infix("+")]` 附加在 trait 方法 `add` 上时，编译器的行为如下。

首先，`+` 词法单元作为连接点绑定到该 trait。然后，编译器把表达式 `a + b` 脱糖为 `<typeof(a) as the_trait<typeof(b)>>::add(a, b)`。优先级和结合性使用[优先级与结合性](../003-expressions/302-precedence-and-associativity.md)表中为 `+` 固定的值（9，左结合）。

三个属性都遵循同一模式，被绑定的只有词法单元、位置和 trait 方法。

操作数按该调用的实参进行类型检查。字面量形式的右操作数（`[1.0, 2.0]`、`{..}`、数字）采用为左操作数选定的 impl 的参数类型，因此当该 impl 的方法接受 `Vector<f64>` 时，`t ⊲ [1.0, 2.0]` 传入的就是 `Vector<f64>`。闭包操作数按其 `fn` 类型匹配 impl，与匹配约束的方式相同（[4.4.2](../004-functions/404-function-type.md#442-函数值与闭包值的类型)），覆盖实现 `impl<F: Tr> Op for F` 也适用于它。对于没有类型参数的 trait，方法自身的参数类型就是右操作数的类型。

接受 `ref self` 的方法，以方法调用接受接收者的方式接受其左操作数（或唯一操作数）：当操作数是变量或变量的字段时，方法所做的修改会写回该操作数，因此在 `fn draw(ref self, d: D)` 下，`g ∼ d` 会像 `g.draw(d)` 一样推进 `g`。闭包中按值捕获的变量会像方法调用时一样被拒绝（[4.6.7](../004-functions/406-capture-semantics.md#467-捕获结构体与引用句柄)）。

运算符不抛出错误（[4.14](../004-functions/414-throws.md)），因此方法必须声明为不带 `throws`；在会抛出错误的方法上使用运算符属性，会在声明处报错。可能失败的运算把错误作为值返回，即 `Result<T, E>` 或 `T | E`，由调用方处理（`try (a ⧵ b).get()`、`match`）；或者保持为不带该属性的具名方法（`try a.solve(b)`）。

```nabi
trait Solve<Rhs = Self> {
    type Output;
    #[infix("⧵", like = "/")]     // ERROR: `solve` throws; operators do not throw
    fn solve(self, rhs: Rhs) throws (str) -> Output;
}
```

## 6.7.4. Unicode 与多个别名

在同一个方法上附加多个 `#[infix]`，会把多个词法单元映射到该方法。

```nabi
trait Join<Rhs = Self> {
    type Output;
    #[infix("⊕", like = "+")]
    #[infix("⊞", like = "+")]
    fn join(self, rhs: Rhs) -> Output;
}
```

两个词法单元都调用 `join`。`÷ ≡ ≠ ≤ ≥` 不需要别名：词法分析器把它们读作 `/ == != <= >=`（[1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-固定词法单元表)），因此它们与 ASCII 词法单元到达同一个方法。实现 ASCII 词法单元对应的 trait（`<=` 对应 `PartialOrd`）即可让两种写法都生效。

注册这些写法之一是编译错误：词法分析器永远不会产生词法单元 `≤`，因此对它的绑定永远不会触发。该错误会指出对应的 ASCII 词法单元和需要实现的 trait。Unicode 运算符前面加点（`.≤`、`.×`）也不构成词法单元：词法分析器会把 `.` 和运算符分开读取，只有 1.10.3 中的 ASCII 运算符才有广播形式。

```nabi
trait Before {
    #[infix("≤")]     // ERROR: `≤` is the fixed operator `<=`; implement `ops::PartialOrd`
    fn before(self, rhs: Self) -> bool;
}
```

## 6.7.5. 冲突检查

如果同一个（词法单元，位置）组合上附加了不同的 trait，在注册时即报错（ERROR）。

```nabi
trait MyAdd { #[infix("+")] fn add(...) ... }
// another library / module.
trait Add { #[infix("+")] fn add(...) ... }
// → two traits for the binary '+' position. ERROR.
```

每个（词法单元，位置）组合只允许一次注册。

## 6.7.6. 同一词法单元的多个位置

像 `-` 这样既可用于前缀位置又可用于二元位置的词法单元，按各自独立的注册处理。

```nabi
trait Sub<Rhs = Self> {
    type Output;
    #[infix("-")]
    fn sub(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}
```

例如，在 `-a + b` 中，前缀 `-` 的结合比二元 `+` 更紧，因此解析为 `(-a) + b`。

二元 `-` 与前缀 `-` 是相互独立的映射，因此把它们附加到不同的 trait 上没有问题。

## 6.7.7. 带点前缀的（广播）映射

[广播运算符](../003-expressions/303-broadcast-operators.md)中带点前缀的词法单元也通过同一魔法注册。

```nabi
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+")]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*")]
    fn dot_mul(self, rhs: Rhs) -> Output;
}
```

只有词法单元不同，注册机制完全相同。

## 6.7.8. 映射的可见性

通过 `#[infix]` 等属性进行的注册，只在该 trait 可见的地方生效。可见是指在 [5.6](../005-modules/506-visibility.md) 的意义下可以访问：能通过路径到达的 `pub` trait 即可，不需要 `use`。在 trait 不可见的地方，运算符也不起作用。

这一规则契合库的隔离：在某个包中定义的用户自定义运算符 trait，只在导入该包的代码中生效。`pub` 运算符 trait 的词法单元也会到达同一批代码：在依赖于注册包的每个包的文件中，词法分析器都会切分出这些词法单元，因此来自依赖的 `<~` 或 `<-` 在那里的词法分析结果与在其自身包中相同（[1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-扩展词法单元)）。`#[literal]` 注册对 `pub` 处理函数遵循同一规则，作用于直接依赖其所在包的包（[6.5.2](605-literal-magic.md#652-形式)）；其他注册则留在其所在包内。

标准库的运算符 trait（`Add`、`Sub`、`Mul` 等）位于 `core::ops`。使用运算符不需要 trait 名在作用域内：`core` trait 的映射在每个源文件中都适用，因此算术和比较表达式无需单独导入即可使用。直接写出 trait 本身，例如 `impl ops::Add for V`，则要经由 prelude 模块 `ops` 或 `use`（[标准库 0.2.3](../../refs-standard-libraries/000-meta/002-import.md)）。

## 6.7.9. 用户定义的运算符

用户也可以在自己的 trait 上附加 `#[infix]` 等属性来定义运算符映射。

```nabi
trait CustomCombine<Rhs = Self> {
    type Output;
    #[infix("⊕")]
    fn combine(self, rhs: Rhs) -> Output;
}

impl CustomCombine for MyType { ... }

let r = a ⊕ b;            // user token
```

不在 3.2 表中的词法单元（如 `⊕`）要用 `like = "+"` 指定一个已有词法单元，并沿用该词法单元的级别和结合性。不能创建新的级别。该词法单元必须满足[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md#1104-扩展词法单元)中扩展词法单元的条件：每个字符都必须是 Unicode `Pattern_Syntax`，且与已有固定词法单元相同时报错。

词法单元可以以固定词法单元的开头开始：`#[infix("<-", like = "+")]` 会让 `c <- 3` 成为一个运算符，因为词法分析器采用最长匹配。这样的词法单元，即以某个固定词法单元的起始字符开头的词法单元，在注册它的包的文件中生效；若来自 `pub` trait，还在依赖该包的包中生效；在其他地方，`a<-1` 仍然是 `a < -1`（[1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-扩展词法单元)）。

二元 `@` 也可以借助 `like` 绑定：`#[infix("@", like = "*")]` 会像 NumPy 那样让 `a @ b` 表示矩阵乘积。`@` 仍是固定词法单元，而语法中没有其他二元 `@`：语言只在其前面没有操作数结束的位置使用它，即元语法之前（`@derive`、`@Self`）、模式中绑定名之后（`n @ (a, b)`），以及紧贴 `return`（`return@f`）。因此，在完整操作数之后的 `@`，在其 trait 可见的任何地方都是运算符，而其他用法在同一文件中保持原有含义。`@` 不能绑定为前缀或后缀运算符，因为这些位置是元语法。

二元 `~` 同样是空闲的：语言只把 `~` 读作前缀按位取反（`~x`），从不出现在操作数之后，因此 `#[infix("~", like = "==")]` 会让 `x ~ d`（“服从某分布”，或模型公式 `y ~ x`）成为运算符，而 `~x` 保持原有含义。其他固定词法单元只能在 3.2 表为其规定了形式的地方绑定：后缀 `%` 是错误，报错会指向数字后缀（`95pct`，[6.5.7](605-literal-magic.md#657-用户定义的字面量)），而 `->` 或 `|>` 是语法。后缀 `!`（阶乘 `n!`）也是错误：在名字之后，`!` 开始一个宏调用（`name!(..)`），在操作数之前则是 `not`；应写成函数或方法（`factorial(n)`、`n.factorial()`）。

词法单元中不含标识符字符：`′`（`f′` 是一个名字）和 `ℱ`（一个字母）不能作为运算符。应写成方法（`a.t()`）或函数（`fourier(x)`），或者选择 `†` 这样的符号（[1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-扩展词法单元)）。

## 6.7.10. 优先级与结合性级别

运算符级别唯一的权威出处是 [3.2.3 优先级表](../003-expressions/302-precedence-and-associativity.md#323-优先级与结合性表)。标准库和用户代码都不能设置或修改级别；新的词法单元通过 `like` 挂靠到某个已有级别上。

## 6.7.11. 何时求值

`#[infix]` 等注册在解析之后紧接着的元展开阶段处理。其后所有的表达式解析都会查阅注册表。

因此，在同一编译单元内，注册与使用的先后顺序无关紧要，因为扫描预处理会预先收集它们。3.2 表中词法单元的优先级从一开始就是固定的，与注册无关。

## 6.7.12. 不允许不同级别的别名

附加在同一 trait 方法上的所有别名都必须是同一级别的词法单元。

```nabi
trait Before {
    #[infix("<")]
    #[infix("⊲", like = "+")]    // ERROR: < is level 4, + is level 9
    fn lt(self, rhs: Self) -> bool;
}
```

这一限制源于“别名含义相同”这一前提。

## 6.7.13. 只能附加到 trait 方法

`#[infix]`、`#[prefix]` 和 `#[postfix]` 只能附加在 trait 的方法签名上。它们不能附加在自由函数、`impl` 方法或 trait 之外的普通方法上。

```nabi
fn my_add(a: i32, b: i32) -> i32 { a + b }
#[infix("+++")]                              // ERROR: cannot attach to a free function
fn my_concat(a: str, b: str) -> str { ... }
```

这样做是为了把运算符映射绑定到 trait 系统，保证分派的一致性。

## 6.7.14. 调用值：`ops::Call`

被调用者是值而非函数、且其类型实现了 `core::ops::Call`（[标准库 2.3](../../refs-standard-libraries/200-core/203-ops.md)）的调用 `v(a)`，就是 `v.call(a)`。实参作为一个值传入：单个实参就是它本身，多个实参组成元组（`p(x, y)` 是 `p.call((x, y))`），没有实参则是 `()`。因此，插值函数、多项式或分布的密度函数读起来就像它们本来所是的函数：`s(1.5)`、`xs |> s`，而 `s.(xs)` 会像对 `fn` 那样把它应用到每个元素上。

```nabi
struct Spline { a: f64; }

impl ops::Call<f64> for Spline {
    type Output = f64;
    fn call(self, x: f64) -> f64 { self.a * x }
}

let s = Spline { a: 2.0 };
s(1.5);                          // 3: s.call(1.5)
```

`Args` 不同的多个 impl 按实参选择，与其他重载的 trait 方法相同（[3.1.3](../003-expressions/301-operator-dispatch.md#313-不经虚表的分派)）：具体的 `Call<f64>` 与泛型的 `impl<const N: usize> Call<Array<f64, N>>` 并存时，`s(1.5)` 和 `s(xs)` 对任意维数的数组都可用，`s(1)` 采用 `f64` 的 impl，`s([1.0, 2.0])` 采用一维的 impl。保存可调用值的字段要写成 `(r.f)(x)` 来调用；`r.f(x)` 表示方法（[4.9.6](../004-functions/409-method.md#496-自由函数与方法的调用形式)）。约束 `F: Call<f64, Output = f64>` 通过它来调用 `f(x)`。实参只能按位置传递。`fn` 值和闭包被直接调用，不实现任何 trait。调用其他任何类型的值都是错误，其帮助信息会给出需要编写的 impl。

## 6.7.15. 小结

运算符魔法与 `#[lang]` 一起，是在[语言与标准库的边界](../000-meta/004-language-vs-standard-libraries.md)中实现语言词法单元与库 trait 绑定的核心机制。

标准库的运算符规范通过这一魔法决定每个词法单元的含义。
