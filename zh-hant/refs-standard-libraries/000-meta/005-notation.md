# 0.5. 表示法

本規格的表示法慣例完全沿用 [refs-lang 0.8 表示法](../../refs-lang/000-meta/008-notation.md)。本章定義只適用於標準函式庫規格的額外慣例。

## 0.5.1. Trait 簽章表示法

每個 trait 的規格都以一個簽章區塊開頭，把所有方法、關聯型別與父 trait 集中在一處。

```nabi
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

這個形式與標準函式庫的實際寫法完全相同，因此使用者也可以在自己的程式碼中使用相同的格式。

## 0.5.2. 方法簽章表

定義多個方法的 trait 也會以如下的表格形式整理。

| 方法   | 簽章                           | 意義                |
|--------|--------------------------------|---------------------|
| `add`  | `fn(self, rhs: Rhs) -> Output` | 算術加法            |
| `sub`  | `fn(self, rhs: Rhs) -> Output` | 算術減法            |

## 0.5.3. Trait 實作矩陣

每個 trait 規格的結尾都有一張表，說明哪些標準函式庫型別實作了該 trait。欄位只有型別與繫結機制；實作檔案的路徑不在規格範圍內。

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

## 0.5.4. 語意

每個方法的意義都以文字寫出，不只涵蓋形式與範例，也涵蓋限制與錯誤情況。

```markdown
### `Add::add(self, rhs)`

Arithmetic addition. Returns the sum of the two values.

- `i32 + i32`: panics on overflow (checks on in every profile; wraps under `-C overflow-checks=no`)
- `f64 + f64`: IEEE 754 addition
- `Int + Int`: arbitrary precision; never overflows

Overflow handling follows the fixed-width arithmetic rules of [2.1 Numeric types](../200-core/201-numbers.md).
Saturating / checked / wrapping variants are exposed as separate methods (`saturating_add`, etc.).
```

## 0.5.5. 範例

每個 trait 與方法都附有一個以上可運作的範例。範例假設隱含的 prelude 匯入 `use core::prelude::*;` 已生效，因此必須是不需任何匯入就能直接執行的程式碼。

```nabi
let a: i32 = 5;
let b: i32 = 3;
let c = a + b;        //→ 8 (i32)

let x: f64 = 1.5;
let y: f64 = 2.5;
let z = x + y;        //→ 4.0 (f64)
```

## 0.5.6. 交互參照

參照其他文件時，依位置使用兩種形式之一。在本規格內，寫成 `[2.7 iter](../200-core/207-iter.md)`，或非正式地稱「refs-standard-libraries 2.7」；參照 refs-lang 時採用 `[refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)` 的形式。

## 0.5.7. 規格與實作的分離

本文件只定義**規格**。編譯器與標準函式庫的實際實作維護在另外的儲存庫中；若與本規格衝突，以本規格為準。

## 0.5.8. `try_` 前綴

`try_` 名稱只標示以下兩種形式之一：

1. 非阻塞的嘗試：立即執行一般形式會等待的動作，並回報無法完成（`Mutex::try_lock`、`Receiver::try_recv`、`Sender::try_send`）。
2. 把所接受的閉包或 future 的擲回傳遞出去的形式（`Iterator::try_fold`、`Iterator::try_for_each`、`par::try_scope`、`task::try_block_on`）。

不能擲回的 trait 方法，其擲回形式同樣保留 `try_` 名稱：`TryFrom::try_from`、`TryInto::try_into`、`File::try_clone`。其他函式都不使用此前綴：會失敗的函式以一般名稱擲回（0.5.9），因此不會有它的 `try_` 對應版本。`try_` 只作為前綴；沒有任何名稱以 `_try` 結尾。

## 0.5.9. 錯誤模型

每個函式都只透過一種管道回報失敗，依失敗的性質選擇：

| 失敗 | 管道 | 範例 |
|---|---|---|
| 只有一種原因的一般「沒有答案」 | `-> T \| none` | `List::get`、`Iterator::find`、`Path::parent`、`Path::strip_prefix`、`checked_*`、`char::from_u32`、`NonZero::new` |
| 由輸入、環境或 I/O 引起，或有多種呼叫端可能需要區分的原因 | `throws (E)` | `FromStr::from_str`、`str::from_utf8`、`Int::parse_radix`、`FormatSpec::parse`、`ClientConfig::new`、`fs` 與 `net` 的操作 |
| 呼叫端能事先低成本檢查的呼叫端錯誤 | panic，`#[track_caller]` | 運算子索引超出範圍、基數不在 2..=36 內、區塊大小為零 |

1. **沒有對應版本。** 不會有兩個函式只在失敗管道上不同。`try? f()` 就是每個擲回呼叫的 `T | none` 形式（[refs-lang 3.14.3](../../refs-lang/003-expressions/314-error-control-flow.md)），因此函式庫中沒有擲回函式的 `_or_none` 對應版本，也沒有會 panic 的對應版本。
2. **沒有哨兵值。** 結果絕不以其自身型別的值（`0`、`-1`、`""`）編碼失敗：要嘛是 `T | none`，要嘛函式擲回。
3. **關閉會擲回。** 作業系統資源（檔案、通訊端、TLS 連線）的 `close()` 會擲回作業系統回報的錯誤。`Drop` 絕不擲回：由 `Drop` 釋放的值會捨棄錯誤。
4. 運算子與其具名方法可以不同：`xs[i]` 會 panic，而 `xs.get(i)` 回傳 `none`。
5. 運算子不能擲回（[2.3](../200-core/203-ops.md)）：可能失敗的運算子回傳 `Result`（`Finite<F> + Finite<F>` 是 `Result<Finite<F>, OverflowError>`）。


## 0.5.10. 錯誤型別

標準函式庫中的每個錯誤型別都遵循下列規則：

1. **私有欄位。** 錯誤由建構函式（`new(..)`，或具名的建構函式，例如 `IoError::from_os_code`）建立，絕不使用結構字面值；讀取時透過以事實命名的 getter（`kind()`、`position()`、`path()`、`os_code()`）。只由函式庫產生的錯誤（`LayoutError`、`NulError`）可以沒有公開的建構函式。由多種原因組成列舉的錯誤（`FromBytesWithNulError`、`VarError`）把資料放在變體欄位中，以模式讀取。每個錯誤型別都有 `message() -> str`：說明出了什麼錯，不含其 `Display` 可能附加的上下文。
2. **Trait 組合。** 每個錯誤型別都實作 `Display`、`Debug`、`Error`、`PartialEq`、`Eq` 與 `Clone`。攜帶型別參數值的錯誤（`SendError<T>`、`TrySendError<T>`）實作前三個。
3. **種類。** 種類 getter 回傳列舉（`IoErrorKind`、`TlsErrorKind`、`JoinErrorKind`），絕不回傳 `str` 或整數，而且該列舉是 `#[non_exhaustive]`（`DecodeErrorKind` 亦同）。作業系統錯誤碼是 `os_code() -> i32 | none`，錯誤不是來自作業系統時為 `none`。
4. **單元結構。** 沒有欄位的錯誤是單元結構（`struct TimeoutError;`），寫成 `TimeoutError`，而不是 `TimeoutError {}`。
5. **每個失敗空間一個錯誤。** core 保留共用的錯誤（`ValueError`、`ParseError`、`OverflowError`、`ConversionError`、`TypeError`、`Utf8Error`）以及其自身失敗空間的錯誤（`ZeroError`、`DecodeError`、`FmtError`、`CancellationError`、`TimeoutError`）；std 模組的每個失敗空間各有一個錯誤型別（`IoError`、`TlsError`、`NabionError`；`env` 有用於讀取變數的 `VarError` 與用於 `join_paths` 的 `JoinPathsError`；`process` 在 `IoError` 之外，另有用於安裝處理器的 `SignalError`）。兩個錯誤絕不涵蓋同一種失敗。每個失敗的 UTF-8 檢查都擲回 `Utf8Error`。
6. **訊息。** 訊息以小寫開頭（專有名詞與型別名稱除外），結尾不加句點，也不加 `error:` 前綴。運算子的 panic 使用 Rust 的文字（`index 5 out of bounds for length 3`）。方法的 panic 以 `Type::method: ` 開頭（自由函式則為 `module::fn: `），接著是問題，若有替代寫法再加上 `; <what to write instead>`：`i32::isqrt: the number is negative; use `checked_isqrt` to get `none` instead`。對錯誤引數會 panic 的公開函式是 `#[track_caller]`。


## 0.5.11. 命名

標準函式庫中的每個名稱都遵循下表。違反此表的名稱是函式庫的缺陷，而不是先例。

| 種類 | 規則 |
|---|---|
| 建構函式 | `new`（必要引數）、`with_<x>`（額外的引數）、`from_<src>`（轉換，`Formatter::from_sink`）、特殊值用 `empty()` / `zeroed()`；以單位建構的函式以該單位命名（`Duration::secs(n)`、`ms`、`ns`）；`List::filled(n, x)`（n 個複本）與 `BinaryHeap::by_ordering(cmp)` 是具名的例外；絕不使用 `to_<x>` |
| 轉換 | `as_` 是不需成本的檢視，`to_` 是新的自有值（有成本），`into_` 會消耗原值；`str` 是字串型別，因此寫 `to_str`、`read_to_str`、`from_str`，絕不寫 `to_string` |
| 數值轉換 | 無損與經檢查的轉換透過 `From` / `TryFrom` / `SaturatingFrom` / `WrappingFrom`；固有的 `to_<num>` 只用於有損的捨入（`to_f64`），而在 `Int` / `Float` 上，經檢查的 `to_<num>() -> T \| none` 與 `TryFrom` 並存（[2.1.2](../200-core/201-numbers.md)）；`Ipv4Addr::to_bits` / `from_bits` 與 `From<Ipv4Addr> for u32` 及 `From<u32> for Ipv4Addr` 並存，與 Rust 相同 |
| getter | 不加 `get_` 前綴；`get` 只用於以鍵或索引查找，或取得包裝型別的值（`NonZero::get`）；唯一具名的例外是 guard 的 `get_clone()`，它是與限定作用域的 `get()` 並存的深層複製（[2.16](../200-core/216-sync.md)） |
| 述詞 | `is_` / `has_`，或 Rust 既有的用詞（`success`、`readonly`） |
| setter | `set_<x>(ref self, ..)` 回傳 unit；控制代碼（檔案、通訊端、游標或讀取器）會改變作業系統或共用狀態，因此其 setter 接受 `self`（`File::set_len`、`TcpStream::set_nodelay`、`Cursor::set_position`）；建構器使用單純的欄位名稱並回傳 `Self` |
| 大小 | `len` = 容器的元素數或位元組數；`<x>_len` = 某個長度屬性；`count` = 計數得到的數量（`count_ones`）；`size` = 記憶體大小 |
| 加入／移除 | 兩端用 `push` / `pop`，以鍵或索引用 `insert` / `remove`，多個用 `extend`，以及 `clear`、`retain`，`take` 會移出 |
| 迭代 | `iter()` 回傳 `<Type>Iter`（`BinaryHeap::iter` 回傳 `HeapIter`）；其他迭代器方法回傳以其自身名稱的 CamelCase 命名、定義於所在模組的型別（見下文），例外是 `Iterator::inspect` 回傳 `InspectIter`，因為 `Inspect` 是反射 trait（2.12），以及擁有所有權的串流配接器（`into_lines`、`into_bytes`、`into_split`、`into_take`、它們的 `_async` 形式、`into_incoming`），它們回傳包住檢視型別的 `io::Owned<V, R>`（[2.23.1](../200-core/223-io.md#2231-read)） |
| 反射 | 一般形式是型別的答案（`T::type_name()`、`T::size()`），`_of_val` 形式是值在執行時的答案（`x.type_name_of_val()`、`x.size_of_val()`、`x.field_names_of_val()`） |
| 非同步 | `<name>_async` 是阻塞的 `<name>` 的對應版本；絕不使用 `async_` 前綴 |
| 縮寫 | 完整拼出，但以下固定集合除外：`len`、`fd`、`io`、`utf8`、`ns` / `us` / `ms` / `secs` / `mins`、`tls`、`der` / `pem`、`mmap`、`pid`、`alpn`、`ocsp`、`dns`，以及 Rust 既有的短形式：`addr`、`dir`、`temp`、`exe`、`env`、`var` / `vars`、`args`、`arg`、`ty`、`os`、`std`（`Stdio`）、`ptr`、`buf`、`recv`、`ttl`、`subsec`，還有數學名稱 `abs`、`rem`、`ln`、`exp`、`cbrt`、`min` / `max` |
| 常數 | `SCREAMING_SNAKE`；常數沒有對應的函式版本（`f64::INFINITY`） |
| 模組 | 單數名詞或範圍（`fs`、`net`、`iter`）；`collections` 與 `ops` 維持複數 |

迭代器型別。`str`：`chars()` -> `Chars`、`bytes()` -> `Bytes`、`char_indices()` -> `CharIndices`、`lines()` -> `Lines`、`split()` -> `Split`、`splitn()` -> `SplitN`、`rsplit()` -> `RSplit`、`rsplitn()` -> `RSplitN`、`split_whitespace()` -> `SplitWhitespace`、`match_indices()` -> `MatchIndices`。`Dict`：`keys()` -> `Keys`、`values()` -> `Values`。`List` / `Slice`：`windows()` -> `slice::Windows`、`chunks()` -> `slice::Chunks`（與配接器 `iter::Windows` / `iter::Chunks` 並存）。`Receiver::iter()` -> `ReceiverIter`。兩個模組共用的葉名稱會連同其模組一起寫出（`slice::Windows`、`str::Bytes`）。

## 0.5.12. Trait 涵蓋範圍、`#[non_exhaustive]` 與 `#[sealed]`

標準函式庫的每個公開型別都實作其類別的 trait：

| 型別類別 | 實作 |
|---|---|
| 每個公開型別 | `Debug` |
| 值型別（位址、路徑、ID、時間、種類、演算法、範圍、規範） | `Clone`、`PartialEq`；相等關係是完整的等價關係時加 `Eq`；有 `Eq` 時加 `Hash`；值有順序時加 `PartialOrd` / `Ord`；體積小時加 `Copy` |
| 有明顯空值或零值的型別 | `Default` |
| 類文字型別 | `Display` |

依種類的 `Debug` 輸出：

- 值印出其內容：`0..3`、`0..=3`、`Start(3)`、`Permissions { mode: 0o644, readonly: false }`；位址印出其文字，與 `Display` 相同（`127.0.0.1:80`，[3.2.4](../300-std/302-net.md#324-socketaddr--ipaddr)）；`StrBuf` 印出其文字並加上引號；
- 控制代碼印出其型別以及用來識別它的資訊：`File { path: "a.txt" }`、`Child { pid: 42 }`、`Thread { id: ThreadId(3), name: "worker" }`；沒有可低成本顯示之資訊的控制代碼印出其型別名稱（`TcpStream`、`Mutex`）；
- 迭代器、串流或 future 印出其型別名稱（`Map`、`Chars`、`AsyncFuture`）；
- 持有秘密的值絕不印出秘密：它只以 `Type { field: value, .. }` 這一種形式印出其型別與公開參數：`x25519::PrivateKey { .. }`、`Hmac { .. }`、`DecapsulationKey768 { .. }`、`rsa::PrivateKey { bits: 2048, .. }`、`Secret { len: 32, .. }`。

用作 `Dict` 鍵與 `Set` 元素、會被比較與排序的函式庫型別，其值 trait 如下：

| 型別 | Trait |
|---|---|
| `Path` | `Clone`、`Eq`、`Hash`、`PartialOrd`、`Ord`（依文字）、`Default`（空）、`Display`（其文字） |
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
| `CStr`、`CString`、`Certificate`（依其 DER 位元組）、`FormatSpec` | `Clone`、`PartialEq`、`Eq`、`Hash` |
| `Range` | `Debug`、`Hash` |
| `RangeInclusive`、`RangeFrom`、`RangeTo`、`RangeToInclusive`、`RangeFull` | `Clone`、`PartialEq`、`Eq`、`Hash`、`Debug` |
| `Duration`（`ZERO`）、`StrBuf`（空）、`FormatSpec`（`FormatSpec::new()`） | `Default` |

`#[non_exhaustive]`（[refs-lang 6.2.10](../../refs-lang/006-meta-system/602-built-in-attributes.md)）標示每個可能新增變體的公開列舉，以及每個可能新增欄位的選項記錄。在其套件之外對它進行 `match` 時需要 `_` 分支，結構模式則需要 `..`：

- 列舉：`IoErrorKind`、`TlsErrorKind`、`CipherSuite`、`AeadAlgorithm`、`EcCurve`、`HashAlgorithm`、`DecodeErrorKind`、`base64::Alphabet`、`nabion::Value`、`Stdio`、`FileType`、`Signal`、`JoinErrorKind`、`fs::Access`、`net::Domain`、`net::SocketType`；
- 記錄：`tls::ClientConfig`、`tls::ServerConfig`、`FieldInfo`、`MethodInfo`、`fs::OpenFlags`、`fs::CreateFlags`、`os::unix::fs::OsOpenFlags` / `OsCreateFlags`、`os::windows::fs::OsOpenFlags` / `OsCreateFlags` / `CreateFileParams`。

封閉的列舉保持窮盡：`Ordering`、`Poll`、`Result`、`SeekFrom`、`Shutdown`、`Alignment`、`Sign`、`IpAddr`、`SocketAddr`、`Component`、`LoadOrdering`、`StoreOrdering`、`UpdateOrdering`、`hex::Case`、`base64::Padding`、`VarError`、`FromBytesWithNulError`、`SendError`、`TrySendError`。這兩份清單合起來涵蓋所有公開列舉。

`#[sealed]`（[refs-lang 6.2](../../refs-lang/006-meta-system/602-built-in-attributes.md)）標示函式庫在健全性或意義上所依賴其實作的 trait；只有定義它的套件能實作，約束與呼叫則不受影響：`AtomicPrimitive`（2.16）、`NicheInt`、`NicheFloat`（2.2）、`Pinnable`（1.6）。`Step`（2.7）保持開放，供使用者的範圍型別使用。`Pod`（1.3）並未封閉（`#[derive(Pod)]` 會檢查使用者的結構），而 `bool` 與 `char` 不是 `Pod`。

## 0.5.13. 非同步對應版本

`x_async` 是阻塞的 `x` 的非同步對應版本：引數相同、錯誤相同，以 await 等待（`try await s.x_async(..)`），並且只暫停該任務。若 `x` 就地操作其接收者，對應版本接受 `ref self`；若 `x` 會消耗接收者，則接受 `self`（`close_async`）。

串流操作的對應版本來自 trait，而不是各型別的方法：實作 `Read`、`BufRead`、`Write`、`Seek` 或 `Close` 的型別，透過實作 `AsyncRead`、`AsyncBufRead`、`AsyncWrite`、`AsyncSeek` 或 `AsyncClose`，取得 `read_async`、`fill_buf_async`、`write_async`、`flush_async`、`seek_async` 或 `close_async`（[2.23.5](../200-core/223-io.md#2235-非同步-trait)）。只有不屬於串流 I/O 的操作才是固有的 `_async` 方法：開啟與連線（`open_async`、`connect_async`）、`accept_async`、`shutdown_async`、資料包與 fd 傳遞呼叫、TLS 控制，以及行程等待（`wait_async`、`output_async`、`status_async`）。

| 型別 | 非同步 trait | 固有對應版本 |
|---|---|---|
| `File` | `AsyncRead`、`AsyncWrite`、`AsyncSeek`、`AsyncClose` | `open_async` 與其他開啟函式、`sync_all_async`、`set_len_async`、`metadata_async`、`lock_exclusive_async`、`lock_shared_async`（`unlock` 從不等待） |
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
| `BufReader`、`BufWriter`、`LineWriter`、`Cursor` | 與其內部串流相同（[2.23.6](../200-core/223-io.md#2236-緩衝區與-cursor)） | |
| `io::Owned<V, R>`、`io::Chain<A, B>` | 與其讀取器（`R`；`A` 與 `B`）相同的 `AsyncClose`（[2.23.5](../200-core/223-io.md#2235-非同步-trait)） | |

`Sender` 的 `close` 不是 I/O，沒有對應版本。`File` 或 `Dir` 的對應版本接受 `ref self`；只有 `close_async` 會消耗控制代碼。

## 0.5.14. 文件與屬性

標準函式庫的每個公開項目都有 `///` 文件註解：每個函式、型別、trait、trait 方法與固有方法，包括 `const fn` 與 `async fn`，以及位於 `cfg` 之後的項目。文件說明項目做什麼、回傳什麼；會擲回、回傳 `none` 或 panic 的函式要說明在什麼時候，`unsafe` 函式要說明呼叫端必須維持什麼。函式庫自身的檢查（`nabic check --check-library`）會拒絕 `missing_docs`（[refs-lang 6.2.9](../../refs-lang/006-meta-system/602-built-in-attributes.md)）。

結果具有順序的函式會說明該順序：

| 項目 | 順序 |
|---|---|
| `Dict`、`Set` 及其迭代器 | 插入順序 |
| `Set::union`、`Set::symmetric_difference` | 先是來自 `self` 的成員，依其順序，再是只在 `other` 中的成員，依其順序 |
| `Set::intersection`、`Set::difference` | `self` 的順序 |
| `fs::walk` | 深度優先的前序（目錄先於其項目），每個目錄的項目依名稱排序 |
| `net::resolve` | 系統解析器的順序，其偏好的位址在前 |
| `env::vars`、`env::vars_os` | 作業系統保存環境的順序；不排序 |
| `task::race` | 最先完成的任務勝出；在同一次喚醒時發現已完成的多個任務，由索引最小者勝出；其他任務會被取消 |
| `task::select` | 依清單順序輪詢；平手時由索引較小者勝出 |

`#[must_use]`（[refs-lang 6.2.11](../../refs-lang/006-meta-system/602-built-in-attributes.md#6211-unused-results-unused_result-discardable-must_use)）標示下列項目，每個型別都附有說明如何使用該值的訊息：

| 種類 | 項目 |
|---|---|
| 迭代器 | `str` 的迭代器（`Chars`、`CharIndices`、`Split`、`Lines`、...）、`List`（`ListIter`）、`Slice`（`SliceIter`、`Windows`、`Chunks`）、`SliceMut`（`SliceMutIter`）、`Dict`（`Keys`、`Values`、`DictIter`）、`Set`（`SetIter`）、`Deque`（`DequeIter`）、`BinaryHeap`（`HeapIter`）與 `Receiver`（`ReceiverIter`）的迭代器；`iter` 配接器與 `TryIterator` 配接器（`TryMap`、`MapErr`、`Results`、...）；`GenIter`、`TryGenIter`；`env::Vars`、`env::VarsOs`；`fs::ReadDir`、`fs::Walk`；`io::Lines`、`io::Split`、`io::Bytes`；`net::Incoming`；`path::Components`、`path::Ancestors` |
| 串流 | `AsyncStream`、`TryAsyncStream`；`stream` 配接器與組合子以及 `TryStream` 配接器；`io::BytesStream`、`io::LinesStream`；`fs::ReadDirStream` |
| Future | `async fn` 呼叫或 `async` 區塊的值；`LockFuture`、`ReadFuture`、`WriteFuture`、`CondvarWait`、`RecvFuture`、`SendFuture`、`task::Join`、`SelectFuture` |
| 建構器與結果 | `process::Command`、`thread::Builder`、`thread::JoinHandle`、`Result` |
| 函式 | `str` 與 `Path` 的值方法；每個 `Duration` 函式，以及 `Instant`、`SystemTime` 與 `BootTime` 的算術（`duration_since`、`checked_add`、`checked_sub`）；`Iterator`、`DoubleEndedIterator` 與 `TryIterator` 的純終端方法（`find`、`rfind`、`min`、`max`、`min_by_key`、`max_by_key`、`reduce`、`last`、`position`）；`Set::union`、`intersection`、`difference`、`symmetric_difference`；整數與浮點數基本型別、`Int` 與 `Float` 的每個方法 |

`Task`，也就是 `spawn` 與 `spawn_blocking` 回傳的值，不是 `#[must_use]`：無論是否 await，工作都會執行。guard 不需要標示：在陳述式中釋放 `MutexGuard` 是錯誤。`#[discardable]` 函式即 refs-lang 6.2.11 所列的那些。
