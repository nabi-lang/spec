# 0.7. 术语表

同样的概念在本规范的许多章节中反复出现，把常用术语集中到一处对读者有帮助。本文档汇集了 Nabi 规范中通篇出现的核心术语，阅读正文时一旦某个概念变得模糊，就可以直接查阅这里。每个条目给出简短的定义及其在 Nabi 中的具体语境，说明的不只是词典意义，还有该术语在规范中所处的位置。

## ABI (Application Binary Interface)

ABI 是编译后的代码在链接和运行时必须遵循的二进制层面的约定。它包括对象布局、调用约定、符号命名等。Nabi 的 ABI 在[第 8 章：ABI / 降级](../008-abi/README.md)中定义。

## AOT (Ahead-of-Time)

AOT 是预先把源代码编译成静态产物（`.a` 或可执行文件）的模式，与 JIT 相对。Nabi 同时支持 JIT 和 AOT 引擎，两者共用直到 nabi-mir 的同一条流水线。

## archive

归档文件（archive）是编译后的包的输出：一个标准 ar 格式的 `.a` 文件。它存放具体代码的目标文件；库的接口（NABIHIR）是它旁边的 `<name>.nabihir` 文件。

## block（块）

块是用 `{ ... }` 括起来的一组语句和表达式。它本身是一个表达式，其最后一个表达式成为块的值。

## block (Immix)

Immix 块是 GC 的内存单位，即堆中一块 32 KiB 的区域。在 Immix 算法中，它被划分为行（line），并作为标记 / 清除的单位。它与上面的代码块是不同的概念。

## concrete fn / generic fn

具体函数（concrete fn）是在调用点处所有类型都是具体类型的函数；泛型函数（generic fn）是仍带有类型参数的函数。在 AOT 下，泛型函数在调用点处特化为具体函数。

## dispatch（分派）

分派是确定一个名称、运算符或方法调用绑定到哪个实现的过程。除了 `any` 槽中的值在运行时按其标签分派（[2.2.3](../002-types/202-any-slot.md#223-dynamic-dispatch)）以外，分派都是静态的；由于 trait 不能作类型，任何值都不携带虚表指针。

## extern "C" 符号

extern "C" 符号是通过 C ABI 暴露或调用的符号；这类符号构成编译器与运行时之间的接口。

## FFI (Foreign Function Interface)

FFI（外部函数接口）是调用其他语言（主要是 C）中的函数，或被其他语言调用的接口。它用 `extern "C"` 块定义。

## GC (Garbage Collector)

GC 是自动内存管理机制。Nabi 的 GC 把 Immix mark-region 与 Sticky Mark-Bit 分代回收结合在一起。

## generic（泛型）

泛型是接受类型作为参数的函数或类型。它在调用点和实例化点被单态化。

## HIR (High-level IR) / MIR (Mid-level IR)

HIR 和 MIR 是编译流水线的中间表示。整体流程是源代码 → AST → HIR → MIR → 机器码（Nabi 的代码生成器）。

## Immix

Immix 是 Blackburn 和 McKinley 于 2008 年发表的 GC 算法。它以 32 KB 的块和 128 B 的行为单位使用 mark-region，并通过机会式疏散（opportunistic evacuation）处理碎片。

## JIT (Just-in-Time)

JIT 是在运行时用 Nabi 的代码生成器立即编译代码的模式。REPL 和 Jupyter 使用这种模式；它与 AOT 相对。

## lang slot / magic（魔法）

lang 槽是语言识别特定库项的机制。`#[lang(slot_name)]` 把一个标准库项连接到编译器的一个槽。

## LOS (Large Object Space)

LOS 是 GC 专门用于大对象的空间。8 KiB 或更大的对象，以及对齐大于 8 字节的对象，保存在 LOS 中且不会被移动（[7.2](../007-runtime/702-allocation.md)）。

## monomorphization（单态化）

单态化是在调用点处为每个具体类型分别编译泛型函数的过程。它以编译时间和二进制文件大小为代价，换来零间接调用开销。

## NABIHIR (Nabi HIR bytecode)

NABIHIR 是预构建库包的接口文件（`<name>.nabihir`）。它位于归档文件旁边，让下游包能够读取声明并特化泛型函数。它的主体是去掉了具体函数体的包源代码（[9.4](../009-tools/904-nabihir-bytecode.md)）。

## niche optimization

niche 优化是利用和类型或枚举中未使用的位模式来消除标签槽的优化。典型的例子是 `Ptr | none` 中的 `none = null pointer`。

## scan_table

scan_table 是 GC 用来查找对象中哪些槽是 GC 指针的元表。它按 `type_id → scan_table` 映射，并由编译器在构建时注册。

## shadow stack

影子栈（shadow stack）是 GC 用来追踪存活对象的辅助栈。编译器直接生成代码，在函数入口和出口处把存放 GC 对象变量所在槽的帧链入和移出链表。帧位于机器栈上，链表头按线程保存在 TLS 中。

## specialize

特化（specialize）是[单态化](#monomorphization)的另一个名称。它用作 AOT / JIT 流水线中某个阶段的名称。

## registry package（注册表包）

发布到注册表的 Nabi 包。它的名称是 `[a-z0-9_-]+`（以字母开头），在代码中 `-` 写作 `_`。注册表把 `-` / `_` 以及大小写不同的名称视为同一个名称。在 `nabi.nabion` 中用 `Dep::version("req")` 或字符串 `"req"` 声明对它的依赖（[9.10.4a](../009-tools/910-nabi-frontend.md#9104a-registry-packages)）。

## sticky mark-bit

sticky mark-bit 是分代 GC 用于追踪代际的位。它在第一次标记时被设置，使该对象成为“老对象”。年轻代 GC 跳过 sticky 对象以降低开销。

## sum type（和类型）

和类型是形如 `A | B`、持有其中一个备选项的类型；它是带标签联合的一种形式。它是封闭的槽：所有变体都是显式的，因此可以对模式匹配进行穷尽检查。

## TLAB (Thread-Local Allocation Buffer)

TLAB 是每个工作线程拥有的线程局部内存缓冲区。快速路径的分配只用一个 bump 指针完成，同步开销为零。

## work-stealing

工作窃取（work-stealing）是调度器的负载均衡机制。每个工作线程有一个本地双端队列，没有工作可做的工作线程会从另一个工作线程双端队列的末端窃取任务。

## unsafe block / unsafe fn

`unsafe { ... }` 块表达式和 `unsafe fn` 函数修饰符是 Nabi 用来显式划定安全抽象保证之外区域的唯一机制。裸指针解引用、调用 `unsafe fn` 等只允许在 unsafe 作用域内进行。详见 [4.15 unsafe](../004-functions/415-unsafe.md)。

## nabic / nabi / nabils / nabir

构成工具链的可执行文件和运行时。`nabic` 是编译器（相当于 rustc，[9.1](../009-tools/901-nabic-cli.md)），带有自己的代码生成器和链接器；`nabi` 是类似 cargo 的前端，负责处理清单、依赖、环境和工具链固定，调用 `nabic`，并以链接了编译器的方式运行各种工具（run、lint、REPL、Jupyter、LSP、格式化工具……）（[9.10](../009-tools/910-nabi-frontend.md)）；`nabicc` 是 C 编译器（`nabi cc`，[9.13](../009-tools/913-c-compiler.md)）；`nabils` 是供编辑器使用的 LSP 服务器可执行文件（`nabi lsp`）；`nabir` 是运行时（`libnabir.a`）。`$NABI_HOME/bin/` 中的可执行文件是指向当前工具链（`$NABI_HOME/toolchains/<version>-<triple>/`）的代理，`nabic` 在相对于其自身可执行文件的 sysroot 中查找标准库和运行时（[9.2.19](../009-tools/902-aot-pipeline.md#9219-installation-layout)）。

## NABION

Nabi Object Notation。一种使用 Nabi 字面量语法的子集来书写值的数据格式，扩展名为 `.nabion`。清单 `nabi.nabion` 和锁文件 `nabi.lock` 使用这种格式（[stdlib 3.8](../../refs-standard-libraries/300-std/308-nabion.md)）。

## packages.nabi-lang.org

Nabi 的默认包注册表，地址为 `https://packages.nabi-lang.org`。它的索引是独立的 git 仓库 `packages-index`（[9.10.4a](../009-tools/910-nabi-frontend.md#9104a-registry-packages)）。

## nabi.nabion / nabi.lock

位于包根目录的清单和锁文件。文件名的拼写必须完全如此（全部小写），两者都使用 [NABION](#nabion) 格式。没有依赖的包不需要它们（[0.3.4](003-source-units.md#034-the-manifest-nabinabion-is-optional)）。`nabi.nabion` 以单个 `Package { .. }` 值声明依赖和工具链固定，`nabi.lock` 固定 `nabi update` 解析出的结果。

## environment（环境）

`nabi` 保存依赖集合的单位。每个包含 `nabi.nabion` 的目录都是一个环境，不需要激活步骤。在项目之外，使用 `$NABI_HOME/envs/` 中的具名环境（默认为 `default`）（[9.10.5](../009-tools/910-nabi-frontend.md#9105-environments)）。

这里收集的条目不只是单词的翻译；它们也构成了 Nabi 所依托基础的简明索引：编译流水线、内存管理、类型系统、并发模型等。阅读正文时每当某个概念变得模糊，回到本章就能弄清每个术语在整个规范中所处的位置。
