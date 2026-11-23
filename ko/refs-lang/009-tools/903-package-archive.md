# 9.3. 패키지 아카이브(.a) 포맷

라이브러리 패키지 `foo`의 빌드 산출물은 두 파일입니다([9.2.15](902-aot-pipeline.md#9215-산출물)).

| 파일        | 내용                                                                          |
|-------------|-------------------------------------------------------------------------------|
| `libfoo.a`  | 패키지의 오브젝트를 codegen unit마다 하나씩 담은 표준 `ar` 아카이브([9.2.16](902-aot-pipeline.md#9216-증분-빌드)) |
| `foo.nabihir`  | 다운스트림이 단형화와 이름 해소에 쓰는 인터페이스([9.4](904-nabihir-bytecode.md)) |

두 파일은 같은 디렉터리에 놓입니다. `--extern foo=libfoo.a`와 `--extern foo=foo.nabihir`은 어느 쪽을 주든 옆에 있는 짝을 찾습니다([9.2.3](902-aot-pipeline.md#923-의존성-해소)). 버전과 producer 같은 메타데이터는 NABIHIR 헤더에 있습니다([9.4](904-nabihir-bytecode.md)).

## 9.3.1. 표준 ar 아카이브

`libfoo.a`는 표준 Unix `ar` 아카이브입니다. `ar`, `ranlib`, `nm`과 C 링커가 변환 없이 다룹니다.

```
ar t libfoo.a
part0.o
part1.o
...
```

## 9.3.2. 오브젝트

각 오브젝트는 [AOT 빌드 파이프라인](902-aot-pipeline.md)이 codegen unit 하나로 만든 표준 ELF / Mach-O / COFF 오브젝트입니다. 오브젝트를 모두 합치면 패키지의 [concrete fn](../000-meta/007-glossary.md#concrete-fn--generic-fn) 가운데 패키지 루트에서 닿는 것만 들어 있습니다([9.2.8](902-aot-pipeline.md#928-specialize-단형화)).

| 섹션      | 내용                                                        |
|-----------|-------------------------------------------------------------|
| `.text`   | concrete `fn`의 코드                                       |
| `.rodata` | `const`, 리터럴, [ScanTable](../008-abi/802-scan-table.md)  |
| `.data`   | `static`의 초기값                                          |
| `.bss`    | 0으로 초기화되는 `static`                                  |

ScanTable은 일반 상수 데이터입니다. 컴파일러가 생성한 등록 코드가 프로세스를 시작할 때 `nabi_gc_register_type`으로 ScanTable을 런타임에 등록합니다([8.2.13](../008-abi/802-scan-table.md#8213-등록-시점)). Nabi 전용 섹션은 없습니다.

## 9.3.3. 정적 라이브러리와 공유 라이브러리

라이브러리의 산출물은 정적 `.a`입니다. 의존성은 아카이브에서 프로그램으로 링크되고, 프로그램이 닿는 표준 라이브러리 코드는 프로그램과 함께 컴파일됩니다([9.3.6](#936-표준-라이브러리)). `--crate-type=cdylib`([9.1.2](901-nabic-cli.md#nabic-build-aot-빌드))은 라이브러리 패키지를 C 코드(또는 Python, R, Julia)가 실행 중에 불러오는 공유 라이브러리로도 링크합니다. 파일은 `lib<name>.so`(Linux), `lib<name>.dylib`(macOS, install name `@rpath/lib<name>.dylib`), 또는 `<name>.dll`과 그 import 라이브러리 `<name>.lib`(Windows)입니다. 패키지, 표준 라이브러리, 런타임이 들어 있고, 패키지의 `#[export] extern "C" fn`과 `#[export_name("sym")] extern "C" fn`([5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-정의외부에-노출))만 자기 이름이나 `sym`으로 내보냅니다. 다른 심볼은 모두 라이브러리 안에서 묶입니다.

- 라이브러리를 불러오면 그 런타임이 시작합니다. 로더가 `nabi_library_init`을 실행합니다(`DT_INIT`, `__mod_init_func` 항목, 또는 프로세스 attach 때의 DLL 진입점 `nabi_dll_main`). 이 함수는 불러온 스레드에서 런타임을 준비하고, 라이브러리의 GC 타입을 등록하고([8.2.13](../008-abi/802-scan-table.md#8213-등록-시점)), 그 스레드를 C 코드 안에 있는 스레드처럼 park합니다. 어느 스레드에서 export를 부르든 그 스레드가 먼저 붙으므로([8.5](../008-abi/805-extern-symbols.md)) C 코드가 초기화 함수를 부를 필요는 없습니다.
- 공유 라이브러리마다 런타임과 힙이 따로 있습니다. GC 객체는 한 Nabi 라이브러리에서 다른 라이브러리로, 또는 그것을 불러온 Nabi 프로그램으로 넘어가지 않습니다. export를 넘는 것은 C 값뿐입니다.
- Linux에서 라이브러리의 스레드 로컬은 initial-exec 모델(`DF_STATIC_TLS`)을 씁니다. glibc는 이것을 정적 TLS 영역에 두는데, `dlopen`으로 연 라이브러리를 위한 자리도 그 안에 있습니다. 라이브러리 하나가 런타임의 스레드 상태([8.3](../008-abi/803-tls-slots.md))로 160바이트를 씁니다. glibc는 나중에 여는 라이브러리를 위해 512바이트를 남겨 두므로(`glibc.rtld.optional_static_tls`) 기본으로는 한 프로세스에 Nabi 라이브러리 세 개가 들어갑니다. 그보다 많으면 `cannot allocate memory in static TLS block`으로 불러오지 못합니다. `GLIBC_TUNABLES=glibc.rtld.optional_static_tls=4096`으로 자리를 늘릴 수 있습니다.
- 버전: `--lib-version <x.y.z>`([9.1.2](901-nabic-cli.md#nabic-build-aot-빌드))는 `nabi build`가 매니페스트의 `version`에서 가져옵니다. ELF soname을 `lib<name>.so.<x>`(1.0 미만에서는 minor 버전이 호환을 깨므로 `lib<name>.so.0.<y>`)로 정하고 파일 옆에 그 이름의 링크를 만들며, Mach-O의 현재 버전과 호환 버전을 `x.y.z`와 `x.0.0`(1.0 미만이면 `0.y.0`)으로 정합니다.
- C 헤더: 빌드는 `--emit=c-header`처럼 `target/<profile>/include/<name>.h`도 씁니다. 패키지에서 `#[export_name]`이 있는 `extern "C"` 함수를 그 이름으로 선언하고, 시그니처가 닿는 타입도 선언합니다. `#[repr(C)]` 구조체와 union은 필드 순서 그대로 `typedef struct`가 되고(`align(N)`은 `_Alignas`, `packed`는 `#pragma pack`), `#[repr(transparent)]` 구조체는 그 필드의 `typedef`, `#[repr(iN)]` enum은 태그 타입의 `typedef`와 variant마다 `#define <ENUM>_<VARIANT> ((<Enum>)<value>)`이 됩니다. 원시 타입은 `<stdint.h>` / `<stddef.h>` / `<stdbool.h>`의 타입이 되고(`usize`는 `size_t`, `isize`는 `ptrdiff_t`, `c_char`는 `char`, `char`는 `uint32_t`, C ABI의 `str`는 `const char *`), `ConstPtr<T>`는 `const T *`, `Ptr<T>` / `ManyPtr<T>`는 `T *`, `CPtr<T>`와 `T | none`은 `/* nullable */` 주석이 붙은 `T *`, `extern "C" fn` 타입은 함수 포인터, `extern "system"` 함수는 `NABI_SYSTEM`(32비트 Windows에서 `__stdcall`), 반환하지 않는 함수는 `_Noreturn`이 됩니다. 항목의 `///` 문서 주석은 C 주석이 됩니다. C가 이름 붙일 수 없는 타입(제네릭 구조체, `#[repr(C)]`가 없는 구조체, 데이터가 있는 enum)을 쓰는 함수는 `/* skipped <name>: <why> */` 줄을 남기고 빠집니다. 헤더에는 `#pragma once`, include guard, C++용 `extern "C"`가 있습니다.
- 타깃: `aarch64-apple-darwin`, `x86_64-unknown-linux-gnu`, `aarch64-unknown-linux-gnu`, `x86_64-pc-windows-msvc`, `aarch64-pc-windows-msvc`이고, Nabi 자체 링커를 씁니다(`NABI_CC`는 안 됨). 다른 타깃은 정적 라이브러리를 C 툴체인으로 링크하라고 알려 주는 에러입니다.
- 다른 공유 라이브러리를 부르는 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 호출은 모든 타깃에서 됩니다.

`--crate-type=staticlib`은 그 대신 C, Swift, Xcode가 그것 하나만 링크하면 되는 정적 라이브러리를 만듭니다. `target/<profile>/static/lib<name>.a`(`-o`로 이름을 정함)는 패키지, 의존성, 패키지가 닿는 표준 라이브러리, 런타임을 담으며, 서로 가리지 않도록 각 아카이브의 멤버 이름을 `<n>-<member>`로 바꿉니다. 런타임은 처음 불린 export에서 시작하고, 공유 라이브러리의 export처럼 그 스레드를 붙입니다. GC 객체는 여기서도 export를 넘지 않습니다. 빌드는 위와 같은 C 헤더와 `target/<profile>/include/module.modulemap`도 씁니다. 이것은 패키지 이름의 clang 모듈로, 라이브러리와 패키지가 지정한 시스템 라이브러리, 프레임워크를 링크하므로 Swift에서 `import <name>`으로 씁니다.

```
module geo {
    header "geo.h"
    link "geo"
    link framework "UIKit"
    export *
}
```

`--xcframework`를 주면 빌드는 라이브러리와 헤더를 `target/<profile>/<name>.xcframework`에도 넣습니다. `xcodebuild -create-xcframework`가 쓰는 것과 같은 XCFramework로, 빌드한 Apple 타깃마다 슬라이스(`ios-arm64`, `ios-arm64-simulator`, `macos-arm64`) 하나에 `lib<name>.a`와 `Headers/`를 두고, `Info.plist`가 그것들을 나열합니다. 빌드할 때마다 그 타깃의 슬라이스를 넣거나 바꾸므로, `aarch64-apple-ios`용 빌드와 `aarch64-apple-ios-sim`용 빌드 두 번으로 Xcode가 둘 다에 쓰는 XCFramework 하나가 됩니다. 기기 타깃은 진행 중이므로([0.6.2](../000-meta/006-platform-targets.md#062-진행-중)) 지금 XCFramework에는 시뮬레이터 슬라이스가 들어갑니다. `--crate-type=staticlib` 없이, 또는 Apple 타깃이 아닌 타깃에 `--xcframework`를 주면 무엇을 쓸지 알려 주는 에러입니다.

iOS 앱은 이렇게 Nabi 코드를 받습니다([0.6.1](../000-meta/006-platform-targets.md#061-지원함)). `aarch64-apple-ios-sim`용 아카이브를, 기기 타깃을 지원하게 되면 `aarch64-apple-ios`용 아카이브도 Xcode 프로젝트에 넣고, `include` 디렉터리를 헤더 검색 경로에 둡니다. wasm 타깃에서 `--crate-type=staticlib`은 에러입니다.

## 9.3.4. 버전 호환 검사

컴파일러는 의존 패키지의 NABIHIR을 읽을 때 그 `version`과 producer(그 NABIHIR을 만든 `nabic`)를 지금의 `nabic`와 비교합니다. 다르면 에러이고, 에러 메시지는 무엇을 다시 빌드할지 밝힙니다([9.4.9](904-nabihir-bytecode.md#949-읽기와-거부)).

## 9.3.5. 심볼 노출

아카이브가 노출하는 심볼은 그 안 오브젝트의 전역 심볼입니다. 이름은 [심볼 맹글링](905-symbol-mangling.md)을 따릅니다.

```
nm libfoo.a
0000000000000000 T _Y3f8a1c2b_foo_geom_distance_a7c91d8e44b2f190
0000000000000040 T _Y3f8a1c2b_foo_geom_Point_origin_b1234567...
```

## 9.3.6. 표준 라이브러리

표준 라이브러리(`bare` / `core` / `std`)는 아카이브로 배포하지 않고, 미리 빌드한 것을 링크하는 빌드도 없습니다. 모든 빌드가 표준 라이브러리를 패키지와 함께 소스에서 컴파일하되(`--bootstrap`), 프로그램이 닿는 부분만 컴파일합니다. 프런트엔드는 프로그램의 루트(`main`, 하니스의 `#[test]`와 `#[bench]` 함수, `#[export_name]`과 WIT export, 컴파일러와 런타임이 부르는 함수, 모든 static의 초기화식)에서 출발해 호출, 함수 값, 쓰는 타입의 `Drop` 구현, `any Trait` 값이 디스패치하는 구현을 따라가며, 만나는 함수마다 타입을 검사하고 낮춥니다([9.2.6](902-aot-pipeline.md#926-타입-검사)). 나머지는 이름 해석, 메서드 찾기, 진단에 쓰는 이름과 시그니처만 읽습니다. `--check-library`를 주면 라이브러리의 모든 본문을 검사하고 컴파일합니다([9.1.2](901-nabic-cli.md#nabic-build-aot-빌드)).

프로그램이 닿는 표준 라이브러리 함수는 모듈마다 하나인 codegen unit으로 컴파일합니다(9.2.16). 함수마다 최적화한 코드를, unit마다 오브젝트를, 그것이 의존하는 모든 것에 대한 키로 toolchain의 함수 캐시(`lib/nabi-cache/fn/`, [9.2.19](902-aot-pipeline.md#9219-설치-레이아웃))에 둡니다. 같은 toolchain의 이후 빌드는 어떤 패키지를 빌드하든 키가 같으면 거기서 가져옵니다. 최적화기는 여전히 표준 라이브러리 함수를 포함한 프로그램 전체를 보므로, 패키지 자신의 함수처럼 표준 라이브러리 함수도 인라인합니다(9.2.10).

라이브러리 패키지의 아카이브는 제네릭 인스턴스처럼, 닿는 표준 라이브러리 코드의 비공개 복사본을 담습니다. 표준 라이브러리의 static만은 예외입니다. 라이브러리는 이것을 자기 심볼 이름 그대로 weak로 정의하므로, 프로그램과 함께 링크하는 모든 라이브러리가 static마다 하나를 함께 씁니다.

## 9.3.7. 링크

```
cc main.o libfoo.a libnabir.a -o my_binary
```

링커는 아카이브의 오브젝트 가운데 필요한 심볼만 링크합니다. `#[export_name]`으로 노출한 함수는 C에서 직접 부를 수 있습니다.

```c
extern uint32_t nabi_compute(int32_t a, int32_t b);   // #[export_name("nabi_compute")]
int main(void) { return (int)nabi_compute(1, 2); }
```

```
cc main.c libfoo.a libnabir.a -o my_binary
```

`--crate-type=staticlib`(9.3.3)이면 아카이브 하나로 됩니다.

```
cc main.c target/release/static/libfoo.a -o my_binary
```

## 9.3.8. 안정성

아카이브 형식은 [ABI 안정성 정책](../008-abi/README.md)을, NABIHIR 형식은 [9.4](904-nabihir-bytecode.md)를 따릅니다.
