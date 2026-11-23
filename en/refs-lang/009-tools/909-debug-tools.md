# 9.9. Debug / diagnostic tools

This section defines the debug / diagnostic tools provided by `nabic` and `nabi` and their integration with external tools.

## 9.9.1. Intermediate output: `--emit`

The per-step results of the [AOT build pipeline](902-aot-pipeline.md) can be dumped.

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

`mir` is the nabi-mir the code generator compiles: above `-O0`, after optimization ([9.2.10](902-aot-pipeline.md#9210-optimization-levels)). `obj` is the object it writes. A disassembler such as `objdump -d` shows the machine code of an `obj`.

`asm` is the object as assembly source, as `clang -S` writes it: assembling it for the target (`clang -c`, `llvm-mc -filetype=obj`) makes an object with the same code bytes, relocations of the same type at the same offsets to the same places, and the same symbols as `obj`, so it links and runs the same. It is read back from the object `nabic` writes, in the GNU / LLVM syntax for the target (Intel syntax under `.intel_syntax noprefix` on x86_64 and x86). Each section has its `.section` directive as the object format names it (ELF `.text.<symbol>`, `.rodata.<symbol>`; Mach-O `__TEXT,__text`; COFF `.text` COMDATs with their selection and leader, `.pdata` associated with its function) and its `.p2align`. A symbol has its binding and visibility (`.globl`, `.weak`, `.hidden`; `.type` and `.size` on ELF; `.private_extern` and `.weak_definition` on Mach-O; `.def` on COFF; `.localentry` on powerpc64). Each function is its instructions, as the code generator encodes them: branch targets are local labels (`.LBB0_3`; `LBB0_3` on Mach-O and x86 Windows), and each relocated field is the expression that makes the assembler emit that relocation (`call nabi_panic_at@PLT`, `[rip + x@GOTPCREL]`, `[rip + x@GOTTPOFF]`, `[rcx + x@SECREL32]`, `[ebx + s@GOTOFF]`, `adrp x0, s@PAGE` / `:lo12:s`, `x@TLVPPAGE`, `#:lower16:(s-(.+16))`, `.word x(GOT_PREL)+0x1c`, `%pcrel_hi(s)` with `%pcrel_lo(.Lpcrel_hi0)`, `s@toc@ha`, `x@got@tprel@l`, `brasl %r14, f@PLT`, `x@INDNTPOFF`, ..). An encoding an assembler would not pick for the instruction's text is forced where the syntax allows (`{disp32}` on a long branch, `{disp8}`, `{rex}`, `mov.s` for the other `mov` form); otherwise the instruction is its bytes, the text in a comment (x86's frame setup `sub rsp, imm32`, whose immediate fits 8 bits). Data (string literals, statics, GC type descriptors, scan tables, vtables) is `.asciz`, `.byte`, `.zero` and pointer-sized directives naming their targets (`.quad`, `.long s-.`, `.rva`, `.secrel32`, `.secidx`). Unwind tables are the directives an assembler builds them from, each after the instruction it describes: `.cfi_startproc`, `.cfi_def_cfa_offset`, `.cfi_def_cfa_register`, `.cfi_offset` and `.cfi_endproc` for an ELF `.eh_frame` (the assembler's CIE is `nabic`'s) and for Mach-O arm64's compact unwind, which the assembler derives from the frame record and register pairs they describe; `.seh_proc`, `.seh_pushreg`, `.seh_stackalloc`, `.seh_setframe`, `.seh_savereg`, `.seh_savexmm`, `.seh_endprologue` and `.seh_endproc` for x64 COFF's `.pdata` / `.xdata`; `.fnstart`, `.save`, `.setfp` (or `.cantunwind`) and `.fnend` for 32-bit ARM's `.ARM.exidx`. The assembler's tables decode the same (`llvm-dwarfdump --eh-frame`, `llvm-readobj --unwind`, `llvm-objdump --unwind-info`). arm64 Windows keeps its `.pdata` / `.xdata` as data with their relocations: an assembler packs unwind codes (packed `.pdata` entries, epilogs sharing the prolog's codes) where `nabic` writes them out. The debug sections of `-g` (DWARF, CodeView) are data too, and 32-bit ARM's build attributes are `.eabi_attribute` directives. RISC-V source starts with `.option arch`, `.option norvc` and `.option norelax`; an instruction in its 16-bit compressed form (above `-O0`) is its `c.` mnemonic (`c.addi sp, -0x10`, `c.beqz a0, .LBB0_3`) under `.option rvc`, which turns back to `.option norvc` before the next 32-bit one; its ELF flags still come from the assembler's options (`clang --target=riscv64-linux-gnu`, or `llvm-mc -mattr=+m,+a,+f,+d,+c -target-abi=lp64d`). What an assembler may write differently means the same: a relocation against a local symbol's section with the symbol's offset; the section, mapping (`$x`, `$d`) and Mach-O `ltmp` symbols it always adds; RISC-V local labels kept as symbols; the `SHT_X86_64_UNWIND` type of an x86_64 `.eh_frame`; a Mach-O function's call to itself and its compact unwind entry resolved within the section; 32-bit ARM's hard-float ELF flag, which no directive sets (the build attributes say it); the name of a COFF weak external's default symbol (`.weak.<symbol>.default` with a suffix the assembler adds); the layout of unwind tables: an x64 `UNWIND_INFO` per function where `nabic` shares one, 32-bit ARM's `pop {r4-r11, lr}` in its short form.

On wasm32 the source is LLVM's WebAssembly assembly, for `clang --target=wasm32-wasi -c` or `llvm-mc -triple=wasm32-wasi -filetype=obj`. The features its instructions need come from the assembler's options, as no directive sets them: `-msimd128` (and `-matomics` on `wasm32-wasip1-threads`) for clang, `-mattr=+bulk-memory,+mutable-globals,+nontrapping-fptoint,+reference-types,+sign-ext,+simd128` (and `+atomics`) for `llvm-mc`. The source declares the globals, the table and every function it uses (`.globaltype __stack_pointer, i32`, `.tabletype __indirect_function_table, funcref`, `.functype f (i32, i64) -> (f64)`), with `.import_module` and `.import_name` for a function imported under another name. Each function is in its own section (`.section .text.<symbol>,"",@`) with its binding (`.globl`, `.weak`, `.hidden`, `.export_name`), its `.functype` and its `.local` line; its instructions have each `end` named by what it closes (`end_block`, `end_loop`, `end_if`, `end_function`) and the untyped `select` by its operands' type (`i32.select`), and each relocated operand is its symbol: `call f`, `global.get __stack_pointer`, `i32.const s` (a data address, or a function's table index), `i32.const s@TLSREL`, `call_indirect __indirect_function_table, (i32) -> (i32)`. Each data segment is a section of its own (`.section .data.<symbol>,"",@`, `"T"` when thread-local) with its `.p2align`, its symbols (`.type s,@object`, `.size`) and `.int32 s` for each pointer; the debug sections and `target_features` are data, a code offset `.int32 f+0x2b`. The assembler's object has the same functions, code (relocated operands 5 bytes long, as here), relocations, data segments, symbols and custom sections. What it writes differently means the same: the order of the function and type indices, the initial size of the memory and table imports, the `NO_STRIP` flag it sets on `__indirect_function_table`, and its `DATACOUNT` section.

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

`listing` is the machine code as text, written by the code generator from the instructions it encodes, so its bytes are exactly those of `obj`. Each function starts with its symbol (`<name>:`); each instruction is a line with its offset from the function's start, its bytes and its text, as `llvm-objdump -dr` prints them for the target (Intel syntax on x86_64 and x86). The code generator's labels head the instructions they mark (`.L3:`), a branch target names its label (`jne 0x1c <.L3>`), and each relocation follows the instruction whose field it fills, with the object format's name for it (`R_X86_64_PLT32`, `ARM64_RELOC_BRANCH26`, `IMAGE_REL_AMD64_REL32`, `R_386_PLT32`, `R_ARM_CALL`, `R_RISCV_CALL_PLT`, `R_PPC64_REL24`, `R_390_PLT32DBL`, `R_WASM_FUNCTION_INDEX_LEB`, ..); on riscv a `R_RISCV_PCREL_LO12_I` names the object's label at its `auipc` (`.Lpcrel_hi0`), and a 16-bit instruction shows as 4 hex digits with the text of the 32-bit instruction it stands for. On wasm32 the offsets count from the function's symbol (its body size), a `.local` line lists its locals, calls and globals show the indices the object gives them, and the control-flow comments are objdump's (`# 1: down to label3`), its label numbers running on over the whole object.

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

The output is written to the following locations.

```
target/debug/build/foo.ast.nabi
target/debug/build/foo.hir
target/debug/build/foo.mir
target/debug/build/foo.o
target/debug/build/foo.s
target/debug/build/foo.lst
target/debug/build/foo.expanded.nabi
```

To select several steps at once, separate them with commas, as in `--emit=hir,mir,obj`. With a single kind, `-o <path>` changes the location. With `--emit`, no executable or archive is produced.

## 9.9.2. NABIHIR inspection

```
nabi inspect --dump path/to/foo.nabihir   # or libfoo.a; reads the sibling foo.nabihir
```

This command prints the [NABIHIR](904-nabihir-bytecode.md) of a prebuilt library in human-readable form. After the version, producer and meta, each interface file follows as `--- <path>` and its text (source with concrete function bodies cut down to `;`). The hand-off summaries follow under `--- summaries`, the call summaries under `--- calls`, one line per function with the facts that hold ([9.4.7](904-nabihir-bytecode.md#947-body)).

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

This output is useful for verifying the correctness and compatibility of [NABIHIR](../000-meta/007-glossary.md#nabihir-nabi-hir-bytecode).

## 9.9.3. Symbol demangle

```
nabi demangle [-v] <symbol>...
nabi demangle _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190
nabi demangle < symbols.txt
```

This command prints each mangled symbol of [Symbol mangling](905-symbol-mangling.md) as the name source writes, one per line. A standard library path starts with its tier (`core` / `std` / `bare`). A method (`<type>.<method>`, or `<type>__<method>` of a generic type's instance) is shown as `<type>::<method>`, one picked by trait (`__by_<trait>`) as `<path::type as trait>::<method>`, and a method of an impl block named by hash as `{impl}::<method>`. The typehash is left out. The Mach-O form, which has one more leading `_` (`__Y...`, as `nm` prints it), and a `.<suffix>` after the symbol are accepted.

```
$ nabi demangle __Ya61f26f1_core_collections_dict_Dict_0_0grow_ce3f36e15e552a70
core::collections::dict::Dict::grow
```

With `-v`, each symbol is a block of the parts it encodes, the blocks separated by blank lines:

```
package:  3f8a1c2b
path:     geom
name:     distance
generics: a7c91d8e44b2f190 (type tuple hash)
```

If any argument is not a Nabi symbol, the exit code is `1`. With no symbol, `nabi demangle` is a filter: it copies standard input to standard output with every Nabi symbol in it replaced by its one-line name and everything else unchanged, so profiler, `nm` and crash output can be piped through it:

```
perf report --stdio | nabi demangle
nm target/debug/my_binary | nabi demangle
```

## 9.9.4. Diagnostics / error messages

### Standard format

Diagnostic messages are printed in the following standard format.

```
error[E0027]: cannot find value `count` in this scope
  --> src/main.nabi:5:13
   |
 5 |     let x = count + 1;
   |             ^^^^^
   |
   = help: did you mean `counter`?
```

This format uses the file + line + column diagnostic units of [Source files](../001-lexical/101-source-files.md).

The header is `<severity>[<code>]: <message>` and `--> path:line:col` gives the primary location. The primary span is underlined with `^`, secondary spans with `-`; a secondary span in another file is introduced by `::: path:line:col`. Trailing `= note:` lines add context and `= help:` lines suggest a fix. For a name not found that an item elsewhere has, the help gives the `use` lines that bring it in (up to three: the package's own items first, then the standard library's, then the dependencies'), or, when the package's only such item is hidden, the visibility that reaches the use and its `use`; an item of the package itself is imported by its path from the package root (``add `use pkg::geo::Point;` ``; `` `geo::secret` has that name, but it is private to module `geo`: write `pub(pkg)` before it, so its whole package sees it, and add `use pkg::geo::secret;` ``).

The primary span is the code at fault: for a value of the wrong type, the value, not the statement that holds it. Messages name library items as they are written (`List::sort`, `Send`, `print`, not `core::marker::Send`). Messages do not cite sections of this specification; what to write instead is in the message, the help lines and `nabic --explain <code>`.

### JSON format (tool integration)

For tool integration, output can also be produced as JSON.

```
nabic build --message-format=json
```

Each diagnostic is one line of JSON object on stdout; no human-format diagnostic is printed, and stderr holds only progress lines.

```json
{"type":"error","code":"E0040","kind":"type","message":"...","spans":[{"file_name":"src/main.nabi","line_start":5,"col_start":10,"byte_start":71,"byte_end":76,"is_primary":true,"label":"...","line_end":5,"col_end":15}],"notes":["..."],"fixes":[],"help":["..."],"lint":null,"schema":1}
```

| Field      | Meaning                                                            |
|------------|--------------------------------------------------------------------|
| `type`     | `error` / `fatal` / `warning` / `note`                             |
| `code`     | The error code, `E0040` ([error codes](#error-codes))             |
| `kind`     | The area the code belongs to (`type`, `parse`, ..)                 |
| `message`  | Message                                                            |
| `spans`    | List of locations. The first element is the primary location (`is_primary: true`); the rest are secondary |
| `notes`    | List of note strings, without the help entries                     |
| `fixes`    | Edits that fix the code, each a span of one file with the `text` that replaces it and a `title` (below); often empty |
| `help`     | The help entries (the `= help:` lines of the human format), without the `help: ` prefix |
| `lint`     | The lint a warning or lint error belongs to (`unused_variables`), the name `#[allow(..)]` and `-A` take ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)); `null` for the others |
| `schema`   | The version of this format, now `1`                                |

A span has `file_name`, `line_start`, `col_start`, `byte_start`, `byte_end`, `is_primary`, `label`, `line_end` and `col_end`. `line_end` / `col_end` are where the span ends (the position just past its last character), counted like `line_start` / `col_start` in code points; they are `null` when the source text is not at hand. A diagnostic that knows only the file and no position has just `{"file_name": .., "is_primary": true}`. A fix has `file_name`, `line_start`, `col_start`, `byte_start`, `byte_end`, `text`, `title` (``convert with `as i32` ``), `line_end` and `col_end`; the language server offers it as a quick fix ([9.9.13](#9913-lsp-server)).

Within one `schema`, fields are only added, never removed, renamed or given another meaning, so a reader should ignore fields it does not know. The test, bench and `nabi deps` records ([9.9.12](#9912-ci--test-integration), [9.10.4](910-nabi-frontend.md#9104-dependencies-nabi-add--remove--update--deps)) follow the same rule with their own `schema`, now `1`.

This format lets IDEs and CI systems read diagnostics mechanically.

### Error codes

Every kind of error has its own code, `E` and four digits: `E0040` for mismatched types, `E0027` for a name that is not found. Codes run from `E0001` with no gaps, numbered in the order they were added, not by area; the `kind` field gives the area. A diagnostic of a lint ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)) has the lint code `E0131` whatever part of the compiler raised it, and names the lint in its note and in `lint`.

`nabic --explain <code>` (also `nabi --explain`) prints the code's page: its title, what it means and what to write instead ([9.1.5](901-nabic-cli.md#915-global-options)). `E0040`, `e0040` and `40` name the same code; a code that does not exist is a usage error (exit code 2). The error pages of the documentation are the same pages, one per code.

The titles are written exactly as `nabic --explain` prints them.

| Code    | `kind`        | Title                        |
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

The `kind` values are `lexical`, `parse`, `resolve`, `visibility`, `module`, `type`, `pattern`, `handoff` (ownership and lifetime of values: use after a hand-off, [stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers); a resource used after it was moved or closed, or not closed, [stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop---cleanup-on-release); a thread-local value kept past `with`; a stream view used past its stream, [stdlib 2.23.1](../../refs-standard-libraries/200-core/223-io.md#2231-read); a `SliceMut` used after its list was resized, [stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt---writable-view)), `meta`, `linkage`, `unsupported`, `internal`, `other` and `lint`.

## 9.9.5. Logging / progress display

### Standard output

Default build output looks like this.

```
   Compiling foo (path/to/foo)
    Building target/debug/libfoo.a
    Finished in 1.23s
```

Progress lines go to stderr; the list of verbs is in [9.1.8](901-nabic-cli.md#918-output-format).

### `--verbose` / `-v`

Verbose mode also shows per-stage timing.

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

Showing the time of each stage is useful for diagnosing build bottlenecks. The stages are `front end` and `codegen + link` (`emit <ext>` with `--emit`).

### `NABI_TIME_PASSES`

`NABI_TIME_PASSES=1` prints the wall time of each compiler pass on stderr.

### `-vv` (very verbose)

In addition to `-v`, prints a `Reading <file>` line for each file processed and a `Specialized <path>(sig)` line for each (item, type) pair [specialized](../000-meta/007-glossary.md#specialize).

## 9.9.6. Debugger integration: gdb / lldb

### DWARF debug information

A `debug` profile build, and a `release` build with `-g` ([9.2.15](902-aot-pipeline.md#9215-output)), emits DWARF debug information with the following properties. Windows targets get the same information as CodeView instead ([below](#codeview-and-pdb-windows)).

- A function's `DW_AT_name` is its Nabi path (`geom::distance`); no linkage name (mangled symbol) is emitted, because debuggers do not understand `_Y` mangling. The wrapper that calls the program's `main` ([9.5.9](905-symbol-mangling.md#959-symbol-of-the-main-function)) is named `nabi_main`, so a backtrace has one `main` frame, the program's.
- Line numbers refer to lines of the original .nabi file.
- Local variables appear with their original names and types. Besides primitive types (integers, floating-point, `bool`, `char`), DWARF types are emitted for structs (field names; an instance of a generic struct is named with its type arguments as source writes them, `core::collections::list::List<i64>`), tuples, `str`, `any`, sum types and arrays. A sum type or enum is a DWARF variant part: an artificial discriminant member, and one `DW_TAG_variant` per variant (dataless variants included) with `DW_AT_discr_value`, each holding a struct named `<Sum>::<Variant>`. Compile units use `DW_LANG_Rust` so debuggers read variant parts (there is no DWARF language code for Nabi). A `GcBuf<T>` reference is described as a pointer to its object (header, then the `T` elements). An `any` value is a variant part over the types stored into it, with the stored tag as the discriminant. The discriminant is an artificial `i32` (the stored tag; a `throws` carrier's `bool` tag is a byte). A function value (a `fn` type or a closure) is a structure named for its type (`fn(i32) -> i32`) with members `code`, a pointer to a `DW_TAG_subroutine_type` (the environment, then the parameters), and `env`, its environment (a pointer to the environment object when the closure's type is known), so a debugger shows the target function. A variable captured by reference shows as its value. Functions the compiler makes (the C `main`, static initializer guards) have subprograms too. Compiler-generated locals (names beginning `#` or `__`) are omitted.
- A lexical block scope is emitted for each block, so shadowed names are visible only inside their block.
- At `-O0` every local keeps its own stack slot, and its location is that slot.
- Above `-O0`, `-g` does not change the code ([9.2.10](902-aot-pipeline.md#9210-optimization-levels)); the debug information describes the optimized code. A variable's location is a location list (`.debug_loc`) by code range: the register holding its value, its stack slot, a constant, or pieces of these (`DW_OP_piece`) for a struct or closure built in place. Where its value no longer exists it has no location, and debuggers show it as optimized out. A variable holding a GC reference across a safepoint is always in its stack slot, the root a collection reads and updates.
- Above `-O0`, a body inlined is a `DW_TAG_inlined_subroutine` (with `DW_AT_call_file`, `DW_AT_call_line`, `DW_AT_call_column`) of an abstract `DW_TAG_subprogram` with `DW_AT_inline`, holding the callee's parameters and locals, so a debugger stops in it and shows it as a frame of its own. Line rows of inlined code name the callee's file, a row is a statement (`is_stmt`) only where the line changes, and the first row after the prologue has `prologue_end`. On Apple targets the unit has `DW_AT_APPLE_optimized`. wasm32 describes the inlined bodies and the variables the same way. Above `-O0` a variable kept in wasm locals has a location list naming the local over the code where it holds the value (`DW_OP_WASM_location 0x0 <local>, DW_OP_stack_value`; a struct, a piece per local), and one in linear memory its frame slot while that holds its value (`DW_OP_fbreg`; the frame base is itself a wasm local). At `-O0` every variable is in its frame slot.

### CodeView and PDB (Windows)

On Windows targets the same information is CodeView (C13) in each object, and the linker gathers it into a PDB next to the executable.

- `.debug$S` holds, in a section associated with each function's code, the function's `S_GPROC32` (its Nabi path as the name) and `S_FRAMEPROC`, each parameter and local as `S_LOCAL` with a frame-pointer relative location (`S_DEFRANGE_FRAMEPOINTER_REL_FULL_SCOPE`), lexical blocks of one code range as `S_BLOCK32` (the locals of any other block belong to the enclosing scope), and the function's line table. The object's source files are in its checksum table.
- Above `-O0` a variable's location is given by code range: `S_DEFRANGE_REGISTER` or `S_DEFRANGE_FRAMEPOINTER_REL`, and for a struct whose fields are apart (in registers, in memory, or both) one record per field: `S_DEFRANGE_SUBFIELD_REGISTER` for a field in a register, `S_DEFRANGE_REGISTER_REL` off the frame pointer with the field's offset for a field in memory. Registers are named by their CodeView numbers: the general and XMM registers on x64 and x86, `X0`-`X30` and, by size, `S`, `D` or `Q` on arm64. A variable whose only value is one integer, `bool` or `char` constant is an `S_CONSTANT`; one whose only value is a float constant is an `S_LDATA32` in its scope, of the value in a read-only data section that goes with the function's code (dbghelp reads no float `S_CONSTANT`); an `S_LOCAL` whose value is nowhere is marked optimized out. A struct, tuple or array parameter the Windows ABI passes by reference (x64: not of 1, 2, 4 or 8 bytes; arm64: over 16 bytes) is an `S_LOCAL` without the parameter flag, at every level: dbghelp reads such a parameter as a pointer to its value, where Nabi describes the value itself. A body inlined is an `S_INLINESITE` of the callee's `LF_FUNC_ID`, with binary annotations mapping its code to its lines; the object's one `DEBUG_S_INLINEELINES` subsection (in the main `.debug$S`, and one per module in the PDB: dbghelp reads only a module's first) gives each callee's file and line, the function lists its callees in `S_INLINEES`, and its own line table gives inlined code the line of the call. Inline sites are never inside an `S_BLOCK32`.
- `.debug$T` holds the types: primitive types as CodeView basic types; structs, tuples, `str`, `any` and function values as `LF_STRUCTURE` with field names; a sum type or enum as a structure of the tag and one member per variant at offset 0, each a structure named `<Sum>::<Variant>` over the whole value; a `GcBuf<T>` or boxed reference as a pointer to its object; a function's signature as `LF_PROCEDURE`.
- The PDB (MSF 7.0) has the type records of every object merged (one copy of each), a module per object with its symbols and line tables, the global symbols (procedure references, global data) and public symbols, section contributions and the section map. Objects with CodeView of their own (the runtime's) are included the same way. The executable's debug directory names the PDB by path, GUID and age.

dbghelp-based debuggers (WinDbg and cdb, the Visual Studio debugger) find functions by name, map addresses to file and line, and show parameters and locals with their values; in optimized code, also the inlined frames (`k`; `bm <module>!<fn>` sets a breakpoint at each place a function was inlined) and the variables in registers, split across registers and memory, or constant (`dv`, `dx`).

### Using gdb / lldb

Standard gdb and lldb commands work on Nabi code as is.

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

### Formatters

The toolchain ships formatters for lldb, gdb and natvis (Visual Studio, WinDbg, cdb) that show the standard library's values as values:

- `str` as its text (`"hello"`);
- `List`, `Slice`, `Set` and `SortedSet` as `size=N` with the elements as children, `Dict` and `SortedDict` as `size=N` with each value as a child named by its key, all in iteration order (insertion order, key order for the sorted ones);
- `Array` as `shape=[..]` with the elements in row-major order, named by index (`[1, 0]`).

gdb shows only the active variant of a sum, an enum or an `any` from the DWARF alone. lldb has no plugin for the unit's language and lists every variant; its formatters show the active variant only (its name as the summary, its fields as the children; a boxed `any` payload through its box). A value read before it is assigned (a stale slot) shows as `<invalid>` where its length cannot be its own, and children are shown up to 10000.

| Debugger | File | Loading |
|---|---|---|
| lldb | `lib/nabi/lldb/nabi.py` | `nabic --print lldbinit` prints the `command script import` line |
| gdb | `lib/nabi/gdb/nabi.py` | `nabic --print gdbinit` prints the `source` line |
| Visual Studio, WinDbg, cdb | `lib/nabi/natvis/nabi.natvis` | `nabic --print natvis` prints the path |

Both Python formatters read the collections' layouts from `lib/nabi/debug/layout.py` and map the standard library's `/nabi/library` source paths to the toolchain's library sources. The natvis file lists a `SortedDict` / `SortedSet` while its tree is at most two levels deep (up to 1023 entries) and shows only the size of a bigger one; it has no view of sums (the debugger shows the tag and every variant).

```
lldb -O "$(nabic --print lldbinit)" ./target/debug/my_binary
nabic --print lldbinit >> ~/.lldbinit
gdb -iex "$(nabic --print gdbinit)" ./target/debug/my_binary
nabic --print gdbinit >> ~/.gdbinit
cdb -c ".nvload $(nabic --print natvis)" my_binary.exe
```

In Visual Studio, copy the file into `%USERPROFILE%\Documents\Visual Studio 2022\Visualizers`.

### macOS / Windows debuggers

Each platform's native debugger handles Nabi binaries.

- On macOS, use lldb or the Xcode debugger.
- On Windows, use the Visual Studio debugger or WinDbg (they read the PDB: functions, source lines, parameters and locals).

### Panic backtraces

`NABI_BACKTRACE=1` or `full` makes a panic print the call stack ([4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors)). It needs no `-g`: the runtime walks the frame pointers every function nabic emits keeps, from the call into the panic entry points of the runtime (`nabi_panic`, `nabi_panic_at`, `nabi_panic_str`, `nabi_try_failed`), and names each return address from a function table that Nabi's linker writes into every executable (`.nabi_symbols`, `__TEXT,__nabi_symbols`, or part of `.rdata` on Windows): each live Nabi function's address range and Nabi path, sorted by address. The runtime's word `nabi_backtrace_symbols` holds the table's offset from itself, so the table needs no relocation.

- A function inlined into its caller has no frame of its own, so an optimized build shows fewer frames. Folded copies ([9.2](902-aot-pipeline.md)) show the name of the copy that stays.
- The walk stops at the first frame that is not Nabi code: C code and the runtime keep no frame pointers it can follow.
- No backtrace, only a `note:` line: a program linked by another linker (`NABI_CC`) has no table, and a panic raised inside the runtime (not at a call from compiled code) has no frame to start from.
- Targets: every native target except powerpc64le. wasm32 code cannot read its own call stack, and a target without an OS has no environment; neither prints one.

## 9.9.7. Profiler integration

### perf (Linux)

CPU profiling can be done with `perf`.

```
perf record ./target/release/my_binary
perf report --stdio | nabi demangle
```

`perf report` names functions from the symbol table, where Nabi functions are `_Y` symbols ([9.5](905-symbol-mangling.md)); `nabi demangle` with no argument ([9.9.3](#993-symbol-demangle)) replaces each with its Nabi name. A profiler that names functions from the debug information of a `-g` build shows the Nabi path (`geom::distance`) itself.

### Tracy / Chrome trace (optional)

Through the standard library's tracing surface, traces compatible with `tracy` or Chrome's `chrome://tracing` can be produced. The choice of a specific library is outside the scope of this specification and is covered by a separate library spec.

## 9.9.8. Memory / GC tools

The runtime reads the following environment variables.

| Variable                  | Effect                                                                       |
|---------------------------|------------------------------------------------------------------------------|
| `NABIR_GC_LOG=1`           | Prints one line per collection to stderr: the kind (minor / full), pause time, mark count and sweep result (live bytes before -> after), then the parts of the pause in microseconds (time to safepoint, mark, sweep, evacuation), the number of threads and the time since the collector started. A concurrent full collection logs its start pause (a minor collection; its mark count is the minor's) and its finish pause separately. Time to safepoint is the wait from the stop request until every other thread stood still: a long value means a thread held the stop back, not that the collection was slow |
| `NABIR_GC_STATS=1`         | At exit, prints collection counts (minor / full), pause time and collector CPU time to stderr. The second line gives block count / resident size / evacuation count / bytes moved, the third full pauses, concurrent marks (and those allocation finished in a pause), the marker's CPU time and the mark assists ([7.5.4](../007-runtime/705-concurrent-collection.md#754-allocation-and-collection)). Then one line per kind of pause that ran (minor, full, concurrent start, concurrent finish): count, p50 / p90 / p99 (within an eighth, from a histogram), longest and total pause, longest and total time to safepoint |
| `NABIR_GC_CONCURRENT=auto\|always\|off` | Concurrent marking for full collections. `auto` applies when old objects are 32 MiB or more, or a pause goal is set ([7.5](../007-runtime/705-concurrent-collection.md)) |
| `NABIR_GC_EVAC=always`     | For testing. Evacuation on every full collection ([7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)) |
| `NABIR_GC_YOUNG=<bytes>`   | Young budget: bytes handed out between collections, exactly (default: 8 MiB per allocating thread, 32 MiB to 128 MiB; as many as the old bytes up to 128 MiB if more) ([7.2.8](../007-runtime/702-allocation.md#728-gc-trigger)) |
| `NABIR_GC_WORKERS=<n>`     | Helper threads that mark beside the collector in stop-the-world pauses (default: one per online CPU beyond the first, at most 7; `0`: none) ([7.3.3](../007-runtime/703-gc-algorithm.md#733-mark-procedure)) |
| `NABIR_GC_STRESS=N`        | For testing. Collects every N allocations; every 8th of those is full once a sixteenth of the old generation was allocated since the last full one |
| `NABIR_HEAP_DUMP=on_oom`   | On OOM, prints to stderr the totals, the top 32 types by bytes and the 32 largest large objects. Best effort: it allocates nothing and takes no lock. Format is runtime-defined |
| `NABIR_WORKERS=<n>`        | Number of executor workers N (default: the online CPUs) ([7.6.8](../007-runtime/706-scheduler.md#768-thread-per-core-option)) |
| `NABIR_THREAD_PER_CORE=1`  | Thread-per-core: tasks pinned to their worker, no stealing (7.6.8) |
| `NABIR_DYNAMIC_WORKERS=1`  | Idle workers park and are taken back when every active worker is busy ([7.7.8](../007-runtime/707-io-and-timer.md#778-dynamic-adjustment-of-worker-count)) |

An unset or empty variable has its default. A value outside those listed prints one warning line on stderr naming the variable, the value and the accepted values, and the variable has its default.

## 9.9.9. REPL / Jupyter debugging

| Aid                  | Effect                                                               |
|----------------------|----------------------------------------------------------------------|
| `<expr>?`            | Evaluates the expression and prints its `Debug` form ([9.7.7](907-jit-cells.md#977-visualization--output)) |
| `it`                 | The last value an expression cell showed; a user's `it` wins ([9.7.7](907-jit-cells.md#977-visualization--output)) |
| `:type <expr>`, `:t` | Prints the static type of the expression (without running it)        |
| `:doc <name>`        | Prints the name's signature and doc comment, the text of a Jupyter `inspect_request` ([9.8.6](908-jupyter-integration.md#986-inspect_request-help)); a method as `:doc xs.len` |
| `:time <expr>`       | Evaluates the expression as a cell and prints the time of each stage (as `NABI_REPL_TIME=1`) and the total on stderr |
| `:vars`              | Lists the bindings as `name: type`, then the items' signatures       |
| `:load <file>`       | Loads a program into the running session ([9.7.10](907-jit-cells.md#9710-loading-a-program-repl-path)); its items replace those of the same name. A script written by `:save` runs its cells again one by one |
| `:reload`            | Loads the last loaded program again                                  |
| `:edit`              | Opens `$VISUAL` or `$EDITOR` (else `vi`; `notepad` on Windows) on a scratch file, then runs what it holds as cells; after a failed cell the text opens again on the next `:edit` |
| `:save <file>`       | Writes the session as a script: the items, then the cells that ran, as typed; a name declared again is a redeclaration there too ([5.1.4](../005-modules/501-variable-declaration.md#514-redeclaration-in-the-same-scope)). `it` is declared only where a later cell reads it. The first line is `// nabi repl session (:save); :load replays it cell by cell` and a `// :cell` line comes before each cell: `:load` of the script runs the cells again one by one, so a name declared again keeps its last binding; `nabi run` runs it as a program |
| `:opt [<level>]`     | Prints, or sets for later cells, the optimization level (`0`, `1`, `2`, `3`, `s`, as `-O`) |
| `:reset`             | Resets the session to its initial state (the `use` of a library path is inserted again) |
| `:add <dep> ..`      | Adds dependencies (`nabi install` into the current environment, or `nabi add` in a package's session) and attaches them to the session ([9.10.5](910-nabi-frontend.md#9105-environments)) |
| `:help`, `:h`        | Lists every command and alias                                        |
| `:quit`, `:q`, `:exit` | Quit (end of input also quits)                                     |
| `exit`, `quit`, `exit()`, `quit()` | Alone in a cell, quit; unless the session defines the name (a binding, an item or a `use`), which wins |
| `NABI_REPL_TIME=1`    | Per cell, per-stage timing and the number of newly compiled functions / functions reused from earlier cells ([9.6.4](906-jit-engine.md#964-reuse-of-library-code)) |

To see a cell's nabi-mir, put the same code in `main.nabi` and use `nabic build --emit=mir` (9.9.1).

## 9.9.10. Meta block debugging

You can check what code the derives / attributes of [user-defined meta blocks](../006-meta-system/608-meta-blocks.md) generate.

```
nabic build --emit=expansions    # → target/<profile>/build/<name>.expanded.nabi
```

This option prints the result of [meta expansion](../006-meta-system/609-template-tokens.md) of the user package as source text. Each expansion's generated source follows a comment like the one below; when generated code is expanded again, that result follows the outer expansion.

```
// expanded from this #[derive(MyHash)] (src/geom.nabi:12:1)
impl MyHash for Point { ... }
```

To see the full AST after all meta expansion and desugaring as Nabi source, use `--emit=source` (`<name>.ast.nabi`).

## 9.9.11. Benchmarks

Microbenchmarks are run with the following command.

```
nabi bench
```

`nabic bench` builds the benchmark harness and `nabi bench` ([9.10](910-nabi-frontend.md)) runs it ([9.1.2](901-nabic-cli.md#nabic-bench-benchmarks)), measuring the functions marked `#[bench]` one at a time in an `-O2` build. Each takes a `Bench` handle and times a closure through `b.iter`, so setup outside the closure is not measured. The measurement is:

1. **Clock.** A monotonic nanosecond clock. Its fixed overhead is amortized by timing `iters` calls per sample, never a single call.
2. **Calibration.** Unless `--iters` is given, `iters` is raised until one sample takes about 100 ms, so clock quantization and overhead are negligible.
3. **Warm-up.** Calls are run for about one sample's worth of time and discarded, to settle caches, branch predictors and paging.
4. **Sampling.** `--samples` samples (default 10) are taken; each records the total time of `iters` calls, and the per-call time is that total divided by `iters`.
5. **Statistics.** The median, the quartiles and the median absolute deviation (MAD) of the samples are computed. Samples outside the Tukey fences (1.5 IQR beyond the quartiles) are counted as outliers. The reported figure is the **median (+/- MAD)** of all samples, which an occasional slow sample barely moves.
6. **GC awareness.** The runtime's collection counter is read around each sample, and the samples during which a collection ran are counted. The human line ends with `[<n> outliers, <m> gc]` when either count is not zero; `--format=nabibench` always gives both.

`black_box(x)` is an identity the optimizer cannot see through; apply it to the inputs and result of the timed closure so the work is neither hoisted out nor folded away. `b.set_bytes(n)` / `b.set_items(n)` turn the per-call time into a throughput (`MiB/s`, `Melem/s`).

Output is aligned and in human units, to a terminal or not; `--format=nabibench` gives a stable one-line-per-bench machine form for regression tools ([9.1.2](901-nabic-cli.md#nabic-bench-benchmarks)), and `--format=json` gives JSON records ([9.9.12](#9912-ci--test-integration)). A benchmark that never calls `b.iter` is reported as having measured nothing, a warning and not a failure. `--iters` / `--samples` adjust the sampling.

## 9.9.12. CI / test integration

### CI mode

In CI environments, machine-readable output is recommended.

```
nabic build --message-format=json --no-color > build.log
```

This output is suitable for CI systems to parse.

### Test and bench records

`nabi test --format json` and `nabi bench --format json` write one JSON object per line on stdout. The human text, the output of the tests, and anything a benchmark prints itself go to stderr, so stdout holds only records. Each record has `schema` (now `1`), `type` and `kind` (`test` or `bench`).

```json
{"schema":1,"type":"case","kind":"test","name":"math::adds","result":"ok","ms":3.215}
{"schema":1,"type":"case","kind":"bench","name":"sum","result":"ok","ms":270.1,"median_ns":6.833,"mad_ns":0.167,"samples":10,"outliers":0,"gc":0,"bytes":0,"items":0}
{"schema":1,"type":"case","kind":"test","name":"net::downloads","result":"ignored","ms":0.002,"reason":"needs the network"}
{"schema":1,"type":"summary","kind":"test","result":"failed","passed":2,"failed":1,"ignored":1,"filtered_out":0,"ms":120.4}
```

| `type`    | Fields                                                              |
|-----------|---------------------------------------------------------------------|
| `case`    | `name`; `result`: `ok`, `failed`, `ignored` (an `#[ignore]` test or doc block, a benchmark that measured nothing) or `timeout` (a test killed at its time limit); `ms`, the wall time of its process; `reason`, when there is one (why it failed, or the ignore reason). A benchmark with `result` `ok` also has `median_ns`, `mad_ns`, `samples`, `outliers`, `gc`, `bytes` and `items`, as in the `#nabibench` line ([9.1.2](901-nabic-cli.md#nabic-bench-benchmarks)) |
| `summary` | `result` (`ok` or `failed`), `ms`, and the counts: for tests `passed`, `failed`, `ignored`, `filtered_out`; for benchmarks `run`, `failed`, `ignored`, `filtered_out` |
| `listed`  | `name`, of each case `--list` keeps, and `ignored` (`true`) for an ignored test; no summary follows ([9.10.1](910-nabi-frontend.md#9101-subcommands-and-global-options)) |

A summary follows the cases of each harness, so a package with both root files has two when its library's run passes. Records follow the `schema` rule of the JSON diagnostics ([9.9.4](#994-diagnostics--error-messages)) with their own number, now `1`: a reader ignores fields it does not know. With `--message-format=json`, the build's diagnostics are JSON lines on the same stdout; their `type` (`error`, `warning`, ..) tells them apart from records.

### Exit codes

The exit codes of the [`nabic` CLI interface](901-nabic-cli.md) are defined as follows.

| Code  | Meaning          |
|-------|------------------|
| `0`   | Success          |
| `1`   | User code error  |
| `2`   | CLI error        |
| `70`  | nabic internal panic |

`nabic test` builds the harness; it does not run the tests. `nabi test` runs them and exits with `1` if any fails. Each test runs in a separate process, so a test's panic (`101`) does not become the runner's `101`.

CI can apply different handling per code (retry, notification, etc.).

## 9.9.13. LSP server

`nabi lsp` is the LSP (Language Server Protocol) server ([9.10.8](910-nabi-frontend.md#nabi-lsp-lsp-server)). The same server is also provided as the standalone binary `nabils`. It communicates over stdio and provides diagnostics, hover (including members), go-to-definition (returning the range of the defining name), references, rename, document symbols, member / path completion, signature help, quick fixes, formatting and semantic tokens (macro calls, user-defined keywords, literal suffixes, operators and attributes, which the static grammar cannot recognize). References and rename are buffer-local.

```
nabi lsp
nabils
```

In this mode, LSP clients such as VS Code, Vim and Emacs receive completion, hover, go-to-definition, diagnostics and so on. Internally it uses the incremental front end of the compiler linked into `nabi`, the one `nabic check` runs.

- **Completion.** After `x.`, the fields and methods of `x`'s type. Of a type from the standard library or another package only the `pub` ones are offered, without `#[doc(hidden)]` members and without the methods of operator traits (those written as `a + b`, `xs[i]` and so on). After `a::`, also in `use a::b::`, the modules and items under `a`. Elsewhere, the locals in scope at the cursor, the buffer's items, the names in scope without a path (the core library's and those the `use` items import) and keywords. The REPL and the Jupyter kernel complete with the same engine ([9.7.8](907-jit-cells.md#978-input-unit-repl-recognition)).
- **Hover and definition.** Hover on a function, method or field, the standard library's included, shows its signature and then its doc comment. Go-to-definition on `x.name` looks `name` up in the type of `x`; on a standard library item it goes to the item in the library's source.
- **Signature help.** Inside the parentheses of a call (asked on `(` and `,`), the callee's signature and doc comment, with the parameter under the cursor active. A method is looked up in its receiver's type, and its `self` is not counted.
- **Quick fixes.** A diagnostic that carries an edit (for example ``convert with `as i32` ``) offers it as a `quickfix` code action. Without one, a help line that says what to write becomes one: ``add `;` `` inserts `;` at the end of the diagnostic's span, ``add `use pkg::a::b;` `` inserts that line after the file's last top-level `use` (with none, after its leading `//!` lines and inner attributes), and ``nabi add <name>`` becomes the command `nabi.add`. On `workspace/executeCommand` the server runs `nabi.add` as `nabi add <name>` in the package directory (for a script, `nabi add <name> <file>` in the file's directory), with the `nabi` named by `$NABI` or else the one beside the server, then resolves the dependencies again and checks the open buffers again, so the new package is found without a restart. It runs no other command and takes no flags.
- **Rename.** `prepareRename` returns the range of the name under the cursor when rename would take it. On a name declared outside the buffer it fails with a message that says where, for example ``cannot rename `println`: it is defined in the standard library``; on a keyword, `` `let` is a keyword ``.
- **Document symbols.** A symbol's range covers the whole item, from its modifiers (`pub`, `const`, ..) to its closing `}` or `;`; its selection range is the name.
- **Diagnostics as you type.** Diagnostics are published after every change of a buffer (`didOpen`, `didChange`), not only on save. The check runs in the background, one at a time, once no edit has come for 30 ms; edits that arrive meanwhile are checked together, on the latest text, and the diagnostics of a check whose buffer changed while it ran are not published (they would point into older text). They are the compiler's own, with the same text and code, so a lexical problem (an invisible or look-alike character, a lone combining mark, a bidirectional control in a comment, a malformed number, a bad escape) and a literal no handler takes (`10Hz`, `95%`, `dat"..."`) show while they are typed, each with the quick fix its diagnostic carries.
- **Dependencies.** Each buffer is checked with the dependencies of its package: the nearest directory above it with `nabi.nabion`, `src/main.nabi` or `src/lib.nabi`, resolved from its manifest and lock and built as `nabi lsp` started there would build them. A buffer outside every package but inside a workspace folder (`workspaceFolders`, else `rootUri`) uses the environment's libraries ([9.10.5](910-nabi-frontend.md#9105-environments)); outside them all, the dependencies of the directory the server started in. Each package is resolved once; a saved manifest or lock (`didSave`), a change the editor reports for one (`workspace/didChangeWatchedFiles`, for which the server registers `**/nabi.nabion` and `**/nabi.lock` when the client allows it) and a change of the workspace folders resolve again and check the open buffers again. A package whose dependencies cannot be resolved is checked without them, and the server says why once (`window/showMessage`).
- **Answers while a check runs.** Hover, go-to-definition, completion, signature help, document symbols and formatting do not wait for the check of an edit: they answer from the last check that type-checked, mapping the position through the edit since. A position on text edited since that check (the name under the cursor included) waits for a check of the current text instead, as do references and rename, which edit by the buffer as it is now. Completion after `x.` takes the type of `x` from that check when only the line being typed changed and `x` is written there as before; otherwise it checks the buffer up to the `.`. A request the client cancels (`$/cancelRequest`) before it is answered gets the error `RequestCancelled` (-32800).
- **Diagnostic ranges.** A diagnostic whose span is empty is reported over the token at that position, or at the end of a line over the token before it.
- **Started by hand.** On a terminal the server first prints ``nabils speaks the language server protocol on stdin; start it from an editor (see `nabi help lsp`)`` to standard error. When the input ends before the client sends `initialize`, it prints ``nabils: input ended before the editor sent `initialize` `` and exits with 1. After `initialize` the exit code is 0 after `shutdown` and 1 otherwise, as the LSP standard says.

The detailed LSP specification is outside this specification (the LSP standard + the toolchain's implementation decisions) and is defined separately.

## 9.9.14. Package inspection

Information about a prebuilt library package can be viewed at once.

```
nabi inspect path/to/libfoo.a        # or foo.nabihir; the sibling file is read too
```

This command shows information from the NABIHIR ([9.4](904-nabihir-bytecode.md)) and the [archive](../000-meta/007-glossary.md#archive) as follows.

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

`nabi inspect --dump <file>` prints the NABIHIR's version, producer, meta and all interface files (9.9.2).

This gives an at-a-glance view of what the package exposes externally.

## 9.9.15. Summary

The tools above are the standard tools Nabi works with (gdb, lldb, perf, Jupyter) and Nabi's own (`--emit`, `nabi demangle`, `nabi inspect`, the LSP server). The front end `nabi` and its tools are defined in [9.10](910-nabi-frontend.md), the nabi-mir file format in [9.11](911-mir-format.md).
