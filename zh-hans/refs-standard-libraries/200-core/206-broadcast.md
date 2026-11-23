# 2.6. broadcast：逐元素运算符

`core::ops` 中的广播 trait 是 Julia 风格点运算符的 trait 依据。点运算符（`.+` / `.*` / `.==` / `f.(x)`）在语法本身上把逐元素的含义区分开来，从而排除了与用户定义的 `*`（例如矩阵乘法）混淆的可能。

本章定义 20 个广播 trait，以及函数广播的脱糖规则。

这些 trait 声明在 `core::ops` 中，这是它们唯一的路径（[0.1.6](../000-meta/001-tier-model.md#016-每个项只有一条路径)）；没有 `broadcast` 子模块。[2.3.10](203-ops.md#2310-unicode-数学运算符-trait) 中的 `⋅` 点积 trait `Dot` 与它们并列。`Dot` 不是 20 个广播 trait 之一：`a ⋅ b` 把两个操作数归约为一个值（`fn dot(self, rhs: Rhs) -> Output`），而不是逐元素运算，标准库也没有为任何类型实现它。

## 2.6.1. 动机：为什么要单独的点运算符

在 NumPy 和 MATLAB 中，`*` 对数组表示逐元素运算、对矩阵却表示乘法，这种**无声的歧义**长期以来都是 bug 的来源（PEP 465 引入 `@` 也是出于同样的考虑）。

相比之下，Nabi 的策略很明确。

```nabi
m * n       // user-defined Mul: matrix multiplication, dot product, etc.
m .* n      // element-wise (Hadamard product), always broadcast
```

点前缀（`.`）是表明逐元素意图的视觉信号，并由编译器强制执行。在任何容器（List、Vec、Matrix 等）中，`.+` 都一致地分派到 `DotAdd::dot_add`。

## 2.6.2. 20 个 trait：运算符对应关系

完整的集合是算术（7）+ 比较（6）+ 逻辑（2）+ 位运算（3）+ 移位（2）= **20** 个 trait。

> [!NOTE]
> 本章定义全部 20 个 trait：7 个算术、6 个比较、2 个逻辑、3 个位运算和 2 个移位。点比较（`.==`/`.!=`/`.<`/`.>`/`.<=`/`.>=`）在 [2.5.8 广播比较](205-cmp.md#258-广播比较-trait) 中只给出**简要签名**；
> 本章是**完整的依据**。

### 算术（7）

| trait       | 运算符 | 优先级 | 含义                       |
|-------------|----------|------------|-------------------------------|
| `DotAdd`    | `.+`     | 9（左）   | 逐元素加法         |
| `DotSub`    | `.-`     | 9（左）   | 逐元素减法      |
| `DotMul`    | `.*`     | 10（左）  | 逐元素乘积（Hadamard） |
| `DotDiv`    | `./`     | 10（左）  | 逐元素除法         |
| `DotRem`    | `.%`     | 10（左）  | 逐元素余数（Rem）  |
| `DotModulo` | `.%%`    | 10（左）  | 逐元素向下取整取模   |
| `DotPow`    | `.**`    | 13（右） | 逐元素幂（`**`，2.3.1） |

### 比较（6）

| trait   | 运算符 | 优先级 | 含义                             |
|---------|----------|------------|-------------------------------------|
| `DotEq` | `.==`    | 4（无）   | 逐元素 == → `Container<bool>` |
| `DotNe` | `.!=`    | 4（无）   | 逐元素 !=                     |
| `DotLt` | `.<`     | 4（无）   | 逐元素 <                      |
| `DotGt` | `.>`     | 4（无）   | 逐元素 >                      |
| `DotLe` | `.<=`    | 4（无）   | 逐元素 <=                     |
| `DotGe` | `.>=`    | 4（无）   | 逐元素 >=                     |

### 逻辑（2）

| trait           | 运算符 | 优先级 | 含义                                  |
|-----------------|----------|------------|------------------------------------------|
| `DotLogicalAnd` | `.&&`    | 3（左）   | 逐元素逻辑与（不短路） |
| `DotLogicalOr`  | `.\|\|`    | 2（左）   | 逐元素逻辑或（不短路）  |

> [!IMPORTANT]
> **`.&&` / `.||` 不短路**。与普通的 `&&` / `||` 不同，广播被降级为一个融合循环，所以两个操作数总会被求值。在语义上，它是 bool 容器的逐元素逻辑组合。放在任一操作数中带副作用的函数调用总会执行。

### 位运算（3）

| trait       | 运算符 | 优先级 | 含义        |
|-------------|----------|------------|----------------|
| `DotBitAnd` | `.&`     | 7（左）   | 逐元素 & |
| `DotBitOr`  | `.\|`     | 5（左）   | 逐元素 \| |
| `DotBitXor` | `.^`     | 6（左）   | 逐元素 ^ |

### 移位（2）

| trait    | 运算符 | 优先级 | 含义         |
|----------|----------|------------|-----------------|
| `DotShl` | `.<<`    | 8（左）   | 逐元素 << |
| `DotShr` | `.>>`    | 8（左）   | 逐元素 >> |

优先级和结合性与普通 trait 相同（[2.3 ops](203-ops.md)、[2.5 cmp](205-cmp.md)）。

## 2.6.3. trait 签名：统一形式

每个点 trait 都遵循同一种统一形式。

```nabi
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+")]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*")]
    fn dot_mul(self, rhs: Rhs) -> Output;
}

trait DotModulo<Rhs = Self> {
    type Output;
    #[infix(".%%")]
    fn dot_modulo(self, rhs: Rhs) -> Output;
}

trait DotEq<Rhs = Self> {
    type Output;
    #[infix(".==")]
    fn dot_eq(self, rhs: Rhs) -> Output;
}

trait DotLogicalAnd<Rhs = Self> {
    type Output;
    #[infix(".&&")]
    fn dot_logical_and(self, rhs: Rhs) -> Output;
}
```

算术点 trait 与 `Add`/`Sub`/`Mul`/`Div`/`Rem`/`Modulo` 完全对称：一个 `Output` 关联类型加一个中缀属性。六个比较点 trait（`DotEq`、`DotNe`、`DotLt`、`DotGt`、`DotLe`、`DotGe`）形式相同，各有自己的 `Output`，通常是 `Container<bool>`。逻辑点 trait（`DotLogicalAnd` / `DotLogicalOr`）通常两边都接受 `Container<bool>` 并返回 `Container<bool>`；由于不短路，两个容器都会被求值到底。

### 函数广播：没有 trait

函数广播的写法如下。

```nabi
f.(x)              // function broadcast
f.(a, b)           // multiple arguments
```

函数广播没有自己的 trait；它按如下方式脱糖。

```nabi
f.(x)        →  x.into_iter().map(f).collect()
g.(a, b)     →  a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

它直接分派到 [2.7 iter](207-iter.md) 的 `map`/`zip`；broadcast 模块不定义单独的 trait。区别在于函数本身是一个可调用对象，而不是分派目标。

## 2.6.4. 语义：逐元素

### 算术点运算：两个形状相同的容器

对两个形状相同的容器进行算术点运算，行为如下。

```nabi
let a: List<i32> = [1, 2, 3];
let b: List<i32> = [10, 20, 30];

let c = a .+ b;          //→ [11, 22, 33]
let d = a .* b;          //→ [10, 40, 90]
let e = b .- a;          //→ [9, 18, 27]
let f = b ./ a;          //→ [10, 10, 10]
```

长度不匹配会在运行时触发 panic，并指出方法名（`List::dot_add: lengths 3 and 2 differ; make both sides the same length`）；融合为一个循环的运算符表达式（2.6.6）可能包含多个运算符，此时消息以 `broadcast:` 开头。如果大小在编译时已知，就作为编译错误报告。

```nabi
let x: [i32; 3] = [1, 2, 3];
let y: [i32; 4] = [1, 2, 3, 4];
let z = x .+ y;          // ERROR (compile: fixed sizes differ)

let p: List<i32> = [1, 2, 3];
let q: List<i32> = [1, 2];
let r = p .+ q;          // panic at runtime: length mismatch
```

### 标量广播

把标量与容器组合，行为如下。

```nabi
let a: List<i32> = [1, 2, 3];
let s = a .+ 10;         //→ [11, 12, 13]  (scalar broadcast)
let m = 2 .* a;          //→ [2, 4, 6]
let t = a ./ 2;          //→ [0, 1, 1]    (i32 truncate)
```

这里由单独的实现（`DotAdd<T> for List<T>`，以及用于标量在左形式的 `DotAdd<List<T>> for T`）处理标量一侧。`Slice<T>` 也有同样的一对实现；定长数组 `[T; N]` 没有标量在左的实现。库容器以同样方式为自己的类型获得标量在左的形式：孤儿规则允许在定义 `Matrix` 的包中写 `impl DotMul<Matrix> for f64`，因为本地类型 `Matrix` 出现在任何类型参数之前（[2.9.5](../../refs-lang/002-types/209-impl.md#295-孤儿规则)），所以 `2.0 .* m` 和 `m .* 2.0` 都可用。

### 比较点运算：返回 `Container<bool>`

比较点运算符返回一个 `Container<bool>`。

```nabi
let a: List<i32> = [1, 5, 3];
let b: List<i32> = [2, 5, 1];

a .== b;                 //→ [false, true, false]
a .< b;                  //→ [true, false, false]
a .>= b;                 //→ [false, true, true]

a .< 4;                  //→ [true, false, true]   (scalar broadcast)
```

### 余数点运算：`.%` 和 `.%%`

余数点运算符逐元素地应用普通 `%` / `%%` 的含义。

```nabi
let a: List<i32> = [-7, 5, -3, 8];
let b: List<i32> = [3, 3, 3, 3];

a .% b;                  //→ [-1, 2, 0, 2]   (Rem: sign follows the left operand)
a .%% b;                 //→ [2, 2, 0, 2]    (Modulo: always in [0, 3))
```

### 逻辑点运算：`.&&` 和 `.||`

逻辑点运算符产生两个 `Container<bool>` 的逐元素逻辑组合。与不带点的形式不同，它们不短路，所以两个容器总会被求值到底。

```nabi
let p: List<bool> = [true, false, true, false];
let q: List<bool> = [true, true, false, false];

p .&& q;                 //→ [true, false, false, false]
p .|| q;                 //→ [true, true, true, false]

// combining comparison results: the most common use
let xs: List<i32> = [1, 5, 3, 8];
(xs .>= 2) .&& (xs .< 6);  //→ [false, true, true, false]
```

### 位运算 / 移位点运算

位运算和移位点运算符的工作方式相同。

```nabi
let xs: List<u8> = [0b1100, 0b1010, 0b0011];
let ys: List<u8> = [0b1010, 0b1010, 0b1111];

xs .& ys;                //→ [0b1000, 0b1010, 0b0011]
xs .| ys;                //→ [0b1110, 0b1010, 0b1111]
xs .^ ys;                //→ [0b0110, 0b0000, 0b1100]

let ns: List<u32> = [1, 2, 4];
ns .<< 2;                //→ [4, 8, 16]
```

## 2.6.5. 函数广播：`f.(x)`

函数广播的写法是在普通函数调用处加一个点。

```nabi
let xs: List<f64> = [0.0, 0.5, 1.0];

let s = f64::sin.(xs);               //→ [0.0, 0.479..., 0.841...]
let r = f64::sqrt.(xs);              //→ [0.0, 0.707..., 1.0]
let g = (|x: f64| x * 2.0).(xs);     //→ [0.0, 1.0, 2.0]
```

可以同时广播多个实参。

```nabi
let a: List<f64> = [1.0, 2.0, 3.0];
let b: List<f64> = [10.0, 20.0, 30.0];

let r = f64::pow.(a, b);             //→ [1.0, 1_048_576.0, ...]
let m = f64::max.(a, b);             //→ [10.0, 20.0, 30.0]
```

是 `List`、`Slice` 或数组的实参一起迭代；其他实参原样传给每次调用（`f64::pow.(xs, 2.0)`）。被迭代的实参长度必须相同，在运行时检查。实参个数**没有限制**，所以任何 N 元函数都适用。

## 2.6.6. 循环融合

编译器把连续的点运算融合为一个循环。

```nabi
let result = f64::sin.(a) .+ f64::cos.(b) .* 2.0;
```

朴素的降级需要 3 次临时 List 分配和 3 个循环。对于 `List`、`Slice`、数组以及任何 `Broadcastable` 类型，Nabi 保证只有一个循环，且没有中间集合（确切条件见 [3.3.4](../../refs-lang/003-expressions/303-broadcast-operators.md#334-自动循环融合)）。就地形式（`y .= ..`、`y .+= ..`，[3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-原地广播)）直接写入目标，完全不分配。

```nabi
// conceptually equivalent to this single elementwise call
let result = elementwise(a, b, |x, y| x.sin() + y.cos() * 2.0);
```

### `Broadcastable`、`BroadcastScalar` 和 `Materialize`

融合协议由 `core::ops` 的三个 trait 组成。

```nabi
#[lang("broadcastable_trait")]
pub trait Broadcastable<const N: usize> {
    type Elem;
    /// The elements as an array: a view sharing the storage.
    fn broadcast_view(self) -> Array<Elem, N>;
}

#[lang("broadcast_scalar_trait")]
pub trait BroadcastScalar {
    type Elem;
    /// The plain value the loop applies to every element.
    fn broadcast_scalar(self) -> Elem;
}

#[lang("materialize_trait")]
pub trait Materialize<U, const N: usize> {
    type Output;
    /// The result built around `out`, a fresh row-major array.
    fn materialize(out: Array<U, N>) -> Output;
}
```

`Array`（[2.9.14](209-collections.md#2914-arrayt-const-n-usizen-维数组)）由缓冲区、偏移、形状和步长组成，因此能描述任何带步长的存储；自定义类型（包中的 `Tensor`、数据框的列）通过返回它持有的 `Array` 来实现 `Broadcastable`，并实现 `Materialize` 以便从结果的元素构建自身。`Array` 两者都实现：它的视图就是它自己，`materialize` 返回交给它的 `Array<U, N>`。实现 `Broadcastable` 意味着承诺该类型的点运算符作用于所视图的元素，就像作用于普通值一样；包装单个数字的类型（带单位的量）实现 `BroadcastScalar`，以标量身份与它们并列。

```nabi
struct Signal { data: Array<f64, 1>; }
impl Broadcastable<1> for Signal {
    type Elem = f64;
    fn broadcast_view(self) -> Array<f64, 1> { self.data }
}
impl Materialize<f64, 1> for Signal {
    type Output = Signal;
    fn materialize(out: Array<f64, 1>) -> Signal { Signal { data: out } }
}
let t = s .* 2.0 .+ k;      // one loop; a Signal
```

融合表达式对每个操作数取视图（`broadcast_view`；标量包装用 `broadcast_scalar`），把各视图拉伸到同一形状（2.6.8），运行一个循环写入一个具有结果形状的新数组，然后把它交给结果类型的 `materialize`。循环直接在缓冲区上运行：当结果和每个视图都连续时，是一段连续位置；否则按最后一个轴逐行遍历，每个视图使用自己的步长。

由哪个类型构建结果：

- 容器全是 `Array` 的表达式树取首要操作数（秩最高者中的第一个）的类型：`Array` 的 `materialize` 给出元素类型为循环所计算的 `U` 的 `Array<U, N>`（`a .< b` 是 `Array<bool, N>`）。
- 作用于自定义类型、或含有 `BroadcastScalar` 操作数的表达式树，先按其点 trait 逐个运算符地确定类型，就好像没有融合一样。如果该类型是与树的秩相同的 `Broadcastable`，并且有其元素类型的 `Materialize`，就由它构建结果；否则这棵树不融合，通过 trait 运行。因此 trait 的类型对融合循环同样成立：一个其 `DotDiv` 给出 `Quantity<Array<f64, R>, { L - L2 }, ..>` 的量类型，使 `d ./ t .+ v0` 成为结果为速度的一个循环，而 `d .+ t` 与不融合时一样是编译错误。
- 不实现树中任何点 trait 的自定义类型，只从 `Broadcastable` 获得运算符：由首要操作数的类型构建结果。

首要操作数没有 `Materialize` 是编译错误，错误会指出该 trait。作用于此类类型的就地形式（`v .= d ./ t`）以同样方式确定类型，类型与目标不符的值是编译错误（`` `.=` writes a `Q<Array<f64, 1>, 1>` into a `Q<Array<f64, 1>, -1>` ``）。

## 2.6.7. trait 实现矩阵

点 trait 的实现与容器类型的对应关系如下。

| 容器        | DotAdd   | DotMul   | DotEq       | 说明                                   |
|------------------|----------|----------|-------------|-----------------------------------------|
| `List<T>`        | T:Add    | T:Mul    | T:PartialEq | 逐元素，长度必须一致        |
| `[T; N]`（定长） | T:Add    | T:Mul    | T:PartialEq | 固定 N，编译时                   |
| `Slice<T>`       | T:Add    | T:Mul    | T:PartialEq | 视图，逐元素                      |
| `i32`/`f64`/等 | 标量在左 | 标量在左 | 标量在左 | 没有标量与标量的实现（用普通的 `+`/`*`/`==`）；`T` 为标量在左形式实现 `DotAdd<List<T>>` / `DotAdd<Slice<T>>`（2.6.4） |
| `T \| none`       | （运算符） | （运算符） | （运算符） | 可选值传播（`none .+ x = none`） |

`T | none` 的点运算含义是可选值传播：只要任一侧是 `none`，结果就是 `none`。它在运算符层面处理，没有库实现：整棵点表达式树的操作数按源代码顺序各求值一次；然后逐个检查每个 `T | none` 操作数，第一个 `none` 使整棵树的值为 `none`，而不运行这棵树。否则树在 `T` 值上运行，按 2.6.6 所述融合或通过点 trait 运行，因此不会以 `none` 调用任何点 trait。值的类型是树的类型 `| none`。

core 的 n 维 `Array<T, N>`（[2.9.14](209-collections.md#2914-arrayt-const-n-usizen-维数组)）实现了每个点 trait：数组与同秩数组、数组与标量、标量与数组，均按 2.6.8 的规则。库的数值容器（`Tensor<T>`、数据框的列）直接实现同样的 trait，包括标量在左的情况：孤儿规则允许在定义 `Tensor` 的包中写 `impl DotMul<Tensor> for f64`（[refs-lang 2.9.5](../../refs-lang/002-types/209-impl.md#295-孤儿规则)）。作用于此类类型的点表达式逐个运算符地调用 trait，除非该类型是 `Broadcastable`（2.6.6）；`Array` 是，所以它的点表达式会融合，而它的 trait 实现用于满足约束和按名调用。

## 2.6.8. NumPy 风格的广播规则

只有一条规则，定义在 core 中，使所有数组类型保持一致：`core::array::broadcast_shape(a, b)` 接受两个秩相同的形状，逐轴比较，相等的长度保留，长度 1 拉伸为另一侧的长度。其他任何组合都会触发 panic，并指出两个形状和轴（`broadcast: shapes [2, 3] and [2, 2] differ on axis 1; each axis must match or have length 1`）。标量作用于每个元素。在点表达式中，秩不同的操作数按最后的轴对齐：较短的一方视为具有长度为 1 的前导轴（`[3]` 向量与 `[2, 3]` 矩阵得到 `[2, 3]`），结果取最高的秩，这一点由类型体现。点 trait 实现本身只接受一种秩，所以泛型代码和按名调用需要显式对齐秩（`v[none, :]` 把 `[3]` 向量变为 `[1, 3]` 的行）。`Array::broadcast_to(shape)` 直接给出拉伸后的视图（拉伸轴上步长为 0，不复制）。

一维容器（`List<T>`、`[T; N]`、`Slice<T>`）只有一个轴，不做长度 1 拉伸：它们的长度必须相等（2.6.4）。

## 2.6.9. 与其他语言的比较

与其他主流语言相比：

| 方面             | NumPy                | Julia                      | MATLAB           | Nabi                           |
|--------------------|----------------------|----------------------------|------------------|-------------------------------|
| 逐元素       | `*`（无声）         | `.*`（显式）            | `.*`（显式）  | `.*`（显式）               |
| 矩阵乘法         | `@`（PEP 465）        | `*`                        | `*`              | `*`（用户定义的 Mul）        |
| 逐元素加法   | `+`（无声）         | `.+`                       | `.+`             | `.+`                          |
| 函数广播 | `np.vectorize(f)(x)` | `f.(x)`                    | `arrayfun(f, x)` | `f.(x)`                       |
| 循环融合        | **无**             | `@.` 宏（自动）     | **无**         | 点运算链自动融合        |
| 比较         | `==` 返回数组   | `.==`                      | `==`（易混淆） | `.==`                         |
| 单独的点 trait | **无**            | **无**（只有 `*` 重载） | **无**       | **DotAdd/DotMul/...**（20 个）    |

Nabi 与其他语言最大的不同有两点。第一，每个点运算符都有自己的 trait。Julia 把点语法叠加在重载的 `+` 之上，而在 Nabi 中 `Add` 和 `DotAdd` 是不同的 trait，`Matrix * Matrix` 与 `Matrix .* Matrix` 走完全不同的分派。第二，编译器强制使用点运算符。通过普通的 `*` 进行广播是**不可能的**，这保证了一种不依赖用户代码的确定性区分。
