# 8.9. Lowering: 제어 흐름과 GC 통합

이 절은 [`if` 표현식](../003-expressions/310-if-expression.md), [`match`](../003-expressions/312-match-expression.md), [루프](../003-expressions/313-loop-expressions.md), [에러 제어 흐름](../003-expressions/314-error-control-flow.md)을 낮추는 규칙을 다룹니다. [GC](../000-meta/007-glossary.md#gc-garbage-collector)의 alloc / barrier / safepoint 코드를 내보내는 규칙도 정의합니다.

nabi-mir는 제어 흐름 그래프입니다. 블록 인자를 가진 기본 블록이고, 각 블록은 goto, `bool`에 대한 두 갈래 분기, return, `unreachable` 중 하나로 끝납니다. 코드 생성기는 이들을 각각 점프, 조건 분기, return, trap으로 내보냅니다. 블록 인자는 점프 전에 대상 블록의 슬롯에 쓰는 store입니다. `-O0`보다 높은 수준에서는 최적화기가 이 그래프 위에서 동작합니다([8.7.3](807-code-generation.md#873-최적화-수준)). 코드는 의사 코드입니다([8.7.1](807-code-generation.md#871-변환-흐름)).

## 8.9.1. `if` 표현식

```nabi
if cond { x } else { y }
```

이 표현식은 두 갈래 분기로 낮춥니다. 값은 합류 블록의 인자로 도착합니다.

```
bb0:
    if cond then bb_then else bb_else
bb_then:
    goto bb_end(x)
bb_else:
    goto bb_end(y)
bb_end(result):
```

`else` 갈래가 없고 값이 unit인 `if`는 합류 블록으로 바로 분기합니다.

```nabi
if cond { do_thing(); }
```

```
bb0:
    if cond then bb_then else bb_end
bb_then:
    call do_thing(tls)
    goto bb_end
bb_end:
```

## 8.9.2. `if let`

[`if` 표현식](../003-expressions/310-if-expression.md)의 `if let pattern = expr` 형태입니다.

```nabi
if let v: T = compute() { process(v) } else { fallback() }
```

패턴 검사는 `match`(8.9.3)처럼 낮춥니다.

```
bb0:
    opt = call compute(tls)            ; (i32 tag, payload)
    if opt.0 != TAG_NONE then bb_then else bb_else
bb_then:
    v = opt.1 as T
    call process(v, tls)
    goto bb_end
bb_else:
    call fallback(tls)
    goto bb_end
```

## 8.9.3. `match`의 결정 트리

[`match`](../003-expressions/312-match-expression.md)의 갈래는 소스 순서대로 검사하며, 각 패턴은 두 갈래 분기 검사의 연속입니다.

```nabi
match v {
    1 => a,
    2 => b,
    n if n > 100 => c,
    _ => d,
}
```

이 `match` 표현식은 다음과 같이 됩니다.

```
bb0:
    if v == 1 then bb_a else bb_test2
bb_test2:
    if v == 2 then bb_b else bb_guard
bb_guard:
    n = v
    if n > 100 then bb_c else bb_d

bb_a:
    goto bb_end(a)
bb_b:
    goto bb_end(b)
bb_c:
    goto bb_end(c)
bb_d:
    goto bb_end(d)

bb_end(result):
    ...
```

튜플, `struct`, `enum` 변형([8.8.6](808-lowering-types.md#886-enum의-표현)), 합 타입 멤버도 같은 방식으로 각 부분을 검사합니다. 범위 패턴은 비교 두 번, 문자열 패턴은 `str` 동등 비교입니다([8.7.6](807-code-generation.md#876-extern-함수런타임-함수)). match는 망라적이므로 마지막 검사 뒤의 블록은 unreachable(trap)입니다.

점프 테이블은 만들지 않습니다. 사슬은 모든 수준에서 쓰인 그대로 컴파일되며, 검사하는 갈래마다 비교와 분기가 하나씩입니다. 앞선 분기들이 증명한 것으로 결정할 수 있는 검사는 최적화기가 없앱니다.

## 8.9.4. `loop` / `while` / `for`

### `loop`

```nabi
let v = loop { ...; if cond { break x; } }
```

루프는 헤더 블록과 그리로 돌아가는 back-edge 점프입니다. `break x`는 `x`를 인자로 출구 블록에 분기합니다.

```
    goto bb_head
bb_head:
    ; body
    if cond then bb_exit(x) else bb_latch
bb_latch:
    ; safepoint poll (8.9.11)
    goto bb_head
bb_exit(v):
```

### `while`

```nabi
while cond { body }
```

조건은 헤더에서 검사합니다.

```
    goto bb_head
bb_head:
    c = ...
    if c then bb_body else bb_exit
bb_body:
    ; body, then the back-edge poll
    goto bb_head
bb_exit:
```

### `for`

```nabi
for x in xs { body }
```

[루프 표현식](../003-expressions/313-loop-expressions.md)의 풀어 쓰기로 `let it = xs.into_iter(); loop { match it.next() { ... } }`가 되고, 위와 같은 lowering을 거칩니다. 정수 범위(`for i in a..b`, 경계는 한 번만 평가)나 배열에 대한 `for`는 바로 세는 `while` 루프로 낮춥니다. 인덱스는 본문 앞에서 증가시키므로 `continue`가 증가를 건너뛰지 못합니다.

## 8.9.5. `break` / `continue`

`break`와 `continue`는 루프의 출구 블록이나 back-edge 블록으로 가는 점프입니다. 레이블(`'outer:`)도 마찬가지로 해당 [블록](../000-meta/007-glossary.md#block-블록)으로 분기합니다.

## 8.9.6. `try` / `catch` / `finally`: 값 기반 분기

[에러 제어 흐름](../003-expressions/314-error-control-flow.md)의 에러는 [Calling convention](806-calling-convention.md#865-던지는-함수의-반환-값-기반-에러-전달)에서 정의한 `{ is_err, payload }` 반환값으로 전달됩니다. 따라서 `try`와 `catch`는 둘 다 일반 분기로 낮춥니다.

```nabi
let v = try f(x);              // f: fn(X) throws (E) -> T
```

`try`로 표시한 호출은 반환값의 `is_err`를 검사합니다. 성공이면 페이로드를 `T`로 꺼내고, 에러면 에러 경로로 분기합니다.

```
bb0:
    r = call f(x, tls)                 ; (is_err, payload)
    if r.0 then bb_err else bb_ok

bb_ok:
    v = r.1 as T
    ...

bb_err:
    e = r.1 as E
    ; if there is an enclosing catch, go to its dispatch block;
    ; otherwise build the enclosing function's return value { 1, e } and return it
    goto bb_catch_dispatch(e)
```

`catch` 절이 붙은 식에는 왼쪽 식의 모든 에러 경로가 모이는 dispatch 블록이 하나 있습니다.

```nabi
try body catch e: A { handler_a } catch _ { handler_any }
```

```
bb_catch_dispatch(e):
    ; test patterns in clause order (same scheme as the decision tree of 8.9.3)
    if <e is A> then bb_handler_a(e) else bb_handler_any(e)

bb_handler_a(ea):
    ; handler_a
    goto bb_end(result_a)

bb_handler_any(ex):
    ; handler_any
    goto bb_end(result_any)

bb_end(result):
    ...
```

매칭되는 절이 없으면 dispatch 블록은 에러를 바깥 `catch`의 dispatch 블록으로 넘깁니다. 넘길 곳이 없으면 둘러싼 함수의 에러 반환값을 만들어 반환합니다. 에러 타입을 적지 않은 `throws`의 에러(`any Error`)를 타입 패턴으로 검사하는 코드는 [`any`](808-lowering-types.md)의 tag 비교로 낮춥니다.

`try?`는 에러 경로에서 `none`을 만들어 `bb_end`로 가고, `try!`는 에러 경로에서 `panic_fn`을 호출합니다.

## 8.9.7. `throw`

`throw`는 다음과 같이 씁니다.

```nabi
throw err;
```

`throw`는 에러 경로로 가는 무조건 분기로 낮춥니다. 가장 가까운 `catch`가 같은 함수 안에 있으면 그 dispatch 블록으로 가고, 없으면 둘러싼 함수의 에러 반환값을 만들어 반환합니다.

```
; when a catch exists in the same function
goto bb_catch_dispatch(err)

; otherwise, error return from the function
r.0 = true                ; is_err
r.1 = err                 ; the payload as E
return r
```

함수 밖으로 에러를 돌려주는 경로도 일반 `return`과 같습니다. 그래서 shadow stack pop 같은 epilogue가 그대로 실행됩니다. 런타임 호출은 없습니다.

## 8.9.8. `finally`

`finally`는 다음과 같이 씁니다.

```nabi
try { body } finally { cleanup }
```

왼쪽 식을 벗어나는 모든 경로가 cleanup 블록을 거치도록 낮춥니다. 정상 종료, `catch`로 처리한 뒤의 종료, 처리하지 않은 에러의 전파, `return` / `break` / `continue`의 각 경로가 여기에 해당합니다. 각 경로는 cleanup 블록으로 분기한 뒤 원래 목적지로 이어집니다. 에러 전파도 일반 분기이므로 별도의 unwind 메커니즘이 필요 없습니다.

```
bb_body_ok(v):        goto bb_cleanup(K_OK, v)
bb_body_err(e):       goto bb_cleanup(K_ERR, e)
bb_body_return(rv):   goto bb_cleanup(K_RET, rv)

bb_cleanup(kind, val):
    ; cleanup
    ; depending on kind, branch to bb_end / error propagation / function return / loop destination
```

cleanup 블록 안에서 새 에러를 던지면 그 에러가 원래 경로를 대신합니다. panic은 프로세스를 끝내므로 cleanup 블록을 거치지 않습니다.

## 8.9.9. GC 할당: fast / slow path

[할당](../007-runtime/702-allocation.md)도 함께 참고하세요. `struct`는 값이므로 할당하지 않습니다(8.8.3). 힙 객체는 클로저 환경, box(`any`), `List` / `Dict` 버퍼, 실행 중에 만든 문자열입니다.

```nabi
let n = 10;
let f = |x: i32| x + n;    // env: header + i32 하나
```

환경은 할당하는 자리에서 인라인으로 할당합니다.

```
cur = load.ptr [t + p]                 ; tlab_cursor
lim = load.ptr [t + 2p]                ; tlab_limit
if 24 <= lim - cur then bb_fast else bb_slow
bb_fast:
    store.ptr [t + p], cur + 24        ; bump the cursor
    store.u32 [cur + 0], TYPE_ID_ENV   ; type_id
    store.u64 [cur + 8], 24            ; gc_word
    env = cur
    goto bb_done
bb_slow:                               ; placed out of line
    env = call nabi_gc_alloc_slow(24, 8, TYPE_ID_ENV)
    goto bb_done
bb_done:
```

`t`는 스레드의 TLS 기준 주소([8.7.7](807-code-generation.md#877-tls-접근)), `24`는 객체 크기(8로 올림한 값), `8`은 정렬, `TYPE_ID_ENV`는 type_id입니다. fast path는 객체가 들어가면 TLAB 커서를 올리고 헤더의 `type_id`와 `gc_word`(= 크기, [8.1.4](801-object-header-abi.md#814-gc_word오프셋-8-u64-또는-ptr))를 씁니다. 헤더 주소가 곧 객체입니다. 들어가지 않으면 `nabi_gc_alloc_slow`를 부릅니다. `gc_flags`는 쓰지 않습니다. TLAB 메모리는 0으로 채워져 있기 때문입니다. 대형 객체 크기 이상이거나 8보다 크게 정렬된 객체는 `nabi_gc_alloc_slow`를 바로 부릅니다.

그다음 16바이트 헤더 뒤에 필드를 씁니다.

```
store.i32 [env + 16], n
```

## 8.9.10. write barrier

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md#742-write-barrier-object-remembering)에서 정한 객체 기록(object-remembering) barrier입니다.

```nabi
*p.field = q;       // GC 객체에 GC 포인터 쓰기
```

이 식은 다음과 같이 낮춥니다.

```
; 1. write barrier: checks only the header of the destination p (may precede the store)
flags = load.u32 [p + 4]               ; gc_flags
if flags & UNLOGGED != 0 then bb_log else bb_store

bb_log:                                ; placed out of line
    call nabi_gc_wb_slow(p, slot, len) ; atomically clear UNLOGGED and add to the remembered set
    goto bb_store

bb_store:
    ; 2. ordinary store
    store.ptr [p + offset_of(field)], q
```

- 저장하는 값 `q`의 헤더는 읽지 않습니다. `q`는 `any` 슬롯의 인라인 정수나 `str`의 내부 포인터일 수 있으므로 헤더로 읽으면 안전하지 않습니다.
- fast path는 load 하나와 분기 하나입니다. 실제로는 `UNLOGGED | CARDED`를 함께 검사합니다. `CARDED`(큰 old 객체)이면 호출 없이 card table에 dirty를 표시합니다. 표시하는 card는 쓴 슬롯이 걸치는 512 B card(`(slot - p) >> 9` ..= `(slot + len - 1 - p) >> 9`)입니다. card table 포인터는 헤더 바로 앞 word입니다. `UNLOGGED`이면 위처럼 `nabi_gc_wb_slow`를 호출합니다. 그래서 barrier는 슬롯 주소와 저장하는 값의 크기를 씁니다([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-object-remembering)).
- `nabi_gc_wb_slow`는 safepoint가 아닙니다. 그래서 barrier와 store 사이에 수집이 끼어들지 않습니다.
- 방금 할당한 객체를 초기화할 때, `static`에 store할 때, 원시 `Ptr<T>` 메모리, 참조가 아닌 atomic에는 barrier를 넣지 않습니다.

## 8.9.11. safepoint poll

[safepoint 프로토콜](804-safepoint.md)도 참고하세요.

```
; safepoint poll (t: the TLS base, 8.7.7)
flag = load.u8 [t + 10p+8]             ; yield_flag
if flag != 0 then bb_slow else bb_cont

bb_slow:                               ; placed out of line
    call nabi_runtime_yield()
    goto bb_cont

bb_cont:
    ; ... continue ...
```

컴파일러는 루프의 back-edge에 위 코드를 내보냅니다. 루프 poll을 거치지 않고 호출 사이클로 다시 들어올 수 있는 함수(재귀 사이클의 멤버, 포인터나 C에서 불리는 함수, export된 함수)에도 내보냅니다. 직접 Nabi 호출이 아닌 경로로 들어오는 함수는 진입 지점에, 나머지는 각 경로의 첫 safepoint 앞에 둡니다([8.4.7](804-safepoint.md#847-safepoint-불변식)). 그 밖의 함수는 진입 시 폴링하지 않습니다. 호출자가 폴링하기 때문입니다.

## 8.9.12. shadow stack 프레임

[배리어와 루트 추적](../007-runtime/704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack)도 함께 참고하세요. 컴파일러는 플랫폼의 도움 없이 프레임을 체인에 잇고 떼는 코드(link / unlink)를 직접 내보냅니다.

GC 참조 word가 2개(`x`, `y`, 스택 슬롯)인 함수의 예입니다. 프레임에는 각 슬롯의 주소를 넣습니다. 그래서 변수에 대입할 때는 평소처럼 `x`에 store하기만 하면 됩니다.

```
; entry: zero-initialize the words, link the frame
; frame: a stack slot (prev: ptr, count: usize, slots: [ptr; 2])
store.ptr [x], null
store.ptr [y], null
store.ptr [frame + 0], load.ptr [t + 0]    ; prev = the shadow stack head (offset 0 of the TLS block)
store.usize [frame + p], 2                 ; count
store.ptr [frame + 2p], &x                 ; the address, not the value
store.ptr [frame + 3p], &y
store.ptr [t + 0], &frame                  ; link

; variable assignment: no frame update
store.ptr [x], obj

; immediately before every return (including error returns): unlink
store.ptr [t + 0], load.ptr [frame + 0]
```

`-O0`보다 높은 수준에서는 프레임 배치가 끝난 뒤 슬롯 목록을 더 짧게 씁니다. 붙어 있는 word `n`개(셋 이상)의 연속은 항목 `(n << 1) | 1`과 그 뒤 첫 word의 주소로 씁니다(주소는 짝수이므로 아래 비트로 둘을 가립니다). 항목이 둘 이상인 목록은 읽기 전용 데이터로 옮깁니다. 이때 프레임은 `count`에 0을, 슬롯 하나에 정적 디스크립터 `{count: u16, entries: [u16; count]}`의 주소를 담습니다. 디스크립터의 항목은 슬롯의 항목과 같고, 주소 대신 프레임에서 그 word까지의 오프셋을 word 단위로 센 값의 두 배를 씁니다(부호 있는 16비트 값이고, 주소처럼 짝수입니다). 그러면 진입 코드는 목록 전체 대신 word 두 개만 저장합니다. 오프셋이 16비트에 들지 않는 목록은 프레임에 남깁니다. 디스크립터는 내용으로 이름을 붙이고, 링크는 같은 것을 하나만 남깁니다.

```
; the same frame above -O0, its list in read-only data
; desc: u16 2, (x - frame) / p * 2, (y - frame) / p * 2
store.ptr [frame + 0], load.ptr [t + 0]
store.usize [frame + p], 0                 ; a descriptor follows
store.ptr [frame + 2p], &desc
store.ptr [t + 0], &frame
```

`t`는 [8.3.8](803-tls-slots.md#838-컴파일러-출력-형식)의 스레드 로컬 블록 `nabi_tls`의 주소입니다([8.7.7](807-code-generation.md#877-tls-접근)). shadow stack 머리는 이 블록의 오프셋 0 슬롯입니다. 에러 반환도 일반 `return`이므로([8.9.6](#896-try--catch--finally-값-기반-분기)) 프레임을 떼는 같은 코드를 거칩니다. 프레임은 위 예보다 늦게, 각 경로의 첫 safepoint 앞에서 이어도 됩니다([8.4.7](804-safepoint.md#847-safepoint-불변식)). 그때는 프레임을 이은 경로의 return에서만 프레임을 뗍니다. 프레임은 GC 참조를 쥐고 있고 safepoint에 닿을 수 있는 함수에만 필요합니다([8.4.2](804-safepoint.md#842-삽입-위치)). 이때 반환하지 않는 함수(panic)의 호출과, 할당도 block도 하지 않는 런타임 함수([8.5.2](805-extern-symbols.md#safepoint가-아닌-함수))의 호출은 세지 않습니다.

## 8.9.13. 인덱싱

[인덱싱과 슬라이싱](../003-expressions/306-indexing-and-slicing.md)의 `arr[i]` 예입니다.

```nabi
arr[5]
```

이 표현식은 표준 라이브러리 함수 `Index::index(arr, 5)` 호출이 되고, 일반 함수 호출과 똑같이 낮춥니다.

`Index` 호출이 인라인되면(`-O1` 이상) 결과는 경계 검사와 load 하나입니다.

```
if 5 >= arr.len then bb_panic else bb_ok
bb_ok:
    val = load.T [arr.data + 5 * size_of(T)]
```

이 결과는 제네릭 단형화와 인라인에서 나옵니다.

## 8.9.14. 종합: hot path의 모습

대표적인 hot path인 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) alloc과 필드 쓰기의 비용은 다음과 같습니다.

- TLAB 커서 검사: load \* 2 / sub / 비교와 분기
- 커서 갱신: store (명령어 1개)
- 헤더 채우기: store \* 2 (type_id, gc_word)
- 필드 채우기: store \* N (필드 수)
- write barrier (필요 시): load / and / 비교와 분기, + (slow) call
- safepoint 폴링 (필요 시): load / 비교와 분기

slow path는 본문 밖에 따로 놓습니다. 작은 객체를 할당하고 채우는 데는 명령어가 대략 10~20개 듭니다. `-O0`에서는 값마다 스택 슬롯을 한 번 더 거치고, `-O0`보다 높으면 들어가는 값은 레지스터에 머뭅니다.
