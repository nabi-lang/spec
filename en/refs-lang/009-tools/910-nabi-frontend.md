# 9.10. `nabi` - package / build front-end and tools

`nabi` is a cargo-style front-end. It owns the manifest `nabi.nabion` and lockfile `nabi.lock`, dependencies, environments, the toolchain pin and script dependencies, and delegates compilation to the compiler `nabic` ([9.1](901-nabic-cli.md)). The tools (run, lint, REPL, Jupyter, LSP, formatter, docs, ...) are part of `nabi` itself (9.10.8). The runtime is `nabir` (`libnabir.a`).

A package without dependencies builds by file conventions alone, without a manifest ([0.3.4](../000-meta/003-source-units.md#034-the-manifest-nabinabion-is-optional)). In that case `nabi build` is equivalent to `nabic build`.

## 9.10.1. Subcommands and global options

| Subcommand                                                        | Action                                  |
|-------------------------------------------------------------------|-----------------------------------------|
| `new`, `init`                                                     | Create a package (9.10.2)               |
| `add`, `remove`, `update`, `deps`                                 | Dependencies (9.10.4)                   |
| `search`, `publish`, `yank`, `login`, `logout`                    | Registry packages (9.10.4a)             |
| `env`                                                             | Environments (9.10.5)                   |
| `component`                                                       | Optional parts: the official packages, for builds without a network (9.10.4a) |
| `toolchain`, `cache`                                              | Toolchain and its build cache (9.10.6)  |
| `install`, `uninstall`                                            | Libraries for the REPL / Jupyter, and executables (9.10.5a) |
| `build`, `check`, `emit`                                          | Delegated to `nabic` with dependencies attached |
| `test`, `bench`                                                   | `nabic` builds the harness with dependencies attached; `nabi` runs it ([9.1](901-nabic-cli.md)) |
| `run`, `dev`, `lint`, `fix`, `repl [path]`, `jupyter`, `lsp`, `doc` | Tools, with dependencies attached (9.10.8) |
| `fmt`, `clean`, `bindgen`, `inspect`, `demangle`, `completions <shell>` | Tools, without dependencies (9.10.8) |
| `cc`                                                              | The C compiler `nabicc`, with the arguments unchanged ([9.13](913-c-compiler.md)) |
| `version`, `help`                                                 | Version; help (below)                   |
| (none)                                                            | Same as `nabi repl`: starts the REPL     |

A delegated subcommand runs the `nabic` subcommand of the same name; a tool runs in `nabi`'s own process, with the compiler linked in (9.10.8). For both, `nabi` consumes only its own global options (`--env` / `--offline` / `--locked` / `--frozen` / `--cwd` / `--color`, anywhere before `--`), passes the remaining arguments to the compiler unchanged, and adds the following. For `test` and `bench`, `nabi` has `nabic` build the harness (`--emit-harness`) rather than run it, then runs the harness itself ([9.1](901-nabic-cli.md)); the runner is not part of the compiler, and its options (below) are `nabi`'s.

- `--extern name=<source directory>` for each dependency package. They are prepared according to `nabi.lock`, with dependencies before their dependents (the order `nabi deps` prints). Prebuilding is done by `nabic` ([9.2.3](902-aot-pipeline.md#prebuilt-dependency-packages)).
- `--extern-dep <package>=<name>` for each dependency of each package (`pkg` the package being built), so that each package sees only its own dependencies ([5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)).
- The package's own features as `--cfg feature=<f>`, and each dependency's as `--extern-feature <name>=<f>` (9.10.4c). The feature options `--features <f,..>` (repeatable; `<dep>/<f>` too), `--all-features` and `--no-default-features` of build / run / test / bench / check / lint / fix / doc / emit / repl / lsp / jupyter / deps are consumed by `nabi` (before `--`).
- The manifest's `lints` as `-A` / `-W` / `-D` (before the arguments). The manifest is that of the project the path selects: for a `.nabi` file, the nearest directory at or above the file containing `src/main.nabi`, `src/lib.nabi` or `nabi.nabion`. A file in the `src/` of a package selects that package ([9.1.1](901-nabic-cli.md#911-general-form)). A lone file elsewhere in a directory that has `nabi.nabion` (a script beside a package's `src/`, or a directory with only `nabi.nabion`) is compiled as a single-file package with that manifest's `lints` (and `lints_from`) and dependencies.
- Outside a package (environments, scripts), `--target-dir` set to the `target/` of that environment or script.

`nabi cc` is neither a delegated subcommand nor a tool: it runs the toolchain's `nabicc` with every argument after `cc` unchanged and adds nothing to them ([9.13.1](913-c-compiler.md#9131-nabicc-and-nabi-cc)).

`nabi repl <path>` resolves dependencies and lints from the project of that path, as `run` does, and uses the manifest name (`--name`).

The runner of `nabi test` takes these options (before `--`); `nabic` is not given them:

- `--threads=N` sets the number of test processes run concurrently. The default is `NABI_TEST_THREADS`, or the CPU count if that is unset; a value of `NABI_TEST_THREADS` that is not a positive number warns (`nabi: warning: ignoring NABI_TEST_THREADS=0 (expected a positive number)`) and is ignored.
- `--show-output` also shows the stdout / stderr of passing tests. Output of failing tests is always shown.
- `--format=human|json` selects the output form (default `human`). `json` writes a JSON record per test and a summary on stdout and the human text on stderr ([9.9.12](909-debug-tools.md#9912-ci--test-integration)).
- `--exact` makes the filter and `--skip` match whole test names: `nabi test --exact math::adds` runs that test and not `math::adds_wide`.
- `--skip <pat>` leaves out the tests `<pat>` matches, with the filter's rules (repeatable).
- `--ignored` runs only the `#[ignore]` tests; `--include-ignored` runs them along with the rest.
- `--timeout=SECS` kills a test that runs longer and fails it as timed out. The default is `NABI_TEST_TIMEOUT`, or 600 if that is unset; `0` is no limit. A value of `NABI_TEST_TIMEOUT` that is not whole seconds warns and is ignored.
- `--list` prints the names of the tests the filter keeps, one per line, and runs none (exit code `0`); an `#[ignore]` test is followed by ` (ignored)`. With `--format=json` each name is a record instead: `{"schema":1,"type":"listed","kind":"test","name":"adds"}`, with `"ignored":true` for an ignored one.

`--doc` and `--no-doc` go to `nabic`: only the doc tests, or none of them ([9.1.2](901-nabic-cli.md#doc-tests)).

The runner of `nabi bench` takes these:

- `--iters=N` fixes the number of calls per sample instead of calibrating.
- `--samples=N` is the number of samples measured (default 10).
- `--format=human|json|nabibench` selects the output form (default `human`): `json` as for tests, `nabibench` the stable line of [9.1.2](901-nabic-cli.md#nabic-bench-benchmarks).
- `--list` prints the names of the benchmarks the filter keeps, as for tests, and runs none.

Both run the executables of another `--target` through its runner ([Executables of other targets](#executables-of-other-targets)).

A bad value is a usage error that names what to write: `` `--threads` takes a positive number, not `0` ``, `` `--timeout` takes whole seconds (0: no limit), not `1.5`: `--timeout=120` ``, `` `--format` takes human or json, not `xml` ``. What the runner prints is described with `nabic test` and `nabic bench` ([9.1.2](901-nabic-cli.md#nabic-test-tests)).

Options, output and exit codes follow those of `nabic` ([9.1](901-nabic-cli.md)). `nabi` treats a closed standard output or standard error the same way as `nabic` ([9.1.6](901-nabic-cli.md#916-exit-codes)). `nabils` is a standalone executable equivalent to `nabi lsp`.

`nabi help` (also `nabi --help`) prints a short page: the subcommands in groups, one line each, and the global options. `nabi help <subcommand>` (also `nabi <subcommand> --help`) prints that subcommand's own help: its usage forms, a description, examples and its options. For a delegated subcommand it is `nabic`'s help written as `nabi`, with `nabi`'s feature options and, for `test` and `bench`, the runner's options added.

An unknown subcommand or option is a usage error (exit code 2). The message names the closest subcommand or option when there is one, including common other names (`compile` for `build`, `rm` for `remove`), and is followed by ``run `nabi help <subcommand>` for usage``. A package or native package that the index lacks names the closest one there and the `nabi search` to run. `nabi` sets `__NABI_CLI=nabi` for the `nabic` it runs, so `nabic`'s usage errors, help and the prefix of its error and warning lines name `nabi` as well ([9.1.7](901-nabic-cli.md#917-environment-variables)).

Running just `nabi [options]` with no subcommand starts the REPL, like `nabi repl`.

```
nabi                 # = nabi repl
nabi --env sci       # = nabi repl --env sci
```

`nabi [options] <file> [args..]` runs a script: it is `nabi run -q <file> -- args..`. Every argument after the file is passed to the program unchanged, and the exit code is the program's. The word is a file when it contains `/` (also `\` on Windows) or has the extension `.nabi`; no subcommand does, so a subcommand is never taken for a file. Together with a shebang line ([1.2.3](../001-lexical/102-lexical-structure.md#shebang-line)), `./script.nabi args` runs the script.

The path of a delegated command or a tool is its first argument that is neither an option nor an option's value, so options may come before it (`nabi run --release t.nabi`). A path that does not exist is an error, `` `<path>` does not exist ``, before anything else (not "not inside a package"); for `test` and `bench`, whose first word may be a filter, only a word ending in `.nabi` is taken for a path.

The global options of `nabi` are:

| Option           | Meaning                                                           |
|------------------|-------------------------------------------------------------------|
| `--env <name>`   | Work in that named environment (the REPL, Jupyter, `nabi install`); an error on a command that works on a package ([9.10.5](#9105-environments)) |
| `--offline`      | Do not use the network. git dependencies must be in the cache     |
| `--locked`       | If `nabi.lock` would need to change, do not change it and fail (for CI) |
| `--frozen`       | `--locked` and `--offline` together                               |
| `--cwd <dir>`    | Change directory before running                                   |
| `-q`, `--quiet`  | Reduce output                                                     |
| `-v`, `--verbose` | More output (also passed to `nabic`)                              |
| `--color <when>`, `--no-color` | `auto`, `always` or `never`: colour, decided as for `nabic` ([9.1.5](901-nabic-cli.md#915-global-options)); passed to the `nabic` it runs as `--color=<when>` |

Global options may also follow the subcommand: `nabi new app -q` is `nabi -q new app`.

Environment variables are `NABI_HOME` (default `~/.nabi`), `NABI_ENV` (9.10.5), `NABI_TOOLCHAIN` (9.10.6), `NABI_REGISTRY_TOKEN` (the default registry's API token, 9.10.4a), and `NABI_DIST` (the release server, 9.10.6). `NO_COLOR`, `CLICOLOR_FORCE`, `COLORTERM` and `TERM` set colour as for `nabic` ([9.1.7](901-nabic-cli.md#917-environment-variables)).

## 9.10.2. `nabi new` / `nabi init`: package creation

```
nabi new [--lib] [--bin] <name>
nabi init [--lib] [--bin] [dir]
nabi init --script <file.nabi>
```

`nabi new` creates the directory `<name>` and fills in default files. `--bin` (the default) creates `src/main.nabi`, which prints a greeting; `--lib` creates `src/lib.nabi` with an example function and one `#[test]`; both together create both, `src/main.nabi` then calling the library's function through `use <name>::add;` ([0.3.1](../000-meta/003-source-units.md#031-executables-and-libraries)). It is an error if `<name>` already exists.

`nabi init` turns an existing directory (the current directory if omitted) into a package. It is an error if that directory already contains `src/main.nabi` or `src/lib.nabi`.

`nabi init --script <file.nabi>` turns a script ([9.10.7](#9107-inline-manifest-for-single-file-scripts)) into a package in the script's directory, named after the file (without `.nabi`). `nabi.nabion` gets the merged `deps` (and `native`) of the script's blocks, paths rewritten relative to it; the script becomes `src/main.nabi` and every file its `mod`s load moves under `src/` at the same relative path, each without its block. It is an error if the script is inside a package, a moved file or `nabi.nabion` already exists, or a submodule file is outside the script's directory.

Both commands also write `nabi.nabion` in canonical form ([NABION 3.8.4](../../refs-standard-libraries/300-std/308-nabion.md#384-canonical-form)), with `nabi_version` the version of the `nabi` that writes it (`nabi_version: "0.1.3"`, its pre-release left out) and pinned to that `nabi`'s series (`toolchain: "0.1"` from any 0.1.x, 9.10.6) so the package stays on that series when a newer one becomes the default, and a `.gitignore` containing `target/`, and run `git init` if not inside a git repository. `init` uses the directory name, adjusted to the naming rules (9.10.3), as the package name: a Windows device name gets the prefix `pkg-` (`nul` → `pkg-nul`), and the name is cut to 64 characters.

After the `Created` line, both print the command to run next, under the message: `cd <dir> && nabi run`, or `nabi test` for a package with only a library; without `cd` when the package is the current directory. On Windows, where Windows PowerShell has no `&&`, `cd <dir>` and the command are two lines. `-q` prints neither.

```
     Created executable package (hello)
             cd hello && nabi run
```

On Windows:

```
     Created executable package (hello)
             cd hello
             nabi run
```

```
hello/
├── .gitignore        <- target/
├── nabi.nabion
└── src/
    └── main.nabi
```

## 9.10.3. `nabi.nabion` / `nabi.lock`

`nabi.nabion` at the package root is the manifest, and `nabi.lock` in the same place is the lockfile. Both are in [NABION](../../refs-standard-libraries/300-std/308-nabion.md) format. The names are spelled exactly this way (all lowercase). Files differing only in case (`Nabi.nabion` etc.) are ignored with a warning and not read.

The manifest is a single NABION value, a `Package { .. }` struct (the root name `Package` may be omitted). It is only read, never compiled or executed.

```nabion
Package {
    name: "geom",                    // string, optional
    version: "0.3.1",                // string, optional
    nabi_version: "0.4.1",           // string, optional (oldest nabi it builds with; `nabi publish` needs it)
    toolchain: "0.4",                // string, optional (toolchain pin, 9.10.6)
    description: "2D geometry",      // string, optional
    license: "MIT OR Apache-2.0",    // string, optional (SPDX expression)
    repository: "https://example.com/geom.git",   // string, optional
    authors: ["Ann Lee <ann@example.com>"],       // list of strings, optional
    homepage: "https://example.com/geom",         // string, optional
    documentation: "https://example.com/geom/doc",   // string, optional
    readme: "README.md",             // string, optional (path relative to nabi.nabion)
    keywords: ["geometry", "math"],  // list of strings, optional (at most 5)
    categories: ["mathematics"],     // list of strings, optional (at most 5)
    license_file: "LICENSE",         // string, optional (path relative to nabi.nabion)
    include: ["src/**", "README.md"],   // list of globs, optional (what the package holds)
    exclude: ["bench/**"],           // list of globs, optional (left out of it)
    links: "z",                      // string, optional (the native library it links)
    default_run: "geom",             // string, optional (the executable `nabi run` runs)
    deployment_target: "13.0",       // string, optional (the oldest macOS the program runs on;
                                     // { macos: "13.0", ios: "16.0" } for iOS too)
    metadata: { docs: { all_features: true } },   // table, optional (for other tools)
    publish: true,                   // bool, optional (false: `nabi publish` refuses)
    deps: {                          // dependency name (string) -> Dep
        "util": Dep::path("../util"),
        "parser": Dep::git("https://example.com/parser.git", tag: "v1.2"),
        "http": "0.9",                                   // from the index (= Dep::version("0.9"))
        "json": Dep::version("^1.4", optional: true),    // only with a feature (9.10.4c)
        "csv": Dep::version("^0.3", registry: "https://example.com/index"),
        "j5": Dep::version("^0.2", package: "json5"),    // the package `json5`, `j5` in code
        "nix": Dep::version("^0.4", target: "cfg(target_family = \"unix\")"),
    },
    dev_deps: {                      // same form as deps. test / bench only
        "fixtures": Dep::path("tests/fixtures"),
    },
    features: { "default": ["json"], "json": ["dep:json"], "tls": ["http/tls"], "full": ["json", "tls"] },
    native: {                        // C libraries (9.10.4b)
        "zlib": "^1.3",                                  // a prebuilt native package
        "png": Dep::native(url: "vendor/png.nabion", sha256: "<64 hex>"),
        "sqlite3": Dep::system(brew: "sqlite", apt: "libsqlite3-dev"),   // installed on the machine
        "Foundation": Dep::framework(),                  // an Apple framework
    },
    lib: { kind: "cdylib" },         // optional (also a shared library for C, 9.3.3; "plugin": 9.12)
    c: { sources: ["c/shim.c"], include: ["c"], define: { "N": "1" }, std: "c17" },   // optional
    lints_from: "..",                // string, optional (starts from ../nabi.nabion's lints)
    lints: { "dead_code": "allow", "nonstandard_style": "deny" },
    profile: { "release": { overflow_checks: false } },   // per build profile
}
```

- Values are NABION literals (strings, `{ "k": v }` dicts, the empty dict `{}`) and `Dep::path("..")` / `Dep::git("url", rev | tag | branch: "..")` / `Dep::version("req", registry: "url")`. `Dep::git` takes at most one of `rev` / `tag` / `branch`, and `registry` of `Dep::version` is optional. A dependency value that is a single string is shorthand for `Dep::version`.
- `Dep::path("..", version: "req")` adds a version requirement to a path dependency. Builds use the path, and the package there must have a version the requirement accepts (`` dependency `linalg` (path ..) is version 0.2.0, which its `version: "^0.1"` does not accept; .. ``); `nabi publish` writes the requirement in its place (9.10.4a). This is how packages developed side by side are published.
- `Dep::path` / `Dep::git` / `Dep::version` also take `optional: true`, `features: ["<f>", ..]` and `default_features: false` (9.10.4c); an optional dev dependency is an error.
- `package: "<name>"` renames a dependency: `<name>` is the package, and the key is what this package's code calls it (`"j": Dep::version("^1", package: "json5")` is `json5`, used as `use j::..`). Each dependent picks its own names, and only its own code sees them, so two packages may call one package differently; it is still one package, built once, and its types are the same under every name. A dependent depending on one package under two keys is an error (`` the root package depends on package `json5` twice, as `j` and `json5`; depend on it once ``).
- `target: "<target>"` limits a dependency to builds for a matching target: a target name (`x86_64-unknown-linux-gnu`), or `cfg(<predicate>)` over the target keys of [6.3.2](../006-meta-system/603-cfg.md#632-predicate-kinds) (`target_os`, `target_arch`, `target_family`, `target_env`, `target_endian`, `target_pointer_width`) with `all(..)`, `any(..)`, `not(..)`. Anything else is an error when the manifest is read. Resolution and the lock cover the dependency for every target; a build for another target leaves it out, with the features it would enable.
- `plugin: true` on a dependency builds it as a plugin beside the program, without linking it ([9.12.2](912-plugins.md#9122-building)); `bundle: true` on a plugin's dependency links it into the plugin instead of taking it from the program. `bundle` in a package that is not a plugin is an error (`` `bundle: true` on `noise` belongs in a plugin's dependencies: give this package `lib: { kind: "plugin" }`, or remove `bundle: true` ``), and so is a dependency with both.
- `Dep::registry` is for `nabi.lock` only; using it in `nabi.nabion` is an error.
- Package names (keys of `deps`, `name`) are lowercase ASCII `[a-z0-9_-]+` beginning with a letter (beginning with a digit, `-` or `_` is an error). The names `std`, `core` and `bare` are the standard library's tiers, which ship with the toolchain: a package, native package or registry name among them is an error (`` `std` is the standard library, which ships with the toolchain; name the package something else, e.g. `std_util` ``), and so is `nabi new` / `nabi init` with it; `nabi add std` says there is nothing to add. A `deps` key among them is an error whose help is `` std ships with the toolchain; remove this line ``, unless it has `package: "<name>"`: then it is that package under the tier's name ([5.7.4](../005-modules/507-paths.md#574-std-core-bare-the-standard-library-tiers)). `package: "std"` is the same error. Code must be able to name the package too, so its code name (below) may not be a keyword (1.5: `fn`, `match`, `self`, ..) or a path start (5.7.1: `pkg`, `sup`); `use pkg::add;` in an executable always means its own root (`` `pkg` already names a package's own root (`pkg::`), so code can't name a package called that; name the package something else, e.g. `pkg_util` ``). Keys are strings (`"nabi-syntax": Dep::version("^0.1")`).
- A package name must also be one every supported OS can store as a file or directory: not a Windows device name (`con`, `prn`, `aux`, `nul`, `com1`-`com9`, `lpt1`-`lpt9`, in any case) and at most 64 characters (`` `nul` is a device name on Windows, so no file can be called that; name the package something else, e.g. `my_nul` ``). `nabi new` / `nabi init --script` / `nabi add` / `nabi publish` refuse such a name. In a `nabi.nabion` that is read (its `name` and the keys of `deps`, `dev_deps`, `native`) it is a warning with `file:line:col`, and the build goes on.
- `version` is a version (`major.minor.patch`, 9.10.4a). One that is not is a warning when the manifest is read (`` `banana` is not a version (major.minor.patch): it needs three numbers; `nabi publish` needs one, e.g. `version: "0.1.0"` ``) and an error at `nabi publish`.
- `nabi_version` is the oldest toolchain the package builds with, a full version: `x.y.z`, no more and no less (`` `nabi_version: "0.1"` is not a toolchain version; write all three parts, e.g. `nabi_version: "0.1.3"` `` when the manifest is read). It admits that toolchain and the later ones of its series ([9.10.6 Compatibility](#compatibility)): `"0.1.3"` admits 0.1.3 up to the last 0.1.x, `"1.4.2"` admits 1.4.2 up to the last 1.x. `nabi publish` needs it (9.10.4a), and resolution picks versions by it. It is not `toolchain`, which pins the toolchain this project builds with (9.10.6).
- In code, the name with `-` replaced by `_` is the top-level path referring to that package (`nabi-syntax` → `use nabi_syntax::..`, `--extern nabi_syntax=..`).
- In the registry and within one dependency graph, names are compared treating `-` and `_` as equal and ignoring case. So `nabi-syntax` and `nabi_syntax` cannot coexist.
- A dependency must have a `src/lib.nabi`; that library is what its dependents get, whether or not the package also has `src/main.nabi`. A dependency with only `src/main.nabi` is an error that says it is a program, for `nabi install`. Transitive dependencies are the `deps` of the dependency package (`dev_deps` are not transitive).
- `native` maps native package names (their own namespace: a package and a native package may share a name) to `Dep::native("req"[, registry: "url"])`, a requirement string, or `Dep::native(url: "<url or path relative to the manifest>", sha256: "<64 hex digits>")` (9.10.4b). It also maps names to `Dep::system(..)`, a library installed on the machine, and `Dep::framework()`, an Apple framework, which are found at build time (9.10.4b). `Dep::native` in `deps` / `dev_deps` is an error. Native dependencies of every package in the graph are linked into executables (those reached only through `dev_deps`, only for test / bench).
- `lib: { kind: "nabi" | "cdylib" | "staticlib" | "plugin" }` (the record may be named `Lib`) says what the library builds. `"nabi"`, the default, is the library other Nabi packages use; with `"cdylib"`, `nabi build` passes `--crate-type=cdylib` and `--lib-version` with the package's `version` ([9.1.2](901-nabic-cli.md#nabic-build-aot-build)), so the build also makes the shared library and its C header ([9.3.3](903-package-archive.md#933-static-and-shared-libraries)), unless the command line gives `--crate-type`. With `"staticlib"` it passes `--crate-type=staticlib`: the build also makes one static library C and Swift link alone, with its header and module map (9.3.3); `xcframework: true` beside it also passes `--xcframework`, and on another kind it is an error. With `"plugin"` it passes `--crate-type=plugin`: the package is a shared library a Nabi program loads ([9.12](912-plugins.md)); a `"plugin"` package with `src/main.nabi` is an error, and so is `nabi run` on it. Another kind is an error that lists the four.
- `c: { sources, include, define, std, flags }` (the record may be named `C`) are C files built into the package by nabicc, Nabi's own C compiler, never a system one: `sources` (paths relative to the manifest, at least one), `include` (header directories, then those of the package's `Dep::system` libraries), `define` (`{ "NAME": "value" }`), `std` (`c89`, `c99`, `c11`, `c17`, `c23` or a `gnu` one) and `flags`, which take only `-O*`, `-g`, `-W*`, `-f*` (not `-flto`) and `-m*`: anything else is an error that lists them. The objects go to `target/<profile>/c/<name>/` and to the link (`--native-object`). Only a source whose inputs (it and the headers it read) or options changed compiles again ([9.13.8](913-c-compiler.md#9138-c-sources-of-a-package)).
- `lints` sets lint levels (lint or group name → `"allow"` | `"warn"` | `"deny"`, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)). They are treated as if prepended to the CLI flags of every compiling command `nabi` runs (delegated to `nabic`, or a tool), so CLI `-A/-W/-D` and attributes in code override them.
- `lints_from` names a directory (relative to this manifest's directory, or absolute) whose `nabi.nabion` supplies starting lint levels. That manifest's `lints`, after its own `lints_from` chain, are applied first, and this manifest's `lints` follow, so an entry here overrides the same lint or group from the chain (all entries are passed as `-A`/`-W`/`-D` in this order; later flags win). It applies only to the package being built or checked (and to an environment's manifest); dependencies' manifests are never consulted for lints. It is an error if the directory does not exist, contains no `nabi.nabion`, or if the chain returns to a manifest already in it (a cycle). `nabi env import` rewrites a relative `lints_from` as an absolute path. A script's `//! ```nabion` block may not use it.
- `profile` maps a build profile (`"debug"` or `"release"`, [9.2.15](902-aot-pipeline.md#9215-output)) to its settings, `{ overflow_checks: bool }` (the record may be named `Profile`). `overflow_checks: false` turns integer overflow checks off in that profile, as `-C overflow-checks=no` does ([9.1.2](901-nabic-cli.md#nabic-build-aot-build)); `-C overflow-checks` on the command line overrides it. Another profile name or setting is an error. Only the package being built (or the environment) is read; dependencies' profiles are not, and a script's `//! ```nabion` block may not use it.
- Package metadata: `authors` are `"Name <email>"` strings, and `nabi new` / `nabi init` do not fill them. `keywords` and `categories` hold at most 5 entries each. An empty entry of `authors`, `keywords`, `categories`, `include` or `exclude` is an error. `readme` and `license_file` are paths inside the package, relative to the manifest's directory (an absolute path, or one whose `..` leaves the package, is an error); without `readme`, `README.md` is the readme when present. `metadata` is a table (`{ .. }`) `nabi` does not read; it goes to the registry as written. `nabi publish` checks that the readme and `license_file` exist, that `default_run` names one of the package's executables, and refuses a package with `publish: false` (9.10.4a).
- `include` / `exclude` are globs over the paths of the files at the published commit (`*` and `?` within a name, `**` across names; a pattern without `/` matches a name at any depth, and one that matches a directory matches what is in it). The package holds a file when `include` is empty or matches it and `exclude` does not; `nabi.nabion` is always in. They set the packed size `nabi publish` sends (9.10.4a); the source is still the whole commit.
- `links` names the native library the package links. Two packages of one build with the same `links` are an error naming both.
- `default_run` names the executable `nabi run` runs; a package has at most one executable (`src/main.nabi`, named after the package), so it is that name.
- `deployment_target` is the oldest macOS the package's programs run on, as numbers (`"13.0"`; another form is an error that shows it); `11.0` when omitted. The record form `{ macos: "13.0", ios: "16.0" }` gives the oldest iOS too (`13.4` when omitted; an older one is an error), each field optional. `nabi` passes them to `nabic` as `MACOSX_DEPLOYMENT_TARGET` and `IPHONEOS_DEPLOYMENT_TARGET`, unless one is set already, which then wins. Objective-C APIs newer than it are called only under `objc::available` ([std 1.10.13](../../refs-standard-libraries/100-bare/110-objc.md#11013-availability-and-the-deployment-target)).
- A field `nabi` does not know is a warning with `file:line:col` (`` unknown field `licence` is ignored; did you mean `license`? ``, with a suggestion when one is close), and the manifest is read without it; `nabi publish` sends it on in `meta` as written (9.10.4a), so a newer registry can use it.
- Errors: unknown `Dep` kinds / named arguments, duplicates, `features` that break the rules of 9.10.4c, package names violating the rules, syntax that is not NABION, anything other than a single `Package { .. }`, trailing tokens. Diagnostics show the location with `file:line:col` and a caret.

| Dependency source | Form                                                         | Pinned in `nabi.lock` |
|-------------------|--------------------------------------------------------------|----------------------|
| path        | `Dep::path(".."[, version: "req"])` (path relative to the manifest) | No (always current content) |
| git         | `Dep::git("url")`, optionally one of `rev:` / `tag:` / `branch:` | Commit and tree hash |
| registry    | `Dep::version("req"[, registry: "url"])` or the string `"req"` ([9.10.4a](#9104a-registry-packages)) | Version, commit, tree hash (with the index, never a mirror) |
| native      | `Dep::native("req")` / `Dep::native(url:, sha256:)` ([9.10.4b](#9104b-native-packages)) | Version and entry SHA-256 (index); none (the hash pins it) |
| system      | `Dep::system(..)` / `Dep::framework()` ([9.10.4b](#9104b-native-packages)) | No (found on each build) |

Nabi has no build scripts: no code from a package runs at build time. C libraries come in as prebuilt native packages, as libraries installed on the machine, or as C sources nabicc compiles (`c`, [9.13.8](913-c-compiler.md#9138-c-sources-of-a-package)), all declared in the manifest (9.10.4b).

Within one build, one name is one package. If the same name is required from different sources (different paths, or different urls / commits), it is an error showing both requirers. So is a code name that two dependents give to different packages (`` `hello` is package `other` for the root package and package `greet` for `mid` ``). One package may have several code names (`package`); `nabi` passes each as `--extern` with the same path.

### Workspaces

Packages developed together can share one lock and one build directory. The root of a **workspace** holds a `nabi.nabion` whose value is `Workspace { .. }` instead of a package:

```nabion
Workspace {
    members: ["app", "libs/*"],   // directories under the root; `dir/*`: each package in `dir`
}
```

- `members` lists directories relative to the root; `dir/*` stands for every direct subdirectory of `dir` that has a `nabi.nabion`. An absolute path, a `..` step or another `*` is an error (`` `../x` is not a member: write a directory under the workspace root, or `dir/*` for each package in `dir` ``), and so is a member directory without `nabi.nabion` (`` member `gone` has no nabi.nabion (..); create the package with `nabi new gone`, or remove it from `members` ``). Each member needs a `name`; two members with one name are an error. `Workspace` takes no other field.
- A package is a member when a workspace root above its directory lists it; a package under the root that is not listed stays on its own.
- The members share `nabi.lock` at the root: their dependencies (dev ones too) are resolved together, so every member builds against the same versions, and a member's own `nabi.lock` is not read. They also share `target/` at the root (`NABI_TARGET_DIR` and `--target-dir` still win). Members depend on each other with `Dep::path`.
- `build`, `check`, `test`, `bench`, `lint`, `fmt`, `doc`, `clean` and `run` take `-p <name>` / `--package <name>` (repeatable: those members) and `--workspace` (every member). At the root, where there is no package, these commands run on every member; `run` needs exactly one (`` `nabi run` runs one package; name it: `nabi run -p <member>` (members: ..) ``). Each member runs in its own directory, in `members` order, with `Member <name> (<dir>)` before it when there are several; after a failure the others still run, the command exits with the first failure's code and names the members that failed. A name that is no member is an error listing the members; `-p` outside a workspace is an error.
- Other commands at the root (`emit`, `add`, ..) are an error saying it is a workspace root and naming `-p <member>` or a member's directory; `nabi update` and `nabi deps` there work on the shared lock. The language server resolves each file's package, members included, from the shared lock.

### `nabi.lock` format

`nabi.lock` is NABION (canonical form) written deterministically by `nabi`. After a header comment it holds one `Locked` per git or registry package, in name order. Path dependencies are not pinned and do not appear. Its field names follow the manifest's.

```nabion
// nabi.lock: generated by `nabi`; do not edit. Commit it with your sources.
Lock {
    version: 1,
    packages: [
        Locked {
            name: "parser",
            source: Dep::git("https://example.com/parser.git", tag: "v1.2"),
            commit: "4f0c2a9e1b7d3c58a6e09f21d4b8c7a3e5f60123",
            tree: "9a1e7c3b5d2f4a6081c9e3b7d5f2a4c6e8b0d1f3",
        },
        Locked {
            name: "json",
            version: "1.4.2",
            source: Dep::registry("https://github.com/nabi-lang/packages-index"),
            commit: "0b6d..",
            tree: "77e1..",
        },
    ],
}
```

| Field     | Meaning                                                              |
|-----------|----------------------------------------------------------------------|
| `version` | In `Lock`, the lock format version (`1`); in `Locked`, registry packages only: the resolved version |
| `name`    | The package's name (the `deps` key, or its `package` when renamed)   |
| `source`  | For git, the `Dep::git(..)` exactly as written in `nabi.nabion`; for registry packages, `Dep::registry("<index url>")`: the index, never the mirror it was fetched from (9.10.4a) |
| `commit`  | The resolved commit (40 hex digits)                                  |
| `tree`    | The git tree hash of that commit. Used to verify the cached checkout |

- The `source` of `Locked` is `Dep::git(..)` or `Dep::registry(..)`.
- A missing `packages` field is read as an empty list.
- Features are not recorded: the lock pins every package of the graph, optional ones included, so it does not depend on the features a build enables.
- Native packages from an index are pinned in an optional `natives: [LockedNative { name, version, source: Dep::registry("<index>"), sha256 }]`, sorted by name and written only when non-empty. `sha256` is the SHA-256 of the entry's canonical index line with `yanked: false`; a changed entry is a "run `nabi update`" error.
- `nabi` writes lock format 1.
- `version` is read before the other fields. A `version` above `1` is a lock a newer `nabi` wrote, and an error naming the toolchain to use instead of deleting it: ``nabi.lock was written by a newer nabi (lock version 2); run that nabi's toolchain here: set `toolchain` in nabi.nabion to its version, or run `nabi +<version> ..` ``.
- Any other `version` than `1` is an "unsupported lockfile version" error.
- Every other error raised while reading the lock appends "(delete nabi.lock and run `nabi update` to make it again)".

## 9.10.4. Dependencies: `nabi add` / `remove` / `update` / `deps`

```
nabi add <name> --path <dir> [--dev]
nabi add <name> --git <url> [--rev <rev> | --tag <tag> | --branch <branch>] [--dev]
nabi add <name>[@<req>] [--registry <url>] [--dev]      # from the index
nabi add <name> .. [--features <f,..>] [--no-default-features] [--optional]   # with any of the above
nabi remove <name>
nabi add .. <file.nabi>                    # into a script's //! nabion block
nabi remove [--native] <name> <file.nabi>
nabi update [name...]
nabi deps [--dev] [--features <f,..>] [--all-features] [--no-default-features] [--format json]
```

- `add` / `remove` edit the package's `nabi.nabion`, preserving comments and layout. `--dev` adds to `dev_deps`. Given a script file as the last argument, `add` / `remove` edit that file's `//! nabion` block instead (9.10.7): a missing block is created after the shebang line and the leading `//!` lines (written in canonical form; an existing block keeps its layout and comments), a block left empty is removed, a path is written relative to the script, and `--dev` / `--optional` are errors. A script inside a package is an error (it uses the package's `nabi.nabion`). Outside a package without a script, or with `--env`, they are an error naming `nabi add <name> <file.nabi>` and `nabi install` / `nabi uninstall` ([9.10.5a](#9105a-nabi-install--nabi-uninstall)). Adding a package without a requirement writes `^<newest version>`. `--features`, `--no-default-features` and `--optional` are written into the `Dep::..`; `--optional` also adds the feature `"<name>": ["dep:<name>"]` unless the package has a feature of that name.
- `add` to a package reports `Added <name> v<version>`, the `version` in the dependency's `nabi.nabion` (for a registry package not yet resolved, the version pinned), or `Added <name>` when it has none; `--dev` adds ` (dev_deps)`.
- Builds use the commits and versions in `nabi.lock`. Newly declared dependencies are added to the lock and removed ones are cleaned up, but only `update` moves something already pinned. Given names, `update` re-resolves only those packages; otherwise it re-resolves all git and registry dependencies.
- `deps` prints the resolved dependency packages as lines `<code name>=<path of its src/lib.nabi>` (the code name has `-` replaced by `_`), dependencies first (`--dev` includes dev dependencies). A package that dependents call by other names (`package`) gets one more such line per name. Each package's lines are followed by its `--extern-feature=<code name>=<f>` lines; `deps` takes the feature options and `--dev`. For each native package (dependents first, for the host) it also prints `# native <name> <version>` followed by its `nabic` arguments, one per line. With no dependencies it prints nothing.
- `deps --format json` prints one JSON object per line instead, in the same order, each with `schema` (now `1`) and `type`. A `package` record has `name`, `version` (a registry package's resolved version, else the `version` of the dependency's `nabi.nabion`, else `null`), `source` (`path+<dir>`, `git+<url>#<commit>` or `registry+<index url>`), `path` (its `src/lib.nabi`) and `features`. A `native` record has `name`, `version` and `args` (its `nabic` arguments). A last `summary` record counts `packages` and `natives`, so a package without dependencies prints only it. Records follow the `schema` rule of the JSON diagnostics ([9.9.4](909-debug-tools.md#994-diagnostics--error-messages)). `--format human` is the default; another value is a CLI error.
- `nabi.lock` is committed to version control.
- `update` prints a line per change in the lock, `Updated <name> v0.1.0 -> v0.1.1` (a git dependency by its short commit; `Locked` for one newly pinned, `Removed` for one dropped), or `note: all dependencies are up to date`; with no dependencies, `` note: no dependencies to update; add one: `nabi add <name>` ``. Outside a package without `--env`, it updates the environment used outside a package (9.10.5) when that has libraries (ending with `` Updated environment `<name>` `` when something changed); otherwise it is an error: `` not inside a package (..): make one with `nabi init`; for an environment's libraries: `nabi update --env <name>` ``.
- `nabi update` re-resolves `nabi.lock`; `nabi toolchain update` replaces the toolchain (9.10.6).

### Cache

Fetched git sources are kept in `$NABI_HOME/cache/` and shared by all projects.

| Path                                       | Contents                                                |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | Bare clone of the repository (one per url)              |
| `cache/registry/db/<repo>-<hash>/`         | Bare clone of a registry index (one per index)            |
| `cache/registry/src/<sha256>/`             | A registry package's files from the registry's archive with that SHA-256 (9.10.4a); never fetched again |
| `cache/git/checkouts/<commit>/`            | Checkout of a commit. Verified by tree hash before use, and re-extracted if different |
| `cache/scripts/<hash>/`                    | `nabi.lock` and `target/` of a script; `<hash>` covers the script's path and the path and block of every file it reaches ([9.10.7](#9107-inline-manifest-for-single-file-scripts)) |
| `cache/native/<sha256>/`                   | A native archive, unpacked after its SHA-256 matched; never fetched again (9.10.4b) |
| `cache/native/meta/<sha256>.nabion`         | A `Dep::native(url:)` file                              |

## 9.10.4a. Registry packages

A package published to a registry is a **registry package**. The default registry is `https://packages.nabi-lang.org`, whose index is the separate git repository `https://github.com/nabi-lang/packages-index`.

### Version requirements

Same as Cargo.

| Requirement              | Meaning                                         |
|--------------------------|-------------------------------------------------|
| `1.2.3`, `^1.2.3`        | `>=1.2.3, <2.0.0` (`^0.2.3` is `<0.3.0`, `^0.0.3` is `<0.0.4`) |
| `~1.2.3`                 | `>=1.2.3, <1.3.0`                               |
| `=1.2.3`                 | Exactly                                         |
| `>=` `>` `<` `<=`        | Comparison                                      |
| `*`, `1.*`, `1.2.x`      | Wildcard                                        |
| `>=1.2, <1.5`            | Comma means all must be satisfied               |

A pre-release version (`1.3.0-beta.1`) matches only when the requirement names a pre-release of the same major.minor.patch.

### Which index

A requirement without `registry:` (a bare requirement) always means the default registry's index, `https://github.com/nabi-lang/packages-index`. The `registry:` of `Dep::version` and `nabi add --registry` set the index for that one dependency; `nabi publish`, `yank`, `login` and `logout` take `--registry <url>` for another index.

`$NABI_HOME/sources.nabion` replaces where an index is fetched from (source replacement):

```nabion
Sources { replace: { "https://github.com/nabi-lang/packages-index": "https://mirror.example.com/packages-index" } }
```

- Each key is an index url (a trailing `/` ignored), its value the git url of a mirror that has the same content. `nabi` fetches the index from the mirror; everything else, the lock (9.10.3), tokens and messages, names the index. A missing file is no replacement; fields `nabi` does not know are skipped.

### Index format

The root of the index repository has a `config.nabion`, `Config { api: "<url>" }`: the base URL of the registry's API (`https://packages.nabi-lang.org` for the default registry). It must be `https://`, except `http://127.0.0.1` and `http://localhost`. A package is one file at the shard path of its published lowercase name (`-` kept as is).

| Name length | Path                     |
|-------------|--------------------------|
| 1           | `1/a`                    |
| 2           | `2/ab`                   |
| 3           | `3/a/abc`                |
| 4 or more   | `ab/cd/abcd..`           |

Each line of the file is one version, as single-line NABION in canonical form. Older versions come first. A line holds only what resolution needs, with the manifest's field names; the descriptive fields go to the registry API (`meta`, below).

```nabion
Package { name: "json", version: "1.4.2", deps: [{ name: "util", req: "^0.3", kind: "normal" }], source: Dep::git("https://example.com/json.git", rev: "0b6d.."), tree: "77e1..", yanked: false, nabi_version: "0.1.0" }
```

| Field       | Meaning |
|-------------|---------|
| `name`      | The package's name as published |
| `version`   | Its version |
| `deps`      | Dependencies: `{ name, req, kind, registry?, optional?, features?, default_features?, package?, target? }` |
| `features`  | Optional: the features (9.10.4c) |
| `source`    | `Dep::git("<url>", rev: "<commit>")` |
| `tree`      | The git tree hash of that commit |
| `archive`   | Optional, written by the registry: the SHA-256 (64 lowercase hex digits) of the source archive it keeps of that commit |
| `yanked`    | Whether it is yanked |
| `nabi_version` | The oldest toolchain that builds it, `x.y.z` (9.10.3) |
| `links`     | Optional: the native library it links (9.10.3) |
| `official`  | Optional, written by the registry: `true` for a package the Nabi project publishes (below) |

- The `kind` in `deps` is `"normal"`, `"dev"` or `"native"`; `registry` is optional. A renamed dependency has `name` the code name and `package` the package's name; a target-specific one has `target` as in the manifest, which the client evaluates for each build (9.10.3).
- A package with features has `features: { "<name>": ["<entry>", ..] }` after `deps`, checked as in 9.10.4c. A `normal` dependency may have `optional: true`, `features: [..]` and `default_features: false` (written only when not the default); `dev` / `native` ones may not be optional, and native ones have no features. `nabi publish` writes them from `nabi.nabion`; resolution reads them from the entry. After checkout the entry's `features` must equal those of the package's `nabi.nabion` at that commit (in any order); otherwise it is an error.
- `native/<shard path>` holds native packages, one `Native { .. }` per line ([9.10.4b](#9104b-native-packages)).
- `nabi_version` is the manifest's (9.10.3). `nabi publish` needs it: without it the error is `` nabi.nabion has no `nabi_version`, the oldest nabi this package builds with; the registry needs it `` with the help `` add `nabi_version: "0.1.3"` (this nabi), or the oldest one you have built it with ``, and one that does not admit the publishing `nabi` is an error naming the toolchains it admits. The registry refuses a line without it or with one that is not `x.y.z`. Readers honour it (Resolution, below); a line without it, in an index of one's own, suits any toolchain.
- The file of a standard library tier (`3/s/std`, `co/re/core`, `ba/re/bare`) holds one line `Library { name: "std", official: true }`: the registry's entry for a library that ships with the toolchain, owned by the Nabi project, with no versions and no archive. Readers that resolve skip it, and no version can be published under its name.
- This is the only line format: there is no format version per line.
- Readers skip fields they do not know in an index line (`Package`, its `deps` entries, `Native`, `Artifact`). A `Native { .. }` file of one's own (`Dep::native(url:)`, `nabi publish --native`) still rejects them. The registry checks lines strictly.
- The package is at the root of the `source` repository. `tree` is the git tree hash of that commit, verified after checkout.
- The registry keeps a gzipped tar of the commit (`git archive`, every path under `<name>-<version>/`) when the version is published, serves it at `<api>/api/v1/packages/<name>/<version>/download`, and writes its SHA-256 as `archive` after `tree`; `nabi publish` never sends `archive`. A value that is not 64 lowercase hex digits is read as absent.
- A registry package comes from, in order: the toolchain's own copy (Official packages, below), for the default index; a git checkout of its commit already in the cache; its archive (`archive` given), from `cache/registry/src/<sha256>/` or downloaded from the `api` of the index's `config.nabion` (`Downloading <name> <version>`); its git `source`. A download whose SHA-256 is not `archive` is an error (`` package `num` 1.2.0: <url> has SHA-256 .., but the index says ..; the archive or the index is damaged .. ``) and the build stops. No archive to be had (none kept, unreachable, no `api`, or `--offline` with none cached) is a note, and the git source is fetched instead. With an archive, neither git nor the package's repository is needed; the lock pins the same commit and tree either way.

### Resolution

- Every requirement on a package counts: the version chosen meets all of them. The packages are picked in the order they are first required, each trying the version pinned in the lock first (kept as long as it meets every requirement, even if yanked since), then the non-yanked versions from the highest down. When a choice leaves some package with no version that meets its requirements, the search backs up and tries the next version of the packages picked before it, so `>=0.1` from one dependent and `^0.1` from another resolve to the highest `0.1.x`.
- A new choice (and `nabi add` without a requirement) skips entries whose `nabi_version` does not admit the running `nabi` ([9.10.6 Compatibility](#compatibility)), or that `nabi` cannot read. When that hides the highest match, a note names that version, the toolchains it needs (`` package `num` 2.0.0 needs nabi 0.2.1 or a later 0.2.x (this is nabi 0.1.5) ``), how to use it (`nabi +0.2.1 ..`, or pin that toolchain) and the version chosen instead; when nothing else matches, the error carries the note as help. A version pinned in the lock is kept regardless.
- For the default index, the versions the toolchain carries (Official packages, below) are tried before the others, after the one pinned in the lock: `^0.2` picks a carried `0.2.0` over `0.2.5` in the index. Their lines need no index, so when every package of the search is carried and no index is cached, none is fetched; when the carried versions leave a requirement unmet, the index is fetched and the search runs again. `nabi update` tries the versions from the highest down, carried or not.
- In one dependency graph, one name is one version. Transitive dependencies follow the `deps` of the index entry. Path and git packages are fixed: their registry requirements count like the root's.
- When no set of versions works, the error names a package whose requirements no version meets, each requirement with who requires it (`` `^2` (required by `a` 1.4.0) ``), the versions the index has, how each requiring package came into the build (`` `a` 1.4.0: the root package requires `a` `^1.0` ``), and whether other versions of them were tried. Its help says what to write: a version of the package that meets the other requirements (`` change the root package's to it, `"b": "^2.3.0"` ``), or a version of a requiring package that gets along with them (`` change the root package's requirement on it, `"c": "^0.4.0"` ``), else to relax one requirement. A single requirement that no version meets is the plain "no version of package .. matches .." error. The search gives up after 100000 choices and says to pin the main dependencies.
- The lock is checked against the index. If the `rev` of a pinned version has changed, it is a "run `nabi update`" error.
- `--offline` uses the cached index and respects `--locked`. A version whose source is in neither cache (no git checkout, no archive) is an error (`` package `json` 0.2.5 is not in the cache, and --offline is set: run the command once without --offline; `nabi component add packages` (or `update`) keeps the official packages for offline use ``); the second half only for an official package.

### Official packages

A package the Nabi project publishes is marked `official: true` in the index (the registry writes it). Code may use one without adding it first: when `build`, `run`, `check`, `test`, `bench`, `repl` or `jupyter` on a package or a script finds a name used as a path root (`use json::parse;`, `json::parse(s)`) that is not a dependency of that package or script (one another package of the build depends on is not) and that the code does not declare itself (a module, an item, an import, a binding), and the default index says the name is official (by its latest version's line, the highest not yanked; names are looked up with `_` and `-` alike, so `nabi_syntax::..` is `nabi-syntax`, added under its index name), `nabi`:

- picks the version the toolchain carries (below), else the highest non-yanked version this toolchain builds (its `nabi_version` admits this toolchain), and writes `"<name>": "^<major>.<minor>"` (`^<version>` below 0.1) into the package's `nabi.nabion` (a workspace member's own; a script's `//! nabion` block, made if missing), resolves and updates `nabi.lock`, prints `Added <name> ^x.y (official)` and goes on with the command;
- for a package the toolchain carries, reads no index; else reads the cached index, and fetches it only when there is none (a failed fetch, or `--offline` with none cached, adds nothing);
- with `--locked` or `--frozen` (named as such in the error), or `CI` set to a non-empty value, never edits a manifest: `` `json` is an official package the code uses, but it is not in nabi.nabion, and --locked never changes it; add it: `nabi add json` ``;
- offline, with the package in neither cache: `` .. it is not in the cache and --offline is set: run `nabi add json` online, or keep the official packages for offline use with `nabi component add packages` ``.

The roots `std`, `core`, `bare`, `nabi`, `pkg`, `sup`, `crate`, `super`, `self` and capitalised names are never packages. A name that is not official keeps the compiler's error (`` `geom` is not a dependency of this package: add it to nabi.nabion (`nabi add geom`) ``). The REPL and the Jupyter kernel do the same for each cell (`nabi add` in a package's session, `nabi install --env` in an environment's). The language server never edits a manifest: it offers the `nabi add <name>` quick fix (9.9.13).

### `nabi component`

```
nabi component list
nabi component add packages [--bundle <dir>]
nabi component update [packages]
nabi component remove packages
```

`packages` is the registry archive (9.10.4a Index format) of the highest non-yanked version, that this toolchain builds (its `nabi_version` admits this toolchain), of every official package in the default index.

- A release toolchain carries it (Distribution, 9.10.6): `lib/nabi/packages/` beside `bin/` holds the index lines of those versions in `index` (one per line, as in an index file) and each archive unpacked in `src/<sha256>/`. Nothing writes there but `remove`; resolution reads it first for the default index (Resolution, above), and its lines name the same commits and trees as the index, so the lock is the one a download makes.
- `add` fetches the index and downloads each archive into `cache/registry/src/<sha256>/`, checked against the line's `archive` (`Downloading json 0.2.0` .., then `Added component packages (<n> official packages)`), and records them in `$NABI_HOME/components.nabion`; with the component added, or carried by the toolchain, `add` again is a note naming `update`. `--bundle <dir>` writes them as a toolchain's `lib/nabi/packages` to `<dir>` instead (replacing it), as the release build does (`Bundled <n> official packages into <dir>`).
- `update` fetches the index again and downloads the newest versions this toolchain builds into the cache; what the toolchain carries stays as it is.
- `remove` deletes what the toolchain carries, the recorded archives and the record; with neither, it is an error.
- `list` shows each component and whether it is added: `added: <n> package(s) with the toolchain`, with `, <m> downloaded` when `add` / `update` downloaded some too.

A version without `archive` is a note and is fetched from git when used. `--offline` is an error for `add` and `update`; another name is an error listing the components.

The component only fills a cache: manifests and the lock still pick the versions. A lock that pins a version neither the toolchain nor the cache holds downloads it when online, and offline is the error above.

### `nabi search` / `publish` / `yank` / `login` / `logout`

```
nabi search [--native] <query>
nabi login [--registry <url>] [<token>]
nabi logout [--registry <url> | --all]
nabi publish [--dry-run] [--registry <url> | --index-dir <checkout>]
nabi yank [--native] <name>@<version> [--undo] [--message <why>] [--registry <url> | --index-dir <checkout>]
```

- `search` searches the index by name and prints one line per match: `<name> <newest version> - <description>`. The description comes from the registry API (`GET <api>/api/v1/packages/<name>`, the newest version's `meta.description`); it is left out when the API has none or does not answer, and under `--offline`. Index lookups accept either spelling, `-` or `_`.
- `publish` publishes (as `cargo publish` does); `publish --dry-run` is a dry run. A package with `publish: false` is refused, with an error naming the line to remove. The readme (9.10.3) must exist. It requires a clean work tree, the package at the repository root, and a `src/lib.nabi`, a `src/main.nabi` or both. A package with only `src/main.nabi` is a program: `nabi install` builds it, and it cannot be a dependency. A normal path dependency is published as its `version:` requirement and rejected without one (`` dependency `linalg` is a path: .. add the version it needs, published from that path: `Dep::path("../linalg", version: "^0.1")` ``); normal git dependencies are rejected, and path / git dev dependencies without `version:` are dropped. It checks for conflicts with the same version and with names equal under `-` / `_` and case folding. A name already owned under a different spelling is rejected. A dry run without `--index-dir` only prints the entry; with `--index-dir` it is also checked against that checkout. The entry's source is the url of git remote `origin` as configured (before `insteadOf` rewrites); it must be `https://`, except `http://127.0.0.1` and `http://localhost`.
- Without `--dry-run`, `publish` adds the entry: with `--index-dir`, as a commit "Publish <name> <version>" in that checkout (not pushed); without it, through the registry API of the index `--registry <url>` names, else of the default registry ("Which index"), which adds it to the index. `--index-dir` and `--registry` cannot be combined. The client never writes to the index repository itself.
- `yank` sets (or with `--undo` clears) `yanked` for that version: with `--index-dir`, as a commit in that checkout; otherwise through the registry API of the index `--registry <url>` names, else of the default registry.
- `--commit` (`publish`), `--index` (`publish` and `yank`) and `--vers` (`yank`) are errors that name the form to write: a bare `publish`, `--index-dir <dir>`, and `yank <name>@<version>`.
- `yank --message <why>` says why (at most 512 bytes); the registry keeps it and shows it. It goes with a yank only: with `--undo`, or with `--index-dir` (a checkout has no place for it), it is an error.
- The registry API is at `api` of the index's `config.nabion`. Requests carry `Authorization: Bearer <token>`, and bodies are single-line NABION (`application/nabion; charset=utf-8`):
  - `GET <api>/api/v1/me` → `Me { login: "..", publish: <bool> }`
  - `POST <api>/api/v1/publish`, body `Publish { entry: <index line>, meta: { .. } }` (the line is `Package { .. }` / `Native { .. }`) → `Published { name, version, file, warnings: ["..", ..] }`; `nabi` prints each warning.
  - `POST <api>/api/v1/yank`, body `Yank { name, version, undo: <bool>, native: <bool>, message: ".." }` (`message` written only with one) → `Yanked { name, version, yanked }`
  - `GET <api>/api/v1/packages/<name>` and `/api/v1/native/<name>`, no token → `Package { name, native, owners, versions: [Version { version, yanked, meta, yank_message, has_readme, downloads, published_by, yanked_by, unyanked_by }, ..] }`; `GET <api>/api/v1/packages/<name>/<version>/readme` → the README as markdown.

  A failure is a 4xx / 5xx status with `Error { message: ".." }`, whose message `nabi` reports; 401 means the token is not valid. `nabi` skips reply fields it does not know.
- `meta` holds what describes a package and does not take part in resolution, from the manifest (9.10.3) and the package. Fields without a value are left out.

  | Field | From |
  |-------|------|
  | `description`, `license`, `license_file`, `authors`, `homepage`, `documentation`, `repository`, `readme` (the path), `keywords`, `categories`, `include`, `exclude`, `publish`, `metadata`, `links`, `default_run` | The manifest's fields of the same names |
  | `has_lib` | Whether the package has `src/lib.nabi` |
  | `bin_names` | Its executables: `[<name>]` with `src/main.nabi`, else `[]` |
  | `readme_content` | The text of the readme (9.10.3), at most 524288 bytes; a longer one is an error at `nabi publish` |
  | `size` | The packed size: the bytes of the files at HEAD the package holds (`include` / `exclude`, 9.10.3) |

  Manifest fields `nabi` does not know are sent in `meta` as written, so a newer registry can store them. For `publish --native`, `meta` is the file's `description`, `license` and `repository`.
- The registry API needs a token; an `--index-dir` checkout needs none. The token for the default registry is `NABI_REGISTRY_TOKEN` if set and non-empty, else the one `nabi login` saved. For another index it is the one `nabi login --registry <url>` saved; `NABI_REGISTRY_TOKEN` is never sent there (set, it warns once: ``NABI_REGISTRY_TOKEN goes only to https://github.com/nabi-lang/packages-index; for <url> run `nabi login --registry <url>` ``). Without a token, it is an error "run `nabi login` first" (``run `nabi login --registry <url>` first`` for another index).
- `nabi login` takes a registry account token, created on the website (`<api>/me`, signed in with GitHub; currently only the registry's maintainers can create one). When the token is omitted it is read from standard input, not echoed (echo is restored on Ctrl-C). `login` asks `/api/v1/me`: it refuses a token the API rejects, and an account whose `publish` is false ("this account cannot publish yet: publishing is open to the registry's maintainers only"), saving nothing. Otherwise it saves the token for the index, `--registry <url>` or else the default registry's, in `$NABI_HOME/tokens.nabion` as `Tokens { tokens: { "<index url>": "<token>" } }` (keys without a trailing `/`; fields and other indexes already there are kept), and prints "Logged in as <login>". A token is sent only to the registry API of its own index.
- `tokens.nabion` is the only token file.
- `tokens.nabion` is written mode 0600 on Unix. On Windows it gets the default permissions of `$NABI_HOME`, which is under the user's profile.
- `nabi logout` removes the default registry's token from `tokens.nabion`; `--registry <url>` removes that index's entry; `--all` removes every token.
- The token reaches `curl` only on its standard input: never on a command line or in the environment, and never printed (errors show `***`).

## 9.10.4b. Native packages

A native package is a prebuilt archive per target triple plus a value:

```nabion
Native {
    name: "png", version: "1.6.43",
    deps: { "zlib": Dep::native("^1.3") },
    targets: {
        "x86_64-unknown-linux-gnu": Artifact {
            url: "https://example.com/png-1.6.43-x86_64-linux.tar.gz",
            sha256: "<64 hex>",
            links: { "png16": "static" },
            runtime: [],
        },
    },
    yanked: false,
}
```

The value is one index line, or a file for `Dep::native(url:, sha256:)`, whose artifact urls may be relative to it. A file `nabi publish --native` reads may also have `description`, `license` and `repository`; they go to the registry API in `meta`, not into the index line.

- The archive (any tar-readable format) holds `lib/`: `lib<n>.a`, `lib<n>.dylib` / `lib<n>.so`, Windows `<n>.lib` (static or import library). It also holds the `runtime` files and license files. Nothing in it is run.
- Hashes are mandatory. An archive is checked before unpacking, and is refused if it has absolute or `..` paths, links leaving it, or lacks a named library or runtime file.
- On Apple targets runtime dylibs must have an `@rpath/` install name.
- Builds use the target triple's artifact; a missing one is an error listing the triples it has.
- `nabi` passes `nabic` `-L <archive>/lib`, `-l kind=name` per link and `--native-runtime` per runtime file, dependents before dependencies.

```
nabi add --native <name>[@<req>]
nabi remove --native <name>
nabi search --native <query>
nabi publish --native <file> [--dry-run] [--registry <url> | --index-dir <dir>]
nabi yank --native <name>@<version> [--undo] [--registry <url> | --index-dir <dir>]
```

`publish --native` fetches and checks every target, and requires absolute urls, a license file, and dependencies from an index only. Publishing it and `yank --native` follow the rules for registry packages: an `--index-dir` checkout, else the registry API with the token (`native: true` in `Yank`).

- On ELF targets a runtime library's soname must equal its file name. A `dylib` link for a `-musl` target is an error.
- `repl` / `jupyter` load native packages: shared libraries through the system loader, static archives linked into the session. `check` / `lint` / `doc` link nothing.
- Copies beside an executable are build outputs.

### System libraries and frameworks

```nabion
native: {
    "z": Dep::system(pkg_config: "zlib", brew: "zlib", vcpkg: "zlib", apt: "zlib1g-dev"),
    "png": Dep::system(pkg_config: "libpng16", libs: ["png16"], static: true),
    "Metal": Dep::framework(),
}
```

`Dep::system` names a C library installed on the machine. All its arguments are optional and named: `pkg_config` (the `.pc` module; default the key), `brew` (the Homebrew formula), `vcpkg` (the vcpkg port), `apt` (the package the error tells a Linux user to install), `libs` (what to link when no `.pc` file says; default the key) and `static: true` (link static archives, with the `.pc` file's private parts). Each build finds it, running no program, in this order:

1. pkg-config `.pc` files, read in-tree: `PKG_CONFIG_PATH`, then `PKG_CONFIG_LIBDIR` if set, else the target's multiarch directory (`/usr/lib/x86_64-linux-gnu/pkgconfig`, ..), `/usr/local/lib/pkgconfig`, `/usr/local/share/pkgconfig`, `/usr/lib64/pkgconfig`, `/usr/lib/pkgconfig`, `/usr/share/pkgconfig` and on macOS Homebrew's `lib/pkgconfig`. Variables (`${prefix}`, `${pcfiledir}`), `Requires` (and with `static`, `Requires.private` and `Libs.private`) are followed; `PKG_CONFIG_SYSROOT_DIR` prefixes the paths they give.
2. Homebrew, on macOS (or with `HOMEBREW_PREFIX` set): `$HOMEBREW_PREFIX/opt/<formula>` (default `/opt/homebrew`), its own `.pc` file, else its `lib/` and `include/`.
3. vcpkg, with `VCPKG_ROOT` (or `VCPKG_INSTALLATION_ROOT`) set: `installed/<triplet>`, the triplet from the target (`x64-windows`, `arm64-windows`, `x64-windows-static` for `static`, `x64-linux`, `arm64-osx`, ..) unless `VCPKG_DEFAULT_TRIPLET` names one; on Windows the DLLs of its `bin/` go beside the executable.
4. The macOS SDK's own libraries (`usr/lib/lib<name>.tbd`: `z`, `sqlite3`, ..).

A cross build looks only where the target's files are: `PKG_CONFIG_PATH`, `PKG_CONFIG_LIBDIR`, the target's multiarch directory, and with `PKG_CONFIG_SYSROOT_DIR` that sysroot's default directories; never the host's Homebrew. `nabi` passes what it finds as `-L` and `-l`; the header directories go to `c` sources. Not found is an error that names what it looked in and the install command of the host: `` system library `png` (pkg-config `libpng16`) not found for x86_64-unknown-linux-gnu (looked in pkg-config (libpng16.pc)): install it with `sudo apt install libpng-dev` (or your distribution's package of it), or set PKG_CONFIG_PATH to the directory holding libpng16.pc `` (`brew install` on macOS, `vcpkg install <port>:<triplet>` on Windows).

`Dep::framework()` takes no arguments: the key is the framework's name (`Foundation`, kept as Apple spells it), linked as `#[link(name = "Foundation", kind = "framework")]` ([5.9.9](../005-modules/509-extern-block.md#599-linklibname-naming-the-library)). A build for a target other than an Apple one leaves it out.

The system libraries of every package in the graph are linked, one per name; a registry package's come from its `nabi.nabion` (an index line has none). They are not in `nabi.lock`: they are what the machine has.

## 9.10.4c. Features

A package declares features in `features`, a dict from feature name to the list of what the feature enables. Names are `[a-z0-9][a-z0-9_-]*`, words that say what they are (`http1`, `http2`, `tls`). Each entry is one of:

| Entry       | Enables                                                              |
|-------------|----------------------------------------------------------------------|
| `<f>`       | Feature `f` of this package                                          |
| `dep:<d>`   | The optional dependency `d`                                          |
| `<d>/<f>`   | Feature `f` of dependency `d`, and `d` itself if it is optional      |
| `<d>?/<f>`  | Feature `f` of the optional dependency `d`, only if something else enables `d` |

- `default` is an ordinary feature that is on unless turned off: `--no-default-features` for the package being built, `default_features: false` on a dependency.
- An optional dependency (`optional: true`) is in the build only when a `dep:<d>` or `<d>/<f>` entry of an enabled feature enables it. There is no implicit feature of its name, and it is an error if no entry enables it.
- It is an error if an entry names a feature or dependency (of `deps`) the package does not have, if `dep:` or `?` names a non-optional dependency, or if features form a cycle. Diagnostics show `file:line:col`. It is an error if a dependency is asked for a feature it does not have.
- Resolution: the lock covers the whole graph. The build then starts from the root package's features (`default` unless turned off, `--features`, or every feature with `--all-features`) and follows every enabled dependency, with the features it asks of it plus its `default`. Features are additive and unified: each package has one feature set for the whole graph. The root's `dev_deps` and the features they ask count only for `test` / `bench`. The result does not depend on the order of anything.
- `--features` takes the root's feature names and `<dep>/<f>` for its direct dependencies.
- Each package is compiled with `cfg(feature = "<f>")` for exactly its enabled features ([6.3.6](../006-meta-system/603-cfg.md#636-where-parameters-come-from)). Native packages have no features.
- A script's ```` ```nabion ```` block has no `features` (9.10.7).

## 9.10.5. Environments

Environments replace conda / venv. There is no activation step.

- A package (a directory with `src/main.nabi`, `src/lib.nabi` or `nabi.nabion`) uses the dependencies of its own `nabi.nabion` and nothing else; it never sees libraries installed into an environment. Neither does a script. A command that works on a package is an error with `--env`.
- A named environment keeps `nabi.nabion`, `nabi.lock` and `target/` in `$NABI_HOME/envs/<name>/`. Its `deps` are the libraries installed with `nabi install` ([9.10.5a](#9105a-nabi-install--nabi-uninstall)), for the REPL and Jupyter; only `nabi install` / `nabi uninstall` and `nabi env import` change them. Path dependencies in it are absolute. Installed libraries are prebuilt into its `target/`.
- The environment used outside a package (the REPL, Jupyter, the language server, `nabi install`) is chosen in the order `--env <name>` > `NABI_ENV` > `nabi env use` (`$NABI_HOME/envs/.current`) > `default`; `default` is created by the first `nabi install` into it. Inside a package, `nabi repl` / `jupyter` / `lsp` without a path use the package, or with `--env` that environment.
- An environment name is ASCII letters, digits, `_`, `-` and `.`, not starting with `.`, at most 64 characters, and not a Windows device name (`con`, `nul.txt`, ..). `nabi env create` refuses a name another environment has in a different case (`Foo` beside `foo`): Windows and macOS see them as one directory.
- Every source of a name (`--env`, `NABI_ENV`, the `nabi env use` file, the arguments of `nabi env`) is checked. A name with a separator (`/`, `\`, `:`), `.` or `..` is an error everywhere (`` `../p` is a path, not an environment name; use a name from `nabi env list`, e.g. `default` ``). Any other name `nabi env create` would refuse is an error too, unless that environment exists (its directory was made by hand): then it is a warning and the command goes on. `nabi jupyter install --env <name>` requires the environment to exist, except `default`.
- `nabi env use` writes the name to `$NABI_HOME/envs/.current`. Only the first line is read. `nabi env remove` of the chosen environment removes the file, so there is no choice.

### Where dependencies come from

| Compiled as | Dependencies | Lock | Prebuilt into |
|---|---|---|---|
| A package | its `nabi.nabion` `deps` (`dev_deps` for test / bench) | `<package>/nabi.lock` | `<package>/target/<profile>/deps/` |
| A script | the merged `deps` of the `//! nabion` blocks of it and of the files its `mod`s load | `cache/scripts/<hash>/nabi.lock` | `cache/scripts/<hash>/target/` |
| The REPL / Jupyter outside a package | the active environment's installed libraries | the environment's | the environment's `target/` |

```
nabi env create <name>
nabi env list                      # marks the current environment with * and (active); with none, says how to make one
nabi env remove <name>
nabi env use <name>                # changes the default environment outside projects
nabi env export <name> <dir>       # writes nabi.nabion / nabi.lock to <dir>
nabi env import <name> <dir | nabi.nabion>    # creates from a project or export (relative paths made absolute)
nabi env shell [name]              # $SHELL with NABI_ENV=<name>
nabi repl --env <name>
```

`:add <dep> ..` in the REPL runs `nabi install <dep> ..` into the current environment, or `nabi add` in a package's session; it then re-reads the dependencies and attaches them to the session.

## 9.10.5a. `nabi install` / `nabi uninstall`

```
nabi install <name>[@<req>] [--registry <url>]
nabi install [<name>] --path <dir | file.nabi>
nabi install [<name>] --git <url> [--rev <rev> | --tag <tag> | --branch <branch>]
nabi install .. [--features <f,..>] [--no-default-features] [--lib | --bin] [--force]
nabi install --native <name>[@<req>] [--registry <url>]
nabi install --list
nabi uninstall [--native] <name>..
```

- The package at the source is fetched into the cache first ([9.10.4](#9104-dependencies-nabi-add--remove--update--deps)). A package with `src/main.nabi`, or a script file, is an executable: it is built with `--release` and its executable placed in `$NABI_HOME/bin/<name>`, as with `cargo install`. A git or registry package is built from a scratch copy of its checkout; a path package in place; a script file in place. `$NABI_HOME/installs.nabion` (`Package { deps: { "<name>": <source> } }`) records each executable and its exact source (a package as `=<version>`, a git package with its `rev:`). When the package also has `src/lib.nabi`, `nabi install` needs `--bin` or `--lib`: without either it is an error naming `--bin` for the executable and `--lib` for the library, and nothing is installed. `--bin` on a package without `src/main.nabi` is an error, and so are `--bin` and `--lib` together.
- A package with only `src/lib.nabi`, or any package with `src/lib.nabi` given `--lib`, is a library and goes into the active environment, as with `pip install`, for `nabi repl`, `nabi jupyter` and `:add` ([9.10.5](#9105-environments)): it is written into the environment's `nabi.nabion` (a registry package without a requirement as `^<newest version>`, a path as an absolute path), resolved into its `nabi.lock`, and prebuilt into its `target/`. Scripts do not see it: a script gets a library with `nabi add <name> <file.nabi>`; the output line names this. If it does not build, nothing is installed. `--lib` on a package without `src/lib.nabi`, or on a script, is an error.
- `<name>` is the library's top-level name, or the executable's file name. If omitted, it is the package's `name`, else the directory name (path), the repository name without `.git` (git), or the file stem (script). It follows the naming rules of [9.10.3](#9103-nabinabion--nabilock).
- `--features` / `--no-default-features` go into the library's `Dep`, or to the executable's build; `--dev` / `--optional` are errors. `--native` adds a native package to the environment's `native` ([9.10.4b](#9104b-native-packages)).
- The proxies `nabi`, `nabic`, `nabicc`, `nabils` ([9.10.6](#9106-toolchain)) are never replaced, and neither is a file in `$NABI_HOME/bin` that `nabi install` did not put there (an error). Installing an executable again: one from the same registry version or git commit with the same features is kept, not rebuilt (`` `tool` 0.1.0 is installed already (<path>); to rebuild it, run `nabi install tool --force` ``), and `--force` rebuilds it (`Reinstalled`); another version replaces it (`Upgraded tool from 0.1.0 to 0.2.0`, `Downgraded ...`), and another git commit too (`Replaced <url> at <commit> with ...`). One from a path or a script file is always rebuilt, as with `cargo install --path`. Library installs are unchanged.
- `--list` prints ``libraries (environment `<env>`):`` and `executables (<bin dir>):`, each followed by `  <name> <source>` lines in install order.
- `nabi uninstall <name>..` removes each from the active environment's libraries (re-resolving its lock), or else removes the executable `nabi install` put in `$NABI_HOME/bin`; `--native` removes a native package from the environment. A name that is neither is an error.
- Apart from `nabi install`, only `nabi toolchain` (toolchains, proxies, `config.nabion`), `nabi login` (`tokens.nabion`), `nabi env` (environments) and the caches write under `$NABI_HOME`.

## 9.10.6. toolchain

`nabi` / `nabic` / `nabicc` / `nabils` in `$NABI_HOME/bin/` are proxies that run the same executable of the active toolchain (rustup-style). The active toolchain is chosen in this order:

1. `+<version>` on the command line (`nabi +0.4 build`)
2. The environment variable `NABI_TOOLCHAIN`
3. `toolchain: "X"` in the block of the script run ([9.10.7](#9107-inline-manifest-for-single-file-scripts)), for `nabi <file>` and `nabi run` / `build` / `check` / `test` of a `.nabi` file outside a package (the proxy reads the first 64 KiB of the file)
4. `toolchain: "X"` in the manifest (the nearest `nabi.nabion`; the proxy reads only this field, so fields it does not know keep the pin)
5. The default (`Config { toolchain }` in `$NABI_HOME/config.nabion`)

A pin is one of:

- a version or a prefix of one, matching the versions under it (`0.3` is the highest `0.3.*`, `0.3.1` is `0.3.1`);
- a requirement as in 9.10.4a (`^0.3.1`, `~0.3.1`, `=0.3.1`, `>=0.3, <0.5`), matching the highest installed version it accepts;
- a toolchain directory name (`0.3.1-<triple>`), matching that directory.

The highest installed release that matches wins; a prerelease (`0.3.1-rc.1`) wins only when no release matches, so `0.3` takes `0.3.0` over `0.3.1-rc.1`, and a series with only candidates installed takes the newest of them. A pin naming a prerelease exactly (`0.3.1-rc.1`) matches only it. A manifest pin that is none of these is a warning saying what to write (`` `toolchain: "banana"`: write a version (`0.1`), a requirement (`^0.1.0`) or a toolchain directory name ``). If no toolchain matches, it is an error naming the pin and where it came from, with the command that installs it: `nabi toolchain install <pin>` for a full version, else `nabi toolchain install <version>` with a full version the pin accepts, or `--from`.

A series is the releases one breaking slot covers: `0.MINOR` while the major version is 0 (`0.1`), then `MAJOR` (`1`). Only the newest series gets fixes and security releases; an older one stays downloadable and installable but is not supported.

### Compatibility

A toolchain builds the code an earlier toolchain of its series built: below 1.0 a change of `y` in `x.y.z` may break code (`0.1.7` to `0.2.0`), from 1.0 a change of `x` does (`1.9.0` to `2.0.0`), and a later `z` (below 1.0) or `y` (from 1.0) does not. A package's `nabi_version` (9.10.3) follows this rule: `"0.1.3"` admits 0.1.3 to the last 0.1.x, `"1.4.2"` admits 1.4.2 to the last 1.x. A pre-release toolchain counts as its release (`0.1.4-dev` as `0.1.4`).

The proxies are copies of the `nabi` of the newest installed toolchain: the highest version for this host, a prerelease only when no release is installed. A newer `nabi` keeps running older toolchains this way, so the proxy contract is fixed across 0.x: the order above, then running `toolchains/<dir>/bin/<tool>` with the same arguments and `NABI_HOME` set, and `NABI_TOOLCHAIN` set to the `+<version>` the proxy took off.

Each toolchain's `nabi` runs only its own `nabic`, the one beside it, so the flags, environment variables and files between `nabi` and `nabic` are never a contract across versions. A `nabi` run directly (a development build, a package manager's, a toolchain's `bin/nabi`) where `+<version>`, `NABI_TOOLCHAIN` or the manifest's `toolchain` names another installed toolchain hands the command to that toolchain's `nabi`: it runs it with the same arguments from the directory it started in, with `NABI_HOME` set, and exits with its status (on Unix it replaces itself; on Windows it waits). For a command that compiles (one delegated to `nabic`, or a tool that compiles), the manifest is that of the package the command works on (`nabi build ../app` reads `../app/nabi.nabion`); for the other commands it is the nearest one. It does not hand off when the pin matches its own version, when the toolchain is the directory it runs from (compared by real path), or when that toolchain's `nabi --version` prints the same line as its own (the same build). Without a pin, the default does not apply: a `nabi` run directly keeps the command. `nabi toolchain`, `help`, `version`, `completions` and `--version` never hand off. A toolchain handed the command that hands it back again is an error naming both executables, never a loop. A pin naming no installed toolchain is the error above for commands that compile; the other commands go on.

```
nabi toolchain list                         # marks the default with * and (default)
nabi toolchain install <version> [--from <dir | archive | url>]
nabi toolchain default [<version>]          # without an argument, prints the current default
nabi toolchain remove <version>
nabi toolchain update [<version>]
nabi toolchain clean                        # remove every installed toolchain but the default
```

- An installation is `$NABI_HOME/toolchains/<version>-<triple>/` and is self-contained ([9.2.19](902-aot-pipeline.md#9219-installation-layout)).
- `install --from` installs from a directory, an archive or its URL; `<version>` is a pin the toolchain must match. Without `--from`, `install` downloads a release (Distribution, below): `<version>` is a full version (`0.1.0`); anything else is an error that says to write one (`` nabi toolchain install: `0.1` is not a full version; write one like `0.1.0` ``). A toolchain already installed is not downloaded again (`toolchain <dir> is already installed`). It does not change the default.
- `install` verifies with `bin/nabic --version --verbose`; the version and host must match.
- The first installed toolchain becomes the default, and the `bin/` proxies are created at that point.
- `install`, `default`, `update`, `remove` and `clean` rewrite the `bin/` proxies when one is missing or is not the newest toolchain's `nabi`, and otherwise leave them. Making an older toolchain the default keeps the newer proxies; removing the newest makes them the next newest's.
- `nabi toolchain update` replaces the default toolchain under `$NABI_HOME`. Without an argument it installs the newest release (`latest/download/nabi-version`), also when that is in another series (rustup-style). When the default moves to another series, the note that the old toolchain stays installed also says that only the newest series gets fixes, so projects should move, and how a project stays on the old series until it does: ``note: nabi <old version> stays installed, but <old series> gets no more fixes; move projects to <new version>. Until a project moves it can pin <old series> (`toolchain: "<old series>"` in nabi.nabion or a script's `//! nabion` block) or run `nabi +<old series> ..` ``. With an argument, it installs that release: the argument is a full version (`update 0.1.1`), and there is no series selection; anything else is an error that says to write one (`` nabi toolchain update: `0.1` is not a full version; write one like `0.1.0`, or nothing for the newest release ``). The installed toolchain becomes the default and the `bin/` proxies are rewritten when the newest changed; on Windows a running proxy is renamed before its copy replaces it. It downloads the release archive and verifies its SHA-256 (Distribution, below), unpacks it apart and moves it into place, then checks that the proxies run it; on failure the previous default and proxies are restored and the new toolchain is removed. Earlier toolchains stay installed. A version already installed is made the default without a download. `nabi update` re-resolves `nabi.lock`; `nabi toolchain update` replaces the toolchain (9.10.4).
- After `nabi toolchain default <version>` or `nabi toolchain update`, the completion scripts the installers wrote (Distribution, below) are rewritten from the new default toolchain's `nabi completions`. Only files that exist are rewritten; none is created, and a toolchain without `completions` leaves them as they are. This prints nothing; `-v` names the files rewritten.
- After `nabi toolchain default <version>` or `nabi toolchain update`, `nabi` looks up `nabi` on `PATH` (on Windows with each `PATHEXT` extension). When the first one found is not `$NABI_HOME/bin/nabi` (compared by real path), it warns with that file and the version its `--version` prints: `warning: your shell runs <path> (nabi X) before ~/.nabi/bin/nabi; put ~/.nabi/bin first on PATH or remove the other one`. `-q` leaves the warning out (the installers run that way).
- `nabi --version -v` (or `nabi version -v`) prints, after the version line, `binary` (this executable), `toolchain` (the active toolchain and why: `+` on the command line, `NABI_TOOLCHAIN`, the `nabi.nabion` that pins it, or the default; `none` when there is none), `nabic` (the `nabic` a command here runs: its own, or the pinned toolchain's when it hands off) and `host`, one `key: value` line each with the values aligned, the form of `nabic --version -v` (`host: <triple>`). `--format json` (`nabi version -v --format json`, `nabi --version --format=json`) prints one JSON object instead: `schema` (now `1`), `version`, `line` (the text after `nabi ` in the version line) and, with `-v`, those four keys.
- `nabi toolchain clean` removes every installed toolchain except the current default, an opt-in tidy-up for the earlier versions that `update` leaves installed. A version a project pins is removed too, so reinstall it if a build needs it; with no default set it is an error.

### Build cache

```
nabi cache dir                              # print the toolchain's cache directory
nabi cache prune [--unused <days>] [--max-size <size>]
nabi cache clean                            # remove everything in it
```

The cache is the active toolchain's `lib/nabi-cache/` ([9.2.19](902-aot-pipeline.md#9219-installation-layout)): the function cache, with the standard library code builds compiled, and the library's collected items ([9.2.16](902-aot-pipeline.md#9216-incremental-build)). Nothing in it is needed; a build computes again what is missing.

- `prune` removes the entries (of the function cache and the library stages) no build read for `--unused` days (default 30, what builds remove themselves once a day), then the least recently read ones until at most `--max-size` is left (bytes, or a number with `K`, `M` or `G`; default the cap, `NABI_CACHE_MAX_SIZE`, [9.2.16](902-aot-pipeline.md#9216-incremental-build)), and prints how many it removed and kept.
- `clean` removes everything in the cache directory.

### Files shared across toolchains

Every toolchain installed in one `$NABI_HOME` reads and writes the same files there: `config.nabion`, `tokens.nabion`, `sources.nabion`, `installs.nabion` and `envs/.current`. A newer `nabi` may have written fields into them that an older one does not know.

- Reading them, `nabi` skips fields it does not know, and in `installs.nabion` also `Dep` arguments and entry kinds it does not know (the entries of a kind it does not know are not listed). With `-v` it names what it skipped once per file: ``note: <file>: skipped `x`, written by a newer nabi (kept in the file)``.
- Writing them, it keeps what it does not know: `nabi toolchain default` sets the one field of `config.nabion`, `nabi install` / `nabi uninstall` set or remove the one entry of `installs.nabion`, `nabi login` sets its index's entry in `tokens.nabion`, and `nabi logout` removes only that entry. `sources.nabion` is only read.
- A field of `nabi.nabion` `nabi` does not know is a warning ([9.10.3](#9103-nabinabion--nabilock)); a `nabi.lock` of a version `nabi` knows stays strict.
- A newer `nabi` adds settings in new files, not as new fields of these, so that older toolchains keep working beside it.

### Distribution

- A release archive is `nabi-<version>-<triple>.tar.gz` (Windows `.zip`), containing the toolchain layout; outside Windows a `nabi-<version>-<triple>.tar.xz` with the same files sits beside it. Each archive has `<archive>.sha256` beside it: the archive's SHA-256 in hex, optionally followed by the file name (sha256sum format).
- `install.sh`, `install.ps1` and `nabi toolchain install --from <url>` download `<url>.sha256` from the same place and refuse an archive whose SHA-256 differs; a missing `.sha256` is an error. An archive given by path is checked against `<path>.sha256` when that file exists. Signatures are not defined; HTTPS vouches for the origin.
- `install.sh` (<https://nabi.sh>) installs into `$NABI_HOME` from an archive or from GitHub releases of the `nabi-lang` organization. It downloads the `.tar.xz` where tar can read it (macOS, or `xz` on `PATH`) and the release has one, else the `.tar.gz`. `nabi toolchain install` downloads the `.tar.gz` (on Windows the `.zip`).
- `install.sh` then sets up the shells, as rustup does. It writes `$NABI_HOME/env` (sh) and `$NABI_HOME/env.fish`, which put `$NABI_HOME/bin` first on `PATH` unless it is there already, and adds the guarded line `[ -f "$HOME/.nabi/env" ] && . "$HOME/.nabi/env"` (for that `$NABI_HOME`) once to each of `~/.profile`, `~/.bashrc`, `~/.bash_profile` and `~/.zshenv` that exists, and to the login shell's file, made if missing (bash: `~/.bashrc`, and `~/.profile` unless `~/.bash_profile` or `~/.bash_login` exists; zsh: `$ZDOTDIR/.zshenv`; other shells: `~/.profile`). Fish gets `~/.config/fish/conf.d/nabi.fish` (under `$XDG_CONFIG_HOME` when set) when it is the login shell or its configuration directory exists. The completion scripts of `nabi` go to `${XDG_DATA_HOME:-~/.local/share}/bash-completion/completions/`, `$NABI_HOME/zsh/` (which `env` adds to zsh's `fpath`) and, for fish, `~/.config/fish/completions/`. It names the files it changed; a new shell picks up the change, and running it again changes nothing. Without `$NABI_HOME` the guarded line does nothing.
- `install.ps1` puts `$NABI_HOME\bin` first on the user's `Path` and writes `$NABI_HOME\completions\nabi.ps1`, loaded by a guarded line in the PowerShell profile, added once.
- `install.sh` and `install.ps1` print status lines on stderr as `nabi` does, in the same palette and by the same rules ([9.1.5](901-nabic-cli.md#915-global-options), [9.1.8](901-nabic-cli.md#colour)), without `tput`. `install.sh` tests its stderr, so `curl .. | sh` on a terminal is coloured.
- `--no-modify-path` (`install.ps1 -NoModifyPath`), or `NABI_NO_MODIFY_PATH=1`, leaves the shell files, `Path` and the profile alone and prints what to add to `PATH`.
- A release archive carries the official packages in `lib/nabi/packages/` (`nabi component`, 9.10.4a), each at its newest version whose `nabi_version` admits that release, so code that uses them builds without a network. `--no-packages` (`install.ps1 -NoPackages`), or `NABI_NO_PACKAGES=1`, runs `nabi component remove packages` for the installed toolchain after the install; if that fails, the toolchain stays installed with them and the script says so.
- Releases are served from `$NABI_DIST` (default `https://github.com/nabi-lang/nabi/releases`): `<dist>/download/v<version>/<archive>` with its `.sha256`, and `<dist>/latest/download/nabi-version`, one line naming the newest release of any series, which a bare `nabi toolchain update` reads. A release candidate (`<version>-rc.N`) is a prerelease: `latest/` never names it, so it is installed only by its full version.
- `nabi toolchain update <version>` / `install <version>`, `install.sh --version <version>` and `install.ps1 -Version <version>` take a full version; a release other than the newest is installed that way. No per-series file is published.

### Installer contract

The `nabi toolchain update` / `install` of every released toolchain and the install scripts of every release must be able to install every later release, so a release archive keeps what they rely on. This contract is frozen; a release that breaks it is not published.

- The name: `nabi-<version>-<triple>.tar.gz` (Windows `nabi-<version>-<triple>.zip`), and outside Windows also `.tar.xz`, served as in Distribution above, each with its `.sha256`.
- The pointer: `<dist>/latest/download/nabi-version`, one line naming the newest release, so that their bare `nabi toolchain update` reaches it.
- The layout: one top directory `nabi-<version>-<triple>/` holding `bin/nabi` and `bin/nabic` (Windows `bin\nabi.exe` and `bin\nabic.exe`), `lib/<runtime>` (`lib/libnabir.a`; Windows `lib\nabir.lib`) and `lib/nabi/core/src/lib.nabi`.
- `bin/nabic --version --verbose` prints `nabic <version>` as its first line (optionally followed by a space and more text) and a line `host: <triple>`.

## 9.10.7. Inline manifest for single-file scripts

An anonymous package ([4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)) gets its dependencies only from ```` ```nabion ```` blocks holding `Package { deps: { .. } }` (and `native`) inside the `//!` lines at the very start (after a shebang line) of its file and of each file its `mod`s load (like uv's inline script metadata). Libraries installed into an environment are never used. A block may contain only `deps`, `native` and `toolchain` (no `features`, so no optional dependencies). `toolchain` pins the toolchain that runs the script (9.10.6) and is allowed only in the block of the file that is run; in a module's block it is an error naming the file run. A `nabi` run directly hands the script to the pinned toolchain's `nabi`, as for a package. Code blocks of other languages are ignored. Paths in a block are relative to the file that holds it. The blocks are merged; one library given different entries in two files is an error naming both files and both entries. Running a submodule file directly uses only the blocks of the files that run reaches. `nabi add <name> .. <file.nabi>` / `nabi remove <name> <file.nabi>` write a block ([9.10.4](#9104-dependencies-nabi-add--remove--update--deps)), and `nabi init --script` turns a script into a package ([9.10.2](#9102-nabi-new--nabi-init-package-creation)). A script keeps its lockfile and builds in `cache/scripts/<hash>/`, keyed by its path and the path and block of every file it reaches. A script's executable is cached by everything it is built from, every file it includes and its libraries included. A `use x::..` of a package a script lacks is an error whose help is the command to run, ``run `nabi add x <file>` ``, with a note when the active environment has `x`; in a package it suggests `nabi add x`. `import x` (and `include` / `require`) is an error whose help is `mod x;` when `x.nabi` is beside the file, `use` when `x` is a path root or a module the file declares, and otherwise `nabi add x <file>`.

````nabi
//! ```nabion
//! Package {
//!     deps: { "util": Dep::git("https://example.com/util.git", tag: "v1") },
//! }
//! ```

use util::greet;
greet();
````

Running a single-file script reuses its executable. `nabi run` of an anonymous package keeps the executable in `<target>/[<triple>/]<profile>/scripts/<key>/`, where `<target>` is the script's `target/` (under `cache/scripts/<hash>/`, beside its `nabi.lock`); a file that a `nabi.nabion` makes a single-file package (9.10.1), and a lone file built by `nabic`, use the lone-file cache instead ([9.2.16](902-aot-pipeline.md#9216-incremental-build)). `<key>` is a hash of everything the build is made from: the toolchain, the build options, lint levels and target, the source and every file it includes, the runtime and the C library the toolchain ships for the target, and the dependencies passed to it with their prebuilt libraries. When the key matches, nothing is compiled and the warnings of the build that made the executable are printed again. A new executable is written under a scratch name and renamed into place, so concurrent runs of one script are safe. An entry unused for a day is removed when another is built.

## 9.10.8. Tools

The tools are `nabi`'s own subcommands. `nabi` runs them in its own process with the compiler linked in; those that compile take the compile options of `nabic build` ([9.1.2](901-nabic-cli.md#nabic-build-aot-build)) and get the package's dependencies, features and lints as 9.10.1 describes. Their diagnostics, progress lines and colour follow [9.1.8](901-nabic-cli.md#918-output-format), and their usage errors and exit codes follow [9.1.6](901-nabic-cli.md#916-exit-codes) unless a tool says otherwise. `nabi lsp`, `nabi repl` and `nabi jupyter` survive an internal compiler error: the language server reports it as an error diagnostic of the analyzed buffer, the REPL / kernel fails that cell ([9.7.5](907-jit-cells.md#975-error-modes)); both keep serving.

### `nabi run`: build and run

```
nabi run [path] [build options] [-- args...]
```

`nabi run` builds an executable package as `nabic build` does and runs it at once (an executable of another `--target` through its runner, below); arguments after `--` are passed to the executable unchanged, any bytes ([9.1.1](901-nabic-cli.md#911-general-form)). The options are those of `nabic build`. A library package has nothing to run, so it is an error.

A package is built to `target/<profile>/<name>`; a lone `.nabi` file is built as an anonymous package and runs from the cache of 9.10.7, so no `target/` is created next to the file. `nabic build <file>.nabi` builds it and copies the executable next to the file instead. If nothing has changed since the previous build, `nabi run` prints `Fresh`, then that build's warnings again, and skips the build ([9.2.16](902-aot-pipeline.md#9216-incremental-build)).

```
nabi run hello.nabi -- arg1 arg2
```

The exit code of `nabi run` is the exit code of the program. If the program terminates by signal `n`, it is `128 + n`.

The program does not outlive `nabi run`. A `SIGTERM`, `SIGHUP`, `SIGINT` or `SIGQUIT` sent to `nabi` goes on to the program (one the terminal sends to its foreground group already reaches it), and if the program ends by that signal, `nabi run` ends by it too. If `nabi run` itself is killed, the program is killed with it; on Windows the program runs in a job object that ends with `nabi`.

### `nabi dev`: run with hot reload

```
nabi dev [path] [-p <member>] [--bin <name>] [build options] [-- args...]
```

`nabi dev` builds the package's program for hot reload ([9.14](914-hot-reload.md)), runs it, and watches its sources. A save rebuilds the local packages it touches and the program takes the new code while it runs, keeping its state.

- The *host* (the runtime and every registry or git package) is built once, as `target/dev/<name>-host`. Each local package (the package, its `path` dependencies, the workspace members it uses) is a hot plugin, built with `nabic build --hot` as `target/dev/<package>.<n>.dylib` (`.so`, `.dll`), build `n` from 1, with its layout `target/dev/<package>.<n>.layout` beside it. `target/dev/` is emptied when a session starts.
- The `src/` trees of the local packages are polled every 100 ms (modification time and size of each `.nabi` file). A change is taken once the files have been still for 50 ms, so a save of several files is one change. It rebuilds the packages whose files changed and every local package that depends on them, dependencies first, incrementally ([9.2.16](902-aot-pipeline.md#9216-incremental-build)), and sends the new versions to the program ([9.14.5](914-hot-reload.md#9145-dev-channel)).
- The program's replies are printed as status lines: `Applied`, `Waiting`, `Refused` (with what to do), `Superseded`.
- A compile error is printed as by `nabi build`, followed by `` Kept the running code of `game`: fix the error and save ``. The program goes on with its old code.
- A change to `nabi.nabion` (the package's, a local dependency's or the workspace root's), to `nabi.lock`, or to the toolchain's `nabi` or `nabic` restarts the program, after a line that says why: `` Restarting nabi.nabion changed ``. A restart resolves the dependencies again and builds everything anew; the program's state is lost.
- Keys: `r` restarts, `q` quits. On a terminal a key acts when pressed (`nabi dev` turns off line editing and echo, and puts them back when it ends); otherwise each line's first letter counts. The program's standard input is empty; its standard output and error are `nabi dev`'s.
- When the program ends by itself, `nabi dev` says so with its exit code and waits: a save or `r` runs it again, `q` quits. A program whose hot plugins do not load ends with exit code 101 after the loader's error ([9.12.6](912-plugins.md#9126-stability-and-errors)).
- Ctrl-C (or `SIGTERM`, `SIGHUP`) ends the session: the program is killed and reaped, and `nabi dev` exits with 130. `q` exits with 0.
- The arguments after `--` go to the program at every start.

`-p <member>` runs a workspace member; `--bin <name>` names the package's executable (a package has one, named after the package). The build options are those of `nabic build`, for the dev profile on this machine only: `--release` or another `-O` level is an error (``hot reload runs debug builds; drop `--release` ``), and so are `--target`, `-o` and `--crate-type` (``` `nabi dev` builds for this machine only; drop `--target` ```). A package without `src/main.nabi` is an error that says to run it in the program's package.

### Executables of other targets

`nabi run`, `nabi test` and `nabi bench` with a `--target` this machine does not run itself start each executable through a target runner, a program given the executable and then its arguments:

- `NABI_RUNNER=<program> [args]` (split on whitespace) is the runner of every target other than the host.
- Without it, `wasm32-wasip2` and `wasm32-wasip1-threads` modules run under `wasmtime` from `PATH`: `wasmtime run` with the host's environment (on `wasm32-wasip2` also its network and name lookup), an engine stack of 8 MiB, as deep as a native main thread's, and the host's file system at `/`. WASI has no working directory, so a relative path resolves from `/`.
- `wasm32-unknown-unknown` modules run under `node` from `PATH` with the toolchain's JS host, `nabi.mjs` in `lib/nabi/targets/wasm32-unknown-unknown/`.
- On Linux, a Linux target of another CPU runs under qemu-user's `qemu-<cpu>` from `PATH` (`qemu-aarch64`, `qemu-arm`, `qemu-riscv64`, `qemu-ppc64le`, `qemu-s390x`, `qemu-i386`, ...), with `-L /usr/<triple>` (`/usr/aarch64-linux-gnu`) when that directory exists and `QEMU_LD_PREFIX` is unset; without `qemu-<cpu>` it runs directly when binfmt_misc has a `qemu-<cpu>` handler.
- A target of the host's operating system for another CPU the host runs runs directly, before qemu-user is looked for: 32-bit x86 on x86_64, and 32-bit ARM on arm64 Linux when the CPU runs AArch32 code at EL0 (Neoverse N1, Cortex-A72; not Neoverse N2 or Apple silicon). No register tells a program that, so `nabi` runs a tiny static 32-bit ARM executable once to find out.

Otherwise, or when the program is missing, the command fails with an error that names what to install, or `NABI_RUNNER`: ``cannot run a wasm32-wasip2 module here: `wasmtime` is not on PATH; install it (https://wasmtime.dev), or set NABI_RUNNER to a program that runs it``. `nabi run` checks before it builds; `nabi test` and `nabi bench` after the harness is built, and `--list` lists without a runner. The exit code is the runner's, which is the program's for all of the above.

```
nabi test --target wasm32-wasip2
NABI_RUNNER="qemu-aarch64 -L /opt/sysroot" nabi run --target aarch64-unknown-linux-gnu
```

### `nabi lint`: linter

```
nabi lint [path] [--list] [--check-library] [-A|-W|-D <lint | group>..]
```

`nabi lint` runs the front end and the lints, without code generation (`nabic check` adds code generation, [9.1.2](901-nabic-cli.md#nabic-check-diagnostics)). Every compiling command reports the same lints ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)); `lint` runs only them and the front end. The diagnostic format is [9.1.8](901-nabic-cli.md#918-output-format); lint diagnostics have code `E0131`. The list of lints and groups is in [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control). `--list` prints every lint with its default level and the release that added it, then the groups. `--check-library` is as for `nabic check`.

Lint levels are set by `-A` / `-W` / `-D` ([9.1.5](901-nabic-cli.md#915-global-options)) and by attributes in code. If any `deny`-level lint fires, the exit code is `1`.

### `nabi fix`: apply fixes

```
nabi fix [path] [--allow-dirty] [--diff] [-A|-W|-D <lint | group>..]
```

`nabi fix` checks the package as `nabi lint` does and applies the fixes the findings carry: a finding of a lint, or a compiler warning, with exactly one edit (a diagnostic with several alternatives is left to the editor's quick fixes). It checks again and repeats while edits are found, at most 8 rounds; an edit overlapping another of the same round waits for the next. Only files of the package are changed; the dependencies' and the standard library's never. The lints with a fix:

| Lint | Fix |
|------|-----|
| `unused_variables` | a `let` name gets `_` in front (`let _y = 2;`) |
| `unused_imports` | the `use` is deleted when it imports only that name and is alone on its line, with no attribute above it |
| `bool_comparison` | `x == true` becomes `x`, `x == false` / `x != true` become `!x` (`!(x)` around an operator) |
| `while_true` | `while true` becomes `loop` |
| `redundant_cast` | `x as T` becomes `x` |
| `redundant_clone` | `.clone()` is deleted |
| `missing_f_prefix` | the string gets the `f` prefix |

The lint levels (`-A` / `-W` / `-D`, the manifest's `lints`, attributes) choose what is reported and so what is fixed: `nabi fix -A unused_imports` keeps the unused imports. When the package is in a git work tree with uncommitted changes under its directory, `nabi fix` refuses (``the work tree has uncommitted changes (src/main.nabi); commit them first so the fixes can be reviewed and undone, or run `nabi fix --allow-dirty` ``); `--allow-dirty` fixes anyway. Outside git there is no such check. `--diff` prints the edits of the first round as `nabi fmt --diff` does and writes nothing. An error that no fix resolves stops it with exit code `1` and ``N errors to fix by hand first; nothing was changed`` (after a round: ``the files fixed so far are kept``). It ends with `Fixed <file> (N fixes)` for each changed file, or `<name>: nothing to fix`.

### `nabi repl`: terminal REPL

`nabi repl` launches an interactive shell in the terminal. `nabi` with no subcommand is the same (9.10.1).

```
nabi repl [path] [-q]
```

Cell semantics follow the rules of [9.7.0](907-jit-cells.md#970-rules-of-the-cell-session): each cell runs exactly once, top-level `let` bindings and items carry over to later cells, and a failed cell leaves no trace in the bindings. An expression cell prints its value (`Display`, else `Debug`, else its type name).

- The default mode compiles each cell with Nabi's code generator and hands the code to a JIT host process that lives for the session; the host loads it and runs it once ([9.6](906-jit-engine.md)). Bindings, `static` state and `any` values carry over between cells. A binding whose type cannot be named at top level (closures, private types, etc.) is rejected when created.
- Diagnostics name `<repl>:line:col` over the cell's own text; an earlier cell's items and runtime locations name `<repl cell N>:line:col` ([9.7.5](907-jit-cells.md#975-error-modes)).
- On a terminal, input has line editing, Tab completion, colours and whole-cell history ([9.7.8](907-jit-cells.md#line-editing)); the history persists in `$NABI_HOME/repl_history`.
- On a terminal the session starts with the toolchain line and a hint ([9.7.8](907-jit-cells.md#line-editing)); `-q` leaves them out.
- Without a path the session starts empty. A program path (a lone `.nabi` file or a package with `src/main.nabi`) is loaded and run, top-level statements as cells and `fn main` as the entry, and the session continues ([9.7.10](907-jit-cells.md#9710-loading-a-program-repl-path)). A library package path (`src/lib.nabi`) is added as a dependency and `use <name>::*;` is the first cell, again after `:reset`.
- Without a path inside a package, the package is not loaded. A name that is not found but that the package defines gets the hint ``load this package: `nabi repl .` ``.
- More than one path is a usage error. `-` is not accepted; standard input is the cell input.
- `NABI_REPL_TIME=1` prints the time of each phase of a cell on stderr ([9.9.9](909-debug-tools.md#999-repl--jupyter-debugging)).

A package with both root files is a program path; its library is a dependency of the session. `nabi repl <path>` resolves dependencies and lints from the project of that path (9.10.1); outside a package the session uses the active environment's libraries (9.10.5).

REPL commands (`:type`, `:doc`, `:time`, `:vars`, `:load`, `:reload`, `:edit`, `:save`, `:opt`, `:reset`, `:add`, `:help`, `:quit` / `:exit`, and `exit` / `quit` alone) and debugging aids are in [9.9.9](909-debug-tools.md#999-repl--jupyter-debugging).

### `nabi jupyter`: Jupyter kernel

```
nabi jupyter install [--env <name>]
nabi jupyter --connection-file <path>
```

`nabi jupyter install` registers the kernel ([9.8.3](908-jupyter-integration.md#983-kernelspec-installation)); the kernelspec runs `nabi jupyter --connection-file {connection_file}`.

`nabi jupyter --connection-file <path>` is the kernel. Jupyter starts it, so users do not run it directly. `--connection-file` takes the path of the ZMQ connection file provided by Jupyter; bare `nabi jupyter` is a usage error pointing to `nabi jupyter install`. Build options such as `--features` are accepted as well. The kernel runs on the same cell session as the REPL ([9.7.0](907-jit-cells.md#970-rules-of-the-cell-session)) and loads the package it starts in as `nabi repl .` does; once it listens it prints `nabi jupyter: ready` on stderr ([9.8.2](908-jupyter-integration.md#982-local-native-kernel-operation)). Integration details are defined in [Jupyter integration](908-jupyter-integration.md).

### `nabi lsp`: LSP server

```
nabi lsp
```

`nabi lsp` runs the Language Server Protocol server on standard input and output. It is invoked by LSP clients such as VS Code, Vim and Emacs. The standalone editor binary `nabils` is the same server. `.nabion` buffers are checked for syntax only. In a package with both root files, a file the library's module tree reaches is checked as part of the library, any other as part of the executable, which sees the library's open buffers. The features are listed in [9.9.13](909-debug-tools.md#9913-lsp-server); checks run in the background and queries answer from the last good check meanwhile.

### `nabi fmt`: formatter

```
nabi fmt [path..] [--check] [--diff]
```

`[path]` accepts files or directories (several allowed) and rewrites them in place to the [standard format](#standard-format). A directory that is a package ([9.2.2](902-aot-pipeline.md#922-package-discovery)) is formatted as its `src/`, `tests/`, `benches/` and `examples/` directories and its `nabi.nabion`; other files in it are left alone. Without a path, `fmt` formats the package the current directory is in: the nearest directory at or above it with `src/main.nabi`, `src/lib.nabi` or `nabi.nabion`. Outside a package, no path is an error (exit code `1`): `` not inside a package; pass a file or directory: `nabi fmt <path>` ``. Directories are searched recursively for `.nabi` and `.nabion` files, skipping `target/`, `node_modules/`, directories beginning with `.`, and what the `.gitignore` at the searched directory or at its package's root ignores. Only these `.gitignore` lines are read: `name` (that name anywhere below), `name/` (directories only), `/name` (directly in the `.gitignore`'s directory) and `*suffix`; comments, `!` lines, other wildcards and paths with `/` inside are skipped. On a terminal, a search that takes more than a second shows `Scanning <n> files…` on standard error. `.nabion` files ([NABION](../../refs-standard-libraries/300-std/308-nabion.md#384-canonical-form)) are checked for being NABION and only re-indented (they are not rewritten in canonical form, so comments are preserved).

At the end, unless `-q`, standard error says what happened: `Formatted <n> files` (with `, <m> already formatted` when some were), `note: <n> files already formatted`, or, with `--check` / `--diff`, `` note: <n> files need formatting; `nabi fmt` without `--check` rewrites them ``. A directory with nothing to format gives `note: no .nabi or .nabion files under <dir>` (exit code `0`).

Options:

- `--check` does not modify files; it prints the files that would change and their diff. If any file would change, the exit code is `1`; suitable for CI.
- `--diff` does not modify files and prints only the diff.

A file with lexical errors is left untouched and the error is reported, with exit code `1`.

#### Standard format

The standard format of `nabi fmt` is a **token-preserving layout rule**. It does not change tokens or comments within a line; it sets line indentation and blank lines, and orders the `mod` / `use` items at the top of a module. `nabi fmt` has no configuration by design: there is one standard format for all Nabi code.

1. Indentation is 4 spaces per bracket (`(` `[` `{` `#[`) depth. A line beginning with a closing bracket takes the depth of the line that opened it. Tabs become spaces.
2. A continuation line, i.e. one whose preceding token is not `;` `,` or a bracket (such as a line splitting a long expression), keeps the amount by which it was indented beyond the block indentation in the original file.
3. Trailing whitespace is removed.
4. Consecutive blank lines are collapsed to one, and blank lines at the start of the file are removed.
5. The file ends with exactly one newline.
6. The `mod name;` and `use` items at the top of a file and of an inline `mod name { }` (before its first other item) are grouped in the order `pub mod`, `mod`, `pub use`, `use`, where any `pub(..)` counts as `pub`. Each group is sorted (`mod` by name, `use` by path segment by segment, `self`, `sup` and `pkg` first), groups are separated by one blank line, and there are no blank lines inside a group. The comments, doc comments and attributes directly above an item move with it; a comment above the first item stays where it is. An item that shares its line with another item ends the top.

Apart from rule 6, the formatter does not move lines, does not split or join lines, and does not change spacing within a line. Comments therefore stay exactly as written, and the token sequence is unchanged except for the order of the items rule 6 sorts. The interior of multi-line strings and block comments is also left as is.

This layout rule stays the standard format until 1.0: no pretty-printer that re-formats code (moving tokens, splitting or joining lines) replaces it before then, so a file `nabi fmt --check` accepts keeps passing on later 0.x toolchains.

### `nabi doc`: documentation generation

```
nabi doc [path] [--std] [--open]
```

Generates HTML API documentation in `target/doc/` from the public items and docs (`///`, `//!`, `#[doc]`, [6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---doc-comment-form)) of the package's library (its executable when it has none) and of its dependency packages.

```
target/doc/
├── <pkg>/index.html           # a package's root module
├── <pkg>/<mod>.html           # one page per public module
├── search-index.js            # search index
├── search.js, keys.js         # search, and its `/` shortcut
├── theme.js                   # light / dark theme
├── nabi.css, style.css        # style sheets
└── fonts/, favicon.svg
```

`<pkg>` is the package's name (the manifest's `name`); a standard library tier is its own name (`core`, `std`, `bare`). A module is public when every module on its path is `pub` and not `#[doc(hidden)]`; each public module has one page: the root's is `<pkg>/index.html`, and `a::b`'s is `<pkg>/a/b.html` (a child of the root named `index` is `<pkg>/index/index.html`). There is no page above the packages' root pages: each page's header shows its package's name, linked to that package's root, and version, its sidebar lists the package's dependencies, linked to their root pages, and with its breadcrumb leads to its own package's root, and the search covers every documented package from any page. A `target/doc/index.html` left by an earlier toolchain is removed; no other file in `target/doc/` is. The pages need no network: the style sheets, scripts and fonts are in the output.

A module's page shows its doc (its `//!` lines, after the `///` on its `mod` item; a plain `//` comment is not a doc comment, and `nabi lint` warns about one that opens a library's or a public module's file, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)), its public child modules, then its items by kind: re-exports, structs (unions among them), enums, traits, type aliases, functions, constants, statics, and last the implementations of types declared elsewhere (`impl str { .. }`). The items are the `pub` ones that are not `#[doc(hidden)]` or `#[internal]`, each with its signature as written, its doc, its members (public fields, variants, trait items, the public methods and constants of its inherent impls) and its trait implementations. What a page shows is decided by these rules:

- An item that a `pub use` brings in from a private module, or through private modules only, is shown on the re-exporting page as if declared there, under the name it is re-exported as and with the `pub use`'s doc first. So is every item of a `pub use private::*`.
- A re-export of an item that a public module's page shows stays a re-export entry, which links to that page.
- The methods of an `impl` written in a private module join their type on the page of the nearest public module above it, else that page's implementations.
- Each item has the anchor `#<kind>.<name>`, `<kind>` being `struct` (a union too), `enum`, `trait`, `type`, `fn`, `const` or `static` (`app/index.html#struct.Point`); a member `#<Type>.<member>` (`#Point.x`, `#Shape.Circle`, `#Point.origin`); an implementation `#impl.<Type>`; a re-export `#reexport.<names>`, the names it brings in joined by `.`, and each name in it `#<kind>.<name>` of the item it names, so `#struct.Event` finds `Event` on the page that re-exports it.
- The types in signatures, and names in inline code that name a documented item, link to the page that shows it, in the package or in a dependency; a name no page shows stays plain text. Every link points at a page and an anchor that exist.

The standard library API on docs.nabi-lang.org is made by these rules: the same page per module (`std/core/io.html` there, `core/io.html` here) with the same anchors, and `nabi doc` pages share its style sheets and layout.

`--std` also documents the standard library tiers into `core/`, `std/`, `bare/`; without a path it documents only them, into `./target/doc/`.

`nabi doc` prints the package's root page (`Generated ./target/doc/<pkg>/index.html`); with `--std` and no path it is `std/index.html`. `--open` then opens that page with `$BROWSER` if set, else with the system's opener (`open` on macOS, `xdg-open` on Linux, `start` on Windows). If that cannot start, a warning names the file to open by hand; the exit code is not affected.

Doc text is interpreted as a subset of markdown (headings, fenced code blocks, inline code, links, `**bold**`, lists, paragraphs). In a Nabi code block a line starting with `# ` (or a lone `#`) is not shown: it is setup for the block's doc test ([9.1.2](901-nabic-cli.md#doc-tests)), which `nabi test` runs.

### `nabi clean`: remove build output

```
nabi clean [path]
```

Deletes the package's whole `target/` directory (including all profiles and cross builds; the `--target-dir` / `NABI_TARGET_DIR` directory if set). Does nothing if it does not exist.

### `nabi bindgen`: C header bindings

```
nabi bindgen <header> [-x objective-c] [-o <file>] [--target <t>] [--c-shim <file.c>] [-- <C compiler args>]
```

Writes the declarations of [std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#176-automatic-binding-generation) for `<header>` to standard output (or to `-o <file>`); its first line begins ``// Generated by `nabi bindgen` ``. `nabi` reads the header with the front end of the toolchain's C compiler ([9.13](913-c-compiler.md)); it needs no other C compiler and no libclang. The arguments after `--` are the C compiler options that change what a header declares: `-I <dir>`, `-isystem <dir>`, `-iquote <dir>`, `-idirafter <dir>`, `-D <name>[=<value>]`, `-U <name>`, `-include <file>`, `--sysroot <dir>`, `-isysroot <dir>`, `-nostdinc`, `-nostdlibinc` and `-nobuiltininc`. `-std=` and warning options are accepted and ignored; any other option is an error that lists these. A header the front end rejects is an error (exit 1) at its `file:line:col` that says what to write instead ([std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#the-c-bindgen-reads) lists what it reads).

With `-x objective-c` (Apple targets) the header is read as Objective-C, and its classes, protocols and categories become `extern "objc"` blocks as `ffi::objc::import` makes them ([std 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-headers-ffiobjcimport)); `-x c` is the default.

A `static inline` function has no symbol to link, so it is skipped with a comment. With `--c-shim <file.c>` it is bound instead: `<file.c>` gets a C wrapper for each, `R nabi_shim_<f>(A a0, ..) { return <f>(a0, ..); }` after `#include "<header file name>"`, and the binding links to the wrapper (`#[link_name = "nabi_shim_<f>"]`). The package compiles `<file.c>` from its manifest's `c.sources`, with the header's directory in `c.include` (9.10.3). A variadic inline function stays skipped.

### `nabi inspect`: package inspection

`nabi inspect` gives an overview of an archive's contents.

```
nabi inspect path/to/libfoo.a        # or foo.nabihir
nabi inspect --dump path/to/foo.nabihir   # or libfoo.a; reads the sibling foo.nabihir
```

It shows the package name and hash, nabic version, NABIHIR version, GC type id range, exports, [generic](../000-meta/007-glossary.md#generic) items, and archive members with their sizes ([9.9.14](909-debug-tools.md#9914-package-inspection)).

With `--dump` it prints the NABIHIR of the prebuilt library as text instead: the header fields, producer, meta and interface files; the format is described in [NABIHIR format](904-nabihir-bytecode.md).

### `nabi demangle`: symbol demangle

`nabi demangle` decodes mangled symbols into a human-readable form.

```
nabi demangle [-v] <symbol>...
nabi demangle
```

For each symbol it prints its name as source writes it, one per line; with `-v`, a block in the [9.5.4](905-symbol-mangling.md) format, separated by blank lines ([9.9.3](909-debug-tools.md#993-symbol-demangle)). If any argument is not a Nabi symbol, the exit code is `1`. With no symbol it filters standard input to standard output, replacing every Nabi symbol in the text with its name (`perf report --stdio | nabi demangle`).

### `nabi completions`: shell completion

```
nabi completions bash|zsh|fish|powershell
```

Prints a completion script for that shell on standard output. It is generated from the same command and option text as `nabi help` (and, for the delegated subcommands, `nabic`'s), so the two agree. It completes subcommands and their options, `+<version>` and the argument of `toolchain default` / `toolchain remove` with the installed toolchains, the environment names of `env use` / `remove` / `shell` / `export` and `--env`, and the value of `--target` with the names `nabic --print target-list` prints. The names are read when completing, by running `nabi __complete toolchains` / `envs` (one per line), so they follow `$NABI_HOME`. Any other shell, or no shell, is a usage error (exit code 2). `install.sh` and `install.ps1` install the scripts, and `nabi toolchain default` / `update` rewrite them (9.10.6).
