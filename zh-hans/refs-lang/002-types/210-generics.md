# 2.10. 泛型

泛型是为函数、类型、trait 和 `impl` 块提供类型参数的机制。在调用处或实例化处，每一组具体类型元组都会产生一份单独的编译结果；这个过程称为单态化。

```
generics        ::= "<" generic_param ("," generic_param)* ","? ">"
generic_param   ::= type_param | const_param
type_param      ::= identifier (":" type_bounds)? ("=" type_expr)?
const_param     ::= "const" identifier ":" type_expr ("=" const_expr)?   // const_expr: 5.2
type_bounds     ::= type_expr ("+" type_expr)*
where_clause    ::= "where" where_pred ("," where_pred)* ","?
where_pred      ::= type_expr ":" type_bounds
```

> [!NOTE]
> `where` 是上下文关键字，只在函数签名、`impl` 头部或 `struct` / `enum` / `union` / 类型别名头部之后才是关键字（[1.5.4](../001-lexical/105-keywords.md#154-上下文关键字)）。

## 2.10.1. 参数的声明位置

泛型参数出现在以下位置：

```nabi
fn id<T>(x: T) -> T { x }                       // function
struct Pair<A, B> { first: A; second: B; }      // struct
enum Result<T, E> { Ok(T), Err(E) }              // enum
union Bits<T> { v: T; raw: u64; }               // union
trait Convert<Target> { ... }                    // trait
impl<T> List<T> { ... }                          // impl
type Maybe<T> = T | none;                        // type alias
```

参数的声明形式在每个位置都相同：在名称之后的 `<...>` 中列出参数名，以逗号分隔。

写在 `impl` 块中的方法不得声明与 impl 的泛型参数同名的泛型参数：`impl<K> W<K> { fn get<K>(..) }` 是错误（E0141），因为在方法内部 `K` 可以指两者之一；应重命名方法的参数（`fn get<K2>`）或使用 impl 的参数。trait 的默认方法写在 trait 中而不是 impl 中，因此它的参数会遮蔽实现它的 impl 中的同名参数：`impl<K, V> Iterator for DictIter<K, V>` 上的 `Iterator::min_by_key<K>` 有它自己的 `K`。

## 2.10.2. 参数的种类

### 类型参数

```nabi
fn first<T>(pair: (T, T)) -> T { pair.0 }
```

参数名遵循[标识符](../001-lexical/104-identifiers.md)的形式。按照惯例，使用 `T`、`U`、`K`、`V` 这样的单个大写字母或简短的 PascalCase 名称，但这不是强制的。

### `const` 参数

这类参数以值作为参数。确切规则在 [`const` 泛型](211-const-generics.md)中单独定义。

### 类型实参推断

如果调用处没有写出类型实参，编译器会推断它们。除实参类型外，期望类型（`let` 的类型注解、实参位置的参数类型、返回位置的返回类型）也用于推断。

```nabi
fn empty<T>() -> List<T> { [] }

let xs: List<i32> = empty();        // T = i32, from the let's type annotation
takes_strs(empty());                // T = str, from the parameter type List<str>
```

没有期望类型的 `let` 由之后的使用推断。在未注解的 `let x = f();` 中，如果仅凭调用不能确定 `f` 的泛型返回类型，就由之后的使用确定。签名始终是显式的。

- **范围**：变量的作用域，即块的剩余部分，包括其中的闭包和 async 主体。
- **确定类型的使用**：把值传给类型固定的位置的使用：函数 / 方法实参、赋值、带注解的 `let`、结构体字段、`return`、主体的值。运算符根据另一个操作数确定开放字面量操作数的类型（`c += x`）；除此之外，运算符、f-string 插值、模式和字段读取都不确定类型。
- **空构造器**：`[]`、`Dict::new()`、`Set::new()`、`channel()`、`Mutex::new([])` 的类型实参以同样方式确定，也可以通过 `d[k] = v` 和元组 `let`（`let (tx, rx) = channel(); tx.send(1)`）确定。
- **相互确定**：多个开放的 `let` 可以相互确定（`let a = List::new(); let b = List::new(); a.push(b); b.push(1);` → `b: List<i32>`，`a: List<List<i32>>`）。
- **字面量的默认类型**：`i64` / `f64` 让位于之后同类（整数 / 实数）的具体使用。在 `ys.push(1); sum_i32(ys)` 中，`ys` 是 `List<i32>`。由 `let` 自身绑定的字面量（`let c = 0`）也同样是开放的（[1.6.3](../001-lexical/106-numeric-literals.md#163-无后缀字面量)）。
- **冲突**：两个不同的使用不会构成和类型；这属于错误（`xs.push(none); xs.push(1)`，与 [5.1.7](../005-modules/501-variable-declaration.md#517-类型标注与类型推断) 的分支规则相同）。单独的 `let v = none;` 仍然是错误。
- 没有上下文的闭包参数（[4.5.2](../004-functions/405-closure.md)）和方法路径值（[3.5.3](../003-expressions/305-member-access-and-method-call.md#方法路径值)）同样由之后的调用确定（`let f = |x| x * 2; f(4)`、`let f = Box::get; f(b)`）。

诊断：

| 情况                 | 位置                                                      | 示例消息                                                                   |
|----------------------|-----------------------------------------------------------|----------------------------------------------------------------------------|
| 始终无法确定         | 该 `let`                                                  | `` type annotations needed: cannot infer the type of `d` (`Dict<_, _>`) `` |
| 冲突                 | 使用处，确定类型的位置作为次要标签                        | `` conflicting uses of `c` (`List<_>`): `str` here, but `i32` fixed at 8:5 `` |
| 字面量被当作浮点数使用 | 使用处，声明作为次要标签                                | `` `t` is used as `f64` here, but it holds an integer (`i64`) from `0` at 3:13 ``，help `` make it a float at the declaration: `let t = 0.0;` `` |

```nabi
let xs = empty();                   // still List<?>
takes_ints(xs);                     // T = i32 here (parameter List<i32>)
let zs = List::new();
zs.push(1);                         // zs: List<i32>, from a later method argument
let ys = empty();                   // ERROR: type annotations needed (no concrete use)
```

没有实参的静态 trait 方法以同样方式确定 `Self`。对类型参数的调用 `T::zero()` 和 trait 路径调用 `Tr::m()` 从期望类型获取 `Self`。

```nabi
let z: i32 = num::Zero::zero();     // Self = i32 (core::num::Zero)
fn sum<T: num::Zero + ops::Add<Output = T>>(xs: List<T>) -> T {
    let acc: T = T::zero();          // Self = T
    ...
}
```

**关联类型绑定**。带绑定的约束 `T: Add<U, Output = V>` 让 `V` 取它所选 impl 的关联类型。该 impl 由 `T` 和 trait 的实参（`U`）选定，因此只有在 `T` 和每个 trait 实参都从实参或期望类型得知之后，才读取该绑定；在此之前 `V` 保持开放。如果某个 trait 实参没有任何东西确定它，`V` 就无法确定（``type annotations needed``）：编译器从不在一个类型对该 trait 的多个 impl 之间猜测。

```nabi
fn plus<T, U, V>(a: T, b: U) -> V where T: Add<U, Output = V> { a + b }
plus(7.0, 0.5)        // U = f64 from 0.5, so Add<f64> for f64: V = f64
plus(3.0, 4.0i)       // U = Complex<f64>: V = Complex<f64>
```

泛型 `enum` 的变体构造器先从其实参确定实例，然后从类型参数的默认值确定，再从它所绑定的 `let` 推断变量确定；否则属于错误，``type annotations needed``。

```nabi
enum Pair<A, B> { One(A), Two(A, B) }
let p = Pair::Two(1, "a");          // Pair<i32, str>, from the arguments
```

## 2.10.3. trait 约束

约束说明参数必须满足的 trait。

```nabi
fn add_all<T: ops::Add<T>>(items: List<T>) -> T { ... }
fn print_all<T: Display>(items: List<T>) { ... }
fn sort_all<T: Ord + Clone>(ref items: List<T>) { ... }
```

单个约束写作 `Param: Trait`；多个约束用 `+` 连接。没有约束时，参数上只能使用所有类型都支持的操作：移动、`Reflect`、存入 `any`、与 `none` 构成和类型，以及 `is`（2.10.9）。

在约束、`where` 子句、`impl`、父 trait 列表或 `any Tr` 中提到的 trait 与其他名称一样解析：其他模块的 trait 需要 `use` 或路径（[5.5](../005-modules/505-use.md)）。否则会产生一个找不到名称的错误，并指出它写在哪里：``cannot find trait `Shape` (named in a bound)``、`(named in an impl)` 或 `(named as a supertrait)`，同时给出要添加的 `use`。

## 2.10.4. where 子句

约束变长时，最好把它们移到 `where` 子句中。

```nabi
fn complex<T, U, V>(a: T, b: U) -> V
where
    T: ops::Add<U, Output = V>,
    V: Display + Clone,
{
    ...
}
```

`where` 关键字位于参数声明和主体之间，后面跟着以逗号分隔的约束。也可以在这里写关联类型上的约束（例如 `T::Output: Foo`）。

出现在关联类型绑定中的类型参数（`I: Iterator<Item = (A, B)>`，或 `where` 子句中的 `Self: Iterator<Item = (A, B)>`）从实参（对 `Self` 而言是接收者）的该关联类型推断。绑定的类型可以是结构化的：`(A, B)`、`(A, List<B>)` 和 `fn(A) -> B` 会绑定其中包含的每个参数。如果关联类型是其他形式，该调用就是错误，错误信息会说明元素必须是什么样子：``` `unzip` needs `Item` of the form `(A, B)`, but `ListIter<i32>` has `Item = i32` ```。

```nabi
fn unzip2<I: Iterator<Item = (A, B)>, A, B>(it: I) -> (List<A>, List<B>) { ... }
unzip2([(1, "a"), (2, "b")].iter());   // A = i32, B = str
```

类型别名的 `where` 子句（[2.6.9](206-sum-types.md#269-与类型别名组合)）在别名的每一次使用处都成立；类型参数通过该使用处作用域中的约束来满足它。

## 2.10.5. 默认值

类型参数可以有默认值。

```nabi
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

没有默认值的参数不能跟在有默认值的参数之后；默认值必须位于末尾。

在类型位置，省略了有默认值的实参的泛型类型就是默认实例（`x: Vec3` 即 `x: Vec3<f32>`）。在表达式路径中（`Vec3::new(..)`、`Vec3 { .. }`、`Vec3::X`、`zero()`），省略的实参与其他实参一样推断：根据实参、期望类型和之后的使用；只有在没有其他东西确定它时才应用其默认值，填入它的无后缀字面量把该默认值作为自己的默认类型（[1.6.3](../001-lexical/106-numeric-literals.md#163-无后缀字面量)）。通过这样的路径读取的关联项从期望类型获取实例，否则从定义它的唯一 impl 获取，否则取默认值：`let v: DVec3 = Vec3::X` 读取 `Vec3<f64>` 的 `X`；`let v = Vec3::X` 读取 `Vec3<f32>` 的 `X`。函数中不是数值类型的默认值（`C = List<Self::Item>`）在调用处取用。

```nabi
struct Vec3<T = f32> { x: T; y: T; z: T; }
fn zero<T: Num = f32>() -> T { T::zero() }

let a = Vec3::new(1.0, 2.0, 3.0);    // Vec3<f32>: the literals take T's default
let b: Vec3<f64> = a;                // a later use fixes it first: a is Vec3<f64>
let r = zero();                      // f32 unless a later use says otherwise
takes_f64(r);                        // r is f64
```

更改公开类型参数的默认值，会改变使用它的代码中推断出的类型。

在显示类型的地方（诊断、帮助文本、`type_name`），省略与其参数默认值相等的末尾实参：对于 `struct W<T = i32>`，`W<i32>` 显示为 `W`，`Dict<str, i32, RandomState>` 显示为 `Dict<str, i32>`。

### 实参个数

泛型 `struct`、`union`、`enum` 或类型别名接受的实参数最多等于其参数个数，最少等于其没有默认值的参数个数。其他情况都是错误，错误信息会指出该声明：

```nabi
struct P<T, U> { a: T; b: U; }
struct D<T, U = i32> { a: T; b: U; }

let p: P<i64> = P { a: 1, b: 2 };          // ERROR: `P` takes 2 generic arguments but 1 was given
                                           //   help: `P` is declared `P<T, U>`; add one for `U`
let d: D<i32, i32, i32> = D { a: 1, b: 2 }; // ERROR: `D` takes 1 to 2 generic arguments but 3 were given
let e: D<i32> = D { a: 1, b: 2 };          // OK: `U` is `i32`
```

函数或方法可以给出比其参数少的显式实参（`f::<i32>(..)`）；其余的会被推断。给出更多实参是同样的错误，给 trait 约束提供多于该 trait 参数个数的实参也是（`` `Tr` takes at most 1 generic argument but 2 were given ``）。写在枚举路径上的实参指明其实例：`E::<i64>::A(1)` 是一个 `E<i64>`。

在期望类型的地方写出不带实参的泛型类型（对 `struct S<T>` 写 `x: S`），当有参数没有默认值时属于错误（所有参数都有默认值的类型可以裸写）：`` `S` needs generic arguments ``，help `` `S` is declared `S<T>`; write it with them, e.g. `S<i32>` ``。

## 2.10.6. 单态化

泛型调用和实例化为每一组具体类型元组产生单独的编译结果。

```nabi
fn id<T>(x: T) -> T { x }

let a = id(42i32);          // id<i32> is generated: an i32 → i32 function
let b = id(3.14f64);        // id<f64> is generated: an f64 → f64 function
let c = id("hello");         // id<str> is generated
```

类型元组相同的调用共享同一个结果，因此只编译一次。其性能与直接调用相同，没有间接调用的开销。代价是编译时间和二进制文件大小，这大部分被丢弃不可达的实例以及链接时的死代码剥离所抵消。

### 单态化缓存模型

[AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 构建收集调用处，并一次性[特化](../000-meta/007-glossary.md#specialize)它们。在 [JIT](../000-meta/007-glossary.md#jit-just-in-time) 模式下，特化在某个类型元组第一次被调用时发生，结果会被缓存。两种模式共享同一套特化机制，它是[执行引擎与用户界面](../000-meta/001-execution-modes.md)中公共流水线的一部分。

## 2.10.7. 与渐进类型的结合

[`any` 槽](202-any-slot.md)中描述的函数参数的渐进性按如下方式处理。

```nabi
fn compute(x, y) { ... }
```

这段代码被转换为隐式泛型 `fn compute<T, U>(x: T, y: U)`。不会合成任何约束（[4.2.2](../004-functions/402-parameters.md#422-省略类型)）：每个实例在其调用处单态化，其主体也在那里检查，错误同样会指向该调用。

如果在参数上显式写了 `any`，它就不会被单态化，而是成为运行时的动态槽。这个区别是渐进模式的核心。

## 2.10.8. 泛型参数的一致性

一个泛型参数在一次调用中只表示一个类型。它在不同的调用中可以表示不同的类型，但在同一次调用中其类型不能改变。

```nabi
fn pair<T>(a: T, b: T) -> (T, T) { (a, b) }

let p = pair(1i32, 2i32);     // OK: T = i32 for both arguments
let q = pair(1i32, 3.14f64);  // ERROR: ambiguous whether T is i32 or f64
```

多个参数（`pair<A, B>`）各自独立推断。

## 2.10.9. 主体检查

泛型函数的主体只能使用通过其约束得知的行为。

```nabi
fn double<T>(x: T) -> T {
    x + x       // ERROR: T has no Add bound
}

fn double<T: ops::Add<T, Output = T>>(x: T) -> T {
    x + x       // OK
}
```

由于这条规则，泛型主体的检查独立于调用处，因此无论调用传入什么具体类型，主体都保证能够编译。这条规则扩展如下。

- trait 默认方法的主体是泛型主体：`Self` 是不透明的，并受该 trait 约束；trait 的参数是不透明的。
- 每个类型，无论是否有约束，都支持移动、`Reflect`（[6.10.1](../006-meta-system/610-reflection.md#6101-reflect自动实现的-trait)）、存入 `any`、与 `none` 构成和类型，以及 `is`。
- 带有任何隐式参数的函数（[4.2.2](../004-functions/402-parameters.md#422-省略类型)）整体豁免。
- const 参数在其值未知的情况下检查；当类型的 const 表达式作为值相等时，类型相等（[2.11.5](211-const-generics.md#2115-const-表达式中的算术)）。
- 非参数类型上的 `where` 子句在主体中被假定成立。
- 未通过检查的主体的实例不会再增加它自己的错误。
- 对类型涉及类型参数的值进行的类型测试（`is`、类型模式）按可反驳的方式检查，并按实例判定。
- 对参数本身的类型测试（`if T is f32 { .. }`，2.10.13）把 `T` 替换为被测试的类型，对其分支检查一次；每个实例只降级其类型所走的分支。

## 2.10.10. 泛型与 trait 作类型

泛型是编译期多态，通过单态化直接调用。

trait 作类型是使用虚表分派的运行时多态，Nabi 在[类型系统概述](README.md)中禁止它。

因此，只要需要多态，就使用泛型（本文档）或和类型（[和类型](206-sum-types.md)）。

## 2.10.11. 型变

Nabi 在泛型实例之间没有子类型关系：每个类型参数都是不变的。`List<i64>` 不是 `List<i64 | none>`，`Box<Cat>` 不是 `Box<any>`，`fn(i64 | none)` 也不是 `fn(i64)`，尽管 `i64` 值可以放入 `i64 | none` 槽。向和类型的扩展（[2.6.4](206-sum-types.md)）和向 `any` 的扩展（[2.2](202-any-slot.md)）作用于整个值，此时编译器可以包装它；它从不深入泛型类型内部，因为 `List<i64>` 和 `List<i64 | none>` 以不同的布局保存元素，并在其所有句柄之间共享同一个缓冲区。要得到更宽的类型，就在创建值的地方写出它（`let xs: List<i64 | none> = [1, 2]`），或者把元素复制到更宽类型的新集合中（`let ys: List<i64 | none> = []; for x in xs { ys.push(x) }`）。

唯一的放宽在函数约束上：`fn(..) -> R + Send` 值可以放入 `fn(..) -> R` 槽（[4.4.9](../004-functions/404-function-type.md#449-函数类型之间的子类型关系不存在)）。

## 2.10.12. 推断与默认值一览

类型推断局限于函数主体内；签名总是要写出。各条规则与其所涉及的构造放在一起；这个列表把它们汇总起来。

- **泛型实参**来自调用的实参类型和期望类型（2.10.2，“类型实参推断”）；`const` 实参同理（[2.11.6](211-const-generics.md#2116-参数推断)）。没有任何东西确定某个实参时，用 `::<..>` 写出它；省略它属于错误。
- **未注解的 `let`** 从其初始化表达式获取类型，初始化表达式留下的开放部分由其作用域中之后的使用确定（2.10.2；[5.1.7](../005-modules/501-variable-declaration.md#517-类型标注与类型推断)）。两个不一致的使用属于错误；它们不会构成和类型。
- **和类型**按集合匹配：`T | none` 与 `i32 | str | none` 匹配得到 `T = i32 | str`（[2.6.4](206-sum-types.md#264-集合语义)）。分支结果不会被扩展为新的和类型；分支类型不同属于错误，除非期望类型是和类型，或者每个分支类型都是某一分支的和类型的成员。
- **字面量默认值**：没有期望类型的整数字面量是 `i64`，浮点字面量是 `f64`（[1.6.3](../001-lexical/106-numeric-literals.md#163-无后缀字面量)），除非它填入一个声明了默认值的参数；之后同类（整数、实数）的使用会替换该默认值（2.10.2）。
- **类型参数的默认值**（`Rhs = Self`，2.10.5）只在参数未写出且没有任何东西推断它时适用；填入该参数的字面量把默认值作为自己的默认类型（[1.6.3](../001-lexical/106-numeric-literals.md#163-无后缀字面量)），因此顺序是：写出的实参，然后是期望类型和使用，然后是参数的默认值，最后是 `i64` / `f64`。
- **闭包**没有参数类型时，从期望的 `fn` 类型获取，否则从之后的调用获取（[4.5.2](../004-functions/405-closure.md)）。

其他一切都没有默认值：不会回退到 `any`、`()` 或和类型。

## 2.10.13. 对类型参数的类型测试

`T is X`（其中 `T` 指作用域中的类型参数，或 trait 默认方法、impl 中的 `Self`）测试类型本身，不涉及值。它在每个实例中都是常量 `bool`：当该实例的 `T` 的每个值都通过 `v is X` 时为真。对单个类型 `X`，这意味着 `T` 就是 `X`；对和类型 `X`，意味着 `T` 是 `X` 的成员或其子和类型（[2.6.4](206-sum-types.md#264-集合语义)）。`T = any` 只通过 `T is any`。

```nabi
fn sum<T: num::Num + Clone>(xs: List<T>) -> T {
    if T is f32 {
        return sum_f32(xs);          // xs: List<f32>; the f32 result is a T
    } else if T is f64 {
        return sum_f64(xs);
    }
    let acc = T::zero();             // generic fallback, checked against the bounds
    for x in xs { acc = acc + x; }
    acc
}
```

- **细化**。在 `X` 是单个类型（不是和类型，也不是 `any`）的 `if T is X { B }` 中，`B` 在定义处检查一次，作用域中各处的 `T` 都替换为 `X`：参数、在分支外声明的局部变量、分支中的闭包以及返回类型。在 `B` 中，`xs: List<T>` 是 `List<f32>`，`f32` 是 `T`。类型测试的 `&&` 链（`if T is f32 && U is i32`）细化每个参数；其他条件也可以加入该链。和类型 `X` 不细化任何东西。
- **其他位置不细化**。`else` 分支、`if` 之后的代码、类型测试的 `||` 以及取反的测试（`!(T is f32)`）都按普通的泛型规则检查。如果类型测试的 `||` 的分支需要 `T` 是某一个类型，就属于错误，错误信息会提示为每个类型各写一个分支。
- **按实例**。实例只降级其类型所走的分支：另一个分支永远不会为它降级，因此在那里可以是类型错误的。快速路径在运行时没有测试开销，也没有 `any` 往返。
- **不绑定**。`is` 之后的类型指的是封闭类型或作用域中已有的参数；在没有声明 `U` 的情况下写 `T is List<U>` 是错误，`T is v: X` 也是错误：没有可以绑定的值。
- **作为值**。`let fast = T is f32;` 是每个实例上的 `bool` 常量；只有 `if` 条件才进行细化。
- 类型用 `is` 比较；`T == f32` 是错误，修复建议为 `is` 形式。
- 值测试与类型测试不同：当 `T = i32 | str` 时，`x is i32` 问的是这个值是否为 `i32`，而 `T is i32` 为假，因为并非 `T` 的每个值都是 `i32`。

类型集合开放时，使用 trait。保留泛型回退的快速路径，使用 `if T is`：回退分支会 panic 或为 `unreachable` 的链是一个封闭的类型集合，`closed_type_switch` lint 会建议改用 trait，为每个类型各写一个 impl。与覆盖实现重叠的具体 impl（E0075）写成覆盖实现中的 `if Self is X` 分支。
