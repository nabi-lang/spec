# 5.3. `static`

`static` 關鍵字（上下文關鍵字，只在項目開頭是關鍵字；其後的 `mut` 同樣只在 `static mut` 位置是關鍵字，[1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)）宣告在程式整個生命週期內持續存在的全域記憶體。

```
static_decl     ::= "static" "mut"? identifier ":" type_expr "=" expr ";"
extern_static   ::= "static" "mut"? identifier ":" type_expr ";"   // in extern block
```

## 5.3.1. 基本形式

```nabi
static SHARED_COUNTER: Atomic<i32> = Atomic::new(0i32);
static DEFAULT_CONFIG: Config = Config::default();
static APP_NAME: str = "nabi";
```

基本形式是 `static`、名稱、帶型別的 `:`，以及帶初始化式的 `=`。型別必須寫出。`const` 初始化式（[`const`](502-const.md)）在編譯期求值；其他任何初始化式在第一次存取時執行一次（[5.3.9](#539-initializers-compile-time-or-first-use)）。

`static` 可以從每個執行緒存取，因此其型別必須是 `Sync`（[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)）；延遲初始化的 static 也是如此。`static XS: List<i32> = [];` 是編譯錯誤（集合型別、`StrBuf` 與 `Cell<T>` 是 `!Sync`）；請寫 `static XS: Mutex<List<i32>> = Mutex::new([]);`。每個執行緒各自的狀態使用 `thread::local`（[stdlib 3.4.4](../../refs-standard-libraries/300-std/304-thread.md#344-thread-local-storage)），其 `ThreadLocal<T>` 是 `Sync`。`extern` 區塊的 static（[5.3.10](#5310-extern-static-ffi)）豁免。

## 5.3.2. 與 `const` 的差異

```
const X: i32 = 42i32;              // inlined at each use (no address)
static X: i32 = 42i32;             // stable memory address (program lifetime)
```

| 面向           | `const`        | `static`                                       |
|----------------|----------------|------------------------------------------------|
| 記憶體位址     | 無             | 穩定的位址                                     |
| 在使用處       | 內嵌           | 記憶體參照                                     |
| 生命週期       | 編譯期         | 程式的生命週期                                 |
| 可變性         | 一律不可變     | 預設不可變（可透過包裝取得可變性）             |

## 5.3.3. 允許的位置

`static` 可以宣告在模組層級或區塊中（[3.9.1](../003-expressions/309-block-expressions.md#391-form)），而不能在 trait 或 `impl` 中（[5.3.8](#538-combination-with-methods--traits)）。

```nabi
static APP_NAME: str = "nabi";                          // module level: OK

fn f() -> i32 {
    static CALLS: Atomic<i32> = Atomic::new(0i32);     // OK: one slot, visible in this block
    CALLS.fetch_add(1i32)
}
```

區塊中的 `static` 是整個程式的一個槽位，只初始化一次，無論其函式的哪個實例在執行。

## 5.3.4. 一般使用者程式碼中的可變全域變數：使用包裝

```nabi
static COUNTER: Atomic<i32> = Atomic::new(0i32);
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

需要內部可變性的全域變數，要包在並行安全的包裝中宣告。

- `Atomic<T>`：無鎖的原子運算（整數、`bool`、指標）。
- `Mutex<T>`：以鎖保護。
- `RwLock<T>`：讀寫鎖，用於本身是 `Sync` 的 `T`（[stdlib 2.16.4](../../refs-standard-libraries/200-core/216-sync.md#2164-rwlockt---read-write-split)）。

`Cell<T>` 是單執行緒的且為 `!Sync`，因此不能作為 `static`；每個執行緒各自的可變全域變數是 `thread::local(..)`。

這些全都是標準函式庫型別；其確切簽章定義於函式庫規格。

## 5.3.5. `static mut`：僅用於 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)

只有為了直接與 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 介接時才能使用 `static mut`，而且只能在 `extern "C"` 區塊內。

```nabi
extern "C" {
    static environ: CPtr<CPtr<c_char>>;   // FFI global (read-only)
    static mut atexit_count: u64;  // FFI mutable global
}
```

- `static mut` 只允許出現在 `extern "C"` 區塊內。
- 一般 Nabi 程式碼中的 `static mut` 是錯誤。
- 對不是 extern `static mut` 的 `static` 指派是編譯錯誤（`COUNTER = ...`），對它呼叫 `ref self` 方法也是（`S.set(3)`，[4.7](../004-functions/407-ref-keyword.md)）。內部的變更透過上述包裝的方法進行。

共享的可變狀態是資料競爭的直接原因，因此有此限制。一般程式碼透過包裝獲得安全性；原始存取只允許在 FFI 領域中進行。

## 5.3.6. 初始化順序

多個 `static` 宣告之間的初始化順序：

- 在同一個模組內，依宣告順序。
- 跨模組時，依相依順序：若一個 `static` 的初始化式參照另一個 `static`，後者先初始化。
- 經由直接參照的循環，即每個初始化式都指名下一個 `static`（`static A: i32 = B; static B: i32 = A;`），是編譯錯誤。
- 經由函式呼叫的循環不在編譯期檢查；它會在執行時 panic。

`const` 初始化式的順序檢查在編譯期進行。非 `const` 初始化式在第一次存取時初始化。初始化是執行緒安全的；初始化式恰好執行一次，同時存取的其他執行緒會等待它完成。透過呼叫讀取自身 `static` 的初始化式會 panic。

## 5.3.7. 可見性

和其他項目一樣，`static` 可以帶有可見性關鍵字。

```nabi
pub static VERSION: str = "1.0.0";
pub(pkg) static INTERNAL: i32 = 42i32;
static LOCAL_ONLY: bool = false;
```

詳細的可見性規則定義於[可見性](506-visibility.md)。

## 5.3.8. 與方法 / trait 的組合

`static` 是容納值的槽位，而不是定義項目者。因此它不會出現在 trait 或 `impl` 區塊的定義中；那裡改放 `const` 或關聯函式。

```nabi
impl Counter {
    static GLOBAL: i32 = 0i32;     // ERROR: no static inside impl
    const ZERO: i32 = 0i32;        // OK: associated const
}
```

## 5.3.9. 初始化式：編譯期或第一次使用

若初始化式是常數運算式（[5.2.4](502-const.md#524-constant-expressions-and-the-evaluator)），就在編譯期初始化。其他任何初始化式在第一次使用時初始化一次（延遲初始化）。延遲初始化是執行緒安全的，同時存取的執行緒會等待初始化完成。初始化式中的函式不必是 `const fn`（`Mutex::new` 等）。

```nabi
static PI: f64 = 3.14159f64;             // const: compile time
static START: Instant = Instant::now();   // initialized once at first access
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

## 5.3.10. `extern static`：FFI

如上所示，`extern "C"` 區塊中的 `static` 從 Nabi 參照在外部定義的全域變數；不寫初始化式。

```nabi
extern "C" {
    static environ: CPtr<CPtr<c_char>>;   // externally defined, read-only from nabi
}
```

詳細規則定義於 [`extern` 區塊](509-extern-block.md)。

## 5.3.11. 生命週期與 GC 根

`static` 的記憶體持續存在到程式終止（`const` 初始化式在啟動時初始化，其他在第一次存取時）。`static` 中每個容納 GC 參照的字組，都在程式啟動時註冊為 GC 根（`nabi_gc_add_root`，[8.5](../008-abi/805-extern-symbols.md)），因此 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 會讓 `static` 所參照的物件保持存活。

其他執行緒所看到的 `static` 遵循記憶體模型（[7.9](../007-runtime/709-memory-model.md)）。GC 與執行環境的行為在 [7. 執行環境模型](../007-runtime/README.md)與 [8. 執行環境 ABI](../008-abi/README.md) 中說明。
