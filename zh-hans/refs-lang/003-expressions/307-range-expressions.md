# 3.7. 区间表达式

区间表达式使用 `..` 或 `..=` 词法单元（[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)）产生一个带起点和终点的区间值。

```
range_expr      ::= expr ".." expr        // half-open
                  | expr "..=" expr       // inclusive
                  | ".." expr             // from start
                  | "..=" expr            // from start, inclusive
                  | expr ".."             // unbounded end
                  | ".."                  // full
```

## 3.7.1. 六种形式

区间表达式可以写成六种形式。

```nabi
0..10           // [0, 1, ..., 9]      exclusive end
0..=10          // [0, 1, ..., 10]     inclusive end
..10            // same as 0..10       start omitted (prefix)
..=10           // same as 0..=10      start omitted, inclusive end
10..            // [10, 11, ...]       end omitted, infinite
..              // full                both omitted, mainly for slicing
```

`..` 产生不含终点的半开区间；`..=` 包含终点。省略的起点视数据而定，表示 0 或某个默认值；省略的终点表示无穷或数据的末尾。

## 3.7.2. 脱糖

每种形式按如下方式脱糖为标准库类型。

```
a .. b          →   Range { start: a, end: b }
a ..= b         →   RangeInclusive { start: a, end: b }
.. b            →   RangeTo { end: b }
..= b           →   RangeToInclusive { end: b }
a ..            →   RangeFrom { start: a }
..              →   RangeFull
```

这些类型位于 `core::range`，由该模块末尾的 `#[literal]` 处理器构造。这一转换所用的确切标准库 trait 和类型定义于库规范。本文档只负责以下事实：语言把这六种形式识别为不同的表达式，并把它们脱糖为标准库类型的实例。

## 3.7.3. 优先级

在[优先级与结合性](302-precedence-and-associativity.md)的层级中，区间运算符（`..`、`..=`）是 prec=0 且不可结合：低于包括空值合并（`??`，prec=1）在内的所有二元运算符，只高于赋值。

```
0..n + 1            →   0 .. (n + 1)        // + binds tighter than ..
a ?? 0..3           →   (a ?? 0) .. 3       // ?? binds tighter than ..
a..b..c             // ERROR: range operators are non-associative
```

由于它们在二元运算符中结合得最弱，起点和终点中的算术表达式读起来很自然。

### 调用实参中的 `..x`

在调用实参位置，`..x` 不是区间而是展开（[4.2.6](../004-functions/402-parameters.md#426-spread-args)）。所以 `f(..5)` 是展开 `5` 的调用；要传递 `RangeTo`，把它包在括号中（`f((..5))`）。`..=x` 不是展开，所以即使在实参位置它也是区间（`f(..=5)` 传递一个 `RangeToInclusive`）。

## 3.7.4. 用途

### 迭代

区间值主要用于循环。

```nabi
for i in 0..10 { ... }
```

这里[循环表达式](313-loop-expressions.md)的 `for` 循环接受区间值并对其迭代。`Range` 和 `RangeInclusive` 在实现了 `Iterator` trait（[`trait`](../002-types/208-trait.md)）时自动可迭代。

### 与切片的区别

切片和区间表达式是不同的记法。

```
arr[1:3]            // a slice of [indexing and slicing](306-indexing-and-slicing.md), uses a colon (`:`)
0..3                // a range value, uses two dots (`..`)
```

两者是含义不同的不同词法单元。切片是索引表达式的内部形式；区间是一个值（Range 类型的实例）。

### 模式匹配

区间模式（[模式](311-patterns.md)）使用相同的词法单元。其含义在那里定义。

```
match x {
    1..=5 => ...,
    _ => ...,
}
```

## 3.7.5. 步长是一个方法

步长不属于记法，而是由标准库方法 `.step_by(n)` 提供。

```nabi
(0..100).step_by(2)         // [0, 2, 4, ..., 98]
```

这符合 Nabi 的总体方向：保持语言语法简单，额外功能以方法提供。切片的 `[::2]` 是单独的记法，之所以引入，是因为在那里步长很自然。

## 3.7.6. 反向

要反向迭代，使用 `.rev()` 方法。

```nabi
(0..10).rev()
```

区间记法本身没有负步长。

## 3.7.7. 空区间

如果起点大于终点，区间为空。

```nabi
5..3            // empty range, valid expression, iterates 0 times
5..=3           // empty range
```

这不是编译错误，而是一个迭代零次的有效值。

## 3.7.8. 使用无限区间

`for` 循环可以在 `0..` 这样的开放区间上运行，并在循环内用 `break` 结束。如果值超过类型的最大值，按算术溢出处理（启用检查时触发 panic）。

```nabi
for i in 0.. {
    if condition(i) { break; }
}
```

无限迭代本身等价于 [`loop { ... }`](313-loop-expressions.md)；唯一的区别在于便利性。
