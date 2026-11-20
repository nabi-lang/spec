# 9.2. AOT 빌드 파이프라인

이 장은 `nabic build`의 단계별 흐름을 정의합니다. [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)과 [컴파일 파이프라인](../000-meta/002-pipeline.md)에서 그린 큰 그림을 구체적인 명령과 파일 흐름으로 옮깁니다.

## 9.2.1. 단계

빌드는 다음 11단계를 거칩니다. 모든 단계가 `nabic` 안에서 실행됩니다.

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

> package discovery는 패키지 발견, dependency resolve는 의존성 해소, source parse는 소스 파싱, name / visibility는 이름 / 가시성 해소, type check는 타입 검사, nabi-hir build는 nabi-hir 생성, specialize는 단형화입니다. collect call-sites + monomorphize는 호출 지점 수집과 단형화, nabi-mir build는 nabi-mir 생성, code generation은 오브젝트 코드 생성, archive / link는 아카이브 생성이나 링크입니다. artifact는 산출물, executable은 실행 파일입니다.

각 단계의 입력, 출력, 책임은 분명히 나뉩니다.

## 9.2.2. 패키지 발견

`nabic build [path]`의 path에서 시작해 디렉터리를 한 단계씩 올라가며 `src/main.nabi`나 `src/lib.nabi`가 있는 디렉터리를 찾습니다. path가 파일이면 그 파일이 있는 디렉터리부터 찾습니다.

찾으면 그 디렉터리가 패키지 루트가 되고, 그 `src/`가 패키지의 소스가 됩니다([소스 단위](../000-meta/003-source-units.md) 참고). 그 밖의 경우는 다음과 같습니다.

- path가 찾은 패키지의 `src/` 밖에 있는 파일(패키지 소스 옆의 스크립트나 도구)이면, 그 파일 하나로 된 익명 실행 패키지입니다. 위로 올라가도 패키지가 없을 때도 같습니다([4.16.8](../004-functions/416-entry-point.md#4168-단일-파일-실행)). 그 파일의 서브모듈은 파일 옆에 있습니다.
- path가 디렉터리인데 패키지가 없으면 에러입니다.

패키지 이름은 패키지 디렉터리의 이름이고, 산출물 이름이 됩니다(9.2.15). 익명 패키지는 파일 이름에서 `.nabi`를 뗀 것이 이름입니다. 루트 파일이 둘 다 있는 패키지는 두 타깃으로 빌드합니다. 먼저 라이브러리를 빌드하고, 그다음 그 라이브러리를 의존 패키지 `<name>`으로 마운트하는 실행 파일을 빌드합니다([0.3.1](../000-meta/003-source-units.md#한-패키지의-라이브러리와-실행-파일)).

## 9.2.3. 의존성 해소

이 단계는 [경로](../005-modules/507-paths.md)와 [`use`](../005-modules/505-use.md)에 나오는 외부 패키지 참조(`use external_pkg::...`)를 처리합니다.

`nabic`는 의존성을 스스로 해소하지 않습니다. 프런트엔드 `nabi`([9.10](910-nabi-frontend.md))이 `nabi.nabion` / `nabi.lock`을 따라 path / git 의존 패키지를 준비하고, 각 패키지의 루트를 `--extern`으로 `nabic`에 넘깁니다.

```
nabic build --extern util=../util --extern parser=target/debug/deps/libparser.a
```

`--extern name=<경로>`는 의존 패키지 하나를 최상위 경로 `name`에 마운트합니다. `name`은 `nabi.nabion`의 `deps` 키에서 `-`를 `_`로 바꾼 식별자입니다. 경로는 두 형태입니다.

| 경로                              | 의존 패키지                                                                  |
|-----------------------------------|------------------------------------------------------------------------------|
| `<dir>` / `<dir>/src/lib.nabi`     | 소스. 그 `src/lib.nabi`를 같은 빌드에서 함께 컴파일                           |
| `<name>.nabihir` / `lib<name>.a`     | 미리 빌드한 패키지. 둘 중 하나를 주면 같은 디렉터리의 짝 파일을 찾음. NABIHIR([9.4](904-nabihir-bytecode.md))을 읽어 선언과 제네릭을 얻고 아카이브를 링크 |

- 사용자 코드는 `use name::..`로 그 패키지의 항목을 씁니다.
- 의존 패키지 안의 `pkg::`는 그 의존 패키지 자신을 가리킵니다.
- 의존 패키지는 별개의 패키지입니다. 바깥에서는 `pub` 항목만 쓸 수 있고, `pub(pkg)` / `pub(sub)` / 비공개 항목은 쓸 수 없습니다([5.7.14](../005-modules/507-paths.md#5714-패키지-사이-경로)).
- 사용자 패키지의 모듈이 의존 패키지와 같은 이름이면 에러입니다.
- 뒤에 오는 `--extern`의 경로가 앞의 것과 같으면 같은 패키지의 다른 이름입니다(의존 패키지를 서로 다른 이름으로 부를 수 있음, [9.10.2](910-nabi-frontend.md)). 패키지는 첫 이름으로 한 번만 빌드하고 마운트합니다. 다른 이름으로 시작하는 경로(`other::..`, 코드와 메타 블록 템플릿 모두)는 그 패키지를 가리키므로, 항목과 타입은 어느 이름으로 불러도 같습니다.

`src/lib.nabi`가 있는 패키지를 `nabic build`로 빌드하면 `lib<name>.a`와 `<name>.nabihir`이 나옵니다. `src/main.nabi`가 있으면 실행 파일 `<name>`이 나옵니다. 둘 다 있는 패키지는 셋을 모두 만들고, 실행 파일을 그 라이브러리에 링크합니다. `--name <n>`은 패키지 이름(산출물 이름과 마운트 이름)을 정합니다. 기본값은 디렉터리 이름입니다.

### 미리 빌드한 의존 패키지

`nabic`는 소스로 받은 `--extern`을 주어진 순서대로 **한 번씩** 미리 빌드해 `target/[<triple>/]<profile>/deps/`에 둡니다. 각 의존 패키지는 앞서 빌드한 의존 패키지를 모두 봅니다. 이후 컴파일은 소스 대신 이 `lib<name>.a`와 `<name>.nabihir`을 씁니다.

```
target/<profile>/deps/
├── lib<name>.a
├── <name>.nabihir
└── <name>.fp               # fingerprint
```

fingerprint가 같으면 다시 빌드하지 않습니다. fingerprint에는 다음이 들어갑니다.

- `nabic`의 정체 (NABIHIR producer와 같은 값)
- 빌드 옵션: `--release` / `-O*`, `-g`, `--target`, `--cfg` (`feature` 키 제외), `--bootstrap`
- 그 의존 패키지의 feature (`--extern-feature`)
- 그 의존 패키지의 `src/` 아래 모든 파일과 `nabi.nabion`
- 그보다 먼저 빌드한 의존 패키지의 fingerprint

미리 빌드한 패키지의 인터페이스는 소비하는 쪽의 feature가 아니라, 그 패키지를 빌드할 때 쓴 feature(그 NABIHIR meta, [9.4.6](904-nabihir-bytecode.md#946-meta))로 읽습니다.

컴파일하는 모든 명령이 이 경로를 씁니다. `build` / `run` / `test` / `bench` / `check` / `lint` / `doc` / `lsp` / `repl` / `jupyter`, 단일 파일 스크립트, 이름 있는 환경이 모두 해당합니다. REPL 호스트도 미리 빌드한 아카이브를 링크합니다. 패키지도 타깃 디렉터리(`--target-dir`, `NABI_TARGET_DIR`)도 없으면 시스템 임시 디렉터리의 단일 파일 캐시 아래에 둡니다(9.2.16).

표준 라이브러리는 빌드마다 소스에서, 프로그램이 닿는 부분만 컴파일합니다. 컴파일한 함수는 toolchain의 함수 캐시에 둡니다([9.3.6](903-package-archive.md#936-표준-라이브러리)).

## 9.2.4. 소스 파싱

컴파일러는 [소스 파일](../001-lexical/101-source-files.md)부터 [구두점과 연산자 토큰](../001-lexical/110-punctuation-and-operator-tokens.md)까지의 어휘 / 구문 규칙에 따라 모든 `.nabi` 파일을 AST로 파싱합니다.

[모듈](../005-modules/504-modules.md)에서 정의한 디렉터리 모듈은 그 안의 모든 .nabi 파일을 합쳐 한 모듈의 본체로 다룹니다.

패키지 루트의 매니페스트 `nabi.nabion`([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock))은 모듈 파일이 아닙니다. 매니페스트는 [NABION](../../refs-standard-libraries/300-std/308-nabion.md) 값 하나(`Package { .. }`)로 읽습니다. 최상위 문 규칙을 적용하지 않고 컴파일하지도 않습니다.

### scanner pre-pass

[`#[keyword]` 매직](../006-meta-system/606-keyword-magic.md)의 scanner pre-pass도 이 단계에서 실행합니다. 먼저 모든 .nabi 파일을 얕게 스캔해 `#[keyword(...)]`, `#[infix(...)]`, `#[literal(...)]` 등록 정보를 모읍니다. 그다음 그 정보를 정식 렉서와 파서에 넘깁니다.

## 9.2.5. 이름 / 가시성 해소

[`use`(가져오기)](../005-modules/505-use.md), [가시성](../005-modules/506-visibility.md), [경로](../005-modules/507-paths.md), [재내보내기(`pub use`)](../005-modules/508-re-export.md)의 규칙에 따라 모든 경로와 식별자를 항목에 연결합니다.

이 단계는 `core::prelude` 묵시적 가져오기를 적용합니다([표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md) 참고). `bare::*`와 `std::*`는 명시적으로 가져와야 하므로 여기서 빠집니다. 또 가시성을 검사해 허용되지 않는 접근을 에러로 보고하고, 가져오기 충돌도 검사합니다.

## 9.2.6. 타입 검사

[2. 타입 시스템](../002-types/README.md)의 규칙으로 모든 식의 타입을 결정합니다.

이 과정에서 각 슬롯을 점진 슬롯으로 둘지 정적 슬롯으로 둘지 정합니다([`any` 슬롯](../002-types/202-any-slot.md) 참고). 트레잇 바운드를 검사하고([제네릭](../002-types/210-generics.md) 참고), 분기의 결과 타입을 확정합니다([3. 표현식](../003-expressions/README.md) 참고). 합 타입과 `any` 슬롯의 좁히기도 여기서 처리합니다.

## 9.2.7. nabi-hir 생성

타입 검사가 끝난 AST를 nabi-hir([HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir)) 자료구조로 변환합니다.

HIR은 타입 정보를 많이 담은 중간 표현이고, 위 4~5단계의 결과를 모두 반영합니다. 구조를 압축해서 같은 타입과 식별자는 표현 하나를 함께 씁니다.

## 9.2.8. specialize: 단형화

[제네릭](../002-types/210-generics.md)과 [const 제네릭](../002-types/211-const-generics.md)을 단형화합니다.

```
1. 모든 호출 지점 수집.
2. 각 호출의 구체 타입 튜플 결정.
3. (item_id, type_tuple) 마다 별개의 specialized HIR 생성.
4. 캐시. 같은 (item_id, type_tuple) 는 한 번만.
```

프로그램 루트(`main`, 내보낸 심볼, `#[export_name]` 항목, 런타임이 참조하는 심볼)에서 닿을 수 없는 함수는 코드를 생성하지 않습니다. 라이브러리 빌드도 같습니다. 패키지의 `pub` 항목과 그 루트에서 닿지 않는 함수는 아카이브에 넣지 않습니다.

이 단계 덕분에 [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)는 vtable 없이 직접 호출할 수 있습니다([트레잇-as-타입 금지](../002-types/212-trait-as-type-banned.md) 참고).

## 9.2.9. nabi-mir 생성

[단형화](../000-meta/007-glossary.md#specialize)한 HIR을 nabi-mir(MIR)로 변환합니다.

MIR은 HIR보다 한 단계 낮은 IR입니다. 이 단계에서 연산자 문법 설탕을 모두 풀고([연산자 디스패치](../003-expressions/301-operator-dispatch.md) 참고), 패턴 매칭의 결정 트리를 정합니다([`match` 표현식](../003-expressions/312-match-expression.md) 참고). 클로저와 코루틴의 프레임 레이아웃도 확정합니다.

## 9.2.10. 최적화 수준

최적화 수준은 코드 생성 전에 nabi-mir를 어떻게 최적화할지 정합니다([8.7.3](../008-abi/807-code-generation.md#873-최적화-수준)). 코드 생성기는 어느 수준에서나 같습니다([0.2.4](../000-meta/002-pipeline.md#024-코드-생성)).

| 수준          | 효과 |
|---------------|------|
| `-O0`         | 최적화 없음. 모든 값이 자기 스택 슬롯에 있음 |
| `-O1`         | 값을 단순화해 레지스터에 둠. 루프는 `-O2`처럼 최적화함(루프 안의 호출을 `-O2`처럼 인라인하므로 `for` 루프의 반복자가 카운트 루프가 되고, strip mining과 벡터화를 거침). 그 밖에서는 작은 함수와 `#[inline]` 함수를, 어디서나 한 곳에서만 부르는 함수를 인라인함. 디버그 정보를 넣음(`debug` 프로필) |
| `-O2`         | 루프 밖에서도 더 큰 본문을 인라인함 |
| `-O3`         | 루프 안에서 더 큰 본문을 인라인하고, 스칼라로 남는 루프를 펼침. float 합을 쌓거나 함수를 부르는 루프는 strip 하나를 통째로 돌 때 카운터를 2회 또는 4회 반복마다 한 번만 검사함 |
| `-Os`         | 본문이 호출과 비슷하게 작을 때만 인라인함(루프 안에서는 그 몇 배까지). 루프는 여전히 카운트 루프로 만들어 strip mining하고, 본문이 짧으면 벡터화함 |

`-g`는 최적화도 코드도 바꾸지 않습니다(8.7.3). 디버그 정보가 최적화된 코드를 따라갑니다. 인라인된 본문은 프레임으로, 각 변수는 레지스터나 스택 슬롯에 있거나 optimized out으로 보입니다([9.9.6](909-debug-tools.md#996-디버거-통합-gdb--lldb)). 최적화기는 패키지를 codegen unit으로 나누기 전에 패키지 전체를 한 번에 봅니다(9.2.16). 그래서 모듈 사이에서도 인라인하고, 빌드마다 프로그램과 함께 컴파일하는 표준 라이브러리 함수 안팎으로도 인라인합니다([9.3.6](903-package-archive.md#936-표준-라이브러리)). 서로 다른 수준으로 빌드한 오브젝트도 같은 런타임 ABI([8](../008-abi/README.md))를 따르므로 함께 링크할 수 있고, 다른 수준으로 미리 빌드한 의존 패키지도 마찬가지입니다. REPL은 `:opt`로 정한 수준에서 셀을 컴파일하되([9.9.9](909-debug-tools.md#999-repl--jupyter-디버그)), 셀의 nabi-mir는 최적화하지 않습니다.

## 9.2.11. 코드 생성기

코드 생성기([8.7](../008-abi/807-code-generation.md))는 nabi-mir에서 바로 기계어와 오브젝트 파일을 만듭니다. 그 사이에는 모든 타깃이 함께 쓰는 저수준 형태 하나를 거칩니다. 대상은 `x86_64`, `aarch64`, i686, armv7, riscv64, riscv32, powerpc64le, s390x, wasm32입니다. `-g`를 주면 DWARF(함수, 줄 테이블, 매개변수, 지역 변수와 그 타입, 최적화된 코드에서는 인라인된 본문과 코드 범위별 변수 위치)를 냅니다([9.9.6](909-debug-tools.md#996-디버거-통합-gdb--lldb)). Windows 타깃에서는 같은 정보를 오브젝트의 CodeView(`.debug$S`, `.debug$T`)로 내고, 링커가 이를 PDB로 모읍니다(9.2.14). 코드 생성기가 아직 지원하지 않는 것을 쓰는 코드는 그것을 밝히는 `E0129` 에러입니다.

## 9.2.12. 오브젝트

모든 타깃의 오브젝트는 `nabic`가 씁니다. Apple 타깃은 Mach-O, Linux, FreeBSD, riscv32는 ELF, Windows는 COFF, wasm32는 wasm 오브젝트입니다. 빌드의 codegen unit(9.2.16)은 병렬로 컴파일하고, unit마다 오브젝트가 하나씩 나옵니다. `--emit=obj`는 링크하지 않고 오브젝트를 씁니다([9.9.1](909-debug-tools.md#991-중간-산출물-출력---emit)).

```
target/debug/build/foo.o
```

## 9.2.13. 아카이브

라이브러리 패키지는 codegen unit(9.2.16)마다 오브젝트를 하나씩 만들어, `nabic`가 직접 쓰는 표준 `ar` 아카이브에 넣습니다. 그 옆에는 인터페이스 파일 `foo.nabihir`([NABIHIR 형식](904-nabihir-bytecode.md))을 씁니다.

```
target/debug/libfoo.a
target/debug/foo.nabihir
```

## 9.2.14. 링킹(실행 파일의 경우)

실행 파일은 `nabic` 자신의 링커로 링크합니다.

```
foo.o + <deps>/lib*.a + <sysroot>/lib/libnabir.a + target libc  →  target/debug/foo
```

Mach-O 실행 파일은 macOS SDK(Xcode Command Line Tools)에 대해 링크합니다. Windows 실행 파일은 두 가지에 대해 링크합니다. 하나는 `nabic`가 함께 싣고 있는 module definition(mingw-w64에서 유래)으로 만든 import library이고, 다른 하나는 UCRT 위에 올린 Nabi 자체 startup입니다. MSVC 도구나 Windows SDK는 필요 없습니다. Nabi의 Windows startup은 `main` 전에 `.CRT$XCA`..`.CRT$XCZ`의 C initializer를 실행합니다. x86 Windows의 import 이름은 mingw-w64의 `lib32` 정의에서 오고 stdcall `@N` 장식을 씁니다. 이미지는 `/SAFESEH:NO`로 링크합니다. Linux 타깃은 native든 cross든 언제나 `<sysroot>/lib/nabi/targets/<triple>/`에 들어 있는 C 라이브러리에 대해 링크합니다. musl은 정적으로 링크하고, glibc는 link stub으로 링크합니다. 그래서 실행 파일에는 glibc 2.34 이상이 필요합니다. 다른 라이브러리는 `-L`과 `LIBRARY_PATH`로 찾습니다. 32비트 glibc 타깃의 동적 로더는 `/lib/ld-linux.so.2`(i686)와 `/lib/ld-linux-armhf.so.3`(armv7hf)입니다. riscv32 이미지(OS 없음, 진행 중)는 정적이며 C 라이브러리를 링크하지 않습니다. 진입점과 힙은 런타임이 갖춥니다([0.6.2](../000-meta/006-platform-targets.md#riscv32imac-unknown-none-elf)). `NABI_CC`를 설정하면 그 C 컴파일러 드라이버가 최종 링크를 대신합니다(`$NABI_CC <objects> <archives> -o <out>`). 이 대체는 선택 사항입니다. `--native-runtime`으로 준 공유 라이브러리는 실행 파일 옆에 복사하고, 실행 파일에는 rpath `@loader_path`(Mach-O) / `$ORIGIN`(ELF)을 넣습니다. Windows는 실행 파일이 있는 디렉터리에서 DLL을 읽습니다. 링커는 실행 파일에서 쓰지 않는 부분을 걷어 냅니다(dead-strip). 참조되지 않는 함수와 데이터를 지우고, `release` 빌드는 똑같은 함수와 똑같은 문자열 리터럴도 하나로 합칩니다([9.2.15](#9215-산출물)). Mach-O 실행 파일에는 ad-hoc 코드 서명이 들어갑니다. arm64 macOS는 서명이 있어야 실행 파일을 실행합니다. Apple 타깃에서 실행 파일에 디버그 정보가 있으면 `nabic`가 오브젝트의 DWARF를 모아 실행 파일 옆에 `.dSYM` 번들을 씁니다. Windows 타깃에서는 링커가 PDB(MSF 7.0)를 씁니다. PDB는 오브젝트들의 CodeView 타입을 합친 것, 오브젝트마다 심볼과 줄 테이블을 담은 모듈, global / public 심볼, 섹션 기여 정보를 담습니다. 실행 파일의 debug directory가 PDB를 가리킵니다(PDB의 GUID와 age를 담은 RSDS 레코드). `wasm32-wasip2` 실행 파일은 core wasm 모듈로 링크한 뒤, toolchain이 그 타깃용으로 싣는 preview 1 adapter로 WASI preview 2 component로 만듭니다. `ffi::wit` world([std 1.8](../../refs-standard-libraries/100-bare/108-wit.md))가 그 import와 export를 정합니다.

## 9.2.15. 산출물

산출물은 cargo처럼 **프로필별** 디렉터리에 놓입니다. 프로필은 `debug`(기본값)와 `release`입니다. 최적화 단계가 `-O2` / `-O3` / `-Os`(`--release` 포함)이면 `release`입니다.

두 프로필은 cargo와 같은 방식으로 다릅니다.

| Profile | 최적화 | 디버그 정보 | 링크 |
|---|---|---|---|
| `debug` | `-O0`, `-O1` | 있음 | 참조되지 않는 코드와 데이터 제거 |
| `release` | `-O2`, `-O3`, `-Os` | `-g`일 때만 | 똑같은 함수와 똑같은 문자열 리터럴도 하나로 합침. `-g`가 없으면 로더에 필요한 심볼만 남김 |

똑같은 코드 합치기(identical code folding)는 함수의 기계어와 재배치를 비교합니다. 두 함수의 바이트가 같고 모든 참조가 같은 곳을 가리키거나, 서로 합쳐지는 함수를 가리키면 하나로 합칩니다. 주소를 쓰는 함수(저장하거나 비교하거나 export하거나 테이블에 넣는 함수)는 합치지 않으므로, 서로 다른 함수는 계속 서로 다른 주소를 가집니다. 호출되거나 점프로만 닿는 함수와, 함수 값이 호출하는 썽크(`<symbol>.v`, [8.11](../008-abi/811-backend-contract.md))만 합칩니다. 함수 값은 비교할 수 없으므로 두 함수 값이 코드를 나눠 써도 프로그램이 알아챌 수 있는 것은 달라지지 않습니다. 합쳐져 사라진 사본의 unwind 항목과 디버그 정보는 지워진 코드처럼 빠지고, 디버거, 프로파일러, backtrace는 남은 사본의 이름을 보여 줍니다. 바이트가 같은 문자열 리터럴은 사본 하나를 같이 씁니다(그 주소에는 뜻이 없습니다). `release` 빌드에서는 `-C icf=no`로 끄고, `debug` 빌드에서는 `-C icf`로 켭니다([9.1](901-nabic-cli.md)).

디버그 정보는 빌드가 만드는 실행 파일이나 아카이브(`build`, `run`, `test`, 그리고 `-O0` / `-O1`을 직접 준 `bench`)에 들어갑니다. `--emit` 출력이나 REPL, Jupyter 셀에는 들어가지 않습니다. panic은 어느 프로필에서든 소스 위치를 알립니다. 빌드의 오브젝트는 출력 옆 한 곳에서 링크하므로, 입력이 같으면 같은 실행 파일이 나옵니다. 오버플로 검사는 두 프로필 모두에서 켜지고, `nabi.nabion`의 `profile`로 한 프로필에서 끌 수 있습니다([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)). 그 밖의 프로필 설정은 없습니다.

```
target/
├── debug/                      # profile (기본)
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

| 산출물          | 경로                                                              |
|-----------------|-------------------------------------------------------------------|
| 실행 파일       | `target/<profile>/<name>`                                         |
| 라이브러리      | `target/<profile>/lib<name>.a`                                    |
| `--emit=<kind>` | `target/<profile>/build/<name>.<ext>` (`ast.nabi` `hir` `mir` `o` `s` `lst` `expanded.nabi`) |
| 중간 산출물     | `target/<profile>/build/`                                         |
| 의존 패키지     | `target/<profile>/deps/` (`lib<dep>.a`, `<dep>.nabihir`, `<dep>.fp`, 9.2.3) |
| 교차 빌드       | `target/<triple>/<profile>/...` (위와 같은 구조)                  |
| 문서            | `target/doc/`                                                     |
| 증분            | `target/<profile>/incremental/<name>/` (실행 파일의 codegen unit), `.../lib<name>/` (라이브러리의 것), 타깃 루트마다의 `.../<name>-<hash>/` (프런트엔드), 9.2.16 |

루트 파일이 둘 다 있는 패키지에는 산출물이 두 개 있습니다. `target/<profile>/<name>`과 `target/<profile>/lib<name>.a`(+ `<name>.nabihir`)입니다. `-o`는 실행 파일의 이름을 정합니다.

이름 `foo`는 패키지 이름입니다(9.2.2). `-o`는 산출물 하나의 경로를 바꿉니다. `--target-dir=<dir>`와 환경 변수 `NABI_TARGET_DIR`은 `target/` 자체의 위치를 바꿉니다. `nabi clean`은 `target/` 전체를 지웁니다. 라이브러리의 NABIHIR은 아카이브 옆의 `foo.nabihir`입니다([9.4](904-nabihir-bytecode.md)).

## 9.2.16. 증분 빌드

증분 빌드는 세 수준에서 동작합니다. 첫째, 입력이 바뀌지 않은 패키지는 빌드하지 않습니다. 둘째, 빌드하는 패키지는 바뀌지 않은 파일의 파싱 결과와 바뀌지 않은 함수의 낮춘 본체를 지난 빌드에서 가져옵니다(아래 프런트엔드). 셋째, 코드는 codegen unit 단위로 컴파일하고, 바뀐 unit만 다시 컴파일합니다. `nabic build` / `nabi run`은 이전 빌드의 fingerprint가 지금과 같으면 `Fresh`를 출력하고 빌드를 건너뜁니다. fingerprint에는 다음이 들어갑니다.

- 컴파일러의 정체(NABIHIR producer와 같은 값), 빌드 옵션, 진단 형식(`--message-format`, 색), 입력 / 출력 경로, 설정되어 있으면 `NABI_CC`
- 패키지와 의존성의 `src/` 아래 파일, 그 `nabi.nabion`, 그리고 빌드가 읽은 그 밖의 모든 파일(`src/` 밖의 `#[path]` / `include_str!`)
- `include_str!` / `include_bytes!`([6.11](../006-meta-system/611-include.md))가 읽는 파일과 그 내용
- 린트 수준(`-A` / `-W` / `-D`, 그리고 이 옵션을 거친 매니페스트의 `lints`)
- toolchain이 그 타깃용으로 배포하는 런타임 라이브러리, C 런타임 오브젝트, C 라이브러리([9.2.19](#9219-설치-레이아웃))와 미리 빌드한 의존성의 `.a`. 각 파일은 크기, 수정 시각, 파일 정체(inode와 상태 변경 시각, Windows에서는 생성 시각)로 비교하므로, 수정 시각을 유지한 복사도 변경으로 봄
- 산출물의 크기, 수정 시각, 파일 정체

빌드가 낸 경고는 린트 경고와 백엔드 경고까지 기록해 둡니다. `Fresh`인 빌드는 그 경고를 다시 출력합니다.

native 빌드든 교차 빌드든 모든 빌드는 프로그램이 닿는 표준 라이브러리 함수를 컴파일합니다([9.3.6](903-package-archive.md#936-표준-라이브러리)). 그 최적화한 코드와 unit의 오브젝트는 단독 파일 빌드를 포함한 toolchain의 모든 빌드를 위해 toolchain의 함수 캐시 `lib/nabi-cache/fn/`(9.2.19)에 둡니다.

- 항목의 키는 결과가 의존하는 것을 모두 담습니다. 최적화한 코드는 그 단계 전의 함수 코드, 최적화 수준, 타깃, 그 단계가 읽는 피호출 함수의 사실이며 모두 이름으로 셉니다. 오브젝트는 unit의 키(아래)입니다. 둘 다 컴파일러의 정체를 담습니다. 표준 라이브러리의 낮춘 본문은 패키지가 자기 본문과 함께 보관합니다(아래 프런트엔드). 빌드는 키가 맞는 항목을 가져오고 자기가 만든 항목을 씁니다. 출력은 캐시가 빈 빌드가 만드는 것과 같습니다.
- 항목은 파일을 다 쓴 뒤 제자리로 이름을 바꿔 넣으므로, 동시에 도는 빌드도 캐시를 안전하게 함께 씁니다.
- 빌드는 하루에 많아야 한 번, 30일 동안 어떤 빌드도 읽지 않은 항목을 지웁니다. 더 지우려면 `nabi cache prune`을 씁니다(9.10.6). 캐시 디렉터리가 읽기 전용이고 비어 있지 않은 toolchain(빌드할 때 캐시를 채워 둔 샌드박스 이미지)은 항목을 읽기만 하고 쓰지 않습니다.
- 캐시에는 함수 캐시와 라이브러리 단계(아래)를 합한 상한 `NABI_CACHE_MAX_SIZE`가 있습니다(바이트, 또는 `K`, `M`, `G`를 붙인 수. 기본값 `5G`. `off`이면 없음). 항목을 쓴 빌드는 그 크기를 캐시 디렉터리의 누계에 더합니다. 누계가 상한을 넘으면 한 번에 한 빌드가 캐시 크기를 재고, 상한을 넘었으면 가장 오래전에 쓴 항목부터 지워 상한의 80% 이하를 남깁니다. 항목을 쓴 때는 마지막으로 쓰거나 읽은 때이며, 많아야 한 시간에 한 번 기록됩니다. 지난 한 시간 안에 쓴 항목과 아직 쓰는 중인 파일은 이렇게 지우지 않으므로, 같은 때 도는 빌드는 쓰는 것을 지킵니다. 그런 항목 때문에 캐시가 상한의 80%를 넘게 남으면, 다음 측정은 상한의 5분의 1만큼이 더 쓰일 때까지 기다립니다. 지운 항목은 그것이 필요한 다음 빌드가 다시 계산합니다.
- `NABI_FN_CACHE=0`이면 캐시를 읽지도 쓰지도 않습니다.

### 프런트엔드

패키지 빌드(`check`, `build`, `run`, `test`, `bench`, `doc`, `lint`)는 프런트엔드가 계산한 것을 `target/<profile>/incremental/<name>-<hash>/`에 둡니다. `<hash>`는 패키지 루트를 가리킵니다. 루트 파일이 둘 다 있는 패키지는 타깃마다 자기 디렉터리가 있고, 그 `<hash>`가 각 루트를 가리킵니다. 빌드 종류(타깃, cfg, 테스트 여부, 어떤 라이브러리를 미리 빌드했는지)마다 하위 디렉터리가 하나씩 있습니다. 같은 종류의 다음 빌드는 여기서 다음을 가져옵니다.

- 텍스트와 파싱 입력(빌드의 확장 연산자, 패키지의 키워드, 메타 블록과 `#[literal]` 패턴, cfg)이 바뀌지 않은 각 파일의 파싱 결과.
- 각 plain 함수 본체를 낮춘 결과(이름 해소, 타입 검사, 제네릭 인스턴스화). plain 함수는 제네릭도 `const`도 아닌 함수이거나, 제네릭이 아닌 `impl`의 메서드입니다. 재사용하려면 본체 자신의 텍스트와 위치가 같고, 프로그램의 인터페이스가 같아야 합니다. 낮출 때 읽은 모든 시그니처, 인스턴스, 그 밖의 사실도 지금과 같아야 합니다. 또 그 결과가 본체 밖에서 가리키는 모든 위치 앞의 텍스트가 같아야 합니다.
- 인터페이스는 반환 타입을 적었거나 `()`인 plain 본체, 그리고 plain 함수의 매개변수와 반환 타입을 뺀 모든 항목입니다. 반환 타입을 추론하는 본체는 인터페이스에 속합니다. 호출자가 그 타입을 읽기 때문입니다. 본체를 고치면 그 본체를 다시 낮춥니다. 반환 타입을 추론하는 본체를 고치면 그 타입을 읽은 본체도 다시 낮춥니다. plain 함수의 시그니처를 고치면 그 함수와, 그 시그니처를 읽은 본체를 다시 낮춥니다. 항목의 그 밖의 변경(타입, 트레잇, `impl` 머리, 제네릭 함수, `use`)은 모든 본체를 다시 낮춥니다.
- 재사용한 결과가 만든 함수, 타입, 인스턴스는 깨끗한 빌드와 같은 번호를 받습니다. 진단, 프로그램, 심볼도 깨끗한 빌드와 같습니다.

시그니처, `impl` 등록, coherence, 프로그램 전체 검사는 모든 빌드에서 실행합니다. 라이브러리 항목을 모으며 만든 것(표준 라이브러리와 의존 패키지의 것)과 라이브러리 파일의 pre-pass 파싱 결과는 toolchain 캐시(`lib/nabi-cache/library/`)에 보관합니다. 캐시 파일은 toolchain, 빌드 플래그, 라이브러리 소스의 해시마다 하나이고, 단독 파일 빌드를 포함해 모든 빌드가 이 캐시를 씁니다. `NABI_INCREMENTAL`은 이 캐시를 옮기지도 끄지도 않습니다. 보관한 것이 꼭 있어야 하지는 않습니다. 없거나 망가졌거나 오래된 항목은 다시 계산합니다. 패키지 밖의 단독 파일은 `NABI_INCREMENTAL`이 디렉터리를 가리킬 때만 자기 파싱 결과와 본체를 보관합니다. `NABI_INCREMENTAL=<dir>`은 결과를 `<dir>/<name>-<hash>/`에 두고, `NABI_INCREMENTAL=0`은 보관을 끕니다. `NABI_INCREMENTAL_VERIFY=1`은 모든 본체를 따로 다시 낮추고, 재사용한 결과가 다르면 빌드를 실패로 끝냅니다. `-v`는 파싱 결과와 본체를 재사용한 파일 수를 보고합니다.

### 코드 생성 단위

`Fresh`가 아닌 빌드는 컴파일한 코드를 codegen unit 단위로 재사용합니다. nabic는 패키지의 코드(실행 파일이든 라이브러리든)를 unit으로 나눕니다. 패키지의 모듈마다 하나, 그 함수나 인스턴스를 컴파일하는 표준 라이브러리와 의존 패키지의 모듈마다 하나(인스턴스는 타입 인자에 따라 여러 unit에 나뉠 수 있음), 그리고 프로그램에 한 번만 있는 것(static, C `main`, GC 타입 등록)을 위한 unit이 하나입니다.

- unit은 같은 타깃 디렉터리에 출력한 지난 빌드 이후 키가 바뀌었을 때만 컴파일합니다. 키가 그대로면 `target/<profile>/incremental/<name>/`(라이브러리는 `lib<name>/`)에 둔 결과를 재사용합니다. 그 뒤 모두 링크합니다. 표준 라이브러리의 unit은 그 대신 모든 패키지의 빌드가 쓰는 함수 캐시(위)에 둡니다. 자기 unit을 보관하지 않는 단독 파일도 함수 캐시는 씁니다. 미리 빌드하는 의존 패키지(9.2.3)는 unit 없이 오브젝트 하나로 컴파일합니다.
- 익명 파일의 unit은 패키지 없이 쓰는 타깃 디렉터리(`--target-dir`, `NABI_TARGET_DIR`, 없으면 단일 파일 캐시)의 `[<triple>/]<profile>/incremental/<stem>-<경로의 hash>/`에 둡니다. 단일 파일 캐시는 Unix에서 `<시스템 임시 디렉토리>/nabic-target-<uid>/`(`<uid>`는 유효 사용자 ID)이고, Windows에서는 `%TEMP%\nabic-target\`입니다. Windows의 임시 디렉터리는 이미 사용자마다 따로 있습니다. 이 디렉터리는 캐시입니다. nabic는 하루에 많아야 한 번, 일주일 동안 어떤 빌드도 쓰지 않은 파일을 지웁니다.
- 단일 파일의 실행 파일도 같은 타깃 디렉터리의 `[<triple>/]<profile>/scripts/<key>/`에 두고, 키가 맞는 동안 재사용합니다. 하루 동안 쓰지 않은 항목은 다른 항목을 빌드할 때 지웁니다([9.10.7](910-nabi-frontend.md#9107-단일-파일-스크립트의-인라인-매니페스트)). `nabi run`은 그것을 실행하고, `nabic build`는 파일 옆에 복사합니다. 스크립트에는 `nabi`가 `$NABI_HOME/cache/scripts/<hash>/` 아래의 스크립트 전용 `target/`을 `--target-dir`로 넘깁니다. 나머지는 단일 파일 캐시에 둡니다. `nabic`가 직접 빌드한 단일 파일과, 디렉터리의 `nabi.nabion` 때문에 단일 파일 패키지가 된 파일입니다(9.10.1).
- 단일 파일 캐시에는 실행하는 실행 파일이 들어 있으므로, Unix에서는 이 디렉터리를 사용자 혼자만 쓰게 둡니다. 컴파일러는 모드 `0700`으로 만들고, 쓰기 전에 심볼릭 링크가 아닌 디렉터리인지, 유효 사용자의 것인지, 그룹과 다른 사용자의 권한 비트가 없는지 확인합니다. 아니면 명령은 종료 코드 `1`과 ``nabic: error: <dir> is not private to you (owner uid N, mode 0755); remove it or set NABI_TARGET_DIR``로 실패합니다. `--target-dir`와 `NABI_TARGET_DIR`는 주어진 그대로 씁니다.
- unit의 키에는 그 함수들의 코드, 다른 unit에서 가져다 쓰는 것의 시그니처와 코드 생성 정보, 쓰는 static과 GC 타입, 타깃, 최적화 수준, 코드 생성 옵션, 컴파일러의 정체가 들어갑니다. `-g`이면 디버그 정보가 가리키는 소스 파일의 경로도 들어갑니다. 그래서 함수의 시그니처가 바뀌면 호출자의 unit을 다시 컴파일하고, 파일을 더하면 그 파일을 가리키는 unit만 다시 컴파일합니다.
- 심볼과 GC type id는 다른 unit의 내용에 의존하지 않습니다. unit은 GC type id를 패키지의 공유 unit이 정의한 심볼에서 읽습니다([9.5.7](905-symbol-mangling.md#957-충돌-회피-typehash의-안정성)). 그래서 산출물은 깨끗한 빌드가 만드는 것과 같습니다.
- 최적화 빌드도 unit 사이에서 인라인합니다. 패키지를 나누기 전에 통째로 최적화하기 때문입니다(9.2.10).
- `-v`는 재사용한 unit과 컴파일한 unit의 수를 보고하고, `-vv`는 컴파일한 unit의 이름을 보고합니다.

## 9.2.17. 컴파일러와 런타임의 분리

`nabic`는 서드파티 크레이트를 하나도 쓰지 않고, 외부 코드 생성기도 링크하지 않습니다.

파이프라인은 컴파일러와 런타임을 떼어 둡니다. `nabic`는 빌드할 때 런타임(`libnabir.a`)을 참조하지 않습니다.

대신 `extern "C"` 심볼([Extern "C" 심볼](../008-abi/805-extern-symbols.md))을 호출하는 코드만 생성합니다. 링크할 때 사용자 빌드가 `libnabir.a`를 함께 링크합니다.

이렇게 나누어 두었으므로 `nabic`와 `libnabir.a`는 서로 독립된 SemVer로 발전할 수 있습니다.

## 9.2.18. 자기 완결적 toolchain

`nabic`는 자기 완결적입니다. 코드 생성기와 링커가 자기 것이고([0.2.4](../000-meta/002-pipeline.md#024-코드-생성)), 외부 컴파일러, 어셈블러, 링커를 실행하지 않습니다. 크로스 컴파일도 내장되어 있습니다. toolchain에는 지원하는 Linux 타깃의 C 라이브러리(musl, glibc link stub)가 함께 들어 있습니다.

| 시스템에서 필요한 것   | 타깃                                     |
|------------------------|------------------------------------------|
| macOS SDK (Xcode Command Line Tools, 재배포 불가) | Apple 타깃     |
| 없음 (제공된 C 라이브러리) | Linux 타깃                      |
| 없음 (제공된 module definition으로 만든 import library, UCRT) | Windows 타깃 |
| `NABI_CC` (선택)        | 최종 링크를 대체 (9.2.14)                |

C 컴파일러 `nabicc`에는 타깃의 C 헤더도 필요합니다. toolchain은 지원하는 Linux와 wasm32 타깃의 헤더를 담고 있고, Apple 타깃은 macOS SDK에서, Windows 타깃은 Windows 호스트의 Windows SDK와 Visual Studio에서 헤더를 가져옵니다([9.13.7](913-c-compiler.md#9137-헤더와-c-라이브러리)).

## 9.2.19. 설치 레이아웃

설치 레이아웃은 rustup / cargo 식입니다. `NABI_HOME`의 기본값은 `~/.nabi`(Windows에서는 `%USERPROFILE%\.nabi`)입니다.

```
$NABI_HOME/
├── bin/                         # proxy(가장 새 toolchain 의 nabi): 활성 toolchain 의 같은 이름 실행 파일을 실행
│   ├── nabi
│   ├── nabic
│   ├── nabicc
│   ├── nabils
│   └── <name>                   # nabi install 이 설치한 실행 파일 (9.10.5a)
├── toolchains/
│   └── <version>-<triple>/      # 자기 완결 toolchain
│       ├── bin/{nabi, nabic, nabicc, nabils}
│       └── lib/
│           ├── libnabir.a        # 런타임
│           ├── nabi/lldb/nabi.py           # -g 프로그램용 lldb formatter (9.9.6)
│           ├── nabi/targets/<triple>/   # --target 이 링크하는 것: 런타임 (libnabir.a / nabir.lib), C 라이브러리,
│           │                           #   그 헤더 (sysroot/, nabicc 용), C 런타임 오브젝트, wasm32-wasip2 의 preview 1 adapter, host 의 것 포함
│           ├── nabi/packages/          # 함께 싣는 공식 패키지, 읽기 전용: index, src/<sha256>/ (9.10.4a)
│           └── nabi/{core, std, bare}/src/   # 표준 라이브러리 소스: tier 마다 패키지처럼 놓이며 루트는 src/lib.nabi (5.4)
├── cache/                       # git, registry, native, 스크립트 (9.10.4)
├── envs/<name>/, envs/.current  # 이름 있는 환경: nabi.nabion, nabi.lock, target/ (9.10.5)
├── installs.nabion               # nabi install 이 bin/ 에 둔 것과 그 출처
├── tokens.nabion                 # registry API 토큰, nabi login 이 씀 (9.10.4a)
├── sources.nabion                # index 미러, 읽기만 함 (9.10.4a)
└── config.nabion                 # Config { toolchain }
```

`nabic`는 표준 라이브러리와 런타임을 자기 실행 파일 기준으로 찾습니다(sysroot = 실행 파일의 `..`). `NABI_HOME`은 보지 않습니다. 표준 라이브러리의 각 계층은 라이브러리 패키지처럼 놓입니다. 루트 모듈은 `lib/nabi/<tier>/src/lib.nabi`고 모듈은 `lib/nabi/<tier>/src/` 아래에 있습니다. 구현 저장소에서 빌드한 `nabic`는 같은 레이아웃을 저장소의 `library/<tier>/src/`에서 읽습니다. 자기 캐시는 toolchain의 `lib/nabi-cache/`입니다(저장소 빌드는 `target/<profile>/nabi-cache/`). 읽기 전용으로 설치한 toolchain(시스템 패키지)은 대신 사용자 캐시 디렉터리의 `nabi/nabic-<sysroot의 해시>/`를 씁니다. `fn/`에는 함수 캐시, 곧 빌드가 컴파일한 표준 라이브러리 함수가 있고(9.2.16), `library/`에는 라이브러리에서 모은 항목과 사전 패스 파싱 결과가 있습니다(9.2.16). `nabi cache`로 이 캐시를 보고, 줄이고, 비웁니다(9.10.6). 활성 toolchain은 [9.10.6](910-nabi-frontend.md#9106-toolchain)이 정한 대로 고릅니다.

Linux(glibc) toolchain은 자신이 만드는 실행 파일과 마찬가지로 glibc 2.34 이상이 필요합니다(9.2.14).
