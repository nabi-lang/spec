# Nabi 标准库参考

本文是 Nabi 标准库的正式规范。语言规范定义语法和语义机制，标准库规范则给出这些机制运作所需的类型、trait 和函数的含义。哪个运算符解析到哪个 trait、集合类型有哪些接口、async 和 I/O 依赖哪些保证，都在这里统一定义。

语言规范另行收录在 [`refs-lang/`](../refs-lang/README.md) 中；本文档只涵盖规范本身。库的实际实现不在规范范围之内。

> [!IMPORTANT]
> **语言规范与标准库规范的边界**
>
> [refs-lang](../refs-lang/README.md) 定义**语法和语义机制**（例如 `+` 词法单元对应哪个 trait
> 方法）。[refs-standard-libraries](README.md)（本文档）定义
> **这些 trait 本身的签名和含义**（例如 `Add::add` 应用哪种数值转换）。
>
> 两份规范遵循同一套[记法约定](../refs-lang/000-meta/008-notation.md)。

---

## 三层级模型

标准库不是一个扁平的平面，而是分为三层。几乎所有系统代码都写在安全抽象之上，但实现这些抽象终究需要在某处处理裸内存和 FFI。此外，文件、网络这类只有在宿主环境中才有意义的功能，在嵌入式等没有这种环境的目标上必须完全不存在。为了把这些差异直接体现在依赖图中，标准库采用三层级结构，每个层级是一个路径根：`bare` / `core` / `std`。

```
bare   (base, depends only on the runtime ABI extern "nabi")
  ↓
core   (foundation, safe, all targets; uses bare internally)
  ↓
std   (target-dep; uses core + bare)
```

`bare` 在最底层只暴露原始能力，`core` 在其上提供安全的基础，并且在每个目标上都相同。`std` 在 `core` 之上加入依赖宿主环境的功能，补全普通应用程序所期望的接口面。日常代码只需要 `core`；需要操作系统资源时才用到 `std`；只有必须直接跨越抽象边界时，才显式导入 `bare`。详细的依赖规则见 [0.1 层级模型](000-meta/001-tier-model.md)。

| 层级        | 路径           | 目录                               | 范围                                           |
|-------------|----------------|------------------------------------|------------------------------------------------|
| `bare` | `bare::*` | [`100-bare/`](100-bare/README.md)  | `Ptr`、`ConstPtr`、`reinterpret`、原始分配、pin、FFI |
| `core` | `core::*` | [`200-core/`](200-core/README.md)  | 数值、运算符、iter、集合类型、fmt、sync、task、io、crypto |
| `std` | `std::*` | [`300-std/`](300-std/README.md)    | fs、net、time、thread、env、process、path、nabion、random、os |

---

## 章节

### 0. 总则：[000-meta/](000-meta/README.md)

适用于整个库的策略和表。

- [0.1. 三层级模型](000-meta/001-tier-model.md)
- [0.2. 导入策略（prelude `core::prelude`）](000-meta/002-import.md)
- [0.3. 内建函数机制：编译器 ↔ 标准库的边界](000-meta/003-intrinsics.md)
- [0.4. `#[lang]` 槽映射表](000-meta/004-lang-slot-table.md)
- [0.5. 记法](000-meta/005-notation.md)

### 1. Bare：[100-bare/](100-bare/README.md)

底层、危险的范围。必须显式导入。

- 1.1. [指针家族（`Ptr`/`ConstPtr`/`ManyPtr`/`CPtr`：Zig 式按元数编码 + Rust strict provenance）](100-bare/101-ptr.md)
- 1.2. [转换操作（`ptr_cast`/`align_cast`/`const_cast`/`reinterpret`）](100-bare/102-cast.md)
- 1.3. [`MaybeUninit<T>` + `Pod` + 原始内存操作](100-bare/103-mem.md)
- 1.4. [原始分配](100-bare/104-alloc.md)
- 1.5. [volatile 内存访问](100-bare/105-volatile.md)
- 1.6. [`Pinned<T>`：GC 固定（Go 1.21 Pinner 模型）](100-bare/106-gc-pin.md)
- 1.7. [FFI：`extern "C"` / `CStr` / `CString`](100-bare/107-ffi.md)
- 1.8. [WASM 组件模型（`ffi::wit`）](100-bare/108-wit.md)

### 2. Core：[200-core/](200-core/README.md)

安全范围。适用于所有目标。其 prelude `use core::prelude::*;` 隐式生效。

- 2.1. [数值类型（定宽、`Int`、`Float`）](200-core/201-numbers.md)
- 2.2. [niche 包装类型（`NonZero`、`NonNan`、`Finite`、`Positive`）](200-core/202-non-zero.md)
- 2.3. [算术 / 位运算 trait（`Add`、`Sub`、...、`Shl`、`Shr`）](200-core/203-ops.md)
- 2.4. [复合赋值 trait（`AddAssign`、...）](200-core/204-ops-assign.md)
- 2.5. [比较 trait（`PartialEq`、`Eq`、`Ord`、`PartialOrd`、`Hash`）](200-core/205-cmp.md)
- 2.6. [广播 trait（`DotAdd`、`DotMul`、...）](200-core/206-broadcast.md)
- 2.7. [迭代 trait（`Iterator`、`IntoIterator`）](200-core/207-iter.md)
- 2.8. [`Stream`（异步迭代器）](200-core/208-stream.md)
- 2.9. [集合类型（`List`、`Dict`、`Set`、`Slice`、`Deque`、`BinaryHeap`、`SortedDict` / `SortedSet`、数组、元组）](200-core/209-collections.md)
- 2.10. [fmt（`Display`、`Debug`、`Formatter`）](200-core/210-fmt.md)
- 2.11. [转换（`From`、`Into`、`TryFrom`、`saturating_into`、...）](200-core/211-conversion.md)
- 2.12. [反射（`Reflect`、`Inspect`）](200-core/212-reflect.md)
- 2.13. [标记 trait（`Clone`、`Copy`、`Drop`）](200-core/213-marker.md)
- 2.14. [错误类型（`TypeError`、`ValueError`、...）](200-core/214-error.md)
- 2.15. [字符串（`str`、`StrBuf`、`char`、`c_char`、Unicode 规范化与分段）](200-core/215-string.md)
- 2.16. [同步原语（`Atomic`、`Mutex`、`RwLock`、`Cell`）](200-core/216-sync.md)
- 2.17. [`Channel`](200-core/217-channel.md)
- 2.18. [`task::` API](200-core/218-task.md)
- 2.19. [`par::` API](200-core/219-par.md)
- 2.20. [`Future` / `await` 集成](200-core/220-future.md)
- 2.21. [`crypto::`：哈希、MAC、AEAD、密钥交换、签名](200-core/221-crypto.md)
- 2.22. [`encoding::`：hex 与 base64](200-core/222-encoding.md)
- 2.23. [`io::`：字节流与标准 I/O（`Read`、`BufRead`、`Write`、`Seek`、异步孪生版本）](200-core/223-io.md)
- 2.24. [`simd::`：可移植 SIMD 向量（`Simd`、`Mask`、`f32x4`、...）](200-core/224-simd.md)

> [!NOTE]
> 张量、自动微分、GPU 等科学计算 / AI 领域不属于标准库，而是拆分为外部库。外部库通过后端接口接入编译器的后端，该接口计划在以后的版本中提供。

### 3. Std：[300-std/](300-std/README.md)

依赖目标的范围。需显式导入。

- 3.1. [`fs::`：文件系统（原子操作、Dir 能力、reflink）](300-std/301-fs.md)
- 3.2. [`net::`：TCP/UDP/Unix + TLS 1.3（HTTP 在外部）](300-std/302-net.md)
- 3.3. [`time::`：时钟、日历日期与时区（配合 `core::time` 的 `Instant` / `Duration`）](300-std/303-time.md)
- 3.4. [`thread::`：操作系统线程（1:1，与任务分开）](300-std/304-thread.md)
- 3.5. [`env::`：环境变量 / 参数 / 目录](300-std/305-env.md)
- 3.6. [`process::`：`Command` / pidfd / kill_on_drop](300-std/306-process.md)
- 3.7. [`path::`：UTF-8 `Path` + `/` 运算符（Python pathlib 风格）](300-std/307-path.md)
- 3.8. [`nabion::`：NABION 数据格式（`.nabion`）](300-std/308-nabion.md)
- 3.9. [`random::`：随机字节、生成器与分布](300-std/309-random.md)
- 3.10. [`os::`：只有部分操作系统才有的功能（`unix` / `linux` / `windows`）](300-std/310-os.md)
- 3.12. [`panic::`：panic 钩子（`set_hook` / `take_hook`）](300-std/312-panic.md)
- `wit::`：生成的 WIT 绑定所调用的组件模型异步 ABI（[1.8](100-bare/108-wit.md)）；属于绑定内部，此处不作规定。

---

## 阅读方式

### 初次阅读

初次接触时，建议先按 [`refs-lang`](../refs-lang/README.md) 的第 0–6 章理解语言的机制，再在此基础上阅读本文档。本文档为 refs-lang 定义的机制赋予含义；如果对这些机制没有感觉，trait 和签名读起来容易显得悬在半空。两份文档对照着来回阅读，是最高效的方式。

### 作为参考

每一项都由稳定的节号（`<chapter>.<section>`）标识。内部链接和外部引用都使用同一个编号，所以即使文件移动，它也能准确指向原处。

```
refs-standard-libraries 2.3        →  core ops (Add/Sub/...)
refs-standard-libraries 1.1        →  bare Ptr<T>
refs-standard-libraries 3.1.2      →  std fs (specific subsection)
```

### 搜索

要快速找到某个 trait 或函数，最轻便的方法是用 `ripgrep` 缩小目录范围后搜索。

```bash
rg "trait Add" spec/en/refs-standard-libraries/200-core/
rg "fn into" spec/en/refs-standard-libraries/
```

---

## 目录 / 文件命名

本文档采用与 `refs-lang` 不同的目录约定，因为两份规范的形态不同：refs-lang 是扁平结构，第 0 章到第 9 章的权威性全部相同；refs-standard-libraries 则是层次结构，三个层级在交叉引用中是关键信息。这种结构差异直接反映在目录名的位数上。

| 规范                    | 前缀                   | 原因                                                            |
|-------------------------|------------------------|-----------------------------------------------------------------|
| refs-lang               | 一位数字（`002-`）     | 10 章（0–9），权威性全部相同；扁平形态                          |
| refs-standard-libraries | 三位数字（`200-`）     | 3 个层级 × N 个模块；层级是交叉引用的关键信息                   |

具体的文件命名规则如下。层级目录的前缀以 100 为单位，其中的模块文件以三位前缀开头，由一位层级数字和两位模块数字组成。

| 位置     | 形式                                   | 示例                              |
|----------|----------------------------------------|-----------------------------------|
| 层级     | `<TNN>-<descriptive-name>/`            | `100-bare/`、`200-core/`          |
| 模块     | `<tier><module>-<name>.md`             | `203-ops.md`、`211-conversion.md` |
|          | （一位层级数字 + 两位模块数字）        |                                   |
| README   | 每个层级 / 章中的 `README.md`          | 目录 + 概述                       |

虽然层级信息已编码在文件路径本身中，但在外部引用本文档时，稳定的节号（`2.3.1` 等）才是正式的记法。文件路径只是为方便在 IDE 和 GitHub 中浏览而采用的辅助记法。
