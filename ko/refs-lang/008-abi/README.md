# 8. 런타임 ABI / lowering 개요

이 블록은 컴파일러(`nabic`)와 런타임(`libnabir.a`, Windows MSVC 타깃에서는 `nabir.lib`)이 함께 따르는 계약(contract)을 정의합니다. 두 쪽을 잇는 것은 이 사양 텍스트입니다. 둘이 공유하는 코드는 양쪽이 함께 빌드하는, 의존성 없는 크레이트 `nabir_abi`(`runtime/nabir_abi`) 하나입니다. 두 쪽이 정의 하나를 쓰도록 GC ABI 상수(`nabir_abi::gc`, [8.1](801-object-header-abi.md)), TLS 슬롯 배치(`nabir_abi::tls`, [8.3](803-tls-slots.md)), 모듈과 타입 디스크립터(`nabir_abi::module`, `nabir_abi::typedesc`, [8.12](812-module-abi.md), [8.13](813-type-descriptor.md)), 그리고 상수 평가가 런타임과 맞아야 하는 포맷 지정, 문자열, Unicode 루틴을 공유합니다.

## 목차

- [8.1. 객체 헤더 ABI](801-object-header-abi.md)
- [8.2. ScanTable 직렬화 포맷](802-scan-table.md)
- [8.3. mutator TLS 슬롯 레이아웃](803-tls-slots.md)
- [8.4. safepoint 프로토콜](804-safepoint.md)
- [8.5. Extern "C" 심볼](805-extern-symbols.md)
- [8.6. 호출 규약](806-calling-convention.md)
- [8.7. 코드 생성](807-code-generation.md)
- [8.8. Lowering: 데이터 타입과 표현식](808-lowering-types.md)
- [8.9. Lowering: 제어 흐름과 GC 통합](809-lowering-control.md)
- [8.10. Lowering: 코루틴과 제네릭](810-lowering-coroutine-generic.md)
- [8.11. 코드 생성기 계약](811-backend-contract.md)
- [8.12. 모듈 디스크립터 ABI](812-module-abi.md)
- [8.13. 타입 디스크립터 ABI](813-type-descriptor.md)

## 두 종류의 계약

### 1. 런타임 ABI

컴파일러가 생성한 코드를 `libnabir.a`와 링크해 함께 실행할 때 쓰는 인터페이스입니다. 객체의 메모리 레이아웃은 컴파일러가 만든 객체를 [GC](../000-meta/007-glossary.md#gc-garbage-collector)가 정확히 해석할 수 있게 정합니다. [extern "C" 심볼](../000-meta/007-glossary.md#extern-c-심볼)은 컴파일러가 호출할 수 있는 런타임 함수입니다. 각 심볼의 스레드 안전성 분류(`thread-safe` / `caller-thread-only` / `process-init` 등)는 [8.5.0 스레드 안전성 분류](805-extern-symbols.md#850-스레드-안전성-분류)에서 정의합니다. TLS 슬롯은 컴파일러가 내보낸 인라인 코드가 읽고 쓰는 스레드 로컬 자리입니다. 호출 규약(calling convention)은 함수를 호출할 때 레지스터와 스택을 어떻게 쓰는지 정합니다.

### 2. Lowering

[컴파일 파이프라인](../000-meta/002-pipeline.md)의 코드 생성 단계에서 Nabi의 고수준 의미를 기계어로 바꾸는 규칙입니다. 모든 값의 표현과, `tagged union`, `any` 슬롯, 패턴 매칭, try/catch, 코루틴, 제네릭이 어떤 load, store, 분기, 호출이 되는지를 정합니다.

## 이 사양 텍스트가 권위를 갖는 이유

포함해 쓰는 그 소스 파일 말고는 컴파일러와 런타임이 코드나 헤더 파일을 공유하지 않습니다. 두 쪽이 같은 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)를 따른다는 보장은 이 사양 텍스트와 통합 테스트의 종단 간(end-to-end) 검증에서 나옵니다.

이 모델은 Swift ABI Stability와 같은 접근입니다. 사양이 권위이고, 구현은 사양을 따릅니다.

## 안정성

심볼 맹글링([9.5](../009-tools/905-symbol-mangling.md))을 포함한 이 블록의 계약은 툴체인 버전의 일부입니다. 1.0 전에는 0.x.0 릴리스마다 ABI가 바뀔 수 있고, 0.x.y 릴리스는 ABI를 깨지 않습니다. 1.0부터는 minor / patch 릴리스에서 추가만 할 수 있고 호환성을 깨면 안 됩니다. major 릴리스에서는 호환성을 깰 수 있지만, 패키지 매니저가 알아챌 수 있게 깹니다.

미리 빌드한 패키지의 NABIHIR 헤더에는 그 패키지를 만든 `nabic`가 기록됩니다([9.4](../009-tools/904-nabihir-bytecode.md)). 다른 `nabic`가 만든 패키지를 쓰면 에러입니다. 에러는 다시 빌드하라고 안내합니다.
