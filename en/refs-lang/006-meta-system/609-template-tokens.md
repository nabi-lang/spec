# 6.9. Template tokens and expansion

This section defines the full list of template tokens usable in the body of a [meta block](608-meta-blocks.md), and when and by what mechanism meta blocks are expanded.

## 6.9.1. Form of template tokens

Template tokens have the following syntactic forms.

```
template_token  ::= "@" identifier
                  | "@" identifier "." identifier
                  | "@" identifier "{" <template body> "}"
                  | "@" identifier "(" <args> ")" "{" <body> "}"
```

Every template token starts with the `@` token ([punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md)). Representative examples:

```
@Self                   target type name
@field                  placeholder for the current field
@field.name             name of the current field
@for_fields { ... }     field iteration block
```

### Token list

| Token                                                    | Value                                                  | Section |
|----------------------------------------------------------|--------------------------------------------------------|---------|
| `@Self`                                                  | Target type (`Name<T, ..>`)                            | 6.9.2 |
| `@Self.name`                                             | Target type name without type arguments                | 6.9.2 |
| `@Self.is_unit` / `.is_tuple` / `.is_named`              | Struct shape (boolean)                                 | 6.9.2 |
| `@Self.field_count`, `@Self.variant_count`               | Number of fields of a struct / union, variants of an `enum` (integer literal) | 6.9.2 |
| `@if_struct` / `@if_enum` / `@if_union { .. }`           | Body only when the target kind matches                 | 6.9.2 |
| `@for_fields { .. }`                                     | Body per field                                         | 6.9.3 |
| `@field`, `@field.name`, `@field.type`                   | Current field (access / name / type)                   | 6.9.3 |
| `@field.is_pub`, `@field.has_attr(x)`                    | Boolean                                                | 6.9.3 |
| `@field.has_default`, `@field.default`                   | Whether the field has `= expr` (boolean) / that expression | 6.9.3 |
| `@field.index`, `@field.binding`                         | The field's position (integer literal) / a local name for its value | 6.9.3 |
| `@Self.pattern`                                          | The struct built from the `@field.binding` locals       | 6.9.3 |
| `@for_variants { .. }`                                   | Body per `enum` variant                                | 6.9.4 |
| `@variant.name`, `@variant.pattern`                      | Current variant (name / matching pattern)              | 6.9.4 |
| `@variant.index`                                         | Current variant's position (integer literal)           | 6.9.4 |
| `@variant.is_unit` / `.is_tuple` / `.is_named`           | Boolean                                                | 6.9.4 |
| `@for_variant_fields { .. }`                             | Body per field of the current variant                  | 6.9.4 |
| `@apply_if(cond) { .. }`                                 | Body only when `cond` is true                          | 6.9.5 |
| `@accept(kind, ..)`                                      | Restricts the target kind (attribute only)             | 6.9.5 |
| `@where_all(Trait)`                                      | `T: Trait` for each type parameter                     | 6.9.5 |
| `@has_derive_attr(a)`, `@derive_attr(a)`                 | The field's derive attribute (boolean / argument)      | 6.9.6 |
| `@has_derive_attr(a, k)`, `@derive_attr(a, k)`           | Key `k` of the field's `#[a(..)]` (boolean / value)    | 6.9.6 |
| `@Self.has_derive_key(k)`                                | The type's `#[attr(..)]` lists key `k` (boolean)       | 6.9.6 |
| `@item`, `@item.name` / `.params` / `.return_type` / `.body` | The item attached to and its parts (attribute only) | 6.9.7 |
| `@attribute`                                             | The attribute arguments at the use site (attribute only) | 6.9.7 |
| `<name token>.as_str()`                                  | String literal of the name                             | 6.9.3 |
| `@use_site(path)`                                        | `path`, looked up where the meta block is used         | 6.9.9 |
| `@concat(a, @Self.name, ..)`                             | One identifier joined from names, index tokens and strings | 6.9.8 |

An `@name` not in the list is an error. `@where_for_fields` is a reserved name; using it is also an error.

## 6.9.2. Type context

### `@Self` target type name

`@Self` denotes the name of the target type the meta block is applied to.

```nabi
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {                 // @Self → Point / the user's type name
        ...
    }
}
```

### `@if_struct { ... }` / `@if_enum { ... }` / `@if_union { ... }`

These tokens generate the enclosed code only when the target type is a `struct`, `enum`, or `union`, respectively.

```nabi
@if_struct {
    fn struct_only_method() { ... }
}

@if_enum {
    fn enum_only_method() { ... }
}
```

If the target type is of another kind, the block is ignored and no code is generated.

### `@Self.name` / `@Self.is_unit` / `.is_tuple` / `.is_named`

`@Self.name` is the target type name without type arguments (`Pair` for `Pair<T>`). `@Self.is_unit` / `@Self.is_tuple` / `@Self.is_named` are booleans indicating whether the target struct is a unit struct, a tuple struct, or a struct with named fields; they are used in `@apply_if`.

`@Self.field_count` is the number of fields of the target `struct` or `union`, and `@Self.variant_count` the number of variants of the target `enum`, as integer literals (`[u8; @Self.field_count]`, `fn count() -> usize { @Self.variant_count }`). `@Self.field_count` on an `enum` is an error (a variant's fields are counted with `0 @for_variant_fields { + 1 }`), and so is `@Self.variant_count` on a type that is not an `enum` (guard it with `@if_enum { .. }`).

## 6.9.3. Field iteration

### `@for_fields { ... }`

`@for_fields` repeats the block for each field of a `struct` or `union`. Inside `@for_variants` it iterates over the fields of the current variant, the same as `@for_variant_fields`. Using it on an `enum` without `@for_variants` is an error.

```nabi
@for_fields {
    println(self.@field);
}
```

This code is generated once per field.

### `@field`

`@field` is a placeholder for the current field, used in field access positions.

```nabi
self.@field                          // self.x, self.y, ...
@Self { @field: 0i32 }                // field initializer position
```

### `@field.name` identifier token

Where an identifier is required, the field name is inserted as an identifier token.

```nabi
let v = @field.name;                  // the name goes into the identifier position
fn @field.name() -> i32 { ... }       // can also be used as a function name
```

If a string is needed, append `.as_str()`. `@field.name.as_str()`, `@variant.name.as_str()`, `@item.name.as_str()`, and `@Self.as_str()` become the string literal of that name (`"x"`) at expansion time. `@Self.as_str()` is the name without type arguments. Name tokens such as `@item.name` written inside a string or f-string are substituted literally.

### `@field.type`

`@field.type` denotes the field's type.

```nabi
fn typed<T>() where T == @field.type { ... }
```

### `@field.is_pub`

This token is true if the field is `pub`, false otherwise.

```nabi
@apply_if(@field.is_pub) {
    // process only pub fields
}
```

### `@field.has_attr(X)`

True if `#[X]` is attached to the field.

```nabi
@apply_if(@field.has_attr(skip)) {
    // process only fields with #[skip]
}
```

### `@field.has_default` / `@field.default`

`@field.has_default` is true if the field has a default value (`x: i32 = 0`, [2.3.3](../002-types/203-struct.md#233-field-default-values)), and `@field.default` is the expression of that `= expr`. Using `@field.default` on a field without a default is an error.

```nabi
@apply_if(@field.has_default) {
    let @field.name = @field.default;
}
```

## 6.9.4. Variant iteration

### `@for_variants { ... }`

`@for_variants` repeats the block for each variant of an `enum`.

```nabi
@for_variants {
    @variant.name => /* ... */
}
```

### `@variant.name` variant name

`@variant.name` is the variant's name.

### `@variant.pattern` matching pattern

`@variant.pattern` is the variant in matching-pattern form, such as `Shape::Circle(r)`, `Shape::Pending`, or `Shape::Click { x: x1, y: y1 }`. The bindings in the `r`, `x1`, `y1` positions are compiler-generated names (hygiene, [6.9.9](#hygiene)), and their values are retrieved with `@field` inside `@for_variant_fields`. Templates cannot use the binding names directly.

```nabi
match self {
    @for_variants {
        @variant.pattern => /* ... */
    }
}
```

A `@variant.pattern` is also an expression: with each `@field` (in `@for_variant_fields`) bound by a `let`, it builds the variant.

### `@variant.index`, `@field.index`, `@field.binding`, `@Self.pattern`

`@variant.index` and `@field.index` are the position of the current variant and field in declaration order, as integer literals: `c.@field.index.push(self.@field)` reaches the tuple column of a field. Outside `@for_fields` / `@for_variant_fields` a `@field` token is an error that says to put it inside `@for_fields { .. }`. `@field.binding` is a local name for the current field's value: on a struct, `@Self.pattern` builds the struct from those locals (`Name { a: <a's binding>, .. }`, `Name(..)` or `Name`); in a variant, it is the name `@variant.pattern` binds.

```nabi
@for_fields {
    let @field.binding: @field.type = read_field(@field.name.as_str());
}
let made: @Self = @Self.pattern;
```

### `@variant.is_unit` / `@variant.is_tuple` / `@variant.is_named`

These tokens test the kind of variant.

```nabi
@apply_if(@variant.is_tuple) {
    // process only tuple variants
}
```

### `@for_variant_fields { ... }`

To iterate over the fields within a variant, use `@for_variant_fields`. `@field.name` etc. can be used inside it.

```nabi
@for_variants {
    @variant.pattern => {
        @for_variant_fields {
            process(@field);
        }
    }
}
```

## 6.9.5. Conditionals and bounds

### `@apply_if(cond) { ... }`

`@apply_if` generates the enclosed code only when `cond` is true.

```nabi
@apply_if(@field.is_pub) {
    // code
}

@apply_if(!@field.has_attr(skip)) {
    // only fields *without* skip
}

@apply_if(@field.is_pub && @field.has_attr(serialize)) {
    // both conditions true
}
```

`cond` is an expression combining boolean values of template tokens (e.g. `@field.is_pub`, `@field.has_attr(...)`) with ordinary boolean operators (`!`, `&&`, `||`), evaluated at compile time.

An `else` branch is expressed without a separate keyword, by an `@apply_if` with the opposite condition.

### `@accept(...)`

`@accept` restricts the kinds of target for an attribute; it is an error in a derive. It takes a list of item kinds separated by commas or `|`.

```nabi
#[impl_attribute(my_wrap)] pub @attribute {
    @accept(struct, enum)                    // accepts only struct, enum; rejects union
    @item
}

#[impl_attribute(timed)] pub @attribute {
    @accept(fn)                              // accepts only fn
    fn @item.name(...) { ... }
}
```

The possible kinds:

| Kind     | Meaning                 |
|----------|-------------------------|
| `struct` | `struct` item           |
| `enum`   | `enum` item             |
| `union`  | `union` item            |
| `fn`     | Function or method      |
| `impl`   | `impl` block            |
| `trait`  | `trait` item            |

With several kinds separated by commas or `|`, matching any one is allowed. If the target is not an allowed kind, it is a compile error at the use site.

### `@where_all(Trait)`

`@where_all` adds, as an automatic where clause, the condition that every field type must satisfy `Trait`.

```nabi
#[impl_derive(Same)] pub @derive {
    @where_all(Same)                         // OK if every field's type is Same
    impl Same for @Self {
        fn same(self, other: Self) -> bool {
            @for_fields {
                if !self.@field.same(other.@field) { return false; }
            }
            true
        }
    }
}
```

Here `Same` is a user trait (a built-in derive such as `Eq` cannot be redefined, [6.8.11](608-meta-blocks.md#6811-combination-with-registrationvisibility)). The generated `impl` then gets `T: Same` for each type parameter of the target type. No clauses are added for concrete field types; whether those fields implement `Same` is checked by type checking of the generated body. The target type's generic parameters are placed after the template's `impl`, and the only bounds are those added by `@where_all`.

## 6.9.6. derive_attribute tokens

The following tokens are additionally usable inside `#[impl_derive_attribute(Trait, attr)]`.

### `@has_derive_attr(name)`

Indicates whether the `derive_attribute`'s target attr is attached to the field.

### `@derive_attr(name)` the attr's argument

`@derive_attr(name)` denotes the argument of the attached attr.

### `@has_derive_attr(name, k)` / `@derive_attr(name, k)`: one key

The arguments of the field's `#[name(..)]` may be keys with a literal value (`k = "v"`) and bare keys (`k`). With a key as the second argument, `@has_derive_attr(name, k)` is whether the field's attribute lists `k`, and `@derive_attr(name, k)` is the value written after `k =`, as a use token (`true` for a bare `k`). When the attribute declares its keys (`attr(k, ..)`, [6.8.4](608-meta-blocks.md#684-impl_derive_attributetrait-attr-per-field-derive-attribute)), `k` must be one of them; another is an error at the template. `@derive_attr(name, k)` on a field whose attribute lacks `k` is an error that says to guard it with `@apply_if(@has_derive_attr(name, k))`.

```nabi
#[impl_derive_attribute(Args, arg(short, help))] pub @derive {
    impl Args for @Self {
        fn specs() -> List<Spec> {
            [@for_fields {
                Spec {
                    name: @field.name.as_str(),
                    @apply_if(@has_derive_attr(arg, short)) { short: @derive_attr(arg, short), }
                    @apply_if(@has_derive_attr(arg, help)) { help: @derive_attr(arg, help), }
                },
            }]
        }
    }
}

// use site
#[derive(Args)]
struct Opts {
    #[arg(short = "v", help = "print more")]
    verbose: bool;
    #[arg(help = "files to read")]
    files: List<str>;
}
// → `Spec { name: "verbose", short: "v", help: "print more" }`,
//   `Spec { name: "files", help: "files to read" }`
```

### `@Self.has_derive_key(k)`

Whether the type's own `#[attr(..)]` lists key `k`. `k` must be declared with `#[impl_derive_attribute(Trait, attr(k, ..))]` ([6.8.4](608-meta-blocks.md#684-impl_derive_attributetrait-attr-per-field-derive-attribute)); an undeclared key is an error. `#[derive(Nabion)]` reads `#[nabion(deny_unknown_fields)]` this way.

```nabi
#[impl_derive_attribute(Serialize, rename)] pub @derive {
    @for_fields {
        @apply_if(@has_derive_attr(rename)) {
            let key = @derive_attr(rename);  // the argument of #[rename("...")]
            obj.set(key, ...);
        }
    }
}
```

## 6.9.7. Attribute-only tokens

The following tokens are used inside `#[impl_attribute(name)]`.

### `@item` reusing the original item

`@item` reuses the entire item the attribute is attached to, as is.

```nabi
#[impl_attribute(deprecated_warning)] pub @attribute {
    @item                                    // the original as is
}
// the attribute above is effectively a no-op; the original passes through unchanged
```

### `@item.body`

`@item.body` is the function body; it is meaningful only for function items.

### `@item.name`, `@item.params`, `@item.return_type`

These tokens denote the parts of a function signature.

```nabi
#[impl_attribute(timed)] pub @attribute {
    fn @item.name(@item.params) -> @item.return_type {
        let start = Instant::now();
        let r = { @item.body };
        log(f"@item.name took {Instant::now() - start}");
        r
    }
}
```

### `@attribute` `#[...]` arguments

`@attribute` means the arguments written at the attachment. If the use site applies an attribute defined by `#[impl_attribute(name)]` as `#[name(args)]`, `@attribute` is those args.

```nabi
#[impl_attribute(log_prefix)] pub @attribute {
    fn @item.name() {
        let p = @attribute;                  // the use site's argument
        log(f"{p}: {@item.name}");
        @item.body
    }
}

// use site
#[log_prefix("INFO")]
fn run() { ... }
// → generated.
//   fn run() {
//       let p = "INFO";
//       log(f"{p}: run");
//       ...
//   }
```

## 6.9.8. Call-site (macro) tokens

The tokens above bind a type or item the definition is attached to. A macro ([6.8.12](608-meta-blocks.md#6812-the-macro-form)) binds the caller's tokens instead, through a matcher, and uses these tokens; `@for` is macro-only too: in an `@attribute` or `@derive` block it is an error (they repeat with `@for_fields` / `@for_variants`). `@concat` works in all three forms. Conversely, the shape tokens (`@apply_if` and the `@Self` / `@field` / `@variant` tokens it tests) are errors in a macro: a macro has no type shape to test.

| Token                                  | In a matcher / template                                        | Section |
|----------------------------------------|----------------------------------------------------------------|---------|
| `@name: <fragment>`                    | Matcher: capture one fragment (`expr`, `stmt`, `block`, `ty`, `pat`, `path`, `ident`, `literal`, `item`, `tt`) | 6.9.8 |
| `@name`                                | Template: the captured fragment, or the current element in a `@for` | 6.9.8 |
| `@rep[sep = ",", trailing] { .. }`     | Matcher: a sub-pattern repeated, with an optional separator and optional trailing one | 6.9.8 |
| `@for @name { .. }`                    | Template: the body once per element of a captured sequence (lockstep over several) | 6.9.3 |
| `@concat(a, @name, ..)`                | Template: join name tokens / string literals into one indexed identifier (also in `@derive` / `@attribute`) | 6.9.8 |

### `@name: <fragment>` fragment capture

In a matcher, `@name: <fragment>` captures part of the call as raw tokens; nothing is parsed at match time, so the first rule whose literal tokens and token counts fit is chosen ([6.8.12](608-meta-blocks.md#matchers-and-fragments)). `ident`, `literal` and `tt` take one identifier, one literal token and one token tree (a token or a balanced group); the other fragments take the run of tokens up to the next literal token of the matcher. The expansion is parsed as ordinary code after transcription, so the captured code is typed and checked there and the tools resolve names inside it; an argument that does not fit where the template puts it is a syntax error reported at the call. `tt` is the fragment for a DSL that is not Nabi. Literal tokens elsewhere in the matcher must appear verbatim. In the template, `@name` substitutes the capture, preserving the call site's hygiene ([6.9.9](#699-expansion-mechanism)).

### `@rep` / `@for` repetition

`@rep[sep = ",", trailing] { <sub-pattern> }` matches the sub-pattern zero or more times; a metavariable captured inside binds a sequence. `@for @name { <body> }` is the same iterator as `@for_fields` ([6.9.3](#693-field-iteration)), here over that sequence; metavariables from one `@rep` iterate in lockstep and must have equal length. `@rep` and `@for` nest.

### `@concat(..)` name synthesis

`@concat(..)` joins name tokens (`@name`, a bare identifier) and string literals into a single identifier, at a definition or path position only. The identifier it forms is recorded, so symbol search and go-to-definition reach the generated item ([6.8.12](608-meta-blocks.md#tooling-and-hygiene)); a non-name piece, or use outside a name position, is an error.

In a `@derive` or `@attribute` block its pieces may also be the name and index tokens of the target (`@Self.name`, `@field.name`, `@variant.name`, `@field.index`, `@variant.index`, `@item.name`), so a derive can declare a named companion item. A result that does not start with a letter or `_` is an error.

```nabi
#[impl_derive(Soa)] pub @derive {
    pub struct @concat(@Self.name, Columns) {
        @for_fields { pub @field.name: List<@field.type>; }
    }
}

#[derive(Soa)]
struct Point { x: f32; y: f32; }    // also declares `PointColumns { x: List<f32>; y: List<f32>; }`
```

## 6.9.9. Expansion mechanism

### Collection and expansion

Meta expansion proceeds as follows.

1. A pre-pass before parsing collects meta blocks from every file of libraries and user packages ([6.8.11](608-meta-blocks.md#6811-combination-with-registrationvisibility)).
2. When the parser meets an item carrying `#[derive(X)]` or `#[x]`, it substitutes the template tokens in that meta block's body with the item's values and parses the result again.
3. Generated items are meta-processed again, repeating until a fixed point is reached.
4. A cycle (`A → B → A`) is an error ([6.8.8](608-meta-blocks.md#688-fixed-point-expansion)).

### Relation to `cfg`

A `#[cfg(..)]` on a meta block itself determines the block's **registration**. cfg evaluation precedes meta block collection, so the derive / attribute of a block whose cfg is false is not registered. Two blocks can therefore define the same name under mutually exclusive cfgs. The `#[cfg]` of modules enclosing the block (inline `mod`s and file modules) also applies to registration. A `#![cfg]` inside the file does not yet apply.

A [`#[cfg]`](603-cfg.md) on an item is copied as is onto every item generated from it. An item whose cfg is false therefore disappears together with its generated output; the overall order is:

```
parse → cfg eval → meta expansion (fixed-point) → name resolution → type checking → ...
```

### Hygiene

Local names bound by a template (`let` / `for` bindings and closure parameters, including inside `{..}` of f-strings) are renamed to non-colliding names at expansion. Template locals therefore neither hide nor capture names that come from the item (`@item.body`, `@item.params`, `let @field.name = ..`).

```nabi
#[impl_derive(Foo)] pub @derive {
    impl Foo for @Self {
        fn foo(self) -> i32 {
            let temp = compute();             // renamed at expansion
            temp * 2
        }
    }
}

// use site
#[derive(Foo)]
struct S { temp: str; }                       // the use site's 'temp' field

let s = S { temp: "hi" };
let r = s.foo();                              // no collision with the meta-generated 'temp'
```

Names bound by `match` arm patterns and the `S { x }` shorthand are not renamed. Templates do not use names in those positions that could collide with the item. Bindings of `@variant.pattern` are compiler-generated names and do not collide.

### Names in an expansion

Each token of an expansion comes either from the **definition** (the template, or code the compiler writes) or from the **use** (what `@Self`, `@field`, `@variant`, `@item`, `@attribute`, `@derive_attr` and macro captures substitute). A path takes the side of its first segment; its generic arguments keep their own.

A path from the definition is resolved and its access checked as if it were written in the module that contains the meta block: names the expansion itself declares come first, then that module's scope (its items and `use`s, `pkg::`, the dependencies of its package, the prelude's names). It may name that module's private items and its package's `pub(pkg)` items. A method name from the definition is looked up with the traits in scope there ([3.5](../003-expressions/305-member-access-and-method-call.md)). A path from the use is resolved and checked at the use site, so code passed into a meta definition gains no access. `pkg::` in a template names the package that defines the meta block, wherever it expands.

Code the compiler writes (built-in derives, `#[literal]` handler calls) is seen from the module of what it names: a handler is called as if from its own module, so it may have any visibility.

Generated items are placed in the use site's module and the names they declare are visible there. Template locals are renamed (Hygiene above).

`@use_site(path)` makes a template path resolve at the use site, for a name the user is expected to define. Its argument must be a path.

```nabi
// package mylib
pub(pkg) fn combine(h: u64, x: u64) -> u64 { .. }

#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self {
        fn my_hash(self) -> u64 {
            let h: u64 = 0u64;
            @for_fields { h = combine(h, self.@field.my_hash()); }   // mylib's, though pub(pkg)
            @use_site(check)(self);                                    // the user's `check`
            h
        }
    }
}
```

A name from the definition that the defining module does not have is an error at the template, with the use as a note, and help naming `@use_site(..)`. When a package is built, every rooted path (`pkg::..`, `std::..`, a dependency's) in its `pub` meta blocks must resolve from the block's module, used or not, so the error is the author's.

### Diagnostic locations

Errors during expansion, parse errors of the expansion result and type errors in the generated code point to the template's location and append the use site (`#[derive(X)]` / `#[x]`) as a note. An error that comes from one field (its type, a value of its attribute) also points at that field (``for field `x` ``). Tokens that come from the item through template tokens point to the item's location.

## 6.9.10. Debugging

`nabic build --emit=expansions` ([9.9.10](../009-tools/909-debug-tools.md#9910-meta-block-debugging)) writes the expansion results of the user package as source text to `target/<profile>/build/<name>.expanded.nabi` (or the `-o` path). Each expansion is a ``// expanded from this `#[derive(X)]` (file:line:col)`` comment followed by the generated source; if the output is expanded again, it follows the outer expansion. A macro call inside an expansion is shown expanded in place.

## 6.9.11. Summary

Template tokens are the code generation vocabulary of Nabi's meta-system. Simple as they are, they cover all the core cases of practical derives and attributes.

Nabi's choice is to limit the flexibility of token transformation in exchange for debuggability, tool friendliness, and guaranteed hygiene.

This concludes chapter 6 (Meta-system).
