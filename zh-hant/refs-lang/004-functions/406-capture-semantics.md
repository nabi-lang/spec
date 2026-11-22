# 4.6. 擷取語意

[閉包](405-closure.md)可以擷取外部環境的變數。本章定義決定擷取是複製值還是持有參照的規則。

## 4.6.1. 預設：值複製

> [!IMPORTANT]
> 擷取的預設意義是**值複製**。建立閉包時，會取得外部變數之值的快照。

```nabi
let count = 0;
let f = || { count + 1 };         // the value 0 of count is copied
count = 10;
f();                               // 1 (f holds 0)
```

這個決定有三個動機。第一，明確性：擷取模式在程式碼中清楚可見。第二，阻止副作用：它防止閉包默默修改外部變數的陷阱。第三，對科學與數值程式碼友善：它避免了 Julia 中參照擷取被裝箱為動態槽位而使效能下降的 100 倍裝箱陷阱。

```
capture_list    ::= "[" capture_item ("," capture_item)* ","? "]"
capture_item    ::= "ref" capture_name | "move" capture_name | capture_name
capture_name    ::= identifier | "self"
```

`self` 指稱方法的接收者。`[ref self]` 只允許在 `ref self` 方法中使用，它在那裡共享接收者（[4.7.4](407-ref-keyword.md#474-closure-capture-lists)）；以值傳遞的 `self` 依值擷取。

## 4.6.2. 參照擷取：`[ref x]` 擷取清單

要擷取對外部變數之繫結的參照，請使用明確的擷取清單。

```nabi
let count = 0;
let inc = [ref count] || { count = count + 1 };
let dec = [ref count] || { count = count - 1 };

inc(); inc(); dec();
// count = 1
```

清單寫在閉包定義之前，形式為 `[ref var, ref var, ...]`。清單中的變數存放在共享儲存格中（裝箱）；外層作用域與閉包使用同一個儲存格，因此在閉包內重新指派會反映到外部變數。該儲存格是 GC 物件，因此閉包可以比變數的作用域存活得更久。不在清單中的變數照常依值擷取。

### 多個與混合的擷取

```nabi
let counter = 0;
let config = load_config();

let handler = [ref counter] || {
    counter = counter + 1;        // ref: reflected in the outer variable
    print(config.version);         // value: snapshot of config at capture time
};
```

只有在擷取清單中指名的變數是 `ref` 擷取；其餘的自動依值擷取。

## 4.6.3. 擷取清單的切分

`[ref x] || body` 的切分如下：`[`、`ref`、`x`、`]` 構成方括號中的擷取清單，而 `||` 或 `|x|` 標示閉包的開始。

歧義由[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)的最長匹配規則與上下文解決。若 `[...]` 之後緊接著 `||` 或 `|x|`，它就是擷取清單；否則它是一般的索引或陣列字面值。

```nabi
let xs = [1, 2, 3];               // List literal
let f = [ref count] || { ... };   // capture list + closure
arr[ref count]                     // ERROR (E0010): ref cannot be used in indexing
```

## 4.6.4. 在主體中重新指派外部變數 → 需要 `ref`

試圖在閉包主體中重新指派一個依值擷取的外部變數，是編譯錯誤（E0057）。

```nabi
let count = 0;
let f = || { count = count + 1 };
// ERROR (E0057): `count` is value-captured; to mutate the outer variable,
//                add `[ref count]` to the closure's capture list
```

這條規則引導使用者刻意寫出 `ref` 擷取。

## 4.6.5. `ref` 簡寫（無）

```nabi
ref || { ... }                    // not adopted
```

沒有把所有東西都以 `ref` 擷取的簡寫。擷取只能透過明確的清單成為 `ref`，讓程式碼清楚顯示哪些變數是以參照持有的。

## 4.6.6. 迴圈變數的自動處理

由於值擷取是預設，每次迭代的迴圈變數都保有自己的值。

```nabi
let fns: List<fn() -> i32> = [];
for i in 0i32..5i32 {
    fns.push(|| i);               // copies the value of i in each iteration
}
fns[0]();                          // 0
fns[4]();                          // 4
```

不需要像 Go 1.22 那樣的每次迭代作用域的另外規則；值擷取自動產生預期的意義。

## 4.6.7. 擷取結構與參照控制代碼

值擷取把值複製進閉包。[`struct`](../002-types/203-struct.md) 是值型別，因此閉包持有它自己的複本；閉包內的欄位寫入不會到達外部變數。要那樣做請使用 `[ref p]`。對值擷取進行欄位寫入是編譯錯誤（E0057），與重新繫結擷取相同，其說明為 ``add `[ref p]` to the closure's capture list``。對單純資料（數字、`bool`、`str`，以及由它們組成、沒有 `Drop` 的結構、元組、陣列與列舉）的值擷取或其欄位呼叫 `ref self` 方法也是如此：閉包的每次呼叫都從它自己的複本開始，因此變更會遺失（以帶種子的產生器 `g` 執行 `List::from_fn(4, |_| g.uniform(0.0, 1.0))` 會得到同一個值四次）。共享其狀態的控制代碼與資源（`Cell`、`File`、通訊端）不受影響。請寫 `[ref g] |_| g.uniform(0.0, 1.0)`；若只要在閉包內改變一份複本，請先把它複製到區域變數中。

集合型別（`List`、`Dict`、`Set`、`StrBuf`……）是參照控制代碼（[stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#reference-handles)）。值擷取複製的是控制代碼，因此物件是共享的。

```nabi
let xs = [1, 2, 3];
let f = || { xs.push(4) };        // copies the handle (same List)
f();
// xs == [1, 2, 3, 4]

let p = Point { x: 0.0, y: 0.0 };
let g = [ref p] || { p.x = 1.0 }; // ref capture: writes the outer p
g();
// p.x == 1.0
```

要得到集合的獨立複本，請呼叫 `.clone()`。

## 4.6.8. `ref` 擷取與物件變更

```nabi
let xs = [1, 2, 3];

// value capture: mutating the object is OK, reassigning the binding is not
let f = || {
    xs.push(4);                   // OK: object mutation
    // xs = [];                    // ERROR: binding reassignment
};

// ref capture: reassigning the binding is also allowed
let g = [ref xs] || {
    xs = [];                      // OK: the outer xs becomes []
};
```

對參照控制代碼而言，值擷取與物件變更可以並行不悖，因為物件是共享的。`ref` 擷取用於必須重新指派繫結本身的時候。

這種區分與 [`impl`](../002-types/209-impl.md) 的 `self` 與 `ref self` 模型相同。

## 4.6.9. 平行與並行上下文中的額外限制

`fn() + Send` 閉包（傳給 `task::spawn_blocking`、`par::for_each`、`thread::spawn` 等的閉包）的每個一般擷取都必須是 `Send + Sync`（[4.4](404-function-type.md#send--sync-bounds)）：閉包與建立它的程式碼共享它。`[move x]` 擷取只屬於閉包，只需要 `Send`（[4.6.11](#4611-move-capture-the-move-x-capture-list)）；`+ Sync` 閉包（`par::for_each`，由每個工作者呼叫）需要每個擷取都是 `Sync`。`[ref x]` 儲存格會被閉包與建立它的程式碼同時寫入，因此無論 `x` 的型別為何，它都絕不是 `Send` 或 `Sync`：帶有 `[ref]` 擷取的閉包（或擷取了這種閉包的閉包，或由 `async [ref x] ||` 閉包產生的 future）不適合任何 `+ Send` / `+ Sync` 槽位。同一執行緒內的使用（`spawn_local`、一般呼叫）不受影響。集合型別是 `!Sync`（[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)），也不能以這種方式依值擷取。

```nabi
let buf: List<i32> = [];
par::for_each(xs, [ref buf] |x| { buf.push(x) });
// ERROR (E0085): a [ref buf] cell cannot cross threads
```

請改用摺疊（`par::fold(xs, 0, |a, b| a + b)`），透過 `Mutex` / `RwLock` / `Atomic` / 通道共享，或從執行緒回傳值（`join()`）。只有被產生的程式碼需要的值，要移入：`thread::spawn([move buf] || buf.len())`。

以這種方式共享的值屬於其新的擁有者：在 `Mutex::new(buf)` 或 `tx.send(buf)` 之後使用 `buf` 是編譯錯誤（E0105，[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)）。請透過鎖使用它，或交出 `buf.clone()`。

```nabi
let m = Mutex::new(buf);
buf.push(1);
// ERROR (E0105): `buf` was moved into the `Mutex` at line 1
//   help: use it through the lock: using g = m.lock() { g.get().push(1) }
```

詳細的平行與並行規則由標準函式庫規格定義；本章只涵蓋語言層級的擷取機制。

被擷取的 `Drop` 資源（`File`、通訊端、通道的 `Sender`）不會被複製：擷取把它移入閉包或 `async` 區塊，之後在外部使用它是錯誤（[stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-release)）。`async` 區塊的任務在結束時釋放它所擷取的、釋放不會失敗的值（`Sender`）；關閉可能失敗的被擷取值（`File`）必須在區塊內關閉（`try f.close()` 或 `using`），否則該區塊是錯誤（E0134）。若外部也要保留該值，請擷取一份複本：`x.clone()`，對 `File` 則是 `try x.try_clone()`。

## 4.6.10. 被擷取變數的生命週期

被擷取的變數與閉包值存活得一樣久。即使閉包離開了外層作用域，被擷取的值仍保存在閉包物件內。

```nabi
fn make_counter() -> fn() -> i32 {
    let count = 0;
    [ref count] || { count = count + 1; count }
    // count is captured by ref; kept until the closure goes away
}

let c = make_counter();
c();                               // 1
c();                               // 2
c();                               // 3
```

在這個範例中，外層函式回傳之後，`count` 仍附著在閉包物件上而存活。[GC](../000-meta/007-glossary.md#gc-garbage-collector) 管理這個生命週期（[7. 執行環境模型](../007-runtime/README.md)）。

## 4.6.11. 移動擷取：`[move x]` 擷取清單

`[move x]` 在建立閉包時把 `x` 交給閉包。它是 `[ref x]` 的對應物：`ref` 擷取與建立閉包的程式碼共享繫結，`move` 擷取則讓那段程式碼失去它。

```nabi
let rows: List<str> = load();
let t = thread::spawn([move rows] || rows.len());
rows.push("x");
// ERROR (E0105): `rows` was moved into a closure at line 2
//   help: the closure owns it now; use it inside the closure, ...
```

- 建立閉包時交出 `x` 的方式與通道傳送相同（[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)）：閉包得到 `x` 所能到達的每個集合的複本，之後使用 `x`（或指向其所容納之物的其他控制代碼）是編譯錯誤（`E0105`）。`Sync` 值（數字、`str`、由它們組成的陣列）豁免：閉包的複本與 `x` 相等，兩者都可以使用。
- 移動擷取只需要 `Send` 就能讓閉包跨越執行緒：沒有其他程式碼能存取它。因此集合、`Command` 或 `Child` 可以交給 `thread::spawn`、`task::spawn_blocking` 或 `par::Scope::spawn`。一般擷取保持 [4.6.9](#469-additional-constraints-in-parallel-and-concurrent-contexts) 的 `Send + Sync` 規則，而 `+ Sync` 槽位（`par::for_each`，由每個工作者呼叫）需要每個擷取都是 `Sync`。
- 擁有非 `Sync` 移動擷取的閉包只能被交出一次。把它傳給型別為 `fn(..) + Send` 但沒有 `Sync` 的參數就會交出它，無論是哪個函式：這樣的參數是被呼叫者可能在另一個執行緒上執行的參數（`thread::spawn`、`thread::Builder::spawn`、`task::spawn_blocking`、`task::spawn_blocking_with_arg` 的 `f`、`par::join`、`par::Scope::spawn`、`par::TryScope::spawn`，以及程式自己帶有這種參數的任何函式）。之後再次使用它，無論它經由什麼途徑傳遞（變數、結構或元組欄位、串列元素、參數、`if` / `match` 分支、回傳其引數之函式的結果），都是編譯錯誤（`E0133`）；在一次呼叫中傳遞它兩次也是（`par::join(f, f)`）。型別沒有寫出 `Sync` 的 `fn(..) + Send` 值可能擁有這樣的擷取，因此也只能交出一次；`+ Send + Sync` 值以及擷取全都是 `Send + Sync` 的閉包可以再次傳遞。呼叫閉包不會交出它：它可以被呼叫任意多次。要在其他執行緒上執行同樣的工作兩次，請建立兩個閉包，各自移入自己的 `.clone()`。
- 呼叫 `async [move ..]` 閉包也不會交出它，但呼叫所產生的 future 是在閉包移入的東西上執行的：閉包及其每個 future 是同一個擁有者。它的 future 可以在這裡一個接一個地 await（`await f(); await f();`）；把閉包或其中一個 future 交出去（交給 `task::spawn`、放進另一個閉包的 `[move f]`）會把其他的一起帶走，之後使用它們是編譯錯誤（`E0105` / `E0133`）。

```nabi
let xs: List<i32> = [];
let f = async [move xs] || { xs.push(1); xs.len() };
let a = await f();                 // OK: runs here
let t = task::spawn(f());          // the task owns f's moved values now
let b = await f();
// ERROR (E0133): a future of `f` was spawned as a task at line 4, and every future
//   of an `async [move ..]` closure shares what it moved in
```
- 必須關閉的值（`File`、通訊端）就像一般擷取那樣移入閉包（[4.6.9](#469-additional-constraints-in-parallel-and-concurrent-contexts)）；由閉包關閉它。
- `move` 與 `ref` 可以在一個清單中混用（`[move xs, ref n]`）；同一個名稱不能兩者都是。清單中主體沒有使用的名稱不會被擷取。

```nabi
let xs: List<i32> = [1, 2];
let n = 0;
let f = [move xs, ref n] || { n += 1; xs.push(n) };   // OK, on this thread
let g = [ref n, move n] || n;                    // ERROR (E0058): both ref and move
```

同一個值也可以改為以引數傳遞：`task::spawn_blocking_with_arg(xs, |xs| xs.len())`（[stdlib 2.18.1](../../refs-standard-libraries/200-core/218-task.md#2181-core-api)）。
