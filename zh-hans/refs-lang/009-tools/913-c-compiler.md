# 9.13. `nabicc`：C 编译器

工具链用自己的 C 编译器编译 C 代码。它用与 `nabi bindgen`（[9.10.8](910-nabi-frontend.md#nabi-bindgenc-头文件绑定)）相同的前端读取 C，将其降级为 nabi-mir，再用 `nabic` 所用的代码生成器和链接器进行编译和链接（[0.2.4](../000-meta/002-pipeline.md#024-代码生成)）。不涉及任何其他 C 编译器、汇编器或链接器，也无需安装它们。

该前端也读取 Objective-C 头文件，供 `ffi::objc::import` 和 `nabi bindgen -x objective-c` 使用（[标准库 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-头文件ffiobjcimport)）：只读取声明（`@interface`、`@protocol`、方法、属性），预定义 `__OBJC__`、`__OBJC2__` 和 ARC 的特性，且每个文件的 `#import` 只读取一次。`nabicc` 本身只编译 C；`.m` 文件不是 C 源文件。

## 9.13.1. `nabicc` 与 `nabi cc`

C 编译器有两种形式，行为相同：

- `nabicc`，一个独立的可执行文件。它安装在工具链的 `bin/` 中，并作为代理安装在 `$NABI_HOME/bin/` 中，与 `nabi`、`nabic` 和 `nabils` 并列（[9.10.6](910-nabi-frontend.md#9106-toolchain)）。
- `nabi cc`。`nabi` 运行其自身工具链中的 `nabicc`，原样传入 `cc` 之后的参数，正如它把工作委派给 `nabic` 一样（[9.10.1](910-nabi-frontend.md#9101-子命令与全局选项)）。`nabi` 的全局选项可以写在 `cc` 之前；`cc` 之后的每个词都属于 C 编译器。

```
nabicc [options] <file>...
nabi cc [options] <file>...
```

命令行与 gcc 和 clang 接受的相同，因此两种形式都可以直接作为构建系统的 C 编译器替换使用：`CC=nabicc` 和 `CC="nabi cc"` 都能与 make、CMake、autoconf 和 Meson 配合使用。

```
make CC="nabi cc"
cmake -DCMAKE_C_COMPILER=nabicc -B build
```

消息会写出所运行的形式：`nabi cc: error: ..` 或 `nabicc: error: ..`。成功时退出码为 `0`；源文件编译失败、选项错误或链接失败时为 `1`。编译器内部错误以 [9.1.6](901-nabic-cli.md#916-退出码) 中的退出码退出。

## 9.13.2. 语言

C89、C99、C11、C17 和 C23 作为同一种方言读取：为其中任何一个编写的程序都能编译，较新的语法（`_Static_assert`、`typeof`、`[[attributes]]`、作为关键字的 `bool`、二进制字面量、数字分隔符）无论选择哪个标准都被接受。`-std=` 选择程序通过宏所看到的标准：

| `-std=` | `__STDC_VERSION__` |
|---------|--------------------|
| `c89`、`c90`、`gnu89`、`gnu90`（以及 `-ansi`） | 未定义 |
| `c99`、`gnu99` | `199901L` |
| `c11`、`gnu11` | `201112L` |
| `c17`、`c18`、`gnu17`、`gnu18` | `201710L` |
| `c23`、`c2x`、`gnu23`、`gnu2x` | `202311L` |

也接受 `iso9899:<year>` 写法。默认为 `gnu17`。`c` 形式（而非 `gnu`）还会定义 `__STRICT_ANSI__`。其他任何标准都是错误，错误消息会列出上述标准。

系统头文件所用的 GNU、Clang、Apple 和 Microsoft 扩展都能读取：`__attribute__((..))`、`__declspec(..)`、`__asm__("symbol")` 标签、`__extension__`、`typeof`、语句表达式、`__builtin_*` 调用（9.13.4）、可空性限定符、匿名结构体和联合体、`#pragma pack`、`#include_next`、`__has_include`、`__has_attribute`、`__has_builtin` 和 `__has_feature`。类型按目标的 C ABI 布局，与 clang 为该目标布局的方式相同。

预定义宏是 clang 为该目标定义的宏（`__x86_64__`、`__aarch64__`、`__linux__`、`__APPLE__`、`_WIN32`、`__SIZEOF_LONG__`、非 MSVC 目标上的 `__GNUC__`、MSVC 目标上的 `_MSC_VER` 等），另加 `__nabicc__`（`1`）。向量扩展及其特性宏（`__SSE__`、`__SSE2__` .. `__AVX512F__`、`__ARM_NEON`、`__ARM_FEATURE_*`、`__wasm_simd128__`、`__riscv_vector`、`__ALTIVEC__` 等）均未定义，`__SIZEOF_INT128__` 也未定义：目前还没有向量内建函数，因此检查这些宏的库会走标量路径。

## 9.13.3. 拒绝的内容

下列构造在其 `file:line:col` 处报错。每个错误都会说明应当改写成什么。在 `inline` 或 `static` 函数的函数体中，只有当单元调用该函数时这些构造才是错误：没有代码使用的头文件内联函数（C 库的 SIMD `wmemchr`、`cpuid` 辅助函数）不会阻止构建。

| 构造 | 改写为 |
|-----------|---------------|
| 内联汇编（`asm`、`__asm__`），但空屏障（`asm("")`、`asm volatile("" ::: "memory")`）和不带操作数的单独自旋提示（`"pause"`、`"yield"`、`"nop"`）除外，它们编译后不产生任何代码 | 用 C 编写这段代码，或使用选择库的可移植代码路径的宏 |
| 标签的地址（`&&label`）和计算型 `goto *p` | `switch` |
| 变长数组（大小不是常量的数组），但参数除外，参数是指针 | 固定大小，或 `malloc` 和 `free` |
| `alloca`（`__builtin_alloca`、`_alloca`） | `malloc` 和 `free`，或定长数组 |
| `_Complex` 值上的运算（`__real__`、`__imag__`、运算符） | 用两个变量分别保存实部和虚部 |
| `__int128` 和 `unsigned __int128` | 两个 64 位的半部分 |
| `long double` 比 `double` 宽时的 `long double` 值（x86 和 x86-64 Linux 及 FreeBSD 上为 80 位 x87；aarch64、riscv64、powerpc64le 和 s390x Linux、Android、riscv32 以及 wasm32 上为 128 位） | `double`（许多库接受诸如 `-DLONGDOUBLE_TYPE=double` 的选项） |
| `_Float16`、`__fp16` 和 `__bf16` 上的运算 | 先转换为 `float` |
| 定义线程局部变量（`_Thread_local`、`thread_local`、`__thread`、`__declspec(thread)`）；线程局部变量的 `extern` 声明（例如 wasi-libc 的 `errno`）可以读取和使用 | 通过参数传递状态，或保存在每个线程各有一份的结构体中 |
| `aarch64-pc-windows-msvc` 上的可变参数函数定义（其他目标可以编译） | `va_list` 版本 `int vf(const char *fmt, va_list ap)`，由一个非可变参数的包装函数调用 |
| Windows 结构化异常处理（`__try`、`__except`、`__finally`、`__leave`） | 不使用它来构建该库（sqlite：`-DSQLITE_OMIT_SEH`） |
| 向量类型（`__attribute__((vector_size(N)))`、`__m128`、`float32x4_t`、SVE 类型） | 库的标量代码路径（其 `NO_SIMD` / `NO_SSE` 选项） |
| `__builtin_frame_address`、`__builtin_return_address` 以及 9.13.4 未列出的内建函数 | 用 C 实现该操作，或使用 C 库的函数 |
| K&R 风格的定义（`int f(a, b) int a; ..`） | 原型：`int f(int a, int b)` |
| 隐式 `int`（`static x;`） | 写出类型：`static int x;` |
| 调用未声明的函数 | `#include` 声明它的头文件，或在调用前写出其原型 |

在 Apple 目标、Windows 和 32 位 Arm 上，目标的 C ABI 使 `long double` 为 64 位，即等同于 `double`；在这些目标上它按 `double` 编译。

被拒绝的选项和输入：

| 选项或输入 | 错误 |
|-----------------|-------|
| `-shared` | `nabi cc` 链接的是可执行文件；用 `ar` 构建静态库（或用 `nabi build` 构建一个包） |
| `.s`、`.S` 或 `.asm` 源文件 | `nabi cc` 不做汇编；用 C 编写该函数，或链接在别处构建的目标文件 |
| C++ 或 Objective-C 源文件（`.cc`、`.cpp`、`.cxx`、`.C`、`.m`、`.mm`），或 `-x` 指定了 `c`、`c-header`、`none` 以外的语言 | `nabi cc` 只编译 C；C++ 和 Objective-C 需要各自的编译器 |

## 9.13.4. 内建函数

下列内建函数无需声明即可编译。每个都带 `__builtin_` 前缀（`__builtin_expect`）；最后几行中的名称有各自的前缀。

| 内建函数 | 含义 |
|---------|---------|
| `va_start`、`va_arg`、`va_end`、`va_copy` | 按目标的 C ABI 处理可变参数，与 `<stdarg.h>` 的用法相同。在没有 `...` 的函数中使用 `va_start` 是错误 |
| `expect`、`expect_with_probability` | 返回该值；忽略提示 |
| `assume`、`assume_aligned`、`prefetch` | 返回该值，或什么也不做；不影响代码 |
| `unreachable` | 到达时为未定义行为 |
| `trap`、`debugtrap`（以及 `__debugbreak`） | 以 trap 指令停止程序 |
| `constant_p` | 整数或浮点常量为 `1`，否则为 `0` |
| `object_size`、`dynamic_object_size` | 未知：类型 `0` 和 `1` 为 `(size_t)-1`，类型 `2` 和 `3` 为 `0` |
| `clz`、`ctz`、`popcount`（各带 `l` 和 `ll` 形式）、`bswap16`、`bswap32`、`bswap64` | 位计数和字节交换 |
| `huge_val`、`inf`、`nan`、`nans`（各带 `f` 和 `l` 形式） | 浮点常量 |
| `isnan`、`isinf`、`isinf_sign`、`isfinite`、`isnormal`、`signbit`、`fpclassify`、`isgreater`、`isgreaterequal`、`isless`、`islessequal`、`islessgreater`、`isunordered` | 分类和比较，供 `<math.h>` 的宏使用 |
| `add_overflow`、`sub_overflow`、`mul_overflow` 及其带类型的形式（`sadd_overflow`、`umulll_overflow` 等） | 存储回绕后的结果并返回是否溢出的算术运算 |
| 内存、字符串和数学函数（`memcpy`、`memset`、`strlen`、`strcmp`、`malloc`、`abs`、`sqrt`、`sin`、`pow`、`fabsf` 等） | 调用 C 库中去掉前缀后同名的函数 |
| 强化调用（`__builtin___memcpy_chk`、`__builtin___snprintf_chk` 等） | 调用普通函数（`memcpy`、`snprintf`）：对象大小未知（见上），因此这些检查总会通过 |
| `offsetof`（`__builtin_offsetof`） | 成员的偏移，为常量表达式 |
| `__atomic_*`、`__sync_*`、`__c11_atomic_*`、`_InterlockedIncrement`、`_InterlockedDecrement`、`_InterlockedExchange`、`_InterlockedExchangeAdd`、`_InterlockedCompareExchange`、`_ReadWriteBarrier` | 加载、存储、交换、比较并交换以及取值并运算，含义与各内建函数相同；`__atomic_is_lock_free` / `__atomic_always_lock_free` 对于不超过两个指针大小的 2 的幂大小为 `1` |

`__func__`、`__FUNCTION__` 和 `__PRETTY_FUNCTION__` 是外围函数的名称。

> [!WARNING]
> 原子内建函数被编译为普通的（volatile）加载和存储以及普通的读-改-写序列，内存栅栏（`__atomic_thread_fence`、`__sync_synchronize`、`_ReadWriteBarrier`）编译后不产生任何代码。它们在单线程中给出各内建函数的结果，但在线程之间并不是原子的。通过它们在线程之间共享内存的 C 代码在 `nabicc` 下目前还不正确。

## 9.13.5. 选项

| 选项 | 含义 |
|--------|---------|
| `-c` | 把每个源文件编译为目标文件；不链接 |
| `-S` | 把每个源文件编译为汇编文本（`<stem>.s`） |
| `-E` | 只做预处理，输出到标准输出或 `-o` |
| `-o <file>` | 输出。与 `-c`、`-S` 或 `-E` 一起使用时只接受一个源文件 |
| `-I <dir>`、`-isystem <dir>`、`-iquote <dir>`、`-idirafter <dir>` | 头文件搜索目录，按 gcc 的方式搜索 |
| `-include <file>` | 在源文件之前读取 `<file>` |
| `--sysroot <dir>`、`-isysroot <dir>` | 目标的头文件和 C 库（9.13.7） |
| `-nostdinc`、`-nostdlibinc`、`-nobuiltininc` | 不使用系统头文件、C 库的头文件或内置头文件 |
| `-D <name>[=<value>]`、`-U <name>` | 定义或移除宏 |
| `-Wp,<opts>` | 本表中的预处理器选项，以逗号分隔 |
| `-std=<std>` | 标准（9.13.2） |
| `-O0`、`-O1`、`-O2`、`-O3`、`-Os`、`-Oz` | 优化级别（默认 `-O0`）；`-O` 即 `-O1`，`-Og` 即 `-O1`，`-Ofast` 即 `-O3` |
| `-g`、`-g0` | 开启或关闭调试信息（`-ggdb`、`-g2`、`-g3`、`-gdwarf*`、`-gcodeview` 会开启它） |
| `-l<lib>`、`-L<dir>`、`-Wl,<opts>` | 链接所用的库、库目录和链接器选项 |
| `-M`、`-MM` | 打印源文件及其读取的头文件的 make 规则，不编译；`-MM` 不包含系统头文件 |
| `-MD`、`-MMD` | 在编译的同时把该规则写入 `<output or stem>.d`；`-MMD` 不包含系统头文件 |
| `-MF <file>` | 把规则写入 `<file>` |
| `-MT <target>`、`-MQ <target>` | 规则的目标（可重复）；默认为目标文件 |
| `-MP` | 为每个头文件添加一条空规则，这样删除某个头文件不会使 make 中止 |
| `--target=<triple>`、`-target <triple>` | 为另一个目标编译（9.13.7） |
| `-x c` | 无论扩展名如何，把后续输入都当作 C 读取 |
| `-dumpmachine` | 打印目标的 C 三元组 |
| `-dumpversion`、`-dumpfullversion` | 打印工具链的版本 |
| `--version` | 打印 `nabicc <version>`；与 `-v` 一起使用时，还打印 `host: <triple>` |
| `-v` | 详细输出（与 `--version` 一起使用） |
| `--help`、`-h` | 打印用法 |

`--target` 接受 Nabi 的目标名称（`nabi --print target-list`，[0.6](../000-meta/006-platform-targets.md)），以及 clang 和 gcc 为这些目标接受的三元组：`x86_64-linux-gnu` 即 `x86_64-unknown-linux-gnu`，`arm64-apple-macos` 即 `aarch64-apple-darwin`，`aarch64-linux-musl` 即 `aarch64-unknown-linux-musl`，`i386-linux-gnu` 即 `i686-unknown-linux-gnu`，`armv7-linux-gnueabihf` 即 `armv7-unknown-linux-gnueabihf`，`armv7-linux-musleabihf` 即 `armv7-unknown-linux-musleabihf`，`riscv64-linux-gnu` 即 `riscv64gc-unknown-linux-gnu`。不属于这些的名称是错误。

以下选项被接受但不起作用：所有其他 `-W*`（警告）、`-f*` 和 `-m*` 选项，`--param`、`-fPIC`、`-fpic`、`-fPIE`、`-fpie`、`-pie`、`-no-pie`、`-rdynamic`、`-pthread`、`-pedantic`、`-pedantic-errors`、`-w`、`-pipe`、`-s`、`-static` 和 `-Qunused-arguments`。代码总是位置无关的。其他任何选项都是错误，错误消息会列出上述选项。

以 `.c`、`.h` 或 `.i` 结尾的输入是 C 源文件；其他所有输入（目标文件、归档文件、`.lib` 文件）按其在命令行上的位置交给链接。各源文件并行编译。没有 `-o` 时，可执行文件为 `a.out`（Windows 目标为 `a.exe`），目标文件为当前目录中的 `<stem>.o`（Windows 目标为 `<stem>.obj`）。既没有源文件也没有链接输入时是错误（`no input files`）。

## 9.13.6. 代码生成

每个 C 函数都作为 C 函数降级为 nabi-mir：一个以其 C 名称、采用 C ABI 的函数体，与带 `#[export_name]` 的 `extern "C" fn` 相同（[5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-定义对外暴露)）。Nabi 的原生代码生成器编译它，在 `-O1` 及以上级别使用 nabi-mir 优化器和代码生成器的优化（[9.2.10](902-aot-pipeline.md#9210-优化级别)）。整数运算遵循 C：没有溢出检查。

调用约定是目标的 C ABI，与 Nabi 的 `extern "C"` 所用的相同（[8.6](../008-abi/806-calling-convention.md)），因此从 Nabi 调用由 `nabicc` 编译的 C，或从这些 C 调用 Nabi 的 `extern "C" fn`，在构造上就是匹配的。浮点运算不做收缩：`a * b + c` 是一次乘法和一次加法，绝不会是融合乘加，与 Nabi 代码相同。

`-g` 生成 DWARF（Linux、Apple、wasm32）或 CodeView（Windows）的行表和类型。在 Apple 目标上，链接会把 DWARF 收集到可执行文件旁边的 `.dSYM` bundle 中；在 Windows 上，链接器写出 PDB（[9.2.14](902-aot-pipeline.md#9214-链接可执行文件)）。

可执行文件由 Nabi 的链接器针对目标的 C 库链接（9.13.7）。在 ELF 目标上总是添加 `-lm`。`-lc`、`-lpthread`、`-ldl` 和 `-lrt` 指的是链接中已有的 C 库组成部分，因而被丢弃，`-nostdlib`、`-nodefaultlibs` 和 `-nostartfiles` 也一样。在 Windows 上，工具链的启动代码调用 C 的 `main(argc, argv, envp)`（不调用 `wmain`），并由链接提供 `atexit`。在 32 位 Arm 和 wasm32 上，链接从工具链为该目标提供的运行时归档文件中获取编译器辅助函数（`__aeabi_idiv`、`__multi3` 等）。

## 9.13.7. 头文件与 C 库

头文件和 C 库取决于目标：

| 目标 | 头文件 | C 库 |
|--------|---------|-----------|
| Apple | macOS SDK：`SDKROOT`，否则为 Xcode 或 Command Line Tools 的 SDK | libSystem |
| Linux，glibc | 工具链为该目标提供的 sysroot，`lib/nabi/targets/<target>/sysroot`：glibc 2.36 头文件，取自与其链接的 C 运行时相同的固定版本 Debian 包（[9.2.14](902-aot-pipeline.md#9214-链接可执行文件)） | 随附的 glibc 链接桩 |
| Linux，musl | 尚无随附的 sysroot：在 musl 主机上使用主机自己的，否则使用 `--sysroot <dir>` | 随附的 musl |
| Windows | 在 Windows 主机上，使用 Windows SDK 的 UCRT 头文件和 Visual Studio 的 C 头文件：`INCLUDE`，否则使用 `vswhere` 找到的最新 Visual Studio 和 Windows SDK | UCRT，通过工具链自己的导入库和启动代码；不链接任何 MSVC 库 |
| wasm32 | wasi-libc 的头文件，作为该目标的 sysroot 随附 | 随附的 wasi-libc |
| Android | Android NDK 的 sysroot（查找方式与链接时相同，[0.6.1](../000-meta/006-platform-targets.md#android)），使用其中该目标的目录 | bionic：NDK 中对应 API 级别的 crt 目标文件和库 |
| riscv32（无操作系统） | 只有独立环境头文件（见下） | 无 |

当目标就是主机且工具链没有为其随附 sysroot 时，使用主机的 `/usr/include`。对于以上都不具备的目标，会报错并指出缺失的目录和 `--sysroot <dir>`。`--sysroot <dir>`（或 `-isysroot <dir>`）覆盖上表：改为搜索其下的头文件（`usr/include`，在 Linux 上加上目标的 multiarch 目录；在 wasm32 上为 `include`）。

在 C 库的头文件之前是内置头文件，即代替 clang 自带头文件的独立环境头文件：`stddef.h`、`stdint.h`、`stdarg.h`、`stdbool.h`、`stdalign.h`、`stdnoreturn.h`、`stdatomic.h`、`limits.h`、`float.h`、`inttypes.h`、`iso646.h`，以及 [std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#bindgen-读取的-c) 中的内建函数头文件，其中的函数在这里没有定义。`-nobuiltininc` 不使用它们；`-nostdinc` 不使用任何系统目录。

## 9.13.8. 包的 C 源文件

包在 `nabi.nabion` 的 `c` 字段中列出其 C 源文件（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）：

```nabion
Package {
    name: "notes",
    c: {
        sources: ["vendor/sqlite3.c"],
        include: ["vendor"],
        define: { "SQLITE_THREADSAFE": "0", "NDEBUG": "" },
        std: "c17",
        flags: ["-Wall"],
    },
}
```

| 字段 | 含义 |
|-------|---------|
| `sources` | C 文件，相对于清单 |
| `include` | 头文件目录，相对于清单（`-I`）；之后是该包的 `Dep::system` 库的头文件目录 |
| `define` | 宏（`-D<name>=<value>`）；空值把宏定义为空 |
| `std` | 标准：`c89`、`c99`、`c11`、`c17`、`c23` 或某个 `gnu` 标准（默认 `gnu17`） |
| `flags` | 其他选项：只能是 `-O*`、`-g`、`-W*`、`-f*`（不含 `-flto`）和 `-m*` |

内置编译器以构建的优化级别编译每个源文件（`-O0`，使用 `--release` 时为 `-O2`，或给定的 `-O`）；`flags` 中的 `-O` 会取而代之。目标文件放在 `target/<profile>/c/<package>/` 中，只有输入（源文件本身及其读取的头文件）或选项发生变化的源文件才会重新编译。凡是链接该包的地方都会链接这些目标文件：该包的可执行文件和测试框架，以及依赖该包的其他包的可执行文件。包的 Nabi 代码通过 `extern "C"` 块（[5.9](../005-modules/509-extern-block.md)）调用这些 C 函数；`--target` 也会为该目标编译 C。

`flags` 中出现上述以外的选项时，在读取清单时报错，并列出 `flags` 接受的选项。不存在的源文件或目录是错误，并指出其名称。C 源文件编译失败会以编译器的错误使构建失败。永远不会运行外部 C 编译器，构建时也不会运行包中的任何代码（9.10.3）。
