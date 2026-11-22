# 3.18. 管線運算式

管線 `|>` 把其左側的值傳給右側的函式，讓一連串自由函式的呼叫依執行順序閱讀。

```
pipe_expr       ::= expr "|>" pipe_target
pipe_target     ::= path                          // x |> f        is f(x)
                  | call_expr                     // x |> f(a, b)  is f(x, a, b)
                  | "(" closure_expr ")"          // x |> (|v| v + 1)
```

## 3.18.1. 意義

`|>` 是沒有 trait 的語言語法：編譯器在名稱解析之前改寫運算式，結果是一般的呼叫。

| 寫法                    | 意義                  |
|-------------------------|-----------------------|
| `x |> f`                | `f(x)`                |
| `x |> f(a, b)`          | `f(x, a, b)`          |
| `x |> p::q::f`          | `p::q::f(x)`          |
| `x |> f::<T>(a)`        | `f::<T>(x, a)`        |
| `x |> (|v| v + 1)`      | `(|v| v + 1)(x)`      |
| `x |> f |> g(a)`        | `g(f(x), a)`          |

左運算元成為呼叫的第一個引數，排在那裡寫出的引數之前；尾隨閉包仍在最後。右側的呼叫就是接收它的呼叫，因此 `x |> make(a)` 是 `make(x, a)`，而不是對 `make(a)` 回傳值的呼叫（請先繫結它：`let g = make(a); x |> g`）。方法呼叫也把它當作第一個引數接收（`x |> s.push_to(a)` 是 `s.push_to(x, a)`）；要在 `x` 上呼叫方法，照常使用 `.`：`x.trim()`。

```nabi
let words = text
    |> split_words
    |> drop_stop_words(stop)
    |> count_by_word;
```

## 3.18.2. 優先順序

`|>` 的結合比其他每個運算子（包括範圍）都鬆，比指派緊（[3.2.3](302-precedence-and-associativity.md#323-precedence-and-associativity-table)）；它是左結合的。因此其左側的整個計算就是被傳遞的值：

```nabi
a + b |> f          // f(a + b)
x > 0 && ok |> f    // f(x > 0 && ok)
0..n |> total       // total(0..n)
a |> f |> g         // g(f(a))
```

`try` 涵蓋其運算元之後的管線，而 `catch` 結合得更鬆，因此 `try s |> parse |> scale(2) catch _ { 0 }` 是 `(try scale(parse(s), 2)) catch _ { 0 }`（[3.14](314-error-control-flow.md)）。`try?` 與 `try!` 以相同方式涵蓋它們，直到 `??` 為止：`try! s |> parse` 是 `try! parse(s)`，`try? s |> parse ?? 0` 是 `(try? parse(s)) ?? 0`。

由於換行只是空白（[1.3](../001-lexical/103-statement-terminator.md)），位於行首的 `|>` 會延續其上方的運算式，如 3.18.1 的範例所示。

## 3.18.3. 右側可以是什麼

右側是指向函式（或任何可呼叫之物）的路徑、一個呼叫，或括號中的閉包。其他任何東西都是編譯錯誤，錯誤訊息會說明應寫什麼：`x |> 2` 與 `x |> a + b` 是錯誤，而由於 `|>` 結合最鬆，`x |> f + 1` 也是（請寫 `(x |> f) + 1`）。

`|>` 是固定語彙單元（[1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-fixed-token-table)）：它不能以 `#[infix]` 註冊，而 `a|>b` 一律是管線。
