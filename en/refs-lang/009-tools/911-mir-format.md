# 9.11. nabi-mir file format

This section defines `<unit>.nabimir`, the file in which `nabic` hands the optimized nabi-mir of one codegen unit to a separately installed code generator ([8.11.13](../008-abi/811-backend-contract.md#81113-separately-installed-code-generators)). It holds everything the code generator contract ([8.11](../008-abi/811-backend-contract.md)) leaves to `nabic`: symbols and their linkage, type layouts, GC types, the safepoint facts, and the target.

> [!NOTE]
> Draft for 0.2. No 0.1 `nabic` writes or reads this format; the text below is a design, and it changes until a release implements it. `--emit=mir` keeps printing nabi-mir as text for people to read ([9.9.1](909-debug-tools.md#991-intermediate-output---emit)).

## 9.11.1. Binary, not text

The format is binary.

- Its reader is a code generator, not a person: the text of `--emit=mir` is for reading and leaves out what a code generator needs (layouts, symbols, GC types, debug facts). A text form would need a grammar and a parser in every code generator for no reader that benefits from it.
- The optimized nabi-mir of a unit that instantiates much of the standard library is large; integers as LEB128 and every string once keep the file small and quick to write for every unit of every build.
- LEB128 is what LLVM and other code generators already read (`decodeULEB128`), and what `nabic` already writes for its incremental caches.
- NABIHIR ([9.4](904-nabihir-bytecode.md)) is source text because its reader is `nabic`'s parser; this file's reader is a code generator, which has no parser for Nabi.

## 9.11.2. Encodings

| Notation | Encoding |
|----------|----------|
| `u8`, `u16`, `u32`, `u64` in the header and section table | Fixed width, little-endian |
| `u` | Unsigned LEB128 (at most 10 bytes, a value of at most 64 bits) |
| `s` | A signed integer, zigzag-mapped (`(v << 1) ^ (v >> 63)`) then as `u` |
| `w` | A 128-bit integer: its low 64 bits as `u`, then its high 64 bits as `u` |
| `b` | One byte: `0` or `1` |
| `str` | A `u` index into the string table (9.11.4); `0` is the empty string |
| `opt<T>` | `b`, then `T` when it is `1` |
| `list<T>` | `u` count, then that many `T` |
| `ty` | A `u` index into the type table (9.11.6) |
| `fn` | A `u` index into the function table (9.11.9) |
| `static` | A `u` index into the static table (9.11.8) |
| `local`, `block` | A `u`: the local's or block's number within its function |
| `pos` | A source position: `u` file (9.11.4; `0`: none), `u` line, `u` column (both 1-based; `0`: unknown) |

A position is where a panic or the debug information names it: the file is the path `path:line:col` shows, after `nabic` has mapped positions in expanded code back to the text they came from. Values in the file are in the target's byte order only where they are bytes of target memory (a constant byte array, 9.11.7); every other number is encoded as above.

## 9.11.3. Layout

```
header (16 bytes)
  magic          4 bytes   "NABM"
  major          u16
  minor          u16
  flags          u32       0
  sections       u32       number of section table entries
section table    sections x 24 bytes
  kind           u32
  reserved       u32       0
  offset         u64       from the start of the file
  size           u64
sections         at their offsets, in any order
```

Each section kind appears at most once. The kinds:

| Kind | Section | Content |
|------|---------|---------|
| 1 | strings | 9.11.4 |
| 2 | files | 9.11.4 |
| 3 | unit | 9.11.5 |
| 4 | target | 9.11.5 |
| 5 | structs | 9.11.6 |
| 6 | types | 9.11.6 |
| 7 | GC types | 9.11.7 |
| 8 | statics | 9.11.8 |
| 9 | functions | 9.11.9 |
| 10 | debug | 9.11.11 |

A reader skips a section of a kind it does not know. Sections 1 through 9 are present in every file; 10 only with `-g`.

## 9.11.4. Strings and files

**strings**: `u` count, then each string as `u` length and that many bytes of UTF-8. String 0 is the empty string. Each distinct string is stored once.

**files**: `list<str>`, the paths of the source files positions name, as panics print them. File index `i` (from 1) is the `i`-th entry.

## 9.11.5. Unit and target

**unit**:

| Field | Encoding | Meaning |
|-------|----------|---------|
| producer | `str` | `nabic <version> (<compiler id>)`, as in NABIHIR ([9.4.4](904-nabihir-bytecode.md#944-header-and-producer)) |
| name | `str` | The unit's name: its home module (`<root>` for the package root), or `<module>#k` for a bucket of a library's instances ([9.2.16](902-aot-pipeline.md#codegen-units)) |
| opt | `u8` | `0` `-O0`, `1` `-O1`, `2` `-O2`, `3` `-O3`, `4` `-Os`, `5` `-Oz` |
| flags | `u` | bit 0: overflow checks on; bit 1: `-g`; bit 2: describe variables by code range (`-g` above `-O0`, not wasm32) |
| source directory | `str` | The directory relative paths in `files` are relative to (debug information) |

**target**: the target ([0.6.5](../000-meta/006-platform-targets.md#065-target-specifications)) as the fields a code generator reads, then the whole specification:

| Field | Encoding |
|-------|----------|
| name | `str` (built-in name or the specification file's stem) |
| arch, os, env, vendor, family | `str` each, as `cfg` names them |
| endian | `u8`: `0` little, `1` big |
| pointer width, `wide_align`, max atomic width | `u` each (bits, bytes, bits) |
| C ABI | `u8`: `c_abi` (`Aarch64` 0, `X86_64SysV` 1, `X86_64Win` 2, `X86SysV` 3, `X86Win` 4, `ArmHf` 5, `Wasm32` 6, `Riscv64` 7, `Riscv32` 8, `Ppc64` 9, `S390x` 10, `Other` 11) |
| object format | `u8`: `0` Mach-O, `1` ELF, `2` COFF, `3` wasm |
| CPU, features | `str` each |
| C symbol prefix | `str` |
| stack probes | `u8`: `0` inline, `1` call, `2` none |
| threads | `b` |
| specification | `str`: the `TargetSpec { .. }` text `nabic --print target-spec` prints |

## 9.11.6. Types

**structs**: `list` of struct records, numbered from 0 in order; a type names a struct by this number.

| Field | Encoding | Meaning |
|-------|----------|---------|
| name | `str` | For the debug information (empty when unnamed) |
| key | `str` | The struct's build-independent name |
| packed | `b` | `#[repr(packed)]` |
| align | `u` | `#[repr(align(N))]`; `0` without |
| field names | `list<str>` | For the debug information |

**types**: `list` of type records. A record refers only to types before it, so the table can be read in one pass; equal types are one record.

| Tag (`u8`) | Type | Then |
|-----|------|------|
| 0 | `()` | |
| 1 | primitive | `u8` primitive (below) |
| 2 | unresolved word | |
| 3 | tuple | `list<ty>` fields |
| 4 | struct | `u` struct number, `list<ty>` fields |
| 5 | `any Tr` value | (the `(u32 tag, payload)` pair of [8.8.9](../008-abi/808-lowering-types.md#889-representation-of-any-slots)) |
| 6 | array | `ty` element, `u` length |
| 7 | function pointer | `list<ty>` parameters, `ty` result |
| 8 | GC reference | |
| 9 | union | `list<(u32 tag as u, ty)>` variants, `u` words, `u8` kind: `0` a sum's payload, `1` plain |

Primitives: `0` `i8`, `1` `i16`, `2` `i32`, `3` `i64`, `4` `isize`, `5` `u8`, `6` `u16`, `7` `u32`, `8` `u64`, `9` `usize`, `10` `f32`, `11` `f64`, `12` `bool`, `13` `c_char`, `14` `char`, `15` `str`.

After its tag and operands each record has its layout, as `nabic` computes it for the target ([8.11.5](../008-abi/811-backend-contract.md#8115-data-layout)):

| Field | Encoding | Meaning |
|-------|----------|---------|
| size, align | `u`, `u` | In bytes |
| offsets | `list<u>` | Tuples and structs: each field's offset; empty for other types |
| roots | `list<u>` | Offsets of the words a shadow-stack frame registers for a local of this type, ascending ([8.11.6](../008-abi/811-backend-contract.md#8116-gc-allocation-roots-and-barriers)); empty when the type holds no GC reference |
| barrier | `b` | A store of this type into a GC object needs a write barrier |

A function value is the tuple `(code, env)` of a function pointer and a GC reference; a throwing function's carrier is the tuple `(is_err, payload)`; a sum is the tuple `(i32, union)` of kind 0. The type table has no other kinds.

## 9.11.7. GC types and constants

**GC types**: `list` of the unit's GC types, each:

| Field | Encoding | Meaning |
|-------|----------|---------|
| id | `u` | The type id, when the code may use it as a constant; `0` otherwise |
| symbol | `str` | In a unit build, the `_Y<pkg>_gctype.<hash>` symbol holding the id ([8.11.3](../008-abi/811-backend-contract.md#8113-symbols-and-linkage)); empty otherwise |
| table | `list<u>` | The ScanTable's `u32` words ([8.2.3](../008-abi/802-scan-table.md#823-serialization-format)), for reading only: the shared unit emits and registers it |

An allocating rvalue names its GC type by its index in this list.

**Constants** (`const`), by tag `u8`:

| Tag | Constant | Then |
|-----|----------|------|
| 0 | `()` | |
| 1 | `bool` | `b` |
| 2 | signed integer | `w` value, `u8` primitive |
| 3 | unsigned integer | `w` value, `u8` primitive |
| 4 | float | `u` bits (an `f32`'s in the low 32), `u8` primitive |
| 5 | `char` | `u` code point |
| 6 | string | `str` |
| 7 | bytes | `u` length, that many bytes |
| 8 | none | |

A constant takes the type of the place it is used in: an integer constant used as a float is that number converted, and `()` and none are the zero value of that type.

## 9.11.8. Statics

**statics**: `list` of static records:

| Field | Encoding | Meaning |
|-------|----------|---------|
| name | `str` | Its item path |
| symbol | `str` | Its linker symbol ([8.11.3](../008-abi/811-backend-contract.md#8113-symbols-and-linkage)) |
| type | `ty` | |
| flags | `u` | bit 0: `mut`; bit 1: defined outside the program (`extern`); bit 2: reached through `__imp_<symbol>` (a DLL's, Windows); bit 3: lazy ([8.11.10](../008-abi/811-backend-contract.md#81110-statics)); bit 4: defined in this program's own objects (else through the GOT where the format has one) |
| init | `opt<const>` | The initial value, for reading only: the shared unit defines the storage |

## 9.11.9. Functions

**functions**: `list` of function records. A function the unit defines has a body; any other function the unit calls or takes the address of is a declaration (no blocks).

| Field | Encoding | Meaning |
|-------|----------|---------|
| name | `str` | The nabi-mir name |
| path | `str` | The item path (a generic instance's: its generic's) |
| symbol | `str` | The body's symbol |
| C entry | `str` | `extern "C"` definitions: the C entry's symbol; else empty |
| thunk | `str` | The function-value thunk's symbol (`<symbol>.v`) when the unit takes the function's address and it takes `nabi_tls`; else empty |
| define | `u8` | `0` declaration, `1` local, `2` global, `3` weak global |
| flags | `u` | below |
| ABI | `u8`, then | `0` Nabi; `1` C: `b` variadic, `b` `extern "system"` |
| C classification | `opt<cabi>` | For a C-ABI function and an `extern "nabi"` declaration that needs one ([8.11.4](../008-abi/811-backend-contract.md#8114-calling-conventions)) |
| wasm import | `opt<(str, str)>` | Module and name of a wasm32 import |
| position | `pos` | The declaration |
| parameters | `list<local>` | |
| parameter types, result | `list<ty>`, `ty` | |
| locals | `list<(local, str name, ty)>` | Every local, the parameters included; an empty name for one the debug information leaves out |
| pins | `list<local>` | Locals that stay registered until their `StorageDead` |
| blocks | `list<block record>` | The first is the entry |
| debug | `opt<debug record>` | 9.11.11; present with `-g` |

Function flags:

| Bit | Meaning |
|-----|---------|
| 0 | Takes the `nabi_tls` address as its hidden last argument |
| 1 | Never reaches a safepoint: no frame, no poll, and a call to it is not a safepoint ([8.11.7](../008-abi/811-backend-contract.md#8117-safepoints-and-polls)) |
| 2 | Re-entrant: polls before its first safepoint |
| 3 | Entered other than by a direct Nabi call: polls at entry |
| 4 | `#[track_caller]`: the last parameter is the caller's position (`str`) |
| 5 | `#[no_safepoint]` runtime declaration |
| 6 | `#[cold]` |
| 7 | `#[inline(never)]` |
| 8 | The user's `main` (the shared unit's C `main` calls it) |
| 9 | The `any` dispatcher |

The C classification `cabi`: `u8` result (`0` direct, `1` coerced, then a `ctype`; `2` through `sret`), `list` of arguments (`0` direct, `1` coerced, then a `ctype`; `2` by reference, then `b` byval), `b` variadic, `b` `str` as a `(ptr, len)` pair, `b` stdcall. A `ctype` is `u8` `0` scalar then `cscalar`, `1` `u` count and `cscalar` (consecutive registers), `2` `list<cscalar>` (a struct of naturally aligned fields), `3` `list<(u offset, cscalar)>`. A `cscalar` is `u8` `0` integer then `u` bits, `1` `f32`, `2` `f64`, `3` two `f32` in one register, `4` pointer.

**Block record**: `block` number, `list<local>` parameters, `u` flags (bit 0: a polled back edge leaves this block, so a poll runs before its terminator), `list<statement>`, terminator.

**Operands**: `u8` `0` then `const`, or `1` then `local`.

**Statements**, by tag `u8`:

| Tag | Statement | Then |
|-----|-----------|------|
| 0 | `local = rvalue` | `local`, rvalue |
| 1 | array element write | `local` array, `u` byte offset, operand index, operand value, `ty` element, `u` length |
| 2 | storage dead | `local` |
| 3 | buffer element store | operand buffer, operand index, operand value, `ty` element |
| 4 | position | `pos`, `u` scope, `u` parent scope |
| 5 | raw store | operand address, operand value, `ty`, `b` volatile |
| 6 | heap field store | operand object, `ty` payload (a tuple), `u` field, operand value |
| 7 | static store | `static`, operand value, `ty` |

**Rvalues**, by tag `u8`:

| Tag | Rvalue | Then |
|-----|--------|------|
| 0 | use | operand |
| 1 | binary | `u8` op, `ty` operand type, operand, operand |
| 2 | unary | `u8` op, `ty` operand type, operand |
| 3 | call | `fn`, `list<operand>`, `ty` result |
| 4 | struct value | `ty`, `list<operand>` fields |
| 5 | field read | operand, `u` field, `ty` field type |
| 6 | cast | operand, `ty` from, `ty` to |
| 7 | bitcast | operand, `ty` from, `ty` to |
| 8 | with field | operand, `ty` struct, `u` field, operand value |
| 9 | array value | `ty` element, `list<operand>` |
| 10 | array element read | `local` array, `u` byte offset, operand index, `ty` element, `u` length |
| 11 | function address | `fn`, `ty` |
| 12 | static read | `static`, `ty` |
| 13 | box | operand, `ty`, `u` GC type + 1 (`0`: the value is the payload word, nothing is allocated) |
| 14 | buffer allocation | `ty` element, operand length, `u` GC type |
| 15 | raw load | operand address, `ty`, `b` volatile |
| 16 | local address | `local` |
| 17 | static address | `static` |
| 18 | size or alignment | `ty`, `b` alignment |
| 19 | buffer element read | operand buffer, operand index, `ty` element |
| 20 | buffer atomic | `u8` op, operand buffer, operand index, `list<operand>`, `ty` element, `u8` order, `u8` failure order |
| 21 | box read | operand, `ty` |
| 22 | heap allocation | `ty` payload, `u` GC type |
| 23 | heap field read | operand object, `ty` payload, `u` field |
| 24 | indirect call | operand callee, `list<operand>`, `list<ty>` parameters, `ty` result, ABI as in the function record, `opt<cabi>` |

Operators (`u8`): `0` `+`, `1` `-`, `2` `*`, `3` `/`, `4` `%`, `6` negation, `7` `!`, `8` `&`, `9` `|`, `10` `^`, `11` `<<`, `12` `>>`, `13` `==`, `14` `!=`, `15` `<`, `16` `<=`, `17` `>`, `18` `>=`, `19` `&&`, `20` `||`, `21` logical not, `22` square root, `23` wrapping `+`, `24` wrapping `-`, `25` wrapping `*`, `26` count ones, `27` leading zeros, `28` trailing zeros, `29` byte swap. `5` is reserved (the floored `%%`, which never reaches a code generator). A comparison's result type is that of its destination.

Atomic operations: `0` load, `1` store, `2` compare-exchange (arguments: expected, new), `3` exchange, `4` add, `5` sub, `6` and, `7` or, `8` xor. Orders: `0` relaxed, `1` acquire, `2` release, `3` acq_rel, `4` seq_cst.

**Terminators**, by tag `u8`: `0` jump (`block`, `list<operand>` arguments for its parameters), `1` branch (operand condition, `block` and `list<operand>` when true, `block` and `list<operand>` when false), `2` return (`opt<operand>`), `3` unreachable.

## 9.11.10. Meaning

Each statement, rvalue and terminator means what it means in nabi-mir; [8.11](../008-abi/811-backend-contract.md) gives what a code generator adds to it (checks, barriers, roots, polls, allocation). Data a nabi-mir value refers to that is not in the file is the shared unit's or another object's, reached by symbol.

## 9.11.11. Debug information

The debug record of a function (with `-g`):

| Field | Encoding | Meaning |
|-------|----------|---------|
| local types | `list<(local, dty)>` | What the debugger shows of named locals whose type alone does not tell it |
| bindings | `list` | Named locals whose assignments the optimizer removed: `local`, `opt<operand>` the value they hold wherever they are visible, `list<opt<operand>>` the fields of a struct built in place, `u` line, `u` scope, `u` parent scope |
| inlined | `list` | Bodies inlined: `u` scope, `u` parent scope, `str` callee path, `pos` declaration, `pos` call, `list<local>` the copies of its parameters |
| results | `list` | Locals an inlined body's result is assigned to: `local`, `u` line, `u` scope, `u` parent scope |

The debug section holds `list<(u struct, list<dty>)>`, the fields' debug types by struct, and `list<(u tag, str name, dty)>`, the types stored in `any` values by tag. A `dty` is `u8` `0` then `ty` (as the type says); `1` sum: `str` name, `ty`, `list<(u tag, str name, dty)>`; `2` GC buffer: `str` name, `dty` element; `3` `any`; `4` boxed value: `str` name, `dty`; `5` captured variable's cell: `dty`; `6` function value: `str` name, `ty`, `dty` environment.

## 9.11.12. Versions and compatibility

`major` and `minor` in the header are the format's version.

- A reader accepts a file whose `major` it reads. Within one `major`, a higher `minor` only adds section kinds, which an older reader skips; the encoding of an existing section, and so every statement, rvalue, type and constant tag, is unchanged.
- Any change to an existing section is a new `major`: a new statement, rvalue or type kind, a new field, a changed meaning. A reader rejects a `major` it does not read, naming both versions (exit status 2 of [8.11.13](../008-abi/811-backend-contract.md#81113-separately-installed-code-generators)).
- While `major` is `0` (this draft), a reader accepts only the exact `major.minor` it was built for.
- A component's version is its toolchain's ([8.11.13](../008-abi/811-backend-contract.md#81113-separately-installed-code-generators)), so a toolchain's `nabic` and its components always agree; the version rules matter for a component built outside the toolchain.

Writing is deterministic: the same unit gives the same bytes. Strings, types and functions are numbered in the order the writer first meets them, walking the unit's functions in nabi-mir order; maps are written sorted by key.
