# 0.2. 编译流水线

本规范把 Nabi 编译器称为 `nabic`。`nabic` 在进程内运行从源代码到链接好的二进制文件的整条流水线：它用自己的代码生成器生成机器码，并用自己的链接器进行链接。

## 0.2.1. 阶段

整体流程如下。

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

> nabi-hir 是高层 IR，它位于类型推断之前，原样保留用户代码的结构。nabi-mir 是中层 IR，其中单态化和运算符脱糖已经完成，所有类型都已确定。
> 代码生成器把 nabi-mir 转换为目标平台的机器码和目标文件；链接器用这些目标文件生成可执行文件。

## 0.2.2. 单向可见性

每个阶段只看紧邻其前的阶段。`nabi-hir` 生成器只参考源代码。`nabi-mir` 生成器只参考 `nabi-hir`。代码生成器只参考 `nabi-mir`。

因此，前面阶段的结果不会再被后面的阶段参考。规范文档遵循同样的规则：编号较小的文件不引用编号较大的文件。

## 0.2.3. 自包含的工具链

`nabic` 不运行任何外部编译器、汇编器或链接器，也不需要在它旁边安装任何工具。唯一可选择启用的例外是 `NABI_CC`，它指定一个 C 编译器驱动程序，由它代替 `nabic` 的链接器完成最终链接（[9.1](../009-tools/901-nabic-cli.md)）。所有中间形式都是 Nabi 自己的：nabi-hir、nabi-mir，以及 nabi-mir 之下代码生成器的低层形式。代码生成器和链接器只为 Nabi 的运行时 ABI（[8](../008-abi/README.md)）及其目标文件格式而编写。

因此，把 Nabi 的高层语义降级为机器操作的责任在于编译器，具体来说在于代码生成阶段（[8.7](../008-abi/807-code-generation.md)）。

## 0.2.4. 代码生成

`nabic` 只有一个代码生成器，用于所有优化级别以及 REPL 单元格。它接收 nabi-mir（在 `-O0` 以上会经过优化），并写出遵循运行时 ABI（[8](../008-abi/README.md)）的 Mach-O、ELF、COFF 或 wasm 目标文件，因此在不同级别下构建的目标文件可以链接在一起。

链接器也是 `nabic` 的一部分：它生成 Mach-O 可执行文件（带 ad-hoc 代码签名，以及存放调试信息的 `.dSYM` 包）、ELF 可执行文件、COFF / PE 可执行文件（使用 `nabic` 生成的导入库，调试信息则写入 PDB），以及 wasm 模块，`wasm32-wasip2` 构建随后再把 wasm 模块转换为 WASI preview 2 组件。交叉编译是内置的：`--target <triple>` 在任何主机上都可用，工具链为其支持的 Linux 目标附带 C 库（musl、glibc 链接桩），所以为 Linux 目标构建 Nabi 代码不需要 sysroot。C 代码由工具链的 C 编译器 `nabicc` 编译，使用同一个代码生成器和链接器；它读取工具链为该目标附带的 C 头文件，或系统的 C 头文件（[9.13](../009-tools/913-c-compiler.md)）。Windows 目标基于 UCRT，链接到由工具链携带的模块定义构建出的导入库，不需要 MSVC 工具或 Windows SDK。Apple 目标需要 macOS SDK（Xcode Command Line Tools），它不可再分发（[9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables)、[0.6](006-platform-targets.md)）。`--emit=mir` 转储优化后的 nabi-mir，`--emit=obj` 转储目标文件（[9.9.1](../009-tools/909-debug-tools.md#991-intermediate-output---emit)）。
