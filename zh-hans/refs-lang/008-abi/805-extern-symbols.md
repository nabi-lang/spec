# 8.5. Extern "C" 符号

本节定义 `libnabir.a` 暴露的[符号](../000-meta/007-glossary.md#extern-c-symbol)的签名。编译器生成的代码和标准库会调用这些符号。

在 Nabi 一侧，它们以运行时 ABI `extern "nabi"`（[5.9.2](../005-modules/509-extern-block.md#592-abi-specifiers)）声明，工具链的包把它们声明为私有项（大多在 `core::rt` 中）。在下面的代码块中，`fn` 值是 `NabiFn`（8.5.0a），其他所有值与 C ABI 中相同。`str` 是 `struct { const uint8_t *ptr; size_t len; }`；接收或返回 `str` 的运行时函数，与 `NabiFn` 一样，遵循平台 C ABI 中 16 字节结构体的规则（[8.6.3](806-calling-convention.md#863-struct-arguments--return-small-struct-rule)）：在 ABI 允许时按值放在寄存器中，从不拆分到寄存器和栈上，在 Windows x64 和 wasm32 上按引用传递。程序可以自行声明运行时符号（`extern "nabi"` / `extern "C"`）；其声明必须与工具链自身对该符号的声明相同，不同的声明是错误。下面的列表给出 C ABI 的形态；如果工具链以 GC 类型（`GcBuf<T>`，例如 `nabi_gc_pin`、`nabi_gc_secret_new` 以及 waker 实参）声明了某个参数，而列表中显示为 `Ptr<u8>` / `usize`，则应当写成该 GC 类型。

> [!IMPORTANT]
> 每个符号都以 `nabi_` 前缀开头，以避免与外部库和用户代码发生符号冲突。

## 8.5.0. 线程安全类别

每个符号定义都带有一个**线程安全性**标注。它是编译器生成的代码与运行时之间契约的一部分，说明该符号可以在哪些调用上下文中安全调用。共有六个类别。

| 类别                  | 含义                                                                         |
|-----------------------|------------------------------------------------------------------------------|
| `thread-safe`         | 可以从多个线程并发调用（运行时在内部同步）                                   |
| `caller-thread-only`  | 绑定于调用线程的上下文；其他线程不能代表该调用者执行                         |
| `process-init`        | 在进程启动时、进入多线程之前恰好调用一次                                     |
| `process-shutdown`    | 在进程退出时、多线程结束之后恰好调用一次                                     |
| `per-thread-init`     | 在每个工作线程开始时，由该线程为自己调用一次                                 |
| `per-thread-shutdown` | 在每个工作线程结束时，由该线程为自己调用一次                                 |

`caller-thread-only` 表示函数的效果与调用者的 TLS 槽或栈帧绑定，因此其他线程不能代表该调用者执行。它并不表示该函数不可重入；多个线程可以并发调用它，各自为自己调用。

## 8.5.0a. `NabiFn` - 函数值的 C ABI 形式

把 Nabi 函数值 `{code, env}`（[4.4](../004-functions/404-function-type.md#representation-of-function-values)）传给运行时符号时使用的 C 表示。

```c
typedef struct {
    void *code;   // code pointer that takes env as its first argument
    void *env;    // capture environment (NULL for named functions)
} NabiFn;
```

`NabiFn` 作为 C 结构体**按值传递**。实际的寄存器分配遵循目标 C ABI 中 16 字节结构体的规则。

## 8.5.1. GC 区域

### `nabi_gc_alloc_slow`

[分配](../007-runtime/702-allocation.md)中 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 的慢速路径。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> GcBuf<u8>;
}
```

实参与返回值的含义如下：

- `size`：对象的总大小：16 字节的对象头加上对齐后的载荷。
- `align`：对象的对齐要求，通常为 8，指定 `#[repr(align(N))]` 时更大。
- `type_id`：对象的类型 ID。
- 返回：指向对象（从对象头开始）的 GC 引用（`GcBuf<u8>`，运行时 ABI 的 GC 指针）；该函数填好 type_id 和 gc_flags。

它从不返回 null。内存耗尽是致命的（`nabi_oom`），超过 `isize::MAX` 的请求是致命错误“allocation too large”（退出码 101）。

### `nabi_alloc_raw` / `nabi_realloc_raw`

在 GC 之外分配原始内存（[标准库 1.4 alloc](../../refs-standard-libraries/100-bare/104-alloc.md)）。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_alloc_raw(size: usize, align: usize, zeroed: i32) -> usize;
    fn nabi_realloc_raw(ptr: usize, old_size: usize, align: usize, new_size: usize) -> usize;
}
```

如果 `zeroed` 非零，分配的内存会被清零。当 `size` 为 0、`align` 不是 2 的幂或分配失败时，`nabi_alloc_raw` 返回 null（`0`）；它不会中止。`nabi_realloc_raw` 失败时也返回 null。失败处理由调用者（`bare::alloc`）负责。

### `nabi_gc_wb_slow`

[屏障与根追踪](../007-runtime/704-barriers-and-roots.md)中写屏障的慢速路径。**线程安全性：`caller-thread-only`**（记录到 TLS 中的写屏障日志缓冲区）。

```
extern "nabi" {
    fn nabi_gc_wb_slow(obj: Ptr<u8>, slot: ConstPtr<u8>, len: usize);
}
```

实参 `obj` 是指向包含被修改槽的对象的指针，`slot` 是该槽的地址，`len` 是即将存入的值的字节大小；没有返回值。效果：把 obj 加入记忆集，日志写满时刷新。在并发标记期间，它还会记录该对象持有的引用，对于大对象则记录 `slot..slot + len` 所跨越的卡中的引用（[7.5.3](../007-runtime/705-concurrent-collection.md#753-satb-barrier)）。

### `nabi_gc_register_type` / `nabi_gc_register_types`

注册类型的 [ScanTable](802-scan-table.md)。**线程安全性：`process-init`**。

```
extern "nabi" {
    fn nabi_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
    fn nabi_gc_register_types(base: u32, tables: ConstPtr<ConstPtr<u8>>, n: usize);   // tables[i] under the id base + i
}
```

在进程启动时、进入多线程之前调用，因此不需要同步。`nabi_gc_register_types` 用一次调用把 `n` 个表注册到连续的 id 之下（[8.2.13](802-scan-table.md#8213-registration-time)）。

### `nabi_gc_pin` / `nabi_gc_unpin`

[FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 对象的固定（见[对象布局与内存空间](../007-runtime/701-object-layout.md)和 [`extern` 块（FFI 声明）](../005-modules/509-extern-block.md)）。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_gc_pin(obj: Ptr<u8>) -> Ptr<u8>;       // returns the pinned object (same address)
    fn nabi_gc_unpin(obj: Ptr<u8>);
}
```

传给 `extern "C"` 函数的 GC 指针（`str` 实参）或传给阻塞运行时调用的 GC 指针，即使不固定也不会被移动，因为该实参在调用结束前一直留在根中（[7.4.1](../007-runtime/704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)）。指向一个没有任何实参作为其根的对象内部的指针（列表的缓冲区），在整个调用期间都需要固定，因为回收可能在任何 C 调用期间运行；C 在调用之后仍保留的任何指针也同样需要固定（[标准库 1.6.5](../../refs-standard-libraries/100-bare/106-gc-pin.md#165-when-a-pin-is-needed)）。固定会在对象上设置 `PINNED`，使其不参与疏散并保持存活；取消固定则将其释放。

### `nabi_gc_collect`

显式触发 [GC](../000-meta/007-glossary.md#gc-garbage-collector)，很少使用。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_gc_collect();
}
```

执行一次完整回收，并在其完成后返回。

### `nabi_gc_secret_new` / `nabi_gc_secret_wipe`

`core::crypto::Secret` 的机密存储（[标准库 2.21.5](../../refs-standard-libraries/200-core/221-crypto.md#2215-security-considerations)）。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_gc_secret_new(len: usize) -> GcBuf<u8>;
    fn nabi_gc_secret_wipe(obj: GcBuf<u8>);
}
```

`nabi_gc_secret_new` 分配一个类型 id 为 1（无引用）、设置了 `WIPE` 的 LOS 对象（[8.1.3](801-object-header-abi.md#813-gc_flags-offset-4-u32)）。其清零的载荷容纳 `len` 字节，向上取整到 8 的倍数。它可能先进行一次回收，超过 `isize::MAX` 的请求是致命错误“allocation too large”。`nabi_gc_secret_wipe` 用 volatile 存储覆写整个载荷，随后加一道编译器屏障；它既不分配也不阻塞，因此不是安全点（[8.4](804-safepoint.md)）。回收器在释放该对象之前也以同样方式覆写其载荷。

## 8.5.2. 运行时区域

### `nabi_runtime_init`

初始化运行时。**线程安全性：`process-init`**。

```
extern "nabi" {
    fn nabi_runtime_init();
}
```

在进程启动后、`main()` 之前立即调用。效果：把调用线程设置为主 mutator（其 TLS 块、栈保护区和 GC 状态）。执行器的工作线程池是由 `core::task` 在首次使用时启动的 Nabi 代码，I/O 轮询器在首次注册时惰性创建。编译器生成的 C ABI `main` 在此调用之后立即调用 `nabi.register_types`（[8.2.13](802-scan-table.md#8213-registration-time)）。

运行时还导出线程局部块 `nabi_tls`（[8.3.8](803-tls-slots.md#838-form-of-compiler-emission)）。

### `nabi_runtime_shutdown`

关闭运行时。**线程安全性：`process-shutdown`**。

```
extern "nabi" {
    fn nabi_runtime_shutdown();
}
```

在 `main()` 结束后立即调用。其他每个 Nabi 线程在完成它当前所在的运行时代码或 C 代码之后，于下一个安全点永久停止。然后释放尚未释放的资源。

### `nabi_exit`

立即终止进程。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_exit(code: i32) -> !;
}
```

[`std::process::exit`](../../refs-standard-libraries/300-std/306-process.md) 调用此符号。`finally` 和 `Drop` 不会运行（[4.16.4](../004-functions/416-entry-point.md#4164-exit-codes-and-unhandled-exceptions)）。

### 字符串辅助函数

支撑 `FromStr` 和 `str::bytes` 的内部辅助函数（[标准库 2.15](../../refs-standard-libraries/200-core/215-string.md)）。**线程安全性：`thread-safe`**。`str` 实参使用前言中所述的结构体形式。

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

在 Nabi 一侧，`nabi_str_slice` 以 `#[track_caller]` 声明，且只有前三个参数（[6.2.12](../006-meta-system/602-built-in-attributes.md#6212-track_caller-passing-the-call-site)）；编译器把调用者的位置 `at` 作为第四个 C 实参传入。

### `nabi_runtime_yield`

[安全点协议](804-safepoint.md)中的 `yield` 函数，即安全点轮询的慢速路径。**线程安全性：`caller-thread-only`**（由调用方 mutator 在轮询自己的 yield_flag 后调用）。

```
extern "nabi" {
    fn nabi_runtime_yield();
}
```

### `nabi_enter_native` / `nabi_leave_native`

原生调用包围的慢速路径（[8.4.10](804-safepoint.md#8410-native-calls)）。**线程安全性：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_enter_native();                // entering C: this thread counts as PARKED
    fn nabi_leave_native();                // back from C: wait out a running collection
}
```

编译器在每个 `extern "C"` 调用（`#[leaf]` 调用除外，[标准库 1.7.1](../../refs-standard-libraries/100-bare/107-ffi.md#leaf---short-calls)）周围内联快速路径：以 seq_cst 方式向 mutator 状态存入 `PARKED` / `RUNNING`，然后检查 yield 标志。只有在标志被设置时才调用这两个函数。

### `nabi_native_reenter` / `nabi_native_resume`

每个 `extern "C" fn` 定义的入口和出口，使 C 回调作为 mutator 运行。**线程安全性：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_native_reenter() -> u32;       // if the thread was PARKED (inside C), leave native; returns whether it was
    fn nabi_native_resume(parked: u32);    // re-enter native if reenter left it
}
```

外部代码调用的入口在返回之前计入该线程的外部入口计数。计数非零时发生的 panic 不会退出：在报告和 panic 钩子之后，如果最内层的 `#[on_panic]` 入口就是最近的外部入口（`nabi_catch_push`），则返回到该入口；否则写出 `` panic in a function C called: aborting (`#[on_panic(return = v)]` on the `extern "C" fn` returns `v` to C instead) `` 并中止进程，退出状态为 134（[5.9.12](../005-modules/509-extern-block.md#5912-responsibility-for-ffi-safety)）。

### `nabi_catch_push` / `nabi_catch_pop`

`#[on_panic]` 入口的恢复记录（[5.9.10](../005-modules/509-extern-block.md#on_panic---a-value-for-c-instead-of-an-abort)）。**线程安全性：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_catch_push(rec: Ptr<u8>);      // push rec; compiled code then calls _setjmp(rec + 64)
    fn nabi_catch_pop(rec: Ptr<u8>);       // the entry returns normally: drop its record
}
```

入口把记录保存在自己的帧中：`64 + 768` 字节，16 字节对齐，运行时的部分在前，`jmp_buf` 位于偏移 64 处。它调用 `nabi_catch_push`，然后对 `jmp_buf` 调用 `_setjmp`。恢复时，恢复记录中保存的影子栈和线程状态，并 `longjmp` 到那里，入口返回其 `#[on_panic]` 值；在正常路径上，入口调用 `nabi_catch_pop`。wasm32 和没有 OS 的目标没有这种记录。

### `nabi_errno_save`

`#[errno]` 调用的 errno（[标准库 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errno---errno-and-getlasterror)）。**线程安全性：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_errno_save();                  // errno, and GetLastError on Windows, into this thread's slots
}
```

编译后的代码在 C 调用返回后立即调用它，早于离开原生状态或取消固定实参：这两者都可能运行改变 errno 的代码。它既不分配也不阻塞。

### `nabi_nosp_enter` / `nabi_nosp_leave`

围绕对 `#[no_safepoint]` 运行时函数调用的检查包围（[8.4.10](804-safepoint.md#8410-native-calls)）。只有调试版 nabic 生成它。**线程安全性：`caller-thread-only`**。

```
extern "nabi" {
    fn nabi_nosp_enter();                  // raise this thread's no_safepoint depth (8.3.1)
    fn nabi_nosp_leave();                  // lower it
}
```

深度非零时，如果线程即将因回收而停下，调试版运行时以致命错误结束进程。在发布版运行时中，这两个函数什么也不做。

### `nabi_thread_init` / `nabi_thread_shutdown`

工作线程的生命周期。**线程安全性：`nabi_thread_init` 为 `per-thread-init`，`nabi_thread_shutdown` 为 `per-thread-shutdown`**。

```
extern "nabi" {
    fn nabi_thread_init();
    fn nabi_thread_shutdown();
}
```

在每个工作线程开始和结束时调用，用于初始化或拆除 [TLS 槽](803-tls-slots.md)中的 TLS。

### 内部符号族

以下符号族是参考运行时的内部辅助函数，不属于 ABI 契约：`nabi_fmt_*`、`nabi_str_*` 辅助函数、`nabi_char_*`、`nabi_nat_*`、`nabi_stdio_*`、`nabi_time_*`、`nabi_gc_stats_*`、`nabi_buf_*`、`nabi_cstr_len`。

### 不是安全点的函数

在 Nabi 一侧，它们声明在标有 `#[no_safepoint]` 的 `extern "nabi"` 块中（[5.9.2](../005-modules/509-extern-block.md#592-abi-specifiers)）。

以下运行时函数保证从不分配、从不阻塞，因此对它们的调用不是安全点（[8.4.2](804-safepoint.md#842-emission-sites)、[8.9.12](809-lowering-control.md#8912-shadow-stack-frame)）：`nabi_str_len`、`nabi_str_byte`、`nabi_str_cmp`、`nabi_str_eq`、`nabi_str_eq_at`、`nabi_str_find_near`、`nabi_str_rfind_near`、`nabi_str_char_count`、`nabi_str_bytes_buf`、`nabi_buf_copy`、`nabi_char_*` 属性与大小写函数、`nabi_fmt_len`、`nabi_f64_*`、`nabi_f32_*`。

对其中任一函数的调用会推迟其他所有线程的回收直到它返回，因此它的工作保持简短。对于较长的输入（64 KiB 或以上），标准库改为调用声明在 `#[no_safepoint]` 块之外的 parked 变体（[8.4.10](804-safepoint.md#8410-native-calls)）：`nabi_str_find_parked` / `nabi_str_rfind_parked`（搜索在前 64 KiB 之后的剩余部分，前 64 KiB 由 `nabi_str_find_near` / `nabi_str_rfind_near` 就地查看）、`nabi_str_char_count_parked`、`nabi_str_grapheme_count_parked`、`nabi_buf_utf8_error_parked`、`nabi_buf_copy_parked`、`nabi_buf_put_str_parked`。

## 8.5.3. 协程区域

### `nabi_coro_frame_alloc`

分配一个[协程](../007-runtime/708-coroutine.md)帧大小的 GC 对象。**线程安全性：`thread-safe`**。编译器不生成它：协程帧是普通的堆分配（[8.10](810-lowering-coroutine-generic.md)）。运行时为 C 和底层代码提供该函数。

```
extern "nabi" {
    fn nabi_coro_frame_alloc(size: usize, type_id: u32) -> GcBuf<u8>;
}
```

它与 `nabi_gc_alloc_slow` 几乎相同，但允许针对帧的特殊处理。

### `nabi_coro_park` / `nabi_coro_unpark`

编译器不生成这两个符号（`await` 使用 `core::task` 的驻留注册表）。运行时为 C 和底层代码提供它们。`frame`（任意 GC 对象）的地址是驻留键，`nabi_coro_unpark(frame)` 唤醒驻留在该键上的所有 waker。在任务执行器之外调用它们会中止。

**线程安全性：`nabi_coro_park` 为 `caller-thread-only`**（驻留调用它的协程自身），**`nabi_coro_unpark` 为 `thread-safe`**（可以从其他线程调用）。

```
extern "nabi" {
    fn nabi_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);
    fn nabi_coro_unpark(frame: Ptr<u8>);
}
```

`nabi_coro_park` 挂起协程并注册 waker。`nabi_coro_unpark` 把协程推入任务队列以恢复它。

这里的 `waker` 在 Nabi 一侧是存放运行时 `Waker`（[2.20.2](../../refs-standard-libraries/200-core/220-future.md)）的单元素 `GcBuf<Waker>`，与 8.5.5 相同。跨越 C ABI 时，它是指向该缓冲区的指针，在列表中显示为 `Ptr<u8>`，包括 [7.8.8](../007-runtime/708-coroutine.md)（`nabi_io_register` 和 `nabi_timer_register` 同理）。`Waker` 的布局由协程实现决定。

## 8.5.4. 调度器区域

任务运行时（spawn、join、取消、执行器）是 `core::task` 中的 Nabi 代码（[标准库 2.18](../../refs-standard-libraries/200-core/218-task.md)）。运行时为其提供下面的执行器槽、线程和驻留符号；其他调度器辅助函数（`nabi_sched_*`、`nabi_lane_*`、`nabi_sync_*`、`nabi_deque_*`）由实现定义（8.5.12）。

### `nabi_task_worker_*` / `nabi_time_now_ns`

在 TLS 槽 `nabi_tls_task_worker`（[8.3.1](803-tls-slots.md#831-tls-slot-structure)）中存取当前线程的执行器工作线程（[标准库 2.18](../../refs-standard-libraries/200-core/218-task.md)）的函数，外加一个时钟。**线程安全性：`caller-thread-only`**（`nabi_time_now_ns` 为 `thread-safe`）。

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

创建一个 1:1 的 OS 线程。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_thread_spawn(f: NabiFn, stack_size: usize, id: u64) -> usize;
    fn nabi_thread_new_id() -> u64;
    fn nabi_thread_id() -> u64;                        // worker id of the current thread
    fn nabi_thread_sleep_ns(ns: i64);
    fn nabi_thread_yield_now();
}
```

`nabi_thread_spawn` 在一个 OS 线程上运行 Nabi 闭包 `f`（一个按值传递的 `NabiFn`）（`stack_size` 为 0 表示默认值）。`id` 是从 `nabi_thread_new_id` 获得的值，成为新线程的 `nabi_tls_worker_id`。返回值是一个 join 句柄，创建失败时为 0。在线程结束之前，`env` 一直是 GC 根。

### `nabi_task_hooks_install` / `nabi_task_key`

`core::task` 的执行器向运行时安装钩子。运行时的 I/O 驱动、定时器以及 `nabi_coro_park` / `nabi_coro_unpark` 使用这些钩子来唤醒和挂起任务。**线程安全性：`thread-safe`**。

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

`nabi_task_key` 把地址用作键，因此会在对象上设置 `NOMOVE`（[8.1.3](801-object-header-abi.md#813-gc_flags-offset-4-u32)）。此后该对象永远不会被移动。只在一次调用内使用的同一性比较使用 `buf_same`（[标准库 2.9](../../refs-standard-libraries/200-core/209-collections.md)），它不固定任何对象。

### `nabi_sync_wait` / `nabi_sync_wake`

同步原语（`Mutex`、`Condvar`、`Once`）的等待与唤醒。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_sync_wait(buf: Ptr<u8>, index: usize, expected: i32, timeout_ns: i64);   // wait while the GcBuf<i32> element equals expected. timeout_ns < 0 means forever
    fn nabi_sync_wake(buf: Ptr<u8>, index: usize);                                    // wake all threads waiting on that element
}
```

`nabi_sync_wait` 可能在没有唤醒理由时返回，因此调用者需重新检查其条件。等待期间，就 GC 而言该线程被视为已停止。

### `nabi_thread_join`

**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_thread_join(handle: usize) -> i32;          // 0 on success
}
```

### `nabi_thread_detach` / `nabi_cpu_count`

**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_thread_detach(handle: usize) -> i32;   // OS resources are released when it ends; can no longer be joined. 0 on success
    fn nabi_cpu_count() -> usize;                  // number of online CPUs (at least 1); size of the par pool
}
```

### `nabi_static_init_begin` / `nabi_static_init_end`

支撑初始化器不是 `const` 的 `static` 的惰性初始化（[5.3](../005-modules/503-static.md)）。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_static_init_begin(state: Ptr<u8>, at: ConstPtr<u8>, at_len: usize) -> i32;   // 1: caller runs the initializer, 0: already ready
    fn nabi_static_init_end(state: Ptr<u8>);            // publish initialization completion
}
```

`state` 是该 `static` 的状态字节（0 为新建，1 为进行中，2 为就绪）。如果另一个线程正在初始化，`nabi_static_init_begin` 会等到它完成，期间就 GC 而言该线程被视为已停止。`at` / `at_len` 是读取该 static 的代码的 `path:line:col`；读取自身 static 的初始化器会在该处 panic（[4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors)）。

### `nabi_gc_add_root`

```
extern "nabi" {
    fn nabi_gc_add_root(word: ConstPtr<usize>);   // register a static's reference word as a GC root until process exit
}
```

编译器生成的启动代码为每个持有 GC 引用的 `static` 调用此函数。

### `nabi_os_*` - OS 辅助函数（内部，可能变更）

支撑标准库 `std`（fs、io、env、process）的内部符号。它们不受 ABI 稳定性保证，可能变更，因此用户代码不直接调用它们。

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

在 Windows 上，运行时还导出 `nabi_os_raw_handle`、`nabi_os_raw_socket`、`nabi_os_handle_fd` 和 `nabi_os_socket_fd`，用于在 fd 与 `HANDLE` / `SOCKET` 值之间转换。

### `nabi_stdio_*` / `nabi_stdin_read` - 标准流（内部，可能变更）

支撑 `print` / `eprint` 和标准输入的内部符号。它们不受 ABI 稳定性保证，可能变更。字节区域是 `GcBuf<u8>` 的 `(buf, off, len)`；失败以 `-errno` 返回。

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

### `nabi_deque_*` - 工作窃取双端队列

任务调度器的无锁 Chase-Lev 双端队列（[7.6.2](../007-runtime/706-scheduler.md#762-work-stealing-structure)）。双端队列及其环形缓冲都是由 GC 追踪的 GC 对象，因此只存在于双端队列中的元素也会保持存活。**线程安全性**：`push` / `pop` 只能由所属线程调用；`steal` 和 `len` 可由任意线程调用。

```
extern "nabi" {
    fn nabi_deque_new() -> GcBuf<u8>;                                 // empty deque (GC object)
    unsafe fn nabi_deque_push(d: Ptr<u8>, item: usize);               // owner: push at the end (GC reference)
    unsafe fn nabi_deque_pop(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;    // owner: newest element into out[index], 1 if any
    unsafe fn nabi_deque_steal(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;  // oldest element into out[index], 1 if any
    unsafe fn nabi_deque_len(d: Ptr<u8>) -> usize;                    // element count (approximate under contention)
}
```

`out` 是存放 GC 引用的 `GcBuf`；结果写在 `index` 处。这些函数不包含安全点。

## 8.5.5. I/O / 定时器区域

### `nabi_io_register` / `nabi_timer_register`

让 C 或底层代码注册一个在 fd 就绪时或在某个时间点被唤醒的任务。它们在内部使用 reactor（[7.7](../007-runtime/707-io-and-timer.md)）和执行器的定时器。在任务执行器之外调用它们会中止。

**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_io_register(fd: i32, interest: u32, waker: Ptr<u8>);
    fn nabi_timer_register(deadline: u64, waker: Ptr<u8>);
}
```

- `fd`：文件描述符。
- `interest`：`1` = 读，`2` = 写（二者可以按位或）。
- `waker`：单元素的 `GcBuf<Waker>`。注册是边沿触发的，**只唤醒一次**（可能出现虚假唤醒）。如果 fd 已经就绪或无法监视，则立即唤醒。
- `deadline`：`nabi_time_now_ns` 时钟上的绝对时间（ns）。到达该时间时，waker 被唤醒一次。

### `nabi_io_*` - reactor（内部，可能变更）

构成 [I/O 驱动](../007-runtime/707-io-and-timer.md)中 reactor 的内部符号。它们不受 ABI 稳定性保证，可能变更。**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_io_poller() -> i32;                    // the process's kqueue / epoll fd (created on first call)
    fn nabi_io_watch(fd: i32, interest: u32) -> i32;   // register fd edge-triggered (re-registering is harmless); interest 1 read, 2 write; 0 or -errno
    fn nabi_io_poll(out: Ptr<u8>, max: usize) -> i32;   // blocks until something changes (parked for the GC), event count or -errno
    fn nabi_io_armed() -> i32;                     // WASI only: 1 if any watch is armed
    fn nabi_io_wait(out: Ptr<u8>, max: usize, timeout_ns: i64) -> i32;   // WASI only: nabi_io_poll with a timeout (< 0: none); 0 = timed out
}
```

`nabi_io_poll` 把至多 `max` 个事件以 `fd * 2 + direction`（读为 0，写为 1）的形式写入 `GcBuf<i64>` 类型的 `out`。

### `nabi_net_*` - 套接字辅助函数（内部，可能变更）

支撑标准库 `std::net`（[标准库 3.2](../../refs-standard-libraries/300-std/302-net.md)）的内部符号。它们不受 ABI 稳定性保证，可能变更，因此用户代码不直接调用它们。套接字总是以非阻塞方式创建，失败以 `-errno` 返回。

```
nabi_net_socket        nabi_net_bind          nabi_net_listen        nabi_net_connect
nabi_net_sock_error    nabi_net_accept        nabi_net_addr          nabi_net_listeners
nabi_net_send          nabi_net_recv          nabi_net_send_to       nabi_net_recv_from
nabi_net_shutdown      nabi_net_setopt        nabi_net_multicast_v4  nabi_net_wait
nabi_net_resolve       nabi_net_unix          nabi_net_unix_pair     nabi_net_unix_path
nabi_net_send_fd       nabi_net_recv_fd       nabi_net_sendfile      nabi_net_send_zerocopy
nabi_net_splice        nabi_net_getopt
```

`nabi_net_socket` 接收 `net::Socket` 的域、类型和协议；`nabi_net_setopt` / `nabi_net_getopt` 按库定义的代码设置和读取套接字选项。`nabi_net_wait` 是阻塞调用用来等待就绪的 `poll(2)`（对 GC 而言是原生区段）；`nabi_net_resolve` 是 `getaddrinfo`，其失败代码 `e` 以 `-(100000 + |e|)` 返回，`nabi_os_strerror` 和 `nabi_os_errno_kind` 像接受 errno 一样接受它。`nabi_net_unix*`、`nabi_net_send_fd` / `nabi_net_recv_fd`、`nabi_net_sendfile`、`nabi_net_send_zerocopy` 和 `nabi_net_splice` 在 Windows 上不导出。

## 8.5.6. 错误传播（无符号）

[错误控制流](../003-expressions/314-error-control-flow.md)中的 `throw` / `try` / `catch` 没有运行时符号。会抛出错误的函数把错误作为其返回值的一部分返回（[8.6.5](806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)），调用处检查该值并分支（[8.9.6](809-lowering-control.md#896-try--catch--finally-value-based-branching)）。因此不存在用于抛出错误或获取当前错误的运行时调用，也没有用于错误处理的 TLS 槽或 jmp_buf。

## 8.5.7. 诊断区域

在 `wasm32-wasip2` 上，下列退出码（`101` 等）以普通失败的形式到达宿主（[4.16.4](../004-functions/416-entry-point.md#4164-exit-codes-and-unhandled-exceptions)）。

### `nabi_panic`

**线程安全性：`thread-safe`**（可以从任何线程调用；进程立即退出）。

```
extern "nabi" {
    fn nabi_panic(msg: ConstPtr<u8>, msg_len: usize) -> !;
}
```

不可恢复的 panic；是致命的。该函数不返回：它把消息打印到 stderr，并以退出码 `101` 终止进程。`catch` 和 `finally` 不会运行。

### `nabi_panic_str`

**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_panic_str(msg: str) -> !;          // msg is (ptr, len)
}
```

与 `nabi_panic` 相同，但以 `str` 接收消息。它不打印位置，只用于运行时内部的 panic。

### `nabi_panic_at`

**线程安全性：`thread-safe`**。

```
extern "nabi" {
    #[track_caller]
    fn nabi_panic_at(msg: str) -> !;
}
```

该声明是 `#[track_caller]`（[6.2.12](../006-meta-system/602-built-in-attributes.md#6212-track_caller-passing-the-call-site)），因此该符号以一个隐藏的尾随 `str` 实参（`"path:line:col"`）接收调用者的位置：`nabi_panic_at(msg, at)`。

按 [4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors) 的格式打印，并以退出码 `101` 退出。core 的 `panic` 以及编译器生成的 panic（边界检查、溢出、除以零等）调用此符号。

### `nabi_uncaught_error` / `nabi_uncaught_error_at`

```
extern "nabi" {
    fn nabi_uncaught_error(what: str) -> !;              // uncaught exception: print what to stderr, exit code 1
    fn nabi_uncaught_error_at(what: str, at: str) -> !;  // followed by a "  --> <at>" line
}
```

处理从入口点传播出来的错误（[4.16.4](../004-functions/416-entry-point.md#4164-exit-codes-and-unhandled-exceptions)）。声明位于 core 中。

### `nabi_oom`

**线程安全性：`thread-safe`**。

```
extern "nabi" {
    fn nabi_oom() -> !;
}
```

把 `fatal error: out of memory` 打印到 stderr，并以退出码 `101` 退出。运行时内部的分配失败和 core 的 `oom()` 都经由此符号。运行时无法按名称调用 Nabi 函数，因此分配失败时它直接使用此符号，而不是调用 `oom_fn` 槽中的函数。

### 栈溢出

线程的栈溢出时，运行时（在可能的情况下）刷新 stdout，把以下内容打印到 stderr，然后中止（SIGABRT）。与 panic 不同，这里没有栈展开，也不执行 `finally`。

```
thread '<name>' has overflowed its stack
fatal runtime error: stack overflow, aborting
```

`<name>` 是用 `thread::Builder::name` 指定的线程名（同时也被设为 OS 线程名，[7. 运行时](../007-runtime/README.md#runtime-error-stack-overflow)）。这覆盖主线程（`nabi_runtime_init`）、`nabi_thread_spawn` 线程（包括执行器工作线程以及深层 `await` 链）以及调用了 `nabi_thread_init` 的 C 线程。运行时把每个线程栈下方的保护范围记录在 TLS 槽 `nabi_tls_stack_guard`（[8.3.1](803-tls-slots.md#831-tls-slot-structure)）中，并安装一个备用信号栈（`sigaltstack`）。只有该范围内的 SIGSEGV / SIGBUS 才被报告为栈溢出；其他故障交给运行时安装其处理器之前设置的动作（默认动作，除非把 Nabi 作为共享库加载的宿主进程设置了别的动作）。在 wasm 上，模块无法捕获陷阱；由宿主报告（随附的 JS 宿主打印上述消息）。

## 8.5.8. 符号命名约定

各符号前缀对应的区域汇总如下。

| 前缀            | 区域           |
|-----------------|----------------|
| `nabi_gc_*`      | GC             |
| `nabi_runtime_*` | 运行时总体     |
| `nabi_thread_*`  | OS 线程        |
| `nabi_coro_*`    | 协程           |
| `nabi_task_*`    | 任务           |
| `nabi_io_*`      | I/O            |
| `nabi_timer_*`   | 定时器         |
| `nabi_panic`     | 诊断           |

这一前缀约定避免与外部库和用户代码发生符号冲突。8.5.1 至 8.5.7 的各表列出规范性的运行时符号；运行时可以导出更多由实现定义的 `nabi_*` 符号（例如 `nabi_fmt_*`、`nabi_os_*`、`nabi_stdio_*`、`nabi_static_init_*`）。

## 8.5.9. 函数签名的兼容性

每个函数签名都属于 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)。变更策略如下：

- 1.0 之前：每个 0.x.0 版本都可能包含破坏性变更；0.x.y 版本只做兼容变更。
- 1.0 起的次版本：只允许兼容变更；增加参数通过函数包装处理。
- 1.0 起的主版本：可能包含破坏性变更，且 ABI 版本哈希会改变。

详见[概述](README.md)中的稳定性策略。

## 8.5.10. 调用约定

以上所有函数都使用目标平台的 C ABI 调用约定，例如 System V x86_64、AAPCS aarch64、MS x64 Windows。

## 8.5.11. 示例：从 TLAB 快速路径发起的慢速调用

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

该示例展示了组合快速路径与慢速路径的生成代码结构（伪代码；快速路径中对对象头的存储见 [8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path)）。

## 8.5.12. 辅助符号（附录，资料性）

> [!IMPORTANT]
> 本节是**资料性**的。下面的符号是参考运行时（`nabir`）导出的辅助符号。它们不属于主 ABI 表（8.5.1 ~ 8.5.11），没有任何稳定性承诺；运行时可以导出更多由实现定义的 `nabi_*` 符号（8.5.8）。

签名使用与正文相同的记法（`Ptr<T>` = 可变裸指针，`ConstPtr<T>` = 只读裸指针）。字符串实参是以 NUL 结尾的 C 字符串。

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

适用以下规则。

- 用户代码不得依赖这些符号。它们的签名可能在不另行通知的情况下变更或被移除。标准库与运行时一同发布，并调用其中的一部分（`nabi_free`、`nabi_gc_weak_new`、`nabi_gc_weak_get`、`nabi_gc_pinned`、`nabi_panic_set_hook`，以及 `nabi_handle_*`、`nabi_errno_*`、`nabi_last_error_*`、`nabi_session_*` 和 `nabi_lock_*` 函数）。
- 它们共用 `nabi_` 前缀，但不包含在 8.5.8 的区域表中。为避免名称冲突，规范中的规范性符号不复用这些名称。
- `nabi_panic` 不属于本附录。[8.5.7](#857-diagnostic-area) 中的签名 `nabi_panic(msg: ConstPtr<u8>, msg_len: usize)` 具有权威性，导出不同签名的实现不符合规范。
