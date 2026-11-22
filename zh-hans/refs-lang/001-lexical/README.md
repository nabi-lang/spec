# 1. 词法结构

本章介绍把 Nabi 源代码切分为词法单元的规则。从文件和行开始，依次讨论标识符、关键字、字面量和运算符词法单元，涵盖切分词法单元时遇到的每一种元素。

类型、表达式、语句等更高的层次接收的都是已经切分好的词法单元，所以切分词法单元时适用的规则全部集中在本章。

## 目录

- [1.1. 源文件](101-source-files.md)。扩展名、编码、行终止符。
- [1.2. 词法结构](102-lexical-structure.md)。词法单元、空白、注释。
- [1.3. 语句终止符](103-statement-terminator.md)。`;` 以及可以省略它的位置。
- [1.4. 标识符](104-identifiers.md)。名称的格式。
- [1.5. 关键字](105-keywords.md)。保留字列表。
- [1.6. 数字字面量](106-numeric-literals.md)。
- [1.7. 字符串字面量](107-string-literals.md)。
- [1.8. 字符字面量](108-character-literals.md)。
- [1.9. 布尔字面量与哨兵字面量](109-other-literals.md)。`true`、`false`、`none`。
- [1.10. 标点与运算符词法单元](110-punctuation-and-operator-tokens.md)。
