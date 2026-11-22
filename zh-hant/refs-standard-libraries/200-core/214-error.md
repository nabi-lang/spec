# 2.14. error：錯誤 trait 與型別

`core::error` / `core::panic` 模組是錯誤處理的依據（`ZeroError` 與 `NonZero` 一同位於 `core::num::niche`）。Nabi 只有一種錯誤模型。可能擲回錯誤的函式在簽章中寫 `throws`，呼叫端以 `try` 標記該呼叫（[refs-lang 4.14](../../refs-lang/004-functions/414-throws.md)）。`Result<T, E>` 是一個普通的列舉，用來**把錯誤當作值保存**；它不是傳遞錯誤的手段。

本章定義 100 個 lang 槽位中的三個：`panic_fn` / `oom_fn` / `unreachable_fn`。

## 2.14.1. 模型概覽：以 `throws` 擲回，以 `Result` 保存

```nabi
// (a) throwing: throws and try
fn parse_int(s: str) throws (ParseError) -> i32 {
    if s.is_empty() || !s.chars().all(|c| c.is_ascii_digit()) {
        throw ParseError::new(s, 0, "not a digit");
    }
    try s.parse::<i32>()
}

let n = try parse_int("42") catch e: ParseError {
    log(f"parse failed: {e:?}");
    0
};

// (b) holding: Result
let results: List<Result<i32, ParseError>> =
    inputs.map(|s| Result::catching(|| try parse_int(s)));

for r in results {
    match r {
        Ok(v) => total = total + v,
        Err(e) => failures.push(e),
    }
}
```

選擇依循一條規則：**錯誤必須流向呼叫端時用 `throws`；錯誤必須保留下來稍後處理時用 `Result`。**

| 情況                                             | 工具                    | 理由                                      |
|--------------------------------------------------|-------------------------|-------------------------------------------|
| 可能失敗的操作（I/O、剖析、查找）                | `throws`                | 每個呼叫處的 `try` 讓流程一目了然         |
| API 邊界／公開 API                               | `throws (E)` / `throws` | 錯誤型別公開在簽章中                      |
| 收集平行執行的結果、把結果傳給回呼               | `Result<T, E>`          | 把錯誤保存為值                            |
| 蒐集多個錯誤一次回報、快取結果                   | `Result<T, E>`          | 存為值，稍後處理                          |
| 忽略失敗，只需要值                               | `try?`                  | 以 `T \| none` 接收                       |
| 失敗代表程式錯誤（bug）                          | `try!`                  | 失敗時觸發 panic                          |

## 2.14.2. `Result<T, E>`：用於保存的列舉

`Result<T, E>` 是 `core::error` 中的普通列舉。

```nabi
enum Result<T, E> {
    Ok(T),
    Err(E),
}
```

> [!NOTE]
> Nabi **沒有 `Some` / `None` 列舉變體**。`Option<T>` 本身**並不存在**；
> 改用和型別 `T | none`。`Result` 之所以定義為獨立的列舉，是因為**兩個分支都帶有有意義的資料**。

### 核心方法

`Result` 的核心方法如下：

```nabi
impl<T, E> Result<T, E> {
    fn is_ok(self) -> bool;
    fn is_err(self) -> bool;
    fn ok(self) -> T | none;
    fn err(self) -> E | none;

    fn unwrap(self) -> T;                               // panics on Err: `Result::unwrap: called on `Err(<e>)`; ...`
    fn unwrap_or(self, default: T) -> T;
    fn unwrap_or_else(self, f: fn(E) -> T) -> T;

    fn map<U>(self, f: fn(T) -> U) -> Result<U, E>;
    fn map_err<F>(self, g: fn(E) -> F) -> Result<T, F>;
    fn and_then<U>(self, f: fn(T) -> Result<U, E>) -> Result<U, E>;
    fn or_else<F>(self, g: fn(E) -> Result<T, F>) -> Result<T, F>;

    fn get(self) throws (E) -> T;                       // throws on Err, back into the error flow
    fn catching(f: fn() throws (E) -> T) -> Result<T, E>;   // captures the error flow as a value
}
```

對 `Err` 呼叫 `unwrap` 會觸發 panic，訊息為 `Result::unwrap: called on `Err(<e>)`` 加上錯誤的文字，再接 `; use `try r.get()` to throw it instead`（0.5.10）。

當 `T` 與 `E` 都實作 `Debug`、`Clone`、`PartialEq`、`Eq` 與 `Hash` 時，`Result<T, E>` 也實作它們（2.14.8）。

`get` 與 `catching` 銜接這兩個世界。`catching` 呼叫一個會擲回的函式，成功時把結果保存為 `Ok`，出錯時保存為 `Err`；`get` 取出保存的值，若為 `Err` 則重新擲回該錯誤。

### `Result` 沒有 `?`

[refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410-the--operator---early-return-of-none) 的後綴 `?` 只用於 `T | none`。對 `Result` 值使用 `?` 時，編譯器會發出下列診斷。

```nabi
fn load(r: Result<Config, IoError>) -> Result<Config, IoError> {
    let cfg = r?;
    // error[E0064]: `?` only works on `T | none`, found `Result<Config, IoError>` - to propagate errors use `throws` and `try`
    ...
}
```

要把保存的錯誤傳遞給呼叫端，請把函式宣告為 `throws`，並寫 `try r.get()`。

```nabi
fn load(r: Result<Config, IoError>) throws (IoError) -> Config {
    try r.get()
}
```

## 2.14.3. `Error` trait：錯誤的外觀

每個錯誤型別的外觀由 `Error` trait 定義。

```nabi
trait Error: Display + Debug {
    const fn source(self) -> any Error | none {
        none                                  // default: no chain
    }
}
```

錯誤型別實作這個 trait，以 `Display` 輸出訊息，以 `Debug` 做偵錯顯示。無型別的 `throws` 是 `throws (any Error)` 的簡寫，因此無型別擲回函式所擲回的值必須實作 `Error`。若要不論型別地保存或傳遞錯誤，請使用 `any Error`，即一種[受約束的 `any`](../../refs-lang/002-types/202-any-slot.md)（`any Error | none` 即 `(any Error) | none`）。

### `#[derive(Error)]`

`#[derive(Error)]` 以預設的 `source()` 實作 `Error`。它對欄位沒有任何要求；`Display` 與 `Debug` 這兩個父 trait 來自它們各自的 derive／屬性或 impl，因此錯誤型別通常寫成 `#[derive(Debug, Error)]` 加上 `#[display("..")]`（[2.10.14](210-fmt.md#21014-衍生政策摘要)）。

### 鏈：`source()`

包裝錯誤時，可以如下表達原因鏈。

```nabi
struct ConfigError {
    msg: str;
    cause: any Error | none;
}

impl Error for ConfigError {
    fn source(self) -> any Error | none {
        self.cause
    }
}

impl Display for ConfigError {
    fn fmt(self, ref f: Formatter) throws (FmtError) {
        try f.write_str(self.msg)
    }
}
```

把值存入 `cause` 時，編譯器會在編譯期檢查其靜態型別是否實作 `Error`。以這種方式建立鏈之後，REPL 或記錄器就能沿著它印出原因的追溯。

```nabi
let cfg = try parse_config(text) catch e: ParseError {
    throw ConfigError { msg: "bad config", cause: e };
};
```

## 2.14.4. 內建錯誤型別

標準函式庫預設提供的錯誤型別定義如下。它們的欄位是私有的（[0.5.10](../000-meta/005-notation.md)）：每個型別都以 `new` 建立，並透過其 getter 與 `message()` 讀取。

```nabi
struct TypeError { .. }
impl TypeError {
    fn new(expected: str, actual: str) -> TypeError;
    fn expected(self) -> str;
    fn actual(self) -> str;
    fn message(self) -> str;          // "expected `i32`, found `str`"
}

struct ValueError { .. }
impl ValueError {
    fn new(message: str) -> ValueError;
    fn message(self) -> str;
}

struct OverflowError { .. }
impl OverflowError {
    fn new(operation: str, type_name: str) -> OverflowError;
    fn operation(self) -> str;        // "try_from" / ...
    fn type_name(self) -> str;        // "i32" / "i64" / ...
    fn message(self) -> str;          // "try_from: value out of range for `i32`"
}

struct ParseError { .. }
impl ParseError {
    fn new(input: str, position: usize, message: str) -> ParseError;
    fn input(self) -> str;
    fn position(self) -> usize;       // byte offset in input where parsing failed
    fn message(self) -> str;          // why, without the input Display adds
}

struct ConversionError { .. }
impl ConversionError {
    fn new(message: str) -> ConversionError;
    fn message(self) -> str;
}

struct ZeroError;                     // core::num::niche; message()
```

每個型別都實作 `Error`、`Display`、`Debug`、`PartialEq`、`Eq` 與 `Clone`。其 `Display` 即 `message()`；`ParseError` 的 `Display` 會把輸入放在前面（`cannot parse "x7": invalid digit`）。各錯誤在何處擲回：

| 型別              | 擲回者                                                                     |
|-------------------|----------------------------------------------------------------------------|
| `TypeError`       | 執行時強制轉型失敗、型別檢查失敗                                           |
| `ValueError`      | 一般的無效值（剖析器／API 等）；`throw "msg"` 擲回的就是它                 |
| `OverflowError`   | `try_into` 等受檢轉換超出範圍（算術溢位則觸發 panic）                      |
| `ParseError`      | `str → number` 等剖析失敗（NABION 使用 `NabionError`）                     |
| `ConversionError` | `TryFrom::Error` 的預設值；非數值轉換失敗（[2.11.4](211-conversion.md#2114-tryfromu-trait可能失敗的轉換)）；`NonNan` / `Finite` 的 `try_from` |
| `ZeroError`       | 把 0 傳給 `NonZero::try_from`（[2.2.2](202-non-zero.md#222-nonzerot禁止零)） |

### Fatal 錯誤

下列型別歸類為 Fatal。

```nabi
struct StackOverflowError;     // Fatal: cannot be caught
struct OutOfMemoryError;       // Fatal: cannot be caught
struct InternalError;          // Fatal: compiler-internal bug
```

它們也各有 `message()`（`"stack overflow"` 等）。Fatal 錯誤不是 `throws` 錯誤，無法以 `catch` 攔截。它與 panic 一樣，把訊息印到 stderr，並以結束碼 `101` 結束行程（[refs-lang 3.14.8](../../refs-lang/003-expressions/314-error-control-flow.md#3148-panics-and-fatal-errors)）。

## 2.14.5. `panic_fn` / `oom_fn` / `unreachable_fn`：編譯器鉤子

錯誤模型的最底層是編譯器直接呼叫的鉤子函式。

```nabi
mod panic {                         // core::panic
    #[lang("panic_fn")]
    #[track_caller]
    fn panic(msg: str) -> ! {
        nabi_panic_at(msg)            // prints to stderr with the caller's location, then exit code 101
    }

    #[lang("oom_fn")]
    fn oom() -> ! {
        nabi_oom()                    // "fatal error: out of memory", exit code 101
    }

    #[lang("unreachable_fn")]
    #[track_caller]
    fn unreachable() -> ! {
        nabi_panic_at("internal error: entered unreachable code")
    }
}
```

每個函式的回傳型別都是 `-> !`（never），絕不會正常回傳。`panic` 與 `unreachable` 帶有 `#[track_caller]`：回報的位置是 `#[track_caller]` 鏈之外的第一個呼叫者。

### panic 發生之處

編譯器在下列位置呼叫 `panic_fn`。所有情況下該 panic 都無法以 `catch` 攔截，`finally` 與 `Drop` 也不會執行。

- `try!` 的錯誤路徑：訊息為錯誤值的 `Display`（若無則為 `Debug`）
- 除以零、整數溢位、邊界檢查失敗
- 標準函式庫中明確的 panic 呼叫，例如對 `Err` 呼叫 `Result::unwrap`

`throw` 不是 panic。`throw` 只產生擲回函式的錯誤結果（[refs-lang 8.9.7](../../refs-lang/008-abi/809-lowering-control.md#897-throw)），不經過 `panic_fn`。

### OOM 處理

GC 配置失敗時，OOM 依下列方式處理。執行環境無法以名稱呼叫 Nabi 函式，因此它直接呼叫執行環境符號 `nabi_oom`，而不是 `oom()`（[refs-lang 8.5.7](../../refs-lang/008-abi/805-extern-symbols.md#857-diagnostic-area)）。

```
nabi_gc_alloc_slow:
    if !try_collect_and_retry() {
        nabi_oom();                       // "fatal error: out of memory", exit 101
    }
```

core 的 `oom()` 呼叫同一個 `nabi_oom`，因此兩條路徑產生相同的輸出與結束碼。

OOM 屬於 Fatal，不會被攔截。

### 使用 `unreachable`

`unreachable` 標記一個不可能到達的分支。

```nabi
match (x: i32 | f64) {
    v: i32 => process_int(v),
    v: f64 => process_float(v),
    // the compiler confirms every variant is handled; no unreachable position
}

// or explicitly.
fn require_positive(x: i32) -> u32 {
    if x < 0 { core::panic::unreachable() }     // guaranteed by the caller
    x as u32
}
```

編譯器會在下列位置自動插入 `unreachable`：
- `match` 窮盡檢查之後的死分支
- `if let ... else` 的 else 保證為發散（divergent）的位置

### `assert` / `assert_eq` / `assert_ne`

```nabi
fn assert(cond: bool, msg: str = "") -> ();
fn assert_eq<T: PartialEq + Debug>(left: T, right: T, msg: str = "") -> ();
fn assert_ne<T: PartialEq + Debug>(left: T, right: T, msg: str = "") -> ();
```

透過 core prelude，它們在任何地方都可使用。條件為假時觸發 panic（結束碼 101）。訊息如下；若 `msg` 不為空，會附加 `: msg`。

```
assertion failed[: msg]
assertion `left == right` failed[: msg]
  left: <Debug of left>
 right: <Debug of right>
```

`assert_ne` 使用 `left != right`，其餘相同。

## 2.14.6. `try` / `catch` / `throw` / `throws` / `finally`

語法由 [refs-lang 3.14 錯誤控制流程](../../refs-lang/003-expressions/314-error-control-flow.md)與 [refs-lang 4.14 `throws` 簽章](../../refs-lang/004-functions/414-throws.md)規範；本節只涵蓋標準函式庫的慣例。

### 簽章模型

函式簽章中的擲回行為有五種形式。

```nabi
fn foo() -> T                      // non-throwing: never throws
fn foo() throws -> T               // untyped: throws (any Error)
fn foo() throws (E) -> T           // typed single: only E
fn foo() throws (A | B | C) -> T   // typed sum: only A | B | C
fn foo(f: fn() throws -> T) rethrows -> T   // throws only when f throws
```

無型別的 `throws` 是公開 API 建議的預設形式，因為它讓 API 規格保持輕量。`throws (E)` 適合函式庫內部 API、嵌入式環境，或攔截端需要依錯誤分支的情況。具型別的和型別形式 `throws (A | B | C)` 把 Swift SE-0413 的具型別擲回（typed throws）延伸到和型別。接受函式參數的標準函式庫高階函式使用 `rethrows`，因此傳入不會擲回的閉包時，呼叫可以不寫 `try`。

### `try` / `catch` 形式

`catch` 是後綴運算式；攔截到錯誤時，處理區塊的值就成為整個運算式的值。

```nabi
let result = try compute() catch e: TypeError {
    log(f"type error: {e}");
    default_value
};
```

### `finally`

`finally` 區塊可以用於資源清理等用途。

```nabi
let f = try File::open(path);
try {
    process(f);
} catch e: IoError {
    log(f"io error: {e}");
} finally {
    try? f.close();     // runs on both normal completion and error; a close error is dropped
}
```

`finally` 的執行語意：
- 正常完成時執行。
- 在 catch 之後完成時執行。
- 未被攔截的錯誤傳遞時，先執行它，再傳遞錯誤。
- 以 `return` / `break` / `continue` 離開時執行。
- 發生 panic 或 Fatal 錯誤時不執行，因為行程會立即結束。

在 `finally` 內擲回的錯誤會取代目前的錯誤。

### `throw` 與分支型別

`throw value;` 是陳述式，以 `throw` 結尾的區塊是發散的。因此分支的結果型別只反映其他分支的型別。例如，`if c { x } else { throw e; }` 的結果型別即 `x` 的型別。

## 2.14.7. `Result` 與 `throws` 之間的轉換

這兩種工具可以明確地互相轉換。

### 擲回呼叫 → `Result<T, E>`

`Result::catching` 呼叫一個會擲回的閉包並保存結果。也可以用 `catch` 運算式直接建立。

```nabi
let r: Result<i32, ParseError> = Result::catching(|| try parse_int(s));
let r: Result<i32, ParseError> = try Ok(parse_int(s)) catch e { Err(e) };
```

### `Result<T, E>` → 擲回

`get` 重新擲回 `Err`。外層函式的 `throws` 必須接受該錯誤型別。

```nabi
fn first_ok(rs: List<Result<i32, ParseError>>) throws (ParseError) -> i32 {
    try rs[0].get()
}
```

### 取出為 `T | none`

要捨棄錯誤只保留值，對擲回呼叫使用 `try?`，對保存的 `Result` 使用 `ok()` / `err()`。

```nabi
let a: i32 | none = try? parse_int("42");
let r: Result<i32, ParseError> = Result::catching(|| try parse_int("42"));
if let Ok(v) = r { process(v); }
if let n: i32 = r.ok() { process(n); }     // extracted as T | none
```

## 2.14.8. trait 實作矩陣

各錯誤型別實作的 trait：

| 型別                 | Display | Debug         | Error | 備註                           |
|----------------------|---------|---------------|-------|--------------------------------|
| `TypeError`          | ✓       | ✓             | ✓     |                                |
| `ValueError`         | ✓       | ✓             | ✓     |                                |
| `OverflowError`      | ✓       | ✓             | ✓     | 受檢轉換超出範圍               |
| `ParseError`         | ✓       | ✓             | ✓     |                                |
| `ConversionError`    | ✓       | ✓             | ✓     | 預設的 `TryFrom::Error`        |
| `ZeroError`          | ✓       | ✓             | ✓     | 位於 `core::num::niche`        |
| `FmtError`           | ✓       | ✓             | ✓     | 寫入失敗（[2.10](210-fmt.md)） |
| `Utf8Error`          | ✓       | ✓             | ✓     | 唯一的 UTF-8 錯誤（[2.15](215-string.md)） |
| `IoError`            | ✓       | ✓             | ✓     | std 層級（依目標而定）         |
| `StackOverflowError` | ✓       | ✓             | ✓     | Fatal：無法攔截                |
| `OutOfMemoryError`   | ✓       | ✓             | ✓     | Fatal：無法攔截                |
| `InternalError`      | ✓       | ✓             | ✓     | Fatal：編譯器錯誤時            |
| `Result<T, E>`       | ✗       | ✓ (T,E:Debug) | ✗     | 用於保存的列舉；`T` 與 `E` 具備時實作 `Clone` / `PartialEq` / `Eq` / `Hash` |
| `any Error`          | ✓       | ✓             | ✓     | 受約束的 any，可形成鏈         |

`Display` 提供對使用者友善的訊息；例如 `TypeError::new("i32", "str")` 讀作 ``"expected `i32`, found `str`"``。`Debug` 原樣呈現結構。表中每個錯誤型別也都實作 `PartialEq`、`Eq` 與 `Clone`（0.5.10）。`any Error` 值透過執行時標籤呼叫其實際型別的 `Display` / `Debug` / `source`。

## 2.14.9. 與其他語言的比較

與其他主要語言相比，Nabi 的錯誤模型定位如下。

| 面向             | Rust           | Swift                     | Java            | Nabi                                      |
|------------------|----------------|---------------------------|-----------------|-------------------------------------------|
| 基本模型         | `Result<T, E>` | `throws`                  | `throws`        | `throws`；`Result` 用於保存               |
| 呼叫處標記       | `?`            | `try`                     | **無**          | `try` / `try?` / `try!`                   |
| typed throws     | **無**         | `throws(E)`（單一型別）   | `throws E1, E2` | `throws (E)` / `throws (A \| B)`          |
| `?` 運算子       | `Result?`      | 可選鏈結                  | **無**          | 僅用於 `T \| none`                        |
| Option 型別      | `Option<T>`    | `Optional<T>`             | `Optional<T>`   | **無**：使用 `T \| none`                  |
| 實作方式         | 回傳值         | 回傳值（錯誤暫存器）      | 堆疊展開        | 回傳值 `{ is_err, payload }`              |
| panic            | `panic!`       | `fatalError`              | `Error`         | `panic_fn`：無法攔截，結束碼 101          |

Nabi 主要有三點不同。第一，沒有 `Option`；由和型別 `T | none` 取代，少了一個型別。第二，錯誤傳遞統一於 `throws` 與 `try`，`Result` 只用於把錯誤保存為值。第三，具型別的和型別擲回 `throws (A | B | C)` 比只接受單一型別的 Swift typed throws 更具表達力。
