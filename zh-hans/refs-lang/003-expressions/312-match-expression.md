# 3.12. `match` 表达式

`match` 是一个表达式，它把一个值与多个模式比较，并对第一个匹配的分支求值。

## 3.12.1. 形式

```
match_expr  ::= "match" expr "{" match_arm ("," match_arm)* ","? "}"
match_arm   ::= pattern ("if" expr)? "=>" expr
```

```nabi
match expr {
    pattern1 => expr1,
    pattern2 => expr2,
    ...
    patternN => exprN,
}
```

`match` 表达式由 `match` 关键字、被匹配值和 `{ ... }` 主体组成。结构体字面量不能出现在被匹配值的顶层（[3.10.3](310-if-expression.md#条件位置中的结构体字面量)）。主体包含零个或多个以逗号分隔的分支；每个分支由一个模式、`=>` 词法单元（[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)）和一个表达式组成。允许尾随逗号。

## 3.12.2. 分支主体

```nabi
match v {
    1 => "one",
    2 => "two",
    _ => "other",
}
```

`=>` 之后跟单个表达式。如果该表达式是[块表达式](309-block-expressions.md)，它可以包含多条语句。

```nabi
match v {                        // v: i32 | none
    n: i32 => {
        let x = n * 2;
        x + 1
    },
    none => 0,
}
```

块分支最后一个表达式的值是该分支的结果。

## 3.12.3. 匹配顺序

分支自上而下检查，对第一个匹配的分支求值。该分支表达式的值是整个 `match` 表达式的结果。

## 3.12.4. 分支结果类型

按照[表达式概述](README.md)中的分支结果规则，如果所有分支结果表达式类型相同，该类型就是 `match` 表达式的结果类型；某个分支的和类型包含其他所有分支的类型时也是如此，与分支顺序无关。除此之外分支类型不同时，接收方必须有显式类型；和类型不会被自动推断。接收槽的类型遵循[变量声明](../005-modules/501-variable-declaration.md)的规则。

```nabi
// no type on the slot → inferred from the expression's result type
let x = match v {
    1 => "one",
    _ => "other",
};                      // expression result str, slot x: str

let y = match v {
    1 => 1i32,
    _ => "other",
};                      // ERROR (E0041): arm type mismatch; type annotation required

// sum type on the slot → member check
let w: i32 | str = match v {
    1 => 1i32,
    _ => "other",
};                      // OK

// single type on the slot → the expression's result must be that type
let z: i32 = match v {
    1 => 1i32,
    _ => "other",
};                      // ERROR (E0040): str arm does not fit i32
```

## 3.12.5. 穷尽检查

分支模式的集合必须覆盖被匹配值可能具有的每个值；否则是编译错误。

```nabi
let x: bool = ...;
match x {
    true => 1,
    // ERROR (E0098): false is missing
}
```

规则如下。覆盖封闭的和类型（`A | B`）的每个成员即为穷尽。对 `enum`，覆盖每个变体即为穷尽。`bool` 在 `true` 和 `false` 都被覆盖时是穷尽的。列表、切片或数组在切片模式覆盖每个长度时是穷尽的（3.11.1）。整数、`char`、字符串和浮点数的字面量模式与区间模式被视为开放集合。即使区间覆盖了该类型的所有值（`0..=255` 覆盖全部 `u8`），也不视为穷尽，因此需要通配符（`_`）或标识符模式（`x`）这样的兜底分支。非穷尽错误会列出缺失的值（`` `false` not covered ``），并提议为它们添加分支。

对 [`any` 槽](../002-types/202-any-slot.md)的动态槽进行匹配时，总是需要兜底分支，因为编译器无法知道会到来哪种类型。

## 3.12.6. 不可达分支

按匹配顺序永远无法到达的分支是编译错误（不是警告）。整数和 `char` 的字面量 / 区间模式按区间比较，因此已被前面分支的区间覆盖的区间同样不可达（`0..=9` 之后的 `3..=5`）。区间不能建立穷尽性（3.12.5）。

```nabi
match v {
    _ => 0,
    1 => 1,         // ERROR (E0099): the _ above always matches
}
```

## 3.12.7. 或模式

[模式](311-patterns.md)中的或模式（`|`）把共享同一主体的多个模式归为一组。

```nabi
match v {
    1 | 2 | 3 => "small",
    4 | 5 | 6 => "medium",
    _         => "large",
}
```

## 3.12.8. 守卫

分支上可以附加[模式](311-patterns.md)中的守卫（`if condition`）。

```nabi
match (x, y) {
    (a, b) if a == b => "equal",
    (a, b) if a > b  => "first bigger",
    _                => "second bigger",
}
```

带守卫的分支在穷尽检查中不算作兜底分支，因为守卫可能为 `false`。因此仅靠带守卫的分支不能保证穷尽。

```nabi
let x: bool = ...;
match x {
    a if a == true  => 1,
    a if a == false => 2,
    // ERROR (E0098): guarded arms alone do not guarantee exhaustiveness; _ required
}
```

## 3.12.9. 由匹配结果进行的类型收窄

在由类型模式或 `enum` 模式匹配的分支内，编译器会收窄被匹配值的类型。

```nabi
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),       // in this arm n: i32, static dispatch possible
    s: str => use_str(s),
}
```

这种收窄与 [`any` 槽](../002-types/202-any-slot.md)和[和类型](../002-types/206-sum-types.md)的类型收窄机制相同。

## 3.12.10. 求值顺序

`match` 按以下顺序求值：

1. 对被匹配值求值（一次）。
2. 自上而下检查分支。
3. 对第一个匹配分支的表达式求值。

被匹配值的副作用恰好发生一次。带守卫分支的守卫只在到达该分支时才求值。

## 3.12.11. match 表达式本身的结果

`match` 表达式本身的结果遵循上述分支结果规则（所有分支类型相同时为该类型，或者为包含所有分支类型的某个分支的和类型；否则接收方必须有显式类型）。

如果被匹配值是发散的（例如 `match return 0 { ... }`），`match` 也是发散的。

## 3.12.12. 发散分支

如果某个分支的表达式是[块表达式](309-block-expressions.md)的发散形式（`return`、`break`、`throw` 等），在分支结果类型检查中，该分支被视为与任何类型兼容。

```nabi
let x: i32 = match v {           // v: i32 | none
    n: i32  => n,
    none    => return 0,         // divergent, OK in an i32 position
};
```
