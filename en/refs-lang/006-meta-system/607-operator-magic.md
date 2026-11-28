# 6.7. Operator magic

The three attributes `#[infix]`, `#[prefix]`, and `#[postfix]` only connect operator tokens to trait methods. Precedence and associativity are fixed at the language level by the table in [3.2.3](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table) and cannot be set or changed by these attributes. They are core magics required for the desugaring mechanism of [operator dispatch](../003-expressions/301-operator-dispatch.md) to work.

```
infix_attr      ::= "#" "[" "infix" "(" string_lit ("," like_arg)? ("," "swap")? ")" "]"
prefix_attr     ::= "#" "[" "prefix" "(" string_lit ("," like_arg)? ")" "]"
postfix_attr    ::= "#" "[" "postfix" "(" string_lit ("," like_arg)? ")" "]"
like_arg        ::= "like" "=" string_lit
```

## 6.7.1. The three attributes

First, an example of how the three attributes are used.

```nabi
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}

trait Degrees {
    #[postfix("°", like = "?")]    // extension token, same level as postfix ?
    fn degrees(self) -> f64;
}
```

Where each attribute is attached and which form of operation it expresses:

| Attribute             | Position      | Form                     |
|-----------------------|---------------|--------------------------|
| `#[infix(sym)]`       | trait method  | binary (`a SYM b`)       |
| `#[prefix(sym)]`      | trait method  | prefix unary (`SYM a`)   |
| `#[postfix(sym)]`     | trait method  | postfix unary (`a SYM`)  |

## 6.7.2. Arguments

The arguments each attribute takes:

| Argument | Meaning                                                            | Required?                                    |
|----------|--------------------------------------------------------------------|----------------------------------------------|
| `sym`    | Operator token (string literal)                                    | Required                                     |
| `like`   | Existing token the new token follows (its level and associativity) | Required, only for tokens not in the 3.2 table |
| `swap`   | Connects with operands swapped (infix only)                        | Optional                                     |

`sym` and `like` are string literals like any other: the token is the string's value, so `"\\"` and `r"\"` both name the one-character token `\` (`a \ b`).

`swap` connects a binary operation with its two operands swapped. `contains` with `#[infix("∈", swap)]` desugars `a ∈ b` into `b.contains(a)`. It exists for operators whose container is not on the left (`∈ ∉`).

`like` cannot be written for tokens in the 3.2 table. There are no arguments specifying precedence or associativity (`prec`, `left` / `right` / `none`, etc.); writing one is a compile error.

```nabi
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]      // new token, same level and associativity as +
```

## 6.7.3. Connection mechanism

How the compiler behaves when `#[infix("+")]` is attached to the trait method `add`:

First, the `+` token is bound to that trait as its connection point. The compiler then desugars the expression `a + b` into `<typeof(a) as the_trait<typeof(b)>>::add(a, b)`. Precedence and associativity are the values fixed for `+` in the table of [precedence and associativity](../003-expressions/302-precedence-and-associativity.md) (9, left-associative).

All three attributes follow the same pattern; what is bound is only the token, the position, and the trait method.

The operands are typed as the arguments of that call. A literal right operand (`[1.0, 2.0]`, `{..}`, a number) takes the parameter type of the impl chosen for the left operand, so `t ⊲ [1.0, 2.0]` passes a `Vector<f64>` when that impl's method takes one. A closure operand meets an impl on its `fn` type, as it does a bound ([4.4.2](../004-functions/404-function-type.md#442-types-of-function-and-closure-values)), and a blanket `impl<F: Tr> Op for F` applies to it. For a trait with no type parameters, the method's own parameter type is the right operand's.

A method that takes `ref self` takes its left (or only) operand as a method call takes its receiver: when the operand is a variable or a field of one, the method's change is written back to it, so `g ∼ d` with `fn draw(ref self, d: D)` advances `g` as `g.draw(d)` does. A value-captured variable in a closure is rejected as for a method call ([4.6.7](../004-functions/406-capture-semantics.md#467-capturing-structs-and-reference-handles)).

Operators do not throw ([4.14](../004-functions/414-throws.md)), so the method must be declared without `throws`; an operator attribute on a throwing method is an error at the declaration. An operation that can fail returns its error as a value, `Result<T, E>` or `T | E`, and the caller handles it (`try (a ⧵ b).get()`, a `match`), or it stays a named method without the attribute (`try a.solve(b)`).

```nabi
trait Solve<Rhs = Self> {
    type Output;
    #[infix("⧵", like = "/")]     // ERROR: `solve` throws; operators do not throw
    fn solve(self, rhs: Rhs) throws (str) -> Output;
}
```

## 6.7.4. Unicode and multiple aliases

Attaching several `#[infix]` to the same method maps several tokens to that method.

```nabi
trait Join<Rhs = Self> {
    type Output;
    #[infix("⊕", like = "+")]
    #[infix("⊞", like = "+")]
    fn join(self, rhs: Rhs) -> Output;
}
```

Both tokens call `join`. `÷ ≡ ≠ ≤ ≥` need no alias: the lexer reads them as `/ == != <= >=` ([1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-fixed-token-table)), so they reach the same methods as the ASCII tokens. Implementing the ASCII token's trait (`PartialOrd` for `<=`) makes both spellings work.

Registering one of these spellings is a compile error: the lexer never produces the token `≤`, so a binding to it could never fire. The error names the ASCII token and the trait to implement. A dot before a Unicode operator (`.≤`, `.×`) is not a token either: the lexer reads `.` and the operator apart, and only the ASCII operators of 1.10.3 have a broadcast form.

```nabi
trait Before {
    #[infix("≤")]     // ERROR: `≤` is the fixed operator `<=`; implement `ops::PartialOrd`
    fn before(self, rhs: Self) -> bool;
}
```

`->` goes the other way. Between two values it is the ASCII spelling of `→`: `a -> b` calls the method registered for `→`, at its level, so `bag -> g` and `bag → g` are one expression. A return type keeps `->` (`fn f() -> T`). With no `→` in scope, `->` between two values is a syntax error that says so.

## 6.7.5. Conflict checking

If different traits are attached to the same (token, position) combination, it is an ERROR at registration.

```nabi
trait MyAdd { #[infix("+")] fn add(...) ... }
// another library / module.
trait Add { #[infix("+")] fn add(...) ... }
// → two traits for the binary '+' position. ERROR.
```

Each (token, position) combination admits exactly one registration.

## 6.7.6. Multiple positions of the same token

A token usable in both prefix and binary positions, such as `-`, is treated as separate registrations.

```nabi
trait Sub<Rhs = Self> {
    type Output;
    #[infix("-")]
    fn sub(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}
```

For example, in `-a + b` the prefix `-` binds tighter than the binary `+`, so it is parsed as `(-a) + b`.

Binary `-` and prefix `-` are separate mappings, so attaching them to different traits is no problem.

## 6.7.7. Dot-prefixed (broadcast) mappings

The dot-prefixed tokens of [broadcast operators](../003-expressions/303-broadcast-operators.md) are registered through the same magic.

```nabi
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+")]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*")]
    fn dot_mul(self, rhs: Rhs) -> Output;
}
```

Only the token differs; the registration mechanism is exactly the same.

## 6.7.8. Visibility of mappings

A registration through attributes such as `#[infix]` takes effect only where the trait is visible. Visible means accessible under [5.6](../005-modules/506-visibility.md): a `pub` trait reachable by path suffices; no `use` is needed. Where the trait is not visible, the operator does not work either.

This rule fits library isolation: a user-defined operator trait defined in a package works only in code that imports that package. The tokens of a `pub` operator trait reach the same code: the lexer cuts them in the files of every package that depends on the registering one, so `<~` or `<-` from a dependency lexes there as it does in its own package ([1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-extension-tokens)). A `#[literal]` registration follows the same rule for a `pub` handler, in the packages that depend on its package directly ([6.5.2](605-literal-magic.md#652-form)); any other stays in its package.

The standard library's operator traits (`Add`, `Sub`, `Mul`, ...) live in `core::ops`. An operator needs no trait name in scope: the mappings of `core` traits apply in every source, so arithmetic and comparison expressions work without a separate import. Naming the trait itself, as in `impl ops::Add for V`, goes through the prelude module `ops` or a `use` ([stdlib 0.2.3](../../refs-standard-libraries/000-meta/002-import.md)).

## 6.7.9. User-defined operators

Users can also define operator mappings by attaching attributes such as `#[infix]` to their own traits.

```nabi
trait CustomCombine<Rhs = Self> {
    type Output;
    #[infix("⊕")]
    fn combine(self, rhs: Rhs) -> Output;
}

impl CustomCombine for MyType { ... }

let r = a ⊕ b;            // user token
```

A token not in the 3.2 table, such as `⊕`, names one existing token with `like = "+"` and follows that token's level and associativity. New levels cannot be created. The token must satisfy the extended-token conditions of [punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md#1104-extension-tokens): every character must be Unicode `Pattern_Syntax`, and it is an error if it equals an existing fixed token.

A token may start like a fixed one: `#[infix("<-", like = "+")]` makes `c <- 3` one operator, since the lexer takes the longest match. Such a token, one that starts with a character a fixed token starts with, works in the files of the package that registers it and, from a `pub` trait, in the packages that depend on it; elsewhere `a<-1` stays `a < -1` ([1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-extension-tokens)).

Binary `@` may be bound too, with `like`: `#[infix("@", like = "*")]` makes `a @ b` a matrix product, as in NumPy. `@` stays the fixed token, and the grammar has no other binary `@`: the language writes it only where no operand ends before it, ahead of meta syntax (`@derive`, `@Self`), after a binding name in a pattern (`n @ (a, b)`) and glued to `return` (`return@f`). An `@` after a complete operand is therefore the operator wherever its trait is visible, and those other uses keep their meaning in the same file. `@` cannot be bound as a prefix or postfix operator, since those positions are meta syntax.

Binary `~` is free the same way: the language reads `~` only as the prefix bitwise not (`~x`), never after an operand, so `#[infix("~", like = "==")]` makes `x ~ d` ("distributed as", or a model formula `y ~ x`) an operator, and `~x` keeps its meaning. Any other fixed token can be bound only where the 3.2 table gives it a form: postfix `%` is an error that points to a number suffix (`95pct`, [6.5.7](605-literal-magic.md#657-user-defined-literals)), and `->` or `|>` are syntax. Postfix `!` (a factorial `n!`) is an error too: after a name `!` starts a macro call (`name!(..)`), and before an operand it is `not`; write a function or a method (`factorial(n)`, `n.factorial()`).

A token holds no identifier character: `′` (`f′` is a name) and `ℱ` (a letter) cannot be operators. Write a method (`a.t()`) or a function (`fourier(x)`), or pick a symbol such as `†` ([1.10.4](../001-lexical/110-punctuation-and-operator-tokens.md#1104-extension-tokens)).

## 6.7.10. Precedence and associativity levels

The single authoritative location for operator levels is the [3.2.3 precedence table](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table). Neither the standard library nor user code can set or change levels; a new token attaches to one of the existing levels via `like`.

## 6.7.11. When it is evaluated

Registrations such as `#[infix]` are processed in the meta-expansion phase right after parsing. All subsequent expression parsing consults the registration table.

The order of registration and use within the same compilation unit therefore does not matter, because the scanner pre-pass collects them in advance. The precedence of tokens in the 3.2 table is fixed from the start, independent of registration.

## 6.7.12. No aliases at different levels

All aliases attached to the same trait method must be tokens of the same level.

```nabi
trait Before {
    #[infix("<")]
    #[infix("⊲", like = "+")]    // ERROR: < is level 4, + is level 9
    fn lt(self, rhs: Self) -> bool;
}
```

This restriction stems from the assumption that aliases share the same meaning.

## 6.7.13. Attachment only to trait methods

`#[infix]`, `#[prefix]`, and `#[postfix]` can be attached only to method signatures of a trait. They cannot be attached to free functions, `impl` methods, or ordinary methods outside a trait.

```nabi
fn my_add(a: i32, b: i32) -> i32 { a + b }
#[infix("+++")]                              // ERROR: cannot attach to a free function
fn my_concat(a: str, b: str) -> str { ... }
```

The reason is to bind operator mappings to the trait system and guarantee consistent dispatch.

## 6.7.14. Calling a value: `ops::Call`

A call `v(a)` whose callee is a value, not a function, of a type that implements `core::ops::Call` ([stdlib 2.3](../../refs-standard-libraries/200-core/203-ops.md)) is `v.call(a)`. The arguments arrive as one value: a single argument as itself, several as a tuple (`p(x, y)` is `p.call((x, y))`) and none as `()`. So an interpolant, a polynomial or a distribution's density reads like the function it is: `s(1.5)`, `xs |> s`, and `s.(xs)` applies it to each element as for a `fn`.

```nabi
struct Spline { a: f64; }

impl ops::Call<f64> for Spline {
    type Output = f64;
    fn call(self, x: f64) -> f64 { self.a * x }
}

let s = Spline { a: 2.0 };
s(1.5);                          // 3: s.call(1.5)
```

Several impls with different `Args` pick by the argument, as other overloaded trait methods do ([3.1.3](../003-expressions/301-operator-dispatch.md#313-dispatch-without-a-vtable)): a concrete `Call<f64>` beside a generic `impl<const N: usize> Call<Array<f64, N>>` gives `s(1.5)` and `s(xs)` for an array of any rank, `s(1)` takes the `f64` impl and `s([1.0, 2.0])` the rank-1 one. A field holding a callable value is called as `(r.f)(x)`; `r.f(x)` names a method ([4.9.6](../004-functions/409-method.md#496-call-forms-of-free-functions-and-methods)). A bound `F: Call<f64, Output = f64>` calls `f(x)` through it. Arguments are positional only. `fn` values and closures are called directly and implement nothing. Calling a value of any other type is an error whose help shows the impl to write.

## 6.7.15. Summary

Operator magic, together with `#[lang]`, is the core mechanism that implements the binding of language tokens to library traits in the [boundary between language and standard library](../000-meta/004-language-vs-standard-libraries.md).

The standard library's operator specification determines the meaning of every token through this magic.
