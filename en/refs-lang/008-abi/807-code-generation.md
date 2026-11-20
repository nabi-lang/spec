# 8.7. Code generation

`nabic` generates machine code itself. Its code generator takes nabi-mir and writes an object file for the target, and its own linker makes the executable ([9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables)). The same code generator serves every optimization level and REPL cells. The lowering rules of 8.8 through 8.10 describe what it emits; this section gives the stages around them.

## 8.7.1. Translation flow

```
nabi source
   | parse, name resolution, type check
nabi-hir
   | specialize / monomorphize, desugar
nabi-mir
   | optimization (-O1 and above, 8.7.3)
nabi-mir
   | lowering: layouts, conventions, GC frames, barriers, checks (8.8 ~ 8.10)
LIR (one low-level form for every target)
   | instruction selection, register allocation (above -O0)
machine code
   | object writer
object (Mach-O, ELF, COFF, wasm)
   | nabic's linker
executable
```

All steps run inside `nabic`. The code examples of 8.8 through 8.10 are pseudo-code: blocks and branches as nabi-mir has them (`bb1:`, `if c then bb2 else bb3`, `goto bb4(x)`), and loads and stores at byte offsets where the emitted code matters. They are not a format `nabic` reads.

## 8.7.2. Responsibility of each stage

| Stage | What it decides |
|-------|-----------------|
| nabi-mir → optimized nabi-mir | Inlining and simplification above `-O0` (8.7.3) |
| nabi-mir → LIR | Everything that does not depend on the instruction set: the layout of every value (8.8), the calling convention ([8.6](806-calling-convention.md)), shadow stack frames, allocation fast paths, write barriers, safepoint polls (8.9), overflow and bounds checks and their panic messages |
| LIR → machine code | The instructions of one instruction set (`x86_64`, `aarch64`, i686, armv7, riscv64, riscv32, powerpc64le, s390x, wasm32); above `-O0`, which values live in registers |
| machine code → object | The container (Mach-O, ELF, COFF, wasm object) with its relocations, unwind information and, with `-g`, debug information ([9.9.6](../009-tools/909-debug-tools.md#996-debugger-integration-gdb--lldb)) |
| objects → executable | `nabic`'s linker; `NABI_CC` overrides it ([9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables)) |

## 8.7.3. Optimization levels

| Level | nabi-mir | Values |
|-------|----------|--------|
| `-O0` | as built | each in its stack slot |
| `-O1` | optimized; calls in loops inlined as at `-O2`, elsewhere only small and `#[inline]` functions and functions called from one place | in registers where they fit |
| `-O2` | optimized; larger bodies inlined outside loops too | in registers where they fit |
| `-O3` | as `-O2`, larger bodies inlined in loops, loops left scalar unrolled | in registers where they fit |
| `-Os` | optimized; a call is inlined only where the body is about as small as the call (a few times that in a loop) | in registers where they fit |

The optimizer inlines direct calls to Nabi functions whose bodies the build has: the package's own, generic instances, and the standard library's, which every build compiles with the program. A call cycle is never inlined into itself, and `#[inline(never)]` and `#[cold]` functions and calls on a path that ends in a panic stay calls. The calls of a body inlined into a loop are weighed again as calls in that loop. A call passing an array of constants (a `const` table) to a parameter its callee only reads goes to a copy of the callee that builds the array itself, so the call copies no table. The optimizer then propagates copies and constants, folds constant expressions, removes common subexpressions and dead code, simplifies the control flow, takes a jump straight past a branch whose side it already knows on the way in (the `Option` an inlined iterator's `next()` returns, tested by the `for` loop around it), turns self tail calls into loops, removes overflow checks whose result always fits, and moves loop-invariant computations out of loops. At `-O3` it also unrolls the strips of the loops the vectorizer leaves scalar (8.7.8; a loop keeping a float sum, or calling a function): a whole strip runs the body 4 times (2 for a longer body) between tests of its counter. Every value the program computes, every trap (overflow, division, bounds) and every side effect stays, in the same order.

`-g` does not change the generated code at any level. Above `-O0` the optimizer records for the debug information what it did: each body it inlines is a scope of its own under the call, and a named local whose assignments it removes keeps what holds its value instead (a constant, another local whose value does not change while the variable is visible, or the fields of a struct built in place). The code generator then says where each variable is over the code ([9.9.6](../009-tools/909-debug-tools.md#996-debugger-integration-gdb--lldb)). In the copies of a loop body `-O3` unrolled, a variable kept as another local's value may show an earlier iteration's value.

Instruction selection is the same at every level, but above `-O0` loops may be vectorized (8.7.8), and RISC-V code takes the 16-bit compressed (`C`) form of every instruction that has one and no relocation patches, branches and jumps where theirs reaches the target. Shadow stack frames, polls and barriers are placed after optimization, on the code that is compiled (8.9). The code is for the target's baseline CPU, but for the features `-C target-cpu` and `-C target-feature` turn on that the code generator uses ([0.6.4](../000-meta/006-platform-targets.md#064-simd)): AVX2 vector loops and one-instruction bit counts on `x86_64`, LSE atomics, RCpc acquire loads and byte dot products on `aarch64`, vector loops with `vector` on `s390x` and with `v` on `riscv64`, one-instruction rotates with `zbb` on `riscv64` and `riscv32`, a fused `mul_add` with `fma` on `x86_64` and with `vfp4` on `armv7`; `-neon`, `-vsx` and `-simd128` leave every loop scalar.

## 8.7.4. Operations

- Integer and floating-point arithmetic, comparisons, `sqrt`, the bit counts (`count_ones`, `leading_zeros`, `trailing_zeros`), `swap_bytes` and atomic operations are instructions (an atomic read-modify-write on `aarch64` without LSE: an exclusive load / store loop). Above `-O0` an atomic operation is as strong as the ordering the program asks for and no stronger, by the usual C++ mappings: on `x86_64` and `i686` only a `SeqCst` store fences; on `aarch64` a `Relaxed` load or store is a plain one, an acquire load `ldar` (`ldapr` with `rcpc`), a release store `stlr`, and read-modify-writes take the acquire / release forms of the LSE or exclusive instructions; on `armv7` a `dmb` comes before a release and after an acquire; on `powerpc64le` `lwsync` before a release (`sync` for `SeqCst`) and `lwsync` or `isync` after an acquire; on RISC-V `fence` and the `aq` / `rl` bits; on `s390x` only a `SeqCst` store serializes; `wasm32` atomics are sequentially consistent. At `-O0` every atomic operation is sequentially consistent, which satisfies every ordering ([0.6.4](../000-meta/006-platform-targets.md#064-simd)).
- Where the instruction set has no instruction for an operation, the code calls the platform's support routine for it. On armv7 that is integer division of every width (`__aeabi_idiv`, `__aeabi_uidiv`, `__aeabi_ldivmod`, `__aeabi_uldivmod`), conversion between 64-bit integers and floating point (`__aeabi_l2d`, `__aeabi_d2lz`, ..) and the overflow check of a 64-bit multiplication (`__mulodi4`). On riscv32, which is soft-float, it is all floating-point arithmetic, comparison and conversion (`__adddf3`, `__ltdf2`, `__floatdidf`, ..) and 64-bit division (`__divdi3`, `__umoddi3`, ..). i686 divides 64-bit integers in a routine inside the object.
- Above `-O0`, `mul_add` is one fused multiply-add instruction where the target has one (`aarch64`, `powerpc64le`, `s390x`, `riscv64`; `x86_64` with `fma`, `armv7` with `vfp4`), and elsewhere a call of the runtime's `nabi_f64_mul_add` / `nabi_f32_mul_add`; both round once.
- The other float methods (`floor`, `round`, `sin`, `exp`, `pow`, ..) are calls to the runtime's `nabi_f64_*` / `nabi_f32_*` functions, which call libm ([8.5](805-extern-symbols.md)): their results come from the target's libm, never from a compile-time evaluation on the host.
- Every branch of nabi-mir is a jump or a two-way branch. A `match` on a tag or an integer is a chain of comparisons in arm order (8.9.3); no jump table is built.

## 8.7.5. Acceleration-domain libraries

Nabi source has no direct access to the code generator. A backend interface through which libraries in acceleration domains such as tensors / automatic differentiation / SIMD / GPU reach a code generator is planned and not implemented: in 0.1 `nabic`'s own code generator is the only one, and the separately installed code generators of [8.11.13](811-backend-contract.md#81113-separately-installed-code-generators) are a draft for 0.2. What a code generator must guarantee of its objects is [8.11](811-backend-contract.md).

## 8.7.6. `extern` functions (runtime functions)

The `nabi_*` functions listed in [Extern "C" symbols](805-extern-symbols.md) are called by symbol with the C ABI ([8.6](806-calling-convention.md)) and resolved at link time against `libnabir.a`. String equality, and so a string pattern, calls `nabi_str_cmp`. The TLAB fast path, the write barrier's fast path and the safepoint poll are inline code; they call `nabi_gc_alloc_slow`, `nabi_gc_wb_slow` and `nabi_runtime_yield` only on their slow paths (8.9.9 ~ 8.9.11).

## 8.7.7. TLS access

Access to the TLS slots of [Mutator TLS slots](803-tls-slots.md) goes through the address of the thread-local block `nabi_tls`. A Nabi function receives it from its caller as its hidden last argument ([8.6.6](806-calling-convention.md#866-nabi-to-nabi-calling-convention)); code entered from outside Nabi computes it once, at entry, with the target's TLS access sequence ([8.3.8](803-tls-slots.md#838-form-of-compiler-emission)). Each slot is then a load or a store at its offset from that address.

```
t = <nabi_tls address>            ; the hidden last argument, or computed once at entry
flag = load.u8 [t + 10p+8]        ; yield_flag: offset 88 when p = 8
```

## 8.7.8. Loop vectorization

Above `-O0` on `x86_64` (SSE2, or AVX2 with the `avx2` feature), `i686` (SSE2), `aarch64` (NEON), `armv7` (NEON, integer lanes only: NEON flushes subnormal floats to zero), `powerpc64le` (VMX and VSX), `wasm32` with SIMD128, `s390x` with `vector` (z13; float lanes of 4 bytes with `vector-enhancements-1`, z14) and `riscv64` with `v` (128 bits of each vector register, at any vector length) ([0.6.4](../000-meta/006-platform-targets.md#064-simd)), a loop that steps a `usize` counter by one up to a bound it does not change, and that polls nowhere (a strip of a counted loop, or a short loop: [8.4.2](804-safepoint.md#842-emission-sites)), may get a vector loop in front of it (at `-Os` only a body of at most 24 nabi-mir statements, whose vector loop is about as long as the scalar one). A `for` loop over a `List`, a slice or a string's bytes (`for x in xs`, `for x in xs[a:]`, `for (i, x) in xs.iter().enumerate()`, `for b in s.bytes()`) is such a loop once its iterator's `next()` is inlined, as a range loop is, and so is a loop whose bounds checks the optimizer removed. The vector loop works on vectors of 16 bytes (32 with AVX2) holding elements of one width and leaves the iterations it does not do to the scalar loop after it. A loop gets one when its body only:

- reads and writes elements of buffers (a `List`'s) at the counter, at the counter plus a small constant, or at the counter plus a value the loop does not change (a slice's start);
- reads elements narrower than the values it computes only to widen them (`b as u32` of a `u8`), and writes narrower elements only as values narrowed for them (`x as u8`), on every target but `powerpc64le`;
- computes on those elements and on values the loop does not change: integer `+`, `-` and `*` (wrapping, or checked; `*` on the widths the target multiplies, a checked one where it also gives the high half of each product: 2-byte lanes on `x86_64` and `i686`, 1 to 4 bytes on `aarch64` and `s390x`, 2 and 4 on `wasm32`, every width on `riscv64`), `&`, `|`, `^`, `!`, shifts by a constant, comparisons, `as` between integers and from `bool`, float `+`, `-`, `*`, `/`, negation, `sqrt`, `min` and `max` (not on `armv7`), and `mul_add` where both the target and its vector unit fuse it (`aarch64`, `powerpc64le`, `riscv64`, `x86_64` with `fma`, `s390x` on 8-byte lanes or with `vector-enhancements-1`);
- branches on such values (`if`, `&&`, `||`): each lane computes both sides and keeps what its test picks. A side may read the elements every lane may read (those a block every iteration runs reads, or those under a bounds check the loop keeps to). Both sides of a test may store to one element: the stores are merged. A store on one side only needs a store under a mask (AVX2, elements of 4 and 8 bytes);
- leaves the loop early (`return`, `break`, a bounds check that fails);
- accumulates an integer reduction: `+` or `-` (wrapping or checked), `&`, `|`, `^`, a count (`if c { n += 1 }`, `n += c as T`), a minimum or maximum (`m = m.min(x)`, `if x > m { m = x }`), on one side of a test or on several, or a sum of products of bytes (`s += (a[i] as i32) * (b[i] as i32)`, four products to a lane on `aarch64` with `dotprod`); or a float minimum or maximum, the same two ways.

What a vectorized loop does is what the scalar loop does, element after element:

- A vector holding an element that would leave the loop or trap (an overflow, a bounds check that fails) is not run: the scalar loop takes it from its first element, and leaves or panics at that element, every element before it done.
- A reduction keeps a partial result per lane and combines the lanes at the end. A checked sum keeps its non-negative and its negative elements apart, which bound every partial sum the scalar loop would make (with one update on any way through an iteration); when either bound does not fit, the loop runs again as scalar code from its start (such a loop writes no memory), so it panics exactly where the scalar loop does, or not at all. A minimum or maximum starts every lane from the accumulator; of two equal integers either may be kept, which does not show. A checked sum of products of bytes keeps the sum of the products' magnitudes too, which bounds every partial sum; when that bound does not fit, the loop runs again as scalar code.
- Float sums are not vectorized: a float sum adds in the program's order, and another order rounds differently. A float minimum or maximum gives the scalar loop's bits for every input. Of numbers neither a NaN nor a zero, the least (greatest) is one value in every order, so the lanes may find it: a vector holding a NaN or a zero (whose sign the scalar `min` / `max` keeps by operand order on some targets) is not run, and the scalar loop takes it and the rest; an accumulator that is a NaN when the loop starts leaves the whole loop scalar. A map `y = a.min(b)` likewise leaves to the scalar loop a vector where a NaN or two zeros meet. A multiplication and an addition are fused into one instruction only where the program calls `mul_add`.
- No element is written that the scalar loop does not write: a store on one side of a test goes out under a mask, or the loop stays scalar.
- Elements of two lists that may be one list keep the order of their reads and writes: where vectors would change it, the vector loop runs only when a test in front of it finds two different buffers. Two rows of one buffer at offsets the loop does not change (`r[o1 + i]` and `r[o2 + i]`) likewise: the vector loop runs only when a test in front of it finds the offsets equal or a whole vector apart. An index through a slice's start (an `isize` add, checked) is exact below `isize::MAX`: a test in front checks that the start and the bound are far below it, else the scalar loop runs.
- A vector loop reaches no safepoint and takes no poll; it runs 1024 iterations at most before the scalar loop, with its poll, runs again.

With `-g` the instructions of a vector loop are at the position of the loop's first statement; while it runs, the loop's variables keep the values they had when it started.

## 8.7.9. Verifying the translation

End-to-end tests (`compiler/nabic/tests/main/integration.rs`, over the programs in `tests/cases/`) build and run .nabi files, at `-O0` and, with `NABI_TEST_OPT=-O2`, optimized; `NABI_TEST_TARGET` runs them for another target.

This verification ensures that the lowering rules defined in this block are correct.
