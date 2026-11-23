# 5.2. `const`

The `const` keyword (a contextual keyword that is a keyword only at the start of an item, [1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)) declares a value determined at compile time. The grammar is:

```
const_decl      ::= "const" identifier ":" type_expr "=" const_expr ";"
const_fn        ::= "const" fn_decl
const_block     ::= "const" block
const_expr      ::= expr                        // evaluable at compile time (5.2.4)
```

## 5.2.1. Basic form

```nabi
const PI: f64 = 3.14159f64;
const MAX_RETRY: i32 = 3i32;
const NAME: str = "nabi";
```

A declaration consists of the `const` keyword, a name, `:`, a type, `=`, and a `const` expression, in that order. The type must be written explicitly; unlike `let`, it is not inferred. The value position accepts only a `const` expression, that is, an expression that can be evaluated at compile time.

## 5.2.2. Permitted positions

`const` may be declared in all of the following positions.

```nabi
// module level
const PI: f64 = 3.14159f64;

// inside a function body
fn area(r: f64) -> f64 {
    const FACTOR: f64 = 3.14159f64;
    FACTOR * r * r
}

// inside an impl block
impl Circle {
    const UNIT_AREA: f64 = 3.14159f64;
    fn unit() -> Circle { Circle { r: 1.0f64 } }
}

// inside a trait: associated const
trait Numbers {
    const ZERO: Self;
    const ONE: Self;
}
```

A `const` in a function body is an item of its block ([3.9.1](../003-expressions/309-block-expressions.md#391-form)): visible in the whole block and the items declared in it. It reads no local variable or parameter of the code around it; the error says to write a `let` instead (`let half = n / 2;`). It may read the const parameters of the generic function or `impl` it is in, and the associated consts and `T::size()` of its type parameters: then it has a value per instance (5.2.7).

```nabi
fn mid<const N: usize>(xs: [i32; N]) -> i32 {
    const HALF: usize = N / 2;      // one value per instance
    xs[HALF]
}
```

## 5.2.3. Meaning of `const`: inlined at each use

A `const` is a value, not an address. Its value is inlined at every place it is used.

```nabi
const X: i32 = 42i32;

fn f() {
    let a = X;                    // a = 42i32 (X is inlined)
    let b = X + 1i32;              // b = 43i32 (X is inlined, then evaluated)
}
```

The result of a `const` does not reside at any memory location, so its address cannot be taken.

```nabi
let p = &X;                       // ERROR (E0012): Nabi has no `&` borrows
```

Nabi has no `&` operator, so no expression takes the address of a `const`. Raw pointers for [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) are defined by the pointer types in [type expressions](../002-types/201-type-expressions.md).

## 5.2.4. Constant expressions and the evaluator

`const` items, array lengths, const generic arguments, `struct` field defaults, `static` initializers, and `const fn` literal handlers receiving constant arguments ([6.5.8](../006-meta-system/605-literal-magic.md#658-const-compatibility-of-literal-handlers)) all use the same compile-time evaluator.

What can be evaluated:

- Values: integers of every width, floating-point, `bool`, `char`, `str`, tuples, `struct`, `enum`, sum types, fixed-size arrays.
- Expressions: literals, references to other `const`s, const generic parameters, operators, `const fn` calls.
- Control flow: `if`, `match`, `loop` / `while` / `for`, labeled `break`, `return`, `throw`. Recursion and generic `const fn` are allowed.

What cannot be evaluated: calls to functions that are not `const fn` (I/O, time, random numbers, etc.), reading a `static`, references to outer `let` variables.

The following are compile errors:

- Integer overflow (including the result of folding an expression made only of literals and constants, `let m: u8 = 2 * 200`), division by zero, `MIN / -1`, shifts by the bit width or more, and out-of-range floating-point → integer conversions. Wrapping arithmetic is only via `wrapping_*`.
  Code that a constant condition rules out never runs and is not checked: in a generic instance with `N = 0`, the `N - 1` of `if N == 0 { 0 } else { a[N - 1] }`, of the rest of a block after `if N == 0 { return ..; }`, and of `N > 0 && a[N - 1] == x` is no error. An overflow in an instance is reported with the call that made the instance and its const arguments.
- A panic or an uncaught `throw` during evaluation.
- Exceeding the limit on evaluation steps or call depth. The limits are implementation-defined (nabi: 4,000,000 steps, depth 128).
- Circular references between `const`s.

## 5.2.5. `const fn`: functions evaluable at compile time

```nabi
const fn double(x: i32) -> i32 {
    x * 2i32
}

const fn gcd(a: i64, b: i64) -> i64 {
    if b == 0i64 { a } else { gcd(b, a % b) }
}

const TWO: i32 = double(1i32);                  // compile time
const TWELVE: i64 = gcd(24i64, 36i64);          // compile time

let y = double(user_input);                      // runtime call
```

- A `const fn` may call only `const fn`s. Calls through `fn` values or closures inside a `const fn` are checked only when evaluated, since a static check cannot follow them. The standard library marks pure functions as `const fn`.
- The body must be pure. I/O and access to mutable globals are errors.
- A result successfully evaluated at compile time equals the runtime result for the same arguments. An operation that would overflow at runtime is an error at compile time (5.2.4).
- A `const fn` cannot take a type as an argument (`const fn sizeof(T: type)` is an error; the help writes `const fn sizeof<T>() -> usize { T::size() }`). Type introspection belongs to meta blocks.
- A `const fn` can also be called at runtime like an ordinary function.

### Generic `const fn`

```nabi
const fn id<T>(x: T) -> T { x }

const A: i32 = id(42i32);                        // evaluated after monomorphization with T = i32
```

### Results that use the heap

If the result contains a heap allocation (`List`, buffers, closures, etc.), it is not embedded in the binary. Such a `const` is evaluated and checked at compile time and constructed at runtime at each use.

Each use is a fresh value, so a mutating call on a `const` (`XS.push(4)` on a `List`, or a `ref self` method on a `const` struct) would change a copy that is dropped. It is a compile error: copy the value into a variable and change that (`let xs = XS; xs.push(4);`).

## 5.2.6. Orthogonality of `const` and other mechanisms

```
generic <T>     = type parameterization (call-site specialize)
meta block      = code generation (impl, struct, etc. at AST level)
const fn        = value computation (compile-time pure function)
```

The three mechanisms are orthogonal. Their domains do not overlap, so they may be combined freely.

## 5.2.7. When `const` is evaluated

Module-level, function-body and `impl`-level `const`s are evaluated once at package build. A `const` that depends on generic parameters (an associated `const` of a generic `impl`, a function-body `const` or a `const { .. }` block that reads a const parameter) is evaluated per monomorphization. Results that do not use the heap are embedded in the binary and have no runtime cost.

## 5.2.8. Relation to `struct` field defaults

[`struct`](../002-types/203-struct.md) field defaults are constant expressions (5.2.4).

## 5.2.9. Relation to `const` generics

Arguments of [const generics](../002-types/211-const-generics.md) are constants that become part of the type, and are evaluated by the same evaluator (5.2.4).

## 5.2.10. `const` blocks

`const { .. }` is an expression evaluated at compile time by the evaluator (5.2.4). Its value is a constant of the block's type; a block of type `()` is a check.

```nabi
fn chunk<const N: usize>(xs: [i32; N]) -> usize {
    const { assert(N > 0, "chunk needs at least one element") };
    const { N / 2 }
}
```

- It reads what a function-body `const` reads: constants, `const fn` calls, the const parameters of the code around it, and the associated consts and `T::size()` / `T::align()` of its type parameters. A `let` variable is an error, whose help drops the `const`.
- In a generic function it is evaluated once per instance, after the instance's arguments are known. The check of the generic body at its definition ([2.10.9](../002-types/210-generics.md#2109-body-checking)) type-checks the block but does not evaluate it.
- A panic or failed `assert` while evaluating it is a compile error at the call that made the instance, naming the instance (`` const block failed in `chunk::<0>`: chunk needs at least one element ``), with the `assert` as a secondary location. `nabi check` reports it too: it instantiates what the program reaches, as `nabi build` does.
- A passing block costs nothing at run time; its value is embedded like any `const`.
- `T::size()` is known to the evaluator for a plain type (a number, `bool`, `char`, `str`, `()` or an array of them); for other types it is an error that says to use a const parameter or a `List`.
