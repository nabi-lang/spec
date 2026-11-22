# 4.13. 串流（`async yield`）

同時使用 `async` 與 `yield` 兩個修飾詞，會定義一個非同步產生器，即隨時間送出多個值的函式。其結果是函式庫型別 `AsyncStream<T>`（實作 `Stream<Item = T>`）的值，帶有 `throws (E)` 時則是 `TryAsyncStream<T, E>`（[4.1.11](401-function-declaration.md#協程函式的結果型別)）。

## 4.13.1. 三種定義形式

定義串流的文法有三種形式。

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

這是[產生器](411-generators.md)與[非同步](412-async.md)的三種形式一起運作。

## 4.13.2. 同步、非同步、產生器與串流的關係

四種形式的關係如下表所示。

| 形式             | 呼叫結果                            | 意義                       |
|------------------|-------------------------------------|----------------------------|
| `fn`             | `R`                                 | 立即得到一個值             |
| `async fn`       | `AsyncFuture<R>` | 將來得到一個值             |
| `yield fn`       | `GenIter<R>` | 同步地得到多個值           |
| `async yield fn` | `AsyncStream<R>`   | 隨時間得到多個值           |

這些是兩個正交維度的所有組合：延遲（同步 / 非同步）與值的數量（單一 / 多個）。

## 4.13.3. 主體中的兩個關鍵字

`async yield` 函式的主體中可以同時使用 `await` 與 `yield`。

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

`await expr;` 暫停函式直到 Future 完成；`yield expr;` 產生一個值並暫停，直到 Stream 推進到下一步。兩種暫停機制共存於一個函式中，而協程框架會追蹤在哪一點發生了哪種暫停。

## 4.13.4. 呼叫與消費

呼叫與消費串流的方式如下。

```nabi
async fn run() {
    let s = read_lines("data.txt");   // s: `stream::AsyncStream<str>`

    for await line in s {              // asynchronous for
        process(line);
    }
}
```

呼叫本身立即回傳一個 **Stream** 物件；實際的消費發生在非同步的 `for` 迴圈，即 `for await` 形式中。

`for await` 是[迴圈](../003-expressions/313-loop-expressions.md)之 `for` 的非同步變體。每次迭代都包含一次 `await`，以非同步方式等待串流的下一個值。

## 4.13.5. `for await` 形式

`for await` 的基本形式如下：

```nabi
for await pat in stream { body }
```

`for await` 是用於非同步迭代的關鍵字組合。每次迭代的效果相當於 `await stream.next()`，主體則像一般的 `for` 主體一樣撰寫。

存放在變數或變數欄位中的串流會原地輪詢，而不是透過複本：提前離開的迴圈讓變數停在迴圈停下之處，剩下的部分由變數的擁有者關閉。擁有資源的串流只有一個擁有者，因此 `let t = s` 會移動它，舊的名稱就不能再被走訪。由 `match` 分支（或 `if let`）在這樣的變數上繫結的串流指稱該變數：`match s { x: S => for await k in x { .. } }` 走訪的是 `s` 本身。

以值傳給函式的串流是該函式的複本：被呼叫者所走訪的部分不會讓呼叫者的變數前進。要在輔助函式中走訪串流的一部分，之後再繼續，請從輔助函式回傳該串流（`async fn first(s: S, k: usize) -> (S, usize)`）。在被走訪時把其控制代碼交給另一個執行緒的標準串流（`ReadDirStream`），在這樣的呼叫之後就不能再被走訪（E0105）。

這種形式只能用在 `async fn`、`async {}` 與 `async ||` 的主體內。

## 4.13.6. `yield` 與 `await` 的求值流程

下面的範例展示兩個關鍵字一起運作。

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

首先，`await fetch(...)` 等待直到收到頁面；接著 `yield page` 產生該頁面並暫停，直到消費者要求下一個值。當消費者要求下一個值時，函式繼續並取得下一頁。

## 4.13.7. Stream trait

結果物件是標準函式庫的 `AsyncStream<T>`（實作 `Stream<Item = T>`）。在語意上，它結合了 Iterator 與 Future：一種等待下一個值的機制。

詳細的簽章與方法（`map`、`filter`、`take`、`merge`、`zip` 等）定義於標準函式庫。

## 4.13.8. 與函式庫函式結合

串流的轉換與組合由標準函式庫函式提供。

```nabi
stream::merge(s1, s2)             // merges two streams
stream::zip(s1, s2)               // pairs up and advances together
s.throttle(delay)                 // rate limiting
```

詳細的函式與方法定義於標準函式庫規格。

## 4.13.9. 無限串流

串流也可以無限地持續下去。

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

這樣的無限串流在消費者以 `break` 離開，或透過 `take(n)` 等函式庫轉換時結束。

## 4.13.10. 與通道及管線的比較

標準函式庫的通道（`channel::channel()`，回傳一個 `Sender<T>` 與一個 `Receiver<T>`）也是隨時間移動多個值的機制，但它與串流的差異如下。

| 面向                    | `Stream<Item = T>`              | 通道（`Sender<T>` / `Receiver<T>`）|
|-------------------------|---------------------------------|------------------------------------|
| 定義                    | `async yield` 函式的結果        | 函式庫型別                         |
| 生產者與消費者          | 整合在一個函式中                | 兩個分開的端點                     |
| 多對多                  | 另外組合（merge 等）            | 自然（多個傳送者）                 |

簡言之，`Stream` 描述一個隨時間產生值的非同步計算，而通道描述多個並行任務交換值。這兩種抽象相輔相成。

## 4.13.11. 方法上的 `async` `yield`

`async yield` 也可以自由地定義為方法。

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

和一般方法一樣，`async yield` 方法可以自由地定義與使用。
