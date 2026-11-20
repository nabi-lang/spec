# 6.10. Reflection

The standard library provides two levels of reflection traits: `Reflect` and `Inspect`.

```
Reflect    automatic: shallow metadata every type carries for free
Inspect    opt-in via #[derive(Inspect)]: rich field/method description
```

The split into two levels controls binary size while still giving every type, for free, what is essential for `any` slots, sum types, and REPL auto-printing.

## 6.10.1. `Reflect` - automatically implemented trait

`Reflect` is minimal type introspection attached automatically to every type. Without the user writing `#[derive(Reflect)]`, the compiler synthesizes an impl for every type.

```nabi
trait Reflect {
    // static: called directly on the type
    fn type_name() -> str;
    fn size() -> usize;
    fn align() -> usize;

    // instance: runtime dispatch on any slots / sum types
    fn type_name_of_val(self) -> str;
    fn size_of_val(self) -> usize;
}
```

```nabi
i32::type_name() == "i32"
i32::size()      == 4
i32::align()     == 4

let x: any = 42i32;
x.type_name_of_val() == "i32"    // runtime type tag check

let v: i32 | f64 = pick();
v.type_name_of_val()              // "i32" or "f64" (sum type variant check)
```

Being automatic, it requires no user code. The compiler already knows the `type_name`, `size`, and `align` of every type and synthesizes them at zero cost.

`Reflect` is provided by the compiler for every type; there is no impl in source.

- A handwritten `impl Reflect for T` is an error. The bound `T: Reflect` always holds.
- If a type has its own method of the same name, such as `size()` / `type_name()`, that method hides the reflection method.
- Names (`type_name`) carry no module path (`List<i64>`, `Config`). The static name of a sum type is written in canonical order ([2.6.4](../002-types/206-sum-types.md#264-set-semantics)) (`f64 | i32`, with `none` last). Instance forms such as `type_name_of_val(self)` report the member held at runtime (for a sum) or the contents of an `any`.
- `core::reflect::cast<T>(value: any) -> T | none` extracts an `any` as `T` by the same rules as `set_field`. A type's methods and fields are `T::methods()` / `T::fields()` ([6.10.2](#6102-inspect---opt-in-trait)); there is no free function for them.

## 6.10.2. `Inspect` - opt-in trait

`Inspect` is rich reflection opted into with `#[derive(Inspect)]`. It exposes the type's description (`TypeInfo`, 6.10.7), field names, dynamic field access and modification, and method descriptions.

```nabi
trait Inspect: Reflect {
    fn type_info() -> TypeInfo;                             // the description (6.10.7)
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // private fields too
    fn set_field(ref self, name: str, value: any) -> bool;  // whether it succeeded

    // Every method has a default body over `type_info()`.
    fn fields() -> List<FieldInfo>;          // the field table of Self
    fn methods() -> List<MethodInfo>;        // the method table of Self
    fn variant_names() -> List<str>;         // empty for a struct
    fn variant_name(self) -> str;            // a struct: self.type_name_of_val()
    fn field_names_of_val(self) -> List<str>; // the fields self holds
}

struct FieldInfo {
    name: str;                 // "0", "1", .. for a tuple field
    type_name: str;            // instantiated: f64 for `v: T` in Box<f64>
    is_pub: bool;              // true only for a plain `pub` field
}

struct MethodInfo {
    name: str;
    arity: usize;              // number of arguments excluding self
    is_static: bool;           // a method that does not take self
    arg_types: List<str>;      // argument type names excluding self
    return_type: str;
}
```

- `get_field` returns the field's value as `any`, or `none` for a nonexistent name. Private fields can be read too.
- `set_field` assigns and returns `true` when the value has **exactly the same** type as the field, or when an integer of a different integer type **fits without loss** (`u.set_field("id", 42)` into a `u64` field). Otherwise it does not assign and returns `false`. There are no other conversions (e.g. `i32` → `f64`).
- `methods()` returns **only the inherent methods** of the type (excluding trait methods), sorted by name. Methods of a generic `impl` show types as written (`T`).
- `fields()` lists every field in declaration order with its type name and whether it is `pub`. **Private fields are listed too**: deriving `Inspect` is the type author's consent, the same as for `get_field`. The type name is the instantiated one (`Box<f64>` reports `v: f64`, `items: List<f64>`), with no module path, as `type_name` writes it.
- An `enum` has no fields at the type level (`field_names()` and `fields()` are empty lists). `variant_names()` lists its variants in declaration order. On a value, `variant_name()` is the variant it holds (`"Circle"`), `field_names_of_val()` that variant's fields (`"0"`, `"1"` for a tuple variant, the names for a named one), and `get_field` reads them. A variant's payload is read-only through `Inspect`: `set_field` on an enum is `false`.
- On a struct, `variant_names()` is an empty list, `variant_name()` is its type name (`type_name_of_val()`), and `field_names_of_val()` is its fields.
- Every method has a default body; `get_field`, `set_field` and the name lists read `type_info()`. The derive writes only the accessors `type_info()` calls (6.10.7). A hand-written `impl Inspect` may override any method.
- Calling an `Inspect` method on a type without the derive is an error whose help says what to write: ``add `#[derive(Inspect)]` to `Point` ``.

Usage example:

```nabi
#[derive(Inspect)]
struct Config { host: str; port: i32; }

let c = Config { host: "localhost", port: 8080 };
Config::field_names()              // ["host", "port"]
c.get_field("port")                // 8080 inside an any
c.set_field("port", 9090)          // true (success)
c.set_field("xxx", 1)              // false (no such field)
Config::fields()[1].type_name      // "i32"
```

`Inspect` is opt-in because its metadata must be stored in the binary. Attaching it to every type automatically would impose too large a binary size cost. Even with the derive, nothing is stored until the program asks: a program that never calls `type_info()`, `info::<T>()` or a method built on them builds to the same code as one without the derive.

`#[derive(Inspect)]` is implemented by the [meta block](608-meta-blocks.md) `#[impl_derive(Inspect)]` in `core::reflect`, a pattern in which the meta-system itself implements a standard library trait ([stdlib 2.12.3](../../refs-standard-libraries/200-core/212-reflect.md#2123-inspect---opt-in-fields--methods)).

## 6.10.3. REPL / Jupyter auto-printing

When the REPL prints a bare last expression it uses `Display`, else `Debug`; a trailing `?` uses `Debug`; `:type` prints the expression's static type, found by type checking without running it ([9.7.7](../009-tools/907-jit-cells.md#977-visualization--output)).

```
>>> let x = 42i32;
>>> x
42

>>> [1, 2, 3]
[1, 2, 3]

>>> Config { host: "h", port: 80 }
Config { host: "h", port: 80 }

>>> Opaque { id: 1 }
Opaque
```

A bare expression uses `Display`, else `Debug`, and a trailing `?` uses `Debug`. A value without either (`Debug` for a trailing `?`) prints only its type name. This behavior is used by REPL and Jupyter cell sessions ([9.7](../009-tools/907-jit-cells.md)).

## 6.10.4. No global `typeof` / `sizeof` functions

Introspection that other languages expose through keywords such as `typeof` or `sizeof` is provided in Nabi only as trait methods.

```nabi
// nabi: method calls
T::size()
x.type_name_of_val()

// other languages: keywords
sizeof(T)        // C / C++
typeof(x)        // JS / TS
```

The reason is that this adds no language syntax. The single `Reflect` trait covers the entire introspection surface.

## 6.10.5. Dispatch on `any` slots and sum types

`any` slots and sum types of the form `T1 | T2` determine the actual type at runtime, so the `type_name_of_val` call is important.

```nabi
fn handle(x: any) {
    if x.type_name_of_val() == "i32" { handle_int(x); }
    else if x.type_name_of_val() == "str" { handle_str(x); }
}

fn classify(v: i32 | f64) -> str {
    match v {
        n: i32 => "integer",
        f: f64 => "float",
    }
}
```

The compiler desugars type-test patterns (`v: T => ...`) into `Reflect` tag checks.

## 6.10.6. Relation between the levels

As the notation `Inspect: Reflect` shows, `Inspect` has `Reflect` as a supertrait. A type implementing `Inspect` therefore always implements `Reflect` too, trivially, since `Reflect` is automatic anyway.

```
every type           Reflect          (automatic, zero cost)
+ #[derive(Inspect)] Inspect+Reflect  (opt-in, metadata embedded)
```

## 6.10.7. `TypeInfo` - the type's description

`T::type_info()` (or `core::reflect::info::<T>()` for any `T`) returns a `TypeInfo`: the compiler's description of `T` in read-only data ([8.13](../008-abi/813-type-descriptor.md)). For an `Inspect` type it lists the fields with their types and byte offsets, the variants of an enum with theirs, the type arguments of an instance, and the runtime attributes (6.10.8); it also reaches the values through `any`.

```nabi
use core::reflect::{Field, TypeInfo, TypeKind};

#[derive(Inspect)]
struct Enemy { name: str; hp: i32 = 100; }

let t = Enemy::type_info();
t.name()                           // "Enemy"
t.kind()                           // TypeKind::Struct
t.size() == Enemy::size()          // true
t.field("hp")                      // a Field: name, type_ref, offset, flags
let e: any = Enemy { name: "orc" };
let hp = t.field("hp") ?? panic("no hp");
hp.get(e)                          // 100 inside an any
```

- Reading a description allocates nothing: `name`, `kind`, `size`, `field_count`, `field_at`, `field(name)`, `variant_at`, `attr::<A>()`. Methods that return a `List` (`fields()`, `variants()`) allocate the list.
- `Field::get(owner: any)` returns the field's value, `none` when `owner` is not of the field's type. `Field::get_as::<T>(owner)` is the value as a `T`. `Field::set(ref owner: any, value: any)` follows `set_field`'s rules and stores a changed copy into `owner`: a struct is a value.
- An enum's fields are its variants' (`Variant::fields()`); `TypeInfo::variant_of(value)` is the variant a value holds. A variant's field reads `none` on a value of another variant. A variant's payload is read-only.
- `TypeInfo::build(fields: Dict<str, any>)` makes a value from named fields; a field left out takes its `= expr` default, and a field without one is an error naming it. `Variant::build` does the same for an enum. `new_default()` is the value with every field at its default, or `none`.
- A list, array or dict's `TypeInfo` reaches its items: `list_items`, `list_len`, `list_get`, `list_from`, `dict_items`, `dict_from`; `element()` and `key()` are the item types.
- `info_of(value: any)` is the description of the type `value` holds, and `find(path)` the one at a path such as `"game::Player"`; both are `none` when the program describes no such type. A type is described when the program asks for it or for a type made of it.
- Two `TypeInfo`s are equal when they describe the same type (by key, [8.13.2](../008-abi/813-type-descriptor.md#8132-keys)), even from different modules.
- `#[inspect(skip)]` on a field leaves it out of the description and of `field_names`, `get_field` and `set_field`.

## 6.10.8. Runtime attributes

A struct marked `#[runtime_attribute(name)]` declares the attribute `#[name(..)]`. Written on a type, a field or an enum variant, its arguments are checked like a literal of that struct, positionally in field order or by field name, and the value is kept in the description. `attr::<A>()` on a `TypeInfo`, `Field` or `Variant` reads it back typed.

```nabi
/// Shown as a slider in the editor.
#[runtime_attribute(range)]
pub struct Range { pub min: f64; pub max: f64; }

#[derive(Inspect)]
struct Enemy {
    #[range(0, 100)] hp: i32 = 100;
    #[range(min = 0.5, max = 4.0)] speed: f64 = 1.0;
}

let hp = Enemy::type_info().field("hp") ?? panic("no hp");
hp.attr::<Range>()                 // Range { min: 0.0, max: 100.0 }
```

- An attribute struct's fields are numbers, `bool`, `char` or `str`; an integer literal fills a float field.
- A wrong argument is an error that says what to write: ``#[range] takes `min: f64, max: f64`; write `#[range(0, 10)]` ``. A name close to a declared one gets ``no attribute `rnage`; did you mean `range` (`Range`)?``.
- When two packages declare the same name, the one of the package that uses it wins; otherwise the use is an error naming both.
- Only attributes declared this way are kept. `doc`, `cfg` and the other built-in attributes never are.

`#[renamed_from("old")]` on a field records the field's earlier name in its description (`Field::renamed_from`), for tools that read data written under the old name.

## 6.10.9. With plugins and hot reload

A description lives in the module that defines the type. Every module registers its descriptions with the runtime when it starts or is loaded, and `info_of` and `find` look them up there; `find` returns the newest module's description of a path. Since equality is by key, two modules' descriptions of one generic instance compare equal.
