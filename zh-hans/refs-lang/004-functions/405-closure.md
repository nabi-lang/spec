# 4.5. 闭包

闭包是出现在表达式位置、没有名字的函数值。它可以捕获外部环境中的变量，因此适合编写依赖周围状态的小函数。

## 4.5.1. 形式

```
closure_expr   ::= capture_list? closure_params (return_type block | closure_body)
closure_params ::= "||"
                 | "|" closure_param_list "|"
closure_param_list ::= closure_param ("," closure_param)* ","?
closure_param ::= (identifier | param_pattern) (":" type_expr)?
closure_body   ::= expr | block
capture_list   ::= "[" capture_item ("," capture_item)* "]"
```

与函数参数一样（[4.2](402-parameters.md)），闭包参数可以是不可反驳的模式：`|(k, v)| f"{k}={v}"` 接受一个元组实参并将其解构，而 `|k, v|` 接受两个实参。

零参数形式 `||` 是单个词法单元（见[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)），不会拆成两个 `|`。

```nabi
|params| body
```

参数由左右两个 `|` 词法单元（[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)）包围。主体是一个表达式或一个块。

### 单表达式主体

```nabi
|x| x + 1
|x: i32| x * 2
|a, b| a + b
```

单个表达式紧跟在 `|...|` 之后，其值即为闭包的返回值。

### 块主体

```nabi
|x: i32| -> i32 {
    let y = x * 2;
    y + 1
}
```

使用块主体时，可以用 `->` 给出返回类型（省略时推断）。

### 零参数形式

```nabi
|| body
|| { body }
```

没有参数时，使用 `||`（两个连续的竖线）。在词法分析上，`||` 在[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)的固定词法单元表中定义为一个词法单元。

## 4.5.2. 参数类型

参数类型可以写，也可以省略。

```nabi
|x: i32| x * 2                    // explicit
|x| x * 2                          // inferred
```

推断由闭包所在的上下文决定，例如它被传入的实参位置的函数类型，或者变量声明的类型。没有上下文时，由后续调用确定参数类型（`let f = |x| x * 2; f(4);` 中为 `x: i32`，[2.10.2](../002-types/210-generics.md#type-argument-inference)）。

```nabi
let f: fn(i32) -> i32 = |x| x * 2;   // x: i32 inferred from context
list.iter().map(|x| x * 2);                  // inferred from map's signature
```

## 4.5.3. 推断或显式的返回类型

```nabi
|x| x * 2                          // inferred
|x: i32| -> i32 { x * 2 }          // explicit: only with a block body
```

显式返回类型（`-> type`）只用于块主体。单表达式主体总是依赖推断。

## 4.5.4. 捕获

闭包可以捕获外层作用域中的变量。

```nabi
let n = 10;
let f = |x| x + n;                // captures n
f(5);                              // 15
```

捕获语义（默认按值捕获，以及 `[ref x]` 这样的捕获列表）在[捕获语义](406-capture-semantics.md)中详细定义。

## 4.5.5. 闭包与函数

| 方面                     | `fn` 具名函数       | `\| ... \|` 闭包        |
|--------------------------|---------------------|-------------------------|
| 名字                     | 有                  | 无                      |
| 定义位置                 | 模块或 `impl`       | 任何表达式位置          |
| 捕获外部变量             | 否                  | 是                      |
| 调用                     | `name(...)`         | `value(...)`            |
| 类型                     | `fn(...) -> ...`    | 相同                    |

两者用同样的函数类型表达，接口兼容，所以都适用于同样的位置。

## 4.5.6. 使用闭包值

```nabi
// assigned to a variable
let f = |x| x * 2;

// function argument
list.iter().map(|x| x * 2);

// function return
fn make_adder(n: i32) -> fn(i32) -> i32 {
    |x| x + n
}

// collection
let handlers: List<fn(i32) -> i32> = [|x| x + 1, |x| x - 1, |x| x * 2];
```

## 4.5.7. 调用

```nabi
let f = |x| x * 2;
let r = f(5);                     // same form as an ordinary function call
```

闭包以 `f(args)` 调用。调用时，捕获的环境与实参一起用于求值主体。

## 4.5.8. 立即调用：IIFE

闭包可以在定义后立即调用。

```nabi
let r = (|x: i32| -> i32 { x * 2 })(5);     // 10
```

这种写法不常见，但它是表达立即求值的局部作用域的一种方式。多数情况下，[块表达式](../003-expressions/309-block-expressions.md)中的普通块更自然。

## 4.5.9. 可读性：单表达式与块

```nabi
list.iter().map(|x| x * 2)               // single expression: short and clear
list.iter().map(|x| { let y = x * 2; y + 1 })   // block: multi-step processing
```

根据主体的长度选择合适的形式。规范完整支持这两种形式。

## 4.5.10. 尾随闭包

当函数调用的最后一个实参是闭包时，为了可读性，可以把它移到括号外面。详见[尾随闭包](408-trailing-closure.md)。

## 4.5.11. 特殊变体：`async` 和 `yield` 修饰符

给闭包加上 `async` 或 `yield` 修饰符，就得到异步闭包或生成器闭包。

```nabi
async || { ... }                  // closure returning a Future
yield || { ... }                  // closure returning an Iterator
async yield || { ... }            // closure returning a Stream
```

详细规则在[生成器](411-generators.md)、[异步](412-async.md)和[流](413-stream.md)中定义。

## 4.5.12. 零参数闭包简写：`{ body }`

在某些上下文中，`{ body }` 被识别为零参数闭包的简写。这种简写常与尾随闭包一起使用。

```nabi
thread::spawn { heavy_work() }    // == thread::spawn(|| { heavy_work() })
```

这种简写的确切规则在[尾随闭包](408-trailing-closure.md)中定义。
