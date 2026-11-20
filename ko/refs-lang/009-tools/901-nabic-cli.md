# 9.1. `nabic` CLI 인터페이스

이 절은 컴파일러 `nabic`의 명령줄 인터페이스를 정의합니다. toolchain은 네 바이너리로 이루어집니다.

| 바이너리 | 역할                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `nabic`   | 컴파일러(rustc에 해당). 패키지를 빌드, 검사, 중간 산출물 출력하고 테스트와 벤치 하네스를 빌드함(이 절) |
| `nabi`    | cargo 식 프런트엔드. 매니페스트, 의존성, 환경, toolchain 고정을 맡고, `nabic`를 부르며, 컴파일러를 링크해 담은 채로 도구(실행, 린트, REPL, Jupyter, LSP, 포매터, 문서 등)를 실행함([9.10](910-nabi-frontend.md)) |
| `nabicc`  | C 컴파일러. `nabi cc`와 같은 컴파일러([9.13](913-c-compiler.md))                       |
| `nabils`  | 에디터용 LSP 서버 실행 파일. `nabi lsp`와 같은 서버                                    |

런타임은 `libnabir.a`(`nabir`)입니다. 별도 도구(`nabi-fmt` 등)는 두지 않습니다. `nabic`는 의존성을 해소하지 않습니다. 의존 패키지는 `nabi`가 `--extern`으로 넘깁니다([9.2.3](902-aot-pipeline.md#923-의존성-해소)).

## 9.1.1. 일반 형태

`nabic`는 다음 형태로 호출합니다.

```
nabic <subcommand> [options] [args]
```

문법 규칙으로 쓰면 다음과 같습니다.

```
cli_invocation  ::= "nabic" subcommand option* arg*
subcommand      ::= "build" | "emit" | "test" | "bench" | "check"
                  | "version" | "help"
option          ::= "--" identifier ("=" value)?
                  | "-" letter
```

`--name value`는 `--name=value`와 같습니다. 전역 옵션(9.1.5)은 서브명령 앞뒤 어디에나 둘 수 있습니다.

서브명령, 옵션, 옵션 값, 경로는 올바른 UTF-8이어야 합니다. 그렇지 않은 인자는 몇 번째인지 알려 주는 사용법 에러(9.1.6)입니다(``argument 3 (`b�c`) is not valid UTF-8; only a program's arguments, after `--`, may be other bytes``). `--` 뒤의 인자는 UTF-8이 아니어도 됩니다. `nabi run`은 시스템이 준 그대로 프로그램에 넘깁니다(Windows에서는 짝 없는 서로게이트가 섞인 UTF-16도 그대로, [9.10.8](910-nabi-frontend.md#nabi-run-빌드-후-실행)).

| 영역            | 서브명령                                              |
|-----------------|-------------------------------------------------------|
| 빌드            | `build`, `emit`, `test`, `bench`, `check`             |
| 그 밖           | `version`, `help`                                     |

도구(`run`, `lint`, `repl`, `jupyter`, `lsp`, `fmt`, `doc`, `clean`, `bindgen`, `inspect`, `demangle`, `completions`)는 `nabi`의 서브명령입니다([9.10.8](910-nabi-frontend.md#9108-도구)).

### 패키지 지정(`path`)

`build` / `emit` / `test` / `bench` / `check`(와 `nabi`의 도구)의 `[path]`는 다음 가운데 하나입니다. 생략하면 현재 디렉터리입니다.

| `path`                    | 패키지                                                                                                  |
|---------------------------|---------------------------------------------------------------------------------------------------------|
| 패키지 디렉터리           | 그 디렉터리. `src/main.nabi`나 `src/lib.nabi`가 없으면 위로 찾음(9.2.2)                                  |
| 패키지 `src/` 안의 파일   | 그 패키지. 파일에서 위로 올라가며 처음 만나는, `src/main.nabi` / `src/lib.nabi`가 있는 디렉터리                    |
| 그 밖의 파일              | 그 파일 하나로 된 익명 실행 패키지([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)). 패키지 디렉터리 안이지만 `src/` 밖에 있는 `.nabi` 파일도 포함 |

디렉터리를 주었는데 위로 올라가도 패키지가 없으면 에러입니다. 익명 패키지의 파일 서브모듈(`mod foo;`)은 그 파일 옆에 있습니다([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)).

패키지 이름(산출물 이름)은 패키지 디렉터리의 이름입니다. 익명 패키지는 파일 이름에서 `.nabi`를 뗀 것이 이름입니다.

## 9.1.2. 핵심 서브명령

### `nabic build`: [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 빌드

`nabic build`는 패키지를 AOT로 빌드합니다.

```
nabic build [path]
```

`[path]`는 위의 패키지 지정 규칙을 따릅니다. 산출물은 [9.2.15](902-aot-pipeline.md#9215-산출물)의 프로필별 `target/`에 놓입니다. `src/main.nabi`는 `target/<profile>/<name>`을 만들고, `src/lib.nabi`는 `target/<profile>/lib<name>.a`와 인터페이스 `target/<profile>/<name>.nabihir`([9.4](904-nabihir-bytecode.md))을 만듭니다. 둘 다 있는 패키지는 둘 다 만듭니다. 프로필은 `-O2` / `-O3` / `-Os`(`--release` 포함)이면 `release`이고, 그 밖에는 `debug`입니다. `--target=<target>`을 주면 `target/<target>/<profile>/`입니다. `--target-dir=<dir>`나 `NABI_TARGET_DIR`이 있으면 `target/` 대신 그 디렉터리를 씁니다.

단독 파일([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행))은 익명 패키지로 임시 디렉터리에 빌드하므로 파일 옆에 `target/`을 만들지 않습니다. 실행 파일은 파일 옆에 "파일 이름의 stem + 타깃의 실행 파일 접미사"라는 이름으로 복사합니다(`hello.nabi`면 `hello`, Windows에서는 `hello.exe`). `-o`로 다른 파일을 지정하면 그곳에 둡니다. 그 경로에 이 스크립트의 이전 빌드가 쓰지 않은 파일이 이미 있으면 그 파일은 건드리지 않습니다. 이때 빌드는 실패하고 `-o <file>`을 안내합니다.

옵션은 다음과 같습니다.

- `--release`는 `release` 프로필로 최적화 빌드(`-O2`와 같음)를 합니다. 기본값은 `debug` 프로필의 디버그 빌드(`-O0`)입니다. 두 프로필의 차이는 [9.2.15](902-aot-pipeline.md#9215-산출물)가 설명합니다.
- `--target-dir=<dir>`는 `target/`의 위치를 바꿉니다(`NABI_TARGET_DIR`과 같음).
- `--crate-type=lib|cdylib|staticlib|plugin`은 라이브러리 패키지의 산출물을 고릅니다. `lib`(기본값)은 정적 아카이브와 그 NABIHIR입니다. `cdylib`은 그 옆에 C 코드용 공유 라이브러리도 링크합니다. 파일은 `lib<name>.so`, `lib<name>.dylib`, 또는 `<name>.dll`과 `<name>.lib`입니다([9.3.3](903-package-archive.md#933-정적-라이브러리와-공유-라이브러리)). `-o`는 그 파일의 이름입니다. 라이브러리의 C 헤더 `target/<profile>/include/<name>.h`도 씁니다([9.3.3](903-package-archive.md#933-정적-라이브러리와-공유-라이브러리)). `staticlib`은 C, Swift, Xcode가 그것 하나만 링크하면 되는 정적 라이브러리 `static/lib<name>.a`(Windows는 `static/<name>.lib`, `-o`로 이름을 정함)를 헤더, `include/module.modulemap`과 함께 만듭니다([9.3.3](903-package-archive.md#933-정적-라이브러리와-공유-라이브러리)). wasm 타깃에서는 에러입니다. 그와 함께 쓰는 `--xcframework`는 그 타깃의 슬라이스를 `<profile>/<name>.xcframework`에도 넣습니다(Apple 타깃). `plugin`은 대신 Nabi 프로그램이 로드하는 플러그인을 링크하며, 파일 이름은 같고 `.lib`와 헤더는 없습니다([9.12](912-plugins.md)). 기기가 앱에 서명된 코드만 불러오는 `aarch64-apple-ios`(진행 중, [0.6.2](../000-meta/006-platform-targets.md#062-진행-중))에서는 `aarch64-apple-ios-sim`을 알려 주는 에러입니다. `src/lib.nabi`가 없는 패키지에서는 에러이며, `src/main.nabi`도 있는 패키지나 wasm, musl 타깃에서 `plugin`을 써도 에러입니다.
- `--lib-version <x.y.z>`는 `cdylib`의 버전을 정합니다. Linux에서는 soname이 `lib<name>.so.<x>`(1.0 미만이면 `lib<name>.so.0.<y>`)가 되고, 라이브러리 옆에 그 이름의 링크를 만듭니다. macOS에서는 install name의 현재 버전(`x.y.z`)과 호환 버전(`x.0.0`, 1.0 미만이면 `0.y.0`)을 정합니다. 없으면 soname은 파일 이름이고 두 버전은 모두 1.0.0입니다. `nabi build`는 패키지의 `version`을 넘깁니다([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)). 세 숫자가 아닌 값은 잘못된 옵션입니다.
- `--bundle=<dep>`(`--crate-type=plugin`과 함께)는 의존성 `<dep>`을 로드하는 프로그램에서 가져오는 대신 플러그인에 링크합니다.
- `--hot`은 `nabi dev`가 실행 중인 프로그램에 로드하는 핫 리로드 플러그인을 빌드합니다(`--crate-type=plugin`을 포함합니다). `src/main.nabi`만 있는 패키지는 루트 플러그인으로 빌드하고 `main`을 `<name>::main`으로 내보냅니다. 루트 파일이 둘 다 있는 패키지는 오류입니다. 핫 패키지 함수의 모든 호출은 핫 런타임이 다시 묶는 슬롯을 거치고, static은 첫 버전의 저장소에 묶이며, 타입은 전부 기술됩니다. 패키지 해시는 그 패키지의 모든 빌드에서 같으므로(이름으로 정합니다) 새 버전도 옛 버전의 이름을 그대로 씁니다. 컴파일러는 플러그인 옆에 `<출력 이름>.layout`도 씁니다. 버전 번호와 패키지 각 타입의 레이아웃입니다. `--hot`은 `-O0`으로만 빌드합니다. `--release`, `-O1` .. `-O3`, `-Os`는 오류입니다([8.4.13](../008-abi/804-safepoint.md#8413-핫-리로드-코드)).
- `--hot-base=<file>`(`--hot`과 함께)은 프로그램이 지금 돌리는 버전의 `.layout`을 가리킵니다. 새 버전은 그다음 번호를 받고 그 버전과 비교됩니다. 기본값 없이 추가한 필드, 타입이 바뀐 필드, 옛 필드를 가리키지 않는 `#[renamed_from]`, 바뀐 타입 매개변수나 종류는 오류(E0149)이며, 대신 무엇을 쓰면 되는지 알려 줍니다.
- `--hot-dep=<dep>`(`--hot`과 함께)은 의존성 `<dep>`도 핫 패키지라고 알립니다. 그 함수는 복사해 넣지 않고 그 패키지 플러그인의 슬롯으로 호출하며, 패키지 해시도 같은 방식으로 고정합니다.
- `--extern <name>=<dir | src/lib.nabi | <name>.nabihir | lib<name>.a>`는 의존 패키지 하나를 최상위 경로 `name`에 마운트합니다([9.2.3](902-aot-pipeline.md#923-의존성-해소)). 소스(`dir` / `src/lib.nabi`)는 같은 빌드에서 함께 컴파일합니다. 미리 빌드한 패키지(`.nabihir` / `.a`, 하나를 주면 옆의 짝을 찾음)는 NABIHIR을 읽고 아카이브를 링크합니다. 여러 번 쓸 수 있고, 보통 `nabi`가 채웁니다. `name`은 식별자이고, 패키지 이름의 `-`를 `_`로 바꾼 것입니다([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)). `std`, `core`, `bare`는 표준 라이브러리의 것입니다. 그런 이름의 라이브러리 패키지는 거부합니다. `--extern std=<path>`는 사용법 에러이지만, 앞의 `--extern`이 같은 경로를 가리키면 그 의존성의 다른 이름입니다(`--extern helper=p --extern std=p`. 코드는 `std::..`로 쓰고, `::std::..`는 여전히 라이브러리입니다). 키워드와 경로 시작점(`pkg`, `sup`)도 `name`으로 쓸 수 없고, 라이브러리나 라이브러리가 있는 패키지의 이름으로도 쓸 수 없습니다([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)).
- `--extern-feature <name>=<f>`는 소스 의존 패키지 `name`을 `cfg(feature = "<f>")`로 빌드합니다. feature마다 하나씩, 여러 번 줄 수 있습니다. 의존 패키지가 보는 feature는 이것뿐입니다. `--cfg`의 `feature` 키는 지금 컴파일하는 패키지에만 적용됩니다. 보통 `nabi`가 채웁니다.
- `--name <n>`은 패키지 이름([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)의 이름 규칙)을 정합니다. 산출물은 `lib<n>.a` / `<n>.nabihir`이고, 마운트 이름은 `-`를 `_`로 바꾼 것입니다. 기본값은 디렉터리 이름입니다. `--extern a-b=..`도 `a_b`로 묶습니다.
- `--bootstrap <file>`은 미리 읽을 표준 라이브러리 루트이고(여러 번 가능), `--nabir <lib>`는 링크할 런타임입니다. 기본값은 sysroot의 `lib/nabi/{core,std,bare}/src/lib.nabi`와 `lib/libnabir.a`입니다([9.2.19](902-aot-pipeline.md#9219-설치-레이아웃)). bootstrap 루트의 계층은 그 `src/` 위 디렉터리의 이름입니다. `bare` 루트 없이 `core/src/lib.nabi`를 주면 그 옆의 `bare/src/lib.nabi`도 읽습니다. 찾지 못하면 에러이고, 에러 메시지에 찾아본 경로를 나열합니다.
- `-O0` / `-O1` / `-O2` / `-O3` / `-Os`는 최적화 단계를 지정합니다([9.2.10](902-aot-pipeline.md#9210-최적화-수준)). `-O0`은 최적화 없음, `-O1`은 루프를 `-O2`처럼 최적화하되 그 밖에는 코드를 거의 늘리지 않고 디버그 정보를 넣는 수준, `-O2`는 속도, `-O3`는 코드를 더 써서 더 빠르게(루프 안에서 더 크게 인라인하고 스칼라 루프를 펼침), `-Os`는 크기를 위한 수준입니다(루프는 여전히 카운트 루프가 되고, 짧으면 벡터화됨).
- `--target=<target>`은 타깃 플랫폼을 지정합니다. [플랫폼 타깃](../000-meta/006-platform-targets.md#061-지원함)의 지원하는 타깃 가운데 하나인 내장 이름(진행 중인 타깃은 메인테이너의 테스트 스위치가 있을 때만, [0.6.2](../000-meta/006-platform-targets.md#062-진행-중))이나, `<name>.nabion` 타깃 사양 파일([0.6.5](../000-meta/006-platform-targets.md#065-타깃-사양))을 받습니다. 어느 호스트에서나 어느 타깃이든 빌드할 수 있습니다([0.2.4](../000-meta/002-pipeline.md#024-코드-생성)). 그 밖의 이름은 에러입니다. 진행 중이거나 지원하지 않는 타깃([0.6.2](../000-meta/006-platform-targets.md#062-진행-중), [0.6.3](../000-meta/006-platform-targets.md#063-지원-안-함))이면 그렇다고 알리고, 대신 쓸 것이 있으면 그것도 알립니다(``target `x86_64-apple-darwin` is not supported (Intel Macs); use `aarch64-apple-darwin` ``, ``target `aarch64-apple-ios` (iOS devices) is in progress: it is not tested on real hardware yet and comes in a later release; for now build for the simulator with `--target aarch64-apple-ios-sim` ``). 그 밖의 이름이면 가장 가까운 내장 타깃이 있을 때 그 이름을 댑니다(``unknown target `wasm32-wasi`; did you mean `wasm32-wasip2`?``).
- `-L <dir>`은 `#[link]`의 라이브러리 검색 디렉터리를 더합니다([5.9.9](../005-modules/509-extern-block.md#599-linklibname-라이브러리-명시)). `LIBRARY_PATH`보다 먼저 찾습니다. 여러 번 쓸 수 있습니다.
- `-l [static=|dylib=|framework=]<name>`은 라이브러리 `<name>`을 링크합니다(`#[link]`처럼 찾음). `framework=`를 붙이면 `#[link(name = "<name>", kind = "framework")]`처럼 Apple 프레임워크 `<name>`을 링크합니다. `-F <dir>`은 SDK보다 먼저 찾을 프레임워크 검색 디렉터리를 더합니다. `--native-runtime <file>`은 실행 파일 옆에 둘 공유 라이브러리를 지정합니다([9.2.14](902-aot-pipeline.md#9214-링킹실행-파일의-경우)). 모두 여러 번 쓸 수 있고, native 패키지나 시스템 라이브러리가 있으면 `nabi`가 넘깁니다([9.10.4b](910-nabi-frontend.md#9104b-네이티브-패키지)). Apple이 아닌 대상에서 프레임워크를 링크하면 링크 에러이고, help는 `extern` 블록을 `#[cfg(target_os = "macos")]` 아래에 두라고 합니다.
- `--native-object <file>`은 오브젝트나 아카이브를 그대로 실행 파일이나 공유 라이브러리에 링크합니다(여러 번 쓸 수 있음). 패키지의 C 소스를 컴파일한 오브젝트를 `nabi`가 넘깁니다([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)).
- `-o <path>` / `--output=<path>`는 출력 위치를 지정합니다.
- `--keep`은 빌드의 임시 디렉터리(오브젝트와 그 밖의 중간 파일)를 지우지 않고 남깁니다. 그 경로는 표준 에러에 알립니다(`nabic: kept temporaries in <dir>`).
- `--cfg <flag>` / `--cfg <key>=<value>`는 [조건부 컴파일](../006-meta-system/603-cfg.md)의 cfg 플래그를 설정합니다. 여러 번 쓸 수 있습니다.
- `--emit=<kind>[,<kind>..]`는 중간 산출물(`source`, `hir`, `mir`, `obj`, `asm`, `listing`, `expansions`, `c-header`)을 `target/<profile>/build/<name>.<ext>`에 씁니다([9.9.1](909-debug-tools.md#991-중간-산출물-출력---emit)). 쉼표로 여러 종류를 한 번에 지정할 수 있고, 이때는 `-o`를 함께 쓸 수 없습니다. `--emit`이 있으면 실행 파일이나 아카이브는 만들지 않습니다.
- `-g`는 `release` 프로필 빌드에 디버그 정보를 더합니다. 디버그 정보는 DWARF이고, Windows 타깃에서는 CodeView와 PDB입니다([9.9.6](909-debug-tools.md#996-디버거-통합-gdb--lldb)). `debug` 프로필 빌드에는 항상 있습니다. `-g`는 생성되는 코드를 바꾸지 않습니다. `-g`를 준 최적화 빌드의 기계어는 주지 않은 빌드와 같고, 디버그 정보가 최적화된 코드가 각 변수를 어디에 두는지 설명합니다([9.2.10](902-aot-pipeline.md#9210-최적화-수준)).
- `--remap-path-prefix <from>=<to>`는 빌드가 만드는 것(panic 위치, 디버그 정보, `.nabihir` 파일)에 디렉터리 `<from>` 아래 소스 파일의 경로를 `<to>` 아래인 것처럼 씁니다. 진단은 실제 경로를 그대로 보여 줍니다. 상대 경로는 작업 디렉터리 기준의 절대 경로로 맞춰 보고, 접두사는 경로 구성 요소 단위로만 맞으며, 나머지는 `/`로 `<to>`에 이어 붙입니다(`<to>`가 비어 있으면 상대 경로로 남습니다). 여러 규칙이 맞으면 마지막에 준 규칙이 이깁니다. 여러 번 쓸 수 있습니다. 표준 라이브러리의 소스는 항상 `/nabi/library` 아래(`/nabi/library/core/src/...`)로 쓰므로, 프로그램은 어느 기계와 설치에서나 같은 경로를 담습니다. 패키지의 `release` 프로필 빌드는 패키지 자신의 파일을 패키지 디렉터리 기준 상대 경로(`src/main.nabi`)로, 소스 의존 패키지의 파일을 그 이름 아래(`json/src/lib.nabi`)로도 씁니다. 그래서 같은 패키지를 두 곳에서 빌드해도 같은 바이트가 나옵니다. `--remap-path-prefix` 규칙은 이 규칙들 뒤에 오므로 이깁니다.
- `-C <option>`은 `--codegen <option>`으로도 쓸 수 있습니다.
- `-C overflow-checks[=yes|no]`는 고정 폭 정수 산술의 오버플로 검사(panic)를 켜거나 끕니다. 기본으로는 모든 프로필에서 켭니다. `nabi.nabion` 프로필로 끌 수 있고([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)), `-C overflow-checks`가 그 설정보다 우선합니다. 값을 생략하면 `yes`입니다. 오버플로가 일어날 수 없는 상수 피연산자 연산은 검사하지 않습니다.
- `-C icf[=yes|no]`는 링크할 때 똑같은 코드를 하나로 합치는 일(identical code folding)을 켜거나 끕니다([9.2.15](902-aot-pipeline.md#9215-산출물)). 기본으로는 `release` 프로필에서 켜고 `debug` 프로필에서 끕니다. 값을 생략하면 `yes`입니다.
- `-C target-cpu=<cpu>|native`는 트리플의 baseline 대신 CPU를 지정합니다([0.6.4](../000-meta/006-platform-targets.md#064-simd)). `-C target-feature=+f1,-f2,..`는 개별 기능(`+avx2` 등)을 켜거나 끄며, 각각 `+feature`나 `-feature`로 씁니다. 그 기능이 필요로 하는 기능도 함께 켜고(`+avx2`는 `avx`를 켭니다), 그 기능을 필요로 하는 기능도 함께 끕니다(`-avx`는 `avx2`를 끕니다). 코드 생성기는 `x86_64`에서 `avx2`, `popcnt`, `lzcnt`, `bmi`를, `aarch64`에서 `lse`, `rcpc`, `dotprod`를, `s390x`에서 `vector`, `vector-enhancements-1`을, `riscv64`에서 `v`를, `riscv64`와 `riscv32`에서 `zbb`를 쓰고, `armv7`의 `-neon`, `powerpc64le`의 `-vsx`, `wasm32`의 `-simd128`이면 벡터 루프를 만들지 않습니다([8.7.8](../008-abi/807-code-generation.md#878-루프-벡터화)). 그 어느 것도 바꾸지 않는 옵션을 주고 코드를 생성하는 빌드나 실행은 그렇다고 경고합니다. 모르는 CPU 이름은 아는 이름을 늘어놓는 에러입니다. `native`는 타깃이 빌드 기기의 플랫폼일 때만 받으며, 이 기기가 가진 기능을 더합니다.
- `-C subsystem=console|windows`는 Windows subsystem을 고릅니다. `console`(기본값, `mainCRTStartup`)이나 `windows`(GUI, `WinMainCRTStartup`, [9.5.9](905-symbol-mangling.md#959-main-함수의-심볼))를 받습니다.
- 그 밖의 `-C` 옵션은 잘못된 옵션(E0132, 아래)입니다: ``unknown codegen option `<option>` ``. 가장 가까운 옵션이 있으면 그 이름을, help에서 옵션 목록을 알립니다.

`nabic build`는 패키지의 린트 진단을 `nabic check`처럼 보고합니다([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). 경고는 출력하고 빌드를 이어 가지만, `deny` 린트는 빌드를 실패시킵니다. 이전 빌드 뒤로 바뀐 것이 없으면 `nabic build`는 `Fresh`를 출력하고, 그 빌드의 경고를 다시 출력한 뒤 빌드를 건너뜁니다([9.2.16](902-aot-pipeline.md#9216-증분-빌드)).

### `nabic emit`: 중간 산출물

```
nabic emit [path] [--emit=<kind>[,<kind>..]]
```

`nabic build --emit=..`와 같습니다. `--emit`을 생략하면 `mir`입니다.

### `nabic test`: 테스트

`nabic test`는 테스트 함수를 모아 테스트 하네스를 빌드합니다. 실행은 `nabi test`([9.10](910-nabi-frontend.md))가 합니다.

```
nabic test [path] [filter]
```

`#[test]`가 붙은 함수를 모아 테스트 실행 파일(하네스)을 빌드합니다. `#[test]`는 `#[cfg(test)]`를 함의하므로([6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-테스트-함수-마커)) 테스트 함수는 일반 빌드에 들어가지 않습니다. 테스트는 `src/`(`#[cfg(test)]` 아래)와 `tests/` 디렉터리([0.3.5](../000-meta/003-source-units.md#035-테스트와-벤치마크-디렉터리)) 양쪽에서 모읍니다. 테스트 빌드는 패키지의 진입점 대신 테스트를 고르는 진입점을 씁니다.

`nabic`는 하네스를 실행하지 않습니다. `--emit-harness <file>`은 빌드한 실행 파일과 그 안의 테스트(`nabi test`가 실행하는 매니페스트)를 `<file>`에 씁니다. 첫 줄은 매니페스트 형식인 `version 2`입니다. 러너는 더 새 버전을 거부합니다. ``the test harness is from a newer nabic (format version N); run the nabi of the same toolchain, e.g. `nabi +<version> test` ``. `--emit-harness`를 주지 않으면 빌드한 실행 파일을 표준 에러에 알립니다. 하네스 실행(테스트 선택, 아래의 프로세스 격리, 병렬 실행, 보고)은 `nabi test` 러너([9.10](910-nabi-frontend.md))가 맡습니다.

인자가 하나이고 그 인자가 존재하는 경로가 아니면 `filter`로 봅니다. `filter`는 모듈 경로를 포함한 테스트 이름(`math::adds`)과 맞춰 봅니다.

- `^` `$` `.` `*` `+` `?`와 `\` 이스케이프를 쓰는 작은 정규식입니다. 고정하지 않은 검색이므로 `^` / `$`가 없으면 이름의 어느 부분과 맞아도 됩니다.
- 이 문자들이 없으면 단순 부분 문자열 검색과 같습니다.

**각 테스트는 자기 프로세스에서 실행됩니다.** 테스트가 panic으로 끝나거나(종료 코드 101) 잡히지 않은 에러로 끝나면(종료 코드 1) 그 테스트만 실패하고, 나머지는 계속 실행됩니다. 시그널로 끝나도 실패이고, 시간 제한(`--timeout`, 기본 600초)을 넘긴 테스트도 실패입니다. 이 테스트는 강제 종료되고 timeout으로 보고됩니다. `#[should_panic]` 테스트는 panic할 때만 통과하고, `#[ignore]` 테스트는 실행하지 않고 ignored로 보고합니다([6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-테스트-함수-마커)).

#### 문서 테스트

라이브러리가 있는 패키지에서 `nabic test`는 문서 테스트도 빌드합니다. 대상은 라이브러리의 `///`, `//!` 문서 주석에 있는 fenced 코드 블록과, 패키지 루트 `README.md`의 ```` ```nabi ```` 블록입니다. 문서 주석에서는 언어를 적지 않은 fence도 Nabi입니다(`nabi doc`이 그렇게 보여 줍니다). 다른 언어의 블록(```` ```text ````, ```` ```nabion ````)은 테스트가 아닙니다. 빌드의 `#[cfg]`가 빼는 모듈 안의 블록이나, 빼는 항목에 붙은 블록(그 기능 없이 `#[cfg(feature = "table")]`)도 테스트가 아닙니다.

각 블록은 라이브러리에 의존하는 프로그램처럼 라이브러리를 쓰는 코드입니다. 라이브러리는 패키지 이름으로 붙은 의존성이고(`stats::mean(x)`), 각 블록은 `use stats::*;`가 있는 것처럼 시작하므로 라이브러리의 공개 아이템은 경로 없이 쓸 수 있습니다(`mean(x)`). 블록 앞쪽의 `use` 아이템은 모듈 수준에 남고, 나머지는 throw할 수 있는 함수의 본문이 되므로 `catch` 없이 `try`를 쓸 수 있으며, 그 안의 아이템은 지역 아이템입니다. `fn main`을 선언한 블록은 그 `main`을 호출합니다. fence에서 `nabi` 뒤에 `,`나 공백으로 구분해 적는 단어가 통과의 뜻을 바꿉니다.

| fence | 블록 |
|-------|------|
| ```` ```nabi ```` | 빌드하고 실행합니다. 반환하면 통과입니다 |
| ```` ```nabi,norun ```` | 빌드만 하고 실행하지 않습니다(파일을 읽거나, 기다리거나, 네트워크가 필요한 코드) |
| ```` ```nabi,should_panic ```` | panic할 때만 통과합니다 |
| ```` ```nabi,compile_fail ```` | 빌드되지 않을 때만 통과합니다. `compile_fail,E0040`은 그 에러 코드도 있어야 합니다 |
| ```` ```nabi,ignore ```` | 빌드도 실행도 하지 않고 ignored로 보고합니다 |

다른 단어는 쓸 수 있는 단어를 알려 주는 경고를 냅니다. `# `로 시작하는 줄(또는 `#`만 있는 줄)은 테스트에 들어가지만 `nabi doc`에는 보이지 않습니다. 읽는 사람에게 필요 없는 준비 코드에 씁니다.

````nabi
/// ```nabi
/// # let data = [1.0, 2.0, 3.0];
/// assert(stats::mean(data) == 2.0);
/// ```
````

문서 테스트의 이름은 `doc <file>:<line>`입니다. 파일은 패키지 기준 경로, 줄은 여는 fence의 줄이고, 블록이 아이템에 붙어 있으면 그 아이템이 뒤에 붙습니다: `doc src/stats.nabi:12 (mean)`, `doc README.md:20`. 필터, `--list`, 레코드는 이 이름을 다른 이름처럼 다룹니다. 한 패키지의 블록은 모두 실행 파일 하나로 빌드되어 라이브러리 자체 테스트 다음에 실행됩니다. `compile_fail` 블록은 하네스를 빌드할 때 하나씩 따로 검사합니다. 블록의 코드는 원래 줄 번호를 유지하므로, 블록 안의 에러는 문서가 있는 파일의 줄을 가리킵니다. 빌드되지 않는 블록은 그 블록의 케이스로 실패하고 에러는 실패 내용에 나옵니다. 나머지 블록은 그 블록 없이 빌드해 실행합니다.

`nabic test --doc`은 문서 테스트만, `--no-doc`은 문서 테스트를 빼고 빌드합니다. `nabi test`는 둘 다 그대로 넘깁니다.

옵션은 `nabic build`의 옵션과 `--emit-harness`입니다. 테스트를 몇 개씩 동시에 실행할지, 성공한 테스트의 출력을 보일지, 출력 형식은 러너인 `nabi test`의 옵션입니다([9.10.1](910-nabi-frontend.md#9101-서브명령과-전역-옵션)).

러너는 stdout에 다음 형태로 출력합니다.

```
running 3 tests
test math::adds ... ok
test math::subs ... FAILED
test util::trims ... ok

failures:

---- math::subs ----
<그 테스트의 stdout / stderr>
(panicked (exit 101))

failures:
    math::subs

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 filtered out; finished in 0.12s
```

ignored 테스트는 `test <name> ... ignored`로 보이고, 이유가 있으면 `, <reason>`이 붙습니다. 실패한 테스트가 하나라도 있으면 `nabi test`는 `1`로 종료합니다.

루트 파일이 둘 다 있는 패키지에서는 라이브러리의 테스트를 먼저 실행합니다. 그다음 실행 파일의 테스트를 실행하며, 이 테스트는 테스트를 뺀 라이브러리에 대해 빌드합니다. 마지막으로 문서 테스트를 실행합니다. 실행 하나가 실패하면 명령이 끝납니다. `bench`도 같은 방식입니다.

### `nabic bench`: 벤치마크

```
nabic bench [path] [filter]
```

`#[bench]`가 붙은 함수를 모아 최적화 빌드(`-O`를 주지 않으면 `-O2`)로 빌드합니다. `-O0`이나 `-O1`을 직접 주면 `nabic test`처럼 디버그 정보가 있는 `debug` 프로필로 빌드합니다. 하네스는 `nabi bench`([9.10](910-nabi-frontend.md))가 실행해 측정합니다([9.9.11](909-debug-tools.md#9911-벤치마크)). `#[bench]`는 `#[test]`처럼 `#[cfg(test)]`를 함의합니다. 벤치는 `src/`와 `benches/` 디렉터리([0.3.5](../000-meta/003-source-units.md#035-테스트와-벤치마크-디렉터리)) 양쪽에서 모읍니다. `path`, `filter`, `--emit-harness` 규칙은 `nabic test`와 같습니다.

벤치마크는 `Bench` 핸들을 받아 클로저를 측정합니다. 그래서 준비 코드는 측정 구간 밖에 둡니다.

```nabi
#[bench]
fn parse_1kb(b: bench::Bench) {
    let input = sample(1024);                   // setup: 한 번만, 측정 안 함
    b.set_bytes(1024);                          // 선택: 처리량 보고
    b.iter(|| parse(hint::black_box(input)));   // 측정 구간
}
```

`b.iter(f)`가 측정 전체(보정, 예열, 표본 수집)를 맡고, 결과는 black_box를 거칩니다. `hint::black_box`([`core::hint`](../../refs-standard-libraries/200-core/README.md#벤치마크--최적화-힌트))는 옵티마이저가 꿰뚫어 볼 수 없는 항등 함수입니다. 측정 구간의 입력과 결과에 적용하면 작업이 구간 밖으로 끌려 나가거나 상수로 접히지 않습니다. `b.set_bytes(n)` / `b.set_items(n)`은 처리량 열을 더합니다. `Bench`, `black_box`, 통계는 라이브러리 API이고, 측정 방법은 [9.9.11](909-debug-tools.md#9911-벤치마크)에서 정의합니다.

표본 하나의 호출 횟수, 표본 수, 출력 형식은 러너인 `nabi bench`의 옵션입니다([9.10.1](910-nabi-frontend.md#9101-서브명령과-전역-옵션)).

벤치는 하나씩 차례로 실행됩니다. 각 벤치는 터미널이든 아니든 사람이 읽는 단위로 정렬한 한 줄을 출력합니다.

```
  json::parse_1kb     1.84 us +/- 0.03 us   543 MiB/s   [2 outliers, 0 gc]
```

`nabi bench --format=nabibench`는 그 대신 회귀 도구가 읽는 안정된 기계용 형식을 벤치마다 한 줄씩 출력합니다. 형식은 `#nabibench name=<name> median_ns=<f> mad_ns=<f> samples=<n> outliers=<n> gc=<n> bytes=<n> items=<n>`입니다. 이 줄은 지금 형태로 고정됩니다. 키를 더하거나 빼거나 순서를 바꾸지 않으므로, 이 줄에 맞춰 쓴 파서는 계속 동작합니다. 새 필드는 `--format=json`의 JSON 레코드에만 들어갑니다. JSON 레코드는 벤치가 직접 출력한 내용과도 분리됩니다(`nabibench`에서는 그 내용이 이 줄과 같은 stdout에 섞입니다).

`b.iter`를 한 번도 부르지 않은 벤치는 아무것도 측정하지 않은 것입니다. 실패는 아닙니다. 그 줄은 `` <name>  warning: measured nothing: time the code with `b.iter(|| ..)` ``이고, 요약은 그 벤치를 실행한 수가 아니라 `<n> measured nothing`으로 셉니다.

### `nabic check`: 진단

```
nabic check [path | -] [--check-library]
```

프런트엔드를 타입 검사와 MIR 생성까지 돌리고, 린트와 코드 생성까지 돌리고 진단을 냅니다. 네이티브 빌드(오브젝트 파일, 링크)는 하지 않습니다. 옵션은 `nabic build`와 같습니다. `--check-library`는 도달 여부와 상관없이 라이브러리의 모든 본문을 낮추고 검사하며, 모든 제네릭 본체를 그 바운드에 맞춰 검사합니다([2.10.9](../002-types/210-generics.md#2109-본체-검사)). `nabic build --check-library`는 그 모두를 빌드의 최적화 수준으로 컴파일해 실행 파일에도 넣습니다(그 밖의 빌드는 프로그램이 닿는 라이브러리 코드만 컴파일함, [9.3.6](903-package-archive.md#936-표준-라이브러리)). 이때 린트는 표준 라이브러리 자체의 코드도 검사하고, `missing_docs`는 `deny`로 봅니다([6.2.9](../006-meta-system/602-built-in-attributes.md)). 루트 파일이 둘 다 있는 패키지는 먼저 라이브러리를 검사합니다. 라이브러리에 에러가 없으면 라이브러리를 함께 컴파일한 실행 파일을 검사합니다. 각 경고는 한 번만 보고합니다.

## 9.1.3. `--print` 옵션

### `nabic --print cfg`: 활성 cfg

```
nabic --print cfg [--target=<target>] [-O<n>] [--cfg ..]
```

옵션이 가리키는 빌드의 cfg([6.3](../006-meta-system/603-cfg.md))를 한 줄에 하나씩 출력합니다. 플래그는 `flag`, 키는 `key="value"` 형태입니다(rustc와 같음). 키에는 `target_env`가 있고, `debug_assertions`는 `-O0`에서 켜집니다.

```
debug_assertions
target_arch="aarch64"
target_endian="little"
target_family="unix"
target_os="macos"
target_pointer_width="64"
```

### `nabic --print target-list` / `target-spec`: 타깃

```
nabic --print target-list
nabic --print target-spec [--target=<target>]
```

`target-list`는 지원하는 타깃 이름을 한 줄에 하나씩 출력합니다. 진행 중인 타깃([0.6.2](../000-meta/006-platform-targets.md#062-진행-중))은 빠집니다. `target-spec`은 타깃의 사양을 NABION으로 출력합니다([0.6.5](../000-meta/006-platform-targets.md#065-타깃-사양)). `--target`이 없으면 호스트의 사양을 출력합니다.

### `nabic --print lldbinit`: lldb 포매터

이 toolchain의 lldb formatter([9.9.6](909-debug-tools.md#포매터))를 로드하는 한 줄 `command script import '<path>'`을 출력합니다. 경로는 `<sysroot>/lib/nabi/lldb/nabi.py`입니다. 로드하면 표준 라이브러리의 `/nabi/library` 경로도 toolchain의 라이브러리 소스로 연결합니다(`target.source-map`).

`--print gdbinit`은 gdb pretty-printer `<sysroot>/lib/nabi/gdb/nabi.py`를 로드하는 gdb 명령 `source <path>`를 출력합니다(라이브러리 경로는 `set substitute-path`로 연결). `--print natvis`는 natvis 시각화 파일 `<sysroot>/lib/nabi/natvis/nabi.natvis`의 경로를 출력합니다.

## 9.1.4. 보조 서브명령

### `nabic version`: 버전

```
nabic version
nabic --version            # = -V
nabic -V -v                # host 줄 추가
```

stdout에 `nabic <버전>` 한 줄을 출력합니다. git 체크아웃에서 빌드한 `nabic`는 그 뒤에 ` (<commit> <date>)`를 붙입니다. `-v`를 더하면 `host: <triple>` 줄을 덧붙입니다.

### `nabic help`: 도움말

```
nabic help [subcommand]
nabic <subcommand> --help
```

서브명령 없이 실행하면 전체 참조(모든 서브명령, 옵션, 환경 변수)를 출력합니다. 서브명령을 주면 그 서브명령만의 도움말을 출력합니다. 도움말에는 사용 형태, 짧은 설명, 예제 하나 이상, 받는 옵션이 들어갑니다. 어느 쪽도 다른 서브명령의 텍스트는 출력하지 않습니다.

## 9.1.5. 전역 옵션

전역 옵션은 다음과 같습니다. 서브명령 앞뒤 어디에나 둘 수 있습니다.

| 옵션                             | 의미                                                        |
|----------------------------------|-------------------------------------------------------------|
| `--verbose` / `-v`               | 단계별 시간 표시([9.9.5](909-debug-tools.md#995-로그--진행-표시)) |
| `-vv`                            | 더 자세한 로그. `Reading <file>`과 `Specialized <path>(sig)` 줄([9.9.5](909-debug-tools.md#995-로그--진행-표시)) |
| `--quiet` / `-q`                 | 진행 줄을 출력하지 않음(진단은 출력)                       |
| `--color=auto\|always\|never`    | stdout과 stderr의 색(9.1.8)                              |
| `--no-color`                     | `--color=never`와 같음                                     |
| `--cwd=<path>`                   | 작업 디렉터리를 바꾼 뒤 실행                                |
| `--print cfg\|target-list\|target-spec\|lldbinit\|gdbinit\|natvis` | 활성 cfg, 지원하는 타깃 이름, 타깃 사양, 또는 Nabi의 디버거 포매터(lldb, gdb, natvis)를 로드하는 것을 출력(9.1.3) |
| `--message-format=human\|json`   | 진단 형식. `json`은 stdout에 한 줄씩(9.1.8)              |
| `--explain <code>`               | 에러 코드(`E0040`)의 설명, 곧 뜻과 대신 쓸 코드를 출력하고 끝냄([9.9.4](909-debug-tools.md#에러-코드)) |
| `-A` / `-W` / `-D <lint \| group>` | 린트 수준(`--allow` / `--warn` / `--deny`). 패키지를 컴파일하는 모든 명령(`build`, `test`, `bench`, `emit`, `check`, 그리고 `nabi`의 `run`, `lint`, `repl`, `jupyter`, `lsp`)에 적용. 여러 번 쓸 수 있고 뒤의 것이 이김 |

`nabic help`는 `--print` 형식들을 빌드 옵션과 함께 보여 줍니다. 출력하는 내용이 `--target`과 `--cfg`에 따라 달라지기 때문입니다. 서브명령 없이도 씁니다.

린트 수준은 린트의 기본 수준 < `-A` / `-W` / `-D` < 코드의 `#[allow(..)]` / `#[warn(..)]` / `#[deny(..)]`(가장 안쪽이 이김) 순으로 정합니다. `nabi`는 매니페스트의 `lints`를 이 플래그로 바꿔 앞에 붙여 넘깁니다([9.10.1](910-nabi-frontend.md#9101-서브명령과-전역-옵션)).

색은 stdout과 stderr에 따로 다음 순서로 정합니다. `--color=never`면 끄고 `--color=always`면 켭니다. 그 밖(`--color=auto`, 기본값)에는 `NO_COLOR`가 비어 있지 않으면 끄고, `CLICOLOR_FORCE`가 비어 있지 않고 `0`도 아니면 켭니다. `TERM=dumb`면 끄고, 아니면 그 스트림이 터미널일 때 색을 씁니다. Windows에서 escape sequence를 받지 않는 콘솔에는 색 없는 글자를 보냅니다.

사용 예는 다음과 같습니다.

```
nabic --verbose build
nabic test -q
nabic --cwd=examples/hello check
```

## 9.1.6. 종료 코드

종료 코드는 다음과 같습니다.

| 코드  | 의미                                            |
|-------|-------------------------------------------------|
| `0`   | 성공                                            |
| `1`   | 사용자 코드 에러(컴파일 에러 / 테스트 실패 등) |
| `2`   | 사용법 에러(잘못된 옵션, 모르는 서브명령 등)   |
| `70`  | nabic 내부 에러(panic)                          |

사용법 에러는 `nabic: <메시지>` 다음에 ``run `nabic help <subcommand>` for usage`` 줄을 출력합니다(알려진 서브명령이 없으면 ``run `nabic help` for usage``). 메시지는 대신 쓸 것을 알려 줍니다. 모르는 서브명령이나 옵션은 가장 가까운 것이 있으면 그 이름을 댑니다(``unknown option `--relase`; did you mean `--release`?``). 파일을 가리키는(`/`를 담거나 `.nabi`로 끝나는) 모르는 서브명령에는 `nabic build <file>`을 제안합니다. `__NABI_CLI`가 `nabi`면(9.1.7) 두 줄 모두 `nabic` 대신 `nabi`라고 씁니다.

아는 옵션에 그 옵션이 받지 않는 값을 주면 진단 `E0132`([9.9.4](909-debug-tools.md#에러-코드))로 알리고, 종료 코드는 그대로 `2`입니다. 모르는 `-C` 옵션이나 `--emit` 종류, 값이 빠진 옵션, 모르는 `-C target-cpu`, 받지 않는 `--print`, `--color`, `--message-format` 값, 형식이 틀린 `-l`, `--extern`, `--extern-feature`, `--explain`에 없는 코드, `--emit` 종류 여럿과 함께 쓴 `-o`가 그렇습니다. `= help:` 줄이 무엇을 써야 하는지 알려 주고(``error[E0132]: unknown emit kind `mirr`; did you mean `mir`?``), 그 뒤에 `run .. for usage` 줄이 옵니다. 명령줄 어디에 있든 `--color`와 `--message-format`을 따라 출력합니다. `--message-format=json`이면 표준 출력에 JSON 한 줄로 나오고 그 줄은 없습니다.

sysroot는 `NABI_HOME`에서 오지 않고 실행 파일 기준입니다. `nabic`는 가져온 의존 패키지의 경로를 `$NABI_HOME/cache/` 아래로 보여 줄 때만 `NABI_HOME`을 읽습니다. nabic 자신이 panic으로 끝나면(internal compiler error) panic 메시지, `nabic: internal compiler error`, 그리고 `note:` 세 줄을 출력합니다. 세 줄은 nabi의 버그라는 것과 그 버그를 일으키는 코드를 어디에 보고하는지, 버전과 호스트(`nabic <version> (<commit> <date>) on <triple>`), 그리고 명령줄입니다. 컴파일러 패스 안이었다면 네 번째 줄 `note: in pass: <pass> > <pass>`가 그 패스들을 `NABI_TIME_PASSES`의 이름으로 알려 줍니다. 그런 다음 `70`(`EX_SOFTWARE`)으로 끝납니다. 프로그램의 panic은 `101`로 끝나므로 둘은 구별됩니다. `nabi`도 자신의 내부 에러를 같은 방식으로 알립니다(`nabi: internal error`, 종료 코드 `70`).

표준 출력이나 표준 에러가 읽는 쪽이 사라진 파이프이면 `nabic`는 메시지 없이 멈춥니다. Unix에서는 `SIGPIPE`로 종료되고(셸은 `141`을 보고), Windows에서는 `1`로 끝납니다. 이 경우는 내부 에러가 아닙니다.

## 9.1.7. 환경 변수

`nabic`의 동작에 영향을 주는 환경 변수는 다음과 같습니다.

| 변수              | 의미                                                                                     |
|-------------------|------------------------------------------------------------------------------------------|
| `NABI_CC`          | `nabic`의 링커 대신 마지막 링크를 하는 선택적 C 컴파일러 드라이버([9.2.14](902-aot-pipeline.md#9214-링킹실행-파일의-경우)) |
| `NABI_VERBOSE`     | `--verbose`와 같음                                                                      |
| `NABI_TIME_PASSES` | `1`이면 패스별 실제 시간을 stderr에 출력([9.9.5](909-debug-tools.md#nabi_time_passes)) |
| `__NABI_ENV`, `__NABI_ENV_LIBS` | `nabi`가 환경에서 스크립트를 실행할 때 설정하는(내부용), 그 환경의 이름과 그 환경에 설치된 라이브러리 목록(쉼표로 구분). 스크립트가 그중 하나를 선언 없이 쓰면 그 이름을 알리는 힌트를 받음 |
| `__NABI_CLI`       | `nabi`가 `nabic`를 실행할 때 `nabi`로 설정(내부용). 사용법 에러(9.1.6), `--help`, 에러와 경고 줄의 `nabic:` 접두사에서 `nabic` 대신 `nabi`라고 씀 |
| `NABI_TARGET_DIR`  | `target/`의 위치(기본값은 패키지 안)                                                     |
| `NABI_INCREMENTAL` | 프런트엔드가 패키지의 파싱 결과와 낮춘 본체를 두는 곳(기본값 `target/<profile>/incremental/`, `0`이면 두지 않음). 라이브러리에서 모은 항목은 `lib/nabi-cache/library/`에 둠([9.2.16](902-aot-pipeline.md#9216-증분-빌드)) |
| `NABI_INCREMENTAL_VERIFY` | `1`이면 모든 본체를 다시 낮추고, 결과가 다르면 실패([9.2.16](902-aot-pipeline.md#9216-증분-빌드)) |
| `NABI_FN_CACHE` | `0`이면 빌드가 보관하는 표준 라이브러리 코드, 곧 toolchain의 함수 캐시를 읽지도 쓰지도 않음([9.2.16](902-aot-pipeline.md#9216-증분-빌드)) |
| `NABI_CACHE_MAX_SIZE` | toolchain 캐시의 상한: 바이트, 또는 `K`, `M`, `G`를 붙인 수(기본값 `5G`). `off`이면 상한 없음([9.2.16](902-aot-pipeline.md#9216-증분-빌드)) |
| `NABI_HOME`        | 가져온 의존 패키지의 표시(9.1.6, 기본값 `~/.nabi`)                                        |
| `NABI_RUNNER`      | `<program> [args]`: `nabi run` / `test` / `bench`가 다른 타깃의 실행 파일을 실행할 프로그램([9.10.8](910-nabi-frontend.md#다른-타깃의-실행-파일)) |
| `LIBRARY_PATH`    | 네이티브 라이브러리 검색 디렉터리. `-L` 다음에 찾음                                      |
| `NO_COLOR`        | 비어 있지 않으면 `--color=auto`가 색을 쓰지 않음(9.1.5)                                |
| `CLICOLOR_FORCE`  | 비어 있지 않고 `0`이 아니면 스트림이 터미널이 아니어도 `--color=auto`가 색을 씀. `NO_COLOR`가 있으면 쓰지 않음(9.1.5) |
| `COLORTERM`       | `truecolor`나 `24bit`이면 24비트 색 값을 씀(9.1.8)                                        |
| `TERM`            | `dumb`이면 `--color=auto`가 색을 쓰지 않음. `256color`를 포함하면 256색 팔레트를 씀(9.1.8) |

설정하지 않았거나 비어 있는 변수는 기본값을 씁니다. 표에 없는 값을 주면 stderr에 경고 한 줄을 출력하고, 그 변수는 기본값을 씁니다. 경고에는 변수, 값, 받는 값이 나옵니다.

`nabi`는 내부 변수 `__NABI_CLI`, `__NABI_ENV`, `__NABI_ENV_LIBS`를 `nabic`에 넘깁니다. `nabic`는 이 변수들을 다른 철자로는 읽지 않습니다. `nabi run`, `nabi test`, `nabi bench`는 시작하는 프로그램의 환경에서 이름이 `__NABI`로 시작하는 변수를 모두 지웁니다(Windows에서는 이름의 대소문자와 상관없이). 프로그램은 나머지 환경을 그대로 받습니다.

## 9.1.8. 출력 형식

### 진단 메시지

진단 메시지는 다음 형식으로 출력합니다.

```
error[E0027]: cannot find value `count` in this scope
  --> src/main.nabi:5:13
   |
 5 |     let x = count + 1;
   |             ^^^^^
   |
   = help: did you mean `counter`?
```

진단은 [소스 파일](../001-lexical/101-source-files.md)에서 정의한 파일 + 줄 + 컬럼 단위를 그대로 씁니다.

머리줄은 `<severity>[<code>]: <message>`이고, `--> path:line:col`이 주 위치입니다. 주 span은 `^`로, 보조 span은 `-`로 밑줄을 긋습니다. 다른 파일의 보조 span은 `::: path:line:col`로 시작합니다. 끝의 `= note:` 줄은 맥락을, `= help:` 줄은 고치는 방법을 덧붙입니다. 메시지는 이 사양의 절을 인용하지 않습니다. 관련 절은 JSON 형식이 나열합니다([9.9.4](909-debug-tools.md#994-진단--에러-메시지)).

### 빌드 진행

진행 상황은 stderr에 오른쪽으로 정렬한 동사로 표시합니다. `-q`면 출력하지 않습니다.

```
   Compiling json v0.1.0
   Compiling foo (path/to/foo)
    Building target/debug/foo
    Finished in 1.23s
     Running `target/debug/foo`
```

| 동사        | 시점                                      |
|-------------|-------------------------------------------|
| `Compiling` | 프런트엔드 시작(패키지 이름과 디렉터리). `check`와 `lint`는 `Checking`, `doc`은 `Documenting`. `nabi`가 가져온 의존 패키지(`$NABI_HOME/cache/` 아래)는 대신 `<name> v<version>`을 보임. |
| `Building`  | 실행 파일 / 아카이브 생성(`build`)        |
| `Built`     | 단독 파일의 실행 파일을 그 옆에 복사함(`build`). `Built hello (use -o to put it elsewhere)`. 테스트 / 벤치 harness를 `--emit-harness` 없이 빌드함(`test`, `bench`). 뒤에 그것을 실행할 `nabi test`나 `nabi bench`를 알리는 note가 따라옴 |
| `Wrote`     | `--emit` 산출물 하나를 씀                 |
| `Reused`    | `-v`일 때 프런트엔드 뒤. 증분 캐시에서 가져온 파일 파싱과 본문 낮춤의 수([9.2.16](902-aot-pipeline.md#9216-증분-빌드)) |
| `Units`     | `-v`일 때 코드 생성 중. (증분 캐시나 함수 캐시에서) 재사용한 코드 단위와 컴파일한 코드 단위의 수 |
| `Finished`  | 빌드 끝(명령 시작부터 걸린 시간) |
| `Running`   | 프로그램 실행(`run`). 현재 디렉터리 기준의 실행 파일. 단독 파일은 파일 이름(`Running hello.nabi`), `-v`면 캐시된 실행 파일 |
| `Created`   | 패키지 생성(`new`). 그 아래에 이어서 실행할 명령을 보임([9.10.2](910-nabi-frontend.md#9102-nabi-new--nabi-init-패키지-생성)) |
| `Removed`   | `target/` 삭제(`clean`)                  |
| `Fresh`     | 최신. 그 빌드의 경고가 뒤따름             |
| `Checked`   | `check` 끝(에러와 경고 수)               |
| `Generated` | 문서를 씀(`doc`)                         |
| `Formatted` | 파일을 고쳐 씀(`fmt`). 끝에 그 수를 알림([`nabi fmt`](910-nabi-frontend.md#nabi-fmt-포매터)) |

프런트엔드에서 실패한 빌드(`build`, `run`, `test`, `bench`, `doc`)는 진단 뒤에 `error: could not build <name> (<n> error(s), <m> warning(s))`로 끝납니다. 수는 `check`가 세는 방식과 같습니다. `-q`와 `--message-format=json`에서는 이 줄을 생략합니다.

`nabi`도 자신의 진행을 같은 방식으로 출력합니다. `Added`, `Installed`, `Updated`, `Downloading` 등이 있습니다([9.10](910-nabi-frontend.md)).

### 색

색이 켜지면(9.1.5) 출력은 이 팔레트를 씁니다. 색 깊이는 터미널을 따릅니다. `COLORTERM`이 `truecolor`나 `24bit`면 24비트 값을 씁니다. 아니고 `TERM`이 `256color`를 포함하면 256색 팔레트를 쓰고, 그 밖에는 기본 16색을 씁니다.

| 부분                                  | 색                       | 24비트      | 256   | 16             |
|---------------------------------------|--------------------------|-------------|-------|----------------|
| 상태 동사(`Compiling`), 목록 표시 `*` | 하늘색, 굵게            | `#38bdf8`   | `81`  | bright cyan    |
| `error`                               | 빨강, 굵게               | `#ff5f5f`   | `203` | bright red     |
| `warning`                             | 호박색, 굵게             | `#e0b057`   | `179` | yellow         |
| `note`                                | 하늘색                   | `#38bdf8`   | `81`  | bright cyan    |
| `help`                                | 연분홍                   | `#f9a8d4`   | `218` | bright magenta |
| 진단의 거터(`-->`, `\|`, 줄 번호)     | 하늘색, 굵게             | `#38bdf8`   | `81`  | bright cyan    |

색만으로 뜻을 나타내지 않습니다. 라벨(`error:`, `warning:`, `note:`, `help:`)과 상태 동사는 색이 있든 없든 출력합니다. 항목 하나를 표시하는 목록(`nabi toolchain list`, `nabi env list`)은 그 항목 앞에 `*`를, 뒤에 `(default)`나 `(active)`를 붙입니다. 파이프로 보낸 출력, `--message-format=json`, `-q`는 영향을 받지 않습니다.

### JSON 출력(도구 통합용)

도구와 통합할 때는 JSON 형식으로도 출력할 수 있습니다.

```
nabic build --message-format=json
```

이 모드에서는 진단 하나가 stdout에 한 줄짜리 JSON 객체로 나옵니다. 그래서 IDE와 CI 도구가 바로 파싱할 수 있습니다. 객체의 모양은 [9.9.4](909-debug-tools.md#994-진단--에러-메시지)에서 정의합니다.

## 9.1.9. 표준 입력

`nabic check -`처럼 인자 자리에 `-`를 적으면 표준 입력을 뜻합니다. 파이프나 스크립트와 함께 쓸 때 씁니다. 표준 입력으로 받은 소스는 익명 실행 패키지의 `main.nabi`로 검사합니다.

## 9.1.10. 외부 도구와 호환

`nabic`는 저수준 컴파일러이고, 의존성 해소는 프런트엔드 `nabi`([9.10](910-nabi-frontend.md))이 맡습니다. `nabi`는 해소한 의존 패키지의 루트를 `--extern`([9.2.3](902-aot-pipeline.md#923-의존성-해소))으로 넘겨 `nabic`를 부릅니다. 다른 빌드 시스템도 같은 방식으로 `nabic`를 직접 호출해 통합할 수 있습니다. CLI 호환성을 깨는 변경은 SemVer 규칙을 따릅니다.
