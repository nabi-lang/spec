# 2.13. 类型路径

由 `::` 词法单元（[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)）连接的标识符序列是指向某个项的路径。本文档只定义类型位置上的路径和关联项访问；模块导入的确切含义在 [`use` 声明](../005-modules/505-use.md)中另行说明。

```
type_path       ::= path_root ("::" path_segment)*
path_root       ::= "pkg" | "sup" | "::" identifier | identifier
path_segment    ::= identifier type_args?
                  | identifier "::" "<" type_expr ">" "::" identifier  // assoc item
```

## 2.13.1. 模块路径

如果类型名定义在另一个模块中，就把到达该位置的模块路径与类型名一起写出。

```nabi
core::time::Duration
core::collections::List
pkg::geom::Point
```

每个标识符指名一个模块或项。最后一个标识符是类型名；前面的部分是说明在哪里能找到该类型的路径。

## 2.13.2. 路径的起点

路径的第一个标识符必须是当前模块中可见的名称：它自己的某个子模块的名称、用 `use` 导入的名称，或上下文关键字。上下文关键字是表示包根的 `pkg` 和表示父模块的 `sup`；它们的确切含义在[路径](../005-modules/507-paths.md)中定义。

## 2.13.3. 关联项访问

`::` 也用于访问附加在类型上的项。

```nabi
i32::MAX                  // const item of an integer type
Point::origin()           // static method
List::<i32>::new()        // generic given with turbofish
T::Output                 // associated type of a trait parameter
```

关联项访问有几种形式。静态方法以 `Type::method(args)` 调用。关联 `const` 以 `Type::CONSTANT` 引用，trait 参数的关联类型以 `T::AssocType` 引用。

经由类型别名的路径指名别名所指类型的项。有 `type B = S<i32>;` 时，`B::mk(x)` 就是 `S::<i32>::mk(x)`；有 `type Al<T> = S<T>;` 时，`Al::mk(x)` 就是推断实参的 `S::mk(x)`，`Al::<i32>::mk(x)` 就是 `S::<i32>::mk(x)`。不带实参指名的泛型别名，只有当别名所指类型的实参是别名的互不相同的参数时，才把实参留给推断；否则，由与别名固定实参一致的、别名所指类型的带类型槽给出实参（有 `type Matrix<T> = Array<T, 2>;` 时的 `let m: Matrix<f64> = Matrix::zeros([2, 3]);`），或者写出实参（有 `type Names<V> = Dict<str, V>;` 时的 `Names::<i32>::new()`）。

经由别名的结构体字面量和结构体模式指名别名所指的结构体。有 `type Q = P<f64>;` 时，`Q { x: 2.0 }` 就是 `P::<f64> { x: 2.0 }`，模式 `Q { x }` 匹配 `P<f64>`。不带实参写出的泛型别名按同一规则推断实参（有 `type A<T> = P<T>;` 时的 `A { x: 3 }`）；否则写出实参（有 `type D<T> = P<List<T>>;` 时的 `D<i64> { x: [1] }`）。

## 2.13.4. 完全限定路径

当同一个方法名存在于多个 trait 和一个固有 `impl` 中而产生歧义时，或者必须说明通过哪个 trait 引用关联类型时，使用完全限定形式。

```nabi
<i32 as Clone>::clone(42i32)
<T as ops::Add<U>>::Output
<Self as Iterator>::next(ref self)
```

完全限定形式是 `<Type as Trait>::item`。`Type as Trait` 写在尖括号内，后面跟着 `::` 以及项名或方法调用。这里 `as` 是关键字用法，说明通过哪个 trait 看待该类型。同一个 `as` 词法单元在表达式位置（二元用法）中有不同的含义，在表达式规范中定义。

完全限定形式在类型位置（作为关联类型的 `<T as Tr>::X`）和调用位置（`<T as Tr>::m(x)`）都有效。对泛型 trait，选择与实参匹配的 impl。`<T as Tr<A>>::X` 以及在约束 `T: Tr<A>` 下的 `T::X` 是 `Tr<A>` impl 的 `X`，它可能与同一类型的 `Tr<B>` impl 的 `X` 不同。`<Sq as Shape>::Out` 这样的投影在函数签名、`static` 类型、结构体字段、枚举载荷和类型别名（无论是否泛型）中都有效，与类型、trait 和 impl 的声明顺序无关。在限定的关联类型之后进行调用，即 `<T as Tr>::X::m()`，在表达式中有效。

完全限定形式在日常使用中很少出现。只有在需要消除歧义或自动推断失败时，才显式使用它。

## 2.13.5. turbofish：表达式位置上的泛型

要在值位置给出泛型实参，使用 `::<...>` 形式。

```nabi
parse::<i32>("42")
collect::<List<f64>>(iter)
```

也就是说，在函数名或方法名后附加 `::<...>`，在该位置给出泛型实参。在类型位置（`:` 之后、`->` 之后、`as` 之后等），直接写 `<...>`，不带 `::`。

之所以有这个区别，是因为在值位置 `<` 会与比较运算符产生歧义。

## 2.13.6. 绝对类型路径

从根开始的绝对路径用 `pkg::` 书写（[路径](../005-modules/507-paths.md)）。Nabi 中不存在以 `::` 开头的形式。

## 2.13.7. 路径与和类型的组合

路径本身是[类型表达式](201-type-expressions.md)的简单名称或路径形式，可以与和类型（`|`）、泛型应用（`<...>`）等其他组合子自由混用。

```nabi
core::io::Lines<T>
pkg::Foo | std::Bar
List<core::range::Range<i32>>
```

优先级按[类型表达式](201-type-expressions.md)中的定义适用：路径（`::`）和泛型应用（`<...>`）的结合比和类型（`|`）更紧。

## 2.13.8. 空路径与单独的 `::`

空路径（只有 `::` 而没有标识符）总是编译错误。同样，标识符之间连续的 `::`（`A::::B`）也是错误。
