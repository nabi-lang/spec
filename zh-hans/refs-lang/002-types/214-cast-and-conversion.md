# 2.14. 转换

本节定义把值从一个类型转到另一个类型的语法。`as` 内置于语言中；`.into()` 和 `.try_into()` 是标准库 trait 上的普通方法调用，其 impl 构成标准库的转换矩阵。

```
cast_expr   ::= expr "as" type_expr
into_expr   ::= expr "." "into" ("::" "<" type_expr ">")? "(" ")"
try_into    ::= expr "." "try_into" ("::" "<" type_expr ">")? "(" ")"
```

## 2.14.1. 三种形式

三种形式及各自执行的内容：

| 形式               | 写法                   | 执行                        |
|--------------------|------------------------|-----------------------------|
| `as` 强制转换      | `expr as Type`         | 内置的强制转换（无 trait）  |
| `.into()` 调用     | `expr.into()`          | `Target::from`（`Into::into`） |
| `.try_into()` 调用 | `expr.try_into::<T>()` | `Target::try_from`（`TryInto::try_into`） |

不存在 `Cast` trait，用户类型也不能添加 `as` 类型对。`From` / `Into` 和 `TryFrom` / `TryInto` 的确切签名，以及哪些类型对实现了它们，在标准库的转换矩阵中定义。

## 2.14.2. `as` 强制转换

第一种形式是 `as` 强制转换。

```nabi
expr as Type
```

`as` 是[关键字](../001-lexical/105-keywords.md)；左边是任意表达式，右边是[类型表达式](201-type-expressions.md)。编译器直接降级该强制转换；不调用任何 trait 方法。它用于不一定保持值的转换，例如位截断、符号扩展和表示形式的改变，其结果与 `.wrapping_into::<T>()` 相同（[stdlib 2.11.5](../../refs-standard-libraries/200-core/211-conversion.md)）。

```nabi
let x: i32 = 256i32;
let y: u8 = x as u8;          // 0
let z: f64 = x as f64;        // 256.0
```

允许的类型对是一个固定的集合：

- 任意类型到其自身（无操作）。
- 整数类型、浮点类型、`bool`、`char` 和 `c_char` 中任意两者之间。有两个限制。目标为 `bool`：只有 `bool` 能强制转换为 `bool`。目标为 `char`：只允许 `u8 as char`（以及 `char as char`），因为更宽的值可能是代理码位，或超出 `U+10FFFF`。其他整数、浮点、`bool` 和 `c_char` → `char` 的强制转换都是类型错误，消息为 ``only `u8` can be cast as `char`, not `u32`; use `char::from_u32(n)` (`char | none`) for a wider value``。
- 无载荷的 `enum` 到整数类型，得到其标签或值（[2.4.2](204-enum.md#casting-to-an-integer)）。
- 与任意精度的 std 类型 `int` / `float` 之间的互相转换。转换为它们时执行无损的 `From` impl，因此没有该 impl 的类型对（`f64 as int`）是错误；从它们转换出来时执行目标的 `WrappingFrom` impl。

从浮点类型到整数类型，`as` 向零截断并饱和：超出目标范围的值得到最接近的端点（`1e10 as i32` 是 `i32::MAX`，`-1.5 as u8` 是 `0`），NaN 得到 `0`，在每个优化级别下都是如此。`as` 结果的完整表格见 [stdlib 2.11.6](../../refs-standard-libraries/200-core/211-conversion.md#2116-as-t---bit-reinterpretation--numeric-cast)。

整数、浮点或 `char` 值 `as bool` 是编译错误，``cannot cast `i32` as `bool` ``：应改写为比较，整数写 `x != 0`，浮点值写 `x != 0.0`。该错误附带这一修改作为修复建议。`bool` 强制转换为整数类型时得到 `0` 和 `1`。

其他任何类型对都是编译错误：``cannot cast `str` as `i32` (only numeric / bool / char casts, and a payload-less enum to an integer)``。允许的 `as` 总是成功。对于可能失败的转换，改用 `.try_into()`。

## 2.14.3. `.into()` 调用

其次是 `.into()` 调用。

```nabi
expr.into()
```

它脱糖为 `Into::into::<Target>(expr)`，即 `Target::from(expr)`，要求存在 `Target: From<typeof(expr)>` 的实现。目标类型从上下文推断，例如变量的声明类型或参数类型。

```nabi
let x: i64 = 100i64;
let y: int = x.into();          // int::from(x)
```

在无法推断目标类型的地方，可以用 turbofish 给出。

```nabi
let y = x.into::<int>();
```

`.into()` 只适用于标准库登记了保值转换的地方，即没有信息损失的转换。它通常用于从较窄的类型转换到较宽的类型。

## 2.14.4. `.try_into()` 调用

最后是 `.try_into()` 调用。

```nabi
expr.try_into::<Type>()
```

它脱糖为 `Target::try_from(expr)`（`Target: TryFrom<typeof(expr)>`）。`try_into` 是 [`throws`](../004-functions/414-throws.md) 方法：成功时返回目标类型的值，失败时抛出错误（目标的 `TryFrom::Error`；对数值转换为 `OverflowError`，[stdlib 2.11.4](../../refs-standard-libraries/200-core/211-conversion.md#2114-tryfromu-trait---fallible-conversion)）。因此该调用必须用 `try` 标注。它用于转换可能失败的情况，例如把大整数转到较小的整数，或把浮点值转到整数。

```nabi
let big: i64 = 10_000_000_000i64;
let n = try big.try_into::<i32>() catch e {
    ...
};
let m: i32 | none = try? big.try_into::<i32>();
```

## 2.14.5. 三种形式汇总

三种形式的区别如下。

| 形式          | 可能失败？           | 可能丢失信息？          | trait        |
|---------------|----------------------|-------------------------|--------------|
| `as`          | 否                   | 是                      | 无（内置）   |
| `.into()`     | 否                   | 否（保值）              | `From<S>`    |
| `.try_into()` | 是（抛出错误）       | 否（成功时）            | `TryFrom<S>` |

按情况选择：有意改变位或精度时，使用 `as`；只需要安全、无损的扩展时，使用 `.into()`；要显式处理失败的可能时，使用 `.try_into()`。

## 2.14.6. 基于上下文的自动转换

无后缀的数值字面量不会被转换；其类型由期望类型确定。`let x: f64 = 1` 中的 `1` 从一开始就是 `f64`。确切规则在[数值字面量](../001-lexical/106-numeric-literals.md#163-unsuffixed-literals)中定义。

与此相对，不允许对运行时变量进行自动转换。必须始终写出上述三种形式之一。唯一的例外是视图而非转换：在期望 `SliceMut<T>` 的地方，`List<T>` 作为整个列表的可写视图传递，`xs[a:b]` 则作为该区间的视图传递（[stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt---writable-view)）。不复制任何东西。

```nabi
let a: i64 = 100i64;
let b: i32 = a;                 // ERROR: explicit conversion required
let b: i32 = a as i32;          // OK
let b: i32 = try! a.try_into::<i32>();     // OK: panics on failure
```

这种显式性防止了精度的悄然丢失。

## 2.14.7. 转换 trait 的位置

`From<U>` / `Into` 和 `TryFrom<U>` / `TryInto` 定义在 `core::convert` 中（[标准库层级](../000-meta/005-standard-libraries-tiers.md)）。哪些 trait 为哪些类型对实现的完整矩阵，在标准库规范中定义。`as` 没有 trait；它的类型对是 [2.14.2](#2142-as-casts) 中的固定集合。
