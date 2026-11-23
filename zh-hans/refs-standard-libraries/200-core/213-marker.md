# 2.13. marker：`Clone` / `Copy` / `Drop`

复制值和清理不再需要的资源，是任何语言都绕不开的问题。`core::marker` 模块用三个 trait 表达这两个问题。`Clone` 定义如何制作值的独立副本；`Copy` 是一个标记，表明对某个类型来说普通的位复制就足够；`Drop` 定义值离开作用域时必须运行的清理代码。这个模块还包含 `NoneSentinel`，即 `none` 字面量类型在库中的名称（2.13.8）。

Nabi 使用 GC，这直接影响这些 trait 的含义。回收内存由 GC 负责，所以 `Drop` 与内存释放无关。`Drop` 只用于终结（finalization）工作，即清理位于语言之外的资源，例如文件句柄和套接字。这使 Nabi 的标记 trait 既不同于 Rust，也不同于 Java/Python。

## 2.13.1. 内存模型：带 GC 的语言中的 Clone/Drop

在 Nabi 的运行时中，[Immix + Sticky Mark-Bit 分代 GC](../../refs-lang/007-runtime/703-gc-algorithm.md) 自动管理堆对象的生命周期。对象何时从内存中消失不由用户决定。但外部资源的清理必须是确定性的，所以 Nabi 的 `Drop` 作为独立于 GC 的机制运作。与 Rust 比较如下：

| 方面                 | Rust（无 GC）                | Nabi（GC）                                        |
|----------------------|------------------------------|---------------------------------------------------|
| 内存回收             | 由 `Drop` 负责               | GC 自动完成；`Drop` **不参与**                    |
| `Drop::drop` 的含义  | 释放资源 + 释放内存          | **只清理外部资源**（文件句柄、套接字等）          |
| 调用时机             | 确定性的（作用域结束）       | 确定性的：最后一个所有者离开作用域处，或 `using` 块结束处（与 GC 回收无关） |
| 遗漏时的影响         | 资源泄漏 + 内存泄漏          | 只有资源泄漏（内存最终由 GC 回收）                |

> [!NOTE]
> `Drop` 值是仿射的：编译器在其最后一个所有者离开作用域处，或在绑定它的 `using` 块结束处调用 `Drop::drop`（[refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)）；关闭可能失败的类型必须显式关闭（2.13.4）。GC 从不调用 `drop`。

由于内存回收和资源清理是分开的，普通对象不必关心 `Drop`（GC 会回收它们），只有处理外部资源的类型才实现 `Drop`，以获得确定性的清理时机。

## 2.13.2. `Clone`：深复制

`Clone::clone` 是需要值的独立副本时调用的方法。它的签名很简单：

```nabi
#[lang("clone_trait")]
trait Clone {
    fn clone(self) -> Self;
}
```

`clone` 执行深复制。即使值在内部持有引用，引用所指向的数据也会被复制，因此之后双方可以独立修改。GC 保证不可变的共享数据仍可以原样共享。

### `Clone` 的含义

下面的示例以最直接的形式展示 `Clone`。

```nabi
let a: List<i32> = [1, 2, 3];
let b = a.clone();           // new List, with a new internal buffer
b.push(4);                   // a is unaffected
a;                           //→ [1, 2, 3]
b;                           //→ [1, 2, 3, 4]
```

这里的“深”恰恰是指：`Clone` 在保持逻辑相等的同时，保证值之间修改互不影响。由此得出两个性质。`let b = a.clone();` 刚执行完时，若定义了 `PartialEq`，则 `a == b` 成立。此后，对 `a` 的任何修改都不会反映到 `b` 上，反之亦然。

### 没有自动调用：`clone_trait` lang 槽

在有些语言中，写 `let y = x;` 会让编译器自动调用 `clone`。Nabi 在这里选择显式的一方。

```nabi
let xs: List<i32> = [1, 2, 3];
let ys = xs;          // GC heap object → reference shared (no clone)
```

`let y = x;` 对 GC 堆对象共享引用，对 `Copy` 类型执行位复制。因此 `Clone::clone` 从不被自动分派；只有用户显式调用时它才运行。正因如此，`Clone` 没有注册为方法槽，而是注册为 trait 槽 `clone_trait`，编译器只在 `#[derive(Clone)]` 的展开中查找这个槽。

### 默认主体

原始类型的 `clone` 相当于位复制，所以直接映射到内建函数；集合类型的 `clone` 分配新的内部缓冲区并递归地克隆元素。

```nabi
impl Clone for i32 {
    #[intrinsic("nabi.clone.copy")]    // bitwise copy
    fn clone(self) -> i32;
}

impl<T: Clone> Clone for List<T> {
    fn clone(self) -> List<T> {
        let r: List<T> = [];
        for x in self {
            r.push(x.clone());
        }
        r
    }
}
```

克隆 GC 对象总会分配新的缓冲区，并递归地克隆元素。

### `#[derive(...)]`

对于字段全部是 `Clone` 的大多数用户定义结构体，没有理由手写主体。`#[derive(Clone)]` 会代为完成。

```nabi
#[derive(Clone)]
struct Config {
    name: str;
    timeout: u32;
    retries: u32;
}

let c1 = Config { name: "default", timeout: 30, retries: 3 };
let c2 = c1.clone();         // every field cloned recursively
```

派生的主体对每个字段调用 `Clone::clone`，并以相同的结构返回结果。因此每个字段都必须（递归地）实现 `Clone`。

## 2.13.3. `Copy`：标记

```nabi
trait Copy: Clone {
    // marker: no methods
}
```

在 Nabi 中，每个值都按值复制。对于集合类型和 GC 对象，复制的是句柄，指向同一个对象；原始值、`str`（不可变的 `(ptr, len)`）、结构体、枚举和元组复制的是其本身（[refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-值语义)）。因此 `Copy` **只是一个标记**，不改变赋值或传参的含义。`Copy` 只用于泛型约束（`T: Copy`），用来选出“位复制即为独立副本”的类型。

### 实现策略

| 类型                             | `Clone`     | `Copy`             | 备注                                      |
|----------------------------------|-------------|--------------------|-------------------------------------------|
| `i*`、`u*`、`f*`                 | ✓           | ✓                  | 原始数字类型                              |
| `bool`、`char`、`c_char`         | ✓           | ✓                  |                                           |
| `str`                            | ✓           | ✓                  | 不可变的 `(ptr, len)` 值                  |
| `int`、`float`                   | ✓           | ✓                  | 不可变的值；堆上的 BigInt 只会被共享      |
| `List<T>`、`Dict<K,V>`、`Set<T>` | ✓（T:Clone）| ✗                  | 复制句柄是共享，而不是独立副本            |
| 元组                             | ✓（全部）   | ✓（全部为 Copy 时）| 逐个分量                                  |
| `Range<T>`                       | ✓（T:Clone）| ✓（T:Copy）        | start/end 都是 Copy 时                    |
| 用户结构体 / 枚举                | derive      | derive             | 每个字段都实现同一 trait 时               |

实现 `Drop` 的类型不能是 `Copy`。

```nabi
#[derive(Clone, Copy)]
struct Vec3 { x: f64; y: f64; z: f64; }
```

> [!IMPORTANT]
> 只有同时实现了 `Clone`，并且每个字段都是 `Copy` 时，才允许派生 `Copy`。

## 2.13.4. `Drop`：释放时的清理

打开的文件必须关闭；加锁的互斥锁必须解锁。这类外部资源的清理不能交给 GC。如果把资源一直持有到无法预测的回收时刻，可能耗尽文件描述符，或者让其他代码卡在一把永远不会释放的锁上。`Drop` trait 和 `using` 表达式解决了这个问题。

```nabi
trait Drop {
    #[lang("drop_method")]
    fn drop(ref self);
}
```

### 所有权

类型实现了 `Drop` 的值是仿射的：它只有一个所有者，并且只释放一次。持有这种值的结构体、元组、枚举或和类型也是仿射的。

- `let b = a` 移动值：`b` 拥有它，之后使用 `a` 是编译错误（E0133）。把它存入结构体、元组、枚举或容器，在闭包或 `async` 块中捕获它，以及返回它，都以同样方式移动它。不持有这种值的变体（`Stdio::Piped`）的值没有可移动的内容。
- 只有被调用者保留该值时，调用才会拿走它：存储它、返回它、关闭它，或者把它传给会保留它的调用。其他调用都只是借用它，调用者仍然拥有它。立即被 await 的 `async fn` 调用也属于这种调用：除非函数主体保留了实参，否则 `await` 结束时实参回到调用者手中。回来的是调用者自己的值，而不是被调用者的副本：被调用者遍历过的流仍停在调用之前的位置（要从被调用者停下的地方继续，就从被调用者返回它）。通过函数值或闭包的调用没有可供查看的主体，所以会拿走该值。
- 遍历容器时（`for x in list`）看到的元素或从容器读出的元素仍归容器所有。
- `clone()`（对 `File` 是 `try_clone()`）生成一个拥有自己所有者的独立值。
- 流适配器（`r.lines()`、`r.take(n)`、`l.incoming()`）是流的作用域内视图，不是第二个所有者：它留在所有者的作用域中（E0139，[2.23.1](223-io.md#2231-read)）；`into_` 形式（`r.into_lines()`）则会拿走流。
- 只有通过 `marker::replace(ref s.f, v)` 或 `marker::take(ref s.f)` 才能把一部分从结构体中移出，它们会在原处留下一个新值。
- `unsafe { marker::share(x) }` 生成一个编译器不追踪的第二个句柄：它既不移动也不释放 `x`。所有者释放该值后再使用它，这种情况由程序自己排除。

```nabi
// core::marker
pub unsafe fn share<T>(x: T) -> T;
#[discardable]
pub fn replace<T>(ref dest: T, value: T) -> T;     // puts `value` in `dest`, returns the old value
#[discardable]
pub fn take<T: Default>(ref dest: T) -> T;         // replace(ref dest, T::default())
```

```nabi
let (tx, rx) = channel::<i32>();
let other = tx;          // moves: `tx` is gone
try tx.send(1);          // error: `tx` was moved at line 2, and is used here
let copy = other.clone(); // a second sender with its own owner
```

### 调用时机

编译器在值的所有者离开作用域处释放它，覆盖每一种退出方式（正常完成、`return`、`break`、错误传播）；任务完成或被取消时，也会释放其 `async` 块捕获的内容。`using` 在其块结束时释放它的值（[refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)）。已经移走的值由新的所有者释放，而不在此处释放。重新声明所有者名称的 `let`（[refs-lang 5.1.4](../../refs-lang/005-modules/501-variable-declaration.md#514-在同一作用域中重新声明)）既不移动也不释放旧值：旧值在旧绑定离开作用域处释放，必须关闭的值要在重新声明之前关闭或移走。

关闭可能失败的类型实现 `io::Close`（`fn close(self) throws (IoError)`）：`File`、基于它的 `BufWriter` 或 `LineWriter`，以及任何持有它的结构体。编译器不会替你关闭这样的值，因为关闭的错误会丢失：用 `try x.close()` 关闭它，或者用 `using` 绑定它。在某条路径上未关闭就离开其作用域是编译错误；只为一次借用它的调用而创建的临时值也是如此（`(try File::open(p)).read_to_str()`、`io::BufReader::new(try File::open(p)).lines()`）：调用之后没有任何东西拥有它，因此也没有任何东西会关闭它（E0134）；请用 `using` 绑定它（`using f = try File::open(p) { try f.read_to_str() }`）。带着错误离开的路径会释放它（此时关闭自身的错误被丢弃）。`try?` 会丢弃它捕获的错误，所以对一个错误可能持有这种值的调用使用 `try?`（`SendError<File>` 会交回未发送的值）是编译错误：用 `try .. catch` 捕获错误，取出值并关闭它。失败的 `try!` 会 panic，所以它持有的错误无需关闭。持有这种值的和类型或枚举按成员关闭：看到该成员的 `match` 分支负责关闭它，只看到编译器会自行释放的成员（`Sender`）的分支则无需关闭。模式把和类型收窄到的成员（`if let x: Pair = p`、`match` 分支、`let .. else`）会像结构体一样逐部分追踪：关闭 `x.a` 后 `x.b` 仍未关闭，而再次关闭 `x.a` 或之后整体使用 `p` 都是错误。

- 在正常完成、`return`、`break` 和错误传播时都会调用它。
- 多个绑定按声明的逆序（LIFO）释放。
- panic 会结束进程，所以不会调用 `drop`。
- drop glue：在值自身的 `drop` 之后，具有 `Drop` 的字段按声明的逆序递归清理。元组元素以及枚举 / 和类型当前持有的载荷也是如此。

```nabi
fn process(path: str) throws (IoError) {
    using lock = MUTEX.lock(), file = try File::open(path) {
        do_work(file);
    }   // ← on normal exit or error: file.drop() then lock.drop()
}

{
    let f = try! File::open("data.txt");
}   // ← error: `f` holds a `File` that is never closed
```

用 `using` 绑定没有实现 `Drop` 的类型是编译错误，错误信息会说明需要 `Drop`。

### 与 GC 的关系

GC 从不调用 `drop`，也没有终结器：释放点就是上面那些，在编译时确定。

### `Drop` 实现示例

最典型的实现是对操作系统资源的包装。

```nabi
extern "C" { fn close(fd: i32) -> i32; }

struct Fd {
    handle: i32;
}

impl Drop for Fd {
    fn drop(ref self) {
        unsafe { close(self.handle); }    // OS file descriptor close; its result is ignored
    }
}
```

`Fd` 的关闭结果被忽略，所以编译器隐式地释放它。关闭可能失败的类型（`File`）还实现 `io::Close`，并如上文所述显式关闭。

### `Drop` 的限制

`Drop` 有几项限制。第一，实现 `Drop` 的类型不能是 `Copy`：一边通过位复制随意增加副本，一边保证确定性的清理，二者自相矛盾。

```nabi
#[derive(Copy)]              // ERROR - conflicts with Drop
struct File { handle: i32; }
impl Drop for File { ... }
```

第二，直接调用 `drop` 是编译错误。只有在释放时恰好调用一次，才能保证正确性。

```nabi
let f = File::open(path);
f.drop();                    // ERROR: calling Drop::drop directly is forbidden
```

要提前关闭资源，请提供 `File::close()` 这样的单独方法。

唯一的例外是 `bare` 中的不安全代码。例如，`unsafe fn drop_in_place`（[1.3 mem](../100-bare/103-mem.md)）是用于手动内存管理的函数。它运行 drop glue：先是类型自身的 `drop`，再如上文所述按声明的逆序运行各字段的 `drop`。

此外，`drop` 的签名没有 `throws`，所以其主体不能抛出错误。`drop` 内部的 panic 与其他 panic 一样会结束进程。

```nabi
impl Drop for X {
    fn drop(ref self) {
        if some_cond { throw "oops" }    // ERROR: drop is not throws
    }
}
```

### 派生策略

与 `Clone` 不同，`Drop` 不能派生。在任何释放点，值自身的 `drop` 完成后，编译器都会按声明的逆序对该结构体中具有 `Drop` 的字段调用 `drop`。因此用户编写的 `Drop` 主体只需处理“字段之外需要清理的内容”。

```nabi
struct Container {
    a: File;
    b: Socket;
}
// when a Container is released, release does b.drop() → a.drop()
// (because the fields themselves implement Drop)

impl Drop for Container {
    fn drop(ref self) {
        log("Container released");
        // field drops are called automatically by the compiler _after this method returns_
    }
}
```

## 2.13.5. lang 槽对应

这个模块通过四个 lang 槽与编译器连接。自动调用点遵循上文所述的策略。

| 槽            | 附加到                       | 自动调用点                                 |
|---------------|------------------------------|--------------------------------------------|
| `clone_trait` | `Clone` trait                | 无；用于 `#[derive(Clone)]` 的展开         |
| `drop_method` | `Drop::drop` 方法            | 每次释放：最后一个所有者离开作用域、任务结束、`using` 块结束（以及 drop glue） |
| `send_trait`  | `Send` trait（2.13.9）       | 自动实现，线程边界检查                     |
| `sync_trait`  | `Sync` trait（2.13.9）       | 自动实现，线程边界检查                     |

详见 [0.4 lang 槽表](../000-meta/004-lang-slot-table.md)的 0.4.2 / 0.4.3 / 0.4.5 节。

## 2.13.6. trait 实现矩阵

标准库的主要类型与这三个 trait 的关系：

| 类型                             | Clone       | Copy           | Drop                            | 行为                                 |
|----------------------------------|-------------|----------------|---------------------------------|--------------------------------------|
| `i*`、`u*`、`f*`                 | ✓           | ✓              | ✗                               | `nabi.clone.copy`（逐位）            |
| `bool`、`char`、`c_char`         | ✓           | ✓              | ✗                               | 逐位                                 |
| `str`                            | ✓           | ✓              | ✗                               | 复制不可变的 `(ptr, len)` 值         |
| `List<T>`、`Dict<K,V>`、`Set<T>` | ✓（T:Clone）| ✗              | ✗                               | 递归克隆                             |
| 元组                             | ✓（全部）   | ✓（全部 Copy） | ✗                               | 逐个分量                             |
| `Range<T>`                       | ✓（T:Clone）| ✓（T:Copy）    | ✗                               | 逐字段（`start`、`end`）             |
| `File`                           | ✗           | ✗              | ✓（另加 `io::Close`）           | 关闭操作系统句柄；显式关闭（2.13.4） |
| `MutexGuard<T>`                  | ✗           | ✗              | ✓（释放时）                     | 解锁互斥锁                           |
| `channel::Sender<T>`             | ✓           | ✗              | ✓                               | 调用 `close()`；隐式释放             |

这张表背后的原则很简单。所有原始类型和上面的集合类型都实现 `Clone`，只有位复制在语义上正确的类型才是 `Copy`。集合类型是 `Clone` 而不是 `Copy`，因为对句柄做位复制并不能保持深复制的含义。`str` 是 `Copy`：它不可变，所以复制其 `(ptr, len)` 就是独立的副本。只有 `File`、`Socket`、`MutexGuard` 等外部资源的包装类型才实现 `Drop`。这些类型通常不实现 `Clone`，以防止资源被重复拥有。

## 2.13.7. 与其他语言的比较

把 Nabi 的模型与现有语言比较，设计意图会更清楚。

| 方面                    | Rust                     | Java/Python（GC）               | Nabi（GC + Drop）        |
|-------------------------|--------------------------|---------------------------------|--------------------------|
| 内存回收                | 由 `Drop` 负责           | GC                              | GC                       |
| 确定性终结              | `Drop`（编译器保证）     | **无**（终结器不确定）          | `Drop`（绑定到作用域）   |
| Copy 的含义             | 逐位 + 强制 `!Drop`      | **无**（每个对象都是引用）      | 逐位 + 强制 `!Drop`      |
| Clone 的含义            | 深复制                   | `clone()`（Object.clone）       | 深复制                   |
| derive                  | `#[derive(Clone, Copy)]` | `Cloneable` 接口                | `#[derive(Clone, Copy)]` |

Nabi 的独特之处在于 GC 与确定性的 `Drop` 共存。由于内存回收由 GC 处理，`Drop` 不表示内存释放，所以普通对象永远不需要实现 `Drop`。只有需要清理外部资源的类型才实现 `Drop`；编译器在其最后一个所有者离开作用域处释放它们，关闭可能失败的类型则由用户用 `close()` 或 `using` 关闭。

这一模型既避免了 Java/Python 终结器的不确定性，也避免了 Rust 因没有 GC 而付出的代价。

## 2.13.8. `NoneSentinel`：`none` 字面量的类型

[refs-lang 1.9.2](../../refs-lang/001-lexical/109-other-literals.md) 中的 `none` 字面量具有编译器内置的 `none` 类型，这是只有一个值的零大小类型。`core::marker` 用类型别名为该类型提供库名称 `NoneSentinel`；不涉及 lang 槽。

```nabi
pub type NoneSentinel = none;
```

用户很少直接写这个名称。在 `T | none` 这样的和类型写法中（[refs-lang 2.6](../../refs-lang/002-types/206-sum-types.md)），`none` 位置就是这个类型；`none` 字面量、`none` 类型写法和 `NoneSentinel` 指的都是同一个内置类型。

## 2.13.9. `Send` / `Sync`：线程边界标记

```nabi
#[lang("send_trait")]
trait Send { }      // safe to move to another thread

#[lang("sync_trait")]
trait Sync { }      // safe to read from multiple threads through shared references
```

这两个 trait 都是没有方法的**自动标记 trait**；编译器按结构实现它们。

- 原始数字类型、`bool`、`char`、`c_char`、`str` 和 `none` 是 `Send + Sync`。
- `struct`、`enum`、`union`、元组、数组或和类型在所有字段（分量）都是 `Send` 时为 `Send`，都是 `Sync` 时为 `Sync`。泛型容器的 `Send` 按同一规则跟随 `T`。
- 集合类型（`List`、`Dict`、`Set`、`StrBuf`、`Deque`、`BinaryHeap`、`Slice`、`SliceMut`）是 `Send` 和 `!Sync`：它们共享一个可变句柄。在线程之间，它们可以冻结后只读共享（`FrozenList`、`FrozenDict`、`FrozenSet`，元素为 `Send + Sync` 时它们也是，[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s只读集合类型)），可以通过交给 `par` 的 `chunks_mut` 视图分成互不相交的部分写入（[2.19.1](219-par.md#2191-核心-api)），也可以通过 `Mutex` / `RwLock` / 通道共享。
- 交出一个值（`Mutex::new(xs)`、`RwLock::new(xs)`、`tx.send(xs)` / `try_send` / `send_async`、`task::spawn_blocking_with_arg(xs, f)`、闭包的 `[move xs]` 捕获、`xs.freeze()`）之后，发送方的句柄不能再用：新的所有者得到一份副本（见下文的运行时复制），发送方不得使用已交出的内容。从守卫中取出的句柄（`get`）、交给守卫的句柄（`set`）或 `update` 的闭包触及的句柄，在守卫的 `using` 块结束后仍指向受保护的对象。之后在任何路径上使用这样的值都是编译错误（交出的值为 `E0105`，从守卫中取出的值为 `E0106`；[refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-诊断--错误消息)）。编译器会通过以下途径追踪该值：
  - 复制（`let ys = xs`、结构体字段或元组元素、字段赋值）；
  - 传递它的字段（`Mutex::new(s.items)`）；
  - 持有它的对象（`all.push(xs)`、捕获它的闭包），双向追踪：交出 `xs` 也会使 `all` 不能再用；
  - 集合的元素。存入集合的内容和从集合读出的内容（`xs[i]`、`get`、迭代、它的切片）是同一个值，所以交出其中任何一方都会使另一方不能再用（`let x = xs[0]; tx.send(xs); x.len()`、`tx.send(xs[0]); xs[1].len()`）。元素已被交出的集合上只允许 `len`、`is_empty`、`capacity`、`clear`、`truncate`、`reserve`、`shrink_to_fit`、`push` 和 `insert`（以及 `Deque` 的 `push_back` / `push_front`）；`clear` 会清空集合，于是它又可以完全使用。从另一个集合复制而来的集合（`extend`、`+=`、`collect`、`par::filter`）与源集合共享元素。用 `pop` 或 `remove` 取出的元素（`List`、`Deque`、`Dict`、`BinaryHeap`）已经离开集合：交出它之后集合仍可使用，除非集合可能两次持有该对象：一个值存了两次，两个存入的值共享一个对象，或者在循环中把一个集合的元素存入另一个集合（元素不按位置区分）；
  - 交出某个内容的直接调用的实参：通过两个实参传入同一个对象，而被调用者通过其中一个交出它、又通过另一个使用它（`f(tx, a, a)`；`h.go(tx, h.items)`，其中 `go` 使用 `self.items`；一个从中 pop 并发送的列表再加上它的一个元素），是编译错误。同一集合的两个元素视为不同的值（`f(tx, rows[0], rows[1])` 被接受）；
  - `ref` 参数。当被调用者交出它的一部分，然后在每条交出它的路径上都给那一部分赋予新值时（`tx.send(self.buf)` 之后 `self.buf = []`），调用者的变量会得到新值并保持可用；指向旧值的其他句柄则不会。按值传递的参数是副本：给它的字段赋新值不会影响调用者，调用者持有的仍是已交出的内容；
  - 调用。交出某个参数、返回它或它的一部分、把它存入另一个参数的对象或 `ref` 参数、把它捕获进所返回的闭包，或者返回守卫参数所持有内容的函数，对其调用者而言也做了同样的事，跨越预构建的库也是如此（[refs-lang 9.4.7](../../refs-lang/009-tools/904-nabihir-bytecode.md#947-主体)）。

  通过函数值的调用被视为可能到达所有以该参数类型作为值使用的函数。条件只在其测试的是编译器在每条路径上都已知的值时才被追踪：由常量设置或复制得来的 `bool`、和类型或枚举的变体（`let sent = false; if c { tx.send(xs); sent = true; } if !sent { xs.len() }`），以及调用以何种方式结束。只在无错误返回时，或返回 `none` / `false` 以外的值时才交出参数的函数，在其调用者对应另一种结果的 `catch`、`try?`、`??`、`else` 或 `if` 路径上不交出该参数；对 `async fn` 的 `try await` 的 `catch` 也是如此。其他条件不追踪：在某条路径上交出的值，在该路径与其他路径汇合处即视为已不存在。

  要继续使用某个值，可以交出它的 `clone()`（独立的值），或者先给变量或字段赋一个新值（对结构体副本，只对那个字段）；`set` 会释放先前 `get` 返回的内容。对 `Sync` 值的使用、读取值的标签（`x is none`、`match x`）以及 `unsafe` 内部的使用（[refs-lang 4.15.2](../../refs-lang/004-functions/415-unsafe.md#4152-unsafe---块表达式)）不受此限制。被启动的闭包的普通捕获无需检查：闭包与创建它的代码共享这些捕获，所以它们必须是 `Send + Sync`。`[move x]` 捕获把 `x` 交给闭包（[refs-lang 4.6.11](../../refs-lang/004-functions/406-capture-semantics.md#4611-移动捕获move-x-捕获列表)），只需要 `Send`。把拥有非 `Sync` 内容的闭包（或未标记 `Sync`、可能拥有这种内容的 `fn(..) + Send` 值）传给类型为 `fn(..) + Send` 而不带 `Sync` 的参数，就是交出它，因为被调用者可能在另一个线程上运行它（`thread::spawn`、`task::spawn_blocking`、`par::join`、`par::Scope::spawn`、`par::TryScope::spawn` 等）；之后无论它以何种方式传递（在一次调用中出现两次、经过返回其实参的函数、从列表或元组中取出），使用它都是编译错误（`E0133`）。`async [move ..]` 闭包和它的调用所生成的 future 是同一个所有者：交出其中一个会把其他的一起带走（[refs-lang 4.6.11](../../refs-lang/004-functions/406-capture-semantics.md#4611-移动捕获move-x-捕获列表)）。`task::spawn` 交出它的 future：任务的帧得到 future 所捕获内容的副本，而对启动者来说，future 及其捕获的内容都已不存在，与发送之后相同（[2.18.1](218-task.md#2181-核心-api)）。

  以下也是编译错误：交出从 `Sync` 对象中读出、而自身不是 `Sync` 的值（用户的 `unsafe impl Sync` 类型的字段），其他代码不经交接就能触及它（请改为交出 `clone()`）；`thread::local` 的值，它被借给 `with` 闭包（3.4.4）；通道的 `recv`、任务的结果或锁的守卫所给出的值归接收方所有，不受此规则约束；在守卫仍持有时（直到 `set` 给锁赋予新值）交出守卫的 `get` 返回的内容，或者把它传给会保留它的被调用者（要保留的副本是 `get_clone()`）；在 `using` 块内部返回或抛出它，在那里 yield 它（生成器中的 `yield g.get()`），让它成为块自身的值（`let x = using g = m.lock() { g.get() }`、`f(using g = m.lock() { g.get() })`），无论以何种形式把它带出（循环的 `break` 值、带标签的块、`if` / `match` 分支、`catch`、闭包的结果），或者把它交给在块内创建、在块之后运行的协程（`async fn` 的 future、生成器：`fut = grow(g.get()[0])`）：块结束时会在使用该值之前释放锁。守卫只在其 `using` 块内有效：守卫的副本（推入集合、被闭包捕获、被被调用者存储、被第二个 `using` 绑定、被协程持有：`async fn` 或生成器的实参、`async` 块的捕获）在该块结束后被使用是编译错误；守卫是 `!Sync`（[2.16.3](216-sync.md#2163-mutext互斥锁)）。`par::for_each`、`par::map` 和 `par::filter` 把每个项交给一个工作线程，所以两个项不得触及同一个非 `Sync` 集合。每次调用都会在运行时检查这一点（[2.19.1](219-par.md#2191-核心-api)）：如果两个项触及同一个集合，这些项会在调用线程上逐个运行，并且每个调用点警告一次。切片只读，所以只要没有哪个项持有列表本身，各项就可以通过列表的切片共享它；可写视图（`SliceMut`）在没有其他项的视图与某个项所写的部分重叠时也可以共享它。

  这些规则之外的情况：`unsafe` 内部的代码；通过函数值或闭包的调用，它会跳过上面的实参检查；以及多次 yield 同一个对象的生成器，其结果被收集进一个集合，然后集合的元素被逐个交出。这样的集合不会被视为两次持有该对象。

  在运行时，每次交接还会复制，所以在上述规则遗漏的途径上（以及 `unsafe` 内部的代码中），交出的值按值触及的任何集合都不会在发送方和新所有者之间共享（例外列在下文）。`Mutex::new`、`RwLock::new`、守卫的 `set` 和 `update`、`send` / `try_send` / `send_async`、`task::spawn_blocking_with_arg` 的值、`[move x]` 捕获（在闭包创建时），以及任务、线程或 `par` 作用域作业的结果（和 `try_scope` 的错误），都会给新所有者一份该值触及的每个集合的副本：经由结构体、元组、数组、枚举和和类型的各部分，以及集合的元素，深度不限，使用工作列表而不是每层一个栈帧。被多次触及的集合只复制一次，所以副本保留了值的共享关系和环。被复制的有 `List`、`Dict`、`Deque`、`StrBuf`、`Slice` 以及建立在它们之上的集合，还有 `GcPtr`（[1.6.3](../100-bare/106-gc-pin.md)）。协程 future（`AsyncFuture`、`TryAsyncFuture`）得到自己的帧，其中持有在同一次复制中生成的、其捕获内容的副本；已经开始或已经完成的 future，或者其帧（或所捕获 future 的帧）带有 `[ref x]` 捕获的 future，不能被交出，交接会 panic（[2.18.1](218-task.md#2181-核心-api)）。`Task` 不会被复制，但一旦被交出，对它的每次 `await` 都会得到结果的一份独立副本（[2.18.2](218-task.md#2182-taskt-类型)）。列表及其缓冲区的每个视图继续共享同一个复制后的缓冲区；`str` 字节的视图只复制这些字节（它们不会改变）。`update` 保留守卫已持有的内容：只复制闭包带进来的内容。发送方的句柄保持原样：上述规则遗漏的句柄所指的对象不会被其他线程使用。抛出错误的发送会带回一个相等的值（调用者的值或其副本）。复制所花的时间和内存与复制的内容成正比；一段普通值（没有 GC 引用）只需一次内存复制。工具链内部的 `core::marker::hand_over(x)` 返回同样的副本。`xs.freeze()` 不复制：冻结后的集合接管存储并让 `xs` 变空，有视图的列表则给出一份副本（[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s只读集合类型)）。保持共享的有：`Sync` 部分（包括冻结后的集合）、具有 `unsafe impl Send` 的类型的值（它们自己管理共享：任务、文件、套接字），以及生成器和流（不是 `Send`：它们永远不会被交出）。`Mutex::new_unshared`（2.16.3）不复制。从守卫中取出的句柄（`get`、`update` 闭包的实参）在运行时不复制：上述规则会追踪它以及守卫本身，追踪它们经由每一次复制、`return`、`yield`、块的值以及块内创建的协程帧离开 `using` 块的情况。
- 以下类型被排除：`Cell<T>`、裸指针（`Ptr<T>`、`ConstPtr<T>`、`ManyPtr<T>`、`CPtr<T>`）、裸 `any` 以及约束中缺少 `Send` / `Sync` 的 `any Tr`（只有当约束同时含有 `Send` 和 `Sync` 时，`any Tr` 才是 `Send`：`any` 的各副本共享它的盒子，就像闭包的普通捕获被共享一样，所以跨线程的 `any` 中不能放集合）、不带约束的函数类型，以及带 `[ref]` 捕获的闭包（[refs-lang 4.6.9](../../refs-lang/004-functions/406-capture-semantics.md#469-并行与并发上下文中的附加约束)）。其他例外在各类型的章节中规定。
- 只有写出约束时，函数类型才具有这些 trait，例如 `fn(..) -> R + Send`（[refs-lang 4.4](../../refs-lang/004-functions/404-function-type.md#send--sync-约束)）。只有 `+ Send` 并不会使它成为 `Sync`：它背后的闭包可能拥有一个非 `Sync` 的 `[move x]` 捕获。被多个线程调用的函数值（`static`、`par::*` 闭包）是 `+ Send + Sync`。

自动实现不正确的类型直接声明。内部持有裸指针但以线程安全方式包装它的类型，用 `unsafe impl` 显式加入（安全性由作者负责）；按结构会自动实现但并非线程安全的类型，用 `impl !Send` 显式退出。`Cell<T>` 的排除就是这样声明的。

```nabi
struct Handle { raw: Ptr<u8>; }
unsafe impl Send for Handle { }
unsafe impl Sync for Handle { }

impl<T> !Send for Cell<T> { }
impl<T> !Sync for Cell<T> { }
impl<T> !Send for Ptr<T> { }
impl<T> !Sync for Ptr<T> { }
impl<T> !Send for ConstPtr<T> { }
impl<T> !Sync for ConstPtr<T> { }
impl<T> !Send for ManyPtr<T> { }
impl<T> !Sync for ManyPtr<T> { }
impl<T> !Send for CPtr<T> { }
impl<T> !Sync for CPtr<T> { }
```

向线程或任务传递值的 API（`thread::spawn`、`task::spawn`、`par::*`、`channel`）用这两个 trait 约束其实参。同步原语的策略遵循 [2.16 sync](216-sync.md)。

### 交接机制

```nabi
#[internal]
pub fn hand_over<T>(x: T) -> T;
```

`hand_over(x)` 返回交接给予新所有者的内容：带有它所触及的每个集合副本的 `x`，或者在它不触及任何集合时返回 `x` 本身。它是 `#[internal]`：由 `core` 和 `std` 调用，其他任何包使用它都是错误（[0.4.4](../000-meta/004-lang-slot-table.md)）。

```nabi
pub struct Handoff<T> { /* private fields */ }
impl<T> Handoff<T> {
    #[internal]
    pub unsafe fn new(value: T) -> Handoff<T>;
    pub fn get(self) -> T;
}
```

`Handoff<T>` 在调用者自己的协议（join、锁、通道交接）下在线程之间传递值，该协议保证两个线程不会同时使用它：只要 `T` 是 `Send`，它就是 `Send + Sync`，即使 `T` 不是 `Sync`。`new` 是 `unsafe` 且 `#[internal]`；`get` 返回该值。其他代码把值移入闭包（`[move x]`）或传给 `task::spawn_blocking_with_arg`。

`Transfer` 和 `Memo` 是 `pub(pkg)`：交接复制通过它们触及 `core` 其他模块中的标准集合类型，其他包看不到它们。

```nabi
pub(pkg) trait Transfer {
    fn transfer_copy(self, m: Memo) -> Self;
    fn transfer_keep(self, m: Memo) -> Self;
    fn transfer_visit(self, m: Memo) -> Self;
}

pub(pkg) struct Memo { /* private fields */ }
```

编译器只对其固定列表中的标准句柄类型调用 `Transfer`；其他类型上的实现会被忽略。除了集合类型和 future，这个列表还包含 `crypto::Secret`（[2.21](221-crypto.md)）以及各加密算法保存的秘密字：交接会在秘密存储中给新所有者一份自己的副本，而 `par` 会像对待集合一样逐个运行触及同一个 `Secret` 的项。`Memo` 按原始对象记录一次交接已生成的副本，以及仍需修正的已复制缓冲区。它的方法以及旁边的 `transfer*` 内建函数同样是 `pub(pkg)`。

## 2.13.10. `Default`：默认值

```nabi
trait Default {
    fn default() -> Self;
}
```

| 类型                                 | `default()`        |
|--------------------------------------|--------------------|
| 整数、`int`                          | `0`                |
| 浮点数、`float`                      | `0.0`              |
| `bool`                               | `false`            |
| `char`                               | `'\0'`             |
| `str`                                | `""`               |
| `List`、`Dict`、`Set` 等集合类型     | 空集合             |
| `T \| none`                          | `none`             |

`Default` 可以派生；它用每个字段声明的默认值构建值（[refs-lang 2.3.3](../../refs-lang/002-types/203-struct.md#233-字段默认值)），没有默认值的字段则用字段类型的 `default()`：`#[derive(Default)] struct Cfg { host: str = "localhost"; port: i32; }` 得到 `Cfg { host: "localhost", port: 0 }`。

## 2.13.11. 可派生的 trait

`#[derive(...)]`（[refs-lang 6.2.1](../../refs-lang/006-meta-system/602-built-in-attributes.md#621-derive-自动实现-trait)）能生成的内置 trait 只有以下这些：

| trait         | 生成的实现                                    | 要求                                   |
|---------------|-----------------------------------------------|----------------------------------------|
| `Clone`       | 逐字段 `clone`                                | 每个字段都是 `Clone`                   |
| `Copy`        | 标记                                          | 同时派生/实现了 `Clone`，且不是 `Drop` |
| `Debug`       | 以 `f"{x:?}"` 形式输出字段名和值              | 每个字段都是 `Debug`                   |
| `Default`     | 逐字段：声明的默认值，否则 `default()`        | 每个没有默认值的字段都是 `Default`     |
| `PartialEq`   | 逐字段 `==` 的 AND                            | 每个字段都是 `PartialEq`               |
| `Eq`          | 标记                                          | 同时派生/实现了 `PartialEq`            |
| `PartialOrd`  | 按字段声明顺序的字典序比较                    | 每个字段都是 `PartialOrd`、`PartialEq` |
| `Ord`         | 按字段声明顺序的字典序比较                    | 同时派生/实现了 `Eq`、`PartialOrd`     |
| `Hash`        | 按声明顺序对每个字段 `hash`；枚举先写入其变体索引（写入内容未规定） | 每个字段都是 `Hash` |
| `Error`       | 带默认 `source()` 的 `Error`（[2.14.3](214-error.md#deriveerror)） | 已实现或派生 `Display` 和 `Debug` |

- `Display` 不能派生；`#[display("..")]` 根据 f-string 编写它（[2.10.14](210-fmt.md#21014-派生策略小结)）。
- 派生 `Eq` / `Ord` 而不派生其超 trait 是编译错误。
- 对 `union` 使用 `#[derive]` 是编译错误。
