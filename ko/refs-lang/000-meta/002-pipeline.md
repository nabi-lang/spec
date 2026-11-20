# 0.2. 컴파일 파이프라인

이 사양은 Nabi 컴파일러를 `nabic`라고 부릅니다. `nabic`는 소스에서 링크된 바이너리까지 파이프라인 전체를 한 프로세스 안에서 실행합니다. 기계어는 자체 코드 생성기로 만들고, 링크는 자체 링커로 합니다.

## 0.2.1. 단계

전체 흐름은 다음과 같습니다.

```
  source code
        |
        v
   nabi-hir         (High-level IR: pre type inference, user code structure preserved)
        |
        v
   nabi-mir         (Mid-level IR: monomorphized / operators desugared, all types fixed)
        |
        v
   machine code     (Nabi's code generator: optimized at -O1 and above)
        |
        v
   object -> linked by nabic's linker
```

> nabi-hir는 고수준 IR입니다. 타입 추론 이전 단계이며 사용자 코드 구조를 그대로 유지합니다. nabi-mir는 중간 IR입니다. 단형화와 연산자 풀어 쓰기(desugaring)가 끝나고 모든 타입이 확정된 형태입니다.
> 코드 생성기는 nabi-mir를 타깃용 기계어와 object 파일로 바꿉니다. 링커는 object들로 실행 파일을 만듭니다.

## 0.2.2. 단방향 가시성

각 단계는 바로 앞 단계만 봅니다. `nabi-hir` 생성기는 소스만 참조합니다. `nabi-mir` 생성기는 `nabi-hir`만 참조합니다. 코드 생성기는 `nabi-mir`만 참조합니다.

그러므로 뒤 단계는 앞 단계의 결과를 다시 참고하지 않습니다. 사양 문서도 같은 규칙을 따릅니다. 번호가 작은 파일은 번호가 큰 파일을 참조하지 않습니다.

## 0.2.3. 자기 완결적 toolchain

`nabic`는 외부 컴파일러, 어셈블러, 링커를 실행하지 않고, 옆에 따로 설치해야 하는 도구도 없습니다. 예외는 사용자가 고르는 `NABI_CC` 하나입니다. `NABI_CC`로 C 컴파일러 드라이버를 지정하면 `nabic`의 링커 대신 그 드라이버가 마지막 링크를 합니다([9.1](../009-tools/901-nabic-cli.md)). 중간 형태는 모두 Nabi 자신의 것입니다. nabi-hir, nabi-mir, 그리고 nabi-mir 아래의 코드 생성기 저수준 형태가 그렇습니다. 코드 생성기와 링커는 Nabi의 런타임 ABI([8](../008-abi/README.md))와 그 object 형식만을 위해 만들었습니다.

그래서 Nabi의 고수준 의미를 기계 연산으로 낮추는 책임은 컴파일러, 특히 코드 생성 단계에 있습니다([8.7](../008-abi/807-code-generation.md)).

## 0.2.4. 코드 생성

`nabic`의 코드 생성기는 하나이며, 모든 최적화 수준과 REPL 셀이 씁니다. 코드 생성기는 nabi-mir(`-O0`보다 높으면 최적화한 것)를 받아 런타임 ABI([8](../008-abi/README.md))를 따르는 Mach-O, ELF, COFF, wasm object를 씁니다. 그래서 서로 다른 수준으로 빌드한 object도 함께 링크할 수 있습니다.

링커도 `nabic`의 일부입니다. Mach-O 실행 파일(ad-hoc 코드 서명, 디버그 정보용 `.dSYM` 번들 포함), ELF 실행 파일, COFF / PE 실행 파일(`nabic`가 만드는 import library, 디버그 정보용 PDB 포함), wasm 모듈을 만들고, `wasm32-wasip2` 빌드는 wasm 모듈을 다시 WASI preview 2 component로 만듭니다. 크로스 컴파일도 내장되어 있습니다. `--target <triple>`는 어느 호스트에서나 동작합니다. toolchain이 지원 대상 Linux용 C 라이브러리(musl, glibc 링크 스텁)를 함께 담고 있으므로 Linux 대상의 Nabi 코드를 빌드하는 데는 sysroot가 필요 없습니다. C 코드는 toolchain의 C 컴파일러 `nabicc`가 같은 코드 생성기와 링커로 컴파일합니다. `nabicc`는 toolchain이 그 대상용으로 담은 C 헤더나 시스템의 C 헤더를 읽습니다([9.13](../009-tools/913-c-compiler.md)). Windows 대상은 toolchain에 든 module definition으로 만든 import library를 UCRT 위에서 링크합니다. MSVC 도구나 Windows SDK는 필요 없습니다. Apple 대상에는 macOS SDK(Xcode Command Line Tools)가 필요하고, 이 SDK는 재배포할 수 없습니다([9.2.14](../009-tools/902-aot-pipeline.md#9214-링킹실행-파일의-경우), [0.6](006-platform-targets.md)). `--emit=mir`는 최적화한 nabi-mir를, `--emit=obj`는 object를 내보냅니다([9.9.1](../009-tools/909-debug-tools.md#991-중간-산출물-출력---emit)).
