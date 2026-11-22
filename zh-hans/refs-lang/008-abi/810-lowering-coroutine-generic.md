# 8.10. 降级：协程与泛型

本节定义[生成器](../004-functions/411-generators.md)、[async](../004-functions/412-async.md) 和[流](../004-functions/413-stream.md)的协程降级，以及[泛型](../002-types/210-generics.md)和[常量泛型](../002-types/211-const-generics.md)的单态化降级。

## 8.10.1. 状态机

[协程](../007-runtime/708-coroutine.md)被降级为编译器构建的状态机。

1. HIR：`yield` / `async` / `async yield` 块像闭包一样被提升为主体函数。块原来的位置留下一个创建帧的表达式。
2. MIR：主体函数被转换为恢复函数。每个挂起点获得一个状态编号，跨越该点仍然存活的局部变量被移入帧字段。
3. 帧类型为 `(i32 state, locals..)`，带有 ScanTable。帧通过普通分配路径创建（[8.9.9](809-lowering-control.md#899-gc-分配快速--慢速路径)）。
4. 恢复函数的入口按 `state` 分派：一串把它与各状态编号比较的双路分支（8.9.3）。

## 8.10.2. 生成器（`yield fn`）

```nabi
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

概念上的恢复函数（`next`）如下：

```
next(frame) -> i32 | none:
    switch frame.state:
      0: frame.i = 0; goto loop
      1: frame.i = frame.i + 1; goto loop
      2: return none
    loop:
      if frame.i < frame.n:
          frame.state = 1
          return frame.i          // yield i
      frame.state = 2
      return none
```

`count_up_to(n)` 分配帧，并返回一个 `state = 0` 的包装类型（`GenIter<i32>`）。

## 8.10.3. Async（`async fn`）

```nabi
async fn fetch() -> Data {
    let r = await op1();
    process(r)
}
```

```
poll(frame, cx) -> Poll<Data>:
    switch frame.state:
      0: frame.fut = op1(); frame.state = 1; goto await1
      1: goto await1
    await1:
      match frame.fut.poll(cx):
        Pending  => return Pending
        Ready(r) => frame.state = 2; return Ready(process(r))
```

`await` 把目标 future 存入帧并轮询它。若结果为 Pending，就保持状态不变并返回 Pending，下一次轮询从同一点继续。

## 8.10.4. 流（`async yield fn`）

[流](../004-functions/413-stream.md)结合了上述两者。同一个帧的状态编号把 `yield` 点和 `await` 点一起计数。`poll_next` 在 `yield` 处返回 `Ready(value)`，在 Pending 的 `await` 处返回 `Pending`，在结束时返回 `Ready(none)`。

## 8.10.5. 帧分配

每次协程调用都会分配一个帧。对于由调用者运行至完成的协程，没有专门的省略机制；从不逃逸的帧可以像任何分配一样被消除（[7.2.12](../007-runtime/702-allocation.md#7212-分配消除)）。

## 8.10.6. 泛型单态化

[泛型](../002-types/210-generics.md)的单态化为调用处的每个类型元组生成一个独立实例。

```nabi
fn id<T>(x: T) -> T { x }

let a = id(42i32);     // id<i32>
let b = id(3.14f64);   // id<f64>
```

单态化之后，程序中每个实例对应一个函数。

```
fn id<i32>(x: i32, tls) -> i32:
    return x

fn id<f64>(x: f64, tls) -> f64:
    return x

fn main(tls):
    a = call id<i32>(42, tls)
    b = call id<f64>(3.14, tls)
```

每个具体的类型元组都会生成一个独立函数（`id<i32>` 代表其修饰后的符号，见下文）。

### 单态化何时发生

单态化按以下流程进行。

```
nabi-hir -> specialize(hir, type_tuple) -> nabi-mir -> machine code
```

在 [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 模式下，先收集所有调用处，再批量运行 [specialize](../000-meta/007-glossary.md#specialize)。在 [JIT](../000-meta/007-glossary.md#jit-just-in-time) 模式下，specialize 在首次调用时运行，结果被缓存（见[执行引擎与用户界面](../000-meta/001-execution-modes.md)）。两种模式共用同一套 specialize 机制。

### 符号命名

[编译流水线](../000-meta/002-pipeline.md)使用的符号修饰规则为：

```
_Y<pkg_hash_8>_<path>_<name>_<typehash_16>
```

包以哈希表示，因此位于包根的函数 `<path>` 为空：`id<i32>` 会变成类似 `_Y3f8a1c2b__id_a7c91d8e44b2f190`（示意）的形式，这个符号就是该函数在目标文件中的名字。非泛型项的 `<typehash>` 为空（`_Y3f8a1c2b__main_`）。标识符中的字面 `_` 转义为 `_0`（[9.5](../009-tools/905-symbol-mangling.md)）。

## 8.10.7. `const` 泛型单态化

[常量泛型](../002-types/211-const-generics.md)的示例：

```nabi
fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }

let r = dot([1.0, 2.0, 3.0], [4.0, 5.0, 6.0]);  // T=f64, N=3
```

它被单态化为如下形式。

```
fn dot<f64, 3>(a: [f64; 3], b: [f64; 3], tls) -> f64:
    ; ... a loop of 3 iterations
```

由于 `const` N=3 是编译期常量，循环边界和数组大小在该实例中都是常量：优化器会折叠依赖于它们的部分。

## 8.10.8. 泛型 trait 分派

```nabi
fn process<T: Describe>(x: T) -> str {
    x.describe()
}
```

具体类型 T 在调用处确定，因此对于 `T = Point`，降级结果为：

```
fn process<Point>(x: (f64, f64), tls) -> (ptr, usize):
    ; call Point's Describe::describe directly
    r = call Point.describe(x, tls)
    return r
```

`Point` 以其值结构体传递（8.8.3），`str` 以其 `(ptr, usize)` 对传递（8.8.1）。不存在通过虚表的间接调用；一切都是直接函数调用。结合 [trait 不能作类型](../002-types/212-trait-as-type-banned.md)的规则，这一模型保证虚表开销为零。

## 8.10.9. 和类型上的分支分派

对[和类型](../002-types/206-sum-types.md)的匹配被降级为标签测试，即一串双路分支（8.9.3）。

```nabi
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),
    s: str => use_str(s),
}
```

在 64 位目标上，`i32 | str` 是 `(i32, [u64; 2])`：标签 0 为 `i32`，标签 1 为 `str`，载荷占两个字，因为 `str` 是持有 GC 引用的双字对（8.8.6）。以基本块表示：

```
bb0:
    if v.0 == 0 then bb_use_int else bb_test_str
bb_test_str:
    if v.0 == 1 then bb_use_str else bb_unreachable

bb_use_int:
    n = v.1 as i32                 ; the payload read as i32
    call use_int(n, tls)
    goto bb_end

bb_use_str:
    s = v.1 as str                 ; the payload read as the (ptr, usize) pair
    call use_str(s, tls)
    goto bb_end

bb_end:
    ...
```

该匹配是穷尽的，因此永远不会到达 `bb_unreachable`。标签检查和分支全部是静态代码。

## 8.10.10. trait 方法的 trait 约束分派

```nabi
fn add_all<T: Add<T, Output = T>>(items: List<T>) -> T {
    let r: T = ...;
    for x in items {
        r = r.add(x);
    }
    r
}
```

单态化之后，每个调用处的每个 T 都会生成一个独立函数。对于 `T = i32`，`r.add(x)` 解析为对 i32 的 `Add::add` 的直接调用，该调用还可以被内联。

## 8.10.11. 小结

编译器自行把协程降级为状态机（8.10.1）。泛型降级通过单态化为每个具体类型生成独立代码，不使用虚表。

这两种机制是本 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 部分中最复杂的降级；其余部分，即[降级：数据类型与表达式](808-lowering-types.md)和[降级：控制流与 GC 集成](809-lowering-control.md)，接近于到机器操作的直接映射。

第 8 章（运行时 ABI / 降级 / 代码生成）到此结束。
