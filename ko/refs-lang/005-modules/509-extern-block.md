# 5.9. `extern` 블록([FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 선언)

이 절은 외부 함수와 전역을 선언하는 블록을 정의합니다. 블록은 `extern` 키워드와 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 지정(`"C"` 등)으로 시작합니다. 이 키워드는 항목 시작에서만 키워드인 문맥 키워드입니다([1.5.4](../001-lexical/105-keywords.md#154-문맥-키워드)). 외부 라이브러리, 시스템 콜, 다른 언어와 연동할 때 씁니다.

```
extern_block    ::= "extern" string_lit "{" extern_item* "}"
extern_item     ::= visibility? "fn" identifier "(" param_list? ")" return_type? ";"
                  | visibility? "static" "mut"? identifier ":" type_expr ";"
```

## 5.9.1. 기본 형태

기본 사용법은 다음과 같습니다.

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
    fn strlen(s: ConstPtr<c_char>) -> usize;

    static environ: Ptr<Ptr<c_char>>;
    static mut atexit_count: u64;
}
```

블록은 `extern`, ABI 문자열(`"C"`, `"system"` 등), `{ ... }`로 이루어집니다. ABI별 의미는 [호출 규약](../008-abi/806-calling-convention.md)에서 정의합니다. 본체에는 본체 없이 시그니처만 있는 함수 선언과 `static` 선언만 올 수 있습니다.

> [!IMPORTANT]
> `extern "C"` 블록 안의 함수 선언은 **묵시적으로 unsafe**입니다. 호출하는 쪽은 `unsafe { ... }` 블록 안에서만 호출할 수 있습니다. 시그니처에 `unsafe`를 따로 적지 않아도 효과는 같습니다. 자세한 정의는 [4.15 unsafe](../004-functions/415-unsafe.md)에 있습니다.

## 5.9.2. ABI 명시

지원하는 ABI는 다음과 같습니다.

| ABI        | 의미                                                                       |
|------------|----------------------------------------------------------------------------|
| `"C"`      | 플랫폼 C ABI. 심볼 이름과 무관하게 항상 같습니다.                          |
| `"system"` | OS 시스템 콜 ABI. 일부 플랫폼에서 `"C"`와 다릅니다(예: Win32 `stdcall`).   |
| `"nabi"`    | 런타임(`nabir`)의 ABI. 아래 참고.                                            |
| `"objc"`   | Objective-C 클래스와 그 메서드. Apple 타깃 전용([stdlib 1.10](../../refs-standard-libraries/100-bare/110-objc.md)). `ffi::objc::import { "AppKit/AppKit.h" }`가 SDK 헤더에서 이 블록을 씁니다([stdlib 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-헤더-ffiobjcimport)). |

`"system"`과 `"C"`가 일치하는 타깃(32비트 x86 Windows를 뺀 지원하는 모든 타깃)에서는 `"system"`이 `"C"`입니다. 32비트 x86 Windows에서는 stdcall입니다([5.9.10](#5910-extern-fn-정의외부에-노출)).

`extern "nabi"`은 toolchain이 선언하는 런타임 함수의 ABI입니다. 그 선언은 toolchain 패키지의 비공개 항목입니다. 대부분은 `core::rt`에 있고, `bare`는 자기가 쓰는 것을 따로 선언합니다. `extern "nabi"` 블록의 `#[no_safepoint]`는 그 함수가 할당도 블로킹도 하지 않는다고 선언합니다. 그래서 그 호출은 safepoint가 아닙니다. 이 속성은 toolchain 내부용이며, `extern "C"` 블록에 쓰면 에러입니다.

| 값                          | `extern "nabi"`의 전달                    |
|-----------------------------|------------------------------------------|
| `str`                       | `(ptr, len)` 두 word                     |
| `fn` 값                     | `{code, env}` ([8.5.0a](../008-abi/805-extern-symbols.md#850a-nabifn-함수-값의-c-abi-형태)) |
| `extern "C" fn(..)` 매개변수 | env 없는 C 함수 포인터                   |
| 그 밖                       | 값 그대로(C ABI와 같음)                  |

> [!NOTE]
> `"Rust"` 같은 다른 언어의 ABI는 대개 안정적이지 않으므로 `extern`의 대상이 아닙니다. 다른 언어와는 양쪽이 합의한 안정적인 ABI(보통 `"C"`)로만 연동할 수 있습니다.

> [!NOTE]
> Nabi는 에러를 반환값으로 전달하고 스택 되감기(unwinding)를 쓰지 않습니다([8.6.5](../008-abi/806-calling-convention.md#865-던지는-함수의-반환-값-기반-에러-전달)). 따라서 외부 언어의 예외나 되감기가 Nabi 프레임을 가로지르면 미정의 동작(UB)입니다. 외부 함수는 호출 경계 안에서 예외를 처리해야 합니다.

ABI가 없는 `extern { ... }` 형태는 컴파일 에러입니다. ABI를 명시해야 합니다.

## 5.9.3. 본체에 등장 가능한 항목

### 외부 함수 선언

외부 함수는 다음과 같이 선언합니다.

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
}
```

시그니처는 일반 함수 선언과 형태가 같지만, 본체 없이 `;`으로 끝납니다. 매개변수와 반환 타입에는 원시 정수, 부동소수, 포인터 같은 FFI 호환 타입만 쓸 수 있습니다.

### 외부 `static`

외부에서 정의한 전역 변수는 다음과 같이 참조합니다.

```nabi
extern "C" {
    static environ: Ptr<Ptr<c_char>>;  // read-only
    static mut atexit_count: u64;      // mutable
}
```

이 형태는 외부에서 정의한 전역 변수를 참조합니다. `static mut`는 `extern` 블록 안에서만 쓸 수 있습니다([`static`](503-static.md)). C의 `errno`는 그런 변수가 아닙니다. 스레드마다 다른 매크로이므로 `static errno`는 엉뚱한 메모리를 읽습니다. 선언에 `#[errno]`를 붙이면 호출 뒤에 errno를 저장합니다([stdlib 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errno-errno와-getlasterror)).

### 등장 불가

`extern` 블록의 본체에는 `struct`, `enum`, `impl`, `fn`의 본체, `const` 등이 올 수 없습니다. 이 블록은 외부 ABI와 맞닿는 경계만 선언합니다.

## 5.9.4. FFI 호환 타입

매개변수, 반환값, `static`의 타입은 모두 FFI 호환이어야 합니다. FFI 타입은 다음과 같습니다.

- 원시 정수: `i8` ~ `i64`, `u8` ~ `u64`, `usize`, `isize`.
- C의 이름 붙은 타입: 타깃의 C 컴파일러가 정하는 크기대로 위 원시 타입을 가리키는 별칭입니다. `c_int`, `c_uint`, `c_long`, `c_ulong`, `c_size_t`, `c_wchar`와 그 밖의 `bare::ffi::types`이고, 모두 prelude에 있습니다([stdlib 1.7.9](../../refs-standard-libraries/100-bare/107-ffi.md#179-c-타입-c_int-c_long-c_void)). `c_long`은 Windows와 32비트 타깃에서 32비트이므로, C의 `long`은 `i64`가 아니라 `c_long`으로 선언합니다. `c_void`는 포인터 뒤의 `void`입니다(`CPtr<c_void>`).
- 원시 부동소수: `f32`, `f64`.
- `bool`(C의 `bool`)과 `c_char`(1바이트).
- `f16` / `bf16`: C의 `_Float16` / `__bf16`. 타깃의 C에 그 타입이 있을 때만 씁니다([8.6.3](../008-abi/806-calling-convention.md#863-struct-인자--반환-작은-struct-규칙)).
- 8바이트나 16바이트인 `Simd<T, N>`(매개변수와 결과): 타깃의 같은 크기 C 벡터(`float32x4_t`, `__m128`). AAPCS64와 x86-64에서 씁니다([8.6.3](../008-abi/806-calling-convention.md#863-struct-인자--반환-작은-struct-규칙)).
- 포인터: `Ptr<T>`, `ConstPtr<T>`, `CPtr<T>`(`bare::ptr`, [표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)의 가장 아래 계층).
- 필드가 모두 FFI 타입인 `struct`와 `union`(그런 타입의 고정 배열 필드 포함). 기본 레이아웃이 이미 C의 레이아웃이고, `#[repr(packed)]`와 `#[repr(align(N))]`은 C에서처럼 레이아웃을 바꿉니다.
- 정수 `#[repr]`(`#[repr(i32)]`)이 붙은 필드 없는 enum: C의 `enum`.
- 함수 포인터: `extern "C" fn(...) -> ...`(아래 "C 함수 포인터 타입" 참고).
- 함수 타입 매개변수: `extern "C"` 선언의 `fn(..) -> R` 매개변수는 `extern "C" fn(..) -> R`과 같습니다. 인자는 함수 이름이어야 합니다. 클로저는 에러(`` `each` is a C function: a `fn` argument must name a function (a C callback has no room for a closure's captures) ``)이고, 그 help가 상태를 둘 곳을 알려 줍니다. `` a C callback cannot capture: keep the state in `Handle::new(..)` (`bare::ffi`), pass `h.as_ptr()` as the `void*` argument, and read it back in an `extern "C" fn` with `Handle::<T>::from_ptr(p).get()` ``([stdlib 1.7.10](../../refs-standard-libraries/100-bare/107-ffi.md#1710-handlet-c가-쥐는-nabi-값)).
- 널 가능 포인터: `Ptr<T> | none`, `ConstPtr<T> | none`, `extern "C" fn(..) -> R | none`(널 니치가 있는 포인터, [2.7.2](../002-types/207-niche-optimization.md#272-자동-niche)), 그리고 `#[niche(0)]` struct `| none`(`NonZero<i32> | none`은 `int`). `none`은 null입니다.
- `extern "C"` 선언의 매개변수인 `str`: 아래 규칙으로 변환합니다.
- 결과 자리의 `()`와 `!`.

`extern "C"` 선언의 `str` 매개변수는 NUL로 끝나는 `char*` 하나입니다. 바이트 뒤에 NUL이 있으면 그 바이트를 그대로 넘기고, 없으면(슬라이스 등) 호출 동안 고정한 사본을 넘깁니다. C는 호출이 끝난 뒤 그 포인터를 보관하면 안 됩니다. `str` 결과는 에러입니다. 돌려받은 `char*`의 바이트는 C의 것이어서(다음 호출이 덮어쓰는 정적 버퍼, 호출자가 해제해야 하는 메모리, 호출과 함께 사라지는 인자 사본 안의 포인터) 그 위의 `str`은 바뀌거나 허공을 가리킬 수 있고, UTF-8이라는 보장도 없습니다. 결과를 `ConstPtr<c_char> | none`으로 선언하고 `CStr::from_ptr`로 문자열을 복사합니다([stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md#172-cstr--cstring-null-종료-문자열)). UTF-8인지도 거기서 알 수 있습니다.

```nabi
extern "C" {
    fn getenv(name: str) -> ConstPtr<c_char> | none;
}
let home = match unsafe { getenv("HOME") } {
    p: ConstPtr<c_char> => unsafe { CStr::from_ptr(p) },
    none => none,
};
```

모듈 `m`의 `extern` 블록에 있는 `pub` 함수는 `m::f`로도 닿을 수 있습니다. 그 심볼에는 접두어가 붙지 않습니다. `extern "C"` 블록의 `nabi_*` 이름은 `str`을 받거나 돌려줄 수 없습니다([8.5](../008-abi/805-extern-symbols.md)).

`extern "C"`와 `extern "system"` 블록, `extern "C" fn` 정의의 매개변수, 결과, `static`에 그 밖의 타입이 있으면 에러입니다. 검사는 해석이 끝난 타입에서 하므로 별칭(`type S = str;`), 제네릭 인스턴스, 필드, 배열 원소도 꿰뚫어 봅니다. 에러마다 대신 쓸 것을 알려 줍니다.

| 타입 | 이유 | 대신 쓸 것 |
|------|------|-----------|
| GC 타입(`List`, `Dict`, 클로저, `any`, 이런 것을 담은 struct) | C가 쥐고 있는 동안 수집기가 옮기거나 해제할 수 있음 | 기본 타입, `CPtr<T>`와 길이, 또는 그런 것으로 된 struct. `List<T>`는 `gc::pin`으로 고정하고 `p.as_data_ptr()`와 `xs.len()`으로 넘김 |
| 결과의 `str`, `extern "C" fn` 정의의 `str` | 바이트가 C의 것임(위), C는 `char*`를 넘김 | `ConstPtr<c_char> \| none`, `CStr::from_ptr`로 복사 |
| `char` | C에는 4바이트 문자가 없음 | `u32`와 `char::from_u32` |
| 튜플 | C에는 튜플이 없음 | `struct` |
| 일반 enum | 레이아웃이 컴파일러의 것임 | `#[repr(i32)]`, 또는 프로그램이 매치하는 `i32` |
| 데이터가 있는 enum | C에는 태그 붙은 enum이 없음 | 태그와 `union`으로 된 struct |
| 배열 매개변수나 결과 | C는 배열을 포인터로 넘김 | `ConstPtr<T>`(`addr_of(xs)`), 또는 struct에 담은 배열 |
| 그 밖의 합 타입(`i32 \| none`) | null 포인터만 C 형태가 있음 | `ConstPtr<T> \| none`, 또는 값과 플래그 |
| 선언의 매개변수가 아닌 자리의 nabi `fn` 값 | 함수와 캡처로 이루어짐 | 이름 붙은 `extern "C" fn`으로 만든 `extern "C" fn(..)` |
| `Mask<T, N>` | C 타입이 없음 | 정수 `Simd`(`mask.select(Simd::splat(1), Simd::splat(0))`) |
| `Simd`를 담고 값으로 넘기는 struct | 집합체 안의 C 벡터는 넘기지 않음 | struct를 가리키는 포인터 |

메시지는 매개변수와 타입을 밝힙니다: `` parameter `xs` of extern function `f` has type `List<i64>`, which C cannot hold: the collector may move or free it ``. `List<T>`이면 help가 쓸 호출을 그대로 보여 줍니다: `` help: C takes a list as a pointer and a length: declare `ptr: ConstPtr<i64>, len: usize`, and pass `p.as_data_ptr()` and `xs.len()` from `let p = gc::pin(xs);` (`bare::gc`) ``. 타깃의 C에 `_Float16`, `__bf16`, 또는 그 크기의 벡터가 없으면 그 타깃의 코드 생성 단계에서 같은 꼴의 도움말과 함께 에러가 납니다. 컴파일러가 볼 수 없는 것은 검사하지 못합니다. C 쪽이 같은 함수를 같은 타입으로 선언하는지는 프로그램이 보장해야 합니다. 타입별 호출 규약은 [호출 규약](../008-abi/806-calling-convention.md)에서 정의합니다.

### C 함수 포인터 타입

```
c_fn_type ::= "unsafe"? "extern" ("\"C\"" | "\"system\"") "fn" "(" (type_expr ("," type_expr)*)? ")" ("->" unary_type)?
```

- ABI 문자열은 `"C"`나 `"system"`만 쓸 수 있습니다. 그 밖은 에러입니다.
- `extern "system" fn(..) -> R`은 `extern "C" fn(..) -> R`과 같은 규칙을 따르므로 Win32 콜백을 표현할 수 있습니다. 두 타입이 다른 타입이 되는 곳은 stdcall을 쓰는 32비트 x86 Windows뿐입니다. 그 밖에서는 같은 타입입니다.
- 이 타입의 값은 이름 있는 C-ABI 함수로만 만듭니다. `extern` 블록에 있는 C 선언의 값은 `unsafe extern "C" fn(..)` 타입이므로 안전한(`unsafe` 없는) 자리에 넣을 수 없습니다. Nabi의 `extern "C" fn` 정의(그 C 진입점, 5.9.10)는 안전한 `extern "C" fn(..)` 값을 줍니다. 그 자리의 일반 `fn`은 에러 `` only an `extern "C"` function (an `extern` block declaration or an `extern "C" fn` definition) is an `extern "C" fn(..)` value ``이고, help는 `` C calls it with the C ABI: declare it `extern "C" fn twice(..)` ``입니다.
- Nabi `fn` 값(클로저, 함수 타입 변수 포함)과 C 함수 포인터는 서로 변환되지 않습니다.
- `unsafe`가 없는 타입으로 하는 호출은 안전합니다. `unsafe extern "C" fn`으로 호출할 때는 `unsafe`가 필요합니다.
- `none`이 아닌 C 함수 포인터 값은 null이 아닙니다. 그래서 `| none`이 없는 `extern "C" fn` 자리(매개변수, 반환, 필드)로 NULL을 받으면 미정의 동작입니다. NULL을 돌려줄 수 있는 C API는 `| none`으로 선언해야 합니다.

#### null이 될 수 있는 C 함수 포인터: `extern "C" fn(..) -> R | none`

C 함수 포인터 타입 하나와 `none`만으로 된 합(순서 무관, 다른 항 없음)은 null niche를 씁니다([2.7.2](../002-types/207-niche-optimization.md#272-자동-niche)).

- 포인터 크기입니다. `none`은 null입니다.
- C의 nullable 함수 포인터와 ABI가 같습니다. extern 매개변수 / 반환, C와 주고받는 struct 필드, extern `static`에 그대로 쓸 수 있습니다.
- 0으로 채운 저장소는 `none`으로 읽힙니다.
- 이 합을 포함하는 더 큰 합은 일반 태그 배치를 씁니다. 둘 사이의 변환은 일반 합 타입 변환입니다.

```nabi
extern "C" {
    fn cmp_i32(a: ConstPtr<u8>, b: ConstPtr<u8>) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32);
}
let f: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;   // C 선언의 값은 unsafe
let g: extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;          // ERROR: 안전한 자리
```

### extern 함수의 이름 공간

`extern` 블록에 선언한 함수는 전역 이름 공간 하나를 이룹니다. 어느 모듈에서든 접두 없는 이름으로 부를 수 있습니다. 이 규칙은 링크 시점에 C 심볼이 전역 공간 하나에 모여 있다는 점을 반영합니다. 일반 항목에 적용되는 "조상 모듈 암묵 탐색 없음" 규칙([5.7.1](507-paths.md#571-경로의-시작-식별자))의 예외입니다. extern 함수에만 적용됩니다. 같은 이름의 extern 함수를 다른 시그니처로 두 번 선언하면 컴파일 에러입니다. `core::rt`에 있는 toolchain 런타임 선언도 여기에 포함됩니다. `nabi_*` 런타임 함수를 다시 선언하는 사용자 `extern`은 그 시그니처와 같아야 합니다.

## 5.9.5. `Ptr<T>` / `ConstPtr<T>`

[표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)의 가장 아래 계층인 `bare::ptr`에서 정의하는 원시 포인터 타입입니다. prelude에는 없으므로 `use bare::ptr::{Ptr, ConstPtr};`로 가져옵니다.

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
    fn read_byte(p: ConstPtr<u8>) -> u8;
}
```

`Ptr<T>`는 가변 원시 포인터로, C의 `T*`에 대응합니다. `ConstPtr<T>`는 읽기 전용 원시 포인터로, C의 `const T*`에 대응합니다. 두 타입 모두 null이 되지 않습니다. null일 수 있는 포인터는 `Ptr<T> | none`(또는 `ConstPtr<T> | none`)입니다. 주소 하나로 표현되고 null이 `none`입니다([niche 최적화](../002-types/207-niche-optimization.md)). `CPtr<T>`는 NULL을 값으로 담을 수 있는 C 포인터입니다(`is_null()`, `as_ptr() -> Ptr<T> | none`).

## 5.9.6. 호출 / 사용

`extern` 블록에 선언한 함수는 일반 Nabi 함수처럼 호출합니다.

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
}

fn use_malloc() {
    let p = unsafe { malloc(1024usize) };
    if p is Ptr<u8> {
        // ... 사용 ...
        unsafe { free(p) };
    }
}
```

`extern "nabi"`을 포함한 모든 `extern` 함수는 unsafe이므로 `unsafe { ... }` 안에서 호출합니다([4.15](../004-functions/415-unsafe.md)). toolchain 패키지는 자기가 선언한 런타임 함수를 안전한 함수로 감싸서 내놓습니다. 그 밖에는 `extern` 함수 호출도 외부 심볼이라는 점만 빼면 일반 호출과 같습니다. 결과 타입에는 Nabi의 일반 의미론이 그대로 적용됩니다. `extern static`은 일반 변수처럼 읽고, `mut`이면 씁니다. Nabi는 C 전역 변수를 검사할 수 없으므로 읽거나 쓰려면 `unsafe`(`unsafe` 블록이나 `unsafe fn` 본문)가 필요합니다. 밖에서 하면 에러이고, 도움말이 고친 형태를 보여 줍니다. `unsafe { optind = 6; }`처럼 씁니다. `addr_of`로 주소를 얻는 것은 아무것도 읽지 않으므로 `unsafe`가 필요 없습니다. 그래도 타입은 FFI 호환이어야 합니다(5.9.4). Nabi는 외부 메모리를 그 타입으로 읽으므로 `static environ: List<str>;`는 정의되지 않은 동작입니다. 포인터로 선언합니다(`static environ: Ptr<Ptr<c_char>>;`).

## 5.9.7. 가시성

`extern` 블록의 각 항목에도 가시성 키워드를 붙일 수 있습니다.

```nabi
extern "C" {
    pub fn public_api(x: i32) -> i32;       // 외부 공개
        fn _internal(x: i32) -> i32;        // 모듈 비공개
    pub static VERSION: i32;
}
```

`extern "C"` 블록 자체에는 가시성 키워드를 붙이지 않습니다. 가시성은 항목마다 붙입니다.

## 5.9.8. `extern` 블록 자체의 위치

`extern "C" { ... }` 블록은 모듈 수준에만 올 수 있습니다. 함수 안이나 `impl` 안에는 둘 수 없습니다.

```nabi
fn f() {
    extern "C" { ... }                     // ERROR: 함수 안에서는 안 됨
}
```

## 5.9.9. `#[link("libname")]`: 라이브러리 명시

외부 라이브러리의 이름은 `#[link]` 속성으로 명시합니다.

```nabi
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
    fn sqrt(x: f64) -> f64;
}
```

extern 블록의 `#[link("x")]`나 `#[link(name = "x", kind = "static" | "dylib")]`는 링커에 `-lx`를 넘깁니다. 라이브러리 검색 경로는 `-L <dir>` 디렉터리, 그다음 `LIBRARY_PATH`입니다. Apple 타깃에서 `kind = "static"`은 거기서 찾은 `lib<name>.a`를 경로로 링크하고, `#[link(name = "Foundation", kind = "framework")]`는 `-F <dir>` 디렉터리, 그다음 SDK의 `System/Library/Frameworks`에서 찾은 프레임워크 `Foundation.framework`를 링크합니다. 다른 타깃에서 프레임워크는 링크 에러입니다. nabic의 `-l`과 네이티브 패키지의 `links`, `nabi.nabion`의 `Dep::system` / `Dep::framework` 라이브러리도 같은 방식으로 라이브러리를 추가합니다([9.10.4b](../009-tools/910-nabi-frontend.md#9104b-네이티브-패키지)). wasm32에서 `#[link(wasm_import_module = "m")]`는 블록의 함수를 core module `m`에서 가져옵니다. 함수의 `#[link_name = "sym"]`은 링크할 심볼이나 import 이름을 정합니다([6.2.3](../006-meta-system/602-built-in-attributes.md#623-link-외부-라이브러리-명시)). `#[link]`가 없으면 링커가 시스템 기본값(`libc` 등)이나 다른 메커니즘으로 심볼을 해소합니다. 패키지 자신의 C 소스로 만든 오브젝트도 그중 하나입니다. 내장 C 컴파일러가 이를 컴파일해 패키지와 함께 링크하고([9.13.8](../009-tools/913-c-compiler.md#9138-패키지의-c-소스)), 그 함수는 다른 함수처럼 `extern "C"` 블록에 선언합니다. `#[link]`의 자세한 동작은 [내장 속성](../006-meta-system/602-built-in-attributes.md)에서 정의합니다.

## 5.9.10. `extern fn` 정의(외부에 노출)

앞 절에서 다룬 형태는 Nabi에서 외부 함수를 호출합니다. 반대로 Nabi 함수를 외부 ABI로 노출하는 형태도 있습니다.

```nabi
#[export]
pub extern "C" fn nabi_compute(x: c_int, y: c_int) -> c_int {
    x + y
}

#[export_name("nabi_area")]
pub extern "C" fn area(w: c_int, h: c_int) -> c_int { w * h }

extern "C" fn callback(arg: i32) -> i32 {
    arg * 2
}
```

일반 함수 선언에 `extern "ABI"` 수식어를 붙이면 그 함수를 해당 ABI와 호환되게 컴파일합니다. `extern "C" fn` 정의도 심볼 이름은 맹글링된 이름([9.5](../009-tools/905-symbol-mangling.md))을 그대로 씁니다. C가 링크할 이름이 필요하면 함수 자신의 이름으로 내보내는 `#[export]`(`nabi_compute`)나, `sym`으로 내보내는 `#[export_name("sym")]`(`area`를 `nabi_area`로)을 붙입니다. 내보낸 함수는 쓰이지 않는 함수 제거의 루트이고, 공유 라이브러리(`--crate-type=cdylib`, [9.3](../009-tools/903-package-archive.md))가 밖에 보이는 것도 이 함수들입니다.

`extern "ABI" fn` 정의는 `"C"`나 `"system"`을 받습니다. 그 밖의 ABI 문자열은 에러입니다. 32비트 x86 Windows에서 `extern "system"` 선언과 정의는 stdcall입니다. 피호출자가 인자를 스택에서 꺼내고, 심볼은 MSVC처럼 `_name@N`입니다.

`extern "C"`와 `extern "system"` fn 정의에는 다음 제약이 있습니다.

- 제네릭, `async`, `throws`일 수 없습니다.
- 매개변수와 반환 타입은 5.9.4의 FFI 호환 타입이어야 합니다. `str` 같은 비 FFI 타입은 쓸 수 없습니다. 5.9.4의 `str` 변환은 C 함수를 '호출'하는 선언에만 적용됩니다.

`#[export]`와 `#[export_name]`은 `extern "C"`와 `extern "system"` fn 정의에만 붙습니다. C는 내보낸 함수를 C 규약으로 부르는데, Nabi ABI 함수는 aggregate, `str`, 좁은 정수를 잘못 받습니다. 그래서 일반 함수에 붙이면 둘 다 에러입니다: `` `compute` is exported to C but uses the Nabi ABI: write `pub extern "C" fn compute(..)` ``. wasm32에서는 그 이름의 wasm export가 됩니다.

```nabi
#[export_name("on_event")]
extern "C" fn on_event(code: i32) -> i32 { code }     // C 에서 on_event 로 보임
extern "C" fn bad(s: str) { }                          // ERROR: str 은 쓸 수 없음
#[export]
pub fn compute(x: i64) -> i64 { x * 2 }               // ERROR: 내보냈지만 Nabi ABI
```

### `#[on_panic]`: abort 대신 C에 줄 값

```nabi
#[on_panic(return = 0)]
extern "C" fn cmp(a: Ptr<i32>, b: Ptr<i32>) -> c_int { .. }

#[on_panic(return = -1.5)]
extern "C" fn half(x: f64) -> f64 { .. }

#[on_panic]
extern "C" fn notify(x: i32) { .. }
```

C가 부른 함수를 C의 프레임 너머로 떠나려는 panic은 프로세스를 abort합니다(5.9.12). `extern "C"`나 `extern "system"` fn 정의에 `#[on_panic(return = v)]`를 붙이면 대신 복구합니다. panic은 평소처럼 보고되고 panic hook이 돈 뒤([stdlib 3.12](../../refs-standard-libraries/300-std/312-panic.md)), 함수가 호출자(C든 Nabi든)에게 `v`를 돌려줍니다. `v`는 결과 타입의 리터럴입니다. 정수, 부동소수, `true` / `false`, 포인터 결과면 `0`(null 포인터)입니다. 결과가 없는 함수는 `#[on_panic]`만 씁니다. 복구하는 함수는 panic 아래에서 가장 가까운 C 진입점입니다. 더 아래에서 C가 Nabi를 다시 부르면 그 안쪽 진입점이 정합니다.

panic과 함수 사이의 프레임은 건너뜁니다. 그 프레임의 `Drop`은 돌지 않으므로 풀었어야 할 것이 샙니다. 건너뛴 프레임이 잠근 `Mutex`는 잠긴 채로 남아, 다음 `lock`이 영원히 기다립니다. lock을 쥔 동안 실패할 수 있는 함수는 panic하지 말고 에러 값을 돌려줍니다. 복구는 `setjmp` / `longjmp`입니다([8.5.12](../008-abi/805-extern-symbols.md#8512-보조-심볼부록-비규범)). wasm32와 OS 없는 타깃에는 이것이 없어서, 거기서는 속성이 없을 때처럼 panic이 abort합니다.

잘못 쓰면 모두 쓸 것을 알려 주는 에러입니다.

| 경우 | 메시지 |
|------|--------|
| 숫자, `bool`, `0`이 아닌 값 | `` `#[on_panic]` on `count`: only a number, a `bool` or `0` (a null pointer) can be returned; write `#[on_panic(return = -1)]` with a value of the result type `` |
| 결과 없는 함수에 값 | `` `#[on_panic]` on `nothing`: it returns nothing, so write `#[on_panic]` alone `` |
| 결과 있는 함수에 값 없음 | `` `#[on_panic]` on `half`: a function with a result needs the value it returns `` |
| `extern "C"`가 아닌 함수 | `` `#[on_panic]` is for functions C calls: write `#[on_panic(return = -1)] extern "C" fn plain(..)` `` |

외부 함수 호출과 외부 노출, 두 메커니즘이 함께 FFI의 양방향을 이룹니다.

## 5.9.11. `extern fn`의 호출 규칙

`extern "C"` 함수는 C 호출 규약(calling convention)을 따릅니다. 그래서 [실행 엔진과 사용자 표면](../000-meta/001-execution-modes.md)에서 다루는 일반 함수 호출 규칙과 일부 다를 수 있습니다. 호출 규약의 차이는 런타임 ABI 사양(7장 / 8장)에서 다룹니다.

## 5.9.12. FFI의 안전성 책임

`extern` 블록의 모든 항목은 사용자가 책임집니다. [표준 라이브러리 계층](../000-meta/005-standard-libraries-tiers.md)의 `unsafe` 정신과 같습니다. 컴파일러가 검증하는 것은 다음과 같습니다.

- 시그니처 형식.
- 모든 타입이 FFI 타입인지(5.9.4, 별칭, 제네릭, 필드를 꿰뚫어 봄), 그리고 타깃의 C가 그 타입을 넘기는지.

컴파일러가 검증하지 않는 것은 다음과 같습니다.

- 외부 함수의 실제 동작(null 반환, 에러 반환, 메모리 누수 등).
- 외부 `static`의 동시성 안전.
- 외부 라이브러리의 실제 ABI 호환.

이 부분은 사용자가 직접 보장해야 합니다.

### C 프레임 아래의 panic

panic은 C가 부른 함수를 C의 프레임 너머로 떠나면 안 됩니다. C는 그것을 전혀 모르고, `exit`는 망가진 상태 위에서 호스트의 핸들러(`atexit`, 인터프리터의 종료 처리)를 돌립니다. 그래서 외부 코드가 부른 `extern "C"`나 `extern "system"` fn, 또는 그 아래의 Nabi 코드에서 난 panic은 보고를 출력하고([4.14.13](../004-functions/414-throws.md#41413-panic과-fatal-에러)), 다음 줄을 쓴 뒤

```text
panic in a function C called: aborting (`#[on_panic(return = v)]` on the `extern "C" fn` returns `v` to C instead)
```

프로세스를 abort합니다(`SIGABRT`, Unix에서 종료 상태 134). 아래에 외부 프레임이 없는 panic은 다른 곳에서처럼 종료 코드 101로 프로세스를 끝냅니다. 공유 라이브러리(`--crate-type=cdylib`)도 같은 규칙을 따르므로, Python, R, Julia 같은 호스트는 자기 프로세스가 `exit`하는 일을 겪지 않습니다. `#[on_panic]`(5.9.10)은 panic을 반환값으로 바꾸고, `std::panic::set_hook`([stdlib 3.12](../../refs-standard-libraries/300-std/312-panic.md))은 abort, exit, 복구 전에 모든 panic을 봅니다.
