# 4.15. `unsafe`: 함수 수식어와 블록 표현식

`unsafe` 키워드는 컴파일러가 안전성을 보증하지 않는 영역을 눈에 보이게 가둡니다. 쓰는 형태는 **함수 / 메서드 선언의 수식어**와 **블록 표현식** 두 가지이며, 둘은 한 쌍으로 동작합니다. `unsafe fn`으로 선언한 함수는 `unsafe { ... }` 블록 안에서 호출해야 합니다.

```
unsafe_fn        ::= "unsafe" fn_decl
unsafe_block     ::= "unsafe" block                // block: 3.9
```

이 모델은 Rust의 `unsafe`와 같은 구조입니다. 목적은 위험 영역을 코드에 명시적으로 드러내는 것입니다.

## 4.15.1. `unsafe fn`: 함수 수식어

함수나 메서드 선언 앞에 `unsafe`를 두면 그 함수의 본체가 안전 추상의 보장 밖에 있다는 뜻입니다. 사전 조건(precondition)을 만족할 책임이 호출자에게 있다는 사실을 시그니처가 바로 알려 줍니다.

```nabi
impl<T> Ptr<T> {
    unsafe fn read(self) -> T;
    unsafe fn write(self, value: T);
}

extern "C" {
    unsafe fn strlen(s: ConstPtr<c_char>) -> usize;
}

unsafe fn transmute_unchecked<A, B>(a: A) -> B { ... }
```

`unsafe fn`의 본체 안에서는 따로 `unsafe { ... }` 블록이 없어도 unsafe 호출을 할 수 있습니다. 함수 시그니처가 이미 unsafe를 선언했으므로 컴파일러는 본체 전체를 unsafe 스코프로 봅니다.

```nabi
unsafe fn copy_two<T>(p: Ptr<T>) -> (T, T) {
    let a = p.read();              // 본체가 unsafe scope: 명시 블록 불필요
    let b = p.add(1).read();
    (a, b)
}
```

### 메서드 수신자와 결합

`impl`의 메서드도 같은 형태를 따릅니다.

```nabi
impl Buffer {
    unsafe fn raw_data(self) -> Ptr<u8>;
    unsafe fn raw_data_mut(ref self) -> Ptr<u8>;
}
```

`unsafe`는 `ref self` / `self` 같은 receiver 표기와 충돌하지 않습니다. 이 키워드는 함수 헤더 맨 앞에 옵니다.

### `extern "C"` 블록 안

사용자가 선언한 C ABI 함수는 정의가 컴파일러의 통제 밖에 있으므로 모두 묵시적으로 `unsafe`입니다([extern 블록](../005-modules/509-extern-block.md)). `unsafe fn`으로 명시하기를 권장합니다. 생략하면 컴파일러가 자동으로 `unsafe`로 처리합니다.

런타임 심볼을 선언하는 `extern "nabi"`를 포함해 모든 `extern` 함수가 그렇습니다. unsafe 범위 밖에서 호출하면 에러(E0061)입니다.

```nabi
extern "C" {
    fn malloc(size: usize) -> CPtr<u8>;        // 자동 unsafe 처리
    unsafe fn free(ptr: CPtr<u8>);             // 명시 (권장)
}
```

## 4.15.2. `unsafe { ... }`: 블록 표현식

unsafe 함수 호출, 원시 포인터 역참조 같은 동작은 `unsafe { ... }` 블록 안에서만 할 수 있습니다. 블록의 결과는 일반 [블록 표현식](../003-expressions/309-block-expressions.md)과 똑같이 마지막 식의 값입니다.

```nabi
let n: usize = unsafe { strlen(cs.as_ptr()) };

let v: i32 = unsafe {
    let p = ptr.add(idx);
    p.read()
};

unsafe {
    free(ptr);
}
```

### 블록 안에서 가능한 동작

`unsafe { ... }` 블록 안에서는 다음 동작을 추가로 할 수 있습니다.

- `unsafe fn`으로 선언한 함수 / 메서드 호출
- 원시 포인터(`Ptr<T>`, `ConstPtr<T>`, `ManyPtr<T>`) 역참조(`p.read()`, `p.write(v)`)
- [`bare::*`](../../refs-standard-libraries/100-bare/README.md) 모듈의 위험 함수 직접 호출
- `union` 필드 읽기(비트 재해석, [union](../002-types/205-union.md)). 필드 쓰기는 `unsafe` 없이 할 수 있습니다.
- 다른 소유자에게 넘긴 값을 그 뒤에 사용([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커)). 다른 스레드가 동시에 그 값을 쓰지 않는다는 점은 블록 작성자가 보장합니다.

그 밖의 일반 안전 동작은 unsafe 블록 안에서도 똑같이 동작합니다. `unsafe`는 '기능을 더할' 뿐 '기존 의미를 바꾸지' 않습니다.

### 범위 좁히기

`unsafe` 블록은 되도록 좁게 두기를 권장합니다. 함수 전체를 `unsafe { ... }`로 감싸지 말고 실제로 위험한 호출 한 줄만 감쌉니다. 그러면 나머지 줄이 안전하다는 점이 눈에 분명해집니다.

```nabi
// 권장: 좁은 unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    let validated = try check_ptr(p);
    unsafe { validated.read() }    // 위험 동작만 unsafe
}

// 비권장: 넓은 unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    unsafe {
        let validated = try check_ptr(p);
        validated.read()
    }
}
```

## 4.15.3. 호출 규칙

`unsafe fn`의 호출은 다음 규칙을 따릅니다.

| 호출 자리                | `unsafe fn` 호출       |
|--------------------------|------------------------|
| 일반 함수 본체           | ✗ (컴파일 에러)        |
| `unsafe fn` 본체         | ✓ (자동 unsafe 스코프) |
| `unsafe { ... }` 블록    | ✓                      |

규칙을 어기면 다음과 같이 컴파일 에러입니다.

```nabi
fn safe_caller(p: Ptr<i32>) -> i32 {
    p.read()    // ERROR (E0061): call to unsafe function `read` outside an `unsafe` block
}
```

해결 방법은 두 가지입니다. `unsafe { ... }`로 감싸거나, 호출자 자신을 `unsafe fn`으로 표시합니다.

### `unsafe fn` 타입

`unsafe fn`과 C 함수를 값으로 쓰면 그 타입은 `unsafe fn(..) -> R`입니다([4.4](404-function-type.md)). 이 타입의 값을 호출하려면 `unsafe`가 필요합니다. 안전한 `fn(..) -> R` 값은 `unsafe fn(..) -> R` 자리에 넣을 수 있지만, 반대로는 넣을 수 없습니다.

```nabi
let f: unsafe fn(usize) -> CPtr<u8> = malloc;   // C 함수 값
let p = unsafe { f(16) };
let g: unsafe fn(i32) -> i32 = |x| x + 1;      // OK: safe → unsafe
let h: fn(usize) -> CPtr<u8> = f;              // ERROR: unsafe → safe
```

## 4.15.4. 트레잇 안에서

`trait`의 메서드도 `unsafe fn`일 수 있습니다. `impl` 쪽은 `unsafe`까지 같은 시그니처로 구현해야 합니다. `unsafe fn` 메서드를 안전한 메서드로 구현하거나 안전한 메서드를 `unsafe fn`으로 구현하면 에러(E0074)입니다. 호출자는 일반 호출 자리와 같은 규칙을 따릅니다.

```nabi
trait RawAccess {
    unsafe fn raw_get(self, idx: usize) -> u8;
}

impl RawAccess for Buffer {
    unsafe fn raw_get(self, idx: usize) -> u8 {
        self.ptr.add(idx).read()
    }
}

let buf: Buffer = ...;
let b = unsafe { buf.raw_get(0) };
```

### `unsafe trait`

impl이 컴파일러가 확인할 수 없는 것을 약속하는 트레잇은 `unsafe trait`로 선언합니다. 그 트레잇의 impl은 모두 `unsafe impl`입니다. 평범한 `impl`은 컴파일 에러(E0078)이고, 에러가 `unsafe impl` 형태를 알려 줍니다. 트레잇을 쓰는 쪽(바운드, 호출)에는 `unsafe`가 필요 없습니다. `bare::mem::Pod`(모든 비트 패턴이 값)가 그런 트레잇입니다([stdlib 1.3.3](../../refs-standard-libraries/100-bare/103-mem.md#133-pod-트레잇)).

```nabi
/// 모든 비트가 0인 패턴이 값이다.
unsafe trait Zeroable {
    fn zeroed() -> Self;
}

unsafe impl Zeroable for i32 {
    fn zeroed() -> i32 { 0 }
}
```

### `unsafe impl`

`unsafe impl`은 `unsafe trait`를 구현하거나 자동 마커 트레잇 `Send` / `Sync`를 직접 구현할 때 씁니다. 트레잇이 약속하는 것(`Send` / `Sync`라면 그 타입이 스레드 경계를 넘어도 안전하다는 것)을 작성자가 보증한다는 표시입니다. 다른 트레잇에 `unsafe impl`을 쓰면 컴파일 에러입니다([2.9.1](../002-types/209-impl.md#291-두-형태)).

```nabi
unsafe impl Send for Handle { }
```

## 4.15.5. 책임 모델

`unsafe`는 컴파일러가 안전 검사를 '포기'한다는 뜻이 아닙니다. 다음 두 책임이 호출자에게 넘어간다는 뜻입니다.

1. **메모리 안전성**: 댕글링 포인터, 잘못된 정렬 접근, 데이터 경쟁 같은 미정의 동작([4.15.8](#4158-미정의-동작))을 일으키지 않을 책임
2. **사전 조건**: `unsafe fn`의 문서가 약속한 사용 조건을 만족할 책임(예: 포인터가 유효한 객체를 가리킬 것, 길이가 버퍼 안일 것)

`unsafe { ... }`는 호출자가 이 두 책임을 받아들인다는 눈에 보이는 표시입니다. 그래서 코드 리뷰 때 위험 영역이 한눈에 들어옵니다.

## 4.15.6. 묵시적 unsafe와 관계

컴파일러는 다음 자리를 `unsafe`를 적지 않아도 자동으로 unsafe 스코프로 봅니다.

- `unsafe fn`의 본체

이 자동 처리 덕분에 위험 영역이 자주 나오는 곳에서 `unsafe { ... }`를 거듭 적지 않아도 됩니다.

## 4.15.7. 다른 언어와 비교

| 언어    | unsafe 표기 모델                                       |
|---------|--------------------------------------------------------|
| Rust    | `unsafe fn` + `unsafe { ... }` (동형)                  |
| C / C++ | 별도 표기 없음, 모든 코드가 잠재적으로 unsafe          |
| Swift   | `@unsafe` 어노테이션 / `withUnsafePointer` 함수형 패턴 |
| Go      | `unsafe` 패키지 (별도 import), 블록 표기 없음          |
| Zig     | `@ptrCast` 등 명시 builtin, 블록 표기 없음             |
| **Nabi** | **Rust 식 `unsafe fn` + `unsafe { ... }` 채택**        |

Nabi는 Rust의 모델을 그대로 따릅니다. 키워드 하나로 위험 영역을 일관되게 구분해 보여 주므로 코드 리뷰와 정적 분석 모두에 유리합니다.

## 4.15.8. 미정의 동작

Nabi의 미정의 동작은 아래 목록이 전부입니다. 실행 중 이 가운데 하나를 하는 프로그램에는 의미가 없습니다. 컴파일러는 그런 일이 없다고 가정할 수 있고, 그 뒤에는 무엇이든 일어날 수 있습니다. 어느 항목이든 `unsafe` 코드(또는 외부 코드)가 있어야 일어나며, 안전한 Nabi는 이 가운데 어느 것도 일으킬 수 없습니다. 라이브러리 각 장에서 미정의 동작이라고 하는 것은 모두 이 가운데 한 종류입니다.

메모리와 포인터:

1. 그 타입의 살아 있는 메모리를 가리키지 않는 포인터로 읽거나 쓰기: null, 댕글링(포인터가 나온 지역 변수, pin, 할당이 사라짐), 객체 범위 밖, 또는 호출자가 보증한 `ManyPtr` / `GcBuf` / `Slice::from_raw_parts` 범위 밖([stdlib 1.1](../../refs-standard-libraries/100-bare/101-ptr.md), [stdlib 1.3.9](../../refs-standard-libraries/100-bare/103-mem.md)).
2. GC 객체 안을 가리키는 원시 포인터를, 그 포인터를 유효하게 한 pin이 풀린 뒤에 쓰기([stdlib 1.6](../../refs-standard-libraries/100-bare/106-gc-pin.md)).
3. `ptr_cast`가 만들 수 있는, 정렬이 맞지 않는 포인터로 읽거나 쓰기([stdlib 1.2.2](../../refs-standard-libraries/100-bare/102-cast.md)).
4. provenance를 잃은 포인터를 역참조하기. 노출된 주소에서 `from_exposed_provenance`로 만든 포인터는 예외입니다([stdlib 1.1.2](../../refs-standard-libraries/100-bare/101-ptr.md)).
5. `mut`이 아닌 `static`을 가리키는 포인터로 쓰기, 또는 `const_cast`의 결과로 쓸 수 없는 메모리에 쓰기([stdlib 1.1.2](../../refs-standard-libraries/100-bare/101-ptr.md), [stdlib 1.2.4](../../refs-standard-libraries/100-bare/102-cast.md)).
6. 겹치는 범위에 `copy_nonoverlapping` 쓰기([stdlib 1.3.4](../../refs-standard-libraries/100-bare/103-mem.md)).
7. `drop_in_place`가 drop glue를 실행한 값을 다시 쓰거나, 두 번 실행하기([stdlib 1.3.4](../../refs-standard-libraries/100-bare/103-mem.md)).

값:

8. 어떤 타입으로든 초기화되지 않은 메모리 읽기: 값을 쓰기 전의 `MaybeUninit::assume_init`, 아무것도 쓰지 않은 바이트 읽기([stdlib 1.3](../../refs-standard-libraries/100-bare/103-mem.md)).
9. `union` 읽기([2.5.2](../002-types/205-union.md)), `transmute_unchecked`, 원시 쓰기, 거짓 `unsafe impl Pod`로 그 타입에 유효하지 않은 값 만들기: `0` / `1`이 아닌 `bool`, 유니코드 스칼라 값이 아닌 `char`, 어떤 변형에도 없는 enum이나 합 타입의 태그, null도 살아 있는 객체도 아닌 GC 참조, null인 `none` 아닌 `extern "C" fn`([stdlib 1.2.6](../../refs-standard-libraries/100-bare/102-cast.md), [5.9.4](../005-modules/509-extern-block.md)).
10. `new_unchecked`로 만든 검사 래퍼의 불변식 깨기: 0인 `NonZero`, NaN인 `NonNan`, NaN이나 무한대인 `Finite`, 음수인 `Positive`([stdlib 2.2.6](../../refs-standard-libraries/200-core/202-non-zero.md)).
11. 수집기가 훑지 않는 GC 힙 밖 메모리에 놓인 GC 참조: 그런 메모리 위의 `ManyPtr::slice`가 GC 참조를 담는 경우, 원시 쓰기로 그곳에 저장한 경우([stdlib 1.1.4](../../refs-standard-libraries/100-bare/101-ptr.md)).

스레드:

12. 데이터 경쟁([7.9.4](../007-runtime/709-memory-model.md#794-데이터-경쟁)): 원시 포인터나 `GcBuf`를 거친 경우, 거짓 `unsafe impl Send` / `Sync`, 다른 것이 닿는 값에 쓴 `Mutex::new_unshared`, `unsafe` 안에서 건너뛴 넘기기 검사, `env::set_var`가 도는 동안 환경을 건드리는 C 코드([stdlib 3.5](../../refs-standard-libraries/300-std/305-env.md)).

외부 코드:

13. 선언한 타입이 외부 정의와 맞지 않는 외부 함수, `static`, 콜백, 또는 C가 원시 비트로 쥐는 자리에 넘긴 GC 값([5.9.4](../005-modules/509-extern-block.md), [5.9.6](../005-modules/509-extern-block.md)).
14. 외부 코드의 예외나 unwind가 Nabi 프레임을 건너는 것([5.9.2](../005-modules/509-extern-block.md)).
15. 선언이 `str`인데 null `char*`를 돌려주는 것, C가 호출이 끝난 뒤에도 `str` 인자의 포인터를 쥐는 것([5.9.4](../005-modules/509-extern-block.md)).
16. 댕글링이거나 NUL 없는 바이트를 가리키는 포인터에 쓴 `CStr::from_ptr`([stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md)).

그 밖의 것은 안전한 코드와 unsafe 코드 모두에서 정의되어 있습니다. 특히 다음은 미정의 동작이 아닙니다. 정수 오버플로(검사가 켜져 있으면 panic, 꺼져 있으면 wrap, [stdlib 2.3](../../refs-standard-libraries/200-core/203-ops.md#오버플로-동작)), 컬렉션의 범위 밖 인덱스나 슬라이스(panic), 0으로 나누기(panic), 스택 오버플로(프로세스 중단, [7](../007-runtime/README.md)), NaN과 무한대를 포함한 모든 부동소수점 연산([stdlib 2.1.4](../../refs-standard-libraries/200-core/201-numbers.md#부동소수점-의미)), `union` 필드 읽기(비트 재해석이며, 만들어진 값이 유효하지 않을 수 있을 뿐입니다, 9번 항목), 크기가 다른 타입 사이의 `transmute_unchecked`(모자란 바이트는 0으로 읽힘), 교착 상태, 누수.
