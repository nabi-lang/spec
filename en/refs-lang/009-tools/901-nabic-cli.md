# 9.1. `nabic` CLI interface

This section defines the command-line interface of the compiler `nabic`. The toolchain consists of four binaries.

| Binary   | Role                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `nabic`   | Compiler (the counterpart of rustc). Builds, checks and emits packages and builds test and bench harnesses (this section) |
| `nabi`    | Cargo-style front-end. Owns the manifest, dependencies, environments and toolchain pin, invokes `nabic`, and runs the tools (run, lint, REPL, Jupyter, LSP, formatter, docs, ...) with the compiler linked in ([9.10](910-nabi-frontend.md)) |
| `nabicc`  | C compiler. The same compiler as `nabi cc` ([9.13](913-c-compiler.md))                 |
| `nabils`  | LSP server executable for editors. The same server as `nabi lsp`                       |

The runtime is `libnabir.a` (`nabir`). There are no separate tools (`nabi-fmt` etc.). `nabic` does not resolve dependencies; dependency packages are passed by `nabi` via `--extern` ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)).

## 9.1.1. General form

`nabic` is invoked in the following general form.

```
nabic <subcommand> [options] [args]
```

As grammar rules:

```
cli_invocation  ::= "nabic" subcommand option* arg*
subcommand      ::= "build" | "emit" | "test" | "bench" | "check"
                  | "version" | "help"
option          ::= "--" identifier ("=" value)?
                  | "-" letter
```

`--name value` is equivalent to `--name=value`. Global options (9.1.5) may appear before or after the subcommand.

The subcommand, options, their values and paths must be valid UTF-8. An argument that is not is a usage error (9.1.6) that names its position: ``argument 3 (`b�c`) is not valid UTF-8; only a program's arguments, after `--`, may be other bytes``. The arguments after `--` need not be: `nabi run` passes them to the program as the system gave them (on Windows, UTF-16 with unpaired surrogates as well; [9.10.8](910-nabi-frontend.md#nabi-run-build-and-run)).

| Area                | Subcommands                                           |
|---------------------|-------------------------------------------------------|
| Build               | `build`, `emit`, `test`, `bench`, `check`             |
| Other               | `version`, `help`                                     |

The tools (`run`, `lint`, `repl`, `jupyter`, `lsp`, `fmt`, `doc`, `clean`, `bindgen`, `inspect`, `demangle`, `completions`) are subcommands of `nabi` ([9.10.8](910-nabi-frontend.md#9108-tools)).

### Package selection (`path`)

The `[path]` of `build` / `emit` / `test` / `bench` / `check` (and of `nabi`'s tools) is one of the following. If omitted, it is the current directory.

| `path`                       | Package                                                                                                 |
|------------------------------|---------------------------------------------------------------------------------------------------------|
| Package directory            | That directory. If it has no `src/main.nabi` or `src/lib.nabi`, the search moves upward (9.2.2)           |
| File in a package's `src/`   | That package: the first directory containing `src/main.nabi` / `src/lib.nabi`, walking upward from the file |
| Any other file               | An anonymous executable package consisting of that single file ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)), including a `.nabi` file in a package directory outside its `src/` |

It is an error if a directory is given and no package is found walking upward. An anonymous package's file submodules (`mod foo;`) are beside its file ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)).

The package name (the output name) is the name of the package directory; for an anonymous package it is the file name without `.nabi`.

## 9.1.2. Core subcommands

### `nabic build`: [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) build

`nabic build` builds a package AOT.

```
nabic build [path]
```

`[path]` follows the package selection rules above. Output is placed in the per-profile `target/` of [9.2.15](902-aot-pipeline.md#9215-output). `src/main.nabi` produces `target/<profile>/<name>`; `src/lib.nabi` produces `target/<profile>/lib<name>.a` plus the interface `target/<profile>/<name>.nabihir` ([9.4](904-nabihir-bytecode.md)); a package with both produces both. The profile is `release` for `-O2` / `-O3` / `-Os` (including `--release`) and `debug` otherwise; with `--target=<target>` the directory is `target/<target>/<profile>/`. If `--target-dir=<dir>` or `NABI_TARGET_DIR` is set, that directory is used instead of `target/`.

A lone file ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)) is built as an anonymous package in a temporary directory, so no `target/` is created next to the file, and its executable is copied next to the file as its stem plus the target's executable suffix (`hello.nabi` gives `hello`, `hello.exe` on Windows), unless `-o` names another file. A file already at that path that an earlier build of the script did not write is left alone: the build fails and names `-o <file>`.

Options:

- `--release` performs an optimized build (equivalent to `-O2`) in the `release` profile; the default is a debug build (`-O0`) in the `debug` profile. The profiles differ as [9.2.15](902-aot-pipeline.md#9215-output) describes.
- `--target-dir=<dir>` relocates `target/` (equivalent to `NABI_TARGET_DIR`).
- `--crate-type=lib|cdylib|staticlib|plugin` picks a library package's output. `lib` (the default) is the static archive and its NABIHIR; `cdylib` also links a shared library for C code next to them, `lib<name>.so`, `lib<name>.dylib` or `<name>.dll` with `<name>.lib` ([9.3.3](903-package-archive.md#933-static-and-shared-libraries)); `-o` names that file. It also writes the library's C header `target/<profile>/include/<name>.h` ([9.3.3](903-package-archive.md#933-static-and-shared-libraries)). `staticlib` makes one static library C, Swift and Xcode link alone, `static/lib<name>.a` (`static/<name>.lib` on Windows; `-o` names it), with the header and `include/module.modulemap` ([9.3.3](903-package-archive.md#933-static-and-shared-libraries)); for a wasm target it is an error. With it, `--xcframework` also adds the target's slice to `<profile>/<name>.xcframework` (Apple targets). `plugin` links a plugin a Nabi program loads instead, the same file names without the `.lib` and with no header ([9.12](912-plugins.md)); for `aarch64-apple-ios` (in progress, [0.6.2](../000-meta/006-platform-targets.md#062-in-progress)), whose devices load only code signed into the app, it is an error that names `aarch64-apple-ios-sim`. On a package without `src/lib.nabi` it is an error, and `plugin` on a package that also has `src/main.nabi`, or for a wasm or musl target, too.
- `--lib-version <x.y.z>` gives a `cdylib` its version: on Linux its soname is `lib<name>.so.<x>`, or `lib<name>.so.0.<y>` below 1.0, and a link of that name is made beside the library; on macOS it sets the current version (`x.y.z`) and the compatibility version (`x.0.0`, or `0.y.0` below 1.0) of its install name. Without it the soname is the file name and both versions are 1.0.0. `nabi build` passes the package's `version` ([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)). A value that is not three numbers is an invalid option.
- `--bundle=<dep>` (with `--crate-type=plugin`) links dependency `<dep>` into the plugin instead of taking it from the program that loads it.
- `--hot` builds a hot-reload plugin, what `nabi dev` loads into a running program (implies `--crate-type=plugin`). A package with only `src/main.nabi` is built as the root plugin, its `main` exported as `<name>::main`; a package with both root files is an error. Every call to a function of a hot package goes through a slot the hot runtime rebinds, its statics bind to the first version's storage, its types are described in full, and its package hash is the same for every build of the package (its name's), so a new version keeps the names of the old. The compiler also writes `<output stem>.layout` beside the plugin: the version number and the layout of each type of the package. `--hot` builds at `-O0` only: `--release`, `-O1` .. `-O3` and `-Os` are errors ([8.4.13](../008-abi/804-safepoint.md#8413-hot-reload-code)).
- `--hot-base=<file>` (with `--hot`) names the `.layout` of the version the program runs. The new version is numbered after it and checked against it: a field added without a default, a field whose type changed, a `#[renamed_from]` that names no old field, or changed type parameters or kind is an error (E0149) that says what to write instead.
- `--hot-dep=<dep>` (with `--hot`) says dependency `<dep>` is a hot package too: its functions are called through its own plugin's slots instead of compiled in, and its package hash is pinned the same way.
- `--extern <name>=<dir | src/lib.nabi | <name>.nabihir | lib<name>.a>` mounts one dependency package at the top-level path `name` ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)). Source (`dir` / `src/lib.nabi`) is compiled together in the same build; a prebuilt package (`.nabihir` / `.a`; given one, its sibling is located) has its NABIHIR read and its archive linked. May be given multiple times and is normally filled in by `nabi`. `name` is an identifier: the package name with `-` replaced by `_` ([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)). `std`, `core` and `bare` are the standard library's: a library package named so is rejected, and `--extern std=<path>` is a usage error unless an earlier `--extern` names the same path, so that it is another name of that dependency (`--extern helper=p --extern std=p`, which code writes `std::..`; `::std::..` is still the library). So are a keyword and a path start (`pkg`, `sup`), as `name` and as the name of a library or of a package with one ([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)).
- `--extern-feature <name>=<f>` builds the source dependency `name` with `cfg(feature = "<f>")`. Repeatable, one per feature. A dependency sees only these: the `feature` keys of `--cfg` belong to the package being compiled. Normally filled in by `nabi`.
- `--name <n>` sets the package name (naming rules of [9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)). The outputs are `lib<n>.a` / `<n>.nabihir` and the mount name is the name with `-` replaced by `_`. The default is the directory name. `--extern a-b=..` is likewise bound as `a_b`.
- `--bootstrap <file>` gives a standard library root to preload (may be repeated); `--nabir <lib>` gives the runtime to link. The defaults are the sysroot's `lib/nabi/{core,std,bare}/src/lib.nabi` and `lib/libnabir.a` ([9.2.19](902-aot-pipeline.md#9219-installation-layout)). A bootstrap root's tier is the name of the directory above its `src/`; given `core/src/lib.nabi` without a `bare` root, `bare/src/lib.nabi` beside it is loaded too. If not found, it is an error that lists the paths searched.
- `-O0` / `-O1` / `-O2` / `-O3` / `-Os` select the optimization level explicitly ([9.2.10](902-aot-pipeline.md#9210-optimization-levels)): `-O0` none; `-O1` loops optimized as at `-O2`, little code added elsewhere, with debug information; `-O2` speed; `-O3` more speed for more code (larger inlining in loops, scalar loops unrolled); `-Os` size, loops still counted and vectorized where short.
- `--target=<target>` selects the target platform: a built-in name, one of the supported targets of [Platform targets](../000-meta/006-platform-targets.md#061-supported) (an in-progress one only with the maintainers' testing switch, [0.6.2](../000-meta/006-platform-targets.md#062-in-progress)), or a `<name>.nabion` target specification file ([0.6.5](../000-meta/006-platform-targets.md#065-target-specifications)). Any target can be built from any host ([0.2.4](../000-meta/002-pipeline.md#024-code-generation)). Any other name is an error. For a target in progress or not supported ([0.6.2](../000-meta/006-platform-targets.md#062-in-progress), [0.6.3](../000-meta/006-platform-targets.md#063-not-supported)) it says so, and what to use instead where there is something (``target `x86_64-apple-darwin` is not supported (Intel Macs); use `aarch64-apple-darwin` ``, ``target `aarch64-apple-ios` (iOS devices) is in progress: it is not tested on real hardware yet and comes in a later release; for now build for the simulator with `--target aarch64-apple-ios-sim` ``); for another name it names the closest built-in target when there is one (``unknown target `wasm32-wasi`; did you mean `wasm32-wasip2`?``).
- `-L <dir>` adds a library search directory for `#[link]` ([5.9.9](../005-modules/509-extern-block.md#599-linklibname-naming-the-library)), searched before `LIBRARY_PATH`. May be repeated.
- `-l [static=|dylib=|framework=]<name>` links the library `<name>` (searched like `#[link]`), or with `framework=` the Apple framework `<name>` as `#[link(name = "<name>", kind = "framework")]` does; `-F <dir>` adds a framework search directory, searched before the SDK's. `--native-runtime <file>` names a shared library to ship beside the executable ([9.2.14](902-aot-pipeline.md#9214-linking-for-executables)). All may be repeated; `nabi` passes them for native packages and system libraries ([9.10.4b](910-nabi-frontend.md#9104b-native-packages)). A framework on a target other than an Apple one is a link error with help to put the `extern` block under `#[cfg(target_os = "macos")]`.
- `--native-object <file>` links an object or archive into the executable or shared library as it is (repeatable); `nabi` passes the objects of a package's C sources ([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)).
- `-o <path>` / `--output=<path>` selects the output location.
- `--keep` keeps the build's temporary directory (objects and other intermediate files) instead of removing it, and names it on standard error (`nabic: kept temporaries in <dir>`).
- `--cfg <flag>` / `--cfg <key>=<value>` sets a cfg flag for [conditional compilation](../006-meta-system/603-cfg.md). May be repeated.
- `--emit=<kind>[,<kind>..]` writes intermediate output (`source`, `hir`, `mir`, `obj`, `asm`, `listing`, `expansions`, `c-header`) to `target/<profile>/build/<name>.<ext>` ([9.9.1](909-debug-tools.md#991-intermediate-output---emit)). Several kinds may be given at once separated by commas; in that case `-o` cannot be used. With `--emit`, no executable or archive is produced.
- `-g` adds debug information to a `release` profile build: DWARF, or for Windows targets CodeView and a PDB ([9.9.6](909-debug-tools.md#996-debugger-integration-gdb--lldb)). It does not change the generated code: an optimized build with `-g` has the same machine code as without it, and its debug information describes where the optimized code keeps each variable ([9.2.10](902-aot-pipeline.md#9210-optimization-levels)). A `debug` profile build always has it.
- `--remap-path-prefix <from>=<to>` writes the paths of source files under the directory `<from>` as under `<to>` in what the build produces: panic locations, debug information and `.nabihir` files. Diagnostics keep the real paths. A relative path is matched as an absolute one from the working directory, a prefix matches whole path components, the rest is joined to `<to>` with `/` (an empty `<to>` leaves it relative), and when several rules match, the last one given wins. May be repeated. The standard library's sources are always written under `/nabi/library` (`/nabi/library/core/src/...`), so a program names the same paths on every machine and install. A `release` profile build of a package also writes the package's own files relative to its directory (`src/main.nabi`) and a source dependency's as under its name (`json/src/lib.nabi`), so building the same package in two places gives the same bytes; `--remap-path-prefix` rules come after these and win.
- `-C <option>` may also be written `--codegen <option>`.
- `-C overflow-checks[=yes|no]` turns overflow checks (panic) for fixed-width integer arithmetic on or off. The default is on in every profile; a `nabi.nabion` profile can turn them off ([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)), and `-C overflow-checks` overrides it. An omitted value means `yes`. Operations on constant operands that cannot overflow skip the check.
- `-C icf[=yes|no]` turns identical code folding at link on or off ([9.2.15](902-aot-pipeline.md#9215-output)). The default is on in the `release` profile and off in the `debug` profile. An omitted value means `yes`.
- `-C target-cpu=<cpu>|native` names a CPU instead of the triple's baseline ([0.6.4](../000-meta/006-platform-targets.md#064-simd)); `-C target-feature=+f1,-f2,..` turns individual features (`+avx2`, ...) on or off, each written `+feature` or `-feature`, with the features they need (`+avx2` turns on `avx`) or that need them (`-avx` turns off `avx2`). The code generator uses `avx2`, `popcnt`, `lzcnt` and `bmi` on `x86_64`, `lse`, `rcpc` and `dotprod` on `aarch64`, `vector` and `vector-enhancements-1` on `s390x`, `v` on `riscv64` and `zbb` on `riscv64` and `riscv32`, and turns vector loops off for `-neon` on `armv7`, `-vsx` on `powerpc64le` and `-simd128` on `wasm32` ([8.7.8](../008-abi/807-code-generation.md#878-loop-vectorization)); a build or run that generates code with an option that changes none of that warns so. An unknown CPU name is an error listing the known ones. `native` is accepted only when the target is the build machine's platform, and adds the features this machine has.
- `-C subsystem=console|windows` selects the Windows subsystem: `console` (default, `mainCRTStartup`) or `windows` (GUI, `WinMainCRTStartup`; [9.5.9](905-symbol-mangling.md#959-symbol-of-the-main-function)).
- Any other `-C` option is an invalid option (E0132, below): ``unknown codegen option `<option>` ``, with the closest option when there is one and a help listing them.

`nabic build` reports the package's lint diagnostics as `nabic check` does ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)): warnings are printed and the build goes on; a `deny` lint fails it. If nothing has changed since the previous build, `nabic build` prints `Fresh`, then that build's warnings again, and skip the build ([9.2.16](902-aot-pipeline.md#9216-incremental-build)).

### `nabic emit`: intermediate output

```
nabic emit [path] [--emit=<kind>[,<kind>..]]
```

Equivalent to `nabic build --emit=..`. If `--emit` is omitted, it is `mir`.

### `nabic test`: tests

`nabic test` collects test functions and builds the test harness; `nabi test` ([9.10](910-nabi-frontend.md)) runs it.

```
nabic test [path] [filter]
```

It collects functions marked `#[test]` and builds a test executable (the harness). `#[test]` implies `#[cfg(test)]` ([6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-test-function-marker)), so test functions are not part of a normal build. Tests are collected both from `src/` (under `#[cfg(test)]`) and from the `tests/` directory ([0.3.5](../000-meta/003-source-units.md#035-test-and-benchmark-directories)). A test build uses an entry point that selects tests instead of the package's entry point.

`nabic` does not run the harness. `--emit-harness <file>` writes the built executables and the tests in them (the manifest `nabi test` runs) to `<file>`; its first line is `version 2`, the manifest format. The runner refuses a newer version: ``the test harness is from a newer nabic (format version N); run the nabi of the same toolchain, e.g. `nabi +<version> test` ``. Without `--emit-harness` the built executables are named on standard error. Running the harness (selecting tests, the process isolation, parallelism and reporting below) is the `nabi test` runner ([9.10](910-nabi-frontend.md)).

If there is a single argument and it is not an existing path, it is taken as `filter`. `filter` matches the test name including its module path (`math::adds`).

- It is a small regular expression using `^` `$` `.` `*` `+` `?` and `\` escapes. The search is unanchored, so without `^` / `$` it may match any part of the name.
- Without these characters it is equivalent to a plain substring search.

**Each test runs in its own process.** If a test panics (exit code 101) or ends with an uncaught error (exit code 1), only that test fails and the rest keep running. Termination by signal is also a failure, and so is a test that runs past the time limit (`--timeout`, 600 seconds by default): it is killed and reported as timed out. A `#[should_panic]` test passes only on a panic; an `#[ignore]` test is reported as ignored and not run ([6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-test-function-marker)).

#### Doc tests

`nabic test` on a package with a library also builds its doc tests: the fenced code blocks of the library's `///` and `//!` doc comments, and the ```` ```nabi ```` blocks of `README.md` at the package root. In doc comments a fence with no language is Nabi too (as `nabi doc` shows it); a block in any other language (```` ```text ````, ```` ```nabion ````) is not a test. Neither is a block in a module the build's `#[cfg]` leaves out, or on an item it leaves out (`#[cfg(feature = "table")]` without that feature).

Each block is code that uses the library, as a program depending on it would: the library is its dependency under the package's name (`stats::mean(x)`), and each block starts as if with `use stats::*;`, so the library's public items need no path (`mean(x)`). The block's leading `use` items stay at module level; the rest is the body of a function that may throw (`try` works without a `catch`), and items in it are local items. A block that declares `fn main` has it called. Words after `nabi` in the fence, separated by `,` or spaces, change what passing means:

| Fence | The block |
|-------|-----------|
| ```` ```nabi ```` | is built and run; it passes when it returns |
| ```` ```nabi,norun ```` | is built, never run (code that reads files, waits or needs a network) |
| ```` ```nabi,should_panic ```` | passes only when it panics |
| ```` ```nabi,compile_fail ```` | passes only when it does not build; `compile_fail,E0040` also needs that error code |
| ```` ```nabi,ignore ```` | is neither built nor run, and reported as ignored |

Another word is warned about, naming the words there are. A line starting with `# ` (or a lone `#`) is part of the test but not shown by `nabi doc`, for setup the reader does not need:

````nabi
/// ```nabi
/// # let data = [1.0, 2.0, 3.0];
/// assert(stats::mean(data) == 2.0);
/// ```
````

A doc test's name is `doc <file>:<line>`, the file relative to the package and the line of its opening fence, followed by the documented item when the block is on one: `doc src/stats.nabi:12 (mean)`, `doc README.md:20`. Filters, `--list` and the records see these names like any other. All the blocks of a package build into one executable, after the library's own tests; each `compile_fail` block is checked on its own when the harness is built. A block's code keeps its lines, so an error in it names the line of the documented file. A block that does not build fails as its own case, its errors shown with the failure, and the other blocks build and run without it.

`nabic test --doc` builds only the doc tests, `--no-doc` none of them; `nabi test` passes both on.

Options are those of `nabic build` and `--emit-harness`. How many tests run at once, whether passing tests show their output, and the output form are options of the runner, `nabi test` ([9.10.1](910-nabi-frontend.md#9101-subcommands-and-global-options)).

The runner prints to stdout in the following form.

```
running 3 tests
test math::adds ... ok
test math::subs ... FAILED
test util::trims ... ok

failures:

---- math::subs ----
<stdout / stderr of that test>
(panicked (exit 101))

failures:
    math::subs

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 filtered out; finished in 0.12s
```

An ignored test shows as `test <name> ... ignored` (with `, <reason>` when it has one). If any test fails, `nabi test` exits with `1`.

In a package with both root files, the library's tests run first, then the executable's (built against the library without its tests), then the doc tests; a failing run ends the command. `bench` works the same way.

### `nabic bench`: benchmarks

```
nabic bench [path] [filter]
```

Collects functions marked `#[bench]` and builds them in an optimized build (`-O2` unless `-O` is given); an explicit `-O0` or `-O1` builds the `debug` profile, with debug information, as for `nabic test`. `nabi bench` ([9.10](910-nabi-frontend.md)) runs the harness and measures them ([9.9.11](909-debug-tools.md#9911-benchmarks)). Like `#[test]`, `#[bench]` implies `#[cfg(test)]`, and benches are collected both from `src/` and from the `benches/` directory ([0.3.5](../000-meta/003-source-units.md#035-test-and-benchmark-directories)). The `path`, `filter` and `--emit-harness` rules are the same as `nabic test`.

A benchmark takes a `Bench` handle and times a closure, so setup stays out of the measurement:

```nabi
#[bench]
fn parse_1kb(b: bench::Bench) {
    let input = sample(1024);                   // setup: run once, not timed
    b.set_bytes(1024);                          // optional: report throughput
    b.iter(|| parse(hint::black_box(input)));   // the timed region
}
```

`b.iter(f)` performs the whole measurement (calibrate, warm up, sample); its result is black-boxed. `hint::black_box` ([`core::hint`](../../refs-standard-libraries/200-core/README.md#benchmarks--optimizer-hints)) is an identity the optimizer cannot see through. Apply it to the inputs and result of the timed region so the work is neither hoisted out nor folded away. `b.set_bytes(n)` / `b.set_items(n)` add a throughput column. `Bench`, `black_box` and the statistics are library API; the method is defined in [9.9.11](909-debug-tools.md#9911-benchmarks).

The calls per sample, the number of samples and the output form are options of the runner, `nabi bench` ([9.10.1](910-nabi-frontend.md#9101-subcommands-and-global-options)).

Benchmarks run one at a time. Each prints one aligned line in human units, to a terminal or not:

```
  json::parse_1kb     1.84 us +/- 0.03 us   543 MiB/s   [2 outliers, 0 gc]
```

`nabi bench --format=nabibench` prints a stable machine form for regression tools instead, one line per benchmark: `#nabibench name=<name> median_ns=<f> mad_ns=<f> samples=<n> outliers=<n> gc=<n> bytes=<n> items=<n>`. This line is frozen as it is: no key is added, removed or reordered, so a reader written for it keeps working. New fields go only in the JSON records of `--format=json`, which also keep a benchmark's own output apart from them (with `nabibench`, it shares stdout with these lines).

A benchmark that never calls `b.iter` measured nothing. It is not a failure: its line is `` <name>  warning: measured nothing: time the code with `b.iter(|| ..)` ``, and the summary counts it as `<n> measured nothing`, not as run.

### `nabic check`: diagnostics

```
nabic check [path | -] [--check-library]
```

Runs the front end through type checking and MIR building, plus lints, then code generation without the native build (no object files, no link), and reports diagnostics. Options are the same as `nabic build`. `--check-library` lowers and checks every body of a library regardless of reachability, and checks every generic body against its bounds; `nabic build --check-library` also compiles every one, optimized at the build's level, into the executable (a build otherwise compiles only the library code the program reaches, [9.3.6](903-package-archive.md#936-standard-library)) ([2.10.9](../002-types/210-generics.md#2109-body-checking)); its lints then cover the standard library's own code too, with `missing_docs` at `deny` ([6.2.9](../006-meta-system/602-built-in-attributes.md)). A package with both root files checks its library, then, if the library has no error, its executable with the library compiled in; each warning is reported once.

## 9.1.3. `--print` options

### `nabic --print cfg`: active cfg

```
nabic --print cfg [--target=<target>] [-O<n>] [--cfg ..]
```

Prints the cfg of the build the options describe ([6.3](../006-meta-system/603-cfg.md)), one per line: a flag as `flag`, a key as `key="value"` (as rustc does). The keys include `target_env`, and `debug_assertions` is set at `-O0`.

```
debug_assertions
target_arch="aarch64"
target_endian="little"
target_family="unix"
target_os="macos"
target_pointer_width="64"
```

### `nabic --print target-list` / `target-spec`: targets

```
nabic --print target-list
nabic --print target-spec [--target=<target>]
```

`target-list` prints the supported target names, one per line; the targets in progress ([0.6.2](../000-meta/006-platform-targets.md#062-in-progress)) are left out. `target-spec` prints the target's specification as NABION ([0.6.5](../000-meta/006-platform-targets.md#065-target-specifications)); without `--target`, the host's.

### `nabic --print lldbinit`: lldb formatters

Prints one line, `command script import '<path>'`, that loads the lldb formatters of this toolchain ([9.9.6](909-debug-tools.md#formatters)): `<sysroot>/lib/nabi/lldb/nabi.py`. Loading them also maps the standard library's `/nabi/library` paths to the toolchain's library sources (`target.source-map`).

`--print gdbinit` prints `source <path>`, the gdb command that loads the gdb pretty-printers, `<sysroot>/lib/nabi/gdb/nabi.py` (they map the library paths with `set substitute-path`). `--print natvis` prints the path of the natvis visualizers, `<sysroot>/lib/nabi/natvis/nabi.natvis`.

## 9.1.4. Auxiliary subcommands

### `nabic version`: version

```
nabic version
nabic --version            # = -V
nabic -V -v                # adds host line
```

Prints one line `nabic <version>` to stdout, followed by ` (<commit> <date>)` when `nabic` was built from a git checkout. With `-v`, a `host: <triple>` line is appended.

### `nabic help`: help

```
nabic help [subcommand]
nabic <subcommand> --help
```

Without a subcommand, prints the full reference: every subcommand, option and environment variable. Given a subcommand, prints that subcommand's own help: its usage forms, a short description, one or more examples and the options it takes. Neither prints text belonging to another subcommand.

## 9.1.5. Global options

The global options are listed below. They may appear before or after the subcommand.

| Option                           | Meaning                                                     |
|----------------------------------|-------------------------------------------------------------|
| `--verbose` / `-v`               | Show per-phase timing ([9.9.5](909-debug-tools.md#995-logging--progress-display)) |
| `-vv`                            | More detailed logging: `Reading <file>` and `Specialized <path>(sig)` lines ([9.9.5](909-debug-tools.md#995-logging--progress-display)) |
| `--quiet` / `-q`                 | Do not print progress lines (diagnostics are still printed) |
| `--color=auto\|always\|never`    | Colour on stdout and stderr (9.1.8)                         |
| `--no-color`                     | Equivalent to `--color=never`                               |
| `--cwd=<path>`                   | Change working directory before running                     |
| `--print cfg\|target-list\|target-spec\|lldbinit\|gdbinit\|natvis` | Print the active cfg, the supported target names, a target's specification, or what loads Nabi's debugger formatters (lldb, gdb, natvis; 9.1.3) |
| `--message-format=human\|json`   | Diagnostic format. `json` writes one line per diagnostic to stdout (9.1.8) |
| `--explain <code>`               | Print the page of an error code (`E0040`): what it means and what to write instead, then exit ([9.9.4](909-debug-tools.md#error-codes)) |
| `-A` / `-W` / `-D <lint \| group>` | Lint level (`--allow` / `--warn` / `--deny`). Applies to every command that compiles the package (`build`, `test`, `bench`, `emit`, `check`, and `nabi`'s `run`, `lint`, `repl`, `jupyter`, `lsp`). May be repeated; the later one wins |

`nabic help` lists the `--print` forms with the build options, since what they print depends on `--target` and `--cfg`; they need no subcommand.

The lint level is determined in the order: the lint's default level < `-A` / `-W` / `-D` < `#[allow(..)]` / `#[warn(..)]` / `#[deny(..)]` in code (innermost wins). `nabi` prepends the manifest's `lints` as these flags ([9.10.1](910-nabi-frontend.md#9101-subcommands-and-global-options)).

Colour is decided for stdout and stderr separately, in this order: `--color=never` turns it off and `--color=always` on; otherwise (`--color=auto`, the default) a non-empty `NO_COLOR` turns it off, a `CLICOLOR_FORCE` that is non-empty and not `0` turns it on, `TERM=dumb` turns it off, and else a stream is coloured when it is a terminal. On Windows a console that does not take escape sequences gets plain text.

Examples:

```
nabic --verbose build
nabic test -q
nabic --cwd=examples/hello check
```

## 9.1.6. Exit codes

Exit codes are defined as follows.

| Code  | Meaning                                              |
|-------|------------------------------------------------------|
| `0`   | Success                                              |
| `1`   | User code error (compile error / test failure, etc.) |
| `2`   | Usage error (invalid option, unknown subcommand, etc.) |
| `70`  | nabic internal error (panic)                          |

A usage error prints `nabic: <message>` and then the line ``run `nabic help <subcommand>` for usage`` (``run `nabic help` for usage`` when no known subcommand was given). The message says what to write instead: an unknown subcommand or option names the closest one when there is one (``unknown option `--relase`; did you mean `--release`?``), and an unknown subcommand that names a file (it contains `/` or ends in `.nabi`) suggests `nabic build <file>`. When `__NABI_CLI` is `nabi` (9.1.7), both lines say `nabi` instead of `nabic`.

A known option given a value it does not take is reported as the diagnostic `E0132` ([9.9.4](909-debug-tools.md#error-codes)), still with exit code `2`: an unknown `-C` option or `--emit` kind, an option without its value, an unknown `-C target-cpu`, a `--print`, `--color` or `--message-format` value it does not take, a malformed `-l`, `--extern` or `--extern-feature`, a code `--explain` does not have, or `-o` with several `--emit` kinds. Its `= help:` line says what to write (``error[E0132]: unknown emit kind `mirr`; did you mean `mir`?``), and the `run .. for usage` line follows. It is printed as `--color` and `--message-format` say wherever they stand on the command line: under `--message-format=json` it is a JSON line on standard output, without that line.

The sysroot does not come from `NABI_HOME`: it is relative to the executable. `nabic` reads `NABI_HOME` only to show a fetched dependency's path under `$NABI_HOME/cache/`. If nabic itself panics (internal compiler error), it prints the panic message, then `nabic: internal compiler error` and three `note:` lines: that this is a bug in nabi and where to report it with the code that triggers it, the version and host (`nabic <version> (<commit> <date>) on <triple>`), and the command line; a fourth, `note: in pass: <pass> > <pass>`, names the compiler passes it was in (as `NABI_TIME_PASSES` names them), when it was in one. It exits with `70` (`EX_SOFTWARE`), apart from a program's panic, which exits with `101`. `nabi` reports its own internal errors the same way (`nabi: internal error`, exit code `70`).

If standard output or standard error is a pipe whose reader has gone, `nabic` stops without a message: on Unix it is terminated by `SIGPIPE` (a shell reports `141`); on Windows it exits with `1`. This is not an internal error.

## 9.1.7. Environment variables

The environment variables that affect `nabic` are:

| Variable          | Meaning                                                                                  |
|-------------------|------------------------------------------------------------------------------------------|
| `NABI_CC`          | Optional C compiler driver that performs the final link instead of `nabic`'s linker ([9.2.14](902-aot-pipeline.md#9214-linking-for-executables)) |
| `NABI_VERBOSE`     | Equivalent to `--verbose`                                                                |
| `NABI_TIME_PASSES` | Per-pass wall time on stderr when `1` ([9.9.5](909-debug-tools.md#nabi_time_passes)) |
| `__NABI_ENV`, `__NABI_ENV_LIBS` | Set by `nabi` when it runs a script from an environment (internal): the environment's name and the comma-separated libraries installed there. A script that uses one of them without declaring it gets a hint naming it |
| `__NABI_CLI`       | Set to `nabi` by `nabi` when it runs `nabic` (internal): usage errors (9.1.6), `--help` and the `nabic:` prefix of error and warning lines name `nabi`, not `nabic` |
| `NABI_TARGET_DIR`  | Location of `target/` (default: inside the package)                                      |
| `NABI_INCREMENTAL` | Where the front end keeps the package's parses and lowered bodies (default `target/<profile>/incremental/`; `0`: nowhere); the library's collected items are kept in `lib/nabi-cache/library/` ([9.2.16](902-aot-pipeline.md#9216-incremental-build)) |
| `NABI_INCREMENTAL_VERIFY` | `1`: lower every body again and fail on a difference ([9.2.16](902-aot-pipeline.md#9216-incremental-build)) |
| `NABI_FN_CACHE` | `0`: neither read nor write the toolchain's function cache, the standard library code builds keep ([9.2.16](902-aot-pipeline.md#9216-incremental-build)) |
| `NABI_CACHE_MAX_SIZE` | The cap of the toolchain's cache: bytes, or a number with `K`, `M` or `G` (default `5G`); `off`: no cap ([9.2.16](902-aot-pipeline.md#9216-incremental-build)) |
| `NABI_HOME`        | The display of fetched dependencies (9.1.6; default `~/.nabi`)                           |
| `NABI_RUNNER`      | `<program> [args]`: what `nabi run` / `test` / `bench` run another target's executables under ([9.10.8](910-nabi-frontend.md#executables-of-other-targets)) |
| `LIBRARY_PATH`    | Native library search directories, after `-L`                                            |
| `NO_COLOR`        | Non-empty: `--color=auto` does not use colour (9.1.5)                                    |
| `CLICOLOR_FORCE`  | Non-empty and not `0`: `--color=auto` uses colour even when the stream is not a terminal, unless `NO_COLOR` is set (9.1.5) |
| `COLORTERM`       | `truecolor` or `24bit`: colour uses 24-bit values (9.1.8)                                |
| `TERM`            | `dumb`: `--color=auto` does not use colour; containing `256color`: colour uses the 256-colour palette (9.1.8) |

An unset or empty variable has its default. A value outside those listed prints one warning line on stderr naming the variable, the value and the accepted values, and the variable has its default.

`nabi` hands `nabic` the internal variables `__NABI_CLI`, `__NABI_ENV` and `__NABI_ENV_LIBS`; `nabic` reads no other spelling of them. `nabi run`, `nabi test` and `nabi bench` remove every variable whose name starts with `__NABI` from the environment of the program they start (on Windows, whatever the case of the name); the program sees the rest of the environment unchanged.

## 9.1.8. Output format

### Diagnostic messages

Diagnostic messages are printed in the following format.

```
error[E0027]: cannot find value `count` in this scope
  --> src/main.nabi:5:13
   |
 5 |     let x = count + 1;
   |             ^^^^^
   |
   = help: did you mean `counter`?
```

Diagnostics use the file + line + column units defined in [Source files](../001-lexical/101-source-files.md).

The header is `<severity>[<code>]: <message>` and `--> path:line:col` gives the primary location. The primary span is underlined with `^`, secondary spans with `-`; a secondary span in another file is introduced by `::: path:line:col`. Trailing `= note:` lines add context and `= help:` lines suggest a fix. Messages cite no sections of this specification; the JSON format lists them ([9.9.4](909-debug-tools.md#994-diagnostics--error-messages)).

### Build progress

Progress is shown on stderr with right-aligned verbs. Nothing is printed with `-q`.

```
   Compiling json v0.1.0
   Compiling foo (path/to/foo)
    Building target/debug/foo
    Finished in 1.23s
     Running `target/debug/foo`
```

| Verb        | When                                          |
|-------------|-----------------------------------------------|
| `Compiling` | Front end starts (package name and directory); `Checking` for `check` and `lint`, `Documenting` for `doc`. A dependency `nabi` fetched (under `$NABI_HOME/cache/`) shows `<name> v<version>` instead |
| `Building`  | Executable / archive produced (`build`)       |
| `Built`     | A lone file's executable copied next to it (`build`): `Built hello (use -o to put it elsewhere)`. A test or bench harness built without `--emit-harness` (`test`, `bench`), followed by a note naming `nabi test` or `nabi bench` to run it |
| `Wrote`     | One `--emit` output written                   |
| `Reused`    | With `-v`, after the front end: how many files' parses and bodies' lowerings came from the incremental cache ([9.2.16](902-aot-pipeline.md#9216-incremental-build)) |
| `Units`     | With `-v`, during code generation: how many code units were reused (from the incremental cache or the function cache) and how many compiled |
| `Finished`  | Build done (time since the command started) |
| `Running`   | Program run (`run`): the executable relative to the current directory; a lone file by its file name (`Running hello.nabi`), with `-v` the cached executable |
| `Created`   | Package created (`new`), then the command to run next under it ([9.10.2](910-nabi-frontend.md#9102-nabi-new--nabi-init-package-creation)) |
| `Removed`   | `target/` deleted (`clean`)                   |
| `Fresh`     | Up to date; that build's warnings follow      |
| `Checked`   | `check` done (error and warning counts)       |
| `Generated` | Documentation written (`doc`)                 |
| `Formatted` | Files rewritten (`fmt`): their count, at the end ([`nabi fmt`](910-nabi-frontend.md#nabi-fmt-formatter)) |

A build (`build`, `run`, `test`, `bench`, `doc`) that fails in the front end ends, after the diagnostics, with `error: could not build <name> (<n> error(s), <m> warning(s))`, counted as `check` counts them. `-q` and `--message-format=json` leave it out.

`nabi` prints its own progress the same way: `Added`, `Installed`, `Updated`, `Downloading` and the like ([9.10](910-nabi-frontend.md)).

### Colour

When colour is on (9.1.5), the output uses this palette. Its depth follows the terminal: 24-bit values when `COLORTERM` is `truecolor` or `24bit`, else the 256-colour palette when `TERM` contains `256color`, else the 16 standard colours.

| Part                                  | Colour                   | 24-bit      | 256   | 16             |
|---------------------------------------|--------------------------|-------------|-------|----------------|
| Status verbs (`Compiling`), list mark `*` | sky, bold            | `#38bdf8`   | `81`  | bright cyan    |
| `error`                               | red, bold                | `#ff5f5f`   | `203` | bright red     |
| `warning`                             | amber, bold              | `#e0b057`   | `179` | yellow         |
| `note`                                | sky                      | `#38bdf8`   | `81`  | bright cyan    |
| `help`                                | light pink               | `#f9a8d4`   | `218` | bright magenta |
| Diagnostic gutter (`-->`, `\|`, line numbers) | sky, bold         | `#38bdf8`   | `81`  | bright cyan    |

Colour never carries meaning alone: the labels (`error:`, `warning:`, `note:`, `help:`) and the status verbs are printed with or without it. A list that marks one entry (`nabi toolchain list`, `nabi env list`) puts `*` before it and `(default)` or `(active)` after it. Piped output, `--message-format=json` and `-q` are not affected.

### JSON output (for tool integration)

For tool integration, output can also be produced as JSON.

```
nabic build --message-format=json
```

In this mode each diagnostic becomes one line of JSON object on stdout, easy for IDEs and CI tools to parse. The shape of the object is defined in [9.9.4](909-debug-tools.md#994-diagnostics--error-messages).

## 9.1.9. Standard input

Writing `-` in an argument position, as in `nabic check -`, means standard input, for use with pipes or scripts. Source from standard input is checked as the `main.nabi` of an anonymous executable package.

## 9.1.10. Compatibility with external tools

`nabic` is a low-level compiler; dependency resolution is the job of the front-end `nabi` ([9.10](910-nabi-frontend.md)). `nabi` invokes `nabic`, passing the roots of resolved dependency packages via `--extern` ([9.2.3](902-aot-pipeline.md#923-dependency-resolution)). Other build systems may integrate by invoking `nabic` directly in the same way. Changes that break CLI compatibility are managed under SemVer rules.
