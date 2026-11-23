# 9.7. JIT 单元格执行流程与状态管理

本节详细定义 [JIT 引擎](906-jit-engine.md)如何处理用户输入，即单元格（cell）。

## 9.7.0. 单元格会话的规则

单元格会话遵守以下规则。其实现是 [9.6 JIT 引擎](906-jit-engine.md)。

1. **只运行一次**。每个单元格的代码恰好运行一次。之前单元格的副作用（I/O、时间、随机数）不会重复。
2. **状态保持**。单元格的顶层 `let` 绑定、`static` 的状态以及 `any` 值（包括 `any Tr`）会延续到之后的单元格。绑定可以被赋值，同名的新 `let` 会以新绑定遮蔽旧绑定（9.7.3）。
3. **项保持且可以重新定义**。`fn` / `struct` / `impl` 等项会保持下来。如果再次定义同一个项，从下一个单元格起使用新定义。即使是在重新定义之前编译的代码，在之后的单元格再次执行到它时，也会使用新定义（调用被重新定义的 `f` 的 `g` 会调用新的 `f`）。重新定义 `static` 会重新开始其状态。类型被重新定义的绑定不能再使用；尝试使用它会报告错误并附带说明。一个单元格可以定义多个项。新的 `impl Tr for T` 替换旧的；新的固有 `impl T` 替换之前定义了其中某个方法的固有 `impl`。如果某个重新定义使之前的项无法再编译，该重新定义会被拒绝：错误指向那个项的内部，给出它的名称和所在单元格，并提示在同一个单元格中重新定义它或使用 `:reset`。
4. **`any` 的身份**。`any` 的类型标签在一个会话的所有单元格中都相同。存储在之前单元格中的 `any` 值，可以在之后的单元格中用 `is` / 模式取出。
5. **失败的单元格**。触发 panic、终止进程或崩溃的单元格不会对绑定留下任何部分效果。会话从该单元格之前的状态继续。已经发生的外部效果（I/O 等）不会撤销。之前单元格中的线程：单元格失败后，会话只保留回到原处的线程（Windows），或一个都不保留（fork 宿主）。运行时会重启自己的执行器线程；join 一个已经不存在的用户线程会报告它已丢失（`JoinErrorKind::Lost`，[标准库 3.4](../../refs-standard-libraries/300-std/304-thread.md)）。会话在一个静止时刻保存：没有任务正处于轮询中，也没有阻塞作业正在运行（宿主在单元格开始前等待它们完成；中断这种等待会取消该单元格），因此任务和作业永远不会丢失。已经不存在的线程在阻塞时持有的锁会被释放。
6. **显示**。表达式单元格用 `Display` 打印其值，没有则用 `Debug`，再没有则只打印其类型名；末尾的 `?` 打印 `Debug` 形式，没有则打印类型名（9.7.7）。

以下绑定和效果不受支持。

- 无法在顶层命名的类型（闭包、私有类型等）的绑定。它们在创建它们的单元格中就被拒绝。
- 在单元格内调用 `exit(..)`。该单元格被视为失败的单元格（规则 5）。（单独的 `exit` 或 `quit` 结束会话，[9.9.9](909-debug-tools.md#999-repl--jupyter-调试)。）
- 失败的单元格所创建的线程。它们不会在该单元格之后存活（规则 5）。

## 9.7.1. 单元格的种类

REPL 的一行或 Jupyter 的一个单元格，按其内容以不同方式处理。

```
1. expression                → evaluate + bind as `it` + print
2. statement (let / call)    → execute (return value is unit)
3. definition (fn / struct)  → add to the session's items
4. import (use / mod)        → update the session's visible names
5. meta / attribute          → handle as appropriate
```

这五种都遵循相同的单元格处理流程，只有最终输出不同。表达式单元格显示的值会绑定到 `it`，供之后的单元格使用（9.7.7）。表达式单元格打印其值（`Display`，没有则 `Debug`，再没有则其类型名）；`let` 单元格不打印任何内容。发散的表达式单元格（[4.3.7](../004-functions/403-return.md#437-发散函数)；`panic(..)`、`exit(..)`、`throw`）作为语句运行。

单元格可以在顶层语句中使用 `await`，就像隐式 `main` 一样（[4.16.5](../004-functions/416-entry-point.md#4165-异步-main)）：此时单元格在执行器上运行直到完成。它的 `let` 绑定和它的值与其他单元格一样延续到之后的单元格。

> [!IMPORTANT]
> 从语言语义上说，一系列单元格等价于把顶层语句和项追加到一个隐式 `main.nabi` 的末尾（[4.16.10](../004-functions/416-entry-point.md#41610-与-jit--repl-的关系)）。单元格中的 `let` 是隐式 `main` 的局部变量，因此之后单元格的语句可以看到它，但在单元格中声明的 `fn` 的函数体内看不到它。下文 9.7.3 的全局存储是一种让局部变量跨越单元格边界保持存活的实现技术，不改变这一可见性规则。单元格重新执行（9.7.4）、自动显示最后一个表达式的值及其绑定 `it`（9.7.7）是只存在于交互界面上的会话操作。

## 9.7.2. 执行流程：详细说明

```
1. parse → AST                      (error: display, discard cell)
2. lower to HIR, type check         (error: display, discard cell)
3. MIR → native machine code        (reuse / redefinition handling, 9.6.3 / 9.6.4)
4. host: load in a process forked from the snapshot, run the cell entry function
5. success: new snapshot, apply bindings and items to the session, print result
   failure: continue from the previous snapshot, display error
```

每个步骤都是事务性的；失败时回滚到之前的状态。

当 REPL 读取终端时，单元格的标准输入是终端，否则为空（[9.7.10](#9710-加载程序repl-path)）；Jupyter 单元格通过客户端读取（[9.8.1](908-jupyter-integration.md#981-内核)）。单元格的标准输出在每个换行处刷新，即使它不是终端也是如此，因此输出会随写随显。REPL 宿主在运行时启动前设置 `NABIR_GC_CONCURRENT=off`，不管用户环境如何设置：并发标记线程不会成为快照的一部分，所以会话中的完整回收在没有它的情况下进行标记（[7.5](../007-runtime/705-concurrent-collection.md)）。

## 9.7.3. 顶层变量存储

变量必须在 REPL 和 Jupyter 的单元格之间共享，因此顶层变量必须在整个会话期间保持存活。

```nabi
[1] let x = 42i32;
[2] let y = x + 1;       // uses the earlier x
```

### 存储形式

```
let x = 42                  → global of the inferred type (i32)
let x: i64 = 42             → global of the written type (i64)
heap payload                → register GC root
                             (compiler-emitted shadow stack frame,
                              linked into nabir shadow stack chain)
```

每个顶层变量在宿主中存储为其类型（写明的或推断的）的一个导出全局变量。之后的单元格引用同一变量时，通过该全局变量的地址找到它。

### GC 根注册

指向堆对象的顶层变量必须作为根被 GC 追踪，因此它们会被链入[影子栈](../000-meta/007-glossary.md#shadow-stack)链。

详细机制见[屏障与根追踪](../007-runtime/704-barriers-and-roots.md)中关于影子栈的部分。

### 同名重新声明

```nabi
[1] let x = 1i32;            // x: i32
[2] let x = "hi";            // redeclaration at cell top level: new binding (x: str)
```

在单元格顶层对已经绑定的名称使用 `let` 是重新声明，与源文件中相同（[5.1.4](../005-modules/501-variable-declaration.md#514-在同一作用域中重新声明)）。它创建一个新绑定，其类型可以与之前的绑定不同。之后的单元格看到的是新绑定。

此时现有的全局变量被覆盖：其内存留在原处，只有名称槽被重定向到新定义。

这支持 REPL 中常见的重新定义变量的用法。

## 9.7.4. 撤销 / 单元格重新执行

### 在 Jupyter 中重新执行单元格

在 Jupyter 中编辑并再次运行一个单元格时，它作为新单元格运行。它重新定义的项和绑定从之后的单元格起以新形式使用（9.7.0 规则 2、3）。之前执行过的单元格的效果不会被抹去。

### 撤销的含义

REPL 没有显式的撤销；工作通过覆盖最后一个单元格或添加新单元格来推进。

在 Jupyter 中，删除或重新运行单元格同样不会撤销之前单元格的效果。要回到初始状态，请重启会话（在 REPL 中使用 `:reset`）。

## 9.7.5. 错误模式

### 单元格编译错误

```
parse error / type error / semantic error.
  - the cell is not added.
  - the session's bindings and items are unchanged.
  - reuse information for already compiled functions is kept.
  - the error is shown to the user.
```

此步骤是完全事务性的，因此会保留错误发生前一刻的状态。如果编译器本身在编译单元格时触发 panic（内部编译器错误），`nabi lsp`、`nabi repl` 和 `nabi jupyter` 都不会因此退出：语言服务器把它报告为被分析缓冲区的一条错误诊断，REPL / 内核则使该单元格失败；两者都继续提供服务。

### 单元格运行时错误

```
panic / uncaught throw / process exit / segfault during execution.
  - the cell is not added; bindings and items stay as they were before the cell.
  - the error is shown to the user.
  - the next cell can proceed.
```

错误会说明单元格是如何结束的，以及会话没有变化：

```
panic             the panic's report, then: the cell panicked; the session is unchanged
uncaught throw    the cell threw `<error>`; the session is unchanged
                    --> <where the error left the cell>
                    raised at <the throw that raised it, when known and elsewhere>
exit(n)           the cell called exit(n); the session is unchanged (`exit` or `:quit` ends the session)
Ctrl-C            interrupted (Ctrl-C); the session is unchanged
other (a fault)   the cell failed; the session is unchanged
```

`<error>` 是错误的描述，与从 `main` 中逃逸出的未捕获错误相同（[4.16.4](../004-functions/416-entry-point.md)）。任何消息都不会提及单元格生成的入口函数的名称。

绑定和项恢复到单元格之前的状态（9.7.0 规则 5），但已经发生的外部效果（输出、文件写入等）不会撤销。失败的单元格不会保留在会话中，因此可以再次输入修正后的单元格。没有 fork 的宿主（Windows）在单元格开始前保存会话的内存和线程，并在单元格失败时恢复它们，可观察到的结果与 fork 快照相同。

### JIT 失败

如果宿主无法加载或链接单元格的代码（未知的 `extern` 符号等），就按单元格运行时错误处理。运行该单元格的 fork 进程结束，由之前的快照接管。

### 位置

正在输入的单元格中的诊断给出 `<repl>:line:col`，按该单元格自身的文本计数。之前单元格的项中的诊断，以及正在运行的代码报告的位置（panic、未捕获的错误、`try!`），给出 `<repl cell N>:line:col`：单元格 N 是会话中输入的第 N 个单元格（`:type` 等命令不是单元格；失败的单元格也计入），行和列按其文本计数。在 Jupyter 内核中，N 是笔记本单元格的执行计数，被拆分为多个 REPL 单元格的笔记本单元格把它们编号为 `N.1`、`N.2` 等（[9.8.1](908-jupyter-integration.md#笔记本单元格与-repl-单元格)）。在之前单元格中定义的函数报告的是该单元格中的位置。

```
>>> fn boom(n: i32) {
...     panic(f"boom {n}");
... }
>>> boom(5)
thread 'main' panicked at <repl cell 1>:2:5:
boom 5
error: the cell panicked; the session is unchanged
```

## 9.7.6. 缓存

在一个会话中，宿主会重用已经编译过的函数（9.6.4）。不存在跨会话的磁盘缓存。

## 9.7.7. 可视化 / 输出

### REPL 输出

```
>>> 1 + 2
3
>>> "hi"
hi
```

最后一个表达式的值用 `Display` 打印，没有则用 `Debug`。末尾的 `?`（如 `xs?`）改为打印 `Debug` 形式，`:type xs` 只打印类型。既没有 `Display` 也没有 `Debug`（对 `?` 而言是没有 `Debug`）的值只打印其类型名（`type_name_of_val`，[标准库 2.12.4](../../refs-standard-libraries/200-core/212-reflect.md#2124-repl-输出)：`any` 值给出它所持有的类型）。`let` 单元格不打印任何内容。在 Jupyter 笔记本中，具有 `RichDisplay` 的值改为显示其 MIME 包（[9.8.2](908-jupyter-integration.md#982-本地原生内核运行方式)）。

```
>>> let xs = [1, 2, 3];
>>> xs
[1, 2, 3]
>>> "hi"?
"hi"
>>> struct Opaque { id: i32 }
>>> Opaque { id: 1 }
Opaque
```

### `it`：最后显示的值

表达式单元格显示的值绑定到 `it`，它和其他绑定一样是会话的顶层绑定：之后的单元格可以读取它、给它赋值，`:vars` 也会列出它。不显示任何内容的单元格（语句、unit 值、失败的单元格）保持 `it` 不变；类型无法在顶层命名的值（9.7.0）也是如此。

```
>>> 2 + 3
5
>>> it * 10
50
>>> it
50
```

用户定义的名称优先：一旦某个单元格用 `let` 绑定了 `it`，或者定义了名为 `it` 的项，表达式单元格就不再绑定它，直到 `:reset`。在 Jupyter 会话中同样如此。

### Jupyter 输出

每个单元格的结果作为 `execute_result` 发送，其 `text/plain` 条目就是 REPL 打印的文本；类型实现了 `RichDisplay` 的值还会加入其包中的 MIME 类型（[9.8](908-jupyter-integration.md)）。

Jupyter 集成的细节在 [Jupyter 集成](908-jupyter-integration.md)中定义。

## 9.7.8. 输入单位：REPL 识别

REPL 以完整的表达式或语句为单位处理输入。

```
>>> 1 + 2
3
>>> let x =
... 42i32;          (continued input)
>>> x
42
```

提示符是 `>>>`，单元格继续时为 `...`。单元格在行尾结束，这一判定基于词法单元：当括号、字符串或块注释仍未闭合，当最后一个词法单元不能结束语句（二元运算符、`.`、`,`、`=`、`->`、`|`，或 `else`、`in` 等关键字），或者当输入以缺少主体的项头结尾（`fn g(x: i32)`，而 `-> i32 { .. }` 在下一行）时，输入以 `...` 继续。字符字面量、字符串和注释中的括号不计入。其他任何行都会结束单元格。`///` 文档注释行也会使输入继续：它属于其后各行上的项。以 `:` 开头的行是命令，在该行结束。当单元格的最后一个顶层语句是不带 `;` 的 `let` 或赋值（如 `let x = 1` 或 `k += 2`）时，会补上 `;`：`let` 照常绑定，单元格不显示任何内容。其他语句不做改动，诊断按输入原样引用单元格。在终端上，每个提示符都从新的一行开始，即使之前的输出没有以换行结尾也是如此，并且不绘制任何标记；管道输入没有提示符。当标准输入在一个不完整的单元格内部结束时，该单元格不会运行，会报告一个错误，`nabi repl` 以 1 退出。失败的单元格不会改变退出码。

### 行编辑

在终端上，REPL 自己编辑每一行；管道输入按原样读取。

- **启动**。当标准输入和标准输出都是终端时，REPL 在第一个提示符之前打印两行：`Nabi <version> (<commit> <date>) on <host triple>`，其中版本文本与 `nabic --version` 打印的相同，然后是 `Type :help for help, :exit to quit.`。`-q` 省略这两行；管道输入永远不会得到它们。
- **按键**。方向键、Home / End 和 Emacs 按键用于移动和编辑：`C-a` / `C-e` 移到行首和行尾，`C-b` / `C-f` 和 `M-b` / `M-f`（也可以是 Alt 或 Ctrl 加方向键）按字符或单词移动，`C-k` / `C-u` / `C-w` / `M-Backspace` 删除到行尾、到行首或前一个单词，`C-d` 删除下一个字符，在空行上则结束输入，`C-l` 清屏，`C-c` 丢弃当前行和未完成的单元格。在 Unix 上，`C-z` 把 REPL 作为作业挂起。
- **显示**。长行横向滚动。列按显示宽度计数：东亚宽字符（East Asian Wide）、全角字符（Fullwidth）和 emoji 占两列，组合标记和其他零宽字符不占列。输入按词法单元类别着色，与文档中相同：关键字和属性为浅粉色，类型为天蓝色，字符串为绿色，数字为琥珀色，注释为灰色，使用 [9.1.8](901-nabic-cli.md#颜色) 的调色板和色深。标准输出上的颜色遵循 [9.1.5](901-nabic-cli.md#915-全局选项)：`--color=never`、`NO_COLOR` 或 `TERM=dumb` 会关闭它。
- **补全**。Tab 把光标前的单词补全为候选项的最长公共前缀，如果这样补不出任何内容，就列出候选项。候选项包括光标处作用域内的局部变量、会话的绑定和项、prelude 的名称（`use core::prelude::*;`）、会话的 `use` 项导入的名称，以及关键字；在 `x.` 之后是 `x` 的类型的字段和方法（对于标准库类型，只有其公开的字段和方法），在 `a::` 之后是 `a` 下的模块和项。在行首按 Tab 缩进四个空格。Jupyter 内核从相同的候选项中补全（[9.8.5](908-jupyter-integration.md#985-代码补全)），语言服务器使用相同的引擎（[9.9.13](909-debug-tools.md#9913-lsp-服务器)）。
- **粘贴**。REPL 开启括号粘贴（bracketed paste）：粘贴的块（包括换行和 Tab）作为文本插入，按下 Enter 时成为一个单元格，而不是逐行运行。编辑时，输入中的换行显示为 `↵`；输入后，单元格显示为 `...` 之后的各行。
- **历史**。历史记录保存完整的单元格：跨多行输入或粘贴的单元格是一个条目。Up / Down（`C-p` / `C-n`）在其中移动；已输入文本时，Up 和 Down 只在以该文本开头的条目之间切换。`C-r` 进行增量反向搜索：输入的文本缩小搜索范围，再按 `C-r` 查找更早的匹配项，Enter 运行匹配项，按其他键则编辑它，`C-g` 返回到当前行。历史记录在启动时加载，保存在 `$NABI_HOME/repl_history` 中；它保留最近的 1000 个单元格，在达到 2000 个时丢弃较早的单元格，并且不保存超过 64 KiB 的单元格。
- **Windows**。控制台被切换到 VT 输入和输出。括号粘贴在支持它的终端（如 Windows Terminal）中可用。

## 9.7.9. 输入单位：Jupyter 单元格

Jupyter 单元格是一次性处理的多行输入。

```
[Cell 1]:
let x = 42i32;
let y = x * 2i32;
y                       # last expression = cell result
```

单元格的最后一个表达式是单元格的结果；其余一切都视为副作用（定义或调用）。

## 9.7.10. 加载程序（`repl <path>`）

带程序路径的 `nabi repl <path>` 把程序加载到一个新会话中，然后在该会话上继续 REPL。如果给出的是库包路径，则把该包添加为依赖，并把 `use <name>::*;` 作为第一个单元格运行（`:reset` 之后也是如此）；不加载其他任何内容。

- **先检查**。在运行任何内容之前检查整个程序。错误在文件自身的位置报告；不运行任何内容，会话以空状态开始。
- **项**。项和导入加入会话。
- **语句**。顶层语句作为单元格运行。每个单元格在一个顶层 `let` 处（或在最后一个语句处）结束。
- **失败**。发生 panic、`exit()` 或错误逃逸时，已完成单元格中的绑定保持其在该单元格结束时的值；失败部分中的赋值会丢失。程序的其余部分不运行，会话继续。
- **不可用的绑定**。之后的单元格无法使用的绑定（私有类型或无法命名的类型），如果之后没有语句提到它就会被丢弃；否则，从开始到最后一次使用它为止的语句作为一个单元格运行。
- **`fn main`**。带有 `fn main` 的程序把 `main` 作为入口运行；之后 `main` 离开会话，因此它的局部变量不可见。
- **位置**。诊断和 panic 使用文件自身的位置。
- **输入**。程序像任何单元格一样读取 stdin。在终端上，正在运行的单元格读取终端（控制终端；在 Windows 上是控制台 `CONIN$`）；单元格结束后，行编辑器以单元格之前的模式收回终端。管道输入属于 REPL：单元格读取的是空的 stdin（`/dev/null`，在 Windows 上是 `NUL`），会立即遇到输入结束，其后的各行仍作为单元格读取。
- **`:load`、`:reload`**。`:load <file>` 按相同规则把程序加载到正在运行的会话中；它的项替换会话中同名的项。`:reload` 再次加载上一次加载的程序（[9.9.9](909-debug-tools.md#999-repl--jupyter-调试)）。由 `:save` 写出的脚本不作为程序加载：它的单元格会像手动输入一样逐个重新运行，因此再次声明的名称保持其最后的绑定。

## 9.7.11. 下一步

[Jupyter 集成](908-jupyter-integration.md)定义基于本地 ZMQ 的 Jupyter 内核集成。
