# 2.15. 字串：`str` / `char` / `c_char`

`core::str` 模組提供字串與字元型別的權威定義。Nabi 採行 **UTF-8 優先**的方針：行程內的每個字串都以 UTF-8 處理，`c_char` 只用於 FFI 邊界。

`str`、`char` 與 `c_char` 是編譯器內建的基本型別，不是結構。編譯器不掛接它們的 lang 槽位（`str_type`、`char_type`、`cchar_type`）（[0.4](../000-meta/004-lang-slot-table.md)）；本模組定義它們的方法與 trait 實作。

## 2.15.1. 字串模型：不可變的 `str` 與可變的 `StrBuf`

`str` 是不可變的 UTF-8 字串值；用來建構字串的可變緩衝區是另一個型別 `StrBuf`。

| 型別     | 意義                                               | 記憶體表示                   | 可變性              |
|----------|----------------------------------------------------|------------------------------|---------------------|
| `str`    | UTF-8 字串值                                       | `(ptr, len)` 值；位元組是永生常數或 GC 物件 | **不可變** |
| `StrBuf` | 可成長的 UTF-8 緩衝區                              | GC 堆積物件                  | 可變                |
| `char`   | Unicode 純量值（語言內建）                         | 4B (u32)                     | （值，不能改變）    |
| `c_char` | C ABI 的 1 位元組 char（FFI 基本型別）             | 1B（i8 或 u8，依目標而定）   | （值）              |

### Rust / Python / Nabi 模型比較

三種語言處理字串的方式：

| 面向           | Rust             | Python             | Nabi                       |
|----------------|------------------|--------------------|----------------------------|
| 值             | `&str`           | `str`              | `str`（不可變的 `(ptr, len)`） |
| 可變緩衝區     | `String`         | `io.StringIO` 等   | `StrBuf`                   |
| 記憶體         | 借用檢查器       | 參考計數           | GC                         |

Nabi 的 `str` 不需要追蹤生命週期，這點接近 Python 不可變的 `str`；而它的表示方式與 Rust 的 `&str` 相同，是一個 `(ptr, len)` 對。執行時建立的字串，其位元組位於 GC 物件中（型別 ID `1`，[refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-runtime-reserved-type-ids)）。

## 2.15.2. `str`：不可變的 UTF-8 字串

`str` 是編譯器內建的基本型別，因此不需要宣告、結構或 `#[keyword]`。它的值是一個 `(ptr, len)` 對：位元組的位址，以及以位元組計的長度（不是字元數）。

`str` 是以值傳遞的 `(ptr, len)` 對，它所指向的位元組絕不會改變。每個修改字串的操作都回傳新的 `str`。

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

`splitn` 與 `rsplitn` 最多產出 `n` 段；最後一段是字串未分割的其餘部分，`n == 0` 則什麼都不產出。`rsplitn` 從尾端開始搜尋，因此 `"a.b.c".rsplitn(2, ".")` 先產出 `"c"`，再產出 `"a.b"`。與 `split` 相同，空的 `sep` 會把整個字串當作一段產出。`match_indices` 由左至右為每個比對產出一個 `(usize, str)` 對；比對成功後從其後方繼續搜尋，因此 `"aaaaa".match_indices("aa")` 產出位移 0 與 2。空的 `pat` 不比對任何東西。`trim_matches` 從兩端移除 `pat` 的完整重複（`"--x--".trim_matches("-") == "x"`、`"aaa".trim_matches("aa") == "a"`）；空的 `pat` 不改變字串。

```nabi
"key=a=b".splitn(2, "=");               //→ "key", "a=b"
"x.tar.gz".rsplitn(2, ".");             //→ "gz", "x.tar"
"ab-ab".match_indices("ab");            //→ (0, "ab"), (3, "ab")
"--x--".trim_start_matches("-");        //→ "x--"
```

`as_ptr` 回傳指向字串原地位元組的指標。在可能出現指標值的函式中（變數或運算式的型別指名某個 `bare::ptr` 型別，包括作為型別引數（`List<ConstPtr<u8>>`）或出現在函式型別中），只要持有該 `str` 的變數仍在作用域內，GC 就不會移動該字串，指標也保持有效。在產生器或 `async` 主體中，這點跨越 `yield` / `await` 仍成立。在其他地方，變數只在其最後一次使用之前持有該字串，而只能透過其他物件到達的字串可能被 GC 移動：它並未被固定（[stdlib 1.6](../100-bare/106-gc-pin.md)）。若要讓位元組更長時間留在原處，請固定該字串（[1.6.6](../100-bare/106-gc-pin.md#166-取出方法)）。

### 依 Python 命名補上的方法

```nabi
fn reversed(self) -> str;                       // characters in reverse order; Python s[::-1]
fn count(self, pat: str) -> usize;              // non-overlapping occurrences; "" counts len+1
fn replacen(self, from: str, to: str, n: usize) -> str;   // the first n occurrences
fn to_titlecase(self) -> str;                   // each word's first letter titlecased, the rest lowercased
```

`reversed` 反轉碼位。`count("")` 與 Python 相同，是字元數加一。在 `to_titlecase` 中，一個單字從不接在字母、數字或撇號之後的字母開始，因此 `"don't"` 仍是一個單字（與 Python 的 `title()` 不同）；對應採用完整的 Unicode 對應（2.15.4）。

### `StrBuf`：可變緩衝區

要一段一段地建構字串，請使用 `StrBuf`。

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

與集合型別一樣，`StrBuf` 是參考控制代碼，因此其方法接受 `self`（[2.9](209-collections.md)）；控制代碼的複本共用同一個緩衝區，所以它不是 `Sync`（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）。它顯示為其內容。

```nabi
let buf = StrBuf::new();
for x in xs {
    buf.push_str(f"{x},");
}
let s: str = buf.to_str();
```

### 字串字面值：`"hello"`

```nabi
let s: str = "hello";        // a str pointing to immortal constant bytes
```

`"hello"` 字面值降階為一個 `str` 值，指向永生區域中的常數位元組。由於 `str` 不可變，因此沒有複製，也沒有寫入時複製（copy-on-write）。

### trait 實作

| trait                       | 實作                           | 行為                                       |
|-----------------------------|--------------------------------|--------------------------------------------|
| `Display`                   | ✓                              | 原樣輸出自身                               |
| `Debug`                     | ✓                              | 帶跳脫的 `"..."`                           |
| `Clone`                     | ✓                              | 相同的值（不可變，因此共用位元組）         |
| `Copy`                      | ✓                              | 複製 `(ptr, len)` 值                       |
| `PartialEq`                 | ✓                              | 內容比較（逐位元組）                       |
| `Eq`                        | ✓                              |                                            |
| `PartialOrd`                | ✓                              | 依位元組的字典序                           |
| `Ord`                       | ✓                              |                                            |
| `Hash`                      | ✓                              |                                            |
| `IntoIterator`              | **無**                         | 明確寫 `.chars()`；避免位元組與字元的歧義  |
| `Add<str>` (`s + t`)        | ✓ → str（串接，新的 str）      |                                            |
| `Add<char>` (`s + c`)       | ✓ → str（附加 `c`，新的 str）  |                                            |
| `AddAssign<str>`, `AddAssign<char>` (`s += t`, `s += c`) | ✓ | `s = s + t`：新的 str 取代 `s`          |
| `FromIterator<char>`        | ✓                              | `(chars).collect::<str>()`                 |
| `FromIterator<str>`         | ✓                              | `(strs).collect::<str>()`                  |

> [!NOTE]
> `str` 是不可變的值，因此 `let t = s;` 只複製 `(ptr, len)`，任何一方都不能改變內容。

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

`str` / `StrBuf` 的邊界是位元組位移。邊界先依 Python 的方式夾限（負數從尾端算起，[refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-slicing-arrstartend)）；若所得位置落在 UTF-8 序列中間，則觸發 panic。結果是與原字串共用位元組的 `str`（見下文）。`str` 切片沒有步幅；`s[::2]` 是型別錯誤。

### 字元邊界與不會觸發 panic 的切片

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

當 `r.start > r.end`、任一邊界超過 `len()`，或任一邊界不是字元邊界時，`get` 回傳 `none`；否則即為 `s[r.start:r.end]`。對任何 `i >= len()`，`floor_char_boundary` 與 `ceil_char_boundary` 回傳 `len()`。`char_slice` 依 `chars()` 產出的順序從 0 開始計算字元；兩個邊界都夾限到 `char_count()`，`a >= b` 得到 `""`。它會走訪字串，O(b)。若要依使用者感知的字元切片，請使用 `grapheme_slice`（[2.15.10](#21510-unicode-文字正規化大小寫摺疊分段)）。

不在字元邊界上的切片會觸發 panic，訊息會指出這些方法：

```text
string slice is not on a char boundary: byte 8 is inside `세` (bytes 7..10)
string offsets count bytes; to work by character, use `chars()`, `char_indices()` or `char_slice(a, b)`
to cut at a boundary, use `floor_char_boundary(i)`; `get(a..b)` gives `none` instead of a panic
```

### 不允許直接索引

```nabi
let s: str = "hello";
let c = s[0];                  // ERROR: `Index<usize>` is not defined
                               //         reason: byte vs char ambiguity + UTF-8 safety
```

必須明確寫出意圖，例如 `s.chars().next()` 或 `s.byte(0)`。

### 子字串不會被複製

對 `str` `s` 而言，`s[a:b]`、`lines`、`split*`、`rsplit*`、`split_whitespace`、`split_once`、`match_indices`、`trim*` 與 `strip_*` 的結果都是與原字串共用位元組的 `str` 值，耗時 O(1)（[refs-lang 8.8.1](../../refs-lang/008-abi/808-lowering-types.md#881-primitive-integers--floating-point)）。因此一小段字串會讓龐大的原字串保持存活。沒有將其分離的 API。`as_bytes()` 同樣是不複製的唯讀 `Slice<u8>` 檢視。`StrBuf` 的切片則是持有該範圍複本的新 `str`，O(b − a)：`str` 不可變，因此不能共用一個仍可能改變的緩衝區。

## 2.15.4. `char`：Unicode 純量值

`char` 是語言內建型別（[2.1](201-numbers.md)）：一個 4 位元組的 Unicode 純量值，不是結構。本模組定義它的方法。

Unicode 純量值的範圍是 0..=0xD7FF 與 0xE000..=0x10FFFF 的聯集。代理字元範圍 0xD800..=0xDFFF 被用作**自動 niche**，因此 `char | none` 可以用 4B 編碼。

### 字面值／方法

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

`char` 的 `to_lowercase` / `to_uppercase` / `to_titlecase` 依完整的 Unicode 對應回傳一個 `str`，結果可能有數個字元（`'ß'.to_uppercase() == "SS"`、`'ß'.to_titlecase() == "Ss"`、`'ǆ'.to_titlecase() == "ǅ"`、`'ﬁ'.to_titlecase() == "Fi"`、`'\u{130}'.to_lowercase() == "i\u{307}"`）。`to_simple_*` 形式依 UCD 簡單對應回傳一個 `char`，沒有對應時回傳字元本身（`'ß'.to_simple_uppercase() == 'ß'`、`'ǆ'.to_simple_titlecase() == 'ǅ'`）；它們絕不配置記憶體。`str` 的 `to_lowercase` / `to_uppercase` 套用完整對應，包括 1:N 對應與上下文規則（希臘文詞尾 sigma `Σ` → `ς`）（`"ß".to_uppercase() == "SS"`）。大小寫對應與地區設定無關。若要做不分大小寫的比較，請使用完整的大小寫摺疊（`casefold`，[2.15.10](#21510-unicode-文字正規化大小寫摺疊分段)），而不是 `to_lowercase`。

Unicode 屬性表與大小寫表採用 UCD 18.0.0。

### 轉換

```nabi
char::from_u32(0x41);                //→ 'A' (char | none; none for a surrogate)
char::from_digit(5, 10);             //→ '5' (char | none; none if 5 is not a digit of the radix)

let n: u32 = c as u32;               //→ 65 (lossless)
let c2: char = n as char;            //→ ERROR: only u8 can `as char`
let b: u8 = 0x41;
let c4: char = b as char;            //→ 'A' (a u8 is always a scalar value)
let c3 = try char::try_from(n);      //→ char (TryFrom<u32>: via from_u32, OverflowError if none)
```

### trait 實作

`char` 實作所有 cmp / fmt trait，以及 `Copy`、`Clone` 與 `Hash`。niche 自動套用，因此 `char | none` 為 4B。

## 2.15.5. `c_char`：C ABI 的 1 位元組（語言內建）

`c_char` 是編譯器內建的 FFI 基本型別，不是結構。它佔一個位元組；它是有號（`i8`）還是無號（`u8`）取決於目標（OS／架構）。本模組依其位元組值（`c as u8`）比較、排序與雜湊它。它實作 `Clone`、`Copy`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash` 與 `Debug`，`Debug` 把位元組印成數字（`65`）；它沒有 `Display`。

它與 C 的 `char` ABI 相容；它是原始位元組，不是 UTF-8 編碼單元。它只用於 FFI：

```nabi
extern "C" {
    fn strlen(s: ConstPtr<c_char>) -> usize;
    fn strcmp(a: ConstPtr<c_char>, b: ConstPtr<c_char>) -> i32;
}
```

### 字面值

```nabi
let c: c_char = c'A';        //→ 65 (i8/u8, target-dependent)
let n: c_char = c'\n';       //→ 10
```

帶前綴的形式 `c'...'` 將它與 `char` 區分開來。

### char 與 c_char

比較這兩個型別即可看清差異。

| 面向         | `char`                              | `c_char`                      |
|--------------|-------------------------------------|-------------------------------|
| 大小         | 4 位元組                            | 1 位元組                      |
| 意義         | Unicode 純量（碼位）                | C ABI 位元組                  |
| 範圍         | 0..=0x10FFFF（不含代理字元）        | 0..=255 或 -128..=127         |
| 多位元組     | 單一 char 即涵蓋 BMP / SMP          | 多位元組字元 = 位元組序列     |
| 字面值       | `'A'`                               | `c'A'`                        |
| 註冊方式     | 語言內建                            | 語言內建                      |
| 用途         | 一般字串處理                        | FFI／位元組層級處理           |

因此，標準函式庫函式原則上**優先使用 `char`**，只在明確的 FFI 邊界使用 `c_char`。

## 2.15.6. UTF-8 不變式：行程內的每個字串

`str` 始終保證是有效的 UTF-8，理由如下。建構函式會驗證：`str::from_utf8(bytes)` 遇到無效的位元組序列時擲回 `Utf8Error`（`throws (Utf8Error)`）。切片會驗證位元組邊界，拒絕落在多位元組序列中間的位置。`StrBuf::push` 或 `push_str` 等原地修改也會維持 UTF-8 完整性。只有 `unsafe` 程式碼能繞過驗證：`StrBuf::push_byte`（不經檢查附加一個位元組的 `unsafe fn`）與 `bare::mem::transmute_unchecked`。

### 為何限制繞過驗證

想想 UTF-8 不變式**被破壞**時會出什麼問題：`chars()` 可能觸發 panic 或無限迴圈，截斷的多位元組序列可能顯示錯誤，還可能出現過長編碼（overlong encoding）之類的安全性問題。

由於標準函式庫保證在建構時驗證，使用者程式碼中的 `str.chars()` 等操作始終安全。不進入不安全程式碼，就無法破壞這個不變式。

### 位元組 ↔ str 轉換

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

`Utf8Error` 實作 `Display`（即其 `message()`）、`Debug`、`Error`、`PartialEq`、`Eq` 與 `Clone`。它是唯一的 UTF-8 錯誤：標準函式庫中每個失敗的 UTF-8 檢查都擲回它（`str::from_utf8`、`CStr::to_str`、`RawStr::to_str`、`RawPath::to_path`）。

`str::from_utf8_lossy(bytes)` 絕不擲回：不合格序列的每個最大子部分（Unicode §3.9，U+FFFD Substitution of Maximal Subparts；即 WHATWG UTF-8 解碼器的行為）都變成一個 U+FFFD。

### UTF-16

```nabi
let u = "h\u{E9}\u{1F600}".encode_utf16();    //→ [104, 233, 55357, 56832]
let s = try str::from_utf16(u);               //→ "hé😀"
let t = try str::from_utf16([0x61u16, 0x62u16, 0xDC00u16]);
                                              //→ throws a Utf16Error, valid_up_to() == 2
str::from_utf16_lossy([0x61u16, 0xD800u16]);  //→ "a\u{FFFD}"
```

`encode_utf16` 給出字串的 UTF-16 碼元，U+FFFF 以上的字元以代理對表示，不含 NUL（要傳給 C 的 NUL 結尾字串，請用 `bare::ffi::WideCString`，[stdlib 1.7.12](../100-bare/107-ffi.md#1712-widecstr--widecstringwchar_t-字串)）。`str::from_utf16` 從任何 `IntoIterator<Item = u16>` 取回碼元：`List<u16>`、`Slice<u16>` 或 `[u16; N]`。它在第一個未配對的代理字元處擲回 `str::Utf16Error`：後面沒有接低位代理的高位代理、單獨出現的低位代理，或位於結尾的高位代理。`str::from_utf16_lossy` 以一個 U+FFFD 取代每個未配對的代理字元，絕不擲回。

```nabi
struct Utf16Error { .. }      // str::Utf16Error; private fields (0.5.10)
impl Utf16Error {
    fn valid_up_to(self) -> usize;   // length of the valid prefix, in units
    fn message(self) -> str;         // "unpaired surrogate at unit {valid_up_to}"
}
```

`Utf16Error` 實作 `Display`（即其 `message()`）、`Debug`、`Error`、`PartialEq`、`Eq` 與 `Clone`。`bare::ffi::WideCStr::to_str` 也會擲回它。

## 2.15.7. `FromStr` trait：剖析

```nabi
trait FromStr {
    type Err;
    fn from_str(s: str) throws (Err) -> Self;
}
```

`str.parse::<T>()` 是 `throws (T::Err)`，並透過這個 trait 分派。對數值型別、`bool`、`char` 與 `str` 而言，`Err` 是 core 的 `ParseError`（`input()`、`position()`、`message()`）（[2.14.4](214-error.md#2144-內建錯誤型別)）。`char` 剖析恰好一個字元的字串，失敗時回報 `empty string` 或 `too many characters`；`str` 回傳其輸入，絕不擲回。數值剖析建立在執行環境輔助函式 `nabi_str_parse_int` / `nabi_str_parse_float` 之上（[refs-lang 8.5](../../refs-lang/008-abi/805-extern-symbols.md#string-helpers)）。浮點數的 `FromStr` 在有限的數字超出型別範圍時以 `number out of range` 失敗；`inf` / `NaN` 文字可以剖析。

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

`FromStr` 與 [2.11 轉換](211-conversion.md)中的 `TryFrom<str>` **是分開的**。它只處理 `str → T` 的轉換，並明確表達可能失敗。

## 2.15.8. 模組對應

各型別的註冊方式與實作位置：

| 型別     | 註冊方式          | 實作                                 |
|----------|-------------------|--------------------------------------|
| `str`    | 語言內建          | 編譯器（方法：標準函式庫）           |
| `StrBuf` | 普通結構          | 標準函式庫                           |
| `c_char` | 語言內建          | 編譯器（方法：標準函式庫）           |
| `char`   | 語言內建          | 編譯器（方法：標準函式庫）           |

`str` 與 `char` 的方法主體呼叫在 `core::rt` 中宣告為 `extern "nabi"` 函式的執行環境輔助函式：`nabi_str_*`（長度、比較、搜尋、切片、大小寫對應與摺疊、修剪、剖析、正規化、分段）與 `nabi_char_*`（Unicode 屬性與大小寫）（[refs-lang 8.5](../../refs-lang/008-abi/805-extern-symbols.md#string-helpers)）。它們不帶 `#[intrinsic]` 屬性。

## 2.15.9. 與其他語言的比較

| 面向            | Rust              | Python 3                | Java                    | Nabi                               |
|-----------------|-------------------|-------------------------|-------------------------|------------------------------------|
| 型別劃分        | `&str` + `String` | `str`（單一）           | `String`                | `str` + `StrBuf`                   |
| 可變性          | 只有 `String` 可變 | **不可變**             | **不可變**              | `str` 不可變，`StrBuf` 可變        |
| 編碼            | UTF-8             | 未規定（依實作而定）    | UTF-16                  | UTF-8                              |
| char            | `char` (4B)       | `str`（單字元）         | `char`（2B = 碼元）     | `char`（4B Unicode 純量）          |
| FFI 位元組      | `c_char` / `u8`   | `bytes`                 | `byte`                  | `c_char`（1B，依目標而定）         |
| len 的意義      | 位元組長度        | 字元數                  | UTF-16 碼元             | 位元組長度（明確使用 `.char_count()`） |
| 索引 `s[i]`     | **無**            | 字元（Python str）      | UTF-16 碼元             | **無**（明確使用 chars/bytes）     |

`str` 是不可變的 `(ptr, len)` 值，可變緩衝區則獨立為 `StrBuf`；有了 GC，就不像 Rust 那樣需要區分生命週期。`len()` 表示**位元組長度**，與 Rust 一致；字元數另以 `char_count()`（或 `chars().count()`）取得。藉由**不提供** `s[i]`，Nabi 與 Rust 一致，並避免位元組與字元的歧義。把 `char` 定義為 4B 的 Unicode 純量，可避免 Java 2B UTF-16 碼元所帶來的代理字元陷阱。最後，`c_char` 是獨立的型別，與 `char` 保持清楚的 ABI 區分。

## 2.15.10. Unicode 文字：正規化、大小寫摺疊、分段

`core::str::unicode` 模組收納文字在碼位之外所需的功能：正規化形式（UAX #15）、完整的大小寫摺疊，以及切分為字素叢集與單字的分段（UAX #29）。它的方法位於 `str` 上，因此不需要匯入。這些表格是執行環境中的 UCD 18.0.0，與 `char` 屬性的版本相同（2.15.4）。

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

`Graphemes`、`WordBounds` 與 `Words` 是 `str` 切片的迭代器，與原字串共用位元組。`Graphemes` 也是 `DoubleEndedIterator`：`next_back` 與 `rev()` 從尾端給出相同的叢集，兩端在某個邊界上相遇。單獨的 `Nfc` 從參數型別解析，因此呼叫時也不需要匯入；指名該型別的程式碼寫 `use core::str::unicode::NormalForm;`。

### 正規化

同一段文字可以用不同的碼位拼寫：`"é"` 是 U+00E9，也可以是 `e` 後接 U+0301 COMBINING ACUTE ACCENT。`==`、`Hash` 與 `Dict` 鍵比較的是位元組，因此會區分這兩者；請先將兩邊正規化。

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

| 形式   | 結果                                                                               |
|--------|------------------------------------------------------------------------------------|
| `Nfd`  | 標準分解，組合符號依標準順序排列                                                   |
| `Nfc`  | 先 `Nfd`，再標準合成（合成排除的字元維持分解）                                     |
| `Nfkd` | `Nfd` 加上相容分解（合字、全形與圈字形式、上標）                                   |
| `Nfkc` | 先 `Nfkd`，再標準合成                                                              |

韓文音節以演算法分解與合成。`Nfc` 是儲存與比較文字時應採用的形式。`Nfkc` / `Nfkd` 會失去區別（`"²"` 變成 `"2"`），因此適合用於搜尋鍵與識別字，不適合用於儲存。

`normalize` 先執行快速檢查（`NFC_QC`、`NFD_QC`、`NFKC_QC` 與 `NFKD_QC` 屬性）：答案為 Yes 時，結果就是 `self`，不做複製；否則從第一個不是 Yes 的字元之前最後一個穩定起始字元（stable starter）之後的文字開始正規化。`is_normalized` 就是這個快速檢查，在答案為 Maybe 時以比較收尾；對於答案為 Yes 或 No 的文字，它不配置任何記憶體。不是 UTF-8 的 `str`（原始 OS 位元組）原樣回傳，且不屬於任何形式。

### 大小寫摺疊

```nabi
"Straße".casefold();                       //→ "strasse"
"ΣΑΣ".casefold();                          //→ "σασ"
"STRASSE".eq_ignore_case("straße");        //→ true
"Cafe\u{301}".eq_ignore_case("CAFÉ");      //→ true
```

`casefold` 套用完整的大小寫摺疊（CaseFolding.txt，狀態 C 與 F）：大多是小寫對應，但也包括 `ß` → `ss`、`ς` → `σ` 以及其他只對比較有意義的差異。它與地區設定無關（不套用突厥語系的 `T` 對應）。摺疊後的文字用於比較，不用於顯示。`eq_ignore_case` 是標準的不分大小寫比對：比較兩邊的 `NFD(casefold(NFD(x)))`，因此也忽略重音的拼寫方式。對於只含 ASCII 的文字，`char::eq_ignore_ascii_case` 就足夠了。

### 字素叢集與單字

字素叢集是讀者視為一個字元的單位：一個字母及其組合符號、以字母（jamo）書寫的韓文音節、由兩個區域指示符組成的旗幟、emoji ZWJ 序列。

```nabi
let s = "e\u{301}🇰🇷👩\u{200D}💻";
s.char_count();                                //→ 7
s.grapheme_count();                            //→ 3
s.graphemes().to_list();                       //→ ["é", "🇰🇷", "👩‍💻"]
s.grapheme_slice(0, 2);                        //→ "é🇰🇷"

"can't stop, 3.14!".words().to_list();         //→ ["can't", "stop", "3.14"]
"a, b".split_word_bounds().to_list();          //→ ["a", ",", " ", "b"]
```

`graphemes` 依循 UAX #29 預設的擴充字素叢集規則（GB1-GB999，包括印度系文字的連寫、emoji ZWJ 序列與區域指示符對）。`grapheme_slice` 計算叢集的方式與 `char_slice` 計算字元相同：兩個邊界都夾限到 `grapheme_count()`，`a >= b` 得到 `""`，O(b)；`s.grapheme_slice(0, n)` 保留前 `n` 個使用者感知的字元。

`split_word_bounds` 產出 UAX #29 預設單字邊界（WB1-WB999）之間的每一段：單字、連續的空白，以及每個標點符號，因此把各段接起來即可還原文字。`words` 保留含有字母或數字字元的段落。這些邊界是預設邊界，不使用字典或地區設定的調整：每個表意文字自成一個單字，一串連續的泰文字母則是一段。

分段通過 UCD 18.0.0 `GraphemeBreakTest.txt` 與 `WordBreakTest.txt` 的每一行，正規化則通過 `NormalizationTest.txt` 的每一行。
