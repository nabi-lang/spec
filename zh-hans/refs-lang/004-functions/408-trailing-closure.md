# 4.8. 尾随闭包

当函数或方法调用的最后一个实参是闭包时，可以把该闭包移到括号外面。这种形式称为尾随闭包（trailing closure）。

## 4.8.1. 形式

尾随闭包的语法如下：

```
trailing_call   ::= primary "(" arg_list? ")" trailing_closure
                  | primary trailing_closure                    // 0-arg call
trailing_closure ::= ("|" closure_params? "|" ("->" type)?)? block
```

```nabi
function(args) |params| { body }
function(args) { body }            // zero-parameter shorthand
```

闭包紧跟在调用的括号之后，函数体是[块表达式](../003-expressions/309-block-expressions.md)中的块。

### 零参数简写：`{ body }`

没有参数的闭包可以从 `|| { body }` 简写为 `{ body }`。

```nabi
let h = thread::spawn { heavy_work() };
// == thread::spawn(|| { heavy_work() })

with_lock(mutex) { critical_section() };
```

这种简写只对紧跟在函数调用之后的 `{ ... }` 有意义。其他位置的 `{ ... }` 都是[块表达式](../003-expressions/309-block-expressions.md)中的普通块表达式。

### 一个或多个参数

有一个或多个参数时，写法如下：

```nabi
let doubled = xs.iter().map |x| { x * 2 };
let pos = xs.iter().filter |x| { x > 0 };
let opened = with_file(path) |f| {
    let data = f.read();
    process(data)
};
pairs.for_each |k, v| { print(f"{k} {v}") };
```

竖线（`|...|`）放在括号外面，紧挨着函数体之前。函数体始终是块：始终写作 `|x| { body }`，绝不写成单行的 `|x| body`。

为了可读性和一致性，尾随闭包默认使用块形式的函数体。

### 无括号形式的规则

像 `xs.iter().map |x| { .. }` 这样不带调用括号直接附加闭包，只有在以下条件下才是尾随闭包：

- 它的前面紧挨着一个路径（`thread::spawn`、`f`）或一个成员名（`.map`）。
- 它后面跟着 `|params|`、可选的 `-> R` 和 `{`。不是这种形状时，`|` 是按位或。
- `||` 始终是逻辑或。没有参数的闭包使用 `{ body }` 简写。
- 在 `if` / `while` / `match` / `for` / `using` 的头部（条件、被匹配的表达式、被迭代的表达式、`using` 初始化表达式）中，无括号的尾随闭包和 `{ body }` 简写都被禁用；那里的 `{` 开启主体块。如果头部需要闭包，请使用括号（`if xs.iter().any(|x| { x > 0 }) { .. }`）。

```nabi
let ys = xs.iter().map |x| -> i64 { x as i64 * 2 };   // explicit return type
let z = a | b;                                  // bitwise OR: not followed by the `|params| {` shape
if ok || retry { .. }                           // logical OR
```

## 4.8.2. 可自由选择显式括号

尾随闭包是允许的写法，而不是必需的写法。

```nabi
xs.iter().map(|x| x * 2);                // OK: explicit parentheses
xs.iter().map |x| { x * 2 };              // OK: trailing
```

团队和用户可以按照自己的风格自由选择。

## 4.8.3. 闭包不是最后一个实参时

尾随闭包只能填充最后一个实参的位置。如果闭包是中间的实参，必须使用显式括号。

```nabi
fn process(input: str, transform: fn(str) -> str, output: str) -> str { ... }

process("in", |s| { s.to_upper() }, "out");           // explicit parentheses (middle argument)
process("in", "out") |s| { s.to_upper() };            // ERROR (E0040): trailing only in the last position
```

## 4.8.4. 零实参调用

当调用没有其他实参时，空括号也可以省略。

```nabi
thread::spawn { heavy() };        // same as thread::spawn() { heavy() }
                                   // == thread::spawn(|| { heavy() })
```

当唯一的参数是闭包时（如 `thread::spawn`），这是最自然的调用形式。

在裸路径（而不是方法名）之后，具有结构体字面量形状的 `{ .. }` 就是结构体字面量（[3.10](../003-expressions/310-if-expression.md)）：`{}`、`{ x }`、`{ x, .. }`、`{ x: .. }` 和 `{ ..x }`。因此 `run { }` 和 `zero { n }` 是结构体字面量，当 `run` / `zero` 是函数时就是错误。这样的函数体要写出括号：`run(|| { })`、`zero(|| n)`。

## 4.8.5. 嵌套

尾随闭包可以自由嵌套。

```nabi
par::scope |s| {
    s.spawn { f() };
    s.spawn { g() };
};
```

同样的简写规则（`{ body }` ↔ `|| { body }`）也适用于内层闭包。

## 4.8.6. 消除解析歧义

尾随闭包带来一种歧义：有些代码既可以读作块表达式，也可以读作尾随闭包。以下规则消除这种歧义。

| 位置                                                                              | 含义                                                                 |
|-----------------------------------------------------------------------------------|----------------------------------------------------------------------|
| 控制关键字（`if`、`else`、`for`、`while`、`loop`、`match`）之后的 `{ ... }`        | 控制主体                                                             |
| 紧跟在函数调用之后的 `{ ... }`                                                    | 尾随闭包（0 个参数）                                                 |
| 紧跟在函数调用之后的 `\|params\| { ... }`                                         | 尾随闭包（1 个或更多参数）                                           |
| 普通表达式位置（如 `x = { ... };`）中的 `{ ... }`                                  | 块表达式（立即求值）；0 参数闭包需要显式写作 `\|\| { ... }`           |
| 表达式内部的 `\|...\| { ... }`                                                    | 普通闭包表达式                                                       |

简而言之：紧跟在函数调用（`(args)`）之后的 `{ ... }` 或 `|...| { ... }`，无论中间有没有空白，都是尾随闭包。

## 4.8.7. 单独的 `{}`：普通块与 0 参数闭包

```nabi
let x = { 42 };                   // block expression (value 42)
let f = { 42 };                   // ??
```

这两行形式相同，哪种含义优先？

规则：单独出现在表达式位置的 `{ body }` 是块表达式。

如果想要 0 参数闭包，请显式写作 `|| { body }`。

```nabi
let f = || { 42 };                // explicit 0-arg closure
let r = f();                       // call
```

也就是说，只有紧跟在函数调用之后的 `{ body }` 才被识别为尾随闭包。

## 4.8.8. 并行 `for` 等类语句位置中的闭包

对于并行循环、事务、上下文管理器等场景，尾随闭包可以产生读起来像关键字的、便于构建 DSL 的形式。

```nabi
par::for_each(xs) |x| { ... }     // looks like par for
with_lock(mutex) { ... }           // looks like a with-statement
retry(3) { ... }                   // looks like a retry keyword
```

这与[关键字](../001-lexical/105-keywords.md)中尽量减少保留关键字的原则一致。把库函数与尾随闭包结合，可以得到读起来像控制流的模式。

## 4.8.9. 与具名参数结合

```nabi
fn run(name: str = "task", body: fn() -> ()) { ... }

run("compute") { ... };           // positional + trailing OK
run(name: "compute") { ... };     // named + trailing OK
```

位置实参、具名参数和尾随闭包可以自由组合。`body` 没有默认值，却位于有默认值的 `name` 之后：默认值必须位于末尾，但 `fn` 类型的最后一个参数不受此限制，这样尾随闭包才能填充它（[4.2.3](402-parameters.md#423-default-values)）。处在该位置的其他类型的参数则需要默认值。

## 4.8.10. 小结

尾随闭包让规范层面无需额外关键字就能写出类似 DSL 的表达。关键在于，用户可以直接以库的形式构建看起来像语言关键字的模式。
