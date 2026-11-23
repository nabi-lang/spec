# 2.9. `impl`

`impl` 块是为类型附加方法、或为类型实现 trait 的容器。

## 2.9.1. 两种形式

`impl` 块的语法有两种形式：

```
impl_block      ::= "impl" generics? type_expr "{" impl_item* "}"
                  | "unsafe"? "impl" generics? type_expr "for" type_expr where_clause? "{" impl_item* "}"
                  | "impl" generics? "!" type_expr "for" type_expr where_clause? "{" "}"
impl_item       ::= fn_decl
                  | "type" identifier "=" type_expr ";"
                  | const_decl
```

在 `impl <` 中，只有当后面跟着泛型参数时（`<>`、`<T>`、`<T:`、`<T,`、`<T =`、`<const N`），`<` 才开启 `generics`；否则它是目标类型的开头，例如固有 `impl <A as P>::Out { .. }`。

### 固有 `impl`：附加方法

首先是固有 `impl`：

```nabi
impl Point {
    fn origin() -> Point { Point { x: 0.0f64, y: 0.0f64 } }
    fn distance(self, other: Point) -> f64 { ... }
}
```

这种形式是 `impl Type { ... }`，主体是一组函数定义。这样定义的函数成为该类型的方法。

定长数组和元组也可以有固有 `impl`。标准库借此为数组提供 `len()` / `is_empty()`。

```nabi
impl<T, const N: usize> [T; N] {
    fn len(self) -> usize { N }
}

impl<A, B> (A, B) {
    fn swap(self) -> (B, A) { (self.1, self.0) }
}
```

### trait `impl`：实现 trait

其次是 trait `impl`：

```nabi
impl Describe for Point {
    fn describe(self) -> str { f"({self.x}, {self.y})" }
}

impl ops::Add for Point {
    type Output = Point;
    fn add(self, rhs: Point) -> Point { ... }
}
```

这种形式是 `impl Trait for Type { ... }`。主体包含 [`trait`](208-trait.md) 要求的方法的主体，以及关联类型的具体定义。trait 提供了默认方法的项可以在主体中省略，此时使用默认主体。trait 声明的没有默认值的关联类型或常量必须定义；遗漏属于错误（E0072），其帮助信息会写出要添加的行，在 impl 的签名中提到了该类型时还会带上签名给出的类型（`type Item = i32;`）。

impl 方法必须满足以下条件：参数个数与 trait 声明相同；除接收者外的每个参数上的 `ref` 相同；`unsafe` 相同（[4.15.4](../004-functions/415-unsafe.md#4154-在-trait-中)）；在它写出类型的每个参数上，类型与声明的类型（经下述替换）相同（未写类型的参数取该类型，[4.1](../004-functions/401-function-declaration.md)）；当且仅当声明返回值时返回值，且返回类型是把声明类型中的 `Self`、trait 的参数及其关联类型替换为 impl 的对应类型后的类型（没有更窄的返回或协变返回）。否则在 impl 处报错，错误信息会显示该声明。impl 方法的 `throws` 必须与 trait 方法的 `throws` 相同；没有 `throws` 的 impl 方法取 trait 方法的 `throws`。impl 不能添加 `throws`，也不能扩大错误类型（[4.14.11](../004-functions/414-throws.md#41411-方法上的-throws)）。

`Send` / `Sync` 是自动的标记 trait，有两种手动声明（[标准库 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync线程边界标记)）。

- 选择加入：`unsafe impl Send for X {}`。`unsafe impl` 只能用于 `Send` / `Sync` 和 `unsafe trait`（[4.15.4](../004-functions/415-unsafe.md#4154-在-trait-中)），且手动实现这些 trait 时必须使用它。
- 选择退出：`impl !Send for X {}`、`impl<T> !Send for Cell<T> {}`。它阻止自动实现，且主体必须为空。`!` 只能与 `Send` / `Sync` 一起使用。

### `impl` 中的常量

`impl` 块中的 `const N: T = v;` 是该类型的关联常量。从外部读作 `Type::N`，在 `impl` 内部读作 `Self::N`。trait 声明的关联常量（[2.8.3a](208-trait.md#283a-关联常量)）在 trait `impl` 中获得其值。

```nabi
impl Circle {
    const UNIT: f64 = 1.0;
    fn unit() -> Circle { Circle { r: Self::UNIT } }
}
let r = Circle::UNIT;
```

`impl` 的目标或 trait 实参可以是投影（`<T as Tr>::X`），也可以指向之后才声明的项；它在所有项注册之后才解析。

`impl` 的目标可以是类型别名，无论是否泛型。别名会被展开，该 `impl` 等同于用别名所指类型写出的 `impl`：有 `type Al<T> = S<T>;` 时，`impl<T> Al<T> { .. }` 就是 `impl<T> S<T> { .. }`，`impl<T> Tr for Al<T>` 就是 `impl<T> Tr for S<T>`。固定了部分实参的别名（`type Names<V> = Dict<str, V>;`）给出的 `impl` 只覆盖那些实例。

## 2.9.2. 方法的 `self` 约定

主体中的函数要成为实例方法，其第一个参数必须是 `self`（或 `ref self`）。

```nabi
impl Point {
    fn origin() -> Point { ... }                    // static: called as Type::method()
    fn distance(self, other: Point) -> f64 { ... }  // instance: called as instance.method()
    fn replace_with(ref self, other: Point) { ... } // ref self: may reassign the caller's variable
}
```

`self` 参数的确切含义和 `ref self` 的行为在函数规范中定义。本文档只指出：有无 `self` 区分了静态方法和实例方法。

## 2.9.3. 一个类型的多个 `impl` 块

同一类型的固有 `impl` 可以拆分为多个块。

```nabi
impl Point {
    fn origin() -> Point { ... }
}

impl Point {
    fn distance(self, other: Point) -> f64 { ... }
}
```

这只是视觉上和组织上的分组；结果与合并成一个主体相同。

但是，同一个 trait 对同一类型只能 `impl` 一次；第二个 impl 无论是否为空都是错误（E0075），错误信息会显示第一个 impl。

```nabi
impl Display for Point { ... }
impl Display for Point { ... }   // ERROR: duplicate implementation (E0075)
```

## 2.9.4. 泛型 `impl`

`impl` 也可以声明为泛型。

```nabi
impl<T> List<T> {
    fn len(self) -> usize { ... }
}

impl<T> ops::Add for Pair<T> where T: ops::Add<T> {
    type Output = Pair<T::Output>;
    fn add(self, rhs: Pair<T>) -> Pair<T::Output> { ... }
}
```

类型参数在 `impl` 之后用 `<...>` 声明，可以出现在目标类型、trait 实参和关联类型中。可以用 `where` 子句添加附加约束，方式与 trait 声明相同。

泛型 `impl` 覆盖其参数能产生的目标实例：`impl<V> P<str, V>` 为 `P<str, i32>` 提供方法，但不为 `P<i32, i32>` 提供。

## 2.9.5. 孤儿规则

禁止为另一个包中定义的类型实现一个包中定义的 trait。

```nabi
// trait Display and type i32 are both defined in the standard library (another package)
impl Display for i32 { ... }     // ERROR: both from external packages (E0076)
```

因此，以下至少一条必须成立：

1. trait 在当前包中定义；
2. 目标类型在当前包中定义；
3. trait 是泛型的，并且在按顺序排列的目标类型和 trait 类型实参中（`impl Tr<T1, .., Tn> for T0`：`T0`、`T1`、..、`Tn`），当前包中定义的某个类型出现在 `impl` 的任何裸类型参数之前（Rust 的规则，RFC 2451）。

规则 3 让一个包可以把标量放在自身类型运算符的左边：

```nabi
// in a package defining `Matrix`
impl Mul<Matrix> for f64 { type Output = Matrix; ... }      // OK: `Matrix` is local (rule 3)
impl DotAdd<Matrix> for f64 { type Output = Matrix; ... }   // OK
impl<T> Mul<Matrix> for T { ... }        // ERROR: the bare `T` comes first (E0076)
impl Add<List<Matrix>> for i32 { ... }   // ERROR: `List<Matrix>` is `core`'s type (E0076)
```

本地类型之前出现裸参数会被拒绝，因为另一个包可能为处在该位置的自身类型写出同样的 `impl`；仅嵌套在外部类型中的本地类型（`List<Matrix>`）不算数。内置类型（基本类型、元组、数组、函数类型）视为在 `core` 中定义。

这条规则的单位是[源单元](../000-meta/003-source-units.md)中的包。因此，同一包内的不同模块可以自由地 `impl`。

这条规则防止两个包同时为同一个（trait，类型）对创建相互冲突的 `impl`，从而保证一致性。

### 一致性

一个（trait，类型）对只有一个 `impl`，并且没有特化。针对某一类型的快速路径以类型测试的形式写在泛型 `impl` 中：`if Self is f32 { .. }`（[2.10.13](210-generics.md#21013-对类型参数的类型测试)）。

- 泛型 `impl` 针对使用它的每个类型进行单态化。
- 具体 `impl` 只有在该具体类型满足泛型 `impl` 的约束时，才与泛型 `impl` 重叠。
- 如果两个泛型 `impl` 的目标（及 trait 实参）可以合一，它们就重叠；这在声明处作为错误报告。合一后仍为泛型的参数上的约束不能把它们区分开。即使目前还没有类型同时满足两个约束，也属于错误（与 Rust 的一致性规则相同）。
- 判断重叠时，`T?` 与 `T | none` 是同一类型。
- `const` 实参（[2.11](211-const-generics.md)）按值比较：`Buf<3>` 和 `Buf<{1 + 2}>` 是同一类型。由 `const` 参数乘以整数之和再加一个常量构成的实参（`Buf<{N + 1}>`、`Buf<{2 * N}>`），只有在两者对某些整数值可以相等时才与另一个实参重叠，其中无符号类型的参数不取负值。因此 `impl<const N: usize> X for Buf<{N + 1}>` 与 `impl X for Buf<0>` 不重叠，`Buf<{2 * N}>` 与 `Buf<{2 * M + 1}>` 也不重叠。其他任何形式（`{N * M}`）都假定为重叠。
- 否定推理只针对已知类型进行。
  1. 通过合一成为具体类型的参数，要检查其约束。`impl<U: Num> X for Pair<U, i32>` 和 `impl<U> X for Pair<str, U>` 以 `U = str` 合一，但 `str: Num` 不成立，因此它们不重叠。
  2. 已知类型构造器上的约束（`List<T>: Iterator`）在该 trait 的所有 impl 都不以该构造器或裸参数为目标时，能把两者区分开。因此 `impl<I: Iterator> IntoIterator for I` 和 `impl<T> IntoIterator for List<T>` 可以共存。trait 实参同理：只要没有以 `Iid` 为目标的 `Dist` impl，`impl<D: Dist> Tilde<D> for S` 和 `impl<D: Dist> Tilde<Iid<D>> for S` 就可以共存，调用或运算符取其实参满足约束的那一个（`s ∼ Iid { .. }` 取第二个）。

```nabi
impl<T: Display> Describe for List<T> { ... }
impl<T: Hash> Describe for List<T> { ... }   // ERROR: i32 satisfies both Display and Hash
impl<T: Shape> Area for T { ... }
impl<T: Color> Area for T { ... }            // ERROR: T and T unify (bounds irrelevant)
```

```nabi
impl<T: Display> Describe for T { ... }
impl Describe for i32 { ... }        // ERROR: overlaps the impl above since i32: Display
```

## 2.9.6. 方法的可见性

固有 `impl` 的方法默认是模块私有的。在同一模块内可以自由调用；要对外公开，需要显式写 `pub`。

```nabi
impl Account {
    pub fn balance(self) -> f64 { self.amount }      // public
        fn _internal_check(self) { ... }              // module-internal only
}
```

这与 [`struct`](203-struct.md) 和 [`enum`](204-enum.md) 的可见性模型一致。`impl` 块本身只是视觉上的分组，不是可见性的单位；可见性的单位始终是模块。

trait `impl` 的方法遵循 trait 的可见性，不能单独标注。

## 2.9.7. 固有 `impl` 与 trait `impl` 之间的方法冲突

如果同一个方法名同时存在于固有 `impl` 和 trait `impl` 中，调用处必须无歧义地选择。确切的解析规则在[方法](../004-functions/409-method.md#trait-之间的名称冲突)中定义。

## 2.9.8. 不得两次定义同一方法

为同一类型定义两个同名的方法是编译错误，无论是在一个 `impl` 块中，还是在多个固有 `impl` 块中（[2.9.3](#293-一个类型的多个-impl-块)），在一个 trait `impl` 中也是如此（没有重载）。允许来自不同 trait 的同名方法（[2.9.7](#297-固有-impl-与-trait-impl-之间的方法冲突)）。
