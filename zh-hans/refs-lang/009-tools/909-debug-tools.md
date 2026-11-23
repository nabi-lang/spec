# 9.9. 调试 / 诊断工具

本节定义 `nabic` 和 `nabi` 提供的调试 / 诊断工具，以及它们与外部工具的集成。

## 9.9.1. 中间输出：`--emit`

可以转储 [AOT 构建流水线](902-aot-pipeline.md)每一步的结果。

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

`mir` 是代码生成器所编译的 nabi-mir：在 `-O0` 以上的级别，是优化之后的结果（[9.2.10](902-aot-pipeline.md#9210-优化级别)）。`obj` 是它写出的目标文件。`objdump -d` 这样的反汇编器可以显示 `obj` 的机器码。

`asm` 是以汇编源代码形式表示的目标文件，与 `clang -S` 写出的形式相同：为目标汇编它（`clang -c`、`llvm-mc -filetype=obj`）得到的目标文件，与 `obj` 有相同的代码字节、在相同偏移处指向相同位置的同类型重定位以及相同的符号，因此链接和运行的结果也相同。它从 `nabic` 写出的目标文件读回，采用目标的 GNU / LLVM 语法（在 x86_64 和 x86 上是 `.intel_syntax noprefix` 下的 Intel 语法）。每个节都带有按目标文件格式命名的 `.section` 指令（ELF 的 `.text.<symbol>`、`.rodata.<symbol>`；Mach-O 的 `__TEXT,__text`；COFF 带有选择方式和 leader 的 `.text` COMDAT，以及与其函数关联的 `.pdata`）和它的 `.p2align`。符号带有其绑定和可见性（`.globl`、`.weak`、`.hidden`；ELF 上的 `.type` 和 `.size`；Mach-O 上的 `.private_extern` 和 `.weak_definition`；COFF 上的 `.def`；powerpc64 上的 `.localentry`）。每个函数就是代码生成器所编码的那些指令：分支目标是局部标签（`.LBB0_3`；在 Mach-O 和 x86 Windows 上是 `LBB0_3`），每个被重定位的字段都是让汇编器发出该重定位的表达式（`call nabi_panic_at@PLT`、`[rip + x@GOTPCREL]`、`[rip + x@GOTTPOFF]`、`[rcx + x@SECREL32]`、`[ebx + s@GOTOFF]`、`adrp x0, s@PAGE` / `:lo12:s`、`x@TLVPPAGE`、`#:lower16:(s-(.+16))`、`.word x(GOT_PREL)+0x1c`、`%pcrel_hi(s)` 配合 `%pcrel_lo(.Lpcrel_hi0)`、`s@toc@ha`、`x@got@tprel@l`、`brasl %r14, f@PLT`、`x@INDNTPOFF`、..）。如果某种编码不是汇编器根据指令文本会选择的编码，则在语法允许处强制指定（长分支上的 `{disp32}`、`{disp8}`、`{rex}`，以及表示另一种 `mov` 形式的 `mov.s`）；否则该指令写成它的字节，文本放在注释中（x86 的栈帧建立指令 `sub rsp, imm32`，其立即数可以放进 8 位）。数据（字符串字面量、静态变量、GC 类型描述符、扫描表、虚表）是 `.asciz`、`.byte`、`.zero` 以及指明其目标的指针大小指令（`.quad`、`.long s-.`、`.rva`、`.secrel32`、`.secidx`）。展开表是汇编器据以构建它们的指令，每条紧跟在它所描述的指令之后：ELF 的 `.eh_frame`（汇编器的 CIE 即 `nabic` 的 CIE）以及 Mach-O arm64 的 compact unwind（汇编器根据这些指令所描述的帧记录和寄存器对推导出它）使用 `.cfi_startproc`、`.cfi_def_cfa_offset`、`.cfi_def_cfa_register`、`.cfi_offset` 和 `.cfi_endproc`；x64 COFF 的 `.pdata` / `.xdata` 使用 `.seh_proc`、`.seh_pushreg`、`.seh_stackalloc`、`.seh_setframe`、`.seh_savereg`、`.seh_savexmm`、`.seh_endprologue` 和 `.seh_endproc`；32 位 ARM 的 `.ARM.exidx` 使用 `.fnstart`、`.save`、`.setfp`（或 `.cantunwind`）和 `.fnend`。汇编器生成的表解码结果相同（`llvm-dwarfdump --eh-frame`、`llvm-readobj --unwind`、`llvm-objdump --unwind-info`）。arm64 Windows 把它的 `.pdata` / `.xdata` 作为带重定位的数据保留：在 `nabic` 逐条写出展开代码的地方，汇编器会把它们打包（打包的 `.pdata` 条目、与序言共享代码的尾声）。`-g` 的调试节（DWARF、CodeView）同样是数据，32 位 ARM 的构建属性是 `.eabi_attribute` 指令。RISC-V 源代码以 `.option arch`、`.option norvc` 和 `.option norelax` 开头；16 位压缩形式的指令（`-O0` 以上）写成其 `c.` 助记符（`c.addi sp, -0x10`、`c.beqz a0, .LBB0_3`），位于 `.option rvc` 之下，并在下一条 32 位指令之前切回 `.option norvc`；它的 ELF 标志仍来自汇编器的选项（`clang --target=riscv64-linux-gnu`，或 `llvm-mc -mattr=+m,+a,+f,+d,+c -target-abi=lp64d`）。汇编器可能写得不同、但含义相同的部分有：针对局部符号所在节、带该符号偏移的重定位；它总会添加的节符号、映射符号（`$x`、`$d`）和 Mach-O `ltmp` 符号；作为符号保留的 RISC-V 局部标签；x86_64 `.eh_frame` 的 `SHT_X86_64_UNWIND` 类型；在节内解析的 Mach-O 函数对自身的调用及其 compact unwind 条目；32 位 ARM 的硬浮点 ELF 标志，它没有指令可以设置（由构建属性表明）；COFF 弱外部符号的默认符号名称（`.weak.<symbol>.default`，带汇编器添加的后缀）；展开表的布局：在 `nabic` 共享一个 `UNWIND_INFO` 的地方，汇编器为每个函数生成一个 x64 `UNWIND_INFO`，以及 32 位 ARM 短形式的 `pop {r4-r11, lr}`。

在 wasm32 上，源代码是 LLVM 的 WebAssembly 汇编，供 `clang --target=wasm32-wasi -c` 或 `llvm-mc -triple=wasm32-wasi -filetype=obj` 使用。指令所需的特性来自汇编器的选项，因为没有指令可以设置它们：clang 使用 `-msimd128`（在 `wasm32-wasip1-threads` 上还有 `-matomics`），`llvm-mc` 使用 `-mattr=+bulk-memory,+mutable-globals,+nontrapping-fptoint,+reference-types,+sign-ext,+simd128`（以及 `+atomics`）。源代码声明它用到的全局变量、表和每个函数（`.globaltype __stack_pointer, i32`、`.tabletype __indirect_function_table, funcref`、`.functype f (i32, i64) -> (f64)`），以另一个名称导入的函数还带有 `.import_module` 和 `.import_name`。每个函数位于自己的节中（`.section .text.<symbol>,"",@`），带有其绑定（`.globl`、`.weak`、`.hidden`、`.export_name`）、`.functype` 和 `.local` 行；在它的指令中，每个 `end` 按其所闭合的结构命名（`end_block`、`end_loop`、`end_if`、`end_function`），无类型的 `select` 按其操作数的类型命名（`i32.select`），每个被重定位的操作数都是它的符号：`call f`、`global.get __stack_pointer`、`i32.const s`（数据地址，或函数的表索引）、`i32.const s@TLSREL`、`call_indirect __indirect_function_table, (i32) -> (i32)`。每个数据段是一个单独的节（`.section .data.<symbol>,"",@`，线程局部时为 `"T"`），带有它的 `.p2align`、它的符号（`.type s,@object`、`.size`），每个指针写成 `.int32 s`；调试节和 `target_features` 是数据，代码偏移写成 `.int32 f+0x2b`。汇编器生成的目标文件具有相同的函数、代码（被重定位的操作数长 5 个字节，与这里相同）、重定位、数据段、符号和自定义节。它写得不同、但含义相同的部分有：函数索引和类型索引的顺序、内存导入和表导入的初始大小、它在 `__indirect_function_table` 上设置的 `NO_STRIP` 标志，以及它的 `DATACOUNT` 节。

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

`listing` 是以文本表示的机器码，由代码生成器根据它所编码的指令写出，因此其字节与 `obj` 的完全一致。每个函数以其符号（`<name>:`）开头；每条指令占一行，包括它相对函数起点的偏移、它的字节和它的文本，与 `llvm-objdump -dr` 为该目标打印的形式相同（在 x86_64 和 x86 上为 Intel 语法）。代码生成器的标签位于它所标记的指令之前（`.L3:`），分支目标写出其标签（`jne 0x1c <.L3>`），每个重定位紧跟在其字段所属的指令之后，并使用目标文件格式为它取的名称（`R_X86_64_PLT32`、`ARM64_RELOC_BRANCH26`、`IMAGE_REL_AMD64_REL32`、`R_386_PLT32`、`R_ARM_CALL`、`R_RISCV_CALL_PLT`、`R_PPC64_REL24`、`R_390_PLT32DBL`、`R_WASM_FUNCTION_INDEX_LEB`、..）；在 riscv 上，`R_RISCV_PCREL_LO12_I` 指明目标文件在其 `auipc` 处的标签（`.Lpcrel_hi0`），16 位指令显示为 4 个十六进制数字，并附上它所代表的 32 位指令的文本。在 wasm32 上，偏移从函数的符号（即其函数体大小）开始计数，`.local` 行列出它的局部变量，调用和全局变量显示目标文件给它们的索引，控制流注释沿用 objdump 的形式（`# 1: down to label3`），其标签编号在整个目标文件中连续递增。

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

输出写到以下位置。

```
target/debug/build/foo.ast.nabi
target/debug/build/foo.hir
target/debug/build/foo.mir
target/debug/build/foo.o
target/debug/build/foo.s
target/debug/build/foo.lst
target/debug/build/foo.expanded.nabi
```

要一次选择多个步骤，用逗号分隔，例如 `--emit=hir,mir,obj`。只选一种时，`-o <path>` 可以改变输出位置。使用 `--emit` 时，不生成可执行文件或归档文件。

## 9.9.2. NABIHIR 检查

```
nabi inspect --dump path/to/foo.nabihir   # or libfoo.a; reads the sibling foo.nabihir
```

该命令以人类可读的形式打印预构建库的 [NABIHIR](904-nabihir-bytecode.md)。在版本、生成者和元信息之后，每个接口文件依次以 `--- <path>` 和它的文本（具体函数体被删减为 `;` 的源代码）给出。移交摘要列在 `--- summaries` 下，调用摘要列在 `--- calls` 下，每个函数一行，列出成立的事实（[9.4.7](904-nabihir-bytecode.md#947-主体)）。

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

这一输出可用于验证 [NABIHIR](../000-meta/007-glossary.md#nabihir-nabi-hir-bytecode) 的正确性和兼容性。

## 9.9.3. 符号还原（demangle）

```
nabi demangle [-v] <symbol>...
nabi demangle _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190
nabi demangle < symbols.txt
```

该命令把[符号修饰](905-symbol-mangling.md)产生的每个修饰后符号打印为源代码中书写的名称，每行一个。标准库路径以其层级（`core` / `std` / `bare`）开头。方法（`<type>.<method>`，或泛型类型实例的 `<type>__<method>`）显示为 `<type>::<method>`，按 trait 选中的方法（`__by_<trait>`）显示为 `<path::type as trait>::<method>`，以哈希命名的 impl 块中的方法显示为 `{impl}::<method>`。typehash 被省略。Mach-O 形式（多一个前导 `_`，即 `__Y...`，与 `nm` 打印的一致）以及符号之后的 `.<suffix>` 也可以接受。

```
$ nabi demangle __Ya61f26f1_core_collections_dict_Dict_0_0grow_ce3f36e15e552a70
core::collections::dict::Dict::grow
```

使用 `-v` 时，每个符号显示为由它所编码的各部分组成的一个块，块之间用空行分隔：

```
package:  3f8a1c2b
path:     geom
name:     distance
generics: a7c91d8e44b2f190 (type tuple hash)
```

只要有一个实参不是 Nabi 符号，退出码就是 `1`。不给出符号时，`nabi demangle` 是一个过滤器：它把标准输入复制到标准输出，其中每个 Nabi 符号都替换为它的单行名称，其余内容保持不变，因此可以把性能分析器、`nm` 和崩溃输出通过管道传给它：

```
perf report --stdio | nabi demangle
nm target/debug/my_binary | nabi demangle
```

## 9.9.4. 诊断 / 错误消息

### 标准格式

诊断消息按以下标准格式打印。

```
error[E0027]: cannot find value `count` in this scope
  --> src/main.nabi:5:13
   |
 5 |     let x = count + 1;
   |             ^^^^^
   |
   = help: did you mean `counter`?
```

该格式使用[源文件](../001-lexical/101-source-files.md)中“文件 + 行 + 列”的诊断单位。

头部是 `<severity>[<code>]: <message>`，`--> path:line:col` 给出主要位置。主要范围用 `^` 标出下划线，次要范围用 `-`；位于另一个文件中的次要范围以 `::: path:line:col` 引出。末尾的 `= note:` 行补充上下文，`= help:` 行给出修改建议。如果找不到的名称在别处有同名的项，help 会给出引入它的 `use` 行（最多三条：先是本包自己的项，然后是标准库的，然后是依赖的）；如果本包中唯一这样的项被隐藏，则给出能让该使用处看到它的可见性以及对应的 `use`；包自身的项按其从包根开始的路径导入（``add `use pkg::geo::Point;` ``；`` `geo::secret` has that name, but it is private to module `geo`: write `pub(pkg)` before it, so its whole package sees it, and add `use pkg::geo::secret;` ``）。

主要范围是出错的代码：对于类型错误的值，是这个值，而不是包含它的语句。消息按书写形式称呼库中的项（`List::sort`、`Send`、`print`，而不是 `core::marker::Send`）。消息不引用本规范的章节；应当改写成什么，写在消息、help 行和 `nabic --explain <code>` 中。

### JSON 格式（工具集成）

为便于工具集成，输出也可以生成为 JSON。

```
nabic build --message-format=json
```

每条诊断是标准输出上的一行 JSON 对象；不打印人类可读格式的诊断，标准错误只包含进度行。

```json
{"type":"error","code":"E0040","kind":"type","message":"...","spans":[{"file_name":"src/main.nabi","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"...","line_end":5,"col_end":15}],"notes":["..."],"fixes":[],"help":["..."],"lint":null,"schema":1}
```

| 字段       | 含义                                                               |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | 错误码，如 `E0040`（[错误码](#错误码)）                       |
| `kind`     | 该错误码所属的领域（`type`、`parse`、..）                          |
| `message`  | 消息                                                               |
| `spans`    | 位置列表。第一个元素是主要位置（`is_primary: true`），其余是次要位置 |
| `notes`    | note 字符串列表，不含 help 条目                                    |
| `fixes`    | 修正代码的编辑，每项是某个文件中的一个范围，带有替换它的 `text` 和一个 `title`（见下文）；常为空 |
| `help`     | help 条目（人类可读格式中的 `= help:` 行），不带 `help: ` 前缀     |
| `lint`     | 警告或 lint 错误所属的 lint（`unused_variables`），即 `#[allow(..)]` 和 `-A` 接受的名称（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）；其他情况为 `null` |
| `schema`   | 此格式的版本，目前为 `1`                                           |

范围包含 `file_name`、`line_start`、`col_start`、`byte_start`、`byte_end`、`is_primary`、`label`、`line_end` 和 `col_end`。`line_end` / `col_end` 是范围结束的位置（紧接其最后一个字符之后的位置），与 `line_start` / `col_start` 一样以码位计数；手头没有源文本时，它们为 `null`。只知道文件而不知道位置的诊断只有 `{"file_name": .., "is_primary": true}`。修正包含 `file_name`、`line_start`、`col_start`、`byte_start`、`byte_end`、`text`、`title`（``convert with `as i32` ``）、`line_end` 和 `col_end`；语言服务器把它作为快速修复提供（[9.9.13](#9913-lsp-服务器)）。

在同一个 `schema` 内，字段只会增加，不会删除、改名或改变含义，因此读取方应当忽略它不认识的字段。测试、基准测试和 `nabi deps` 的记录（[9.9.12](#9912-ci--测试集成)、[9.10.4](910-nabi-frontend.md#9104-依赖nabi-add--remove--update--deps)）遵循同样的规则，各有自己的 `schema`，目前为 `1`。

借助这一格式，IDE 和 CI 系统可以机械地读取诊断。

### 错误码

每一种错误都有自己的代码，由 `E` 和四位数字组成：类型不匹配是 `E0040`，找不到名称是 `E0027`。代码从 `E0001` 开始连续编号，没有空缺，按添加的顺序而不是按领域编号；领域由 `kind` 字段给出。lint（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）的诊断不论由编译器的哪个部分产生，都使用 lint 代码 `E0131`，并在其 note 和 `lint` 中写出该 lint 的名称。

`nabic --explain <code>`（`nabi --explain` 同样可用）打印该代码的说明页：标题、含义以及应当改写成什么（[9.1.5](901-nabic-cli.md#915-全局选项)）。`E0040`、`e0040` 和 `40` 指同一个代码；不存在的代码是用法错误（退出码 2）。文档中的错误页面就是这些页面，每个代码一页。

标题与 `nabic --explain` 打印的完全一致。

| 代码    | `kind`        | 标题                         |
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
| `E0149` | `type` | change a running program cannot take |

`kind` 的取值有 `lexical`、`parse`、`resolve`、`visibility`、`module`、`type`、`pattern`、`handoff`（值的所有权和生存期：移交之后的使用，见[标准库 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync线程边界标记)；资源在被移动或关闭之后使用，或没有关闭，见[标准库 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop释放时的清理)；线程局部值保留到 `with` 之后；流视图在其流之外使用，见[标准库 2.23.1](../../refs-standard-libraries/200-core/223-io.md#2231-read)；`SliceMut` 在其列表调整大小之后使用，见[标准库 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt可写视图)）、`meta`、`linkage`、`unsupported`、`internal`、`other` 和 `lint`。

## 9.9.5. 日志 / 进度显示

### 标准输出

默认的构建输出如下。

```
   Compiling foo (path/to/foo)
    Building target/debug/libfoo.a
    Finished in 1.23s
```

进度行写到标准错误；动词列表见 [9.1.8](901-nabic-cli.md#918-输出格式)。

### `--verbose` / `-v`

详细模式还会显示每个阶段的耗时。

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

显示每个阶段的耗时有助于诊断构建瓶颈。阶段有 `front end` 和 `codegen + link`（使用 `--emit` 时为 `emit <ext>`）。

### `NABI_TIME_PASSES`

`NABI_TIME_PASSES=1` 在标准错误上打印每个编译器 pass 的墙钟时间。

### `-vv`（非常详细）

在 `-v` 的基础上，为处理的每个文件打印一行 `Reading <file>`，并为每个被[特化](../000-meta/007-glossary.md#specialize)的（项，类型）对打印一行 `Specialized <path>(sig)`。

## 9.9.6. 调试器集成：gdb / lldb

### DWARF 调试信息

`debug` 构建配置的构建，以及带 `-g` 的 `release` 构建（[9.2.15](902-aot-pipeline.md#9215-输出)），会生成具有以下性质的 DWARF 调试信息。Windows 目标则以 CodeView 形式得到相同的信息（[见下文](#codeview-与-pdbwindows)）。

- 函数的 `DW_AT_name` 是它的 Nabi 路径（`geom::distance`）；不生成链接名称（修饰后的符号），因为调试器不理解 `_Y` 修饰。调用程序 `main` 的包装函数（[9.5.9](905-symbol-mangling.md#959-main-函数的符号)）命名为 `nabi_main`，因此栈回溯中只有一个 `main` 帧，即程序自己的。
- 行号指向原始 .nabi 文件中的行。
- 局部变量以其原始名称和类型出现。除基本类型（整数、浮点数、`bool`、`char`）之外，还为结构体（字段名；泛型结构体的实例以源代码中书写的类型实参命名，如 `core::collections::list::List<i64>`）、元组、`str`、`any`、和类型和数组生成 DWARF 类型。和类型或枚举是一个 DWARF variant part：一个人工的判别成员，以及每个变体（包括不带数据的变体）一个带 `DW_AT_discr_value` 的 `DW_TAG_variant`，每个都包含一个名为 `<Sum>::<Variant>` 的结构体。编译单元使用 `DW_LANG_Rust`，以便调试器读取 variant part（DWARF 没有 Nabi 的语言代码）。`GcBuf<T>` 引用描述为指向其对象（对象头，然后是 `T` 元素）的指针。`any` 值是一个覆盖存入其中各类型的 variant part，以存储的标签作为判别值。判别值是一个人工的 `i32`（存储的标签；`throws` 载体的 `bool` 标签是一个字节）。函数值（`fn` 类型或闭包）是一个以其类型命名的结构（`fn(i32) -> i32`），成员为 `code`（指向 `DW_TAG_subroutine_type` 的指针，先是环境，再是参数）和 `env`（它的环境；闭包类型已知时是指向环境对象的指针），因此调试器会显示目标函数。按引用捕获的变量显示为它的值。编译器生成的函数（C 的 `main`、静态变量初始化守卫）也有子程序。编译器生成的局部变量（名称以 `#` 或 `__` 开头）被省略。
- 每个块生成一个词法块作用域，因此被遮蔽的名称只在其块内可见。
- 在 `-O0` 下，每个局部变量都保有自己的栈槽，其位置就是该栈槽。
- 在 `-O0` 以上，`-g` 不改变代码（[9.2.10](902-aot-pipeline.md#9210-优化级别)）；调试信息描述的是优化后的代码。变量的位置是按代码范围给出的位置列表（`.debug_loc`）：保存其值的寄存器、它的栈槽、一个常量，或者对于原地构建的结构体或闭包，是这些的若干片段（`DW_OP_piece`）。在其值已不存在的地方，它没有位置，调试器把它显示为已被优化掉。跨越安全点持有 GC 引用的变量始终位于其栈槽中，即回收时读取并更新的根。
- 在 `-O0` 以上，被内联的函数体是一个抽象 `DW_TAG_subprogram`（带 `DW_AT_inline`）的 `DW_TAG_inlined_subroutine`（带 `DW_AT_call_file`、`DW_AT_call_line`、`DW_AT_call_column`），其中包含被调用方的参数和局部变量，因此调试器可以停在其中，并把它显示为单独的一帧。内联代码的行表行指明被调用方的文件，只有在行号变化处，行才是语句（`is_stmt`），序言之后的第一行带有 `prologue_end`。在 Apple 目标上，编译单元带有 `DW_AT_APPLE_optimized`。wasm32 以同样的方式描述内联函数体和变量。在 `-O0` 以上，保存在 wasm 局部变量中的变量有一个位置列表，在它持有该值的代码范围内指明该局部变量（`DW_OP_WASM_location 0x0 <local>, DW_OP_stack_value`；结构体则每个局部变量一个片段）；位于线性内存中的变量，在其帧槽持有其值期间，位置是该帧槽（`DW_OP_fbreg`；帧基址本身是一个 wasm 局部变量）。在 `-O0` 下，每个变量都位于其帧槽中。

### CodeView 与 PDB（Windows）

在 Windows 目标上，同样的信息以 CodeView（C13）形式存在于每个目标文件中，由链接器汇集到可执行文件旁的 PDB 中。

- `.debug$S` 在与每个函数的代码关联的节中保存：该函数的 `S_GPROC32`（以其 Nabi 路径为名称）和 `S_FRAMEPROC`；每个参数和局部变量，作为带帧指针相对位置（`S_DEFRANGE_FRAMEPOINTER_REL_FULL_SCOPE`）的 `S_LOCAL`；只有一个代码范围的词法块，作为 `S_BLOCK32`（其他块的局部变量属于外围作用域）；以及该函数的行表。目标文件的源文件列在其校验和表中。
- 在 `-O0` 以上，变量的位置按代码范围给出：`S_DEFRANGE_REGISTER` 或 `S_DEFRANGE_FRAMEPOINTER_REL`；对于字段分散存放（在寄存器中、在内存中或两者兼有）的结构体，每个字段一条记录：位于寄存器中的字段用 `S_DEFRANGE_SUBFIELD_REGISTER`，位于内存中的字段用相对帧指针、带字段偏移的 `S_DEFRANGE_REGISTER_REL`。寄存器以其 CodeView 编号命名：x64 和 x86 上的通用寄存器和 XMM 寄存器，arm64 上的 `X0`-`X30`，以及按大小区分的 `S`、`D` 或 `Q`。唯一取值为一个整数、`bool` 或 `char` 常量的变量是 `S_CONSTANT`；唯一取值为一个浮点常量的变量，是其作用域中的一个 `S_LDATA32`，指向与函数代码一起的只读数据节中的该值（dbghelp 不读取浮点 `S_CONSTANT`）；值无处可寻的 `S_LOCAL` 被标记为已被优化掉。Windows ABI 按引用传递的结构体、元组或数组参数（x64：大小不是 1、2、4 或 8 字节；arm64：超过 16 字节），在每个级别上都是不带参数标志的 `S_LOCAL`：dbghelp 把这样的参数读作指向其值的指针，而 Nabi 描述的是值本身。被内联的函数体是被调用方 `LF_FUNC_ID` 的 `S_INLINESITE`，用二进制注解把它的代码映射到它的行；目标文件唯一的 `DEBUG_S_INLINEELINES` 子节（位于主 `.debug$S` 中，在 PDB 中每个模块一个：dbghelp 只读取模块的第一个）给出每个被调用方的文件和行，函数在 `S_INLINEES` 中列出它的被调用方，它自己的行表为内联代码给出调用所在的行。内联点从不位于 `S_BLOCK32` 之内。
- `.debug$T` 保存类型：基本类型作为 CodeView 基本类型；结构体、元组、`str`、`any` 和函数值作为带字段名的 `LF_STRUCTURE`；和类型或枚举作为一个结构，包含标签以及每个变体一个位于偏移 0 的成员，每个成员都是一个名为 `<Sum>::<Variant>`、覆盖整个值的结构；`GcBuf<T>` 或装箱的引用作为指向其对象的指针；函数签名作为 `LF_PROCEDURE`。
- PDB（MSF 7.0）包含合并后的所有目标文件的类型记录（每种只有一份）、每个目标文件一个模块（带其符号和行表）、全局符号（过程引用、全局数据）和公开符号、节贡献以及节映射。自带 CodeView 的目标文件（运行时的）以同样的方式纳入。可执行文件的调试目录以路径、GUID 和 age 指明 PDB。

基于 dbghelp 的调试器（WinDbg 和 cdb、Visual Studio 调试器）可以按名称查找函数，把地址映射到文件和行，并显示参数和局部变量及其值；在优化后的代码中，还能显示内联帧（`k`；`bm <module>!<fn>` 在函数被内联的每一处设置断点），以及位于寄存器中、分散在寄存器和内存中或为常量的变量（`dv`、`dx`）。

### 使用 gdb / lldb

标准的 gdb 和 lldb 命令可以直接用于 Nabi 代码。

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

工具链为 lldb、gdb 和 natvis（Visual Studio、WinDbg、cdb）提供格式化器，把标准库的值显示为值：

- `str` 显示为其文本（`"hello"`）；
- `List`、`Slice`、`Set` 和 `SortedSet` 显示为 `size=N`，元素作为子项；`Dict` 和 `SortedDict` 显示为 `size=N`，每个值作为以其键命名的子项；全部按迭代顺序（插入顺序，有序容器按键的顺序）；
- `Array` 显示为 `shape=[..]`，元素按行优先顺序排列，以索引命名（`[1, 0]`）。

仅凭 DWARF，gdb 就只显示和类型、枚举或 `any` 的当前变体。lldb 没有针对该编译单元语言的插件，会列出每个变体；它的格式化器只显示当前变体（其名称作为摘要，其字段作为子项；装箱的 `any` 载荷通过其箱子显示）。在赋值之前读取的值（过时的槽）在其长度不可能成立时显示为 `<invalid>`，子项最多显示 10000 个。

| 调试器 | 文件 | 加载方式 |
|---|---|---|
| lldb | `lib/nabi/lldb/nabi.py` | `nabic --print lldbinit` 打印 `command script import` 行 |
| gdb | `lib/nabi/gdb/nabi.py` | `nabic --print gdbinit` 打印 `source` 行 |
| Visual Studio、WinDbg、cdb | `lib/nabi/natvis/nabi.natvis` | `nabic --print natvis` 打印路径 |

两个 Python 格式化器都从 `lib/nabi/debug/layout.py` 读取集合类型的布局，并把标准库的 `/nabi/library` 源路径映射到工具链的库源代码。natvis 文件在 `SortedDict` / `SortedSet` 的树至多两层深（最多 1023 个条目）时列出其内容，更大的只显示大小；它没有和类型的视图（调试器显示标签和每个变体）。

```
lldb -O "$(nabic --print lldbinit)" ./target/debug/my_binary
nabic --print lldbinit >> ~/.lldbinit
gdb -iex "$(nabic --print gdbinit)" ./target/debug/my_binary
nabic --print gdbinit >> ~/.gdbinit
cdb -c ".nvload $(nabic --print natvis)" my_binary.exe
```

在 Visual Studio 中，把该文件复制到 `%USERPROFILE%\Documents\Visual Studio 2022\Visualizers`。

### macOS / Windows 调试器

各平台的原生调试器都能处理 Nabi 二进制文件。

- 在 macOS 上，使用 lldb 或 Xcode 调试器。
- 在 Windows 上，使用 Visual Studio 调试器或 WinDbg（它们读取 PDB：函数、源代码行、参数和局部变量）。

### panic 栈回溯

`NABI_BACKTRACE=1` 或 `full` 使 panic 打印调用栈（[4.14.13](../004-functions/414-throws.md#41413-panic-与致命错误)）。它不需要 `-g`：运行时从对运行时 panic 入口点（`nabi_panic`、`nabi_panic_at`、`nabi_panic_str`、`nabi_try_failed`）的调用开始，沿着 nabic 生成的每个函数都保留的帧指针遍历，并根据 Nabi 的链接器写入每个可执行文件的函数表（`.nabi_symbols`、`__TEXT,__nabi_symbols`，或在 Windows 上作为 `.rdata` 的一部分）为每个返回地址命名：该表按地址排序，记录每个存活 Nabi 函数的地址范围和 Nabi 路径。运行时的字 `nabi_backtrace_symbols` 保存该表相对于自身的偏移，因此该表不需要重定位。

- 被内联到调用方中的函数没有自己的帧，因此优化构建显示的帧更少。被折叠的副本（[9.2](902-aot-pipeline.md)）显示保留下来的那个副本的名称。
- 遍历在第一个不是 Nabi 代码的帧处停止：C 代码和运行时不保留可供遍历的帧指针。
- 不输出栈回溯、只输出一行 `note:` 的情况：由其他链接器（`NABI_CC`）链接的程序没有该表；在运行时内部（而不是在编译代码的调用处）触发的 panic 没有可以开始遍历的帧。
- 目标：除 powerpc64le 之外的所有原生目标。wasm32 代码无法读取自己的调用栈，没有操作系统的目标没有环境；两者都不打印栈回溯。

## 9.9.7. 性能分析器集成

### perf（Linux）

可以用 `perf` 进行 CPU 性能分析。

```
perf record ./target/release/my_binary
perf report --stdio | nabi demangle
```

`perf report` 根据符号表为函数命名，而 Nabi 函数在其中是 `_Y` 符号（[9.5](905-symbol-mangling.md)）；不带实参的 `nabi demangle`（[9.9.3](#993-符号还原demangle)）把每个符号替换为它的 Nabi 名称。根据 `-g` 构建的调试信息为函数命名的性能分析器会直接显示 Nabi 路径（`geom::distance`）。

### Tracy / Chrome trace（可选）

通过标准库的追踪接口，可以生成与 `tracy` 或 Chrome 的 `chrome://tracing` 兼容的追踪数据。具体库的选择不在本规范范围之内，由单独的库规范涵盖。

## 9.9.8. 内存 / GC 工具

运行时读取以下环境变量。

| 变量                      | 作用                                                                         |
|---------------------------|------------------------------------------------------------------------------|
| `NABIR_GC_LOG=1`           | 每次回收在标准错误上打印一行：种类（minor / full）、停顿时间、标记数和清除结果（存活字节数，前 -> 后），然后是以微秒为单位的停顿各部分（到达安全点的时间、标记、清除、疏散）、线程数以及自回收器启动以来的时间。并发的完全回收分别记录其开始停顿（一次 minor 回收；其标记数为该 minor 回收的）和结束停顿。到达安全点的时间是从发出停止请求到其他所有线程都停下来的等待时间：该值较大意味着有线程拖延了停止，而不是回收本身慢 |
| `NABIR_GC_STATS=1`         | 退出时在标准错误上打印回收次数（minor / full）、停顿时间和回收器 CPU 时间。第二行给出块数 / 常驻大小 / 疏散次数 / 移动的字节数，第三行给出完全局停顿次数、并发标记次数（以及其中由分配在停顿中完成的次数）、标记器的 CPU 时间和标记协助（[7.5.4](../007-runtime/705-concurrent-collection.md#754-分配与回收)）。然后，对运行过的每种停顿（minor、full、concurrent start、concurrent finish）各打印一行：次数、p50 / p90 / p99（基于直方图，误差在八分之一以内）、最长和总停顿时间、最长和总的到达安全点时间 |
| `NABIR_GC_CONCURRENT=auto\|always\|off` | 完全回收的并发标记。`auto` 在老对象达到 32 MiB 或以上，或者设置了停顿目标时生效（[7.5](../007-runtime/705-concurrent-collection.md)） |
| `NABIR_GC_EVAC=always`     | 用于测试。每次完全回收都进行疏散（[7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)） |
| `NABIR_GC_YOUNG=<bytes>`   | 年轻代预算：两次回收之间分配出去的字节数，精确值（默认：每个进行分配的线程 8 MiB，范围 32 MiB 到 128 MiB；如果老年代字节更多，则与老年代字节数相同，最多 128 MiB）（[7.2.8](../007-runtime/702-allocation.md#728-gc-触发)） |
| `NABIR_GC_WORKERS=<n>`     | 在全局停顿（stop-the-world）期间与回收器一起标记的辅助线程数（默认：除第一个之外每个在线 CPU 一个，最多 7 个；`0`：不使用）（[7.3.3](../007-runtime/703-gc-algorithm.md#733-标记过程)） |
| `NABIR_GC_STRESS=N`        | 用于测试。每 N 次分配回收一次；自上次完全回收以来分配量达到老年代的十六分之一后，其中每第 8 次为完全回收 |
| `NABIR_HEAP_DUMP=on_oom`   | OOM 时在标准错误上打印总量、按字节数排列的前 32 种类型以及最大的 32 个大对象。尽力而为：它不分配内存，也不获取锁。格式由运行时定义 |
| `NABIR_WORKERS=<n>`        | 执行器工作线程数 N（默认：在线 CPU 数）（[7.6.8](../007-runtime/706-scheduler.md#768-每核一线程选项)） |
| `NABIR_THREAD_PER_CORE=1`  | 每核一线程：任务固定在其工作线程上，不进行窃取（7.6.8） |
| `NABIR_DYNAMIC_WORKERS=1`  | 空闲的工作线程驻留，在所有活跃工作线程都繁忙时被重新启用（[7.7.8](../007-runtime/707-io-and-timer.md#778-动态调整工作线程数)） |

未设置或为空的变量取其默认值。取值不在所列范围内时，在标准错误上打印一行警告，指明该变量、该值以及可接受的取值，该变量取其默认值。

## 9.9.9. REPL / Jupyter 调试

| 辅助手段             | 作用                                                                 |
|----------------------|----------------------------------------------------------------------|
| `<expr>?`            | 对表达式求值并打印其 `Debug` 形式（[9.7.7](907-jit-cells.md#977-可视化--输出)） |
| `it`                 | 最后一个表达式单元显示的值；用户自己定义的 `it` 优先（[9.7.7](907-jit-cells.md#977-可视化--输出)） |
| `:type <expr>`、`:t` | 打印表达式的静态类型（不运行它）                                      |
| `:doc <name>`        | 打印该名称的签名和文档注释，即 Jupyter `inspect_request` 的文本（[9.8.6](908-jupyter-integration.md#986-inspect_request帮助)）；方法写作 `:doc xs.len` |
| `:time <expr>`       | 把表达式作为一个单元求值，并在标准错误上打印每个阶段的耗时（与 `NABI_REPL_TIME=1` 相同）以及总耗时 |
| `:vars`              | 以 `name: type` 形式列出绑定，然后列出各项的签名                     |
| `:load <file>`       | 把程序加载到正在运行的会话中（[9.7.10](907-jit-cells.md#9710-加载程序repl-path)）；其中的项替换同名的项。由 `:save` 写出的脚本会逐个重新运行其单元 |
| `:reload`            | 重新加载上次加载的程序                                               |
| `:edit`              | 用 `$VISUAL` 或 `$EDITOR`（否则用 `vi`；Windows 上用 `notepad`）打开一个临时文件，然后把其中的内容作为单元运行；某个单元失败后，下一次 `:edit` 会再次打开该文本 |
| `:save <file>`       | 把会话写成脚本：先是各项，然后是运行过的单元，保持输入时的原样；再次声明的名称在脚本中也是重新声明（[5.1.4](../005-modules/501-variable-declaration.md#514-在同一作用域中重新声明)）。只在后面的单元读取 `it` 的地方声明 `it`。第一行是 `// nabi repl session (:save); :load replays it cell by cell`，每个单元之前有一行 `// :cell`：用 `:load` 加载该脚本会逐个重新运行其单元，因此再次声明的名称保留其最后的绑定；`nabi run` 把它作为程序运行 |
| `:opt [<level>]`     | 打印优化级别，或为之后的单元设置优化级别（`0`、`1`、`2`、`3`、`s`，与 `-O` 相同） |
| `:reset`             | 把会话重置为初始状态（库路径的 `use` 会重新插入）                    |
| `:add <dep> ..`      | 添加依赖（在当前环境中执行 `nabi install`，或在包的会话中执行 `nabi add`），并把它们挂接到会话上（[9.10.5](910-nabi-frontend.md#9105-环境)） |
| `:help`、`:h`        | 列出所有命令和别名                                                   |
| `:quit`、`:q`、`:exit` | 退出（输入结束也会退出）                                           |
| `exit`、`quit`、`exit()`、`quit()` | 单独位于一个单元中时退出；除非会话定义了该名称（绑定、项或 `use`），此时以该定义为准 |
| `NABI_REPL_TIME=1`    | 对每个单元显示各阶段的耗时，以及新编译的函数数 / 从之前单元复用的函数数（[9.6.4](906-jit-engine.md#964-库代码的复用)） |

要查看某个单元的 nabi-mir，把同样的代码放进 `main.nabi`，然后使用 `nabic build --emit=mir`（9.9.1）。

## 9.9.10. 元块调试

可以检查[用户定义的元块](../006-meta-system/608-meta-blocks.md)的派生 / 属性生成了什么代码。

```
nabic build --emit=expansions    # → target/<profile>/build/<name>.expanded.nabi
```

该选项把用户包的[元展开](../006-meta-system/609-template-tokens.md)结果作为源文本打印。每次展开所生成的源代码都跟在一条如下所示的注释之后；当生成的代码再次被展开时，其结果跟在外层展开之后。

```
// expanded from this #[derive(MyHash)] (src/geom.nabi:12:1)
impl MyHash for Point { ... }
```

要以 Nabi 源代码形式查看所有元展开和脱糖之后的完整 AST，使用 `--emit=source`（`<name>.ast.nabi`）。

## 9.9.11. 基准测试

微基准测试用以下命令运行。

```
nabi bench
```

`nabic bench` 构建基准测试框架，`nabi bench`（[9.10](910-nabi-frontend.md)）运行它（[9.1.2](901-nabic-cli.md#nabic-bench基准测试)），在 `-O2` 构建中逐个测量标有 `#[bench]` 的函数。每个函数接收一个 `Bench` 句柄，并通过 `b.iter` 为一个闭包计时，因此闭包之外的准备工作不计入测量。测量过程如下：

1. **时钟**。使用单调的纳秒时钟。通过每个样本对 `iters` 次调用计时（而从不对单次调用计时）来分摊其固定开销。
2. **校准**。除非给出 `--iters`，否则不断增大 `iters`，直到一个样本耗时约 100 ms，使时钟量化误差和开销可以忽略。
3. **预热**。运行大约一个样本时长的调用并丢弃结果，使缓存、分支预测器和分页稳定下来。
4. **采样**。采集 `--samples` 个样本（默认 10 个）；每个样本记录 `iters` 次调用的总时间，每次调用的时间为该总时间除以 `iters`。
5. **统计**。计算样本的中位数、四分位数和中位数绝对偏差（MAD）。位于 Tukey 栅栏之外（超出四分位数 1.5 个 IQR）的样本计为离群值。报告的数值是所有样本的**中位数（+/- MAD）**，偶尔出现的慢样本几乎不会使它变动。
6. **感知 GC**。在每个样本前后读取运行时的回收计数器，统计运行期间发生过回收的样本数。只要其中任一计数不为零，人类可读的行就以 `[<n> outliers, <m> gc]` 结尾；`--format=nabibench` 总是给出两者。

`black_box(x)` 是优化器无法看穿的恒等函数；把它应用于被计时闭包的输入和结果，使工作既不会被提升到外面，也不会被折叠掉。`b.set_bytes(n)` / `b.set_items(n)` 把每次调用的时间换算为吞吐量（`MiB/s`、`Melem/s`）。

无论是否输出到终端，输出都经过对齐并使用人类可读的单位；`--format=nabibench` 为回归工具提供稳定的、每个基准测试一行的机器可读形式（[9.1.2](901-nabic-cli.md#nabic-bench基准测试)），`--format=json` 提供 JSON 记录（[9.9.12](#9912-ci--测试集成)）。从未调用 `b.iter` 的基准测试被报告为什么也没有测量，这是警告而不是失败。`--iters` / `--samples` 可以调整采样。

## 9.9.12. CI / 测试集成

### CI 模式

在 CI 环境中，建议使用机器可读的输出。

```
nabic build --message-format=json --no-color > build.log
```

这种输出适合 CI 系统解析。

### 测试与基准测试记录

`nabi test --format json` 和 `nabi bench --format json` 在标准输出上每行写一个 JSON 对象。人类可读的文本、测试的输出以及基准测试自己打印的任何内容都写到标准错误，因此标准输出只包含记录。每条记录都有 `schema`（目前为 `1`）、`type` 和 `kind`（`test` 或 `bench`）。

```json
{"schema":1,"type":"case","kind":"test","name":"math::adds","result":"ok","ms":3.215}
{"schema":1,"type":"case","kind":"bench","name":"sum","result":"ok","ms":270.1,"median_ns":6.833,"mad_ns":0.167,"samples":10,"outliers":0,"gc":0,"bytes":0,"items":0}
{"schema":1,"type":"case","kind":"test","name":"net::downloads","result":"ignored","ms":0.002,"reason":"needs the network"}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"ignored":1,"filtered_out":0,"ms":120.4}
```

| `type`    | 字段                                                                |
|-----------|---------------------------------------------------------------------|
| `case`    | `name`；`result`：`ok`、`failed`、`ignored`（带 `#[ignore]` 的测试或文档块、什么也没有测量的基准测试）或 `timeout`（因超出时间限制而被终止的测试）；`ms`，其进程的墙钟时间；`reason`，有原因时给出（失败原因或忽略原因）。`result` 为 `ok` 的基准测试还有 `median_ns`、`mad_ns`、`samples`、`outliers`、`gc`、`bytes` 和 `items`，与 `#nabibench` 行相同（[9.1.2](901-nabic-cli.md#nabic-bench基准测试)） |
| `summary` | `result`（`ok` 或 `failed`）、`ms` 以及计数：测试为 `passed`、`failed`、`ignored`、`filtered_out`；基准测试为 `run`、`failed`、`ignored`、`filtered_out` |
| `listed`  | `--list` 保留的每个用例的 `name`，被忽略的测试还有 `ignored`（`true`）；其后没有摘要（[9.10.1](910-nabi-frontend.md#9101-子命令与全局选项)） |

每个测试框架的用例之后都有一条摘要，因此同时具有两个根文件的包，在其库的运行通过时会有两条摘要。记录遵循 JSON 诊断的 `schema` 规则（[9.9.4](#994-诊断--错误消息)），使用自己的编号，目前为 `1`：读取方忽略它不认识的字段。使用 `--message-format=json` 时，构建的诊断是同一标准输出上的 JSON 行；它们的 `type`（`error`、`warning`、..）可以把它们与记录区分开。

### 退出码

[`nabic` CLI 接口](901-nabic-cli.md)的退出码定义如下。

| 代码  | 含义             |
|-------|------------------|
| `0`   | 成功             |
| `1`   | 用户代码错误     |
| `2`   | CLI 错误         |
| `70`  | nabic 内部 panic |

`nabic test` 构建测试框架，但不运行测试。`nabi test` 运行它们，只要有一个失败就以 `1` 退出。每个测试在单独的进程中运行，因此某个测试的 panic（`101`）不会变成运行器的 `101`。

CI 可以对每个代码采取不同的处理（重试、通知等）。

## 9.9.13. LSP 服务器

`nabi lsp` 是 LSP（Language Server Protocol）服务器（[9.10.8](910-nabi-frontend.md#nabi-lsplsp-服务器)）。同一个服务器也以独立的二进制文件 `nabils` 提供。它通过标准输入输出通信，提供诊断、悬停提示（包括成员）、跳转到定义（返回定义名称的范围）、引用、重命名、文档符号、成员 / 路径补全、签名帮助、快速修复、格式化和语义标记（宏调用、用户定义的关键字、字面量后缀、运算符和属性，这些是静态语法无法识别的）。引用和重命名限于当前缓冲区。

```
nabi lsp
nabils
```

在这种模式下，VS Code、Vim、Emacs 等 LSP 客户端可以获得补全、悬停提示、跳转到定义、诊断等功能。它在内部使用链接进 `nabi` 的编译器增量前端，即 `nabic check` 运行的那个前端。

- **补全**。在 `x.` 之后，补全 `x` 的类型的字段和方法。对于来自标准库或其他包的类型，只提供 `pub` 的成员，不包括 `#[doc(hidden)]` 成员，也不包括运算符 trait 的方法（写作 `a + b`、`xs[i]` 等的那些）。在 `a::` 之后（也包括 `use a::b::` 中），补全 `a` 之下的模块和项。在其他位置，补全光标处作用域中的局部变量、缓冲区中的项、无需路径即在作用域中的名称（核心库的名称以及 `use` 项导入的名称）和关键字。REPL 和 Jupyter 内核使用同一引擎进行补全（[9.7.8](907-jit-cells.md#978-输入单位repl-识别)）。
- **悬停提示与定义**。悬停在函数、方法或字段（包括标准库中的）上时，先显示其签名，再显示其文档注释。在 `x.name` 上跳转到定义时，在 `x` 的类型中查找 `name`；在标准库的项上，跳转到库源代码中的该项。
- **签名帮助**。在调用的括号内（在 `(` 和 `,` 处请求），显示被调用方的签名和文档注释，光标所在的参数处于激活状态。方法在其接收者的类型中查找，其 `self` 不计入。
- **快速修复**。带有编辑的诊断（例如 ``convert with `as i32` ``）把该编辑作为 `quickfix` 代码操作提供。没有编辑时，说明应当写什么的 help 行会变成快速修复：``add `;` `` 在诊断范围的末尾插入 `;`，``add `use pkg::a::b;` `` 在文件最后一个顶层 `use` 之后插入该行（没有 `use` 时，插在开头的 `//!` 行和内部属性之后），``nabi add <name>`` 变成命令 `nabi.add`。收到 `workspace/executeCommand` 时，服务器在包目录中以 `nabi add <name>` 运行 `nabi.add`（对于脚本，在文件所在目录中运行 `nabi add <name> <file>`），使用 `$NABI` 指定的 `nabi`，否则使用服务器旁边的那个；然后重新解析依赖并重新检查打开的缓冲区，因此无需重启即可找到新包。它不运行其他命令，也不接受任何标志。
- **重命名**。当重命名会作用于光标下的名称时，`prepareRename` 返回该名称的范围。对于在缓冲区之外声明的名称，它会失败，并给出说明声明位置的消息，例如 ``cannot rename `println`: it is defined in the standard library``；对于关键字，消息为 `` `let` is a keyword ``。
- **文档符号**。符号的范围覆盖整个项，从其修饰符（`pub`、`const`、..）到其结尾的 `}` 或 `;`；其选择范围是名称。
- **边输入边诊断**。每次缓冲区变化（`didOpen`、`didChange`）之后都会发布诊断，而不仅仅在保存时。检查在后台进行，一次一个，在 30 ms 内没有新的编辑时开始；在此期间到达的编辑会基于最新文本一并检查，而在运行期间其缓冲区发生变化的检查，其诊断不会发布（它们会指向旧的文本）。这些诊断就是编译器自己的诊断，文本和代码相同，因此词法问题（不可见字符或形似字符、孤立的组合标记、注释中的双向控制字符、格式错误的数字、错误的转义）以及没有处理器接受的字面量（`10Hz`、`95%`、`dat"..."`）会在输入时就显示出来，每个都带有其诊断所携带的快速修复。
- **依赖**。每个缓冲区都使用其所属包的依赖进行检查：其上方最近的、包含 `nabi.nabion`、`src/main.nabi` 或 `src/lib.nabi` 的目录，根据其清单和锁文件解析，并按在该处启动的 `nabi lsp` 会构建的方式构建。不属于任何包、但位于某个工作区文件夹（`workspaceFolders`，否则为 `rootUri`）内的缓冲区，使用环境的库（[9.10.5](910-nabi-frontend.md#9105-环境)）；不在这些文件夹内的，使用服务器启动目录的依赖。每个包只解析一次；保存清单或锁文件（`didSave`）、编辑器报告它们发生了变化（`workspace/didChangeWatchedFiles`，客户端允许时服务器为此注册 `**/nabi.nabion` 和 `**/nabi.lock`）以及工作区文件夹发生变化时，会重新解析并重新检查打开的缓冲区。依赖无法解析的包会在没有依赖的情况下检查，服务器会说明一次原因（`window/showMessage`）。
- **检查运行期间的应答**。悬停提示、跳转到定义、补全、签名帮助、文档符号和格式化不等待对编辑的检查：它们根据最近一次完成类型检查的结果作答，并通过此后的编辑映射位置。位于该次检查之后被编辑过的文本上的位置（包括光标下的名称）则等待对当前文本的检查，引用和重命名也一样，因为它们按缓冲区的当前状态进行编辑。在 `x.` 之后补全时，如果只有正在输入的这一行发生了变化，且 `x` 在其中的写法与之前相同，则从该次检查中取得 `x` 的类型；否则检查缓冲区直到 `.` 为止。客户端在应答之前取消的请求（`$/cancelRequest`）会得到错误 `RequestCancelled`（-32800）。
- **诊断范围**。范围为空的诊断报告在该位置的词法单元上，或者在行末时报告在其前一个词法单元上。
- **手动启动**。在终端上，服务器首先向标准错误打印 ``nabils speaks the language server protocol on stdin; start it from an editor (see `nabi help lsp`)``。如果在客户端发送 `initialize` 之前输入就结束了，它打印 ``nabils: input ended before the editor sent `initialize` `` 并以 1 退出。在 `initialize` 之后，按照 LSP 标准的规定，`shutdown` 之后退出码为 0，否则为 1。

详细的 LSP 规范不在本规范范围之内（由 LSP 标准加上工具链的实现决策构成），另行定义。

## 9.9.14. 包检查

可以一次查看预构建库包的信息。

```
nabi inspect path/to/libfoo.a        # or foo.nabihir; the sibling file is read too
```

该命令按如下方式显示来自 NABIHIR（[9.4](904-nabihir-bytecode.md)）和[归档文件](../000-meta/007-glossary.md#archive)的信息。

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

`nabi inspect --dump <file>` 打印 NABIHIR 的版本、生成者、元信息以及所有接口文件（9.9.2）。

由此可以一目了然地看到包对外暴露了什么。

## 9.9.15. 总结

上述工具包括 Nabi 所配合的标准工具（gdb、lldb、perf、Jupyter）以及 Nabi 自己的工具（`--emit`、`nabi demangle`、`nabi inspect`、LSP 服务器）。前端 `nabi` 及其工具在 [9.10](910-nabi-frontend.md) 中定义，nabi-mir 文件格式在 [9.11](911-mir-format.md) 中定义。
