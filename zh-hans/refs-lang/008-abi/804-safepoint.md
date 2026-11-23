# 8.4. 安全点协议

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 要执行临时的 STW（Stop-The-World）工作，每个 mutator 线程都必须已到达安全点。本文档定义用于这种同步的 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 协议。

## 8.4.1. `yield` 标志检查：快速路径

mutator 定期检查 [TLS 槽](803-tls-slots.md) `nabi_tls_yield_flag`（u8）。检查过程如下。

```
poll:
    flag = TLS[yield_flag]
    if flag != 0:
        call nabi_runtime_yield()    // slow path
    // proceed normally
```

生成的代码如下：

```
t = <nabi_tls address>            ; the hidden last argument (8.3.8)
flag = load.u8 [t + 88]           ; yield_flag at 10p+8, p = 8
if flag != 0 then bb_slow else bb_cont

bb_slow:                          ; placed out of line
    call nabi_runtime_yield()
    goto bb_cont

bb_cont:
  ; ... continue
```

这是两条指令的快速路径加一个分支，而该分支几乎总是求值为 `false`。

## 8.4.2. 生成位置

编译器在以下位置生成安全点轮询。

| 位置                  | 理由                      |
|-----------------------|---------------------------|
| 函数入口（或每条路径的第一个安全点之前） | 调用之后可以立即与 GC 协作 |
| 循环回边              | 防止无限循环阻塞 GC       |
| 函数调用之后（可选）  | 如果调用耗时较长，可以立即协作 |

具体的生成策略由编译器决定；本规范只保证以下最低频率。

- 每条循环回边都有轮询，例外是迭代次数为不超过 64 的编译期常量、且循环体不会到达安全点的循环；每个调用环都包含一个轮询。
- 不可能到达安全点的函数（除上述有界循环外没有循环，没有分配、间接调用、惰性 `static`，也没有调用可能到达安全点的函数）不需要轮询。

二者合起来，限定了从 yield 请求到下一个安全点之间的时间。

## 8.4.3. `nabi_runtime_yield`：慢速路径

慢速路径的签名如下：

```
extern "C" {
    fn nabi_runtime_yield();
}
```

mutator 调用该函数与 GC 协作。其效果如下：

1. 把该 mutator 的 `mutator_state`（[mutator TLS 槽布局](803-tls-slots.md)）设为 `GC_COOPERATING`。
2. 暴露 GC 所需的根信息，例如[影子栈](../000-meta/007-glossary.md#shadow-stack)遍历。
3. 等待 GC 完成。
4. GC 完成后，把 `mutator_state` 设回 `RUNNING`。

`yield_flag` 在 `nabi_runtime_yield` 内部自动清除，或由 GC 清除。

## 8.4.4. 设置 / 清除 `yield` 标志

GC 需要 STW 停顿时：

```
gc_initiate_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 1

  // wait until every mutator yields
  for each mutator thread:
    wait until mutator_state == GC_COOPERATING
```

GC 结束 STW 时：

```
gc_finish_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 0
  notify all (resume)
```

## 8.4.5. 快速路径的开销

mutator 承担的开销是对 `yield` 标志的一次读取和一个分支。

几乎在所有情况下该标志都为 0，分支不会跳转，现代 CPU 的分支预测器处理它的开销几乎为零。

以这一开销为代价，每个线程都获得了一种确定性的协作式同步机制。

## 8.4.6. STW 阶段

STW 回收（minor 回收、带疏散的完整回收、`nabi_gc_collect`）是单次停顿。发起回收的线程停止所有其他 mutator（`GC_COOPERATING`，若处于阻塞的运行时调用中则为 `PARKED`），完成回收后再让它们恢复。

带并发标记的完整回收有两次停顿：初始停顿和最终停顿（[7.5.2](../007-runtime/705-concurrent-collection.md#752-阶段)）。在两次停顿之间，一个标记线程与 mutator 同时运行。

## 8.4.7. 安全点不变式

到达安全点时，mutator 的每个存活 GC 对象变量都必须已注册在影子栈中。

编译器通过在函数每条路径的第一个安全点之前链接[屏障与根跟踪](../007-runtime/704-barriers-and-roots.md)中的影子栈帧来保证这一点：在入口处链接，或在到达安全点的路径上稍后链接。不会到达安全点的路径不必链接，帧也只在链接过的地方于返回时解除链接。同样，只由 Nabi 直接调用进入的函数可以把入口轮询移到链接之后：每个调用环仍然包含一个轮询，因为闭合该环的调用就是一个安全点。从 C 调用或经由函数值调用的函数在入口处轮询。在安全点持有 GC 引用的局部变量被注册在帧中，且每次写入时都保持所注册的字为最新。只有回边轮询才看得到的局部变量，可以改为由轮询的慢速路径仅为该运行时调用而注册（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-根跟踪编译器直接生成的影子栈) 规则 4），因此，安全点只有其轮询的循环在运行期间既不链接帧，也不写根字。编译器还可以（在寄存器中）保留私有副本，因为回收从不改写根字：从根直接到达的对象不会被移动（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-根跟踪编译器直接生成的影子栈)）。热重载代码是例外（8.4.13）。

## 8.4.8. 循环回边轮询

如果一个长循环不与 GC 协作，STW 就会永远等待。因此编译器在每条循环回边上生成轮询（8.4.2 中的有界循环除外）。

```
loop:
    body
    poll                              // back-edge poll
    branch loop
```

这一轮询的开销同样是两条指令和一个分支，与循环体相比可以忽略。

轮询留在回边上。其慢速路径（`nabi_runtime_yield` 调用）可以分支到放在循环头之前的重入块，而不必返回到该边。

编译器检测每一个自然循环，并为除 8.4.2 有界循环之外的每个循环生成轮询。

## 8.4.9. 内联函数与轮询

小函数被内联时，其入口轮询可能随之消失。但内联结果被吸收进调用者的函数体，因此调用者的入口轮询已经足够。

轮询在优化之后放置在实际编译的代码上：内联函数体中的循环与其他循环一样，在调用者中于其回边处轮询。

## 8.4.10. 原生调用

处于 C 调用或阻塞的运行时调用中的线程无法到达安全点，因此算作 `PARKED`：GC 视其为已在协作。

- 编译器包围每个 `extern "C"` 调用：调用期间为 `PARKED`，调用之后为 `RUNNING`，每次存储之后都检查 yield 标志。调用者的实参根保留在其影子栈帧中。快速路径是内联的；`nabi_enter_native` / `nabi_leave_native` 是慢速路径（[8.5.2](805-extern-symbols.md#852-运行时区域)）。对 `#[leaf]` 声明的调用不加包围：线程保持 `RUNNING`，并推迟所有回收直到调用返回（[标准库 1.7.1](../../refs-standard-libraries/100-bare/107-ffi.md#leaf短调用)）。
- 每个 `extern "C" fn` 定义都以 `nabi_native_reenter` 开始、以 `nabi_native_resume` 结束，因此 C 回调作为 mutator 运行。
- 运行时以同样方式包围自己的阻塞调用（I/O 等待、`nabi_sync_wait`、join、sleep），以及不触及任何 GC 对象（只触及调用者帧保持存活的缓冲区）、且不存储引用的长时间计算：`int` 内核（乘法、除法、gcd；以及从 64 Ki 个 limb 起的加法、减法、移位和单 limb 除法），以及从 64 KiB 起的 `str` 搜索与计数、UTF-8 检查和字节复制（[8.5.2](805-extern-symbols.md#不是安全点的函数)）。它们的 `extern "nabi"` 声明不是 `#[no_safepoint]`（[5.9.2](../005-modules/509-extern-block.md#592-abi-说明符)），因此对它们的调用是安全点，缓冲区也已登记为根。
- 对 `#[no_safepoint]` 运行时函数的调用不加包围。其他所有线程的回收都要等到它返回，因此这样的函数只做有界的工作量；较长的输入交给它的 parked 变体处理。调试版 nabic 用 `nabi_nosp_enter()` / `nabi_nosp_leave()` 包裹它（[8.5.2](805-extern-symbols.md#852-运行时区域)），这两个函数升高和降低线程的 `no_safepoint` 深度（[8.3.1](803-tls-slots.md#831-tls-槽结构)）；如果线程在深度非零时会因回收而停下，调试版运行时以致命错误失败。发布版 nabic 两个调用都不生成。
- 通过 `nabi_*` 符号（[8.5](805-extern-symbols.md)，运行时的保留前缀）的 `extern "C"` 声明进行的调用不加包围：该运行时函数作为 mutator 运行。

```
call_extern_c:
    mutator_state = PARKED      // seq_cst
    if yield_flag:
        nabi_enter_native()      // wake a collection waiting for this thread
    c_function()
    mutator_state = RUNNING     // seq_cst
    if yield_flag:
        nabi_leave_native()      // wait out a running collection
```

## 8.4.11. 用户代码的直接调用

用户代码也可以（很少见地）直接调用 `nabi_runtime_yield()`。

```nabi
extern "C" {
    fn nabi_runtime_yield();
}

fn long_loop() {
    for i in 0..1_000_000 {
        ...
        if i % 1000 == 0 { unsafe { nabi_runtime_yield(); } }
    }
}
```

`nabi_runtime_yield` 是 `nabi_*` 符号，因此该调用不加包围（8.4.10），线程作为正在运行的 mutator 让出。编译器已经在循环回边上生成轮询，所以用户几乎不需要直接调用它。这种写法只用于内联汇编内部等特殊情形。

## 8.4.12. 稳定性

ABI 是 `nabi_tls` 偏移 10p+8 处的槽（`nabi_tls_yield_flag`）及其类型（u8）（[8.3.1](803-tls-slots.md#831-tls-槽结构)）。

`nabi_runtime_yield` 遵循无参数、无返回值的简单调用约定。

## 8.4.13. 热重载代码

用 `--hot`（[9.1](../009-tools/901-nabic-cli.md)）构建的代码运行在一个可能改写根字的热运行时之下：状态迁移会把已变更类型的对象复制为新对象，并更新所有指向它们的引用，包括根。因此这类代码不会跨安全点保留 GC 引用的私有副本：每个跨安全点存活的引用，在安全点之后都从其根字重新读取。

- 热构建是 `-O0` 构建。其中每个局部变量都位于其帧槽中，而该帧槽正是影子栈帧注册的字，因此每次使用都读取根字。
- 对热包中函数的调用，无论其当前函数体做什么，都算作安全点：它所经由的槽可能到达一个会停下的后续版本。调用者把跨该调用存活的值作为根。
- 持有热包中某个类型的值的函数，以描述符形式链接其影子栈帧（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-根跟踪编译器直接生成的影子栈)），即使它不注册任何根也是如此；其描述符之前的字存放其帧记录（它所持有的类型）的地址，它的某个根字也存放该地址。热运行时在那里找到旧代码中已停下的帧。

不用 `--hot` 构建的代码（宿主程序、C）的帧仍遵循 8.4.7 的规则；迁移不会改写这类帧的根。
