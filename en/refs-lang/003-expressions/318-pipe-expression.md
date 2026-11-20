# 3.18. Pipe expression

The pipe `|>` passes the value on its left to the function on its right, so a sequence of free-function calls reads in the order it runs.

```
pipe_expr       ::= expr "|>" pipe_target
pipe_target     ::= path                          // x |> f        is f(x)
                  | call_expr                     // x |> f(a, b)  is f(x, a, b)
                  | "(" closure_expr ")"          // x |> (|v| v + 1)
```

## 3.18.1. Meaning

`|>` is language syntax with no trait: the compiler rewrites the expression before name resolution, and the result is an ordinary call.

| Written                 | Meaning               |
|-------------------------|-----------------------|
| `x |> f`                | `f(x)`                |
| `x |> f(a, b)`          | `f(x, a, b)`          |
| `x |> p::q::f`          | `p::q::f(x)`          |
| `x |> f::<T>(a)`        | `f::<T>(x, a)`        |
| `x |> (|v| v + 1)`      | `(|v| v + 1)(x)`      |
| `x |> f |> g(a)`        | `g(f(x), a)`          |

The left operand becomes the first argument of the call, before the arguments written there; a trailing closure stays the last. A call on the right is the call that receives it, so `x |> make(a)` is `make(x, a)`, not a call of what `make(a)` returns (bind that first: `let g = make(a); x |> g`). A method call receives it as its first argument too (`x |> s.push_to(a)` is `s.push_to(x, a)`); to call a method on `x`, use `.` as usual: `x.trim()`.

```nabi
let words = text
    |> split_words
    |> drop_stop_words(stop)
    |> count_by_word;
```

## 3.18.2. Precedence

`|>` binds looser than every other operator, ranges included, and tighter than assignment ([3.2.3](302-precedence-and-associativity.md#323-precedence-and-associativity-table)); it is left-associative. So the whole computation on its left is the value passed on:

```nabi
a + b |> f          // f(a + b)
x > 0 && ok |> f    // f(x > 0 && ok)
0..n |> total       // total(0..n)
a |> f |> g         // g(f(a))
```

A `try` covers the pipes after its operand, and `catch` binds looser still, so `try s |> parse |> scale(2) catch _ { 0 }` is `(try scale(parse(s), 2)) catch _ { 0 }` ([3.14](314-error-control-flow.md)). `try?` and `try!` cover them the same way, up to a `??`: `try! s |> parse` is `try! parse(s)`, and `try? s |> parse ?? 0` is `(try? parse(s)) ?? 0`.

Since a line break is only whitespace ([1.3](../001-lexical/103-statement-terminator.md)), a `|>` that starts a line continues the expression above it, as in the example of 3.18.1.

## 3.18.3. What the right side may be

The right side is a path to a function (or to anything callable), a call, or a closure in parentheses. Anything else is a compile error that says what to write: `x |> 2` and `x |> a + b` are errors, and because `|>` binds loosest, `x |> f + 1` is one too (write `(x |> f) + 1`).

`|>` is a fixed token ([1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-fixed-token-table)): it cannot be registered with `#[infix]`, and `a|>b` is always the pipe.
