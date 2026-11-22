# 2.7. niche 优化

niche 是在某个类型的有效位模式中永远不会出现的位模式。把和类型的标签存放在这类位模式中，就不再需要单独的标签槽，从而得到零内存开销的和类型。

这种优化之所以可行，是因为[和类型](206-sum-types.md)的内存表示交给编译器决定。

## 2.7.1. 动机

```nabi
let p: Ptr<u8> | none;     // 64-bit pointer + none
```

这个类型必须能用 8 字节表示，而不是 16 字节（指针 8 + 标签 1 + 填充 7）。指针的 `null` 位模式（0x0000_0000_0000_0000）永远不会出现在有效指针中，因此可以把这个位置用于 `none`。

这种节省对科学计算和数据代码的内存效率至关重要（灵感来自 Julia 的 isbits `union`）。

## 2.7.2. 自动 niche

`none` 的位由实现定义。由一个项加上 `none` 组成的和类型 `T | none`，如果 `T` 有 niche，其大小就等于 `T` 的大小，`none` 用该 niche 值表示。niche 值以全零位存储：当 niche 值不为 0 时，该项以其 niche 标量与 niche 值异或后的结果存储，这个结果永远不为零。因此，对每个 niche 和类型，零填充的存储都读出为 `none`。

| 类型                                       | niche 值                              |
|--------------------------------------------|---------------------------------------|
| `bool`                                     | `2`                                   |
| `char`（编译器基本类型）                   | 代理码位 `0xD800`                     |
| 无字段的 `enum`、载荷中不含 GC 引用的 `enum` | 第一个未使用的标签                  |
| `extern "C" fn(..) -> R`                   | null                                  |
| `Ptr<T>` / `ConstPtr<T>` / `ManyPtr<T>`    | null（`#[niche(0)]`）                 |
| 单字段 `struct` / 元组                     | 该字段的 niche                        |
| 多字段 `struct` / 元组                     | 第一个具有 niche 的字段的 niche，跳过 `bool` 和浮点数的 niche |

- 载荷中持有 GC 引用的 `enum` 没有 niche。
- `bool` 或浮点标量上的非零 niche（`bool`，或 `#[niche(NAN)]` 浮点包装类型）只经由单字段包装类型传递。多字段聚合会跳过这样的字段，所以 `(i64, bool) | none` 有标签，而 `(i64, char) | none` 使用 `char` 的 niche。
- 当 niche 值为 0（空指针、`extern "C" fn`、`NonZero`）时，该项的位保持不变，因此这样的 `T | none` 可以作为可空值原样跨越 C ABI 传递（[5.9.4](../005-modules/509-extern-block.md#可空的-c-函数指针extern-c-fn---r--none)）。
- 除 `none` 外有两项或更多的和类型带有标签。

```nabi
Ptr<u8> | none           // 8 bytes, null is none
bool | none              // 1 byte
```

## 2.7.3. 没有自动 niche 的类型

整数（`i8` .. `i64`、`u8` .. `u64`、`usize`、`isize`）和浮点类型（`f32`、`f64`）没有 niche，因为每个位模式都有效。需要 niche 时，使用包装类型（2.7.6）或 `#[niche]`（2.7.4）。

## 2.7.4. 选择启用的 niche：`#[niche(..)]`

```
niche_attr ::= "#" "[" "niche" "(" niche_arg ("," niche_arg)* ","? ")" "]"
niche_arg  ::= int_lit | "-" int_lit | const_path | range
range      ::= niche_val "..=" niche_val | ".." niche_val | niche_val ".."
const_path ::= "MIN" | "MAX" | "NAN" | "INFINITY" | "NEG_INFINITY" | type "::" const_name
```

```nabi
#[niche(0)]
struct UserId(u64);

#[niche(i32::MIN..=-1)]
struct NonNegative(i32);

#[niche(NAN, INFINITY, NEG_INFINITY)]
struct Real { value: f64; }
```

- 实参可以是整数、`MIN` / `MAX`（`T::MIN`）、`NAN` / `INFINITY` / `NEG_INFINITY`（`F::NAN`）、区间 `a..=b` / `..b` / `a..`，以及其中多个的组合。它们按实例化后的字段类型解读。
- 列出的值是一种承诺：它们不是该类型的有效值。编译器不检查这一点；如果出现这样的值，和类型的判别就会出错。

## 2.7.5. 可以附加的位置

`#[niche(..)]` 附加在单字段 `struct` 上（元组 newtype 或只有一个具名字段）。把它附加在 `enum`、`union` 或多字段 `struct` 上是错误。

## 2.7.6. 标准库包装类型

| 名称          | 排除的值                              | 定义                          |
|---------------|---------------------------------------|-------------------------------|
| `NonZero<T>`  | 0（仅限整数 T）                       | [标准库 2.2.2](../../refs-standard-libraries/200-core/202-non-zero.md#222-nonzerot禁止零) |
| `NonNan<F>`   | NaN                                   | [标准库 2.2.3](../../refs-standard-libraries/200-core/202-non-zero.md#223-nonnanf禁止-nan) |
| `Finite<F>`   | NaN、±∞                               | [标准库 2.2.4](../../refs-standard-libraries/200-core/202-non-zero.md#224-finitef禁止-nan-和-) |
| `Positive<T>` | 负数（含 0）。无符号 `T` 没有 niche（`Positive<u32> \| none` 为 8 字节） | [标准库 2.2.5](../../refs-standard-libraries/200-core/202-non-zero.md#225-positivet禁止负值) |
