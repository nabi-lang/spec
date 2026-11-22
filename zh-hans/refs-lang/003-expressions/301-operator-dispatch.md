# 3.1. 运算符分派

本章定义一个总原则：[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)中定义的运算符词法单元如何转换为 trait 方法调用。哪个运算符映射到哪个 trait 的哪个方法，这张具体的对应表由标准库定义。

## 3.1.1. 每个运算符都是 trait 方法

正如[语言与标准库的边界](../000-meta/004-language-vs-standard-libraries.md)所确立的，语言提供运算符语法，而行为由标准库的 trait 决定。

```
a + b         →   <typeof(a) as Add<typeof(b)>>::add(a, b)
a - b         →   <typeof(a) as Sub<typeof(b)>>::sub(a, b)
a * b         →   <typeof(a) as Mul<typeof(b)>>::mul(a, b)
a < b         →   <typeof(a) as PartialOrd<typeof(b)>>::lt(a, b)
-a            →   <typeof(a) as Neg>::neg(a)
!a            →   <typeof(a) as Not>::not(a)
```

所有运算符都遵循同一模式：语言识别词法单元，并把它转换为 trait 方法调用。

## 3.1.2. 运算符 → trait 的映射位于标准库

每个运算符词法单元映射到哪个 trait 方法，这张确切的表由标准库定义。本文档只负责映射的存在和分派机制。

标准库通过元系统属性 `#[infix]` / `#[prefix]` / `#[postfix]` 把映射附加到 trait 方法上。确切机制定义于 [`#[infix]` / `#[prefix]` / `#[postfix]` 魔法](../006-meta-system/607-operator-magic.md)。

```
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
```

该示例展示 trait 和方法签名的形状；其含义来自第 2 章的 trait 规范与第 6 章的魔法规范的结合。第 3 章只定义语言把词法单元转换为上面那样的调用。

## 3.1.3. 不经虚表的分派

遵循 [trait 不能作类型](../002-types/212-trait-as-type-banned.md)的规则，运算符调用的分派设计为无需虚表间接即可解析。

- 对于静态槽，trait 方法在编译期确定，并编译为直接函数调用。
- 对于 `any` 槽，trait 方法由 [`any` 槽](../002-types/202-any-slot.md)的运行时标签检查确定，然后直接调用。开销是一次标签检查加一次直接调用。

两种情况下都不会因查找虚表而多出间接调用。让科学计算和数值代码的热循环免于这一开销，正是 [trait 不能作类型](../002-types/212-trait-as-type-banned.md)的动机，而这条分派规则是它的结果。

### 选择 impl

二元运算 `a op b` 从左操作数类型的该运算符 trait 的 impl 中进行选择。

- 如果一个类型有同一运算符 trait 的多个 impl（`Mul<i64>`、`Mul<f64>` 等），选择 `Rhs` 与右操作数类型匹配的 impl。泛型 impl 在其参数绑定到该类型时匹配（`impl<const N: usize> Call<Array<f64, N>>` 以 `N = 2` 接受 `Array<f64, 2>`）；类型与操作数完全相同的 impl 优先于泛型 impl。当多个泛型 impl 都接受该操作数时，选择其约束对所绑定内容成立的那一个：有 `impl<D: Distribution> Tilde<D> for S` 和 `impl<D: Distribution> Tilde<Iid<D>> for S` 时，不是 `Distribution` 的 `Iid<Normal>` 选择第二个。
- 如果左侧是基本类型、右侧是结构体，则使用为该基本类型声明的 impl（`2.0 * d` 是 `impl Mul<Duration> for f64`）。
- 省略了带默认值的 trait 实参的 impl（`impl<T> Add for W<T>`）按默认值解读，`Self` 即该 impl 的类型（`Add<W<T>>`）。若右操作数中无后缀的字面量仍未确定（`let v = W::new(1.0); m * v`），它取存在 impl 的那一个数字类型。
- 当该 trait 有针对左操作数的 impl，但没有一个接受右操作数时，错误会写出两个类型和缺少的 impl（`` operator `*` cannot be applied to `Vec3` and `f64`: there is no `impl ops::Mul<f64> for Vec3` ``），帮助信息会说明应当怎么写：两侧使用同一元素类型（`Vec3::<f64>::X`），或对数字做转换（`x as f32`）。

这条规则不限于运算符。凡是由填充 trait 类型参数的值的类型来选择 impl 的地方，它都适用：运算符的 `Rhs`，`T::from(x)` / `T::try_from(x)` / `Tr::f(x)` 的实参，处于已知类型位置的 `.into()` / `.try_into()` 的接收者，以及当各 impl 在该参数上不同时，方法调用 `x.m(v)` 或路径调用 `T::m(x, v)` 的实参。如果该值是无后缀字面量，选择按以下顺序进行：

1. 字面量默认类型（[1.6.3](../001-lexical/106-numeric-literals.md#163-unsuffixed-literals)：若它填充的参数有默认值则取该默认值，否则为 `i64` / `f64`）的 impl，前提是它存在且值能放进该类型。
2. 否则，取其参数类型能容纳该值的唯一 impl（对整数，是值在范围内的整数类型或 `int`；对实数，是能有限表示该值的 `f32` / `f64`，或 `float`）。`d * 2` 是 `Duration` 的 `Mul<i64>`。没有整数 impl 的整数，取能精确容纳它的浮点 impl，正如它填充浮点槽时那样（1.6.3）：有 `Call<f64>` 和 `Call<Vector<f64>>` 时，`s(1)` 即 `s(1.0)`。
3. 如果存在多个这样的 impl，则是要求加后缀的编译错误。

列表字面量 `[..]` 若有 `List` impl 则取之，否则取与字面量嵌套深度对应的 `Array` impl：有 `Call<f64>`、`Call<Vector<f64>>` 和 `Call<Matrix<f64>>` 时，`s([1.0, 2.0])` 是 `Vector` 那一个，`s([[1.0], [2.0]])` 是 `Matrix` 那一个，而泛型的 `Call<Array<f64, N>>` 两者都接受，`N` 取自深度。

这条规则只适用于裸字面量和带符号字面量（`-1`），不适用于 `1 + 1` 这样的表达式。以 `Tr::f(..)` 形式调用的静态 trait 方法，其 `Self` 由期望类型确定。当没有实参填充该 trait 的参数时，由调用的期望类型选择以它为返回类型的 impl（或约束）；否则该调用有歧义（[4.9.6](../004-functions/409-method.md#496-call-forms-of-free-functions-and-methods)）。

## 3.1.4. 没有映射的运算

如果给定的一对类型不存在 trait 实现，该运算符调用是编译错误。

```
let a: i32 = 1i32;
let b: str = "hi";
let c = a + b;          // ERROR: no Add impl for (i32, str)
```

诊断会写出该运算符和两个操作数类型。

```
error[E0054]: operator `+` cannot be applied to `i32` and `str`
```

## 3.1.5. 一元、二元、后缀

运算符按位置分为三类之一。

| 位置                  | 形式     | 映射机制           |
|-----------------------|----------|--------------------|
| 二元（中缀）          | `a OP b` | `#[infix("OP")]`   |
| 前缀一元              | `OP a`   | `#[prefix("OP")]`  |
| 后缀一元              | `a OP`   | `#[postfix("OP")]` |

同一个词法单元可以出现在多个位置。例如 `-` 既用作二元（`a - b`），也用作前缀一元（`-a`）。位置由词法单元出现的上下文确定：只有其后有操作数时为前缀；只有其前有操作数时为后缀；两侧都有操作数时为二元。

## 3.1.6. 优先级与结合性

除按位置分派外，运算符还有优先级和结合性。它们定义于[优先级与结合性](302-precedence-and-associativity.md)。

优先级和结合性不属于词法单元 → trait 的映射。3.2 中的表在语言层面固定它们；`#[infix("+")]` 这样的注册只是把词法单元连接到 trait 方法。

## 3.1.7. 用户定义的运算符

标准库通过 `#[infix]` / `#[prefix]` / `#[postfix]` 赋予基本运算符含义。用户代码可以用同一机制为其他 trait 附加新的映射。

```
#[infix("⊕", like = "+")]          // the new token follows the level of an existing token
fn combine(self, rhs: Rhs) -> Output;
```

同一（词法单元，位置，类型对）的映射必须唯一；冲突是编译错误。

要把不在[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)固定词法单元表中的字符序列注册为新运算符，必须通过同一机制注册该词法单元本身。确切步骤定义于 [`#[infix]` / `#[prefix]` / `#[postfix]` 魔法](../006-meta-system/607-operator-magic.md)。

## 3.1.8. Unicode 运算符

`÷ ≡ ≠ ≤ ≥` 是 `/ == != <= >=` 的另一种写法：词法分析器把它们读作那些词法单元（[1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-fixed-token-table)），因此它们分派到相同的 trait 方法。ASCII 范围之外的其他运算符（`×`、`⋅`、`∘`、`√`、`∈`、`∉`、`⊂`、`⊆`、`∪`、`∩`）是独立的词法单元，和任何运算符一样由 `#[infix]` / `#[prefix]` 绑定到 trait 方法；`×` 是 `ops::Cross`：

```
trait Cross<Rhs = Self> {
    type Output;
    #[infix("×")]
    fn cross(self, rhs: Rhs) -> Output;
}
```

一个方法也可以带多个 `#[infix]` 注册；此时每个词法单元都映射到同一方法，行为完全相同。
