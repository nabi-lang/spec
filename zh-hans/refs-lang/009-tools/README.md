# 9. 执行与工具概览

本部分规定 Nabi 编译器 `nabic`、前端 `nabi` 以及执行环境（REPL / Jupyter 内核）。它说明[执行引擎与用户界面](../000-meta/001-execution-modes.md)中定义的两种引擎（[JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)）和三种用户界面（REPL / Jupyter / 直接运行 AOT 输出）如何实现。

## 目录

- [9.1. `nabic` CLI 接口](901-nabic-cli.md)
- [9.2. AOT 构建流水线](902-aot-pipeline.md)
- [9.3. 包归档文件（.a）格式](903-package-archive.md)
- [9.4. NABIHIR 格式](904-nabihir-bytecode.md)
- [9.5. 符号修饰方案](905-symbol-mangling.md)
- [9.6. JIT 引擎](906-jit-engine.md)
- [9.7. JIT 单元的执行流程与状态管理](907-jit-cells.md)
- [9.8. Jupyter 内核集成](908-jupyter-integration.md)
- [9.9. 调试与诊断工具](909-debug-tools.md)
- [9.10. `nabi`：包与构建前端及工具](910-nabi-frontend.md)
- [9.11. nabi-mir 文件格式](911-mir-format.md)
- [9.12. 插件](912-plugins.md)
- [9.13. `nabicc`：C 编译器](913-c-compiler.md)
- [9.14. 热重载](914-hot-reload.md)

## 工具组成

工具的组成如下。

```
nabi  (front-end, cargo-like; the compiler linked in)
  │   new / init, add / remove / update / deps (nabi.nabion, nabi.lock), env, toolchain
  │
  ├── tools (run in nabi itself)
  │   ├── nabi run         : build, then run
  │   ├── nabi dev         : run, applying edits while it runs (hot reload)
  │   ├── nabi lint        : linter
  │   ├── nabi repl        : terminal REPL (each cell runs once)
  │   ├── nabi jupyter     : Jupyter kernel (REPL cell session + ZMQ)
  │   ├── nabi lsp         : LSP server (IDE integration)
  │   ├── nabi fmt         : formatter
  │   ├── nabi doc         : API documentation
  │   ├── nabi clean       : remove build artifacts
  │   ├── nabi bindgen     : C header bindings
  │   ├── nabi inspect     : package inspect
  │   ├── nabi demangle    : symbol demangle
  │   └── nabi completions : shell completion
  │
  │   build / check / emit / test / bench
  │        └── resolve deps, then call nabic ... --extern <dep roots>
  │   cc
  │        └── run nabicc with the arguments unchanged
  ▼
nabic  (compiler)
  │
  ├── nabic build      : AOT compile .nabi source to .a / executable
  ├── nabic emit       : intermediate output
  ├── nabic test       : collect + build the #[test] harness (nabi test runs it)
  ├── nabic bench      : collect + build the #[bench] harness (nabi bench runs it)
  ├── nabic check      : front end, lints and code generation (no native build)
  └── nabic version / help / --print / --explain

nabicc  (= nabi cc, the C compiler)
nabils  (= nabi lsp, for editors)
libnabir.a  (runtime, nabir)
```

工具链由以下部分组成：编译器 `nabic`（相当于 rustc）、cargo 风格的前端 `nabi`、C 编译器 `nabicc`（[9.13](913-c-compiler.md)）、供编辑器使用的 LSP 可执行文件 `nabils`，以及运行时 `libnabir.a`。`nabi` 负责清单 `nabi.nabion` / `nabi.lock`、依赖、环境和工具链固定；构建命令交给 `nabic` 执行，并通过 `--extern` 附上解析好的依赖包；各个工具在链接了编译器的 `nabi` 自身中运行（[9.10](910-nabi-frontend.md)）。不存在单独的工具（如 `nabi-fmt`）。

## 两种引擎的实现

[执行引擎与用户界面](../000-meta/001-execution-modes.md)中定义的两种引擎按如下方式划分。

```
JIT engine  ──  used by nabi repl / jupyter
AOT engine  ──  used by nabic build / test, nabi run
```

两种引擎都经过同一条编译流水线（nabi-hir → nabi-mir → 机器码），只在代码的去向上分开：加载到会话的宿主中，或链接进可执行文件。详见[执行引擎与用户界面](../000-meta/001-execution-modes.md)和[编译流水线](../000-meta/002-pipeline.md)。

## 三种用户界面的实现

三种用户界面及实现它们的命令如下：

```
surface              command
────────             ──────
REPL                 nabi repl
Jupyter              nabi jupyter         (invoked by Jupyter, not directly)
AOT direct exec      nabic build → run artifact
```

各界面的实现在后续文档中说明。

## 包的构建输出

[源单元](../000-meta/003-source-units.md)中的库包编译为 `libfoo.a` 和接口文件 `foo.nabihir`。

```
foo/
├── src/
│   ├── lib.nabi              <- input
│   └── ...
       v nabic build
foo/target/debug/
├── libfoo.a                 <- artifact (static archive)
├── foo.nabihir                 <- interface (NABIHIR)
├── build/                   <- intermediates (foo.o, ...)
└── ...
```

各构建配置的目录布局在 [9.2.15](902-aot-pipeline.md#9215-输出) 中定义，[归档文件](../000-meta/007-glossary.md#archive)格式在[包归档文件](903-package-archive.md)中定义。

## NABIHIR：泛型函数的可移植 IR

泛型函数必须在调用处[单态化](../000-meta/007-glossary.md#monomorphization单态化)，因此仅有二进制代码并不够。为此，库旁边放着一个接口文件 NABIHIR。NABIHIR 保存去掉了具体函数体的包源代码（因为单态化以 AST 为输入）。

详细格式在 [NABIHIR 格式](904-nabihir-bytecode.md)中定义。

## 唯一的依赖：`libnabir.a`

可执行文件链接用户包的 `libfoo.a`、运行时 `libnabir.a`，以及它用到的标准库函数；这些函数要么随它从源代码编译，要么取自工具链的函数缓存（[9.3.6](903-package-archive.md#936-标准库)）。除外部的操作系统库之外，没有其他依赖。

详细的构建流程在 [AOT 构建流水线](902-aot-pipeline.md)中定义。
