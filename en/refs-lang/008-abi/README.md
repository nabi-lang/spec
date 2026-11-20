# 8. Runtime ABI / lowering overview

This block defines the shared contract between the compiler (`nabic`) and the runtime (`libnabir.a`; `nabir.lib` on Windows MSVC targets). The compiler and the runtime are connected through the text of this specification. The code they share is one crate without dependencies that both build, `nabir_abi` (`runtime/nabir_abi`), so both sides use one definition: the GC ABI constants (`nabir_abi::gc`, [8.1](801-object-header-abi.md)), the TLS slot layout (`nabir_abi::tls`, [8.3](803-tls-slots.md)), the module and type descriptors (`nabir_abi::module`, `nabir_abi::typedesc`, [8.12](812-module-abi.md), [8.13](813-type-descriptor.md)), and the format-spec, string and Unicode routines that constant evaluation must agree on with the runtime.

## Contents

- [8.1. Object header ABI](801-object-header-abi.md)
- [8.2. ScanTable serialization format](802-scan-table.md)
- [8.3. Mutator TLS slot layout](803-tls-slots.md)
- [8.4. Safepoint protocol](804-safepoint.md)
- [8.5. Extern "C" symbols](805-extern-symbols.md)
- [8.6. Calling convention](806-calling-convention.md)
- [8.7. Code generation](807-code-generation.md)
- [8.8. Lowering: data types and expressions](808-lowering-types.md)
- [8.9. Lowering: control flow and GC integration](809-lowering-control.md)
- [8.10. Lowering: coroutines and generics](810-lowering-coroutine-generic.md)
- [8.11. Code generator contract](811-backend-contract.md)
- [8.12. Module descriptor ABI](812-module-abi.md)
- [8.13. Type descriptor ABI](813-type-descriptor.md)

## Two kinds of contract

### 1. Runtime ABI

The interface used when compiler-generated code is linked and run together with `libnabir.a`. The memory layout of objects is specified so that the [GC](../000-meta/007-glossary.md#gc-garbage-collector) can interpret compiler-created objects exactly. [Extern "C" symbols](../000-meta/007-glossary.md#extern-c-symbol) are runtime functions the compiler may call; the thread-safety class of each symbol (`thread-safe` / `caller-thread-only` / `process-init`, etc.) is defined in [8.5.0 Thread-safety classes](805-extern-symbols.md#850-thread-safety-classes). TLS slots are thread-local locations read and written by inline code the compiler emits. The calling convention specifies register and stack usage in function calls.

### 2. Lowering

The rules, applied in the code generation stage of the [compilation pipeline](../000-meta/002-pipeline.md), that translate Nabi's high-level semantics into machine code: the representation of every value and how `tagged union`, `any` slots, pattern matching, try/catch, coroutines and generics become loads, stores, branches and calls.

## Why this specification text is authoritative

Apart from those included source files, the compiler and the runtime share no code or header files. The guarantee that both follow the same [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) comes from this specification text and from end-to-end verification in integration tests.

This model follows the same approach as Swift ABI Stability: the specification is authoritative and the implementation follows it.

## Stability

The contract in this block, symbol mangling ([9.5](../009-tools/905-symbol-mangling.md)) included, is part of the toolchain version. Before 1.0 the ABI may change in every 0.x.0 release; 0.x.y releases do not break it. From 1.0, minor / patch releases may only add and must not break compatibility, and major releases may break it in a way the package manager can detect.

The NABIHIR header of a prebuilt package records the `nabic` that built it ([9.4](../009-tools/904-nabihir-bytecode.md)). Using a package built by a different `nabic` is an error that asks for a rebuild.
