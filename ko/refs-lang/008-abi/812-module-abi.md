# 8.12. 모듈 디스크립터 ABI

*모듈*은 프로세스 안에서 따로 로드되는 Nabi 코드 한 덩어리입니다. 프로그램 자신(*호스트*)이거나 플러그인입니다. 모듈마다 읽기 전용 데이터에 *모듈 디스크립터* 하나를 두고, 이것으로 런타임에 자신을 알립니다. 로더는 이 디스크립터를 읽어 모듈이 프로세스와 맞는지 검사하고, 모듈이 가져오는 심볼을 바인딩하고, 모듈의 GC 타입, `any` 태그, 루트, 타입 디스크립터([8.13](813-type-descriptor.md))를 등록합니다. 레이아웃은 컴파일러와 런타임이 함께 빌드하는 `nabir_abi::module`에 한 번만 정의합니다.

## 8.12.1. 심볼

| 심볼 | 정의하는 쪽 | 내용 |
|------|-------------|------|
| `nabi_module` (Mach-O `_nabi_module`) | 플러그인 | 디스크립터. 플러그인이 내보내는 유일한 심볼입니다 |
| `nabi_host_module` | 호스트 | 디스크립터. 런타임이 직접 참조합니다 |

## 8.12.2. 레이아웃 규칙

모든 포인터는 절대 주소이고 운영체제의 재배치로 확정됩니다. `Str`은 UTF-8 바이트를 가리키는 `{ptr, len: usize}`이고, `Slice<T>`는 항목 `len`개를 가리키는 `{ptr, len: usize}`입니다. 모든 구조체는 두 포인터 폭 모두에서 암묵적인 패딩이 없습니다. `u64`는 8의 배수 위치에 있고, 포인터 크기 워드는 둘씩 짝을 이룹니다. 그래서 `u64` 정렬이 4인 32비트 타깃(i686)과 8인 타깃(wasm32)이 모든 오프셋에서 일치합니다. 필드는 뒤에 덧붙이기만 합니다. 읽는 쪽이 쓸 수 있는 필드는 `version`이 알려 줍니다.

## 8.12.3. `ModuleDesc`

| 필드 | 타입 | 의미 |
|------|------|------|
| `magic` | `u32` | little-endian `u32`로 읽은 `"NABM"` |
| `version` | `u16` | 1 |
| `flags` | `u16` | `HOST` 1, `PLUGIN` 2, `HOT` 4 (핫 리로드 빌드), `NO_UNLOAD` 8 (자기 코드의 C 콜백을 내보냄) |
| `hot_version` | `u64` | `HOT`가 아니면 0 |
| `text_lo`, `text_hi` | `usize` | 모듈의 코드는 `[text_lo, text_hi)`입니다 |
| `toolchain` | `Str` | 컴파일러 id. 한 프로세스의 모든 모듈은 같은 값을 가집니다 |
| `target` | `Str` | 타깃 트리플 |
| `name` | `Str` | 플러그인의 패키지 이름, 또는 호스트의 프로그램 이름 |
| `provides` | `Slice<PkgDesc>` | 코드와 static이 이 모듈에 있는 패키지 |
| `needs` | `Slice<PkgDesc>` | 프로세스에서 가져오는 패키지 |
| `imports` | `Slice<Import>` | 프로세스에서 가져오는 심볼 |
| `exports` | `Slice<Export>` | 내보내는 심볼. `sym` 순으로 정렬합니다 |
| `gc_types` | `GcTypes` | ScanTable([8.2](802-scan-table.md))과 로더가 매긴 id |
| `tags` | `Tags` | 코드가 쓰는 `any` 태그의 타입 키와 로더가 매긴 태그 |
| `roots` | `Slice<Root>` | static 루트 |
| `types` | `Slice<*const TypeDesc>` | 타입 디스크립터([8.13](813-type-descriptor.md)) |
| `dispatch` | `Slice<Dispatch>` | 메서드와 태그별 `any Tr` 메서드 본문 |
| `functions` | `Slice<FnRange>` | `HOT`에서만: 함수마다 범위와 프레임 정보 |

구성 요소:

| 구조체 | 필드 |
|--------|------|
| `PkgDesc` | `name: Str`, `version: Str`, `pkg8: [u8; 8]` (`_Y` 심볼 안의 패키지 해시 16진 숫자, [9.5](../009-tools/905-symbol-mangling.md)), `sha256: [u8; 32]` (패키지 내용 전체의 해시) |
| `Import` | `sym: Str` (맹글링된 이름), `sig: u64`, `slot: *mut usize`, `kind: usize` (`FN` 0, `DATA` 1, `HOT_FN` 2) |
| `Export` | `sym: Str`, `path: Str` (`pub`이고 제네릭이 아닌 함수나 static이면 `enemies::spawn`, 아니면 빈 문자열), `sig: u64`, `addr: usize`, `kind: usize` (`Import`와 같고, `FN_VALUE` 3은 함수의 값 thunk), `sig_text: Str` (`sig`가 해시하는 타입을 `type_name`이 쓰는 대로 쓴 것, `fn(i32) -> List<str>`. 에러 메시지용이며 `sig`가 0이면 빈 문자열) |
| `GcTypes` | `tables: *const *const u8`, `ids: *mut u32`, `n: u32`, `reserved: u32` |
| `TagKey` | `key: Str` (타입 키, 8.13.2), `hash: u64` (`key`의 FNV-1a 64) |
| `Tags` | `keys: *const TagKey`, `tags: *mut u32`, `n: u32`, `reserved: u32` |
| `Root` | `word: *const usize`, `any: usize` (1이면 `any`의 payload 워드이고 태그는 바로 앞 워드) |
| `Dispatch` | `method: u64`, `code: usize`, `tag_index: usize` (`tags`의 인덱스) |
| `FnRange` | `start: u32` (`text_lo`부터), `len: u32`, `frame: *const u8`, `reserved: usize` |

`sig`는 타입 키로 쓴 정규 시그니처 텍스트(예: `fn(Y3f8a1c2b:game::Player, i32) -> str`)의 SHA-256 앞 64비트입니다. 데이터이면 static 타입 키의 해시입니다. `FN` import는 `slot`을 거쳐 점프하는 stub으로 호출합니다. `DATA` import의 `slot`에는 데이터의 주소가 들어 있습니다.

## 8.12.4. 런타임 엔트리

다음 엔트리는 [8.5](805-extern-symbols.md)에 더해집니다. 모두 `thread-safe`입니다.

| 엔트리 | 의미 |
|--------|------|
| `nabi_types_register(descs: *const *const TypeDesc, n: usize, tags: *const TagKey, out_tags: *mut u32, nt: usize) -> i32` | 타입 디스크립터 `n`개를 등록하고 태그 `nt`개를 정합니다(8.12.5). 0을 돌려주거나, `descs[i]`의 키가 다른 레이아웃 해시로 이미 등록되어 있으면 `1 + i`를 돌려줍니다. 이때는 아무것도 등록하지 않습니다. |
| `nabi_types_by_tag(tag: u32) -> *const TypeDesc` | `any` 태그(`ANY_BOXED` 비트는 무시)의 디스크립터, 없으면 null |
| `nabi_types_by_path(p: *const u8, len: usize) -> *const TypeDesc` | `game::Player` 같은 경로로 가장 최근에 등록된 디스크립터, 없으면 null |
| `nabi_types_unregister(descs: *const *const TypeDesc, n: usize)` | 모듈을 해제할 때 그 등록을 지웁니다 |
| `nabi_module_register(m: *const ModuleDesc) -> i32` | 짧은 stop-the-world 안에서 모듈을 검사하고 바인딩하고 표를 등록합니다. 성공하면 0 |
| `nabi_any_dispatch(method: u64, tag: u32) -> usize` | 로드된 모듈에서 `tag` 값에 대한 `method`의 코드, 없으면 0 |
| `nabi_tls_get() -> *mut u8` | 호출한 스레드의 `nabi_tls` 블록([8.3](803-tls-slots.md)) |

타입 레지스트리를 읽는 쪽은 락을 잡지 않고 메모리를 할당하지 않습니다.

## 8.12.5. 태그

모듈의 코드는 쓰는 타입마다 `any` 태그를 `Tags.tags`에서 읽습니다. `nabi_types_register`는 키마다, `out_tags`에 이미 들어 있는 태그(`u32::MAX`가 아닌 값)를 그대로 둡니다. 호스트와 플러그인이 없는 프로그램은 이렇게 컴파일 시점 태그를 넘깁니다. 전에 등록된 키는 같은 태그를 받습니다. 그래서 한 타입은 모듈이 달라도 한 태그를 가집니다. 새 키는 키의 FNV-1a 해시 하위 31비트를 받고, 쓰이고 있는 태그를 모두 피해 옮겨 갑니다.

## 8.12.6. 컴파일러가 내보내는 것

플러그인의 글루([8.7](807-code-generation.md))는 `nabi_module`과 그것이 가리키는 테이블을, 호스트의 글루는 `nabi_host_module`을 내보냅니다. `std::plugin`에 닿지 않고 `--crate-type=plugin`도 아닌 프로그램은 둘 다 내보내지 않으며, 그 코드는 플러그인이 없을 때와 같습니다.

| 필드 | 플러그인 | 호스트 |
|---|---|---|
| `flags` | `PLUGIN` | `HOST` |
| `text_lo`, `text_hi` | 링커가 정의하는 `nabi.module.text_lo` / `nabi.module.text_hi`로의 재배치. 로더는 코드 범위를 매핑된 이미지에서 얻습니다 | 같음 |
| `provides` | 자기 패키지와 번들한 패키지([9.12.2](../009-tools/912-plugins.md#9122-빌드)) | 표준 라이브러리를 뺀, 링크하는 모든 패키지 |
| `needs` | 표준 라이브러리를 뺀, 빌드의 나머지 모든 패키지 | 비어 있음 |
| `imports` | `nabi.module.imports`로의 재배치. 링커가 정의되지 않은 채 남은 심벌(모든 `nabi_*` 엔트리와 `_Y` 심벌)로 만드는 테이블이며, 길이는 링커가 디스크립터에 씁니다. `sig`는 0 | 비어 있음 |
| `exports` | `sym` 순으로 정렬: 자기 패키지의 함수마다 `FN`(`pub`이고 제네릭이 아니면 `path` 설정), 그 값 썽크마다 `FN_VALUE`(`path`, `sig`, `sig_text` 설정), 자기 static마다 `DATA`(`path` 설정), 그리고 프로세스에서 가져오는 static의 자기 사본(아래) | `nabi.module.exports`로의 재배치. 링커가 만드는 테이블(정의된 모든 `nabi_*` 엔트리와 `_Y` 심벌, `sig` 0)이며 길이는 링커가 씁니다 |
| `gc_types` | 코드가 쓰는 모든 ScanTable. `ids`는 처음에 0 | 비어 있음. 호스트는 여느 프로그램처럼 `nabi_gc_register_type`으로 테이블을 등록합니다 |
| `tags` | 코드가 쓰는 태그마다 키. `tags`는 처음에 `u32::MAX` | 키와 컴파일 시점 태그 |
| `roots` | 정의하는 모든 static의 모든 참조 워드(`any`는 0) | 비어 있음. 여느 프로그램처럼 등록합니다 |
| `types` | 자기 타입 디스크립터([8.13](813-type-descriptor.md)). `tag_index`는 자기 `tags`의 인덱스 | 비어 있음. 여느 프로그램처럼 `nabi_types_register`로 디스크립터를 등록합니다 |
| `dispatch` | `any Tr` 디스패처마다, 그 갈래가 받는 태그마다: `<트레이트 경로>.<메서드>`의 FNV-1a 64 해시, 디스패처의 값 코드 | 호스트의 디스패처에 대해 같음 |

호스트의 `nabi.register_types`([8.5](805-extern-symbols.md))는 마지막에 `nabi_module_register(&nabi_host_module)`을 부릅니다. 그래서 호스트의 태그, 디스패치 항목, 내보내기 테이블은 `main`보다, 그리고 어느 플러그인보다 먼저 프로세스에 들어갑니다.

플러그인의 코드가 프로그램의 코드와 다른 점은 다음과 같습니다.

- **GC 타입 id.** 할당은 `nabi.module.gc_ids[0]`을 읽고 테이블의 인덱스를 더합니다. 로더는 모듈에 연속된 범위를 주므로 `ids[i] = ids[0] + i`입니다.
- **`any` 태그.** `any`에 저장할 때와 `any`의 태그를 검사할 때 상수 대신 `nabi.module.tags`에서 태그를 읽습니다(모듈의 태그 목록에서 그 타입의 인덱스 `i`의 `tags[i]`). `ANY_BOXED`는 실행 시점에 OR합니다.
- **TLS.** `nabi_tls`([8.3](803-tls-slots.md))의 주소를 얻던 코드, 즉 함수 값의 썽크와 C 엔트리는 가져온 `nabi_tls_get()`을 부릅니다. Nabi 사이의 호출은 어디서나처럼 블록을 숨은 마지막 인자로 넘기므로 대부분의 코드는 묻지 않습니다.
- **프로세스에서 가져오는 static.** 플러그인이 소유하지도 번들하지도 않은 패키지의 static(표준 라이브러리의 것 포함)은 타깃의 GOT 형태로 접근하며, 링커가 그것을 임포트의 슬롯으로 돌립니다. 플러그인은 자기 사본도 `<sym>.own`으로(지연 초기화되는 것은 `<sym>.state.own`, `<sym>.init.own`도) 정의하고 `<sym>`이라는 이름으로 내보냅니다. 로더는 임포트를 호스트의 static에, 없으면 먼저 로드된 플러그인의 것에, 그것도 없으면 이 사본에 묶습니다. 그래서 프로세스에서 static마다 인스턴스는 정확히 하나입니다.
- **디스패처.** 디스패처에서 모르는 태그를 받는 갈래는 `nabi_any_dispatch(method, tag)`를 부르고, 그 결과를 디스패처 자신의 타입의 함수 값으로 부릅니다. 호스트가 아닌 프로그램에서 이 갈래는 [8.8](808-lowering-types.md)의 트랩 그대로입니다.
- **시그니처.** 내보내기의 `sig`와 `Plugin::get::<F>`가 넘기는 값은 타입 키로 쓴 시그니처 텍스트(`fn(Y3f8a1c2b:engine::World,i32)->str`)의 SHA-256 앞 8바이트(리틀 엔디언)입니다. 양쪽이 같은 텍스트에서 컴파일 시점에 계산하므로 해시가 같으면 같은 타입입니다.
