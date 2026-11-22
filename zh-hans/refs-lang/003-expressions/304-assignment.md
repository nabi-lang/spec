# 3.4. 赋值与复合赋值

赋值（`=`）和复合赋值（`+=`、`-=` 等）更新变量、字段或索引位置的值。两者都是语句；简单赋值也可以以 3.4.5 的形式出现在表达式位置，而复合赋值不可以。语法如下：

```
assign_stmt     ::= lvalue "=" expr ";"
compound_stmt   ::= lvalue compound_op expr ";"
assign_expr     ::= lvalue "=" expr                    // in parentheses or an arm body (3.4.5)
compound_op     ::= "+=" | "-=" | "*=" | "**=" | "/=" | "%=" | "%%=" | "&=" | "|="
                  | "^=" | "<<=" | ">>="
assign_target   ::= lvalue | tuple_target
tuple_target    ::= "(" target_item ("," target_item)+ ","? ")"
target_item     ::= lvalue | tuple_target | "_"
lvalue          ::= identifier
                  | lvalue "." identifier
                  | lvalue "." int_lit                 // tuple position
                  | lvalue "[" expr "]"                // index
```

`assign_stmt` 的左侧是 `assign_target`：由位置组成的元组会对其中每个位置赋值（3.4.6）。其他形式的左侧是 `lvalue`。

## 3.4.1. 简单赋值

简单赋值的形状如下。

```nabi
x = expr;
obj.field = expr;
arr[i] = expr;
```

左侧（左值）是一个可重新赋值的位置，右侧（右值）是任意表达式，结果值为单元类型（`()`）。赋值通常是语句；当它出现在表达式位置时（3.4.5），其结果是单元值。因此，像 C 的 `if (x = read())` 这类很少需要的模式，会被单元结果挡住。

### 左值的种类

左侧允许的形式及其含义：

| 形式             | 示例              | 含义                                             |
|------------------|-------------------|--------------------------------------------------|
| 简单变量         | `x = ...`         | 把值写入该变量的槽                               |
| 字段访问         | `obj.field = ...` | 把值写入对象的字段                               |
| 元组位置         | `pair.0 = ...`    | 把值写入元组 `struct` / 元组的某个位置           |
| 索引             | `arr[i] = ...`    | 把值写入被索引的位置                             |

索引赋值脱糖为[索引与切片](306-indexing-and-slicing.md)的写入 trait。切片不是左值：`arr[1:3] = other;` 会被拒绝（[3.6](306-indexing-and-slicing.md)）。

### 不能作为左值的位置

以下内容不能作为左值。

```
1 = 2;                    // a literal is not an lvalue
foo() = 2;                // a call result is not an lvalue
foo().x = 2;              // nor is a field of one
(x + 1) = 2;              // an arithmetic result is not an lvalue
(a, 1) = (b, 2);          // each element of a tuple target is a place or `_` (3.4.6)
```

`List`、`Dict` 或其他共享集合的元素即使来自调用，也是一个位置：`rows()[0] = 1;` 写入 `rows()` 返回的列表。调用结果的字段不是位置，作为调用结果的定长数组的元素也不是。

某个东西能否作为左值，在编译期根据其形式和被索引集合的类型检查，不进行运行时检查。对不是位置的东西赋值是错误（E0140），无论用 `=` 还是复合赋值（3.4.3）：写入会落到一个临时值上并丢失。

## 3.4.2. 声明新变量与重新赋值

结合 `let` [关键字](../001-lexical/105-keywords.md)，区分两种形式。

```nabi
let x = 1;       // new declaration, 'let' required
x = 2;           // reassignment, without 'let'
```

创建新变量需要 `let`；再次写入现有变量的槽时不写 `let`。对未声明的名字不带 `let` 赋值是编译错误。对同一作用域中已声明的名字使用 `let` 是重新声明：它创建一个新绑定，类型可以不同，并保持旧槽原样（[5.1.4](../005-modules/501-variable-declaration.md#514-redeclaration-in-the-same-scope)）。确切规则定义于[变量声明](../005-modules/501-variable-declaration.md)。

```nabi
let n = "42";
let n: i32 = try! n.parse();   // redeclaration: a new binding of another type
n = n + 1;                     // reassignment of the new binding
```

```
counter = counter + 1;     // OK if counter is declared
coutner = ...;             // ERROR: `coutner` is not declared; a variable is created only by `let`
```

这条规则堵住了拼写错误悄悄创建新变量的陷阱（这是 Julia 和 Python 中常见的 bug）。

## 3.4.3. 复合赋值

复合赋值把一次运算和一次赋值合为一种形式。

```nabi
x += 1;            // semantically the same as x = x + 1
x -= 1;
x *= 2;
arr[i] += 1;
```

[标点与运算符词法单元](../001-lexical/110-punctuation-and-operator-tokens.md)中的所有复合赋值词法单元（`+=`、`-=`、`*=`、`**=`、`/=`、`%=`、`%%=`、`&=`、`|=`、`^=`、`<<=`、`>>=`）都遵循同一模式。复合赋值是语句，以 `;` 结束（在 `}` 之前可省略，[1.3.4](../001-lexical/103-statement-terminator.md#134-block-like-expression-statements)）；它没有值，不能出现在表达式位置（`let y = (x += 1);` 无法解析）。原地广播形式 `.=` `.+=` `.-=` `.*=` `./=` `.%=` `.%%=` `.**=` `.&=` `.|=` `.^=` `.<<=` `.>>=` 是同类语句，有其自身含义（[3.3.7](303-broadcast-operators.md#337-in-place-broadcast)）。

### 脱糖与左值求值

复合赋值按如下方式脱糖。如果该类型实现了 `OpAssign` trait，`a += b` 调用它；否则即为 `a = a + b`（[stdlib 2.4.1](../../refs-standard-libraries/200-core/204-ops-assign.md)）。每个运算符都是如此。

```
lvalue OP= rhs       →   OpAssign::op_assign(ref lvalue, rhs)   // if implemented
                     →   lvalue = (lvalue OP rhs)               // otherwise
```

编译器保证左值不会被求值两次。

```nabi
arr[expensive_index()] += 1;
// expensive_index() is called only once
```

如果左值是普通变量或字段，就原样引用两次；如果它包含调用或计算，该部分会绑定到一个临时值并只求值一次。这一保证防止带副作用的索引或方法被无意中调用两次。先求值左值还是 `rhs`，取决于操作数类型（3.4.7）。

左值必须是一个位置（3.4.1）。`f() += 1` 和 `d.entry(k).or_insert(0) += 1` 是错误（E0140）：它们各自会更新一个临时值。通过锁守卫的 `get()` 写入（`g.get().total += 1`）也是如此，因为它返回副本；帮助信息会给出 `g.update(|v| { v.total += 1; v })`，整体替换时则给出 `g.set(v)`。要在字典中计数，使用带默认值的索引（[3.6.1](306-indexing-and-slicing.md#361-indexing-arri)）：

```nabi
counts[w, default: 0] += 1;
```

### trait 映射

每种复合赋值映射到独立的 **Assign** trait 族。

```
x += y       →   <typeof(x) as AddAssign<typeof(y)>>::add_assign(ref x, y)
x -= y       →   <typeof(x) as SubAssign<typeof(y)>>::sub_assign(ref x, y)
```

（有 `AddAssign`、`SubAssign`、`MulAssign`、`DivAssign` 等。）

这些 trait 可以实现为原地更新，从而得到比 `x = x + y` 更高效的实现。例如，向大型集合追加元素时可以不创建新集合。

实现 `AddAssign` 是可选的：没有它时，`x += y` 回退为 `x = x + y`。而 `x + y` 本身总是需要 `Add`。

### 签名

`AddAssign` 等的签名形状如下。

```nabi
trait AddAssign<Rhs = Self> {
    #[infix("+=")]
    fn add_assign(ref self, rhs: Rhs);
}
```

与其他运算符一样，`*Assign` trait 通过 `#[infix("op=")]` 注册（[3.1](301-operator-dispatch.md)）。`ref self` 指向左值的槽；没有返回类型，结果是单元值。`ref self` 以及更新左值槽的确切含义定义于函数规范。

## 3.4.4. 链式赋值

nabi 不允许 C 和 Python 风格的链式赋值（`a = b = c = 1`）。

```
a = b = c = 1;             // ERROR
```

由于赋值的结果是单元值，`b = c = 1` 求值为单元值，整体就成了 `a = (unit)`，这没有意义。把预期的含义写成分开的赋值：

```nabi
c = 1; b = 1; a = 1;
```

## 3.4.5. 表达式中的赋值

简单赋值可以放在表达式位置，此时其结果是单元值，但这样做几乎没有价值。

```nabi
let x = (y = 1);           // x = unit (nearly meaningless)
```

不鼓励这种模式，工具或 lint 工具可以对其发出警告。

在语句位置之外，赋值要写在括号中 `(y = 1)`，或作为 match 分支的主体。不支持不加括号的 `let x = y = 1;`。复合赋值没有表达式形式。

## 3.4.6. 解构赋值

`=` 左侧由位置组成的元组，会用右侧的对应元素对每个位置赋值：

```nabi
(a, b) = (b, a);                    // swap
for _ in 0..n {
    (x, y) = (y, x + y);            // Fibonacci step
}
(xs[i], xs[j]) = (xs[j], xs[i]);    // swap two elements
(p.x, _, p.y) = (1, 2, 3);          // `_` discards
(c, (d, e)) = t;                    // nested
```

先完整求值右侧，再从左到右对每个位置赋值；位置自身的接收者或索引在对它赋值时求值（3.4.7）。所以 `(a, b) = (b, a + b)` 读取的是旧的 `a` 和 `b`。每个位置都必须已经存在：赋值不会声明（3.4.2），尚未声明的名字是 E0057，帮助信息会提示先用 `let` 声明它。两侧的形状必须一致（E0101）。其含义等同于

```nabi
{ let (t0, t1) = (b, a + b); a = t0; b = t1; }
```

其中使用隐藏的临时值。用 `let` 解构则会声明新变量（[3.15](315-destructuring.md)）；在循环内，`let (a, b) = (b, a + b);` 为该次迭代声明新的 `a` 和 `b`，外层的保持原样。

## 3.4.7. 求值顺序

与 Rust 相同，赋值先求值其值，再求值其位置。位置的各部分（它作为字段或元素所属的接收者，然后是其索引）在值之后从左到右各求值一次，存储最后进行。

| 形式 | 顺序 |
|------|-------|
| `place = value` | `value`，然后 `place` |
| `place op= value`，两个操作数都是基本类型（整数、浮点数、`bool`、`char`、`str`） | `value`，然后 `place` |
| `place op= value`，其他任何操作数 | `place`，然后 `value` |
| `(p0, p1) = value`（3.4.6） | 完整的 `value`，然后 `p0`，然后 `p1` |
| `place .= value`（[3.3.7](303-broadcast-operators.md#337-in-place-broadcast)） | `value` 的各操作数按源代码顺序，然后 `place` |
| `place .op= value` | `place`，然后 `value` 的各操作数 |

```nabi
xs[f()] = g();          // g(), then f()
xs[f()] += g();         // xs: List<i64>: g(), then f()
ms[f()] += g();         // ms: List<Money> with AddAssign: f(), then g()
(xs[f()], ys[h()]) = (g(), k());   // g(), k(), then f(), then h()
```

其他操作数的复合赋值是对该位置调用 `OpAssign` 方法（3.4.3），所以其位置先求值，正如任何方法调用中的接收者那样（[4.2.8](../004-functions/402-parameters.md#428-parameter-evaluation-order)）；`.op=` 把目标作为 `target .op value` 的第一个操作数读取。值先求值时，触发 panic 或抛出错误的值不会求值位置的任何部分，而改变了位置所读取变量的值（通过 `[ref]` 捕获改变的索引、通过 `ref self` 方法改变的字段）会被位置看到：

```nabi
let i = 0;
let bump = [ref i] || -> i64 { i += 1; return 1; };
zs[i] = bump();         // stores at zs[1]
```
