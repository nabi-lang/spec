# 9.1. `nabic` CLI 接口

本节定义编译器 `nabic` 的命令行接口。工具链由四个二进制文件组成。

| 二进制文件   | 作用                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `nabic`   | 编译器（相当于 rustc）。构建、检查和输出包，并构建测试和基准测试的测试框架（本节） |
| `nabi`    | Cargo 风格的前端。负责清单、依赖、环境和工具链固定，调用 `nabic`，并运行链接了编译器的各个工具（run、lint、REPL、Jupyter、LSP、格式化工具、文档……）（[9.10](910-nabi-frontend.md)） |
| `nabicc`  | C 编译器。与 `nabi cc` 是同一个编译器（[9.13](913-c-compiler.md)）                 |
| `nabils`  | 供编辑器使用的 LSP 服务器可执行文件。与 `nabi lsp` 是同一个服务器                       |

运行时是 `libnabir.a`（`nabir`）。没有单独的工具（如 `nabi-fmt`）。`nabic` 不解析依赖；依赖包由 `nabi` 通过 `--extern` 传入（[9.2.3](902-aot-pipeline.md#923-依赖解析)）。

## 9.1.1. 一般形式

`nabic` 按以下一般形式调用。

```
nabic <subcommand> [options] [args]
```

写成语法规则：

```
cli_invocation  ::= "nabic" subcommand option* arg*
subcommand      ::= "build" | "emit" | "test" | "bench" | "check"
                  | "version" | "help"
option          ::= "--" identifier ("=" value)?
                  | "-" letter
```

`--name value` 与 `--name=value` 等价。全局选项（9.1.5）可以出现在子命令之前或之后。

子命令、选项、选项的值和路径必须是有效的 UTF-8。不是有效 UTF-8 的实参属于用法错误（9.1.6），错误会指出它的位置：``argument 3 (`b�c`) is not valid UTF-8; only a program's arguments, after `--`, may be other bytes``。`--` 之后的实参不受此限：`nabi run` 按系统给出的原样把它们传给程序（在 Windows 上，也包括含有未配对代理码位的 UTF-16；[9.10.8](910-nabi-frontend.md#nabi-run构建并运行)）。

| 领域                | 子命令                                           |
|---------------------|-------------------------------------------------------|
| 构建               | `build`、`emit`、`test`、`bench`、`check`             |
| 其他               | `version`、`help`                                     |

各个工具（`run`、`lint`、`repl`、`jupyter`、`lsp`、`fmt`、`doc`、`clean`、`bindgen`、`inspect`、`demangle`、`completions`）是 `nabi` 的子命令（[9.10.8](910-nabi-frontend.md#9108-工具)）。

### 包的选择（`path`）

`build` / `emit` / `test` / `bench` / `check`（以及 `nabi` 的各个工具）的 `[path]` 是以下之一。省略时为当前目录。

| `path`                       | 包                                                                                                 |
|------------------------------|---------------------------------------------------------------------------------------------------------|
| 包目录            | 该目录。如果其中没有 `src/main.nabi` 或 `src/lib.nabi`，则向上查找（9.2.2）           |
| 包的 `src/` 中的文件   | 该包：从该文件向上查找时第一个包含 `src/main.nabi` / `src/lib.nabi` 的目录 |
| 其他任何文件               | 由这一个文件组成的匿名可执行包（[4.16.8](../004-functions/416-entry-point.md#4168-单文件执行)），包括包目录中位于其 `src/` 之外的 `.nabi` 文件 |

如果给出的是目录，而向上查找找不到包，则是错误。匿名包的文件子模块（`mod foo;`）位于其文件旁边（[4.16.8](../004-functions/416-entry-point.md#4168-单文件执行)）。

包名（即输出名）是包目录的名称；匿名包的包名是去掉 `.nabi` 的文件名。

## 9.1.2. 核心子命令

### `nabic build`：[AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 构建

`nabic build` 以 AOT 方式构建一个包。

```
nabic build [path]
```

`[path]` 遵循上面的包选择规则。输出放在 [9.2.15](902-aot-pipeline.md#9215-输出) 规定的按构建配置划分的 `target/` 中。`src/main.nabi` 生成 `target/<profile>/<name>`；`src/lib.nabi` 生成 `target/<profile>/lib<name>.a` 以及接口文件 `target/<profile>/<name>.nabihir`（[9.4](904-nabihir-bytecode.md)）；两者都有的包两者都生成。`-O2` / `-O3` / `-Os`（包括 `--release`）的构建配置为 `release`，其他情况为 `debug`；使用 `--target=<target>` 时目录为 `target/<target>/<profile>/`。如果设置了 `--target-dir=<dir>` 或 `NABI_TARGET_DIR`，就用该目录代替 `target/`。

单个文件（[4.16.8](../004-functions/416-entry-point.md#4168-单文件执行)）作为匿名包在临时目录中构建，因此不会在该文件旁边创建 `target/`；它的可执行文件会以文件主名加上目标的可执行文件后缀复制到该文件旁边（`hello.nabi` 得到 `hello`，在 Windows 上为 `hello.exe`），除非 `-o` 指定了另一个文件。如果该路径上已有一个不是由该脚本之前的构建写入的文件，则保留不动：构建失败，并提示 `-o <file>`。

选项：

- `--release` 在 `release` 构建配置中执行优化构建（等价于 `-O2`）；默认是在 `debug` 构建配置中执行调试构建（`-O0`）。两种构建配置的差别见 [9.2.15](902-aot-pipeline.md#9215-输出)。
- `--target-dir=<dir>` 改变 `target/` 的位置（等价于 `NABI_TARGET_DIR`）。
- `--crate-type=lib|cdylib|staticlib|plugin` 选择库包的输出。`lib`（默认）是静态归档文件及其 NABIHIR；`cdylib` 还会在它们旁边链接一个供 C 代码使用的共享库，即 `lib<name>.so`、`lib<name>.dylib`，或 `<name>.dll` 加 `<name>.lib`（[9.3.3](903-package-archive.md#933-静态库与共享库)）；`-o` 指定该文件。它还会写出该库的 C 头文件 `target/<profile>/include/<name>.h`（[9.3.3](903-package-archive.md#933-静态库与共享库)）。`staticlib` 生成一个 C、Swift 和 Xcode 可以单独链接的静态库 `static/lib<name>.a`（在 Windows 上为 `static/<name>.lib`；`-o` 指定它），并附带头文件和 `include/module.modulemap`（[9.3.3](903-package-archive.md#933-静态库与共享库)）；对 wasm 目标使用它是错误。与它一起使用时，`--xcframework` 还会把该目标的切片加入 `<profile>/<name>.xcframework`（Apple 目标）。`plugin` 则链接一个由 Nabi 程序加载的插件，文件名相同，但没有 `.lib`，也没有头文件（[9.12](912-plugins.md)）；对 `aarch64-apple-ios`（进行中，[0.6.2](../000-meta/006-platform-targets.md#062-进行中)），由于其设备只加载签名进应用中的代码，这是错误，错误信息会提示 `aarch64-apple-ios-sim`。对没有 `src/lib.nabi` 的包使用这些选项是错误；对同时有 `src/main.nabi` 的包，或对 wasm 或 musl 目标使用 `plugin` 也是错误。
- `--lib-version <x.y.z>` 为 `cdylib` 指定版本：在 Linux 上，其 soname 为 `lib<name>.so.<x>`，低于 1.0 时为 `lib<name>.so.0.<y>`，并在库旁边创建一个同名的链接；在 macOS 上，它设置其 install name 的当前版本（`x.y.z`）和兼容版本（`x.0.0`，低于 1.0 时为 `0.y.0`）。不指定时，soname 为文件名，两个版本都是 1.0.0。`nabi build` 传入包的 `version`（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。不是三个数字的值属于无效选项。
- `--bundle=<dep>`（与 `--crate-type=plugin` 一起使用）把依赖 `<dep>` 链接进插件，而不是从加载它的程序中获取。
- `--hot` 构建一个热重载插件，也就是 `nabi dev` 加载到正在运行的程序中的那种插件（隐含 `--crate-type=plugin`）。只有 `src/main.nabi` 的包被构建为根插件，其 `main` 以 `<name>::main` 导出；两个根文件都有的包是错误。对热包中函数的每次调用都经过一个由热重载运行时重新绑定的槽；它的静态变量绑定到第一个版本的存储；它的类型被完整描述；它的包哈希对该包的每次构建都相同（即其名称的哈希），因此新版本保留旧版本的名称。编译器还会在插件旁边写出 `<output stem>.layout`：版本号以及该包每个类型的布局。`--hot` 只在 `-O0` 下构建：`--release`、`-O1` .. `-O3` 和 `-Os` 都是错误（[8.4.13](../008-abi/804-safepoint.md#8413-热重载代码)）。
- `--hot-base=<file>`（与 `--hot` 一起使用）指定程序正在运行的版本的 `.layout`。新版本的编号接在它之后，并与它对照检查：新增了没有默认值的字段、字段类型改变、`#[renamed_from]` 指向不存在的旧字段，或类型参数或种类发生改变，都是错误（E0149），错误信息会说明应当改写成什么。
- `--hot-dep=<dep>`（与 `--hot` 一起使用）表示依赖 `<dep>` 也是热包：它的函数通过它自己插件的槽调用，而不是编译进来，它的包哈希也以同样方式固定。
- `--extern <name>=<dir | src/lib.nabi | <name>.nabihir | lib<name>.a>` 把一个依赖包挂载到顶层路径 `name` 上（[9.2.3](902-aot-pipeline.md#923-依赖解析)）。源代码（`dir` / `src/lib.nabi`）在同一次构建中一起编译；预构建的包（`.nabihir` / `.a`；给出其中一个时会找到它旁边的另一个）读取其 NABIHIR 并链接其归档文件。可以多次给出，通常由 `nabi` 填入。`name` 是一个标识符：把包名中的 `-` 替换为 `_`（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。`std`、`core` 和 `bare` 属于标准库：如此命名的库包会被拒绝，`--extern std=<path>` 是用法错误，除非前面的某个 `--extern` 指定了同一路径，使它成为该依赖的另一个名称（`--extern helper=p --extern std=p`，代码中写作 `std::..`；`::std::..` 仍然是标准库）。关键字和路径起始（`pkg`、`sup`）同样如此，无论是作为 `name`，还是作为库的名称或带库的包的名称（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。
- `--extern-feature <name>=<f>` 以 `cfg(feature = "<f>")` 构建源代码依赖 `name`。可重复，每个特性一次。依赖只能看到这些特性：`--cfg` 的 `feature` 键属于正在编译的包。通常由 `nabi` 填入。
- `--extern-dep <package>=<name>`：包 `<package>`（一个 `--extern` 名称，或表示正在构建的包的 `pkg`）依赖外部包 `<name>`。可重复。只要给出任意一个，每个包就只能看到它所依赖的外部包（[9.2.3](902-aot-pipeline.md#923-依赖解析)）；一个都不给时，每个包都能看到所有外部包。通常由 `nabi` 填入。
- `--name <n>` 设置包名（命名规则见 [9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。输出为 `lib<n>.a` / `<n>.nabihir`，挂载名是把 `-` 替换为 `_` 后的名称。默认是目录名。`--extern a-b=..` 同样绑定为 `a_b`。
- `--bootstrap <file>` 指定要预加载的标准库根（可重复）；`--nabir <lib>` 指定要链接的运行时。默认值为 sysroot 中的 `lib/nabi/{core,std,bare}/src/lib.nabi` 和 `lib/libnabir.a`（[9.2.19](902-aot-pipeline.md#9219-安装布局)）。引导根的层级是其 `src/` 上一级目录的名称；给出 `core/src/lib.nabi` 而没有 `bare` 根时，也会加载它旁边的 `bare/src/lib.nabi`。如果找不到，则是错误，错误信息会列出查找过的路径。
- `-O0` / `-O1` / `-O2` / `-O3` / `-Os` 显式选择优化级别（[9.2.10](902-aot-pipeline.md#9210-优化级别)）：`-O0` 不优化；`-O1` 像 `-O2` 那样优化循环，其他地方几乎不增加代码，并带调试信息；`-O2` 优化速度；`-O3` 以更多代码换取更高速度（循环中更大范围的内联，展开标量循环）；`-Os` 优化大小，但循环仍会转为计数循环，循环体较短时仍会向量化。
- `--target=<target>` 选择目标平台：内置名称，即 [平台目标](../000-meta/006-platform-targets.md#061-已支持) 中受支持的目标之一（进行中的目标只能配合维护者的测试开关使用，[0.6.2](../000-meta/006-platform-targets.md#062-进行中)），或者 `<name>.nabion` 目标规格文件（[0.6.5](../000-meta/006-platform-targets.md#065-目标规格)）。任何目标都可以在任何主机上构建（[0.2.4](../000-meta/002-pipeline.md#024-代码生成)）。其他名称都是错误。对于进行中或不受支持的目标（[0.6.2](../000-meta/006-platform-targets.md#062-进行中)、[0.6.3](../000-meta/006-platform-targets.md#063-不支持)），错误信息会这样说明，并在有替代时指出应改用什么（``target `x86_64-apple-darwin` is not supported (Intel Macs); use `aarch64-apple-darwin` ``，``target `aarch64-apple-ios` (iOS devices) is in progress: it is not tested on real hardware yet and comes in a later release; for now build for the simulator with `--target aarch64-apple-ios-sim` ``）；对于其他名称，如果有最接近的内置目标，则给出它（``unknown target `wasm32-wasi`; did you mean `wasm32-wasip2`?``）。
- `-L <dir>` 为 `#[link]` 添加一个库搜索目录（[5.9.9](../005-modules/509-extern-block.md#599-linklibname指名库)），在 `LIBRARY_PATH` 之前搜索。可重复。
- `-l [static=|dylib=|framework=]<name>` 链接库 `<name>`（像 `#[link]` 一样搜索），或者使用 `framework=` 时，像 `#[link(name = "<name>", kind = "framework")]` 那样链接 Apple framework `<name>`；`-F <dir>` 添加一个 framework 搜索目录，在 SDK 的目录之前搜索。`--native-runtime <file>` 指定一个要随可执行文件一起分发、放在其旁边的共享库（[9.2.14](902-aot-pipeline.md#9214-链接可执行文件)）。这些选项都可重复；`nabi` 为原生包和系统库传入它们（[9.10.4b](910-nabi-frontend.md#9104b-原生包)）。在非 Apple 目标上使用 framework 是链接错误，帮助信息会建议把该 `extern` 块放在 `#[cfg(target_os = "macos")]` 之下。
- `--native-object <file>` 把一个目标文件或归档文件原样链接进可执行文件或共享库（可重复）；`nabi` 传入包的 C 源文件编译出的目标文件（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。
- `-o <path>` / `--output=<path>` 选择输出位置。
- `--keep` 保留构建的临时目录（目标文件和其他中间文件），而不是删除它，并在标准错误上给出其名称（`nabic: kept temporaries in <dir>`）。
- `--cfg <flag>` / `--cfg <key>=<value>` 为[条件编译](../006-meta-system/603-cfg.md)设置一个 cfg 标志。可重复。
- `--emit=<kind>[,<kind>..]` 把中间输出（`source`、`hir`、`mir`、`obj`、`asm`、`listing`、`expansions`、`c-header`）写到 `target/<profile>/build/<name>.<ext>`（[9.9.1](909-debug-tools.md#991-中间输出--emit)）。可以用逗号分隔一次给出多种；这种情况下不能使用 `-o`。使用 `--emit` 时，不生成可执行文件或归档文件。
- `-g` 为 `release` 构建配置的构建添加调试信息：DWARF，或者对于 Windows 目标为 CodeView 和 PDB（[9.9.6](909-debug-tools.md#996-调试器集成gdb--lldb)）。它不改变生成的代码：带 `-g` 的优化构建与不带 `-g` 时机器码相同，其调试信息描述优化后的代码把每个变量保存在哪里（[9.2.10](902-aot-pipeline.md#9210-优化级别)）。`debug` 构建配置的构建总是带有调试信息。
- `--remap-path-prefix <from>=<to>` 在构建产物中，把目录 `<from>` 下的源文件路径写成在 `<to>` 之下：panic 位置、调试信息和 `.nabihir` 文件。诊断保留真实路径。相对路径按从工作目录出发的绝对路径匹配，前缀按完整的路径分量匹配，其余部分用 `/` 拼接到 `<to>` 上（`<to>` 为空时保持相对路径），多条规则匹配时，最后给出的那条生效。可重复。标准库的源文件总是写在 `/nabi/library` 之下（`/nabi/library/core/src/...`），因此程序在每台机器和每次安装上给出的路径都相同。包的 `release` 构建配置构建还会把包自身的文件写成相对于其目录的路径（`src/main.nabi`），把源代码依赖的文件写成在其名称之下（`json/src/lib.nabi`），因此在两个地方构建同一个包会得到相同的字节；`--remap-path-prefix` 规则排在这些规则之后，并优先生效。
- `-C <option>` 也可以写作 `--codegen <option>`。
- `-C overflow-checks[=yes|no]` 开启或关闭定宽整数运算的溢出检查（panic）。在每个构建配置中默认开启；`nabi.nabion` 的构建配置可以关闭它（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)），`-C overflow-checks` 会覆盖该设置。省略值表示 `yes`。不可能溢出的常量操作数运算会跳过检查。
- `-C icf[=yes|no]` 开启或关闭链接时的相同代码折叠（[9.2.15](902-aot-pipeline.md#9215-输出)）。在 `release` 构建配置中默认开启，在 `debug` 构建配置中默认关闭。省略值表示 `yes`。
- `-C target-cpu=<cpu>|native` 指定一个 CPU，代替三元组的基线（[0.6.4](../000-meta/006-platform-targets.md#064-simd)）；`-C target-feature=+f1,-f2,..` 开启或关闭单个特性（`+avx2` 等），每个写作 `+feature` 或 `-feature`，同时连带开启它们所需的特性（`+avx2` 会开启 `avx`），或连带关闭需要它们的特性（`-avx` 会关闭 `avx2`）。代码生成器在 `x86_64` 上使用 `avx2`、`popcnt`、`lzcnt` 和 `bmi`，在 `aarch64` 上使用 `lse`、`rcpc` 和 `dotprod`，在 `s390x` 上使用 `vector` 和 `vector-enhancements-1`，在 `riscv64` 上使用 `v`，在 `riscv64` 和 `riscv32` 上使用 `zbb`，并且对 `armv7` 上的 `-neon`、`powerpc64le` 上的 `-vsx` 和 `wasm32` 上的 `-simd128` 关闭向量循环（[8.7.8](../008-abi/807-code-generation.md#878-循环向量化)）；如果生成代码的构建或运行使用了不改变上述任何一项的选项，会发出警告。未知的 CPU 名称是错误，错误信息会列出已知的名称。只有当目标是构建机器所在平台时才接受 `native`，它会加上本机具有的特性。
- `-C subsystem=console|windows` 选择 Windows 子系统：`console`（默认，`mainCRTStartup`）或 `windows`（GUI，`WinMainCRTStartup`；[9.5.9](905-symbol-mangling.md#959-main-函数的符号)）。
- 其他任何 `-C` 选项都是无效选项（E0132，见下文）：``unknown codegen option `<option>` ``，如果有最接近的选项则给出它，并附带列出所有选项的帮助。

`nabic build` 像 `nabic check` 一样报告包的 lint 诊断（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）：打印警告，构建继续；`deny` 级别的 lint 会使构建失败。如果自上次构建以来没有任何改变，`nabic build` 打印 `Fresh`，再次打印那次构建的警告，并跳过构建（[9.2.16](902-aot-pipeline.md#9216-增量构建)）。

### `nabic emit`：中间输出

```
nabic emit [path] [--emit=<kind>[,<kind>..]]
```

等价于 `nabic build --emit=..`。省略 `--emit` 时为 `mir`。

### `nabic test`：测试

`nabic test` 收集测试函数并构建测试框架；`nabi test`（[9.10](910-nabi-frontend.md)）运行它。

```
nabic test [path] [filter]
```

它收集标有 `#[test]` 的函数，并构建一个测试可执行文件（测试框架）。`#[test]` 隐含 `#[cfg(test)]`（[6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-测试函数标记)），因此测试函数不属于普通构建。测试既从 `src/`（位于 `#[cfg(test)]` 之下）收集，也从 `tests/` 目录收集（[0.3.5](../000-meta/003-source-units.md#035-测试与基准测试目录)）。测试构建使用一个选择测试的入口点，而不是包的入口点。

`nabic` 不运行测试框架。`--emit-harness <file>` 把构建出的可执行文件及其中的测试（即 `nabi test` 运行的清单）写入 `<file>`；它的第一行是清单格式 `version 2`。运行器拒绝更新的版本：``the test harness is from a newer nabic (format version N); run the nabi of the same toolchain, e.g. `nabi +<version> test` ``。不使用 `--emit-harness` 时，构建出的可执行文件的名称打印在标准错误上。运行测试框架（选择测试，以及下文的进程隔离、并行和报告）是 `nabi test` 运行器的工作（[9.10](910-nabi-frontend.md)）。

如果只有一个实参，且它不是已存在的路径，则把它当作 `filter`。`filter` 匹配包括模块路径在内的测试名（`math::adds`）。

- 它是一个小型正则表达式，使用 `^` `$` `.` `*` `+` `?` 和 `\` 转义。搜索不锚定，因此不带 `^` / `$` 时它可以匹配名称的任何部分。
- 不含这些字符时，它等价于普通的子串搜索。

**每个测试在自己的进程中运行**。如果一个测试 panic（退出码 101）或以未捕获的错误结束（退出码 1），只有该测试失败，其余测试继续运行。被信号终止也算失败，运行超过时间限制（`--timeout`，默认 600 秒）的测试同样算失败：它会被杀死并报告为超时。`#[should_panic]` 测试只有在 panic 时才通过；`#[ignore]` 测试报告为已忽略，不运行（[6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-测试函数标记)）。

#### 文档测试

对带有库的包，`nabic test` 还会构建其文档测试：库的 `///` 和 `//!` 文档注释中的围栏代码块，以及包根目录下 `README.md` 中的 ```` ```nabi ```` 块。在文档注释中，没有标注语言的围栏也是 Nabi（`nabi doc` 也这样显示）；其他任何语言的块（```` ```text ````、```` ```nabion ````）都不是测试。位于构建的 `#[cfg]` 所排除的模块中，或位于被排除的项上的块（没有该特性时的 `#[cfg(feature = "table")]`），也不是测试。

每个块都是使用该库的代码，就像依赖它的程序那样：该库以包名作为它的依赖（`stats::mean(x)`），并且每个块开头相当于有 `use stats::*;`，因此库的公开项不需要路径（`mean(x)`）。块开头的 `use` 项留在模块级；其余部分是一个可以抛出错误的函数的主体（`try` 不需要 `catch` 即可使用），其中的项是局部项。声明了 `fn main` 的块会调用它。围栏中 `nabi` 之后的词以 `,` 或空格分隔，用来改变“通过”的含义：

| 围栏 | 该块 |
|-------|-----------|
| ```` ```nabi ```` | 构建并运行；返回时通过 |
| ```` ```nabi,norun ```` | 构建，但从不运行（读取文件、等待或需要网络的代码） |
| ```` ```nabi,should_panic ```` | 只有在 panic 时才通过 |
| ```` ```nabi,compile_fail ```` | 只有在构建失败时才通过；`compile_fail,E0040` 还要求出现该错误码 |
| ```` ```nabi,ignore ```` | 既不构建也不运行，并报告为已忽略 |

其他词会触发警告，警告中列出可用的词。以 `# ` 开头的行（或单独的 `#`）属于测试的一部分，但 `nabi doc` 不显示，用于读者不需要看的准备代码：

````nabi
/// ```nabi
/// # let data = [1.0, 2.0, 3.0];
/// assert(stats::mean(data) == 2.0);
/// ```
````

文档测试的名称是 `doc <file>:<line>`，其中文件相对于包，行是其开头围栏所在的行；如果块位于某个项上，后面再跟上被文档化的项：`doc src/stats.nabi:12 (mean)`、`doc README.md:20`。过滤器、`--list` 和记录对这些名称的处理与其他名称相同。一个包的所有块构建成一个可执行文件，排在库自身的测试之后；每个 `compile_fail` 块在构建测试框架时单独检查。块中的代码保留其行号，因此其中的错误会指出被文档化文件中的行。无法构建的块作为单独的用例失败，其错误随失败一起显示，其他块不受影响，照常构建并运行。

`nabic test --doc` 只构建文档测试，`--no-doc` 一个都不构建；`nabi test` 会传递这两个选项。

选项为 `nabic build` 的选项以及 `--emit-harness`。同时运行多少个测试、通过的测试是否显示其输出以及输出形式，都是运行器 `nabi test` 的选项（[9.10.1](910-nabi-frontend.md#9101-子命令与全局选项)）。

运行器以如下形式打印到 stdout。

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

被忽略的测试显示为 `test <name> ... ignored`（如果有原因，则加上 `, <reason>`）。只要有测试失败，`nabi test` 就以 `1` 退出。

在两个根文件都有的包中，先运行库的测试，再运行可执行文件的测试（针对不含其测试的库构建），然后运行文档测试；某次运行失败即结束该命令。`bench` 的工作方式相同。

### `nabic bench`：基准测试

```
nabic bench [path] [filter]
```

收集标有 `#[bench]` 的函数，并以优化构建来构建它们（未给出 `-O` 时为 `-O2`）；显式的 `-O0` 或 `-O1` 会像 `nabic test` 那样构建带调试信息的 `debug` 构建配置。`nabi bench`（[9.10](910-nabi-frontend.md)）运行测试框架并测量它们（[9.9.11](909-debug-tools.md#9911-基准测试)）。与 `#[test]` 一样，`#[bench]` 隐含 `#[cfg(test)]`，基准测试既从 `src/` 收集，也从 `benches/` 目录收集（[0.3.5](../000-meta/003-source-units.md#035-测试与基准测试目录)）。`path`、`filter` 和 `--emit-harness` 的规则与 `nabic test` 相同。

基准测试接受一个 `Bench` 句柄并对一个闭包计时，因此准备工作不计入测量：

```nabi
#[bench]
fn parse_1kb(b: bench::Bench) {
    let input = sample(1024);                   // setup: run once, not timed
    b.set_bytes(1024);                          // optional: report throughput
    b.iter(|| parse(hint::black_box(input)));   // the timed region
}
```

`b.iter(f)` 执行整个测量过程（校准、预热、采样）；其结果会经过 black box 处理。`hint::black_box`（[`core::hint`](../../refs-standard-libraries/200-core/README.md#基准测试--优化器提示)）是一个优化器无法看穿的恒等函数。把它用在计时区域的输入和结果上，使这些工作既不会被提升到外面，也不会被折叠掉。`b.set_bytes(n)` / `b.set_items(n)` 会增加一列吞吐量。`Bench`、`black_box` 和统计功能属于库 API；测量方法在 [9.9.11](909-debug-tools.md#9911-基准测试) 中定义。

每次采样的调用次数、采样数和输出形式都是运行器 `nabi bench` 的选项（[9.10.1](910-nabi-frontend.md#9101-子命令与全局选项)）。

基准测试一次运行一个。无论是否输出到终端，每个基准测试都打印一行以人类可读单位表示的对齐行：

```
  json::parse_1kb     1.84 us +/- 0.03 us   543 MiB/s   [2 outliers, 0 gc]
```

`nabi bench --format=nabibench` 则为回归工具打印一种稳定的机器格式，每个基准测试一行：`#nabibench name=<name> median_ns=<f> mad_ns=<f> samples=<n> outliers=<n> gc=<n> bytes=<n> items=<n>`。这一行按现状冻结：不增加、删除或重排任何键，因此为它编写的读取程序可以一直工作。新字段只加入 `--format=json` 的 JSON 记录中，JSON 记录还会把基准测试自身的输出与这些记录分开（使用 `nabibench` 时，它与这些行共用 stdout）。

从不调用 `b.iter` 的基准测试什么也没有测量。这不算失败：它的行是 `` <name>  warning: measured nothing: time the code with `b.iter(|| ..)` ``，汇总把它计为 `<n> measured nothing`，而不计为已运行。

### `nabic check`：诊断

```
nabic check [path | -] [--check-library]
```

运行前端，直到类型检查和 MIR 构建，再加上 lint，然后执行不含原生构建的代码生成（不生成目标文件，不链接），并报告诊断。选项与 `nabic build` 相同。`--check-library` 不论可达性如何，都降级并检查库的每个主体，并对照约束检查每个泛型主体；`nabic build --check-library` 还会以构建的优化级别把它们全部编译进可执行文件（否则构建只编译程序可达的库代码，[9.3.6](903-package-archive.md#936-标准库)）（[2.10.9](../002-types/210-generics.md#2109-主体检查)）；此时它的 lint 也覆盖标准库自身的代码，`missing_docs` 为 `deny` 级别（[6.2.9](../006-meta-system/602-built-in-attributes.md)）。两个根文件都有的包先检查其库，如果库没有错误，再检查编译进该库的可执行文件；每条警告只报告一次。

## 9.1.3. `--print` 选项

### `nabic --print cfg`：生效的 cfg

```
nabic --print cfg [--target=<target>] [-O<n>] [--cfg ..]
```

打印这些选项所描述的构建的 cfg（[6.3](../006-meta-system/603-cfg.md)），每行一个：标志写作 `flag`，键写作 `key="value"`（与 rustc 相同）。键中包括 `target_env`，并且在 `-O0` 下设置 `debug_assertions`。

```
debug_assertions
target_arch="aarch64"
target_endian="little"
target_family="unix"
target_os="macos"
target_pointer_width="64"
```

### `nabic --print target-list` / `target-spec`：目标

```
nabic --print target-list
nabic --print target-spec [--target=<target>]
```

`target-list` 打印受支持的目标名称，每行一个；进行中的目标（[0.6.2](../000-meta/006-platform-targets.md#062-进行中)）不列出。`target-spec` 以 NABION 打印该目标的规格（[0.6.5](../000-meta/006-platform-targets.md#065-目标规格)）；不给出 `--target` 时打印主机的规格。

### `nabic --print lldbinit`：lldb 格式化器

打印一行 `command script import '<path>'`，用于加载本工具链的 lldb 格式化器（[9.9.6](909-debug-tools.md#格式化器)）：`<sysroot>/lib/nabi/lldb/nabi.py`。加载它们还会把标准库的 `/nabi/library` 路径映射到工具链的库源代码（`target.source-map`）。

`--print gdbinit` 打印 `source <path>`，即加载 gdb pretty-printer 的 gdb 命令，路径为 `<sysroot>/lib/nabi/gdb/nabi.py`（它们用 `set substitute-path` 映射库路径）。`--print natvis` 打印 natvis 可视化器的路径 `<sysroot>/lib/nabi/natvis/nabi.natvis`。

## 9.1.4. 辅助子命令

### `nabic version`：版本

```
nabic version
nabic --version            # = -V
nabic -V -v                # adds host line
```

向 stdout 打印一行 `nabic <version>`；如果 `nabic` 是从 git 检出构建的，后面再跟上 ` (<commit> <date>)`。使用 `-v` 时，追加一行 `host: <triple>`。

### `nabic help`：帮助

```
nabic help [subcommand]
nabic <subcommand> --help
```

不带子命令时，打印完整的参考：每个子命令、选项和环境变量。给出子命令时，打印该子命令自身的帮助：其用法形式、简短说明、一个或多个示例以及它接受的选项。两者都不打印属于其他子命令的文本。

## 9.1.5. 全局选项

全局选项如下。它们可以出现在子命令之前或之后。

| 选项                           | 含义                                                     |
|----------------------------------|-------------------------------------------------------------|
| `--verbose` / `-v`               | 显示各阶段耗时（[9.9.5](909-debug-tools.md#995-日志--进度显示)） |
| `-vv`                            | 更详细的日志：`Reading <file>` 和 `Specialized <path>(sig)` 行（[9.9.5](909-debug-tools.md#995-日志--进度显示)） |
| `--quiet` / `-q`                 | 不打印进度行（仍然打印诊断） |
| `--color=auto\|always\|never`    | stdout 和 stderr 上的颜色（9.1.8）                         |
| `--no-color`                     | 等价于 `--color=never`                               |
| `--cwd=<path>`                   | 运行前切换工作目录                     |
| `--print cfg\|target-list\|target-spec\|lldbinit\|gdbinit\|natvis` | 打印生效的 cfg、受支持的目标名称、某个目标的规格，或加载 Nabi 调试器格式化器的内容（lldb、gdb、natvis；9.1.3） |
| `--message-format=human\|json`   | 诊断格式。`json` 每条诊断向 stdout 写一行（9.1.8） |
| `--explain <code>`               | 打印某个错误码（`E0040`）的说明页：它的含义以及应当改写成什么，然后退出（[9.9.4](909-debug-tools.md#错误码)） |
| `-A` / `-W` / `-D <lint \| group>` | lint 级别（`--allow` / `--warn` / `--deny`）。适用于每个编译该包的命令（`build`、`test`、`bench`、`emit`、`check`，以及 `nabi` 的 `run`、`lint`、`repl`、`jupyter`、`lsp`）。可重复；后给出的生效 |

`nabic help` 把 `--print` 的各种形式与构建选项列在一起，因为它们打印的内容取决于 `--target` 和 `--cfg`；它们不需要子命令。

lint 级别按以下顺序确定：lint 的默认级别 < `-A` / `-W` / `-D` < 代码中的 `#[allow(..)]` / `#[warn(..)]` / `#[deny(..)]`（最内层的生效）。`nabi` 把清单中的 `lints` 作为这些标志放在前面（[9.10.1](910-nabi-frontend.md#9101-子命令与全局选项)）。

stdout 和 stderr 的颜色分别按以下顺序决定：`--color=never` 关闭颜色，`--color=always` 开启颜色；否则（`--color=auto`，默认），非空的 `NO_COLOR` 关闭颜色，非空且不为 `0` 的 `CLICOLOR_FORCE` 开启颜色，`TERM=dumb` 关闭颜色，其他情况下，流是终端时才着色。在 Windows 上，不接受转义序列的控制台得到纯文本。

示例：

```
nabic --verbose build
nabic test -q
nabic --cwd=examples/hello check
```

## 9.1.6. 退出码

退出码定义如下。

| 代码  | 含义                                              |
|-------|------------------------------------------------------|
| `0`   | 成功                                              |
| `1`   | 用户代码错误（编译错误、测试失败等） |
| `2`   | 用法错误（无效选项、未知子命令等） |
| `70`  | nabic 内部错误（panic）                          |

用法错误先打印 `nabic: <message>`，然后打印一行 ``run `nabic help <subcommand>` for usage``（没有给出已知子命令时为 ``run `nabic help` for usage``）。消息会说明应当改写成什么：未知的子命令或选项会在有最接近的名称时给出它（``unknown option `--relase`; did you mean `--release`?``），而指向某个文件的未知子命令（包含 `/` 或以 `.nabi` 结尾）会建议 `nabic build <file>`。当 `__NABI_CLI` 为 `nabi`（9.1.7）时，这两行都写 `nabi` 而不是 `nabic`。

已知选项被赋予它不接受的值时，报告为诊断 `E0132`（[9.9.4](909-debug-tools.md#错误码)），退出码仍为 `2`：未知的 `-C` 选项或 `--emit` 种类、缺少值的选项、未知的 `-C target-cpu`、`--print`、`--color` 或 `--message-format` 不接受的值、格式错误的 `-l`、`--extern`、`--extern-feature` 或 `--extern-dep`、`--explain` 没有的错误码，或者 `-o` 与多种 `--emit` 一起使用。它的 `= help:` 行说明应当写什么（``error[E0132]: unknown emit kind `mirr`; did you mean `mir`?``），后面跟着 `run .. for usage` 行。无论 `--color` 和 `--message-format` 在命令行中的什么位置，它都按它们的设置打印：在 `--message-format=json` 下，它是标准输出上的一行 JSON，没有那一行。

sysroot 不来自 `NABI_HOME`：它相对于可执行文件确定。`nabic` 读取 `NABI_HOME` 只是为了显示已获取依赖在 `$NABI_HOME/cache/` 下的路径。如果 nabic 自身 panic（编译器内部错误），它会打印 panic 消息，然后打印 `nabic: internal compiler error` 和三行 `note:`：说明这是 nabi 的 bug 以及到哪里附上触发它的代码进行报告、版本和主机（`nabic <version> (<commit> <date>) on <triple>`），以及命令行；如果当时位于某个编译器 pass 中，还有第四行 `note: in pass: <pass> > <pass>`，给出它所在的编译器 pass（名称与 `NABI_TIME_PASSES` 中的相同）。它以 `70`（`EX_SOFTWARE`）退出；程序的 panic 则不同，以 `101` 退出。`nabi` 以同样方式报告它自己的内部错误（`nabi: internal error`，退出码 `70`）。

如果标准输出或标准错误是一个读取端已经消失的管道，`nabic` 会不加消息地停止：在 Unix 上它被 `SIGPIPE` 终止（shell 报告 `141`）；在 Windows 上它以 `1` 退出。这不是内部错误。

## 9.1.7. 环境变量

影响 `nabic` 的环境变量如下：

| 变量          | 含义                                                                                  |
|-------------------|------------------------------------------------------------------------------------------|
| `NABI_CC`          | 可选的 C 编译器驱动程序，代替 `nabic` 的链接器执行最终链接（[9.2.14](902-aot-pipeline.md#9214-链接可执行文件)） |
| `NABI_VERBOSE`     | 等价于 `--verbose`                                                                |
| `NABI_TIME_PASSES` | 为 `1` 时，在 stderr 上输出每个 pass 的实际耗时（[9.9.5](909-debug-tools.md#nabi_time_passes)） |
| `__NABI_ENV`、`__NABI_ENV_LIBS` | `nabi` 从某个环境中运行脚本时设置（内部使用）：环境的名称，以及安装在其中的以逗号分隔的库。使用了其中某个库却没有声明它的脚本会得到一条指出该库的提示 |
| `__NABI_CLI`       | `nabi` 运行 `nabic` 时由 `nabi` 设为 `nabi`（内部使用）：用法错误（9.1.6）、`--help` 以及错误和警告行的 `nabic:` 前缀都写 `nabi` 而不是 `nabic` |
| `NABI_TARGET_DIR`  | `target/` 的位置（默认：在包内）                                      |
| `NABI_INCREMENTAL` | 前端保存包的解析结果和降级后主体的位置（默认 `target/<profile>/incremental/`；`0`：不保存）；库收集到的项保存在 `lib/nabi-cache/library/` 中（[9.2.16](902-aot-pipeline.md#9216-增量构建)） |
| `NABI_INCREMENTAL_VERIFY` | `1`：再次降级每个主体，出现差异时失败（[9.2.16](902-aot-pipeline.md#9216-增量构建)） |
| `NABI_FN_CACHE` | `0`：既不读取也不写入工具链的函数缓存，即构建所保存的标准库代码（[9.2.16](902-aot-pipeline.md#9216-增量构建)） |
| `NABI_CACHE_MAX_SIZE` | 工具链缓存的上限：字节数，或带 `K`、`M` 或 `G` 的数（默认 `5G`）；`off`：无上限（[9.2.16](902-aot-pipeline.md#9216-增量构建)） |
| `NABI_HOME`        | 已获取依赖的显示位置（9.1.6；默认 `~/.nabi`）                           |
| `NABI_RUNNER`      | `<program> [args]`：`nabi run` / `test` / `bench` 在其下运行其他目标的可执行文件的程序（[9.10.8](910-nabi-frontend.md#其他目标的可执行文件)） |
| `LIBRARY_PATH`    | 原生库搜索目录，排在 `-L` 之后                                            |
| `NO_COLOR`        | 非空：`--color=auto` 不使用颜色（9.1.5）                                    |
| `CLICOLOR_FORCE`  | 非空且不为 `0`：即使流不是终端，`--color=auto` 也使用颜色，除非设置了 `NO_COLOR`（9.1.5） |
| `COLORTERM`       | `truecolor` 或 `24bit`：颜色使用 24 位值（9.1.8）                                |
| `TERM`            | `dumb`：`--color=auto` 不使用颜色；包含 `256color`：颜色使用 256 色调色板（9.1.8） |

未设置或为空的变量取其默认值。取值不在所列范围内时，在 stderr 上打印一行警告，指出变量、值和可接受的值，该变量取其默认值。

`nabi` 把内部变量 `__NABI_CLI`、`__NABI_ENV` 和 `__NABI_ENV_LIBS` 交给 `nabic`；`nabic` 不读取它们的其他拼写。`nabi run`、`nabi test` 和 `nabi bench` 会从它们启动的程序的环境中删除每个名称以 `__NABI` 开头的变量（在 Windows 上，不论名称的大小写）；程序看到的其余环境保持不变。

## 9.1.8. 输出格式

### 诊断消息

诊断消息按以下格式打印。

```
error[E0027]: cannot find value `count` in this scope
  --> src/main.nabi:5:13
   |
 5 |     let x = count + 1;
   |             ^^^^^
   |
   = help: did you mean `counter`?
```

诊断使用[源文件](../001-lexical/101-source-files.md)中定义的“文件 + 行 + 列”单位。

头部为 `<severity>[<code>]: <message>`，`--> path:line:col` 给出主要位置。主要区间用 `^` 标出下划线，次要区间用 `-`；位于另一个文件中的次要区间以 `::: path:line:col` 引出。末尾的 `= note:` 行补充上下文，`= help:` 行建议修复方法。消息不引用本规范的章节；JSON 格式会列出它们（[9.9.4](909-debug-tools.md#994-诊断--错误消息)）。

### 构建进度

进度以右对齐的动词显示在 stderr 上。使用 `-q` 时什么也不打印。

```
   Compiling json v0.1.0
   Compiling foo (path/to/foo)
    Building target/debug/foo
    Finished in 1.23s
     Running `target/debug/foo`
```

| 动词        | 何时                                          |
|-------------|-----------------------------------------------|
| `Compiling` | 前端开始（包名和目录）；`check` 和 `lint` 时为 `Checking`，`doc` 时为 `Documenting`。`nabi` 获取的依赖（位于 `$NABI_HOME/cache/` 下）改为显示 `<name> v<version>` |
| `Building`  | 生成可执行文件 / 归档文件（`build`）       |
| `Built`     | 单个文件的可执行文件已复制到其旁边（`build`）：`Built hello (use -o to put it elsewhere)`。不使用 `--emit-harness` 构建的测试或基准测试框架（`test`、`bench`），后面跟一条说明，指出用 `nabi test` 或 `nabi bench` 运行它 |
| `Wrote`     | 写出了一个 `--emit` 输出                   |
| `Reused`    | 使用 `-v` 时，在前端之后：有多少个文件的解析结果和多少个主体的降级结果来自增量缓存（[9.2.16](902-aot-pipeline.md#9216-增量构建)） |
| `Units`     | 使用 `-v` 时，在代码生成期间：有多少个代码单元被复用（来自增量缓存或函数缓存），有多少个被编译 |
| `Finished`  | 构建完成（自命令开始以来的时间） |
| `Running`   | 运行程序（`run`）：相对于当前目录的可执行文件；单个文件按其文件名显示（`Running hello.nabi`），使用 `-v` 时显示缓存的可执行文件 |
| `Created`   | 已创建包（`new`），后面是在其下接着要运行的命令（[9.10.2](910-nabi-frontend.md#9102-nabi-new--nabi-init创建包)） |
| `Removed`   | 已删除 `target/`（`clean`）                   |
| `Fresh`     | 已是最新；后面跟着那次构建的警告      |
| `Checked`   | `check` 完成（错误数和警告数）       |
| `Generated` | 已写出文档（`doc`）                 |
| `Formatted` | 已重写文件（`fmt`）：在最后给出其数量（[`nabi fmt`](910-nabi-frontend.md#nabi-fmt格式化工具)） |

在前端失败的构建（`build`、`run`、`test`、`bench`、`doc`）在诊断之后以 `error: could not build <name> (<n> error(s), <m> warning(s))` 结束，计数方式与 `check` 相同。`-q` 和 `--message-format=json` 不输出这一行。

`nabi` 以同样方式打印它自己的进度：`Added`、`Installed`、`Updated`、`Downloading` 等（[9.10](910-nabi-frontend.md)）。

### 颜色

开启颜色时（9.1.5），输出使用以下调色板。其色深随终端而定：`COLORTERM` 为 `truecolor` 或 `24bit` 时使用 24 位值，否则在 `TERM` 包含 `256color` 时使用 256 色调色板，再否则使用 16 种标准颜色。

| 部分                                  | 颜色                   | 24 位      | 256   | 16             |
|---------------------------------------|--------------------------|-------------|-------|----------------|
| 状态动词（`Compiling`）、列表标记 `*` | 天蓝，粗体            | `#38bdf8`   | `81`  | 亮青色    |
| `error`                               | 红色，粗体                | `#ff5f5f`   | `203` | 亮红色     |
| `warning`                             | 琥珀色，粗体              | `#e0b057`   | `179` | 黄色         |
| `note`                                | 天蓝                      | `#38bdf8`   | `81`  | 亮青色    |
| `help`                                | 浅粉色               | `#f9a8d4`   | `218` | 亮品红 |
| 诊断边栏（`-->`、`\|`、行号） | 天蓝，粗体         | `#38bdf8`   | `81`  | 亮青色    |

颜色从不单独承载含义：无论有没有颜色，都会打印标签（`error:`、`warning:`、`note:`、`help:`）和状态动词。标记某一项的列表（`nabi toolchain list`、`nabi env list`）在该项前面加 `*`，在后面加 `(default)` 或 `(active)`。管道输出、`--message-format=json` 和 `-q` 不受影响。

### JSON 输出（用于工具集成）

为了便于工具集成，输出也可以生成为 JSON。

```
nabic build --message-format=json
```

在这种模式下，每条诊断成为 stdout 上的一行 JSON 对象，便于 IDE 和 CI 工具解析。对象的结构在 [9.9.4](909-debug-tools.md#994-诊断--错误消息) 中定义。

## 9.1.9. 标准输入

在实参位置写 `-`，如 `nabic check -`，表示标准输入，用于管道或脚本。来自标准输入的源代码作为匿名可执行包的 `main.nabi` 进行检查。

## 9.1.10. 与外部工具的兼容性

`nabic` 是一个低层编译器；依赖解析是前端 `nabi` 的工作（[9.10](910-nabi-frontend.md)）。`nabi` 调用 `nabic`，通过 `--extern` 传入已解析的依赖包的根（[9.2.3](902-aot-pipeline.md#923-依赖解析)）。其他构建系统可以以同样方式直接调用 `nabic` 来集成。破坏 CLI 兼容性的变更按 SemVer 规则管理。
