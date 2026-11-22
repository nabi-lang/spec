# 4.11. 產生器（`yield`）

`yield` 關鍵字（[關鍵字](../001-lexical/105-keywords.md)）以三種形式使用，用來定義產生器，即一次產生一個值的函式。

## 4.11.1. 三種形式

產生器的文法如下：

```
generator_fn    ::= "yield" fn_decl
generator_block ::= "yield" block
generator_closure ::= "yield" closure_expr
yield_stmt      ::= "yield" expr? ";"
```

```nabi
yield fn name(args) -> R { ... yield expr; ... }     // named generator function
yield { ... yield expr; ... }                         // inline block (value)
yield || { ... yield expr; ... }                      // closure modifier
```

在三種形式中，結果都是函式庫型別 `GenIter<T>`（實作 `Iterator<Item = T>`）的值，帶有 `throws (E)` 時則是 `TryGenIter<T, E>`。這條規則定義於 [4.1.11](401-function-declaration.md#協程函式的結果型別)。

## 4.11.2. `yield` 陳述式：送出值

`yield` 陳述式可以帶值或不帶值使用。

```nabi
yield expr;
yield;
```

`yield expr;` 把 `expr` 的值作為目前迭代步驟的輸出送出，並暫停函式或區塊的求值。求值在下一次迭代時繼續。`yield;` 不帶值（unit）地推進一步。

## 4.11.3. 具名產生器函式

具名產生器函式的寫法如下。

```nabi
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

`yield fn` 是標示為產生器的函式宣告；主體以 `yield expr;` 送出值。`-> R` 位置給出產出值的型別，呼叫的結果是 `GenIter<R>`。`-> R` 必須寫出；元素型別不會從主體推論。它像一般函式一樣被呼叫，結果是一個迭代器物件。

```nabi
let it = count_up_to(5i32);       // it: `iter::GenIter<i32>`
for x in it { print(f"{x} "); }   // 0 1 2 3 4
```

## 4.11.4. 行內 `yield` 區塊：值

要在運算式位置立即建立迭代器，請使用行內 `yield` 區塊。

```nabi
let it = yield {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};
```

這種形式出現在運算式位置，立即產生一個 `GenIter<T>`。主體是[區塊運算式](../003-expressions/309-block-expressions.md)的區塊。它是以匿名、行內方式撰寫簡短產生器的形式。

## 4.11.5. `yield` 閉包

`yield` 閉包產生一個可呼叫的產生器。

```nabi
let make_iter = yield || {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};

let it = make_iter();             // each call creates a new Iterator
```

`yield ||` 建立一個可呼叫的產生器閉包；每次呼叫都建立一個新的迭代器。對外部的擷取遵循[擷取語意](406-capture-semantics.md)的規則。

## 4.11.6. 三種形式的對應

三種產生器形式一對一地對應於[函式宣告](401-function-declaration.md)、[閉包](405-closure.md)與[區塊運算式](../003-expressions/309-block-expressions.md)的三種形式（具名、區塊、閉包）。

```
plain        fn f() { ... }       ↔  || { ... }       ↔  { ... } (block)
yield form   yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
```

也就是說，`yield` 作為修飾詞附加在相同的函式、區塊與閉包形式上，把它們變成產生器。

## 4.11.7. `yield` 語彙單元的四個位置

`yield` 語彙單元可以出現在四個位置。

```nabi
yield fn name(...) -> T { ... }   // declaration modifier
yield { ... }                      // block marker
yield || { ... }                   // closure modifier
yield expr;                         // statement
```

這四個位置由上下文清楚區分；剖析器依下一個語彙單元決定形式。

| `yield` 之後的語彙單元 | 意義                                                                       |
|---------------------|----------------------------------------------------------------------------|
| `fn`                | 宣告修飾詞                                                                 |
| `{`                 | [區塊](../000-meta/007-glossary.md#區塊block)標記（運算式位置）              |
| `\|\|` 或 `\|`      | 閉包修飾詞                                                                 |
| 運算式開頭的語彙單元 | `yield` 陳述式                                                            |

## 4.11.8. 與 Iterator trait 的互動

產生器函式的結果是實作了標準函式庫 `Iterator<Item = T>` trait 的值，因此可以直接用在[迴圈](../003-expressions/313-loop-expressions.md)的 `for` 迴圈中。

```nabi
yield fn naturals() -> i32 {
    let i = 0;
    loop {
        yield i;
        i = i + 1;
    }
}

for n in naturals() {
    if n > 100 { break; }
    print(n);
}
```

`Iterator` trait 的確切定義與方法（`map`、`filter`、`take`、`collect` 等）定義於標準函式庫。

## 4.11.9. 無限產生器

上面的 `naturals()` 是無止盡地產生值的無限產生器。這樣的產生器可以用 `for` 迴圈中的 `break`、函式庫的 `take(n)` 方法等方式結束。

## 4.11.10. 產生器的結束

當主體正常完成時，迭代器也隨之結束，下一次 `next()` 呼叫回傳 `none`。

```nabi
yield fn finite() -> i32 {
    yield 1i32;
    yield 2i32;
    yield 3i32;
    // function completes normally → iterator terminates
}
```

也可以用 `return;` 明確地結束它。

```nabi
yield fn early_stop(stop: i32) -> i32 {
    let i = 0;
    loop {
        if i >= stop { return; }
        yield i;
        i = i + 1;
    }
}
```

被 `for` 迴圈提前離開的產生器會被關閉。這發生在迴圈主體以 `break`（也包括帶標籤的 `break` 或外層迴圈的 `continue`）、`return`、擲回的錯誤、`try` 傳播、任務的取消，或外圍產生器的 `yield` 離開迴圈之時。關閉會讓產生器從它所等待的 `yield` 處繼續，然後像 `return` 一樣離開主體：包圍該 `yield` 的 `finally` 區塊與 `using` 的 drop 會從最內層開始執行。關閉期間到達的 `yield` 以相同方式結束產生器；其值被捨棄。在第一次 `next()` 之前就被關閉的產生器不會執行其主體的任何部分。關閉之後，`next()` 回傳 `none`。

```nabi
yield fn lines() -> str {
    using f = open_file() {          // dropped when the loop below breaks
        for l in f.lines() { yield l; }
    }
}

for l in lines() {
    if l == "" { break; }            // closes the generator: `f` is dropped
}
```

只有迴圈本身持有的產生器才會被關閉：`for` 運算式就是產生器值（呼叫、`yield` 區塊）。存放在變數、欄位或元素中的產生器不會被關閉，因此迭代可以在迴圈之後繼續：

```nabi
let g = lines();
for header in g { if header == "" { break; } }   // g stays open
for body in g { print(body); }                    // goes on after the blank line
```

從未被關閉、也從未結束的產生器，其清理會一直擱置，直到 GC 回收它為止；那時清理也不會執行。

## 4.11.11. 產生器的狀態

產生器在呼叫之間保有內部狀態。主體的區域變數、迭代位置與協程框架會保存到下一次 `next()` 呼叫。

這個狀態運作在 [7. 執行環境模型](../007-runtime/README.md)的協程基礎設施之上，並作為編譯器產生的（無堆疊）狀態機運作。詳細的實作機制定義於執行環境規格。

## 4.11.12. `yield` 函式的參數與回傳

由於 `yield fn` 是產生器，`-> T` 表示主體中產出值的型別，而呼叫的結果是 `GenIter<T>`。

```nabi
yield fn f() -> i32 { yield 1i32; }
// call result: `GenIter<i32>`
```

參數的使用方式與一般函式相同。

## 4.11.13. 作為方法的產生器

`yield fn` 也可以用在[方法](409-method.md)的 `impl` 區塊內，定義產生器方法。

```nabi
impl<T> List<T> {
    yield fn iter(self) -> T {
        for x in self.elements { yield x; }
    }
}
```

呼叫與行為和一般的實例方法相同。
