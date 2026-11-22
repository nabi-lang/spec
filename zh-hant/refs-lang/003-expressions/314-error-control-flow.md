# 3.14. 錯誤控制流程

`throw`、`try`、`catch` 與 `finally` 是用於引發錯誤、標示錯誤可能流經之處，以及接收並處理錯誤的控制流程。後綴 `?` 把 `T | none` 值的 `none` 提前回傳給呼叫者。

```
throw_stmt     ::= "throw" expr ";"
try_expr       ::= ("try" | "try?" | "try!") expr
catch_expr     ::= expr catch_clause+ finally_clause?
                 | expr finally_clause
catch_clause   ::= "catch" pattern? block            // struct patterns also allowed: catch E { f, .. } { .. }
finally_clause ::= "finally" block
question_expr  ::= expr "?"
```

`catch` 與 `finally` 是上下文關鍵字，只在位於運算式或區塊之後之後綴子句的位置是關鍵字（[1.5.4](../001-lexical/105-keywords.md#154-上下文關鍵字)）。可能擲回錯誤之函式的簽章（`throws`、`rethrows`）定義於 [`throws` 簽章](../004-functions/414-throws.md)。本文件涵蓋運算式與陳述式層級的控制流程。

## 3.14.1. `throw`：引發錯誤

```nabi
throw err;
```

`throw` 關鍵字之後接任何運算式構成的陳述式；運算式的結果值作為錯誤值被擲回。擲回的錯誤由包圍該陳述式的最近 `catch` 運算式處理；若未被處理，就成為外層函式的錯誤結果。此時外層函式的 `throws` 簽章必須接受該錯誤型別（[4.14.5](../004-functions/414-throws.md#4145-throws-與主體的一致)）。

`throw` 陳述式之後流程不會繼續，因此以 `throw` 陳述式結尾的區塊是發散的。

擲回的 `str` 會被包裝：`msg: str` 時的 `throw msg` 擲回 `ValueError::new(msg)`（[stdlib 2.14](../../refs-standard-libraries/200-core/214-error.md)）。外層函式必須和接受其他擲回錯誤一樣接受 `ValueError`：未定型的 `throws` 會接受，定型的 `throws (E)` 必須列出它（`throws (E | ValueError)`），否則就是 E0062，其說明會指出這一點。在 `str` 被當作 `str` 接收之處，它會原樣擲回：函式宣告了 `throws (str)`（或含有 `str` 的和型別），或外層的 `catch e: str` 分支接收它。

```nabi
fn check(age: i32) throws -> i32 {
    if age < 0 { throw f"age {age} is negative"; }   // a ValueError
    age
}
```

```nabi
let x = if cond { 42i32 } else { throw ValueError::new("not allowed"); };
// x: i32 (else branch is divergent)
```

## 3.14.2. `try`：標示呼叫處

```nabi
let n = try parse(s);
```

`try` 是前綴運算式運算子。對會擲回錯誤之函式（帶有 `throws` 簽章的函式）的呼叫，必須位於被 `try` 涵蓋的運算式內；否則就是編譯錯誤（E0062）。這條規則讓每個錯誤可能流經之處都在程式碼中可見。

`try e` 的值與型別就是 `e` 的值與型別。若 `e` 內會擲回錯誤的呼叫引發錯誤，由最近的 `catch` 處理；若沒有 `catch` 或沒有任何 `catch` 相符，就傳播到外層函式。

### 涵蓋範圍

`try` 涵蓋其右側的整個運算式。一個 `try` 可以一起標示多個呼叫。

```nabi
try a(b());                 // covers both a and b
try f().g().h();            // covers the whole chain
try {                       // a block is also an expression, so every call in it is covered
    let cfg = load(path);
    apply(cfg);
};
```

`try` 的運算元向右延伸到運算式結束之處，停在 `catch` / `finally` 子句、陳述式終止符 `;`、結尾括號或逗號之前。因此 `try a + b()` 是 `try (a + b())`，而 `try parse(s) catch _ { 0 }` 的 `catch` 附加在整個 `try parse(s)` 上。確切的結合規則定義於[優先順序與結合性](302-precedence-and-associativity.md)。

當 `try` 作為二元運算子的右運算元或出現在函式引數位置時，它只涵蓋該運算元。

唯一的例外是 `for` 運算元：`try for pat in xs { body }` 涵蓋可迭代的 `xs` 以及迴圈的每一步，而不涵蓋 `body`（[3.13.3](313-loop-expressions.md#3133-for以迭代器為基礎的重複)）。主體中會擲回錯誤的呼叫需要它自己的 `try`。

```nabi
let total = base + try fetch_extra();
log(try describe(item));
```

### 與 `await` 一起

`async` 會擲回錯誤之函式的錯誤在 `await` 處交付，因此 `await` 運算式由 `try` 涵蓋（[4.14.10](../004-functions/414-throws.md#41410-throws-對和型別簽章的影響)）。產生器的錯誤同樣在 `next` 處接收。

```nabi
let data = try await fetch(url);
```

## 3.14.3. `try?` 與 `try!`

`try` 的兩種變體不把錯誤傳播到外層函式；它們就地把錯誤轉換為另一種形式。

| 形式     | `e: T` 時的結果型別      | 發生錯誤時                        |
|----------|--------------------------|-----------------------------------|
| `try e`  | `T`                      | 最近的 `catch`，或傳播            |
| `try? e` | `T?`                     | 捨棄錯誤並得到 `none`             |
| `try! e` | `T`                      | panic（3.14.8）                   |

```nabi
let port = try? env::var("PORT") ?? "8080";      // (try? env::var("PORT")) ?? "8080"
let cfg = try! load_builtin_config();      // failure is a program bug
```

`try? e` 的型別是以 `T` 作為泛型項的 `T?`（[2.6.4 泛型和型別的分層](../002-types/206-sum-types.md#泛型和型別的分層)）。若 `T` 沒有 `none`，就是攤平的 `T | none`。若 `T` 已經有 `none`，結果是巢狀的 `(T | none)?`，因此擲回的錯誤（外層的 `none`）與回傳的 `none`（內層的）保持區分。`??`、`?`、`is none` 與模式只作用於外層。

```nabi
fn lookup(k: str) throws (IoError) -> i32 | none { .. }

let r = try? lookup("a");             // (i32 | none)?
match r {
    none => print("lookup failed"),   // outer: an error was thrown
    v => print(f"{v:?}"),             // inner: i32 | none from lookup
}
let v = try? lookup("a") ?? none;     // i32 | none: error and absence merged
```

`try?` / `try!` 的 `?` / `!` 必須緊接在 `try` 之後。`try !x` 是 `try (!x)`。由於 `try?` 與 `try!` 不傳播錯誤，外層函式不需要是 `throws`。它們的涵蓋範圍與 `try` 相同，只是在 `??` 之前結束：`try? f() ?? d` 是 `(try? f()) ?? d`，錯誤時得到 `d`，這正是寫出預設值的本意。要在 `f() ?? d` 出錯時得到 `none`，請將它分組：`try? (f() ?? d)`。在 `T | none` 結果上，`??` 仍只作用於外層，如上例所示（`try? lookup("a") ?? none` 會把錯誤與不存在的值合併）。

## 3.14.4. `catch`：後綴處理運算式

```nabi
let n = try parse(s) catch e: ParseError { 0 } catch _ { -1 };
```

`catch` 子句是附加在其左側運算式上的後綴子句。若在對左側運算式求值時發生錯誤，就由上到下檢查 `catch` 子句，並對第一個相符子句的區塊求值。若左側運算式正常完成，就不執行任何 `catch` 區塊。左側運算式的錯誤，是由以 `try` 標示的會擲回錯誤之呼叫以及 `throw` 陳述式所產生的錯誤。

若沒有任何子句相符，錯誤會再次傳播，而外層函式必須 `throws` 該錯誤型別。

### 區塊形式

`try` 涵蓋一個區塊、而 `catch` / `finally` 附加在該運算式上的形式，以相同規則解讀。

```nabi
try {
    body
} catch e {
    handler
}
```

### 模式與型別比對

```nabi
try fetch(url)
catch e: TimeoutError { retry() }
catch HttpError { code, msg } { fail(code, msg) }
catch e: any Error { log(f"{e}"); fallback() }
```

`catch` 之後可以接[模式](311-patterns.md)的任何模式形式。`e: TimeoutError` 是型別檢查模式，在相符的分支中 `e` 被收窄為該型別。`catch e: A | B` 把捕捉到的錯誤繫結為和型別 `A | B`（錯誤和型別的子集）。未定型 `throws` 的錯誤是 [`any Error`](../002-types/202-any-slot.md) 值，因此要以一個名稱接收所有錯誤，請使用 `catch e: any Error` 或 `catch e`。依照 [trait 不能作為型別](../002-types/212-trait-as-type-banned.md)，寫出單獨的 trait 名稱 `catch e: Error` 是編譯錯誤（E0071）。

省略模式的 `catch { ... }` 與 `catch _ { ... }` 相同。

### 窮盡性

若 `catch` 子句涵蓋了左側運算式可能擲回的每一種錯誤型別，`catch` 運算式就不再擲回錯誤。對定型 `throws (A | B)` 的呼叫，涵蓋 `A` 與 `B` 就足夠；對未定型的 `throws`，必須有一個接受所有值的子句，例如 `catch e`、`catch _` 或 `catch e: any Error`。

```nabi
fn parse_or_zero(s: str) -> i32 {          // non-throwing
    try parse(s) catch _ { 0 }             // covers every error
}
```

### 無法到達的 `try` 與 `catch` 的警告

在下列情況下，編譯器會發出警告（E0096）：

- 沒有涵蓋任何會擲回錯誤之呼叫的 `try`
- 沒有任何錯誤能到達的 `catch` 子句或分支

觸發警告的子句仍會進行型別檢查。

## 3.14.5. `finally`

```nabi
try {
    body
} finally {
    cleanup
}
```

它也可以與 `catch` 結合：

```nabi
try {
    body
} catch e: IoError {
    handler
} finally {
    cleanup
}
```

`finally` 區塊在離開左側運算式的每一種情況下都會執行：正常完成、由 `catch` 處理、未處理之錯誤的傳播，以及經由 `return` / `break` / `continue` 離開。因此它適合用於釋放資源、釋放鎖以及還原暫時的狀態。若 `catch` 區塊擲回新的錯誤，或在 `finally` 區塊內擲回錯誤，新的錯誤優先。

離開 `finally` 區塊本身的 `return`、`break` 或 `continue` 是編譯錯誤（E0059）：它會丟掉正在傳遞中的錯誤。請把跳躍移到 `try` 運算式之後。跳到 `finally` 區塊內之迴圈或帶標籤區塊的跳躍，或那裡閉包中的 `return`，都留在內部，是允許的。

panic 與致命錯誤（3.14.8）會終止行程，因此 `finally` 不會執行。

## 3.14.6. 運算式的值與型別

`try`、`catch` 與 `finally` 都是運算式。

```nabi
let x = try parse(input) catch _ { default };
```

`catch` 運算式的值，在左側運算式正常完成時是其值，在捕捉到錯誤時是處理區塊的值。左側運算式與每個處理區塊都必須是相同型別（[運算式概觀](README.md)的分支結果規則）；以 `throw` 或 `return` 結尾的處理區塊是發散的，不納入型別檢查。`finally` 區塊不產生值，也不影響運算式的結果。

## 3.14.7. 擲回的值與函式簽章

`throw err` 的 `err` 是任何運算式。帶有定型 `throws (E)` 的函式可以擲回對應於 `E` 成員的值；帶有未定型 `throws` 的函式是 `throws (any Error)` 的簡寫，可以擲回實作了標準函式庫 `Error` trait 的值。擲回的 `str` 是 `ValueError`（3.14.1）。

```nabi
fn parse(s: str) throws -> i32 { ... }              // untyped, any Error
fn parse(s: str) throws (TypeError) -> i32 { ... }  // single type
fn parse(s: str) throws (A | B | C) -> i32 { ... }  // sum type
```

簽章中沒有 `throws` 的函式，不會有任何錯誤離開它。當 `throws (T)` 的 `T` 是和型別時，它就是該函式可能擲回之錯誤的封閉集合，呼叫者用它來撰寫 `catch` 子句並檢查窮盡性。

函式簽章與 `throws` 的意義詳細定義於 [`throws` 簽章](../004-functions/414-throws.md)。

## 3.14.8. panic 與致命錯誤

與 `throws` 錯誤不同，下列情況不會以值的形式交付，也無法以 `catch` 捕捉。

- panic。包括呼叫標準函式庫的 `panic`、`try!` 失敗、除以零、整數溢位、邊界檢查失敗等。
- 致命錯誤。無法繼續執行的狀態，例如 `StackOverflowError`、`OutOfMemoryError` 與 `InternalError`。

在這兩種情況下，執行環境都會把訊息印到標準錯誤（stderr），並以結束碼 `101` 終止行程。哪些情況是 panic 或致命錯誤的確切清單，由標準函式庫定義。

失敗的 `try!` 在 `try!` 的位置 panic。其訊息以與離開 `main` 之未捕捉錯誤相同的方式顯示錯誤（[4.16.4](../004-functions/416-entry-point.md)：其 `Display`，否則其 `Debug`，否則其型別名稱），接著顯示引發它的 `throw` 的位置，以及它到達 `try!` 所經過的位置：

```text
thread 'main' panicked at src/main.nabi:12:13:
try! failed: no such key `port`
  raised at src/config.nabi:30:9
  through src/main.nabi:12:18
```

當錯誤就是在該位置擲回的，或 `throw` 未知時，會省略 `through` 那一行：此時 `raised at` 給出錯誤到達 `try!` 所經過的位置。`catch` 內的 `throw` 會重新引發錯誤，因此顯示的是它的位置。被子句都不相符的 `catch` 傳下去的錯誤，保留其 `throw` 的位置。標準函式庫擲回的錯誤，或作為值傳給另一個執行緒或任務的錯誤，不會指出任何 `throw`；`try_block_on`（因而 `async` main 也是）保留其主體中 `throw` 的位置。

## 3.14.9. 求值順序

`catch` 運算式與 `finally` 依下列順序求值：

1. 對左側運算式求值。
2. 若左側運算式引發錯誤，找出相符的 `catch` 子句並對其區塊求值。
3. 若有 `finally` 區塊，無論以何種方式離開左側運算式，都對它求值。
4. 若沒有任何 `catch` 子句相符，執行 `finally`，然後把錯誤向外傳播。

對 `throw` 陳述式求值時，先對其運算式求值，然後擲回錯誤。

## 3.14.10. `?` 運算子：提前回傳 `none`

`?` 是後綴一元運算子，它解開 `T | none` 值，若為 `none` 就立即從外層函式回傳 `none`。

```nabi
fn first_word_len(text: str | none) -> usize | none {
    let t = text?;                      // none → return none immediately
    let w = t.split(" ").next()?;
    w.len()
}
```

在 `e: T | none` 的 `e?` 中，運算式的值是 `T`。若 `e` 是 `none`，外層函式執行 `return none`。因此外層函式的回傳型別必須包含 `none` 成員（`_ | none`）；否則就是編譯錯誤（E0064）。由於是後綴的，它可以自然地放在方法鏈的末端。

### 去糖規則

`?` 由編譯器直接展開，不經過任何 trait。

```
expr?
```

展開為：

```nabi
match expr {
    none => return none,
    v => v,
}
```

### 適用範圍

`?` 只適用於 `T | none`。它不是傳播錯誤的手段，不能用在標準函式庫的 `Result` 值或會擲回錯誤的呼叫上。錯誤傳播以 `throws` 簽章與 `try` 標示。

```nabi
fn load() -> Result<Config, IoError> {
    let raw = read_result(path)?;     // ERROR (E0064): `?` is only for `T | none`.
                                      //   Use `throws` and `try` for error propagation.
    ...
}
```

閉包主體內的 `?` 作用於閉包的回傳型別，與外層函式的回傳型別無關。

```nabi
let lens = users.iter().map(|u| -> usize | none { let nick = u.nickname?; nick.len() });
```

要以替代值填補 `none`，或讓成員存取穿過去，請使用[可選鏈結與空值合併](308-optional-chaining-and-coalescing.md)的 `?.` 與 `??`。
