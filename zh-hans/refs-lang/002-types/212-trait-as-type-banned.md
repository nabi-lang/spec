# 2.12. trait 不能作类型

> [!IMPORTANT]
> 禁止在类型表达式位置直接写 trait 名。需要一个其实现类型在运行时才确定的值时，写 `any Trait`，它的开销在语法中可见（2.12.3）。

## 2.12.1. 禁止的内容

```nabi
let a: Number = 32i32;                          // ERROR: Number is a trait
fn process(x: Display) -> str { ... }            // ERROR
struct Holder { item: Iterator; }                // ERROR
```

如这些示例所示，trait 名直接作为变量类型、参数类型、返回类型或结构体字段类型出现，是编译错误。

## 2.12.2. 理由

这条禁令针对的是**隐藏的**动态分派。在类型位置直接写 trait 名，一眼看去与普通的静态类型无法区分，实际上却产生运行时分派。这种分派有实际的开销。实现在运行时选择并调用，因此在分支预测失败时，它可能比直接调用慢两倍以上。它需要额外访问分派数据，不利于缓存。最重要的是，它阻碍了编译器的内联和优化。

为了让科学计算和数据代码中的核心数值循环不会在不知情的情况下承担这种开销，Nabi 不允许把裸 trait 名用作类型。运行时多态只存在于写了 `any` 的地方。`any Trait` 之所以被允许，是因为它的动态分派开销通过 `any` 关键字在语法中可见。

## 2.12.3. 需要多态时：四种替代方案

需要多态时，Nabi 提供四种替代方案。

### 1. 用和类型表示封闭的候选集合

如果候选集合事先固定，使用和类型。

```nabi
let a: i32 | f32 | f64 = 32i32;
fn process(x: i32 | f32) -> str { ... }
```

这适用于候选在编译期固定的情况。内存和分派的开销是显式的（见[和类型](206-sum-types.md)），模式匹配可以检查穷尽性。

### 2. 用泛型表示开放的多态

如果候选是开放的，使用泛型。

```nabi
fn process<T: Display>(x: T) -> str { ... }
struct Holder<T: Iterator> { item: T; }
```

这适用于候选在编译期不固定或因调用者而异的情况。单态化产生直接调用，因此开销接近于零（见[泛型](210-generics.md)），并且可以通过 trait 约束调用方法。

### 3. 实现类型在运行时确定时使用 `any Trait`

如果不同实现类型的值必须混放在同一个槽或集合中，使用受约束的 `any`。

```nabi
let shapes: List<any Shape> = [Circle { r: 1.0 }, Square { side: 2.0 }];
fn report(e: any Error) -> str { f"{e}" }
struct ConfigError { msg: str; cause: any Error | none; }
```

`any Trait` 槽只接受实现了 `Trait` 的类型的值（在编译期检查），并且可以在其值上调用 `Trait` 及其父 trait 的方法。调用按运行时标签分派，开销模型与 [`any` 槽](202-any-slot.md)相同。确切规则在 [2.2.10](202-any-slot.md#2210-受约束的-anyany-trait) 中定义。

### 4. 用 `some Trait` 表示函数的隐藏返回类型

函数可以返回一个具体类型，而其调用者只能通过 trait 看到它：`-> some Iterator<Item = i32>`（[4.1.13](../004-functions/401-function-declaration.md#4113-不透明返回类型some-trait)）；参数也可以接受这样的类型：`x: some Display`（[4.1.14](../004-functions/401-function-declaration.md#4114-some-参数)）。要把该类型存入字段或在签名中指名，给它起一个名字：`type Evens = some Iterator<Item = i32>;`（[4.1.15](../004-functions/401-function-declaration.md#4115-具名不透明类型type-x--some-trait)）。分派是静态的，与泛型相同：`some Tr` 是一个被隐藏的静态类型，而 `any Tr` 持有任意实现了 `Tr` 的类型的值，并在运行时分派。

```nabi
pub fn evens(n: i32) -> some Iterator<Item = i32> { (0..n).filter(|x| x % 2 == 0) }
```

## 2.12.4. 选择标准

用以下标准在各替代方案之间选择。

| 情况                                                          | 选择        |
|---------------------------------------------------------------|-------------|
| 候选是一个小的固定集合（3 到 5 个）                           | 和类型      |
| 候选可以由用户 / 库扩展                                       | 泛型        |
| 函数返回一个调用者无需指名的类型                              | `-> some Trait` |
| 该隐藏类型要存入字段或在签名中指名                            | `type X = some Trait;` |
| 候选只在运行时确定（例如插件、异构集合）                      | `any Trait` |
| 必须持有任何东西，没有约束                                    | `any`       |

当候选类型只能在运行时确定时，使用 `any Trait`。它会产生动态分派开销，但这种开销被显式地集中在写了 `any` 的地方。

## 2.12.5. 允许父 trait 和约束表达式

需要说明的是：被禁止的只是 trait 名出现在类型位置。以下写法都是允许的。

```nabi
trait Ord: Eq { ... }                            // supertrait declaration
fn f<T: ops::Add + Display>(x: T) { ... }              // trait bound
fn g<T>(x: T) where T: Ord, T: Clone { ... }      // where clause
impl Display for Point { ... }                    // trait position of an impl
```

在这些位置，trait 名表达的是约束，而不是类型。

## 2.12.6. 编译错误的形式

当 trait 名出现在类型位置时，编译器在该类型处报告 E0071：

```text
error[E0071]: trait `core::fmt::Display` cannot be used as a type directly
  = help: use `any Display` for a value of any type implementing it, or a generic parameter `<T: Display>`
```

帮助信息给出了上述替代方案中的两种。另外两种是由具体类型组成的和类型（`SomeType | OtherType`），以及用于返回类型的 `some Trait`。
