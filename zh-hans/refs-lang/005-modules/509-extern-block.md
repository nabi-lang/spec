# 5.9. `extern` 块（[FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 声明）

`extern` 关键字（一个上下文关键字，只在项的开头才是关键字，[1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)）与 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 说明符（`"C"` 等）结合，定义一个声明外部函数和全局变量的块。它用于与外部库、系统调用和其他语言交互。

```
extern_block    ::= "extern" string_lit "{" extern_item* "}"
extern_item     ::= visibility? "fn" identifier "(" param_list? ")" return_type? ";"
                  | visibility? "static" "mut"? identifier ":" type_expr ";"
```

## 5.9.1. 基本形式

基本用法：

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
    fn strlen(s: ConstPtr<c_char>) -> usize;

    static environ: Ptr<Ptr<c_char>>;
    static mut atexit_count: u64;
}
```

块由 `extern`、ABI 字符串（`"C"`、`"system"` 等；各 ABI 的含义在[调用约定](../008-abi/806-calling-convention.md)中定义）和 `{ ... }` 组成。主体只能包含带签名而无函数体的函数声明，以及 `static` 声明。

> [!IMPORTANT]
> `extern "C"` 块内的函数声明**隐式为不安全的**。调用方只能在 `unsafe { ... }` 块内调用它们；即使签名中不写 `unsafe`，效果也相同。完整定义请见 [4.15 unsafe](../004-functions/415-unsafe.md)。

## 5.9.2. ABI 说明符

支持的 ABI：

| ABI        | 含义                                                                          |
|------------|----------------------------------------------------------------------------------|
| `"C"`      | 平台的 C ABI。无论符号名如何，始终相同。                   |
| `"system"` | 操作系统的系统调用 ABI。在某些平台上与 `"C"` 不同（如 Win32 `stdcall`）。 |
| `"nabi"`    | 运行时（`nabir`）的 ABI。见下文。                                      |
| `"objc"`   | Apple 目标上的 Objective-C 类及其方法（[stdlib 1.10](../../refs-standard-libraries/100-bare/110-objc.md)）；`ffi::objc::import { "AppKit/AppKit.h" }` 根据 SDK 的头文件生成这些块（[stdlib 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-headers-ffiobjcimport)）。 |

在 `"system"` 与 `"C"` 一致的目标上（除 32 位 x86 Windows 之外的所有受支持目标），`"system"` 就是 `"C"`。在 32 位 x86 Windows 上，它是 stdcall（[5.9.10](#5910-extern-fn-definitions-exposed-externally)）。

`extern "nabi"` 是工具链所声明的运行时函数的 ABI；这些声明是工具链各包的私有项（大部分在 `core::rt` 中；`bare` 声明它自己使用的那些）。`extern "nabi"` 块上的 `#[no_safepoint]` 声明其函数既不分配也不阻塞；对它们的调用不是安全点。它是工具链内部使用的，用在 `extern "C"` 块上是错误。

| 值                           | 在 `extern "nabi"` 下的传递方式             |
|---------------------------------|------------------------------------------|
| `str`                           | 两个字 `(ptr, len)`                   |
| `fn` 值                      | `{code, env}`（[8.5.0a](../008-abi/805-extern-symbols.md#850a-nabifn---c-abi-form-of-function-values)） |
| `extern "C" fn(..)` 参数   | 不带 env 的 C 函数指针           |
| 其他一切                   | 值本身（与 C ABI 相同）      |

> [!NOTE]
> `"Rust"` 等其他语言的 ABI 通常不稳定，不是 `extern` 的目标。只有通过双方约定的稳定 ABI（通常是 `"C"`）才能与其他语言交互。

> [!NOTE]
> Nabi 以返回值传播错误，不使用栈展开（[8.6.5](../008-abi/806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)）。因此，外部语言的异常或栈展开穿过 Nabi 栈帧是未定义行为；外部函数必须在调用边界内处理异常。

不带 ABI 的单纯 `extern { ... }` 是编译错误；必须指定 ABI。

## 5.9.3. 主体中允许的项

### 外部函数声明

外部函数的声明方式如下：

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
}
```

签名的形式与普通函数声明相同，但没有函数体，以 `;` 结尾。参数和返回类型只能使用 FFI 兼容的类型，如原始整数、浮点数和指针。

### 外部 `static`

在外部定义的全局变量按如下方式引用：

```nabi
extern "C" {
    static environ: Ptr<Ptr<c_char>>;  // read-only
    static mut atexit_count: u64;      // mutable
}
```

这一形式引用在外部定义的全局变量；`static mut` 只允许出现在 `extern` 块内（[`static`](503-static.md)）。C 的 `errno` 不是这样的变量：它是一个按线程区分的宏，所以 `static errno` 会读到错误的内存。在声明上加 `#[errno]` 会在调用之后保存它（[stdlib 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errno---errno-and-getlasterror)）。

### 不允许的内容

`struct`、`enum`、`impl`、`fn` 函数体、`const` 等不出现在 `extern` 块的主体中。该块只声明与外部 ABI 之间的边界。

## 5.9.4. FFI 兼容类型

参数、返回值和 `static` 的类型都必须是 FFI 兼容的。FFI 类型如下：

- 原始整数：`i8` 到 `i64`、`u8` 到 `u64`、`usize`、`isize`。
- C 的具名类型，即这些原始类型的别名，其大小与目标的 C 编译器所定的大小一致：`c_int`、`c_uint`、`c_long`、`c_ulong`、`c_size_t`、`c_wchar` 以及 `bare::ffi::types` 中的其余类型，全部在 prelude 中（[stdlib 1.7.9](../../refs-standard-libraries/100-bare/107-ffi.md#179-c-types---c_int-c_long-c_void)）。`c_long` 在 Windows 和 32 位目标上是 32 位，所以 C 的 `long` 要声明为 `c_long`，而不是 `i64`。`c_void` 表示指针背后的 `void`（`CPtr<c_void>`）。
- 原始浮点数：`f32`、`f64`。
- `bool`（C 的 `bool`）和 `c_char`（1 字节）。
- `f16` / `bf16`：C 的 `_Float16` / `__bf16`，仅在其 C 具有这些类型的目标上（[8.6.3](../008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)）。
- 8 或 16 字节的 `Simd<T, N>`，作为参数或结果：在 AAPCS64 和 x86-64 上对应目标的同等大小的 C 向量（`float32x4_t`、`__m128`）（[8.6.3](../008-abi/806-calling-convention.md#863-struct-arguments--return-small-struct-rule)）。
- 指针：`Ptr<T>`、`ConstPtr<T>`、`CPtr<T>`（`bare::ptr`，[标准库层级](../000-meta/005-standard-libraries-tiers.md)中的最低层）。
- 字段均为 FFI 类型（包括由它们构成的定长数组）的 `struct` 和 `union` 类型。默认布局已经是 C 的布局；`#[repr(packed)]` 和 `#[repr(align(N))]` 按与 C 相同的方式改变布局。
- 带整数 `#[repr]`（`#[repr(i32)]`）的无字段枚举：即 C 的 `enum`。
- 函数指针：`extern "C" fn(...) -> ...`（请见下文的“C 函数指针类型”）。
- 函数类型的参数：`extern "C"` 声明中的 `fn(..) -> R` 参数与 `extern "C" fn(..) -> R` 相同。实参必须指名一个函数；闭包是错误（`` `each` is a C function: a `fn` argument must name a function (a C callback has no room for a closure's captures) ``），其帮助信息说明状态应放在哪里：`` a C callback cannot capture: keep the state in `Handle::new(..)` (`bare::ffi`), pass `h.as_ptr()` as the `void*` argument, and read it back in an `extern "C" fn` with `Handle::<T>::from_ptr(p).get()` ``（[stdlib 1.7.10](../../refs-standard-libraries/100-bare/107-ffi.md#1710-handlet---nabi-values-c-holds)）。
- 可空指针：`Ptr<T> | none`、`ConstPtr<T> | none`、`extern "C" fn(..) -> R | none`（带空值 niche 的指针，[2.7.2](../002-types/207-niche-optimization.md#272-automatic-niches)），以及任何 `#[niche(0)]` 结构体 `| none`（`NonZero<i32> | none` 是一个 `int`）；`none` 即 null。
- 作为 `extern "C"` 声明参数的 `str`：按下面的规则转换。
- 作为结果的 `()` 和 `!`。

`extern "C"` 声明中的 `str` 参数是一个以 NUL 结尾的 `char*`。如果字节之后紧跟 NUL，字节按原样传递；否则（切片等）传递一个在调用期间被固定的副本。调用返回后，C 不得保留该指针。`str` 结果是错误：返回的 `char*` 的字节属于 C（可能是下次调用会覆盖的静态缓冲区、调用方必须释放的内存、指向实参副本且随调用一起失效的指针），所以基于它们的 `str` 可能改变或悬空，而且这些字节不一定是 UTF-8。应把结果声明为 `ConstPtr<c_char> | none`，并用 `CStr::from_ptr`（[stdlib 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md#172-cstr--cstring---null-terminated-string)）复制字符串，它还会说明字符串是否为 UTF-8：

```nabi
extern "C" {
    fn getenv(name: str) -> ConstPtr<c_char> | none;
}
let home = match unsafe { getenv("HOME") } {
    p: ConstPtr<c_char> => unsafe { CStr::from_ptr(p) },
    none => none,
};
```

模块 `m` 中 `extern` 块的 `pub` 函数也可以通过 `m::f` 访问；其符号不带前缀。`extern "C"` 块中的 `nabi_*` 名称不得接收或返回 `str`（[8.5](../008-abi/805-extern-symbols.md)）。

在 `extern "C"` 和 `extern "system"` 块的参数、结果和 `static` 中，以及在 `extern "C" fn` 定义中，其他任何类型都是错误。检查针对解析后的类型进行，所以会看穿别名（`type S = str;`）、泛型实例、字段或数组元素。每个错误都会说明应改写成什么：

| 类型 | 原因 | 改写为 |
|------|-----|---------------|
| GC 类型（`List`、`Dict`、闭包、`any`、包含它们的结构体） | 在 C 持有期间，回收器可能移动或释放它 | 原始类型、`CPtr<T>` 加长度，或由它们构成的结构体；用 `gc::pin` 固定的 `List<T>`，以 `p.as_data_ptr()` 和 `xs.len()` 传递 |
| 作为结果的 `str`，或 `extern "C" fn` 定义中的 `str` | 字节归 C 所有（见上文）；C 传递的是 `char*` | `ConstPtr<c_char> \| none`，用 `CStr::from_ptr` 复制 |
| `char` | C 没有四字节字符 | `u32` 和 `char::from_u32` |
| 元组 | C 没有元组 | `struct` |
| 普通枚举 | 其布局由编译器决定 | `#[repr(i32)]`，或由程序匹配的 `i32` |
| 带数据的枚举 | C 没有带标签的枚举 | 由标签和 `union` 组成的结构体 |
| 数组参数或结果 | C 以指针传递数组 | `ConstPtr<T>`（`addr_of(xs)`），或放在结构体中的数组 |
| 其他和类型（`i32 \| none`） | 只有空指针有 C 形式 | `ConstPtr<T> \| none`，或值加一个标志 |
| 声明参数以外的 nabi `fn` 值 | 它是一个函数加上它的捕获 | `extern "C" fn(..)`，由具名的 `extern "C" fn` 构造 |
| `Mask<T, N>` | 没有对应的 C 类型 | 整数的 `Simd`（`mask.select(Simd::splat(1), Simd::splat(0))`） |
| 包含 `Simd` 的结构体，按值传递 | 聚合体中的 C 向量无法传递 | 指向该结构体的指针 |

消息会指出参数和类型：`` parameter `xs` of extern function `f` has type `List<i64>`, which C cannot hold: the collector may move or free it ``。对于 `List<T>`，帮助信息就是应写的调用：`` help: C takes a list as a pointer and a length: declare `ptr: ConstPtr<i64>, len: usize`, and pass `p.as_data_ptr()` and `xs.len()` from `let p = gc::pin(xs);` (`bare::gc`) ``。当目标的 C 没有 `_Float16`、没有 `__bf16` 或没有该大小的向量时，错误来自该目标的代码生成，并附有同类帮助信息。编译器无法检查它看不到的内容：确保 C 一侧以相同的类型声明同一个函数，仍是程序的责任。每种类型的调用约定在[调用约定](../008-abi/806-calling-convention.md)中定义。

### C 函数指针类型

```
c_fn_type ::= "unsafe"? "extern" ("\"C\"" | "\"system\"") "fn" "(" (type_expr ("," type_expr)*)? ")" ("->" unary_type)?
```

- ABI 字符串只能是 `"C"` 或 `"system"`。其他任何字符串都是错误。
- `extern "system" fn(..) -> R` 遵循与 `extern "C" fn(..) -> R` 相同的规则，因此可以表达 Win32 回调。只有在 32 位 x86 Windows 上它才是一个不同的类型（stdcall）；在其他地方两者是同一类型。
- 这种类型的值只能由具名的 C ABI 函数构造。`extern` 块中 C 声明的值具有类型 `unsafe extern "C" fn(..)`，不能放在安全的（非 `unsafe` 的）位置；Nabi 的 `extern "C" fn` 定义（其 C 入口点，5.9.10）产生安全的 `extern "C" fn(..)` 值。在该位置使用普通 `fn` 是错误，`` only an `extern "C"` function (an `extern` block declaration or an `extern "C" fn` definition) is an `extern "C" fn(..)` value ``，帮助信息为 `` C calls it with the C ABI: declare it `extern "C" fn twice(..)` ``。
- Nabi 的 `fn` 值（包括闭包和函数类型的变量）与 C 函数指针不能互相转换。
- 通过不带 `unsafe` 的类型调用是安全的；通过 `unsafe extern "C" fn` 调用需要 `unsafe`。
- 非 `none` 的 C 函数指针值不是 null。因此，在不带 `| none` 的 `extern "C" fn` 位置（参数、返回值、字段）收到 NULL 是未定义行为，可能返回 NULL 的 C API 必须用 `| none` 声明。

#### 可空的 C 函数指针：`extern "C" fn(..) -> R | none`

只由一个 C 函数指针类型和 `none` 构成的和类型（顺序任意，无其他成员）使用空值 niche（[2.7.2](../002-types/207-niche-optimization.md#272-automatic-niches)）。

- 它的大小与指针相同，`none` 即 null。
- 它的 ABI 与 C 的可空函数指针相同。可以原样用于 extern 参数和返回值、与 C 交换的结构体字段，以及 extern `static`。
- 以零填充的存储读出为 `none`。
- 包含它的更大的和类型使用普通的带标签布局，两者之间的转换是普通的和类型转换。

```nabi
extern "C" {
    fn cmp_i32(a: ConstPtr<u8>, b: ConstPtr<u8>) -> i32;
    fn qsort(base: Ptr<u8>, n: usize, size: usize, cmp: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32);
}
let f: unsafe extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;   // a C declaration's value is unsafe
let g: extern "C" fn(ConstPtr<u8>, ConstPtr<u8>) -> i32 = cmp_i32;          // ERROR: safe position
```

### extern 函数的名称空间

在 `extern` 块中声明的函数构成单一的全局名称空间，可以从任何模块以不带前缀的名称调用。这对应于 C 符号在链接时位于单一全局空间的情况。它是普通项“不隐式查找祖先模块”规则（[5.7.1](507-paths.md#571-starting-identifier-of-a-path)）的一个例外，只适用于 extern 函数。以不同签名两次声明同名的 extern 函数是编译错误。工具链在 `core::rt` 中的运行时声明也算在内：重新声明某个 `nabi_*` 运行时函数的用户 `extern` 必须与其签名一致。

## 5.9.5. `Ptr<T>` / `ConstPtr<T>`

在 `bare::ptr`（[标准库层级](../000-meta/005-standard-libraries-tiers.md)中的最低层）中定义的裸指针类型。它们不在 prelude 中：`use bare::ptr::{Ptr, ConstPtr};`。

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
    fn read_byte(p: ConstPtr<u8>) -> u8;
}
```

`Ptr<T>` 是对应 C 的 `T*` 的可变裸指针，`ConstPtr<T>` 是对应 C 的 `const T*` 的只读裸指针。两者都永远不为 null。可能为 null 的指针是 `Ptr<T> | none`（或 `ConstPtr<T> | none`）：一个地址，null 表示 `none`（[niche 优化](../002-types/207-niche-optimization.md)）。`CPtr<T>` 是可以把 NULL 作为值持有的 C 指针（`is_null()`、`as_ptr() -> Ptr<T> | none`）。

## 5.9.6. 调用与使用

在 `extern` 块中声明的函数像普通 Nabi 函数一样调用。

```nabi
extern "C" {
    fn malloc(size: usize) -> Ptr<u8> | none;
    fn free(p: Ptr<u8>);
}

fn use_malloc() {
    let p = unsafe { malloc(1024usize) };
    if p is Ptr<u8> {
        // ... use ...
        unsafe { free(p) };
    }
}
```

每个 `extern` 函数（包括 `extern "nabi"`）都是不安全的，要在 `unsafe { ... }` 内调用（[4.15](../004-functions/415-unsafe.md)）；工具链的各包把它们声明的运行时函数包装在安全函数中。除此之外，`extern` 函数调用与普通调用的区别仅在于它是外部符号。Nabi 的普通语义适用于结果类型。`extern static` 像普通变量一样读取，并且（如果是 `mut`）像普通变量一样写入。Nabi 无法检查 C 的全局变量，所以读写它需要 `unsafe`（`unsafe` 块或 `unsafe fn` 主体）；在其外部是错误，帮助信息引用修正写法：`unsafe { optind = 6; }`。用 `addr_of` 取其地址不读取任何内容，不需要 `unsafe`。它的类型仍然必须是 FFI 兼容的（5.9.4）：Nabi 会把外部内存当作该类型读取，所以 `static environ: List<str>;` 是未定义行为；应把它声明为指针（`static environ: Ptr<Ptr<c_char>>;`）。

## 5.9.7. 可见性

`extern` 块的每个项都可以带可见性关键字。

```nabi
extern "C" {
    pub fn public_api(x: i32) -> i32;       // public
        fn _internal(x: i32) -> i32;        // module-private
    pub static VERSION: i32;
}
```

`extern "C"` 块本身不带可见性关键字；可见性按项指定。

## 5.9.8. `extern` 块本身的位置

`extern "C" { ... }` 块只能出现在模块级。它不能放在函数或 `impl` 内。

```nabi
fn f() {
    extern "C" { ... }                     // ERROR: not inside a function
}
```

## 5.9.9. `#[link("libname")]`：指名库

要指名外部库，使用 `#[link]` 属性。

```nabi
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
    fn sqrt(x: f64) -> f64;
}
```

extern 块上的 `#[link("x")]` 或 `#[link(name = "x", kind = "static" | "dylib")]` 向链接器传递 `-lx`。库搜索路径依次是 `-L <dir>` 目录，然后是 `LIBRARY_PATH`。在 Apple 目标上，`kind = "static"` 按路径链接在那里找到的 `lib<name>.a`，`#[link(name = "Foundation", kind = "framework")]` 链接 framework `Foundation.framework`，依次在 `-F <dir>` 目录和 SDK 的 `System/Library/Frameworks` 中搜索；在其他目标上，framework 是链接错误。nabic 的 `-l` 和原生包的 `links` 以同样的方式添加库，`nabi.nabion` 中的 `Dep::system` / `Dep::framework` 库也是如此（[9.10.4b](../009-tools/910-nabi-frontend.md#9104b-native-packages)）。在 wasm32 上，`#[link(wasm_import_module = "m")]` 从 core 模块 `m` 导入该块的函数，函数上的 `#[link_name = "sym"]` 设置其链接符号或导入名（[6.2.3](../006-meta-system/602-built-in-attributes.md#623-link-naming-an-external-library)）。没有 `#[link]` 时，符号在链接时通过系统默认库（`libc` 等）或其他机制解析，例如包自身 C 源文件的目标文件，内置 C 编译器会编译它们并与包一起链接（[9.13.8](../009-tools/913-c-compiler.md#9138-c-sources-of-a-package)）；它们的函数与其他函数一样在 `extern "C"` 块中声明。`#[link]` 的行为在[内置属性](../006-meta-system/602-built-in-attributes.md)中定义。

## 5.9.10. `extern fn` 定义（对外暴露）

前面各节的形式是从 nabi 调用外部函数。反方向的形式通过外部 ABI 暴露 Nabi 函数。

```nabi
#[export]
pub extern "C" fn nabi_compute(x: c_int, y: c_int) -> c_int {
    x + y
}

#[export_name("nabi_area")]
pub extern "C" fn area(w: c_int, h: c_int) -> c_int { w * h }

extern "C" fn callback(arg: i32) -> i32 {
    arg * 2
}
```

普通函数声明上的 `extern "ABI"` 限定符表示该函数要按与该 ABI 兼容的方式编译。`extern "C" fn` 定义保留其经过修饰的符号名（[9.5](../009-tools/905-symbol-mangling.md)）；要得到 C 可以链接的名称，加上 `#[export]`，它以函数自身的名称（`nabi_compute`）导出，或者加上 `#[export_name("sym")]`，它以 `sym` 导出（`area` 导出为 `nabi_area`）。导出的函数是消除不可达函数时的根，共享库（`--crate-type=cdylib`，[9.3](../009-tools/903-package-archive.md)）对外可见的正是这些导出。

`extern "ABI" fn` 定义接受 `"C"` 或 `"system"`；其他任何 ABI 字符串都是错误。在 32 位 x86 Windows 上，`extern "system"` 声明或定义是 stdcall：由被调用方弹出实参，符号与 MSVC 一样为 `_name@N`。

`extern "C"` 和 `extern "system"` fn 定义有以下限制：

- 不能是泛型、`async` 或 `throws`。
- 参数和返回类型必须是 5.9.4 中的 FFI 兼容类型；不能使用 `str` 等非 FFI 类型。（5.9.4 中的 `str` 转换只适用于*调用* C 函数的声明。）

`#[export]` 和 `#[export_name]` 只能用在 `extern "C"` 和 `extern "system"` fn 定义上。C 以 C 约定调用导出函数，而 Nabi ABI 的函数会弄错其聚合体、`str` 和窄整数，所以在普通函数上使用其中任何一个都是错误：`` `compute` is exported to C but uses the Nabi ABI: write `pub extern "C" fn compute(..)` ``。在 wasm32 上，导出是该名称的 wasm 导出。

```nabi
#[export_name("on_event")]
extern "C" fn on_event(code: i32) -> i32 { code }     // visible from C as on_event
extern "C" fn bad(s: str) { }                          // ERROR: str cannot be used
#[export]
pub fn compute(x: i64) -> i64 { x * 2 }               // ERROR: exported, but the Nabi ABI
```

### `#[on_panic]`：给 C 一个值，而不是中止

```nabi
#[on_panic(return = 0)]
extern "C" fn cmp(a: Ptr<i32>, b: Ptr<i32>) -> c_int { .. }

#[on_panic(return = -1.5)]
extern "C" fn half(x: f64) -> f64 { .. }

#[on_panic]
extern "C" fn notify(x: i32) { .. }
```

一个会穿过 C 的栈帧离开 C 所调用函数的 panic 会中止进程（5.9.12）。`extern "C"` 或 `extern "system"` fn 定义上的 `#[on_panic(return = v)]` 改为恢复：panic 照常报告，panic 钩子运行（[stdlib 3.12](../../refs-standard-libraries/300-std/312-panic.md)），然后函数向其调用方（C 或 Nabi）返回 `v`。`v` 是结果类型的字面量：整数、浮点数、`true` / `false`，或对指针结果而言的 `0`（空指针）。没有结果的函数单独写 `#[on_panic]`。负责恢复的函数是 panic 之下最近的 C 入口：当 C 在更深处再次回调 Nabi 时，由那个内层入口决定。

panic 与该函数之间的栈帧会被跳过。它们的 `Drop` 不会运行，所以它们本应释放的东西会泄漏，在被跳过的栈帧中加锁的 `Mutex` 会保持锁定：下一次对它的 `lock` 会永远等待。持有锁时可能失败的函数应返回错误值，而不是触发 panic。恢复通过 `setjmp` / `longjmp` 实现（[8.5.12](../008-abi/805-extern-symbols.md#8512-auxiliary-symbols-appendix-non-normative)）；wasm32 和没有操作系统的目标都没有它们，在那里 panic 与不带该属性时一样中止。

每种误用都是错误，并说明应写什么：

| 情况 | 消息 |
|------|---------|
| 值不是数字、`bool` 或 `0` | `` `#[on_panic]` on `count`: only a number, a `bool` or `0` (a null pointer) can be returned; write `#[on_panic(return = -1)]` with a value of the result type `` |
| 为没有结果的函数提供了值 | `` `#[on_panic]` on `nothing`: it returns nothing, so write `#[on_panic]` alone `` |
| 有结果的函数没有提供值 | `` `#[on_panic]` on `half`: a function with a result needs the value it returns `` |
| 函数不是 `extern "C"` | `` `#[on_panic]` is for functions C calls: write `#[on_panic(return = -1)] extern "C" fn plain(..)` `` |

调用外部函数与对外暴露函数，共同构成 FFI 的两个方向。

## 5.9.11. `extern fn` 的调用规则

`extern "C"` 函数遵循 C 调用约定，可能与[执行引擎与用户界面](../000-meta/001-execution-modes.md)中讨论的普通函数调用规则部分不同。调用约定上的差异在运行时 ABI 规范（第 7、8 章）中讨论。

## 5.9.12. FFI 安全的责任

`extern` 块的每个项都由用户负责（与[标准库层级](../000-meta/005-standard-libraries-tiers.md)的 `unsafe` 精神一致）。编译器验证：

- 签名的形式。
- 每个类型都是 FFI 类型（5.9.4），会看穿别名、泛型和字段，并且目标的 C 能够传递它。

编译器不验证：

- 外部函数的实际行为（返回 null、返回错误、内存泄漏等）。
- 外部 `static` 的并发安全性。
- 外部库真正的 ABI 兼容性。

用户必须保证这些。

### C 栈帧之下的 panic

panic 不得穿过 C 的栈帧离开 C 所调用的函数：C 对此一无所知，而 `exit` 会在已损坏的状态上运行宿主的处理程序（`atexit`、解释器的关闭流程）。因此，在外部代码调用的 `extern "C"` 或 `extern "system"` fn 中，或在其下层的 Nabi 代码中发生的 panic，会打印其报告（[4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors)），然后打印这一行

```text
panic in a function C called: aborting (`#[on_panic(return = v)]` on the `extern "C" fn` returns `v` to C instead)
```

并中止进程（`SIGABRT`，在 Unix 上退出状态为 134）。下层没有外部栈帧的 panic 以退出码 101 结束进程，与在其他任何地方一样。共享库（`--crate-type=cdylib`）遵循同样的规则，所以 Python、R 或 Julia 这样的宿主永远不会看到其进程 `exit`。`#[on_panic]`（5.9.10）把 panic 变成返回值，`std::panic::set_hook`（[stdlib 3.12](../../refs-standard-libraries/300-std/312-panic.md)）会在中止、退出或恢复之前看到每个 panic。
