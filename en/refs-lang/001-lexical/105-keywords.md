# 1.5. Keywords

Keywords are tokens that look like identifiers but are reserved by the language. When a keyword appears in identifier position, the compiler produces a keyword token rather than an identifier.

This document defines only the list of keywords the language reserves directly. The exact meaning of each keyword is introduced in later parts of the specification; this document states only their form and classification.

## 1.5.1. Reserved keyword list

There are two classes of keywords. The principle is the following single rule.

> [!IMPORTANT]
> A word that starts a statement or expression and could be confused with "identifier + what follows" is a **reserved keyword**; a word whose position of appearance is fixed is a **contextual keyword** (1.5.4).

A reserved keyword is tokenized as a keyword wherever it appears and cannot be used as an identifier. The only exception is the name positions of 1.5.5.

```
let       fn        if        else      match     while     loop      for
return    break     continue  try       throw
async     await     yield     unsafe    using
struct    enum      trait     impl      pub       mod       use
as        is        ref
true      false     none      self      Self
macro     defer     dyn       move      gen       do
```

> [!NOTE]
> `try?` and `try!` are not separate keywords but token sequences of the keyword `try` and the punctuation `?` / `!` ([Punctuation and operator tokens](110-punctuation-and-operator-tokens.md#dot--question-mark)).

`self` (lowercase) refers to the instance identifier (the method receiver), and `Self` (uppercase) refers to the name of the type that the current `impl` targets. `using` is the keyword introducing an RAII-style resource management block ([`using` expression](../003-expressions/317-using-expression.md)).

## 1.5.2. Meaning is defined later

The meanings of the tokens above are defined across the specification: variables, visibility and modules in Chapter 5; types and containers in Chapter 2; control flow in Chapter 3; functions, closures and coroutines in Chapter 4; meta and magic in Chapter 6.

This Chapter 1 covers only the classification rule that, at the lexical stage, these tokens are separated as keywords rather than identifiers.

## 1.5.3. Registered keywords (extension mechanism)

Beyond the reserved list above, there is a mechanism by which the standard libraries can register additional keywords. It is part of the meta-system. The registration mechanism and exact processing rules are defined in [`#[keyword(...)]` magic](../006-meta-system/606-keyword-magic.md).

Within the scope of this document, only the following is promised. Reserved keywords are not identifiers, except in the name positions of 1.5.5, and are separated as keywords. A registered keyword takes effect only within the compilation unit that contains its registration. If the two sets collide, the registration is rejected; reserved keywords cannot be overridden.

## 1.5.4. Contextual keywords

The following words are keywords only in designated positions and are ordinary identifiers elsewhere.

| Word                               | Position where it is a keyword                                         | Definition |
|------------------------------------|------------------------------------------------------------------------|------|
| `type`                             | At item start, when followed by an identifier and then `=` or `<` (type alias) | [2.6](../002-types/206-sum-types.md) |
| `any`                              | Type position (`any`, `any Tr`)                                        | [2.2](../002-types/202-any-slot.md) |
| `some`                             | A function's return or parameter type, or a type alias's right side, before a bound (`-> some Tr`, `x: some Tr`, `type X = some Tr;`) | [4.1.13](../004-functions/401-function-declaration.md#4113-opaque-return-types-some-trait), [4.1.15](../004-functions/401-function-declaration.md#4115-named-opaque-types-type-x--some-trait) |
| `in`                               | The `in` of `for pat in`; in an expression, the membership operator `in` / `!in` (`∈` / `∉`) | [3.13.3](../003-expressions/313-loop-expressions.md#3133-for-iterator-based-repetition), [3.2.3](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table) |
| `where`                            | After a function signature, an `impl` header, or a `struct` / `enum` / `union` / type alias header | [2.10.4](../002-types/210-generics.md#2104-where-clauses) |
| `throws`, `rethrows`               | After the parameter `)` of a function signature                        | [4.14](../004-functions/414-throws.md) |
| `catch`, `finally`                 | After an expression or block (postfix clause)                          | [3.14](../003-expressions/314-error-control-flow.md) |
| `const`                            | At item start, in `impl` / `trait` bodies too (a `const` item, `const fn`), and in a generic parameter list (`<const N: usize>`) | [5.2](../005-modules/502-const.md), [2.11](../002-types/211-const-generics.md) |
| `static`, `extern`, `mut`          | At item start (including `extern "C"`, `static mut`)                   | [5.3](../005-modules/503-static.md), [5.9](../005-modules/509-extern-block.md) |
| `union`                            | At item start, when immediately followed by an identifier              | [2.5](../002-types/205-union.md) |
| `pkg`, `sub`                       | Immediately after `pub(`                                               | [5.6](../005-modules/506-visibility.md) |
| `pkg`, `sup`                       | As the first token of a path, when immediately followed by `::`        | [5.7](../005-modules/507-paths.md) |

```nabi
let type = 1;                       // identifier
type Pair = (i32, i32);             // type alias (item start + identifier + =)
let in = 3;  for x in xs { }        // identifier / the in of for
fn catch(e: str) { }                // identifier: not after an expression
```

Declaring a type named `any` is a compile error (`struct any { }`), because it could not be distinguished from `any` in type position.

### Visibility qualifiers: `pkg` / `sub`

`pkg` and `sub` are used as qualifiers inside the parentheses of `pub(pkg)` and `pub(sub)`, and are ordinary identifiers elsewhere.

```nabi
pub(sub) fn helper() { ... }   // inside pub( ): visibility qualifier
trait Sub { fn sub(self, rhs: Self) -> Self; }  // in ordinary position: identifier
```

### Path roots: `pkg` / `sup`

These two words are always path roots when they are the first token of a path and are immediately followed by `::` ([5.7 Paths](../005-modules/507-paths.md)). Elsewhere they are ordinary identifiers.

- `pkg::xxx`: from the current package root
- `sup::xxx`: from the parent module

The standard library tiers `std`, `core` and `bare` ([0.5 Standard library tiers](../000-meta/005-standard-libraries-tiers.md)) and the word `nabi` are not keywords: a local name may take them ([5.7.4](../005-modules/507-paths.md#574-std-core-bare-the-standard-library-tiers)).

Because `pkg::name` is always interpreted as a root, using `pkg` or `sup` as the name of a module or type (`struct`, `enum`, `union`, `trait`, type alias, `mod`) is a compile error. They may be used as names of variables, functions, fields and methods.

```nabi
let pkg = 42;                   // variable name OK
fn sup() { }                    // function name OK
self.pkg                        // field / member name OK
mod sup { ... }                 // ERROR: cannot be a module name
struct pkg { }                  // ERROR: cannot be a type name
::core::iter::Iterator     // the library past a local `core`
```

## 1.5.5. Keywords in name positions

In the following positions, all keywords, including reserved keywords, are used as identifiers, because the surrounding tokens make it clear that a name is meant.

- Immediately after `.` (member access, method call): `x.type`, `stream.match(p)`
- `struct` / `enum` field name declarations: `struct Token { type: Kind; }`
- Field names in `struct` literals and `struct` patterns: `Ev { match: 1 }`, `Ev { match: m } => ...`
- Names of named arguments: `f(type: 1)`. A named argument names a parameter, and a parameter name cannot be a reserved keyword, so only contextual keywords appear here in practice.

`self` is an exception and cannot be an identifier even in these positions. Elsewhere a keyword is used as a name by writing it as a raw identifier, `r#match` ([1.4.4](104-identifiers.md#raw-identifiers)).

```nabi
struct Token { type: Kind; for: str; }   // OK: field name
let k = tok.type;                         // OK: after .
make(type: Kind::Ident);                  // OK: named argument (contextual keyword)
struct S { self: i32; }                   // ERROR: self is the exception
```

## 1.5.6. Keywords reserved for later use

The last row of 1.5.1, `macro`, `defer`, `dyn`, `move`, `gen` and `do`, are reserved keywords that no syntax uses yet, except `move` in a closure's capture list (`[move x]`, [4.6.11](../004-functions/406-capture-semantics.md#4611-move-capture-the-move-x-capture-list)). They are reserved so that later releases can give them syntax without breaking code, and so that a form carried over from another language gets an error that says what Nabi writes instead:

| Keyword | Reserved for | What Nabi writes today |
|---------|--------------|------------------------|
| `macro` | a macro item | a meta block's `@macro` ([6.8](../006-meta-system/608-meta-blocks.md)) |
| `defer` | a statement run when the scope ends | `using` ([3.17](../003-expressions/317-using-expression.md)) or `finally` ([3.14](../003-expressions/314-error-control-flow.md)) |
| `dyn`   | nothing: kept so Rust's `dyn Trait` meets an error | `any Trait` ([2.2](../002-types/202-any-slot.md), [2.12](../002-types/212-trait-as-type-banned.md)); `dyn Trait`, `&dyn Trait` and `Box<dyn Trait>` are compile errors whose help writes `any Trait` |
| `move`  | outside a capture list, nothing: kept so Rust's `move \|x\| ..` meets an error | a `[move x]` capture list ([4.6.11](../004-functions/406-capture-semantics.md#4611-move-capture-the-move-x-capture-list)); `move \|..\|` is a compile error that says so |
| `gen`   | a generator form | `yield fn`, `yield { .. }`, `yield \|\| ..` ([4.11](../004-functions/411-generators.md)) |
| `do`    | a loop that tests its condition after the body | `loop { ..; if !cond { break } }` ([3.13](../003-expressions/313-loop-expressions.md)) |

Like every reserved keyword they cannot be names outside the name positions of 1.5.5; an identifier spelled as one is a compile error whose help names the raw identifier ([1.4.4](104-identifiers.md#raw-identifiers)).

```nabi
fn gen(n: i32) -> i32 { n }     // ERROR: `gen` is a reserved keyword, not a name
fn r#gen(n: i32) -> i32 { n }   // OK
let s = Step { move: 1 };       // OK: a field name (1.5.5)
let m = s.move;                 // OK: after .
```

The `@macro` of a meta block ([6.8](../006-meta-system/608-meta-blocks.md)) is meta syntax, not a use of the keyword.
