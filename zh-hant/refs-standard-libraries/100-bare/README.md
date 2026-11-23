# 1. Bare 層級

bare 層級（`bare`）定義低階或危險的標準函式庫項目。原始指標、transmute、原始配置、GC 固定、FFI 基本操作等在安全抽象之外運作的工具，全都集中在這裡。

## 相依關係

向上，它只相依於執行環境透過執行環境 ABI `extern "nabi"` 公開的符號。向下，`core`、`std` 和使用者程式碼相依於它，但只能透過明確匯入。

## 使用者匯入政策

```nabi
use bare::ptr::CPtr;                             // explicit import required (danger signal)
use bare::alloc::{alloc, Layout};

let layout = try! Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // null on failure
```

隱含套用的 prelude 不包含不安全的項目。因此 `bare::*` 必須另外明確匯入，讓使用者進入危險領域的位置在程式碼中清楚可見。

## 模組內容

### 指標家族（4 種）

- [1.1. `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>` / `CPtr<T>`：Zig 風格、以型別編碼數量](101-ptr.md)
- [1.2. 強制轉型操作：`ptr_cast`、`align_cast`、`const_cast`、`reinterpret`](102-cast.md)

### 記憶體 / 轉換

- [1.3. `MaybeUninit<T>` / `transmute_unchecked`：位元層級的轉換](103-mem.md)
- [1.4. 原始配置（`alloc`、`dealloc`、`realloc`）](104-alloc.md)
- [1.5. 揮發性記憶體存取：`volatile::read` / `volatile::write`](105-volatile.md)

### GC 整合

- [1.6. `Pinned<T>` / `gc::pin`：不讓 GC 移動物件](106-gc-pin.md)

### FFI

- [1.7. extern "C" / `CStr` / `CString` / `CPtr`：C ABI 相容；C 型別、`Handle`、errno、寬字串、`Library`](107-ffi.md)
- [1.8. WASM Component Model 介面（`ffi::wit`）](108-wit.md)
- [1.10. Objective-C（`extern "objc"`、`ffi::objc`）](110-objc.md)

沒有 1.9 節。這個編號從缺，Objective-C 保留 1.10，以免指向它的連結失效。

## 核心設計原則

### 1. 嚴格來源（strict provenance）：採用 Rust 2024 模型

`Ptr<T>` 是（位址，來源）的配對，而不是單純的整數。因此 `usize ↔ Ptr<T>` 的來回轉換會失去來源，只能透過明確的逃生門進行。

```nabi
let p: Ptr<i32> = ...;
let addr: usize = p.addr();              // safe: address only
let q: Ptr<i32> = p.with_addr(addr + 4); // safe: provenance kept, only the address changes
let r: Ptr<i32> = unsafe { bare::ptr::from_exposed_provenance(addr) };  // unsafe: fresh provenance
```

指標永遠不能使用 `as` 強制轉型。這個方向的依據是 [Ralf Jung「Pointers Are Complicated」](https://www.ralfj.de/blog/2022/04/11/provenance-exposed.html)，並與 Rust 1.84（2024 年 12 月）穩定化的模型同構。

### 2. 數量以型別編碼：Zig 模型

| 型別          | 意義                                     | C 對應          |
|---------------|------------------------------------------|-----------------|
| `Ptr<T>`      | 恰好**一個** T                           | `T*`（單一）    |
| `ConstPtr<T>` | 恰好**一個** const T                     | `const T*`      |
| `ManyPtr<T>`  | **N 個連續的** T（長度未知，與 C 相容） | `T*`（陣列） |
| `Slice<T>`    | **N 個連續的** T（含長度）               | (ptr, len)      |
| `CPtr<T>`     | 與 C 相容、**可為 null** 的指標          | `T*`（可為 null） |

`Slice<T>` 是在 [2.9 collections](../200-core/209-collections.md) 中定義的安全檢視，不屬於 bare 層級。本章只涵蓋 `Slice` 與指標之間的原始路徑：`ManyPtr::slice`，以及串列資料指標的固定。

### 3. GC 與原始指標：兩個分開的型別層級

| 型別       | 意義                                       | GC 移動物件時 |
|------------|--------------------------------------------|------------------------------|
| `GcPtr<T>` | 指向 GC 堆積上物件的參考（通常是 `T`） | **自動更新** |
| `Ptr<T>`   | 原始指標（非受管或已固定）                 | **失效**                  |

這個區分沿用 Go 1.21 的 `runtime.Pinner` 模型。原始 `Ptr<T>` 只在 GC 固定期間有效，因此在固定解除後使用它是 UB。

```nabi
let obj: List<i32> = [1, 2, 3];                // GcPtr (automatic)
using pin = bare::gc::pin(obj) {          // pin begins
    let raw: Ptr<i32> = pin.as_data_ptr();
    ffi_function(raw, obj.len());
}                                              // pin released, raw invalid
```

### 4. 強制轉型只以明確命名的操作進行

C 和 Rust 中對指標使用 `as` 強制轉型的陷阱是最大的危險。因此 Nabi 只以明確命名的函式公開每一種不安全的強制轉型。

| C/Rust 寫法       | Nabi 寫法                                | 意義                           |
|-------------------|------------------------------------------|--------------------------------|
| `ptr as *U`       | `bare::ptr_cast<U>(p)`               | 只變更型別（保留來源） |
| `ptr as *Aligned` | `bare::align_cast(p)`                | 對齊向上轉型（未對齊時為 `none`） |
| `*const → *mut`   | `bare::const_cast(p)`                | 移除 const                     |
| `transmute<A,B>`  | `bare::reinterpret<A:Pod, B:Pod>(a)` | 位元重新解讀（強制 Pod） |

### 5. 以 `using` 區塊達成 RAII：不依賴終結器

每個 GC 語言都學到了：在 GC 環境中終結器並不可靠。因此檔案、通訊端、互斥鎖等外部資源以確定性的方式釋放，絕不交給 GC：`Drop` 型別是仿射（affine）的，編譯器會在它最後一個擁有者離開作用域的位置（在每一個離開路徑上）或在其 `using` 區塊結束時釋放它，而關閉可能失敗的資源必須明確關閉。GC 永遠不會呼叫 `drop`，也沒有終結器（[stdlib 2.13.4](../200-core/213-marker.md#2134-drop釋放時的清理)）。

```nabi
use bare::ptr::{ConstPtr, CPtr};

extern "C" {
    fn fopen(path: ConstPtr<c_char>, mode: ConstPtr<c_char>) -> CPtr<u8>;
    fn fclose(f: CPtr<u8>) -> i32;
}

// a user-written wrapper: Drop closes the C handle
struct File { raw: CPtr<u8>; }

impl Drop for File {
    fn drop(ref self) { unsafe { fclose(self.raw) }; }
}

using f = File { raw: opened } {   // opened: non-null CPtr<u8> from fopen
    // ... use f.raw ...
}     // ← _guaranteed_ fclose at end of scope (RAII via Drop)

// ordinary GC objects need no using; the GC reclaims them
let xs: List<i32> = [1, 2, 3];     // GC automatic
```

依據是 [2.13 marker](../200-core/213-marker.md) 的 `Drop` trait；`using` 區塊的語法依據是 [refs-lang 3.17 using 運算式](../../refs-lang/003-expressions/317-using-expression.md)。
