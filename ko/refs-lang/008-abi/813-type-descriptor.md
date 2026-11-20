# 8.13. 타입 디스크립터 ABI

*타입 디스크립터*(`TypeDesc`)는 타입 하나를 적은 읽기 전용 기록입니다. 리플렉션이 이것을 읽고([6.10](../006-meta-system/610-reflection.md)), 모듈 로더는 두 모듈이 한 타입에 대해 일치하는지 이것으로 검사합니다([8.12](812-module-abi.md)). 레이아웃은 `nabir_abi::typedesc`에 한 번만 정의하고, [8.12.2](812-module-abi.md#8122-레이아웃-규칙)의 레이아웃 규칙을 따릅니다.

## 8.13.1. 디스크립터가 생기는 때

컴파일러는 프로그램이 요구할 때만 타입의 디스크립터를 내보냅니다. `Inspect` 타입의 `T::type_info()`나 `core::reflect::info::<T>()`가 그 요구이고, 필드, variant, 타입 인자가 가리키는 타입의 디스크립터도 재귀적으로 내보냅니다. 요구하지 않는 프로그램에는 디스크립터가 없고, 그 오브젝트는 리플렉션이 없을 때와 바이트 단위로 같습니다.

디스크립터는 읽기 전용 데이터의 hidden, weak 심볼입니다. 그래서 한 프로그램의 두 오브젝트가 같은 디스크립터를 내보내도 하나로 합쳐집니다. 디스크립터가 있는 프로그램의 `nabi.register_types`는 시작할 때 디스크립터를 컴파일 시점 `any` 태그와 함께 `nabi_types_register`에 넘깁니다.

## 8.13.2. 키

타입의 *키*는 모듈이 달라도 타입을 가리키는 이름입니다.

- struct, enum, union: `Y<pkg8>:`에 전체 경로를 붙입니다. `Y3f8a1c2b:game::Player`. `pkg8`은 [9.5](../009-tools/905-symbol-mangling.md)처럼 정의한 패키지의 해시입니다.
- 제네릭 인스턴스: 기본 키 뒤 `<..>`에 인자의 키를 씁니다. `Y..:core::collections::List<Y..:game::Player>`.
- 원시 타입, `str`, `any`, `none`: 그 이름입니다.
- 튜플, 합 타입, 배열, 함수 타입: 구성 요소의 키로 소스처럼 씁니다. 합 타입의 멤버는 정규 순서로 씁니다.

`key_hash`는 키의 SHA-256 앞 8바이트를 little-endian으로 읽은 값입니다. 두 디스크립터는 키가 같을 때, 그리고 그때만 같은 타입입니다.

## 8.13.3. `TypeDesc`

| 필드 | 타입 | 의미 |
|------|------|------|
| `magic` | `u32` | little-endian `u32`로 읽은 `"NABT"` |
| `version` | `u16` | 1 |
| `kind` | `u16` | `STRUCT` 1, `TUPLE_STRUCT` 2, `UNIT` 3, `ENUM` 4, `UNION` 5, `SUM` 6, `PRIM` 7, `STR` 8, `LIST` 9, `DICT` 10, `ARRAY` 11, `TUPLE` 12, `FN` 13, `ANY` 14, `OPAQUE` 15 |
| `flags` | `u32` | `INSPECT` 1 (필드, 속성, `ops`가 채워짐), `LAYOUT_ONLY` 2, `INSTANCE` 4, `HAS_DEFAULT` 8 |
| `tag_index` | `u32` | 모듈 `Tags`의 인덱스([8.12.5](812-module-abi.md#8125-태그)). 없으면 `u32::MAX` |
| `key_hash` | `u64` | 8.13.2 |
| `layout_hash` | `u64` | 크기, 정렬, 필드 오프셋과 타입 키, variant 태그, GC 워드의 해시 |
| `size` | `u64` | `T::size()`와 같음 |
| `align` | `u32` | `T::align()`와 같음 |
| `reserved0` | `u32` | 0 |
| `key` | `Str` | 8.13.2 |
| `name` | `Str` | `T::type_name()`과 같음: `Player`, `List<f64>` |
| `path` | `Str` | `game::Player`. 경로가 없는 타입이면 빈 문자열 |
| `fields` | `Slice<FieldDesc>` | struct의 필드를 선언 순서로. enum이면 비어 있음 |
| `variants` | `Slice<VariantDesc>` | enum의 variant를 선언 순서로 |
| `generics` | `Slice<TypeRef>` | 인스턴스의 타입 인자. 합 타입의 멤버, 튜플의 구성 요소, 리스트나 배열의 원소, 딕셔너리의 키와 값 |
| `attrs` | `Slice<AttrDesc>` | 타입에 붙은 런타임 속성([6.10.8](../006-meta-system/610-reflection.md#6108-런타임-속성)) |
| `ops` | `*const TypeOps` | 동적 접근(8.13.5). 없으면 null |
| `reserved1` | `usize` | 0 |

## 8.13.4. 구성 요소

| 구조체 | 필드 |
|--------|------|
| `TypeRef` | `name: Str`, `key_hash: u64`, `desc: *const TypeDesc` (내보낸 디스크립터가 없으면 null), `kind: usize` |
| `FieldDesc` | `name: Str` (튜플 struct나 variant에서는 `"0"`, `"1"`, ..), `renamed_from: Str` (`#[renamed_from(..)]`이 없으면 빈 문자열), `ty: TypeRef`, `attrs: Slice<AttrDesc>`, `offset: u64` (값의 시작부터. 자리가 고정되지 않은 필드면 `u64::MAX`), `flags: u32` (`PUB` 1, `HAS_DEFAULT` 2, `SKIP` 4), `index: u32`, `init_default: FnValue` (핫 빌드에서만) |
| `VariantDesc` | `name: Str`, `fields: Slice<FieldDesc>`, `tag: u32` (저장되는 태그), `index: u32`, `attrs: Slice<AttrDesc>` |
| `AttrDesc` | `ty: TypeRef` (속성 struct), `value: *const u8` (그 struct 레이아웃의 상수), `reserved: usize` |
| `FnValue` | `code: usize`, `env: usize`: Nabi 함수 값([8.8](808-lowering-types.md)). `env`는 0 |

## 8.13.5. `TypeOps`

`TypeOps`에는 Nabi 함수 값이 들어 있고(타입에 없는 항목은 `{0, 0}`), Nabi 호출 규약으로 부릅니다.

| 필드 | 시그니처 | 의미 |
|------|----------|------|
| `get` | `fn(owner: any, field: usize) -> any \| none` | 필드의 값. `owner`가 이 타입이 아니면 `none`. enum의 필드 번호는 `variant << 16 \| index`이고, 다른 variant이면 `none`입니다. `usize::MAX`를 주면 `owner`가 담은 variant의 인덱스를 `usize`로 돌려줍니다. |
| `set` | `fn(owner: any, field: usize, value: any) -> any \| none` | 필드를 바꾼 `owner`의 복사본. `owner`나 `value`가 맞지 않으면 `none` |
| `build` | `fn(variant: usize, fields: Dict<str, any>) -> (any \| none, str)` | 이름 붙은 필드로 만든 새 값(enum이면 `variant`, struct면 0). 빠진 필드는 기본값을 씁니다. 잘못된 점이 있으면 `none`과 그것을 알리는 메시지 |
| `new_default` | `fn() -> any \| none` | 예약(`{0, 0}`). 필드 없이 `build`하면 모든 필드가 기본값인 값이 나옵니다 |
| `list` | `*const ListOps` | 리스트와 배열: `len: fn(any) -> usize`, `get: fn(any, usize) -> any \| none`, `items: fn(any) -> List<any> \| none`, `from: fn(List<any>) -> any \| none` |
| `dict` | `*const DictOps` | 딕셔너리: `len: fn(any) -> usize`, `items: fn(any) -> List<(any, any)> \| none`, `from: fn(List<(any, any)>) -> any \| none`, `reserved` |
