# 9. 執行 / 工具概觀

本部分規定 Nabi 編譯器 `nabic`、前端 `nabi`，以及執行環境（REPL / Jupyter 核心）。它涵蓋[執行引擎與使用者介面](../000-meta/001-execution-modes.md)中定義的兩個引擎（[JIT](../000-meta/007-glossary.md#jit即時編譯) / [AOT](../000-meta/007-glossary.md#aot預先編譯)）與三種使用者介面（REPL / Jupyter / 直接執行 AOT 產出）如何實作。

## 目錄

- [9.1. `nabic` CLI 介面](901-nabic-cli.md)
- [9.2. AOT 建置管線](902-aot-pipeline.md)
- [9.3. 套件封存檔（.a）格式](903-package-archive.md)
- [9.4. NABIHIR 格式](904-nabihir-bytecode.md)
- [9.5. 符號修飾方案](905-symbol-mangling.md)
- [9.6. JIT 引擎](906-jit-engine.md)
- [9.7. JIT 儲存格的執行流程與狀態管理](907-jit-cells.md)
- [9.8. Jupyter 核心整合](908-jupyter-integration.md)
- [9.9. 偵錯 / 診斷工具](909-debug-tools.md)
- [9.10. `nabi`：套件 / 建置前端與工具](910-nabi-frontend.md)
- [9.11. nabi-mir 檔案格式](911-mir-format.md)
- [9.12. 外掛](912-plugins.md)
- [9.13. `nabicc`：C 編譯器](913-c-compiler.md)
- [9.14. 熱重載](914-hot-reload.md)

## 工具介面

工具介面的組成如下。

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

工具鏈由編譯器 `nabic`（相當於 rustc）、cargo 風格的前端 `nabi`、C 編譯器 `nabicc`（[9.13](913-c-compiler.md)）、編輯器 LSP 執行檔 `nabils`，以及執行環境 `libnabir.a` 組成。`nabi` 負責清單 `nabi.nabion` / `nabi.lock`、相依套件、環境與工具鏈釘選；建置指令委派給 `nabic`，並以 `--extern` 附上已解析的相依套件，而各工具在連結了編譯器的 `nabi` 本身之中執行（[9.10](910-nabi-frontend.md)）。沒有另外的工具（`nabi-fmt` 等）。

## 兩個引擎的實作

[執行引擎與使用者介面](../000-meta/001-execution-modes.md)中定義的兩個引擎分工如下。

```
JIT engine  ──  used by nabi repl / jupyter
AOT engine  ──  used by nabic build / test, nabi run
```

兩個引擎都經過相同的編譯管線（nabi-hir → nabi-mir → 機器碼），只在程式碼的去向上分歧：載入工作階段的主機，或連結進執行檔。詳見[執行引擎與使用者介面](../000-meta/001-execution-modes.md)與[編譯管線](../000-meta/002-pipeline.md)。

## 三種使用者介面的實作

三種使用者介面以及實作它們的指令如下：

```
surface              command
────────             ──────
REPL                 nabi repl
Jupyter              nabi jupyter         (invoked by Jupyter, not directly)
AOT direct exec      nabic build → run artifact
```

各介面的實作在下列文件中說明。

## 套件建置產出

[原始碼單元](../000-meta/003-source-units.md)的函式庫套件編譯為 `libfoo.a` 以及介面 `foo.nabihir`。

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

各設定檔的配置定義於 [9.2.15](902-aot-pipeline.md#9215-輸出)，[封存檔](../000-meta/007-glossary.md#封存檔archive)格式定義於[套件封存檔](903-package-archive.md)。

## NABIHIR：泛型函式的可攜 IR

泛型函式必須在呼叫處[單態化](../000-meta/007-glossary.md#單態化monomorphization)，因此只有二進位程式碼是不夠的。所以函式庫旁邊會放一個介面檔 NABIHIR。NABIHIR 容納去除了具體函式主體的套件原始碼（因為單態化以 AST 作為輸入）。

詳細格式定義於 [NABIHIR 格式](904-nabihir-bytecode.md)。

## 唯一的相依：`libnabir.a`

執行檔連結使用者套件的 `libfoo.a`、執行環境 `libnabir.a`，以及它所觸及的標準函式庫函式，這些函式隨它一起從原始碼編譯，或取自工具鏈的函式快取（[9.3.6](903-package-archive.md#936-標準函式庫)）。除了作業系統的外部函式庫之外，沒有其他相依。

詳細的建置流程定義於 [AOT 建置管線](902-aot-pipeline.md)。
