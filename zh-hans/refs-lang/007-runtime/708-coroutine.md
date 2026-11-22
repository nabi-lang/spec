# 7.8. 协程

[生成器](../004-functions/411-generators.md)、[async](../004-functions/412-async.md) 和[流](../004-functions/413-stream.md)使用同一种协程降级。

## 7.8.1. 状态机降级

Nabi 的协程是无栈的，被降级为由编译器自己构建的状态机：编译器必须知道帧布局，才能构建帧的 ScanTable。没有针对各架构的上下文切换代码；在每个受支持的目标上都相同。

```
yield / async / async yield block
  → lifted into a body function (HIR, like a closure)
  → state machine (MIR): resume function + frame type
  → ordinary code generation
```

`yield fn` / `async fn` 是把其主体包进这样一个块并返回包装类型的 `fn`。

## 7.8.2. 协程帧

```
Frame {
    state  : i32        // suspend point to continue from next
    locals : ...        // only locals live across suspend points
}
```

- 帧是普通的 GC 对象，由分配的快速 / 慢速路径创建（[7.2](702-allocation.md)）。每个类型都有一个帧 [ScanTable](../008-abi/802-scan-table.md)。
- 挂起期间，帧中的 GC 引用由 ScanTable 跟踪；恢复函数运行期间，其局部变量由影子栈跟踪（[7.4.1](704-barriers-and-roots.md#741-根跟踪编译器直接生成的影子栈)）。
- 长期存在的任务的帧会通过 sticky mark-bit 成为老对象（[7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-分代回收)）。
- 调用结果（`GenIter<T>`、`AsyncFuture<T>`、`AsyncStream<T>` 及其 `Try*` 变体）是包装帧的值（[4.1.11](../004-functions/401-function-declaration.md#协程函数的结果类型)）。

## 7.8.3. 生成器

```
1. Call count() → allocate frame, return wrapper.
2. next() → resume function continues from state. At yield, saves the value and state and returns.
3. Next next() → continues from the saved state. When finished, none.
```

由调用方推动进展（`for x in g { .. }`）。提前离开生成器的 `for` 会关闭它：生成器以关闭模式恢复，并在它所等待的 `yield` 处像执行 `return` 一样离开，同时运行其清理（[4.11.10](../004-functions/411-generators.md#41110-生成器的终止)）。

## 7.8.4. Async

```
1. Call fetch() → allocate frame, return Future.
2. poll → resume function proceeds. If an awaited future is Pending, saves state and returns Pending.
3. Wake (parking registry, I/O driver, timer) → the task re-enters the queue and a worker polls it.
```

由调度器和运行时推动进展（[7.6](706-scheduler.md)）。

## 7.8.5. 流

`async yield` 把两者结合起来。在同一个帧中，`yield` 产出一个值，`await` 以 Pending 挂起。

## 7.8.6. 分配消除

每次协程调用都会分配一个帧。对于由调用方运行至完成的协程，没有专门的消除；从不逃逸的帧可以像任何分配一样被移除（[7.2.12](702-allocation.md#7212-分配消除)）。

## 7.8.7. `yield` / `await` 的降级

| 关键字         | 降级                                                                      |
|----------------|---------------------------------------------------------------------------|
| `yield expr;`  | 保存状态 + 返回值（Iterator）/ 返回 `Ready(value)`（Stream）              |
| `await expr;`  | 轮询目标 future。如果为 Pending，保存状态 + 返回 `Pending`                |
| `return expr;` | 结束状态机 + 返回结果                                                     |

## 7.8.8. 运行时符号

`await` 的等待与唤醒由 `core::task` 的驻留注册表处理（`core::task` 通过 `nabi_task_hooks_install` 安装的钩子，[8.5.4](../008-abi/805-extern-symbols.md#854-调度器区域)）。运行时的 `nabi_coro_frame_alloc` / `nabi_coro_park` / `nabi_coro_unpark` 供 C 和底层代码使用；编译器不使用它们（[8.5](../008-abi/805-extern-symbols.md)）。

## 7.8.9. 栈深度

嵌套的 `await` 通过原生调用进行，由外层的恢复函数调用内层 future 的 poll。因此 `await` 递归的深度受原生栈限制，极深的递归会导致栈溢出（[运行时错误](README.md#运行时错误栈溢出)）。如果需要深度递归，就把它拆分成多个任务。

## 7.8.10. 协程帧的生命周期

与其他 GC 对象一样，帧只要可达就一直存在。运行中的任务被任务队列引用，等待中的任务被驻留注册表引用，等待结果的一方被它自己的帧引用。当没有任何东西引用它时，GC 将其回收。没有显式的释放。
