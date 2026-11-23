# 8.11. 코드 생성기 계약

이 문서는 코드 생성기가 쓴 오브젝트가 `nabic`의 링커로 링크되고, 런타임 위에서 실행되고, 다른 코드 생성기가 쓴 오브젝트와 섞이려면 코드 생성기가 무엇을 보장해야 하는지 모읍니다. 다른 코드 생성기란 `nabic` 자신의 것([8.7](807-code-generation.md))과 따로 설치하는 것(8.11.13)입니다. 대부분은 이 장의 다른 곳에서 정의합니다. 이 문서는 요구 사항마다 이름을 붙이고 정의로 링크합니다. `nabic`의 코드 생성기가 계약보다 더 하는 곳은 그렇다고 적습니다.

> [!NOTE]
> 0.2를 위한 초안입니다. 0.1에서는 `nabic` 자신의 코드 생성기가 유일하고, 아래 요구 사항은 모두 그것이 하는 일입니다. 8.11.13의 프로세스 인터페이스와 그것이 읽는 파일 형식([9.11](../009-tools/911-mir-format.md))은 구현되지 않았습니다. 0.1의 `nabic`는 별도 코드 생성기를 실행하지 않고, `-C codegen-backend`는 알 수 없는 codegen 옵션입니다.

## 8.11.1. 일의 분담

코드 생성기는 codegen unit 하나([9.2.16](../009-tools/902-aot-pipeline.md#코드-생성-단위))의 함수를 재배치 가능한 오브젝트 하나로 만듭니다. 오브젝트 둘 이상이 맞춰야 하는 것은 모두 코드 생성기가 실행되기 전에 `nabic`가 정하고, 그 결과를 unit의 nabi-mir와 함께 넘깁니다.

| `nabic`가 정하는 것 | 코드 생성기가 정하는 것 |
|---------------------|-------------------------|
| nabi-mir 최적화(인라이닝, [8.7.3](807-code-generation.md#873-최적화-수준)), 코루틴 상태 기계([8.10](810-lowering-coroutine-generic.md)) | 함수 하나 안의 명령어 선택, 레지스터 할당, 스케줄링, 프레임 배치 |
| unit, 그리고 각 unit이 정의하는 함수 | 8.11.6과 8.11.7의 규칙 안에서 하는 자체 최적화 |
| 모든 함수와 static의 심볼, 그 링키지와 가시성(8.11.3) | 자기 지역 심볼(8.11.3) |
| 모든 타입의 레이아웃(8.11.5) | |
| GC type id와 그 ScanTable([8.2](802-scan-table.md)) | |
| safepoint에 닿지 않는 함수, 진입에서 poll하는 함수, poll하는 back edge(8.11.7) | poll하는 edge 위에서 poll이 놓이는 자리 |
| 공유 unit: C `main`, static, `nabi.register_types`, lazy static 초기화 함수([9.2.16](../009-tools/902-aot-pipeline.md#코드-생성-단위)) | |
| 링킹([9.2.14](../009-tools/902-aot-pipeline.md#9214-링킹실행-파일의-경우)) | |

공유 unit은 언제나 `nabic` 자신의 코드 생성기가 컴파일합니다. 한 프로그램 안에 서로 다른 코드 생성기의 오브젝트가 있어도, 한 unit을 다른 코드 생성기로 다시 컴파일해도, 링크되고 똑같이 동작해야 합니다.

## 8.11.2. 입력

입력은 unit 하나의 최적화한 nabi-mir입니다([9.11](../009-tools/911-mir-format.md)). unit이 정의하는 함수와 그 본문, 그 함수들이 가리키는 모든 함수와 static의 선언, 그 함수들이 쓰는 타입, 레이아웃, GC 타입, 타깃, 그리고 빌드 옵션(`-O` 수준, overflow 검사, `-g`)이 들어 있습니다.

이 nabi-mir는 `--emit=mir`가 출력하는 것과 같습니다([9.9.1](../009-tools/909-debug-tools.md#991-중간-산출물-출력---emit)). 코루틴 suspend 지점은 없고(코루틴은 이미 프레임 객체 위의 상태 기계입니다, [8.10.1](810-lowering-coroutine-generic.md#8101-상태-기계)), 내림 나머지 `%%`도 없습니다(펼쳐져 있습니다). 코드 생성기는 각 문장을 nabi-mir에서의 의미대로 컴파일하고, [8.7.3](807-code-generation.md#873-최적화-수준)이 최적화기에 요구하는 것처럼 모든 값, 모든 trap, 모든 부수 효과를 프로그램 순서대로 지킵니다.

## 8.11.3. 심볼과 링키지

모든 함수와 static은 입력에 자기 심볼을 지니고 있습니다. 맹글링한 이름([9.5](../009-tools/905-symbol-mangling.md)), `#[export_name]`, 또는 `extern` 선언의 C 이름(i686 Windows에서는 stdcall 접미사 `@N` 포함)입니다. 코드 생성기는 이 이름을 그대로 쓰고 맹글링하지 않습니다. 심볼을 global, weak, hidden 중 어느 것으로 정의할지도 입력이 정한 그대로 따릅니다. 다른 unit이 부르는 함수는 global이고, 다른 패키지 코드의 사본은 `<symbol>.copy_Y<hash>_`라는 이름으로 그 패키지의 unit 사이에서만 global입니다.

`nabic`와 그 코드 생성기는 다음 파생 심볼도 씁니다.

| 심볼 | 정의하는 쪽 | 뜻 |
|------|-------------|-----|
| Nabi 함수의 `<symbol>` | 그 함수를 정의하는 unit | 본문. 숨은 `nabi_tls` 인자가 있는 Nabi 규약을 따릅니다([8.6.6](806-calling-convention.md#866-nabi-함수-사이의-호출-규약)) |
| `<symbol>.c` | `#[export_name]` 없는 `extern "C"` 함수를 정의하는 unit(있으면 그 export 이름) | 그 함수의 C 진입점(8.11.8) |
| `<symbol>.v` | 그 함수의 주소를 쓰는 모든 오브젝트, 지역 심볼로 | 함수 값이 부르는 thunk. `nabi_tls` 인자가 없고, 블록 주소를 직접 읽어 `<symbol>`을 부릅니다([8.6.11](806-calling-convention.md#8611-함수-포인터-abi)) |
| `<static>.state`, `<static>.init` | 공유 unit | lazy static의 상태 바이트와 초기화 함수(8.11.10) |
| `_Y<pkg>_gctype.<hash>` | unit 빌드의 공유 unit | GC type id를 담은 `u32`(8.11.6) |
| `nabi.scan.<hash>` | 공유 unit, weak이고 hidden | 내용으로 이름 붙인 ScanTable의 바이트([8.2.13](802-scan-table.md#8213-등록-시점)) |
| `nabi.register_types`, `nabi.register_types.<lib>` | 공유 unit | GC 타입과 static 루트 등록([8.2.13](802-scan-table.md#8213-등록-시점)) |
| `nabi.any.<hash>` | `any` 디스패처가 있는 unit, 라이브러리에서는 weak | `any` 메서드 디스패처([9.4.8](../009-tools/904-nabihir-bytecode.md#948-분리-컴파일을-위한-규칙)) |
| `nabi.str.<hash>` | 모든 오브젝트, 지역 심볼로 | 문자열 리터럴의 바이트와 그 뒤의 NUL 바이트 |
| `nabi_tls` | 런타임 | 스레드 로컬 블록([8.3](803-tls-slots.md)) |

코드 생성기가 스스로 만드는 다른 심볼은 모두 그 오브젝트의 지역 심볼입니다. `nabic`의 것은 `nabi.`로 시작하는 이름을 씁니다(`nabi.i386.udivmod64`, ..).

모든 심볼 이름의 바이트는 압축하지 않은 그대로 오브젝트에 있어야 합니다. `nabic`는 unit의 오브젝트를 패키지 hash 자리에 placeholder를 넣어 쓰고, 링크하기 전에 오브젝트의 바이트에서 `_Y<placeholder>_`를 `_Y<hash>_`로 바꿉니다([9.2.16](../009-tools/902-aot-pipeline.md#코드-생성-단위)).

문자열 리터럴의 바이트 뒤에는 `str`에 들지 않는 NUL 바이트가 있습니다. throw 위치([8.11.11](#81111-에러-panic-unwinding))와 `char*`로 넘기는 `str`은 그 바이트까지 읽습니다.

## 8.11.4. 호출 규약

- Nabi 함수 사이: [8.6.6](806-calling-convention.md#866-nabi-함수-사이의-호출-규약). Nabi 함수가 다른 오브젝트에서 받을 수 있는 호출은 모두 이 규약을 정확히 따릅니다. 마지막 인자로 숨은 `nabi_tls` 주소를 넘기는 것도 포함합니다. 64바이트를 넘는 aggregate는 주소로 넘기고, 앞에 오는 결과 포인터로 돌려받습니다. 다른 오브젝트가 닿을 수 없는 함수(지역 함수이고 값으로 쓰지 않는 함수)만 자기만의 규약을 쓸 수 있습니다.
- `extern "C"` 선언과 정의, C 함수 포인터를 통한 호출: aggregate 분류를 포함한 플랫폼 C ABI([8.6.1](806-calling-convention.md#861-기본-플랫폼-c-abi) ~ [8.6.3](806-calling-convention.md#863-struct-인자--반환-작은-struct-규칙)), 좁은 정수의 확장, i686 Windows의 `extern "system"`에 쓰는 stdcall, C 가변 인자([8.6.10](806-calling-convention.md#8610-가변-인자variadic-함수)). 입력은 이런 시그니처마다 `nabic`가 한 분류를 줍니다([9.11.9](../009-tools/911-mir-format.md#9119-함수)). 코드 생성기는 분류된 조각을 그 자리에 놓을 뿐 다시 분류하지 않습니다.
- `extern "C"` 호출의 `str` 인자는 `char*`로 넘깁니다. 호출 앞에서 `nabi_str_cstr(s)`를 불러 그것이 돌려준 포인터를 넘기고, 호출 뒤에 `nabi_str_cstr_done(s, p)`를 부릅니다. 포인터는 바이트 뒤에 NUL 바이트가 있으면 `s` 자신의 것이고, 아니면 pin한 사본입니다.
- `extern "nabi"` 런타임 선언: C ABI. 단 `str`은 값으로 넘기는 `(ptr, len)` C 구조체입니다([8.5](805-extern-symbols.md)).
- 함수 값: `{code, env}`. 환경이 첫 인자이고 `nabi_tls` 인자는 없습니다([8.6.11](806-calling-convention.md#8611-함수-포인터-abi), [8.6.12](806-calling-convention.md#8612-클로저-호출)).
- 던지는 함수는 carrier `(is_err, payload)`를 돌려줍니다([8.6.5](806-calling-convention.md#865-던지는-함수의-반환-값-기반-에러-전달)). nabi-mir에서는 입력이 준 타입의 평범한 반환값입니다.
- 스택 probe는 타깃의 `stack_probes`를 따릅니다([0.6.5](../000-meta/006-platform-targets.md#065-타깃-사양)). 한 페이지 이상인 프레임은 인라인으로든 `__chkstk`(Windows, `nabic`의 시작 오브젝트가 정의합니다)로든 각 페이지를 차례로 건드립니다. 런타임은 guard 범위의 fault를 스택 오버플로로 보고합니다([8.5.7](805-extern-symbols.md#스택-오버플로)).

## 8.11.5. 데이터 레이아웃

모든 nabi-mir 타입의 레이아웃은 타깃이 정하고, 입력은 타입마다 크기, 정렬, 필드 오프셋을 줍니다([9.11.6](../009-tools/911-mir-format.md#9116-타입)). 코드 생성기는 그대로 씁니다.

| nabi-mir 타입 | 레이아웃 |
|---------------|----------|
| `()` | 크기 0, 정렬 1 |
| `i8` `u8` `bool` `c_char` | 1바이트. `bool`은 0 또는 1 |
| `i16` `u16` | 2바이트 |
| `i32` `u32` `f32` `char` | 4바이트. `char`는 Unicode scalar value |
| `i64` `u64` `f64` | 8바이트. 타깃의 `wide_align`에 정렬(i686 Linux는 4, 나머지는 8) |
| `isize` `usize`, GC 참조, 함수 포인터, 해석되지 않은 word | 포인터 폭 word 하나 |
| `str` | word 둘 `(ptr, len)`. `ptr`은 GC 객체 안을 가리킬 수 있습니다 |
| 튜플, 구조체 | 필드를 순서대로 둔 C 레이아웃. `#[repr(packed)]`는 padding 없이 정렬 1, `#[repr(align(N))]`는 최소 `N` |
| `[T; N]` | `T` 크기의 원소 `N`개 |
| word `n`개짜리 union | word `n`개. 정렬은 가장 크게 정렬된 variant를 따르고 최소 word |
| 합, enum | `(i32 tag, payload)` 또는 niche 레이아웃([8.8.6](808-lowering-types.md#886-enum의-표현), [8.8.8](808-lowering-types.md#888-합-타입a--b의-표현)) |
| `any Tr` | `(u32 tag, payload word)`([8.8.9](808-lowering-types.md#889-any-슬롯의-표현)) |
| 함수 값 | `(code, env)` |

`s390x`를 뺀 모든 타깃은 little-endian입니다. GC 객체는 16바이트 헤더([8.1](801-object-header-abi.md)) 뒤에 payload가 오고, 크기는 [8.1.7](801-object-header-abi.md#817-헤더--payload-총-크기-계산)의 것입니다.

## 8.11.6. GC: 할당, 루트, barrier

**할당.** 코드 생성기가 할당하는 모든 GC 객체(nabi-mir의 `HeapBox`, `HeapAlloc`, `BufAlloc`)는 입력이 함께 준 type id를 받습니다. 상수이거나, unit 빌드에서는 `_Y<pkg>_gctype.<hash>`에 있는 `u32`입니다. word보다 넓지 않은 수, `bool`, `char`의 `HeapBox`는 아무것도 할당하지 않고 payload word가 곧 값입니다([8.2.7](802-scan-table.md#827-any-슬롯)). 할당은 [8.9.9](809-lowering-control.md#899-gc-할당-fast--slow-path)의 TLAB fast path이거나 `nabi_gc_alloc_slow(size, align, type_id)` 호출입니다. `LOS_THRESHOLD`(8 KiB) 이상이거나 8보다 크게 정렬된 객체는 언제나 이 함수를 부릅니다. fast path는 `type_id`와 `gc_word`를 쓰고 `gc_flags`는 0으로 둡니다. payload는 0입니다. 코드 생성기는 언제든 대신 `nabi_gc_alloc_slow`를 불러도 됩니다.

**루트.** 모든 safepoint(8.11.7)에서, 그 safepoint를 넘어 살아 있는 모든 지역 변수의 GC 참조가 들어갈 수 있는 모든 word는 메모리에 있고, `nabi_tls` 오프셋 0에 연결된 shadow stack 프레임에 등록되어 있습니다([7.4.1](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack), [8.9.12](809-lowering-control.md#8912-shadow-stack-프레임)).

- 프레임은 머신 스택(wasm32에서는 linear memory)에 있는 `{prev: ptr, count: usize, slots: [ptr; count]}`입니다. 슬롯마다 word 하나의 주소를 담거나, 슬롯 하나에 `(n << 1) | 1`을, 다음 슬롯에 붙어 있는 word `n`개 중 첫 word의 주소를 담습니다. `count`가 0이면 프레임의 슬롯은 하나이고, 읽기 전용 데이터에 있는 디스크립터 `{count: u16, entries: [u16; count]}`의 주소를 담습니다. 디스크립터의 항목은 슬롯의 항목과 같고, 주소 대신 프레임에서 그 word까지의 오프셋을 word 단위로 센 값의 두 배를 부호 있는 값으로 씁니다([8.9.12](809-lowering-control.md#8912-shadow-stack-프레임)). 지역 변수에 필요한 word는 입력이 그 타입의 루트 오프셋으로 준 것입니다. 어느 variant든 참조를 담을 수 있는 모든 word, `any`의 payload word, `str`의 포인터 word입니다.
- 프레임은 각 경로의 첫 safepoint 전에 연결하고, 연결한 경로의 모든 return 전에 풉니다. 경로가 합쳐지는 곳에서는 모두 연결했거나 모두 연결하지 않았습니다. 등록된 word는 수집이 읽을 수 있는 때에는 언제나 0이거나 유효한 값입니다. 연결 전에 0으로 채우고, 쓸 때마다 최신으로 둡니다.
- safepoint를 넘어 살아 있는 것: 그 뒤에 살아 있는 지역 변수. 런타임 함수나 C 호출에서는 그 피연산자도. Nabi 함수 호출에서는 호출자가 그 뒤에 읽는 것만입니다(피호출자가 자기 매개변수를 루트로 둡니다). 주소를 쓴 지역 변수(`AddrOf`)와 함수가 pin 목록에 둔 지역 변수는 `StorageDead`까지 등록된 채로 있습니다.
- back-edge poll만 보는 지역 변수는 대신 poll의 slow path에서만 등록해도 됩니다([7.4.1](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack) 규칙 4). `nabic`의 코드 생성기는 진입에서 프레임 하나를 연결하고 이런 지역 변수도 모두 거기에 등록합니다.
- 죽은 지역 변수의 word는 0으로 지워도 됩니다(`StorageDead`, 마지막 사용의 끝). 쓰레기를 붙잡지 않게 하려는 것이고 필수는 아닙니다.

**레지스터와 움직이는 객체.** 전체 수집은 객체를 옮길 수 있습니다(evacuation, [7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)). 힙의 정확한 참조는 모두 고쳐 쓰지만 루트 word는 고쳐 쓰지 않습니다. 등록된 word, ambiguous word, pin이 닿는 객체는 제자리에 있고, 큰 객체는 옮기지 않습니다. 그래서 다음을 지킵니다.

- GC 참조가 safepoint를 넘어 레지스터에 남을 수 있는 것은, 그 safepoint를 넘어 등록되어 있는 word의 사본일 때뿐입니다. 그 객체 안을 가리키는 포인터도 마찬가지입니다.
- 그 밖의 GC 참조나 GC 객체 안을 가리키는 포인터(힙 객체의 필드에서 읽은 참조 등)는 safepoint를 넘어 살아 있지 않습니다. 그 뒤에 다시 읽습니다. 코드 생성기는 이런 값을 safepoint를 넘어 등록되지 않은 레지스터, spill 슬롯, 스택 슬롯에 두면 안 됩니다.
- `nabic`의 코드 생성기는 루트를 레지스터에 두지 않습니다. safepoint를 넘어 GC 참조를 담는 변수는 등록된 자기 스택 슬롯에 있습니다.

**write barrier.** GC word가 있는 값을 GC 객체(힙 객체의 필드, 버퍼 원소, 코루틴 프레임 슬롯)에 저장할 때마다 [8.9.10](809-lowering-control.md#8910-write-barrier)의 barrier를 둡니다. 대상 객체의 `gc_flags & (UNLOGGED | CARDED)`를 검사하고, `UNLOGGED`이면 `nabi_gc_wb_slow(obj, slot, len)`(슬롯 주소와 저장하는 값의 크기)를 부르고, `CARDED`이면 쓴 슬롯이 걸치는 card 바이트에 `1`을 저장합니다. card 바이트는 `card_table + ((slot - obj) >> 9)`에 있고, card table의 주소는 헤더 바로 앞 word에 있습니다. barrier와 그 저장 사이에는 safepoint가 없습니다. 방금 할당한 객체에 다음 safepoint 전에 하는 저장, static에 하는 저장, raw 메모리에 하는 저장, GC word가 없는 값의 저장에는 필요 없습니다. barrier는 저장하는 값의 헤더를 읽지 않습니다.

수집기는 컴파일된 코드에 이 밖의 것을 요구하지 않습니다([7.4.1](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack)). stack map도 register map도 없습니다.

## 8.11.7. safepoint와 poll

safepoint는 스레드가 수집을 위해 멈출 수 있는 지점입니다. 입력이 safepoint에 닿지 않는다고 표시하지 않은 Nabi 함수 호출, 함수 값을 통한 호출, `#[no_safepoint]`가 아닌 런타임 함수 호출, C 호출(park합니다, 8.11.8), 할당, lazy static의 읽기, 쓰기, 주소, 그리고 poll입니다.

- **poll.** `nabi_tls + 10p+8`의 `u8`(`yield_flag`)을 읽고, 0이 아니면 `nabi_runtime_yield()`를 부릅니다([8.4.1](804-safepoint.md#841-yield-플래그-검사-fast-path)).
- **back edge.** 입력이 poll한다고 표시한 back edge에는 모두 poll이 있습니다([8.4.8](804-safepoint.md#848-루프-back-edge-poll)). 표시하지 않은 것은 safepoint에 닿지 않고 많아야 64번 도는 루프의 것입니다([8.4.2](804-safepoint.md#842-삽입-위치)).
- **진입.** 입력이 재진입 가능하다고 표시한 함수는 각 경로의 첫 safepoint 전에 poll합니다. 직접 Nabi 호출이 아닌 길(C, 함수 값, export, `any` 디스패처)로도 들어오고 safepoint에 닿기는 하는 함수라면 진입에서 poll합니다([8.4.7](804-safepoint.md#847-safepoint-불변식), [8.9.11](809-lowering-control.md#8911-safepoint-poll)).
- 입력이 safepoint에 닿지 않는다고 표시한 함수는 프레임도 poll도 필요 없고, 그 함수의 호출은 safepoint가 아닙니다.

poll을 더 두는 것은 괜찮습니다. 벡터화한 루프 안에는 poll이 없고, 스칼라 루프의 poll이 다시 실행되기 전까지 많아야 1024번 돕니다([8.7.8](807-code-generation.md#878-루프-벡터화)).

## 8.11.8. C로 가는 호출과 C에서 오는 호출

- **C로.** 심볼이 `nabi_`로 시작하지 않는 `extern "C"` 함수 호출마다 앞뒤를 감쌉니다. 앞에서는 `mutator_state`(`nabi_tls + 10p+9`)에 `2`(PARKED)를 sequentially consistent하게 저장하고, `yield_flag`를 읽어 0이 아니면 `nabi_enter_native()`를 부릅니다. 뒤에서는 `0`(RUNNING)을 저장하고, 같은 읽기를 하고, 0이 아니면 `nabi_leave_native()`를 부릅니다([8.4.10](804-safepoint.md#8410-네이티브-호출)). 호출자의 루트는 호출 내내 등록된 채로 있습니다.
- **C에서.** `extern "C"` 정의에는 C 진입점(`<symbol>.c` 또는 export 이름)이 있습니다. C 인자를 받고, `nabi_native_reenter()`를 부르고(런타임이 모르는 스레드면 붙이는 일도 합니다, [8.3.8](803-tls-slots.md#838-컴파일러-출력-형식)), 이 스레드의 `nabi_tls` 주소와 함께 본문을 부르고, `nabi_native_resume(<what reenter returned>)`을 부르고, C 방식으로 돌아갑니다. wasm32에서는 export한 함수이면 먼저 `nabi_init_once()`를 부릅니다.
- `#[no_safepoint]` 런타임 함수와 `nabi_*` 심볼의 호출은 감싸지 않습니다.

## 8.11.9. 스레드 로컬 접근

Nabi 함수는 마지막 인자로 받은 주소에서 각 오프셋([8.3.1](803-tls-slots.md#831-tls-슬롯-구조))의 `nabi_tls` 슬롯을 읽습니다. Nabi 밖에서 들어오는 코드(C 진입점, 함수 값 thunk, export한 Nabi 함수, `any` 디스패처)는 주소를 한 번 계산합니다([8.7.7](807-code-generation.md#877-tls-접근)). 접근 순서와 그에 따른 재배치는 오브젝트 형식에 따라 다릅니다.

| 타깃 | 모델 | `nabic`가 쓰는 재배치 |
|------|------|------------------------|
| ELF x86_64 / aarch64 / i686 / armv7 / riscv64 / powerpc64le / s390x | initial exec. 스레드 포인터에 GOT 슬롯의 오프셋을 더합니다 | `R_X86_64_GOTTPOFF`, `R_AARCH64_TLSIE_*`, `R_386_TLS_GOTIE`, `R_ARM_TLS_IE32`, `R_RISCV_TLS_GOT_HI20`, `R_PPC64_GOT_TPREL16_*`, `R_390_TLS_IEENT` |
| ELF riscv32(OS 없음) | local exec | `R_RISCV_TPREL_HI20` / `_ADD` / `_LO12_I` |
| Mach-O arm64 | 스레드 로컬 변수 descriptor. getter를 통해 부릅니다 | `ARM64_RELOC_TLVP_LOAD_PAGE21` / `_PAGEOFF12` |
| COFF | `_tls_index`에 있는 TEB의 TLS 배열에 섹션 기준 오프셋을 더합니다 | `SECREL`(arm64: `SECREL_HIGH12A` / `_LOW12A`) |
| 스레드가 있는 wasm32 | `__tls_base`에 오프셋을 더합니다 | `R_WASM_MEMORY_ADDR_TLS_SLEB` |
| 스레드가 없는 wasm32 | 평범한 전역 | 메모리 주소 |

런타임은 언제나 정적으로 링크하므로 이 모델로 충분합니다. 링커가 처리하는 곳이라면 코드 생성기는 대신 general dynamic을 써도 됩니다.

## 8.11.10. static

static의 저장 공간, 초깃값 바이트, 루트 등록은 공유 unit의 것입니다. 코드 생성기는 static을 그 심볼로 읽고 씁니다. 오브젝트 형식에 GOT가 있고 static이 프로그램 자신의 오브젝트 밖(prebuilt 라이브러리나 C 라이브러리)에 정의되어 있으면 GOT를 거치고, Windows에서 DLL이 정의한 static은 `__imp_<symbol>`을 거칩니다. static에 하는 저장에는 write barrier가 없습니다(static은 루트입니다).

lazy static을 읽거나, 쓰거나, 그 주소를 쓰기 전에 코드는 상태 바이트 `<static>.state`를 atomic하게 읽고, `2`(초기화됨)가 아니면 접근한 소스 위치(`path:line:col`)를 넘겨 `<static>.init(at: str)`을 부릅니다. 이 호출은 `nabi_tls` 인자가 없는 Nabi 규약을 따릅니다. 이것은 safepoint입니다.

## 8.11.11. 에러, panic, unwinding

Nabi 프로그램의 어떤 것도 머신 스택을 unwind하지 않습니다.

- `throw`와 `try`는 carrier에 대한 평범한 return과 분기입니다([8.6.5](806-calling-convention.md#865-던지는-함수의-반환-값-기반-에러-전달), [8.9.6](809-lowering-control.md#896-try--catch--finally-값-기반-분기)). 모든 프레임은 자기 return 경로로 나가며 shadow stack 프레임을 풉니다.
- `throw`는 `nabi_throw_site(at: str)`를 불러 자기 위치를 남깁니다. `nabic`의 코드 생성기는 호출 대신 리터럴의 주소를 런타임이 읽는 구현 정의 슬롯([8.3.1](803-tls-slots.md#831-tls-슬롯-구조))에 저장합니다. 코드 생성기는 함께 배포되는 런타임에 대해서만 이렇게 할 수 있습니다.
- panic은 돌아오지 않는 함수(`nabi_panic_at`, ..)의 호출입니다. 프로세스를 끝내고 `finally`는 실행하지 않습니다. 코드 생성기가 스스로 내보내는 검사는 `nabi_panic_at(msg, at)`으로 panic합니다. `at`은 실행 중인 문장의 위치(`path:line:col`)이고, `#[track_caller]` 함수에서는 마지막 매개변수입니다. 런타임은 [4.14.13](../004-functions/414-throws.md#41413-panic과-fatal-에러)처럼 출력합니다. 검사와 그 메시지는 다음과 같습니다.

| 검사 | 언제 | 메시지 |
|------|------|--------|
| 정수 `+` `-` `*` 오버플로 | overflow 검사가 켜져 있고, `wrapping_*`가 아니고, 피연산자로 안전이 확인되지 않을 때 | `attempt to add with overflow`, `attempt to subtract with overflow`, `attempt to multiply with overflow` |
| 최솟값의 정수 부정 | overflow 검사가 켜져 있을 때 | `attempt to negate with overflow` |
| 나눗셈과 나머지 | 언제나 | `attempt to divide by zero`, `attempt to divide with overflow`, `attempt to calculate the remainder with a divisor of zero`, `attempt to calculate the remainder with overflow` |
| 폭 이상의 시프트 | 언제나 | `attempt to shift left with overflow`, `attempt to shift right with overflow` |
| 배열 인덱스(`ArrayIndex`, `ArrayWrite`) | 인덱스가 범위 안의 상수가 아니면 언제나 | `index out of bounds` |

다른 검사(slice, `List`, `unwrap`)는 이미 nabi-mir 안의 호출입니다.

unwind 테이블은 프로그램이 아니라 디버거, 프로파일러, 운영체제를 위한 것입니다. 코드 생성기는 모든 함수에 오브젝트 형식의 unwind 정보를 내보냅니다. ELF에서는 `.eh_frame` FDE(armv7은 `.ARM.exidx` 항목), Mach-O에서는 compact unwind 또는 `__eh_frame`, COFF x86_64와 arm64에서는 `.pdata` / `.xdata`입니다(i686에는 없고, 프레임은 `ebp`로 따라갑니다). 어떤 CIE도 personality 루틴을 가리키지 않고, LSDA는 내보내지 않습니다. armv7의 compact `.ARM.exidx` 항목은 `__aeabi_unwind_cpp_pr0`를 암묵적으로 가리키고, 런타임이 이것을 weak으로 정의합니다. `nabic`의 코드 생성기는 모든 함수에 프레임 포인터를 두므로 프로파일러는 테이블 없이 스택을 따라갈 수 있습니다.

## 8.11.12. 오브젝트

코드 생성기는 `nabic`의 링커가 읽는 타깃 형식의 재배치 가능한 오브젝트 하나를 씁니다([9.2.14](../009-tools/902-aot-pipeline.md#9214-링킹실행-파일의-경우)).

| 형식 | 요구 사항 |
|------|-----------|
| ELF | `ET_REL`, 타깃 ABI의 machine과 flag(powerpc64le: ELFv2, `e_flags` 2. armv7: EABI5 hard-float. riscv: RVC, riscv64는 double-float ABI도). 64비트 타깃은 RELA 재배치만, 32비트는 REL 또는 RELA. 압축한 할당 섹션은 없습니다(`SHF_COMPRESSED` 디버그 섹션은 버립니다). `SHF_EXECINSTR` 없는 `.note.GNU-stack` 섹션 |
| Mach-O | `MH_OBJECT`, arm64만, `MH_SUBSECTIONS_VIA_SYMBOLS` |
| COFF | 타깃의 machine. i686의 C 이름에는 접두사 `_`가 붙습니다. 섹션이 65279개를 넘으면 `/bigobj` 형식 |
| wasm | WebAssembly tool conventions를 따르는 재배치 가능 모듈(`linking` 섹션 버전 2, `reloc.CODE` / `reloc.DATA`). `env`에서 `__linear_memory`, `__stack_pointer`, `__indirect_function_table`을, 스레드가 있으면 `__tls_base`도 import합니다. 각 함수의 타입은 그 규약이 정하는 것이고, `nabi_tls` 주소는 마지막 `i32` 매개변수입니다 |

- 함수와 데이터 객체는 각자 자기 섹션(ELF), COMDAT(COFF), subsection(Mach-O), 자기 segment(wasm)에 두는 것이 좋습니다. 링커는 참조되지 않는 것을 버리고, `release` 빌드에서는 같은 함수를 하나로 합칩니다([9.2.15](../009-tools/902-aot-pipeline.md#9215-산출물)). 섹션을 함께 쓰는 것들은 함께 남거나 함께 버려지고, 합쳐지지 않습니다.
- 링커가 처리하는 재배치 종류만 쓸 수 있습니다. `nabic`의 코드 생성기가 쓰는 것, 그리고 toolchain이 함께 배포하는 런타임 라이브러리와 C 라이브러리에 있는 것입니다. 그 밖의 것은 그 이름을 대는 링크 에러입니다(`unsupported relocation ..`).
- 명령어 집합에 없는 연산은 런타임 라이브러리가 제공하는 플랫폼 지원 루틴의 호출입니다. armv7에서는 `__aeabi_idiv`, `__aeabi_ldivmod`, `__aeabi_l2d`, `__mulodi4`, .., riscv32에서는 soft-float 루틴과 `__divdi3`, ..입니다([8.7.4](807-code-generation.md#874-연산)).
- `sqrt` 밖의 부동소수 메서드는 입력에 이미 있는 `nabi_f64_*` / `nabi_f32_*` 호출입니다. 코드 생성기는 이것을 컴파일 시간에 계산하지 않습니다.
- 출력은 결정적입니다. 같은 입력은 같은 바이트를 냅니다.

**디버그 정보**는 선택 사항입니다. 빌드에 `-g`가 있으면 코드 생성기는 [9.9.6](../009-tools/909-debug-tools.md#996-디버거-통합-gdb--lldb)처럼 함수를 기술하는 것이 좋습니다. 함수는 Nabi 경로로 이름 붙이고([9.5.10](../009-tools/905-symbol-mangling.md#9510-디버그-정보)), 줄 테이블은 문장의 위치에서 만들고, 매개변수와 이름 있는 지역 변수는 입력이 기술한 타입으로 적습니다. 링커는 32비트 형식의 DWARF 2 ~ 4를 읽고(`.dSYM` 번들은 DWARF 5나 64비트 DWARF인 오브젝트를 건너뜁니다), COFF에서는 `.debug$S` / `.debug$T`의 CodeView C13을 읽어 PDB로 합칩니다. 디버그 정보는 코드를 바꾸면 안 됩니다.

## 8.11.13. 따로 설치하는 코드 생성기

> [!NOTE]
> 0.2를 위한 초안입니다. 이 절의 어떤 것도 구현되지 않았습니다.

`nabic` 자신의 것이 아닌 코드 생성기는 별도 실행 파일이고, `nabi component add <component>`로 설치하는 toolchain 컴포넌트입니다. `codegen-llvm`은 `nabic-codegen-llvm`을 설치하고, 이 실행 파일은 버전이 고정된 LLVM을 정적으로 링크해 자기 안에 둡니다. `nabic` 자신은 다른 코드 생성기를 담지 않고 외부 코드 생성기를 링크하지 않습니다([9.2.17](../009-tools/902-aot-pipeline.md#9217-컴파일러와-런타임의-분리)).

**선택.** `-C codegen-backend=<name>`(기본값 `native`, 또는 `llvm` 같은 컴포넌트 이름), 또는 `nabi.nabion` profile의 `codegen_backend: "<name>"`([9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock))입니다. 옵션이 profile보다 우선합니다. 선택은 빌드가 컴파일하는 모든 unit에 적용됩니다. prebuilt 라이브러리는 이미 가진 오브젝트를 그대로 쓰고, 공유 unit과 REPL 셀, Jupyter 셀은 언제나 `native`입니다. 컴포넌트가 설치되지 않은 이름은 설치 명령을 알려 주는 에러입니다: ``codegen backend `llvm` is not installed; run `nabi component add codegen-llvm` ``. 컴포넌트는 `nabic`가 지원하는 모든 타깃을 컴파일하거나, 지원하지 않는 타깃을 알립니다(`E0129`).

**커널 컴포넌트.** 컴포넌트는 일부 함수만 컴파일할 수도 있습니다. `codegen-mlir`은 라이브러리가 자기에게 표시한 함수를 컴파일하고, unit의 나머지는 선택된 코드 생성기가 맡습니다. `nabic`는 이런 함수를 따로 unit에 둡니다. 함수를 표시하는 방법은 아직 정하지 않았습니다.

**찾기.** `nabic`는 자기 `bin/` 디렉터리(toolchain의 것, [9.2.19](../009-tools/902-aot-pipeline.md#9219-설치-레이아웃))에서 `nabic-codegen-<name>`을 실행합니다. `PATH`는 찾지 않습니다. 컴포넌트는 toolchain마다 설치하고, 버전은 toolchain의 버전과 같습니다.

**실행.** 프로세스 하나가 unit 하나를 컴파일합니다.

```
nabic-codegen-<name> <unit>.nabimir -o <unit>.o
```

- `<unit>.nabimir`는 [9.11](../009-tools/911-mir-format.md) 형식의 unit nabi-mir입니다. 빌드 옵션도 그 안에 있고, 다른 인자는 없습니다.
- `-o`는 쓸 오브젝트입니다. `nabic`는 종료 상태가 0일 때만 그것을 읽고, 다른 상태이면 무엇이 쓰였든 무시합니다.
- `nabic`는 코어 수만큼 프로세스를 동시에 실행하고, `--keep`이면 실패한 unit의 입력을 남깁니다([9.1.2](../009-tools/901-nabic-cli.md#nabic-build-aot-빌드)).
- 작업 디렉터리는 빌드의 것이고, 환경은 `nabic`의 것입니다.

**진단.** 프로세스는 진단을 표준 출력에 한 줄에 JSON 객체 하나씩, `--message-format=json` 형식(schema 1, [9.9.4](../009-tools/909-debug-tools.md#json-형식도구-통합))으로 씁니다. span은 입력 문장의 위치입니다. `nabic`는 이것을 빌드가 요청한 형식으로 자기 진단처럼 출력합니다. 컴파일할 수 없는 구성은 그것을 쓰는 문장에서 그 이름을 대는 `E0129` 에러입니다. 다른 코드 생성기로 조용히 넘어가는 일은 없습니다. 표준 에러는 그대로 전달합니다.

**종료 상태.**

| 상태 | 뜻 |
|------|-----|
| `0` | 오브젝트를 썼습니다 |
| `1` | 에러를 표준 출력에 보고했습니다 |
| `2` | 사용법 에러, 또는 컴포넌트가 읽지 못하는 입력 형식 버전(두 버전을 모두 알립니다) |
| `70` | 컴포넌트의 내부 에러 |

**버전.** `nabic-codegen-<name> --version`은 첫 줄에 `nabic-codegen-<name> <version>`을, 그리고 읽는 입력 형식 버전을 알리는 `mir: <major>.<minor>` 줄을 출력합니다. `nabic`는 다른 toolchain 버전의 컴포넌트를 거부하고, 다시 설치하는 명령을 알려 주는 에러를 냅니다. 컴포넌트의 정체(버전 줄과 파일의 stamp, [9.4.4](../009-tools/904-nabihir-bytecode.md#944-헤더와-producer)의 `nabic`와 같은 방식)는 모든 unit의 키에 들어갑니다([9.2.16](../009-tools/902-aot-pipeline.md#코드-생성-단위)). 그래서 컴포넌트가 바뀌면 그것이 컴파일한 unit을 다시 컴파일합니다.

## 8.11.14. 검증

코드 생성기는 [8.7.9](807-code-generation.md#879-변환-검증)의 end-to-end 테스트로, 그 오브젝트를 프로그램에 넣고 GC 스트레스 설정(`NABIR_GC_STRESS`, `NABIR_GC_EVAC=always`, [9.9.8](../009-tools/909-debug-tools.md#998-메모리--gc-도구))과 함께 검증합니다. 이 설정에서는 모든 할당이 수집할 수 있고, 모든 전체 수집이 옮길 수 있는 객체를 모두 옮깁니다. 두 코드 생성기의 오브젝트를 링크한 프로그램은, 각 unit을 어느 쪽에서 가져오든 한 코드 생성기로 만든 프로그램처럼 동작합니다.
