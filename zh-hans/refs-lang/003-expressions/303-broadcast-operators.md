# 3.3. 广播运算符

[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)中定义的点前缀运算符（`.+`、`.*`、`.==` 等）是用于逐元素运算（广播）的独立词法单元。它们与普通运算符经由同一机制工作（脱糖为 trait 方法调用），但映射到另一组 trait。

## 3.3.1. 脱糖模式

点前缀运算符按如下方式脱糖。

```
a .+ b      →   <typeof(a) as DotAdd<typeof(b)>>::dot_add(a, b)
a .* b      →   <typeof(a) as DotMul<typeof(b)>>::dot_mul(a, b)
a .== b     →   <typeof(a) as DotEq<typeof(b)>>::dot_eq(a, b)
```

这与[运算符分派](301-operator-dispatch.md)中普通映射的形式相同，只有目标 trait 不同。点前缀运算符映射到另一个名称以 **Dot** 开头的 trait 族，如下所示。

| 点前缀词法单元 | 不带点的形式      | 点前缀 trait       |
|--------------------|-------------------|--------------------|
| `.+`         | `+` ↔ `Add`       | `DotAdd`        |
| `.-`         | `-` ↔ `Sub`       | `DotSub`        |
| `.*`         | `*` ↔ `Mul`       | `DotMul`        |
| `.**`        | `**` ↔ `Pow`      | `DotPow`        |
| `./`         | `/` ↔ `Div`       | `DotDiv`        |
| `.%`         | `%` ↔ `Rem`       | `DotRem`        |
| `.%%`        | `%%` ↔ `Modulo`   | `DotModulo`     |
| `.==`        | `==` ↔ `PartialEq::eq` | `DotEq`    |
| `.!=`        | `!=` ↔ `PartialEq::ne` | `DotNe`    |
| `.<`         | `<` ↔ `PartialOrd::lt` | `DotLt`    |
| `.>`         | `>` ↔ `PartialOrd::gt` | `DotGt`    |
| `.<=`        | `<=` ↔ `PartialOrd::le` | `DotLe`   |
| `.>=`        | `>=` ↔ `PartialOrd::ge` | `DotGe`   |
| `.&&`        | `&&`（短路）      | `DotLogicalAnd` |
| `.\|\|`        | `\|\|`（短路）      | `DotLogicalOr`  |
| `.&`         | `&`               | `DotBitAnd`     |
| `.\|`         | `\|`               | `DotBitOr`      |
| `.^`         | `^`               | `DotBitXor`     |
| `.<<`        | `<<`              | `DotShl`        |
| `.>>`        | `>>`              | `DotShr`        |

这些 trait 的确切签名以及哪些类型实现它们，由标准库定义。本文档只负责语言把点前缀词法单元脱糖到一个独立 trait 族这一事实。

> [!IMPORTANT]
> **点前缀逻辑运算符不短路**：与不带点的形式不同，`.&&` 和 `.||` **不**短路。由于广播被降级为融合循环，两个操作数总是都会求值；在语义上，它是对 bool 容器的逐元素逻辑组合。放在任一操作数中的带副作用的函数调用总会执行。

## 3.3.2. 优先级与结合性

点前缀运算符与其不带点的对应运算符具有相同的优先级和结合性。

```
a .+ b .* c     →   a .+ (b .* c)              // .* binds tighter than .+ (same as * > +)
a .+ b .+ c     →   (a .+ b) .+ c              // left-associative
```

## 3.3.3. 函数与方法广播

在函数或方法后附加 `.`，即把它应用到每个元素。

```nabi
sin.(x)            // apply sin to each element of x
f64::sqrt.(arr)    // method form also allowed
g.(a, b)           // multiple arguments
```

这种形式返回一个新集合，其中是对每个元素做普通函数或方法调用的结果：一个 `List`，或者当每个容器实参都是定长数组时为定长数组。它脱糖为对标准库 `Iterator::map`（或等价 trait）的调用，并收集其结果。

```
f.(x)        →   x.iter().map(f)                                  // collected
g.(a, b)     →   a.iter().zip(b.iter()).map(|(x, y)| g(x, y))     // collected
```

函数与方法广播没有自己的 trait；它是迭代器变换之上的语法糖。这是它与二元点前缀运算符的不同之处。

## 3.3.4. 自动循环融合

由点前缀运算符和函数广播组成的表达式树被融合为一个循环。

```
sin.(a) .+ cos.(b) .* 2.0f64
```

当容器操作数全部是 core 的 `List`、`Slice` 或定长数组，或者全部是 `Broadcastable`（[标准库 2.6.6](../../refs-standard-libraries/200-core/206-broadcast.md#266-循环融合)），且点运算符的其他每个操作数都是基本类型、元素类型或（与自定义 `Broadcastable` 类型并列时）`BroadcastScalar` 时，保证融合：结果在一趟中构建，没有中间集合。每个操作数在循环之前按源代码顺序求值一次。

- 对 core 集合，长度在第一个元素之前检查（定长数组在编译期检查）。结果是 `List`，或者当每个容器操作数都是数组时为数组。
- 对 `Broadcastable` 操作数（`Array` 或自定义类型），每个操作数被视为一个数组，这些视图在第一个元素之前按广播规则（[标准库 2.6.8](../../refs-standard-libraries/200-core/206-broadcast.md#268-numpy-风格的广播规则)）拉伸为同一形状。结果通过 `Materialize` 构建：操作数为 `Array` 时用主导操作数（秩最高者中的第一个）的 `Materialize`，为自定义类型时用点 trait 所给出类型的 `Materialize`，因此融合循环的类型与逐个应用运算符所得的类型相同（标准库 2.6.6）。
- 混合这两类的树是错误，其帮助信息给出转换列表的写法（`Array::from_list(xs, [xs.len()])`）。

其他任何树，即含有既非容器、也非基本类型或元素类型的操作数的树，不会融合：每个运算符调用自己的点 trait（3.3.1）并产生自己的结果。

融合循环是普通循环，与其他循环一样编译（[0.6.4](../000-meta/006-platform-targets.md#064-simd)）。

## 3.3.5. 与普通运算符的语义差别

> [!IMPORTANT]
> 点前缀运算符之所以是独立的 trait，是因为同样的数据可以承载数学上不同的含义。

```
m * n           // matrix product (Mul impl, user-defined)
m .* n          // element-wise product (Hadamard product, DotMul)
```

为避免 NumPy 的 `*` 隐式按元素运算所带来的混淆，Nabi 始终在视觉上把两种含义分开。

这种分离直接应用了[语言与标准库的边界](../000-meta/004-language-vs-standard-libraries.md)中的原则“语法属于语言，含义属于库”：两种不同的含义映射到两种不同的语法。

## 3.3.6. 用户定义的 trait 实现

要让一个类型支持点前缀运算，按与普通运算符相同的方式实现 **Dot** trait 族。

```nabi
impl ops::DotAdd for MyVector {
    type Output = MyVector;
    fn dot_add(self, rhs: MyVector) -> MyVector { ... }
}
```

点前缀运算通过这一相同模式得到支持，无需额外的 trait 或宏。实现了 `Broadcastable` 的类型则获得融合表达式和原地广播（3.3.4、3.3.7）；此时它的点 trait impl 用于满足 trait 约束和按名调用。

## 3.3.7. 原地广播

`target .= value` 把 `value`（一个点表达式、容器或标量）的元素逐个写入 `target`；对 `+ - * / % %% ** & | ^ << >>` 中的 `op`，`target .op= value` 即 `target .= target .op value`，且 `target` 只求值一次。它们与 `=` 一样是语句。

```nabi
y .= a .* x .+ b;          // one loop, no array allocated
y .+= 1.0;                 // y .= y .+ 1.0
m[:, 0] .= 0.0;            // a view is a target: the column of m
xs .*= ys;                 // a List, lengths equal
```

- `.=` 先按源代码顺序求值各操作数，再求值目标；`.op=` 先求值目标（它是第一个操作数），再求值其他操作数（[3.4.7](304-assignment.md#347-求值顺序)）。
- `List` 或定长数组目标接受与其长度相同的操作数（在第一个元素之前检查，定长数组在编译期检查）；每个元素通过 `IndexSet` 存储。`Slice` 是只读视图，不能作为目标。
- `Broadcastable` 目标保持其形状：操作数拉伸到该形状（标准库 2.6.8），秩更高的操作数是编译错误；结果形状不同时触发 panic（`broadcast: the result has shape [2, 3], which does not fit the destination of shape [1, 3]; the destination is not stretched`）。元素通过目标的视图写入，因此数组的切片（`a[1:]`）会写入该数组。
- 与目标在其他位置共享缓冲区的操作数（`v .= v[::-1]`）从循环前制作的副本中读取；同一视图（`v .+= v`）或与之分离的缓冲区则就地读取。结果如同所有元素在存储任何元素之前都已计算完毕。
- 比较以及 `.&&`、`.||`、`.??` 通过 `.=` 进行：`ok .= ok .&& xs .> 0`。
- 其他任何目标都是错误，其帮助信息会把运算完整写出（`x = x .+ ..;`）。

## 3.3.8. 广播合并 `.??`

`a .?? b` 是逐元素的 `??`：`a` 中每个 `none` 元素变为 `b` 的对应元素，`b` 为标量时则变为 `b` 本身。它与树的其余部分融合；结合方式与 `??` 相同（第 1 层，右结合）。其结果的元素是各元素的 `??`（`List<f64 | none> .?? 0.0` 是 `List<f64>`）。没有容器操作数时它是错误，帮助信息为 `x ?? y`。

`none` 与 NaN 保持区分：`.??` 只替换 `none`，NaN 元素是一个照常通过的值（`[1.0, none, NaN] .?? 0.0` 是 `[1.0, 0.0, NaN]`）。`f64 | none` 不以 NaN 作为其 `none` 来存储：NaN 是浮点算术的普通结果，把两者混为一谈会丢失缺失值或凭空造出缺失值。要把 NaN 当作缺失，对它做掩码：`a[f64::is_nan.(a)] = 0.0`。
