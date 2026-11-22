# 4.1. 函数声明

本节定义以 `fn` 关键字（[关键字](../001-lexical/105-keywords.md)）开头的具名函数的声明形式。

## 4.1.1. 形式

函数声明的语法如下：

```
fn_decl       ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
fn_modifiers  ::= "unsafe"? "async"? "yield"?
return_type   ::= "->" type_expr
param_list    ::= param ("," param)* ","?
```

其中 `visibility?` 是[可见性](../005-modules/506-visibility.md)中的 `pub`、`pub(pkg)`、`pub(sub)` 之一，或者省略（模块私有）。`fn_modifiers` 是 [unsafe](415-unsafe.md)、[async](412-async.md) 和 [yield（生成器）](411-generators.md)三个修饰符的位置，每个修饰符**按固定顺序写零次或一次**（详细规则见 [4.1.11 fn 修饰符的顺序与组合](#4111-fn-修饰符的顺序与组合)）。`throws_clause?` 的含义在 [throws](414-throws.md) 中定义；本文档不涉及可见性、修饰符和 throws 关键字本身的含义。

非正式地说，整体形状如下：

```nabi
fn name(parameters) -> ReturnType {
    body
}
```

各组成部分的含义如下：

- 以 `fn` 关键字开头。
- 函数名遵循[标识符](../001-lexical/104-identifiers.md)的形式。
- 随后是带括号的参数列表，详细规则在[参数](402-parameters.md)中定义。
- 返回类型可选：在 `->` 词法单元（[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)）之后写一个[类型表达式](../002-types/201-type-expressions.md)，或者写不透明的 `some Tr`（[4.1.13](#4113-不透明返回类型some-trait)）。
- 主体是一个[块表达式](../003-expressions/309-block-expressions.md)。

## 4.1.2. 最简单的示例

最简单的函数声明如下：

```nabi
fn greet() {
    print("hello");
}

fn add(x: i32, y: i32) -> i32 {
    x + y
}
```

## 4.1.3. 省略返回类型

返回类型的 `->` 部分可以省略。省略的含义取决于函数的[可见性](../005-modules/506-visibility.md)。

在模块私有或 `pub(sub)` 函数上：

- 返回类型从主体推断；主体最后一个表达式的类型即为返回类型。
- 如果无法推断或存在歧义，则是编译错误。

下面的示例省略了返回类型，但根据最后一个表达式的类型推断出 `i32`。

```nabi
fn double(x: i32) {              // return type omitted
    x * 2                         // inferred: i32
}
// the above is equivalent to fn double(x: i32) -> i32
```

如果主体不产生值（最后一个表达式以终止符结尾），返回类型为 unit（`()`）。

```nabi
fn print_msg(s: str) {
    print(s);                     // terminator at the end: returns unit
}
```

`pub` 或 `pub(pkg)` 函数会被其他模块调用，所以它的签名要写明，修改主体不能改变签名。在这类函数上，缺少 `->` 就表示 `()`，无论主体做什么，主体永不返回时也是如此。如果主体给出了值（最后一个表达式或 `return v` 的类型不是 `()`），则是编译错误（E0138），错误会指出应当写的类型。

```nabi
pub fn log(s: str) {              // returns ()
    print(s);
}

pub fn double(x: i32) {           // ERROR: the body gives i32; write `-> i32`
    x * 2
}

pub fn double2(x: i32) -> i32 {   // OK
    x * 2
}
```

固有 `impl` 的方法按自身的可见性遵循同样的规则。trait `impl` 的方法按 trait 声明的写法书写 `->`，因为返回类型必须与之一致（[2.9.1](../002-types/209-impl.md)）；只有未写类型的参数从 trait 取得类型（[4.2.2](402-parameters.md#422-省略类型)）。

如果写了返回类型，但函数主体与之不符，则是编译错误。

```nabi
fn bad() -> i32 {
    "hello"                       // ERROR: str is not i32
}
```

## 4.1.4. 省略参数类型

在模块私有或 `pub(sub)` 函数上，未写类型的参数视为隐式泛型（[`any` 槽](../002-types/202-any-slot.md)、[泛型](../002-types/210-generics.md)）。

```nabi
fn id(x) {                        // x: T (generic)
    x
}
// monomorphized by the type at the call site
let a = id(42i32);                // id<i32>
let b = id("hi");                 // id<str>
```

`pub` 或 `pub(pkg)` 函数要写明每个参数的类型；未写类型的参数是编译错误（[4.2.2](402-parameters.md#422-省略类型)）。

如果参数要作为动态槽，就显式写 `any`。

```nabi
fn dyn_id(x: any) {               // x is an any slot (runtime dispatch)
    x
}
```

## 4.1.5. 主体

函数主体遵循[块表达式](../003-expressions/309-block-expressions.md)的规则，并具有以下性质：

- 包含零条或多条语句。
- 可以以零个或一个最终表达式（不带终止符）结尾，其值成为函数的返回值。
- 可以在末尾用 `return`（[`return`](403-return.md)）显式返回。

## 4.1.6. 空主体

主体为空的函数返回 unit。

```nabi
fn nothing() { }                  // empty body: returns unit
```

## 4.1.7. 返回类型与主体的最后一个表达式

在显式写出返回类型的函数中，不带终止符的最后一个表达式成为返回值。

```nabi
fn f() -> i32 {
    42i32                         // last expression is the return value
}
```

在上面的示例中，`42i32` 没有终止符，所以它成为函数的返回值。如果带上终止符（`42i32;`），值会被丢弃，主体返回 unit，这与声明的 `-> i32` 冲突，是编译错误。

```nabi
fn f() -> i32 {
    42i32;                        // ERROR: terminator discards the value
                                  // (does not match return type i32)
}
```

## 4.1.8. 函数主体中的显式 `return`

在主体内部，`return` 关键字用于显式返回，详见 [`return`](403-return.md)。

```nabi
fn early(x: i32) -> i32 {
    if x < 0 { return -1; }
    x * 2
}
```

## 4.1.9. 可见性

函数上的可见性关键字（`pub`、`pub(pkg)`、`pub(sub)`）控制函数的公开范围。

```nabi
pub fn area(r: f64) -> f64 { ... }              // callable from external packages
pub(pkg) fn helper(x: i32) -> i32 { ... }       // only within the same package
fn _internal(s: str) { ... }                     // module-private (default)
```

各可见性级别的确切含义在[可见性](../005-modules/506-visibility.md)中定义。

## 4.1.10. 同一模块内不得重复定义

在同一模块中声明两个同名函数是编译错误（E0026）。Nabi 没有重载。

```nabi
fn add(x: i32, y: i32) -> i32 { x + y }
fn add(x: f64, y: f64) -> f64 { x + y }   // ERROR: duplicate
```

要按类型改变行为，可以组合 trait 和泛型（[`trait`](../002-types/208-trait.md)）。

模块中的函数、`const` 和 `static` 共享同一个值命名空间：在同一模块中，与其中另一项同名的 `const`、`static` 或函数同样是编译错误。元组结构体或单元结构体的构造器也在这个命名空间中（[2.3.7](../002-types/203-struct.md#237-同一模块中不得重复定义)）：`struct Point(i32, i32);` 与 `fn Point`、`const Point` 或 `static Point` 并存即为重复定义。参数个数不同、`async` 或类型参数都不能使两个函数成为不同的函数。要接受不同数量的实参，可以为一个函数的尾部参数提供默认值（[4.2.3](402-parameters.md#423-默认值)）。

## 4.1.11. fn 修饰符的顺序与组合

函数修饰符（`unsafe`、`async`、`yield`）**必须按以下顺序书写**：

```
unsafe → async → yield
```

违反这一顺序是语法错误（E0014）。同一修饰符写两次在语法层面也不可能（`unsafe?` 中的 `?` 只允许零次或一次）。

```nabi
unsafe async yield fn ok() -> i32 { ... }   // OK: prescribed order
async unsafe fn bad1() { ... }           // ERROR: async before unsafe
yield async fn bad2() { ... }            // ERROR: yield before async
unsafe unsafe fn bad3() { ... }          // ERROR: duplicate unsafe
```

这条规则保证含义相同的函数总是写成同一种形式，从而简化格式化工具、lint 工具和代码阅读。

### 8 种可能的组合

三个修饰符都是可选的，因此共有 8 种可能的组合，定义如下。

| 组合                    | 调用结果                               | 含义                 | 定义位置                           |
|-------------------------|----------------------------------------|----------------------|------------------------------------|
| `fn`                    | `R`                                    | 普通函数             | [4.1](401-function-declaration.md) |
| `unsafe fn`             | `R`                                    | 不安全函数           | [4.15](415-unsafe.md)              |
| `async fn`              | `AsyncFuture<R>`                   | 异步函数             | [4.12](412-async.md)               |
| `yield fn`              | `GenIter<R>`                       | 生成器               | [4.11](411-generators.md)          |
| `async yield fn`        | `AsyncStream<R>`                   | 异步流               | [4.13](413-stream.md)              |
| `unsafe async fn`       | `AsyncFuture<R>`                   | 不安全异步函数       | （各修饰符之和）                   |
| `unsafe yield fn`       | `GenIter<R>`                       | 不安全生成器         | （各修饰符之和）                   |
| `unsafe async yield fn` | `AsyncStream<R>`                   | 不安全异步流         | （各修饰符之和）                   |

带 `unsafe` 的组合很简单：整个函数主体成为不安全作用域（如 4.15 所定义），结果类型和协程行为遵循其他修饰符（`async` / `yield`）的定义。也就是说，`unsafe` 只是在主体内加入 `unsafe` 块所允许的操作（[4.15.2](415-unsafe.md#4152-unsafe---块表达式)）：调用 `unsafe fn`、解引用裸指针、读取 `union` 字段，以及在值移交之后继续使用它。

### 协程函数的结果类型

调用标有 `async` / `yield` 的函数（或块、闭包）会得到一个标准库类型。声明中的 `-> R` 给出元素 / 结果类型 `T`。

| 形式             | 结果类型          | 实现的 trait           | 带 `throws (E)` 时        |
|------------------|-------------------|------------------------|---------------------------|
| `yield fn`       | `GenIter<T>`      | `Iterator<Item = T>`   | `TryGenIter<T, E>`        |
| `async fn`       | `AsyncFuture<T>`  | `Future<Output = T>`   | `TryAsyncFuture<T, E>`    |
| `async yield fn` | `AsyncStream<T>`  | `Stream<Item = T>`     | `TryAsyncStream<T, E>`    |

这些类型位于 `iter`（`GenIter`、`TryGenIter`）、`future`（`AsyncFuture`、`TryAsyncFuture`）和 `stream`（`AsyncStream`、`TryAsyncStream`）中，不在 prelude 里：要通过其模块（`future::AsyncFuture<T>`）或 `use` 来命名。`Try*` 变体在下一个推进点抛出错误。例如，`TryGenIter<T, E>` 的 `next` 是 `next(ref self) throws (E) -> T | none`，`TryAsyncFuture` 在 `try await` 处抛出。这些变体不实现普通的 `Iterator` / `Future` / `Stream` trait（[4.14.10](414-throws.md#41410-throws-对和类型签名的影响)）；`TryGenIter` 实现 `TryIterator`，`TryAsyncStream` 实现 `TryStream`（[标准库 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator可能失败的迭代)、[2.8.11](../../refs-standard-libraries/200-core/208-stream.md#2811-trystream可失败的流)）。

```nabi
async fn fetch_user(id: i64) -> User { ... }
let f: future::AsyncFuture<User> = fetch_user(42i64);

yield fn lines(path: str) throws (IoError) -> str { ... }
let it: iter::TryGenIter<str, IoError> = lines("a.txt");
```

### 闭包与块形式

同样的顺序规则也适用于 `async` 和 `yield` 的闭包 / 块形式。例如，产生 Stream 的闭包总是写作 `async yield ||`；`yield async ||` 不是闭包形式，而是读作对 `async ||` 闭包应用 `yield`：在生成器之外这是错误（E0059，生成器之外的 `yield`），在生成器内部则产出该闭包。闭包 / 块形式见各修饰符的文档。

## 4.1.12. 参数、返回与主体的求值顺序

调用的求值顺序如下：

1. 求值实参表达式（从左到右）。
2. 匹配参数，开始求值函数主体。
3. 求值主体；完成时，最后一个表达式或 `return` 的值即为结果。
4. 把结果交给调用处。

详细的求值顺序在[参数](402-parameters.md)的参数求值顺序一节中定义。

## 4.1.13. 不透明返回类型：`some Trait`

```
opaque_ret ::= "some" (bound | "(" bound ("+" bound)* ")")
bound      ::= type_path | fn_type
```

函数的返回类型（`->` 之后，也包括 `throws (E) ->` 之后）可以是 `some Tr`：由主体选定、对调用者隐藏的、具有 `Tr` 的一个具体类型。`Tr` 是一个 trait，可以像在约束中那样带关联类型绑定（`some Iterator<Item = str>`），也可以是函数类型（`some fn(i32) -> i32`）。多个约束要放在括号中：`some (Iterator<Item = i32> + Clone)`。

```nabi
pub fn evens(n: i32) -> some Iterator<Item = i32> {
    (0..n).filter(|x| x % 2 == 0)
}

fn adder(k: i32) -> some fn(i32) -> i32 {
    |x| x + k                     // x: i32, from the bound
}

let total = evens(10).map(|x| x * x).sum();
```

- 具体类型是主体值的类型，按 [4.1.3](#413-省略返回类型) 推断，在 `pub` 函数上也是如此。调用是静态的：没有标签，也没有装箱，与[泛型](../002-types/210-generics.md)相同。
- 每个函数只有一个具体类型：返回路径类型不同是错误（`the return paths of `f` have different types ..`），帮助信息为 `` to return several types, write `-> any Tr` ``。
- 具体类型必须具有每个约束：否则在约束处报 `` `f` returns `ListIter<i32>`, which is not `Iterator<Item = str>` ``。
- 调用者只能看到约束：可以调用约束（及其父 trait）的方法；约束是迭代器时可以迭代它；可以把它传给期望具有这些约束的泛型参数的位置；约束是函数类型时可以调用它。命名或使用具体类型是错误：`` no method `contains` on `some Iterator<Item = i32>`: callers see only its bounds ``；`` `let b` expects `Range<i32>`, found `some Iterator<Item = i32>` ``。
- 同一个函数总是返回同一个类型，所以两次调用的结果属于同一类型。不同函数的结果是不同的类型。
- `some` 只能作为函数的返回类型、参数的类型（[4.1.14](#4114-some-参数)）或具名不透明类型的右侧（[4.1.15](#4115-具名不透明类型type-x--some-trait)）。在其他位置（`let` 注解、字段、类型实参、闭包参数）使用是错误，错误会说明应当写什么：变量不写注解，字段或类型实参写泛型 `<T: Tr>`，多种类型的值写 `any Tr`。
- 跨包时，函数主体随包的接口一起传递，所以依赖方的构建知道具体类型；其使用者仍然只能看到约束。
- 没有 `Fn` trait：函数约束就是函数类型 `fn(A) -> R`。

## 4.1.14. `some` 参数

参数的类型可以是 `some Tr`（约束同 [4.1.13](#4113-不透明返回类型some-trait)）：一个匿名类型参数。`fn show(x: some Display)` 就是 `T` 未命名的 `fn show<T: Display>(x: T)`。

```nabi
fn show(x: some Display) -> str { f"<{x}>" }
fn both(a: some Display, b: some Display) -> str { f"{a}{b}" }
fn count(it: some Iterator<Item = i32>) -> usize { it.count() }
fn twice(f: some fn(i32) -> i32, x: i32) -> i32 { f(f(x)) }

let s = both(1, 2.5);            // a: i64, b: f64
let n = count([1, 2, 3].iter());
let t = twice(|x| x * 3, 2);     // x: i32, from the bound
```

- 每次调用中，参数有一个静态类型，即实参的类型，与[泛型](../002-types/210-generics.md)相同：没有标签，也没有装箱。实参必须具有每个约束。
- 每个 `some` 参数都是独立的类型参数：`both(1, 2.5)` 中 `a` 和 `b` 的类型不同。
- 主体只能看到约束，与具名参数 `T: Tr` 相同。
- `some fn(A) -> R` 接受函数值（函数或闭包），闭包的参数从中取得类型。
- 自由函数、方法和 trait 方法都可以有 `some` 参数，并可与具名类型参数并用。
- 该类型没有名字：调用处的 `::<..>` 只给出具名类型参数。多写是错误，帮助信息说明 `some` 参数的类型就是实参的类型：传入实参，或者声明 `<T: Tr>` 以便用 `::<..>` 选择 `T`。

`any Tr` 和 `some Tr` 是一对。`any Tr` 是动态槽：它容纳任意具有 `Tr` 的类型的值，每次都可以不同，调用在运行时分派（[2.2](../002-types/202-any-slot.md)）。`some Tr` 是一个隐藏的静态类型：由被调用者的主体（对返回而言）或调用的实参（对参数而言）确定，调用静态分派。

## 4.1.15. 具名不透明类型：`type X = some Trait`

```
opaque_alias ::= visibility? "type" identifier "=" opaque_ret ";"
```

`type X = some Tr;` 为一个具有 `Tr` 的具体类型命名（约束同 [4.1.13](#4113-不透明返回类型some-trait)）。与 `-> some Tr` 不同，这个类型有名字，所以可以写在任何能写类型的位置：字段、`let` 注解、类型实参（`List<X>`）、参数和返回类型。

```nabi
mod nums {
    pub type Evens = some Iterator<Item = i64>;

    pub fn evens(n: i64) -> Evens {        // defines Evens
        (0..n).filter(|x| x % 2 == 0)
    }

    pub fn first(e: Evens) -> i64 | none { e.next() }
}

struct Shelf { evens: nums::Evens; more: List<nums::Evens>; }

fn total(e: nums::Evens) -> i64 { e.sum() }

let s = Shelf { evens: nums::evens(10), more: [nums::evens(4)] };
println(total(s.evens));                  // 20
```

- 定义函数是 X 所在模块中返回类型为 `X`（写作 `X` 或 `self::X`）的自由函数。每个定义函数的具体类型是其主体值的类型，推断方式同 `-> some Tr`，并且必须具有每个约束。定义函数必须至少有一个，并且必须都给出同一类型：一个也没有时报 `` `X` has no defining function ``（E0145），帮助信息为 ``a function in this module that returns `X` gives its type``；两个不一致时报 `` `g` returns `B`, but `X` is `A` (from `f`) ``（E0145），帮助信息建议返回同一类型或调用 `f`。
- 定义函数没有类型参数，`X` 也没有：泛型的定义函数是错误（E0145），其帮助信息给出写法 `fn f<T>(..) -> some Tr`，即每次调用一个隐藏类型（[4.1.13](#4113-不透明返回类型some-trait)）。
- 方法，以及返回 `List<X>` 或 `X | none` 的函数，不定义 `X`，只是使用它。
- 在 X 所在模块内部（不含其子模块），`X` 就是其具体类型：模块中的函数可以像对待该类型的任何值一样调用其方法、读取其字段、构建它。
- 在 X 所在模块外部，`X` 是一个只具有其约束的独立类型：值以 `X` 的身份被存储、传递、返回、放进 `List<X>`，并通过约束（及其父 trait）的方法使用，与 `-> some Tr` 的结果相同。具体类型是隐藏的：约束未提供的方法或字段报 `` no method `len` on `Evens`: outside `nums`, only its bounds are visible ``；在期望其他类型的位置使用 `X` 的值，或在期望 `X` 的位置使用其他类型的值（模块外的函数从自身主体返回 `X`），报 `` `let n` expects `i64`, found `X`: outside `m`, `X` is opaque ``（E0146），帮助信息建议把值保持为 `X`、从 X 的定义函数取得一个值，或者把代码移到 X 所在模块中。
- 约束为函数类型时（`type Step = some fn(i64) -> i64;`），值只能被调用，所以该类型在任何地方都保持为其具体类型。
- 调用和存储都是静态的：类型为 `X` 的字段以内联方式容纳具体类型，没有标签，也没有装箱。
- `pub type X = some Tr;` 导出这个名字。跨包时，定义函数的主体随包的接口一起传递（[4.1.13](#4113-不透明返回类型some-trait)），所以依赖方的构建能确定 `X` 的布局；依赖方只能看到约束。
- `type X = any Tr;` 是动态槽 `any Tr` 的普通类型别名（[2.6.9](../002-types/206-sum-types.md#269-与类型别名组合)）：`X` 的每个值的类型可以不同，调用在运行时分派。`type X = some Tr;` 是由其模块确定的一个静态类型。
