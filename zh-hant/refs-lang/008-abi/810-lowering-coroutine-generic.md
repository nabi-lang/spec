# 8.10. 降階：協程與泛型

本節定義[產生器](../004-functions/411-generators.md)、[非同步](../004-functions/412-async.md)與[串流](../004-functions/413-stream.md)的協程降階，以及[泛型](../002-types/210-generics.md)與[常數泛型](../002-types/211-const-generics.md)的單態化降階。

## 8.10.1. 狀態機

[協程](../007-runtime/708-coroutine.md)被降階為由編譯器建立的狀態機。

1. HIR：`yield` / `async` / `async yield` 區塊像閉包一樣被提升為主體函式。建立框架的運算式留在原區塊的位置。
2. MIR：主體函式被轉換成恢復函式。每個暫停點得到一個狀態編號，而跨越該點仍存活的區域變數被移入框架欄位。
3. 框架型別是 `(i32 state, locals..)`，並具有 ScanTable。框架經由一般的配置路徑建立（[8.9.9](809-lowering-control.md#899-gc-配置快速--慢速路徑)）。
4. 恢復函式的進入處依 `state` 分派：一串把它與每個狀態編號比較的雙向分支（8.9.3）。

## 8.10.2. 產生器（`yield fn`）

```nabi
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

概念上的恢復函式（`next`）是：

```
next(frame) -> i32 | none:
    switch frame.state:
      0: frame.i = 0; goto loop
      1: frame.i = frame.i + 1; goto loop
      2: return none
    loop:
      if frame.i < frame.n:
          frame.state = 1
          return frame.i          // yield i
      frame.state = 2
      return none
```

`count_up_to(n)` 配置框架，並回傳 `state = 0` 的包裝（`GenIter<i32>`）。

## 8.10.3. 非同步（`async fn`）

```nabi
async fn fetch() -> Data {
    let r = await op1();
    process(r)
}
```

```
poll(frame, cx) -> Poll<Data>:
    switch frame.state:
      0: frame.fut = op1(); frame.state = 1; goto await1
      1: goto await1
    await1:
      match frame.fut.poll(cx):
        Pending  => return Pending
        Ready(r) => frame.state = 2; return Ready(process(r))
```

`await` 把目標 future 存入框架並輪詢它。若為 Pending，它回傳 Pending，讓狀態保持原樣，下一次輪詢從同一點繼續。

## 8.10.4. 串流（`async yield fn`）

[串流](../004-functions/413-stream.md)結合了兩者。單一框架的狀態編號把 `yield` 點與 `await` 點一起計數。`poll_next` 在 `yield` 處回傳 `Ready(value)`，在 Pending 的 `await` 上回傳 `Pending`，在結尾回傳 `Ready(none)`。

## 8.10.5. 框架配置

每次協程呼叫都會配置一個框架。對於由其呼叫者執行到完成的協程，沒有專門的省略；從不逃逸的框架可以像任何配置一樣被移除（[7.2.12](../007-runtime/702-allocation.md#7212-配置消除)）。

## 8.10.6. 泛型單態化

[泛型](../002-types/210-generics.md)的單態化為呼叫處的每個型別組合產生個別的實例。

```nabi
fn id<T>(x: T) -> T { x }

let a = id(42i32);     // id<i32>
let b = id(3.14f64);   // id<f64>
```

單態化之後，程式中每個實例各有一個函式。

```
fn id<i32>(x: i32, tls) -> i32:
    return x

fn id<f64>(x: f64, tls) -> f64:
    return x

fn main(tls):
    a = call id<i32>(42, tls)
    b = call id<f64>(3.14, tls)
```

每個具體的型別組合都會建立一個個別的函式（下文中 `id<i32>` 代表其修飾後的符號）。

### 單態化何時發生

單態化的進行方式如下。

```
nabi-hir -> specialize(hir, type_tuple) -> nabi-mir -> machine code
```

在 [AOT](../000-meta/007-glossary.md#aot預先編譯) 模式下，先收集所有呼叫處，然後一次執行[特化](../000-meta/007-glossary.md#特化specialize)。在 [JIT](../000-meta/007-glossary.md#jit即時編譯) 模式下，特化在第一次呼叫時執行，其結果會被快取（見[執行引擎與使用者介面](../000-meta/001-execution-modes.md)）。兩種模式共用相同的特化機制。

### 符號命名

[編譯管線](../000-meta/002-pipeline.md)所使用的符號修飾規則是：

```
_Y<pkg_hash_8>_<path>_<name>_<typehash_16>
```

套件就是雜湊，因此位於套件根的函式其 `<path>` 為空：`id<i32>` 會變成類似 `_Y3f8a1c2b__id_a7c91d8e44b2f190` 的東西（示意），而這個符號就是該函式在目的檔中的名稱。非泛型的項目其 `<typehash>` 為空（`_Y3f8a1c2b__main_`）。識別字中的字面 `_` 會跳脫為 `_0`（[9.5](../009-tools/905-symbol-mangling.md)）。

## 8.10.7. `const` 泛型單態化

[常數泛型](../002-types/211-const-generics.md)的一個例子：

```nabi
fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }

let r = dot([1.0, 2.0, 3.0], [4.0, 5.0, 6.0]);  // T=f64, N=3
```

這被單態化為下列內容。

```
fn dot<f64, 3>(a: [f64; 3], b: [f64; 3], tls) -> f64:
    ; ... a loop of 3 iterations
```

由於 `const` N=3 是編譯期常數，迴圈上限與陣列大小在實例中都是常數：最佳化器會摺疊依賴它們的部分。

## 8.10.8. 泛型 trait 分派

```nabi
fn process<T: Describe>(x: T) -> str {
    x.describe()
}
```

具體型別 T 在呼叫處決定，因此對於 `T = Point`，降階結果是：

```
fn process<Point>(x: (f64, f64), tls) -> (ptr, usize):
    ; call Point's Describe::describe directly
    r = call Point.describe(x, tls)
    return r
```

`Point` 以其值結構傳遞（8.8.3），`str` 以其 `(ptr, usize)` 組合傳遞（8.8.1）。沒有經由 vtable 的間接呼叫；一切都是直接的函式呼叫。結合[禁止把 trait 當作型別](../002-types/212-trait-as-type-banned.md)，這個模型保證零 vtable 成本。

## 8.10.9. 和型別上的分支分派

對[和型別](../002-types/206-sum-types.md)的比對被降階為標籤測試，即一串雙向分支（8.9.3）。

```nabi
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),
    s: str => use_str(s),
}
```

在 64 位元目標上，`i32 | str` 是 `(i32, [u64; 2])`：標籤 0 是 `i32`，標籤 1 是 `str`，而酬載是兩個字組，因為 `str` 是容納 GC 參照的兩字組組合（8.8.6）。以區塊表示：

```
bb0:
    if v.0 == 0 then bb_use_int else bb_test_str
bb_test_str:
    if v.0 == 1 then bb_use_str else bb_unreachable

bb_use_int:
    n = v.1 as i32                 ; the payload read as i32
    call use_int(n, tls)
    goto bb_end

bb_use_str:
    s = v.1 as str                 ; the payload read as the (ptr, usize) pair
    call use_str(s, tls)
    goto bb_end

bb_end:
    ...
```

match 是窮盡的，因此 `bb_unreachable` 永遠不會到達。標籤檢查與分支全都是靜態程式碼。

## 8.10.10. trait 方法的 trait 約束分派

```nabi
fn add_all<T: Add<T, Output = T>>(items: List<T>) -> T {
    let r: T = ...;
    for x in items {
        r = r.add(x);
    }
    r
}
```

單態化之後，每個呼叫處的每個 T 都會建立一個個別的函式。對於 `T = i32`，`r.add(x)` 解析為對 i32 之 `Add::add` 的直接呼叫，它也可以被內嵌。

## 8.10.11. 總結

編譯器自己把協程降階為狀態機（8.10.1）。泛型降階透過單態化為每個具體型別產生個別的程式碼，不使用 vtable。

這兩種機制是本 [ABI](../000-meta/007-glossary.md#abi應用程式二進位介面) 部分中最複雜的降階；其餘的[降階：資料型別與運算式](808-lowering-types.md)與[降階：控制流程與 GC 整合](809-lowering-control.md)，都接近直接對應到機器運算。
