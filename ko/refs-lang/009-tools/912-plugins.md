# 9.12. 플러그인

*플러그인*은 Nabi 프로그램이 실행 중에 로드하는 공유 라이브러리로 빌드한 라이브러리 패키지입니다([3.11 `std::plugin`](../../refs-standard-libraries/300-std/311-plugin.md)). `cdylib`(9.3.3)와 달리 플러그인에는 자기 런타임이 없습니다. 플러그인은 로드한 프로그램의 런타임 위에서 돌기 때문에 그 값이 곧 프로그램의 값이고(GC 힙 하나, 스케줄러 하나, static마다 사본 하나), 모든 `pub` Nabi 함수를 어떤 Nabi 값으로든 경계 너머에서 부를 수 있습니다.

## 9.12.1. 모델

프로세스에는 런타임이 하나 있고, 그것은 *호스트*, 즉 플러그인을 로드하는 프로그램에 있습니다. 플러그인은 런타임도, 호스트에서 가져오는 패키지도 링크하지 않습니다. 대신 모듈 디스크립터([8.12](../008-abi/812-module-abi.md))를 가지며, 그 임포트 테이블에는 쓰는 런타임 엔트리, 표준 라이브러리 static, 의존성 심벌이 모두 이름으로 적혀 있습니다. Nabi 모듈 로더는 플러그인을 로드할 때 각각을 이름으로 호스트에(또는 먼저 로드된 플러그인에) 묶습니다. 운영체제의 로더는 파일을 매핑하고 C 심벌(C 라이브러리와 `#[link]` 라이브러리)만 해결합니다.

`std::plugin`에 닿는 프로그램이 호스트이며, 따로 선언할 것은 없습니다. 호스트는 런타임 전체를 남기고 자기 Nabi 심벌을 내보내기 테이블에 적습니다. 이것은 호스트만 가지므로 플러그인을 로드하지 않는 프로그램에는 비용이 없습니다.

| | cdylib | plugin |
|-|--------|--------|
| 로드하는 쪽 | 아무 C 호스트 | Nabi 프로그램 |
| 런타임 | 자기 사본, 로드될 때 시작 | 호스트의 것 |
| 인터페이스 | `pub extern "C" fn`, `#[repr(C)]` 타입 | 모든 `pub fn`, 모든 Nabi 값 |
| GC 힙, 스케줄러, static | 따로 | 호스트와 공유 |

## 9.12.2. 빌드

```nabion
Package {
    name: "enemies",
    version: "0.1.0",
    lib: { kind: "plugin" },
    deps: {
        "engine": "^0.3",                             // 호스트에서 가져옴
        "noise": Dep::version("^1.2", bundle: true),  // 플러그인에 링크됨
    },
}
```

- `lib: { kind: "plugin" }`이면 `nabi build`가 `--crate-type=plugin`을 넘깁니다([9.1](901-nabic-cli.md)). 결과는 `target/<profile>/libenemies.dylib`(macOS), `libenemies.so`(Linux), `enemies.dll`(Windows)이며, 여느 라이브러리처럼 `libenemies.a`와 `enemies.nabihir` 옆에 놓입니다. `std::plugin::file("enemies")`는 타깃에 맞는 파일 이름을 줍니다.
- 플러그인 패키지에는 `src/lib.nabi`가 있고 `src/main.nabi`는 없습니다(`` a plugin is loaded by a program, so move main.nabi to a package of its own that loads this one with `std::plugin::load` ``). 플러그인에 `nabi run`을 하면 오류입니다.
- 의존성은 기본으로 호스트에서 가져옵니다. 플러그인은 그것에 대해 컴파일되고, 그 static은 호스트의 것이며, 디스크립터는 그것을 `needs`에 적습니다. `bundle: true`이면 의존성을 플러그인에 링크합니다. 그러면 그 static은 플러그인 자신의 것이고 `provides`에 적힙니다. 플러그인이 아닌 패키지의 의존성에 `bundle`을 쓰면 오류입니다.
- 프로그램의 의존성 `Dep::path("../physics", plugin: true)`는 프로그램의 target 디렉터리에(그래서 프로그램 옆에, 프로그램과 같은 프로필과 타깃으로) 플러그인으로 빌드되고 링크되지 않습니다. 프로그램은 `plugin::load(plugin::beside("physics"))`로 그것을 로드합니다. 한 의존성에 `plugin: true`와 `bundle: true`를 함께 쓰면 오류입니다.
- 플러그인이 닿는 표준 라이브러리 코드는 플러그인 안에 컴파일되고(프로그램처럼 닿는 것만, [9.3.6](903-package-archive.md#936-표준-라이브러리)), 그 static은 호스트의 것입니다. 한 프로세스의 모든 모듈은 같은 툴체인으로 빌드되므로(9.12.6) 코드가 겹쳐도 문제가 없습니다.
- 제네릭 인스턴스는 그것이 필요한 모듈에 컴파일됩니다. `List<Orc>`는 `Orc`를 정의하는 플러그인에 있습니다. 제네릭 코드에는 자기 static이 없으므로 그 사본들은 서로 바꿔 써도 됩니다.
- 플러그인과 호스트에서는 코드젠 유닛([9.2.16](902-aot-pipeline.md#9216-증분-빌드))을 쓰지 않습니다. 그 디스크립터에 실제 패키지 해시가 들어가기 때문입니다.

## 9.12.3. 정체성

두 모듈에서 *타입 키*가 같으면 같은 타입입니다. 타입 키는 패키지의 내용 해시, 경로, 제네릭 인자의 키로 이루어집니다(`Y3f8a1c2b:engine::Body`, [8.13](../008-abi/813-type-descriptor.md)). 패키지 해시는 내용 해시이므로([9.5](905-symbol-mangling.md)) 호스트와 다른 `engine` 소스로 빌드한 플러그인은 다른 키를 가지고, 서로 다른 두 `Body` 레이아웃을 섞는 대신 패키지가 없다는 오류로 로드에 실패합니다.

- 심벌은 맹글링된 이름으로 묶입니다. 플러그인의 `_Y3f8a1c2b_engine_spawn_..` 임포트는 패키지 `3f8a1c2b`를 제공하는 모듈에만 묶입니다.
- `any` 태그는 프로세스 단위로 정해집니다. 호스트는 컴파일 시점 태그를 등록하고, 플러그인의 코드는 로더가 채우는 디스크립터의 태그 테이블에서 태그를 읽습니다(8.12.6). 프로세스가 이미 본 키는 그 태그를 유지하므로, 플러그인이 `any`에 저장한 `engine::Body`는 호스트의 `engine::Body`입니다.
- GC 타입 id는 구조적이며 모듈마다 따로입니다. 로더는 플러그인마다 범위를 줍니다.
- 두 모듈이 모두 기술하는 타입(`Inspect` 타입, 제네릭 인스턴스)은 레이아웃 해시가 같아야 합니다. 다르면, 흔히 의존성을 다른 기능으로 빌드한 경우인데, 로드를 거부합니다(`` engine::Body is laid out differently in the program and the plugin; build both with the same features of its package ``).

## 9.12.4. 로드와 호출

`plugin::load(path)`는 파일을 매핑하고, 모든 스레드를 잠깐 멈춘 상태에서 디스크립터를 등록합니다. 툴체인과 타깃을 확인하고, `needs`의 패키지가 모두 제공되는지 확인하며(빠진 것은 모두 한 오류로 알립니다), 임포트를 묶고, GC 타입 id와 `any` 태그를 주고, 루트, 타입 디스크립터, `any Tr` 메서드 본문을 등록한 뒤 플러그인을 공개합니다. 플러그인에는 초기화 함수가 없습니다. 그 static은 어디서나처럼 처음 쓸 때 초기화됩니다.

`p.get::<F>("enemies::spawn")`은 경로로 제네릭이 아닌 `pub` 함수를 찾고 시그니처가 정확히 `F`인지 확인합니다. 양쪽 모두 타입 키로 쓴 시그니처 텍스트(`fn({engine::World},i32)->List<..>`)를 해시하므로, 맞지 않으면 크래시가 아니라 `WrongType`입니다. 결과는 평범한 함수 값입니다. 제네릭 함수를 `get`하면 찾지 못하며(`NoSuchItem`, 메시지가 구체적인 `pub fn`을 내보내라고 알려 줍니다), `p.get_static::<T>(path)`는 static을 읽습니다.

경계를 넘을 때:

- `str`, `List`, `Dict`, 클로저를 비롯한 모든 GC 값은 하나의 힙을 가리키는 참조이며 양쪽으로 그대로 넘어갑니다.
- 클로저는 그것을 만든 플러그인을 클로저가 수거될 때까지 매핑된 채로 둡니다(9.12.5).
- `any`와 `any Tr`은 그대로 넘어갑니다. `any Tr` 메서드의 디스패처는 자기 모듈의 타입을 압니다. 모르는 태그는 프로세스에 묻고(`nabi_any_dispatch`, 8.12.6), 값을 저장한 모듈의 디스패처가 불립니다. 플러그인을 로드하지 않는 프로그램은 [2.2.10](../002-types/202-any-slot.md)의 닫힌 디스패처를 그대로 씁니다.
- 합 타입, 열거형, `throws` 오류는 구조적 태그를 가진 값이므로 그대로 넘어갑니다. 플러그인 안의 panic은 다른 곳에서처럼 프로세스를 끝냅니다.

## 9.12.5. 언로드

`p.unload()`는 미뤄집니다.

1. 로드된 다른 플러그인이 `p`에서 임포트하면 무엇이 쓰는지와 함께 `Pending`을 돌려줍니다. `p`가 자기 코드의 C 콜백을 내준 적이 있으면(`NO_UNLOAD`) `Kept`를 돌려줍니다. 그런 플러그인은 매핑된 채로 남습니다.
2. 그렇지 않으면 `p`는 바로 조회에서 빠지고(`get`과 `plugin::loaded()`가 더는 보지 못합니다) *은퇴* 상태가 됩니다.
3. 이후의 전체 수거가 은퇴한 플러그인이 아직 살아 있는지 확인합니다. 그 GC 타입의 객체, 코드가 그 안에 있는 함수 값이나 코루틴, 스택 위의 그 프레임이 살아 있음의 표시입니다. 아무것도 남지 않으면 루트, 타입, 메서드 본문을 내리고 파일을 매핑 해제합니다. 이미 그렇게 되었으면 `unload`는 `Done`을 돌려줍니다.

`plugin::wait_unloaded(p, ms)`는 플러그인이 없어지거나 시간이 다 될 때까지 수거합니다(테스트와 도구용).

## 9.12.6. 안정성과 오류

툴체인 빌드를 넘어 안정적인 플러그인 ABI는 없습니다. 한 프로세스의 모든 모듈은 같은 툴체인 빌드에서 나와야 합니다. 디스크립터의 `toolchain`은 컴파일러 id이며 정확히 비교합니다(`` the plugin was built by nabic 0.1.3 (a1b2c3), this program by nabic 0.1.4 (d4e5f6); rebuild the plugin with this program's toolchain ``).

| `PluginError` | 언제 | 메시지가 알려 주는 할 일 |
|---|---|---|
| `NotFound` | 경로에 파일이 없음 | `nabi build`가 쓴 파일을 줄 것 |
| `NotAPlugin` | Nabi 디스크립터가 없는 파일 | 패키지를 `lib: { kind: "plugin" }`으로 빌드하거나, C 라이브러리는 `bare::ffi`로 로드할 것 |
| `Toolchain` / `Target` | 다른 컴파일러 id / 타깃 | 플러그인을 다시 빌드할 것 |
| `MissingPackages` | 프로세스에 없는 `needs`의 패키지 | 프로그램에 추가하거나 플러그인에서 `bundle: true` |
| `MissingSymbols` | 아무도 제공하지 않는 임포트 | 프로그램의 패키지에 대해 플러그인을 빌드할 것 |
| `Layout` | 한 키에 두 레이아웃 | 둘을 같은 기능으로 빌드할 것 |
| `NoSuchItem` | 경로에 `pub fn` / static이 없음 | 비슷한 이름. 구체적인 함수를 내보낼 것 |
| `WrongType` | 다른 시그니처 | 써야 할 타입 |
| `NotLoaded` | 언로드된 플러그인 | 다시 로드할 것 |
| `Os` | 시스템 로더 실패 | 그 메시지 |

## 9.12.7. 타깃

| 타깃 | 파일 | 상태 |
|---|---|---|
| aarch64-apple-darwin | `MH_DYLIB`, 설치 이름 `@rpath/lib<name>.dylib`, ad hoc 서명, `_nabi_module`만 내보냄 | 지원. hardened runtime 호스트가 남이 서명한 플러그인을 로드하려면 `disable-library-validation` 권한이 필요합니다 |
| x86_64-unknown-linux-gnu, aarch64-unknown-linux-gnu | `ET_DYN`, `nabi_module`만 내보냄 | 지원 |
| x86_64-pc-windows-msvc, aarch64-pc-windows-msvc | `nabi_module`을 내보내고 자기 엔트리를 가진 DLL. `LoadLibraryExW`로 로드 | 지원. 로드된 DLL 파일은 잠깁니다 |
| *-linux-musl | | 오류. C 라이브러리가 정적으로 링크되어 코드를 로드할 수 없습니다(`` build plugins and the program for `x86_64-unknown-linux-gnu`, or build one program ``) |
| wasm32-* | | 오류: `` `--crate-type=plugin` needs a native target ``. `std::plugin`에 닿는 wasm 프로그램도 같은 식으로 링크에 실패합니다(`` on wasm, build one program: make the plugin's package an ordinary dependency ``) |
| 그 밖의 타깃 | | 아직 링크하지 않음 |

`NABI_CC`로는 플러그인과 호스트를 링크할 수 없습니다. 임포트 스텁과 테이블을 만드는 Nabi 자체 링커가 필요합니다.
