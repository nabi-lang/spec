# 9.3. Package archive (.a) format

The build output of a library package `foo` is two files ([9.2.15](902-aot-pipeline.md#9215-output)).

| File        | Contents                                                                      |
|-------------|-------------------------------------------------------------------------------|
| `libfoo.a`  | A standard `ar` archive holding the package's objects, one per codegen unit ([9.2.16](902-aot-pipeline.md#9216-incremental-build)) |
| `foo.nabihir`  | The interface used downstream for monomorphization and name resolution ([9.4](904-nabihir-bytecode.md)) |

The two files sit in the same directory. `--extern foo=libfoo.a` and `--extern foo=foo.nabihir` each locate the sibling ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)). Metadata such as version and producer is in the NABIHIR header ([9.4](904-nabihir-bytecode.md)).

## 9.3.1. Standard ar archive

`libfoo.a` is a standard Unix `ar` archive. `ar`, `ranlib`, `nm` and C linkers handle it without conversion.

```
ar t libfoo.a
part0.o
part1.o
...
```

## 9.3.2. The objects

Each object is a standard ELF / Mach-O / COFF object produced by the [AOT build pipeline](902-aot-pipeline.md) for one codegen unit. Together they contain only those of the package's [concrete fns](../000-meta/007-glossary.md#concrete-fn--generic-fn) reachable from the package's roots ([9.2.8](902-aot-pipeline.md#928-specialize-monomorphization)).

| Section   | Contents                                                    |
|-----------|-------------------------------------------------------------|
| `.text`   | Code of concrete `fn`s                                      |
| `.rodata` | `const`, literals, [ScanTable](../008-abi/802-scan-table.md) |
| `.data`   | Initial values of `static`s                                 |
| `.bss`    | Zero-initialized `static`s                                  |

ScanTables are ordinary constant data; generated registration code registers them with the runtime via `nabi_gc_register_type` at process start ([8.2.13](../008-abi/802-scan-table.md#8213-registration-time)). There are no nabi-specific sections.

## 9.3.3. Static and shared libraries

A library's output is a static `.a`; dependencies link into a program from archives, and the standard library code it reaches is compiled with it ([9.3.6](#936-standard-library)). `--crate-type=cdylib` ([9.1.2](901-nabic-cli.md#nabic-build-aot-build)) also links a library package into a shared library that C code (or Python, R, Julia) loads at run time: `lib<name>.so` (Linux), `lib<name>.dylib` (macOS, install name `@rpath/lib<name>.dylib`) or `<name>.dll` plus its import library `<name>.lib` (Windows). It holds the package, the standard library and the runtime, and exports only the package's `#[export] extern "C" fn`s and `#[export_name("sym")] extern "C" fn`s ([5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-definitions-exposed-externally)), under their own name or `sym`; every other symbol binds inside the library.

- Loading the library starts its runtime: the loader runs `nabi_library_init` (`DT_INIT`, a `__mod_init_func` entry, or the DLL entry point `nabi_dll_main` on process attach), which sets up the runtime on the loading thread, registers the library's GC types ([8.2.13](../008-abi/802-scan-table.md#8213-registration-time)) and parks the thread as one in C code. An export called on any thread attaches it first ([8.5](../008-abi/805-extern-symbols.md)), so C code needs no init call.
- Each shared library has its own runtime and heap: GC objects do not cross from one Nabi library to another, or to a Nabi program that loads it. Only C values cross the exports.
- On Linux the library's thread-locals use the initial-exec model (`DF_STATIC_TLS`): glibc places them in its static TLS area, which also has room for a library opened with `dlopen`. A library takes 160 bytes of it, the runtime's thread state ([8.3](../008-abi/803-tls-slots.md)); glibc keeps 512 bytes for libraries opened later (`glibc.rtld.optional_static_tls`), so three Nabi libraries fit in one process by default. More fail to load with `cannot allocate memory in static TLS block`; `GLIBC_TUNABLES=glibc.rtld.optional_static_tls=4096` makes room.
- Versions: `--lib-version <x.y.z>` ([9.1.2](901-nabic-cli.md#nabic-build-aot-build)), which `nabi build` takes from the manifest's `version`, sets the ELF soname to `lib<name>.so.<x>` (`lib<name>.so.0.<y>` below 1.0, where a minor version breaks), with a link of that name beside the file, and the Mach-O current and compatibility versions to `x.y.z` and `x.0.0` (`0.y.0` below 1.0).
- The C header: the build also writes `target/<profile>/include/<name>.h`, as `--emit=c-header` does. It declares each `extern "C"` function of the package that has an `#[export_name]`, under that name, and the types their signatures reach: a `#[repr(C)]` struct or union as a `typedef struct` with its fields in order (`_Alignas` for `align(N)`, `#pragma pack` for `packed`), a `#[repr(transparent)]` struct as a `typedef` of its field, a `#[repr(iN)]` enum as a `typedef` of its tag type with one `#define <ENUM>_<VARIANT> ((<Enum>)<value>)` per variant. Primitives map to `<stdint.h>` / `<stddef.h>` / `<stdbool.h>` types (`usize` is `size_t`, `isize` `ptrdiff_t`, `c_char` `char`, `char` `uint32_t`, a C-ABI `str` `const char *`), `ConstPtr<T>` to `const T *`, `Ptr<T>` / `ManyPtr<T>` to `T *`, `CPtr<T>` to `T *` with a `/* nullable */` comment, as is `T | none`, an `extern "C" fn` type to a function pointer, an `extern "system"` function to `NABI_SYSTEM` (`__stdcall` on 32-bit Windows), a never-returning one to `_Noreturn`. Each item's `///` doc comment becomes a C comment. A function whose types C cannot name (a generic struct, a struct without `#[repr(C)]`, an enum with data) is left out with a `/* skipped <name>: <why> */` line. The header has `#pragma once`, an include guard and `extern "C"` for C++.
- Targets: `aarch64-apple-darwin`, `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`, `x86_64-pc-windows-msvc` and `aarch64-pc-windows-msvc`, with Nabi's own linker (not `NABI_CC`). Others are an error that says to link the static library with the C toolchain instead.
- [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) calls into other shared libraries work on every target.

`--crate-type=staticlib` instead makes one static library C, Swift and Xcode link alone: `target/<profile>/static/lib<name>.a` (`-o` names it) holds the package, its dependencies, the standard library it reaches and the runtime, each archive's members renamed `<n>-<member>` so none hides another. The runtime starts on the first export called, which attaches its thread as a shared library's exports do; GC objects do not cross the exports either. The build also writes the C header (as above) and `target/<profile>/include/module.modulemap`, a clang module named after the package that links the library and the system libraries and frameworks the package names, so Swift says `import <name>`:

```
module geo {
    header "geo.h"
    link "geo"
    link framework "UIKit"
    export *
}
```

With `--xcframework` the build also puts the library and its headers into `target/<profile>/<name>.xcframework`, an XCFramework as `xcodebuild -create-xcframework` writes one: a slice per Apple target built (`ios-arm64`, `ios-arm64-simulator`, `macos-arm64`), each with `lib<name>.a` and `Headers/`, and an `Info.plist` listing them. Each build adds or replaces its target's slice, so a build for `aarch64-apple-ios` and one for `aarch64-apple-ios-sim` make one XCFramework that Xcode takes for both; the device target is in progress ([0.6.2](../000-meta/006-platform-targets.md#062-in-progress)), so for now an XCFramework holds the simulator's slice. `--xcframework` without `--crate-type=staticlib`, or for a target that is not Apple's, is an error that says what to write.

It is how an iOS app takes Nabi code ([0.6.1](../000-meta/006-platform-targets.md#061-supported)): an archive for `aarch64-apple-ios-sim`, and once the device target is supported one for `aarch64-apple-ios`, added to the Xcode project with the `include` directory on its header search path. On a wasm target `--crate-type=staticlib` is an error.

## 9.3.4. Version compatibility check

When a dependency's NABIHIR is read, its `version` and producer (the `nabic` that produced that NABIHIR) are compared with the current `nabic`. If they differ, it is an error stating what to rebuild ([9.4.9](904-nabihir-bytecode.md#949-reading-and-rejection)).

## 9.3.5. Symbol exposure

The symbols an archive exposes are the global symbols of its objects, following [Symbol mangling](905-symbol-mangling.md).

```
nm libfoo.a
0000000000000000 T _Y3f8a1c2b_foo_geom_distance_a7c91d8e44b2f190
0000000000000040 T _Y3f8a1c2b_foo_geom_Point_origin_b1234567...
```

## 9.3.6. Standard library

The standard library (`bare` / `core` / `std`) is not distributed as archives, and no build links a prebuilt one. Every build compiles it from source (`--bootstrap`) with the package, and only the part the program reaches: from the program's roots (`main`, the `#[test]` and `#[bench]` functions of a harness, `#[export_name]` and WIT exports, the functions the compiler and the runtime call, and every static's initializer) the front end follows calls, function values, the `Drop` implementations of the types it uses and the implementations an `any Trait` value dispatches to, and type-checks and lowers each function it meets ([9.2.6](902-aot-pipeline.md#926-type-checking)). Of the rest it reads only names and signatures, for name resolution, method lookup and diagnostics. `--check-library` checks and compiles every library body instead ([9.1.2](901-nabic-cli.md#nabic-build-aot-build)).

The standard library functions the program reaches are compiled by codegen units, one per module (9.2.16). Each function's optimized code and each unit's object are kept in the toolchain's function cache (`lib/nabi-cache/fn/`, [9.2.19](902-aot-pipeline.md#9219-installation-layout)) by a key over everything they depend on, and every later build of the toolchain whose key is the same takes them from there, whatever package it builds. The optimizer still sees the whole program, the standard library's functions included, so it inlines them as it inlines the package's own (9.2.10).

A library package's archive holds private copies of the standard library code it reaches, as of its generic instances. The standard library's statics are the exception: a library defines them weak under their own symbols, so the program and every library linked into it share one of each.

## 9.3.7. Linking

```
cc main.o libfoo.a libnabir.a -o my_binary
```

The linker links only the needed symbols from the archive's objects. Functions exposed with `#[export_name]` can be called directly from C.

```c
extern uint32_t nabi_compute(int32_t a, int32_t b);   // #[export_name("nabi_compute")]
int main(void) { return (int)nabi_compute(1, 2); }
```

```
cc main.c libfoo.a libnabir.a -o my_binary
```

With `--crate-type=staticlib` (9.3.3) the one archive is enough:

```
cc main.c target/release/static/libfoo.a -o my_binary
```

## 9.3.8. Stability

The archive format follows the [ABI stability policy](../008-abi/README.md); the NABIHIR format follows [9.4](904-nabihir-bytecode.md).
