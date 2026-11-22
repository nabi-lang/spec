# 0.6. 平台目标

Nabi 可以从任何主机为下列内置目标，以及具有相同架构和目标文件格式的自定义目标（0.6.5）进行构建（`--target`，[0.2.4](002-pipeline.md#024-code-generation)）。每个目标都处于三种状态之一：已支持（0.6.1）、进行中（0.6.2）或不支持（0.6.3）。

## 0.6.1. 已支持

已支持的目标是内置的：`nabic --print target-list` 会列出它们，`--target` 接受其中每一个。

### 64 位桌面 / 服务器

这一组涵盖 Linux、macOS 和 Windows 的主要架构，包括用于静态二进制文件的 musl 变体，以及 FreeBSD。

| 目标                         | 说明                                       |
|------------------------------|--------------------------------------------|
| `x86_64-unknown-linux-gnu`   | 主要的 Linux 服务器 / 桌面                 |
| `aarch64-apple-darwin`       | macOS Apple Silicon                        |
| `aarch64-unknown-linux-gnu`  | ARM 服务器（AWS Graviton、Raspberry Pi 5 等） |
| `x86_64-pc-windows-msvc`     | Windows 桌面                               |
| `aarch64-pc-windows-msvc`    | Windows ARM                                |
| `x86_64-unknown-linux-musl`  | Alpine / `static` 二进制文件               |
| `aarch64-unknown-linux-musl` | 同上，ARM                                  |
| `x86_64-unknown-freebsd`     | FreeBSD 14                                 |

### WebAssembly

支持三个 WebAssembly 目标，涵盖基于浏览器的执行和服务器端执行。WASI sockets（仅限 preview 2 组件）和 WASI threads（一个 preview 1 核心模块提案）目前无法在同一个模块中结合使用，所以服务器端 WASM 有两个目标。

| 目标                     | 说明                             |
|--------------------------|----------------------------------|
| `wasm32-unknown-unknown` | 浏览器环境；`target_os = "unknown"`（[6.3.2](../006-meta-system/603-cfg.md)） |
| `wasm32-wasip2` | 服务器端 WASM：一个 WASI preview 2 组件。使用 `wasi:sockets` 联网；没有线程（`thread::spawn` 触发 panic；`thread::Builder::spawn` 抛出 `Unsupported`） |
| `wasm32-wasip1-threads` | 带线程的服务器端 WASM：一个使用 wasi-threads（共享内存）的 WASI preview 1 核心模块。只能在主机传入的监听器上联网；connect / bind / DNS 抛出 `Unsupported` |

> [!NOTE]
> wasmtime 在 v46 之后移除了 wasi-threads（WAMR 的 iwasm 和 wasmer 仍然实现了它），所以 `wasm32-wasip1-threads` 将来会依赖 WASI shared-everything-threads。

这些名称与生态系统的其他部分保持一致，在那里 `wasm32-wasi` 指 WASI preview 1；Nabi 没有名为 `wasm32-wasi`、`wasm32-wasi-threads` 或 `riscv32imac-unknown-none` 的目标，以其中之一作为 `--target` 是一个未知目标，错误会指出最接近的内置目标（[9.1.2](../009-tools/901-nabic-cli.md#912-core-subcommands)）。

### 32 位

为了仍在活跃使用的嵌入式和遗留环境，支持以下四个 32 位目标。

| 目标                            | 说明                                 |
|---------------------------------|--------------------------------------|
| `i686-unknown-linux-gnu`        | 32 位 x86 Linux                      |
| `i686-pc-windows-msvc`          | 32 位 Windows                        |
| `armv7-unknown-linux-gnueabihf` | 32 位 ARM Linux，硬浮点（运行 32 位操作系统的 Raspberry Pi 2 到 4 等） |
| `armv7-unknown-linux-musleabihf` | 同上，musl：`static` 二进制文件 |

### 移动平台

| 目标                    | 说明                                      |
|-------------------------|-------------------------------------------|
| `aarch64-apple-ios-sim` | Apple Silicon Mac 上的 iOS 模拟器         |

`aarch64-apple-ios-sim`（Apple Silicon Mac 上的 iOS 模拟器）为 iOS 13.4 及更高版本构建，iOS 13.4 是第一个其加载器能读取 chained fixups 的 iOS 版本，而 chained fixups 正是 Apple 链接器写出的格式：`IPHONEOS_DEPLOYMENT_TARGET`，或 nabi.nabion 中的 `deployment_target: { ios: "16.0" }`（[9.10](../009-tools/910-nabi-frontend.md)），可以提高版本，更低的版本是错误，错误会指出最低版本。链接需要 Xcode 的 iOS SDK：`SDKROOT`，否则使用开发者目录（`DEVELOPER_DIR`，否则为 `xcode-select` 指定的目录）中的 `iPhoneSimulator` SDK；如果都没有，构建会提示安装 Xcode。如果 `SDKROOT` 是其他平台的 SDK（例如对 iOS 目标给出了 macOS SDK，依据其 `SDKSettings.json` 或目录名判断），它会被跳过并附带一条提示，转而使用开发者目录中的 SDK。目标文件和可执行文件会标明其平台（`LC_BUILD_VERSION`：iOS 模拟器），链接器拒绝为其他 Apple 平台构建的目标文件。基准 CPU 是 `apple-m1`。`target_os` 为 `"ios"`，`target_abi` 为 `"sim"`（[6.3.2](../006-meta-system/603-cfg.md)）。可执行文件采用 ad hoc 签名，由模拟器运行：`nabi run`、`nabi test` 和 `nabi bench` 通过该目标的 `simrun.sh` 运行它，`simrun.sh` 在已启动的设备（或 `NABI_SIM_DEVICE` 指定的设备）上运行 `xcrun simctl spawn`，并把每个环境变量以 `SIMCTL_CHILD_<name>` 的形式传递下去，因为 `simctl` 只把这些变量交给程序。应用中的 Nabi 代码是一个由 Xcode 链接的库：`--crate-type=staticlib`（[9.3.3](../009-tools/903-package-archive.md#933-static-and-shared-libraries)）构建一个包含该包、其依赖和运行时的归档文件，Swift 读取其模块映射（`import <name>`）。`--crate-type=plugin` 和 `--hot` 可以在模拟器上使用；REPL 不能在那里运行。维护者在自己的机器上运行模拟器测试语料。

设备目标 `aarch64-apple-ios`（iPhone 和 iPad）处于进行中状态（0.6.2）。下面这段文字只适用于它。它与模拟器一样为 iOS 13.4 及更高版本构建，使用 `iPhoneOS` SDK 链接，并在 `LC_BUILD_VERSION` 中标明 iOS 平台；它的基准 CPU 是 `apple-a7`（与 A9 一样没有 LSE 原子指令），并且不设置 `target_abi`。设备只有在开发者用 Xcode 为其签名后才能运行可执行文件，所以 `nabi run`、`nabi test` 和 `nabi bench` 会报错，错误会指出模拟器目标。在设备上，应用无法启动进程，所以 `Command` 在那里抛出 `Unsupported`，并且没有任何东西会在运行时加载代码：`--crate-type=plugin` 和 `--hot` 是错误。

### Android

| 目标                    | 说明                                      |
|-------------------------|-------------------------------------------|
| `x86_64-linux-android`  | Android x86-64（模拟器），API 级别 29 及更高 |

`x86_64-linux-android` 为 Android 10（API 级别 29）及更高版本构建可执行文件和共享库（`--crate-type=cdylib`，由 `System.loadLibrary` 加载）；把它们打包进应用是 Android 构建的工作。API 级别 29 是最低级别，因为 Android 的 C 库 bionic 从该级别起才能运行 ELF 线程局部存储，而运行时把每个线程的状态保存在其中。`ANDROID_PLATFORM=<n>`（或 NDK 自己的构建工具所接受的 `android-<n>`）为更高的级别构建；低于 29 的级别是错误，错误会提示写 `ANDROID_PLATFORM=29` 或更高。链接从 Android NDK 中取得该级别的 C 运行时目标文件和 bionic 的库，NDK 不属于工具链：`ANDROID_NDK_HOME`，否则 `ANDROID_NDK_ROOT`，否则 Android SDK（`ANDROID_HOME`，否则 `ANDROID_SDK_ROOT`，否则 Android Studio 为其安排的位置）中最新的 `ndk/<version>`。没有 NDK 时，错误会说明查找过的位置，并提示设置 `ANDROID_NDK_HOME`；如果 NDK 中没有该级别的库，错误会列出它拥有的级别。`nabicc` 从同一个 NDK 中取得 bionic 的头文件（[9.13.7](../009-tools/913-c-compiler.md#9137-headers-and-c-library)）。`target_os` 为 `"android"`，没有 `target_env`。内核是 Linux，所以 `std` 在 `target_os = "linux"` 下提供的内容（`std::os::linux`、`open` 标志……）在 Android 上同样存在。可执行文件是面向 `/system/bin/linker64` 的 PIE。`x86_64-linux-android` 为 x86_64 Android ABI `x86-64-v2`（SSE4.2、POPCNT）构建，支持最大 16 KiB 的页面。应用的进程运行在一个 seccomp 过滤器之下，对于 bionic 不会发出的系统调用，该过滤器会杀死进程，所以运行时在那里不发出任何自己的系统调用：`ChildExt::pidfd` 抛出错误（`ENOSYS`），`Metadata::created` 抛出 `Unsupported`，文件到文件的复制通过 `sendfile` 进行。在 Android 14（API 级别 34）之前，bionic 的 `posix_spawn` 无法改变子进程的目录，所以带有 `current_dir` 的 `Command` 会先 fork，再在那里 exec。

设备目标 `aarch64-linux-android`（Android arm64，API 级别 29 及更高）处于进行中状态（0.6.2）。它的构建方式与 `x86_64-linux-android` 相同，上面的文字同样适用于它，只是它支持最大 64 KiB 的页面。

这些目标覆盖了科学、数据、Jupyter 和桌面工作中的大多数用户。

### 32 位支持

由于支持 32 位目标，整个规范都被定义为可以在 32 位和 64 位环境中工作。对象头布局在两种位宽下相同。在 32 位环境中，64 位整数的原子操作使用目标原生的 64 位原子指令（i686 上是 `cmpxchg8b`，ARMv7 上是 `ldrexd` / `strexd`），没有这类指令时则使用系统原子库（riscv32，进行中：使用运行时自带的，因为它只有一个线程）。`usize` 的位宽自动跟随目标的原生指针宽度。在 wasm32 上，堆是线性内存，按需增长，直到 wasm32 的 4 GiB 上限。在其他 32 位目标上，垃圾回收器可用的堆约为 3.5 GB：块堆最多预留 2 GiB 的连续空间（操作系统拒绝时更少），地址空间的其余部分存放大对象，Windows x86 映像带有 large-address-aware 标志。在 riscv32 上，堆是开发板分配的内存（0.6.2）。在 64 位上，堆实际上没有限制。时间在 32 位目标上同样使用 64 位接口（`time64`），所以 `SystemTime` 在任何地方的范围都相同。

### 测试

CI 按下表所示构建和测试已支持的目标。级别指 `nabic` 的优化级别，测试语料是编译器的端到端测试套件和差分测试套件。

| 目标 | 每个 pull request | 合并队列（合并的必要条件） | 每晚 |
|---|---|---|---|
| `x86_64-unknown-linux-gnu` | 全部测试，`-O0` 下的测试语料 | 全部测试，`-O0` 下的测试语料 | 全部测试；`-O0`、`-O1`、`-O2`、`-O3`、`-Os` 和 `-O2 -g` 下的测试语料；`-O2` 下的 GC 压力测试 |
| `x86_64-pc-windows-msvc` | 全部测试，`-O0` 下的测试语料 | 全部测试，`-O0` 下的测试语料 | 全部测试；`-O0` 和 `-O2` 下的测试语料 |
| `aarch64-apple-darwin` | | 全部测试，`-O0` 下的测试语料 | 全部测试；`-O0`、`-O1`、`-O2`、`-O3`、`-Os` 和 `-O2 -g` 下的测试语料 |
| `aarch64-unknown-linux-gnu` | | 全部测试，`-O0` 下的测试语料 | 全部测试；`-O0` 和 `-O2` 下的测试语料；`-O2` 下的 GC 压力测试 |
| `aarch64-pc-windows-msvc` | | | 全部测试；`-O0` 和 `-O2` 下的测试语料，分别使用 arm64 工具链和在模拟下运行的 x64 工具链 |
| `x86_64-unknown-linux-musl`、`aarch64-unknown-linux-musl` | | | 在能运行该目标的主机上，`-O0` 和 `-O2` 下的测试语料 |
| `i686-pc-windows-msvc` | | | `-O0`、`-O2` 和 `-O2 -g` 下的测试语料 |
| `i686-unknown-linux-gnu` | | | `-O0` 和 `-O2` 下的测试语料 |
| `armv7-unknown-linux-gnueabihf` | | | 在 qemu-user 下，`-O0` 和 `-O2` 下的测试语料 |
| `wasm32-wasip2`、`wasm32-wasip1-threads`、`wasm32-unknown-unknown` | | | 在 wasmtime 和 node 下，`-O0` 和 `-O2` 下的测试语料 |
| `x86_64-unknown-freebsd` | | | 在 FreeBSD 14 虚拟机中，`-O2` 下的测试语料 |

只有合并队列的运行通过后，变更才会合并。每晚任务失败时会创建一个 issue。每晚运行还会在 qemu-user 下以 `-O0` 和 `-O2` 构建三个进行中目标（0.6.2）的测试语料：`riscv64gc-unknown-linux-gnu`、`powerpc64le-unknown-linux-gnu` 和 `s390x-unknown-linux-gnu`。目标与优化级别的其他组合在发布之前由维护者在自己的测试机器上运行，而不在 CI 中运行；`armv7-unknown-linux-gnueabihf` 和 `armv7-unknown-linux-musleabihf` 在那里以原生方式在每个级别上运行，所用的是一台 CPU 也能运行 32 位 ARM 代码的 arm64 机器（Neoverse N1）；`x86_64-linux-android` 只在那里于 Android 模拟器中测试，`aarch64-apple-ios-sim` 则在 iOS 模拟器中测试。`wasm32-unknown-unknown` 在那里也会在 Chromium、Firefox 和 WebKit 中以 `-O0` 和 `-O2` 运行其测试语料，每个程序运行在页面中专用的 Worker 里。同样在那里，WASI 目标还会在 wasmtime 之外的运行时上以 `-O0` 和 `-O2` 运行测试语料：`wasm32-wasip2` 在 jco 下运行（组件被转译为 JavaScript，运行在 Node.js 上），`wasm32-wasip1-threads` 在 WAMR 的 iwasm 和 wasmer 下运行。组件的导出在 jco 下也会被调用，这些运行时之一无法运行的用例（属于运行时的缺陷或空缺）会在该运行时上跳过。

## 0.6.2. 进行中

下列目标是内置的，但尚未在其所面向的硬件上测试，所以它们不受支持。每个目标都将在 0.2.0 之前的某个后续版本中得到支持，届时移入 0.6.1。以它们的名称作为 `--target`（无论是给 `nabic`、`nabicc` 还是 `nabi`，`--print target-spec` 也一样）是错误，错误会说明该目标正在进行中，以及目前应该为什么目标构建；`nabic --print target-list` 不列出它们。

| 目标 | 处于进行中的原因 | 目前的替代 |
|--------|-----------------------|---------|
| `aarch64-apple-ios`（iOS 设备） | 能构建和链接；尚未在 iPhone 或 iPad 上运行 | `aarch64-apple-ios-sim`，在 iOS 模拟器中 |
| `aarch64-linux-android`（Android arm64） | 已在 qemu-user 下配合 Android 系统映像测试；尚未在设备上测试 | `x86_64-linux-android`，在 Android 模拟器中 |
| `riscv64gc-unknown-linux-gnu`（64 位 RISC-V Linux） | 已在 qemu-user 下测试；尚未在 RISC-V 硬件上测试 | 无 |
| `powerpc64le-unknown-linux-gnu`（IBM Power） | 已在 qemu-user 下测试；尚未在 POWER 硬件上测试 | 无 |
| `s390x-unknown-linux-gnu`（IBM Z；大端） | 已在 qemu-user 下测试；尚未在 IBM Z 上测试 | 无 |
| `riscv32imac-unknown-none-elf`（RISC-V 嵌入式，无操作系统） | 已在 qemu-riscv32 下测试；尚未在开发板上测试 | 无 |

在测试期间，它们的代码保留在工具链中。维护者在环境中设置 `NABI_IN_PROGRESS_TARGETS=1` 来构建它们，这让 `--target` 和 `--print target-list` 接受它们。这是一个用于测试工具链的开关，不属于 Nabi：它不作任何承诺，并且可能在任何版本中改变或消失。本页对这些目标的描述，反映的是它们目前的构建情况。

### `riscv32imac-unknown-none-elf`

`riscv32imac-unknown-none-elf` 没有操作系统。程序与运行时（不带 C 库）链接成一个静态映像，并在一个线程上运行：与 `wasm32-wasip2` 上一样，`thread::spawn` 触发 panic，`thread::Builder::spawn` 抛出 `Unsupported`。文件、进程、套接字和 C 库函数都不存在，使用它们的程序无法链接；没有环境变量（设置环境变量会抛出 `Unsupported`），并且 `random` 因缺少熵源而触发 panic。运行时提供映像的入口点 `_start` 及其堆；`_start` 像 Linux 进程启动时那样从初始栈中取得参数，设置 TLS 块并调用 `main`。内存、输出、时间和程序的结束来自四个开发板函数：`nabi_bare_morecore(len)`、`nabi_bare_write(fd, ptr, len)`、`nabi_bare_clock_ns(realtime)` 和 `nabi_bare_exit(code)`。运行时自己对 `_start` 和这些函数的定义是弱符号，并发出 Linux 系统调用，`qemu-riscv32` 正是以此在用户模式下运行该映像；开发板定义自己的版本，其启动代码设置 `sp` 和 `gp`，并调用 `nabir_bare_start(null)`（无参数）。浮点运算、64 位除法和数学函数来自运行时（compiler_builtins），所以 `exp`、`ln` 等的结果可能与 glibc 的结果在最后一位上有所不同。

## 0.6.3. 不支持

其他所有目标都不受支持。以其名称作为 `--target` 是一个未知目标（[9.1.2](../009-tools/901-nabic-cli.md#912-core-subcommands)）。对于下列名称，错误会说明原因，并在有替代时指出应改用什么。

| 目标 | 替代 |
|--------|---------|
| `arm-unknown-linux-gnueabi`（ARMv6，软浮点：Raspberry Pi Zero / 1） | 无 |
| `armv7-linux-androideabi`（32 位 Android） | 无；`aarch64-linux-android` 正在进行中（0.6.2） |
| `thumbv6m-none-eabi`、`thumbv7m-none-eabi`、`thumbv7em-none-eabi`、`thumbv7em-none-eabihf`、`thumbv8m.main-none-eabi`、`thumbv8m.main-none-eabihf`（Cortex-M） | 无 |
| `x86_64-apple-ios`（Intel Mac 上的 iOS 模拟器） | `aarch64-apple-ios-sim`（0.6.1），在 Apple Silicon Mac 上 |
| `x86_64-apple-darwin`（Intel Mac） | `aarch64-apple-darwin`，在 Apple Silicon Mac 上 |

## 0.6.4. SIMD

构建以三元组的基准 CPU 为目标，所以二进制文件是可移植的：`x86_64` 为 x86-64 基准（SSE2），`i686` 为 `pentium4`（SSE2），`aarch64-apple-darwin` 和 `aarch64-apple-ios-sim` 为 `apple-m1`，`aarch64-apple-ios` 为 `apple-a7`（NEON，无 LSE 原子指令），其他 `aarch64` 目标为 armv8-a（NEON，无 LSE 原子指令），`armv7` 为带 NEON 的 ARMv7，`powerpc64le` 为 POWER8（VSX），`s390x` 为 z10（无向量设施），`wasm32` 为 SIMD128。在 `-O0` 以上，代码生成器在 `x86_64`、`i686`、`aarch64`、`armv7`（仅整数通道）、`powerpc64le` 和 `wasm32` 上针对该基准对循环进行向量化，在 `s390x` 和 `riscv64` 上则借助其向量特性进行向量化（[8.7.8](../008-abi/807-code-generation.md#878-loop-vectorization)）：遍历列表元素的循环，包括融合的点表达式所变成的那一个循环（[3.3.4](../003-expressions/303-broadcast-operators.md#334-automatic-loop-fusion)），可以用 128 位向量运行。向量化的循环计算与标量循环相同的结果，并在相同的地方陷入（trap）。在其他目标上，每个循环都是标量代码。`-C target-cpu=<cpu>|native` 和 `-C target-feature=+avx2,...`（[9.1](../009-tools/901-nabic-cli.md)）选择 CPU 的特性，代码生成器在 `x86_64` 上使用其中一部分：有 `avx2` 时（`x86-64-v3`、`haswell`、`znver1` 及更新的 CPU），向量循环使用 256 位 AVX2 向量，并在由测试选出通道的地方使用掩码存储；有 `popcnt`、`lzcnt` 和 `bmi` 时，位计数（`count_ones`、`leading_zeros`、`trailing_zeros`）各自是一条指令；有 `fma` 时，`mul_add` 无论在标量代码还是向量循环中都是一条 `vfmadd` 指令，但程序没有用 `mul_add` 连接起来的乘法和加法永远不会被融合，因为那样会产生不同的舍入。在 `aarch64` 上，它使用 ARMv8.1 原子指令 `lse`（`apple-m1`、`cortex-a76`、`neoverse-n1` 及更新的 CPU）：原子的读-改-写、交换或比较交换都是一条指令（`ldadd`、`swp`、`cas`……）；没有它时，例如在 `aarch64` Linux 和 Windows 的基准上，每个操作都是一个独占加载 / 存储循环（`ldxr` / `stxr`，或按内存序要求使用 `ldaxr` / `stlxr`），它能在每个 ARMv8.0 CPU（Cortex-A53、Cortex-A72……）上运行。在 `-O0` 以上的 `aarch64` Linux 上，每个这类操作还会测试一个由运行时在启动时根据内核的 `AT_HWCAP` 设置的标志，在 CPU 支持 LSE 时采用 LSE 指令。有 ARMv8.3 RCpc 加载 `rcpc` 时（`apple-m1`、`cortex-a76`、`neoverse-n1` 及更新的 CPU），在 `-O0` 以上，要求 `Acquire` 的原子加载是 `ldapr`，正如获取加载所允许的那样，较早的释放存储可以越过它；否则是 `ldar`。有 `dotprod` 时（同样的 CPU），对字节乘积求和的循环（`s += (a[i] as i32) * (b[i] as i32)`，元素为 `i8` 或 `u8`）每次运行十六个乘积，每个通道四个（`sdot`、`udot`）。`fullfp16`、`crc`、`sve` 和 `sve2` 不改变任何代码。在 `s390x` 上，它使用 `z13` 及更新 CPU 的向量设施 `vector`：128 位的向量循环，浮点通道为 8 字节，有 `vector-enhancements-1` 时（`z14` 及更新）也可以是 4 字节；基准 z10 没有向量寄存器。在 `riscv64` 上，它使用 `v`（RVV 1.0，`-C target-feature=+v`）：128 位的向量循环，无论向量长度是多少，每台支持 V 扩展的机器都能容纳。在 `riscv64` 和 `riscv32` 上，它使用 `zbb`：32 位或 64 位整数按常量循环移位（`rotate_left`、`rotate_right`，或拼出循环移位的两次移位加一次或运算）是一条指令（`rori`），否则是两次移位加一次或运算。在 `armv7` 上，它使用 `vfp4`：`mul_add` 是一条 `vfma` 指令。`armv7` 上的 `-C target-feature=-neon`、`powerpc64le` 上的 `-vsx` 和 `wasm32` 上的 `-simd128` 会使所有循环都以标量方式构建（默认的 `+simd128` 会重新打开 SIMD128）。其他特性，以及 `i686` 上的所有特性，不改变任何代码：只指定了这些特性的构建会发出相应警告。CPU 名称代表其特性（`x86-64-v2`、`x86-64-v3`、`x86-64-v4`、`haswell`、`znver4`、`apple-m1`、`neoverse-n1`、`pwr9`、`z13`……）；未知的名称是错误，错误会列出已知的名称。`native` 在基准特性之上加入构建机器的特性，这些特性在 `nabic` 运行时检测：生成的二进制文件可能无法在其他机器上运行。除了这个标志之外，生成的代码在运行时不做任何特性检测，也不会为了在运行时挑选而把任何东西编译成多个版本，`par::map` 也不例外（[stdlib 2.19.6](../../refs-standard-libraries/200-core/219-par.md#2196-simd--gpu-integration)）。

可移植的 SIMD 向量，即带有逐通道运算符的 `Simd<T, N>`，是 `core::simd`（[stdlib 2.24](../../refs-standard-libraries/200-core/224-simd.md)）；在上述目标上，它们的运算在每个优化级别下都是向量代码，在其他地方则是逐通道的循环。

## 0.6.5. 目标规格

每个目标都由一份目标规格描述；编译器、链接器和运行时构建不读取关于目标的任何其他信息。目标由 `--target` 指定：一个内置名称（`nabic --print target-list`：即 0.6.1 中的目标），或一个文件 `<name>.nabion` 的路径，该文件包含一个 NABION 值 `TargetSpec { .. }`（自定义目标，以文件的主干名命名）。

`nabic --print target-spec [--target=<t>]` 以这种形式打印目标的规格，所以自定义目标通常从复制一个内置目标开始。自定义文件必须给出每个字段；未知字段是错误。自定义目标的运行时，以及在该目标附带 C 库时的 C 库，来自工具链的 `lib/nabi/targets/<name>/`；没有这个目录时，需要链接的构建会失败，并指出该目录。

| 字段 | 含义 |
|-------|---------|
| `c_target`、`deployment_target` | clang 所接受形式的目标三元组（`NABI_CC`，以及 `nabi bindgen` 的 C 前端）；附加在其后的最低操作系统版本（Mach-O；在 macOS 上可用 `MACOSX_DEPLOYMENT_TARGET` 覆盖，在 iOS 上可用 `IPHONEOS_DEPLOYMENT_TARGET` 覆盖；Android：API 级别，可用 `ANDROID_PLATFORM` 覆盖） |
| `arch`、`os`、`env`、`abi`、`family`、`endian`、`pointer_width` | `target_*` cfg 值（[6.3.2](../006-meta-system/603-cfg.md)）；为空的 `env` / `abi` / `family` 不设置任何键。在 Apple 模拟器上 `abi` 为 `"sim"`，否则为空 |
| `vendor` | 资料性字段 |
| `wide_align`、`max_atomic_width`、`c_abi` | `i64` / `u64` / `f64` 的对齐、最宽的原子类型、C 调用约定（[8.6.3](../008-abi/806-calling-convention.md)） |
| `cpu`、`features` | 以 clang 的名称表示的默认 CPU 和特性（即 0.6.4 中的 SIMD 基准） |
| `object_format`、`linker`、`pre_link_args`、`crt_objects`、`late_link_args` | 目标文件格式（`ObjectFormat::MachO`、`Elf`、`Coff`、`Wasm`）；`nabic` 的链接器生成什么（`Linker::MachO`、`Elf`、`Coff`、`Wasm`，或 `WasmComponent`：把 wasm 模块转换为 WASI preview 2 组件），它同时决定链接参数的命令行风格；目标自己的链接器参数；C 运行时目标文件；位于输入之后的参数或文件 |
| `dynamic_linking`、`dynamic_loader`、`libc` | 共享库支持、ELF 解释器、C 库以及它是否随工具链一起提供 |
| `rust_target` | 运行时为之构建的 Rust 目标 |
| `exe_suffix`、`staticlib_prefix` / `staticlib_suffix`、`dylib_prefix` / `dylib_suffix`、`c_symbol_prefix` | 文件和符号的命名 |
| `stack_size`、`stack_probes`、`threads` | 主线程的栈、栈探测、线程支持 |
| `native_backend`、`repl` | 代码生成器是否接受该目标；REPL 主机是否在其上运行 |

规格有一个稳定的哈希；每个构建缓存都以它为键。
