# 3.16. `let-else`

`let-else` 在 `let` 的左侧放置一个可反驳的模式，并同时写出匹配失败时所走的发散分支。

## 3.16.1. 形式

语法是：

```
let_else_stmt   ::= "let" pattern (":" type_expr)? "=" expr "else" block ";"
```

```nabi
let pattern = expr else { diverge };
```

`let` 之后跟一个模式、`=` 和一个表达式，然后是 `else` 和一个块。接受任何可反驳的模式。`else` 主体必须是[块表达式](309-block-expressions.md)中的发散块，因此它必须通过 `return`、`break`、`continue`、`throw` 等方式退出。

## 3.16.2. 行为

```nabi
let value = compute() else {          // compute(): T | none
    return default;
};
// here value is a value of type T (if none, control left via the else above)
```

与 `if let`（[3.10.5](310-if-expression.md#3105-if-let与模式匹配结合)）一样，可选值上的普通名字取得去掉 `none` 之后的值，只去掉一层；`let value: T = compute() else { .. }` 仍然在更宽的和类型中检测 `T`。

求值过程如下。表达式的结果与模式匹配；成功时，模式的绑定被引入当前作用域，求值继续进行到下一行。失败时，对 `else` 主体求值，由于 `else` 是发散的，控制在此处离开。

## 3.16.3. 必须发散

`else` 主体必须是发散的。正常完成的主体是编译错误。

```nabi
let v: T = compute() else {
    print("no value");          // ERROR (E0060): not divergent
};
```

这一要求防止匹配失败后在没有绑定变量的情况下继续求值。只有 `else` 主体退出，才能保证下一行上绑定存在。

## 3.16.4. 解开和类型与 `enum`

最常见的用法是接受 `T | none` 的某一种情况或某个 `enum` 变体，在其他所有情况下提前退出。

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

这种模式让纵向的流程比 `match` 更简洁，便于把主流程与例外情况明确分开。

> [!NOTE]
> 在 `let identifier : Type = expr else { ... };` 形式中，`: Type` 被解释为**可反驳的类型检查模式**（[模式](311-patterns.md)中的 `type_check_pattern`）。与普通 `let` 中的槽类型标注不同，它可能匹配失败，这正是 `else` 分支的意义所在。也就是说，`let n: i32 = x else { ... };` 等价于 `match x { v: i32 => v, _ => { ... } }`，`: i32` 不是 `n` 的槽类型，而是 `x` 必须通过的检查。

## 3.16.5. 可见性

匹配成功时，模式的绑定变量被引入 `let-else` 所在的作用域，而不是 `else` 主体内部。

```nabi
{
    let v: T = compute() else { return; };
    // v is visible here
    consume(v);
}
// v is not visible here (scope ended)
```

绑定变量在 `else` 主体中不可见，因为匹配刚刚失败，不存在值。

## 3.16.6. 守卫（不采用）

```
let v: T = expr if v > 0 else { ... };       // not adopted
```

不采用 `let-else` 中的守卫（`if condition`）。如果需要守卫，应在之后另行检查。

```nabi
let v: T = compute() else { return; };
if v <= 0 { return; }
```

## 3.16.7. `else` 的退出关键字

`else` 主体中常见的退出关键字及其作用：

| 关键字                          | 作用                                       |
|---------------------------------|--------------------------------------------|
| `return val;`                   | 从函数返回                                 |
| `break;` 和 `break value;`      | 退出外层循环                               |
| `continue;`                     | 进入循环的下一次迭代                       |
| `throw err;`                    | 抛出错误                                   |
| `panic("msg")`                  | 库的 panic 函数，不可恢复的退出            |
| `loop { ... }`（无 `break`）    | 发散循环                                   |

这些形式，以及编译器归类为发散的任何形式，都是允许的。

## 3.16.8. 与 `match` 的比较

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

两者表达相同的含义。当 `let-else` 扁平的主流程更易读时，选择它。

## 3.16.9. 求值顺序

`let-else` 按以下顺序求值：

1. 对右侧表达式求值。
2. 把结果与模式匹配。
3. 成功时，进行绑定并继续下一行。
4. 失败时，对 `else` 主体求值；由于它是发散的，控制离开。

如果右侧表达式有副作用，它只被求值一次。
