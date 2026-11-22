# 3.11. 模式

模式按形状检查一个值，同时把它的各部分绑定到变量。模式出现在多个位置：[`match` 表达式](312-match-expression.md)的分支、[`if` 表达式](310-if-expression.md)的 `if let`、[解构赋值](315-destructuring.md)的 `let` 左侧等。

```
pattern         ::= or_pattern
or_pattern      ::= unary_pattern ("|" unary_pattern)*
unary_pattern   ::= literal_pattern
                  | identifier_pattern
                  | "_"                                  // wildcard
                  | type_check_pattern
                  | tuple_pattern
                  | slice_pattern
                  | struct_pattern
                  | enum_variant_pattern
                  | range_pattern
                  | binding_pattern
                  | "(" pattern ")"                      // grouping
binding_pattern ::= identifier "@" unary_pattern
type_check_pattern  ::= identifier ":" type_expr
tuple_pattern   ::= "(" (pattern ("," pattern)* ","?)? ")"
slice_pattern   ::= "[" (slice_item ("," slice_item)* ","?)? "]"
slice_item      ::= pattern | ".." identifier?                  // ".." at most once
struct_pattern  ::= path "{" (field_pattern ("," field_pattern)* ","?)? "}"
field_pattern   ::= identifier (":" pattern)?
enum_variant_pattern ::= path ("(" (pattern | "..") ("," (pattern | ".."))* ","? ")")?   // ".." at most once
range_pattern   ::= literal_pattern (".." | "..=") literal_pattern
```

## 3.11.1. 模式的种类

### 字面量模式

字面量模式把值与常量比较。

```nabi
0
3.14f64
"hello"
true
none
'A'
```

从[数值字面量](../001-lexical/106-numeric-literals.md)到[布尔和哨兵字面量](../001-lexical/109-other-literals.md)，每一种字面量形式都可以出现在模式位置；匹配使用相等比较（`==`）。因此浮点字面量模式 `0.0` 也匹配 `-0.0`，并且没有字面量能匹配 NaN：应在守卫中检测它（`x if x.is_nan() =>`）。当被匹配值是和类型时，字面量模式或区间模式匹配它所适合的成员类型（`1` 匹配 `i32 | str` 中的 `i32` 成员）。

### 标识符模式（绑定）

标识符模式在匹配时引入一个新变量。

```nabi
x
count
name
```

无论值是什么，它总是匹配，并把值绑定到该标识符。它不得与[关键字](../001-lexical/105-keywords.md)中的保留字冲突。如果一个裸名字解析为用 `use` 导入的 `enum` 变体，它就不是绑定，而是该变体的 `enum` 模式（[5.5.7](../005-modules/505-use.md#导入-enum-变体)）。一个裸名字如果是被匹配值的 `enum` 类型的变体，无论是否导入，也表示该变体（[2.4.8](../002-types/204-enum.md#248-由期望类型确定的裸变体)）；带载荷的变体用其载荷模式匹配（`V(..)`）。

解析为 `const` 或 `static` 的名字（不论大小写，`LIMIT`、`limit`），或者与被匹配值的 `enum` 类型的某个变体相差不超过两次编辑的首字母大写名字（`Gren` 对应 `Green`），是编译错误：作为绑定，它会匹配所有值。要与常量比较，写一个守卫：`x if x == LIMIT`；对于变体，写出它的路径。其他首字母大写的绑定（`Foo =>`）是普通绑定，并会触发 `non_snake_case` lint。

```nabi
const LIMIT: i32 = 5;
match n {
    LIMIT => ..,              // ERROR (E0103): `LIMIT` here would be a binding that matches every value
    x if x == LIMIT => ..,    // compares with the const
    _ => ..,
}
```

### 通配符模式：`_`

```nabi
_
```

通配符模式同样总是匹配，但它丢弃值而不绑定。用它来显式忽略某一部分。

带类型的通配符 `_: T` 与类型检查模式一样，只在值为 `T` 时匹配，且不绑定任何东西。

```nabi
match v {
    _: i32 => "int",          // matches if i32, no binding
    _ => "other",
}
```

### 类型检查模式：` name: Type`

```nabi
n: i32
v: User
```

类型检查模式只在左侧动态槽或和类型的值属于给定类型时匹配。匹配成功时，收窄为该类型的值被绑定到标识符。

这种模式以模式的形式表达了 [`any` 槽](../002-types/202-any-slot.md)和[和类型](../002-types/206-sum-types.md)的类型收窄机制。

`:` 之后的类型是完整的 `type_expr`，因此包括 `|`。`n: i32 | i64` 是单个类型检查模式，把值收窄为和类型 `i32 | i64`。由类型检查模式组成的或模式要用圆括号书写：`(n: i32) | (m: i64)`。对普通类型 `A` 的值，`v: A | B` 总是匹配；`v` 是作为 `A | B` 的该值。

```nabi
match v {
    n: i32 => use_int(n),       // matches if v is i32
    s: str => use_str(s),       // matches if v is str
}
```

### 元组模式

```nabi
(a, b)
(_, second)
(first, ..rest)
(..init, last)
(head, ..mid, tail)
(..)
```

`(p)` 是分组模式，与 `p` 相同，不是元组。单元素元组带尾随逗号书写：`(p,)`。元组类型（`(T,)`）和元组表达式（`(e,)`）遵循同样的规则。

元组模式用子模式逐一匹配元组的每个元素。`..` 是剩余模式，用于忽略元组的一部分，或把它绑定到单个变量。`..` 在一个模式中最多出现一次，且长度必须与值一致（有 `..` 时长度可以自由匹配）。

### 切片模式

```nabi
[]
[x]
[first, ..rest]
[.., last]
[first, .., last]
[0, n]
[[a, b], ..]
```

切片模式按元素匹配 `List<T>`、`Slice<T>`、`SliceMut<T>` 或定长数组 `[T; N]`，也通过成员匹配含有一个此类成员的和类型。没有 `..` 时，它匹配元素个数恰好等于其项数的值；有 `..`（最多一次）时，它匹配元素个数至少为其两侧项数之和的值，两侧之间的元素任意。`..name` 绑定这些元素：即值的切片 `xs[a:b]`（[3.6](306-indexing-and-slicing.md)），一个共享这些元素的 `Slice<T>`。每一项匹配其所在位置的元素，`..` 之前从前往后数，`..` 之后从后往前数，读取方式与 `xs[i]` 相同：得到元素的副本，正如元组模式绑定其部分的副本。

对于定长数组，长度是已知的：项数与之不符的模式是编译错误（E0101），而 `let [a, b, c] = arr;` 或 `let [first, ..rest] = arr;` 可以解构它。列表或切片的长度只在运行时才知道，因此 `let` 要用 `let .. else`（[3.16](316-let-else.md)）来拆分它。

穷尽性按长度计算：`[]`、`[x]` 和 `[x, y, ..]` 覆盖所有列表，遗漏某个长度的匹配会指出它（`` `[_, _, ..]` not covered ``）。

### `enum` 模式

```nabi
Circle(r)
Rectangle(w, h)
Pending
Click { x, y }
Click { x, .. }
Rectangle(..)
Rectangle(w, ..)
```

`enum` 模式以变体名开头。元组变体使用按位置的子模式（`Circle(r)`），具名变体使用字段名和子模式（`Click { x, y }`），单元变体只用名字（`Pending`）。元组变体模式可以使用一次剩余模式 `..`（`V(..)`、`V(a, ..)`、`V(.., z)`）。

> [!NOTE]
> 其他语言用 `Option<T>` 表达的东西，在 Nabi 中用[和类型](../002-types/206-sum-types.md)的 `T | none` 表达。因此不存在 `Some(v)` 或 `None` 这样的 `enum` 变体模式。对 `T | none` 的匹配使用类型检查模式（`v: T`）和字面量模式（`none`）。

### `struct` 模式

```nabi
Point { x, y }
Point { x, .. }
Point { x: px, y: py }
Point { x, y: 0.0 }      // y matches a literal, x is bound
```

`struct` 模式由 `struct` 名和 `{ ... }` 内的字段子模式组成。字段名与绑定变量名相同时可以简写：`Point { x }` 与 `Point { x: x }` 相同。`..` 忽略其余字段。

### 类型模式（简单）

```nabi
i32
f64
bool
```

如果被匹配值是和类型或 `any`，裸类型名（`f64 =>`、`i32 | str =>`）是类型模式，只检查值是否属于该类型，不绑定任何东西。对其他被匹配值，同一个名字是标识符模式（绑定）。

### 区间模式

```nabi
1..5
1..=5
'a'..='z'
```

[区间表达式](307-range-expressions.md)的词法单元原样用于模式位置，值落在区间内时模式匹配。可用于整数和 `char`（`'a'..='z'`）。

### 或模式

```nabi
1 | 2 | 3
"yes" | "y" | "Y"
Pending | Circle(0.0f64)
```

或模式用 `|` 组合多个子模式；任一子模式匹配，整体即匹配。各子模式绑定的变量必须名字和类型全部相同。

```nabi
match v {
    Ok(x) | OkAlt(x) => consume(x), // both variants bind the same name x with the same type
}
```

### 绑定模式：`name @ pat`

```nabi
n @ 1..=9
p @ Point { x: 0, .. }
```

如果子模式 `pat` 匹配，`name @ pat` 把整个值绑定到 `name`。`pat` 内部的绑定同样会创建。或模式要用圆括号括起来，如 `n @ (1 | 2)`。绑定模式不仅可以用在 `match` 中，也可以用在 `let` 和 `for` 的模式中。

```nabi
match age {
    n @ 0..=17 => f"minor {n}",
    n          => f"adult {n}",     // ranges are open sets, so a catch-all is needed
}
```

## 3.11.2. 守卫

```nabi
pattern if condition
```

守卫是在模式匹配之后检查的 `bool` 条件；条件为 `false` 时，该分支视为匹配失败。

```nabi
match (x, y) {
    (a, b) if a == b => "equal",
    (a, b) if a > b  => "first bigger",
    _                => "second bigger",
}
```

守卫是一个 `bool` 表达式，含义与 [`if` 表达式](310-if-expression.md)的条件相同。

## 3.11.3. 穷尽检查

穷尽检查验证一组模式是否覆盖了所有可能的输入；确切规则在 [`match` 表达式](312-match-expression.md)中定义。

简而言之，对封闭的和类型或 `enum` 的匹配必须覆盖每个变体，或者用通配符、标识符模式或类型模式覆盖其余情况。切片模式覆盖长度：覆盖直到最长项数的每个长度，`..` 模式则覆盖从其项数起的所有长度。

## 3.11.4. 或模式中变量绑定的一致性

```nabi
(v: i32) | (_: i32) => v    // ERROR (E0102): one side binds, the other ignores
(n: i32) | (n: i64) => ...  // ERROR (E0102): n has different types; the diagnostic suggests n: i32 | i64
n: i32 | i64 => ...         // OK: n: i32 | i64
A(x) | B(y) => ...          // ERROR (E0102): x / y are asymmetric
```

或模式的每个子模式都必须绑定名字相同、类型相同的变量。一个名字在所有备选中只有一个类型。

## 3.11.5. 模式位置一览

| 位置                        | 形式                                          |
|-----------------------------|-----------------------------------------------|
| `match` 分支                | `match v { pattern => expr, ... }`            |
| `if let`                    | `if let pattern = expr { ... } else { ... }`  |
| `let-else`                  | `let pattern = expr else { ... };`            |
| `let` 解构                  | `let pattern = expr;`                         |
| `for` 循环                  | `for pattern in iter { ... }`                 |

各位置允许哪些模式，在该位置的规范中定义。有些位置允许可反驳的模式；另一些只允许不可反驳的模式。

## 3.11.6. 可反驳与不可反驳

可反驳的模式是匹配可能失败的模式：字面量、`enum` 变体、类型检查、区间等。不可反驳的模式总是匹配：标识符、通配符，以及解构形状由类型保证的元组或 `struct`。

`let` 解构只允许不可反驳的模式，而 `match`、`if let` 和 `let-else` 两者都允许。
