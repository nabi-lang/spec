# 7.3. GC 알고리듬

Nabi의 GC는 [Immix](../000-meta/007-glossary.md#immix) mark-region[^immix] 위에 sticky mark-bit 세대 수집[^whippet]을 얹은 수집기입니다. minor 수집과 evacuation은 stop-the-world 방식입니다. 이때 mutator는 safepoint([8.4](../008-abi/804-safepoint.md))에서 멈춥니다. full 수집의 mark 단계는 동시에 실행할 수 있습니다([7.5](705-concurrent-collection.md)). 루트와 write barrier는 [7.4](704-barriers-and-roots.md)에서 정의합니다.

> [!NOTE]
> 이 절은 비규범(참고용)입니다. 참조 런타임의 GC를 설명하며, 어느 릴리스에서든 바뀔 수 있습니다. 규범은 [8.1](../008-abi/801-object-header-abi.md)에서 ABI로 고정한 객체 레이아웃뿐입니다.

## 7.3.1. Immix mark-region

### block과 line

```
block (32 KiB) = line (128 B) × 256
```

[block](../000-meta/007-glossary.md#block-블록)은 OS에서 받습니다(mmap). 객체는 line 경계를 넘을 수 있습니다. mark 단계는 객체와 함께 그 객체가 걸친 line을 표시합니다. sweep 단계는 표시되지 않은 line을 회수합니다. 산 객체 사이의 작은 틈도 line 단위로 다시 씁니다.

### sweep: block 분류

| 분류       | 상태                       | 처리                         |
|------------|----------------------------|------------------------------|
| clean      | 모든 line이 비어 있음      | free pool로                  |
| recyclable | 일부 line만 살아 있음      | 빈 line을 할당에 다시 씀     |
| full       | 거의 모든 line이 살아 있음 | 그대로 둠                    |

sweep은 비밀 버퍼(`WIPE`, [7.1.2](701-object-layout.md#712-메모리-공간))를 빼고는 죽은 객체를 0으로 지우지 않습니다. line은 할당기가 TLAB을 채울 때 0으로 지웁니다.

### opportunistic evacuation

full 수집은 단편화된 block의 산 객체를 다른 block으로 옮길 수 있습니다.

- 후보는 직전 sweep이 남긴 line 상태를 보고 고릅니다. 절반 이하로 찬 block이 후보입니다. 그 뒤에 나눠 준 line은 차 있는 것으로 셉니다. 그 객체가 살아 있는지 아직 모르기 때문입니다. 단편화가 임계값을 넘었을 때만 옮깁니다.
- 옮긴 객체의 옛 자리에는 forwarding pointer를 남깁니다. 같은 수집 안에서 모든 정밀 참조를 새 위치로 고칩니다.
- 다음 객체는 옮기지 않습니다. 루트가 직접 가리키는 객체([7.4.1](704-barriers-and-roots.md#루트가-가리키는-객체는-옮기지-않음)), 모호한 word(`AMBIGUOUS`: type id `2` buffer, untagged `union`)가 가리키는 객체, `PINNED` / `NOMOVE` 객체, LOS 객체.
- 다음 사이클의 할당 예산을 넘는 빈 block은 OS에 돌려줍니다.
- 테스트용 설정 `NABIR_GC_EVAC=always`를 쓰면 full 수집마다 evacuation을 합니다([9.9.8](../009-tools/909-debug-tools.md#998-메모리--gc-도구)).

## 7.3.2. sticky mark-bit 세대 수집

nursery 공간은 따로 없습니다. 세대는 헤더의 `STICKY` 비트([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags오프셋-4-u32))로 나눕니다. 수집에서 살아남은 객체는 `STICKY` 비트가 켜지고 old 객체가 됩니다. 객체는 제자리에 머물고, 승격(promotion)하려고 복사하지 않습니다.

| 수집          | mark 출발점                                      | 범위                                          |
|---------------|--------------------------------------------------|-----------------------------------------------|
| minor (young) | 루트, remembered set, dirty card ([7.4.2](704-barriers-and-roots.md#742-write-barrier-object-remembering)) | old 객체 안으로 들어가지 않음. sweep은 지난 수집 뒤 나눠 준 block만(동시 mark 뒤라면 그 mark가 line을 나눠 준 block도) |
| full (old)    | 루트                                             | `STICKY`를 모두 지우고 전체 mark / sweep      |

- young line은 객체 조회가 닿은 block에서만 파싱합니다.
- minor 수집의 장부 작업은 힙이 아니라 바뀐 것만큼 듭니다. 빈 line이 있는 block의 목록은 수집에서 다음 수집으로 이어지고, 그 사이에 나눠 준 block만 다시 sweep하고 목록에 올립니다. dirty card는 word(card 8개) 단위로 훑습니다.
- full 수집은 두 경우에 일어납니다. old 객체의 바이트 수가 `max(4 MiB, 직전 full 수집 뒤 live × 2)`에 이를 때와 `nabi_gc_collect`를 호출할 때입니다.

## 7.3.3. mark의 절차

1. 루트([7.4.1](704-barriers-and-roots.md#741-루트-추적-컴파일러가-직접-내보내는-shadow-stack))에서 시작합니다.
2. 도달한 객체마다 헤더의 `MARK`를 켜고, 객체가 걸친 line을 표시하고, `type_id`의 [ScanTable](../008-abi/802-scan-table.md)로 자식을 따라갑니다.
3. 큐가 비면 끝납니다.

stop-the-world 일시 정지에서는 큐에 나눌 만큼 쌓이면 mark를 여러 스레드에서 합니다(도우미 스레드 `NABIR_GC_WORKERS`개, [9.9.8](../009-tools/909-debug-tools.md#998-메모리--gc-도구)). 스레드마다 자기 스택에서 mark하고 공유 풀로 일을 주고받습니다. `MARK`는 atomic `fetch_or`로 켜므로 객체마다 한 번만 훑고, block의 young line은 한 스레드가 파싱하는 동안 다른 스레드가 기다립니다. 동시 mark([7.5](705-concurrent-collection.md))는 marker 스레드와 그것을 돕는 스레드에서 돕니다.

ScanTable은 `nabi_gc_register_type`([8.5.1](../008-abi/805-extern-symbols.md#851-gc-영역))으로 등록합니다.

## 7.3.4. migration (핫 리로드)

`nabi dev`([9.14](../009-tools/914-hot-reload.md)) 아래에서 핫 패키지의 새 버전은 타입의 레이아웃을 바꿀 수 있습니다. 그 객체는 migration으로 다시 씁니다. migration은 full stop-the-world 수집이고, mark와 sweep(있으면 evacuation도) 뒤의 마지막 단계에서 바뀐 타입의 산 객체를 모두 다시 씁니다.

- 대상: 핫 빌드는 핫 타입을 값으로 담는 box와 buffer마다 자기 type id를 주므로, type id가 타입과 버전을 가리킵니다. 영향을 받는 타입은 바뀐 타입과, 그 타입을 값으로 담는 모든 타입입니다.
- 방법: 남은 필드는 이름(또는 `#[renamed_from]`)으로 복사하고, 추가된 필드는 일시 정지 전에 기본값이 만든 값을 받고, 없어진 필드는 버리고, enum variant는 이름으로 맞춥니다([9.14.3](../009-tools/914-hot-reload.md#9143-바꿀-수-있는-것)). 새 레이아웃이 들어가는 객체는 제자리에서 다시 씁니다. 들어가지 않는 객체는 새 객체로 복사하고 evacuation처럼 forwarding하며, 그 객체를 가리키는 힙 참조를 모두 고칩니다. buffer 안을 가리키는 내부 참조와 weak cell도 고칩니다. 루트 word는 고치지 않습니다.
- 먼저 dry pass를 합니다. 다음 경우에는 migration이 아무것도 바꾸지 않고 변경이 기다립니다. 루트가 가리키거나 `PINNED` / `NOMOVE`인 바뀐 타입의 객체가 커져야 할 때, 모호한 word가 옮겨질 객체의 중간을 가리킬 때, `any`가 바뀐 타입의 값을 인라인으로 담을 때, 옛 코드의 클로저나 코루틴, 함수 값이 바뀐 타입을 담거나 쓸 때, 산 enum 값이 새 버전에서 없어진 variant를 담을 때, no-collect 영역이 열려 있을 때입니다.
- GC 통계에서 migration은 full 수집으로 셉니다.

## 참고 문헌

[^immix]: Blackburn & McKinley, _Immix: A Mark-Region Garbage Collector with Space Efficiency, Fast Collection, and Mutator Performance_, PLDI 2008.

[^whippet]: Wingo, Whippet: a sub-toolkit for embeddable garbage collection. <https://wingolog.org/tags/whippet>
