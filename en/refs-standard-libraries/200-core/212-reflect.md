# 2.12. reflect - `Reflect` / `Inspect`

The `core::reflect` module provides the authoritative signatures for runtime type information and **field introspection**. The detailed semantics are governed by [refs-lang 6.10 reflection](../../refs-lang/006-meta-system/610-reflection.md); this chapter covers only the standard library's trait shapes.

## 2.12.1. Two-level traits

Reflection is split into two levels of traits.

| Trait     | Automatic / opt-in            | Provides                          | Where used                                         |
|-----------|-------------------------------|-----------------------------------|----------------------------------------------------|
| `Reflect` | **automatic** (every type)    | type_name / size / align          | a `core` trait, in scope everywhere (rarely used) |
| `Inspect` | opt-in (`#[derive(Inspect)]`) | field_names / fields / variant_name / get_field / methods | trait available through the implicit prelude, derive is explicit |

The split into automatic and opt-in controls binary size. If every struct carried field reflection metadata, the binary would grow; the opt-in scheme generates the metadata only for the types that need it.

## 2.12.2. `Reflect` - both static and instance

```nabi
trait Reflect {
    fn type_name() -> str;        // static: T::type_name()
    fn size() -> usize;
    fn align() -> usize;

    fn type_name_of_val(self) -> str; // instance: x.type_name_of_val() (any / sum also OK)
    fn size_of_val(self) -> usize;
}
```

This trait is implemented automatically for every type; the compiler attaches the type info to the binary automatically.

The compiler provides `Reflect` for every type; there is no impl in source.

- A hand-written `impl Reflect for T` is an error. The bound `T: Reflect` always holds.
- If a type has its own method with the same name, such as `size()` / `type_name()`, that method shadows the reflection method.
- The name (`type_name`) carries no module path (`List<i64>`, `Config`), nor trailing type arguments equal to their default (`Dict<str, i32>`, [refs-lang 2.10.5](../../refs-lang/002-types/210-generics.md#2105-defaults)). The static name of a sum type is written in canonical order ([2.6.4](../../refs-lang/002-types/206-sum-types.md#264-set-semantics)) (`f64 | i32`, with `none` last). Instance forms such as `type_name_of_val(self)` report the member present at runtime (for a sum) or the content of an `any`.
- `core::reflect::cast<T>(value: any) -> T | none` extracts a `T` from an `any` using the same rules as `set_field`. A value stored whole as a sum, such as a `T | none` field read through `get_field` or `Field::get`, gives the member it holds: `cast::<str>` of a field `a: str | none` holding `"x"` is `"x"`. A sum `T` also takes one of its members, so `set_field` on a `T | none` field accepts a `T`. There is no free function for a type's methods or fields: they are `T::methods()` / `T::fields()` and need `#[derive(Inspect)]`, the type's consent.

### Static vs instance

One suffix rule: the plain form is the type's answer, fixed at compile time; the `_of_val` form is the value's answer at run time.

```nabi
i32::type_name();          //→ "i32"           (static, decided at compile time)
i32::size();               //→ 4
i32::align();              //→ 4

let x: i32 = 42;
x.type_name_of_val();      //→ "i32"           (instance, usually the same as static)

let y: i32 | f64 = 42i32;
y.type_name_of_val();      //→ "i32"           (runtime dispatch, checks the sum tag)

let z: any = "hello";
z.type_name_of_val();      //→ "str"           (runtime, any slot dispatch)
```

`type_name_of_val` is a runtime dispatch, so for a sum type or `any` it returns the name of the variant currently held.

### No global functions

```nabi
typeof(x);                 // ERROR: no global typeof
sizeof(T);                 // ERROR: no global sizeof
```

Instead, everything is uniformly a method.

```nabi
x.type_name_of_val()   // instance
T::type_name()             // static
T::size()
T::align()
```

This is part of [refs-lang 3.5 Member access and method calls](../../refs-lang/003-expressions/305-member-access-and-method-call.md): all introspection is unified in method form.

## 2.12.3. `Inspect` - opt-in fields / methods

```nabi
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // private fields too
    fn set_field(ref self, name: str, value: any) -> bool;  // whether it succeeded

    // Default bodies; the derive overrides the value forms for an enum.
    fn fields() -> List<FieldInfo>;          // the field table of Self
    fn methods() -> List<MethodInfo>;        // the method table of Self
    fn variant_names() -> List<str>;         // empty for a struct
    fn variant_name(self) -> str;            // a struct: self.type_name_of_val()
    fn field_names_of_val(self) -> List<str>; // the fields self holds
}

#[non_exhaustive]
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
pub struct FieldInfo {
    pub name: str;                 // "0", "1", .. for a tuple field
    pub type_name: str;            // instantiated: f64 for `v: T` in Box<f64>
    pub is_pub: bool;              // true only for a plain `pub` field
}

#[non_exhaustive]
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
pub struct MethodInfo {
    pub name: str;
    pub arity: usize;              // number of arguments excluding self
    pub is_static: bool;           // method that does not take self
    pub arg_types: List<str>;      // argument type names excluding self
    pub return_type: str;
}
```

- `get_field` returns the field's value as `any`, or `none` for an unknown name. Private fields can be read too.
- `set_field` assigns and returns `true` when the value's type is **exactly the same** as the field's type, or when an integer of a different integer type **fits without loss** (`u.set_field("id", 42)` into a `u64` field). Otherwise it does not assign and returns `false`. There are no other conversions (e.g. `i32` → `f64`).
- `methods()` returns **only the type's inherent methods** (excluding trait methods), sorted by name. Methods of a generic `impl` show the types as written (`T`).
- `fields()` lists every field in declaration order with its type name and whether it is `pub`. **Private fields are listed too**: deriving `Inspect` is the type author's consent, the same as for `get_field`. The type name is the instantiated one (`Box<f64>` reports `v: f64`, `items: List<f64>`), with no module path, as `type_name` writes it.
- An `enum` has no fields at the type level (`field_names()` and `fields()` are empty lists). `variant_names()` lists its variants in declaration order. On a value, `variant_name()` is the variant it holds (`"Circle"`), `field_names_of_val()` that variant's fields (`"0"`, `"1"` for a tuple variant, the names for a named one), and `get_field` reads them. A variant's payload is read-only through `Inspect`: `set_field` on an enum is `false`.
- On a struct, `variant_names()` is an empty list, `variant_name()` is its type name (`type_name_of_val()`), and `field_names_of_val()` is its fields.
- `fields()`, `methods()`, `variant_names()`, `variant_name()` and `field_names_of_val()` have default bodies, so a hand-written `impl Inspect` needs only the first three methods.
- Calling an `Inspect` method on a type without the derive is an error whose help says what to write: ``add `#[derive(Inspect)]` to `Point` ``.

### Derive example

```nabi
#[derive(Inspect)]
struct User {
    id: u64;
    name: str;
    email: str;
}

let u = User { id: 1, name: "Alice", email: "a@b" };

User::field_names();       //→ ["id", "name", "email"]
u.get_field("name");       //→ any("Alice")
u.set_field("id", 42);     //→ true
u.id;                      //→ 42
User::fields()[0].type_name; //→ "u64"

#[derive(Inspect)]
enum Shape { Circle(f64), Rect { w: f64; h: f64; } }

let s = Shape::Rect { w: 2.0, h: 3.0 };
Shape::variant_names();    //→ ["Circle", "Rect"]
s.variant_name();          //→ "Rect"
s.field_names_of_val();    //→ ["w", "h"]
s.get_field("w");          //→ any(2.0)
```

### Using meta blocks

`#[derive(Inspect)]` is implemented as a meta block in `core::reflect` ([refs-lang 6.8](../../refs-lang/006-meta-system/608-meta-blocks.md), [6.9](../../refs-lang/006-meta-system/609-template-tokens.md)). It writes only the three accessors the type's description points at; every other method has a default body over `type_info()`. In summary:

```nabi
#[impl_derive(Inspect)]
pub @derive {
    impl core::reflect::Inspect for @Self {
        // Field `field` of `owner`: a struct's by position; an enum's as
        // `variant << 16 | index`, and `usize::MAX` for the variant index.
        fn __inspect_get(owner: any, field: usize) -> any | none {
            match core::reflect::cast::<@Self>(owner) {
                s: @Self => {
                    @if_struct {
                        @for_fields {
                            if field == @field.index { let held: any = s.@field; return held; }
                        }
                    }
                    @if_enum { /* per variant, through @variant.pattern */ }
                    return none;
                },
                none => { return none; },
            }
        }
        // A copy of `owner` with one field set (structs only).
        fn __inspect_set(owner: any, field: usize, value: any) -> any | none { /* .. */ }
        // A value from named fields; a missing field takes its default.
        fn __inspect_build(variant: usize, fields: Dict<str, any>) -> (any | none, str) {
            @if_struct {
                @for_fields {
                    let @field.binding: @field.type = /* fields[name], or @field.default */;
                }
                let built: @Self = @Self.pattern;
                let held: any = built;
                return (held, "");
            }
            /* @if_enum: the same per variant */
        }
    }
}
```

### Where dynamic dispatch is needed

`Inspect` is used in domains that need access by field name, such as config objects, serializers and debuggers. In ordinary code `Reflect` alone is enough.

```nabi
fn dump_struct<T: Inspect>(x: T) {
    for f in T::fields() {
        if let v: any = x.get_field(f.name) {
            println(f"{f.name}: {f.type_name} ({v.type_name_of_val()})");
        }
    }
}
```

## 2.12.4. REPL output

The REPL prints a bare last expression with `Display`, else `Debug`, a trailing `?` with `Debug`, and a value without them with this chapter's `Reflect::type_name_of_val` ([2.10 fmt](210-fmt.md), [refs-lang 6.10.3](../../refs-lang/006-meta-system/610-reflection.md#6103-repl--jupyter-auto-printing)).

```
>>> let x = 42i32;
>>> x
42

>>> [1, 2, 3]
[1, 2, 3]

>>> [1, 2, 3]?
[1, 2, 3]

>>> User { id: 1, name: "Alice" }?
User { id: 1, name: "Alice" }
```

A bare expression uses `Display`, else `Debug`, and a trailing `?` uses `Debug`. A value without either (`Debug` for a trailing `?`) prints only its type name; an `any` value, the name of the type it holds.

## 2.12.5. Module mapping

| Item               | Location                                          | Mechanism                         |
|--------------------|---------------------------------------------------|-----------------------------------|
| `Reflect` (automatic) | compiler + `core::reflect`                | type info table, attached to the binary  |
| `Inspect` derive   | `core::reflect` + meta block `#[impl_derive]` | code generation from template tokens |
| `MethodInfo`       | `core::reflect`                              | plain struct                      |
| `FieldInfo`       | `core::reflect`                              | compiler field table (instantiated types) |
| `any` type         | compiler built-in                                 | type tag + payload                |

## 2.12.6. Comparison with other languages

| Aspect              | Rust                         | Java               | Python              | Nabi                                         |
|---------------------|------------------------------|--------------------|---------------------|---------------------------------------------|
| Type name           | `std::any::type_name::<T>()` | `getClass()`       | `type(x).__name__`  | `x.type_name_of_val()` method                   |
| Field introspection | **none** (libraries such as `bincode`) | Reflection API | `vars(x)` `getattr` | `x.get_field(name)` (opt-in)          |
| Runtime dispatch    | `dyn Trait` (vtable)         | automatic          | automatic           | sum type tag + `any` / `any Trait` (opt-in) |
| Automatic impl      | **none**                     | every object       | every object        | `Reflect` automatic, `Inspect` opt-in       |
| Memory cost         | 0                            | metadata for every class | dict for every object | only `Reflect` automatic (opt-in split) |

Nabi differs in two main ways. First, the two-level split provides the lightweight `Reflect` automatically and the heavyweight `Inspect` as opt-in. This is more memory-efficient than the all-automatic model of Java or Python and more ergonomic than Rust's all-manual model. Second, for method consistency there are **no** global functions such as `typeof(x)`; all introspection is provided in method form.

The detailed reflection semantics are governed by [refs-lang 6.10](../../refs-lang/006-meta-system/610-reflection.md).

## 2.12.7. `TypeInfo`

The description `T::type_info()` / `info::<T>()` returns ([refs-lang 6.10.7](../../refs-lang/006-meta-system/610-reflection.md#6107-typeinfo---the-types-description), layout [8.13](../../refs-lang/008-abi/813-type-descriptor.md)). Accessors that return a value read it in place and allocate nothing; those that return a `List` allocate it.

```nabi
pub fn info<T>() -> TypeInfo;                 // any T; fields and accessors when T: Inspect
pub fn info_of(value: any) -> TypeInfo | none; // the type value holds, if described
pub fn find(path: str) -> TypeInfo | none;    // "game::Player"

pub enum TypeKind { Struct, Enum, Union, Sum, Prim, Str, List, Dict, Array, Tuple, Fn, Any, Opaque }
pub struct ReflectError { pub message: str; }  // + Display, Error

impl TypeInfo {                                 // + Clone, PartialEq (by key), Debug
    pub fn name(self) -> str;                   // as type_name: "Player", "List<f64>"
    pub fn path(self) -> str;                   // "game::Player"
    pub fn package(self) -> str;
    pub fn type_key(self) -> str;               // "Y3f8a1c2b:game::Player"
    pub fn layout_hash(self) -> u64;
    pub fn kind(self) -> TypeKind;
    pub fn size(self) -> usize;
    pub fn align(self) -> usize;
    pub fn is_inspect(self) -> bool;
    pub fn fields(self) -> List<Field>;
    pub fn field_count(self) -> usize;
    pub fn field_at(self, i: usize) -> Field;
    pub fn field(self, name: str) -> Field | none;
    pub fn field_of_val(self, value: any, name: str) -> Field | none;
    pub fn variants(self) -> List<Variant>;
    pub fn variant_count(self) -> usize;
    pub fn variant_at(self, i: usize) -> Variant;
    pub fn variant(self, name: str) -> Variant | none;
    pub fn variant_of(self, value: any) -> Variant | none;
    pub fn generics(self) -> List<TypeRef>;
    pub fn generic_count(self) -> usize;
    pub fn generic_at(self, i: usize) -> TypeRef;
    pub fn element(self) -> TypeRef | none;     // list / array element, dict value
    pub fn key(self) -> TypeRef | none;         // dict key
    pub fn attr<A>(self) -> A | none;
    pub fn attrs(self) -> List<Attr>;
    pub fn holds(self, value: any) -> bool;
    pub fn new_default(self) -> any | none;
    pub fn build(self, fields: Dict<str, any>) throws (ReflectError) -> any;
    pub fn list_len(self, value: any) -> usize;
    pub fn list_get(self, value: any, i: usize) -> any | none;
    pub fn list_items(self, value: any) -> List<any> | none;
    pub fn list_from(self, items: List<any>) throws (ReflectError) -> any;
    pub fn dict_items(self, value: any) -> List<(any, any)> | none;
    pub fn dict_from(self, items: List<(any, any)>) throws (ReflectError) -> any;
    pub unsafe fn from_desc(at: usize) -> TypeInfo; // the descriptor at address at, which must stay mapped
}

impl TypeRef { pub fn name(self) -> str; pub fn info(self) -> TypeInfo | none; }

impl Field {
    pub fn name(self) -> str;                   // "0", "1", .. in a tuple
    pub fn renamed_from(self) -> str;
    pub fn type_ref(self) -> TypeRef;
    pub fn offset(self) -> usize;               // usize::MAX: no fixed place
    pub fn index(self) -> usize;
    pub fn is_pub(self) -> bool;
    pub fn has_default(self) -> bool;
    pub fn attr<A>(self) -> A | none;
    pub fn attrs(self) -> List<Attr>;
    pub fn get(self, owner: any) -> any | none;
    pub fn get_as<T>(self, owner: any) -> T | none;
    pub fn set(self, ref owner: any, value: any) -> bool;
}

impl Variant {
    pub fn name(self) -> str;
    pub fn index(self) -> usize;
    pub fn fields(self) -> List<Field>;
    pub fn field_count(self) -> usize;
    pub fn field_at(self, i: usize) -> Field;
    pub fn field(self, name: str) -> Field | none;
    pub fn attr<A>(self) -> A | none;
    pub fn attrs(self) -> List<Attr>;
    pub fn build(self, fields: Dict<str, any>) throws (ReflectError) -> any;
}

impl Attr { pub fn type_ref(self) -> TypeRef; pub fn get<A>(self) -> A | none; }
```

`std::nabion::to_value_any` and `from_value_any` ([3.8.5](../300-std/308-nabion.md#385-library-stdnabion)) are built on these.
