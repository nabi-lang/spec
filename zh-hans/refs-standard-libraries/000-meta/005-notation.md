# 0.5. 记法

本规范的记法约定原样沿用 [refs-lang 0.8 记法](../../refs-lang/000-meta/008-notation.md)。本章定义只适用于标准库规范的附加约定。

## 0.5.1. trait 签名记法

每个 trait 规范都以一个签名块开头，把所有方法、关联类型和超 trait 集中在一处。

```nabi
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

这种形式与标准库的实际写法完全相同，所以用户可以在自己的代码中使用同样的格式。

## 0.5.2. 方法签名表

定义了多个方法的 trait 还会以表格形式总结，如下所示。

| 方法   | 签名                           | 含义                |
|--------|--------------------------------|---------------------|
| `add`  | `fn(self, rhs: Rhs) -> Output` | 算术加法            |
| `sub`  | `fn(self, rhs: Rhs) -> Output` | 算术减法            |

## 0.5.3. trait 实现矩阵

每个 trait 规范的末尾有一张表，说明哪些标准库类型实现了该 trait。列只有类型和绑定机制；实现文件的路径不在规范范围内。

```markdown
## X.Y.Z. Trait implementation matrix

| Type  | binding                          | Notes                          |
|-------|----------------------------------|--------------------------------|
| `i32` | `#[intrinsic("nabi.add.i32")]`    | Direct to compiler PrimOp      |
| `i64` | `#[intrinsic("nabi.add.i64")]`    |                                |
| `f32` | `#[intrinsic("nabi.add.f32")]`    |                                |
| `f64` | `#[intrinsic("nabi.add.f64")]`    |                                |
| `Int` | (no intrinsic, arbitrary precision) | Defined by the standard library |
```

## 0.5.4. 语义

每个方法的含义以文字写出，不仅包括形式和示例，还包括限制和错误情况。

```markdown
### `Add::add(self, rhs)`

Arithmetic addition. Returns the sum of the two values.

- `i32 + i32`: panics on overflow (checks on in every profile; wraps under `-C overflow-checks=no`)
- `f64 + f64`: IEEE 754 addition
- `Int + Int`: arbitrary precision; never overflows

Overflow handling follows the fixed-width arithmetic rules of [2.1 Numeric types](../200-core/201-numbers.md).
Saturating / checked / wrapping variants are exposed as separate methods (`saturating_add`, etc.).
```

## 0.5.5. 示例

每个 trait 和方法都附有一个或多个可运行的示例。示例假定隐式的 prelude 导入 `use core::prelude::*;` 已生效，所以它们必须是无需任何导入即可原样运行的代码。

```nabi
let a: i32 = 5;
let b: i32 = 3;
let c = a + b;        //→ 8 (i32)

let x: f64 = 1.5;
let y: f64 = 2.5;
let z = x + y;        //→ 4.0 (f64)
```

## 0.5.6. 交叉引用

引用其他文档时，根据位置使用两种形式之一。在本规范内部，写作 `[2.7 iter](../200-core/207-iter.md)`，或者非正式地称为“refs-standard-libraries 2.7”；引用 refs-lang 时采用 `[refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)` 的形式。

## 0.5.7. 规范与实现的分离

本文档只定义**规范**。编译器和标准库的实际实现在独立的仓库中维护；与本规范冲突时，以本规范为准。

## 0.5.8. `try_` 前缀

`try_` 名称只标记以下两种形式之一：

1. 非阻塞的尝试：立即做普通形式需要等待的事，并报告做不到的情况（`Mutex::try_lock`、`Receiver::try_recv`、`Sender::try_send`）。
2. 传递其所接收的闭包或 future 抛出的错误的形式（`Iterator::try_fold`、`Iterator::try_for_each`、`par::try_scope`、`task::try_block_on`）。

不能抛出错误的 trait 方法的抛出形式也保留 `try_` 名称：`TryFrom::try_from`、`TryInto::try_into`、`File::try_clone`。其他函数都不使用这个前缀：会失败的函数以其普通名称抛出错误（0.5.9），所以它没有 `try_` 孪生版本。`try_` 只作前缀；没有以 `_try` 结尾的名称。

## 0.5.9. 错误模型

每个函数只通过一种渠道报告失败，渠道按失败的性质选择：

| 失败 | 渠道 | 示例 |
|---|---|---|
| 只有一种原因的普通“没有结果” | `-> T \| none` | `List::get`、`Iterator::find`、`Path::parent`、`Path::strip_prefix`、`checked_*`、`char::from_u32`、`NonZero::new` |
| 由输入、环境或 I/O 引起，或有多种调用方可能需要区分的原因 | `throws (E)` | `FromStr::from_str`、`str::from_utf8`、`Int::parse_radix`、`FormatSpec::parse`、`ClientConfig::new`、`fs` 和 `net` 的操作 |
| 调用方可以事先廉价检查的调用方缺陷 | panic，`#[track_caller]` | 运算符索引越界、基数不在 2..=36 内、块大小为零 |

1. **没有孪生函数**。不存在只在失败渠道上不同的两个函数。`try? f()` 就是每个抛出调用的 `T | none` 形式（[refs-lang 3.14.3](../../refs-lang/003-expressions/314-error-control-flow.md)），所以库中没有抛出函数的 `_or_none` 孪生版本，也没有它的 panic 孪生版本。
2. **没有哨兵值**。结果从不用其自身类型的某个值（`0`、`-1`、`""`）编码失败：它要么是 `T | none`，要么函数抛出错误。
3. **关闭会抛出错误**。操作系统资源（文件、套接字、TLS 连接）的 `close()` 抛出操作系统报告的错误。`Drop` 从不抛出错误：由 `Drop` 释放的值会丢弃该错误。
4. 运算符与其具名方法可以不同：`xs[i]` 触发 panic 的地方，`xs.get(i)` 返回 `none`。
5. 运算符不能抛出错误（[2.3](../200-core/203-ops.md)）：可能失败的运算符返回 `Result`（`Finite<F> + Finite<F>` 是 `Result<Finite<F>, OverflowError>`）。


## 0.5.10. 错误类型

标准库中的每个错误类型都遵循以下规则：

1. **私有字段**。错误由构造函数（`new(..)`，或 `IoError::from_os_code` 这类具名构造函数）构建，从不使用结构体字面量；通过以事实命名的 getter 读取（`kind()`、`position()`、`path()`、`os_code()`）。只由库引发的错误（`LayoutError`、`NulError`）可以没有公开构造函数。由多种原因组成的枚举型错误（`FromBytesWithNulError`、`VarError`）把数据放在变体字段中，通过模式读取。每个错误类型都有 `message() -> str`：出了什么问题，不含其 `Display` 可能附加的上下文。
2. **trait 集合**。每个错误类型都实现 `Display`、`Debug`、`Error`、`PartialEq`、`Eq` 和 `Clone`。携带类型参数值的错误（`SendError<T>`、`TrySendError<T>`）实现前三个。
3. **种类**。种类 getter 返回枚举（`IoErrorKind`、`TlsErrorKind`、`JoinErrorKind`），从不返回 `str` 或整数，并且该枚举是 `#[non_exhaustive]`（`DecodeErrorKind` 也是）。操作系统错误码是 `os_code() -> i32 | none`，错误不是来自操作系统时为 `none`。
4. **单元结构体**。没有字段的错误是单元结构体（`struct TimeoutError;`），写作 `TimeoutError`，而不是 `TimeoutError {}`。
5. **每个失败空间一个错误**。core 保留共享的错误（`ValueError`、`ParseError`、`OverflowError`、`ConversionError`、`TypeError`、`Utf8Error`）以及它自身各失败空间的错误（`ZeroError`、`DecodeError`、`FmtError`、`CancellationError`、`TimeoutError`）；std 的模块每个失败空间有一个错误类型（`IoError`、`TlsError`、`NabionError`；`env` 有读取变量用的 `VarError` 和 `join_paths` 用的 `JoinPathsError`；`process` 在 `IoError` 之外还有安装处理函数用的 `SignalError`）。两个错误从不覆盖同一种失败。每次失败的 UTF-8 检查都抛出 `Utf8Error`。
6. **消息**。消息以小写开头（专有名词和类型名除外），没有结尾的句号，也没有 `error:` 前缀。运算符的 panic 使用 Rust 的文本（`index 5 out of bounds for length 3`）。方法的 panic 以 `Type::method: ` 开头（自由函数为 `module::fn: `），然后是问题，有替代写法时再接 `; <what to write instead>`：`i32::isqrt: the number is negative; use `checked_isqrt` to get `none` instead`。对错误实参触发 panic 的公开函数是 `#[track_caller]`。


## 0.5.11. 命名

标准库中的每个名称都遵循下表。违反它的名称是库的缺陷，而不是先例。

| 种类 | 规则 |
|---|---|
| 构造函数 | `new`（必需实参）、`with_<x>`（额外实参）、`from_<src>`（转换，`Formatter::from_sink`），特殊值用 `empty()` / `zeroed()`；单位构造函数以其单位命名（`Duration::secs(n)`、`ms`、`ns`）；`List::filled(n, x)`（n 个副本）和 `BinaryHeap::by_ordering(cmp)` 是具名的例外；从不使用 `to_<x>` |
| 转换 | `as_` 是无开销的视图，`to_` 是新的自有值（有开销），`into_` 消耗原值；`str` 就是字符串类型，所以写 `to_str`、`read_to_str`、`from_str`，从不写 `to_string` |
| 数值转换 | 无损和检查（checked）转换通过 `From` / `TryFrom` / `SaturatingFrom` / `WrappingFrom`；固有的 `to_<num>` 只用于有损舍入（`to_f64`），以及 `Int` / `Float` 上与 `TryFrom` 并存的检查版 `to_<num>() -> T \| none`（[2.1.2](../200-core/201-numbers.md)）；`Ipv4Addr::to_bits` / `from_bits` 与 `From<Ipv4Addr> for u32` 和 `From<u32> for Ipv4Addr` 并存，与 Rust 相同 |
| getter | 不用 `get_` 前缀；`get` 只用于按键或按索引的查找，或包装类型的值（`NonZero::get`）；唯一具名的例外是守卫的 `get_clone()`，它是与有作用域限制的 `get()` 并存的深拷贝（[2.16](../200-core/216-sync.md)） |
| 谓词 | `is_` / `has_`，或 Rust 已确立的词（`success`、`readonly`） |
| setter | `set_<x>(ref self, ..)` 返回 unit；句柄（文件、套接字、游标或读取器）改变的是操作系统状态或共享状态，所以其 setter 接收 `self`（`File::set_len`、`TcpStream::set_nodelay`、`Cursor::set_position`）；构建器使用不带前缀的字段名并返回 `Self` |
| 大小 | `len` = 容器的元素数或字节数；`<x>_len` = 某个长度属性；`count` = 计数得到的数（`count_ones`）；`size` = 内存大小 |
| 添加 / 删除 | 两端用 `push` / `pop`，按键或按索引用 `insert` / `remove`，多个用 `extend`，以及 `clear`、`retain`，`take` 表示移出 |
| 迭代 | `iter()` 返回 `<Type>Iter`（`BinaryHeap::iter` 返回 `HeapIter`）；其他迭代器方法返回以其自身名称的 CamelCase 命名、位于定义模块中的类型（见下文），例外是 `Iterator::inspect`，它返回 `InspectIter`，因为 `Inspect` 是反射 trait（2.12），以及拥有所有权的流适配器（`into_lines`、`into_bytes`、`into_split`、`into_take`、它们的 `_async` 形式、`into_incoming`），它们返回包裹视图类型的 `io::Owned<V, R>`（[2.23.1](../200-core/223-io.md#2231-read)） |
| 反射 | 普通形式是类型给出的答案（`T::type_name()`、`T::size()`），`_of_val` 形式是值在运行时给出的答案（`x.type_name_of_val()`、`x.size_of_val()`、`x.field_names_of_val()`） |
| 异步 | `<name>_async` 是阻塞的 `<name>` 的孪生版本；从不用 `async_` 前缀 |
| 缩写 | 写全称，固定集合除外：`len`、`fd`、`io`、`utf8`、`ns` / `us` / `ms` / `secs` / `mins`、`tls`、`der` / `pem`、`mmap`、`pid`、`alpn`、`ocsp`、`dns`，以及 Rust 已确立的简写：`addr`、`dir`、`temp`、`exe`、`env`、`var` / `vars`、`args`、`arg`、`ty`、`os`、`std`（`Stdio`）、`ptr`、`buf`、`recv`、`ttl`、`subsec`，还有数学名称 `abs`、`rem`、`ln`、`exp`、`cbrt`、`min` / `max` |
| 常量 | `SCREAMING_SNAKE`；常量没有函数孪生版本（`f64::INFINITY`） |
| 模块 | 单数名词或领域（`fs`、`net`、`iter`）；`collections` 和 `ops` 保持复数 |

迭代器类型。`str`：`chars()` -> `Chars`、`bytes()` -> `Bytes`、`char_indices()` -> `CharIndices`、`lines()` -> `Lines`、`split()` -> `Split`、`splitn()` -> `SplitN`、`rsplit()` -> `RSplit`、`rsplitn()` -> `RSplitN`、`split_whitespace()` -> `SplitWhitespace`、`match_indices()` -> `MatchIndices`。`Dict`：`keys()` -> `Keys`、`values()` -> `Values`。`List` / `Slice`：`windows()` -> `slice::Windows`、`chunks()` -> `slice::Chunks`（与适配器 `iter::Windows` / `iter::Chunks` 并存）。`Receiver::iter()` -> `ReceiverIter`。两个模块共用的末段名称连同其模块一起写（`slice::Windows`、`str::Bytes`）。

## 0.5.12. trait 覆盖、`#[non_exhaustive]` 与 `#[sealed]`

标准库的每个公开类型都实现其所属类别的 trait：

| 类型类别 | 实现 |
|---|---|
| 每个公开类型 | `Debug` |
| 值类型（地址、路径、ID、时间、种类、算法、区间、说明符） | `Clone`、`PartialEq`；相等关系为全关系时实现 `Eq`；实现 `Eq` 时实现 `Hash`；值有顺序时实现 `PartialOrd` / `Ord`；体积小时实现 `Copy` |
| 有明显的空值或零值的类型 | `Default` |
| 类文本类型 | `Display` |

按种类划分的 `Debug` 输出：

- 值打印其内容：`0..3`、`0..=3`、`Start(3)`、`Permissions { mode: 0o644, readonly: false }`；地址与 `Display` 一样打印其文本（`127.0.0.1:80`，[3.2.4](../300-std/302-net.md#324-socketaddr--ipaddr)）；`StrBuf` 打印为带引号的文本；
- 句柄打印其类型以及标识它的内容：`File { path: "a.txt" }`、`Child { pid: 42 }`、`Thread { id: ThreadId(3), name: "worker" }`；没有可廉价显示内容的句柄打印其类型名（`TcpStream`、`Mutex`）；
- 迭代器、流或 future 打印其类型名（`Map`、`Chars`、`AsyncFuture`）；
- 持有机密的值从不打印机密：它只以 `Type { field: value, .. }` 这一种形式打印其类型和公开参数：`x25519::PrivateKey { .. }`、`Hmac { .. }`、`DecapsulationKey768 { .. }`、`rsa::PrivateKey { bits: 2048, .. }`、`Secret { len: 32, .. }`。

用作 `Dict` 键和 `Set` 元素、会被比较和排序的库类型的值 trait：

| 类型 | trait |
|---|---|
| `Path` | `Clone`、`Eq`、`Hash`、`PartialOrd`、`Ord`（按文本）、`Default`（空）、`Display`（其文本） |
| `Ipv4Addr`、`Ipv6Addr` | `Clone`、`Copy`、`Eq`、`Hash`、`PartialOrd`、`Ord` |
| `IpAddr` | `Clone`、`Copy`、`PartialEq`、`Eq`、`Hash`、`Debug` |
| `Domain`、`SocketType`、`Protocol`、`Access` | `Clone`、`Copy`、`PartialEq`、`Eq`、`Hash`、`Debug` |
| `SocketAddr`、`SocketAddrV4`、`SocketAddrV6` | `Clone`、`Copy`、`PartialEq`、`Eq`、`Hash`、`Debug` |
| `Ordering` | `Clone`、`Copy`、`Hash`、`PartialOrd`、`Ord`（`Less < Equal < Greater`） |
| `IoErrorKind` | `Clone`、`Copy`、`Eq`、`Hash`、`Display`（`not found`） |
| `TlsErrorKind`、`CipherSuite`、`AeadAlgorithm`、`EcCurve`、`HashAlgorithm` | `Hash` |
| `Signal`、`FileType`、`JoinErrorKind` | `Clone`、`Copy`、`Eq`、`Hash` |
| `Component`、`RawStr`、`RawPath` | `Clone`、`Eq`、`Hash` |
| `SeekFrom`、`Shutdown`、`Alignment`、`Sign`、`LoadOrdering`、`StoreOrdering`、`UpdateOrdering` | `Clone`、`Copy`、`PartialEq`、`Eq`、`Hash`、`Debug` |
| `ThreadId` | `Clone`、`Copy`、`Eq`、`Hash`、`Debug` |
| `Permissions`、`Layout`、`ExitStatus` | `Clone`、`Copy`、`PartialEq`、`Eq`、`Hash` |
| `CStr`、`CString`、`Certificate`（按其 DER 字节）、`FormatSpec` | `Clone`、`PartialEq`、`Eq`、`Hash` |
| `Range` | `Debug`、`Hash` |
| `RangeInclusive`、`RangeFrom`、`RangeTo`、`RangeToInclusive`、`RangeFull` | `Clone`、`PartialEq`、`Eq`、`Hash`、`Debug` |
| `Duration`（`ZERO`）、`StrBuf`（空）、`FormatSpec`（`FormatSpec::new()`） | `Default` |

`#[non_exhaustive]`（[refs-lang 6.2.10](../../refs-lang/006-meta-system/602-built-in-attributes.md)）标记每个可能增加变体的公开枚举，以及每个可能增加字段的选项记录。在其所属包之外对它们做 `match` 需要 `_` 分支，结构体模式需要 `..`：

- 枚举：`IoErrorKind`、`TlsErrorKind`、`CipherSuite`、`AeadAlgorithm`、`EcCurve`、`HashAlgorithm`、`DecodeErrorKind`、`base64::Alphabet`、`nabion::Value`、`Stdio`、`FileType`、`Signal`、`JoinErrorKind`、`fs::Access`、`net::Domain`、`net::SocketType`；
- 记录：`tls::ClientConfig`、`tls::ServerConfig`、`FieldInfo`、`MethodInfo`、`fs::OpenFlags`、`fs::CreateFlags`、`os::unix::fs::OsOpenFlags` / `OsCreateFlags`、`os::windows::fs::OsOpenFlags` / `OsCreateFlags` / `CreateFileParams`。

封闭的枚举保持穷尽：`Ordering`、`Poll`、`Result`、`SeekFrom`、`Shutdown`、`Alignment`、`Sign`、`IpAddr`、`SocketAddr`、`Component`、`LoadOrdering`、`StoreOrdering`、`UpdateOrdering`、`hex::Case`、`base64::Padding`、`VarError`、`FromBytesWithNulError`、`SendError`、`TrySendError`。这两个列表合起来涵盖了每个公开枚举。

`#[sealed]`（[refs-lang 6.2](../../refs-lang/006-meta-system/602-built-in-attributes.md)）标记库的健全性或语义所依赖其实现的 trait；只有定义它们的包能实现它们，约束和调用不受影响：`AtomicPrimitive`（2.16）、`NicheInt`、`NicheFloat`（2.2）、`Pinnable`（1.6）。`Step`（2.7）对用户的区间类型保持开放。`Pod`（1.3）不是密封的（`#[derive(Pod)]` 会检查用户的结构体），`bool` 和 `char` 不是 `Pod`。

## 0.5.13. 异步孪生版本

`x_async` 是阻塞的 `x` 的异步孪生版本：实参相同、错误相同，经 await 调用（`try await s.x_async(..)`），并且只挂起任务。`x` 原地操作其接收者时，孪生版本接收 `ref self`；`x` 消耗接收者时，孪生版本接收 `self`（`close_async`）。

流操作的孪生版本来自 trait，而不是各类型的方法：实现了 `Read`、`BufRead`、`Write`、`Seek` 或 `Close` 的类型，通过实现 `AsyncRead`、`AsyncBufRead`、`AsyncWrite`、`AsyncSeek` 或 `AsyncClose` 获得 `read_async`、`fill_buf_async`、`write_async`、`flush_async`、`seek_async` 或 `close_async`（[2.23.5](../200-core/223-io.md#2235-异步-trait)）。只有不属于流 I/O 的操作才是固有的 `_async` 方法：打开和连接（`open_async`、`connect_async`）、`accept_async`、`shutdown_async`、数据报和 fd 传递调用、TLS 控制以及进程等待（`wait_async`、`output_async`、`status_async`）。

| 类型 | 异步 trait | 固有孪生版本 |
|---|---|---|
| `File` | `AsyncRead`、`AsyncWrite`、`AsyncSeek`、`AsyncClose` | `open_async` 及其他打开函数、`sync_all_async`、`set_len_async`、`metadata_async`、`lock_exclusive_async`、`lock_shared_async`（`unlock` 从不等待） |
| `TcpStream` | `AsyncRead`、`AsyncWrite`、`AsyncClose` | `connect_async`、`shutdown_async` |
| `TcpListener` | `AsyncClose` | `accept_async` |
| `UdpSocket` | `AsyncClose` | `connect_async`、`send_async`、`recv_async`、`send_to_async`、`recv_from_async` |
| `Socket` | `AsyncRead`、`AsyncWrite`、`AsyncClose` | `connect_async`、`accept_async`、`shutdown_async`、`send_to_async`、`recv_from_async` |
| `UnixStream` | `AsyncRead`、`AsyncWrite`、`AsyncClose` | `connect_async`、`shutdown_async`、`send_with_fd_async`、`recv_with_fd_async` |
| `UnixListener` | `AsyncClose` | `accept_async` |
| `UnixDatagram` | `AsyncClose` | `connect_async`、`send_async`、`recv_async`、`send_with_fd_async`、`recv_with_fd_async` |
| `tls::Client`、`tls::Server` | `AsyncRead`、`AsyncWrite`、`AsyncClose` | `new_async`、`with_config_async`、`close_notify_async`、`key_update_async` |
| `net::ReadHalf<T>` | `AsyncRead`、`AsyncClose` | `shutdown_async` |
| `net::WriteHalf<T>` | `AsyncWrite`、`AsyncClose` | `shutdown_async`、`key_update_async`（TLS） |
| `ChildStdin` | `AsyncWrite`、`AsyncClose` | |
| `ChildStdout`、`ChildStderr` | `AsyncRead`、`AsyncClose` | |
| `Child` | | `wait_async` |
| `Command` | | `output_async`、`status_async` |
| `BufReader`、`BufWriter`、`LineWriter`、`Cursor` | 与其内部流相同（[2.23.6](../200-core/223-io.md#2236-缓冲区与-cursor)） | |
| `io::Owned<V, R>`、`io::Chain<A, B>` | 与其读取器（`R`；`A` 和 `B`）相同的 `AsyncClose`（[2.23.5](../200-core/223-io.md#2235-异步-trait)） | |

`Sender` 的 `close` 不是 I/O，没有孪生版本。`File` 或 `Dir` 的孪生版本接收 `ref self`；只有 `close_async` 消耗句柄。

## 0.5.14. 文档与属性

标准库的每个公开项都有 `///` 文档注释：每个函数、类型、trait、trait 方法和固有方法，包括 `const fn`、`async fn` 以及受 `cfg` 控制的项。文档说明该项做什么、返回什么；会抛出错误、返回 `none` 或触发 panic 的函数说明在什么情况下会这样，`unsafe` 函数说明其调用方必须遵守什么。库自身的检查（`nabic check --check-library`）拒绝 `missing_docs`（[refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md)）。

结果有顺序的函数会说明其顺序：

| 项 | 顺序 |
|---|---|
| `Dict`、`Set` 及其迭代器 | 插入顺序 |
| `Set::union`、`Set::symmetric_difference` | 先是来自 `self` 的成员，按其顺序；然后是只在 `other` 中的成员，按其顺序 |
| `Set::intersection`、`Set::difference` | `self` 的顺序 |
| `fs::walk` | 深度优先的前序（目录在其条目之前），每个目录的条目按名称排序 |
| `net::resolve` | 系统解析器的顺序，其首选地址在前 |
| `env::vars`、`env::vars_os` | 操作系统保存环境的顺序；不排序 |
| `task::race` | 最先完成的任务胜出；同一次唤醒时发现已完成的多个任务中，索引最小的胜出；其他任务被取消 |
| `task::select` | 按列表顺序轮询；平局时索引较小者胜出 |

`#[must_use]`（[refs-lang 6.2.11](../../refs-lang/006-meta-system/602-built-in-attributes.md#6211-未使用的结果unused_resultdiscardablemust_use)）标记以下各项，每个类型都带有一条说明如何使用该值的消息：

| 种类 | 项 |
|---|---|
| 迭代器 | `str` 的迭代器（`Chars`、`CharIndices`、`Split`、`Lines`、...）、`List`（`ListIter`）、`Slice`（`SliceIter`、`Windows`、`Chunks`）、`SliceMut`（`SliceMutIter`）、`Dict`（`Keys`、`Values`、`DictIter`）、`Set`（`SetIter`）、`Deque`（`DequeIter`）、`BinaryHeap`（`HeapIter`）和 `Receiver`（`ReceiverIter`）的迭代器；`iter` 适配器和 `TryIterator` 适配器（`TryMap`、`MapErr`、`Results`、...）；`GenIter`、`TryGenIter`；`env::Vars`、`env::VarsOs`；`fs::ReadDir`、`fs::Walk`；`io::Lines`、`io::Split`、`io::Bytes`；`net::Incoming`；`path::Components`、`path::Ancestors` |
| 流 | `AsyncStream`、`TryAsyncStream`；`stream` 适配器和组合子，以及 `TryStream` 适配器；`io::BytesStream`、`io::LinesStream`；`fs::ReadDirStream` |
| future | `async fn` 调用或 `async` 块的值；`LockFuture`、`ReadFuture`、`WriteFuture`、`CondvarWait`、`RecvFuture`、`SendFuture`、`task::Join`、`SelectFuture` |
| 构建器与结果 | `process::Command`、`thread::Builder`、`thread::JoinHandle`、`Result` |
| 函数 | `str` 和 `Path` 的值方法；每个 `Duration` 函数，以及 `Instant`、`SystemTime` 和 `BootTime` 的算术（`duration_since`、`checked_add`、`checked_sub`）；`Iterator`、`DoubleEndedIterator` 和 `TryIterator` 的纯终结方法（`find`、`rfind`、`min`、`max`、`min_by_key`、`max_by_key`、`reduce`、`last`、`position`）；`Set::union`、`intersection`、`difference`、`symmetric_difference`；整数和浮点原始类型、`Int` 和 `Float` 的每个方法 |

`spawn` 和 `spawn_blocking` 返回的 `Task` 不是 `#[must_use]`：无论是否 await，工作都会运行。守卫不需要标记：在语句中丢弃 `MutexGuard` 是错误。`#[discardable]` 函数就是 refs-lang 6.2.11 列出的那些。
