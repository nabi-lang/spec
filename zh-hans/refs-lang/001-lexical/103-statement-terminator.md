# 1.3. 语句终止符

> [!IMPORTANT]
> Nabi 语句以分号（`;`）结束。仅靠换行不能结束语句。

```
stmt_terminator ::= ";"
```

## 1.3.1. 规则

语句终止符遵循几条简单的规则。一条语句恰好以一个 `;` 结束，多条语句可以共用一行。一条语句也可以跨越多行，因为换行只是词法单元之间的空白。

示意如下：

```
<stmt> ;
<stmt> ; <stmt> ;             // several on one line
<part of a stmt
 spanning lines> ;            // multiple lines. ends at the final ;
```

## 1.3.2. 为什么不在换行处自动插入

Nabi 不采用 JavaScript 的 ASI（Automatic Semicolon Insertion，自动分号插入）、Go 的行尾自动分号或 Swift 基于行的终止等自动规则。

理由如下。第一，自动规则会在少数情况下造成歧义；JavaScript 的 ASI 陷阱就是众所周知的例子。第二，显式终止符让格式化工具、LSP 以及编辑器的“跳到下一条语句”等工具能够确定地判断语句边界。最后，在 REPL 快速输入等场合，把多条语句写在一行上是很自然的。

## 1.3.3. 空语句

只由 `;` 构成的空语句是允许的，它没有任何效果。

```nabi
;;;
```

上面的代码被识别为三条空语句。接受这种形式是为了让工具生成的代码有更大的自由度。

## 1.3.4. 类块表达式语句

类块表达式以 `}` 结尾：`if`、`match`、`loop`、`while`、`for`，或者块（`{ }`、`unsafe { }`、带标签的块）。当这样的表达式出现在语句位置时，可以省略末尾的 `;`（与 Rust 规则相同）。此时该表达式的值必须是 unit；要丢弃非 unit 的值，请加上 `;`。在那里出现非 unit 的值是错误（E0067），无论在函数体中还是在入口文件的顶层语句中都是如此（[4.16.2](../004-functions/416-entry-point.md#4162-顶层语句的含义隐式-main)）。

带块体[尾随闭包](../004-functions/408-trailing-closure.md)的调用（`retry(3) { .. }`、`xs.each |x| { .. }`），当闭包的 `}` 之后没有任何延续时，同样不需要 `;` 就结束语句（[4.8.8](../004-functions/408-trailing-closure.md)）。这种情况下该调用是一个表达式语句，如同写了 `;`：其值不必是 unit，会被丢弃；与任何表达式语句一样，`()`、`none` 或 `Never` 以外的值会触发 lint `unused_result` 警告（`_ = run { .. };` 或 `#[discardable]` 函数可以消除它，[6.2.11](../006-meta-system/602-built-in-attributes.md#6211-未使用的结果unused_resultdiscardablemust_use)）。当调用位于语句开头时，它像以关键字开头的语句一样在那个 `}` 处结束，所以 `}` 之后的二元运算符开始一条新语句（见下文）。

```nabi
if ready { start(); }        // no ;
for x in xs { print(x); }
let n = if a { 1 } else { 2 };   // a let statement still ends with ;
```

如果二元运算符跟在以关键字（`if`、`match` 等）开头的类块表达式语句的 `}` 之后，或者跟在位于语句开头的尾随闭包调用的 `}` 之后，它开始一条新语句：`run { .. }` 之后跟一行 `-x;` 是两条语句。只有 `.method()` 这样的成员访问和后缀 `?` 会延续该表达式；与 Rust 一样，`}` 之后的 `(` 或 `[` 开始一个新的表达式。以 `unsafe { }` 开头的语句在二元运算符方面是例外：`unsafe { f() } != 0` 是一个表达式，但其 `}` 之后的 `(` 或 `[` 仍然开始一个新表达式。在更大的表达式内部，例如 `let v = run { .. } - x;` 的初始化表达式中，该调用是普通的操作数。

```nabi
if c { a(); } else { b(); }
-x;                          // new statement (not the if statement minus x)
match v { ... }.len();       // . continues
unsafe { step(); }
(a + b) * c                  // a new expression, not a call of the block
run { a(); }                 // the call statement ends at }
-x;                          // new statement
let v = run { a(); } - x;    // in a let, - continues the call
```

在块中间作为语句出现的非类块表达式需要 `;`。只有块的最后一个表达式（块的值）可以不带 `;`。

`}` 之前的最后一条语句可以省略其 `;`；省略与写出效果相同。这适用于 `let`（任何模式）、赋值、复合赋值、`return`、`break`、`continue` 和 `throw`。块的值与带 `;` 时相同：unit，或者对于 `return` / `break` / `continue` / `throw` 是发散的。不带 `;` 的最后一个表达式仍然是块的值，块中间的语句仍然需要 `;`。

```nabi
while i < n { i += 1 }           // same as { i += 1; }
if found { idx = k } else { idx = -1 }
let f = [ref total] |v: i32| { total += v };
let u = { let y = 1 };           // u is unit, as with { let y = 1; }
let s = { let y = 1; y + 1 };    // s is 2: the last expression is the value
```

语法由[块表达式](../003-expressions/309-block-expressions.md#391-形式)中的 `stmt` 定义。

## 1.3.5. 与其他词法单元相邻

`;` 词法单元可以不带空白地紧贴其前后的词法单元。例如，不留间隔地写出 `<stmt>;<stmt>;` 仍然会被正确地切分为两条语句。为可读性而加空格是推荐的风格，但规范本身不强制风格。
