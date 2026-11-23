# 1.2. 词法结构

本章给出字符流如何变成词法单元流的总体图景。每种词法单元的详细格式在后续文档中定义，这里只给出概要。

```
token           ::= identifier
                  | keyword
                  | numeric_lit
                  | string_lit
                  | byte_string_lit
                  | fstring_tokens                // f"..." / f"""...""", see 1.2.1
                  | char_lit
                  | sentinel_lit                  // true | false | none
                  | label_token                   // 'name
                  | punctuation
label_token     ::= "'" identifier                // no closing quote
whitespace      ::= space_char+
                  | line_comment
                  | block_comment
space_char      ::= U+0009 | U+000A | U+000B | U+000C | U+000D | U+0020
line_comment    ::= "//" <any char except line_terminator>* line_terminator
block_comment   ::= "/*" <any char>* "*/"          // recursive
shebang_line    ::= "#!" <any char except line_terminator>*   // first line only, see 1.2.3
line_terminator ::= U+000A | U+000D U+000A? | U+000D
```

## 1.2.1. 词法单元

编译器的词法分析器（**lexer**）读取源文件，产生词法单元流。词法单元分为以下几类。

| 种类                         | 示例                                |
|------------------------------|-------------------------------------|
| 标识符                       | `count`、`α`、`Δt`                  |
| 关键字                       | `let`、`if`、`fn`                   |
| 数字字面量                   | `42`、`3.14`、`0xFF`、`1_000`       |
| 字符串字面量                 | `"hello"`、`r"raw\n"`、`f"x = {x}"`、`b"bytes"` |
| 字符字面量                   | `'A'`、`c'A'`                       |
| 布尔 / 哨兵字面量            | `true`、`false`、`none`             |
| 标签                         | `'outer`                            |
| 运算符 / 标点                | `+`、`==`、`..`、`[`、`;`           |

标签是 `'` 后跟一个标识符，没有闭合引号（`'outer`）；它为 `break` / `continue` 命名一个循环或块（[3.13](../003-expressions/313-loop-expressions.md)）。带闭合引号的 `'A'` 则是字符字面量。

格式字符串不是单个词法单元，而是一族词法单元。`f"`（或 `f"""`）开启它，字面文本段与插值孔交替出现，闭合的 `"`（或 `"""`）结束它。插值孔由一个 `{` 词法单元、其表达式的普通词法单元、可选的 `:` 加格式说明文本（其中可以含有嵌套的 `{...}` 或 `${...}` 插值孔）以及一个 `}` 词法单元组成（[1.7.3](107-string-literals.md#173-格式字符串)）。

各类词法单元的确切格式规则在后续文档中依次定义。

## 1.2.2. 空白

空白字符只作为词法单元之间的分隔符才有意义，本身不构成词法单元。

以下码位被识别为空白。

| 码位       | 名称                 |
|------------|----------------------|
| `U+0009`   | 制表符（TAB）        |
| `U+000A`   | 换行（LF）           |
| `U+000B`   | 垂直制表符（VT）     |
| `U+000C`   | 换页（FF）           |
| `U+000D`   | 回车（CR）           |
| `U+0020`   | 空格                 |

上表之外的 Unicode 类空格字符（`U+00A0` NO-BREAK SPACE、`U+200B` ZERO WIDTH SPACE 等）不是空白。它们只能作为普通码位出现在字符串和注释中；出现在其他任何位置都是词法错误。

缩进没有意义。Nabi 不采用 Python 式的有意义缩进。

## 1.2.3. 注释

Nabi 支持三种注释：行注释、块注释和文档注释。文件第一行的 shebang 行像注释一样被忽略。

### Shebang 行

如果文件的第一行（BOM 之后）以 `#!` 开头，该行就是 shebang 行，像注释一样被忽略，所以带有 `#!/usr/bin/env nabi` 的脚本可以用 `./script.nabi` 运行（[9.10.1](../009-tools/910-nabi-frontend.md#9101-子命令与全局选项)）。例外情况是：跳过空白和文档注释以外的注释后，`#!` 之后的第一个词法单元是 `[` 时，该行不是 shebang 行，`#!` 开启一个内部属性（`#![allow(unused)]`）。属性的 `#![` 是一个内部不含空白的词法单元（[1.10.3](110-punctuation-and-operator-tokens.md)），所以 `#! [allow(unused)]` 是错误。之后任何一行上的 `#!` 都不是 shebang 行。

### 行注释

```nabi
// comment to the end of this line
```

从 `//` 到该行的行终止符为止是注释。换行本身不属于注释。

### 块注释

```nabi
/* spanning
   multiple lines */
```

`/*` 与 `*/` 之间的文本是注释，并且允许嵌套。只有内部的 `/*`/`*/` 对都正确闭合之后，外层注释才会闭合（与 Rust 规则相同），这便于临时禁用大段代码。

```nabi
/* outer
   /* inner */
   still a comment */
```

### 文档注释

与 Rust 一样，文档注释有四种形式。

```nabi
/// outer doc comment (the following item)
//! inner doc comment (the enclosing file, module or function)
/** outer block doc comment */
/*! inner block doc comment */
```

- 行文档注释延伸到行终止符；`/// text` 即 `#[doc = "text"]`（标记之后的一个空格被去掉），`//! text` 即 `#![doc = "text"]`（[6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---文档注释形式)）。以 `////` 开头的行是普通注释。
- 块文档注释延伸到与之匹配的 `*/`，并像块注释一样嵌套。`/**/`、`/***/` 以及以 `/***` 开头的注释是普通注释。其文本构成一个 `#[doc]` / `#![doc]` 属性。空白的第一行和空白的最后一行被去掉。当注释跨越多行，且第一行之后的每一行都以可选空白加 `*` 开头时，去掉这个前缀。然后去掉非空白行共有的缩进以及 `*/` 之前的空白。换行为 LF。
- 内部文档注释只出现在文件、模块体或函数体的开头。出现在其他任何位置都是错误。
- 外部文档注释为其后的项编写文档。不为任何东西编写文档的外部文档注释（位于 `}`、`)`、`]` 或文件末尾之前）是错误，位于函数参数上的也是错误。位于语句、match 分支、结构体字面量字段、元组字段或表达式上的外部文档注释不被任何工具保留，并触发 `unused_doc_comments` lint（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）。
- 预构建包的 NABIHIR 接口文本（[9.4](../009-tools/904-nabihir-bytecode.md)）保留文档注释。

## 1.2.4. 编译器输出

词法分析器的输出由词法单元流和位置信息组成。位置信息遵循[源文件](101-source-files.md)中的诊断单位规则。

该输出是下一阶段（解析器）的唯一输入；解析器不再查看原始源代码。
