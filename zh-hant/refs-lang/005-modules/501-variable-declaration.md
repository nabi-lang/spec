# 5.1. 變數宣告

新變數以 `let` 關鍵字（[關鍵字](../001-lexical/105-keywords.md)）宣告。本文件定義其形式，以及區分重新指派與新宣告的規則。

## 5.1.1. 新宣告：`let`

```
let_stmt    ::= "let" let_lhs (":" type_expr)? "=" expr ";"
let_lhs     ::= identifier
              | irrefutable_pattern    // tuple / struct destructure (see 5.1.11)
```

最常見的形式是單一識別字（`let x = ...`）。左側為不可反駁的模式（5.1.11）時，它就是[解構指派](../003-expressions/315-destructuring.md)。`let-else` 形式（[let-else](../003-expressions/316-let-else.md)）是另一種語法。

```nabi
let x = 42i32;
let y: i32 = 10i32;
let z;                            // ERROR: no initializer
let z: i32;                        // ERROR: no initializer
```

基本形式是 `let`、名稱、可選的 `: type`，然後是 `=` 與初始化式。初始化式是必要的；不存在未初始化的變數。有型別標註時，變數是該型別的靜態槽位；沒有時，是由初始化式推論出之型別的靜態槽位（5.1.7）。[`any` 槽位](../002-types/202-any-slot.md)只在明確寫出 `: any` 時才會出現。

這排除了讀取尚未填入之槽位的可能（Rust 的借用檢查器所阻擋的那類問題）。當初始值無法事先決定時，請使用[和型別](../002-types/206-sum-types.md) `T | none` 明確表示不存在。

```nabi
let cache: Data | none = none;     // explicitly "not yet"
cache = compute();                  // filled later
```

## 5.1.2. 重新指派：不寫 `let`

要為已宣告的變數指派新值，請省略 `let`。

```nabi
let counter: i32 = 0i32;
counter = counter + 1i32;         // OK: reassignment
counter = counter + 1i32;         // OK: reassignment again
```

對從未宣告過的名稱不帶 `let` 地指派是編譯錯誤。

```nabi
let counter: i32 = 0i32;
coutner = counter + 1i32;         // ERROR (E0057): `coutner` is not declared
                                   //        help: did you mean `counter`?
```

這擋住了 Julia 與 Python 中常見、打錯字而默默建立新變數的陷阱。

## 5.1.3. 判別新宣告與重新指派

| 形式              | 意義                                                      |
|-------------------|-----------------------------------------------------------|
| `let name = ...;` | 新宣告。在該作用域中建立新的繫結。                        |
| `name = ...;`     | 重新指派。再次寫入既有繫結的槽位。                        |

明確性優先：打錯字是編譯錯誤。

## 5.1.4. 同一作用域中的重新宣告

`let` 可以宣告一個在同一作用域中已經繫結的名稱。這會建立一個具有自己槽位的新繫結；從該陳述式結束起，該名稱指稱新的繫結（遮蔽）。

```nabi
let line = "  42  ";
let line = line.trim();                // the initializer reads the old `line`
let n: i32 = try! line.parse();
let n = n * 2;                         // a new slot, of the same type here
let x = "abc";
let x = x.len();                       // the type may differ: str, then usize
let (a, b) = (1, 2);
let (a, b) = (b, a);                   // patterns follow the same rule
```

- 初始化式在新繫結存在之前檢查，因此其中的名稱指稱舊的繫結。新繫結在該陳述式之後才開始。
- 新繫結有自己的型別，無論是寫出的或推論的（5.1.7）；它不必與舊的相符。每個槽位都保持其宣告時的型別：舊的槽位不會改變，而不帶 `let` 的重新指派（5.1.2）寫入最新的繫結。
- 舊的繫結不會被重新指派、捨棄或關閉。它像以前一樣存活到其作用域結束（5.1.10），只是無法再以名稱存取。從它建立的閉包、視圖與守衛物件仍然參照它。
- 必須關閉的值（[stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-release)）不會因重新宣告而關閉：請在重新宣告其名稱之前關閉或移動它。把它移入新的值是預期的用法。讓它保持開啟是一般的編譯錯誤（E0134），其說明會提示在重新宣告之前關閉或移動它。

```nabi
let f = try fs::open(path);
let f = io::BufReader::new(f);         // OK: the reader owns the file now
try for l in f.lines() { println(l); }
try f.close();

let g = try fs::open(path);
let g = 1;                             // ERROR (E0134): the first `g` is never closed
                                        //        help: close or move it before redeclaring it
```

從舊繫結建立的串流視圖，在舊值移動之處結束，移入新繫結時也是如此（E0139，[stdlib 2.23.1](../../refs-standard-libraries/200-core/223-io.md#2231-read)）。

要改變一個繫結的值而不是建立新的繫結，請不帶 `let` 地指派（5.1.2）。REPL 與 Jupyter 在儲存格的頂層遵循相同的規則（[9.7.3](../009-tools/907-jit-cells.md#同名的重新宣告)）。

## 5.1.5. 作用域：以區塊為單位

變數可見的範圍依[區塊運算式](../003-expressions/309-block-expressions.md)的區塊決定。

```nabi
let y = 1;
if cond {
    y = 2;                         // reassigns the outer y (same slot)
    let z = 2;                     // declares block-local z
}
// z is not accessible here
```

區塊內的 `let` 是屬於該區塊的新區域繫結，因此該變數在區塊之外不可見。內層同名的 `let` 只在該區塊內隱藏外部變數。

## 5.1.6. 巢狀作用域：各自獨立的繫結

在內層區塊中重新宣告同名的變數時，它是另一個繫結。

```nabi
let a = 1;
{
    let a: i32 = 42i32;            // block-local a, a different slot from the outer a
    a = 10i32;                     // reassigns the inner a (outer a unaffected)
}
// here a = 1 (outer a unchanged)
```

內層區塊的 `let` 建立一個全新的繫結。即使外部存在同名的變數，兩者也是獨立的槽位。

這個設計同時保留了「內層變數在區塊結束時消失」的意義，以及保護外部變數的意圖。

### 與同一作用域中重新宣告的差異

兩者都建立一個隱藏舊名稱的新繫結，也都不改變舊的槽位。差別在於舊名稱在哪裡重新出現：內層區塊中的繫結隨區塊結束，之後外部的名稱又重新可見。同一作用域中的重新宣告（5.1.4）持續到該作用域結束，因此舊的繫結無法再以名稱存取。

```nabi
let a = 1;
{
    let a = "inner";               // hides the outer a inside the block
}
// a = 1 again
let a = a + 1;                     // same scope: a = 2 from here to the end
```

## 5.1.7. 型別標註與型別推論

```nabi
let a = 42i32;                     // a: i32 (inferred)
let b: i32 = 42i32;                // b: i32 (annotated)
let c: any = 42i32;                // c: any (annotated)
let d = if cond { 1i32 } else { 2i32 };   // d: i32 (inferred)
```

有型別標註時，變數是該型別的靜態槽位；沒有時，是由初始化式運算式推論出之型別的靜態槽位（區域型別推論）。之後的重新指派都對照該型別檢查。[`any` 槽位](../002-types/202-any-slot.md)只在明確寫出 `: any` 時才會出現。

```nabi
let n = 1i32;
n = 2i32;                          // OK
n = "hi";                          // ERROR: n is i32
```

若初始化式沒有完全決定型別，推論會從變數作用域內之後的使用（把值傳到型別已決定之位置的使用）繼續（[2.10.2](../002-types/210-generics.md#型別引數推論)）。若仍無法決定型別，就是編譯錯誤，必須標註型別。

```nabi
let x = none;                      // ERROR: type cannot be determined
let x: Data | none = none;         // OK

let xs = [];                       // ERROR if there are no later uses: element type cannot be determined
let xs: List<i32> = [];            // OK

let y = return;                    // ERROR: initializer only diverges
```

分支型別不同的 `if` / `match` 運算式不會推論出任何型別，除非某個分支的和型別容納每個分支的型別（此時就是該和型別）。若需要和型別，請標註槽位。

```nabi
let e = if cond { 1i32 } else { "hi" };          // ERROR: branch type mismatch
let e: i32 | str = if cond { 1i32 } else { "hi" };   // OK
```

推論出的型別與分支結果規則，請見[運算式概觀](../003-expressions/README.md)。

## 5.1.8. 多重宣告（無）

```
let a = 1, b = 2;                  // not adopted
```

不提供以逗號一次宣告多個變數的 C 風格形式。請每行寫一個，或使用[解構指派](../003-expressions/315-destructuring.md)的元組解構。

```nabi
let (a, b) = (1, 2);               // destructuring assignment: OK
let a = 1; let b = 2;              // two statements: OK
```

## 5.1.9. `let` 可以出現的位置

`let` 是陳述式（見[運算式概觀](../003-expressions/README.md)中的轉換規則）。

`let` 不允許出現在模組層級（模組層級的值槽位必須以 [`const`](502-const.md) 或 [`static`](503-static.md) 宣告）。例外是進入檔案 `main.nabi`，其頂層陳述式（包括 `let`）構成隱含 `main`（[4.16.2](../004-functions/416-entry-point.md#4162-頂層陳述式的意義隱含-main)）；在其他任何模組中它們是錯誤（E0036）。它可以用在函式主體與區塊中。`if` / `match` / `loop` 等控制結構的主體本身就是區塊，因此也可以在那裡使用。`let` 不能直接出現在運算式位置，因為 `let` 是陳述式而不是運算式。

## 5.1.10. 以 `let` 宣告之變數的生命週期

變數從宣告處存活到引入它的作用域結束為止。同一作用域中對其名稱的重新宣告（5.1.4）不會結束它。

經由 `ref` 擷取（[擷取語意](../004-functions/406-capture-semantics.md)）被閉包擷取的變數，在該閉包物件存活期間也會額外保持存活。它由 [GC](../000-meta/007-glossary.md#gc垃圾回收器) 管理，因此使用者絕不需要手動釋放它。

## 5.1.11. `let` 的左側與[解構指派](../003-expressions/315-destructuring.md)

`let` 的左側可以是[模式](../003-expressions/311-patterns.md)中不可反駁的模式。這就是[解構指派](../003-expressions/315-destructuring.md)。

```nabi
let (x, y) = compute();
let Point { x, y } = origin;
```

解構的語意定義於[解構指派](../003-expressions/315-destructuring.md)。每個子模式都被繫結到其變數，而所有變數一起被引入單一作用域。
