# 0.3. Intrinsic mechanism - the compiler ↔ standard library boundary

The body of a standard library trait method, such as `i32::add`, must ultimately be lowered to the machine's `add` instruction. This chapter defines the path by which standard library method bodies are connected to the compiler backend (PrimOp / MIR / code generation).

## 0.3.1. The problem - bootstrap cycle

The core of the problem is a bootstrap cycle between the standard library and the compiler. The following flow shows where it gets stuck.

```
user code:         let r = a + b;        (a, b: i32)
↓
compiler:          dispatch Add::add(a, b) call     (refs-lang 3.1)
↓
standard library:  impl Add for i32 { fn add(self, rhs) { ... } }
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
              what goes in the "..." here?
              without an intrinsic, "self + rhs" would dispatch Add::add again → infinite recursion
              a raw machine `add` cannot be written either (not part of the user language surface)
↓
machine code:      add w0, w0, w1          (aarch64)
```

A path is therefore needed by which the standard library body leads straight to the compiler backend's `add` instruction without calling itself.

## 0.3.2. The solution - the `#[intrinsic("nabi.OP.TYPE")]` attribute

The solution is to attach an attribute recognized by the compiler to the standard library method body. When the compiler encounters such a call, it converts it directly to PrimOp / MIR instead of lowering the body.

```nabi
impl Add for i32 {
    type Output = i32;

    #[intrinsic("nabi.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // an empty body is OK; the compiler uses the intrinsic path
        // even if a body exists, the compiler takes precedence
    }
}
```

Free functions (`mem::size_of`, `ptr::addr_of`) and inherent methods (`MaybeUninit::as_ptr`) carry `#[intrinsic]` the same way; a call to one is the intrinsic.

The compiler proceeds as follows. When dispatching an `Add::add(a, b)` call, it finds `i32::add` in the method_table and then checks whether `#[intrinsic("nabi.add.i32")]` is attached to that method. If so, it emits `PrimOp::Add { ty: i32 }` directly instead of lowering the body; this then passes through MIR and code generation and ultimately reaches the target's 32-bit `add` instruction.

This mechanism absorbs the compiler's interception into trait dispatch.

## 0.3.3. Intrinsic naming convention

Intrinsic names take one of three forms:

- `nabi.<OP>.<TYPE>`: a primitive operation on one type (`nabi.add.i32`), including the compound assignments `nabi.<OP>_assign.<TYPE>` (`nabi.add_assign.i32`);
- `nabi.<area>.<op>`: an operation of one area, such as `ptr` or `atomic` (`nabi.ptr.read`, `nabi.atomic.cas`);
- `nabi.<name>`: a generic operation with no type in its name (`nabi.size_of`, `nabi.align_of`, `nabi.addr_of`, `nabi.drop_in_place`, `nabi.hand_over`), including the GC buffer and hand-off families `nabi.buf_<op>` and `nabi.transfer_<op>` (`nabi.buf_get`, `nabi.transfer_keep`).

The primitive form:

```
nabi.add.i32        i32 + i32
nabi.add.i64
nabi.add.f32
nabi.add.f64
nabi.sub.i32
nabi.mul.i32
nabi.div.i32
nabi.rem.i32
nabi.and.i32        bitwise AND
nabi.or.i32
nabi.xor.i32
nabi.shl.i32
nabi.shr.i32
nabi.eq.i32
nabi.lt.i32
...
nabi.neg.i32        unary
nabi.not.bool       logical not (bool)
nabi.not.i32        bitwise not (integer)
nabi.cast.i32.i64   type cast
```

Each OP of the primitive form corresponds 1:1 to the compiler's `PrimOp` enum. The names themselves follow common instruction names and Rust's intrinsic naming conventions.

## 0.3.4. Body fallback policy

A method with an intrinsic attached may still have a body. The following example illustrates this.

```nabi
impl Add for BigInt {
    type Output = BigInt;

    // BigInt is not fixed-width, so no intrinsic; the body is authoritative
    fn add(self, rhs: BigInt) -> BigInt {
        // actual BigInt addition implementation
        ...
    }
}

impl Add for i32 {
    type Output = i32;

    #[intrinsic("nabi.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // body: runs when taken out as an fn value or in environments that do not know the intrinsic
        self + rhs                    // primitive operators lower directly to machine operations
    }
}
```

The rules are summarized as follows. On the normal compilation path with intrinsics enabled, the body is ignored; in experimental backends or interpreted environments that do not recognize the intrinsic, the body is actually used. If the body is empty, it is treated as a stub `panic("intrinsic not available")`.

When the method is taken out as an `fn` value (`let f = i32::add;`), calling that value runs the body. The intrinsic applies only at direct call sites, so the body of an intrinsic method must also be a correct implementation with the same meaning. Inside the body, `self + rhs` on a primitive is itself a direct call of this intrinsic, so it lowers to the machine operation, not back into the body.

An intrinsic may also be a declaration with no body (`pub fn size_of<T>() -> usize;`). It has no fallback: the intrinsic is its only implementation, and it is used only by direct calls.

## 0.3.5. Intrinsics in user code

`#[intrinsic(...)]` is a mechanism that operates only at the boundary where the standard library is coupled to the compiler, so user code cannot attach this attribute directly: on a function, a generic function or a method alike.

```nabi
// user code: rejected.
#[intrinsic("nabi.add.i32")]
fn my_add(a: i32, b: i32) -> i32 { ... }
// error[E0115]: `#[intrinsic]` on `my_add`: the intrinsic attribute is for the standard library only
```

This restriction prevents opening an escape hatch through which the compiler backend could be called arbitrarily from outside the standard library traits.

## 0.3.6. Comparison with other mechanisms

The intrinsic is only one of several magics that connect the compiler and the standard library; the following table summarizes the role of each.

| Mechanism             | Purpose                                                          | Spec                                                                   |
|-----------------------|------------------------------------------------------------------|------------------------------------------------------------------------|
| `#[intrinsic("...")]` | Standard library method ↔ compiler backend PrimOp                | This document                                                          |
| `#[lang(...)]`        | Standard library item ↔ compiler slot (none, iter, index, etc.)  | [refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)     |
| `#[infix(...)]`       | Token ↔ standard library trait method                            | [refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md) |
| `#[literal(...)]`     | Literal form ↔ standard library handler                          | [refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md)  |

The four magics are mutually orthogonal. `#[intrinsic]` handles coupling with the compiler backend, while all the others handle coupling with the language syntax.
