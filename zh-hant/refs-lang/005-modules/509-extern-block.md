# 5.9. `extern` 區塊（[FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 宣告）

`extern` 關鍵字（上下文關鍵字，只在項目開頭是關鍵字，[1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)）與 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 指定符（`"C"` 等）結合，定義外部函式與全域變數的宣告區塊。它用於與外部函式庫、系統呼叫及其他語言介接。

```
extern_block    ::= "extern" string_lit "{" extern_item* "}"
extern_item     ::= visibility? "fn" identifier "(" param_list? ")" return_type? ";"
                  | visibility? "static" "mut"? identifier ":" type_expr ";"
```

## 5.9.1. 基本形式

基本用法：

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
    fn strlen(s: ConstPtr<c_char>) -> usize;

    static environ: Ptr<Ptr<c_char>>;
    static mut atexit_count: u64;
}
```

區塊由 `extern`、ABI 字串（`"C"`、`"system"` 等；各 ABI 的意義定義於[呼叫慣例](../008-abi/806-calling-convention.md)）以及 `{ ... }` 組成。主體只能包含有簽章而沒有主體的函式宣告，以及 `static` 宣告。

> [!IMPORTANT]
> `extern "C"` 區塊內的函式宣告**隱含為 unsafe**。呼叫者只能在 `unsafe { ... }` 區塊內呼叫它們；即使不在簽章中寫 `unsafe`，效果也相同。完整定義見 [4.15 unsafe](../004-functions/415-unsafe.md)。

## 5.9.2. ABI 指定符

支援的 ABI：

| ABI        | 意義                                                                             |
|------------|----------------------------------------------------------------------------------|
| `"C"`      | 平台的 C ABI。無論符號名稱為何都相同。                                           |
| `"system"` | 作業系統的系統呼叫 ABI。在某些平台上與 `"C"` 不同（例如 Win32 的 `stdcall`）。   |
| `"nabi"`    | 執行環境（`nabir`）的 ABI。見下文。                                               |
| `"objc"`   | Apple 目標上的 Objective-C 類別及其方法（[stdlib 1.10](../../refs-standard-libraries/100-bare/110-objc.md)）；`ffi::objc::import { "AppKit/AppKit.h" }` 從 SDK 的標頭檔寫出這些區塊（[stdlib 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-headers-ffiobjcimport)）。 |

在 `"system"` 與 `"C"` 相同的目標上（除 32 位元 x86 Windows 之外的每個支援目標），`"system"` 就是 `"C"`。在 32 位元 x86 Windows 上它是 stdcall（[5.9.10](#5910-extern-fn-definitions-exposed-externally)）。

`extern "nabi"` 是工具鏈所宣告之執行環境函式的 ABI；這些宣告是工具鏈套件的私有項目（大部分位於 `core::rt`；`bare` 宣告它自己使用的那些）。在 `extern "nabi"` 區塊上的 `#[no_safepoint]` 宣告其函式既不配置記憶體也不阻塞；對它們的呼叫不是安全點。它是工具鏈內部的屬性，用在 `extern "C"` 區塊上是錯誤。

| 值                              | 在 `extern "nabi"` 下的傳遞方式            |
|---------------------------------|------------------------------------------|
| `str`                           | 兩個字組 `(ptr, len)`                    |
| `fn` 值                         | `{code, env}`（[8.5.0a](../008-abi/805-extern-symbols.md#850a-nabifn---c-abi-form-of-function-values)） |
| `extern "C" fn(..)` 參數        | 沒有 env 的 C 函式指標                   |
| 其他任何東西                    | 原樣的值（與 C ABI 相同）                |

> [!NOTE]
> `"Rust"` 等其他語言的 ABI 通常不穩定，不是 `extern` 的目標。與其他語言介接，只能透過雙方都同意的穩定 ABI（通常是 `"C"`）。

> [!NOTE]
> Nabi 以回傳值傳播錯誤，不使用堆疊展開（[8.6.5](../008-abi/806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)）。因此來自外部語言的例外或展開穿越 Nabi 框架是未定義行為；外部函式必須在呼叫邊界內處理例外。

不帶 ABI 的單純 `extern { ... }` 是編譯錯誤；必須指定 ABI。

## 5.9.3. 主體中允許的項目

### 外部函式宣告

外部函式的宣告方式如下：

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
}
```

簽章與一般函式宣告形式相同，但沒有主體，並以 `;` 結束。參數與回傳型別只能使用與 FFI 相容的型別，例如基本整數、浮點數與指標。

### 外部 `static`

在外部定義的全域變數以下列方式參照：

```nabi
extern "C" {
    static environ: Ptr<Ptr<c_char>>;  // read-only
    static mut atexit_count: u64;      // mutable
}
```

這種形式參照在外部定義的全域變數；`static mut` 只允許出現在 `extern` 區塊內（[`static`](503-static.md)）。C 的 `errno` 不是這樣的變數：它是每個執行緒各自的巨集，因此 `static errno` 會讀到錯誤的記憶體。在宣告上加上 `#[errno]` 會在呼叫之後儲存它（[stdlib 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errno---errno-and-getlasterror)）。

### 不允許的項目

`struct`、`enum`、`impl`、`fn` 主體、`const` 等不會出現在 `extern` 區塊的主體中。該區塊只宣告與外部 ABI 的邊界。

## 5.9.4. 與 FFI 相容的型別

參數、回傳與 `static` 的型別都必須與 FFI 相容。FFI 型別如下：

- 基本整數：`i8` 到 `i64`、`u8` 到 `u64`、`usize`、`isize`。
- C 的具名型別，即這些基本型別的別名，大小與目標的 C 編譯器所定的相同：`c_int`、`c_uint`、`c_long`、`c_ulong`、`c_size_t`、`c_wchar` 以及 `bare::ffi::types` 的其餘部分，全都在 prelude 中（[stdlib 1.7.9](../../refs-standard-libraries/100-bare/107-ffi.md#179-c-types---c_int-c_long-c_void)）。`c_long` 在 Windows 與 32 位元目標上是 32 位元，因此 C 的 `long` 要宣告為 `c_long`，而不是 `i64`。`c_void` 代表指標背後的 `void`（`CPtr<c_void>`）。
- 基本浮點數：`f32`、`f64`。
- `bool`（C 的 `bool`）與 `c_char`（1 位元組）。
- `f16` / `bf16`：C 的 `_Float16` / `__bf16`，在其 C 具有它們的目標上（[8.6.3](../008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)）。
- 8 或 16 位元組的 `Simd<T, N>`，作為參數或結果：目標中該大小的 C 向量（`float32x4_t`、`__m128`），在 AAPCS64 與 x86-64 上（[8.6.3](../008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)）。
- 指標：`Ptr<T>`、`ConstPtr<T>`、`CPtr<T>`（`bare::ptr`，[標準函式庫層級](../000-meta/005-standard-libraries-tiers.md)中最低的一層）。
- 欄位都是 FFI 型別（包括由它們組成的固定長度陣列）的 `struct` 與 `union` 型別。預設配置就已經是 C 的配置；`#[repr(packed)]` 與 `#[repr(align(N))]` 像在 C 中那樣改變它。
- 帶有整數 `#[repr]`（`#[repr(i32)]`）的無欄位列舉：C 的 `enum`。
- 函式指標：`extern "C" fn(...) -> ...`（見下文的「C 函式指標型別」）。
- 函式型別的參數：`extern "C"` 宣告中的 `fn(..) -> R` 參數與 `extern "C" fn(..) -> R` 相同。引數必須指稱一個函式；閉包是錯誤（`` `each` is a C function: a `fn` argument must name a function (a C callback has no room for a closure's captures) ``），其說明會指出其狀態放在哪裡：`` a C callback cannot capture: keep the state in `Handle::new(..)` (`bare::ffi`), pass `h.as_ptr()` as the `void*` argument, and read it back in an `extern "C" fn` with `Handle::<T>::from_ptr(p).get()` ``（[stdlib 1.7.10](../../refs-standard-libraries/100-bare/107-ffi.md#1710-handlet---nabi-values-c-holds)）。
- 可為 null 的指標：`Ptr<T> | none`、`ConstPtr<T> | none`、`extern "C" fn(..) -> R | none`（帶有 null niche 的指標，[2.7.2](../002-types/207-niche-optimization.md#272-automatic-niches)），以及任何 `#[niche(0)]` 結構 `| none`（`NonZero<i32> | none` 是一個 `int`）；`none` 就是 null。
- 作為 `extern "C"` 宣告之參數的 `str`：依下列規則轉換。
- 作為結果的 `()` 與 `!`。

`extern "C"` 宣告中的 `str` 參數是單一個以 NUL 結尾的 `char*`。若位元組之後接著 NUL，就原樣傳遞那些位元組；否則（切片等）傳遞一份在呼叫期間釘選的複本。C 不得在呼叫回傳之後保留那個指標。`str` 結果是錯誤：回傳之 `char*` 的位元組屬於 C（下一次呼叫會覆寫的靜態緩衝區、呼叫者必須釋放的記憶體、指向隨呼叫消失之引數複本內部的指標），因此建立在它們之上的 `str` 可能改變或懸置，而且它們不一定是 UTF-8。請把結果宣告為 `ConstPtr<c_char> | none`，並以 `CStr::from_ptr` 複製字串（[stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md#172-cstr--cstring---null-terminated-string)），它也會說明是否為 UTF-8：

```nabi
extern "C" {
    fn getenv(name: str) -> ConstPtr<c_char> | none;
}
let home = match unsafe { getenv("HOME") } {
    p: ConstPtr<c_char> => unsafe { CStr::from_ptr(p) },
    none => none,
};
```

模組 `m` 中 `extern` 區塊的 `pub` 函式也可以透過 `m::f` 存取；其符號不帶前綴。`extern "C"` 區塊中的 `nabi_*` 名稱不得接受或回傳 `str`（[8.5](../008-abi/805-extern-symbols.md)）。

在 `extern "C"` 與 `extern "system"` 區塊的參數、結果與 `static` 中，以及 `extern "C" fn` 定義中，其他任何型別都是錯誤。檢查在解析後的型別上進行，因此能看穿別名（`type S = str;`）、泛型實例、欄位或陣列元素。每個錯誤都會說明應改寫成什麼：

| 型別 | 原因 | 改寫成 |
|------|-----|---------------|
| GC 型別（`List`、`Dict`、閉包、`any`、容納其中之一的結構） | 在 C 持有它時，回收器可能移動或釋放它 | 基本型別、`CPtr<T>` 與長度，或由它們組成的結構；以 `gc::pin` 釘選並以 `p.as_data_ptr()` 與 `xs.len()` 傳遞的 `List<T>` |
| 作為結果的 `str`，或 `extern "C" fn` 定義中的 `str` | 位元組屬於 C（見上文）；C 傳遞的是 `char*` | `ConstPtr<c_char> \| none`，以 `CStr::from_ptr` 複製 |
| `char` | C 沒有四位元組的字元 | `u32` 與 `char::from_u32` |
| 元組 | C 沒有元組 | `struct` |
| 一般列舉 | 其配置由編譯器決定 | `#[repr(i32)]`，或由程式比對的 `i32` |
| 帶資料的列舉 | C 沒有帶標籤的列舉 | 由標籤與 `union` 組成的結構 |
| 陣列參數或結果 | C 以指標傳遞陣列 | `ConstPtr<T>`（`addr_of(xs)`），或放在結構中的陣列 |
| 其他和型別（`i32 \| none`） | 只有 null 指標有 C 形式 | `ConstPtr<T> \| none`，或值加上旗標 |
| 宣告參數以外的 nabi `fn` 值 | 它是函式加上其擷取 | 由具名 `extern "C" fn` 建立的 `extern "C" fn(..)` |
| `Mask<T, N>` | 沒有 C 型別 | 整數的 `Simd`（`mask.select(Simd::splat(1), Simd::splat(0))`） |
| 以值傳遞、容納 `Simd` 的結構 | 聚合型別中的 C 向量不會被傳遞 | 指向該結構的指標 |

訊息會指出參數與型別：`` parameter `xs` of extern function `f` has type `List<i64>`, which C cannot hold: the collector may move or free it ``。對 `List<T>` 而言，說明就是應寫的呼叫：`` help: C takes a list as a pointer and a length: declare `ptr: ConstPtr<i64>, len: usize`, and pass `p.as_data_ptr()` and `xs.len()` from `let p = gc::pin(xs);` (`bare::gc`) ``。若目標的 C 沒有 `_Float16`、沒有 `__bf16`，或沒有該大小的向量，錯誤會在為該目標產生程式碼時出現，並附帶同類的說明。編譯器無法檢查它看不到的東西：C 一方是否以相同型別宣告相同的函式，仍由程式負責確保。各型別的呼叫慣例定義於[呼叫慣例](../008-abi/806-calling-convention.md)。

### C 函式指標型別

```
c_fn_type ::= "unsafe"? "extern" ("\"C\"" | "\"system\"") "fn" "(" (type_expr ("," type_expr)*)? ")" ("->" unary_type)?
```

- ABI 字串只能是 `"C"` 或 `"system"`。其他任何字串都是錯誤。
- `extern "system" fn(..) -> R` 遵循與 `extern "C" fn(..) -> R` 相同的規則，因此可以表達 Win32 回呼。它只在 32 位元 x86 Windows 上是不同的型別，在那裡它是 stdcall；在其他地方兩者是同一個型別。
- 這種型別的值只能由具名的 C ABI 函式建立。`extern` 區塊中 C 宣告的值，其型別是 `unsafe extern "C" fn(..)`，不能放在安全（非 `unsafe`）的位置；Nabi 的 `extern "C" fn` 定義（其 C 進入點，5.9.10）則產生安全的 `extern "C" fn(..)` 值。在那個位置放一般的 `fn` 是錯誤，`` only an `extern "C"` function (an `extern` block declaration or an `extern "C" fn` definition) is an `extern "C" fn(..)` value ``，說明 `` C calls it with the C ABI: declare it `extern "C" fn twice(..)` ``。
- Nabi 的 `fn` 值（包括閉包與函式型別的變數）與 C 函式指標之間不會互相轉換。
- 透過不帶 `unsafe` 的型別呼叫是安全的；透過 `unsafe extern "C" fn` 呼叫需要 `unsafe`。
- 非 `none` 的 C 函式指標值不是 null。因此在不帶 `| none` 的 `extern "C" fn` 位置（參數、回傳、欄位）接收 NULL 是未定義行為，而可能回傳 NULL 的 C API 必須以 `| none` 宣告。

#### 可為 null 的 C 函式指標：`extern "C" fn(..) -> R | none`

只由一個 C 函式指標型別與 `none` 組成（任何順序，沒有其他成員）的和型別使用 null niche（[2.7.2](../002-types/207-niche-optimization.md#272-automatic-niches)）。

- 它與指標一樣大，而 `none` 就是 null。
- 它與 C 可為 null 的函式指標具有相同的 ABI。它可以原樣用在 extern 參數 / 回傳、與 C 交換的結構欄位，以及 extern `static` 中。
- 以零填滿的儲存空間讀出來是 `none`。
- 包含它的較大和型別使用一般的帶標籤配置，兩者之間的轉換是一般的和型別轉換。

```nabi
extern "C" {
    fn cmp_i32(a: ConstPtr<u8>, b: ConstPtr<u8>) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32);
}
let f: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;   // a C declaration's value is unsafe
let g: extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;          // ERROR: safe position
```

### extern 函式的命名空間

在 `extern` 區塊中宣告的函式構成單一的全域命名空間，可以從任何模組以不帶前綴的名稱呼叫。這與在連結時位於單一全域空間的 C 符號相呼應。它是一般項目「不會隱含地搜尋祖先模組」規則（[5.7.1](507-paths.md#571-starting-identifier-of-a-path)）的例外，只適用於 extern 函式。以不同簽章兩次宣告同名的 extern 函式是編譯錯誤。工具鏈在 `core::rt` 中的執行環境宣告也算在內：重新宣告 `nabi_*` 執行環境函式的使用者 `extern` 必須與其簽章相符。

## 5.9.5. `Ptr<T>` / `ConstPtr<T>`

定義於 `bare::ptr` 的原始指標型別，`bare` 是[標準函式庫層級](../000-meta/005-standard-libraries-tiers.md)中最低的一層。它們不在 prelude 中：`use bare::ptr::{Ptr, ConstPtr};`。

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
    fn read_byte(p: ConstPtr<u8>) -> u8;
}
```

`Ptr<T>` 是對應 C 之 `T*` 的可變原始指標，`ConstPtr<T>` 是對應 C 之 `const T*` 的唯讀原始指標。兩者都絕不會是 null。可能為 null 的指標是 `Ptr<T> | none`（或 `ConstPtr<T> | none`）：一個位址，以 null 表示 `none`（[niche 最佳化](../002-types/207-niche-optimization.md)）。`CPtr<T>` 是可以把 NULL 當作值保存的 C 指標（`is_null()`、`as_ptr() -> Ptr<T> | none`）。

## 5.9.6. 呼叫 / 使用

在 `extern` 區塊中宣告的函式像一般 Nabi 函式一樣呼叫。

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
}

fn use_malloc() {
    let p = unsafe { malloc(1024usize) };
    if p is Ptr<u8> {
        // ... use ...
        unsafe { free(p) };
    }
}
```

每個 `extern` 函式（包括 `extern "nabi"`）都是不安全的，要在 `unsafe { ... }` 內呼叫（[4.15](../004-functions/415-unsafe.md)）；工具鏈的套件把它們所宣告的執行環境函式包裝在安全的函式中。除此之外，`extern` 函式呼叫與一般呼叫的差別只在於它是外部符號。Nabi 的一般語意適用於結果型別。`extern static` 像一般變數一樣讀取，（若是 `mut`）也像一般變數一樣寫入。Nabi 無法檢查 C 的全域變數，因此讀取或寫入它需要 `unsafe`（`unsafe` 區塊或 `unsafe fn` 主體）；在其外是錯誤，其說明會引用修正方式：`unsafe { optind = 6; }`。以 `addr_of` 取得其位址不會讀取任何東西，不需要 `unsafe`。其型別仍必須與 FFI 相容（5.9.4）：Nabi 以該型別讀取外部記憶體，因此 `static environ: List<str>;` 是未定義行為；請把它宣告為指標（`static environ: Ptr<Ptr<c_char>>;`）。

## 5.9.7. 可見性

`extern` 區塊的每個項目都可以帶有可見性關鍵字。

```nabi
extern "C" {
    pub fn public_api(x: i32) -> i32;       // public
        fn _internal(x: i32) -> i32;        // module-private
    pub static VERSION: i32;
}
```

`extern "C"` 區塊本身不接受可見性關鍵字；可見性依項目逐一給出。

## 5.9.8. `extern` 區塊本身的位置

`extern "C" { ... }` 區塊只能出現在模組層級。它不能放在函式或 `impl` 內。

```nabi
fn f() {
    extern "C" { ... }                     // ERROR: not inside a function
}
```

## 5.9.9. `#[link("libname")]`：指名函式庫

要指名外部函式庫，請使用 `#[link]` 屬性。

```nabi
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
    fn sqrt(x: f64) -> f64;
}
```

在 extern 區塊上的 `#[link("x")]` 或 `#[link(name = "x", kind = "static" | "dylib")]` 會把 `-lx` 傳給連結器。函式庫的搜尋路徑是 `-L <dir>` 目錄，然後是 `LIBRARY_PATH`。在 Apple 目標上，`kind = "static"` 以路徑連結在那裡找到的 `lib<name>.a`，而 `#[link(name = "Foundation", kind = "framework")]` 連結框架 `Foundation.framework`，在 `-F <dir>` 目錄中搜尋，然後是 SDK 的 `System/Library/Frameworks`；在其他目標上，框架是連結錯誤。nabic 的 `-l` 與原生套件的 `links` 以相同方式加入函式庫，`nabi.nabion` 的 `Dep::system` / `Dep::framework` 函式庫也是（[9.10.4b](../009-tools/910-nabi-frontend.md#9104b-native-packages)）。在 wasm32 上，`#[link(wasm_import_module = "m")]` 從核心模組 `m` 匯入該區塊的函式，而函式上的 `#[link_name = "sym"]` 設定其連結的符號或匯入名稱（[6.2.3](../006-meta-system/602-built-in-attributes.md#623-link-naming-an-external-library)）。沒有 `#[link]` 時，符號在連結時透過系統預設（`libc` 等）或其他機制解析，例如套件自己的 C 原始碼的目的檔，內建的 C 編譯器會編譯它們並與套件一起連結（[9.13.8](../009-tools/913-c-compiler.md#9138-c-sources-of-a-package)）；它們的函式和其他函式一樣在 `extern "C"` 區塊中宣告。`#[link]` 的行為定義於[內建屬性](../006-meta-system/602-built-in-attributes.md)。

## 5.9.10. `extern fn` 定義（對外公開）

前面各節的形式是從 nabi 呼叫外部函式。反向的形式則透過外部 ABI 公開 Nabi 函式。

```nabi
#[export]
pub extern "C" fn nabi_compute(x: c_int, y: c_int) -> c_int {
    x + y
}

#[export_name("nabi_area")]
pub extern "C" fn area(w: c_int, h: c_int) -> c_int { w * h }

extern "C" fn callback(arg: i32) -> i32 {
    arg * 2
}
```

在一般函式宣告上的 `extern "ABI"` 限定詞，標示該函式要編譯為與該 ABI 相容。`extern "C" fn` 定義保持其修飾後的符號名稱（[9.5](../009-tools/905-symbol-mangling.md)）；若要 C 能連結的名稱，請加上 `#[export]`，它以函式自己的名稱（`nabi_compute`）匯出，或加上 `#[export_name("sym")]`，它以 `sym` 匯出（`area` 匯出為 `nabi_area`）。匯出的函式是無法到達函式消除的根，而匯出的函式就是共用函式庫（`--crate-type=cdylib`，[9.3](../009-tools/903-package-archive.md)）所公開的東西。

`extern "ABI" fn` 定義接受 `"C"` 或 `"system"`；其他任何 ABI 字串都是錯誤。在 32 位元 x86 Windows 上，`extern "system"` 宣告或定義是 stdcall：由被呼叫者彈出其引數，符號是 `_name@N`，與 MSVC 相同。

`extern "C"` 與 `extern "system"` fn 定義有下列限制：

- 它們不能是泛型、`async` 或 `throws`。
- 參數與回傳型別必須是 5.9.4 中與 FFI 相容的型別；不能使用 `str` 等非 FFI 型別。（5.9.4 的 `str` 轉換只適用於*呼叫* C 函式的宣告。）

`#[export]` 與 `#[export_name]` 只能用在 `extern "C"` 與 `extern "system"` fn 定義上。C 以 C 慣例呼叫匯出的函式，而 Nabi ABI 的函式會把其聚合型別、`str` 與窄整數處理錯，因此在一般函式上使用其中任一者都是錯誤：`` `compute` is exported to C but uses the Nabi ABI: write `pub extern "C" fn compute(..)` ``。在 wasm32 上，匯出就是同名的 wasm 匯出。

```nabi
#[export_name("on_event")]
extern "C" fn on_event(code: i32) -> i32 { code }     // visible from C as on_event
extern "C" fn bad(s: str) { }                          // ERROR: str cannot be used
#[export]
pub fn compute(x: i64) -> i64 { x * 2 }               // ERROR: exported, but the Nabi ABI
```

### `#[on_panic]`：給 C 一個值而不是中止

```nabi
#[on_panic(return = 0)]
extern "C" fn cmp(a: Ptr<i32>, b: Ptr<i32>) -> c_int { .. }

#[on_panic(return = -1.5)]
extern "C" fn half(x: f64) -> f64 { .. }

#[on_panic]
extern "C" fn notify(x: i32) { .. }
```

會穿越 C 的框架離開 C 所呼叫之函式的 panic 會中止行程（5.9.12）。在 `extern "C"` 或 `extern "system"` fn 定義上的 `#[on_panic(return = v)]` 則改為恢復：panic 照常回報，panic 掛鉤也會執行（[stdlib 3.12](../../refs-standard-libraries/300-std/312-panic.md)），然後該函式把 `v` 回傳給其呼叫者，無論是 C 還是 Nabi。`v` 是結果型別的字面值：整數、浮點數、`true` / `false`，或對指標結果而言的 `0`（null 指標）。沒有結果的函式只寫 `#[on_panic]`。恢復的函式是 panic 之下最近的 C 進入點：當 C 在更深處再次呼叫回 Nabi 時，由那個內層的進入點決定。

panic 與該函式之間的框架會被略過。它們的 `Drop` 不會執行，因此它們本應釋放的東西會洩漏，而在被略過的框架中鎖住的 `Mutex` 會保持鎖住：下一次對它的 `lock` 會永遠等待。在持有鎖時可能失敗的函式，應回傳錯誤值而不是 panic。恢復是以 `setjmp` / `longjmp` 實現的（[8.5.12](../008-abi/805-extern-symbols.md#8512-auxiliary-symbols-appendix-non-normative)）；wasm32 與沒有作業系統的目標沒有它，在那裡 panic 會像沒有該屬性時一樣中止。

每一種誤用都是錯誤，錯誤訊息會說明應寫什麼：

| 情況 | 訊息 |
|------|---------|
| 不是數字、`bool` 或 `0` 的值 | `` `#[on_panic]` on `count`: only a number, a `bool` or `0` (a null pointer) can be returned; write `#[on_panic(return = -1)]` with a value of the result type `` |
| 為沒有結果的函式給出值 | `` `#[on_panic]` on `nothing`: it returns nothing, so write `#[on_panic]` alone `` |
| 有結果的函式沒有給出值 | `` `#[on_panic]` on `half`: a function with a result needs the value it returns `` |
| 不是 `extern "C"` 的函式 | `` `#[on_panic]` is for functions C calls: write `#[on_panic(return = -1)] extern "C" fn plain(..)` `` |

呼叫外部函式與對外公開函式，一起構成 FFI 的兩個方向。

## 5.9.11. `extern fn` 的呼叫規則

`extern "C"` 函式遵循 C 呼叫慣例，可能與[執行引擎與使用者介面](../000-meta/001-execution-modes.md)中所涵蓋的一般函式呼叫規則有部分不同。呼叫慣例的差異在執行環境 ABI 規格（第 7 / 8 章）中說明。

## 5.9.12. FFI 安全的責任

`extern` 區塊的每個項目都是使用者的責任（與[標準函式庫層級](../000-meta/005-standard-libraries-tiers.md)的 `unsafe` 精神一致）。編譯器會驗證：

- 簽章的形式。
- 每個型別都是 FFI 型別（5.9.4），會看穿別名、泛型與欄位，以及目標的 C 能傳遞它。

編譯器不會驗證：

- 外部函式的實際行為（回傳 null、回傳錯誤、記憶體洩漏等）。
- 外部 `static` 的並行安全性。
- 外部函式庫真正的 ABI 相容性。

使用者必須保證這些。

### C 框架之下的 panic

panic 不得穿越 C 的框架離開 C 所呼叫的函式：C 對它一無所知，而 `exit` 會在損壞的狀態上執行主機的處理函式（`atexit`、直譯器的關閉流程）。因此在外部程式碼所呼叫之 `extern "C"` 或 `extern "system"` fn 中，或在其下的 Nabi 程式碼中發生的 panic，會印出其報告（[4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors)），然後印出這一行

```text
panic in a function C called: aborting (`#[on_panic(return = v)]` on the `extern "C" fn` returns `v` to C instead)
```

並中止行程（`SIGABRT`，在 Unix 上結束狀態為 134）。其下沒有外部框架的 panic，會和在其他任何地方一樣以結束碼 101 結束行程。共用函式庫（`--crate-type=cdylib`）遵循相同的規則，因此 Python、R 或 Julia 之類的主機永遠不會看到其行程 `exit`。`#[on_panic]`（5.9.10）把 panic 轉成回傳值，而 `std::panic::set_hook`（[stdlib 3.12](../../refs-standard-libraries/300-std/312-panic.md)）會在中止、結束或恢復之前看到每一次 panic。
