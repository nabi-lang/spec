# 4.15. `unsafe`：函数修饰符与块表达式

`unsafe` 关键字以两种形式出现，用来可见地圈定编译器不保证安全的区域：**函数 / 方法声明上的修饰符**和**块表达式**。两者成对使用：声明为 `unsafe fn` 的函数必须在 `unsafe { ... }` 块中调用。

```
unsafe_fn        ::= "unsafe" fn_decl
unsafe_block     ::= "unsafe" block_expr
```

这一模型与 Rust 的 `unsafe` 同构；其目的是在代码中把危险区域显式标出。

## 4.15.1. `unsafe fn`：函数修饰符

在函数或方法声明前加上 `unsafe`，表示该函数的函数体处于安全抽象的保证之外。签名直接声明由调用方负责满足前置条件。

```nabi
impl<T> Ptr<T> {
    unsafe fn read(self) -> T;
    unsafe fn write(self, value: T);
}

extern "C" {
    unsafe fn strlen(s: ConstPtr<c_char>) -> usize;
}

unsafe fn transmute_unchecked<A, B>(a: A) -> B { ... }
```

在 `unsafe fn` 自身的函数体中，不需要单独的 `unsafe { ... }` 块就可以进行不安全的调用。由于签名已经声明了不安全，整个函数体都被视为不安全作用域。

```nabi
unsafe fn copy_two<T>(p: Ptr<T>) -> (T, T) {
    let a = p.read();              // body is an unsafe scope: no explicit block needed
    let b = p.add(1).read();
    (a, b)
}
```

### 与方法接收者一起使用

`impl` 中的方法遵循同样的形式。

```nabi
impl Buffer {
    unsafe fn raw_data(self) -> Ptr<u8>;
    unsafe fn raw_data_mut(ref self) -> Ptr<u8>;
}
```

`unsafe` 与 `ref self` / `self` 等接收者写法不冲突，它位于函数头的最前面。

### 在 `extern "C"` 块中

用户声明的 C ABI 函数全部隐式为 `unsafe`，因为它们的定义不在编译器的控制范围内（[extern 块](../005-modules/509-extern-block.md)）。推荐显式写出（`unsafe fn`）；省略时自动视为 `unsafe`。

每一个 `extern` 函数都是如此，包括运行时符号的 `extern "nabi"` 声明：在不安全作用域之外调用它们是错误（E0061）。

```nabi
extern "C" {
    fn malloc(size: usize) -> CPtr<u8>;        // treated as unsafe automatically
    unsafe fn free(ptr: CPtr<u8>);             // explicit (recommended)
}
```

## 4.15.2. `unsafe { ... }`：块表达式

调用不安全函数、对裸指针解引用等操作只允许出现在 `unsafe { ... }` 块中。与普通[块表达式](../003-expressions/309-block-expressions.md)一样，块的结果是其最后一个表达式的值。

```nabi
let n: usize = unsafe { strlen(cs.as_ptr()) };

let v: i32 = unsafe {
    let p = ptr.add(idx);
    p.read()
};

unsafe {
    free(ptr);
}
```

### 块中允许的操作

在 `unsafe { ... }` 块中，额外允许以下操作：

- 调用声明为 `unsafe fn` 的函数 / 方法
- 对裸指针（`Ptr<T>`、`ConstPtr<T>`、`ManyPtr<T>`）解引用（`p.read()`、`p.write(v)`）
- 直接调用 [`bare::*`](../../refs-standard-libraries/100-bare/README.md) 模块中的危险函数
- 读取 `union` 字段（位重解释，[联合体](../002-types/205-union.md)）。写入字段不需要 `unsafe`。
- 在值被交给另一个所有者之后继续使用它（[标准库 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)）：由块的作者保证没有其他线程同时使用它。

其他所有普通的安全操作在不安全块中照常工作：`unsafe` *增加能力*，而不*改变已有含义*。

### 收窄作用域

`unsafe` 块应当尽可能窄。与其把整个函数包在 `unsafe { ... }` 中，不如只包住真正危险的那一行，这样其他各行的安全性就一目了然。

```nabi
// recommended: narrow unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    let validated = try check_ptr(p);
    unsafe { validated.read() }    // only the dangerous operation is unsafe
}

// not recommended: wide unsafe scope
fn safe_wrapper(p: Ptr<i32>) throws -> i32 {
    unsafe {
        let validated = try check_ptr(p);
        validated.read()
    }
}
```

## 4.15.3. 调用规则

`unsafe fn` 的调用遵循以下规则。

| 调用位置                       | 能否调用 `unsafe fn`         |
|--------------------------------|------------------------------|
| 普通函数体                     | ✗（编译错误）                |
| `unsafe fn` 函数体             | ✓（自动成为不安全作用域）    |
| `unsafe { ... }` 块            | ✓                            |

违反规则是编译错误：

```nabi
fn safe_caller(p: Ptr<i32>) -> i32 {
    p.read()    // ERROR (E0061): call to unsafe function `read` outside an `unsafe` block
}
```

有两种修正方法：把调用包在 `unsafe { ... }` 中，或者把调用方自身标记为 `unsafe fn`。

### `unsafe fn` 类型

当 `unsafe fn` 或 C 函数被当作值使用时，其类型是 `unsafe fn(..) -> R`（[4.4](404-function-type.md)）。调用这种类型的值需要 `unsafe`。安全的 `fn(..) -> R` 值可以用在 `unsafe fn(..) -> R` 的位置，反之则不行。

```nabi
let f: unsafe fn(usize) -> CPtr<u8> = malloc;   // C function value
let p = unsafe { f(16) };
let g: unsafe fn(i32) -> i32 = |x| x + 1;      // OK: safe → unsafe
let h: fn(usize) -> CPtr<u8> = f;              // ERROR: unsafe → safe
```

## 4.15.4. 在 trait 中

trait 方法也可以使用 `unsafe fn`。`impl` 一方必须用相同的签名（包括 `unsafe`）实现它们：用安全的实现去实现 `unsafe fn` 方法，或用 `unsafe fn` 去实现安全的方法，都是错误（E0074）。调用方遵循与普通调用位置相同的规则。

```nabi
trait RawAccess {
    unsafe fn raw_get(self, idx: usize) -> u8;
}

impl RawAccess for Buffer {
    unsafe fn raw_get(self, idx: usize) -> u8 {
        self.ptr.add(idx).read()
    }
}

let buf: Buffer = ...;
let b = unsafe { buf.raw_get(0) };
```

### `unsafe trait`

如果一个 trait 的实现要承诺编译器无法检查的事情，就把它声明为 `unsafe trait`。它的每一个实现都是 `unsafe impl`；普通的 `impl` 是编译错误（E0078），错误会给出 `unsafe impl` 的写法。使用该 trait（约束、调用）不需要 `unsafe`。`bare::mem::Pod`（每一种位模式都是一个值）就是这样的 trait（[标准库 1.3.3](../../refs-standard-libraries/100-bare/103-mem.md#133-the-pod-trait)）。

```nabi
/// Every all-zero bit pattern is a value.
unsafe trait Zeroable {
    fn zeroed() -> Self;
}

unsafe impl Zeroable for i32 {
    fn zeroed() -> i32 { 0 }
}
```

### `unsafe impl`

`unsafe impl` 用于实现 `unsafe trait`，或手动实现自动标记 trait `Send` / `Sync`。它标明作者保证该 trait 所承诺的内容（对 `Send` / `Sync` 而言，即该类型可以安全地跨越线程边界）。对其他任何 trait 使用 `unsafe impl` 都是编译错误（[2.9.1](../002-types/209-impl.md#291-two-forms)）。

```nabi
unsafe impl Send for Handle { }
```

## 4.15.5. 责任模型

`unsafe` 并不意味着编译器*放弃*安全检查。它意味着以下两项责任转移给调用方：

1. **内存安全**：不得引发悬垂指针、未对齐访问或数据竞争等未定义行为（[4.15.8](#4158-undefined-behavior)）的责任。
2. **前置条件**：满足 `unsafe fn` 文档所承诺的使用条件（例如指针指向有效对象、长度在缓冲区之内）的责任。

`unsafe { ... }` 是调用方接受这两项责任的可见信号，因此在代码审查时危险区域一目了然。

## 4.15.6. 与隐式不安全的关系

以下位置无需显式 `unsafe`，自动被视为不安全作用域：

- `unsafe fn` 的函数体

这种自动处理减少了在常见危险区域中反复书写 `unsafe { ... }` 的噪音。

## 4.15.7. 与其他语言的比较

| 语言     | unsafe 标记模型                                             |
|----------|-------------------------------------------------------------|
| Rust     | `unsafe fn` + `unsafe { ... }`（同构）                      |
| C / C++  | 没有单独的标记；所有代码都潜在不安全                        |
| Swift    | `@unsafe` 注解 / `withUnsafePointer` 函数式模式             |
| Go       | `unsafe` 包（单独导入）；没有块标记                         |
| Zig      | `@ptrCast` 等显式内置函数；没有块标记                       |
| **Nabi**  | **采用 Rust 风格的 `unsafe fn` + `unsafe { ... }`**        |

Nabi 原样沿用 Rust 的模型。危险区域的视觉分隔由单一关键字一致地标出，这对代码审查和静态分析都有好处。

## 4.15.8. 未定义行为

以下是 Nabi 中未定义行为的完整列表。运行时做了其中任何一件事的程序都没有意义：编译器可以假定它永远不会发生，之后可能发生任何事情。每一项都需要 `unsafe` 代码（或外部代码）才会发生；安全的 Nabi 不会引发其中任何一项。每个提到未定义行为的库章节，所指的都是以下种类之一。

内存与指针：

1. 通过一个不指向其类型的存活内存的指针进行读写：空指针、悬垂指针（它所来自的局部变量、pin 或分配已经不存在）、越出其对象边界，或超出调用方担保的 `ManyPtr` / `GcBuf` / `Slice::from_raw_parts` 范围（[标准库 1.1](../../refs-standard-libraries/100-bare/101-ptr.md)、[标准库 1.3.9](../../refs-standard-libraries/100-bare/103-mem.md)）。
2. 在使指向 GC 对象的裸指针有效的 pin 被释放之后，继续使用该裸指针（[标准库 1.6](../../refs-standard-libraries/100-bare/106-gc-pin.md)）。
3. 通过未对齐的指针进行读写，`ptr_cast` 可能产生这样的指针（[标准库 1.2.2](../../refs-standard-libraries/100-bare/102-cast.md)）。
4. 对失去来源（provenance）的指针解引用，但由 `from_exposed_provenance` 从已暴露地址生成的指针除外（[标准库 1.1.2](../../refs-standard-libraries/100-bare/101-ptr.md)）。
5. 通过指向非 `mut` `static` 的指针写入，或通过 `const_cast` 的结果向不可写的内存写入（[标准库 1.1.2](../../refs-standard-libraries/100-bare/101-ptr.md)、[标准库 1.2.4](../../refs-standard-libraries/100-bare/102-cast.md)）。
6. 对重叠的范围使用 `copy_nonoverlapping`（[标准库 1.3.4](../../refs-standard-libraries/100-bare/103-mem.md)）。
7. 在 `drop_in_place` 运行其释放胶水代码（drop glue）之后继续使用该值，或运行它两次（[标准库 1.3.4](../../refs-standard-libraries/100-bare/103-mem.md)）。

值：

8. 以任何类型读取未初始化的内存：在值被写入之前调用 `MaybeUninit::assume_init`，或读取没有任何东西写入过的字节（[标准库 1.3](../../refs-standard-libraries/100-bare/103-mem.md)）。
9. 通过 `union` 读取（[2.5.2](../002-types/205-union.md)）、`transmute_unchecked`、裸写入或虚假的 `unsafe impl Pod`，产生对其类型无效的值：不是 `0` / `1` 的 `bool`、不是 Unicode 标量值的 `char`、没有任何变体对应的枚举或和类型标签、既不是 null 也不是存活对象的 GC 引用、为 null 的非 `none` `extern "C" fn`（[标准库 1.2.6](../../refs-standard-libraries/100-bare/102-cast.md)、[5.9.4](../005-modules/509-extern-block.md)）。
10. 破坏用 `new_unchecked` 创建的受检包装类型的不变量：为零的 `NonZero`、为 NaN 的 `NonNan`、为 NaN 或无穷的 `Finite`、为负的 `Positive`（[标准库 2.2.6](../../refs-standard-libraries/200-core/202-non-zero.md)）。
11. 位于 GC 堆之外、收集器不扫描的内存中的 GC 引用：在这样的内存上保存 GC 引用的 `ManyPtr::slice`，或通过裸写入存放在那里的 GC 引用（[标准库 1.1.4](../../refs-standard-libraries/100-bare/101-ptr.md)）。

线程：

12. 数据竞争（[7.9.4](../007-runtime/709-memory-model.md#794-data-races)）：通过裸指针或 `GcBuf`、虚假的 `unsafe impl Send` / `Sync`、对其他东西也能触及的值使用 `Mutex::new_unshared`、在 `unsafe` 中跳过移交检查，或在 `env::set_var` 运行期间访问环境的 C 代码（[标准库 3.5](../../refs-standard-libraries/300-std/305-env.md)）。

外部代码：

13. 声明类型与外部定义不匹配的外部函数、`static` 或回调，或者在 C 持有原始位的位置传入 GC 值（[5.9.4](../005-modules/509-extern-block.md)、[5.9.6](../005-modules/509-extern-block.md)）。
14. 来自外部代码的异常或栈展开跨越 Nabi 栈帧（[5.9.2](../005-modules/509-extern-block.md)）。
15. 在声明为 `str` 的位置返回 null 的 `char*`；C 在调用返回之后仍保留 `str` 实参的指针（[5.9.4](../005-modules/509-extern-block.md)）。
16. 对悬垂指针或指向不含 NUL 的字节的指针使用 `CStr::from_ptr`（[标准库 1.7.2](../../refs-standard-libraries/100-bare/107-ffi.md)）。

其他一切在安全代码和不安全代码中都是有定义的。特别地，以下情况不是未定义行为：整数溢出（启用检查时触发 panic，关闭检查时回绕，[标准库 2.3](../../refs-standard-libraries/200-core/203-ops.md#overflow-behavior)）、集合类型的越界索引或切片（触发 panic）、除以零（触发 panic）、栈溢出（进程中止，[7](../007-runtime/README.md)）、所有浮点运算（包括 NaN 和无穷，[标准库 2.1.4](../../refs-standard-libraries/200-core/201-numbers.md#floating-point-semantics)）、读取 `union` 字段（位重解释；只有它产生的值可能无效，见第 9 项）、不同大小类型之间的 `transmute_unchecked`（缺失的字节读作零）、死锁以及内存泄漏。
