# 0.2. Compilation pipeline

This specification calls the Nabi compiler `nabic`. `nabic` runs the whole pipeline, from source to a linked binary, in-process: it generates machine code with its own code generator and links with its own linker.

## 0.2.1. Stages

The overall flow is as follows.

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

> nabi-hir is the high-level IR; it precedes type inference and preserves the structure of user code as is. nabi-mir is the mid-level IR, in which monomorphization and operator desugaring are complete and all types are fixed.
> The code generator turns nabi-mir into machine code and an object file for the target; the linker makes the executable from the objects.

## 0.2.2. One-way visibility

Each stage looks only at the stage immediately before it. The `nabi-hir` generator refers only to the source. The `nabi-mir` generator refers only to `nabi-hir`. The code generator refers only to `nabi-mir`.

Results of earlier stages are thus never consulted again by later stages. The specification documents follow the same rule: a lower-numbered file does not refer to a higher-numbered file.

## 0.2.3. Self-contained toolchain

`nabic` runs no external compiler, assembler or linker, and no tool has to be installed beside it. The one opt-in exception is `NABI_CC`, which names a C compiler driver to do the final link instead of `nabic`'s linker ([9.1](../009-tools/901-nabic-cli.md)). Every intermediate form is Nabi's own: nabi-hir, nabi-mir and, below nabi-mir, the code generator's low-level form. The code generator and the linker are written for Nabi's runtime ABI ([8](../008-abi/README.md)) and its object formats alone.

The responsibility for lowering Nabi's high-level semantics to machine operations therefore lies in the compiler, specifically in the code generation stage ([8.7](../008-abi/807-code-generation.md)).

## 0.2.4. Code generation

`nabic` has one code generator, used at every optimization level and for REPL cells. It consumes nabi-mir, optimized above `-O0`, and writes Mach-O, ELF, COFF or wasm objects that follow the runtime ABI ([8](../008-abi/README.md)), so objects built at different levels link together.

The linker is part of `nabic` too: Mach-O executables (with an ad-hoc code signature, and a `.dSYM` bundle for debug information), ELF executables, COFF / PE executables (with import libraries `nabic` makes and, for debug information, a PDB), and wasm modules, which `wasm32-wasip2` builds then turn into a WASI preview 2 component. Cross-compilation is built in: `--target <triple>` works from any host, and the toolchain ships the C library (musl, glibc link stubs) for its supported Linux targets, so building Nabi code for a Linux target needs no sysroot. C code is compiled by the toolchain's C compiler `nabicc`, with the same code generator and linker; it reads the C headers the toolchain ships for the target, or the system's ([9.13](../009-tools/913-c-compiler.md)). Windows targets link against import libraries built from module definitions the toolchain carries, over the UCRT, with no MSVC tools or Windows SDK. Apple targets need the macOS SDK (Xcode Command Line Tools), which is not redistributable ([9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables), [0.6](006-platform-targets.md)). `--emit=mir` dumps the optimized nabi-mir and `--emit=obj` the object ([9.9.1](../009-tools/909-debug-tools.md#991-intermediate-output---emit)).
