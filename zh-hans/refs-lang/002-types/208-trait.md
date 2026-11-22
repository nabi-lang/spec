# 2.8. `trait`

`trait` 是声明一组共同行为的容器。实现了这组行为的类型即满足该 trait。

本文档只定义 trait 的声明形式。实现 trait 的 `impl` 块在 [`impl`](209-impl.md) 中定义。

## 2.8.1. 基本形式

trait 声明的语法如下：

```
trait_decl  ::= visibility? "unsafe"? "trait" identifier generics? supertraits? where_clause? "{" trait_item* "}"
supertraits ::= ":" type_expr ("+" type_expr)*
trait_item  ::= method_sig
              | assoc_type_decl
              | method_default
              | assoc_const_decl
method_sig  ::= fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? ";"
method_default ::= fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
assoc_type_decl ::= "type" identifier (":" type_expr ("+" type_expr)*)? where_clause? ("=" type_expr)? ";"
assoc_const_decl ::= "const" identifier ":" type_expr ("=" const_expr)? ";"
```

trait 声明的示例：

```nabi
trait Describe {
    fn describe(self) -> str;
}

trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}

trait Iterator {
    type Item;
    fn next(ref self) -> Item | none;
}
```

trait 的主体用 `{ ... }` 括起来。主体中可以包含方法签名、关联类型和默认方法的主体。

## 2.8.2. 方法签名

方法签名写作：

```nabi
fn name(params) -> ReturnType;
```

没有主体的签名是 trait 要求的方法，实现必须提供其主体。带主体的则是默认方法，除非实现覆盖它，否则使用这个主体。`self` 和 `ref self` 参数的含义在函数规范中定义。

下面的示例同时包含必需方法和默认方法：

```nabi
trait Greeter {
    fn greet(self) -> str;                          // required
    fn greet_loudly(self) -> str { self.greet() }    // default: may be overridden
}
```

## 2.8.3. 关联类型

关联类型让 trait 在每个实现中可以有不同的类型。

```nabi
trait Iterator {
    type Item;
    fn next(ref self) -> Item | none;
}
```

关联类型声明为 `type Name;`，由实现填入具体类型。在一个实现中，关联类型恰好固定为一个具体类型；一个实现不能同时有多种可能。

关联类型可以在同一 trait 的签名中使用，如上面的 `Item | none`。它也可以在 throws 子句中使用，如 `throws (Self::X)`，trait 和 impl 的签名中都可以。

### 默认值

关联类型可以用 `type X: Bounds = T;` 给出默认值。impl 可以省略该关联类型以使用默认值，也可以用 `type X = U;` 覆盖它；没有默认值的关联类型必须给出（E0072）。默认值在 trait 的定义处解析，并代入 trait 的参数。

```nabi
trait TryFrom<U> {
    type Error = ConversionError;
    fn try_from(value: U) throws (Self::Error) -> Self;
}
impl TryFrom<RawPath> for Path { type Error = Utf8Error; ... }   // overridden
impl TryFrom<Celsius> for Kelvin { ... }                           // Error = ConversionError
```

在 trait 和所有 impl（trait impl 和固有 impl）内部，裸名 `Item` 和 `Self::Item` 都表示该关联类型，在方法签名和方法体中都可以使用。在 impl 中，它是该 impl 选定的具体类型。如果关联类型遮蔽了外部同名的类型（模块中的 `struct Item`、原始类型等），裸名在那里就有歧义：在 trait 或 impl 中使用它是编译错误，其帮助信息会要求写 `Self::Item` 或外部类型的路径（`pkg::Item`、`std::process::Output`）。声明这样的关联类型但不使用裸名时会给出警告（`assoc_type_shadowing`，[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）。同名的 trait 不是类型，因此不会发出警告（`trait IntoFuture { type IntoFuture; }` 没有问题）。

如果一个类型实现的两个 trait 都有同名的关联类型，那么在该类型的 impl 主体中，裸 `X` / `Self::X` 有歧义，属于错误。这时用 `<Self as Tr>::X` 指明 trait。

```nabi
impl Iterator for Stream { type Item = i32; ... }
impl Source for Stream { type Item = str; ... }

impl Stream {
    fn first(self) -> Self::Item { ... }                   // ERROR: Iterator::Item? Source::Item?
    fn first(self) -> <Self as Iterator>::Item { ... }     // OK
}
```

### 从外部引用关联类型：`T::AssocType`

要从外部引用受 trait 约束的参数的关联类型，使用 `T::AssocType` 写法。

```nabi
fn first_item<T: Iterator>(ref it: T) -> T::Item | none { it.next() }
```

如果约束是泛型 trait `T: Tr<A>`，`T::X` 就是 `Tr<A>` 实现中的 `X`，因为一个类型可以用不同的 `X` 实现 `Tr<A>` 和 `Tr<B>`（`<T as Tr<A>>::X` 与之相同，[2.13.4](213-type-paths.md#2134-完全限定路径)）。在约束 `T: Tr<A> + Tr<B>` 下，或者两个约束 trait 都声明了 `X` 时，裸 `T::X` 有歧义，属于错误；应写 `<T as Tr<A>>::X`。关联常量同理（2.8.3a）。

这种写法是使用[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)中 `::` 路径分隔符的关联项访问。路径的一般规则（模块路径、turbofish、完全限定形式）在[类型路径](213-type-paths.md)中定义。

## 2.8.3a. 关联常量

trait 可以用 `const N: T;` 声明关联常量，并用 `= value` 给出默认值。每个 `impl` 设定其值（有默认值时可以省略），在受该 trait 约束的类型参数上读作 `T::N`。

```nabi
trait Shape {
    const SIDES: u32;
    const NAME: str = "shape";
}

impl Shape for Square { const SIDES: u32 = 4; }

fn sides<T: Shape>() -> u32 { T::SIDES }      // same as <T as Shape>::SIDES
```

在泛型类型的 `impl` 中，关联常量在每个实例化中可以有不同的值（`impl<const N: usize> Shape for Poly<N> { const SIDES: u32 = N as u32; }`）。有歧义时，使用完全限定形式 `<T as Tr>::N`（[2.13.4](213-type-paths.md#2134-完全限定路径)）。

## 2.8.4. 父 trait

trait 可以要求只为满足另一个 trait 的类型实现它。

```nabi
trait Eq {
    fn eq(self, other: Self) -> bool;
}

trait Ord: Eq {
    fn cmp(self, other: Self) -> Ordering;
}
```

父 trait 写在 `:` 之后。因此实现 `Ord` 也要求实现 `Eq`。多个父 trait 用 `+` 连接，如 `trait Foo: Bar + Baz`。泛型 impl 必须对其约束允许的每一个实例化都为目标提供父 trait；这在 impl 处检查。

## 2.8.5. 泛型 trait

trait 也可以接受类型参数。

```nabi
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}

trait Convert<Target> {
    fn convert(self) -> Target;
}
```

类型参数在 trait 名之后的 `<...>` 中声明，参数可以有默认值，如 `Rhs = Self`。`Self` 是表示实现该 trait 的类型的关键字（[关键字](../001-lexical/105-keywords.md)）。

泛型 trait 的确切单态化规则在[泛型](210-generics.md)中定义。

## 2.8.6. where 子句：参数的附加约束

可以在 trait 声明或方法签名上写附加的 trait 约束。

```nabi
trait Sortable<T> where T: Ord {
    fn sort(ref self);
}

trait Container {
    type Item where Item: Clone;
    fn first(self) -> Item;
}
```

约束跟在 `where` 关键字之后，形式为 `Type: Trait`，以逗号分隔。附在 trait 本身上时，约束整个 trait；附在方法上时，只约束该方法。

## 2.8.7. trait 方法的分派

调用 trait 方法时，编译器查看调用处的接收者类型，并静态调用为该类型实现的方法。

```nabi
let p = Point { x: 1.0, y: 2.0 };
let s = p.describe();       // Point's Describe implementation is called statically
```

对静态类型的接收者，这种分派总是静态的，因为[类型系统概述](README.md)禁止 trait 作类型（把 trait 名写作变量、参数或返回类型）。只有对 `any Trait` 值的调用才在运行时分派（[2.2.10](202-any-slot.md#2210-受约束的-anyany-trait)）。

因此，需要多态时，使用泛型（[泛型](210-generics.md)）、和类型（[和类型](206-sum-types.md)）或 `any Trait`。

## 2.8.8. 可见性

trait 的项没有自己的可见性：其方法、关联类型和常量遵循 trait 的可见性，就像 `enum` 的变体遵循 `enum` 一样。在 trait 项上写 `pub` 是错误（E0016），在 trait `impl` 的方法上写 `pub` 也是错误（[2.9.6](209-impl.md#296-方法的可见性)）。`fn_modifiers` 与函数声明的相同（[4.1](../004-functions/401-function-declaration.md)）；`unsafe trait` 见 [4.15.4](../004-functions/415-unsafe.md#4154-在-trait-中)。可见性的确切规则在[可见性](../005-modules/506-visibility.md)中定义。

`#[sealed]` trait 只在其定义所在的包中实现；其他包仍然可以在约束和调用中使用它（[6.2.15](../006-meta-system/602-built-in-attributes.md#6215-sealed-只在定义它的包中实现)）。

## 2.8.9. 同一模块中不得重复定义

在同一模块中声明两个同名的 trait 是编译错误。在一个 trait 中声明两个同名的方法是编译错误（没有重载）。
