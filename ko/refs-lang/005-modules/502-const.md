# 5.2. `const`

`const` 키워드(항목 시작에서만 키워드인 문맥 키워드, [1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드))는 컴파일 타임에 정해지는 값을 선언합니다. 문법은 다음과 같습니다.

```
const_decl      ::= "const" identifier ":" type_expr "=" const_expr ";"
const_fn        ::= "const" fn_decl
const_block     ::= "const" block
const_expr      ::= expr                        // evaluable at compile time (5.2.4)
```

## 5.2.1. 기본 형태

```nabi
const PI: f64 = 3.14159f64;
const MAX_RETRY: i32 = 3i32;
const NAME: str = "nabi";
```

선언은 `const` 키워드, 이름, `:`, 타입, `=`, `const` 표현식 순서로 이루어집니다. 타입은 명시해야 합니다. `let`과 달리 추론하지 않습니다. 값 자리에는 `const` 표현식, 곧 컴파일 타임에 평가할 수 있는 식만 올 수 있습니다.

## 5.2.2. 가능한 위치

`const`는 다음 위치 어디에서나 선언할 수 있습니다.

```nabi
// 모듈 수준
const PI: f64 = 3.14159f64;

// 함수 본체 안
fn area(r: f64) -> f64 {
    const FACTOR: f64 = 3.14159f64;
    FACTOR * r * r
}

// impl 블록 안
impl Circle {
    const UNIT_AREA: f64 = 3.14159f64;
    fn unit() -> Circle { Circle { r: 1.0f64 } }
}

// 트레잇 안: 연관 const
trait Numbers {
    const ZERO: Self;
    const ONE: Self;
}
```

함수 본체의 `const`는 그 블록의 항목입니다([3.9.1](../003-expressions/309-block-expressions.md#391-형태)). 블록 전체와 그 안에 선언한 항목에서 보입니다. 둘러싼 코드의 지역 변수나 매개변수는 읽지 않으며, 에러는 대신 `let`을 쓰라고 알려 줍니다(`let half = n / 2;`). 자신이 속한 제네릭 함수나 `impl`의 const 매개변수, 타입 매개변수의 연관 const와 `T::size()`는 읽을 수 있습니다. 그러면 인스턴스마다 값을 가집니다(5.2.7).

```nabi
fn mid<const N: usize>(xs: [i32; N]) -> i32 {
    const HALF: usize = N / 2;      // 인스턴스마다 값 하나
    xs[HALF]
}
```

## 5.2.3. `const`의 의미: 사용처마다 인라인

`const`는 주소가 아니라 값입니다. 쓰이는 자리마다 그 값이 인라인됩니다.

```nabi
const X: i32 = 42i32;

fn f() {
    let a = X;                    // a = 42i32 (X 가 인라인)
    let b = X + 1i32;              // b = 43i32 (X 가 인라인 후 평가)
}
```

`const`의 결과는 어떤 메모리 위치에도 있지 않으므로 주소를 얻을 수 없습니다.

```nabi
let p = &X;                       // ERROR (E0012): Nabi에는 `&` 빌림이 없음
```

Nabi에는 `&` 연산자가 없으므로 `const`의 주소를 얻는 식은 없습니다. [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)용 raw 포인터는 [타입 표현식](../002-types/201-type-expressions.md)의 포인터 타입에서 정의합니다.

## 5.2.4. 상수 식과 평가기

`const` 항목, 배열 길이, const 제네릭 인자, `struct` 필드 기본값, `static` 초기값, 상수 인자를 받은 `const fn` 리터럴 핸들러([6.5.8](../006-meta-system/605-literal-magic.md#658-리터럴-핸들러의-const-호환))는 모두 같은 컴파일 타임 평가기를 씁니다.

평가할 수 있는 것은 다음과 같습니다.

- 값: 모든 폭의 정수, 부동소수, `bool`, `char`, `str`, 튜플, `struct`, `enum`, 합 타입, 고정 크기 배열.
- 식: 리터럴, 다른 `const`의 참조, const 제네릭 매개변수, 연산자, `const fn` 호출.
- 제어 흐름: `if`, `match`, `loop` / `while` / `for`, 라벨 `break`, `return`, `throw`. 재귀와 제네릭 `const fn`도 쓸 수 있습니다.

평가할 수 없는 것: `const fn`이 아닌 함수 호출(I/O, 시각, 난수 등), `static` 읽기, 바깥 `let` 변수의 참조.

다음은 컴파일 에러입니다.

- 정수 오버플로(리터럴과 상수만으로 된 식을 접은 결과 포함, `let m: u8 = 2 * 200`), 0으로 나누기, `MIN / -1`, 비트 폭 이상의 시프트, 범위 밖의 부동소수 → 정수 변환. 감싸는(wrapping) 산술은 `wrapping_*`로만 합니다.
  상수 조건이 배제하는 코드는 실행되지 않으므로 검사하지 않습니다. `N = 0`인 제네릭 인스턴스에서 `if N == 0 { 0 } else { a[N - 1] }`, `if N == 0 { return ..; }` 뒤의 블록 나머지, `N > 0 && a[N - 1] == x`의 `N - 1`은 에러가 아닙니다. 인스턴스 안의 오버플로는 그 인스턴스를 만든 호출과 const 인자를 함께 보고합니다.
- 평가 중에 일어난 panic과 잡히지 않은 `throw`.
- 평가 단계 수나 호출 깊이의 한도 초과. 한도는 구현이 정합니다(nabi: 4,000,000 단계, 깊이 128).
- `const` 사이의 순환 참조.

## 5.2.5. `const fn`: 컴파일 타임 평가 가능한 함수

```nabi
const fn double(x: i32) -> i32 {
    x * 2i32
}

const fn gcd(a: i64, b: i64) -> i64 {
    if b == 0i64 { a } else { gcd(b, a % b) }
}

const TWO: i32 = double(1i32);                  // 컴파일 타임
const TWELVE: i64 = gcd(24i64, 36i64);          // 컴파일 타임

let y = double(user_input);                      // 런타임 호출
```

- `const fn`은 `const fn`만 부를 수 있습니다. `const fn` 안에서 `fn` 값이나 클로저로 하는 호출은 정적 검사가 따라갈 수 없으므로 평가할 때만 검사합니다. 표준 라이브러리는 순수 함수를 `const fn`으로 표시합니다.
- 본체는 순수해야 합니다. I/O와 가변 전역 접근은 에러입니다.
- 컴파일 타임 평가에 성공한 결과는 같은 인자의 런타임 결과와 같습니다. 런타임에 오버플로가 날 연산은 컴파일 타임에 에러입니다(5.2.4).
- 타입은 인자로 받을 수 없습니다(`const fn sizeof(T: type)` 같은 선언은 에러이고, 도움말은 `const fn sizeof<T>() -> usize { T::size() }`를 씁니다). 타입 인트로스펙션은 메타 블록이 맡습니다.
- `const fn`은 런타임에도 일반 함수처럼 부를 수 있습니다.

### 제네릭 `const fn`

```nabi
const fn id<T>(x: T) -> T { x }

const A: i32 = id(42i32);                        // T = i32 단형화 후 평가
```

### 힙을 쓰는 결과

결과에 힙 할당(`List`, 버퍼, 클로저 등)이 있으면 바이너리에 넣지 않습니다. 그런 `const`는 컴파일 타임에 평가해 검사하고, 사용처에서 런타임에 만듭니다.

사용할 때마다 새 값이므로 `const`에 값을 바꾸는 호출(`List`의 `XS.push(4)`, `const` 구조체의 `ref self` 메서드)은 곧 버려질 사본을 바꾸게 됩니다. 이는 컴파일 에러입니다. 값을 변수에 복사해 그 변수를 바꿉니다(`let xs = XS; xs.push(4);`).

## 5.2.6. `const`와 다른 메커니즘의 직교성

```
generic <T>     = 타입 파라미터화 (call-site specialize)
meta block      = 코드 생성 (impl, struct 등 AST 레벨)
const fn        = 값 계산 (compile-time pure 함수)
```

세 메커니즘은 서로 직교합니다. 영역이 겹치지 않으므로 자유롭게 조합할 수 있습니다.

## 5.2.7. `const`의 평가 시점

모듈 수준, 함수 본체, `impl` 수준의 `const`는 패키지 빌드 때 한 번 평가합니다. 제네릭 매개변수에 의존하는 `const`(제네릭 `impl`의 연관 `const`, const 매개변수를 읽는 함수 본체 `const`나 `const { .. }` 블록)는 단형화마다 평가합니다. 힙을 쓰지 않는 결과는 바이너리에 들어가므로 런타임 비용이 없습니다.

## 5.2.8. `struct` 필드 기본값과 관계

[`struct`](../002-types/203-struct.md)의 필드 기본값은 상수 식입니다(5.2.4).

## 5.2.9. `const` 제네릭과 관계

[const 제네릭](../002-types/211-const-generics.md)의 인자는 타입의 일부가 되는 상수입니다. 같은 평가기로 평가합니다(5.2.4).

## 5.2.10. `const` 블록

`const { .. }`는 평가기(5.2.4)가 컴파일 타임에 평가하는 식입니다. 값은 블록 타입의 상수이고, 타입이 `()`인 블록은 검사입니다.

```nabi
fn chunk<const N: usize>(xs: [i32; N]) -> usize {
    const { assert(N > 0, "chunk needs at least one element") };
    const { N / 2 }
}
```

- 함수 본체 `const`가 읽는 것을 읽습니다. 상수, `const fn` 호출, 둘러싼 코드의 const 매개변수, 타입 매개변수의 연관 const와 `T::size()` / `T::align()`입니다. `let` 변수는 에러이고, 도움말은 `const`를 빼라고 알려 줍니다.
- 제네릭 함수 안에서는 인스턴스의 인자가 정해진 뒤 인스턴스마다 한 번 평가합니다. 정의 시점의 제네릭 본체 검사([2.10.9](../002-types/210-generics.md#2109-본체-검사))는 블록의 타입만 검사하고 평가하지 않습니다.
- 평가 중의 패닉이나 실패한 `assert`는 그 인스턴스를 만든 호출에서 나는 컴파일 에러입니다. 에러는 인스턴스 이름을 대고(`` const block failed in `chunk::<0>`: chunk needs at least one element ``), `assert`를 보조 위치로 보여 줍니다. `nabi check`도 이를 보고합니다. `nabi build`처럼 프로그램이 닿는 것을 인스턴스화하기 때문입니다.
- 통과한 블록은 런타임 비용이 없습니다. 값은 다른 `const`처럼 바이너리에 들어갑니다.
- 평가기는 단순 타입(숫자, `bool`, `char`, `str`, `()`, 이들의 배열)의 `T::size()`를 압니다. 다른 타입에서는 const 매개변수나 `List`를 쓰라는 에러입니다.
