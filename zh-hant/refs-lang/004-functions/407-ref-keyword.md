# 4.7. `ref` 關鍵字

`ref` 是標示「這個位置的變更會反映到呼叫者（或外部）的變數」的關鍵字。對參數與呼叫引數而言，它以 copy-in / copy-out 實作；在擷取清單中，則以共享儲存格實作。它不是一等的參照值。

## 4.7.1. 三個位置

```
1. function parameter:      fn f(ref x: i32) { x = ... }
2. closure capture list:    [ref x] before |...| { ... }
3. at the call:             f(ref x)
```

在這三個位置中，它都有相同的意義：變更會反映到外部變數。`for ref x in xs` 以相同方式繫結串列或陣列的每個元素：變更會反映到該元素（[3.13.3](../003-expressions/313-loop-expressions.md#for-ref元素作為位置)）。

## 4.7.2. 函式參數

```nabi
fn increment(ref n: i32) {
    n = n + 1;
}

fn swap(ref a: i32, ref b: i32) {
    (a, b) = (b, a);
}
```

在參數名稱之前加上 `ref`，在主體中重新指派該參數就會反映到呼叫者的變數。參數本身的型別（上例中的 `i32`）與平常相同。

### 意義：copy-in / copy-out

`ref` 參數是 copy-in / copy-out。呼叫時，呼叫者位置（一個變數，或變數的欄位）的值被複製到參數中；被呼叫者使用該值工作；呼叫結束時，參數的最終值被寫回呼叫者的位置。

- **寫回時機**：寫回不只發生在正常回傳時，也發生在被呼叫者擲回錯誤時。因此在 `throw` 之前所做的變更，呼叫者看得到。
- **排他性**：呼叫進行期間，不得經由其他路徑存取同一個位置，編譯器會檢查這一點。把同一個位置以 `ref` 傳遞兩次，或在一個引數中以 `ref` 傳遞而在另一個引數中讀取它，是編譯錯誤（E0050）。不同的欄位（`ref p.x`、`ref p.y`）不重疊，是允許的。
- **位置**：引數必須指稱一個變數或變數的欄位。元素（`inc(ref xs[2])`）是編譯錯誤：`` `ref` argument `n` of `inc` must name a variable or a field of one ``。請把元素複製到變數中，傳遞該變數，再把它存回去。`ref self` 接收者也可以是元素或 `Dict` 項目（[見下文](#ref-self-的意義copy-in--copy-out)）。

```nabi
fn bump_then_fail(ref n: i32) throws {
    n = n + 1;
    throw MyError {};
}
let a = 1i32;
try bump_then_fail(ref a) catch _ { };   // a = 2 (written back on throw too)
swap(ref a, ref a);                       // ERROR (E0050): same place passed as ref twice
add(ref a, a);                            // ERROR (E0050): place passed as ref read by another argument
swap(ref p.x, ref p.y);                   // OK: distinct fields
```

> [!IMPORTANT]
> 呼叫者必須在呼叫處寫出 `ref`；不寫就呼叫是編譯錯誤。這讓哪些變數可能改變一目了然。

## 4.7.3. 呼叫時明確寫出（C# 風格）

```nabi
let x = 5i32;
let y = 10i32;
increment(ref x);                 // x = 6
swap(ref x, ref y);
```

只有在函式參數宣告為 `ref` 之處，呼叫者也必須寫 `ref`。省略 `ref`，或為非 `ref` 參數寫 `ref`，是編譯錯誤（E0050）。以路徑呼叫 `ref self` 方法時（`Counter::increment(ref c)`，4.7.5）也適用相同規則。

這條規則讓哪些變數可能改變在呼叫處一目了然，動機與 C# 的 `ref` 模式相同。

```nabi
fn read(x: i32) { ... }
fn write(ref x: i32) { ... }

read(a);                          // plain: clearly a does not change
write(ref a);                      // ref: clearly a may change
```

## 4.7.4. 閉包擷取清單

```nabi
let count = 0;
let inc = [ref count] || { count = count + 1 };
```

依照[擷取語意](406-capture-semantics.md)中定義的 `ref` 擷取，擷取清單中的 `ref` 變數存放在共享儲存格中（裝箱）。外層作用域與閉包讀寫同一個儲存格，因此閉包可以比變數的作用域存活得更久。

`ref` 參數（包括 `ref self`）以相同方式擷取（`[ref n]`、`[ref self]`）。函式主體與閉包共享該參數的唯一儲存格，因此透過閉包所做的寫入是參數最終值的一部分，並會被寫回呼叫者（[4.7.2](#472-函式參數)）。閉包在呼叫結束之後所做的寫入不會到達呼叫者。

```nabi
fn bump(ref c: Counter) {
    let add = [ref c] |x: i32| { c.count = c.count + x; };
    add(5);
}
let c = Counter { count: 1 };
bump(ref c);                      // c.count = 6
```

## 4.7.5. 方法接收者的例外

[`impl`](../002-types/209-impl.md) 中的方法可以有 `ref self` 參數。

```nabi
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}
```

此時呼叫處不需要 `ref`。

```nabi
let c = Counter { count: 0 };
c.increment();                    // no 'ref': handled automatically for method receivers
```

這是方法接收者專用的規則，與一般的 `ref` 參數不同。方法呼叫語法本身就是自動[分派](../000-meta/007-glossary.md#分派dispatch)接收者的機制，因此額外的標記只會是視覺上的雜訊。

```nabi
fn external_inc(ref c: Counter) { ... }
external_inc(ref c);              // ordinary function calls require 'ref'
Counter::increment(ref c);        // calling a ref self method by path also requires ref
```

### `ref self` 的意義：copy-in / copy-out

`ref self` 的行為是 copy-in / copy-out。呼叫時，接收者的值被複製到 `self` 中；呼叫結束時，`self` 的最終值被寫回接收者的位置。

- **排他性**：呼叫進行期間，不得經由其他路徑（另一個引數、擷取它的閉包等）存取接收者變數。若編譯器看得到這一點，就是編譯錯誤（類似 Swift 的排他性規則）。
- **暫存接收者**：若接收者是暫存值（`make().increment()`），寫回的結果會被捨棄。
- **元素與項目**：元素（`cs[0].increment()`，也包括欄位之下：`cs[0].pos.shift(1)`）或 `Dict` 項目（`d["a"].increment()`）透過索引寫回：元素被複製出來，方法在複本上執行，然後複本被存回去（`cs[0] = c`），在 `throw` 之後也是如此。索引只求值一次。型別沒有索引存入（只有 `Index`）的元素是暫存值。
- **靜態變數與常數**：在 `static`（[5.3](../005-modules/503-static.md)）或 `const`（[5.2](../005-modules/502-const.md)）上呼叫 `ref self` 方法是編譯錯誤（E0057）：``cannot call `increment` on `S`: it takes `ref self`, and a static is never written``；常數的每次使用都是新的值。請把它複製到變數中，再在那裡呼叫方法。

```nabi
c.increment();                     // copies c in, writes back to c at the end
c.add_from(|| c.count);            // ERROR (E0050): c accessed through a closure during the call
```

## 4.7.6. `ref` 與型別

`ref` 是修飾詞而不是型別，不出現在[型別運算式](../002-types/201-type-expressions.md)的任何地方。

```nabi
let x: ref i32 = ...;             // ERROR: ref is not a type
struct S { x: ref i32; }          // ERROR: same reason
List<ref i32>                      // ERROR: same reason
```

因此 `ref` 只能出現在：

- 函式與方法的參數宣告。
- 閉包擷取清單。
- 函式與方法的呼叫引數。

由於 `ref` 不能用在回傳型別中，Nabi 沒有回傳參照的函式（沒有 Rust 的 `deref_mut` 之類的形式）。改變內部值的 API 以交換值的方式運作，例如 `set` / `update(f)`。

## 4.7.7. 參照不能是一等值

```nabi
let r = ref x;                    // ERROR: ref itself is not an expression
let r: SomePointerType = ref x;   // ERROR
```

`ref` 不是產生值的運算式；它只是附加在呼叫、參數與擷取清單上的修飾詞語彙單元。

協程函式（`yield fn`、`async fn`、`async yield fn`）不能有 `ref` 參數；那樣做是編譯錯誤（E0023）。主體在呼叫回傳之後還會繼續執行，因此沒有可以寫回的時間點。

唯一的例外是 `async` 方法上的 `ref self`（[4.12.13](412-async.md#41213-方法上的-async)）：future 持有接收者，寫回在 future 完成時發生。這樣的呼叫要在其所在之處 await（`await r.read_one()`）。

出於同樣的理由，帶有 `ref` 參數的函式不能用作函式值（`let f = increment;` 是錯誤），閉包也不能有 `ref` 參數。函式型別 `fn(..)` 沒有表達 `ref` 的位置。

若需要一等的參照（例如 [FFI](../000-meta/007-glossary.md#ffi外部函式介面)），請使用[型別運算式](../002-types/201-type-expressions.md)的原始指標型別（`Ptr<T>` 與 `ConstPtr<T>`），它們屬於[標準函式庫層級](../000-meta/005-standard-libraries-tiers.md)的 `unsafe` 領域。

## 4.7.8. 總結：依值傳遞與依 `ref` 傳遞

```nabi
fn by_value(x: i32) { x = 10i32; }
fn by_ref(ref x: i32) { x = 10i32; }

let a = 5i32;
by_value(a);                      // a = 5 (unchanged)
by_ref(ref a);                    // a = 10
```

| 面向                                     | 依值傳遞（預設）        | 依 `ref` 傳遞（`ref`）|
|------------------------------------------|-------------------------|-----------------------|
| 在參數中重新指派的效果                   | 只影響區域              | 影響呼叫者的變數      |
| 呼叫時的明確語彙單元                     | 無                      | 需要 `ref`            |
| 視覺訊號                                 | 一般呼叫                | 標示可能的變更        |

## 4.7.9. 結構與參照控制代碼

`struct` 是值型別：依值傳遞會複製它，因此被呼叫者中的欄位寫入只影響區域。要改變呼叫者的結構，請以 `ref`（或 `ref self`）接收它，或回傳新的值。參照控制代碼（`List`、`Dict`……，[stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#reference-handles)）以控制代碼的身分被複製，因此對其元素的變更呼叫者看得到。

```nabi
fn set_x(p: Point) { p.x = 10.0; }          // local copy: the caller's p is unchanged
fn set_x_ref(ref p: Point) { p.x = 10.0; }  // written back to the caller's p
fn fill(xs: List<i32>) { xs.push(1); }      // same List: the caller sees it
fn reset(ref xs: List<i32>) { xs = []; }    // the caller's variable holds a new List
```

| 情境                                              | 需要的關鍵字                    |
|---------------------------------------------------|---------------------------------|
| 改變呼叫者之結構的欄位                            | `ref`                           |
| 改變集合的元素                                    | 一般的依值傳遞就足夠            |
| 讓呼叫者的變數容納另一個值                        | `ref`                           |

## 4.7.10. 同時對兩個繫結使用 `ref`

```nabi
fn swap(ref a: i32, ref b: i32) {
    (a, b) = (b, a);
}

let x = 1i32;
let y = 2i32;
swap(ref x, ref y);               // x = 2, y = 1
```

如上所示，一次呼叫可以有多個 `ref` 參數。

依 4.7.2 的排他性規則，在呼叫處把同一個位置作為兩個 `ref` 傳遞是編譯錯誤。

```nabi
swap(ref x, ref x);               // ERROR (E0050): same place passed as ref twice
```
