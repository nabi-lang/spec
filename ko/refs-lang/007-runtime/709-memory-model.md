# 7.9. 메모리 모델

이 절은 다른 스레드가 쓴 메모리를 한 스레드가 무엇으로 볼 수 있는지 정합니다. Nabi는 C++20(ISO/IEC 14882:2020, [intro.multithread]와 [atomics.order])의 메모리 모델을 따르며, Rust도 같은 모델을 씁니다. 아래 용어는 C++20의 뜻 그대로입니다. 이 절은 그 위에 어떤 라이브러리 연산이 동기화하는지, 안전한 Nabi가 무엇을 보장하는지를 덧붙입니다. 7.1~7.5와 달리 이 절은 규범입니다.

여기서 "스레드"는 OS 스레드입니다. task([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md))는 worker 스레드에서 돌고, 중단 지점마다 다른 worker에서 재개될 수 있습니다. 스케줄러가 task 자신의 단계 순서를 지키므로, 한 task 안의 코드는 어디서 재개되든 자기가 앞서 쓴 것을 봅니다. 두 task 사이에는 두 스레드 사이와 같은 아래 규칙이 적용됩니다.

## 7.9.1. 관계

- **sequenced before**: 한 스레드 안에서 평가 순서([3. 표현식](../003-expressions/README.md))로 앞서는 평가.
- **수정 순서(modification order)**: 원자 객체마다 그 객체에 대한 모든 수정이 하나의 전순서를 이루며, 모든 스레드가 이를 일관되게 관찰합니다(C++20 coherence 규칙).
- **synchronizes with**: 원자 객체에 대한 release 연산은, 같은 객체에서 그 연산이 쓴 값이나 그 release sequence의 이후 값을 읽는 acquire 연산과 동기화합니다. 7.9.3의 라이브러리 연산도 거기 적힌 대로 동기화합니다.
- **happens before**: sequenced before와 synchronizes with의 추이적 폐포(C++20의 simply-happens-before). Nabi에는 `consume` 순서가 없으므로 두 개념은 같습니다.

원자적이지 않은 읽기는, 그 위치에 대한 쓰기 가운데 그 읽기보다 happens before인 마지막 쓰기를 봅니다. 단, 그 읽기와 순서가 정해지지 않은 다른 쓰기가 없어야 합니다.

## 7.9.2. 원자 순서

`Atomic<T>`([stdlib 2.16.2](../../refs-standard-libraries/200-core/216-sync.md#2162-atomict-lock-free-원자-타입))는 C++20의 다섯 순서를 C++20의 뜻 그대로 받습니다.

| 순서      | C++20                        | 뜻 |
|-----------|------------------------------|----|
| `Relaxed` | `memory_order_relaxed`       | 원자적이며 수정 순서에 들어감. 동기화 없음. |
| `Acquire` | `memory_order_acquire`       | 읽은 값을 쓴 release와 동기화하는 load(또는 read-modify-write의 load 쪽). |
| `Release` | `memory_order_release`       | release sequence를 시작하는 store(또는 read-modify-write의 store 쪽). |
| `AcqRel`  | `memory_order_acq_rel`       | 둘 다인 read-modify-write. |
| `SeqCst`  | `memory_order_seq_cst`       | load는 acquire, store는 release이며, 모든 `SeqCst` 연산의 단일 전순서에 들어감. 기본값. |

실패한 compare-exchange는 실패 순서를 가진 load입니다. 연산마다 순서 타입이 그 연산이 받을 수 있는 순서만 담으므로(2.16.2) `consume`과 acquire store는 없습니다. 한 객체에 대한 원자 연산은 순서와 관계없이 찢어지지 않으며 서로 데이터 경쟁을 이루지 않습니다.

Nabi에는 단독 fence가 없습니다. 모든 순서는 원자 연산에 붙습니다. `volatile` 접근([stdlib 1.5](../../refs-standard-libraries/100-bare/105-volatile.md))은 원자적이지 않으며 스레드 사이의 순서를 정하지 않습니다.

## 7.9.3. 동기화하는 연산

다음 각 쌍에서 앞 연산은 뒤 연산과 동기화합니다. 그래서 앞 연산보다 sequenced before인 모든 것은 뒤 연산보다 sequenced after인 모든 것보다 happens before입니다.

| 앞 | 뒤 |
|----|----|
| `Mutex` 잠금 해제(가드 해제) | 그 `Mutex`의 다음 성공한 잠금(`lock`, `try_lock`, `lock_async`) |
| `RwLock` 쓰기 잠금 해제 | 다음 `read` / `write` 획득 |
| `RwLock` 읽기 잠금 해제 | 다음 `write` 획득 |
| `Condvar::wait`의 잠금 해제 | `Mutex` 잠금 해제와 같음. 다시 잠그는 것은 `Mutex` 잠금과 같음 |
| `Once::call_once`의 `f`가 돌아옴 | 그 `Once`의 모든 `call_once`가 돌아옴, `true`를 돌려주는 `is_completed` |
| 지연 초기화되는 `static`의 초기값 계산이 끝남 | 그 `static`의 모든 접근([5.3.9](../005-modules/503-static.md#539-초기값과-const)) |
| 채널의 `send` / `try_send` / `send_async` | 그 값을 받는 `recv`([stdlib 2.17](../../refs-standard-libraries/200-core/217-channel.md)) |
| `thread::spawn` / `Builder::spawn` | 새 스레드의 첫 단계 |
| 스레드의 마지막 단계 | 그 결과를 돌려주는 `join`(또는 `JoinHandle`의 `await`)([stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)) |
| `task::spawn` / `spawn_blocking` | task의 첫 단계 |
| task의 마지막 단계 | 결과를 돌려주는 그 `Task`의 `await`([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)) |
| `par` 호출이나 `par::scope`의 시작 | 각 작업의 첫 단계 |
| 각 `par` 작업의 마지막 단계 | `par` 호출의 반환, 또는 scope의 끝([stdlib 2.19](../../refs-standard-libraries/200-core/219-par.md)) |

참조 라이브러리에서 잠금, `Once`, 채널은 `SeqCst` 원자 연산으로 만들어져 있으므로, 위 순서는 `SeqCst` 전순서에도 들어갑니다. `Atomic<i32>::wait` / `wake`는 막고 깨울 뿐입니다. 그 자체로는 아무것도 동기화하지 않으며, `wait`가 비교하는 값은 그 원자 객체 자신의 순서로 읽습니다.

수집기, 그 write barrier([7.4.2](704-barriers-and-roots.md#742-write-barrier-object-remembering)), stop-the-world 정지는 수집기와만 동기화합니다. 하드웨어 fence를 우연히 거치더라도 사용자 코드에는 happens-before 간선을 주지 않습니다.

## 7.9.4. 데이터 경쟁

같은 메모리 위치에 대한 두 접근은, 적어도 하나가 쓰기이고 둘 다 원자적이지는 않을 때 **충돌**합니다. 충돌하는 두 접근이 서로 다른 스레드에서 일어나고 어느 쪽도 다른 쪽보다 happens before가 아니면, 그 프로그램에는 **데이터 경쟁**이 있습니다. 데이터 경쟁은 미정의 동작입니다([4.15.8](../004-functions/415-unsafe.md#4158-미정의-동작)).

데이터 경쟁은 `unsafe`가 들여보낸 코드를 거쳐서만 생깁니다. 안전한 Nabi에는 데이터 경쟁이 없습니다. 이유는 다음과 같습니다.

- `static`은 모든 스레드에서 닿으므로 타입이 `Sync`여야 합니다([5.3.1](../005-modules/503-static.md#531-기본-형태)). `static mut`은 `extern "C"` 블록에만 있고, 건드리려면 `unsafe`가 필요합니다.
- 컬렉션(`List`, `Dict`, `StrBuf`, ...)은 `!Sync`이며, `Cell<T>`, 원시 포인터, 가드도 그렇습니다([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커)). 공유되는 가변 상태는 `Mutex`, `RwLock`, `Atomic`, 채널 안에서만 스레드를 건넙니다. 얼린 컬렉션(`xs.freeze()`, [stdlib 2.9.7b](../../refs-standard-libraries/200-core/209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s-읽기-전용-컬렉션))은 아무것도 쓰지 않으므로 `Sync`입니다. 그 저장 공간에는 바꿀 수 있는 어떤 핸들도 닿지 않습니다.
- 값을 다른 소유자에게 넘기면(`Mutex::new`, `send`, `task::spawn`, `[move x]`, 스레드의 결과, ...) 새 소유자는 그 값이 닿는 모든 컬렉션의 자기 복사본을 받고, 보낸 쪽이 그 뒤에 값을 쓰면 컴파일 오류입니다(2.13.9). spawn하는 클로저의 일반 캡처는 `Send + Sync`여야 합니다.
- `par`는 두 항목이 `Sync`가 아닌 한 컬렉션에 닿지 않는지 실행 시간에 확인하고, 그런 항목은 하나씩 차례로 실행합니다([stdlib 2.19.1](../../refs-standard-libraries/200-core/219-par.md#2191-핵심-api)). 항목이 뷰(`chunks_mut`)로 한 리스트에 쓰는 것은, 쓰는 부분에 다른 항목의 뷰나 슬라이스가 겹치지 않을 때뿐입니다.

데이터 경쟁을 가능하게 하므로 작성자가 주의해야 하는 것은 다음과 같습니다.

- `Ptr<T>`, `ManyPtr<T>`, `CPtr<T>`, `GcBuf<T>`를 거친 읽기와 쓰기([stdlib 1.1](../../refs-standard-libraries/100-bare/101-ptr.md), [stdlib 1.3](../../refs-standard-libraries/100-bare/103-mem.md));
- 공유를 실제로 보호하지 않는 타입에 붙인 `unsafe impl Send` / `unsafe impl Sync`([4.15.4](../004-functions/415-unsafe.md#4154-트레잇-안에서));
- 다른 것이 아직 닿는 값에 쓴 `Mutex::new_unshared`([stdlib 2.16.3](../../refs-standard-libraries/200-core/216-sync.md#2163-mutext-배타적-잠금));
- `unsafe` 안에서 건너뛴 넘기기 검사(2.13.9);
- `extern static mut`과 FFI로 닿는 외부 코드([5.9](../005-modules/509-extern-block.md)). `env::set_var`가 도는 동안 `getenv` / `setenv`를 부르는 C 코드도 여기 듭니다([stdlib 3.5](../../refs-standard-libraries/300-std/305-env.md)).

## 7.9.5. GC 참조

GC 참조는 정렬된 기계어 한 워드이며, 한 번의 접근으로 읽고 씁니다. 수집기는 이에 기대므로 반쯤 쓰인 참조를 만나지 않고, 훑는 참조 워드를 모두 객체나 null로 다룹니다. 그렇다고 경쟁하는 참조 접근이 정의되는 것은 아닙니다. 참조에 대한 데이터 경쟁도 다른 데이터 경쟁처럼 미정의 동작이며, `str`(포인터와 길이, 두 워드)이나 구조체는 경쟁하면 찢어질 수 있습니다.
