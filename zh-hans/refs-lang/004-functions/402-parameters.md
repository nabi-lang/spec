# 4.2. 参数

本节定义函数参数的形式，以及调用处实参与参数的匹配规则。

```
param           ::= self_param | regular_param
self_param      ::= "ref"? "self"
regular_param   ::= "ref"? identifier (":" type_expr)? ("=" const_expr)?
                  | param_pattern (":" type_expr)?
param_pattern   ::= tuple_pattern | struct_pattern
```

`param_pattern` 是代替名字的不可反驳的模式（[3.11](../003-expressions/311-patterns.md)）：`fn f((a, b): (i32, i32))`、`fn g(P { x, y }: P)`。实参在主体开头被解构，效果同 `let (a, b) = arg;`（[3.15](../003-expressions/315-destructuring.md)）：长度与参数类型不同的元组模式是 E0101，可反驳的模式是 E0100。这样的参数对调用者没有名字，所以不能有默认值，也不能用具名参数传入。`ref` 参数总是一个名字（`ref (a, b)` 是错误，E0010）。

## 4.2.1. 基本形式

```nabi
fn f(x: i32, y: i32) -> i32 { x + y }
```

每个参数写作 ` name: type`，以逗号分隔。参数名遵循[标识符](../001-lexical/104-identifiers.md)的形式，类型遵循[类型表达式](../002-types/201-type-expressions.md)的规则。

## 4.2.2. 省略类型

在模块私有或 `pub(sub)` 函数上，未写类型的参数视为泛型。详见[泛型](../002-types/210-generics.md)的单态化规则。在 trait `impl` 中，未写类型的方法参数取 trait 在该位置声明的类型（代入 trait 参数和关联类型），而不是隐式类型参数。

```nabi
fn id(x) { x }                                   // x is generic; monomorphized at call
fn pair(x, y) { ... }                             // both parameters generic
fn mixed(a: i32, b) { ... }                       // a is i32, b is generic
```

上面的 `id` 等价于显式使用[泛型](../002-types/210-generics.md)的形式：

```nabi
fn id<T>(x: T) -> T { x }
```

主体中用到的 trait 不会自动合成为签名中的 `where` 子句。每个调用处分别单态化，主体对 trait 的使用在那里检查。

```nabi
fn compute(x, y) { x + y }                       // signature: <T, U>(x: T, y: U)
                                                 //   no where clause synthesized

compute(1i32, 2i32);                             // OK: i32 + i32 dispatch
compute(1.0f64, 2.0f64);                         // OK: separate monomorphized instance
compute(MyType, 1i32);                           // error at the call site:
                                                 //   "MyType has no Add<i32>"
                                                 //   (reported clearly where it is called,
                                                 //    not at the signature)
```

如果合成一个累积主体中所有 trait 的 `where` 子句，签名会膨胀，由此推断出的类型也会以庞大的形式出现在工具中；因此不做合成。

`pub` 或 `pub(pkg)` 函数要写明每个参数的类型（上述 trait `impl` 方法除外）。其他模块会调用它，隐式参数会让主体的修改改变调用者可以传入的内容。在这类函数上，未写类型的参数是编译错误；应写明类型，或写一个带有主体所需约束的类型参数。

```nabi
pub fn add(x, y) { x + y }                       // ERROR: `x` and `y` have no type
pub fn add(x: i32, y: i32) -> i32 { x + y }      // OK
pub fn add<T: ops::Add<Output = T>>(x: T, y: T) -> T { x + y }   // OK: any T with Add
```

如果参数要作为动态槽，就显式写 `any`（见 [`any` 槽](../002-types/202-any-slot.md)）。

```nabi
fn handle(x: any) { ... }                         // x is an any slot
```

## 4.2.3. 默认值

参数可以有默认值。

```nabi
fn retry(n: i32 = 3i32, delay: f64 = 0.1f64) { ... }

retry();                          // n=3, delay=0.1
retry(5i32);                       // n=5, delay=0.1
retry(5i32, 1.0f64);               // n=5, delay=1.0
```

默认值适用以下规则。只允许 `const` 表达式：字面量、`const` 变量和 `const` 函数调用。这与 [`struct`](../002-types/203-struct.md) 字段默认值的约束相同。只有调用处省略的参数才会被求值，并且每次调用都重新求值。默认值是确定的：同一个 `const` 表达式总是产生同一个值。不能使用当前时间或随机数这类运行时调用；需要时改用工厂函数模式。默认值中的名字在声明该函数的模块的作用域中解析，而不是在调用者的作用域中解析，此外还可以使用函数自身的 `const` 参数，它们由前面的实参确定（与 `a: Array<f64, N>` 一起写 `axis: usize = N - 1`）。默认值取实参绑定后的参数类型，所以 `xs: List<T> = []` 是该次调用的 `T` 的 `List<T>`。不是 `const` 表达式的默认值在声明处报错。默认值位于尾部：在有默认值的参数之后出现没有默认值的参数，在声明处报错。有两种参数例外：可变参数（4.2.5），以及由尾随闭包填充的、`fn` 类型的最后一个参数（[4.8.9](408-trailing-closure.md#489-与具名参数结合)）。

## 4.2.4. 具名参数调用

在调用处，可以按参数名给出实参。

```nabi
fn make_user(name: str, age: i32 = 0i32, email: str = "") -> User { ... }

make_user(name: "Alice", age: 30i32);
make_user(name: "Bob", email: "bob@example.com");      // age uses its default
make_user("Charlie", age: 25i32);                       // only the first argument positional, the rest named
```

具名参数调用适用以下规则。位置实参必须在前，具名实参在后；在具名实参之后出现位置实参是错误（E0049）。在一次调用中同时以位置和名字给出同一个参数是错误。所有位置实参匹配完后，其余参数必须由具名实参或默认值填充。

## 4.2.5. 可变参数：`...T`

接受可变数量实参的参数。

```nabi
fn sum(xs: ...i32) -> i32 {
    let total = 0i32;
    for x in xs { total += x; }
    total
}

sum();                            // xs: empty slice
sum(1i32);                         // xs = [1]
sum(1i32, 2i32, 3i32);             // xs = [1, 2, 3]
let ys: List<i32> = [4, 5];
sum(..ys);                         // xs = [4, 5] (list spread)
```

可变参数适用以下规则：

- 只有最后一个参数可以是可变参数。
- 在主体中，`xs: ...T` 是元素类型为 `T` 的只读切片视图（`Slice<T>`，[标准库 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet视图)）。调用中剩余的实参被打包成一个，长度可以每次调用不同。因此函数不会按实参个数分别单态化。
- 单个实参、元组展开和列表展开（`..xs`）可以混用（4.2.6）。
- 主体可以读取、索引和迭代它。如果需要修改，就用库方法构建一个新的集合。

### 异构可变参数：`...any`

```nabi
fn log_all(xs: ...any) { ... }
log_all(1i32, "hi", 3.14f64);     // various types allowed
```

参数类型为 `any` 时，可变参数接受异构的实参。

## 4.2.6. 展开：`..args`

在调用处把已有集合展开为实参的形式。

```nabi
let args = (1i32, 2i32, 3i32);
foo(..args);                      // foo(1, 2, 3)

let xs: List<i32> = [1i32, 2i32, 3i32];
sum(..xs);                         // sum(1, 2, 3) (received as variadic)
```

展开适用以下规则：

- 元组展开在编译期检查长度和类型。
- 展开到固定参数的 `List` / `Slice` 填充到下一个实参之前的槽位，并在运行时检查长度。数组展开填充其长度个槽位。
- Dict 展开转换为具名实参。每个键必须指向一个未被其他方式填充的参数（在运行时检查），其值必须具有该参数的类型。Dict 的值类型 `V` 不适配的、带默认值的参数取其默认值；如果有键指向这样的参数，则在运行时 panic。
- 在可变参数位置，元组、`List`、`Slice` 和数组的展开可以与普通实参混用（`f(a, ..xs, b)`）。可变参数可以原样传给另一个可变参数（`fn f(xs: ...i32) { g(..xs) }`）。
- 当操作数是内部不含 `..` 的 `List` 字面量，或键均为字符串字面量的 Dict 字面量时，编译器也会在编译期检查。长度与要填充的槽位不同，或者键指向的参数已无需填充时，给出警告 lint `literal_spread_mismatch`（`spread of 3 elements into 2 parameters; this call panics`）。运行时检查仍然保留。
- 展开非集合值是错误（E0051，``spread `f(..x)` needs a tuple, a `List`, a `Slice`, an array or a `Dict<str, V>`, found `i32` ``）。
- 不带实参调用元素类型 `T` 为开放泛型的可变参数函数时，无法确定 `T`，所以需要类型注解。

- 在 `List` 字面量中，`..x` 展开元组、数组、`List` 或 `Slice`（`[0, ..xs, 9]`）；否则它是区间。

```nabi
// fn train(lr: f64, momentum: f64, epochs: i32 = 10)
let opts = {"lr": 0.01, "momentum": 0.9};   // Dict<str, f64>
train(..opts);                  // train(lr: 0.01, momentum: 0.9), epochs: 10
// A key "epochs" in opts would panic: `f64` does not fit `epochs: i32`.
```

在一次调用中，展开和普通实参也可以自由混用。

```nabi
foo(0, ..middle, 10);
foo(..a, ..b, ..c);
```

## 4.2.7. 丢弃：`_`

显式丢弃调用结果的写法。

```nabi
_ = compute_with_side_effect();
```

这里的 `_` 是[标识符](../001-lexical/104-identifiers.md)中的单个下划线。它在左侧位置接收值，但不绑定到任何地方，表示用户有意忽略结果。`let _ = e;` 与之相同，并且可以在同一作用域中使用任意多次（`_` 不是绑定，所以不算重复声明）。

## 4.2.8. 参数求值顺序

调用时，实参表达式按书写顺序（从左到右）求值。方法调用先求值接收者。

```nabi
foo(a(), b(), c());               // a() first, then b(), c() last
```

具名实参同样按书写顺序求值。

```nabi
make_user(name: f(), age: g());   // f() first, then g()
```

这一顺序保证带副作用的实参以可预测的顺序运行。

## 4.2.9. 参数作用域

参数被引入函数主体的顶层作用域，在主体中像普通变量一样使用。

默认值表达式在不含参数的环境中求值。因此它们不能引用同一函数的其他参数。

```nabi
fn f(a: i32, b: i32 = a + 1) { ... }   // ERROR: default cannot refer to a
```

这一约束与 [`struct`](../002-types/203-struct.md) 字段默认值约束的动机相同：确定性和可预测的开销。

## 4.2.10. 静态参数与渐进参数

| 写法         | 含义                                                 |
|--------------|------------------------------------------------------|
| `x: i32`     | 静态。调用处只接受 i32                               |
| `x`          | 泛型。按调用处的类型单态化                           |
| `x: any`     | 动态槽。接受任何带运行时标签的类型                   |

这三种形式可以在一个函数中自由混用。
