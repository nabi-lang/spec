# 5.1. 变量声明

新变量用 `let` 关键字（[关键字](../001-lexical/105-keywords.md)）声明。本文档定义它的形式，以及区分重新赋值与新声明的规则。

## 5.1.1. 新声明：`let`

```
let_stmt    ::= "let" let_lhs (":" type_expr)? "=" expr ";"
let_lhs     ::= identifier
              | irrefutable_pattern    // tuple / struct destructure (see 5.1.11)
```

最常见的形式是单个标识符（`let x = ...`）。左侧为不可反驳的模式（5.1.11）时，它就成为[解构赋值](../003-expressions/315-destructuring.md)。`let-else` 形式（[let-else](../003-expressions/316-let-else.md)）是另一种语法。

```nabi
let x = 42i32;
let y: i32 = 10i32;
let z;                            // ERROR: no initializer
let z: i32;                        // ERROR: no initializer
```

基本形式是 `let`、名称、可选的 `: type`，然后是 `=` 和初始化式。初始化式是必需的；不存在未初始化的变量。有类型标注时，变量是该类型的静态槽；没有类型标注时，变量是从初始化式推断出的类型的静态槽（5.1.7）。只有显式写出 `: any` 时才会产生 [`any` 槽](../002-types/202-any-slot.md)。

这排除了读取尚未填充的槽的情况（即 Rust 的借用检查器所阻止的那类问题）。无法事先确定初始值时，用[和类型](../002-types/206-sum-types.md) `T | none` 显式表示“没有值”。

```nabi
let cache: Data | none = none;     // explicitly "not yet"
cache = compute();                  // filled later
```

## 5.1.2. 重新赋值：不写 `let`

要给已声明的变量赋新值，省略 `let`。

```nabi
let counter: i32 = 0i32;
counter = counter + 1i32;         // OK: reassignment
counter = counter + 1i32;         // OK: reassignment again
```

不写 `let` 而给从未声明过的名称赋值，是编译错误。

```nabi
let counter: i32 = 0i32;
coutner = counter + 1i32;         // ERROR (E0057): `coutner` is not declared
                                   //        help: did you mean `counter`?
```

这避免了 Julia 和 Python 中常见的陷阱：拼写错误会悄悄创建一个新变量。

## 5.1.3. 判定新声明还是重新赋值

| 形式              | 含义                                                   |
|-------------------|-----------------------------------------------------------|
| `let name = ...;` | 新声明。在该作用域中创建新绑定。             |
| `name = ...;`     | 重新赋值。再次写入已有绑定的槽。 |

显式优先：拼写错误是编译错误。

## 5.1.4. 在同一作用域中重新声明

`let` 可以声明在同一作用域中已绑定的名称。这会创建一个拥有自己槽的新绑定；从该语句结束起，这个名称指向新绑定（遮蔽）。

```nabi
let line = "  42  ";
let line = line.trim();                // the initializer reads the old `line`
let n: i32 = try! line.parse();
let n = n * 2;                         // a new slot, of the same type here
let x = "abc";
let x = x.len();                       // the type may differ: str, then usize
let (a, b) = (1, 2);
let (a, b) = (b, a);                   // patterns follow the same rule
```

- 初始化式在新绑定存在之前检查，所以其中的名称指向旧绑定。新绑定从该语句之后开始。
- 新绑定有自己的类型，可以是写出的，也可以是推断的（5.1.7）；它不必与旧绑定一致。每个槽保持声明时的类型：旧槽不会改变，不写 `let` 的重新赋值（5.1.2）写入最新的绑定。
- 旧绑定不会被重新赋值、丢弃或关闭。它照旧存活到其作用域结束（5.1.10），只是无法再通过名称访问。由它创建的闭包、视图和守卫仍然指向它。
- 必须关闭的值（[stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop释放时的清理)）不会因重新声明而关闭：在重新声明其名称之前，必须关闭或移动它。把它移动进新值是预期的用法。让它保持未关闭会触发通常的编译错误（E0134），其帮助信息提示在重新声明之前关闭或移动它。

```nabi
let f = try fs::open(path);
let f = io::BufReader::new(f);         // OK: the reader owns the file now
try for l in f.lines() { println(l); }
try f.close();

let g = try fs::open(path);
let g = 1;                             // ERROR (E0134): the first `g` is never closed
                                        //        help: close or move it before redeclaring it
```

由旧绑定创建的流视图在旧值被移动处结束，移动进新绑定时也是如此（E0139，[stdlib 2.23.1](../../refs-standard-libraries/200-core/223-io.md#2231-read)）。

要改变某个绑定的值而不是创建新绑定，不写 `let` 直接赋值（5.1.2）。REPL 和 Jupyter 在单元（cell）顶层遵循同样的规则（[9.7.3](../009-tools/907-jit-cells.md#同名重新声明)）。

## 5.1.5. 作用域：按块

变量可见的范围按[块表达式](../003-expressions/309-block-expressions.md)的块确定。

```nabi
let y = 1;
if cond {
    y = 2;                         // reassigns the outer y (same slot)
    let z = 2;                     // declares block-local z
}
// z is not accessible here
```

块内的 `let` 是属于该块的新局部绑定，所以该变量在块外不可见。内层同名的 `let` 只在该块内隐藏外层变量。

## 5.1.6. 嵌套作用域：独立的绑定

在内层块中重新声明同名变量时，它是一个独立的绑定。

```nabi
let a = 1;
{
    let a: i32 = 42i32;            // block-local a, a different slot from the outer a
    a = 10i32;                     // reassigns the inner a (outer a unaffected)
}
// here a = 1 (outer a unchanged)
```

内层块的 `let` 创建一个全新的绑定。即使外层存在同名变量，两者也是互相独立的槽。

这种设计既保留了“内层变量在块结束时消失”的含义，也保留了保护外层变量的意图。

### 与同一作用域中重新声明的区别

两者都会创建隐藏旧名称的新绑定，都不改变旧槽。区别在于旧名称在哪里重新出现：内层块中的绑定随块结束，之后外层名称再次可见。同一作用域中的重新声明（5.1.4）持续到该作用域结束，所以旧绑定不会再能通过名称访问。

```nabi
let a = 1;
{
    let a = "inner";               // hides the outer a inside the block
}
// a = 1 again
let a = a + 1;                     // same scope: a = 2 from here to the end
```

## 5.1.7. 类型标注与类型推断

```nabi
let a = 42i32;                     // a: i32 (inferred)
let b: i32 = 42i32;                // b: i32 (annotated)
let c: any = 42i32;                // c: any (annotated)
let d = if cond { 1i32 } else { 2i32 };   // d: i32 (inferred)
```

有类型标注时，变量是该类型的静态槽；没有类型标注时，变量是从初始化表达式推断出的类型的静态槽（局部类型推断）。之后的重新赋值按该类型检查。只有显式写出 `: any` 时才会产生 [`any` 槽](../002-types/202-any-slot.md)。

```nabi
let n = 1i32;
n = 2i32;                          // OK
n = "hi";                          // ERROR: n is i32
```

如果初始化式不能完全确定类型，推断会根据变量作用域内之后的使用继续进行（即把该值传到类型已确定的位置的使用）（[2.10.2](../002-types/210-generics.md#类型实参推断)）。如果类型仍然无法确定，就是编译错误，必须标注类型。

```nabi
let x = none;                      // ERROR: type cannot be determined
let x: Data | none = none;         // OK

let xs = [];                       // ERROR if there are no later uses: element type cannot be determined
let xs: List<i32> = [];            // OK

let y = return;                    // ERROR: initializer only diverges
```

各分支类型不同的 `if` / `match` 表达式不会推断出类型，除非某个分支的和类型包含所有分支的类型（此时结果就是该和类型）。需要和类型时，给槽加标注。

```nabi
let e = if cond { 1i32 } else { "hi" };          // ERROR: branch type mismatch
let e: i32 | str = if cond { 1i32 } else { "hi" };   // OK
```

推断的结果类型和分支结果规则，请见[表达式概述](../003-expressions/README.md)。

## 5.1.8. 多重声明（不提供）

```
let a = 1, b = 2;                  // not adopted
```

不提供 C 风格的、用逗号一次声明多个变量的形式。每行写一个，或者使用[解构赋值](../003-expressions/315-destructuring.md)中的元组解构。

```nabi
let (a, b) = (1, 2);               // destructuring assignment: OK
let a = 1; let b = 2;              // two statements: OK
```

## 5.1.9. `let` 可以出现的位置

`let` 是语句（请见[表达式概述](../003-expressions/README.md)中的转换规则）。

模块级不允许 `let`（模块级的值槽必须用 [`const`](502-const.md) 或 [`static`](503-static.md) 声明）。例外是入口文件 `main.nabi`，它的顶层语句（包括 `let`）构成隐式的 `main`（[4.16.2](../004-functions/416-entry-point.md#4162-顶层语句的含义隐式-main)）；在其他任何模块中，这些语句都是错误（E0036）。`let` 可以在函数体和块中使用。`if` / `match` / `loop` 等控制结构的主体本身也是块，所以那里也可以使用。`let` 不能直接出现在表达式位置，因为 `let` 是语句，不是表达式。

## 5.1.10. 用 `let` 声明的变量的生存期

变量从声明处一直存活到引入它的作用域结束。在同一作用域中重新声明其名称（5.1.4）不会结束它。

通过 `ref` 捕获（[捕获语义](../004-functions/406-capture-semantics.md)）被闭包捕获的变量，在该闭包对象存活期间也会保持存活。它由 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 管理，所以用户从不手动释放它。

## 5.1.11. `let` 的左侧与[解构赋值](../003-expressions/315-destructuring.md)

`let` 的左侧可以是[模式](../003-expressions/311-patterns.md)中的不可反驳的模式。这就是[解构赋值](../003-expressions/315-destructuring.md)。

```nabi
let (x, y) = compute();
let Point { x, y } = origin;
```

解构的语义在[解构赋值](../003-expressions/315-destructuring.md)中定义。每个子模式绑定到它的变量，所有变量一起引入同一个作用域。
