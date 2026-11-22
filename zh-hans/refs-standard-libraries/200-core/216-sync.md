# 2.16. sync：并发原语

`core::sync` 模块定义用于安全访问共享状态的原语 trait 和类型。在 Nabi 的 GC 和 M:N 调度器之下，它们用于在多个任务或线程之间共享数据。

`Atomic`、`Mutex`、`RwLock` 和 `Cell` 在 [prelude](../000-meta/002-import.md) 中，无需导入即可使用；排序枚举（`LoadOrdering`、`StoreOrdering`、`UpdateOrdering`）不在其中，但它们的变体由参数类型解析（`x.load(SeqCst)`）。如果希望并发代码能被明显地识别出来，推荐使用如下的窄范围导入。

```nabi
use core::sync::{Atomic, Mutex, RwLock, Once, Condvar, Cell};
```

## 2.16.1. 原语概览

每种原语覆盖不同的并发场景。下表概括了各类型的含义和适用场合。

| 类型        | 含义                                       | 使用场合                       |
|-------------|--------------------------------------------|--------------------------------|
| `Atomic<T>` | 无锁原子操作（T = 原始类型）               | 计数器、标志、序列号           |
| `Mutex<T>`  | 互斥锁（单个写者）                         | 共享的可变状态                 |
| `RwLock<T>` | 多个读者或单个写者                         | 读多写少的状态                 |
| `Cell<T>`   | 线程局部的内部可变性（无锁）               | 单线程可变                     |
| `Once`      | 恰好运行一次初始化函数（2.16.4a）          | 延迟的全局初始化               |
| `Condvar`   | 在 `Mutex` 下等待条件（2.16.4a）           | 生产者 / 消费者                |
| `Frozen<T>` | 只读共享一个复制的值（2.16.4b）            | 共享配置                       |
| `FrozenList<T>`, `FrozenDict<K, V>` | 没有人修改、无需复制即可共享的集合（[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s只读集合类型)） | 表、以读为主的映射 |

## 2.16.2. `Atomic<T>`：无锁原子操作

`Atomic<T>` 是最轻量的原语。内部用针对各 T 的原子指令实现，形式如下。

```nabi
struct Atomic<T> {
    // internal: a one-element cell driven by per-T atomic instructions
}

trait AtomicPrimitive { /* marker: i8/i16/i32/i64/isize, u8/u16/u32/u64/usize, bool, Ptr<T> */ }

impl<T: AtomicPrimitive + PartialEq> Atomic<T> {
    const fn new(value: T) -> Atomic<T>;

    fn load(self, ordering: LoadOrdering = SeqCst) -> T;
    fn store(self, value: T, ordering: StoreOrdering = SeqCst);
    #[discardable]
    fn swap(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn compare_exchange(
        self, current: T, new: T,
        success: UpdateOrdering, failure: LoadOrdering,
    ) -> Result<T, T>;                  // the failure result is a value too, not an error
    fn compare_exchange_weak(
        self, current: T, new: T,
        success: UpdateOrdering, failure: LoadOrdering,
    ) -> Result<T, T>;                  // the failure result is a value too, not an error

    // integers only; each fetch_* is #[discardable]
    fn fetch_add(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn fetch_sub(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn fetch_and(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn fetch_or(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn fetch_xor(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
}

impl Atomic<i32> {
    fn wait(self, expected: i32);                          // waits while the value is expected
    fn wait_timeout(self, expected: i32, timeout: Duration);   // wait, at most timeout (none left: returns at once)
    fn wake(self);                                         // wakes every thread waiting on this cell
}
```

`wait` 可能虚假返回，所以调用方要重新检查条件（futex 语义）。

`store` 是原子存储，不是交换。在本实现中 `compare_exchange_weak` 从不虚假失败，其行为与 `compare_exchange` 相同。之所以要求 `PartialEq`，是因为 `compare_exchange` 要把旧值与 `current` 比较；`Ptr<T>` 按地址比较。

### `LoadOrdering`、`StoreOrdering`、`UpdateOrdering`

原子操作有五种内存排序强度。每类操作都有自己的排序类型，只包含它能接受的排序，所以无效的排序是编译错误，而不是运行时检查（与 Swift Atomics 的划分相同）。

```nabi
enum LoadOrdering   { Relaxed, Acquire, SeqCst }                     // load; a failed compare-exchange
enum StoreOrdering  { Relaxed, Release, SeqCst }                     // store
enum UpdateOrdering { Relaxed, Acquire, Release, AcqRel, SeqCst }    // swap, fetch_*, a compare-exchange that succeeds
```

| 排序      | 含义                                            |
|-----------|-------------------------------------------------|
| `Relaxed` | 不同步（计数器等）                              |
| `Acquire` | 保证加载之后的同步                              |
| `Release` | 保证存储之前的同步                              |
| `AcqRel`  | 读-改-写操作的双向同步                          |
| `SeqCst`  | 全局顺序一致性（默认，安全）                    |

这些排序的含义与 C++20 相同；它们所属的内存模型（happens-before、哪些库操作会同步、数据竞争）见 [refs-lang 7.9](../../refs-lang/007-runtime/709-memory-model.md)。默认值是安全的 `SeqCst`。加载不能 release，存储不能 acquire：`a.store(x, Acquire)` 是编译错误（`StoreOrdering` 没有 `Acquire`；存储接受 `Relaxed`、`Release` 或 `SeqCst`）。compare-exchange 的失败排序是一种加载排序。变体不带前缀书写：由参数的类型选择枚举（[refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-bare-variants-from-the-expected-type)）。这三个类型不在 prelude 中，它们的名称也不与比较用的 `cmp::Ordering` 冲突。排序实参必须是编译期常量。弱排序应当谨慎使用，只用在性能关键的地方。

### 示例

最常见的用法是递增计数器，写法如下。

```nabi
static COUNTER: Atomic<i32> = Atomic::new(0);

task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
// result: COUNTER == 2 (no race)

let v = COUNTER.load(SeqCst);     //→ 2
```

### CAS（compare-and-swap）

原子操作不直接提供的更新，可以建立在 compare-and-swap 循环之上。下面的代码保留曾经存入的最大值。

```nabi
fn store_max(cell: Atomic<i64>, x: i64) {
    let cur = cell.load(Acquire);
    loop {
        if cur >= x { break; }
        match cell.compare_exchange(cur, x, AcqRel, Acquire) {
            Ok(_)       => break,
            Err(actual) => cur = actual,          // another task got there first; retry with its value
        }
    }
}
```

元素类型必须是 `AtomicPrimitive`：`List<T>` 这样的集合不能放进 `Atomic`。这个 trait 是密封的（[refs-lang 6.2.15](../../refs-lang/006-meta-system/602-built-in-attributes.md#6215-sealed-impls-only-in-the-defining-package)）：上面列出的类型就是它的全部实现，在其他包中写 `impl AtomicPrimitive` 是错误。共享的集合放进 `Mutex<T>`（2.16.3）。

## 2.16.3. `Mutex<T>`：互斥锁

只用原子操作难以表达的共享状态，使用 `Mutex<T>`。它内部由一个 futex 和载荷组成；用 `using` 绑定的守卫负责释放锁。

```nabi
struct Mutex<T> {
    // internal: futex + payload
}

impl<T> Mutex<T> {
    const fn new(value: T) -> Mutex<T>;
    const unsafe fn new_unshared(value: T) -> Mutex<T>;   // new without the hand-off copy

    fn lock(self) -> MutexGuard<T>;            // blocks the thread
    fn lock_async(self) -> LockFuture<T>;      // await m.lock_async(): suspends only the task (Future<Output = MutexGuard<T>>)
    fn try_lock(self) -> MutexGuard<T> | none; // non-blocking
    fn with<R>(self, f: fn(T) -> R) -> R;      // lock, f(value), unlock
}

struct MutexGuard<T> { /* RAII guard */ }

impl<T> Drop for MutexGuard<T> {
    fn drop(ref self) {
        // futex_wake + release the lock
    }
}

impl<T> MutexGuard<T> {
    fn get(self) -> T;                  // a scoped view of the protected value
    fn get_clone(self) -> T where T: Clone;   // a deep copy, free to outlive the lock
    fn set(self, value: T);             // replaces the value
    fn update(self, f: fn(T) -> T);     // reads, transforms and writes back
}
```

守卫不转发它所保护的值的方法：对 `MutexGuard<List<i32>>` 调用 `g.push(1)` 是错误。要通过 `get`、`set` 和 `update` 访问该值。

`m.with(f)` 加锁，以受保护的值调用 `f`，在 `f` 返回时解锁，并返回其结果：`log.with(|v| v.push(x))`、`let n = log.with(|v| v.len())`。该值像 `get()` 的视图一样借给 `f`，适用 `ThreadLocal::with` 的规则：它不得离开 `f`（被返回、存储、发送，或被被调用方或闭包保留），所以 `f` 不能是 `async`（E0136）；要返回副本（`.clone()`）或由它计算出的结果。要跨语句或跨 `await` 持有锁，请保留守卫（`using g = m.lock()`）。`RwLock::read_with(f)` 和 `write_with(f)` 分别在读锁和写锁下与此相同。

### 示例

用 `using` 绑定的守卫在离开块时被 `drop`，从而释放锁。不用 `using` 接收的守卫不会自动释放，所以守卫必须始终用 `using` 绑定。守卫来自函数调用、方法调用或 `await`。接收到的守卫只能用于初始化绑定（`let`、模式、`using`）、赋给变量，或交给调用方（`return`、函数的值）。把它传给函数、在它上面调用方法、把它存入字段或集合，或丢弃它，都是编译错误。用 `using` 绑定的守卫可以传给函数；不用 `using` 绑定的守卫不会因为被传递而释放。用普通 `let`（或 `if let`）绑定一个永远不会被 `using` 释放的守卫，或丢弃 `lock` 返回的守卫，都是编译错误。

`Mutex::new(value)` 接管 `value`：之后使用调用方的句柄是编译错误；同样是编译错误的还有：在守卫的 `using` 块结束后使用 `get` 返回的值或 `set` 给出的值；在守卫持有期间（直到 `set` 给锁一个新值为止）交出 `get` 返回的值；把它返回、yield、作为块的值，或交给在块之后运行的协程（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）。抛出持有它的错误（`throw Leak { xs: g.get() };`）也是如此：捕获方在锁释放之后才运行。`get` 返回的是有作用域的视图：它与守卫一样既不是 `Send` 也不是 `Sync`，把它传给会保留它的被调用方（存入 `Cell`、字段、捕获、线程局部变量）会在调用处产生编译错误 E0106，其帮助信息会给出 `get_clone()`。`get_clone()` 返回深拷贝（`T: Clone`），可以比锁存活得更久。请在 `using` 块内处理视图，或保留一份 `get_clone()`。守卫是 `!Sync`，只在其 `using` 块内有效：在块结束后使用它的副本（放在集合、闭包、字段中）是编译错误；要在另一个线程上运行的闭包中使用该值，先取深拷贝（`let v = g.get_clone();`），再捕获 `v`。`RwLockReadGuard` 和 `RwLockWriteGuard` 有同样的 `get` 和 `get_clone`。在运行时，`new`、`set` 和 `update` 存储该值带入的集合的副本（2.13.9）；`new_unshared` 跳过复制，由调用方保证没有其他东西能到达 `value` 的集合（例如刚刚创建的值）。

```nabi
static STATE: Mutex<List<i32>> = Mutex::new([]);

task::spawn(async {
    using g = STATE.lock() {
        g.update(|xs| { xs.push(42); xs });
    }   // ← g.drop() → lock released ([2.13 marker](213-marker.md))
});
```

### 避免死锁：`try_lock`

为了避免死锁，可以用 `try_lock` 做非阻塞的尝试，而不是阻塞。

```nabi
match STATE.try_lock() {
    v: sync::MutexGuard<List<i32>> => using g = v { g.update(|xs| { xs.push(1); xs }); },
    none => { log("locked, skip"); },
}
```

### 中毒（poisoning）：**无**

Nabi 的 Mutex 与 Rust 的 PoisonError 模型不同。即使发生 panic，锁也会自动释放，下一个任务可以正常进入。这样设计有两个原因。

1. panic 不能被捕获，会结束进程，所以其他任务永远不会观察到 panic 中途的锁状态。唯一的例外是 C 调用的函数上的 `#[on_panic]`：在那里被跳过的栈帧不会运行它们的 drop，它们锁住的 `Mutex` 会保持锁定（持有锁时请返回错误值，而不是触发 panic）。当错误通过 `throw` 传播出 `using` 块时，[`using` 会调用 Drop](213-marker.md#2134-drop释放时的清理)，所以由 `MutexGuard` 的 drop 负责释放锁。
2. 中毒看似保守，却给用户带来很大负担，而且在大多数情况下是误报。

如果确实必须在带着错误离开之前恢复一致性，请显式处理：

```nabi
using g = MUTEX.lock() {
    try unsafe_op(g) catch e {
        g.set(known_good_state());
        throw e;
    };
}
```

## 2.16.4. `RwLock<T>`：读写分离

读远多于写的地方更适合用 `RwLock<T>`。把多个读者与单个写者分开，可以提高吞吐量。

```nabi
struct RwLock<T> { /* ... */ }

impl<T> RwLock<T> {
    const fn new(value: T) -> RwLock<T>;

    fn read(self) -> RwLockReadGuard<T>;        // multiple readers OK
    fn write(self) -> RwLockWriteGuard<T>;      // single writer

    fn try_read(self) -> RwLockReadGuard<T> | none;
    fn try_write(self) -> RwLockWriteGuard<T> | none;

    fn read_async(self) -> ReadFuture<T>;       // await l.read_async(): suspends only the task
    fn write_async(self) -> WriteFuture<T>;

    fn read_with<R>(self, f: fn(T) -> R) -> R;  // as Mutex::with, under the read lock
    fn write_with<R>(self, f: fn(T) -> R) -> R; // as Mutex::with, under the write lock
}

struct RwLockReadGuard<T> { /* RAII guard: releases the read lock */ }

impl<T> RwLockReadGuard<T> {
    fn get(self) -> T;                  // a scoped view of the protected value
    fn get_clone(self) -> T where T: Clone;   // a deep copy, free to outlive the lock
}

struct RwLockWriteGuard<T> { /* RAII guard: releases the write lock */ }

impl<T> RwLockWriteGuard<T> {
    fn get(self) -> T;
    fn get_clone(self) -> T where T: Clone;
    fn set(self, value: T);             // replaces the value
    fn update(self, f: fn(T) -> T);     // reads, transforms and writes back
}

// both guards: Drop (releases the lock), Debug, !Sync
```

它适合配置或缓存这类以读为主的状态；对于很少写入的工作负载，它的吞吐量高于 Mutex。

```nabi
static CONFIG: RwLock<Config> = RwLock::new(Config::default());

using c = CONFIG.read() {        // concurrent with other readers OK
    process(c);
}   // ← read guard drop, lock released

using w = CONFIG.write() {       // briefly blocks all readers
    w.set(Config::load_from_file());
}
```

只有当 `T` 是 `Send + Sync` 时，`RwLock<T>` 才是 `Sync`（[2.16.5](#send--sync-标记)），所以这里的 `Config` 必须是 `Sync`。放在其中的集合是冻结的（`RwLock<FrozenDict<K, V>>`）：读者共享它，写者整体替换它（2.16.4b）。

与 `Mutex` 一样，`RwLock::new(value)` 接管 `value`：之后使用调用方的句柄是编译错误；同样是编译错误的还有：在守卫的 `using` 块结束后使用其 `get` 返回的值或 `set` 给出的值；在守卫持有期间（直到 `set` 给锁一个新值为止）交出 `get` 返回的值；从块内返回它（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）。请在 `using` 块内处理它，或保留一份 `clone()`。与 `Mutex` 一样，`new` 以及写守卫的 `set` 和 `update` 存储副本。

`RwLock` 的守卫也适用同样的规则：用普通 `let`（或 `if let`）绑定 `read` / `write` 的守卫而永远不被 `using` 释放，或丢弃它，都是编译错误。

## 2.16.4a. `Once` / `Condvar`

两个与 Rust 形式相同的原语。没有中毒。

```nabi
struct Once { /* ... */ }
impl Once {
    const fn new() -> Once;
    fn call_once(self, f: fn());          // runs f only the first time; concurrent callers wait until it finishes
    fn is_completed(self) -> bool;        // true once a call_once has finished
}

struct Condvar { /* ... */ }
impl Condvar {
    const fn new() -> Condvar;
    fn wait<T>(self, guard: MutexGuard<T>) -> MutexGuard<T>;   // releases the lock and waits, relocks on wakeup
    fn wait_async<T>(self, guard: MutexGuard<T>) -> CondvarWait<T>;   // g = await cv.wait_async(g): suspends only the task
    fn notify_one(self);
    fn notify_all(self);
}
```

`wait` 可能被虚假唤醒，所以要在循环中重新检查条件。`notify_one` 至少唤醒一个等待者，可能唤醒不止一个异步等待者。

### 异步锁与取消

`lock_async` / `read_async` / `write_async` / `wait_async` 的 future 在被取消的轮询中不会获取锁（[2.18.4](218-task.md#2184-协作式取消taskcancel)）。被取消的 `wait_async` 按如下方式结束。

- 如果互斥锁空闲，它会重新获取锁，于是调用方的 `using` 守卫照常释放它。
- 如果另一个任务持有锁，它不获取锁就离开，被丢弃的守卫不会释放该互斥锁。

## 2.16.4b. `Frozen<T>`：只读共享

只创建一次、之后只读的值（配置、信任锚、表）通过 `Frozen<T>` 共享。任何线程都可以无锁读取它。

```nabi
struct Frozen<T> { /* ... */ }   // Sync when T: Send

impl<T> Frozen<T> {
    const fn new(value: T) -> Frozen<T>;   // keeps a copy no one else reaches
    fn get(self) -> T;                       // the reader's own copy
}
```

对于单个大集合，请改为冻结它（[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s只读集合类型)）：`FrozenList` / `FrozenDict` / `FrozenSet` 被每个线程原地读取，完全不复制。`Frozen<T>` 用于由多个部分组成的值，其中一些部分是集合，每个读者可以在自己的副本中修改它们。

`new` 像移交（hand-off）一样保留 `value` 所到达的集合的副本（[2.13.9](213-marker.md#2139-send--sync线程边界标记)），所以外部没有任何东西能到达它所持有的内容。`get` 给每个读者一份自己的副本：修改它不会改变其他读者看到的内容。`Sync` 的部分（`Mutex`、自己加锁的缓存、不可变 `Sync` 类型的值）不被复制，保持共享，所以一次 `get` 只需复制该值的其他集合。请把大的只读部分排除在这份副本之外：把它们作为 `Sync` 类型持有，或放在单独的 `Frozen` 字段中，在需要的地方读取。

```nabi
struct Settings {
    name: str;
    limits: List<i64>;               // copied by each get: small
    table: Frozen<List<Entry>>;      // shared: read with table.get() where needed
}

static SETTINGS: Frozen<Settings> = Frozen::new(load());
```

标准库的 TLS `Certificate` 就是这样的类型：解析后永不改变（它的访问器给出副本），所以 `Frozen<ClientConfig>` 的 `get` 共享其信任锚。

### 以读为主的状态：快照

被许多线程读取、偶尔改变的状态（服务器的路由表、缓存、会重新加载的设置），是放在锁后面的冻结快照。读者取得当前快照（`Sync` 值可以离开锁的闭包或 `using` 块），在不持有锁的情况下读取它；写者从副本构建下一个快照并放进去。已经取得旧快照的读者继续读取旧快照，当没有读者再使用它时由回收器释放。

```nabi
static ROUTES: Mutex<FrozenDict<str, Route>> = Mutex::new(FrozenDict::new());

fn lookup(path: str) -> Route | none {
    let routes = ROUTES.with(|r| r);          // the current snapshot; the lock is held only for this
    routes.get(path)
}

fn add_route(path: str, r: Route) {
    using g = ROUTES.lock() {
        let next = g.get().to_dict();       // a copy to change
        next[path] = r;
        g.set(next.freeze());
    }
}
```

`RwLock<FrozenDict<..>>` 用 `read_with` / `write` 以同样的方式工作；如果锁保护的只是取快照这一操作，`Mutex` 就够了。

## 2.16.5. `Cell<T>`：单线程内部可变性

最后，`Cell<T>` 提供只在单个任务内有效的内部可变性。

```nabi
struct Cell<T> { /* ... */ }

impl<T> Cell<T> {
    const fn new(value: T) -> Cell<T>;
    fn get(self) -> T            where T: Copy;
    fn set(ref self, value: T);
    #[discardable]
    fn replace(ref self, value: T) -> T;
    #[discardable]
    fn take(ref self) -> T       where T: Default;
}
```

`Cell<T>` 表示线程局部的内部可变性，所以跨线程不安全。它只能在单个任务内共享，没有锁使它非常快。

```nabi
let c: Cell<i32> = Cell::new(0);
c.set(42);
let v = c.get();          //→ 42

// sharing across tasks: compile error (Cell has no Send / Sync)
task::spawn(async { c.set(100) });       // ERROR: Cell is not cross-task safe
```

### `Send` / `Sync` 标记

`Send` / `Sync` 是 [2.13.9](213-marker.md#2139-send--sync线程边界标记) 中的自动标记 trait。本章的类型遵循以下策略：

- `Atomic<T>`：Send + Sync，包括 `Atomic<Ptr<T>>`（单元中保存的是地址）
- `Mutex<T>`：`T: Send` 时为 Send + Sync
- `RwLock<T>`：`T: Send` 时为 Send；`T: Send + Sync` 时为 Sync。它的读者同时共享该值，所以集合（`!Sync`）的 `RwLock` 不能在线程之间共享；请使用 `Mutex<T>`。
- `MutexGuard<T>`、`RwLockReadGuard<T>`、`RwLockWriteGuard<T>`：Send，`!Sync`。守卫只在其 `using` 块内有效，其 `get` 返回的视图既不是 `Send` 也不是 `Sync`（由 E0106 追踪，2.16.3）
- `Frozen<T>`：`T: Send` 时为 Sync（2.16.4b）
- `FrozenList<T>`、`FrozenSet<T, S>`：元素（及哈希器）为 `Send + Sync` 时为 Sync；`FrozenDict<K, V, S>` 在 `K`、`V` 和 `S` 都是时为 Sync（[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s只读集合类型)）
- `Cell<T>`：两者都不是（仅限单个任务）。上面的 `task::spawn` 之所以是错误，是因为 `async` 块捕获了 `Cell`，因而不是 `Send`

## 2.16.6. 模块映射

每个项都定义在 `core::sync` 中，这是它唯一的路径（[0.1.6](../000-meta/001-tier-model.md#016-每个项只有一条路径)）；没有按类型划分的子模块。

| 类型         | 模块              | 内部结构                         |
|--------------|-------------------|----------------------------------|
| `Atomic<T>`  | `core::sync` | 原子指令（`nabi.atomic.*`）           |
| `Mutex<T>`   | `core::sync` | futex / 等待队列（按目标）            |
| `RwLock<T>`  | `core::sync` | futex + 读者计数器                    |
| `Cell<T>`    | `core::sync` | 普通包装，无锁                        |
| `MutexGuard` | `core::sync` | 通过 Drop 实现 RAII                   |
| `RwLockReadGuard`, `RwLockWriteGuard` | `core::sync` | 通过 Drop 实现 RAII |
| `LockFuture`, `ReadFuture`, `WriteFuture` | `core::sync` | `lock_async` / `read_async` / `write_async` 的 future |
| `Once`       | `core::sync` | 状态字 + 等待队列                     |
| `Condvar`, `CondvarWait` | `core::sync` | 等待队列；`CondvarWait` 是 `wait_async` 的 future |
| `Frozen<T>`  | `core::sync` | 其他人无法到达的副本                  |
| `LoadOrdering`, `StoreOrdering`, `UpdateOrdering` | `core::sync` | 内存排序（2.16.2） |
| `AtomicPrimitive` | `core::sync` | 密封的标记 trait                 |
