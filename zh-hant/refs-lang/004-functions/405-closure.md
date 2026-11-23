# 4.5. 閉包

閉包是出現在運算式位置、沒有名稱的函式值。它可以擷取外部環境的變數，因此便於撰寫依賴周圍狀態的小函式。

## 4.5.1. 形式

```
closure_expr   ::= capture_list? closure_params (return_type block | closure_body)
closure_params ::= "||"
                 | "|" closure_param_list "|"
closure_param_list ::= closure_param ("," closure_param)* ","?
closure_param ::= (identifier | param_pattern) (":" type_expr)?
closure_body   ::= expr | block
capture_list   ::= "[" capture_item ("," capture_item)* ","? "]"   // capture_item: 4.6.1
```

閉包參數可以和函式參數一樣是不可反駁的模式（[4.2](402-parameters.md)）：`|(k, v)| f"{k}={v}"` 接受一個元組引數並解構它，而 `|k, v|` 接受兩個引數。

零參數形式 `||` 是單一語彙單元（見[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)），不會被拆成兩個 `|`。

```nabi
|params| body
```

參數以左右兩個 `|` 語彙單元括起來（[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)）。主體是運算式或區塊。

### 單一運算式主體

```nabi
|x| x + 1
|x: i32| x * 2
|a, b| a + b
```

`|...|` 之後直接接單一運算式，其值就是閉包的回傳值。

### 區塊主體

```nabi
|x: i32| -> i32 {
    let y = x * 2;
    y + 1
}
```

使用區塊主體時，可以用 `->` 給出回傳型別（省略時由推論決定）。

### 零參數形式

```nabi
|| body
|| { body }
```

沒有參數時，使用 `||`（兩個連續的管線符號）。在切分上，`||` 在[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)的固定語彙單元表中定義為一個語彙單元。

## 4.5.2. 參數型別

參數型別可以寫出或省略。

```nabi
|x: i32| x * 2                    // explicit
|x| x * 2                          // inferred
```

推論由閉包被使用的上下文決定，例如它被傳入之引數位置的函式型別，或變數宣告的型別。沒有上下文時，由之後的呼叫決定參數型別（`let f = |x| x * 2; f(4);` 中的 `x: i32`，[2.10.2](../002-types/210-generics.md#型別引數推論)）。

```nabi
let f: fn(i32) -> i32 = |x| x * 2;   // x: i32 inferred from context
list.iter().map(|x| x * 2);                  // inferred from map's signature
```

## 4.5.3. 推論或明確的回傳型別

```nabi
|x| x * 2                          // inferred
|x: i32| -> i32 { x * 2 }          // explicit: only with a block body
```

明確的回傳型別（`-> type`）只與區塊主體一起使用。單一運算式主體一律依賴推論。

## 4.5.4. 擷取

閉包可以擷取外層作用域的變數。

```nabi
let n = 10;
let f = |x| x + n;                // captures n
f(5);                              // 15
```

擷取語意（預設依值擷取，以及 `[ref x]` 之類的擷取清單）詳細定義於[擷取語意](406-capture-semantics.md)。

## 4.5.5. 閉包與函式

| 面向                     | `fn` 具名函式       | `\| ... \|` 閉包        |
|--------------------------|---------------------|-------------------------|
| 名稱                     | 有                  | 無                      |
| 定義位置                 | 模組或 `impl`       | 任何運算式位置          |
| 擷取外部變數             | 否                  | 是                      |
| 呼叫                     | `name(...)`         | `value(...)`            |
| 型別                     | `fn(...) -> ...`    | 相同                    |

兩者以相同的函式型別表達，介面相容，因此任一者都適合同一個位置。

## 4.5.6. 使用閉包值

```nabi
// assigned to a variable
let f = |x| x * 2;

// function argument
list.iter().map(|x| x * 2);

// function return
fn make_adder(n: i32) -> fn(i32) -> i32 {
    |x| x + n
}

// collection
let handlers: List<fn(i32) -> i32> = [|x| x + 1, |x| x - 1, |x| x * 2];
```

## 4.5.7. 呼叫

```nabi
let f = |x| x * 2;
let r = f(5);                     // same form as an ordinary function call
```

閉包以 `f(args)` 呼叫。呼叫時，擷取的環境與引數一起用來對主體求值。

## 4.5.8. 立即呼叫：IIFE

閉包可以在定義之後立即呼叫。

```nabi
let r = (|x: i32| -> i32 { x * 2 })(5);     // 10
```

這並不常見，但它是表達具有立即求值之區域作用域的一種方式。大多數情況下，[區塊運算式](../003-expressions/309-block-expressions.md)的一般區塊更自然。

## 4.5.9. 可讀性：單一運算式與區塊

```nabi
list.iter().map(|x| x * 2)               // single expression: short and clear
list.iter().map(|x| { let y = x * 2; y + 1 })   // block: multi-step processing
```

請選擇適合主體長度的形式。規格完整支援兩種形式。

## 4.5.10. 尾隨閉包

當函式呼叫的最後一個引數是閉包時，為了可讀性可以把它移到括號之外。詳細內容定義於[尾隨閉包](408-trailing-closure.md)。

## 4.5.11. 特殊變體：`async` 與 `yield` 修飾詞

為閉包加上 `async` 或 `yield` 修飾詞，會建立非同步閉包或產生器閉包。

```nabi
async || { ... }                  // closure returning a Future
yield || { ... }                  // closure returning an Iterator
async yield || { ... }            // closure returning a Stream
```

詳細規則定義於[產生器](411-generators.md)、[非同步](412-async.md)與[串流](413-stream.md)。

## 4.5.12. 零引數閉包的簡寫：`{ body }`

在某些上下文中，`{ body }` 被辨識為零參數閉包的簡寫。這種簡寫常與尾隨閉包一起使用。

```nabi
thread::spawn { heavy_work() }    // == thread::spawn(|| { heavy_work() })
```

這種簡寫的確切規則定義於[尾隨閉包](408-trailing-closure.md)。
