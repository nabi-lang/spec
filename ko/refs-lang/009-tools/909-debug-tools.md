# 9.9. 디버그 / 진단 도구

이 절은 `nabic`와 `nabi`의 디버그 / 진단 도구와 외부 도구 통합을 정의합니다.

## 9.9.1. 중간 산출물 출력: `--emit`

[AOT 빌드 파이프라인](902-aot-pipeline.md)의 단계별 결과를 덤프할 수 있습니다.

```
nabic build --emit=source        # 메타 확장과 desugar 뒤의 전체 AST 를 nabi 소스로
nabic build --emit=hir          # nabi-hir 텍스트 출력
nabic build --emit=mir           # nabi-mir 텍스트 출력 (-O0 보다 높으면 최적화한 것)
nabic build --emit=obj           # object 파일
nabic build --emit=asm           # 오브젝트의 어셈블리 소스
nabic build --emit=listing       # 기계어를 오프셋, 바이트와 함께 텍스트로
nabic build --emit=expansions    # 메타 확장 결과 (9.9.10)
nabic build --emit=c-header      # 패키지 export 의 C 헤더 (9.3.3)
```

`mir`은 코드 생성기가 컴파일하는 nabi-mir입니다. `-O0`보다 높은 수준에서는 최적화한 뒤의 것입니다([9.2.10](902-aot-pipeline.md#9210-최적화-수준)). `obj`는 코드 생성기가 쓰는 오브젝트입니다. `obj`의 기계어는 `objdump -d` 같은 디스어셈블러로 볼 수 있습니다.

`asm`은 오브젝트를 `clang -S`처럼 어셈블리 소스로 쓴 것입니다. 대상용으로 어셈블하면(`clang -c`, `llvm-mc -filetype=obj`) 코드 바이트가 같고, 같은 오프셋에 같은 종류의 재배치가 같은 곳을 가리키며, 심볼도 같은 오브젝트가 나오므로 `obj`와 똑같이 링크되고 실행됩니다. `nabic`가 쓴 오브젝트를 다시 읽어 대상의 GNU / LLVM 문법으로 씁니다(x86_64와 x86은 `.intel_syntax noprefix` 아래 Intel 문법). 섹션마다 오브젝트 형식이 붙이는 이름으로 `.section` 지시어(ELF `.text.<symbol>`, `.rodata.<symbol>`, Mach-O `__TEXT,__text`, COFF는 선택 방식과 대표 심볼을 단 `.text` COMDAT, 함수에 연관된 `.pdata`)와 `.p2align`을 씁니다. 심볼에는 바인딩과 가시성을 씁니다(`.globl`, `.weak`, `.hidden`, ELF의 `.type`과 `.size`, Mach-O의 `.private_extern`과 `.weak_definition`, COFF의 `.def`, powerpc64의 `.localentry`). 함수는 코드 생성기가 인코딩한 명령어이고, 분기 대상은 지역 레이블(`.LBB0_3`, Mach-O와 x86 Windows는 `LBB0_3`)이며, 재배치되는 필드는 어셈블러가 그 재배치를 내게 하는 식으로 씁니다(`call nabi_panic_at@PLT`, `[rip + x@GOTPCREL]`, `[rip + x@GOTTPOFF]`, `[rcx + x@SECREL32]`, `[ebx + s@GOTOFF]`, `adrp x0, s@PAGE` / `:lo12:s`, `x@TLVPPAGE`, `#:lower16:(s-(.+16))`, `.word x(GOT_PREL)+0x1c`, `%pcrel_hi(s)`와 `%pcrel_lo(.Lpcrel_hi0)`, `s@toc@ha`, `x@got@tprel@l`, `brasl %r14, f@PLT`, `x@INDNTPOFF` 등). 어셈블러가 명령어 텍스트에서 고르지 않을 인코딩은 문법이 허락하면 강제하고(긴 분기의 `{disp32}`, `{disp8}`, `{rex}`, 다른 형식의 `mov`인 `mov.s`), 그렇지 않으면 바이트로 쓰고 명령어는 주석에 둡니다(즉시값이 8비트에 들어가는 x86 프레임 설정 `sub rsp, imm32`). 데이터(문자열 리터럴, 정적 변수, GC 타입 기술자, 스캔 테이블, vtable)는 `.asciz`, `.byte`, `.zero`와 대상을 이름으로 가리키는 포인터 크기 지시어(`.quad`, `.long s-.`, `.rva`, `.secrel32`, `.secidx`)입니다. 언와인드 테이블은 어셈블러가 그것을 만들어 내는 지시어로, 각각 그것이 설명하는 명령어 뒤에 씁니다. ELF `.eh_frame`(어셈블러의 CIE는 `nabic`의 것과 같음)과, 어셈블러가 지시어가 설명하는 프레임 레코드와 레지스터 쌍에서 끌어내는 Mach-O arm64의 compact unwind는 `.cfi_startproc`, `.cfi_def_cfa_offset`, `.cfi_def_cfa_register`, `.cfi_offset`, `.cfi_endproc`이고, x64 COFF의 `.pdata` / `.xdata`는 `.seh_proc`, `.seh_pushreg`, `.seh_stackalloc`, `.seh_setframe`, `.seh_savereg`, `.seh_savexmm`, `.seh_endprologue`, `.seh_endproc`이며, 32비트 ARM의 `.ARM.exidx`는 `.fnstart`, `.save`, `.setfp`(또는 `.cantunwind`), `.fnend`입니다. 어셈블러가 만든 테이블은 같게 해석됩니다(`llvm-dwarfdump --eh-frame`, `llvm-readobj --unwind`, `llvm-objdump --unwind-info`). arm64 Windows는 `.pdata` / `.xdata`를 재배치를 단 데이터로 둡니다. `nabic`가 언와인드 코드를 풀어 쓰는 곳에서 어셈블러는 그것을 압축하기 때문입니다(압축된 `.pdata` 항목, 프롤로그의 코드를 함께 쓰는 에필로그). `-g`의 디버그 섹션(DWARF, CodeView)도 데이터이고, 32비트 ARM의 빌드 속성은 `.eabi_attribute` 지시어입니다. RISC-V 소스는 `.option arch`, `.option norvc`, `.option norelax`로 시작합니다. 16비트 압축 형식의 명령어(`-O0`보다 높은 수준)는 `.option rvc` 아래에 `c.` 니모닉으로 쓰고(`c.addi sp, -0x10`, `c.beqz a0, .LBB0_3`), 다음 32비트 명령어 앞에서 `.option norvc`로 돌아갑니다. ELF 플래그는 어셈블러 옵션에서 옵니다(`clang --target=riscv64-linux-gnu`, 또는 `llvm-mc -mattr=+m,+a,+f,+d,+c -target-abi=lp64d`). 어셈블러가 다르게 쓸 수 있는 것은 뜻이 같습니다. 지역 심볼 대신 그 섹션과 심볼의 오프셋을 가리키는 재배치, 어셈블러가 늘 더하는 섹션 심볼과 매핑 심볼(`$x`, `$d`), Mach-O `ltmp` 심볼, 심볼로 남는 RISC-V 지역 레이블, x86_64 `.eh_frame`의 `SHT_X86_64_UNWIND` 타입, 섹션 안에서 해석되는 Mach-O 함수의 자기 호출과 compact unwind 항목, 어떤 지시어로도 켜지지 않는 32비트 ARM의 hard-float ELF 플래그(빌드 속성이 같은 것을 말함), COFF weak external 기본 심볼의 이름(어셈블러가 접미사를 붙인 `.weak.<symbol>.default`), 언와인드 테이블의 배치(`nabic`가 하나를 함께 쓰는 곳에서 함수마다 하나씩인 x64 `UNWIND_INFO`, 짧은 형식으로 쓴 32비트 ARM의 `pop {r4-r11, lr}`)가 그렇습니다.

wasm32에서 소스는 LLVM의 WebAssembly 어셈블리이고, `clang --target=wasm32-wasi -c`나 `llvm-mc -triple=wasm32-wasi -filetype=obj`로 어셈블합니다. 명령어에 필요한 기능은 어떤 지시어로도 켜지지 않으므로 어셈블러 옵션에서 옵니다. clang에는 `-msimd128`(`wasm32-wasip1-threads`에서는 `-matomics`도), `llvm-mc`에는 `-mattr=+bulk-memory,+mutable-globals,+nontrapping-fptoint,+reference-types,+sign-ext,+simd128`(와 `+atomics`)을 줍니다. 소스는 쓰는 전역, 테이블, 모든 함수를 선언하고(`.globaltype __stack_pointer, i32`, `.tabletype __indirect_function_table, funcref`, `.functype f (i32, i64) -> (f64)`), 다른 이름으로 가져오는 함수에는 `.import_module`과 `.import_name`을 씁니다. 함수는 저마다 자기 섹션(`.section .text.<symbol>,"",@`)에 바인딩(`.globl`, `.weak`, `.hidden`, `.export_name`), `.functype`, `.local` 줄과 함께 있습니다. 명령어에서 `end`는 닫는 것의 이름으로(`end_block`, `end_loop`, `end_if`, `end_function`), 타입 없는 `select`는 피연산자의 타입으로(`i32.select`) 쓰고, 재배치되는 피연산자는 심볼로 씁니다(`call f`, `global.get __stack_pointer`, 데이터 주소나 함수의 테이블 인덱스인 `i32.const s`, `i32.const s@TLSREL`, `call_indirect __indirect_function_table, (i32) -> (i32)`). 데이터 세그먼트는 각각 자기 섹션(`.section .data.<symbol>,"",@`, 스레드 지역이면 `"T"`)이고, `.p2align`, 심볼(`.type s,@object`, `.size`), 포인터마다 `.int32 s`를 씁니다. 디버그 섹션과 `target_features`는 데이터이고, 코드 오프셋은 `.int32 f+0x2b`입니다. 어셈블러가 만든 오브젝트는 함수, 코드(여기서처럼 재배치되는 피연산자는 5바이트), 재배치, 데이터 세그먼트, 심볼, 사용자 정의 섹션이 같습니다. 어셈블러가 다르게 쓰는 것은 뜻이 같습니다. 함수와 타입 인덱스의 순서, 메모리와 테이블 가져오기의 초기 크기, 어셈블러가 `__indirect_function_table`에 붙이는 `NO_STRIP` 플래그, 그리고 `DATACOUNT` 섹션이 그렇습니다.

```
	.intel_syntax noprefix

	.section	.text._Y1645ec0a__square_,"ax",@progbits
	.p2align	4
	.globl	_Y1645ec0a__square_
	.type	_Y1645ec0a__square_,@function
	.size	_Y1645ec0a__square_, 79
_Y1645ec0a__square_:
	push	rbp
	mov	rbp, rsp
	.byte	0x48, 0x81, 0xec, 0x30, 0x00, 0x00, 0x00	# sub rsp, 0x30
	mov	dword ptr [rsp], edi
.LBB0_0:
	movsxd	rcx, dword ptr [rsp]
	mov.s	rax, rcx
	imul	eax, ecx
	{disp32} jo	.LBB0_1
	..
.LBB0_1:
	lea	rax, [rip + nabi.str.4454f63dcefead0a]
	..
	call	nabi_panic_at@PLT
```

`listing`은 그 기계어를 텍스트로 쓴 것입니다. 코드 생성기가 인코딩하는 명령어에서 바로 쓰므로 바이트는 `obj`의 것과 똑같습니다. 함수마다 심볼(`<name>:`)로 시작하고, 명령어 한 줄에 함수 시작으로부터의 오프셋, 바이트, 텍스트를 대상의 `llvm-objdump -dr`이 출력하는 형식으로 씁니다(x86_64와 x86은 Intel 문법). 코드 생성기의 레이블은 그것이 가리키는 명령어 앞에 오고(`.L3:`), 분기 대상은 레이블 이름으로 표시하며(`jne 0x1c <.L3>`), 재배치는 그 필드를 채우는 명령어 바로 뒤에 오브젝트 형식의 이름(`R_X86_64_PLT32`, `ARM64_RELOC_BRANCH26`, `IMAGE_REL_AMD64_REL32`, `R_386_PLT32`, `R_ARM_CALL`, `R_RISCV_CALL_PLT`, `R_PPC64_REL24`, `R_390_PLT32DBL`, `R_WASM_FUNCTION_INDEX_LEB` 등)으로 씁니다. riscv의 `R_RISCV_PCREL_LO12_I`는 그 `auipc`에 오브젝트가 붙인 레이블(`.Lpcrel_hi0`)을 가리키고, 16비트 명령어는 16진수 4자리와 그 명령어가 나타내는 32비트 명령어의 텍스트로 보입니다. wasm32에서는 오프셋을 함수의 심볼(본문 크기)부터 세고, `.local` 줄에 지역 변수를 나열하며, 호출과 전역은 오브젝트가 매긴 인덱스로 씁니다. 제어 흐름 주석은 objdump의 것(`# 1: down to label3`)이고, 레이블 번호는 오브젝트 전체에 걸쳐 이어집니다.

```
<_Y6514eeb1__square_>:
       0: 55                           	push	rbp
       1: 48 89 e5                     	mov	rbp, rsp
       4: 48 81 ec 10 00 00 00         	sub	rsp, 0x10
       b: 8b f7                        	mov	esi, edi
.L0:
       d: 48 63 c6                     	movsxd	rax, esi
      10: 48 8b f8                     	mov	rdi, rax
      13: 0f af f8                     	imul	edi, eax
      16: 0f 80 08 00 00 00            	jo	0x24 <.L1>
      ..
      3f: e8 00 00 00 00               	call	0x44 <_Y6514eeb1__square_+0x44>
		0000000000000040:  R_X86_64_PLT32	nabi_panic_at-0x4
```

산출물은 다음 위치에 씁니다.

```
target/debug/build/foo.ast.nabi
target/debug/build/foo.hir
target/debug/build/foo.mir
target/debug/build/foo.o
target/debug/build/foo.s
target/debug/build/foo.lst
target/debug/build/foo.expanded.nabi
```

여러 단계를 한 번에 지정하려면 `--emit=hir,mir,obj`처럼 쉼표로 구분합니다. 종류가 하나면 `-o <path>`로 위치를 바꿀 수 있습니다. `--emit`을 주면 실행 파일이나 아카이브는 만들지 않습니다.

## 9.9.2. NABIHIR 검사

```
nabi inspect --dump path/to/foo.nabihir   # 또는 libfoo.a. 옆의 foo.nabihir 를 읽음
```

이 명령은 미리 빌드한 라이브러리의 [NABIHIR](904-nabihir-bytecode.md)을 사람이 읽을 수 있는 형식으로 출력합니다. 버전, producer, 메타 다음에 인터페이스 파일마다 `--- <경로>`와 그 텍스트가 이어집니다. 텍스트는 구체 함수 본문을 `;`로 잘라 낸 소스입니다. 그 뒤에 넘김 요약이 `--- summaries` 아래에, 호출 요약이 `--- calls` 아래에 함수마다 한 줄로 성립하는 사실과 함께 이어집니다([9.4.7](904-nabihir-bytecode.md#947-body)).

```
nabihir 1
producer: nabic 0.1.0 (81a1b80-18e41085f35ccdd9-18be644-6ae7cf0f.6bda9c9)
package: geom (hash: 3f8a1c2b)
source: /home/me/geom/src
gc types: 4096..4160
features: tls

--- lib.nabi
pub struct Point { pub x: f64; pub y: f64; }
pub fn distance(p1: Point, p2: Point) -> f64;
pub fn id<T>(x: T) -> T { x }
```

이 출력은 [NABIHIR](../000-meta/007-glossary.md#nabihir-nabi-hir-bytecode)의 정확성과 호환성을 검증할 때 쓸 수 있습니다.

## 9.9.3. 심볼 디맹글링

```
nabi demangle [-v] <symbol>...
nabi demangle _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190
nabi demangle < symbols.txt
```

이 명령은 [심볼 맹글링](905-symbol-mangling.md)으로 맹글링한 심볼을 소스에 쓰는 이름으로 한 줄에 하나씩 출력합니다. 표준 라이브러리 경로는 계층(`core` / `std` / `bare`)으로 시작합니다. 메서드(`<type>.<method>`, 또는 제네릭 타입 인스턴스의 `<type>__<method>`)는 `<type>::<method>`로, 트레잇으로 고른 메서드(`__by_<trait>`)는 `<path::type as trait>::<method>`로, 해시로 이름 붙은 impl 블록의 메서드는 `{impl}::<method>`로 보여 줍니다. 타입 해시는 뺍니다. 앞에 `_`가 하나 더 붙은 Mach-O 형식(`nm`이 출력하는 `__Y...`)과 심볼 뒤의 `.<suffix>`도 받습니다.

```
$ nabi demangle __Ya61f26f1_core_collections_dict_Dict_0_0grow_ce3f36e15e552a70
core::collections::dict::Dict::grow
```

`-v`를 주면 심볼마다 그 심볼이 담은 부분을 블록으로 출력하고, 블록은 빈 줄로 나눕니다.

```
package:  3f8a1c2b
path:     geom
name:     distance
generics: a7c91d8e44b2f190 (type tuple hash)
```

Nabi 심볼이 아닌 인자가 있으면 종료 코드는 `1`입니다. 심볼을 주지 않으면 `nabi demangle`은 필터입니다. 표준 입력을 표준 출력으로 옮기면서 그 안의 Nabi 심볼을 모두 한 줄 이름으로 바꾸고 나머지는 그대로 둡니다. 그래서 프로파일러, `nm`, 크래시 출력을 파이프로 넘길 수 있습니다.

```
perf report --stdio | nabi demangle
nm target/debug/my_binary | nabi demangle
```

## 9.9.4. 진단 / 에러 메시지

### 표준 형식

진단 메시지는 다음 표준 형식으로 출력합니다.

```
error[E0027]: cannot find value `count` in this scope
  --> src/main.nabi:5:13
   |
 5 |     let x = count + 1;
   |             ^^^^^
   |
   = help: did you mean `counter`?
```

이 형식은 [소스 파일](../001-lexical/101-source-files.md)에서 정한 파일, 줄, 열 단위의 진단 위치를 씁니다.

머리줄은 `<severity>[<code>]: <message>`이고 `--> path:line:col`이 주 위치입니다. 주 span에는 `^`로, 보조 span에는 `-`로 밑줄을 긋습니다. 다른 파일의 보조 span은 `::: path:line:col`로 시작합니다. 끝의 `= note:` 줄은 맥락을, `= help:` 줄은 고치는 방법을 덧붙입니다. 찾지 못한 이름이 다른 곳에 항목으로 있으면 help는 그 항목을 가져오는 `use` 줄을 알립니다. 줄은 최대 세 개이고, 패키지 자신의 항목, 표준 라이브러리, 의존 패키지 순서로 고릅니다. 패키지에 그런 항목이 하나뿐이고 그 항목이 비공개이면 쓰는 곳까지 닿는 공개 범위와 그 `use`를 알립니다. 패키지 자신의 항목은 패키지 루트부터의 경로로 가져옵니다(``add `use pkg::geo::Point;` ``, `` `geo::secret` has that name, but it is private to module `geo`: write `pub(pkg)` before it, so its whole package sees it, and add `use pkg::geo::secret;` ``).

주 span은 잘못된 코드를 가리킵니다. 타입이 맞지 않는 값이면 그 값을 담은 문장이 아니라 값 자체를 가리킵니다. 메시지는 라이브러리 항목을 코드에 쓰는 이름으로 부릅니다(`core::marker::Send`가 아니라 `List::sort`, `Send`, `print`). 메시지는 이 사양의 절을 인용하지 않습니다. 대신 쓸 것은 메시지와 help 줄, `nabic --explain <code>`에 있습니다.

### JSON 형식(도구 통합)

도구와 통합할 때는 JSON 형식으로도 출력할 수 있습니다.

```
nabic build --message-format=json
```

진단 하나가 stdout의 JSON 객체 한 줄입니다. 사람용 형식의 진단은 출력하지 않고, stderr에는 진행 줄만 나옵니다.

```json
{"type":"error","code":"E0040","kind":"type","message":"...","spans":[{"file_name":"src/main.nabi","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"...","line_end":5,"col_end":15}],"notes":["..."],"fixes":[],"help":["..."],"lint":null,"schema":1}
```

| 필드       | 뜻                                                                 |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | 에러 코드, `E0040`([에러 코드](#에러-코드))                        |
| `kind`     | 코드가 속한 영역(`type`, `parse`, ..)                              |
| `message`  | 메시지                                                             |
| `spans`    | 위치 목록. 첫 원소가 주 위치(`is_primary: true`), 나머지는 보조 위치 |
| `notes`    | note 문자열 목록. help 항목은 빠짐                                 |
| `fixes`    | 코드를 고치는 편집 목록. 각 편집은 파일 하나의 span에 그 자리를 바꿀 `text`와 `title`을 더한 것(아래). 비어 있을 때가 많음 |
| `help`     | help 항목(사람용 형식의 `= help:` 줄)에서 `help: ` 접두어를 뗀 것  |
| `lint`     | 경고나 린트 에러가 속한 린트(`unused_variables`). `#[allow(..)]`와 `-A`가 받는 이름([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)). 나머지는 `null` |
| `schema`   | 이 형식의 버전. 지금은 `1`                                         |

span에는 `file_name`, `line_start`, `col_start`, `byte_start`, `byte_end`, `is_primary`, `label`, `line_end`, `col_end`가 있습니다. `line_end` / `col_end`는 span이 끝나는 위치(마지막 문자 바로 다음)이고, `line_start` / `col_start`처럼 코드 포인트로 셉니다. 소스 텍스트가 없으면 `null`입니다. 위치 없이 파일만 아는 진단은 `{"file_name": .., "is_primary": true}`만 둡니다. fix에는 `file_name`, `line_start`, `col_start`, `byte_start`, `byte_end`, `text`, `title`(``convert with `as i32` ``), `line_end`, `col_end`가 있습니다. LSP 서버는 이것을 빠른 수정으로 내놓습니다([9.9.13](#9913-lsp-서버)).

같은 `schema` 안에서는 필드를 더하기만 하고, 빼거나 이름이나 뜻을 바꾸지 않습니다. 그러니 읽는 쪽은 모르는 필드를 무시해야 합니다. 테스트, 벤치, `nabi deps` 레코드([9.9.12](#9912-ci--테스트-통합), [9.10.4](910-nabi-frontend.md#9104-의존성-nabi-add--remove--update--deps))도 각자의 `schema`로 같은 규칙을 따르며, 그 번호는 지금 `1`입니다.

IDE와 CI 시스템은 이 형식으로 진단을 기계적으로 읽을 수 있습니다.

### 에러 코드

에러는 종류마다 자기 코드가 있습니다. 코드는 `E`와 네 자리 숫자입니다. 타입이 맞지 않으면 `E0040`, 이름을 찾지 못하면 `E0027`입니다. 번호는 `E0001`부터 빈틈 없이, 영역이 아니라 추가된 순서대로 붙고, 영역은 `kind` 필드가 알려 줍니다. 린트([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-린트-제어)) 진단은 컴파일러의 어느 단계가 냈든 린트 코드 `E0131`을 받고, 린트 이름은 note와 `lint`에 적습니다.

`nabic --explain <code>`(`nabi --explain`도 같음)는 코드의 설명을 출력합니다. 제목, 뜻, 대신 쓸 코드를 알려 줍니다([9.1.5](901-nabic-cli.md#915-전역-옵션)). `E0040`, `e0040`, `40`은 같은 코드입니다. 없는 코드는 사용법 에러(종료 코드 2)입니다. 문서의 에러 페이지도 같은 내용이며, 코드마다 한 페이지입니다.

제목은 `nabic --explain`이 출력하는 그대로 적습니다.

| 코드    | `kind`        | 제목                         |
|---------|---------------|------------------------------|
| `E0001` | `lexical` | unexpected character in source |
| `E0002` | `lexical` | bidirectional control character in source |
| `E0003` | `lexical` | malformed numeric literal |
| `E0004` | `lexical` | literal out of range for its type |
| `E0005` | `lexical` | unterminated literal or comment |
| `E0006` | `lexical` | invalid escape in a literal |
| `E0007` | `lexical` | a character literal holds one character |
| `E0008` | `lexical` | malformed f-string |
| `E0009` | `lexical` | name cannot be a raw identifier |
| `E0010` | `parse` | unexpected token |
| `E0011` | `parse` | nesting too deep |
| `E0012` | `parse` | syntax from another language |
| `E0013` | `parse` | struct literal in a block head |
| `E0014` | `parse` | modifiers out of order |
| `E0015` | `parse` | doc comment documents nothing |
| `E0016` | `parse` | visibility where it has no meaning |
| `E0017` | `parse` | item in the wrong place |
| `E0018` | `parse` | `return` with a value in top-level statements |
| `E0019` | `parse` | operators that need parentheses |
| `E0020` | `parse` | `return` in a trailing closure |
| `E0021` | `parse` | reserved word used as a name |
| `E0022` | `parse` | `let` without a value |
| `E0023` | `parse` | invalid function signature |
| `E0024` | `parse` | invalid generic parameters |
| `E0025` | `parse` | unsupported ABI |
| `E0026` | `resolve` | name defined more than once |
| `E0027` | `resolve` | cannot find value or function |
| `E0028` | `resolve` | cannot find type or trait |
| `E0029` | `resolve` | cannot find enum variant |
| `E0030` | `module` | unresolved import |
| `E0031` | `resolve` | ambiguous name |
| `E0032` | `module` | conflicting imports |
| `E0033` | `module` | invalid re-export |
| `E0034` | `visibility` | item is private |
| `E0035` | `module` | module file problem |
| `E0036` | `module` | invalid program entry point |
| `E0037` | `module` | invalid dependency |
| `E0038` | `resolve` | variable named like a nested function |
| `E0039` | `resolve` | nested item uses an outer local |
| `E0040` | `type` | mismatched types |
| `E0041` | `type` | branches of different types |
| `E0042` | `type` | type annotations needed |
| `E0043` | `type` | no such field |
| `E0044` | `type` | missing field in a literal |
| `E0045` | `type` | field given more than once |
| `E0046` | `type` | no such method or associated item |
| `E0047` | `type` | item used in the wrong form |
| `E0048` | `type` | wrong number of arguments |
| `E0049` | `type` | invalid named argument |
| `E0050` | `type` | `ref` parameter and argument mismatch |
| `E0051` | `type` | invalid spread argument |
| `E0052` | `type` | value is not callable |
| `E0053` | `type` | invalid cast |
| `E0054` | `type` | operator cannot be applied |
| `E0055` | `type` | literal fits several types |
| `E0056` | `type` | invalid indexing |
| `E0057` | `type` | invalid assignment |
| `E0058` | `type` | invalid closure capture |
| `E0059` | `type` | keyword outside its context |
| `E0060` | `type` | `let .. else` that does not diverge |
| `E0061` | `type` | operation needs `unsafe` |
| `E0062` | `type` | error not handled |
| `E0063` | `type` | thrown value is not an `Error` |
| `E0064` | `type` | optional operator on a value that is not optional |
| `E0065` | `type` | lock guard not released by `using` |
| `E0066` | `type` | `using` on a value without `Drop` |
| `E0067` | `type` | block statement leaves a value |
| `E0068` | `type` | value cannot be iterated |
| `E0069` | `type` | trait bound not satisfied |
| `E0070` | `type` | not a trait |
| `E0071` | `type` | trait used as a type |
| `E0072` | `type` | impl is missing a trait item |
| `E0073` | `type` | impl item is not in the trait |
| `E0074` | `type` | impl item does not match the trait |
| `E0075` | `type` | conflicting implementations |
| `E0076` | `type` | impl of a foreign trait for a foreign type |
| `E0077` | `type` | supertrait not implemented |
| `E0078` | `type` | impl not allowed |
| `E0079` | `type` | cannot evaluate at compile time |
| `E0080` | `type` | definition depends on itself |
| `E0081` | `type` | invalid const generic argument |
| `E0082` | `type` | wrong generic arguments |
| `E0083` | `type` | invalid enum value |
| `E0084` | `type` | invalid foreign-function use |
| `E0085` | `type` | value cannot cross threads |
| `E0086` | `type` | invalid union |
| `E0087` | `type` | ambiguous associated item |
| `E0088` | `type` | invalid format |
| `E0089` | `type` | invalid compiler built-in call |
| `E0090` | `type` | invalid coroutine use |
| `E0091` | `type` | invalid broadcast |
| `E0092` | `type` | `Drop::drop` called directly |
| `E0093` | `type` | generic instantiation too deep |
| `E0094` | `type` | invalid representation |
| `E0095` | `type` | type too large |
| `E0096` | `type` | `try` or `catch` that does nothing |
| `E0097` | `type` | NABION value of the wrong shape |
| `E0098` | `pattern` | non-exhaustive match |
| `E0099` | `pattern` | unreachable match arm |
| `E0100` | `pattern` | pattern does not fit the value's type |
| `E0101` | `pattern` | wrong number of fields in a pattern |
| `E0102` | `pattern` | or-pattern alternatives bind differently |
| `E0103` | `pattern` | binding looks like a variant |
| `E0104` | `pattern` | invalid rest pattern |
| `E0105` | `handoff` | value used after it was handed over |
| `E0106` | `handoff` | value used after its lock was released |
| `E0107` | `handoff` | shared value handed over |
| `E0108` | `meta` | cannot derive |
| `E0109` | `meta` | malformed attribute |
| `E0110` | `meta` | attribute on the wrong item |
| `E0111` | `meta` | invalid `#[lang]` item |
| `E0112` | `meta` | invalid operator declaration |
| `E0113` | `meta` | invalid `#[keyword]` declaration |
| `E0114` | `meta` | invalid literal handler |
| `E0115` | `meta` | invalid `#[intrinsic]` |
| `E0116` | `meta` | malformed macro definition |
| `E0117` | `meta` | no macro rule matches |
| `E0118` | `meta` | invalid meta template |
| `E0119` | `meta` | invalid meta block |
| `E0120` | `meta` | recursive expansion |
| `E0121` | `meta` | invalid `cfg` predicate |
| `E0122` | `meta` | file include failed |
| `E0123` | `meta` | registered twice |
| `E0124` | `meta` | meta name cannot be resolved |
| `E0125` | `other` | cannot read or write a file |
| `E0126` | `linkage` | link failed |
| `E0127` | `linkage` | external tool failed |
| `E0128` | `module` | invalid WIT |
| `E0129` | `unsupported` | not supported by this backend |
| `E0130` | `internal` | internal compiler error |
| `E0131` | `lint` | lint |
| `E0132` | `other` | invalid compiler option |
| `E0133` | `handoff` | resource used after it was moved or closed |
| `E0134` | `handoff` | resource not closed |
| `E0135` | `resolve` | trait of the method not in scope |
| `E0136` | `handoff` | thread-local value kept past `with` |
| `E0137` | `handoff` | list view used after a resize |
| `E0138` | `type` | `pub` function signature not written out |
| `E0139` | `handoff` | stream view used past its stream |
| `E0140` | `type` | assignment to a value that is not a place |
| `E0141` | `resolve` | method generic parameter named like its impl's |
| `E0142` | `type` | rows of an array literal differ in length |
| `E0143` | `type` | multi-axis index on a type indexed along one axis |
| `E0144` | `type` | integer raised to a negative power |
| `E0145` | `type` | named `some` type not defined by its module |
| `E0146` | `type` | named `some` type used as its concrete type |
| `E0147` | `lexical` | invisible character in code |
| `E0148` | `lexical` | character that only looks like a token |

`kind` 값은 `lexical`, `parse`, `resolve`, `visibility`, `module`, `type`, `pattern`, `handoff`(값의 소유와 수명. 넘긴 뒤의 사용, [stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync-스레드-경계-마커). 옮기거나 닫은 뒤에 쓰거나 닫지 않은 자원, [stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop-해제-시-정리). `with` 뒤에도 간직한 스레드 로컬 값. 스트림보다 오래 쓴 스트림 뷰, [stdlib 2.23.1](../../refs-standard-libraries/200-core/223-io.md#2231-read). 리스트 크기를 바꾼 뒤의 `SliceMut` 사용, [stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt-쓰기-가능한-뷰)), `meta`, `linkage`, `unsupported`, `internal`, `other`, `lint`입니다.

## 9.9.5. 로그 / 진행 표시

### 표준 출력

기본 빌드 출력은 다음과 같습니다.

```
   Compiling foo (path/to/foo)
    Building target/debug/libfoo.a
    Finished in 1.23s
```

진행 줄은 stderr에 나옵니다. 동사 목록은 [9.1.8](901-nabic-cli.md#918-출력-형식)에 있습니다.

### `--verbose` / `-v`

상세 모드에서는 단계별 시간을 함께 표시합니다.

```
         Std prebuilt (<path>)
   Compiling foo (path/to/foo)
   Stage: front end (134ms)
      Reused: 12 of 107 files parsed, 3 of 4 bodies lowered
    Building ./target/debug/foo
       Units 2 reused, 1 compiled
   Stage: codegen + link (812ms)
    Finished in 0.95s
```

각 단계의 시간이 나오므로 빌드 병목을 찾을 때 쓸 수 있습니다. 단계는 `front end`와 `codegen + link`(`--emit`이면 `emit <ext>`)입니다.

### `NABI_TIME_PASSES`

`NABI_TIME_PASSES=1`은 컴파일러 패스마다 걸린 실제 시간을 stderr에 출력합니다.

### `-vv`(매우 자세히)

`-v`의 출력에 더해 처리한 파일마다 `Reading <file>` 줄을 출력합니다. [특수화](../000-meta/007-glossary.md#specialize)한 (항목, 타입) 쌍마다 `Specialized <path>(sig)` 줄도 출력합니다.

## 9.9.6. 디버거 통합: gdb / lldb

### DWARF 디버그 정보

`debug` 프로필 빌드와 `-g`를 준 `release` 빌드([9.2.15](902-aot-pipeline.md#9215-산출물))는 다음과 같은 DWARF 디버그 정보를 내보냅니다. Windows 타깃은 같은 정보를 DWARF 대신 CodeView로 받습니다([아래](#codeview와-pdb-windows)).

- 함수의 `DW_AT_name`은 Nabi 경로(`geom::distance`)입니다. linkage name(맹글링된 심볼)은 두지 않습니다. 디버거가 `_Y` 맹글링을 모르기 때문입니다. 프로그램의 `main`을 부르는 래퍼([9.5.9](905-symbol-mangling.md#959-main-함수의-심볼))의 이름은 `nabi_main`입니다. 그래서 백트레이스에서 `main` 프레임은 프로그램의 것 하나뿐입니다.
- 줄 번호는 원본 .nabi 파일의 줄을 가리킵니다.
- 지역 변수는 원본 이름과 타입으로 보입니다. 원시 타입(정수, 부동소수, `bool`, `char`) 외에 struct(필드 이름. 제네릭 struct의 인스턴스는 타입 인자를 소스에 쓰는 대로 붙인 이름, `core::collections::list::List<i64>`), 튜플, `str`, `any`, 합 타입, 배열의 DWARF 타입을 냅니다.
- 합 타입과 enum은 DWARF variant part입니다. variant part에는 artificial discriminant 멤버가 하나 있고, 변형마다(데이터 없는 변형 포함) `DW_AT_discr_value`가 붙은 `DW_TAG_variant`가 하나씩 있습니다. 각 변형은 이름이 `<Sum>::<Variant>`인 struct를 담습니다. 디버거가 variant part를 읽도록 컴파일 단위는 `DW_LANG_Rust`를 씁니다(Nabi의 DWARF 언어 코드는 없음).
- `GcBuf<T>` 참조는 그 객체(헤더, 그다음 `T` 원소)를 가리키는 포인터로 기술합니다.
- `any` 값은 그 안에 저장된 타입의 variant part이고, discriminant는 저장된 태그입니다. discriminant는 artificial `i32`입니다. `throws` carrier의 `bool` 태그는 바이트입니다.
- 함수 값(`fn` 타입이나 클로저)은 그 타입 이름(`fn(i32) -> i32`)의 struct입니다. 멤버 `code`는 `DW_TAG_subroutine_type`을 가리키는 포인터이고, 그 매개변수는 환경, 그다음 함수 매개변수입니다. 멤버 `env`는 그 환경이고, 클로저의 타입을 알면 환경 객체를 가리키는 포인터입니다. 그래서 디버거가 대상 함수를 보여 줍니다. 참조로 캡처한 변수는 그 값으로 보입니다.
- 컴파일러가 만드는 함수(C `main`, static 초기화 가드)에도 subprogram이 있습니다. 컴파일러가 만든 지역 변수(이름이 `#`이나 `__`로 시작)는 빠집니다.
- 블록마다 lexical block scope를 냅니다. 그래서 가려진(shadowed) 이름은 해당 블록 안에서만 보입니다.
- `-O0`에서는 모든 지역 변수가 자기 스택 슬롯을 가지고, 위치는 그 슬롯입니다.
- `-O0`보다 높은 수준에서도 `-g`는 코드를 바꾸지 않습니다([9.2.10](902-aot-pipeline.md#9210-최적화-수준)). 디버그 정보가 최적화된 코드를 설명합니다. 변수의 위치는 코드 범위별 location list(`.debug_loc`)입니다. 값을 담은 레지스터, 스택 슬롯, 상수, 또는 제자리에서 만든 struct나 클로저라면 이들의 조각(`DW_OP_piece`)입니다. 값이 더는 없는 곳에서는 위치가 없고, 디버거는 이를 optimized out으로 보여 줍니다. safepoint를 넘어 GC 참조를 담는 변수는 늘 스택 슬롯에 있습니다. 그 슬롯이 컬렉션이 읽고 고치는 root입니다.
- `-O0`보다 높은 수준에서 인라인된 본문은 `DW_AT_inline`이 붙은 추상 `DW_TAG_subprogram`의 `DW_TAG_inlined_subroutine`(`DW_AT_call_file`, `DW_AT_call_line`, `DW_AT_call_column` 포함)이고, 피호출 함수의 매개변수와 지역 변수를 담습니다. 그래서 디버거는 그 안에서 멈추고 그것을 별도의 프레임으로 보여 줍니다. 인라인된 코드의 줄 행은 피호출 함수의 파일을 가리키고, 행은 줄이 바뀌는 곳에서만 문장(`is_stmt`)이며, 프롤로그 다음 첫 행에는 `prologue_end`가 붙습니다. Apple 타깃에서는 단위에 `DW_AT_APPLE_optimized`가 붙습니다. wasm32도 인라인된 본문과 변수를 같은 방식으로 설명합니다. `-O0`보다 높은 수준에서 wasm 지역(local)에 둔 변수는 그 지역이 값을 담는 코드 범위에 걸쳐 그 지역을 가리키는 location list(`DW_OP_WASM_location 0x0 <local>, DW_OP_stack_value`, struct라면 지역마다 조각 하나)를 갖고, 선형 메모리에 둔 변수는 그 슬롯이 값을 담는 동안 프레임 슬롯(`DW_OP_fbreg`, 프레임 베이스 자체도 wasm 지역)을 위치로 갖습니다. `-O0`에서는 모든 변수가 프레임 슬롯에 있습니다.

### CodeView와 PDB (Windows)

Windows 타깃에서는 같은 정보가 오브젝트마다 CodeView(C13)로 들어가고, 링커가 이를 실행 파일 옆의 PDB로 모읍니다.

- `.debug$S`는 함수마다 그 함수의 코드에 연결된 섹션에 함수의 `S_GPROC32`(이름은 Nabi 경로)와 `S_FRAMEPROC`, 매개변수와 지역 변수마다 프레임 포인터 기준 위치(`S_DEFRANGE_FRAMEPOINTER_REL_FULL_SCOPE`)를 단 `S_LOCAL`, 코드 범위가 하나인 렉시컬 블록의 `S_BLOCK32`, 함수의 줄 테이블을 담습니다. 그 밖의 블록의 지역 변수는 바깥 범위에 속합니다. 오브젝트의 소스 파일은 체크섬 테이블에 있습니다.
- `-O0`보다 높은 수준에서 변수의 위치는 코드 범위별로 `S_DEFRANGE_REGISTER`나 `S_DEFRANGE_FRAMEPOINTER_REL`로 주고, 필드가 흩어진 struct(레지스터, 메모리, 또는 둘 다)는 필드마다 레코드 하나를 둡니다. 레지스터에 있는 필드는 `S_DEFRANGE_SUBFIELD_REGISTER`, 메모리에 있는 필드는 필드 오프셋을 단 프레임 포인터 기준 `S_DEFRANGE_REGISTER_REL`입니다. 레지스터는 CodeView 번호로 가리킵니다. x64와 x86에서는 범용 레지스터와 XMM 레지스터, arm64에서는 `X0`-`X30`과 크기에 따라 `S`, `D`, `Q`입니다. 값이 정수, `bool`, `char` 상수 하나뿐인 변수는 `S_CONSTANT`입니다. 값이 부동소수점 상수 하나뿐인 변수는 그 스코프의 `S_LDATA32`이고, 값은 함수 코드와 함께 다니는 읽기 전용 데이터 섹션에 둡니다(dbghelp는 부동소수점 `S_CONSTANT`를 읽지 못합니다). 값이 어디에도 없는 `S_LOCAL`은 optimized out으로 표시합니다. Windows ABI가 참조로 넘기는 struct, 튜플, 배열 매개변수(x64는 1, 2, 4, 8바이트가 아닌 것, arm64는 16바이트를 넘는 것)는 모든 최적화 수준에서 매개변수 플래그 없는 `S_LOCAL`입니다. dbghelp는 그런 매개변수를 값을 가리키는 포인터로 읽지만, Nabi는 값 자체를 설명하기 때문입니다. 인라인된 본문은 피호출 함수의 `LF_FUNC_ID`를 가리키는 `S_INLINESITE`이고, 바이너리 주석이 그 코드를 줄에 대응시킵니다. 오브젝트에 하나뿐인 `DEBUG_S_INLINEELINES` 서브섹션(주 `.debug$S`에 있고, PDB에서는 모듈마다 하나입니다. dbghelp는 모듈의 첫 서브섹션만 읽습니다)이 각 피호출 함수의 파일과 줄을 주고, 함수는 피호출 함수 목록을 `S_INLINEES`로 담고, 함수 자신의 줄 테이블은 인라인된 코드에 호출한 줄을 줍니다. 인라인 사이트는 `S_BLOCK32` 안에 두지 않습니다.
- `.debug$T`는 타입을 담습니다. 원시 타입은 CodeView 기본 타입입니다. struct, 튜플, `str`, `any`, 함수 값은 필드 이름을 가진 `LF_STRUCTURE`입니다. 합 타입과 enum은 태그와, 변형마다 오프셋 0에 놓인 멤버 하나를 가진 구조체이고, 각 멤버는 값 전체에 걸친 `<Sum>::<Variant>`라는 구조체입니다. `GcBuf<T>`나 박스 참조는 그 객체를 가리키는 포인터입니다. 함수의 시그니처는 `LF_PROCEDURE`입니다.
- PDB(MSF 7.0)는 모든 오브젝트의 타입 레코드를 합친 것(같은 레코드는 하나), 오브젝트마다 심볼과 줄 테이블을 담은 모듈, global 심볼(프로시저 참조, 전역 데이터)과 public 심볼, 섹션 기여 정보와 섹션 맵을 담습니다. 자체 CodeView가 있는 오브젝트(런타임)도 같은 방식으로 들어갑니다. 실행 파일의 debug directory가 경로, GUID, age로 PDB를 가리킵니다.

dbghelp 기반 디버거(WinDbg와 cdb, Visual Studio 디버거)는 이름으로 함수를 찾고, 주소를 파일과 줄로 옮기고, 매개변수와 지역 변수를 값과 함께 보여 줍니다. 최적화된 코드에서는 인라인된 프레임(`k`. `bm <module>!<fn>`은 함수가 인라인된 곳마다 중단점을 겁니다)과, 레지스터에 있거나 레지스터와 메모리에 나뉘어 있거나 상수인 변수(`dv`, `dx`)도 보여 줍니다.

### gdb / lldb 사용

표준 gdb와 lldb 명령어가 Nabi 코드에서 그대로 동작합니다.

```
gdb -iex "$(nabic --print gdbinit)" ./target/debug/my_binary
(gdb) break main
(gdb) run
(gdb) print x
$1 = 42
(gdb) print xs
$2 = size=3 = {1, 2, 3}     # 포매터를 쓸 때(아래)
(gdb) bt
#0  main at src/main.nabi:5
...
```

### 포매터

toolchain은 표준 라이브러리 값을 값 그대로 보여 주는 lldb, gdb, natvis(Visual Studio, WinDbg, cdb) 포매터를 함께 배포합니다.

- `str`은 그 텍스트(`"hello"`)로 보여 줍니다.
- `List`, `Slice`, `Set`, `SortedSet`은 `size=N`과 원소 children으로, `Dict`와 `SortedDict`는 `size=N`과 키를 이름으로 한 값 children으로 보여 줍니다. 순서는 모두 순회 순서입니다(삽입 순서, 정렬된 것은 키 순서).
- `Array`는 `shape=[..]`와 행 우선 순서의 원소로 보여 주고, 원소 이름은 인덱스(`[1, 0]`)입니다.

gdb는 DWARF만으로 합, enum, `any`의 활성 변형만 보여 줍니다. lldb에는 그 단위 언어의 플러그인이 없어서 모든 변형을 나열합니다. lldb 포매터는 활성 변형만 보여 줍니다(변형 이름을 summary로, 그 필드를 children으로. 박스에 담긴 `any` 페이로드는 그 박스를 거쳐). 대입 전에 읽은 값(오래된 슬롯)은 길이가 그 값의 것일 수 없을 때 `<invalid>`로 보이고, children은 10000개까지 보여 줍니다.

| 디버거 | 파일 | 로드 |
|---|---|---|
| lldb | `lib/nabi/lldb/nabi.py` | `nabic --print lldbinit`이 `command script import` 줄을 출력 |
| gdb | `lib/nabi/gdb/nabi.py` | `nabic --print gdbinit`이 `source` 줄을 출력 |
| Visual Studio, WinDbg, cdb | `lib/nabi/natvis/nabi.natvis` | `nabic --print natvis`가 경로를 출력 |

두 Python 포매터는 컬렉션 레이아웃을 `lib/nabi/debug/layout.py`에서 읽고, 표준 라이브러리의 `/nabi/library` 소스 경로를 toolchain의 라이브러리 소스로 연결합니다. natvis 파일은 트리가 두 단계 이하인 `SortedDict` / `SortedSet`(항목 1023개까지)을 나열하고, 더 큰 것은 크기만 보여 줍니다. 합 타입은 보여 주지 않습니다(디버거가 태그와 모든 변형을 보여 줌).

```
lldb -O "$(nabic --print lldbinit)" ./target/debug/my_binary
nabic --print lldbinit >> ~/.lldbinit
gdb -iex "$(nabic --print gdbinit)" ./target/debug/my_binary
nabic --print gdbinit >> ~/.gdbinit
cdb -c ".nvload $(nabic --print natvis)" my_binary.exe
```

Visual Studio에서는 이 파일을 `%USERPROFILE%\Documents\Visual Studio 2022\Visualizers`에 복사합니다.

### macOS / Windows 디버거

각 플랫폼의 기본 디버거가 Nabi 바이너리를 처리합니다.

- macOS에서는 lldb나 Xcode 디버거를 씁니다.
- Windows에서는 Visual Studio 디버거나 WinDbg를 씁니다(PDB에서 함수, 소스 줄, 매개변수와 지역 변수를 읽음).

### panic 백트레이스

`NABI_BACKTRACE=1`이나 `full`이면 panic이 호출 스택을 출력합니다([4.14.13](../004-functions/414-throws.md#41413-panic과-fatal-에러)). `-g`는 필요 없습니다. 런타임은 nabic이 내보내는 모든 함수가 유지하는 프레임 포인터를 따라갑니다. 출발점은 런타임의 panic 진입점(`nabi_panic`, `nabi_panic_at`, `nabi_panic_str`, `nabi_try_failed`)을 부른 호출입니다. 각 반환 주소의 이름은 Nabi의 링커가 모든 실행 파일에 써 넣는 함수 표(`.nabi_symbols`, `__TEXT,__nabi_symbols`, Windows에서는 `.rdata`의 일부)에서 찾습니다. 표에는 살아 있는 Nabi 함수마다 주소 범위와 Nabi 경로가 주소 순으로 들어 있습니다. 런타임의 워드 `nabi_backtrace_symbols`가 자기 위치에서 표까지의 거리를 담으므로 표에는 재배치가 필요 없습니다.

- 호출한 쪽에 인라인된 함수는 자기 프레임이 없습니다. 그래서 최적화 빌드는 프레임이 더 적게 보입니다. 합쳐진 사본([9.2](902-aot-pipeline.md))은 남은 사본의 이름으로 보입니다.
- 걷기는 Nabi 코드가 아닌 첫 프레임에서 멈춥니다. C 코드와 런타임에는 따라갈 프레임 포인터가 없습니다.
- 백트레이스 대신 `note:` 한 줄만 나오는 경우가 있습니다. 다른 링커(`NABI_CC`)로 링크한 프로그램에는 표가 없고, 런타임 안에서 일어난 panic(컴파일된 코드의 호출이 아닌 것)에는 출발할 프레임이 없습니다.
- 타깃: powerpc64le를 뺀 모든 네이티브 타깃입니다. wasm32 코드는 자기 호출 스택을 읽을 수 없고, OS가 없는 타깃에는 환경이 없습니다. 둘 다 백트레이스를 출력하지 않습니다.

## 9.9.7. 프로파일러 통합

### perf (Linux)

CPU 프로파일링은 `perf`로 할 수 있습니다.

```
perf record ./target/release/my_binary
perf report --stdio | nabi demangle
```

`perf report`는 심볼 테이블에서 함수 이름을 읽고, 거기서 Nabi 함수는 `_Y` 심볼입니다([9.5](905-symbol-mangling.md)). 인자 없는 `nabi demangle`([9.9.3](#993-심볼-디맹글링))이 각 심볼을 Nabi 이름으로 바꿉니다. `-g` 빌드의 디버그 정보에서 함수 이름을 읽는 프로파일러는 Nabi 경로(`geom::distance`)를 그대로 보여 줍니다.

### Tracy / Chrome trace(선택)

표준 라이브러리의 tracing 표면으로 `tracy`나 Chrome `chrome://tracing`과 호환되는 트레이스를 출력할 수 있습니다. 구체적인 라이브러리 선택은 이 사양의 범위 밖이고, 별도 라이브러리 사양에서 다룹니다.

## 9.9.8. 메모리 / GC 도구

런타임은 다음 환경 변수를 읽습니다.

| 변수                      | 효과                                                                         |
|---------------------------|------------------------------------------------------------------------------|
| `NABIR_GC_LOG=1`           | 수집마다 한 줄을 stderr에 출력. 종류(minor / full), 일시 정지 시간, mark 수, sweep 결과(살아 있는 바이트의 전 -> 후), 이어서 일시 정지의 부분별 시간(마이크로초: safepoint까지 걸린 시간, mark, sweep, evacuation), 스레드 수, 수집기가 시작된 뒤의 시각. 동시 full 수집은 시작 일시 정지(minor 수집, mark 수는 그 minor의 것)와 끝 일시 정지를 따로 기록함. safepoint까지 걸린 시간은 멈춤 요청부터 다른 모든 스레드가 멈출 때까지 기다린 시간. 이 값이 길면 수집이 느린 것이 아니라 어떤 스레드가 멈춤을 붙잡은 것 |
| `NABIR_GC_STATS=1`         | 종료할 때 수집 횟수(minor / full), 일시 정지 시간, 수집기 CPU 시간을 stderr에 출력. 둘째 줄은 블록 수 / 상주 크기 / evacuation 횟수 / 옮긴 바이트, 셋째 줄은 full 일시 정지, 동시 mark(그중 할당이 pause 안에서 끝낸 것), marker의 CPU 시간, mark assist 횟수([7.5.4](../007-runtime/705-concurrent-collection.md#754-할당과-수집)). 이어서 실행된 일시 정지 종류(minor, full, 동시 시작, 동시 끝)마다 한 줄: 횟수, p50 / p90 / p99(히스토그램에서 구해 1/8 안쪽), 가장 긴 일시 정지와 합계, safepoint까지 걸린 시간의 최대와 합계 |
| `NABIR_GC_CONCURRENT=auto\|always\|off` | full 수집의 동시 mark. `auto`는 old 객체가 32 MiB 이상이거나 일시 정지 목표가 있을 때([7.5](../007-runtime/705-concurrent-collection.md)) |
| `NABIR_GC_EVAC=always`     | 테스트용. full 수집마다 evacuation([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) |
| `NABIR_GC_YOUNG=<bytes>`   | young 예산. 수집 사이에 내주는 바이트를 그대로 정함(기본값: 할당하는 스레드마다 8 MiB, 32 MiB~128 MiB, old 바이트가 더 많으면 그만큼이되 128 MiB까지)([7.2.8](../007-runtime/702-allocation.md#728-gc-트리거)) |
| `NABIR_GC_WORKERS=<n>`     | stop-the-world 일시 정지에서 수집기 옆에서 함께 mark하는 도우미 스레드 수(기본값: 첫 CPU를 뺀 온라인 CPU마다 하나, 최대 7. `0`: 없음)([7.3.3](../007-runtime/703-gc-algorithm.md#733-mark의-절차)) |
| `NABIR_GC_STRESS=N`        | 테스트용. 할당 N번마다 수집. 그 8번째마다, 마지막 full 이후 old 세대의 16분의 1 이상을 할당했으면 full |
| `NABIR_HEAP_DUMP=on_oom`   | OOM이 나면 합계, 바이트 기준 상위 32개 타입, 가장 큰 큰 객체 32개를 stderr에 출력. 최선 노력이며 할당하지 않고 락을 잡지 않음. 형식은 런타임이 정함 |
| `NABIR_WORKERS=<n>`        | executor 워커 수 N(기본값은 온라인 CPU 수)([7.6.8](../007-runtime/706-scheduler.md#768-thread-per-core-옵션)) |
| `NABIR_THREAD_PER_CORE=1`  | thread-per-core. task를 자기 워커에 고정하고 훔쳐 가지 않음(7.6.8) |
| `NABIR_DYNAMIC_WORKERS=1`  | 쉬는 워커는 재우고, 활성 워커가 모두 바쁘면 다시 씀([7.7.8](../007-runtime/707-io-and-timer.md#778-워커-수의-동적-조절)) |

설정하지 않았거나 비어 있는 변수는 기본값을 씁니다. 표에 없는 값을 주면 stderr에 경고를 한 줄 출력합니다. 경고는 변수, 준 값, 받는 값을 알립니다. 그 변수는 기본값을 씁니다.

## 9.9.9. REPL / Jupyter 디버그

| 수단                 | 효과                                                                 |
|----------------------|----------------------------------------------------------------------|
| `<식>?`              | 식을 평가하고 `Debug` 형태로 출력([9.7.7](907-jit-cells.md#977-시각화--출력)) |
| `it`                 | 식 셀이 마지막으로 보여 준 값. 사용자의 `it`이 우선([9.7.7](907-jit-cells.md#977-시각화--출력)) |
| `:type <식>`, `:t`   | 식의 정적 타입을 출력(실행하지 않음)                                 |
| `:doc <이름>`        | 이름의 시그니처와 문서 주석을 출력. Jupyter `inspect_request`와 같은 텍스트([9.8.6](908-jupyter-integration.md#986-inspect_request-도움말)). 메서드는 `:doc xs.len` |
| `:time <식>`         | 식을 셀로 평가하고 단계별 시간(`NABI_REPL_TIME=1`처럼)과 합계를 stderr에 출력 |
| `:vars`              | 바인딩을 `name: type`으로, 이어서 항목의 시그니처를 나열              |
| `:load <file>`       | 실행 중인 세션에 프로그램을 로드([9.7.10](907-jit-cells.md#9710-프로그램-로드repl-path)). 그 항목이 같은 이름 항목을 대신함. `:save`로 쓴 스크립트는 그 셀을 하나씩 다시 실행 |
| `:reload`            | 마지막으로 로드한 프로그램을 다시 로드                               |
| `:edit`              | `$VISUAL`이나 `$EDITOR`(없으면 `vi`, Windows는 `notepad`)로 임시 파일을 열고 그 내용을 셀로 실행. 셀이 실패하면 다음 `:edit`에서 그 텍스트를 다시 엶 |
| `:save <file>`       | 세션을 스크립트로 씀. 항목을 먼저, 이어서 실행된 셀을 입력한 그대로 씀. 다시 선언한 이름은 거기서도 재선언임([5.1.4](../005-modules/501-variable-declaration.md#514-같은-스코프-안의-재선언)). `it`은 이후 셀이 읽는 곳에서만 선언. 첫 줄은 `// nabi repl session (:save); :load replays it cell by cell`이고 각 셀 앞에 `// :cell` 줄이 옴. 스크립트를 `:load`로 로드하면 셀을 하나씩 다시 실행하므로 다시 선언한 이름은 마지막 바인딩을 유지함. `nabi run`은 프로그램으로 실행함 |
| `:opt [<level>]`     | 최적화 수준(`-O`처럼 `0`, `1`, `2`, `3`, `s`)을 출력하거나 이후 셀에 설정 |
| `:reset`             | 세션을 처음 상태로 되돌림(라이브러리 경로의 `use`는 다시 넣음)         |
| `:add <dep> ..`      | 의존성을 추가하고(현재 환경에서는 `nabi install`, 패키지의 세션에서는 `nabi add`) 세션에 붙임([9.10.5](910-nabi-frontend.md#9105-환경)) |
| `:help`, `:h`        | 모든 명령과 별칭을 나열                                               |
| `:quit`, `:q`, `:exit` | 종료(입력이 끝나도 종료)                                           |
| `exit`, `quit`, `exit()`, `quit()` | 셀에 단독으로 있으면 종료. 세션이 그 이름(바인딩, 항목, `use`)을 정의하면 그쪽이 우선 |
| `NABI_REPL_TIME=1`    | 셀마다 단계별 시간과, 새로 컴파일한 함수 / 앞 셀에서 재사용한 함수 수([9.6.4](906-jit-engine.md#964-라이브러리-코드의-재사용)) |

셀의 nabi-mir를 보려면 같은 코드를 `main.nabi`로 두고 `nabic build --emit=mir`을 씁니다(9.9.1).

## 9.9.10. 메타 블록 디버그

[사용자 정의 메타 블록](../006-meta-system/608-meta-blocks.md)의 derive와 속성이 어떤 코드를 생성하는지 확인할 수 있습니다.

```
nabic build --emit=expansions    # → target/<profile>/build/<name>.expanded.nabi
```

이 옵션은 사용자 패키지의 [메타 확장](../006-meta-system/609-template-tokens.md) 결과를 소스 텍스트로 출력합니다. 확장마다 다음 주석 뒤에 생성된 소스가 이어집니다. 생성물을 다시 확장한 결과는 바깥 확장 뒤에 이어집니다.

```
// expanded from this #[derive(MyHash)] (src/geom.nabi:12:1)
impl MyHash for Point { ... }
```

메타 확장과 디슈거링을 모두 거친 전체 AST를 Nabi 소스로 보려면 `--emit=source`(`<name>.ast.nabi`)를 씁니다.

## 9.9.11. 벤치마크

마이크로벤치마크는 다음 명령으로 실행할 수 있습니다.

```
nabi bench
```

`nabic bench`가 벤치마크 하네스를 빌드하고, `nabi bench`([9.10](910-nabi-frontend.md))가 하네스를 실행합니다([9.1.2](901-nabic-cli.md#nabic-bench-벤치마크)). 하네스는 `#[bench]`가 붙은 함수를 `-O2` 빌드에서 하나씩 측정합니다. 각 함수는 `Bench` 핸들을 받아 `b.iter`로 클로저를 측정합니다. 그래서 클로저 밖의 준비 코드는 측정되지 않습니다. 측정 방법은 다음과 같습니다.

1. **클록.** 모노토닉 나노초 클록을 씁니다. 표본마다 호출 한 번이 아니라 `iters`번의 호출을 측정해 고정 오버헤드를 상각합니다.
2. **보정.** `--iters`를 주지 않으면 표본 하나가 약 100 ms가 될 때까지 `iters`를 키웁니다. 그러면 클록 양자화와 오버헤드가 무시할 수준이 됩니다.
3. **Warm-up.** 표본 하나 정도의 시간 동안 호출을 돌리고 결과는 버립니다. 이렇게 캐시, 분기 예측기, 페이징을 안정시킵니다.
4. **표본.** 표본을 `--samples`개(기본 10개) 얻습니다. 각 표본은 `iters`번 호출한 총시간을 기록합니다. 호출 한 번의 시간은 총시간을 `iters`로 나눈 값입니다.
5. **통계.** 표본의 중앙값, 사분위수, 중앙값 절대편차(MAD)를 계산합니다. Tukey 울타리(사분위수에서 1.5 IQR 밖)를 벗어난 표본은 이상치로 셉니다. 보고 수치는 모든 표본의 **중앙값(+/- MAD)**이므로 가끔 나오는 느린 표본에 거의 흔들리지 않습니다.
6. **GC 인지.** 표본마다 런타임의 수집 카운터를 전후로 읽고, 수집이 일어난 표본의 수를 셉니다. 두 수 중 하나라도 0이 아니면 사람용 줄 끝에 `[<n> outliers, <m> gc]`를 붙입니다. `--format=nabibench`는 언제나 둘 다 적습니다.

`black_box(x)`는 옵티마이저가 꿰뚫어 볼 수 없는 항등 함수입니다. 측정 클로저의 입력과 결과에 적용하세요. 그러면 작업이 밖으로 끌려 나가거나 접혀 사라지지 않습니다. `b.set_bytes(n)` / `b.set_items(n)`은 호출당 시간을 처리량(`MiB/s`, `Melem/s`)으로 바꿉니다.

출력은 터미널이든 아니든 사람이 읽는 단위로 정렬됩니다. `--format=nabibench`는 회귀 도구가 읽는 안정된 기계용 형식을 벤치마다 한 줄씩 출력합니다([9.1.2](901-nabic-cli.md#nabic-bench-벤치마크)). `--format=json`은 JSON 레코드를 출력합니다([9.9.12](#9912-ci--테스트-통합)). `b.iter`를 한 번도 부르지 않은 벤치는 아무것도 측정하지 않았다고 보고합니다. 실패가 아니라 경고입니다. `--iters` / `--samples`로 표본을 조절합니다.

## 9.9.12. CI / 테스트 통합

### CI 모드

CI 환경에서는 기계가 읽을 수 있는 출력을 권장합니다.

```
nabic build --message-format=json --no-color > build.log
```

이 출력은 CI 시스템이 파싱하기 좋습니다.

### 테스트와 벤치 레코드

`nabi test --format json`과 `nabi bench --format json`은 stdout에 JSON 객체를 한 줄에 하나씩 씁니다. 사람용 텍스트, 테스트의 출력, 벤치가 직접 출력한 내용은 stderr로 가므로 stdout에는 레코드만 남습니다. 레코드마다 `schema`(지금은 `1`), `type`, `kind`(`test`나 `bench`)가 있습니다.

```json
{"schema":1,"type":"case","kind":"test","name":"math::adds","result":"ok","ms":3.215}
{"schema":1,"type":"case","kind":"bench","name":"sum","result":"ok","ms":270.1,"median_ns":6.833,"mad_ns":0.167,"samples":10,"outliers":0,"gc":0,"bytes":0,"items":0}
{"schema":1,"type":"case","kind":"test","name":"net::downloads","result":"ignored","ms":0.002,"reason":"needs the network"}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"ignored":1,"filtered_out":0,"ms":120.4}
```

| `type`    | 필드                                                                |
|-----------|---------------------------------------------------------------------|
| `case`    | `name`, `result`(`ok`, `failed`, `ignored`(`#[ignore]` 테스트나 문서 블록, 아무것도 측정하지 않은 벤치) 또는 `timeout`(시간 제한에서 강제 종료된 테스트)), `ms`(그 프로세스의 실제 경과 시간), 이유가 있으면 `reason`(실패 이유나 ignore 이유). `result`가 `ok`인 벤치에는 `#nabibench` 줄([9.1.2](901-nabic-cli.md#nabic-bench-벤치마크))과 같은 `median_ns`, `mad_ns`, `samples`, `outliers`, `gc`, `bytes`, `items`도 있음 |
| `summary` | `result`(`ok`나 `failed`), `ms`, 그리고 개수. 테스트는 `passed`, `failed`, `ignored`, `filtered_out`, 벤치는 `run`, `failed`, `ignored`, `filtered_out` |
| `listed`  | `--list`가 남긴 케이스마다 `name`, ignored 테스트는 `ignored`(`true`). summary는 뒤따르지 않습니다([9.10.1](910-nabi-frontend.md#9101-서브명령과-전역-옵션)) |

summary는 하네스마다 그 case들 뒤에 옵니다. 그래서 루트 파일이 둘 다 있는 패키지는 라이브러리 실행이 통과하면 summary가 둘입니다. 레코드는 자기 번호(지금은 `1`)로 JSON 진단의 `schema` 규칙([9.9.4](#994-진단--에러-메시지))을 따르므로, 읽는 쪽은 모르는 필드를 무시합니다. `--message-format=json`을 함께 주면 빌드 진단도 같은 stdout에 JSON 줄로 나옵니다. 진단은 `type`(`error`, `warning`, ..)으로 레코드와 구별합니다.

### 종료 코드

[`nabic` CLI 인터페이스](901-nabic-cli.md)의 종료 코드는 다음과 같습니다.

| 코드  | 의미             |
|-------|------------------|
| `0`   | 성공             |
| `1`   | 사용자 코드 에러 |
| `2`   | CLI 에러         |
| `70`  | nabic 내부 panic  |

`nabic test`는 하네스를 빌드할 뿐 테스트를 실행하지 않습니다. 실행은 `nabi test`가 하고, 실패한 테스트가 있으면 `1`로 끝납니다. 테스트마다 별도 프로세스에서 돌므로 테스트의 panic(`101`)이 러너의 `101`이 되지는 않습니다.

CI는 코드마다 다른 처리(재시도, 알림 등)를 할 수 있습니다.

## 9.9.13. LSP 서버

`nabi lsp`는 LSP(Language Server Protocol) 서버입니다([9.10.8](910-nabi-frontend.md#nabi-lsp-lsp-서버)). 같은 서버를 독립 바이너리 `nabils`로도 배포합니다. 서버는 stdio로 통신합니다. 지원하는 기능은 다음과 같습니다.

- 진단, hover(멤버 포함), go-to-definition(정의하는 이름의 범위를 돌려줌)
- references, rename, 문서 심볼
- 멤버 / 경로 완성, 시그니처 도움말, 빠른 수정, 포맷
- semantic token. 정적 문법이 알아볼 수 없는 매크로 호출, 사용자 정의 키워드, 리터럴 접미사, 연산자, 속성에 붙입니다.

references와 rename은 버퍼 안에서만 동작합니다.

```
nabi lsp
nabils
```

이 모드에서 VS Code, Vim, Emacs 같은 LSP 클라이언트는 자동 완성, hover, go-to-definition, 진단 등을 받습니다. 서버는 내부에서 `nabi`에 링크된 컴파일러의 증분 프런트엔드, 곧 `nabic check`가 돌리는 프런트엔드를 씁니다.

- **자동 완성.** `x.` 뒤에서는 `x` 타입의 필드와 메서드가 후보입니다. 표준 라이브러리나 다른 패키지의 타입이면 `pub`인 것만 내놓습니다. `#[doc(hidden)]` 멤버와 연산자 트레잇(`a + b`, `xs[i]`처럼 쓰는 것)의 메서드는 뺍니다. `a::` 뒤에서는(`use a::b::` 안에서도) `a` 아래의 모듈과 항목이 후보입니다. 그 밖에서는 커서 위치에서 보이는 지역 변수, 버퍼의 항목, 경로 없이 쓸 수 있는 이름(core 라이브러리의 이름과 `use` 항목이 가져온 이름), 키워드가 후보입니다. REPL과 Jupyter 커널도 같은 엔진으로 완성합니다([9.7.8](907-jit-cells.md#978-입력-단위-repl-식별)).
- **hover와 정의.** 함수, 메서드, 필드 위의 hover는 시그니처와 그 문서 주석을 보여 줍니다. 표준 라이브러리의 항목도 마찬가지입니다. `x.name`의 go-to-definition은 `x`의 타입에서 `name`을 찾습니다. 표준 라이브러리 항목이면 라이브러리 소스의 그 항목으로 갑니다.
- **시그니처 도움말.** 호출의 괄호 안에서(`(`와 `,`에서 요청) 호출되는 함수의 시그니처와 문서 주석을 보여 줍니다. 커서가 있는 매개변수는 활성으로 표시합니다. 메서드는 receiver의 타입에서 찾고, 그 `self`는 세지 않습니다.
- **빠른 수정.** 편집을 담은 진단(``convert with `as i32` `` 등)은 그 편집을 `quickfix` 코드 액션으로 내놓습니다. 편집이 없으면 무엇을 쓸지 말하는 help 줄이 코드 액션이 됩니다.
  - ``add `;` ``는 진단 span의 끝에 `;`를 넣습니다.
  - ``add `use pkg::a::b;` ``는 그 줄을 파일의 마지막 최상위 `use` 뒤에 넣습니다. 그런 줄이 없으면 맨 앞의 `//!` 줄과 내부 속성 뒤에 넣습니다.
  - ``nabi add <name>``은 명령 `nabi.add`가 됩니다.

  `workspace/executeCommand`를 받으면 서버는 패키지 디렉터리에서 `nabi.add`를 `nabi add <name>`으로 실행합니다. 스크립트면 그 파일의 디렉터리에서 `nabi add <name> <file>`로 실행합니다. `nabi`는 `$NABI`이 가리키는 것을, 없으면 서버 옆의 것을 씁니다. 실행한 뒤에는 의존성을 다시 해석하고 열린 버퍼를 다시 검사하므로, 다시 시작하지 않아도 새 패키지를 찾습니다. 다른 명령은 실행하지 않고 플래그도 받지 않습니다.
- **rename.** `prepareRename`은 rename이 받아들일 수 있는 이름이면 커서 아래 이름의 범위를 돌려줍니다. 버퍼 밖에서 선언된 이름이면 어디서 왔는지 말하는 메시지로 실패합니다. 예를 들어 ``cannot rename `println`: it is defined in the standard library``입니다. 키워드이면 `` `let` is a keyword ``입니다.
- **문서 심볼.** 심볼의 range는 수식어(`pub`, `const`, ..)부터 닫는 `}`나 `;`까지 항목 전체를 덮습니다. selection range는 이름입니다.
- **입력하는 대로 진단.** 진단은 저장할 때만이 아니라 버퍼가 바뀔 때마다(`didOpen`, `didChange`) 보냅니다. 검사는 백그라운드에서 한 번에 하나씩, 30 ms 동안 편집이 없으면 시작합니다. 그사이 도착한 편집은 함께 최신 텍스트로 검사하고, 검사하는 동안 버퍼가 바뀌었으면 그 검사의 진단은 보내지 않습니다(이전 텍스트를 가리키므로). 진단은 컴파일러의 것과 글과 코드가 같으므로, 어휘 문제(보이지 않는 문자나 닮은 문자, 홀로 쓴 결합 부호, 주석 안의 양방향 제어 문자, 잘못된 숫자, 잘못된 이스케이프)와 처리기가 없는 리터럴(`10Hz`, `95%`, `dat"..."`)이 입력하는 동안 보이고, 진단마다 그 진단에 딸린 빠른 수정이 함께 옵니다.
- **의존성.** 각 버퍼는 그 패키지의 의존성으로 검사합니다. 패키지는 버퍼 위로 가장 가까운, `nabi.nabion`이나 `src/main.nabi`, `src/lib.nabi`가 있는 디렉터리입니다. 의존성은 그 매니페스트와 lock에서 해석하고, 거기서 시작한 `nabi lsp`처럼 빌드합니다. 어느 패키지에도 속하지 않지만 작업 공간 폴더(`workspaceFolders`, 없으면 `rootUri`) 안에 있는 버퍼는 환경의 라이브러리를 쓰고([9.10.5](910-nabi-frontend.md#9105-환경)), 어느 폴더에도 없으면 서버가 시작한 디렉터리의 의존성을 씁니다. 패키지마다 한 번 해석합니다. 매니페스트나 lock을 저장하거나(`didSave`), 에디터가 그 변경을 알리거나(`workspace/didChangeWatchedFiles`, 클라이언트가 허락하면 서버가 `**/nabi.nabion`과 `**/nabi.lock`을 등록), 작업 공간 폴더가 바뀌면 다시 해석하고 열린 버퍼를 다시 검사합니다. 의존성을 해석할 수 없는 패키지는 의존성 없이 검사하고, 서버가 이유를 한 번 알립니다(`window/showMessage`).
- **검사 중의 응답.** hover, 정의로 이동, 완성, 시그니처 도움말, 문서 심볼, 포매팅은 편집의 검사를 기다리지 않습니다. 타입 검사를 통과한 마지막 검사로 답하고, 위치는 그 뒤의 편집을 거쳐 옮깁니다. 그 검사 뒤에 편집된 텍스트 위의 위치(커서 아래 이름 포함)는 현재 텍스트의 검사를 기다리고, 지금의 버퍼로 편집하는 참조와 이름 바꾸기도 기다립니다. `x.` 뒤의 완성은 입력 중인 줄만 바뀌었고 `x`가 그 검사 때와 같게 쓰여 있으면 `x`의 타입을 그 검사에서 가져오고, 아니면 버퍼를 `.` 앞까지 검사합니다. 답하기 전에 클라이언트가 취소한 요청(`$/cancelRequest`)은 오류 `RequestCancelled`(-32800)를 받습니다.
- **진단 범위.** span이 빈 진단은 그 위치의 토큰 위에 보고합니다. 그 위치가 줄 끝이면 앞 토큰 위에 보고합니다.
- **손으로 시작했을 때.** 터미널에서 시작하면 서버는 먼저 표준 에러에 ``nabils speaks the language server protocol on stdin; start it from an editor (see `nabi help lsp`)``를 출력합니다. 클라이언트가 `initialize`를 보내기 전에 입력이 끝나면 ``nabils: input ended before the editor sent `initialize` ``를 출력하고 1로 끝납니다. `initialize` 뒤에는 LSP 표준대로 `shutdown`을 받았으면 0으로, 아니면 1로 끝납니다.

LSP의 세부 사항은 이 사양의 범위 밖입니다. LSP 표준과 toolchain의 구현 결정으로 따로 정합니다.

## 9.9.14. 패키지 검사

미리 빌드한 라이브러리 패키지의 정보를 한 번에 볼 수 있습니다.

```
nabi inspect path/to/libfoo.a        # 또는 foo.nabihir. 짝 파일을 함께 읽음
```

이 명령은 NABIHIR([9.4](904-nabihir-bytecode.md))과 [아카이브](../000-meta/007-glossary.md#archive)의 정보를 다음과 같이 표시합니다.

```
Package: foo
Hash: 3f8a1c2b
nabic version: 0.1.0 (<compiler id>)
NABIHIR version: 1
GC type ids: 4096..4160

Exports:
  pub fn distance(p1: Point, p2: Point) -> f64
  pub struct Point
  ...

Generic items (NABIHIR):
  pub fn id<T>(x: T) -> T
  ...

Concrete .o files:
  part0.o (12 KB)
  part1.o (3 KB)
```

`nabi inspect --dump <file>`은 NABIHIR의 버전, producer, 메타와 인터페이스 파일 전체를 출력합니다(9.9.2).

패키지가 바깥에 무엇을 노출하는지 한눈에 볼 수 있습니다.

## 9.9.15. 정리

이 절의 도구는 Nabi와 함께 쓰는 표준 도구(gdb, lldb, perf, Jupyter)와 Nabi 고유 도구(`--emit`, `nabi demangle`, `nabi inspect`, LSP 서버)입니다. 프런트엔드 `nabi`와 그 도구는 [9.10](910-nabi-frontend.md)에서, nabi-mir 파일 형식은 [9.11](911-mir-format.md)에서 정의합니다.
