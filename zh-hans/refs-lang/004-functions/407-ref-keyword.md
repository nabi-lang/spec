# 4.7. `ref` 关键字

`ref` 是一个关键字，表示“此位置上的改变会反映到调用者（或外部）的变量上”。对参数和调用实参，它以复制入 / 复制出（copy-in / copy-out）实现；在捕获列表中，它以共享单元实现。它不是一等的引用值。

## 4.7.1. 三种位置

```
1. function parameter:      fn f(ref x: i32) { x = ... }
2. closure capture list:    [ref x] before |...| { ... }
3. at the call:             f(ref x)
```

在这三种位置上，它的含义相同：改变会反映到外部变量上。`for ref x in xs` 以同样的方式绑定列表或数组的每个元素：改变会反映到元素上（[3.13.3](../003-expressions/313-loop-expressions.md#for-ref-elements-as-places)）。

## 4.7.2. 函数参数

```nabi
fn increment(ref n: i32) {
    n = n + 1;
}

fn swap(ref a: i32, ref b: i32) {
    (a, b) = (b, a);
}
```

在参数名前加 `ref`，在主体中对该参数重新赋值就会反映到调用者的变量上。参数本身的类型（上例中的 `i32`）与通常相同。

### 含义：复制入 / 复制出

`ref` 参数采用复制入 / 复制出。调用时，调用者位置（一个变量或变量的字段）上的值被复制到参数中；被调用者处理这个值；调用结束时，参数的最终值被写回调用者的位置。

- **写回时机**：不仅正常返回时会写回，被调用者抛出错误时也会写回。因此，`throw` 之前所做的改变对调用者可见。
- **排他性**：调用进行期间，不得通过其他路径访问同一位置，编译器会检查这一点。把同一位置两次作为 `ref` 传入，或者在一个实参中作为 `ref` 传入、又在另一个实参中读取它，都是编译错误（E0050）。不同的字段（`ref p.x`、`ref p.y`）互不重叠，是允许的。
- **位置**：实参必须指向一个变量或变量的字段。元素（`inc(ref xs[2])`）是编译错误：`` `ref` argument `n` of `inc` must name a variable or a field of one ``。应把元素复制到一个变量中，传入该变量，再存回去。`ref self` 接收者也可以是元素或 `Dict` 条目（[见下文](#meaning-of-ref-self---copy-in--copy-out)）。

```nabi
fn bump_then_fail(ref n: i32) throws {
    n = n + 1;
    throw MyError {};
}
let a = 1i32;
try bump_then_fail(ref a) catch _ { };   // a = 2 (written back on throw too)
swap(ref a, ref a);                       // ERROR (E0050): same place passed as ref twice
add(ref a, a);                            // ERROR (E0050): place passed as ref read by another argument
swap(ref p.x, ref p.y);                   // OK: distinct fields
```

> [!IMPORTANT]
> 调用者必须在调用处写 `ref`；不写就调用是编译错误。这样可以清楚地看出哪些变量可能改变。

## 4.7.3. 在调用处显式标注（C# 风格）

```nabi
let x = 5i32;
let y = 10i32;
increment(ref x);                 // x = 6
swap(ref x, ref y);
```

当且仅当函数参数声明为 `ref` 时，调用者也必须写 `ref`。省略 `ref`，或者为非 `ref` 参数写 `ref`，都是编译错误（E0050）。按路径调用 `ref self` 方法（`Counter::increment(ref c)`，4.7.5）时同样适用这条规则。

这条规则让调用处可以看出哪些变量可能改变，其动机与 C# 的 `ref` 模式相同。

```nabi
fn read(x: i32) { ... }
fn write(ref x: i32) { ... }

read(a);                          // plain: clearly a does not change
write(ref a);                      // ref: clearly a may change
```

## 4.7.4. 闭包捕获列表

```nabi
let count = 0;
let inc = [ref count] || { count = count + 1 };
```

按照[捕获语义](406-capture-semantics.md)中定义的 `ref` 捕获，捕获列表中的 `ref` 变量存放在一个共享单元中（装箱）。外部作用域和闭包读写同一个单元，所以闭包可以比变量的作用域活得更久。

`ref` 参数（包括 `ref self`）以同样的方式捕获（`[ref n]`、`[ref self]`）。函数主体和闭包共享该参数的同一个单元，所以通过闭包所做的写入是参数最终值的一部分，会被写回调用者（[4.7.2](#472-function-parameters)）。闭包在调用结束后所做的写入不会到达调用者。

```nabi
fn bump(ref c: Counter) {
    let add = [ref c] |x: i32| { c.count = c.count + x; };
    add(5);
}
let c = Counter { count: 1 };
bump(ref c);                      // c.count = 6
```

## 4.7.5. 方法接收者的例外

[`impl`](../002-types/209-impl.md) 中的方法可以有 `ref self` 参数。

```nabi
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}
```

这种情况下，调用处不需要写 `ref`。

```nabi
let c = Counter { count: 0 };
c.increment();                    // no 'ref': handled automatically for method receivers
```

这是方法接收者的特殊规则，与普通 `ref` 参数不同。方法调用语法本身就是自动[分派](../000-meta/007-glossary.md#dispatch)接收者的机制，额外的标记只会造成视觉干扰。

```nabi
fn external_inc(ref c: Counter) { ... }
external_inc(ref c);              // ordinary function calls require 'ref'
Counter::increment(ref c);        // calling a ref self method by path also requires ref
```

### `ref self` 的含义：复制入 / 复制出

`ref self` 的行为是复制入 / 复制出。调用时，接收者的值被复制到 `self` 中；调用结束时，`self` 的最终值被写回接收者的位置。

- **排他性**：调用进行期间，不得通过其他路径（另一个实参、带捕获的闭包等）访问接收者变量。如果编译器能看出这一点，就是编译错误（类似 Swift 的排他性规则）。
- **临时接收者**：如果接收者是临时值（`make().increment()`），写回的结果被丢弃。
- **元素与条目**：元素（`cs[0].increment()`，也包括字段之下的元素：`cs[0].pos.shift(1)`）或 `Dict` 条目（`d["a"].increment()`）通过索引写回：元素被复制出来，方法在副本上运行，副本再被存回（`cs[0] = c`），`throw` 之后也是如此。索引只求值一次。类型不提供索引存储（只有 `Index`）的元素是临时值。
- **静态变量与常量**：对 `static`（[5.3](../005-modules/503-static.md)）或 `const`（[5.2](../005-modules/502-const.md)）调用 `ref self` 方法是编译错误（E0057）：``cannot call `increment` on `S`: it takes `ref self`, and a static is never written``；常量的每次使用都是一个新值。应把它复制到一个变量中，再在变量上调用方法。

```nabi
c.increment();                     // copies c in, writes back to c at the end
c.add_from(|| c.count);            // ERROR (E0050): c accessed through a closure during the call
```

## 4.7.6. `ref` 与类型

`ref` 是修饰符，不是类型，不出现在任何[类型表达式](../002-types/201-type-expressions.md)中。

```nabi
let x: ref i32 = ...;             // ERROR: ref is not a type
struct S { x: ref i32; }          // ERROR: same reason
List<ref i32>                      // ERROR: same reason
```

因此 `ref` 只能出现在：

- 函数和方法的参数声明中。
- 闭包捕获列表中。
- 函数和方法的调用实参中。

由于 `ref` 不能用在返回类型中，Nabi 没有返回引用的函数（没有 Rust 中 `deref_mut` 那样的形式）。改变内部值的 API 以交换值的方式进行，如 `set` / `update(f)`。

## 4.7.7. 引用不能是一等值

```nabi
let r = ref x;                    // ERROR: ref itself is not an expression
let r: SomePointerType = ref x;   // ERROR
```

`ref` 不是产生值的表达式；它只是附加在调用、参数和捕获列表上的修饰词法单元。

协程函数（`yield fn`、`async fn`、`async yield fn`）不能有 `ref` 参数；这样写是编译错误（E0023）。调用返回后主体仍在继续运行，所以没有可以写回的时间点。

唯一的例外是 `async` 方法上的 `ref self`（[4.12.13](412-async.md#41213-async-on-methods)）：future 持有接收者，写回在 future 完成时进行。这样的调用要在发起处等待（`await r.read_one()`）。

出于同样的原因，带 `ref` 参数的函数不能作为函数值使用（`let f = increment;` 是错误），闭包也不能有 `ref` 参数。函数类型 `fn(..)` 没有表达 `ref` 的位置。

如果需要一等引用（例如 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)），使用[类型表达式](../002-types/201-type-expressions.md)中的裸指针类型（`Ptr<T>` 和 `ConstPtr<T>`），它们属于[标准库层级](../000-meta/005-standard-libraries-tiers.md)中的 `unsafe` 领域。

## 4.7.8. 小结：按值传递与按 `ref` 传递

```nabi
fn by_value(x: i32) { x = 10i32; }
fn by_ref(ref x: i32) { x = 10i32; }

let a = 5i32;
by_value(a);                      // a = 5 (unchanged)
by_ref(ref a);                    // a = 10
```

| 方面                                     | 按值传递（默认）        | 按 `ref` 传递（`ref`）|
|------------------------------------------|-------------------------|-----------------------|
| 在参数中重新赋值的影响                   | 仅限局部                | 调用者的变量          |
| 调用处的显式词法单元                     | 无                      | 必须写 `ref`          |
| 视觉信号                                 | 普通调用                | 标示可能的改变        |

## 4.7.9. 结构体与引用句柄

`struct` 是值类型：按值传递会复制它，所以被调用者中的字段写入是局部的。要改变调用者的结构体，就以 `ref`（或 `ref self`）接收它，或者返回新值。引用句柄（`List`、`Dict` 等，[标准库 2.9](../../refs-standard-libraries/200-core/209-collections.md#reference-handles)）作为句柄被复制，所以对其元素的改变对调用者可见。

```nabi
fn set_x(p: Point) { p.x = 10.0; }          // local copy: the caller's p is unchanged
fn set_x_ref(ref p: Point) { p.x = 10.0; }  // written back to the caller's p
fn fill(xs: List<i32>) { xs.push(1); }      // same List: the caller sees it
fn reset(ref xs: List<i32>) { xs = []; }    // the caller's variable holds a new List
```

| 场景                                              | 需要的关键字                    |
|---------------------------------------------------|---------------------------------|
| 改变调用者结构体的字段                            | `ref`                           |
| 改变集合的元素                                    | 普通的按值传递即可              |
| 让调用者的变量持有另一个值                        | `ref`                           |

## 4.7.10. 同时对两个绑定使用 `ref`

```nabi
fn swap(ref a: i32, ref b: i32) {
    (a, b) = (b, a);
}

let x = 1i32;
let y = 2i32;
swap(ref x, ref y);               // x = 2, y = 1
```

如上所示，一次调用可以有多个 `ref` 参数。

在调用处把同一位置作为两个 `ref` 传入，违反 4.7.2 的排他性规则，是编译错误。

```nabi
swap(ref x, ref x);               // ERROR (E0050): same place passed as ref twice
```
