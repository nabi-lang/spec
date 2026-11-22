# 8. 运行时 ABI / 降级概览

本部分定义编译器（`nabic`）与运行时（`libnabir.a`；在 Windows MSVC 目标上为 `nabir.lib`）之间的共同契约。编译器与运行时通过本规范的文本相连。二者共享的代码只有一个没有依赖、双方都会构建的 crate，即 `nabir_abi`（`runtime/nabir_abi`），因此双方使用同一份定义：GC ABI 常量（`nabir_abi::gc`，[8.1](801-object-header-abi.md)）、TLS 槽布局（`nabir_abi::tls`，[8.3](803-tls-slots.md)）、模块描述符与类型描述符（`nabir_abi::module`、`nabir_abi::typedesc`，[8.12](812-module-abi.md)、[8.13](813-type-descriptor.md)），以及常量求值必须与运行时保持一致的格式说明、字符串和 Unicode 例程。

## 目录

- [8.1. 对象头 ABI](801-object-header-abi.md)
- [8.2. ScanTable 序列化格式](802-scan-table.md)
- [8.3. Mutator TLS 槽布局](803-tls-slots.md)
- [8.4. 安全点协议](804-safepoint.md)
- [8.5. Extern "C" 符号](805-extern-symbols.md)
- [8.6. 调用约定](806-calling-convention.md)
- [8.7. 代码生成](807-code-generation.md)
- [8.8. 降级：数据类型与表达式](808-lowering-types.md)
- [8.9. 降级：控制流与 GC 集成](809-lowering-control.md)
- [8.10. 降级：协程与泛型](810-lowering-coroutine-generic.md)
- [8.11. 代码生成器契约](811-backend-contract.md)
- [8.12. 模块描述符 ABI](812-module-abi.md)
- [8.13. 类型描述符 ABI](813-type-descriptor.md)

## 两种契约

### 1. 运行时 ABI

编译器生成的代码与 `libnabir.a` 链接并一起运行时所用的接口。对象的内存布局有明确规定，使 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 能准确解释编译器创建的对象。[Extern "C" 符号](../000-meta/007-glossary.md#extern-c-符号)是编译器可以调用的运行时函数；每个符号的线程安全类别（`thread-safe` / `caller-thread-only` / `process-init` 等）在 [8.5.0 线程安全类别](805-extern-symbols.md#850-线程安全类别)中定义。TLS 槽是编译器生成的内联代码读写的线程局部位置。调用约定规定函数调用中寄存器与栈的使用方式。

### 2. 降级

在[编译流水线](../000-meta/002-pipeline.md)的代码生成阶段应用的规则，负责把 Nabi 的高层语义翻译为机器码：每个值的表示，以及 `tagged union`、`any` 槽、模式匹配、try/catch、协程和泛型如何变为加载、存储、分支和调用。

## 为什么本规范文本具有权威性

除了那些被包含的源文件，编译器与运行时不共享任何代码或头文件。二者遵循同一 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 的保证来自本规范文本，以及集成测试中的端到端验证。

这一模式与 Swift ABI Stability 的做法相同：规范具有权威性，实现遵循规范。

## 稳定性

本部分的契约（包括符号修饰，[9.5](../009-tools/905-symbol-mangling.md)）是工具链版本的一部分。1.0 之前，ABI 可以在每个 0.x.0 版本中变化；0.x.y 版本不会破坏它。从 1.0 起，次版本 / 修订版本只可以增加内容，不得破坏兼容性；主版本可以破坏兼容性，但方式必须能被包管理器检测到。

预构建包的 NABIHIR 头记录了构建它的 `nabic`（[9.4](../009-tools/904-nabihir-bytecode.md)）。使用由另一个 `nabic` 构建的包是错误，该错误会要求重新构建。
