# 6.8. 用户定义的元定义

**元定义**是用户定义的代码生成器。它们都属于同一种机制：定义绑定输入的各个部分，并以卫生的方式（[6.8.9](#689-不操作词法单元流)）、展开到不动点（[6.8.8](#688-不动点展开)）地转写一段由 Nabi 代码和模板记号（[6.9](609-template-tokens.md)）组成的模板。它们的区别只在于如何被**触发**以及绑定什么输入。共有四种形式；前三种由项上的属性触发，第四种由调用触发：

| 形式                                    | 定义方式                           | 触发方式               | 绑定内容                                |
|-----------------------------------------|------------------------------------|------------------------|-----------------------------------------|
| derive                                  | `#[impl_derive(T)] @derive {}`     | 类型上的 `#[derive(T)]` | 类型的形状（`@Self`、`@for_fields`）    |
| 属性                                    | `#[impl_attribute(n)] @attribute {}` | 项上的 `#[n]`        | 该项（`@item`）                         |
| 逐字段 derive 属性                      | `#[impl_derive_attribute(T, a)] @derive {}` | `#[derive(T)]` 加字段上的 `#[a]` | 形状加字段属性    |
| **宏**                                  | `#[impl_macro(n)] @macro { (m) => {t} }` | `n!(..)` 调用      | 调用的词法单元（`@x: expr`、`@rep`）    |

前三种根据作者所附加的**类型或项**生成代码；宏则根据**调用方所写的内容**生成代码，这是其他形式无法满足的需求（[6.8.12](#6812-宏形式)）。它们共用模板引擎、`@` 记号、卫生性以及下文所述的工具保证；本章和 [6.9](609-template-tokens.md) 会指出所绑定输入的不同之处。这四种形式就是 Nabi 元编程的全部表面：它们共同取代了 Rust 的 derive 宏、属性宏和类函数宏，而无需过程式代码或原始的词法单元操作。

```
meta_block      ::= meta_attr visibility? meta_kind block
meta_attr       ::= "#" "[" meta_kind_attr "(" meta_args ")" "]"
meta_kind_attr  ::= "impl_derive" | "impl_attribute" | "impl_derive_attribute" | "impl_macro"
meta_kind       ::= "@derive" | "@attribute" | "@macro"
meta_args       ::= identifier ("," identifier)*

# the @macro block holds rules instead of a flat template:
macro_body      ::= macro_rule+
macro_rule      ::= "(" matcher ")" "=>" block
```

## 6.8.1. 由属性触发的形式

三种由属性触发的形式：

```nabi
#[impl_derive(TraitName)] pub @derive {
    impl TraitName for @Self { ... }
}

#[impl_attribute(name)] pub @attribute {
    @item                                    // reuse the original item
    fn additional_helper() { ... }            // generate an additional item
}

#[impl_derive_attribute(Trait, attr)] pub @derive {
    @for_fields {
        @apply_if(@has_derive_attr(attr)) {
            // process only fields carrying that attr
        }
    }
}
```

各形式的用途：

| 形式                                    | 用途                                       |
|-----------------------------------------|--------------------------------------------|
| `#[impl_derive(TraitName)]`             | 用户定义的 derive                          |
| `#[impl_attribute(name)]`               | 用户定义的属性（项包装器）                 |
| `#[impl_derive_attribute(Trait, attr)]` | 逐字段 derive 属性                         |

第四种，即由调用触发的 `macro`，在 [6.8.12](#6812-宏形式) 中定义。

## 6.8.2. `#[impl_derive(TraitName)]` 用户定义的 derive

用户定义 derive 的示例：

```nabi
#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self {
        fn my_hash(self) -> u64 {
            let h: u64 = 0u64;
            @for_fields {
                h = combine_hash(h, self.@field.my_hash());
            }
            h
        }
    }
}

// use site
#[derive(MyHash)]
struct Point { x: f64; y: f64; }
// → generated automatically.
//   impl MyHash for Point {
//       fn my_hash(self) -> u64 {
//           let h: u64 = 0u64;
//           h = combine_hash(h, self.x.my_hash());
//           h = combine_hash(h, self.y.my_hash());
//           h
//       }
//   }
```

这种形式由 `#[impl_derive(name)]` 和主体（模板块）组成。当使用处出现 `#[derive(name)]` 时，模板会针对该类型展开以生成代码。

## 6.8.3. `#[impl_attribute(name)]` 用户定义的属性

用户定义属性的示例：

```nabi
#[impl_attribute(timed)] pub @attribute {
    @accept(fn)                                  // attachable only to fn items

    fn @item.name(@item.params) -> @item.return_type {
        let start = Instant::now();
        let result = {
            @item.body
        };
        log(f"@item.name took {Instant::now() - start}");
        result
    }
}

// use site
#[timed]
fn process(x: i32) -> i32 {
    expensive(x)
}
// → generated automatically.
//   fn process(x: i32) -> i32 {
//       let start = Instant::now();
//       let result = { expensive(x) };
//       log(f"process took {Instant::now() - start}");
//       result
//   }
```

这是对项的包装：它复用原始项（`@item`）的内容和签名来生成变换后的形式。它不仅可以附加在函数上，也可以附加在 `struct`、`enum` 和 `impl` 上；目标限制用 `@accept` 表达。

## 6.8.4. `#[impl_derive_attribute(Trait, attr)]` 逐字段 derive 属性

逐字段 derive 属性的示例：

```nabi
#[impl_derive_attribute(Serialize, rename)] pub @derive {
    impl Serialize for @Self {
        fn serialize(self) -> Json {
            let obj = JsonObj::new();
            @for_fields {
                @apply_if(@has_derive_attr(rename)) {
                    obj.set(@derive_attr(rename), self.@field.serialize());
                }
                @apply_if(!@has_derive_attr(rename)) {
                    obj.set(@field.name.as_str(), self.@field.serialize());
                }
            }
            obj.into()
        }
    }
}

// use site
#[derive(Serialize)]
struct User {
    #[rename("user_name")]
    name: str;
    age: i32;
}
// → generated automatically.
//   impl Serialize for User {
//       fn serialize(self) -> Json {
//           let obj = JsonObj::new();
//           obj.set("user_name", self.name.serialize());
//           obj.set("age", self.age.serialize());
//           obj.into()
//       }
//   }
```

这种形式接受两个参数：被 derive 的 Trait，以及标记逐字段信息的属性。它在 derive 主体中读取字段的属性信息并据此分支。

该属性可以用 `attr(k, ..)` 的形式声明它接受的键：`#[impl_derive_attribute(Nabion, nabion(deny_unknown_fields))]`。此时，类型、其变体和其字段上的每个 `#[attr(..)]` 都只能列出已声明的键；其他任何键都是错误，报错会列出有效的键。`@Self.has_derive_key(k)` 从类型上读取某个键（[6.9.6](609-template-tokens.md#696-derive_attribute-记号)）。

## 6.8.5. 主体（模板块）

在每种形式中，主体都是同时包含普通 Nabi 代码和模板记号的模板块（由属性触发的形式使用 `@Self`、`@field`、`@for_fields`；宏使用 `@x`、`@rep`、`@for`、`@concat`，[6.8.12](#6812-宏形式)）。

模板记号的完整列表定义在[模板记号与展开](609-template-tokens.md)中。

## 6.8.6. `pub` / 可见性

可见性的写法如下：

```nabi
pub @derive { ... }
pub(pkg) @attribute { ... }
@derive { ... }                  // private
```

可见性关键字附加在模板块本身上。其效果：

| 可见性                        | 可使用的范围                                               |
|-------------------------------|------------------------------------------------------------|
| `pub`（包括 `pub(pkg)`）      | 整个构建。其他包要带上本包的名字来写：`#[derive(dep::X)]` / `#[dep::x]` / `dep::x!(..)`（6.8.11） |
| 私有                          | 定义该元块的文件内                                         |

元名称没有模块路径：一个名称属于注册它的元块所在的包，在该包中直接书写，在其他包中带上该包的名字书写（6.8.11）。元块的模板所能看到的，就是其所在模块能看到的：模板中写出的名字在那里解析，也在那里进行访问检查（[6.9.9](609-template-tokens.md#展开中的名字)）。元块或关键字处理函数的 `#[cfg]` 按定义它的包的 cfg 求值（使用该包自己的特性，[9.10.4c](../009-tools/910-nabi-frontend.md#9104c-特性)）。

## 6.8.7. 主体引导符（`@derive` / `@attribute`）

主体引导符表明模板的种类。

| 引导符       | 用途                                                      |
|--------------|-----------------------------------------------------------|
| `@derive`    | derive 定义（`impl_derive`、`impl_derive_attribute`）     |
| `@attribute` | 属性定义（`impl_attribute`）                              |
| `@macro`     | 宏定义（`impl_macro`）；其块中保存的是规则（[6.8.12](#6812-宏形式)） |

引导符决定了哪些模板记号在主体中有意义。例如，`@item` 只用于属性，而匹配器记号（`@x: expr`、`@rep`）只用于 `@macro`。

## 6.8.8. 不动点展开

如果模板展开的结果产生了带有其他 derive 或属性的项，该项会被递归展开。

```nabi
#[impl_attribute(adds_helper)] pub @attribute {
    @item                                       // the original item as is

    #[derive(MyHelper)]                          // new item: derive processed again
    struct Helper { ... }
}
```

应用这个属性时，主体中的 `#[derive(MyHelper)]` 会在下一轮展开中处理。

当展开到达不再剩下任何待展开的 derive 或属性的不动点时，展开结束。

循环展开（A 调用 B，B 又调用 A）会被编译器检测到，属于错误（ERROR）。报错信息会给出循环路径（`A → B → A`）。

## 6.8.9. 不操作词法单元流

每种元定义（三种由属性触发的形式以及宏）都只用模板记号生成代码。Nabi 没有类似 Rust 的 `syn` 或 `quote` 那样的任意词法单元变换设施，也没有在编译期运行代码的过程式形式。

这一限制保证元定义的输出是编译器能够理解的 Nabi 代码，从而同时确保工具友好性和可调试性。

## 6.8.10. 与过程宏的比较

Rust 过程宏与 Nabi 元块的比较：

| 方面            | Rust 过程宏                    | Nabi 元块                                       |
|-----------------|--------------------------------|-------------------------------------------------|
| 输入            | 任意的词法单元流               | 项的结构信息（字段、变体等）                    |
| 输出            | 任意的词法单元流               | Nabi 代码和模板记号                             |
| 卫生性          | 取决于过程宏本身               | 自动（卫生的）                                  |
| 调试            | 检查输出的词法单元             | 检查展开后的 AST                                |
| 表达力          | 非常高                         | 有限                                            |

Nabi 的选择是用表达力换取可预测性和易调试性。

## 6.8.11. 与注册和可见性的配合

带有 `#[impl_derive(TraitName)]` 的元块注册名为 `TraitName` 的 derive，`#[impl_attribute(name)]` 注册属性 `#[name]`，`#[impl_macro(name)]` 注册宏 `name!`。编译器在解析之前从库和用户包的每个文件中收集元块，因此无论定义位置或文件顺序如何，都可以使用它们。如果元块上的 `#[cfg]` 为假，该块不会被注册（[6.9.9](609-template-tokens.md#与-cfg-的关系)）。

元名称属于注册它的元块所在的包。在该包中，它直接书写（`#[derive(MyHash)]`、`#[x]`、`x!(..)`），或者带上 `pkg::`。其他包要在前面写上定义包的名字，与该包路径的根相同：`#[derive(dep::MyHash)]`、`#[dep::x]`、`dep::x!(..)`。限定符只能是包本身；元块不是名字空间中的项，没有模块路径。

模块级的 `use dep::x!;` 导入宏 `x!`，于是该文件可以用 `x!(..)` 调用它。它不导入任何项，也不能重命名（`as`）或重新导出（`pub use`）。对与某个 derive 同名的项（通常是其 trait）的 `use`（`use dep::MyHash;`）也会导入该 derive，而 `use dep::*;` 会导入 `dep` 的所有元名称。

```nabi
// package json
#[impl_macro(log)] pub @macro { ... }
#[impl_derive(MyHash)] pub @derive {
    impl pkg::MyHash for @Self { ... }
}

// a package that depends on json
use json::log!;

#[derive(json::MyHash)]
struct Foo { ... }

fn main() {
    json::log!("start");
    log!("imported");
}
```

不带限定的名字依次解析为本包的块、`use` 导入的块、标准库的块（标准库的元名称保持不带限定：`#[derive(Inspect)]`）。否则就是错误，报错会列出带限定的形式：`` `log!` is package `json`'s macro, which a bare name does not reach ``，帮助信息为 ``write `json::log!`, or add `use json::log!;` ``。如果两个包的 `use` 都导入了这个名字，它就是有歧义的，同样是错误。在展开内部，模板所写的元名称从定义它的包和文件出发遵循同一规则，因此依赖包的宏无论在哪里展开都能正常工作；由使用处传入的名字则从使用处出发遵循该规则（[6.9.9](609-template-tokens.md#展开中的名字)）。

以下情况是错误：

- 同一个包的两个元块注册了同一个 derive、属性或宏名称（一个宏就是一个包含其全部规则的 `@macro` 块）。两个不同的包可以各自注册同一个名称。
- 带限定的宏、属性或 derive（`dep::x!`、`#[dep::x]`、`#[derive(dep::X)]`）在所指名的包中并未注册。
- 元块重新定义了内置 derive 的名称（[6.2.1](602-built-in-attributes.md)：`Clone`、`Copy`、`Default`、`Debug`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash`、`Error`）或内置属性的名称（`cfg`、`derive`、`test`、`inline`、`repr`、`keyword`、`bench`、`path`，以及 6.2 至 6.7 中的其他属性）。帮助信息会建议一个新名字（`#[impl_attribute(my_bench)]`）。

## 6.8.12. 宏形式

**宏**是由调用触发的元定义。derive 或属性根据其所附加的类型或项生成代码，而宏则像 `name!(..)` 这样被**调用**，并在该位置展开为代码，根据调用方所写的内容生成代码。这是由属性触发的形式无法满足的需求：由调用而非类型的字段驱动地重复或生成代码。

### 定义

与其他形式一样，宏由 `#[impl_macro(name)]`、可见性、`@macro` 引导符和一个块组成。块中保存的不是平铺的模板，而是一条或多条**规则**：每条规则是括号中的匹配器、`=>`（与 [`match`](../003-expressions/312-match-expression.md) 分支中的箭头相同）以及花括号中的模板。

```nabi
#[impl_macro(name)] pub @macro {
    ( <matcher> ) => { <template> }
    ( <matcher> ) => { <template> }
    ..
}
```

`#[impl_macro(name)]` 注册宏 `name!`，与 `#[impl_derive(T)]` 注册 derive `T` 完全相同。调用时，规则自上而下依次尝试，展开第一条匹配器匹配成功的规则；如果没有规则匹配，就在调用处产生编译错误。与 derive 或属性一样，宏的名字属于其所在的包：在该包中写 `name!`，在其他包中写 `dep::name!` 或使用 `use dep::name!;`（[6.8.11](#6811-与注册和可见性的配合)）。每次调用末尾的 `!` 表明它是宏而不是函数。

```nabi
#[impl_macro(twice)] pub @macro {
    ( @e: expr ) => { { let v = @e; (v, v) } }
}

let p = twice!(compute());      // (the value, the same value)
```

### 调用

宏的调用形式为 `name!( .. )`、`name![ .. ]` 或 `name!{ .. }`，从其他包调用则为 `dep::name!( .. )`；这些定界符可以互换，不改变含义。实参是任意的**词法单元树**：`()` `[]` `{}` 配对平衡的任意词法单元，因此即使不了解该宏，词法分析器和格式化工具也能对调用进行词法切分。宏可以在以下位置展开，具体取决于其模板产生的内容：

| 位置       | 示例                                      | 模板产生                |
|------------|-------------------------------------------|-------------------------|
| 表达式     | `let xs = grid![1, 2; 3, 4];`             | 一个表达式              |
| 语句       | `debug!("here");`                         | 零条或多条语句          |
| 项         | `int_impl!(i8, i16, i32);`（模块级）      | 零个或多个项            |
| 类型       | `let f: fn_ptr!(i32, i32) = ..;`          | 一个类型                |
| 模式       | `match x { pair!(a, b) => .. }`           | 一个模式                |

在表达式位置，调用是一个初等表达式：其展开结果是一个操作数，如同加了括号，后缀形式可以跟在它后面（`grid![1, 2; 3, 4].len()`、`table!{ .. }.head(5)`、`first!(xs)?`）。

### 匹配器与片段

匹配器是一个词法单元模式。字面词法单元必须原样出现在调用中；**元变量** `@name: <fragment>` 捕获调用的一部分供模板使用。片段决定捕获多少词法单元：`ident` 捕获一个标识符，`literal` 捕获一个字面量词法单元，`tt` 捕获一棵词法单元树（一个词法单元或一个配对平衡的组）。`expr`、`stmt`、`block`、`ty`、`pat`、`path` 和 `item` 捕获一段非空的词法单元序列，直到括号深度为 0 处匹配器的下一个字面词法单元（或者直到调用的末尾或外层组的末尾）。匹配器中的括号组 `( .. )`、`[ .. ]` 或 `{ .. }` 匹配调用中一个配对平衡的组，其内部作为一个独立的匹配器进行匹配：`[ @rep[sep = ","] { @x: expr } ]` 匹配 `[5, 6]`。每个片段都作为原始词法单元捕获，匹配时不解析；整个展开结果在转写之后才解析，因此如果某个实参放在模板所放的位置上并不合法，就是展开结果中的语法错误，在调用处报告。

```nabi
#[impl_macro(clamp)] pub @macro {
    ( @x: expr , @lo: expr , @hi: expr ) => {
        { let v = @x; if v < @lo { @lo } else if v > @hi { @hi } else { v } }
    }
}
```

### 重复

`@rep` 匹配一个子模式零次或多次。`@for` 对每个元素转写一次；它与 `@for_fields`（[6.9.3](609-template-tokens.md#693-字段迭代)）是同一个迭代器，这里作用于捕获到的序列。

```nabi
#[impl_macro(list)] pub @macro {
    ( @rep[sep = ","] { @x: expr } ) => {
        { let acc = List::new(); @for @x { acc.push(@x); } acc }
    }
}

let xs = list![a, b, c];        // acc.push(a); acc.push(b); acc.push(c);
```

`@rep[sep = ","] { <sub-pattern> }` 匹配以 `sep` 词法单元分隔的重复子模式（`sep` 可选；`trailing` 还允许一个末尾分隔符）。在 `@rep` 内部捕获的元变量绑定的是一个序列。`@for @x { <body> }` 对每个元素输出一次主体；来自同一个 `@rep` 的多个元变量同步迭代，长度必须相等。`@rep` 和 `@for` 可以嵌套：位于两层 `@rep` 内部的元变量绑定的是序列的序列，内层的 `@for @x` 遍历外层当前元素（`(@rep[sep = ";"] { @rep[sep = ","] { @x: expr } }) => { [@for @x { [@for @x { @x, }], }] }` 把 `1, 2; 3` 变为 `[[1, 2], [3]]`）。形状记号（`@Self`、`@field`、`@apply_if` 等）属于由类型驱动的形式，不属于宏：宏没有可供检测的形状，因此使用它们是错误。

### 名字合成

`@concat(..)` 把名字词法单元和字符串字面量连接成一个标识符，因此宏可以生成各不相同的具名项：

```nabi
#[impl_macro(accessors)] pub @macro {
    ( @s: ident ; @rep[sep = ","] { @f: ident : @t: ty } ) => {
        impl @s { @for @f { fn @concat(get_, @f)(self) -> @t { self.@f } } }
    }
}
// accessors!{ P; x: i32, y: i32 } -> impl P { fn get_x(self) -> i32 { self.x } .. }  (and get_y)
```

由 `@concat` 生成的名字会被**记录**：编译器以生成的名字为每个合成的项建立索引，因此符号搜索、跳转到定义和查找引用都能找到它。`@concat` 只在定义位置或路径位置构成标识符；它的组成部分是名字词法单元和字符串字面量，不能拼装任意语法。

### 库中的样板代码

使用宏的经典理由，是在作者写出的固定列表上重复同一个定义，这是任何由属性触发的形式都做不到的。它取代了 [`core::num`](../../refs-standard-libraries/200-core/README.md) 中手工复制的逐类型 impl：

```nabi
#[impl_macro(nonzero_div)] pub @macro {
    ( @rep[sep = ","] { @t: ty } ) => {
        @for @t {
            impl Div<NonZero<@t>> for @t {
                type Output = @t;
                const fn div(self, rhs: NonZero<@t>) -> @t { self / rhs.0 }
            }
        }
    }
}

nonzero_div!(i8, i16, i32, i64, u8, u16, u32, u64);
```

### 递归

模板可以调用其他宏，这些调用会依次展开。宏不得调用自身：在宏自身的展开结果中直接或经由其他宏调用该宏，会立即以编译错误拒绝，报错会指出该调用（`` `name!` expands too deeply or cyclically ``）。不同宏的嵌套展开限制为 256 层，超出时报同一错误。要在列表上重复，应使用 `@rep` / `@for` 而不是递归。

### 内置 `!` 形式是保留的

`cfg!`（[6.3](603-cfg.md)）和 `include_str!` / `include_bytes!`（[6.11](611-include.md)）以 `!` 结尾，但它们是固定的编译器内置形式，而不是宏，不能被重新定义、遮蔽或用作宏名。它们是编译器提供的仅有的 `!` 形式；其他所有 `name!` 都是用 Nabi 定义的宏。

### 工具与卫生性

宏保留每种元定义的保证（[6.8.9](#689-不操作词法单元流)），并加上调用处带来的特定要求：

- **卫生性**是自动的：模板所绑定的名字（`let` 或 `for` 循环的变量）是全新的，不会捕获调用处的名字，也不会被调用处捕获；通过元变量传入的名字在调用处解析。模板所写的其他名字，例如 `impl Tr for E` 头部中的类型或导入的 `Complex`，指向宏定义处的项。
- **源码区间（span）**：每个展开出的词法单元都带有调用的区间，因此诊断会指向调用处，并以注释（note）形式显示展开结果。
- **解析**：展开结果作为普通代码解析，合成的名字会被建立索引，因此悬停提示、跳转到定义和查找引用都能找到宏生成的项。
- **按需展开**：语言服务器可以就地展开调用（[9.9.13](../009-tools/909-debug-tools.md)）。
- **格式化**：当调用的实参是 Nabi 片段时，格式化工具会格式化它们；原始的 `tt` DSL 则在其配对平衡的定界符内保持原样。

## 6.8.13. 小结

这四种形式（derive、属性、逐字段 derive 属性以及由调用触发的宏）基于同一个模板引擎，构成了 Nabi 元编程的全部表面。它们缺少过程宏的表达力，但覆盖了核心用例：derive、属性、逐字段属性以及调用处的代码生成，每一种都是卫生的，并且可以被工具追踪。

[模板记号与展开](609-template-tokens.md)定义了模板记号的确切列表（包括由类型驱动和由调用驱动的输入）以及展开机制。
