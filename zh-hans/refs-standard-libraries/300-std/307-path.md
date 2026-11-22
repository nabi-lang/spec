# 3.7. `path::`：UTF-8 路径（Python 风格）

`std::path` 模块定义路径的表示。要点是它把 UTF-8 newtype 放在首位：既避免 Rust 中 `Path` 与 `OsStr` 分立带来的使用负担，又采用 Python `pathlib.Path` 的直观性。

## 3.7.1. 两种类型：`Path` / `RawPath`

Nabi 的路径表示分为两种类型。

| 类型      | 含义                                          | 使用频率              |
|-----------|-----------------------------------------------|-----------------------|
| `Path`    | UTF-8 路径（95% 以上）                        | 普通代码的默认选择    |
| `RawPath` | 操作系统原生字节（Unix 字节 / Windows WTF-16） | 1%（非 UTF-8 文件名） |

大多数代码只需要 `Path`。只有在确实需要遗留字节的地方，例如处理老旧的 unix 系统时，才需要 `RawPath`。

## 3.7.2. `Path`：UTF-8

`Path` 是不可变值。`join`、`/` 和 `with_*` 都返回新的 `Path`；没有原地修改路径的 `PathBuf` / `push`。按照普通复合赋值规则，`p /= q` 就是 `p = p / q`。

`Path` 的接口如下。

```nabi
struct Path { /* UTF-8 str newtype, GC heap */ }

impl Path {
    fn new(s: str) -> Path;
    fn empty() -> Path;
    fn from_components<I: IntoIterator<Item = str>>(components: I) -> Path;

    // decomposition
    fn parent(self) -> Path | none;
    fn file_name(self) -> str | none;
    fn file_stem(self) -> str | none;
    fn extension(self) -> str | none;
    fn components(self) -> Components;   // implements Iterator<Item = Component>
    fn ancestors(self) -> Ancestors;     // implements Iterator<Item = Path>

    // manipulation
    fn join<P: AsPath>(self, other: P) -> Path;   // a str too
    fn with_file_name(self, name: str) -> Path;
    fn with_extension(self, ext: str) -> Path;
    fn strip_prefix<P: AsPath>(self, base: P) -> Path | none;   // none when base is not a prefix

    fn is_absolute(self) -> bool;
    fn is_relative(self) -> bool;
    fn has_root(self) -> bool;
    fn starts_with<P: AsPath>(self, base: P) -> bool;
    fn ends_with<P: AsPath>(self, suffix: P) -> bool;
    fn normalize(self) -> Path;             // resolve "." / ".." (lexical)

    // OS interaction (convenience; fs::* internally)
    fn exists(self) -> bool;
    fn is_file(self) -> bool;
    fn is_dir(self) -> bool;
    fn is_symlink(self) -> bool;
    fn metadata(self) throws (IoError) -> Metadata;
    fn canonicalize(self) throws (IoError) -> Path;     // realpath (verified against the filesystem)

    fn as_str(self) -> str;
    fn to_raw(self) -> RawPath;
}

enum Component {
    RootDir,                    // "/"
    Prefix(str),                // "C:" (Windows)
    CurDir,                     // "."
    ParentDir,                  // ".."
    Normal(str),                // "data"
}

impl Component {
    fn as_str(self) -> str;     // as written in a path: "/", ".", "..", the name
}
```

## 3.7.3. `/` 运算符：Python pathlib 风格

引入 `/` 运算符是为了让路径拼接更易读。

```nabi
impl Div<str> for Path {
    type Output = Path;
    #[infix("/")]
    fn div(self, rhs: str) -> Path { self.join(Path::new(rhs)) }
}

impl Div<Path> for Path { ... }

impl From<str> for Path { ... }   // Path::new
impl Display for Path { ... }     // the text: /etc/nginx
impl Debug for Path { ... }       // the quoted text: "/etc/nginx"
```

也允许 `p /= "x"` 和 `p /= q`（`q: Path`），含义是 `p = p / ..`。

下面的示例清楚地展示了**效果**。

```nabi
let p = Path::new("/etc") / "nginx" / "nginx.conf";
                                          //→ "/etc/nginx/nginx.conf"

let home = env::home_dir()? / ".config" / "myapp" / "settings.toml";
```

这提供了与 Python `Path("/etc") / "nginx" / "nginx.conf"` **相同的使用体验**，也避免了 Rust `PathBuf::join` 和 `push` 的冗长。

## 3.7.3a. `AsPath`：路径参数

```nabi
trait AsPath {
    fn as_path(self) -> Path;
}
impl AsPath for str { ... }
impl AsPath for Path { ... }
```

`fs` 的路径参数以 `AsPath` 接收，所以 `str` 和 `Path` 都可以传入（`fs::read("a.txt")`、`fs::read(dir / "a.txt")`）。

## 3.7.4. UTF-8 不变式

`Path` **始终是 UTF-8**。`Path::new(s)` 接收 `str`，而 `str` 已经是 UTF-8，所以它不做任何转换或验证。

操作系统可以保存非 UTF-8 的名称：Unix 上的任意字节、Windows UTF-16 中未配对的代理项。`fs` 和 `env` 从不对这样的名称做有损转换：本应把它作为 `Path` 返回的调用会抛出带 `InvalidData` 的 `IoError`。代码从别处（例如通过 FFI）得到的操作系统字节构成 `RawPath`（3.7.5），需要显式转换：

```nabi
let raw: RawPath = ...;
let p: Path = try Path::try_from(raw);    // throws Utf8Error if not UTF-8
let q: Path = try raw.try_into();          // same
```

这个转换是 `impl TryFrom<RawPath> for Path { type Error = Utf8Error; ... }`（[2.11.4](../200-core/211-conversion.md#2114-tryfromu-trait---fallible-conversion)）。

## 3.7.5. `RawPath`：字节 / WTF-16

`RawPath` 用于需要原样使用操作系统原生表示的地方。

```nabi
struct RawPath { /* windows: List<u16>, elsewhere: List<u8> */ }   // + Clone, PartialEq, Eq, Hash, Debug

impl RawPath {
    fn from_bytes<B: AsBytes>(bytes: B) -> RawPath;    // every target but windows
    fn from_wide(wide: Slice<u16>) -> RawPath;         // windows

    fn as_bytes(self) -> Slice<u8>;                    // every target but windows
    fn as_wide(self) -> Slice<u16>;                    // windows only

    fn to_path(self) throws (Utf8Error) -> Path;       // as RawStr::to_str
    fn to_path_lossy(self) -> Path;
}
```

`to_path` 抛出 `Utf8Error`（[2.15.6](../200-core/215-string.md)），即 `str::from_utf8` 和 `RawStr::to_str` 的错误；在 Windows 上，它的 `valid_up_to()` 以 UTF-16 单元计数。`to_path_lossy` 把每个最长的非法子序列（Unix）/ 每个孤立代理项（Windows）替换为一个 U+FFFD。

大多数代码从不会遇到这个类型。它只出现在代码直接与操作系统交换系统调用的地方。

## 3.7.6. canonicalize 与 normalize

这两个函数看起来相似，但行为和开销明显不同。

| 函数             | 含义                          | 文件系统访问        |
|------------------|-------------------------------|---------------------|
| `normalize()`    | 纯词法清理（解析 ".."）       | **否**              |
| `canonicalize()` | 解析符号链接并转换为绝对路径  | **是**（`realpath`） |

```nabi
Path::new("/a/b/../c").normalize();              //→ "/a/c" (lexical)
try Path::new("/a/b/../c").canonicalize();       //→ absolute path with symlinks resolved
```

`normalize` 速度快，不做系统调用，所以 90% 的场合用它就够了。

## 3.7.7. WASI 兼容性

`Path` 也能自然地与 WASI Preview 2 的**能力路径**（capability path）结合。

```nabi
let dir: Dir = try Dir::root().open_dir("data");
let f = try dir.open(Path::new("config.toml"));
                                          // Path is relative to dir; escape rejected
```

能力模型在 [3.1 fs](301-fs.md) 中详细介绍。

## 3.7.8. 与其他语言的比较

为了明确这一设计决定，下面与相近的语言进行比较。

| 方面          | Rust                      | Python `pathlib` | Go `filepath`   | Nabi                         |
|---------------|---------------------------|------------------|-----------------|-----------------------------|
| 类型          | `Path` / `PathBuf`        | `Path`           | `string`        | `Path`                      |
| 拥有与视图    | 借用 `&Path` / 拥有       | **不可变**       | **不可变**      | **不可变**（`/` 生成新的 Path） |
| `/` 运算符    | **无**（显式 `.join()`）  | `/`              | **无**          | `/`                         |
| 非 UTF-8      | `OsStr`（混用）           | `bytes`（分开）  | `string`（易混淆） | 单独的 `RawPath`         |
| canonicalize  | `.canonicalize()`         | `.resolve()`     | `filepath.Abs`  | `.canonicalize()`           |
| 方法链        | 可以（大多数情况）        | 很好             | 可以            | 很好                        |

总之，Nabi **最大的决定**是把 UTF-8 默认与 `/` 运算符结合起来，让 Python pathlib 的使用体验与 Rust 级别的性能兼得。
