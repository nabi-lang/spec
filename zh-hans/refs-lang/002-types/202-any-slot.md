# 2.2. `any` 槽

`any` 是表示可变槽的类型名，这种槽可以持有任何类型的值（它是上下文关键字，只在类型位置中是关键字，[1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)；声明名为 `any` 的类型是错误）。它是 Nabi 渐进类型的核心概念。

> [!IMPORTANT]
> `any` 槽只存在于写了 `any` 的地方。省略类型注解的变量会成为静态槽，其类型由初始化表达式的类型推断而来（[5.1.7](../005-modules/501-variable-declaration.md#517-type-annotations-and-type-inference)）。Nabi 的渐进类型由一行规则决定：“默认静态，只有写了 `any` 的地方才是动态的。”

## 2.2.1. 与静态槽的对比

首先看静态槽与 `any` 槽的区别：

```nabi
let x: i32 = 42i32;     // x is i32; cannot hold a value of another type
let y: any = 42i32;     // y is an any slot; any type allowed
let z = 42i32;          // z is i32; static slot inferred from the initializer
```

在这三行中，`x` 和 `z` 的含义相同；`y` 是唯一的 `any` 槽。省略类型注解时，编译器取初始化表达式的类型作为槽的类型。这样，REPL 和笔记本用户不写注解也能快速输入，同时仍然得到静态分派；而产生动态分派开销的地方在声明语法中一目了然。

## 2.2.2. 槽的内存表示

`any` 槽中的值表示为一个标签与一个载荷的组合。

```
+------+----------+
| tag  | payload  |
+------+----------+
```

标签标识载荷是哪种类型；载荷就是值本身：小值（基本类型等）直接存储，大值存储为指向堆对象的引用。

标签和载荷的确切位宽、对齐与编码在 [8. 运行时 ABI](../008-abi/README.md) 中定义；本文档只保证其形状（两部分：标签 + 载荷）。

## 2.2.3. 动态分派

对 `any` 槽中的值施加运算时，运行时检查标签并调用相应的 trait 方法。

```nabi
let v: any = 42i32;
let w = v + 1i32;           // at runtime, looks at v's tag and calls i32::add
```

这种分派的开销明显高于静态调用，因此对性能要求高的代码应当使用静态槽。

### `any` 上的运算符

对 `any` 操作数施加运算符时，规则如下：

- 候选是具有该运算符的基本类型，并且两个操作数必须是同一类型。
- 如果一个操作数具有静态类型，该类型把候选固定为一个。如果该静态操作数不是基本类型，则是编译错误。
- 不带后缀的字面量操作数不是静态类型：当它能取另一个操作数运行时标签的类型时，就取该类型（当 `v` 持有 `i32` 时，`v + 1` 中的 `1` 是 `i32`；持有 `f32` 时是 `f32`）。如果标签是该字面量不能取的类型（对 `u8` 的 `300`、对 `i32` 的 `0.5`），程序会 panic，并同时指出两者（`` `+` on an `any` holding `u8`: `300` does not fit `u8` ``）。
- 如果操作数类型在运行时不匹配，程序会 panic（退出码 101）。
- 如果每个候选的结果都是 `bool`，结果就是 `bool`；否则结果是装箱的 `any`。

```nabi
let v: any = 42i32;
let w = v + 1i32;           // w: any (i32 + i32)
let b = v == 1i32;          // b: bool
let c = v + 1;              // 1 is i32 here: it takes the tag's type
let p = v + Point {};       // ERROR: Point is not a primitive type
```

## 2.2.4. 重新赋值：可以改变类型

`any` 槽可以自由地用不同类型的值重新赋值。

```nabi
let s: any = 42i32;
s = "hello";        // OK: the slot is any
s = 3.14f64;        // OK
```

相比之下，静态槽只允许用相同类型重新赋值。

```nabi
let s: i32 = 42i32;
s = 10i32;          // OK
s = "hello";        // ERROR
```

## 2.2.5. 检查槽的当前类型

`any` 槽中值的当前类型可以用后缀运算符 `is` 检查。`e is T` 的结果是 `bool`，其优先级与 `as` 相同（[3.2.3](../003-expressions/302-precedence-and-associativity.md#323-precedence-and-associativity-table)）。

```nabi
if v is i32 { ... }
```

检查是一次运行时标签比较。在 `any` 上，当存储的值是 `A` 或 `B` 时，以及存储的是整个和类型 `A | B` 的值时，`is (A | B)` 和模式 `x: A | B` 都匹配。绑定会被重新标记为和类型 `A | B`。如果操作数的静态类型既不是 `any` 也不是和类型，类型检查 `e is T` 是编译期常量。没有被之前任何用法固定的无后缀字面量，在那里取其默认类型（[1.6.3](../001-lexical/106-numeric-literals.md#163-unsuffixed-literals)）：在 `let v = 3;` 之后，`v is i64` 为 `true`。目标为和类型时，它检验成员关系：对普通类型 `A` 的值，`e is (A | B)` 为 `true`。

对静态类型为 `enum` 的值，`e is Enum::Variant` 检查变体。它是一次运行时变体比较，只要是该变体，无论载荷如何都为 `true`。它不收窄类型；需要载荷时，使用 `match` 或 `if let`。

```nabi
if shape is Shape::Circle { count += 1; }        // checks only whether it is Circle(_)
```

## 2.2.6. 类型收窄

在 `is` 检查或模式匹配成功的分支内，编译器把槽类型视为收窄到所检查的类型，因此在该分支内可以进行静态分派。

```nabi
let v: any = ...;
if v is i32 {
    let w = v + 1i32;       // inside this branch v is i32: static dispatch
}
```

收窄规则如下：

- 收窄条件是 `x is T`，其中 `x` 必须是局部变量。它可以是整个条件，也可以是 `a && b && ..` 链中的一项。对可选的 `x`（含 `none` 的和类型），`x != none` 和 `!(x is none)` 把 `x` 收窄为除 `none` 以外的成员。
- 在后续的 `&&` 项、`if` 的 `then` 块以及 `while` 的主体中，`x` 是 `T`。
- `x == none`（或 `x is none`）作为整个条件或 `a || b || ..` 链中的一项时，在 `else` 块中把 `x` 收窄为除 `none` 以外的成员。
- 对类型在其他成员之外还含有 `none` 的任何值，`x == none`、`x != none`、`none == x` 和 `none != x` 都是 `none` 检验，与 `x is none` 相同：其他成员不需要 `PartialEq`，所以 `fn has<T>(v: T | none) -> bool { v != none }` 不需要对 `T` 加约束。对不含 `none` 的类型，或 `none` 类型本身，这种比较是错误（E0054）。
- 在一个 `if` 语句之后，如果它的某个分支总是离开（`return`、`break`、`continue`、`throw`、`!` 调用），另一个分支的收窄在外围块的剩余部分中保持有效：`if x == none { return; }` 使其后的 `x` 不为 `none`。
- 除此以外，在 `||` 或 `!` 之下、在 `else` 块中，以及对字段路径（`p.x is T`），都不发生收窄。
- 变体检查 `x is Enum::Variant`（2.2.5）不收窄。
- 当操作数的静态类型是被检查和类型的普通成员 `A` 时，`x is (A | B)` 不收窄；`x` 保持类型 `A`。
- 当 `x` 被 `[ref x]` 捕获（[4.6.2](../004-functions/406-capture-semantics.md#462-reference-capture-the-ref-x-capture-list)）时，在收窄作用域中每次读取 `x` 都会再次检查共享单元；如果某个闭包存入了不是 `T` 的值，读取会 panic。
- 在收窄作用域内，`x` 的类型是 `T`。对 `x`（或其字段 / 元素）的赋值会转换回去并写入原来的槽；作为 `ref` 实参传出的 `x`（或其字段）的最终值，在调用返回时也是如此。赋予一个属于声明类型但不是 `T` 的值，会在该作用域的剩余部分结束收窄：`x` 重新具有其声明类型，正常的流类型继续进行。赋予声明类型以外的值仍是类型错误。

```nabi
let v: any = 1i32;
if v is i32 && v > 0i32 {      // v: i32 in the second term
    v = v + 1i32;              // written to the original any slot
    v = "x";                   // ends the narrowing: v is any again
}
if v is i32 || flag { ... }    // no narrowing

fn first(xs: List<i32>) -> i32 {
    let x = xs.pop();          // i32 | none
    if x == none { return 0; }
    x + 1                      // x: i32 for the rest of the block
}
```

关于用模式进行收窄，请见[模式](../003-expressions/311-patterns.md)中的类型检查模式。

## 2.2.7. 渐进类型的边界

`any` 槽只存在于写了 `any` 的地方。省略类型注解的含义按位置分别定义如下。

### 没有类型注解的变量

```nabi
let n = 42i32;          // n: i32
let s = "hi";           // s: the string type chosen by the literal handler
let n2: any = 42i32;    // dynamic slot only when written explicitly
```

`let` 遵循局部类型推断。初始化表达式的类型成为槽的类型，之后的重新赋值都按它检查。无法从初始化表达式确定类型时的规则在 [5.1.7](../005-modules/501-variable-declaration.md#517-type-annotations-and-type-inference) 中定义。

### 没有类型注解的函数参数

```nabi
fn compute(x, y) { ... }
```

这是一个泛型函数。与变量一样，它遵循“省略的类型 = 由编译器静态决定”的规则。它按调用处的类型进行单态化：如果一个调用者传入两个 `i32`，那次调用就编译为接受两个 `i32` 的函数；如果另一个调用者传入两个 `f64`，就编译出另一个函数。

要让参数成为动态槽，必须显式写出 `any`。

```nabi
fn compute(x: any, y: any) { ... }    // dynamic slots
```

### 结构体字段

结构体字段必须写出类型。

```nabi
struct Point { x: f64; y: f64; }      // OK
struct Mixed { name; value; }          // ERROR: missing field types
```

要让字段成为动态的，同样使用显式的 `any`。

```nabi
struct Cell { tag: i32; payload: any; }   // OK
```

这一要求使编译器能够确定结构体的内存布局。

## 2.2.8. 槽的种类由声明固定

变量的槽是动态的还是静态的，单看声明语法就能知道。写 `: any` 构成动态槽；其他情况（显式或推断的类型）构成静态槽。槽的种类一旦固定就不再改变。重新声明该名称的 `let` 会创建一个新槽，新槽有自己的种类（[5.1.4](../005-modules/501-variable-declaration.md#514-redeclaration-in-the-same-scope)）；旧槽保持其种类。

```nabi
let x = 1;              // static slot (inferred from initializer)
x = "hi";               // ERROR: x is a static slot
let x: any = 5i32;      // a new binding: a dynamic slot
x = "hi";               // OK
```

细节在[变量声明](../005-modules/501-variable-declaration.md)中定义。

## 2.2.9. 对编译器与工具的友好性

带有 `any` 槽的代码类型检查较弱。编译器无法预先知道槽的实际类型，因此工具（LSP、静态分析）只能在这一限制内尽力而为。渐进与静态之间的选择是用户的显式决定，体现为是否写了 `any`，而规范完全支持这两种模式。

## 2.2.10. 受约束的 `any`：`any Trait`

在 `any` 之后写一个 trait，构成只持有实现了该 trait 的类型的值的动态槽。

```nabi
let e: any Error = ParseError::new(s, 0, "empty");
let shapes: List<any Shape> = [Circle { r: 1.0 }, Square { side: 2.0 }];
let item: any (Display + Hash) = 42i32;
```

共有三种形式：单独的 `any`、带一个 trait 的 `any Trait`，以及在括号中带多个 trait 的 `any (A + B)`。单独的 `any` 等同于没有约束的 `any ()`（语法：[类型表达式](201-type-expressions.md)中的 `any_type`）。`any` 只绑定紧随其后的那一个 trait，所以 `any Error | none` 是 `(any Error) | none`。

在类型位置中写单独的 trait 名仍然是禁止的（[trait 不能作类型](212-trait-as-type-banned.md)）。之所以允许 `any Trait`，是因为动态分派的开销由 `any` 关键字显式标出。

### 存入时的检查

把值存入 `any Tr` 槽时，编译器在编译期检查该值的静态类型是否实现了 `Tr`。`any (A + B)` 要求同时实现 `A` 和 `B`。

```nabi
let a: any Display = 42i32;          // OK: i32: Display
let b: any Display = Opaque {};      // ERROR: Opaque does not implement Display

let v: any = 42i32;
let c: any Display = v;              // ERROR: no implicit any → any Display conversion
if v is i32 {
    let d: any Display = v;          // OK: after narrowing, the static type is i32
}
```

`any` 类型之间的转换只允许在放宽的方向上进行。

- 当 `B` 的每个约束都被 `A` 的某个约束覆盖时，允许 `any A` → `any B`。覆盖指同一个 trait，或沿父 trait 可达（可经多步）。`Tr<args>` 覆盖 `Tr`。
- `any X` → 单独的 `any` 总是允许的。
- 其他任何方向（如 `any` → `any Tr` 这样的收窄转换）都是错误。这避免了隐藏运行时检查的隐式转换；应先用 `is` 检查或类型模式取出具体类型，然后再存入。

```nabi
let e: any Error = ...;
let d: any Display = e;              // OK: Display is a supertrait of Error
let x: any = e;                      // OK
let e2: any Error = x;               // ERROR: narrowing conversion
```

### 使用

在 `any Tr` 值上，只能调用 `Tr` 及其父 trait 的方法，调用在运行时按标签分派。不过，如果一个方法的签名（不含 `self`）提及以下任何一项，就不能调用它：

- `Self`
- 未被约束的实参固定的 trait 类型参数
- 未被约束的 `X = ..` 绑定固定的关联类型

约束可以省略实参和绑定；省略的部分视为不受约束。对 `any Tr<A, X = B>` 的调用，按代入 `A` 和 `B` 后的 trait 声明签名检查。因此，即使 `Tr` 只由泛型实现来实现，也可以调用。例如，在 `any Iterator` 上不能调用 `next`（`Item` 未固定），但在 `any Iterator<Item = i32>` 上可以。开销模型与 2.2.3 中的 `any` 相同。

分派覆盖程序存入 `any` 槽的那些类型。加载插件（[9.12](../009-tools/912-plugins.md)）的程序还会接收只有插件知道的类型的值：对于分派不认识的标签，它调用存入该值的模块所注册的方法体。不加载任何插件的程序没有这一步。

具有自身类型参数的方法（`fn write<B: AsBytes>(ref self, buf: B) -> usize`）也可以调用。每次调用都像泛型调用那样固定这些参数：先看 `::<..>`，其次看实参类型，再次看结果的期望类型；任何一处都不能固定的参数是错误，错误会要求写出 `::<..>`。其约束在调用处检查。用到的每个实例化都有自己针对各实现者的分派，每个分派调用为这些类型单态化的实现者方法。

```nabi
fn log(w: any Write) throws (IoError) {
    try w.write("line\n");                 // B = str
    try w.write([0u8, 1]);                  // B = [u8; 2]
}
```

要使用其他方法或字段，按 2.2.5-2.2.6 用 `is` 检查或类型模式收窄到具体类型。

```nabi
fn describe(e: any Error) -> str {
    let msg = f"{e}";                       // Display (supertrait of Error)
    match e {
        p: ParseError => f"{msg} at {p.pos}",   // narrowed to the concrete type
        _ => msg,
    }
}
```

### 表示

`any Tr` 值与 `any` 具有相同的标签 + 载荷表示（2.2.2）。trait 方法的分派表在链接时根据程序中存入 `any` 槽的所有类型构建（封闭世界，假定单包编译）。通过泛型实现或覆盖实现来实现该 trait 的类型也包括在内。因此，没有任何值携带自己的虚表指针。
