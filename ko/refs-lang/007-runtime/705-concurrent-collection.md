# 7.5. 동시 mark

full 수집은 mark 단계를 mutator와 동시에 실행할 수 있고, block sweep은 그 뒤에 mutator와 함께 합니다. minor 수집, 명시적 `nabi_gc_collect` 호출, evacuation은 언제나 stop-the-world입니다([7.3](703-gc-algorithm.md)).

> [!NOTE]
> 이 절은 비규범(참고용)입니다. 참조 런타임의 GC를 설명하며, 어느 릴리스에서든 바뀔 수 있습니다. 규범은 [8.1](../008-abi/801-object-header-abi.md)에서 ABI로 고정한 객체 레이아웃뿐입니다.

## 7.5.1. 언제

| `NABIR_GC_CONCURRENT` | full 수집                                    |
|----------------------|----------------------------------------------|
| `auto` (기본)       | old 객체가 32 MiB 이상이거나 일시 정지 목표가 있으면 동시 mark([bare 1.6.12](../../refs-standard-libraries/100-bare/106-gc-pin.md#1612-set_pause_goal-일시-정지-목표)) |
| `always`            | 언제나 동시 mark                              |
| `off`               | 언제나 STW                                    |

설정하지 않았거나 비어 있는 변수는 기본값을 씁니다. 값이 표에 없으면 런타임은 stderr에 경고 한 줄을 출력하고 기본값을 씁니다. 경고에는 변수 이름, 그 값, 허용되는 값이 들어갑니다.

## 7.5.2. 단계

1. **시작 pause (STW).** minor 수집을 한 번 하고, safepoint에서 루트의 스냅숏을 뜹니다. 전역 `MARKING` 플래그를 켭니다.
2. **동시 mark.** marker 스레드 하나가 스냅숏에서 출발해 mark합니다. mutator는 계속 실행되고, 할당이 marker를 앞지른 mutator는 직접 mark를 돕습니다(7.5.4).
3. **마지막 pause (STW).** SATB 버퍼를 비우고 루트를 다시 훑은 뒤 큰 객체를 sweep합니다. `MARKING`을 끕니다.
4. **지연 sweep.** block은 마지막 pause 뒤에 mutator가 실행되는 동안 sweep합니다. marker 스레드가 하고, 빈 block을 찾지 못한 할당 스레드가 힙을 늘리는 대신 block 몇 개를 sweep하고, 남은 것은 다음 pause가 마칩니다. 모든 pause는 먼저 이것부터 끝냅니다. sweep한 block은 바로 다시 나눠 줍니다. 다음 사이클의 예산을 넘는 빈 block은 그 전에 OS에 돌려줍니다.

## 7.5.3. SATB 배리어

객체 단위의 snapshot-at-the-beginning(Yuasa) 방식입니다. mark 중 어떤 객체에 처음 store하기 전에 그 순간 객체가 가진 참조를 기록하므로, mark를 시작할 때 도달 가능했던 것은 모두 mark된 채로 남습니다. 따로 barrier가 필요하지 않습니다. 세대 수집의 write barrier([7.4.2](704-barriers-and-roots.md#742-write-barrier-object-remembering))가 이미 객체에 처음 store할 때 slow path를 타고, 그 slow path가 기록을 맡습니다. 세대 수집 barrier는 store 대상 객체를 기억합니다. 저장되는 값을 기록하는 incremental update barrier가 아닙니다.

- `UNLOGGED` 객체에 처음 쓰면 slow path를 탑니다. 이때 `MARKING`이 켜져 있으면 slow path는 그 객체의 현재 참조를 모두 SATB 버퍼에 기록합니다(TLS `nabi_tls_satb_cursor` / `nabi_tls_satb_limit`, [8.3](../008-abi/803-tls-slots.md)).
- slow path는 기록하는 동안 헤더의 `SATB` 비트를 켭니다. 같은 객체에 경쟁하는 store는 그 비트가 꺼질 때까지 기다립니다.
- 큰(LOS) 객체는 512 B card 하나씩 기록합니다. card마다 처음 하는 store가 그 card가 가진 참조를 기록합니다(slow path는 슬롯과 저장하는 크기를 받습니다). card table 뒤에 card마다 두 비트를 두어 기록 중인지, 기록했는지 나타냅니다. store는 객체 전체가 아니라 많아야 card 하나를 기다립니다. 객체는 mark가 끝날 때까지 `UNLOGGED`로 남습니다.
- 그래서 컴파일러가 내보내는 barrier 코드는 동시 mark를 쓰든 안 쓰든 같습니다.

## 7.5.4. 할당과 수집

- mark 중의 할당은 line 단위로 black입니다. 그 line의 객체는 살아 있는 것으로 봅니다. mark 중에 할당된 객체는 mark가 끝나면 young 객체가 됩니다.
- mark 중에는 minor 수집을 하지 않습니다. mark에는 young 예산의 두 배까지 할당할 수 있습니다. 그것을 넘으면 할당하는 스레드가 남은 mark의 일부(객체 4096개)를 직접 하고(mark assist), 한 만큼 할당할 자리를 더 얻습니다. 얻는 양은 객체 하나에 young 예산을 직전 mark가 mark한 객체 수로 나눈 만큼입니다. 예산의 네 배를 넘거나 힙 상한([bare 1.6.14](../../refs-standard-libraries/100-bare/106-gc-pin.md#1614-set_heap_limit-힙-상한))에 이르러야 할당이 pause 안에서 mark를 끝냅니다. assist가 더 mark할 것이 없음을 알면 mark를 끝냅니다(짧은 마지막 pause).
- evacuation은 STW full 수집에서만 합니다. 지연 sweep이 힙의 단편화를 찾으면 다음 full 수집은 evacuation을 포함한 STW로 합니다(일시 정지 목표가 있으면 하지 않습니다). 이 판단에서 mark 중에 할당한 line은 차 있는 것으로 셉니다([7.3.1](703-gc-algorithm.md#731-immix-mark-region)). 그래서 할당만으로 다음 full 수집이 world를 멈추지는 않습니다.
- `no_collect` 구간([bare 1.6.13](../../refs-standard-libraries/100-bare/106-gc-pin.md#1613-no_collect-수집하지-않는-구간))이 열려 있는 동안 marker는 mark를 계속하되, 마지막 pause는 구간이 닫힐 때까지 미룹니다.
