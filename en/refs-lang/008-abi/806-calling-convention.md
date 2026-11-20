# 8.6. Calling convention

This document defines the register and stack conventions used for function calls. It follows the platform C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) by default and states only the parts that differ.

## 8.6.1. Default: platform C ABI

Every function declared as an [Extern "C" symbol](805-extern-symbols.md) uses the platform's C ABI. Calls between Nabi functions use the same registers, but pass aggregates differently (8.6.6).

| Platform                 | ABI                |
|--------------------------|--------------------|
| Linux x86_64             | System V AMD64     |
| Linux aarch64            | AAPCS64            |
| Linux i686               | System V i386      |
| Linux armv7              | AAPCS              |
| macOS aarch64            | AAPCS64 (Apple variant) |
| Windows x86_64           | MS x64             |
| Windows aarch64          | AAPCS64 (MS variant) |
| Windows i686             | MS x86 (cdecl)     |
| FreeBSD x86_64           | System V AMD64     |
| Linux riscv64            | RISC-V LP64D       |
| riscv32 (no OS)          | RISC-V ILP32 (soft float) |
| Linux powerpc64le        | 64-bit ELFv2       |
| Linux s390x              | s390x ELF (z/Architecture) |
| WASM                     | WebAssembly C ABI  |

`nabic` classifies how aggregates (struct / tuple / array) are passed across the C boundary by the same rules as the platform's C compilers (8.6.3), then places every scalar in its register or stack slot. This specification defines only the remaining variations and additions. Under the WebAssembly C ABI an aggregate with a single scalar is passed as that scalar, and any other aggregate by pointer to a caller copy (results through `sret`); runtime `str` arguments are passed by reference.

## 8.6.2. Register arguments / return

The default argument and return registers defined by each ABI are used as is.

| Platform       | Argument registers (integer) | Return (integer) |
|----------------|----------------------------|-------------|
| Linux x86_64   | rdi, rsi, rdx, rcx, r8, r9 | rax, rdx    |
| Windows x86_64 | rcx, rdx, r8, r9           | rax         |
| aarch64        | x0 ~ x7                    | x0, x1      |
| i686           | (stack only)               | eax, edx    |
| armv7          | r0 ~ r3                    | r0, r1      |
| riscv64        | a0 ~ a7                    | a0, a1      |
| riscv32        | a0 ~ a7                    | a0, a1      |
| powerpc64le    | r3 ~ r10                   | r3, r4      |
| s390x          | r2 ~ r6                    | r2          |

Floating-point and vector values likewise use the standard registers defined by each ABI (xmm0-7, vN, sN, etc.).

## 8.6.3. `struct` arguments / return: small `struct` rule

When an aggregate is passed by value across the C boundary (`extern "C"` declarations and definitions), nabic classifies it according to the target ABI by the same rules as the platform's C compilers and passes it in the registers that classification gives. An HFA fits in up to four float / double registers (eight on powerpc64le). In the table, `iN` / `[n x T]` name the integer or floating-point registers (or stack words) an aggregate is passed in, `byval` a copy on the stack, and `sret` a result written through a hidden pointer the caller passes.

| Target                                    | Classification                                                                            |
|-------------------------------------------|-------------------------------------------------------------------------------------------|
| AAPCS64 (Linux / Apple / Windows arm64)   | An HFA (1 ~ 4 elements of the same floating-point type) becomes `[n x f32]` / `[n x f64]`. Other aggregates of 16 bytes or less become `i64` / `[2 x i64]` (return values of 8 bytes or less become `iN`). Larger ones: arguments are passed as a pointer to a caller copy, returns use `sret` |
| System V x86_64                           | Each eightbyte is classified INTEGER / SSE and placed in integer / SSE registers. If registers run out, the whole aggregate is passed `byval`. Above 16 bytes: arguments `byval`, returns `sret` |
| Windows x64                               | Aggregates of size 1 / 2 / 4 / 8 bytes become `iN`. Others: arguments by reference (pointer to a caller copy), returns `sret` |
| System V i386 / Windows x86 (i686)       | Aggregate arguments are passed `byval` on the stack (4-byte aligned). Results: on Windows an aggregate of 1 / 2 / 4 / 8 bytes becomes `iN`; any other aggregate result (every one on System V) uses `sret` |
| AAPCS hard-float (armv7)                  | An HFA becomes `[n x f32]` / `[n x f64]`. Other aggregates of 64 bytes or less become `[n x i32]` (alignment 4 or less) or `[n x i64]`; larger ones are passed `byval`. Results: an HFA in floating-point registers, an aggregate of 4 bytes or less as `i8` / `i16` / `i32`, others `sret` |
| WebAssembly (wasm32)                      | An aggregate with a single scalar becomes that scalar. Any other aggregate argument is passed as a pointer to a caller copy, and a result uses `sret` |
| RISC-V LP64D (riscv64)                    | A struct (no union) of one float, two floats, or a float and an integer of 8 bytes or less, counted with nested structs and arrays flattened, goes in floating-point / integer registers as its fields while enough of both kinds are free; padded and `#[repr(packed)]` structs qualify too, each field read from its own offset. Otherwise an aggregate of 8 bytes or less becomes `i64`, of 16 bytes or less `[2 x i64]` (`i128` if 16-byte aligned), and a larger argument is passed as a pointer to a caller copy. Results follow the same rule with two registers of each kind; larger ones use `sret` |
| RISC-V ILP32 (riscv32)                    | Soft float: no floating-point registers, so a struct of floats follows the integer rule. An aggregate of 4 bytes or less becomes `i32`, of 8 bytes or less `[2 x i32]` (`i64` if 8-byte aligned), and a larger argument is passed as a pointer to a caller copy. Results of 8 bytes or less come back in `a0` / `a1`; larger ones use `sret` |
| ELFv2 (powerpc64le)                       | An HFA (1 ~ 8 elements of the same floating-point type) becomes `[n x f32]` / `[n x f64]`. Other aggregates of 8 bytes or less become `i64`, larger ones `[n x i64]` through `r3`-`r10` and the parameter save area. Results: an HFA in `f1`-`f8`, an aggregate of 8 bytes or less as `i64`, of 16 bytes or less as `[2 x i64]` in `r3` / `r4`, others `sret` |
| s390x                                     | An aggregate of 1 / 2 / 4 / 8 bytes becomes that integer (a lone `f32` / `f64` field stays a float). Other aggregate arguments are passed as a pointer to a caller copy. Aggregate results always use `sret` |
| Other targets                             | Passing an aggregate value across the C boundary is a compile error                       |

- A `union` is classified by its members, overlaid at its start, as C classifies it, not by its storage bytes: it is an HFA when every member is one of the same floating-point type and the members fill it (AAPCS64, armv7, powerpc64le), and System V x86_64 merges the members' classes per eightbyte (`union { f64, f32 }` is SSE). RISC-V never passes a union as a float struct, s390x passes one as an integer of its size, and wasm32 passes one as a scalar only when it has a single member.
- `f16` / `bf16` are C's `_Float16` / `__bf16`. A scalar goes in the low 16 bits of the floating-point register an `f32` would take (`h`/`v` on AAPCS64, `xmm` on x86_64, `s` on armv7, `fa` on riscv64 NaN-boxed, `f` on s390x), and a half is a floating-point member of an aggregate: AAPCS64 has HFAs of halves, System V counts them as SSE, riscv64 as float fields. armv7 passes a struct of halves by the integer rule; i686 passes a half on the stack, and riscv32 in an integer register. A half result on i686 (C returns it in `xmm0`), any half on powerpc64le and wasm32, and `bf16` on s390x, where C has no such type, are compile errors that say to pass the bits (`u16`, `x.to_bits()`, `f16::from_bits`). A half as a C variadic argument is an error too (`x.to_f64()`).
- `Simd<T, N>` of 8 or 16 bytes (2 lanes at least) is C's vector of that size (`float32x4_t`, `__m128`, `__attribute__((vector_size(16)))`). On AAPCS64 it takes one `v` register as a parameter and a result (16 bytes on the stack, 16-aligned, once they run out). On System V x86_64 it takes one `xmm` register (16-aligned stack bytes once they run out) and comes back in `xmm0`; on Windows x64 a vector argument is passed by reference to a 16-aligned caller copy and a result comes back in `xmm0`. Another size, another target, a `Simd` inside an aggregate passed by value and a `Simd` variadic argument are compile errors; `Mask<T, N>` has no C type.
- Alignment and target details, as the C compilers lay them out: on AAPCS64 a 16-byte aggregate that is 16-aligned is an `i128` in an even register pair (on Linux when its fields are 16-aligned, its own `#[repr(align(16))]` not counting; on Windows by its alignment; never on Apple), and a Windows variadic callee takes its fixed homogeneous float aggregates in integer registers (by reference past 16 bytes). On armv7 the even-pair and `byval` alignment comes from the fields too, and a variadic callee returns a homogeneous float aggregate in memory. On powerpc64le a 16-aligned aggregate starts at an even doubleword. On s390x a struct is a lone float through nested structs, never through an array. On System V x86_64 an over-aligned aggregate whose high eightbyte is padding takes one register. On i686 Windows an aggregate with a `#[repr(align(N))]` past 4 cannot be passed by value to a cdecl function (MSVC passes it by reference to a 16-aligned copy, which an i686 frame there is not), an error that says to pass a pointer; a stdcall function takes it `byval`, and its `@N` leaves the hidden result pointer out.
- `signext` / `zeroext` on narrow integer parameters and returns (`i8` / `i16` / `u8` / `u16` / `bool` / `c_char`) follow the platform C ABI: x86_64 SysV, Apple arm64, i686, armv7, wasm32 and RISC-V extend `i8` / `i16` / `bool`; riscv64 also sign-extends 32-bit values (`i32` / `u32` / `char`); armv7 and RISC-V zero-extend `c_char`; Linux AAPCS64 and Windows arm64 extend none; Windows x64 extends only `bool`.
- Functions carry a stack probe, so a large frame touches the guard page instead of skipping it: an inline probe, or a `__chkstk` call on Windows. wasm32 and riscv32 targets have no probe.
- The same classification applies to `extern "C" fn` **definitions**. A definition has a separate C-ABI entry point called from C; calls from Nabi keep using the Nabi convention.
- On System V i386, the in-aggregate alignment of `i64` / `u64` / `f64` is 4 bytes (same as C).
- This classification does not apply to calls between Nabi functions (8.6.6). It is needed only where Nabi meets C, including runtime functions taking or returning `str` (a 16-byte struct; by reference on Windows x64, [8.5](805-extern-symbols.md)).

## 8.6.4. GC object pointers: as ordinary pointers

GC object pointers are passed with the same ABI as the platform's ordinary pointers. No special registers or tagged pointers are used.

They must, however, be registered in the [shadow stack](../000-meta/007-glossary.md#shadow-stack) of [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md) before and after the call.

This registration is done by the link / unlink of the shadow stack frame emitted by the compiler ([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)).

## 8.6.5. Return of throwing functions: value-based error propagation

A function with a [`throws` signature](../004-functions/414-throws.md) returns its error as part of the return value. No separate unwinding mechanism or runtime symbol is used.

```
fn f(args) throws (E) -> T
```

The ABI return type of this function is a `struct` of two fields:

```
Throws<T, E> = (bool, union(T, E))
; field 0: is_err (0 on success, 1 on error)
; field 1: payload. Read as T if is_err is 0, as E if 1
```

The payload is a union with the size and alignment of the larger of `T` and `E`. If `T` is unit, the payload holds only `E`. The error channel is distinguished by the signature, so `T` and `E` may be the same type. For untyped `throws` (`throws (any Error)`), `E` is an `any Error` value in the same tag + payload representation as [`any`](../002-types/202-any-slot.md) ([8.8](808-lowering-types.md)).

Field 0 may be stored wider than one byte (an `i32` when the payload holds GC references, so the collector reads it as a sum tag); only bit 0 is significant, and when the payload is at least 4-aligned the offsets, size and registers are those of `(bool, payload)`. Values of `ref` parameters ([4.7](../004-functions/407-ref-keyword.md)) of a throwing function travel beside the carrier as an implementation-defined part of the return value.

This `struct` is passed through the platform C ABI like any other return value: in registers within the limits of 8.6.3, via sret beyond them. A `try` at the call site checks field 0 and branches; the detailed lowering is defined in [8.9.6](809-lowering-control.md#896-try--catch--finally-value-based-branching).

This scheme was chosen for the following reasons.

- It works the same on every platform (including WASM; no DWARF or platform EH infrastructure is needed).
- It does not conflict with the [shadow stack](../000-meta/007-glossary.md#shadow-stack). Errors also leave frames through the ordinary return path, so each frame pops its shadow stack normally.
- The cost on the no-error path is a single branch after return; entering `try` or `catch` costs nothing.

## 8.6.6. nabi-to-nabi calling convention

Calls between Nabi functions pass scalars in the registers of 8.6.2. An aggregate (struct, tuple, array, `str`) over 64 bytes is passed by address and returned through a leading result pointer. The callee copies a by-address argument on entry, so the caller's value is not modified. The callee may write the result through the result pointer at any point before it returns, after copying its by-address arguments. The result pointer may therefore point at a by-address argument, but at no other memory the callee reads during the call. This holds for every nabi-to-nabi call, including calls into prebuilt code, exported functions and function values. A direct call also passes the caller's `nabi_tls` block address ([8.3.8](803-tls-slots.md#838-form-of-compiler-emission)) as a hidden last pointer argument, after the declared ones; a function value, an `#[export_name]` Nabi function and the `any` dispatcher take none (8.6.11). Smaller aggregates are passed and returned leaf by leaf in the target's argument and result registers, as the rules below give. Every build follows them, so code from any build, prebuilt or not, calls any other. It differs from the C ABI classification of 8.6.3. The rules below apply to aggregates of 64 bytes or less.

> [!NOTE]
> A value passed by address is moved in memory (`memcpy`), never in registers.

**Arguments**

- An aggregate is split into its scalar leaves, each taking the next register of its class; when registers run out, the remaining leaves go on the stack. A `str` may therefore be split between the last argument register and the stack.
- AArch64: an array of `i64` / `ptr` / `f32` / `f64` takes consecutive registers of its class. If it does not fit in the remaining registers, it goes on the stack whole and the rest of that register class is marked used. Arrays of `i8` / `i16` / `i32` are split like structs.
- armv7: a homogeneous float aggregate of 1 to 4 members takes consecutive VFP registers and an integer array consecutive core registers, as AAPCS places them; an integer block may be split between `r3` and the stack while nothing else is on the stack.
- Stack slots: on Apple targets, top-level scalar arguments take their natural size and alignment, and `i1` / `i8` / `i16` leaves of an aggregate take 4-byte slots. On i686, armv7 and riscv32 a slot is 4 bytes and an 8-byte scalar takes two (8-byte aligned on armv7 and riscv32). On the other targets every stack slot is 8 bytes.

**Results**

- AArch64: up to 8 integer (`x0`-`x7`) and 8 floating-point (`v0`-`v7`) registers. A larger result is returned through an sret pointer passed in `x8` (not returned).
- x86-64: up to three integer registers (`rax`, `rdx`, `rcx`); floating-point results use `xmm0` / `xmm1`, then x87 `st0` / `st1`. A result needing a 4th integer or 5th floating-point register is returned through an sret pointer passed in `rdi` and returned in `rax`.
- i686: up to three integer words (`eax`, `edx`, `ecx`) and one floating-point value in `st0`. armv7: `r0`-`r3` and the VFP registers. riscv64 / riscv32: two XLEN-sized values (`a0` `a1`, and on riscv64 `fa0` `fa1`). powerpc64le: `r3`-`r10` and `f1`-`f8`. s390x: `r2`-`r5` and `f0` `f2` `f4` `f6`. wasm32: one value. A larger result is returned through a hidden result pointer passed as the first argument.

**Windows**

- Windows x64: integer and floating-point arguments share four register positions (`rcx` `rdx` `r8` `r9` / `xmm0`-`xmm3`); each argument takes the next position, in the register of its class. Stack slots are 8 bytes, after the caller's 32-byte home area. The sret pointer is passed in `rcx`; results are otherwise as on x86-64.
- Windows arm64: AAPCS64 as above. A C variadic call passes every floating-point argument in integer registers.

**Narrow integers**

A callee must not rely on the upper bits of narrow integer or `bool` arguments and results; a caller may extend them. The compiled code masks an incoming `bool` to bit 0 and re-extends narrow values.

A function no other module can reach (internal, never used as a value) may use an implementation-defined convention.

Calls to `extern "C"` and `extern "nabi"` runtime functions use the C ABI rules of 8.6.1-8.6.5.

## 8.6.7. Tail calls

Above `-O0`, a function's call to itself in tail position becomes a jump to its start ([8.7.3](807-code-generation.md#873-optimization-levels)): recursion that ends in its own call runs in one frame. Other calls in tail position are ordinary calls. Coroutine bodies are state machines and are not affected.

## 8.6.8. ABI of inline functions

The effect of `#[inline]` ([built-in attributes](../006-meta-system/602-built-in-attributes.md)) is that the call itself may disappear. In that case the calling convention has no meaning.

`#[inline(never)]` functions are always called and follow the ordinary ABI.

## 8.6.9. Evaluation order of argument passing

The left-to-right evaluation order of [parameters](../004-functions/402-parameters.md) is orthogonal to the calling convention.

Left-to-right evaluation means each argument expression is evaluated in order starting from the left. Which register or stack slot receives each result is a separate question decided by the ABI.

Some C and C++ ABIs favor right-to-left evaluation because of stack push order. Nabi, however, guarantees left-to-right semantics, and the compiler reconciles the two requirements using temporaries and registers as needed.

## 8.6.10. Variadic functions

The `...T` variadic of [parameters](../004-functions/402-parameters.md) is a language-level variadic that bundles the arguments into a single slice view (`Slice<T>`) (usually the same ABI as a struct argument). It is a separate concept from the platform's C variadic (`...` variable arguments).

C variadic functions (e.g. `printf`) can be called directly through `extern "C"`, in which case the ABI follows the platform's C variadic convention.

```
extern "C" {
    fn printf(fmt: ConstPtr<c_char>, ...);
}
```

- `...` may appear only in declarations in an `extern` block, and only as the last parameter. Nabi function definitions cannot have C variadics.
- C default argument promotion applies to arguments in variadic positions: `bool`, `i8`, `i16`, `u8`, `u16` and `c_char` are passed as `c_int` (`i32`), `f32` as `f64`, and `str` as `char*` (a NUL-terminated byte pointer).
- Aggregates (struct / tuple / array) cannot be passed as variadic arguments (compile error).

## 8.6.11. Function pointer ABI

`fn(...) -> ...` from [Function types](../004-functions/404-function-type.md) uses the same calling convention for indirect calls.

```
code = f.0                       ; the function value's code pointer
result = call code(f.1, ...)     ; indirect call: the environment, then the arguments
```

Arguments and returns have the same ABI, so direct and indirect calls share the same interface, except for the hidden `nabi_tls` address of 8.6.6: a function value takes none. Its code pointer is a thunk that reads the address itself and calls the function with it.

## 8.6.12. Closure calls

A closure as described in [Closures](../004-functions/405-closure.md) is a single unit bundling an environment and a code pointer.

A call works as follows.

```
The environment pointer as the first argument, followed by the ordinary arguments.
```

This environment argument is how the closure body accesses captured variables.

The body is compiled like an ordinary function (the environment is just the first argument).

## 8.6.13. Async function calls

An `async fn` from [Async](../004-functions/412-async.md) returns a Future ptr immediately when called, following the ordinary function call ABI.

```
let f = async_fn(args);     // returns immediately (Future ptr)
let r = await f;             // coroutine mechanism
```

`await` consists of ordinary calls (polling the target future) and returns of the state machine ([8.10](810-lowering-coroutine-generic.md)), and works on top of the ordinary ABI.

## 8.6.14. Stability

The C ABI parts (8.6.1 ~ 8.6.3, 8.6.10) follow the platform C ABI, so they have no separate stability decision: Nabi stays compatible within the range the platform guarantees compatibility.

The nabi-to-nabi convention (8.6.6, 8.6.11) and the return value layout of throwing functions (8.6.5) are part of the runtime ABI; any change follows the stability policy in the [overview](README.md#stability).
