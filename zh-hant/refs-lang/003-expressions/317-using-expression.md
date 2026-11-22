# 3.17. `using` 運算式：確定性的資源釋放

`using` 關鍵字（[關鍵字](../001-lexical/105-keywords.md)）是把資源綁定到區塊生命週期的運算式形式。無論區塊以何種方式結束（正常完成、提前 `return`、經由 `?` 回傳 `none`、錯誤傳播），所繫結資源的 `Drop::drop` 一律會被呼叫。

GC 會自動回收記憶體，但外部資源（檔案、通訊端、互斥鎖、FFI 控制代碼、GC 釘選等）必須確定性地釋放。`Drop` 值是仿射的：編譯器在其擁有者離開作用域之處釋放它（[stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop釋放時的清理)）。`using` 讓那個點變得明確：它把值綁定到其區塊，並在區塊結束時釋放它；對於關閉可能失敗的值，它也是不必寫 `try x.close()` 就能關閉它的方式。

## 3.17.1. 形式

```
using_expr     ::= "using" using_binding ("," using_binding)* block
using_stmt     ::= "using" using_binding ("," using_binding)* ";"
using_binding  ::= pattern "=" expr
```

`using_stmt` 是 [3.17.9](#3179-陳述式形式) 的陳述式形式。

各組成部分的意義：

- `pattern` 可以是[模式](311-patterns.md)中定義的任何不可反駁的模式（識別字、元組、結構解構等）。大多數情況下使用單一識別字。被釋放的是 `expr` 的整個值。
- `expr` 是建立或取得資源的運算式。其結果型別必須實作 [`Drop`](../../refs-standard-libraries/200-core/213-marker.md) trait。
- 若取得資源可能失敗，請以 `try` 標示 `expr`（`using f = try fs::open(path) { ... }`）。若取得時引發錯誤，控制在沒有繫結任何資源的情況下離開，因此不會呼叫 `Drop`。詳細語意遵循[錯誤控制流程](314-error-control-flow.md)。
- 以 `,` 連接的多個繫結構成多個資源；詳細行為定義於 [3.17.4 多個資源](#3174-多個資源)。
- `block` 遵循[區塊運算式](309-block-expressions.md)的規則，其最後一個運算式的值就是整個 `using` 運算式的值。

## 3.17.2. 最簡單的範例

```nabi
using f = try fs::open(path) {
    try f.read_to_end()               // the last expression is the value of using
}                                  // ← f.drop() is called automatically here
```

`fs::open(path)` 的回傳型別是 `File`，而 `File: Drop`，因此它可以作為 `using` 的目標。區塊結束時，`File::drop` 被呼叫，作業系統的控制代碼被關閉。

## 3.17.3. 與 `try` 結合

當取得資源本身可能失敗時，以 `try` 標示取得的運算式。

```nabi
fn read_text(path: str) throws (IoError) -> str {
    using f = try fs::open(path) {     // on failure, IoError propagates
        try f.read_to_str()            // try can propagate inside the body too
    }                                  // if f was bound, it is dropped on both success and error
}
```

這種形式在語意上等價於下列展開後的程式碼。

```nabi
let f = try fs::open(path);
try {
    f.read_to_str()
} finally {
    f.drop();          // a call inserted by the compiler; writing it directly is an error
}
```

即使錯誤在主體內傳播，`drop` 也會在對應於 `finally` 的位置被呼叫。

## 3.17.4. 多個資源

要把多個資源綁定到一個區塊，請巢狀使用 `using`，或以逗號連接繫結。本規格允許兩種形式，其意義等價。

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

兩種形式都遵循**反向釋放**：先 drop `b`，再 drop `a`。這與 Java 的 try-with-resources 規則相同。依相反順序釋放，可以安全地拆解相依關係（`b` 參照 `a`）。

## 3.17.5. 作為運算式的 `using`

`using` 是運算式而不是陳述式，其值是主體區塊的值。

```nabi
let content = using f = try fs::open(path) {
    try f.read_to_end()
};
```

這是與以函式為範圍的 `defer`（Go）最大的結構差異。`using` 運算式遵循[區塊運算式](309-block-expressions.md)的相關規則，可以用在任何運算式位置。

## 3.17.6. drop 被呼叫的時機

`using` 保證在下列五條路徑上都會 `drop` 所繫結的資源：

- 區塊正常完成（最後一個運算式已求值）
- 主體內的 `return`
- 主體內的 `break` / `continue`
- 主體內經由 `?` 回傳 `none`
- 主體內的 `throw`，以及以 `try` 標示之呼叫的錯誤傳播

panic 與致命錯誤會立即終止行程，因此不會呼叫 `drop`（[3.14.8](314-error-control-flow.md#3148-panic-與致命錯誤)）。這是 panic 的一般結果，而非 `using` 特有的行為。

一個值只會被釋放一次（[stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop釋放時的清理)）。`using x = a { .. }` 把 `a` 移入 `x`，因此第二個 `using y = a { .. }`，或在第一個區塊之後使用 `a`，是編譯錯誤（E0133）：請給每個 `using` 它自己建立的值（`using g = m.lock() { .. }`），或在一個 `using` 區塊內完成所有工作。在區塊內關閉的值（`try x.close()`）不會再被釋放。

## 3.17.7. 可以繫結的型別

`using` 右側運算式的型別必須實作 [`Drop`](../../refs-standard-libraries/200-core/213-marker.md)，或包含具有 `Drop` 的欄位（drop glue，[stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop釋放時的清理)）。否則就是編譯錯誤（E0066）。

```nabi
using x = 42 { ... }   // ERROR (E0066): using requires a type implementing Drop; i64 does not implement Drop
```

這項限制確保只有有意義的資源才以 `using` 繫結，在規格層級排除了對一般值使用 `using`。

## 3.17.8. 與其他語言的比較

| 語言     | 形式                          | 範圍單位     | 備註                                    |
|----------|-------------------------------|--------------|-----------------------------------------|
| Nabi      | `using x = try e { ... }`、`using x = e;` | 區塊 | 運算式、多個資源、整合 `try` |
| C#       | `using (var x = e) { ... }`   | 區塊         | 需要 `IDisposable`                      |
| Java     | `try (var x = e) { ... }`     | try 區塊     | 需要 `AutoCloseable`，多個資源          |
| Python   | `with x as f: ...`            | 區塊         | `__enter__` / `__exit__`                |
| Swift    | `defer { x.close() }`         | 函式         | 必須明確寫出                            |
| Go       | `defer x.Close()`             | **函式**     | 不以區塊為範圍                          |
| Zig      | `defer x.close();`            | 區塊         | `errdefer` 變體                         |
| Rust     | （自動 RAII，沒有關鍵字）     | 區塊         | 沒有 GC；drop 時機自動決定              |

Nabi 的 `using` 結合了 GC 語言（C#、Java、Python）的明確區塊模式，以及作為運算式的可組合性、`try` 的整合與多個資源的反向釋放。

## 3.17.9. 陳述式形式

`using x = e;` 把 `x` 綁定到外層區塊的其餘部分，並在該區塊結束時釋放它：

```nabi
fn copy_lines(from: Path) throws (IoError) {
    using r = BufReader::new(try fs::open(from));
    try for line in r.lines() {
        println(f"> {line}");
    }
}                                  // ← r is released here
```

它的意義是以同一區塊中其後的陳述式為主體的區塊形式：

```nabi
using a = e;
s1;
s2
// is
using a = e { s1; s2 }
```

因此區塊形式的每一條規則都成立：在 3.17.6 的所有路徑上釋放、多個繫結以及同一區塊中的多個 `using` 陳述式依相反順序釋放、3.17.7 的 `Drop` 要求，以及對所繫結值之視圖的檢查。區塊的值是 `using` 之後各陳述式的值。在 `main.nabi` 的頂層陳述式中，區塊的其餘部分就是程式的其餘部分。

陳述式形式要位於陳述式開頭；在其他位置（`let x = using r = e;`）是編譯錯誤，其說明會指出區塊形式。由於其後沒有主體，結構字面值可以不加括號地結束其繫結（`using r = R { n: 1 };`）。對短於區塊其餘部分的範圍，以及把 `using` 當作運算式時，仍使用區塊形式。
