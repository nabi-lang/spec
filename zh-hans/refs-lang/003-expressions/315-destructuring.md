# 3.15. 解构赋值

在 `let` 的左侧写一个[模式](311-patterns.md)中的不可反驳的模式，就能把一个表达式的结果拆解到多个变量中。

## 3.15.1. 形式

解构赋值的语法是：

```
destructure_let ::= "let" irrefutable_pattern (":" type_expr)? "=" expr ";"
```

```nabi
let pattern = expr;
```

这种形式把表达式的结果与模式匹配，并一次性声明模式中的所有绑定变量。

## 3.15.2. 元组解构

最基本的形式是元组解构，用法如下。

```nabi
let (a, b) = (1, 2);                    // a = 1, b = 2
let (q, r) = divmod(10, 3);             // receive two values at once
let (q, _) = divmod(10, 3);             // discard the second
```

元素个数必须完全一致；否则是编译错误。

```nabi
let (a, b) = (1, 2, 3);                 // ERROR (E0101): length mismatch
```

## 3.15.3. 剩余模式

要一起接收其余元素，使用带 `..` 的剩余模式。

```nabi
let (first, ..rest) = (1, 2, 3, 4);     // rest = (2, 3, 4)
let (..init, last) = (1, 2, 3, 4);      // init = (1, 2, 3), last = 4
let (head, ..mid, tail) = (1, 2, 3, 4, 5);  // mid = (2, 3, 4)
let (..) = (1, 2, 3);                    // discard everything
```

`..` 在一个模式中最多出现一次，用于把其余部分打包到一个变量中，或者丢弃它们。

## 3.15.4. 嵌套解构

解构模式可以任意深度嵌套，因此复杂的结构可以一次拆解。

```nabi
let ((a, b), c) = ((1, 2), 3);          // a = 1, b = 2, c = 3
let (Point { x, y }, scale) = (origin, 2.0f64);
```

解构的形状只需与值的结构一致。

## 3.15.5. 结构体解构

对于结构体值，可以直接使用[模式](311-patterns.md)中的 `struct` 模式。

```nabi
let Point { x, y } = origin;            // x = origin.x, y = origin.y
let Point { x, .. } = origin;           // discard other fields
let Point { x: px, y: py } = origin;    // rename
```

这样可以一起取出字段，只接收部分字段而丢弃其余字段，或者给它们重新命名。

## 3.15.6. 类型标注

可以为解构的每个部分分别给出类型，也可以为整个模式一次性给出类型。

```nabi
let (a: i32, b: str) = (1i32, "hi");    // type on each position
let (a, b): (i32, str) = (1i32, "hi");  // type on the whole pattern
```

标注的类型是槽类型，不是运行时类型检查。它必须与值的类型一致；否则是编译错误。

```nabi
let (a: i32, b: str) = (1, 2);          // ERROR (E0040): second type mismatch
```

## 3.15.7. 只允许不可反驳的模式

`let pattern = expr;` 中的模式必须是不可反驳的，即必须总能匹配。

允许的模式：

- 标识符、通配符。
- 对元组、结构体和元组结构体的完整匹配（不缺少字段或位置）。
- 上述各项的嵌套。

不允许的模式：

- 字面量模式（`1`、`"hello"` 等）。
- 只匹配某个特定 `enum` 变体的模式（`Circle(r)`、`Click { x, y }` 等）。
- 类型检查模式（`v: i32`，针对和类型或 `any` 槽）。
- 区间模式。

这些形式可能匹配失败，因此用 `let` 接收它们是不安全的。应改用 [`let-else`](316-let-else.md)、[`match` 表达式](312-match-expression.md)或 `if let`（[`if` 表达式](310-if-expression.md)）。

## 3.15.8. 默认值（不采用）

不采用解构位置上的默认值。

```
let (a, b = 0i32) = ...;        // not adopted
```

函数参数的默认值被认为已经足够。要在解构位置处理可能缺失的值，使用 `match` 或 `let-else`。

## 3.15.9. 解构后变量的可见性

由解构声明的每个变量都被引入 `let` 所在的作用域。

```nabi
let (a, b) = (1, 2);
// a and b are both visible here
```

由同一个 `let` 声明的变量同时变为可见；没有哪个先于另一个可见。

## 3.15.10. 被匹配表达式的求值次数

解构的右侧表达式恰好求值一次，其结果与模式匹配。因此带副作用的表达式保证只被调用一次。

## 3.15.11. 多重赋值的推广

解构赋值可以看作多重赋值的推广。

```nabi
let (q, r) = divmod(num, den);          // two results
let (head, ..tail) = items;             // split
```

解构 `let` 声明新变量；它不写入已有变量。`let (a, b) = (b, a);` 在其所在块的剩余部分遮蔽 `a` 和 `b`，因此在循环主体内，外层的 `a` 和 `b` 保持原值。要对已有变量重新赋值，写不带 `let` 的元组（3.15.12）：`(a, b) = (b, a + b);`。

## 3.15.12. 不带 `let`：解构赋值

由已有位置组成的元组可以不带 `let` 地赋值：

```nabi
(a, b) = (b, a + b);             // reassigns a and b
```

`let` 声明，裸元组目标重新赋值，与 `let x = ..` 和 `x = ..` 的区分相同。规则见 [3.4.6](304-assignment.md#346-解构赋值)。

## 3.15.13. 参数中的模式

函数或闭包的参数可以是不可反驳的模式，而不是名字（[4.2](../004-functions/402-parameters.md)、[4.5](../004-functions/405-closure.md)）：

```nabi
fn index(self, (i, j): (usize, usize)) -> f64 { self.data[i * self.cols + j] }
pairs.map(|(k, v)| f"{k}={v}")
```

该模式在主体开始时按本节的规则解构实参。
