# 0.3. 内建函数机制：编译器 ↔ 标准库的边界

标准库 trait 方法（例如 `i32::add`）的主体最终必须降级为机器的 `add` 指令。本章定义标准库方法主体与编译器后端（PrimOp / MIR / 代码生成）相连的路径。

## 0.3.1. 问题：自举循环

问题的核心是标准库与编译器之间的自举循环。下面的流程显示了卡住的位置。

```
user code:         let r = a + b;        (a, b: i32)
↓
compiler:          dispatch Add::add(a, b) call     (refs-lang 3.1)
↓
standard library:  impl Add for i32 { fn add(self, rhs) { ... } }
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
              what goes in the "..." here?
              without an intrinsic, "self + rhs" would dispatch Add::add again → infinite recursion
              a raw machine `add` cannot be written either (not part of the user language surface)
↓
machine code:      add w0, w0, w1          (aarch64)
```

因此需要一条路径，让标准库的主体不调用自身，直接通向编译器后端的 `add` 指令。

## 0.3.2. 解决方案：`#[intrinsic("nabi.OP.TYPE")]` 属性

解决方案是在标准库方法主体上加一个编译器能识别的属性。编译器遇到这样的调用时，不降级主体，而是直接把它转换为 PrimOp / MIR。

```nabi
impl Add for i32 {
    type Output = i32;

    #[intrinsic("nabi.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // an empty body is OK; the compiler uses the intrinsic path
        // even if a body exists, the compiler takes precedence
    }
}
```

自由函数（`mem::size_of`、`ptr::addr_of`）和固有方法（`MaybeUninit::as_ptr`）以同样的方式带有 `#[intrinsic]`；对它们的调用就是内建函数。

编译器按如下方式处理。分派 `Add::add(a, b)` 调用时，它在 method_table 中找到 `i32::add`，然后检查该方法上是否带有 `#[intrinsic("nabi.add.i32")]`。如果有，就不降级主体，而是直接生成 `PrimOp::Add { ty: i32 }`；它随后经过 MIR 和代码生成，最终成为目标的 32 位 `add` 指令。

这一机制把编译器的拦截吸收进了 trait 分派中。

## 0.3.3. 内建函数命名约定

内建函数名有以下三种形式：

- `nabi.<OP>.<TYPE>`：针对一个类型的原始操作（`nabi.add.i32`），包括复合赋值 `nabi.<OP>_assign.<TYPE>`（`nabi.add_assign.i32`）；
- `nabi.<area>.<op>`：某一领域（如 `ptr` 或 `atomic`）的操作（`nabi.ptr.read`、`nabi.atomic.cas`）；
- `nabi.<name>`：名称中不带类型的泛型操作（`nabi.size_of`、`nabi.align_of`、`nabi.addr_of`、`nabi.drop_in_place`、`nabi.hand_over`），包括 GC 缓冲区和移交系列 `nabi.buf_<op>` 与 `nabi.transfer_<op>`（`nabi.buf_get`、`nabi.transfer_keep`）。

原始操作形式：

```
nabi.add.i32        i32 + i32
nabi.add.i64
nabi.add.f32
nabi.add.f64
nabi.sub.i32
nabi.mul.i32
nabi.div.i32
nabi.rem.i32
nabi.and.i32        bitwise AND
nabi.or.i32
nabi.xor.i32
nabi.shl.i32
nabi.shr.i32
nabi.eq.i32
nabi.lt.i32
...
nabi.neg.i32        unary
nabi.not.bool       logical not (bool)
nabi.not.i32        bitwise not (integer)
nabi.cast.i32.i64   type cast
```

原始操作形式的每个 OP 都与编译器的 `PrimOp` 枚举一一对应。名称本身沿用常见的指令名和 Rust 的内建函数命名约定。

## 0.3.4. 主体回退策略

带有内建函数属性的方法仍然可以有主体。下面的示例说明了这一点。

```nabi
impl Add for BigInt {
    type Output = BigInt;

    // BigInt is not fixed-width, so no intrinsic; the body is authoritative
    fn add(self, rhs: BigInt) -> BigInt {
        // actual BigInt addition implementation
        ...
    }
}

impl Add for i32 {
    type Output = i32;

    #[intrinsic("nabi.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // body: runs when taken out as an fn value or in environments that do not know the intrinsic
        self + rhs                    // primitive operators lower directly to machine operations
    }
}
```

规则总结如下。在启用内建函数的正常编译路径上，主体被忽略；在不识别该内建函数的实验性后端或解释执行环境中，实际使用的是主体。如果主体为空，它被视为桩 `panic("intrinsic not available")`。

把方法取出作为 `fn` 值（`let f = i32::add;`）时，调用该值会运行主体。内建函数只在直接调用处生效，所以内建方法的主体也必须是含义相同的正确实现。在主体内部，原始类型上的 `self + rhs` 本身就是对该内建函数的直接调用，所以它降级为机器操作，而不会再回到主体。

内建函数也可以是没有主体的声明（`pub fn size_of<T>() -> usize;`）。它没有回退：内建函数是它唯一的实现，只能通过直接调用使用。

## 0.3.5. 用户代码中的内建函数

`#[intrinsic(...)]` 是只在标准库与编译器耦合的边界上起作用的机制，所以用户代码不能直接加这个属性：无论是函数、泛型函数还是方法都不行。

```nabi
// user code: rejected.
#[intrinsic("nabi.add.i32")]
fn my_add(a: i32, b: i32) -> i32 { ... }
// error[E0115]: `#[intrinsic]` on `my_add`: the intrinsic attribute is for the standard library only
```

这一限制防止开出一个可以在标准库 trait 之外任意调用编译器后端的后门。

## 0.3.6. 与其他机制的比较

内建函数只是连接编译器与标准库的几种魔法之一；下表总结了各自的作用。

| 机制                  | 用途                                                             | 规范                                                                   |
|-----------------------|------------------------------------------------------------------|------------------------------------------------------------------------|
| `#[intrinsic("...")]` | 标准库方法 ↔ 编译器后端 PrimOp                                   | 本文档                                                                 |
| `#[lang(...)]`        | 标准库项 ↔ 编译器槽（none、iter、index 等）                      | [refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)     |
| `#[infix(...)]`       | 词法单元 ↔ 标准库 trait 方法                                     | [refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md) |
| `#[literal(...)]`     | 字面量形式 ↔ 标准库处理函数                                      | [refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md)  |

这四种魔法相互正交。`#[intrinsic]` 负责与编译器后端的耦合，其余几种都负责与语言语法的耦合。
