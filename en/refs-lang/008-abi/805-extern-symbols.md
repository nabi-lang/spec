# 8.5. Extern "C" symbols

Defines the signatures of the [symbols](../000-meta/007-glossary.md#extern-c-symbol) exposed by `libnabir.a`. Compiler-generated code and the standard library call these symbols.

On the Nabi side they are declared with the runtime ABI `extern "nabi"` ([5.9.2](../005-modules/509-extern-block.md#592-abi-specifiers)), which the toolchain's packages declare as private items (most in `core::rt`). In the blocks below a `fn` value is `NabiFn` (8.5.0a) and all other values are as in the C ABI. `str` is `struct { const uint8_t *ptr; size_t len; }`; runtime functions taking or returning `str` follow the platform C ABI for a 16-byte struct ([8.6.3](806-calling-convention.md#863-struct-arguments--return-small-struct-rule)), like `NabiFn`: by value in registers where the ABI allows, never split between registers and the stack, and by reference on Windows x64 and wasm32. A program may declare a runtime symbol itself (`extern "nabi"` / `extern "C"`); its declaration must be the same as the toolchain's own declaration of that symbol, and a different one is an error. The listings below give the C ABI shape; where the toolchain declares a parameter with a GC type (`GcBuf<T>`, as for `nabi_gc_pin`, `nabi_gc_secret_new` and the waker arguments) that the listing shows as `Ptr<u8>` / `usize`, the GC type is the one to write.

> [!IMPORTANT]
> Every symbol starts with the `nabi_` prefix, to avoid symbol collisions with external libraries and user code.

## 8.5.0. Thread-safety classes

Each symbol definition carries a **Thread-safety** annotation. It is part of the contract between compiler-emitted code and the runtime and states in which calling contexts the symbol may be called safely. There are six classes.

| Class                 | Meaning                                                                      |
|-----------------------|------------------------------------------------------------------------------|
| `thread-safe`         | Safe to call concurrently from multiple threads (the runtime synchronizes internally) |
| `caller-thread-only`  | Bound to the calling thread's context; another thread cannot act on behalf of that caller |
| `process-init`        | Exactly once at process start, before entering mt                           |
| `process-shutdown`    | Exactly once at process exit, after mt ends                                  |
| `per-thread-init`     | Once at the start of each worker thread, by that thread for itself           |
| `per-thread-shutdown` | Once at the end of each worker thread, by that thread for itself             |

`caller-thread-only` means the function's effects are tied to the caller's TLS slots or stack frame, so another thread cannot act on behalf of that caller. It does not mean the function is non-reentrant; multiple threads may call it concurrently, each for itself.

## 8.5.0a. `NabiFn` - C ABI form of function values

The C representation used when passing a Nabi function value `{code, env}` ([4.4.8](../004-functions/404-function-type.md#representation-of-function-values)) to a runtime symbol.

```c
typedef struct {
    void *code;   // code pointer that takes env as its first argument
    void *env;    // capture environment (NULL for named functions)
} NabiFn;
```

`NabiFn` is **passed by value** as a C struct. The actual register assignment follows the target C ABI's rules for 16-byte structs.

## 8.5.1. GC area

### `nabi_gc_alloc_slow`

The [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) slow path from [Allocation](../007-runtime/702-allocation.md). **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> GcBuf<u8>;
}
```

The arguments and return value mean:

- `size`: the total object size: the 16-byte header plus the payload, aligned.
- `align`: the object's alignment requirement, usually 8, larger when `#[repr(align(N))]` is specified.
- `type_id`: the object's type ID.
- Returns: a GC reference (`GcBuf<u8>`, the runtime ABI's GC pointer) to the object, starting at the header; the function fills in type_id and gc_flags.

It never returns null. Out of memory is fatal (`nabi_oom`), and a request above `isize::MAX` is the fatal error "allocation too large" (exit 101).

### `nabi_alloc_raw` / `nabi_realloc_raw`

Allocates raw memory outside the GC ([stdlib 1.4 alloc](../../refs-standard-libraries/100-bare/104-alloc.md)). **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_alloc_raw(size: usize, align: usize, zeroed: i32) -> usize;
    fn nabi_realloc_raw(ptr: usize, old_size: usize, align: usize, new_size: usize) -> usize;
}
```

If `zeroed` is nonzero, the allocated memory is zero-filled. `nabi_alloc_raw` returns null (`0`) if `size` is 0, if `align` is not a power of two, or if allocation fails; it does not abort. `nabi_realloc_raw` also returns null on failure. Failure handling is the caller's (`bare::alloc`) responsibility.

### `nabi_gc_wb_slow`

The write barrier slow path from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md). **Thread-safety: `caller-thread-only`** (records into the wb log buffer in TLS).

```
extern "nabi" {
    fn nabi_gc_wb_slow(obj: Ptr<u8>, slot: ConstPtr<u8>, len: usize);
}
```

The argument `obj` is a pointer to the object containing the modified slot, `slot` the address of the slot and `len` the size in bytes of the value about to be stored there; there is no return value. Effect: adds obj to the remembered set and flushes when the log is full. During a concurrent mark it also records the references the object holds, or for a large object those of the cards `slot..slot + len` spans ([7.5.3](../007-runtime/705-concurrent-collection.md#753-satb-barrier)).

### `nabi_gc_register_type` / `nabi_gc_register_types`

Registers a type's [ScanTable](802-scan-table.md). **Thread-safety: `process-init`**.

```
extern "nabi" {
    fn nabi_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
    fn nabi_gc_register_types(base: u32, tables: ConstPtr<ConstPtr<u8>>, n: usize);   // tables[i] under the id base + i
}
```

Called at process start, before entering mt, so no synchronization is needed. `nabi_gc_register_types` registers `n` tables under consecutive ids with one call ([8.2.13](802-scan-table.md#8213-registration-time)).

### `nabi_gc_pin` / `nabi_gc_unpin`

Pinning of [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) objects (see [Object layout and memory spaces](../007-runtime/701-object-layout.md) and [`extern` blocks (FFI declarations)](../005-modules/509-extern-block.md)). **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_gc_pin(obj: Ptr<u8>) -> Ptr<u8>;       // returns the pinned object (same address)
    fn nabi_gc_unpin(obj: Ptr<u8>);
}
```

A GC pointer passed to an `extern "C"` function (a `str` argument) or a blocking runtime call is not moved even without pinning, because the argument stays in a root until the call ends ([7.4.1](../007-runtime/704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)). A pointer into an object that no argument roots (a list's buffer) needs a pin for the whole call, since a collection may run during any C call, and so does any pointer C keeps after the call ([stdlib 1.6.5](../../refs-standard-libraries/100-bare/106-gc-pin.md#165-when-a-pin-is-needed)). Pin sets `PINNED` on the object, excluding it from evacuation and keeping it alive; unpin releases it.

### `nabi_gc_collect`

An explicit [GC](../000-meta/007-glossary.md#gc-garbage-collector) trigger, rarely used. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_gc_collect();
}
```

Performs a full collection and returns after it finishes.

### `nabi_gc_secret_new` / `nabi_gc_secret_wipe`

Secret storage for `core::crypto::Secret` ([stdlib 2.21.5](../../refs-standard-libraries/200-core/221-crypto.md#2215-security-considerations)). **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_gc_secret_new(len: usize) -> GcBuf<u8>;
    fn nabi_gc_secret_wipe(obj: GcBuf<u8>);
}
```

`nabi_gc_secret_new` allocates a LOS object of type id 1 (no references) with `WIPE` set ([8.1.3](801-object-header-abi.md#813-gc_flags-offset-4-u32)). Its zeroed payload holds `len` bytes rounded up to a multiple of 8. It may collect first, and a request above `isize::MAX` is the fatal error "allocation too large". `nabi_gc_secret_wipe` overwrites the whole payload with volatile stores followed by a compiler fence; it neither allocates nor blocks, so it is no safepoint ([8.4](804-safepoint.md)). The collector overwrites the payload the same way before it frees the object.

## 8.5.2. Runtime area

### `nabi_runtime_init`

Initializes the runtime. **Thread-safety: `process-init`**.

```
extern "nabi" {
    fn nabi_runtime_init();
}
```

Called right after process start, before `main()`. Effects: sets up the calling thread as the main mutator (its TLS block, stack guard and GC state). The executor's worker pool is Nabi code started by `core::task` on first use, and the I/O poller is created lazily on first registration. The compiler-generated C-ABI `main` calls `nabi.register_types` right after this call ([8.2.13](802-scan-table.md#8213-registration-time)).

The runtime additionally exports the thread-local block `nabi_tls` ([8.3.8](803-tls-slots.md#838-form-of-compiler-emission)).

### `nabi_runtime_shutdown`

Shuts down the runtime. **Thread-safety: `process-shutdown`**.

```
extern "nabi" {
    fn nabi_runtime_shutdown();
}
```

Called right after `main()` ends. Every other Nabi thread stops for good at its next safepoint, after finishing any runtime or C code it is in. Then outstanding resources are released.

### `nabi_exit`

Terminates the process immediately. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_exit(code: i32) -> !;
}
```

[`std::process::exit`](../../refs-standard-libraries/300-std/306-process.md) calls this symbol. `finally` and `Drop` are not run ([4.16.4](../004-functions/416-entry-point.md#4164-exit-codes-and-unhandled-exceptions)).

### String helpers

Internal helpers backing `FromStr` and `str::bytes` ([stdlib 2.15](../../refs-standard-libraries/200-core/215-string.md)). **Thread-safety: `thread-safe`**. `str` arguments use the struct form of the preamble.

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

On the Nabi side `nabi_str_slice` is declared `#[track_caller]` with the first three parameters only ([6.2.12](../006-meta-system/602-built-in-attributes.md#6212-track_caller-passing-the-call-site)); the compiler passes `at`, the caller's location, as the fourth C argument.

### `nabi_runtime_yield`

The `yield` function of the [Safepoint protocol](804-safepoint.md), i.e. the slow path of the safepoint poll. **Thread-safety: `caller-thread-only`** (called by the calling mutator as a result of polling its own yield_flag).

```
extern "nabi" {
    fn nabi_runtime_yield();
}
```

### `nabi_enter_native` / `nabi_leave_native`

Slow paths of the native bracket ([8.4.10](804-safepoint.md#8410-native-calls)). **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_enter_native();                // entering C: this thread counts as PARKED
    fn nabi_leave_native();                // back from C: wait out a running collection
}
```

The compiler inlines the fast path around every `extern "C"` call but a `#[leaf]` one ([stdlib 1.7.1](../../refs-standard-libraries/100-bare/107-ffi.md#leaf---short-calls)): a seq_cst store of `PARKED` / `RUNNING` to the mutator state, then a check of the yield flag. Only when the flag is set does it call these.

### `nabi_native_reenter` / `nabi_native_resume`

The entry and exit of every `extern "C" fn` definition, so that a C callback runs as a mutator. **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_native_reenter() -> u32;       // if the thread was PARKED (inside C), leave native; returns whether it was
    fn nabi_native_resume(parked: u32);    // re-enter native if reenter left it
}
```

An entry that foreign code called counts in the thread's foreign-entry count until it returns. A panic while the count is nonzero does not exit: after the report and the panic hook it returns to the innermost `#[on_panic]` entry when that entry is the nearest foreign one (`nabi_catch_push`), and otherwise writes `` panic in a function C called: aborting (`#[on_panic(return = v)]` on the `extern "C" fn` returns `v` to C instead) `` and aborts the process, exit status 134 ([5.9.12](../005-modules/509-extern-block.md#5912-responsibility-for-ffi-safety)).

### `nabi_catch_push` / `nabi_catch_pop`

The recovery record of an `#[on_panic]` entry ([5.9.10](../005-modules/509-extern-block.md#on_panic---a-value-for-c-instead-of-an-abort)). **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_catch_push(rec: Ptr<u8>);      // push rec; compiled code then calls _setjmp(rec + 64)
    fn nabi_catch_pop(rec: Ptr<u8>);       // the entry returns normally: drop its record
}
```

The entry keeps the record in its frame: `64 + 768` bytes, 16-aligned, the runtime's part first and the `jmp_buf` at offset 64. It calls `nabi_catch_push`, then `_setjmp` on the `jmp_buf`. A recovery restores the shadow stack and the thread state the record saved and `longjmp`s there, and the entry returns its `#[on_panic]` value; on the normal path the entry calls `nabi_catch_pop`. wasm32 and the targets without an OS have no records.

### `nabi_errno_save`

errno for an `#[errno]` call ([stdlib 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errno---errno-and-getlasterror)). **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_errno_save();                  // errno, and GetLastError on Windows, into this thread's slots
}
```

Compiled code calls it right after the C call returns, before it leaves the native state or unpins arguments: both may run code that changes errno. It neither allocates nor blocks.

### `nabi_nosp_enter` / `nabi_nosp_leave`

The check bracket around a call to a `#[no_safepoint]` runtime function ([8.4.10](804-safepoint.md#8410-native-calls)). Only a debug build of nabic emits it. **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_nosp_enter();                  // raise this thread's no_safepoint depth (8.3.1)
    fn nabi_nosp_leave();                  // lower it
}
```

While the depth is nonzero, a debug runtime ends the process with a fatal error if the thread is about to stop for a collection. In a release runtime both functions do nothing.

### `nabi_thread_init` / `nabi_thread_shutdown`

Worker thread lifecycle. **Thread-safety: `nabi_thread_init` is `per-thread-init`, `nabi_thread_shutdown` is `per-thread-shutdown`**.

```
extern "nabi" {
    fn nabi_thread_init();
    fn nabi_thread_shutdown();
}
```

Called at the start and end of each worker thread to initialize or tear down the TLS of the [TLS slots](803-tls-slots.md).

### Internal symbol families

These families are runtime-internal helpers of the reference runtime and are not part of the ABI contract: `nabi_fmt_*`, `nabi_str_*` helpers, `nabi_char_*`, `nabi_nat_*`, `nabi_stdio_*`, `nabi_time_*`, `nabi_gc_stats_*`, `nabi_buf_*`, `nabi_cstr_len`.

### Functions that are not safepoints

On the Nabi side they are declared in an `extern "nabi"` block marked `#[no_safepoint]` ([5.9.2](../005-modules/509-extern-block.md#592-abi-specifiers)).

The following runtime functions are guaranteed never to allocate and never to block, so a call to them is not a safepoint ([8.4.2](804-safepoint.md#842-emission-sites), [8.9.12](809-lowering-control.md#8912-shadow-stack-frame)): `nabi_str_len`, `nabi_str_byte`, `nabi_str_cmp`, `nabi_str_eq`, `nabi_str_eq_at`, `nabi_str_find_near`, `nabi_str_rfind_near`, `nabi_str_char_count`, `nabi_str_bytes_buf`, `nabi_buf_copy`, the `nabi_char_*` property and case functions, `nabi_fmt_len`, `nabi_f64_*`, `nabi_f32_*`.

A call to one of them holds back every other thread's collection until it returns, so its work is kept short. Over a long input (64 KiB or more) the standard library calls a parked variant declared outside the `#[no_safepoint]` block instead ([8.4.10](804-safepoint.md#8410-native-calls)): `nabi_str_find_parked` / `nabi_str_rfind_parked` (the rest of a search after its first 64 KiB, which `nabi_str_find_near` / `nabi_str_rfind_near` look at in place), `nabi_str_char_count_parked`, `nabi_str_grapheme_count_parked`, `nabi_buf_utf8_error_parked`, `nabi_buf_copy_parked`, `nabi_buf_put_str_parked`.

## 8.5.3. Coroutine area

### `nabi_coro_frame_alloc`

Allocates a [coroutine](../007-runtime/708-coroutine.md) frame-sized GC object. **Thread-safety: `thread-safe`**. The compiler does not emit it: coroutine frames are ordinary heap allocations ([8.10](810-lowering-coroutine-generic.md)). The runtime provides it for C and low-level code.

```
extern "nabi" {
    fn nabi_coro_frame_alloc(size: usize, type_id: u32) -> GcBuf<u8>;
}
```

Nearly identical to `nabi_gc_alloc_slow`, but allows frame-specific handling.

### `nabi_coro_park` / `nabi_coro_unpark`

The compiler does not emit these two symbols (`await` uses the parking registry of `core::task`). The runtime provides them for C and low-level code. The address of `frame` (any GC object) is the parking key, and `nabi_coro_unpark(frame)` wakes every waker parked on that key. Calling them outside a task executor aborts.

**Thread-safety: `nabi_coro_park` is `caller-thread-only`** (parks the calling coroutine itself), **`nabi_coro_unpark` is `thread-safe`** (may be called from another thread).

```
extern "nabi" {
    fn nabi_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);
    fn nabi_coro_unpark(frame: Ptr<u8>);
}
```

`nabi_coro_park` suspends the coroutine and registers the waker. `nabi_coro_unpark` resumes the coroutine by pushing it onto the task queue.

Here `waker` is, on the Nabi side, a single-element `GcBuf<Waker>` holding the runtime's `Waker` ([2.20.2](../../refs-standard-libraries/200-core/220-future.md)), as in 8.5.5. Across the C ABI it is a pointer to that buffer, shown as `Ptr<u8>` in the listings, including [7.8.8](../007-runtime/708-coroutine.md) (likewise for `nabi_io_register` and `nabi_timer_register`). The layout of `Waker` is the coroutine implementation's.

## 8.5.4. Scheduler area

The task runtime (spawn, join, cancel, executor) is Nabi code in `core::task` ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)). For it the runtime provides the executor slot, thread and parking symbols below; further scheduler helpers (`nabi_sched_*`, `nabi_lane_*`, `nabi_sync_*`, `nabi_deque_*`) are implementation-defined (8.5.12).

### `nabi_task_worker_*` / `nabi_time_now_ns`

Functions that store and retrieve the current thread's executor worker ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)) in the TLS slot `nabi_tls_task_worker` ([8.3.1](803-tls-slots.md#831-tls-slot-structure)), plus a clock. **Thread-safety: `caller-thread-only`** (`nabi_time_now_ns` is `thread-safe`).

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

Creates a 1:1 OS thread. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_thread_spawn(f: NabiFn, stack_size: usize, id: u64) -> usize;
    fn nabi_thread_new_id() -> u64;
    fn nabi_thread_id() -> u64;                        // worker id of the current thread
    fn nabi_thread_sleep_ns(ns: i64);
    fn nabi_thread_yield_now();
}
```

`nabi_thread_spawn` runs the Nabi closure `f` (a `NabiFn`, passed by value) on an OS thread (`stack_size` 0 means the default). `id` is a value obtained from `nabi_thread_new_id` and becomes the new thread's `nabi_tls_worker_id`. The return value is a join handle, or 0 if creation fails. `env` is a GC root until the thread finishes.

### `nabi_task_hooks_install` / `nabi_task_key`

The executor of `core::task` installs hooks into the runtime. The runtime's I/O driver, timer, and `nabi_coro_park` / `nabi_coro_unpark` use these hooks to wake and suspend tasks. **Thread-safety: `thread-safe`**.

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

`nabi_task_key` uses the address as a key, so it sets `NOMOVE` ([8.1.3](801-object-header-abi.md#813-gc_flags-offset-4-u32)) on the object. The object is never moved afterwards. Identity comparisons used only within a call use `buf_same` ([stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md)), which pins nothing.

### `nabi_sync_wait` / `nabi_sync_wake`

Waiting and waking for synchronization primitives (`Mutex`, `Condvar`, `Once`). **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_sync_wait(buf: Ptr<u8>, index: usize, expected: i32, timeout_ns: i64);   // wait while the GcBuf<i32> element equals expected. timeout_ns < 0 means forever
    fn nabi_sync_wake(buf: Ptr<u8>, index: usize);                                    // wake all threads waiting on that element
}
```

`nabi_sync_wait` may return without a reason to wake, so the caller re-checks its condition. While waiting, the thread is considered stopped with respect to the GC.

### `nabi_thread_join`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_thread_join(handle: usize) -> i32;          // 0 on success
}
```

### `nabi_thread_detach` / `nabi_cpu_count`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_thread_detach(handle: usize) -> i32;   // OS resources are released when it ends; can no longer be joined. 0 on success
    fn nabi_cpu_count() -> usize;                  // number of online CPUs (at least 1); size of the par pool
}
```

### `nabi_static_init_begin` / `nabi_static_init_end`

Back the lazy initialization of `static`s whose initializer is not `const` ([5.3](../005-modules/503-static.md)). **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_static_init_begin(state: Ptr<u8>, at: ConstPtr<u8>, at_len: usize) -> i32;   // 1: caller runs the initializer, 0: already ready
    fn nabi_static_init_end(state: Ptr<u8>);            // publish initialization completion
}
```

`state` is the state byte of that `static` (0 new, 1 in progress, 2 ready). If another thread is initializing, `nabi_static_init_begin` waits until it finishes, during which the thread is considered stopped with respect to the GC. `at` / `at_len` is the `path:line:col` of the code reading the static; an initializer that reads its own static panics there ([4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors)).

### `nabi_gc_add_root`

```
extern "nabi" {
    fn nabi_gc_add_root(word: ConstPtr<usize>);   // register a static's reference word as a GC root until process exit
}
```

Startup code generated by the compiler calls this for every `static` that holds a GC reference.

### `nabi_os_*` - OS helpers (internal, subject to change)

Internal symbols backing the standard library `std` (fs, io, env, process). They are not covered by ABI stability and may change, so user code does not call them directly.

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

On Windows the runtime also exports `nabi_os_raw_handle`, `nabi_os_raw_socket`, `nabi_os_handle_fd` and `nabi_os_socket_fd`, which convert between fds and `HANDLE` / `SOCKET` values.

### `nabi_stdio_*` / `nabi_stdin_read` - standard streams (internal, subject to change)

Internal symbols backing `print` / `eprint` and standard input. They are not covered by ABI stability and may change. Byte regions are `(buf, off, len)` of a `GcBuf<u8>`; failures are returned as `-errno`.

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

### `nabi_deque_*` - work-stealing deque

The lock-free Chase-Lev deque of the task scheduler ([7.6.2](../007-runtime/706-scheduler.md#762-work-stealing-structure)). The deque and its ring are GC objects traced by the GC, so elements that exist only inside a deque are kept alive. **Thread-safety:** `push` / `pop` only from the owning thread; `steal` and `len` from any thread.

```
extern "nabi" {
    fn nabi_deque_new() -> GcBuf<u8>;                                 // empty deque (GC object)
    unsafe fn nabi_deque_push(d: Ptr<u8>, item: usize);               // owner: push at the end (GC reference)
    unsafe fn nabi_deque_pop(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;    // owner: newest element into out[index], 1 if any
    unsafe fn nabi_deque_steal(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;  // oldest element into out[index], 1 if any
    unsafe fn nabi_deque_len(d: Ptr<u8>) -> usize;                    // element count (approximate under contention)
}
```

`out` is a `GcBuf` holding GC references; the result is written at `index`. These functions contain no safepoints.

## 8.5.5. I/O / Timer area

### `nabi_io_register` / `nabi_timer_register`

Let C or low-level code register a task to be woken when an fd is ready or at a point in time. Internally they use the reactor ([7.7](../007-runtime/707-io-and-timer.md)) and the executor's timer. Calling them outside a task executor aborts.

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_io_register(fd: i32, interest: u32, waker: Ptr<u8>);
    fn nabi_timer_register(deadline: u64, waker: Ptr<u8>);
}
```

- `fd`: the file descriptor.
- `interest`: `1` = read, `2` = write (the two may be ORed).
- `waker`: a single-element `GcBuf<Waker>`. Registration is edge-triggered and wakes **only once** (spurious wakeups are possible). If the fd is already ready or cannot be watched, it wakes immediately.
- `deadline`: an absolute time (ns) on the `nabi_time_now_ns` clock. When that time is reached, the waker is woken once.

### `nabi_io_*` - reactor (internal, subject to change)

Internal symbols making up the reactor of the [I/O driver](../007-runtime/707-io-and-timer.md). They are not covered by ABI stability and may change. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_io_poller() -> i32;                    // the process's kqueue / epoll fd (created on first call)
    fn nabi_io_watch(fd: i32, interest: u32) -> i32;   // register fd edge-triggered (re-registering is harmless); interest 1 read, 2 write; 0 or -errno
    fn nabi_io_poll(out: Ptr<u8>, max: usize) -> i32;   // blocks until something changes (parked for the GC), event count or -errno
    fn nabi_io_armed() -> i32;                     // WASI only: 1 if any watch is armed
    fn nabi_io_wait(out: Ptr<u8>, max: usize, timeout_ns: i64) -> i32;   // WASI only: nabi_io_poll with a timeout (< 0: none); 0 = timed out
}
```

`nabi_io_poll` writes up to `max` events into `out`, a `GcBuf<i64>`, as `fd * 2 + direction` (read 0, write 1).

### `nabi_net_*` - socket helpers (internal, subject to change)

Internal symbols backing the standard library `std::net` ([stdlib 3.2](../../refs-standard-libraries/300-std/302-net.md)). They are not covered by ABI stability and may change, so user code does not call them directly. Sockets are always created non-blocking, and failures are returned as `-errno`.

```
nabi_net_socket        nabi_net_bind          nabi_net_listen        nabi_net_connect
nabi_net_sock_error    nabi_net_accept        nabi_net_addr          nabi_net_listeners
nabi_net_send          nabi_net_recv          nabi_net_send_to       nabi_net_recv_from
nabi_net_shutdown      nabi_net_setopt        nabi_net_multicast_v4  nabi_net_wait
nabi_net_resolve       nabi_net_unix          nabi_net_unix_pair     nabi_net_unix_path
nabi_net_send_fd       nabi_net_recv_fd       nabi_net_sendfile      nabi_net_send_zerocopy
nabi_net_splice        nabi_net_getopt
```

`nabi_net_socket` takes the domain, type and protocol of `net::Socket`; `nabi_net_setopt` / `nabi_net_getopt` set and read socket options by codes the library defines. `nabi_net_wait` is a `poll(2)` with which a blocking call waits for readiness (a native section for the GC); `nabi_net_resolve` is `getaddrinfo`; its failure code `e` is returned as `-(100000 + |e|)`, which `nabi_os_strerror` and `nabi_os_errno_kind` accept like an errno. `nabi_net_unix*`, `nabi_net_send_fd` / `nabi_net_recv_fd`, `nabi_net_sendfile`, `nabi_net_send_zerocopy` and `nabi_net_splice` are not exported on Windows.

## 8.5.6. Error propagation (no symbols)

`throw` / `try` / `catch` from [Error control flow](../003-expressions/314-error-control-flow.md) have no runtime symbols. A throwing function returns the error as part of its return value ([8.6.5](806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)), and the call site checks that value and branches ([8.9.6](809-lowering-control.md#896-try--catch--finally-value-based-branching)). There is therefore no runtime call to throw an error or fetch the current error, and there are no TLS slots or jmp_buf for error handling.

## 8.5.7. Diagnostic area

Exit codes below (`101` etc.) reach the host as plain failure on `wasm32-wasip2` ([4.16.4](../004-functions/416-entry-point.md#4164-exit-codes-and-unhandled-exceptions)).

### `nabi_panic`

**Thread-safety: `thread-safe`** (callable from any thread; the process exits immediately).

```
extern "nabi" {
    fn nabi_panic(msg: ConstPtr<u8>, msg_len: usize) -> !;
}
```

An unrecoverable panic; fatal. This function does not return: it prints the message to stderr and terminates the process with exit code `101`. `catch` and `finally` are not run.

### `nabi_panic_str`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_panic_str(msg: str) -> !;          // msg is (ptr, len)
}
```

Same as `nabi_panic` but takes the message as a `str`. It prints no location and is used only for panics internal to the runtime.

### `nabi_panic_at`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    #[track_caller]
    fn nabi_panic_at(msg: str) -> !;
}
```

The declaration is `#[track_caller]` ([6.2.12](../006-meta-system/602-built-in-attributes.md#6212-track_caller-passing-the-call-site)), so the symbol takes the caller's location as a hidden trailing `str` argument (`"path:line:col"`): `nabi_panic_at(msg, at)`.

Prints in the format of [4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors) and exits with code `101`. Core's `panic` and compiler-generated panics (bounds checks, overflow, division by zero, etc.) call this symbol.

### `nabi_uncaught_error` / `nabi_uncaught_error_at`

```
extern "nabi" {
    fn nabi_uncaught_error(what: str) -> !;              // uncaught exception: print what to stderr, exit code 1
    fn nabi_uncaught_error_at(what: str, at: str) -> !;  // followed by a "  --> <at>" line
}
```

Handle errors propagated out of the entry point ([4.16.4](../004-functions/416-entry-point.md#4164-exit-codes-and-unhandled-exceptions)). The declarations live in core.

### `nabi_oom`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_oom() -> !;
}
```

Prints `fatal error: out of memory` to stderr and exits with code `101`. Allocation failures inside the runtime and core's `oom()` both go through this symbol. The runtime cannot call Nabi functions by name, so on allocation failure it uses this symbol directly instead of calling the function in the `oom_fn` slot.

### Stack overflow

When a thread's stack overflows, the runtime flushes stdout (if possible), prints the following to stderr and aborts (SIGABRT). Unlike a panic, there is no unwinding and no `finally` execution.

```
thread '<name>' has overflowed its stack
fatal runtime error: stack overflow, aborting
```

`<name>` is the thread name given with `thread::Builder::name` (also set as the OS thread name, [7. Runtime](../007-runtime/README.md#runtime-error-stack-overflow)). This covers the main thread (`nabi_runtime_init`), `nabi_thread_spawn` threads (including executor workers, and deep `await` chains), and C threads that called `nabi_thread_init`. The runtime records the guard range below each thread's stack in the TLS slot `nabi_tls_stack_guard` ([8.3.1](803-tls-slots.md#831-tls-slot-structure)) and installs an alternate signal stack (`sigaltstack`). Only SIGSEGV / SIGBUS within that range is reported as a stack overflow; other faults go to the action set before the runtime installed its handler (the default action, unless a host process that loads Nabi as a shared library set one). On wasm a module cannot catch the trap; the host reports it (the shipped JS host prints the message above).

## 8.5.8. Symbol naming conventions

The area of each symbol prefix is summarized below.

| Prefix          | Area           |
|-----------------|----------------|
| `nabi_gc_*`      | GC             |
| `nabi_runtime_*` | Runtime in general |
| `nabi_thread_*`  | OS threads     |
| `nabi_coro_*`    | Coroutines     |
| `nabi_task_*`    | Tasks          |
| `nabi_io_*`      | I/O            |
| `nabi_timer_*`   | Timers         |
| `nabi_panic`     | Diagnostics    |

This prefix convention avoids symbol collisions with external libraries and user code. The tables of 8.5.1 through 8.5.7 list the normative runtime symbols; the runtime may export further implementation-defined `nabi_*` symbols (e.g. `nabi_fmt_*`, `nabi_os_*`, `nabi_stdio_*`, `nabi_static_init_*`).

## 8.5.9. Compatibility of function signatures

Each function signature is part of the [ABI](../000-meta/007-glossary.md#abi-application-binary-interface). The change policy is:

- before 1.0: breaking changes are possible in every 0.x.0 release; a 0.x.y release makes only compatible changes.
- from 1.0, minor: only compatible changes are allowed; adding arguments is handled through function wrappers.
- from 1.0, major: breaking changes are possible, and the ABI version hash changes.

See the stability policy in the [overview](README.md) for details.

## 8.5.10. Calling convention

All functions above use the target platform's C ABI calling convention, e.g. System V x86_64, AAPCS aarch64, MS x64 Windows.

## 8.5.11. Example: slow call from the TLAB fast path

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

The example shows the structure of the emitted code combining the fast path and the slow path (pseudo-code; the header stores of the fast path are in [8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path)).

## 8.5.12. Auxiliary symbols (appendix, non-normative)

> [!IMPORTANT]
> This section is **non-normative**. The symbols below are auxiliary symbols exported by the reference runtime (`nabir`). They are not part of the main ABI tables (8.5.1 ~ 8.5.11) and carry no stability promise; the runtime may export further implementation-defined `nabi_*` symbols (8.5.8).

Signatures use the same notation as the main text (`Ptr<T>` = mutable raw pointer, `ConstPtr<T>` = read-only raw pointer). String arguments are NUL-terminated C strings.

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

The following rules apply.

- User code must not depend on these symbols. Their signatures may change or be removed without notice. The standard library ships with the runtime and calls some of them (`nabi_free`, `nabi_gc_weak_new`, `nabi_gc_weak_get`, `nabi_gc_pinned`, `nabi_panic_set_hook`, the `nabi_handle_*`, `nabi_errno_*`, `nabi_last_error_*`, `nabi_session_*` and `nabi_lock_*` functions).
- They share the `nabi_` prefix but are not included in the area table of 8.5.8. To avoid name collisions, normative symbols of the specification do not reuse these names.
- `nabi_panic` does not belong to this appendix. The signature `nabi_panic(msg: ConstPtr<u8>, msg_len: usize)` in [8.5.7](#857-diagnostic-area) is authoritative, and an implementation exporting a different signature is non-conformant.
