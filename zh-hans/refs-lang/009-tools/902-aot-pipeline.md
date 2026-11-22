# 9.2. AOT 构建流水线

本章逐步定义 `nabic build` 的流程。它把[执行引擎与用户界面](../000-meta/001-execution-modes.md)和[编译流水线](../000-meta/002-pipeline.md)给出的整体图景落实到具体的命令和文件流转上。

## 9.2.1. 步骤

构建分 11 个步骤进行，全部在 `nabic` 内部完成。

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

> 依次为：包发现、依赖解析、源代码解析、名称 / 可见性解析、类型检查、nabi-hir 构建、特化（收集调用点并单态化）、nabi-mir 构建、生成目标文件的代码生成、归档或链接，最后是输出：一个 `.a` 归档文件或一个可执行文件。

每个步骤的输入、输出和职责都划分得清清楚楚。

## 9.2.2. 包发现

从 `nabic build [path]` 的路径出发向上查找，定位包含 `src/main.nabi` 或 `src/lib.nabi` 的目录（如果 path 是文件，就从其所在目录开始）；查找每次向上移动一级目录。

如果找到，该目录就是包根，其 `src/` 就是包的源代码（见[源单元](../000-meta/003-source-units.md)）。其他情形：

- 如果 path 是位于所找到的包的 `src/` 之外的文件（放在包源代码旁边的脚本或工具），或者向上查找没有找到包，那么它是一个只由这一个文件构成的匿名可执行包（[4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)）。它的文件子模块位于它旁边。
- 如果 path 是目录且没有包，则是错误。

包名是包目录的名称（匿名包则是去掉 `.nabi` 的文件名），并成为输出名（9.2.15）。同时具有两个根文件的包按两个目标构建：先构建其库，再构建其可执行文件，可执行文件把该库作为依赖 `<name>` 挂载（[0.3.1](../000-meta/003-source-units.md#library-and-executable-in-one-package)）。

## 9.2.3. 依赖解析

此步骤处理[路径](../005-modules/507-paths.md)和 [`use`](../005-modules/505-use.md) 中的外部包引用（`use external_pkg::...`）。

`nabic` 本身不解析依赖。前端 `nabi`（[9.10](910-nabi-frontend.md)）按照 `nabi.nabion` / `nabi.lock` 准备 path / git 依赖包，并通过 `--extern` 把每个包的根传给 `nabic`。

```
nabic build --extern util=../util --extern parser=target/debug/deps/libparser.a
```

`--extern name=<path>` 把一个依赖包挂载到顶层路径 `name`。`name` 是把 `nabi.nabion` 中 `deps` 的键里的 `-` 替换为 `_` 得到的标识符。路径有两种形式。

| 路径                              | 依赖包                                                           |
|-----------------------------------|------------------------------------------------------------------------------|
| `<dir>` / `<dir>/src/lib.nabi`     | 源代码。该 `src/lib.nabi` 在同一次构建中一并编译            |
| `<name>.nabihir` / `lib<name>.a`     | 预构建的包。给出其中任一个，就会找到同一目录中的另一个文件。读取 NABIHIR（[9.4](904-nabihir-bytecode.md)）获得声明和泛型，并链接归档文件 |

- 用户代码通过 `use name::..` 使用该包的项。
- 依赖包内部的 `pkg::` 指该依赖包自身。
- 依赖包是独立的包。从外部只能使用 `pub` 项；`pub(pkg)` / `pub(sub)` / 私有项不可使用（[5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)）。
- 用户包根部的模块与构建中的某个包同名是错误。
- 如果后出现的 `--extern` 的路径与之前某个 `--extern` 的路径相同，它就是同一个包的另一个名称（不同的包可以用不同的名称称呼同一个依赖，[9.10.2](910-nabi-frontend.md)）。该包只以第一个名称构建并挂载一次；以另一个名称开头的路径（`other::..`，出现在如此称呼它的包的代码和元块模板中）也指向它，因此它的项和类型在每个名称下都相同。
- `--extern-dep <package>=<name>` 表示包 `<package>`（一个 `--extern` 名称，或代表正在构建的包的 `pkg`）依赖外部包 `<name>`；`nabi` 为每个包的每个依赖各传一个。只要有任何一个，每个包就只能以它自己给出的名称看到它所依赖的外部包（[5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)）；没有任何 `--extern-dep` 提到的外部包（即 `nabic` 挂载的、包自身的库）属于正在构建的包，并依赖该包所依赖的一切。如果一个也没有，每个包都能看到所有外部包。

对含有 `src/lib.nabi` 的包执行 `nabic build` 会生成 `lib<name>.a` 和 `<name>.nabihir`，含有 `src/main.nabi` 时生成可执行文件 `<name>`；两者都有的包会生成全部三个文件，可执行文件链接该库。`--name <n>` 设置包名（输出名和挂载名）；默认是目录名。

### 预构建的依赖包

`nabic` 按给出的顺序，把每个源代码形式的 `--extern` 预构建**一次**，放入 `target/[<triple>/]<profile>/deps/`。每个依赖包在它之前的依赖包之后构建，并能看到它所依赖的那些包（`--extern-dep`；没有时则是全部）。后续编译使用这些 `lib<name>.a` 和 `<name>.nabihir`，而不是源代码。

```
target/<profile>/deps/
├── lib<name>.a
├── <name>.nabihir
└── <name>.fp               # fingerprint
```

如果指纹匹配，就不会重新构建。指纹包括：

- `nabic` 的身份（与 NABIHIR 生成者相同的值）
- 构建选项：`--release` / `-O*`、`-g`、`--target`、`--cfg`（不含其 `feature` 键）、`--bootstrap`
- 该依赖的特性（`--extern-feature`），以及它和在它之前构建的包的 `--extern-dep`
- 该依赖包 `src/` 下的所有文件，以及它的 `nabi.nabion`
- 在它之前构建的依赖包的指纹

预构建包的接口按它构建时所用的特性读取（其 NABIHIR 元数据，[9.4.6](904-nabihir-bytecode.md#946-meta)），而不是按使用方的特性。

每个进行编译的命令都走这条路径：`build` / `run` / `test` / `bench` / `check` / `lint` / `doc` / `lsp` / `repl` / `jupyter`、单文件脚本，以及具名环境。REPL 宿主也会链接预构建的归档文件。如果既没有包也没有目标目录（`--target-dir`、`NABI_TARGET_DIR`），它们会放在系统临时目录下的单文件缓存中（9.2.16）。

标准库在每次构建时都从源代码编译，并且只编译程序可达的部分；编译好的函数保存在工具链的函数缓存中（[9.3.6](903-package-archive.md#936-standard-library)）。

## 9.2.4. 源代码解析

所有 `.nabi` 文件按照从[源文件](../001-lexical/101-source-files.md)到[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)的词法 / 语法规则解析为 AST。

[模块](../005-modules/504-modules.md)中的目录模块把它的所有 .nabi 文件合并为一个模块的主体。

包根处的清单 `nabi.nabion`（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）不是模块文件。清单作为单个 [NABION](../../refs-standard-libraries/300-std/308-nabion.md) 值（`Package { .. }`）读取；顶层语句规则不适用于它，它也不会被编译。

### 扫描器预处理

[`#[keyword]` 魔法](../006-meta-system/606-keyword-magic.md)的扫描器预处理也在此步骤中执行。先浅层扫描所有 .nabi 文件，收集 `#[keyword(...)]`、`#[infix(...)]`、`#[literal(...)]` 注册，再把这些注册交给真正的词法分析器 / 解析器。

## 9.2.5. 名称 / 可见性解析

每个路径和标识符都按照 [`use`（导入）](../005-modules/505-use.md)、[可见性](../005-modules/506-visibility.md)、[路径](../005-modules/507-paths.md)和[重新导出（`pub use`）](../005-modules/508-re-export.md)的规则映射到一个项。

此步骤应用隐式的 `core::prelude` 导入（见[标准库层级](../000-meta/005-standard-libraries-tiers.md)；`bare::*` 和 `std::*` 需要显式导入，因此不包括在内），执行可见性检查并把不当访问报告为错误，还检查导入冲突。

## 9.2.6. 类型检查

每个表达式的类型按照 [2. 类型系统](../002-types/README.md)的规则确定。

这包括在渐进槽和静态槽之间作出判定（见 [`any` 槽](../002-types/202-any-slot.md)）、检查 trait 约束（见[泛型](../002-types/210-generics.md)），以及确定分支的结果类型（见 [3. 表达式](../003-expressions/README.md)）。和类型与 `any` 槽的类型收窄也在这里处理。

## 9.2.7. nabi-hir 构建

经过类型检查的 AST 被转换为 nabi-hir（[HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir)）数据结构。

HIR 是一种类型信息丰富的中间表示，反映了上述步骤 4-5 的全部结果。它在结构上经过压缩，相同的类型和标识符共享同一个表示。

## 9.2.8. specialize：单态化

此步骤对[泛型](../002-types/210-generics.md)和[常量泛型](../002-types/211-const-generics.md)进行单态化。

```
1. Collect all call sites.
2. Determine the concrete type tuple of each call.
3. Create a separate specialized HIR per (item_id, type_tuple).
4. Cache. Each (item_id, type_tuple) only once.
```

从程序的根（`main`、导出的符号、`#[export_name]` 项、运行时引用的符号）不可达的函数不会生成。库构建同样如此：从包的 `pub` 项和这些根不可达的函数不会放入归档文件。

正是这一步让 [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 能够不经虚表直接调用（见 [trait 不能作类型](../002-types/212-trait-as-type-banned.md)）。

## 9.2.9. nabi-mir 构建

[特化](../000-meta/007-glossary.md#specialize)后的 HIR 被转换为 nabi-mir（MIR）。

MIR 是比 HIR 低一层的 IR。此步骤完成运算符脱糖（见[运算符分派](../003-expressions/301-operator-dispatch.md)），确定模式匹配的决策树（见 [`match` 表达式](../003-expressions/312-match-expression.md)），并固定闭包和协程的栈帧布局。

## 9.2.10. 优化级别

优化级别决定 nabi-mir 在代码生成之前如何优化（[8.7.3](../008-abi/807-code-generation.md#873-optimization-levels)）；代码生成器在每个级别都相同（[0.2.4](../000-meta/002-pipeline.md#024-code-generation)）。

| 级别         | 效果 |
|---------------|--------|
| `-O0`         | 不优化；每个值都放在自己的栈槽中 |
| `-O1`         | 值经过简化并放在寄存器中；循环按 `-O2` 优化（其中的调用按 `-O2` 内联，因此 `for` 循环的迭代器变为计数循环，并经过分段（strip mining）和向量化），其他地方内联小函数和 `#[inline]` 函数，只在一处被调用的函数在所有地方内联；带调试信息（`debug` 构建配置） |
| `-O2`         | 在循环之外也内联较大的函数体 |
| `-O3`         | 在循环之内也内联较大的函数体，并对保持标量的循环进行展开：一个保持浮点累加和或调用函数的循环，其整段每 2 次或 4 次迭代才检查一次计数器 |
| `-Os`         | 只在函数体与调用差不多小（在循环中为其数倍）时内联，循环仍变为计数循环并分段，函数体较短时进行向量化 |

`-g` 既不改变优化，也不改变代码（8.7.3）：调试信息跟随优化后的代码，内联的函数体作为栈帧，每个变量位于寄存器中、栈槽中或已被优化掉（[9.9.6](909-debug-tools.md#996-debugger-integration-gdb--lldb)）。优化器在包被划分为代码生成单元（9.2.16）之前一次看到整个包，因此它可以跨模块内联，也可以内联进出标准库的函数，而标准库在每次构建时都与程序一起编译（[9.3.6](903-package-archive.md#936-standard-library)）。在不同级别构建的目标文件遵循相同的运行时 ABI（[8](../008-abi/README.md)），可以链接在一起，包括在另一个级别构建的预构建依赖。REPL 按 `:opt` 设置的级别编译单元（[9.9.9](909-debug-tools.md#999-repl--jupyter-debugging)），但不优化其 nabi-mir。

## 9.2.11. 代码生成器

代码生成器（[8.7](../008-abi/807-code-generation.md)）经由所有目标共用的一种低层形式，直接从 nabi-mir 生成机器码和目标文件，支持 `x86_64`、`aarch64`、i686、armv7、riscv64、riscv32、powerpc64le、s390x 和 wasm32。使用 `-g` 时，它生成 DWARF：函数、行表、参数、局部变量及其类型，在优化后的代码中还包括内联的函数体以及每个变量按代码范围所在的位置（[9.9.6](909-debug-tools.md#996-debugger-integration-gdb--lldb)）；对于 Windows 目标，它在目标文件中生成等价的 CodeView（`.debug$S`、`.debug$T`），由链接器汇集成 PDB（9.2.14）。使用了它尚不支持的功能的代码会产生 `E0129` 错误，并指出该功能。

## 9.2.12. 目标文件

每个目标的目标文件都由 `nabic` 写出：Apple 目标为 Mach-O，Linux、FreeBSD 和 riscv32 为 ELF，Windows 为 COFF，wasm32 为 wasm 目标文件。一次构建的代码生成单元（9.2.16）并行编译，每个单元生成一个目标文件。`--emit=obj` 写出目标文件而不进行链接（[9.9.1](909-debug-tools.md#991-intermediate-output---emit)）。

```
target/debug/build/foo.o
```

## 9.2.13. 归档文件

库包把它的目标文件（每个代码生成单元一个，9.2.16）放进由 `nabic` 自己写出的标准 `ar` 归档文件，并在旁边写出接口文件 `foo.nabihir`（[NABIHIR 格式](904-nabihir-bytecode.md)）。

```
target/debug/libfoo.a
target/debug/foo.nabihir
```

## 9.2.14. 链接（可执行文件）

可执行文件由 `nabic` 自己的链接器链接：

```
foo.o + <deps>/lib*.a + <sysroot>/lib/libnabir.a + target libc  →  target/debug/foo
```

Mach-O 可执行文件链接 macOS SDK（Xcode Command Line Tools）。Windows 可执行文件链接 `nabic` 根据其自带的模块定义（源自 mingw-w64）构建的导入库，并在 UCRT 之上使用 Nabi 自己的启动代码；不需要 MSVC 工具或 Windows SDK。Nabi 的 Windows 启动代码在 `main` 之前运行 `.CRT$XCA`..`.CRT$XCZ` 中的 C 初始化函数。对于 x86 Windows，导入名称来自 mingw-w64 的 `lib32` 定义，带有 stdcall 的 `@N` 修饰，映像以 `/SAFESEH:NO` 链接。Linux 目标无论是本机构建还是交叉构建，总是链接 `<sysroot>/lib/nabi/targets/<triple>/` 中随附的 C 库：musl 静态链接，glibc 通过链接桩链接（可执行文件需要 glibc 2.34 或更高版本）。其他库通过 `-L` 和 `LIBRARY_PATH` 查找。32 位 glibc 目标指定的动态加载器为 `/lib/ld-linux.so.2`（i686）和 `/lib/ld-linux-armhf.so.3`（armv7hf）。riscv32 映像（无操作系统，进行中）是静态的，不链接 C 库：运行时自带入口点和堆（[0.6.2](../000-meta/006-platform-targets.md#riscv32imac-unknown-none-elf)）。如果设置了 `NABI_CC`，则改由该 C 编译器驱动程序执行最终链接（`$NABI_CC <objects> <archives> -o <out>`）；这是一个可选的覆盖手段。用 `--native-runtime` 给出的共享库会被复制到可执行文件旁边，可执行文件获得 rpath `@loader_path`（Mach-O）/ `$ORIGIN`（ELF）；Windows 从可执行文件所在目录加载 DLL。可执行文件会经过死代码剥离：删除未被引用的函数和数据；`release` 构建还会折叠相同的函数，并合并相同的字符串字面量（[9.2.15](#9215-output)）。Mach-O 可执行文件带有临时（ad-hoc）代码签名，arm64 macOS 要求有它才能运行。在 Apple 目标上，当可执行文件带有调试信息时，`nabic` 把目标文件中的 DWARF 汇集到可执行文件旁边的 `.dSYM` 包中；在 Windows 目标上，链接器写出 PDB（MSF 7.0）：合并后的目标文件 CodeView 类型、每个目标文件一个模块（含其符号和行表）、全局符号和公开符号，以及节贡献；可执行文件的调试目录指向它（一条带有 PDB 的 GUID 和 age 的 RSDS 记录）。`wasm32-wasip2` 可执行文件先链接为核心 wasm 模块，再用工具链为该目标随附的 preview 1 适配器制成 WASI preview 2 组件；`ffi::wit` world（[std 1.8](../../refs-standard-libraries/100-bare/108-wit.md)）决定它的导入和导出。

## 9.2.15. 输出

输出与 cargo 一样放在**按构建配置划分**的目录中。构建配置有 `debug`（默认）和 `release`；优化级别为 `-O2` / `-O3` / `-Os`（包括 `--release`）时，构建配置为 `release`。

各构建配置的区别与 cargo 相同：

| 构建配置 | 优化 | 调试信息 | 链接 |
|---|---|---|---|
| `debug` | `-O0`、`-O1` | 开启 | 删除未被引用的代码和数据 |
| `release` | `-O2`、`-O3`、`-Os` | 仅在使用 `-g` 时 | 还折叠相同的函数并合并相同的字符串字面量；不使用 `-g` 时，只保留加载器需要的符号 |

相同代码折叠按机器码和重定位比较函数：当两个函数的字节相同，并且每个引用都指向同一位置，或指向本身也会折叠在一起的函数时，二者折叠为一个。被取地址的函数（被存储、比较、导出、放入表中）永远不会被折叠，因此不同的函数保持不同的地址；只有被调用和跳转到的函数才会折叠，函数值所调用的 thunk（`<symbol>.v`，[8.11](../008-abi/811-backend-contract.md)）也会折叠：函数值不能比较，所以两个函数值共享代码不会改变程序能看到的任何东西。被折叠副本的展开条目和调试信息会像被删除的代码一样丢弃：调试器、性能分析器或栈回溯给出的是保留下来的那个副本的名称。字节相同的字符串字面量共享一个副本（它们的地址没有意义）。`-C icf=no` 在 `release` 构建中关闭此功能，`-C icf` 在 `debug` 构建中开启此功能（[9.1](901-nabic-cli.md)）。

调试信息属于构建所生成的可执行文件或归档文件（`build`、`run`、`test`，以及显式指定 `-O0` / `-O1` 时的 `bench`），不属于 `--emit` 输出，也不属于 REPL 和 Jupyter 单元。在两种构建配置中，panic 都会给出其源代码位置。一次构建的目标文件从输出旁边的同一位置链接，因此相同的输入生成相同的可执行文件。两种构建配置都开启溢出检查；`nabi.nabion` 中的 `profile` 可以为某个构建配置关闭它（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。构建配置没有其他设置。

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

| 输出          | 路径                                                              |
|-----------------|-------------------------------------------------------------------|
| 可执行文件      | `target/<profile>/<name>`                                         |
| 库         | `target/<profile>/lib<name>.a`                                    |
| `--emit=<kind>` | `target/<profile>/build/<name>.<ext>`（`ast.nabi` `hir` `mir` `o` `s` `lst` `expanded.nabi`） |
| 中间产物   | `target/<profile>/build/`                                         |
| 依赖    | `target/<profile>/deps/`（`lib<dep>.a`、`<dep>.nabihir`、`<dep>.fp`，9.2.3） |
| 交叉构建     | `target/<triple>/<profile>/...`（结构同上）         |
| 文档   | `target/doc/`                                                     |
| 增量     | `target/<profile>/incremental/<name>/`（可执行文件的代码生成单元）、`.../lib<name>/`（库的），以及每个目标根一个的 `.../<name>-<hash>/`（前端），9.2.16 |

同时具有两个根文件的包有两种输出：`target/<profile>/<name>` 和 `target/<profile>/lib<name>.a`（+ `<name>.nabihir`）；`-o` 指定可执行文件的名称。

名称 `foo` 是包名（9.2.2）。`-o` 改变单个输出的路径；`--target-dir=<dir>` 和环境变量 `NABI_TARGET_DIR` 改变 `target/` 本身的位置。`nabi clean` 删除整个 `target/`。库的 NABIHIR 是归档文件旁边的 `foo.nabihir`（[9.4](904-nabihir-bytecode.md)）。

## 9.2.16. 增量构建

增量构建在三个层次上起作用：输入没有变化的包根本不构建；需要构建的包从上次构建中取用其未变化的文件和函数的解析结果与降级后的函数体（前端，见下文）；它的代码按代码生成单元编译，只有发生变化的单元才重新编译。如果上次构建的指纹等于当前指纹，`nabic build` / `nabi run` 打印 `Fresh` 并跳过构建。指纹包括：

- 编译器的身份（与 NABIHIR 生成者相同的值）、构建选项、诊断格式（`--message-format`、颜色）、输入 / 输出路径、`NABI_CC`（如果设置了）
- 包及其依赖的 `src/` 下的文件、它们的 `nabi.nabion`，以及构建读取的所有其他文件（`src/` 之外的 `#[path]` / `include_str!`）
- 由 `include_str!` / `include_bytes!` 读取的文件（[6.11](../006-meta-system/611-include.md)）及其内容
- lint 级别（`-A` / `-W` / `-D`，以及通过它们生效的清单中的 `lints`）
- 运行时库、工具链为该目标随附的 C 运行时目标文件和 C 库（[9.2.19](#9219-installation-layout)），以及预构建依赖的 `.a`，每一项都按其大小、修改时间和文件身份（inode 和状态变更时间；在 Windows 上为创建时间）比较，因此保留了修改时间的副本也算作变化
- 输出的大小、修改时间和文件身份

构建发出的警告（包括 lint 警告和后端的警告）会被记录下来，`Fresh` 构建会再次打印它们。

每次构建，无论是本机构建还是交叉构建，都会编译程序可达的标准库函数（[9.3.6](903-package-archive.md#936-standard-library)）；它们的优化后代码和所在单元的目标文件保存在工具链的函数缓存 `lib/nabi-cache/fn/`（9.2.19）中，供该工具链的每次构建使用，单文件的构建也不例外：

- 条目的键涵盖其结果所依赖的一切：对于优化后代码，是该步骤之前的函数代码、级别、目标，以及该步骤从它所调用的函数中读取的内容，全部按名称记录；对于目标文件，是该单元的键（见下文）；两者都包含编译器的身份。包会把标准库降级后的函数体与自己的函数体一起保存（前端，见下文）。构建取用键匹配的条目，并写入自己生成的条目；输出与使用空缓存的构建所生成的输出相同。
- 条目是先完整写出再重命名到位的文件，因此同时运行的多个构建可以安全地共享缓存。
- 构建每天最多一次删除 30 天内没有任何构建读取过的条目；`nabi cache prune` 删除更多条目（9.10.6）。缓存目录为只读且非空的工具链（构建时已预热的沙箱映像）读取其条目而不写入任何条目。
- 缓存有上限 `NABI_CACHE_MAX_SIZE`（字节数，或带 `K`、`M` 或 `G` 的数字；默认 `5G`；`off`：无上限），涵盖函数缓存及其库阶段（见下文）。写入了条目的构建把条目的大小累加到缓存目录中的一个累计总量上；当总量超过上限时，同一时间只有一个构建测量缓存，如果超过上限，就删除最久未使用的条目，直到剩余不超过上限的 80%。条目的使用时间是它最后一次被写入或读取的时间，每小时最多记录一次。最近一小时内使用过的条目和仍在写入的文件永远不会以这种方式删除，因此同时运行的构建可以保留它们正在使用的内容；当这些条目使缓存保持在上限的 80% 以上时，下一次测量要等到又写入了上限的五分之一之后才进行。被删除的条目由下一个需要它的构建重新计算。
- `NABI_FN_CACHE=0` 既不读取也不写入缓存。

### 前端

包的构建（`check`、`build`、`run`、`test`、`bench`、`doc`、`lint`）把其前端计算的结果保存在 `target/<profile>/incremental/<name>-<hash>/` 下，其中 `<hash>` 标识包根（同时具有两个根文件的包的每个目标都有自己的目录，`<hash>` 标识其根），每种构建（目标、cfg、是否为测试、哪些库是预构建的）各有一个子目录。下一次同种构建从中取用：

- 文本和解析输入（构建的扩展运算符、包的关键字、元块和 `#[literal]` 模式，以及 cfg）都没有变化的每个文件的解析结果。
- 每个普通函数体的降级结果（名称解析、类型检查、泛型实例化）：普通函数体指非泛型、非 `const` 函数，或非泛型 `impl` 的方法。当它自身的文本和位置没有变化、程序的接口没有变化、降级时读取的每个签名、实例和其他事实现在都相同，并且其结果在函数体之外指向的每个位置之前的文本都没有变化时，就重用它。
- 接口是除以下内容之外的所有项：写明了返回类型（或为 `()`）的普通函数体，以及普通函数的参数和返回类型；返回类型靠推断的函数体属于接口（其调用者会读取该类型）。编辑函数体会重新降级该函数体，编辑返回类型靠推断的函数体还会重新降级读取其类型的函数体；编辑普通函数的签名会重新降级它以及读取其签名的函数体；对项的任何其他更改（类型、trait、`impl` 头、泛型函数、`use`）都会重新降级所有函数体。
- 重用的降级结果所生成的函数、类型和实例按干净构建的方式编号；诊断、程序及其符号都与干净构建相同。

签名、`impl` 注册、一致性和全程序检查在每次构建中都会运行。收集库的项（标准库的和依赖的）所得的结果，以及库文件的预处理解析结果，保存在工具链的缓存中（`lib/nabi-cache/library/`，按工具链、构建标志和库源代码的哈希每个一个文件），供每次构建使用，单文件的构建也不例外；`NABI_INCREMENTAL` 既不会移动它们，也不会禁用它们。保存的任何内容都不是必需的：缺失、损坏或过时的条目会重新计算。不属于任何包的单个文件不保存自己的解析结果或函数体，除非 `NABI_INCREMENTAL` 指定了一个目录。`NABI_INCREMENTAL=<dir>` 把结果保存在 `<dir>/<name>-<hash>/` 下；`NABI_INCREMENTAL=0` 将其关闭。`NABI_INCREMENTAL_VERIFY=1` 还会重新降级每个函数体，如果重用的降级结果与之不同，就使构建失败。`-v` 报告有多少文件的解析结果和函数体被重用。

### 代码生成单元

不是 `Fresh` 的构建按代码生成单元重用已编译的代码。nabic 把一个包的代码（可执行文件的或库的）划分为单元：包的每个模块是一个单元，标准库和依赖中被它编译了函数或实例的每个模块也各是一个单元（实例可以按其类型实参分散到多个单元中）；另有一个单元存放程序中只有一份的内容（静态变量、C `main`、GC 类型注册）。

- 只有当单元的键自上次构建到同一目标目录以来发生了变化时，才编译该单元。否则就重用保存在 `target/<profile>/incremental/<name>/`（库的为 `lib<name>/`）下的结果；然后链接所有内容。标准库单元则保存在函数缓存中（见上文），供每个包的构建使用；单个文件也会使用它，尽管它不保存自己的单元。预构建依赖（9.2.3）编译为一个目标文件，不划分单元。
- 匿名文件的单元保存在不使用包时所用的目标目录中（`--target-dir`、`NABI_TARGET_DIR`，否则为单文件缓存），位于 `[<triple>/]<profile>/incremental/<stem>-<hash of its path>/` 下。单文件缓存在 Unix 上是 `<system temporary directory>/nabic-target-<uid>/`，`<uid>` 为有效用户 ID；在 Windows 上是 `%TEMP%\nabic-target\`，Windows 的临时目录是用户自己的。它是缓存：nabic 每天最多一次删除一周内没有任何构建写入过的文件。
- 单个文件的可执行文件保存在同一目标目录中，位于 `[<triple>/]<profile>/scripts/<key>/` 下，在其键匹配时被重用；一天未使用的条目会在构建另一个条目时被删除（[9.10.7](910-nabi-frontend.md#9107-inline-manifest-for-single-file-scripts)）。`nabi run` 运行它，`nabic build` 把它复制到文件旁边。对于脚本，`nabi` 用 `$NABI_HOME/cache/scripts/<hash>/` 下脚本自己的 `target/` 传入 `--target-dir`；其余内容由单文件缓存保存：由 `nabic` 自身构建的单个文件，以及位于其 `nabi.nabion` 使之成为单文件包的目录中的文件（9.10.1）。
- 单文件缓存保存会被运行的可执行文件，因此在 Unix 上它是私有的：编译器以模式 `0700` 创建它，使用前检查它是目录（不是符号链接）、属于有效用户、并且对组和其他用户没有任何权限位。否则命令以退出码 `1` 失败，并输出 ``nabic: error: <dir> is not private to you (owner uid N, mode 0755); remove it or set NABI_TARGET_DIR``。`--target-dir` 和 `NABI_TARGET_DIR` 按给出的值使用。
- 单元的键涵盖其函数的代码、它从其他单元使用的内容的签名和代码生成事实（函数的签名变化会重新编译其调用者所在的单元）、它使用的静态变量和 GC 类型、目标、优化级别、代码生成选项、编译器的身份，以及在 `-g` 下其调试信息所指的源文件路径（添加文件只会重新编译引用它的单元）。
- 符号和 GC 类型 ID 不依赖于其他单元的内容（单元从包的共享单元所定义的符号中读取 GC 类型 ID；[9.5.7](905-symbol-mangling.md#957-collision-avoidance-typehash-stability)），因此输出与干净构建所生成的输出相同。
- 优化构建仍然跨单元内联：包在划分之前作为整体进行优化（9.2.10）。
- `-v` 报告有多少单元被重用、多少被编译；`-vv` 列出被编译的单元的名称。

## 9.2.17. 编译器与运行时的分离

`nabic` 不使用任何第三方 crate，也不链接任何外部代码生成器。

流水线把编译器和运行时分开：`nabic` 在构建时不引用运行时（`libnabir.a`）。

它只生成对 `extern "C"` 符号的调用（[Extern "C" 符号](../008-abi/805-extern-symbols.md)），在链接时由用户构建一并链接 `libnabir.a`。

这种分离使 `nabic` 和 `libnabir.a` 可以按各自独立的 SemVer 演进。

## 9.2.18. 自包含的工具链

`nabic` 是自包含的：它的代码生成器和链接器都是它自己的（[0.2.4](../000-meta/002-pipeline.md#024-code-generation)），它不运行任何外部编译器、汇编器或链接器。交叉编译是内置的：工具链为其支持的 Linux 目标随附 C 库（musl、glibc 链接桩）。

| 需要系统提供的内容 | 目标                                   |
|------------------------|------------------------------------------|
| macOS SDK（Xcode Command Line Tools；不可再分发） | Apple 目标 |
| 无（随附 C 库） | Linux 目标                   |
| 无（由随附的模块定义构建的导入库；UCRT） | Windows 目标 |
| `NABI_CC`（可选）    | 覆盖最终链接（9.2.14）        |

C 编译器 `nabicc` 还需要目标的 C 头文件：工具链为其 Linux 和 wasm32 目标随附这些头文件，Apple 目标从 macOS SDK 获取，Windows 目标则在 Windows 主机上从 Windows SDK 和 Visual Studio 获取（[9.13.7](913-c-compiler.md#9137-headers-and-c-library)）。

## 9.2.19. 安装布局

安装布局沿用 rustup / cargo 的做法。`NABI_HOME` 的默认值是 `~/.nabi`（在 Windows 上为 `%USERPROFILE%\.nabi`）。

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

`nabic` 相对于自己的可执行文件查找标准库和运行时（sysroot = 可执行文件的 `..`），不查看 `NABI_HOME`。标准库的每个层级都按库包的方式布局：其根模块是 `lib/nabi/<tier>/src/lib.nabi`，其模块位于 `lib/nabi/<tier>/src/` 下。在实现仓库中构建的 `nabic` 从仓库的 `library/<tier>/src/` 读取相同的布局。它自己的缓存是工具链的 `lib/nabi-cache/`（仓库构建使用 `target/<profile>/nabi-cache/`）；以只读方式安装的工具链（系统软件包）则改用用户缓存目录中的 `nabi/nabic-<hash of the sysroot>/`。缓存在 `fn/` 中保存函数缓存，即各次构建编译的标准库函数（9.2.16），在 `library/` 中保存库收集到的项和预处理解析结果（9.2.16）。`nabi cache` 可以显示、修剪和清空它（9.10.6）。活动工具链的选择由 [9.10.6](910-nabi-frontend.md#9106-toolchain) 定义。

Linux（glibc）工具链需要 glibc 2.34 或更高版本，与它构建的可执行文件一样（9.2.14）。
