# 0.4. `#[lang]` 槽映射表

`#[lang(slot_name)]` 属性（[refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)）把标准库的项注册到编译器可以查找的**槽**中。本章以表格形式把所有槽名、其含义以及它所附加的项集中在一处。

槽表是规范中的权威位置；编译器、标准库和用户都以它为准来保持一致。槽名一旦出现偏差，语言与库之间的约定就会失效，所以任何变更都从这张表开始。

## 0.4.1. 槽的分类

100 个槽按职责分为四组。下表一目了然地列出每组的作用和槽数，并标明后续各节的详细表属于哪个类别。

| 分组    | 职责                                                                        | 槽数    |
|---------|-----------------------------------------------------------------------------|---------|
| 类型    | Nabi 识别为**具体类型**的标准库类型                                          | 20      |
| trait   | Nabi 用于**运算符 / 控制流脱糖**、derive 以及线程边界检查的 trait            | 37 |
| 方法    | Nabi **自动调用**的方法（drop）                                              | 1       |
| 函数    | Nabi 以**内建调用**方式调用的标准库函数，以及降级后的代码所调用的钩子        | 42 |
| 合计    |                                                                             | **100** |

## 0.4.2. 类型槽（20 个）

类型槽指明编译器遇到字面量或关键字时要降级到的具体类型。这一组又细分为两类。第一类是以小写标识符直接暴露的语言默认类型。它们不需要另外注册关键字，是用户每天都会遇到的原始类型。

语言默认类型（内置原始类型，不需要 `#[keyword]`）：

| 槽名         | 含义                         | 标准库项                 | 参见 |
|--------------|------------------------------|--------------------------|------|
| `i32_type`   | `i32` 原始类型               | 内置（方法：`core::num`） | 2.1  |
| `i64_type`   | `i64` 原始类型               | 内置（方法：`core::num`） | 2.1  |
| `f32_type`   | `f32` 原始类型               | 内置（方法：`core::num`） | 2.1  |
| `f64_type`   | `f64` 原始类型               | 内置（方法：`core::num`） | 2.1  |
| `usize_type` | `usize` 原始类型             | 内置（方法：`core::num`） | 2.1  |
| `isize_type` | `isize` 原始类型             | 内置（方法：`core::num`） | 2.1  |
| `bool_type`  | `bool` 原始类型              | 内置（方法：`core::num`） | 2.1  |
| `cchar_type` | `c_char` 原始类型（FFI）     | 内置（实现：`core::num`） | 2.15 |
| `str_type`   | `str` 原始类型（UTF-8 视图） | 内置（方法：`core::str`） | 2.15 |
| `char_type`  | `char` 原始类型（Unicode 标量值） | 内置（方法：`core::str`） | 2.15 |

第二类是与语言语法绑定的槽：自动提升的 `int` 和 `float`、存储用浮点数 `f16` 和 `bf16`、`none` 哨兵值，以及 `Range` 字面量。`int`、`float`、`f16` 和 `bf16` 是 PascalCase 命名的库结构体，通过 `#[keyword(...)]` 暴露小写名称；`none` 是内置的，库用 `pub type NoneSentinel = none` 把它的类型命名为 `NoneSentinel`。

与语法绑定的槽：

| 槽名         | 含义                          | 标准库项                                                  | 参见 |
|--------------|-------------------------------|-----------------------------------------------------------|------|
| `none_type`  | `none` 字面量的类型           | 内置（别名：`core::marker::NoneSentinel`） | 2.13 |
| `int_type`   | `int` 自动提升类型            | `core::num::bigint::Int` + `#[keyword("int")]` | 2.1  |
| `float_type` | `float` 自动提升类型          | `core::num::bigfloat::Float` + `#[keyword("float")]` | 2.1  |
| `f16_type`   | `f16` 存储用浮点数（IEEE 754 binary16） | `core::num::half::F16` + `#[keyword("f16")]` | 2.1  |
| `bf16_type`  | `bf16` 存储用浮点数（bfloat16） | `core::num::half::BF16` + `#[keyword("bf16")]` | 2.1  |
| `range_type` | `Range<T>`（`a..b`）          | `core::range::Range` + `#[literal( expr ".." expr )]` | 2.7  |
| `slice_arg_type` | 多部分索引中的切片部分 `a:b:k` | `core::index::SliceArg` | 2.3  |
| `rest_axis_type` | 轴索引中与其他部分并列的 `..` | `core::index::Rest` | 2.3  |
| `new_axis_type`  | 轴索引中的 `none`             | `core::index::NewAxis` | 2.3  |
| `ndarray_type`   | 位于其槽中的嵌套列表字面量    | `core::array::Array` | 2.9.14 |

内置提供原始类型的编译器可以不附加 `i32_type` ... `str_type`、`none_type` 和 `char_type`。参考编译器就是这样做的：这些槽名被接受，但库不给它们附加任何项（`str` 和 `c_char` 不是库结构体，`NoneSentinel` 是别名而不是 `#[lang]` 项）。在 0.4.6 查找其中某个槽的地方，编译器使用其内置类型。

## 0.4.3. trait 槽（37 个）

trait 槽指明把用户编写的语法脱糖为标准库 trait 调用时，编译器要找哪个 trait。`for` 循环、索引、切片、格式化和 `await` 等核心语法都通过这里注册的 trait 来解析，所以这张表是语言语法与库之间最粗的桥梁。

| 槽名                  | 含义                            | 标准库 trait                    | 参见 |
|-----------------------|---------------------------------|---------------------------------|------|
| `iterator_trait`      | `for x in xs` 中的步进          | `core::iter::Iterator`     | 2.7  |
| `into_iterator_trait` | `for x in xs` 的开始            | `core::iter::IntoIterator` | 2.7  |
| `try_iterator_trait`  | `try for x in xs` 中的步进      | `core::iter::TryIterator`  | 2.7.13 |
| `index_trait`         | `arr[i]` 读取                   | `core::index::Index` | 2.3  |
| `index_set_trait`     | `arr[i] = v` 写入               | `core::index::IndexSet` | 2.3  |
| `slice_trait`         | `s[1:3]` 切片                   | `core::index::Sliceable` | 2.3  |
| `display_trait`       | `f"{x}"` 默认格式               | `core::fmt::Display`       | 2.10 |
| `debug_trait`         | `f"{x:?}"` 调试格式             | `core::fmt::Debug`         | 2.10 |
| `future_trait`        | `await expr` 中的步进           | `core::future::Future`     | 2.20 |
| `axis_indexed_trait`  | 把索引部分作为轴参数            | `core::index::AxisIndexed` | 2.3  |
| `broadcastable_trait` | 融合点表达式的操作数            | `core::ops::Broadcastable` | 2.6.6 |
| `broadcast_scalar_trait` | 融合点表达式中的标量包装     | `core::ops::BroadcastScalar` | 2.6.6 |
| `materialize_trait`   | 融合点表达式的结果              | `core::ops::Materialize`   | 2.6.6 |
| `close_trait`         | 可能失败的关闭：由所有者关闭    | `core::io::Close`     | 2.23 |

接下来是运算符 trait 槽。运算符词法单元的分派目标由各 trait 方法上的 `#[infix]` / `#[prefix]`（[refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md)）决定，优先级由 [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) 中的表固定；下面这些槽是固定位置，让编译器在原始类型降级和诊断时能按名称找到“加法 trait”“比较 trait”等。复合赋值（`AddAssign` 等，2.4）、广播（`DotAdd` 等，2.6）、幂（`Pow`，`**`）和向下取整取模（`Modulo`，`%%`）trait 只需注册 `#[infix]`，没有槽：降级和诊断从不按名称查找它们。在原始类型上，`**` 调用普通方法（`pow`、`powi`），`%%` 是 `#[intrinsic]` 方法（[0.3](003-intrinsics.md)），所以它们的降级也不需要槽。

| 槽名                | 含义                                     | 标准库 trait                 | 参见 |
|---------------------|------------------------------------------|------------------------------|------|
| `add_trait`         | `a + b`                                  | `core::ops::Add`        | 2.3  |
| `sub_trait`         | `a - b`                                  | `core::ops::Sub`        | 2.3  |
| `mul_trait`         | `a * b`                                  | `core::ops::Mul`        | 2.3  |
| `div_trait`         | `a / b`                                  | `core::ops::Div`        | 2.3  |
| `rem_trait`         | `a % b`                                  | `core::ops::Rem`        | 2.3  |
| `neg_trait`         | `-a`（前缀）                             | `core::ops::Neg`        | 2.3  |
| `not_trait`         | `!a`                                     | `core::ops::Not`        | 2.3  |
| `bitnot_trait`      | `~a`                                     | `core::ops::BitNot`     | 2.3  |
| `bitand_trait`      | `a & b`                                  | `core::ops::BitAnd`     | 2.3  |
| `bitor_trait`       | `a \| b`                                 | `core::ops::BitOr`      | 2.3  |
| `bitxor_trait`      | `a ^ b`                                  | `core::ops::BitXor`     | 2.3  |
| `shl_trait`         | `a << b`                                 | `core::ops::Shl`        | 2.3  |
| `shr_trait`         | `a >> b`                                 | `core::ops::Shr`        | 2.3  |
| `logical_and_trait` | `a && b`                                 | `core::ops::LogicalAnd` | 2.3  |
| `logical_or_trait`  | `a \|\| b`                               | `core::ops::LogicalOr`  | 2.3  |
| `call_trait`        | 对非函数值的 `v(a)`                      | `core::ops::Call`       | 2.3  |
| `eq_trait`          | `==` / `!=`、字面量模式                  | `core::ops::PartialEq` | 2.5  |
| `cmp_trait`         | `<` / `>` / `<=` / `>=`、区间模式        | `core::ops::PartialOrd` | 2.5  |
| `ord_trait`         | 全序（`#[derive(Ord)]`）                 | `core::cmp::Ord`        | 2.5  |
| `hash_trait`        | 哈希（`#[derive(Hash)]`）                | `core::cmp::Hash`       | 2.5  |
| `clone_trait`       | 克隆（`#[derive(Clone)]`）               | `core::marker::Clone`   | 2.13 |
| `send_trait`        | 自动标记：可以安全地跨线程移动           | `core::marker::Send`   | 2.13 |
| `sync_trait`        | 自动标记：可以安全地跨线程共享           | `core::marker::Sync`  | 2.13 |

> [!NOTE]
> 比较类的槽名与 trait 名并非一一对应。`eq_trait` 附加在定义相等运算符的 `PartialEq` 上，`cmp_trait` 附加在定义顺序运算符的 `PartialOrd` 上。标记 trait `Eq`、`Copy` 和 `Default` 没有槽：和类型按结构获得它们（`T` 是 `Eq` 时 `T | none` 也是 `Eq`；它的 `Default` 是 `none`），因此编译器通过它们在 `core` 中的固定路径找到它们。

## 0.4.4. 函数槽（42 个）

函数槽用于编译器像调用内建函数一样直接调用的标准库函数。它们主要涵盖异常终止路径（`throw`、OOM、unreachable），用于在正常流程之外需要统一入口点的地方。

| 槽名             | 含义                                     | 标准库函数                      | 参见 |
|------------------|------------------------------------------|---------------------------------|------|
| `panic_fn`       | panic 时调用的中止函数                   | `core::panic::panic`       | 2.14 |
| `oom_fn`         | OOM 中止（运行时分配失败时直接调用 `nabi_oom`） | `core::panic::oom` | 2.14 |
| `unreachable_fn` | 编译器在不可达位置调用                   | `core::panic::unreachable` | 2.14 |

其余三十九个是**钩子**：编译器生成的代码会调用的函数，用于负索引、切片边界、可写视图、展开、f-string 占位、取消点、集合上的广播、嵌套数组字面量和融合点表达式。降级通过槽而不是路径找到每个钩子，所以钩子不必对它被降级进去的代码可见。钩子不是 API：每个钩子在 `core` 中都是 `pub(pkg)`，在其他包中手写的调用是错误。

| 槽名                 | 调用时机                                               | 标准库函数                                | 参见 |
|----------------------|--------------------------------------------------------|-------------------------------------------|------|
| `spread_len_fn`      | `List` / `Slice` 展开 `f(..xs)` 检查其长度             | `core::index::spread_len`            | 2.3  |
| `spread_names_fn`    | `Dict` 展开 `f(..d)` 检查其键                          | `core::index::spread_names`          | 2.3  |
| `spread_misfit_fn`   | `Dict` 展开中无法填入参数的键                          | `core::index::spread_misfit`         | 2.3  |
| `index_from_end_fn`  | 有符号索引 `xs[i]`，负数从末尾算起                     | `core::index::index_from_end`        | 2.3  |
| `slice_bound_fn`     | 有符号切片边界 `xs[i:]`                                | `core::index::slice_bound`           | 2.3  |
| `slice_range_fn`     | 带步长的切片 `xs[a:b:k]`                               | `core::index::slice_range`           | 2.3  |
| `slice_mut_fn`       | 在期望 `SliceMut` 的位置使用列表的 `xs` 或 `xs[a:b]` | `core::index::slice_mut`（`pub(pkg)`） | 2.9  |
| `view_live_fn`       | 在列表或视图上的 `for ref` 循环的每一步：视图过期时触发 panic | `core::index::view_live`（`pub(pkg)`） | 2.9  |
| `fstr_display_fn`    | f-string 占位 `{v}`                                    | `core::fmt::fstr_display`            | 2.10 |
| `fstr_debug_fn`      | f-string 占位 `{v:?}`                                  | `core::fmt::fstr_debug`              | 2.10 |
| `fstr_num_fn`        | 带格式说明符的 `int` 或 `float` 占位                   | `core::fmt::fstr_num`                | 2.10 |
| `fstr_pack_fn`       | 格式说明符由计算得出的占位 `{v:{spec}}`                | `core::fmt::fstr_pack`               | 2.10 |
| `fstr_text_any_fn`   | 非数值上的计算得出的格式说明符                         | `core::fmt::fstr_text_any`           | 2.10 |
| `fstr_args_fn`       | `Arguments` f-string：覆盖其各片段的闭包               | `core::fmt::fstr_args`               | 2.10 |
| `fstr_put_str_fn`    | `Arguments` f-string：字面量片段，或已转为文本的占位   | `core::fmt::fstr_put_str` | 2.10 |
| `fstr_put_display_fn` | `Arguments` f-string：占位 `{v}`                      | `core::fmt::fstr_put_display`        | 2.10 |
| `fstr_put_debug_fn`  | `Arguments` f-string：占位 `{v:?}`                     | `core::fmt::fstr_put_debug`          | 2.10 |
| `cancel_point_fn`    | 轮询得到 `Pending` 的 `await`（取消点）                | `core::task::cancel_point`           | 2.18 |
| `cancel_left_fn`     | 轮询结果为 `Ready` 的 `await`                          | `core::task::cancel_left`            | 2.18 |
| `cleanup_running_fn` | 因取消而离开的帧进入其清理                             | `core::task::cleanup_running`        | 2.18 |
| `cleanup_done_fn`    | 该帧完成了其清理                                       | `core::task::cleanup_done`           | 2.18 |
| `broadcast_len_fn`   | 广播 `xs .+ ys` 检查长度是否一致                       | `core::collections::broadcast::check_len` | 2.6 |
| `axis_from_u64_fn`   | 轴索引中的无符号位置                                   | `core::index::axis_from_u64`         | 2.3  |
| `axis_from_i64_fn`   | 轴索引中的宽有符号位置                                 | `core::index::axis_from_i64`         | 2.3  |
| `array_literal_fn`   | 位于 `Array` 槽中的嵌套列表字面量                      | `core::array::array_literal`         | 2.9.14 |
| `fused_shape_fn` | 融合点表达式：操作数拉伸到的形状 | `core::array::fused_shape` | 2.6.6 |
| `fused_view_fn` | 融合点表达式：以该形状查看的操作数 | `core::array::fused_view` | 2.6.6 |
| `fused_alloc_fn` | 融合点表达式：新的结果数组 | `core::array::fused_alloc` | 2.6.6 |
| `fused_flat_fn` | 融合点表达式：视图是连续的 | `core::array::fused_flat` | 2.6.6 |
| `fused_inner_fn` | 融合点表达式：一行（最后一个轴）的长度 | `core::array::fused_inner` | 2.6.6 |
| `fused_rows_fn` | 融合点表达式：行数 | `core::array::fused_rows` | 2.6.6 |
| `fused_row_fn` | 融合点表达式：一行在缓冲区中的位置 | `core::array::fused_row` | 2.6.6 |
| `fused_step_fn` | 融合点表达式：沿最后一个轴的步长 | `core::array::fused_step` | 2.6.6 |
| `fused_at_fn` | 融合点表达式：连续段中的一个元素 | `core::array::fused_at` | 2.6.6 |
| `fused_put_fn` | 融合点表达式：向连续段写入一个元素 | `core::array::fused_put` | 2.6.6 |
| `fused_get_fn` | 融合点表达式：跨步行中的一个元素 | `core::array::fused_get` | 2.6.6 |
| `fused_set_fn` | 融合点表达式：向跨步行写入一个元素 | `core::array::fused_set` | 2.6.6 |
| `fused_dest_fn` | 融合点表达式：原地结果与其目标相符 | `core::array::fused_dest` | 2.6.6 |
| `fused_source_fn` | 融合点表达式：与目标重叠的原地源会被复制 | `core::array::fused_source` | 2.6.6 |
| `fused_list_alloc_fn` | 列表上的融合点表达式：按索引填充的结果列表 | `core::collections::broadcast::fused_list_alloc` | 2.6.6 |

### 层级共享的项：`#[internal]`

有几个 `pub` 项不是由编译器调用，而是由工具链的另一个层级调用：`std` 使用 `core` 中的这些项，`core` 使用 `bare` 中的这些项。它们标记为 `#[internal]`（[refs-lang 6.2.15](../../refs-lang/006-meta-system/602-built-in-attributes.md#6215-internal-由工具链的各个包共用)）：`bare`、`core` 和 `std` 可以使用它们，其他任何包都会得到错误“internal to the Nabi toolchain”，文档和语言服务器也会略去它们。

| 项                                     | 使用者                                                    | 参见 |
|----------------------------------------|-----------------------------------------------------------|------|
| `core::marker::hand_over`         | `std::thread`（移交线程的结果）                       | 2.13 |
| `core::marker::Handoff::new`     | `std::thread`（线程的结果传回给 join 方）              | 2.13 |
| `core::marker::view`              | `std::net`（监听器的 `incoming` 视图）                | 2.13 |
| `core::task::wake_io`             | `std::net`（关闭或 shutdown 后唤醒等待者）            | 2.18 |
| `core::task::host_enter`          | `std::wit`（组件导出进入宿主循环）                    | 2.18 |
| `core::task::host_run`            | `std::wit`（宿主循环的一轮）                          | 2.18 |
| `core::task::host_leave`          | `std::wit`（结束 `host_enter`）                       | 2.18 |
| `core::task::readable` / `writable` | `std::net`（等待套接字就绪）                          | 2.18 |
| `core::io::IoError::retold`       | `std::fs`（针对某个路径重新表述的错误）               | 2.23 |
| `core::io::IoError::resolver`     | `std::net`（失败的名称查找）                          | 2.23 |
| `core::io::Owned::of`            | `std::net`（`TcpListener::into_incoming`）            | 2.23 |
| `core::io::Read::copy_handle` / `Write::copy_handle` | `std` 的 fs、net、process 和 `os::unix::net`（`io::copy` 交给内核的操作系统句柄） | 2.23 |
| `core::num::bigint::Int::from_literal` / `bigfloat::Float::from_literal` | `std::nabion`（数值字面量的数字） | 2.1 |
| `bare::gc::Pinnable::gc_object`   | `core::marker`（复制 `GcPtr` 的移交）                 | 1.6  |

## 0.4.5. 方法槽（1 个）

方法槽用于用户不写调用、由编译器自动调用的方法。只有一个：`using` 绑定被释放时的处置。`?` 运算符由编译器针对 `T | none` 直接展开（[refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410--运算符提前返回-none)），所以它没有方法槽。克隆值（`Clone::clone`）从不会被自动调用（[2.13.2](../200-core/213-marker.md#2132-clone深复制)），所以它注册为 trait 槽 `clone_trait`，而不是方法槽。

| 槽名          | 含义                                 | 标准库方法                      | 参见 |
|---------------|--------------------------------------|---------------------------------|------|
| `drop_method` | 在值被释放的位置调用（`using`，或其所有者离开作用域） | `core::marker::Drop::drop` | 2.13 |

## 0.4.6. 槽的使用方：编译器在哪里查找槽

前面几节列出的是“有哪些槽”，本节总结的是反方向：“编译器在什么时候查找哪个槽”。追踪一行用户语法会解析到哪个标准库项时，下表是起点。

| 编译器动作                                         | 查找的槽                                 |
|----------------------------------------------------|------------------------------------------|
| 降级 `none` 字面量                                 | `none_type`                              |
| `42` 的字面量类型（无期望类型）                    | `i64_type`                               |
| `3.14` 的字面量类型（无期望类型）                  | `f64_type`                               |
| 期望类型为 `int`、`float`、`f16` 或 `bf16` 的数值字面量，以及 `f16` / `bf16` 字面量后缀 | `int_type`、`float_type`、`f16_type`、`bf16_type` |
| `"hello"` 的字面量类型                             | `str_type`                               |
| 脱糖 `for x in xs`                                 | `into_iterator_trait`、`iterator_trait`、`try_iterator_trait` |
| 脱糖 `arr[i]`                                      | `index_trait`                            |
| 脱糖 `arr[i] = v`                                  | `index_set_trait`                        |
| 脱糖 `s[1:3]`                                      | `slice_trait`                            |
| 脱糖默认的 `f"{x}"` 占位                           | `display_trait`                          |
| 脱糖 `f"{x:?}"`                                    | `debug_trait`                            |
| 脱糖 `await expr`                                  | `future_trait`                           |
| 算术 / 位运算 / 逻辑运算符的原始类型降级           | `add_trait` ... `logical_or_trait` |
| `==` / `!=`、字面量模式匹配                        | `eq_trait`                               |
| `<` / `>` / `<=` / `>=`、区间模式匹配              | `cmp_trait`                              |
| 展开 `#[derive(Clone / Ord / Hash)]`               | `clone_trait`、`ord_trait`、`hash_trait` |
| 降级 panic（包括 `try!` 失败）                     | `panic_fn`                               |
| OOM 时                                             | `oom_fn`                                 |
| `match` 对所有分支做穷尽检查后的不可达位置         | `unreachable_fn`                 |
| 负索引或切片边界、带步长的切片                     | `index_from_end_fn`、`slice_bound_fn`、`slice_range_fn` |
| 在期望 `SliceMut` 的位置使用列表或其区间           | `slice_mut_fn`                          |
| 在列表或视图上的 `for ref` 循环的每一步            | `view_live_fn`                         |
| 展开 `f(..xs)` / `f(..d)`                          | `spread_len_fn`、`spread_names_fn`、`spread_misfit_fn` |
| f-string 占位                                      | `fstr_display_fn` ... `fstr_text_any_fn` |
| `await`，以及取消时运行的清理                      | `cancel_point_fn`、`cancel_left_fn`、`cleanup_running_fn`、`cleanup_done_fn` |
| 两个或更多集合上的广播                             | `broadcast_len_fn`                       |
| 释放值（`using` 块结束，或其最后一个所有者离开作用域） | `drop_method`              |
| 自动实现 `Send` / `Sync`、`+ Send` 检查            | `send_trait`、`sync_trait`        |

## 0.4.7. 重复 / 缺失检测

如果两个标准库项附加到同一个槽，编译器会拒绝（refs-lang 6.4.6 的重复检测规则）。反之，如果缺少 `iterator_trait` 这类必需的槽，依赖它的功能（这里是 `for` 循环）的编译会被拒绝。这两条都是迫使槽约定只有单一事实来源的手段。

实际附加的项在 [200-core/](../200-core/) 的各章中说明。每个已附加的 trait、方法、类型和中止函数槽，其 `#[lang(...)]` 附加位置都在“参见”列所指的章中。钩子 `_fn` 槽（`spread_len_fn` ... `fused_list_alloc_fn`）面向编译器，只在这张表中出现；0.4.2 中未附加的内置类型槽没有附加位置。不在这张表中的槽名，按 [refs-lang 6.4.8](../../refs-lang/006-meta-system/604-lang-magic.md) 是编译错误。例如，`Stream` 没有槽（[2.8.10](../200-core/208-stream.md)）。

## 0.4.8. 槽命名策略

最后是添加新槽或重命名现有槽时要遵循的命名规则。名称一致，只需扫一眼表就能猜出槽的种类，所以这一策略不能轻视。

名称使用 snake_case：类型槽带 `_type` 后缀，trait 槽带 `_trait` 后缀，函数槽带 `_fn` 后缀，方法槽带 `_method` 后缀（`drop_method`）。指代整个 trait 的槽统一为一个 `_trait`，即使编译器处理该 trait 的多个方法或 derive 也是如此；方法槽仅限于编译器**自动调用某一个特定方法**的情况。按这一标准，`Clone` 注册为 `clone_trait`，而不是 `clone_method`。后缀还决定附加位置：`_trait` 槽附加在 **trait 声明本身**上，`_method` 槽附加在 **trait 中对应的方法**上，`_type` 槽附加在类型声明上，`_fn` 槽附加在函数声明上；附加到其他位置会被 [refs-lang 6.4.7](../../refs-lang/006-meta-system/604-lang-magic.md) 的签名检查拒绝。
