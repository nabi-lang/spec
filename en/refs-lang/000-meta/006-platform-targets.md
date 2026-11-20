# 0.6. Platform targets

Nabi builds for the built-in targets below, and for custom targets of the same architectures and object formats (0.6.5), from any host (`--target`, [0.2.4](002-pipeline.md#024-code-generation)). Every target is in one of three states: supported (0.6.1), in progress (0.6.2) or not supported (0.6.3).

## 0.6.1. Supported

The supported targets are built in: `nabic --print target-list` lists them, and `--target` takes each.

### 64-bit desktop / server

This group covers the major architectures of Linux, macOS and Windows, including musl variants for static binaries, and FreeBSD.

| Target                       | Notes                                      |
|------------------------------|--------------------------------------------|
| `x86_64-unknown-linux-gnu`   | Primary Linux server / desktop             |
| `aarch64-apple-darwin`       | macOS Apple Silicon                        |
| `aarch64-unknown-linux-gnu`  | ARM servers (AWS Graviton, Raspberry Pi 5, etc.) |
| `x86_64-pc-windows-msvc`     | Windows desktop                            |
| `aarch64-pc-windows-msvc`    | Windows ARM                                |
| `x86_64-unknown-linux-musl`  | Alpine / `static` binary                   |
| `aarch64-unknown-linux-musl` | Same, ARM                                  |
| `x86_64-unknown-freebsd`     | FreeBSD 14                                 |

### WebAssembly

Three WebAssembly targets are supported, covering browser-based and server-side execution. WASI sockets (preview 2 components only) and WASI threads (a preview 1 core-module proposal) cannot be combined in one module today, so server-side WASM has two targets.

| Target                   | Notes                            |
|--------------------------|----------------------------------|
| `wasm32-unknown-unknown` | Browser environment; `target_os = "unknown"` ([6.3.2](../006-meta-system/603-cfg.md)) |
| `wasm32-wasip2` | Server-side WASM: a WASI preview 2 component. `wasi:sockets` networking; no threads (`thread::spawn` panics; `thread::Builder::spawn` throws `Unsupported`) |
| `wasm32-wasip1-threads` | Server-side WASM with threads: a WASI preview 1 core module using wasi-threads (shared memory). Networking only on listeners the host passes in; connect / bind / DNS throw `Unsupported` |

> [!NOTE]
> wasmtime removed wasi-threads after v46 (WAMR's iwasm and wasmer still implement it), so `wasm32-wasip1-threads` will depend on WASI shared-everything-threads in the future.

These names follow the rest of the ecosystem, where `wasm32-wasi` means WASI preview 1; Nabi has no target named `wasm32-wasi`, `wasm32-wasi-threads` or `riscv32imac-unknown-none`, and `--target` with one of them is an unknown target that names the closest built-in one ([9.1.2](../009-tools/901-nabic-cli.md#912-core-subcommands)).

### 32-bit

For embedded and legacy environments still in active use, the following four 32-bit targets are supported.

| Target                          | Notes                                |
|---------------------------------|--------------------------------------|
| `i686-unknown-linux-gnu`        | 32-bit x86 Linux                     |
| `i686-pc-windows-msvc`          | 32-bit Windows                       |
| `armv7-unknown-linux-gnueabihf` | 32-bit ARM Linux, hard float (Raspberry Pi 2 to 4 on a 32-bit OS, etc.) |
| `armv7-unknown-linux-musleabihf` | Same, musl: a `static` binary |

### Mobile

| Target                  | Notes                                     |
|-------------------------|-------------------------------------------|
| `aarch64-apple-ios-sim` | iOS simulator on Apple Silicon Macs       |

`aarch64-apple-ios-sim` (the iOS simulator on Apple silicon Macs) builds for iOS 13.4 and later, the first iOS whose loader reads chained fixups, the format Apple's linker writes: `IPHONEOS_DEPLOYMENT_TARGET`, or `deployment_target: { ios: "16.0" }` in nabi.nabion ([9.10](../009-tools/910-nabi-frontend.md)), raises the version, and an older one is an error that names the oldest. Linking needs Xcode's iOS SDK: `SDKROOT`, else the `iPhoneSimulator` SDK of the developer directory (`DEVELOPER_DIR`, else what `xcode-select` names); without one the build says to install Xcode. An `SDKROOT` that is another platform's SDK (a macOS one for an iOS target, as its `SDKSettings.json` or directory name says) is passed over with a note, and the developer directory's SDK is used. Objects and executables name their platform (`LC_BUILD_VERSION`: the iOS simulator), and the linker refuses an object built for another Apple platform. The baseline CPU is `apple-m1`. `target_os` is `"ios"` and `target_abi` is `"sim"` ([6.3.2](../006-meta-system/603-cfg.md)). Executables are signed ad hoc, and the simulator runs them: `nabi run`, `nabi test` and `nabi bench` run one through the target's `simrun.sh`, which runs `xcrun simctl spawn` on the booted device (or the device `NABI_SIM_DEVICE` names) and passes every environment variable on as `SIMCTL_CHILD_<name>`, the only ones `simctl` hands the program. An app's Nabi code is a library that Xcode links: `--crate-type=staticlib` ([9.3.3](../009-tools/903-package-archive.md#933-static-and-shared-libraries)) builds one archive with the package, its dependencies and the runtime, and Swift reads its module map (`import <name>`). `--crate-type=plugin` and `--hot` work on the simulator; the REPL does not run there. The maintainers run the simulator corpora on their own machines.

The device target, `aarch64-apple-ios` (iPhones and iPads), is in progress (0.6.2). The text below applies only to it. It builds for iOS 13.4 and later as the simulator does, links with the `iPhoneOS` SDK, and names the iOS platform in `LC_BUILD_VERSION`; its baseline CPU is `apple-a7` (no LSE atomics, as on the A9), and it sets no `target_abi`. A device runs an executable only once the developer signs it for it with Xcode, so `nabi run`, `nabi test` and `nabi bench` are an error that names the simulator target. On a device an app cannot start processes, so `Command` throws `Unsupported` there, and nothing loads code at run time: `--crate-type=plugin` and `--hot` are errors.

### Android

| Target                  | Notes                                     |
|-------------------------|-------------------------------------------|
| `x86_64-linux-android`  | Android x86-64 (emulator), API level 29 and later |

`x86_64-linux-android` builds executables and shared libraries (`--crate-type=cdylib`, which `System.loadLibrary` loads) for Android 10 (API level 29) and later; packaging them into an app is the Android build's job. API level 29 is the lowest because bionic, Android's C library, runs ELF thread-local storage only from there, and the runtime keeps each thread's state in it. `ANDROID_PLATFORM=<n>` (or `android-<n>`, as the NDK's own build tools take it) builds for a later level; a level below 29 is an error that says to write `ANDROID_PLATFORM=29` or higher. A link takes the C runtime objects and bionic's libraries of that level from the Android NDK, which is not part of the toolchain: `ANDROID_NDK_HOME`, else `ANDROID_NDK_ROOT`, else the newest `ndk/<version>` of the Android SDK (`ANDROID_HOME`, else `ANDROID_SDK_ROOT`, else Android Studio's place for it). Without an NDK, the error says where it looked and to set `ANDROID_NDK_HOME`; an NDK without the level's libraries is an error that names the levels it has. `nabicc` takes bionic's headers from the same NDK ([9.13.7](../009-tools/913-c-compiler.md#9137-headers-and-c-library)). `target_os` is `"android"` and `target_env` is absent. The kernel is Linux, so what `std` has under `target_os = "linux"` (`std::os::linux`, the `open` flags, ...) is there on Android too. An executable is a PIE for `/system/bin/linker64`. `x86_64-linux-android` builds for the x86_64 Android ABI, `x86-64-v2` (SSE4.2, POPCNT), and for pages of up to 16 KiB. An app's process runs under a seccomp filter that kills it for system calls bionic does not make, so the runtime makes none of its own there: `ChildExt::pidfd` throws (`ENOSYS`), `Metadata::created` throws `Unsupported`, and a copy from file to file goes through `sendfile`. Before Android 14 (API level 34) bionic's `posix_spawn` cannot change the child's directory, so a `Command` with `current_dir` forks and execs there.

The device target, `aarch64-linux-android` (Android arm64, API level 29 and later), is in progress (0.6.2). It builds as `x86_64-linux-android` does, and the text above applies to it too, except that it builds for pages of up to 64 KiB.

These targets cover most users in science, data, Jupyter and desktop work.

### 32-bit support

Because 32-bit targets are supported, the specification as a whole is defined to work in both 32-bit and 64-bit environments. The object header layout is the same at both bit widths. Atomic operations on 64-bit integers use the target's native 64-bit atomic instructions on 32-bit environments (`cmpxchg8b` on i686, and `ldrexd` / `strexd` on ARMv7) or, where none exist, the system atomic library (riscv32, in progress: the runtime's, for its one thread). The bit width of `usize` follows the target's native pointer width automatically. On wasm32 the heap is linear memory, grown on demand up to the 4 GiB wasm32 limit. On other 32-bit targets the heap available to the garbage collector is about 3.5 GB: the block heap reserves at most 2 GiB contiguous (less if the OS refuses), the rest of the address space holds large objects, and Windows x86 images are large-address-aware. On riscv32 the heap is the memory the board hands out (0.6.2). On 64-bit it is effectively unlimited. Time uses 64-bit interfaces (`time64`) on 32-bit targets too, so `SystemTime` has the same range everywhere.

### Testing

CI builds and tests the supported targets as the table below shows. The levels are `nabic` opt levels, and the corpora are the compiler's end-to-end and differential test suites.

| Target | Every pull request | Merge queue (required to merge) | Nightly |
|---|---|---|---|
| `x86_64-unknown-linux-gnu` | all tests, corpora at `-O0` | all tests, corpora at `-O0` | all tests; corpora at `-O0`, `-O1`, `-O2`, `-O3`, `-Os` and `-O2 -g`; GC stress at `-O2` |
| `x86_64-pc-windows-msvc` | all tests, corpora at `-O0` | all tests, corpora at `-O0` | all tests; corpora at `-O0` and `-O2` |
| `aarch64-apple-darwin` | | all tests, corpora at `-O0` | all tests; corpora at `-O0`, `-O1`, `-O2`, `-O3`, `-Os` and `-O2 -g` |
| `aarch64-unknown-linux-gnu` | | all tests, corpora at `-O0` | all tests; corpora at `-O0` and `-O2`; GC stress at `-O2` |
| `aarch64-pc-windows-msvc` | | | all tests; corpora at `-O0` and `-O2`, with the arm64 toolchain and with the x64 one under emulation |
| `x86_64-unknown-linux-musl`, `aarch64-unknown-linux-musl` | | | corpora at `-O0` and `-O2` on a host that runs the target |
| `i686-pc-windows-msvc` | | | corpora at `-O0`, `-O2` and `-O2 -g` |
| `i686-unknown-linux-gnu` | | | corpora at `-O0` and `-O2` |
| `armv7-unknown-linux-gnueabihf` | | | corpora at `-O0` and `-O2` under qemu-user |
| `wasm32-wasip2`, `wasm32-wasip1-threads`, `wasm32-unknown-unknown` | | | corpora at `-O0` and `-O2` under wasmtime and node |
| `x86_64-unknown-freebsd` | | | corpora at `-O2` in a FreeBSD 14 virtual machine |

A change merges only when the merge queue run passes. A failing nightly job opens an issue. The nightly run also builds the corpora of three in-progress targets (0.6.2), `riscv64gc-unknown-linux-gnu`, `powerpc64le-unknown-linux-gnu` and `s390x-unknown-linux-gnu`, at `-O0` and `-O2` under qemu-user. The other combinations of target and opt level are run on the maintainers' own test machines before a release, not in CI; `armv7-unknown-linux-gnueabihf` and `armv7-unknown-linux-musleabihf` run there natively, at every level, on an arm64 machine whose CPU also runs 32-bit ARM code (Neoverse N1); `x86_64-linux-android` is tested only there, in the Android emulator, and `aarch64-apple-ios-sim` in the iOS simulator. `wasm32-unknown-unknown` runs its corpora at `-O0` and `-O2` there in Chromium, Firefox and WebKit too, each program in a dedicated Worker of a page. There, too, the WASI targets run on runtimes besides wasmtime, corpora at `-O0` and `-O2`: `wasm32-wasip2` under jco (the component transpiled to JavaScript, on Node.js), `wasm32-wasip1-threads` under WAMR's iwasm and under wasmer. A component's exports are called under jco too, and a case one of these runtimes cannot run (a bug or gap of the runtime) is skipped there.

## 0.6.2. In progress

The targets below are built in, but not yet tested on the hardware they are for, so they are not supported. Each is to be supported in a later release, before 0.2.0, and then moves to 0.6.1. `--target` with one of their names (to `nabic`, `nabicc` or `nabi`, `--print target-spec` too) is an error that says the target is in progress and what to build for now; `nabic --print target-list` leaves them out.

| Target | Why it is in progress | For now |
|--------|-----------------------|---------|
| `aarch64-apple-ios` (iOS devices) | builds and links; not yet run on an iPhone or iPad | `aarch64-apple-ios-sim`, in the iOS simulator |
| `aarch64-linux-android` (Android arm64) | tested under qemu-user with an Android system image; not yet on a device | `x86_64-linux-android`, in the Android emulator |
| `riscv64gc-unknown-linux-gnu` (64-bit RISC-V Linux) | tested under qemu-user; not yet on RISC-V hardware | none |
| `powerpc64le-unknown-linux-gnu` (IBM Power) | tested under qemu-user; not yet on POWER hardware | none |
| `s390x-unknown-linux-gnu` (IBM Z; big-endian) | tested under qemu-user; not yet on IBM Z | none |
| `riscv32imac-unknown-none-elf` (RISC-V embedded, no operating system) | tested under qemu-riscv32; not yet on a board | none |

Their code stays in the toolchain while they are tested. The maintainers build them with `NABI_IN_PROGRESS_TARGETS=1` in the environment, which lets `--target` and `--print target-list` take them. It is a switch for testing the toolchain, not part of Nabi: it promises nothing, and may change or go away in any release. What this page says of these targets describes them as they build now.

### `riscv32imac-unknown-none-elf`

`riscv32imac-unknown-none-elf` has no operating system. A program links with the runtime and no C library into one static image, and runs on one thread: as on `wasm32-wasip2`, `thread::spawn` panics and `thread::Builder::spawn` throws `Unsupported`. Files, processes, sockets and C library functions are not there, and a program that uses them does not link; there are no environment variables (setting one throws `Unsupported`), and `random` panics for want of an entropy source. The runtime brings the image's entry point `_start` and its heap; `_start` takes the arguments from the initial stack as a Linux process starts, sets up the TLS block and calls `main`. Memory, output, time and the program's end come from four board functions: `nabi_bare_morecore(len)`, `nabi_bare_write(fd, ptr, len)`, `nabi_bare_clock_ns(realtime)` and `nabi_bare_exit(code)`. Its own definitions of `_start` and of these are weak and make Linux system calls, which is how `qemu-riscv32` runs the image in user mode; a board defines its own, and its startup sets up `sp` and `gp` and calls `nabir_bare_start(null)` (no arguments). Floating point, 64-bit division and the math functions come from the runtime (compiler_builtins), so results of `exp`, `ln` and the like may differ from glibc's in the last bit.

## 0.6.3. Not supported

Every other target is not supported. `--target` with its name is an unknown target ([9.1.2](../009-tools/901-nabic-cli.md#912-core-subcommands)). For the names below the error says why, and what to use instead where there is something.

| Target | Instead |
|--------|---------|
| `arm-unknown-linux-gnueabi` (ARMv6, soft float: Raspberry Pi Zero / 1) | none |
| `armv7-linux-androideabi` (32-bit Android) | none; `aarch64-linux-android` is in progress (0.6.2) |
| `thumbv6m-none-eabi`, `thumbv7m-none-eabi`, `thumbv7em-none-eabi`, `thumbv7em-none-eabihf`, `thumbv8m.main-none-eabi`, `thumbv8m.main-none-eabihf` (Cortex-M) | none |
| `x86_64-apple-ios` (iOS simulator on Intel Macs) | `aarch64-apple-ios-sim` (0.6.1), on an Apple Silicon Mac |
| `x86_64-apple-darwin` (Intel Macs) | `aarch64-apple-darwin`, on an Apple Silicon Mac |

## 0.6.4. SIMD

Builds target the triple's baseline CPU, so binaries are portable: x86-64 baseline (SSE2) for `x86_64`, `pentium4` (SSE2) for `i686`, `apple-m1` for `aarch64-apple-darwin` and `aarch64-apple-ios-sim`, `apple-a7` (NEON, no LSE atomics) for `aarch64-apple-ios`, armv8-a (NEON, no LSE atomics) for other `aarch64` targets, ARMv7 with NEON for `armv7`, POWER8 (VSX) for `powerpc64le`, z10 (no vector facility) for `s390x`, and SIMD128 for `wasm32`. Above `-O0` the code generator vectorizes loops for that baseline on `x86_64`, `i686`, `aarch64`, `armv7` (integer lanes only), `powerpc64le` and `wasm32`, and with their vector features on `s390x` and `riscv64` ([8.7.8](../008-abi/807-code-generation.md#878-loop-vectorization)): a loop over list elements, the one loop a fused dot expression becomes ([3.3.4](../003-expressions/303-broadcast-operators.md#334-automatic-loop-fusion)) among them, may run in 128-bit vectors. A vectorized loop computes what the scalar loop computes and traps where it traps. On the other targets every loop is scalar code. `-C target-cpu=<cpu>|native` and `-C target-feature=+avx2,...` ([9.1](../009-tools/901-nabic-cli.md)) pick the CPU's features, and the code generator uses some of them on `x86_64`: with `avx2` (`x86-64-v3`, `haswell`, `znver1` and later CPUs) vector loops use 256-bit AVX2 vectors and store under a mask where a test picks the lanes, and with `popcnt`, `lzcnt` and `bmi` the bit counts (`count_ones`, `leading_zeros`, `trailing_zeros`) are one instruction each; with `fma` a `mul_add` is one `vfmadd` instruction, scalar and in vector loops, but a multiplication and an addition the program does not join in `mul_add` are never fused, which would round differently. On `aarch64` it uses `lse`, the ARMv8.1 atomics (`apple-m1`, `cortex-a76`, `neoverse-n1` and later CPUs): an atomic read-modify-write, exchange or compare-exchange is one instruction (`ldadd`, `swp`, `cas`, ...); without it, as on the baseline of `aarch64` Linux and Windows, each is an exclusive load / store loop (`ldxr` / `stxr`, `ldaxr` / `stlxr` as the ordering asks) that runs on every ARMv8.0 CPU (Cortex-A53, Cortex-A72, ...). On `aarch64` Linux above `-O0` each of these also tests a flag the runtime sets at start from the kernel's `AT_HWCAP`, and takes the LSE instruction when the CPU has LSE. With `rcpc`, the ARMv8.3 RCpc loads (`apple-m1`, `cortex-a76`, `neoverse-n1` and later CPUs), an atomic load that asks for `Acquire` is `ldapr` above `-O0`, which an earlier release store may pass, as an acquire load allows; else it is `ldar`. With `dotprod` (the same CPUs) a loop that sums products of bytes (`s += (a[i] as i32) * (b[i] as i32)`, of `i8` or `u8`) runs sixteen products at a time, four to a lane (`sdot`, `udot`). `fullfp16`, `crc`, `sve` and `sve2` change no code. On `s390x` it uses `vector`, the vector facility of `z13` and later CPUs: vector loops of 128 bits, with float lanes of 8 bytes, of 4 too with `vector-enhancements-1` (`z14` and later); the baseline, z10, has no vector registers. On `riscv64` it uses `v` (RVV 1.0, `-C target-feature=+v`): vector loops of 128 bits, which every V machine holds, whatever its vector length. On `riscv64` and `riscv32` it uses `zbb`: a rotate of a 32- or 64-bit integer by a constant (`rotate_left`, `rotate_right`, or the two shifts and an or that spell one) is one instruction (`rori`), else two shifts and an or. On `armv7` it uses `vfp4`: a `mul_add` is one `vfma` instruction. `-C target-feature=-neon` on `armv7`, `-vsx` on `powerpc64le` and `-simd128` on `wasm32` build with every loop scalar (`+simd128`, the default, turns SIMD128 back on). Other features, and every feature on `i686`, change no code: a build naming only those warns so. A CPU name stands for its features (`x86-64-v2`, `x86-64-v3`, `x86-64-v4`, `haswell`, `znver4`, `apple-m1`, `neoverse-n1`, `pwr9`, `z13`, ...); an unknown one is an error that lists the known ones. `native` adds the features of the build machine, detected when `nabic` runs, to the baseline's: the binary may not run on another machine. Apart from that flag, the generated code does no feature detection at run time, and nothing is compiled in several versions to pick from at run time, `par::map` included ([stdlib 2.19.6](../../refs-standard-libraries/200-core/219-par.md#2196-simd--gpu-integration)).

Portable SIMD vectors, `Simd<T, N>` with lane-wise operators, are `core::simd` ([stdlib 2.24](../../refs-standard-libraries/200-core/224-simd.md)); their operations are vector code on the targets above at every optimization level, and lane loops elsewhere.

## 0.6.5. Target specifications

Every target is described by one target specification; the compiler, the linker and the runtime build read nothing else about a target. A target is named by `--target`: a built-in name (`nabic --print target-list`: the targets of 0.6.1), or the path of a file `<name>.nabion` holding a NABION value `TargetSpec { .. }` (a custom target, named by the file's stem).

`nabic --print target-spec [--target=<t>]` prints a target's specification in that form, so a custom target usually starts as a copy of a built-in one. A custom file must give every field; an unknown field is an error. A custom target's runtime and, where the target ships one, its C library come from `lib/nabi/targets/<name>/` of the toolchain; without that directory, a build that links fails naming it.

| Field | Meaning |
|-------|---------|
| `c_target`, `deployment_target` | The target triple as clang takes it (`NABI_CC`, and the C front end of `nabi bindgen`); the minimum OS version appended to it (Mach-O; `MACOSX_DEPLOYMENT_TARGET` overrides on macOS, `IPHONEOS_DEPLOYMENT_TARGET` on iOS; Android: the API level, `ANDROID_PLATFORM` overrides) |
| `arch`, `os`, `env`, `abi`, `family`, `endian`, `pointer_width` | The `target_*` cfg values ([6.3.2](../006-meta-system/603-cfg.md)); an empty `env` / `abi` / `family` sets no key. `abi` is `"sim"` on an Apple simulator, else empty |
| `vendor` | Informational |
| `wide_align`, `max_atomic_width`, `c_abi` | `i64` / `u64` / `f64` alignment, the widest atomic, the C calling convention ([8.6.3](../008-abi/806-calling-convention.md)) |
| `cpu`, `features` | The default CPU and features in clang's names (the SIMD baseline of 0.6.4) |
| `object_format`, `linker`, `pre_link_args`, `crt_objects`, `late_link_args` | Object format (`ObjectFormat::MachO`, `Elf`, `Coff`, `Wasm`); what `nabic`'s linker makes (`Linker::MachO`, `Elf`, `Coff`, `Wasm`, or `WasmComponent`: a wasm module made into a WASI preview 2 component), which also sets the command-line style of the link arguments; the target's own linker arguments; C runtime objects; arguments or files after the inputs |
| `dynamic_linking`, `dynamic_loader`, `libc` | Shared-library support, the ELF interpreter, the C library and whether it ships with the toolchain |
| `rust_target` | The Rust target the runtime is built for |
| `exe_suffix`, `staticlib_prefix` / `staticlib_suffix`, `dylib_prefix` / `dylib_suffix`, `c_symbol_prefix` | File and symbol naming |
| `stack_size`, `stack_probes`, `threads` | The main thread's stack, stack probing, thread support |
| `native_backend`, `repl` | The code generator takes the target; the REPL host runs on it |

A specification has a stable hash; every build cache keys on it.
