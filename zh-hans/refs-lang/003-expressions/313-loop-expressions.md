# 3.13. 循环表达式

重复执行用三种形式表达：`loop`、`while` 和 `for`。三者都是表达式；本文档定义它们各自的语法和值的求值规则。

```
loop_expr   ::= label? "loop" block
              | label? "while" expr block
              | label? "for" pattern "in" expr block
              | label? "for" "await" pattern "in" expr block
              | label? "for" "ref" identifier "in" expr block
label       ::= "'" identifier ":"
break_stmt  ::= "break" label_target? expr? ";"
continue_stmt ::= "continue" label_target? ";"
label_target ::= "'" identifier
```

> [!NOTE]
> `in` 是上下文关键字，只在 `for pat in` 位置是关键字（[1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)）。

`for await` 的含义和用法在[流](../004-functions/413-stream.md)中定义。它从异步流中逐个接收项，并对每一项运行主体。

标签（`'name:`）的含义在 [3.13.4 标签](#3134-labels-controlling-nested-loops)中定义。带标签的**块**（`'b: { ... }`）在[块表达式](309-block-expressions.md)中定义。

## 3.13.1. `loop`：无限重复

```nabi
loop {
    ...
}
```

`loop` 关键字之后跟一个块主体，无限重复。只能通过 `break`、`return` 或 `throw` 退出。

### 用 `break value` 返回值

```nabi
let v = loop {
    let x = compute();
    if x.converged { break x; }
};
```

如果 `break` 之后跟一个表达式，它的值就是 `loop` 表达式的结果。如果循环像 `break;` 那样不带值退出，`loop` 的结果为 unit。与分支一样，一个 `loop`（或带标签的块）的所有 `break` 值必须是同一种类型：某个 `break` 值的和类型包含其他所有 `break` 值的类型时，该和类型就是这一类型，与哪个 `break` 在前无关（当 `x: i32 | none` 时，`break none` 和 `break x` 得到 `i32 | none`）。如果接收方有类型标注，它们必须具有该类型。混用 `break;` 和 `break v` 是编译错误（E0041）。

### 结果类型

至少出现一次 `break value` 的 `loop`，以该值的类型为结果类型。只有 `break;` 的 `loop` 的结果类型为 unit。完全没有 `break` 的 `loop` 是发散的，永远不会完成。

## 3.13.2. `while`：基于条件的重复

```nabi
while cond {
    ...
}
```

`while` 关键字之后跟一个 bool 条件和一个块主体。每次迭代开始时对 `cond` 求值：为 `true` 时运行主体；为 `false` 时循环结束。

### 结果值

`while` 的结果值总是 unit。

```nabi
let x = while cond { ... };     // x: unit
```

理由是，`while` 正常结束（`cond` 为 `false`）时，没有可以自然返回的值。它不是像 `loop { break v; }` 那样用于显式 `break value` 的形式：目标为 `while`（或 `for`）的带值 `break` 是编译错误（E0059）。

要返回值，使用 `loop { ... break v; ... }`。

### `while let`：与模式匹配结合

```nabi
while let v = compute() {        // compute(): T | none; v: T
    consume(v);
}
```

```
while_let ::= "while" "let" pattern (":" type_expr)? "=" expr block
```

每次迭代时对表达式求值，并按与 `if let` 相同的规则匹配（[3.10.5](310-if-expression.md#3105-if-let-combining-with-pattern-matching)），因此普通名字取得去掉 `none` 之后的值。匹配成功时运行主体；否则循环结束。模式的绑定在主体中可见。

可以使用[模式](311-patterns.md)中的每一种模式形式；通常用它匹配 `enum` 或和类型的变体。

## 3.13.3. `for`：基于迭代器的重复

```nabi
for pattern in iter_expr {
    ...
}
```

`for` 关键字之后跟一个模式、`in`、一个迭代表达式和一个块主体。迭代表达式必须是迭代器或可迭代的值。结构体字面量不能出现在迭代表达式或 `while` 条件的顶层（[3.10.3](310-if-expression.md#struct-literals-in-condition-position)）。每次迭代时，下一个元素与模式匹配，并在主体中绑定。

### 迭代的脱糖

```
for pat in xs { body }
```

在语义上等价于：

```
let it = <typeof(xs) as IntoIterator>::into_iter(xs);
loop {
    match <typeof(it) as Iterator>::next(ref it) {     // returns: T | none
        pat: T => body,
        none   => break,
    }
}
```

（确切的 trait 名和方法名由标准库定义。）

当值实现了 `IntoIterator` 时，循环逐步推进它的 `Iterator`。否则，当它实现了 `TryIterator`（[标准库 2.7.13](../../refs-standard-libraries/200-core/207-iter.md#2713-tryiterator---fallible-iteration)）时，循环直接推进它；它的 `next` 会抛出错误，因此循环必须写成 `try for`（或 `try! for` / `try? for`）。其他任何值都是编译错误（E0068）。

```
try for pat in xs { body }
```

在语义上等价于：

```
let it = try xs;                                     // the iterable is covered
loop {
    match try <typeof(it) as TryIterator>::next(ref it) {   // each step is covered
        pat: T => body,                              // the body is not covered
        none   => break,
    }
}
```

`try for` 的 `try` 只覆盖迭代本身：可迭代表达式和每一次 `next`。它不覆盖主体；主体中会抛出错误的调用与其他地方一样，用它自己的 `try` 标记（[3.14.2](314-error-control-flow.md#coverage)）。某一步产生的错误去往循环的 `try` 指定之处：`try for` 时传出函数，`try! for` 时触发 panic，`try? for` 时去往循环末尾，此时循环的值为 `none`。循环之后的 `catch` 附着于整个 `try for` 表达式（[4.14.10](../004-functions/414-throws.md#try-for)）。同样的规则适用于对 `TryStream` 的 `try for await`（[标准库 2.8.11](../../refs-standard-libraries/200-core/208-stream.md#2811-trystream---fallible-streams)）。

保存在变量或变量字段中的 `TryIterator` 是就地推进的：上面的 `it` 就是那个位置，而不是副本，正如 `for await` 中的流是就地轮询的（[4.13.5](../004-functions/413-stream.md#4135-the-for-await-form)）。由针对这种位置的 `match` 分支（或 `if let`）绑定的也是如此：绑定指称该位置，循环把该位置留在它停下之处。

对保存在变量中的 `Iterator` 进行的普通 `for` 推进的是一个副本：迭代器的 `into_iter` 就是迭代器本身，而上面的 `it` 是一个新变量。原变量保持自己的位置，因此 `let r = 0..3; for i in r { .. } for i in r { .. }` 会遍历 `r` 两次。要从循环停下之处继续，应手动推进迭代器（`while let x: T = it.next() { .. }`），这样作用于变量本身。

`List` 的迭代器在每一步读取列表当时的状态，因此改变所遍历列表长度的主体会改变循环：`push` 会让它继续下去，`remove` 会跳过移入被删除位置的元素。警告 lint `resize_while_iterating`（[6.2](../006-meta-system/602-built-in-attributes.md)）会报告主体中对被迭代变量调用 `push`、`pop`、`insert`、`remove`、`clear`、`truncate`、`drain`、`retain`、`dedup`、`extend` 或 `append` 的情况，除非同一块中其后跟有 `break`、`return` 或 `throw`。要修改列表，应遍历副本（`for x in xs.clone()`），或者收集修改并在循环之后应用。

### 结果值

`for` 的结果值总是 unit（理由与 `while` 相同），目标为 `for` 的带值 `break` 是编译错误（E0059）。

### 模式位置

只能使用[模式](311-patterns.md)中不可反驳的模式（名字、通配符、元组解构等）。

```nabi
for x in 0..10 { ... }                    // simple name
for (i, v) in xs.enumerate() { ... }      // tuple destructuring
for Point { x, y } in points { ... }      // struct destructuring
```

可反驳的模式（例如匹配 `enum` 变体）在 `for` 的模式位置会被拒绝（E0100），因为对匹配失败的处理会有歧义。需要这种形式时，使用 `while let`，或者组合使用 `for` 和 `if let`。

### `for ref`：作为位置的元素

```nabi
struct Point { x: f64; y: f64; }

let ps: List<Point> = [Point { x: 1.0, y: 2.0 }, Point { x: 3.0, y: 4.0 }];
for ref p in ps {
    p.x = 0.0;                            // writes the element of ps
}
println(f"{ps[1].x}");                    // 0
```

普通 `for` 绑定每个元素的副本：结构体是值（[2.3.8](../002-types/203-struct.md#238-value-semantics)），因此对绑定的字段写入不会改变循环之外的任何东西，并且当没有任何地方读取它时是编译错误（E0057）。`for ref x in xs` 则把 `x` 作为位置绑定到每个元素。读取 `x` 就是读取元素；`x = v`、`x.f = v`、`x.f += v` 以及对 `ref self` 方法的调用（`x.m()`，[4.7](../004-functions/407-ref-keyword.md)）会写入它。模式是单个名字。

`xs` 是 `List`、`SliceMut` 或数组。数组在其存放之处被写入，因此它必须是变量或字段，而不是临时值。其他任何可迭代值都是编译错误（E0068）：它的项不是位置。该循环等价于：

```
let v = xs;                    // a List: a SliceMut view of it; an array: xs itself
for i in 0..v.len() {          // the length is read once
    body                       // with x standing for v[i]
}
```

列表通过 `SliceMut` 视图遍历，因此适用该视图的规则（[标准库 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt---writable-view)）：在主体中改变 `xs` 的大小（`push`、`remove`、`clear` 等），如果主体中能看出这次改变大小和之后对 `x` 的使用，就是编译错误 `E0137`；否则（在被调用的函数内改变大小）会在下次使用 `x` 时触发 panic。应收集要添加或删除的元素，在循环之后修改 `xs`。

闭包或 `async` 块不能捕获 `x`；应先复制元素（`let v = x;`）。`if x is T` 不会收窄 `x`，`for await` 不能是 `for ref`。

## 3.13.4. 标签：控制嵌套循环

```nabi
'outer: loop {
    for x in xs {
        if cond { break 'outer; }
        if other { continue 'outer; }
    }
}
```

循环前可以加上 `'name:` 形式的标签。`break 'name` 和 `continue 'name` 直接控制带该标签的循环。

### 标签词法单元

标签由一个单引号、一个标识符和一个冒号组成（`'outer:`）。单引号与开始[字符字面量](../001-lexical/108-character-literals.md)的词法单元相同，但在与冒号组合的位置上被识别为标签。确切的切分规则是词法规范的补充定义；本文档只规定语言层面的形式。

### `break label value`

`loop` 标签可以与值结合。

```nabi
'outer: loop {
    for x in xs {
        if found(x) { break 'outer x; }
    }
}
```

`break 'outer x` 带着值 x 退出标签为 outer 的 `loop`。

## 3.13.5. `break` 和 `continue` 的默认行为

不带标签时，它们指向最内层的循环。

```nabi
loop {
    for x in xs {
        if cond { break; }       // exits the for
    }
    // continues here
}
```

## 3.13.6. 求值顺序

每种循环都按上文定义的含义求值。主体中的表达式和语句按[块表达式](309-block-expressions.md)的顺序求值，副作用遵循这一顺序。

循环终止的检查方式如下。`loop` 不做检查，只通过显式的 `break` 结束。`while` 在每次迭代开始时对 `cond` 求值。`for` 在每次迭代开始时调用 `Iterator::next`。

## 3.13.7. 发散循环

无法通过 `break`、`return`、`throw` 等方式终止的循环（例如内部没有 `break`、也没有向外退出的控制流的 `loop { ... }`）被归类为发散的。这种循环表达式的类型与任何类型兼容。
