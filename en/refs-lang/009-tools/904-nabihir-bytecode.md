# 9.4. NABIHIR format

This section defines the interface file `<name>.nabihir` of a prebuilt library package. An [NABIHIR](../000-meta/007-glossary.md#nabihir-nabi-hir-bytecode) file sits next to that package's `lib<name>.a` ([Package archive](903-package-archive.md)) and is not a member of the archive.

## 9.4.1. Motivation

Monomorphization in the [AOT build pipeline](902-aot-pipeline.md) [specializes](../000-meta/007-glossary.md#specialize) by the concrete types at the call site. So when another package calls a [generic](../000-meta/007-glossary.md#generic) `fn` of a package, specialization must happen at that call site. The object code of [concrete fns](../000-meta/007-glossary.md#concrete-fn--generic-fn) alone is not enough, so a separate interface is provided for downstream to read.

The input of monomorphization is the AST. The NABIHIR body is therefore **stripped package source**, which the downstream parser reads.

## 9.4.2. No external libraries

Under the zero-dependency principle no serialization library is used either; NABIHIR is designed from scratch.

## 9.4.3. Overall layout

```
NABIHIR file (little-endian):
+-----------------------------------+
|  Header                           |  (16 bytes fixed)
|    magic       "NABH" (4 bytes)   |
|    version     u32  (= 1)         |
|    flags       u32                |
|    body_offset u32                |
+-----------------------------------+
|  Producer string        str       |  "nabic <version> (<compiler id>)"
+-----------------------------------+
|  Meta                             |
|    name                 str       |  mount name (--extern <name>=)
|    hash                 str       |  package hash (0.3.3)
|    source dir           str       |
|    gc type offset       varint    |
|    gc type count        varint    |
|    features  varint n, n × str    |
+-----------------------------------+
|  Body (at body_offset)            |
|    file count           varint    |
|    file × n: path str, text str   |
+-----------------------------------+
|  Summaries                        |
|    count                varint    |
|    n × (function str, summary str) |
+-----------------------------------+
|  Call summaries                   |
|    count                varint    |
|    n × (function str,             |
|         flags varint,             |
|         keeps_none varint,        |
|         returns varint)           |
+-----------------------------------+
```

`str` is UTF-8 bytes preceded by a PrefixVarInt length (9.4.5). There are no string / type / item sections.

## 9.4.4. Header and producer

| Field         | Size | Meaning                                                      |
|---------------|------|--------------------------------------------------------------|
| `magic`       | 4    | `"NABH"`                                                     |
| `version`     | u32  | Format version (`1`). Bumped when the layout or body content changes |
| `flags`       | u32  | Reserved (0)                                                 |
| `body_offset` | u32  | Start offset of the body                                     |

The producer string is `nabic <version> (<compiler id>)`. The compiler id is a stamp of that `nabic` executable that changes whenever it is written or replaced: its size, modification time, inode and change time (`<size>-<mtime>-<inode>-<ctime>.<ctime_nsec>`, hexadecimal; on Windows the file id and creation time take the place of the inode and change time).

## 9.4.5. VarInt encoding

A PrefixVarInt, a variant of LEB128. The number of leading 1 bits of the first byte is the number of following bytes, and the value occupies the remaining bits in big-endian order.

```
1-byte int  : 0xxxxxxx                    (0 ~ 127)
2-byte int  : 10xxxxxx xxxxxxxx           (0 ~ 16383)
3-byte int  : 110xxxxx xxxxxxxx xxxxxxxx  (0 ~ 2097151)
...
```

## 9.4.6. Meta

| Field            | Meaning                                                                           |
|------------------|-----------------------------------------------------------------------------------|
| name             | The package's mount name. It appears in symbols, so it must equal `--extern <name>=` |
| hash             | Package hash ([0.3.3](../000-meta/003-source-units.md#033-package-identifier)). The value used in symbol mangling. For the standard library package `nabi` it is a `pkg=hash;...` list, one entry per tier |
| source dir       | The directory holding the package source at build time. The base for file paths in the body; downstream reads the body, not the disk |
| gc type offset / count | This package's GC type id range `[offset, offset + count)` (9.4.8)          |
| features         | The `feature` cfg values the package was built with. Downstream parses the body with the build's other cfg and these features |

## 9.4.7. Body

The body holds **every source file** of the package as (path, text) pairs. Each file is stripped as follows.

- The bodies of concrete functions are cut down to `;`. Newlines are kept, so diagnostic line numbers match the original. These functions are linked as archive symbols.
- Bodies that downstream must monomorphize or inline are kept: generic functions, `async` / `yield` functions, `#[inline]` functions, `extern "C"` definitions, default methods of traits, and methods of generic `impl`s.
- Declarations such as signatures, types, traits, `impl`s and `use`s are left as is.
- `include_str!` / `include_bytes!` in the kept text are replaced by the literal they read (line breaks kept), so downstream reads no included file and a moved library still works.

After the source files come the `.wit` files the package's `ffi::wit` items read ([stdlib 1.8](../../refs-standard-libraries/100-bare/108-wit.md)), with their `deps/`, whole. Their paths are relative to the source dir and may leave it through `..`. Downstream reads WIT from the body, not the disk.

After the body come the hand-off summaries ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)) of the package's functions. A summary is lines of tab-separated fields:

- `g <root> <mutex|rwlock|channel> <line> <function> <file>`: hands `<root>` over;
- `s <root> <root>`: one object;
- `i <root> <root>`: the first going away takes the second along;
- `t <guard parameter> <root>`: tied to that guard.
- `x <tag|*> [<root>]`: on the exits whose returned value has that tag (`*`: not known; a `bool`'s 0 / 1, a sum or enum variant, a carrier's 1 for an error, 0 for a value, 2 + the value's tag), `<root>` is among those handed over; with no `x` lines every exit hands over all `g` roots.

A root is a parameter index or `r` (the return value), then a `.`-separated field path (a tuple index has bit 31 set). An entry named `#fn` (`#fne`: a closure, environment first) followed by U+001F-separated parameter type keys joins the summaries of the functions used as values with those parameter types.

Last come the call summaries: what a call of a function whose code is in the archive does, so that an optimized downstream build need not treat it as able to do anything. Entries are sorted by function name (its item path, as downstream declares it); a function with nothing known has none, and neither has a generic instance (downstream makes its own) or an `any` dispatcher (the program's replaces it). Each fact holds of the archive's code:

| Field        | Meaning |
|--------------|---------|
| `flags`      | bit 0 `readonly`: writes no memory its caller can read (it may still allocate, reach a safepoint and panic); bit 1 `quiet`: never reaches a GC safepoint (no allocation, poll or blocking wait), so a call does not move objects; bit 2 `cold`: `#[cold]`, or never returns |
| `keeps_none` | bit `i`: the value of parameter `i` is not stored, captured or passed where it may be kept, and lives on past the call only in the result if `returns` says so |
| `returns`    | bit `i`: the result may be parameter `i`'s value or derived from it |

Downstream reads them only above `-O0`. A `readonly` call clobbers no load, so values loaded before it are reused after it and loads move across it; a `quiet` call is no safepoint for its caller; a `cold` call marks its block as rarely run; a stack address passed in a `keeps_none` parameter does not escape. Panics are not moved: a call stays where it is.

## 9.4.8. Rules for separate compilation

A prebuilt library cannot see the whole program at once, so packages are reconciled by the following rules.

- **`any` tag.** The tag of an `any` slot is a 31-bit hash of the type name, so separately built packages use the same tag for the same type.
- **`any` seed.** So that the program knows the types a library puts into `any`, the library appends a small Nabi function using those types to the end of the body. The program's closed-world `any` dispatcher includes these types too.
- **Dispatcher symbol.** The `any` dispatcher is the symbol `nabi.any.<hash>`, defined weak in libraries and strong in the program. At link time the program's definition wins.
- **Sum member order.** Terms of a sum are ordered by a structural key, not by internal type numbers ([2.6.4](../002-types/206-sum-types.md#canonical-order)), so tags agree across packages.
- **GC type id.** Each package has its own GC type id range (9.4.6). It is an error for the ranges of two packages in one program to overlap.
- **Method symbols.** Methods of dependency packages are mangled by the stable path `<module>::<type>.<method>[__by_<trait>]` ([9.5](905-symbol-mangling.md)).
- **Call summaries.** A downstream build's code depends on the call summaries of the functions it calls (9.4.7). An incremental build's codegen unit keys on the summaries of its callees in other units and packages, so a rebuilt dependency whose summaries changed recompiles just the units calling the changed functions.

## 9.4.9. Reading and rejection

`--extern name=<name>.nabihir` or `--extern name=lib<name>.a` each locate the sibling file ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)). Downstream parses the body as a source overlay to obtain declarations, declares stripped functions as prebuilt symbols, monomorphizes generics in place, and then links the archive.

If any of the following differs, the NABIHIR is rejected with a "rebuild" error.

- `version`
- producer (an NABIHIR made by a different `nabic`)
- name (the `--extern` name versus the NABIHIR's mount name)

## 9.4.10. NABIHIR inspection

`nabi inspect --dump <file>` prints the NABIHIR's version, producer, meta (including `features:`), the paths and text of the interface files, the summaries, and the call summaries (`--- calls`: one line per function, its name and set facts) ([9.9.2](909-debug-tools.md#992-nabihir-inspection)). `nabi inspect` ([9.9.14](909-debug-tools.md#9914-package-inspection)) shows a summary together with the archive.

## 9.4.11. Stability

Only NABIHIR made by the same `nabic` is read (9.4.9). When the format changes, `version` is bumped, and a different version requires a rebuild. The stability policy of the [Overview](../008-abi/README.md) applies.

The standard library is not distributed as NABIHIR ([9.3.6](903-package-archive.md#936-standard-library)).
