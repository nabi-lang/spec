# 2.6. 和类型（`A | B`）

和类型是一种带标签联合，其值是若干已有类型之一。它与 `enum` 相似，但其变体没有单独的名称；已有类型直接用作变体。

## 2.6.1. 形式

和类型的示例：

```nabi
i32 | f32
T | none
A | B | C
i32 | str | bool
```

各项用[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)中的 `|` 词法单元分隔。项可以是任何类型表达式：简单名称、路径、泛型或另一个和类型。

## 2.6.2. 含义

和类型的值属于其中某一项，属于哪一项由标签跟踪。

```nabi
let x: i32 | f32 = 42i32;       // tag is at the i32 position
let y: i32 | f32 = 3.14f32;     // tag is at the f32 position
```

## 2.6.3. 内存表示

默认表示把标签和载荷内联放置。

```
+------+----------------------+
| tag  | payload (size of     |
|      |  largest variant)    |
+------+----------------------+
```

标签标识哪一项处于活动状态。载荷的大小取所有项中最大的那个，对齐取所有项中最大的对齐。

这种表示适用 [niche 优化](207-niche-optimization.md)，因此标签槽可能消失。

### 值递归与大小限制

- 按值包含自身的递归类型（即使经由结构体、枚举、元组、和类型、数组或泛型实例）在声明处是错误。递归请使用 GC 引用类型（`List<T>` 等）。
- Nabi 的实现限制（由实现定义，[0.4.4](../000-meta/004-language-vs-standard-libraries.md#044-implementation-limits)）：超过 65536 个展平标量槽或 4 GiB 的值类型是错误。

确切的放置方式由实现决定，但有一点得到保证：零填充的 `T | none` 存储（`GcBuf<T | none>` 和集合类型缓冲区的元素、新分配的 GC 对象的字段）读出为 `none`。

### 确切位宽

标签的位宽由编译器决定，取决于项的数量，可以是 1、2、4、8 位等。用户代码无法直接访问标签位。

## 2.6.4. 集合语义

> [!IMPORTANT]
> 和类型具有集合运算的语义。

### 规范化

和类型始终以规范形式处理。

- 嵌套的和类型被展平。`(A | B) | C`、`A | (B | C)` 和 `A | B | C` 是同一类型。
- 重复项被去除。`A | A | B` 是 `A | B`。
- 项的顺序无关紧要。`A | B` 和 `B | A` 是同一类型。
- 如果只剩一项，就是该类型本身。`A | A` 是 `A`。

展开类型别名的结果会再次规范化（在 `type N = i32 | f64;` 下，`N | i32` 是 `f64 | i32`）。泛型参数的代入遵循下文的“泛型和类型的分层”。

因此，形状相同的和类型是同一类型，用户不必在意写的是 `i32 | f32` 还是 `f32 | i32`。

### 泛型和类型的分层

带有泛型项的和类型 `T | X1 | ... | Xn`（例如 `T | none`、`T | ParseErr`）在语义上具有**自己的一层**。`X1 ... Xn` 是在该和类型中写出的非泛型项。泛型项有以下四种：

- 类型参数（`T`）
- `Self`
- 关联类型（`Self::X`、`T::X`、`<T as Tr>::X`）
- 泛型类型别名的参数

分层规则在泛型项被实际类型替换时适用。如果代入的类型与一同写出的项重叠，就形成嵌套的层；否则就被展平。下文中 `T` 代表一个泛型项。

**何时保留一层**。如果代入 `T` 的类型与 `X1 ... Xn` 有任何共同的项，代入后的 `T` 保持为单个项，成为**嵌套的层**。如果没有重叠，就被展平。展平不会丢失区分值的信息，因此结果表现为、并被当作扁平的和类型。

| 泛型和类型           | `T` 的代入           | 结果                                                |
|----------------------|----------------------|-----------------------------------------------------|
| `T \| none`          | `i32 \| str`         | `i32 \| str \| none`（无重叠，展平）                 |
| `T \| none`          | `i32 \| none`        | `(i32 \| none)?`（两层）                             |
| `T \| ParseErr`      | `i32 \| ParseErr`    | `(i32 \| ParseErr) \| ParseErr`                      |
| `T \| str \| none`   | `i32 \| str`         | `(i32 \| str) \| str \| none`（`str` 重叠）          |

- 重叠指任何项，不只是 `none`。
- 只由两个参数组成的和类型 `T | U`，两者都被代入，因此即使重叠也被展平，而不是嵌套。如果必须区分这两个值，使用 `enum`。
- 每一层分别规范化，因此 `A | B == B | A` 在每一层都成立。
- 在源代码中用具体类型写出的和类型总是被展平。写 `(i32 | none) | none` 得到 `i32 | none`。
- 非泛型的类型别名是写出的类型，因此总是被展平。在 `type Opt = i32 | none; type Flat = Opt | none;` 下，`Flat` 是 `i32 | none`。
- 泛型别名的参数是泛型项，遵循上述规则（见下文“书写嵌套类型”）。

**书写嵌套类型**。可选类型用后缀 `?` 书写（[2.1.5a](201-type-expressions.md#215a-optional-types-t)）。`T?` 等同于 `Maybe<T>`，因此其操作数是泛型项，适用同样的分层规则。其他嵌套（`(E | i32) | E` 等）用泛型别名或 `Self::X` 书写。

```nabi
type Maybe<T> = T | none;
let r: (i32 | none)? = xs.get(0);       // two layers
let q: Maybe<i32 | none> = xs.get(0);   // same type
let p: i32? = 1;                         // i32 | none (no overlap)
let f: (i32 | str)? = none;              // i32 | str | none
```

**存入值**。把值存入嵌套的和类型时，属于 `T` 某一项类型的值进入内层。属于外层某项类型的值（`none` 字面量、`ParseErr` 值等）是外层的值。要把外层项类型的值放入内层，先把它放进类型为 `T` 的槽，再存入该槽。

**转换**。嵌套类型不会隐式转换为扁平类型；这是类型错误。

```nabi
let v: i32 | none = xs.get(0);    // ERROR: xs: List<i32 | none>
// mismatched types: `let v` expects `i32 | none`, found `(i32 | none)?`
```

**运算**。模式、`is`、`?.`、`??` 和 `?` 作用于**最外层**。外层的另一侧绑定到内层的 `T`。在覆盖了外层所有项的分支之后，绑定模式绑定内层。

**诊断中的写法**。诊断用括号显示嵌套的层。如果外层只有 `none`，写作 `(i32 | none)?`；否则写作 `(E | i32) | E`、`(i32 | none) | str | none`。`(i32 | none)?` 也是含义相同的合法源代码。

```nabi
struct ParseErr { msg: str; }

fn check<T>(x: T) -> T | ParseErr {
    if bad(x) { return ParseErr { msg: "bad" }; }   // ParseErr of the outer layer
    x                                                // a value of T: inner layer
}

let a = check(1i32);                  // T = i32, no overlap: ParseErr | i32
let v: i32 | ParseErr = ParseErr { msg: "earlier" };
let r = check(v);                     // T = ParseErr | i32, overlap: (ParseErr | i32) | ParseErr
match r {
    e: ParseErr => print(f"check failed: {e.msg}"),        // outer layer: error made by check
    inner => match inner {                                // inner: ParseErr | i32
        e: ParseErr => print(f"original value was an error: {e.msg}"), // inner layer: error that was in v
        n: i32 => print(n),
    },
}
```

**关联类型**。在 trait 及其所有实现内部，裸名称 `X` 和 `Self::X` 都表示该关联类型（[2.8.3](208-trait.md#283-associated-types)）。关联类型是泛型项，因此适用同样的规则。当 `Item` 含有 `none` 时，`fn next(ref self) -> Self::Item?` 会嵌套，因此产出 `i32 | none` 元素的用户迭代器也会把存储的 `none` 当作元素进行迭代。

关于 `none` 的情形（嵌套可选类型），见 2.6.5 中的示例。

### 规范顺序

规范形式的各项具有确定的顺序。

- `none` 总是排在最后。
- 其余各项按结构键排序：基本类型按其名称，结构体或枚举按其完整路径，泛型实例按 `path<arg keys>`。嵌套的和类型会被展平（见上文“规范化”），因此和类型永远不会是和类型的项；泛型实参中的和类型以其按规范顺序排列的各项为键（`List<A | B>` 的键是 `List<key(A)|key(B)>`）。由“泛型和类型的分层”保留下来的一层是单个项，其键是 `(` + 按规范顺序以 `|` 连接的各项键 + `)`。顺序与书写顺序无关。
- 顺序以及由此确定的标签从不依赖于某次构建的内部类型编号，因此预构建的库与其使用者保持一致（[9.4.8](../009-tools/904-nabihir-bytecode.md#948-rules-for-separate-compilation)）。

规范顺序固定的是表示（标签），而不是值的顺序。只有在除 `none` 外至多一项的和类型中，值才有顺序（[标准库 2.5.7](../../refs-standard-libraries/200-core/205-cmp.md#257-implementation-matrix)）：

- 当 `T` 具有 `PartialOrd` / `Ord` 时，`T | none`（以及内层为单项的嵌套 `(T | none)?`）也具有。与 Rust 和 Haskell 一样，`none` 小于任何 `T` 值；两个 `T` 值按 `T` 的顺序比较。`min` / `max` / `clamp` 和排序都遵循这一顺序。
- 除 `none` 外有两项或更多的和类型（`i32 | str`、`i32 | str | none`）没有 `PartialOrd` / `Ord`。对这种值使用 `<`、`>`、`<=`、`>=`、`cmp`、`min` / `max` / `clamp`，以及对其集合使用 `sort` 或 `max`，都是编译错误。可以用 `sort_by_key` / `sort_by` / `min_by_key` 按键排序这类值，或对各项进行 `match`。

```nabi
let xs: List<i32 | none> = [3, none, 1];
xs.sort();                       // [none, 1, 3]
let ys: List<i32 | str> = [1, "a"];
ys.sort();                       // ERROR: `i32 | str` has no order
ys.sort_by_key(|v| match v { n: i32 => (0, n, ""), s: str => (1, 0, s) });
```

### 子集

- 和类型的值可以原样放入包含其所有项的更大的和类型。`i32 | none` 的值可以放在 `i32 | str | none` 的位置。单个项的值（`i32`）同样如此。
- 类型模式 `v: A | B` 可以匹配更大和类型中各项的一个子集。在该分支中，`v` 是 `A | B`。

```nabi
let v: i32 | str | f64 | none = ...;
match v {
    n: i32 | f64 => use_num(n),     // n: f64 | i32
    s: str => use_str(s),
    none => {},
}
```

### 泛型推断

泛型推断把和类型当作集合处理。用 `T | none` 匹配 `i32 | str | none` 会收集剩余的项：`T = i32 | str`。

```nabi
fn unwrap_or<T>(x: T | none, d: T) -> T { ... }
let v: i32 | str | none = ...;
unwrap_or(v, 0i32);       // T = i32 | str
```

## 2.6.5. 与 `none` 组合

[布尔与哨兵字面量](../001-lexical/109-other-literals.md)中的 `none` 最常作为和类型的一项出现。

```nabi
T | none
```

这种形式表示值可能存在也可能不存在，对应其他语言中的 `Option<T>`、`T?` 或 `Maybe T`。

与 niche 优化结合时（例如对类似指针的 T），可以用 null 表示 `none`，而没有内存开销。确切规则在 [niche 优化](207-niche-optimization.md)中定义。

### 嵌套可选类型

这是把 2.6.4 中“泛型和类型的分层”规则应用于 `none`。如果泛型 `T | none` 中的 `T` 已经包含 `none`，外层的 `none` 层会保留下来，成为**嵌套可选类型**（类似 Swift 的 `T??`）。`Maybe<i32 | none>`，即 `T = i32 | none` 时的 `Maybe<T> = T | none`，也就是 `(i32 | none)?`，有两层。`none` 模式、`is none`、`?.`、`??` 和 `?` 作用于外层，非 none 的一侧绑定到内层的 `T`（`i32 | none`）。

因此，`List<i32 | none>` 会把存储的 `none` 当作元素进行迭代，而 `get` / `pop` / `Dict` 查找能区分“不存在”与“存储的是 none”。

```nabi
let xs: List<i32 | none> = [1, none, 3];

for x in xs { print(f"{x:?}"); }  // 1, none, 3 (stored none is also an element)

match xs.get(1) {                // get: (i32 | none) | none (two layers)
    none => print("out of range"),     // outer none: no element
    v => match v {               // v: i32 | none (stored value)
        none => print("stored none"),
        n: i32 => print(n),
    },
}

let d: Dict<str, i32 | none> = {"a": none};
d.get("a") ?? 0i32;              // none: key present, value is none (?? strips only the outer layer)
d.get("b") ?? 0i32;              // 0: key absent
```

## 2.6.6. 与 `any` 的区别

[`any` 槽](202-any-slot.md)中的 `any` 是开放的槽：任何类型都可能出现，变体可以自由增加。

相比之下，和类型（`A | B`）是封闭的槽：只可能是列出的变体。作为回报，编译器知道所有可能的项，可以对模式匹配做穷尽检查。

| 方面            | `any`                                  | `A \| B`                                      |
|-----------------|----------------------------------------|-----------------------------------------------|
| 可能的类型      | 所有类型                               | 只有列出的项                                  |
| 分派            | 运行时标签检查                         | 运行时标签检查（项更少，分支更少）            |
| 穷尽性          | 不可能                                 | 可能                                          |
| 内存            | 标签 + 载荷（通用）                    | 标签 + 最大的项（更紧凑）                     |
| 渐进性          | 渐进（新类型无需修改代码即可）         | 静态（增加变体会改变类型）                    |

## 2.6.7. 与 `enum` 的区别

在 `enum` 中，具名变体在容器内定义，变体是 `enum` 的一部分。

和类型直接把已有类型用作变体，因此没有单独的变体名称。

```nabi
enum Result { Ok(i32), Err(str) }       // variant names Ok / Err are defined inside the enum
type Either = i32 | str;                  // no variant names: just i32 or str
```

当变体各有不同含义、需要名称时，选择 `enum`。当原样组合已有类型最为自然时，选择和类型。

## 2.6.8. 类型收窄

和类型的值可以通过模式匹配或 `is` 检查收窄到某一项，方式与 [`any` 槽](202-any-slot.md)相同。

```nabi
let v: i32 | str = ...;
match v {
    n: i32 => ...,           // inside this arm v is i32
    s: str => ...,           // inside this arm v is str
}
```

确切的收窄语法在模式与表达式规范中定义。

## 2.6.9. 与类型别名组合

较长的和类型可以用 `type` 起一个别名（`type Num = i32 | f64;`）。循环的类型别名（即使经由泛型实参）是错误。类型别名的 `where` 写在被别名的类型之后（`type A<T> = B<T> where T: X;`）。省略的实参取其参数的默认值，因此所有参数都有默认值的别名可以裸写：在 `type Vel<V = f64> = Q<V, 1, -1>;` 下，`Vel` 就是 `Vel<f64>`。诊断用代码为泛型类型所写的别名来称呼它（`expects Vel, found Len`，而不是 `Q<f64, 1, -1>`）；正在构建的包所写的别名优先于依赖所写的别名。别名的 `where` 在每次使用该别名时检查。当使用处的实参提及类型参数时，由使用处作用域内的约束决定，与泛型主体中一样（[2.10.9](210-generics.md#2109-body-checking)）：外围函数、`impl` 或 trait 的参数约束与 `where` 子句，以及 trait 的父 trait 和关联类型约束。在 `type W<T> = S<T> where T: Num;` 下，`fn get<T: Num>(x: W<T>)` 和 `impl<T: Num> W<T> { .. }` 有效；`fn get<T>(x: W<T>)` 即使从未被调用也是错误。声明中的类型同样如此：`struct`、`enum` 或 `union` 的字段，trait `impl` 的 trait 实参与关联类型，以及 trait 方法的签名，在不满足别名的 `where` 时都会被拒绝，即使没有任何地方使用它们；帮助信息会指出应在哪个泛型中添加约束（``add `T: Num` to `F`'s generics``）。`any Tr` 的别名（`type Shown = any Display;`）也是这样的别名：它代表该动态槽。`type X = some Tr;` 不是别名，而是具名的不透明类型，是由其模块的函数固定的一个静态类型（[4.1.15](../004-functions/401-function-declaration.md#4115-named-opaque-types-type-x--some-trait)）。`type` 是上下文关键字，只在项的开头且后跟标识符和 `=` 或 `<` 时才是关键字（[1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)）。确切的别名语法在类型别名规范中定义。

```nabi
type Number = i32 | i64 | f32 | f64;
type Maybe<T> = T | none;
```
