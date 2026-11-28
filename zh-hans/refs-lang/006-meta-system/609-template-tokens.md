# 6.9. 模板记号与展开

本节定义可在[元块](608-meta-blocks.md)主体中使用的模板记号的完整列表，以及元块在何时、以何种机制展开。

## 6.9.1. 模板记号的形式

模板记号有以下几种语法形式。

```
template_token  ::= "@" identifier
                  | "@" identifier "." identifier
                  | "@" identifier "{" <template body> "}"
                  | "@" identifier "(" <args> ")" "{" <body> "}"
```

每个模板记号都以 `@` 词法单元开头（[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)）。典型示例：

```
@Self                   target type name
@field                  placeholder for the current field
@field.name             name of the current field
@for_fields { ... }     field iteration block
```

### 记号列表

| 记号                                                     | 值                                                     | 节      |
|----------------------------------------------------------|--------------------------------------------------------|---------|
| `@Self`                                                  | 目标类型（`Name<T, ..>`）                              | 6.9.2 |
| `@Self.name`                                             | 不带类型实参的目标类型名                               | 6.9.2 |
| `@Self.is_unit` / `.is_tuple` / `.is_named`              | 结构体的形状（布尔值）                                 | 6.9.2 |
| `@Self.field_count`、`@Self.variant_count`               | 结构体 / 联合体的字段数、`enum` 的变体数（整数字面量） | 6.9.2 |
| `@if_struct` / `@if_enum` / `@if_union { .. }`           | 仅当目标种类匹配时才生成的主体                         | 6.9.2 |
| `@for_fields { .. }`                                     | 对每个字段生成的主体                                   | 6.9.3 |
| `@field`、`@field.name`、`@field.type`                   | 当前字段（访问 / 名字 / 类型）                         | 6.9.3 |
| `@field.is_pub`、`@field.has_attr(x)`                    | 布尔值                                                 | 6.9.3 |
| `@field.has_default`、`@field.default`                   | 字段是否带有 `= expr`（布尔值）/ 该表达式              | 6.9.3 |
| `@field.index`、`@field.binding`                         | 字段的位置（整数字面量）/ 表示其值的局部名字           | 6.9.3 |
| `@Self.pattern`                                          | 由 `@field.binding` 局部变量构造的结构体               | 6.9.3 |
| `@for_variants { .. }`                                   | 对每个 `enum` 变体生成的主体                           | 6.9.4 |
| `@variant.name`、`@variant.pattern`                      | 当前变体（名字 / 匹配模式）                            | 6.9.4 |
| `@variant.index`                                         | 当前变体的位置（整数字面量）                           | 6.9.4 |
| `@variant.is_unit` / `.is_tuple` / `.is_named`           | 布尔值                                                 | 6.9.4 |
| `@for_variant_fields { .. }`                             | 对当前变体的每个字段生成的主体                         | 6.9.4 |
| `@apply_if(cond) { .. }`                                 | 仅当 `cond` 为真时才生成的主体                         | 6.9.5 |
| `@accept(kind, ..)`                                      | 限制目标种类（仅用于属性）                             | 6.9.5 |
| `@where_all(Trait)`                                      | 为每个类型参数生成 `T: Trait`                          | 6.9.5 |
| `@has_derive_attr(a)`、`@derive_attr(a)`                 | 字段的 derive 属性（布尔值 / 参数）                    | 6.9.6 |
| `@has_derive_attr(a, k)`、`@derive_attr(a, k)`           | 字段 `#[a(..)]` 的键 `k`（布尔值 / 值）                | 6.9.6 |
| `@Self.has_derive_key(k)`                                | 类型的 `#[attr(..)]` 中是否列出了键 `k`（布尔值）      | 6.9.6 |
| `@item`、`@item.name` / `.params` / `.return_type` / `.body` | 所附加的项及其各部分（仅用于属性）                 | 6.9.7 |
| `@attribute`                                             | 使用处的属性参数（仅用于属性）                         | 6.9.7 |
| `<name token>.as_str()`                                  | 该名字的字符串字面量                                   | 6.9.3 |
| `@use_site(path)`                                        | 在使用元块的位置查找的 `path`                          | 6.9.9 |
| `@concat(a, @Self.name, ..)`                             | 由名字、索引记号和字符串连接而成的一个标识符           | 6.9.8 |

不在列表中的 `@name` 是错误。`@where_for_fields` 是保留名，使用它也是错误。

## 6.9.2. 类型上下文

### `@Self` 目标类型名

`@Self` 表示元块所应用的目标类型的名字。

```nabi
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {                 // @Self → Point / the user's type name
        ...
    }
}
```

### `@if_struct { ... }` / `@if_enum { ... }` / `@if_union { ... }`

这些记号分别仅在目标类型是 `struct`、`enum` 或 `union` 时才生成所包含的代码。

```nabi
@if_struct {
    fn struct_only_method() { ... }
}

@if_enum {
    fn enum_only_method() { ... }
}
```

如果目标类型属于其他种类，该块被忽略，不生成代码。

### `@Self.name` / `@Self.is_unit` / `.is_tuple` / `.is_named`

`@Self.name` 是不带类型实参的目标类型名（`Pair<T>` 对应 `Pair`）。`@Self.is_unit` / `@Self.is_tuple` / `@Self.is_named` 是布尔值，分别表示目标结构体是否为单元结构体、元组结构体或带具名字段的结构体；它们用于 `@apply_if`。

`@Self.field_count` 是目标 `struct` 或 `union` 的字段数，`@Self.variant_count` 是目标 `enum` 的变体数，二者都是整数字面量（`[u8; @Self.field_count]`、`fn count() -> usize { @Self.variant_count }`）。在 `enum` 上使用 `@Self.field_count` 是错误（变体的字段数用 `0 @for_variant_fields { + 1 }` 计算），在不是 `enum` 的类型上使用 `@Self.variant_count` 也是错误（用 `@if_enum { .. }` 加以保护）。

## 6.9.3. 字段迭代

### `@for_fields { ... }`

`@for_fields` 对 `struct` 或 `union` 的每个字段重复该块。在 `@for_variants` 内部，它遍历当前变体的字段，与 `@for_variant_fields` 相同。在 `enum` 上不经 `@for_variants` 而使用它是错误。

```nabi
@for_fields {
    println(self.@field);
}
```

这段代码对每个字段生成一次。

### `@field`

`@field` 是当前字段的占位符，用于字段访问位置。

```nabi
self.@field                          // self.x, self.y, ...
@Self { @field: 0i32 }                // field initializer position
```

### `@field.name` 标识符记号

在需要标识符的位置，字段名作为标识符词法单元插入。

```nabi
let v = @field.name;                  // the name goes into the identifier position
fn @field.name() -> i32 { ... }       // can also be used as a function name
```

如果需要字符串，就在后面加上 `.as_str()`。`@field.name.as_str()`、`@variant.name.as_str()`、`@item.name.as_str()` 和 `@Self.as_str()` 在展开时变为该名字的字符串字面量（`"x"`）。`@Self.as_str()` 是不带类型实参的名字。写在字符串或 f-string 内部的 `@item.name` 等名字记号会按字面替换。

### `@field.type`

`@field.type` 表示字段的类型。

```nabi
fn typed<T>() where T == @field.type { ... }
```

### `@field.is_pub`

如果字段是 `pub`，该记号为真，否则为假。

```nabi
@apply_if(@field.is_pub) {
    // process only pub fields
}
```

### `@field.has_attr(X)`

如果字段上附加了 `#[X]`，则为真。

```nabi
@apply_if(@field.has_attr(skip)) {
    // process only fields with #[skip]
}
```

### `@field.has_default` / `@field.default`

如果字段有默认值（`x: i32 = 0`，[2.3.3](../002-types/203-struct.md#233-字段默认值)），`@field.has_default` 为真，`@field.default` 是该 `= expr` 中的表达式。在没有默认值的字段上使用 `@field.default` 是错误。

```nabi
@apply_if(@field.has_default) {
    let @field.name = @field.default;
}
```

## 6.9.4. 变体迭代

### `@for_variants { ... }`

`@for_variants` 对 `enum` 的每个变体重复该块。

```nabi
@for_variants {
    @variant.name => /* ... */
}
```

### `@variant.name` 变体名

`@variant.name` 是变体的名字。

### `@variant.pattern` 匹配模式

`@variant.pattern` 是匹配模式形式的变体，例如 `Shape::Circle(r)`、`Shape::Pending` 或 `Shape::Click { x: x1, y: y1 }`。`r`、`x1`、`y1` 位置上的绑定是编译器生成的名字（卫生性，[6.9.9](#卫生性)），其值在 `@for_variant_fields` 内部用 `@field` 取得。模板不能直接使用这些绑定名。

```nabi
match self {
    @for_variants {
        @variant.pattern => /* ... */
    }
}
```

`@variant.pattern` 同时也是一个表达式：当每个 `@field`（在 `@for_variant_fields` 中）都由 `let` 绑定时，它构造出该变体。

### `@variant.index`、`@field.index`、`@field.binding`、`@Self.pattern`

`@variant.index` 和 `@field.index` 是当前变体和字段按声明顺序的位置，为整数字面量：`c.@field.index.push(self.@field)` 访问某个字段所对应的元组列。在 `@for_fields` / `@for_variant_fields` 之外使用 `@field` 记号是错误，报错会提示把它放进 `@for_fields { .. }` 中。`@field.binding` 是表示当前字段值的局部名字：在结构体上，`@Self.pattern` 用这些局部变量构造结构体（`Name { a: <a's binding>, .. }`、`Name(..)` 或 `Name`）；在变体中，它就是 `@variant.pattern` 所绑定的名字。

```nabi
@for_fields {
    let @field.binding: @field.type = read_field(@field.name.as_str());
}
let made: @Self = @Self.pattern;
```

### `@variant.is_unit` / `@variant.is_tuple` / `@variant.is_named`

这些记号检测变体的种类。

```nabi
@apply_if(@variant.is_tuple) {
    // process only tuple variants
}
```

### `@for_variant_fields { ... }`

要遍历变体内的字段，使用 `@for_variant_fields`。在其内部可以使用 `@field.name` 等记号。

```nabi
@for_variants {
    @variant.pattern => {
        @for_variant_fields {
            process(@field);
        }
    }
}
```

## 6.9.5. 条件与约束

### `@apply_if(cond) { ... }`

`@apply_if` 仅在 `cond` 为真时才生成所包含的代码。

```nabi
@apply_if(@field.is_pub) {
    // code
}

@apply_if(!@field.has_attr(skip)) {
    // only fields *without* skip
}

@apply_if(@field.is_pub && @field.has_attr(serialize)) {
    // both conditions true
}
```

`cond` 是把模板记号的布尔值（例如 `@field.is_pub`、`@field.has_attr(...)`）与普通布尔运算符（`!`、`&&`、`||`）组合而成的表达式，在编译期求值。

`else` 分支不需要单独的关键字，用条件相反的 `@apply_if` 表达。

### `@accept(...)`

`@accept` 限制属性的目标种类；在 derive 中使用它是错误。它接受一个用逗号或 `|` 分隔的项种类列表。

```nabi
#[impl_attribute(my_wrap)] pub @attribute {
    @accept(struct, enum)                    // accepts only struct, enum; rejects union
    @item
}

#[impl_attribute(timed)] pub @attribute {
    @accept(fn)                              // accepts only fn
    fn @item.name(...) { ... }
}
```

可用的种类：

| 种类     | 含义                    |
|----------|-------------------------|
| `struct` | `struct` 项             |
| `enum`   | `enum` 项               |
| `union`  | `union` 项              |
| `fn`     | 函数或方法              |
| `impl`   | `impl` 块               |
| `trait`  | `trait` 项              |

用逗号或 `|` 分隔多个种类时，匹配其中任意一个即被允许。如果目标不是允许的种类，就在使用处产生编译错误。

### `@where_all(Trait)`

`@where_all` 以自动 where 子句的形式，添加“每个字段类型都必须满足 `Trait`”这一条件。

```nabi
#[impl_derive(Same)] pub @derive {
    @where_all(Same)                         // OK if every field's type is Same
    impl Same for @Self {
        fn same(self, other: Self) -> bool {
            @for_fields {
                if !self.@field.same(other.@field) { return false; }
            }
            true
        }
    }
}
```

这里的 `Same` 是用户 trait（`Eq` 这样的内置 derive 不能被重新定义，[6.8.11](608-meta-blocks.md#6811-与注册和可见性的配合)）。于是，生成的 `impl` 会为目标类型的每个类型参数加上 `T: Same`。对于具体的字段类型不添加子句；这些字段是否实现了 `Same`，由对生成主体的类型检查来检验。目标类型的泛型参数放在模板的 `impl` 之后，唯一的约束就是 `@where_all` 添加的那些。

## 6.9.6. derive_attribute 记号

以下记号可以在 `#[impl_derive_attribute(Trait, attr)]` 内部额外使用。

### `@has_derive_attr(name)`

表示 `derive_attribute` 的目标 attr 是否附加在该字段上。

### `@derive_attr(name)` attr 的参数

`@derive_attr(name)` 表示所附加 attr 的参数。

### `@has_derive_attr(name, k)` / `@derive_attr(name, k)`：单个键

字段的 `#[name(..)]` 的参数可以是带字面量值的键（`k = "v"`）和不带值的键（`k`）。第二个参数给出键时，`@has_derive_attr(name, k)` 表示字段的属性是否列出了 `k`，`@derive_attr(name, k)` 是写在 `k =` 之后的值，作为来自使用处的记号（不带值的 `k` 为 `true`）。属性声明了它接受的键时（`attr(k, ..)`，[6.8.4](608-meta-blocks.md#684-impl_derive_attributetrait-attr-逐字段-derive-属性)），`k` 必须是其中之一，否则在模板处报错。对属性中没有 `k` 的字段使用 `@derive_attr(name, k)` 是错误，报错会提示用 `@apply_if(@has_derive_attr(name, k))` 包起来。

```nabi
#[impl_derive_attribute(Args, arg(short, help))] pub @derive {
    impl Args for @Self {
        fn specs() -> List<Spec> {
            [@for_fields {
                Spec {
                    name: @field.name.as_str(),
                    @apply_if(@has_derive_attr(arg, short)) { short: @derive_attr(arg, short), }
                    @apply_if(@has_derive_attr(arg, help)) { help: @derive_attr(arg, help), }
                },
            }]
        }
    }
}

// use site
#[derive(Args)]
struct Opts {
    #[arg(short = "v", help = "print more")]
    verbose: bool;
    #[arg(help = "files to read")]
    files: List<str>;
}
// → `Spec { name: "verbose", short: "v", help: "print more" }`,
//   `Spec { name: "files", help: "files to read" }`
```

### `@Self.has_derive_key(k)`

表示类型自身的 `#[attr(..)]` 中是否列出了键 `k`。`k` 必须用 `#[impl_derive_attribute(Trait, attr(k, ..))]` 声明（[6.8.4](608-meta-blocks.md#684-impl_derive_attributetrait-attr-逐字段-derive-属性)）；未声明的键是错误。`#[derive(Nabion)]` 就是这样读取 `#[nabion(deny_unknown_fields)]` 的。

```nabi
#[impl_derive_attribute(Serialize, rename)] pub @derive {
    @for_fields {
        @apply_if(@has_derive_attr(rename)) {
            let key = @derive_attr(rename);  // the argument of #[rename("...")]
            obj.set(key, ...);
        }
    }
}
```

## 6.9.7. 仅用于属性的记号

以下记号在 `#[impl_attribute(name)]` 内部使用。

### `@item` 复用原始项

`@item` 原样复用属性所附加的整个项。

```nabi
#[impl_attribute(deprecated_warning)] pub @attribute {
    @item                                    // the original as is
}
// the attribute above is effectively a no-op; the original passes through unchanged
```

### `@item.body`

`@item.body` 是函数体，只对函数项有意义。

### `@item.name`、`@item.params`、`@item.return_type`

这些记号表示函数签名的各个部分。

```nabi
#[impl_attribute(timed)] pub @attribute {
    fn @item.name(@item.params) -> @item.return_type {
        let start = Instant::now();
        let r = { @item.body };
        log(f"@item.name took {Instant::now() - start}");
        r
    }
}
```

### `@attribute` `#[...]` 参数

`@attribute` 表示附加时写出的参数。如果使用处以 `#[name(args)]` 的形式应用由 `#[impl_attribute(name)]` 定义的属性，`@attribute` 就是这些 args。

```nabi
#[impl_attribute(log_prefix)] pub @attribute {
    fn @item.name() {
        let p = @attribute;                  // the use site's argument
        log(f"{p}: {@item.name}");
        @item.body
    }
}

// use site
#[log_prefix("INFO")]
fn run() { ... }
// → generated.
//   fn run() {
//       let p = "INFO";
//       log(f"{p}: run");
//       ...
//   }
```

## 6.9.8. 调用处（宏）记号

上述记号绑定的是定义所附加的类型或项。宏（[6.8.12](608-meta-blocks.md#6812-宏形式)）则通过匹配器绑定调用方的词法单元，并使用以下记号；`@for` 也只用于宏：在 `@attribute` 或 `@derive` 块中使用它是错误（它们用 `@for_fields` / `@for_variants` 重复）。`@concat` 在三种形式中都可以使用。反过来，形状记号（`@apply_if` 及其所检测的 `@Self` / `@field` / `@variant` 记号）在宏中是错误：宏没有可供检测的类型形状。

| 记号                                   | 在匹配器 / 模板中                                              | 节      |
|----------------------------------------|----------------------------------------------------------------|---------|
| `@name: <fragment>`                    | 匹配器：捕获一个片段（`expr`、`stmt`、`block`、`ty`、`pat`、`path`、`ident`、`literal`、`item`、`tt`） | 6.9.8 |
| `@name`                                | 模板：捕获到的片段，或 `@for` 中的当前元素                     | 6.9.8 |
| `@rep[sep = ",", trailing] { .. }`     | 匹配器：重复的子模式，带可选的分隔符和可选的末尾分隔符         | 6.9.8 |
| `@for @name { .. }`                    | 模板：对捕获序列的每个元素输出一次主体（多个序列同步迭代）     | 6.9.3 |
| `@concat(a, @name, ..)`                | 模板：把名字词法单元 / 字符串字面量连接成一个会被编入索引的标识符（也可用于 `@derive` / `@attribute`） | 6.9.8 |

### `@name: <fragment>` 片段捕获

在匹配器中，`@name: <fragment>` 把调用的一部分作为原始词法单元捕获；匹配时不做任何解析，因此选择的是第一条字面词法单元和词法单元数量都吻合的规则（[6.8.12](608-meta-blocks.md#匹配器与片段)）。`ident`、`literal` 和 `tt` 分别捕获一个标识符、一个字面量词法单元和一棵词法单元树（一个词法单元或一个配对平衡的组）；其他片段捕获直到匹配器下一个字面词法单元为止的一段词法单元。展开结果在转写之后作为普通代码解析，因此捕获的代码在那里进行类型检查和其他检查，工具也会解析其中的名字；如果某个实参在模板放置它的位置上不合适，就是在调用处报告的语法错误。`tt` 是用于非 Nabi DSL 的片段。匹配器中其他位置的字面词法单元必须原样出现。在模板中，`@name` 替换为捕获的内容，并保留调用处的卫生性（[6.9.9](#699-展开机制)）。

### `@rep` / `@for` 重复

`@rep[sep = ",", trailing] { <sub-pattern> }` 匹配子模式零次或多次；在其内部捕获的元变量绑定的是一个序列。`@for @name { <body> }` 与 `@for_fields`（[6.9.3](#693-字段迭代)）是同一个迭代器，这里作用于该序列；来自同一个 `@rep` 的元变量同步迭代，长度必须相等。`@rep` 和 `@for` 可以嵌套。

### `@concat(..)` 名字合成

`@concat(..)` 把名字词法单元（`@name`、不带修饰的标识符）和字符串字面量连接成单个标识符，只能用于定义位置或路径位置。它构成的标识符会被记录，因此符号搜索和跳转到定义都能找到生成的项（[6.8.12](608-meta-blocks.md#工具与卫生性)）；非名字的组成部分，或在名字位置之外使用，都是错误。

在 `@derive` 或 `@attribute` 块中，它的组成部分还可以是目标的名字记号和索引记号（`@Self.name`、`@field.name`、`@variant.name`、`@field.index`、`@variant.index`、`@item.name`），因此 derive 可以声明一个具名的伴随项。结果不以字母或 `_` 开头时是错误。

```nabi
#[impl_derive(Soa)] pub @derive {
    pub struct @concat(@Self.name, Columns) {
        @for_fields { pub @field.name: List<@field.type>; }
    }
}

#[derive(Soa)]
struct Point { x: f32; y: f32; }    // also declares `PointColumns { x: List<f32>; y: List<f32>; }`
```

## 6.9.9. 展开机制

### 收集与展开

元展开按以下步骤进行。

1. 解析之前的预处理从库和用户包的每个文件中收集元块（[6.8.11](608-meta-blocks.md#6811-与注册和可见性的配合)）。
2. 解析器遇到带有 `#[derive(X)]` 或 `#[x]` 的项时，用该项的值替换对应元块主体中的模板记号，并再次解析结果。
3. 生成的项会再次进行元处理，如此重复，直到到达不动点。
4. 循环（`A → B → A`）是错误（[6.8.8](608-meta-blocks.md#688-不动点展开)）。

### 与 `cfg` 的关系

元块自身上的 `#[cfg(..)]` 决定该块是否被**注册**。cfg 求值先于元块收集，因此 cfg 为假的块的 derive / 属性不会被注册。于是，两个块可以在互斥的 cfg 下定义同一个名字。包含该块的模块（内联 `mod` 和文件模块）上的 `#[cfg]` 也作用于注册。文件内部的 `#![cfg]` 此时尚不生效。

项上的 [`#[cfg]`](603-cfg.md) 会原样复制到由它生成的每个项上。因此，cfg 为假的项会连同其生成的输出一起消失；整体顺序如下：

```
parse → cfg eval → meta expansion (fixed-point) → name resolution → type checking → ...
```

### 卫生性

模板所绑定的局部名字（`let` / `for` 绑定以及闭包参数，包括 f-string 的 `{..}` 内部）在展开时会被重命名为不会冲突的名字。因此，模板的局部变量既不会隐藏也不会捕获来自该项的名字（`@item.body`、`@item.params`、`let @field.name = ..`）。

```nabi
#[impl_derive(Foo)] pub @derive {
    impl Foo for @Self {
        fn foo(self) -> i32 {
            let temp = compute();             // renamed at expansion
            temp * 2
        }
    }
}

// use site
#[derive(Foo)]
struct S { temp: str; }                       // the use site's 'temp' field

let s = S { temp: "hi" };
let r = s.foo();                              // no collision with the meta-generated 'temp'
```

由 `match` 分支模式绑定的名字以及 `S { x }` 简写形式不会被重命名。模板不在这些位置使用可能与该项冲突的名字。`@variant.pattern` 的绑定是编译器生成的名字，不会冲突。

### 展开中的名字

展开结果中的每个词法单元，要么来自**定义**（模板，或编译器写出的代码），要么来自**使用**（`@Self`、`@field`、`@variant`、`@item`、`@attribute`、`@derive_attr` 以及宏捕获所替换的内容）。路径归属于其第一个路径段所在的一方；其泛型实参各自保持自己的归属。

来自定义的路径，其解析和访问检查如同它写在包含该元块的模块中：首先是展开结果自身声明的名字，然后是该模块的作用域（其项和 `use`、`pkg::`、其所在包的依赖、prelude 中的名字）。它可以指名该模块的私有项及其所在包的 `pub(pkg)` 项。来自定义的方法名，按那里作用域中的 trait 查找（[3.5](../003-expressions/305-member-access-and-method-call.md)）。来自使用的路径在使用处解析和检查，因此传入元定义的代码不会获得额外的访问权限。模板中的 `pkg::` 指定义该元块的包，无论在哪里展开。

编译器写出的代码（内置 derive、`#[literal]` 处理函数调用）以它所指名的项所在的模块为视角：处理函数如同从它自己的模块中被调用，因此它可以具有任意可见性。

生成的项放在使用处的模块中，它们声明的名字在那里可见。模板的局部变量会被重命名（见上文“卫生性”）。

`@use_site(path)` 让模板中的路径在使用处解析，用于期望由用户定义的名字。其参数必须是路径。

```nabi
// package mylib
pub(pkg) fn combine(h: u64, x: u64) -> u64 { .. }

#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self {
        fn my_hash(self) -> u64 {
            let h: u64 = 0u64;
            @for_fields { h = combine(h, self.@field.my_hash()); }   // mylib's, though pub(pkg)
            @use_site(check)(self);                                    // the user's `check`
            h
        }
    }
}
```

来自定义、而定义模块中并不存在的名字，是模板处的错误，使用处作为附注（note）附上，帮助信息会提到 `@use_site(..)`。构建一个包时，其 `pub` 元块中的每条带根路径（`pkg::..`、`std::..`、依赖的路径）都必须能从该块所在的模块解析，无论是否被使用，因此这类错误归于作者。

### 诊断位置

展开过程中的错误、展开结果的解析错误以及生成代码中的类型错误指向模板的位置，并把使用处（`#[derive(X)]` / `#[x]`）作为附注（note）附上。来自某个字段的错误（字段的类型、属性中写的值）也指向该字段（``for field `x` ``）。经由模板记号来自该项的词法单元，指向该项的位置。

## 6.9.10. 调试

`nabic build --emit=expansions`（[9.9.10](../009-tools/909-debug-tools.md#9910-元块调试)）把用户包的展开结果作为源文本写入 `target/<profile>/build/<name>.expanded.nabi`（或 `-o` 指定的路径）。每个展开结果都是一条 ``// expanded from this `#[derive(X)]` (file:line:col)`` 注释，后跟生成的源代码；如果输出再次被展开，它会跟在外层展开之后。展开中的宏调用会在原处显示为展开后的样子。

## 6.9.11. 小结

模板记号是 Nabi 元系统的代码生成词汇。它们虽然简单，却覆盖了实用 derive 和属性的所有核心情形。

Nabi 的选择是限制词法单元变换的灵活性，以换取可调试性、工具友好性和有保证的卫生性。

第 6 章（元系统）到此结束。
