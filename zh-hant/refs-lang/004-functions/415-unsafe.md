# 4.15. `unsafe`：函式修飾詞與區塊運算式

`unsafe` 關鍵字以兩種形式出現，用來把編譯器不保證安全性的區域明顯地圈起來：**函式 / 方法宣告上的修飾詞**，以及**區塊運算式**。兩者成對運作：宣告為 `unsafe fn` 的函式必須在 `unsafe { ... }` 區塊內呼叫。

```
unsafe_fn        ::= "unsafe" fn_decl
unsafe_block     ::= "unsafe" block_expr
```

這個模型與 Rust 的 `unsafe` 同構；其目的是在程式碼中明確標出危險的區域。

## 4.15.1. `unsafe fn`：函式修飾詞

在函式或方法宣告之前放上 `unsafe`，表示該函式的主體超出安全抽象的保證。簽章直接說明呼叫者要負責滿足前置條件。

```nabi
impl<T> Ptr<T> {
    unsafe fn read(self) -> T;
    unsafe fn write(self, value: T);
}

extern "C" {
    unsafe fn strlen(s: ConstPtr<c_char>) -> usize;
}

unsafe fn transmute_unchecked<A, B>(a: A) -> B { ... }
```

在 `unsafe fn` 本身的主體內，不需要另外的 `unsafe { ... }` 區塊就能進行不安全的呼叫。由於簽章已經宣告了 unsafe，整個主體都被視為不安全作用域。

```nabi
unsafe fn copy_two<T>(p: Ptr<T>) -> (T, T) {
    let a = p.read();              // body is an unsafe scope: no explicit block needed
    let b = p.add(1).read();
    (a, b)
}
```

### 與方法接收者一起

`impl` 中的方法遵循相同的形式。

```nabi
impl Buffer {
    unsafe fn raw_data(self) -> Ptr<u8>;
    unsafe fn raw_data_mut(ref self) -> Ptr<u8>;
}
```

`unsafe` 與 `ref self` / `self` 等接收者記法不衝突，並位於函式標頭的最前面。

### 在 `extern "C"` 區塊內

使用者宣告的 C ABI 函式全都隱含為 `unsafe`，因為它們的定義不在編譯器的控制範圍內（[extern 區塊](../005-modules/509-extern-block.md)）。建議明確寫出（`unsafe fn`）；若省略，會自動被視為 `unsafe`。

每個 `extern` 函式都是如此，包括執行環境符號的 `extern "nabi"` 宣告：在不安全作用域之外呼叫它是錯誤（E0061）。

```nabi
extern "C" {
    fn malloc(size: usize) -> CPtr<u8>;        // treated as unsafe automatically
    unsafe fn free(ptr: CPtr<u8>);             // explicit (recommended)
}
```

## 4.15.2. `unsafe { ... }`：區塊運算式

呼叫不安全函式或解參照原始指標等操作，只允許在 `unsafe { ... }` 區塊內進行。與一般的[區塊運算式](../003-expressions/309-block-expressions.md)相同，區塊的結果是其最後一個運算式的值。

```nabi
let n: usize = unsafe { strlen(cs.as_ptr()) };

let v: i32 = unsafe {
    let p = ptr.add(idx);
    p.read()
};

unsafe {
    free(ptr);
}
```

### 區塊中允許的操作

在 `unsafe { ... }` 區塊內，額外允許下列操作：

- 呼叫宣告為 `unsafe fn` 的函式 / 方法
- 解參照原始指標（`Ptr<T>`、`ConstPtr<T>`、`ManyPtr<T>`）（`p.read()`、`p.write(v)`）
- 直接呼叫 [`bare::*`](../../refs-standard-libraries/100-bare/README.md) 模組的危險函式
- 讀取 `union` 欄位（位元重新解讀，[union](../002-types/205-union.md)）。寫入欄位不需要 `unsafe` 就是允許的。
- 在值被交給另一個擁有者之後使用它（[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)）：區塊的作者保證沒有其他執行緒同時使用它。

其他所有一般的安全操作在不安全區塊內的運作方式都相同：`unsafe` *增加能力*；它不*改變既有的意義*。

### 縮小範圍

`unsafe` 區塊應當盡可能保持窄小。與其把整個函式包在 `unsafe { ... }` 中，不如只包住真正危險的那一行，讓其他各行的安全性一目了然。

```nabi
// recommended: narrow unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    let validated = try check_ptr(p);
    unsafe { validated.read() }    // only the dangerous operation is unsafe
}

// not recommended: wide unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    unsafe {
        let validated = try check_ptr(p);
        validated.read()
    }
}
```

## 4.15.3. 呼叫規則

`unsafe fn` 的呼叫遵循下列規則。

| 呼叫位置                       | 能否呼叫 `unsafe fn`？       |
|--------------------------------|------------------------------|
| 一般函式主體                   | ✗（編譯錯誤）                |
| `unsafe fn` 主體               | ✓（自動的不安全作用域）      |
| `unsafe { ... }` 區塊          | ✓                            |

違反規則是編譯錯誤：

```nabi
fn safe_caller(p: Ptr<i32>) -> i32 {
    p.read()    // ERROR (E0061): call to unsafe function `read` outside an `unsafe` block
}
```

有兩種修正方式：把呼叫包在 `unsafe { ... }` 中，或把呼叫者本身標示為 `unsafe fn`。

### `unsafe fn` 型別

當 `unsafe fn` 或 C 函式被用作值時，其型別是 `unsafe fn(..) -> R`（[4.4](404-function-type.md)）。呼叫這種型別的值需要 `unsafe`。安全的 `fn(..) -> R` 值可以用在 `unsafe fn(..) -> R` 位置，反之則不行。

```nabi
let f: unsafe fn(usize) -> CPtr<u8> = malloc;   // C function value
let p = unsafe { f(16) };
let g: unsafe fn(i32) -> i32 = |x| x + 1;      // OK: safe → unsafe
let h: fn(usize) -> CPtr<u8> = f;              // ERROR: unsafe → safe
```

## 4.15.4. 在 trait 中

trait 方法也可以使用 `unsafe fn`。`impl` 一方必須以相同的簽章（包括 `unsafe`）實作它們：對 `unsafe fn` 方法提供安全的 impl，或對安全的方法提供 `unsafe fn` impl，是錯誤（E0074）。呼叫者遵循與一般呼叫位置相同的規則。

```nabi
trait RawAccess {
    unsafe fn raw_get(self, idx: usize) -> u8;
}

impl RawAccess for Buffer {
    unsafe fn raw_get(self, idx: usize) -> u8 {
        self.ptr.add(idx).read()
    }
}

let buf: Buffer = ...;
let b = unsafe { buf.raw_get(0) };
```

### `unsafe trait`

其 impl 承諾了編譯器無法檢查之事的 trait，宣告為 `unsafe trait`。它的每個 impl 都是 `unsafe impl`；一般的 `impl` 是編譯錯誤（E0078），錯誤訊息會給出 `unsafe impl` 形式。使用該 trait（約束、呼叫）不需要 `unsafe`。`bare::mem::Pod`（每一種位元模式都是一個值）就是一例（[stdlib 1.3.3](../../refs-standard-libraries/100-bare/103-mem.md#133-the-pod-trait)）。

```nabi
/// Every all-zero bit pattern is a value.
unsafe trait Zeroable {
    fn zeroed() -> Self;
}

unsafe impl Zeroable for i32 {
    fn zeroed() -> i32 { 0 }
}
```

### `unsafe impl`

`unsafe impl` 實作 `unsafe trait`，或手動實作自動標記 trait `Send` / `Sync`。它標示作者保證該 trait 所承諾之事（對 `Send` / `Sync` 而言，即該型別可以安全地跨越執行緒邊界）。對其他任何 trait 使用 `unsafe impl` 是編譯錯誤（[2.9.1](../002-types/209-impl.md#291-兩種形式)）。

```nabi
unsafe impl Send for Handle { }
```

## 4.15.5. 責任模型

`unsafe` 不表示編譯器*放棄*安全性檢查。它表示下列兩項責任轉移給呼叫者：

1. **記憶體安全**：不造成懸置指標、未對齊存取或資料競爭等未定義行為（[4.15.8](#4158-未定義行為)）的責任。
2. **前置條件**：滿足 `unsafe fn` 文件所承諾之使用條件（例如指標指向有效物件、長度在緩衝區之內）的責任。

`unsafe { ... }` 是呼叫者接受這兩項責任的可見訊號，因此在程式碼審查時，危險的區域一眼就能看出。

## 4.15.6. 與隱含 unsafe 的關係

下列位置不需要明確的 `unsafe`，就會自動被視為不安全作用域：

- `unsafe fn` 的主體

這種自動處理減少了在常見危險區域中反覆寫 `unsafe { ... }` 的雜訊。

## 4.15.7. 與其他語言的比較

| 語言     | unsafe 記法模型                                             |
|----------|-------------------------------------------------------------|
| Rust     | `unsafe fn` + `unsafe { ... }`（同構）                      |
| C / C++  | 沒有另外的記法；所有程式碼都可能不安全                      |
| Swift    | `@unsafe` 標註 / `withUnsafePointer` 函式式模式             |
| Go       | `unsafe` 套件（另外匯入）；沒有區塊記法                     |
| Zig      | `@ptrCast` 等明確的內建函式；沒有區塊記法                   |
| **Nabi**  | **採用 Rust 風格的 `unsafe fn` + `unsafe { ... }`**         |

Nabi 原樣沿用 Rust 的模型。危險區域的視覺區隔一致地以單一關鍵字標示，對程式碼審查與靜態分析都有益。

## 4.15.8. 未定義行為

以下是 Nabi 中未定義行為的完整清單。執行時做了其中任何一件事的程式沒有意義：編譯器可以假設它永遠不會發生，之後發生任何事都有可能。每一項都需要 `unsafe` 程式碼（或外部程式碼）才會發生；安全的 Nabi 無法造成其中任何一項。每個指出未定義行為的函式庫章節，指出的都是其中某一類。

記憶體與指標：

1. 透過不指向其型別之存活記憶體的指標讀取或寫入：null、懸置（它來源的區域變數、釘選或配置已不存在）、超出其物件的邊界，或超出呼叫者擔保的 `ManyPtr` / `GcBuf` / `Slice::from_raw_parts` 範圍（[stdlib 1.1](../../refs-standard-libraries/100-bare/101-ptr.md)、[stdlib 1.3.9](../../refs-standard-libraries/100-bare/103-mem.md)）。
2. 在使原始指標有效的釘選被釋放之後，使用指向 GC 物件的該原始指標（[stdlib 1.6](../../refs-standard-libraries/100-bare/106-gc-pin.md)）。
3. 透過未對齊的指標讀取或寫入，例如 `ptr_cast` 可能產生的指標（[stdlib 1.2.2](../../refs-standard-libraries/100-bare/102-cast.md)）。
4. 解參照一個遺失了來源資訊（provenance）的指標，但由 `from_exposed_provenance` 從已公開的位址建立的指標除外（[stdlib 1.1.2](../../refs-standard-libraries/100-bare/101-ptr.md)）。
5. 透過指向非 `mut` `static` 的指標寫入，或透過 `const_cast` 的結果寫入不可寫的記憶體（[stdlib 1.1.2](../../refs-standard-libraries/100-bare/101-ptr.md)、[stdlib 1.2.4](../../refs-standard-libraries/100-bare/102-cast.md)）。
6. 對重疊的範圍使用 `copy_nonoverlapping`（[stdlib 1.3.4](../../refs-standard-libraries/100-bare/103-mem.md)）。
7. 在 `drop_in_place` 執行了值的 drop glue 之後使用該值，或執行它兩次（[stdlib 1.3.4](../../refs-standard-libraries/100-bare/103-mem.md)）。

值：

8. 以任何型別讀取未初始化的記憶體：在值寫入之前呼叫 `MaybeUninit::assume_init`，或讀取沒有任何東西寫入過的位元組（[stdlib 1.3](../../refs-standard-libraries/100-bare/103-mem.md)）。
9. 透過 `union` 讀取（[2.5.2](../002-types/205-union.md)）、`transmute_unchecked`、原始寫入或虛假的 `unsafe impl Pod`，產生對其型別無效的值：`0` / `1` 以外的 `bool`、不是 Unicode 純量值的 `char`、沒有任何變體具有的列舉或和型別標籤、既不是 null 也不是存活物件的 GC 參照、為 null 的非 `none` `extern "C" fn`（[stdlib 1.2.6](../../refs-standard-libraries/100-bare/102-cast.md)、[5.9.4](../005-modules/509-extern-block.md)）。
10. 破壞以 `new_unchecked` 建立之受檢包裝的不變條件：為零的 `NonZero`、為 NaN 的 `NonNan`、為 NaN 或無限的 `Finite`、為負的 `Positive`（[stdlib 2.2.6](../../refs-standard-libraries/200-core/202-non-zero.md)）。
11. 在回收器不掃描的 GC 堆積之外的記憶體中存放 GC 參照：在這種記憶體上、容納 GC 參照的 `ManyPtr::slice`，或以原始寫入存放在那裡的 GC 參照（[stdlib 1.1.4](../../refs-standard-libraries/100-bare/101-ptr.md)）。

執行緒：

12. 資料競爭（[7.9.4](../007-runtime/709-memory-model.md#794-資料競爭)）：透過原始指標或 `GcBuf`、虛假的 `unsafe impl Send` / `Sync`、對其他東西也能存取的值使用 `Mutex::new_unshared`、在 `unsafe` 內略過交接檢查，或在 `env::set_var` 執行期間存取環境的 C 程式碼（[stdlib 3.5](../../refs-standard-libraries/300-std/305-env.md)）。

外部程式碼：

13. 宣告的型別與外部定義不相符的外部函式、`static` 或回呼，或在 C 持有原始位元之處傳入 GC 值（[5.9.4](../005-modules/509-extern-block.md)、[5.9.6](../005-modules/509-extern-block.md)）。
14. 來自外部程式碼的例外或展開穿越 Nabi 框架（[5.9.2](../005-modules/509-extern-block.md)）。
15. 在宣告為 `str` 之處回傳 null 的 `char*`；C 在呼叫回傳之後仍保留 `str` 引數的指標（[5.9.4](../005-modules/509-extern-block.md)）。
16. 對懸置的指標，或指向沒有 NUL 之位元組的指標使用 `CStr::from_ptr`（[stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md)）。

其他一切都是有定義的，無論在安全或不安全的程式碼中皆然。特別是下列都不是未定義行為：整數溢位（啟用檢查時 panic，停用時回繞，[stdlib 2.3](../../refs-standard-libraries/200-core/203-ops.md#overflow-behavior)）、集合的越界索引或切片（panic）、除以零（panic）、堆疊溢位（行程中止，[7](../007-runtime/README.md)）、每一種浮點運算，包括 NaN 與無限（[stdlib 2.1.4](../../refs-standard-libraries/200-core/201-numbers.md#floating-point-semantics)）、讀取 `union` 欄位（位元重新解讀；只有它所產生的值可能無效，見第 9 項）、大小不同之型別間的 `transmute_unchecked`（缺少的位元組讀作零）、死結，以及洩漏。
