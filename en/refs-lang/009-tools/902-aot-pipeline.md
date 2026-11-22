# 9.2. AOT build pipeline

This chapter defines the step-by-step flow of `nabic build`. It makes the overall picture given in [Execution engines and user surfaces](../000-meta/001-execution-modes.md) and [Compile pipeline](../000-meta/002-pipeline.md) concrete in terms of commands and file flow.

## 9.2.1. Steps

The build proceeds in 11 steps, all inside `nabic`.

```
1. package discovery       (find package root)
2. dependency resolve      (resolve deps to sources or prebuilt .nabihir + .a)
3. source parse            (.nabi → AST)
4. name / visibility       (resolve)
5. type check              (type check)
6. nabi-hir build           (HIR build)
7. specialize              (collect call-sites + monomorphize)
8. nabi-mir build
9. code generation         (optimized at -O1 and above)
10. archive / link         (nabic's linker)
11. artifact .a / executable
```

> In order: package discovery, dependency resolution, source parsing, name / visibility resolution, type checking, nabi-hir construction, specialization (call-site collection and monomorphization), nabi-mir construction, code generation to objects, archiving or linking, and finally the output: an `.a` archive or an executable.

The input, output and responsibility of each step are kept clearly separate.

## 9.2.2. Package discovery

Starting from the path of `nabic build [path]` and searching upward, a directory containing `src/main.nabi` or `src/lib.nabi` is located (starting from the containing directory if path is a file); the search moves up one directory at a time.

If found, that directory is the package root and its `src/` the package's source (see [Source units](../000-meta/003-source-units.md)). The other cases:

- If path is a file outside the found package's `src/` (a script or tool beside the package's source), or no package is found walking upward, it is an anonymous executable package consisting of that single file ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)). Its file submodules are beside it.
- If path is a directory and there is no package, it is an error.

The package name is the name of the package directory (for an anonymous package, the file name without `.nabi`) and becomes the output name (9.2.15). A package with both root files is built as two targets: its library first, then its executable, which mounts the library as dependency `<name>` ([0.3.1](../000-meta/003-source-units.md#library-and-executable-in-one-package)).

## 9.2.3. Dependency resolution

This step handles external package references (`use external_pkg::...`) in [paths](../005-modules/507-paths.md) and [`use`](../005-modules/505-use.md).

`nabic` does not resolve dependencies itself. The front-end `nabi` ([9.10](910-nabi-frontend.md)) prepares path / git dependency packages according to `nabi.nabion` / `nabi.lock` and passes the root of each package to `nabic` via `--extern`.

```
nabic build --extern util=../util --extern parser=target/debug/deps/libparser.a
```

`--extern name=<path>` mounts one dependency package at the top-level path `name`. `name` is the identifier obtained from the `deps` key in `nabi.nabion` by replacing `-` with `_`. The path takes two forms.

| Path                              | Dependency package                                                           |
|-----------------------------------|------------------------------------------------------------------------------|
| `<dir>` / `<dir>/src/lib.nabi`     | Source. That `src/lib.nabi` is compiled together in the same build            |
| `<name>.nabihir` / `lib<name>.a`     | Prebuilt package. Given either one, the sibling file in the same directory is located. NABIHIR ([9.4](904-nabihir-bytecode.md)) is read for declarations and generics, and the archive is linked |

- User code uses that package's items via `use name::..`.
- `pkg::` inside a dependency package refers to that dependency package itself.
- A dependency package is a separate package. Only `pub` items are usable from outside; `pub(pkg)` / `pub(sub)` / private items are not ([5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)).
- It is an error for a module at the root of the user package to have the same name as a package of the build.
- A later `--extern` whose path is that of an earlier one is another name for the same package (packages may call a dependency differently, [9.10.2](910-nabi-frontend.md)). The package is built and mounted once, under the first name; a path that starts with another name (`other::..`, in the code and meta block templates of a package that calls it so) names it, so its items and types are the same under every name.
- `--extern-dep <package>=<name>` says that package `<package>` (an `--extern` name, or `pkg` for the package being built) depends on the extern `<name>`; `nabi` passes one for each dependency of each package. With any, each package sees only the externs it depends on, by the names it gives them ([5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)); an extern no `--extern-dep` names (the package's own library, which `nabic` mounts) is the built package's, and depends on what it depends on. Without any, every package sees every extern.

`nabic build` on a package with `src/lib.nabi` produces `lib<name>.a` and `<name>.nabihir`, and with `src/main.nabi` the executable `<name>`; a package with both produces all three, the executable linked against that library. `--name <n>` sets the package name (the output name and mount name); the default is the directory name.

### Prebuilt dependency packages

`nabic` prebuilds each source `--extern` **once**, in the order given, into `target/[<triple>/]<profile>/deps/`. Each dependency package is built after the ones before it, and sees those it depends on (`--extern-dep`; without it, all of them). Subsequent compilation uses these `lib<name>.a` and `<name>.nabihir` instead of the source.

```
target/<profile>/deps/
├── lib<name>.a
├── <name>.nabihir
└── <name>.fp               # fingerprint
```

If the fingerprint matches, it is not rebuilt. The fingerprint includes:

- The identity of `nabic` (the same value as the NABIHIR producer)
- Build options: `--release` / `-O*`, `-g`, `--target`, `--cfg` (without its `feature` keys), `--bootstrap`
- The dependency's features (`--extern-feature`), and the `--extern-dep`s of it and of the packages built before it
- All files under that dependency package's `src/`, and its `nabi.nabion`
- The fingerprints of dependency packages built before it

A prebuilt package's interface is read with the features it was built with (its NABIHIR meta, [9.4.6](904-nabihir-bytecode.md#946-meta)), not the consumer's.

Every command that compiles uses this path: `build` / `run` / `test` / `bench` / `check` / `lint` / `doc` / `lsp` / `repl` / `jupyter`, single-file scripts, and named environments. The REPL host also links the prebuilt archives. If there is neither a package nor a target directory (`--target-dir`, `NABI_TARGET_DIR`), they are placed under the lone-file cache in the system temporary directory (9.2.16).

The standard library is compiled from source with every build, only the part the program reaches; its compiled functions are kept in the toolchain's function cache ([9.3.6](903-package-archive.md#936-standard-library)).

## 9.2.4. Source parsing

All `.nabi` files are parsed into an AST according to the lexical / syntactic rules from [Source files](../001-lexical/101-source-files.md) through [Punctuation and operator tokens](../001-lexical/110-punctuation-and-operator-tokens.md).

A directory module of [Modules](../005-modules/504-modules.md) combines all its .nabi files into the body of one module.

The manifest `nabi.nabion` at the package root ([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)) is not a module file. The manifest is read as a single [NABION](../../refs-standard-libraries/300-std/308-nabion.md) value (`Package { .. }`); top-level statement rules do not apply to it and it is not compiled.

### Scanner pre-pass

The scanner pre-pass of the [`#[keyword]` magic](../006-meta-system/606-keyword-magic.md) is also performed in this step. All .nabi files are first scanned shallowly, `#[keyword(...)]`, `#[infix(...)]`, `#[literal(...)]` registrations are collected, and the registrations are passed to the real lexer / parser.

## 9.2.5. Name / visibility resolution

Every path and identifier is mapped to an item according to the rules of [`use` (imports)](../005-modules/505-use.md), [Visibility](../005-modules/506-visibility.md), [Paths](../005-modules/507-paths.md) and [Re-export (`pub use`)](../005-modules/508-re-export.md).

This step applies the implicit `core::prelude` import (see [Standard library tiers](../000-meta/005-standard-libraries-tiers.md); `bare::*` and `std::*` are excluded as they require explicit import), performs visibility checks reporting improper access as errors, and checks for import conflicts.

## 9.2.6. Type checking

The type of every expression is determined by the rules of [2. Type system](../002-types/README.md).

This includes deciding between gradual and static slots (see [`any` slot](../002-types/202-any-slot.md)), checking trait bounds (see [Generics](../002-types/210-generics.md)), and fixing the result types of branches (see [3. Expressions](../003-expressions/README.md)). Narrowing of sum types and `any` slots is handled here as well.

## 9.2.7. nabi-hir construction

The type-checked AST is converted into the nabi-hir ([HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir)) data structure.

HIR is an intermediate representation rich in type information, reflecting all results of steps 4-5 above. It is structurally compressed so that identical types and identifiers share a representation.

## 9.2.8. specialize: monomorphization

Monomorphization of [generics](../002-types/210-generics.md) and [const generics](../002-types/211-const-generics.md) is performed.

```
1. Collect all call sites.
2. Determine the concrete type tuple of each call.
3. Create a separate specialized HIR per (item_id, type_tuple).
4. Cache. Each (item_id, type_tuple) only once.
```

Functions unreachable from the program's roots (`main`, exported symbols, `#[export_name]` items, symbols referenced by the runtime) are not emitted. The same holds for library builds: functions not reachable from the package's `pub` items and those roots are not put into the archive.

This step is what lets [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) make direct calls without vtables (see [trait-as-type ban](../002-types/212-trait-as-type-banned.md)).

## 9.2.9. nabi-mir construction

The [specialized](../000-meta/007-glossary.md#specialize) HIR is converted into nabi-mir (MIR).

MIR is an IR one level lower than HIR. In this step operator desugaring is completed (see [Operator dispatch](../003-expressions/301-operator-dispatch.md)), the decision tree of pattern matching is determined (see [`match` expression](../003-expressions/312-match-expression.md)), and the frame layout of closures and coroutines is fixed.

## 9.2.10. Optimization levels

The optimization level decides how nabi-mir is optimized before code generation ([8.7.3](../008-abi/807-code-generation.md#873-optimization-levels)); the code generator is the same at every level ([0.2.4](../000-meta/002-pipeline.md#024-code-generation)).

| Level         | Effect |
|---------------|--------|
| `-O0`         | No optimization; every value in its stack slot |
| `-O1`         | Values simplified and in registers; loops optimized as at `-O2` (calls in them inlined as at `-O2`, so a `for` loop's iterator becomes a counted loop, strip mined and vectorized), small and `#[inline]` functions inlined elsewhere, and functions called from one place everywhere; with debug information (the `debug` profile) |
| `-O2`         | Also larger bodies inlined outside loops |
| `-O3`         | Also larger bodies inlined in loops, and loops left scalar unrolled: a whole strip of a loop that keeps a float sum or calls a function tests its counter once every 2 or 4 iterations |
| `-Os`         | Inlining only where the body is about as small as the call (a few times that in a loop), loops still made counted and strip mined, and vectorized where the body is short |

`-g` changes neither the optimizations nor the code (8.7.3): the debug information follows the optimized code, with inlined bodies as frames and each variable in a register, in its stack slot or optimized out ([9.9.6](909-debug-tools.md#996-debugger-integration-gdb--lldb)). The optimizer sees the whole package at once, before it is split into codegen units (9.2.16), so it inlines across modules and into and out of the standard library's functions, which every build compiles with the program ([9.3.6](903-package-archive.md#936-standard-library)). Objects built at different levels follow the same runtime ABI ([8](../008-abi/README.md)) and link together, including prebuilt dependencies built at another level. The REPL compiles cells at the level `:opt` sets ([9.9.9](909-debug-tools.md#999-repl--jupyter-debugging)), without optimizing their nabi-mir.

## 9.2.11. Code generator

The code generator ([8.7](../008-abi/807-code-generation.md)) generates machine code and the object file straight from nabi-mir, through one low-level form shared by every target, for `x86_64`, `aarch64`, i686, armv7, riscv64, riscv32, powerpc64le, s390x and wasm32. With `-g` it emits DWARF: functions, line tables, parameters, locals and their types, and in optimized code the bodies inlined and where each variable is by code range ([9.9.6](909-debug-tools.md#996-debugger-integration-gdb--lldb)); for Windows targets it emits the same as CodeView in the objects (`.debug$S`, `.debug$T`), which the linker gathers into a PDB (9.2.14). Code that uses something it does not support yet is an `E0129` error naming it.

## 9.2.12. Objects

Every target's objects are written by `nabic`: Mach-O for Apple targets, ELF for Linux, FreeBSD and riscv32, COFF for Windows, and wasm objects for wasm32. A build's codegen units (9.2.16) are compiled in parallel, one object each. `--emit=obj` writes the object instead of linking it ([9.9.1](909-debug-tools.md#991-intermediate-output---emit)).

```
target/debug/build/foo.o
```

## 9.2.13. Archives

A library package puts its objects, one per codegen unit (9.2.16), into a standard `ar` archive written by `nabic` itself, and writes the interface file `foo.nabihir` ([NABIHIR format](904-nabihir-bytecode.md)) next to it.

```
target/debug/libfoo.a
target/debug/foo.nabihir
```

## 9.2.14. Linking (for executables)

An executable is linked by `nabic`'s own linker:

```
foo.o + <deps>/lib*.a + <sysroot>/lib/libnabir.a + target libc  →  target/debug/foo
```

A Mach-O executable links against the macOS SDK (Xcode Command Line Tools). A Windows executable links against import libraries `nabic` builds from module definitions it carries (derived from mingw-w64), with Nabi's own startup over the UCRT; no MSVC tools or Windows SDK are needed. Nabi's Windows startup runs the C initializers in `.CRT$XCA`..`.CRT$XCZ` before `main`. For x86 Windows the import names come from mingw-w64's `lib32` definitions, with stdcall `@N` decoration, and images link with `/SAFESEH:NO`. Linux targets, native or cross, always link the shipped C library in `<sysroot>/lib/nabi/targets/<triple>/`: musl statically, glibc through link stubs (executables need glibc 2.34 or later). Other libraries are found through `-L` and `LIBRARY_PATH`. The dynamic loaders named for 32-bit glibc targets are `/lib/ld-linux.so.2` (i686) and `/lib/ld-linux-armhf.so.3` (armv7hf). A riscv32 image (no OS, in progress) is static and links no C library: the runtime brings its entry point and heap ([0.6.2](../000-meta/006-platform-targets.md#riscv32imac-unknown-none-elf)). If `NABI_CC` is set, that C compiler driver performs the final link instead (`$NABI_CC <objects> <archives> -o <out>`); this is an optional override. Shared libraries given with `--native-runtime` are copied beside the executable, which gets rpath `@loader_path` (Mach-O) / `$ORIGIN` (ELF); Windows loads DLLs from the executable's directory. Executables are dead-stripped: unreferenced functions and data are removed; a `release` build also folds identical functions and merges identical string literals ([9.2.15](#9215-output)). A Mach-O executable carries an ad-hoc code signature, which arm64 macOS requires to run it. When the executable has debug information on Apple targets, `nabic` gathers the objects' DWARF into a `.dSYM` bundle next to the executable; on Windows targets the linker writes a PDB (MSF 7.0): the objects' CodeView types merged, a module per object with its symbols and line tables, global and public symbols, and section contributions; the executable's debug directory names it (an RSDS record with the PDB's GUID and age). A `wasm32-wasip2` executable is linked as a core wasm module, then made into a WASI preview 2 component, with the preview 1 adapter the toolchain ships for the target; an `ffi::wit` world ([std 1.8](../../refs-standard-libraries/100-bare/108-wit.md)) shapes its imports and exports.

## 9.2.15. Output

Output is placed in **per-profile** directories, as in cargo. The profiles are `debug` (default) and `release`; the profile is `release` when the optimization level is `-O2` / `-O3` / `-Os` (including `--release`).

The profiles differ as in cargo:

| Profile | Optimization | Debug information | Link |
|---|---|---|---|
| `debug` | `-O0`, `-O1` | on | unreferenced code and data removed |
| `release` | `-O2`, `-O3`, `-Os` | only with `-g` | also identical functions folded and identical string literals merged; without `-g`, no symbols beyond what the loader needs |

Identical code folding compares functions by their machine code and relocations: two fold into one when their bytes are the same and every reference points to the same place, or to functions that fold together themselves. A function whose address is taken (stored, compared, exported, placed in a table) is never folded, so distinct functions keep distinct addresses; only functions that are called and jumped to are, and the thunks function values call (`<symbol>.v`, [8.11](../008-abi/811-backend-contract.md)): function values cannot be compared, so two of them sharing their code changes nothing a program can see. A folded copy's unwind entry and debug information are dropped, as for removed code: a debugger, profiler or backtrace names the copy that stays. String literals with the same bytes share one copy (their addresses mean nothing). `-C icf=no` turns this off in a `release` build, and `-C icf` turns it on in a `debug` build ([9.1](901-nabic-cli.md)).

Debug information belongs to the executable or archive a build makes (`build`, `run`, `test`, and `bench` at an explicit `-O0` / `-O1`), not to `--emit` output or REPL and Jupyter cells. A panic names its source location in either profile. A build's objects are linked from one place beside the output, so the same inputs make the same executable. Overflow checks are on in both profiles; `profile` in `nabi.nabion` can turn them off for one ([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)). Profiles have no other settings.

```
target/
├── debug/                      # profile (default)
│   ├── foo                     # executable (src/main.nabi)
│   ├── libfoo.a                # library archive (src/lib.nabi)
│   ├── foo.nabihir              # library interface (9.4)
│   ├── build/                  # intermediates and --emit output (foo.mir, ...)
│   └── deps/                   # prebuilt dependencies (lib<dep>.a, <dep>.nabihir, <dep>.fp)
├── release/                    # --release
│   └── ...
├── aarch64-unknown-linux-gnu/  # cross build (--target=<triple>)
│   └── debug/ ...
└── doc/                        # nabi doc
```

| Output          | Path                                                              |
|-----------------|-------------------------------------------------------------------|
| Executable      | `target/<profile>/<name>`                                         |
| Library         | `target/<profile>/lib<name>.a`                                    |
| `--emit=<kind>` | `target/<profile>/build/<name>.<ext>` (`ast.nabi` `hir` `mir` `o` `s` `lst` `expanded.nabi`) |
| Intermediates   | `target/<profile>/build/`                                         |
| Dependencies    | `target/<profile>/deps/` (`lib<dep>.a`, `<dep>.nabihir`, `<dep>.fp`, 9.2.3) |
| Cross build     | `target/<triple>/<profile>/...` (same structure as above)         |
| Documentation   | `target/doc/`                                                     |
| Incremental     | `target/<profile>/incremental/<name>/` (an executable's codegen units), `.../lib<name>/` (a library's), and `.../<name>-<hash>/` per target root (the front end), 9.2.16 |

A package with both root files has both outputs, `target/<profile>/<name>` and `target/<profile>/lib<name>.a` (+ `<name>.nabihir`); `-o` names the executable.

The name `foo` is the package name (9.2.2). `-o` changes the path of a single output; `--target-dir=<dir>` and the environment variable `NABI_TARGET_DIR` change the location of `target/` itself. `nabi clean` deletes the whole `target/`. A library's NABIHIR is `foo.nabihir` next to the archive ([9.4](904-nabihir-bytecode.md)).

## 9.2.16. Incremental build

Incremental builds work at three levels: a package whose inputs did not change is not built at all; a package that is built takes the parses and lowered bodies of its unchanged files and functions from the last build (front end, below); and its code is compiled by codegen units, of which only the changed ones are compiled again. If the fingerprint of the previous build equals the current one, `nabic build` / `nabi run` print `Fresh` and skip the build. The fingerprint includes:

- The compiler's identity (the same value as the NABIHIR producer), build options, diagnostic format (`--message-format`, color), input / output paths, `NABI_CC` if set
- The files under `src/` of the package and its dependencies, their `nabi.nabion`, and every other file the build read (`#[path]` / `include_str!` outside `src/`)
- Files read by `include_str!` / `include_bytes!` ([6.11](../006-meta-system/611-include.md)), with their contents
- The lint levels (`-A` / `-W` / `-D`, and the manifest's `lints` through them)
- The runtime library, the C runtime objects and C library the toolchain ships for the target ([9.2.19](#9219-installation-layout)) and the `.a` of prebuilt dependencies, each by its size, modification time and file identity (inode and status change time; on Windows its creation time), so a copy that keeps the modification time is a change
- The size, modification time and file identity of the outputs

Warnings emitted by a build, lint warnings and the back end's included, are recorded, and a `Fresh` build prints them again.

Every build, native or cross, compiles the standard library functions the program reaches ([9.3.6](903-package-archive.md#936-standard-library)); their optimized code and their units' objects are kept in the toolchain's function cache, `lib/nabi-cache/fn/` (9.2.19), for every build of the toolchain, a lone file's too:

- An entry's key covers what its result depends on: for optimized code, the function's code before the step, the level, the target and what the step reads of the functions it calls, all by name; for an object, the unit's key (below); both name the compiler's identity. A package keeps the standard library's lowered bodies with its own (Front end, below). A build takes an entry whose key matches and writes the ones it made; the output is the one a build with an empty cache makes.
- Entries are files written whole and then renamed into place, so builds running at the same time share the cache safely.
- At most once a day a build removes the entries no build read for 30 days; `nabi cache prune` removes more (9.10.6). A toolchain whose cache directory is read-only and not empty (a sandbox image warmed when it was built) reads its entries and writes none.
- The cache has a cap, `NABI_CACHE_MAX_SIZE` (bytes, or a number with `K`, `M` or `G`; default `5G`; `off`: none), over its function cache and its library stages (below). A build that wrote entries adds their size to a running total in the cache directory; when the total passes the cap, one build at a time measures the cache and, if it is over the cap, removes the entries used longest ago until at most 80% of the cap is left. An entry's use is its last write or read, recorded at most once an hour. Entries used in the last hour, and files still being written, are never removed this way, so builds running at the same time keep what they use; when they keep the cache over 80% of the cap, the next measure waits until another fifth of the cap is written. A removed entry is computed again by the next build that needs it.
- `NABI_FN_CACHE=0` neither reads nor writes the cache.

### Front end

A build of a package (`check`, `build`, `run`, `test`, `bench`, `doc`, `lint`) keeps what its front end computed under `target/<profile>/incremental/<name>-<hash>/`, where `<hash>` names the package root (each target of a package with both root files has its own, `<hash>` naming its root), with one subdirectory per kind of build (target, cfg, test or not, which libraries are prebuilt). The next build of the same kind takes from it:

- The parse of each file whose text and parse inputs (the build's extension operators, the package's keywords, meta blocks and `#[literal]` patterns, and cfg) are unchanged.
- The lowering (name resolution, type checking, generic instantiation) of each plain function body: a non-generic, non-`const` function, or a method of a non-generic `impl`. It is reused when its own text and position are unchanged, the program's interface is unchanged, every signature, instance and other fact the lowering read is the same now, and the text before every position its result names outside the body is unchanged.
- The interface is every item except the plain bodies whose return type is written (or `()`) and the plain functions' parameters and return types; a body whose return type is inferred is part of it (its callers read the type). A body edit lowers that body again, and an edit of a body whose return type is inferred also lowers the bodies that read its type; a plain function's signature edit lowers it and the bodies that read its signature; any other change to an item (a type, a trait, an `impl` header, a generic function, a `use`) lowers every body again.
- Functions, types and instances a reused lowering made are numbered as in a clean build; diagnostics, the program and its symbols are those of a clean build.

Signatures, `impl` registration, coherence and the whole-program checks run in every build. What collecting the library's items made (the standard library's and the dependencies'), and the library files' pre-pass parses, are kept in the toolchain's cache (`lib/nabi-cache/library/`, one file per hash of toolchain, build flags and library sources) for every build, a lone file's too; `NABI_INCREMENTAL` neither moves nor disables them. Nothing kept is required: a missing, damaged or stale entry is computed again. A lone file outside any package keeps no parses or bodies of its own unless `NABI_INCREMENTAL` names a directory. `NABI_INCREMENTAL=<dir>` keeps the results under `<dir>/<name>-<hash>/`; `NABI_INCREMENTAL=0` turns them off. `NABI_INCREMENTAL_VERIFY=1` also lowers every body again and fails the build if a reused lowering differs. `-v` reports how many files' parses and bodies were reused.

### Codegen units

A build that is not `Fresh` reuses compiled code by codegen units. nabic divides a package's code (an executable's or a library's) into units: each module of the package is one, and so is each module of the standard library and of a dependency whose functions or instances it compiles (instances may be spread over several units by their type arguments); and one unit holds what the program has once (statics, the C `main`, GC type registration).

- A unit is compiled only when its key changed since the last build into the same target directory. Otherwise its result, kept under `target/<profile>/incremental/<name>/` (a library's: `lib<name>/`), is reused; then everything is linked. A standard library unit is kept in the function cache instead (above), for every package's builds; a lone file uses it too, though it keeps no units of its own. A prebuilt dependency (9.2.3) is compiled as one object, without units.
- An anonymous file's units are kept in the target directory used without a package (`--target-dir`, `NABI_TARGET_DIR`, else the lone-file cache), under `[<triple>/]<profile>/incremental/<stem>-<hash of its path>/`. The lone-file cache is `<system temporary directory>/nabic-target-<uid>/` on Unix, `<uid>` the effective user ID, and `%TEMP%\nabic-target\` on Windows, whose temporary directory is the user's own. It is a cache: at most once a day, nabic removes its files that no build wrote for a week.
- A lone file's executable is kept in the same target directory, under `[<triple>/]<profile>/scripts/<key>/`, and reused while its key matches; an entry unused for a day is removed when another is built ([9.10.7](910-nabi-frontend.md#9107-inline-manifest-for-single-file-scripts)). `nabi run` runs it and `nabic build` copies it next to the file. For a script, `nabi` passes `--target-dir` with the script's own `target/` under `$NABI_HOME/cache/scripts/<hash>/`; the lone-file cache holds the rest: lone files built by `nabic` itself, and a file in a directory whose `nabi.nabion` makes it a single-file package (9.10.1).
- The lone-file cache holds executables that are run, so on Unix it is private: the compiler creates it with mode `0700`, and before using it checks that it is a directory (not a symbolic link), owned by the effective user, with no permission bits for group or others. Otherwise the command fails with exit code `1` and ``nabic: error: <dir> is not private to you (owner uid N, mode 0755); remove it or set NABI_TARGET_DIR``. `--target-dir` and `NABI_TARGET_DIR` are used as given.
- A unit's key covers its functions' code, the signatures and code-generation facts of what it uses from other units (a function's signature change recompiles its callers' units), the statics and GC types it uses, the target, the optimization level, the code generation options, the compiler's identity and, under `-g`, the paths of the source files its debug info names (adding a file recompiles only the units that refer to it).
- Symbols and GC type ids do not depend on what other units contain (a unit reads GC type ids from symbols the package's shared unit defines; [9.5.7](905-symbol-mangling.md#957-collision-avoidance-typehash-stability)), so the output is the one a clean build makes.
- Optimized builds still inline across units: the package is optimized whole before it is divided (9.2.10).
- `-v` reports how many units were reused and compiled; `-vv` names the compiled ones.

## 9.2.17. Separation of compiler and runtime

`nabic` uses no third-party crates and links no external code generator.

The pipeline keeps the compiler and runtime apart: `nabic` does not reference the runtime (`libnabir.a`) at build time.

Instead it only emits calls to `extern "C"` symbols ([Extern "C" symbols](../008-abi/805-extern-symbols.md)), and at link time the user build links `libnabir.a` alongside.

This separation lets `nabic` and `libnabir.a` evolve under independent SemVer.

## 9.2.18. Self-contained toolchain

`nabic` is self-contained: its code generator and linker are its own ([0.2.4](../000-meta/002-pipeline.md#024-code-generation)), and it runs no external compiler, assembler or linker. Cross-compilation is built in: the toolchain ships the C library (musl, glibc link stubs) for its supported Linux targets.

| Needed from the system | Target                                   |
|------------------------|------------------------------------------|
| macOS SDK (Xcode Command Line Tools; not redistributable) | Apple targets |
| nothing (shipped C library) | Linux targets                   |
| nothing (import libraries built from shipped module definitions; UCRT) | Windows targets |
| `NABI_CC` (optional)    | Overrides the final link (9.2.14)        |

The C compiler `nabicc` also needs the target's C headers: the toolchain ships them for its Linux and wasm32 targets, Apple targets take them from the macOS SDK, and Windows targets from the Windows SDK and Visual Studio on a Windows host ([9.13.7](913-c-compiler.md#9137-headers-and-c-library)).

## 9.2.19. Installation layout

The installation layout follows rustup / cargo. The default of `NABI_HOME` is `~/.nabi` (`%USERPROFILE%\.nabi` on Windows).

```
$NABI_HOME/
├── bin/                         # proxies (the newest toolchain's nabi): run the active toolchain's same-named executable
│   ├── nabi
│   ├── nabic
│   ├── nabicc
│   ├── nabils
│   └── <name>                   # an executable installed by nabi install (9.10.5a)
├── toolchains/
│   └── <version>-<triple>/      # self-contained toolchain
│       ├── bin/{nabi, nabic, nabicc, nabils}
│       └── lib/
│           ├── libnabir.a        # runtime
│           ├── nabi/lldb/nabi.py           # lldb formatters for -g programs (9.9.6)
│           ├── nabi/targets/<triple>/   # what --target links: the runtime (libnabir.a / nabir.lib), C library,
│           │                           #   its headers (sysroot/, for nabicc),
│           │                           #   and C runtime objects, wasm32-wasip2's preview 1 adapter; the host's included
│           ├── nabi/packages/          # the official packages it carries, read only: index, src/<sha256>/ (9.10.4a)
│           └── nabi/{core, std, bare}/src/   # standard library sources: each tier is laid out as a package, root src/lib.nabi (5.4)
├── cache/                       # git, registry, native, scripts (9.10.4)
├── envs/<name>/, envs/.current  # named environments: nabi.nabion, nabi.lock, target/ (9.10.5)
├── installs.nabion               # what nabi install put in bin/, and where from
├── tokens.nabion                 # registry API tokens, written by nabi login (9.10.4a)
├── sources.nabion                # index mirrors, only read (9.10.4a)
└── config.nabion                 # Config { toolchain }
```

`nabic` finds the standard library and runtime relative to its own executable (sysroot = the executable's `..`) and does not look at `NABI_HOME`. Each tier of the standard library is laid out like a library package: its root module is `lib/nabi/<tier>/src/lib.nabi` and its modules are under `lib/nabi/<tier>/src/`. A `nabic` built in the implementation repository reads the same layout from the repository's `library/<tier>/src/`. Its own cache is the toolchain's `lib/nabi-cache/` (a repository build uses `target/<profile>/nabi-cache/`); a toolchain installed read-only (a system package) uses `nabi/nabic-<hash of the sysroot>/` in the user's cache directory instead. It holds in `fn/` the function cache, the standard library functions builds compiled (9.2.16), and in `library/` the library's collected items and pre-pass parses (9.2.16). `nabi cache` shows, prunes and empties it (9.10.6). Selection of the active toolchain is defined by [9.10.6](910-nabi-frontend.md#9106-toolchain).

A Linux (glibc) toolchain needs glibc 2.34 or later, as the executables it builds do (9.2.14).
