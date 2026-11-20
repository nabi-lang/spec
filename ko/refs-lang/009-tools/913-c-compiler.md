# 9.13. `nabicc`: C 컴파일러

toolchain은 자체 C 컴파일러로 C를 컴파일합니다. C는 `nabi bindgen`([9.10.8](910-nabi-frontend.md#nabi-bindgen-c-헤더-바인딩))과 같은 프런트엔드로 읽고, nabi-mir로 하강한 뒤, `nabic`가 쓰는 코드 생성기와 링커로 컴파일하고 링크합니다([0.2.4](../000-meta/002-pipeline.md#024-코드-생성)). 다른 C 컴파일러, 어셈블러, 링커는 관여하지 않고, 설치할 필요도 없습니다.

프런트엔드는 `ffi::objc::import`와 `nabi bindgen -x objective-c`를 위해 Objective-C 헤더도 읽습니다([stdlib 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-헤더-ffiobjcimport)). 선언(`@interface`, `@protocol`, 메서드, 프로퍼티)만 읽으며, `__OBJC__`, `__OBJC2__`, ARC의 기능을 미리 정의하고 `#import`는 파일마다 한 번만 읽습니다. `nabicc` 자체는 C만 컴파일합니다. `.m` 파일은 C 소스가 아닙니다.

## 9.13.1. `nabicc`와 `nabi cc`

C 컴파일러는 동작이 같은 두 형태로 있습니다.

- 독립 실행 파일 `nabicc`. toolchain의 `bin/`에 설치되고, `$NABI_HOME/bin/`에는 `nabi`, `nabic`, `nabils` 옆에 프록시로 놓입니다([9.10.6](910-nabi-frontend.md#9106-toolchain)).
- `nabi cc`. `nabi`가 `nabic`에 위임하듯이, `nabi`는 자기 toolchain의 `nabicc`를 실행하고 `cc` 뒤의 인자를 그대로 넘깁니다([9.10.1](910-nabi-frontend.md#9101-서브명령과-전역-옵션)). `nabi`의 전역 옵션은 `cc` 앞에 올 수 있고, `cc` 뒤의 낱말은 모두 C 컴파일러의 것입니다.

```
nabicc [options] <file>...
nabi cc [options] <file>...
```

명령줄은 gcc와 clang이 받는 형태입니다. 그래서 두 형태 모두 빌드 시스템의 C 컴파일러를 그대로 대신합니다. `CC=nabicc`와 `CC="nabi cc"`는 make, CMake, autoconf, Meson에서 모두 동작합니다.

```
make CC="nabi cc"
cmake -DCMAKE_C_COMPILER=nabicc -B build
```

메시지는 실행한 형태의 이름을 씁니다. `nabi cc: error: ..` 또는 `nabicc: error: ..`입니다. 종료 코드는 성공하면 `0`이고, 소스 컴파일이 실패하거나 옵션이 잘못되었거나 링크가 실패하면 `1`입니다. 내부 컴파일러 에러는 [9.1.6](901-nabic-cli.md#916-종료-코드)의 코드로 끝납니다.

## 9.13.2. 언어

C89, C99, C11, C17, C23은 하나의 방언으로 읽습니다. 어느 표준용으로 쓴 프로그램이든 컴파일되고, 새 문법(`_Static_assert`, `typeof`, `[[attributes]]`, 키워드 `bool`, 2진 리터럴, 숫자 구분자)은 표준과 상관없이 받습니다. `-std=`는 프로그램이 매크로로 보는 표준을 고릅니다.

| `-std=` | `__STDC_VERSION__` |
|---------|--------------------|
| `c89`, `c90`, `gnu89`, `gnu90`(`-ansi`도) | 정의하지 않음 |
| `c99`, `gnu99` | `199901L` |
| `c11`, `gnu11` | `201112L` |
| `c17`, `c18`, `gnu17`, `gnu18` | `201710L` |
| `c23`, `c2x`, `gnu23`, `gnu2x` | `202311L` |

`iso9899:<year>` 표기도 받습니다. 기본값은 `gnu17`입니다. `gnu`가 아닌 `c` 형태는 `__STRICT_ANSI__`도 정의합니다. 그 밖의 표준은 이 목록을 나열하는 에러입니다.

시스템 헤더가 쓰는 GNU, Clang, Apple, Microsoft 확장을 읽습니다. `__attribute__((..))`, `__declspec(..)`, `__asm__("symbol")` 레이블, `__extension__`, `typeof`, 문장 표현식, `__builtin_*` 호출(9.13.4), nullability 한정자, 이름 없는 구조체와 공용체, `#pragma pack`, `#include_next`, `__has_include`, `__has_attribute`, `__has_builtin`, `__has_feature`입니다. 타입은 clang이 그 타깃에서 하듯이 타깃의 C ABI로 배치합니다.

미리 정의된 매크로는 그 타깃에서 clang이 정의하는 것(`__x86_64__`, `__aarch64__`, `__linux__`, `__APPLE__`, `_WIN32`, `__SIZEOF_LONG__`, MSVC가 아닌 타깃의 `__GNUC__`, MSVC 타깃의 `_MSC_VER` 등)에 `__nabicc__`(`1`)를 더한 것입니다. 벡터 확장과 그 기능 매크로(`__SSE__`, `__SSE2__` .. `__AVX512F__`, `__ARM_NEON`, `__ARM_FEATURE_*`, `__wasm_simd128__`, `__riscv_vector`, `__ALTIVEC__` 등)와 `__SIZEOF_INT128__`는 정의하지 않습니다. 아직 벡터 intrinsic이 없으므로, 이 매크로를 검사하는 라이브러리는 스칼라 경로를 탑니다.

## 9.13.3. 거부하는 것

아래 구문은 그 `file:line:col`에서 에러입니다. 각 에러는 대신 무엇을 쓸지 알립니다. `inline`이나 `static` 함수의 본문에 있는 구문은 유닛이 그 함수를 호출할 때만 에러입니다. 아무 코드도 쓰지 않는 헤더의 인라인 함수(C 라이브러리의 SIMD `wmemchr`, `cpuid` 도우미)는 빌드를 막지 않습니다.

| 구문 | 대신 쓸 것 |
|------|------------|
| 인라인 어셈블리(`asm`, `__asm__`). 빈 배리어(`asm("")`, `asm volatile("" ::: "memory")`)와 피연산자 없는 스핀 힌트 하나(`"pause"`, `"yield"`, `"nop"`)는 예외이고, 아무 코드도 만들지 않습니다 | C로 쓴 코드, 또는 라이브러리의 이식 가능한 경로를 고르는 매크로 |
| 레이블의 주소(`&&label`)와 계산된 `goto *p` | `switch` |
| 가변 길이 배열(크기가 상수가 아닌 배열). 포인터인 매개변수는 예외 | 고정 크기, 또는 `malloc`과 `free` |
| `alloca`(`__builtin_alloca`, `_alloca`) | `malloc`과 `free`, 또는 고정 크기 배열 |
| `_Complex` 값의 산술(`__real__`, `__imag__`, 연산자) | 실수부와 허수부를 두 변수에 |
| `__int128`와 `unsigned __int128` | 64비트 반쪽 둘 |
| `long double`이 `double`보다 넓은 곳의 `long double` 값(x86과 x86-64 Linux, FreeBSD의 80비트 x87, aarch64, riscv64, powerpc64le, s390x Linux와 Android, riscv32, wasm32의 128비트) | `double`(많은 라이브러리가 `-DLONGDOUBLE_TYPE=double` 같은 옵션을 받음) |
| `_Float16`, `__fp16`, `__bf16`의 산술 | 먼저 `float`로 변환 |
| 스레드 지역 변수(`_Thread_local`, `thread_local`, `__thread`, `__declspec(thread)`)의 정의. wasi-libc의 `errno` 같은 `extern` 선언은 읽고 씀 | 매개변수로 넘기는 상태, 또는 스레드마다 받는 구조체에 담은 상태 |
| `aarch64-pc-windows-msvc`의 가변 인자 함수 정의(다른 타깃은 컴파일함) | `va_list` 판(`int vf(const char *fmt, va_list ap)`)과 그것을 부르는 가변 인자 없는 래퍼 |
| Windows 구조적 예외 처리(`__try`, `__except`, `__finally`, `__leave`) | 그것 없이 빌드한 라이브러리(sqlite: `-DSQLITE_OMIT_SEH`) |
| 벡터 타입(`__attribute__((vector_size(N)))`, `__m128`, `float32x4_t`, SVE 타입) | 라이브러리의 스칼라 경로(`NO_SIMD` / `NO_SSE` 옵션) |
| `__builtin_frame_address`, `__builtin_return_address`, 9.13.4에 없는 builtin | C로 쓴 연산, 또는 C 라이브러리의 함수 |
| K&R 형식 정의(`int f(a, b) int a; ..`) | 프로토타입: `int f(int a, int b)` |
| 암묵적 `int`(`static x;`) | 타입: `static int x;` |
| 선언하지 않은 함수의 호출 | 그 함수를 선언하는 헤더의 `#include`, 또는 호출 앞의 프로토타입 |

Apple 타깃, Windows, 32비트 Arm에서는 타깃의 C ABI가 `long double`을 64비트로 정하므로 `long double`은 `double`이고, `double`로 컴파일됩니다.

거부하는 옵션과 입력은 다음과 같습니다.

| 옵션이나 입력 | 에러 |
|---------------|------|
| `-shared` | `nabi cc` links executables; build a static library with `ar` (or `nabi build` a package) |
| `.s`, `.S`, `.asm` 소스 | `nabi cc` does not assemble; write the function in C, or link an object built elsewhere |
| C++나 Objective-C 소스(`.cc`, `.cpp`, `.cxx`, `.C`, `.m`, `.mm`), 또는 `c`, `c-header`, `none`이 아닌 언어의 `-x` | `nabi cc` compiles C only; C++ and Objective-C need their own compiler |

## 9.13.4. builtin

다음 builtin은 선언 없이 컴파일됩니다. 각각 `__builtin_` 접두사를 붙여 씁니다(`__builtin_expect`). 마지막 줄들의 이름에는 저마다의 접두사가 있습니다.

| builtin | 뜻 |
|---------|----|
| `va_start`, `va_arg`, `va_end`, `va_copy` | 타깃의 C ABI에 따른 가변 인자. `<stdarg.h>`가 이것을 씁니다. `...`가 없는 함수의 `va_start`는 에러입니다 |
| `expect`, `expect_with_probability` | 그 값. 힌트는 무시합니다 |
| `assume`, `assume_aligned`, `prefetch` | 그 값, 또는 없음. 코드에 영향이 없습니다 |
| `unreachable` | 도달하면 정의되지 않은 동작 |
| `trap`, `debugtrap`(`__debugbreak`도) | trap 명령어로 프로그램을 멈춥니다 |
| `constant_p` | 정수나 부동소수 상수면 `1`, 아니면 `0` |
| `object_size`, `dynamic_object_size` | 알 수 없음. 타입 `0`과 `1`은 `(size_t)-1`, 타입 `2`와 `3`은 `0` |
| `clz`, `ctz`, `popcount`(각각 `l`, `ll` 형태도), `bswap16`, `bswap32`, `bswap64` | 비트 세기와 바이트 순서 뒤집기 |
| `huge_val`, `inf`, `nan`, `nans`(각각 `f`, `l` 형태도) | 부동소수 상수 |
| `isnan`, `isinf`, `isinf_sign`, `isfinite`, `isnormal`, `signbit`, `fpclassify`, `isgreater`, `isgreaterequal`, `isless`, `islessequal`, `islessgreater`, `isunordered` | `<math.h>` 매크로를 위한 분류와 비교 |
| `add_overflow`, `sub_overflow`, `mul_overflow`와 타입별 형태(`sadd_overflow`, `umulll_overflow` 등) | 감싼 결과를 저장하고 오버플로 여부를 돌려주는 산술 |
| 메모리, 문자열, 수학 함수(`memcpy`, `memset`, `strlen`, `strcmp`, `malloc`, `abs`, `sqrt`, `sin`, `pow`, `fabsf` 등) | 접두사를 뗀 이름의 C 라이브러리 함수 호출 |
| fortify 호출(`__builtin___memcpy_chk`, `__builtin___snprintf_chk` 등) | 일반 함수(`memcpy`, `snprintf`) 호출. 객체 크기를 알 수 없으므로(위) 검사는 통과했을 것입니다 |
| `offsetof`(`__builtin_offsetof`) | 멤버의 오프셋. 상수 표현식입니다 |
| `__atomic_*`, `__sync_*`, `__c11_atomic_*`, `_InterlockedIncrement`, `_InterlockedDecrement`, `_InterlockedExchange`, `_InterlockedExchangeAdd`, `_InterlockedCompareExchange`, `_ReadWriteBarrier` | 각 builtin의 뜻대로 하는 load, store, exchange, compare-and-swap, fetch-and-op. `__atomic_is_lock_free` / `__atomic_always_lock_free`는 포인터 두 개 크기까지의 2의 거듭제곱 크기에서 `1` |

`__func__`, `__FUNCTION__`, `__PRETTY_FUNCTION__`은 둘러싼 함수의 이름입니다.

> [!WARNING]
> atomic builtin은 일반(volatile) load와 store, 일반 read-modify-write로 컴파일되고, fence(`__atomic_thread_fence`, `__sync_synchronize`, `_ReadWriteBarrier`)는 아무 코드도 만들지 않습니다. 스레드 하나 안에서는 각 builtin의 결과를 내지만, 스레드 사이에서는 원자적이지 않습니다. 이것으로 스레드 사이에 메모리를 공유하는 C 코드는 `nabicc`에서 아직 올바르지 않습니다.

## 9.13.5. 옵션

| 옵션 | 뜻 |
|------|----|
| `-c` | 소스마다 오브젝트로 컴파일하고 링크하지 않음 |
| `-S` | 소스마다 어셈블리 텍스트(`<stem>.s`)로 컴파일 |
| `-E` | 전처리만 하고 표준 출력이나 `-o`로 |
| `-o <file>` | 출력. `-c`, `-S`, `-E`와 함께면 소스를 하나만 받음 |
| `-I <dir>`, `-isystem <dir>`, `-iquote <dir>`, `-idirafter <dir>` | 헤더를 찾는 디렉터리. gcc와 같은 순서로 찾음 |
| `-include <file>` | 소스 앞에 `<file>`을 읽음 |
| `--sysroot <dir>`, `-isysroot <dir>` | 타깃의 헤더와 C 라이브러리(9.13.7) |
| `-nostdinc`, `-nostdlibinc`, `-nobuiltininc` | 시스템 헤더, C 라이브러리의 헤더, 내장 헤더를 각각 뺌 |
| `-D <name>[=<value>]`, `-U <name>` | 매크로를 정의하거나 지움 |
| `-Wp,<opts>` | 이 표의 전처리 옵션. 쉼표로 구분 |
| `-std=<std>` | 표준(9.13.2) |
| `-O0`, `-O1`, `-O2`, `-O3`, `-Os`, `-Oz` | 최적화 수준(기본 `-O0`). `-O`와 `-Og`는 `-O1`, `-Ofast`는 `-O3` |
| `-g`, `-g0` | 디버그 정보를 켜거나 끔(`-ggdb`, `-g2`, `-g3`, `-gdwarf*`, `-gcodeview`도 켬) |
| `-l<lib>`, `-L<dir>`, `-Wl,<opts>` | 링크할 라이브러리, 라이브러리 디렉터리, 링커 옵션 |
| `-M`, `-MM` | 소스와 그 소스가 읽는 헤더의 make 규칙을 출력하고 컴파일하지 않음. `-MM`은 시스템 헤더를 뺌 |
| `-MD`, `-MMD` | 컴파일하면서 그 규칙을 `<출력 또는 stem>.d`에 씀. `-MMD`는 시스템 헤더를 뺌 |
| `-MF <file>` | 규칙을 `<file>`에 씀 |
| `-MT <target>`, `-MQ <target>` | 규칙의 대상(여러 번 가능). 기본은 오브젝트 |
| `-MP` | 헤더마다 빈 규칙을 더해, 헤더를 지워도 make가 멈추지 않게 함 |
| `--target=<triple>`, `-target <triple>` | 다른 타깃용으로 컴파일(9.13.7) |
| `-x c` | 뒤따르는 입력을 확장자와 상관없이 C로 읽음 |
| `-dumpmachine` | 타깃의 C 트리플을 출력 |
| `-dumpversion`, `-dumpfullversion` | toolchain의 버전을 출력 |
| `--version` | `nabicc <version>`을 출력. `-v`와 함께면 `host: <triple>`도 |
| `-v` | 자세히(`--version`과 함께) |
| `--help`, `-h` | 사용법을 출력 |

`--target`은 Nabi의 타깃 이름(`nabi --print target-list`, [0.6](../000-meta/006-platform-targets.md))과, clang과 gcc가 그 타깃에 쓰는 트리플을 받습니다. `x86_64-linux-gnu`는 `x86_64-unknown-linux-gnu`, `arm64-apple-macos`는 `aarch64-apple-darwin`, `aarch64-linux-musl`은 `aarch64-unknown-linux-musl`, `i386-linux-gnu`는 `i686-unknown-linux-gnu`, `armv7-linux-gnueabihf`는 `armv7-unknown-linux-gnueabihf`, `armv7-linux-musleabihf`는 `armv7-unknown-linux-musleabihf`, `riscv64-linux-gnu`는 `riscv64gc-unknown-linux-gnu`입니다. 어느 것도 아닌 이름은 에러입니다.

다음은 받되 효과가 없습니다. 그 밖의 모든 `-W*`(경고), `-f*`, `-m*` 옵션, `--param`, `-fPIC`, `-fpic`, `-fPIE`, `-fpie`, `-pie`, `-no-pie`, `-rdynamic`, `-pthread`, `-pedantic`, `-pedantic-errors`, `-w`, `-pipe`, `-s`, `-static`, `-Qunused-arguments`입니다. 코드는 언제나 위치 독립적입니다. 그 밖의 옵션은 위 옵션을 나열하는 에러입니다.

`.c`, `.h`, `.i`로 끝나는 입력은 C 소스입니다. 그 밖의 입력(오브젝트, 아카이브, `.lib` 파일)은 명령줄의 그 자리 그대로 링크로 갑니다. 소스는 병렬로 컴파일합니다. `-o`가 없으면 실행 파일은 `a.out`(Windows 타깃은 `a.exe`)이고, 오브젝트는 현재 디렉터리의 `<stem>.o`(Windows 타깃은 `<stem>.obj`)입니다. 소스도 링크 입력도 없으면 에러입니다(`no input files`).

## 9.13.6. 코드 생성

C 함수는 저마다 nabi-mir의 C 함수로 하강합니다. `#[export_name]`이 붙은 `extern "C" fn`처럼([5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-정의외부에-노출)), C 이름 아래의 C ABI 본체입니다. Nabi의 네이티브 코드 생성기가 이것을 컴파일하고, `-O1` 이상에서는 nabi-mir 최적화기와 코드 생성기의 최적화를 적용합니다([9.2.10](902-aot-pipeline.md#9210-최적화-수준)). 정수 산술은 C를 따릅니다. 오버플로 검사는 없습니다.

호출 규약은 타깃의 C ABI이고, Nabi의 `extern "C"`가 쓰는 것과 같습니다([8.6](../008-abi/806-calling-convention.md)). 그래서 Nabi에서 `nabicc`가 컴파일한 C를 부르는 호출과, 그 C에서 Nabi의 `extern "C" fn`을 부르는 호출은 구조상 맞습니다. 부동소수 연산은 축약하지 않습니다. Nabi 코드와 마찬가지로 `a * b + c`는 곱셈과 덧셈이고, fused multiply-add가 되지 않습니다.

`-g`는 DWARF(Linux, Apple, wasm32)나 CodeView(Windows)의 줄 테이블과 타입을 냅니다. Apple 타깃에서는 링크가 DWARF를 실행 파일 옆의 `.dSYM` 번들로 모으고, Windows에서는 링커가 PDB를 씁니다([9.2.14](902-aot-pipeline.md#9214-링킹실행-파일의-경우)).

실행 파일은 Nabi의 링커가 타깃의 C 라이브러리(9.13.7)와 링크합니다. ELF 타깃에서는 `-lm`을 언제나 더합니다. `-lc`, `-lpthread`, `-ldl`, `-lrt`는 링크에 이미 있는 C 라이브러리의 일부를 가리키므로 뺍니다. `-nostdlib`, `-nodefaultlibs`, `-nostartfiles`도 뺍니다. Windows에서는 toolchain의 시작 코드가 C의 `main(argc, argv, envp)`을 부르고(`wmain`은 부르지 않음), 링크가 `atexit`를 줍니다. 32비트 Arm과 wasm32에서는 링크가 컴파일러 도우미 함수(`__aeabi_idiv`, `__multi3`, ..)를 타깃용 toolchain 런타임 아카이브에서 가져옵니다.

## 9.13.7. 헤더와 C 라이브러리

헤더와 C 라이브러리는 타깃에 따라 정해집니다.

| 타깃 | 헤더 | C 라이브러리 |
|------|------|--------------|
| Apple | macOS SDK. `SDKROOT`, 없으면 Xcode나 Command Line Tools의 SDK | libSystem |
| Linux, glibc | toolchain이 그 타깃용으로 담은 sysroot `lib/nabi/targets/<target>/sysroot`. 링크하는 C 런타임과 같은, 고정한 Debian 패키지의 glibc 2.36 헤더([9.2.14](902-aot-pipeline.md#9214-링킹실행-파일의-경우)) | 담아 둔 glibc 링크 stub |
| Linux, musl | 아직 담은 sysroot가 없음: musl 호스트에서는 호스트의 것, 그 밖에는 `--sysroot <dir>` | 담아 둔 musl |
| Windows | Windows 호스트에서 Windows SDK의 UCRT 헤더와 Visual Studio의 C 헤더. `INCLUDE`, 없으면 `vswhere`로 찾은 가장 새 Visual Studio와 Windows SDK | toolchain 자체의 import library와 시작 코드를 거친 UCRT. MSVC 라이브러리는 링크하지 않음 |
| wasm32 | 타깃의 sysroot로 담아 둔 wasi-libc 헤더 | 담아 둔 wasi-libc |
| Android | Android NDK의 sysroot(링크할 때처럼 찾음, [0.6.1](../000-meta/006-platform-targets.md#android))와 그 안의 타깃용 디렉터리 | bionic: API 레벨에 맞는 NDK의 crt 오브젝트와 라이브러리 |
| riscv32(OS 없음) | freestanding 헤더만(아래) | 없음 |

타깃이 호스트이고 toolchain이 그 타깃의 sysroot를 담고 있지 않으면 호스트의 `/usr/include`를 씁니다. 어느 것도 없는 타깃은 에러이고, 에러는 없는 디렉터리와 `--sysroot <dir>`를 알립니다. `--sysroot <dir>`(또는 `-isysroot <dir>`)는 이 표를 덮습니다. 그 아래의 헤더(`usr/include`, Linux에서는 타깃의 multiarch 디렉터리와 함께, wasm32에서는 `include`)를 대신 찾습니다.

C 라이브러리의 헤더보다 내장 헤더를 먼저 찾습니다. clang 자체의 헤더를 대신하는 freestanding 헤더로, `stddef.h`, `stdint.h`, `stdarg.h`, `stdbool.h`, `stdalign.h`, `stdnoreturn.h`, `stdatomic.h`, `limits.h`, `float.h`, `inttypes.h`, `iso646.h`와 [std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#bindgen이-읽는-c)의 intrinsic 헤더입니다. intrinsic 헤더의 함수는 여기서 정의가 없습니다. `-nobuiltininc`는 내장 헤더를, `-nostdinc`는 모든 시스템 디렉터리를 뺍니다.

## 9.13.8. 패키지의 C 소스

패키지는 `nabi.nabion`의 필드 `c`에 C 소스를 적습니다([9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)).

```nabion
Package {
    name: "notes",
    c: {
        sources: ["vendor/sqlite3.c"],
        include: ["vendor"],
        define: { "SQLITE_THREADSAFE": "0", "NDEBUG": "" },
        std: "c17",
        flags: ["-Wall"],
    },
}
```

| 필드 | 뜻 |
|------|----|
| `sources` | C 파일. 매니페스트 기준 상대 경로 |
| `include` | 헤더 디렉터리. 매니페스트 기준 상대 경로(`-I`). 그 뒤에 패키지의 `Dep::system` 라이브러리의 것이 옴 |
| `define` | 매크로(`-D<name>=<value>`). 빈 값은 매크로를 빈 값으로 정의 |
| `std` | 표준. `c89`, `c99`, `c11`, `c17`, `c23` 또는 `gnu` 판(기본 `gnu17`) |
| `flags` | 추가 옵션. `-O*`, `-g`, `-W*`, `-f*`(`-flto` 제외), `-m*`만 |

내장 컴파일러는 소스마다 빌드의 최적화 수준(`-O0`, `--release`면 `-O2`, 또는 준 `-O`)으로 컴파일합니다. `flags`의 `-O`가 있으면 그것을 씁니다. 오브젝트는 `target/<profile>/c/<package>/`에 쓰고, 입력(소스와 그것이 읽은 헤더)이나 옵션이 바뀐 소스만 다시 컴파일합니다. 오브젝트는 패키지가 링크되는 곳마다 함께 링크됩니다. 패키지의 실행 파일과 테스트 하네스, 그리고 그 패키지에 의존하는 패키지의 실행 파일입니다. 패키지의 Nabi 코드는 `extern "C"` 블록([5.9](../005-modules/509-extern-block.md))으로 C 함수를 부릅니다. `--target`은 C도 그 타깃용으로 컴파일합니다.

`flags`에 이 밖의 옵션이 있으면 매니페스트를 읽을 때 에러이고, 에러는 `flags`가 받는 옵션을 나열합니다. 없는 소스나 디렉터리는 그 이름을 밝히는 에러입니다. 컴파일에 실패한 C 소스는 컴파일러의 에러와 함께 빌드를 실패시킵니다. 외부 C 컴파일러는 결코 실행하지 않고, 빌드할 때 패키지의 코드는 실행되지 않습니다(9.10.3).
