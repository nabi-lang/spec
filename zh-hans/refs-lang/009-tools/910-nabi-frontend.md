# 9.10. `nabi`：包 / 构建前端与工具

`nabi` 是 cargo 风格的前端。它负责清单 `nabi.nabion` 和锁文件 `nabi.lock`、依赖、环境、工具链固定以及脚本依赖，并把编译交给编译器 `nabic`（[9.1](901-nabic-cli.md)）。各种工具（run、lint、REPL、Jupyter、LSP、格式化工具、文档等）是 `nabi` 自身的一部分（9.10.8）。运行时是 `nabir`（`libnabir.a`）。

没有依赖的包只凭文件约定即可构建，不需要清单（[0.3.4](../000-meta/003-source-units.md#034-清单nabinabion是可选的)）。这时 `nabi build` 等同于 `nabic build`。

## 9.10.1. 子命令与全局选项

| 子命令                                                            | 作用                                    |
|-------------------------------------------------------------------|-----------------------------------------|
| `new`、`init`                                                     | 创建包（9.10.2）                        |
| `add`、`remove`、`update`、`deps`                                 | 依赖（9.10.4）                          |
| `search`、`publish`、`yank`、`login`、`logout`                    | 注册表包（9.10.4a）                     |
| `env`                                                             | 环境（9.10.5）                          |
| `component`                                                       | 可选部分：官方包，供无网络的构建使用（9.10.4a） |
| `toolchain`、`cache`                                              | 工具链及其构建缓存（9.10.6）            |
| `install`、`uninstall`                                            | 供 REPL / Jupyter 使用的库，以及可执行文件（9.10.5a） |
| `build`、`check`、`emit`                                          | 附上依赖后交给 `nabic`                  |
| `test`、`bench`                                                   | `nabic` 附上依赖构建测试框架（harness），`nabi` 运行它（[9.1](901-nabic-cli.md)） |
| `run`、`dev`、`lint`、`fix`、`repl [path]`、`jupyter`、`lsp`、`doc` | 工具，附上依赖（9.10.8）              |
| `fmt`、`clean`、`bindgen`、`inspect`、`demangle`、`completions <shell>` | 工具，不附依赖（9.10.8）          |
| `cc`                                                              | C 编译器 `nabicc`，实参原样传递（[9.13](913-c-compiler.md)） |
| `version`、`help`                                                 | 版本；帮助（见下文）                    |
| （无）                                                            | 同 `nabi repl`：启动 REPL               |

委托的子命令运行 `nabic` 中同名的子命令；工具则在 `nabi` 自己的进程中运行，编译器链接在其中（9.10.8）。两种情况下，`nabi` 只消耗它自己的全局选项（`--env` / `--offline` / `--locked` / `--frozen` / `--cwd` / `--color`，位于 `--` 之前的任意位置），把其余实参原样传给编译器，并添加以下内容。对 `test` 和 `bench`，`nabi` 让 `nabic` 构建测试框架（`--emit-harness`）而不运行它，然后由自己运行测试框架（[9.1](901-nabic-cli.md)）；运行器不属于编译器，它的选项（见下文）是 `nabi` 的选项。

- 每个依赖包对应一个 `--extern name=<source directory>`。它们按照 `nabi.lock` 准备，依赖排在依赖它的包之前（即 `nabi deps` 打印的顺序）。预构建由 `nabic` 完成（[9.2.3](902-aot-pipeline.md#预构建的依赖包)）。
- 每个包（`pkg` 是正在构建的包）的每个依赖对应一个 `--extern-dep <package>=<name>`，使每个包只看到自己的依赖（[5.7.14](../005-modules/507-paths.md#5714-包之间的路径)）。
- 包自身的特性以 `--cfg feature=<f>` 传递，每个依赖的特性以 `--extern-feature <name>=<f>` 传递（9.10.4c）。build / run / test / bench / check / lint / fix / doc / emit / repl / lsp / jupyter / deps 的特性选项 `--features <f,..>`（可重复；也可写 `<dep>/<f>`）、`--all-features` 和 `--no-default-features` 由 `nabi` 消耗（位于 `--` 之前）。
- 清单的 `lints` 以 `-A` / `-W` / `-D` 传递（位于实参之前）。所用清单是路径所选项目的清单：对 `.nabi` 文件，是该文件所在目录或其上级中最近的、含有 `src/main.nabi`、`src/lib.nabi` 或 `nabi.nabion` 的目录。包的 `src/` 中的文件选中该包（[9.1.1](901-nabic-cli.md#911-一般形式)）。位于含有 `nabi.nabion` 的目录中其他位置的单独文件（包的 `src/` 旁边的脚本，或只有 `nabi.nabion` 的目录）按单文件包编译，使用该清单的 `lints`（以及 `lints_from`）和依赖。
- 在包之外（环境、脚本），`--target-dir` 设为该环境或脚本的 `target/`。

`nabi cc` 既不是委托的子命令，也不是工具：它运行工具链的 `nabicc`，把 `cc` 之后的每个实参原样传递，不添加任何内容（[9.13.1](913-c-compiler.md#9131-nabicc-与-nabi-cc)）。

`nabi repl <path>` 与 `run` 一样，从该路径所在的项目解析依赖和 lint，并使用清单中的名称（`--name`）。

`nabi test` 的运行器接受以下选项（位于 `--` 之前），这些选项不传给 `nabic`：

- `--threads=N` 设置并发运行的测试进程数。默认值为 `NABI_TEST_THREADS`，未设置时为 CPU 数；`NABI_TEST_THREADS` 的值不是正数时发出警告（`nabi: warning: ignoring NABI_TEST_THREADS=0 (expected a positive number)`）并被忽略。
- `--show-output` 也显示通过的测试的 stdout / stderr。失败测试的输出总是显示。
- `--format=human|json` 选择输出形式（默认 `human`）。`json` 在 stdout 上为每个测试写一条 JSON 记录和一条汇总，在 stderr 上写人类可读的文本（[9.9.12](909-debug-tools.md#9912-ci--测试集成)）。
- `--exact` 使过滤器和 `--skip` 匹配完整的测试名：`nabi test --exact math::adds` 运行该测试，而不运行 `math::adds_wide`。
- `--skip <pat>` 按过滤器的规则排除 `<pat>` 匹配的测试（可重复）。
- `--ignored` 只运行 `#[ignore]` 测试；`--include-ignored` 把它们与其余测试一起运行。
- `--timeout=SECS` 终止运行时间更长的测试，并判定其因超时而失败。默认值为 `NABI_TEST_TIMEOUT`，未设置时为 600；`0` 表示不限时。`NABI_TEST_TIMEOUT` 的值不是整数秒时发出警告并被忽略。
- `--list` 打印过滤器保留的测试名，每行一个，不运行任何测试（退出码 `0`）；`#[ignore]` 测试后面跟 ` (ignored)`。使用 `--format=json` 时，每个名称改为一条记录：`{"schema":1,"type":"listed","kind":"test","name":"adds"}`，被忽略的测试带 `"ignored":true`。

`--doc` 和 `--no-doc` 传给 `nabic`：只运行文档测试，或完全不运行文档测试（[9.1.2](901-nabic-cli.md#文档测试)）。

`nabi bench` 的运行器接受以下选项：

- `--iters=N` 固定每个样本的调用次数，不做校准。
- `--samples=N` 是测量的样本数（默认 10）。
- `--format=human|json|nabibench` 选择输出形式（默认 `human`）：`json` 与测试相同，`nabibench` 是 [9.1.2](901-nabic-cli.md#nabic-bench基准测试) 中的稳定行格式。
- `--list` 与测试一样，打印过滤器保留的基准测试名，不运行任何基准测试。

两者都通过另一个 `--target` 的运行器运行该目标的可执行文件（[其他目标的可执行文件](#其他目标的可执行文件)）。

无效的值是用法错误，错误消息会指出应当怎么写：`` `--threads` takes a positive number, not `0` ``、`` `--timeout` takes whole seconds (0: no limit), not `1.5`: `--timeout=120` ``、`` `--format` takes human or json, not `xml` ``。运行器打印的内容在 `nabic test` 和 `nabic bench` 中说明（[9.1.2](901-nabic-cli.md#nabic-test测试)）。

选项、输出和退出码遵循 `nabic` 的规定（[9.1](901-nabic-cli.md)）。`nabi` 对已关闭的标准输出或标准错误的处理方式与 `nabic` 相同（[9.1.6](901-nabic-cli.md#916-退出码)）。`nabils` 是一个独立的可执行文件，等同于 `nabi lsp`。

`nabi help`（也可写 `nabi --help`）打印一个简短的页面：分组列出的子命令（每个一行）以及全局选项。`nabi help <subcommand>`（也可写 `nabi <subcommand> --help`）打印该子命令自己的帮助：用法形式、说明、示例及其选项。对委托的子命令，帮助是以 `nabi` 写出的 `nabic` 帮助，加上 `nabi` 的特性选项；对 `test` 和 `bench`，还加上运行器的选项。

未知的子命令或选项是用法错误（退出码 2）。存在最接近的子命令或选项时，消息会指出它，包括常见的别名（`compile` 对应 `build`，`rm` 对应 `remove`），消息后面跟着 ``run `nabi help <subcommand>` for usage``。索引中没有的包或原生包，会指出索引中最接近的那一个以及应当运行的 `nabi search`。`nabi` 为它运行的 `nabic` 设置 `__NABI_CLI=nabi`，因此 `nabic` 的用法错误、帮助以及错误和警告行的前缀也会写作 `nabi`（[9.1.7](901-nabic-cli.md#917-环境变量)）。

只运行 `nabi [options]` 而不带子命令时，启动 REPL，与 `nabi repl` 相同。

```
nabi                 # = nabi repl
nabi --env sci       # = nabi repl --env sci
```

`nabi [options] <file> [args..]` 运行脚本：它就是 `nabi run -q <file> -- args..`。文件之后的每个实参原样传给程序，退出码就是程序的退出码。一个词含有 `/`（在 Windows 上还有 `\`）或扩展名为 `.nabi` 时就是文件；没有子命令满足这一条件，所以子命令永远不会被当作文件。配合 shebang 行（[1.2.3](../001-lexical/102-lexical-structure.md#shebang-行)），`./script.nabi args` 即可运行脚本。

委托命令或工具的路径是它的第一个既不是选项、也不是选项值的实参，因此选项可以写在路径之前（`nabi run --release t.nabi`）。不存在的路径是错误 `` `<path>` does not exist ``，先于其他任何检查报告（而不是“不在包内”）；对 `test` 和 `bench`，第一个词可能是过滤器，只有以 `.nabi` 结尾的词才被当作路径。

`nabi` 的全局选项如下：

| 选项             | 含义                                                              |
|------------------|-------------------------------------------------------------------|
| `--env <name>`   | 在该具名环境中工作（REPL、Jupyter、`nabi install`）；用于针对包的命令时是错误（[9.10.5](#9105-环境)） |
| `--offline`      | 不使用网络。git 依赖必须已在缓存中                                |
| `--locked`       | 如果 `nabi.lock` 需要改变，则不改变它并以失败结束（用于 CI）      |
| `--frozen`       | 同时使用 `--locked` 和 `--offline`                                |
| `--cwd <dir>`    | 运行前切换目录                                                    |
| `-q`、`--quiet`  | 减少输出                                                          |
| `-v`、`--verbose` | 更多输出（也传给 `nabic`）                                       |
| `--color <when>`、`--no-color` | `auto`、`always` 或 `never`：颜色，判定方式与 `nabic` 相同（[9.1.5](901-nabic-cli.md#915-全局选项)）；以 `--color=<when>` 传给它运行的 `nabic` |

全局选项也可以写在子命令之后：`nabi new app -q` 就是 `nabi -q new app`。

环境变量有 `NABI_HOME`（默认 `~/.nabi`）、`NABI_ENV`（9.10.5）、`NABI_TOOLCHAIN`（9.10.6）、`NABI_REGISTRY_TOKEN`（默认注册表的 API 令牌，9.10.4a）和 `NABI_DIST`（发布服务器，9.10.6）。`NO_COLOR`、`CLICOLOR_FORCE`、`COLORTERM` 和 `TERM` 设置颜色的方式与 `nabic` 相同（[9.1.7](901-nabic-cli.md#917-环境变量)）。

## 9.10.2. `nabi new` / `nabi init`：创建包

```
nabi new [--lib] [--bin] <name>
nabi init [--lib] [--bin] [dir]
nabi init --script <file.nabi>
```

`nabi new` 创建目录 `<name>` 并填入默认文件。`--bin`（默认）创建打印问候语的 `src/main.nabi`；`--lib` 创建带有一个示例函数和一个 `#[test]` 的 `src/lib.nabi`；两者同时使用时两个文件都创建，这时 `src/main.nabi` 通过 `use <name>::add;` 调用库中的函数（[0.3.1](../000-meta/003-source-units.md#031-可执行文件与库)）。`<name>` 已存在时是错误。

`nabi init` 把一个已有的目录（省略时为当前目录）变成包。该目录已含有 `src/main.nabi` 或 `src/lib.nabi` 时是错误。

`nabi init --script <file.nabi>` 把脚本（[9.10.7](#9107-单文件脚本的内联清单)）变成位于脚本所在目录的包，包名取自文件名（去掉 `.nabi`）。`nabi.nabion` 获得脚本各块合并后的 `deps`（以及 `native`），路径改写为相对于它的路径；脚本成为 `src/main.nabi`，它的 `mod` 加载的每个文件都按相同的相对路径移到 `src/` 下，并各自去掉其块。脚本位于包内、某个要移动的文件或 `nabi.nabion` 已存在，或者某个子模块文件位于脚本目录之外时，是错误。

两个命令还会以规范形式（[NABION 3.8.4](../../refs-standard-libraries/300-std/308-nabion.md#384-规范形式)）写出 `nabi.nabion`，其中 `nabi_version` 是写出它的 `nabi` 的版本（`nabi_version: "0.1.3"`，省略预发布部分），并固定到该 `nabi` 的系列（任何 0.1.x 都写 `toolchain: "0.1"`，9.10.6），使包在更新的系列成为默认后仍停留在该系列；还会写出内容为 `target/` 的 `.gitignore`，并在不位于 git 仓库内时运行 `git init`。`init` 使用按命名规则（9.10.3）调整后的目录名作为包名：Windows 设备名加上前缀 `pkg-`（`nul` → `pkg-nul`），名称截断为 64 个字符。

在 `Created` 行之后，两个命令都在消息下方打印接下来要运行的命令：`cd <dir> && nabi run`，只有库的包则为 `nabi test`；包就是当前目录时不带 `cd`。在 Windows 上，由于 Windows PowerShell 没有 `&&`，`cd <dir>` 和该命令分为两行。`-q` 时两者都不打印。

```
     Created executable package (hello)
             cd hello && nabi run
```

在 Windows 上：

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

包根目录下的 `nabi.nabion` 是清单，同一位置的 `nabi.lock` 是锁文件。两者都采用 [NABION](../../refs-standard-libraries/300-std/308-nabion.md) 格式。文件名必须完全按此拼写（全部小写）。仅大小写不同的文件（`Nabi.nabion` 等）会被忽略并发出警告，不会被读取。

清单是单个 NABION 值，即一个 `Package { .. }` 结构体（根名称 `Package` 可以省略）。它只被读取，从不被编译或执行。

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

- 值是 NABION 字面量（字符串、`{ "k": v }` 字典、空字典 `{}`）以及 `Dep::path("..")` / `Dep::git("url", rev | tag | branch: "..")` / `Dep::version("req", registry: "url")`。`Dep::git` 最多接受 `rev` / `tag` / `branch` 中的一个，`Dep::version` 的 `registry` 是可选的。只有一个字符串的依赖值是 `Dep::version` 的简写。
- `Dep::path("..", version: "req")` 为路径依赖添加版本要求。构建使用该路径，而该路径下的包必须具有满足要求的版本（`` dependency `linalg` (path ..) is version 0.2.0, which its `version: "^0.1"` does not accept; .. ``）；`nabi publish` 会把要求写到路径的位置（9.10.4a）。并行开发的多个包就是这样发布的。
- `Dep::path` / `Dep::git` / `Dep::version` 还接受 `optional: true`、`features: ["<f>", ..]` 和 `default_features: false`（9.10.4c）；可选的开发依赖是错误。
- `package: "<name>"` 为依赖改名：`<name>` 是包本身，键是本包代码中对它的称呼（`"j": Dep::version("^1", package: "json5")` 就是 `json5`，以 `use j::..` 使用）。每个依赖方自选名称，且只有它自己的代码能看到这些名称，因此两个包可以用不同的名称称呼同一个包；它仍然是同一个包，只构建一次，它的类型在每个名称下都相同。一个依赖方以两个键依赖同一个包是错误（`` the root package depends on package `json5` twice, as `j` and `json5`; depend on it once ``）。
- `target: "<target>"` 把依赖限定于针对匹配目标的构建：目标名称（`x86_64-unknown-linux-gnu`），或者基于 [6.3.2](../006-meta-system/603-cfg.md#632-谓词种类) 中目标键（`target_os`、`target_arch`、`target_family`、`target_env`、`target_endian`、`target_pointer_width`）并可用 `all(..)`、`any(..)`、`not(..)` 组合的 `cfg(<predicate>)`。其他任何形式在读取清单时都是错误。解析和锁文件对所有目标都覆盖该依赖；针对其他目标的构建会去掉它，以及它本会启用的特性。
- 依赖上的 `plugin: true` 把它构建为程序旁边的插件，而不链接它（[9.12.2](912-plugins.md#9122-构建)）；插件的依赖上的 `bundle: true` 把该依赖链接进插件，而不是从程序中获取。在不是插件的包中使用 `bundle` 是错误（`` `bundle: true` on `noise` belongs in a plugin's dependencies: give this package `lib: { kind: "plugin" }`, or remove `bundle: true` ``），两者同时出现在一个依赖上也是错误。
- `Dep::registry` 只用于 `nabi.lock`；在 `nabi.nabion` 中使用它是错误。
- 包名（`deps` 的键、`name`）是以字母开头的小写 ASCII `[a-z0-9_-]+`（以数字、`-` 或 `_` 开头是错误）。名称 `std`、`core` 和 `bare` 是随工具链提供的标准库层级：包、原生包或注册表名称取其中之一是错误（`` `std` is the standard library, which ships with the toolchain; name the package something else, e.g. `std_util` ``），以它为名执行 `nabi new` / `nabi init` 也是错误；`nabi add std` 会提示没有可添加的内容。`deps` 的键取其中之一是错误，其帮助信息为 `` std ships with the toolchain; remove this line ``，除非它带有 `package: "<name>"`：这时它是以层级名称出现的那个包（[5.7.4](../005-modules/507-paths.md#574-stdcorebare标准库层级)）。`package: "std"` 是同样的错误。代码也必须能够称呼这个包，因此它的代码名（见下文）不得是关键字（1.5：`fn`、`match`、`self` 等）或路径起始（5.7.1：`pkg`、`sup`）；在可执行文件中 `use pkg::add;` 总是指它自己的根（`` `pkg` already names a package's own root (`pkg::`), so code can't name a package called that; name the package something else, e.g. `pkg_util` ``）。键是字符串（`"nabi-syntax": Dep::version("^0.1")`）。
- 包名还必须能被每个受支持的操作系统存储为文件或目录：不能是 Windows 设备名（`con`、`prn`、`aux`、`nul`、`com1`-`com9`、`lpt1`-`lpt9`，大小写不限），且最多 64 个字符（`` `nul` is a device name on Windows, so no file can be called that; name the package something else, e.g. `my_nul` ``）。`nabi new` / `nabi init --script` / `nabi add` / `nabi publish` 拒绝这样的名称。在被读取的 `nabi.nabion` 中（它的 `name` 以及 `deps`、`dev_deps`、`native` 的键），这是带 `file:line:col` 的警告，构建继续进行。
- `version` 是一个版本号（`major.minor.patch`，9.10.4a）。不是版本号的值在读取清单时是警告（`` `banana` is not a version (major.minor.patch): it needs three numbers; `nabi publish` needs one, e.g. `version: "0.1.0"` ``），在 `nabi publish` 时是错误。
- `nabi_version` 是该包可用于构建的最旧工具链，是完整的版本号：`x.y.z`，不多也不少（读取清单时报告 `` `nabi_version: "0.1"` is not a toolchain version; write all three parts, e.g. `nabi_version: "0.1.3"` ``）。它接纳该工具链及其系列中更新的工具链（[9.10.6 兼容性](#兼容性)）：`"0.1.3"` 接纳 0.1.3 直到最后一个 0.1.x，`"1.4.2"` 接纳 1.4.2 直到最后一个 1.x。`nabi publish` 需要它（9.10.4a），解析也据此挑选版本。它不同于 `toolchain`，后者固定本项目构建所用的工具链（9.10.6）。
- 在代码中，把名称中的 `-` 换成 `_` 后得到的名称，就是指向该包的顶层路径（`nabi-syntax` → `use nabi_syntax::..`，`--extern nabi_syntax=..`）。
- 在注册表中以及同一个依赖图内，比较名称时 `-` 与 `_` 视为相同，且不区分大小写。因此 `nabi-syntax` 和 `nabi_syntax` 不能共存。
- 依赖必须有 `src/lib.nabi`；无论该包是否还有 `src/main.nabi`，依赖方得到的都是这个库。只有 `src/main.nabi` 的依赖是错误，错误消息会说明它是一个程序，应当用于 `nabi install`。传递依赖是依赖包的 `deps`（`dev_deps` 不传递）。
- `native` 把原生包名（它们有自己的命名空间：包和原生包可以同名）映射到 `Dep::native("req"[, registry: "url"])`、要求字符串，或 `Dep::native(url: "<url or path relative to the manifest>", sha256: "<64 hex digits>")`（9.10.4b）。它还把名称映射到 `Dep::system(..)`（机器上已安装的库）和 `Dep::framework()`（Apple 框架），这两者在构建时查找（9.10.4b）。在 `deps` / `dev_deps` 中使用 `Dep::native` 是错误。图中每个包的原生依赖都链接进可执行文件（只经由 `dev_deps` 到达的，只用于 test / bench）。
- `lib: { kind: "nabi" | "cdylib" | "staticlib" | "plugin" }`（该记录可以命名为 `Lib`）说明库构建成什么。默认值 `"nabi"` 是供其他 Nabi 包使用的库；使用 `"cdylib"` 时，`nabi build` 传递 `--crate-type=cdylib` 以及带包的 `version` 的 `--lib-version`（[9.1.2](901-nabic-cli.md#nabic-buildaot-构建)），因此构建还会生成共享库及其 C 头文件（[9.3.3](903-package-archive.md#933-静态库与共享库)），除非命令行给出了 `--crate-type`。使用 `"staticlib"` 时传递 `--crate-type=staticlib`：构建还会生成一个 C 和 Swift 可以单独链接的静态库，以及它的头文件和模块映射（9.3.3）；在它旁边写 `xcframework: true` 还会传递 `--xcframework`，用于其他种类时是错误。使用 `"plugin"` 时传递 `--crate-type=plugin`：该包是由 Nabi 程序加载的共享库（[9.12](912-plugins.md)）；带有 `src/main.nabi` 的 `"plugin"` 包是错误，对它运行 `nabi run` 也是错误。其他种类是错误，错误消息会列出这四种。
- `c: { sources, include, define, std, flags }`（该记录可以命名为 `C`）是由 nabicc（Nabi 自己的 C 编译器，从不使用系统编译器）构建进包的 C 文件：`sources`（相对于清单的路径，至少一个）、`include`（头文件目录，其后是该包的 `Dep::system` 库的目录）、`define`（`{ "NAME": "value" }`）、`std`（`c89`、`c99`、`c11`、`c17`、`c23` 或某个 `gnu` 版本）以及 `flags`，后者只接受 `-O*`、`-g`、`-W*`、`-f*`（`-flto` 除外）和 `-m*`：其他任何内容都是错误，错误消息会列出它们。目标文件放到 `target/<profile>/c/<name>/` 并参与链接（`--native-object`）。只有输入（源文件本身及其读取的头文件）或选项发生变化的源文件才会重新编译（[9.13.8](913-c-compiler.md#9138-包的-c-源文件)）。
- `lints` 设置 lint 级别（lint 或组名 → `"allow"` | `"warn"` | `"deny"`，[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）。它们被视为前置于 `nabi` 运行的每个编译命令（委托给 `nabic` 的，或工具）的 CLI 标志之前，因此 CLI 的 `-A/-W/-D` 和代码中的属性会覆盖它们。
- `lints_from` 指定一个目录（相对于本清单所在目录，或绝对路径），由该目录的 `nabi.nabion` 提供初始 lint 级别。先应用那个清单的 `lints`（在它自己的 `lints_from` 链之后），再应用本清单的 `lints`，因此这里的条目会覆盖链中同一 lint 或组的设置（所有条目按此顺序以 `-A`/`-W`/`-D` 传递；后面的标志优先）。它只作用于正在构建或检查的包（以及环境的清单）；从不为 lint 查阅依赖的清单。该目录不存在、不含 `nabi.nabion`，或者链回到其中已有的清单（形成环）时，是错误。`nabi env import` 把相对的 `lints_from` 改写为绝对路径。脚本的 `//! ```nabion` 块不得使用它。
- `profile` 把构建配置（`"debug"` 或 `"release"`，[9.2.15](902-aot-pipeline.md#9215-输出)）映射到其设置 `{ overflow_checks: bool }`（该记录可以命名为 `Profile`）。`overflow_checks: false` 在该构建配置中关闭整数溢出检查，与 `-C overflow-checks=no` 的效果相同（[9.1.2](901-nabic-cli.md#nabic-buildaot-构建)）；命令行上的 `-C overflow-checks` 会覆盖它。其他构建配置名称或设置是错误。只读取正在构建的包（或环境）的配置；不读取依赖的构建配置，脚本的 `//! ```nabion` 块也不得使用它。
- 包元数据：`authors` 是 `"Name <email>"` 字符串，`nabi new` / `nabi init` 不会填写它们。`keywords` 和 `categories` 各最多 5 项。`authors`、`keywords`、`categories`、`include` 或 `exclude` 中的空条目是错误。`readme` 和 `license_file` 是包内的路径，相对于清单所在目录（绝对路径，或经 `..` 离开包的路径，是错误）；没有 `readme` 时，若存在 `README.md`，则以它为 readme。`metadata` 是 `nabi` 不读取的表（`{ .. }`）；它原样发送到注册表。`nabi publish` 检查 readme 和 `license_file` 是否存在、`default_run` 是否指向包中的某个可执行文件，并拒绝带有 `publish: false` 的包（9.10.4a）。
- `include` / `exclude` 是作用于所发布提交中文件路径的 glob（`*` 和 `?` 在一个名称内匹配，`**` 跨越多个名称；不含 `/` 的模式匹配任意深度的名称，匹配目录的模式也匹配目录中的内容）。当 `include` 为空或匹配某个文件、且 `exclude` 不匹配它时，包中包含该文件；`nabi.nabion` 总是包含在内。它们决定 `nabi publish` 发送的打包大小（9.10.4a）；源代码仍是整个提交。
- `links` 指定包链接的原生库。同一构建中两个包具有相同的 `links` 是错误，错误消息会指出这两个包。
- `default_run` 指定 `nabi run` 运行的可执行文件；一个包最多有一个可执行文件（`src/main.nabi`，以包名命名），因此就是这个名称。
- `deployment_target` 是包中程序可运行的最旧 macOS，以数字表示（`"13.0"`；其他形式是错误，错误消息会给出正确形式）；省略时为 `11.0`。记录形式 `{ macos: "13.0", ios: "16.0" }` 同时给出最旧的 iOS（省略时为 `13.4`；更旧的版本是错误），每个字段都是可选的。`nabi` 把它们作为 `MACOSX_DEPLOYMENT_TARGET` 和 `IPHONEOS_DEPLOYMENT_TARGET` 传给 `nabic`，除非其中某个已经设置，这时已设置的值优先。比它更新的 Objective-C API 只在 `objc::available` 下调用（[标准库 1.10.13](../../refs-standard-libraries/100-bare/110-objc.md#11013-可用性与部署目标)）。
- `nabi` 不认识的字段是带 `file:line:col` 的警告（`` unknown field `licence` is ignored; did you mean `license`? ``，存在相近字段时附带建议），读取清单时忽略该字段；`nabi publish` 把它原样放在 `meta` 中发送（9.10.4a），以便更新的注册表可以使用它。
- 错误：未知的 `Dep` 种类 / 具名参数、重复项、违反 9.10.4c 规则的 `features`、违反规则的包名、不是 NABION 的语法、单个 `Package { .. }` 以外的任何内容、尾随的词法单元。诊断以 `file:line:col` 和插入符号显示位置。

| 依赖来源          | 形式                                                         | 是否在 `nabi.lock` 中固定 |
|-------------------|--------------------------------------------------------------|----------------------|
| 路径        | `Dep::path(".."[, version: "req"])`（相对于清单的路径） | 否（总是使用当前内容） |
| git         | `Dep::git("url")`，可选 `rev:` / `tag:` / `branch:` 之一 | 提交和树哈希 |
| 注册表      | `Dep::version("req"[, registry: "url"])` 或字符串 `"req"`（[9.10.4a](#9104a-注册表包)） | 版本、提交、树哈希（与索引一起，从不使用镜像） |
| 原生        | `Dep::native("req")` / `Dep::native(url:, sha256:)`（[9.10.4b](#9104b-原生包)） | 版本和条目 SHA-256（索引）；无（由哈希固定） |
| 系统        | `Dep::system(..)` / `Dep::framework()`（[9.10.4b](#9104b-原生包)） | 否（每次构建时查找） |

Nabi 没有构建脚本：包中的代码不会在构建时运行。C 库以预构建的原生包、机器上已安装的库，或由 nabicc 编译的 C 源文件（`c`，[9.13.8](913-c-compiler.md#9138-包的-c-源文件)）的形式引入，全部在清单中声明（9.10.4b）。

在同一次构建中，一个名称就是一个包。如果同一名称被要求来自不同来源（不同的路径，或不同的 url / 提交），则是错误，错误消息会显示两个要求方。两个依赖方把同一个代码名给了不同的包也是错误（`` `hello` is package `other` for the root package and package `greet` for `mid` ``）。一个包可以有多个代码名（`package`）；`nabi` 把每个代码名都以相同路径作为 `--extern` 传递。

### 工作区

一起开发的包可以共享一个锁文件和一个构建目录。**工作区**的根目录中有一个 `nabi.nabion`，其值是 `Workspace { .. }` 而不是包：

```nabion
Workspace {
    members: ["app", "libs/*"],   // directories under the root; `dir/*`: each package in `dir`
}
```

- `members` 列出相对于根目录的目录；`dir/*` 代表 `dir` 中每个含有 `nabi.nabion` 的直接子目录。绝对路径、`..` 步骤或其他 `*` 是错误（`` `../x` is not a member: write a directory under the workspace root, or `dir/*` for each package in `dir` ``），没有 `nabi.nabion` 的成员目录也是错误（`` member `gone` has no nabi.nabion (..); create the package with `nabi new gone`, or remove it from `members` ``）。每个成员都需要 `name`；两个成员同名是错误。`Workspace` 不接受其他字段。
- 当其目录上方的工作区根目录列出某个包时，该包是成员；根目录下未被列出的包保持独立。
- 成员共享根目录下的 `nabi.lock`：它们的依赖（包括开发依赖）一起解析，因此每个成员都针对相同的版本构建，成员自己的 `nabi.lock` 不会被读取。它们还共享根目录下的 `target/`（`NABI_TARGET_DIR` 和 `--target-dir` 仍然优先）。成员之间用 `Dep::path` 相互依赖。
- `build`、`check`、`test`、`bench`、`lint`、`fmt`、`doc`、`clean` 和 `run` 接受 `-p <name>` / `--package <name>`（可重复：这些成员）和 `--workspace`（所有成员）。在根目录处（那里没有包），这些命令在每个成员上运行；`run` 需要恰好一个成员（`` `nabi run` runs one package; name it: `nabi run -p <member>` (members: ..) ``）。每个成员在自己的目录中按 `members` 的顺序运行，有多个成员时在其前面打印 `Member <name> (<dir>)`；某个成员失败后其余成员仍会运行，命令以第一个失败的退出码退出，并指出失败的成员。不是成员的名称是错误，错误消息会列出所有成员；在工作区之外使用 `-p` 是错误。
- 在根目录处运行其他命令（`emit`、`add` 等）是错误，错误消息会说明这是工作区根目录，并指出 `-p <member>` 或某个成员的目录；在那里 `nabi update` 和 `nabi deps` 作用于共享的锁文件。语言服务器从共享的锁文件为每个文件解析其所属的包，包括成员。

### `nabi.lock` 格式

`nabi.lock` 是由 `nabi` 确定性地写出的 NABION（规范形式）。在头部注释之后，它为每个 git 或注册表包保存一个 `Locked`，按名称排序。路径依赖不被固定，也不出现在其中。它的字段名与清单一致。

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

| 字段      | 含义                                                                 |
|-----------|----------------------------------------------------------------------|
| `version` | 在 `Lock` 中是锁文件格式版本（`1`）；在 `Locked` 中仅用于注册表包：解析得到的版本 |
| `name`    | 包名（`deps` 的键，改名时为它的 `package`）                          |
| `source`  | 对 git，是与 `nabi.nabion` 中写法完全相同的 `Dep::git(..)`；对注册表包，是 `Dep::registry("<index url>")`：即索引，从不是获取时所用的镜像（9.10.4a） |
| `commit`  | 解析得到的提交（40 位十六进制数字）                                  |
| `tree`    | 该提交的 git 树哈希。用于验证缓存的检出                              |

- `Locked` 的 `source` 是 `Dep::git(..)` 或 `Dep::registry(..)`。
- 缺少 `packages` 字段时读作空列表。
- 不记录特性：锁文件固定图中的每个包（包括可选的包），因此它不依赖于构建启用了哪些特性。
- 来自索引的原生包固定在可选的 `natives: [LockedNative { name, version, source: Dep::registry("<index>"), sha256 }]` 中，按名称排序，仅在非空时写出。`sha256` 是该条目在 `yanked: false` 时规范索引行的 SHA-256；条目发生变化时，是“run `nabi update`”错误。
- `nabi` 写出锁文件格式 1。
- `version` 先于其他字段读取。`version` 大于 `1` 表示锁文件由更新的 `nabi` 写出，这是一个错误，错误消息会指出应当使用的工具链，而不是删除它：``nabi.lock was written by a newer nabi (lock version 2); run that nabi's toolchain here: set `toolchain` in nabi.nabion to its version, or run `nabi +<version> ..` ``。
- `1` 以外的其他任何 `version` 是“unsupported lockfile version”错误。
- 读取锁文件时引发的其他所有错误都会附加“(delete nabi.lock and run `nabi update` to make it again)”。

## 9.10.4. 依赖：`nabi add` / `remove` / `update` / `deps`

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

- `add` / `remove` 编辑包的 `nabi.nabion`，保留注释和布局。`--dev` 添加到 `dev_deps`。最后一个实参是脚本文件时，`add` / `remove` 改为编辑该文件的 `//! nabion` 块（9.10.7）：缺少的块会创建在 shebang 行和开头的 `//!` 行之后（以规范形式写出；已有的块保留其布局和注释），变空的块会被删除，路径写成相对于脚本的形式，`--dev` / `--optional` 是错误。位于包内的脚本是错误（它使用包的 `nabi.nabion`）。在包外且没有脚本，或者使用了 `--env` 时，它们是错误，错误消息会指出 `nabi add <name> <file.nabi>` 和 `nabi install` / `nabi uninstall`（[9.10.5a](#9105a-nabi-install--nabi-uninstall)）。添加包而不带要求时写入 `^<newest version>`。`--features`、`--no-default-features` 和 `--optional` 写入 `Dep::..` 中；`--optional` 还会添加特性 `"<name>": ["dep:<name>"]`，除非该包已有同名特性。
- 对包执行 `add` 时报告 `Added <name> v<version>`，即依赖的 `nabi.nabion` 中的 `version`（对尚未解析的注册表包，为所固定的版本），没有版本时报告 `Added <name>`；`--dev` 会加上 ` (dev_deps)`。
- 构建使用 `nabi.lock` 中的提交和版本。新声明的依赖会加入锁文件，被删除的依赖会被清理，但只有 `update` 会改变已固定的内容。给出名称时，`update` 只重新解析这些包；否则重新解析所有 git 和注册表依赖。
- `deps` 以 `<code name>=<path of its src/lib.nabi>` 行的形式打印解析后的依赖包（代码名中的 `-` 换成 `_`），依赖在前（`--dev` 包含开发依赖）。依赖方以其他名称（`package`）称呼的包，每多一个名称就多一行。每个包的行之后跟着它的 `--extern-feature=<code name>=<f>` 行；`deps` 接受特性选项和 `--dev`。对每个原生包（依赖方在前，针对主机），它还打印 `# native <name> <version>`，后面跟着它的 `nabic` 实参，每行一个。没有依赖时不打印任何内容。
- `deps --format json` 改为按相同顺序每行打印一个 JSON 对象，每个都带有 `schema`（目前为 `1`）和 `type`。`package` 记录有 `name`、`version`（注册表包为解析得到的版本，否则为依赖的 `nabi.nabion` 中的 `version`，再否则为 `null`）、`source`（`path+<dir>`、`git+<url>#<commit>` 或 `registry+<index url>`）、`path`（它的 `src/lib.nabi`）和 `features`。`native` 记录有 `name`、`version` 和 `args`（它的 `nabic` 实参）。最后一条 `summary` 记录统计 `packages` 和 `natives`，因此没有依赖的包只打印这一条。记录遵循 JSON 诊断的 `schema` 规则（[9.9.4](909-debug-tools.md#994-诊断--错误消息)）。`--format human` 是默认值；其他值是 CLI 错误。
- `nabi.lock` 应提交到版本控制中。
- `update` 为锁文件中的每处变化打印一行 `Updated <name> v0.1.0 -> v0.1.1`（git 依赖以其短提交表示；新固定的为 `Locked`，被删除的为 `Removed`），或者打印 `note: all dependencies are up to date`；没有依赖时打印 `` note: no dependencies to update; add one: `nabi add <name>` ``。在包外且没有 `--env` 时，如果包外使用的环境（9.10.5）有库，它就更新该环境（有变化时以 `` Updated environment `<name>` `` 结尾）；否则是错误：`` not inside a package (..): make one with `nabi init`; for an environment's libraries: `nabi update --env <name>` ``。
- `nabi update` 重新解析 `nabi.lock`；`nabi toolchain update` 替换工具链（9.10.6）。

### 缓存

获取的 git 源代码保存在 `$NABI_HOME/cache/` 中，由所有项目共享。

| 路径                                       | 内容                                                    |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | 仓库的裸克隆（每个 url 一个）                           |
| `cache/registry/db/<repo>-<hash>/`         | 注册表索引的裸克隆（每个索引一个）                      |
| `cache/registry/src/<sha256>/`             | 注册表包的文件，来自注册表中具有该 SHA-256 的压缩包（9.10.4a）；不会再次获取 |
| `cache/git/checkouts/<commit>/`            | 某个提交的检出。使用前按树哈希验证，不一致时重新检出    |
| `cache/scripts/<hash>/`                    | 脚本的 `nabi.lock` 和 `target/`；`<hash>` 涵盖脚本的路径，以及它所到达的每个文件的路径和块（[9.10.7](#9107-单文件脚本的内联清单)） |
| `cache/native/<sha256>/`                   | 原生压缩包，在其 SHA-256 匹配后解压；不会再次获取（9.10.4b） |
| `cache/native/meta/<sha256>.nabion`         | 一个 `Dep::native(url:)` 文件                           |

## 9.10.4a. 注册表包

发布到注册表的包是**注册表包**。默认注册表是 `https://packages.nabi-lang.org`，其索引是独立的 git 仓库 `https://github.com/nabi-lang/packages-index`。

### 版本要求

与 Cargo 相同。

| 要求                     | 含义                                            |
|--------------------------|-------------------------------------------------|
| `1.2.3`、`^1.2.3`        | `>=1.2.3, <2.0.0`（`^0.2.3` 是 `<0.3.0`，`^0.0.3` 是 `<0.0.4`） |
| `~1.2.3`                 | `>=1.2.3, <1.3.0`                               |
| `=1.2.3`                 | 精确匹配                                        |
| `>=` `>` `<` `<=`        | 比较                                            |
| `*`、`1.*`、`1.2.x`      | 通配符                                          |
| `>=1.2, <1.5`            | 逗号表示必须全部满足                            |

预发布版本（`1.3.0-beta.1`）只在要求指定了同一 major.minor.patch 的预发布版本时才匹配。

### 使用哪个索引

不带 `registry:` 的要求（裸要求）总是指默认注册表的索引 `https://github.com/nabi-lang/packages-index`。`Dep::version` 的 `registry:` 和 `nabi add --registry` 为该依赖单独设置索引；`nabi publish`、`yank`、`login` 和 `logout` 接受 `--registry <url>` 以使用其他索引。

`$NABI_HOME/sources.nabion` 替换获取索引的位置（源替换）：

```nabion
Sources { replace: { "https://github.com/nabi-lang/packages-index": "https://mirror.example.com/packages-index" } }
```

- 每个键是一个索引 url（忽略末尾的 `/`），其值是内容相同的镜像的 git url。`nabi` 从镜像获取索引；其余一切，包括锁文件（9.10.3）、令牌和消息，都指称索引本身。文件不存在时不进行替换；`nabi` 不认识的字段被跳过。

### 索引格式

索引仓库的根目录有一个 `config.nabion`，即 `Config { api: "<url>" }`：注册表 API 的基础 URL（默认注册表为 `https://packages.nabi-lang.org`）。它必须是 `https://`，`http://127.0.0.1` 和 `http://localhost` 除外。一个包是一个文件，位于其发布时的小写名称（`-` 保持原样）对应的分片路径上。

| 名称长度    | 路径                     |
|-------------|--------------------------|
| 1           | `1/a`                    |
| 2           | `2/ab`                   |
| 3           | `3/a/abc`                |
| 4 或更多    | `ab/cd/abcd..`           |

文件的每一行是一个版本，采用规范形式的单行 NABION。较旧的版本在前。一行只保存解析所需的内容，使用清单的字段名；描述性字段交给注册表 API（`meta`，见下文）。

```nabion
Package { name: "json", version: "1.4.2", deps: [{ name: "util", req: "^0.3", kind: "normal" }], source: Dep::git("https://example.com/json.git", rev: "0b6d.."), tree: "77e1..", yanked: false, nabi_version: "0.1.0" }
```

| 字段        | 含义 |
|-------------|---------|
| `name`      | 发布时的包名 |
| `version`   | 它的版本 |
| `deps`      | 依赖：`{ name, req, kind, registry?, optional?, features?, default_features?, package?, target? }` |
| `features`  | 可选：特性（9.10.4c） |
| `source`    | `Dep::git("<url>", rev: "<commit>")` |
| `tree`      | 该提交的 git 树哈希 |
| `archive`   | 可选，由注册表写入：注册表为该提交保存的源代码压缩包的 SHA-256（64 位小写十六进制数字） |
| `yanked`    | 是否已撤回 |
| `nabi_version` | 能构建它的最旧工具链，`x.y.z`（9.10.3） |
| `links`     | 可选：它链接的原生库（9.10.3） |
| `official`  | 可选，由注册表写入：对 Nabi 项目发布的包为 `true`（见下文） |

- `deps` 中的 `kind` 是 `"normal"`、`"dev"` 或 `"native"`；`registry` 是可选的。改名的依赖以 `name` 为代码名、以 `package` 为包名；特定于目标的依赖带有与清单相同的 `target`，由客户端在每次构建时求值（9.10.3）。
- 带特性的包在 `deps` 之后有 `features: { "<name>": ["<entry>", ..] }`，按 9.10.4c 检查。`normal` 依赖可以带 `optional: true`、`features: [..]` 和 `default_features: false`（只在不是默认值时写出）；`dev` / `native` 依赖不得是可选的，原生依赖没有特性。`nabi publish` 从 `nabi.nabion` 写出它们；解析从条目中读取它们。检出后，条目的 `features` 必须与该提交中包的 `nabi.nabion` 的特性相同（顺序不限）；否则是错误。
- `native/<shard path>` 保存原生包，每行一个 `Native { .. }`（[9.10.4b](#9104b-原生包)）。
- `nabi_version` 即清单中的同名字段（9.10.3）。`nabi publish` 需要它：缺少时的错误为 `` nabi.nabion has no `nabi_version`, the oldest nabi this package builds with; the registry needs it ``，帮助为 `` add `nabi_version: "0.1.3"` (this nabi), or the oldest one you have built it with ``；不接纳执行发布的 `nabi` 的值是错误，错误消息会指出它接纳的工具链。注册表拒绝没有它或其值不是 `x.y.z` 的行。读取方遵守它（见下文“解析”）；在自己的索引中，没有它的行适用于任何工具链。
- 标准库层级的文件（`3/s/std`、`co/re/core`、`ba/re/bare`）保存一行 `Library { name: "std", official: true }`：这是注册表中随工具链提供的库的条目，归 Nabi 项目所有，没有版本，也没有压缩包。进行解析的读取方跳过它，任何版本都不能以它的名称发布。
- 这是唯一的行格式：每行没有格式版本。
- 读取方跳过索引行中不认识的字段（`Package`、它的 `deps` 条目、`Native`、`Artifact`）。自己的 `Native { .. }` 文件（`Dep::native(url:)`、`nabi publish --native`）仍然拒绝它们。注册表严格检查各行。
- 包位于 `source` 仓库的根目录。`tree` 是该提交的 git 树哈希，在检出后验证。
- 版本发布时，注册表保存该提交的 gzip 压缩 tar 包（`git archive`，每个路径都位于 `<name>-<version>/` 下），在 `<api>/api/v1/packages/<name>/<version>/download` 提供它，并把其 SHA-256 作为 `archive` 写在 `tree` 之后；`nabi publish` 从不发送 `archive`。不是 64 位小写十六进制数字的值视为不存在。
- 注册表包按以下顺序获取：对默认索引，先用工具链自带的副本（见下文“官方包”）；然后是缓存中已有的该提交的 git 检出；然后是它的压缩包（给出了 `archive` 时），来自 `cache/registry/src/<sha256>/` 或从索引的 `config.nabion` 的 `api` 下载（`Downloading <name> <version>`）；最后是它的 git `source`。SHA-256 不等于 `archive` 的下载是错误（`` package `num` 1.2.0: <url> has SHA-256 .., but the index says ..; the archive or the index is damaged .. ``），构建停止。无法获得压缩包（没有保存、无法访问、没有 `api`，或在 `--offline` 下没有缓存）时给出提示（note），改为获取 git 源。有压缩包时，既不需要 git，也不需要包的仓库；无论哪种方式，锁文件固定的都是同一个提交和树。

### 解析

- 对一个包的每个要求都计入：所选版本满足所有要求。各包按首次被要求的顺序挑选，每个包先尝试锁文件中固定的版本（只要满足所有要求就保留，即使之后被撤回），再从高到低尝试未撤回的版本。当某个选择使某个包没有满足其要求的版本时，搜索会回退，尝试在它之前挑选的包的下一个版本，因此一个依赖方的 `>=0.1` 和另一个依赖方的 `^0.1` 会解析为最高的 `0.1.x`。
- 新的选择（以及不带要求的 `nabi add`）会跳过 `nabi_version` 不接纳当前运行的 `nabi`（[9.10.6 兼容性](#兼容性)）或 `nabi` 无法读取的条目。当这隐藏了最高的匹配版本时，会给出提示，指出该版本、它需要的工具链（`` package `num` 2.0.0 needs nabi 0.2.1 or a later 0.2.x (this is nabi 0.1.5) ``）、如何使用它（`nabi +0.2.1 ..`，或固定该工具链）以及改为选择的版本；没有其他匹配时，错误把该提示作为帮助。锁文件中固定的版本无论如何都会保留。
- 对默认索引，在锁文件中固定的版本之后，先尝试工具链自带的版本（见下文“官方包”），再尝试其他版本：`^0.2` 会选择自带的 `0.2.0` 而不是索引中的 `0.2.5`。这些版本的行不需要索引，因此当搜索中的每个包都是自带的且没有缓存索引时，不会获取索引；当自带的版本无法满足某个要求时，获取索引并再次搜索。`nabi update` 从高到低尝试各版本，不论是否自带。
- 在一个依赖图中，一个名称就是一个版本。传递依赖遵循索引条目的 `deps`。路径包和 git 包是固定的：它们对注册表的要求与根包的要求同样计入。
- 当没有任何版本组合可行时，错误会指出某个没有版本能满足其要求的包，列出每个要求及其要求方（`` `^2` (required by `a` 1.4.0) ``）、索引中有的版本、每个要求方是如何进入构建的（`` `a` 1.4.0: the root package requires `a` `^1.0` ``），以及是否尝试过它们的其他版本。它的帮助说明应当怎么写：满足其他要求的该包的某个版本（`` change the root package's to it, `"b": "^2.3.0"` ``），或能与它们相容的某个要求方的版本（`` change the root package's requirement on it, `"c": "^0.4.0"` ``），否则放宽某个要求。没有任何版本能满足的单个要求，是普通的“no version of package .. matches ..”错误。搜索在 100000 次选择后放弃，并建议固定主要依赖。
- 锁文件会对照索引检查。如果某个固定版本的 `rev` 发生了变化，则是“run `nabi update`”错误。
- `--offline` 使用缓存的索引，并遵守 `--locked`。源不在任一缓存中（没有 git 检出，也没有压缩包）的版本是错误（`` package `json` 0.2.5 is not in the cache, and --offline is set: run the command once without --offline; `nabi component add packages` (or `update`) keeps the official packages for offline use ``）；后半句只针对官方包。

### 官方包

Nabi 项目发布的包在索引中标记为 `official: true`（由注册表写入）。代码可以不先添加就使用它：当对包或脚本执行 `build`、`run`、`check`、`test`、`bench`、`repl` 或 `jupyter` 时，如果发现某个用作路径根的名称（`use json::parse;`、`json::parse(s)`）不是该包或脚本的依赖（构建中其他包所依赖的不算），代码自身也没有声明它（模块、项、导入、绑定），而默认索引表明该名称是官方的（依据其最新版本，即最高的未撤回版本的那一行；查找名称时 `_` 与 `-` 视为相同，因此 `nabi_syntax::..` 就是 `nabi-syntax`，以其索引名添加），`nabi` 就会：

- 挑选工具链自带的版本（见下文），否则挑选本工具链能构建的（其 `nabi_version` 接纳本工具链）最高的未撤回版本，把 `"<name>": "^<major>.<minor>"`（低于 0.1 时为 `^<version>`）写入包的 `nabi.nabion`（工作区成员写入它自己的；脚本写入其 `//! nabion` 块，缺少时创建），解析并更新 `nabi.lock`，打印 `Added <name> ^x.y (official)`，然后继续执行命令；
- 对工具链自带的包，不读取索引；否则读取缓存的索引，只在没有缓存时才获取（获取失败，或在 `--offline` 下没有缓存时，不添加任何内容）；
- 使用 `--locked` 或 `--frozen`（错误中会如实写出所用选项），或 `CI` 设为非空值时，从不编辑清单：`` `json` is an official package the code uses, but it is not in nabi.nabion, and --locked never changes it; add it: `nabi add json` ``；
- 离线且包不在任一缓存中时：`` .. it is not in the cache and --offline is set: run `nabi add json` online, or keep the official packages for offline use with `nabi component add packages` ``。

根 `std`、`core`、`bare`、`nabi`、`pkg`、`sup`、`crate`、`super`、`self` 以及首字母大写的名称从不是包。不是官方的名称保留编译器的错误（`` `geom` is not a dependency of this package: add it to nabi.nabion (`nabi add geom`) ``）。REPL 和 Jupyter 内核对每个单元格都这样做（在包的会话中用 `nabi add`，在环境的会话中用 `nabi install --env`）。语言服务器从不编辑清单：它提供 `nabi add <name>` 快速修复（9.9.13）。

### `nabi component`

```
nabi component list
nabi component add packages [--bundle <dir>]
nabi component update [packages]
nabi component remove packages
```

`packages` 是默认索引中每个官方包的、本工具链能构建的（其 `nabi_version` 接纳本工具链）最高未撤回版本的注册表压缩包（9.10.4a 索引格式）。

- 发布版工具链自带它（分发，9.10.6）：`bin/` 旁边的 `lib/nabi/packages/` 在 `index` 中保存这些版本的索引行（每行一个，与索引文件相同），并把每个压缩包解压到 `src/<sha256>/`。除 `remove` 外，没有任何操作写入那里；对默认索引，解析首先读取它（见上文“解析”），它的行与索引指向相同的提交和树，因此锁文件与下载得到的相同。
- `add` 获取索引，把每个压缩包下载到 `cache/registry/src/<sha256>/`，对照该行的 `archive` 检查（`Downloading json 0.2.0` ..，然后是 `Added component packages (<n> official packages)`），并记录在 `$NABI_HOME/components.nabion` 中；组件已添加或由工具链自带时，再次 `add` 会给出提示，指向 `update`。`--bundle <dir>` 改为把它们作为工具链的 `lib/nabi/packages` 写入 `<dir>`（替换已有内容），与发布构建的做法相同（`Bundled <n> official packages into <dir>`）。
- `update` 再次获取索引，把本工具链能构建的最新版本下载到缓存中；工具链自带的内容保持不变。
- `remove` 删除工具链自带的内容、已记录的压缩包以及记录；两者都没有时是错误。
- `list` 显示每个组件及其是否已添加：`added: <n> package(s) with the toolchain`，`add` / `update` 也下载了一些时加上 `, <m> downloaded`。

没有 `archive` 的版本会给出提示，并在使用时从 git 获取。对 `add` 和 `update` 使用 `--offline` 是错误；其他名称是错误，错误消息会列出所有组件。

组件只是填充缓存：版本仍由清单和锁文件挑选。锁文件固定了工具链和缓存都没有的版本时，在线情况下会下载它，离线情况下是上述错误。

### `nabi search` / `publish` / `yank` / `login` / `logout`

```
nabi search [--native] <query>
nabi login [--registry <url>] [<token>]
nabi logout [--registry <url> | --all]
nabi publish [--dry-run] [--registry <url> | --index-dir <checkout>]
nabi yank [--native] <name>@<version> [--undo] [--message <why>] [--registry <url> | --index-dir <checkout>]
```

- `search` 按名称搜索索引，每个匹配项打印一行：`<name> <newest version> - <description>`。说明来自注册表 API（`GET <api>/api/v1/packages/<name>`，最新版本的 `meta.description`）；API 没有说明或没有应答时，以及在 `--offline` 下，省略说明。索引查找接受 `-` 或 `_` 任一种拼写。
- `publish` 执行发布（与 `cargo publish` 相同）；`publish --dry-run` 是试运行。带有 `publish: false` 的包被拒绝，错误消息会指出应当删除的那一行。readme（9.10.3）必须存在。它要求工作树干净、包位于仓库根目录，并具有 `src/lib.nabi`、`src/main.nabi` 或两者都有。只有 `src/main.nabi` 的包是程序：由 `nabi install` 构建，不能作为依赖。普通的路径依赖以其 `version:` 要求发布，没有该要求时被拒绝（`` dependency `linalg` is a path: .. add the version it needs, published from that path: `Dep::path("../linalg", version: "^0.1")` ``）；普通的 git 依赖被拒绝，没有 `version:` 的路径 / git 开发依赖被丢弃。它检查与同一版本的冲突，以及与在 `-` / `_` 等同和大小写折叠下相同的名称的冲突。已被以其他拼写占用的名称会被拒绝。不带 `--index-dir` 的试运行只打印条目；带 `--index-dir` 时还会对照该检出进行检查。条目的源是按配置的 git 远程 `origin` 的 url（在 `insteadOf` 改写之前）；它必须是 `https://`，`http://127.0.0.1` 和 `http://localhost` 除外。
- 不带 `--dry-run` 时，`publish` 添加条目：使用 `--index-dir` 时，作为该检出中的一个提交“Publish <name> <version>”（不推送）；不使用时，通过 `--registry <url>` 所指索引的注册表 API，否则通过默认注册表的 API（见“使用哪个索引”），由注册表把它添加到索引中。`--index-dir` 和 `--registry` 不能同时使用。客户端从不自己写入索引仓库。
- `yank` 为该版本设置（使用 `--undo` 时清除）`yanked`：使用 `--index-dir` 时，作为该检出中的一个提交；否则通过 `--registry <url>` 所指索引的注册表 API，再否则通过默认注册表的 API。
- `--commit`（`publish`）、`--index`（`publish` 和 `yank`）和 `--vers`（`yank`）是错误，错误消息会指出应当写的形式：不带选项的 `publish`、`--index-dir <dir>` 和 `yank <name>@<version>`。
- `yank --message <why>` 说明原因（最多 512 字节）；注册表保存并显示它。它只能与撤回一起使用：与 `--undo` 一起，或与 `--index-dir` 一起（检出中没有存放它的位置），是错误。
- 注册表 API 位于索引的 `config.nabion` 的 `api`。请求带有 `Authorization: Bearer <token>`，请求体是单行 NABION（`application/nabion; charset=utf-8`）：
  - `GET <api>/api/v1/me` → `Me { login: "..", publish: <bool> }`
  - `POST <api>/api/v1/publish`，请求体 `Publish { entry: <index line>, meta: { .. } }`（该行是 `Package { .. }` / `Native { .. }`）→ `Published { name, version, file, warnings: ["..", ..] }`；`nabi` 打印每条警告。
  - `POST <api>/api/v1/yank`，请求体 `Yank { name, version, undo: <bool>, native: <bool>, message: ".." }`（`message` 只在有说明时写出）→ `Yanked { name, version, yanked }`
  - `GET <api>/api/v1/packages/<name>` 和 `/api/v1/native/<name>`，不需要令牌 → `Package { name, native, owners, versions: [Version { version, yanked, meta, yank_message, has_readme, downloads, published_by, yanked_by, unyanked_by }, ..] }`；`GET <api>/api/v1/packages/<name>/<version>/readme` → markdown 格式的 README。

  失败是带有 `Error { message: ".." }` 的 4xx / 5xx 状态，`nabi` 报告其中的消息；401 表示令牌无效。`nabi` 跳过应答中不认识的字段。
- `meta` 保存描述包、但不参与解析的内容，来自清单（9.10.3）和包。没有值的字段被省略。

  | 字段 | 来源 |
  |-------|------|
  | `description`、`license`、`license_file`、`authors`、`homepage`、`documentation`、`repository`、`readme`（路径）、`keywords`、`categories`、`include`、`exclude`、`publish`、`metadata`、`links`、`default_run` | 清单中的同名字段 |
  | `has_lib` | 包是否有 `src/lib.nabi` |
  | `bin_names` | 它的可执行文件：有 `src/main.nabi` 时为 `[<name>]`，否则为 `[]` |
  | `readme_content` | readme（9.10.3）的文本，最多 524288 字节；更长的在 `nabi publish` 时是错误 |
  | `size` | 打包大小：包所含的 HEAD 处文件的字节数（`include` / `exclude`，9.10.3） |

  `nabi` 不认识的清单字段原样放在 `meta` 中发送，以便更新的注册表能够存储它们。对 `publish --native`，`meta` 是该文件的 `description`、`license` 和 `repository`。
- 注册表 API 需要令牌；`--index-dir` 检出不需要。默认注册表的令牌是 `NABI_REGISTRY_TOKEN`（如果已设置且非空），否则是 `nabi login` 保存的令牌。对其他索引，是 `nabi login --registry <url>` 保存的令牌；`NABI_REGISTRY_TOKEN` 从不发送到那里（已设置时警告一次：``NABI_REGISTRY_TOKEN goes only to https://github.com/nabi-lang/packages-index; for <url> run `nabi login --registry <url>` ``）。没有令牌时是错误“run `nabi login` first”（对其他索引为 ``run `nabi login --registry <url>` first``）。
- `nabi login` 接受注册表账户令牌，该令牌在网站上创建（`<api>/me`，用 GitHub 登录；目前只有注册表的维护者可以创建）。省略令牌时从标准输入读取，不回显（按 Ctrl-C 时恢复回显）。`login` 询问 `/api/v1/me`：它拒绝 API 不接受的令牌，以及 `publish` 为 false 的账户（“this account cannot publish yet: publishing is open to the registry's maintainers only”），不保存任何内容。否则，它把令牌按索引（`--registry <url>`，否则为默认注册表的索引）保存到 `$NABI_HOME/tokens.nabion` 中，形式为 `Tokens { tokens: { "<index url>": "<token>" } }`（键不带末尾的 `/`；已有的字段和其他索引保留），并打印“Logged in as <login>”。令牌只发送到它自己的索引的注册表 API。
- `tokens.nabion` 是唯一的令牌文件。
- 在 Unix 上，`tokens.nabion` 以模式 0600 写入。在 Windows 上，它获得 `$NABI_HOME` 的默认权限，而 `$NABI_HOME` 位于用户的配置文件目录下。
- `nabi logout` 从 `tokens.nabion` 中删除默认注册表的令牌；`--registry <url>` 删除该索引的条目；`--all` 删除所有令牌。
- 令牌只通过标准输入传给 `curl`：从不出现在命令行或环境中，也从不打印（错误中显示 `***`）。

## 9.10.4b. 原生包

原生包是每个目标三元组一个的预构建压缩包，加上一个值：

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

该值是一行索引，或者是供 `Dep::native(url:, sha256:)` 使用的文件，文件中的产物 url 可以相对于该文件。`nabi publish --native` 读取的文件还可以有 `description`、`license` 和 `repository`；它们放在 `meta` 中发送到注册表 API，而不写入索引行。

- 压缩包（任何 tar 可读的格式）含有 `lib/`：`lib<n>.a`、`lib<n>.dylib` / `lib<n>.so`、Windows 的 `<n>.lib`（静态库或导入库）。它还含有 `runtime` 文件和许可证文件。其中的任何内容都不会被运行。
- 哈希是必需的。压缩包在解压前接受检查，如果含有绝对路径或 `..` 路径、指向包外的链接，或缺少所指名的库或运行时文件，则被拒绝。
- 在 Apple 目标上，运行时 dylib 必须具有 `@rpath/` 安装名。
- 构建使用目标三元组对应的产物；缺少时是错误，错误消息会列出它具有的三元组。
- `nabi` 向 `nabic` 传递 `-L <archive>/lib`、每个链接一个 `-l kind=name`、每个运行时文件一个 `--native-runtime`，依赖方在依赖之前。

```
nabi add --native <name>[@<req>]
nabi remove --native <name>
nabi search --native <query>
nabi publish --native <file> [--dry-run] [--registry <url> | --index-dir <dir>]
nabi yank --native <name>@<version> [--undo] [--registry <url> | --index-dir <dir>]
```

`publish --native` 获取并检查每个目标，要求使用绝对 url、具有许可证文件，且依赖只来自索引。它的发布和 `yank --native` 遵循注册表包的规则：使用 `--index-dir` 检出，否则使用带令牌的注册表 API（`Yank` 中为 `native: true`）。

- 在 ELF 目标上，运行时库的 soname 必须等于它的文件名。为 `-musl` 目标使用 `dylib` 链接是错误。
- `repl` / `jupyter` 加载原生包：共享库通过系统加载器加载，静态归档文件链接进会话。`check` / `lint` / `doc` 不链接任何内容。
- 放在可执行文件旁边的副本是构建输出。

### 系统库与框架

```nabion
native: {
    "z": Dep::system(pkg_config: "zlib", brew: "zlib", vcpkg: "zlib", apt: "zlib1g-dev"),
    "png": Dep::system(pkg_config: "libpng16", libs: ["png16"], static: true),
    "Metal": Dep::framework(),
}
```

`Dep::system` 指定机器上已安装的 C 库。它的所有参数都是可选的具名参数：`pkg_config`（`.pc` 模块；默认为键）、`brew`（Homebrew formula）、`vcpkg`（vcpkg port）、`apt`（错误提示 Linux 用户安装的包）、`libs`（没有 `.pc` 文件说明时要链接的内容；默认为键）以及 `static: true`（链接静态库，并使用 `.pc` 文件的私有部分）。每次构建都按以下顺序查找它，不运行任何程序：

1. pkg-config `.pc` 文件，由 `nabi` 自己读取：`PKG_CONFIG_PATH`，然后是 `PKG_CONFIG_LIBDIR`（如果已设置），否则是目标的 multiarch 目录（`/usr/lib/x86_64-linux-gnu/pkgconfig` 等）、`/usr/local/lib/pkgconfig`、`/usr/local/share/pkgconfig`、`/usr/lib64/pkgconfig`、`/usr/lib/pkgconfig`、`/usr/share/pkgconfig`，在 macOS 上还有 Homebrew 的 `lib/pkgconfig`。会跟随变量（`${prefix}`、`${pcfiledir}`）、`Requires`（使用 `static` 时还有 `Requires.private` 和 `Libs.private`）；`PKG_CONFIG_SYSROOT_DIR` 作为它们给出的路径的前缀。
2. Homebrew，在 macOS 上（或设置了 `HOMEBREW_PREFIX` 时）：`$HOMEBREW_PREFIX/opt/<formula>`（默认 `/opt/homebrew`），使用它自己的 `.pc` 文件，否则使用它的 `lib/` 和 `include/`。
3. vcpkg，设置了 `VCPKG_ROOT`（或 `VCPKG_INSTALLATION_ROOT`）时：`installed/<triplet>`，triplet 由目标决定（`x64-windows`、`arm64-windows`、`static` 时为 `x64-windows-static`、`x64-linux`、`arm64-osx` 等），除非 `VCPKG_DEFAULT_TRIPLET` 指定了一个；在 Windows 上，它的 `bin/` 中的 DLL 放到可执行文件旁边。
4. macOS SDK 自带的库（`usr/lib/lib<name>.tbd`：`z`、`sqlite3` 等）。

交叉构建只在目标文件所在的位置查找：`PKG_CONFIG_PATH`、`PKG_CONFIG_LIBDIR`、目标的 multiarch 目录，以及设置了 `PKG_CONFIG_SYSROOT_DIR` 时该 sysroot 的默认目录；从不使用主机的 Homebrew。`nabi` 把找到的内容作为 `-L` 和 `-l` 传递；头文件目录交给 `c` 源文件。找不到时是错误，错误消息会指出查找过的位置以及主机上的安装命令：`` system library `png` (pkg-config `libpng16`) not found for x86_64-unknown-linux-gnu (looked in pkg-config (libpng16.pc)): install it with `sudo apt install libpng-dev` (or your distribution's package of it), or set PKG_CONFIG_PATH to the directory holding libpng16.pc ``（在 macOS 上为 `brew install`，在 Windows 上为 `vcpkg install <port>:<triplet>`）。

`Dep::framework()` 不接受参数：键就是框架名（`Foundation`，保持 Apple 的拼写），以 `#[link(name = "Foundation", kind = "framework")]` 的方式链接（[5.9.9](../005-modules/509-extern-block.md#599-linklibname指名库)）。针对非 Apple 目标的构建会去掉它。

图中每个包的系统库都会被链接，每个名称一个；注册表包的系统库来自它的 `nabi.nabion`（索引行中没有）。它们不在 `nabi.lock` 中：它们就是机器上已有的内容。

## 9.10.4c. 特性

包在 `features` 中声明特性，这是一个从特性名到该特性所启用内容的列表的字典。名称为 `[a-z0-9][a-z0-9_-]*`，用能说明其含义的词（`http1`、`http2`、`tls`）。每个条目是以下之一：

| 条目        | 启用                                                                 |
|-------------|----------------------------------------------------------------------|
| `<f>`       | 本包的特性 `f`                                                       |
| `dep:<d>`   | 可选依赖 `d`                                                         |
| `<d>/<f>`   | 依赖 `d` 的特性 `f`，如果 `d` 是可选的，还启用 `d` 本身              |
| `<d>?/<f>`  | 可选依赖 `d` 的特性 `f`，仅当有其他内容启用了 `d` 时                 |

- `default` 是一个普通特性，除非被关闭，否则处于开启状态：对正在构建的包用 `--no-default-features` 关闭，对依赖用 `default_features: false` 关闭。
- 可选依赖（`optional: true`）只有在某个已启用特性的 `dep:<d>` 或 `<d>/<f>` 条目启用它时才进入构建。不存在以它的名称命名的隐式特性，没有任何条目启用它时是错误。
- 条目指向包中不存在的特性或（`deps` 中的）依赖、`dep:` 或 `?` 指向非可选依赖，或特性之间形成环时，是错误。诊断显示 `file:line:col`。向某个依赖要求它没有的特性是错误。
- 解析：锁文件覆盖整个图。然后构建从根包的特性（除非关闭否则为 `default`、`--features`，或使用 `--all-features` 时为所有特性）出发，沿每个已启用的依赖前进，带上对它要求的特性以及它的 `default`。特性是累加并统一的：每个包在整个图中只有一个特性集。根包的 `dev_deps` 及其要求的特性只对 `test` / `bench` 计入。结果与任何事物的顺序无关。
- `--features` 接受根包的特性名，以及针对其直接依赖的 `<dep>/<f>`。
- 每个包都恰好以其已启用的特性对应的 `cfg(feature = "<f>")` 编译（[6.3.6](../006-meta-system/603-cfg.md#636-参数从何而来)）。原生包没有特性。
- 脚本的 ```` ```nabion ```` 块没有 `features`（9.10.7）。

## 9.10.5. 环境

环境取代 conda / venv，不需要激活步骤。

- 包（含有 `src/main.nabi`、`src/lib.nabi` 或 `nabi.nabion` 的目录）只使用它自己的 `nabi.nabion` 中的依赖，看不到安装到环境中的库。脚本也一样。作用于包的命令带上 `--env` 是错误。
- 具名环境把 `nabi.nabion`、`nabi.lock` 和 `target/` 保存在 `$NABI_HOME/envs/<name>/` 中。它的 `deps` 是用 `nabi install`（[9.10.5a](#9105a-nabi-install--nabi-uninstall)）为 REPL 和 Jupyter 安装的库；只有 `nabi install` / `nabi uninstall` 和 `nabi env import` 会改动它们。其中的路径依赖是绝对路径。已安装的库会预先构建到它的 `target/` 中。
- 在包之外使用的环境（REPL、Jupyter、语言服务器、`nabi install`）按 `--env <name>` > `NABI_ENV` > `nabi env use`（`$NABI_HOME/envs/.current`）> `default` 的顺序选定；`default` 在第一次向它执行 `nabi install` 时创建。在包内，不带路径的 `nabi repl` / `jupyter` / `lsp` 使用该包；带 `--env` 时使用那个环境。
- 环境名由 ASCII 字母、数字、`_`、`-` 和 `.` 组成，不得以 `.` 开头，最长 64 个字符，且不得是 Windows 设备名（`con`、`nul.txt` 等）。如果另一个环境的名称只在大小写上不同（`Foo` 与 `foo`），`nabi env create` 会拒绝：Windows 和 macOS 把它们视为同一个目录。
- 名称的每个来源（`--env`、`NABI_ENV`、`nabi env use` 写入的文件、`nabi env` 的参数）都会被检查。含有分隔符（`/`、`\`、`:`）的名称、`.` 或 `..` 在任何地方都是错误（`` `../p` is a path, not an environment name; use a name from `nabi env list`, e.g. `default` ``）。`nabi env create` 会拒绝的其他名称也是错误，除非该环境已存在（其目录是手动创建的）：这时只是警告，命令继续执行。`nabi jupyter install --env <name>` 要求环境已存在，`default` 除外。
- `nabi env use` 把名称写入 `$NABI_HOME/envs/.current`。只读取第一行。对当前选中的环境执行 `nabi env remove` 会删除该文件，于是不再有选中的环境。

### 依赖从哪里来

| 编译对象 | 依赖 | 锁文件 | 预先构建到 |
|---|---|---|---|
| 包 | 其 `nabi.nabion` 的 `deps`（test / bench 时为 `dev_deps`） | `<package>/nabi.lock` | `<package>/target/<profile>/deps/` |
| 脚本 | 它自身以及它的 `mod` 所加载文件中 `//! nabion` 块的 `deps` 合并结果 | `cache/scripts/<hash>/nabi.lock` | `cache/scripts/<hash>/target/` |
| 包之外的 REPL / Jupyter | 活动环境中已安装的库 | 该环境的锁文件 | 该环境的 `target/` |

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

REPL 中的 `:add <dep> ..` 会向当前环境执行 `nabi install <dep> ..`，在包的会话中则执行 `nabi add`；随后它重新读取依赖，并把它们附加到会话上。

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

- 来源处的包先被获取到缓存中（[9.10.4](#9104-依赖nabi-add--remove--update--deps)）。含有 `src/main.nabi` 的包或脚本文件是可执行程序：它以 `--release` 构建，其可执行文件放到 `$NABI_HOME/bin/<name>`，与 `cargo install` 相同。git 包或注册表包从其检出内容的临时副本构建；路径包和脚本文件就地构建。`$NABI_HOME/installs.nabion`（`Package { deps: { "<name>": <source> } }`）记录每个可执行程序及其确切来源（包记为 `=<version>`，git 包带上它的 `rev:`）。如果包还有 `src/lib.nabi`，`nabi install` 需要 `--bin` 或 `--lib`：两者都没有时是错误，错误信息指出 `--bin` 对应可执行程序、`--lib` 对应库，且不安装任何东西。对没有 `src/main.nabi` 的包使用 `--bin` 是错误，同时使用 `--bin` 和 `--lib` 也是错误。
- 只有 `src/lib.nabi` 的包，或带 `--lib` 的任何含有 `src/lib.nabi` 的包，是库，安装到活动环境中，与 `pip install` 相同，供 `nabi repl`、`nabi jupyter` 和 `:add`（[9.10.5](#9105-环境)）使用：它被写入环境的 `nabi.nabion`（没有版本要求的注册表包记为 `^<newest version>`，路径记为绝对路径），解析进其 `nabi.lock`，并预先构建到其 `target/` 中。脚本看不到它：脚本用 `nabi add <name> <file.nabi>` 获得库，输出行会指出这一点。如果构建失败，什么都不安装。对没有 `src/lib.nabi` 的包或对脚本使用 `--lib` 是错误。
- `<name>` 是库的顶层名称，或可执行文件的文件名。省略时取包的 `name`，否则取目录名（路径）、去掉 `.git` 的仓库名（git）或文件主名（脚本）。它遵循 [9.10.3](#9103-nabinabion--nabilock) 的命名规则。
- `--features` / `--no-default-features` 写入库的 `Dep`，或用于可执行程序的构建；`--dev` / `--optional` 是错误。`--native` 把原生包加入环境的 `native`（[9.10.4b](#9104b-原生包)）。
- 代理 `nabi`、`nabic`、`nabicc`、`nabils`（[9.10.6](#9106-toolchain)）永远不会被替换，`$NABI_HOME/bin` 中不是 `nabi install` 放进去的文件也不会被替换（这是错误）。再次安装可执行程序时：来自同一注册表版本或同一 git 提交且特性相同的，保留而不重新构建（`` `tool` 0.1.0 is installed already (<path>); to rebuild it, run `nabi install tool --force` ``），`--force` 会重新构建它（`Reinstalled`）；另一个版本会替换它（`Upgraded tool from 0.1.0 to 0.2.0`、`Downgraded ...`），另一个 git 提交也一样（`Replaced <url> at <commit> with ...`）。来自路径或脚本文件的总是重新构建，与 `cargo install --path` 相同。库的安装不受影响。
- `--list` 打印 ``libraries (environment `<env>`):`` 和 `executables (<bin dir>):`，各自后接按安装顺序排列的 `  <name> <source>` 行。
- `nabi uninstall <name>..` 从活动环境的库中逐个移除（并重新解析其锁文件），否则移除 `nabi install` 放在 `$NABI_HOME/bin` 中的可执行程序；`--native` 从环境中移除原生包。两者都不是的名称是错误。
- 除 `nabi install` 外，只有 `nabi toolchain`（工具链、代理、`config.nabion`）、`nabi login`（`tokens.nabion`）、`nabi env`（环境）和各缓存会在 `$NABI_HOME` 下写入。

## 9.10.6. toolchain

`$NABI_HOME/bin/` 中的 `nabi` / `nabic` / `nabicc` / `nabils` 是代理，它们运行活动工具链中的同名可执行文件（rustup 风格）。活动工具链按以下顺序选定：

1. 命令行上的 `+<version>`（`nabi +0.4 build`）
2. 环境变量 `NABI_TOOLCHAIN`
3. 所运行脚本的块中的 `toolchain: "X"`（[9.10.7](#9107-单文件脚本的内联清单)），适用于 `nabi <file>`，以及包之外对 `.nabi` 文件执行的 `nabi run` / `build` / `check` / `test`（代理读取文件的前 64 KiB）
4. 清单中的 `toolchain: "X"`（最近的 `nabi.nabion`；代理只读取这个字段，所以它不认识的字段不影响固定）
5. 默认值（`$NABI_HOME/config.nabion` 中的 `Config { toolchain }`）

固定值是以下之一：

- 一个版本或版本的前缀，匹配它之下的版本（`0.3` 是最高的 `0.3.*`，`0.3.1` 就是 `0.3.1`）；
- 9.10.4a 中那样的版本要求（`^0.3.1`、`~0.3.1`、`=0.3.1`、`>=0.3, <0.5`），匹配它接受的最高已安装版本；
- 工具链目录名（`0.3.1-<triple>`），匹配那个目录。

匹配的已安装正式版中最高者胜出；预发布版（`0.3.1-rc.1`）只在没有正式版匹配时胜出，所以 `0.3` 选 `0.3.0` 而不是 `0.3.1-rc.1`，而只安装了候选版的系列取其中最新的。精确指名预发布版的固定值（`0.3.1-rc.1`）只匹配它本身。清单中的固定值如果不属于以上任何一种，会产生警告，说明应当怎么写（`` `toolchain: "banana"`: write a version (`0.1`), a requirement (`^0.1.0`) or a toolchain directory name ``）。如果没有工具链匹配，就是错误，错误信息指出固定值及其来源，并给出安装它的命令：完整版本时为 `nabi toolchain install <pin>`，否则为带一个该固定值接受的完整版本的 `nabi toolchain install <version>`，或 `--from`。

系列是一个破坏性版本槽位涵盖的版本：主版本号为 0 时是 `0.MINOR`（`0.1`），之后是 `MAJOR`（`1`）。只有最新的系列会得到修复和安全更新；较旧的系列仍可下载和安装，但不再受支持。

### 兼容性

工具链能构建同系列中更早的工具链构建过的代码：1.0 之前，`x.y.z` 中 `y` 的变化可能破坏代码（`0.1.7` 到 `0.2.0`），从 1.0 起是 `x` 的变化（`1.9.0` 到 `2.0.0`），而更高的 `z`（1.0 之前）或 `y`（1.0 起）不会。包的 `nabi_version`（9.10.3）遵循这条规则：`"0.1.3"` 接受 0.1.3 到最后一个 0.1.x，`"1.4.2"` 接受 1.4.2 到最后一个 1.x。预发布工具链按其正式版计（`0.1.4-dev` 按 `0.1.4` 计）。

代理是最新已安装工具链的 `nabi` 的副本：即本主机上的最高版本，只在没有安装正式版时才是预发布版。较新的 `nabi` 由此继续运行较旧的工具链，所以代理约定在整个 0.x 中固定：先按上面的顺序选定，再以相同参数运行 `toolchains/<dir>/bin/<tool>`，设置 `NABI_HOME`，并把 `NABI_TOOLCHAIN` 设为代理取下的 `+<version>`。

每个工具链的 `nabi` 只运行它自己的 `nabic`，即它旁边的那一个，所以 `nabi` 与 `nabic` 之间的标志、环境变量和文件从不构成跨版本的约定。直接运行的 `nabi`（开发构建、包管理器提供的、工具链的 `bin/nabi`），如果 `+<version>`、`NABI_TOOLCHAIN` 或清单的 `toolchain` 指向另一个已安装的工具链，就把命令交给那个工具链的 `nabi`：从它启动时所在的目录以相同参数运行，设置 `NABI_HOME`，并以其状态码退出（在 Unix 上替换自身；在 Windows 上等待）。对于会编译的命令（委托给 `nabic` 的命令，或会编译的工具），清单是该命令所作用的包的清单（`nabi build ../app` 读取 `../app/nabi.nabion`）；对于其他命令，是最近的清单。以下情况不移交：固定值匹配它自己的版本，工具链就是它运行所在的目录（按真实路径比较），或那个工具链的 `nabi --version` 打印的行与它自己的相同（同一构建）。没有固定值时，默认值不适用：直接运行的 `nabi` 保留命令。`nabi toolchain`、`help`、`version`、`completions` 和 `--version` 从不移交。接到移交的工具链如果又把命令交回，是错误，错误信息指出两个可执行文件，绝不会形成循环。对于会编译的命令，指向未安装工具链的固定值是上面所说的错误；其他命令继续执行。

```
nabi toolchain list                         # marks the default with * and (default)
nabi toolchain install <version> [--from <dir | archive | url>]
nabi toolchain default [<version>]          # without an argument, prints the current default
nabi toolchain remove <version>
nabi toolchain update [<version>]
nabi toolchain clean                        # remove every installed toolchain but the default
```

- 安装的工具链位于 `$NABI_HOME/toolchains/<version>-<triple>/`，且是自包含的（[9.2.19](902-aot-pipeline.md#9219-安装布局)）。
- `install --from` 从目录、压缩包或其 URL 安装；`<version>` 是工具链必须匹配的固定值。不带 `--from` 时，`install` 下载一个发布版本（见下文“分发”）：`<version>` 是完整版本（`0.1.0`）；其他任何写法都是错误，错误信息说明应写完整版本（`` nabi toolchain install: `0.1` is not a full version; write one like `0.1.0` ``）。已安装的工具链不会再次下载（`toolchain <dir> is already installed`）。它不改变默认值。
- `install` 用 `bin/nabic --version --verbose` 验证；版本和主机必须匹配。
- 第一个安装的工具链成为默认值，`bin/` 代理也在此时创建。
- 当某个 `bin/` 代理缺失或不是最新工具链的 `nabi` 时，`install`、`default`、`update`、`remove` 和 `clean` 会重写代理，否则保持不变。把较旧的工具链设为默认值时保留较新的代理；移除最新的工具链后，代理改为次新工具链的。
- `nabi toolchain update` 替换 `$NABI_HOME` 下的默认工具链。不带参数时，它安装最新的发布版本（`latest/download/nabi-version`），即使它属于另一个系列（rustup 风格）。默认值移到另一个系列时，说明旧工具链仍保持安装的提示还会指出只有最新系列会得到修复，因此项目应当迁移，并说明项目在迁移之前如何留在旧系列：``note: nabi <old version> stays installed, but <old series> gets no more fixes; move projects to <new version>. Until a project moves it can pin <old series> (`toolchain: "<old series>"` in nabi.nabion or a script's `//! nabion` block) or run `nabi +<old series> ..` ``。带参数时，它安装那个发布版本：参数是完整版本（`update 0.1.1`），没有系列选择；其他任何写法都是错误，错误信息说明应写完整版本（`` nabi toolchain update: `0.1` is not a full version; write one like `0.1.0`, or nothing for the newest release ``）。安装的工具链成为默认值，最新版本变化时重写 `bin/` 代理；在 Windows 上，正在运行的代理会先被重命名，再由其副本替换。它下载发布包并验证其 SHA-256（见下文“分发”），在别处解包后移动到位，然后检查代理能运行它；失败时恢复之前的默认值和代理，并删除新工具链。更早的工具链保持安装。已安装的版本直接设为默认值，不再下载。`nabi update` 重新解析 `nabi.lock`；`nabi toolchain update` 替换工具链（9.10.4）。
- 执行 `nabi toolchain default <version>` 或 `nabi toolchain update` 之后，安装脚本写入的补全脚本（见下文“分发”）会根据新默认工具链的 `nabi completions` 重写。只重写已存在的文件，不创建新文件；没有 `completions` 的工具链保持它们不变。这不打印任何内容；`-v` 会列出重写的文件。
- 执行 `nabi toolchain default <version>` 或 `nabi toolchain update` 之后，`nabi` 在 `PATH` 上查找 `nabi`（在 Windows 上尝试每个 `PATHEXT` 扩展名）。如果找到的第一个不是 `$NABI_HOME/bin/nabi`（按真实路径比较），它会发出警告，给出那个文件及其 `--version` 打印的版本：`warning: your shell runs <path> (nabi X) before ~/.nabi/bin/nabi; put ~/.nabi/bin first on PATH or remove the other one`。`-q` 省略该警告（安装脚本就是这样运行的）。
- `nabi --version -v`（或 `nabi version -v`）在版本行之后打印 `binary`（本可执行文件）、`toolchain`（活动工具链及其原因：命令行上的 `+`、`NABI_TOOLCHAIN`、固定它的 `nabi.nabion`，或默认值；没有时为 `none`）、`nabic`（此处命令运行的 `nabic`：它自己的，或移交时固定工具链的）和 `host`，每项一行 `key: value`，值对齐，形式与 `nabic --version -v` 相同（`host: <triple>`）。`--format json`（`nabi version -v --format json`、`nabi --version --format=json`）改为打印一个 JSON 对象：`schema`（目前为 `1`）、`version`、`line`（版本行中 `nabi ` 之后的文本），带 `-v` 时还有上述四个键。
- `nabi toolchain clean` 移除除当前默认值之外的所有已安装工具链，这是一项需要主动执行的清理，用于 `update` 留下的旧版本。项目固定的版本也会被移除，所以如果构建需要它，请重新安装；没有设置默认值时是错误。

### 构建缓存

```
nabi cache dir                              # print the toolchain's cache directory
nabi cache prune [--unused <days>] [--max-size <size>]
nabi cache clean                            # remove everything in it
```

缓存是活动工具链的 `lib/nabi-cache/`（[9.2.19](902-aot-pipeline.md#9219-安装布局)）：函数缓存（存放构建时编译的标准库代码）以及库收集的项（[9.2.16](902-aot-pipeline.md#9216-增量构建)）。其中的内容都不是必需的；构建会重新计算缺失的部分。

- `prune` 移除 `--unused` 天内没有任何构建读取过的条目（函数缓存和库阶段的条目；默认 30 天，即构建每天一次自行清理时所用的天数），然后按最近读取时间从旧到新移除，直到剩余不超过 `--max-size`（字节数，或带 `K`、`M` 或 `G` 的数字；默认为上限 `NABI_CACHE_MAX_SIZE`，[9.2.16](902-aot-pipeline.md#9216-增量构建)），并打印移除和保留的数量。
- `clean` 移除缓存目录中的所有内容。

### 工具链之间共享的文件

安装在同一 `$NABI_HOME` 中的每个工具链都读写那里的相同文件：`config.nabion`、`tokens.nabion`、`sources.nabion`、`installs.nabion` 和 `envs/.current`。较新的 `nabi` 可能在其中写入了较旧的 `nabi` 不认识的字段。

- 读取时，`nabi` 跳过它不认识的字段，在 `installs.nabion` 中还跳过它不认识的 `Dep` 参数和条目种类（不认识的种类的条目不列出）。带 `-v` 时，它对每个文件只说明一次跳过了什么：``note: <file>: skipped `x`, written by a newer nabi (kept in the file)``。
- 写入时，它保留不认识的内容：`nabi toolchain default` 只设置 `config.nabion` 的那一个字段，`nabi install` / `nabi uninstall` 设置或移除 `installs.nabion` 中的那一个条目，`nabi login` 设置 `tokens.nabion` 中其索引的条目，`nabi logout` 只移除该条目。`sources.nabion` 只读不写。
- `nabi.nabion` 中 `nabi` 不认识的字段会产生警告（[9.10.3](#9103-nabinabion--nabilock)）；`nabi` 认识其版本的 `nabi.lock` 仍按严格规则处理。
- 较新的 `nabi` 在新文件中添加设置，而不是作为这些文件的新字段，以便较旧的工具链在它旁边继续工作。

### 分发

- 发布包是 `nabi-<version>-<triple>.tar.gz`（Windows 为 `.zip`），包含工具链布局；Windows 之外，旁边还有一个文件相同的 `nabi-<version>-<triple>.tar.xz`。每个压缩包旁边都有 `<archive>.sha256`：压缩包的十六进制 SHA-256，后面可选跟文件名（sha256sum 格式）。
- `install.sh`、`install.ps1` 和 `nabi toolchain install --from <url>` 从同一位置下载 `<url>.sha256`，并拒绝 SHA-256 不一致的压缩包；缺少 `.sha256` 是错误。按路径给出的压缩包在 `<path>.sha256` 存在时对照它检查。不定义签名；来源由 HTTPS 担保。
- `install.sh`（<https://nabi.sh>）从压缩包或 `nabi-lang` 组织的 GitHub releases 安装到 `$NABI_HOME`。如果 tar 能读取 `.tar.xz`（macOS，或 `PATH` 上有 `xz`）且该版本提供了它，就下载 `.tar.xz`，否则下载 `.tar.gz`。`nabi toolchain install` 下载 `.tar.gz`（在 Windows 上为 `.zip`）。
- `install.sh` 随后像 rustup 那样配置 shell。它写入 `$NABI_HOME/env`（sh）和 `$NABI_HOME/env.fish`，二者在 `$NABI_HOME/bin` 尚不在 `PATH` 上时把它放到 `PATH` 最前面；并把带保护的一行 `[ -f "$HOME/.nabi/env" ] && . "$HOME/.nabi/env"`（针对该 `$NABI_HOME`）添加一次到 `~/.profile`、`~/.bashrc`、`~/.bash_profile` 和 `~/.zshenv` 中已存在的每一个，以及登录 shell 的文件（不存在时创建；bash：`~/.bashrc`，以及在 `~/.bash_profile` 和 `~/.bash_login` 都不存在时的 `~/.profile`；zsh：`$ZDOTDIR/.zshenv`；其他 shell：`~/.profile`）。当 fish 是登录 shell 或其配置目录存在时，fish 得到 `~/.config/fish/conf.d/nabi.fish`（设置了 `$XDG_CONFIG_HOME` 时位于其下）。`nabi` 的补全脚本放到 `${XDG_DATA_HOME:-~/.local/share}/bash-completion/completions/`、`$NABI_HOME/zsh/`（`env` 会把它加入 zsh 的 `fpath`），fish 则放到 `~/.config/fish/completions/`。它会列出改动的文件；新开的 shell 会生效，再次运行不会改动任何东西。没有 `$NABI_HOME` 时，那行带保护的代码不起作用。
- `install.ps1` 把 `$NABI_HOME\bin` 放到用户 `Path` 的最前面，并写入 `$NABI_HOME\completions\nabi.ps1`，由 PowerShell 配置文件中一行带保护的代码加载，该行只添加一次。
- `install.sh` 和 `install.ps1` 像 `nabi` 一样在 stderr 上打印状态行，使用相同的配色和相同的规则（[9.1.5](901-nabic-cli.md#915-全局选项)、[9.1.8](901-nabic-cli.md#颜色)），不使用 `tput`。`install.sh` 会检测它的 stderr，所以在终端上执行 `curl .. | sh` 时输出带颜色。
- `--no-modify-path`（`install.ps1 -NoModifyPath`）或 `NABI_NO_MODIFY_PATH=1` 不改动 shell 文件、`Path` 和配置文件，只打印应当添加到 `PATH` 的内容。
- 发布包在 `lib/nabi/packages/` 中附带官方包（`nabi component`，9.10.4a），每个都是其 `nabi_version` 接受该版本的最新版本，所以使用它们的代码无需网络即可构建。`--no-packages`（`install.ps1 -NoPackages`）或 `NABI_NO_PACKAGES=1` 在安装后对已安装的工具链运行 `nabi component remove packages`；如果失败，工具链连同这些包保持安装，脚本会说明这一点。
- 发布版本从 `$NABI_DIST`（默认 `https://github.com/nabi-lang/nabi/releases`）提供：`<dist>/download/v<version>/<archive>` 及其 `.sha256`，以及 `<dist>/latest/download/nabi-version`，后者用一行指出所有系列中最新的发布版本，不带参数的 `nabi toolchain update` 读取它。候选发布版（`<version>-rc.N`）是预发布版：`latest/` 从不指向它，所以只能通过其完整版本安装。
- `nabi toolchain update <version>` / `install <version>`、`install.sh --version <version>` 和 `install.ps1 -Version <version>` 接受完整版本；最新版本之外的发布版本就这样安装。不发布按系列划分的文件。

### 安装器约定

每个已发布工具链的 `nabi toolchain update` / `install` 以及每个版本的安装脚本都必须能够安装之后的每个版本，所以发布包保留它们所依赖的内容。这一约定已冻结；破坏它的版本不予发布。

- 名称：`nabi-<version>-<triple>.tar.gz`（Windows 为 `nabi-<version>-<triple>.zip`），Windows 之外还有 `.tar.xz`，按上面“分发”中的方式提供，各自带有 `.sha256`。
- 指针：`<dist>/latest/download/nabi-version`，用一行指出最新的发布版本，使它们不带参数的 `nabi toolchain update` 能找到它。
- 布局：一个顶层目录 `nabi-<version>-<triple>/`，其中有 `bin/nabi` 和 `bin/nabic`（Windows 为 `bin\nabi.exe` 和 `bin\nabic.exe`）、`lib/<runtime>`（`lib/libnabir.a`；Windows 为 `lib\nabir.lib`）以及 `lib/nabi/core/src/lib.nabi`。
- `bin/nabic --version --verbose` 的第一行打印 `nabic <version>`（后面可选跟一个空格和更多文本），并打印一行 `host: <triple>`。

## 9.10.7. 单文件脚本的内联清单

匿名包（[4.16.8](../004-functions/416-entry-point.md#4168-单文件执行)）只从 ```` ```nabion ```` 块获得依赖。这些块含有 `Package { deps: { .. } }`（以及 `native`），位于它的文件以及它的 `mod` 加载的每个文件最开头（shebang 行之后）的 `//!` 行中（类似 uv 的内联脚本元数据）。安装到环境中的库永远不会被使用。块中只能有 `deps`、`native` 和 `toolchain`（没有 `features`，因此没有可选依赖）。`toolchain` 固定运行该脚本的工具链（9.10.6），只允许出现在被运行文件的块中；出现在模块的块中是错误，错误信息指出被运行的文件。直接运行的 `nabi` 会像对待包那样，把脚本交给固定工具链的 `nabi`。其他语言的代码块被忽略。块中的路径相对于含有该块的文件。各块会被合并；同一个库在两个文件中给出不同条目是错误，错误信息指出两个文件和两个条目。直接运行子模块文件时，只使用该次运行所到达的文件的块。`nabi add <name> .. <file.nabi>` / `nabi remove <name> <file.nabi>` 会写入一个块（[9.10.4](#9104-依赖nabi-add--remove--update--deps)），`nabi init --script` 把脚本转为包（[9.10.2](#9102-nabi-new--nabi-init创建包)）。脚本的锁文件和构建保存在 `cache/scripts/<hash>/` 中，以其路径以及它所到达的每个文件的路径和块为键。脚本的可执行文件按构建它的所有内容缓存，包括它包含的每个文件和它的库。脚本使用自己缺少的包的 `use x::..` 是错误，其帮助是要运行的命令 ``run `nabi add x <file>` ``，活动环境中有 `x` 时附带一条提示；在包中，它建议 `nabi add x`。`import x`（以及 `include` / `require`）是错误：`x.nabi` 位于文件旁边时帮助为 `mod x;`，`x` 是路径根或文件声明的模块时为 `use`，否则为 `nabi add x <file>`。

````nabi
//! ```nabion
//! Package {
//!     deps: { "util": Dep::git("https://example.com/util.git", tag: "v1") },
//! }
//! ```

use util::greet;
greet();
````

运行单文件脚本时会复用它的可执行文件。对匿名包执行 `nabi run` 时，可执行文件保存在 `<target>/[<triple>/]<profile>/scripts/<key>/` 中，其中 `<target>` 是脚本的 `target/`（位于 `cache/scripts/<hash>/` 下，在其 `nabi.lock` 旁边）；由 `nabi.nabion` 变为单文件包的文件（9.10.1），以及由 `nabic` 构建的单独文件，改用单独文件缓存（[9.2.16](902-aot-pipeline.md#9216-增量构建)）。`<key>` 是构建所依据的所有内容的哈希：工具链、构建选项、lint 级别和目标、源代码及其包含的每个文件、工具链为该目标附带的运行时和 C 库，以及传给它的依赖及其预先构建的库。键匹配时不编译任何内容，并再次打印生成该可执行文件的那次构建的警告。新的可执行文件先以临时名称写入，再重命名到位，所以同一脚本的并发运行是安全的。一天未使用的条目会在构建另一个条目时被移除。

## 9.10.8. 工具

工具是 `nabi` 自己的子命令。`nabi` 在自己的进程中运行它们，并链接了编译器；会编译的工具接受 `nabic build` 的编译选项（[9.1.2](901-nabic-cli.md#nabic-buildaot-构建)），并按 9.10.1 所述获得包的依赖、特性和 lint。除非某个工具另有说明，它们的诊断、进度行和颜色遵循 [9.1.8](901-nabic-cli.md#918-输出格式)，用法错误和退出码遵循 [9.1.6](901-nabic-cli.md#916-退出码)。`nabi lsp`、`nabi repl` 和 `nabi jupyter` 在编译器内部错误后仍能继续：语言服务器把它报告为所分析缓冲区的一条错误诊断，REPL / 内核让该单元格失败（[9.7.5](907-jit-cells.md#975-错误模式)）；两者都继续提供服务。

### `nabi run`：构建并运行

```
nabi run [path] [build options] [-- args...]
```

`nabi run` 像 `nabic build` 那样构建可执行包，并立即运行它（其他 `--target` 的可执行文件通过其运行器运行，见下文）；`--` 之后的参数原样传给可执行文件，可以是任意字节（[9.1.1](901-nabic-cli.md#911-一般形式)）。选项与 `nabic build` 相同。库包没有可运行的东西，所以是错误。

包构建到 `target/<profile>/<name>`；单独的 `.nabi` 文件作为匿名包构建，从 9.10.7 的缓存中运行，所以不会在文件旁边创建 `target/`。`nabic build <file>.nabi` 则会构建它并把可执行文件复制到文件旁边。如果自上次构建以来没有任何变化，`nabi run` 打印 `Fresh`，再次打印那次构建的警告，并跳过构建（[9.2.16](902-aot-pipeline.md#9216-增量构建)）。

```
nabi run hello.nabi -- arg1 arg2
```

`nabi run` 的退出码就是程序的退出码。如果程序因信号 `n` 终止，退出码为 `128 + n`。

程序不会在 `nabi run` 结束后继续存活。发给 `nabi` 的 `SIGTERM`、`SIGHUP`、`SIGINT` 或 `SIGQUIT` 会转发给程序（终端发给其前台进程组的信号已经会到达程序），如果程序因该信号结束，`nabi run` 也因它结束。如果 `nabi run` 本身被杀死，程序随之被杀死；在 Windows 上，程序在一个随 `nabi` 结束的作业对象中运行。

### `nabi dev`：带热重载运行

```
nabi dev [path] [-p <member>] [--bin <name>] [build options] [-- args...]
```

`nabi dev` 为热重载（[9.14](914-hot-reload.md)）构建包的程序，运行它，并监视其源代码。保存时会重新构建所涉及的本地包，程序在运行中采用新代码，同时保留其状态。

- *宿主*（运行时以及每个注册表包或 git 包）只构建一次，为 `target/dev/<name>-host`。每个本地包（该包、它的 `path` 依赖、它使用的工作区成员）是一个热插件，用 `nabic build --hot` 构建为 `target/dev/<package>.<n>.dylib`（`.so`、`.dll`），`n` 是从 1 开始的构建序号，旁边是其布局文件 `target/dev/<package>.<n>.layout`。会话开始时清空 `target/dev/`。
- 每 100 ms 轮询一次本地包的 `src/` 目录树（每个 `.nabi` 文件的修改时间和大小）。文件静止 50 ms 后才采纳变更，所以一次保存多个文件算作一次变更。它以增量方式（[9.2.16](902-aot-pipeline.md#9216-增量构建)）、依赖优先，重新构建文件有变化的包以及依赖它们的每个本地包，并把新版本发送给程序（[9.14.5](914-hot-reload.md#9145-开发通道)）。
- 程序的回复打印为状态行：`Applied`、`Waiting`、`Refused`（附带应当怎么做）、`Superseded`。
- 编译错误像 `nabi build` 那样打印，后接 `` Kept the running code of `game`: fix the error and save ``。程序继续运行旧代码。
- 对 `nabi.nabion`（该包的、本地依赖的或工作区根的）、`nabi.lock`，或工具链的 `nabi` 或 `nabic` 的改动会重启程序，重启前打印一行说明原因：`` Restarting nabi.nabion changed ``。重启会重新解析依赖并全部重新构建；程序的状态会丢失。
- 按键：`r` 重启，`q` 退出。在终端上，按键按下即生效（`nabi dev` 关闭行编辑和回显，结束时恢复）；否则以每行的第一个字母为准。程序的标准输入为空；其标准输出和标准错误就是 `nabi dev` 的。
- 程序自行结束时，`nabi dev` 报告其退出码并等待：保存或按 `r` 会再次运行它，按 `q` 退出。热插件加载失败的程序在加载器的错误之后以退出码 101 结束（[9.12.6](912-plugins.md#9126-稳定性与错误)）。
- Ctrl-C（或 `SIGTERM`、`SIGHUP`）结束会话：程序被杀死并回收，`nabi dev` 以 130 退出。`q` 以 0 退出。
- `--` 之后的参数在每次启动时传给程序。

`-p <member>` 运行一个工作区成员；`--bin <name>` 指定包的可执行文件（一个包有一个可执行文件，以包名命名）。构建选项与 `nabic build` 相同，只用于本机的 dev 构建配置：`--release` 或其他 `-O` 级别是错误（``hot reload runs debug builds; drop `--release` ``），`--target`、`-o` 和 `--crate-type` 也是错误（``` `nabi dev` builds for this machine only; drop `--target` ```）。没有 `src/main.nabi` 的包是错误，错误信息说明应在程序所在的包中运行它。

### 其他目标的可执行文件

当 `nabi run`、`nabi test` 和 `nabi bench` 带有本机自身不能运行的 `--target` 时，每个可执行文件通过目标运行器启动。运行器是一个程序，先接收可执行文件，再接收其参数：

- `NABI_RUNNER=<program> [args]`（按空白拆分）是主机之外每个目标的运行器。
- 没有它时，`wasm32-wasip2` 和 `wasm32-wasip1-threads` 模块在 `PATH` 上的 `wasmtime` 下运行：`wasmtime run`，使用主机的环境（在 `wasm32-wasip2` 上还有其网络和名称查找）、8 MiB 的引擎栈（与原生主线程一样深），并把主机的文件系统挂在 `/`。WASI 没有工作目录，所以相对路径从 `/` 解析。
- `wasm32-unknown-unknown` 模块在 `PATH` 上的 `node` 下运行，使用工具链的 JS 宿主，即 `lib/nabi/targets/wasm32-unknown-unknown/` 中的 `nabi.mjs`。
- 在 Linux 上，其他 CPU 的 Linux 目标在 `PATH` 上 qemu-user 的 `qemu-<cpu>` 下运行（`qemu-aarch64`、`qemu-arm`、`qemu-riscv64`、`qemu-ppc64le`、`qemu-s390x`、`qemu-i386` 等）；当 `/usr/<triple>`（`/usr/aarch64-linux-gnu`）目录存在且未设置 `QEMU_LD_PREFIX` 时带上 `-L /usr/<triple>`。没有 `qemu-<cpu>` 时，如果 binfmt_misc 有 `qemu-<cpu>` 处理程序，就直接运行。
- 主机操作系统上、面向主机能够运行的另一种 CPU 的目标会直接运行，先于查找 qemu-user：x86_64 上的 32 位 x86，以及 CPU 在 EL0 运行 AArch32 代码时 arm64 Linux 上的 32 位 ARM（Neoverse N1、Cortex-A72；Neoverse N2 和 Apple silicon 不行）。没有寄存器能告诉程序这一点，所以 `nabi` 会运行一次一个极小的静态 32 位 ARM 可执行文件来确定。

否则，或者当所需程序缺失时，命令失败，错误信息指出要安装什么，或 `NABI_RUNNER`：``cannot run a wasm32-wasip2 module here: `wasmtime` is not on PATH; install it (https://wasmtime.dev), or set NABI_RUNNER to a program that runs it``。`nabi run` 在构建之前检查；`nabi test` 和 `nabi bench` 在测试框架构建之后检查，`--list` 不需要运行器即可列出。退出码是运行器的退出码，对上述所有运行器而言就是程序的退出码。

```
nabi test --target wasm32-wasip2
NABI_RUNNER="qemu-aarch64 -L /opt/sysroot" nabi run --target aarch64-unknown-linux-gnu
```

### `nabi lint`：lint 工具

```
nabi lint [path] [--list] [--check-library] [-A|-W|-D <lint | group>..]
```

`nabi lint` 运行前端和 lint，不进行代码生成（`nabic check` 会加上代码生成，[9.1.2](901-nabic-cli.md#nabic-check诊断)）。每个会编译的命令都报告相同的 lint（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）；`lint` 只运行它们和前端。诊断格式见 [9.1.8](901-nabic-cli.md#918-输出格式)；lint 诊断的代码为 `E0131`。lint 和组的列表见 [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)。`--list` 打印每个 lint 及其默认级别和引入它的版本，然后打印各组。`--check-library` 与 `nabic check` 中的相同。

lint 级别由 `-A` / `-W` / `-D`（[9.1.5](901-nabic-cli.md#915-全局选项)）和代码中的属性设置。如果有任何 `deny` 级别的 lint 触发，退出码为 `1`。

### `nabi fix`：应用修复

```
nabi fix [path] [--allow-dirty] [--diff] [-A|-W|-D <lint | group>..]
```

`nabi fix` 像 `nabi lint` 那样检查包，并应用检查结果所带的修复：恰好带有一处编辑的 lint 结果或编译器警告（有多个备选方案的诊断留给编辑器的快速修复）。只要还能找到编辑，它就再次检查并重复，最多 8 轮；与同一轮中另一处编辑重叠的编辑等到下一轮。只修改该包的文件；依赖和标准库的文件永远不改。带修复的 lint：

| Lint | 修复 |
|------|-----|
| `unused_variables` | 在 `let` 的名称前加 `_`（`let _y = 2;`） |
| `unused_imports` | 当 `use` 只导入该名称、单独占一行且上方没有属性时，删除该 `use` |
| `bool_comparison` | `x == true` 变为 `x`，`x == false` / `x != true` 变为 `!x`（含运算符时为 `!(x)`） |
| `while_true` | `while true` 变为 `loop` |
| `redundant_cast` | `x as T` 变为 `x` |
| `redundant_clone` | 删除 `.clone()` |
| `missing_f_prefix` | 字符串加上 `f` 前缀 |

lint 级别（`-A` / `-W` / `-D`、清单的 `lints`、属性）决定报告什么，因而决定修复什么：`nabi fix -A unused_imports` 保留未使用的导入。当包位于 git 工作树中，且其目录下有未提交的改动时，`nabi fix` 拒绝执行（``the work tree has uncommitted changes (src/main.nabi); commit them first so the fixes can be reviewed and undone, or run `nabi fix --allow-dirty` ``）；`--allow-dirty` 照样修复。在 git 之外没有这项检查。`--diff` 像 `nabi fmt --diff` 那样打印第一轮的编辑，不写入任何内容。没有修复能解决的错误会使它以退出码 `1` 停止，并打印 ``N errors to fix by hand first; nothing was changed``（在某一轮之后则为 ``the files fixed so far are kept``）。最后，它对每个改动的文件打印 `Fixed <file> (N fixes)`，或打印 `<name>: nothing to fix`。

### `nabi repl`：终端 REPL

`nabi repl` 在终端中启动一个交互式 shell。不带子命令的 `nabi` 与它相同（9.10.1）。

```
nabi repl [path] [-q]
```

单元格的语义遵循 [9.7.0](907-jit-cells.md#970-单元格会话的规则) 的规则：每个单元格恰好运行一次，顶层 `let` 绑定和项延续到之后的单元格，失败的单元格不在绑定中留下任何痕迹。表达式单元格打印其值（`Display`，否则 `Debug`，再否则为其类型名）。

- 默认模式用 Nabi 的代码生成器编译每个单元格，并把代码交给一个存活于整个会话的 JIT 宿主进程；宿主加载它并运行一次（[9.6](906-jit-engine.md)）。绑定、`static` 状态和 `any` 值在单元格之间延续。类型无法在顶层命名的绑定（闭包、私有类型等）在创建时被拒绝。
- 诊断针对单元格自身的文本给出 `<repl>:line:col`；更早单元格的项和运行时位置给出 `<repl cell N>:line:col`（[9.7.5](907-jit-cells.md#975-错误模式)）。
- 在终端上，输入支持行编辑、Tab 补全、颜色和整单元格历史（[9.7.8](907-jit-cells.md#行编辑)）；历史保存在 `$NABI_HOME/repl_history` 中。
- 在终端上，会话以工具链行和一条提示开始（[9.7.8](907-jit-cells.md#行编辑)）；`-q` 省略它们。
- 不带路径时，会话以空状态开始。程序路径（单独的 `.nabi` 文件或含有 `src/main.nabi` 的包）会被加载并运行，顶层语句作为单元格、`fn main` 作为入口，然后会话继续（[9.7.10](907-jit-cells.md#9710-加载程序repl-path)）。库包路径（`src/lib.nabi`）被添加为依赖，`use <name>::*;` 作为第一个单元格，`:reset` 之后也是如此。
- 在包内不带路径时，不加载该包。找不到但由该包定义的名称会得到提示 ``load this package: `nabi repl .` ``。
- 多于一个路径是用法错误。不接受 `-`；标准输入就是单元格输入。
- `NABI_REPL_TIME=1` 在 stderr 上打印单元格每个阶段的耗时（[9.9.9](909-debug-tools.md#999-repl--jupyter-调试)）。

两个根文件都有的包是程序路径；它的库是会话的依赖。`nabi repl <path>` 从该路径所在的项目解析依赖和 lint（9.10.1）；在包之外，会话使用活动环境的库（9.10.5）。

REPL 命令（`:type`、`:doc`、`:time`、`:vars`、`:load`、`:reload`、`:edit`、`:save`、`:opt`、`:reset`、`:add`、`:help`、`:quit` / `:exit`，以及单独的 `exit` / `quit`）和调试辅助见 [9.9.9](909-debug-tools.md#999-repl--jupyter-调试)。

### `nabi jupyter`：Jupyter 内核

```
nabi jupyter install [--env <name>]
nabi jupyter --connection-file <path>
```

`nabi jupyter install` 注册内核（[9.8.3](908-jupyter-integration.md#983-安装-kernelspec)）；kernelspec 运行 `nabi jupyter --connection-file {connection_file}`。

`nabi jupyter --connection-file <path>` 就是内核。它由 Jupyter 启动，用户不直接运行它。`--connection-file` 接受 Jupyter 提供的 ZMQ 连接文件的路径；单独的 `nabi jupyter` 是用法错误，错误信息指向 `nabi jupyter install`。`--features` 等构建选项也被接受。内核运行在与 REPL 相同的单元格会话上（[9.7.0](907-jit-cells.md#970-单元格会话的规则)），并像 `nabi repl .` 那样加载它启动时所在的包；开始监听后，它在 stderr 上打印 `nabi jupyter: ready`（[9.8.2](908-jupyter-integration.md#982-本地原生内核运行方式)）。集成细节定义在 [Jupyter 集成](908-jupyter-integration.md)中。

### `nabi lsp`：LSP 服务器

```
nabi lsp
```

`nabi lsp` 在标准输入和标准输出上运行 Language Server Protocol 服务器。它由 VS Code、Vim、Emacs 等 LSP 客户端调用。独立的编辑器二进制文件 `nabils` 是同一个服务器。`.nabion` 缓冲区只检查语法。在两个根文件都有的包中，库的模块树能到达的文件作为库的一部分检查，其他文件作为可执行程序的一部分检查，可执行程序能看到库已打开的缓冲区。功能列在 [9.9.13](909-debug-tools.md#9913-lsp-服务器) 中；检查在后台运行，在此期间查询根据上一次成功的检查作答。

### `nabi fmt`：格式化工具

```
nabi fmt [path..] [--check] [--diff]
```

`[path]` 接受文件或目录（可以多个），并把它们就地重写为[标准格式](#标准格式)。作为包的目录（[9.2.2](902-aot-pipeline.md#922-包发现)）按其 `src/`、`tests/`、`benches/` 和 `examples/` 目录以及其 `nabi.nabion` 格式化；其中的其他文件不受影响。不带路径时，`fmt` 格式化当前目录所在的包：即当前目录或其上级中最近的含有 `src/main.nabi`、`src/lib.nabi` 或 `nabi.nabion` 的目录。在包之外，不带路径是错误（退出码 `1`）：`` not inside a package; pass a file or directory: `nabi fmt <path>` ``。目录会被递归搜索 `.nabi` 和 `.nabion` 文件，跳过 `target/`、`node_modules/`、以 `.` 开头的目录，以及被所搜索目录或其包根目录下的 `.gitignore` 忽略的内容。只读取以下几种 `.gitignore` 行：`name`（其下任意位置的该名称）、`name/`（仅目录）、`/name`（直接位于 `.gitignore` 所在目录中）和 `*suffix`；注释、`!` 行、其他通配符以及中间含 `/` 的路径被跳过。在终端上，耗时超过一秒的搜索会在标准错误上显示 `Scanning <n> files…`。`.nabion` 文件（[NABION](../../refs-standard-libraries/300-std/308-nabion.md#384-规范形式)）会检查是否为 NABION，并且只重新缩进（不会重写为规范形式，所以注释得以保留）。

最后，除非带 `-q`，标准错误会说明发生了什么：`Formatted <n> files`（有些文件已格式化时附带 `, <m> already formatted`）、`note: <n> files already formatted`，或者在 `--check` / `--diff` 下为 `` note: <n> files need formatting; `nabi fmt` without `--check` rewrites them ``。没有可格式化内容的目录给出 `note: no .nabi or .nabion files under <dir>`（退出码 `0`）。

选项：

- `--check` 不修改文件；它打印将会改变的文件及其 diff。如果有任何文件将会改变，退出码为 `1`；适用于 CI。
- `--diff` 不修改文件，只打印 diff。

有词法错误的文件保持不变，并报告该错误，退出码为 `1`。

#### 标准格式

`nabi fmt` 的标准格式是一种**保留词法单元的布局规则**。它不改变一行之内的词法单元或注释；它设定行缩进和空行，并对模块顶部的 `mod` / `use` 项排序。`nabi fmt` 有意不提供配置：所有 Nabi 代码只有一种标准格式。

1. 缩进为每层括号（`(` `[` `{` `#[`）深度 4 个空格。以闭括号开头的行取打开它的那一行的深度。制表符变为空格。
2. 续行，即前一个词法单元不是 `;`、`,` 或括号的行（例如拆分长表达式的行），保持原文件中超出块缩进的那部分缩进量。
3. 删除行尾空白。
4. 连续的空行合并为一行，删除文件开头的空行。
5. 文件以恰好一个换行结尾。
6. 文件顶部以及内联 `mod name { }` 顶部（在其第一个其他项之前）的 `mod name;` 和 `use` 项按 `pub mod`、`mod`、`pub use`、`use` 的顺序分组，任何 `pub(..)` 都算作 `pub`。每组内排序（`mod` 按名称，`use` 按路径逐段排序，`self`、`sup` 和 `pkg` 在前），组与组之间以一个空行分隔，组内没有空行。项正上方的注释、文档注释和属性随项移动；第一个项上方的注释留在原处。与另一个项共享一行的项结束顶部区域。

除规则 6 外，格式化工具不移动行，不拆分或合并行，也不改变一行之内的间距。因此注释完全保持原样，词法单元序列除规则 6 排序的项的顺序外保持不变。多行字符串和块注释的内部也保持原样。

这条布局规则在 1.0 之前一直是标准格式：在此之前，不会有重新格式化代码（移动词法单元、拆分或合并行）的美化打印器取代它，所以 `nabi fmt --check` 接受的文件在之后的 0.x 工具链上继续通过。

### `nabi doc`：文档生成

```
nabi doc [path] [--std] [--open]
```

根据包的库（没有库时为其可执行程序）及其依赖包的公开项和文档（`///`、`//!`、`#[doc]`，[6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---文档注释形式)），在 `target/doc/` 中生成 HTML API 文档。

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

`<pkg>` 是包的名称（清单的 `name`）；标准库层级使用它自己的名称（`core`、`std`、`bare`）。当模块路径上的每个模块都是 `pub` 且不是 `#[doc(hidden)]` 时，该模块是公开的；每个公开模块有一个页面：根模块的是 `<pkg>/index.html`，`a::b` 的是 `<pkg>/a/b.html`（根模块下名为 `index` 的子模块是 `<pkg>/index/index.html`）。包的根页面之上没有更高一层的页面：每个页面的页眉显示其包名（链接到该包的根页面）和版本，侧边栏列出该包的依赖（链接到它们的根页面），并与面包屑导航一起通向它自己的包的根页面，而搜索在任何页面上都覆盖每个已生成文档的包。更早的工具链留下的 `target/doc/index.html` 会被删除；`target/doc/` 中的其他文件不会。页面不需要网络：样式表、脚本和字体都在输出中。

模块的页面显示其文档（它的 `//!` 行，排在其 `mod` 项上的 `///` 之后；普通的 `//` 注释不是文档注释，`nabi lint` 会对位于库文件或公开模块文件开头的这类注释发出警告，[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）、其公开子模块，然后按种类列出其项：重新导出、结构体（联合体也在其中）、枚举、trait、类型别名、函数、常量、静态变量，最后是在别处声明的类型的实现（`impl str { .. }`）。这些项是不带 `#[doc(hidden)]` 或 `#[internal]` 的 `pub` 项，每项附带按原样书写的签名、其文档、其成员（公开字段、变体、trait 项、其固有 impl 的公开方法和常量）以及其 trait 实现。页面显示什么由以下规则决定：

- 由 `pub use` 从私有模块引入、或只经由私有模块引入的项，显示在进行重新导出的页面上，就像在那里声明的一样，使用重新导出的名称，并把该 `pub use` 的文档放在最前面。`pub use private::*` 的每个项也是如此。
- 对某个公开模块页面所显示的项的重新导出，保持为一个重新导出条目，链接到那个页面。
- 写在私有模块中的 `impl` 的方法，归入其上方最近的公开模块页面上的对应类型，否则归入该页面的实现部分。
- 每个项都有锚点 `#<kind>.<name>`，`<kind>` 为 `struct`（联合体也是）、`enum`、`trait`、`type`、`fn`、`const` 或 `static`（`app/index.html#struct.Point`）；成员为 `#<Type>.<member>`（`#Point.x`、`#Shape.Circle`、`#Point.origin`）；实现为 `#impl.<Type>`；重新导出为 `#reexport.<names>`，即它引入的名称以 `.` 连接，其中每个名称为它所指项的 `#<kind>.<name>`，所以 `#struct.Event` 能在重新导出 `Event` 的页面上找到它。
- 签名中的类型，以及行内代码中指向已生成文档的项的名称，会链接到显示该项的页面，无论在本包还是在依赖中；没有页面显示的名称保持为纯文本。每个链接都指向存在的页面和锚点。

docs.nabi-lang.org 上的标准库 API 就是按这些规则生成的：每个模块一个页面，页面相同（那里是 `std/core/io.html`，这里是 `core/io.html`），锚点相同，`nabi doc` 的页面也共用它的样式表和布局。

`--std` 还会把标准库层级的文档生成到 `core/`、`std/`、`bare/` 中；不带路径时只生成它们，输出到 `./target/doc/`。

`nabi doc` 打印包的根页面（`Generated ./target/doc/<pkg>/index.html`）；带 `--std` 且不带路径时为 `std/index.html`。`--open` 随后打开该页面：设置了 `$BROWSER` 时用它，否则用系统的打开程序（macOS 上为 `open`，Linux 上为 `xdg-open`，Windows 上为 `start`）。如果无法启动，会发出警告，指出需要手动打开的文件；退出码不受影响。

文档文本按 markdown 的一个子集解释（标题、围栏代码块、行内代码、链接、`**bold**`、列表、段落）。在 Nabi 代码块中，以 `# ` 开头的行（或单独的 `#`）不显示：它是该块的文档测试（[9.1.2](901-nabic-cli.md#文档测试)）的准备代码，由 `nabi test` 运行。

### `nabi clean`：删除构建输出

```
nabi clean [path]
```

删除包的整个 `target/` 目录（包括所有构建配置和交叉构建；如果设置了 `--target-dir` / `NABI_TARGET_DIR`，则删除该目录）。目录不存在时什么都不做。

### `nabi bindgen`：C 头文件绑定

```
nabi bindgen <header> [-x objective-c] [-o <file>] [--target <t>] [--c-shim <file.c>] [-- <C compiler args>]
```

把 `<header>` 的 [标准库 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#176-自动生成绑定) 声明写到标准输出（或 `-o <file>`）；其第一行以 ``// Generated by `nabi bindgen` `` 开头。`nabi` 用工具链的 C 编译器（[9.13](913-c-compiler.md)）的前端读取头文件；它不需要其他 C 编译器，也不需要 libclang。`--` 之后的参数是会改变头文件所声明内容的 C 编译器选项：`-I <dir>`、`-isystem <dir>`、`-iquote <dir>`、`-idirafter <dir>`、`-D <name>[=<value>]`、`-U <name>`、`-include <file>`、`--sysroot <dir>`、`-isysroot <dir>`、`-nostdinc`、`-nostdlibinc` 和 `-nobuiltininc`。`-std=` 和警告选项会被接受并忽略；其他任何选项都是错误，错误信息列出上述选项。前端拒绝的头文件是错误（退出码 1），报告在其 `file:line:col` 处，并说明应当改写成什么（[标准库 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#bindgen-读取的-c) 列出它读取的内容）。

带 `-x objective-c`（Apple 目标）时，头文件按 Objective-C 读取，其类、协议和类别像 `ffi::objc::import` 那样变为 `extern "objc"` 块（[标准库 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-头文件ffiobjcimport)）；默认为 `-x c`。

`static inline` 函数没有可链接的符号，所以会被跳过并留下一条注释。带 `--c-shim <file.c>` 时则会为它生成绑定：`<file.c>` 在 `#include "<header file name>"` 之后为每个这样的函数得到一个 C 包装函数 `R nabi_shim_<f>(A a0, ..) { return <f>(a0, ..); }`，绑定链接到该包装函数（`#[link_name = "nabi_shim_<f>"]`）。包从其清单的 `c.sources` 编译 `<file.c>`，并把头文件所在目录放入 `c.include`（9.10.3）。可变参数的 inline 函数仍被跳过。

### `nabi inspect`：包检查

`nabi inspect` 给出归档文件内容的概览。

```
nabi inspect path/to/libfoo.a        # or foo.nabihir
nabi inspect --dump path/to/foo.nabihir   # or libfoo.a; reads the sibling foo.nabihir
```

它显示包名和哈希、nabic 版本、NABIHIR 版本、GC 类型 id 范围、导出、[泛型](../000-meta/007-glossary.md#generic泛型)项，以及归档成员及其大小（[9.9.14](909-debug-tools.md#9914-包检查)）。

带 `--dump` 时，它改为以文本形式打印预先构建的库的 NABIHIR：头部字段、生成者、元数据和接口文件；格式见 [NABIHIR 格式](904-nabihir-bytecode.md)。

### `nabi demangle`：符号还原

`nabi demangle` 把经过修饰的符号解码为人类可读的形式。

```
nabi demangle [-v] <symbol>...
nabi demangle
```

对每个符号，它按源代码中的写法打印其名称，每行一个；带 `-v` 时，打印 [9.5.4](905-symbol-mangling.md) 格式的块，以空行分隔（[9.9.3](909-debug-tools.md#993-符号还原demangle)）。如果有任何参数不是 Nabi 符号，退出码为 `1`。不带符号时，它把标准输入过滤到标准输出，把文本中的每个 Nabi 符号替换为其名称（`perf report --stdio | nabi demangle`）。

### `nabi completions`：shell 补全

```
nabi completions bash|zsh|fish|powershell
```

在标准输出上打印该 shell 的补全脚本。它由与 `nabi help`（对于委托的子命令则是 `nabic` 的）相同的命令和选项文本生成，所以两者一致。它补全子命令及其选项、`+<version>`，以及用已安装工具链补全 `toolchain default` / `toolchain remove` 的参数，用环境名补全 `env use` / `remove` / `shell` / `export` 和 `--env`，用 `nabic --print target-list` 打印的名称补全 `--target` 的值。这些名称在补全时通过运行 `nabi __complete toolchains` / `envs`（每行一个）读取，因此跟随 `$NABI_HOME`。其他任何 shell 或不指定 shell 是用法错误（退出码 2）。`install.sh` 和 `install.ps1` 安装这些脚本，`nabi toolchain default` / `update` 会重写它们（9.10.6）。
