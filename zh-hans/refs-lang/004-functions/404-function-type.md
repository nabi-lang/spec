# 4.4. 函数类型

函数和闭包都是值，这些值都有类型。本文档详细介绍[类型表达式](../002-types/201-type-expressions.md)中的函数类型形式。

## 4.4.1. 形式

函数类型的形状如下：

```nabi
fn(ParamTypes) -> ReturnType
fn(i32, i32) -> i32
fn(str) -> bool
fn() -> ()
```

`fn` 关键字之后是带括号的参数类型列表，然后是 `->` 和返回类型。不写参数名，只写类型。省略返回类型时为 unit，所以 `fn(i32, i32)` 与 `fn(i32, i32) -> ()` 视为相同。

可能抛出错误的函数的类型在与函数声明相同的位置写 [`throws`](414-throws.md)。

```nabi
fn(str) throws -> i32               // untyped: throws (any Error)
fn(str) throws (ParseError) -> i32  // typed
fn(Path) throws (IoError)           // return type omitted: unit
```

`unsafe fn(..) -> R` 是调用时需要 `unsafe` 的函数的类型。`unsafe fn` 和 C 函数作为值具有这个类型。在期望 `unsafe fn` 的位置可以使用安全的 `fn` 值，反之则不行（[4.15](415-unsafe.md#unsafe-fn-types)）。

`throws` 是函数类型的一部分，所以调用可抛出错误的函数类型的值时，同样必须用 `try` 标注。`rethrows` 只用于函数声明，从不用于函数类型。

## 4.4.2. 函数值与闭包值的类型

函数和闭包都用同一种形式的函数类型表达。

```nabi
fn add(x: i32, y: i32) -> i32 { x + y }
let f = add;                                    // f: fn(i32, i32) -> i32

let g = |x: i32| -> i32 { x * 2 };              // g: fn(i32) -> i32
let h: fn(i32) -> i32 = |x| x + 1;              // explicit type
```

由于函数和闭包具有相同的函数类型，凡是要求函数类型的位置，两者都适用。

要求 trait 的位置也是如此。闭包无论是否有捕获，都是其 `fn(..) -> R` 类型的值，所以它满足为该类型实现的 trait：它可以作为约束 `F: Tr` 的实参、`Tr` 方法的接收者和其运算符的操作数，覆盖实现 `impl<F: Tr> Other for F` 也涵盖它。当约束的 trait 只为一个与闭包参数个数相同的 `fn` 类型实现时，该类型为闭包未注解的参数提供类型。闭包也可以通过其 `fn` 成员填充 `fn(..) -> R | none` 槽。

```nabi
trait Integrand { fn integral(self, a: f64, b: f64) -> f64; }
impl Integrand for fn(f64) -> f64 {
    fn integral(self, a: f64, b: f64) -> f64 { (self(a) + self(b)) * 0.5 * (b - a) }
}
fn integrate<F: Integrand>(f: F, a: f64, b: f64) -> f64 { f.integral(a, b) }

let k = 3.0;
integrate(|x: f64| -> f64 { k * x }, 0.0, 2.0);     // 6: the closure as fn(f64) -> f64
integrate(|x| x * 2.0, 0.0, 2.0);                   // 4: x: f64 from the one fn impl
```

附加了 `+ Send` / `+ Sync` 的约束或槽会对捕获提出相应要求，见下文；错误会指出是哪个捕获，以及应当改写成什么。

## 4.4.3. 赋值给变量

函数或闭包可以赋值给显式标注了函数类型的变量。

```nabi
let f: fn(i32) -> i32 = |x| x * 2;
let g: fn(str, i32) -> str = format;
```

函数和闭包都可以赋值给函数类型的槽。

## 4.4.4. 作为函数实参和返回值

函数类型主要用作其他函数的实参和返回值。

```nabi
fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn make_adder(n: i32) -> fn(i32) -> i32 { |x| x + n }
```

第一种模式是以函数为实参的高阶函数；第二种模式以函数作为结果返回。这两种模式是函数类型的核心用法。

## 4.4.5. 集合中的函数

和其他类型一样，函数类型可以作为集合的元素类型。

```nabi
let handlers: List<fn(Event) -> ()> = [...];
let dispatch: Dict<str, fn(Request) -> Response> = {...};
```

把函数类型存进集合，就能把行为当作数据处理。

## 4.4.6. 带捕获的闭包

带捕获的闭包也用同样的函数类型表达。对用户来说，它们呈现同样的接口。

```nabi
let n = 10;
let f: fn(i32) -> i32 = |x| x + n;     // captures n, but the type is fn(i32) -> i32
```

捕获在内部附加到闭包对象上，并在调用时自动使用。因此用户代码不需要关心是否有捕获，捕获也不出现在类型中。

Rust 中 `Fn`、`FnMut` 和 `FnOnce` trait 的区分在 nabi 中不存在。捕获默认按值进行，这降低了语义上的复杂度。详细的捕获语义在[捕获语义](406-capture-semantics.md)中定义。

### `Send` / `Sync` 约束

在函数类型后附加 `+ Send` / `+ Sync`，就在类型中记录了该值可以跨越线程边界（[标准库 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)）。

```nabi
fn(i32) -> i32 + Send
fn() + Send + Sync
```

- 闭包字面量出现在 `+ Send` 位置时，每个普通捕获都必须是 `Send + Sync`：普通捕获是创建者会继续使用的句柄，所以两个线程都能访问它。`[move x]` 捕获只属于该闭包（[4.6.11](406-capture-semantics.md#4611-move-capture-the-move-x-capture-list)），所以只需要 `Send`。
- `+ Sync` 位置要求每个捕获都是 `Sync`，无论是否移动。`fn(..) + Send` 值不是 `Sync`，除非其类型写明：它背后的闭包可能拥有一个不是 `Sync` 的移动捕获。被多个线程同时调用的值（`static`、`par::*` 闭包）的类型是 `fn(..) + Send + Sync`。
- 派生任务的函数（`task::spawn_blocking`、`thread::spawn`、`par::Scope::spawn`）接受 `+ Send` 闭包，所以被派生的代码不与其创建者共享任何非 `Sync` 的值；`[move x]` 改为把值交给它。`par::for_each`、`par::map` 和 `par::filter` 接受 `+ Send + Sync` 闭包，每个工作线程都会调用它。
- 要使用 `fn` 类型的变量，该变量的类型必须已经带有同样的约束。`fn() -> T` 值不适用于 `fn() -> T + Send` 位置。捕获了 `fn(..) + Send` 值的闭包，只有在把该值移入（`[move f]`），或者该值的类型是 `+ Send + Sync` 时，才能跨线程。
- 具名函数没有捕获，因此总是 `Send + Sync`。
- 约束是类型的一部分：分别以 `fn(i32) + Send + Sync` 值和 `fn(i32)` 值调用的泛型函数有两个实例。`fn(i32)` 上的 impl 对两者都适用。

```nabi
let c: Cell<i32> = Cell::new(0);
let xs: List<i32> = [1, 2];
let f: fn() + Send = || print("hi");             // OK: no captures
let g: fn() + Send = || c.set(1);                // ERROR: Cell is not Send
let h: fn() = || print("hi");
let k: fn() + Send = h;                          // ERROR: h's type has no + Send
let m: fn() + Send = || print(xs.len());         // ERROR: List is not Sync
let n: fn() + Send = [move xs] || print(xs.len());  // OK: the closure owns xs
```

## 4.4.7. 渐进的参数与返回

函数类型中的泛型参数无法直接用这种写法表达。泛型函数的值在使用处单态化为具有具体类型的函数值。

```nabi
fn id<T>(x: T) -> T { x }

let f = id;                       // ERROR: id itself is generic; T is not determined
let f: fn(i32) -> i32 = id;       // OK: monomorphized to i32
let g = id::<i32>;                // OK: monomorphized with turbofish
```

要在函数类型中放入动态槽，就为参数或返回写 `any`。

```nabi
fn(any) -> any
```

这是接受任意类型、可以返回任意类型的函数的类型。它会产生动态分派的开销。

## 4.4.8. 发散函数的类型

[返回](403-return.md)中介绍的发散函数，其类型在返回类型位置写 `!`（`fn() -> !`）。`fn(A) -> !` 值可以用在参数相同的任何 `fn(A) -> R` 位置，可抛出错误的 `fn(A) throws (E) -> !` 适用于 `fn(A) throws (E) -> R`。

```nabi
let f: fn() -> i32 = || panic("never");     // fn() -> ! value → fn() -> i32 position
```

### 函数值的表示

函数值是一个 `{code, env}` 对：`code` 是代码指针，`env` 是捕获环境。对具名函数的值，`env` 为 null。`extern "C"` 声明中函数类型的参数，其类型为 `extern "C" fn(..)`，即不带 `env` 的 C 函数指针，不能与 Nabi 函数值相互转换（[5.9.4](../005-modules/509-extern-block.md#c-function-pointer-types)）。方法路径（`S::new`、`i32::add`）也是函数值（[3.5.3](../003-expressions/305-member-access-and-method-call.md#method-path-values)）。

## 4.4.9. 函数类型之间的子类型关系（不存在）

> [!IMPORTANT]
> Rust 或 TypeScript 等语言中的函数类型子类型关系在 nabi 中不存在。函数类型不会自动转换为更一般或更具体的函数类型。参数类型和返回类型必须完全一致。

```nabi
let f: fn(i32 | f64) -> i32 = |x: i32| x;     // ERROR: parameter type does not match exactly
```

函数类型中的参数类型和返回类型必须完全一致；它们不会被拓宽为和类型。

唯一的例外是 4.4.8 的发散返回和错误通道。如果参数类型和返回类型相同，在期望可能抛出更多错误的函数类型的位置，可以使用抛出更少错误的函数值。这种转换（不抛出 → 抛出，较窄的错误 → 较宽的错误）可以由编译器生成的适配器实现。

```nabi
fn parse_typed(s: str) throws (ParseError) -> i32 { ... }

let p: fn(str) throws (ParseError) -> i32 = |s| s.len() as i32;   // OK: non-throwing → throwing
let q: fn(str) throws -> i32 = parse_typed;   // OK: throws (ParseError) → throws (any Error)
let r: fn(str) -> i32 = parse_typed;          // ERROR: throwing → non-throwing
```

`throws (A)` 函数值可以用在 `throws (A | B)` 位置；如果 `A` 实现了 `Error`，也可以用在未标注类型的 `throws` 位置。

这种转换适用于所有函数值，包括函数类型的变量和带捕获的闭包。编译器会创建一个包装原值的适配器 `{code, env}`。

```nabi
let g: fn(str) throws (ParseError) -> i32 = parse_typed;
let h: fn(str) throws -> i32 = g;             // OK: wrapped in an adapter
```

其动机是显式性。Nabi 的方针是：函数转换由用户以[转换](../002-types/214-cast-and-conversion.md)的显式形式写出。

## 4.4.10. 函数类型的求值

函数类型是不产生值的类型。它只规定接受什么形状的函数，本身没有求值行为。

调用函数遵循[成员访问与方法调用](../003-expressions/305-member-access-and-method-call.md)中描述的普通调用规则（`f(...)`）。

## 4.4.11. 不含参数名

函数类型不含参数名。定义（`fn name(x: i32)`）中有名字，但类型写法（`fn(i32)`）只保留位置。

```nabi
fn(i32, i32) -> i32                // OK
fn(x: i32, y: i32) -> i32          // ERROR: function types have no names
```

调用时的具名实参（[参数](402-parameters.md)）引用的是函数定义中的名字。因此，通过函数类型调用时，具名实参不起作用；通过值调用只接受位置实参。

```nabi
fn make_user(name: str, age: i32) -> User { ... }
let f: fn(str, i32) -> User = make_user;

make_user(name: "A", age: 30i32);    // named OK: uses the definition's names
f("A", 30i32);                        // positional OK
f(name: "A", age: 30i32);             // ERROR: the type has no names
```
