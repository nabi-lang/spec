# 8.3. Mutator TLS slot layout

Each OS thread (= scheduler worker = mutator) holds runtime information at fixed locations in thread-local storage (TLS). Code emitted by the compiler and the runtime read and write the same TLS slots, so this document defines the offset and type of each slot.

## 8.3.1. TLS slot structure

The TLS slots are one `thread_local` block, `nabi_tls`, defined and exported by the runtime; each slot is at a fixed offset in it. Compiled code reaches each slot at its offset from the block's address, which a Nabi function receives from its caller (8.3.8).

`p` is the pointer width in bytes.

| Offset   | Slot                        | Type | Meaning                                 |
|----------|-----------------------------|------|-----------------------------------------|
| 0        | `nabi_tls_shadow_stack_top`  | ptr  | Head of the shadow stack chain          |
| p        | `nabi_tls_tlab_cursor`       | ptr  | Next allocation position in the TLAB    |
| 2p       | `nabi_tls_tlab_limit`        | ptr  | End of the TLAB                         |
| 3p       | `nabi_tls_wb_log_cursor`     | ptr  | Position in the write barrier log       |
| 4p       | `nabi_tls_wb_log_limit`      | ptr  | End of the write barrier log            |
| 5p       | `nabi_tls_satb_cursor`       | ptr  | Position in the SATB buffer (during concurrent mark) |
| 6p       | `nabi_tls_satb_limit`        | ptr  | End of the SATB buffer                  |
| 7p       | `nabi_tls_task_worker`       | ptr  | This thread's executor worker (GC object, 0 if none) |
| 8p       | `nabi_tls_stack_guard`       | [usize; 2] | `[lo, hi)`: a fault in this range is a stack overflow of this thread (`[0, 0)` means unknown, [8.5.7](805-extern-symbols.md#stack-overflow)) |
| 10p      | `nabi_tls_worker_id`         | u64  | Worker id                               |
| 10p+8    | `nabi_tls_yield_flag`        | u8   | Whether to `yield` at a safepoint       |
| 10p+9    | `nabi_tls_mutator_state`     | u8   | Mutator state                           |

The **ABI contract** is the block symbol `nabi_tls` and each slot's offset, type and meaning. Slot names are for reference only; they are not symbols. The block is 8-aligned and at least 10p+10 bytes; implementation-defined slots may follow the last ABI slot.

**Implementation-defined slots**: the runtime may keep further thread-local state, after the ABI slots or elsewhere, for scheduling, statistics, profiling, debugging and so on. The compiler and external packages must not reference it.

> [!NOTE]
> Nabi's runtime places these slots after the ABI part. The block is 16p+16 bytes.
>
> - 10p+10: `no_safepoint` (`u8`). The depth of `#[no_safepoint]` calls a debug nabic brackets with `nabi_nosp_enter` / `nabi_nosp_leave` ([8.4.10](804-safepoint.md#8410-native-calls)).
> - 10p+11: `attached` (`u8`). How the thread joined the runtime: 0 not yet or no longer, 1 `nabi_thread_init` or a thread the runtime started, 2 attached on the C entry of an `extern "C"` definition (8.3.8).
> - 10p+16: `task_lane`
> - 11p+16: `static_inits`
> - 12p+16: `held_locks`. The locks this thread holds. In a session host, the locks a thread that is gone held while blocked are released.
> - 13p+16: `thread_name` (`{ptr, len}`). The thread's name in reports; null when unnamed.
> - 15p+16: `throw_site`. The address of the NUL-terminated position of the `throw` whose error is in flight, which compiled code stores at a `throw`; null or empty when not known.

## 8.3.2. Access mechanism

Each OS and architecture has its own TLS access mechanism.

| Platform       | Mechanism                             |
|----------------|---------------------------------------|
| Linux x86_64   | thread pointer `%fs:0` + offset       |
| Linux aarch64  | thread pointer `tpidr_el0` + offset   |
| macOS aarch64  | TLV descriptor, called through its getter |
| Windows x86_64 | TEB (Thread Environment Block) TLS array at `_tls_index` + offset |
| `wasm32-wasip1-threads` | `__tls_base` global + offset   |
| other WASM (no threads) | the block is a plain global  |

The compiler emits the appropriate instructions per target and TLS model (8.3.8).

## 8.3.3. Meaning of each slot

### `nabi_tls_yield_flag` (u8)

A flag the [GC](../000-meta/007-glossary.md#gc-garbage-collector) sets for a safepoint; mutators check it when polling. Bit 0 is the runtime's stop request, raised and cleared by the runtime; other bits belong to the embedding host (e.g. the REPL's interrupt) and are left alone by the runtime. A poll calls `nabi_runtime_yield` whenever the flag is nonzero.

The detailed mechanism is defined under Safepoint Polling in [Barriers and roots](../007-runtime/704-barriers-and-roots.md).

### `nabi_tls_tlab_cursor` (ptr)

Points to where the next allocation happens in the [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) of [Allocation](../007-runtime/702-allocation.md).

```
allocate(size):
    tmp = tlab_cursor
    new_cursor = tmp + size
    if new_cursor > tlab_limit:
        slow path
    tlab_cursor = new_cursor
    return tmp
```

These four instructions form the TLAB fast path, which the compiler emits inline.

### `nabi_tls_tlab_limit` (ptr)

Points to the end of the TLAB. When the cursor passes this value, the slow path is entered.

### `nabi_tls_shadow_stack_top` (ptr)

Head of the shadow stack chain. Defined in detail in 8.3.8.

### `nabi_tls_satb_cursor` / `nabi_tls_satb_limit` (ptr)

The SATB buffer recorded by the write barrier slow path during concurrent mark ([7.5](../007-runtime/705-concurrent-collection.md)). When full, it is handed to the collector. Not used outside mark.

### `nabi_tls_wb_log_cursor` / `nabi_tls_wb_log_limit` (ptr)

Cursor and limit of the write barrier log buffer from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md).

```
write_barrier(p, q):
    if !needs_log: return
    if wb_log_cursor >= wb_log_limit: flush
    *wb_log_cursor++ = p
```

Most operations are non-atomic and thread-local.

The two slots point to a per-thread log chunk of 512 words (4 KiB on 64-bit targets, 2 KiB on 32-bit). A full chunk is pushed onto a lock-free stack that the collector drains ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-object-remembering)).

### `nabi_tls_mutator_state` (u8)

The mutator's current state.

| Value | Meaning                       |
|-------|-------------------------------|
| `0`   | RUNNING. Normal execution     |
| `1`   | GC_COOPERATING. Cooperating with the GC |
| `2`   | PARKED. In sleep / wait       |
| `3` ~ | Reserved                      |

This value is used for GC cycle synchronization and safepoint cooperation.

### `nabi_tls_worker_id` (u64)

The thread's worker id (used inside the scheduler), used for [work-stealing](../000-meta/007-glossary.md#work-stealing), statistics and the like.

## 8.3.4. Slot width and alignment

Each slot has the width of its type and lies at an offset aligned to it.

## 8.3.5. 32-bit vs. 64-bit

Pointer slots are p bytes (8 on 64-bit, 4 on 32-bit), so offsets differ between 32-bit and 64-bit as given by the table.

## 8.3.6. TLS slot initialization

When a worker starts, the slots are initialized as follows.

```
nabi_thread_init():
  nabi_tls_yield_flag = 0
  nabi_tls_tlab_cursor = NULL          // filled by the slow path on first allocation
  nabi_tls_tlab_limit = NULL
  nabi_tls_shadow_stack_top = NULL     // empty shadow stack chain (8.3.8)
  nabi_tls_wb_log_cursor = wb_log_base
  nabi_tls_wb_log_limit = wb_log_base + LOG_SIZE
  nabi_tls_mutator_state = RUNNING
  nabi_tls_worker_id = (allocated)
  nabi_tls_task_worker = 0
  nabi_tls_stack_guard = [guard_lo, guard_hi)   // guard range below the stack
```

The scheduler calls `nabi_thread_init` when a worker starts; this function is part of the [extern symbols](805-extern-symbols.md).

## 8.3.7. TLS slot teardown

When a worker exits, it goes through the following steps.

```
nabi_thread_shutdown():
  // log flush
  // shadow stack walk (and a final mark, if possible)
  // return worker_id
```

## 8.3.8. Form of compiler emission

The block is a `thread_local` global defined and exported by the runtime. The compiler references it as an external declaration and loads slots at their offsets from the block's address. A Nabi function takes that address from its caller as a hidden last argument (8.6.6), so a call between Nabi functions reads no TLS. Code entered from outside Nabi takes the address itself, once, and passes it on: the C `main`, the C entry of an `extern "C"` definition, an `#[export_name]` Nabi function, the `any` dispatcher, a lazy `static`'s initializer, and the thunk a function value goes through (8.6.11).

The C entry of an `extern "C"` definition may run on a thread that C code started and the runtime does not know. On such a thread it attaches the thread first: it initializes the slots as `nabi_thread_init` does (8.3.6) and registers the thread as a mutator, then runs the body. Back in C between such calls the thread counts as `PARKED` ([8.4.10](804-safepoint.md#8410-native-calls)), so a collection does not wait for it, and when the thread ends it leaves the registry as `nabi_thread_shutdown` does (8.3.7). A target that cannot attach a thread this way (WebAssembly) aborts instead, with `nabi: callback on a thread nabi does not know; call nabi_thread_init() first`.

```
; nabi_tls: defined by the runtime; compiler output refers to it as an undefined symbol
t = <address of nabi_tls>         ; once, where code enters nabi
                                  ; (a nabi function: its last argument)

; safepoint poll (yield_flag at 10p+8)
flag = load.u8 [t + 88]           ; 10p+8 with p = 8
if flag != 0 then bb_slow else bb_cont
```

On ELF, `nabi_tls` uses the initial-exec TLS model (the compiler emits access code in this model and does not rely on linker relaxation). The runtime is always statically linked into the executable, so this model suffices. Mach-O uses the platform's TLV descriptor scheme, and Windows the thread's TLS array at `_tls_index`. An implementation that cannot choose a TLS model may use general-dynamic instead.

### Shadow stack head

The slot at offset 0 of `nabi_tls` (`nabi_tls_shadow_stack_top`) is the head of the [shadow stack](../000-meta/007-glossary.md#shadow-stack) chain from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler), i.e. the address of the innermost `ShadowFrame` (null if none). It is the only chain head. Code emitted by the compiler links the frame before the first safepoint of each path through the function, and unlinks it immediately before every `return` of a path that linked it (8.4.7).

## 8.3.9. Stability

The offsets, types and meanings in the 8.3.1 table are part of the [ABI](../000-meta/007-glossary.md#abi-application-binary-interface); changes follow the stability policy in the [overview](README.md#stability). Changing an offset, or the type or meaning of an existing slot, breaks the ABI (before 1.0 a 0.x.0 release, from 1.0 a major release). A new slot takes an offset after the existing ones, a compatible change.

Implementation-defined slots are not part of the ABI and may change in any release.
