# 9.13. `nabicc` - the C compiler

The toolchain compiles C with its own C compiler. It reads C with the same front end as `nabi bindgen` ([9.10.8](910-nabi-frontend.md#nabi-bindgen-c-header-bindings)), lowers it to nabi-mir, and compiles and links it with the code generator and linker that `nabic` uses ([0.2.4](../000-meta/002-pipeline.md#024-code-generation)). No other C compiler, assembler or linker takes part, and none has to be installed.

The front end also reads Objective-C headers, for `ffi::objc::import` and `nabi bindgen -x objective-c` ([stdlib 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-headers-ffiobjcimport)): declarations only (`@interface`, `@protocol`, methods, properties), with `__OBJC__`, `__OBJC2__` and ARC's features predefined and `#import` read once per file. `nabicc` itself compiles C only; a `.m` file is not a C source.

## 9.13.1. `nabicc` and `nabi cc`

The C compiler has two forms with the same behaviour:

- `nabicc`, a standalone executable. It is installed in the toolchain's `bin/` and as a proxy in `$NABI_HOME/bin/` beside `nabi`, `nabic` and `nabils` ([9.10.6](910-nabi-frontend.md#9106-toolchain)).
- `nabi cc`. `nabi` runs the `nabicc` of its own toolchain with the arguments after `cc` unchanged, as it delegates to `nabic` ([9.10.1](910-nabi-frontend.md#9101-subcommands-and-global-options)). `nabi`'s global options may come before `cc`; every word after it is the C compiler's.

```
nabicc [options] <file>...
nabi cc [options] <file>...
```

The command line is the one gcc and clang take, so either form is a drop-in C compiler for build systems: `CC=nabicc` and `CC="nabi cc"` both work with make, CMake, autoconf and Meson.

```
make CC="nabi cc"
cmake -DCMAKE_C_COMPILER=nabicc -B build
```

Messages name the form that was run: `nabi cc: error: ..` or `nabicc: error: ..`. The exit code is `0` on success and `1` when a source fails to compile, an option is wrong or the link fails. An internal compiler error exits with the code of [9.1.6](901-nabic-cli.md#916-exit-codes).

## 9.13.2. Language

C89, C99, C11, C17 and C23 are read as one dialect: a program written for any of them compiles, and newer syntax (`_Static_assert`, `typeof`, `[[attributes]]`, `bool` as a keyword, binary literals, digit separators) is accepted whatever the standard. `-std=` selects the standard the program sees through its macros:

| `-std=` | `__STDC_VERSION__` |
|---------|--------------------|
| `c89`, `c90`, `gnu89`, `gnu90` (also `-ansi`) | not defined |
| `c99`, `gnu99` | `199901L` |
| `c11`, `gnu11` | `201112L` |
| `c17`, `c18`, `gnu17`, `gnu18` | `201710L` |
| `c23`, `c2x`, `gnu23`, `gnu2x` | `202311L` |

The `iso9899:<year>` spellings are accepted too. The default is `gnu17`. A `c` form (not `gnu`) also defines `__STRICT_ANSI__`. Any other standard is an error that lists these.

The GNU, Clang, Apple and Microsoft extensions that system headers use are read: `__attribute__((..))`, `__declspec(..)`, `__asm__("symbol")` labels, `__extension__`, `typeof`, statement expressions, `__builtin_*` calls (9.13.4), nullability qualifiers, anonymous structs and unions, `#pragma pack`, `#include_next`, `__has_include`, `__has_attribute`, `__has_builtin` and `__has_feature`. Types are laid out by the target's C ABI, as clang lays them out for the target.

The predefined macros are clang's for the target (`__x86_64__`, `__aarch64__`, `__linux__`, `__APPLE__`, `_WIN32`, `__SIZEOF_LONG__`, `__GNUC__` on non-MSVC targets, `_MSC_VER` on MSVC targets, ...), plus `__nabicc__` (`1`). The vector extensions and their feature macros (`__SSE__`, `__SSE2__` .. `__AVX512F__`, `__ARM_NEON`, `__ARM_FEATURE_*`, `__wasm_simd128__`, `__riscv_vector`, `__ALTIVEC__`, ...) are not defined, nor is `__SIZEOF_INT128__`: there are no vector intrinsics yet, so a library that checks these macros takes its scalar path.

## 9.13.3. What it refuses

The constructs below are errors at their `file:line:col`. Each error says what to write instead. In the body of an `inline` or `static` function, one is an error only when the unit calls the function: a header's inline function that no code uses (a C library's SIMD `wmemchr`, a `cpuid` helper) does not stop the build.

| Construct | Write instead |
|-----------|---------------|
| inline assembly (`asm`, `__asm__`), except an empty barrier (`asm("")`, `asm volatile("" ::: "memory")`) and a lone spin hint without operands (`"pause"`, `"yield"`, `"nop"`), which compile to nothing | the code in C, or the macro that selects the library's portable code path |
| a label's address (`&&label`) and computed `goto *p` | a `switch` |
| a variable length array (an array whose size is not a constant), except a parameter, which is a pointer | a fixed size, or `malloc` and `free` |
| `alloca` (`__builtin_alloca`, `_alloca`) | `malloc` and `free`, or a fixed-size array |
| arithmetic on `_Complex` values (`__real__`, `__imag__`, operators) | the real and imaginary parts in two variables |
| `__int128` and `unsigned __int128` | two 64-bit halves |
| `long double` values where `long double` is wider than `double` (80-bit x87 on x86 and x86-64 Linux and FreeBSD; 128-bit on aarch64, riscv64, powerpc64le and s390x Linux, on Android, on riscv32 and on wasm32) | `double` (many libraries take an option such as `-DLONGDOUBLE_TYPE=double`) |
| arithmetic on `_Float16`, `__fp16` and `__bf16` | a conversion to `float` first |
| defining a thread-local variable (`_Thread_local`, `thread_local`, `__thread`, `__declspec(thread)`); an `extern` declaration of one, such as wasi-libc's `errno`, is read and used | the state passed through a parameter, or kept in a struct each thread gets |
| a variadic function definition on `aarch64-pc-windows-msvc` (other targets compile them) | a `va_list` version, `int vf(const char *fmt, va_list ap)`, called from a non-variadic wrapper |
| Windows structured exception handling (`__try`, `__except`, `__finally`, `__leave`) | the library built without it (sqlite: `-DSQLITE_OMIT_SEH`) |
| vector types (`__attribute__((vector_size(N)))`, `__m128`, `float32x4_t`, SVE types) | the library's scalar code path (its `NO_SIMD` / `NO_SSE` option) |
| `__builtin_frame_address`, `__builtin_return_address` and builtins not listed in 9.13.4 | the operation in C, or the C library's function |
| a K&R-style definition (`int f(a, b) int a; ..`) | a prototype: `int f(int a, int b)` |
| an implicit `int` (`static x;`) | the type: `static int x;` |
| a call of an undeclared function | the `#include` of the header that declares it, or its prototype before the call |

`long double` is `double` on Apple targets, on Windows and on 32-bit Arm, where the target's C ABI makes it 64-bit; there it compiles as `double`.

Options and inputs that are refused:

| Option or input | Error |
|-----------------|-------|
| `-shared` | `nabi cc` links executables; build a static library with `ar` (or `nabi build` a package) |
| a `.s`, `.S` or `.asm` source | `nabi cc` does not assemble; write the function in C, or link an object built elsewhere |
| a C++ or Objective-C source (`.cc`, `.cpp`, `.cxx`, `.C`, `.m`, `.mm`), or `-x` with a language other than `c`, `c-header` or `none` | `nabi cc` compiles C only; C++ and Objective-C need their own compiler |

## 9.13.4. Builtins

These builtins compile without a declaration. Each takes the `__builtin_` prefix (`__builtin_expect`); the names in the last rows have their own prefixes.

| Builtin | Meaning |
|---------|---------|
| `va_start`, `va_arg`, `va_end`, `va_copy` | Variable arguments by the target's C ABI, as `<stdarg.h>` uses them. `va_start` in a function without `...` is an error |
| `expect`, `expect_with_probability` | The value; the hint is ignored |
| `assume`, `assume_aligned`, `prefetch` | The value, or nothing; no effect on code |
| `unreachable` | Undefined behaviour if reached |
| `trap`, `debugtrap` (also `__debugbreak`) | Stops the program with a trap instruction |
| `constant_p` | `1` for an integer or floating-point constant, else `0` |
| `object_size`, `dynamic_object_size` | Unknown: `(size_t)-1` for types `0` and `1`, `0` for types `2` and `3` |
| `clz`, `ctz`, `popcount` (each with `l` and `ll`), `bswap16`, `bswap32`, `bswap64` | Bit counts and byte swaps |
| `huge_val`, `inf`, `nan`, `nans` (each with `f` and `l`) | Floating-point constants |
| `isnan`, `isinf`, `isinf_sign`, `isfinite`, `isnormal`, `signbit`, `fpclassify`, `isgreater`, `isgreaterequal`, `isless`, `islessequal`, `islessgreater`, `isunordered` | Classification and comparison, for the `<math.h>` macros |
| `add_overflow`, `sub_overflow`, `mul_overflow` and the typed forms (`sadd_overflow`, `umulll_overflow`, ...) | Arithmetic that stores the wrapped result and returns whether it overflowed |
| memory, string and math functions (`memcpy`, `memset`, `strlen`, `strcmp`, `malloc`, `abs`, `sqrt`, `sin`, `pow`, `fabsf`, ...) | A call of the C library's function of the name without the prefix |
| fortified calls (`__builtin___memcpy_chk`, `__builtin___snprintf_chk`, ...) | A call of the plain function (`memcpy`, `snprintf`): object sizes are unknown (above), so the checks would pass |
| `offsetof` (`__builtin_offsetof`) | The offset of a member, a constant expression |
| `__atomic_*`, `__sync_*`, `__c11_atomic_*`, `_InterlockedIncrement`, `_InterlockedDecrement`, `_InterlockedExchange`, `_InterlockedExchangeAdd`, `_InterlockedCompareExchange`, `_ReadWriteBarrier` | Loads, stores, exchanges, compare-and-swap and fetch-and-op with the meaning of each builtin; `__atomic_is_lock_free` / `__atomic_always_lock_free` are `1` for a power-of-two size up to two pointers |

`__func__`, `__FUNCTION__` and `__PRETTY_FUNCTION__` are the name of the enclosing function.

> [!WARNING]
> The atomic builtins compile as plain (volatile) loads and stores and plain read-modify-write sequences, and the fences (`__atomic_thread_fence`, `__sync_synchronize`, `_ReadWriteBarrier`) compile to nothing. They give each builtin's result in a single thread, but are not atomic across threads. C code that shares memory between threads through them is not yet correct under `nabicc`.

## 9.13.5. Options

| Option | Meaning |
|--------|---------|
| `-c` | Compile each source to an object; do not link |
| `-S` | Compile each source to assembly text (`<stem>.s`) |
| `-E` | Preprocess only, to standard output or `-o` |
| `-o <file>` | The output. With `-c`, `-S` or `-E` it takes one source |
| `-I <dir>`, `-isystem <dir>`, `-iquote <dir>`, `-idirafter <dir>` | Header search directories, searched as gcc searches them |
| `-include <file>` | Read `<file>` before the source |
| `--sysroot <dir>`, `-isysroot <dir>` | The target's headers and C library (9.13.7) |
| `-nostdinc`, `-nostdlibinc`, `-nobuiltininc` | Leave out the system headers, the C library's headers, or the built-in headers |
| `-D <name>[=<value>]`, `-U <name>` | Define or remove a macro |
| `-Wp,<opts>` | Preprocessor options of this table, separated by commas |
| `-std=<std>` | The standard (9.13.2) |
| `-O0`, `-O1`, `-O2`, `-O3`, `-Os`, `-Oz` | The optimization level (default `-O0`); `-O` is `-O1`, `-Og` is `-O1`, `-Ofast` is `-O3` |
| `-g`, `-g0` | Debug information on or off (`-ggdb`, `-g2`, `-g3`, `-gdwarf*`, `-gcodeview` turn it on) |
| `-l<lib>`, `-L<dir>`, `-Wl,<opts>` | Libraries, library directories and linker options for the link |
| `-M`, `-MM` | Print a make rule of the source and the headers it reads, and do not compile; `-MM` leaves out system headers |
| `-MD`, `-MMD` | Write that rule while compiling, to `<output or stem>.d`; `-MMD` leaves out system headers |
| `-MF <file>` | Write the rule to `<file>` |
| `-MT <target>`, `-MQ <target>` | The rule's target (repeatable); default the object |
| `-MP` | Add an empty rule for each header, so a deleted header does not stop make |
| `--target=<triple>`, `-target <triple>` | Compile for another target (9.13.7) |
| `-x c` | Read the following inputs as C whatever their extension |
| `-dumpmachine` | Print the target's C triple |
| `-dumpversion`, `-dumpfullversion` | Print the toolchain's version |
| `--version` | Print `nabicc <version>`; with `-v`, also `host: <triple>` |
| `-v` | Verbose (with `--version`) |
| `--help`, `-h` | Print usage |

`--target` takes Nabi's target names (`nabi --print target-list`, [0.6](../000-meta/006-platform-targets.md)) and the triples clang and gcc take for them: `x86_64-linux-gnu` is `x86_64-unknown-linux-gnu`, `arm64-apple-macos` is `aarch64-apple-darwin`, `aarch64-linux-musl` is `aarch64-unknown-linux-musl`, `i386-linux-gnu` is `i686-unknown-linux-gnu`, `armv7-linux-gnueabihf` is `armv7-unknown-linux-gnueabihf`, `armv7-linux-musleabihf` is `armv7-unknown-linux-musleabihf`, `riscv64-linux-gnu` is `riscv64gc-unknown-linux-gnu`. A name that is none of these is an error.

These are accepted and have no effect: every other `-W*` (warnings), `-f*` and `-m*` option, `--param`, `-fPIC`, `-fpic`, `-fPIE`, `-fpie`, `-pie`, `-no-pie`, `-rdynamic`, `-pthread`, `-pedantic`, `-pedantic-errors`, `-w`, `-pipe`, `-s`, `-static` and `-Qunused-arguments`. The code is always position independent. Any other option is an error that lists the options above.

Inputs ending in `.c`, `.h` or `.i` are C sources; every other input (objects, archives, `.lib` files) goes to the link in its place on the command line. The sources are compiled in parallel. Without `-o` the executable is `a.out` (`a.exe` for a Windows target), and an object is `<stem>.o` (`<stem>.obj` for a Windows target) in the current directory. Without a source and without link inputs it is an error (`no input files`).

## 9.13.6. Code generation

Each C function is lowered to nabi-mir as a C function: a body with the C ABI under its C name, as an `extern "C" fn` with `#[export_name]` is ([5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-definitions-exposed-externally)). Nabi's native code generator compiles it, with the nabi-mir optimizer and the code generator's optimizations at `-O1` and above ([9.2.10](902-aot-pipeline.md#9210-optimization-levels)). Integer arithmetic follows C: there are no overflow checks.

The calling convention is the target's C ABI, the same one Nabi's `extern "C"` uses ([8.6](../008-abi/806-calling-convention.md)), so a call from Nabi to C compiled by `nabicc`, or from that C to an `extern "C" fn` of Nabi, matches by construction. Floating-point operations are not contracted: `a * b + c` is a multiply and an add, never a fused multiply-add, as in Nabi code.

`-g` emits DWARF (Linux, Apple, wasm32) or CodeView (Windows) line tables and types. On Apple targets the link gathers the DWARF into a `.dSYM` bundle beside the executable; on Windows the linker writes a PDB ([9.2.14](902-aot-pipeline.md#9214-linking-for-executables)).

Executables are linked by Nabi's linker against the target's C library (9.13.7). On ELF targets `-lm` is always added. `-lc`, `-lpthread`, `-ldl` and `-lrt` name parts of the C library the link already has and are dropped, as are `-nostdlib`, `-nodefaultlibs` and `-nostartfiles`. On Windows the toolchain's startup calls C's `main(argc, argv, envp)` (a `wmain` is not called) and the link provides `atexit`. On 32-bit Arm and wasm32 the link takes the compiler helper functions (`__aeabi_idiv`, `__multi3`, ..) from the toolchain's runtime archive for the target.

## 9.13.7. Headers and C library

The headers and the C library come from the target:

| Target | Headers | C library |
|--------|---------|-----------|
| Apple | The macOS SDK: `SDKROOT`, else Xcode's or the Command Line Tools' | libSystem |
| Linux, glibc | The toolchain's sysroot for the target, `lib/nabi/targets/<target>/sysroot`: glibc 2.36 headers from the same pinned Debian packages as the C runtime it links ([9.2.14](902-aot-pipeline.md#9214-linking-for-executables)) | The shipped glibc link stubs |
| Linux, musl | No shipped sysroot yet: the host's own on a musl host, else `--sysroot <dir>` | The shipped musl |
| Windows | On a Windows host, the Windows SDK's UCRT headers and Visual Studio's C headers: `INCLUDE`, else the newest Visual Studio and Windows SDK found by `vswhere` | The UCRT, through the toolchain's own import libraries and startup; no MSVC library is linked |
| wasm32 | wasi-libc's headers, shipped as the target's sysroot | The shipped wasi-libc |
| Android | The Android NDK's sysroot (found as for a link, [0.6.1](../000-meta/006-platform-targets.md#android)), with its directory for the target | bionic: the NDK's crt objects and libraries of the API level |
| riscv32 (no OS) | Only the freestanding headers (below) | None |

When the target is the host and the toolchain ships no sysroot for it, the host's `/usr/include` serves. A target with none of these is an error that names the missing directory and `--sysroot <dir>`. `--sysroot <dir>` (or `-isysroot <dir>`) overrides the table: the headers under it are searched instead (`usr/include`, with the target's multiarch directory on Linux; `include` on wasm32).

Before the C library's headers come the built-in headers, the freestanding ones that stand for clang's own: `stddef.h`, `stdint.h`, `stdarg.h`, `stdbool.h`, `stdalign.h`, `stdnoreturn.h`, `stdatomic.h`, `limits.h`, `float.h`, `inttypes.h`, `iso646.h`, and the intrinsic headers of [std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#the-c-bindgen-reads), whose functions have no definition here. `-nobuiltininc` leaves them out; `-nostdinc` leaves out every system directory.

## 9.13.8. C sources of a package

A package lists its C sources in the field `c` of `nabi.nabion` ([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)):

```nabion
Package {
    name: "notes",
    c: {
        sources: ["vendor/sqlite3.c"],
        include: ["vendor"],
        define: { "SQLITE_THREADSAFE": "0", "NDEBUG": "" },
        std: "c17",
        flags: ["-Wall"],
    },
}
```

| Field | Meaning |
|-------|---------|
| `sources` | C files, relative to the manifest |
| `include` | Header directories, relative to the manifest (`-I`); those of the package's `Dep::system` libraries follow |
| `define` | Macros (`-D<name>=<value>`); an empty value defines the macro empty |
| `std` | The standard: `c89`, `c99`, `c11`, `c17`, `c23` or a `gnu` one (default `gnu17`) |
| `flags` | Further options: `-O*`, `-g`, `-W*`, `-f*` (not `-flto`) and `-m*` only |

The built-in compiler compiles each source at the optimization level of the build (`-O0`, or `-O2` with `--release`, or the `-O` given); an `-O` in `flags` takes its place. The objects go to `target/<profile>/c/<package>/`, and only a source whose inputs (it and the headers it read) or options changed compiles again. They are linked with the package wherever it is linked: into its executable and test harnesses, and into the executables of the packages that depend on it. Its Nabi code calls the C functions through an `extern "C"` block ([5.9](../005-modules/509-extern-block.md)); `--target` compiles the C for that target too.

An option in `flags` other than these is an error when the manifest is read, listing the options `flags` takes. A source or directory that does not exist is an error naming it. A C source that fails to compile fails the build with the compiler's error. No external C compiler is ever run, and no code from the package runs at build time (9.10.3).
