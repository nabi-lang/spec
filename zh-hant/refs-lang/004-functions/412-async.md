# 4.12. 非同步（`async` 與 `await`）

`async` 與 `await` 關鍵字（[關鍵字](../001-lexical/105-keywords.md)）是定義非同步計算（承諾未來會有值的函式）並等待其結果的機制。

## 4.12.1. 三種定義形式

非同步定義有三種文法形式。

```
async_fn        ::= "async" fn_decl
async_block     ::= "async" block
async_closure   ::= "async" closure_expr
await_expr      ::= "await" unary_expr
```

```nabi
async fn name(args) -> R { ... await expr; ... }    // named async function
async { ... await expr; ... }                        // inline block (value)
async || { ... await expr; ... }                     // closure modifier
```

在三種形式中，結果都是函式庫型別 `AsyncFuture<T>`（實作 `Future<Output = T>`）的值，帶有 `throws (E)` 時則是 `TryAsyncFuture<T, E>`（[4.1.11](401-function-declaration.md#result-types-of-coroutine-functions)）。這個結構與[產生器](411-generators.md)的 `yield` 以相同三種形式使用的模式一致。

## 4.12.2. `await` 運算子

`await` 是前綴一元運算子（[3.2.3](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table)，prec=13a），套用於實作了 `Future<Output = T>` 的值；其結果是 `T`。`await f() == x` 是 `(await f()) == x`。

```nabi
await expr
```

它暫停直到 Future 完成，然後回傳結果值。`await` 本身是運算式，結果值是該運算子的求值結果。

```nabi
let data = await fetch_data();
let processed = await transform(data);
```

`await` 只允許出現在 `async` 函式、`async` 區塊與 `async` 閉包的主體內。在一般函式主體中使用 `await` 是編譯錯誤。

## 4.12.3. 具名 `async` 函式

非同步函式以 `async fn` 宣告。

```nabi
async fn fetch_user(id: i64) -> User {
    let response = await http_get(f"/users/{id}");
    await parse_user(response)
}
```

在主體內，`await` 等待其他非同步結果；`-> R` 位置給出完成時所產生之值的型別。呼叫的結果是 `AsyncFuture<R>`。`-> R` 可以省略；此時 `R` 遵循 [4.1.3](401-function-declaration.md#413-omitting-the-return-type)：由主體推論，或在 `pub` 或 `pub(pkg)` 函式上為 `()`。

```nabi
let f = fetch_user(42i64);        // f: `AsyncFuture<User>`
let u = await f;                   // waits for the result
```

呼叫本身立即回傳；那一刻所建立的只是一個 **Future** 物件。實際的工作在 `await` 時開始，或者排程器也可能更早開始它。

## 4.12.4. 行內 `async` 區塊：值

`async` 區塊在運算式位置立即建立一個 `AsyncFuture<T>`。

```nabi
let f = async {
    let a = await op1();
    let b = await op2();
    a + b
};

let r = await f;
```

主體是[區塊運算式](../003-expressions/309-block-expressions.md)。這種形式用於以匿名、行內方式撰寫簡短的非同步工作。

和閉包一樣，`async` 區塊**在建立時以複製的方式**擷取外部的值（[4.6](406-capture-semantics.md)）。區塊建立之後改變外部變數，不會改變 future 內的值，而 future 內的重新指派在外部也看不到。

```nabi
let n = 1;
let f = async { n * 10 };
n = 2;
await f;                       //→ 10 (n at creation time)
```

## 4.12.5. `async` 閉包

`async ||` 建立一個可呼叫的非同步閉包。

```nabi
let task = async || {
    let data = await fetch();
    process(data)
};

let f1 = task();                   // f1: `AsyncFuture<...>`
let f2 = task();                   // a new Future each time
```

每次呼叫都建立一個新的 **Future**。帶有 `[ref x]` 擷取清單的 `async [ref x] || { .. }`，其 `x` 的共享儲存格由外層作用域與該閉包建立的每個 future 一起使用。

```nabi
let hits = 0;
let bump = async [ref hits] || { hits += 1; };
await bump();
await bump();
hits;                          //→ 2
```

## 4.12.6. 三種形式的對應

三種形式的對應如下。

```
plain        fn f() { ... }       ↔  || { ... }       ↔  { ... }
yield form   yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
async form   async fn f() { ... } ↔  async || { ... } ↔  async { ... }
```

修飾詞 `yield` 與 `async` 附加在相同的函式、區塊與閉包形式上，一致地把它們變成產生器或 **Future**。

## 4.12.7. Future trait

非同步函式、區塊或閉包的結果是標準函式庫的 `AsyncFuture<T>`（實作 `Future<Output = T>`）。確切的 trait 定義以及 `then`、`map` 與 `join` 等方法定義於標準函式庫。

## 4.12.8. `await` 的語意

下面的範例展示主體如何求值。

```nabi
async fn f() -> i32 {
    let a = await op1();
    let b = await op2();
    a + b
}
```

首先，`op1()` 被呼叫並回傳一個 **Future**。接著 `await` 等待該 Future 完成；在它完成之前，目前的函式 `f` 也被暫停。Future 完成時，其結果被繫結到 `a`，求值繼續。`op2()` 與第二個 `await` 遵循相同的模式，所有 `await` 都結束之後，`a + b` 被求值並作為函式的結果回傳。

這種暫停與繼續運作在與[產生器（`yield`）](411-generators.md)相同的協程機制上。

## 4.12.9. 並行執行：函式庫函式

要並行推進多個 Future，請使用標準函式庫函式。

```nabi
let rs = await task::join([task::spawn(op1()), task::spawn(op2())]);   // all: List<T>
let first = await task::race([task::spawn(op1()), task::spawn(op2())]); // whichever finishes first
let r = try await task::timeout(d, task::spawn(op1()));                // TimeoutError on timeout
```

`join` / `race` 接受 `List<Task<T>>`，其結果要被 `await`（[stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)）。

依照[語言與標準函式庫的邊界](../000-meta/004-language-vs-standard-libraries.md)的原則，`join`、`race`、`timeout`、`select` 等是函式庫函式，而不是語言關鍵字。它們確切的簽章定義於標準函式庫。

## 4.12.10. 允許 `await` 的位置

允許 `await` 的位置如下：

| 位置                         | 是否允許 `await`                     |
|------------------------------|--------------------------------------|
| `async fn` 主體              | 可以                                 |
| `async { ... }` 主體         | 可以                                 |
| `async \|...\| { ... }` 主體 | 可以                                 |
| 一般 `fn` 主體               | 錯誤                                 |
| 一般 `{ ... }` 區塊          | 錯誤                                 |
| `yield` 函式或區塊主體       | 錯誤（但 `async yield` 可以）        |

## 4.12.11. 非同步函式呼叫立即回傳

由於非同步函式呼叫會立即回傳，下列模式是可能的。

```nabi
let f = fetch_data();             // returns immediately; work has started or is scheduled
do_other_work();                   // do other work in parallel
let data = await f;                // wait when the result is needed
```

這是高效並行的基本模式：耗時的工作在需要其結果時才等待，而不是在呼叫之後立刻等待。

## 4.12.12. 與任務的關係

要把非同步工作作為獨立的執行單位啟動，請使用標準函式庫的 `task::spawn`。

```nabi
let handle = task::spawn(async { compute() });       // runs as a new task
let result = await handle;                     // waits for completion
```

`task::spawn` 把一個 future（這裡是 `async` 區塊）註冊到排程器，並回傳一個實作了 `Future` 的 `Task<T>` 控制代碼。

`task::spawn` 是函式庫函式，與 `async` 正交。把函式標示為 `async` 只是說明「這個函式是非同步的」；要實際把它作為獨立的執行單位啟動，需要明確地呼叫 `task::spawn` 之類的函式。

## 4.12.13. 方法上的 `async`

和一般[方法](409-method.md)一樣，方法可以標示為 `async`。

```nabi
impl Database {
    async fn query(self, sql: str) -> ResultSet { ... }
}

let r = await db.query("SELECT ...");
```

`async` 方法可以接受 `ref self`（[4.7.5](407-ref-keyword.md#475-exception-for-method-receivers)）。它的 future 作用於接收者的值，並在完成時把它寫回接收者的位置：無論是帶著值、帶著錯誤，還是被取消。因此值型別的接收者能看到方法所做的變更，與同步的 `ref self` 方法相同。

```nabi
struct Reader { pos: usize; data: List<u8>; }

impl Reader {
    async fn read_one(ref self) -> u8 | none {
        if self.pos >= self.data.len() { return none; }
        self.pos += 1;
        self.data[self.pos - 1]
    }
}

let r = Reader { pos: 0, data: [1u8, 2, 3] };
while await r.read_one() is u8 {}
r.pos;                              //→ 3
```

future 在完成之前持有接收者，因此呼叫要在其所在之處被 await：`await r.read_one()`，或對會擲回錯誤的方法寫 `try await r.read(buf)`。保留 future 以便之後再 await 是編譯錯誤（`E0090`），因為那樣就沒有任何東西會把接收者寫回，而接收者在此期間也可能改變：

```nabi
let f = r.read_one();               // ERROR: its future holds `r` until it completes
```

暫存的接收者，或經由 `any` 取得的接收者（其值是複本），會捨棄寫回，與同步的 `ref self` 呼叫相同。其他的 `ref` 參數在每個協程函式上仍然是錯誤（[4.7.7](407-ref-keyword.md#477-references-cannot-be-first-class-values)）。

## 4.12.14. 錯誤處理

[錯誤控制流程](../003-expressions/314-error-control-flow.md)的 `try`、`catch`、`throw` 與 `finally` 在非同步函式中照常運作。會擲回錯誤之 `async` 函式的錯誤在 `await` 處交付，因此 `await` 運算式要以 `try` 涵蓋（`try await fetch()`）。與 `throws` 簽章的互動在 [`throws` 簽章](414-throws.md)中說明。

```nabi
async fn safe_fetch() throws -> Data {
    try await fetch_or_throw() catch e: NetworkError {
        try await fetch_fallback()
    }
}
```
