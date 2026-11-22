# 2.4. 复合赋值 trait

`core::ops` 的 `*Assign` trait 族把 `a += b`、`a *= b` 等复合赋值运算符分派为**就地**更新。[refs-lang 3.4 赋值与复合赋值](../../refs-lang/003-expressions/304-assignment.md)中定义的语法会脱糖为这些 trait。

> [!NOTE]
> Nabi 的普通代码不使用 `mut` 关键字（[refs-lang 5.1 变量声明](../../refs-lang/005-modules/501-variable-declaration.md)）。
> 变量总是用 `let` 声明，对同一个名字再次赋值就是**重新赋值**。
> 堆对象（List、Dict 等）能否就地更新，不由 `let mut` 声明符决定，而由
> **对象是否公开可变方法**决定。`*Assign` trait 就是这些可变方法所在的位置。

## 2.4.1. 关键区别：`+=` 与先 `=` 再 `+`

```nabi
a += b;
```

这个表达式有两种可能的含义。采用**就地分派**时，它解析为 `<typeof(a) as AddAssign<typeof(b)>>::add_assign(ref a, b)`，直接修改 `a` 所引用对象的**内容**，不创建新对象。采用**回退**（先 `Add::add` 再重新赋值）时，它变为 `a = a + b`，把 `Add` 调用的结果存回 `a`，并创建一个新对象。

Nabi 会自动在两种方式之间选择。

```
Compiler behavior (on a += b):
1. If typeof(a) implements AddAssign<typeof(b)>
     → dispatch AddAssign::add_assign(ref a, b)
2. Otherwise, if typeof(a) implements Add<typeof(b)>
     → desugar to a = Add::add(a, b)
3. If neither, compile error
```

这一策略使集合类型可以做就地优化。例如 `xs: List<i32>` 执行 `xs += y` 时，会直接 push，而不创建新的 List。

## 2.4.2. 算术 *Assign trait

### `AddAssign`：`+=`

```nabi
trait AddAssign<Rhs = Self> {
    #[infix("+=")]
    fn add_assign(ref self, rhs: Rhs);
}
```

```nabi
let counter: i32 = 0;
counter += 5;       //→ 5 (counter is updated in place)
counter += 3;       //→ 8

let xs: List<i32> = [1, 2];
xs += [3, 4];       //→ [1, 2, 3, 4]  (List's add_assign: efficient push)
```

### `SubAssign`：`-=`

```nabi
trait SubAssign<Rhs = Self> {
    #[infix("-=")]
    fn sub_assign(ref self, rhs: Rhs);
}
```

### `MulAssign`：`*=`

```nabi
trait MulAssign<Rhs = Self> {
    #[infix("*=")]
    fn mul_assign(ref self, rhs: Rhs);
}
```

### `PowAssign`：`**=`

```nabi
trait PowAssign<Rhs = Self> {
    #[infix("**=")]
    fn pow_assign(ref self, rhs: Rhs);
}
```

`x **= e` 即 `x = x ** e`；指数的实现与 `Pow`（2.3.1）相同：整数用 `u32`，浮点数用该浮点类型或 `i32`。

### `DivAssign`：`/=`

```nabi
trait DivAssign<Rhs = Self> {
    #[infix("/=")]
    fn div_assign(ref self, rhs: Rhs);
}
```

### `RemAssign`：`%=`

```nabi
trait RemAssign<Rhs = Self> {
    #[infix("%=")]
    fn rem_assign(ref self, rhs: Rhs);
}
```

### `ModuloAssign`：`%%=`

```nabi
trait ModuloAssign<Rhs = Self> {
    #[infix("%%=")]
    fn modulo_assign(ref self, rhs: Rhs);
}
```

`%%=` 是 [`Modulo`](203-ops.md#modulo向下取整取模)（向下取整取模，结果的符号与除数相同）的复合赋值形式。它是独立于 `RemAssign` 的 trait，并且就地更新，因此除数为正时，结果总在 `[0, rhs)` 之内。

## 2.4.3. 位运算 *Assign trait

### `BitAndAssign`：`&=`

```nabi
trait BitAndAssign<Rhs = Self> {
    #[infix("&=")]
    fn bitand_assign(ref self, rhs: Rhs);
}
```

```nabi
let flags: u32 = 0xFF;
flags &= 0x0F;     //→ 0x0F
```

### `BitOrAssign`：`|=`

```nabi
trait BitOrAssign<Rhs = Self> {
    #[infix("|=")]
    fn bitor_assign(ref self, rhs: Rhs);
}
```

### `BitXorAssign`：`^=`

```nabi
trait BitXorAssign<Rhs = Self> {
    #[infix("^=")]
    fn bitxor_assign(ref self, rhs: Rhs);
}
```

### `ShlAssign`：`<<=`

```nabi
trait ShlAssign<Rhs = Self> {
    #[infix("<<=")]
    fn shl_assign(ref self, rhs: Rhs);
}
```

```nabi
let n: i32 = 1;
n <<= 4;       //→ 16
```

### `ShrAssign`：`>>=`

```nabi
trait ShrAssign<Rhs = Self> {
    #[infix(">>=")]
    fn shr_assign(ref self, rhs: Rhs);
}
```

## 2.4.4. 签名模式：`ref self`

每个 `*Assign` 方法都接受 `ref self`（[refs-lang 4.7 ref 关键字](../../refs-lang/004-functions/407-ref-keyword.md)）。它引用调用方的变量绑定本身，所以修改是就地发生的。

```nabi
fn add_assign(ref self, rhs: Rhs);
       ^^^^^
       directly modifies the object the caller's variable refers to
```

这个签名是就地行为的核心。如果只接受 `self`（按值），修改的就是副本，调用方看不到变化。

## 2.4.5. 优先级 / 结合性：prec=-2，右结合

每个 `*Assign` 词法单元在 [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) 的表中都固定在同一级别。其优先级为 `prec = -2`，低于普通算术（`+` 为 9，`*` 为 10）、`..`（=0）和 `|>`（=-1），是最低的一级。不允许链式赋值（[refs-lang 3.4.4](../../refs-lang/003-expressions/304-assignment.md)）。

## 2.4.6. trait 实现矩阵

原始类型的每个 `*Assign` 实现都带有 `#[intrinsic("nabi.OP_assign.TYPE")]`。

| 类型         | trait                                       | 行为                                            |
|--------------|---------------------------------------------|-------------------------------------------------|
| `i*`, `u*`   | 6 个算术 + 5 个位运算（共 11 个 trait）     | `nabi.add_assign.i32`, `nabi.shl_assign.u32`, ... |
| `f32`, `f64` | 仅 6 个算术                                 | `nabi.add_assign.f64`, ...                       |
| `bool`       | `BitAndAssign`/`BitOrAssign`/`BitXorAssign` | `nabi.and_assign.bool`, ...                      |
| `int`        | 算术 + 位运算；`ShlAssign<u32>` / `ShrAssign<u32>` | 自动提升逻辑；替换为新值（不可变）       |
| `float`      | 算术                                        | （同上：自动提升）                              |
| `List<T>`    | `AddAssign<T>`, `AddAssign<List<T>>`        | `xs += x` push `x`；`xs += ys` 就地 push `ys` 的所有元素 |
| `str`        | `AddAssign<str>`, `AddAssign<char>`         | `s = s + t`：新的 str 替换 `s`（str 不可变）    |
| `Dict<K,V>`  | `AddAssign<Dict<K,V>>`（`K: Hash`）         | 就地合并；键相同时以右侧的值为准                |
| `Set<T>`     | `AddAssign<T>`（`T: Hash`）                 | `s += x` 插入 `x`                               |
| `Deque<T>`   | `AddAssign<T>`                              | `q += x` 把 `x` push 到末尾                     |

```nabi
let xs: List<i32> = [1];
xs += 2;              //→ [1, 2]
xs += [3, 4];         //→ [1, 2, 3, 4]

let s = "ab";
s += 'c';             //→ "abc"

let d: Dict<str, i32> = {"a": 1, "b": 2};
d += {"b": 20, "c": 3};   //→ {"a": 1, "b": 20, "c": 3}
```

左侧类型实现了 `XxxAssign` 时，`a op= b` 调用 `XxxAssign::xxx_assign`（`AddAssign::add_assign` 等）；否则它就是 `a = a op b`。

## 2.4.7. 集合类型上的 *Assign：主要用途

对于 `List<T>` / `Dict<K,V>` 这样的可变容器，`*Assign` 执行的是**真正的就地**操作。

```nabi
let xs: List<i32> = [1, 2, 3];
let alias = xs;             // shares the same object (Python-style, refs-lang 4.7)

xs += [4, 5];               // push onto xs (no new object)
alias;                       //→ [1, 2, 3, 4, 5]  (alias sees the change too)

// compare: xs = xs + [4, 5] creates a new List, and alias does not see the change
```

这一含义与 Nabi 对堆对象的自动共享模型（[refs-lang 4.7](../../refs-lang/004-functions/407-ref-keyword.md)）完全一致。

## 2.4.8. 用户定义的 *Assign

如果用户类型只实现了 `Add` 而没有实现 `AddAssign`，编译器使用回退方式。

```nabi
struct Vec3 { x: f64; y: f64; z: f64; }

impl ops::Add for Vec3 {
    type Output = Vec3;
    fn add(self, rhs: Vec3) -> Vec3 { ... }
}
// AddAssign not defined

let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
v += Vec3 { x: 0.5, y: 0.5, z: 0.5 };
// → compiler: no AddAssign, Add exists → desugar 'v = v + (...)'
// Vec3 is immutable, so a new object is created and reassigned to v
```

在性能要紧的地方，建议用户显式实现 `AddAssign`。

```nabi
impl ops::AddAssign for Vec3 {
    fn add_assign(ref self, rhs: Vec3) {
        self.x += rhs.x;
        self.y += rhs.y;
        self.z += rhs.z;
    }
}
```

## 2.4.9. *Assign 的超 trait 关系：无

Nabi **不**把 `Add` 作为 `AddAssign` 的超 trait（supertrait）。这两个 trait 相互独立。

原因在于，有些类型（例如集合类型）的 `AddAssign` 有意义，`Add` 却没有意义：`Set<T>` 和 `Deque<T>` 实现了 `AddAssign<T>`（插入或 push 一个元素），但没有实现 `Add<T>`。如果有超 trait 关系，这样的类型就无法只实现 `AddAssign`。

## 2.4.10. 相关：广播有没有 `*Assign`

广播词法单元没有 `*Assign` trait。就地广播是语言语法：`y .= v` 以及 `y .+= v`、`.-=`、`.*=`、`./=`、`.%=`、`.%%=`、`.**=`、`.&=`、`.|=`、`.^=`、`.<<=`、`.>>=` 逐元素写入 `List`、定长数组或 `Broadcastable` 目标，与右侧的表达式融合，且不分配内存（[refs-lang 3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-原地广播)）。比较运算以及 `.&&`、`.||`、`.??` 通过 `.=` 实现（`ok .= ok .&& xs .> 0`）。

```nabi
xs .+= ys;                // xs[i] = xs[i] + ys[i]; lengths must match
y .= a .* x .+ b;         // one loop into y
```

长度不一致时，会像融合表达式一样触发 panic（`broadcast: lengths 3 and 2 differ; make both sides the same length`）。
