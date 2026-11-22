# 4.11. 生成器（`yield`）

`yield` 关键字（[关键字](../001-lexical/105-keywords.md)）以三种形式定义生成器，即一次产生一个值的函数。

## 4.11.1. 三种形式

生成器的语法如下：

```
generator_fn    ::= "yield" fn_decl
generator_block ::= "yield" block
generator_closure ::= "yield" closure_expr
yield_stmt      ::= "yield" expr? ";"
```

```nabi
yield fn name(args) -> R { ... yield expr; ... }     // named generator function
yield { ... yield expr; ... }                         // inline block (value)
yield || { ... yield expr; ... }                      // closure modifier
```

在这三种形式中，结果都是库类型 `GenIter<T>`（实现 `Iterator<Item = T>`）的值；带 `throws (E)` 时是 `TryGenIter<T, E>`。这条规则在 [4.1.11](401-function-declaration.md#result-types-of-coroutine-functions) 中定义。

## 4.11.2. `yield` 语句：产出值

`yield` 语句可以带值，也可以不带值。

```nabi
yield expr;
yield;
```

`yield expr;` 把 `expr` 的值作为当前迭代步的输出产出，并挂起函数或块的求值。下一次迭代时求值恢复。`yield;` 不带值（unit）地前进一步。

## 4.11.3. 具名生成器函数

具名生成器函数的写法如下。

```nabi
yield fn count_up_to(n: i32) -> i32 {
    let i = 0;
    while i < n {
        yield i;
        i = i + 1;
    }
}
```

`yield fn` 是标记为生成器的函数声明；函数体用 `yield expr;` 产出值。`-> R` 的位置给出被产出值的类型，调用结果是 `GenIter<R>`。必须写出 `-> R`；元素类型不会从函数体推断。它像普通函数一样调用，结果是一个迭代器对象。

```nabi
let it = count_up_to(5i32);       // it: `iter::GenIter<i32>`
for x in it { print(f"{x} "); }   // 0 1 2 3 4
```

## 4.11.4. 内联 `yield` 块：值

要在表达式位置立即创建迭代器，使用内联 `yield` 块。

```nabi
let it = yield {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};
```

这种形式出现在表达式位置，立即产生一个 `GenIter<T>`。函数体是[块表达式](../003-expressions/309-block-expressions.md)中的块。它是以匿名、内联方式编写简短生成器的形式。

## 4.11.5. `yield` 闭包

`yield` 闭包产生一个可调用的生成器。

```nabi
let make_iter = yield || {
    yield 1i32;
    yield 2i32;
    yield 3i32;
};

let it = make_iter();             // each call creates a new Iterator
```

`yield ||` 创建一个可调用的生成器闭包；每次调用都创建一个新的迭代器。外层捕获遵循[捕获语义](406-capture-semantics.md)的规则。

## 4.11.6. 三种形式的对应关系

三种生成器形式与[函数声明](401-function-declaration.md)、[闭包](405-closure.md)和[块表达式](../003-expressions/309-block-expressions.md)的三种形式（具名、块、闭包）一一对应。

```
plain        fn f() { ... }       ↔  || { ... }       ↔  { ... } (block)
yield form   yield fn f() { ... } ↔  yield || { ... } ↔  yield { ... }
```

也就是说，`yield` 作为修饰符附加在同样的函数、块和闭包形式上，把它们变成生成器。

## 4.11.7. `yield` 词法单元的四种位置

`yield` 词法单元可以出现在四种位置。

```nabi
yield fn name(...) -> T { ... }   // declaration modifier
yield { ... }                      // block marker
yield || { ... }                   // closure modifier
yield expr;                         // statement
```

这四种位置由上下文明确区分；解析器根据下一个词法单元决定形式。

| `yield` 之后的词法单元 | 含义                                                                       |
|---------------------|----------------------------------------------------------------------------|
| `fn`                | 声明修饰符                                                                 |
| `{`                 | [块](../000-meta/007-glossary.md#block)标记（表达式位置）                   |
| `\|\|` 或 `\|`      | 闭包修饰符                                                                 |
| 表达式起始词法单元  | `yield` 语句                                                               |

## 4.11.8. 与 Iterator trait 的交互

生成器函数的结果是实现标准库 `Iterator<Item = T>` trait 的值，因此可以直接用在[循环](../003-expressions/313-loop-expressions.md)中的 `for` 循环里。

```nabi
yield fn naturals() -> i32 {
    let i = 0;
    loop {
        yield i;
        i = i + 1;
    }
}

for n in naturals() {
    if n > 100 { break; }
    print(n);
}
```

`Iterator` trait 的确切定义和方法（`map`、`filter`、`take`、`collect` 等）在标准库中定义。

## 4.11.9. 无限生成器

上面的 `naturals()` 是一个无休止产生值的无限生成器。这类生成器可以用 `for` 循环中的 `break`、库的 `take(n)` 方法等来终止。

## 4.11.10. 生成器的终止

函数体正常完成时，迭代器也随之终止，下一次 `next()` 调用返回 `none`。

```nabi
yield fn finite() -> i32 {
    yield 1i32;
    yield 2i32;
    yield 3i32;
    // function completes normally → iterator terminates
}
```

也可以用 `return;` 显式终止。

```nabi
yield fn early_stop(stop: i32) -> i32 {
    let i = 0;
    loop {
        if i >= stop { return; }
        yield i;
        i = i + 1;
    }
}
```

被 `for` 循环提前离开的生成器会被关闭。当循环体通过 `break`（也包括带标签的 `break` 或外层循环的 `continue`）、`return`、抛出的错误、`try` 传播、任务的取消，或外围生成器的 `yield` 离开循环时，就会发生这种情况。关闭会在生成器所等待的 `yield` 处恢复它，随后它像执行 `return` 一样离开函数体：该 `yield` 外围的 `finally` 块和 `using` 释放从最内层开始依次运行。关闭过程中到达的 `yield` 以同样方式结束生成器；它的值被丢弃。在第一次 `next()` 之前就被关闭的生成器，不运行函数体的任何部分。关闭之后，`next()` 返回 `none`。

```nabi
yield fn lines() -> str {
    using f = open_file() {          // dropped when the loop below breaks
        for l in f.lines() { yield l; }
    }
}

for l in lines() {
    if l == "" { break; }            // closes the generator: `f` is dropped
}
```

只有循环自身持有的生成器才会被关闭：即 `for` 表达式就是生成器值（一个调用、一个 `yield` 块）。存放在变量、字段或元素中的生成器不会被关闭，因此循环结束后还可以继续迭代：

```nabi
let g = lines();
for header in g { if header == "" { break; } }   // g stays open
for body in g { print(body); }                    // goes on after the blank line
```

从未被关闭、也从未完成的生成器，其清理操作会一直挂起，直到 GC 回收它；届时这些清理操作不会运行。

## 4.11.11. 生成器状态

生成器在调用之间保持内部状态。函数体的局部变量、迭代位置和协程帧会一直保留到下一次 `next()` 调用。

这种状态基于 [7. 运行时模型](../007-runtime/README.md)的协程基础设施，以编译器生成的（无栈）状态机方式运作。详细的实现机制在运行时规范中定义。

## 4.11.12. `yield` 函数的参数与返回

由于 `yield fn` 是生成器，`-> T` 表示函数体中被产出值的类型，调用结果是 `GenIter<T>`。

```nabi
yield fn f() -> i32 { yield 1i32; }
// call result: `GenIter<i32>`
```

参数的用法与普通函数相同。

## 4.11.13. 作为方法的生成器

`yield fn` 也可以用在[方法](409-method.md)的 `impl` 块中，定义生成器方法。

```nabi
impl<T> List<T> {
    yield fn iter(self) -> T {
        for x in self.elements { yield x; }
    }
}
```

其调用和行为与普通实例方法相同。
