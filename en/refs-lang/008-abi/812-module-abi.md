# 8.12. Module descriptor ABI

A *module* is one loadable unit of Nabi code in a process: the program itself (the *host*) or a plugin. Each module describes itself to the runtime with one *module descriptor* in read-only data. The loader reads it to check that the module fits the process, to bind the symbols it imports, and to register its GC types, `any` tags, roots and type descriptors ([8.13](813-type-descriptor.md)). The layout is defined once in `nabir_abi::module`, which the compiler and the runtime both build.

## 8.12.1. Symbols

| Symbol | Defined by | What it is |
|--------|-----------|------------|
| `nabi_module` (Mach-O `_nabi_module`) | a plugin | Its descriptor, the plugin's only exported symbol |
| `nabi_host_module` | a host | Its descriptor, reached by the runtime directly |

## 8.12.2. Layout rules

All pointers are absolute and fixed by the operating system's relocations. A `Str` is `{ptr, len: usize}` over UTF-8 bytes, a `Slice<T>` is `{ptr, len: usize}` over `len` items. Every structure is laid out with no implicit padding at either pointer width: a `u64` sits at a multiple of 8 and pointer-sized words come in pairs. So 32-bit targets whose `u64` alignment is 4 (i686) and 8 (wasm32) agree on every offset. Fields are only ever appended; `version` says which fields a reader may use.

## 8.12.3. `ModuleDesc`

| Field | Type | Meaning |
|-------|------|---------|
| `magic` | `u32` | `"NABM"` as a little-endian `u32` |
| `version` | `u16` | 1 |
| `flags` | `u16` | `HOST` 1, `PLUGIN` 2, `HOT` 4 (a hot-reload build), `NO_UNLOAD` 8 (gave a C callback of its own code out) |
| `hot_version` | `u64` | 0 unless `HOT` |
| `text_lo`, `text_hi` | `usize` | The module's code is `[text_lo, text_hi)` |
| `toolchain` | `Str` | The compiler id; every module of a process has the same one |
| `target` | `Str` | The target triple |
| `name` | `Str` | The plugin's package name, or the host's program name |
| `provides` | `Slice<PkgDesc>` | The packages whose code and statics live in this module |
| `needs` | `Slice<PkgDesc>` | The packages it imports from the process |
| `imports` | `Slice<Import>` | The symbols it takes from the process |
| `exports` | `Slice<Export>` | The symbols it gives, sorted by `sym` |
| `gc_types` | `GcTypes` | Its ScanTables ([8.2](802-scan-table.md)) and the ids the loader gives them |
| `tags` | `Tags` | The type keys of the `any` tags its code uses and the tags the loader gives them |
| `roots` | `Slice<Root>` | Its static roots |
| `types` | `Slice<*const TypeDesc>` | Its type descriptors ([8.13](813-type-descriptor.md)) |
| `dispatch` | `Slice<Dispatch>` | Its `any Tr` method bodies, by method and tag |
| `functions` | `Slice<FnRange>` | `HOT` only: the extent and frame facts of each function |

The parts:

| Structure | Fields |
|-----------|--------|
| `PkgDesc` | `name: Str`, `version: Str`, `pkg8: [u8; 8]` (the hex digits of the package hash in its `_Y` symbols, [9.5](../009-tools/905-symbol-mangling.md)), `sha256: [u8; 32]` (the package's full content hash) |
| `Import` | `sym: Str` (the mangled name), `sig: u64`, `slot: *mut usize`, `kind: usize` (`FN` 0, `DATA` 1, `HOT_FN` 2) |
| `Export` | `sym: Str`, `path: Str` (`enemies::spawn` for a `pub` non-generic function or static, else empty), `sig: u64`, `addr: usize`, `kind: usize` (as `Import`, plus `FN_VALUE` 3: a function's value thunk), `sig_text: Str` (the type `sig` hashes as `type_name` writes it, `fn(i32) -> List<str>`, for error messages; empty when `sig` is 0) |
| `GcTypes` | `tables: *const *const u8`, `ids: *mut u32`, `n: u32`, `reserved: u32` |
| `TagKey` | `key: Str` (a type key, 8.13.2), `hash: u64` (FNV-1a 64 of `key`) |
| `Tags` | `keys: *const TagKey`, `tags: *mut u32`, `n: u32`, `reserved: u32` |
| `Root` | `word: *const usize`, `any: usize` (1: the payload word of an `any`, its tag the word before) |
| `Dispatch` | `method: u64`, `code: usize`, `tag_index: usize` (into `tags`) |
| `FnRange` | `start: u32` (from `text_lo`), `len: u32`, `frame: *const u8`, `reserved: usize` |

`sig` is the first 64 bits of SHA-256 over the canonical signature text written with type keys, such as `fn(Y3f8a1c2b:game::Player, i32) -> str`. For data it is the hash of the static's type key. A `FN` import is called through a stub that jumps through `slot`; a `DATA` import's `slot` holds the data's address.

## 8.12.4. Runtime entries

These entries extend [8.5](805-extern-symbols.md). All are `thread-safe`.

| Entry | Meaning |
|-------|---------|
| `nabi_types_register(descs: *const *const TypeDesc, n: usize, tags: *const TagKey, out_tags: *mut u32, nt: usize) -> i32` | Registers `n` type descriptors and resolves `nt` tags (8.12.5). Returns 0, or `1 + i` when `descs[i]`'s key is already registered with another layout hash; nothing is registered then. |
| `nabi_types_by_tag(tag: u32) -> *const TypeDesc` | The descriptor of an `any` tag (its `ANY_BOXED` bit ignored), or null |
| `nabi_types_by_path(p: *const u8, len: usize) -> *const TypeDesc` | The newest registered descriptor of a path such as `game::Player`, or null |
| `nabi_types_unregister(descs: *const *const TypeDesc, n: usize)` | Drops a registration when its module is freed |
| `nabi_module_register(m: *const ModuleDesc) -> i32` | Checks and binds a module and registers its tables, inside a short stop-the-world; 0 on success |
| `nabi_any_dispatch(method: u64, tag: u32) -> usize` | The code of `method` for a value of `tag` from a loaded module, or 0 |
| `nabi_tls_get() -> *mut u8` | The calling thread's `nabi_tls` block ([8.3](803-tls-slots.md)) |

Readers of the type registry take no lock and allocate nothing.

## 8.12.5. Tags

A module's code reads the `any` tag of each type it uses from `Tags.tags`. For each key, `nabi_types_register` keeps a tag already in `out_tags` (not `u32::MAX`): a host and a program without plugins pass their compile-time tags this way. A key registered before gets the same tag, which is what makes one type one tag across modules. A new key gets the low 31 bits of the key's FNV-1a hash, moved past every tag in use.

## 8.12.6. What the compiler emits

A plugin's glue ([8.7](807-code-generation.md)) emits `nabi_module` and the tables it points at; a host's emits `nabi_host_module`. A program that reaches neither `std::plugin` nor `--crate-type=plugin` emits neither, and its code is the same as without plugins.

| Field | Plugin | Host |
|---|---|---|
| `flags` | `PLUGIN` | `HOST` |
| `text_lo`, `text_hi` | relocations to `nabi.module.text_lo` / `nabi.module.text_hi`, which the linker defines; the loader takes the code's extent from the mapped image | same |
| `provides` | its own package and those it bundles ([9.12.2](../009-tools/912-plugins.md#9122-building)) | every package it links but the standard library |
| `needs` | every other package of the build but the standard library | empty |
| `imports` | a relocation to `nabi.module.imports`, the table the linker builds from the symbols left undefined (every `nabi_*` entry and `_Y` symbol); the linker writes its length into the descriptor; `sig` 0 | empty |
| `exports` | sorted by `sym`: `FN` for each function of its package (`path` set for `pub` non-generic ones), `FN_VALUE` for their value thunks (`path`, `sig` and `sig_text` set), `DATA` for its statics (`path` set), and its own copies of statics it takes from the process (below) | a relocation to `nabi.module.exports`, the table the linker builds (every defined `nabi_*` entry and `_Y` symbol, `sig` 0); the linker writes its length |
| `gc_types` | every ScanTable its code uses; `ids` initially 0 | empty: a host registers its tables with `nabi_gc_register_type` as any program |
| `tags` | the key of each tag its code uses; `tags` initially `u32::MAX` | the keys with the compile-time tags |
| `roots` | every reference word of every static it defines (`any` 0) | empty: registered as any program's |
| `types` | its type descriptors ([8.13](813-type-descriptor.md)), whose `tag_index` indexes its `tags` | empty: it registers its descriptors with `nabi_types_register` as any program |
| `dispatch` | per `any Tr` dispatcher and per tag its arms take: the method's FNV-1a 64 hash of `<trait path>.<method>`, the dispatcher's value code | the same for the host's dispatchers |

A host's `nabi.register_types` ([8.5](805-extern-symbols.md)) ends with `nabi_module_register(&nabi_host_module)`, so its tags, dispatch entries and export table are in the process before `main` and before any plugin.

How a plugin's code differs from a program's:

- **GC type ids.** An allocation reads `nabi.module.gc_ids[0]` and adds the table's index: the loader gives a module a contiguous range, so `ids[i] = ids[0] + i`.
- **`any` tags.** Storing into `any` and testing an `any`'s tag load the tag from `nabi.module.tags` (`tags[i]` for the type's index in the module's tag list) instead of using a constant; `ANY_BOXED` is or-ed in at run time.
- **TLS.** Code that would take the address of `nabi_tls` ([8.3](803-tls-slots.md)), a function value's thunk or a C entry, calls the import `nabi_tls_get()`. Nabi-to-Nabi calls pass the block as the hidden last argument as everywhere, so most code never asks.
- **Statics it takes from the process.** A static of a package the plugin neither owns nor bundles (the standard library's included) is reached through the GOT form of the target, which the linker points at the import's slot. The plugin also defines its own copy as `<sym>.own` (and `<sym>.state.own`, `<sym>.init.own` for a lazily initialized one) and exports it under `<sym>`. The loader binds the import to the host's static, else to a plugin's loaded before, else to this copy, so every static has exactly one instance in the process.
- **Dispatchers.** A dispatcher's arm for a tag it does not know calls `nabi_any_dispatch(method, tag)` and calls the result as a function value of the dispatcher's own type. In a program that is no host the arm stays the trap of [8.8](808-lowering-types.md).
- **Signatures.** An export's `sig`, and the value `Plugin::get::<F>` passes, is the first 8 bytes (little-endian) of the SHA-256 of the signature's text written with type keys (`fn(Y3f8a1c2b:engine::World,i32)->str`). Both sides compute it at compile time from the same text, so an equal hash means the same type.
