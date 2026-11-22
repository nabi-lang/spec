# 5.3. `static`

`static` 关键字（一个上下文关键字，只在项的开头才是关键字；其后的 `mut` 同样只在 `static mut` 位置才是关键字，[1.5.4](../001-lexical/105-keywords.md#154-上下文关键字)）声明在程序整个生存期内持续存在的全局内存。

```
static_decl     ::= "static" "mut"? identifier ":" type_expr "=" expr ";"
extern_static   ::= "static" "mut"? identifier ":" type_expr ";"   // in extern block
```

## 5.3.1. 基本形式

```nabi
static SHARED_COUNTER: Atomic<i32> = Atomic::new(0i32);
static DEFAULT_CONFIG: Config = Config::default();
static APP_NAME: str = "nabi";
```

基本形式是 `static`、名称、`:` 加类型，以及 `=` 加初始化式。类型必须写出。`const` 初始化式（[`const`](502-const.md)）在编译期求值；其他初始化式在首次访问时运行一次（[5.3.9](#539-初始化式编译期或首次使用时)）。

`static` 可以从每个线程访问，所以它的类型必须是 `Sync`（[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync线程边界标记)）；延迟初始化的 static 也是如此。`static XS: List<i32> = [];` 是编译错误（集合类型、`StrBuf` 和 `Cell<T>` 都是 `!Sync`）；应写作 `static XS: Mutex<List<i32>> = Mutex::new([]);`。每个线程各自的状态使用 `thread::local`（[stdlib 3.4.4](../../refs-standard-libraries/300-std/304-thread.md#344-线程局部存储)），其 `ThreadLocal<T>` 是 `Sync`。`extern` 块中的 static（[5.3.10](#5310-extern-staticffi)）不受此限制。

## 5.3.2. 与 `const` 的区别

```
const X: i32 = 42i32;              // inlined at each use (no address)
static X: i32 = 42i32;             // stable memory address (program lifetime)
```

| 方面         | `const`        | `static`                                       |
|----------------|----------------|------------------------------------------------|
| 内存地址 | 无           | 稳定的地址                                 |
| 在使用处    | 内联        | 内存引用                               |
| 生存期       | 编译期   | 程序生存期                               |
| 可变性     | 始终不可变 | 默认不可变（通过包装器实现可变） |

## 5.3.3. 允许的位置

`static` 可以在模块级或块中（[3.9.1](../003-expressions/309-block-expressions.md#391-形式)）声明，不能在 trait 或 `impl` 中声明（[5.3.8](#538-与方法trait-的组合)）。

```nabi
static APP_NAME: str = "nabi";                          // module level: OK

fn f() -> i32 {
    static CALLS: Atomic<i32> = Atomic::new(0i32);     // OK: one slot, visible in this block
    CALLS.fetch_add(1i32)
}
```

块中的 `static` 在整个程序中只有一个槽，只初始化一次，无论运行的是其所在函数的哪个实例。

## 5.3.4. 普通用户代码中的可变全局变量：使用包装器

```nabi
static COUNTER: Atomic<i32> = Atomic::new(0i32);
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

需要内部可变性的全局变量，要包装在并发安全的包装器中声明。

- `Atomic<T>`：无锁原子操作（整数、`bool`、指针）。
- `Mutex<T>`：基于锁的保护。
- `RwLock<T>`：读写锁，用于本身是 `Sync` 的 `T`（[stdlib 2.16.4](../../refs-standard-libraries/200-core/216-sync.md#2164-rwlockt读写分离)）。

`Cell<T>` 是单线程的且为 `!Sync`，所以不能作为 `static`；每个线程各自的可变全局变量用 `thread::local(..)`。

这些都是标准库类型；它们的确切签名在库规范中定义。

## 5.3.5. `static mut`：仅用于 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)

只有在直接与 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 交互时才可以使用 `static mut`，而且只能在 `extern "C"` 块内使用。

```nabi
extern "C" {
    static environ: CPtr<CPtr<c_char>>;   // FFI global (read-only)
    static mut atexit_count: u64;  // FFI mutable global
}
```

- `static mut` 只允许出现在 `extern "C"` 块内。
- 普通 Nabi 代码中的 `static mut` 是错误（ERROR）。
- 给不是 extern `static mut` 的 `static` 赋值是编译错误（`COUNTER = ...`），对其调用 `ref self` 方法也是（`S.set(3)`，[4.7](../004-functions/407-ref-keyword.md)）。内部修改通过上述包装器的方法进行。

共享的可变状态是数据竞争的直接成因，因此有这一限制。普通代码通过包装器获得安全性；裸访问只允许在 FFI 领域中进行。

## 5.3.6. 初始化顺序

多个 `static` 声明之间的初始化顺序：

- 同一模块内，按声明顺序。
- 跨模块时，按依赖顺序：如果一个 `static` 的初始化式引用另一个 `static`，后者先初始化。
- 通过直接引用形成的循环，即每个初始化式都指名下一个 `static`（`static A: i32 = B; static B: i32 = A;`），是编译错误。
- 通过函数调用形成的循环不在编译期检查；它会在运行时触发 panic。

`const` 初始化式的顺序检查在编译期进行。非 `const` 初始化式在首次访问时初始化。初始化是线程安全的；初始化式恰好运行一次，并发访问的其他线程会等待它完成。通过调用读取自身 `static` 的初始化式会触发 panic。

## 5.3.7. 可见性

与其他项一样，`static` 可以带可见性关键字。

```nabi
pub static VERSION: str = "1.0.0";
pub(pkg) static INTERNAL: i32 = 42i32;
static LOCAL_ONLY: bool = false;
```

详细的可见性规则在[可见性](506-visibility.md)中定义。

## 5.3.8. 与方法、trait 的组合

`static` 是存放值的槽，不是定义项的东西。因此它不出现在 trait 或 `impl` 块的定义中；那里应使用 `const` 或关联函数。

```nabi
impl Counter {
    static GLOBAL: i32 = 0i32;     // ERROR: no static inside impl
    const ZERO: i32 = 0i32;        // OK: associated const
}
```

## 5.3.9. 初始化式：编译期或首次使用时

如果初始化式是常量表达式（[5.2.4](502-const.md#524-常量表达式与求值器)），它在编译期初始化。其他初始化式在首次使用时初始化一次（延迟初始化）。延迟初始化是线程安全的，并发访问的线程会等待初始化完成。初始化式中的函数不必是 `const fn`（如 `Mutex::new` 等）。

```nabi
static PI: f64 = 3.14159f64;             // const: compile time
static START: Instant = Instant::now();   // initialized once at first access
static CACHE: Mutex<Dict<str, Data>> = Mutex::new(Dict::new());
```

## 5.3.10. `extern static`：FFI

如上所示，`extern "C"` 块中的 `static` 从 Nabi 引用一个在外部定义的全局变量；不写初始化式。

```nabi
extern "C" {
    static environ: CPtr<CPtr<c_char>>;   // externally defined, read-only from nabi
}
```

详细规则在 [`extern` 块](509-extern-block.md)中定义。

## 5.3.11. 生存期与 GC 根

`static` 的内存一直保留到程序终止（`const` 初始化式在启动时初始化，其他初始化式在首次访问时初始化）。`static` 中每个存放 GC 引用的字都在程序启动时注册为 GC 根（`nabi_gc_add_root`，[8.5](../008-abi/805-extern-symbols.md)），所以 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 会让 `static` 引用的对象保持存活。

其他线程看到的 `static` 内容遵循内存模型（[7.9](../007-runtime/709-memory-model.md)）。GC 和运行时行为在 [7. 运行时模型](../007-runtime/README.md)和 [8. 运行时 ABI](../008-abi/README.md) 中讨论。
