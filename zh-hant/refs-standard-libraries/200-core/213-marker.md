# 2.13. marker：`Clone` / `Copy` / `Drop`

複製值，以及清理不再需要的資源，是任何語言都無法迴避的問題。`core::marker` 模組以三個 trait 表達這兩個問題。`Clone` 定義如何製作值的獨立副本；`Copy` 是一個標記，表示對某個型別而言單純的位元複製就足夠；`Drop` 定義值離開作用域時必須執行的清理程式碼。這個模組也包含 `NoneSentinel`，即 `none` 字面值之型別在函式庫中的名稱（2.13.8）。

Nabi 使用 GC，這直接影響這些 trait 的意義。記憶體的回收由 GC 負責，因此 `Drop` 與記憶體釋放無關。`Drop` 只限於清理位於語言之外的資源的終結工作，例如檔案控制代碼和通訊端。這使 Nabi 的標記 trait 與 Rust 和 Java/Python 都不同。

## 2.13.1. 記憶體模型：有 GC 的語言中的 Clone/Drop

在 Nabi 的執行環境中，[Immix + Sticky Mark-Bit 分代 GC](../../refs-lang/007-runtime/703-gc-algorithm.md) 自動管理堆積物件的生命週期。使用者不決定物件何時從記憶體中消失。然而，外部資源的清理必須以確定的方式發生，因此 Nabi 的 `Drop` 是一個獨立於 GC 的機制。與 Rust 相比：

| 面向               | Rust（無 GC）                 | Nabi（GC）                                          |
|----------------------|------------------------------|---------------------------------------------------|
| 記憶體回收   | 由 `Drop` 負責        | 由 GC 自動處理；`Drop` **不負責**             |
| `Drop::drop` 的意義 | 釋放資源 + 釋放記憶體 | **僅清理外部資源**（檔案控制代碼、通訊端……） |
| 呼叫時機          | 確定（離開作用域時）   | 確定：在最後一個擁有者離開作用域之處，或 `using` 區塊結束時（與 GC 回收無關） |
| 省略時的影響    | 資源洩漏 + 記憶體洩漏  | 僅資源洩漏（GC 最終會回收記憶體） |

> [!NOTE]
> `Drop` 值是仿射的（affine）：編譯器在其最後一個擁有者離開作用域之處，或在繫結它的 `using` 區塊結束時呼叫 `Drop::drop`（[refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)）；關閉可能失敗的型別必須明確關閉（2.13.4）。GC 永遠不會呼叫 `drop`。

由於記憶體回收與資源清理是分開的，一般物件不需要在意 `Drop`（由 GC 回收），只有處理外部資源的型別才實作 `Drop`，以取得確定的清理時間點。

## 2.13.2. `Clone`：深層複製

`Clone::clone` 是需要值的獨立副本時呼叫的方法。它的簽章很簡單：

```nabi
#[lang("clone_trait")]
trait Clone {
    fn clone(self) -> Self;
}
```

`clone` 執行深層複製。即使值在內部持有參考，它們所指向的資料也會一併複製，因此之後雙方可以各自獨立地修改。GC 保證不可變的共用資料仍然可以原樣共用。

### `Clone` 的意義

下面的範例以最直接的形式展示 `Clone`。

```nabi
let a: List<i32> = [1, 2, 3];
let b = a.clone();           // new List, with a new internal buffer
b.push(4);                   // a is unaffected
a;                           //→ [1, 2, 3]
b;                           //→ [1, 2, 3, 4]
```

這裡的「深層」精確地說是指：`Clone` 保持邏輯上的相等，同時保證兩個值之間的修改互不影響。由此得出兩個性質。在 `let b = a.clone();` 之後立即有：若定義了 `PartialEq`，則 `a == b` 成立。在那之後，對 `a` 的任何修改都不會反映到 `b`，反之亦然。

### 不自動呼叫：`clone_trait` lang 槽位

在某些語言中，寫下 `let y = x;` 會讓編譯器自動呼叫 `clone`。Nabi 在這裡選擇明確的一方。

```nabi
let xs: List<i32> = [1, 2, 3];
let ys = xs;          // GC heap object → reference shared (no clone)
```

對 GC 堆積物件，`let y = x;` 共用參考；對 `Copy` 型別，則執行位元複製。因此 `Clone::clone` 永遠不會被自動分派；它只在使用者明確呼叫時執行。基於這個原因，`Clone` 不是登記為方法槽位，而是登記為 trait 槽位 `clone_trait`，編譯器只在展開 `#[derive(Clone)]` 時查找這個槽位。

### 預設主體

基本型別的 `clone` 等同於位元複製，因此直接對應到一個內建函式；集合型別的 `clone` 會配置新的內部緩衝區，並遞迴地 clone 各個元素。

```nabi
impl Clone for i32 {
    #[intrinsic("nabi.clone.copy")]    // bitwise copy
    fn clone(self) -> i32;
}

impl<T: Clone> Clone for List<T> {
    fn clone(self) -> List<T> {
        let r: List<T> = [];
        for x in self {
            r.push(x.clone());
        }
        r
    }
}
```

clone 一個 GC 物件，一定會配置新的緩衝區並遞迴地 clone 各個元素。

### `#[derive(...)]`

對大多數所有欄位都是 `Clone` 的使用者定義結構，沒有理由手寫主體。`#[derive(Clone)]` 會代為產生。

```nabi
#[derive(Clone)]
struct Config {
    name: str;
    timeout: u32;
    retries: u32;
}

let c1 = Config { name: "default", timeout: 30, retries: 3 };
let c2 = c1.clone();         // every field cloned recursively
```

衍生的主體對每個欄位呼叫 `Clone::clone`，並以相同的結構回傳結果。因此每個欄位都必須遞迴地實作 `Clone`。

## 2.13.3. `Copy`：標記

```nabi
trait Copy: Clone {
    // marker: no methods
}
```

在 Nabi 中，每個值都以值的方式複製。對集合型別與 GC 物件，複製的是控制代碼，指向同一個物件；基本型別的值、`str`（不可變的 `(ptr, len)`）、結構、列舉和元組則複製其本身（[refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-value-semantics)）。因此 `Copy` **只是一個標記**，不改變指派或引數傳遞的意義。`Copy` 只用在泛型約束（`T: Copy`）中，用來選出「位元複製就是獨立副本」的型別。

### impl 政策

| 型別                             | `Clone`     | `Copy`             | 備註                                     |
|----------------------------------|-------------|--------------------|-------------------------------------------|
| `i*`, `u*`, `f*`                 | ✓           | ✓                  | 基本數值                         |
| `bool`, `char`, `c_char`         | ✓           | ✓                  |                                           |
| `str`                            | ✓           | ✓                  | 不可變的 `(ptr, len)` 值              |
| `int`, `float`                   | ✓           | ✓                  | 不可變的值；堆積上的 BigInt 只會被共用  |
| `List<T>`, `Dict<K,V>`, `Set<T>` | ✓ (T:Clone) | ✗                  | 複製控制代碼是共用，而不是獨立副本  |
| 元組                           | ✓（全部）     | ✓（若全部為 Copy）    | 逐個分量                             |
| `Range<T>`                       | ✓ (T:Clone) | ✓ (T:Copy)         | 當 start/end 兩者都是 Copy 時              |
| 使用者的 struct / enum               | derive      | derive             | 當每個欄位都實作相同的 trait 時 |

實作 `Drop` 的型別不能是 `Copy`。

```nabi
#[derive(Clone, Copy)]
struct Vec3 { x: f64; y: f64; z: f64; }
```

> [!IMPORTANT]
> 只有在同時實作了 `Clone`，而且每個欄位都是 `Copy` 時，才允許衍生 `Copy`。

## 2.13.4. `Drop`：釋放時的清理

開啟的檔案必須關閉；鎖上的互斥鎖必須解鎖。這類外部資源的清理不能交給 GC。把資源一直持有到無法預測的回收時刻，可能耗盡檔案描述元，或讓其他程式碼卡在一個從未釋放的鎖上。`Drop` trait 與 `using` 運算式解決這個問題。

```nabi
trait Drop {
    #[lang("drop_method")]
    fn drop(ref self);
}
```

### 所有權

型別實作了 `Drop` 的值是仿射的：它只有一個擁有者，並且只釋放一次。持有這種值的結構、元組、列舉或和型別也是仿射的。

- `let b = a` 會移動該值：`b` 擁有它，之後使用 `a` 是編譯錯誤（E0133）。把它存入結構、元組、列舉或容器，在閉包或 `async` 區塊中擷取它，以及回傳它，都以相同方式移動它。不持有這種值的變體（`Stdio::Piped`）的值，沒有可以移動的東西。
- 呼叫只有在被呼叫端保留該值時才會取走它：存起來、回傳、關閉，或傳給會保留它的呼叫。其他任何呼叫都是借用它，呼叫端仍然擁有它。立即被 await 的 `async fn` 呼叫也屬於這種呼叫：除非函式主體保留了引數，否則引數會在 `await` 結束時回到呼叫端。回來的是呼叫端自己的值，不是被呼叫端的副本：被呼叫端走過的串流，會停在呼叫之前的位置（要從它停下的地方繼續，就從被呼叫端回傳它）。透過函式值或閉包的呼叫沒有主體可以檢查，因此會取走該值。
- 迭代容器時看到的元素（`for x in list`）或從容器讀出的元素，仍然屬於該容器。
- `clone()`（`File` 用 `try_clone()`）會建立一個有自己擁有者的獨立值。
- 串流配接器（`r.lines()`、`r.take(n)`、`l.incoming()`）是串流的有作用域檢視，不是第二個擁有者：它停留在擁有者的作用域內（E0139，[2.23.1](223-io.md#2231-read)）；`into_` 形式（`r.into_lines()`）則會取走串流。
- 只有透過 `marker::replace(ref s.f, v)` 或 `marker::take(ref s.f)`，才能把一部分從結構中移出，它們會在原位留下新的值。
- `unsafe { marker::share(x) }` 建立一個編譯器不追蹤的第二個控制代碼：它既不移動也不釋放 `x`。在擁有者釋放該值之後使用它，須由程式自行排除。

```nabi
// core::marker
pub unsafe fn share<T>(x: T) -> T;
#[discardable]
pub fn replace<T>(ref dest: T, value: T) -> T;     // puts `value` in `dest`, returns the old value
#[discardable]
pub fn take<T: Default>(ref dest: T) -> T;         // replace(ref dest, T::default())
```

```nabi
let (tx, rx) = channel::<i32>();
let other = tx;          // moves: `tx` is gone
try tx.send(1);          // error: `tx` was moved at line 2, and is used here
let copy = other.clone(); // a second sender with its own owner
```

### 呼叫時機

編譯器在擁有者離開作用域之處，於每一種離開方式（正常完成、`return`、`break`、錯誤傳遞）釋放值；任務完成或被取消時，則釋放其 `async` 區塊所擷取的值。`using` 在其區塊結束時釋放它的值（[refs-lang 3.17 `using`](../../refs-lang/003-expressions/317-using-expression.md)）。已經移走的值由新的擁有者釋放，而不是在這裡。重新宣告擁有者名稱的 `let`（[refs-lang 5.1.4](../../refs-lang/005-modules/501-variable-declaration.md#514-redeclaration-in-the-same-scope)）既不移動也不釋放舊值：舊值在舊繫結離開作用域之處釋放，而必須關閉的值要在重新宣告之前關閉或移走。

關閉可能失敗的型別實作 `io::Close`（`fn close(self) throws (IoError)`）：`File`、包著 `File` 的 `BufWriter` 或 `LineWriter`，以及任何持有這些值的結構。編譯器不會代你關閉這種值，因為關閉時的錯誤會遺失：用 `try x.close()` 關閉，或以 `using` 繫結。在某條路徑上不關閉就離開其作用域是編譯錯誤；只為一次借用它的呼叫而建立的暫時值（`(try File::open(p)).read_to_str()`、`io::BufReader::new(try File::open(p)).lines()`）也是編譯錯誤：呼叫之後沒有任何東西擁有它，因此沒有任何東西會關閉它（E0134）；請以 `using` 繫結它（`using f = try File::open(p) { try f.read_to_str() }`）。帶著錯誤離開的路徑會釋放它（這時關閉本身的錯誤會被丟棄）。`try?` 會丟棄它攔截到的錯誤，因此對錯誤可能持有這種值的呼叫使用 `try?`（`SendError<File>` 會交回未送出的值）是編譯錯誤：請用 `try .. catch` 攔截錯誤，取出值並關閉它。失敗的 `try!` 會觸發 panic，因此它持有的錯誤不需要關閉。持有這種值的和型別或列舉按成員關閉：看到該值的 `match` 分支要關閉它，而只看到由編譯器釋放之成員（`Sender`）的分支則不需要關閉。模式將和型別收窄成的成員（`if let x: Pair = p`、`match` 分支、`let .. else`）會像結構一樣逐部分追蹤：關閉 `x.a` 之後 `x.b` 仍未關閉，而再次關閉 `x.a`，或之後整體使用 `p`，都是錯誤。

- 正常完成、`return`、`break` 和錯誤傳遞時都會呼叫。
- 多個繫結以宣告的相反順序釋放（LIFO）。
- panic 會結束行程，因此不會呼叫 `drop`。
- drop glue：在值本身的 `drop` 之後，具有 `Drop` 的欄位會以宣告的相反順序遞迴地清理。元組元素，以及列舉 / 和型別目前持有的酬載也一樣。

```nabi
fn process(path: str) throws (IoError) {
    using lock = MUTEX.lock(), file = try File::open(path) {
        do_work(file);
    }   // ← on normal exit or error: file.drop() then lock.drop()
}

{
    let f = try! File::open("data.txt");
}   // ← error: `f` holds a `File` that is never closed
```

以 `using` 繫結未實作 `Drop` 的型別是編譯錯誤，錯誤會說明需要 `Drop`。

### 與 GC 的關係

GC 永遠不會呼叫 `drop`，也沒有終結器：釋放的時間點就是上述各處，在編譯時就已確定。

### `Drop` 實作範例

最典型的實作是包裝 OS 資源。

```nabi
extern "C" { fn close(fd: i32) -> i32; }

struct Fd {
    handle: i32;
}

impl Drop for Fd {
    fn drop(ref self) {
        unsafe { close(self.handle); }    // OS file descriptor close; its result is ignored
    }
}
```

`Fd` 的關閉結果會被忽略，因此由編譯器隱含地釋放它。關閉可能失敗的型別（`File`）還會實作 `io::Close`，並如上所述明確關閉。

### `Drop` 的限制

`Drop` 帶有幾項限制。第一，實作 `Drop` 的型別不能是 `Copy`：一方面以位元複製任意增加副本，一方面又保證確定的清理，兩者互相矛盾。

```nabi
#[derive(Copy)]              // ERROR - conflicts with Drop
struct File { handle: i32; }
impl Drop for File { ... }
```

第二，直接呼叫 `drop` 是編譯錯誤。只有在釋放時恰好呼叫一次，才能保證正確性。

```nabi
let f = File::open(path);
f.drop();                    // ERROR: calling Drop::drop directly is forbidden
```

要提早關閉資源，請另外提供一個方法，例如 `File::close()`。

唯一的例外是 `bare` 中的不安全程式碼。例如 `unsafe fn drop_in_place`（[1.3 mem](../100-bare/103-mem.md)）是用於手動記憶體管理的函式。它執行 drop glue：先執行型別本身的 `drop`，再如上所述以宣告的相反順序處理其欄位。

此外，`drop` 沒有 `throws` 簽章，因此它的主體不能擲回錯誤。`drop` 中的 panic 和其他 panic 一樣會結束行程。

```nabi
impl Drop for X {
    fn drop(ref self) {
        if some_cond { throw "oops" }    // ERROR: drop is not throws
    }
}
```

### derive 政策

與 `Clone` 不同，`Drop` 不能衍生。無論在哪個釋放時間點，值本身的 `drop` 結束之後，編譯器都會以宣告的相反順序，對該結構中具有 `Drop` 的欄位呼叫 `drop`。因此使用者撰寫的 `Drop` 主體只處理「欄位之外需要清理的部分」。

```nabi
struct Container {
    a: File;
    b: Socket;
}
// when a Container is released, release does b.drop() → a.drop()
// (because the fields themselves implement Drop)

impl Drop for Container {
    fn drop(ref self) {
        log("Container released");
        // field drops are called automatically by the compiler _after this method returns_
    }
}
```

## 2.13.5. lang 槽位對應

這個模組透過四個 lang 槽位與編譯器連接。自動呼叫的時間點依照上述政策。

| 槽位          | 附加於                  | 自動呼叫時間點                       |
|---------------|------------------------------|--------------------------------------------|
| `clone_trait` | `Clone` trait                | 無；用於 `#[derive(Clone)]` 的展開  |
| `drop_method` | `Drop::drop` 方法          | 每次釋放：最後一個擁有者離開作用域、任務結束、`using` 區塊結束（以及 drop glue） |
| `send_trait`  | `Send` trait（2.13.9）        | 自動實作，執行緒邊界檢查 |
| `sync_trait`  | `Sync` trait（2.13.9）        | 自動實作，執行緒邊界檢查 |

詳情請見 [0.4 lang 槽位表](../000-meta/004-lang-slot-table.md) 的 0.4.2 / 0.4.3 / 0.4.5 節。

## 2.13.6. trait 實作矩陣

標準函式庫的主要型別與這三個 trait 的關係：

| 型別                             | Clone       | Copy           | Drop                            | 行為                             |
|----------------------------------|-------------|----------------|---------------------------------|--------------------------------------|
| `i*`, `u*`, `f*`                 | ✓           | ✓              | ✗                               | `nabi.clone.copy`（逐位元）           |
| `bool`, `char`, `c_char`         | ✓           | ✓              | ✗                               | 逐位元                              |
| `str`                            | ✓           | ✓              | ✗                               | 複製不可變的 `(ptr, len)` 值 |
| `List<T>`, `Dict<K,V>`, `Set<T>` | ✓ (T:Clone) | ✗              | ✗                               | 遞迴 clone                      |
| 元組                           | ✓（全部）     | ✓（全部為 Copy）   | ✗                               | 逐個分量                        |
| `Range<T>`                       | ✓ (T:Clone) | ✓ (T:Copy)     | ✗                               | 逐欄位（`start`、`end`）           |
| `File`                           | ✗           | ✗              | ✓（另有 `io::Close`）            | 關閉 OS 控制代碼；明確關閉（2.13.4） |
| `MutexGuard<T>`                  | ✗           | ✗              | ✓（於釋放時）                  | 解鎖互斥鎖                    |
| `channel::Sender<T>`             | ✓           | ✗              | ✓                               | 呼叫 `close()`；隱含地釋放 |

這張表背後的原則很簡單。每個基本型別與上列的集合型別都實作 `Clone`，只有位元複製在語意上正確的型別才是 `Copy`。集合型別是 `Clone` 但不是 `Copy`，因為控制代碼的位元複製無法保持深層複製的意義。`str` 是 `Copy`：它是不可變的，因此複製其 `(ptr, len)` 就是獨立副本。只有 `File`、`Socket`、`MutexGuard` 這類外部資源的包裝會實作 `Drop`。這些型別通常不實作 `Clone`，以防止資源被重複擁有。

## 2.13.7. 與其他語言的比較

把 Nabi 的模型與既有語言比較，能讓設計意圖更清楚。

| 面向                  | Rust                     | Java/Python（GC）                | Nabi（GC + Drop）          |
|-------------------------|--------------------------|---------------------------------|--------------------------|
| 記憶體回收      | 由 `Drop` 負責    | GC                              | GC                       |
| 確定的終結  | `Drop`（由編譯器保證） | **無**（終結器不確定） | `Drop`（綁定作用域） |
| Copy 的意義         | 逐位元 + 強制 `!Drop` | **無**（每個物件都是參考） | 逐位元 + 強制 `!Drop` |
| Clone 的意義        | 深層複製                | `clone()`（Object.clone）        | 深層複製                |
| derive                  | `#[derive(Clone, Copy)]` | `Cloneable` 介面           | `#[derive(Clone, Copy)]` |

Nabi 的特點在於 GC 與確定的 `Drop` 並存。由於記憶體回收由 GC 處理，`Drop` 並不代表記憶體釋放，因此一般物件永遠不需要實作 `Drop`。只有需要清理外部資源的型別才實作 `Drop`；編譯器在其最後一個擁有者離開作用域之處釋放它們，而關閉可能失敗的型別，則由使用者以 `close()` 或 `using` 關閉。

這個模型同時避免了 Java/Python 終結器的不確定性，以及 Rust 因沒有 GC 而付出的代價。

## 2.13.8. `NoneSentinel`：`none` 字面值的型別

[refs-lang 1.9.2](../../refs-lang/001-lexical/109-other-literals.md) 的 `none` 字面值具有編譯器內建的 `none` 型別，這是一個只有單一值的零大小型別。`core::marker` 以型別別名為這個型別取了函式庫中的名稱 `NoneSentinel`；不涉及任何 lang 槽位。

```nabi
pub type NoneSentinel = none;
```

使用者很少直接寫出這個名稱。在 `T | none` 這類和型別寫法中（[refs-lang 2.6](../../refs-lang/002-types/206-sum-types.md)），`none` 的位置就是這個型別；`none` 字面值、`none` 型別寫法和 `NoneSentinel` 都指同一個內建型別。

## 2.13.9. `Send` / `Sync`：執行緒邊界標記

```nabi
#[lang("send_trait")]
trait Send { }      // safe to move to another thread

#[lang("sync_trait")]
trait Sync { }      // safe to read from multiple threads through shared references
```

這兩個 trait 都是沒有方法的**自動標記 trait**；編譯器依結構實作它們。

- 基本數值、`bool`、`char`、`c_char`、`str` 和 `none` 是 `Send + Sync`。
- `struct`、`enum`、`union`、元組、陣列或和型別，若其所有欄位（分量）都是 `Send` 則為 `Send`，若都是 `Sync` 則為 `Sync`。泛型容器的 `Send` 依相同規則跟隨 `T`。
- 集合型別（`List`、`Dict`、`Set`、`StrBuf`、`Deque`、`BinaryHeap`、`Slice`、`SliceMut`）是 `Send` 且 `!Sync`：它們共用可變的控制代碼。要跨執行緒使用，可以凍結它們以唯讀方式共用（`FrozenList`、`FrozenDict`、`FrozenSet`，在元素為 `Send + Sync` 時也是 `Send + Sync`，[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s唯讀集合型別)），透過交給 `par` 的 `chunks_mut` 檢視分別寫入互不重疊的部分（[2.19.1](219-par.md#2191-核心-api)），或透過 `Mutex` / `RwLock` / 通道共用。
- 交出一個值（`Mutex::new(xs)`、`RwLock::new(xs)`、`tx.send(xs)` / `try_send` / `send_async`、`task::spawn_blocking_with_arg(xs, f)`、閉包的 `[move xs]` 擷取、`xs.freeze()`）之後，傳送端的控制代碼就不能再使用：新的擁有者得到一份副本（見下文的執行時複製），而傳送端不得使用它已交出的東西。從 guard 取出的控制代碼（`get`）、交給 guard 的控制代碼（`set`），或 `update` 的閉包所接觸的控制代碼，在 guard 的 `using` 區塊結束之後仍然指向受保護的物件。之後在任何路徑上使用這種值都是編譯錯誤（已交出的值為 `E0105`，從 guard 取出的值為 `E0106`；[refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-diagnostics--error-messages)）。編譯器會沿著以下各處追蹤該值：
  - 副本（`let ys = xs`、結構欄位或元組元素、欄位指派）；
  - 傳遞它的欄位（`Mutex::new(s.items)`）；
  - 持有它的物件（`all.push(xs)`、擷取它的閉包），雙向追蹤：交出 `xs` 也會讓 `all` 不能再使用；
  - 集合型別的元素。存入集合的東西與從集合讀出的東西（`xs[i]`、`get`、迭代、它的切片）是同一個值，因此交出任一方都會讓另一方不能再使用（`let x = xs[0]; tx.send(xs); x.len()`、`tx.send(xs[0]); xs[1].len()`）。在元素已被交出的集合上，只允許 `len`、`is_empty`、`capacity`、`clear`、`truncate`、`reserve`、`shrink_to_fit`、`push` 和 `insert`（以及 `Deque` 的 `push_back` / `push_front`）；`clear` 會清空它，因此之後就能完全正常使用。從另一個集合複製而來的集合（`extend`、`+=`、`collect`、`par::filter`）與來源共用元素。以 `pop` 或 `remove` 取出的元素（`List`、`Deque`、`Dict`、`BinaryHeap`）已經離開集合：交出它之後集合仍可使用，除非集合可能持有該物件兩次：同一個值存了兩次、兩個存入的值共用一個物件，或在迴圈中把一個集合的元素存入另一個集合（元素不以位置區分）；
  - 交出某物之直接呼叫的引數：透過兩個引數傳入同一個物件，而被呼叫端透過其中一個交出它、透過另一個使用它（`f(tx, a, a)`；`h.go(tx, h.items)`，其中 `go` 使用 `self.items`，即一個從中 pop 並送出、再加上其一個元素的串列），是編譯錯誤。同一個集合的兩個元素視為不同（`f(tx, rows[0], rows[1])` 可被接受）；
  - `ref` 參數。當被呼叫端交出它的一部分，並在每條交出它的路徑上為那一部分賦予新值（`tx.send(self.buf)` 之後的 `self.buf = []`），呼叫端的變數會取回新值並保持可用；指向舊值的其他控制代碼則不行。以值傳遞的參數是副本：為其欄位賦予新值不會傳到呼叫端，呼叫端持有的仍然是已交出的東西；
  - 呼叫。交出參數、回傳參數或其一部分、把它存入另一個參數的物件或 `ref` 參數、把它擷取到所回傳的閉包中，或回傳 guard 參數所持有之物的函式，對其呼叫端也有同樣效果，跨越預先建置的函式庫亦然（[refs-lang 9.4.7](../../refs-lang/009-tools/904-nabihir-bytecode.md#947-body)）。

  透過函式值的呼叫，視為可能到達每一個以其參數型別作為值使用的函式。條件只在它所測試的值在每條路徑上都為編譯器所知時才會追蹤：由常數設定或複製而來的 `bool`、和型別或列舉變體（`let sent = false; if c { tx.send(xs); sent = true; } if !sent { xs.len() }`），以及呼叫如何結束。只有在沒有錯誤地回傳，或回傳 `none` / `false` 以外的值時才交出參數的函式，在其呼叫端代表另一種結果的 `catch`、`try?`、`??`、`else` 或 `if` 路徑上不會交出它；對 `async fn` 使用 `try await` 時的 `catch` 也一樣。其他條件不追蹤：在某條路徑上交出的值，在該路徑與其他路徑匯合之處就消失了。

  要繼續使用某個值，請交出 `clone()`（一個獨立的值），或先為變數或欄位賦予新值（對結構副本而言，只有該欄位）；`set` 會釋放先前 `get` 所回傳的東西。`Sync` 值的使用、讀取值的標籤（`x is none`、`match x`），以及 `unsafe` 內的使用（[refs-lang 4.15.2](../../refs-lang/004-functions/415-unsafe.md#4152-unsafe------block-expression)）不受此限。被啟動之閉包的一般擷取不需要檢查：閉包與建立它的程式碼共用這些值，因此它們必須是 `Send + Sync`。`[move x]` 擷取把 `x` 交給閉包（[refs-lang 4.6.11](../../refs-lang/004-functions/406-capture-semantics.md#4611-move-capture-the-move-x-capture-list)），只需要 `Send`。把擁有非 `Sync` 之值的閉包（或未標記 `Sync`、因而可能擁有這種值的 `fn(..) + Send` 值）傳給型別為不帶 `Sync` 的 `fn(..) + Send` 的參數，就是交出它，因為被呼叫端可能在另一個執行緒上執行它（`thread::spawn`、`task::spawn_blocking`、`par::join`、`par::Scope::spawn`、`par::TryScope::spawn`……）；之後無論它經過什麼途徑（在一次呼叫中出現兩次、經過回傳其引數的函式、從串列或元組中取出），使用它都是編譯錯誤（`E0133`）。`async [move ..]` 閉包與其呼叫所產生的 future 是同一個擁有者：交出其中之一會連同其他一起交出（[refs-lang 4.6.11](../../refs-lang/004-functions/406-capture-semantics.md#4611-move-capture-the-move-x-capture-list)）。`task::spawn` 會交出它的 future：任務的框架得到 future 所擷取之物的副本，而對啟動端而言，future 及其所擷取之物都已消失，與送出之後一樣（[2.18.1](218-task.md#2181-核心-api)）。

  以下也是編譯錯誤：交出從 `Sync` 物件讀出、但本身不是 `Sync` 的值（使用者 `unsafe impl Sync` 型別的欄位），因為其他程式碼不經交接就能到達它（請改為交出 `clone()`）；`thread::local` 的值，它被借給 `with` 閉包（3.4.4）；通道的 `recv`、任務的結果或鎖的 guard 所給出的值屬於接收端，不在此規則的範圍內；在 guard 仍持有時交出 guard 的 `get` 所回傳之物（直到 `set` 為鎖賦予新值為止），或把它傳給會保留它的被呼叫端（要保留的副本是 `get_clone()`）；在 `using` 區塊內回傳或擲回它、在那裡 yield 它（產生器中的 `yield g.get()`）、讓它成為區塊本身的值（`let x = using g = m.lock() { g.get() }`、`f(using g = m.lock() { g.get() })`），無論是以何種形式帶出（迴圈的 `break` 值、帶標籤的區塊、`if` / `match` 分支、`catch`、閉包的結果），或把它交給在區塊內建立、在區塊之後執行的協程（`async fn` 的 future、產生器：`fut = grow(g.get()[0])`）：區塊的結束會在該值被使用之前釋放鎖。guard 只在其 `using` 區塊內有效：在該區塊結束之後使用它的副本（被推入集合、被閉包擷取、被被呼叫端存起來、被第二個 `using` 繫結、被協程持有：`async fn` 或產生器的引數、`async` 區塊的擷取）是編譯錯誤；guard 是 `!Sync`（[2.16.3](216-sync.md#2163-mutext互斥鎖)）。`par::for_each`、`par::map` 和 `par::filter` 把每個項目交給一個工作者，因此兩個項目不得到達同一個非 `Sync` 的集合。每次呼叫都會在執行時檢查這一點（[2.19.1](219-par.md#2191-核心-api)）：如果兩個項目到達同一個集合，這些項目會在呼叫端執行緒上逐一執行，並且每個呼叫位置發出一次警告。切片只會讀取，因此項目可以透過串列的切片共用該串列，只要沒有項目持有串列本身；可寫入的檢視（`SliceMut`）在沒有其他項目的檢視與其中一個所寫入的部分重疊時，也可以共用它。

  不在這些規則範圍內的有：`unsafe` 內的程式碼；透過函式值或閉包的呼叫，它會跳過上述的引數檢查；以及多次 yield 同一個物件、被收集到集合中、之後再把該集合的元素逐一交出的產生器。這種集合不會被視為持有該物件兩次。

  在執行時，每次交接也都會複製，因此在上述規則漏掉的途徑上（以及 `unsafe` 內的程式碼中），被交出的值以值方式到達的任何集合，都不會最終由傳送端與新擁有者共用（例外列於下文）。`Mutex::new`、`RwLock::new`、guard 的 `set` 和 `update`、`send` / `try_send` / `send_async`、`task::spawn_blocking_with_arg` 的值、`[move x]` 擷取（在建立閉包時），以及任務、執行緒或 `par` 作用域工作的結果（還有 `try_scope` 的錯誤），會給新擁有者一份該值所到達之每個集合的副本：經由結構、元組、陣列、列舉和和型別的各部分，以及集合的元素，深度不限，使用工作清單而非每層一個堆疊框架。被到達多次的集合只複製一次，因此副本保留了值的共用關係與環狀參考。被複製的有 `List`、`Dict`、`Deque`、`StrBuf`、`Slice` 以及建立在它們之上的集合型別，還有 `GcPtr`（[1.6.3](../100-bare/106-gc-pin.md)）。協程 future（`AsyncFuture`、`TryAsyncFuture`）會得到自己的框架，其中持有在同一次複製中做出的、它所擷取之物的副本；已經開始或已經完成的 future，或其框架（或所擷取之 future 的框架）含有 `[ref x]` 擷取的 future，不能交出，交接時會觸發 panic（[2.18.1](218-task.md#2181-核心-api)）。`Task` 不會被複製，但一旦交出，它的每次 `await` 都會得到結果的獨立副本（[2.18.2](218-task.md#2182-taskt-型別)）。串列與其緩衝區的每個檢視繼續共用同一個被複製的緩衝區；`str` 位元組的檢視只複製那些位元組（它們不會改變）。`update` 保留 guard 已經持有的東西：只複製閉包帶進來的部分。傳送端的控制代碼維持原樣：上述規則遺漏的控制代碼，指向的是沒有其他執行緒使用的物件。擲回錯誤的送出會帶回一個相等的值（呼叫端的值或其副本）。複製所花的時間與記憶體與複製的量成正比；一段連續的單純值（沒有 GC 參考）只是一次記憶體複製。工具鏈內部的 `core::marker::hand_over(x)` 回傳相同的副本。`xs.freeze()` 不會複製：凍結後的集合接管儲存空間並讓 `xs` 變空，而帶有檢視的串列會給出副本（[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s唯讀集合型別)）。保持共用的有：`Sync` 部分（包括凍結的集合型別）、具有 `unsafe impl Send` 之型別的值（它們自行管理共用：任務、檔案、通訊端），以及產生器和串流（不是 `Send`：它們永遠不會被交出）。`Mutex::new_unshared`（2.16.3）不會複製。從 guard 取出的控制代碼（`get`、`update` 的閉包引數）在執行時不會被複製：上述規則會追蹤它以及 guard 本身，經過每個副本、`return`、`yield`、區塊的值以及在區塊內建立之協程的框架，一直追到 `using` 區塊之外。
- 以下排除在外：`Cell<T>`、原始指標（`Ptr<T>`、`ConstPtr<T>`、`CPtr<T>`）、單獨的 `any` 以及約束缺少 `Send` / `Sync` 的 `any Tr`（只有當約束同時具有 `Send` 和 `Sync` 時，`any Tr` 才是 `Send`：`any` 的副本共用其 box，就像閉包的一般擷取被共用一樣，因此跨執行緒的 `any` 之中不得有任何集合）、沒有約束的函式型別，以及帶有 `[ref]` 擷取的閉包（[refs-lang 4.6.9](../../refs-lang/004-functions/406-capture-semantics.md#469-additional-constraints-in-parallel-and-concurrent-contexts)）。其他例外規定在各型別的章節中。
- 函式型別只有在寫出約束時才具有這些 trait，例如 `fn(..) -> R + Send`（[refs-lang 4.4](../../refs-lang/004-functions/404-function-type.md#send--sync-bounds)）。單獨的 `+ Send` 不會讓它成為 `Sync`：其背後的閉包可能擁有非 `Sync` 的 `[move x]` 擷取。由多個執行緒呼叫的函式值（`static`、`par::*` 閉包）是 `+ Send + Sync`。

自動實作不正確的型別會直接宣告。內部持有原始指標、但以執行緒安全方式包裝它的型別，以 `unsafe impl` 選擇性加入（安全性由作者負責）；在結構上會被自動實作、但並非執行緒安全的型別，以 `impl !Send` 選擇退出。`Cell<T>` 的排除就是以這種方式宣告的。

```nabi
struct Handle { raw: Ptr<u8>; }
unsafe impl Send for Handle { }
unsafe impl Sync for Handle { }

impl<T> !Send for Cell<T> { }
impl<T> !Sync for Cell<T> { }
impl<T> !Send for Ptr<T> { }
impl<T> !Sync for Ptr<T> { }
impl<T> !Send for CPtr<T> { }
impl<T> !Sync for CPtr<T> { }
```

把值傳給執行緒或任務的 API（`thread::spawn`、`task::spawn`、`par::*`、`channel`）以這兩個 trait 約束其引數。同步原語的政策依照 [2.16 sync](216-sync.md)。

### 交接機制

```nabi
#[internal]
pub fn hand_over<T>(x: T) -> T;
```

`hand_over(x)` 回傳交接時給新擁有者的東西：附帶它所到達之每個集合副本的 `x`，或在它不到達任何集合時回傳 `x` 本身。它是 `#[internal]`：由 `core` 和 `std` 呼叫，從其他任何套件使用都是錯誤（[0.4.4](../000-meta/004-lang-slot-table.md)）。

```nabi
pub struct Handoff<T> { /* private fields */ }
impl<T> Handoff<T> {
    #[internal]
    pub unsafe fn new(value: T) -> Handoff<T>;
    pub fn get(self) -> T;
}
```

`Handoff<T>` 在呼叫端自己的協定（join、鎖、通道交接）下於執行緒之間傳遞值，該協定保證不會有兩個執行緒同時使用它：只要 `T` 是 `Send`，它就是 `Send + Sync`，即使 `T` 不是 `Sync`。`new` 是 `unsafe` 且 `#[internal]`；`get` 回傳該值。其他程式碼則把值移入閉包（`[move x]`），或傳給 `task::spawn_blocking_with_arg`。

`Transfer` 和 `Memo` 是 `pub(pkg)`：交接複製透過它們到達 `core` 其他模組中的標準集合型別，其他套件看不到它們。

```nabi
pub(pkg) trait Transfer {
    fn transfer_copy(self, m: Memo) -> Self;
    fn transfer_keep(self, m: Memo) -> Self;
    fn transfer_visit(self, m: Memo) -> Self;
}

pub(pkg) struct Memo { /* private fields */ }
```

編譯器只為其固定清單中的標準控制代碼型別呼叫 `Transfer`；其他任何型別上的 impl 都會被忽略。除了集合型別與 future 之外，這份清單還包含 `crypto::Secret`（[2.21](221-crypto.md)）以及密碼學演算法所保存的秘密字組：交接會在秘密儲存區中給新擁有者一份自己的副本，而 `par` 會像對集合一樣，逐一執行到達同一個 `Secret` 的項目。`Memo` 依原始物件記錄一次交接所做出的副本，以及仍待修正的被複製緩衝區。它的方法以及旁邊的 `transfer*` 內建函式也都是 `pub(pkg)`。

## 2.13.10. `Default`：預設值

```nabi
trait Default {
    fn default() -> Self;
}
```

| 型別                                 | `default()`        |
|--------------------------------------|--------------------|
| 整數、`int`                      | `0`                |
| 浮點數、`float`              | `0.0`              |
| `bool`                               | `false`            |
| `char`                               | `'\0'`             |
| `str`                                | `""`               |
| `List`、`Dict`、`Set` 等集合型別 | 空集合 |
| `T \| none`                          | `none`             |

`Default` 可以衍生；它以每個欄位宣告的預設值建立值（[refs-lang 2.3.3](../../refs-lang/002-types/203-struct.md#233-field-default-values)），沒有預設值的欄位則用欄位型別的 `default()`：`#[derive(Default)] struct Cfg { host: str = "localhost"; port: i32; }` 得到 `Cfg { host: "localhost", port: 0 }`。

## 2.13.11. 可衍生的 trait

`#[derive(...)]`（[refs-lang 6.2.1](../../refs-lang/006-meta-system/602-built-in-attributes.md#621-derive-automatic-trait-implementation)）能產生的內建 trait 只有這些：

| Trait         | 產生的實作                      | 需求                            |
|---------------|-----------------------------------------------|----------------------------------------|
| `Clone`       | 逐欄位 `clone`                             | 每個欄位都是 `Clone`                    |
| `Copy`        | 標記                                        | 同時衍生 / 實作 `Clone`，且不是 `Drop` |
| `Debug`       | 以 `f"{x:?}"` 形式呈現欄位名稱與值     | 每個欄位都是 `Debug`                    |
| `Default`     | 逐欄位：宣告的預設值，否則為 `default()` | 每個沒有預設值的欄位都是 `Default` |
| `PartialEq`   | 逐欄位 `==` 的 AND                         | 每個欄位都是 `PartialEq`                |
| `Eq`          | 標記                                        | 同時衍生 / 實作 `PartialEq`   |
| `PartialOrd`  | 依欄位宣告順序的字典序比較 | 每個欄位都是 `PartialOrd`、`PartialEq` |
| `Ord`         | 依欄位宣告順序的字典序比較 | 同時衍生 / 實作 `Eq`、`PartialOrd` |
| `Hash`        | 依宣告順序對每個欄位 `hash`；列舉先寫入其變體索引（寫入內容未規定） | 每個欄位都是 `Hash` |
| `Error`       | 帶有預設 `source()` 的 `Error`（[2.14](214-error.md#deriveerror)） | 已實作或衍生 `Display` 和 `Debug` |

- `Display` 不能衍生；`#[display("..")]` 以 f-string 寫出它（[2.10.14](210-fmt.md#21014-衍生政策摘要)）。
- 衍生 `Eq` / `Ord` 而沒有其父 trait 是編譯錯誤。
- 對 `union` 套用 `#[derive]` 是編譯錯誤。
