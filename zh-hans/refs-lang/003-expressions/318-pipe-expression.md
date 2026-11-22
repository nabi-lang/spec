# 3.18. 管道表达式

管道 `|>` 把其左侧的值传给其右侧的函数，因此一连串自由函数调用可以按其执行顺序阅读。

```
pipe_expr       ::= expr "|>" pipe_target
pipe_target     ::= path                          // x |> f        is f(x)
                  | call_expr                     // x |> f(a, b)  is f(x, a, b)
                  | "(" closure_expr ")"          // x |> (|v| v + 1)
```

## 3.18.1. 含义

`|>` 是不涉及 trait 的语言语法：编译器在名字解析之前改写表达式，结果是一个普通调用。

| 写法                    | 含义                  |
|-------------------------|-----------------------|
| `x |> f`                | `f(x)`                |
| `x |> f(a, b)`          | `f(x, a, b)`          |
| `x |> p::q::f`          | `p::q::f(x)`          |
| `x |> f::<T>(a)`        | `f::<T>(x, a)`        |
| `x |> (|v| v + 1)`      | `(|v| v + 1)(x)`      |
| `x |> f |> g(a)`        | `g(f(x), a)`          |

左操作数成为调用的第一个实参，位于那里所写的实参之前；尾随闭包仍然是最后一个。右侧的调用就是接收它的调用，因此 `x |> make(a)` 是 `make(x, a)`，而不是对 `make(a)` 返回值的调用（应先绑定：`let g = make(a); x |> g`）。方法调用同样把它作为第一个实参接收（`x |> s.push_to(a)` 是 `s.push_to(x, a)`）；要在 `x` 上调用方法，照常使用 `.`：`x.trim()`。

```nabi
let words = text
    |> split_words
    |> drop_stop_words(stop)
    |> count_by_word;
```

## 3.18.2. 优先级

`|>` 的结合比其他所有运算符（包括区间）都松，比赋值紧（[3.2.3](302-precedence-and-associativity.md#323-precedence-and-associativity-table)）；它是左结合的。因此其左侧的整个计算就是被传递的值：

```nabi
a + b |> f          // f(a + b)
x > 0 && ok |> f    // f(x > 0 && ok)
0..n |> total       // total(0..n)
a |> f |> g         // g(f(a))
```

`try` 覆盖其操作数之后的管道，而 `catch` 的结合更松，因此 `try s |> parse |> scale(2) catch _ { 0 }` 是 `(try scale(parse(s), 2)) catch _ { 0 }`（[3.14](314-error-control-flow.md)）。`try?` 和 `try!` 以同样方式覆盖它们，直到 `??` 为止：`try! s |> parse` 是 `try! parse(s)`，`try? s |> parse ?? 0` 是 `(try? parse(s)) ?? 0`。

由于换行只是空白（[1.3](../001-lexical/103-statement-terminator.md)），以 `|>` 开始的一行会延续上一行的表达式，如 3.18.1 中的示例所示。

## 3.18.3. 右侧可以是什么

右侧是指向函数（或任何可调用之物）的路径、一个调用，或者括在圆括号中的闭包。其他任何东西都是编译错误，错误信息会说明应该写什么：`x |> 2` 和 `x |> a + b` 是错误；由于 `|>` 结合最松，`x |> f + 1` 也是错误（应写 `(x |> f) + 1`）。

`|>` 是固定词法单元（[1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-fixed-token-table)）：它不能用 `#[infix]` 注册，`a|>b` 总是管道。
