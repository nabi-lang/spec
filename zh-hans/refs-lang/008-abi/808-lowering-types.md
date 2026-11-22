# 8.8. 降级：数据类型与表达式

本节定义 Nabi 的值、类型和表达式在 `nabic` 生成的代码中如何表示。布局用 Nabi 类型书写：`(A, B)` 是位于各自自然偏移处的一对字段，`[T; n]` 是数组，`ptr` 是指针宽度的地址，`p` 是以字节计的指针宽度。代码为伪代码（[8.7.1](807-code-generation.md#871-翻译流程)）。

## 8.8.1. 基本整数 / 浮点数

基本数值类型是对应宽度的机器值。

| Nabi 类型          | 表示                                                                    |
|-------------------|-------------------------------------------------------------------------|
| `i8` ~ `i64`      | 1 ~ 8 字节，二进制补码                                                  |
| `u8` ~ `u64`      | 1 ~ 8 字节；有无符号由运算决定                                          |
| `usize` / `isize` | 目标的指针宽度                                                          |
| `f32` / `f64`     | IEEE 754 binary32 / binary64                                            |
| `bool`            | 一个字节，`0` 或 `1`                                                    |
| `char`            | 4 字节（一个 Unicode 标量值）                                           |
| `c_char`          | 一个字节                                                                |

`str` 是按值传递的不可变对 `(ptr, usize)`（一个指针和一个字节长度）。字面量指向永生的常量；运行时创建的字符串指向某个 GC 对象的载荷（类型 id `1`，[8.2.12](802-scan-table.md#8212-运行时保留的类型-id)）。

- `s[a:b]` 是 `(ptr + a, b - a)`，与原字符串共享字节（O(1)）。
- `ptr` 可以指向 GC 对象的任意字节（包括末尾的 NUL），并使整个对象保持存活。GC 把它视为内部指针，疏散时会保持偏移不变地重写它。
- 字面量和运行时新创建的字符串在其字节之后有一个 NUL。切片可能没有。

## 8.8.2. 算术运算

在基本类型上，[运算符分派](../003-expressions/301-operator-dispatch.md)中的运算符是操作数宽度的单条指令：加、减、乘，按操作数类型的有符号或无符号除法与取余，以及有符号、无符号或有序的浮点比较。

```
a + b       ->  add.i32 a, b            ; with overflow checks: add, then branch to the panic on overflow
a / b       ->  div.s32 a, b            ; after the checks for b == 0 and MIN / -1
a < b       ->  lt.s32 a, b             ; lt.u32 for unsigned, lt.f64 (ordered) for floats
```

带检查的运算在结果放不下时分支到其 panic（`-C overflow-checks`，[9.1.2](../009-tools/901-nabic-cli.md#nabic-buildaot-构建)）；优化构建会去掉它能证明不会失败的检查（[8.7.3](807-code-generation.md#873-优化级别)）。用户定义的 trait 实现在泛型单态化之后是函数调用。

## 8.8.3. `struct` 的表示

`struct` 是值类型（[2.3.8](../002-types/203-struct.md#238-值语义)），没有对象头。

```nabi
struct Point { x: f64; y: f64; }
```

它的表示就是按 [2.3.6](../002-types/203-struct.md#236-内存布局) 的布局排列的字段，包括 `#[repr]` 的对齐和偏移。

```
(f64, f64)        ; x at 0, y at 8; size 16, align 8
```

字段相同的两个 struct 具有相同的布局。struct 值位于寄存器、栈槽中，或内嵌在其容器中：另一个 struct 的字段、`List` 缓冲区的元素、闭包环境或协程帧的槽。赋值和实参传递会复制字段。编译器只有在装箱无法被观察到的地方，才可以在内部对值装箱。

### 字段访问

`p.x` 读取位于其偏移处的字段：从 struct 的槽或内嵌位置加载，或者读取持有该字段的寄存器。

```
val = load.f64 [p + 0]
```

### 字段写入

`p.x = 1.0f64` 更新变量自己的副本：向其槽中该字段的偏移处存储。

```
store.f64 [p + 0], 1.0
```

当 struct 内嵌存储在 GC 堆对象中且字段持有 GC 引用时，存储要经过写屏障（[屏障与根](../007-runtime/704-barriers-and-roots.md)）。位于栈槽中的 struct 的指针字段是 GC 根（[7.4](../007-runtime/704-barriers-and-roots.md)）。

## 8.8.4. 元组 `struct` 的表示

考虑元组 `struct`：

```nabi
struct Pair(i32, i32);
```

它的表示与普通 `struct` 相同，只是字段按位置索引而不是按名称访问。

## 8.8.5. 元组类型

元组类型

```nabi
(i32, str, bool)
```

布局为 `(i32, (ptr, usize), bool)`：`str` 无论出现在哪里，都是它的那一对（8.8.1）。与 `struct` 一样，它是没有头的值。

## 8.8.6. `enum` 的表示

[`enum`](../002-types/204-enum.md) 中的 `enum` 是没有对象头的值，表示为带标签的 `union`：与和类型（8.8.8）相同的 `(i32 tag, payload)` 布局。

```nabi
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

它表示为：

```
(i32, [u64; 2])    ; 64-bit target: tag at 0, payload at 8 (16 bytes, 8-aligned)
```

标签持有变体的索引（`Ok`=0，`Err`=1，`Pending`=2）。载荷是带数据的各变体的 union：其大小和对齐等于其中最大者，每个变体的载荷以其自身布局存储在起始处。可能持有 GC 引用的载荷是一串指针宽度的字，回收器依据标签扫描它（[8.2](802-scan-table.md)，`SUM_PAYLOAD`）。

没有字段的 enum 只有标签：一个 `i32`，或其 `#[repr(..)]` 指定的整数。

### 变体构造

`Result::Ok(42)` 在寄存器或目标槽中构建值：不分配。

```
store.i32 [v + 0], 0        ; tag: Ok
store.i32 [v + 8], 42       ; the payload in its own type
```

由较短变体的载荷构建的 union 值，在该载荷之后的字节中为零，因此从中复制出的根字要么为 null，要么是引用，绝不会是陈旧的栈内容（[8.9.12](809-lowering-control.md#8912-影子栈帧)）。

### 变体匹配

匹配变体：

```nabi
match v {
    Ok(n) => ...,
    Err(s) => ...,
    Pending => ...,
}
```

按分支顺序把标签与每个分支的变体索引比较（8.9.3）；没有分支匹配的值会到达一个不可达块。

```
bb0:
    tag = v.0
    if tag == 0 then bb_ok else bb1
bb1:
    if tag == 1 then bb_err else bb2
bb2:
    if tag == 2 then bb_pending else bb_unreachable
bb_ok:
    n = v.1 as i32          ; the payload read as the variant's type
```

## 8.8.7. （无标签）`union` 的表示

考虑 [`union`](../002-types/205-union.md) 中的 `union`。

```nabi
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

它的大小和对齐等于其最大字段（这里为 8 字节），每个字段都从偏移 0 开始。字段访问按该字段的类型读写同一块内存。在源代码中，字段读取必须位于 `unsafe` 之内（[2.5](../002-types/205-union.md)）。

## 8.8.8. 和类型（`A | B`）的表示

[和类型](../002-types/206-sum-types.md)中的和类型按如下方式处理。

```nabi
type X = i32 | f64
```

它表示为 `(i32, u64)`，即一个标签和一个载荷槽，构建方式与 `enum`（8.8.6）相同：这里载荷持有 `i32` 或 `f64`。`none` 成员为标签 0，不带载荷，因此清零的内存读回来就是 `none`。

若适用 niche 优化（[niche 优化](../002-types/207-niche-optimization.md)），标签槽会消失，只留下载荷的位模式。

```nabi
Ptr<u8> | none
```

这个类型是单个指针：一个自动 niche，其中 null 表示 `none`。

## 8.8.9. `any` 槽的表示

考虑 [`any` 槽](../002-types/202-any-slot.md)中的动态槽。

```nabi
let x: any = ...
```

它是一个 `(u32 tag, payload)` 对。`any Tr` 和裸 `any` 的表示相同。

```
(u32, ptr)
```

标签的低 31 位标识值的类型：类型名的 31 位 FNV-1a 哈希，因此单独编译的各个包能取得一致（同一构建中发生冲突时取下一个空闲值）。不宽于指针的数值、`bool` 和 `char` 值以其位直接存储在载荷字中。其他值是 GC 堆上的箱，载荷就是该箱，并且标签的第 31 位（`ANY_BOXED`）被置位；GC 在扫描 `any` 槽时读取该位（[8.2.7](802-scan-table.md)）。

### `any` 上的类型测试

对 `any` 值的类型模式

```nabi
match v {
    n: i32 => n + 1,
    s: str => s.len() as i32,
    _ => 0,
}
```

会屏蔽掉 `ANY_BOXED`，并把标签与每个模式类型的标签比较，形成一串双路分支：

```
bb0:
    t = v.0 & 0x7fffffff
    if t == TAG_I32 then bb_i32 else bb1
bb_i32:
    n = v.1 as i32          ; the payload word's low 32 bits
bb1:
    if t == TAG_STR then bb_str else bb_other
bb_str:
    s = load.str [v.1 + 16] ; a boxed `str`: the pair from the box payload
```

这些运行时测试就是 `any` 槽的开销。

## 8.8.10. 数组类型：`[T; N]`

考虑[类型表达式](../002-types/201-type-expressions.md)中的定长数组。

```nabi
[i32; 8]
```

它是连续的 8 个 `i32` 元素，共 32 字节。在 GC 对象内部，数组前面有一个 16 字节的头。

## 8.8.11. 函数类型：`fn(...) -> ...`

考虑一个[函数类型](../004-functions/404-function-type.md)。

```nabi
fn(i32, i32) -> i32
```

每个函数值，无论是具名函数还是闭包，都是一个 `{code, env}` 对：

```
(ptr, ptr)        ; code, env
```

`code` 把 `env` 作为第一个实参，然后是声明的实参，不接收 TLS 地址（[8.6.11](806-calling-convention.md#8611-函数指针-abi)）。作为值使用的具名函数，其 `env` 为 null，代码指针指向一个丢弃 `env` 的 thunk。只有 C 函数类型（`extern "C" fn(..)`）是裸代码指针。

## 8.8.12. 闭包的表示

考虑一个带捕获的[闭包](../004-functions/405-closure.md)。

```nabi
let n = 10;
let f = |x| x + n;       // captures n
```

它是一个函数值（8.8.11），其 `env` 是持有各捕获的 GC 对象，每个捕获一个字段，以 struct 的形式位于 16 字节的头之后：

```
env object: header (16 bytes), then (i32)     ; a copy of n
f = (code of the closure body, env)

; on call
r = call f.0(f.1, x)
```

不捕获任何东西的闭包，其 `env` 为 null，不分配任何内存。

## 8.8.13. 标识符 / 变量访问

考虑[变量声明](../005-modules/501-variable-declaration.md)中的局部变量。

```nabi
let counter = 0i32;
counter = counter + 1i32;
```

每个局部变量都有一个栈槽，赋值就是向它存储：

```
store.i32 [counter], 0
c = load.i32 [counter]
c2 = add.i32 c, 1
store.i32 [counter], c2
```

在 `-O0`，每个值都留在其槽中。在 `-O0` 以上，从不被取地址、从不被整体复制、也不向调试器描述的局部变量可以改为放在寄存器中，优化器会把已知的值转发给对它的读取（[8.7.3](807-code-generation.md#873-优化级别)）。持有 GC 引用的局部变量总是保留其槽：影子栈帧持有该槽的地址（8.9.12）。

## 8.8.14. 字面量脱糖

考虑 [`#[literal]` 魔法](../006-meta-system/605-literal-magic.md)中的集合类型字面量。

```nabi
[1, 2, 3]
```

它脱糖为对标准库函数 `list_literal(1, 2, 3)` 的调用，并遵循普通函数调用的降级。

字符串和数值字面量作为编译期 `const` 处理，存储在 .rodata 中，然后作为指针或值使用。
