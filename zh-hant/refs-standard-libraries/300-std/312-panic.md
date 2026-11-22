# 3.12. `panic::`：panic 鉤子

`std::panic` 讓程式在行程結束之前看到每一次 panic：用來記錄它、排清檔案，或通知主機。panic 無法被攔截（[refs-lang 4.14.13](../../refs-lang/004-functions/414-throws.md#41413-panic-與致命錯誤)）；鉤子在退出的途中執行。

```nabi
// std::panic
pub struct PanicInfo {
    pub message: str;           // the panic's message
    pub location: str | none;   // "path:line:col", when known
}

pub fn set_hook(hook: fn(PanicInfo));
pub fn take_hook();
```

## 3.12.1. `set_hook` / `take_hook`

`set_hook(hook)` 讓 `hook` 從此在行程中的每一次 panic 時執行：在觸發 panic 的執行緒上，於報告印出之後、行程結束（代碼 101）、中止（C 框架下的 panic，[refs-lang 5.9.12](../../refs-lang/005-modules/509-extern-block.md#5912-ffi-安全的責任)）或復原（`#[on_panic]`，[refs-lang 5.9.10](../../refs-lang/005-modules/509-extern-block.md#on_panic給-c-一個值而不是中止)）之前執行。鉤子可以是任何 `fn(PanicInfo)` 值，包括閉包。之後的 `set_hook` 會取代它，`take_hook()` 則移除它。鉤子內發生的 panic 會被報告，但不會再次傳給鉤子。鉤子最多只看到訊息的前 1024 個位元組。`PanicInfo` 實作 `Debug`。

```nabi
fn main() {
    std::panic::set_hook(|info| {
        eprintln(f"hook: {info.message}");
    });
    let xs: List<i32> = [];
    println(f"{xs[9]}");
}
```

```text
thread 'main' panicked at main.nabi:6:16:
index 9 out of bounds for length 0
hook: index 9 out of bounds for length 0
```

由 Python、R 或其他主機載入的共用函式庫（`--crate-type=cdylib`）在 panic 時會中止而不是結束（[refs-lang 5.9.12](../../refs-lang/005-modules/509-extern-block.md#5912-ffi-安全的責任)）：鉤子是它先行報告的機會。

執行環境端是 `nabi_panic_set_hook`（[refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-輔助符號附錄非規範性)）。
