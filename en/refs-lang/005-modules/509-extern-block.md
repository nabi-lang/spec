# 5.9. `extern` blocks ([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) declarations)

The `extern` keyword (a contextual keyword that is a keyword only at the start of an item, [1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)) combined with an [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) specifier (`"C"` etc.) defines a block of declarations for external functions and globals. It is used to interface with external libraries, system calls, and other languages.

```
extern_block    ::= "extern" string_lit "{" extern_item* "}"
extern_item     ::= visibility? "fn" identifier "(" param_list? ")" return_type? ";"
                  | visibility? "static" "mut"? identifier ":" type_expr ";"
```

## 5.9.1. Basic form

Basic usage:

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
    fn strlen(s: ConstPtr<c_char>) -> usize;

    static environ: Ptr<Ptr<c_char>>;
    static mut atexit_count: u64;
}
```

A block consists of `extern`, an ABI string (`"C"`, `"system"`, etc.; the meaning of each ABI is defined in [Calling Convention](../008-abi/806-calling-convention.md)), and `{ ... }`. The body may contain only function declarations with a signature and no body, and `static` declarations.

> [!IMPORTANT]
> Function declarations inside an `extern "C"` block are **implicitly unsafe**. The caller may call them only inside an `unsafe { ... }` block; the effect is the same without writing `unsafe` in the signature. See [4.15 unsafe](../004-functions/415-unsafe.md) for the full definition.

## 5.9.2. ABI specifiers

The supported ABIs:

| ABI        | Meaning                                                                          |
|------------|----------------------------------------------------------------------------------|
| `"C"`      | The platform C ABI. Always the same regardless of symbol name.                   |
| `"system"` | The OS system-call ABI. Differs from `"C"` on some platforms (e.g. Win32 `stdcall`). |
| `"nabi"`    | The ABI of the runtime (`nabir`). See below.                                      |
| `"objc"`   | Objective-C classes and their methods, on Apple targets ([stdlib 1.10](../../refs-standard-libraries/100-bare/110-objc.md)); `ffi::objc::import { "AppKit/AppKit.h" }` writes these blocks from the SDK's headers ([stdlib 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-headers-ffiobjcimport)). |

On targets where `"system"` and `"C"` coincide (every supported target but 32-bit x86 Windows), `"system"` is `"C"`. On 32-bit x86 Windows it is stdcall ([5.9.10](#5910-extern-fn-definitions-exposed-externally)).

`extern "nabi"` is the ABI of runtime functions declared by the toolchain; those declarations are private items of the toolchain's packages (`core::rt` holds most of them; `bare` declares the ones it uses itself). `#[no_safepoint]` on an `extern "nabi"` block declares that its functions neither allocate nor block; calls to them are not safepoints. It is toolchain-internal and an error on `extern "C"` blocks.

| Value                           | Passing under `extern "nabi"`             |
|---------------------------------|------------------------------------------|
| `str`                           | two words `(ptr, len)`                   |
| `fn` value                      | `{code, env}` ([8.5.0a](../008-abi/805-extern-symbols.md#850a-nabifn---c-abi-form-of-function-values)) |
| `extern "C" fn(..)` parameter   | C function pointer without env           |
| Anything else                   | The value as is (same as the C ABI)      |

> [!NOTE]
> ABIs of other languages such as `"Rust"` are generally unstable and are not targets of `extern`. Interfacing with other languages is possible only over a stable ABI both sides agree on (usually `"C"`).

> [!NOTE]
> Nabi propagates errors as return values and does not use stack unwinding ([8.6.5](../008-abi/806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)). Exceptions or unwinding from a foreign language crossing a Nabi frame are therefore undefined behavior; foreign functions must handle exceptions within the call boundary.

A plain `extern { ... }` without an ABI is a compile error; the ABI must be specified.

## 5.9.3. Items permitted in the body

### External function declarations

An external function is declared as:

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
}
```

The signature has the same form as an ordinary function declaration, but has no body and ends with `;`. Parameters and return types may use only FFI-compatible types such as primitive integers, floating-point, and pointers.

### External `static`

A global variable defined externally is referenced as:

```nabi
extern "C" {
    static environ: Ptr<Ptr<c_char>>;  // read-only
    static mut atexit_count: u64;      // mutable
}
```

This form references a global variable defined externally; `static mut` is allowed only inside `extern` blocks ([`static`](503-static.md)). C's `errno` is not such a variable: it is a per-thread macro, so a `static errno` reads the wrong memory. `#[errno]` on the declaration saves it after the call ([stdlib 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errno---errno-and-getlasterror)).

### Not permitted

`struct`, `enum`, `impl`, `fn` bodies, `const`, etc. do not appear in the body of an `extern` block. The block only declares the boundary with an external ABI.

## 5.9.4. FFI-compatible types

The types of parameters, returns, and `static`s must all be FFI-compatible. The FFI types are:

- Primitive integers: `i8` to `i64`, `u8` to `u64`, `usize`, `isize`.
- C's named types, aliases of these primitives sized as the target's C compiler sizes them: `c_int`, `c_uint`, `c_long`, `c_ulong`, `c_size_t`, `c_wchar` and the rest of `bare::ffi::types`, all in the prelude ([stdlib 1.7.9](../../refs-standard-libraries/100-bare/107-ffi.md#179-c-types---c_int-c_long-c_void)). `c_long` is 32 bits on Windows and on 32-bit targets, so a C `long` is declared `c_long`, not `i64`. `c_void` stands for `void` behind a pointer (`CPtr<c_void>`).
- Primitive floating-point: `f32`, `f64`.
- `bool` (C `bool`) and `c_char` (1 byte).
- `f16` / `bf16`: C's `_Float16` / `__bf16`, on targets whose C has them ([8.6.3](../008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)).
- `Simd<T, N>` of 8 or 16 bytes, as a parameter or result: the target's C vector of that size (`float32x4_t`, `__m128`), on AAPCS64 and x86-64 ([8.6.3](../008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)).
- Pointers: `Ptr<T>`, `ConstPtr<T>`, `CPtr<T>` (`bare::ptr`, the lowest of the [standard library tiers](../000-meta/005-standard-libraries-tiers.md)).
- `struct` and `union` types whose fields are FFI types (fixed arrays of them included). The default layout is already C's; `#[repr(packed)]` and `#[repr(align(N))]` change it as in C.
- Fieldless enums with an integer `#[repr]` (`#[repr(i32)]`): a C `enum`.
- Function pointers: `extern "C" fn(...) -> ...` (see "C function pointer types" below).
- Function-typed parameters: an `fn(..) -> R` parameter of an `extern "C"` declaration is the same as `extern "C" fn(..) -> R`. The argument must name a function; a closure is an error (`` `each` is a C function: a `fn` argument must name a function (a C callback has no room for a closure's captures) ``) whose help says where its state goes: `` a C callback cannot capture: keep the state in `Handle::new(..)` (`bare::ffi`), pass `h.as_ptr()` as the `void*` argument, and read it back in an `extern "C" fn` with `Handle::<T>::from_ptr(p).get()` `` ([stdlib 1.7.10](../../refs-standard-libraries/100-bare/107-ffi.md#1710-handlet---nabi-values-c-holds)).
- Nullable pointers: `Ptr<T> | none`, `ConstPtr<T> | none`, `extern "C" fn(..) -> R | none` (a pointer with a null niche, [2.7.2](../002-types/207-niche-optimization.md#272-automatic-niches)), and any `#[niche(0)]` struct `| none` (`NonZero<i32> | none` is an `int`); `none` is null.
- `str` as a parameter of an `extern "C"` declaration: converted by the rule below.
- `()` and `!` as a result.

A `str` parameter in an `extern "C"` declaration is a single NUL-terminated `char*`. If a NUL follows the bytes, the bytes are passed as is; otherwise (a slice, etc.) a copy pinned for the duration of the call is passed. C must not retain that pointer after the call returns. A `str` result is an error: the bytes of a returned `char*` belong to C (a static buffer the next call overwrites, memory the caller must free, a pointer into an argument's copy that dies with the call), so a `str` over them could change or dangle, and they need not be UTF-8. Declare the result `ConstPtr<c_char> | none` and copy the string with `CStr::from_ptr` ([stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md#172-cstr--cstring---null-terminated-string)), which also says whether it is UTF-8:

```nabi
extern "C" {
    fn getenv(name: str) -> ConstPtr<c_char> | none;
}
let home = match unsafe { getenv("HOME") } {
    p: ConstPtr<c_char> => unsafe { CStr::from_ptr(p) },
    none => none,
};
```

A `pub` function of an `extern` block in module `m` is also reachable as `m::f`; its symbol is unprefixed. A `nabi_*` name in an `extern "C"` block may not take or return `str` ([8.5](../008-abi/805-extern-symbols.md)).

Every other type is an error in the parameters, results and `static`s of `extern "C"` and `extern "system"` blocks and of `extern "C" fn` definitions. The check runs on the resolved type, so an alias (`type S = str;`), a generic instance, a field or an array element is seen through. Each error says what to write instead:

| Type | Why | Write instead |
|------|-----|---------------|
| A GC type (`List`, `Dict`, a closure, `any`, a struct holding one) | the collector may move or free it while C holds it | a primitive, a `CPtr<T>` and a length, or a struct of those; a `List<T>` pinned with `gc::pin` and passed as `p.as_data_ptr()` and `xs.len()` |
| `str` as a result, or in an `extern "C" fn` definition | C owns the bytes (above); C passes a `char*` | `ConstPtr<c_char> \| none`, copied with `CStr::from_ptr` |
| `char` | C has no four-byte character | `u32` and `char::from_u32` |
| A tuple | C has no tuples | a `struct` |
| A plain enum | its layout is the compiler's | `#[repr(i32)]`, or an `i32` matched by the program |
| An enum with data | C has no tagged enum | a struct of a tag and a `union` |
| An array parameter or result | C passes an array as a pointer | `ConstPtr<T>` (`addr_of(xs)`), or the array in a struct |
| Another sum (`i32 \| none`) | only a null pointer has a C form | `ConstPtr<T> \| none`, or the value and a flag |
| A nabi `fn` value outside a declaration's parameter | it is a function and its captures | `extern "C" fn(..)`, made from a named `extern "C" fn` |
| `Mask<T, N>` | no C type | a `Simd` of integers (`mask.select(Simd::splat(1), Simd::splat(0))`) |
| A struct holding a `Simd`, passed by value | C vectors in aggregates are not passed | a pointer to the struct |

The message names the parameter and the type: `` parameter `xs` of extern function `f` has type `List<i64>`, which C cannot hold: the collector may move or free it ``. For a `List<T>` the help is the call to write: `` help: C takes a list as a pointer and a length: declare `ptr: ConstPtr<i64>, len: usize`, and pass `p.as_data_ptr()` and `xs.len()` from `let p = gc::pin(xs);` (`bare::gc`) ``. Where the target's C has no `_Float16`, no `__bf16` or no vector of the size, the error comes from code generation for that target, with the same kind of help. The compiler cannot check what it cannot see: that the C side declares the same function with the same types remains the program's to ensure. The calling convention for each type is defined in [Calling Convention](../008-abi/806-calling-convention.md).

### C function pointer types

```
c_fn_type ::= "unsafe"? "extern" ("\"C\"" | "\"system\"") "fn" "(" (type_expr ("," type_expr)*)? ")" ("->" unary_type)?
```

- The ABI string may only be `"C"` or `"system"`. Anything else is an error.
- `extern "system" fn(..) -> R` follows the same rules as `extern "C" fn(..) -> R`, so Win32 callbacks are expressible. It is a distinct type only on 32-bit x86 Windows, where it is stdcall; elsewhere the two are the same type.
- Values of this type are made only from named C-ABI functions. The value of a C declaration in an `extern` block has type `unsafe extern "C" fn(..)` and cannot be put in a safe (non-`unsafe`) position; a Nabi `extern "C" fn` definition (its C entry point, 5.9.10) yields a safe `extern "C" fn(..)` value. A plain `fn` in that position is an error, `` only an `extern "C"` function (an `extern` block declaration or an `extern "C" fn` definition) is an `extern "C" fn(..)` value ``, with help `` C calls it with the C ABI: declare it `extern "C" fn twice(..)` ``.
- Nabi `fn` values (including closures and variables of function type) and C function pointers do not convert to each other.
- Calling through a type without `unsafe` is safe; calling through `unsafe extern "C" fn` requires `unsafe`.
- A non-`none` C function pointer value is not null. Receiving NULL into an `extern "C" fn` position without `| none` (parameter, return, field) is therefore undefined behavior, and a C API that may return NULL must be declared with `| none`.

#### Nullable C function pointers: `extern "C" fn(..) -> R | none`

A sum consisting only of one C function pointer type and `none` (in any order, no other members) uses the null niche ([2.7.2](../002-types/207-niche-optimization.md#272-automatic-niches)).

- It is pointer-sized, and `none` is null.
- It has the same ABI as a C nullable function pointer. It can be used as is in extern parameters / returns, struct fields exchanged with C, and extern `static`s.
- Zero-filled storage reads as `none`.
- A larger sum containing it uses the ordinary tagged layout, and conversion between them is ordinary sum type conversion.

```nabi
extern "C" {
    fn cmp_i32(a: ConstPtr<u8>, b: ConstPtr<u8>) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32);
}
let f: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;   // a C declaration's value is unsafe
let g: extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;          // ERROR: safe position
```

### Name space of extern functions

Functions declared in `extern` blocks form a single global name space and can be called by unprefixed name from any module. This mirrors C symbols, which live in a single global space at link time. It is an exception to the "no implicit search of ancestor modules" rule for ordinary items ([5.7.1](507-paths.md#571-starting-identifier-of-a-path)) and applies only to extern functions. Declaring an extern function of the same name twice with different signatures is a compile error. The toolchain's runtime declarations in `core::rt` count: a user `extern` that redeclares a `nabi_*` runtime function must match its signature.

## 5.9.5. `Ptr<T>` / `ConstPtr<T>`

Raw pointer types defined in `bare::ptr`, the lowest of the [standard library tiers](../000-meta/005-standard-libraries-tiers.md). They are not in the prelude: `use bare::ptr::{Ptr, ConstPtr};`.

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
    fn read_byte(p: ConstPtr<u8>) -> u8;
}
```

`Ptr<T>` is a mutable raw pointer corresponding to C's `T*`, and `ConstPtr<T>` is a read-only raw pointer corresponding to C's `const T*`. Neither is ever null. A pointer that may be null is `Ptr<T> | none` (or `ConstPtr<T> | none`): one address, with null as `none` ([niche optimization](../002-types/207-niche-optimization.md)). `CPtr<T>` is a C pointer that may hold NULL as a value (`is_null()`, `as_ptr() -> Ptr<T> | none`).

## 5.9.6. Calls / use

Functions declared in an `extern` block are called like ordinary Nabi functions.

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
}

fn use_malloc() {
    let p = unsafe { malloc(1024usize) };
    if p is Ptr<u8> {
        // ... use ...
        unsafe { free(p) };
    }
}
```

Every `extern` function, `extern "nabi"` included, is unsafe and is called inside `unsafe { ... }` ([4.15](../004-functions/415-unsafe.md)); the toolchain's packages wrap the runtime functions they declare in safe functions. Otherwise an `extern` function call differs from an ordinary call only in that it is an external symbol. Nabi's ordinary semantics apply to the result type. An `extern static` is read and (if `mut`) written like an ordinary variable. Nabi cannot check a C global, so reading or writing one needs `unsafe` (an `unsafe` block or an `unsafe fn` body); outside it is an error whose help quotes the fix: `unsafe { optind = 6; }`. Taking its address with `addr_of` reads nothing and needs no `unsafe`. Its type must still be FFI-compatible (5.9.4): Nabi reads the foreign memory as that type, so `static environ: List<str>;` is undefined behavior; declare it as a pointer (`static environ: Ptr<Ptr<c_char>>;`).

## 5.9.7. Visibility

Each item of an `extern` block may carry a visibility keyword.

```nabi
extern "C" {
    pub fn public_api(x: i32) -> i32;       // public
        fn _internal(x: i32) -> i32;        // module-private
    pub static VERSION: i32;
}
```

The `extern "C"` block itself takes no visibility keyword; visibility is given per item.

## 5.9.8. Position of the `extern` block itself

An `extern "C" { ... }` block may appear only at module level. It cannot be placed inside a function or an `impl`.

```nabi
fn f() {
    extern "C" { ... }                     // ERROR: not inside a function
}
```

## 5.9.9. `#[link("libname")]`: naming the library

To name the external library, use the `#[link]` attribute.

```nabi
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
    fn sqrt(x: f64) -> f64;
}
```

`#[link("x")]` or `#[link(name = "x", kind = "static" | "dylib")]` on an extern block passes `-lx` to the linker. The library search path is the `-L <dir>` directories, then `LIBRARY_PATH`. On Apple targets, `kind = "static"` links the `lib<name>.a` found there by its path, and `#[link(name = "Foundation", kind = "framework")]` links the framework `Foundation.framework`, searched in the `-F <dir>` directories, then the SDK's `System/Library/Frameworks`; on other targets a framework is a link error. nabic's `-l` and a native package's `links` add libraries the same way, and `Dep::system` / `Dep::framework` libraries of `nabi.nabion` too ([9.10.4b](../009-tools/910-nabi-frontend.md#9104b-native-packages)). On wasm32, `#[link(wasm_import_module = "m")]` imports the block's functions from core module `m`, and `#[link_name = "sym"]` on a function sets its linked symbol or import name ([6.2.3](../006-meta-system/602-built-in-attributes.md#623-link-naming-an-external-library)). Without `#[link]`, symbols are resolved at link time through system defaults (`libc`, etc.) or other mechanisms, such as the objects of the package's own C sources, which the built-in C compiler compiles and links with the package ([9.13.8](../009-tools/913-c-compiler.md#9138-c-sources-of-a-package)); their functions are declared in an `extern "C"` block like any other. The behavior of `#[link]` is defined in [built-in attributes](../006-meta-system/602-built-in-attributes.md).

## 5.9.10. `extern fn` definitions (exposed externally)

The forms in the preceding sections call external functions from nabi. The reverse form exposes a Nabi function through an external ABI.

```nabi
#[export]
pub extern "C" fn nabi_compute(x: c_int, y: c_int) -> c_int {
    x + y
}

#[export_name("nabi_area")]
pub extern "C" fn area(w: c_int, h: c_int) -> c_int { w * h }

extern "C" fn callback(arg: i32) -> i32 {
    arg * 2
}
```

An `extern "ABI"` qualifier on an ordinary function declaration marks the function to be compiled compatible with that ABI. An `extern "C" fn` definition keeps its mangled symbol name ([9.5](../009-tools/905-symbol-mangling.md)); for a name C can link to, add `#[export]`, which exports it under the function's own name (`nabi_compute`), or `#[export_name("sym")]`, which exports it as `sym` (`area` as `nabi_area`). An exported function is a root for unreachable-function elimination, and the exports are what a shared library (`--crate-type=cdylib`, [9.3](../009-tools/903-package-archive.md)) makes visible.

An `extern "ABI" fn` definition takes `"C"` or `"system"`; any other ABI string is an error. On 32-bit x86 Windows an `extern "system"` declaration or definition is stdcall: the callee pops its arguments, and the symbol is `_name@N` as with MSVC.

`extern "C"` and `extern "system"` fn definitions have these restrictions:

- They cannot be generic, `async`, or `throws`.
- Parameter and return types must be FFI-compatible types from 5.9.4; non-FFI types such as `str` cannot be used. (The `str` conversion of 5.9.4 applies only to declarations that *call* C functions.)

`#[export]` and `#[export_name]` go only on `extern "C"` and `extern "system"` fn definitions. C calls an export with the C convention, and a function of the Nabi ABI would get its aggregates, `str`s and narrow integers wrong, so on a plain function either one is an error: `` `compute` is exported to C but uses the Nabi ABI: write `pub extern "C" fn compute(..)` ``. On wasm32 an export is a wasm export of that name.

```nabi
#[export_name("on_event")]
extern "C" fn on_event(code: i32) -> i32 { code }     // visible from C as on_event
extern "C" fn bad(s: str) { }                          // ERROR: str cannot be used
#[export]
pub fn compute(x: i64) -> i64 { x * 2 }               // ERROR: exported, but the Nabi ABI
```

### `#[on_panic]` - a value for C instead of an abort

```nabi
#[on_panic(return = 0)]
extern "C" fn cmp(a: Ptr<i32>, b: Ptr<i32>) -> c_int { .. }

#[on_panic(return = -1.5)]
extern "C" fn half(x: f64) -> f64 { .. }

#[on_panic]
extern "C" fn notify(x: i32) { .. }
```

A panic that would leave a function C called through C's frames aborts the process (5.9.12). `#[on_panic(return = v)]` on an `extern "C"` or `extern "system"` fn definition recovers instead: the panic is reported as usual and the panic hook runs ([stdlib 3.12](../../refs-standard-libraries/300-std/312-panic.md)), then the function returns `v` to its caller, C or Nabi. `v` is a literal of the result type: an integer, a float, `true` / `false`, or `0` for a pointer result (a null pointer). A function without a result takes `#[on_panic]` alone. The function that recovers is the nearest C entry below the panic: when C calls back into Nabi again further down, that inner entry decides.

The frames between the panic and the function are skipped. Their `Drop`s do not run, so what they would release leaks, and a `Mutex` locked in a skipped frame stays locked: the next `lock` of it waits forever. A function that can fail while it holds a lock returns an error value instead of panicking. The recovery is `setjmp` / `longjmp` ([8.5.12](../008-abi/805-extern-symbols.md#8512-auxiliary-symbols-appendix-non-normative)); wasm32 and the targets without an OS have none, and there the panic aborts as without the attribute.

Each misuse is an error that says what to write:

| Case | Message |
|------|---------|
| a value that is not a number, a `bool` or `0` | `` `#[on_panic]` on `count`: only a number, a `bool` or `0` (a null pointer) can be returned; write `#[on_panic(return = -1)]` with a value of the result type `` |
| a value for a function without a result | `` `#[on_panic]` on `nothing`: it returns nothing, so write `#[on_panic]` alone `` |
| no value for a function with a result | `` `#[on_panic]` on `half`: a function with a result needs the value it returns `` |
| a function that is not `extern "C"` | `` `#[on_panic]` is for functions C calls: write `#[on_panic(return = -1)] extern "C" fn plain(..)` `` |

Calling external functions and exposing functions externally together form the two directions of FFI.

## 5.9.11. Calling rules of `extern fn`

`extern "C"` functions follow the C calling convention and may differ in part from the ordinary function call rules covered in [execution engines and user surface](../000-meta/001-execution-modes.md). The differences in calling convention are covered in the runtime ABI specification (chapters 7 / 8).

## 5.9.12. Responsibility for FFI safety

Every item of an `extern` block is the user's responsibility (consistent with the `unsafe` spirit of the [standard library tiers](../000-meta/005-standard-libraries-tiers.md)). The compiler verifies:

- The signature form.
- That every type is an FFI type (5.9.4), seen through aliases, generics and fields, and that the target's C passes it.

The compiler does not verify:

- The actual behavior of external functions (null returns, error returns, memory leaks, etc.).
- Concurrency safety of external `static`s.
- True ABI compatibility of external libraries.

The user must guarantee these.

### Panics under C frames

A panic must not leave a function C called through C's frames: C knows nothing of it, and an `exit` would run the host's handlers (`atexit`, an interpreter's shutdown) on broken state. A panic in an `extern "C"` or `extern "system"` fn that foreign code called, or in Nabi code below it, therefore prints its report ([4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors)), then the line

```text
panic in a function C called: aborting (`#[on_panic(return = v)]` on the `extern "C" fn` returns `v` to C instead)
```

and aborts the process (`SIGABRT`, exit status 134 on Unix). A panic with no foreign frame below it ends the process with exit code 101, as it does anywhere else. A shared library (`--crate-type=cdylib`) follows the same rule, so a host such as Python, R or Julia never sees its process `exit`. `#[on_panic]` (5.9.10) turns the panic into a return value, and `std::panic::set_hook` ([stdlib 3.12](../../refs-standard-libraries/300-std/312-panic.md)) sees every panic before the abort, the exit or the recovery.
