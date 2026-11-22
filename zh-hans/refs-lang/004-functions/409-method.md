# 4.9. 方法

本文档详细定义在 [`impl`](../002-types/209-impl.md) 中定义的方法的参数和调用约定。它是函数规范的一部分，重点说明方法与自由函数的区别。

## 4.9.1. 方法定义的位置

方法定义在 [`impl`](../002-types/209-impl.md) 块中。

```nabi
impl Point {
    fn origin() -> Point { Point { x: 0.0f64, y: 0.0f64 } }
    fn distance(self, other: Point) -> f64 { ... }
    fn translate(ref self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }
}
```

## 4.9.2. 按第一个参数的形式划分的方法种类

### 静态方法：没有 `self`

```nabi
impl Point {
    fn origin() -> Point { ... }              // static
    fn from_polar(r: f64, theta: f64) -> Point { ... }
}

let p = Point::origin();                       // Type::method() call
let q = Point::from_polar(1.0f64, 0.0f64);
```

静态方法是第一个参数既不是 `self` 也不是 `ref self` 的函数。它以 `Type::method(args)` 的形式调用，即通过[类型路径](../002-types/213-type-paths.md)进行的静态项访问。

### 实例方法：`self`

```nabi
impl Point {
    fn distance(self, other: Point) -> f64 { ... }
    fn norm(self) -> f64 { ... }
}

let d = p.distance(q);                         // instance.method() call
let n = p.norm();
```

这种形式的第一个参数是 `self`。`self` 表示实例的值（对 `struct` 而言是一份副本，[2.3.8](../002-types/203-struct.md#238-值语义)），函数体用 `self.field` 访问字段。它以 `instance.method(args)` 的形式调用。

### 实例方法：`ref self`

```nabi
impl Point {
    fn translate(ref self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }

    fn replace_with(ref self, other: Point) {
        self = other;                          // the caller's variable becomes other
    }
}

let p = Point { x: 1.0, y: 2.0 };
p.translate(5.0f64, 0.0f64);                   // called without explicit ref
p.replace_with(other);                         // likewise
```

这种形式的第一个参数是 `ref self`。在函数体内，可以给 `self` 及其字段赋值，结果会写回调用方的变量（见 [`ref` 关键字](407-ref-keyword.md)）。调用方不写 `ref`，因为方法调用的接收者会被自动处理。

## 4.9.3. `self` 关键字的两种用法

```nabi
impl Point {
    fn distance(self, other: Point) -> f64 {
        let dx = self.x - other.x;             // uses the instance self
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }
}
```

`self`（[关键字](../001-lexical/105-keywords.md)）用在两个位置。在参数位置，它是接收实例的参数；在函数体内，它是指代该参数的标识符。

参数位置上的 `self` 不写类型。它的类型自动为 `Self`，即当前 `impl` 的目标类型。

## 4.9.4. `Self` 关键字：类型位置

`Self`（[关键字](../001-lexical/105-keywords.md)，首字母大写）是指代当前 `impl` 目标类型的类型名。

```nabi
impl Point {
    fn new(x: f64, y: f64) -> Self {           // Self == Point
        Self { x, y }                          // Self can also construct instances
    }
}
```

两者必须明确区分：小写的 `self` 用作参数和实例标识符；大写的 `Self` 用作类型名。

## 4.9.5. 方法调用接收者的自动 `ref`

对于 `ref self` 方法，即使调用方在调用处不写 `ref`，也会自动应用 `ref`。

```nabi
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}

let c = Counter { count: 0 };
c.increment();                                 // no explicit 'ref'
```

这是 [`ref` 关键字](407-ref-keyword.md)中普通函数调用规则（必须显式写 `ref`）的一个例外。理由是：方法调用语法 `x.method()` 本身就是接收者[分派](../000-meta/007-glossary.md#dispatch分派)机制，而且无论接收者是 `self` 还是 `ref self`，调用形式都相同。再额外显式写 `ref` 只会造成视觉噪音。

这与 Rust、C# 和 Swift 的方法调用约定一致。不过，按路径调用同一个方法时，接收者是普通实参，因此要写 `ref`（`Counter::increment(ref c)`，[4.7.3](407-ref-keyword.md#473-在调用处显式标注c-风格)）。

## 4.9.6. 自由函数与方法的调用形式

| 形式            | 调用                    |
|-----------------|-------------------------|
| 自由函数        | `function_name(args)`   |
| 静态方法        | `Type::method(args)`    |
| 实例方法        | `instance.method(args)` |

调用形式清楚地区分了函数的含义。`x.f(args)` 始终指方法：它绝不会调用字段 `f`，即使该字段存放的是 `fn` 值、闭包或带有 `ops::Call` 的值。这样的字段要用括号括起来调用，写作 `(x.f)(args)`，与 Rust 相同。当 `x` 没有方法 `f`、但有可调用的字段 `f` 时，错误的帮助信息会提示写成 `(x.f)(..)`。

### trait 之间的名称冲突

当 `x.method()` 有多个候选时，按以下顺序选择一个：

1. 固有 `impl` 的方法优先于 trait 方法。
2. 在 trait 方法中，当前包定义的或显式 `use` 的 trait 优先于 `core` trait（`core` trait 在任何地方都处于方法作用域中）。当前包是调用所在的实际包。依赖包的 trait 除非被 `use`，否则不优先于 core 的 trait。
3. 如果仍有多于一个候选，就是歧义错误，诊断会建议写成 `Trait::method(x)`。同一个 trait 仅在类型参数上不同的多个实现（`Call<f64>`、`Call<Array<f64, N>>`）不构成冲突：由实参选择实现（[3.1.3](../003-expressions/301-operator-dispatch.md#313-不经虚表的分派)）。

```nabi
use pkg::geom::Union;              // user trait: union method
a.union(b);                        // pkg::geom::Union takes precedence over core's Union
core::ops::Union::union(a, b) // explicit to use the core one
```

## 4.9.7. 在方法内调用其他方法

```nabi
impl Point {
    fn distance(self, other: Point) -> f64 { ... }

    fn distance_squared(self, other: Point) -> f64 {
        let d = self.distance(other);          // another method of the same instance
        d * d
    }
}
```

可以用 `self.method()` 调用同一实例的其他方法。适用与普通实例调用相同的规则。

## 4.9.8. 方法的可见性

方法的可见性遵循 [`impl`](../002-types/209-impl.md) 中定义的模块私有默认值。

```nabi
impl Account {
    pub fn balance(self) -> f64 { ... }        // public
    fn _internal(self) { ... }                  // module-private
}
```

可见性级别的确切规则在[可见性](../005-modules/506-visibility.md)中定义。

## 4.9.9. 方法签名的自由度

方法的参数、返回和函数体遵循[参数](402-parameters.md)和[返回](403-return.md)中的普通函数规则。具体而言，方法中可以自由使用默认值、具名参数、可变参数和展开。也可以有发散方法（函数体总是通过 `throw` 或无限循环退出），泛型方法既可以使用 `impl` 块的泛型参数，也可以使用方法自己的泛型。

```nabi
impl<T> List<T> {
    fn map<U>(self, f: fn(T) -> U) -> List<U> { ... }
}
```

方法自己的泛型跟在函数名之后（`fn map<U>(...)`）。在 trait `impl` 中，没有写类型的方法参数取 trait 在该位置声明的类型（代入 trait 参数和关联类型），而不是隐式的类型参数。

## 4.9.10. 方法中 `self` 的小结

| 第一个参数      | 含义                      | 调用时                               | 函数体内能否重新赋值 `self`？          |
|-----------------|---------------------------|--------------------------------------|---------------------------------------|
| （无）          | 静态方法                  | `Type::method()`                     | -                                     |
| `self`          | 实例方法（值）            | `instance.method()`                  | 否（仅限局部）                         |
| `ref self`      | 实例方法（`ref`）         | `instance.method()`（自动 `ref`）     | 是（反映到调用方）                     |

方法只有这三种形式。
