# 1.6. `Pinned<T>`: GC pinning

`bare::gc` 모듈은 GC 힙의 객체를 잠시 움직이지 못하게 고정하는 메커니즘을
정의합니다. 원시 `Ptr<T>`가 GC 객체를 가리키는 동안 객체가 이동하면 포인터가
무효가 됩니다. 이 장은 그 충돌을 푸는 방법을 다룹니다.

Nabi unsafe tier에서 가장 중요한 장입니다. 포인터와 GC의 충돌을 어떻게 조정하는지
정면으로 설명합니다.

## 1.6.1. 동기: pinning이 필요한 이유

[refs-lang 7.3 GC 알고리듬](../../refs-lang/007-runtime/703-gc-algorithm.md)의 Immix mark-region GC는 **evacuation**(객체를
복사해 옮기기)을 합니다. 따라서 객체의 주소는 GC 주기 사이에 얼마든지 바뀔
수 있습니다.

```
  GC cycle 시작:    obj at 0x1000
  GC evacuation:    obj at 0x2000  (복사 후 forward)
  GC cycle 끝:      obj at 0x2000
```

관리 코드(`GcPtr<T>`)에서는 GC가 루트를 추적하면서 모든 포인터를 자동으로
갱신합니다. 그러나 GC는 원시 `Ptr<T>`를 알지 못하므로 갱신하지 않습니다.
그 포인터는 **낡은 포인터**(stale pointer)가 됩니다.

```nabi
use bare::gc;
use bare::ptr::Ptr;

let xs: List<i32> = [1, 2, 3];                 // GcPtr (자동 갱신)
let raw: Ptr<i32> = Ptr::dangling();
using pin = gc::pin(xs) {
    raw = pin.as_data_ptr();                    // raw ptr, pin 안에서만 유효
}                                               // 여기서 unpin
gc::collect();                                  // xs 가 이동할 수 있음
unsafe { raw.read() };                          // UB: raw 가 stale
```

해법은 **pinning**입니다. GC에 "이 객체를 옮기지 말라"고 명시적으로 알립니다.

## 1.6.2. `Pinned<T>` guard

```nabi
struct Pinned<T> {
    obj: T;
    // 내부: GC 의 pin set 등록
}

#[sealed]
trait Pinnable {
    // 값의 data 를 담은 GC 객체; 움직일 것이 없으면 none
    #[internal("write `pin(x)` to keep a value in place")]
    fn gc_object(self) -> GcBuf<u8> | none;
}
impl<T> Pinnable for List<T> { /* list 의 buffer; 비어 있으면 none */ }
impl<T> Pinnable for Slice<T> { /* 보는 buffer; 비어 있으면 none */ }
impl<T, const N: usize> Pinnable for Array<T, N> { /* 원소 buffer; 비어 있으면 none (2.9.14) */ }
impl Pinnable for str { /* none (2.15.2) */ }
impl Pinnable for StrBuf { /* byte buffer; 공간이 없으면 none (2.15.2) */ }
impl<T> Pinnable for GcPtr<T> { /* 그 cell */ }

fn pin<T: Pinnable>(obj: T) -> Pinned<T>;
fn collect();                           // 지금 전체 collection; pin 되지 않은 객체는 모두 이동 가능
fn stats() -> GcStats;                  // 수집기 카운터 (1.6.10)
fn collect_young();                     // 지금 minor 수집 (1.6.11)
fn collect_young_within(ms: f64) -> bool;
fn set_pause_goal(ms: f64);             // 1.6.12
fn no_collect<R>(f: fn() -> R) -> R;    // 1.6.13
fn set_heap_limit(bytes: u64);          // 1.6.14
struct Arena<T>;                        // 1.6.15

impl<T> Pinned<T> {
    fn obj(self) -> T;
}

impl<T> Drop for Pinned<T> {
    fn drop(ref self) {
        // GC pin set 에서 제거
    }
}
```

guard입니다. `pin()`이 반환한 `Pinned<T>`가 살아 있는 동안 GC는 그 객체를 옮기지 않습니다.
`Pinned<T>`는 한 번만 해제되는 `Drop` 값입니다([2.13.4](../200-core/213-marker.md#2134-drop-해제-시-정리)). guard의 소유자가 스코프를 벗어나는 곳에서,
또는 guard를 묶은 `using` 블록이 끝날 때 `Pinned::drop`이 호출되어 고정이 풀립니다. pin은 횟수를 셉니다. 객체는 고정한 횟수만큼 풀어 준 뒤에야 다시 움직입니다. `pin`은 `gc_object`가 가리키는 객체를 고정합니다. `none`이 돌아오면 아무것도 고정하지 않습니다. `GcBuf<u8>`은 [1.3.9](103-mem.md#139-gcbuft와-buf_-기본-요소)에 나오는, 타입이 지워진 GC 버퍼입니다. `Pinnable`은 sealed이고 `gc_object`는 toolchain 내부 항목입니다([6.2.16](../../refs-lang/006-meta-system/602-built-in-attributes.md#6216-internal-toolchain-패키지끼리-공유)). 프로그램은 `pin(x)`로 고정하고 `gc_object`를 부르지 않습니다.

### 예: FFI 호출

```nabi
let buf: List<u8> = read_input();

using pin = bare::gc::pin(buf) {
    let raw: Ptr<u8> = pin.as_data_ptr();
    unsafe {
        ffi::process(raw, buf.len());          // FFI 가 buf 데이터 사용
    }
}     // ← scope 끝에서 자동 unpin
```

여기서 `using` 블록은 [refs-lang 3.17 using 표현식](../../refs-lang/003-expressions/317-using-expression.md)이 정의하는 RAII 형태를
따릅니다.

## 1.6.3. `GcPtr<T>`와 `Ptr<T>`: 타입 수준 분리

```nabi
struct GcPtr<T> { /* compiler-managed; root 자동 추적 */ }
struct Ptr<T>   { /* raw; root 추적 X */ }
```

| 측면       | `GcPtr<T>`                    | `Ptr<T>`                                         |
|------------|-------------------------------|--------------------------------------------------|
| GC 루트    | **자동 추적**                 | **추적하지 않음**                                |
| 이동 시    | **자동 갱신**                 | **낡음**                                         |
| 사용처     | 일반 코드 (`List`/`Dict`/...) | unsafe + FFI만                                  |
| 생성       | GC 할당 (`List::new()` 등)    | `gc::pin().as_data_ptr()` 또는 `alloc::alloc(layout)` |
| 유효 기간  | 객체에 도달할 수 있는 동안    | pin guard 스코프 안에서만                        |

타입 시스템이 두 종류를 나누므로 사용자 코드는 보통 `GcPtr`만 다룹니다.
이 포인터는 `List<T>` 등의 안에 보이지 않게 감싸여 있습니다.
`Ptr<T>`는 unsafe에 들어간 뒤에야 명시적으로 나타납니다.

### 연산

```nabi
impl<T> GcPtr<T> {
    fn new(value: T) -> GcPtr<T>;           // value 를 담은 새 GC cell
    fn get(self) -> T;
    fn set(self, value: T);                 // GcPtr 의 모든 복사본이 봄
    fn same(self, other: GcPtr<T>) -> bool; // 동일성: 같은 cell
}
impl<T> !Sync for GcPtr<T> {}
impl<T> Pinnable for GcPtr<T> {}
```

### 내부 구현: GcPtr의 형태

```nabi
struct GcPtr<T> {
    // 내부: 보통 단순 `*T`. GC 가 외부에서 root set 으로 추적
    // 지역 변수면 컴파일러가 emit 하는 shadow stack frame 슬롯에 둠
}
```

Nabi의 GC는 **스레드별 shadow stack**으로 루트 집합을 추적합니다. 이 shadow stack은 컴파일러가 직접 내보냅니다([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack)).

## 1.6.4. pin / unpin의 GC 측 메커니즘

```
GC pin set 구조.
  - 해시 테이블: Ptr<Object> → pin count
  - GC 가 evacuation 시 pin count > 0 인 객체 _건너뜀_

pin(obj):
  1. obj 의 헤더에서 pin bit 확인 (또는 pin set 에 등록)
  2. count++
  3. Pinned<T> 반환

drop(Pinned):
  1. count--
  2. count == 0 면 pin bit 해제
```

이 구조는 Go 1.21의 `runtime.Pinner`와 거의 같습니다. 자세한 내용은
[pkg.go.dev/runtime#Pinner](https://pkg.go.dev/runtime#Pinner)에 있습니다.

### 비용

비용은 다음과 같습니다.

- pin 한 번은 O(1) 해시 삽입입니다.
- unpin 한 번도 O(1) 해시 삭제입니다.
- GC 주기 안에 pin 집합 검사가 추가되지만, 비용은 작습니다.

대부분 자주 실행되는 경로(hot path)에 들어가지 않습니다. FFI 호출 자체가 그리 잦지 않기 때문입니다.

## 1.6.5. pin이 필요한 때

모든 `extern "C"` 호출은 호출하는 스레드를 park합니다([1.7.1](107-ffi.md#171-extern-c-블록)). C가 도는 동안 수집기는 그 스레드를 기다리지 않으므로, 아무리 짧은 C 호출 중에도 수집이 돌아 객체를 옮길 수 있습니다. C가 호출 중에 쓰는, GC 객체 안을 가리키는 포인터는 다음 경우에만 유효합니다.

- 객체가 호출 내내, 그리고 C가 포인터를 쥐고 있는 동안 pin되어 있을 때(`gc::pin`, 1.6.2)
- 포인터가 루트가 직접 가리키는 객체를 가리킬 때. 호출자의 프레임이 호출이 돌아올 때까지 루트로 잡는 `str` 인자([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음))와, [2.15.2](../200-core/215-string.md#2152-str-불변-utf-8-문자열)의 규칙 아래의 `str::as_ptr` / `CStr::as_ptr`입니다.

그 밖의 GC 객체 안 포인터, 곧 리스트의 버퍼(`as_data_ptr`), slice의 원소, `StrBuf`의 바이트, `GcPtr`의 값은 pin이 필요합니다. 어떤 호출이든 컴파일러가 알아서 pin을 빼는 일은 없습니다. pin은 프로그램이 씁니다. `#[leaf]`([1.7.1](107-ffi.md#171-extern-c-블록))는 호출을 싸게 할 뿐 이 규칙을 바꾸지 않습니다. C가 호출 사이에 들고 있는 값은 `Handle`([1.7.10](107-ffi.md#1710-handlet-c가-쥐는-nabi-값))에 담습니다. `Handle`은 release될 때까지 값을 pin합니다.

원시 포인터를 들지 않는 코드에는 pin이 필요 없습니다.

| 자리             | 이유                                    |
|------------------|-----------------------------------------|
| 일반 메서드 호출 | GcPtr 자동 갱신, 원시 포인터 안 만듦   |
| GcPtr끼리 연산  | 모두 GC가 추적                         |
| `extern "C"` 함수에 넘기는 `str` | 인자의 루트가 호출 동안 문자열을 제자리에 둠 |

### pin된 `List`나 `StrBuf`는 늘어날 수 없음

리스트나 `StrBuf`가 capacity를 넘어 늘어나면 버퍼가 바뀌는데, C가 옛 버퍼의 주소를 쥐고 있을 수 있습니다. 그래서 버퍼가 pin된 동안 늘어나려 하면(`reserve`, 그리고 그것을 거치는 `push`, `push_str`, `extend` 등) 대신 panic합니다.

```text
cannot grow a list while it is pinned (C may hold its address): call `xs.reserve(n)` before `gc::pin(xs)`
cannot grow a `StrBuf` while it is pinned (C may hold its address): call `b.reserve(n)` before `gc::pin(b)`
```

pin하기 전에 잡아 둔 공간은 pin이 이어지는 동안 채울 수 있습니다.

```nabi
use bare::gc::pin;

let ys: List<i64> = [1];
ys.reserve(7);                     // 8개 들어갈 공간
let py = pin(ys);
for i in 0..7 { ys.push(i); }      // 들어감: 버퍼가 그대로
ys.push(9);                        // panic: 버퍼가 늘어나야 함
```

## 1.6.6. 추출 메서드

```nabi
impl<T> Pinned<List<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // List 의 buffer 시작
    fn as_data_many_ptr(self) -> ManyPtr<T>; // 다수 ptr
}

impl<T> Pinned<Slice<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // slice 의 첫 원소; 비어 있으면 dangling
    fn as_data_many_ptr(self) -> ManyPtr<T>; // 다수 ptr
}

impl<T, const N: usize> Pinned<Array<T, N>> {
    fn as_data_ptr(self) -> Ptr<T>;          // 첫 원소; 간격은 원소 단위의 strides()
    fn as_data_many_ptr(self) -> ManyPtr<T>; // 다수 ptr
}

impl<T> Pinned<GcPtr<T>> {
    fn as_value_ptr(self) -> Ptr<T>;         // cell 의 값
}

impl Pinned<str> {
    fn as_bytes_ptr(self) -> ConstPtr<u8>;   // UTF-8 bytes
    fn as_c_ptr(self) -> ConstPtr<c_char>;   // null-terminated 보장 안 함. 별도 변환
}

impl Pinned<StrBuf> {
    fn as_bytes_ptr(self) -> Ptr<u8>;        // 첫 바이트 (NUL 보장 안 함); 공간이 없으면 dangling
}
```

이 컨테이너 메서드가 추출 API입니다. 결과는 `Pinned`를 들고 있는 동안 유효합니다. `str::as_ptr`와 `CStr::as_ptr`는 pin 없이 주소를 주고, 더 좁은 [2.15.2](../200-core/215-string.md#2152-str-불변-utf-8-문자열)의 규칙을 따릅니다. 제네릭 `as_ptr`은 없습니다. `as_value_ptr`는 `List`의 `as_data_ptr`와 구별하려고 붙인 이름입니다. 컨테이너마다 전용 추출 메서드가 있습니다.
그래서 버퍼 시작이나 null 종료 형태처럼 각 컨테이너의 레이아웃에 맞는 형태로
데이터를 꺼낼 수 있습니다.

## 1.6.7. GC 런타임 연계

각 항목에 대응하는 GC 런타임 호출은 다음과 같습니다.

| 항목             | 런타임 호출                    |
|------------------|--------------------------------|
| `Pinned<T>`      | `nabi_gc_pin` / `nabi_gc_unpin`  |
| `pin<T>()`       | (위와 동일)                    |
| `collect()`      | `nabi_gc_collect`               |
| `stats()`        | `nabi_gc_stats` (1.6.10)        |
| `collect_young()` / `collect_young_within(ms)` | `nabi_gc_collect_young` / `nabi_gc_collect_young_within` (1.6.11) |
| `set_pause_goal(ms)` | `nabi_gc_set_pause_goal` (1.6.12) |
| `no_collect(f)`  | `nabi_gc_no_collect` (1.6.13)   |
| `set_heap_limit(bytes)` | `nabi_gc_set_heap_limit` (1.6.14) |
| `Arena<T>`       | `nabi_gc_arena_epoch` (1.6.15)  |
| `GcPtr<T>`       | shadow stack 프레임 슬롯 (컴파일러가 내보냄) |
| `as_data_ptr` 등 | 컨테이너별 레이아웃에 따름     |
| `Weak<T>`        | `nabi_gc_weak_new` / `nabi_gc_weak_get` (1.6.9) |
| pin된 버퍼의 늘어남 검사 (1.6.5) | `nabi_gc_pinned` |

## 1.6.8. 다른 언어와 비교

| 측면             | Java JNI                       | .NET fixed             | Go 1.21 Pinner           | Nabi                       |
|------------------|--------------------------------|------------------------|--------------------------|---------------------------|
| pin API          | `Get*Critical`                 | `fixed (T* p = ...)`   | `runtime.Pinner`         | `bare::gc::pin(obj)`  |
| RAII guard       | `Release` 명시                 | block scope            | `defer pin.Unpin()`      | `using` block (자동 drop) |
| 타입 분리        | **없음** (`jobject` 모두 동일) | **없음**               | **없음** (모든 포인터 같음) | `GcPtr<T>` vs `Ptr<T>`    |
| pin 후 만료 검증 | **없음**                       | 컴파일러 (block scope) | **없음** (수동)          | 타입 시스템 + 스코프      |
| 다중 pin         | overlapping OK                 | **제한**               | overlapping OK           | overlapping OK (count)    |

Nabi가 다른 언어와 크게 다른 점은 두 가지입니다.

1. 타입 시스템이 강제합니다. `GcPtr<T>`와 `Ptr<T>`는 서로 다른
   타입이고, 컴파일러는 자동 변환을 거부합니다. 둘 사이는 pin guard로만 건널 수 있습니다.
2. `using` 블록으로 해제합니다. `using` 블록을 빠져나갈 때 `Pinned<T>`가 해제되므로 unpin을
   빠뜨릴 수 없습니다. Go의 수동 `defer` 방식보다 안전한
   설계입니다.

> 이 정책은 Nabi 포인터 모델의 뼈대입니다. unsafe와 GC, 두 영역이 안전하게
> 만나는 접점입니다.

## 1.6.9. `Weak<T>`: 약한 참조

```nabi
// bare::gc
struct Weak<T> { /* opaque */ }

impl<T> Weak<T> {
    fn new(target: GcPtr<T>) -> Weak<T>;
    fn get(self) -> GcPtr<T> | none;        // 대상이 해제되면 none
}
impl<T> !Sync for Weak<T> {}
impl<T> !Send for Weak<T> {}    // 넘김이 그것이 가리키는 것을 복사할 수 없음 (2.13.9)
```

약한 참조는 객체를 살려 두지 않습니다. 수집이 객체를 해제하면 그 객체를 가리키는 모든 `Weak`는 `none`을 읽습니다. 객체를 옮기는 수집은 `Weak`를 갱신합니다. 이 동작이 `WEAK_REF` 스캔 종류([refs-lang 8.2.8](../../refs-lang/008-abi/802-scan-table.md#828-weak_ref))의 라이브러리 인터페이스입니다.

런타임에서 `Weak`는 type id `3`([refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-런타임-예약-type-id))의 GC 셀이고, `WEAK_REF` 워드 하나를 담습니다. `nabi_gc_weak_new`로 만들고 `nabi_gc_weak_get`으로 읽습니다([refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-보조-심볼부록-비규범)).

## 1.6.10. `stats`: 수집기 카운터

```nabi
// bare::gc
struct GcStats {
    pub collections: u64;      // 모든 종류
    pub minor: u64;            // minor 수집
    pub full: u64;             // 처음부터 끝까지 world를 멈춘 full 수집
    pub concurrent: u64;       // 동시에 mark한 full 수집 (7.5)
    pub pause_total_ns: u64;   // 프로그램이 멈춰 있던 시간, 모든 일시 정지의 합
    pub pause_max_ns: u64;     // 가장 긴 일시 정지
    pub live_bytes: u64;       // 마지막 수집 때 살아 있던 바이트와 그 뒤 할당한 바이트
    pub heap_bytes: u64;       // 운영체제에서 가져와 쥐고 있는 바이트 (block과 큰 객체)
    pub moved_bytes: u64;      // evacuation이 옮긴 바이트
    pub assists: u64;          // 할당하는 스레드가 동시 mark를 도운 횟수 (7.5.4)
    pub grown_bytes: u64;      // no_collect 구간이 수집 대신 늘린 힙 바이트 (1.6.13)
    pub limit_collections: u64; // 힙 상한이 앞당긴 수집 (1.6.14)
    pub young_budget: u64;     // 지금 minor 수집 사이에 할당할 수 있는 바이트
}

fn stats() -> GcStats;
```

`stats()`는 프로그램이 시작된 뒤의 카운터를 읽습니다. 지표를 내보내거나 프레임 예산을 확인할 때 씁니다. 수집기의 lock을 잠깐 잡으므로, 자주 도는 루프가 아니라 가끔(내보낼 때, 프레임마다 한 번) 읽습니다. 일시 정지는 멈춤 요청부터 세므로 safepoint까지 걸린 시간도 들어갑니다([refs-lang 9.9.8](../../refs-lang/009-tools/909-debug-tools.md#998-메모리--gc-도구)).

`collect()`는 full 수집 전체 동안 모든 스레드를 멈춥니다. 런타임이 스스로 시작하는 수집(minor 수집, 또는 동시에 mark하는 full 수집)보다 깁니다. 일시 정지를 짧게 두려면 수집은 런타임에 맡기거나, 프레임 사이에 `collect_young`을 부릅니다(1.6.11).

## 1.6.11. `collect_young`: 필요할 때 minor 수집

```nabi
// bare::gc
fn collect_young();
fn collect_young_within(ms: f64) -> bool;
```

`collect_young()`은 지금 minor 수집을 합니다. 지난 수집 뒤에 할당한 객체만 보므로 힙 크기가 아니라 그중 살아남은 것만큼 듭니다. 두 프레임 사이, 두 요청 사이가 수집하기 싼 때입니다. 동시 mark가 진행 중이면([refs-lang 7.5](../../refs-lang/007-runtime/705-concurrent-collection.md)) 아무것도 하지 않습니다. mark는 긴 일시 정지 없이 스스로 끝납니다.

`collect_young_within(ms)`는 최근 minor 수집으로 보아 `ms` 밀리초(프레임에 남은 시간) 안에 끝날 때만 수집하고, 했는지를 돌려줍니다. 추정은 최근 minor 수집의 young 바이트당 비용에 지금의 young 바이트를 곱하고 고정 부분을 더한 값입니다. 아직 minor 수집을 한 적이 없으면 수집합니다.

## 1.6.12. `set_pause_goal`: 일시 정지 목표

```nabi
// bare::gc
fn set_pause_goal(ms: f64);   // 0: 목표 없음 (기본값)
```

목표가 있으면 수집기는 일을 나누어 일시 정지를 `ms` 밀리초 가까이에 둡니다.

- young 예산이 목표를 따릅니다. 최근 minor 수집으로 보아 목표 안에 드는 양의 4분의 3이며, 1 MiB부터 기본 예산까지입니다([refs-lang 7.2.8](../../refs-lang/007-runtime/702-allocation.md#728-gc-트리거)). 지금 값은 `GcStats::young_budget`에 있습니다.
- full 수집은 힙 크기와 상관없이 모두 동시에 mark하고, 힙을 스스로 evacuation하지 않습니다. 힙을 압축하는 것은 `collect()`뿐입니다.
- 동시 mark 뒤의 sweep과 mark assist는 원래 일시 정지 밖에서 합니다([refs-lang 7.5.2](../../refs-lang/007-runtime/705-concurrent-collection.md#752-단계)).

약속이 아니라 목표입니다. 살아 있는 힙이 빠르게 자랄 때나, 할당이 많았던 동시 mark 바로 뒤처럼 일시 정지가 더 길어질 수 있습니다.

## 1.6.13. `no_collect`: 수집하지 않는 구간

```nabi
// bare::gc
fn no_collect<R>(f: fn() -> R) -> R;
```

`no_collect(f)`는 `f`를 부르고, 그것이 실행되는 동안 어느 스레드의 할당도 수집을 시작하지 않습니다. minor도, full도, stress 수집(`NABIR_GC_STRESS`)도 없습니다. 구간은 `f`가 어떻게 끝나든 끝나는 곳에서 닫힙니다. 구간은 겹칠 수 있고 여러 스레드에서 동시에 열려 있을 수 있으며, 마지막 구간이 닫히면 수집이 다시 시작됩니다.

- 구간 안에서 young 예산을 다 쓰면 대신 힙을 늘립니다. 한 번에 1 MiB 이상이며, `GcStats::grown_bytes`가 셉니다. 수집은 구간이 끝난 뒤, 수집이 필요한 다음 할당에서 합니다.
- 진행 중인 동시 mark는 구간 동안 mark를 계속하고, 마지막 pause는 구간 뒤에 합니다.
- 힙 상한(1.6.14)은 그대로 지킵니다. 상한을 넘길 할당은 구간 안에서도 수집합니다.
- 구간 안에서 부른 `collect()`와 `collect_young()`은 그대로 수집합니다.

## 1.6.14. `set_heap_limit`: 힙 상한

```nabi
// bare::gc
fn set_heap_limit(bytes: u64);   // 0: 상한 없음 (기본값)
```

사이클마다 예산을 `bytes`로 제한합니다. 힙이 상한에 가까워질수록 수집이 일찍 오고, 상한 때문에 앞당긴 수집은 full 수집입니다(`GcStats::limit_collections`가 셉니다). 상한은 프로세스 메모리 전체가 아니라 GC 객체의 바이트를, 나눠 준 young 바이트까지 셉니다. 살아 있는 데이터가 상한을 넘어도 오류가 아닙니다. 그때 제한은 살아 있는 바이트에 그 8분의 1(최소 4 MiB)을 더한 값이 되어, 힙은 상한을 조금씩 넘으며 매번 full 수집을 합니다. 상한을 넘길 동시 mark는 pause 안에서 끝냅니다.

## 1.6.15. `Arena<T>`: 한 프레임의 값

```nabi
// bare::gc
struct Arena<T> { /* 복사본끼리 공유 */ }
struct ArenaRef<T> { /* 평범한 데이터: 인덱스와 세대 */ }
impl<T> Copy for ArenaRef<T> {}

impl<T> Arena<T> {
    fn new() -> Arena<T>;
    fn with_capacity(n: usize) -> Arena<T>;
    fn alloc(self, value: T) -> ArenaRef<T>;   // 제자리에 덧붙임
    fn get(self, r: ArenaRef<T>) -> T;         // 지난 참조면 panic
    fn try_get(self, r: ArenaRef<T>) -> T | none;
    fn set(self, r: ArenaRef<T>, value: T);    // 지난 참조면 panic
    fn holds(self, r: ArenaRef<T>) -> bool;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;
    fn reset(self);                            // 한 번에 비움. 지금까지의 ArenaRef는 모두 지난 참조
    fn values(self) -> List<T>;                // 새 리스트, 할당한 순서
}
impl<T> !Sync for Arena<T> {}
```

`Arena<T>`는 한 프레임(또는 한 요청)의 값을 버퍼 하나에 담습니다. `alloc`은 값을 제자리에 덧붙이므로 수집기는 값마다 객체를 할당하거나 추적하지 않고, `reset`은 버퍼를 다음 프레임을 위해 남긴 채 arena를 한 번에 비웁니다. `Arena`의 복사본은 같은 arena를 공유합니다.

메모리 안전성은 두 규칙에 기댑니다.

- 버퍼는 평범한 GC 객체입니다. 값이 arena에 있는 동안 그 값이 가리키는 것은 살아 있고, arena 안의 어떤 것도 허공을 가리키지 않습니다. (`reset` 뒤에 GC 참조를 가진 타입의 값은 지우고, 그렇지 않은 타입의 값은 다음 `alloc`이 덮어씁니다.)
- `ArenaRef`는 인덱스와 세대로 된 평범한 데이터입니다. `reset`마다 프로그램의 어떤 arena도 쓴 적 없는 세대를 받으므로([refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-보조-심볼부록-비규범)), reset 전의 참조나 다른 arena의 참조는 `get`과 `set`의 검사에 걸려 panic합니다(`try_get`은 `none`, `holds`는 `false`). 어디에든, GC 객체 안에도 둘 수 있습니다. 늦게 쓰면 정해진 panic이 날 뿐, 다음 프레임의 값을 읽는 일은 없습니다.

그래서 arena에는 lifetime도, 브랜드를 단 scope도, `T`가 무엇을 가져도 되는지에 대한 규칙도 필요 없습니다. 대가는 접근마다 정수 비교 한 번입니다.

```nabi
use bare::gc;
use bare::gc::Arena;

struct Particle { x: f64; y: f64; }

let particles: Arena<Particle> = Arena::with_capacity(4096);
loop {
    let p = particles.alloc(Particle { x: 0.0, y: 0.0 });
    update(particles, p);
    draw(particles);
    particles.reset();                         // 이 프레임의 particle은 사라짐
    gc::collect_young_within(frame_left_ms());
}
```
