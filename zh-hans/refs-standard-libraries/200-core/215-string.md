# 2.15. 字符串：`str` / `char` / `c_char`

`core::str` 模块给出字符串和字符类型的权威定义。Nabi 遵循 **UTF-8 优先**的策略：进程内的每个字符串都按 UTF-8 处理，`c_char` 只在 FFI 边界使用。

`str`、`char` 和 `c_char` 是编译器内置的原始类型，不是结构体。编译器不为它们的 lang 槽（`str_type`、`char_type`、`cchar_type`）挂接任何项（[0.4](../000-meta/004-lang-slot-table.md)）；本模块定义它们的方法和 trait 实现。

## 2.15.1. 字符串模型：不可变的 `str` 与可变的 `StrBuf`

`str` 是不可变的 UTF-8 字符串值；用于构建字符串的可变缓冲区是单独的类型 `StrBuf`。

| 类型     | 含义                                               | 内存表示                     | 可变性              |
|----------|----------------------------------------------------|------------------------------|---------------------|
| `str`    | UTF-8 字符串值                                     | `(ptr, len)` 值；字节是永生常量或 GC 对象 | **不可变** |
| `StrBuf` | 可增长的 UTF-8 缓冲区                              | GC 堆对象                    | 可变                |
| `char`   | Unicode 标量值（语言内置）                         | 4B (u32)                     | （值，不能改变）    |
| `c_char` | C ABI 1 字节字符（FFI 原始类型）                   | 1B（i8 或 u8，依赖目标）     | （值）              |

### Rust / Python / Nabi 模型比较

三种语言处理字符串的方式：

| 方面           | Rust             | Python             | Nabi                       |
|----------------|------------------|--------------------|----------------------------|
| 值             | `&str`           | `str`              | `str`（不可变的 `(ptr, len)`） |
| 可变缓冲区     | `String`         | `io.StringIO` 等   | `StrBuf`                   |
| 内存           | 借用检查器       | 引用计数           | GC                         |

Nabi 的 `str` 不需要追踪生命周期，这一点接近 Python 的不可变 `str`；它像 Rust 的 `&str` 一样表示为 `(ptr, len)` 对。运行时创建的字符串的字节位于一个 GC 对象中（type id `1`，[refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-runtime-reserved-type-ids)）。

## 2.15.2. `str`：不可变的 UTF-8 字符串

`str` 是编译器内置的原始类型，所以不需要声明、结构体或 `#[keyword]`。它的值是一个 `(ptr, len)` 对：字节的地址和以字节计的长度（不是字符数）。

`str` 是按值传递的 `(ptr, len)` 对，它指向的字节永远不会改变。每个修改字符串的操作都返回新的 `str`。

### 核心方法

```nabi
impl str {
    fn new() -> str;                        // ""
    fn from_utf8<B: AsBytes>(bytes: B) throws (Utf8Error) -> str;
    fn from_utf8_lossy<B: AsBytes>(bytes: B) -> str;   // each ill-formed sequence → U+FFFD
    fn from_utf16<U: IntoIterator<Item = u16>>(units: U) throws (Utf16Error) -> str;
    fn from_utf16_lossy<U: IntoIterator<Item = u16>>(units: U) -> str;   // each unpaired surrogate → U+FFFD

    fn len(self) -> usize;                  // byte length
    fn char_count(self) -> usize;           // number of chars, same as chars().count()
    fn is_empty(self) -> bool;
    fn as_bytes(self) -> Slice<u8>;         // read-only view without copying
    fn byte(self, i: usize) -> u8;          // O(1), panics when out of range
    fn as_ptr(self) -> ConstPtr<u8>;        // FFI
    fn encode_utf16(self) -> List<u16>;     // UTF-16 code units, no NUL (2.15.6)

    // reading methods
    fn chars(self) -> Chars;            // Unicode scalar iterator
    fn bytes(self) -> Bytes;            // u8 iterator
    fn char_indices(self) -> CharIndices;
    fn lines(self) -> Lines;            // strips a trailing \r, does not yield a final empty line
    fn split(self, sep: str) -> Split;  // keeps empty pieces ("a,,b" → "a", "", "b")
    fn split_whitespace(self) -> SplitWhitespace;   // splits on Unicode White_Space (same set as trim)
    fn splitn(self, n: usize, sep: str) -> SplitN;    // at most n pieces; the last is the unsplit rest
    fn rsplit(self, sep: str) -> RSplit;              // split's pieces, last first
    fn rsplitn(self, n: usize, sep: str) -> RSplitN;  // at most n pieces from the end, last first
    fn match_indices(self, pat: str) -> MatchIndices; // (byte offset, match), non-overlapping

    fn starts_with(self, pat: str) -> bool;
    fn ends_with(self, pat: str) -> bool;
    fn contains(self, pat: str) -> bool;
    fn find(self, pat: str) -> usize | none;
    fn rfind(self, pat: str) -> usize | none;

    fn strip_prefix(self, prefix: str) -> str | none;   // the rest after removing the prefix, if present
    fn strip_suffix(self, suffix: str) -> str | none;
    fn split_once(self, sep: str) -> (str, str) | none; // before / after the first sep
    fn repeat(self, n: usize) -> str;

    fn trim(self) -> str;                   // strips Unicode White_Space
    fn trim_start(self) -> str;
    fn trim_end(self) -> str;
    fn trim_matches(self, pat: str) -> str;           // strips every leading and trailing repeat of pat
    fn trim_start_matches(self, pat: str) -> str;
    fn trim_end_matches(self, pat: str) -> str;

    fn to_lowercase(self) -> str;           // new str (full Unicode mapping: 1:N, including context rules)
    fn to_uppercase(self) -> str;
    fn replace(self, from: str, to: str) -> str;   // empty from matches at every char boundary: "ab".replace("", "-") == "-a-b-"

    fn parse<T: FromStr>(self) throws (T::Err) -> T;
}

// UTF-8-safe slicing: s[a:b] dispatches here and validates byte boundaries
impl Sliceable<Range<usize>> for str { type Output = str; }   // shares bytes
```

`splitn` 和 `rsplitn` 最多产生 `n` 段；最后一段是字符串未分割的剩余部分，`n == 0` 时不产生任何内容。`rsplitn` 从末尾开始搜索，所以 `"a.b.c".rsplitn(2, ".")` 先产生 `"c"`，再产生 `"a.b"`。与 `split` 一样，空的 `sep` 把整个字符串作为一段产生。`match_indices` 从左到右为每个匹配产生一个 `(usize, str)` 对；匹配之后从其后继续搜索，所以 `"aaaaa".match_indices("aa")` 产生偏移 0 和 2。空的 `pat` 不匹配任何内容。`trim_matches` 从两端去掉 `pat` 的完整重复（`"--x--".trim_matches("-") == "x"`，`"aaa".trim_matches("aa") == "a"`）；空的 `pat` 保持字符串不变。

```nabi
"key=a=b".splitn(2, "=");               //→ "key", "a=b"
"x.tar.gz".rsplitn(2, ".");             //→ "gz", "x.tar"
"ab-ab".match_indices("ab");            //→ (0, "ab"), (3, "ab")
"--x--".trim_start_matches("-");        //→ "x--"
```

`as_ptr` 返回指向字符串原位字节的指针。在可能出现指针值的函数中（变量或表达式的类型写出了 `bare::ptr` 类型，包括作为类型实参（`List<ConstPtr<u8>>`）或出现在函数类型中），只要持有该 `str` 的变量还在作用域内，GC 就不会移动该字符串，指针也保持有效。在生成器或 `async` 主体中，这一点跨 `yield` / `await` 成立。在其他地方，变量只持有字符串到它最后一次使用为止，并且只能通过另一个对象到达的字符串可能被 GC 移动：它没有被固定（[stdlib 1.6](../100-bare/106-gc-pin.md)）。要让字节在原位保持更久，请固定该字符串（[1.6.6](../100-bare/106-gc-pin.md#166-提取方法)）。

### 按 Python 命名补齐的方法

```nabi
fn reversed(self) -> str;                       // characters in reverse order; Python s[::-1]
fn count(self, pat: str) -> usize;              // non-overlapping occurrences; "" counts len+1
fn replacen(self, from: str, to: str, n: usize) -> str;   // the first n occurrences
fn to_titlecase(self) -> str;                   // each word's first letter titlecased, the rest lowercased
```

`reversed` 按码位反转。`count("")` 与 Python 一样，是字符数加一。在 `to_titlecase` 中，一个单词从不跟在字母、数字或撇号之后的字母开始，所以 `"don't"` 仍是一个单词（与 Python 的 `title()` 不同）；映射使用完整的 Unicode 映射（2.15.4）。

### `StrBuf`：可变缓冲区

逐段构建字符串时使用 `StrBuf`。

```nabi
struct StrBuf { /* GC heap handle: len / capacity / bytes */ }

impl StrBuf {
    fn new() -> StrBuf;                     // capacity 16
    fn with_capacity(cap: usize) -> StrBuf;
    fn len(self) -> usize;                  // byte length
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;             // in bytes
    fn clear(self);                         // length 0, capacity kept
    fn reserve(self, additional: usize);    // room for `additional` more bytes (grows by doubling)
    fn push(self, c: char);                 // appends one char (UTF-8 encoded)
    unsafe fn push_byte(self, b: u8);       // caller keeps the contents UTF-8; the safe byte path is str::from_utf8
    fn push_str(self, s: str);
    fn to_str(self) -> str;                 // immutable copy of the current contents
}

impl !Sync for StrBuf {}
```

与集合类型一样，`StrBuf` 是引用句柄，所以它的方法接受 `self`（[2.9](209-collections.md)）；句柄的各个副本共享同一个缓冲区，所以它不是 `Sync`（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）。它显示为其内容。

```nabi
let buf = StrBuf::new();
for x in xs {
    buf.push_str(f"{x},");
}
let s: str = buf.to_str();
```

### 字符串字面量：`"hello"`

```nabi
let s: str = "hello";        // a str pointing to immortal constant bytes
```

`"hello"` 字面量降级为一个 `str` 值，指向永生区域中的常量字节。由于 `str` 不可变，不存在复制或写时复制。

### trait 实现

| trait                       | impl                           | 行为                                       |
|-----------------------------|--------------------------------|--------------------------------------------|
| `Display`                   | ✓                              | 原样输出自身                               |
| `Debug`                     | ✓                              | 带转义的 `"..."`                           |
| `Clone`                     | ✓                              | 相同的值（不可变，所以共享字节）           |
| `Copy`                      | ✓                              | 复制 `(ptr, len)` 值                       |
| `PartialEq`                 | ✓                              | 比较内容（逐字节）                         |
| `Eq`                        | ✓                              |                                            |
| `PartialOrd`                | ✓                              | 按字节的字典序                             |
| `Ord`                       | ✓                              |                                            |
| `Hash`                      | ✓                              |                                            |
| `IntoIterator`              | **无**                         | 显式写 `.chars()`；避免字节与字符的歧义    |
| `Add<str>` (`s + t`)        | ✓ → str（拼接，新 str）        |                                            |
| `Add<char>` (`s + c`)       | ✓ → str（追加 `c`，新 str）    |                                            |
| `AddAssign<str>`, `AddAssign<char>` (`s += t`, `s += c`) | ✓ | `s = s + t`：新 str 替换 `s`            |
| `FromIterator<char>`        | ✓                              | `(chars).collect::<str>()`                 |
| `FromIterator<str>`         | ✓                              | `(strs).collect::<str>()`                  |

> [!NOTE]
> `str` 是不可变值，所以 `let t = s;` 只复制 `(ptr, len)`，两边都不能改变内容。

## 2.15.3. `s[a:b]` 切片：UTF-8 安全

```nabi
let s: str = "hello, 세계!";
s.len();                       //→ 14  (byte length: "hello, " 7 + Korean 3 bytes × 2 + "!" 1)
s.chars().count();             //→ 10  (char count)
s.char_count();                //→ 10  (same, without an iterator)

let s2 = s[7:13];              //→ "세계"  (byte boundary OK)
let s3 = s[8:9];               // panic: byte in the middle of a UTF-8 sequence
let s4 = s[7:100];             //→ "세계!" (end clamped to the length)
let s5 = s[-1:];               //→ "!"
```

`str` / `StrBuf` 的边界是字节偏移。边界先像 Python 那样截断到有效范围（负数从末尾计数，[refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-slicing-arrstartend)）；如果得到的位置在 UTF-8 序列中间，则触发 panic。结果是与原字符串共享字节的 `str`（见下文）。`str` 切片没有步长；`s[::2]` 是类型错误。

### 字符边界与不会 panic 的切片

```nabi
impl str {
    fn is_char_boundary(self, i: usize) -> bool;       // i starts a char, or i == len()
    fn floor_char_boundary(self, i: usize) -> usize;   // the boundary at or before i
    fn ceil_char_boundary(self, i: usize) -> usize;    // the boundary at or after i
    fn get(self, r: Range<usize>) -> str | none;       // none instead of a panic or a clamp
    fn char_slice(self, a: usize, b: usize) -> str;    // chars a..b, counted like chars()
}
```

```nabi
let s = "hello, 세계!";
s.is_char_boundary(8);         //→ false (inside "세")
s.floor_char_boundary(8);      //→ 7
s.ceil_char_boundary(8);       //→ 10
s.get(7..10);                  //→ "세"
s.get(8..10);                  //→ none (off a boundary)
s.get(7..100);                 //→ none (past the end: no clamping)
s.char_slice(7, 9);            //→ "세계"
s.char_slice(0, 5);            //→ "hello" (the first 5 chars)
```

当 `r.start > r.end`、任一边界超过 `len()` 或任一边界不是字符边界时，`get` 返回 `none`；否则结果为 `s[r.start:r.end]`。对任何 `i >= len()`，`floor_char_boundary` 和 `ceil_char_boundary` 返回 `len()`。`char_slice` 按 `chars()` 产生的顺序从 0 开始计数字符；两个边界都截断到 `char_count()`，`a >= b` 时得到 `""`。它需要遍历字符串，复杂度为 O(b)。要按用户感知的字符切片，使用 `grapheme_slice`（[2.15.10](#21510-unicode-文本规范化大小写折叠分段)）。

不在字符边界上的切片会触发 panic，消息会指出以下内容：

```text
string slice is not on a char boundary: byte 8 is inside `세` (bytes 7..10)
string offsets count bytes; to work by character, use `chars()`, `char_indices()` or `char_slice(a, b)`
to cut at a boundary, use `floor_char_boundary(i)`; `get(a..b)` gives `none` instead of a panic
```

### 不允许直接索引

```nabi
let s: str = "hello";
let c = s[0];                  // ERROR: `Index<usize>` is not defined
                               //         reason: byte vs char ambiguity + UTF-8 safety
```

必须像 `s.chars().next()` 或 `s.byte(0)` 那样明确写出意图。

### 子串不会被复制

对于 `str` 类型的 `s`，`s[a:b]`、`lines`、`split*`、`rsplit*`、`split_whitespace`、`split_once`、`match_indices`、`trim*` 和 `strip_*` 的结果都是与原字符串共享字节的 `str` 值，复杂度为 O(1)（[refs-lang 8.8.1](../../refs-lang/008-abi/808-lowering-types.md#881-primitive-integers--floating-point)）。因此一个小片段会让很大的原字符串保持存活。没有用于分离它的 API。`as_bytes()` 同样是不复制的只读 `Slice<u8>` 视图。`StrBuf` 的切片是持有该范围副本的新 `str`，复杂度为 O(b − a)：`str` 不可变，所以不能共享一个仍可能改变的缓冲区。

## 2.15.4. `char`：Unicode 标量值

`char` 是语言内置类型（[2.1](201-numbers.md)）：一个 4 字节的 Unicode 标量值，不是结构体。本模块定义它的方法。

Unicode 标量值的范围是 0..=0xD7FF 与 0xE000..=0x10FFFF 的并集。代理项范围 0xD800..=0xDFFF 用作**自动 niche**，所以 `char | none` 可以用 4B 编码。

### 字面量与方法

```nabi
let c: char = 'A';                   //→ U+0041
let k: char = '한';                  //→ U+D55C
let e: char = '😀';                  //→ U+1F600

c.to_lowercase();                    //→ "a"
c.is_alphabetic();                   //→ true
c.is_ascii();                        //→ true
'1'.to_digit(10);                    //→ 1 (u32 | none; none if not a digit)
c.len_utf8();                        //→ 1 (number of bytes when UTF-8 encoded)
e.len_utf8();                        //→ 4
```

```nabi
impl char {
    fn from_u32(n: u32) -> char | none;                  // none for a surrogate / out of range
    unsafe fn from_u32_unchecked(n: u32) -> char;        // no check; the caller guarantees a scalar value
    fn from_digit(d: u32, radix: u32) -> char | none;    // none if d >= radix; panics if radix is outside 2..=36
    fn to_digit(self, radix: u32) -> u32 | none;         // none if not a digit; panics if radix is outside 2..=36
    fn is_digit(self, radix: u32) -> bool;               // panics if radix is outside 2..=36
    fn len_utf8(self) -> usize;
    fn len_utf16(self) -> usize;

    // Unicode properties
    fn is_alphabetic(self) -> bool;
    fn is_lowercase(self) -> bool;
    fn is_uppercase(self) -> bool;
    fn is_whitespace(self) -> bool;
    fn is_numeric(self) -> bool;
    fn is_alphanumeric(self) -> bool;
    fn is_control(self) -> bool;
    fn to_lowercase(self) -> str;           // the full mapping, 1:N included
    fn to_uppercase(self) -> str;
    fn to_titlecase(self) -> str;
    fn to_simple_lowercase(self) -> char;   // UCD simple mappings (slc / suc / stc), 1:1
    fn to_simple_uppercase(self) -> char;
    fn to_simple_titlecase(self) -> char;

    // ASCII
    fn is_ascii(self) -> bool;
    fn is_ascii_uppercase(self) -> bool;     fn is_ascii_lowercase(self) -> bool;
    fn is_ascii_alphabetic(self) -> bool;    fn is_ascii_digit(self) -> bool;
    fn is_ascii_alphanumeric(self) -> bool;  fn is_ascii_hexdigit(self) -> bool;
    fn is_ascii_whitespace(self) -> bool;    fn is_ascii_control(self) -> bool;
    fn is_ascii_graphic(self) -> bool;       fn is_ascii_punctuation(self) -> bool;
    fn to_ascii_uppercase(self) -> char;     fn to_ascii_lowercase(self) -> char;
    fn eq_ignore_ascii_case(self, other: char) -> bool;
}
```

`char` 的 `to_lowercase` / `to_uppercase` / `to_titlecase` 返回使用完整 Unicode 映射的 `str`，结果可能是多个字符（`'ß'.to_uppercase() == "SS"`，`'ß'.to_titlecase() == "Ss"`，`'ǆ'.to_titlecase() == "ǅ"`，`'ﬁ'.to_titlecase() == "Fi"`，`'\u{130}'.to_lowercase() == "i\u{307}"`）。`to_simple_*` 形式返回使用 UCD 简单映射的一个 `char`，没有映射时返回字符本身（`'ß'.to_simple_uppercase() == 'ß'`，`'ǆ'.to_simple_titlecase() == 'ǅ'`）；它们从不分配。`str` 的 `to_lowercase` / `to_uppercase` 应用完整映射，包括 1:N 映射和上下文规则（希腊语词尾 sigma `Σ` → `ς`）（`"ß".to_uppercase() == "SS"`）。大小写映射与区域设置无关。要做不区分大小写的比较，使用完整大小写折叠（`casefold`，[2.15.10](#21510-unicode-文本规范化大小写折叠分段)），而不是 `to_lowercase`。

Unicode 属性表和大小写表采用 UCD 18.0.0。

### 转换

```nabi
char::from_u32(0x41);                //→ 'A' (char | none; none for a surrogate)
char::from_digit(5, 10);             //→ '5' (char | none; none if 5 is not a digit of the radix)

let n: u32 = c as u32;               //→ 65 (lossless)
let c2: char = n as char;            //→ ERROR: only u8 can `as char`
let b: u8 = 0x41;
let c4: char = b as char;            //→ 'A' (a u8 is always a scalar value)
let c3 = try char::try_from(n);      //→ char (TryFrom<u32>: via from_u32, OverflowError if none)
```

### trait 实现

`char` 实现所有 cmp / fmt trait，以及 `Copy`、`Clone` 和 `Hash`。niche 自动生效，所以 `char | none` 占 4B。

## 2.15.5. `c_char`：C ABI 1 字节（语言内置）

`c_char` 是编译器内置的 FFI 原始类型，不是结构体。它占一个字节；它是有符号（`i8`）还是无符号（`u8`）取决于目标（OS / 架构）。本模块按其字节（`c as u8`）对它进行比较、排序和哈希。它实现 `Clone`、`Copy`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash` 和 `Debug`，`Debug` 把字节打印为数字（`65`）；它没有 `Display`。

它与 C 的 `char` ABI 兼容；它是原始字节，不是 UTF-8 编码单元。它只用于 FFI：

```nabi
extern "C" {
    fn strlen(s: ConstPtr<c_char>) -> usize;
    fn strcmp(a: ConstPtr<c_char>, b: ConstPtr<c_char>) -> i32;
}
```

### 字面量

```nabi
let c: c_char = c'A';        //→ 65 (i8/u8, target-dependent)
let n: c_char = c'\n';       //→ 10
```

带前缀的形式 `c'...'` 把它与 `char` 区分开。

### char 与 c_char

比较这两种类型，区别一目了然。

| 方面         | `char`                              | `c_char`                      |
|--------------|-------------------------------------|-------------------------------|
| 大小         | 4 字节                              | 1 字节                        |
| 含义         | Unicode 标量（码位）                | C ABI 字节                    |
| 范围         | 0..=0x10FFFF（不含代理项）          | 0..=255 或 -128..=127         |
| 多字节       | 单个 char 覆盖 BMP / SMP            | 一个多字节字符 = 字节序列     |
| 字面量       | `'A'`                               | `c'A'`                        |
| 注册方式     | 语言内置                            | 语言内置                      |
| 用途         | 一般字符串处理                      | FFI / 字节级处理              |

因此标准库函数原则上**优先使用 `char`**，只在明确的 FFI 边界使用 `c_char`。

## 2.15.6. UTF-8 不变式：进程内的每个字符串

`str` 始终保证是有效的 UTF-8，原因如下。构造函数会进行验证：`str::from_utf8(bytes)` 遇到无效字节序列时抛出 `Utf8Error`（`throws (Utf8Error)`）。切片会验证字节边界，拒绝多字节序列中间的位置。`StrBuf::push` 或 `push_str` 等原地修改也会保持 UTF-8 完整性。只有 `unsafe` 代码能绕过验证：`StrBuf::push_byte`（不经检查追加一个字节的 `unsafe fn`）和 `bare::mem::transmute_unchecked`。

### 为什么限制绕过验证

考虑 UTF-8 不变式**被破坏**时会出什么问题：`chars()` 可能 panic 或陷入无限循环，被截断的多字节序列可能显示错误，还可能出现超长编码之类的安全问题。

由于标准库在构造时保证验证，用户代码中的 `str.chars()` 等总是安全的。不进入 unsafe 代码就无法破坏这个不变式。

### 字节 ↔ str 转换

```nabi
let bytes: List<u8> = [0x68, 0x65, 0x6C, 0x6C, 0x6F];
let s: str = try str::from_utf8(bytes);
                                          //→ "hello"

let invalid: List<u8> = [0xFF, 0xFE];
let t: str = try str::from_utf8(invalid);
                                          //→ throws a Utf8Error, valid_up_to() == 0
```

```nabi
struct Utf8Error { .. }       // private fields (0.5.10)
impl Utf8Error {
    fn valid_up_to(self) -> usize;   // length of the valid UTF-8 prefix, in bytes
    fn message(self) -> str;         // "invalid UTF-8 at byte {valid_up_to}"
}
```

`Utf8Error` 实现 `Display`（即其 `message()`）、`Debug`、`Error`、`PartialEq`、`Eq` 和 `Clone`。它是唯一的 UTF-8 错误：标准库中每个失败的 UTF-8 检查都抛出它（`str::from_utf8`、`CStr::to_str`、`RawStr::to_str`、`RawPath::to_path`）。

`str::from_utf8_lossy(bytes)` 从不抛出错误：非法序列的每个最大子部分（Unicode §3.9，U+FFFD Substitution of Maximal Subparts；即 WHATWG UTF-8 解码器的行为）变成一个 U+FFFD。

### UTF-16

```nabi
let u = "h\u{E9}\u{1F600}".encode_utf16();    //→ [104, 233, 55357, 56832]
let s = try str::from_utf16(u);               //→ "hé😀"
let t = try str::from_utf16([0x61u16, 0x62u16, 0xDC00u16]);
                                              //→ throws a Utf16Error, valid_up_to() == 2
str::from_utf16_lossy([0x61u16, 0xD800u16]);  //→ "a\u{FFFD}"
```

`encode_utf16` 给出字符串的 UTF-16 码元，U+FFFF 以上的字符表示为代理对，末尾不带 NUL（要得到传给 C 的以 NUL 结尾的字符串，使用 `bare::ffi::WideCString`，[stdlib 1.7.12](../100-bare/107-ffi.md#1712-widecstr--widecstringwchar_t-字符串)）。`str::from_utf16` 从任意 `IntoIterator<Item = u16>` 取回码元：`List<u16>`、`Slice<u16>` 或 `[u16; N]`。它在第一个未配对的代理项处抛出 `str::Utf16Error`：后面没有低代理项的高代理项、单独的低代理项，或位于末尾的高代理项。`str::from_utf16_lossy` 用一个 U+FFFD 替换每个未配对的代理项，从不抛出错误。

```nabi
struct Utf16Error { .. }      // str::Utf16Error; private fields (0.5.10)
impl Utf16Error {
    fn valid_up_to(self) -> usize;   // length of the valid prefix, in units
    fn message(self) -> str;         // "unpaired surrogate at unit {valid_up_to}"
}
```

`Utf16Error` 实现 `Display`（即其 `message()`）、`Debug`、`Error`、`PartialEq`、`Eq` 和 `Clone`。`bare::ffi::WideCStr::to_str` 也抛出它。

## 2.15.7. `FromStr` trait：解析

```nabi
trait FromStr {
    type Err;
    fn from_str(s: str) throws (Err) -> Self;
}
```

`str.parse::<T>()` 是 `throws (T::Err)`，通过这个 trait 分派。对数值类型、`bool`、`char` 和 `str`，`Err` 是 core 的 `ParseError`（`input()`、`position()`、`message()`）（[2.14.4](214-error.md#2144-内置错误类型)）。`char` 解析恰好包含一个字符的字符串，失败时给出 `empty string` 或 `too many characters`；`str` 返回其输入，从不抛出错误。数值解析建立在运行时辅助函数 `nabi_str_parse_int` / `nabi_str_parse_float` 之上（[refs-lang 8.5](../../refs-lang/008-abi/805-extern-symbols.md#string-helpers)）。浮点数的 `FromStr` 在有限数字超出类型范围时以 `number out of range` 失败；`inf` / `NaN` 文本可以解析。

```nabi
let n: i32 = try "42".parse();         // FromStr for i32
let f: f64 = try "3.14".parse();       // FromStr for f64
let b: bool = try "true".parse();      // FromStr for bool
let c: char = try "x".parse();         // FromStr for char

// user-defined
impl FromStr for Color {
    type Err = ParseColorError;
    fn from_str(s: str) throws (ParseColorError) -> Color { ... }
}
let c: Color = try "red".parse();
```

`FromStr` 与 [2.11 conversion](211-conversion.md) 中的 `TryFrom<str>` **相互独立**。它只处理 `str → T` 转换，并明确表示可能失败。

## 2.15.8. 模块映射

各类型的注册方式与实现位置：

| 类型     | 注册方式          | 实现                                 |
|----------|-------------------|--------------------------------------|
| `str`    | 语言内置          | 编译器（方法：标准库）               |
| `StrBuf` | 普通结构体        | 标准库                               |
| `c_char` | 语言内置          | 编译器（方法：标准库）               |
| `char`   | 语言内置          | 编译器（方法：标准库）               |

`str` 和 `char` 的方法体调用在 `core::rt` 中声明为 `extern "nabi"` 函数的运行时辅助函数：`nabi_str_*`（长度、比较、搜索、切片、大小写映射与折叠、修剪、解析、规范化、分段）和 `nabi_char_*`（Unicode 属性与大小写）（[refs-lang 8.5](../../refs-lang/008-abi/805-extern-symbols.md#string-helpers)）。它们不带 `#[intrinsic]` 属性。

## 2.15.9. 与其他语言的比较

| 方面            | Rust              | Python 3                | Java                    | Nabi                               |
|-----------------|-------------------|-------------------------|-------------------------|------------------------------------|
| 类型划分        | `&str` + `String` | `str`（单一）           | `String`                | `str` + `StrBuf`                   |
| 可变性          | 只有 `String` 可变 | **不可变**             | **不可变**              | `str` 不可变，`StrBuf` 可变        |
| 编码            | UTF-8             | 未规定（依赖实现）      | UTF-16                  | UTF-8                              |
| char            | `char` (4B)       | `str`（单字符）         | `char`（2B = 码元）     | `char`（4B Unicode 标量）          |
| FFI 字节        | `c_char` / `u8`   | `bytes`                 | `byte`                  | `c_char`（1B，依赖目标）           |
| len 的含义      | 字节长度          | 字符数                  | UTF-16 码元             | 字节长度（显式 `.char_count()`）   |
| 索引 `s[i]`     | **无**            | 字符（Python str）      | UTF-16 码元             | **无**（显式 chars/bytes）         |

`str` 是不可变的 `(ptr, len)` 值，可变缓冲区单独分出为 `StrBuf`；有 GC，所以不像 Rust 那样需要区分生命周期。`len()` 表示**字节长度**，与 Rust 一致；字符数用 `char_count()`（或 `chars().count()`）单独获取。Nabi **不提供** `s[i]`，与 Rust 一致，避免了字节与字符的歧义。把 `char` 定义为 4B 的 Unicode 标量，避免了 Java 2B UTF-16 码元带来的代理项陷阱。最后，`c_char` 是单独的类型，与 `char` 在 ABI 上界限分明。

## 2.15.10. Unicode 文本：规范化、大小写折叠、分段

模块 `core::str::unicode` 提供文本在码位之外所需的功能：规范化形式（UAX #15）、完整大小写折叠，以及按字素簇和单词分段（UAX #29）。它的方法在 `str` 上，所以不需要导入。这些表是运行时中的 UCD 18.0.0，与 `char` 属性的版本相同（2.15.4）。

```nabi
enum NormalForm { Nfc, Nfd, Nfkc, Nfkd }

impl str {
    fn normalize(self, form: NormalForm) -> str;        // self itself when already in form
    fn is_normalized(self, form: NormalForm) -> bool;
    fn casefold(self) -> str;                            // full case folding
    fn eq_ignore_case(self, other: str) -> bool;         // canonical caseless match

    fn graphemes(self) -> Graphemes;                     // extended grapheme clusters
    fn grapheme_count(self) -> usize;
    fn grapheme_slice(self, a: usize, b: usize) -> str;  // clusters a..b
    fn split_word_bounds(self) -> WordBounds;            // every piece between word boundaries
    fn words(self) -> Words;                             // the pieces holding a letter or a digit
}
```

`Graphemes`、`WordBounds` 和 `Words` 是产生与原字符串共享字节的 `str` 切片的迭代器。`Graphemes` 也是 `DoubleEndedIterator`：`next_back` 和 `rev()` 从末尾给出同样的字素簇，两端在某个边界上相遇。单独写的 `Nfc` 由参数类型解析，所以调用也不需要导入；写出该类型名的代码要写 `use core::str::unicode::NormalForm;`。

### 规范化

同一段文本可以用不同的码位拼写：`"é"` 可以是 U+00E9，也可以是 `e` 后跟 U+0301 COMBINING ACUTE ACCENT。`==`、`Hash` 和 `Dict` 键比较的是字节，所以会把两者区分开；请先把两边都规范化。

```nabi
let a = "caf\u{e9}";                  // é as one code point
let b = "cafe\u{301}";                // e + combining acute
a == b;                               //→ false (different bytes)
a == b.normalize(Nfc);                //→ true
b.normalize(Nfd).len();               //→ 6
a.is_normalized(Nfc);                 //→ true
"ﬁ①".normalize(Nfkc);                 //→ "fi1"
"\u{1100}\u{1161}".normalize(Nfc);    //→ "가" (Hangul jamo compose)
```

| 形式   | 结果                                                                               |
|--------|------------------------------------------------------------------------------------|
| `Nfd`  | 标准分解，组合符号按标准顺序排列                                                   |
| `Nfc`  | 先 `Nfd`，再标准组合（组合排除项保持分解状态）                                     |
| `Nfkd` | `Nfd` 加上兼容分解（连字、全角和带圈形式、上标）                                   |
| `Nfkc` | 先 `Nfkd`，再标准组合                                                              |

韩文音节按算法分解和组合。`Nfc` 是存储和比较文本时应当使用的形式。`Nfkc` / `Nfkd` 会丢失区别（`"²"` 变成 `"2"`），所以适合用作搜索键和标识符，不适合存储。

`normalize` 首先运行快速检查（`NFC_QC`、`NFD_QC`、`NFKC_QC` 和 `NFKD_QC` 属性）：回答为 Yes 时，结果就是 `self`，不做复制；否则，从第一个不是 Yes 的字符之前最后一个稳定起始字符之后的文本开始规范化。`is_normalized` 就是这个快速检查，在回答为 Maybe 时再以一次比较收尾；对回答为 Yes 或 No 的文本，它不做任何分配。不是 UTF-8 的 `str`（操作系统原始字节）原样返回，不属于任何形式。

### 大小写折叠

```nabi
"Straße".casefold();                       //→ "strasse"
"ΣΑΣ".casefold();                          //→ "σασ"
"STRASSE".eq_ignore_case("straße");        //→ true
"Cafe\u{301}".eq_ignore_case("CAFÉ");      //→ true
```

`casefold` 应用完整大小写折叠（CaseFolding.txt，状态 C 和 F）：大多是小写映射，但也包括 `ß` → `ss`、`ς` → `σ` 以及其他只对比较有意义的差异。它与区域设置无关（不应用突厥语族的 `T` 映射）。折叠后的文本用于比较，而不是显示。`eq_ignore_case` 是标准的不区分大小写匹配：比较两边的 `NFD(casefold(NFD(x)))`，所以也忽略重音符号的拼写方式。对只有 ASCII 的文本，`char::eq_ignore_ascii_case` 就够了。

### 字素簇与单词

字素簇是读者视为一个字符的单位：带组合符号的字母、用字母（jamo）写成的韩文音节、由两个区域指示符组成的旗帜、emoji ZWJ 序列。

```nabi
let s = "e\u{301}🇰🇷👩\u{200D}💻";
s.char_count();                                //→ 7
s.grapheme_count();                            //→ 3
s.graphemes().to_list();                       //→ ["é", "🇰🇷", "👩‍💻"]
s.grapheme_slice(0, 2);                        //→ "é🇰🇷"

"can't stop, 3.14!".words().to_list();         //→ ["can't", "stop", "3.14"]
"a, b".split_word_bounds().to_list();          //→ ["a", ",", " ", "b"]
```

`graphemes` 遵循 UAX #29 默认的扩展字素簇规则（GB1-GB999，包括印度系文字的连字、emoji ZWJ 序列和区域指示符对）。`grapheme_slice` 按 `char_slice` 计数字符的方式计数字素簇：两个边界都截断到 `grapheme_count()`，`a >= b` 时得到 `""`，复杂度为 O(b)；`s.grapheme_slice(0, n)` 保留前 `n` 个用户感知的字符。

`split_word_bounds` 产生 UAX #29 默认单词边界（WB1-WB999）之间的每一段：单词、连续空格和每个标点符号，所以把各段连接起来就得到原文本。`words` 保留含有字母或数字字符的段。边界采用默认规则，不使用词典或区域设置定制：每个表意文字自成一个单词，一串泰文字母是一段。

分段通过了 UCD 18.0.0 `GraphemeBreakTest.txt` 和 `WordBreakTest.txt` 的每一行，规范化通过了 `NormalizationTest.txt` 的每一行。
