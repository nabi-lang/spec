# 3.8. `nabion::` - the NABION data format

NABION (Nabi Object Notation) is a data format that writes values in Nabi's literal syntax. The file extension is `.nabion`. It fills the same role as Rust's RON and Zig's ZON; the package manifest `nabi.nabion` and the lockfile `nabi.lock` ([refs-lang 9.10.3](../../refs-lang/009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)) use this format.

## 3.8.1. Format

NABION is a subset of Nabi expression syntax. The library reads it with its own lexer and recursive-descent parser over that subset; the compiler's parser is not used at run time. One file holds one value.

| Form        | Example                                               | Notes                                                 |
|-------------|-------------------------------------------------------|-------------------------------------------------------|
| none        | `none`                                                |                                                       |
| bool        | `true`, `false`                                       |                                                       |
| Integer     | `42`, `-7`, `0xff`, `0b1010`, `0o17`, `1_000_000`      | No suffix                                             |
| Float       | `3.14`, `-1.5e-3`, `2E10`                              | No suffix                                             |
| char        | `'a'`, `'\n'`                                         | `c'..'` not allowed                                   |
| String      | `"text"`, raw strings, multi-line strings             | f-strings not allowed                                 |
| List        | `[a, b, c]`                                           |                                                       |
| dict        | `{ "k": v, 1: w }`                                    | Keys are values (strings, numbers, etc.)              |
| Tuple       | `(a, b)`, `(a,)`                                      | `(x)` is grouping, not a tuple                        |
| struct      | `Point { x: 1, y: 2 }`, `{ x: 1, y: 2 }`, `Pair(1, 2)`, `Unit {}` | The name is optional. Field names are identifiers. Tuple structs are `Name(a, b)` |
| enum variant | `Shape::Unit`, `Shape::Circle(1.0, at: p)`, `Shape::Rect { w: 2, h: 3 }` | Written with the full path            |

- A trailing comma is allowed. Comments are `//` and `/* */`.
- `{}` is both an empty struct and an empty dict (the target type decides). A single `{ .. }` may contain either fields (identifier keys) or keys (value keys), not both; mixing them is an error.
- Not allowed: bare identifiers, operators (only unary `-` before a number is allowed), method calls, `..`, blocks, f-strings, Set literals (`{1, 2}`; write a Set as a list), `c'..'` literals, generic paths (`List<i32>::..`), and tokens left after the value. All are errors showing `line:col`.
- NaN and infinity have no NABION notation. `nan` / `inf` in the input are rejected as bare identifiers.
- NABION floats are `f64`. A float literal past the `f64` range (`1e999`) is an error naming the range: ``float `1e999` is out of range (NABION floats are f64, up to about 1.8e308)``. `nabion::parse` and the toolchain's reader (`nabi.nabion`, `nabi.lock`, `nabi fmt` on a `.nabion` file) follow the same rule.

> [!NOTE]
> Known limitation: a dict whose first key is `none` (`{ none: 1 }`) is not supported, because the reader takes `none:` as a field name and reads the braces as a struct.

```nabion
// config.nabion
Config {
    name: "geom",
    retries: 3,
    ratio: 0.75,
    tags: ["a", "b"],
    limits: { "cpu": 2, "mem": 512 },
    mode: Mode::Fast,
    proxy: none,
}
```

## 3.8.2. Correspondence with Nabi types

| NABION          | Nabi type                                                                     |
|----------------|------------------------------------------------------------------------------|
| `none`         | `none` of `T \| none`                                                        |
| `true` / `false` | `bool`                                                                     |
| Integer        | Every integer type, `int`. Also accepted where a float is expected (`f32` / `f64` / `float`) |
| Float          | `f32`, `f64`, `float`                                                         |
| char           | `char`                                                                        |
| String         | `str`                                                                         |
| List           | `List<T>`, `[T; N]`, `Set<T>`                                                 |
| dict           | `Dict<K, V>`                                                                  |
| Tuple          | Tuples of 1 ~ 6 elements                                                      |
| struct         | `struct` (`#[derive(Nabion)]`)                                                 |
| enum variant   | `enum` (`#[derive(Nabion)]`)                                                   |
| Any value      | `Value`                                                                       |

## 3.8.3. Reading

`parse` reads text into a `Value`.

| Input                        | `Value`                                                        |
|------------------------------|----------------------------------------------------------------|
| `{ ident: v, .. }`           | Unnamed `Struct`                                               |
| `{ "k": v, .. }`, `{}`       | `Dict`                                                         |
| `Name { f: v }`              | Named `Struct`                                                 |
| `A::B { f: v }`              | `Variant` (values in `named`)                                  |
| `A::B(a, k: b)`, `A::B`      | `Variant` (`args`, `named`)                                    |
| `Name(a, b)`                 | `Variant` with a single-segment path (tuple struct)            |
| Integer                      | `Int` (exact `int`)                                            |

In typed reads (`from_str`, `from_value`), the target type determines the interpretation.

- A struct target accepts `Struct`, `{}` and `A::B { .. }`, so struct names with a module path (`geom::Point { .. }`) are accepted too.
- A written name must match the target. For structs, the last path segment is compared; for enum variants, the last two segments.
- A field the type does not have is skipped, so a format can add fields without breaking older readers. With `#[nabion(deny_unknown_fields)]` on the type it is an error at the position of that field name. This applies to structs and to named variants.
- A missing field takes the field's `= default` if it has one, `none` if its type is `T | none`, and is an error otherwise (`FromNabion::from_missing`).
- An integer value outside the range of the target integer type is an error.
- Named variants can also be read as `E::V(a, k: b)`. Positional arguments fill the fields in order, followed by named arguments.

## 3.8.4. Canonical form

The canonical form emitted by `nabion::to_str` and by `nabi` (when editing the manifest or writing `nabi.lock`) is as follows. The canonical form is also the readable form, so there is no separate pretty function.

- Indentation is 4 spaces.
- A value is written on one line if it fits within 100 columns; otherwise on multiple lines with a comma after each element.
- Floats use the shortest notation that reads back to the same value (`1.0`, `1.0e300`). A float with no NABION form is thrown as a `NabionError` with its path: NaN / infinity ("NaN/infinity has no NABION form") and a `float` past the `f64` range ("float 1.2e600 is out of range (NABION floats are f64, up to about 1.8e308)").
- A variant with only named values has one canonical form, `E::V { .. }` (`Value::Variant` keeps no form flag).
- Bidi controls (U+202A-U+202E, U+2066-U+2069) in strings and chars are written escaped (`\u{202e}`).
- `nabion::to_str_flat` writes the same text on one line, whatever its width.
- Reading the canonical form back yields the same value (round-trip).

`nabi fmt` checks that a `.nabion` file is NABION and only re-indents it. It does not rewrite it into canonical form, so comments and layout are preserved.

## 3.8.5. Library: `std::nabion`

```nabi
use std::nabion;

#[non_exhaustive]
pub enum Value {
    None,
    Bool(bool),
    Int(int),
    Float(float),
    Char(char),
    Str(str),
    List(List<Value>),
    Dict(Dict<Value, Value>),
    Tuple(List<Value>),
    Struct { name: str | none; fields: Dict<str, Value>; },
    Variant { path: str; args: List<Value>; named: Dict<str, Value>; },
}
impl Value {
    fn to_str(self) throws (NabionError) -> str;   // canonical form
    fn kind_name(self) -> str;       // kind name for error messages (below)
}                                    // + Display, Debug, PartialEq, Eq, Hash

/// `v` as a `Value` through its type's description (`core::reflect`):
/// `Inspect` structs and enums, lists, dicts, sums and primitives.
pub fn to_value_any(v: any) throws (NabionError) -> Value;
/// A value of the type `info` describes, made from `v`; a struct field
/// `v` leaves out takes its default.
pub fn from_value_any(info: core::reflect::TypeInfo, v: Value) throws (NabionError) -> any;

pub struct NabionError { .. }         // private fields (0.5.10); + Display, Debug, Error, PartialEq, Eq, Clone
impl NabionError {
    pub fn new(message: str) -> NabionError;      // a decode error with no position
    pub fn within(self, seg: str) -> NabionError; // self one level down: seg (".name", "[3]") goes in front of the path
    pub fn message(self) -> str;     // without the position Display adds
    pub fn line(self) -> u32;        // 1-based; 0 when the value did not come from text
    pub fn column(self) -> u32;      // 1-based, in code points; 0 as for line
    pub fn path(self) -> str;        // ".servers[2].port"; empty at the root
}

pub trait ToNabion { fn to_nabion(self) -> Value; }
pub trait FromNabion {
    fn from_nabion(v: Value) throws (NabionError) -> Self;
    fn from_missing() throws (NabionError) -> Self;   // default: error. none for T | none
}

pub fn parse(s: str) throws (NabionError) -> Value;
pub fn to_str<T: ToNabion>(v: T) throws (NabionError) -> str;       // canonical form
pub fn to_str_flat<T: ToNabion>(v: T) throws (NabionError) -> str;  // canonical form on one line
pub fn from_str<T: FromNabion>(s: str) throws (NabionError) -> T;
pub fn to_value<T: ToNabion>(x: T) -> Value;
pub fn from_value<T: FromNabion>(v: Value) throws (NabionError) -> T;
```

- `Display` of `NabionError` is `line:col: msg` if there is a position, otherwise `at <path>: msg`. Errors from typed reads carry paths such as `.servers[1].port` or `.limits["cpu"]`, and `from_str` converts that path to the `line:col` of the input.
- `Value` is `#[non_exhaustive]`: a `match` outside the standard library needs a `_` arm.
- `Value::kind_name` is the kind name used in error messages: in order, `"none"`, `"bool"`, `"integer"`, `"float"`, `"char"`, `"string"`, `"list"`, `"dict"`, `"tuple"`, `"struct"`, and for `Variant` ``"variant `<path>`"``.
- `==` and `hash` on `Value` are structural. Dict entries and struct fields are compared regardless of order, and floating-point values are compared by identity (NaN == NaN, -0.0 != 0.0). An unnamed empty struct and an empty dict are equal.
- `Display`, `Debug`, `==` and `hash` on `Value` never panic. `Display` writes the canonical form unchecked (a NaN shows as `nan`); `Debug` writes it on one line.
- Standard implementations: every integer type, `f32` / `f64`, `int` / `float`, `bool` / `char` / `str`, `Value`, `T | none`, `List`, `Set`, `Dict`, tuples of 1 ~ 6 elements, `[T; N]`.

The code `@derive` generates for `#[derive(Nabion)]` calls `Fields`, `struct_args`, `variant_of`, `variant_args`, `unknown_variant` and `from_value_at`. They are `pub(pkg)`: names a template writes are checked in the template's module ([refs-lang 6.9.9](../../refs-lang/006-meta-system/609-template-tokens.md#names-in-an-expansion)), so the derive works in any package, and a direct use outside `std` is an error. The reader and writer are private to the standard library.

### `#[derive(Nabion)]`

Implements both `ToNabion` and `FromNabion` for structs and enums (including generic ones).

| Definition           | Written form           |
|----------------------|------------------------|
| Struct with named fields | `Name { f: v }`    |
| Tuple struct         | `Name(a, b)`           |
| Unit struct          | `Name {}`              |
| Unit variant         | `E::V`                 |
| Tuple variant        | `E::V(a)`              |
| Named variant        | `E::V { f: v }`        |

A field's `= default` is used for missing fields (3.8.3). `#[nabion(deny_unknown_fields)]` on the type makes a field it does not have an error (3.8.3); `#[nabion(..)]` takes no other key, and a misspelled one is an error naming the valid keys. The derive is built with the meta tokens `@Self.name`, `@Self.is_unit` / `is_tuple` / `is_named`, `@field.has_default` / `@field.default`, `@Self.has_derive_key` ([refs-lang 6.9](../../refs-lang/006-meta-system/609-template-tokens.md)).

```nabi
#[derive(Nabion)]
struct Server { host: str; port: u16 = 80; tls: bool | none; }

#[derive(Nabion)]
struct Config { name: str; servers: List<Server>; limits: Dict<str, i32>; }

let cfg: Config = try nabion::from_str(text);   // errors are "line:col: msg"
print(try nabion::to_str(cfg));
```

### Limitations

- `Dict` / `Set` are written in insertion order, as they iterate (2.9).
- `[T; 0]` cannot be read.
- Tuples are limited to 6 elements.
- A dict whose first key is `none` cannot be read (3.8.1).

## 3.8.6. Comparison with other formats

| Format | struct notation             | Notes                             |
|--------|-----------------------------|-----------------------------------|
| RON    | `Name(field: v)`            | A dedicated syntax different from Rust |
| ZON    | `.{ .name = "x" }`          | Zig's anonymous struct literal    |
| NABION  | `Name { field: v }`         | Nabi's own literal syntax as is    |

NABION adds no separate syntax and uses Nabi literals directly, so a value written in source and a value written in a file look the same.
