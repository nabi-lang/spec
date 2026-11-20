# 9. Execution / tools overview

This block specifies the Nabi compiler `nabic`, the front-end `nabi`, and the execution environments (REPL / Jupyter kernel). It covers how the two engines ([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)) and the three user surfaces (REPL / Jupyter / direct execution of AOT output) defined in [Execution engines and user surfaces](../000-meta/001-execution-modes.md) are implemented.

## Contents

- [9.1. `nabic` CLI interface](901-nabic-cli.md)
- [9.2. AOT build pipeline](902-aot-pipeline.md)
- [9.3. Package archive (.a) format](903-package-archive.md)
- [9.4. NABIHIR format](904-nabihir-bytecode.md)
- [9.5. Symbol mangling scheme](905-symbol-mangling.md)
- [9.6. JIT engine](906-jit-engine.md)
- [9.7. JIT cell execution flow and state management](907-jit-cells.md)
- [9.8. Jupyter kernel integration](908-jupyter-integration.md)
- [9.9. Debug / diagnostic tools](909-debug-tools.md)
- [9.10. `nabi` - package / build front-end and tools](910-nabi-frontend.md)
- [9.11. nabi-mir file format](911-mir-format.md)
- [9.12. Plugins](912-plugins.md)
- [9.13. `nabicc` - the C compiler](913-c-compiler.md)
- [9.14. Hot reload](914-hot-reload.md)

## Tool surface

The tool surface is organized as follows.

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

The toolchain consists of the compiler `nabic` (the counterpart of rustc), the cargo-style front-end `nabi`, the C compiler `nabicc` ([9.13](913-c-compiler.md)), the editor LSP executable `nabils`, and the runtime `libnabir.a`. `nabi` owns the manifest `nabi.nabion` / `nabi.lock`, dependencies, environments and the toolchain pin; build commands are delegated to `nabic` with the resolved dependency packages attached via `--extern`, and the tools run in `nabi` itself with the compiler linked in ([9.10](910-nabi-frontend.md)). There are no separate tools (`nabi-fmt` etc.).

## Implementation of the two engines

The two engines defined in [Execution engines and user surfaces](../000-meta/001-execution-modes.md) are divided as follows.

```
JIT engine  ──  used by nabi repl / jupyter
AOT engine  ──  used by nabic build / test, nabi run
```

Both engines go through the same compile pipeline (nabi-hir → nabi-mir → machine code) and diverge only in what happens to the code: loaded into a session's host, or linked into an executable. See [Execution engines and user surfaces](../000-meta/001-execution-modes.md) and [Compile pipeline](../000-meta/002-pipeline.md) for details.

## Implementation of the three user surfaces

The three user surfaces and the commands that implement them are:

```
surface              command
────────             ──────
REPL                 nabi repl
Jupyter              nabi jupyter         (invoked by Jupyter, not directly)
AOT direct exec      nabic build → run artifact
```

The implementation of each surface is covered in the following documents.

## Package build output

A library package of a [source unit](../000-meta/003-source-units.md) compiles to `libfoo.a` and the interface `foo.nabihir`.

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

The per-profile layout is defined in [9.2.15](902-aot-pipeline.md#9215-output), and the [archive](../000-meta/007-glossary.md#archive) format in [Package archive](903-package-archive.md).

## NABIHIR: portable IR for generic functions

Generic functions must be [monomorphized](../000-meta/007-glossary.md#monomorphization) at the call site, so binary code alone is not enough. An interface file, NABIHIR, therefore sits next to the library. NABIHIR holds the package source with concrete function bodies stripped (because monomorphization takes the AST as input).

The detailed format is defined in [NABIHIR format](904-nabihir-bytecode.md).

## Single dependency: `libnabir.a`

An executable links the user package's `libfoo.a`, the runtime `libnabir.a`, and the standard library functions it reaches, compiled from source with it or taken from the toolchain's function cache ([9.3.6](903-package-archive.md#936-standard-library)). There are no further dependencies beyond external OS libraries.

The detailed build flow is defined in [AOT build pipeline](902-aot-pipeline.md).
