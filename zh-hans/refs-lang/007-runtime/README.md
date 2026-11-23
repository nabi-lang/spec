# 7. 运行时概述

本部分定义 Nabi 运行时系统（[GC](../000-meta/007-glossary.md#gc-garbage-collector)、任务调度器、协程）的实现规范。该系统在[执行引擎与用户界面](../000-meta/001-execution-modes.md)的两种引擎（[JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)）下行为完全相同。

## 目录

本章各节如下：

- [7.1. 对象布局与内存空间](701-object-layout.md)
- [7.2. 分配](702-allocation.md)
- [7.3. GC 算法](703-gc-algorithm.md)
- [7.4. 屏障与根跟踪](704-barriers-and-roots.md)
- [7.5. 并发标记](705-concurrent-collection.md)
- [7.6. 调度器](706-scheduler.md)
- [7.7. I/O 驱动与定时器](707-io-and-timer.md)
- [7.8. 协程](708-coroutine.md)
- [7.9. 内存模型](709-memory-model.md)

## 关键决策

运行时核心组件的决策概括如下：

```
GC          in-house Immix + Sticky Mark-Bit generations
Scheduler   M:N work-stealing
Coroutine   compiler-generated state machine (stackless)
```

三者均为自研实现，不依赖任何外部库。

## 规范地位

GC 各节（7.1 至 7.5）是资料性的：它们描述参考运行时目前如何分配和回收，后续的运行时可以采用其他算法、空间或屏障。规范性的是在 [8.1](../008-abi/801-object-header-abi.md) 中作为 ABI 固定的对象布局：每个 GC 对象的载荷之前有一个 16 字节的头。库代码依赖于它：`Pinned::as_data_ptr` 和 `as_value_ptr`（[标准库 1.6.6](../../refs-standard-libraries/100-bare/106-gc-pin.md#166-提取方法)）把对象地址加 16 算作载荷地址。[7.9](709-memory-model.md) 的内存模型同样是规范性的：程序和每个后端都依赖它。

## 零依赖原则

运行时不依赖外部库。具体来说，它不使用：

- 外部 GC 库（Boehm GC、MMTk 等）；
- 外部异步运行时；
- 外部协程库。

平台系统接口（`libc`、操作系统头文件等）不算作依赖，因为它们是操作系统接口不可或缺的一部分。

## M:N 模型

调度器采用 M:N 混合模型。

```
OS Thread (worker) x N  ──── nabi task / coroutine x M
                                   (typically M >> N)
```

N 默认为 CPU 核心数，可以通过运行时配置调整（`NABIR_WORKERS`，[7.6.8](706-scheduler.md#768-每核一线程选项)）。每个工作线程既是调度器的工作线程，也是 mutator；全局停顿（stop-the-world）回收在发起它的线程上运行（[7.6.10](706-scheduler.md#7610-集成的-gc-工作角色)）。I/O 驱动是一个单独的 reactor 线程（[7.7](707-io-and-timer.md)），并发标记在单独的标记线程上运行（[7.5](705-concurrent-collection.md)）。

## 单一 .a 输出

运行时以单个静态库 `libnabir.a` 分发。构建用户包时链接 `libnabir.a`，生成的可执行文件不再有其他外部依赖。

## 运行时错误：栈溢出

当线程的栈溢出时，运行时会（在可能的情况下）刷新 stdout，向 stderr 打印以下内容，然后中止（SIGABRT）。与 panic 不同，这里没有栈展开，`finally` 也不会运行。

```
thread '<name>' has overflowed its stack
fatal runtime error: stack overflow, aborting
```

`<name>` 是线程的名称：即用 `thread::Builder::name`（[标准库 3.4](../../refs-standard-libraries/300-std/304-thread.md)）指定的名称，它同时被设为操作系统线程名。主线程为 `main`，未命名的线程为 `<unnamed>`。这适用于主线程、`thread::spawn` 线程以及执行器的工作线程（包括很深的 `await` 链）。检测机制（保护区间和 `sigaltstack`）与 TLS 槽见 [8.5.7](../008-abi/805-extern-symbols.md#栈溢出)。在 wasm 上，模块无法捕获该 trap，由宿主报告（随附的 JS 宿主会打印上述消息）。

## 与编译器的接口

运行时通过固定的 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 与编译器生成的代码通信。该接口由三部分组成：

- 对象布局（16 字节头 + 载荷）。
- `nabi_gc_alloc_slow`、`nabi_runtime_yield`、`nabi_thread_spawn` 等 `extern "C"` 函数。
- 协程是编译器生成的状态机（[7.8](708-coroutine.md)）。对于 GC 根，编译器直接生成影子栈帧（[7.4.1](704-barriers-and-roots.md#741-根跟踪编译器直接生成的影子栈)）。

该 ABI 在 [8. 运行时 ABI / 降级](../008-abi/README.md)中详细定义；本章只涉及内部行为。
