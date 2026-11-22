# 8.5. Extern "C" 符號

定義 `libnabir.a` 所公開之[符號](../000-meta/007-glossary.md#extern-c-符號)的簽章。編譯器產生的程式碼與標準函式庫會呼叫這些符號。

在 Nabi 一方，它們以執行環境 ABI `extern "nabi"` 宣告（[5.9.2](../005-modules/509-extern-block.md#592-abi-指定符)），工具鏈的套件把它們宣告為私有項目（大多數在 `core::rt`）。在下列區塊中，`fn` 值是 `NabiFn`（8.5.0a），其他所有值都與 C ABI 相同。`str` 是 `struct { const uint8_t *ptr; size_t len; }`；接受或回傳 `str` 的執行環境函式，與 `NabiFn` 一樣遵循平台 C ABI 對 16 位元組結構的規則（[8.6.3](806-calling-convention.md#863-struct-引數--回傳小-struct-規則)）：在 ABI 允許之處以值放在暫存器中傳遞，絕不拆分到暫存器與堆疊之間，在 Windows x64 與 wasm32 上則以參照傳遞。程式可以自己宣告執行環境符號（`extern "nabi"` / `extern "C"`）；其宣告必須與工具鏈自己對該符號的宣告相同，不同的宣告是錯誤。下列清單給出 C ABI 的形狀；當工具鏈以 GC 型別宣告某個參數（`GcBuf<T>`，例如 `nabi_gc_pin`、`nabi_gc_secret_new` 與 waker 引數），而清單顯示為 `Ptr<u8>` / `usize` 時，應寫的是該 GC 型別。

> [!IMPORTANT]
> 每個符號都以 `nabi_` 前綴開頭，以避免與外部函式庫及使用者程式碼的符號衝突。

## 8.5.0. 執行緒安全類別

每個符號定義都帶有 **Thread-safety** 標註。它是編譯器發出的程式碼與執行環境之間契約的一部分，說明該符號在哪些呼叫情境中可以安全地呼叫。共有六個類別。

| 類別                  | 意義                                                                         |
|-----------------------|------------------------------------------------------------------------------|
| `thread-safe`         | 可以從多個執行緒同時安全呼叫（執行環境在內部同步）                           |
| `caller-thread-only`  | 繫結到呼叫執行緒的情境；其他執行緒不能代替該呼叫者行事                       |
| `process-init`        | 在行程啟動時、進入多執行緒之前恰好一次                                       |
| `process-shutdown`    | 在行程結束時、多執行緒結束之後恰好一次                                       |
| `per-thread-init`     | 在每個工作執行緒開始時一次，由該執行緒為自己呼叫                             |
| `per-thread-shutdown` | 在每個工作執行緒結束時一次，由該執行緒為自己呼叫                             |

`caller-thread-only` 表示該函式的效果繫結到呼叫者的 TLS 槽位或堆疊框架，因此其他執行緒不能代替該呼叫者行事。它不表示該函式不可重入；多個執行緒可以同時呼叫它，各自為自己。

## 8.5.0a. `NabiFn`：函式值的 C ABI 形式

把 Nabi 函式值 `{code, env}`（[4.4](../004-functions/404-function-type.md#函式值的表示方式)）傳給執行環境符號時所使用的 C 表示方式。

```c
typedef struct {
    void *code;   // code pointer that takes env as its first argument
    void *env;    // capture environment (NULL for named functions)
} NabiFn;
```

`NabiFn` 以 C 結構的形式**以值傳遞**。實際的暫存器指派遵循目標 C ABI 對 16 位元組結構的規則。

## 8.5.1. GC 區域

### `nabi_gc_alloc_slow`

[記憶體配置](../007-runtime/702-allocation.md)中 [TLAB](../000-meta/007-glossary.md#tlab執行緒區域配置緩衝區) 的慢速路徑。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> GcBuf<u8>;
}
```

引數與回傳值的意義：

- `size`：物件的總大小：16 位元組的標頭加上對齊後的酬載。
- `align`：物件的對齊要求，通常是 8，指定了 `#[repr(align(N))]` 時更大。
- `type_id`：物件的型別 ID。
- 回傳：指向該物件、從標頭開始的 GC 參照（`GcBuf<u8>`，即執行環境 ABI 的 GC 指標）；函式會填入 type_id 與 gc_flags。

它絕不回傳 null。記憶體不足是致命錯誤（`nabi_oom`），超過 `isize::MAX` 的請求是致命錯誤「allocation too large」（結束碼 101）。

### `nabi_alloc_raw` / `nabi_realloc_raw`

在 GC 之外配置原始記憶體（[stdlib 1.4 alloc](../../refs-standard-libraries/100-bare/104-alloc.md)）。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_alloc_raw(size: usize, align: usize, zeroed: i32) -> usize;
    fn nabi_realloc_raw(ptr: usize, old_size: usize, align: usize, new_size: usize) -> usize;
}
```

若 `zeroed` 不為零，配置的記憶體會以零填滿。當 `size` 為 0、`align` 不是 2 的冪，或配置失敗時，`nabi_alloc_raw` 回傳 null（`0`）；它不會中止。`nabi_realloc_raw` 失敗時也回傳 null。失敗的處理是呼叫者（`bare::alloc`）的責任。

### `nabi_gc_wb_slow`

[屏障與根追蹤](../007-runtime/704-barriers-and-roots.md)中寫入屏障的慢速路徑。**Thread-safety：`caller-thread-only`**（記錄到 TLS 中的 wb 記錄緩衝區）。

```
extern "nabi" {
    fn nabi_gc_wb_slow(obj: Ptr<u8>, slot: ConstPtr<u8>, len: usize);
}
```

引數 `obj` 是指向包含被修改槽位之物件的指標，`slot` 是該槽位的位址，`len` 是即將存入那裡之值的位元組大小；沒有回傳值。效果：把 obj 加入記憶集，記錄滿了時清出。在並行標記期間，它還會記錄該物件所持有的參照，對大型物件而言則是 `slot..slot + len` 所跨越之卡片中的參照（[7.5.3](../007-runtime/705-concurrent-collection.md#753-satb-屏障)）。

### `nabi_gc_register_type` / `nabi_gc_register_types`

註冊型別的 [ScanTable](802-scan-table.md)。**Thread-safety：`process-init`**。

```
extern "nabi" {
    fn nabi_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
    fn nabi_gc_register_types(base: u32, tables: ConstPtr<ConstPtr<u8>>, n: usize);   // tables[i] under the id base + i
}
```

在行程啟動時、進入多執行緒之前呼叫，因此不需要同步。`nabi_gc_register_types` 以一次呼叫把 `n` 個表註冊在連續的 id 之下（[8.2.13](802-scan-table.md#8213-註冊時機)）。

### `nabi_gc_pin` / `nabi_gc_unpin`

釘選 [FFI](../000-meta/007-glossary.md#ffi外部函式介面) 物件（見[物件配置與記憶體空間](../007-runtime/701-object-layout.md)與 [`extern` 區塊（FFI 宣告）](../005-modules/509-extern-block.md)）。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_gc_pin(obj: Ptr<u8>) -> Ptr<u8>;       // returns the pinned object (same address)
    fn nabi_gc_unpin(obj: Ptr<u8>);
}
```

傳給 `extern "C"` 函式（`str` 引數）或阻塞之執行環境呼叫的 GC 指標，即使不釘選也不會被移動，因為該引數在呼叫結束之前都留在根中（[7.4.1](../007-runtime/704-barriers-and-roots.md#被根指向的物件不會被移動)）。指向沒有任何引數為其建立根之物件內部的指標（串列的緩衝區）在整個呼叫期間都需要釘選，因為在任何 C 呼叫期間都可能進行回收；C 在呼叫之後保留的任何指標也是如此（[stdlib 1.6.5](../../refs-standard-libraries/100-bare/106-gc-pin.md#165-何時需要固定)）。釘選會在物件上設定 `PINNED`，使它不被疏散並保持存活；解除釘選則釋放它。

### `nabi_gc_collect`

明確的 [GC](../000-meta/007-glossary.md#gc垃圾回收器) 觸發，很少使用。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_gc_collect();
}
```

進行完整回收，並在完成後回傳。

### `nabi_gc_secret_new` / `nabi_gc_secret_wipe`

`core::crypto::Secret` 的機密儲存空間（[stdlib 2.21.5](../../refs-standard-libraries/200-core/221-crypto.md#2215-安全性考量)）。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_gc_secret_new(len: usize) -> GcBuf<u8>;
    fn nabi_gc_secret_wipe(obj: GcBuf<u8>);
}
```

`nabi_gc_secret_new` 配置一個型別 id 為 1（沒有參照）、設定了 `WIPE` 的 LOS 物件（[8.1.3](801-object-header-abi.md#813-gc_flags偏移-4u32)）。其清零的酬載容納向上取整為 8 的倍數的 `len` 位元組。它可能會先進行回收，而超過 `isize::MAX` 的請求是致命錯誤「allocation too large」。`nabi_gc_secret_wipe` 以 volatile 存入覆寫整個酬載，其後接一道編譯器柵欄；它既不配置記憶體也不阻塞，因此不是安全點（[8.4](804-safepoint.md)）。回收器在釋放該物件之前也會以相同方式覆寫其酬載。

## 8.5.2. 執行環境區域

### `nabi_runtime_init`

初始化執行環境。**Thread-safety：`process-init`**。

```
extern "nabi" {
    fn nabi_runtime_init();
}
```

在行程啟動之後、`main()` 之前立即呼叫。效果：把呼叫它的執行緒設定為主 mutator（其 TLS 區塊、堆疊保護與 GC 狀態）。執行器的工作者池是由 `core::task` 在第一次使用時啟動的 Nabi 程式碼，而 I/O 輪詢器在第一次註冊時才延遲建立。編譯器產生的 C ABI `main` 在這個呼叫之後立即呼叫 `nabi.register_types`（[8.2.13](802-scan-table.md#8213-註冊時機)）。

執行環境還匯出執行緒區域的區塊 `nabi_tls`（[8.3.8](803-tls-slots.md#838-編譯器發出的形式)）。

### `nabi_runtime_shutdown`

關閉執行環境。**Thread-safety：`process-shutdown`**。

```
extern "nabi" {
    fn nabi_runtime_shutdown();
}
```

在 `main()` 結束之後立即呼叫。其他每個 Nabi 執行緒在完成它所在的任何執行環境或 C 程式碼之後，會在其下一個安全點永久停止。然後釋放尚未釋放的資源。

### `nabi_exit`

立即終止行程。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_exit(code: i32) -> !;
}
```

[`std::process::exit`](../../refs-standard-libraries/300-std/306-process.md) 呼叫這個符號。`finally` 與 `Drop` 不會執行（[4.16.4](../004-functions/416-entry-point.md#4164-結束碼與未處理的例外)）。

### 字串輔助函式

支撐 `FromStr` 與 `str::bytes` 的內部輔助函式（[stdlib 2.15](../../refs-standard-libraries/200-core/215-string.md)）。**Thread-safety：`thread-safe`**。`str` 引數使用前言中的結構形式。

```
extern "nabi" {
    fn nabi_str_eq(a: str, b: str) -> i32;                            // 1 if equal, else 0
    fn nabi_str_byte(s: str, i: usize) -> u8;                         // i-th byte (bounds check is the caller's)
    fn nabi_str_slice(s: str, start: usize, end: usize, at: str) -> str;  // s[start:end]. panics at location at if the range / UTF-8 boundary is wrong
    fn nabi_str_parse_int(s: str, radix: u32, signed: i32, bits: u32, out: GcBuf<i64>) -> i32;
                                        // 0 ok, 1 empty, 2 invalid digit, 3 out of range; value bits in out[0]
    fn nabi_str_parse_float(s: str, out: GcBuf<f64>) -> i32;           // 0 ok, 2 invalid, 3 out of range (finite digits past the type's range); value in out[0]
}
```

在 Nabi 一方，`nabi_str_slice` 宣告為只帶前三個參數的 `#[track_caller]`（[6.2.12](../006-meta-system/602-built-in-attributes.md#6212-track_caller-傳遞呼叫處)）；編譯器把呼叫者的位置 `at` 作為第四個 C 引數傳入。

### `nabi_runtime_yield`

[安全點協定](804-safepoint.md)的 `yield` 函式，即安全點輪詢的慢速路徑。**Thread-safety：`caller-thread-only`**（由呼叫它的 mutator 在輪詢自己的 yield_flag 之後呼叫）。

```
extern "nabi" {
    fn nabi_runtime_yield();
}
```

### `nabi_enter_native` / `nabi_leave_native`

原生包圍的慢速路徑（[8.4.10](804-safepoint.md#8410-原生呼叫)）。**Thread-safety：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_enter_native();                // entering C: this thread counts as PARKED
    fn nabi_leave_native();                // back from C: wait out a running collection
}
```

編譯器在每個 `extern "C"` 呼叫（`#[leaf]` 呼叫除外，[stdlib 1.7.1](../../refs-standard-libraries/100-bare/107-ffi.md#leaf短呼叫)）周圍以行內方式產生快速路徑：對 mutator 狀態進行 `PARKED` / `RUNNING` 的 seq_cst 存入，然後檢查 yield 旗標。只有在旗標已設定時才會呼叫這些函式。

### `nabi_native_reenter` / `nabi_native_resume`

每個 `extern "C" fn` 定義的進入與離開，讓 C 回呼以 mutator 的身分執行。**Thread-safety：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_native_reenter() -> u32;       // if the thread was PARKED (inside C), leave native; returns whether it was
    fn nabi_native_resume(parked: u32);    // re-enter native if reenter left it
}
```

由外部程式碼呼叫的進入點，在回傳之前都計入該執行緒的外部進入次數。在該次數不為零時發生的 panic 不會結束行程：在報告與 panic 掛鉤之後，若最內層的 `#[on_panic]` 進入點就是最近的外部進入點，它就回到那裡（`nabi_catch_push`），否則寫出 `` panic in a function C called: aborting (`#[on_panic(return = v)]` on the `extern "C" fn` returns `v` to C instead) `` 並中止行程，結束狀態為 134（[5.9.12](../005-modules/509-extern-block.md#5912-ffi-安全的責任)）。

### `nabi_catch_push` / `nabi_catch_pop`

`#[on_panic]` 進入點的恢復紀錄（[5.9.10](../005-modules/509-extern-block.md#on_panic給-c-一個值而不是中止)）。**Thread-safety：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_catch_push(rec: Ptr<u8>);      // push rec; compiled code then calls _setjmp(rec + 64)
    fn nabi_catch_pop(rec: Ptr<u8>);       // the entry returns normally: drop its record
}
```

進入點把紀錄保存在其框架中：`64 + 768` 位元組，16 對齊，執行環境的部分在前，`jmp_buf` 在偏移 64。它呼叫 `nabi_catch_push`，然後對 `jmp_buf` 呼叫 `_setjmp`。恢復時會還原紀錄所儲存的影子堆疊與執行緒狀態，並 `longjmp` 到那裡，而進入點回傳其 `#[on_panic]` 值；在正常路徑上，進入點呼叫 `nabi_catch_pop`。wasm32 與沒有作業系統的目標沒有紀錄。

### `nabi_errno_save`

`#[errno]` 呼叫的 errno（[stdlib 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errnoerrno-與-getlasterror)）。**Thread-safety：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_errno_save();                  // errno, and GetLastError on Windows, into this thread's slots
}
```

編譯後的程式碼在 C 呼叫回傳之後、離開原生狀態或解除引數釘選之前立即呼叫它：兩者都可能執行改變 errno 的程式碼。它既不配置記憶體也不阻塞。

### `nabi_nosp_enter` / `nabi_nosp_leave`

對 `#[no_safepoint]` 執行環境函式之呼叫的檢查包圍（[8.4.10](804-safepoint.md#8410-原生呼叫)）。只有偵錯版 nabic 會發出它。**Thread-safety：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_nosp_enter();                  // raise this thread's no_safepoint depth (8.3.1)
    fn nabi_nosp_leave();                  // lower it
}
```

在深度不為零時，若執行緒即將為回收而停下，偵錯版執行環境會以致命錯誤結束行程。在發行版執行環境中，這兩個函式不做任何事。

### `nabi_thread_init` / `nabi_thread_shutdown`

工作執行緒的生命週期。**Thread-safety：`nabi_thread_init` 是 `per-thread-init`，`nabi_thread_shutdown` 是 `per-thread-shutdown`**。

```
extern "nabi" {
    fn nabi_thread_init();
    fn nabi_thread_shutdown();
}
```

在每個工作執行緒的開始與結束時呼叫，以初始化或拆除 [TLS 槽位](803-tls-slots.md)的 TLS。

### 內部符號家族

這些家族是參考執行環境的內部輔助函式，不屬於 ABI 契約：`nabi_fmt_*`、`nabi_str_*` 輔助函式、`nabi_char_*`、`nabi_nat_*`、`nabi_stdio_*`、`nabi_time_*`、`nabi_gc_stats_*`、`nabi_buf_*`、`nabi_cstr_len`。

### 不是安全點的函式

在 Nabi 一方，它們宣告在標示了 `#[no_safepoint]` 的 `extern "nabi"` 區塊中（[5.9.2](../005-modules/509-extern-block.md#592-abi-指定符)）。

下列執行環境函式保證絕不配置記憶體、絕不阻塞，因此對它們的呼叫不是安全點（[8.4.2](804-safepoint.md#842-發出位置)、[8.9.12](809-lowering-control.md#8912-影子堆疊框架)）：`nabi_str_len`、`nabi_str_byte`、`nabi_str_cmp`、`nabi_str_eq`、`nabi_str_eq_at`、`nabi_str_find_near`、`nabi_str_rfind_near`、`nabi_str_char_count`、`nabi_str_bytes_buf`、`nabi_buf_copy`、`nabi_char_*` 屬性與大小寫函式、`nabi_fmt_len`、`nabi_f64_*`、`nabi_f32_*`。

對其中之一的呼叫會阻擋其他每個執行緒的回收直到它回傳，因此其工作保持簡短。對長的輸入（64 KiB 以上），標準函式庫改為呼叫宣告在 `#[no_safepoint]` 區塊之外的停駐版本（[8.4.10](804-safepoint.md#8410-原生呼叫)）：`nabi_str_find_parked` / `nabi_str_rfind_parked`（搜尋在其前 64 KiB 之後的其餘部分，前 64 KiB 由 `nabi_str_find_near` / `nabi_str_rfind_near` 就地檢查）、`nabi_str_char_count_parked`、`nabi_str_grapheme_count_parked`、`nabi_buf_utf8_error_parked`、`nabi_buf_copy_parked`、`nabi_buf_put_str_parked`。

## 8.5.3. 協程區域

### `nabi_coro_frame_alloc`

配置一個[協程](../007-runtime/708-coroutine.md)框架大小的 GC 物件。**Thread-safety：`thread-safe`**。編譯器不會發出它：協程框架是一般的堆積配置（[8.10](810-lowering-coroutine-generic.md)）。執行環境為 C 與低階程式碼提供它。

```
extern "nabi" {
    fn nabi_coro_frame_alloc(size: usize, type_id: u32) -> GcBuf<u8>;
}
```

與 `nabi_gc_alloc_slow` 幾乎相同，但允許框架特有的處理。

### `nabi_coro_park` / `nabi_coro_unpark`

編譯器不會發出這兩個符號（`await` 使用 `core::task` 的停駐登錄）。執行環境為 C 與低階程式碼提供它們。`frame`（任何 GC 物件）的位址是停駐鍵，而 `nabi_coro_unpark(frame)` 會喚醒停駐在該鍵上的每個 waker。在任務執行器之外呼叫它們會中止。

**Thread-safety：`nabi_coro_park` 是 `caller-thread-only`**（停駐呼叫它的協程本身），**`nabi_coro_unpark` 是 `thread-safe`**（可以從另一個執行緒呼叫）。

```
extern "nabi" {
    fn nabi_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);
    fn nabi_coro_unpark(frame: Ptr<u8>);
}
```

`nabi_coro_park` 暫停協程並註冊 waker。`nabi_coro_unpark` 把協程推入任務佇列以恢復它。

這裡的 `waker` 在 Nabi 一方是容納執行環境之 `Waker`（[2.20.2](../../refs-standard-libraries/200-core/220-future.md)）的單一元素 `GcBuf<Waker>`，與 8.5.5 相同。跨越 C ABI 時，它是指向該緩衝區的指標，在清單中顯示為 `Ptr<u8>`，包括 [7.8.8](../007-runtime/708-coroutine.md)（`nabi_io_register` 與 `nabi_timer_register` 也一樣）。`Waker` 的配置由協程實作決定。

## 8.5.4. 排程器區域

任務執行環境（產生、join、取消、執行器）是 `core::task` 中的 Nabi 程式碼（[stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)）。執行環境為它提供下列執行器槽位、執行緒與停駐符號；其他的排程器輔助函式（`nabi_sched_*`、`nabi_lane_*`、`nabi_sync_*`、`nabi_deque_*`）由實作定義（8.5.12）。

### `nabi_task_worker_*` / `nabi_time_now_ns`

在 TLS 槽位 `nabi_tls_task_worker`（[8.3.1](803-tls-slots.md#831-tls-槽位結構)）中儲存與取回目前執行緒之執行器工作者（[stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)）的函式，外加一個時鐘。**Thread-safety：`caller-thread-only`**（`nabi_time_now_ns` 是 `thread-safe`）。

```
extern "nabi" {
    fn nabi_task_worker_present() -> i32;       // 1 if this thread is a worker
    fn nabi_task_worker_get() -> usize;         // worker object address, 0 if none
    fn nabi_task_worker_set(w: usize);          // a live GC object or 0. Pinned while installed
    fn nabi_task_worker_clear();                // = nabi_task_worker_set(0)
    fn nabi_time_now_ns() -> i64;               // monotonic clock (ns)
}
```

### `nabi_thread_spawn`

建立一個 1:1 的作業系統執行緒。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_thread_spawn(f: NabiFn, stack_size: usize, id: u64) -> usize;
    fn nabi_thread_new_id() -> u64;
    fn nabi_thread_id() -> u64;                        // worker id of the current thread
    fn nabi_thread_sleep_ns(ns: i64);
    fn nabi_thread_yield_now();
}
```

`nabi_thread_spawn` 在作業系統執行緒上執行 Nabi 閉包 `f`（一個以值傳遞的 `NabiFn`）（`stack_size` 為 0 表示預設值）。`id` 是從 `nabi_thread_new_id` 取得的值，成為新執行緒的 `nabi_tls_worker_id`。回傳值是 join 控制代碼，建立失敗時為 0。`env` 在執行緒結束之前都是 GC 根。

### `nabi_task_hooks_install` / `nabi_task_key`

`core::task` 的執行器把掛鉤安裝到執行環境中。執行環境的 I/O 驅動程式、計時器以及 `nabi_coro_park` / `nabi_coro_unpark` 使用這些掛鉤來喚醒與暫停任務。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_task_hooks_install(
        io: extern "C" fn(fd: i32, interest: u32, waker: Ptr<u8>),
        timer: extern "C" fn(deadline_ns: u64, waker: Ptr<u8>),
        park: extern "C" fn(key: Ptr<u8>, waker: Ptr<u8>),
        unpark: extern "C" fn(key: Ptr<u8>),
        restart: extern "C" fn(),               // restart executor work lost with a failed REPL cell
    );                                          // the first installation wins
    unsafe fn nabi_task_key(obj: Ptr<u8>) -> usize;   // GC object address as a park key
}
```

`nabi_task_key` 把位址當作鍵使用，因此它在該物件上設定 `NOMOVE`（[8.1.3](801-object-header-abi.md#813-gc_flags偏移-4u32)）。該物件之後絕不會被移動。只在一次呼叫內使用的同一性比較使用 `buf_same`（[stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md)），它不會釘選任何東西。

### `nabi_sync_wait` / `nabi_sync_wake`

同步基本元件（`Mutex`、`Condvar`、`Once`）的等待與喚醒。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_sync_wait(buf: Ptr<u8>, index: usize, expected: i32, timeout_ns: i64);   // wait while the GcBuf<i32> element equals expected. timeout_ns < 0 means forever
    fn nabi_sync_wake(buf: Ptr<u8>, index: usize);                                    // wake all threads waiting on that element
}
```

`nabi_sync_wait` 可能在沒有喚醒理由的情況下回傳，因此呼叫者要重新檢查其條件。等待期間，該執行緒就 GC 而言被視為已停止。

### `nabi_thread_join`

**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_thread_join(handle: usize) -> i32;          // 0 on success
}
```

### `nabi_thread_detach` / `nabi_cpu_count`

**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_thread_detach(handle: usize) -> i32;   // OS resources are released when it ends; can no longer be joined. 0 on success
    fn nabi_cpu_count() -> usize;                  // number of online CPUs (at least 1); size of the par pool
}
```

### `nabi_static_init_begin` / `nabi_static_init_end`

支撐初始化式不是 `const` 之 `static` 的延遲初始化（[5.3](../005-modules/503-static.md)）。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_static_init_begin(state: Ptr<u8>, at: ConstPtr<u8>, at_len: usize) -> i32;   // 1: caller runs the initializer, 0: already ready
    fn nabi_static_init_end(state: Ptr<u8>);            // publish initialization completion
}
```

`state` 是該 `static` 的狀態位元組（0 新的，1 進行中，2 就緒）。若另一個執行緒正在初始化，`nabi_static_init_begin` 會等到它完成，期間該執行緒就 GC 而言被視為已停止。`at` / `at_len` 是讀取該 static 之程式碼的 `path:line:col`；讀取自身 static 的初始化式會在那裡 panic（[4.14.13](../004-functions/414-throws.md#41413-panic-與致命錯誤)）。

### `nabi_gc_add_root`

```
extern "nabi" {
    fn nabi_gc_add_root(word: ConstPtr<usize>);   // register a static's reference word as a GC root until process exit
}
```

編譯器產生的啟動程式碼會為每個容納 GC 參照的 `static` 呼叫它。

### `nabi_os_*`：作業系統輔助函式（內部，可能改變）

支撐標準函式庫 `std`（fs、io、env、process）的內部符號。它們不在 ABI 穩定性的涵蓋範圍內，可能改變，因此使用者程式碼不直接呼叫它們。

```
nabi_os_errno_kind     nabi_os_strerror       nabi_os_open           nabi_os_close
nabi_os_read           nabi_os_write          nabi_os_poll_read      nabi_os_fsync
nabi_os_ftruncate      nabi_os_lseek          nabi_os_dup            nabi_os_fchmod
nabi_os_mmap           nabi_os_munmap         nabi_os_mmap_copy      nabi_os_flock
nabi_os_copy_fd        nabi_os_clone          nabi_os_stat           nabi_os_mkdir
nabi_os_unlink         nabi_os_rename         nabi_os_link           nabi_os_opendir
nabi_os_readdir        nabi_os_closedir       nabi_os_getcwd         nabi_os_chdir
nabi_os_realpath       nabi_os_current_exe    nabi_os_argc           nabi_os_arg
nabi_os_env_at         nabi_os_getenv         nabi_os_setenv         nabi_os_spawn
nabi_os_waitpid        nabi_os_signal_install nabi_os_raise          nabi_os_kill
nabi_os_pidfd_open     nabi_os_pidfd_signal   nabi_os_getpid         nabi_os_getppid
nabi_os_abort          nabi_os_random         nabi_os_trust_roots
nabi_os_openat_raw     nabi_os_create_file2   nabi_os_chmod
```

在 Windows 上，執行環境還匯出 `nabi_os_raw_handle`、`nabi_os_raw_socket`、`nabi_os_handle_fd` 與 `nabi_os_socket_fd`，它們在 fd 與 `HANDLE` / `SOCKET` 值之間轉換。

### `nabi_stdio_*` / `nabi_stdin_read`：標準串流（內部，可能改變）

支撐 `print` / `eprint` 與標準輸入的內部符號。它們不在 ABI 穩定性的涵蓋範圍內，可能改變。位元組區域是 `GcBuf<u8>` 的 `(buf, off, len)`；失敗以 `-errno` 回傳。

```
extern "nabi" {
    fn nabi_stdio_write(fd: i32, s: str, newline: i32) -> i32;                       // s (then "\n" if newline != 0) to fd 1 or 2; 0 or -errno
    unsafe fn nabi_stdio_write_buf(fd: i32, buf: Ptr<u8>, off: usize, len: usize) -> i32;   // len bytes of buf from off to fd 1 or 2
    fn nabi_stdio_flush() -> i32;                                                     // write out the stdout buffer; 0 or -errno
    unsafe fn nabi_stdin_read(buf: Ptr<u8>, off: usize, len: usize, delim: i32) -> isize;
        // up to len bytes of stdin into buf at off, stopping after the first delim byte (delim < 0: no stop);
        // the count (0 at end of input) or -errno
}
```

### `nabi_deque_*`：工作竊取雙端佇列

任務排程器的無鎖 Chase-Lev 雙端佇列（[7.6.2](../007-runtime/706-scheduler.md#762-工作竊取結構)）。雙端佇列及其環形緩衝區是由 GC 追蹤的 GC 物件，因此只存在於雙端佇列中的元素會保持存活。**Thread-safety：**`push` / `pop` 只能由擁有它的執行緒呼叫；`steal` 與 `len` 可以從任何執行緒呼叫。

```
extern "nabi" {
    fn nabi_deque_new() -> GcBuf<u8>;                                 // empty deque (GC object)
    unsafe fn nabi_deque_push(d: Ptr<u8>, item: usize);               // owner: push at the end (GC reference)
    unsafe fn nabi_deque_pop(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;    // owner: newest element into out[index], 1 if any
    unsafe fn nabi_deque_steal(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;  // oldest element into out[index], 1 if any
    unsafe fn nabi_deque_len(d: Ptr<u8>) -> usize;                    // element count (approximate under contention)
}
```

`out` 是容納 GC 參照的 `GcBuf`；結果寫在 `index` 處。這些函式不含任何安全點。

## 8.5.5. I/O / 計時器區域

### `nabi_io_register` / `nabi_timer_register`

讓 C 或低階程式碼註冊一個任務，在 fd 就緒時或在某個時間點被喚醒。它們在內部使用 reactor（[7.7](../007-runtime/707-io-and-timer.md)）與執行器的計時器。在任務執行器之外呼叫它們會中止。

**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_io_register(fd: i32, interest: u32, waker: Ptr<u8>);
    fn nabi_timer_register(deadline: u64, waker: Ptr<u8>);
}
```

- `fd`：檔案描述子。
- `interest`：`1` = 讀，`2` = 寫（兩者可以 OR 起來）。
- `waker`：單一元素的 `GcBuf<Waker>`。註冊是邊緣觸發的，**只喚醒一次**（可能有虛假喚醒）。若 fd 已經就緒或無法被監看，會立即喚醒。
- `deadline`：`nabi_time_now_ns` 時鐘上的絕對時間（ns）。到達該時間時，waker 被喚醒一次。

### `nabi_io_*`：reactor（內部，可能改變）

構成 [I/O 驅動程式](../007-runtime/707-io-and-timer.md)之 reactor 的內部符號。它們不在 ABI 穩定性的涵蓋範圍內，可能改變。**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_io_poller() -> i32;                    // the process's kqueue / epoll fd (created on first call)
    fn nabi_io_watch(fd: i32, interest: u32) -> i32;   // register fd edge-triggered (re-registering is harmless); interest 1 read, 2 write; 0 or -errno
    fn nabi_io_poll(out: Ptr<u8>, max: usize) -> i32;   // blocks until something changes (parked for the GC), event count or -errno
    fn nabi_io_armed() -> i32;                     // WASI only: 1 if any watch is armed
    fn nabi_io_wait(out: Ptr<u8>, max: usize, timeout_ns: i64) -> i32;   // WASI only: nabi_io_poll with a timeout (< 0: none); 0 = timed out
}
```

`nabi_io_poll` 把最多 `max` 個事件以 `fd * 2 + direction`（讀 0，寫 1）的形式寫入 `out`，一個 `GcBuf<i64>`。

### `nabi_net_*`：通訊端輔助函式（內部，可能改變）

支撐標準函式庫 `std::net`（[stdlib 3.2](../../refs-standard-libraries/300-std/302-net.md)）的內部符號。它們不在 ABI 穩定性的涵蓋範圍內，可能改變，因此使用者程式碼不直接呼叫它們。通訊端一律以非阻塞方式建立，失敗以 `-errno` 回傳。

```
nabi_net_socket        nabi_net_bind          nabi_net_listen        nabi_net_connect
nabi_net_sock_error    nabi_net_accept        nabi_net_addr          nabi_net_listeners
nabi_net_send          nabi_net_recv          nabi_net_send_to       nabi_net_recv_from
nabi_net_shutdown      nabi_net_setopt        nabi_net_multicast_v4  nabi_net_wait
nabi_net_resolve       nabi_net_unix          nabi_net_unix_pair     nabi_net_unix_path
nabi_net_send_fd       nabi_net_recv_fd       nabi_net_sendfile      nabi_net_send_zerocopy
nabi_net_splice        nabi_net_getopt
```

`nabi_net_socket` 接受 `net::Socket` 的網域、型別與協定；`nabi_net_setopt` / `nabi_net_getopt` 以函式庫定義的代號設定與讀取通訊端選項。`nabi_net_wait` 是阻塞呼叫用來等待就緒的 `poll(2)`（對 GC 而言是原生區段）；`nabi_net_resolve` 是 `getaddrinfo`；其失敗代號 `e` 以 `-(100000 + |e|)` 回傳，`nabi_os_strerror` 與 `nabi_os_errno_kind` 會像接受 errno 一樣接受它。`nabi_net_unix*`、`nabi_net_send_fd` / `nabi_net_recv_fd`、`nabi_net_sendfile`、`nabi_net_send_zerocopy` 與 `nabi_net_splice` 在 Windows 上不匯出。

## 8.5.6. 錯誤傳播（沒有符號）

[錯誤控制流程](../003-expressions/314-error-control-flow.md)的 `throw` / `try` / `catch` 沒有執行環境符號。會擲回錯誤的函式把錯誤作為其回傳值的一部分回傳（[8.6.5](806-calling-convention.md#865-會擲回錯誤之函式的回傳以值為基礎的錯誤傳播)），呼叫處檢查該值並分支（[8.9.6](809-lowering-control.md#896-try--catch--finally以值為基礎的分支)）。因此沒有用來擲回錯誤或取得目前錯誤的執行環境呼叫，也沒有用於錯誤處理的 TLS 槽位或 jmp_buf。

## 8.5.7. 診斷區域

下列結束碼（`101` 等）在 `wasm32-wasip2` 上以單純的失敗到達主機（[4.16.4](../004-functions/416-entry-point.md#4164-結束碼與未處理的例外)）。

### `nabi_panic`

**Thread-safety：`thread-safe`**（可以從任何執行緒呼叫；行程會立即結束）。

```
extern "nabi" {
    fn nabi_panic(msg: ConstPtr<u8>, msg_len: usize) -> !;
}
```

無法恢復的 panic；致命錯誤。這個函式不會回傳：它把訊息印到 stderr，並以結束碼 `101` 終止行程。`catch` 與 `finally` 不會執行。

### `nabi_panic_str`

**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_panic_str(msg: str) -> !;          // msg is (ptr, len)
}
```

與 `nabi_panic` 相同，但以 `str` 接受訊息。它不印出位置，只用於執行環境內部的 panic。

### `nabi_panic_at`

**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    #[track_caller]
    fn nabi_panic_at(msg: str) -> !;
}
```

其宣告是 `#[track_caller]`（[6.2.12](../006-meta-system/602-built-in-attributes.md#6212-track_caller-傳遞呼叫處)），因此該符號以隱藏的尾端 `str` 引數（`"path:line:col"`）接受呼叫者的位置：`nabi_panic_at(msg, at)`。

以 [4.14.13](../004-functions/414-throws.md#41413-panic-與致命錯誤) 的格式印出，並以結束碼 `101` 結束。core 的 `panic` 以及編譯器產生的 panic（邊界檢查、溢位、除以零等）呼叫這個符號。

### `nabi_uncaught_error` / `nabi_uncaught_error_at`

```
extern "nabi" {
    fn nabi_uncaught_error(what: str) -> !;              // uncaught exception: print what to stderr, exit code 1
    fn nabi_uncaught_error_at(what: str, at: str) -> !;  // followed by a "  --> <at>" line
}
```

處理傳播出進入點的錯誤（[4.16.4](../004-functions/416-entry-point.md#4164-結束碼與未處理的例外)）。其宣告位於 core。

### `nabi_oom`

**Thread-safety：`thread-safe`**。

```
extern "nabi" {
    fn nabi_oom() -> !;
}
```

把 `fatal error: out of memory` 印到 stderr，並以結束碼 `101` 結束。執行環境內部的配置失敗與 core 的 `oom()` 都經過這個符號。執行環境無法以名稱呼叫 Nabi 函式，因此在配置失敗時，它直接使用這個符號，而不是呼叫 `oom_fn` 槽位中的函式。

### 堆疊溢位

當執行緒的堆疊溢位時，執行環境會（盡可能）清空 stdout，把下列內容印到 stderr，然後中止（SIGABRT）。與 panic 不同，沒有展開，也不會執行 `finally`。

```
thread '<name>' has overflowed its stack
fatal runtime error: stack overflow, aborting
```

`<name>` 是以 `thread::Builder::name` 給出的執行緒名稱（也設為作業系統的執行緒名稱，[7. 執行環境](../007-runtime/README.md#執行時錯誤堆疊溢位)）。這涵蓋主執行緒（`nabi_runtime_init`）、`nabi_thread_spawn` 執行緒（包括執行器工作者以及很深的 `await` 鏈），以及呼叫了 `nabi_thread_init` 的 C 執行緒。執行環境把每個執行緒堆疊下方的保護範圍記錄在 TLS 槽位 `nabi_tls_stack_guard`（[8.3.1](803-tls-slots.md#831-tls-槽位結構)）中，並安裝替代的訊號堆疊（`sigaltstack`）。只有在該範圍內的 SIGSEGV / SIGBUS 會被回報為堆疊溢位；其他錯誤交給執行環境安裝其處理函式之前所設定的動作（預設動作，除非以共用函式庫載入 Nabi 的主機行程設定了一個）。在 wasm 上，模組無法捕捉這個 trap；由主機回報它（隨附的 JS 主機會印出上述訊息）。

## 8.5.8. 符號命名慣例

各符號前綴的區域整理如下。

| 前綴            | 區域           |
|-----------------|----------------|
| `nabi_gc_*`      | GC             |
| `nabi_runtime_*` | 執行環境整體   |
| `nabi_thread_*`  | 作業系統執行緒 |
| `nabi_coro_*`    | 協程           |
| `nabi_task_*`    | 任務           |
| `nabi_io_*`      | I/O            |
| `nabi_timer_*`   | 計時器         |
| `nabi_panic`     | 診斷           |

這種前綴慣例避免了與外部函式庫及使用者程式碼的符號衝突。8.5.1 到 8.5.7 的表格列出具規範性的執行環境符號；執行環境可以匯出更多由實作定義的 `nabi_*` 符號（例如 `nabi_fmt_*`、`nabi_os_*`、`nabi_stdio_*`、`nabi_static_init_*`）。

## 8.5.9. 函式簽章的相容性

每個函式簽章都是 [ABI](../000-meta/007-glossary.md#abi應用程式二進位介面) 的一部分。變更方針如下：

- 1.0 之前：每個 0.x.0 版本都可能有破壞性變更；0.x.y 版本只做相容的變更。
- 從 1.0 起的次版本：只允許相容的變更；新增引數以函式包裝處理。
- 從 1.0 起的主版本：可能有破壞性變更，ABI 版本雜湊會改變。

詳見[概觀](README.md)中的穩定性方針。

## 8.5.10. 呼叫慣例

上述所有函式都使用目標平台的 C ABI 呼叫慣例，例如 System V x86_64、AAPCS aarch64、MS x64 Windows。

## 8.5.11. 範例：從 TLAB 快速路徑進行的慢速呼叫

```
t = <nabi_tls address>                ; 8.3.8
cursor = load.ptr [t + 8]             ; tlab_cursor at p
limit = load.ptr [t + 16]             ; tlab_limit at 2p
if 24 <= limit - cursor then bb_fast else bb_slow

bb_slow:                              ; placed out of line
    obj = call nabi_gc_alloc_slow(24, 8, 12345)
    goto bb_cont

bb_fast:
    store.ptr [t + 8], cursor + 24
    obj = cursor
    goto bb_cont

bb_cont:
    ; ... continue
```

這個範例展示結合快速路徑與慢速路徑之發出程式碼的結構（虛擬碼；快速路徑的標頭存入見 [8.9.9](809-lowering-control.md#899-gc-配置快速--慢速路徑)）。

## 8.5.12. 輔助符號（附錄，非規範性）

> [!IMPORTANT]
> 本節是**非規範性**的。下列符號是參考執行環境（`nabir`）匯出的輔助符號。它們不屬於主要的 ABI 表格（8.5.1 ~ 8.5.11），也沒有任何穩定性承諾；執行環境可以匯出更多由實作定義的 `nabi_*` 符號（8.5.8）。

簽章使用與本文相同的記法（`Ptr<T>` = 可變的原始指標，`ConstPtr<T>` = 唯讀的原始指標）。字串引數是以 NUL 結尾的 C 字串。

```
extern "nabi" {
    // memory
    fn nabi_free(ptr: usize);                               // 0 (null) ignored

    // C strings
    fn nabi_cstr_len(s: ConstPtr<u8>) -> usize;             // strlen; null → 0

    // GC
    fn nabi_gc_collect_young();                             // a minor collection; none during a concurrent mark
    fn nabi_gc_collect_young_within(ns: u64) -> u32;        // the same if the estimate fits in ns; 1 if it ran
    fn nabi_gc_set_pause_goal(ns: u64);                     // 0: none (bare::gc::set_pause_goal)
    fn nabi_gc_set_heap_limit(bytes: u64);                  // 0: none (bare::gc::set_heap_limit)
    fn nabi_gc_no_collect(enter: u32);                      // 1: into a no-collect region, 0: out of it
    fn nabi_gc_arena_epoch() -> u64;                        // a generation no bare::gc::Arena used before
    fn nabi_gc_weak_new(target: Ptr<u8>) -> Ptr<u8>;       // a weak cell (type id 3) to target
    fn nabi_gc_weak_get(cell: Ptr<u8>) -> Ptr<u8>;         // the target, or null once freed
    fn nabi_gc_pinned(obj: GcBuf<u8>) -> i32;              // 1 while obj is pinned (a List / StrBuf refuses to grow)

    // FFI (bare::ffi, std::panic); name is a str (ptr, len)
    fn nabi_handle_new(obj: GcBuf<u8>, name: str) -> usize;  // a handle, never 0, that roots and pins obj (a value of type name)
    fn nabi_handle_get(h: usize, name: str) -> GcBuf<u8>;  // its object; panics when released or of another type
    fn nabi_handle_retain(h: usize);
    fn nabi_handle_release(h: usize);                      // the last release unpins the object
    fn nabi_panic_set_hook(hook: extern "C" fn(Ptr<u8>, ConstPtr<u8>, usize, ConstPtr<u8>, usize) | null, ud: Ptr<u8>) -> Ptr<u8>;
        // hook(ud, msg, msg_len, at, at_len) runs on every panic after its report;
        // returns the previous ud, for its owner to release
    fn nabi_errno_get() -> i32;                            // the saved errno
    fn nabi_errno_set(v: i32);                             // the saved errno and the C library's
    fn nabi_last_error_get() -> u32;                       // the saved GetLastError (0 off Windows)
    fn nabi_last_error_set(v: u32);                        // the saved value and, on Windows, SetLastError

    // process exit
    fn nabi_set_exit_hook(hook: extern "C" fn(i32) | null);
        // called with the exit status, on the ending thread, before a panic,
        // uncaught error, exit or (Windows) stack overflow ends the process;
        // the process ends when it returns

    // REPL host (session save / restore)
    fn nabi_session_hold(timeout_ns: i64) -> i32;          // stop the world at a quiet moment: no poll / blocking job
                                                           //   running, no thread stopped inside a lock; 0: not within
                                                           //   timeout_ns (still requested)
    fn nabi_session_unhold();                               // withdraw a pending or granted hold
    fn nabi_session_resume();                               // resume the world after a hold
    fn nabi_session_enter();                                // executor threads' gates around a poll / blocking job
    fn nabi_session_leave();
    fn nabi_session_pass();
    fn nabi_lock_notes() -> i32;                            // 1 in a session host
    fn nabi_lock_note(word: GcBuf<i32>, kind: i32) -> i32;  // kind: 0 Mutex, 1 read, 2 write, 3 running Once; 1 = noted
    fn nabi_lock_drop(word: GcBuf<i32>);
    fn nabi_world_threads(out: Ptr<u64>, cap: usize) -> usize;   // world stopped: OS ids of the other mutators (Windows)
    fn nabi_session_restored(lost: ConstPtr<u64>, n: usize);     // session put back; n == SIZE_MAX: every other thread is gone
    fn nabi_thread_lost(worker_id: u64) -> i32;             // 1 if that thread was found gone after a failed cell
    fn nabi_io_quiesce();                                   // Windows: cancel in-flight I/O before memory is put back
    fn nabi_watch_writes();                                 // Windows: write-watch the heap; call before runtime init
    fn nabi_stdio_line_buffered();                          // flush stdout at each newline, as on a terminal
    fn nabi_set_stdin_hook(hook: extern "C" fn(Ptr<u8>, usize) -> isize | null);
        // read stdin through hook (null: fd 0): the count, 0 at the end of
        // input, or -errno; stdout is flushed before each call
    static nabi_page_epoch: u64;                            // Windows: bumped when watched pages change without a write
                                                           //   or a watched block is released
}
```

適用下列規則。

- 使用者程式碼不得依賴這些符號。它們的簽章可能在沒有通知的情況下改變或被移除。標準函式庫與執行環境一起提供，並呼叫其中一些（`nabi_free`、`nabi_gc_weak_new`、`nabi_gc_weak_get`、`nabi_gc_pinned`、`nabi_panic_set_hook`，以及 `nabi_handle_*`、`nabi_errno_*`、`nabi_last_error_*`、`nabi_session_*` 與 `nabi_lock_*` 函式）。
- 它們共用 `nabi_` 前綴，但不包含在 8.5.8 的區域表中。為了避免名稱衝突，規格中具規範性的符號不會重用這些名稱。
- `nabi_panic` 不屬於這個附錄。[8.5.7](#857-診斷區域) 中的簽章 `nabi_panic(msg: ConstPtr<u8>, msg_len: usize)` 具權威性，匯出不同簽章的實作不符合規範。
