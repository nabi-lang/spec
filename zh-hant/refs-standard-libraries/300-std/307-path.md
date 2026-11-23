# 3.7. `path::`：UTF-8 路徑（Python 風格）

`std::path` 模組定義路徑的表示方式。重點在於以 UTF-8 newtype 為優先：既避免 Rust 將 `Path` 與 `OsStr` 分開所帶來的使用負擔，又採用 Python `pathlib.Path` 的直覺用法。

## 3.7.1. 兩種型別：`Path` / `RawPath`

Nabi 的路徑表示分為兩種型別。

| 型別      | 意義                                       | 使用頻率      |
|-----------|-----------------------------------------------|-----------------------|
| `Path`    | UTF-8 路徑（95% 以上）                             | 一般程式碼的預設選擇 |
| `RawPath` | 作業系統原生位元組（Unix 位元組 / Windows WTF-16） | 1%（非 UTF-8 檔名） |

大多數程式碼只需要 `Path`。只有真正需要舊式位元組的地方，例如處理老舊的 unix 系統時，才需要 `RawPath`。

## 3.7.2. `Path`：UTF-8

`Path` 是不可變的值。`join`、`/` 和 `with_*` 都回傳新的 `Path`；沒有就地修改路徑的 `PathBuf` / `push`。依照一般的複合指派規則，`p /= q` 就是 `p = p / q`。

`Path` 的介面如下。

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

## 3.7.3. `/` 運算子：Python pathlib 風格

引入 `/` 運算子是為了讓路徑串接更容易閱讀。

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

也允許 `p /= "x"` 和 `p /= q`（`q: Path`），意義為 `p = p / ..`。

從下面的範例可以清楚看出其**效果**。

```nabi
let p = Path::new("/etc") / "nginx" / "nginx.conf";
                                          //→ "/etc/nginx/nginx.conf"

let home = env::home_dir()? / ".config" / "myapp" / "settings.toml";
```

這提供與 Python 的 `Path("/etc") / "nginx" / "nginx.conf"` **相同的易用性**，並避免 Rust 的 `PathBuf::join` 與 `push` 的冗長寫法。

## 3.7.3a. `AsPath`：路徑引數

```nabi
trait AsPath {
    fn as_path(self) -> Path;
}
impl AsPath for str { ... }
impl AsPath for Path { ... }
```

`fs` 的路徑引數以 `AsPath` 接收，因此 `str` 與 `Path` 都可以傳入（`fs::read("a.txt")`、`fs::read(dir / "a.txt")`）。

## 3.7.4. UTF-8 不變式

`Path` **永遠是 UTF-8**。`Path::new(s)` 接收的 `str` 本身已經是 UTF-8，因此不做任何轉換或驗證。

作業系統可能保存非 UTF-8 的名稱：Unix 上的任意位元組，Windows UTF-16 中未成對的代理項。`fs` 與 `env` 絕不會有損地轉換這類名稱：原本會以 `Path` 回傳這類名稱的呼叫，會擲回帶有 `InvalidData` 的 `IoError`。由程式碼從其他途徑（例如透過 FFI）取得的作業系統位元組所建構的 `RawPath`（3.7.5），則以明確的方式轉換：

```nabi
let raw: RawPath = ...;
let p: Path = try Path::try_from(raw);    // throws Utf8Error if not UTF-8
let q: Path = try raw.try_into();          // same
```

這個轉換是 `impl TryFrom<RawPath> for Path { type Error = Utf8Error; ... }`（[2.11.4](../200-core/211-conversion.md#2114-tryfromu-trait可能失敗的轉換)）。

## 3.7.5. `RawPath`：位元組 / WTF-16

`RawPath` 用於需要原樣使用作業系統原生表示的地方。

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

`to_path` 擲回 `Utf8Error`（[2.15.6](../200-core/215-string.md)），即 `str::from_utf8` 與 `RawStr::to_str` 的錯誤；在 Windows 上，其 `valid_up_to()` 以 UTF-16 單元計數。`to_path_lossy` 將每個最大的格式不正確子序列（Unix）/ 每個孤立的代理項（Windows）替換為一個 U+FFFD。

大多數程式碼從不會遇到這個型別。它只出現在程式碼直接與作業系統交換系統呼叫的地方。

## 3.7.6. canonicalize 與 normalize

這兩個函式看起來相似，但行為與成本有明顯差異。

| 函式         | 意義                       | 檔案系統存取  |
|------------------|-------------------------------|---------------------|
| `normalize()`    | 純粹在文字層面整理（解析 ".."） | **否**     |
| `canonicalize()` | 解析符號連結並轉換為絕對路徑 | **是**（`realpath`） |

```nabi
Path::new("/a/b/../c").normalize();              //→ "/a/c" (lexical)
try Path::new("/a/b/../c").canonicalize();       //→ absolute path with symlinks resolved
```

`normalize` 速度快且不做系統呼叫，因此在 90% 的場合都足以勝任。

## 3.7.7. WASI 相容性

`Path` 也能自然地與 WASI Preview 2 的**能力路徑**整合。

```nabi
let dir: Dir = try Dir::root().open_dir("data");
let f = try dir.open(Path::new("config.toml"));
                                          // Path is relative to dir; escape rejected
```

能力模型在 [3.1 fs](301-fs.md) 中有詳細說明。

## 3.7.8. 與其他語言的比較

為了釐清這項設計決策，以下與相近的語言比較。

| 面向        | Rust                      | Python `pathlib` | Go `filepath`   | Nabi                         |
|---------------|---------------------------|------------------|-----------------|-----------------------------|
| 型別          | `Path` / `PathBuf`        | `Path`           | `string`        | `Path`                      |
| 擁有 vs 檢視 | 借用的 `&Path` / 擁有所有權  | **不可變**    | **不可變**   | **不可變**（`/` 產生新的 Path） |
| `/` 運算子  | **無**（明確的 `.join()`） | `/`          | **無**        | `/`                         |
| 非 UTF-8     | `OsStr`（混合）           | `bytes`（分開） | `string`（易混淆） | 獨立的 `RawPath`   |
| canonicalize  | `.canonicalize()`         | `.resolve()`     | `filepath.Abs`  | `.canonicalize()`           |
| 方法串接 | 可以（大多數情況）             | 優秀        | 可以              | 優秀                   |

總而言之，Nabi **最大的決策**是 UTF-8 預設與 `/` 運算子的組合，將 Python pathlib 的易用性與 Rust 等級的效能結合在一起。
