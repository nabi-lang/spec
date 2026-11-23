# 8.5. Extern "C" 심볼

`libnabir.a`가 노출하는 [심볼](../000-meta/007-glossary.md#extern-c-심볼)의 시그니처를 정의합니다. 컴파일러가 생성한 코드와 표준 라이브러리가 이 심볼을 호출합니다.

Nabi 쪽 선언은 런타임 ABI `extern "nabi"`([5.9.2](../005-modules/509-extern-block.md#592-abi-명시))입니다. toolchain 패키지가 이 선언을 비공개 항목으로 둡니다(대부분 `core::rt`). 아래 블록에서 `fn` 값은 `NabiFn`(8.5.0a)이고, 그 밖의 값은 C ABI와 같습니다. `str` 타입은 `struct { const uint8_t *ptr; size_t len; }`입니다. `str` 값을 받거나 돌려주는 런타임 함수는 `NabiFn`처럼 16바이트 struct의 플랫폼 C ABI 규칙을 따릅니다([8.6.3](806-calling-convention.md#863-struct-인자--반환-작은-struct-규칙)). ABI가 허용하면 레지스터로 값 전달하고, 레지스터와 스택에 나뉘지 않으며, Windows x64와 wasm32에서는 참조로 넘깁니다. 프로그램은 런타임 심볼을 직접 선언할 수 있습니다(`extern "nabi"` / `extern "C"`). 그 선언은 toolchain이 그 심볼을 선언한 것과 같아야 합니다. 다르면 에러입니다. 아래 목록은 C ABI 모양을 보여 줍니다. 목록이 `Ptr<u8>` / `usize`로 보여 주는 매개변수를 toolchain이 GC 타입(`GcBuf<T>`)으로 선언했다면(`nabi_gc_pin`, `nabi_gc_secret_new`, waker 인자 등) 그 GC 타입을 씁니다.

> [!IMPORTANT]
> 모든 심볼은 `nabi_` 접두사로 시작합니다. 외부 라이브러리나 사용자 코드와 심볼이 충돌하지 않게 하려는 것입니다.

## 8.5.0. 스레드 안전성 분류

각 심볼 정의에는 **Thread-safety** 표기가 붙습니다. 이 표기는 컴파일러가 내보낸 코드와 런타임 사이 계약의 일부입니다. 표기는 그 심볼을 어떤 호출 컨텍스트에서 안전하게 호출할 수 있는지 밝힙니다. 분류는 다음 6가지입니다.

| 분류                  | 의미                                                                         |
|-----------------------|------------------------------------------------------------------------------|
| `thread-safe`         | 여러 스레드가 동시에 호출해도 안전 (런타임이 내부에서 동기화)               |
| `caller-thread-only`  | 호출자 스레드의 컨텍스트에 묶임. 다른 스레드가 그 호출자를 대신할 수 없음   |
| `process-init`        | 프로세스 시작 시점, mt 진입 전 단 한 번                                      |
| `process-shutdown`    | 프로세스 종료 시점, mt 종료 후 단 한 번                                      |
| `per-thread-init`     | 각 워커 스레드가 시작할 때 그 스레드가 자기 자신에 대해 한 번               |
| `per-thread-shutdown` | 각 워커 스레드가 끝날 때 그 스레드가 자기 자신에 대해 한 번                 |

`caller-thread-only`는 함수의 효과가 호출자의 TLS 슬롯이나 스택 프레임에 묶여 있다는 뜻입니다. 그래서 다른 스레드가 그 호출자를 대신할 수 없습니다. 함수가 재진입할 수 없다는 뜻은 아닙니다. 여러 스레드가 각자 자기 자신을 위해 동시에 호출해도 됩니다.

## 8.5.0a. `NabiFn`: 함수 값의 C ABI 형태

Nabi 함수 값 `{code, env}`([4.4.8](../004-functions/404-function-type.md#함수-값의-표현))를 런타임 심볼에 넘길 때의 C 표현입니다.

```c
typedef struct {
    void *code;   // env 를 첫 인자로 받는 코드 포인터
    void *env;    // 캡처 환경 (이름 있는 함수는 NULL)
} NabiFn;
```

`NabiFn`은 C struct로 **값 전달**됩니다. 실제 레지스터 배치는 타깃 C ABI의 16바이트 struct 규칙을 따릅니다.

## 8.5.1. GC 영역

### `nabi_gc_alloc_slow`

[할당](../007-runtime/702-allocation.md)에서 다루는 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer)의 slow path입니다. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_gc_alloc_slow(size: usize, align: usize, type_id: u32) -> GcBuf<u8>;
}
```

인자와 반환값의 의미는 다음과 같습니다.

- `size`: 객체의 총 크기입니다. 16바이트 헤더에 페이로드를 더해 정렬한 값입니다.
- `align`: 객체의 정렬 요구입니다. 보통 8이며, `#[repr(align(N))]`이 붙으면 더 커집니다.
- `type_id`: 객체의 타입 ID입니다.
- 반환: 객체의 GC 참조(`GcBuf<u8>`, 런타임 ABI의 GC 포인터)를 돌려줍니다. 참조는 헤더 자리에서 시작합니다. type_id와 gc_flags는 함수가 채웁니다.

null을 반환하지 않습니다. 메모리가 부족하면 fatal(`nabi_oom`)입니다. `isize::MAX`를 넘는 요청은 fatal 에러 "allocation too large"(종료 코드 101)입니다.

### `nabi_alloc_raw` / `nabi_realloc_raw`

GC 밖의 raw 메모리를 할당합니다([stdlib 1.4 alloc](../../refs-standard-libraries/100-bare/104-alloc.md)). **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_alloc_raw(size: usize, align: usize, zeroed: i32) -> usize;
    fn nabi_realloc_raw(ptr: usize, old_size: usize, align: usize, new_size: usize) -> usize;
}
```

`zeroed`가 0이 아니면 할당한 메모리를 0으로 채웁니다. `nabi_alloc_raw`는 `size`가 0이거나, `align`이 2의 거듭제곱이 아니거나, 할당에 실패하면 null(`0`)을 반환합니다. 이때 abort로 끝내지 않습니다. `nabi_realloc_raw`도 실패하면 null을 반환합니다. 실패 처리는 호출자(`bare::alloc`)가 맡습니다.

### `nabi_gc_wb_slow`

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md)의 write barrier slow path입니다. **Thread-safety: `caller-thread-only`**(TLS의 wb log 버퍼에 기록).

```
extern "nabi" {
    fn nabi_gc_wb_slow(obj: Ptr<u8>, slot: ConstPtr<u8>, len: usize);
}
```

인자 `obj`는 변경된 슬롯이 속한 객체의 포인터, `slot`은 그 슬롯의 주소, `len`은 곧 저장할 값의 바이트 크기이고, 반환값은 없습니다. 이 함수는 obj를 remembered set에 추가하고, log가 차면 비웁니다(flush). 동시 mark 중이면 객체가 가진 참조도 기록합니다. 큰 객체는 `slot..slot + len`이 걸치는 card의 참조만 기록합니다([7.5.3](../007-runtime/705-concurrent-collection.md#753-satb-배리어)).

### `nabi_gc_register_type` / `nabi_gc_register_types`

타입의 [ScanTable](802-scan-table.md)을 등록합니다. **Thread-safety: `process-init`**.

```
extern "nabi" {
    fn nabi_gc_register_type(type_id: u32, scan_table: ConstPtr<u8>);
    fn nabi_gc_register_types(base: u32, tables: ConstPtr<ConstPtr<u8>>, n: usize);   // tables[i]를 id base + i로
}
```

프로세스가 시작할 때 mt에 들어가기 전에 호출하므로 동기화가 필요 없습니다. `nabi_gc_register_types`는 표 `n`개를 연속된 id로 한 번에 등록합니다([8.2.13](802-scan-table.md#8213-등록-시점)).

### `nabi_gc_pin` / `nabi_gc_unpin`

[FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 객체를 고정(pin)합니다([객체 레이아웃과 메모리 공간](../007-runtime/701-object-layout.md), [`extern` 블록 (FFI 선언)](../005-modules/509-extern-block.md) 참고). **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_gc_pin(obj: Ptr<u8>) -> Ptr<u8>;       // 고정된 객체 (같은 주소) 반환
    fn nabi_gc_unpin(obj: Ptr<u8>);
}
```

`extern "C"` 함수(`str` 인자)나 blocking 런타임 호출에 넘긴 GC 포인터는 따로 고정하지 않아도 옮겨지지 않습니다. 호출이 끝날 때까지 인자가 루트에 있기 때문입니다([7.4.1](../007-runtime/704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음)). 인자가 루트로 잡지 않는 객체 안을 가리키는 포인터(리스트의 버퍼)는 호출 내내 고정해야 합니다. 어떤 C 호출 중에도 수집이 돌 수 있기 때문입니다. C 코드가 호출 뒤에도 보관하는 포인터도 마찬가지입니다([stdlib 1.6.5](../../refs-standard-libraries/100-bare/106-gc-pin.md#165-pin이-필요한-때)). pin은 객체에 `PINNED`를 켭니다. 그러면 객체는 evacuation 대상에서 빠지고 GC가 객체를 살려 둡니다. unpin은 고정을 풉니다.

### `nabi_gc_collect`

명시적인 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 트리거입니다. 드물게 씁니다. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_gc_collect();
}
```

full 수집을 하고, 수집이 끝나면 돌아옵니다.

### `nabi_gc_secret_new` / `nabi_gc_secret_wipe`

`core::crypto::Secret`([stdlib 2.21.5](../../refs-standard-libraries/200-core/221-crypto.md#2215-보안-고려-사항))의 비밀 저장소입니다. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_gc_secret_new(len: usize) -> GcBuf<u8>;
    fn nabi_gc_secret_wipe(obj: GcBuf<u8>);
}
```

`nabi_gc_secret_new`는 type id 1(참조 없음)인 LOS 객체를 할당하고 `WIPE`([8.1.3](801-object-header-abi.md#813-gc_flags오프셋-4-u32))를 켭니다. 0으로 채운 페이로드의 크기는 `len`을 8의 배수로 올린 바이트 수입니다. 할당하기 전에 수집할 수 있습니다. `isize::MAX`를 넘는 요청은 fatal 에러 "allocation too large"입니다. `nabi_gc_secret_wipe`는 페이로드 전체를 volatile store로 덮어쓰고 compiler fence를 둡니다. 할당도 block도 하지 않으므로 safepoint가 아닙니다([8.4](804-safepoint.md)). 수집기는 객체를 해제하기 전에 같은 방식으로 페이로드를 덮어씁니다.

## 8.5.2. 런타임 영역

### `nabi_runtime_init`

런타임을 초기화합니다. **Thread-safety: `process-init`**.

```
extern "nabi" {
    fn nabi_runtime_init();
}
```

프로세스가 시작한 직후, `main()` 전에 호출합니다. 이 함수는 호출한 스레드를 주 mutator로 준비합니다(TLS 블록, 스택 가드, GC 상태). executor의 워커 풀은 `core::task`가 처음 쓸 때 시작하는 Nabi 코드이고, I/O poller는 처음 등록할 때 만듭니다. 컴파일러가 생성한 C-ABI `main`은 이 호출 직후 `nabi.register_types`를 호출합니다([8.2.13](802-scan-table.md#8213-등록-시점)).

런타임은 이 밖에 스레드 로컬 블록 `nabi_tls`를 내보냅니다([8.3.8](803-tls-slots.md#838-컴파일러-출력-형식)).

### `nabi_runtime_shutdown`

런타임을 종료합니다. **Thread-safety: `process-shutdown`**.

```
extern "nabi" {
    fn nabi_runtime_shutdown();
}
```

`main()`이 끝난 직후 호출합니다. 다른 모든 Nabi 스레드는 지금 실행 중인 런타임 / C 코드를 마친 뒤 다음 safepoint에서 영구히 멈춥니다. 그다음 런타임은 남은 자원을 정리합니다.

### `nabi_exit`

프로세스를 즉시 종료합니다. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_exit(code: i32) -> !;
}
```

[`std::process::exit`](../../refs-standard-libraries/300-std/306-process.md)가 이 심볼을 호출합니다. `finally`와 `Drop`은 실행하지 않습니다([4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)).

### 문자열 헬퍼

`FromStr`과 `str::bytes`를 뒷받침하는 내부 보조 함수입니다([stdlib 2.15](../../refs-standard-libraries/200-core/215-string.md)). **Thread-safety: `thread-safe`**. `str` 인자는 머리말에서 정한 struct 형태입니다.

```
extern "nabi" {
    fn nabi_str_eq(a: str, b: str) -> i32;                            // 같으면 1, 아니면 0
    fn nabi_str_byte(s: str, i: usize) -> u8;                         // i 번째 바이트 (경계 검사는 호출자)
    fn nabi_str_slice(s: str, start: usize, end: usize, at: str) -> str;  // s[start:end]. 범위 / UTF-8 경계가 틀리면 at 위치로 panic
    fn nabi_str_parse_int(s: str, radix: u32, signed: i32, bits: u32, out: GcBuf<i64>) -> i32;
                                        // 0 ok, 1 빈 문자열, 2 잘못된 자릿수, 3 범위 밖; 값의 비트는 out[0]
    fn nabi_str_parse_float(s: str, out: GcBuf<f64>) -> i32;           // 0 ok, 2 잘못됨, 3 범위 초과 (유한한 숫자가 타입 범위를 넘음); 값은 out[0]
}
```

Nabi 쪽에서 `nabi_str_slice`는 앞의 세 매개변수만으로 `#[track_caller]`를 붙여 선언합니다([6.2.12](../006-meta-system/602-built-in-attributes.md#6212-track_caller-호출-위치-전달)). 넷째 C 인자 `at`, 곧 호출자의 위치는 컴파일러가 넘깁니다.

### `nabi_runtime_yield`

[safepoint 프로토콜](804-safepoint.md)의 `yield` 함수, 곧 safepoint 폴링의 slow path입니다. **Thread-safety: `caller-thread-only`**(호출자 mutator가 자기 yield_flag를 폴링한 결과로 호출).

```
extern "nabi" {
    fn nabi_runtime_yield();
}
```

### `nabi_enter_native` / `nabi_leave_native`

네이티브 bracket의 slow path입니다([8.4.10](804-safepoint.md#8410-네이티브-호출)). **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_enter_native();                // C 로 들어감: 이 스레드는 PARKED
    fn nabi_leave_native();                // C 에서 돌아옴: 진행 중인 수집을 기다림
}
```

컴파일러는 `#[leaf]` 호출([stdlib 1.7.1](../../refs-standard-libraries/100-bare/107-ffi.md#leaf-짧은-호출))을 뺀 모든 `extern "C"` 호출 앞뒤에 fast path를 인라인으로 내보냅니다. fast path는 mutator 상태에 `PARKED` / `RUNNING`을 seq_cst로 쓰고 yield 플래그를 확인합니다. 플래그가 켜져 있을 때만 이 함수를 호출합니다.

### `nabi_native_reenter` / `nabi_native_resume`

모든 `extern "C" fn` 정의의 입구와 출구입니다. 이 함수 덕분에 C 콜백이 mutator로 실행됩니다. **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_native_reenter() -> u32;       // 스레드가 PARKED (C 안) 였으면 native 를 떠남; 그랬는지 반환
    fn nabi_native_resume(parked: u32);    // reenter 가 떠났으면 다시 native 로
}
```

외부 코드가 부른 진입점은 돌아올 때까지 스레드의 외부 진입 횟수에 셉니다. 이 횟수가 0이 아닐 때 난 panic은 exit하지 않습니다. 보고와 panic hook 뒤에, 가장 안쪽 `#[on_panic]` 진입점이 가장 가까운 외부 진입점이면 그리로 돌아가고(`nabi_catch_push`), 아니면 `` panic in a function C called: aborting (`#[on_panic(return = v)]` on the `extern "C" fn` returns `v` to C instead) ``를 쓰고 프로세스를 abort합니다. 종료 상태는 134입니다([5.9.12](../005-modules/509-extern-block.md#5912-ffi의-안전성-책임)).

### `nabi_catch_push` / `nabi_catch_pop`

`#[on_panic]` 진입점의 복구 레코드입니다([5.9.10](../005-modules/509-extern-block.md#on_panic-abort-대신-c에-줄-값)). **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_catch_push(rec: Ptr<u8>);      // rec 를 push; 컴파일된 코드가 이어서 _setjmp(rec + 64) 호출
    fn nabi_catch_pop(rec: Ptr<u8>);       // 진입점이 정상으로 돌아감: 레코드를 버림
}
```

진입점은 레코드를 자기 프레임에 둡니다. `64 + 768`바이트, 16바이트 정렬이고, 런타임 몫이 앞에, `jmp_buf`가 오프셋 64에 있습니다. 진입점은 `nabi_catch_push`를 부르고 이어서 `jmp_buf`에 `_setjmp`를 부릅니다. 복구는 레코드가 저장한 shadow stack과 스레드 상태를 되돌리고 그리로 `longjmp`하며, 진입점은 `#[on_panic]` 값을 돌려줍니다. 정상 경로에서는 진입점이 `nabi_catch_pop`을 부릅니다. wasm32와 OS 없는 타깃에는 레코드가 없습니다.

### `nabi_errno_save`

`#[errno]` 호출의 errno입니다([stdlib 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errno-errno와-getlasterror)). **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_errno_save();                  // errno 와 (Windows 에서) GetLastError 를 이 스레드의 슬롯에
}
```

컴파일된 코드는 C 호출이 돌아온 직후, 네이티브 상태를 떠나거나 인자를 unpin하기 전에 이것을 부릅니다. 둘 다 errno를 바꾸는 코드를 돌 수 있기 때문입니다. 할당도 블록도 하지 않습니다.

### `nabi_nosp_enter` / `nabi_nosp_leave`

`#[no_safepoint]` 런타임 함수 호출을 감싸는 검사용 bracket입니다([8.4.10](804-safepoint.md#8410-네이티브-호출)). debug 빌드의 nabic만 내보냅니다. **Thread-safety: `caller-thread-only`**.

```
extern "nabi" {
    fn nabi_nosp_enter();                  // 이 스레드의 no_safepoint 깊이를 올림 (8.3.1)
    fn nabi_nosp_leave();                  // 내림
}
```

깊이가 0이 아닌 동안 스레드가 수집 때문에 멈추려 하면 debug 런타임은 fatal 에러로 프로세스를 끝냅니다. release 런타임에서는 두 함수 모두 아무것도 하지 않습니다.

### `nabi_thread_init` / `nabi_thread_shutdown`

워커 스레드의 수명 주기를 맡습니다. **Thread-safety: `nabi_thread_init`은 `per-thread-init`, `nabi_thread_shutdown`은 `per-thread-shutdown`**.

```
extern "nabi" {
    fn nabi_thread_init();
    fn nabi_thread_shutdown();
}
```

각 워커 스레드가 시작하고 끝날 때 호출합니다. [TLS 슬롯](803-tls-slots.md)의 TLS를 초기화하거나 정리합니다.

### 내부 심볼 계열

다음 묶음은 참조 런타임의 내부 보조 함수이며 ABI 계약에 속하지 않습니다. `nabi_fmt_*`, `nabi_str_*` 보조 함수, `nabi_char_*`, `nabi_nat_*`, `nabi_stdio_*`, `nabi_time_*`, `nabi_gc_stats_*`, `nabi_buf_*`, `nabi_cstr_len`.

### safepoint가 아닌 함수

Nabi 쪽에서는 `#[no_safepoint]`가 붙은 `extern "nabi"` 블록에 선언합니다([5.9.2](../005-modules/509-extern-block.md#592-abi-명시)).

다음 런타임 함수는 할당도 block도 하지 않음이 보장됩니다. 그래서 이 함수를 호출하는 곳은 safepoint가 아닙니다([8.4.2](804-safepoint.md#842-삽입-위치), [8.9.12](809-lowering-control.md#8912-shadow-stack-프레임)). `nabi_str_len`, `nabi_str_byte`, `nabi_str_cmp`, `nabi_str_eq`, `nabi_str_eq_at`, `nabi_str_find_near`, `nabi_str_rfind_near`, `nabi_str_char_count`, `nabi_str_bytes_buf`, `nabi_buf_copy`, `nabi_char_*` 속성 / 대소문자 함수, `nabi_fmt_len`, `nabi_f64_*`, `nabi_f32_*`.

이 함수를 호출하면 돌아올 때까지 다른 모든 스레드의 수집이 기다립니다. 그래서 이 함수가 하는 일은 짧게 둡니다. 입력이 길면(64 KiB 이상) 표준 라이브러리는 `#[no_safepoint]` 블록 밖에 선언한 parked 변형을 대신 부릅니다([8.4.10](804-safepoint.md#8410-네이티브-호출)). `nabi_str_find_parked` / `nabi_str_rfind_parked`(검색의 처음 64 KiB는 `nabi_str_find_near` / `nabi_str_rfind_near`가 그 자리에서 보고, 나머지를 맡음), `nabi_str_char_count_parked`, `nabi_str_grapheme_count_parked`, `nabi_buf_utf8_error_parked`, `nabi_buf_copy_parked`, `nabi_buf_put_str_parked`입니다.

## 8.5.3. 코루틴 영역

### `nabi_coro_frame_alloc`

[코루틴](../007-runtime/708-coroutine.md) 프레임 크기의 GC 객체를 할당합니다. **Thread-safety: `thread-safe`**. 컴파일러는 이 심볼을 내보내지 않습니다. 코루틴 프레임은 일반 힙 할당입니다([8.10](810-lowering-coroutine-generic.md)). 런타임은 C와 저수준 코드를 위해 이 심볼을 제공합니다.

```
extern "nabi" {
    fn nabi_coro_frame_alloc(size: usize, type_id: u32) -> GcBuf<u8>;
}
```

`nabi_gc_alloc_slow`와 거의 같지만, 프레임 전용 처리를 따로 할 수 있습니다.

### `nabi_coro_park` / `nabi_coro_unpark`

컴파일러는 이 두 심볼을 내보내지 않습니다(`await`는 `core::task`의 parking registry를 씁니다). 런타임은 C나 저수준 코드가 쓰도록 이 심볼을 둡니다. `frame`(아무 GC 객체)의 주소가 parking key입니다. `nabi_coro_unpark(frame)`은 그 key에 park한 waker를 모두 깨웁니다. task executor 밖에서 호출하면 abort로 끝납니다.

**Thread-safety: `nabi_coro_park`는 `caller-thread-only`**(호출한 코루틴 자신을 park), **`nabi_coro_unpark`는 `thread-safe`**(다른 스레드에서 호출 가능).

```
extern "nabi" {
    fn nabi_coro_park(frame: Ptr<u8>, waker: Ptr<u8>);
    fn nabi_coro_unpark(frame: Ptr<u8>);
}
```

`nabi_coro_park`는 코루틴을 일시 중단하고 waker를 등록합니다. `nabi_coro_unpark`는 코루틴을 task 큐에 넣어 재개합니다.

여기서 `waker`는 Nabi 쪽에서 런타임의 `Waker`([2.20.2](../../refs-standard-libraries/200-core/220-future.md))를 담은 원소 하나짜리 `GcBuf<Waker>`이며, 8.5.5와 같습니다. C ABI에서는 그 버퍼를 가리키는 포인터이고, 목록에는 [7.8.8](../007-runtime/708-coroutine.md)을 비롯해 `Ptr<u8>`로 적습니다(`nabi_io_register`, `nabi_timer_register`도 같음). `Waker`의 레이아웃은 코루틴 구현이 정합니다.

## 8.5.4. 스케줄러 영역

task 런타임(spawn, join, cancel, executor)은 `core::task`의 Nabi 코드입니다([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)). 이를 위해 런타임은 아래의 executor 슬롯, 스레드, parking 심볼을 제공합니다. 그 밖의 스케줄러 보조 심볼(`nabi_sched_*`, `nabi_lane_*`, `nabi_sync_*`, `nabi_deque_*`)은 구현이 정합니다(8.5.12).

### `nabi_task_worker_*` / `nabi_time_now_ns`

현재 스레드의 executor worker([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md))를 TLS 슬롯 `nabi_tls_task_worker`([8.3.1](803-tls-slots.md#831-tls-슬롯-구조))에 넣고 꺼내는 함수, 그리고 시계입니다. **Thread-safety: `caller-thread-only`**(`nabi_time_now_ns`는 `thread-safe`).

```
extern "nabi" {
    fn nabi_task_worker_present() -> i32;       // 이 스레드가 worker 이면 1
    fn nabi_task_worker_get() -> usize;         // worker 객체 주소, 없으면 0
    fn nabi_task_worker_set(w: usize);          // 살아 있는 GC 객체 또는 0. 설치하는 동안 pin
    fn nabi_task_worker_clear();                // = nabi_task_worker_set(0)
    fn nabi_time_now_ns() -> i64;               // monotonic clock (ns)
}
```

### `nabi_thread_spawn`

1:1 OS 스레드를 만듭니다. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_thread_spawn(f: NabiFn, stack_size: usize, id: u64) -> usize;
    fn nabi_thread_new_id() -> u64;
    fn nabi_thread_id() -> u64;                        // 현재 스레드의 worker id
    fn nabi_thread_sleep_ns(ns: i64);
    fn nabi_thread_yield_now();
}
```

`nabi_thread_spawn`은 Nabi 클로저 `f`(아래 `NabiFn`, 값 전달)를 OS 스레드에서 실행합니다(`stack_size` 0은 기본값). `id`는 `nabi_thread_new_id`로 받은 값이며, 새 스레드의 `nabi_tls_worker_id`가 됩니다. 반환값은 join 핸들이고, 생성에 실패하면 0입니다. 스레드가 끝날 때까지 `env`는 GC 루트입니다.

### `nabi_task_hooks_install` / `nabi_task_key`

`core::task`의 executor가 런타임에 훅을 설치합니다. 런타임의 I/O 드라이버, 타이머, `nabi_coro_park` / `nabi_coro_unpark`는 이 훅으로 task를 깨우고 멈춥니다. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_task_hooks_install(
        io: extern "C" fn(fd: i32, interest: u32, waker: Ptr<u8>),
        timer: extern "C" fn(deadline_ns: u64, waker: Ptr<u8>),
        park: extern "C" fn(key: Ptr<u8>, waker: Ptr<u8>),
        unpark: extern "C" fn(key: Ptr<u8>),
        restart: extern "C" fn(),               // 실패한 REPL 셀과 함께 사라진 executor 작업을 재시작
    );                                          // 처음 설치한 것이 이김
    unsafe fn nabi_task_key(obj: Ptr<u8>) -> usize;   // GC 객체 주소를 park 키로
}
```

`nabi_task_key`는 주소를 키로 쓰므로 그 객체에 `NOMOVE`([8.1.3](801-object-header-abi.md#813-gc_flags오프셋-4-u32))를 켭니다. 그 객체는 그 뒤로 옮겨지지 않습니다. 호출 안에서만 쓰는 동일성 비교에는 `buf_same`([stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md))을 씁니다. 이 함수는 아무것도 고정하지 않습니다.

### `nabi_sync_wait` / `nabi_sync_wake`

동기화 기본 요소(`Mutex`, `Condvar`, `Once`)의 대기와 깨우기입니다. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_sync_wait(buf: Ptr<u8>, index: usize, expected: i32, timeout_ns: i64);   // GcBuf<i32> 원소가 expected 인 동안 대기. timeout_ns < 0 이면 무한
    fn nabi_sync_wake(buf: Ptr<u8>, index: usize);                                    // 그 원소에서 대기 중인 스레드를 모두 깨움
}
```

`nabi_sync_wait`는 깨울 이유가 없어도 돌아올 수 있습니다. 그래서 호출자가 조건을 다시 확인합니다. GC는 대기 중인 스레드를 정지한 것으로 봅니다.

### `nabi_thread_join`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_thread_join(handle: usize) -> i32;          // 0 이면 성공
}
```

### `nabi_thread_detach` / `nabi_cpu_count`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_thread_detach(handle: usize) -> i32;   // 끝나면 OS 자원 반환, 더 이상 join 불가. 0 이면 성공
    fn nabi_cpu_count() -> usize;                  // 온라인 CPU 수 (최소 1), par 풀의 크기
}
```

### `nabi_static_init_begin` / `nabi_static_init_end`

초기값이 `const`가 아닌 `static`의 지연 초기화([5.3](../005-modules/503-static.md))를 뒷받침합니다. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_static_init_begin(state: Ptr<u8>, at: ConstPtr<u8>, at_len: usize) -> i32;   // 1: 호출자가 초기화식 실행, 0: 이미 준비됨
    fn nabi_static_init_end(state: Ptr<u8>);            // 초기화 완료 공개
}
```

`state`는 그 `static`의 상태 바이트(0 새로, 1 진행 중, 2 준비됨)입니다. 다른 스레드가 초기화 중이면 `nabi_static_init_begin`은 초기화가 끝날 때까지 기다립니다. 그동안 GC는 이 스레드를 정지한 것으로 봅니다. `at` / `at_len`은 그 static을 읽는 코드의 `path:line:col`입니다. 자기 static을 읽는 초기화식은 그 자리에서 panic이 일어납니다([4.14.13](../004-functions/414-throws.md#41413-panic과-fatal-에러)).

### `nabi_gc_add_root`

```
extern "nabi" {
    fn nabi_gc_add_root(word: ConstPtr<usize>);   // static 의 참조 word 를 프로세스 끝까지 GC root 로 등록
}
```

컴파일러가 만든 시작 코드가 GC 참조를 담는 모든 `static`에 대해 호출합니다.

### `nabi_os_*`: OS 헬퍼(내부, 변경 가능)

표준 라이브러리 `std`(fs, io, env, process)를 뒷받침하는 내부 심볼입니다. ABI 안정성 대상이 아니며 바뀔 수 있습니다. 그래서 사용자 코드는 직접 호출하지 않습니다.

```
nabi_os_errno_kind     nabi_os_strerror       nabi_os_open           nabi_os_close
nabi_os_read           nabi_os_write          nabi_os_poll_read      nabi_os_fsync
nabi_os_ftruncate      nabi_os_lseek          nabi_os_dup            nabi_os_fchmod
nabi_os_mmap           nabi_os_munmap         nabi_os_mmap_copy      nabi_os_flock
nabi_os_copy_fd        nabi_os_clone          nabi_os_stat           nabi_os_mkdir
nabi_os_unlink         nabi_os_rename         nabi_os_link           nabi_os_opendir
nabi_os_readdir        nabi_os_closedir       nabi_os_getcwd         nabi_os_chdir
nabi_os_realpath       nabi_os_current_exe    nabi_os_argc           nabi_os_arg
nabi_os_env_at         nabi_os_getenv         nabi_os_setenv         nabi_os_spawn
nabi_os_waitpid        nabi_os_signal_install nabi_os_raise          nabi_os_kill
nabi_os_pidfd_open     nabi_os_pidfd_signal   nabi_os_getpid         nabi_os_getppid
nabi_os_abort          nabi_os_random         nabi_os_trust_roots
nabi_os_openat_raw     nabi_os_create_file2   nabi_os_chmod
```

Windows에서는 런타임이 fd와 `HANDLE` / `SOCKET` 값을 서로 바꾸는 `nabi_os_raw_handle`, `nabi_os_raw_socket`, `nabi_os_handle_fd`, `nabi_os_socket_fd`도 내보냅니다.

### `nabi_stdio_*` / `nabi_stdin_read`: 표준 스트림(내부, 변경 가능)

`print` / `eprint`와 표준 입력을 뒷받침하는 내부 심볼입니다. ABI 안정성 대상이 아니며 바뀔 수 있습니다. 바이트 영역은 `GcBuf<u8>`의 `(buf, off, len)`입니다. 실패는 `-errno`로 돌려줍니다.

```
extern "nabi" {
    fn nabi_stdio_write(fd: i32, s: str, newline: i32) -> i32;                       // s (newline != 0 이면 이어서 "\n") 를 fd 1 또는 2 로; 0 또는 -errno
    unsafe fn nabi_stdio_write_buf(fd: i32, buf: Ptr<u8>, off: usize, len: usize) -> i32;   // buf 의 off 부터 len 바이트를 fd 1 또는 2 로
    fn nabi_stdio_flush() -> i32;                                                     // stdout 버퍼를 내보냄; 0 또는 -errno
    unsafe fn nabi_stdin_read(buf: Ptr<u8>, off: usize, len: usize, delim: i32) -> isize;
        // stdin 에서 최대 len 바이트를 buf 의 off 위치로 읽고, 첫 delim 바이트 뒤에서 멈춤 (delim < 0: 멈추지 않음);
        // 읽은 수 (입력 끝이면 0) 또는 -errno
}
```

### `nabi_deque_*`: work-stealing deque

task 스케줄러([7.6.2](../007-runtime/706-scheduler.md#762-work-stealing-구조))의 lock-free Chase-Lev deque입니다. deque와 그 ring은 GC가 추적하는 GC 객체입니다. 그래서 deque 안에만 있는 원소도 살아 있습니다. **Thread-safety:** `push` / `pop`은 소유 스레드만, `steal`과 `len`은 어느 스레드든 호출할 수 있습니다.

```
extern "nabi" {
    fn nabi_deque_new() -> GcBuf<u8>;                                 // 빈 deque (GC 객체)
    unsafe fn nabi_deque_push(d: Ptr<u8>, item: usize);               // 소유자: 끝에 넣음 (GC 참조)
    unsafe fn nabi_deque_pop(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;    // 소유자: 가장 새 원소를 out[index] 에, 있으면 1
    unsafe fn nabi_deque_steal(d: Ptr<u8>, out: Ptr<u8>, index: usize) -> i32;  // 가장 오래된 원소를 out[index] 에, 있으면 1
    unsafe fn nabi_deque_len(d: Ptr<u8>) -> usize;                    // 원소 수 (경합 중의 근사값)
}
```

`out`은 GC 참조를 담는 `GcBuf`이며, 함수는 결과를 `index` 자리에 씁니다. 이 함수들 안에는 safepoint가 없습니다.

## 8.5.5. I/O / 타이머 영역

### `nabi_io_register` / `nabi_timer_register`

C나 저수준 코드가 fd가 준비될 때나 특정 시각에 task를 깨우도록 등록합니다. 내부에서는 reactor([7.7](../007-runtime/707-io-and-timer.md))와 executor의 타이머를 씁니다. task executor 밖에서 호출하면 abort로 끝납니다.

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_io_register(fd: i32, interest: u32, waker: Ptr<u8>);
    fn nabi_timer_register(deadline: u64, waker: Ptr<u8>);
}
```

- `fd`: 파일 디스크립터입니다.
- `interest`: `1` = 읽기, `2` = 쓰기(둘을 OR할 수 있음).
- `waker`: 원소가 하나인 `GcBuf<Waker>`입니다. 등록은 edge-triggered이고 **한 번만** 깨웁니다. 가짜 깨움(spurious wakeup)이 있을 수 있습니다. fd가 이미 준비되어 있거나 감시할 수 없으면 곧바로 깨웁니다.
- `deadline`: `nabi_time_now_ns` 시계 기준의 절대 시각(ns)입니다. 그 시각이 되면 waker를 한 번 깨웁니다.

### `nabi_io_*`: reactor(내부, 변경 가능)

[I/O 드라이버](../007-runtime/707-io-and-timer.md)의 reactor를 이루는 내부 심볼입니다. ABI 안정성 대상이 아니며 바뀔 수 있습니다. **Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_io_poller() -> i32;                    // 프로세스의 kqueue / epoll fd (처음 부를 때 생성)
    fn nabi_io_watch(fd: i32, interest: u32) -> i32;   // fd 를 edge-triggered 로 등록 (다시 걸어도 무해); interest 1 읽기, 2 쓰기; 0 또는 -errno
    fn nabi_io_poll(out: Ptr<u8>, max: usize) -> i32;   // 변화가 올 때까지 막힘 (GC 에는 park), 사건 수 또는 -errno
    fn nabi_io_armed() -> i32;                     // WASI 전용: 걸린 watch 가 있으면 1
    fn nabi_io_wait(out: Ptr<u8>, max: usize, timeout_ns: i64) -> i32;   // WASI 전용: 시간 제한이 있는 nabi_io_poll (< 0: 제한 없음); 0 = 시간 초과
}
```

`nabi_io_poll`은 사건을 `GcBuf<i64>`인 `out`에 `fd * 2 + direction`(읽기 0, 쓰기 1) 형태로 최대 `max`개 씁니다.

### `nabi_net_*`: 소켓 헬퍼(내부, 변경 가능)

표준 라이브러리 `std::net`([stdlib 3.2](../../refs-standard-libraries/300-std/302-net.md))을 뒷받침하는 내부 심볼입니다. ABI 안정성 대상이 아니며 바뀔 수 있습니다. 그래서 사용자 코드는 직접 호출하지 않습니다. 소켓은 항상 non-blocking으로 만들고, 실패는 `-errno`로 돌려줍니다.

```
nabi_net_socket        nabi_net_bind          nabi_net_listen        nabi_net_connect
nabi_net_sock_error    nabi_net_accept        nabi_net_addr          nabi_net_listeners
nabi_net_send          nabi_net_recv          nabi_net_send_to       nabi_net_recv_from
nabi_net_shutdown      nabi_net_setopt        nabi_net_multicast_v4  nabi_net_wait
nabi_net_resolve       nabi_net_unix          nabi_net_unix_pair     nabi_net_unix_path
nabi_net_send_fd       nabi_net_recv_fd       nabi_net_sendfile      nabi_net_send_zerocopy
nabi_net_splice        nabi_net_getopt
```

`nabi_net_socket`은 `net::Socket`의 도메인, 타입, 프로토콜을 받습니다. `nabi_net_setopt` / `nabi_net_getopt`는 라이브러리가 정한 코드로 소켓 옵션을 쓰고 읽습니다. `nabi_net_wait`는 blocking 호출이 준비를 기다리는 `poll(2)`입니다(GC가 보기에는 네이티브 구간). `nabi_net_resolve`는 `getaddrinfo`입니다. 그 실패 코드 `e`는 `-(100000 + |e|)`로 돌려줍니다. `nabi_os_strerror`와 `nabi_os_errno_kind`는 이 값을 errno처럼 받습니다. `nabi_net_unix*`, `nabi_net_send_fd` / `nabi_net_recv_fd`, `nabi_net_sendfile`, `nabi_net_send_zerocopy`, `nabi_net_splice`는 Windows에서 export하지 않습니다.

## 8.5.6. 에러 전달(심볼 없음)

[에러 제어 흐름](../003-expressions/314-error-control-flow.md)의 `throw` / `try` / `catch`에는 런타임 심볼이 없습니다. 에러를 던지는 함수는 에러를 반환값의 일부로 돌려주고([8.6.5](806-calling-convention.md#865-던지는-함수의-반환-값-기반-에러-전달)), 호출 지점이 그 값을 검사해 분기합니다([8.9.6](809-lowering-control.md#896-try--catch--finally-값-기반-분기)). 따라서 에러를 던지거나 현재 에러를 꺼내려고 런타임을 호출하는 일은 없습니다. 에러 처리용 TLS 슬롯이나 jmp_buf도 없습니다.

## 8.5.7. 진단 영역

`wasm32-wasip2`에서는 아래 종료 코드(`101` 등)가 호스트에 단순한 실패로 전달됩니다([4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)).

### `nabi_panic`

**Thread-safety: `thread-safe`**(어느 스레드에서든 호출 가능, 호출하면 즉시 프로세스 종료).

```
extern "nabi" {
    fn nabi_panic(msg: ConstPtr<u8>, msg_len: usize) -> !;
}
```

복구할 수 없는 panic이며 fatal입니다. 이 함수는 반환하지 않습니다. 메시지를 stderr에 출력한 뒤 종료 코드 `101`로 프로세스를 끝냅니다. `catch`나 `finally`는 실행하지 않습니다.

### `nabi_panic_str`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_panic_str(msg: str) -> !;          // msg 는 (ptr, len)
}
```

`nabi_panic`과 같지만 메시지를 `str`로 받습니다. 위치는 출력하지 않습니다. 런타임 내부 panic에만 씁니다.

### `nabi_panic_at`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    #[track_caller]
    fn nabi_panic_at(msg: str) -> !;
}
```

선언에 `#[track_caller]`([6.2.12](../006-meta-system/602-built-in-attributes.md#6212-track_caller-호출-위치-전달))가 붙어 있습니다. 그래서 심볼은 호출자의 위치를 숨은 마지막 `str` 인자(`"path:line:col"`)로 받습니다. 형태는 `nabi_panic_at(msg, at)`입니다.

[4.14.13](../004-functions/414-throws.md#41413-panic과-fatal-에러)의 형식으로 출력하고 종료 코드 `101`로 끝냅니다. core의 `panic`과 컴파일러가 생성한 panic(경계 검사, 오버플로, 0으로 나누기 등)이 이 심볼을 호출합니다.

### `nabi_uncaught_error` / `nabi_uncaught_error_at`

```
extern "nabi" {
    fn nabi_uncaught_error(what: str) -> !;              // 미처리 예외: what 을 stderr 에 출력하고 종료 코드 1
    fn nabi_uncaught_error_at(what: str, at: str) -> !;  // 그 뒤에 "  --> <at>" 줄
}
```

진입점 밖으로 전파된 에러를 처리합니다([4.16.4](../004-functions/416-entry-point.md#4164-종료-코드와-미처리-예외)). 선언은 core에 있습니다.

### `nabi_oom`

**Thread-safety: `thread-safe`**.

```
extern "nabi" {
    fn nabi_oom() -> !;
}
```

`fatal error: out of memory`를 stderr에 출력하고 종료 코드 `101`로 종료합니다. 런타임 안의 할당 실패와 core의 `oom()` 모두 이 심볼을 거칩니다. 런타임은 Nabi 함수를 이름으로 호출할 수 없습니다. 그래서 할당에 실패하면 `oom_fn` 슬롯의 함수를 호출하지 않고 이 심볼을 직접 씁니다.

### 스택 오버플로

스레드의 스택이 넘치면 런타임은 가능한 경우 stdout을 비우고(flush), stderr에 다음을 출력한 뒤 abort(SIGABRT)로 끝냅니다. panic과 달리 unwind도 없고 `finally`도 실행하지 않습니다.

```
thread '<name>' has overflowed its stack
fatal runtime error: stack overflow, aborting
```

`<name>`은 `thread::Builder::name`으로 준 스레드 이름입니다(OS 스레드 이름으로도 설정함, [7. 런타임](../007-runtime/README.md#런타임-에러-스택-오버플로)). 대상은 main 스레드(`nabi_runtime_init`), `nabi_thread_spawn` 스레드(executor 워커 포함, 깊은 `await` 사슬도), `nabi_thread_init`을 호출한 C 스레드입니다. 런타임은 각 스레드의 스택 아래 guard 범위를 TLS 슬롯 `nabi_tls_stack_guard`([8.3.1](803-tls-slots.md#831-tls-슬롯-구조))에 기록하고, 대체 신호 스택(`sigaltstack`)을 둡니다. 그 범위 안의 SIGSEGV / SIGBUS만 스택 오버플로로 보고합니다. 그 밖의 fault는 런타임이 핸들러를 두기 전의 동작으로 갑니다(Nabi를 공유 라이브러리로 불러온 호스트 프로세스가 정한 것이 없으면 기본 동작). wasm에서는 모듈이 trap을 잡을 수 없으므로 호스트가 보고합니다. 함께 배포하는 JS 호스트는 위 메시지를 출력합니다.

## 8.5.8. 심볼 명명 규약

접두사별 영역은 다음과 같습니다.

| Prefix          | 영역           |
|-----------------|----------------|
| `nabi_gc_*`      | GC 관련        |
| `nabi_runtime_*` | 런타임 일반    |
| `nabi_thread_*`  | OS 스레드 관련 |
| `nabi_coro_*`    | 코루틴 관련    |
| `nabi_task_*`    | task 관련      |
| `nabi_io_*`      | I/O            |
| `nabi_timer_*`   | 타이머         |
| `nabi_panic`     | 진단           |

이 접두사 규약으로 외부 라이브러리나 사용자 코드와 심볼이 충돌하지 않게 합니다. 8.5.1~8.5.7의 표가 규범적인 런타임 심볼입니다. 런타임은 구현이 정하는 `nabi_*` 심볼(예: `nabi_fmt_*`, `nabi_os_*`, `nabi_stdio_*`, `nabi_static_init_*`)을 더 내보낼 수 있습니다.

## 8.5.9. 함수 시그니처의 호환성

각 함수의 시그니처는 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)의 일부입니다. 변경 정책은 다음과 같습니다.

- 1.0 전: 0.x.0 릴리스마다 호환성을 깨는 변경을 할 수 있습니다. 0.x.y 릴리스에서는 호환되는 변경만 합니다.
- 1.0부터 minor: 호환되는 변경만 할 수 있습니다. 인자를 추가할 때는 함수 래퍼로 처리합니다.
- 1.0부터 major: 호환성을 깨는 변경을 할 수 있습니다. 이때 ABI 버전 해시가 바뀝니다.

자세한 내용은 [개요](README.md)의 안정성 정책을 참고하세요.

## 8.5.10. 호출 규약

위 모든 함수의 호출 규약은 타깃 플랫폼의 C ABI(예: System V x86_64, AAPCS aarch64, MS x64 Windows)를 따릅니다.

## 8.5.11. 예: TLAB fast path의 slow 호출

```
t = <nabi_tls address>                ; 8.3.8
cursor = load.ptr [t + 8]             ; tlab_cursor at p
limit = load.ptr [t + 16]             ; tlab_limit at 2p
if 24 <= limit - cursor then bb_fast else bb_slow

bb_slow:                              ; placed out of line
    obj = call nabi_gc_alloc_slow(24, 8, 12345)
    goto bb_cont

bb_fast:
    store.ptr [t + 8], cursor + 24
    obj = cursor
    goto bb_cont

bb_cont:
    ; ... continue
```

위 예시는 fast path와 slow path를 묶은, 내보내는 코드의 구조를 보여 줍니다(의사 코드. fast path의 헤더 store는 [8.9.9](809-lowering-control.md#899-gc-할당-fast--slow-path)에 있습니다).

## 8.5.12. 보조 심볼(부록, 비규범)

> [!IMPORTANT]
> 이 절은 **비규범**(non-normative)입니다. 아래 심볼은 참조 런타임(`nabir`)이 내보내는 보조 심볼입니다. ABI 본 표(8.5.1~8.5.11)에 속하지 않으며 안정성 약속도 없습니다. 런타임은 구현이 정하는 `nabi_*` 심볼을 더 내보낼 수 있습니다(8.5.8).

시그니처는 본문과 같은 표기(`Ptr<T>` = 가변 raw 포인터, `ConstPtr<T>` = 읽기 전용 raw 포인터)를 따릅니다. 문자열 인자는 NUL로 끝나는 C 문자열입니다.

```
extern "nabi" {
    // 메모리
    fn nabi_free(ptr: usize);                               // 0(null) 무시

    // C 문자열
    fn nabi_cstr_len(s: ConstPtr<u8>) -> usize;             // strlen; null → 0

    // GC
    fn nabi_gc_collect_young();                             // minor 수집. 동시 mark 중에는 하지 않음
    fn nabi_gc_collect_young_within(ns: u64) -> u32;        // 추정이 ns 안이면 같은 수집. 했으면 1
    fn nabi_gc_set_pause_goal(ns: u64);                     // 0: 없음 (bare::gc::set_pause_goal)
    fn nabi_gc_set_heap_limit(bytes: u64);                  // 0: 없음 (bare::gc::set_heap_limit)
    fn nabi_gc_no_collect(enter: u32);                      // 1: 수집하지 않는 구간에 들어감, 0: 나감
    fn nabi_gc_arena_epoch() -> u64;                        // 어떤 bare::gc::Arena도 쓴 적 없는 세대
    fn nabi_gc_weak_new(target: Ptr<u8>) -> Ptr<u8>;       // target 을 가리키는 weak cell (type id 3)
    fn nabi_gc_weak_get(cell: Ptr<u8>) -> Ptr<u8>;         // 대상, 해제되었으면 null
    fn nabi_gc_pinned(obj: GcBuf<u8>) -> i32;              // obj 가 pin 된 동안 1 (List / StrBuf 가 늘어나기를 거부)

    // FFI (bare::ffi, std::panic); name 은 str (ptr, len)
    fn nabi_handle_new(obj: GcBuf<u8>, name: str) -> usize;  // obj(타입 name 의 값)를 루트로 잡고 pin 하는 handle, 0 이 아님
    fn nabi_handle_get(h: usize, name: str) -> GcBuf<u8>;  // 그 객체; release 됐거나 타입이 다르면 panic
    fn nabi_handle_retain(h: usize);
    fn nabi_handle_release(h: usize);                      // 마지막 release 가 객체를 unpin
    fn nabi_panic_set_hook(hook: extern "C" fn(Ptr<u8>, ConstPtr<u8>, usize, ConstPtr<u8>, usize) | null, ud: Ptr<u8>) -> Ptr<u8>;
        // hook(ud, msg, msg_len, at, at_len) 이 모든 panic 의 보고 뒤에 돔;
        // 이전 ud 를 돌려주어 그 주인이 release 하게 함
    fn nabi_errno_get() -> i32;                            // 저장된 errno
    fn nabi_errno_set(v: i32);                             // 저장된 errno 와 C 라이브러리의 errno
    fn nabi_last_error_get() -> u32;                       // 저장된 GetLastError (Windows 밖에서는 0)
    fn nabi_last_error_set(v: u32);                        // 저장된 값과, Windows 에서는 SetLastError

    // 프로세스 종료
    fn nabi_set_exit_hook(hook: extern "C" fn(i32) | null);
        // panic, 처리되지 않은 에러, exit, (Windows) stack overflow 가 프로세스를
        // 끝내기 전에, 끝나는 스레드에서 종료 상태와 함께 호출; hook 이 반환하면
        // 프로세스가 끝남

    // REPL host (세션 저장 / 복원)
    fn nabi_session_hold(timeout_ns: i64) -> i32;          // 조용한 순간에 world 를 멈춤: 실행 중인 poll / blocking job 이
                                                           //   없고 lock 안에서 멈춘 스레드가 없을 때. 0: timeout_ns 안에
                                                           //   못 함 (요청은 남음)
    fn nabi_session_unhold();                               // 대기 중이거나 얻은 hold 를 거둠
    fn nabi_session_resume();                               // hold 뒤에 world 를 재개
    fn nabi_session_enter();                                // poll / blocking job 둘레의 executor 스레드 gate
    fn nabi_session_leave();
    fn nabi_session_pass();
    fn nabi_lock_notes() -> i32;                            // 세션 host 에서 1
    fn nabi_lock_note(word: GcBuf<i32>, kind: i32) -> i32;  // kind: 0 Mutex, 1 read, 2 write, 3 실행 중인 Once; 1 = 기록함
    fn nabi_lock_drop(word: GcBuf<i32>);
    fn nabi_world_threads(out: Ptr<u64>, cap: usize) -> usize;   // world 정지 중: 다른 mutator 의 OS id (Windows)
    fn nabi_session_restored(lost: ConstPtr<u64>, n: usize);     // 세션을 되돌림. n == SIZE_MAX: 다른 스레드는 모두 사라짐
    fn nabi_thread_lost(worker_id: u64) -> i32;             // 실패한 셀 뒤에 그 스레드가 사라졌으면 1
    fn nabi_io_quiesce();                                   // Windows: 메모리를 되돌리기 전에 진행 중인 I/O 취소
    fn nabi_watch_writes();                                 // Windows: heap 을 write-watch. 런타임 초기화 전에 호출
    fn nabi_stdio_line_buffered();                          // 터미널에서처럼 줄바꿈마다 stdout 을 flush
    fn nabi_set_stdin_hook(hook: extern "C" fn(Ptr<u8>, usize) -> isize | null);
        // stdin 을 hook 으로 읽음 (null: fd 0): 읽은 수, 입력의 끝이면 0,
        // 또는 -errno. 매 호출 전에 stdout 을 flush
    static nabi_page_epoch: u64;                            // Windows: watch 하는 page 가 쓰기 없이 바뀌거나
                                                           //   watch 하는 block 이 해제되면 증가
}
```

다음 규칙을 적용합니다.

- 사용자 코드는 이 심볼에 의존하면 안 됩니다. 시그니처는 예고 없이 바뀌거나 제거될 수 있습니다. 표준 라이브러리는 런타임과 함께 배포되며 그중 일부(`nabi_free`, `nabi_gc_weak_new`, `nabi_gc_weak_get`, `nabi_gc_pinned`, `nabi_panic_set_hook`, `nabi_handle_*`, `nabi_errno_*`, `nabi_last_error_*`, `nabi_session_*`, `nabi_lock_*` 함수)를 호출합니다.
- `nabi_` 접두사를 함께 쓰지만 8.5.8의 영역 표에는 들어가지 않습니다. 이름 충돌을 피하려고 사양의 규범 심볼은 위 이름을 다시 쓰지 않습니다.
- `nabi_panic`은 이 부록에 속하지 않습니다. 시그니처는 [8.5.7](#857-진단-영역)의 `nabi_panic(msg: ConstPtr<u8>, msg_len: usize)`가 기준입니다. 구현이 이와 다른 시그니처를 내보내면 사양에 맞지 않습니다(non-conformance).
