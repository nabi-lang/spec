# 2.14. error：错误 trait 与类型

`core::error` / `core::panic` 模块是错误处理的权威定义（`ZeroError` 与 `NonZero` 一起位于 `core::num::niche`）。Nabi 只有一种错误模型。可能抛出错误的函数在签名中写 `throws`，调用方用 `try` 标记该调用（[refs-lang 4.14](../../refs-lang/004-functions/414-throws.md)）。`Result<T, E>` 是用于**把错误作为值保存**的普通 enum，它不是传播错误的手段。

本章定义 100 个 lang 槽中的三个：`panic_fn` / `oom_fn` / `unreachable_fn`。

## 2.14.1. 模型概览：抛出用 `throws`，保存用 `Result`

```nabi
// (a) throwing: throws and try
fn parse_int(s: str) throws (ParseError) -> i32 {
    if s.is_empty() || !s.chars().all(|c| c.is_ascii_digit()) {
        throw ParseError::new(s, 0, "not a digit");
    }
    try s.parse::<i32>()
}

let n = try parse_int("42") catch e: ParseError {
    log(f"parse failed: {e:?}");
    0
};

// (b) holding: Result
let results: List<Result<i32, ParseError>> =
    inputs.map(|s| Result::catching(|| try parse_int(s)));

for r in results {
    match r {
        Ok(v) => total = total + v,
        Err(e) => failures.push(e),
    }
}
```

选择哪一种遵循一条规则：**错误必须流向调用方时用 `throws`，错误必须保留下来以后再处理时用 `Result`。**

| 场景                                             | 工具                    | 原因                                      |
|--------------------------------------------------|-------------------------|-------------------------------------------|
| 可能失败的操作（I/O、解析、查找）                | `throws`                | 每个调用点的 `try` 让控制流清晰可见       |
| API 边界 / 公开 API                              | `throws (E)` / `throws` | 错误类型在签名中公开                      |
| 收集并行执行的结果、把结果传给回调               | `Result<T, E>`          | 把错误作为值保存                          |
| 汇集多个错误一次性报告、缓存结果                 | `Result<T, E>`          | 作为值存储，以后再处理                    |
| 忽略失败，只需要值                               | `try?`                  | 以 `T \| none` 接收                       |
| 失败即程序 bug                                   | `try!`                  | 失败时 panic                              |

## 2.14.2. `Result<T, E>`：用于保存的 enum

`Result<T, E>` 是 `core::error` 中的普通 enum。

```nabi
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

> [!NOTE]
> Nabi **没有 `Some` / `None` 这样的 enum 变体**。`Option<T>` 本身**不存在**，
> 用和类型 `T | none` 代替。`Result` 之所以定义为单独的 enum，是因为**两个分支都携带有意义的数据**。

### 核心方法

`Result` 的核心方法如下：

```nabi
impl<T, E> Result<T, E> {
    fn is_ok(self) -> bool;
    fn is_err(self) -> bool;
    fn ok(self) -> T | none;
    fn err(self) -> E | none;

    fn unwrap(self) -> T;                               // panics on Err: `Result::unwrap: called on `Err(<e>)`; ...`
    fn unwrap_or(self, default: T) -> T;
    fn unwrap_or_else(self, f: fn(E) -> T) -> T;

    fn map<U>(self, f: fn(T) -> U) -> Result<U, E>;
    fn map_err<F>(self, g: fn(E) -> F) -> Result<T, F>;
    fn and_then<U>(self, f: fn(T) -> Result<U, E>) -> Result<U, E>;
    fn or_else<F>(self, g: fn(E) -> Result<T, F>) -> Result<T, F>;

    fn get(self) throws (E) -> T;                       // throws on Err, back into the error flow
    fn catching(f: fn() throws (E) -> T) -> Result<T, E>;   // captures the error flow as a value
}
```

对 `Err` 调用 `unwrap` 会 panic，消息为 `Result::unwrap: called on `Err(<e>)`` 加上错误的文本，然后是 `; use `try r.get()` to throw it instead`（0.5.10）。

当 `T` 和 `E` 都实现 `Debug`、`Clone`、`PartialEq`、`Eq` 和 `Hash` 时，`Result<T, E>` 也实现它们（2.14.8）。

`get` 和 `catching` 连接两种方式。`catching` 调用一个可能抛出错误的函数，成功时把结果保存为 `Ok`，出错时保存为 `Err`；`get` 取出保存的值，如果是 `Err` 则重新抛出该错误。

### `Result` 没有 `?`

[refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410-the--operator---early-return-of-none) 的后缀 `?` 只用于 `T | none`。对 `Result` 值使用 `?` 时，编译器给出如下诊断。

```nabi
fn load(r: Result<Config, IoError>) -> Result<Config, IoError> {
    let cfg = r?;
    // error[E0064]: `?` only works on `T | none`, found `Result<Config, IoError>` - to propagate errors use `throws` and `try`
    ...
}
```

要把保存的错误传播给调用方，把函数声明为 `throws`，并写 `try r.get()`。

```nabi
fn load(r: Result<Config, IoError>) throws (IoError) -> Config {
    try r.get()
}
```

## 2.14.3. `Error` trait：错误的接口

所有错误类型的接口都由 `Error` trait 定义。

```nabi
trait Error: Display + Debug {
    const fn source(self) -> any Error | none {
        none                                  // default: no chain
    }
}
```

错误类型实现这个 trait，用 `Display` 输出消息，用 `Debug` 做调试显示。无类型的 `throws` 是 `throws (any Error)` 的简写，所以无类型抛出函数抛出的值必须实现 `Error`。要不论类型地保存或传递错误，使用[受约束的 `any`](../../refs-lang/002-types/202-any-slot.md) `any Error`（`any Error | none` 即 `(any Error) | none`）。

### `#[derive(Error)]`

`#[derive(Error)]` 以默认的 `source()` 实现 `Error`。它对字段没有要求；超 trait `Display` 和 `Debug` 来自各自的 derive / 属性或 impl，所以错误类型通常写成 `#[derive(Debug, Error)]` 加 `#[display("..")]`（[2.10.14](210-fmt.md#21014-derive-policy-summary)）。

### 错误链：`source()`

包装错误时，可以这样表达原因链。

```nabi
struct ConfigError {
    msg: str;
    cause: any Error | none;
}

impl Error for ConfigError {
    fn source(self) -> any Error | none {
        self.cause
    }
}

impl Display for ConfigError {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(self.msg)
    }
}
```

把值存入 `cause` 时，编译器在编译期检查其静态类型是否实现了 `Error`。用这种方式构建的链可以被 REPL 或日志记录器沿着追溯，打印原因的回溯信息。

```nabi
let cfg = try parse_config(text) catch e: ParseError {
    throw ConfigError { msg: "bad config", cause: e };
};
```

## 2.14.4. 内置错误类型

标准库默认提供的错误类型定义如下。它们的字段是私有的（[0.5.10](../000-meta/005-notation.md)）：每个类型都用 `new` 构造，通过 getter 和 `message()` 读取。

```nabi
struct TypeError { .. }
impl TypeError {
    fn new(expected: str, actual: str) -> TypeError;
    fn expected(self) -> str;
    fn actual(self) -> str;
    fn message(self) -> str;          // "expected `i32`, found `str`"
}

struct ValueError { .. }
impl ValueError {
    fn new(message: str) -> ValueError;
    fn message(self) -> str;
}

struct OverflowError { .. }
impl OverflowError {
    fn new(operation: str, type_name: str) -> OverflowError;
    fn operation(self) -> str;        // "try_from" / ...
    fn type_name(self) -> str;        // "i32" / "i64" / ...
    fn message(self) -> str;          // "try_from: value out of range for `i32`"
}

struct ParseError { .. }
impl ParseError {
    fn new(input: str, position: usize, message: str) -> ParseError;
    fn input(self) -> str;
    fn position(self) -> usize;       // byte offset in input where parsing failed
    fn message(self) -> str;          // why, without the input Display adds
}

struct ConversionError { .. }
impl ConversionError {
    fn new(message: str) -> ConversionError;
    fn message(self) -> str;
}

struct ZeroError;                     // core::num::niche; message()
```

每个类型都实现 `Error`、`Display`、`Debug`、`PartialEq`、`Eq` 和 `Clone`。其 `Display` 就是 `message()`；`ParseError` 的 `Display` 会把输入放在前面（`cannot parse "x7": invalid digit`）。各错误的抛出位置：

| 类型              | 抛出方                                                                     |
|-------------------|----------------------------------------------------------------------------|
| `TypeError`       | 运行时强制转换失败、类型检查失败                                           |
| `ValueError`      | 一般的无效值（解析器 / API 等）；`throw "msg"` 抛出的就是它               |
| `OverflowError`   | `try_into` 等检查（checked）转换中超出范围（算术溢出则触发 panic）         |
| `ParseError`      | `str → number` 等解析失败（NABION 使用 `NabionError`）                     |
| `ConversionError` | `TryFrom::Error` 的默认值；非数值转换失败（[2.11.4](211-conversion.md#2114-tryfromu-trait---fallible-conversion)）；`NonNan` / `Finite` 的 `try_from` |
| `ZeroError`       | 向 `NonZero::try_from` 传入 0（[2.2.2](202-non-zero.md#222-nonzerot---zero-forbidden)） |

### 致命错误

以下类型归类为致命（Fatal）错误。

```nabi
struct StackOverflowError;     // Fatal: cannot be caught
struct OutOfMemoryError;       // Fatal: cannot be caught
struct InternalError;          // Fatal: compiler-internal bug
```

它们也都有 `message()`（`"stack overflow"` 等）。致命错误不是 `throws` 错误，不能用 `catch` 捕获。它与 panic 一样，向 stderr 打印消息，并以退出码 `101` 结束进程（[refs-lang 3.14.8](../../refs-lang/003-expressions/314-error-control-flow.md#3148-panics-and-fatal-errors)）。

## 2.14.5. `panic_fn` / `oom_fn` / `unreachable_fn`：编译器钩子

错误模型的最底层是编译器直接调用的钩子函数。

```nabi
mod panic {                         // core::panic
    #[lang("panic_fn")]
    #[track_caller]
    fn panic(msg: str) -> ! {
        nabi_panic_at(msg)            // prints to stderr with the caller's location, then exit code 101
    }

    #[lang("oom_fn")]
    fn oom() -> ! {
        nabi_oom()                    // "fatal error: out of memory", exit code 101
    }

    #[lang("unreachable_fn")]
    #[track_caller]
    fn unreachable() -> ! {
        nabi_panic_at("internal error: entered unreachable code")
    }
}
```

每个函数的返回类型都是 `-> !`（never），永远不会正常返回。`panic` 和 `unreachable` 是 `#[track_caller]`：报告的位置是 `#[track_caller]` 调用链之外的第一个调用方。

### panic 发生的位置

编译器在以下位置调用 `panic_fn`。所有情况下 panic 都不能用 `catch` 捕获，`finally` 和 `Drop` 也不会运行。

- `try!` 的错误路径：消息是错误值的 `Display`（没有则用 `Debug`）
- 除以零、整数溢出、边界检查失败
- 标准库中显式的 panic 调用，例如对 `Err` 调用 `Result::unwrap`

`throw` 不是 panic。`throw` 只产生抛出函数的错误结果（[refs-lang 8.9.7](../../refs-lang/008-abi/809-lowering-control.md#897-throw)），不经过 `panic_fn`。

### OOM 处理

GC 分配失败时，按如下方式处理 OOM。运行时无法按名称调用 Nabi 函数，所以它直接调用运行时符号 `nabi_oom`，而不是 `oom()`（[refs-lang 8.5.7](../../refs-lang/008-abi/805-extern-symbols.md#857-diagnostic-area)）。

```
nabi_gc_alloc_slow:
    if !try_collect_and_retry() {
        nabi_oom();                       // "fatal error: out of memory", exit 101
    }
```

core 的 `oom()` 调用同一个 `nabi_oom`，所以两条路径产生相同的输出和退出码。

OOM 是致命错误，不会被捕获。

### 使用 `unreachable`

`unreachable` 标记不可能到达的分支。

```nabi
match (x: i32 | f64) {
    v: i32 => process_int(v),
    v: f64 => process_float(v),
    // the compiler confirms every variant is handled; no unreachable position
}

// or explicitly.
fn require_positive(x: i32) -> u32 {
    if x < 0 { core::panic::unreachable() }     // guaranteed by the caller
    x as u32
}
```

编译器在以下位置自动插入 `unreachable`：
- `match` 穷尽检查之后的死分支
- `if let ... else` 的 else 保证发散的位置

### `assert` / `assert_eq` / `assert_ne`

```nabi
fn assert(cond: bool, msg: str = "") -> ();
fn assert_eq<T: PartialEq + Debug>(left: T, right: T, msg: str = "") -> ();
fn assert_ne<T: PartialEq + Debug>(left: T, right: T, msg: str = "") -> ();
```

它们通过 core prelude 在任何地方都可以使用。条件为假时触发 panic（退出码 101）。消息如下；`msg` 不为空时追加 `: msg`。

```
assertion failed[: msg]
assertion `left == right` failed[: msg]
  left: <Debug of left>
 right: <Debug of right>
```

`assert_ne` 使用 `left != right`，其余相同。

## 2.14.6. `try` / `catch` / `throw` / `throws` / `finally`

语法由 [refs-lang 3.14 错误控制流](../../refs-lang/003-expressions/314-error-control-flow.md)和 [refs-lang 4.14 `throws` 签名](../../refs-lang/004-functions/414-throws.md)规定；本节只介绍标准库的约定。

### 签名模型

函数签名中的抛出行为有五种形式。

```nabi
fn foo() -> T                      // non-throwing: never throws
fn foo() throws -> T               // untyped: throws (any Error)
fn foo() throws (E) -> T           // typed single: only E
fn foo() throws (A | B | C) -> T   // typed sum: only A | B | C
fn foo(f: fn() throws -> T) rethrows -> T   // throws only when f throws
```

无类型的 `throws` 是公开 API 推荐的默认形式，因为它让 API 规格保持简洁。`throws (E)` 适合库内部 API、嵌入式环境，或捕获方需要按错误分支处理的情况。带类型的和类型 `throws (A | B | C)` 把 Swift SE-0413 的 typed throws 扩展到了和类型。接受函数参数的标准库高阶函数使用 `rethrows`，这样传入不抛出错误的闭包时，调用可以不写 `try`。

### `try` / `catch` 形式

`catch` 是后缀表达式；捕获到错误时，处理块的值成为整个表达式的值。

```nabi
let result = try compute() catch e: TypeError {
    log(f"type error: {e}");
    default_value
};
```

### `finally`

`finally` 块可以用于资源清理等。

```nabi
let f = try File::open(path);
try {
    process(f);
} catch e: IoError {
    log(f"io error: {e}");
} finally {
    try? f.close();     // runs on both normal completion and error; a close error is dropped
}
```

`finally` 的执行语义：
- 正常完成时运行。
- catch 之后完成时运行。
- 未捕获的错误传播时，先运行它，再继续传播错误。
- 通过 `return` / `break` / `continue` 离开时运行。
- 发生 panic 或致命错误时不运行，因为进程立即结束。

在 `finally` 内部抛出的错误会替换当前的错误。

### `throw` 与分支类型

`throw value;` 是语句，以 `throw` 结尾的块是发散的。因此分支的结果类型只反映其他分支的类型。例如，`if c { x } else { throw e; }` 的结果类型就是 `x` 的类型。

## 2.14.7. `Result` 与 `throws` 之间的转换

两种工具可以显式地相互转换。

### 抛出调用 → `Result<T, E>`

`Result::catching` 调用一个可能抛出错误的闭包并保存结果。也可以直接用 `catch` 表达式构造。

```nabi
let r: Result<i32, ParseError> = Result::catching(|| try parse_int(s));
let r: Result<i32, ParseError> = try Ok(parse_int(s)) catch e { Err(e) };
```

### `Result<T, E>` → 抛出

`get` 重新抛出 `Err`。外层函数的 `throws` 必须接受该错误类型。

```nabi
fn first_ok(rs: List<Result<i32, ParseError>>) throws (ParseError) -> i32 {
    try rs[0].get()
}
```

### 取出为 `T | none`

要丢弃错误只保留值，对抛出调用使用 `try?`，对保存的 `Result` 使用 `ok()` / `err()`。

```nabi
let a: i32 | none = try? parse_int("42");
let r: Result<i32, ParseError> = Result::catching(|| try parse_int("42"));
if let Ok(v) = r { process(v); }
if let n: i32 = r.ok() { process(n); }     // extracted as T | none
```

## 2.14.8. trait 实现矩阵

各错误类型实现的 trait：

| 类型                 | Display | Debug         | Error | 备注                           |
|----------------------|---------|---------------|-------|--------------------------------|
| `TypeError`          | ✓       | ✓             | ✓     |                                |
| `ValueError`         | ✓       | ✓             | ✓     |                                |
| `OverflowError`      | ✓       | ✓             | ✓     | 检查转换中超出范围             |
| `ParseError`         | ✓       | ✓             | ✓     |                                |
| `ConversionError`    | ✓       | ✓             | ✓     | 默认的 `TryFrom::Error`        |
| `ZeroError`          | ✓       | ✓             | ✓     | 位于 `core::num::niche`        |
| `FmtError`           | ✓       | ✓             | ✓     | 写入失败（[2.10](210-fmt.md)） |
| `Utf8Error`          | ✓       | ✓             | ✓     | 唯一的 UTF-8 错误（[2.15](215-string.md)） |
| `IoError`            | ✓       | ✓             | ✓     | std 层级（依赖目标）           |
| `StackOverflowError` | ✓       | ✓             | ✓     | 致命：不能捕获                 |
| `OutOfMemoryError`   | ✓       | ✓             | ✓     | 致命：不能捕获                 |
| `InternalError`      | ✓       | ✓             | ✓     | 致命：编译器 bug 时            |
| `Result<T, E>`       | ✗       | ✓ (T,E:Debug) | ✗     | 用于保存的 enum；`T` 和 `E` 实现 `Clone` / `PartialEq` / `Eq` / `Hash` 时也实现它们 |
| `any Error`          | ✓       | ✓             | ✓     | 受约束的 any，可以成链         |

`Display` 提供对用户友好的消息；例如 `TypeError::new("i32", "str")` 显示为 ``"expected `i32`, found `str`"``。`Debug` 按原样显示结构。表中每个错误类型也都实现 `PartialEq`、`Eq` 和 `Clone`（0.5.10）。`any Error` 值通过运行时标签调用其实际类型的 `Display` / `Debug` / `source`。

## 2.14.9. 与其他语言的比较

与其他主流语言相比，Nabi 的错误模型定位如下。

| 方面             | Rust           | Swift                     | Java            | Nabi                                       |
|------------------|----------------|---------------------------|-----------------|-------------------------------------------|
| 基本模型         | `Result<T, E>` | `throws`                  | `throws`        | `throws`；`Result` 用于保存               |
| 调用点标记       | `?`            | `try`                     | **无**          | `try` / `try?` / `try!`                   |
| typed throws     | **无**         | `throws(E)`（单一类型）   | `throws E1, E2` | `throws (E)` / `throws (A \| B)`          |
| `?` 运算符       | `Result?`      | 可选链                    | **无**          | 只用于 `T \| none`                        |
| Option 类型      | `Option<T>`    | `Optional<T>`             | `Optional<T>`   | **无**：使用 `T \| none`                  |
| 实现方式         | 返回值         | 返回值（错误寄存器）      | 栈展开          | 返回值 `{ is_err, payload }`              |
| panic            | `panic!`       | `fatalError`              | `Error`         | `panic_fn`：不能捕获，退出码 101          |

Nabi 主要有三点不同。第一，没有 `Option`，由和类型 `T | none` 代替，少了一个类型。第二，错误传播统一到 `throws` 和 `try`，`Result` 只用于把错误作为值保存。第三，带类型的和类型 throws `throws (A | B | C)` 比只接受单一类型的 Swift typed throws 表达力更强。
