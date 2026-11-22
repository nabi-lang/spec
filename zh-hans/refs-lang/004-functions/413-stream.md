# 4.13. 流（`async yield`）

同时使用 `async` 和 `yield` 两个修饰符，可以定义异步生成器，即随时间推移产出多个值的函数。其结果是库类型 `AsyncStream<T>`（实现 `Stream<Item = T>`）的值；带 `throws (E)` 时是 `TryAsyncStream<T, E>`（[4.1.11](401-function-declaration.md#协程函数的结果类型)）。

## 4.13.1. 三种定义形式

定义流的语法有三种形式。

```
stream_fn       ::= "async" "yield" fn_decl
stream_block    ::= "async" "yield" block
stream_closure  ::= "async" "yield" closure_expr
```

```nabi
async yield fn name(args) -> R { ... await ...; ... yield ...; }   // named
async yield { ... await ...; yield ...; }                           // inline block
async yield || { ... await ...; yield ...; }                        // closure modifier
```

这是[生成器](411-generators.md)和[异步](412-async.md)的三种形式协同工作的结果。

## 4.13.2. 同步、异步、生成器与流的关系

四种形式的关系如下表所示。

| 形式             | 调用结果                            | 含义                       |
|------------------|-------------------------------------|----------------------------|
| `fn`             | `R`                                 | 立即得到一个值             |
| `async fn`       | `AsyncFuture<R>` | 未来得到一个值             |
| `yield fn`       | `GenIter<R>` | 同步得到多个值             |
| `async yield fn` | `AsyncStream<R>`   | 随时间得到多个值           |

它们是两个正交维度的全部组合：延迟（同步/异步）和值的数量（单个/多个）。

## 4.13.3. 函数体中的两个关键字

`async yield` 函数的函数体中可以同时使用 `await` 和 `yield`。

```nabi
async yield fn read_lines(file: str) -> str {
    let f = await open(file);
    loop {
        let line = await f.read_line();
        match line {
            l: str => yield l,
            none   => break,
        }
    }
}
```

`await expr;` 挂起函数直到 Future 完成；`yield expr;` 产出一个值并挂起，直到 Stream 推进到下一步。两种挂起机制共存于同一个函数中，由协程帧记录在哪个位置发生了哪一种挂起。

## 4.13.4. 调用与消费

调用和消费流的方式如下。

```nabi
async fn run() {
    let s = read_lines("data.txt");   // s: `stream::AsyncStream<str>`

    for await line in s {              // asynchronous for
        process(line);
    }
}
```

调用本身立即返回一个 **Stream** 对象；实际的消费发生在异步 `for` 循环中，即 `for await` 形式。

`for await` 是[循环](../003-expressions/313-loop-expressions.md)中 `for` 的异步变体。每次迭代都包含一次 `await`，异步地等待流的下一个值。

## 4.13.5. `for await` 形式

`for await` 的基本形式是：

```nabi
for await pat in stream { body }
```

`for await` 是用于异步迭代的关键字组合。每次迭代的效果相当于 `await stream.next()`，循环体的写法与普通 `for` 循环体相同。

存放在变量中（或变量的字段中）的流是就地轮询的，而不是通过副本：提前离开的循环会让该变量停留在循环停止的位置，剩余部分由该变量的所有者关闭。拥有资源的流只有一个所有者，因此 `let t = s` 会移动它，旧名字不能再被遍历。在这样的变量上由 `match` 分支（或 `if let`）绑定的流指代该变量本身：`match s { x: S => for await k in x { .. } }` 遍历的就是 `s` 本身。

按值传给函数的流是该函数的副本：被调用方遍历的部分不会让调用方的变量前进。要在辅助函数中遍历流的一部分并在之后继续，请从辅助函数返回该流（`async fn first(s: S, k: usize) -> (S, usize)`）。在被遍历期间把句柄交给另一个线程的标准流（`ReadDirStream`），在这样的调用之后不能再次遍历（E0105）。

这种形式只能用在 `async fn`、`async {}` 和 `async ||` 的主体中。

## 4.13.6. `yield` 与 `await` 的求值流程

下面的示例展示两个关键字如何协同工作。

```nabi
async yield fn paginate() -> Page {
    let next = "/start";
    loop {
        let page = await fetch(next);
        yield page;
        match page.next_url {
            url: str => next = url,
            none     => break,
        }
    }
}
```

首先，`await fetch(...)` 等待直到收到页面；然后 `yield page` 产出该页面并挂起，直到消费方请求下一个值。消费方请求下一个值时，函数恢复并获取下一页。

## 4.13.7. Stream trait

结果对象是标准库的 `AsyncStream<T>`（实现 `Stream<Item = T>`）。从语义上说，它结合了 Iterator 和 Future：一种等待下一个值的机制。

详细的签名和方法（`map`、`filter`、`take`、`merge`、`zip` 等）在标准库中定义。

## 4.13.8. 与库函数结合

流的变换与组合由标准库函数提供。

```nabi
stream::merge(s1, s2)             // merges two streams
stream::zip(s1, s2)               // pairs up and advances together
s.throttle(delay)                 // rate limiting
```

详细的函数和方法在标准库规范中定义。

## 4.13.9. 无限流

流也可以无限持续。

```nabi
async yield fn ticks() -> Instant {
    loop {
        yield Instant::now();
        await task::sleep(Duration::secs(1));
    }
}

async fn run() {
    for await t in ticks() {
        if should_stop() { break; }
        print(t);
    }
}
```

这样的无限流在消费方用 `break` 退出，或通过 `take(n)` 等库变换时终止。

## 4.13.10. 与通道和管道的比较

标准库的通道（`channel::channel()`，返回一个 `Sender<T>` 和一个 `Receiver<T>`）也是随时间传递多个值的机制，但它与流有以下区别。

| 方面                    | `Stream<Item = T>`              | 通道（`Sender<T>` / `Receiver<T>`） |
|-------------------------|---------------------------------|------------------------------------|
| 定义                    | `async yield` 函数的结果        | 库类型                             |
| 生产方与消费方          | 集成在一个函数中                | 两个独立的端点                     |
| 多对多                  | 需单独组合（merge 等）          | 天然支持（多个发送方）             |

简而言之，`Stream` 建模的是一个随时间产出值的异步计算，而通道建模的是多个并发任务之间交换值。两种抽象互为补充。

## 4.13.11. 方法上的 `async` `yield`

`async yield` 也可以自由地定义为方法。

```nabi
impl Database {
    async yield fn iter_rows(self, sql: str) -> Row {
        let cursor = await self.query(sql);
        loop {
            let row = await cursor.next();
            match row {
                r: Row => yield r,
                none   => break,
            }
        }
    }
}

async fn run(db: Db) {
    for await row in db.iter_rows("SELECT ...") {
        process(row);
    }
}
```

与普通方法一样，`async yield` 方法可以自由定义和使用。
