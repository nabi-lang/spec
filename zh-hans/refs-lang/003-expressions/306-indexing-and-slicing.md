# 3.6. 索引与切片

用方括号（`[`、`]`）括起的表达式以两种形式访问值的一部分：索引，取出给定位置的元素；切片，截取一个区域。

```
index_expr      ::= primary "[" part ("," part)* ","? "]"
part            ::= expr | slice_arg
slice_arg       ::= expr? ":" expr? (":" expr?)?
```

单独只含一个 `slice_arg` 的方括号是切片（3.6.2）；其他所有方括号都是索引，每个以逗号分隔的部分是一个实参（3.6.1、3.6.3）。

## 3.6.1. 索引：`arr[i]`

索引在方括号内写一个索引表达式，以取得该位置的值。

```nabi
let v = arr[0];
let last = arr[arr.len() - 1];
let m = matrix[i, j];        // multidimensional, indices separated by commas
```

左侧可以是任意表达式；方括号内是一个或多个以逗号分隔的索引表达式。结果是被索引位置上的值。

### 脱糖

索引表达式按如下方式脱糖为 trait 调用。

```
arr[i]                 →   <typeof(arr) as Index<typeof(i)>>::index(arr, i)
arr[i] = v;            →   <typeof(arr) as IndexSet<typeof(i)>>::index_set(ref arr, i, v)
matrix[i, j]           →   <typeof(matrix) as Index<(typeof(i), typeof(j))>>::index(matrix, (i, j))
```

具名的最后一部分 `default:` 给出带默认值的索引 `d[k, default: v]`：

```
d[k, default: v]                  →   d.get_or(k, v)                      // a read; d unchanged
d[k, default: v] = e;             →   { let t = d; let i = k; t.get_or_insert(i, v); t[i] = e; }
d[k, default: v] op= e;           →   the same, with t[i] op= e
d[k, default: v].f op= e;         →   the same, with t[i].f op= e
d[k, default: v].m(args)          →   d.get_or_insert(k, v).m(args)
```

`d` 和 `k` 只求值一次；`v` 在表达式每次执行时求值。一个类型只要有 `get_or` 和 `get_or_insert`，就支持这种形式（`Dict`，[标准库 2.9.13](../../refs-standard-libraries/200-core/209-collections.md#2913-默认值运算符与构造函数)）。

读和写映射到不同的 trait，所以试图写入只读数据是编译错误。多维索引脱糖为单个索引 trait 调用，各索引被打包为一个元组：`g[i, j]` 即通过 `Index<(I, J)>` 的 `g[(i, j)]`。当一个类型有多个 `Index<K>` impl 时，由实参类型选出一个。

### 负索引

对于按位置索引的类型（`List`、`Slice`、定长数组 `[T; N]`，以及 `str` / `StrBuf` 的切片边界），负的**有符号整数**索引从末尾计数。实际位置为 `len + i`。

这是关于索引的值的规则，而不是关于其写法的规则：运行时算出的负值与字面量一样从末尾计数，没有单独的从末尾计数的形式。`i = 0` 时，`xs[i - 1]` 是最后一个元素，而不是 panic。

```nabi
let xs = [10, 20, 30];
xs[-1];                // 30
let i = -2;
xs[i];                 // 20 (variables too, not just literals)
let j = 0;
xs[j - 1];             // 30 (a computed -1 is the last element)
xs[-4];                // panic: index -4 out of bounds for length 3
```

- 无符号整数索引就是位置本身，没有从末尾计数的额外开销。
- 无类型的负字面量索引是 `isize`。
- 若 `len + i < 0`，则触发 panic（`index -4 out of bounds for length 3`）。越过末尾的非负索引也会触发 panic（`xs[3]`）。
- 不得从末尾计数的索引应为无符号（`usize`），或在使用前检查。
- 按键索引的类型（`Dict`）不受影响。负键就只是一个键。
- 转换使用 `core::index::index_from_end(i: isize, len: usize) -> usize`。

读取先求值目标，再求值索引；目标只求值一次。赋值先求值值，再求值目标和索引（[3.4.7](304-assignment.md#347-求值顺序)）。

### 边界检查与失败

方括号索引在有效范围外可能失败；失败如何表现由 `Index` 实现决定。通常是 panic 或错误。

要在不触发 panic 的情况下得到 `T | none` 值，使用库提供的方法，如 `.get(i)`。

## 3.6.2. 切片：`arr[start:end]`

当方括号中含有带冒号的切片形式时，使用切片。

```nabi
let part = arr[1:3];
let head = arr[:5];
let tail = arr[5:];
let all = arr[:];
let stepped = arr[::2];
let reverse = arr[::-1];
let full = arr[a:b:s];
let same = arr[1..3];        // a range also works (same as arr[1:3])
let whole = arr[..];         // same as arr[:]
```

方括号内的切片形式按以下 BNF 作为切片处理。

```
slice_expr  := expr? ':' expr? (':' expr?)?
              | range_expr                      // arr[a..b], arr[..]
              | expr ',' expr ',' ...           // a comma means indexing
```

基本形式 `[start : end]` 表示从 start（含）到 end（不含）的区域。可以加上步长：`[start : end : step]`。语义**与 Python 切片相同**。

- 边界被钳制到长度内。无论用哪种形式写边界（`a:b` 或区间 `a..b`），切片都不会因范围而触发 panic。`xs[5:100]` 尽可能多地截取；`start >= end`（正步长）给出空结果。
- 负的有符号整数边界从末尾计数（`len + i`），然后再钳制。`xs[-10:]` 是全部；`xs[:-10]` 是空。
- 正步长时，省略的 `start` 为 0，省略的 `end` 为长度。省略的步长为 1。
- 负步长时，边界按 Python 的方式解读：从 `start` 向后走到 `end` 之前为止。省略的 `start` 是最后一个元素；省略的 `end` 位于第一个元素之前。
- 步长为 0 时触发 panic（`slice step must not be zero`）。

```nabi
let xs = [1, 2, 3, 4, 5];
xs[-2:];               // [4, 5]
xs[1:-1];              // [2, 3, 4]
xs[5:100];             // [] (clamp)
xs[-10:];              // [1, 2, 3, 4, 5]
xs[:-10];              // []
xs[3:1];               // [] (start >= end)
xs[1..10];             // [2, 3, 4, 5] (a range clamps too)
xs[3..1];              // []
"abc"[1..10];          // "bc"
xs[::-1];              // [5, 4, 3, 2, 1]
xs[-1:0:-1];           // [5, 4, 3, 2]
xs[3:1:-1];            // [4, 3] (positions 3, 2)
xs[1:4:-2];            // []
xs[::0];               // panic: step 0
```

单元素索引（3.6.1）则不同：负值从末尾计数，但越界会触发 panic。两条规则都是有意为之：索引指称一个元素，它必须存在；切片指称一个区域，它可以为空。

`str` / `StrBuf` 的边界是字节偏移；它们先被钳制，若所得位置落在 UTF-8 字符的中间，则触发 panic。`str` 切片没有步长（`s[::2]` 是错误，E0056）。

### 脱糖

切片脱糖为一个区间和对 `Sliceable` trait 的调用（[标准库 2.3.4](../../refs-standard-libraries/200-core/203-ops.md#234-索引-trait)）。

```
arr[a : b]            →   arr.slice(a..b)                       // unsigned bounds, no step
arr[a : b]            →   arr.slice(slice_bound(a, n)..slice_bound(b, n))   // signed bounds
arr[a : b : k]        →   arr.slice(slice_range(a, b, k, n)).step(k)
arr[: b] / arr[a :]   →   omitted bounds use the defaults above (none for slice_range when a step is present)
arr[a..b] / arr[..]   →   arr.slice(a..b) / arr.slice(0..n)
arr[a : b]            →   slice_mut(arr, a..b)                  // a List where a SliceMut is expected
```

`n` 是 `arr.len()`，`arr` 只求值一次。每个 `Sliceable<Range<usize>>` 实现都用 `clamp_range` 钳制区间，所以直接调用 `arr.slice(9..2)` 也不会触发 panic。所用的辅助函数位于 `core::index`；它们是 `pub(pkg)` 的，由降级后的代码调用，不属于 API（从用户代码调用是 E0034）。

| 辅助函数                                                       | 含义                                                               |
|----------------------------------------------------------------|--------------------------------------------------------------------|
| `clamp_range(r: Range<usize>, n: usize) -> Range<usize>`       | 把终点钳制到 `n`，把起点钳制到终点                                 |
| `slice_bound(i: isize, len: usize) -> usize`                   | 为负时取 `len + i`；小于 0 时变为 0                                |
| `slice_range(start: isize \| none, stop: isize \| none, step: isize, len: usize) -> Range<usize>` | 与 Python 的 `slice.indices` 等价的正向区间。附加 `.step(step)` 后即得 `arr[start:stop:step]` |
| `index_from_end(i: isize, len: usize) -> usize`                | 用于单个索引。为负时取 `len + i`；溢出时触发 panic                 |
| `slice_mut(xs: List<T>, r: Range<usize> \| none) -> SliceMut<T>` | `xs[r]` 的可写视图（`none`：整个 `xs`），`r` 被钳制；仅供降级后的代码使用 |

### `Slice` 结果的种类

结果类型由 `Sliceable` 的关联类型确定。在标准库中如下（[标准库 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet视图)）。

| 目标                 | 结果                                                                 |
|----------------------|----------------------------------------------------------------------|
| `List<T>`            | `Slice<T>`：列表缓冲区的视图（元素写入可见）                         |
| `Slice<T>`           | `Slice<T>`：视图的视图                                               |
| `[T; N]`             | `Slice<T>`：数组副本的视图（数组是值）                               |
| 期望 `SliceMut<T>` 处的 `List<T>` | `SliceMut<T>`：列表的可写视图（[标准库 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt可写视图)）；无步长 |
| `SliceMut<T>`        | `SliceMut<T>`：视图的可写视图                                        |
| `str`、`StrBuf`      | 新的 `str`：字节副本（因为 `str` 必须保持为 C 字符串），带 UTF-8 边界检查 |

### 切片赋值

切片是一个位置：`arr[a:b:k] = v;` 通过它进行存储。

```
xs[a : b] = ys;       →   { let t = xs; slice_mut(t, a..b).copy_from_slice(ys[:]) }   // a List
m[a : b : k] = v;     →   IndexSet::index_set(m, SliceArg::new(a, b, k), v)           // an AxisIndexed type (3.6.3)
```

- 在 `List` 上，值来自长度完全相同的 `List` / `Slice`（否则 `copy_from_slice` 触发 panic；改变长度用 `splice`）。列表的带步长切片不能被赋值（错误，帮助信息提示逐个存储元素）。
- 在 `Array` 这样的按轴索引类型上，选区是一个视图，值是填满它的标量（`a[1:3] = 0.0`）或拉伸到其形状的数组（`a[1:3] = b`），取决于哪个 `IndexSet` impl 接受该值的类型。
- `str`、`StrBuf`、`Slice` 和定长数组不接受切片赋值（错误）；`SliceMut` 视图逐个元素写入，或用 `copy_from_slice` 写入。

## 3.6.3. 多轴索引

方括号中每个以逗号分隔的部分是一个实参，而一个部分可以是切片：`m[1:, ::2]`、`m[i, :]`。各部分作为一个元组通过 `Index` / `IndexSet` 传递（3.6.1），切片部分以值 `core::index::SliceArg { start, stop, step }` 传递（各为 `isize | none`，缺失的部分为 `none`）：

```
m[1:, ::2]          →   Index::index(m, (SliceArg::new(1, none, none), SliceArg::new(none, none, 2)))
m[i, :]             →   Index::index(m, (i, SliceArg::FULL))
m[1:3, :] = x       →   IndexSet::index_set(m, (SliceArg::new(1, 3, none), SliceArg::FULL), x)
```

`SliceArg::indices(len)` 对一个轴应用 3.6.2 的规则，返回 `(first, count, step)`。

实现标记 trait `core::index::AxisIndexed` 的类型（`Array` 实现了它，[标准库 2.9.14](../../refs-standard-libraries/200-core/209-collections.md#2914-arrayt-const-n-usizen-维数组)）把每个部分作为一个*轴实参*接收，因此一组基于 `isize` 的 impl 就能服务于索引的所有写法：

| 部分                        | 轴实参                           | 含义（对 `Array`）                             |
|-----------------------------|----------------------------------|-----------------------------------------------|
| 任意类型的整数              | `isize`（字面量就是它；类型仅由字面量给出的变量，如 `for i in 0..4` 的 `i`，按其 `i64` 处理；`usize`、`u64`、`i64` 和 `u32` 会被检查，其他类型会被拓宽） | 一个位置；该轴被去掉；负值从末尾计数（3.6.1） |
| `a:b:k`                     | `SliceArg`                       | 该轴上带步长的区间（3.6.2 的规则）             |
| 与其他部分并列的 `..`       | `core::index::Rest`         | 其他部分剩下的每个轴，整个保留                 |
| `none`                      | `core::index::NewAxis`      | 一个长度为 1 的新轴                            |
| 其他任何东西（掩码）        | 按原样                           | 由 impl 决定（`Array<bool, N>`：掩码）         |

```nabi
use core::array::Array;

let a: Array<f64, 2> = [[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]];
a[1, :]          // [4.0, 5.0, 6.0]: row 1, a view
a[:, -1]         // [3.0, 6.0]: the last column
a[::-1, 1:]      // rows reversed, columns 1..
a[.., 0]         // [1.0, 4.0]
a[:, none]       // shape [2, 1, 3]
a[a .> 2.0]      // [3.0, 4.0, 5.0, 6.0]: a mask selects a copy
a[:, 0] = 0.0;   // fills a column
```

单独的 `..`（`a[..]`）仍是整个区间（3.6.2）。不是 `AxisIndexed` 的类型按各部分自己的类型接收它们（整数字面量取其默认类型），由其 `Index` impl 决定接受什么。

在沿单个轴索引的类型（`List`、`Slice`、定长数组、`str`）上使用其中含有切片的多部分索引是错误（E0143），帮助信息会提示在分开的方括号中切片和索引（`m[1:][0]`），或把数据保存在 `Array` 中。

## 3.6.4. 空方括号与格式错误的形式

以下形式都是格式错误的。

```
arr[]               // ERROR: empty brackets
arr[a:b:c:d]        // ERROR: three or more colons
arr[: : :]          // ERROR: three colons
```

这些形式在词法和语法阶段就被拒绝。

## 3.6.5. 索引与切片依赖于 `trait`

语言本身不决定哪些类型支持索引或切片。只有当标准库为某个类型实现了 `Index`、`IndexSet` 和 `Sliceable` 时，`arr[i]` 和 `arr[1:3]` 才对该类型有效。

语言定义方括号词法单元的形式和脱糖规则；含义属于库。
