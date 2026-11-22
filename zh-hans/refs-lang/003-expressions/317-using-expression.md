# 3.17. `using` 表达式：确定性地释放资源

`using` 关键字（[关键字](../001-lexical/105-keywords.md)）是一种把资源绑定到块的生命周期的表达式形式。无论块以何种方式结束（正常完成、提前 `return`、由 `?` 返回 `none`、错误传播），被绑定资源的 `Drop::drop` 总会被调用。

GC 会自动回收内存，但外部资源（文件、套接字、互斥锁、FFI 句柄、GC 固定等）必须确定性地释放。`Drop` 值是仿射的：编译器在其所有者离开作用域之处释放它（[标准库 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-release)）。`using` 让这一点变得显式：它把值绑定到它的块，在块结束时释放它；对于关闭操作可能失败的值，它也是无需写 `try x.close()` 就能关闭该值的方式。

## 3.17.1. 形式

```
using_expr     ::= "using" using_binding ("," using_binding)* block
using_stmt     ::= "using" using_binding ("," using_binding)* ";"
using_binding  ::= pattern "=" expr
```

`using_stmt` 是 [3.17.9](#3179-statement-form) 中的语句形式。

各组成部分的含义：

- `pattern` 可以是[模式](311-patterns.md)中定义的任何不可反驳的模式（标识符、元组、结构体解构等）。大多数情况下使用单个标识符。被释放的是 `expr` 的整个值。
- `expr` 是创建或获取资源的表达式。其结果类型必须实现 [`Drop`](../../refs-standard-libraries/200-core/213-marker.md) trait。
- 如果获取资源可能失败，用 `try` 标记 `expr`（`using f = try fs::open(path) { ... }`）。如果获取时产生错误，控制在未绑定任何资源的情况下离开，因此不会发生 `Drop` 调用。详细语义遵循[错误控制流](314-error-control-flow.md)。
- 用 `,` 连接的多个绑定构成多重资源；详细行为在 [3.17.4 多重资源](#3174-multiple-resources)中定义。
- `block` 遵循[块表达式](309-block-expressions.md)的规则，其最后一个表达式的值是整个 `using` 表达式的值。

## 3.17.2. 最简单的示例

```nabi
using f = try fs::open(path) {
    try f.read_to_end()               // the last expression is the value of using
}                                  // ← f.drop() is called automatically here
```

`fs::open(path)` 的返回类型是 `File`，且 `File: Drop`，因此它可以作为 `using` 的目标。块结束时调用 `File::drop`，关闭操作系统句柄。

## 3.17.3. 与 `try` 结合

当获取资源本身可能失败时，用 `try` 标记获取表达式。

```nabi
fn read_text(path: str) throws (IoError) -> str {
    using f = try fs::open(path) {     // on failure, IoError propagates
        try f.read_to_str()            // try can propagate inside the body too
    }                                  // if f was bound, it is dropped on both success and error
}
```

这种形式在语义上等价于以下展开后的代码。

```nabi
let f = try fs::open(path);
try {
    f.read_to_str()
} finally {
    f.drop();          // a call inserted by the compiler; writing it directly is an error
}
```

即使错误在主体内传播，`drop` 也会在对应于 `finally` 的位置被调用。

## 3.17.4. 多重资源

要把多个资源绑定到一个块，可以嵌套 `using`，或者用逗号连接绑定。本规范允许两种形式，且两者含义等价。

```nabi
using a = AcquireA() {
    using b = AcquireB(a) {
        body
    }
}

// or the shorthand form
using a = AcquireA(), b = AcquireB(a) {
    body
}
```

两种形式都遵循**逆序释放**：先 drop `b`，再 drop `a`。这与 Java 的 try-with-resources 规则相同。逆序释放能安全地解开依赖关系（`b` 引用 `a`）。

## 3.17.5. 作为表达式的 `using`

`using` 是表达式而不是语句，它的值是主体块的值。

```nabi
let content = using f = try fs::open(path) {
    try f.read_to_end()
};
```

这是与函数作用域的 `defer`（Go）在结构上最大的区别。`using` 表达式遵循[块表达式](309-block-expressions.md)的相关规则，可以用在任何表达式位置。

## 3.17.6. drop 被调用的时机

`using` 保证在以下全部五条路径上 drop 被绑定的资源：

- 块正常完成（最后一个表达式已求值）
- 主体内的 `return`
- 主体内的 `break` / `continue`
- 主体内由 `?` 返回 `none`
- 主体内的 `throw`，以及来自用 `try` 标记的调用的错误传播

panic 和致命错误会立即终止进程，因此不会调用 `drop`（[3.14.8](314-error-control-flow.md#3148-panics-and-fatal-errors)）。这是 panic 的一般结果，而不是 `using` 特有的行为。

一个值只释放一次（[标准库 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-release)）。`using x = a { .. }` 把 `a` 移入 `x`，因此第二个 `using y = a { .. }`，或在第一个块之后使用 `a`，都是编译错误（E0133）：应让每个 `using` 接收它自己创建的值（`using g = m.lock() { .. }`），或者在一个 `using` 块内完成全部工作。在块内已关闭的值（`try x.close()`）不会再次释放。

## 3.17.7. 可以绑定的类型

`using` 右侧表达式的类型必须实现 [`Drop`](../../refs-standard-libraries/200-core/213-marker.md)，或者包含具有 `Drop` 的字段（drop glue，[标准库 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-release)）。否则是编译错误（E0066）。

```nabi
using x = 42 { ... }   // ERROR (E0066): using requires a type implementing Drop; i64 does not implement Drop
```

这一限制确保只有有意义的资源才用 `using` 绑定，在规范层面排除了对普通值使用 `using`。

## 3.17.8. 与其他语言的比较

| 语言     | 形式                          | 作用域单位   | 备注                                    |
|----------|-------------------------------|--------------|-----------------------------------------|
| Nabi      | `using x = try e { ... }`、`using x = e;` | 块 | 表达式，多重资源，与 `try` 集成 |
| C#       | `using (var x = e) { ... }`   | 块           | 需要 `IDisposable`                      |
| Java     | `try (var x = e) { ... }`     | try 块       | 需要 `AutoCloseable`，多重资源          |
| Python   | `with x as f: ...`            | 块           | `__enter__` / `__exit__`                |
| Swift    | `defer { x.close() }`         | 函数         | 必须显式书写                            |
| Go       | `defer x.Close()`             | **函数**     | 不是块作用域                            |
| Zig      | `defer x.close();`            | 块           | 有 `errdefer` 变体                      |
| Rust     | （自动 RAII，无关键字）       | 块           | 无 GC；drop 点自动确定                  |

Nabi 的 `using` 把 GC 语言（C#、Java、Python）的显式块模式，与作为表达式的可组合性、与 `try` 的集成以及多重资源的逆序释放结合在一起。

## 3.17.9. 语句形式

`using x = e;` 把 `x` 绑定到外层块的剩余部分，并在该块结束时释放它：

```nabi
fn copy_lines(from: Path) throws (IoError) {
    using r = BufReader::new(try fs::open(from));
    try for line in r.lines() {
        println(f"> {line}");
    }
}                                  // ← r is released here
```

它的含义是覆盖同一块中其后各语句的块形式：

```nabi
using a = e;
s1;
s2
// is
using a = e { s1; s2 }
```

因此块形式的每条规则都成立：在 3.17.6 的所有路径上释放；多个绑定以及同一块中的多条 `using` 语句按逆序释放；3.17.7 的 `Drop` 要求；以及对被绑定值的视图的检查。块的值是 `using` 之后各语句的值。在 `main.nabi` 的顶层语句中，块的剩余部分就是程序的剩余部分。

语句形式必须开始一条语句；在其他位置（`let x = using r = e;`）是编译错误，其帮助信息会指出块形式。由于其后没有主体，结构体字面量可以不加圆括号地结束其绑定（`using r = R { n: 1 };`）。块形式保留用于比块的剩余部分更短的作用域，以及作为表达式的 `using`。
