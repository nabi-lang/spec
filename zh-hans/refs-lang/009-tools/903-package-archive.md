# 9.3. 包归档文件（.a）格式

库包 `foo` 的构建输出是两个文件（[9.2.15](902-aot-pipeline.md#9215-output)）。

| 文件        | 内容                                                                      |
|-------------|-------------------------------------------------------------------------------|
| `libfoo.a`  | 一个标准 `ar` 归档文件，存放该包的目标文件，每个代码生成单元一个（[9.2.16](902-aot-pipeline.md#9216-incremental-build)） |
| `foo.nabihir`  | 下游用于单态化和名称解析的接口（[9.4](904-nabihir-bytecode.md)） |

这两个文件位于同一目录中。`--extern foo=libfoo.a` 和 `--extern foo=foo.nabihir` 都会找到另一个文件（[9.2.3](902-aot-pipeline.md#923-dependency-resolution)）。版本和生成者等元数据位于 NABIHIR 头部（[9.4](904-nabihir-bytecode.md)）。

## 9.3.1. 标准 ar 归档文件

`libfoo.a` 是标准的 Unix `ar` 归档文件。`ar`、`ranlib`、`nm` 和 C 链接器无需转换即可处理它。

```
ar t libfoo.a
part0.o
part1.o
...
```

## 9.3.2. 目标文件

每个目标文件都是由 [AOT 构建流水线](902-aot-pipeline.md)为一个代码生成单元生成的标准 ELF / Mach-O / COFF 目标文件。它们合起来只包含该包中从包的根可达的那些[具体 fn](../000-meta/007-glossary.md#concrete-fn--generic-fn)（[9.2.8](902-aot-pipeline.md#928-specialize-monomorphization)）。

| 节（section）   | 内容                                                    |
|-----------|-------------------------------------------------------------|
| `.text`   | 具体 `fn` 的代码                                      |
| `.rodata` | `const`、字面量、[ScanTable](../008-abi/802-scan-table.md) |
| `.data`   | `static` 的初始值                                 |
| `.bss`    | 零初始化的 `static`                                  |

ScanTable 是普通的常量数据；生成的注册代码在进程启动时通过 `nabi_gc_register_type` 向运行时注册它们（[8.2.13](../008-abi/802-scan-table.md#8213-registration-time)）。没有 nabi 专有的节。

## 9.3.3. 静态库与共享库

库的输出是静态 `.a`；依赖从归档文件链接进程序，程序可达的标准库代码与它一起编译（[9.3.6](#936-standard-library)）。`--crate-type=cdylib`（[9.1.2](901-nabic-cli.md#nabic-build-aot-build)）还会把库包链接成一个共享库，由 C 代码（或 Python、R、Julia）在运行时加载：`lib<name>.so`（Linux）、`lib<name>.dylib`（macOS，install name 为 `@rpath/lib<name>.dylib`），或 `<name>.dll` 加上其导入库 `<name>.lib`（Windows）。它包含该包、标准库和运行时，只导出该包的 `#[export] extern "C" fn` 和 `#[export_name("sym")] extern "C" fn`（[5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-definitions-exposed-externally)），名称为它们自己的名称或 `sym`；其他所有符号都在库内部绑定。

- 加载该库会启动其运行时：加载器运行 `nabi_library_init`（`DT_INIT`、一个 `__mod_init_func` 条目，或在进程附加时运行 DLL 入口点 `nabi_dll_main`），它在加载线程上建立运行时，注册库的 GC 类型（[8.2.13](../008-abi/802-scan-table.md#8213-registration-time)），并把该线程作为处于 C 代码中的线程挂起。在任何线程上调用导出函数时，都会先附加该线程（[8.5](../008-abi/805-extern-symbols.md)），因此 C 代码不需要调用初始化函数。
- 每个共享库都有自己的运行时和堆：GC 对象不会从一个 Nabi 库跨到另一个 Nabi 库，也不会跨到加载它的 Nabi 程序。只有 C 值会跨越导出函数。
- 在 Linux 上，库的线程局部变量使用 initial-exec 模型（`DF_STATIC_TLS`）：glibc 把它们放在其静态 TLS 区域中，该区域也为用 `dlopen` 打开的库留有空间。一个库占用其中 160 字节，即运行时的线程状态（[8.3](../008-abi/803-tls-slots.md)）；glibc 为之后打开的库保留 512 字节（`glibc.rtld.optional_static_tls`），因此默认情况下一个进程可以容纳三个 Nabi 库。更多的库会加载失败，报告 `cannot allocate memory in static TLS block`；`GLIBC_TUNABLES=glibc.rtld.optional_static_tls=4096` 可以腾出空间。
- 版本：`--lib-version <x.y.z>`（[9.1.2](901-nabic-cli.md#nabic-build-aot-build)），`nabi build` 从清单的 `version` 取得它；它把 ELF soname 设为 `lib<name>.so.<x>`（低于 1.0 时为 `lib<name>.so.0.<y>`，此时次版本号的变化即为破坏性变更），并在文件旁边创建一个同名的链接；把 Mach-O 的当前版本和兼容版本设为 `x.y.z` 和 `x.0.0`（低于 1.0 时为 `0.y.0`）。
- C 头文件：构建还会写出 `target/<profile>/include/<name>.h`，与 `--emit=c-header` 相同。它以 `#[export_name]` 给出的名称声明该包中每个带有 `#[export_name]` 的 `extern "C"` 函数，以及它们的签名所涉及的类型：`#[repr(C)]` 结构体或联合体写成按顺序列出字段的 `typedef struct`（`align(N)` 用 `_Alignas`，`packed` 用 `#pragma pack`），`#[repr(transparent)]` 结构体写成其字段的 `typedef`，`#[repr(iN)]` 枚举写成其标签类型的 `typedef`，每个变体一条 `#define <ENUM>_<VARIANT> ((<Enum>)<value>)`。基本类型映射到 `<stdint.h>` / `<stddef.h>` / `<stdbool.h>` 中的类型（`usize` 为 `size_t`，`isize` 为 `ptrdiff_t`，`c_char` 为 `char`，`char` 为 `uint32_t`，C ABI 的 `str` 为 `const char *`），`ConstPtr<T>` 映射为 `const T *`，`Ptr<T>` / `ManyPtr<T>` 映射为 `T *`，`CPtr<T>` 映射为带 `/* nullable */` 注释的 `T *`，`T | none` 也是如此；`extern "C" fn` 类型映射为函数指针，`extern "system"` 函数带 `NABI_SYSTEM`（在 32 位 Windows 上为 `__stdcall`），永不返回的函数带 `_Noreturn`。每个项的 `///` 文档注释变成 C 注释。类型无法在 C 中表示的函数（泛型结构体、没有 `#[repr(C)]` 的结构体、带数据的枚举）会被省略，并留下一行 `/* skipped <name>: <why> */`。头文件带有 `#pragma once`、include 保护和供 C++ 使用的 `extern "C"`。
- 目标：`aarch64-apple-darwin`、`x86_64-unknown-linux-gnu`、`aarch64-unknown-linux-gnu`、`x86_64-pc-windows-msvc` 和 `aarch64-pc-windows-msvc`，使用 Nabi 自己的链接器（不是 `NABI_CC`）。其他目标是错误，错误信息会建议改用 C 工具链链接静态库。
- 对其他共享库的 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 调用在每个目标上都可以工作。

`--crate-type=staticlib` 则生成一个 C、Swift 和 Xcode 可以单独链接的静态库：`target/<profile>/static/lib<name>.a`（`-o` 指定它）包含该包、其依赖、它可达的标准库以及运行时，每个归档文件的成员都重命名为 `<n>-<member>`，使它们互不遮挡。运行时在第一次调用导出函数时启动，并像共享库的导出函数那样附加其线程；GC 对象同样不会跨越导出函数。构建还会写出 C 头文件（同上）和 `target/<profile>/include/module.modulemap`，这是一个以包命名的 clang 模块，链接该库以及包所指定的系统库和 framework，因此 Swift 中写 `import <name>`：

```
module geo {
    header "geo.h"
    link "geo"
    link framework "UIKit"
    export *
}
```

使用 `--xcframework` 时，构建还会把库及其头文件放入 `target/<profile>/<name>.xcframework`，即一个与 `xcodebuild -create-xcframework` 所写格式相同的 XCFramework：每个已构建的 Apple 目标一个切片（`ios-arm64`、`ios-arm64-simulator`、`macos-arm64`），每个切片含有 `lib<name>.a` 和 `Headers/`，以及一个列出它们的 `Info.plist`。每次构建都会添加或替换其目标的切片，因此一次为 `aarch64-apple-ios` 的构建和一次为 `aarch64-apple-ios-sim` 的构建会生成一个 Xcode 对两者都接受的 XCFramework；设备目标仍在进行中（[0.6.2](../000-meta/006-platform-targets.md#062-in-progress)），所以目前 XCFramework 只包含模拟器的切片。不带 `--crate-type=staticlib` 使用 `--xcframework`，或用于非 Apple 目标，都是错误，错误信息会说明应当写什么。

iOS 应用正是这样使用 Nabi 代码的（[0.6.1](../000-meta/006-platform-targets.md#061-supported)）：一个 `aarch64-apple-ios-sim` 的归档文件（设备目标得到支持后，再加一个 `aarch64-apple-ios` 的归档文件），加入 Xcode 项目，并把 `include` 目录加入其头文件搜索路径。在 wasm 目标上，`--crate-type=staticlib` 是错误。

## 9.3.4. 版本兼容性检查

读取依赖的 NABIHIR 时，会把它的 `version` 和生成者（生成该 NABIHIR 的 `nabic`）与当前的 `nabic` 比较。如果不同，则是错误，错误信息会说明需要重新构建什么（[9.4.9](904-nabihir-bytecode.md#949-reading-and-rejection)）。

## 9.3.5. 符号暴露

归档文件暴露的符号是其目标文件的全局符号，遵循[符号修饰](905-symbol-mangling.md)。

```
nm libfoo.a
0000000000000000 T _Y3f8a1c2b_foo_geom_distance_a7c91d8e44b2f190
0000000000000040 T _Y3f8a1c2b_foo_geom_Point_origin_b1234567...
```

## 9.3.6. 标准库

标准库（`bare` / `core` / `std`）不以归档文件形式分发，任何构建都不链接预构建的标准库。每次构建都从源代码（`--bootstrap`）与包一起编译它，而且只编译程序可达的部分：前端从程序的根（`main`、测试框架中的 `#[test]` 和 `#[bench]` 函数、`#[export_name]` 和 WIT 导出、编译器和运行时调用的函数，以及每个静态变量的初始化式）出发，沿着调用、函数值、所用类型的 `Drop` 实现以及 `any Trait` 值分派到的实现，对遇到的每个函数进行类型检查和降级（[9.2.6](902-aot-pipeline.md#926-type-checking)）。对其余部分，它只读取名称和签名，用于名称解析、方法查找和诊断。`--check-library` 则检查并编译每个库主体（[9.1.2](901-nabic-cli.md#nabic-build-aot-build)）。

程序可达的标准库函数按代码生成单元编译，每个模块一个单元（9.2.16）。每个函数优化后的代码和每个单元的目标文件，以覆盖其全部依赖的键保存在工具链的函数缓存中（`lib/nabi-cache/fn/`，[9.2.19](902-aot-pipeline.md#9219-installation-layout)），此后该工具链的每次构建只要键相同，就从那里取用，无论构建的是哪个包。优化器仍然看到整个程序，包括标准库的函数，因此会像内联包自身的函数一样内联它们（9.2.10）。

库包的归档文件含有它可达的标准库代码的私有副本，与其泛型实例相同。标准库的静态变量是例外：库以它们自己的符号把它们定义为弱符号，因此程序和链接进程序的每个库共享每个静态变量的同一份。

## 9.3.7. 链接

```
cc main.o libfoo.a libnabir.a -o my_binary
```

链接器只从归档文件的目标文件中链接需要的符号。以 `#[export_name]` 暴露的函数可以直接从 C 调用。

```c
extern uint32_t nabi_compute(int32_t a, int32_t b);   // #[export_name("nabi_compute")]
int main(void) { return (int)nabi_compute(1, 2); }
```

```
cc main.c libfoo.a libnabir.a -o my_binary
```

使用 `--crate-type=staticlib`（9.3.3）时，一个归档文件就够了：

```
cc main.c target/release/static/libfoo.a -o my_binary
```

## 9.3.8. 稳定性

归档文件格式遵循 [ABI 稳定性策略](../008-abi/README.md)；NABIHIR 格式遵循 [9.4](904-nabihir-bytecode.md)。
