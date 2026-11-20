# 9. 실행 / 도구 개요

이 블록은 Nabi의 컴파일러 `nabic`, 프런트엔드 `nabi`, 실행 환경(REPL / Jupyter 커널)을 정의합니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)은 두 엔진([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time))과 세 사용자 표면(REPL / Jupyter / AOT 산출물 직접 실행)을 정합니다. 이 블록은 그것을 어떻게 구현하는지 다룹니다.

## 목차

- [9.1. `nabic` CLI 인터페이스](901-nabic-cli.md)
- [9.2. AOT 빌드 파이프라인](902-aot-pipeline.md)
- [9.3. 패키지 아카이브(.a) 포맷](903-package-archive.md)
- [9.4. NABIHIR 포맷](904-nabihir-bytecode.md)
- [9.5. 심볼 맹글링 스킴](905-symbol-mangling.md)
- [9.6. JIT 엔진](906-jit-engine.md)
- [9.7. JIT 셀 실행 흐름과 상태 관리](907-jit-cells.md)
- [9.8. Jupyter 커널 통합](908-jupyter-integration.md)
- [9.9. 디버그 / 진단 도구](909-debug-tools.md)
- [9.10. `nabi`: 패키지 / 빌드 프런트엔드와 도구](910-nabi-frontend.md)
- [9.11. nabi-mir 파일 형식](911-mir-format.md)
- [9.12. 플러그인](912-plugins.md)
- [9.13. `nabicc`: C 컴파일러](913-c-compiler.md)
- [9.14. 핫 리로드](914-hot-reload.md)

## 도구 표면

도구 표면의 구성은 다음과 같습니다.

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

toolchain은 컴파일러 `nabic`(rustc에 해당), cargo 식 프런트엔드 `nabi`, C 컴파일러 `nabicc`([9.13](913-c-compiler.md)), 에디터용 LSP 실행 파일 `nabils`, 런타임 `libnabir.a`로 이루어집니다. `nabi`는 매니페스트 `nabi.nabion` / `nabi.lock`, 의존성, 환경, toolchain 고정을 맡습니다. 빌드 명령은 해소한 의존 패키지를 `--extern`으로 붙여 `nabic`에 넘기고, 도구는 컴파일러를 링크해 담은 `nabi` 안에서 실행합니다([9.10](910-nabi-frontend.md)). 별도 도구(`nabi-fmt` 등)는 두지 않습니다.

## 두 엔진의 구현

[실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)에서 정한 두 엔진은 다음과 같이 나뉩니다.

```
JIT engine  ──  used by nabi repl / jupyter
AOT engine  ──  used by nabic build / test, nabi run
```

두 엔진은 같은 컴파일 파이프라인(nabi-hir → nabi-mir → 기계어)을 거치고, 만든 코드를 어떻게 쓰는지에서만 갈라집니다. 세션의 호스트에 로드하거나, 실행 파일로 링크합니다. 자세한 내용은 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)과 [컴파일 파이프라인](../000-meta/002-pipeline.md)에 있습니다.

## 세 사용자 표면의 구현

세 사용자 표면과 각 표면을 구현하는 명령은 다음과 같습니다.

```
surface              command
────────             ──────
REPL                 nabi repl
Jupyter              nabi jupyter         (invoked by Jupyter, not directly)
AOT direct exec      nabic build → run artifact
```

각 표면의 구현은 이어지는 절에서 다룹니다.

## 패키지 빌드 산출물

[소스 단위](../000-meta/003-source-units.md)의 라이브러리 패키지를 컴파일하면 `libfoo.a`와 인터페이스 `foo.nabihir`이 나옵니다.

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

프로필별 레이아웃은 [9.2.15](902-aot-pipeline.md#9215-산출물)가 정의합니다. [아카이브](../000-meta/007-glossary.md#archive) 형식은 [패키지 아카이브](903-package-archive.md)가 정의합니다.

## NABIHIR: 제네릭 함수의 휴대용 IR

제네릭 함수는 호출 지점에서 [단형화](../000-meta/007-glossary.md#monomorphization-단형화)해야 하므로 바이너리 코드만으로는 부족합니다. 그래서 라이브러리 옆에 인터페이스 파일 NABIHIR을 둡니다. NABIHIR에는 구체 함수의 본문을 잘라 낸 패키지 소스가 들어갑니다. 단형화가 AST를 입력으로 쓰기 때문입니다.

자세한 형식은 [NABIHIR 포맷](904-nabihir-bytecode.md)에서 정의합니다.

## 단일 의존성: `libnabir.a`

실행 파일은 사용자 패키지의 `libfoo.a`, 런타임 `libnabir.a`, 그리고 함께 소스에서 컴파일하거나 toolchain의 함수 캐시에서 가져온, 프로그램이 닿는 표준 라이브러리 함수를 링크합니다([9.3.6](903-package-archive.md#936-표준-라이브러리)). 외부 OS 라이브러리 외에는 추가 의존성이 없습니다.

자세한 빌드 흐름은 [AOT 빌드 파이프라인](902-aot-pipeline.md)에서 정의합니다.
