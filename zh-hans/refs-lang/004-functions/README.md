# 4. 函数概览

本章定义函数及其各种变体：闭包、方法和协程。本文档先给出总体图景，再进入详细规则。

## 目录

- [4.1. 函数声明](401-function-declaration.md)
- [4.2. 参数](402-parameters.md)
- [4.3. 返回](403-return.md)
- [4.4. 函数类型](404-function-type.md)
- [4.5. 闭包](405-closure.md)
- [4.6. 捕获语义](406-capture-semantics.md)
- [4.7. `ref` 关键字](407-ref-keyword.md)
- [4.8. 尾随闭包](408-trailing-closure.md)
- [4.9. 方法](409-method.md)
- [4.10. 嵌套函数](410-nested-functions.md)
- [4.11. 生成器（`yield`）](411-generators.md)
- [4.12. 异步（`async` 与 `await`）](412-async.md)
- [4.13. 流（`async yield`）](413-stream.md)
- [4.14. `throws` 签名](414-throws.md)
- [4.15. `unsafe`（函数修饰符、块表达式与未定义行为）](415-unsafe.md)
- [4.16. 入口点（`main.nabi`、`fn main`、顶层语句）](416-entry-point.md)

## 函数的作用

函数是接收输入值并产生结果值的代码单元。Nabi 函数可以具备以下所有特征。

按命名：有写作 `fn name(...)` 的具名函数，也有写作 `|...|` 的匿名闭包。按类型标注：函数是静态的或渐进的。写出参数类型时是静态的；在私有或 `pub(sub)` 函数上省略参数类型时，在调用处进行泛型单态化（[`any` 槽](../002-types/202-any-slot.md)、[泛型](../002-types/210-generics.md)）。`pub` 或 `pub(pkg)` 函数必须写出参数类型（E0138），trait 实现中的方法除外，它从 trait 取得参数类型（[4.2.2](402-parameters.md#422-省略类型)）。按执行模型：同步、异步和生成器形式都可以：普通函数、`async fn`（返回 Future）、`yield fn`（返回 Iterator）和 `async yield fn`（返回 Stream）。按异常行为：函数要么可以抛出，要么不能抛出。在签名中写上 `throws` 就允许它抛出（[错误控制流](../003-expressions/314-error-control-flow.md)）。最后，按归属：函数要么是方法，定义在 `impl` 块中并附属于某个类型，要么是模块级的自由函数。

## 函数定义的两种形式

函数以两种形式定义。

### 具名函数

第一种是有名字的函数。

```nabi
fn area(r: f64) -> f64 {
    3.14159 * r * r
}
```

这种形式由 `fn` 关键字、名字、参数、返回类型和函数体组成。它可以定义在模块级，也可以定义在 [`impl`](../002-types/209-impl.md) 块中。

### 匿名闭包

第二种是没有名字的闭包。

```nabi
|x: f64| -> f64 { 3.14159 * x * x }
|x| x * 2
|| compute()
```

这种形式用 `|...|` 括住参数列表，后面跟函数体。它直接出现在表达式位置，可以赋给变量，也可以直接作为函数实参传递。

两种形式由同一个内部机制处理。唯一的区别在于是否有名字，以及创建闭包时是否捕获环境。

## 调用

函数和闭包都用 `()` 调用。

```nabi
let a = area(2.0f64);
let f = |x: f64| x * 2;
let b = f(3.0f64);
```

调用时，实参与形参逐一匹配。确切的匹配规则在[参数](402-parameters.md)中定义。

## 返回

函数体最后一个表达式的值成为函数的返回值。函数体遵循[块表达式](../003-expressions/309-block-expressions.md)的块规则。

```nabi
fn add(x: i32, y: i32) -> i32 { x + y }
fn early(x: i32) -> i32 {
    if x < 0 { return 0; }     // explicit return for early exit
    x * 2
}
```

因此 `return` 关键字不是必需的；最后一个表达式会自动返回。详细规则见[返回](403-return.md)。

## 函数与表达式

函数和闭包都是值。它们可以赋给变量，作为实参传给其他函数，也可以存放在集合类型中。

```nabi
let f = |x| x * 2;
let v = list.iter().map(f);
let g = if cond { f } else { |x| x + 1 };
```

函数类型在[函数类型](404-function-type.md)中定义，由参数类型和返回类型组成。

## 调用分派

按照 [trait 不能作类型](../002-types/212-trait-as-type-banned.md)的规则，函数调用的分派无需经过虚表间接跳转即可解析。

具名函数在静态确定的函数地址上直接调用。闭包是捕获的环境与代码指针组成的一对值；调用它是一次直接调用，把捕获的环境作为第一个实参传入。泛型函数通过[泛型](../002-types/210-generics.md)的单态化，在每个调用处解析为直接调用。

因此，函数和闭包调用原则上都是直接调用，可以内联。它们的性能与普通方法调用相当。
