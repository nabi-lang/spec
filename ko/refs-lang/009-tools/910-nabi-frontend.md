# 9.10. `nabi`: 패키지 / 빌드 프런트엔드와 도구

`nabi`는 cargo 같은 프런트엔드입니다. 매니페스트 `nabi.nabion`, lockfile `nabi.lock`, 의존성, 환경, toolchain 고정, 스크립트 의존성을 맡습니다. 컴파일은 컴파일러 `nabic`([9.1](901-nabic-cli.md))를 불러 처리합니다. 도구(실행, 린트, REPL, Jupyter, LSP, 포매터, 문서 등)는 `nabi` 자신에 들어 있습니다(9.10.8). 런타임은 `nabir`(`libnabir.a`)입니다.

의존성이 없는 패키지는 매니페스트 없이 파일 규약만으로 빌드됩니다([0.3.4](../000-meta/003-source-units.md#034-매니페스트nabinabion는-선택-사항)). 이때 `nabi build`는 `nabic build`와 같습니다.

## 9.10.1. 서브명령과 전역 옵션

| 서브명령                                                          | 동작                                    |
|-------------------------------------------------------------------|-----------------------------------------|
| `new`, `init`                                                     | 패키지 생성(9.10.2)                     |
| `add`, `remove`, `update`, `deps`                                 | 의존성(9.10.4)                          |
| `search`, `publish`, `yank`, `login`, `logout`                    | 레지스트리 패키지(9.10.4a)               |
| `env`                                                             | 환경(9.10.5)                            |
| `component`                                                       | 선택 구성 요소: 네트워크 없는 빌드를 위한 공식 패키지(9.10.4a) |
| `toolchain`, `cache`                                              | toolchain과 그 빌드 캐시(9.10.6)        |
| `install`, `uninstall`                                            | REPL / Jupyter용 라이브러리와 실행 파일(9.10.5a) |
| `build`, `check`, `emit`                                          | 의존성을 붙여 `nabic`에 위임            |
| `test`, `bench`                                                   | `nabic`가 의존성을 붙여 하네스를 빌드하고, `nabi`가 실행([9.1](901-nabic-cli.md)) |
| `run`, `dev`, `lint`, `fix`, `repl [path]`, `jupyter`, `lsp`, `doc` | 도구. 의존성을 붙임(9.10.8)             |
| `fmt`, `clean`, `bindgen`, `inspect`, `demangle`, `completions <shell>` | 도구. 의존성 없음(9.10.8)        |
| `cc`                                                              | C 컴파일러 `nabicc`. 인자는 그대로([9.13](913-c-compiler.md)) |
| `version`, `help`                                                 | 버전, 도움말(아래)                      |
| (없음)                                                            | `nabi repl`과 같음. REPL 시작           |

위임하는 서브명령은 같은 이름의 `nabic` 서브명령을 실행합니다. 도구는 컴파일러를 링크해 담은 `nabi`의 프로세스 안에서 실행합니다(9.10.8). 어느 쪽이든 `nabi`는 자기 전역 옵션(`--env` / `--offline` / `--locked` / `--frozen` / `--cwd` / `--color`, `--` 앞 어디든)만 소비합니다. 나머지 인자는 그대로 컴파일러에 넘기고 다음을 더합니다. `test`와 `bench`에서 `nabi`는 `nabic`로 하네스를 빌드하게만 하고(`--emit-harness`) 실행은 직접 합니다([9.1](901-nabic-cli.md)). 그래서 러너는 컴파일러에 속하지 않고, 러너의 옵션(아래)도 `nabi`의 것입니다.

- 의존 패키지마다 `--extern name=<source directory>`. `nabi.lock`을 따라 준비하고, 의존되는 쪽이 의존하는 쪽보다 앞에 옵니다(`nabi deps`가 출력하는 순서). 미리 빌드는 `nabic`가 합니다([9.2.3](902-aot-pipeline.md#미리-빌드한-의존-패키지)).
- 패키지마다 그 의존성마다 `--extern-dep <package>=<name>`(빌드 중인 패키지는 `pkg`). 그래서 각 패키지는 자기 의존성만 봅니다([5.7.14](../005-modules/507-paths.md#5714-패키지-사이-경로)).
- 패키지 자신의 feature는 `--cfg feature=<f>`로, 각 의존 패키지의 feature는 `--extern-feature <name>=<f>`로 넘깁니다(9.10.4c). build / run / test / bench / check / lint / fix / doc / emit / repl / lsp / jupyter / deps의 feature 옵션은 `nabi`가 소비합니다(`--` 앞). 이런 옵션에는 `--features <f,..>`(여러 번 가능, `<dep>/<f>`도), `--all-features`, `--no-default-features`가 있습니다.
- 매니페스트의 `lints`를 `-A` / `-W` / `-D`로 넘깁니다(인자 앞에). 매니페스트는 경로가 고르는 프로젝트의 것입니다. `.nabi` 파일이면 그 파일 위치나 그 위에서 `src/main.nabi`, `src/lib.nabi`, `nabi.nabion` 중 하나가 있는 가장 가까운 디렉터리입니다. 패키지 `src/` 안의 파일은 그 패키지를 고릅니다([9.1.1](901-nabic-cli.md#911-일반-형태)). `nabi.nabion`이 있는 디렉터리의 그 밖의 단독 파일이 있습니다. 패키지 `src/` 옆의 스크립트나 `nabi.nabion`만 있는 디렉터리의 파일이 그렇습니다. 이런 파일은 그 매니페스트의 `lints`(와 `lints_from`)와 의존성을 쓰는 단일 파일 패키지로 컴파일합니다.
- 패키지 밖(환경, 스크립트)이면 `--target-dir`에 그 환경이나 스크립트의 `target/`을 넘깁니다.

`nabi cc`는 위임하는 서브명령도 도구도 아닙니다. toolchain의 `nabicc`를 실행하고 `cc` 뒤의 인자를 모두 그대로 넘기며, 아무것도 더하지 않습니다([9.13.1](913-c-compiler.md#9131-nabicc와-nabi-cc)).

`nabi repl <path>`는 `run`처럼 그 경로의 프로젝트에서 의존성과 린트를 정하고, 매니페스트 이름을 씁니다(`--name`).

`nabi test`의 러너는 다음 옵션을 받습니다(`--` 앞). `nabic`에는 넘기지 않습니다.

- `--threads=N`은 동시에 실행할 테스트 프로세스 수를 지정합니다. 기본값은 `NABI_TEST_THREADS`이고, 그것도 없으면 CPU 수입니다. `NABI_TEST_THREADS`가 양수가 아니면 경고하고(`nabi: warning: ignoring NABI_TEST_THREADS=0 (expected a positive number)`) 무시합니다.
- `--show-output`은 성공한 테스트의 stdout / stderr도 표시합니다. 실패한 테스트의 출력은 늘 표시합니다.
- `--format=human|json`은 출력 형식을 고릅니다(기본값 `human`). `json`은 테스트마다 JSON 레코드 하나와 summary를 stdout에, 사람이 읽는 텍스트를 stderr에 씁니다([9.9.12](909-debug-tools.md#9912-ci--테스트-통합)).
- `--exact`는 필터와 `--skip`이 테스트 이름 전체와 맞아야 하게 합니다. `nabi test --exact math::adds`는 그 테스트만 실행하고 `math::adds_wide`는 실행하지 않습니다.
- `--skip <pat>`은 `<pat>`과 맞는 테스트를 필터와 같은 규칙으로 뺍니다(여러 번 쓸 수 있음).
- `--ignored`는 `#[ignore]` 테스트만, `--include-ignored`는 나머지와 함께 그 테스트도 실행합니다.
- `--timeout=SECS`는 그보다 오래 실행되는 테스트를 강제 종료하고 timeout 실패로 보고합니다. 기본값은 `NABI_TEST_TIMEOUT`, 없으면 600입니다. `0`은 제한 없음입니다. 정수 초가 아닌 `NABI_TEST_TIMEOUT` 값은 경고를 내고 무시합니다.
- `--list`는 필터가 남긴 테스트의 이름을 한 줄에 하나씩 출력하고 아무것도 실행하지 않습니다(종료 코드 `0`). `#[ignore]` 테스트 뒤에는 ` (ignored)`가 붙습니다. `--format=json`이면 이름마다 레코드 하나를 씁니다: `{"schema":1,"type":"listed","kind":"test","name":"adds"}`. ignored 테스트에는 `"ignored":true`가 붙습니다.

`--doc`과 `--no-doc`은 `nabic`에 넘어갑니다. 문서 테스트만 실행하거나, 문서 테스트를 빼고 실행합니다([9.1.2](901-nabic-cli.md#문서-테스트)).

`nabi bench`의 러너는 다음 옵션을 받습니다.

- `--iters=N`은 보정 대신 표본 하나의 호출 횟수를 고정합니다.
- `--samples=N`은 측정할 표본 수입니다(기본값 10).
- `--format=human|json|nabibench`는 출력 형식을 고릅니다(기본값 `human`). `json`은 테스트와 같고, `nabibench`는 [9.1.2](901-nabic-cli.md#nabic-bench-벤치마크)의 고정된 줄입니다.
- `--list`는 테스트처럼 필터가 남긴 벤치마크의 이름을 출력하고 아무것도 실행하지 않습니다.

둘 다 다른 `--target`의 실행 파일은 그 타깃의 러너로 실행합니다([다른 타깃의 실행 파일](#다른-타깃의-실행-파일)).

잘못된 값은 무엇을 써야 하는지 알려 주는 사용법 에러입니다: `` `--threads` takes a positive number, not `0` ``, `` `--timeout` takes whole seconds (0: no limit), not `1.5`: `--timeout=120` ``, `` `--format` takes human or json, not `xml` ``. 러너가 출력하는 내용은 `nabic test`와 `nabic bench`에서 설명합니다([9.1.2](901-nabic-cli.md#nabic-test-테스트)).

옵션과 출력, 종료 코드는 `nabic`의 것([9.1](901-nabic-cli.md))을 따릅니다. `nabi`는 닫힌 표준 출력이나 표준 에러를 `nabic`와 같은 방식으로 다룹니다([9.1.6](901-nabic-cli.md#916-종료-코드)). `nabils`는 `nabi lsp`와 같은 독립 실행 파일입니다.

`nabi help`(`nabi --help`도)는 짧은 페이지를 출력합니다. 묶음별로 서브명령을 한 줄씩 적고 전역 옵션을 적습니다. `nabi help <subcommand>`(`nabi <subcommand> --help`도)는 그 서브명령만의 도움말을 출력합니다. 사용 형태, 설명, 예제, 옵션이 들어갑니다. 위임하는 서브명령이면 `nabic`의 도움말을 `nabi`로 고쳐 쓰고 `nabi`의 feature 옵션을, `test`와 `bench`에는 러너의 옵션도 더합니다.

모르는 서브명령이나 옵션은 사용법 에러(종료 코드 2)입니다. 가장 가까운 서브명령이나 옵션이 있으면 메시지가 그 이름을 댑니다. 흔히 쓰는 다른 이름도 알아듣습니다(`build` 대신 `compile`, `remove` 대신 `rm`). 이어서 ``run `nabi help <subcommand>` for usage``를 출력합니다. index에 없는 패키지나 native 패키지는 그 index에서 가장 가까운 이름과 실행할 `nabi search`를 알려 줍니다. `nabi`는 자신이 실행하는 `nabic`에 `__NABI_CLI=nabi`을 설정합니다. 그래서 `nabic`의 사용법 에러, 도움말, 에러와 경고 줄의 접두사도 `nabi`로 나옵니다([9.1.7](901-nabic-cli.md#917-환경-변수)).

서브명령 없이 `nabi [options]`만 실행하면 `nabi repl`과 같이 REPL을 시작합니다.

```
nabi                 # = nabi repl
nabi --env sci       # = nabi repl --env sci
```

`nabi [options] <file> [args..]`은 스크립트를 실행합니다. `nabi run -q <file> -- args..`와 같습니다. 파일 뒤의 모든 인자는 그대로 프로그램에 넘기고, 종료 코드는 프로그램의 것입니다. 그 단어가 `/`(Windows에서는 `\`도)를 담거나 확장자가 `.nabi`면 파일로 봅니다. 그런 서브명령은 없으므로 서브명령을 파일로 잘못 아는 일은 없습니다. shebang 줄([1.2.3](../001-lexical/102-lexical-structure.md#shebang-줄))과 함께 쓰면 `./script.nabi args`가 스크립트를 실행합니다.

위임하는 명령과 도구의 경로는 옵션도 옵션의 값도 아닌 첫 인자입니다. 그래서 옵션이 그 앞에 와도 됩니다(`nabi run --release t.nabi`). 없는 경로는 다른 무엇보다 먼저 `` `<path>` does not exist `` 에러가 됩니다("not inside a package"가 아님). `test`와 `bench`는 첫 단어가 필터일 수 있으므로 `.nabi`로 끝나는 단어만 경로로 봅니다.

`nabi`의 전역 옵션은 다음과 같습니다.

| 옵션             | 의미                                                              |
|------------------|-------------------------------------------------------------------|
| `--env <name>`   | 그 이름의 환경에서 작업함(REPL, Jupyter, `nabi install`). 패키지에 작용하는 명령에서는 에러([9.10.5](#9105-환경)) |
| `--offline`      | 네트워크를 쓰지 않음. git 의존은 캐시에 있어야 함                 |
| `--locked`       | `nabi.lock`을 바꿔야 하면 바꾸지 않고 에러(CI용)                 |
| `--frozen`       | `--locked`와 `--offline`을 함께                                   |
| `--cwd <dir>`    | 디렉터리를 바꾼 뒤 실행                                           |
| `-q`, `--quiet`  | 출력을 줄임                                                       |
| `-v`, `--verbose` | 출력을 늘림(`nabic`에도 넘김)                                    |
| `--color <when>`, `--no-color` | `auto`, `always`, `never` 중 하나로 색을 정함. `nabic`와 같은 방식([9.1.5](901-nabic-cli.md#915-전역-옵션)). 실행하는 `nabic`에 `--color=<when>`으로 넘김 |

전역 옵션은 서브명령 뒤에 와도 됩니다. `nabi new app -q`는 `nabi -q new app`과 같습니다.

환경 변수는 `NABI_HOME`(기본 `~/.nabi`), `NABI_ENV`(9.10.5), `NABI_TOOLCHAIN`(9.10.6), `NABI_REGISTRY_TOKEN`(기본 레지스트리의 API 토큰, 9.10.4a), `NABI_DIST`(릴리스 서버, 9.10.6)입니다. `NO_COLOR`, `CLICOLOR_FORCE`, `COLORTERM`, `TERM`은 `nabic`와 같은 방식으로 색을 정합니다([9.1.7](901-nabic-cli.md#917-환경-변수)).

## 9.10.2. `nabi new` / `nabi init`: 패키지 생성

```
nabi new [--lib] [--bin] <name>
nabi init [--lib] [--bin] [dir]
nabi init --script <file.nabi>
```

`nabi new`는 `<name>` 디렉터리를 만들고 기본 파일을 채웁니다. `--bin`(기본값)은 인사말을 출력하는 `src/main.nabi`를 만듭니다. `--lib`은 예시 함수와 `#[test]` 하나를 담은 `src/lib.nabi`를 만듭니다. 둘을 함께 주면 둘 다 만들고, 이때 `src/main.nabi`는 `use <name>::add;`로 라이브러리의 함수를 부릅니다([0.3.1](../000-meta/003-source-units.md#031-실행-파일과-라이브러리)). `<name>`이 이미 있으면 에러입니다.

`nabi init`은 이미 있는 디렉터리(생략하면 현재 디렉터리)를 패키지로 만듭니다. 그 디렉터리에 `src/main.nabi`나 `src/lib.nabi`가 이미 있으면 에러입니다.

`nabi init --script <file.nabi>`은 스크립트([9.10.7](#9107-단일-파일-스크립트의-인라인-매니페스트))를 패키지로 바꿉니다. 패키지는 그 스크립트의 디렉터리에 두고, 이름은 파일 이름에서 `.nabi`를 뗀 것입니다. `nabi.nabion`은 스크립트 블록을 합친 `deps`(와 `native`)를 받고, 경로는 그 파일 기준으로 다시 씁니다. 스크립트는 `src/main.nabi`가 됩니다. 그 `mod`가 읽는 모든 파일은 같은 상대 경로로 `src/` 아래로 옮기고, 각 파일의 블록은 뺍니다. 다음 경우는 에러입니다. 스크립트가 패키지 안에 있을 때, 옮길 파일이나 `nabi.nabion`이 이미 있을 때, 서브모듈 파일이 스크립트의 디렉터리 밖에 있을 때입니다.

두 명령은 정규 형식([NABION 3.8.4](../../refs-standard-libraries/300-std/308-nabion.md#384-정규-형식))의 `nabi.nabion`과 `target/`을 담은 `.gitignore`도 씁니다. `nabi.nabion`의 `nabi_version`은 그것을 쓰는 `nabi`의 버전이고(`nabi_version: "0.1.3"`, prerelease 부분은 뺌), toolchain은 그 `nabi`의 시리즈로 고정합니다(0.1.x 어느 것이든 `toolchain: "0.1"`, 9.10.6). 그래서 더 새 시리즈가 기본값이 되어도 패키지는 그 시리즈에 머뭅니다. git 저장소 안이 아니면 `git init`을 실행합니다. `init`은 디렉터리 이름을 이름 규칙(9.10.3)에 맞게 바꿔 패키지 이름으로 씁니다. Windows 장치 이름이면 앞에 `pkg-`를 붙이고(`nul` → `pkg-nul`), 64자를 넘으면 자릅니다.

두 명령은 `Created` 줄 아래에 이어서 실행할 명령을 출력합니다. 보통은 `cd <dir> && nabi run`이고, 라이브러리만 있는 패키지는 `nabi test`입니다. 패키지가 현재 디렉터리면 `cd` 없이 씁니다. Windows PowerShell에는 `&&`가 없으므로 Windows에서는 `cd <dir>`와 그 명령을 두 줄로 씁니다. `-q`면 둘 다 출력하지 않습니다.

```
     Created executable package (hello)
             cd hello && nabi run
```

Windows에서는 다음과 같습니다.

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

패키지 루트의 `nabi.nabion`이 매니페스트이고, 같은 자리의 `nabi.lock`이 lockfile입니다. 둘 다 [NABION](../../refs-standard-libraries/300-std/308-nabion.md) 형식입니다. 이름은 이 철자 그대로(모두 소문자)입니다. 대소문자만 다른 파일(`Nabi.nabion` 등)은 경고를 내고 무시하며, 읽지 않습니다.

매니페스트는 NABION 값 하나, 곧 `Package { .. }` struct입니다(루트 이름 `Package`는 생략 가능). 매니페스트는 읽기만 하고 컴파일하거나 실행하지 않습니다.

```nabion
Package {
    name: "geom",                    // 문자열, 선택
    version: "0.3.1",                // 문자열, 선택
    nabi_version: "0.4.1",           // 문자열, 선택 (빌드되는 가장 오래된 nabi, `nabi publish`에 필요)
    toolchain: "0.4",                // 문자열, 선택 (toolchain 고정, 9.10.6)
    description: "2D geometry",      // 문자열, 선택
    license: "MIT OR Apache-2.0",    // 문자열, 선택 (SPDX 식)
    repository: "https://example.com/geom.git",   // 문자열, 선택
    authors: ["Ann Lee <ann@example.com>"],       // 문자열 목록, 선택
    homepage: "https://example.com/geom",         // 문자열, 선택
    documentation: "https://example.com/geom/doc",   // 문자열, 선택
    readme: "README.md",             // 문자열, 선택 (nabi.nabion 기준 경로)
    keywords: ["geometry", "math"],  // 문자열 목록, 선택 (많아야 5 개)
    categories: ["mathematics"],     // 문자열 목록, 선택 (많아야 5 개)
    license_file: "LICENSE",         // 문자열, 선택 (nabi.nabion 기준 경로)
    include: ["src/**", "README.md"],   // glob 목록, 선택 (게시할 때 담을 파일)
    exclude: ["bench/**"],           // glob 목록, 선택 (뺄 파일)
    links: "z",                      // 문자열, 선택 (링크하는 native 라이브러리)
    default_run: "geom",             // 문자열, 선택 (`nabi run` 이 실행할 실행 파일)
    deployment_target: "13.0",       // 문자열, 선택 (프로그램이 도는 가장 오래된 macOS;
                                     // iOS도 정하려면 { macos: "13.0", ios: "16.0" })
    metadata: { docs: { all_features: true } },   // 표, 선택 (다른 도구용)
    publish: true,                   // bool, 선택 (false: `nabi publish` 가 거부)
    deps: {                          // 의존 이름 (문자열) -> Dep
        "util": Dep::path("../util"),
        "parser": Dep::git("https://example.com/parser.git", tag: "v1.2"),
        "http": "0.9",                                   // index에서 (= Dep::version("0.9"))
        "json": Dep::version("^1.4", optional: true),    // feature 가 켤 때만 (9.10.4c)
        "csv": Dep::version("^0.3", registry: "https://example.com/index"),
        "j5": Dep::version("^0.2", package: "json5"),    // 패키지 `json5`, 코드에서는 `j5`
        "nix": Dep::version("^0.4", target: "cfg(target_family = \"unix\")"),
    },
    dev_deps: {                      // deps 와 같은 형태. test / bench 에서만
        "fixtures": Dep::path("tests/fixtures"),
    },
    features: { "default": ["json"], "json": ["dep:json"], "tls": ["http/tls"], "full": ["json", "tls"] },
    native: {                        // C 라이브러리 (9.10.4b)
        "zlib": "^1.3",                                  // 미리 빌드한 native 패키지
        "png": Dep::native(url: "vendor/png.nabion", sha256: "<64 hex>"),
        "sqlite3": Dep::system(brew: "sqlite", apt: "libsqlite3-dev"),   // 기계에 설치된 것
        "Foundation": Dep::framework(),                  // Apple 프레임워크
    },
    lib: { kind: "cdylib" },         // 선택 (C용 공유 라이브러리도, 9.3.3; "plugin": 9.12)
    c: { sources: ["c/shim.c"], include: ["c"], define: { "N": "1" }, std: "c17" },   // 선택
    lints_from: "..",                // 문자열, 선택 (../nabi.nabion 의 lints 에서 시작)
    lints: { "dead_code": "allow", "nonstandard_style": "deny" },
    profile: { "release": { overflow_checks: false } },   // 빌드 프로필별 설정
}
```

- 값은 NABION의 리터럴(문자열, `{ "k": v }` dict, 빈 dict `{}`)과 `Dep::path("..")` / `Dep::git("url", rev | tag | branch: "..")` / `Dep::version("req", registry: "url")`입니다. `Dep::git`에는 `rev` / `tag` / `branch` 중 많아야 하나만 씁니다. `Dep::version`의 `registry`는 선택입니다. 의존성 값이 문자열 하나면 `Dep::version`을 줄여 쓴 것입니다.
- `Dep::path("..", version: "req")`는 path 의존성에 버전 요구사항을 더합니다. 빌드는 그 경로를 쓰고, 그 경로의 패키지 버전은 요구사항을 만족해야 합니다(`` dependency `linalg` (path ..) is version 0.2.0, which its `version: "^0.1"` does not accept; .. ``). `nabi publish`는 경로 대신 요구사항을 씁니다(9.10.4a). 나란히 개발하는 패키지는 이렇게 게시합니다.
- `Dep::path` / `Dep::git` / `Dep::version`은 `optional: true`, `features: ["<f>", ..]`, `default_features: false`도 받습니다(9.10.4c). optional인 dev 의존은 에러입니다.
- `package: "<name>"`은 의존성의 이름을 바꿉니다. `<name>`이 패키지이고, 키는 이 패키지의 코드가 그 패키지를 부르는 이름입니다(`"j": Dep::version("^1", package: "json5")`는 `json5`이고 `use j::..`로 씁니다). 이름은 의존하는 패키지마다 따로 정하고 그 패키지의 코드만 보므로, 두 패키지가 한 패키지를 서로 다른 이름으로 부를 수 있습니다. 그래도 한 패키지이고 한 번만 빌드하며, 어느 이름으로 불러도 타입은 같습니다. 한 패키지에 키 둘로 의존하면 에러입니다(`` the root package depends on package `json5` twice, as `j` and `json5`; depend on it once ``).
- `target: "<target>"`은 의존을 맞는 target의 빌드로 한정합니다. target 이름(`x86_64-unknown-linux-gnu`)이나, [6.3.2](../006-meta-system/603-cfg.md#632-predicate-종류)의 target 키(`target_os`, `target_arch`, `target_family`, `target_env`, `target_endian`, `target_pointer_width`)에 대한 `cfg(<predicate>)`이고, `all(..)`, `any(..)`, `not(..)`을 씁니다. 그 밖의 값은 매니페스트를 읽을 때 에러입니다. 해소와 lock은 모든 target에 대해 그 의존을 다룹니다. 다른 target의 빌드는 그 의존과, 그 의존이 켤 feature를 뺍니다.
- 의존성에 `plugin: true`를 쓰면 그 의존성을 링크하지 않고 프로그램 옆에 플러그인으로 빌드합니다([9.12.2](912-plugins.md#9122-빌드)). 플러그인의 의존성에 `bundle: true`를 쓰면 프로그램에서 가져오는 대신 플러그인에 링크합니다. 플러그인이 아닌 패키지의 `bundle`은 에러이며(`` `bundle: true` on `noise` belongs in a plugin's dependencies: give this package `lib: { kind: "plugin" }`, or remove `bundle: true` ``), 둘을 함께 쓴 의존성도 에러입니다.
- `Dep::registry`는 `nabi.lock` 전용입니다. `nabi.nabion`에 쓰면 에러입니다.
- 패키지 이름(`deps`의 키, `name`)은 `[a-z0-9_-]+`의 소문자 ASCII이고 글자로 시작해야 합니다. 숫자, `-`, `_`로 시작하면 에러입니다.
- 이름 `std`, `core`, `bare`는 toolchain과 함께 배포되는 표준 라이브러리 계층입니다. 패키지, native 패키지, 레지스트리 이름이 그중 하나면 에러이고(`` `std` is the standard library, which ships with the toolchain; name the package something else, e.g. `std_util` ``), 그 이름으로 `nabi new` / `nabi init`을 실행해도 마찬가지입니다. `nabi add std`는 더할 것이 없다고 알려 줍니다. `deps`의 키가 그중 하나면 help가 `` std ships with the toolchain; remove this line ``인 에러입니다. 다만 `package: "<name>"`이 있으면 그 패키지를 계층 이름으로 부르는 것입니다([5.7.4](../005-modules/507-paths.md#574-std-core-bare-표준-라이브러리-계층)). `package: "std"`도 같은 에러입니다.
- 코드가 패키지를 이름으로 부를 수 있어야 하므로 코드 이름(아래)은 키워드(1.5: `fn`, `match`, `self`, ..)나 경로 시작점(5.7.1: `pkg`, `sup`)일 수 없습니다. 실행 파일의 `use pkg::add;`는 언제나 자기 루트를 가리킵니다(`` `pkg` already names a package's own root (`pkg::`), so code can't name a package called that; name the package something else, e.g. `pkg_util` ``). 키는 문자열입니다(`"nabi-syntax": Dep::version("^0.1")`).
- 패키지 이름은 지원하는 모든 OS가 파일이나 디렉터리 이름으로 쓸 수 있어야 합니다. Windows 장치 이름(`con`, `prn`, `aux`, `nul`, `com1`-`com9`, `lpt1`-`lpt9`, 대소문자 무관)이 아니어야 하고 64자를 넘을 수 없습니다(`` `nul` is a device name on Windows, so no file can be called that; name the package something else, e.g. `my_nul` ``). `nabi new` / `nabi init --script` / `nabi add` / `nabi publish`는 이런 이름을 거부합니다. 읽는 `nabi.nabion`(`name`과 `deps`, `dev_deps`, `native`의 키)에 있으면 `file:line:col`이 붙은 경고를 내고 빌드는 계속합니다.
- `version`은 버전(`major.minor.patch`, 9.10.4a)입니다. 버전이 아니면 매니페스트를 읽을 때 경고하고(`` `banana` is not a version (major.minor.patch): it needs three numbers; `nabi publish` needs one, e.g. `version: "0.1.0"` ``), `nabi publish`에서는 에러입니다.
- `nabi_version`은 패키지가 빌드되는 가장 오래된 toolchain이고, 전체 버전 `x.y.z`입니다. 그보다 짧거나 길면 매니페스트를 읽을 때 에러입니다(`` `nabi_version: "0.1"` is not a toolchain version; write all three parts, e.g. `nabi_version: "0.1.3"` ``). 그 toolchain과 같은 시리즈의 이후 toolchain을 허용합니다([9.10.6 호환성](#호환성)). `"0.1.3"`은 0.1.3부터 마지막 0.1.x까지, `"1.4.2"`는 1.4.2부터 마지막 1.x까지입니다. `nabi publish`에 필요하고(9.10.4a), 해소는 이것으로 버전을 고릅니다. 이 프로젝트를 빌드할 toolchain을 고정하는 `toolchain`(9.10.6)과는 다릅니다.
- 코드에서는 `-`를 `_`로 바꾼 이름이 그 패키지를 가리키는 최상위 경로입니다(`nabi-syntax` → `use nabi_syntax::..`, `--extern nabi_syntax=..`).
- 레지스트리와 한 의존 그래프 안에서 이름은 `-`와 `_`를 같게, 대소문자를 구분하지 않고 비교합니다. 그래서 `nabi-syntax`와 `nabi_syntax`는 함께 있을 수 없습니다.
- 의존 패키지에는 `src/lib.nabi`가 있어야 합니다. 그 패키지에 `src/main.nabi`가 있든 없든, 의존하는 쪽이 얻는 것은 그 라이브러리입니다. `src/main.nabi`만 있는 의존 패키지는 에러이고, 에러는 그 패키지가 `nabi install`용 프로그램이라고 알립니다. 전이 의존은 의존 패키지의 `deps`입니다(`dev_deps`는 전이되지 않음).
- `native`는 native 패키지 이름을 `Dep::native("req"[, registry: "url"])`, 요구 문자열, 또는 `Dep::native(url: "<url 또는 매니페스트 기준 경로>", sha256: "<64 자리 hex>")`에 연결합니다(9.10.4b). native 패키지 이름은 따로 이름 공간을 쓰므로 패키지와 native 패키지는 같은 이름을 쓸 수 있습니다. 이름을 기계에 설치된 라이브러리인 `Dep::system(..)`이나 Apple 프레임워크인 `Dep::framework()`에도 연결할 수 있고, 이 둘은 빌드할 때 찾습니다(9.10.4b). `deps` / `dev_deps`의 `Dep::native`는 에러입니다. 그래프에 있는 모든 패키지의 native 의존이 실행 파일에 링크됩니다(`dev_deps`로만 닿는 것은 test / bench에서만).
- `lib: { kind: "nabi" | "cdylib" | "staticlib" | "plugin" }`(레코드 이름은 `Lib`이어도 됨)는 라이브러리가 무엇을 빌드하는지 정합니다. 기본값 `"nabi"`는 다른 Nabi 패키지가 쓰는 라이브러리입니다. `"cdylib"`이면 명령줄에 `--crate-type`이 없는 한 `nabi build`가 `--crate-type=cdylib`와 패키지 `version`의 `--lib-version`을 넘기므로([9.1.2](901-nabic-cli.md#nabic-build-aot-빌드)) 빌드는 공유 라이브러리와 그 C 헤더도 만듭니다([9.3.3](903-package-archive.md#933-정적-라이브러리와-공유-라이브러리)). `"staticlib"`이면 `--crate-type=staticlib`을 넘기므로 빌드는 C와 Swift가 그것 하나만 링크하면 되는 정적 라이브러리와 그 헤더, module map도 만듭니다(9.3.3). 그 옆의 `xcframework: true`는 `--xcframework`도 넘기며, 다른 종류에 쓰면 에러입니다. `"plugin"`이면 `--crate-type=plugin`을 넘기며, 패키지는 Nabi 프로그램이 로드하는 공유 라이브러리가 됩니다([9.12](912-plugins.md)). `src/main.nabi`가 있는 `"plugin"` 패키지는 에러이며, 그것에 `nabi run`을 해도 에러입니다. 다른 종류는 네 가지를 알려 주는 에러입니다.
- `c: { sources, include, define, std, flags }`(레코드 이름은 `C`이어도 됨)는 Nabi의 자체 C 컴파일러 nabicc가 패키지에 넣어 빌드하는 C 파일입니다. 시스템 C 컴파일러는 쓰지 않습니다. `sources`(매니페스트 기준 경로, 하나 이상), `include`(헤더 디렉터리, 그 뒤에 패키지의 `Dep::system` 라이브러리의 것), `define`(`{ "NAME": "value" }`), `std`(`c89`, `c99`, `c11`, `c17`, `c23` 또는 `gnu` 판), `flags`입니다. `flags`는 `-O*`, `-g`, `-W*`, `-f*`(`-flto` 제외), `-m*`만 받고, 다른 것은 이 목록을 알려 주는 에러입니다. 오브젝트는 `target/<profile>/c/<name>/`에 쓰고 링크에 넣습니다(`--native-object`). 입력(소스와 그것이 읽은 헤더)이나 옵션이 바뀐 소스만 다시 컴파일합니다([9.13.8](913-c-compiler.md#9138-패키지의-c-소스)).
- `lints`는 린트 수준을 정합니다(린트나 그룹 이름 → `"allow"` | `"warn"` | `"deny"`, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). 이 수준은 `nabi`가 실행하는 모든 컴파일 명령(`nabic`에 위임하는 명령과 도구)의 CLI 플래그 앞에 붙은 것처럼 다룹니다. 그래서 CLI `-A/-W/-D`와 코드의 속성이 이를 덮습니다.
- `lints_from`은 시작 린트 수준을 주는 `nabi.nabion`이 있는 디렉터리를 가리킵니다(이 매니페스트의 디렉터리 기준 상대 경로, 또는 절대 경로). 먼저 그 매니페스트의 `lints`를 그 자신의 `lints_from` 사슬 뒤에 적용합니다. 이 매니페스트의 `lints`는 그 뒤에 옵니다. 그래서 여기 항목이 사슬의 같은 린트나 그룹을 덮습니다. 모든 항목은 이 순서로 `-A`/`-W`/`-D`로 넘어가고, 뒤의 플래그가 이깁니다. 이 설정은 빌드하거나 검사하는 패키지(와 환경의 매니페스트)에만 적용합니다. 의존 패키지의 매니페스트는 린트에 쓰지 않습니다. 디렉터리가 없거나, `nabi.nabion`이 없거나, 사슬이 이미 지난 매니페스트로 돌아오면(순환) 에러입니다. `nabi env import`는 상대 `lints_from`을 절대 경로로 바꿔 씁니다. 스크립트의 `//! ```nabion` 블록에서는 쓸 수 없습니다.
- `profile`은 빌드 프로필(`"debug"` 또는 `"release"`, [9.2.15](902-aot-pipeline.md#9215-산출물))을 그 설정 `{ overflow_checks: bool }`에 대응시킵니다(레코드 이름 `Profile`은 써도 됩니다). `overflow_checks: false`는 `-C overflow-checks=no`([9.1.2](901-nabic-cli.md#nabic-build-aot-빌드))처럼 그 프로필에서 정수 오버플로 검사를 끕니다. 명령줄의 `-C overflow-checks`가 이 설정보다 우선합니다. 다른 프로필 이름이나 설정은 에러입니다. 빌드하는 패키지(또는 환경)의 것만 읽고 의존 패키지의 프로필은 읽지 않습니다. 스크립트의 `//! ```nabion` 블록에는 쓸 수 없습니다.
- 패키지 메타데이터
  - `authors`는 `"Name <email>"` 문자열입니다. `nabi new` / `nabi init`은 채우지 않습니다.
  - `keywords`와 `categories`는 각각 많아야 5개입니다. `authors`, `keywords`, `categories`, `include`, `exclude`의 빈 항목은 에러입니다.
  - `readme`와 `license_file`은 매니페스트의 디렉터리 기준으로 패키지 안의 경로입니다. 절대 경로나 `..`로 패키지를 벗어나는 경로는 에러입니다. `readme`가 없으면 `README.md`가 있을 때 그 파일이 readme입니다.
  - `metadata`는 `nabi`가 읽지 않는 표(`{ .. }`)이고, 적은 그대로 레지스트리로 갑니다.
  - `nabi publish`는 readme와 `license_file`이 있는지, `default_run`이 패키지의 실행 파일 이름인지 검사하고, `publish: false`인 패키지는 거부합니다(9.10.4a).
- `include` / `exclude`는 게시하는 커밋의 파일 경로에 대한 glob입니다(`*`와 `?`는 이름 하나 안에서, `**`는 여러 이름에 걸쳐 맞습니다. `/`가 없는 패턴은 어느 깊이의 이름에나 맞고, 디렉터리에 맞는 패턴은 그 안의 파일에도 맞습니다). `include`가 비었거나 파일에 맞고 `exclude`가 맞지 않으면 패키지가 그 파일을 담습니다. `nabi.nabion`은 늘 담습니다. 이 둘은 `nabi publish`가 보내는 묶음 크기를 정합니다(9.10.4a). source는 여전히 커밋 전체입니다.
- `links`는 패키지가 링크하는 native 라이브러리의 이름입니다. 한 빌드의 두 패키지가 같은 `links`를 쓰면 둘을 밝히는 에러입니다.
- `default_run`은 `nabi run`이 실행할 실행 파일의 이름입니다. 패키지의 실행 파일은 많아야 하나(`src/main.nabi`, 패키지 이름)이므로 그 이름입니다.
- `deployment_target`은 패키지의 프로그램이 도는 가장 오래된 macOS로, 숫자로 씁니다(`"13.0"`. 다른 형태는 그것을 보여 주는 에러). 생략하면 `11.0`입니다. 레코드 형태 `{ macos: "13.0", ios: "16.0" }`는 가장 오래된 iOS도 정하며(생략하면 `13.4`, 그보다 낮으면 에러), 각 필드는 선택입니다. `nabi`는 이 값들을 `MACOSX_DEPLOYMENT_TARGET`과 `IPHONEOS_DEPLOYMENT_TARGET`으로 `nabic`에 넘기며, 그 변수가 이미 있으면 그쪽이 우선합니다. 이보다 새로운 Objective-C API는 `objc::available` 안에서만 부릅니다([std 1.10.13](../../refs-standard-libraries/100-bare/110-objc.md#11013-가용성과-배포-타깃)).
- `nabi`가 모르는 필드는 `file:line:col`이 붙은 경고이고(`` unknown field `licence` is ignored; did you mean `license`? ``, 가까운 이름이 있으면 제안), 매니페스트는 그 필드 없이 읽습니다. `nabi publish`는 그 필드를 적은 그대로 `meta`에 담아 보냅니다(9.10.4a). 그래서 새 레지스트리가 그 필드를 쓸 수 있습니다.
- 다음은 에러입니다. 모르는 `Dep` 종류 / 이름 있는 인자, 중복, 9.10.4c의 규칙을 어기는 `features`, 규칙에 맞지 않는 패키지 이름, NABION이 아닌 문법, `Package { .. }` 하나가 아닌 값, 뒤에 남은 토큰. 진단은 `file:line:col`과 캐럿(^)으로 위치를 보여 줍니다.

| 의존성 출처 | 형태                                                         | `nabi.lock`에 고정  |
|-------------|--------------------------------------------------------------|---------------------|
| path        | `Dep::path(".."[, version: "req"])`(매니페스트 기준 상대 경로) | 없음(늘 현재 내용)  |
| git         | `Dep::git("url")`, 선택으로 `rev:` / `tag:` / `branch:` 중 하나 | 커밋과 tree 해시    |
| registry    | `Dep::version("req"[, registry: "url"])` 또는 문자열 `"req"`([9.10.4a](#9104a-레지스트리-패키지)) | 버전, 커밋, tree 해시(index와 함께, 미러는 적지 않음) |
| native      | `Dep::native("req")` / `Dep::native(url:, sha256:)`([9.10.4b](#9104b-네이티브-패키지)) | 버전과 항목 SHA-256(index), 없음(해시가 고정) |
| system      | `Dep::system(..)` / `Dep::framework()`([9.10.4b](#9104b-네이티브-패키지)) | 안 됨(빌드마다 찾음) |

Nabi에는 빌드 스크립트가 없습니다. 빌드할 때 패키지의 코드는 실행되지 않습니다. C 라이브러리는 미리 빌드된 native 패키지, 기계에 설치된 라이브러리, 또는 nabicc가 컴파일하는 C 소스(`c`, [9.13.8](913-c-compiler.md#9138-패키지의-c-소스))로 들어오고, 모두 매니페스트에 적습니다(9.10.4b).

한 빌드 안에서 한 이름은 한 패키지입니다. 같은 이름이 서로 다른 출처(다른 경로, 또는 다른 url / 커밋)로 요구되면 에러입니다. 에러는 두 요구처를 보여 줍니다. 두 의존하는 쪽이 같은 코드 이름을 서로 다른 패키지에 붙여도 에러입니다(`` `hello` is package `other` for the root package and package `greet` for `mid` ``). 한 패키지는 코드 이름을 여럿 가질 수 있습니다(`package`). `nabi`는 이름마다 같은 경로로 `--extern`을 넘깁니다.

### 워크스페이스

함께 개발하는 패키지들은 lock 하나와 빌드 디렉터리 하나를 함께 쓸 수 있습니다. **워크스페이스**의 루트에는 패키지 대신 `Workspace { .. }` 값을 담은 `nabi.nabion`이 있습니다.

```nabion
Workspace {
    members: ["app", "libs/*"],   // 루트 아래의 디렉터리. `dir/*`: `dir` 안의 패키지 하나하나
}
```

- `members`는 루트 기준 상대 디렉터리를 나열합니다. `dir/*`는 `dir`의 바로 아래 하위 디렉터리 가운데 `nabi.nabion`이 있는 것 전부입니다. 절대 경로, `..` 단계, 그 밖의 `*`는 에러입니다(`` `../x` is not a member: write a directory under the workspace root, or `dir/*` for each package in `dir` ``). `nabi.nabion`이 없는 멤버 디렉터리도 에러입니다(`` member `gone` has no nabi.nabion (..); create the package with `nabi new gone`, or remove it from `members` ``). 멤버마다 `name`이 있어야 하고, 이름이 같은 멤버가 둘이면 에러입니다. `Workspace`는 다른 필드를 받지 않습니다.
- 패키지 디렉터리 위의 워크스페이스 루트가 그 패키지를 나열하면 그 패키지는 멤버입니다. 루트 아래에 있어도 나열되지 않은 패키지는 따로 섭니다.
- 멤버들은 루트의 `nabi.lock`을 함께 씁니다. 멤버들의 의존성(dev 의존성 포함)을 한꺼번에 해소하므로 모든 멤버가 같은 버전으로 빌드되고, 멤버 자신의 `nabi.lock`은 읽지 않습니다. 루트의 `target/`도 함께 씁니다(`NABI_TARGET_DIR`과 `--target-dir`가 여전히 우선합니다). 멤버끼리는 `Dep::path`로 의존합니다.
- `build`, `check`, `test`, `bench`, `lint`, `fmt`, `doc`, `clean`, `run`은 `-p <name>` / `--package <name>`(되풀이 가능: 그 멤버들)와 `--workspace`(모든 멤버)를 받습니다. 패키지가 없는 루트에서는 이 명령들이 모든 멤버에서 돕니다. `run`은 정확히 하나가 필요합니다(`` `nabi run` runs one package; name it: `nabi run -p <member>` (members: ..) ``). 멤버마다 자기 디렉터리에서 `members` 순서대로 돌고, 여럿일 때는 앞에 `Member <name> (<dir>)`이 붙습니다. 하나가 실패해도 나머지는 돌고, 명령은 첫 실패의 종료 코드로 끝나며 실패한 멤버들을 밝힙니다. 멤버가 아닌 이름은 멤버 목록을 담은 에러이고, 워크스페이스 밖에서 `-p`를 쓰면 에러입니다.
- 루트에서 그 밖의 명령(`emit`, `add`, ..)은 워크스페이스 루트라는 것과 `-p <member>` 또는 멤버 디렉터리를 알려 주는 에러입니다. 루트에서 `nabi update`와 `nabi deps`는 함께 쓰는 lock으로 동작합니다. 언어 서버는 파일마다 그 패키지를 찾아, 멤버도 함께 쓰는 lock으로 해소합니다.

### `nabi.lock` 형식

`nabi.lock`은 `nabi`가 결정적으로 쓰는 NABION(정규 형식)입니다. 머리 주석 뒤에 git 패키지와 레지스트리 패키지마다 `Locked`를 하나씩 이름순으로 둡니다. path 의존은 고정하지 않으므로 나오지 않습니다. 필드 이름은 매니페스트를 따릅니다.

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

| 필드      | 뜻                                                                   |
|-----------|----------------------------------------------------------------------|
| `version` | `Lock`에서는 lock 형식 버전(`1`), `Locked`에서는 레지스트리 패키지만: 해소된 버전 |
| `name`    | 패키지 이름(`deps` 키, 이름을 바꿨으면 그 `package`)                 |
| `source`  | git은 `nabi.nabion`에 적은 `Dep::git(..)` 그대로, 레지스트리 패키지는 `Dep::registry("<index url>")`. 받아 온 미러가 아니라 index를 적음(9.10.4a) |
| `commit`  | 해소된 커밋(40자리 16진수)                                         |
| `tree`    | 그 커밋의 git tree 해시. 캐시의 checkout을 검증하는 데 씀           |

- `Locked`의 `source`는 `Dep::git(..)` 또는 `Dep::registry(..)`입니다.
- `packages` 필드가 없으면 빈 리스트로 읽습니다.
- feature는 기록하지 않습니다. lock은 optional인 것을 포함해 그래프의 모든 패키지를 고정하므로 빌드가 켜는 feature에 의존하지 않습니다.
- index의 native 패키지는 선택 필드 `natives: [LockedNative { name, version, source: Dep::registry("<index>"), sha256 }]`에 고정합니다. 이름순으로 정렬하고 비어 있지 않을 때만 씁니다. `sha256`은 `yanked: false`로 둔 항목의 정규 index 줄의 SHA-256입니다. 항목이 바뀌면 "run `nabi update`" 에러입니다.
- `nabi`는 lock 형식 1로 씁니다.
- `version`은 다른 필드보다 먼저 읽습니다. `version`이 `1`보다 크면 새 `nabi`가 쓴 lock이고, 지우라는 대신 쓸 toolchain을 알리는 에러입니다. ``nabi.lock was written by a newer nabi (lock version 2); run that nabi's toolchain here: set `toolchain` in nabi.nabion to its version, or run `nabi +<version> ..` ``.
- 그 밖에 `version`이 `1`이 아니면 "unsupported lockfile version" 에러입니다.
- lock을 읽다 난 그 밖의 모든 에러는 "(delete nabi.lock and run `nabi update` to make it again)"를 덧붙입니다.

## 9.10.4. 의존성: `nabi add` / `remove` / `update` / `deps`

```
nabi add <name> --path <dir> [--dev]
nabi add <name> --git <url> [--rev <rev> | --tag <tag> | --branch <branch>] [--dev]
nabi add <name>[@<req>] [--registry <url>] [--dev]      # index에서
nabi add <name> .. [--features <f,..>] [--no-default-features] [--optional]   # 위의 어느 것과도
nabi remove <name>
nabi add .. <file.nabi>                    # 스크립트의 //! nabion 블록에
nabi remove [--native] <name> <file.nabi>
nabi update [name...]
nabi deps [--dev] [--features <f,..>] [--all-features] [--no-default-features] [--format json]
```

- `add` / `remove`는 패키지의 `nabi.nabion`을 고치고, 주석과 배치를 보존합니다. `--dev`는 `dev_deps`에 넣습니다.
- 마지막 인자로 스크립트 파일을 주면 `add` / `remove`는 패키지의 `nabi.nabion` 대신 그 파일의 `//! nabion` 블록을 고칩니다(9.10.7). 블록이 없으면 shebang 줄과 맨 앞의 `//!` 줄 뒤에 만듭니다. 새 블록은 정규 형식으로 쓰고, 이미 있는 블록은 배치와 주석을 유지합니다. 비게 된 블록은 지웁니다. path는 스크립트 기준으로 쓰고, `--dev` / `--optional`은 에러입니다. 패키지 안의 스크립트는 에러입니다(패키지의 `nabi.nabion`을 씀).
- `add` / `remove`를 스크립트 없이 패키지 밖에서 쓰거나 `--env`와 함께 쓰면 에러입니다. 에러는 `nabi add <name> <file.nabi>`과 `nabi install` / `nabi uninstall`을 알립니다([9.10.5a](#9105a-nabi-install--nabi-uninstall)).
- 레지스트리 패키지를 요구 없이 더하면 `^<newest version>`을 적습니다. `--features`, `--no-default-features`, `--optional`은 `Dep::..`에 적습니다. 패키지에 같은 이름의 feature가 없으면 `--optional`은 feature `"<name>": ["dep:<name>"]`도 더합니다.
- 패키지에 `add`하면 `Added <name> v<version>`을 알립니다. 버전은 의존 패키지 `nabi.nabion`의 `version`입니다(아직 해석되지 않은 레지스트리 패키지는 고정한 버전). 버전이 없으면 `Added <name>`입니다. `--dev`는 ` (dev_deps)`를 덧붙입니다.
- 빌드는 `nabi.lock`의 커밋과 버전을 씁니다. 새로 선언된 의존은 lock에 더하고 지운 의존은 정리합니다. 하지만 이미 고정된 것을 옮기는 것은 `update`뿐입니다. `update`는 이름을 주면 그 패키지만, 없으면 모든 git 의존과 레지스트리 의존을 다시 해소합니다.
- `deps`는 해소한 의존 패키지를 `<code name>=<path of its src/lib.nabi>` 줄로 출력합니다. 코드 이름은 `-`를 `_`로 바꾼 것이고, 의존되는 쪽이 먼저 옵니다(`--dev`면 dev 의존 포함). 의존하는 쪽이 다른 이름(`package`)으로 부르는 패키지는 이름마다 이런 줄이 하나씩 더 있습니다. 각 패키지의 줄 뒤에는 그 `--extern-feature=<code name>=<f>` 줄이 옵니다. `deps`는 feature 옵션과 `--dev`를 받습니다. native 패키지마다(의존하는 쪽이 먼저, 호스트용) `# native <name> <version>`와 그 `nabic` 인자를 한 줄에 하나씩 출력합니다. 의존성이 없으면 아무것도 출력하지 않습니다.
- `deps --format json`은 같은 순서로 JSON 객체를 한 줄에 하나씩 대신 출력합니다. 객체마다 `schema`(지금은 `1`)와 `type`이 있습니다. `package` 레코드에는 `name`, `version`(레지스트리 패키지면 해소한 버전, 아니면 그 의존성 `nabi.nabion`의 `version`, 그것도 없으면 `null`), `source`(`path+<dir>`, `git+<url>#<commit>`, `registry+<index url>`), `path`(그 `src/lib.nabi`), `features`가 있습니다. `native` 레코드에는 `name`, `version`, `args`(그 `nabic` 인자)가 있습니다. 마지막 `summary` 레코드는 `packages`와 `natives`의 수를 세므로, 의존성이 없는 패키지는 이것만 출력합니다. 레코드는 JSON 진단의 `schema` 규칙([9.9.4](909-debug-tools.md#994-진단--에러-메시지))을 따릅니다. 기본값은 `--format human`이고, 다른 값은 CLI 에러입니다.
- `nabi.lock`은 버전 관리에 함께 넣습니다.
- `update`는 lock에서 바뀐 것마다 `Updated <name> v0.1.0 -> v0.1.1` 줄을 출력합니다. git 의존성은 짧은 커밋으로 적고, 새로 고정된 것은 `Locked`, 빠진 것은 `Removed`입니다. 바뀐 것이 없으면 `note: all dependencies are up to date`를, 의존성이 없으면 `` note: no dependencies to update; add one: `nabi add <name>` ``를 출력합니다.
- 패키지 밖에서 `--env` 없이 이 명령을 실행하면, 패키지 밖에서 쓰는 환경(9.10.5)에 라이브러리가 있을 때 그 환경을 업데이트합니다. 무언가 바뀌었으면 끝에 `` Updated environment `<name>` ``을 출력합니다. 라이브러리가 없으면 에러입니다. `` not inside a package (..): make one with `nabi init`; for an environment's libraries: `nabi update --env <name>` ``
- `nabi update`는 `nabi.lock`을 다시 해소하고, `nabi toolchain update`는 toolchain을 교체합니다(9.10.6).

### 캐시

가져온 git 소스는 `$NABI_HOME/cache/`에 두고 모든 프로젝트가 공유합니다.

| 경로                                       | 내용                                                    |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | 저장소의 bare clone(url마다 하나)                      |
| `cache/registry/db/<repo>-<hash>/`         | 레지스트리 index의 bare clone(index마다 하나)          |
| `cache/registry/src/<sha256>/`             | 그 SHA-256의 레지스트리 아카이브에서 푼 레지스트리 패키지의 파일(9.10.4a). 다시 받지 않음 |
| `cache/git/checkouts/<commit>/`            | 커밋의 checkout. 쓰기 전에 tree 해시로 검증하고, 다르면 다시 풀어 놓음 |
| `cache/scripts/<hash>/`                    | 스크립트의 `nabi.lock`과 `target/`. `<hash>`는 스크립트의 경로와, 스크립트가 닿는 모든 파일의 경로와 블록을 반영함([9.10.7](#9107-단일-파일-스크립트의-인라인-매니페스트)) |
| `cache/native/<sha256>/`                   | SHA-256이 맞은 뒤 푼 native 아카이브. 다시 받지 않음(9.10.4b) |
| `cache/native/meta/<sha256>.nabion`         | `Dep::native(url:)` 파일                                |

## 9.10.4a. 레지스트리 패키지

레지스트리에 올린 패키지를 **레지스트리 패키지**라 합니다. 기본 레지스트리는 `https://packages.nabi-lang.org`이고, index는 별도 git 저장소 `https://github.com/nabi-lang/packages-index`입니다.

### 버전 요구

Cargo와 같습니다.

| 요구                     | 뜻                                              |
|--------------------------|-------------------------------------------------|
| `1.2.3`, `^1.2.3`        | `>=1.2.3, <2.0.0`(`^0.2.3`은 `<0.3.0`, `^0.0.3`은 `<0.0.4`) |
| `~1.2.3`                 | `>=1.2.3, <1.3.0`                               |
| `=1.2.3`                 | 정확히                                          |
| `>=` `>` `<` `<=`        | 비교                                            |
| `*`, `1.*`, `1.2.x`      | 와일드카드                                      |
| `>=1.2, <1.5`            | 쉼표로 이은 조건을 모두 만족                    |

pre-release 버전(`1.3.0-beta.1`)은 요구에 같은 major.minor.patch의 pre-release를 적었을 때만 맞습니다.

### 사용하는 index

`registry:`가 없는 요구(맨 요구)는 언제나 기본 레지스트리의 index, `https://github.com/nabi-lang/packages-index`를 뜻합니다. `Dep::version`의 `registry:`와 `nabi add --registry`는 그 의존 하나의 index를 정합니다. `nabi publish`, `yank`, `login`, `logout`은 다른 index에 `--registry <url>`을 받습니다.

`$NABI_HOME/sources.nabion`은 index를 받아 올 곳을 바꿉니다(source replacement).

```nabion
Sources { replace: { "https://github.com/nabi-lang/packages-index": "https://mirror.example.com/packages-index" } }
```

- 키는 index url(끝의 `/`는 무시)이고, 값은 같은 내용을 가진 미러의 git url입니다. `nabi`는 index를 미러에서 받아 옵니다. 그 밖의 모든 것, 곧 lock(9.10.3), 토큰, 메시지는 index를 적습니다. 파일이 없으면 바꾸지 않고, `nabi`가 모르는 필드는 건너뜁니다.

### index 형식

index 저장소의 루트에는 `config.nabion`, `Config { api: "<url>" }`가 있습니다. 이 값은 레지스트리 API의 기준 URL입니다(기본 레지스트리는 `https://packages.nabi-lang.org`). URL은 `https://`여야 하고, `http://127.0.0.1`과 `http://localhost`만 예외입니다. 패키지 하나는 파일 하나이고, 파일은 공개한 소문자 이름(`-`는 그대로)의 shard 경로에 있습니다.

| 이름 길이 | 경로                     |
|-----------|--------------------------|
| 1         | `1/a`                    |
| 2         | `2/ab`                   |
| 3         | `3/a/abc`                |
| 4 이상    | `ab/cd/abcd..`           |

파일의 각 줄은 버전 하나이고, 정규 형식의 한 줄 NABION입니다. 오래된 버전이 먼저 옵니다. 줄에는 해소에 필요한 것만 매니페스트의 필드 이름으로 담습니다. 설명하는 필드는 레지스트리 API로 갑니다(아래 `meta`).

```nabion
Package { name: "json", version: "1.4.2", deps: [{ name: "util", req: "^0.3", kind: "normal" }], source: Dep::git("https://example.com/json.git", rev: "0b6d.."), tree: "77e1..", yanked: false, nabi_version: "0.1.0" }
```

| 필드        | 뜻 |
|-------------|----|
| `name`      | 게시한 패키지 이름 |
| `version`   | 그 버전 |
| `deps`      | 의존: `{ name, req, kind, registry?, optional?, features?, default_features?, package?, target? }` |
| `features`  | 선택: feature(9.10.4c) |
| `source`    | `Dep::git("<url>", rev: "<commit>")` |
| `tree`      | 그 커밋의 git tree 해시 |
| `archive`   | 선택, 레지스트리가 적음: 레지스트리가 보관하는 그 커밋의 소스 아카이브의 SHA-256(소문자 16진 64자리) |
| `yanked`    | 철회 여부 |
| `nabi_version` | 패키지를 빌드하는 가장 오래된 toolchain, `x.y.z`(9.10.3) |
| `links`     | 선택: 링크하는 native 라이브러리(9.10.3) |
| `official`  | 선택, 레지스트리가 적음: Nabi 프로젝트가 게시하는 패키지이면 `true`(아래) |

- `deps`의 `kind`는 `"normal"`, `"dev"`, `"native"` 중 하나이고, `registry`는 선택입니다. 이름을 바꾼 의존은 `name`이 코드 이름이고 `package`가 패키지 이름입니다. target별 의존은 매니페스트처럼 `target`을 담고, 클라이언트가 빌드마다 판단합니다(9.10.3).
- feature가 있는 패키지는 `deps` 뒤에 `features: { "<name>": ["<entry>", ..] }`를 두고, 9.10.4c처럼 검사합니다. `normal` 의존에는 `optional: true`, `features: [..]`, `default_features: false`를 둘 수 있습니다(기본값이 아닐 때만 적음). `dev` / `native` 의존은 optional일 수 없고, native 의존에는 feature가 없습니다. `nabi publish`가 `nabi.nabion`에서 이 값을 옮겨 적고, 해소할 때는 항목에서 읽습니다. checkout한 뒤 항목의 `features`는 그 커밋에 있는 패키지의 `nabi.nabion`과 같아야 합니다(순서 무관). 다르면 에러입니다.
- `native/<shard path>`는 native 패키지를 한 줄에 `Native { .. }` 하나씩 담습니다([9.10.4b](#9104b-네이티브-패키지)).
- `nabi_version`은 매니페스트의 것(9.10.3)입니다. `nabi publish`에 필요합니다. 없으면 `` nabi.nabion has no `nabi_version`, the oldest nabi this package builds with; the registry needs it `` 에러이고 help는 `` add `nabi_version: "0.1.3"` (this nabi), or the oldest one you have built it with ``입니다. 게시하는 `nabi`를 허용하지 않는 값은 허용하는 toolchain을 알리는 에러입니다. 레지스트리는 이 필드가 없거나 `x.y.z`가 아닌 줄을 거부합니다. 읽는 쪽은 이 필드를 따릅니다(아래 해소). 직접 둔 index에서 이 필드가 없는 줄은 어느 toolchain에나 맞습니다.
- 표준 라이브러리 계층의 파일(`3/s/std`, `co/re/core`, `ba/re/bare`)에는 `Library { name: "std", official: true }` 한 줄이 있습니다. toolchain과 함께 배포되는 라이브러리의 레지스트리 항목이고, Nabi 프로젝트가 소유하며, 버전도 아카이브도 없습니다. 해소하는 쪽은 이 줄을 건너뛰고, 그 이름으로는 어떤 버전도 게시할 수 없습니다.
- 줄 형식은 이것 하나뿐입니다. 줄마다 형식 버전을 두지 않습니다.
- 읽는 쪽은 index 줄(`Package`, 그 `deps` 항목, `Native`, `Artifact`)의 모르는 필드를 건너뜁니다. 직접 만든 `Native { .. }` 파일(`Dep::native(url:)`, `nabi publish --native`)의 모르는 필드는 여전히 거부합니다. 레지스트리는 줄을 엄격하게 검사합니다.
- 패키지는 `source` 저장소의 루트에 있습니다. `tree`는 그 커밋의 git tree 해시이고, checkout한 뒤에 검증합니다.
- 레지스트리는 버전이 게시될 때 그 커밋의 gzip tar(`git archive`, 모든 경로가 `<name>-<version>/` 아래)를 보관하고, `<api>/api/v1/packages/<name>/<version>/download`에서 내주며, 그 SHA-256을 `tree` 뒤에 `archive`로 적습니다. `nabi publish`는 `archive`를 보내지 않습니다. 소문자 16진 64자리가 아닌 값은 없는 것으로 읽습니다.
- 레지스트리 패키지는 다음 순서로 가져옵니다: 기본 index라면 toolchain이 싣고 있는 사본(아래 공식 패키지), 캐시에 이미 있는 그 커밋의 git checkout, 아카이브(`archive`가 있을 때. `cache/registry/src/<sha256>/`에서, 없으면 index의 `config.nabion`의 `api`에서 받음: `Downloading <name> <version>`), git `source`. 받은 파일의 SHA-256이 `archive`와 다르면 에러이고(`` package `num` 1.2.0: <url> has SHA-256 .., but the index says ..; the archive or the index is damaged .. ``) 빌드가 멈춥니다. 받을 아카이브가 없으면(보관하지 않음, 닿지 않음, `api` 없음, 캐시에 없는데 `--offline`) note를 내고 git 소스를 대신 가져옵니다. 아카이브가 있으면 git도 패키지의 저장소도 필요 없습니다. 어느 쪽이든 lock은 같은 커밋과 tree를 고정합니다.

### 해소

- 한 패키지에 대한 요구는 모두 셉니다. 고른 버전은 그 요구를 전부 만족합니다. 패키지는 처음 요구된 순서로 고르고, 패키지마다 lock에 고정된 버전을 먼저 해 봅니다(모든 요구를 만족하는 동안 그대로 둡니다. 그 뒤에 철회(yank)되었어도 마찬가지입니다). 그다음에는 철회되지 않은 버전을 가장 높은 것부터 해 봅니다. 어떤 선택 때문에 요구를 만족하는 버전이 하나도 없는 패키지가 생기면, 탐색은 되돌아가 그 앞에 고른 패키지들의 다음 버전을 해 봅니다. 그래서 한쪽의 `>=0.1`과 다른 쪽의 `^0.1`은 가장 높은 `0.1.x`로 풀립니다.
- 새로 고를 때(요구 없이 `nabi add`할 때도)는 `nabi_version`이 실행 중인 `nabi`를 허용하지 않거나([9.10.6 호환성](#호환성)) `nabi`가 읽을 수 없는 항목을 건너뜁니다. 그래서 가장 높은 버전이 가려지면 note가 그 버전, 필요한 toolchain(`` package `num` 2.0.0 needs nabi 0.2.1 or a later 0.2.x (this is nabi 0.1.5) ``), 쓰는 방법(`nabi +0.2.1 ..`, 또는 그 toolchain으로 고정), 대신 고른 버전을 알려 줍니다. 그 밖에 맞는 버전이 없으면 에러가 그 note를 help로 담습니다. lock에 고정된 버전은 그대로 둡니다.
- 기본 index에서는 toolchain이 싣고 있는 버전(아래 공식 패키지)을 lock에 고정된 버전 다음, 다른 버전보다 먼저 해 봅니다. `^0.2`는 index의 `0.2.5`보다 싣고 있는 `0.2.0`을 고릅니다. 그 줄은 index가 없어도 되므로, 탐색의 모든 패키지를 싣고 있고 캐시된 index가 없으면 index를 받지 않습니다. 싣고 있는 버전으로 채워지지 않는 요구가 남으면 index를 받고 탐색을 다시 합니다. `nabi update`는 싣고 있든 아니든 가장 높은 버전부터 해 봅니다.
- 한 의존 그래프에서 이름 하나는 버전 하나입니다. 전이 의존은 index 항목의 `deps`를 따릅니다. path 패키지와 git 패키지는 고정이고, 그 레지스트리 요구는 루트의 요구처럼 셉니다.
- 맞는 버전 조합이 없으면 에러는 어떤 버전도 요구를 다 만족하지 못하는 패키지를 밝히고, 요구마다 누가 요구했는지(`` `^2` (required by `a` 1.4.0) ``), index에 있는 버전, 요구한 패키지가 빌드에 들어온 경로(`` `a` 1.4.0: the root package requires `a` `^1.0` ``), 그 패키지들의 다른 버전도 해 봤는지를 보여 줍니다. help는 무엇을 적을지 알려 줍니다: 나머지 요구를 만족하는 그 패키지의 버전(`` change the root package's to it, `"b": "^2.3.0"` ``), 또는 나머지 요구와 맞는, 요구한 패키지의 버전(`` change the root package's requirement on it, `"c": "^0.4.0"` ``), 그것도 없으면 요구 하나를 느슨하게 하라는 말입니다. 어떤 버전도 만족하지 못하는 요구 하나는 평범한 "no version of package .. matches .." 에러입니다. 탐색은 선택 100000번 뒤에 멈추고, 주요 의존성을 고정하라고 알려 줍니다.
- lock은 index와 대조합니다. 고정된 버전의 `rev`가 바뀌었으면 "run `nabi update`" 에러입니다.
- `--offline`은 캐시된 index를 쓰고, `--locked`를 지킵니다. 어느 캐시에도 소스가 없는 버전(git checkout도 아카이브도 없음)은 에러입니다(`` package `json` 0.2.5 is not in the cache, and --offline is set: run the command once without --offline; `nabi component add packages` (or `update`) keeps the official packages for offline use ``). 뒷부분은 공식 패키지일 때만 붙습니다.

### 공식 패키지

Nabi 프로젝트가 게시하는 패키지는 index에 `official: true`로 표시됩니다(레지스트리가 적습니다). 코드는 이런 패키지를 먼저 추가하지 않고 쓸 수 있습니다. 패키지나 스크립트에서 `build`, `run`, `check`, `test`, `bench`, `repl`, `jupyter`가 경로의 첫 이름으로 쓰인 이름(`use json::parse;`, `json::parse(s)`)을 찾았는데 그것이 그 패키지나 스크립트의 의존성이 아니고(빌드의 다른 패키지가 의존하는 것은 의존성이 아님) 코드 스스로 선언한 것(모듈, 항목, import, 바인딩)도 아니며, 기본 index가 그 이름을 공식 패키지라고 하면(yank되지 않은 가장 높은 버전, 즉 최신 버전의 줄로 판단합니다. 이름은 `_`와 `-`를 같게 보고 찾으므로 `nabi_syntax::..`는 `nabi-syntax`이고, index의 이름으로 추가됩니다) `nabi`는 다음과 같이 합니다.

- toolchain이 싣고 있는 버전(아래)을, 없으면 이 toolchain이 빌드하는(`nabi_version`이 이 toolchain을 허용하는), 철회되지 않은 가장 높은 버전을 골라 `"<name>": "^<major>.<minor>"`(0.1 아래는 `^<version>`)를 패키지의 `nabi.nabion`(워크스페이스 멤버는 그 멤버의 것, 스크립트는 `//! nabion` 블록. 없으면 만듦)에 적고, 해소해 `nabi.lock`을 고친 뒤 `Added <name> ^x.y (official)`를 출력하고 명령을 이어 갑니다.
- toolchain이 싣고 있는 패키지라면 index를 읽지 않습니다. 아니면 캐시된 index를 읽고, 캐시가 없을 때만 받습니다(받지 못하거나, 캐시가 없는데 `--offline`이면 아무것도 추가하지 않습니다).
- `--locked`나 `--frozen`이거나(에러에 그 옵션 이름이 나옵니다) `CI`가 비어 있지 않은 값으로 설정되어 있으면 매니페스트를 고치지 않습니다: `` `json` is an official package the code uses, but it is not in nabi.nabion, and --locked never changes it; add it: `nabi add json` ``.
- 오프라인인데 패키지가 어느 캐시에도 없으면: `` .. it is not in the cache and --offline is set: run `nabi add json` online, or keep the official packages for offline use with `nabi component add packages` ``.

`std`, `core`, `bare`, `nabi`, `pkg`, `sup`, `crate`, `super`, `self`와 대문자로 시작하는 이름은 패키지가 아닙니다. 공식 패키지가 아닌 이름은 컴파일러의 에러를 그대로 받습니다(`` `geom` is not a dependency of this package: add it to nabi.nabion (`nabi add geom`) ``). REPL과 Jupyter 커널도 셀마다 같게 합니다(패키지의 세션에서는 `nabi add`, 환경의 세션에서는 `nabi install --env`). 언어 서버는 매니페스트를 고치지 않고 `nabi add <name>` 빠른 수정을 제안합니다(9.9.13).

### `nabi component`

```
nabi component list
nabi component add packages [--bundle <dir>]
nabi component update [packages]
nabi component remove packages
```

`packages`는 기본 index에 있는 모든 공식 패키지의, 이 toolchain이 빌드하는(`nabi_version`이 이 toolchain을 허용하는) 철회되지 않은 가장 높은 버전의 레지스트리 아카이브(9.10.4a index 형식)입니다.

- 릴리스 toolchain은 이것을 싣고 있습니다(9.10.6 배포). `bin/` 옆의 `lib/nabi/packages/`에는 그 버전들의 index 줄을 담은 `index`(index 파일처럼 한 줄에 하나)와 아카이브마다 푼 `src/<sha256>/`이 있습니다. `remove` 말고는 아무것도 여기에 쓰지 않습니다. 해소는 기본 index에서 이것을 먼저 읽고(위 해소), 그 줄은 index와 같은 커밋과 tree를 가리키므로 lock은 받아서 만든 것과 같습니다.
- `add`는 index를 받고 아카이브마다 `cache/registry/src/<sha256>/`에 받아 줄의 `archive`와 대조하고(`Downloading json 0.2.0` .., 그다음 `Added component packages (<n> official packages)`), `$NABI_HOME/components.nabion`에 기록합니다. 이미 추가했거나 toolchain이 싣고 있으면 `add`는 `update`를 알려 주는 note입니다. `--bundle <dir>`은 대신 toolchain의 `lib/nabi/packages`로 `<dir>`에 씁니다(있던 것은 바꿉니다). 릴리스 빌드가 이렇게 합니다(`Bundled <n> official packages into <dir>`).
- `update`는 index를 다시 받고 이 toolchain이 빌드하는 가장 새 버전을 캐시에 받습니다. toolchain이 싣고 있는 것은 그대로 둡니다.
- `remove`는 toolchain이 싣고 있는 것, 기록한 아카이브, 기록을 지웁니다. 둘 다 없으면 에러입니다.
- `list`는 구성 요소마다 추가했는지 보여 줍니다: `added: <n> package(s) with the toolchain`, `add` / `update`가 받은 것도 있으면 `, <m> downloaded`가 붙습니다.

`archive`가 없는 버전은 note를 내고, 쓸 때 git에서 가져옵니다. `add`와 `update`에 `--offline`을 주면 에러이고, 그 밖의 이름은 구성 요소 목록을 담은 에러입니다.

구성 요소는 캐시만 채웁니다. 버전은 여전히 매니페스트와 lock이 고릅니다. toolchain에도 캐시에도 없는 버전을 lock이 고정하면 온라인에서는 그 버전을 받고, 오프라인에서는 위의 에러입니다.

### `nabi search` / `publish` / `yank` / `login` / `logout`

```
nabi search [--native] <query>
nabi login [--registry <url>] [<token>]
nabi logout [--registry <url> | --all]
nabi publish [--dry-run] [--registry <url> | --index-dir <checkout>]
nabi yank [--native] <name>@<version> [--undo] [--message <why>] [--registry <url> | --index-dir <checkout>]
```

- `search`는 index에서 이름으로 찾고, 찾은 것마다 `<name> <newest version> - <description>`을 한 줄씩 출력합니다. description은 레지스트리 API(`GET <api>/api/v1/packages/<name>`, 가장 새 버전의 `meta.description`)에서 얻습니다. API에 없거나 API가 답하지 않으면, 그리고 `--offline`이면 그 부분은 생략합니다. index 조회는 `-` / `_` 어느 쪽 철자든 받습니다.
- `publish`는 `cargo publish`처럼 게시합니다. `publish --dry-run`은 dry run입니다.
  - `publish: false`인 패키지는 거부합니다. 에러는 지울 줄을 알려 줍니다.
  - readme(9.10.3)가 있어야 합니다. 깨끗한 작업 트리, 저장소 루트의 패키지, 그리고 `src/lib.nabi`, `src/main.nabi`, 또는 둘 다가 필요합니다.
  - `src/main.nabi`만 있는 패키지는 프로그램입니다. `nabi install`이 그것을 빌드하고, 의존 패키지가 될 수 없습니다.
  - path의 normal 의존은 `version:` 요구사항으로 게시하고, `version:`이 없으면 거부합니다(`` dependency `linalg` is a path: .. add the version it needs, published from that path: `Dep::path("../linalg", version: "^0.1")` ``). git의 normal 의존은 거부하고, `version:`이 없는 path / git의 dev 의존은 뺍니다.
  - 같은 버전이 있는지, 그리고 `-` / `_`와 대소문자를 같게 보았을 때 이름이 충돌하는지 검사합니다. 다른 철자가 이미 차지한 이름은 거부합니다. 9.10.3의 규칙을 어기는 이름(Windows 장치 이름, 64자 초과)도 거부합니다.
  - `--index-dir` 없는 dry run은 항목만 출력합니다. `--index-dir`가 있으면 그 checkout에 대해서도 검사합니다.
  - 항목의 source는 설정된 그대로(`insteadOf` 재작성 전)의 git remote `origin` url입니다. 이 url은 `https://`여야 하고, `http://127.0.0.1`과 `http://localhost`만 예외입니다.
- `--dry-run`이 없으면 `publish`는 항목을 더합니다. `--index-dir`가 있으면 그 checkout에 "Publish <name> <version>" 커밋으로 더합니다(push하지 않음). 없으면 `--registry <url>`이 가리키는 index, 그것도 없으면 기본 레지스트리("사용하는 index")의 레지스트리 API로 보내고, API가 항목을 index에 넣습니다. `--index-dir`와 `--registry`는 함께 쓸 수 없습니다. 클라이언트는 index 저장소에 직접 쓰지 않습니다.
- `yank`는 그 버전의 `yanked`를 켭니다(`--undo`면 끕니다). `--index-dir`가 있으면 그 checkout의 커밋으로, 없으면 `--registry <url>`이 가리키는 index, 그것도 없으면 기본 레지스트리의 API로 처리합니다.
- `--commit`(`publish`), `--index`(`publish`와 `yank`), `--vers`(`yank`)는 에러이고, 대신 쓸 형식을 알립니다. 각각 옵션 없는 `publish`, `--index-dir <dir>`, `yank <name>@<version>`입니다.
- `yank --message <why>`는 철회 이유를 적습니다(많아야 512바이트). 레지스트리가 보관하고 보여 줍니다. 철회할 때만 씁니다. `--undo`나 `--index-dir`(checkout에는 둘 곳이 없음)와 함께 쓰면 에러입니다.
- 레지스트리 API의 주소는 index의 `config.nabion`의 `api`에 있습니다. 요청은 `Authorization: Bearer <token>`을 싣고, 본문은 한 줄 NABION(`application/nabion; charset=utf-8`)입니다.
  - `GET <api>/api/v1/me` → `Me { login: "..", publish: <bool> }`
  - `POST <api>/api/v1/publish`, 본문 `Publish { entry: <index line>, meta: { .. } }`(줄은 `Package { .. }` / `Native { .. }`) → `Published { name, version, file, warnings: ["..", ..] }`. `nabi`는 경고를 하나씩 출력합니다.
  - `POST <api>/api/v1/yank`, 본문 `Yank { name, version, undo: <bool>, native: <bool>, message: ".." }`(`message`는 있을 때만 적음) → `Yanked { name, version, yanked }`
  - `GET <api>/api/v1/packages/<name>`과 `/api/v1/native/<name>`, 토큰 없음 → `Package { name, native, owners, versions: [Version { version, yanked, meta, yank_message, has_readme, downloads, published_by, yanked_by, unyanked_by }, ..] }`. `GET <api>/api/v1/packages/<name>/<version>/readme` → markdown README.

  실패는 4xx / 5xx 상태와 `Error { message: ".." }`로 옵니다. `nabi`는 그 message를 보고합니다. 401은 토큰이 유효하지 않다는 뜻입니다. `nabi`는 응답에서 모르는 필드를 건너뜁니다.
- `meta`는 패키지를 설명하고 해소에는 쓰지 않는 것을 매니페스트(9.10.3)와 패키지에서 모아 담습니다. 값이 없는 필드는 뺍니다.

  | 필드 | 출처 |
  |------|------|
  | `description`, `license`, `license_file`, `authors`, `homepage`, `documentation`, `repository`, `readme`(경로), `keywords`, `categories`, `include`, `exclude`, `publish`, `metadata`, `links`, `default_run` | 같은 이름의 매니페스트 필드 |
  | `has_lib` | 패키지에 `src/lib.nabi`가 있는지 |
  | `bin_names` | 실행 파일: `src/main.nabi`가 있으면 `[<name>]`, 없으면 `[]` |
  | `readme_content` | readme(9.10.3)의 내용, 많아야 524288바이트. 더 길면 `nabi publish`에서 에러 |
  | `size` | 묶음 크기: 패키지가 담는 HEAD의 파일 바이트 합(`include` / `exclude`, 9.10.3) |

  `nabi`가 모르는 매니페스트 필드는 적은 그대로 `meta`에 담아 보내므로, 새 레지스트리가 저장할 수 있습니다. `publish --native`의 `meta`는 그 파일의 `description`, `license`, `repository`입니다.
- 레지스트리 API에는 토큰이 필요하고, `--index-dir` checkout에는 필요 없습니다. 기본 레지스트리의 토큰은 `NABI_REGISTRY_TOKEN`이 설정되어 있고 비어 있지 않으면 그 값, 아니면 `nabi login`이 저장한 토큰입니다. 다른 index의 토큰은 `nabi login --registry <url>`이 저장한 것입니다. `NABI_REGISTRY_TOKEN`은 거기로 보내지 않습니다(설정되어 있으면 경고를 한 번 냄: ``NABI_REGISTRY_TOKEN goes only to https://github.com/nabi-lang/packages-index; for <url> run `nabi login --registry <url>` ``). 토큰이 없으면 "run `nabi login` first" 에러이고, 다른 index에서는 ``run `nabi login --registry <url>` first``입니다.
- `nabi login`은 웹사이트에서 만든 레지스트리 계정 토큰을 받습니다(`<api>/me`, GitHub로 로그인. 지금은 레지스트리 maintainer만 만들 수 있음). 토큰을 생략하면 표준 입력에서 화면에 보이지 않게 읽습니다(Ctrl-C에도 에코를 되돌림). `login`은 `/api/v1/me`에 물어봅니다. API가 거부하는 토큰과 `publish`가 false인 계정("this account cannot publish yet: publishing is open to the registry's maintainers only")은 아무것도 저장하지 않고 거절합니다. 그 밖에는 `--registry <url>`이 가리키는 index, 없으면 기본 레지스트리의 토큰을 `$NABI_HOME/tokens.nabion`에 `Tokens { tokens: { "<index url>": "<token>" } }`로 저장하고 "Logged in as <login>"을 출력합니다. 키에는 끝의 `/`를 붙이지 않고, 이미 있던 필드와 다른 index의 토큰은 그대로 둡니다. 토큰은 자기 index의 레지스트리 API로만 보냅니다.
- 토큰 파일은 `tokens.nabion` 하나뿐입니다.
- `tokens.nabion`은 Unix에서 mode 0600으로 씁니다. Windows에서는 사용자 프로필 아래에 있는 `$NABI_HOME`의 기본 권한을 받습니다.
- `nabi logout`은 `tokens.nabion`에서 기본 레지스트리의 토큰을 지웁니다. `--registry <url>`이면 그 index의 항목을, `--all`이면 모든 토큰을 지웁니다.
- 토큰은 `curl`의 표준 입력으로만 전달합니다. 명령줄이나 환경에 넣지 않고 출력하지도 않습니다(에러에서는 `***`로 보임).

## 9.10.4b. 네이티브 패키지

native 패키지는 타깃 트리플마다 미리 빌드한 아카이브와 다음과 같은 값으로 이루어집니다.

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

이 값은 index 한 줄이거나 `Dep::native(url:, sha256:)`의 파일입니다. 파일 안의 artifact url은 그 파일 기준 상대 경로여도 됩니다. `nabi publish --native`가 읽는 파일에는 `description`, `license`, `repository`도 둘 수 있습니다. 이 필드는 index 줄이 아니라 레지스트리 API의 `meta`로 갑니다.

- 아카이브(tar로 읽을 수 있는 어떤 형식이든)에는 `lib/`가 들어갑니다. 그 안에는 `lib<n>.a`, `lib<n>.dylib` / `lib<n>.so`, Windows `<n>.lib`(static 또는 import library)가 있습니다. `runtime` 파일과 라이선스 파일도 들어갑니다. 그 안의 어떤 것도 실행하지 않습니다.
- 해시는 필수입니다. 아카이브는 풀기 전에 검사합니다. 절대 경로나 `..` 경로, 밖으로 나가는 링크가 있거나, 이름 붙은 라이브러리나 runtime 파일이 없으면 거부합니다.
- Apple 타깃에서 runtime dylib에는 `@rpath/` install name이 있어야 합니다.
- 빌드는 타깃 트리플의 artifact를 씁니다. 없으면 에러이고, 에러는 artifact가 있는 트리플을 나열합니다.
- `nabi`는 `nabic`에 `-L <archive>/lib`, link마다 `-l kind=name`, runtime 파일마다 `--native-runtime`을 넘깁니다. 의존하는 쪽이 의존되는 쪽보다 앞에 옵니다.

```
nabi add --native <name>[@<req>]
nabi remove --native <name>
nabi search --native <query>
nabi publish --native <file> [--dry-run] [--registry <url> | --index-dir <dir>]
nabi yank --native <name>@<version> [--undo] [--registry <url> | --index-dir <dir>]
```

`publish --native`는 모든 타깃을 받아 검사합니다. 절대 url, 라이선스 파일, index에서 온 의존성만 허용합니다. 게시와 `yank --native`는 레지스트리 패키지와 같은 규칙을 따릅니다. `--index-dir` checkout을 쓰고, 없으면 토큰으로 레지스트리 API를 씁니다(`Yank`의 `native: true`).

- ELF 타깃에서 runtime 라이브러리의 soname은 파일 이름과 같아야 합니다. `-musl` 타깃의 `dylib` link는 에러입니다.
- `repl` / `jupyter`는 native 패키지를 로드합니다. 공유 라이브러리는 시스템 로더로 로드하고, static 아카이브는 세션에 링크합니다. `check` / `lint` / `doc`은 아무것도 링크하지 않습니다.
- 실행 파일 옆의 사본은 빌드 산출물입니다.

### 시스템 라이브러리와 프레임워크

```nabion
native: {
    "z": Dep::system(pkg_config: "zlib", brew: "zlib", vcpkg: "zlib", apt: "zlib1g-dev"),
    "png": Dep::system(pkg_config: "libpng16", libs: ["png16"], static: true),
    "Metal": Dep::framework(),
}
```

`Dep::system`은 기계에 설치된 C 라이브러리를 가리킵니다. 인자는 모두 선택이고 이름으로 씁니다. `pkg_config`(`.pc` 모듈, 기본값은 키), `brew`(Homebrew formula), `vcpkg`(vcpkg 포트), `apt`(Linux 사용자에게 에러가 설치하라고 알려 줄 패키지), `libs`(`.pc` 파일이 없을 때 링크할 것, 기본값은 키), `static: true`(정적 아카이브를 `.pc` 파일의 private 부분과 함께 링크)입니다. 빌드마다 어떤 프로그램도 실행하지 않고 다음 순서로 찾습니다.

1. pkg-config `.pc` 파일을 직접 읽습니다. `PKG_CONFIG_PATH`, 그다음 `PKG_CONFIG_LIBDIR`가 있으면 그것, 없으면 타깃의 multiarch 디렉터리(`/usr/lib/x86_64-linux-gnu/pkgconfig`, ..), `/usr/local/lib/pkgconfig`, `/usr/local/share/pkgconfig`, `/usr/lib64/pkgconfig`, `/usr/lib/pkgconfig`, `/usr/share/pkgconfig`, macOS에서는 Homebrew의 `lib/pkgconfig`입니다. 변수(`${prefix}`, `${pcfiledir}`)와 `Requires`(`static`이면 `Requires.private`과 `Libs.private`도)를 따라가고, `PKG_CONFIG_SYSROOT_DIR`는 그 경로들 앞에 붙습니다.
2. macOS에서(또는 `HOMEBREW_PREFIX`가 있으면) Homebrew: `$HOMEBREW_PREFIX/opt/<formula>`(기본값 `/opt/homebrew`)의 `.pc` 파일, 없으면 그 `lib/`와 `include/`.
3. `VCPKG_ROOT`(또는 `VCPKG_INSTALLATION_ROOT`)가 있으면 vcpkg: `installed/<triplet>`. triplet은 `VCPKG_DEFAULT_TRIPLET`가 없으면 타깃에서 정합니다(`x64-windows`, `arm64-windows`, `static`이면 `x64-windows-static`, `x64-linux`, `arm64-osx`, ..). Windows에서는 그 `bin/`의 DLL을 실행 파일 옆에 둡니다.
4. macOS SDK 자체의 라이브러리(`usr/lib/lib<name>.tbd`: `z`, `sqlite3`, ..).

크로스 빌드는 타깃의 파일이 있는 곳만 찾습니다. `PKG_CONFIG_PATH`, `PKG_CONFIG_LIBDIR`, 타깃의 multiarch 디렉터리, `PKG_CONFIG_SYSROOT_DIR`가 있으면 그 sysroot의 기본 디렉터리이고, 호스트의 Homebrew는 보지 않습니다. `nabi`는 찾은 것을 `-L`과 `-l`로 넘기고, 헤더 디렉터리는 `c` 소스에 씁니다. 찾지 못하면 어디를 찾았는지와 호스트의 설치 명령을 알려 주는 에러입니다: `` system library `png` (pkg-config `libpng16`) not found for x86_64-unknown-linux-gnu (looked in pkg-config (libpng16.pc)): install it with `sudo apt install libpng-dev` (or your distribution's package of it), or set PKG_CONFIG_PATH to the directory holding libpng16.pc ``(macOS에서는 `brew install`, Windows에서는 `vcpkg install <port>:<triplet>`).

`Dep::framework()`는 인자가 없습니다. 키가 프레임워크 이름이고(`Foundation`, Apple의 표기 그대로), `#[link(name = "Foundation", kind = "framework")]`처럼 링크합니다([5.9.9](../005-modules/509-extern-block.md#599-linklibname-라이브러리-명시)). Apple이 아닌 타깃의 빌드에서는 빠집니다.

그래프에 있는 모든 패키지의 시스템 라이브러리를 이름마다 하나씩 링크합니다. 레지스트리 패키지의 것은 그 `nabi.nabion`에서 가져옵니다(index 줄에는 없음). `nabi.lock`에는 들어가지 않습니다. 기계에 있는 것이기 때문입니다.

## 9.10.4c. feature

패키지는 `features`에 feature를 선언합니다. 이 dict는 feature 이름마다 그 feature가 켜는 항목의 리스트를 적습니다. 이름은 `[a-z0-9][a-z0-9_-]*`이고, 무엇을 켜는지 말해 주는 단어로 짓습니다(`http1`, `http2`, `tls`). 각 항목은 다음 중 하나입니다.

| 항목        | 켜는 것                                                              |
|-------------|----------------------------------------------------------------------|
| `<f>`       | 이 패키지의 feature `f`                                              |
| `dep:<d>`   | optional 의존 `d`                                                    |
| `<d>/<f>`   | 의존 `d`의 feature `f`, 그리고 `d`가 optional이면 `d` 자체        |
| `<d>?/<f>`  | optional 의존 `d`의 feature `f`. 다른 무언가가 `d`를 켤 때만       |

- `default`는 끄지 않는 한 켜져 있는 보통의 feature입니다. 빌드하는 패키지에서는 `--no-default-features`로, 의존에서는 `default_features: false`로 끕니다.
- optional 의존(`optional: true`)은 켜진 feature의 `dep:<d>`나 `<d>/<f>` 항목이 켤 때만 빌드에 들어갑니다. 그 이름의 암묵 feature는 없습니다. 켜는 항목이 하나도 없으면 에러입니다.
- 다음은 에러입니다. 항목이 패키지에 없는 feature나 (`deps`의) 의존을 가리킬 때, `dep:`나 `?`가 optional이 아닌 의존을 가리킬 때, feature가 순환할 때입니다. 진단은 `file:line:col`을 보여 줍니다. 의존 패키지에 없는 feature를 요구해도 에러입니다.
- 해소: lock은 그래프 전체를 덮습니다. 빌드는 루트 패키지의 feature에서 시작합니다. 끄지 않았으면 `default`, `--features`로 준 것, `--all-features`이면 모든 feature입니다. 켜진 의존마다 그 의존에 요구한 feature와 그 의존의 `default`를 가지고 따라갑니다. feature는 더해지기만 하고 합쳐집니다. 그래서 그래프 전체에서 패키지마다 feature 집합은 하나입니다. 루트의 `dev_deps`와 그 의존이 요구하는 feature는 `test` / `bench`에서만 셉니다. 결과는 어떤 것의 순서에도 의존하지 않습니다.
- `--features`는 루트의 feature 이름과, 직접 의존의 `<dep>/<f>`를 받습니다.
- 각 패키지는 켜진 feature에 대해서만 `cfg(feature = "<f>")`가 참인 상태로 컴파일됩니다([6.3.6](../006-meta-system/603-cfg.md#636-cfg-값의-출처)). native 패키지에는 feature가 없습니다.
- 스크립트의 ```` ```nabion ```` 블록에는 `features`가 없습니다(9.10.7).

## 9.10.5. 환경

환경은 conda / venv를 대신합니다. 활성화 단계는 없습니다.

- 패키지(`src/main.nabi`, `src/lib.nabi`, `nabi.nabion` 중 하나가 있는 디렉터리)는 자기 `nabi.nabion`의 의존성만 씁니다. 환경에 설치한 라이브러리는 보지 않습니다. 스크립트도 마찬가지입니다. 패키지에 작용하는 명령을 `--env`와 함께 쓰면 에러입니다.
- 이름 있는 환경은 `$NABI_HOME/envs/<name>/`에 `nabi.nabion`, `nabi.lock`, `target/`을 둡니다. 그 `deps`는 REPL과 Jupyter를 위해 `nabi install`([9.10.5a](#9105a-nabi-install--nabi-uninstall))로 설치한 라이브러리입니다. 이 목록을 바꾸는 것은 `nabi install` / `nabi uninstall`과 `nabi env import`뿐입니다. 그 안의 path 의존은 절대 경로입니다. 설치한 라이브러리는 그 `target/`에 미리 빌드합니다.
- 패키지 밖(REPL, Jupyter, 언어 서버, `nabi install`)에서 쓸 환경은 `--env <name>` > `NABI_ENV` > `nabi env use`(`$NABI_HOME/envs/.current`) > `default` 순으로 정합니다. `default` 환경은 그 환경으로 처음 `nabi install`을 실행할 때 만들어집니다. 패키지 안에서 경로 없는 `nabi repl` / `jupyter` / `lsp`는 그 패키지를 쓰고, `--env`가 있으면 그 환경을 씁니다.
- 환경 이름은 ASCII 글자, 숫자, `_`, `-`, `.`로 쓰고, `.`로 시작할 수 없으며, 64자를 넘을 수 없고, Windows 장치 이름(`con`, `nul.txt`, ..)일 수 없습니다. `nabi env create`는 다른 환경과 대소문자만 다른 이름(`foo`가 있을 때 `Foo`)을 거부합니다. Windows와 macOS는 둘을 한 디렉터리로 보기 때문입니다.
- 이름이 오는 모든 곳(`--env`, `NABI_ENV`, `nabi env use`의 파일, `nabi env`의 인자)을 검사합니다. 구분자(`/`, `\`, `:`)가 들어 있거나 `.`, `..`인 이름은 어디서든 에러입니다(`` `../p` is a path, not an environment name; use a name from `nabi env list`, e.g. `default` ``). `nabi env create`가 거부할 다른 이름도 에러입니다. 다만 그 환경이 이미 있으면(디렉터리를 손으로 만든 것) 경고만 하고 명령을 계속합니다. `nabi jupyter install --env <name>`은 그 환경이 있어야 합니다(`default`는 예외).
- `nabi env use`는 이름을 `$NABI_HOME/envs/.current`에 씁니다. 첫 줄만 읽습니다. 고른 환경을 `nabi env remove`로 지우면 이 파일도 지우므로 고른 환경이 없게 됩니다.

### 의존성의 출처

| 컴파일 대상 | 의존성 | Lock | 미리 빌드하는 곳 |
|---|---|---|---|
| 패키지 | 그 `nabi.nabion`의 `deps`(test / bench는 `dev_deps`) | `<package>/nabi.lock` | `<package>/target/<profile>/deps/` |
| 스크립트 | 스크립트와 그 `mod`가 읽는 파일의 `//! nabion` 블록을 합친 `deps` | `cache/scripts/<hash>/nabi.lock` | `cache/scripts/<hash>/target/` |
| 패키지 밖의 REPL / Jupyter | 활성 환경에 설치한 라이브러리 | 환경의 것 | 환경의 `target/` |

```
nabi env create <name>
nabi env list                      # 현재 환경에 * 와 (active) 표시, 없으면 만드는 법을 알려 줌
nabi env remove <name>
nabi env use <name>                # 프로젝트 밖의 기본 환경을 바꿈
nabi env export <name> <dir>       # nabi.nabion / nabi.lock 을 <dir> 에 씀
nabi env import <name> <dir | nabi.nabion>    # 프로젝트나 export 에서 만듦 (상대 path 는 절대로)
nabi env shell [name]              # NABI_ENV=<name> 인 $SHELL
nabi repl --env <name>
```

REPL의 `:add <dep> ..`는 현재 환경에서는 `nabi install <dep> ..`을, 패키지의 세션에서는 `nabi add`를 실행합니다. 그 뒤 의존성을 다시 읽어 세션에 붙입니다.

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

- 소스의 패키지는 먼저 캐시로 가져옵니다([9.10.4](#9104-의존성-nabi-add--remove--update--deps)).
- `src/main.nabi`가 있는 패키지나 스크립트 파일은 실행 파일입니다. `cargo install`처럼 `--release`로 빌드해 그 실행 파일을 `$NABI_HOME/bin/<name>`에 둡니다. git 패키지와 레지스트리 패키지는 checkout의 임시 사본에서, path 패키지와 스크립트 파일은 제자리에서 빌드합니다. `$NABI_HOME/installs.nabion`(`Package { deps: { "<name>": <source> } }`)이 각 실행 파일과 그 정확한 출처를 기록합니다. 레지스트리 패키지는 `=<version>`으로, git 패키지는 그 `rev:`와 함께 기록합니다. 패키지에 `src/lib.nabi`도 있으면 `nabi install`에는 `--bin`이나 `--lib`가 있어야 합니다. 둘 다 없으면 에러이고 아무것도 설치하지 않습니다. 에러는 실행 파일을 원하면 `--bin`을, 라이브러리를 원하면 `--lib`를 쓰라고 알립니다. `src/main.nabi`가 없는 패키지에 `--bin`을 주거나 `--bin`과 `--lib`를 함께 주면 에러입니다.
- `src/lib.nabi`만 있는 패키지, 또는 `--lib`를 준 `src/lib.nabi`가 있는 패키지는 라이브러리입니다. 라이브러리는 `pip install`처럼 `nabi repl`, `nabi jupyter`, `:add`를 위해 활성 환경([9.10.5](#9105-환경))에 들어갑니다. 환경의 `nabi.nabion`에 적고(요구 없는 레지스트리 패키지는 `^<newest version>`, path는 절대 경로), 그 `nabi.lock`으로 해소하고, 그 `target/`에 미리 빌드합니다. 스크립트는 이 라이브러리를 보지 않습니다. 스크립트는 `nabi add <name> <file.nabi>`으로 라이브러리를 얻고, 출력 줄이 이를 알립니다. 빌드되지 않으면 아무것도 설치하지 않습니다. `src/lib.nabi`가 없는 패키지나 스크립트에 `--lib`를 주면 에러입니다.
- `<name>`은 라이브러리의 최상위 이름, 또는 실행 파일의 파일 이름입니다. 생략하면 패키지의 `name`을 씁니다. 그것도 없으면 디렉터리 이름(path), `.git`을 뗀 저장소 이름(git), 파일 stem(스크립트)을 씁니다. [9.10.3](#9103-nabinabion--nabilock)의 이름 규칙을 따릅니다.
- `--features` / `--no-default-features`는 라이브러리의 `Dep`에, 또는 실행 파일의 빌드에 들어갑니다. `--dev` / `--optional`은 에러입니다. `--native`는 환경의 `native`에 native 패키지를 더합니다([9.10.4b](#9104b-네이티브-패키지)).
- 프록시 `nabi`, `nabic`, `nabicc`, `nabils`([9.10.6](#9106-toolchain))는 절대 교체하지 않습니다. `nabi install`이 두지 않은 `$NABI_HOME/bin`의 파일도 마찬가지입니다(에러). 실행 파일을 다시 설치할 때는 다음과 같습니다.
  - 레지스트리의 같은 버전이나 같은 git 커밋에 feature도 같으면 다시 빌드하지 않고 유지합니다(`` `tool` 0.1.0 is installed already (<path>); to rebuild it, run `nabi install tool --force` ``). `--force`를 주면 다시 빌드합니다(`Reinstalled`).
  - 버전이 다르면 교체합니다(`Upgraded tool from 0.1.0 to 0.2.0`, `Downgraded ...`). git 커밋이 달라도 마찬가지입니다(`Replaced <url> at <commit> with ...`).
  - path나 스크립트 파일에서 온 것은 `cargo install --path`처럼 언제나 다시 빌드합니다.
  - 라이브러리 설치는 그대로입니다.
- `--list`는 ``libraries (environment `<env>`):``와 `executables (<bin dir>):`를 출력합니다. 각각 뒤에 설치 순서대로 `  <name> <source>` 줄이 옵니다.
- `nabi uninstall <name>..`은 각 이름을 활성 환경의 라이브러리에서 지웁니다(lock을 다시 해소). 라이브러리가 아니면 `nabi install`이 `$NABI_HOME/bin`에 둔 실행 파일을 지웁니다. `--native`는 환경에서 native 패키지를 지웁니다. 둘 다 아닌 이름은 에러입니다.
- `nabi install` 말고 `$NABI_HOME` 아래에 쓰는 것은 `nabi toolchain`(toolchain, 프록시, `config.nabion`), `nabi login`(`tokens.nabion`), `nabi env`(환경), 그리고 캐시뿐입니다.

## 9.10.6. toolchain

`$NABI_HOME/bin/`의 `nabi` / `nabic` / `nabicc` / `nabils`는 활성 toolchain의 같은 실행 파일을 실행하는 프록시입니다(rustup 방식). 활성 toolchain은 다음 순서로 정합니다.

1. 명령줄의 `+<version>`(`nabi +0.4 build`)
2. 환경 변수 `NABI_TOOLCHAIN`
3. 실행하는 스크립트 블록([9.10.7](#9107-단일-파일-스크립트의-인라인-매니페스트))의 `toolchain: "X"`. `nabi <file>`, 그리고 패키지 밖 `.nabi` 파일의 `nabi run` / `build` / `check` / `test`에 적용합니다(프록시는 파일의 처음 64 KiB를 읽음).
4. 매니페스트의 `toolchain: "X"`(가장 가까운 `nabi.nabion`). 프록시는 이 필드만 읽으므로 모르는 필드가 있어도 고정은 유지됩니다.
5. 기본값(`$NABI_HOME/config.nabion`의 `Config { toolchain }`)

고정은 다음 중 하나입니다.

- 버전이나 그 접두사. 그 아래 버전과 맞습니다(`0.3`은 `0.3.*` 가운데 가장 높은 버전, `0.3.1`은 `0.3.1`).
- 9.10.4a와 같은 버전 요구(`^0.3.1`, `~0.3.1`, `=0.3.1`, `>=0.3, <0.5`). 받아들이는 설치 버전 가운데 가장 높은 것과 맞습니다.
- toolchain 디렉터리 이름(`0.3.1-<triple>`). 그 디렉터리와 맞습니다.

맞는 설치 릴리스 가운데 가장 높은 것이 이깁니다. prerelease(`0.3.1-rc.1`)는 맞는 릴리스가 없을 때만 이깁니다. 그래서 `0.3`은 `0.3.1-rc.1`보다 `0.3.0`을 고르고, 후보만 설치된 시리즈는 그 가운데 가장 새 것을 고릅니다. prerelease를 정확히 적은 고정(`0.3.1-rc.1`)은 그것만 맞습니다. 매니페스트의 고정이 이 중 어느 것도 아니면 무엇을 쓸지 알려 주는 경고입니다(`` `toolchain: "banana"`: write a version (`0.1`), a requirement (`^0.1.0`) or a toolchain directory name ``). 맞는 toolchain이 없으면 에러입니다. 에러는 고정과 그 출처를 밝히고, 그것을 설치하는 명령을 안내합니다. 전체 버전이면 `nabi toolchain install <pin>`, 아니면 고정이 받아들이는 전체 버전으로 `nabi toolchain install <version>`, 또는 `--from`입니다.

시리즈는 깨지는 변경 한 단계가 묶는 릴리스입니다. 메이저 버전이 0인 동안은 `0.MINOR`(`0.1`), 그 뒤로는 `MAJOR`(`1`)입니다. 수정과 보안 릴리스는 가장 새 시리즈에만 나옵니다. 이전 시리즈는 계속 내려받아 설치할 수 있지만 지원하지 않습니다.

### 호환성

toolchain은 같은 시리즈의 이전 toolchain이 빌드한 코드를 빌드합니다. 1.0 아래에서는 `x.y.z`의 `y`가 바뀌면 코드가 깨질 수 있고(`0.1.7`에서 `0.2.0`), 1.0부터는 `x`가 바뀌면 그렇습니다(`1.9.0`에서 `2.0.0`). 그보다 뒤의 `z`(1.0 아래)나 `y`(1.0부터)는 깨지 않습니다. 패키지의 `nabi_version`(9.10.3)도 이 규칙을 따릅니다. `"0.1.3"`은 0.1.3부터 마지막 0.1.x까지, `"1.4.2"`는 1.4.2부터 마지막 1.x까지 허용합니다. prerelease toolchain은 그 릴리스로 칩니다(`0.1.4-dev`는 `0.1.4`).

프록시는 설치된 toolchain 가운데 가장 새것의 `nabi` 사본입니다. 이 호스트의 가장 높은 버전이고, 프리릴리스는 릴리스가 하나도 설치되지 않았을 때만 고릅니다. 이렇게 하면 새 `nabi`가 옛 toolchain도 계속 실행하므로, 프록시 계약은 0.x 동안 고정됩니다. 위의 순서로 고른 뒤 `toolchains/<dir>/bin/<tool>`을 같은 인자와 `NABI_HOME`을 설정해 실행하고, 프록시가 떼어 낸 `+<version>`은 `NABI_TOOLCHAIN`으로 넘깁니다.

각 toolchain의 `nabi`는 자기 `nabic`, 곧 바로 옆의 `nabic`만 실행합니다. 따라서 `nabi`와 `nabic` 사이의 플래그, 환경 변수, 파일은 버전 사이의 계약이 아닙니다. 직접 실행한 `nabi`(개발 빌드, 패키지 관리자의 `nabi`, toolchain의 `bin/nabi`)은 `+<version>`, `NABI_TOOLCHAIN`, 매니페스트의 `toolchain`이 설치된 다른 toolchain을 가리키면 명령을 그 toolchain의 `nabi`에 넘깁니다. 같은 인자로, 처음 시작한 디렉터리에서, `NABI_HOME`을 설정해 실행하고 그 종료 상태로 끝납니다(Unix에서는 자신을 그 프로세스로 바꾸고, Windows에서는 기다립니다). 컴파일하는 명령(`nabic`에 위임하는 명령이나 컴파일하는 도구)이면 매니페스트는 그 명령이 다루는 패키지의 것이고(`nabi build ../app`은 `../app/nabi.nabion`을 읽음), 그 밖의 명령이면 가장 가까운 것입니다. 고정한 버전이 자기 버전과 맞을 때, 그 toolchain이 자기가 실행되는 디렉터리일 때(실제 경로로 비교), 그 toolchain의 `nabi --version`이 자기와 같은 줄을 출력할 때(같은 빌드)는 넘기지 않습니다. 고정이 없으면 기본값은 적용하지 않습니다. 직접 실행한 `nabi`가 명령을 그대로 맡습니다. `nabi toolchain`, `help`, `version`, `completions`, `--version`은 넘기지 않습니다. 명령을 넘겨받은 toolchain이 다시 되돌려 넘기면 두 실행 파일을 알리는 에러이고, 무한히 반복하지 않습니다. 설치된 toolchain을 가리키지 않는 고정은 컴파일하는 명령이면 위의 에러이고, 그 밖의 명령은 그대로 진행합니다.

```
nabi toolchain list                         # 기본 toolchain 에 * 와 (default) 표시
nabi toolchain install <version> [--from <dir | archive | url>]
nabi toolchain default [<version>]          # 인자 없으면 지금의 기본을 출력
nabi toolchain remove <version>
nabi toolchain update [<version>]
nabi toolchain clean                        # 기본값을 제외한 설치된 모든 toolchain 제거
```

- 설치본은 `$NABI_HOME/toolchains/<version>-<triple>/`에 있고, 그 안에서 완결됩니다([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)).
- `install --from`은 디렉터리, 아카이브, 또는 그 URL에서 설치합니다. `<version>`은 toolchain이 맞아야 하는 고정입니다. `--from`이 없으면 `install`은 릴리스를 내려받습니다(아래 배포). 이때 `<version>`은 전체 버전(`0.1.0`)입니다. 그 밖의 것은 전체 버전을 쓰라는 에러입니다(`` nabi toolchain install: `0.1` is not a full version; write one like `0.1.0` ``). 이미 설치된 toolchain은 다시 내려받지 않습니다(`toolchain <dir> is already installed`). 기본값은 바꾸지 않습니다.
- `install`은 `bin/nabic --version --verbose`로 검증합니다. 버전과 호스트가 맞아야 합니다.
- 처음 설치한 toolchain이 기본값이 되고, 이때 `bin/` 프록시를 만듭니다.
- `install`, `default`, `update`, `remove`, `clean`은 `bin/` 프록시가 하나라도 없거나 가장 새 toolchain의 `nabi`가 아니면 다시 쓰고, 아니면 그대로 둡니다. 옛 toolchain을 기본값으로 해도 새 프록시가 남고, 가장 새 toolchain을 지우면 그다음으로 새것의 프록시가 됩니다.
- `nabi toolchain update`는 `$NABI_HOME` 아래의 기본 toolchain을 교체합니다. 인자가 없으면 가장 새 릴리스(`latest/download/nabi-version`)를 설치합니다. 그것이 다른 시리즈여도 마찬가지입니다(rustup 방식). 기본값이 다른 시리즈로 옮겨 가면, 이전 toolchain이 남는다는 note가 수정은 가장 새 시리즈에만 나오니 프로젝트를 옮기라는 것과, 옮기기 전까지 이전 시리즈에 머무는 방법도 알려 줍니다. ``note: nabi <old version> stays installed, but <old series> gets no more fixes; move projects to <new version>. Until a project moves it can pin <old series> (`toolchain: "<old series>"` in nabi.nabion or a script's `//! nabion` block) or run `nabi +<old series> ..` ``. 인자를 주면 그 릴리스를 설치합니다. 인자는 전체 버전(`update 0.1.1`)이고, 시리즈로 고르는 기능은 없습니다. 그 밖의 것은 전체 버전을 쓰라는 에러입니다(`` nabi toolchain update: `0.1` is not a full version; write one like `0.1.0`, or nothing for the newest release ``). 설치한 toolchain이 기본값이 되고 가장 새 toolchain이 바뀌었으면 `bin/` 프록시를 다시 씁니다. Windows에서는 실행 중인 프록시의 이름을 바꾼 뒤 그 사본으로 교체합니다. 릴리스 아카이브를 받아 SHA-256을 검증하고(아래 배포), 따로 풀어 제자리로 옮깁니다. 그런 다음 프록시가 새 toolchain을 실행하는지 확인합니다. 실패하면 이전 기본값과 프록시를 되돌리고 새 toolchain을 지웁니다. 이전 toolchain은 설치된 채로 남습니다. 이미 설치된 버전은 내려받지 않고 기본값이 됩니다. `nabi update`는 `nabi.lock`을 다시 해소하고, `nabi toolchain update`는 toolchain을 교체합니다(9.10.4).
- `nabi toolchain default <version>`이나 `nabi toolchain update` 뒤에는 설치 스크립트가 쓴 자동 완성 스크립트(아래 배포)를 다시 씁니다. 새 기본 toolchain의 `nabi completions`로 씁니다. 있는 파일만 다시 쓰고 새로 만들지는 않습니다. `completions`가 없는 toolchain이면 그대로 둡니다. 아무것도 출력하지 않고, `-v`이면 다시 쓴 파일을 알려 줍니다.
- `nabi toolchain default <version>`이나 `nabi toolchain update` 뒤에 `nabi`는 `PATH`에서 `nabi`를 찾습니다(Windows에서는 `PATHEXT`의 각 확장자로). 처음 찾은 것이 `$NABI_HOME/bin/nabi`이 아니면(실제 경로로 비교) 경고합니다. 경고는 그 파일과 그 `--version`이 출력하는 버전을 알립니다. `warning: your shell runs <path> (nabi X) before ~/.nabi/bin/nabi; put ~/.nabi/bin first on PATH or remove the other one`. `-q`이면 경고하지 않습니다(설치 스크립트가 이렇게 실행함).
- `nabi --version -v`(또는 `nabi version -v`)는 버전 줄 다음에 다음 줄을 출력합니다. `binary`(이 실행 파일), `toolchain`(활성 toolchain과 그 이유), `nabic`(여기서 명령이 실행할 `nabic`. 자기 것이거나, 명령을 넘길 때는 고정한 toolchain의 것), `host`입니다. 이유는 명령줄의 `+`, `NABI_TOOLCHAIN`, 고정한 `nabi.nabion`, 기본값 중 하나이고, toolchain이 없으면 `none`입니다. 각 줄은 값을 맞춰 정렬한 `key: value` 형태로, `nabic --version -v`(`host: <triple>`)와 같습니다. `--format json`(`nabi version -v --format json`, `nabi --version --format=json`)은 대신 JSON 객체 하나를 출력합니다. `schema`(지금은 `1`), `version`, `line`(버전 줄에서 `nabi ` 뒤의 텍스트)이 있고, `-v`면 위 네 키도 있습니다.
- `nabi toolchain clean`은 지금의 기본값을 제외한 설치된 모든 toolchain을 지웁니다. `update`가 남겨 둔 이전 버전을 정리하는 선택(opt-in) 명령입니다. 프로젝트가 고정한 버전도 함께 지워지므로 빌드에 필요하면 다시 설치해야 합니다. 기본 toolchain이 없으면 에러입니다.

### 빌드 캐시

```
nabi cache dir                              # toolchain의 캐시 디렉터리를 출력
nabi cache prune [--unused <days>] [--max-size <size>]
nabi cache clean                            # 안의 것을 모두 지움
```

캐시는 활성 toolchain의 `lib/nabi-cache/`입니다([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)). 빌드가 컴파일한 표준 라이브러리 코드가 든 함수 캐시와, 라이브러리에서 모은 항목이 있습니다([9.2.16](902-aot-pipeline.md#9216-증분-빌드)). 꼭 있어야 하는 것은 없습니다. 빌드는 없는 것을 다시 계산합니다.

- `prune`은 `--unused`일(기본값 30, 빌드가 하루에 한 번 스스로 지우는 기준) 동안 어떤 빌드도 읽지 않은 항목(함수 캐시와 라이브러리 단계)을 지우고, 남은 크기가 `--max-size`(바이트, 또는 `K`, `M`, `G`를 붙인 수. 기본값은 상한 `NABI_CACHE_MAX_SIZE`, [9.2.16](902-aot-pipeline.md#9216-증분-빌드)) 이하가 될 때까지 가장 오래전에 읽은 항목부터 지운 뒤, 지운 항목과 남긴 항목의 수를 출력합니다.
- `clean`은 캐시 디렉터리 안의 것을 모두 지웁니다.

### toolchain이 함께 쓰는 파일

한 `$NABI_HOME`에 설치된 모든 toolchain은 그 안의 같은 파일을 읽고 씁니다. `config.nabion`, `tokens.nabion`, `sources.nabion`, `installs.nabion`, `envs/.current`입니다. 새 `nabi`가 옛 `nabi`가 모르는 필드를 써 두었을 수 있습니다.

- 읽을 때 `nabi`는 모르는 필드를 건너뜁니다. `installs.nabion`에서는 모르는 `Dep` 인자와 항목 종류도 건너뜁니다(모르는 종류의 항목은 목록에 나오지 않음). `-v`이면 건너뛴 것을 파일마다 한 번 알립니다. ``note: <file>: skipped `x`, written by a newer nabi (kept in the file)``.
- 쓸 때는 모르는 것을 그대로 둡니다. `nabi toolchain default`는 `config.nabion`의 그 필드 하나만, `nabi install` / `nabi uninstall`은 `installs.nabion`의 그 항목 하나만 설정하거나 지웁니다. `nabi login`은 `tokens.nabion`에서 그 index의 항목을 설정하고, `nabi logout`은 그 항목만 지웁니다. `sources.nabion`은 읽기만 합니다.
- `nabi.nabion`에서 `nabi`가 모르는 필드는 경고이고([9.10.3](#9103-nabinabion--nabilock)), `nabi`가 아는 버전의 `nabi.lock`은 여전히 엄격하게 읽습니다.
- 새 `nabi`는 새 설정을 이 파일들의 새 필드가 아니라 새 파일에 둡니다. 그래야 이전 toolchain도 함께 계속 동작합니다.

### 배포

- 릴리스 아카이브는 `nabi-<version>-<triple>.tar.gz`(Windows는 `.zip`)이고, 안은 toolchain 레이아웃입니다. Windows가 아니면 같은 파일을 담은 `nabi-<version>-<triple>.tar.xz`도 옆에 있습니다. 각 아카이브 옆에는 `<archive>.sha256`이 있습니다. 이 파일에는 아카이브의 SHA-256을 hex로 적고, 그 뒤에 파일 이름이 올 수 있습니다(sha256sum 형식).
- `install.sh`, `install.ps1`, `nabi toolchain install --from <url>`은 같은 곳에서 `<url>.sha256`을 받고, SHA-256이 다른 아카이브는 거부합니다. `.sha256`이 없으면 에러입니다. 경로로 준 아카이브는 `<path>.sha256`이 있을 때 그것과 대조합니다. 서명은 정의하지 않습니다. 출처는 HTTPS가 보증합니다.
- `install.sh`(<https://nabi.sh>)는 아카이브나 `nabi-lang` 조직의 GitHub release에서 `$NABI_HOME`에 설치합니다. tar가 읽을 수 있고(macOS, 또는 `PATH`에 `xz`) 릴리스에 있으면 `.tar.xz`를, 아니면 `.tar.gz`를 받습니다. `nabi toolchain install`은 `.tar.gz`(Windows에서는 `.zip`)를 받습니다.
- 이어서 `install.sh`는 rustup처럼 셸을 설정합니다.
  - `$NABI_HOME/env`(sh)와 `$NABI_HOME/env.fish`를 씁니다. 이 파일은 `$NABI_HOME/bin`이 `PATH`에 아직 없으면 맨 앞에 넣습니다.
  - 가드한 줄 `[ -f "$HOME/.nabi/env" ] && . "$HOME/.nabi/env"`(그 `$NABI_HOME`의 것)을 한 번씩 더합니다. 더하는 곳은 `~/.profile`, `~/.bashrc`, `~/.bash_profile`, `~/.zshenv` 중 있는 파일과 로그인 셸의 파일입니다. 로그인 셸의 파일은 없으면 만듭니다. bash는 `~/.bashrc`이고, `~/.bash_profile`이나 `~/.bash_login`이 없으면 `~/.profile`도 씁니다. zsh는 `$ZDOTDIR/.zshenv`, 그 밖의 셸은 `~/.profile`입니다.
  - fish가 로그인 셸이거나 그 설정 디렉터리가 있으면 `~/.config/fish/conf.d/nabi.fish`(`$XDG_CONFIG_HOME`이 있으면 그 아래)를 씁니다.
  - `nabi`의 자동 완성 스크립트는 `${XDG_DATA_HOME:-~/.local/share}/bash-completion/completions/`, `$NABI_HOME/zsh/`(`env`가 zsh의 `fpath`에 더함), fish는 `~/.config/fish/completions/`에 둡니다.

  바꾼 파일을 알려 주고, 새 셸이 그 변경을 읽습니다. 다시 실행해도 바뀌는 것은 없습니다. `$NABI_HOME`이 없으면 가드한 줄은 아무 일도 하지 않습니다.
- `install.ps1`은 `$NABI_HOME\bin`을 사용자 `Path` 맨 앞에 둡니다. 그리고 `$NABI_HOME\completions\nabi.ps1`을 쓰고, PowerShell 프로필에 가드한 줄을 한 번 더해 이 파일을 읽게 합니다.
- `install.sh`와 `install.ps1`은 `nabi`처럼 stderr에 상태 줄을 출력하고, 같은 팔레트와 규칙([9.1.5](901-nabic-cli.md#915-전역-옵션), [9.1.8](901-nabic-cli.md#색))을 `tput` 없이 따릅니다. `install.sh`는 stderr를 보므로 터미널에서 `curl .. | sh`도 색이 납니다.
- `--no-modify-path`(`install.ps1 -NoModifyPath`)나 `NABI_NO_MODIFY_PATH=1`을 주면 셸 파일, `Path`, 프로필을 건드리지 않고 `PATH`에 더할 내용을 출력합니다.
- 릴리스 아카이브는 공식 패키지를 `lib/nabi/packages/`에 싣고 있으므로(`nabi component`, 9.10.4a) 그것을 쓰는 코드는 네트워크 없이 빌드됩니다. 패키지마다 `nabi_version`이 그 릴리스를 허용하는 가장 새 버전입니다. `--no-packages`(`install.ps1 -NoPackages`)나 `NABI_NO_PACKAGES=1`은 설치 뒤에 설치한 toolchain에 대해 `nabi component remove packages`를 실행합니다. 그것이 실패해도 toolchain은 패키지와 함께 설치된 채로 남고, 스크립트가 그렇다고 알려 줍니다.
- 릴리스는 `$NABI_DIST`(기본값 `https://github.com/nabi-lang/nabi/releases`)에서 내려받습니다. `<dist>/download/v<version>/<archive>`와 그 `.sha256`이 있습니다. `<dist>/latest/download/nabi-version`은 시리즈와 상관없이 가장 새 릴리스를 한 줄로 적고, 인자 없는 `nabi toolchain update`가 이것을 읽습니다. 릴리스 후보(`<version>-rc.N`)는 prerelease입니다. `latest/`는 이를 적지 않으므로, 전체 버전으로만 설치합니다.
- `nabi toolchain update <version>` / `install <version>`, `install.sh --version <version>`, `install.ps1 -Version <version>`은 전체 버전을 받습니다. 가장 새 릴리스가 아닌 릴리스는 이렇게 설치합니다. 시리즈별 파일은 내보내지 않습니다.

### 설치 계약

이미 낸 모든 toolchain의 `nabi toolchain update` / `install`과 모든 릴리스의 설치 스크립트는 이후의 모든 릴리스를 설치할 수 있어야 합니다. 그래서 릴리스 아카이브는 이들이 기대는 것을 지킵니다. 이 계약은 고정이며, 이를 깨는 릴리스는 내보내지 않습니다.

- 이름: `nabi-<version>-<triple>.tar.gz`(Windows는 `nabi-<version>-<triple>.zip`)이고, Windows가 아니면 `.tar.xz`도 있습니다. 위의 배포 절에서처럼 내려받으며, 각각 옆에 `.sha256`이 있습니다.
- 포인터: `<dist>/latest/download/nabi-version`은 가장 새 릴리스를 한 줄로 적습니다. 이들의 인자 없는 `nabi toolchain update`가 이것으로 가장 새 릴리스에 닿습니다.
- 레이아웃: 최상위 디렉터리 `nabi-<version>-<triple>/` 하나에 `bin/nabi`과 `bin/nabic`(Windows는 `bin\nabi.exe`와 `bin\nabic.exe`), `lib/<runtime>`(`lib/libnabir.a`, Windows는 `lib\nabir.lib`), `lib/nabi/core/src/lib.nabi`가 있습니다.
- `bin/nabic --version --verbose`는 첫 줄에 `nabic <version>`(뒤에 공백과 다른 글이 올 수 있음)을, 그리고 `host: <triple>` 줄을 출력합니다.

## 9.10.7. 단일 파일 스크립트의 인라인 매니페스트

익명 패키지([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행))는 의존성을 ```` ```nabion ```` 블록에서만 얻습니다. 이 블록은 그 파일과 그 `mod`가 읽는 각 파일의 맨 앞(shebang 줄 뒤) `//!` 줄 안에 있고, `Package { deps: { .. } }`(와 `native`)를 담습니다(uv의 inline script metadata와 같음). 환경에 설치한 라이브러리는 쓰지 않습니다. 블록에는 `deps`, `native`, `toolchain`만 둘 수 있습니다(`features`가 없으므로 optional 의존도 없음). `toolchain`은 스크립트를 실행할 toolchain을 고정하고(9.10.6), 실행하는 파일의 블록에만 둘 수 있습니다. 모듈의 블록에 있으면 실행하는 파일을 밝히는 에러입니다. 직접 실행한 `nabi`는 패키지와 마찬가지로 스크립트를 고정된 toolchain의 `nabi`에 넘깁니다. 다른 언어의 코드 블록은 무시합니다. 블록 안의 path는 그 블록을 담은 파일 기준입니다.

블록은 합쳐집니다. 두 파일이 한 라이브러리에 서로 다른 항목을 주면 에러이고, 에러는 두 파일과 두 항목을 알립니다. 서브모듈 파일을 직접 실행하면 그 실행이 닿는 파일의 블록만 씁니다. `nabi add <name> .. <file.nabi>` / `nabi remove <name> <file.nabi>`이 블록을 씁니다([9.10.4](#9104-의존성-nabi-add--remove--update--deps)). `nabi init --script`는 스크립트를 패키지로 바꿉니다([9.10.2](#9102-nabi-new--nabi-init-패키지-생성)).

스크립트는 lockfile과 빌드를 `cache/scripts/<hash>/`에 둡니다. 키는 스크립트 경로와, 스크립트가 닿는 모든 파일의 경로와 블록입니다. 스크립트의 실행 파일은 빌드를 이루는 모든 것을 키로 캐시합니다. 여기에는 스크립트가 포함하는 모든 파일과 그 라이브러리가 들어갑니다.

스크립트에 없는 패키지를 `use x::..`로 쓰면 에러입니다. 에러는 실행할 명령 ``run `nabi add x <file>` ``을 help로 주고, 활성 환경에 `x`가 있으면 note를 덧붙입니다. 패키지 안에서는 `nabi add x`를 제안합니다. `import x`(와 `include` / `require`)는 에러입니다. help는 파일 옆에 `x.nabi`가 있으면 `mod x;`, `x`가 경로 루트이거나 파일이 선언한 모듈이면 `use`, 그 밖에는 `nabi add x <file>`입니다.

````nabi
//! ```nabion
//! Package {
//!     deps: { "util": Dep::git("https://example.com/util.git", tag: "v1") },
//! }
//! ```

use util::greet;
greet();
````

단일 파일 스크립트를 다시 실행하면 이전 실행 파일을 재사용합니다. 익명 패키지의 `nabi run`은 실행 파일을 `<target>/[<triple>/]<profile>/scripts/<key>/`에 둡니다. `<target>`은 스크립트의 `target/`(`cache/scripts/<hash>/` 아래, 그 `nabi.lock` 옆)입니다. `nabi.nabion` 때문에 단일 파일 패키지가 된 파일(9.10.1)과 `nabic`가 빌드한 단일 파일은 대신 단일 파일 캐시를 씁니다([9.2.16](902-aot-pipeline.md#9216-증분-빌드)). `<key>`는 빌드를 이루는 모든 것의 해시입니다. 여기에는 다음이 들어갑니다.

- toolchain, 빌드 옵션, 린트 수준과 타깃
- 소스와 소스가 포함하는 모든 파일
- toolchain이 그 타깃용으로 배포하는 런타임과 C 라이브러리
- 미리 빌드한 라이브러리와 함께 넘긴 의존 패키지

키가 맞으면 아무것도 컴파일하지 않고, 그 실행 파일을 만든 빌드의 경고를 다시 출력합니다. 새 실행 파일은 임시 이름으로 쓴 뒤 제자리로 이름을 바꿉니다. 그래서 한 스크립트를 동시에 실행해도 안전합니다. 하루 동안 쓰지 않은 항목은 다른 항목을 빌드할 때 지웁니다.

## 9.10.8. 도구

도구는 `nabi` 자신의 서브명령입니다. `nabi`는 컴파일러를 링크해 담고 있어서 도구를 자기 프로세스 안에서 실행합니다. 컴파일하는 도구는 `nabic build`의 컴파일 옵션([9.1.2](901-nabic-cli.md#nabic-build-aot-빌드))을 받고, 9.10.1처럼 패키지의 의존성, feature, 린트를 받습니다. 진단, 진행 줄, 색은 [9.1.8](901-nabic-cli.md#918-출력-형식)을 따르고, 사용법 에러와 종료 코드는 도구가 따로 정하지 않으면 [9.1.6](901-nabic-cli.md#916-종료-코드)을 따릅니다. `nabi lsp`, `nabi repl`, `nabi jupyter`는 내부 컴파일러 에러를 견딥니다. 언어 서버는 이를 분석한 버퍼의 에러 진단으로 보고하고, REPL / 커널은 그 셀을 실패로 처리합니다([9.7.5](907-jit-cells.md#975-에러-모드)). 둘 다 계속 서비스합니다.

### `nabi run`: 빌드 후 실행

```
nabi run [path] [build options] [-- args...]
```

`nabi run`은 실행 패키지를 `nabic build`처럼 빌드한 뒤 곧바로 실행합니다(다른 `--target`의 실행 파일은 아래의 러너로 실행합니다). `--` 뒤에 적은 인자는 어떤 바이트든 실행 파일에 그대로 전달됩니다([9.1.1](901-nabic-cli.md#911-일반-형태)). 옵션은 `nabic build`와 같습니다. 라이브러리 패키지에는 실행할 것이 없으므로 에러입니다.

패키지는 `target/<profile>/<name>`에 빌드합니다. 단독 `.nabi` 파일은 익명 패키지로 빌드해 9.10.7의 캐시에서 실행하므로 파일 옆에 `target/`을 만들지 않습니다. `nabic build <file>.nabi`는 이를 빌드해 실행 파일을 파일 옆에 복사합니다. 이전 빌드 뒤로 바뀐 것이 없으면 `nabi run`은 `Fresh`를 출력하고, 그 빌드의 경고를 다시 출력한 뒤 빌드를 건너뜁니다([9.2.16](902-aot-pipeline.md#9216-증분-빌드)).

```
nabi run hello.nabi -- arg1 arg2
```

`nabi run`의 종료 코드는 프로그램의 종료 코드입니다. 프로그램이 시그널 `n`으로 끝나면 `128 + n`입니다.

프로그램은 `nabi run`보다 오래 남지 않습니다. `nabi`에 보낸 `SIGTERM`, `SIGHUP`, `SIGINT`, `SIGQUIT`는 프로그램에 전달됩니다(터미널이 포그라운드 그룹에 보내는 시그널은 이미 프로그램에 닿습니다). 프로그램이 그 시그널로 끝나면 `nabi run`도 같은 시그널로 끝납니다. `nabi run` 자체가 강제 종료되면 프로그램도 함께 종료됩니다. Windows에서는 프로그램이 `nabi`와 함께 끝나는 job object 안에서 실행됩니다.

### `nabi dev`: 핫 리로드로 실행

```
nabi dev [path] [-p <member>] [--bin <name>] [build options] [-- args...]
```

`nabi dev`는 패키지의 프로그램을 핫 리로드용으로 빌드하고([9.14](914-hot-reload.md)) 실행한 뒤 소스를 지켜봅니다. 저장하면 그 저장이 건드린 로컬 패키지를 다시 빌드하고, 프로그램은 실행 중에 새 코드를 받아 상태를 유지합니다.

- *호스트*(런타임과 모든 레지스트리 패키지, git 패키지)는 `target/dev/<name>-host`로 한 번 빌드합니다. 로컬 패키지(이 패키지, 그 `path` 의존성, 이 패키지가 쓰는 워크스페이스 멤버)는 각각 핫 플러그인이며, `nabic build --hot`으로 `target/dev/<package>.<n>.dylib`(`.so`, `.dll`)에 빌드합니다. 빌드 번호 `n`은 1부터이고, 그 레이아웃 `target/dev/<package>.<n>.layout`이 옆에 놓입니다. 세션이 시작될 때 `target/dev/`를 비웁니다.
- 로컬 패키지의 `src/` 트리를 100 ms마다 훑습니다(각 `.nabi` 파일의 수정 시각과 크기). 파일이 50 ms 동안 그대로이면 변경을 받아들이므로, 여러 파일을 한 번에 저장해도 변경은 하나입니다. 파일이 바뀐 패키지와 그에 의존하는 모든 로컬 패키지를 의존성 먼저 증분으로 다시 빌드하고([9.2.16](902-aot-pipeline.md#9216-증분-빌드)), 새 버전을 프로그램에 보냅니다([9.14.5](914-hot-reload.md#9145-dev-채널)).
- 프로그램의 답은 상태 줄로 출력합니다: `Applied`, `Waiting`, `Refused`(할 일과 함께), `Superseded`.
- 컴파일 에러는 `nabi build`처럼 출력하고, 이어서 `` Kept the running code of `game`: fix the error and save ``를 출력합니다. 프로그램은 옛 코드로 계속 실행됩니다.
- `nabi.nabion`(이 패키지의 것, 로컬 의존성의 것, 워크스페이스 루트의 것), `nabi.lock`, 또는 toolchain의 `nabi`나 `nabic`가 바뀌면, 이유를 말하는 줄(`` Restarting nabi.nabion changed ``)을 출력하고 프로그램을 재시작합니다. 재시작은 의존성을 다시 해소하고 모두 새로 빌드하므로 프로그램의 상태는 사라집니다.
- 키: `r`은 재시작, `q`는 끝냄입니다. 터미널에서는 키를 누르는 즉시 동작합니다(`nabi dev`는 줄 편집과 에코를 끄고, 끝날 때 되돌립니다). 그 밖에는 각 줄의 첫 글자가 키입니다. 프로그램의 표준 입력은 비어 있고, 표준 출력과 표준 에러는 `nabi dev`의 것입니다.
- 프로그램이 스스로 끝나면 `nabi dev`는 그 종료 코드를 알리고 기다립니다. 저장하거나 `r`을 누르면 다시 실행하고, `q`로 끝냅니다. 핫 플러그인이 로드되지 않은 프로그램은 로더의 에러([9.12.6](912-plugins.md#9126-안정성과-오류)) 뒤에 종료 코드 101로 끝납니다.
- Ctrl-C(또는 `SIGTERM`, `SIGHUP`)는 세션을 끝냅니다. 프로그램을 종료하고 회수한 뒤 `nabi dev`는 130으로 끝납니다. `q`는 0으로 끝납니다.
- `--` 뒤의 인자는 시작할 때마다 프로그램에 넘깁니다.

`-p <member>`는 워크스페이스 멤버를 실행하고, `--bin <name>`은 패키지의 실행 파일을 가리킵니다(패키지에는 패키지 이름을 딴 실행 파일이 하나 있습니다). 빌드 옵션은 `nabic build`의 것이되 이 기계의 dev 프로파일만 됩니다. `--release`나 다른 `-O` 수준은 에러이고(``hot reload runs debug builds; drop `--release` ``), `--target`, `-o`, `--crate-type`도 에러입니다(``` `nabi dev` builds for this machine only; drop `--target` ```). `src/main.nabi`가 없는 패키지는 프로그램의 패키지에서 실행하라는 에러입니다.

### 다른 타깃의 실행 파일

`nabi run`, `nabi test`, `nabi bench`에 이 기계가 직접 실행하지 못하는 `--target`을 주면, 실행 파일마다 타깃 러너를 거쳐 시작합니다. 타깃 러너는 실행 파일과 그 인자를 차례로 받는 프로그램입니다.

- `NABI_RUNNER=<program> [args]`(공백으로 나눔)는 호스트가 아닌 모든 타깃의 러너입니다.
- 이 변수가 없으면 `wasm32-wasip2`와 `wasm32-wasip1-threads` 모듈은 `PATH`의 `wasmtime`으로 실행합니다. `wasmtime run`에 호스트의 환경(`wasm32-wasip2`에서는 네트워크와 이름 조회도), 네이티브 주 스레드만큼 깊은 8 MiB 엔진 스택, `/`에 둔 호스트 파일 시스템을 줍니다. WASI에는 작업 디렉터리가 없으므로 상대 경로는 `/`에서 찾습니다.
- `wasm32-unknown-unknown` 모듈은 `PATH`의 `node`와 toolchain의 JS 호스트(`lib/nabi/targets/wasm32-unknown-unknown/`의 `nabi.mjs`)로 실행합니다.
- Linux에서 CPU가 다른 Linux 타깃은 `PATH`에 있는 qemu-user의 `qemu-<cpu>`(`qemu-aarch64`, `qemu-arm`, `qemu-riscv64`, `qemu-ppc64le`, `qemu-s390x`, `qemu-i386`, ...)로 실행합니다. `/usr/<triple>`(`/usr/aarch64-linux-gnu`) 디렉터리가 있고 `QEMU_LD_PREFIX`가 없으면 `-L /usr/<triple>`을 더합니다. `qemu-<cpu>`가 없어도 binfmt_misc에 `qemu-<cpu>` 처리기가 등록되어 있으면 직접 실행합니다.
- 호스트와 운영체제가 같고 호스트가 실행할 수 있는 다른 CPU의 타깃은 qemu-user를 찾기 전에 직접 실행합니다. x86_64에서 32비트 x86, 그리고 CPU가 EL0에서 AArch32 코드를 실행하는 arm64 Linux(Neoverse N1, Cortex-A72. Neoverse N2와 Apple silicon은 아님)에서 32비트 ARM이 그렇습니다. 프로그램에 그것을 알려 주는 레지스터가 없으므로, `nabi`는 작은 정적 32비트 ARM 실행 파일을 한 번 실행해 알아냅니다.

그 밖의 경우나 프로그램이 없으면, 무엇을 설치할지 또는 `NABI_RUNNER`를 알려 주는 에러로 명령이 실패합니다: ``cannot run a wasm32-wasip2 module here: `wasmtime` is not on PATH; install it (https://wasmtime.dev), or set NABI_RUNNER to a program that runs it``. `nabi run`은 빌드하기 전에 확인합니다. `nabi test`와 `nabi bench`는 하네스를 빌드한 뒤에 확인하고, `--list`는 러너 없이도 목록을 출력합니다. 종료 코드는 러너의 종료 코드이고, 위의 러너는 모두 프로그램의 종료 코드를 그대로 돌려줍니다.

```
nabi test --target wasm32-wasip2
NABI_RUNNER="qemu-aarch64 -L /opt/sysroot" nabi run --target aarch64-unknown-linux-gnu
```

### `nabi lint`: 린터

```
nabi lint [path] [--list] [--check-library] [-A|-W|-D <lint | group>..]
```

`nabi lint`는 프런트엔드와 린트를 돌리고 코드 생성은 하지 않습니다(`nabic check`는 코드 생성까지 함, [9.1.2](901-nabic-cli.md#nabic-check-진단)). 컴파일하는 모든 명령이 같은 린트를 보고합니다([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). `lint`는 린트와 프런트엔드만 돌립니다. 진단 형식은 [9.1.8](901-nabic-cli.md#918-출력-형식)을 따르고, 린트 진단의 코드는 `E0131`입니다. 린트 목록과 그룹은 [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)에 있습니다. `--list`는 모든 린트를 기본 수준, 추가된 릴리스와 함께 출력하고, 이어서 그룹을 출력합니다. `--check-library`는 `nabic check`와 같습니다.

린트 수준은 `-A` / `-W` / `-D`([9.1.5](901-nabic-cli.md#915-전역-옵션))와 코드의 속성이 정합니다. `deny` 수준의 린트가 하나라도 걸리면 종료 코드는 `1`입니다.

### `nabi fix`: 수정 적용

```
nabi fix [path] [--allow-dirty] [--diff] [-A|-W|-D <lint | group>..]
```

`nabi fix`는 `nabi lint`처럼 패키지를 검사하고, 찾은 것에 딸린 수정을 적용합니다. 대상은 편집이 정확히 하나인 린트 결과와 컴파일러 경고입니다(대안이 여럿인 진단은 에디터의 빠른 수정에 맡김). 다시 검사해 편집이 있는 동안 되풀이하며, 최대 8번입니다. 같은 차례의 다른 편집과 겹치는 편집은 다음 차례로 미룹니다. 패키지의 파일만 바꾸고, 의존 패키지와 표준 라이브러리는 바꾸지 않습니다. 수정이 있는 린트는 다음과 같습니다.

| 린트 | 수정 |
|------|-----|
| `unused_variables` | `let` 이름 앞에 `_`를 붙임(`let _y = 2;`) |
| `unused_imports` | 그 이름만 가져오고 한 줄에 홀로 있으며 위에 속성이 없는 `use`를 지움 |
| `bool_comparison` | `x == true`는 `x`로, `x == false` / `x != true`는 `!x`로 바꿈(연산자가 있으면 `!(x)`) |
| `while_true` | `while true`를 `loop`로 바꿈 |
| `redundant_cast` | `x as T`를 `x`로 바꿈 |
| `redundant_clone` | `.clone()`을 지움 |
| `missing_f_prefix` | 문자열에 `f` 접두사를 붙임 |

린트 수준(`-A` / `-W` / `-D`, 매니페스트의 `lints`, 속성)이 무엇을 보고할지, 곧 무엇을 고칠지 정합니다. `nabi fix -A unused_imports`는 쓰지 않는 import를 남깁니다. 패키지가 git 작업 트리 안에 있고 그 디렉터리 아래에 커밋하지 않은 변경이 있으면 `nabi fix`는 거절합니다(``the work tree has uncommitted changes (src/main.nabi); commit them first so the fixes can be reviewed and undone, or run `nabi fix --allow-dirty` ``). `--allow-dirty`이면 그래도 고칩니다. git 밖에서는 이 검사가 없습니다. `--diff`는 첫 차례의 편집을 `nabi fmt --diff`처럼 출력하고 아무것도 쓰지 않습니다. 수정으로 풀리지 않는 에러가 있으면 종료 코드 `1`과 ``N errors to fix by hand first; nothing was changed``로 멈춥니다(한 차례 뒤라면 ``the files fixed so far are kept``). 끝에 바뀐 파일마다 `Fixed <file> (N fixes)`, 없으면 `<name>: nothing to fix`를 알립니다.

### `nabi repl`: 터미널 REPL

`nabi repl`은 터미널에서 인터랙티브 셸을 띄웁니다. 서브명령 없는 `nabi`도 같습니다(9.10.1).

```
nabi repl [path] [-q]
```

셀의 의미는 [9.7.0](907-jit-cells.md#970-셀-세션의-규칙)의 규칙을 따릅니다. 각 셀은 한 번만 실행됩니다. 최상위 `let` 바인딩과 항목은 이후 셀로 이어지고, 실패한 셀은 바인딩에 흔적을 남기지 않습니다. 식 셀은 값을 출력합니다. `Display`로 출력하고, 그것이 없으면 `Debug`로, 둘 다 없으면 타입 이름을 출력합니다.

- 기본 모드는 각 셀을 Nabi의 코드 생성기로 컴파일해, 세션 동안 사는 JIT 호스트 프로세스에 넘깁니다. 호스트는 셀을 로드해 한 번 실행합니다([9.6](906-jit-engine.md)). 바인딩, `static` 상태, `any` 값이 셀 사이에 이어집니다. 최상위에서 이름을 적을 수 없는 타입(클로저, 비공개 타입 등)의 바인딩은 만들 때 거부합니다.
- 진단은 셀 자신의 텍스트를 기준으로 `<repl>:line:col`로 표시합니다. 앞 셀의 항목과 실행 중 위치는 `<repl cell N>:line:col`로 표시합니다([9.7.5](907-jit-cells.md#975-에러-모드)).
- 터미널에서는 줄 편집, Tab 자동 완성, 색, 셀 단위 기록을 지원합니다([9.7.8](907-jit-cells.md#줄-편집)). 기록은 `$NABI_HOME/repl_history`에 남습니다.
- 터미널에서는 세션이 toolchain 줄과 안내 한 줄로 시작합니다([9.7.8](907-jit-cells.md#줄-편집)). `-q`를 주면 이 줄을 생략합니다.
- 경로가 없으면 빈 세션으로 시작합니다. 프로그램 경로(단독 `.nabi` 파일이나 `src/main.nabi`가 있는 패키지)를 주면 로드해 실행하고 세션을 이어 갑니다. 이때 최상위 문은 셀로, `fn main`은 진입점으로 실행합니다([9.7.10](907-jit-cells.md#9710-프로그램-로드repl-path)). 라이브러리 패키지 경로(`src/lib.nabi`)는 의존성으로 붙이고, `use <name>::*;`를 첫 셀로 넣습니다. `:reset` 뒤에도 다시 넣습니다.
- 패키지 안에서 경로 없이 시작하면 그 패키지를 로드하지 않습니다. 찾지 못한 이름을 그 패키지가 정의하고 있으면 ``load this package: `nabi repl .` `` 힌트를 붙입니다.
- 경로가 둘 이상이면 사용법 에러입니다. `-`는 받지 않습니다. 표준 입력은 셀 입력으로 씁니다.
- `NABI_REPL_TIME=1`이면 셀의 단계마다 걸린 시간을 stderr에 출력합니다([9.9.9](909-debug-tools.md#999-repl--jupyter-디버그)).

루트 파일이 둘 다 있는 패키지는 프로그램 경로로 다루고, 그 라이브러리는 세션의 의존 패키지가 됩니다. `nabi repl <path>`는 그 경로의 프로젝트에서 의존성과 린트를 정합니다(9.10.1). 패키지 밖에서는 세션이 활성 환경의 라이브러리를 씁니다(9.10.5).

REPL 명령(`:type`, `:doc`, `:time`, `:vars`, `:load`, `:reload`, `:edit`, `:save`, `:opt`, `:reset`, `:add`, `:help`, `:quit` / `:exit`, 그리고 단독 `exit` / `quit`)과 디버그 수단은 [9.9.9](909-debug-tools.md#999-repl--jupyter-디버그)에 있습니다.

### `nabi jupyter`: Jupyter 커널

```
nabi jupyter install [--env <name>]
nabi jupyter --connection-file <path>
```

`nabi jupyter install`은 커널을 등록합니다([9.8.3](908-jupyter-integration.md#983-kernelspec-설치)). kernelspec은 `nabi jupyter --connection-file {connection_file}`을 실행합니다.

`nabi jupyter --connection-file <path>`가 커널입니다. Jupyter가 시작하므로 사용자가 직접 실행하는 명령이 아닙니다. `--connection-file`은 Jupyter가 제공하는 ZMQ 연결 정보 파일의 경로를 받습니다. 인자 없는 `nabi jupyter`는 `nabi jupyter install`을 알려 주는 사용법 에러입니다. `--features` 같은 빌드 옵션도 함께 받습니다. 커널은 REPL과 같은 셀 세션([9.7.0](907-jit-cells.md#970-셀-세션의-규칙)) 위에서 동작하고, 시작한 곳의 패키지를 `nabi repl .`처럼 로드합니다. 연결을 받을 수 있게 되면 stderr에 `nabi jupyter: ready`를 출력합니다([9.8.2](908-jupyter-integration.md#982-로컬-네이티브-커널-동작)). 자세한 통합 내용은 [Jupyter 통합](908-jupyter-integration.md)에서 정의합니다.

### `nabi lsp`: LSP 서버

```
nabi lsp
```

`nabi lsp`는 표준 입출력으로 Language Server Protocol 서버를 실행합니다. VS Code, Vim, Emacs 같은 LSP 클라이언트가 호출하는 용도입니다. 에디터용 독립 바이너리 `nabils`도 같은 서버입니다. `.nabion` 버퍼는 구문만 검사합니다. 루트 파일이 둘 다 있는 패키지에서는 라이브러리의 모듈 트리가 닿는 파일을 라이브러리의 일부로 검사하고, 그 밖의 파일은 실행 파일의 일부로 검사합니다. 실행 파일은 라이브러리의 열린 버퍼를 봅니다. 기능 목록은 [9.9.13](909-debug-tools.md#9913-lsp-서버)에 있습니다. 검사는 백그라운드에서 돌고, 그동안 질의는 마지막으로 통과한 검사로 답합니다.

### `nabi fmt`: 포매터

```
nabi fmt [path..] [--check] [--diff]
```

`[path]`는 파일이나 디렉터리를 받고(여러 개 가능), [표준 포맷](#표준-형식)에 맞게 그 자리에서 고쳐 씁니다. 패키지인 디렉터리([9.2.2](902-aot-pipeline.md#922-패키지-발견))는 `src/`, `tests/`, `benches/`, `examples/` 디렉터리와 `nabi.nabion`을 포맷하고, 그 안의 다른 파일은 건드리지 않습니다. 경로가 없으면 현재 디렉터리가 속한 패키지를 포맷합니다. 그 패키지는 현재 디렉터리나 그 위에서 `src/main.nabi`, `src/lib.nabi`, `nabi.nabion` 중 하나가 있는 가장 가까운 디렉터리입니다. 패키지 밖에서 경로 없이 실행하면 에러(종료 코드 `1`)이고, 메시지는 `` not inside a package; pass a file or directory: `nabi fmt <path>` ``입니다. 디렉터리는 재귀로 `.nabi`와 `.nabion` 파일을 찾습니다. 이때 `target/`, `node_modules/`, `.`으로 시작하는 디렉터리, 그리고 찾는 디렉터리나 그 패키지 루트의 `.gitignore`가 무시하는 것은 건너뜁니다. `.gitignore`에서는 `name`(아래 어디서든 그 이름), `name/`(디렉터리만), `/name`(`.gitignore`가 있는 디렉터리 바로 아래), `*suffix` 형태의 줄만 읽습니다. 주석, `!` 줄, 다른 와일드카드, 중간에 `/`가 있는 경로는 건너뜁니다. 터미널에서는 찾는 데 1초가 넘게 걸리면 표준 에러에 `Scanning <n> files…`를 보여 줍니다. `.nabion` 파일([NABION](../../refs-standard-libraries/300-std/308-nabion.md#384-정규-형식))은 NABION인지 검사하고 들여쓰기만 다시 맞춥니다. 정규 형식으로 다시 쓰지 않으므로 주석이 남습니다.

`-q`가 아니면 마지막에 한 일을 표준 에러로 알립니다. 메시지는 `Formatted <n> files`(이미 포맷된 파일이 있으면 `, <m> already formatted`를 덧붙임), `note: <n> files already formatted` 가운데 하나입니다. `--check` / `--diff`이면 `` note: <n> files need formatting; `nabi fmt` without `--check` rewrites them ``입니다. 포맷할 것이 없는 디렉터리는 `note: no .nabi or .nabion files under <dir>`입니다(종료 코드 `0`).

옵션은 다음과 같습니다.

- `--check`는 파일을 바꾸지 않고, 바뀔 파일과 그 diff를 출력합니다. 바뀔 파일이 있으면 종료 코드는 `1`이므로 CI에서 쓰기 좋습니다.
- `--diff`는 파일을 바꾸지 않고 diff만 출력합니다.

어휘 에러가 있는 파일은 건드리지 않고 에러를 보고합니다. 종료 코드는 `1`입니다.

#### 표준 형식

`nabi fmt`의 표준 포맷은 **토큰을 보존하는 배치 규칙**입니다. 줄 안의 토큰과 주석은 바꾸지 않고, 줄의 들여쓰기와 빈 줄을 정하며, 모듈 맨 위의 `mod` / `use` 항목을 정렬합니다. `nabi fmt`에는 일부러 설정이 없습니다. 모든 Nabi 코드의 표준 포맷은 하나입니다.

1. 들여쓰기는 괄호(`(` `[` `{` `#[`) 깊이마다 공백 4칸입니다. 닫는 괄호로 시작하는 줄은 여는 줄의 깊이를 따릅니다. 탭은 공백으로 바뀝니다.
2. 이어지는 줄은 원래 파일에서 블록 들여쓰기보다 더 들여쓴 만큼을 유지합니다. 이어지는 줄은 앞 토큰이 `;` `,`나 괄호가 아닌 줄입니다(긴 식을 나눈 줄 등).
3. 줄 끝 공백을 지웁니다.
4. 연속된 빈 줄은 하나로 줄이고, 파일 처음의 빈 줄은 지웁니다.
5. 파일은 줄바꿈 하나로 끝납니다.
6. 파일과 인라인 `mod name { }` 맨 위(다른 첫 항목 앞)의 `mod name;`과 `use` 항목은 `pub mod`, `mod`, `pub use`, `use` 순서로 묶습니다. `pub(..)`은 모두 `pub`으로 칩니다. 묶음마다 정렬하고(`mod`는 이름순, `use`는 경로를 마디마다 비교하되 `self`, `sup`, `pkg`가 먼저), 묶음 사이에는 빈 줄 하나를 두고 묶음 안에는 빈 줄을 두지 않습니다. 항목 바로 위의 주석, 문서 주석, 속성은 항목과 함께 움직입니다. 첫 항목 위의 주석은 제자리에 남습니다. 다른 항목과 한 줄에 있는 항목에서 맨 위가 끝납니다.

6번을 빼면 포매터는 줄을 옮기지 않고, 줄을 나누거나 합치지 않으며, 줄 안의 간격을 바꾸지 않습니다. 따라서 주석은 적힌 그대로 남고, 토큰 순서는 6번이 정렬하는 항목의 순서 말고는 바뀌지 않습니다. 여러 줄에 걸친 문자열과 블록 주석의 안쪽도 그대로 둡니다.

1.0 전까지는 이 배치 규칙이 표준 포맷입니다. 그때까지 코드를 다시 짜는 pretty-printer(토큰을 옮기거나 줄을 나누고 합치는 포매터)로 바꾸지 않습니다. 따라서 `nabi fmt --check`를 통과한 파일은 이후 0.x toolchain에서도 계속 통과합니다.

### `nabi doc`: 문서 생성

```
nabi doc [path] [--std] [--open]
```

패키지의 라이브러리(없으면 실행 파일)와 그 의존 패키지의 공개 항목과 문서(`///`, `//!`, `#[doc]`, [6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---문서-주석-형태))로 HTML API 문서를 `target/doc/`에 만듭니다.

```
target/doc/
├── <pkg>/index.html           # 패키지의 루트 모듈
├── <pkg>/<mod>.html           # 공개 모듈마다 한 쪽
├── search-index.js            # 검색용 색인
├── search.js, keys.js         # 검색과 그 `/` 단축키
├── theme.js                   # 밝은 / 어두운 테마
├── nabi.css, style.css        # 스타일 시트
└── fonts/, favicon.svg
```

`<pkg>`는 패키지 이름(매니페스트의 `name`)이고, 표준 라이브러리 계층은 자기 이름(`core`, `std`, `bare`)입니다. 경로 위의 모든 모듈이 `pub`이고 `#[doc(hidden)]`이 아니면 그 모듈은 공개 모듈이고, 공개 모듈마다 한 쪽이 있습니다. 루트는 `<pkg>/index.html`, `a::b`는 `<pkg>/a/b.html`입니다(루트의 자식 중 이름이 `index`인 모듈은 `<pkg>/index/index.html`). 패키지 루트 쪽 위에 따로 쪽은 없습니다. 각 쪽 머리말에는 그 쪽 패키지의 이름(패키지 루트로 링크)과 버전이 나오고, 사이드바에는 패키지의 의존성이 각 루트 쪽 링크와 함께 나옵니다. 사이드바와 경로 표시는 그 쪽 패키지의 루트로 이어집니다. 검색은 어느 쪽에서든 문서화된 패키지 전체를 찾습니다. 이전 toolchain이 남긴 `target/doc/index.html`은 지우고, `target/doc/`의 다른 파일은 지우지 않습니다. 스타일 시트, 스크립트, 글꼴이 출력에 들어 있으므로 네트워크 없이 볼 수 있습니다.

모듈의 쪽에는 모듈의 문서(그 `//!` 줄. `mod` 항목에 `///`가 있으면 그 뒤에 붙습니다. 평범한 `//` 주석은 문서 주석이 아니며, 라이브러리나 공개 모듈의 파일을 그런 주석으로 시작하면 `nabi lint`가 경고합니다, [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)), 공개 자식 모듈, 그리고 종류별 항목이 이 순서로 나옵니다. 재내보내기, 구조체(공용체 포함), 열거형, 트레잇, 타입 별칭, 함수, 상수, 정적 변수, 마지막으로 다른 곳에서 선언한 타입의 구현(`impl str { .. }`)입니다. 항목은 `#[doc(hidden)]`도 `#[internal]`도 아닌 `pub` 항목이고, 각 항목에는 작성한 그대로의 시그니처, 문서, 멤버(공개 필드, 변형, 트레잇 항목, 고유 impl의 공개 메서드와 상수), 트레잇 구현이 함께 나옵니다. 쪽에 무엇을 보일지는 다음 규칙으로 정합니다.

- 비공개 모듈에 있는 항목, 또는 비공개 모듈만 거쳐 닿는 항목을 `pub use`로 내보내면, 그 항목은 내보내는 쪽에 그곳에서 선언한 것처럼 내보낸 이름으로 나오고, `pub use`의 문서가 먼저 옵니다. `pub use private::*`의 항목도 모두 그렇습니다.
- 공개 모듈의 쪽에 나오는 항목을 다시 내보내면 재내보내기 항목으로 남고, 그 쪽으로 링크합니다.
- 비공개 모듈에 쓴 `impl`의 메서드는 그 위로 가장 가까운 공개 모듈의 쪽에서 해당 타입에 붙고, 타입이 없으면 그 쪽의 구현에 들어갑니다.
- 항목마다 앵커 `#<kind>.<name>`이 붙고, `<kind>`는 `struct`(공용체도), `enum`, `trait`, `type`, `fn`, `const`, `static` 중 하나입니다(`app/index.html#struct.Point`). 멤버는 `#<Type>.<member>`(`#Point.x`, `#Shape.Circle`, `#Point.origin`), 구현은 `#impl.<Type>`, 재내보내기는 가져오는 이름을 `.`로 이은 `#reexport.<names>`입니다. 재내보내기 안의 이름마다 그 이름이 가리키는 항목의 `#<kind>.<name>`도 붙으므로, `#struct.Event`로 `Event`를 내보내는 쪽에서도 찾을 수 있습니다.
- 시그니처의 타입과, 인라인 코드에서 문서화된 항목을 가리키는 이름은 그 항목을 보여 주는 쪽으로 링크합니다. 패키지 안이든 의존성이든 같습니다. 어느 쪽에도 나오지 않는 이름은 그냥 텍스트로 남습니다. 모든 링크는 실제로 있는 쪽과 앵커를 가리킵니다.

docs.nabi-lang.org의 표준 라이브러리 API도 이 규칙으로 만듭니다. 모듈마다 쪽이 하나이고(그곳은 `std/core/io.html`, 여기는 `core/io.html`) 앵커가 같으며, `nabi doc`의 쪽은 그 스타일 시트와 배치를 그대로 씁니다.

`--std`를 주면 표준 라이브러리 계층도 `core/`, `std/`, `bare/`로 문서화합니다. 경로가 없으면 표준 라이브러리 문서만 `./target/doc/`에 만듭니다.

`nabi doc`은 패키지의 루트 쪽을 출력합니다(`Generated ./target/doc/<pkg>/index.html`). 경로 없이 `--std`를 주면 `std/index.html`입니다. `--open`은 문서를 만든 뒤 그 쪽을 엽니다. `$BROWSER`가 있으면 그것으로, 없으면 시스템의 opener(macOS는 `open`, Linux는 `xdg-open`, Windows는 `start`)로 엽니다. opener를 시작할 수 없으면 직접 열 파일을 경고로 알립니다. 종료 코드는 바뀌지 않습니다.

문서 텍스트는 markdown의 부분집합(제목, fenced 코드 블록, 인라인 코드, 링크, `**bold**`, 목록, 문단)으로 해석합니다. Nabi 코드 블록에서 `# `로 시작하는 줄(또는 `#`만 있는 줄)은 보이지 않습니다. `nabi test`가 실행하는 그 블록의 문서 테스트([9.1.2](901-nabic-cli.md#문서-테스트))를 위한 준비 코드입니다.

### `nabi clean`: 빌드 산출물 제거

```
nabi clean [path]
```

패키지의 `target/` 디렉터리 전체를 지웁니다. 모든 프로필과 교차 빌드가 포함되고, `--target-dir` / `NABI_TARGET_DIR`이 있으면 그 디렉터리를 지웁니다. 디렉터리가 없으면 아무 일도 하지 않습니다.

### `nabi bindgen`: C 헤더 바인딩

```
nabi bindgen <header> [-x objective-c] [-o <file>] [--target <t>] [--c-shim <file.c>] [-- <C compiler args>]
```

`<header>`의 선언을 [std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#176-자동-바인딩-생성) 형식으로 표준 출력(또는 `-o <file>`)에 씁니다. 출력의 첫 줄은 ``// Generated by `nabi bindgen` ``로 시작합니다. `nabi`는 헤더를 toolchain의 C 컴파일러([9.13](913-c-compiler.md))의 프런트엔드로 읽습니다. 다른 C 컴파일러도 libclang도 필요하지 않습니다. `--` 뒤의 인자는 헤더가 선언하는 내용을 바꾸는 C 컴파일러 옵션입니다. `-I <dir>`, `-isystem <dir>`, `-iquote <dir>`, `-idirafter <dir>`, `-D <name>[=<value>]`, `-U <name>`, `-include <file>`, `--sysroot <dir>`, `-isysroot <dir>`, `-nostdinc`, `-nostdlibinc`, `-nobuiltininc`를 받습니다. `-std=`와 경고 옵션은 받되 무시하고, 그 밖의 옵션은 받는 옵션을 나열하는 에러입니다. 프런트엔드가 거부하는 헤더는 에러(exit 1)이고, 그 `file:line:col`에서 대신 무엇을 쓸지 알립니다(읽는 C의 범위는 [std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#bindgen이-읽는-c)).

`-x objective-c`(Apple 타깃)를 주면 헤더를 Objective-C로 읽고, 그 클래스, 프로토콜, 카테고리는 `ffi::objc::import`가 만드는 대로 `extern "objc"` 블록이 됩니다([std 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-헤더-ffiobjcimport)). 기본값은 `-x c`입니다.

`static inline` 함수는 링크할 심벌이 없어서 주석과 함께 건너뜁니다. `--c-shim <file.c>`를 주면 대신 바인딩합니다. `<file.c>`에는 `#include "<헤더 파일 이름>"` 뒤에 함수마다 C 래퍼 `R nabi_shim_<f>(A a0, ..) { return <f>(a0, ..); }`가 들어가고, 바인딩은 그 래퍼에 링크합니다(`#[link_name = "nabi_shim_<f>"]`). 패키지는 매니페스트의 `c.sources`로 `<file.c>`를 컴파일하고, 헤더의 디렉터리를 `c.include`에 둡니다(9.10.3). 가변 인자 인라인 함수는 계속 건너뜁니다.

### `nabi inspect`: 패키지 검사

`nabi inspect`는 아카이브의 내용을 보여 줍니다.

```
nabi inspect path/to/libfoo.a        # 또는 foo.nabihir
nabi inspect --dump path/to/foo.nabihir   # 또는 libfoo.a. 옆의 foo.nabihir 를 읽음
```

패키지 이름과 해시, nabic 버전, NABIHIR 버전, GC type id 구간, 내보낸 항목, [제네릭](../000-meta/007-glossary.md#generic--제네릭) 항목, 아카이브 멤버와 크기를 한 번에 표시합니다([9.9.14](909-debug-tools.md#9914-패키지-검사)).

`--dump`를 주면 대신 미리 빌드한 라이브러리의 NABIHIR을 텍스트로 출력합니다. 헤더 필드, producer, meta와 interface 파일을 출력하고, 포맷은 [NABIHIR 포맷](904-nabihir-bytecode.md)에서 정의합니다.

### `nabi demangle`: 심볼 디맹글링

`nabi demangle`은 맹글링된 심볼을 사람이 읽을 수 있는 형식으로 풀어 출력합니다.

```
nabi demangle [-v] <symbol>...
nabi demangle
```

심볼마다 소스에 쓰는 이름을 한 줄에 하나씩 출력합니다. `-v`를 주면 [9.5.4](905-symbol-mangling.md) 형식의 블록을 빈 줄로 나눠 출력합니다([9.9.3](909-debug-tools.md#993-심볼-디맹글링)). Nabi 심볼이 아닌 인자가 있으면 종료 코드는 `1`입니다. 심볼을 주지 않으면 표준 입력을 표준 출력으로 걸러, 텍스트 안의 Nabi 심볼을 모두 그 이름으로 바꿉니다(`perf report --stdio | nabi demangle`).

### `nabi completions`: 셸 자동완성

```
nabi completions bash|zsh|fish|powershell
```

그 셸의 자동 완성 스크립트를 표준 출력에 냅니다. 스크립트는 `nabi help`(위임하는 서브명령은 `nabic`의 도움말)와 같은 명령 / 옵션 텍스트에서 만듭니다. 그래서 둘은 어긋나지 않습니다. 완성하는 대상은 다음과 같습니다.

- 서브명령과 그 옵션
- 설치된 toolchain으로 `+<version>`과 `toolchain default` / `toolchain remove`의 인자
- `env use` / `remove` / `shell` / `export`와 `--env`의 환경 이름
- `nabic --print target-list`가 출력하는 이름으로 `--target`의 값

이름은 완성할 때 `nabi __complete toolchains` / `envs`(한 줄에 하나)를 실행해 읽으므로 `$NABI_HOME`을 따릅니다. 그 밖의 셸이거나 셸을 주지 않으면 사용법 에러(종료 코드 2)입니다. `install.sh`와 `install.ps1`이 스크립트를 설치하고, `nabi toolchain default` / `update`가 다시 씁니다(9.10.6).
