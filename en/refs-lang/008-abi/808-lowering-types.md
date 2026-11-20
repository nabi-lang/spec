# 8.8. Lowering: data types and expressions

This section defines how Nabi values, types and expressions are represented in the code `nabic` generates. Layouts are written as Nabi types: `(A, B)` is a pair of fields at their natural offsets, `[T; n]` an array, `ptr` a pointer-width address, `p` the pointer width in bytes. Code is pseudo-code ([8.7.1](807-code-generation.md#871-translation-flow)).

## 8.8.1. Primitive integers / floating-point

Primitive numeric types are machine values of their width.

| Nabi type          | Representation                                                        |
|-------------------|-------------------------------------------------------------------------|
| `i8` ~ `i64`      | 1 ~ 8 bytes, two's complement                                           |
| `u8` ~ `u64`      | 1 ~ 8 bytes; the operation decides signedness                            |
| `usize` / `isize` | the target's pointer width                                              |
| `f32` / `f64`     | IEEE 754 binary32 / binary64                                            |
| `bool`            | one byte, `0` or `1`                                                    |
| `char`            | 4 bytes (a Unicode scalar value)                                        |
| `c_char`          | one byte                                                                |

`str` is an immutable pair `(ptr, usize)` passed by value (a pointer and a byte length). Literals point to immortal constants; strings created at runtime point to the payload of a GC object (type id `1`, [8.2.12](802-scan-table.md#8212-runtime-reserved-type-ids)).

- `s[a:b]` is `(ptr + a, b - a)` and shares the original bytes (O(1)).
- `ptr` may point to any byte of a GC object (including the trailing NUL) and keeps the whole object alive. The GC treats it as an interior pointer, and evacuation rewrites it preserving the offset.
- Literals and strings newly created by the runtime have a NUL after their bytes. Slices may not.

## 8.8.2. Arithmetic operations

On primitive types, the operators of [operator dispatch](../003-expressions/301-operator-dispatch.md) are single instructions of the operand width: add, subtract, multiply, signed or unsigned divide and remainder by the operand type, and signed, unsigned or ordered floating-point comparisons.

```
a + b       ->  add.i32 a, b            ; with overflow checks: add, then branch to the panic on overflow
a / b       ->  div.s32 a, b            ; after the checks for b == 0 and MIN / -1
a < b       ->  lt.s32 a, b             ; lt.u32 for unsigned, lt.f64 (ordered) for floats
```

A checked operation branches to its panic when the result does not fit (`-C overflow-checks`, [9.1.2](../009-tools/901-nabic-cli.md#nabic-build-aot-build)); an optimized build drops a check it proves cannot fail ([8.7.3](807-code-generation.md#873-optimization-levels)). User-defined trait implementations are function calls after generic monomorphization.

## 8.8.3. Representation of `struct`

A `struct` is a value type ([2.3.8](../002-types/203-struct.md#238-value-semantics)) and has no object header.

```nabi
struct Point { x: f64; y: f64; }
```

Its representation is its fields in the layout of [2.3.6](../002-types/203-struct.md#236-memory-layout), `#[repr]` alignment and offsets included.

```
(f64, f64)        ; x at 0, y at 8; size 16, align 8
```

Two structs with the same fields have the same layout. A struct value lives in registers, in stack slots, or inline in its container: a field of another struct, an element of a `List` buffer, a slot of a closure environment or coroutine frame. Assignment and argument passing copy the fields. The compiler may box a value internally only where the box cannot be observed.

### Field access

`p.x` reads the field at its offset: a load from the struct's slot or inline location, or the register that holds the field.

```
val = load.f64 [p + 0]
```

### Field write

`p.x = 1.0f64` updates the variable's own copy: a store to the field's offset in its slot.

```
store.f64 [p + 0], 1.0
```

When the struct is stored inline in a GC heap object and the field holds a GC reference, the store goes through the write barrier ([Barriers and roots](../007-runtime/704-barriers-and-roots.md)). Pointer fields of a struct in a stack slot are GC roots ([7.4](../007-runtime/704-barriers-and-roots.md)).

## 8.8.4. Representation of tuple `struct`

Consider the tuple `struct`:

```nabi
struct Pair(i32, i32);
```

It is represented the same as an ordinary `struct`, except that fields are accessed by positional index instead of by name.

## 8.8.5. Tuple types

The tuple type

```nabi
(i32, str, bool)
```

is laid out as `(i32, (ptr, usize), bool)`: `str` is its pair (8.8.1) wherever it appears. Like a `struct`, it is a value with no header.

## 8.8.6. Representation of `enum`

An `enum` from [`enum`](../002-types/204-enum.md) is a value with no object header, represented as a tagged `union`: the same `(i32 tag, payload)` layout as a sum type (8.8.8).

```nabi
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

It is represented as:

```
(i32, [u64; 2])    ; 64-bit target: tag at 0, payload at 8 (16 bytes, 8-aligned)
```

The tag holds the variant's index (`Ok`=0, `Err`=1, `Pending`=2). The payload is a union of the variants with data: as large and as aligned as the largest of them, each variant's payload stored at its start in its own layout. A payload that can hold a GC reference is a run of pointer-width words, which the collector scans per the tag ([8.2](802-scan-table.md), `SUM_PAYLOAD`).

A fieldless enum is its tag alone: an `i32`, or the integer of its `#[repr(..)]`.

### Variant construction

`Result::Ok(42)` builds the value in registers or in the destination's slot: no allocation.

```
store.i32 [v + 0], 0        ; tag: Ok
store.i32 [v + 8], 42       ; the payload in its own type
```

A union value built from a shorter variant's payload holds zero in the bytes past that payload, so a root word copied from it is null or a reference, never stale stack ([8.9.12](809-lowering-control.md#8912-shadow-stack-frame)).

### Variant matching

Matching variants:

```nabi
match v {
    Ok(n) => ...,
    Err(s) => ...,
    Pending => ...,
}
```

compares the tag with each arm's variant index in arm order (8.9.3); a value no arm matches reaches an unreachable block.

```
bb0:
    tag = v.0
    if tag == 0 then bb_ok else bb1
bb1:
    if tag == 1 then bb_err else bb2
bb2:
    if tag == 2 then bb_pending else bb_unreachable
bb_ok:
    n = v.1 as i32          ; the payload read as the variant's type
```

## 8.8.7. Representation of (untagged) `union`

Consider a `union` from [`union`](../002-types/205-union.md).

```nabi
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

It is as large and as aligned as its largest field (here 8 bytes), and every field starts at offset 0. A field access reads or writes the same memory as that field's type. Field reads must be inside `unsafe` in the source ([2.5](../002-types/205-union.md)).

## 8.8.8. Representation of sum types (`A | B`)

Sum types from [Sum types](../002-types/206-sum-types.md) are handled as follows.

```nabi
type X = i32 | f64
```

This is represented as `(i32, u64)`, a tag and a payload slot, built as for an `enum` (8.8.6): here the payload holds the `i32` or the `f64`. A `none` member is tag 0 and carries no payload, so zeroed memory reads back as `none`.

If niche optimization ([niche optimization](../002-types/207-niche-optimization.md)) applies, the tag slot disappears and only the payload's bit pattern remains.

```nabi
Ptr<u8> | none
```

This type is a single pointer: an automatic niche in which null means `none`.

## 8.8.9. Representation of `any` slots

Consider the dynamic slot from [`any` slot](../002-types/202-any-slot.md).

```nabi
let x: any = ...
```

It is a `(u32 tag, payload)` pair. `any Tr` and bare `any` have the same representation.

```
(u32, ptr)
```

The low 31 bits of the tag identify the value's type: a 31-bit FNV-1a hash of the type's name, so separately compiled packages agree (a collision in one build takes the next free value). Number, `bool` and `char` values no wider than a pointer are stored directly in the payload word, as its bits. Other values are boxes on the GC heap, the payload is the box, and bit 31 of the tag (`ANY_BOXED`) is set; the GC reads that bit when scanning an `any` slot ([8.2.7](802-scan-table.md)).

### Type tests on `any`

A type pattern on an `any` value

```nabi
match v {
    n: i32 => n + 1,
    s: str => s.len() as i32,
    _ => 0,
}
```

masks off `ANY_BOXED` and compares the tag with each pattern type's tag, a chain of two-way branches:

```
bb0:
    t = v.0 & 0x7fffffff
    if t == TAG_I32 then bb_i32 else bb1
bb_i32:
    n = v.1 as i32          ; the payload word's low 32 bits
bb1:
    if t == TAG_STR then bb_str else bb_other
bb_str:
    s = load.str [v.1 + 16] ; a boxed `str`: the pair from the box payload
```

These tests at run time are the cost of `any` slots.

## 8.8.10. Array types: `[T; N]`

Consider a fixed-size array from [Type expressions](../002-types/201-type-expressions.md).

```nabi
[i32; 8]
```

It is 8 `i32` elements in a row, 32 bytes. Inside a GC object, a 16-byte header precedes the array.

## 8.8.11. Function types: `fn(...) -> ...`

Consider a [function type](../004-functions/404-function-type.md).

```nabi
fn(i32, i32) -> i32
```

Every function value, a named function or a closure, is a `{code, env}` pair:

```
(ptr, ptr)        ; code, env
```

`code` takes `env` as its first argument, then the declared arguments, and no TLS address ([8.6.11](806-calling-convention.md#8611-function-pointer-abi)). A named function used as a value has a null `env` and a code pointer to a thunk that drops it. Only a C function type (`extern "C" fn(..)`) is a bare code pointer.

## 8.8.12. Representation of closures

Consider a capturing [closure](../004-functions/405-closure.md).

```nabi
let n = 10;
let f = |x| x + n;       // captures n
```

It is a function value (8.8.11) whose `env` is a GC object holding the captures, one field each, in a struct after the 16-byte header:

```
env object: header (16 bytes), then (i32)     ; a copy of n
f = (code of the closure body, env)

; on call
r = call f.0(f.1, x)
```

A closure that captures nothing has a null `env` and allocates nothing.

## 8.8.13. Identifier / variable access

Consider local variables from [Variable declarations](../005-modules/501-variable-declaration.md).

```nabi
let counter = 0i32;
counter = counter + 1i32;
```

Every local has a stack slot, and an assignment is a store to it:

```
store.i32 [counter], 0
c = load.i32 [counter]
c2 = add.i32 c, 1
store.i32 [counter], c2
```

At `-O0` every value stays in its slot. Above `-O0` a local that is never addressed, never copied whole and not described to the debugger may live in a register instead, and the optimizer forwards known values to its reads ([8.7.3](807-code-generation.md#873-optimization-levels)). A local holding a GC reference always keeps its slot: the shadow stack frame holds the slot's address (8.9.12).

## 8.8.14. Literal desugaring

Consider a collection literal from [`#[literal]` magic](../006-meta-system/605-literal-magic.md).

```nabi
[1, 2, 3]
```

It desugars to a call to the standard library function `list_literal(1, 2, 3)` and follows ordinary function call lowering.

String and numeric literals are handled as compile-time `const`s stored in .rodata and are then used as pointers or values.
