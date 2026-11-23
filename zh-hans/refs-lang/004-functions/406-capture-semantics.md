# 4.6. 捕获语义

[闭包](405-closure.md)可以捕获外部环境中的变量。本章定义决定捕获是复制值还是持有引用的规则。

## 4.6.1. 默认：值复制

> [!IMPORTANT]
> 捕获的默认含义是**值复制**。创建闭包时，会对外部变量的值取一个快照。

```nabi
let count = 0;
let f = || { count + 1 };         // the value 0 of count is copied
count = 10;
f();                               // 1 (f holds 0)
```

这一决定有三个动机。第一是显式性：捕获方式在代码中清晰可见。第二是阻断副作用：它避免了闭包悄悄修改外部变量的陷阱。第三是对科学和数值代码友好：它避开了 Julia 中引用捕获被装箱为动态槽、导致性能下降 100 倍的陷阱。

```
capture_list    ::= "[" capture_item ("," capture_item)* ","? "]"
capture_item    ::= "ref" capture_name | "move" capture_name | capture_name
capture_name    ::= identifier | "self"
```

`self` 指方法的接收者。`[ref self]` 只允许用在 `ref self` 方法中，此时它共享接收者（[4.7.4](407-ref-keyword.md#474-闭包捕获列表)）；按值的 `self` 按值捕获。

## 4.6.2. 引用捕获：`[ref x]` 捕获列表

要捕获对外部变量绑定的引用，使用显式的捕获列表。

```nabi
let count = 0;
let inc = [ref count] || { count = count + 1 };
let dec = [ref count] || { count = count - 1 };

inc(); inc(); dec();
// count = 1
```

列表写在闭包定义之前，形式为 `[ref var, ref var, ...]`。列表中的变量存放在一个共享单元中（装箱）；外部作用域和闭包使用同一个单元，所以在闭包内重新赋值会反映到外部变量上。该单元是 GC 对象，所以闭包可以比变量的作用域活得更久。不在列表中的变量照常按值捕获。

### 多个捕获与混合捕获

```nabi
let counter = 0;
let config = load_config();

let handler = [ref counter] || {
    counter = counter + 1;        // ref: reflected in the outer variable
    print(config.version);         // value: snapshot of config at capture time
};
```

只有捕获列表中列出的变量是 `ref` 捕获；其余变量自动按值捕获。

## 4.6.3. 捕获列表的词法分析

`[ref x] || body` 的词法分析如下：`[`、`ref`、`x`、`]` 构成带方括号的捕获列表，`||` 或 `|x|` 标志闭包的开始。

歧义由[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)的最长匹配规则和上下文消解。如果 `[...]` 之后紧跟 `||` 或 `|x|`，它就是捕获列表；否则它是普通的索引或数组字面量。

```nabi
let xs = [1, 2, 3];               // List literal
let f = [ref count] || { ... };   // capture list + closure
arr[ref count]                     // ERROR (E0010): ref cannot be used in indexing
```

## 4.6.4. 在主体中重新赋值外部变量 → 需要 `ref`

在闭包主体中试图重新赋值按值捕获的外部变量是编译错误（E0057）。

```nabi
let count = 0;
let f = || { count = count + 1 };
// ERROR (E0057): `count` is value-captured; to mutate the outer variable,
//                add `[ref count]` to the closure's capture list
```

这条规则引导用户有意识地写明 `ref` 捕获。

## 4.6.5. `ref` 简写（不存在）

```nabi
ref || { ... }                    // not adopted
```

没有把所有变量都按 `ref` 捕获的简写。捕获只能通过显式列表成为 `ref`，这样代码能清楚地表明哪些变量是按引用持有的。

## 4.6.6. 循环变量的自动处理

由于默认是值捕获，每次迭代的循环变量都保有自己的值。

```nabi
let fns: List<fn() -> i32> = [];
for i in 0i32..5i32 {
    fns.push(|| i);               // copies the value of i in each iteration
}
fns[0]();                          // 0
fns[4]();                          // 4
```

不需要 Go 1.22 那样的每次迭代独立作用域的单独规则；值捕获自动产生预期的含义。

## 4.6.7. 捕获结构体与引用句柄

值捕获把值复制进闭包。[`struct`](../002-types/203-struct.md) 是值类型，所以闭包持有它自己的副本；闭包内的字段写入不会影响外部变量。要影响外部变量，使用 `[ref p]`。对值捕获的字段写入是编译错误（E0057），与重新绑定捕获相同，帮助信息为 ``add `[ref p]` to the closure's capture list``。对普通数据（数字、`bool`、`str`，以及由它们构成且没有 `Drop` 的结构体、元组、数组和枚举）的值捕获或其字段调用 `ref self` 方法也是如此：闭包的每次调用都从自己的副本开始，所以修改会丢失（对带种子的生成器 `g`，`List::from_fn(4, |_| g.uniform(0.0, 1.0))` 会四次给出同一个值）。共享其状态的句柄和资源（`Cell`、`File`、套接字）不受影响。应写 `[ref g] |_| g.uniform(0.0, 1.0)`；如果只想在闭包内修改一个副本，先把它复制到一个局部变量中。

集合类型（`List`、`Dict`、`Set`、`StrBuf` 等）是引用句柄（[标准库 2.9](../../refs-standard-libraries/200-core/209-collections.md#引用句柄)）。值捕获复制的是句柄，所以对象是共享的。

```nabi
let xs = [1, 2, 3];
let f = || { xs.push(4) };        // copies the handle (same List)
f();
// xs == [1, 2, 3, 4]

let p = Point { x: 0.0, y: 0.0 };
let g = [ref p] || { p.x = 1.0 }; // ref capture: writes the outer p
g();
// p.x == 1.0
```

要得到集合的独立副本，调用 `.clone()`。

## 4.6.8. `ref` 捕获与对象修改

```nabi
let xs = [1, 2, 3];

// value capture: mutating the object is OK, reassigning the binding is not
let f = || {
    xs.push(4);                   // OK: object mutation
    // xs = [];                    // ERROR: binding reassignment
};

// ref capture: reassigning the binding is also allowed
let g = [ref xs] || {
    xs = [];                      // OK: the outer xs becomes []
};
```

对引用句柄而言，值捕获与对象修改可以并存，因为对象是共享的。需要重新赋值绑定本身时，才使用 `ref` 捕获。

这一区分与 [`impl`](../002-types/209-impl.md) 的 `self` 和 `ref self` 模型相同。

## 4.6.9. 并行与并发上下文中的附加约束

`fn() + Send` 闭包（传给 `task::spawn_blocking`、`par::for_each`、`thread::spawn` 等的闭包）的每个普通捕获都必须是 `Send + Sync`（[4.4.6](404-function-type.md#send--sync-约束)）：闭包与创建它的代码共享该捕获。`[move x]` 捕获只属于闭包，只需要 `Send`（[4.6.11](#4611-移动捕获move-x-捕获列表)）；`+ Sync` 闭包（`par::for_each` 的闭包，每个工作线程都会调用它）要求每个捕获都是 `Sync`。`[ref x]` 单元会被闭包和创建它的代码同时写入，所以无论 `x` 是什么类型，它都绝不是 `Send` 或 `Sync`：带 `[ref]` 捕获的闭包（或捕获了这种闭包的闭包，或由 `async [ref x] ||` 闭包产生的 future）不适用于任何 `+ Send` / `+ Sync` 槽。同一线程内的使用（`spawn_local`、普通调用）不受影响。集合类型是 `!Sync`（[标准库 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync线程边界标记)），同样不能以这种方式按值捕获。

```nabi
let buf: List<i32> = [];
par::for_each(xs, [ref buf] |x| { buf.push(x) });
// ERROR (E0085): a [ref buf] cell cannot cross threads
```

应改用折叠（`par::fold(xs, 0, |a, b| a + b)`），通过 `Mutex` / `RwLock` / `Atomic` / 通道共享，或者从线程返回值（`join()`）。只有被派生的代码需要的值，就把它移入：`thread::spawn([move buf] || buf.len())`。

以这种方式共享的值属于它的新所有者：在 `Mutex::new(buf)` 或 `tx.send(buf)` 之后使用 `buf` 是编译错误（E0105，[标准库 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync线程边界标记)）。应通过锁使用它，或者交出 `buf.clone()`。

```nabi
let m = Mutex::new(buf);
buf.push(1);
// ERROR (E0105): `buf` was moved into the `Mutex` at line 1
//   help: use it through the lock: using g = m.lock() { g.get().push(1) }
```

详细的并行与并发规则由标准库规范定义；本章只涉及语言层面的捕获机制。

被捕获的 `Drop` 资源（`File`、套接字、通道 `Sender`）不会被复制：捕获把它移入闭包或 `async` 块，之后在外部使用它是错误（[标准库 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop释放时的清理)）。对释放不会失败的值（`Sender`），`async` 块的任务结束时释放其捕获的内容；关闭可能失败的被捕获值（`File`）必须在块内关闭（`try f.close()` 或 `using`），否则该块是错误（E0134）。如果外部也要保留该值，就捕获一个副本：`x.clone()`，对 `File` 则用 `try x.try_clone()`。

## 4.6.10. 被捕获变量的生命周期

被捕获的变量与闭包值存活同样长的时间。即使闭包离开了外部作用域，被捕获的值仍保存在闭包对象中。

```nabi
fn make_counter() -> fn() -> i32 {
    let count = 0;
    [ref count] || { count = count + 1; count }
    // count is captured by ref; kept until the closure goes away
}

let c = make_counter();
c();                               // 1
c();                               // 2
c();                               // 3
```

在这个示例中，外部函数返回后，`count` 仍附着在闭包对象上存活。这一生命周期由 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 管理（[7. 运行时模型](../007-runtime/README.md)）。

## 4.6.11. 移动捕获：`[move x]` 捕获列表

`[move x]` 在创建闭包时把 `x` 交给闭包。它是 `[ref x]` 的对应物：`ref` 捕获与创建闭包的代码共享绑定，`move` 捕获则让那段代码不再拥有它。

```nabi
let rows: List<str> = load();
let t = thread::spawn([move rows] || rows.len());
rows.push("x");
// ERROR (E0105): `rows` was moved into a closure at line 2
//   help: the closure owns it now; use it inside the closure, ...
```

- 创建闭包时移交 `x` 的方式与通道发送相同（[标准库 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync线程边界标记)）：闭包得到 `x` 所能到达的每个集合的副本，之后使用 `x`（或指向其内容的其他句柄）是编译错误（`E0105`）。`Sync` 值（数字、`str`、由它们构成的数组）例外：闭包的副本与 `x` 相等，两者都可以使用。
- 移动捕获只需要 `Send` 就能让闭包跨线程：没有其他代码能访问它。所以集合、`Command` 或 `Child` 可以交给 `thread::spawn`、`task::spawn_blocking` 或 `par::Scope::spawn`。普通捕获仍遵循 [4.6.9](#469-并行与并发上下文中的附加约束) 的 `Send + Sync` 规则，`+ Sync` 槽（`par::for_each`，每个工作线程都会调用它）要求每个捕获都是 `Sync`。
- 拥有非 `Sync` 移动捕获的闭包只能移交一次。把它传给类型为 `fn(..) + Send` 且不带 `Sync` 的参数，无论是什么函数，都会移交它：这种参数是被调用者可能在另一线程上运行的参数（`thread::spawn`、`thread::Builder::spawn`、`task::spawn_blocking`、`task::spawn_blocking_with_arg` 的 `f`、`par::join`、`par::Scope::spawn`、`par::TryScope::spawn`，以及程序自身中带这种参数的任何函数）。此后再次使用它，无论以何种方式传递（变量、结构体或元组字段、列表元素、参数、`if` / `match` 分支、返回其实参的函数的结果），都是编译错误（`E0133`）；把它两次传给同一调用（`par::join(f, f)`）也是如此。类型未写明 `Sync` 的 `fn(..) + Send` 值可能拥有这种捕获，所以同样只能移交一次；`+ Send + Sync` 值，以及所有捕获都是 `Send + Sync` 的闭包，可以再次传递。调用闭包不会移交它：它可以被调用任意多次。要在其他线程上把同样的工作运行两次，就创建两个闭包，各自移入自己的 `.clone()`。
- 调用 `async [move ..]` 闭包同样不会移交它，但调用产生的 future 运行在闭包移入的内容之上：闭包和它的每个 future 是同一个所有者。它的 future 可以在这里依次等待（`await f(); await f();`）；移交闭包或其中一个 future（交给 `task::spawn`，或放进另一个闭包的 `[move f]`）会把其余的一并带走，之后使用它们是编译错误（`E0105` / `E0133`）。

```nabi
let xs: List<i32> = [];
let f = async [move xs] || { xs.push(1); xs.len() };
let a = await f();                 // OK: runs here
let t = task::spawn(f());          // the task owns f's moved values now
let b = await f();
// ERROR (E0133): a future of `f` was spawned as a task at line 4, and every future
//   of an `async [move ..]` closure shares what it moved in
```
- 必须关闭的值（`File`、套接字）移入闭包的方式与普通捕获已有的方式相同（[4.6.9](#469-并行与并发上下文中的附加约束)）；由闭包关闭它。
- `move` 和 `ref` 可以在一个列表中混用（`[move xs, ref n]`）；同一个名字不能两者兼是。列表中主体未使用的名字不会被捕获。

```nabi
let xs: List<i32> = [1, 2];
let n = 0;
let f = [move xs, ref n] || { n += 1; xs.push(n) };   // OK, on this thread
let g = [ref n, move n] || n;                    // ERROR (E0058): both ref and move
```

同一个值也可以作为实参传入：`task::spawn_blocking_with_arg(xs, |xs| xs.len())`（[标准库 2.18.1](../../refs-standard-libraries/200-core/218-task.md#2181-核心-api)）。
