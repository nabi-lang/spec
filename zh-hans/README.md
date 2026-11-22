# Nabi 语言规范

[English](../en/README.md) · [한국어](../ko/README.md) · **简体中文**

> **状态**：活跃的草稿，描述 Nabi 0.1。规范以三个版本发布：英文版（[`en/`](../en/README.md)，默认）、韩文版（[`ko/`](../ko/README.md)）和简体中文版（`zh-hans/`）。各版本以相同的节号涵盖相同的内容。

Nabi 是一门编程语言，本仓库是它的形式规范。它从词法结构和类型系统开始，在一处定义语言的每一层：表达式、运行时模型、ABI，以及伴随规范的工具。编译器和标准库的实现位于单独的仓库中；本仓库只保存对“Nabi 是什么”这一问题的规范性回答。

由 [Han Damin](https://github.com/miniex)（`@miniex`）设计和维护。

| 内容                 | 位置                                                    |
|----------------------|---------------------------------------------------------|
| GitHub 组织          | [nabi-lang](https://github.com/nabi-lang)                 |
| 规范仓库             | <https://github.com/nabi-lang/spec>                      |
| 网站                 | <https://nabi-lang.org>                                  |
| 文档                 | <https://docs.nabi-lang.org>                             |
| 包注册表             | [packages.nabi-lang.org](https://packages.nabi-lang.org)        |
| 注册表索引           | <https://github.com/nabi-lang/packages-index>          |

## Nabi 为何存在

Nabi 源于它的作者想要编写的某一类代码。它不面向特定的受众，也不主张某种学术立场。每个设计决定的核心都是一个简单的动机：“一门能做作者想做的一切的语言”。这句话差不多就是 Nabi 的全部。

## 理念

一门出于个人动机诞生的语言，需要在每次做决定时都能依靠的原则，才能保持一致。两条标语指引着 Nabi 的设计。

> [!IMPORTANT]
> **“核心要窄，库要丰富。”**

第一条原则是关于规范大小的承诺。Nabi 规范定义得尽可能少。几乎所有决定语言*手感*的东西（数值语义、转换、集合类型、I/O、并发）都位于标准库中。小的核心更容易演进、验证和教授；各个领域的分量改由丰富的库来承担。

> [!IMPORTANT]
> **“规则越清晰，你在其中就越自由。”**

第二条原则关乎自由与约束之间的关系。Nabi 不回避约束。静态类型、封闭的和类型、trait 不能作类型、禁止隐式提升：每一个决定都有意收窄了用户可以走的路。在收窄后的空间里，代码变得可预测，并且可以深入控制。

控制与自由之间的界线是清晰的。能够一次决定、无须在每一行重新决定的东西（类型、分派、内存模型、求值顺序），由语言一次性决定。必然因人而异的东西（库设计、领域建模、算法、代码结构）留给用户，元系统则让语言能够扩展自身，使库可以决定含义。

因此，自由与控制并不对立。Nabi 的立场是：自由的领域只有建立在受到良好控制的核心之上，才能安全地存续。关于这两条标语的完整讨论见 [`refs-lang/000-meta/`](refs-lang/000-meta/)。

## 布局

仓库有固定的结构。顶层目录按语言版本划分，每个版本包含两份参考：

- [`refs-lang/`](refs-lang/)：语言参考，涵盖词法结构、类型、表达式、运行时模型、ABI 和工具。
- [`refs-standard-libraries/`](refs-standard-libraries/)：标准库参考。

每个版本都遵循这一结构：`en/refs-lang/`、`ko/refs-lang/`、`zh-hans/refs-lang/` 等，文件相同，节号相同。

## 版本

规范有三个语言版本：

- **English**（[`en/`](../en/README.md)）：默认版本。
- **한국어**（[`ko/`](../ko/README.md)）：韩文。
- **简体中文**（`zh-hans/`）：本版本。

各版本使用相同的文件布局和节号（`3.11.4`），所以同一个节号在任何版本中都指同一条规则。如果版本之间有出入，以英文文本为准，这种出入是需要在各版本中修正的缺陷。

## 阅读顺序

规范按从第 0 章到第 9 章的顺序阅读而写。新概念总是只用到那时为止已介绍的概念来解释，这一不变量在每一章都成立。无论翻开哪一章，它所需要的概念都来自本章或之前的章节。有些交叉引用指向后面完整讨论某个主题的地方；它们是资料性的，章节并不依赖它们。初次阅读的读者应当从 [`refs-lang/README.md`](refs-lang/README.md) 开始。

## 约定

代码围栏中的 ASCII 图和制表符图使用英文标签，并在其下方的文字中解释。除了翻译过的注释、附带的字符串，以及代表用户所写部分的语法占位符（元变量）之外，各版本中的代码示例完全相同。

## 术语

简体中文版通篇一致地使用下列术语（同时列出英文版的对应词，供译者参考）。[术语表](refs-lang/000-meta/007-glossary.md)定义了核心术语。

| English | 简体中文 |
|---|---|
| specification, chapter, section | 规范，章，节 |
| language reference / standard library reference | 语言参考 / 标准库参考 |
| sum type | 和类型 |
| variant | 变体 |
| trait; trait-as-type ban | trait；trait 不能作类型 |
| associated type / constant | 关联类型 / 常量 |
| monomorphization; concrete | 单态化；具体的 |
| slot; `any` slot; lang slot ("magic") | 槽；`any` 槽；lang 槽（“魔法”） |
| implicit promotion / conversion | 隐式提升 / 隐式转换 |
| conversion / cast | 转换 / 强制转换 |
| expression / statement; statement terminator | 表达式 / 语句；语句终止符 |
| block; body | 块；主体 |
| arm (of `match`) / branch | （`match` 的）分支 / 分支 |
| pattern; binding; destructuring (assignment) | 模式；绑定；解构（赋值） |
| optional chaining / coalescing | 可选链 / 空值合并 |
| broadcast | 广播 |
| literal; identifier; keyword; reserved word | 字面量；标识符；关键字；保留字 |
| boolean; sentinel; floating-point; integer | 布尔值；哨兵值；浮点数；整数 |
| punctuation; trailing comma | 标点；尾随逗号 |
| parameter / argument | 参数 / 实参 |
| closure; capture; coroutine; async | 闭包；捕获；协程；异步 |
| entry point | 入口点 |
| module; visibility; path; item | 模块；可见性；路径；项 |
| package; registry | 包；注册表 |
| manifest; lockfile; environment; dependency; pin | 清单；锁文件；环境；依赖；固定 |
| meta-system; attribute; template tokens | 元系统；属性；模板记号 |
| runtime; object; root; barrier; generation | 运行时；对象；根；屏障；代 |
| lowering; dialect; symbol | 降级（lowering）；方言；符号 |
| error; "is an error" / "is a compile error"; diagnostic | 错误；“是错误” / “是编译错误”；诊断 |
| raw pointer; collection; iterator | 裸指针；集合类型；迭代器 |
| evaluation order; incremental build | 求值顺序；增量构建 |

已经是英文的名称在各版本中都保持原样：NABION、`nabi.nabion`、`nabi.lock`、`nabic`、`nabi`、`nabicc`、`nabils`、`nabir`、NABIHIR、HIR、MIR、LIR、AOT、JIT、GC、Immix、LOS、TLAB、shadow stack、`scan_table`、niche optimization、sticky mark-bit、work-stealing、ABI、FFI、toolchain、sysroot。

## 许可证

本规范以 [Creative Commons Attribution 4.0 International (CC BY 4.0)](../LICENSE) 发布。

Copyright © 2026 DaminStudio and the Nabi contributors

只要注明出处，就可以自由地分享、改编和翻译文本。许可证全文见 [LICENSE](../LICENSE)；易读的摘要见 <https://creativecommons.org/licenses/by/4.0/deed.zh-hans>。

## 商标

“Nabi”是本项目的名称，也是规范正本的标识。CC BY 4.0 只涵盖**文本**：只要注明出处，文本本身可以自由地复制、翻译和改编。但是，请不要把“Nabi”这个名称用于派生的规范、替代规范或竞争性的实现。请使用其他名称，以免与正本项目混淆。

这是一项**请求**，而不是注册商标。如果项目发展到需要正式保护的程度，会考虑注册。
