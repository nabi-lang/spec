# 1.3. 陳述式終止符

> [!IMPORTANT]
> Nabi 的陳述式以分號（`;`）結束。單靠換行無法結束陳述式。

```
stmt_terminator ::= ";"
```

## 1.3.1. 規則

陳述式終止符遵循幾條簡單的規則。陳述式以恰好一個 `;` 結束，多個陳述式可以共用一行。陳述式也可以跨越多行，因為換行只是語彙單元之間的空白。

示意如下：

```
<stmt> ;
<stmt> ; <stmt> ;             // several on one line
<part of a stmt
 spanning lines> ;            // multiple lines. ends at the final ;
```

## 1.3.2. 為什麼不在換行處自動插入

Nabi 不採用 JavaScript 的 ASI（Automatic Semicolon Insertion）、Go 的行尾自動分號或 Swift 以行為基礎的終止等自動規則。

理由如下。第一，自動規則會在少數情況下造成歧義；JavaScript 的 ASI 陷阱就是著名的例子。第二，明確的終止符讓格式化工具、LSP 與編輯器的「跳到下一個陳述式」等工具能確定地判斷陳述式邊界。最後，在快速的 REPL 輸入等場合，把多個陳述式寫在一行上是很自然的。

## 1.3.3. 空陳述式

只由 `;` 構成的空陳述式是允許的，而且沒有任何效果。

```nabi
;;;
```

上面的程式碼被辨識為三個空陳述式。接受這種形式，是為了讓工具產生的程式碼有更多自由。

## 1.3.4. 類區塊運算式陳述式

類區塊運算式以 `}` 結尾：`if`、`match`、`loop`、`while`、`for`，或區塊（`{ }`、`unsafe { }`、帶標籤的區塊）。當它出現在陳述式位置時，結尾的 `;` 可以省略（與 Rust 的規則相同）。此時該運算式的值必須是 unit；要捨棄非 unit 的值，請加上 `;`。那裡出現非 unit 的值是錯誤（E0067），無論在函式主體中還是在進入檔案的頂層陳述式中都是如此（[4.16.2](../004-functions/416-entry-point.md#4162-頂層陳述式的意義隱含-main)）。

帶有區塊主體[尾隨閉包](../004-functions/408-trailing-closure.md)的呼叫（`retry(3) { .. }`、`xs.each |x| { .. }`），當閉包的 `}` 之後沒有東西接續它時，也不需要 `;` 就能結束陳述式（[4.8.8](../004-functions/408-trailing-closure.md)）。此時該呼叫是一個運算式陳述式，就像寫了 `;` 一樣：它的值不必是 unit，而且會被捨棄；`()`、`none` 或 `Never` 以外的值會和任何運算式陳述式一樣以 lint `unused_result` 發出警告（`_ = run { .. };` 或 `#[discardable]` 函式可讓它安靜，[6.2.11](../006-meta-system/602-built-in-attributes.md#6211-未使用的結果unused_resultdiscardablemust_use)）。當呼叫位於陳述式開頭時，它和以關鍵字開頭的陳述式一樣在那個 `}` 結束，因此 `}` 之後的二元運算子會開始一個新的陳述式（見下文）。

```nabi
if ready { start(); }        // no ;
for x in xs { print(x); }
let n = if a { 1 } else { 2 };   // a let statement still ends with ;
```

若二元運算子緊接在以關鍵字（`if`、`match` 等）開頭的類區塊運算式陳述式之 `}` 之後，或在位於陳述式開頭的尾隨閉包呼叫之 `}` 之後，它會開始一個新的陳述式：`run { .. }` 之後接著 `-x;` 這一行，是兩個陳述式。只有 `.method()` 之類的成員存取與後綴 `?` 會延續運算式；`}` 之後的 `(` 或 `[` 會開始一個新的運算式，與 Rust 相同。以 `unsafe { }` 開頭的陳述式在二元運算子方面是例外：`unsafe { f() } != 0` 是一個運算式，但其 `}` 之後的 `(` 或 `[` 仍會開始一個新的運算式。在較大的運算式內部，例如 `let v = run { .. } - x;` 的初始化式中，該呼叫是一般的運算元。

```nabi
if c { a(); } else { b(); }
-x;                          // new statement (not the if statement minus x)
match v { ... }.len();       // . continues
unsafe { step(); }
(a + b) * c                  // a new expression, not a call of the block
run { a(); }                 // the call statement ends at }
-x;                          // new statement
let v = run { a(); } - x;    // in a let, - continues the call
```

出現在區塊中間、作為陳述式的非類區塊運算式需要 `;`。只有區塊的最後一個運算式（區塊的值）可以不帶 `;`。

`}` 之前的最後一個陳述式可以省略它的 `;`；省略與寫出相同。這涵蓋 `let`（任何模式）、指派、複合指派、`return`、`break`、`continue` 與 `throw`。區塊的值就是帶有 `;` 時的值：unit，或對 `return` / `break` / `continue` / `throw` 而言是發散的。不帶 `;` 的最後一個運算式仍然是區塊的值，而區塊中間的陳述式仍然需要 `;`。

```nabi
while i < n { i += 1 }           // same as { i += 1; }
if found { idx = k } else { idx = -1 }
let f = [ref total] |v: i32| { total += v };
let u = { let y = 1 };           // u is unit, as with { let y = 1; }
let s = { let y = 1; y + 1 };    // s is 2: the last expression is the value
```

文法由[區塊運算式](../003-expressions/309-block-expressions.md#391-形式)中的 `stmt` 定義。

## 1.3.5. 與其他語彙單元相鄰

`;` 語彙單元可以不帶空白地緊貼在其前後的語彙單元上。例如，不留空隙地寫成 `<stmt>;<stmt>;` 仍然能正確切分為兩個陳述式。為了可讀性而加上空格是建議的風格，但規格本身不強制風格。
