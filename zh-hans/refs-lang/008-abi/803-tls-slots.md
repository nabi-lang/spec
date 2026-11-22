# 8.3. 修改器 TLS 槽布局

每个 OS 线程（= 调度器工作线程 = 修改器（mutator））都在线程局部存储（TLS）中的固定位置保存运行时信息。编译器生成的代码与运行时读写同一组 TLS 槽，因此本文档定义每个槽的偏移和类型。

## 8.3.1. TLS 槽结构

TLS 槽是由运行时定义并导出的一个 `thread_local` 块 `nabi_tls`；每个槽位于其中的固定偏移处。编译后的代码按相对于块地址的偏移访问各个槽，而 Nabi 函数从其调用者处接收这个块地址（8.3.8）。

`p` 是以字节计的指针宽度。

| 偏移     | 槽                          | 类型 | 含义                                    |
|----------|-----------------------------|------|-----------------------------------------|
| 0        | `nabi_tls_shadow_stack_top`  | ptr  | 影子栈链的头                            |
| p        | `nabi_tls_tlab_cursor`       | ptr  | TLAB 中下一次分配的位置                 |
| 2p       | `nabi_tls_tlab_limit`        | ptr  | TLAB 的末尾                             |
| 3p       | `nabi_tls_wb_log_cursor`     | ptr  | 写屏障日志中的位置                      |
| 4p       | `nabi_tls_wb_log_limit`      | ptr  | 写屏障日志的末尾                        |
| 5p       | `nabi_tls_satb_cursor`       | ptr  | SATB 缓冲区中的位置（并发标记期间）     |
| 6p       | `nabi_tls_satb_limit`        | ptr  | SATB 缓冲区的末尾                       |
| 7p       | `nabi_tls_task_worker`       | ptr  | 本线程的执行器工作线程（GC 对象，没有则为 0） |
| 8p       | `nabi_tls_stack_guard`       | [usize; 2] | `[lo, hi)`：在此范围内的故障是本线程的栈溢出（`[0, 0)` 表示未知，[8.5.7](805-extern-symbols.md#stack-overflow)） |
| 10p      | `nabi_tls_worker_id`         | u64  | 工作线程 id                             |
| 10p+8    | `nabi_tls_yield_flag`        | u8   | 是否在安全点 `yield`                    |
| 10p+9    | `nabi_tls_mutator_state`     | u8   | 修改器状态                              |

**ABI 契约**是块符号 `nabi_tls` 以及每个槽的偏移、类型和含义。槽名仅供参考，它们不是符号。块按 8 对齐，至少 10p+10 字节；最后一个 ABI 槽之后可以跟随由实现定义的槽。

**由实现定义的槽**：运行时可以在 ABI 槽之后或其他位置保存更多线程局部状态，用于调度、统计、性能分析、调试等。编译器和外部包不得引用这些状态。

> [!NOTE]
> Nabi 的运行时把下列槽放在 ABI 部分之后。整个块为 16p+16 字节。
>
> - 10p+10：`no_safepoint`（`u8`）。调试版 nabic 用 `nabi_nosp_enter` / `nabi_nosp_leave` 包围的 `#[no_safepoint]` 调用的深度（[8.4.10](804-safepoint.md#8410-native-calls)）。
> - 10p+11：`attached`（`u8`）。线程加入运行时的方式：0 表示尚未加入或已不再加入，1 表示通过 `nabi_thread_init` 或是运行时启动的线程，2 表示在 `extern "C"` 定义的 C 入口处接入（8.3.8）。
> - 10p+16：`task_lane`
> - 11p+16：`static_inits`
> - 12p+16：`held_locks`。本线程持有的锁。在会话宿主中，已经结束的线程在阻塞时持有的锁会被释放。
> - 13p+16：`thread_name`（`{ptr, len}`）。报告中使用的线程名；未命名时为 null。
> - 15p+16：`throw_site`。错误正在传播中的那个 `throw` 的位置（以 NUL 结尾）的地址，由编译后的代码在 `throw` 处存入；未知时为 null 或空。

## 8.3.2. 访问机制

每种 OS 和架构都有各自的 TLS 访问机制。

| 平台           | 机制                                  |
|----------------|---------------------------------------|
| Linux x86_64   | 线程指针 `%fs:0` + 偏移               |
| Linux aarch64  | 线程指针 `tpidr_el0` + 偏移           |
| macOS aarch64  | TLV 描述符，通过其 getter 调用        |
| Windows x86_64 | TEB（Thread Environment Block）中 `_tls_index` 处的 TLS 数组 + 偏移 |
| `wasm32-wasip1-threads` | `__tls_base` 全局变量 + 偏移 |
| 其他 WASM（无线程） | 块是一个普通全局变量             |

编译器按目标和 TLS 模型生成相应的指令（8.3.8）。

## 8.3.3. 各槽的含义

### `nabi_tls_yield_flag`（u8）

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 为安全点设置的标志；修改器在轮询时检查它。位 0 是运行时的停止请求，由运行时置位和清除；其他位属于嵌入宿主（例如 REPL 的中断），运行时不会改动。只要标志非零，轮询就调用 `nabi_runtime_yield`。

详细机制定义在[屏障与根](../007-runtime/704-barriers-and-roots.md)的安全点轮询部分。

### `nabi_tls_tlab_cursor`（ptr）

指向[分配](../007-runtime/702-allocation.md)中 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 内下一次分配发生的位置。

```
allocate(size):
    tmp = tlab_cursor
    new_cursor = tmp + size
    if new_cursor > tlab_limit:
        slow path
    tlab_cursor = new_cursor
    return tmp
```

这四条指令构成 TLAB 快速路径，由编译器以内联方式生成。

### `nabi_tls_tlab_limit`（ptr）

指向 TLAB 的末尾。游标越过该值时进入慢速路径。

### `nabi_tls_shadow_stack_top`（ptr）

影子栈链的头。详见 8.3.8。

### `nabi_tls_satb_cursor` / `nabi_tls_satb_limit`（ptr）

并发标记期间由写屏障慢速路径记录的 SATB 缓冲区（[7.5](../007-runtime/705-concurrent-collection.md)）。写满时交给回收器。标记之外不使用。

### `nabi_tls_wb_log_cursor` / `nabi_tls_wb_log_limit`（ptr）

[屏障与根追踪](../007-runtime/704-barriers-and-roots.md)中写屏障日志缓冲区的游标和界限。

```
write_barrier(p, q):
    if !needs_log: return
    if wb_log_cursor >= wb_log_limit: flush
    *wb_log_cursor++ = p
```

大多数操作是非原子的，并且是线程局部的。

这两个槽指向每线程一个、大小为 512 字的日志块（64 位目标上为 4 KiB，32 位上为 2 KiB）。写满的块被推入一个由回收器取空的无锁栈（[7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-object-remembering)）。

### `nabi_tls_mutator_state`（u8）

修改器的当前状态。

| 值    | 含义                          |
|-------|-------------------------------|
| `0`   | RUNNING。正常执行             |
| `1`   | GC_COOPERATING。正在与 GC 协作 |
| `2`   | PARKED。处于睡眠 / 等待       |
| `3` ~ | 保留                          |

该值用于 GC 周期同步和安全点协作。

### `nabi_tls_worker_id`（u64）

线程的工作线程 id（在调度器内部使用），用于[工作窃取](../000-meta/007-glossary.md#work-stealing)、统计等。

## 8.3.4. 槽的宽度与对齐

每个槽的宽度与其类型一致，并位于按该类型对齐的偏移处。

## 8.3.5. 32 位与 64 位

指针槽为 p 字节（64 位上为 8，32 位上为 4），因此 32 位与 64 位之间的偏移按表中给出的方式不同。

## 8.3.6. TLS 槽的初始化

工作线程启动时，各槽按如下方式初始化。

```
nabi_thread_init():
  nabi_tls_yield_flag = 0
  nabi_tls_tlab_cursor = NULL          // filled by the slow path on first allocation
  nabi_tls_tlab_limit = NULL
  nabi_tls_shadow_stack_top = NULL     // empty shadow stack chain (8.3.8)
  nabi_tls_wb_log_cursor = wb_log_base
  nabi_tls_wb_log_limit = wb_log_base + LOG_SIZE
  nabi_tls_mutator_state = RUNNING
  nabi_tls_worker_id = (allocated)
  nabi_tls_task_worker = 0
  nabi_tls_stack_guard = [guard_lo, guard_hi)   // guard range below the stack
```

调度器在工作线程启动时调用 `nabi_thread_init`；该函数属于[外部符号](805-extern-symbols.md)。

## 8.3.7. TLS 槽的拆除

工作线程退出时，经过以下步骤。

```
nabi_thread_shutdown():
  // log flush
  // shadow stack walk (and a final mark, if possible)
  // return worker_id
```

## 8.3.8. 编译器生成的形式

该块是由运行时定义并导出的 `thread_local` 全局变量。编译器以外部声明引用它，并按相对于块地址的偏移读取各槽。Nabi 函数从调用者处以隐藏的最后一个实参接收该地址（8.6.6），因此 Nabi 函数之间的调用不读取 TLS。从 Nabi 之外进入的代码自己获取一次该地址并继续传递：C 的 `main`、`extern "C"` 定义的 C 入口、`#[export_name]` Nabi 函数、`any` 分派器、惰性 `static` 的初始化器，以及函数值所经过的 thunk（8.6.11）。

`extern "C"` 定义的 C 入口可能运行在由 C 代码启动、运行时并不知道的线程上。在这样的线程上，它先接入该线程：按 `nabi_thread_init` 的方式初始化各槽（8.3.6），并把该线程注册为修改器，然后再运行函数体。在两次这样的调用之间回到 C 时，该线程算作 `PARKED`（[8.4.10](804-safepoint.md#8410-native-calls)），因此回收不会等待它；线程结束时，它像 `nabi_thread_shutdown` 那样离开注册表（8.3.7）。无法以这种方式接入线程的目标（WebAssembly）则改为中止，并输出 `nabi: callback on a thread nabi does not know; call nabi_thread_init() first`。

```
; nabi_tls: defined by the runtime; compiler output refers to it as an undefined symbol
t = <address of nabi_tls>         ; once, where code enters nabi
                                  ; (a nabi function: its last argument)

; safepoint poll (yield_flag at 10p+8)
flag = load.u8 [t + 88]           ; 10p+8 with p = 8
if flag != 0 then bb_slow else bb_cont
```

在 ELF 上，`nabi_tls` 使用 initial-exec TLS 模型（编译器按此模型生成访问代码，不依赖链接器松弛）。运行时总是静态链接进可执行文件，因此这一模型已足够。Mach-O 使用平台的 TLV 描述符方案，Windows 使用线程在 `_tls_index` 处的 TLS 数组。无法选择 TLS 模型的实现可以改用 general-dynamic。

### 影子栈头

`nabi_tls` 偏移 0 处的槽（`nabi_tls_shadow_stack_top`）是[屏障与根追踪](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)中[影子栈](../000-meta/007-glossary.md#shadow-stack)链的头，即最内层 `ShadowFrame` 的地址（没有则为 null）。它是唯一的链头。编译器生成的代码在函数每条路径的第一个安全点之前链接帧，并在链接过帧的路径的每个 `return` 之前立即解除链接（8.4.7）。

## 8.3.9. 稳定性

8.3.1 表中的偏移、类型和含义属于 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface)；其变更遵循[概述](README.md#stability)中的稳定性策略。改变偏移，或改变已有槽的类型或含义，会破坏 ABI（1.0 之前为 0.x.0 版本，1.0 起为主版本）。新槽使用现有槽之后的偏移，属于兼容变更。

由实现定义的槽不属于 ABI，可以在任何版本中变更。
