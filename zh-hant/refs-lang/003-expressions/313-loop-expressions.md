# 3.13. 迴圈運算式

重複執行以三種形式表達：`loop`、`while` 與 `for`。三者都是運算式；本文件定義各自的語法與值的求值規則。

```
loop_expr   ::= label? "loop" block
              | label? "while" expr block
              | label? "for" pattern "in" expr block
              | label? "for" "await" pattern "in" expr block
              | label? "for" "ref" identifier "in" expr block
label       ::= "'" identifier ":"
break_stmt  ::= "break" label_target? expr? ";"
continue_stmt ::= "continue" label_target? ";"
label_target ::= "'" identifier
```

> [!NOTE]
> `in` 是上下文關鍵字，只在 `for pat in` 位置是關鍵字（[1.5.4](../001-lexical/105-keywords.md#154-上下文關鍵字)）。

`for await` 的意義與用法定義於[串流](../004-functions/413-stream.md)。它從非同步串流逐一接收項目，並為每個項目執行主體。

標籤（`'name:`）的意義定義於 [3.13.4 標籤](#3134-標籤控制巢狀迴圈)。帶標籤的**區塊**（`'b: { ... }`）定義於[區塊運算式](309-block-expressions.md)。

## 3.13.1. `loop`：無限重複

```nabi
loop {
    ...
}
```

`loop` 關鍵字之後接一個區塊主體，它會無限重複。只能以 `break`、`return` 或 `throw` 離開。

### 以 `break value` 回傳值

```nabi
let v = loop {
    let x = compute();
    if x.converged { break x; }
};
```

若 `break` 之後接一個運算式，其值就是 `loop` 運算式的結果。若迴圈像 `break;` 這樣不帶值離開，`loop` 的結果是 unit。一個 `loop`（或帶標籤的區塊）的所有 `break` 值都必須是同一個型別，與分支相同：某個 `break` 值的和型別若容納其他每個 `break` 值的型別，就是該型別，無論哪個 `break` 在前（在 `x: i32 | none` 之下，`break none` 與 `break x` 得到 `i32 | none`）。若接收的一方有型別標註，它們必須是該型別。混用 `break;` 與 `break v` 是編譯錯誤（E0041）。

### 結果型別

至少出現一次 `break value` 的 `loop`，以該值的型別作為其結果型別。只有 `break;` 的 `loop` 的結果型別是 unit。完全沒有 `break` 的 `loop` 是發散的，永遠不會完成。

## 3.13.2. `while`：依條件重複

```nabi
while cond {
    ...
}
```

`while` 關鍵字之後接一個 bool 條件與一個區塊主體。在每次迭代開始時對 `cond` 求值：若為 `true`，執行主體；若為 `false`，迴圈結束。

### 結果值

`while` 的結果值一律是 unit。

```nabi
let x = while cond { ... };     // x: unit
```

理由是，當 `while` 正常結束時（`cond` 為 `false`），沒有自然的值可以回傳。它不是像 `loop { break v; }` 那樣用於明確 `break value` 的形式：以 `while`（或 `for`）為目標、帶有值的 `break` 是編譯錯誤（E0059）。

要回傳值，請使用 `loop { ... break v; ... }`。

### `while let`：與模式比對結合

```nabi
while let v = compute() {        // compute(): T | none; v: T
    consume(v);
}
```

```
while_let ::= "while" "let" pattern (":" type_expr)? "=" expr block
```

每次迭代都會對運算式求值，並依與 `if let` 相同的規則比對（[3.10.5](310-if-expression.md#3105-if-let與模式比對結合)），因此單純的名稱取得去掉 `none` 的值。比對成功時執行主體；否則迴圈結束。模式的繫結在主體中可見。

[模式](311-patterns.md)的每一種模式形式都可以使用；通常用來比對 `enum` 或和型別的變體。

## 3.13.3. `for`：以迭代器為基礎的重複

```nabi
for pattern in iter_expr {
    ...
}
```

`for` 關鍵字之後依序接一個模式、`in`、一個迭代運算式與一個區塊主體。迭代運算式必須是迭代器或可迭代的值。結構字面值不能出現在迭代運算式或 `while` 條件的頂層（[3.10.3](310-if-expression.md#條件位置的結構字面值)）。每次迭代時，下一個元素與模式比對，並在主體中繫結。

### 迭代的去糖

```
for pat in xs { body }
```

在語意上等價於：

```
let it = <typeof(xs) as IntoIterator>::into_iter(xs);
loop {
    match <typeof(it) as Iterator>::next(ref it) {     // returns: T | none
        pat: T => body,
        none   => break,
    }
}
```

（確切的 trait 與方法名稱由標準函式庫定義。）

當值實作了 `IntoIterator` 時，迴圈步進其 `Iterator`。否則，當它實作了 `TryIterator`（[stdlib 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator可能失敗的迭代)）時，迴圈直接步進它；它的 `next` 會擲回錯誤，因此迴圈必須寫成 `try for`（或 `try! for` / `try? for`）。其他任何值都是編譯錯誤（E0068）。

```
try for pat in xs { body }
```

在語意上等價於：

```
let it = try xs;                                     // the iterable is covered
loop {
    match try <typeof(it) as TryIterator>::next(ref it) {   // each step is covered
        pat: T => body,                              // the body is not covered
        none   => break,
    }
}
```

`try for` 的 `try` 只涵蓋迭代：可迭代運算式以及每一次 `next`。它不涵蓋主體；主體中會擲回錯誤的呼叫要以它自己的 `try` 標示，與其他任何地方相同（[3.14.2](314-error-control-flow.md#涵蓋範圍)）。某一步的錯誤會送往迴圈的 `try` 所指之處：`try for` 送出函式，`try! for` 成為 panic，`try? for` 送到迴圈末端，此時迴圈的值是 `none`。迴圈之後的 `catch` 附加在整個 `try for` 運算式上（[4.14.10](../004-functions/414-throws.md#try-for)）。同樣的規則適用於走訪 `TryStream` 的 `try for await`（[stdlib 2.8.11](../../refs-standard-libraries/200-core/208-stream.md#2811-trystream可能失敗的串流)）。

存放在變數或變數欄位中的 `TryIterator` 會原地步進：上面的 `it` 就是那個位置，而不是複本，就像 `for await` 中的串流是原地輪詢的一樣（[4.13.5](../004-functions/413-stream.md#4135-for-await-形式)）。由 `match` 分支（或 `if let`）在這樣的位置上繫結的也是如此：繫結指稱該位置，而迴圈讓該位置停在它停下之處。

走訪存放在變數中之 `Iterator` 的一般 `for` 步進的是複本：迭代器的 `into_iter` 就是迭代器本身，而上面的 `it` 是一個新的變數。該變數保有自己的位置，因此 `let r = 0..3; for i in r { .. } for i in r { .. }` 會走訪 `r` 兩次。要從迴圈停下之處繼續，請手動步進迭代器（`while let x: T = it.next() { .. }`），它作用於變數本身。

`List` 的迭代器在每一步都讀取串列當下的樣子，因此改變其所走訪之串列長度的主體會改變迴圈：`push` 讓它繼續下去，`remove` 會略過移入被移除位置的元素。警告 lint `resize_while_iterating`（[6.2](../006-meta-system/602-built-in-attributes.md)）會回報主體中對被迭代變數呼叫 `push`、`pop`、`insert`、`remove`、`clear`、`truncate`、`drain`、`retain`、`dedup`、`extend` 或 `append` 的情況，除非同一個區塊中其後接著 `break`、`return` 或 `throw`。要改變串列，請走訪複本（`for x in xs.clone()`），或收集變更並在迴圈之後套用。

### 結果值

`for` 的結果值一律是 unit（理由與 `while` 相同），而以 `for` 為目標、帶有值的 `break` 是編譯錯誤（E0059）。

### 模式位置

只能使用[模式](311-patterns.md)中不可反駁的模式（名稱、萬用字元、元組解構等）。

```nabi
for x in 0..10 { ... }                    // simple name
for (i, v) in xs.enumerate() { ... }      // tuple destructuring
for Point { x, y } in points { ... }      // struct destructuring
```

可反駁的模式（例如比對 `enum` 變體）在 `for` 的模式位置會被拒絕（E0100），因為處理比對失敗會有歧義。需要這種形式時，請使用 `while let`，或結合 `for` 與 `if let`。

### `for ref`：元素作為位置

```nabi
struct Point { x: f64; y: f64; }

let ps: List<Point> = [Point { x: 1.0, y: 2.0 }, Point { x: 3.0, y: 4.0 }];
for ref p in ps {
    p.x = 0.0;                            // writes the element of ps
}
println(f"{ps[1].x}");                    // 0
```

一般的 `for` 繫結每個元素的複本：結構是值（[2.3.8](../002-types/203-struct.md#238-值語意)），因此對繫結的欄位寫入不會改變迴圈外的任何東西，而且在沒有任何東西讀取它時是編譯錯誤（E0057）。`for ref x in xs` 則把 `x` 繫結到每個元素本身作為位置。讀取 `x` 就是讀取元素；`x = v`、`x.f = v`、`x.f += v` 以及呼叫 `ref self` 方法（`x.m()`，[4.7](../004-functions/407-ref-keyword.md)）會寫入它。模式是單一名稱。

`xs` 是 `List`、`SliceMut` 或陣列。陣列寫入其存放之處，因此它必須是變數或欄位，而不是暫存值。其他任何可迭代的東西都是編譯錯誤（E0068）：它們的項目不是位置。迴圈等價於：

```
let v = xs;                    // a List: a SliceMut view of it; an array: xs itself
for i in 0..v.len() {          // the length is read once
    body                       // with x standing for v[i]
}
```

串列透過 `SliceMut` 視圖走訪，因此適用該視圖的規則（[stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt可寫入檢視)）：在主體中改變 `xs` 的大小（`push`、`remove`、`clear`……），若主體中看得到該改變以及之後對 `x` 的使用，就是編譯錯誤 `E0137`；否則（在被呼叫的函式內改變大小）會在下一次使用 `x` 時 panic。請收集要加入或移除的元素，在迴圈之後再改變 `xs`。

閉包或 `async` 區塊不能擷取 `x`；請先複製元素（`let v = x;`）。`if x is T` 不會收窄 `x`，而 `for await` 不能是 `for ref`。

## 3.13.4. 標籤：控制巢狀迴圈

```nabi
'outer: loop {
    for x in xs {
        if cond { break 'outer; }
        if other { continue 'outer; }
    }
}
```

迴圈之前可以加上 `'name:` 形式的標籤。`break 'name` 與 `continue 'name` 直接控制帶有該標籤的迴圈。

### 標籤語彙單元

標籤由一個單引號、一個識別字與一個冒號組成（`'outer:`）。單引號與開始[字元字面值](../001-lexical/108-character-literals.md)的是同一個語彙單元，但在與冒號結合的位置會被辨識為標籤。確切的切分規則是語彙規格的補充定義；本文件只規定語言層級的形式。

### `break label value`

`loop` 標籤可以與值結合。

```nabi
'outer: loop {
    for x in xs {
        if found(x) { break 'outer x; }
    }
}
```

`break 'outer x` 帶著值 x 離開標籤為 outer 的 `loop`。

## 3.13.5. `break` 與 `continue` 的預設行為

不帶標籤時，它們指向最內層的迴圈。

```nabi
loop {
    for x in xs {
        if cond { break; }       // exits the for
    }
    // continues here
}
```

## 3.13.6. 求值順序

每個迴圈依上述定義的意義求值。主體的運算式與陳述式依[區塊運算式](309-block-expressions.md)的順序求值，副作用也遵循該順序。

迴圈的終止檢查如下。`loop` 不做任何檢查，只由明確的 `break` 結束。`while` 在每次迭代開始時對 `cond` 求值。`for` 在每次迭代開始時呼叫 `Iterator::next`。

## 3.13.7. 發散的迴圈

無法以 `break`、`return`、`throw` 等方式終止的迴圈（例如內部沒有 `break`、也沒有往外離開之流程的 `loop { ... }`）被歸類為發散的。這種迴圈運算式的型別與任何型別相容。
