# 8.11. Code generator contract

This section collects what a code generator must guarantee so that the objects it writes link with `nabic`'s linker, run on the runtime, and mix with objects other code generators wrote: `nabic`'s own ([8.7](807-code-generation.md)), and a separately installed one (8.11.13). Most of it is defined elsewhere in this chapter; this section names each requirement and links to its definition. Where `nabic`'s code generator does more than the contract asks, the text says so.

> [!NOTE]
> Draft for 0.2. In 0.1 `nabic`'s own code generator is the only one, and every requirement below is what it does. The process interface of 8.11.13 and the file format it reads ([9.11](../009-tools/911-mir-format.md)) are not implemented: no 0.1 `nabic` runs a separate code generator, and `-C codegen-backend` is an unknown codegen option.

## 8.11.1. Division of work

A code generator turns the functions of one codegen unit ([9.2.16](../009-tools/902-aot-pipeline.md#codegen-units)) into one relocatable object. `nabic` decides everything that more than one object must agree on, before any code generator runs, and hands the result over with the unit's nabi-mir:

| `nabic` decides | The code generator decides |
|-----------------|----------------------------|
| Optimization of nabi-mir (inlining, [8.7.3](807-code-generation.md#873-optimization-levels)); coroutine state machines ([8.10](810-lowering-coroutine-generic.md)) | Instruction selection, register allocation, scheduling, frame layout within one function |
| The units, and which functions each defines | Its own optimizations within the rules of 8.11.6 and 8.11.7 |
| The symbol of every function and static, its linkage and visibility (8.11.3) | Local symbols of its own (8.11.3) |
| The layout of every type (8.11.5) | |
| GC type ids and their ScanTables ([8.2](802-scan-table.md)) | |
| Which functions never reach a safepoint, which poll at entry, which back edges poll (8.11.7) | Where the poll sits on a polled edge |
| The shared unit: the C `main`, statics, `nabi.register_types`, lazy-static initializers ([9.2.16](../009-tools/902-aot-pipeline.md#codegen-units)) | |
| Linking ([9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables)) | |

The shared unit is always compiled by `nabic`'s own code generator. Objects of different code generators in one program, and objects of one unit compiled again by another code generator, must link and behave the same.

## 8.11.2. Input

The input is the optimized nabi-mir of one unit ([9.11](../009-tools/911-mir-format.md)): the functions the unit defines, with bodies; declarations of every function and static they refer to; the types, layouts and GC types they use; the target; and the build's options (`-O` level, overflow checks, `-g`).

The nabi-mir is the one `--emit=mir` prints ([9.9.1](../009-tools/909-debug-tools.md#991-intermediate-output---emit)). It has no coroutine suspend points (coroutines are state machines over a frame object by then, [8.10.1](810-lowering-coroutine-generic.md#8101-state-machines)) and no floored `%%` (it is expanded). A code generator compiles each statement with the meaning it has in nabi-mir and keeps every value, every trap and every side effect in program order, as [8.7.3](807-code-generation.md#873-optimization-levels) requires of the optimizer.

## 8.11.3. Symbols and linkage

Every function and static carries its symbol in the input: the mangled name ([9.5](../009-tools/905-symbol-mangling.md)), an `#[export_name]`, or an `extern` declaration's C name (with its `@N` stdcall suffix on i686 Windows). A code generator uses these names as given and does not mangle. It defines a symbol as global, weak or hidden exactly as the input says; a function another unit calls is global, a private copy of another package's code is named `<symbol>.copy_Y<hash>_` and is global only within the package's units.

`nabic` and its code generator also use these derived symbols:

| Symbol | Defined by | What it is |
|--------|-----------|------------|
| `<symbol>` of a Nabi function | the unit that defines the function | The body, in the Nabi convention with the hidden `nabi_tls` argument ([8.6.6](806-calling-convention.md#866-nabi-to-nabi-calling-convention)) |
| `<symbol>.c` | the unit that defines an `extern "C"` function without `#[export_name]` (with one, the export name) | Its C entry (8.11.8) |
| `<symbol>.v` | every object that takes the function's address, as a local symbol | The thunk a function value calls: no `nabi_tls` argument; it reads the block's address and calls `<symbol>` ([8.6.11](806-calling-convention.md#8611-function-pointer-abi)) |
| `<static>.state`, `<static>.init` | the shared unit | A lazy static's state byte and initializer (8.11.10) |
| `_Y<pkg>_gctype.<hash>` | the shared unit, in a unit build | A `u32` holding a GC type id (8.11.6) |
| `nabi.scan.<hash>` | the shared unit, weak and hidden | A ScanTable's bytes, named by their content ([8.2.13](802-scan-table.md#8213-registration-time)) |
| `nabi.register_types`, `nabi.register_types.<lib>` | the shared unit | GC type and static root registration ([8.2.13](802-scan-table.md#8213-registration-time)) |
| `nabi.any.<hash>` | the unit of the `any` dispatcher; weak in a library | The `any` method dispatcher ([9.4.8](../009-tools/904-nabihir-bytecode.md#948-rules-for-separate-compilation)) |
| `nabi.str.<hash>` | any object, as a local symbol | A string literal's bytes followed by a NUL byte |
| `nabi_tls` | the runtime | The thread-local block ([8.3](803-tls-slots.md)) |

Any other symbol a code generator makes itself is local to its object. `nabic`'s own use names starting with `nabi.` (`nabi.i386.udivmod64`, ..).

The bytes of every symbol name must appear in the object as they are, uncompressed: `nabic` writes a unit's objects with a placeholder for the package hash and replaces `_Y<placeholder>_` with `_Y<hash>_` in the object's bytes before linking ([9.2.16](../009-tools/902-aot-pipeline.md#codegen-units)).

A string literal's bytes end with a NUL byte that is not part of the `str`: a throw site ([8.11.11](#81111-errors-panics-and-unwinding)) and a `str` passed as `char*` are read up to it.

## 8.11.4. Calling conventions

- Between Nabi functions: [8.6.6](806-calling-convention.md#866-nabi-to-nabi-calling-convention). Every call a Nabi function can receive from another object follows it exactly, including the hidden `nabi_tls` address as the last argument; aggregates over 64 bytes go by address and come back through a leading result pointer. Only a function no other object can reach (local, never used as a value) may use a convention of its own.
- `extern "C"` declarations and definitions, and calls through C function pointers: the platform C ABI with aggregate classification ([8.6.1](806-calling-convention.md#861-default-platform-c-abi) ~ [8.6.3](806-calling-convention.md#863-struct-arguments--return-small-struct-rule)), narrow-integer extension, stdcall on i686 Windows for `extern "system"`, and C variadics ([8.6.10](806-calling-convention.md#8610-variadic-functions)). The input gives each such signature's classification as `nabic` makes it ([9.11.9](../009-tools/911-mir-format.md#9119-functions)); a code generator places the classified parts and does not classify again.
- A `str` argument of an `extern "C"` call is passed as `char*`: the code calls `nabi_str_cstr(s)` before the call, passes the pointer it returns, and calls `nabi_str_cstr_done(s, p)` after the call (the pointer is `s`'s own when a NUL byte follows its bytes, else a pinned copy).
- `extern "nabi"` runtime declarations: the C ABI, with a `str` as a by-value `(ptr, len)` C struct ([8.5](805-extern-symbols.md)).
- Function values: `{code, env}`, the environment first, no `nabi_tls` argument ([8.6.11](806-calling-convention.md#8611-function-pointer-abi), [8.6.12](806-calling-convention.md#8612-closure-calls)).
- Throwing functions return their carrier `(is_err, payload)` ([8.6.5](806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)); in nabi-mir that is an ordinary return value of the type the input gives.
- Stack probes follow the target's `stack_probes` ([0.6.5](../000-meta/006-platform-targets.md#065-target-specifications)): a frame of a page or more touches each page in order, inline or through `__chkstk` (Windows; `nabic`'s startup object defines it). The runtime reports a fault in the guard range as a stack overflow ([8.5.7](805-extern-symbols.md#stack-overflow)).

## 8.11.5. Data layout

The layout of every nabi-mir type is fixed by the target, and the input gives each type's size, alignment and field offsets ([9.11.6](../009-tools/911-mir-format.md#9116-types)). A code generator uses them as given:

| nabi-mir type | Layout |
|---------------|--------|
| `()` | size 0, alignment 1 |
| `i8` `u8` `bool` `c_char` | 1 byte; a `bool` is 0 or 1 |
| `i16` `u16` | 2 bytes |
| `i32` `u32` `f32` `char` | 4 bytes; a `char` is a Unicode scalar value |
| `i64` `u64` `f64` | 8 bytes, aligned to the target's `wide_align` (4 on i686 Linux, else 8) |
| `isize` `usize`, a GC reference, a function pointer, an unresolved word | one pointer-width word |
| `str` | two words `(ptr, len)`; `ptr` may point into a GC object |
| tuple, struct | C layout of the fields in order; `#[repr(packed)]`: no padding, alignment 1; `#[repr(align(N))]`: at least `N` |
| `[T; N]` | `N` elements of `T`'s size |
| union of `n` words | `n` words, aligned to its most aligned variant and at least to a word |
| sum, enum | `(i32 tag, payload)` or a niche layout ([8.8.6](808-lowering-types.md#886-representation-of-enum), [8.8.8](808-lowering-types.md#888-representation-of-sum-types-a--b)) |
| `any Tr` | `(u32 tag, payload word)` ([8.8.9](808-lowering-types.md#889-representation-of-any-slots)) |
| function value | `(code, env)` |

All targets but `s390x` are little-endian. A GC object is a 16-byte header ([8.1](801-object-header-abi.md)) followed by its payload; its size is that of [8.1.7](801-object-header-abi.md#817-total-size-of-header--payload).

## 8.11.6. GC: allocation, roots and barriers

**Allocation.** Every GC object a code generator allocates (`HeapBox`, `HeapAlloc`, `BufAlloc` in nabi-mir) gets the type id the input gives with it: a constant, or in a unit build the `u32` at `_Y<pkg>_gctype.<hash>`. A `HeapBox` of a number, `bool` or `char` no wider than a word allocates nothing: the payload word is the value ([8.2.7](802-scan-table.md#827-any-slots)). Allocation is the TLAB fast path of [8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path) or a call to `nabi_gc_alloc_slow(size, align, type_id)`; objects of `LOS_THRESHOLD` (8 KiB) or more, or aligned past 8, always call it. The fast path writes `type_id` and `gc_word` and leaves `gc_flags` zero; the payload is zero. A code generator may always call `nabi_gc_alloc_slow` instead.

**Roots.** At every safepoint (8.11.7), every word a GC reference may be in, of every local live across that safepoint, is in memory and registered in a shadow-stack frame linked into `nabi_tls` offset 0 ([7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler), [8.9.12](809-lowering-control.md#8912-shadow-stack-frame)):

- The frame is `{prev: ptr, count: usize, slots: [ptr; count]}` on the machine stack (in linear memory on wasm32); each slot holds the address of one word, or a slot `(n << 1) | 1` and the next the address of the first of `n` adjacent words. With `count` 0 the frame has one slot: the address of a descriptor in read-only data, `{count: u16, entries: [u16; count]}`, its entries the slots' with each address twice the word's offset from the frame in words, signed ([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)). The words a local needs are those the input gives as the type's root offsets: every word any variant may hold a reference in, an `any`'s payload word, a `str`'s pointer word.
- The frame is linked before the first safepoint of each path and unlinked before every return of a path that linked it; where paths join, all or none have linked. A registered word is zero or a valid value whenever a collection can read it: zeroed before the link, kept current at every write.
- What is live across a safepoint: the locals live after it; at a call to a runtime function or to C, also its operands; at a call to a Nabi function, only what the caller reads afterwards (the callee roots its own parameters). A local whose address is taken (`AddrOf`), and every local the function lists as pinned, stays registered until its `StorageDead`.
- A local only back-edge polls see may instead be registered by the poll's slow path alone ([7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler) rule 4). `nabic`'s code generator links one frame at entry and registers every such local in it.
- A dead local's word may be cleared to zero (`StorageDead`, the end of its last use). This keeps it from retaining garbage; it is not required.

**Registers and moving objects.** A full collection may move objects (evacuation, [7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)); it rewrites every precise reference in the heap and never rewrites a root word. An object stays where it is when a registered word, an ambiguous word or a pin reaches it, and a large object never moves. Therefore:

- A GC reference may stay in a register across a safepoint only as a copy of a word that is registered across that safepoint; so may a pointer into that object.
- Any other GC reference or pointer into a GC object, such as a reference loaded from a field of a heap object, is not live across a safepoint: it is loaded again afterwards. A code generator must not keep such a value in a register, a spill slot or a stack slot that is not registered across a safepoint.
- `nabic`'s code generator keeps no root in a register: a variable holding a GC reference across a safepoint lives in its registered stack slot.

**Write barrier.** Every store of a value with GC words into a GC object (a field of a heap object, a buffer element, a coroutine frame slot) has the barrier of [8.9.10](809-lowering-control.md#8910-write-barrier): test `gc_flags & (UNLOGGED | CARDED)` of the destination object; if `UNLOGGED`, call `nabi_gc_wb_slow(obj, slot, len)` (the slot's address and the stored value's size); if `CARDED`, store `1` into the card bytes the written slot spans, at `card_table + ((slot - obj) >> 9)`, the card table's address being the word before the header. No safepoint lies between the barrier and its store. Stores into a just-allocated object before the next safepoint, into statics, into raw memory, and of values with no GC words need none. The barrier never reads the stored value's header.

The collector requires nothing else from compiled code ([7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)): there are no stack maps and no register maps.

## 8.11.7. Safepoints and polls

A safepoint is a point where the thread may stop for a collection: a call to a Nabi function the input does not mark as never reaching one, a call through a function value, a call to a runtime function that is not `#[no_safepoint]`, a call to C (it parks, 8.11.8), an allocation, a read, write or address of a lazy static, and a poll.

- **Poll.** Load the `u8` at `nabi_tls + 10p+8` (`yield_flag`); if it is nonzero, call `nabi_runtime_yield()` ([8.4.1](804-safepoint.md#841-yield-flag-check-fast-path)).
- **Back edges.** Every back edge the input marks as polled has a poll on it ([8.4.8](804-safepoint.md#848-loop-back-edge-poll)). The unmarked ones belong to loops of at most 64 iterations that reach no safepoint ([8.4.2](804-safepoint.md#842-emission-sites)).
- **Entry.** A function the input marks as re-entrant polls before its first safepoint on each path, and at entry when it is also entered other than by a direct Nabi call (from C, through a function value, exported, the `any` dispatcher) and reaches a safepoint at all ([8.4.7](804-safepoint.md#847-safepoint-invariant), [8.9.11](809-lowering-control.md#8911-safepoint-poll)).
- A function the input marks as never reaching a safepoint needs no frame and no poll, and a call to it is not a safepoint.

Extra polls are allowed. A vectorized loop has none inside, and runs at most 1024 iterations before the scalar loop's poll runs again ([8.7.8](807-code-generation.md#878-loop-vectorization)).

## 8.11.8. Calls into C and from C

- **Into C.** Around every call to an `extern "C"` function whose symbol does not start with `nabi_`: before it, a sequentially consistent store of `2` (PARKED) to `mutator_state` (`nabi_tls + 10p+9`), then a load of `yield_flag` and, if nonzero, a call to `nabi_enter_native()`; after it, a store of `0` (RUNNING), the same load and, if nonzero, `nabi_leave_native()` ([8.4.10](804-safepoint.md#8410-native-calls)). The caller's roots stay registered across the call.
- **From C.** An `extern "C"` definition has a C entry (`<symbol>.c` or its export name) that takes the C arguments, calls `nabi_native_reenter()` (which also attaches a thread the runtime does not know, [8.3.8](803-tls-slots.md#838-form-of-compiler-emission)), calls the body with this thread's `nabi_tls` address, calls `nabi_native_resume(<what reenter returned>)`, and returns the C way. On wasm32 it first calls `nabi_init_once()` when the function is exported.
- Calls to `#[no_safepoint]` runtime functions and to `nabi_*` symbols are not bracketed.

## 8.11.9. Thread-local access

A Nabi function reads `nabi_tls` slots at their offsets ([8.3.1](803-tls-slots.md#831-tls-slot-structure)) from the address it received as its last argument. Code entered from outside Nabi (a C entry, a function-value thunk, an exported Nabi function, the `any` dispatcher) computes the address once ([8.7.7](807-code-generation.md#877-tls-access)). The access sequence, and so the relocations, depend on the object format:

| Target | Model | Relocations `nabic` uses |
|--------|-------|--------------------------|
| ELF x86_64 / aarch64 / i686 / armv7 / riscv64 / powerpc64le / s390x | initial exec, thread pointer plus the offset from a GOT slot | `R_X86_64_GOTTPOFF`, `R_AARCH64_TLSIE_*`, `R_386_TLS_GOTIE`, `R_ARM_TLS_IE32`, `R_RISCV_TLS_GOT_HI20`, `R_PPC64_GOT_TPREL16_*`, `R_390_TLS_IEENT` |
| ELF riscv32 (no OS) | local exec | `R_RISCV_TPREL_HI20` / `_ADD` / `_LO12_I` |
| Mach-O arm64 | thread-local variable descriptor, called through its getter | `ARM64_RELOC_TLVP_LOAD_PAGE21` / `_PAGEOFF12` |
| COFF | the TEB's TLS array at `_tls_index`, plus the section-relative offset | `SECREL` (arm64: `SECREL_HIGH12A` / `_LOW12A`) |
| wasm32 with threads | `__tls_base` plus the offset | `R_WASM_MEMORY_ADDR_TLS_SLEB` |
| wasm32 without threads | a plain global | memory address |

The runtime is always linked statically, so these models suffice; a code generator may use general dynamic instead where the linker resolves it.

## 8.11.10. Statics

A static's storage, initial bytes and root registration belong to the shared unit. A code generator reads and writes a static through its symbol, through the GOT where the object format has one and the static is defined outside the program's own objects (in a prebuilt library or a C library), and on Windows through `__imp_<symbol>` for a static a DLL defines. Stores into statics have no write barrier (statics are roots).

Before the first read, write or address of a lazy static, code loads its state byte `<static>.state` atomically and, unless it is `2` (initialized), calls `<static>.init(at: str)` with the source position of the access (`path:line:col`), in the Nabi convention without the `nabi_tls` argument. This is a safepoint.

## 8.11.11. Errors, panics and unwinding

Nothing in a Nabi program unwinds the machine stack.

- `throw` and `try` are ordinary returns and branches on the carrier ([8.6.5](806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation), [8.9.6](809-lowering-control.md#896-try--catch--finally-value-based-branching)); every frame leaves through its own return path and unlinks its shadow-stack frame.
- A `throw` records its position by a call to `nabi_throw_site(at: str)`; `nabic`'s code generator instead stores the literal's address into the implementation-defined slot the runtime reads ([8.3.1](803-tls-slots.md#831-tls-slot-structure)), which a code generator may do only with the runtime it ships with.
- A panic is a call to a function that does not return (`nabi_panic_at`, ..); it ends the process and runs no `finally`. The checks a code generator emits itself panic through `nabi_panic_at(msg, at)`, `at` being the position of the statement being executed (`path:line:col`), or in a `#[track_caller]` function its last parameter; the runtime prints it as [4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors) shows. The checks and their messages:

| Check | When | Message |
|-------|------|---------|
| Integer `+` `-` `*` overflow | overflow checks on, not `wrapping_*`, operands not constant-safe | `attempt to add with overflow`, `attempt to subtract with overflow`, `attempt to multiply with overflow` |
| Integer negation of the minimum | overflow checks on | `attempt to negate with overflow` |
| Division and remainder | always | `attempt to divide by zero`, `attempt to divide with overflow`, `attempt to calculate the remainder with a divisor of zero`, `attempt to calculate the remainder with overflow` |
| Shift by the width or more | always | `attempt to shift left with overflow`, `attempt to shift right with overflow` |
| Array index (`ArrayIndex`, `ArrayWrite`) | always, unless the index is a constant in range | `index out of bounds` |

Other checks (slices, `List`, `unwrap`) are calls in nabi-mir already.

Unwind tables are for debuggers, profilers and the operating system, not for the program. A code generator emits for every function the unwind information of its object format: `.eh_frame` FDEs on ELF (armv7: `.ARM.exidx` entries), compact unwind or `__eh_frame` on Mach-O, `.pdata` / `.xdata` on COFF x86_64 and arm64 (i686 has none; frames are walked through `ebp`). No CIE names a personality routine and no LSDA is emitted; on armv7 a compact `.ARM.exidx` entry implies `__aeabi_unwind_cpp_pr0`, which the runtime defines weakly. `nabic`'s code generator keeps a frame pointer in every function, so profilers can walk the stack without tables.

## 8.11.12. Objects

A code generator writes one relocatable object in the target's format, which `nabic`'s linker reads ([9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables)):

| Format | Requirements |
|--------|--------------|
| ELF | `ET_REL`, the machine and flags of the target's ABI (powerpc64le: ELFv2, `e_flags` 2; armv7: EABI5 hard-float; riscv: RVC, and on riscv64 the double-float ABI). 64-bit targets: RELA relocations only; 32-bit: REL or RELA. No compressed allocated section (`SHF_COMPRESSED` debug sections are dropped). A `.note.GNU-stack` section without `SHF_EXECINSTR` |
| Mach-O | `MH_OBJECT`, arm64 only, `MH_SUBSECTIONS_VIA_SYMBOLS` |
| COFF | the target's machine; i686 C names carry the `_` prefix; `/bigobj` format past 65279 sections |
| wasm | a relocatable module following the WebAssembly tool conventions (`linking` section version 2, `reloc.CODE` / `reloc.DATA`), importing `__linear_memory`, `__stack_pointer`, `__indirect_function_table` and, with threads, `__tls_base` from `env`; each function's type is the one its convention gives, the `nabi_tls` address as the last `i32` parameter |

- Each function and each data object should be in a section of its own (ELF), a COMDAT (COFF), a subsection (Mach-O) or a segment of its own (wasm): the linker drops unreferenced ones and, in a `release` build, folds identical functions ([9.2.15](../009-tools/902-aot-pipeline.md#9215-output)). Things that share a section are kept or dropped together and never folded.
- Only relocation kinds the linker resolves may appear: those `nabic`'s code generator writes, and those in the runtime library and the C libraries the toolchain ships. Any other is a link error naming it (`unsupported relocation ..`).
- Routines the instruction set lacks are calls to the platform's support routines, which the runtime library provides: on armv7 `__aeabi_idiv`, `__aeabi_ldivmod`, `__aeabi_l2d`, `__mulodi4`, ..; on riscv32 the soft-float routines and `__divdi3`, .. ([8.7.4](807-code-generation.md#874-operations)).
- Floating-point methods other than `sqrt` are the `nabi_f64_*` / `nabi_f32_*` calls the input already has; a code generator does not fold them at compile time.
- Output is deterministic: the same input gives the same bytes.

**Debug information** is optional. When the build has `-g`, a code generator should describe its functions as [9.9.6](../009-tools/909-debug-tools.md#996-debugger-integration-gdb--lldb) does: functions named by their Nabi paths ([9.5.10](../009-tools/905-symbol-mangling.md#9510-debug-information)), line tables from the statements' positions, and parameters and named locals with the types the input describes. The linker reads DWARF 2 to 4 in 32-bit format (a `.dSYM` bundle skips an object with DWARF 5 or 64-bit DWARF), and on COFF CodeView C13 in `.debug$S` / `.debug$T`, which it merges into the PDB. Debug information must not change the code.

## 8.11.13. Separately installed code generators

> [!NOTE]
> Draft for 0.2. Nothing in this subsection is implemented.

A code generator other than `nabic`'s own is a separate executable, a toolchain component installed with `nabi component add <component>`: `codegen-llvm` installs `nabic-codegen-llvm`, which contains its own statically linked copy of a fixed LLVM version. `nabic` itself contains no other code generator and links no external code generator ([9.2.17](../009-tools/902-aot-pipeline.md#9217-separation-of-compiler-and-runtime)).

**Selection.** `-C codegen-backend=<name>` (`native`, the default, or a component's name such as `llvm`), or `codegen_backend: "<name>"` in a profile of `nabi.nabion` ([9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)), which the option overrides. The choice applies to every unit the build compiles; prebuilt libraries keep the objects they have, and the shared unit, REPL cells and Jupyter cells are always `native`. A name whose component is not installed is an error that names the command installing it: ``codegen backend `llvm` is not installed; run `nabi component add codegen-llvm` ``. A component compiles for every target `nabic` does, or says which it does not (`E0129`).

**Kernel components.** A component may compile only some functions: `codegen-mlir` compiles the functions a library marks for it, and the rest of the unit stays with the selected code generator. `nabic` puts such functions into units of their own. How a function is marked is not specified yet.

**Finding it.** `nabic` runs `nabic-codegen-<name>` from its own `bin/` directory (the toolchain's, [9.2.19](../009-tools/902-aot-pipeline.md#9219-installation-layout)); it does not search `PATH`. A component is installed per toolchain and its version is the toolchain's.

**Invocation.** One process compiles one unit:

```
nabic-codegen-<name> <unit>.nabimir -o <unit>.o
```

- `<unit>.nabimir` is the unit's nabi-mir in the format of [9.11](../009-tools/911-mir-format.md). The options of the build are in it; there are no other arguments.
- `-o` names the object to write. `nabic` reads it only after exit status 0; on any other status it ignores whatever was written.
- `nabic` runs as many processes at once as it has cores, and keeps the input of a failed unit with `--keep` ([9.1.2](../009-tools/901-nabic-cli.md#nabic-build-aot-build)).
- The working directory is the build's; the environment is `nabic`'s.

**Diagnostics.** The process writes its diagnostics to standard output, one JSON object per line, in the format of `--message-format=json` (schema 1, [9.9.4](../009-tools/909-debug-tools.md#json-format-tool-integration)). Their spans are positions of the input's statements. `nabic` prints them as its own, in the format the build asks for. A construct it cannot compile is an error `E0129` naming it, at the statement that uses it; there is no silent fallback to another code generator. Standard error is passed through unchanged.

**Exit status.**

| Status | Meaning |
|--------|---------|
| `0` | The object was written |
| `1` | Errors were reported on standard output |
| `2` | Usage error, or an input format version the component does not read (it names both versions) |
| `70` | Internal error of the component |

**Version.** `nabic-codegen-<name> --version` prints `nabic-codegen-<name> <version>` as its first line and a line `mir: <major>.<minor>` naming the input format version it reads; `nabic` refuses a component of another toolchain version with an error that names the command reinstalling it. The identity of the component (its version line and its file's stamp, as for `nabic` in [9.4.4](../009-tools/904-nabihir-bytecode.md#944-header-and-producer)) is part of every unit's key ([9.2.16](../009-tools/902-aot-pipeline.md#codegen-units)), so a changed component recompiles the units it compiled.

## 8.11.14. Verification

A code generator is checked by the end-to-end tests of [8.7.9](807-code-generation.md#879-verifying-the-translation) with its objects in the program, and with the GC stress settings (`NABIR_GC_STRESS`, `NABIR_GC_EVAC=always`, [9.9.8](../009-tools/909-debug-tools.md#998-memory--gc-tools)), under which every allocation may collect and every full collection moves every object it may. A program that links objects of two code generators, each unit from either, behaves as a program built by one.
