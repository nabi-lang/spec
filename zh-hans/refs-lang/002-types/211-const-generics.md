# 2.11. `const` 泛型

`const` 泛型在[泛型](210-generics.md)参数位置接受一个值。整数、布尔值以及在编译期确定的简短表达式都可以作为参数。

## 2.11.1. 动机

它们把在编译期确定的值（数组长度、矩阵维度、嵌入式内存池大小）作为类型的一部分携带。

```nabi
struct Matrix<T, const M: usize, const N: usize> {
    data: [T; M * N];
}

fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }
```

由于数组长度成为类型的一部分，调用时的长度检查没有任何开销。编译器还可以应用依赖长度的优化，例如 SIMD 和循环展开；而矩阵乘法这类关心维度兼容性的运算，可以在编译期验证。

## 2.11.2. 声明形式

```nabi
const ParamName : Type
```

`const` 关键字之后依次是参数名、`:` 和类型。类型必须是下文所述允许的 `const` 类型之一。

多个参数以逗号分隔。类型参数和 `const` 参数可以自由混合。

```nabi
fn replicate<T, const N: usize>(value: T) -> [T; N] { ... }
struct Tagged<T, const TAG: u32> { value: T; }
```

## 2.11.3. 允许的类型

`const` 参数的类型是整数类型（`i8` 到 `i64`、`u8` 到 `u64`、`usize`、`isize`）、`bool`、`char` 或 `c_char` 之一。

浮点类型、`str`、和类型以及用户定义的结构体不能作为 `const` 参数；这样的参数在其声明处属于错误。排除浮点类型是因为其位表示存在不确定性，排除用户定义类型是因为相等比较存在一致性问题。

## 2.11.4. 实例化时的实参

在调用处或实例化处，为 `const` 参数传入一个 `const` 表达式。

```nabi
let m: Matrix<f64, 3, 4> = ...;
let v: [i32; 16] = ...;
let arr = replicate::<f64, 8>(0.0f64);
```

实参必须能在编译期求值。允许的有：字面量、`const` 变量、`const` 函数调用、其他 `const` 参数，以及它们的简单算术组合。运行时变量不能用作 `const` 实参；这样做是编译错误。实参的类型必须是参数的类型：对于 `const X: usize`，`S<true>` 和 `S<-1>` 都是错误。

在泛型实参列表中，裸实参是字面量、取负的字面量（`-1`）或名称（`N`、`const` 变量）。其他任何形式，包括算术，都要用花括号括起来：

```nabi
fn grow<const N: usize>(b: Buf<N>) -> Buf<{ N + 1 }> { ... }
let b: Buf<{ 2 * 8 }> = ...;
```

数组长度（`[T; M * N]`）本身已是表达式位置，不需要花括号。

## 2.11.5. `const` 表达式中的算术

`const` 参数之间的简单算术可以在类型内部使用。

```nabi
struct Matrix<T, const M: usize, const N: usize> {
    data: [T; M * N];
}

fn flatten<T, const M: usize, const N: usize>(
    m: Matrix<T, M, N>
) -> [T; M * N] { ... }
```

允许的运算符是 `+`、`-`、`*`、`/`、`%` 和一元 `-`。当前规范不允许比较和条件，只允许简单算术。类型系统能够识别 `M * N` 和 `N * M` 是同一个值（交换律）。

更复杂的 `const` 表达式，例如条件或递归，由 `const` 函数规范涵盖。

## 2.11.6. 参数推断

如果 `const` 实参可以从实参类型、结构体字段类型或期望类型（`let` 注解、参数类型、返回类型）推断出来，就可以省略。

```nabi
fn replicate<T, const N: usize>(value: T) -> [T; N] { ... }

let arr: [i32; 8] = replicate(0i32);     // N = 8 inferred
let arr2 = replicate::<i32, 8>(0i32);    // explicit
```

推断方式与类型参数相同（[2.10](210-generics.md)）：每个实参和期望类型都参与推断，当有多处确定 `N` 时它们必须一致，否则该调用是类型错误。没有任何东西确定的 `const` 实参必须写出（`replicate::<i32, 8>(0i32)`）；省略它是编译错误。各来源之间没有默认值，也没有优先级。

头部可以根据参数计算实参：`impl<const A: i32> Root for Q<{ 2 * A }>` 以 `A = 1` 适用于 `Q<2>`，而不适用于 `Q<3>`。当参数只出现一次、只在常量的 `+`、`-`、`*` 和取负之下，并且解出的值是精确的时，该参数才被求解；其他任何头部表达式都不适用于任何类型。

### 重复字面量 `[v; N]`

`[v; N]` 是由 `N` 个元素组成的 `[T; N]` 数组，每个元素都是 `v` 的副本。`N` 是 `const` 表达式，可以是 `const` 参数。

`v` 在写入任何元素之前恰好求值一次，因此其副作用只发生一次（`N` 为 0 时也是如此）。之后每个元素都是独立的值：

- 如果 `T` 是 `Copy`，或者是其中没有任何引用共享存储之物的值类型（数字、`bool`、`char`、`str`、`none`、函数，以及由这些构成的数组、元组、结构体和枚举），每个元素都是 `v` 的按位副本，不调用 `clone`。
- 否则，元素 `0` 到 `N - 2` 是 `v.clone()`，按索引顺序调用，最后一个元素就是 `v` 本身。`List` 元素、列表的数组或包含 `Dict` 的结构体会得到 `N` 个独立的集合，因此通过一个元素写入不会改变其他元素。
- 如果 `T` 不是 `Clone`，该字面量是错误（E0069），其帮助信息会写出 `array::from_fn::<T, N>(|i| ..)`（[stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md)），它用各自的值构建每个元素；`List` 重复字面量的帮助信息会写出 `List::from_fn(N, |_| ..)`。类型参数需要 `T: Clone` 约束。
- 如果 `T` 持有只有一个所有者的值（`File` 这样必须关闭的类型，或 `Drop` 类型，处于任意数组深度），该字面量是错误，帮助信息相同：`N` 个元素不能共享同一个所有者。

```nabi
fn zeros<const N: usize>() -> [f64; N] { [0.0; N] }
let z: [f64; 4] = zeros();              // N = 4, from the expected type
let w = [1u8; 16];                       // [u8; 16]

let g = [[0; 3]; 2];
g[0][1] = 5;                             // g[1] is still [0, 0, 0]
let rows = [List::<i32>::new(); 3];
rows[0].push(1);                         // rows[1] and rows[2] stay empty
let fs = [try File::open("a"); 2];       // ERROR: a `File` has one owner (E0069)
```

## 2.11.7. 单态化

`const` 泛型的工作方式与[泛型](210-generics.md)的单态化相同。每一种 `const` 实参组合都会产生单独的编译结果。

```nabi
let a = replicate(0i32);    // N = ?, determined at inference time
```

`const` 实参组合相同的调用共享结果。

## 2.11.8. 相等性

`const` 实参值相同的两个类型是同一类型。

```nabi
let a: [i32; 4] = ...;
let b: [i32; 2 + 2] = ...;
// a and b have the same type
```

之所以能有这种相等性，是因为 `const` 表达式的求值是确定性的。

## 2.11.9. 可以定义的位置

`const` 泛型在以下所有位置都可以用相同的语法使用：函数和方法；结构体、`enum` 和 `union`；trait、`impl` 块和类型别名。

这种一致性与[泛型](210-generics.md)中的普通泛型相同，因此没有额外的规则需要记忆。

## 2.11.10. 用作索引

`const` 参数可以像普通 `const` 变量一样直接用在值位置。

```nabi
fn first<T, const N: usize>(arr: [T; N]) -> T {
    arr[0]
}

fn middle<T, const N: usize>(arr: [T; N]) -> T {
    arr[N / 2]
}
```

在值位置，`N` 是其声明类型（这里是 `usize`）的常量，与 `const` 项相同（[5.2](../005-modules/502-const.md)）：它可以用作索引、确定重复字面量 `[v; N]` 的大小、出现在常量表达式中（2.11.5），并作为 `const` 实参继续传递。它不是位置，因此不能被赋值。

函数体中的 `const` 和 `const { .. }` 块可以读取它，此时它们在每个实例中各有一个值（[5.2.10](../005-modules/502-const.md#5210-const-blocks)）。对实参的检查放在这样的块中，这样错误的实例就是其调用处的编译错误，而不是运行时的 panic：

```nabi
fn halves<const N: usize>(arr: [i32; N]) -> usize {
    const { assert(N % 2 == 0, "halves needs an even length") };
    const HALF: usize = N / 2;
    HALF
}
```

数组长度只能读取普通类型的大小：在 `T = i32` 时，`[u8; 4 * T::size()]` 是 `[u8; 16]`；但在泛型主体或结构体中 `T` 是类型参数时，它是错误，其帮助信息会建议使用 const 参数（`const N: usize`）或 `List<u8>`。
