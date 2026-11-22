# 9.9. 偵錯 / 診斷工具

本節定義 `nabic` 與 `nabi` 所提供的偵錯 / 診斷工具，以及它們與外部工具的整合。

## 9.9.1. 中間輸出：`--emit`

[AOT 建置管線](902-aot-pipeline.md)各步驟的結果都可以傾印出來。

```
nabic build --emit=source        # full AST after meta expansion and desugaring, as nabi source
nabic build --emit=hir          # nabi-hir text output
nabic build --emit=mir           # nabi-mir text output (optimized above -O0)
nabic build --emit=obj           # object file
nabic build --emit=asm           # assembly source of the object
nabic build --emit=listing       # machine code as text, with offsets and bytes
nabic build --emit=expansions    # meta expansion result (9.9.10)
nabic build --emit=c-header      # C header for the package's exports (9.3.3)
```

`mir` 是程式碼產生器所編譯的 nabi-mir：在 `-O0` 以上，是最佳化之後的（[9.2.10](902-aot-pipeline.md#9210-最佳化等級)）。`obj` 是它所寫出的目的檔。`objdump -d` 之類的反組譯器會顯示 `obj` 的機器碼。

`asm` 是以組合語言原始碼表示的目的檔，與 `clang -S` 的寫法相同：為目標組譯它（`clang -c`、`llvm-mc -filetype=obj`）會產生一個目的檔，其程式碼位元組相同，在相同偏移上有指向相同位置、型別相同的重定位，符號也與 `obj` 相同，因此它的連結與執行結果也相同。它是從 `nabic` 寫出的目的檔讀回來的，採用目標的 GNU / LLVM 語法（在 x86_64 與 x86 上是 `.intel_syntax noprefix` 下的 Intel 語法）。每個區段都有目的檔格式所命名的 `.section` 指示詞（ELF 的 `.text.<symbol>`、`.rodata.<symbol>`；Mach-O 的 `__TEXT,__text`；COFF 的 `.text` COMDAT 及其選擇方式與 leader、與其函式關聯的 `.pdata`）以及其 `.p2align`。符號帶有其繫結與可見性（`.globl`、`.weak`、`.hidden`；ELF 上的 `.type` 與 `.size`；Mach-O 上的 `.private_extern` 與 `.weak_definition`；COFF 上的 `.def`；powerpc64 上的 `.localentry`）。每個函式就是其指令，依程式碼產生器的編碼方式：分支目標是區域標籤（`.LBB0_3`；在 Mach-O 與 x86 Windows 上是 `LBB0_3`），每個被重定位的欄位都是讓組譯器發出該重定位的運算式（`call nabi_panic_at@PLT`、`[rip + x@GOTPCREL]`、`[rip + x@GOTTPOFF]`、`[rcx + x@SECREL32]`、`[ebx + s@GOTOFF]`、`adrp x0, s@PAGE` / `:lo12:s`、`x@TLVPPAGE`、`#:lower16:(s-(.+16))`、`.word x(GOT_PREL)+0x1c`、`%pcrel_hi(s)` 搭配 `%pcrel_lo(.Lpcrel_hi0)`、`s@toc@ha`、`x@got@tprel@l`、`brasl %r14, f@PLT`、`x@INDNTPOFF`……）。組譯器不會為指令文字選擇的編碼，在語法允許之處會被強制指定（長分支上的 `{disp32}`、`{disp8}`、`{rex}`、表示另一種 `mov` 形式的 `mov.s`）；否則指令以其位元組表示，文字放在註解中（x86 的框架設定 `sub rsp, imm32`，其立即值放得進 8 位元）。資料（字串字面值、static、GC 型別描述子、掃描表、vtable）是 `.asciz`、`.byte`、`.zero` 以及指名其目標的指標大小指示詞（`.quad`、`.long s-.`、`.rva`、`.secrel32`、`.secidx`）。展開表是組譯器用來建立它們的指示詞，各自位於它所描述的指令之後：ELF `.eh_frame`（組譯器的 CIE 就是 `nabic` 的）以及 Mach-O arm64 的 compact unwind 用 `.cfi_startproc`、`.cfi_def_cfa_offset`、`.cfi_def_cfa_register`、`.cfi_offset` 與 `.cfi_endproc`，組譯器從它們所描述的框架紀錄與暫存器對推導出 compact unwind；x64 COFF 的 `.pdata` / `.xdata` 用 `.seh_proc`、`.seh_pushreg`、`.seh_stackalloc`、`.seh_setframe`、`.seh_savereg`、`.seh_savexmm`、`.seh_endprologue` 與 `.seh_endproc`；32 位元 ARM 的 `.ARM.exidx` 用 `.fnstart`、`.save`、`.setfp`（或 `.cantunwind`）與 `.fnend`。組譯器產生的表解碼結果相同（`llvm-dwarfdump --eh-frame`、`llvm-readobj --unwind`、`llvm-objdump --unwind-info`）。arm64 Windows 把其 `.pdata` / `.xdata` 保留為帶有重定位的資料：在 `nabic` 把展開碼完整寫出之處，組譯器會壓縮它們（packed `.pdata` 項目、共用 prolog 展開碼的 epilog）。`-g` 的偵錯區段（DWARF、CodeView）也是資料，32 位元 ARM 的建置屬性則是 `.eabi_attribute` 指示詞。RISC-V 原始碼以 `.option arch`、`.option norvc` 與 `.option norelax` 開頭；16 位元壓縮形式的指令（`-O0` 以上）在 `.option rvc` 之下以其 `c.` 助憶符表示（`c.addi sp, -0x10`、`c.beqz a0, .LBB0_3`），在下一個 32 位元指令之前會切回 `.option norvc`；其 ELF 旗標仍來自組譯器的選項（`clang --target=riscv64-linux-gnu`，或 `llvm-mc -mattr=+m,+a,+f,+d,+c -target-abi=lp64d`）。組譯器可能寫得不同的地方意義相同：以符號的偏移對區域符號之區段進行的重定位；它總是會加上的區段、對應（`$x`、`$d`）與 Mach-O `ltmp` 符號；保留為符號的 RISC-V 區域標籤；x86_64 `.eh_frame` 的 `SHT_X86_64_UNWIND` 型別；Mach-O 函式對自身的呼叫及其 compact unwind 項目在區段內解析；32 位元 ARM 的硬體浮點 ELF 旗標，沒有任何指示詞會設定它（建置屬性會說明它）；COFF weak external 之預設符號的名稱（`.weak.<symbol>.default`，帶有組譯器加上的後綴）；展開表的配置：在 `nabic` 共用一個之處每個函式各一個 x64 `UNWIND_INFO`、32 位元 ARM 的 `pop {r4-r11, lr}` 的短形式。

在 wasm32 上，原始碼是 LLVM 的 WebAssembly 組合語言，供 `clang --target=wasm32-wasi -c` 或 `llvm-mc -triple=wasm32-wasi -filetype=obj` 使用。其指令所需的功能來自組譯器的選項，因為沒有任何指示詞會設定它們：clang 用 `-msimd128`（在 `wasm32-wasip1-threads` 上還有 `-matomics`），`llvm-mc` 用 `-mattr=+bulk-memory,+mutable-globals,+nontrapping-fptoint,+reference-types,+sign-ext,+simd128`（以及 `+atomics`）。原始碼宣告它所使用的全域變數、表以及每個函式（`.globaltype __stack_pointer, i32`、`.tabletype __indirect_function_table, funcref`、`.functype f (i32, i64) -> (f64)`），對以另一個名稱匯入的函式還有 `.import_module` 與 `.import_name`。每個函式位於自己的區段中（`.section .text.<symbol>,"",@`），帶有其繫結（`.globl`、`.weak`、`.hidden`、`.export_name`）、其 `.functype` 與其 `.local` 行；其指令中的每個 `end` 以它所關閉的東西命名（`end_block`、`end_loop`、`end_if`、`end_function`），未定型的 `select` 以其運算元的型別命名（`i32.select`），每個被重定位的運算元就是其符號：`call f`、`global.get __stack_pointer`、`i32.const s`（資料位址，或函式的表索引）、`i32.const s@TLSREL`、`call_indirect __indirect_function_table, (i32) -> (i32)`。每個資料段都是自己的一個區段（`.section .data.<symbol>,"",@`，執行緒區域時為 `"T"`），帶有其 `.p2align`、其符號（`.type s,@object`、`.size`），每個指標則是 `.int32 s`；偵錯區段與 `target_features` 是資料，程式碼偏移為 `.int32 f+0x2b`。組譯器產生的目的檔有相同的函式、程式碼（被重定位的運算元長 5 個位元組，與這裡相同）、重定位、資料段、符號與自訂區段。它寫得不同的地方意義相同：函式與型別索引的順序、記憶體與表匯入的初始大小、它在 `__indirect_function_table` 上設定的 `NO_STRIP` 旗標，以及其 `DATACOUNT` 區段。

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

`listing` 是以文字表示的機器碼，由程式碼產生器從它所編碼的指令寫出，因此其位元組與 `obj` 的完全相同。每個函式以其符號（`<name>:`）開頭；每道指令一行，包括其相對於函式開頭的偏移、其位元組與其文字，與 `llvm-objdump -dr` 為該目標印出的相同（在 x86_64 與 x86 上是 Intel 語法）。程式碼產生器的標籤位於它們所標示的指令之前（`.L3:`），分支目標會指出其標籤（`jne 0x1c <.L3>`），每個重定位都跟在它所填寫之欄位的指令之後，並使用目的檔格式對它的名稱（`R_X86_64_PLT32`、`ARM64_RELOC_BRANCH26`、`IMAGE_REL_AMD64_REL32`、`R_386_PLT32`、`R_ARM_CALL`、`R_RISCV_CALL_PLT`、`R_PPC64_REL24`、`R_390_PLT32DBL`、`R_WASM_FUNCTION_INDEX_LEB`……）；在 riscv 上，`R_RISCV_PCREL_LO12_I` 會指出目的檔位於其 `auipc` 處的標籤（`.Lpcrel_hi0`），16 位元指令顯示為 4 個十六進位數字，並附上它所代表之 32 位元指令的文字。在 wasm32 上，偏移從函式的符號（其主體大小）起算，一行 `.local` 列出其區域變數，呼叫與全域變數顯示目的檔給它們的索引，控制流程註解則是 objdump 的（`# 1: down to label3`），其標籤編號在整個目的檔中連續計數。

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

輸出寫到下列位置。

```
target/debug/build/foo.ast.nabi
target/debug/build/foo.hir
target/debug/build/foo.mir
target/debug/build/foo.o
target/debug/build/foo.s
target/debug/build/foo.lst
target/debug/build/foo.expanded.nabi
```

要一次選擇多個步驟，請以逗號分隔，如 `--emit=hir,mir,obj`。只有單一種類時，`-o <path>` 可以改變位置。有 `--emit` 時，不會產生執行檔或封存檔。

## 9.9.2. 檢視 NABIHIR

```
nabi inspect --dump path/to/foo.nabihir   # or libfoo.a; reads the sibling foo.nabihir
```

這個指令以人類易讀的形式印出預先建置之函式庫的 [NABIHIR](904-nabihir-bytecode.md)。在版本、產生者與 meta 之後，每個介面檔案以 `--- <path>` 及其文字（具體函式主體被刪減成 `;` 的原始碼）接續。交出摘要接在 `--- summaries` 之下，呼叫摘要接在 `--- calls` 之下，每個函式一行，列出成立的事實（[9.4.7](904-nabihir-bytecode.md#947-主體)）。

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

這個輸出對驗證 [NABIHIR](../000-meta/007-glossary.md#nabihirnabi-hir-位元組碼) 的正確性與相容性很有用。

## 9.9.3. 符號解修飾

```
nabi demangle [-v] <symbol>...
nabi demangle _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190
nabi demangle < symbols.txt
```

這個指令把[符號修飾](905-symbol-mangling.md)的每個修飾後符號，依原始碼的寫法每行印出一個名稱。標準函式庫的路徑以其層級（`core` / `std` / `bare`）開頭。方法（`<type>.<method>`，或泛型型別實例的 `<type>__<method>`）顯示為 `<type>::<method>`，依 trait 選出的方法（`__by_<trait>`）顯示為 `<path::type as trait>::<method>`，而以雜湊命名之 impl 區塊的方法顯示為 `{impl}::<method>`。typehash 會被省略。也接受 Mach-O 形式，即開頭多一個 `_`（`__Y...`，與 `nm` 印出的相同），以及符號之後的 `.<suffix>`。

```
$ nabi demangle __Ya61f26f1_core_collections_dict_Dict_0_0grow_ce3f36e15e552a70
core::collections::dict::Dict::grow
```

有 `-v` 時，每個符號是一個由它所編碼之各部分組成的區塊，區塊之間以空行分隔：

```
package:  3f8a1c2b
path:     geom
name:     distance
generics: a7c91d8e44b2f190 (type tuple hash)
```

若有任何引數不是 Nabi 符號，結束碼是 `1`。沒有符號時，`nabi demangle` 是一個過濾器：它把標準輸入複製到標準輸出，其中的每個 Nabi 符號都換成其單行名稱，其他一切不變，因此效能分析工具、`nm` 與當機輸出都可以經由管道送進它：

```
perf report --stdio | nabi demangle
nm target/debug/my_binary | nabi demangle
```

## 9.9.4. 診斷 / 錯誤訊息

### 標準格式

診斷訊息以下列標準格式印出。

```
error[E0027]: cannot find value `count` in this scope
  --> src/main.nabi:5:13
   |
 5 |     let x = count + 1;
   |             ^^^^^
   |
   = help: did you mean `counter`?
```

這個格式使用[原始碼檔案](../001-lexical/101-source-files.md)的檔案 + 行 + 欄診斷單位。

標頭是 `<severity>[<code>]: <message>`，而 `--> path:line:col` 給出主要位置。主要範圍以 `^` 標底線，次要範圍以 `-` 標示；位於另一個檔案中的次要範圍以 `::: path:line:col` 引入。結尾的 `= note:` 行補充脈絡，`= help:` 行建議修正方式。對於找不到但其他地方有某個項目具有該名稱的情況，說明會給出引入它的 `use` 行（最多三行：先是套件自己的項目，然後是標準函式庫的，然後是相依套件的），或者當套件唯一的這類項目被隱藏時，給出能讓該使用處看見它的可見性及其 `use`；套件本身的項目以從套件根起算的路徑匯入（``add `use pkg::geo::Point;` ``；`` `geo::secret` has that name, but it is private to module `geo`: write `pub(pkg)` before it, so its whole package sees it, and add `use pkg::geo::secret;` ``）。

主要範圍是出錯的程式碼：對型別錯誤的值，是該值，而不是容納它的陳述式。訊息以函式庫項目的寫法指稱它們（`List::sort`、`Send`、`print`，而不是 `core::marker::Send`）。訊息不引用本規格的章節；應改寫成什麼寫在訊息、說明行與 `nabic --explain <code>` 中。

### JSON 格式（工具整合）

為了工具整合，輸出也可以 JSON 形式產生。

```
nabic build --message-format=json
```

每個診斷是 stdout 上的一行 JSON 物件；不會印出人類格式的診斷，stderr 只容納進度行。

```json
{"type":"error","code":"E0040","kind":"type","message":"...","spans":[{"file_name":"src/main.nabi","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"...","line_end":5,"col_end":15}],"notes":["..."],"fixes":[],"help":["..."],"lint":null,"schema":1}
```

| 欄位       | 意義                                                               |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | 錯誤碼，如 `E0040`（[錯誤碼](#錯誤碼)）                       |
| `kind`     | 錯誤碼所屬的領域（`type`、`parse`……）                              |
| `message`  | 訊息                                                               |
| `spans`    | 位置清單。第一個元素是主要位置（`is_primary: true`）；其餘是次要位置 |
| `notes`    | 附註字串的清單，不含說明項目                                       |
| `fixes`    | 修正程式碼的編輯，每個都是一個檔案中的範圍，附上取代它的 `text` 與一個 `title`（見下文）；經常是空的 |
| `help`     | 說明項目（人類格式的 `= help:` 行），不含 `help: ` 前綴            |
| `lint`     | 警告或 lint 錯誤所屬的 lint（`unused_variables`），即 `#[allow(..)]` 與 `-A` 所接受的名稱（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）；其他情況為 `null` |
| `schema`   | 這個格式的版本，目前是 `1`                                         |

範圍有 `file_name`、`line_start`、`col_start`、`byte_start`、`byte_end`、`is_primary`、`label`、`line_end` 與 `col_end`。`line_end` / `col_end` 是範圍結束之處（恰好越過其最後一個字元的位置），與 `line_start` / `col_start` 一樣以碼位計算；手邊沒有原始碼文字時為 `null`。只知道檔案而不知道位置的診斷只有 `{"file_name": .., "is_primary": true}`。修正有 `file_name`、`line_start`、`col_start`、`byte_start`、`byte_end`、`text`、`title`（``convert with `as i32` ``）、`line_end` 與 `col_end`；語言伺服器把它作為快速修正提供（[9.9.13](#9913-lsp-伺服器)）。

在同一個 `schema` 內，欄位只會新增，絕不會被移除、重新命名或賦予另一個意義，因此讀取者應當忽略它不認識的欄位。測試、效能測試與 `nabi deps` 的紀錄（[9.9.12](#9912-ci--測試整合)、[9.10.4](910-nabi-frontend.md#9104-相依套件nabi-add--remove--update--deps)）以它們自己的 `schema`（目前是 `1`）遵循相同的規則。

這個格式讓 IDE 與 CI 系統能以機械方式讀取診斷。

### 錯誤碼

每種錯誤都有自己的錯誤碼，即 `E` 加上四位數字：型別不符是 `E0040`，找不到的名稱是 `E0027`。錯誤碼從 `E0001` 開始連續編號，依加入的順序編號，而不是依領域；`kind` 欄位給出領域。lint（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）的診斷無論由編譯器的哪個部分引發，都有 lint 錯誤碼 `E0131`，並在其附註與 `lint` 中指出該 lint。

`nabic --explain <code>`（也可以是 `nabi --explain`）印出錯誤碼的說明頁：其標題、其意義以及應改寫成什麼（[9.1.5](901-nabic-cli.md#915-全域選項)）。`E0040`、`e0040` 與 `40` 指稱同一個錯誤碼；不存在的錯誤碼是用法錯誤（結束碼 2）。文件中的錯誤頁面就是這些頁面，每個錯誤碼一頁。

標題完全依 `nabic --explain` 印出的樣子寫出。

| 錯誤碼  | `kind`        | 標題                         |
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

`kind` 的值有 `lexical`、`parse`、`resolve`、`visibility`、`module`、`type`、`pattern`、`handoff`（值的所有權與生命週期：交出之後的使用，[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync執行緒邊界標記)；被移動或關閉之後使用的資源，或未關閉的資源，[stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop釋放時的清理)；保留超過 `with` 的執行緒區域值；超過其串流使用的串流視圖，[stdlib 2.23.1](../../refs-standard-libraries/200-core/223-io.md#2231-read)；在其串列調整大小之後使用的 `SliceMut`，[stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt可寫入檢視)）、`meta`、`linkage`、`unsupported`、`internal`、`other` 與 `lint`。

## 9.9.5. 記錄 / 進度顯示

### 標準輸出

預設的建置輸出看起來像這樣。

```
   Compiling foo (path/to/foo)
    Building target/debug/libfoo.a
    Finished in 1.23s
```

進度行輸出到 stderr；動詞清單見 [9.1.8](901-nabic-cli.md#918-輸出格式)。

### `--verbose` / `-v`

詳細模式還會顯示各階段的耗時。

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

顯示每個階段的時間有助於診斷建置的瓶頸。階段有 `front end` 與 `codegen + link`（有 `--emit` 時是 `emit <ext>`）。

### `NABI_TIME_PASSES`

`NABI_TIME_PASSES=1` 在 stderr 上印出每個編譯器階段的實際耗時。

### `-vv`（非常詳細）

除了 `-v` 之外，還會為每個處理的檔案印出一行 `Reading <file>`，並為每個被[特化](../000-meta/007-glossary.md#特化specialize)的（項目，型別）組合印出一行 `Specialized <path>(sig)`。

## 9.9.6. 偵錯工具整合：gdb / lldb

### DWARF 偵錯資訊

`debug` 設定檔的建置，以及帶有 `-g` 的 `release` 建置（[9.2.15](902-aot-pipeline.md#9215-輸出)），會發出具有下列性質的 DWARF 偵錯資訊。Windows 目標則改為以 CodeView 取得相同的資訊（[見下文](#codeview-與-pdbwindows)）。

- 函式的 `DW_AT_name` 是其 Nabi 路徑（`geom::distance`）；不發出連結名稱（修飾後的符號），因為偵錯工具不理解 `_Y` 修飾。呼叫程式之 `main` 的包裝函式（[9.5.9](905-symbol-mangling.md#959-main-函式的符號)）命名為 `nabi_main`，因此回溯中只有一個 `main` 框架，即程式的那一個。
- 行號指向原始 .nabi 檔案的行。
- 區域變數以其原始名稱與型別出現。除了基本型別（整數、浮點數、`bool`、`char`）之外，還會為結構（欄位名稱；泛型結構的實例以原始碼寫出其型別引數的方式命名，如 `core::collections::list::List<i64>`）、元組、`str`、`any`、和型別與陣列發出 DWARF 型別。和型別或列舉是 DWARF variant part：一個人工的判別成員，以及每個變體（包括不帶資料的變體）一個帶有 `DW_AT_discr_value` 的 `DW_TAG_variant`，每個都容納一個名為 `<Sum>::<Variant>` 的結構。編譯單元使用 `DW_LANG_Rust`，讓偵錯工具讀取 variant part（Nabi 沒有 DWARF 語言碼）。`GcBuf<T>` 參照被描述為指向其物件（標頭，然後是 `T` 元素）的指標。`any` 值是涵蓋存入其中之各型別的 variant part，以存放的標籤作為判別值。判別值是人工的 `i32`（存放的標籤；`throws` 承載結構的 `bool` 標籤是一個位元組）。函式值（`fn` 型別或閉包）是以其型別命名的結構（`fn(i32) -> i32`），成員為 `code`，即指向 `DW_TAG_subroutine_type`（環境，然後是參數）的指標，以及 `env`，即其環境（當閉包的型別已知時，是指向環境物件的指標），因此偵錯工具會顯示目標函式。以參照擷取的變數顯示為其值。編譯器產生的函式（C 的 `main`、static 初始化器防護）也有 subprogram。編譯器產生的區域變數（名稱以 `#` 或 `__` 開頭）會被省略。
- 每個區塊都會發出一個詞法區塊範圍，因此被遮蔽的名稱只在其區塊內可見。
- 在 `-O0`，每個區域變數都保有自己的堆疊槽位，其位置就是該槽位。
- 在 `-O0` 以上，`-g` 不改變程式碼（[9.2.10](902-aot-pipeline.md#9210-最佳化等級)）；偵錯資訊描述最佳化後的程式碼。變數的位置是依程式碼範圍的位置清單（`.debug_loc`）：容納其值的暫存器、其堆疊槽位、常數，或對就地建立的結構或閉包而言，是這些的片段（`DW_OP_piece`）。在其值已不存在之處，它沒有位置，偵錯工具會顯示它已被最佳化掉。跨越安全點容納 GC 參照的變數一律位於其堆疊槽位，即回收所讀取與更新的根。
- 在 `-O0` 以上，被內嵌的主體是帶有 `DW_AT_inline` 之抽象 `DW_TAG_subprogram` 的 `DW_TAG_inlined_subroutine`（帶有 `DW_AT_call_file`、`DW_AT_call_line`、`DW_AT_call_column`），容納被呼叫者的參數與區域變數，因此偵錯工具會停在其中，並把它顯示為獨立的框架。內嵌程式碼的行列指名被呼叫者的檔案，只有在行改變之處，該列才是陳述式（`is_stmt`），而 prologue 之後的第一列帶有 `prologue_end`。在 Apple 目標上，單元帶有 `DW_AT_APPLE_optimized`。wasm32 以相同方式描述被內嵌的主體與變數。在 `-O0` 以上，保存在 wasm 區域變數中的變數有一個位置清單，在它容納該值的程式碼上指名該區域變數（`DW_OP_WASM_location 0x0 <local>, DW_OP_stack_value`；結構則每個區域變數一個片段），而位於線性記憶體中的變數在其框架槽位容納其值時，位置是該槽位（`DW_OP_fbreg`；框架基址本身是一個 wasm 區域變數）。在 `-O0`，每個變數都位於其框架槽位。

### CodeView 與 PDB（Windows）

在 Windows 目標上，相同的資訊是每個目的檔中的 CodeView（C13），由連結器彙整成執行檔旁邊的 PDB。

- `.debug$S` 在與每個函式之程式碼關聯的區段中，容納函式的 `S_GPROC32`（以其 Nabi 路徑為名稱）與 `S_FRAMEPROC`、以框架指標相對位置（`S_DEFRANGE_FRAMEPOINTER_REL_FULL_SCOPE`）表示的每個參數與區域變數 `S_LOCAL`、作為 `S_BLOCK32` 的單一程式碼範圍之詞法區塊（其他任何區塊的區域變數都屬於外圍範圍），以及函式的行號表。目的檔的原始碼檔案位於其檢查碼表中。
- 在 `-O0` 以上，變數的位置依程式碼範圍給出：`S_DEFRANGE_REGISTER` 或 `S_DEFRANGE_FRAMEPOINTER_REL`，而對欄位分散的結構（在暫存器中、在記憶體中，或兩者皆有），每個欄位一筆紀錄：暫存器中的欄位用 `S_DEFRANGE_SUBFIELD_REGISTER`，記憶體中的欄位用從框架指標加上欄位偏移的 `S_DEFRANGE_REGISTER_REL`。暫存器以其 CodeView 編號命名：x64 與 x86 上的通用與 XMM 暫存器，arm64 上的 `X0`-`X30`，以及依大小的 `S`、`D` 或 `Q`。唯一的值是一個整數、`bool` 或 `char` 常數的變數是 `S_CONSTANT`；唯一的值是浮點數常數的變數，是其範圍中的一個 `S_LDATA32`，指向與函式程式碼一起的唯讀資料區段中的該值（dbghelp 不讀取浮點數的 `S_CONSTANT`）；其值不在任何地方的 `S_LOCAL` 會被標示為已最佳化掉。Windows ABI 以參照傳遞的結構、元組或陣列參數（x64：不是 1、2、4 或 8 位元組者；arm64：超過 16 位元組者），在每個等級都是不帶參數旗標的 `S_LOCAL`：dbghelp 把這樣的參數讀成指向其值的指標，而 Nabi 描述的是值本身。被內嵌的主體是被呼叫者之 `LF_FUNC_ID` 的 `S_INLINESITE`，帶有把其程式碼對應到其行的二進位註記；目的檔唯一的 `DEBUG_S_INLINEELINES` 子區段（在主要的 `.debug$S` 中，在 PDB 中則每個模組一個：dbghelp 只讀取模組的第一個）給出每個被呼叫者的檔案與行，函式在 `S_INLINEES` 中列出其被呼叫者，而它自己的行號表給內嵌程式碼呼叫處的行。內嵌位置絕不會位於 `S_BLOCK32` 內。
- `.debug$T` 容納型別：基本型別為 CodeView 基本型別；結構、元組、`str`、`any` 與函式值為帶有欄位名稱的 `LF_STRUCTURE`；和型別或列舉為由標籤以及每個變體一個位於偏移 0 之成員組成的結構，每個成員都是涵蓋整個值、名為 `<Sum>::<Variant>` 的結構；`GcBuf<T>` 或裝箱的參照為指向其物件的指標；函式的簽章為 `LF_PROCEDURE`。
- PDB（MSF 7.0）有合併後的每個目的檔的型別紀錄（每種一份）、每個目的檔一個帶有其符號與行號表的模組、全域符號（函式參照、全域資料）與公開符號、區段貢獻以及區段對應表。帶有自己 CodeView 的目的檔（執行環境的）以相同方式納入。執行檔的偵錯目錄以路徑、GUID 與 age 指名該 PDB。

以 dbghelp 為基礎的偵錯工具（WinDbg 與 cdb、Visual Studio 偵錯工具）以名稱找出函式，把位址對應到檔案與行，並顯示參數與區域變數及其值；在最佳化後的程式碼中，還會顯示內嵌的框架（`k`；`bm <module>!<fn>` 在函式被內嵌的每個位置設定中斷點），以及位於暫存器中、分散在暫存器與記憶體之間或為常數的變數（`dv`、`dx`）。

### 使用 gdb / lldb

標準的 gdb 與 lldb 指令可以原樣用於 Nabi 程式碼。

```
gdb -iex "$(nabic --print gdbinit)" ./target/debug/my_binary
(gdb) break main
(gdb) run
(gdb) print x
$1 = 42
(gdb) print xs
$2 = size=3 = {1, 2, 3}     # with the formatters (below)
(gdb) bt
#0  main at src/main.nabi:5
...
```

### 格式化器

工具鏈隨附供 lldb、gdb 與 natvis（Visual Studio、WinDbg、cdb）使用的格式化器，把標準函式庫的值顯示為值：

- `str` 顯示為其文字（`"hello"`）；
- `List`、`Slice`、`Set` 與 `SortedSet` 顯示為 `size=N`，元素作為子項目，`Dict` 與 `SortedDict` 顯示為 `size=N`，每個值作為以其鍵命名的子項目，全部依迭代順序排列（插入順序，排序者則為鍵的順序）；
- `Array` 顯示為 `shape=[..]`，元素依列優先順序排列，以索引命名（`[1, 0]`）。

gdb 僅憑 DWARF 就只顯示和型別、列舉或 `any` 的作用中變體。lldb 沒有針對該單元之語言的外掛，會列出每個變體；其格式化器只顯示作用中的變體（其名稱作為摘要，其欄位作為子項目；裝箱的 `any` 酬載經由其 box 顯示）。在被指派之前讀取的值（殘留的槽位），在其長度不可能是它自己的長度之處顯示為 `<invalid>`，子項目最多顯示 10000 個。

| 偵錯工具 | 檔案 | 載入方式 |
|---|---|---|
| lldb | `lib/nabi/lldb/nabi.py` | `nabic --print lldbinit` 印出 `command script import` 行 |
| gdb | `lib/nabi/gdb/nabi.py` | `nabic --print gdbinit` 印出 `source` 行 |
| Visual Studio、WinDbg、cdb | `lib/nabi/natvis/nabi.natvis` | `nabic --print natvis` 印出路徑 |

兩個 Python 格式化器都從 `lib/nabi/debug/layout.py` 讀取集合的配置，並把標準函式庫的 `/nabi/library` 原始碼路徑對應到工具鏈的函式庫原始碼。natvis 檔案在 `SortedDict` / `SortedSet` 的樹最多兩層深時（最多 1023 個項目）列出其內容，更大的則只顯示其大小；它沒有和型別的檢視（偵錯工具會顯示標籤與每個變體）。

```
lldb -O "$(nabic --print lldbinit)" ./target/debug/my_binary
nabic --print lldbinit >> ~/.lldbinit
gdb -iex "$(nabic --print gdbinit)" ./target/debug/my_binary
nabic --print gdbinit >> ~/.gdbinit
cdb -c ".nvload $(nabic --print natvis)" my_binary.exe
```

在 Visual Studio 中，把該檔案複製到 `%USERPROFILE%\Documents\Visual Studio 2022\Visualizers`。

### macOS / Windows 偵錯工具

每個平台的原生偵錯工具都能處理 Nabi 二進位檔。

- 在 macOS 上，使用 lldb 或 Xcode 偵錯工具。
- 在 Windows 上，使用 Visual Studio 偵錯工具或 WinDbg（它們讀取 PDB：函式、原始碼行、參數與區域變數）。

### panic 回溯

`NABI_BACKTRACE=1` 或 `full` 會讓 panic 印出呼叫堆疊（[4.14.13](../004-functions/414-throws.md#41413-panic-與致命錯誤)）。它不需要 `-g`：執行環境從對執行環境之 panic 進入點（`nabi_panic`、`nabi_panic_at`、`nabi_panic_str`、`nabi_try_failed`）的呼叫開始，走訪 nabic 所發出之每個函式都保留的框架指標，並依 Nabi 的連結器寫進每個執行檔的函式表（`.nabi_symbols`、`__TEXT,__nabi_symbols`，或在 Windows 上是 `.rdata` 的一部分）為每個回傳位址命名：每個存活之 Nabi 函式的位址範圍與 Nabi 路徑，依位址排序。執行環境的字組 `nabi_backtrace_symbols` 容納該表相對於它自己的偏移，因此該表不需要重定位。

- 被內嵌進其呼叫者的函式沒有自己的框架，因此最佳化建置顯示的框架較少。被合併的複本（[9.2](902-aot-pipeline.md)）顯示留下來的那份複本的名稱。
- 走訪在第一個不是 Nabi 程式碼的框架處停止：C 程式碼與執行環境不保留它能跟隨的框架指標。
- 沒有回溯，只有一行 `note:`：由另一個連結器（`NABI_CC`）連結的程式沒有該表，而在執行環境內部（不是在來自編譯後程式碼的呼叫處）引發的 panic 沒有可以開始的框架。
- 目標：powerpc64le 以外的每個原生目標。wasm32 程式碼無法讀取自己的呼叫堆疊，而沒有作業系統的目標沒有環境；兩者都不會印出回溯。

## 9.9.7. 效能分析工具整合

### perf（Linux）

CPU 效能分析可以用 `perf` 進行。

```
perf record ./target/release/my_binary
perf report --stdio | nabi demangle
```

`perf report` 從符號表為函式命名，其中 Nabi 函式是 `_Y` 符號（[9.5](905-symbol-mangling.md)）；不帶引數的 `nabi demangle`（[9.9.3](#993-符號解修飾)）會把每個都換成其 Nabi 名稱。從 `-g` 建置之偵錯資訊為函式命名的效能分析工具會直接顯示 Nabi 路徑（`geom::distance`）。

### Tracy / Chrome trace（選用）

經由標準函式庫的追蹤介面，可以產生與 `tracy` 或 Chrome 的 `chrome://tracing` 相容的追蹤。具體函式庫的選擇不在本規格的範圍內，由另外的函式庫規格涵蓋。

## 9.9.8. 記憶體 / GC 工具

執行環境讀取下列環境變數。

| 變數                      | 效果                                                                         |
|---------------------------|------------------------------------------------------------------------------|
| `NABIR_GC_LOG=1`           | 每次回收在 stderr 印出一行：種類（minor / full）、暫停時間、標記數量與清除結果（存活位元組 之前 -> 之後），然後是以微秒表示的暫停各部分（到達安全點的時間、標記、清除、疏散）、執行緒數量，以及自回收器啟動以來的時間。並行的完整回收分別記錄其開始暫停（一次 minor 回收；其標記數量是 minor 的）與其結束暫停。到達安全點的時間是從停止請求到其他每個執行緒都靜止下來的等待時間：數值很長表示有執行緒拖住了停止，而不是回收很慢 |
| `NABIR_GC_STATS=1`         | 結束時在 stderr 印出回收次數（minor / full）、暫停時間與回收器的 CPU 時間。第二行給出區塊數量 / 常駐大小 / 疏散次數 / 移動的位元組，第三行給出完整暫停、並行標記（以及由配置在暫停中完成的那些）、標記器的 CPU 時間與標記協助（[7.5.4](../007-runtime/705-concurrent-collection.md#754-配置與回收)）。然後是每種執行過的暫停（minor、full、並行開始、並行結束）一行：次數、p50 / p90 / p99（誤差在八分之一以內，來自直方圖）、最長與總暫停時間、最長與總到達安全點時間 |
| `NABIR_GC_CONCURRENT=auto\|always\|off` | 完整回收的並行標記。`auto` 在老物件達 32 MiB 以上，或設定了暫停目標時適用（[7.5](../007-runtime/705-concurrent-collection.md)） |
| `NABIR_GC_EVAC=always`     | 測試用。每次完整回收都進行疏散（[7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)） |
| `NABIR_GC_YOUNG=<bytes>`   | 年輕代預算：兩次回收之間確切交出的位元組數（預設：每個配置中的執行緒 8 MiB，介於 32 MiB 到 128 MiB；若老位元組更多，則與老位元組一樣多，最多 128 MiB）（[7.2.8](../007-runtime/702-allocation.md#728-gc-觸發)） |
| `NABIR_GC_WORKERS=<n>`     | 在 stop-the-world 暫停中於回收器旁一起標記的輔助執行緒（預設：第一個之外每個上線的 CPU 一個，最多 7 個；`0`：無）（[7.3.3](../007-runtime/703-gc-algorithm.md#733-標記步驟)） |
| `NABIR_GC_STRESS=N`        | 測試用。每 N 次配置回收一次；一旦自上次完整回收以來已配置了老世代的十六分之一，其中每 8 次就有一次是完整回收 |
| `NABIR_HEAP_DUMP=on_oom`   | OOM 時，在 stderr 印出總量、依位元組排序的前 32 種型別，以及最大的 32 個大型物件。盡力而為：它不配置任何東西，也不取得任何鎖。格式由執行環境定義 |
| `NABIR_WORKERS=<n>`        | 執行器工作者的數量 N（預設：上線的 CPU 數）（[7.6.8](../007-runtime/706-scheduler.md#768-thread-per-core-選項)） |
| `NABIR_THREAD_PER_CORE=1`  | thread-per-core：任務固定在其工作者上，不竊取（7.6.8） |
| `NABIR_DYNAMIC_WORKERS=1`  | 閒置的工作者會停駐，並在每個作用中的工作者都忙碌時被取回（[7.7.8](../007-runtime/707-io-and-timer.md#778-工作者數量的動態調整)） |

未設定或為空的變數採用其預設值。所列值以外的值會在 stderr 上印出一行警告，指出該變數、該值與可接受的值，而該變數採用其預設值。

## 9.9.9. REPL / Jupyter 偵錯

| 輔助                 | 效果                                                                 |
|----------------------|----------------------------------------------------------------------|
| `<expr>?`            | 對運算式求值並印出其 `Debug` 形式（[9.7.7](907-jit-cells.md#977-視覺化--輸出)） |
| `it`                 | 運算式儲存格最後顯示的值；使用者的 `it` 勝出（[9.7.7](907-jit-cells.md#977-視覺化--輸出)） |
| `:type <expr>`、`:t` | 印出運算式的靜態型別（不執行它）                                     |
| `:doc <name>`        | 印出名稱的簽章與文件註解，即 Jupyter `inspect_request` 的文字（[9.8.6](908-jupyter-integration.md#986-inspect_request說明)）；方法寫成 `:doc xs.len` |
| `:time <expr>`       | 把運算式作為儲存格求值，並在 stderr 上印出每個階段的時間（與 `NABI_REPL_TIME=1` 相同）以及總時間 |
| `:vars`              | 以 `name: type` 列出繫結，然後是各項目的簽章                         |
| `:load <file>`       | 把程式載入執行中的工作階段（[9.7.10](907-jit-cells.md#9710-載入程式repl-path)）；其項目取代同名的項目。由 `:save` 寫出的腳本會一個一個地再次執行其儲存格 |
| `:reload`            | 再次載入最後載入的程式                                               |
| `:edit`              | 以 `$VISUAL` 或 `$EDITOR`（否則為 `vi`；在 Windows 上為 `notepad`）開啟一個暫存檔，然後把其內容作為儲存格執行；失敗的儲存格之後，下一次 `:edit` 會再次開啟該文字 |
| `:save <file>`       | 把工作階段寫成腳本：各項目，然後是已執行的儲存格，依輸入時的樣子；再次宣告的名稱在那裡也是重新宣告（[5.1.4](../005-modules/501-variable-declaration.md#514-同一作用域中的重新宣告)）。只有在之後的儲存格讀取 `it` 之處才宣告它。第一行是 `// nabi repl session (:save); :load replays it cell by cell`，每個儲存格之前有一行 `// :cell`：`:load` 該腳本會一個一個地再次執行儲存格，因此再次宣告的名稱保留其最後的繫結；`nabi run` 把它當作程式執行 |
| `:opt [<level>]`     | 印出，或為之後的儲存格設定，最佳化等級（`0`、`1`、`2`、`3`、`s`，與 `-O` 相同） |
| `:reset`             | 把工作階段重設為其初始狀態（會再次插入函式庫路徑的 `use`）           |
| `:add <dep> ..`      | 新增相依套件（`nabi install` 到目前的環境，或在套件的工作階段中 `nabi add`），並把它們附加到工作階段（[9.10.5](910-nabi-frontend.md#9105-環境)） |
| `:help`、`:h`        | 列出每個指令與別名                                                   |
| `:quit`、`:q`、`:exit` | 結束（輸入結束也會結束）                                           |
| `exit`、`quit`、`exit()`、`quit()` | 單獨位於儲存格中時結束；除非工作階段定義了該名稱（繫結、項目或 `use`），此時以定義為準 |
| `NABI_REPL_TIME=1`    | 每個儲存格的各階段耗時，以及新編譯的函式數量 / 從先前儲存格重複使用的函式數量（[9.6.4](906-jit-engine.md#964-重複使用函式庫程式碼)） |

要查看儲存格的 nabi-mir，請把相同的程式碼放進 `main.nabi`，並使用 `nabic build --emit=mir`（9.9.1）。

## 9.9.10. meta 區塊偵錯

你可以檢查[使用者定義之 meta 區塊](../006-meta-system/608-meta-blocks.md)的 derive / 屬性產生了什麼程式碼。

```
nabic build --emit=expansions    # → target/<profile>/build/<name>.expanded.nabi
```

這個選項把使用者套件之 [meta 展開](../006-meta-system/609-template-tokens.md)的結果以原始碼文字印出。每次展開所產生的原始碼都接在如下的註解之後；當產生的程式碼再次被展開時，其結果接在外層展開之後。

```
// expanded from this #[derive(MyHash)] (src/geom.nabi:12:1)
impl MyHash for Point { ... }
```

要以 Nabi 原始碼查看所有 meta 展開與去糖之後的完整 AST，請使用 `--emit=source`（`<name>.ast.nabi`）。

## 9.9.11. 效能測試

微效能測試以下列指令執行。

```
nabi bench
```

`nabic bench` 建置效能測試 harness，`nabi bench`（[9.10](910-nabi-frontend.md)）執行它（[9.1.2](901-nabic-cli.md#nabic-bench效能測試)），在 `-O2` 建置中一次一個地量測標示了 `#[bench]` 的函式。每個都接受一個 `Bench` 控制代碼，並經由 `b.iter` 對一個閉包計時，因此閉包外的準備工作不會被量測。量測方式如下：

1. **時鐘。** 一個單調的奈秒時鐘。其固定開銷藉由每個樣本計時 `iters` 次呼叫來攤提，絕不只計時單一呼叫。
2. **校準。** 除非給出 `--iters`，否則會提高 `iters`，直到一個樣本大約需要 100 ms，讓時鐘的量化誤差與開銷可以忽略。
3. **暖身。** 執行大約一個樣本時間份量的呼叫並捨棄其結果，讓快取、分支預測器與分頁穩定下來。
4. **取樣。** 取 `--samples` 個樣本（預設 10）；每個樣本記錄 `iters` 次呼叫的總時間，每次呼叫的時間就是該總時間除以 `iters`。
5. **統計。** 計算各樣本的中位數、四分位數與中位數絕對偏差（MAD）。位於 Tukey 圍籬（四分位數之外 1.5 IQR）之外的樣本被算作離群值。回報的數字是所有樣本的**中位數（+/- MAD）**，偶爾出現的慢樣本幾乎不會影響它。
6. **察覺 GC。** 在每個樣本前後讀取執行環境的回收計數器，並計算期間執行了回收的樣本數。當任一計數不為零時，人類可讀的那一行以 `[<n> outliers, <m> gc]` 結尾；`--format=nabibench` 一律給出兩者。

`black_box(x)` 是最佳化器無法看穿的恆等函式；把它套用到被計時之閉包的輸入與結果，好讓工作既不會被提升到外面，也不會被摺疊掉。`b.set_bytes(n)` / `b.set_items(n)` 把每次呼叫的時間轉換成吞吐量（`MiB/s`、`Melem/s`）。

無論是否輸出到終端機，輸出都是對齊的，並使用人類易讀的單位；`--format=nabibench` 給出供回歸工具使用、每個效能測試一行的穩定機器形式（[9.1.2](901-nabic-cli.md#nabic-bench效能測試)），`--format=json` 則給出 JSON 紀錄（[9.9.12](#9912-ci--測試整合)）。從未呼叫 `b.iter` 的效能測試會被回報為沒有量測到任何東西，這是警告而不是失敗。`--iters` / `--samples` 調整取樣。

## 9.9.12. CI / 測試整合

### CI 模式

在 CI 環境中，建議使用機器可讀的輸出。

```
nabic build --message-format=json --no-color > build.log
```

這個輸出適合 CI 系統剖析。

### 測試與效能測試的紀錄

`nabi test --format json` 與 `nabi bench --format json` 在 stdout 上每行寫出一個 JSON 物件。人類可讀的文字、測試的輸出，以及效能測試自己印出的任何東西都輸出到 stderr，因此 stdout 只容納紀錄。每筆紀錄都有 `schema`（目前是 `1`）、`type` 與 `kind`（`test` 或 `bench`）。

```json
{"schema":1,"type":"case","kind":"test","name":"math::adds","result":"ok","ms":3.215}
{"schema":1,"type":"case","kind":"bench","name":"sum","result":"ok","ms":270.1,"median_ns":6.833,"mad_ns":0.167,"samples":10,"outliers":0,"gc":0,"bytes":0,"items":0}
{"schema":1,"type":"case","kind":"test","name":"net::downloads","result":"ignored","ms":0.002,"reason":"needs the network"}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"ignored":1,"filtered_out":0,"ms":120.4}
```

| `type`    | 欄位                                                                |
|-----------|---------------------------------------------------------------------|
| `case`    | `name`；`result`：`ok`、`failed`、`ignored`（`#[ignore]` 測試或文件區塊、沒有量測到任何東西的效能測試）或 `timeout`（在其時間限制被終止的測試）；`ms`，其行程的實際耗時；`reason`，有的話（失敗的原因，或忽略的原因）。`result` 為 `ok` 的效能測試還有 `median_ns`、`mad_ns`、`samples`、`outliers`、`gc`、`bytes` 與 `items`，與 `#nabibench` 行相同（[9.1.2](901-nabic-cli.md#nabic-bench效能測試)） |
| `summary` | `result`（`ok` 或 `failed`）、`ms`，以及各計數：測試為 `passed`、`failed`、`ignored`、`filtered_out`；效能測試為 `run`、`failed`、`ignored`、`filtered_out` |
| `listed`  | `--list` 所保留之每個案例的 `name`，被忽略的測試還有 `ignored`（`true`）；之後沒有摘要（[9.10.1](910-nabi-frontend.md#9101-子指令與全域選項)） |

每個 harness 的案例之後都有一個摘要，因此兩個根檔案都有的套件在其函式庫的執行通過時會有兩個摘要。紀錄以它們自己的編號（目前是 `1`）遵循 JSON 診斷的 `schema` 規則（[9.9.4](#994-診斷--錯誤訊息)）：讀取者忽略它不認識的欄位。有 `--message-format=json` 時，建置的診斷是同一個 stdout 上的 JSON 行；它們的 `type`（`error`、`warning`……）把它們與紀錄區分開來。

### 結束碼

[`nabic` CLI 介面](901-nabic-cli.md)的結束碼定義如下。

| 碼    | 意義             |
|-------|------------------|
| `0`   | 成功             |
| `1`   | 使用者程式碼錯誤 |
| `2`   | CLI 錯誤         |
| `70`  | nabic 內部 panic |

`nabic test` 建置 harness；它不執行測試。`nabi test` 執行它們，若有任何失敗則以 `1` 結束。每個測試都在獨立的行程中執行，因此測試的 panic（`101`）不會變成執行器的 `101`。

CI 可以依結束碼採取不同的處理（重試、通知等）。

## 9.9.13. LSP 伺服器

`nabi lsp` 是 LSP（Language Server Protocol）伺服器（[9.10.8](910-nabi-frontend.md#nabi-lsplsp-伺服器)）。同一個伺服器也以獨立的二進位檔 `nabils` 提供。它經由 stdio 通訊，並提供診斷、懸停提示（包括成員）、跳至定義（回傳定義名稱的範圍）、參照、重新命名、文件符號、成員 / 路徑補全、簽章說明、快速修正、格式化與語意語彙單元（巨集呼叫、使用者定義的關鍵字、字面值後綴、運算子與屬性，這些是靜態文法無法辨識的）。參照與重新命名限於緩衝區內。

```
nabi lsp
nabils
```

在這個模式下，VS Code、Vim 與 Emacs 等 LSP 用戶端會得到補全、懸停提示、跳至定義、診斷等功能。在內部，它使用連結進 `nabi` 之編譯器的增量前端，也就是 `nabic check` 所執行的那一個。

- **補全。** 在 `x.` 之後，是 `x` 之型別的欄位與方法。對來自標準函式庫或其他套件的型別，只提供 `pub` 者，不含 `#[doc(hidden)]` 成員，也不含運算子 trait 的方法（寫成 `a + b`、`xs[i]` 等的那些）。在 `a::` 之後，包括在 `use a::b::` 中，是 `a` 之下的模組與項目。在其他地方，是游標處範圍內的區域變數、緩衝區的項目、不需路徑即在範圍內的名稱（core 函式庫的名稱以及 `use` 項目所匯入的名稱）與關鍵字。REPL 與 Jupyter 核心以相同的引擎補全（[9.7.8](907-jit-cells.md#978-輸入單位repl-的辨識)）。
- **懸停提示與定義。** 在函式、方法或欄位（包括標準函式庫的）上懸停，會顯示其簽章，然後是其文件註解。對 `x.name` 跳至定義會在 `x` 的型別中查找 `name`；對標準函式庫的項目，會前往函式庫原始碼中的該項目。
- **簽章說明。** 在呼叫的括號內（在 `(` 與 `,` 時請求），顯示被呼叫者的簽章與文件註解，游標下的參數為作用中。方法在其接收者的型別中查找，其 `self` 不計入。
- **快速修正。** 帶有編輯的診斷（例如 ``convert with `as i32` ``）把它作為 `quickfix` 程式碼動作提供。沒有編輯時，說明該寫什麼的說明行會變成一個：``add `;` `` 在診斷範圍的結尾插入 `;`，``add `use pkg::a::b;` `` 把該行插入在檔案最後一個頂層 `use` 之後（沒有的話，插入在其開頭的 `//!` 行與內部屬性之後），而 ``nabi add <name>`` 會變成指令 `nabi.add`。收到 `workspace/executeCommand` 時，伺服器在套件目錄中把 `nabi.add` 作為 `nabi add <name>` 執行（對腳本而言，是在檔案的目錄中執行 `nabi add <name> <file>`），使用由 `$NABI` 指名的 `nabi`，否則使用伺服器旁邊的那一個，然後重新解析相依套件並重新檢查開啟的緩衝區，因此不需要重新啟動就能找到新的套件。它不執行其他任何指令，也不接受任何旗標。
- **重新命名。** 當重新命名會作用於游標下的名稱時，`prepareRename` 回傳該名稱的範圍。對在緩衝區之外宣告的名稱，它會以說明位置的訊息失敗，例如 ``cannot rename `println`: it is defined in the standard library``；對關鍵字則是 `` `let` is a keyword ``。
- **文件符號。** 符號的範圍涵蓋整個項目，從其修飾詞（`pub`、`const`……）到其結尾的 `}` 或 `;`；其選取範圍是名稱。
- **輸入時即時診斷。** 診斷在緩衝區每次改變之後（`didOpen`、`didChange`）就會發布，而不只在儲存時。檢查在背景中進行，一次一個，在 30 ms 內沒有新的編輯時開始；期間到達的編輯會以最新的文字一起檢查，而在執行期間其緩衝區改變之檢查的診斷不會被發布（它們會指向較舊的文字）。它們就是編譯器自己的診斷，文字與錯誤碼相同，因此詞法問題（不可見或外觀相似的字元、單獨的組合用附加符號、註解中的雙向控制字元、格式錯誤的數字、錯誤的跳脫）以及沒有處理器接受的字面值（`10Hz`、`95%`、`dat"..."`）在輸入時就會顯示，各自附上其診斷所帶的快速修正。
- **相依套件。** 每個緩衝區都以其套件的相依套件檢查：其上方最近一個含有 `nabi.nabion`、`src/main.nabi` 或 `src/lib.nabi` 的目錄，從其資訊清單與鎖定檔解析，並以在那裡啟動的 `nabi lsp` 會建置它們的方式建置。位於所有套件之外但位於某個工作區資料夾（`workspaceFolders`，否則為 `rootUri`）內的緩衝區使用環境的函式庫（[9.10.5](910-nabi-frontend.md#9105-環境)）；位於它們所有之外時，使用伺服器啟動所在之目錄的相依套件。每個套件只解析一次；儲存的資訊清單或鎖定檔（`didSave`）、編輯器回報的這類檔案的變更（`workspace/didChangeWatchedFiles`，用戶端允許時，伺服器會為此註冊 `**/nabi.nabion` 與 `**/nabi.lock`），以及工作區資料夾的變更，都會重新解析並重新檢查開啟的緩衝區。其相依套件無法解析的套件會在沒有它們的情況下檢查，伺服器會說明原因一次（`window/showMessage`）。
- **檢查進行中時的回應。** 懸停提示、跳至定義、補全、簽章說明、文件符號與格式化不會等待編輯的檢查：它們從最後一次完成型別檢查的檢查回應，並透過此後的編輯對應位置。位於該檢查之後被編輯之文字上的位置（包括游標下的名稱），則會等待對目前文字的檢查，參照與重新命名也是，因為它們依緩衝區的現狀編輯。在 `x.` 之後的補全，若只有正在輸入的那一行改變，且 `x` 在那裡的寫法與之前相同，就從該檢查取得 `x` 的型別；否則它會檢查緩衝區到 `.` 為止。用戶端在得到回應之前取消的請求（`$/cancelRequest`）會得到錯誤 `RequestCancelled`（-32800）。
- **診斷範圍。** 範圍為空的診斷會回報在該位置的語彙單元上，或在行尾時回報在其前一個語彙單元上。
- **手動啟動。** 在終端機上，伺服器會先在標準錯誤印出 ``nabils speaks the language server protocol on stdin; start it from an editor (see `nabi help lsp`)``。當輸入在用戶端傳送 `initialize` 之前就結束時，它會印出 ``nabils: input ended before the editor sent `initialize` `` 並以 1 結束。在 `initialize` 之後，依 LSP 標準所述，`shutdown` 之後的結束碼是 0，否則是 1。

詳細的 LSP 規格不在本規格的範圍內（LSP 標準 + 工具鏈的實作決策），另行定義。

## 9.9.14. 檢視套件

可以一次檢視預先建置之函式庫套件的資訊。

```
nabi inspect path/to/libfoo.a        # or foo.nabihir; the sibling file is read too
```

這個指令依下列方式顯示來自 NABIHIR（[9.4](904-nabihir-bytecode.md)）與[封存檔](../000-meta/007-glossary.md#封存檔archive)的資訊。

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

`nabi inspect --dump <file>` 印出 NABIHIR 的版本、產生者、meta 以及所有介面檔案（9.9.2）。

這讓人一眼看出套件對外公開了什麼。

## 9.9.15. 總結

上述工具是 Nabi 所搭配的標準工具（gdb、lldb、perf、Jupyter）以及 Nabi 自己的工具（`--emit`、`nabi demangle`、`nabi inspect`、LSP 伺服器）。前端 `nabi` 及其工具定義於 [9.10](910-nabi-frontend.md)，nabi-mir 檔案格式定義於 [9.11](911-mir-format.md)。
