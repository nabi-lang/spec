# 2.3. `struct`

`struct` 是把若干字段组合成一个整体的容器语法，用于定义积类型。本章介绍 `struct` 可以采用的形式，以及结构体的各项规则：实例创建、默认值、展开、可见性和内存布局。

## 2.3.1. 三种形式

`struct` 声明遵循以下语法。

```
struct_decl     ::= visibility? "struct" identifier generics? where_clause? "{" named_field_list? "}"
                  | visibility? "struct" identifier generics? "(" tuple_field_list? ")" where_clause? ";"   // tuple struct: where comes after the field list
                  | visibility? "struct" identifier generics? where_clause? ";"                              // unit struct
named_field_list ::= named_field (";" named_field)* ";"?
named_field     ::= identifier ":" type_expr
tuple_field_list ::= type_expr ("," type_expr)* ","?
```

主体的形状决定了三种形式中的一种，下面逐一介绍。

### 具名 `struct`：具名字段

最常见的形式是具名 `struct`，其中每个字段都有名称。

```nabi
struct Point {
    x: f64;
    y: f64;
}

struct User {
    id: i64;
    name: str;
    age: i32;
}
```

规则：主体用 `{ ... }` 括起。每个字段写作 `name: Type;`，字段末尾的 `;` 与[语句终止符](../001-lexical/103-statement-terminator.md)是同一个终止符。字段类型是必需的，因为如 [`any` 槽](202-any-slot.md)中所规定，结构体字段不允许渐进类型。

### 元组 `struct`：位置字段

其次，元组 `struct` 只按位置标识字段，不带名称。

```nabi
struct Pair(i32, i32);
struct Wrapper(str);
struct Color(u8, u8, u8);
```

这种形式的主体用 `( ... )` 括起，每个元素只有类型，没有名称。字段通过 `.0`、`.1` 这样的位置索引访问；确切语法在表达式规范中介绍。也允许单字段形式 `struct W(T);`，可以用作 newtype 模式。

### 单元 `struct`：无字段的标记

最后，单元 `struct` 完全没有字段。

```nabi
struct Marker;
struct Sentinel;
```

这种形式没有主体，在声明之后紧接 `;` 结束。它主要用于只有单个实例才有意义的标记类型。

单元结构体可以带类型参数（`struct Tag<T>;`），作为携带类型的标记。它的值要么写出实参，即 `Tag::<i64>`，要么把实参交给它所放入的槽决定（`let t: Tag<i64> = Tag;`，或在 `fn f(t: Tag<str>)` 下写 `f(Tag)`）。两者都没有时，`let t = Tag;` 是错误，错误会要求给出类型（`let t: Tag<i64> = Tag;`）。

## 2.3.2. 实例创建：预览

每种形式的实例创建语法在表达式规范中正式定义。本文档只预览各形式的样子。

```nabi
let p = Point { x: 1.0f64, y: 2.0f64 };    // named
let q = Pair(1i32, 2i32);                   // tuple
let m = Marker;                              // unit
let t = Tag::<i64>;                          // generic unit
```

## 2.3.3. 字段默认值

具名 `struct` 的字段可以有默认值。

```nabi
struct Config {
    retry: i32 = 3i32;
    timeout: f64 = 30.0f64;
    verbose: bool = false;
}
```

默认值受以下约束。只允许 `const` 表达式：字面量、`const` 变量、`const` 函数调用，以及由这类值组成的 `List` / `Dict` 字面量（`[]`、`[1, 2]`、`{"a": 1}`）。集合类型的默认值在编译期检查，并为每个实例重新构建，因此实例之间从不共享它。`const` 表达式在 [`const`](../005-modules/502-const.md) 中定义。需要运行时调用的表达式（如当前时间或随机数）不能使用；这样的代码是编译错误。引用同一结构体的其他字段也是禁止的。默认值只在实例创建时省略了该字段才求值，并且为每个实例分别求值。由于求值是确定性的，编译器可以施加缓存优化。

这些约束保证了确定性、可预测的开销和语义化版本上的安全。如果需要运行时初始化，使用单独的工厂函数，即 `fn new() -> T` 模式。字段默认值中的名称在声明该结构体的模块的作用域中解析，而不是在构建值的地方解析。派生的 `Default` 也使用声明的默认值（[标准库 2.13.10](../../refs-standard-libraries/200-core/213-marker.md#21310-default默认值)）。

结构体所在模块之外的字面量，只有在私有字段有默认值时才可以省略它（[5.6.8](../005-modules/506-visibility.md#568-struct-字段的可见性)）。`#[non_exhaustive]` 结构体为首次发布之后添加的每个字段提供默认值，这样其他包中的字面量可以继续编译（[6.2.10](../006-meta-system/602-built-in-attributes.md#6210-non_exhaustive-退出外部穷尽性)）。

## 2.3.4. 字段展开

有时新实例应当复制现有实例的字段，只修改其中一部分。

```nabi
let base = Point { x: 0.0f64, y: 0.0f64 };
let p = Point { x: 5.0f64, ..base };
```

这里 `..base` 表示原样取 base 的所有字段，并用新值覆盖显式写出的字段。

在语法上：实例值跟在 `..` 词法单元（[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)）之后。`..` 在一个实例创建表达式中只能出现一次；不允许两个或更多展开。把展开放在显式字段之后是惯用写法。写出的字段先按字段顺序求值，然后再求值 base。

## 2.3.5. 可见性

每个字段都可以用[关键字](../001-lexical/105-keywords.md)中定义的可见性关键字控制其对外暴露。

```nabi
struct Account {
    pub id: i64;        // readable from outside
    pub balance: f64;
        secret: str;    // only within the same module (default = module-private)
}
```

元组结构体的字段也各有自己的可见性（`struct S(pub i32, i32);`，默认私有）。关于构造、`x.N` 访问和模式的规则，以及各可见性级别的含义，见 [5.6.8](../005-modules/506-visibility.md#568-struct-字段的可见性)。

## 2.3.6. 内存布局

字段按声明顺序以自然对齐布局，与 `#[repr(C)]` 相同（[6.2.2](../006-meta-system/602-built-in-attributes.md#622-repr-显式内存表示)）。`packed` 和 `align(N)` 会改变这一点。

## 2.3.7. 同一模块中不得重复定义

在同一模块中声明两个同名结构体是编译错误。同一结构体内的字段名重复也是错误。这条规则与可见性级别无关。元组结构体或单元结构体的构造器是一个值，位于函数、`const` 和 `static` 的命名空间中（[4.1.10](../004-functions/401-function-declaration.md#4110-同一模块内不得重复定义)）：同一模块中的 `struct Point(i32, i32);` 与名为 `Point` 的 `fn`、`const` 或 `static` 构成重复定义（`E0026`）。

## 2.3.8. 值语义

`struct` 是值类型。赋值、传参、返回以及读取元素（`dict[k]`、`list[i]`、`entry(k).or_insert_with(..)`）都会复制结构体。修改只能通过 `ref` 参数或 `ref self`（[4.7](../004-functions/407-ref-keyword.md)），或者把值写回，才能在别处可见。

```nabi
let a = Point { x: 1.0, y: 2.0 };
let b = a;
b.x = 5.0;                        // a.x unchanged

let ps: List<Point> = [a];
let q = ps[0];
q.x = 9.0;                        // ps[0].x unchanged
ps[0] = q;                        // written back
```

对之后没有任何读取的副本进行字段写入是编译错误 `E0057`：这个修改会丢失。受检查的副本有：按值传递的参数或 `self`、闭包的按值参数（`xs.for_each(|p| { p.x = 0.0; })`）、`for` 绑定，以及对元素的 `let`（`let q = xs[i];`）。当之后没有任何读取、循环迭代、返回或调用能看到该修改时，这次写入就是错误；只为同一变量的另一次写入提供数据的读取不算，被闭包或协程块捕获的变量不受检查。帮助信息会指出应当改写成什么：`ref self` 或 `ref` 参数（[4.7](../004-functions/407-ref-keyword.md)）、`for ref`（[3.13.3](../003-expressions/313-loop-expressions.md#for-ref作为位置的元素)）、返回闭包的实参并使用其结果，或者把元素写回。

```nabi
impl Counter {
    fn inc(self) { self.n += 1; }   // error[E0057]: `self.n` is set on a copy of the receiver; write `fn inc(ref self)`
}
for p in ps { p.x = 0.0; }        // error[E0057]: `p.x` is set on a copy of the element; write `for ref p in ps`
for ref p in ps { p.x = 0.0; }    // writes every element of ps
ps.for_each(|p| { p.x = 0.0; });  // error[E0057]: `p.x` is set on a copy of the closure's argument
let qs = ps.map(|p| { p.x = 0.0; p }).to_list();  // returned: kept
let c = self;                     // a local copy on purpose: change `c`, then read or return it
```

共享通过引用句柄进行：集合类型（`List`、`Dict`、`Set`、`StrBuf`、`Deque` 等，[标准库 2.9](../../refs-standard-libraries/200-core/209-collections.md#引用句柄)）、`Mutex`、`RwLock`、原子类型、通道，以及 `[ref x]` 捕获单元。复制句柄会共享对象。

|                         | 值类型（`struct`、`enum`、元组、基本类型）      | 引用句柄（`List`、`Dict`、`Mutex` 等）        |
|-------------------------|-------------------------------------------------|-----------------------------------------------|
| `let b = a;`            | 独立的副本                                      | 同一个对象                                    |
| 字段 / 元素写入         | 只作用于该副本                                  | 通过每个句柄都可见                            |
| 独立副本                | 赋值本身                                        | `.clone()`                                    |

持有句柄的结构体字段复制的是句柄：复制该结构体会共享那个对象。
