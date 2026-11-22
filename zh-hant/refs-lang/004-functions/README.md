# 4. 函式概觀

本章定義函式及其變體：閉包、方法與協程。本文件在詳細規則之前先給出整體圖像。

## 目錄

- [4.1. 函式宣告](401-function-declaration.md)
- [4.2. 參數](402-parameters.md)
- [4.3. 回傳](403-return.md)
- [4.4. 函式型別](404-function-type.md)
- [4.5. 閉包](405-closure.md)
- [4.6. 擷取語意](406-capture-semantics.md)
- [4.7. `ref` 關鍵字](407-ref-keyword.md)
- [4.8. 尾隨閉包](408-trailing-closure.md)
- [4.9. 方法](409-method.md)
- [4.10. 巢狀函式](410-nested-functions.md)
- [4.11. 產生器（`yield`）](411-generators.md)
- [4.12. 非同步（`async` 與 `await`）](412-async.md)
- [4.13. 串流（`async yield`）](413-stream.md)
- [4.14. `throws` 簽章](414-throws.md)
- [4.15. `unsafe`（函式修飾詞、區塊運算式與未定義行為）](415-unsafe.md)
- [4.16. 進入點（`main.nabi`、`fn main`、頂層陳述式）](416-entry-point.md)

## 函式的角色

函式是接受輸入值並產生結果值的程式碼單位。Nabi 的函式可以具有下列所有特性。

依命名：有寫成 `fn name(...)` 的具名函式，以及寫成 `|...|` 的匿名閉包。依型別標註：函式是靜態的或漸進式的。若寫出參數的型別，就是靜態的；若在私有或 `pub(sub)` 函式上省略，就會在呼叫處進行泛型單態化（[`any` 槽位](../002-types/202-any-slot.md)、[泛型](../002-types/210-generics.md)）。`pub` 或 `pub(pkg)` 函式要寫出其參數型別（E0138），但 trait impl 的方法除外，它從 trait 取得參數型別（[4.2.2](402-parameters.md#422-omitting-the-type)）。依執行模型：同步、非同步與產生器形式都是可能的：一般函式、`async fn`（回傳 Future）、`yield fn`（回傳 Iterator）與 `async yield fn`（回傳 Stream）。依例外行為：函式要麼能擲回錯誤，要麼不能。在簽章中放上 `throws` 就允許它擲回錯誤（[錯誤控制流程](../003-expressions/314-error-control-flow.md)）。最後，依歸屬：函式要麼是定義在 `impl` 區塊中、附加在型別上的方法，要麼是模組層級的自由函式。

## 函式定義的兩種形式

函式以兩種形式定義。

### 具名函式

第一種是有名稱的函式。

```nabi
fn area(r: f64) -> f64 {
    3.14159 * r * r
}
```

這種形式由 `fn` 關鍵字、名稱、參數、回傳型別與主體組成。它可以定義在模組層級，或定義在 [`impl`](../002-types/209-impl.md) 區塊內。

### 匿名閉包

第二種是沒有名稱的閉包。

```nabi
|x: f64| -> f64 { 3.14159 * x * x }
|x| x * 2
|| compute()
```

這種形式把參數清單以 `|...|` 括起來，其後接主體。它直接出現在運算式位置，可以指派給變數，或直接作為函式引數傳遞。

兩種形式由相同的內部機制處理。唯一的差異在於是否有名稱，以及建立閉包時是否擷取環境。

## 呼叫

函式與閉包都以 `()` 呼叫。

```nabi
let a = area(2.0f64);
let f = |x: f64| x * 2;
let b = f(3.0f64);
```

呼叫時，引數與參數配對。確切的配對規則定義於[參數](402-parameters.md)。

## 回傳

主體最後一個運算式的值成為函式的回傳值。函式主體遵循[區塊運算式](../003-expressions/309-block-expressions.md)的區塊規則。

```nabi
fn add(x: i32, y: i32) -> i32 { x + y }
fn early(x: i32) -> i32 {
    if x < 0 { return 0; }     // explicit return for early exit
    x * 2
}
```

因此 `return` 關鍵字不是必要的；最後一個運算式會自動回傳。詳細規則見[回傳](403-return.md)。

## 函式與運算式

函式與閉包是值。它們可以指派給變數、作為引數傳給其他函式，以及存放在集合中。

```nabi
let f = |x| x * 2;
let v = list.iter().map(f);
let g = if cond { f } else { |x| x + 1 };
```

函式型別定義於[函式型別](404-function-type.md)，由參數型別與回傳型別組成。

## 呼叫的分派

依照 [trait 不能作為型別](../002-types/212-trait-as-type-banned.md)，函式呼叫的分派不經 vtable 間接即可解析。

具名函式在靜態決定的函式位址上直接呼叫。閉包是擷取的環境與程式碼指標的組合；呼叫是把擷取的環境作為第一個引數傳入的直接呼叫。泛型函式透過[泛型](../002-types/210-generics.md)的單態化，在每個呼叫處解析為直接呼叫。

因此函式與閉包的呼叫原則上都是直接呼叫，可以被內嵌。其效能等同於一般的方法呼叫。
