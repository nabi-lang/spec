# 4.14. `throws` 簽章

本節定義函式簽章的 `throws` / `rethrows` 形式，它們與[錯誤控制流程](../003-expressions/314-error-control-flow.md)的 `throw`、`try`、`catch` 與 `finally` 結合使用。

> [!IMPORTANT]
> Nabi 只有一種錯誤模型：`throws`。可能擲回錯誤的函式在其簽章中寫出 `throws`，呼叫者則以 `try` 標示呼叫。標準函式庫的 [`Result<T, E>`](../../refs-standard-libraries/200-core/214-error.md) 只在必須把錯誤當作值保存時使用。規則只有一行：**要擲回它 → `throws`；要保存它 → `Result`。**

## 4.14.1. 形式

在[函式宣告](401-function-declaration.md)的 `fn_decl` 中，`throws_clause?` 位於參數括號之後、回傳型別箭頭之前。

```
throws_clause   ::= "throws" ("(" type_expr ")")?
                  | "rethrows"
```

因此 `fn_decl` 的完整形式如下：

```
fn_decl ::= visibility? fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
```

這裡的 `fn_modifiers` 是 [4.1 函式宣告](401-function-declaration.md)中定義的 `unsafe? async? yield?`。

```nabi
fn name(args) -> R                              // non-throwing
fn name(args) throws -> R                       // untyped: shorthand for throws (any Error)
fn name(args) throws (E) -> R                   // typed: single error type
fn name(args) throws (A | B | C) -> R           // typed: sum type of errors
fn name(f: fn(T) throws -> U) rethrows -> R     // throws only when the function argument throws
```

`throws` 與 `rethrows` 是上下文關鍵字，只在參數括號 `)` 之後才是關鍵字（[1.5.4](../001-lexical/105-keywords.md#154-上下文關鍵字)），並放在回傳型別箭頭之前。`throws` 之後可選的括號給出可能擲回之錯誤的型別。

## 4.14.2. 不擲回錯誤的函式

```nabi
fn add(x: i32, y: i32) -> i32 { x + y }
```

在沒有 `throws` 的函式中，不會有任何錯誤離開函式。若主體中 `throw` 的錯誤或以 `try` 標示之呼叫的錯誤沒有在函式內被 `catch` 完整處理，就是編譯錯誤。呼叫者不帶 `try` 地呼叫這個函式。

```nabi
fn bad() -> i32 {
    throw ValueError::new("nope"); // ERROR (E0062): error leaves a non-throwing function
}
```

這條規則強制簽章說明函式是否可能擲回錯誤，從而阻止錯誤默默傳播。進入點是唯一的例外：沒有回傳型別的明確 `fn main()` 不需宣告就能擲回錯誤（[4.16.4](416-entry-point.md#4164-結束碼與未處理的例外)）。

## 4.14.3. 未定型的 `throws`

```nabi
fn parse(s: str) throws -> i32 { ... }
```

未定型的 `throws` 是 `throws (any Error)` 的簡寫。函式可以擲回任何實作了標準函式庫 [`Error`](../../refs-standard-libraries/200-core/214-error.md) trait 之型別的錯誤，而擲回的值以 `any Error` 值，即[受約束的 `any`](../002-types/202-any-slot.md)，交付。呼叫者在 `catch` 中以型別模式收窄為具體型別，或以 `catch e: any Error` 或全收分支攔截所有錯誤。

當錯誤種類的集合對呼叫者而言並非封閉時，例如公開 API，建議使用未定型的形式。

## 4.14.4. 定型的 `throws`：單一型別與和型別

```nabi
fn fetch_user(id: i64) throws (NotFound) -> User { ... }
fn parse_or_load(s: str) throws (ParseError | IoError | TimeoutError) -> Data { ... }
```

括號中可以說明可能擲回的確切型別，而和型別（`A | B | C`）可以原樣使用。

定型的形式適合內部或函式庫 API、嵌入式環境，以及錯誤種類集合封閉的情況。此時編譯器能檢查呼叫者的 `catch` 分支是否涵蓋了所有錯誤型別；若是，該運算式就被視為不再擲回錯誤。

```nabi
fn caller() -> Data {             // non-throwing: all three types handled
    try parse_or_load(input)
    catch e: ParseError { ... }
    catch e: IoError { ... }
    catch e: TimeoutError { ... }
}
```

## 4.14.5. `throws` 與主體的一致

每個可能離開函式的錯誤都必須與簽章的 `throws` 相容。錯誤源自主體中的 `throw` 陳述式，以及以 `try` 標示的會擲回錯誤之呼叫；在函式內被 `catch` 處理的錯誤不納入檢查。

```nabi
fn f() throws (A) -> i32 {
    throw A_instance;             // OK
    throw B_instance;             // ERROR (E0062): only A allowed
}
```

相容性規則如下：對定型的 `throws (E)`，每個源頭的錯誤型別都必須是 `E` 的成員。對未定型的 `throws`，每個源頭的錯誤型別都必須實作 `Error`，而值以 `any Error` 交付。

錯誤型別之間沒有自動轉換。要把被呼叫者的錯誤以不同的型別傳播，請以 `catch` 包裝它並重新擲回。

```nabi
fn load_config(path: str) throws (ConfigError) -> Config {
    try parse_toml(read_text(path))
    catch e: IoError { throw ConfigError::io(e); }
    catch e: ParseError { throw ConfigError::syntax(e); }
}
```

## 4.14.6. 呼叫者的義務

呼叫 `throws` 函式的運算式必須位於被 `try` 涵蓋的運算式內。不帶 `try` 地呼叫會擲回錯誤的函式是編譯錯誤。`try` 涵蓋其右側的整個運算式，因此 `try a(b())` 涵蓋 `a` 與 `b` 兩個呼叫，`try { a(); b(); }` 涵蓋區塊中的所有呼叫（[3.14.2](../003-expressions/314-error-control-flow.md#3142-try標示呼叫處)）。

以 `try` 標示之呼叫的錯誤由最近的 `catch` 處理；未被處理的錯誤傳播到外層函式，而外層函式的 `throws` 必須接受該錯誤型別（4.14.5）。

```nabi
fn caller() {                     // non-throwing
    parse(s);                     // ERROR (E0062): throwing call not marked with try
    try parse(s);                 // ERROR (E0062): error propagates but caller is not throws
}

fn caller2() throws {             // propagates
    try parse(s);                 // OK
}

fn caller3() {                    // catches
    try parse(s) catch _ {
        ...
    };
}

fn caller4() -> i32 {             // discards the error or turns it into a panic
    let a = try? parse(s) ?? 0;   // try?: none on error, then 0
    let b = try! parse(t);        // try!: panic on error
    a + b
}
```

因此每個錯誤可能流經之處，都會以 `try` 呈現在程式碼中；而錯誤能傳播多遠，則可從函式簽章的 `throws` 看出。

## 4.14.7. `rethrows`

```nabi
fn map<U>(self, f: fn(T) throws -> U) rethrows -> List<U> {
    let out: List<U> = [];
    for x in self {
        out.push(try f(x));
    }
    out
}
```

`rethrows` 是接受函式型別參數之高階函式的形式。`rethrows` 函式只在作為引數傳入的函式或閉包擲回錯誤時才會擲回錯誤。

- 若在呼叫處傳給會擲回錯誤之函式型別參數的每個引數都不會擲回錯誤，該呼叫就不擲回錯誤，不帶 `try` 地進行。
- 若其中任何一個引數會擲回錯誤，該呼叫就會擲回錯誤，而擲回的錯誤型別是那些引數的錯誤型別。呼叫者以 `try` 標示它。
- 在 `rethrows` 函式的主體中，唯一能離開函式的錯誤，是來自函式型別參數之呼叫的錯誤。若主體直接擲回它自己的錯誤，就是編譯錯誤。

```nabi
let lens = names.map(|s| s.len());              // non-throwing closure: no try
let nums = try names.map(|s| try parse(s));     // throwing closure: try required
```

當 `rethrows` 函式被用作值時，由預期型別（一個函式型別）決定使用哪個實例。沒有預期型別時，錯誤參數被設為 `any Error`。

## 4.14.8. 函式型別中的 `throws`

`throws` 也附加在[函式型別](404-function-type.md)上。

```nabi
fn(T) -> U                    // non-throwing
fn(T) throws -> U             // untyped: throws (any Error)
fn(T) throws (E) -> U         // typed
```

不擲回錯誤的函式值可以用在參數與回傳型別相同、會擲回錯誤的函式型別位置；`throws (A)` 函式值可以用在 `throws (A | B)` 位置，或（若 `A` 實作了 `Error`）未定型的 `throws` 位置。反方向是不允許的。詳見 [4.4](404-function-type.md)。

## 4.14.9. `throws` 與 `async` 及 `yield` 結合

三個修飾詞可以在同一個函式上結合。

```nabi
async fn fetch(url: str) throws (NetworkError) -> Data { ... }

yield fn lines(s: str) throws (ParseError) -> str { ... }

async yield fn read_paginated(url: str) throws (NetworkError) -> Page { ... }
```

`throws` 是簽章位置的關鍵字，與 `async` 或 `yield` 正交地結合。

## 4.14.10. `throws` 對和型別簽章的影響

```nabi
async fn fetch_data() throws (NetworkError) -> Data { ... }
```

有人可能會問：呼叫這個函式，是否會得到容納 `Data | NetworkError` 的 Future？

答案是 **`TryAsyncFuture<Data, NetworkError>`**（[4.1.11](401-function-declaration.md#協程函式的結果型別)）。錯誤不會混入結果型別；它在 `await` 處經由錯誤通道交付。因此 `await` 的一方要以 `try` 標示它。`TryAsyncFuture` 不實作一般的 `Future` trait。

```nabi
let f = fetch_data();              // f: future::TryAsyncFuture<Data, NetworkError>
let d = try await f catch e: NetworkError {
    ...
};
```

這個決定避免了結果變成帶有結果與錯誤兩個變體的 `enum`。

`yield` 與 `async yield` 遵循相同的模式。帶有 `throws (E)` 的產生器是 `TryGenIter<T, E>`，串流是 `TryAsyncStream<T, E>`；錯誤在 `next` 處經由錯誤通道交付。

### `try for`

走訪 `TryIterator`（會擲回錯誤之產生器的 `TryGenIter`、`io::Bytes`……）的 `for` 在每次 `next` 時都會擲回錯誤，因此寫成 `try for`。`try` 只涵蓋迭代：可迭代運算式與每一次 `next`。主體不被涵蓋；那裡會擲回錯誤的呼叫以它自己的 `try` 標示（[3.13.3](../003-expressions/313-loop-expressions.md#3133-for以迭代器為基礎的重複)）。走訪 `TryStream` 的 `for await` 以相同方式運作。

```nabi
fn total(path: str) throws (IoError | ParseError) -> i64 {
    let n: i64 = 0;
    try for line in lines(path) {           // IoError from each next propagates
        n += try i64::from_str(line);       // the body marks its own call
    }
    n
}
try for await ev in events() { try handle(ev); }
for line in lines("a.txt") { ... }          // ERROR (E0062): `next` can throw; write `try for`
```

| 形式 | 某一步發生錯誤時 |
|---|---|
| `try for` | 傳播到最近的 `catch`，否則離開函式 |
| `try! for` | panic（[3.14.3](../003-expressions/314-error-control-flow.md#3143-try-與-try)） |
| `try? for` | 結束迴圈；`for` 運算式為 `none`（否則為 `()`） |
| `try for .. catch e: E { .. }` | `catch` 附加在整個 `try for` 運算式上 |

主體中的 `try` 和任何巢狀的 `try` 一樣導引其錯誤：送往迴圈之後的 `catch`，或迴圈的 `try?` / `try!`，否則離開函式。要在錯誤之後繼續，請走訪 `.results()`，它以 `Result` 給出每一步（[stdlib 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator可能失敗的迭代)）。

## 4.14.11. 方法上的 `throws`

和一般[方法](409-method.md)一樣，方法可以有 `throws` 簽章。

```nabi
impl Database {
    fn query(self, sql: str) throws (SqlError) -> ResultSet { ... }
}
```

呼叫者的義務與傳播規則完全與自由函式相同。

實作 trait 的 impl 方法，其 `throws` 必須與 trait 方法的 `throws` 相同。若 impl 加入 `throws` 或放寬錯誤型別，就是編譯錯誤（E0074）。不寫 `throws` 的 impl 方法取用 trait 方法的 `throws`，其錯誤型別所指稱的就是在 trait 處所指稱的：`impl Display for P { fn fmt(self, ref f: Formatter) { try f.write_str(..); } }` 擲回 `FmtError`。

運算子 trait 方法（`Add::add` 等）宣告時不帶 `throws`，因此運算子不會擲回錯誤。可能失敗的運算以具名方法提供，或回傳 `Result`（[stdlib 2.3](../../refs-standard-libraries/200-core/203-ops.md)）。使用者運算子也是如此：在宣告了 `throws` 的方法上使用 `#[infix]` / `#[prefix]` / `#[postfix]` 是錯誤（[6.7.3](../006-meta-system/607-operator-magic.md#673-連接機制)）。

## 4.14.12. 以值為基礎的實作模型

會擲回錯誤的函式以值的形式回傳其結果。回傳值是一個成功旗標與一個酬載的組合，酬載容納成功值或錯誤值。錯誤通道由簽章區分，因此成功型別與錯誤型別可以是同一個型別。呼叫處的 `try` 檢查旗標並分支；發生錯誤時，它前往最近的 `catch`，或把錯誤作為外層函式的錯誤結果回傳。

在這個模型中，沒有錯誤之路徑的成本是一次分支，而進入 `try` 或 `catch` 不需要任何成本。不使用堆疊展開，也不使用執行時的錯誤交付函式。確切的回傳值配置定義於 [8.6.5](../008-abi/806-calling-convention.md#865-會擲回錯誤之函式的回傳以值為基礎的錯誤傳播)，降階定義於 [8.9.6](../008-abi/809-lowering-control.md#896-try--catch--finally以值為基礎的分支)。

## 4.14.13. panic 與致命錯誤

panic 不是 `throws` 錯誤。panic（標準函式庫的 `panic`、失敗的 `try!`、除以零、整數溢位、邊界檢查失敗）以及[錯誤控制流程](../003-expressions/314-error-control-flow.md)的致命錯誤，不屬於任何 `throws` 簽章，也無法以 `catch` 攔截。執行環境把下列內容寫到標準錯誤（stderr），並以結束碼 `101` 終止行程。

```
thread '<name>' panicked at <path>:<line>:<col>:
<message>
```

- `<name>` 是執行緒名稱：主執行緒為 `main`，未命名的執行緒為 `<unnamed>`。
- 發生在 C 所呼叫之函式之下的 panic，會在報告之後中止行程而不是結束，或把該函式的 `#[on_panic]` 值回傳給 C（[5.9.12](../005-modules/509-extern-block.md#5912-ffi-安全的責任)）。以 `std::panic::set_hook` 設定的掛鉤在每種情況下都會在報告之後執行（[stdlib 3.12](../../refs-standard-libraries/300-std/312-panic.md)）。
- 位置是失敗的運算式。在 [`#[track_caller]`](../006-meta-system/602-built-in-attributes.md#6212-track_caller-傳遞呼叫處) 函式的鏈之內，它是該鏈之外的第一個呼叫位置。
- 環境中有 `NABI_BACKTRACE=1` 時，其後接著 `stack backtrace:`：發生 panic 之執行緒呼叫堆疊上的函式，從最內層開始，每行一個，以 Nabi 路徑表示（`   0: geom::distance`）。頂端的標準函式庫 panic 函式會被省略，最後一行 `note:` 說明如何看到它們。`NABI_BACKTRACE=full` 列出每個框架及其回傳位址（`   0: 0x000000010000a4f0 - geom::distance`），直到第一個不是 Nabi 程式碼的框架（`<unknown>`）為止。未設定、為空或為 `0`：沒有回溯。[9.9.6](../009-tools/909-debug-tools.md#panic-回溯) 說明它如何產生以及在哪裡可用。
