# 6.11. 编译期文件嵌入

## 6.11.1. `include_str!` / `include_bytes!`

```nabi
const TEMPLATE: str = include_str!("templates/page.html");
static ICON: [u8; 1406] = include_bytes!("res/icon.png");
```

`include_str!("path")` 求值为文件 `path` 的内容，类型为 `str`；`include_bytes!("path")` 求值为其字节，类型为 `[u8; N]`（`N` 为文件长度；即字节串，[1.7.5](../001-lexical/107-string-literals.md#175-byte-strings)，包括其 `Slice<u8>` 规则）。实参恰好是一个字符串字面量。路径相对于包含该调用的源文件所在的目录。文件在编译期读取；结果是一个常量，可以出现在任何允许字面量出现的位置，包括 `const` 和 `static` 的初始化表达式。

如果文件无法读取，或者对 `include_str!` 而言文件内容不是有效的 UTF-8，就是编译错误。被包含的文件是构建的一项输入：修改它会重新构建使用它的部分（[9.2.16](../009-tools/902-aot-pipeline.md#9216-incremental-build)）。

与 `cfg!`（[6.3.5](603-cfg.md#635-cfg-compile-time-bool-value)）一样，它们是编译器内置形式，而不是用户定义的宏。
