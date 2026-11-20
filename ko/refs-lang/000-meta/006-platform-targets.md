# 0.6. 플랫폼 타깃

Nabi는 아래의 내장 타깃과, 같은 아키텍처와 object 형식의 사용자 정의 타깃(0.6.5)용으로 어느 호스트에서든 빌드합니다(`--target`, [0.2.4](002-pipeline.md#024-코드-생성)). 모든 타깃은 세 상태 가운데 하나입니다. 지원함(0.6.1), 진행 중(0.6.2), 지원 안 함(0.6.3)입니다.

## 0.6.1. 지원함

지원하는 타깃은 내장 타깃입니다. `nabic --print target-list`가 이 타깃을 나열하고, `--target`은 그 어느 것이든 받습니다.

### 64비트 데스크톱 / 서버

Linux, macOS, Windows의 주요 아키텍처를 모두 다룹니다. 정적 바이너리용 musl 변형과 FreeBSD까지 들어 있습니다.

| 타깃                         | 비고                                       |
|------------------------------|--------------------------------------------|
| `x86_64-unknown-linux-gnu`   | Linux 서버 / 데스크톱 주력                 |
| `aarch64-apple-darwin`       | macOS Apple Silicon                        |
| `aarch64-unknown-linux-gnu`  | ARM 서버 (AWS Graviton, Raspberry Pi 5 등) |
| `x86_64-pc-windows-msvc`     | Windows 데스크톱                           |
| `aarch64-pc-windows-msvc`    | Windows ARM                                |
| `x86_64-unknown-linux-musl`  | Alpine / `static` binary                   |
| `aarch64-unknown-linux-musl` | 동일, ARM                                  |
| `x86_64-unknown-freebsd`     | FreeBSD 14                                 |

### WebAssembly

브라우저 실행과 서버사이드 실행을 모두 고려해 WebAssembly 타깃 세 가지를 지원합니다. 지금은 WASI 소켓(preview 2 component 전용)과 WASI threads(preview 1 core module 제안)를 한 모듈에서 함께 쓸 수 없습니다. 그래서 서버사이드 WASM 타깃이 둘입니다.

| 타깃                     | 비고                             |
|--------------------------|----------------------------------|
| `wasm32-unknown-unknown` | 브라우저 환경. `target_os = "unknown"`([6.3.2](../006-meta-system/603-cfg.md)) |
| `wasm32-wasip2` | 서버사이드 WASM: WASI preview 2 component. `wasi:sockets` 네트워크, 스레드 없음(`thread::spawn`은 panic을 일으키고, `thread::Builder::spawn`은 `Unsupported`를 던짐) |
| `wasm32-wasip1-threads` | 스레드가 있는 서버사이드 WASM: wasi-threads를 쓰는 WASI preview 1 core module(공유 메모리). 네트워크는 호스트가 넘긴 listener에서만 쓸 수 있음. connect / bind / DNS는 `Unsupported`를 던짐 |

> [!NOTE]
> wasmtime이 v46 이후 wasi-threads를 제거했습니다(WAMR의 iwasm과 wasmer는 아직 구현합니다). 그래서 `wasm32-wasip1-threads`는 앞으로 WASI shared-everything-threads에 기댈 예정입니다.

이 이름들은 생태계의 나머지 도구를 따릅니다. 그곳에서 `wasm32-wasi`는 WASI preview 1을 뜻합니다. Nabi에는 `wasm32-wasi`, `wasm32-wasi-threads`, `riscv32imac-unknown-none`이라는 타깃이 없습니다. 이 이름을 `--target`에 주면 모르는 타깃이고, 에러는 가장 가까운 내장 타깃을 알립니다([9.1.2](../009-tools/901-nabic-cli.md#912-핵심-서브명령)).

### 32비트

아직 활발히 쓰이는 임베디드와 레거시 환경을 위해 다음 32비트 타깃 네 가지를 지원합니다.

| 타깃                            | 비고                                 |
|---------------------------------|--------------------------------------|
| `i686-unknown-linux-gnu`        | 32-bit x86 Linux                     |
| `i686-pc-windows-msvc`          | 32-bit Windows                       |
| `armv7-unknown-linux-gnueabihf` | 32-bit ARM Linux, hard float (32비트 OS의 Raspberry Pi 2~4 등) |
| `armv7-unknown-linux-musleabihf` | 동일, musl: `static` 바이너리 |

### 모바일

| 타깃                    | 비고                                      |
|-------------------------|-------------------------------------------|
| `aarch64-apple-ios-sim` | Apple Silicon Mac의 iOS 시뮬레이터        |

`aarch64-apple-ios-sim`(Apple silicon Mac의 iOS 시뮬레이터)은 iOS 13.4 이상을 대상으로 빌드합니다. 13.4는 Apple 링커가 쓰는 형식인 chained fixup을 로더가 읽는 첫 iOS입니다. `IPHONEOS_DEPLOYMENT_TARGET`이나 nabi.nabion의 `deployment_target: { ios: "16.0" }`([9.10](../009-tools/910-nabi-frontend.md))로 버전을 올릴 수 있고, 더 낮은 버전은 가장 낮은 버전을 알려 주는 오류입니다. 링크에는 Xcode의 iOS SDK가 필요합니다. `SDKROOT`, 없으면 개발자 디렉터리(`DEVELOPER_DIR`, 없으면 `xcode-select`가 가리키는 곳)의 `iPhoneSimulator` SDK를 씁니다. SDK가 없으면 빌드는 Xcode를 설치하라고 알려 줍니다. `SDKROOT`가 다른 플랫폼의 SDK이면(iOS 타깃에 macOS SDK처럼, `SDKSettings.json`이나 디렉터리 이름으로 판단) 노트를 남기고 건너뛰며 개발자 디렉터리의 SDK를 씁니다. 오브젝트와 실행 파일은 자기 플랫폼(`LC_BUILD_VERSION`: iOS 시뮬레이터)을 적고, 링커는 다른 Apple 플랫폼용으로 빌드한 오브젝트를 거부합니다. baseline CPU는 `apple-m1`입니다. `target_os`는 `"ios"`이고 `target_abi`는 `"sim"`입니다([6.3.2](../006-meta-system/603-cfg.md)). 실행 파일은 ad hoc 서명되고, 시뮬레이터는 그대로 실행합니다. `nabi run`, `nabi test`, `nabi bench`는 실행 파일을 타깃의 `simrun.sh`로 실행합니다. 이 스크립트는 부팅된 기기(또는 `NABI_SIM_DEVICE`가 가리키는 기기)에서 `xcrun simctl spawn`을 실행하고, `simctl`이 프로그램에 넘기는 것은 `SIMCTL_CHILD_<name>`뿐이므로 모든 환경 변수를 그 이름으로 넘깁니다. 앱의 Nabi 코드는 Xcode가 링크하는 라이브러리입니다. `--crate-type=staticlib`([9.3.3](../009-tools/903-package-archive.md#933-정적-라이브러리와-공유-라이브러리))은 패키지, 의존성, 런타임을 담은 아카이브 하나를 만들고, Swift는 그 module map을 읽습니다(`import <name>`). `--crate-type=plugin`과 `--hot`은 시뮬레이터에서 동작하고, REPL은 돌지 않습니다. 시뮬레이터 corpus는 관리자가 자기 기계에서 돌립니다.

기기 타깃 `aarch64-apple-ios`(iPhone과 iPad)는 진행 중입니다(0.6.2). 아래 내용은 이 타깃에만 해당합니다. 시뮬레이터처럼 iOS 13.4 이상을 대상으로 빌드하고, `iPhoneOS` SDK로 링크하며, `LC_BUILD_VERSION`에 iOS 플랫폼을 적습니다. baseline CPU는 `apple-a7`(A9처럼 LSE atomic 없음)이고, `target_abi`는 두지 않습니다. 기기는 개발자가 Xcode로 그 기기용 서명을 한 뒤에만 실행 파일을 실행하므로, `nabi run`, `nabi test`, `nabi bench`는 시뮬레이터 타깃을 알려 주는 오류입니다. 기기에서 앱은 프로세스를 시작할 수 없으므로 `Command`는 `Unsupported`를 던지고, 실행 중에 코드를 불러오는 기능도 없습니다. `--crate-type=plugin`과 `--hot`은 오류입니다.

### Android

| 타깃                    | 비고                                      |
|-------------------------|-------------------------------------------|
| `x86_64-linux-android`  | Android x86-64(에뮬레이터), API 레벨 29 이상 |

`x86_64-linux-android`는 Android 10(API 레벨 29) 이상용 실행 파일과 공유 라이브러리(`--crate-type=cdylib`, `System.loadLibrary`가 읽는 것)를 빌드합니다. 이것을 앱으로 묶는 일은 Android 빌드가 맡습니다. 가장 낮은 레벨이 29인 까닭은 Android의 C 라이브러리인 bionic이 이 레벨부터 ELF thread-local storage를 실행하고, 런타임이 스레드마다의 상태를 거기에 두기 때문입니다. `ANDROID_PLATFORM=<n>`(또는 NDK의 빌드 도구가 받는 형태인 `android-<n>`)는 더 높은 레벨용으로 빌드합니다. 29보다 낮은 레벨은 에러이고, 에러는 `ANDROID_PLATFORM=29` 이상을 쓰라고 알립니다. 링크는 그 레벨의 C 런타임 오브젝트와 bionic 라이브러리를 Android NDK에서 가져옵니다. NDK는 toolchain에 들어 있지 않습니다. 찾는 순서는 `ANDROID_NDK_HOME`, 그다음 `ANDROID_NDK_ROOT`, 그다음 Android SDK(`ANDROID_HOME`, 그다음 `ANDROID_SDK_ROOT`, 그다음 Android Studio가 두는 곳)에서 가장 새 `ndk/<version>`입니다. NDK가 없으면 에러는 어디를 찾았는지와 `ANDROID_NDK_HOME`을 설정하라는 것을 알립니다. 그 레벨의 라이브러리가 없는 NDK는 에러이고, 에러는 NDK에 있는 레벨을 알립니다. `nabicc`는 bionic 헤더를 같은 NDK에서 가져옵니다([9.13.7](../009-tools/913-c-compiler.md#9137-헤더와-c-라이브러리)). `target_os`는 `"android"`이고 `target_env`는 없습니다. 커널은 Linux이므로 `std`가 `target_os = "linux"` 아래에 둔 것(`std::os::linux`, `open` 플래그 등)은 Android에도 있습니다. 실행 파일은 `/system/bin/linker64`용 PIE입니다. `x86_64-linux-android`는 x86_64 Android ABI인 `x86-64-v2`(SSE4.2, POPCNT)용으로, 그리고 최대 16 KiB 페이지용으로 빌드합니다. 앱의 프로세스는 bionic이 하지 않는 시스템 호출을 하면 프로세스를 끝내는 seccomp 필터 아래에서 실행됩니다. 그래서 런타임은 거기서 자기만의 시스템 호출을 하지 않습니다. `ChildExt::pidfd`는 던지고(`ENOSYS`), `Metadata::created`는 `Unsupported`를 던지며, 파일에서 파일로의 복사는 `sendfile`을 거칩니다. Android 14(API 레벨 34) 전에는 bionic의 `posix_spawn`이 자식의 디렉터리를 바꾸지 못하므로, 거기서 `current_dir`을 준 `Command`는 fork한 뒤 exec합니다.

기기 타깃 `aarch64-linux-android`(Android arm64, API 레벨 29 이상)는 진행 중입니다(0.6.2). `x86_64-linux-android`처럼 빌드하고 위 내용이 그대로 해당하지만, 최대 64 KiB 페이지용으로 빌드한다는 점만 다릅니다.

이 타깃들로 과학, 데이터, Jupyter, 데스크톱 분야 사용자 대부분을 지원합니다.

### 32비트 지원

32비트 타깃을 지원하므로, 사양 전체는 32비트와 64비트 환경 양쪽에서 동작하도록 규정합니다. 객체 헤더 레이아웃은 두 비트 폭에서 같습니다. 32비트 환경에서 64비트 정수의 atomic 연산은 타깃의 네이티브 64비트 atomic 명령(i686의 `cmpxchg8b`, ARMv7의 `ldrexd` / `strexd`)을 씁니다. 그런 명령이 없으면 시스템 atomic 라이브러리를 씁니다(진행 중인 riscv32에서는 스레드 하나를 위한 런타임의 것). `usize`의 비트 폭은 타깃의 기본 포인터 폭을 자동으로 따릅니다. wasm32의 힙은 linear memory입니다. 필요할 때 wasm32 한도인 4 GiB까지 늘어납니다. 그 밖의 32비트 타깃에서 가비지 컬렉터(GC)가 쓸 수 있는 힙은 약 3.5 GB입니다. block heap은 연속 공간을 최대 2 GiB 예약합니다. OS가 거절하면 그보다 적게 예약합니다. 주소 공간의 나머지는 큰 객체가 차지하고, Windows x86 이미지는 large-address-aware입니다. riscv32의 힙은 보드가 내주는 메모리입니다(0.6.2). 64비트에서는 사실상 무제한입니다. 시간은 32비트 타깃에서도 64비트 인터페이스(`time64`)를 씁니다. 그래서 `SystemTime`의 범위는 어디서나 같습니다.

### 테스트

CI는 지원하는 타깃을 아래 표처럼 빌드하고 테스트합니다. 레벨은 `nabic`의 최적화 레벨이고, 코퍼스는 컴파일러의 end-to-end 테스트와 differential 테스트 모음입니다.

| 타깃 | PR마다 | 머지 큐(머지에 필요) | 야간 |
|---|---|---|---|
| `x86_64-unknown-linux-gnu` | 전체 테스트, 코퍼스 `-O0` | 전체 테스트, 코퍼스 `-O0` | 전체 테스트. 코퍼스 `-O0`, `-O1`, `-O2`, `-O3`, `-Os`, `-O2 -g`. GC 스트레스 `-O2` |
| `x86_64-pc-windows-msvc` | 전체 테스트, 코퍼스 `-O0` | 전체 테스트, 코퍼스 `-O0` | 전체 테스트. 코퍼스 `-O0`, `-O2` |
| `aarch64-apple-darwin` | | 전체 테스트, 코퍼스 `-O0` | 전체 테스트. 코퍼스 `-O0`, `-O1`, `-O2`, `-O3`, `-Os`, `-O2 -g` |
| `aarch64-unknown-linux-gnu` | | 전체 테스트, 코퍼스 `-O0` | 전체 테스트. 코퍼스 `-O0`, `-O2`. GC 스트레스 `-O2` |
| `aarch64-pc-windows-msvc` | | | 모든 테스트, 코퍼스 `-O0`, `-O2`(arm64 toolchain으로, 그리고 에뮬레이션으로 도는 x64 toolchain으로) |
| `x86_64-unknown-linux-musl`, `aarch64-unknown-linux-musl` | | | 그 타깃을 실행하는 호스트에서 코퍼스 `-O0`, `-O2` |
| `i686-pc-windows-msvc` | | | 코퍼스 `-O0`, `-O2`, `-O2 -g` |
| `i686-unknown-linux-gnu` | | | 코퍼스 `-O0`, `-O2` |
| `armv7-unknown-linux-gnueabihf` | | | qemu-user에서 코퍼스 `-O0`, `-O2` |
| `wasm32-wasip2`, `wasm32-wasip1-threads`, `wasm32-unknown-unknown` | | | wasmtime과 node에서 코퍼스 `-O0`, `-O2` |
| `x86_64-unknown-freebsd` | | | FreeBSD 14 가상 머신에서 코퍼스 `-O2` |

변경은 머지 큐 실행을 통과해야 머지됩니다. 야간 작업이 실패하면 이슈가 열립니다. 야간 실행은 진행 중인 타깃(0.6.2) 셋, `riscv64gc-unknown-linux-gnu`, `powerpc64le-unknown-linux-gnu`, `s390x-unknown-linux-gnu`의 코퍼스도 qemu-user에서 `-O0`, `-O2`로 빌드합니다. 그 밖의 타깃과 최적화 레벨 조합은 CI가 아니라 릴리스 전에 메인테이너의 테스트 머신에서 실행합니다. `armv7-unknown-linux-gnueabihf`와 `armv7-unknown-linux-musleabihf`는 거기서 32비트 ARM 코드도 실행하는 CPU(Neoverse N1)의 arm64 머신으로 모든 레벨에서 네이티브로 실행합니다. `x86_64-linux-android`는 그 머신의 Android 에뮬레이터에서만, `aarch64-apple-ios-sim`은 iOS 시뮬레이터에서만 테스트합니다. `wasm32-unknown-unknown`은 그 머신에서 Chromium, Firefox, WebKit으로도 코퍼스 `-O0`, `-O2`를 실행합니다. 프로그램마다 페이지의 전용 Worker에서 실행합니다. WASI 타깃도 그 머신에서 wasmtime 말고 다른 런타임으로 코퍼스 `-O0`, `-O2`를 실행합니다. `wasm32-wasip2`는 jco(컴포넌트를 JavaScript로 바꿔 Node.js에서 실행)로, `wasm32-wasip1-threads`는 WAMR의 iwasm과 wasmer로 실행합니다. 컴포넌트의 export 호출은 jco에서도 확인하고, 이 런타임 가운데 하나가 돌리지 못하는 케이스(런타임의 버그나 빈틈)는 그 런타임에서 건너뜁니다.

## 0.6.2. 진행 중

아래 타깃은 내장되어 있지만, 대상 하드웨어에서 아직 테스트하지 않았으므로 지원하지 않습니다. 각 타깃은 이후 릴리스에서, 0.2.0 전에 지원할 예정이며, 그때 0.6.1로 옮깁니다. 그 이름을 `--target`에 주면(`nabic`, `nabicc`, `nabi` 모두, `--print target-spec`도) 타깃이 진행 중이라는 것과 지금 무엇으로 빌드할지 알리는 에러입니다. `nabic --print target-list`는 이 타깃을 나열하지 않습니다.

| 타깃 | 진행 중인 이유 | 지금 쓸 것 |
|------|----------------|------------|
| `aarch64-apple-ios`(iOS 기기) | 빌드와 링크는 되지만 iPhone이나 iPad에서 아직 실행하지 않음 | iOS 시뮬레이터에서 `aarch64-apple-ios-sim` |
| `aarch64-linux-android`(Android arm64) | Android 시스템 이미지를 둔 qemu-user에서 테스트했고, 기기에서는 아직 안 함 | Android 에뮬레이터에서 `x86_64-linux-android` |
| `riscv64gc-unknown-linux-gnu`(64비트 RISC-V Linux) | qemu-user에서 테스트했고, RISC-V 하드웨어에서는 아직 안 함 | 없음 |
| `powerpc64le-unknown-linux-gnu`(IBM Power) | qemu-user에서 테스트했고, POWER 하드웨어에서는 아직 안 함 | 없음 |
| `s390x-unknown-linux-gnu`(IBM Z, big-endian) | qemu-user에서 테스트했고, IBM Z에서는 아직 안 함 | 없음 |
| `riscv32imac-unknown-none-elf`(RISC-V 임베디드, 운영체제 없음) | qemu-riscv32에서 테스트했고, 보드에서는 아직 안 함 | 없음 |

테스트하는 동안 이 타깃의 코드는 toolchain에 남습니다. 메인테이너는 환경에 `NABI_IN_PROGRESS_TARGETS=1`을 두고 이 타깃을 빌드합니다. 이것이 있으면 `--target`과 `--print target-list`가 이 타깃을 받습니다. toolchain을 테스트하기 위한 스위치이고 Nabi의 일부가 아닙니다. 아무것도 약속하지 않으며, 어느 릴리스에서든 바뀌거나 없어질 수 있습니다. 이 페이지가 이 타깃에 관해 말하는 내용은 지금 빌드되는 모습을 적은 것입니다.

### `riscv32imac-unknown-none-elf`

`riscv32imac-unknown-none-elf`에는 운영체제가 없습니다. 프로그램은 C 라이브러리 없이 런타임과 함께 정적 이미지 하나로 링크되고, 스레드 하나로 실행됩니다. `wasm32-wasip2`와 마찬가지로 `thread::spawn`은 panic하고 `thread::Builder::spawn`은 `Unsupported`를 던집니다. 파일, 프로세스, 소켓, C 라이브러리 함수는 없으며, 이를 쓰는 프로그램은 링크되지 않습니다. 환경 변수는 없고(설정하면 `Unsupported`를 던짐), `random`은 엔트로피 소스가 없어 panic합니다. 런타임은 이미지의 진입점 `_start`와 힙을 갖춥니다. `_start`는 Linux 프로세스가 시작할 때처럼 초기 스택에서 인자를 얻고, TLS 블록을 마련한 뒤 `main`을 부릅니다. 메모리, 출력, 시간, 프로그램 종료는 보드 함수 네 개에서 얻습니다: `nabi_bare_morecore(len)`, `nabi_bare_write(fd, ptr, len)`, `nabi_bare_clock_ns(realtime)`, `nabi_bare_exit(code)`. 런타임이 직접 정의한 `_start`와 이 함수들은 weak이고 Linux 시스템 호출을 씁니다. `qemu-riscv32`가 user mode로 이미지를 실행하는 방식입니다. 보드는 자기 정의를 두며, 보드의 startup은 `sp`와 `gp`를 마련한 뒤 `nabir_bare_start(null)`을 부릅니다(인자 없음). 부동소수점, 64비트 나눗셈, 수학 함수는 런타임(compiler_builtins)에서 오므로 `exp`, `ln` 같은 함수의 결과가 glibc와 마지막 비트에서 다를 수 있습니다.

## 0.6.3. 지원 안 함

그 밖의 타깃은 지원하지 않습니다. 그 이름을 `--target`에 주면 모르는 타깃입니다([9.1.2](../009-tools/901-nabic-cli.md#912-핵심-서브명령)). 아래 이름에서는 에러가 이유를 알리고, 대신 쓸 것이 있으면 그것도 알립니다.

| 타깃 | 대신 쓸 것 |
|------|------------|
| `arm-unknown-linux-gnueabi`(ARMv6, soft float: Raspberry Pi Zero / 1) | 없음 |
| `armv7-linux-androideabi`(32비트 Android) | 없음. `aarch64-linux-android`는 진행 중(0.6.2) |
| `thumbv6m-none-eabi`, `thumbv7m-none-eabi`, `thumbv7em-none-eabi`, `thumbv7em-none-eabihf`, `thumbv8m.main-none-eabi`, `thumbv8m.main-none-eabihf`(Cortex-M) | 없음 |
| `x86_64-apple-ios`(Intel Mac의 iOS 시뮬레이터) | Apple Silicon Mac에서 `aarch64-apple-ios-sim`(0.6.1) |
| `x86_64-apple-darwin`(Intel Mac) | Apple Silicon Mac에서 `aarch64-apple-darwin` |

## 0.6.4. SIMD

빌드는 트리플의 baseline CPU를 대상으로 합니다. 그래서 바이너리를 다른 기기로 옮겨도 동작합니다. `x86_64`는 x86-64 baseline(SSE2), `i686`은 `pentium4`(SSE2), `aarch64-apple-darwin`과 `aarch64-apple-ios-sim`은 `apple-m1`, `aarch64-apple-ios`는 `apple-a7`(NEON, LSE atomic 없음), 그 밖의 `aarch64`는 armv8-a(NEON, LSE atomic 없음), `armv7`은 NEON이 있는 ARMv7, `powerpc64le`는 POWER8(VSX), `s390x`는 z10(벡터 기능 없음), `wasm32`는 SIMD128이 기준입니다. `-O0`보다 높은 수준에서 코드 생성기는 `x86_64`, `i686`, `aarch64`, `armv7`(정수 레인만), `powerpc64le`, `wasm32`에서 그 baseline으로, `s390x`와 `riscv64`에서는 그 벡터 기능이 켜지면 루프를 벡터화합니다([8.7.8](../008-abi/807-code-generation.md#878-루프-벡터화)). 리스트 원소를 도는 루프는, 융합된 점 식이 되는 루프 하나([3.3.4](../003-expressions/303-broadcast-operators.md#334-자동-루프-융합))도 포함해, 128비트 벡터로 돌 수 있습니다. 벡터화한 루프는 스칼라 루프가 계산하는 것을 계산하고, 스칼라 루프가 trap하는 곳에서 trap합니다. 그 밖의 타깃에서는 모든 루프가 스칼라 코드입니다. `-C target-cpu=<cpu>|native`와 `-C target-feature=+avx2,...`([9.1](../009-tools/901-nabic-cli.md))는 CPU의 기능을 고르고, 코드 생성기는 `x86_64`에서 그 가운데 일부를 씁니다. `avx2`(`x86-64-v3`, `haswell`, `znver1`과 그 뒤의 CPU)가 있으면 벡터 루프가 256비트 AVX2 벡터를 쓰고 조건이 레인을 고르는 곳에서는 마스크를 걸어 쓰며, `popcnt`, `lzcnt`, `bmi`가 있으면 비트 세기(`count_ones`, `leading_zeros`, `trailing_zeros`)가 각각 명령어 하나입니다. `fma`가 있으면 `mul_add`가 스칼라 코드에서도 벡터 루프에서도 `vfmadd` 명령어 하나이지만, 프로그램이 `mul_add`로 묶지 않은 곱셈과 덧셈은 하나로 합치지 않습니다. 합치면 반올림이 달라집니다. `aarch64`에서는 ARMv8.1 atomic인 `lse`(`apple-m1`, `cortex-a76`, `neoverse-n1`과 그 뒤의 CPU)를 씁니다. `lse`가 있으면 atomic read-modify-write, 교환, compare-exchange가 각각 명령어 하나(`ldadd`, `swp`, `cas` 등)입니다. `aarch64` Linux와 Windows의 baseline처럼 `lse`가 없으면 각각 배타적 load / store 루프(순서에 따라 `ldxr` / `stxr`나 `ldaxr` / `stlxr`)가 되며, 이 루프는 모든 ARMv8.0 CPU(Cortex-A53, Cortex-A72 등)에서 돕니다. `aarch64` Linux에서는 `-O0`보다 높은 수준이면 이런 연산마다 런타임이 시작할 때 커널의 `AT_HWCAP`로 정한 플래그를 보고, CPU에 LSE가 있으면 LSE 명령어를 씁니다. ARMv8.3 RCpc load인 `rcpc`(`apple-m1`, `cortex-a76`, `neoverse-n1`과 그 뒤의 CPU)가 있으면 `-O0`보다 높은 수준에서 `Acquire`를 요구하는 atomic load가 `ldapr`입니다. 앞선 release store가 이 load를 앞지를 수 있으며, acquire load에는 그래도 됩니다. 없으면 `ldar`입니다. `dotprod`(같은 CPU들)가 있으면 바이트의 곱을 더하는 루프(`i8`이나 `u8`의 `s += (a[i] as i32) * (b[i] as i32)`)가 곱 16개씩, 레인 하나에 넷씩 돕니다(`sdot`, `udot`). `fullfp16`, `crc`, `sve`, `sve2`는 코드를 바꾸지 않습니다. `s390x`에서는 `z13`과 그 뒤의 CPU가 가진 벡터 기능인 `vector`를 씁니다. 벡터 루프가 128비트이고 부동소수 레인은 8바이트이며, `vector-enhancements-1`(`z14`와 그 뒤)이 있으면 4바이트도 됩니다. baseline인 z10에는 벡터 레지스터가 없습니다. `riscv64`에서는 `v`(RVV 1.0, `-C target-feature=+v`)를 씁니다. 벡터 길이가 얼마든 모든 V 기기가 담는 128비트로 벡터 루프가 돕니다. `riscv64`와 `riscv32`에서는 `zbb`를 씁니다. `zbb`가 있으면 32비트나 64비트 정수를 상수만큼 회전하는 것(`rotate_left`, `rotate_right`, 또는 회전을 이루는 두 shift와 or)이 명령어 하나(`rori`)이고, 없으면 두 shift와 or입니다. `armv7`에서는 `vfp4`를 씁니다. `vfp4`가 있으면 `mul_add`가 `vfma` 명령어 하나입니다. `armv7`의 `-C target-feature=-neon`, `powerpc64le`의 `-vsx`, `wasm32`의 `-simd128`은 모든 루프를 스칼라로 빌드합니다(기본값인 `+simd128`은 SIMD128을 다시 켭니다). 그 밖의 기능, 그리고 `i686`의 모든 기능은 코드를 바꾸지 않으며, 그런 것만 준 빌드는 그렇다고 경고합니다. CPU 이름은 그 CPU의 기능을 뜻하고(`x86-64-v2`, `x86-64-v3`, `x86-64-v4`, `haswell`, `znver4`, `apple-m1`, `neoverse-n1`, `pwr9`, `z13` 등), 모르는 이름은 아는 이름을 늘어놓는 에러입니다. `native`는 `nabic`가 돌 때 감지한 빌드 기기의 기능을 baseline의 기능에 더합니다. 그 바이너리는 다른 기기에서 돌지 않을 수 있습니다. 그 플래그 말고는 생성된 코드가 런타임에 CPU 기능을 감지하지 않으며, 여러 버전으로 컴파일해 런타임에 고르는 것은 없고 `par::map`도 마찬가지입니다([stdlib 2.19.6](../../refs-standard-libraries/200-core/219-par.md#2196-simd--gpu-통합)).

이식 가능한 SIMD 벡터, 곧 레인별 연산자를 가진 `Simd<T, N>`은 `core::simd`입니다([stdlib 2.24](../../refs-standard-libraries/200-core/224-simd.md)). 그 연산은 위 타깃에서 모든 최적화 수준에서 벡터 코드이고, 그 밖에서는 레인 루프입니다.

## 0.6.5. 타깃 사양

모든 타깃은 타깃 사양 하나로 기술합니다. 컴파일러, 링커, 런타임 빌드는 타깃에 관해 이 사양 말고는 아무것도 읽지 않습니다. 타깃은 `--target`으로 지정합니다. 값은 두 가지 중 하나입니다. 하나는 내장 이름(`nabic --print target-list`: 0.6.1의 타깃)입니다. 다른 하나는 NABION 값 `TargetSpec { .. }`을 담은 파일 `<name>.nabion`의 경로입니다. 이 경우는 사용자 정의 타깃이고, 이름은 파일 이름에서 확장자를 뺀 부분(stem)입니다.

`nabic --print target-spec [--target=<t>]`는 타깃 사양을 이 형태로 출력합니다. 그래서 사용자 정의 타깃은 보통 내장 타깃을 복사해서 시작합니다. 사용자 정의 파일은 모든 필드를 적어야 합니다. 모르는 필드는 에러입니다. 사용자 정의 타깃의 런타임과 C 라이브러리(타깃이 함께 싣는 경우)는 toolchain의 `lib/nabi/targets/<name>/`에서 가져옵니다. 이 디렉터리가 없으면 링크하는 빌드는 그 디렉터리 이름을 알리며 실패합니다.

| 필드 | 의미 |
|------|------|
| `c_target`, `deployment_target` | clang이 받는 형태의 타깃 트리플(`NABI_CC`, `nabi bindgen`의 C 프런트엔드). 그 뒤에 붙는 최소 OS 버전(Mach-O, macOS에서는 `MACOSX_DEPLOYMENT_TARGET`, iOS에서는 `IPHONEOS_DEPLOYMENT_TARGET`이 덮어씀. Android에서는 API 레벨, `ANDROID_PLATFORM`이 덮어씀) |
| `arch`, `os`, `env`, `abi`, `family`, `endian`, `pointer_width` | `target_*` cfg 값([6.3.2](../006-meta-system/603-cfg.md)). 비어 있는 `env` / `abi` / `family`는 키를 두지 않음. `abi`는 Apple 시뮬레이터에서 `"sim"`, 그 밖에는 비어 있음 |
| `vendor` | 정보용 |
| `wide_align`, `max_atomic_width`, `c_abi` | `i64` / `u64` / `f64` 정렬, 가장 넓은 atomic, C 호출 규약([8.6.3](../008-abi/806-calling-convention.md)) |
| `cpu`, `features` | clang 이름으로 쓴 기본 CPU와 feature(0.6.4의 SIMD 기준선) |
| `object_format`, `linker`, `pre_link_args`, `crt_objects`, `late_link_args` | 오브젝트 형식(`ObjectFormat::MachO`, `Elf`, `Coff`, `Wasm`). `nabic`의 링커가 만드는 것(`Linker::MachO`, `Elf`, `Coff`, `Wasm`, 또는 `WasmComponent`: wasm 모듈을 WASI preview 2 component로 만든 것)이며, 링크 인자의 명령행 형식도 정함. 타깃 자체의 링커 인자, C 런타임 오브젝트, 입력 뒤의 인자나 파일 |
| `dynamic_linking`, `dynamic_loader`, `libc` | 공유 라이브러리 지원, ELF interpreter, C 라이브러리와 그 라이브러리가 toolchain에 들어 있는지 여부 |
| `rust_target` | 런타임을 빌드하는 Rust 타깃 |
| `exe_suffix`, `staticlib_prefix` / `staticlib_suffix`, `dylib_prefix` / `dylib_suffix`, `c_symbol_prefix` | 파일과 심볼 이름 |
| `stack_size`, `stack_probes`, `threads` | main 스레드의 스택, stack probing, 스레드 지원 |
| `native_backend`, `repl` | 코드 생성기가 이 타깃을 받는지, REPL 호스트가 이 타깃에서 도는지 |

타깃 사양에는 안정된 해시가 있습니다. 모든 빌드 캐시는 이 해시를 키로 씁니다.
