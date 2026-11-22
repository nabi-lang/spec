# 6.4. `#[lang(...)]` 魔法

`#[lang(slot_name)]` 属性把标准库的项连接到编译器直接识别的槽。编译器和运行时正是通过这条通道得知哪个项承载哪种含义。

## 6.4.1. 动机

如[语言与标准库的边界](../000-meta/004-language-vs-standard-libraries.md)所述，语言只提供 `+`、`-` 这样的语法，由标准库决定其含义。要让这个模型成立，编译器必须知道哪个库 trait 或类型连接到哪种语言含义。

```
compiler slot           ← language-known anchor
                          (e.g. "add trait", "option slot", "iterator trait")

library item            ← trait / type / fn defined by standard libraries

  ←── #[lang(slot)] connects the two sides ──→
```

## 6.4.2. 形式

```
lang_attr       ::= "#" "[" "lang" "(" string_lit ")" "]"
```

```nabi
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

#[lang("iterator_trait")]
trait Iterator { ... }

#[lang("display_trait")]
trait Display { ... }              // the default f"{x}" format
```

写作 `#[lang("slot_name")]`，以字符串字面量形式接受槽名作为实参。它可以附着在 trait、类型、函数、`const` 和 `static` 等项上。

## 6.4.3. 槽名的分类

槽名构成编译器预先知道的固定集合；用户不能创建新槽。

代表性的槽：

| 槽                      | 所连接项的种类         | 编译器用途                        |
|-------------------------|------------------------|-----------------------------------|
| `"add_trait"`           | trait                  | `+` 运算符的脱糖                  |
| `"sub_trait"`           | trait                  | `-` 运算符的脱糖                  |
| `"mul_trait"`           | trait                  | `*` 运算符的脱糖                  |
| ...                     | ...                    | ...                               |
| `"none_type"`           | 类型                   | `none` 字面量的类型               |
| `"iterator_trait"`      | trait                  | `for` 循环的脱糖                  |
| `"into_iterator_trait"` | trait                  | `for` 循环的脱糖                  |
| `"future_trait"`        | trait                  | `await` 的脱糖                    |
| `"index_trait"`         | trait                  | `[i]` 索引的脱糖                  |
| `"slice_trait"`         | trait                  | `[1:3]` 切片的脱糖                |
| `"display_trait"`       | trait                  | `f"{x}"` 格式化                   |
| `"debug_trait"`         | trait                  | `f"{x:?}"` 调试格式               |
| `"clone_trait"`         | trait                  | `#[derive(Clone)]`                |
| `"eq_trait"`            | trait                  | `==` / `!=` 运算符（`PartialEq`） |
| ...                     | ...                    | ...                               |

完整的槽列表以及每个槽所附着的项，以[标准库 0.4 `#[lang]` 槽映射表](../../refs-standard-libraries/000-meta/004-lang-slot-table.md)为准；上表只是摘录。对槽集合的修改从 0.4 的表开始。

## 6.4.4. 映射的效果

当某个 trait 带有 `#[lang("add_trait")]` 时，编译器把该 trait 连接到加法的含义。之后的 `+` 运算符按如下方式脱糖：

```
a + b    →    <typeof(a) as the_trait<typeof(b)>>::add(a, b)
```

其中 `the_trait` 是带有 `#[lang("add_trait")]` 的 trait。

## 6.4.5. 在用户代码中使用（罕见）

`#[lang]` 几乎只由标准库使用。只有在 _no_std_ 环境中构建自己的库，或在嵌入式系统、操作系统等必须重新定义语言核心的环境中，用户代码才会直接附着它。普通用户永远不需要直接附着 `#[lang]`。

## 6.4.6. 每个槽一个项

把同一个槽名附着到多个项上是编译错误。

```nabi
#[lang("add_trait")] trait MyAdd { ... }
#[lang("add_trait")] trait OtherAdd { ... }    // ERROR: duplicate slot
```

每个槽恰好映射到一个项。这种唯一性在整个程序范围内（跨所有链接的包）成立。因此，用户包不能重新附着并替换由 `core` 定义的槽（`panic_fn`、`oom_fn`、`unreachable_fn` 等）；这样做是编译错误。

## 6.4.7. 槽的签名检查

每个槽都预先规定了它所接受项的签名。

```nabi
#[lang("add_trait")]
trait Add<Rhs = Self> {              // the form the slot requires
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

签名与槽要求的形式不符时，是编译错误。

```nabi
#[lang("add_trait")]
trait BadAdd { fn run() }            // ERROR: does not match the form of the 'add_trait' slot
```

## 6.4.8. 未知的槽名

```nabi
#[lang("nonexistent_slot")]
trait T { ... }
```

使用未知的槽名同样是编译错误。

## 6.4.9. 槽的稳定性

槽名集合本身是语言规范的一部分，因此添加、修改或删除槽都会改变语言版本（SemVer）。

## 6.4.10. 魔法属性与普通属性的区别

| 方面          | `#[derive]`、`#[inline]` 等             | `#[lang]`                       |
|---------------|-----------------------------------------|---------------------------------|
| 由谁附着      | 用户                                    | 标准库（大多数情况）            |
| 效果          | 代码生成或编译器提示                    | 绑定到语言含义                  |
| 槽或键        | 自由（以属性名识别）                    | 固定的槽集合                    |

`#[lang]` 是元系统中特权最高的属性。其他魔法（`#[literal]`、`#[keyword]` 以及运算符魔法）都可以看作 `#[lang]` 的细化形式。

## 6.4.11. `#[lang]` 与其他魔法的关系

运算符魔法（`#[infix]` 等）通常附着在带有 `#[lang(...)]` 的 trait 的方法上。

```nabi
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

这里 `#[lang("add_trait")]` 声明该 trait 是加法含义所在之处，`#[infix("+", ...)]` 把该方法标记为 `+` 词法单元的分派目标。两种魔法协同工作，在两个方向上把语言与库连接起来。

`#[literal]` 和 `#[keyword]` 同样可以理解为分别连接到字面量位置和关键字注册槽的工具。
