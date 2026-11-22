# 5.2. `const`

`const` 关键字（一个上下文关键字，只在项的开头才是关键字，[1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)）声明在编译期确定的值。语法如下：

```
const_decl      ::= "const" identifier ":" type_expr "=" const_expr ";"
const_fn        ::= "const" fn_decl
const_block     ::= "const" block
```

## 5.2.1. 基本形式

```nabi
const PI: f64 = 3.14159f64;
const MAX_RETRY: i32 = 3i32;
const NAME: str = "nabi";
```

声明依次由 `const` 关键字、名称、`:`、类型、`=` 和一个 `const` 表达式组成。类型必须显式写出；与 `let` 不同，它不会被推断。值的位置只接受 `const` 表达式，即可以在编译期求值的表达式。

## 5.2.2. 允许的位置

`const` 可以在下列所有位置声明。

```nabi
// module level
const PI: f64 = 3.14159f64;

// inside a function body
fn area(r: f64) -> f64 {
    const FACTOR: f64 = 3.14159f64;
    FACTOR * r * r
}

// inside an impl block
impl Circle {
    const UNIT_AREA: f64 = 3.14159f64;
    fn unit() -> Circle { Circle { r: 1.0f64 } }
}

// inside a trait: associated const
trait Numbers {
    const ZERO: Self;
    const ONE: Self;
}
```

函数体中的 `const` 是其所在块的一个项（[3.9.1](../003-expressions/309-block-expressions.md#391-form)）：在整个块以及块中声明的项中都可见。它不读取周围代码的任何局部变量或参数；此时的错误提示改写为 `let`（`let half = n / 2;`）。它可以读取所在泛型函数或 `impl` 的常量参数，以及其类型参数的关联常量和 `T::size()`：这时它每个实例各有一个值（5.2.7）。

```nabi
fn mid<const N: usize>(xs: [i32; N]) -> i32 {
    const HALF: usize = N / 2;      // one value per instance
    xs[HALF]
}
```

## 5.2.3. `const` 的含义：在每次使用处内联

`const` 是一个值，不是一个地址。它的值在每个使用处内联。

```nabi
const X: i32 = 42i32;

fn f() {
    let a = X;                    // a = 42i32 (X is inlined)
    let b = X + 1i32;              // b = 43i32 (X is inlined, then evaluated)
}
```

`const` 的结果不驻留在任何内存位置，所以不能取它的地址。

```nabi
let p = &X;                       // ERROR (E0012): Nabi has no `&` borrows
```

Nabi 没有 `&` 运算符，所以没有任何表达式能取 `const` 的地址。用于 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 的裸指针由[类型表达式](../002-types/201-type-expressions.md)中的指针类型定义。

## 5.2.4. 常量表达式与求值器

`const` 项、数组长度、常量泛型实参、`struct` 字段默认值、`static` 初始化式，以及接收常量实参的 `const fn` 字面量处理器（[6.5.8](../006-meta-system/605-literal-magic.md#658-const-compatibility-of-literal-handlers)），都使用同一个编译期求值器。

可以求值的内容：

- 值：各种位宽的整数、浮点数、`bool`、`char`、`str`、元组、`struct`、`enum`、和类型、定长数组。
- 表达式：字面量、对其他 `const` 的引用、常量泛型参数、运算符、`const fn` 调用。
- 控制流：`if`、`match`、`loop` / `while` / `for`、带标签的 `break`、`return`、`throw`。允许递归和泛型 `const fn`。

不能求值的内容：对非 `const fn` 函数的调用（I/O、时间、随机数等）、读取 `static`、引用外层的 `let` 变量。

下列情况是编译错误：

- 整数溢出（包括仅由字面量和常量组成的表达式折叠后的结果，如 `let m: u8 = 2 * 200`）、除以零、`MIN / -1`、移位量大于等于位宽，以及超出范围的浮点数 → 整数转换。回绕运算只能通过 `wrapping_*` 进行。
  被常量条件排除的代码永远不会运行，也不检查：在 `N = 0` 的泛型实例中，`if N == 0 { 0 } else { a[N - 1] }` 中的 `N - 1`、`if N == 0 { return ..; }` 之后块中其余部分里的 `N - 1`，以及 `N > 0 && a[N - 1] == x` 中的 `N - 1` 都不是错误。实例中的溢出会连同生成该实例的调用及其常量实参一起报告。
- 求值期间发生 panic 或未捕获的 `throw`。
- 超出求值步数或调用深度的限制。限制由实现定义（nabi：4,000,000 步，深度 128）。
- `const` 之间的循环引用。

## 5.2.5. `const fn`：可在编译期求值的函数

```nabi
const fn double(x: i32) -> i32 {
    x * 2i32
}

const fn gcd(a: i64, b: i64) -> i64 {
    if b == 0i64 { a } else { gcd(b, a % b) }
}

const TWO: i32 = double(1i32);                  // compile time
const TWELVE: i64 = gcd(24i64, 36i64);          // compile time

let y = double(user_input);                      // runtime call
```

- `const fn` 只能调用 `const fn`。`const fn` 内部通过 `fn` 值或闭包进行的调用只在求值时检查，因为静态检查无法追踪它们。标准库把纯函数标记为 `const fn`。
- 函数体必须是纯的。I/O 和访问可变全局变量都是错误。
- 在编译期成功求值的结果，与同样实参下的运行时结果相同。在运行时会溢出的运算，在编译期是错误（5.2.4）。
- `const fn` 不能以类型作为实参（`const fn sizeof(T: type)` 是错误；帮助信息改写为 `const fn sizeof<T>() -> usize { T::size() }`）。类型内省属于元块。
- `const fn` 也可以像普通函数一样在运行时调用。

### 泛型 `const fn`

```nabi
const fn id<T>(x: T) -> T { x }

const A: i32 = id(42i32);                        // evaluated after monomorphization with T = i32
```

### 使用堆的结果

如果结果包含堆分配（`List`、缓冲区、闭包等），它不会嵌入二进制文件。这样的 `const` 在编译期求值并检查，在运行时于每次使用处构造。

每次使用都是一个新值，所以对 `const` 的修改调用（对 `List` 调用 `XS.push(4)`，或对 `const` 结构体调用 `ref self` 方法）只会改变一个随即被丢弃的副本。这是编译错误：把值复制到变量中，再修改该变量（`let xs = XS; xs.push(4);`）。

## 5.2.6. `const` 与其他机制的正交性

```
generic <T>     = type parameterization (call-site specialize)
meta block      = code generation (impl, struct, etc. at AST level)
const fn        = value computation (compile-time pure function)
```

这三种机制彼此正交。它们的领域互不重叠，所以可以自由组合。

## 5.2.7. `const` 何时求值

模块级、函数体级和 `impl` 级的 `const` 在包构建时求值一次。依赖泛型参数的 `const`（泛型 `impl` 的关联 `const`，或读取常量参数的函数体 `const` 或 `const { .. }` 块）按每次单态化求值。不使用堆的结果嵌入二进制文件，没有运行时开销。

## 5.2.8. 与 `struct` 字段默认值的关系

[`struct`](../002-types/203-struct.md) 字段默认值是常量表达式（5.2.4）。

## 5.2.9. 与 `const` 泛型的关系

[常量泛型](../002-types/211-const-generics.md)的实参是成为类型一部分的常量，由同一个求值器（5.2.4）求值。

## 5.2.10. `const` 块

`const { .. }` 是由求值器（5.2.4）在编译期求值的表达式。它的值是块类型的一个常量；类型为 `()` 的块是一项检查。

```nabi
fn chunk<const N: usize>(xs: [i32; N]) -> usize {
    const { assert(N > 0, "chunk needs at least one element") };
    const { N / 2 }
}
```

- 它读取的内容与函数体 `const` 相同：常量、`const fn` 调用、周围代码的常量参数，以及其类型参数的关联常量和 `T::size()` / `T::align()`。读取 `let` 变量是错误，帮助信息提示去掉 `const`。
- 在泛型函数中，它在实例的实参已知后，每个实例求值一次。在定义处对泛型主体进行的检查（[2.10.9](../002-types/210-generics.md#2109-body-checking)）对该块做类型检查，但不求值。
- 求值时发生 panic 或 `assert` 失败，是在生成该实例的调用处报告的编译错误，并指明实例（`` const block failed in `chunk::<0>`: chunk needs at least one element ``），`assert` 作为次要位置。`nabi check` 同样会报告它：它与 `nabi build` 一样，实例化程序所能到达的内容。
- 通过检查的块在运行时没有任何开销；它的值像任何 `const` 一样被嵌入。
- 对于简单类型（数字、`bool`、`char`、`str`、`()` 或由它们构成的数组），求值器知道 `T::size()`；对于其他类型，这是错误，提示改用常量参数或 `List`。
