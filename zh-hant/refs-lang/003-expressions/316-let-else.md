# 3.16. `let-else`

`let-else` 把可反駁的模式放在 `let` 的左側，並一起寫出比對失敗時所走的發散分支。

## 3.16.1. 形式

文法如下：

```
let_else_stmt   ::= "let" pattern (":" type_expr)? "=" expr "else" block ";"
```

```nabi
let pattern = expr else { diverge };
```

`let` 之後依序接模式、`=` 與運算式，然後是 `else` 與一個區塊。任何可反駁的模式都可以接受。`else` 主體必須是[區塊運算式](309-block-expressions.md)的發散區塊，因此必須以 `return`、`break`、`continue`、`throw` 等方式離開。

## 3.16.2. 行為

```nabi
let value = compute() else {          // compute(): T | none
    return default;
};
// here value is a value of type T (if none, control left via the else above)
```

與 `if let`（[3.10.5](310-if-expression.md#3105-if-let-combining-with-pattern-matching)）相同，可選值上的單純名稱取得去掉其 `none` 的值，只去掉一層；`let value: T = compute() else { .. }` 仍會在較寬的和型別中測試 `T`。

求值依下列方式進行。運算式的結果與模式比對；成功時，模式的繫結被引入目前的作用域，求值繼續到下一行。失敗時，對 `else` 主體求值，而由於 `else` 是發散的，控制在那一點離開。

## 3.16.3. 必須發散

`else` 主體必須是發散的。正常完成的主體是編譯錯誤。

```nabi
let v: T = compute() else {
    print("no value");          // ERROR (E0060): not divergent
};
```

這項要求防止求值在比對失敗之後、在沒有繫結變數的情況下繼續。只有 `else` 主體離開，才能保證下一行存在那些繫結。

## 3.16.4. 解開和型別與 `enum`

最常見的用法，是接受 `T | none` 的一種情況或某個 `enum` 變體，並在其他所有情況下提前離開。

```nabi
// unwrap one member of a sum type
let n: i32 = x else {                    // x: f32 | i32
    panic("expected i32");
};
// here n: i32

// unwrap an enum variant
let Ok(data) = parse(input) else {
    return Err(ParseFailed);
};
// data is the contents of the Ok variant
```

這種模式讓垂直的流程比 `match` 更簡潔，有助於明確區分主要流程與例外情況。

> [!NOTE]
> 在 `let identifier : Type = expr else { ... };` 形式中，`: Type` 被解讀為**可反駁的型別檢查模式**（[模式](311-patterns.md)的 `type_check_pattern`）。與一般 `let` 中的槽位型別標註不同，它可能比對失敗，這正是 `else` 分支有意義的原因。也就是說，`let n: i32 = x else { ... };` 等價於 `match x { v: i32 => v, _ => { ... } }`，而 `: i32` 不是 `n` 的槽位型別，而是 `x` 必須通過的檢查。

## 3.16.5. 可見性

比對成功時，模式的繫結變數被引入 `let-else` 所在的作用域，而不是 `else` 主體內。

```nabi
{
    let v: T = compute() else { return; };
    // v is visible here
    consume(v);
}
// v is not visible here (scope ended)
```

繫結變數在 `else` 主體中不可見，因為比對才剛失敗，沒有任何值存在。

## 3.16.6. 守衛（無）

```
let v: T = expr if v > 0 else { ... };       // not adopted
```

不採用 `let-else` 中的守衛（`if condition`）。若需要守衛，請在之後另外檢查。

```nabi
let v: T = compute() else { return; };
if v <= 0 { return; }
```

## 3.16.7. `else` 的離開關鍵字

`else` 主體中常見的離開關鍵字及其效果：

| 關鍵字                          | 效果                                       |
|---------------------------------|--------------------------------------------|
| `return val;`                   | 從函式回傳                                 |
| `break;` 與 `break value;`      | 離開外層迴圈                               |
| `continue;`                     | 進入迴圈的下一次迭代                       |
| `throw err;`                    | 引發錯誤                                   |
| `panic("msg")`                  | 函式庫的 panic 函式，無法恢復的離開        |
| `loop { ... }`（沒有 `break`）  | 發散的迴圈                                 |

上列任何一種，以及編譯器歸類為發散的任何形式，都是允許的。

## 3.16.8. 與 `match` 的比較

```nabi
// let-else
let v: T = compute() else { return; };       // compute(): T | none
consume(v);

// match equivalent
let v = match compute() {
    v: T => v,
    none => return,
};
consume(v);
```

兩者表達相同的意義。當 `let-else` 扁平的主要流程較易讀時，選擇它。

## 3.16.9. 求值順序

`let-else` 依下列順序求值：

1. 對右側運算式求值。
2. 把結果與模式比對。
3. 成功時，繫結並繼續到下一行。
4. 失敗時，對 `else` 主體求值；由於它是發散的，控制離開。

若右側運算式帶有副作用，它只會被求值一次。
