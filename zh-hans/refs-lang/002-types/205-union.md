# 2.5. `union`

`union` 是一种无标签的容器，其字段共享同一块内存。语言不跟踪在任一时刻哪个字段的值是有意义的；这是用户的责任。

```
union_decl      ::= "union" identifier generics? where_clause? "{" named_field_list? "}"
```

## 2.5.1. 基本形式

最简单的 `union` 声明如下：

```nabi
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

主体和字段语法与 [`struct`](203-struct.md) 中的具名 `struct` 相同。区别在于关键字 `union` 和语义：所有字段共享同一内存位置。

### 泛型联合体

`union` 可以有类型参数（[2.10](210-generics.md)），与 `struct` 一样可以带约束、`where` 子句和默认值。每个实例都是独立的类型，具有自己的布局：其大小和对齐取决于这些实参下的字段类型。

```nabi
union Bits<T> { v: T; raw: u64; }
union Small<T = u16> { v: T; raw: u8; }

let a = Bits::<f64> { raw: 0x3FF0000000000000u64 };
let b = Bits { v: 7i32 };              // Bits<i32>, inferred from the field given
let x = unsafe { a.v };                // 1.0 (the bits of 1.0f64)
```

在联合体字面量中，实参从给出的那一个字段推断。约束按实例检查，诊断会指出具体实例（``reading field `v` of union `Bits<i32>` needs an `unsafe` block``）。`union` 至少声明一个字段。

## 2.5.2. 语义

下面的示例展示了 `union` 的行为。

```nabi
let x = EpollData { fd: 5i32 };
x.fd = 7i32;                    // write: safe
let y = unsafe { x.val };       // read: requires unsafe; reinterprets x's bit pattern as u64
```

用某个字段创建实例，会把该字段的位写入内存。之后通过另一个字段读取，会把同样的内存位重新解释为那个字段的类型。写入字段是安全的，但读取字段只允许在 [`unsafe`](../004-functions/415-unsafe.md#4152-unsafe---块表达式) 内进行，因为语言不保证读到的位是该类型（例如 `bool`、`char`、指针）的有效值。

## 2.5.3. 安全责任在哪里

`union` 的位重新解释是语言定义的行为，但结果的含义由程序所在领域负责。例如，向 `f32` 字段写入 `1.0` 后读取 `i32` 字段，得到 `1065353216`（`0x3F800000`）；反之，把 `i32` 的位按 `f32` 读取，得到由 IEEE 754 位表示给出的实数值。语言不检查该值对领域是否有意义。

这一模型与 C 的 `union` 不同。在 C 中，从另一个成员读取是未定义行为；在 Nabi 中，它被定义为位重新解释。由于结果不保证是有效值，读取被标记为 `unsafe`。

## 2.5.4. 默认布局交给编译器

`union` 的默认布局交给编译器决定。它只需满足最大字段的大小和对齐；确切的字段对齐和填充由编译器决定。

如果需要与 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 兼容的 `union`，加上 `#[repr(C)]`。

```nabi
#[repr(C)]
union CUnion {
    a: i32;
    b: f32;
}
```

带 `#[repr(C)]` 的 `union` 与 C 的 `union` 位兼容，用于 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 接口。确切含义在[内置属性](../006-meta-system/602-built-in-attributes.md)中定义。

## 2.5.5. `union` 与 `#[repr(C)] union`

这两种形式的区别如下。

| 形式                         | 用于        | 布局              | 位兼容性                  |
|------------------------------|-------------|-------------------|---------------------------|
| `union T { ... }`            | 普通代码    | 交给编译器        | 只在 Nabi 内部保证        |
| `#[repr(C)] union T { ... }` | FFI         | 与 C ABI 相同     | 与 C 代码兼容             |

## 2.5.6. 可见性与重复定义

适用与 [`struct`](203-struct.md) 相同的规则。

> [!WARNING]
> 读取 `union` 字段是位重新解释，只允许在 `unsafe` 内进行（写入是安全的）。读到的位是否是该类型的有效值、对领域是否有意义，由作者负责。

## 2.5.7. `union` 与和类型

`union` 和和类型看起来可能相似，但它们是完全不同的工具。

```nabi
union  Foo { a: i32; b: f32; }    // same memory, no tag
type   Bar = i32 | f32;            // separate memory, tagged
```

各方面的区别：

| 方面                           | `union`                          | 和类型（`A \| B`）             |
|--------------------------------|----------------------------------|-------------------------------|
| 跟踪哪个字段处于活动状态       | 否                               | 由标签跟踪                    |
| 字段之间的转换                 | 位重新解释                       | 变体改变（重建值）            |
| 内存占用                       | 最大字段的大小                   | 标签 + 最大变体的大小         |
| 读取字段                       | 需要 `unsafe`（位重新解释）      | 安全，通过模式 / `is`         |

两者的用途完全不同：`union` 在位层面提供不同的解释，和类型在值层面表达不同的种类。不得将二者混淆。
