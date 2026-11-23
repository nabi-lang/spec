# 0.1. 3계층 모델

Nabi 표준 라이브러리는 세 계층(tier)으로 나뉩니다. 계층마다 의존할 수 있는 영역이 다르고, 사용자 코드에서 가져오는 규칙도 다릅니다.

```
bare   (base, runtime ABI extern "nabi" 만 의존)
  ↓
core     (foundation, 안전, 모든 target; uses bare internally)
  ↓
std   (target-dep; uses foundation + bare)
```

각 계층은 자기 경로 루트입니다. `bare::..`, `core::..`, `std::..`([refs-lang 5.7.4](../../refs-lang/005-modules/507-paths.md#574-std-core-bare-표준-라이브러리-계층)).

세 층으로 나누는 이유는 분명합니다. 메모리 안전성을 깨뜨릴 수 있는 원시 연산은 좁은 영역에 가두어야 감사하기 쉽습니다. 모든 플랫폼에서 똑같이 동작해야 하는 핵심 추상은 OS 의존 코드와 섞이지 않아야 이식성을 보장할 수 있습니다. 플랫폼마다 쓸 수 있는지가 다른 기능은 그 사실을 시그니처와 import 지점에서 드러내야 합니다. 그래야 사용자가 wasm 빌드와 네이티브 빌드를 일관되게 다룰 수 있습니다. 세 계층이 이 세 요구를 하나씩 맡습니다.

## 0.1.1. 계층별 책임

### `bare`

맨 아래 계층인 `bare`는 런타임의 `extern "nabi"` 심볼에만 의존합니다. 이 계층은 `Ptr<T>`, `ConstPtr<T>`, `transmute`, 원시 할당, pin/unpin, volatile 같은 원시 도구를 내놓습니다. 그래서 쓰려면 항상 `use`를 명시해야 합니다. 명시 자체가 위험 신호입니다. 코드 리뷰에서 위험한 영역이 한눈에 보이도록 일부러 둔 장치입니다.

```nabi
use bare::ptr::CPtr;
use bare::alloc::{alloc, Layout};
let layout = try Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // 실패 시 null (1.4 alloc 참고)
```

### `core`(foundation)

가운데 foundation 계층인 `core`에는 `core::collections`, `core::iter`, `core::ops` 같은 모듈이 있습니다. 이 계층은 내부에서만 `bare`에 의존합니다. 숫자 타입, 산술 트레잇, iter, 컬렉션, fmt, 변환, sync, task, future, 바이트 스트림 I/O 등 언어를 일상적으로 쓰는 데 필요한 추상은 거의 모두 이 층에 있습니다. 사용자 코드에는 **prelude `use core::prelude::*;`가 묵시적으로 자동 적용**되므로 따로 가져오지 않고 바로 쓸 수 있습니다. 자세한 정책은 [0.2 import 정책](002-import.md)에 있습니다.

```nabi
let xs: List<i32> = [1, 2, 3];        // List 직접 사용, prelude 가 가져옴
let r = a + b;                         // Add 트레잇 자동
let m = Mutex::new(0);                 // sync::Mutex 도 자동
task::spawn(async { /* ... */ });              // task 모듈도 자동
```

prelude는 정해 둔 이름 목록입니다([0.2.3](002-import.md)). `core`의 나머지는 prelude 모듈을 거쳐 부르거나(`task::sleep`, `ops::Add`) 가져옵니다.

### `std`

맨 위 계층인 `std`는 필요에 따라 foundation과 `bare`를 모두 씁니다. 이 층에는 `fs`, `net`, `time`(벽시계와 부팅 시계), `thread`(OS 스레드), `env`, `process`, `path`, `random`처럼 플랫폼에 의존하는 항목이 있습니다. 타깃마다 쓸 수 있는 항목이 다르므로 사용하는 쪽은 항상 명시적으로 가져와야 합니다. 그래야 어떤 플랫폼 기능을 쓰는지 코드에 드러납니다.

```nabi
use std::fs;
use std::time::SystemTime;
```

## 0.1.2. 의존성 규칙(강제)

허용하는 의존성은 다음과 같습니다.

| 의존하는 측 | 의존 가능 대상                                          |
|-------------|---------------------------------------------------------|
| `bare`  | runtime ABI `extern "nabi"`                              |
| `core`  | `bare`                                              |
| `std`  | `core`, `bare`                                  |
| 사용자 코드 | `core` (묵시), `std` (명시), `bare` (명시) |

다음 방향의 의존은 **금지**합니다. foundation(`core`)은 `std`를 가져오면 안 됩니다. foundation은 모든 타깃에서 빌드할 수 있어야 하므로, 플랫폼 의존 코드를 끌어들이는 순간 이식성이 무너집니다. `bare`도 foundation이나 `std`를 가져오면 안 됩니다. 저수준 층이 고수준 층에 의존하기 시작하면 계층 구조 자체가 의미를 잃습니다. 마지막으로 개별 `std::*` 모듈끼리 순환 의존하면 안 됩니다. 그래서 std 모듈 하나하나는 독립적으로 빌드할 수 있는 단위로 남습니다.

이 규칙이 다스리는 대상은 계층이 직접 적는 것, 곧 그 계층의 `use`와 경로입니다. 각 계층 내부의 `use core::**`는 여전히 그 계층에 언어 어휘를 공급합니다. bare 자신의 시그니처와 impl이 쓰는 0.4의 slot 항목, `List`, `Slice`, `Result`, `panic`, 그리고 bare의 타입이 구현하는 트레잇 `Debug` / `Formatter` / `FmtError`, `Hash` / `Hasher`, `Error`, `AsBytes`와 `Utf8Error`, `str::from_utf8`이 여기에 들어갑니다(1.6, 1.7).

`core`가 `std`의 항목을 이름으로 적는 곳은 prelude 하나뿐입니다. `core::prelude`는 해당 타깃에서 `std::os`의 Ext 트레잇을 나열하므로(0.2.3) 그 메서드는 `use` 없이 씁니다. 이것은 암시적 가져오기(0.2)이지 core 자체 코드의 의존이 아니며, core 코드는 여전히 `std` 없이 빌드됩니다.

## 0.1.3. 빌드 단위

세 계층은 각각 별도 빌드 단위(별도 아카이브)이고, 컴파일 단위로 캐시할 수 있습니다. 링크 순서는 `std` → `core` → `bare` → 런타임이고, 사용자 패키지가 그 맨 위에 놓입니다. 컴파일러가 처음 쓸 때 캐시에 빌드하고 자동으로 링크하므로 사용자가 의존성 선언에 추가하지 않아도 됩니다([0.5.4 제공 형태와 링크 메커니즘](../../refs-lang/000-meta/005-standard-libraries-tiers.md#054-제공-형태와-링크-메커니즘) 참고).

## 0.1.4. 타깃 가용성

`bare`와 foundation(`core`)은 wasm32를 포함한 [지원하는 모든 타깃](../../refs-lang/000-meta/006-platform-targets.md)에서 빌드됩니다. 반면 `std`는 타깃에 따라 쓸 수 있는 범위가 다릅니다. 예를 들어 `wasm32-wasip2`와 브라우저는 스레드 하나로만 동작하므로 `std::thread`로 스레드를 시작할 수 없습니다(`wasm32-wasip1-threads`에는 스레드가 있습니다). `std::fs`도 wasm32-unknown-unknown에서는 쓸 수 없고 WASI 타깃(`wasm32-wasip2`, `wasm32-wasip1-threads`)에만 있습니다. 가용성은 각 `std::*` 모듈 사양의 가용성 절에 적습니다. 그래도 API 자체는 모든 타깃에서 빌드됩니다. 타깃에 없는 연산은 실행 중에 `IoErrorKind::Unsupported`로 실패합니다. 브라우저에서 stdout / stderr는 호스트 콘솔로 가고, stdin은 EOF 상태입니다. 예외는 한 OS 계열에만 속한 것입니다. 그것은 해당 계열의 타깃에만 있는 `std::os::<family>`(`unix`, `linux`, `windows`)에 있으므로, 다른 곳에서 쓰면 필요한 `#[cfg]`를 알려 주는 컴파일 타임 에러가 납니다([3.10](../300-std/310-os.md)).

## 0.1.5. 진화 정책

세 계층은 안정성 정책도 다릅니다. `bare` API는 안정적이지 않고 컴파일러, 런타임과 함께 바뀝니다. 그래서 사용자 코드는 버전을 고정(pin)하기를 권장합니다. foundation(`core`) API는 SemVer 안정성을 목표로 하고, 호환되지 않는 변경은 breaking 릴리스(1.0 전에는 `0.x.0`)를 기다립니다. `std` API는 타깃별 가용성 차이만큼 변동이 크지만, 시그니처 자체는 안정적으로 유지합니다.

한 릴리스 시리즈 안에서는 항목을 없애거나 호환되지 않게 바꾸지 않습니다. 대체되는 항목은 계속 동작하고, `#[deprecated(since = "..", note = "..")]`로 표시합니다([6.2.7](../../refs-lang/006-meta-system/602-built-in-attributes.md#627-deprecated-사용-자제-표시)). 메시지에는 대신 쓸 것을 적습니다. `unsafe`가 되어야 하는 안전한 함수는 `#[deprecated_safe]`로 표시합니다([6.2.13](../../refs-lang/006-meta-system/602-built-in-attributes.md#6213-deprecated_safe-지금은-안전-나중에-unsafe)). 다음 호환성이 깨지는 릴리스에서 그 항목을 없애거나 `unsafe`로 만듭니다.

## 0.1.6. 항목마다 경로 하나

모든 항목의 경로는 하나입니다. 모듈은 이름공간처럼 공개 하위 모듈과 비공개 하위 모듈로 나뉩니다. 공개 하위 모듈은 사용자가 이름으로 적는 이름공간이고(`core::num::niche`, `core::crypto::ed25519`, `std::net::tls`, `std::os::unix::fs`), 부모 코드의 일부만 담는 하위 모듈은 비공개이며(`core::io`의 `mod buffer;`) 그 항목은 부모가 `pub use`로 다시 내보냅니다. 항목의 경로는 가는 길의 모든 모듈이 공개이면 정의한 모듈의 경로이고, 아니면 그 항목을 다시 내보내는 부모의 경로입니다. `BufReader`는 `core::io::BufReader`, `TcpStream`은 `std::net::TcpStream`, `sleep`은 `core::task::sleep`입니다. 같은 항목에 둘째 경로를 주는 꼴은 두 가지뿐입니다. 부모 모듈이 공개 자식 모듈을 펼치는 꼴(`core::collections::list::List`에 대한 `core::collections::List`, `core::crypto::Secret`, `bare` 루트의 캐스트)과 암시적 가져오기(0.2)입니다. 암시적 가져오기는 해당 타깃에서 `std::os`의 Ext 트레잇도 이름으로 적습니다(0.1.2). 어떤 모듈도 부모나 형제, 다른 계층의 항목을 다시 내보내지 않습니다. `Duration`은 `core::time::Duration`일 뿐이고, `SeekFrom`은 `core::io::SeekFrom`, `size_of`는 `bare::mem::size_of`입니다. 문서와 진단이 항목의 경로를 보일 때는 가장 짧은 공개 경로를 쓰고, 비공개 모듈을 거치는 경로는 쓰지 않습니다(`std::net::WriteHalf`). 표준 라이브러리 자체의 테스트가 이 밖의 `pub use`를 거부합니다.
