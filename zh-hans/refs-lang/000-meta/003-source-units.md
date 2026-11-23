# 0.3. 源单元

Nabi 的编译单元是**包**。包在文件系统上按固定约定（`src/main.nabi` / `src/lib.nabi`）识别。清单 `nabi.nabion` 只在声明依赖等信息时才存在；没有依赖的包不需要它（0.3.4）。

## 0.3.1. 可执行文件与库

一个包包含一个可执行文件、一个库，或两者都有。包含哪一种只由其源目录 `src/` 中的根文件决定：`src/main.nabi` 是可执行文件的根，`src/lib.nabi` 是库的根。

包目录在 `src/` 中存放包的源代码，其他一切放在它旁边：清单 `nabi.nabion` 和锁文件 `nabi.lock`（0.3.4）、`target/`（构建输出，[9.2.15](../009-tools/902-aot-pipeline.md#9215-输出)），以及 `README`、`LICENSE` 或测试数据等文件。只有 `src/` 下的文件才是包的源代码；`src/` 之外的 `.nabi` 文件不属于该包（它不随包一起编译，不计入包的标识符（0.3.3），单独运行时它是一个匿名包，[4.16.8](../004-functions/416-entry-point.md#4168-单文件执行)）。唯一的例外是 `tests/` 和 `benches/` 目录，它们只在 `nabic test` / `nabic bench` 时编译（[0.3.5](#035-测试与基准测试目录)）。

### 可执行包：`src/main.nabi`

包含 `src/main.nabi` 的目录被识别为可执行包。

```
my_app/
├── nabi.nabion     <- manifest (optional, 0.3.4)
├── src/
│   ├── main.nabi   <- entry point
│   └── ...         <- other module files
└── target/         <- build output
```

`main.nabi` 是入口模块，它以两种形式之一提供入口点：显式的 `fn main` 或顶层语句。入口点的确切形式和顶层语句的含义在 [4.16 入口点](../004-functions/416-entry-point.md)中定义。运行不属于任何包的单个文件时，该文件被视为一个匿名包的 `main.nabi`（[4.16.8](../004-functions/416-entry-point.md#4168-单文件执行)）。

### 库包：`src/lib.nabi`

包含 `src/lib.nabi` 的目录被识别为库包。

```
my_lib/
├── nabi.nabion
└── src/
    ├── lib.nabi    <- library root
    └── ...
```

库包没有入口点；它是一组供外部使用的定义。除 `main.nabi` 以外的每个模块文件（包括 `lib.nabi`）只能包含项，不得包含顶层语句，因此导入一个模块永远不会执行任何代码（[4.16.6](../004-functions/416-entry-point.md#4166-模块只包含定义)）。

### 同一个包中的库与可执行文件

一个包可以同时拥有 `src/main.nabi` 和 `src/lib.nabi`。它们是从同一个 `src/` 构建的两个目标：库，以及使用该库的可执行文件。可执行文件把库看作一个以包名命名的依赖（`use my_app::parse;`；即 [9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock) 中的代码名，`-` 替换为 `_`）。库与可执行文件是不同的包：在可执行文件中只能看到库的 `pub` 项，看不到 `pub(pkg)`、`pub(sub)` 或私有项（[5.7.14](../005-modules/507-paths.md#5714-包之间的路径)），并且在可执行文件 `use` 它之前，它的任何项都不在作用域中。两个目标中的 `pkg::` 都指该目标自己的根。每个目标有自己的模块树，所以两者都可以为同一个 `src/util.nabi` 声明 `mod util;`，各自得到一份副本。依赖这个包的其他包只得到库。

## 0.3.2. 文件与模块

包内的其他模块是 `src/` 下的文件，采用以下两种形式之一。

```
foo.nabi            <- single-file module (no submodules)
```

或者可以附带一个文件夹：

```
foo.nabi            <- module body
foo/                <- foo's submodules
├── bar.nabi        foo::bar submodule
└── baz.nabi        foo::baz submodule
```

在文件夹形式中，主体文件（`foo.nabi`）与同名文件夹（`foo/`）并列。只有在主体中用 `mod name;` 声明的子模块才映射到文件夹中的文件。这与 Rust 2018+ 的规则相同；详细的映射规则在[模块](../005-modules/504-modules.md)中定义。

此外，也可以不使用单独的文件，而以 `mod name { ... }` 或 `pub mod name { ... }` 的形式在主体中放置内联模块。它的行为与 Rust 的内联模块相同；其详细形式和映射同样在[模块](../005-modules/504-modules.md)中定义。

## 0.3.3. 包标识符

每个包被赋予一个基于内容的哈希。哈希的输入是该包所有源文件的拼接：按相对于 `src/` 的路径（以 `/` 分隔）的字节顺序排序，每个文件序列化为 `path + \0 + contents + \0`。取其 SHA-256 的前 8 个字符作为标识符。

该哈希具有以下性质：内容相同的包无论在哪里构建都产生相同的哈希，哪怕只改动一个字符也会产生新的哈希。

该哈希用于产物的符号名和缓存键；具体用途在后续文档中定义。

## 0.3.4. 清单（`nabi.nabion`）是可选的

什么构成一个包只由文件是否存在决定。如果目录中存在 `src/main.nabi` 或 `src/lib.nabi`，该目录就是一个包，二者中存在哪些决定了它包含可执行文件、库，还是两者都有。清单不会改变这一约定。

只有在需要依赖、开发依赖或工具链固定等信息时，才在包根目录、`src/` 旁边放置清单 `nabi.nabion`（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）；工具解析出的结果记录在锁文件 `nabi.lock` 中。这两个文件名的拼写必须完全如此（全部小写）。大小写不同的名称（`Nabi.nabion` 等）会产生警告。两个文件都使用 [NABION](../../refs-standard-libraries/300-std/308-nabion.md) 格式；清单只作为单个 `Package { .. }` 值读取，永远不会被编译或执行。

这一决定带来两点好处。第一，编译器只看文件就能处理没有依赖的包。第二，清单只做增补，永远不会破坏已有的包定义。

## 0.3.5. 测试与基准测试目录

一个包可以在 `src/` 旁边放置 `tests/` 和 `benches/` 目录。与 `src/` 之外的其他文件（0.3.1）不同，它们会被编译，但只用于对应的命令：`tests/` 用于 `nabic test`，`benches/` 用于 `nabic bench`。它们永远不会出现在普通构建、库接口（`.nabihir`）或包标识符（0.3.3）中，这些都只包含 `src/`。

每个目录作为一个合成的根模块挂载（`tests/` 挂在模块 `tests` 下，`benches/` 挂在 `benches` 下），按[模块](../005-modules/504-modules.md#544-包根)的普通规则布局：`tests/parse.nabi` 是 `tests::parse`，它旁边的 `tests/parse/` 文件夹存放该模块的子模块。不存在 `mod.nabi`（[5.4.6](../005-modules/504-modules.md#546-自定义路径path)）；命名方式与 `src/` 下相同。

这些代码属于包内：与包一起编译，通过 `pkg::` 访问包，具有与 `src/` 下模块相同的包内可见性（而不是依赖方只能看到 `pub` 的视图），因此可以测试 `pub(pkg)` 项以及包自身的非公开模块。它们可以使用清单中的 `dev_deps`（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）。在同时拥有 `src/lib.nabi` 和 `src/main.nabi` 的包中，这两个目录基于库构建；可执行文件自身的测试以内联方式留在 `src/` 下。

这里的文件隐式地只用于测试或基准测试，所以不需要 `#[cfg(test)]`。用 `#[test]` 或 `#[bench]` 标记要测试或测量的函数（[6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-测试函数标记)）；目录中的其他项是共享的辅助代码。

```
my_lib/
├── nabi.nabion
├── src/
│   └── lib.nabi
├── tests/              <- nabic test only
│   ├── parse.nabi      tests::parse   (uses pkg::...)
│   └── common.nabi     tests::common  (shared helper, no #[test])
└── benches/            <- nabic bench only
    └── parse.nabi      benches::parse
```
