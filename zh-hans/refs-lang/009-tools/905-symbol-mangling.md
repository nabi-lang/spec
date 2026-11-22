# 9.5. 符号修饰方案

[AOT 构建流水线](902-aot-pipeline.md)中的[特化](../000-meta/007-glossary.md#specialize)会使一个函数按具体类型拆分为多个独立的函数。本节定义为这些函数赋予唯一符号名的规则。

## 9.5.1. 形式

符号的整体语法如下：

```
mangled_symbol  ::= "_Y" pkg_hash "_" mod_path "_" item_name "_" type_hash
pkg_hash        ::= [0-9a-f]{8}            // package SHA-256 prefix
mod_path        ::= identifier ("_" identifier)*
item_name       ::= identifier
type_hash       ::= [0-9a-f]{16}           // generic args SHA-256 prefix
                  | ""                       // concrete (non-generic) item
```

写在一行上：

```
_Y <pkg_hash_8> _ <path> _ <name> _ <typehash_16>
```

一个实际的例子：

```
_Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
   |       |       |    |
   |       |       |    +-- typehash (16 chars, hash of generic args)
   |       |       +-- name (push)
   |       +-- path (std_vec)
   +-- pkg hash (8 chars)
```

## 9.5.2. 各部分的含义

### `_Y`：Nabi 符号前缀

每个 Nabi 符号都以 `_Y` 开头。在 C 中，以下划线加大写字母开头的名称保留给实现，因此它们不会与普通 C 符号冲突；选择 `_Y` 也是为了不与广泛使用的修饰方案重叠（C++ Itanium 的 `_Z`、Rust v0 的 `_R`、D 的 `_D`、Swift 的 `$s`）。它还使 Nabi 生成的符号在 `nm` 或链接器错误输出中容易辨认。前缀之后紧跟小写十六进制的 `<pkg_hash_8>`，因此仅凭第一个字符就能把它与后跟大写字母的保留前缀（`_YK` 等，[9.5.5](#955-保留前缀)）区分开。

### `<pkg_hash_8>`

[源单元](../000-meta/003-source-units.md)中的包哈希：基于内容的 SHA-256 哈希的前 8 个字符。

```
3f8a1c2b
```

内容相同的包无论在哪里构建，都会得到相同的哈希。因此，不同包中的同名函数永远不会冲突。

### `<path>`：模块路径

[路径](../005-modules/507-paths.md)中定义的模块路径，把 `::` 替换为 `_`。标准库路径保留其层级（`std::vec` → `std_vec`），用户路径去掉 `pkg::`。库的依赖方通过库的名称来指称它的项，因此在库中，路径以该名称开头（库 `app` 的 `pkg::geom::f` → `app_geom_f`）。

```
std::vec      → std_vec
pkg::geom   → geom               (pkg:: is already identified by the package hash)
foo::inner::sub → foo_inner_sub
```

### `<name>`：项名

[标识符](../001-lexical/104-identifiers.md)中定义的标识符。`<path>` 的每一段都按同样的规则转义。

由于 `_` 是分隔符，标识符中 `[A-Za-z0-9]` 以外的字符按如下方式转义。

| 源字符 | 编码 |
|---|---|
| `[A-Za-z0-9]` | 不变 |
| `_` | `_0` |
| 其他任何字符（包括 Unicode） | `_1<小写十六进制码位>_` |

标识符不以数字开头，因此 `0` / `1` 永远不会紧跟在分隔符 `_` 之后。所以转义和分隔符总是可以区分。

```
mod a { fn b }   → _Y<pkg>_a_b_
fn a__b          → _Y<pkg>__a_0_0b_
fn α             → _Y<pkg>___13b1__
```

### 方法名

方法以 `<type>.<method>` 作为名称、以定义该类型的模块作为路径进行修饰。如果多个 trait 为同一类型提供了同名方法，则追加 `__by_<trait>`。依赖包（[9.2.3](902-aot-pipeline.md#923-依赖解析)）的方法也按这一稳定路径 `<module>::<type>.<method>[__by_<trait>]` 修饰，因此分别构建的包与程序生成相同的符号。`.` 和 `_` 按上表转义。

### 嵌套函数

嵌套函数（[4.10](../004-functions/410-nested-functions.md)）在其外层项所在的模块中修饰，以 `<outer>/<name>` 作为名称，其中 `<outer>` 是外层函数的名称（方法为 `<type>.<method>`，trait 默认主体为 `<trait>.<method>`，嵌套函数则递归地为 `<outer>/<name>`）。同一个源文件中具有相同 `<outer>/<name>` 的两个嵌套函数按源代码顺序得到 `/2`、`/3`……。`/` 不出现在任何其他项名中，因此该符号与模块路径和方法都不同。`/` 按上表转义（`_12f_`）。

```
mod a { fn outer() { fn helper() } }   → _Y<pkg>_a_outer_12f_helper_
```

### `<typehash_16>`：类型元组哈希

[泛型](../002-types/210-generics.md)的具体类型元组用 SHA-256 哈希，取前 16 个字符。

```
fn id<T>(x: T) -> T

call id(42i32)   → typehash = SHA-256("(i32)")[0..16]
call id(3.14f64) → typehash = SHA-256("(f64)")[0..16]
```

### 具体 `fn` 的 typehash

对于不接受泛型实参的函数，typehash 为空，因此符号以分隔符 `_` 结尾。

```
_Y3f8a1c2b_geom_compute_            (empty typehash)
```

泛型实例总是带有 16 个字符，因此空的 typehash 可以无歧义地表明没有泛型实参。

## 9.5.3. 对比：Rust 的 `_ZN`

Rust 的 Itanium 风格修饰如下所示。

```
_ZN3std3vec10push_inner17h2d4c3e9d8f1a2b3cE
```

- 每个组成部分都带有长度前缀（`3std3vec`）。
- 包含一个 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 标识哈希（17h...）。
- 以 `E` 结尾。

它的优点是与 Itanium 兼容，但难以阅读，长度前缀的编码效率也不高。

相比之下，Nabi 的 `_Y...` 易于阅读，并且没有长度前缀（各组成部分以 `_` 分隔）。符号还原也同样简单。

## 9.5.4. 符号还原工具

把符号解码为人类可读的形式：

```
nabi demangle <symbol>
```

输出如下所示：

```
$ nabi demangle _Y3f8a1c2b_std_vec_push_a7c91d8e44b2f190
package:  3f8a1c2b
path:     std::vec
name:     push
generics: a7c91d8e44b2f190 (type tuple hash)
```

## 9.5.5. 保留前缀

已使用和保留的前缀如下：

| 前缀   | 含义                                                                        |
|----------|--------------------------------------------------------------------------------|
| `_Y...`  | 普通 Nabi 符号                                                            |
| `nabi_*`  | `extern` "C" 运行时函数（[Extern "C" 符号](../008-abi/805-extern-symbols.md)） |
| `_YK...` | （保留）关键字处理器                                                    |
| `_YL...` | （保留）字面量处理器                                                    |
| `_YI...` | （保留）中缀 / 前缀 / 后缀魔法                                      |
| `_YM...` | （保留）元块生成的代码（impl_derive 等）                    |

`_YK` / `_YL` / `_YI` / `_YM` 只保留命名空间；所有普通符号都以 `_Y` 前缀生成。

## 9.5.6. 符号长度

大多数 Nabi 符号的长度分布如下。

```
_Y + 8 + _ + path (tens) + _ + name (a few to tens) + _ + 16
≈ 30 ~ 80 characters
```

很长的路径与深层[泛型](../000-meta/007-glossary.md#generic泛型)组合时可能超过 100 个字符。调试器和工具可以毫无问题地处理这样的长度（一般建议不超过 2KB）。

## 9.5.7. 避免冲突：typehash 的稳定性

泛型实参的类型元组先以确定性方式序列化，再用 SHA-256 哈希。

```
type tuple → canonical serialization → SHA-256 → first 16 characters
```

相同的类型无论重新构建多少次，都得到相同的哈希。不同构建中的同一个特化也得到相同的符号名。nabic 为闭包和其他生成的函数取的名称来自它们所在的项（对于实例，还包括其类型实参），因此它们在每次构建中也都相同。

正是这种稳定性，使分离构建、增量构建以及依赖包之间的符号匹配成为可能。

### 规范序列化规则

类型元组的序列化是确定性的。

- 基本类型写成简短的标识符（`i32` / `f64` 等）。
- 用户类型采用 `{pkg_hash}**{path}**{name}` 的形式，再加上其泛型实参。
- 和类型的成员先规范化顺序再连接（遵循[和类型（`A | B`）](../002-types/206-sum-types.md)的集合语义）。
- `struct` 由其字段类型组成（字段名无关）。

确切的规范化算法由 `nabic` 选择。本规范只要求它是确定性的。

## 9.5.8. `extern` "C" 函数：不修饰

在 [`extern` 块](../005-modules/509-extern-block.md)中声明的外部函数原样使用该 ABI 的标准命名约定。不应用 Nabi 修饰。

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8>;     // symbol: "malloc" (as is)
}
```

在另一个方向上，把 `extern "C"` 函数对外暴露时，`#[export_name("name")]` 直接指定名称，`#[export]` 以函数自身的名称导出它（见 [`extern` 块（[FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 声明）](../005-modules/509-extern-block.md)）。这样的项是不可达函数消除的根，总会被生成（[9.2.8](902-aot-pipeline.md#928-specialize单态化)）。

```nabi
#[export_name("nabi_compute")]
pub extern "C" fn compute(x: i32, y: i32) -> i32 { x + y }
// symbol: "nabi_compute" (not mangled)
```

这两种机制实现了 Nabi 与 C 之间的集成。

## 9.5.9. main 函数的符号

[源单元](../000-meta/003-source-units.md)中的入口点函数原样使用平台的普通 main 符号。

```
main                                (Linux / macOS)
mainCRTStartup                      (Windows console)
WinMainCRTStartup + WinMain         (Windows GUI, -C subsystem=windows)
_start                              (no-OS)
__main_void                         (wasm32-wasip*; called by wasi-libc's _start)
_initialize, then main              (wasm32-unknown-unknown; reactor exports called by the host)
```

在 `wasm32-unknown-unknown` 上，模块是一个 reactor：宿主先调用 `_initialize`，再调用导出的 `main`，之后运行时保持运行（不关闭）。`#[export_name("x")]` 函数是名为 `x` 的 wasm 导出。在每个 wasm32 目标上，运行时在 `__main_void`、`main` 或某个导出调用中最先发生的那一个时启动。

在 Windows 上，Nabi 自己的启动目标文件提供 `mainCRTStartup`（控制台）或 `WinMainCRTStartup` 和 `WinMain`（GUI），它们调用 nabic 生成的包装函数 `nabi_main`；Windows 可执行文件没有 `main` 符号。在调试信息（DWARF 和 CodeView）中，包装函数在每个目标上都是 `nabi_main`，在由 C 库调用它的地方其符号仍为 `main`，因此在栈回溯中只有程序自己的 `main` 名为 `main`。

平台入口点在 nabic 生成的包装函数中调用用户定义的 main 函数。包装函数依次调用 `nabi_runtime_init` 和类型注册函数 `nabi.register_types`（[8.2.13](../008-abi/802-scan-table.md#8213-注册时机)），然后调用入口函数，按 [4.16.4](../004-functions/416-entry-point.md#4164-退出码与未处理的异常) 的规则确定退出码，并经由 `nabi_runtime_shutdown` 返回。

当 `main.nabi` 由顶层语句组成时（[4.16.2](../004-functions/416-entry-point.md#4162-顶层语句的含义隐式-main)），编译器合成的隐式 `main` 就是入口函数。隐式 `main` 按与普通函数相同的规则修饰，但用户代码不能按名称引用它；只有包装函数调用它。

## 9.5.10. 调试信息

每个修饰后的符号都在 DWARF 和 PDB 调试信息中带有其原始名称。

```
debugger output.
  fn distance<f64> at geom.nabi:42
    (mangled: _Y3f8a1c2b_geom_distance_a7c91d8e44b2f190)
```

这使 gdb、lldb 和 VS Code 等工具能以原始名称显示 Nabi 函数。协程的恢复函数名为 `<creator>::{coroutine#k}`，其中 `k` 对该创建者的闭包和协程一起计数。

## 9.5.11. 稳定性

符号修饰方案也是 ABI 的一部分。因此，变更必须遵循[概述](../008-abi/README.md)中的稳定性策略。

- 1.0 之前，该方案可以在每个 0.x.0 版本中改变，在一个 0.x 系列内保持不变。
- 从 1.0 起，在同一主版本号内保持兼容；跨主版本号时不兼容，ABI 哈希也随之改变。
