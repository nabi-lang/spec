# 6.2. 内置属性

本节定义由编译器直接处理的内置属性。`#[cfg]` 单独在[条件编译](603-cfg.md)中介绍，魔法属性与元块从 [`#[lang(...)]` 魔法](604-lang-magic.md)一节起说明。

```
derive_attr     ::= "#" "[" "derive" "(" trait_name ("," trait_name)* ","? ")" "]"
trait_name      ::= identifier
repr_attr       ::= "#" "[" "repr" "(" repr_arg ("," repr_arg)* ")" "]"
repr_arg        ::= "C" | "transparent" | "packed" | "align" "(" int_lit ")"
                  | identifier                          // primitive int repr
inline_attr     ::= "#" "[" "inline" ("(" ("always" | "never") ")")? "]"
deprecated_attr ::= "#" "[" "deprecated" ("(" deprecated_args ")" | "=" string_lit)? "]"
deprecated_args ::= string_lit | deprecated_kv ("," deprecated_kv)*
deprecated_kv   ::= ("since" | "note") "=" string_lit
deprecated_safe ::= "#" "[" "deprecated_safe" ("(" string_lit ")" | "=" string_lit)? "]"
sealed_attr     ::= "#" "[" "sealed" "]"
internal_attr   ::= "#" "[" "internal" "]"
discardable_attr ::= "#" "[" "discardable" "]"
```

## 6.2.1. `#[derive(...)]` 自动实现 trait

```nabi
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
struct Point { x: i64; y: i64; }
```

`#[derive(...)]` 的实参是用逗号分隔的 trait 名称。对列出的每个 trait，都会生成适合该类型的自动实现。

### 可派生的内置 trait

只有 `Clone`、`Copy`、`Debug`、`Default`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash` 和 `Error` 可以派生。每个 trait 的生成规则和要求在[标准库 2.13.11](../../refs-standard-libraries/200-core/213-marker.md#21311-derivable-traits) 中定义。

- `Display` 不能派生。显示格式由类型的作者决定，通过 `impl Display` 或 `#[display("..")]` 给出（[标准库 2.10.14](../../refs-standard-libraries/200-core/210-fmt.md#21014-derive-policy-summary)）。
- `Error` 以默认的 `source()` 实现 `Error`；`Display` 与 `Debug` 来自它们各自的 derive、属性或 impl（[标准库 2.14](../../refs-standard-libraries/200-core/214-error.md#deriveerror)）。
- 派生 `Eq` / `Ord` 而不派生其父 trait（`PartialEq` / `PartialOrd`、`Eq`）是编译错误。`Copy` 要求 `Clone`，且不能与 `Drop` 同时使用。
- 在 `union` 上使用 `#[derive]` 是编译错误。

### 与用户定义的 derive 组合

用户定义的 trait 也可以派生，如 `#[derive(MyTrait)]`。这要求 `MyTrait` 由 `#[impl_derive(MyTrait)]` 元块（[6.8](608-meta-blocks.md)）定义。上述内置 derive 名称不能被元块重新定义。

内置 derive 与用户定义的 derive 可以在同一个 `#[derive(...)]` 中自由混用。

```nabi
#[derive(PartialEq, Eq, Clone, MyCustomTrait)]
struct Foo { ... }
```

## 6.2.2. `#[repr(...)]` 显式内存表示

```nabi
#[repr(C)]
struct CCompatible { x: i32; y: i32; }

#[repr(transparent)]
struct Wrapped(InnerType);

#[repr(packed)]
struct Tight { a: u8; b: u32; }

#[repr(align(16))]
struct Aligned { ... }

#[repr(u32)]
enum Status { ... }
```

| 实参                        | 含义                                                                                                 |
|-----------------------------|------------------------------------------------------------------------------------------------------|
| `C`                         | C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 布局：按声明顺序，自然对齐。与 `struct` 的默认布局相同 |
| `transparent`               | 恰好一个字段，且没有其他 `repr` 实参。表示与该字段相同，并以该字段的形式跨越 C ABI |
| `packed`                    | 忽略对齐，紧密排列（仅用于 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface)）。不能持有 GC 引用；与 `align(N)` 互斥 |
| `align(N)`                  | 强制至少 N 字节的对齐                                                                                |
| `u8` 至 `u64` / `i8` 至 `i64` | `enum` 的标签类型。32 位以外的宽度只用于没有字段的枚举；带数据的枚举接受 `i32` / `u32` |

没有 `#[repr]` 时，[`struct`](../002-types/203-struct.md) 按声明顺序和自然对齐排列（与 `repr(C)` 相同）；[`enum`](../002-types/204-enum.md) 与和类型的布局由编译器决定（[2.7](../002-types/207-niche-optimization.md)）。

## 6.2.3. `#[link(...)]` 指定外部库

```nabi
#[link("m")]
extern "C" {
    fn sin(x: f64) -> f64;
    fn cos(x: f64) -> f64;
}

#[link(name = "openssl", kind = "static")]
extern "C" { ... }
```

`#[link(...)]` 指定 [`extern` 块](../005-modules/509-extern-block.md)要链接的外部库；它附着在 `extern` 块上。

| 实参形式                    | 含义                        |
|-----------------------------|-----------------------------|
| `"libname"`                 | 普通库名                    |
| `name = "..."`              | 库名（键值形式）            |
| `kind = "static" / "dylib" / "framework"` | 链接方式（`framework`：Apple 框架） |
| `wasm_import_module = "m"` | wasm32：块中的函数从核心模块 `m` 导入；在其他目标上忽略 |

### `#[link_name = "sym"]`

extern 函数上的 `#[link_name = "sym"]` 给出它所链接的符号，在 `wasm_import_module` 块中则给出它的导入名。

```nabi
#[link(wasm_import_module = "env")]
extern "C" {
    #[link_name = "log_line"]
    fn host_log(ptr: ConstPtr<u8>, len: usize);
}
```

## 6.2.4. `#[inline]` 内联提示

```nabi
#[inline]
fn small_helper(x: i32) -> i32 { x + 1 }

#[inline(always)]
fn very_hot() { ... }

#[inline(never)]
fn never_inline() { ... }
```

| 实参     | 含义                                          |
|----------|-----------------------------------------------|
| （无）   | 建议内联（由编译器决定）                      |
| `always` | 强制内联                                      |
| `never`  | 禁止内联                                      |

`#[inline]` 只是优化提示，不影响程序的含义。

## 6.2.5. `#[test]` 测试函数标记

```nabi
#[test]
fn test_addition() {
    assert_eq(1i32 + 1i32, 2i32);
}
```

`assert_eq` 见[标准库 2.14.5](../../refs-standard-libraries/200-core/214-error.md#assert--assert_eq--assert_ne)。函数上的 `#[test]` 把它标记为测试函数。`nabic test` 把这些函数收集到测试框架中，测试运行器（`nabi test` 等）会自动运行它们。在普通构建中它们被排除，就像同时附着了 `#[cfg(test)]` 一样。无论测试函数自身及其外层模块的可见性如何，测试函数都会运行；`#[test]` 不改变其他代码所见的可见性。

有两个属性与 `#[test]` 配合使用，说明怎样才算通过：

```nabi
#[test]
#[should_panic(expected = "empty input")]
fn rejects_empty() {
    parse_config("");
}

#[test]
#[ignore = "needs the network"]
fn downloads() { }
```

- `#[should_panic]` 只在测试触发 panic 时通过；`#[should_panic(expected = "text")]` 还要求 panic 的输出包含 `text`。正常返回或以未捕获的错误结束的测试判为失败。
- `#[ignore]`（或 `#[ignore = "reason"]`）使测试不参加普通运行；它被报告为已忽略，并附上原因。`nabi test --ignored` 只运行这类测试，`--include-ignored` 也会运行它们（[9.10.1](../009-tools/910-nabi-frontend.md#9101-subcommands-and-global-options)）。

在没有 `#[test]` 的函数上使用其中任一属性是错误，错误会提示添加 `#[test]`；任何其他实参形式也是错误，错误会列出这两种形式。

`#[bench]` 标记基准测试函数，并且与 `#[test]` 一样隐含 `#[cfg(test)]`。`nabic bench` 收集这些函数并构建测试框架，`nabi bench` 运行它（[9.1.2](../009-tools/901-nabic-cli.md#nabic-bench-benchmarks)）。无论基准测试函数自身及其外层模块的可见性如何，它都会运行；`#[bench]` 不改变其他代码所见的可见性。

## 6.2.6. `#[cold]` 很少调用的函数

```nabi
#[cold]
fn handle_error() { ... }
```

`#[cold]` 告诉编译器该函数很少被调用。代码生成器随后把该函数放在热代码路径之外，以改善缓存友好性。

## 6.2.7. `#[deprecated]` 不推荐使用的标记

```nabi
#[deprecated]
fn old_api() { ... }

#[deprecated = "use new_api instead"]
fn old_api2() { ... }

impl Config {
    #[deprecated(since = "1.2.0", note = "use `Config::load`")]
    pub fn new() -> Config { ... }
}
```

使用标记为 `#[deprecated]` 的项会产生警告（lint `deprecated`），而不是错误。以 `= "..."`、`("...")` 或 `note = "..."` 给出的消息会随警告一起显示；消息应当说明应改写成什么。`since = "x.y.z"` 指明弃用该项的版本。

无论这些项在哪里声明（包本身、依赖或标准库），警告都覆盖以下各项：

- 被调用或作为值使用的函数和方法；trait 方法上的标记覆盖它的所有 impl；
- 被读取、被赋值、在结构体字面量中被设置或在模式中被命名的字段；
- 在类型中或作为路径开头（`Old::new`）被命名的结构体、枚举、联合体、类型别名和 trait；
- 常量和静态变量。

只有正在构建的包中的使用才会发出警告（[6.2.9](#629-allow--deny--warn-lint-control)）。类型自身的 `impl` 不会因该类型发出警告，结构体自身的方法也不会因其字段发出警告。类型通过其完整路径识别，或者通过没有其他类型共用的最后一段识别。该属性可以用在任何项上；在其他项（变体、impl、模块）上它没有效果。

`since` 只是资料性的：任何版本的弃用都归入 `deprecated` 和 `warnings`。语言服务器把这类使用标记为已弃用，编辑器因此会给它加删除线。

## 6.2.8. `#[doc = "..."]` 文档注释形式

```nabi
#[doc = "Returns the sum of two integers."]
fn add(x: i32, y: i32) -> i32 { x + y }
```

[词法结构](../001-lexical/102-lexical-structure.md#doc-comments)中的 `///` 文档注释就是这个属性。

```nabi
/// This function ...
fn f() { ... }

// same meaning
#[doc = "This function ..."]
fn f() { ... }
```

内部形式 `#![doc]` 同样与 `//!` 效果相同。

### `#[doc(hidden)]`

```nabi
/// What a dict spread checks (lowered code calls it).
#[doc(hidden)]
pub const fn spread_misfit<V>(d: Dict<str, V>, key: str, why: str) { .. }
```

`#[doc(hidden)]` 让公开项不出现在 API 文档（`nabi doc`）和语言服务器的补全中。它用于那些只是为了让编译器生成的代码能调用才设为 `pub` 的项；该项仍是公开的，编译和解析都与之前一样。用在模块上时，它隐藏模块内的一切。它可以与该项的文档注释并存，文档注释仍用于悬停提示。任何其他 `#[doc(..)]` 形式都是错误：应写 `#[doc = ".."]` 或 `#[doc(hidden)]`。工具链的各个包共用、但用户不得调用的项使用 `#[internal]`（[6.2.16](#6216-internal-shared-by-the-toolchains-packages)），而不是 `#[doc(hidden)]`。

## 6.2.9. `#[allow(...)]` / `#[deny(...)]` / `#[warn(...)]` lint 控制

```nabi
#[allow(unused)]
fn _placeholder() { }

#[deny(deprecated)]
fn strict_zone() { ... }
```

这些属性在所附着项的作用域内改变 lint 诊断的级别。lint 在每个编译该包的命令中运行：`nabic build`、`test`、`bench`、`emit` 和 `check`（[9.1.2](../009-tools/901-nabic-cli.md#912-core-subcommands)），`nabi run`、`nabi lint`（[9.10.8](../009-tools/910-nabi-frontend.md#nabi-lint-linter)），REPL 以及语言服务器（`doc` 除外）。它们检查正在构建的包，而不检查其依赖。`warn` 级别的 lint 会被打印出来，但不会使构建失败；`deny` 级别的 lint 是错误，构建以退出码 1 失败。`allow` 忽略该诊断，`warn` 把它作为警告发出，`deny` 把它作为错误发出。`#![allow(..)]` 这样的内部形式作用于外层模块（写在文件开头时作用于整个文件）。实参是 lint 名或组名。项上的 `#[allow]` 也覆盖它的变体、字段和方法。

级别按以下顺序确定，后者覆盖前者：各 lint 的默认级别 < `nabi.nabion` 的 `lints` 字段（经过其 `lints_from` 链之后）（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）< 命令行的 `-A` / `-W` / `-D` < 源代码中的属性（最内层的优先）。`lints` 被视为加在命令行标志之前，因此写在它之后的命令行标志会覆盖它。lint 诊断的代码是 `E0131`。

每个 lint 的默认级别都是 `warn`，例外是 `nan_comparison` 和 `self_assignment`：它们总是错误写法，默认为 `deny`；以及 `missing_docs`，默认为 `allow`。“起始版本”一列是加入该 lint 的版本。

| lint                     | 起始版本 | 检查内容                                                            |
|--------------------------|---------|---------------------------------------------------------------------|
| `unused_variables`       | 0.1.0   | 从未被读取的局部变量或参数。被赋值的 `ref` 参数，或以 `ref` 捕获（`[ref n]`）的变量，算作已使用：写入会传到调用者 |
| `unused_imports`         | 0.1.0   | 模块从未使用其名称的 `use`                                          |
| `dead_code`              | 0.1.0   | 没有人调用的非 `pub` 函数                                           |
| `unreachable_code`       | 0.1.0   | `return` / `break` / `continue` / `throw` / `panic` 之后的语句      |
| `unused_must_use`        | 0.1.0   | 丢弃 `#[must_use]` 函数的结果，或丢弃 `#[must_use]` 类型的值（[6.2.11](#6211-unused-results-unused_result-discardable-must_use)） |
| `unused_result`          | 0.1.0   | 值不是 `()`、`none` 或 `Never` 的表达式语句，调用 `#[discardable]` 函数的除外；要有意丢弃值，写 `_ = expr;`（[6.2.11](#6211-unused-results-unused_result-discardable-must_use)） |
| `unused_doc_comments`    | 0.1.0   | 写在不会被文档化之处（语句、match 分支、结构体字面量或元组字段、表达式）的文档注释；普通注释用 `//` |
| `ignored_error`          | 0.1.0   | 悄悄丢弃错误的 `try? f();` 语句；`let _ = try? f();` 表示有意的尽力而为调用，不会被报告 |
| `nan_comparison`         | 0.1.0   | 与 NaN 比较（结果总为假；`!=` 总为真）；应使用 `.is_nan()`（deny） |
| `bool_comparison`        | 0.1.0   | 把 `bool` 与 `true` / `false` 比较                                  |
| `self_assignment`        | 0.1.0   | `x = x;`（deny）                                                    |
| `while_true`             | 0.1.0   | 用 `while true` 代替 `loop`                                         |
| `closed_type_switch`     | 0.1.0   | 回退分支触发 panic 的 `if T is ..` 链：一组封闭的类型适合用 trait 表达 |
| `deprecated`             | 0.1.0   | 使用 `#[deprecated]` 的函数、方法、字段、类型、常量或静态变量（[6.2.7](#627-deprecated-discouraged-use-marker)） |
| `deprecated_safe`        | 0.1.0   | 在 `unsafe` 之外调用 `#[deprecated_safe]` 函数，或使用这样的静态变量（[6.2.14](#6214-deprecated_safe-safe-today-unsafe-later)） |
| `redundant_cast`         | 0.1.0   | `x` 已经是 `T` 时的 `x as T`                                        |
| `redundant_clone`        | 0.1.0   | 对基本类型的值调用 `.clone()`                                       |
| `non_snake_case`         | 0.1.0   | 不是 snake_case 的函数、局部变量、参数或模块名                      |
| `non_camel_case_types`   | 0.1.0   | 不是 UpperCamelCase 的类型、trait 或变体名：首字母不得小写，不得有 `__`，`_` 只能出现在没有大小写之分的字符之间（`V1_0`） |
| `non_upper_case_globals` | 0.1.0   | 不是 UPPER_SNAKE_CASE 的 `const` / `static` 名                      |
| `sup_path`               | 0.1.0   | 内联 `mod name { }` 之外的 `sup::` 路径或 `pub(sub)`；应从包根写出路径（`pkg::a::b`，附注会给出它）或使用 `pub(pkg)`（[5.7](../005-modules/507-paths.md)） |
| `local_use`              | 0.1.0   | 函数体内的 `use`；把它移到文件顶部，与其他 `use` 项放在一起（[5.5.8](../005-modules/505-use.md)） |
| `confusable_idents`      | 0.1.0   | 彼此相似的标识符（[1.4](../001-lexical/104-identifiers.md)）        |
| `hidden_chars`           | 0.1.0   | 直接写在字符串、字符字面量或注释中的不可见字符或未分配字符；应写成 `\u{..}` 转义（[1.1.2](../001-lexical/101-source-files.md#112-character-encoding)） |
| `missing_f_prefix`       | 0.1.0   | 普通字符串字面量包含 `{name}`，而 `name` 是作用域中的局部变量：缺少 `f` 前缀 |
| `literal_spread_mismatch` | 0.1.0   | 展开到固定参数中的 `List` 或 Dict 字面量无法填满这些参数：长度不同，或某个键没有对应的剩余参数（[4.2](../004-functions/402-parameters.md)）；该调用会在运行时触发 panic |
| `assoc_type_shadowing`   | 0.1.0   | 关联类型与外层类型同名，在其 trait 或 `impl` 内遮蔽了外层类型（2.8.3） |
| `plain_module_doc`       | 0.1.0   | 库的根文件或公开模块（路径上的每个模块都是 `pub`）的文件，没有 `//!` 文档却以 `//` 行开头：这些行看似模块文档，实际并不是（[9.10.8](../009-tools/910-nabi-frontend.md#nabi-doc-documentation-generation)）；帮助信息会给出应写的 `//!` 行 |
| `missing_docs`           | 0.1.0   | 没有 `///` 文档注释的 `pub` 项、固有 impl 的 `pub` 方法、`pub` trait 的方法或内联 `pub mod`；`#[doc(hidden)]` 视为已有文档，trait impl 的方法不检查（allow） |
| `resize_while_iterating` | 0.1.0   | 对 `List` 的 `for x in xs`，其循环体改变了 `xs` 的长度（`push`、`remove` 等），且同一块中其后没有 `break`、`return` 或 `throw`：循环会看到新的长度（[3.13.3](../003-expressions/313-loop-expressions.md#3133-for-iterator-based-repetition)） |

| 组                  | 包含                                                                                         |
|---------------------|----------------------------------------------------------------------------------------------|
| `unused`            | `unused_variables`、`unused_imports`、`dead_code`、`unreachable_code`、`unused_must_use`、`unused_result`、`unused_doc_comments` |
| `nonstandard_style` | `non_snake_case`、`non_camel_case_types`、`non_upper_case_globals`、`sup_path`、`local_use` |
| `warnings`          | 级别为 warn 的所有 lint，包括后续版本加入的                                                  |
| `all`               | 所有 lint，包括后续版本加入的                                                                |

`warnings` 采用 rustc 的含义：对它使用 deny（`lints` 中的 `"warnings": "deny"`、`-D warnings` 或 `#![deny(warnings)]`）会把每个警告都变成错误，包括后续版本加入的 lint 和弃用。必须跨版本持续构建的项目应当固定其工具链（`nabi.nabion` 中的 `toolchain`，[9.10.6](../009-tools/910-nabi-frontend.md)）。各 lint 的“起始版本”一列只是资料性的。

有些警告来自编译器本身而不是 lint 阶段，但仍属于某个 lint：易混淆的标识符属于 `confusable_idents`，字符串或注释中的不可见字符属于 `hidden_chars`，位置不当的文档注释属于 `unused_doc_comments`。它们的级别与其他 lint 一样，通过 lint 名设置。

每条 lint 诊断都会说明其级别从何而来：`` `#[warn(unused_variables)]` on by default ``、`` `-D unused` on the command line ``，级别来自清单时为 `` `"unused_variables": "deny"` in `lints` of nabi.nabion ``，或者在设置该级别的属性上加标签。未知的 lint 名或组名会产生警告，警告给出最接近的已知名称，并指向 `nabi lint --list`。在属性、`-A` / `-W` / `-D` 或 `lints` 中写出的已移除 lint（大多已变成错误）会被接受但不起作用；即使在 `-D warnings` 下也不会报告。

`nabi lint --list` 打印所有 lint 及其默认级别和加入的版本，然后打印各组。

有些检查结果带有能解决问题的编辑（`while true` -> `loop`，删除未使用的 `use`）；编辑器把它作为快速修复提供，`nabi fix`（[9.10.8](../009-tools/910-nabi-frontend.md#nabi-fix-apply-fixes)）则在整个包中应用这些编辑。

`nabic check --check-library`（[9.1.2](../009-tools/901-nabic-cli.md#nabic-check-diagnostics)）和 `nabi lint --check-library`（[9.10.8](../009-tools/910-nabi-frontend.md#nabi-lint-linter)）还会按标准库自己的级别对标准库本身的代码运行 lint：全部采用默认级别，其中 `missing_docs` 为 `deny`，即标准库的每个公开项都有文档注释（[标准库 0.5.14](../../refs-standard-libraries/000-meta/005-notation.md#0514-docs-and-attributes)）。

## 6.2.10. `#[non_exhaustive]` 退出外部穷尽性

```nabi
#[non_exhaustive]
pub enum ApiError {
    NotFound,
    Unauthorized,
}
```

`#[non_exhaustive]` 可以附着在 `enum` 或 `struct` 上。在 `enum` 上时，外部包对该类型的 `match` 即使处理了每个变体，也需要额外的兜底分支（`_ => ...`）。这样库作者以后可以添加新变体，而不破坏 SemVer 兼容性。

```nabi
#[non_exhaustive]
pub struct ClientOptions {
    pub retries: i32 = 3;
    pub verbose: bool = false;
}

let o = ClientOptions { retries: 5 };           // OK in another package
let p = ClientOptions { verbose: true, ..o };   // OK
let ClientOptions { retries, .. } = p;          // `..` required in another package
```

在 `struct` 上时，它标记一个其所属包可能添加字段的记录：

- 外部包对它的结构体模式（`let`、`match`、`if let`）以 `..` 结尾；缺少 `..` 是错误，错误会给出加上 `..` 后的模式。定义它的包可以列出所有字段。
- 结构体首次发布之后添加的每个字段都有默认值（[2.3.3](../002-types/203-struct.md#233-field-default-values)），因此省略它的字面量仍能编译。编译器看不到结构体的历史；这条规则由作者遵守，后续的语义化版本检查会强制执行它。
- 与 Rust 不同，其他包中仍允许结构体字面量和 `..base` 展开（[2.3.4](../002-types/203-struct.md#234-field-spread)）：字面量省略有默认值的字段，写出其余字段。没有合理默认值的字段在每个字面量中都是必需的。

## 6.2.11. 未使用的结果：`unused_result`、`#[discardable]`、`#[must_use]`

```nabi
fn calculate() -> i32 { ... }

#[discardable]
fn insert(self, k: K, v: V) -> V | none { ... }

#[must_use("thread is never joined")]
pub struct JoinHandle<T> { ... }
```

值不是 `()`、`none` 或 `Never` 的表达式语句会触发 lint `unused_result` 警告：`calculate();` 丢弃了一个 `i32`。经过 `try`、`try!` 和 `await` 时，值就是调用的值。`try? f();` 语句改由 `ignored_error` 报告，块状语句（`if`、`match`、块、循环）不检查：它的值是其各分支的值。要有意丢弃一个值，写 `_ = expr;`（[4.2](../004-functions/402-parameters.md)）或 `let _ = expr;`。

函数或方法上的 `#[discardable]` 表示其结果可以丢弃：调用它的语句不会产生警告。它不带实参。用在 trait 方法上时覆盖它的所有 impl。标准库为那些返回被替换的值或附带报告的方法加上了这个标记：`Dict::insert` / `remove`、`Set::insert` / `remove`、`List::pop` / `remove` / `drain`、`Deque::pop_front` / `pop_back`、`BinaryHeap::pop`、`Atomic::swap` / `fetch_*`、`Cell::replace` / `take`、`marker::replace` / `take`、`mem::replace`、`Dir::copy`、`fs::copy`、`io::copy` 和 `io::copy_async`。短写（`Write::write -> usize`）、`checked_*` 以及 `str::trim` 这类纯方法不可丢弃。

`#[must_use]` 可以附着在函数或方法、`struct` 或 `enum` 上，可带一条消息（`#[must_use("..")]`）。用在函数上时它附加这条消息：丢弃其结果时警告的是 `unused_must_use`，而不是 `unused_result`。用在类型上时它更强：值为该类型的表达式语句，或从未被读取的该类型局部变量，都会产生带有属性消息的 `unused_must_use` 警告，即使该值是由 `#[discardable]` 函数返回的。`async fn` 调用或 `async` 块的值（从未被 await 的 future 永远不会运行）以及 `thread::JoinHandle`（[标准库 3.4.2](../../refs-standard-libraries/300-std/304-thread.md#342-api)；从未被 join 的线程可能在程序结束前无法完成；`detach` 表示它独立运行）都是 `#[must_use]`；其余的列在[标准库 0.5.14](../../refs-standard-libraries/000-meta/005-notation.md#0514-docs-and-attributes) 中。`#[must_use]` 类型就是该属性所在的类型：其他包中同名的类型不是。

类型的消息写成接在 “this” 之后的形式：警告读作 `this <message>`。第一个 `": "` 之后的文字成为一行 `help:`，其中 `{}` 代表被丢弃表达式的源代码（或局部变量的名称）：`#[must_use("future is never awaited, so its body never runs: write `await {}`")]` 会给出警告 ``this future is never awaited, so its body never runs``，并附带 ``help: write `await fetch()` ``。没有消息时，警告会指出类型：``this value of type `T` is never used``。

## 6.2.12. `#[track_caller]` 传递调用位置

```nabi
#[track_caller]
fn expect_positive(n: i32) -> i32 {
    if n <= 0 { panic("not positive") }      // location: where expect_positive was called
    n
}
```

- 函数以一个隐藏的末尾 `str` 实参接收调用位置。函数内部的 panic 报告的是该位置（[4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors)）。
- 被跟踪的函数调用另一个被跟踪的函数时，会把自己收到的位置传下去。
- 被跟踪函数的函数值报告的是该值被创建的位置。
- 用在 `async fn` 上时，只跟踪调用本身，而不跟踪协程主体。
- 用在 `extern`（运行时）声明上时，该符号多接收一个位置实参（[8.5.7](../008-abi/805-extern-symbols.md#857-diagnostic-area)）。
- 它不能附着在 `main`、`extern "C" fn` 或 `#[export_name]` 函数上。

在输入无效时触发 panic 的标准库函数（`panic`、`unreachable`、`assert*`、索引、切片、`unwrap`、`List::insert` / `remove` 等）都是 `#[track_caller]`。

## 6.2.13. 小结

上述属性是由编译器和工具直接识别的内置属性；它们不需要用户通过元块来定义。

`#[lang]` 和 `#[literal]` 这样的魔法属性与语言本身的含义绑定，在单独的文档中定义。

### 其他内置属性

编译器还识别以下属性。每个属性都在其功能所在之处规定。

| 属性                           | 位置                          | 含义 |
|--------------------------------|-------------------------------|---------|
| `#[path("file.nabi")]`          | `mod name;`                   | 从该文件而不是 `name.nabi` 加载模块（[5.4.6](../005-modules/504-modules.md#546-custom-paths-path)） |
| `#![no_implicit_nabi]`          | 文件（内部）                  | 关闭该文件的隐式 `use core::prelude::*;`（[标准库 0.2.6](../../refs-standard-libraries/000-meta/002-import.md#026-opting-out---no_implicit_nabi)） |
| `#[export_name("sym")]`        | `extern "C"` / `extern "system"` fn | 以未修饰的符号 `sym` 导出该函数（在 wasm32 上是同名的 wasm 导出）；是不可达函数消除的根（[9.5.8](../009-tools/905-symbol-mangling.md)）。用在普通函数上是错误（[5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-definitions-exposed-externally)） |
| `#[export]`                    | `extern "C"` / `extern "system"` fn | 以函数自身名称作为 `#[export_name]`（[5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-definitions-exposed-externally)） |
| `#[on_panic(return = v)]`、`#[on_panic]` | `extern "C"` / `extern "system"` fn | 函数中的 panic 先被报告，然后函数返回 `v`（整数、浮点数或 `bool` 字面量，空指针写 `0`；没有结果时什么也不返回），而不是中止进程（[5.9.10](../005-modules/509-extern-block.md#on_panic---a-value-for-c-instead-of-an-abort)） |
| `#[errno]`                     | `extern "C"` / `extern "system"` 块或声明 | 每次调用返回后立即保存 errno（Windows 上还有 `GetLastError`），供 `bare::ffi::errno()` 使用（[标准库 1.7.11](../../refs-standard-libraries/100-bare/107-ffi.md#1711-errno---errno-and-getlasterror)）；用在 `extern "nabi"` 块和函数上是错误 |
| `#[leaf]`                      | `extern "C"` / `extern "system"` 块或声明 | 调用时跳过其前后的 park：开销更小，但该函数不得阻塞，也不得回调（[标准库 1.7.1](../../refs-standard-libraries/100-bare/107-ffi.md#leaf---short-calls)）；用在 `extern "nabi"` 块上是错误 |
| `#[no_mangle]`                 | 函数                          | 使该函数免于命名 lint 和未使用函数 lint。它不改变符号；要改变符号请用 `#[export_name]` |
| `#[wit::export]`               | 函数、`pub struct`            | 实现一个 WIT 导出：函数或资源（[标准库 1.8](../../refs-standard-libraries/100-bare/108-wit.md)） |
| `#[niche(..)]`                 | newtype 结构体                | 声明该类型永远不会持有的值，用作 `T \| none` 的 niche（[2.7.4](../002-types/207-niche-optimization.md#274-opt-in-niches-niche)） |
| `#[display("text")]`          | `struct`、`enum`、枚举变体    | 通过写出 `f"text"` 实现 `Display`（[标准库 2.10.14](../../refs-standard-libraries/200-core/210-fmt.md#21014-derive-policy-summary)） |
| `#[intrinsic("nabi.OP.TYPE")]`  | 仅标准库                      | 把函数或方法绑定到编译器操作；在用户代码中是错误（[标准库 0.3](../../refs-standard-libraries/000-meta/003-intrinsics.md)） |
| `#[no_safepoint]`              | `extern "nabi"` 块             | 工具链内部使用：块中的函数既不分配也不阻塞，因此对它们的调用不是安全点；用在其他块上是错误（[5.9.2](../005-modules/509-extern-block.md#592-abi-specifiers)） |
| `#[sealed]`                    | `trait`                       | 只有定义它的包实现该 trait（[6.2.15](#6215-sealed-impls-only-in-the-defining-package)） |
| `#[internal]`                  | 函数、方法、`const`、`static`、字段 | 只在 `bare`、`core` 和 `std` 之间共用（[6.2.16](#6216-internal-shared-by-the-toolchains-packages)） |
| `#[runtime_attribute(name)]`   | `struct`                      | 声明运行时属性 `#[name(..)]`，它保留在 `Inspect` 描述中（[6.10.8](610-reflection.md#6108-runtime-attributes)） |
| `#[inspect(skip)]`             | 字段                          | 让该字段不出现在其类型的 `Inspect` 描述中（[6.10.7](610-reflection.md#6107-typeinfo---the-types-description)） |
| `#[renamed_from("old")]`       | 字段                          | 在描述中记录该字段以前的名称（[6.10.8](610-reflection.md#6108-runtime-attributes)） |
| `#[available(macos = "13.0")]` | 函数、方法、`extern "objc"` 块的方法 | 该 API 所需的 macOS 版本：除非部署目标或外层的 `if objc::available(macos: "13.0")` 覆盖了它，否则调用是错误（[标准库 1.10.13](../../refs-standard-libraries/100-bare/110-objc.md#11013-availability-and-the-deployment-target)） |
| `#[objc::class(..)]`、`#[objc::protocol(..)]`、`#[objc::category(..)]`、`#[selector("..")]`、`#[returns_retained]`、`#[returns_not_retained]` | `extern "objc"` 块及其方法 | Objective-C 类及其方法（[标准库 1.10.2](../../refs-standard-libraries/100-bare/110-objc.md#1102-extern-objc-blocks)） |

## 6.2.14. `#[deprecated_safe]` 现在安全，以后 `unsafe`

```nabi
#[deprecated_safe("`ptr` must point at a NUL-terminated string")]
pub fn from_ptr(ptr: Ptr<u8>) -> CStr { ... }
```

`#[deprecated_safe("note")]` 标记一个将在后续版本中变为 `unsafe` 的库函数、方法、extern 函数或静态变量。在 `unsafe` 块或 `unsafe fn` 之外调用它或使用该静态变量，会触发 lint `deprecated_safe` 警告并显示附注：把调用包在 `unsafe { .. }` 中，这样现在能编译，在该项变为 `unsafe` 后也依然正确。在 `unsafe` 内部不会产生警告。

它供标准库和库包使用。用在正在构建的程序的函数上是错误：应改为把该函数写成 `unsafe fn`。用在函数、方法、extern 函数或静态变量以外的项上是错误。

## 6.2.15. `#[sealed]` 只在定义它的包中实现

```nabi
/// The element types `Atomic<T>` supports.
#[sealed]
pub trait AtomicPrimitive {}
```

trait 上的 `#[sealed]` 表示只有定义该 trait 的包实现它。在其他任何包中对该 trait 的 `impl` 都是错误，错误会指出该 trait 是密封的以及拥有它的包。其他方面与任何 trait 相同：其他包可以在约束中使用它（`T: AtomicPrimitive`）、调用它的方法，并在 `use` 中命名它。

当库依赖于知道实现某个 trait 的所有类型时，就密封该 trait：出于健全性（原子操作只为这些类型存在），或因为其他 impl 会被忽略。标准库密封了 `AtomicPrimitive`（[标准库 2.16](../../refs-standard-libraries/200-core/216-sync.md)）。

工具链的每个层级都是各自独立的包（[5.6.4](../005-modules/506-visibility.md#564-pubpkg-within-the-package)）：`core` 密封的 trait，其 impl 都在 `core` 中。`#[sealed]` 不带实参；用在 trait 以外的任何东西上都是错误。

## 6.2.16. `#[internal]` 由工具链的各个包共用

```nabi
/// Wake every task waiting on `fd`'s readiness.
#[internal]
pub fn wake_io(fd: i32) { .. }
```

`#[internal]` 标记一个 `pub` 项，工具链自身的包 `bare`、`core` 和 `std` 彼此共用它，但它不属于它们的 API。它可以附着在函数、方法、`const`、`static` 或字段上。

- 从其他任何包中使用它（调用、函数值、读取）都是错误：“internal to the Nabi toolchain”。该项仍然能被解析，因此错误会指出它。
- `#[internal("..")]` 指明应改写成什么，错误会显示这一点（`Int::from_literal` 指向 `s.parse::<Int>()`）。没有附注时，错误说明该项仅供编译器使用。
- 它与 `#[doc(hidden)]` 项一样，不出现在 API 文档（`nabi doc`）、语言服务器的补全和悬停列表以及“did you mean”建议中。
- 它不是新的可见性级别（[5.6.1](../005-modules/506-visibility.md#561-four-levels)）：该项是 `pub`，这项检查与可见性是分开的。
- 只有工具链的包可以写它。用在其他任何包的项上是错误；要把项限制在包内，把它设为 `pub(pkg)`。

带有它的项列在[标准库 0.4.4](../../refs-standard-libraries/000-meta/004-lang-slot-table.md) 中。只在一个层级内部使用的项是 `pub(pkg)`，编译器从降级后的代码调用的函数通过 lang 槽访问，因此两者都不需要 `#[internal]`。
