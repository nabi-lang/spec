# 8.8. Lowering: 데이터 타입과 표현식

이 절은 Nabi의 값, 타입, 식이 `nabic`가 만드는 코드에서 어떻게 표현되는지 정의합니다. 레이아웃은 Nabi 타입으로 씁니다. `(A, B)`는 자연 오프셋에 놓인 필드 쌍, `[T; n]`은 배열, `ptr`은 포인터 폭의 주소, `p`는 바이트 단위 포인터 폭입니다. 코드는 의사 코드입니다([8.7.1](807-code-generation.md#871-변환-흐름)).

## 8.8.1. 원시 정수 / 부동소수

원시 수치 타입은 그 폭의 기계 값입니다.

| Nabi 타입          | 표현                                                                    |
|-------------------|-------------------------------------------------------------------------|
| `i8` ~ `i64`      | 1 ~ 8바이트, 2의 보수                                                   |
| `u8` ~ `u64`      | 1 ~ 8바이트. 부호 해석은 연산이 정함                                    |
| `usize` / `isize` | 타깃의 포인터 폭                                                        |
| `f32` / `f64`     | IEEE 754 binary32 / binary64                                            |
| `bool`            | 1바이트, `0` 또는 `1`                                                   |
| `char`            | 4바이트 (Unicode 스칼라 값)                                             |
| `c_char`          | 1바이트                                                                 |

`str` 타입은 불변 쌍 `(ptr, usize)`(포인터와 바이트 길이)이며 값으로 전달합니다. 리터럴은 수집되지 않는(immortal) 상수를 가리킵니다. 런타임에 만든 문자열은 GC 객체(type id `1`, [8.2.12](802-scan-table.md#8212-런타임-예약-type-id))의 페이로드를 가리킵니다.

- `s[a:b]`는 `(ptr + a, b - a)`이며 원래 바이트를 공유합니다(O(1)).
- `ptr`은 GC 객체의 어느 바이트든(끝의 NUL 포함) 가리킬 수 있고, 객체 전체를 살려 둡니다. GC는 이 포인터를 내부 포인터로 다룹니다. evacuation은 오프셋을 유지한 채 포인터를 고쳐 씁니다.
- 리터럴과 런타임이 새로 만든 문자열은 바이트 뒤에 NUL이 있습니다. 슬라이스에는 없을 수 있습니다.

## 8.8.2. 산술 연산

원시 타입에서 [연산자 디스패치](../003-expressions/301-operator-dispatch.md)의 연산자는 피연산자 폭의 명령어 하나입니다. 덧셈, 뺄셈, 곱셈, 피연산자 타입에 따른 부호 있는 / 없는 나눗셈과 나머지, 부호 있는 / 없는 정수 비교와 ordered 부동소수 비교가 그렇습니다.

```
a + b       ->  add.i32 a, b            ; with overflow checks: add, then branch to the panic on overflow
a / b       ->  div.s32 a, b            ; after the checks for b == 0 and MIN / -1
a < b       ->  lt.s32 a, b             ; lt.u32 for unsigned, lt.f64 (ordered) for floats
```

검사가 붙은 연산은 결과가 들어맞지 않으면 panic으로 분기합니다(`-C overflow-checks`, [9.1.2](../009-tools/901-nabic-cli.md#nabic-build-aot-빌드)). 최적화 빌드는 실패할 수 없다고 증명한 검사를 뺍니다([8.7.3](807-code-generation.md#873-최적화-수준)). 사용자 정의 트레잇 구현은 제네릭 단형화를 거친 뒤 함수 호출이 됩니다.

## 8.8.3. `struct`의 표현

`struct`는 값 타입이며([2.3.8](../002-types/203-struct.md#238-값-의미)) 객체 헤더가 없습니다.

```nabi
struct Point { x: f64; y: f64; }
```

표현은 필드를 [2.3.6](../002-types/203-struct.md#236-메모리-레이아웃)의 배치대로 놓은 것이며, `#[repr]`의 정렬과 오프셋도 그대로 따릅니다.

```
(f64, f64)        ; x at 0, y at 8; size 16, align 8
```

필드가 같은 두 struct는 레이아웃도 같습니다. struct 값은 레지스터나 스택 슬롯에 있거나, 담는 쪽 안에 인라인으로 있습니다. 담는 쪽은 다른 struct의 필드, `List` 버퍼의 원소, 클로저 환경이나 코루틴 프레임의 슬롯입니다. 대입과 인자 전달은 필드를 복사합니다. 컴파일러는 box가 겉으로 드러나지 않는 곳에서만 값을 내부적으로 box에 담을 수 있습니다.

### 필드 접근

`p.x`는 그 오프셋의 필드를 읽습니다. struct의 슬롯이나 인라인 위치에서 읽는 load이거나, 필드를 담은 레지스터입니다.

```
val = load.f64 [p + 0]
```

### 필드 쓰기

`p.x = 1.0f64`는 변수 자신의 사본을 바꿉니다. 슬롯 안 필드 오프셋에 쓰는 store입니다.

```
store.f64 [p + 0], 1.0
```

struct가 GC 힙 객체 안에 인라인으로 있고 필드가 GC 참조를 담으면 그 store는 write barrier를 거칩니다([배리어와 루트](../007-runtime/704-barriers-and-roots.md)). 스택 슬롯에 있는 struct의 포인터 필드는 GC 루트입니다([7.4](../007-runtime/704-barriers-and-roots.md)).

## 8.8.4. tuple `struct`의 표현

다음은 튜플 `struct`의 예입니다.

```nabi
struct Pair(i32, i32);
```

일반 `struct`와 똑같이 표현합니다. 필드 이름 대신 위치 인덱스로 접근한다는 점만 다릅니다.

## 8.8.5. 튜플 타입

다음 튜플 타입은

```nabi
(i32, str, bool)
```

`(i32, (ptr, usize), bool)`로 배치합니다. `str`은 어디에 있든 그 쌍입니다(8.8.1). `struct`처럼 헤더 없는 값입니다.

## 8.8.6. `enum`의 표현

[`enum`](../002-types/204-enum.md)의 `enum`은 객체 헤더가 없는 값이며, 태그 있는 `union`으로 표현합니다. 합 타입(8.8.8)과 같은 `(i32 tag, payload)` 배치입니다.

```nabi
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

표현은 다음과 같습니다.

```
(i32, [u64; 2])    ; 64-bit target: tag at 0, payload at 8 (16 bytes, 8-aligned)
```

tag에는 변형의 번호(`Ok`=0, `Err`=1, `Pending`=2)가 들어갑니다. 페이로드는 데이터가 있는 변형들의 union입니다. 그중 가장 큰 것만큼 크고 그만큼 정렬되며, 각 변형의 페이로드는 자기 레이아웃으로 시작 위치에 놓입니다. GC 참조를 담을 수 있는 페이로드는 포인터 폭 워드의 연속이며, 컬렉터는 tag를 보고 훑습니다([8.2](802-scan-table.md), `SUM_PAYLOAD`).

필드가 없는 enum은 tag 하나입니다. `i32`, 또는 `#[repr(..)]`의 정수입니다.

### 변형 생성

`Result::Ok(42)`는 레지스터나 대상의 슬롯에서 값을 만듭니다. 할당은 없습니다.

```
store.i32 [v + 0], 0        ; tag: Ok
store.i32 [v + 8], 42       ; the payload in its own type
```

더 짧은 변형의 페이로드로 만든 union 값은 그 페이로드 너머의 바이트를 0으로 둡니다. 그래서 거기서 복사한 루트 워드는 null이거나 참조이고, 낡은 스택 값이 되지 않습니다([8.9.12](809-lowering-control.md#8912-shadow-stack-프레임)).

### 변형 매칭

변형을 매칭하는

```nabi
match v {
    Ok(n) => ...,
    Err(s) => ...,
    Pending => ...,
}
```

은 tag를 각 갈래의 변형 번호와 갈래 순서대로 비교합니다(8.9.3). 어느 갈래에도 맞지 않는 값은 unreachable 블록에 닿습니다.

```
bb0:
    tag = v.0
    if tag == 0 then bb_ok else bb1
bb1:
    if tag == 1 then bb_err else bb2
bb2:
    if tag == 2 then bb_pending else bb_unreachable
bb_ok:
    n = v.1 as i32          ; the payload read as the variant's type
```

## 8.8.7. 비태그 `union`의 표현

[`union`](../002-types/205-union.md)의 `union`의 예입니다.

```nabi
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

가장 큰 필드만큼 크고 그만큼 정렬되며(여기서는 8바이트), 모든 필드가 오프셋 0에서 시작합니다. 필드 접근은 같은 메모리를 그 필드의 타입으로 읽거나 씁니다. 필드 읽기는 소스에서 `unsafe` 안에 있어야 합니다([2.5](../002-types/205-union.md)).

## 8.8.8. 합 타입(`A | B`)의 표현

[합 타입](../002-types/206-sum-types.md)의 합 타입은 다음과 같이 다룹니다.

```nabi
type X = i32 | f64
```

이 타입은 `(i32, u64)`로 표현합니다. tag와 페이로드 자리를 묶은 것이며, `enum`(8.8.6)과 같은 방식으로 만듭니다. 여기서는 페이로드가 `i32`나 `f64`를 담습니다. `none` 멤버는 tag 0이고 페이로드가 없으므로, 0으로 채운 메모리는 `none`으로 읽힙니다.

[niche 최적화](../002-types/207-niche-optimization.md)가 적용되면 tag 자리가 사라지고 페이로드의 비트 패턴만 남습니다.

```nabi
Ptr<u8> | none
```

위 타입은 포인터 하나입니다. null이 `none`을 뜻하는 자동 niche입니다.

## 8.8.9. `any` 슬롯의 표현

[`any` 슬롯](../002-types/202-any-slot.md)의 동적 슬롯은 다음과 같이 씁니다.

```nabi
let x: any = ...
```

`(u32 tag, payload)` 쌍입니다. `any Tr`과 맨 `any`의 표현은 같습니다.

```
(u32, ptr)
```

tag의 하위 31비트는 값의 타입을 가리킵니다. 타입 이름의 31비트 FNV-1a 해시이므로 따로 컴파일한 패키지끼리도 값이 같습니다(한 빌드 안에서 충돌하면 다음 빈 값을 씁니다). 포인터 폭 이하의 숫자, `bool`, `char` 값은 페이로드 워드에 비트 그대로 담습니다. 그 밖의 값은 GC 힙의 box이고, 페이로드는 그 box이며, tag의 31번 비트(`ANY_BOXED`)가 켜집니다. GC는 `any` 슬롯을 훑을 때 이 비트를 읽습니다([8.2.7](802-scan-table.md)).

### `any`의 타입 검사

`any` 값에 대한 타입 패턴

```nabi
match v {
    n: i32 => n + 1,
    s: str => s.len() as i32,
    _ => 0,
}
```

은 `ANY_BOXED`를 지운 tag를 각 패턴 타입의 tag와 비교하는 두 갈래 분기의 사슬입니다.

```
bb0:
    t = v.0 & 0x7fffffff
    if t == TAG_I32 then bb_i32 else bb1
bb_i32:
    n = v.1 as i32          ; the payload word's low 32 bits
bb1:
    if t == TAG_STR then bb_str else bb_other
bb_str:
    s = load.str [v.1 + 16] ; a boxed `str`: the pair from the box payload
```

실행 시간의 이런 검사가 `any` 슬롯의 비용입니다.

## 8.8.10. 배열 타입: `[T; N]`

[타입 표현식](../002-types/201-type-expressions.md)의 고정 크기 배열의 예입니다.

```nabi
[i32; 8]
```

`i32` 원소 8개가 연달아 놓인 32바이트입니다. GC 객체 안에서는 배열 앞에 16바이트 헤더가 붙습니다.

## 8.8.11. 함수 타입: `fn(...) -> ...`

[함수 타입](../004-functions/404-function-type.md)의 예입니다.

```nabi
fn(i32, i32) -> i32
```

함수 값은 이름 있는 함수든 클로저든 모두 `{code, env}` 쌍입니다.

```
(ptr, ptr)        ; code, env
```

`code`는 `env`를 첫 인자로 받고, 그다음에 선언한 인자를 받으며, TLS 주소는 받지 않습니다([8.6.11](806-calling-convention.md#8611-함수-포인터-abi)). 값으로 쓴 이름 있는 함수는 `env`가 null이고, 코드 포인터는 `env`를 버리는 thunk를 가리킵니다. C 함수 타입(`extern "C" fn(..)`)만 맨 코드 포인터입니다.

## 8.8.12. 클로저의 표현

캡처가 있는 [클로저](../004-functions/405-closure.md)의 예입니다.

```nabi
let n = 10;
let f = |x| x + n;       // n 캡처
```

`env`가 캡처를 담은 GC 객체인 함수 값(8.8.11)입니다. 캡처는 16바이트 헤더 뒤의 struct에 하나씩 필드로 들어갑니다.

```
env object: header (16 bytes), then (i32)     ; a copy of n
f = (code of the closure body, env)

; on call
r = call f.0(f.1, x)
```

아무것도 캡처하지 않는 클로저는 `env`가 null이고 아무것도 할당하지 않습니다.

## 8.8.13. 식별자 / 변수 접근

[변수 선언](../005-modules/501-variable-declaration.md)의 지역 변수의 예입니다.

```nabi
let counter = 0i32;
counter = counter + 1i32;
```

모든 지역 변수에는 스택 슬롯이 있고, 대입은 그 슬롯에 쓰는 store입니다.

```
store.i32 [counter], 0
c = load.i32 [counter]
c2 = add.i32 c, 1
store.i32 [counter], c2
```

`-O0`에서는 모든 값이 자기 슬롯에 머뭅니다. `-O0`보다 높은 수준에서는 주소를 취하지 않고, 통째로 복사하지 않고, 디버거에 기술하지 않는 지역 변수를 슬롯 대신 레지스터에 둘 수 있고, 최적화기는 알려진 값을 그 변수를 읽는 곳에 바로 넘깁니다([8.7.3](807-code-generation.md#873-최적화-수준)). GC 참조를 담은 지역 변수는 늘 슬롯을 유지합니다. shadow stack 프레임이 슬롯의 주소를 들고 있기 때문입니다(8.9.12).

## 8.8.14. 리터럴 풀어 쓰기

[`#[literal]` 매직](../006-meta-system/605-literal-magic.md)의 컬렉션 리터럴의 예입니다.

```nabi
[1, 2, 3]
```

표준 라이브러리 함수 `list_literal(1, 2, 3)` 호출로 풀리고, 일반 함수 호출을 낮추는 규칙을 따릅니다.

문자열 리터럴과 숫자 리터럴은 .rodata에 저장되는 컴파일 타임 `const`로 처리합니다. 그 뒤에는 포인터나 값으로 씁니다.
