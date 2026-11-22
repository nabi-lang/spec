# 4.8. 尾隨閉包

當函式或方法呼叫的最後一個引數是閉包時，該閉包可以移到括號之外。這種形式稱為尾隨閉包。

## 4.8.1. 形式

尾隨閉包的文法如下：

```
trailing_call   ::= primary "(" arg_list? ")" trailing_closure
                  | primary trailing_closure                    // 0-arg call
trailing_closure ::= ("|" closure_params? "|" ("->" type)?)? block
```

```nabi
function(args) |params| { body }
function(args) { body }            // zero-parameter shorthand
```

閉包直接接在呼叫的括號之後，主體是[區塊運算式](../003-expressions/309-block-expressions.md)的區塊。

### 零參數簡寫：`{ body }`

沒有參數的閉包可以從 `|| { body }` 簡寫為 `{ body }`。

```nabi
let h = thread::spawn { heavy_work() };
// == thread::spawn(|| { heavy_work() })

with_lock(mutex) { critical_section() };
```

這種簡寫只對緊接在函式呼叫之後的 `{ ... }` 有意義。其他任何位置的 `{ ... }` 都是[區塊運算式](../003-expressions/309-block-expressions.md)的一般區塊運算式。

### 一個以上的參數

有一個以上的參數時，寫法如下：

```nabi
let doubled = xs.iter().map |x| { x * 2 };
let pos = xs.iter().filter |x| { x > 0 };
let opened = with_file(path) |f| {
    let data = f.read();
    process(data)
};
pairs.for_each |k, v| { print(f"{k} {v}") };
```

管線符號（`|...|`）放在括號之外，緊接在主體之前。主體一律是區塊：一律是 `|x| { body }`，絕不是單行的 `|x| body`。

為了可讀性與一致性，尾隨閉包預設使用區塊主體。

### 不帶括號形式的規則

不帶呼叫括號地附加閉包，例如 `xs.iter().map |x| { .. }`，只在下列條件下才是尾隨閉包：

- 它前面緊接著路徑（`thread::spawn`、`f`）或成員名稱（`.map`）。
- 它後面接著 `|params|`、可選的 `-> R` 與 `{`。沒有這種形狀時，`|` 是位元 OR。
- `||` 一律是邏輯 OR。沒有參數的閉包使用 `{ body }` 簡寫。
- 在 `if` / `while` / `match` / `for` / `using` 的頭部內（條件、受檢值、被迭代的運算式、`using` 初始化式），不帶括號的尾隨閉包與 `{ body }` 簡寫都被停用；那裡的 `{` 開啟主體區塊。若頭部需要閉包，請使用括號（`if xs.iter().any(|x| { x > 0 }) { .. }`）。

```nabi
let ys = xs.iter().map |x| -> i64 { x as i64 * 2 };   // explicit return type
let z = a | b;                                  // bitwise OR: not followed by the `|params| {` shape
if ok || retry { .. }                           // logical OR
```

## 4.8.2. 可以自由選擇明確的括號

尾隨閉包是允許的，而非必要的。

```nabi
xs.iter().map(|x| x * 2);                // OK: explicit parentheses
xs.iter().map |x| { x * 2 };              // OK: trailing
```

團隊與使用者可以依其風格自由選擇。

## 4.8.3. 閉包不是最後一個引數時

尾隨閉包只能填入最後一個引數位置。若閉包是中間的引數，必須使用明確的括號。

```nabi
fn process(input: str, transform: fn(str) -> str, output: str) -> str { ... }

process("in", |s| { s.to_upper() }, "out");           // explicit parentheses (middle argument)
process("in", "out") |s| { s.to_upper() };            // ERROR (E0040): trailing only in the last position
```

## 4.8.4. 零引數的呼叫

當呼叫沒有其他引數時，空的括號也可以省略。

```nabi
thread::spawn { heavy() };        // same as thread::spawn() { heavy() }
                                   // == thread::spawn(|| { heavy() })
```

當唯一的參數是閉包時，例如 `thread::spawn`，這是最自然的呼叫形式。

在單獨的路徑（而非方法名稱）之後，具有結構字面值形狀的 `{ .. }` 就是結構字面值（[3.10](../003-expressions/310-if-expression.md)）：`{}`、`{ x }`、`{ x, .. }`、`{ x: .. }` 與 `{ ..x }`。因此 `run { }` 與 `zero { n }` 是結構字面值，當 `run` / `zero` 是函式時就是錯誤。這樣的主體請寫出括號：`run(|| { })`、`zero(|| n)`。

## 4.8.5. 巢狀

尾隨閉包可以自由巢狀。

```nabi
par::scope |s| {
    s.spawn { f() };
    s.spawn { g() };
};
```

相同的簡寫規則（`{ body }` ↔ `|| { body }`）也適用於內層閉包。

## 4.8.6. 解決剖析器的歧義

尾隨閉包帶來了一種歧義：某些程式碼既可以讀作區塊運算式，也可以讀作尾隨閉包。下列規則解決它。

| 位置                                                                              | 意義                                                                 |
|-----------------------------------------------------------------------------------|----------------------------------------------------------------------|
| 控制關鍵字（`if`、`else`、`for`、`while`、`loop`、`match`）之後的 `{ ... }`        | 控制主體                                                             |
| 緊接在函式呼叫之後的 `{ ... }`                                                     | 尾隨閉包（0 個引數）                                                 |
| 緊接在函式呼叫之後的 `\|params\| { ... }`                                          | 尾隨閉包（1 個以上引數）                                             |
| `x = { ... };` 之類的一般運算式位置中的 `{ ... }`                                  | 區塊運算式（立即求值）；0 引數閉包需要明確的 `\|\| { ... }`          |
| 運算式內的 `\|...\| { ... }`                                                       | 一般的閉包運算式                                                     |

簡言之：緊接在函式呼叫（`(args)`）之後的 `{ ... }` 或 `|...| { ... }`，無論中間有無空白，都是尾隨閉包。

## 4.8.7. 單獨的 `{}`：一般區塊與 0 引數閉包

```nabi
let x = { 42 };                   // block expression (value 42)
let f = { 42 };                   // ??
```

這兩行形式相同；哪一種意義優先？

規則是：單獨出現在運算式位置的 `{ body }` 是區塊運算式。

若意在 0 引數閉包，請明確寫出 `|| { body }`。

```nabi
let f = || { 42 };                // explicit 0-arg closure
let r = f();                       // call
```

也就是說，只有緊接在函式呼叫之後的 `{ body }` 才會被辨識為尾隨閉包。

## 4.8.8. par `for` 之類陳述式般位置中的閉包

對於平行迴圈、交易、情境管理器等，尾隨閉包能產生讀起來像關鍵字、適合 DSL 的形式。

```nabi
par::for_each(xs) |x| { ... }     // looks like par for
with_lock(mutex) { ... }           // looks like a with-statement
retry(3) { ... }                   // looks like a retry keyword
```

這與[關鍵字](../001-lexical/105-keywords.md)中盡量減少保留關鍵字的原則一致。把函式庫函式與尾隨閉包結合，就能得到讀起來像控制流程的模式。

## 4.8.9. 與具名引數結合

```nabi
fn run(name: str = "task", body: fn() -> ()) { ... }

run("compute") { ... };           // positional + trailing OK
run(name: "compute") { ... };     // named + trailing OK
```

位置引數、具名引數與尾隨閉包可以自由結合。`body` 沒有預設值，卻接在有預設值的 `name` 之後：預設值位於尾端，但型別為 `fn` 的最後一個參數豁免，好讓尾隨閉包能填入它（[4.2.3](402-parameters.md#423-default-values)）。在那個位置的其他型別參數則需要預設值。

## 4.8.10. 總結

尾隨閉包讓規格層級在不增加關鍵字的情況下，也能寫出類似 DSL 的運算式。重點在於使用者可以直接以函式庫的形式，建立看起來像語言關鍵字的模式。
