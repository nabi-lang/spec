# 8.11. 代码生成器契约

本节汇总代码生成器必须保证的事项，使它写出的目标文件能与 `nabic` 的链接器链接、能在运行时上运行，并能与其他代码生成器写出的目标文件混用：这里的代码生成器包括 `nabic` 自带的（[8.7](807-code-generation.md)）和单独安装的（8.11.13）。其中大部分内容在本章其他地方定义；本节为每项要求命名，并链接到其定义。若 `nabic` 的代码生成器做得比契约要求的更多，正文会注明。

> [!NOTE]
> 0.2 草案。在 0.1 中，`nabic` 自带的代码生成器是唯一的代码生成器，下文的每项要求都是它的实际行为。8.11.13 的进程接口及其读取的文件格式（[9.11](../009-tools/911-mir-format.md)）尚未实现：0.1 的 `nabic` 不会运行独立的代码生成器，`-C codegen-backend` 是未知的代码生成选项。

## 8.11.1. 分工

代码生成器把一个代码生成单元（[9.2.16](../009-tools/902-aot-pipeline.md#代码生成单元)）的函数转换为一个可重定位目标文件。凡是多个目标文件必须达成一致的事项，都由 `nabic` 在任何代码生成器运行之前决定，并把结果连同该单元的 nabi-mir 一起交出：

| `nabic` 决定 | 代码生成器决定 |
|-----------------|----------------------------|
| nabi-mir 的优化（内联，[8.7.3](807-code-generation.md#873-优化级别)）；协程状态机（[8.10](810-lowering-coroutine-generic.md)） | 单个函数内的指令选择、寄存器分配、调度和帧布局 |
| 各单元，以及每个单元定义哪些函数 | 在 8.11.6 和 8.11.7 的规则范围内的自身优化 |
| 每个函数和静态变量的符号、链接属性与可见性（8.11.3） | 它自己的局部符号（8.11.3） |
| 每个类型的布局（8.11.5） | |
| GC 类型 id 及其 ScanTable（[8.2](802-scan-table.md)） | |
| 哪些函数从不到达安全点、哪些在入口处轮询、哪些回边轮询（8.11.7） | 轮询在被轮询的边上所处的位置 |
| 共享单元：C `main`、静态变量、`nabi.register_types`、惰性静态变量的初始化器（[9.2.16](../009-tools/902-aot-pipeline.md#代码生成单元)） | |
| 链接（[9.2.14](../009-tools/902-aot-pipeline.md#9214-链接可执行文件)） | |

共享单元总是由 `nabic` 自带的代码生成器编译。同一程序中由不同代码生成器生成的目标文件，以及由另一个代码生成器重新编译的同一单元的目标文件，都必须能够链接，且行为相同。

## 8.11.2. 输入

输入是一个单元经过优化的 nabi-mir（[9.11](../009-tools/911-mir-format.md)）：该单元定义的函数及其主体；它们引用的每个函数和静态变量的声明；它们使用的类型、布局和 GC 类型；目标；以及构建选项（`-O` 级别、溢出检查、`-g`）。

这份 nabi-mir 就是 `--emit=mir` 打印的那一份（[9.9.1](../009-tools/909-debug-tools.md#991-中间输出--emit)）。其中没有协程挂起点（此时协程已经是基于帧对象的状态机，[8.10.1](810-lowering-coroutine-generic.md#8101-状态机)），也没有向下取整的 `%%`（它已被展开）。代码生成器按每条语句在 nabi-mir 中的含义编译它，并按程序顺序保留每个值、每个 trap 和每个副作用，这与 [8.7.3](807-code-generation.md#873-优化级别) 对优化器的要求相同。

## 8.11.3. 符号与链接属性

每个函数和静态变量都在输入中带有自己的符号：修饰后的名字（[9.5](../009-tools/905-symbol-mangling.md)）、`#[export_name]`，或 `extern` 声明的 C 名字（在 i686 Windows 上带 `@N` stdcall 后缀）。代码生成器按原样使用这些名字，不做修饰。它完全按照输入的说明把符号定义为全局、弱或隐藏；被其他单元调用的函数是全局的，另一个包代码的私有副本命名为 `<symbol>.copy_Y<hash>_`，并且只在该包的各单元内是全局的。

`nabic` 及其代码生成器还使用以下派生符号：

| 符号 | 定义者 | 含义 |
|--------|-----------|------------|
| Nabi 函数的 `<symbol>` | 定义该函数的单元 | 函数主体，采用带隐藏 `nabi_tls` 参数的 Nabi 调用约定（[8.6.6](806-calling-convention.md#866-nabi-到-nabi-的调用约定)） |
| `<symbol>.c` | 定义不带 `#[export_name]` 的 `extern "C"` 函数的单元（若带有，则为导出名） | 其 C 入口（8.11.8） |
| `<symbol>.v` | 取该函数地址的每个目标文件，作为局部符号 | 函数值调用的 thunk：没有 `nabi_tls` 参数；它读取该块的地址并调用 `<symbol>`（[8.6.11](806-calling-convention.md#8611-函数指针-abi)） |
| `<static>.state`、`<static>.init` | 共享单元 | 惰性静态变量的状态字节和初始化器（8.11.10） |
| `_Y<pkg>_gctype.<hash>` | 共享单元，在单元构建中 | 存放 GC 类型 id 的 `u32`（8.11.6） |
| `nabi.scan.<hash>` | 共享单元，弱且隐藏 | ScanTable 的字节，以其内容命名（[8.2.13](802-scan-table.md#8213-注册时机)） |
| `nabi.register_types`、`nabi.register_types.<lib>` | 共享单元 | GC 类型和静态根的注册（[8.2.13](802-scan-table.md#8213-注册时机)） |
| `nabi.any.<hash>` | `any` 分派器所在的单元；在库中为弱符号 | `any` 方法分派器（[9.4.8](../009-tools/904-nabihir-bytecode.md#948-分离编译的规则)） |
| `nabi.str.<hash>` | 任何目标文件，作为局部符号 | 字符串字面量的字节，后接一个 NUL 字节 |
| `nabi_tls` | 运行时 | 线程局部块（[8.3](803-tls-slots.md)） |

代码生成器自己创建的其他任何符号都是其目标文件的局部符号。`nabic` 自己使用以 `nabi.` 开头的名字（`nabi.i386.udivmod64`，..）。

每个符号名的字节都必须原样、不经压缩地出现在目标文件中：`nabic` 写出单元的目标文件时，用占位符代替包哈希，并在链接前把目标文件字节中的 `_Y<placeholder>_` 替换为 `_Y<hash>_`（[9.2.16](../009-tools/902-aot-pipeline.md#代码生成单元)）。

字符串字面量的字节以一个 NUL 字节结尾，该字节不属于 `str`：抛出点（[8.11.11](#81111-错误panic-与栈展开)）和作为 `char*` 传递的 `str` 都读到它为止。

## 8.11.4. 调用约定

- Nabi 函数之间：[8.6.6](806-calling-convention.md#866-nabi-到-nabi-的调用约定)。Nabi 函数可能从其他目标文件收到的每个调用都必须严格遵循它，包括作为最后一个参数的隐藏 `nabi_tls` 地址；超过 64 字节的聚合类型按地址传递，并通过开头的结果指针返回。只有其他目标文件无法访问的函数（局部的、从不作为值使用）才可以使用自己的调用约定。
- `extern "C"` 声明和定义，以及通过 C 函数指针的调用：采用带聚合分类的平台 C ABI（[8.6.1](806-calling-convention.md#861-默认平台-c-abi) ~ [8.6.3](806-calling-convention.md#863-struct-实参--返回小-struct-规则)）、窄整数扩展、i686 Windows 上 `extern "system"` 的 stdcall，以及 C 可变参数（[8.6.10](806-calling-convention.md#8610-可变参数函数)）。输入给出 `nabic` 对每个此类签名所做的分类（[9.11.9](../009-tools/911-mir-format.md#9119-函数)）；代码生成器按分类放置各部分，不再重新分类。
- `extern "C"` 调用的 `str` 实参作为 `char*` 传递：代码在调用前调用 `nabi_str_cstr(s)`，传入它返回的指针，并在调用后调用 `nabi_str_cstr_done(s, p)`（若 `s` 的字节后紧跟一个 NUL 字节，该指针就是 `s` 自己的指针，否则是一个被固定的副本）。
- `extern "nabi"` 运行时声明：采用 C ABI，`str` 作为按值传递的 `(ptr, len)` C 结构体（[8.5](805-extern-symbols.md)）。
- 函数值：`{code, env}`，环境在前，没有 `nabi_tls` 参数（[8.6.11](806-calling-convention.md#8611-函数指针-abi)、[8.6.12](806-calling-convention.md#8612-闭包调用)）。
- 会抛出错误的函数返回其载体 `(is_err, payload)`（[8.6.5](806-calling-convention.md#865-抛出函数的返回基于值的错误传播)）；在 nabi-mir 中，这是一个类型由输入给出的普通返回值。
- 栈探测遵循目标的 `stack_probes`（[0.6.5](../000-meta/006-platform-targets.md#065-目标规格)）：一页或更大的帧按顺序触及每一页，可以内联完成，也可以通过 `__chkstk`（Windows；由 `nabic` 的启动目标文件定义）完成。运行时把保护区内的故障报告为栈溢出（[8.5.7](805-extern-symbols.md#栈溢出)）。

## 8.11.5. 数据布局

每个 nabi-mir 类型的布局由目标确定，输入给出每个类型的大小、对齐和字段偏移（[9.11.6](../009-tools/911-mir-format.md#9116-类型)）。代码生成器按原样使用它们：

| nabi-mir 类型 | 布局 |
|---------------|--------|
| `()` | 大小 0，对齐 1 |
| `i8` `u8` `bool` `c_char` | 1 字节；`bool` 为 0 或 1 |
| `i16` `u16` | 2 字节 |
| `i32` `u32` `f32` `char` | 4 字节；`char` 是 Unicode 标量值 |
| `i64` `u64` `f64` | 8 字节，按目标的 `wide_align` 对齐（i686 Linux 上为 4，其他为 8） |
| `isize` `usize`、GC 引用、函数指针、未解析的字 | 一个指针宽度的字 |
| `str` | 两个字 `(ptr, len)`；`ptr` 可以指向 GC 对象内部 |
| 元组、结构体 | 按字段顺序的 C 布局；`#[repr(packed)]`：无填充，对齐 1；`#[repr(align(N))]`：至少为 `N` |
| `[T; N]` | `N` 个大小为 `T` 的元素 |
| `n` 个字的联合体 | `n` 个字，按其对齐要求最高的变体对齐，且至少按字对齐 |
| 和类型、枚举 | `(i32 tag, payload)` 或 niche 布局（[8.8.6](808-lowering-types.md#886-enum-的表示)、[8.8.8](808-lowering-types.md#888-和类型a--b的表示)） |
| `any Tr` | `(u32 tag, payload word)`（[8.8.9](808-lowering-types.md#889-any-槽的表示)） |
| 函数值 | `(code, env)` |

除 `s390x` 外，所有目标都是小端。GC 对象是一个 16 字节的头（[8.1](801-object-header-abi.md)）后接其载荷；其大小按 [8.1.7](801-object-header-abi.md#817-对象头--载荷的总大小) 计算。

## 8.11.6. GC：分配、根和屏障

**分配**。代码生成器分配的每个 GC 对象（nabi-mir 中的 `HeapBox`、`HeapAlloc`、`BufAlloc`）都获得输入随之给出的类型 id：一个常量，或在单元构建中位于 `_Y<pkg>_gctype.<hash>` 的 `u32`。不宽于一个字的数字、`bool` 或 `char` 的 `HeapBox` 不分配任何东西：载荷字就是该值（[8.2.7](802-scan-table.md#827-any-槽)）。分配走 [8.9.9](809-lowering-control.md#899-gc-分配快速--慢速路径) 的 TLAB 快速路径，或调用 `nabi_gc_alloc_slow(size, align, type_id)`；大小达到 `LOS_THRESHOLD`（8 KiB）或以上的对象，或对齐超过 8 的对象，总是调用后者。快速路径写入 `type_id` 和 `gc_word`，并让 `gc_flags` 保持为零；载荷为零。代码生成器总是可以改为调用 `nabi_gc_alloc_slow`。

**根**。在每个安全点（8.11.7），每个跨越该安全点仍然存活的局部变量中，可能存放 GC 引用的每个字都位于内存中，并注册在一个链入 `nabi_tls` 偏移 0 的影子栈帧中（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-根跟踪编译器直接生成的影子栈)、[8.9.12](809-lowering-control.md#8912-影子栈帧)）：

- 该帧是位于机器栈上（在 wasm32 上位于线性内存中）的 `{prev: ptr, count: usize, slots: [ptr; count]}`；每个槽存放一个字的地址，或者一个槽存放 `(n << 1) | 1`、下一个槽存放 `n` 个相邻字中第一个的地址。当 `count` 为 0 时，该帧只有一个槽：只读数据中一个描述符 `{count: u16, entries: [u16; count]}` 的地址，其条目即各槽，每个地址替换为该字相对于帧的偏移（以字为单位、有符号）的两倍（[8.9.12](809-lowering-control.md#8912-影子栈帧)）。局部变量需要的字就是输入作为该类型根偏移给出的那些字：任何变体可能存放引用的每个字、`any` 的载荷字、`str` 的指针字。
- 帧在每条路径的第一个安全点之前链入，并在链入了它的路径的每次返回之前解除链接；在路径汇合处，要么全部已链入，要么全部未链入。只要回收可能读取已注册的字，该字就为零或一个有效值：链入之前清零，每次写入时保持最新。
- 跨越安全点存活的内容：安全点之后存活的局部变量；在调用运行时函数或 C 时，还包括其操作数；在调用 Nabi 函数时，只包括调用者之后会读取的内容（被调用者自己为其参数登记根）。被取地址（`AddrOf`）的局部变量，以及函数列为被固定的每个局部变量，在其 `StorageDead` 之前保持注册。
- 只被回边轮询看到的局部变量，也可以改为仅由轮询的慢速路径注册（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-根跟踪编译器直接生成的影子栈) 规则 4）。`nabic` 的代码生成器在入口处链入一个帧，并把所有这类局部变量注册在其中。
- 已死亡局部变量的字可以清零（`StorageDead`，即其最后一次使用结束时）。这可以避免它保留垃圾对象；但并非必须。

**寄存器与移动的对象**。完全回收可能移动对象（疏散，[7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)）；它会改写堆中每个精确引用，但从不改写根字。当已注册的字、不确定字或固定指向某个对象时，该对象留在原处，大对象也从不移动。因此：

- GC 引用只有作为某个跨越安全点注册的字的副本时，才可以跨越该安全点留在寄存器中；指向该对象内部的指针也是如此。
- 其他任何 GC 引用或指向 GC 对象内部的指针，例如从堆对象字段加载的引用，都不在安全点之后存活：安全点之后要重新加载。代码生成器不得把这样的值保存在未跨越安全点注册的寄存器、溢写槽或栈槽中。
- `nabic` 的代码生成器不在寄存器中保存任何根：跨越安全点持有 GC 引用的变量位于其已注册的栈槽中。

**写屏障**。每次把带有 GC 字的值存入 GC 对象（堆对象的字段、缓冲区元素、协程帧槽）时，都带有 [8.9.10](809-lowering-control.md#8910-写屏障) 的屏障：测试目标对象的 `gc_flags & (UNLOGGED | CARDED)`；若为 `UNLOGGED`，调用 `nabi_gc_wb_slow(obj, slot, len)`（槽的地址和所存值的大小）；若为 `CARDED`，则在 `card_table + ((slot - obj) >> 9)` 处把 `1` 存入被写槽所跨越的卡字节，卡表的地址位于头之前的那个字。屏障与其存储之间没有安全点。在下一个安全点之前存入刚分配的对象、存入静态变量、存入原始内存，以及存储不带 GC 字的值，都不需要屏障。屏障从不读取所存值的头。

回收器对编译后的代码没有其他要求（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-根跟踪编译器直接生成的影子栈)）：没有栈映射，也没有寄存器映射。

## 8.11.7. 安全点与轮询

安全点是线程可以为回收而停下的位置：调用输入未标记为从不到达安全点的 Nabi 函数、通过函数值调用、调用非 `#[no_safepoint]` 的运行时函数、调用 C（它会进入驻留状态，8.11.8）、分配、读写惰性静态变量或取其地址，以及轮询。

- **轮询**。加载 `nabi_tls + 10p+8` 处的 `u8`（`yield_flag`）；若非零，调用 `nabi_runtime_yield()`（[8.4.1](804-safepoint.md#841-yield-标志检查快速路径)）。
- **回边**。输入标记为被轮询的每条回边上都有一次轮询（[8.4.8](804-safepoint.md#848-循环回边轮询)）。未标记的回边属于至多 64 次迭代且不到达任何安全点的循环（[8.4.2](804-safepoint.md#842-生成位置)）。
- **入口**。输入标记为可重入的函数，在每条路径的第一个安全点之前轮询；若它还会以直接 Nabi 调用以外的方式进入（从 C、通过函数值、被导出、`any` 分派器），并且确实会到达安全点，则在入口处轮询（[8.4.7](804-safepoint.md#847-安全点不变式)、[8.9.11](809-lowering-control.md#8911-安全点轮询)）。
- 输入标记为从不到达安全点的函数不需要帧，也不需要轮询，对它的调用也不是安全点。

允许额外的轮询。向量化循环内部没有轮询，并且在标量循环的轮询再次运行之前至多运行 1024 次迭代（[8.7.8](807-code-generation.md#878-循环向量化)）。

## 8.11.8. 调用 C 与从 C 调用

- **调用 C**。在每次调用符号不以 `nabi_` 开头的 `extern "C"` 函数前后：调用前，以顺序一致的方式把 `2`（PARKED）存入 `mutator_state`（`nabi_tls + 10p+9`），然后加载 `yield_flag`，若非零则调用 `nabi_enter_native()`；调用后，存入 `0`（RUNNING），做同样的加载，若非零则调用 `nabi_leave_native()`（[8.4.10](804-safepoint.md#8410-原生调用)）。调用者的根在调用期间保持注册。
- **从 C 调用**。`extern "C"` 定义有一个 C 入口（`<symbol>.c` 或其导出名），它接受 C 参数，调用 `nabi_native_reenter()`（它也会附加运行时不认识的线程，[8.3.8](803-tls-slots.md#838-编译器生成的形式)），用本线程的 `nabi_tls` 地址调用函数主体，调用 `nabi_native_resume(<what reenter returned>)`，然后按 C 的方式返回。在 wasm32 上，若该函数被导出，它会先调用 `nabi_init_once()`。
- 对 `#[no_safepoint]` 运行时函数和 `nabi_*` 符号的调用不加这层包围。

## 8.11.9. 线程局部访问

Nabi 函数从它作为最后一个参数收到的地址出发，按偏移（[8.3.1](803-tls-slots.md#831-tls-槽结构)）读取 `nabi_tls` 槽。从 Nabi 外部进入的代码（C 入口、函数值 thunk、被导出的 Nabi 函数、`any` 分派器）只计算一次该地址（[8.7.7](807-code-generation.md#877-tls-访问)）。访问序列以及相应的重定位取决于目标文件格式：

| 目标 | 模型 | `nabic` 使用的重定位 |
|--------|-------|--------------------------|
| ELF x86_64 / aarch64 / i686 / armv7 / riscv64 / powerpc64le / s390x | initial exec，线程指针加上 GOT 槽中的偏移 | `R_X86_64_GOTTPOFF`、`R_AARCH64_TLSIE_*`、`R_386_TLS_GOTIE`、`R_ARM_TLS_IE32`、`R_RISCV_TLS_GOT_HI20`、`R_PPC64_GOT_TPREL16_*`、`R_390_TLS_IEENT` |
| ELF riscv32（无操作系统） | local exec | `R_RISCV_TPREL_HI20` / `_ADD` / `_LO12_I` |
| Mach-O arm64 | 线程局部变量描述符，通过其 getter 调用 | `ARM64_RELOC_TLVP_LOAD_PAGE21` / `_PAGEOFF12` |
| COFF | 位于 `_tls_index` 的 TEB TLS 数组，加上相对于节的偏移 | `SECREL`（arm64：`SECREL_HIGH12A` / `_LOW12A`） |
| 带线程的 wasm32 | `__tls_base` 加上偏移 | `R_WASM_MEMORY_ADDR_TLS_SLEB` |
| 不带线程的 wasm32 | 普通全局变量 | 内存地址 |

运行时总是静态链接，因此这些模型就足够了；在链接器能够解析的情况下，代码生成器可以改用 general dynamic。

## 8.11.10. 静态变量

静态变量的存储、初始字节和根注册都属于共享单元。代码生成器通过静态变量的符号读写它；若目标文件格式有 GOT，且该静态变量定义在程序自身的目标文件之外（在预构建库或 C 库中），则通过 GOT 读写；在 Windows 上，对由 DLL 定义的静态变量，则通过 `__imp_<symbol>` 读写。对静态变量的存储没有写屏障（静态变量是根）。

在第一次读写惰性静态变量或取其地址之前，代码以原子方式加载其状态字节 `<static>.state`；除非它为 `2`（已初始化），否则以访问的源代码位置（`path:line:col`）调用 `<static>.init(at: str)`，采用不带 `nabi_tls` 参数的 Nabi 调用约定。这是一个安全点。

## 8.11.11. 错误、panic 与栈展开

Nabi 程序中没有任何东西会展开机器栈。

- `throw` 和 `try` 是基于载体的普通返回和分支（[8.6.5](806-calling-convention.md#865-抛出函数的返回基于值的错误传播)、[8.9.6](809-lowering-control.md#896-try--catch--finally基于值的分支)）；每个帧都经由自己的返回路径离开，并解除其影子栈帧的链接。
- `throw` 通过调用 `nabi_throw_site(at: str)` 记录其位置；`nabic` 的代码生成器则改为把该字面量的地址存入运行时读取的由实现定义的槽（[8.3.1](803-tls-slots.md#831-tls-槽结构)），代码生成器只有在与其随附的运行时一起使用时才可以这样做。
- panic 是对一个不返回的函数（`nabi_panic_at`，..）的调用；它结束进程，不运行任何 `finally`。代码生成器自己生成的检查通过 `nabi_panic_at(msg, at)` 触发 panic，其中 `at` 是正在执行的语句的位置（`path:line:col`），在 `#[track_caller]` 函数中则是其最后一个参数；运行时按 [4.14.13](../004-functions/414-throws.md#41413-panic-与致命错误) 所示的形式打印它。各检查及其消息如下：

| 检查 | 条件 | 消息 |
|-------|------|---------|
| 整数 `+` `-` `*` 溢出 | 开启溢出检查，不是 `wrapping_*`，操作数不能保证常量安全 | `attempt to add with overflow`、`attempt to subtract with overflow`、`attempt to multiply with overflow` |
| 对最小值取负 | 开启溢出检查 | `attempt to negate with overflow` |
| 除法和取余 | 总是 | `attempt to divide by zero`、`attempt to divide with overflow`、`attempt to calculate the remainder with a divisor of zero`、`attempt to calculate the remainder with overflow` |
| 移位量达到或超过位宽 | 总是 | `attempt to shift left with overflow`、`attempt to shift right with overflow` |
| 数组索引（`ArrayIndex`、`ArrayWrite`） | 总是，除非索引是范围内的常量 | `index out of bounds` |

其他检查（切片、`List`、`unwrap`）在 nabi-mir 中已经是调用。

展开表是为调试器、性能分析器和操作系统准备的，不是为程序准备的。代码生成器为每个函数生成其目标文件格式的展开信息：ELF 上为 `.eh_frame` FDE（armv7：`.ARM.exidx` 条目），Mach-O 上为 compact unwind 或 `__eh_frame`，COFF x86_64 和 arm64 上为 `.pdata` / `.xdata`（i686 没有；通过 `ebp` 遍历帧）。没有 CIE 指定 personality 例程，也不生成 LSDA；在 armv7 上，紧凑的 `.ARM.exidx` 条目隐含 `__aeabi_unwind_cpp_pr0`，运行时以弱符号定义它。`nabic` 的代码生成器在每个函数中都保留帧指针，因此性能分析器无需展开表也能遍历栈。

## 8.11.12. 目标文件

代码生成器以目标的格式写出一个可重定位目标文件，供 `nabic` 的链接器读取（[9.2.14](../009-tools/902-aot-pipeline.md#9214-链接可执行文件)）：

| 格式 | 要求 |
|--------|--------------|
| ELF | `ET_REL`，采用目标 ABI 的机器类型和标志（powerpc64le：ELFv2，`e_flags` 2；armv7：EABI5 硬浮点；riscv：RVC，在 riscv64 上还采用双精度浮点 ABI）。64 位目标：只用 RELA 重定位；32 位：REL 或 RELA。不得有压缩的可分配节（`SHF_COMPRESSED` 调试节会被丢弃）。一个不带 `SHF_EXECINSTR` 的 `.note.GNU-stack` 节 |
| Mach-O | `MH_OBJECT`，仅 arm64，`MH_SUBSECTIONS_VIA_SYMBOLS` |
| COFF | 目标的机器类型；i686 的 C 名字带 `_` 前缀；超过 65279 个节时使用 `/bigobj` 格式 |
| wasm | 遵循 WebAssembly 工具约定的可重定位模块（`linking` 节版本 2、`reloc.CODE` / `reloc.DATA`），从 `env` 导入 `__linear_memory`、`__stack_pointer`、`__indirect_function_table`，带线程时还导入 `__tls_base`；每个函数的类型由其调用约定给出，`nabi_tls` 地址是最后一个 `i32` 参数 |

- 每个函数和每个数据对象都应当位于自己的节（ELF）、COMDAT（COFF）、子节（Mach-O）或自己的段（wasm）中：链接器丢弃未被引用的部分，并在 `release` 构建中合并相同的函数（[9.2.15](../009-tools/902-aot-pipeline.md#9215-输出)）。共享同一个节的内容会被一起保留或一起丢弃，且从不被合并。
- 只可以出现链接器能解析的重定位种类：`nabic` 的代码生成器写出的种类，以及运行时库和工具链随附的 C 库中的种类。其他任何种类都是链接错误，错误会指出它（`unsupported relocation ..`）。
- 指令集缺少的例程是对平台支持例程的调用，这些例程由运行时库提供：在 armv7 上为 `__aeabi_idiv`、`__aeabi_ldivmod`、`__aeabi_l2d`、`__mulodi4`，..；在 riscv32 上为软浮点例程和 `__divdi3`，..（[8.7.4](807-code-generation.md#874-运算)）。
- 除 `sqrt` 外的浮点方法是输入中已有的 `nabi_f64_*` / `nabi_f32_*` 调用；代码生成器不在编译期折叠它们。
- 输出是确定性的：相同的输入得到相同的字节。

**调试信息**是可选的。当构建带有 `-g` 时，代码生成器应当像 [9.9.6](../009-tools/909-debug-tools.md#996-调试器集成gdb--lldb) 那样描述其函数：函数以其 Nabi 路径命名（[9.5.10](../009-tools/905-symbol-mangling.md#9510-调试信息)），行表来自各语句的位置，参数和具名局部变量带有输入所描述的类型。链接器读取 32 位格式的 DWARF 2 到 4（`.dSYM` 包会跳过带有 DWARF 5 或 64 位 DWARF 的目标文件），在 COFF 上读取 `.debug$S` / `.debug$T` 中的 CodeView C13，并把它合并到 PDB 中。调试信息不得改变代码。

## 8.11.13. 单独安装的代码生成器

> [!NOTE]
> 0.2 草案。本小节的内容均未实现。

`nabic` 自带代码生成器以外的代码生成器是一个独立的可执行文件，是用 `nabi component add <component>` 安装的工具链组件：`codegen-llvm` 安装 `nabic-codegen-llvm`，其中包含它自己静态链接的固定版本 LLVM 副本。`nabic` 本身不包含其他代码生成器，也不链接任何外部代码生成器（[9.2.17](../009-tools/902-aot-pipeline.md#9217-编译器与运行时的分离)）。

**选择**。使用 `-C codegen-backend=<name>`（`native` 为默认值，或组件名，如 `llvm`），或在 `nabi.nabion` 的构建配置中写 `codegen_backend: "<name>"`（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)），选项会覆盖后者。该选择适用于构建所编译的每个单元；预构建库保留它们已有的目标文件，共享单元、REPL 单元（cell）和 Jupyter 单元总是使用 `native`。若某个名字对应的组件未安装，就是错误，错误会指出安装它的命令：``codegen backend `llvm` is not installed; run `nabi component add codegen-llvm` ``。组件为 `nabic` 支持的每个目标编译，否则要说明不支持哪些（`E0129`）。

**内核组件**。组件可以只编译部分函数：`codegen-mlir` 编译库为它标记的函数，单元的其余部分仍由所选的代码生成器编译。`nabic` 把这类函数放进它们自己的单元。函数如何标记尚未规定。

**查找**。`nabic` 从它自己的 `bin/` 目录（工具链的目录，[9.2.19](../009-tools/902-aot-pipeline.md#9219-安装布局)）运行 `nabic-codegen-<name>`；它不搜索 `PATH`。组件按工具链安装，其版本就是工具链的版本。

**调用**。一个进程编译一个单元：

```
nabic-codegen-<name> <unit>.nabimir -o <unit>.o
```

- `<unit>.nabimir` 是该单元的 nabi-mir，格式见 [9.11](../009-tools/911-mir-format.md)。构建选项包含在其中；没有其他参数。
- `-o` 指定要写出的目标文件。`nabic` 只在退出状态为 0 后才读取它；若为其他状态，则忽略已写出的任何内容。
- `nabic` 同时运行的进程数与其核心数相同，并在使用 `--keep` 时保留失败单元的输入（[9.1.2](../009-tools/901-nabic-cli.md#nabic-buildaot-构建)）。
- 工作目录是构建的工作目录；环境是 `nabic` 的环境。

**诊断**。进程把诊断写到标准输出，每行一个 JSON 对象，格式同 `--message-format=json`（schema 1，[9.9.4](../009-tools/909-debug-tools.md#json-格式工具集成)）。其范围是输入中各语句的位置。`nabic` 把它们当作自己的诊断，按构建要求的格式打印。组件无法编译的构造是一个指出该构造的错误 `E0129`，位于使用它的语句处；不会静默回退到另一个代码生成器。标准错误原样传递。

**退出状态**。

| 状态 | 含义 |
|--------|---------|
| `0` | 已写出目标文件 |
| `1` | 已在标准输出上报告错误 |
| `2` | 用法错误，或组件不能读取的输入格式版本（它会指出两个版本） |
| `70` | 组件的内部错误 |

**版本**。`nabic-codegen-<name> --version` 在第一行打印 `nabic-codegen-<name> <version>`，并打印一行 `mir: <major>.<minor>`，指出它读取的输入格式版本；`nabic` 拒绝其他工具链版本的组件，错误会指出重新安装它的命令。组件的身份（其版本行及其文件的时间戳，与 [9.4.4](../009-tools/904-nabihir-bytecode.md#944-头部与生成者) 中的 `nabic` 相同）是每个单元的键的一部分（[9.2.16](../009-tools/902-aot-pipeline.md#代码生成单元)），因此组件一旦变化，它编译过的单元都会重新编译。

## 8.11.14. 验证

代码生成器通过 [8.7.9](807-code-generation.md#879-验证翻译) 的端到端测试来检查，测试程序中包含它的目标文件，并使用 GC 压力设置（`NABIR_GC_STRESS`、`NABIR_GC_EVAC=always`，[9.9.8](../009-tools/909-debug-tools.md#998-内存--gc-工具)）：在这些设置下，每次分配都可能触发回收，每次完全回收都会移动它所能移动的每个对象。链接了两个代码生成器的目标文件、每个单元来自其中任一个的程序，其行为与由一个代码生成器构建的程序相同。
