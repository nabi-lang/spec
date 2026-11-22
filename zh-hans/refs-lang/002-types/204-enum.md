# 2.4. `enum`

`enum` 定义一种类型，其值是若干变体之一。它与和类型相似，区别在于它是具名的容器。

## 2.4.1. 基本形式

`enum` 的语法如下：

```
enum_decl   ::= visibility? "enum" identifier generics? where_clause? "{" variant_list? "}"
variant_list ::= variant ("," variant)* ","?
variant     ::= identifier                                  // unit
              | identifier "(" type_expr ("," type_expr)* ","? ")"  // tuple
              | identifier "{" named_field_list? "}"        // struct
              | identifier "=" const_expr                   // discriminant
```

最简单的形式如下：

```nabi
enum Direction {
    North,
    East,
    South,
    West,
}
```

主体用 `{ ... }` 括起，变体之间用逗号分隔。允许尾随逗号。

## 2.4.2. 四种变体

`enum` 变体按其持有数据的方式分为四种。

### 单元变体：无数据

最简单的单元变体不持有数据。

```nabi
enum Color {
    Red,
    Green,
    Blue,
}
```

它只由一个不带数据的标签构成。

### 元组变体：位置字段

元组变体具有位置字段。

```nabi
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}
```

它像 [`struct`](203-struct.md) 中的元组 `struct` 那样具有位置字段。

### 具名变体：具名字段

具名变体具有具名字段。

```nabi
enum Event {
    Click { x: i32; y: i32; },
    Scroll { dy: i32; },
    Resize { w: i32; h: i32; },
}
```

每个变体都有形如具名 `struct` 的字段。字段以 `;` 结尾，与 [`struct`](203-struct.md) 一致。

### 值变体：显式整数值

最后，值变体直接为变体指定一个整数值。

```nabi
enum Status {
    Ok = 0,
    NotFound = 404,
    InternalError = 500,
}
```

指定的值必须是 `const` 整数表达式。两个变体不得具有相同的值，并且每个值都必须能放入标签类型（[6.2.2](../006-meta-system/602-built-in-attributes.md#622-repr-显式内存表示)）。

这种形式用于 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 兼容、映射外部协议、位标志等。值变体不能与其他种类的变体混用：只要有一个变体是值变体，整个 `enum` 就必须只由值变体组成。

### 强制转换为整数

不带载荷的 `enum`（只有单元变体，或只有值变体）可以用 `as` 强制转换为整数。结果是变体的值；对于单元变体，结果是其标签（从 0 开始的声明序号）。

```nabi
let code = Status::NotFound as i32;   // 404
```

## 2.4.3. 混用变体种类

单元变体、元组变体和具名变体可以在一个 `enum` 中自由混用。如上所述，值变体只能单独使用。

下面的示例同时使用了这三种变体。

```nabi
enum Result {
    Ok(i32),                    // tuple
    Err { code: i32; msg: str; }, // named
    Pending,                    // unit
}
```

## 2.4.4. 可见性

变体没有自己的可见性；它们遵循 `enum` 的可见性。`pub enum` 的所有变体都是公开的（[5.6.9](../005-modules/506-visibility.md#569-enum-变体的可见性)）。

## 2.4.5. 内存布局

默认布局交给编译器决定：编译器可以选择标签与载荷的对齐和紧凑方式。这种自由使 [niche 优化](207-niche-optimization.md)成为可能。

如果需要 C ABI 兼容，可以用 `#[repr(...)]` 属性强制指定表示。其确切含义在[内置属性](../006-meta-system/602-built-in-attributes.md)中定义。

## 2.4.6. 同一模块中不得重复定义

在同一模块中声明两个同名 `enum` 是编译错误。同一 `enum` 内的变体名重复也是错误。

## 2.4.7. `enum`、和类型与 `union`：预览

这三种形式的比较如下。

| 形式              | 命名方式                                       | 标签 | 变体是否共享内存？                 |
|-------------------|------------------------------------------------|------|------------------------------------|
| `enum`            | 在容器内定义的具名变体                         | 有   | 否（各自的载荷）                   |
| 和类型（`A \| B`） | 变体是已有的类型                              | 有   | 否（内联的标签 + 载荷）            |
| `union`           | 无标签；同一块内存按多种类型读取               | 无   | 是（有意重叠）                     |

详细差异在各容器的文档中定义。

## 2.4.8. 由期望类型确定的裸变体

变体可以带其枚举的路径书写（`Ordering::Less`、`Shape::Circle`），当所在位置的期望类型能选出该枚举时，也可以裸写。普通查找无法解析的裸名称（[5.7.1](../005-modules/507-paths.md#571-路径的起始标识符)），在期望类型是带有变体 `V` 的枚举 `E`，或是恰好只有一个枚举成员带有变体 `V` 的和类型时，表示变体 `V`。这适用于任何枚举，无论是用户的还是标准库的，都不需要 `use`。

期望类型来自：

- 带类型的 `let`；
- 参数（包括方法的参数）以及参数的默认值；
- `return` 和函数的尾表达式；
- 结构体字面量中的结构体字段；
- 元素类型已知的列表或字典字面量中的元素；
- 左侧类型为 `E` 的 `==` / `!=` 的右侧；
- 在类型为 `E` 的被匹配值上，`match`、`if let` 和 `let` 的模式（[3.11](../003-expressions/311-patterns.md)）。

```nabi
enum Shape { Circle, Square, Tri(i32) }

fn area(s: Shape = Square) -> i32 {
    match s { Circle => 1, Square => 2, Tri(n) => n }
}

fn main() {
    let s: Shape = Tri(3);
    let o: Ordering = Less;
    let n = Atomic::new(0);
    n.store(1, SeqCst);                     // StoreOrdering::SeqCst
    let lt = 1.cmp(2) == Less;
    println(f"{area()} {area(Circle)} {lt}");
}
```

局部变量、项和 `use` 优先：期望类型只是对其他方式都无法解析的名称的后备，因此 core 枚举后来新增的变体永远不会抢占用户的名称。泛型枚举的实例取期望类型的实例（在 `-> Result<i32, str>` 函数中，`Ok(1)` 是 `Result<i32, str>::Ok`）。

没有期望类型时，裸变体是错误，错误会用最短路径指出它：

```nabi
let o = Less;   // ERROR: `Less` is not in scope: it is the variant `Ordering::Less`
                // help: write `Ordering::Less`
```
