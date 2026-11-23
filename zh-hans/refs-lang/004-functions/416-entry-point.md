# 4.16. 入口点

本节定义可执行包启动时运行哪些代码以及按什么顺序运行。[源单元](../000-meta/003-source-units.md)中曾说“入口点的确切形式在函数规范中定义”，这里就是那个留待定义的位置。

## 4.16.1. 入口模块：`main.nabi`

可执行包的根文件 `src/main.nabi`（[0.3.1](../000-meta/003-source-units.md#可执行包srcmainnabi)）是**入口模块**。入口模块以下面两种形式中的**恰好一种**提供入口点。

| 形式                  | 入口点                                               | 用途                             |
|-----------------------|------------------------------------------------------|----------------------------------|
| 显式 `fn main`        | 用户声明的 `fn main` 函数                            | 普通应用程序                     |
| 顶层语句              | 以顶层语句为函数体的**隐式 `main`**                  | 脚本、示例、短小的程序           |

```ebnf
main_file   ::= item* fn_main item*            // form 1: explicit main
              | (item | top_stmt)*             // form 2: top-level statements (no fn main)
top_stmt    ::= let_stmt | let_else_stmt | expr ";" | block_like_expr ";"?   // 3.9.1
module_file ::= item*                          // lib.nabi and every module file other than main.nabi
```

`item` 是[模块中的项](../005-modules/504-modules.md#547-模块内的项)（`fn`、`struct`、`enum`、`trait`、`impl`、`const`、`static`、`mod`、`use` 等），`let_stmt` / `let_else_stmt` / `expr ";"` 与[块表达式](../003-expressions/309-block-expressions.md)中的语句相同。控制流（`if`、`match`、`for`、`while`、`loop`、`try`）由表达式构成，因此包含在 `expr ";"` 中。与函数体不同，这里没有不带终止符的最终表达式（`tail_expr`）：隐式 `main` 不返回值。只有项而没有语句（也没有 `fn main`）的 `main.nabi` 得到一个空的隐式 `main`，导出 WIT 的包除外：这样的包是没有入口点的 reactor（[标准库 1.8.2](../../refs-standard-libraries/100-bare/108-wit.md#导出)）。

> [!IMPORTANT]
> 如果同一个 `main.nabi` **同时**包含顶层语句和 `fn main` 声明，就是**编译错误**（E0036）。这排除了读者必须猜测两个入口点中哪一个会运行的情况。

```nabi
// main.nabi: ERROR
fn main() { run(); }
run();                      // ERROR: top-level statement in a file with fn main
```

## 4.16.2. 顶层语句的含义：隐式 `main`

没有 `fn main` 的 `main.nabi` 中的顶层语句**按源代码顺序**汇集成一个函数体，作为编译器合成的隐式 `main` 的函数体。项不被执行，也不放入该函数体；即使项和语句交错出现，也只有语句之间的相对顺序有意义。

```nabi
// main.nabi
let names = ["ada", "grace"];
for n in names {
    greet(n);               // greet is declared below; visible regardless of order
}

fn greet(name: str) {
    println(f"hello, {name}");
}

let total = names.len();
println(f"{total} greeted");
```

上面的文件与下面的含义相同：

```nabi
fn greet(name: str) {
    println(f"hello, {name}");
}

fn main() throws {          // form of the implicit main (see 4.16.4)
    let names = ["ada", "grace"];
    for n in names {
        greet(n);
    }
    let total = names.len();
    println(f"{total} greeted");
}
```

从这一等价关系可以得出以下规则。

- **项不论顺序均可见**。`main.nabi` 中的项与普通模块项一样属于模块作用域，因此顶层语句可以引用在它之后声明的 `fn` / `struct` 等。
- **顶层 `let` 是隐式 `main` 的局部变量**。它不是全局变量，只在其声明之后的顶层语句中可见（[变量声明](../005-modules/501-variable-declaration.md)的块作用域规则原样适用）。
- **顶层 `let` 在项的主体中不可见**。在上面的例子中，如果 `greet` 的函数体引用 `names` 或 `total`，就是名称解析错误。可以用闭包捕获该局部变量（`let shout = |s| println(f"{s}! ({total})");`）。闭包是一条语句，在隐式 `main` 的函数体中创建。

> [!NOTE]
> **为什么顶层 `let` 对项不可见**。项的存在与顺序无关，而 `let` 要到运行时才得到值。如果 `fn greet` 能看到顶层 `let`，就必须为 `greet` 在该 `let` **之前**被调用的情况（例如在第一行调用 `greet()`）定义含义，这相当于把“全局初始化顺序”的语义带进语言。Nabi 的原则是没有全局初始化顺序（4.16.6），因此顶层 `let` 被限定为隐式 `main` 的局部变量。需要在项之间共享的状态应当以参数传递，或声明为 `static`。

## 4.16.3. 显式 `main` 的签名

显式 `fn main` 没有参数，也没有泛型参数，并且必须是以下形式之一。其他任何形式都是编译错误（E0036）。

| 签名                              | 正常退出时的退出码       | 说明                                   |
|-----------------------------------|--------------------------|----------------------------------------|
| `fn main()`                       | `0`                      | 隐式抛出：未处理的异常 → 4.16.4        |
| `fn main() -> i32`                | 返回值                   | 返回值就是进程退出码                   |
| `fn main() throws` / `throws (E)` | `0`                      | 未处理的异常 → 4.16.4                  |
| 以上三种形式加上 `async`          | 同上                     | 4.16.5                                 |

没有返回 `Result` 的 `main`，也没有同时带 `throws` 和 `-> i32` 的 `main`。如果出错时需要特定的退出码，请在 `catch` 中调用 [`std::process::exit(code)`](../../refs-standard-libraries/300-std/306-process.md)。

命令行参数和环境变量通过标准库 [`std::env`](../../refs-standard-libraries/300-std/305-env.md) 获取，而不是通过参数。`unsafe` 和 `yield` 修饰符不能用于 `main`。可见性（`pub`）不影响它是否为入口点。

在 `lib.nabi` 或普通模块中声明的 `fn main` 不是入口点；它只是一个名为 `main` 的普通函数。

## 4.16.4. 退出码与未处理的异常

进程的退出码由入口点的结束方式决定。

- **正常退出**：按 4.16.3 的表格，为 `0` 或返回的 `i32`。
- **未处理的异常**：在 `throws` main（包括不带 `->` 的 `fn main()`）中，如果错误传播出入口点，运行时向标准错误（stderr）打印 ``error: `main` returned an uncaught error: ``，后接异常值，并以退出码 **`1`** 退出。输出格式：如果该值实现 [`Display`](../../refs-standard-libraries/200-core/210-fmt.md) 则用 `Display`，否则用 `Debug`，两者都没有则打印类型名（[`Error`](../../refs-standard-libraries/200-core/214-error.md) trait 要求 `Display`，因此普通错误类型用 `Display` 打印）。下一行给出错误离开 `main` 的位置，形如 `  --> path:line:col`。当引发它的 `throw` 已知且位于别处时，最后一行给出其位置，形如 `  raised at path:line:col`（具体指明哪一个 `throw`，遵循失败的 `try!` 的规则，[3.14.8](../003-expressions/314-error-control-flow.md#3148-panic-与致命错误)）。
- **panic**：panic 和致命错误不会被 `catch` 捕获；运行时把消息打印到标准错误，并以退出码 **`101`** 退出；在被 C 调用的函数之下则改为中止（[4.14.13](414-throws.md#41413-panic-与致命错误)）。
- **显式退出**：[`std::process::exit(code)`](../../refs-standard-libraries/300-std/306-process.md) 可以从任何位置立即终止进程。这条路径不运行 `finally` 或 `Drop`。

不带 `->` 也不带 `throws` 的显式 `fn main()` 与隐式 `main` 一样，被读作 `fn main() throws`（无类型）：脚本中的 `try` 不需要 `main` 上的 `throws`。`fn main() -> i32` 不抛出（带 `-> i32` 的 `throws` 不是入口形式）。

**隐式 `main`** 被视为无类型的 `fn main() throws`（错误类型为 `any Error`）（如果函数体包含 `await`，则为 `async fn main() throws`，4.16.5）。因此，顶层语句中用 `try` 标记的可抛出调用所产生的错误，无需单独的 `catch` 就可以传播出入口点，未处理的异常按上述规则得到退出码 `1`。如果执行正常运行到结尾，退出码为 `0`。

在 `wasm32-wasip2`（preview 2 组件）上，`wasi:cli/exit` 只携带成功或失败，因此任何非零退出码（包括 `1` 和 `101`）都以失败（状态 1）到达宿主。`wasm32-wasip1-threads`（preview 1）携带完整的退出码，但宿主可以把 126 或更大的 `proc_exit` 变为失败（状态 1）。`abort` 等 trap 以 134 退出。

> [!IMPORTANT]
> **顶层 `return;` 结束程序**。在 `main.nabi` 的顶层语句中（包括嵌套的块和 `let .. else`），裸 `return;` 从隐式 `main` 返回：`finally` 块和 `using` 释放会运行，退出码为 `0`。这让脚本可以提前停止，例如 `let x = opt else { return; };`。顶层语句不返回任何值，因此在那里写 `return v;` 是编译错误（E0018），其帮助信息会提示 `return;` 和 `process::exit(v)`。如需其他退出码，请调用 `process::exit(code)`，或使用显式的 `fn main() -> i32` 形式。

## 4.16.5. 异步 `main`

`async fn main()` 作为顶层任务运行在运行时[调度器](../007-runtime/706-scheduler.md)上。平台入口包装函数初始化运行时，把 `main` 创建的 Future 交给调度器，等待该 Future 完成，然后按 4.16.4 的规则确定退出码（行为与 [`task::block_on`](../../refs-standard-libraries/200-core/220-future.md) 相同）。`main` 结束时仍在运行的其他任务不会被等待。

当 `await` 直接出现在顶层语句中（而不是在另一个 `async` 块或闭包中）时，隐式 `main` 成为 `async fn main() throws`；否则它是同步的 `fn main() throws`。

```nabi
// main.nabi: implicit async main
use std::fs;
let body = try await fs::read_to_str_async("notes.txt");
println(body.len());
```

## 4.16.6. 模块只包含定义

顶层语句**只允许出现在入口模块 `main.nabi` 中**。`lib.nabi` 和 `main.nabi` 之外的每一个模块文件（以及内联 `mod { ... }` 的主体）只能包含项（`module_file ::= item*`）；在那里出现顶层语句是**编译错误**。对于内联 `mod x { stmt }` 主体中的语句，会给出一条指向本节的专门诊断。

因此，**导入模块（`mod`、`use`）不运行任何代码**。这一点有意与 Python 不同。在 Python 中，`import` 会运行模块文件的顶层代码，因此一个既作为库又作为脚本使用的文件必须用 `if __name__ == "__main__":` 守卫把两种用途分开。Nabi 不需要这样的守卫：作为脚本运行的代码只存在于 `main.nabi` 中，其他每个文件在导入时都不产生任何效果。

这一决定的理由：

- **AOT 是语义的基准**（[0.1.4](../000-meta/001-execution-modes.md#014-语义基准)）。要在 AOT 输出中支持导入时执行，就需要在链接时合成每个模块的初始化函数及其调用顺序。只包含定义的模块不需要这样的机制。
- **没有初始化顺序语义**。跨模块的导入时代码需要定义“哪个模块先初始化”以及“循环导入期间什么可见”。Nabi 保证这些问题永远不会出现。全局状态的初始值在编译期确定（4.16.7）。
- **缓存与增量构建**。由于模块的含义仅由其定义决定，基于内容哈希的缓存（[0.3.3](../000-meta/003-source-units.md#033-包标识符)）和增量构建无需考虑执行的副作用。
- **包的种类由文件是否存在决定**（[0.3.4](../000-meta/003-source-units.md#034-清单nabinabion是可选的)）。可执行代码恰好位于 `src/main.nabi`，因此“这是不是可执行包？”和“可执行代码在哪里？”这两个问题的答案总是一致。

## 4.16.7. 全局状态

模块级的全局状态只用 [`static`](../005-modules/503-static.md) 声明。`const` 初始化表达式在编译期求值；其他初始化表达式在首次访问其 `static` 时运行，绝不会早于入口点（[5.3.9](../005-modules/503-static.md#539-初始化式编译期或首次使用时)）。因此程序执行总是从入口点的第一条语句开始，在此之前不运行任何用户代码。

## 4.16.8. 单文件执行

不属于任何包的单个 `.nabi` 文件可以直接运行。

```
nabi run hello.nabi
nabic build hello.nabi
```

如果作为参数给出的文件不在某个包的 `src/` 中，编译器就构造一个仅由该文件组成的**匿名可执行包**，并把该文件视为这个包的 `main.nabi`。文件名没有意义，入口点规则（4.16.1 至 4.16.5）与 `main.nabi` 完全相同地适用。位于包的 `src/` 之外的 `.nabi` 文件（例如包根目录中的脚本）也以这种方式运行。匿名包可以有文件子模块：`mod foo;` 加载该文件旁边的 `foo.nabi`（其自身的子模块位于 `foo/` 中），与包的 `src/` 中完全相同（[5.4.4](../005-modules/504-modules.md#544-包根)）。只有没有文件的源代码（标准输入、REPL 会话）不能这样做；它们使用内联模块。如果需要依赖，就在文件及其任何子模块文件的开头用内联清单块声明（[9.10.7](../009-tools/910-nabi-frontend.md#9107-单文件脚本的内联清单)）。详细的命令形式在 [`nabic` CLI](../009-tools/901-nabic-cli.md) 和 [`nabi run`](../009-tools/910-nabi-frontend.md#nabi-run构建并运行) 中定义。

## 4.16.9. 没有入口点属性

Nabi **没有**用于指定入口点的属性（例如 `#[main]`）。根文件 `src/main.nabi` 已经指定了入口模块，名字 `main`（或顶层语句）指定了入口函数，因此不存在表达同一信息的第二种手段。只保留一种确定入口点的机制，意味着仅凭文件结构就能确定“运行哪个函数”。`main` 不是保留字，语言也不保留 `#[main]` 这个名字。

## 4.16.10. 与 JIT / REPL 的关系

REPL 输入和 Jupyter 单元格（[JIT 单元格](../009-tools/907-jit-cells.md)）的含义等同于**按顺序把顶层语句和项追加到一个隐式 `main.nabi` 的末尾**。单元格中的 `let` 是隐式 `main` 的局部变量，在后续单元格中可见；单元格中的 `fn` / `struct` 成为模块项。4.16.2 的规则原样适用，因此在单元格中声明的 `fn` 的函数体不能引用之前单元格中的顶层 `let`（可以用闭包捕获）。

得益于这一等价关系，[0.1.1](../000-meta/001-execution-modes.md#语义相同) 中“JIT 与 AOT 共享相同语义”的原则对入口点同样成立。把笔记本的各个单元格按顺序拼接而成的文件，就是一个产生相同结果的 `main.nabi`。不过，以下三项是只存在于交互式会话中的**会话操作**，而不是语言语义的扩展：

- 删除和重新运行单元格（[9.7.4](../009-tools/907-jit-cells.md#974-撤销--单元格重新执行)）。
- 自动显示不带终止符的最终表达式的值（[9.7.7](../009-tools/907-jit-cells.md#977-可视化--输出)）。在文件中，这等同于在该表达式后追加 `;`。
- 在该最终表达式后加上 `?`（`xs?`），显示其 `Debug` 形式（[9.7.7](../009-tools/907-jit-cells.md#977-可视化--输出)）。在文件中，末尾的 `?` 是 [`?` 运算符](../003-expressions/314-error-control-flow.md#31410--运算符提前返回-none)。
