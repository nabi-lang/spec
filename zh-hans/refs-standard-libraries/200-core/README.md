# 2. Core 层级

core 层级（`core`）是安全的标准库，也是 Nabi 在所有目标上都能构建的实用接口。大约 90% 的用户代码会用到这一层级的项，因此它决定了 Nabi 在日常编程中的面貌。

## 位置

规范文档位于：

- 规范：`spec/en/refs-standard-libraries/200-core/`

## 依赖

向上，core 层级在内部依赖 `bare`；向下，`std` 和用户代码依赖它。因此 core 层级位于安全区域的中心，处在 `bare` 与 `std` 之间。

## 用户导入策略

`use core::prelude::*;` 是**隐式应用**的：编译器会把它加在每个用户文件的开头。prelude 中的名称（List、Dict、Display、Iterator、Mutex、task 等）无须显式导入即可使用；core 的其余部分通过 prelude 模块（`ops::Add`、`task::sleep`）或 `use` 访问。

需要时也可以使用范围较窄的显式导入。

```nabi
use core::collections::List;           // narrow, explicit
use core::ops::{Add, Sub};           // operator traits are not in the prelude
```

> [!NOTE]
> prelude 的列表是规范性的，其中不含枚举变体：裸变体由
> 期望类型解析（[refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-由期望类型确定的裸变体)）。

完整的策略见 [0.2 导入策略](../000-meta/002-import.md)。

## 模块索引

core 层级提供的模块按领域分组如下。每一项的末尾列出它所规定的 `core` 模块。

### 数值 / niche

- [2.1. 数值类型（定宽类型、`Int`、`Float`）](201-numbers.md)：`num`
- [2.2. niche 包装类型（`NonZero`、`NonNan`、`Finite`、`Positive`）](202-non-zero.md)：`num`

### 运算符 trait

- [2.3. 算术 / 位 / 逻辑运算（`Add`、`Sub`、`Mul`、`Div`、`Rem`、`Neg`、`Not`、`BitNot`、`BitAnd`、`BitOr`、`BitXor`、`Shl`、`Shr`、`LogicalAnd`、`LogicalOr`、`Index`）](203-ops.md)：`ops`、`index`
- [2.4. 复合赋值（`AddAssign`、`SubAssign`、……、`ShrAssign`）](204-ops-assign.md)：`ops`
- [2.5. 比较（`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash`、`Hasher`）](205-cmp.md)：`cmp`、`hash`
- [2.6. 广播（`DotAdd`、`DotMul`、……、`DotShr`）](206-broadcast.md)：`ops`

### iter / stream

- [2.7. iter（`Iterator`、`IntoIterator`、`Range`、`RangeInclusive`）](207-iter.md)：`iter`、`range`
- [2.8. `Stream`（异步迭代器）](208-stream.md)：`stream`

### 集合类型

- [2.9. 集合类型（`List`、`Dict`、`Set`、`Slice`、`Deque`、`BinaryHeap`、`SortedDict` / `SortedSet`、数组、元组）](209-collections.md)：`collections`、`array`、`tuple`

### 格式化 / 转换

- [2.10. fmt（`Display`、`Debug`、`Formatter`、`print`）](210-fmt.md)：`fmt`、`print`
- [2.11. 转换（`From`、`Into`、`TryFrom`、`TryInto`、`saturating_into`、`wrapping_into`）](211-conversion.md)：`convert`

### 元 / 标记

- [2.12. 反射（`Reflect`、`Inspect`）](212-reflect.md)：`reflect`
- [2.13. marker（`Clone`、`Copy`、`Drop`、`NoneSentinel`）](213-marker.md)：`marker`

### 错误 / 字符串

- [2.14. 错误（`Error`、`Result`、`TypeError`、`ValueError`、`OverflowError`、`panic`）](214-error.md)：`error`、`panic`
- [2.15. 字符串（`str`、`StrBuf`、`char`、`c_char`、Unicode 规范化与分段）](215-string.md)：`str`

### 并发 / 异步

- [2.16. 同步原语（`Atomic`、`Mutex`、`RwLock`、`Cell`）](216-sync.md)：`sync`
- [2.17. `Channel`：消息传递](217-channel.md)：`channel`
- [2.18. `task::`：M:N 调度器任务 API](218-task.md)：`task`
- `time`：`Duration` / `Instant`，与 std 的时钟一起在 [3.3](../300-std/303-time.md) 中规定
- [2.19. `par::`：并行处理 API](219-par.md)：`par`
- [2.20. `Future` / `await` 集成](220-future.md)：`future`

### 密码学

- [2.21. `crypto::`：哈希、MAC、AEAD、密钥交换、签名](221-crypto.md)：`crypto`

### 编码

- [2.22. `encoding::`：hex 与 base64](222-encoding.md)：`encoding`

### I/O

- [2.23. io（`Read`、`BufRead`、`Write`、`Seek`、`AsyncRead`、`AsyncWrite`、`BufReader`、`Cursor`、标准输入输出）](223-io.md)：`io`

### SIMD

- [2.24. simd（`Simd`、`Mask`、`f32x4` 等）](224-simd.md)：`simd`

### 基准测试 / 优化器提示

- `bench`：`Bench`，即 `#[bench]` 函数接收的句柄。测量方式在 [refs-lang 9.9.11](../../refs-lang/009-tools/909-debug-tools.md#9911-基准测试) 中规定。

  ```nabi
  pub struct Bench { /* private fields */ }   // Debug: `Bench { name: "parse_1kb", .. }`
  impl Bench {
      pub fn iter<T>(ref self, f: fn() -> T);  // calibrate, warm up, sample and report f
      pub fn set_bytes(ref self, n: i64);      // bytes per call: adds a throughput column
      pub fn set_items(ref self, n: i64);      // items per call: adds a throughput column
  }
  ```
- `hint`：`black_box<T>(x: T) -> T`：原样返回 `x`，但优化器无法看穿它，因此既不会把产生 `x` 的计算折叠掉，也不会把对结果的使用折叠掉。基准测试用它包住计时区域的输入和结果。

> [!NOTE]
> 可移植的 SIMD 向量属于 core 层级（[2.24](224-simd.md)）。张量、自动微分、GPU 等科学计算领域不属于 core 层级，而是拆分为**外部库**。外部库通过后端接口接入编译器的后端，该接口计划在以后的版本中提供。

## Core 的设计原则

core 层级的设计遵循四个方向。

第一，trait 优先。所有语义上的决定都由 trait 方法分派完成：`+` 解析为 `Add::add`，`[i]` 索引解析为 `Index::index`。

第二，转换是显式的。可能丢失信息的转换（如 `i64 → i32`）会以编译错误或运行时错误拒绝，饱和、回绕等策略必须按每个操作写明（[2.11](211-conversion.md)）。

第三，这一层级以**对 GC 友好**为目标。集合类型和并发对象由 GC 管理。`Drop` 资源是仿射的：它在最后一个所有者离开作用域处或其 `using` 块结束处释放（[2.13.4](213-marker.md)）。

最后，保证**目标无关性**。每一项在所有支持的目标上行为都相同，依赖目标的项拆分到 `std` 中。
