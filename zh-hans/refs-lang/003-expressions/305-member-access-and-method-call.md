# 3.5. 成员访问与方法调用

`.` 词法单元（[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)）以两种形式访问附属于某个值的项。

```
member_expr     ::= primary "." identifier
                  | primary "." int_lit               // tuple position
method_call     ::= primary "." identifier ("::" "<" type_expr ("," type_expr)* ">")?
                    "(" arg_list? ")"
arg_list        ::= expr ("," expr)* ","?
```

## 3.5.1. 成员访问：`obj.field`

```nabi
let p = Point { x: 1.0f64, y: 2.0f64 };
let xv = p.x;               // 1.0
let yv = p.y;               // 2.0
```

这里 `obj` 是任意表达式，`field` 遵循[标识符](../001-lexical/104-identifiers.md)的形式。结果是该字段的值。

### 元组 `struct` 与元组的按位置访问

```nabi
let pair = Pair(1i32, 2i32);
let first = pair.0;
let second = pair.1;
```

点之后的十进制数字被解释为位置索引；这与[数字字面量](../001-lexical/106-numeric-literals.md)中 `1.foo` 规则的词法切分相同。例如，`pair.0` 切分为三个词法单元：标识符 `pair`、`.` 和整数 `0`。解析器把它们解释为按位置访问。

### 静态解析与动态解析

如果左侧的静态类型已知，字段位置在编译期确定，并编译为直接内存访问。如果左侧是 [`any` 槽](../002-types/202-any-slot.md)，则发生动态分派：在运行时检查标签，并查找相应的字段位置。

## 3.5.2. 方法调用：`obj.method(args)`

```nabi
let s = "hello";
let n = s.len();
let p = Point { x: 1.0, y: 2.0 };
let d = p.distance(other);
```

形式为 `expr . method_name ( arguments )`。方法解析查看左侧的静态类型，查找定义在该类型上的方法，或该类型所满足的 trait 的方法。

### 方法解析顺序

同名方法可以定义在多处。编译器按以下顺序查找候选：

1. 首先是固有 `impl`（[`impl`](../002-types/209-impl.md)）中定义的方法。
2. 然后是 trait `impl` 的方法，限于左侧类型所满足、且在当前模块的作用域内的 trait：`core` 的 trait（其中包括 prelude 的 trait：它们就是内置类型自身的方法）、用 `use` 导入的 trait（在模块中或外围块中）、在同一模块中定义的 trait，以及类型参数接收者的约束。`std` 或包中的 trait 需要 `use`（对 `(0..6).outcomes()` 要写 `use std::random::SampleRange;`），与 Rust 相同。通过类型书写的关联函数（`Point::origin()`）能看到这些 trait 以及其他所有标准库 trait。元模板写出的方法名，按模板所在模块作用域内的 trait 查找（[6.9.9](../006-meta-system/609-template-tokens.md#names-in-an-expansion)）。

3. 当两者都没有该方法，而接收者的类型有一个无参数的方法 `iter()` 时，若 `iter()` 返回的类型有 `m`，则 `x.m(args)` 即 `x.iter().m(args)`。零参数的 `x.min()` / `x.max()` 也以同样方式转发，尽管存在 `Ord::min` / `max`（它们接受一个参数）。因此集合可以响应迭代器方法：`xs.map(f).sum()`（[stdlib 2.7.14](../../refs-standard-libraries/200-core/207-iter.md#2714-collections-answer-iterator-methods)）。

类型为类型参数的接收者（`x: T`，`T: Root`）只拥有其约束所给出的东西，所以在泛型主体中，`x.sqrt()` 对每个实例都是 `Root::sqrt`，对带有固有 `sqrt` 的类型（`T = Complex<f64>`）的实例也是如此。

如果存在多个 trait 候选，而其中恰好一个来自在该模块中定义或 `use` 的 trait（固有方法总是优先），则选择那一个。否则，同时存在多个候选是有歧义的编译错误。此时用户必须用[类型路径](../002-types/213-type-paths.md)的完全限定形式显式消解。

只有不在作用域内的 trait 才提供的方法是错误（E0135），错误会写出需要添加的 `use`：``method `hi` comes from trait `a::Greet`, which is not in scope``，帮助信息为 ``add `use pkg::a::Greet;` ``。其他包的 trait 以其公开所在的路径命名（`linalg::AsVector`，而不是声明它的模块）。完全限定形式（`a::Greet::hi(x)`）不需要 `use`。

当同一泛型 trait 的多个 impl 都提供该方法时（对 `List<i64>` 有 `Conv<f64>` 和 `Conv<i64>`），由 trait 的实参选出一个：类型参数的约束（`x.conv()`，`x: X`，`X: Conv<f64>`）、路径的实参（`Conv::<f64>::conv(x)`）或限定路径的实参（`<List<i64> as Conv<f64>>::conv(x)`）。

### 方法接收者的自动 `ref`

方法调用处的接收者（`obj`）按方法的 `self` 种类（`self` / `ref self`）自动处理。调用处无需写 `ref`。

```nabi
p.distance(other);          // self
p.replace_with(other);      // ref self: caller does not write 'ref'
```

这是方法接收者的特殊规则，不同于普通函数调用的显式 `ref` 规则。详情定义于函数规范。

### 方法调用的脱糖

```
obj.method(a, b)
```

上面的表达式在语义上等价于：

```
<typeof(obj) as TraitContaining_method>::method(obj, a, b)
```

（若定义在固有 `impl` 中，则为 `Type::method(obj, a, b)`。）

这一脱糖遵循与[运算符分派](301-operator-dispatch.md)中 trait 方法映射相同的机制。`.` 词法单元本身不带 `#[infix(".")]` 这样的映射，但所得调用的形状相同。

### Turbofish：显式指定泛型方法

方法的泛型实参可以用[类型路径](../002-types/213-type-paths.md)的 turbofish 形式显式给出。

```nabi
let v = expr.parse::<i32>();
let m = arr.collect::<List<f64>>();
```

形式为 `.method::<...>(args)`：`.` 之后是方法名，然后是 turbofish，再是实参括号。

## 3.5.3. 与静态项访问的区别

`Type::item`（[类型路径](../002-types/213-type-paths.md)）访问附属于类型的项。

```
Type::item       // static method / associated const / associated type (no instance)
obj.item         // member / method attached to an instance (instance required)
```

这两种形式是完全不同的访问。词法单元不同（`::` 与 `.`），含义也不同。

### 方法路径值

不带括号书写的方法路径（`T::method`、`Trait::method`、`<T as Tr>::method`、`i32::add`、`S::new`）是一个函数值。

| 方法种类         | 函数值的类型                                 |
|------------------|----------------------------------------------|
| 静态方法         | `fn(args) -> R`                              |
| 实例方法         | `fn(Self, args) -> R`（`self` 是第一个实参） |

```nabi
let mk = Point::new;               // fn(f64, f64) -> Point
let add = i32::add;                // fn(i32, i32) -> i32
xs.map(Point::norm);               // fn(Point) -> f64
```

- 带 `ref self` 或 `ref` 参数的方法不能作为值（[4.7.6](../004-functions/407-ref-keyword.md#476-ref-and-types)）。
- 会抛出错误的方法给出会抛出错误的函数类型（`fn(..) throws (E) -> R`）。
- 对于 `Tr::m`，或 `Self` 未确定的泛型类型的方法，函数类型由期望类型确定。没有期望类型时，由之后的调用确定（`let f = Box::get; f(b)`，[2.10.2](../002-types/210-generics.md#type-argument-inference)）；若仍未确定，则是 “type annotations needed” 错误。
- 泛型方法的类型实参由 turbofish 或期望的函数类型确定。
- `#[intrinsic]` 方法也可以作为值。

附属于实例的形式，如 `obj.method`，不是函数值（3.5.4）。

## 3.5.4. `.method`（不调用）

当不带括号写 `.method` 时，Nabi 不把它解释为函数值，也不赋予部分应用或方法引用之类的其他含义。

```
let f = obj.method;        // ERROR (E0047): not a method call; the expression is incomplete
let f = || obj.method();   // wrap in an explicit closure
```

这一决定避免了隐式部分应用的歧义，并让用户直接表明意图。

## 3.5.5. 安全成员访问：`?.`

对可能不存在的值进行点访问，使用可选链书写，定义于[可选链与空值合并](308-optional-chaining-and-coalescing.md)。
