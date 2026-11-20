# 9.11. nabi-mir 파일 형식

이 문서는 `<unit>.nabimir`를 정의합니다. `nabic`가 codegen unit 하나의 최적화한 nabi-mir를 따로 설치한 코드 생성기([8.11.13](../008-abi/811-backend-contract.md#81113-따로-설치하는-코드-생성기))에 넘기는 파일입니다. 코드 생성기 계약([8.11](../008-abi/811-backend-contract.md))이 `nabic`에 맡기는 것이 모두 들어 있습니다. 심볼과 그 링키지, 타입 레이아웃, GC 타입, safepoint 정보, 타깃입니다.

> [!NOTE]
> 0.2를 위한 초안입니다. 0.1의 `nabic`는 이 형식을 쓰지도 읽지도 않습니다. 아래 내용은 설계이고, 릴리스가 구현하기 전까지 바뀝니다. `--emit=mir`는 사람이 읽도록 nabi-mir를 계속 텍스트로 출력합니다([9.9.1](909-debug-tools.md#991-중간-산출물-출력---emit)).

## 9.11.1. 텍스트가 아닌 바이너리

형식은 바이너리입니다.

- 읽는 쪽은 사람이 아니라 코드 생성기입니다. `--emit=mir`의 텍스트는 읽기 위한 것이고, 코드 생성기에 필요한 것(레이아웃, 심볼, GC 타입, 디버그 정보)이 빠져 있습니다. 텍스트 형식이면 모든 코드 생성기에 문법과 parser가 필요한데, 그로써 득을 보는 독자가 없습니다.
- 표준 라이브러리를 많이 인스턴스화하는 unit의 최적화한 nabi-mir는 큽니다. 정수를 LEB128로 쓰고 문자열을 한 번씩만 두면, 모든 빌드의 모든 unit에서 파일이 작고 빨리 써집니다.
- LEB128은 LLVM과 다른 코드 생성기가 이미 읽는 형식이고(`decodeULEB128`), `nabic`가 증분 캐시에 이미 쓰는 형식입니다.
- NABIHIR([9.4](904-nabihir-bytecode.md))가 소스 텍스트인 것은 읽는 쪽이 `nabic`의 parser이기 때문입니다. 이 파일을 읽는 쪽은 Nabi parser가 없는 코드 생성기입니다.

## 9.11.2. 인코딩

| 표기 | 인코딩 |
|------|--------|
| 헤더와 섹션 테이블의 `u8`, `u16`, `u32`, `u64` | 고정 폭, little-endian |
| `u` | 부호 없는 LEB128(최대 10바이트, 64비트 이하의 값) |
| `s` | 부호 있는 정수. zigzag로 바꾼 뒤(`(v << 1) ^ (v >> 63)`) `u`로 씁니다 |
| `w` | 128비트 정수. 하위 64비트를 `u`로, 이어서 상위 64비트를 `u`로 씁니다 |
| `b` | 바이트 하나. `0` 또는 `1` |
| `str` | 문자열 테이블(9.11.4)의 `u` 인덱스. `0`은 빈 문자열입니다 |
| `opt<T>` | `b`, 그리고 그것이 `1`이면 `T` |
| `list<T>` | `u` 개수, 그리고 그만큼의 `T` |
| `ty` | 타입 테이블(9.11.6)의 `u` 인덱스 |
| `fn` | 함수 테이블(9.11.9)의 `u` 인덱스 |
| `static` | static 테이블(9.11.8)의 `u` 인덱스 |
| `local`, `block` | `u`. 그 함수 안에서 지역 변수나 블록의 번호 |
| `pos` | 소스 위치. `u` 파일(9.11.4, `0`이면 없음), `u` 줄, `u` 열(둘 다 1부터, `0`이면 모름) |

위치는 panic이나 디버그 정보가 이름 대는 곳입니다. 파일은 `path:line:col`이 보여 주는 경로이고, `nabic`가 펼쳐진 코드 안의 위치를 원래 텍스트로 되돌린 뒤의 것입니다. 파일 안의 값이 타깃의 바이트 순서를 따르는 것은 타깃 메모리의 바이트일 때(상수 바이트 배열, 9.11.7)뿐입니다. 다른 수는 모두 위의 방식으로 씁니다.

## 9.11.3. 배치

```
header (16 bytes)
  magic          4 bytes   "NABM"
  major          u16
  minor          u16
  flags          u32       0
  sections       u32       number of section table entries
section table    sections x 24 bytes
  kind           u32
  reserved       u32       0
  offset         u64       from the start of the file
  size           u64
sections         at their offsets, in any order
```

섹션 종류마다 많아야 하나씩 있습니다. 종류는 다음과 같습니다.

| 종류 | 섹션 | 내용 |
|------|------|------|
| 1 | strings | 9.11.4 |
| 2 | files | 9.11.4 |
| 3 | unit | 9.11.5 |
| 4 | target | 9.11.5 |
| 5 | structs | 9.11.6 |
| 6 | types | 9.11.6 |
| 7 | GC types | 9.11.7 |
| 8 | statics | 9.11.8 |
| 9 | functions | 9.11.9 |
| 10 | debug | 9.11.11 |

읽는 쪽은 모르는 종류의 섹션을 건너뜁니다. 1 ~ 9 섹션은 모든 파일에 있고, 10은 `-g`일 때만 있습니다.

## 9.11.4. 문자열과 파일

**strings**: `u` 개수, 그리고 문자열마다 `u` 길이와 그만큼의 UTF-8 바이트. 문자열 0은 빈 문자열입니다. 같은 문자열은 한 번만 둡니다.

**files**: `list<str>`. 위치가 가리키는 소스 파일의 경로를 panic이 출력하는 그대로 둡니다. 파일 인덱스 `i`(1부터)는 `i`번째 항목입니다.

## 9.11.5. unit과 타깃

**unit**:

| 필드 | 인코딩 | 뜻 |
|------|--------|-----|
| producer | `str` | NABIHIR([9.4.4](904-nabihir-bytecode.md#944-헤더와-producer))와 같은 `nabic <version> (<compiler id>)` |
| name | `str` | unit의 이름. 그 home 모듈(패키지 루트는 `<root>`), 또는 라이브러리 인스턴스 묶음이면 `<module>#k`([9.2.16](902-aot-pipeline.md#코드-생성-단위)) |
| opt | `u8` | `0` `-O0`, `1` `-O1`, `2` `-O2`, `3` `-O3`, `4` `-Os`, `5` `-Oz` |
| flags | `u` | bit 0: overflow 검사 켬. bit 1: `-g`. bit 2: 변수를 코드 범위별로 기술(`-O0`보다 높은 `-g`, wasm32 제외) |
| source directory | `str` | `files`의 상대 경로가 기준으로 삼는 디렉터리(디버그 정보) |

**target**: 코드 생성기가 읽는 필드로 쓴 타깃([0.6.5](../000-meta/006-platform-targets.md#065-타깃-사양)), 그리고 사양 전체입니다.

| 필드 | 인코딩 |
|------|--------|
| name | `str`(내장 이름이나 사양 파일의 stem) |
| arch, os, env, vendor, family | 각각 `str`. `cfg`가 부르는 이름 |
| endian | `u8`: `0` little, `1` big |
| 포인터 폭, `wide_align`, 최대 atomic 폭 | 각각 `u`(비트, 바이트, 비트) |
| C ABI | `u8`: `c_abi`(`Aarch64` 0, `X86_64SysV` 1, `X86_64Win` 2, `X86SysV` 3, `X86Win` 4, `ArmHf` 5, `Wasm32` 6, `Riscv64` 7, `Riscv32` 8, `Ppc64` 9, `S390x` 10, `Other` 11) |
| 오브젝트 형식 | `u8`: `0` Mach-O, `1` ELF, `2` COFF, `3` wasm |
| CPU, features | 각각 `str` |
| C 심볼 접두사 | `str` |
| 스택 probe | `u8`: `0` 인라인, `1` 호출, `2` 없음 |
| threads | `b` |
| 사양 | `str`. `nabic --print target-spec`이 출력하는 `TargetSpec { .. }` 텍스트 |

## 9.11.6. 타입

**structs**: 구조체 레코드의 `list`. 순서대로 0부터 번호를 매기고, 타입은 이 번호로 구조체를 가리킵니다.

| 필드 | 인코딩 | 뜻 |
|------|--------|-----|
| name | `str` | 디버그 정보용(이름이 없으면 빈 문자열) |
| key | `str` | 빌드와 무관한 구조체의 이름 |
| packed | `b` | `#[repr(packed)]` |
| align | `u` | `#[repr(align(N))]`. 없으면 `0` |
| 필드 이름 | `list<str>` | 디버그 정보용 |

**types**: 타입 레코드의 `list`. 레코드는 자기보다 앞의 타입만 가리키므로 한 번에 읽을 수 있습니다. 같은 타입은 레코드 하나입니다.

| 태그(`u8`) | 타입 | 이어서 |
|-----|------|------|
| 0 | `()` | |
| 1 | 원시 타입 | `u8` 원시 타입(아래) |
| 2 | 해석되지 않은 word | |
| 3 | 튜플 | `list<ty>` 필드 |
| 4 | 구조체 | `u` 구조체 번호, `list<ty>` 필드 |
| 5 | `any Tr` 값 | ([8.8.9](../008-abi/808-lowering-types.md#889-any-슬롯의-표현)의 `(u32 tag, payload)` 쌍) |
| 6 | 배열 | `ty` 원소, `u` 길이 |
| 7 | 함수 포인터 | `list<ty>` 매개변수, `ty` 결과 |
| 8 | GC 참조 | |
| 9 | union | `list<(u32 tag를 u로, ty)>` variant, `u` word 수, `u8` 종류: `0` 합의 payload, `1` 평범한 것 |

원시 타입: `0` `i8`, `1` `i16`, `2` `i32`, `3` `i64`, `4` `isize`, `5` `u8`, `6` `u16`, `7` `u32`, `8` `u64`, `9` `usize`, `10` `f32`, `11` `f64`, `12` `bool`, `13` `c_char`, `14` `char`, `15` `str`.

레코드마다 태그와 피연산자 뒤에, `nabic`가 타깃에 맞게 계산한 레이아웃이 옵니다([8.11.5](../008-abi/811-backend-contract.md#8115-데이터-레이아웃)).

| 필드 | 인코딩 | 뜻 |
|------|--------|-----|
| size, align | `u`, `u` | 바이트 단위 |
| offsets | `list<u>` | 튜플과 구조체: 각 필드의 오프셋. 다른 타입은 빈 목록 |
| roots | `list<u>` | 이 타입의 지역 변수를 위해 shadow stack 프레임이 등록하는 word의 오프셋, 오름차순([8.11.6](../008-abi/811-backend-contract.md#8116-gc-할당-루트-barrier)). 타입에 GC 참조가 없으면 빈 목록 |
| barrier | `b` | 이 타입을 GC 객체에 저장할 때 write barrier가 필요합니다 |

함수 값은 함수 포인터와 GC 참조의 튜플 `(code, env)`이고, 던지는 함수의 carrier는 튜플 `(is_err, payload)`이고, 합은 종류 0인 튜플 `(i32, union)`입니다. 타입 테이블에 다른 종류는 없습니다.

## 9.11.7. GC 타입과 상수

**GC types**: unit이 쓰는 GC 타입의 `list`. 각각:

| 필드 | 인코딩 | 뜻 |
|------|--------|-----|
| id | `u` | 코드가 상수로 쓸 수 있으면 type id, 아니면 `0` |
| symbol | `str` | unit 빌드에서 id를 담은 `_Y<pkg>_gctype.<hash>` 심볼([8.11.3](../008-abi/811-backend-contract.md#8113-심볼과-링키지)), 아니면 빈 문자열 |
| table | `list<u>` | ScanTable의 `u32` word([8.2.3](../008-abi/802-scan-table.md#823-직렬화-포맷)). 읽기 전용입니다. 내보내고 등록하는 것은 공유 unit입니다 |

할당하는 rvalue는 이 목록의 인덱스로 GC 타입을 가리킵니다.

**상수**(`const`), 태그 `u8`별:

| 태그 | 상수 | 이어서 |
|-----|------|------|
| 0 | `()` | |
| 1 | `bool` | `b` |
| 2 | 부호 있는 정수 | `w` 값, `u8` 원시 타입 |
| 3 | 부호 없는 정수 | `w` 값, `u8` 원시 타입 |
| 4 | 부동소수 | `u` 비트(`f32`는 하위 32비트), `u8` 원시 타입 |
| 5 | `char` | `u` 코드 포인트 |
| 6 | 문자열 | `str` |
| 7 | 바이트 | `u` 길이, 그만큼의 바이트 |
| 8 | none | |

상수는 쓰이는 자리의 타입을 따릅니다. 부동소수로 쓰이는 정수 상수는 그 수를 변환한 값이고, `()`와 none은 그 타입의 0 값입니다.

## 9.11.8. static

**statics**: static 레코드의 `list`.

| 필드 | 인코딩 | 뜻 |
|------|--------|-----|
| name | `str` | 항목 경로 |
| symbol | `str` | 링커 심볼([8.11.3](../008-abi/811-backend-contract.md#8113-심볼과-링키지)) |
| type | `ty` | |
| flags | `u` | bit 0: `mut`. bit 1: 프로그램 밖에서 정의(`extern`). bit 2: `__imp_<symbol>`을 거침(Windows의 DLL static). bit 3: lazy([8.11.10](../008-abi/811-backend-contract.md#81110-static)). bit 4: 프로그램 자신의 오브젝트에서 정의(아니면 형식에 GOT가 있을 때 GOT를 거침) |
| init | `opt<const>` | 초깃값. 읽기 전용입니다. 저장 공간을 정의하는 것은 공유 unit입니다 |

## 9.11.9. 함수

**functions**: 함수 레코드의 `list`. unit이 정의하는 함수에는 본문이 있습니다. unit이 부르거나 주소를 쓰는 다른 함수는 선언(블록 없음)입니다.

| 필드 | 인코딩 | 뜻 |
|------|--------|-----|
| name | `str` | nabi-mir 이름 |
| path | `str` | 항목 경로(제네릭 인스턴스는 그 제네릭의 경로) |
| symbol | `str` | 본문의 심볼 |
| C entry | `str` | `extern "C"` 정의: C 진입점의 심볼. 아니면 빈 문자열 |
| thunk | `str` | unit이 함수의 주소를 쓰고 그 함수가 `nabi_tls`를 받으면 함수 값 thunk의 심볼(`<symbol>.v`). 아니면 빈 문자열 |
| define | `u8` | `0` 선언, `1` 지역, `2` global, `3` weak global |
| flags | `u` | 아래 |
| ABI | `u8`, 이어서 | `0` Nabi. `1` C: `b` 가변 인자, `b` `extern "system"` |
| C 분류 | `opt<cabi>` | C ABI 함수와, 분류가 필요한 `extern "nabi"` 선언용([8.11.4](../008-abi/811-backend-contract.md#8114-호출-규약)) |
| wasm import | `opt<(str, str)>` | wasm32 import의 모듈과 이름 |
| position | `pos` | 선언 위치 |
| 매개변수 | `list<local>` | |
| 매개변수 타입, 결과 | `list<ty>`, `ty` | |
| 지역 변수 | `list<(local, str 이름, ty)>` | 매개변수를 포함한 모든 지역 변수. 디버그 정보가 빼는 것은 빈 이름 |
| pins | `list<local>` | `StorageDead`까지 등록된 채로 있는 지역 변수 |
| 블록 | `list<블록 레코드>` | 첫 블록이 진입점 |
| debug | `opt<디버그 레코드>` | 9.11.11. `-g`일 때 있습니다 |

함수 flags:

| 비트 | 뜻 |
|-----|-----|
| 0 | 숨은 마지막 인자로 `nabi_tls` 주소를 받습니다 |
| 1 | safepoint에 닿지 않습니다. 프레임도 poll도 없고, 이 함수의 호출은 safepoint가 아닙니다([8.11.7](../008-abi/811-backend-contract.md#8117-safepoint와-poll)) |
| 2 | 재진입 가능. 첫 safepoint 전에 poll합니다 |
| 3 | 직접 Nabi 호출이 아닌 길로도 들어옵니다. 진입에서 poll합니다 |
| 4 | `#[track_caller]`. 마지막 매개변수가 호출자의 위치(`str`)입니다 |
| 5 | `#[no_safepoint]` 런타임 선언 |
| 6 | `#[cold]` |
| 7 | `#[inline(never)]` |
| 8 | 사용자의 `main`(공유 unit의 C `main`이 부릅니다) |
| 9 | `any` 디스패처 |

C 분류 `cabi`: `u8` 결과(`0` 직접, `1` 변환 뒤 `ctype`, `2` `sret`), 인자의 `list`(`0` 직접, `1` 변환 뒤 `ctype`, `2` 참조로 뒤에 `b` byval), `b` 가변 인자, `b` `str`을 `(ptr, len)` 쌍으로, `b` stdcall. `ctype`은 `u8` `0` 뒤에 `cscalar`(스칼라), `1` 뒤에 `u` 개수와 `cscalar`(연속 레지스터), `2` 뒤에 `list<cscalar>`(자연 정렬 필드의 구조체), `3` 뒤에 `list<(u 오프셋, cscalar)>`입니다. `cscalar`는 `u8` `0` 뒤에 `u` 비트 수(정수), `1` `f32`, `2` `f64`, `3` 레지스터 하나에 든 `f32` 둘, `4` 포인터입니다.

**블록 레코드**: `block` 번호, `list<local>` 매개변수, `u` flags(bit 0: poll하는 back edge가 이 블록에서 나갑니다. terminator 앞에서 poll을 실행합니다), `list<statement>`, terminator.

**피연산자**: `u8` `0` 뒤에 `const`, 또는 `1` 뒤에 `local`.

**문장**, 태그 `u8`별:

| 태그 | 문장 | 이어서 |
|-----|------|------|
| 0 | `local = rvalue` | `local`, rvalue |
| 1 | 배열 원소 쓰기 | `local` 배열, `u` 바이트 오프셋, 피연산자 인덱스, 피연산자 값, `ty` 원소, `u` 길이 |
| 2 | storage dead | `local` |
| 3 | 버퍼 원소 저장 | 피연산자 버퍼, 피연산자 인덱스, 피연산자 값, `ty` 원소 |
| 4 | 위치 | `pos`, `u` scope, `u` 부모 scope |
| 5 | raw 저장 | 피연산자 주소, 피연산자 값, `ty`, `b` volatile |
| 6 | 힙 필드 저장 | 피연산자 객체, `ty` payload(튜플), `u` 필드, 피연산자 값 |
| 7 | static 저장 | `static`, 피연산자 값, `ty` |

**rvalue**, 태그 `u8`별:

| 태그 | rvalue | 이어서 |
|-----|--------|------|
| 0 | use | 피연산자 |
| 1 | 이항 | `u8` 연산자, `ty` 피연산자 타입, 피연산자, 피연산자 |
| 2 | 단항 | `u8` 연산자, `ty` 피연산자 타입, 피연산자 |
| 3 | 호출 | `fn`, `list<피연산자>`, `ty` 결과 |
| 4 | 구조체 값 | `ty`, `list<피연산자>` 필드 |
| 5 | 필드 읽기 | 피연산자, `u` 필드, `ty` 필드 타입 |
| 6 | cast | 피연산자, `ty` from, `ty` to |
| 7 | bitcast | 피연산자, `ty` from, `ty` to |
| 8 | 필드 바꾼 값 | 피연산자, `ty` 구조체, `u` 필드, 피연산자 값 |
| 9 | 배열 값 | `ty` 원소, `list<피연산자>` |
| 10 | 배열 원소 읽기 | `local` 배열, `u` 바이트 오프셋, 피연산자 인덱스, `ty` 원소, `u` 길이 |
| 11 | 함수 주소 | `fn`, `ty` |
| 12 | static 읽기 | `static`, `ty` |
| 13 | box | 피연산자, `ty`, `u` GC 타입 + 1(`0`: 값이 payload word이고 아무것도 할당하지 않습니다) |
| 14 | 버퍼 할당 | `ty` 원소, 피연산자 길이, `u` GC 타입 |
| 15 | raw 읽기 | 피연산자 주소, `ty`, `b` volatile |
| 16 | 지역 변수 주소 | `local` |
| 17 | static 주소 | `static` |
| 18 | 크기 또는 정렬 | `ty`, `b` 정렬 |
| 19 | 버퍼 원소 읽기 | 피연산자 버퍼, 피연산자 인덱스, `ty` 원소 |
| 20 | 버퍼 atomic | `u8` 연산, 피연산자 버퍼, 피연산자 인덱스, `list<피연산자>`, `ty` 원소, `u8` 순서, `u8` 실패 순서 |
| 21 | box 읽기 | 피연산자, `ty` |
| 22 | 힙 할당 | `ty` payload, `u` GC 타입 |
| 23 | 힙 필드 읽기 | 피연산자 객체, `ty` payload, `u` 필드 |
| 24 | 간접 호출 | 피연산자 callee, `list<피연산자>`, `list<ty>` 매개변수, `ty` 결과, 함수 레코드와 같은 ABI, `opt<cabi>` |

연산자(`u8`): `0` `+`, `1` `-`, `2` `*`, `3` `/`, `4` `%`, `6` 부정, `7` `!`, `8` `&`, `9` `|`, `10` `^`, `11` `<<`, `12` `>>`, `13` `==`, `14` `!=`, `15` `<`, `16` `<=`, `17` `>`, `18` `>=`, `19` `&&`, `20` `||`, `21` 논리 not, `22` 제곱근, `23` wrapping `+`, `24` wrapping `-`, `25` wrapping `*`, `26` 1인 비트 수, `27` 앞쪽 0 비트 수, `28` 뒤쪽 0 비트 수, `29` 바이트 순서 뒤집기. `5`는 예약입니다(내림 나머지 `%%`. 코드 생성기에는 오지 않습니다). 비교의 결과 타입은 대상의 타입입니다.

atomic 연산: `0` load, `1` store, `2` compare-exchange(인자: 기대값, 새 값), `3` exchange, `4` add, `5` sub, `6` and, `7` or, `8` xor. 순서: `0` relaxed, `1` acquire, `2` release, `3` acq_rel, `4` seq_cst.

**terminator**, 태그 `u8`별: `0` 점프(`block`, 그 매개변수에 넘길 `list<피연산자>`), `1` 분기(피연산자 조건, 참일 때 `block`과 `list<피연산자>`, 거짓일 때 `block`과 `list<피연산자>`), `2` return(`opt<피연산자>`), `3` unreachable.

## 9.11.10. 의미

각 문장, rvalue, terminator는 nabi-mir에서의 뜻을 그대로 가집니다. 코드 생성기가 거기에 더하는 것(검사, barrier, 루트, poll, 할당)은 [8.11](../008-abi/811-backend-contract.md)이 정합니다. nabi-mir 값이 가리키지만 파일에 없는 데이터는 공유 unit이나 다른 오브젝트의 것이고, 심볼로 닿습니다.

## 9.11.11. 디버그 정보

함수의 디버그 레코드(`-g`일 때):

| 필드 | 인코딩 | 뜻 |
|------|--------|-----|
| 지역 변수 타입 | `list<(local, dty)>` | 타입만으로는 디버거가 알 수 없는, 이름 있는 지역 변수를 보이는 방법 |
| bindings | `list` | 최적화기가 대입을 없앤 이름 있는 지역 변수: `local`, 보이는 곳 어디서나 담고 있는 값 `opt<피연산자>`, 제자리에서 만든 구조체의 필드 `list<opt<피연산자>>`, `u` 줄, `u` scope, `u` 부모 scope |
| inlined | `list` | 인라인한 본문: `u` scope, `u` 부모 scope, `str` 피호출자 경로, `pos` 선언, `pos` 호출, `list<local>` 매개변수의 사본 |
| results | `list` | 인라인한 본문의 결과를 대입받는 지역 변수: `local`, `u` 줄, `u` scope, `u` 부모 scope |

debug 섹션에는 구조체별 필드의 디버그 타입 `list<(u 구조체, list<dty>)>`와, 태그별로 `any` 값에 저장되는 타입 `list<(u 태그, str 이름, dty)>`가 있습니다. `dty`는 `u8` `0` 뒤에 `ty`(타입이 말하는 그대로), `1` 합: `str` 이름, `ty`, `list<(u 태그, str 이름, dty)>`, `2` GC 버퍼: `str` 이름, `dty` 원소, `3` `any`, `4` box한 값: `str` 이름, `dty`, `5` 캡처한 변수의 cell: `dty`, `6` 함수 값: `str` 이름, `ty`, `dty` 환경입니다.

## 9.11.12. 버전과 호환성

헤더의 `major`와 `minor`가 형식의 버전입니다.

- 읽는 쪽은 자기가 읽는 `major`의 파일을 받습니다. 한 `major` 안에서 더 높은 `minor`는 섹션 종류만 더하고, 오래된 읽는 쪽은 그것을 건너뜁니다. 기존 섹션의 인코딩, 따라서 모든 문장, rvalue, 타입, 상수 태그는 바뀌지 않습니다.
- 기존 섹션을 바꾸는 것은 모두 새 `major`입니다. 새 문장, rvalue, 타입 종류, 새 필드, 바뀐 뜻이 그렇습니다. 읽는 쪽은 읽지 못하는 `major`를 두 버전을 모두 알리며 거부합니다([8.11.13](../008-abi/811-backend-contract.md#81113-따로-설치하는-코드-생성기)의 종료 상태 2).
- `major`가 `0`인 동안(이 초안) 읽는 쪽은 자기가 빌드된 `major.minor`와 정확히 같은 것만 받습니다.
- 컴포넌트의 버전은 그 툴체인의 버전이므로([8.11.13](../008-abi/811-backend-contract.md#81113-따로-설치하는-코드-생성기)) 툴체인의 `nabic`와 그 컴포넌트는 언제나 맞습니다. 버전 규칙은 툴체인 밖에서 빌드한 컴포넌트를 위한 것입니다.

쓰기는 결정적입니다. 같은 unit은 같은 바이트를 냅니다. 문자열, 타입, 함수는 unit의 함수를 nabi-mir 순서로 훑으며 처음 만나는 순서대로 번호를 매기고, map은 키로 정렬해 씁니다.
