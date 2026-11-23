# 4.15. `unsafe` - function modifier and block expression

The `unsafe` keyword appears in two forms that visibly confine regions whose safety the compiler does not guarantee: the **modifier on function / method declarations** and the **block expression**. The two work as a pair: a function declared `unsafe fn` must be called inside an `unsafe { ... }` block.

```
unsafe_fn        ::= "unsafe" fn_decl
unsafe_block     ::= "unsafe" block                // block: 3.9
```

This model is isomorphic to Rust's `unsafe`; its purpose is to make dangerous regions explicit in the code.

## 4.15.1. `unsafe fn` - function modifier

Placing `unsafe` before a function or method declaration means that the function's body is outside the guarantees of the safe abstraction. The signature states directly that the caller is responsible for satisfying the preconditions.

```nabi
impl<T> Ptr<T> {
    unsafe fn read(self) -> T;
    unsafe fn write(self, value: T);
}

extern "C" {
    unsafe fn strlen(s: ConstPtr<c_char>) -> usize;
}

unsafe fn transmute_unchecked<A, B>(a: A) -> B { ... }
```

Inside the body of an `unsafe fn` itself, unsafe calls are allowed without a separate `unsafe { ... }` block. Because the signature already declares unsafe, the whole body is treated as an unsafe scope.

```nabi
unsafe fn copy_two<T>(p: Ptr<T>) -> (T, T) {
    let a = p.read();              // body is an unsafe scope: no explicit block needed
    let b = p.add(1).read();
    (a, b)
}
```

### With method receivers

Methods in `impl` follow the same form.

```nabi
impl Buffer {
    unsafe fn raw_data(self) -> Ptr<u8>;
    unsafe fn raw_data_mut(ref self) -> Ptr<u8>;
}
```

`unsafe` does not conflict with receiver notations such as `ref self` / `self`, and it comes first in the function header.

### Inside `extern "C"` blocks

User-declared C ABI functions are all implicitly `unsafe`, because their definitions are outside the compiler's control ([extern blocks](../005-modules/509-extern-block.md)). Writing it explicitly (`unsafe fn`) is recommended; if omitted, it is treated as `unsafe` automatically.

The same holds for every `extern` function, `extern "nabi"` declarations of the runtime's symbols included: calling one outside an unsafe scope is an error (E0061).

```nabi
extern "C" {
    fn malloc(size: usize) -> CPtr<u8>;        // treated as unsafe automatically
    unsafe fn free(ptr: CPtr<u8>);             // explicit (recommended)
}
```

## 4.15.2. `unsafe { ... }` - block expression

Operations such as calling unsafe functions or dereferencing raw pointers are allowed only inside an `unsafe { ... }` block. Like an ordinary [block expression](../003-expressions/309-block-expressions.md), the block's result is the value of its last expression.

```nabi
let n: usize = unsafe { strlen(cs.as_ptr()) };

let v: i32 = unsafe {
    let p = ptr.add(idx);
    p.read()
};

unsafe {
    free(ptr);
}
```

### Operations allowed in the block

Inside an `unsafe { ... }` block, the following operations are additionally allowed:

- Calling functions / methods declared `unsafe fn`
- Dereferencing raw pointers (`Ptr<T>`, `ConstPtr<T>`, `ManyPtr<T>`) (`p.read()`, `p.write(v)`)
- Directly calling dangerous functions of the [`bare::*`](../../refs-standard-libraries/100-bare/README.md) modules
- Reading `union` fields (bit reinterpretation, [union](../002-types/205-union.md)). Writing fields is allowed without `unsafe`.
- Using a value after it was handed to another owner ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)): the block's author guarantees no other thread uses it at the same time.

All other ordinary safe operations work the same inside an unsafe block: `unsafe` *adds capabilities*; it does not *change existing meaning*.

### Narrowing the scope

`unsafe` blocks should be kept as narrow as possible. Instead of wrapping a whole function in `unsafe { ... }`, wrapping only the one line that is actually dangerous makes the safety of the other lines visibly clear.

```nabi
// recommended: narrow unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    let validated = try check_ptr(p);
    unsafe { validated.read() }    // only the dangerous operation is unsafe
}

// not recommended: wide unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    unsafe {
        let validated = try check_ptr(p);
        validated.read()
    }
}
```

## 4.15.3. Call rules

Calls of `unsafe fn` follow these rules.

| Call position                  | Can call `unsafe fn`?        |
|--------------------------------|------------------------------|
| ordinary function body         | ✗ (compile error)            |
| `unsafe fn` body               | ✓ (automatic unsafe scope)   |
| `unsafe { ... }` block         | ✓                            |

Violating the rules is a compile error:

```nabi
fn safe_caller(p: Ptr<i32>) -> i32 {
    p.read()    // ERROR (E0061): call to unsafe function `read` outside an `unsafe` block
}
```

There are two fixes: wrap the call in `unsafe { ... }`, or mark the caller itself `unsafe fn`.

### `unsafe fn` types

When an `unsafe fn` or a C function is used as a value, its type is `unsafe fn(..) -> R` ([4.4](404-function-type.md)). Calling a value of this type requires `unsafe`. A safe `fn(..) -> R` value may be used in an `unsafe fn(..) -> R` position, but not the reverse.

```nabi
let f: unsafe fn(usize) -> CPtr<u8> = malloc;   // C function value
let p = unsafe { f(16) };
let g: unsafe fn(i32) -> i32 = |x| x + 1;      // OK: safe → unsafe
let h: fn(usize) -> CPtr<u8> = f;              // ERROR: unsafe → safe
```

## 4.15.4. In traits

Trait methods may also use `unsafe fn`. The `impl` side must implement them with the same signature, `unsafe` included: a safe impl of an `unsafe fn` method, or an `unsafe fn` impl of a safe one, is an error (E0074). Callers follow the same rules as for ordinary call positions.

```nabi
trait RawAccess {
    unsafe fn raw_get(self, idx: usize) -> u8;
}

impl RawAccess for Buffer {
    unsafe fn raw_get(self, idx: usize) -> u8 {
        self.ptr.add(idx).read()
    }
}

let buf: Buffer = ...;
let b = unsafe { buf.raw_get(0) };
```

### `unsafe trait`

A trait whose impls promise something the compiler cannot check is declared `unsafe trait`. Every impl of it is `unsafe impl`; a plain `impl` is a compile error (E0078) that gives the `unsafe impl` form. Using the trait (bounds, calls) needs no `unsafe`. `bare::mem::Pod` (every bit pattern is a value) is one ([stdlib 1.3.3](../../refs-standard-libraries/100-bare/103-mem.md#133-the-pod-trait)).

```nabi
/// Every all-zero bit pattern is a value.
unsafe trait Zeroable {
    fn zeroed() -> Self;
}

unsafe impl Zeroable for i32 {
    fn zeroed() -> i32 { 0 }
}
```

### `unsafe impl`

`unsafe impl` implements an `unsafe trait`, or the auto marker traits `Send` / `Sync` by hand. It marks that the author guarantees what the trait promises (for `Send` / `Sync`, that the type is safe to cross thread boundaries). Using `unsafe impl` for any other trait is a compile error ([2.9.1](../002-types/209-impl.md#291-two-forms)).

```nabi
unsafe impl Send for Handle { }
```

## 4.15.5. Responsibility model

`unsafe` does not mean the compiler *gives up* safety checking. It means that the following two responsibilities transfer to the caller:

1. **Memory safety**: the responsibility not to cause undefined behavior ([4.15.8](#4158-undefined-behavior)) such as dangling pointers, misaligned access or data races.
2. **Preconditions**: the responsibility to satisfy the usage conditions promised by the `unsafe fn`'s documentation (e.g., that a pointer points to a valid object, that a length is within the buffer).

`unsafe { ... }` is the visible signal that the caller accepts these two responsibilities, so dangerous regions are apparent at a glance during code review.

## 4.15.6. Relationship to implicit unsafe

The following positions are treated as unsafe scopes automatically, without an explicit `unsafe`:

- The body of an `unsafe fn`

This automatic treatment reduces noisy repetition of `unsafe { ... }` in commonly occurring dangerous regions.

## 4.15.7. Comparison with other languages

| Language | unsafe notation model                                       |
|----------|-------------------------------------------------------------|
| Rust     | `unsafe fn` + `unsafe { ... }` (isomorphic)                 |
| C / C++  | no separate notation; all code is potentially unsafe        |
| Swift    | `@unsafe` annotation / `withUnsafePointer` functional pattern |
| Go       | `unsafe` package (separate import); no block notation       |
| Zig      | explicit builtins such as `@ptrCast`; no block notation     |
| **Nabi**  | **adopts Rust-style `unsafe fn` + `unsafe { ... }`**        |

Nabi follows Rust's model as-is. Visual separation of dangerous regions is marked consistently with a single keyword, which benefits both code review and static analysis.

## 4.15.8. Undefined behavior

This is the complete list of undefined behavior in Nabi. A program whose run does one of these things has no meaning: the compiler may assume it never happens, and anything may follow. Every item needs `unsafe` code (or foreign code) to happen; safe Nabi cannot cause any of them. Each library chapter that names an undefined behavior names one of these kinds.

Memory and pointers:

1. Reading or writing through a pointer that does not point at live memory of its type: null, dangling (the local, pin or allocation it came from is gone), out of the bounds of its object, or past a `ManyPtr` / `GcBuf` / `Slice::from_raw_parts` range the caller vouched for ([stdlib 1.1](../../refs-standard-libraries/100-bare/101-ptr.md), [stdlib 1.3.9](../../refs-standard-libraries/100-bare/103-mem.md)).
2. Using a raw pointer into a GC object after the pin that made it valid was released ([stdlib 1.6](../../refs-standard-libraries/100-bare/106-gc-pin.md)).
3. Reading or writing through a misaligned pointer, as `ptr_cast` can make ([stdlib 1.2.2](../../refs-standard-libraries/100-bare/102-cast.md)).
4. Dereferencing a pointer whose provenance was lost, other than one made by `from_exposed_provenance` from an exposed address ([stdlib 1.1.2](../../refs-standard-libraries/100-bare/101-ptr.md)).
5. Writing through a pointer to a non-`mut` `static`, or through `const_cast`'s result to memory that is not writable ([stdlib 1.1.2](../../refs-standard-libraries/100-bare/101-ptr.md), [stdlib 1.2.4](../../refs-standard-libraries/100-bare/102-cast.md)).
6. `copy_nonoverlapping` on overlapping ranges ([stdlib 1.3.4](../../refs-standard-libraries/100-bare/103-mem.md)).
7. Using a value after `drop_in_place` ran its drop glue, or running it twice ([stdlib 1.3.4](../../refs-standard-libraries/100-bare/103-mem.md)).

Values:

8. Reading uninitialized memory, at any type: `MaybeUninit::assume_init` before the value was written, or a read of bytes nothing wrote ([stdlib 1.3](../../refs-standard-libraries/100-bare/103-mem.md)).
9. Producing a value that is not valid for its type, by a `union` read ([2.5.2](../002-types/205-union.md)), `transmute_unchecked`, a raw write or a false `unsafe impl Pod`: a `bool` other than `0` / `1`, a `char` that is not a Unicode scalar value, an enum or sum tag no variant has, a GC reference that is not null or a live object, a non-`none` `extern "C" fn` that is null ([stdlib 1.2.6](../../refs-standard-libraries/100-bare/102-cast.md), [5.9.4](../005-modules/509-extern-block.md)).
10. Breaking the invariant of a checked wrapper made with `new_unchecked`: a zero `NonZero`, a NaN `NonNan`, a NaN or infinite `Finite`, a negative `Positive` ([stdlib 2.2.6](../../refs-standard-libraries/200-core/202-non-zero.md)).
11. A GC reference in memory outside the GC heap that the collector does not scan: a `ManyPtr::slice` over such memory holding GC references, or one stored there by a raw write ([stdlib 1.1.4](../../refs-standard-libraries/100-bare/101-ptr.md)).

Threads:

12. A data race ([7.9.4](../007-runtime/709-memory-model.md#794-data-races)): through raw pointers or `GcBuf`, a false `unsafe impl Send` / `Sync`, `Mutex::new_unshared` on a value something else reaches, a hand-off check skipped inside `unsafe`, or C code that touches the environment while `env::set_var` runs ([stdlib 3.5](../../refs-standard-libraries/300-std/305-env.md)).

Foreign code:

13. A foreign function, `static` or callback whose declared type does not match the foreign definition, or a GC value passed where C holds raw bits ([5.9.4](../005-modules/509-extern-block.md), [5.9.6](../005-modules/509-extern-block.md)).
14. An exception or unwind from foreign code crossing a Nabi frame ([5.9.2](../005-modules/509-extern-block.md)).
15. A null `char*` returned where the declaration says `str`; C keeping a `str` argument's pointer after the call returns ([5.9.4](../005-modules/509-extern-block.md)).
16. `CStr::from_ptr` on a pointer that is dangling or points at bytes without a NUL ([stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md)).

Everything else is defined, in safe and unsafe code alike. In particular these are not undefined behavior: integer overflow (a panic where checks are on, wrapping where they are off, [stdlib 2.3](../../refs-standard-libraries/200-core/203-ops.md#overflow-behavior)), an out-of-bounds index or slice of a collection (a panic), division by zero (a panic), stack overflow (the process aborts, [7](../007-runtime/README.md)), every floating-point operation, NaN and infinity included ([stdlib 2.1.4](../../refs-standard-libraries/200-core/201-numbers.md#floating-point-semantics)), reading a `union` field (bit reinterpretation; only the value it makes can be invalid, item 9), a `transmute_unchecked` between types of different sizes (the missing bytes read as zero), a deadlock, and a leak.
