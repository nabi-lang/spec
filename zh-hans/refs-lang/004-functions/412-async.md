# 4.12. 异步（`async` 与 `await`）

`async` 和 `await` 关键字（[关键字](../001-lexical/105-keywords.md)）是定义异步计算（即承诺未来某个值的函数）并等待其结果的机制。

## 4.12.1. 三种定义形式

异步定义有三种语法形式。

```
async_fn        ::= "async" fn_decl
async_block     ::= "async" block
async_closure   ::= "async" closure_expr
await_expr      ::= "await" unary_expr
```

```nabi
async fn name(args) -> R { ... await expr; ... }    // named async function
async { ... await expr; ... }                        // inline block (value)
async || { ... await expr; ... }                     // closure modifier
```

在这三种形式中，结果都是库类型 `AsyncFuture<T>`（实现 `Future<Output = T>`）的值；带 `throws (E)` 时是 `TryAsyncFuture<T, E>`（[4.1.11](401-function-declaration.md#result-types-of-coroutine-functions)）。这一结构与[生成器](411-generators.md)的 `yield` 以同样三种形式使用的模式一致。

## 4.12.2. `await` 运算符

`await` 是前缀一元运算符（[3.2.3](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table)，prec=13a），作用于实现 `Future<Output = T>` 的值；其结果是 `T`。`await f() == x` 即 `(await f()) == x`。

```nabi
await expr
```

它挂起直到 Future 完成，然后返回结果值。`await` 本身是表达式，结果值就是该运算符的求值结果。

```nabi
let data = await fetch_data();
let processed = await transform(data);
```

`await` 只允许出现在 `async` 函数、`async` 块和 `async` 闭包的主体中。在普通函数体中使用 `await` 是编译错误。

## 4.12.3. 具名 `async` 函数

异步函数用 `async fn` 声明。

```nabi
async fn fetch_user(id: i64) -> User {
    let response = await http_get(f"/users/{id}");
    await parse_user(response)
}
```

在函数体内，`await` 等待其他异步结果；`-> R` 的位置给出完成时产生的值的类型。调用结果是 `AsyncFuture<R>`。`-> R` 可以省略；此时 `R` 遵循 [4.1.3](401-function-declaration.md#413-omitting-the-return-type)：从函数体推断，或者在 `pub` 或 `pub(pkg)` 函数上为 `()`。

```nabi
let f = fetch_user(42i64);        // f: `AsyncFuture<User>`
let u = await f;                   // waits for the result
```

调用本身立即返回；此时创建的只是一个 **Future** 对象。实际工作在 `await` 时开始，调度器也可以更早开始它。

## 4.12.4. 内联 `async` 块：值

`async` 块在表达式位置立即创建一个 `AsyncFuture<T>`。

```nabi
let f = async {
    let a = await op1();
    let b = await op2();
    a + b
};

let r = await f;
```

主体是一个[块表达式](../003-expressions/309-block-expressions.md)。这种形式用于以匿名、内联方式编写简短的异步工作。

与闭包一样，`async` 块**在创建时复制**外层值来捕获它们（[4.6](406-capture-semantics.md)）。块创建之后修改外层变量，不会改变 future 内部的值；future 内部的重新赋值在外部也不可见。

```nabi
let n = 1;
let f = async { n * 10 };
n = 2;
await f;                       //→ 10 (n at creation time)
```

## 4.12.5. `async` 闭包

`async ||` 创建一个可调用的异步闭包。

```nabi
let task = async || {
    let data = await fetch();
    process(data)
};

let f1 = task();                   // f1: `AsyncFuture<...>`
let f2 = task();                   // a new Future each time
```

每次调用都创建一个新的 **Future**。使用 `[ref x]` 捕获列表时（`async [ref x] || { .. }`），`x` 的共享单元由外层作用域和该闭包创建的每一个 future 共同使用。

```nabi
let hits = 0;
let bump = async [ref hits] || { hits += 1; };
await bump();
await bump();
hits;                          //→ 2
```

## 4.12.6. 三种形式的对应关系

三种形式的对应关系如下。

```
plain        fn f() { ... }       ↔  || { ... }       ↔  { ... }
yield form   yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
async form   async fn f() { ... } ↔  async || { ... } ↔  async { ... }
```

修饰符 `yield` 和 `async` 附加在同样的函数、块和闭包形式上，一致地把它们变成生成器或 **Future**。

## 4.12.7. Future trait

async 函数、块或闭包的结果是标准库的 `AsyncFuture<T>`（实现 `Future<Output = T>`）。确切的 trait 定义以及 `then`、`map`、`join` 等方法在标准库中定义。

## 4.12.8. `await` 的语义

下面的示例展示函数体如何求值。

```nabi
async fn f() -> i32 {
    let a = await op1();
    let b = await op2();
    a + b
}
```

首先调用 `op1()`，它返回一个 **Future**。然后 `await` 等待该 Future 完成；在完成之前，当前函数 `f` 也被挂起。Future 完成时，其结果绑定到 `a`，求值恢复。`op2()` 和第二个 `await` 遵循同样的模式，所有 `await` 结束后，求值 `a + b` 并作为函数结果返回。

这种挂起与恢复运行在与[生成器（`yield`）](411-generators.md)相同的协程机制之上。

## 4.12.9. 并发执行：库函数

要并发推进多个 Future，使用标准库函数。

```nabi
let rs = await task::join([task::spawn(op1()), task::spawn(op2())]);   // all: List<T>
let first = await task::race([task::spawn(op1()), task::spawn(op2())]); // whichever finishes first
let r = try await task::timeout(d, task::spawn(op1()));                // TimeoutError on timeout
```

`join` / `race` 接收 `List<Task<T>>`，其结果要被 `await`（[标准库 2.18](../../refs-standard-libraries/200-core/218-task.md)）。

按照[语言与标准库的边界](../000-meta/004-language-vs-standard-libraries.md)原则，`join`、`race`、`timeout`、`select` 等是库函数，而不是语言关键字。它们的确切签名在标准库中定义。

## 4.12.10. 允许 `await` 的位置

允许 `await` 的位置如下：

| 位置                         | 是否允许 `await`                     |
|------------------------------|--------------------------------------|
| `async fn` 函数体            | OK                                   |
| `async { ... }` 主体         | OK                                   |
| `async \|...\| { ... }` 主体 | OK                                   |
| 普通 `fn` 函数体             | ERROR                                |
| 普通 `{ ... }` 块            | ERROR                                |
| `yield` 函数或块的主体       | ERROR（但 `async yield` 可以）       |

## 4.12.11. async 函数调用立即返回

由于 async 函数调用立即返回，可以采用以下模式。

```nabi
let f = fetch_data();             // returns immediately; work has started or is scheduled
do_other_work();                   // do other work in parallel
let data = await f;                // wait when the result is needed
```

这是高效并发的基本模式：耗时的工作在需要其结果时才等待，而不是在调用之后立刻等待。

## 4.12.12. 与任务的关系

要把异步工作作为独立的执行单元启动，使用标准库的 `task::spawn`。

```nabi
let handle = task::spawn(async { compute() });       // runs as a new task
let result = await handle;                     // waits for completion
```

`task::spawn` 把一个 future（这里是一个 `async` 块）注册到调度器，并返回一个实现 `Future` 的 `Task<T>` 句柄。

`task::spawn` 是库函数，与 `async` 正交。把函数标记为 `async` 只是声明“这个函数是异步的”；要真正把它作为独立的执行单元启动，必须显式调用 `task::spawn` 之类的函数。

## 4.12.13. 方法上的 `async`

与普通[方法](409-method.md)一样，方法也可以标记为 `async`。

```nabi
impl Database {
    async fn query(self, sql: str) -> ResultSet { ... }
}

let r = await db.query("SELECT ...");
```

`async` 方法可以接收 `ref self`（[4.7.5](407-ref-keyword.md#475-exception-for-method-receivers)）。它的 future 操作接收者的值，并在完成时把该值写回接收者所在的位置：无论是带着值完成、带着错误完成，还是被取消。因此值类型的接收者能看到方法所做的修改，与同步的 `ref self` 方法相同。

```nabi
struct Reader { pos: usize; data: List<u8>; }

impl Reader {
    async fn read_one(ref self) -> u8 | none {
        if self.pos >= self.data.len() { return none; }
        self.pos += 1;
        self.data[self.pos - 1]
    }
}

let r = Reader { pos: 0, data: [1u8, 2, 3] };
while await r.read_one() is u8 {}
r.pos;                              //→ 3
```

future 在完成之前一直持有接收者，因此调用必须在发起处被等待：`await r.read_one()`，对可抛出的方法则是 `try await r.read(buf)`。把 future 保留下来以后再等待是编译错误（`E0090`），因为没有任何东西会把接收者写回，而且接收者在此期间可能发生变化：

```nabi
let f = r.read_one();               // ERROR: its future holds `r` until it completes
```

临时接收者，或通过 `any` 访问的接收者（其值是副本），会丢弃写回，与同步 `ref self` 调用相同。其他 `ref` 参数在所有协程函数上仍然是错误（[4.7.7](407-ref-keyword.md#477-references-cannot-be-first-class-values)）。

## 4.12.14. 错误处理

[错误控制流](../003-expressions/314-error-control-flow.md)中的 `try`、`catch`、`throw` 和 `finally` 在 async 函数中照常工作。可抛出的 `async` 函数的错误在 `await` 处交付，因此 `await` 表达式要用 `try` 覆盖（`try await fetch()`）。与 `throws` 签名的交互在 [`throws` 签名](414-throws.md)中介绍。

```nabi
async fn safe_fetch() throws -> Data {
    try await fetch_or_throw() catch e: NetworkError {
        try await fetch_fallback()
    }
}
```
