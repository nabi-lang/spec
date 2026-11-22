# 9.6. JIT 引擎

本节定义 REPL 和 Jupyter 使用的 JIT 执行引擎。它是[执行引擎与用户界面](../000-meta/001-execution-modes.md)中所定义的 JIT 引擎的实现。单元会话的规则由 [9.7.0](907-jit-cells.md#970-rules-of-the-cell-session) 定义。

## 9.6.1. 结构：nabic 与 JIT 宿主

会话在一个单独的**宿主进程**中运行，该进程包含运行时，并在整个会话期间存在。宿主由 `nabic` 预先构建，由 `nabic` 的链接器链接；它自己加载单元的目标文件。

```
nabic (front end)                         JIT host (one per session)
  cell → parse → HIR → MIR                   all of libnabir.a
     → code generator                        runtime initialized once
     → machine code + relocations ───────►   load into executable memory, resolve symbols
                                             → run the cell entry function once
```

- 单元是代码生成器（[9.2.11](902-aot-pipeline.md#9211-code-generator)）生成的目标文件，由宿主映射并链接，优化级别由 `:opt` 设置；它的 nabi-mir 不做优化，因为之后的单元可能会重新定义它本来要内联的内容（[9.2.10](902-aot-pipeline.md#9210-optimization-levels)）。REPL 在其规格声明支持 REPL 的目标上运行（[0.6.5](../000-meta/006-platform-targets.md#065-target-specifications)，`repl`）：`x86_64` 和 `aarch64` 的 Linux（glibc）、macOS 和 Windows。
- 宿主把代码放入可执行内存，针对运行时、libc 和之前的单元解析其重定位，并运行该单元的入口函数一次。
- 宿主导出运行时（`libnabir.a`）的每个符号，单元代码在宿主进程中找到运行时和 libc 的符号。

## 9.6.2. 代码加载

单元是一个目标文件。它的代码是带重定位的位置无关代码。宿主为整个会话维护一张符号表；单元加载之后，其定义被加入该表。无法解析的符号会使该单元失败，错误信息会指出该符号（[9.7](907-jit-cells.md)）。

## 9.6.3. 符号与重新定义

一个会话使用一张符号表。所有单元的定义都累积在其中。

- 当某个单元重新定义一个已定义的符号时，宿主把该定义重命名为 `<name>.r<cell>`。之后的单元中的引用链接到最新的名称。
- 重新定义从下一个单元起生效（[9.7.0](907-jit-cells.md#970-rules-of-the-cell-session) 规则 3）。
- 依赖于被重新定义符号的函数由 `nabic` 重新编译。因此，即使是在重新定义之前编译的代码，当之后的单元再次到达它时，也会使用新的定义（调用被重新定义的 `f` 的 `g`）。
- 单元代码和旧定义在会话结束之前不会被释放。

## 9.6.4. 库代码的复用

标准库和依赖包的代码在每个会话中只编译一次。

- `nabic` 记住宿主已经加载的函数及其 MIR。如果某个单元的函数与一个已加载的函数完全相同，则只传递一个引用，而不传递代码。
- 跨单元的名称是固定的，使相同的 MIR 意味着相同的含义。字符串字面量全局变量以其内容的哈希命名，GC 类型 id 以 [ScanTable](../008-abi/802-scan-table.md) 的内容为键，在会话期间固定。
- 直接或间接调用本单元新定义内容（被修改的函数、`static`、绑定槽）的函数会被重新编译。
- 单元的目标文件只包含该单元新增的内容：它新定义的函数，以及对于其代码所读取的之前单元的静态变量，它们的初始化守卫。它只命名其代码引用的符号。

## 9.6.5. 执行流程（一行 REPL / 一个 Jupyter 单元）

```
1. nabic: input → parse → HIR (on error the cell is discarded; session state unchanged)
2. nabic: cell → MIR → code generator → machine code
         (already loaded functions as references, redefinitions under new names)
3. host: load the code in a process forked from the session snapshot and run the cell once
4. success: that process becomes the new snapshot. If there is a value, print the value (`Display`)
   failure (panic, exit, signal, load failure): the previous snapshot takes over (9.7.0 rule 5)
```

第 3 步在 Unix 上使用 fork。没有 fork 的宿主（Windows）在单元执行之前保存会话的内存和线程，在单元失败时恢复它们，可观察的结果与 fork 出的快照相同。

一个单元的延迟约为 0.1 秒，其中大部分花在前端（资料性）。

## 9.6.6. JIT 与 AOT 的区别

| 方面      | JIT（REPL / Jupyter）                         | AOT                               |
|-------------|----------------------------------------------|-----------------------------------|
| 流水线    | 直到 nabi-mir 都相同                           | 相同                              |
| 后端     | Nabi 的代码生成器，加载到宿主中  | Nabi 的代码生成器（高于 `-O0` 时优化），然后是 `nabic` 的链接器 |
| 单位        | 单元（库在每个会话中一次）            | 包                           |
| 结果      | 宿主内存                                  | 可执行文件 / `.a`                 |

两个引擎共享相同的单态化、nabi-mir 和代码生成器，并遵循相同的 ABI，因此它们的语义相同。

## 9.6.7. 下一步：单元执行 / 状态 / 错误

[JIT 单元执行流程与状态管理](907-jit-cells.md)定义每个单元的行为（顶层变量、重新定义、错误）。
