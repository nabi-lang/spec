# 8.13. Type descriptor ABI

A *type descriptor* (`TypeDesc`) is the read-only record of one type. Reflection reads it ([6.10](../006-meta-system/610-reflection.md)), and the module loader uses it to check that two modules agree on a type ([8.12](812-module-abi.md)). The layout is defined once in `nabir_abi::typedesc`; the layout rules of [8.12.2](812-module-abi.md#8122-layout-rules) apply.

## 8.13.1. When descriptors exist

The compiler emits a descriptor for a type only when the program asks for it: `T::type_info()` or `core::reflect::info::<T>()` of an `Inspect` type, and recursively the descriptors of the types its fields, variants and type arguments name. A program that never asks has none, and its objects are byte for byte what they are without reflection.

Each descriptor is a hidden, weak symbol in read-only data, so the copies two objects of one program emit fold into one. When a program has descriptors, its `nabi.register_types` passes them to `nabi_types_register` at startup, with their compile-time `any` tags.

## 8.13.2. Keys

The *key* of a type names it across modules:

- a struct, enum or union: `Y<pkg8>:` and its full path, `Y3f8a1c2b:game::Player`, where `pkg8` is the defining package's hash as in [9.5](../009-tools/905-symbol-mangling.md);
- a generic instance: the base key with the argument keys in `<..>`, `Y..:core::collections::List<Y..:game::Player>`;
- a primitive, `str`, `any` and `none`: its name;
- a tuple, a sum, an array and a function type: written from the keys of their parts as in source, with sum members in canonical order.

`key_hash` is the first 8 bytes of SHA-256 of the key, little-endian. Two descriptors are of the same type exactly when their keys are equal.

## 8.13.3. `TypeDesc`

| Field | Type | Meaning |
|-------|------|---------|
| `magic` | `u32` | `"NABT"` as a little-endian `u32` |
| `version` | `u16` | 1 |
| `kind` | `u16` | `STRUCT` 1, `TUPLE_STRUCT` 2, `UNIT` 3, `ENUM` 4, `UNION` 5, `SUM` 6, `PRIM` 7, `STR` 8, `LIST` 9, `DICT` 10, `ARRAY` 11, `TUPLE` 12, `FN` 13, `ANY` 14, `OPAQUE` 15 |
| `flags` | `u32` | `INSPECT` 1 (fields, attributes and `ops` filled), `LAYOUT_ONLY` 2, `INSTANCE` 4, `HAS_DEFAULT` 8 |
| `tag_index` | `u32` | Index into the module's `Tags` ([8.12.5](812-module-abi.md#8125-tags)); `u32::MAX` when none |
| `key_hash` | `u64` | 8.13.2 |
| `layout_hash` | `u64` | A hash of the size, alignment, field offsets and type keys, variant tags and GC words |
| `size` | `u64` | As `T::size()` |
| `align` | `u32` | As `T::align()` |
| `reserved0` | `u32` | 0 |
| `key` | `Str` | 8.13.2 |
| `name` | `Str` | As `T::type_name()`: `Player`, `List<f64>` |
| `path` | `Str` | `game::Player`; empty for a type without one |
| `fields` | `Slice<FieldDesc>` | A struct's fields in declaration order; empty for an enum |
| `variants` | `Slice<VariantDesc>` | An enum's variants in declaration order |
| `generics` | `Slice<TypeRef>` | An instance's type arguments; a sum's members; a tuple's parts; a list's or array's element; a dict's key, then value |
| `attrs` | `Slice<AttrDesc>` | Runtime attributes on the type ([6.10.8](../006-meta-system/610-reflection.md#6108-runtime-attributes)) |
| `ops` | `*const TypeOps` | Dynamic access (8.13.5), or null |
| `reserved1` | `usize` | 0 |

## 8.13.4. Parts

| Structure | Fields |
|-----------|--------|
| `TypeRef` | `name: Str`, `key_hash: u64`, `desc: *const TypeDesc` (null when none was emitted), `kind: usize` |
| `FieldDesc` | `name: Str` (`"0"`, `"1"`, .. in a tuple struct or variant), `renamed_from: Str` (empty unless `#[renamed_from(..)]`), `ty: TypeRef`, `attrs: Slice<AttrDesc>`, `offset: u64` (from the start of the value; `u64::MAX` when the field has no fixed place), `flags: u32` (`PUB` 1, `HAS_DEFAULT` 2, `SKIP` 4), `index: u32`, `init_default: FnValue` (hot builds only) |
| `VariantDesc` | `name: Str`, `fields: Slice<FieldDesc>`, `tag: u32` (the stored tag), `index: u32`, `attrs: Slice<AttrDesc>` |
| `AttrDesc` | `ty: TypeRef` (the attribute struct), `value: *const u8` (a constant in that struct's layout), `reserved: usize` |
| `FnValue` | `code: usize`, `env: usize`: a Nabi function value ([8.8](808-lowering-types.md)); `env` is 0 |

## 8.13.5. `TypeOps`

`TypeOps` holds Nabi function values (`{0, 0}` where the type has none), called with the Nabi calling convention:

| Field | Signature | Meaning |
|-------|-----------|---------|
| `get` | `fn(owner: any, field: usize) -> any \| none` | The field's value, or `none` when `owner` is not of this type. An enum's field number is `variant << 16 \| index` and answers `none` for another variant; `usize::MAX` answers the index of the variant `owner` holds, as a `usize`. |
| `set` | `fn(owner: any, field: usize, value: any) -> any \| none` | A copy of `owner` with the field set, or `none` when `owner` or `value` does not fit |
| `build` | `fn(variant: usize, fields: Dict<str, any>) -> (any \| none, str)` | A new value from named fields (an enum's `variant`, 0 for a struct), a missing field taking its default; or `none` and a message naming what is wrong |
| `new_default` | `fn() -> any \| none` | Reserved (`{0, 0}`); `build` with no fields makes the value with every field at its default |
| `list` | `*const ListOps` | Lists and arrays: `len: fn(any) -> usize`, `get: fn(any, usize) -> any \| none`, `items: fn(any) -> List<any> \| none`, `from: fn(List<any>) -> any \| none` |
| `dict` | `*const DictOps` | Dicts: `len: fn(any) -> usize`, `items: fn(any) -> List<(any, any)> \| none`, `from: fn(List<(any, any)>) -> any \| none`, `reserved` |
