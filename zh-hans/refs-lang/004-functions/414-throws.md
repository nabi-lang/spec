# 4.14. `throws` 签名

本节定义函数签名的 `throws` / `rethrows` 形式，它们与[错误控制流](../003-expressions/314-error-control-flow.md)中的 `throw`、`try`、`catch` 和 `finally` 配合使用。

> [!IMPORTANT]
> Nabi 只有一种错误模型：`throws`。可能抛出错误的函数在签名中写上 `throws`，调用方用 `try` 标记调用。标准库的 [`Result<T, E>`](../../refs-standard-libraries/200-core/214-error.md) 只在必须把错误作为值保存时使用。规则只有一行：**抛出它 → `throws`；保存它 → `Result`**。

## 4.14.1. 形式

在[函数声明](401-function-declaration.md)的 `fn_decl` 中，`throws_clause?` 位于参数括号之后、返回类型箭头之前。

```
throws_clause   ::= "throws" ("(" type_expr ")")?
                  | "rethrows"
```

因此 `fn_decl` 的完整形式是：

```
fn_decl ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
```

其中 `fn_modifiers` 是 [4.1 函数声明](401-function-declaration.md)中定义的 `unsafe? async? yield?`。

```nabi
fn name(args) -> R                              // non-throwing
fn name(args) throws -> R                       // untyped: shorthand for throws (any Error)
fn name(args) throws (E) -> R                   // typed: single error type
fn name(args) throws (A | B | C) -> R           // typed: sum type of errors
fn name(f: fn(T) throws -> U) rethrows -> R     // throws only when the function argument throws
```

`throws` 和 `rethrows` 是上下文关键字，只在参数右括号 `)` 之后才是关键字（[1.5.4](../001-lexical/105-keywords.md#154-上下文关键字)），放在返回类型箭头之前。`throws` 之后可选的括号给出可能抛出的错误类型。

## 4.14.2. 不抛出的函数

```nabi
fn add(x: i32, y: i32) -> i32 { x + y }
```

在没有 `throws` 的函数中，任何错误都不能离开该函数。如果函数体中 `throw` 产生的错误，或用 `try` 标记的调用产生的错误，没有被函数内部的 `catch` 完全处理，就是编译错误。调用方不用 `try` 就可以调用这个函数。

```nabi
fn bad() -> i32 {
    throw ValueError::new("nope"); // ERROR (E0062): error leaves a non-throwing function
}
```

这条规则强制签名写明函数是否可能抛出，从而杜绝错误的静默传播。入口点是唯一的例外：没有返回类型的显式 `fn main()` 不声明也可以抛出（[4.16.4](416-entry-point.md#4164-退出码与未处理的异常)）。

## 4.14.3. 无类型的 `throws`

```nabi
fn parse(s: str) throws -> i32 { ... }
```

无类型的 `throws` 是 `throws (any Error)` 的简写。函数可以抛出任何实现标准库 [`Error`](../../refs-standard-libraries/200-core/214-error.md) trait 的类型的错误，抛出的值作为 `any Error` 值（一个[受约束的 `any`](../002-types/202-any-slot.md)）交付。调用方在 `catch` 中用类型模式收窄到具体类型，或者用 `catch e: any Error` 或兜底分支捕获全部错误。

当错误种类的集合对调用方并不封闭时（如公开 API），推荐使用无类型形式。

## 4.14.4. 有类型的 `throws`：单一类型与和类型

```nabi
fn fetch_user(id: i64) throws (NotFound) -> User { ... }
fn parse_or_load(s: str) throws (ParseError | IoError | TimeoutError) -> Data { ... }
```

括号中可以写出确切的可抛出类型，也可以直接使用和类型（`A | B | C`）。

有类型形式适合内部 API 或库 API、嵌入式环境，以及错误种类集合封闭的情况。此时编译器可以检查调用方的 `catch` 分支是否覆盖了所有错误类型；如果覆盖了，该表达式就被视为不再抛出。

```nabi
fn caller() -> Data {             // non-throwing: all three types handled
    try parse_or_load(input)
    catch e: ParseError { ... }
    catch e: IoError { ... }
    catch e: TimeoutError { ... }
}
```

## 4.14.5. `throws` 与函数体的一致

每一个可能离开函数的错误都必须与签名的 `throws` 兼容。错误来源于函数体中的 `throw` 语句，以及用 `try` 标记的可抛出调用；被函数内部 `catch` 处理的错误不参与检查。

```nabi
fn f() throws (A) -> i32 {
    throw A_instance;             // OK
    throw B_instance;             // ERROR (E0062): only A allowed
}
```

兼容规则：对于有类型的 `throws (E)`，每一个来源错误类型都必须是 `E` 的成员。对于无类型的 `throws`，每一个来源错误类型都必须实现 `Error`，值作为 `any Error` 交付。

错误类型之间没有自动转换。要把被调用方的错误作为另一种类型传播，请用 `catch` 包装它并再次抛出。

```nabi
fn load_config(path: str) throws (ConfigError) -> Config {
    try parse_toml(read_text(path))
    catch e: IoError { throw ConfigError::io(e); }
    catch e: ParseError { throw ConfigError::syntax(e); }
}
```

## 4.14.6. 调用方的义务

调用 `throws` 函数的表达式必须位于被 `try` 覆盖的表达式之内。不带 `try` 调用可抛出的函数是编译错误。`try` 覆盖其右侧的整个表达式，因此 `try a(b())` 同时覆盖 `a` 和 `b` 的调用，`try { a(); b(); }` 覆盖块中的所有调用（[3.14.2](../003-expressions/314-error-control-flow.md#3142-try标记调用点)）。

用 `try` 标记的调用产生的错误由最近的 `catch` 处理；未处理的错误传播到外围函数，外围函数的 `throws` 必须接受该错误类型（4.14.5）。

```nabi
fn caller() {                     // non-throwing
    parse(s);                     // ERROR (E0062): throwing call not marked with try
    try parse(s);                 // ERROR (E0062): error propagates but caller is not throws
}

fn caller2() throws {             // propagates
    try parse(s);                 // OK
}

fn caller3() {                    // catches
    try parse(s) catch _ {
        ...
    };
}

fn caller4() -> i32 {             // discards the error or turns it into a panic
    let a = try? parse(s) ?? 0;   // try?: none on error, then 0
    let b = try! parse(t);        // try!: panic on error
    a + b
}
```

因此，每一个错误可能流经的位置都在代码中以 `try` 的形式可见，而错误传播到多远则体现在函数签名的 `throws` 中。

## 4.14.7. `rethrows`

```nabi
fn map<U>(self, f: fn(T) throws -> U) rethrows -> List<U> {
    let out: List<U> = [];
    for x in self {
        out.push(try f(x));
    }
    out
}
```

`rethrows` 是供接收函数类型参数的高阶函数使用的形式。`rethrows` 函数只在作为实参传入的函数或闭包抛出时才抛出。

- 如果调用处传给可抛出函数类型参数的每一个实参都不抛出，该调用就不抛出，调用时不写 `try`。
- 如果这些实参中有任何一个可抛出，该调用就可抛出，抛出的错误类型就是这些实参的错误类型。调用方用 `try` 标记它。
- 在 `rethrows` 函数的函数体中，唯一可以离开函数的错误来自对函数类型参数的调用。如果函数体直接抛出自己的错误，就是编译错误。

```nabi
let lens = names.map(|s| s.len());              // non-throwing closure: no try
let nums = try names.map(|s| try parse(s));     // throwing closure: try required
```

当 `rethrows` 函数被当作值使用时，由期望类型（一个函数类型）决定使用哪个实例。没有期望类型时，错误参数被设为 `any Error`。

## 4.14.8. 函数类型中的 `throws`

`throws` 也可以附加在[函数类型](404-function-type.md)上。

```nabi
fn(T) -> U                    // non-throwing
fn(T) throws -> U             // untyped: throws (any Error)
fn(T) throws (E) -> U         // typed
```

不抛出的函数值可以用在参数和返回类型相同的可抛出函数类型位置；`throws (A)` 函数值可以用在 `throws (A | B)` 位置，或者（如果 `A` 实现 `Error`）用在无类型 `throws` 的位置。反方向不允许。详见 [4.4](404-function-type.md)。

## 4.14.9. `throws` 与 `async` 和 `yield` 的组合

三种修饰符可以组合用在同一个函数上。

```nabi
async fn fetch(url: str) throws (NetworkError) -> Data { ... }

yield fn lines(s: str) throws (ParseError) -> str { ... }

async yield fn read_paginated(url: str) throws (NetworkError) -> Page { ... }
```

`throws` 是签名位置上的关键字，与 `async` 或 `yield` 正交组合。

## 4.14.10. `throws` 对和类型签名的影响

```nabi
async fn fetch_data() throws (NetworkError) -> Data { ... }
```

有人会问，调用这个函数得到的是否是一个保存 `Data | NetworkError` 的 Future。

答案是 **`TryAsyncFuture<Data, NetworkError>`**（[4.1.11](401-function-declaration.md#协程函数的结果类型)）。错误不会混入结果类型；它在 `await` 处通过错误通道交付。因此执行 `await` 的一方要用 `try` 标记它。`TryAsyncFuture` 不实现普通的 `Future` trait。

```nabi
let f = fetch_data();              // f: future::TryAsyncFuture<Data, NetworkError>
let d = try await f catch e: NetworkError {
    ...
};
```

这一决定避免了结果变成一个有结果和错误两个变体的 `enum`。

`yield` 和 `async yield` 遵循同样的模式。带 `throws (E)` 的生成器是 `TryGenIter<T, E>`，流是 `TryAsyncStream<T, E>`；错误在 `next` 处通过错误通道交付。

### `try for`

对 `TryIterator`（可抛出生成器的 `TryGenIter`、`io::Bytes` 等）的 `for` 在每次 `next` 时都可能抛出，因此写作 `try for`。这个 `try` 只覆盖迭代本身：被迭代的表达式和每一次 `next`。循环体不被覆盖；其中的可抛出调用要用自己的 `try` 标记（[3.13.3](../003-expressions/313-loop-expressions.md#3133-for基于迭代器的重复)）。对 `TryStream` 的 `for await` 同理。

```nabi
fn total(path: str) throws (IoError | ParseError) -> i64 {
    let n: i64 = 0;
    try for line in lines(path) {           // IoError from each next propagates
        n += try i64::from_str(line);       // the body marks its own call
    }
    n
}
try for await ev in events() { try handle(ev); }
for line in lines("a.txt") { ... }          // ERROR (E0062): `next` can throw; write `try for`
```

| 形式 | 某一步出错时 |
|---|---|
| `try for` | 传播到最近的 `catch`，否则传出函数 |
| `try! for` | 触发 panic（[3.14.3](../003-expressions/314-error-control-flow.md#3143-try-与-try)） |
| `try? for` | 结束循环；`for` 表达式的值为 `none`（否则为 `()`） |
| `try for .. catch e: E { .. }` | `catch` 附加在整个 `try for` 表达式上 |

循环体中的 `try` 像任何嵌套的 `try` 一样路由其错误：交给循环之后的 `catch`，或交给循环的 `try?` / `try!`，否则传出函数。要在出错后继续，请对 `.results()` 进行循环，它把每一步作为 `Result` 给出（[标准库 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator可能失败的迭代)）。

## 4.14.11. 方法上的 `throws`

与普通[方法](409-method.md)一样，方法也可以有 `throws` 签名。

```nabi
impl Database {
    fn query(self, sql: str) throws (SqlError) -> ResultSet { ... }
}
```

调用方的义务和传播规则与自由函数完全相同。

实现 trait 的 impl 方法的 `throws` 必须与 trait 方法的 `throws` 相同。如果 impl 增加了 `throws` 或放宽了错误类型，就是编译错误（E0074）。不写 `throws` 的 impl 方法取 trait 方法的 `throws`，其中的错误类型指代它们在 trait 处所指代的类型：`impl Display for P { fn fmt(self, ref f: Formatter) { try f.write_str(..); } }` 抛出 `FmtError`。

运算符 trait 的方法（`Add::add` 等）声明时不带 `throws`，因此运算符不会抛出。可能失败的运算以具名方法提供，或者返回 `Result`（[标准库 2.3](../../refs-standard-libraries/200-core/203-ops.md)）。用户运算符同样如此：在用 `throws` 声明的方法上使用 `#[infix]` / `#[prefix]` / `#[postfix]` 是错误（[6.7.3](../006-meta-system/607-operator-magic.md#673-连接机制)）。

## 4.14.12. 基于值的实现模型

可抛出函数以值的形式返回其结果。返回值是一个成功标志与一个载荷组成的对，载荷保存成功值或错误值之一。错误通道由签名区分，因此成功类型和错误类型可以是同一个类型。调用处的 `try` 检查标志并分支；出错时，它转到最近的 `catch`，或者把错误作为外围函数的错误结果返回。

在这一模型中，无错误路径的开销只是一次分支，进入 `try` 或 `catch` 没有任何开销。不使用栈展开，也不使用运行时的错误交付函数。确切的返回值布局在 [8.6.5](../008-abi/806-calling-convention.md#865-抛出函数的返回基于值的错误传播) 中定义，降级在 [8.9.6](../008-abi/809-lowering-control.md#896-try--catch--finally基于值的分支) 中定义。

## 4.14.13. panic 与致命错误

panic 不是 `throws` 错误。panic（标准库的 `panic`、失败的 `try!`、除以零、整数溢出、边界检查失败）以及[错误控制流](../003-expressions/314-error-control-flow.md)中的致命错误不属于任何 `throws` 签名，也不能用 `catch` 捕获。运行时把以下内容写到标准错误（stderr），并以退出码 `101` 终止进程。

```
thread '<name>' panicked at <path>:<line>:<col>:
<message>
```

- `<name>` 是线程名：主线程为 `main`，未命名线程为 `<unnamed>`。
- 在被 C 调用的函数之下发生的 panic，会在报告之后中止（abort）进程而不是退出，或者把该函数的 `#[on_panic]` 值返回给 C（[5.9.12](../005-modules/509-extern-block.md#5912-ffi-安全的责任)）。用 `std::panic::set_hook` 设置的钩子在任何情况下都在报告之后运行（[标准库 3.12](../../refs-standard-libraries/300-std/312-panic.md)）。
- 位置是失败的表达式。在 [`#[track_caller]`](../006-meta-system/602-built-in-attributes.md#6212-track_caller-传递调用位置) 函数链内部，位置是该链之外的第一个调用位置。
- 环境中设置了 `NABI_BACKTRACE=1` 时，后面跟着一段 `stack backtrace:`：发生 panic 的线程调用栈中的函数，从最内层开始，每行一个，以 Nabi 路径表示（`   0: geom::distance`）。顶部的标准库 panic 函数被省略，最后一行 `note:` 说明如何查看它们。`NABI_BACKTRACE=full` 列出每一个栈帧及其返回地址（`   0: 0x000000010000a4f0 - geom::distance`），一直到第一个不是 Nabi 代码的栈帧（`<unknown>`）。未设置、为空或为 `0`：不输出栈回溯。[9.9.6](../009-tools/909-debug-tools.md#panic-栈回溯) 说明栈回溯如何生成以及在哪里可用。
