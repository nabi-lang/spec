# 0.2. 匯入政策：prelude `core::prelude`

> 本文件定義標準函式庫的隱含匯入政策。每個使用者檔案不需要 `use` 就能看到單一模組 `core::prelude` 的名稱；`core` 中的其他一切，以及 `bare` 與 `std` 的全部內容，都以路徑指名或明確匯入。

Nabi 會在每個使用者原始檔的開頭**隱含**加上下面這一行。

```nabi
use core::prelude::*;
```

`core::prelude` 是由 `pub use` 重新匯出組成的普通模組（[refs-lang 5.8](../../refs-lang/005-modules/508-re-export.md)）。它的清單固定如下（0.2.3），且具有規範性：只有本文件列出的名稱才屬於 prelude。

prelude 不包含任何列舉變體。單獨的變體名稱會依據該位置預期的型別解析（[refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-依預期型別使用不帶路徑的變體)）：`let o: Ordering = Less;`、`x.load(SeqCst)`、`return Ok(v);`。

## 0.2.1. 核心設計決策

| 模型                                          | 語言                  | Nabi 是否採用  |
|-----------------------------------------------|-----------------------|----------------|
| 只用明確的 use                                | C / C++               | （冗長）       |
| 核心函式庫的所有公開名稱                      | Swift、Python builtins | 不採用        |
| **簡短、精選的 prelude 模組**                 | Rust、Gleam、Elm      | **採用**       |

### 為什麼採用精選清單

若對整個 `core` 隱含使用 glob，每個公開的 core 項目都會在每個檔案中成為單獨的名稱：大約 160 個型別、100 個 trait、90 個函式，以及所有列舉變體。其中有幾個會互相衝突（`iter` 與 `stream` 中的 `Map`、`Filter`、`Zip`；`task` 與 `par` 中的 `join`；`Ordering` 與 `fmt::Alignment` 中的變體 `Equal`），而且每新增一個 core 項目，都會改變使用者程式碼的名稱解析。

prelude 保留的是幾乎每個檔案都會寫到的東西：集合型別與核心型別、寫在 `impl` 標頭與約束中的 trait、輸出與 panic 函式，以及 core 的頂層模組，讓 `task::sleep` 與 `par::map` 保持簡短。其中每個名稱在其命名空間中都是唯一的（函式庫測試會在每個目標上檢查這一點），因此單獨的 prelude 名稱永遠不會有歧義。由於隱含匯入的優先順序最低（0.2.4），在 prelude 中加入名稱永遠不會破壞定義或匯入同名項目的程式。

## 0.2.2. `**` 遞迴 glob

```nabi
use foo::bar;            // single item
use foo::{a, b};         // multiple
use foo::*;              // single-level glob (only foo's direct children)
use foo::**;             // recursive glob (foo + all submodules)
```

`**` 語法的規範性定義請見 [refs-lang 5.5 use](../../refs-lang/005-modules/505-use.md)。

## 0.2.3. prelude 清單

下列名稱恰好就是 `core::prelude` 的名稱。與 Nabi 的其他地方一樣，模組、型別與 trait 共用一個命名空間，函式則使用另一個，因此 `channel`、`panic` 與 `print` 都同時是模組與函式。

**模組（29 個）**。`core` 的頂層模組，但不含 `prelude` 本身與 `hash`（以 `use` 匯入，[2.5.11](../200-core/205-cmp.md)）：

`array` `bench` `channel` `cmp` `collections` `convert` `crypto` `encoding` `error` `fmt` `future` `hint` `index` `io` `iter` `marker` `num` `ops` `panic` `par` `print` `range` `reflect` `str` `stream` `sync` `task` `time` `tuple`

**型別（36 個）**。

| 分組 | 名稱 |
|---|---|
| 集合型別 | `List` `Dict` `Set` `Deque` `BinaryHeap` `Slice` `SliceMut` `StrBuf` `FrozenList` `FrozenDict` `FrozenSet` |
| 數值與範圍 | `Range` `RangeInclusive` `Int` `Float` |
| 一般 | `Result` `Ordering` `Duration` `Instant` |
| 同步與通道 | `Mutex` `RwLock` `Atomic` `Cell` `Sender` `Receiver` `Task` |
| 格式化 | `Formatter` `FmtError` |
| 核心錯誤 | `ParseError` `ValueError` `OverflowError` `ConversionError` `TypeError` `StackOverflowError` `OutOfMemoryError` `InternalError` |

**Trait（31 個）**。

| 分組 | 名稱 |
|---|---|
| 標記 | `Clone` `Copy` `Drop` `Default` `Send` `Sync` |
| 比較 | `PartialEq` `Eq` `PartialOrd` `Ord` `Hash` |
| 格式化與錯誤 | `Debug` `Display` `RichDisplay` `Error` |
| 迭代與非同步 | `Iterator` `IntoIterator` `FromIterator` `TryIterator` `Future` `IntoFuture` |
| I/O | `Read` `Write` `AsyncRead` `AsyncWrite` |
| 轉換 | `From` `Into` `TryFrom` `TryInto` `FromStr` |
| 反射 | `Inspect`（也包括其 derive，[refs-lang 6.8.11](../../refs-lang/006-meta-system/608-meta-blocks.md)） |

運算子 trait 不在 prelude 中：要寫 `ops::Add`，或 `use core::ops::Add;`。運算子本身不需要名稱，在任何地方都能運作（[refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-對應的可見性)），每個 `core` trait 的方法也一樣（[refs-lang 3.5](../../refs-lang/003-expressions/305-member-access-and-method-call.md)）。

**作業系統擴充 trait**。在具備這些 trait 的目標上，prelude 也會指名 `PermissionsExt`、`CommandExt` 與 `ExitStatusExt`（unix），以及 `ChildExt` 與 `TcpStreamExt`（linux），因此在這些目標上使用其方法不需要 `use`（[3.10.1](../300-std/310-os.md#3101-prelude-中的-ext-trait)）。它們不計入上面的數量。

**函式（13 個）**。

`print` `println` `eprint` `eprintln` `dbg` `display` `panic` `assert` `assert_eq` `assert_ne` `unreachable` `channel` `channel_bounded`

**變體**。無。請見 [refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-依預期型別使用不帶路徑的變體)。

```nabi
// user code
fn main() {
    let xs: List<i32> = [1, 2, 3];        // prelude: List
    let total: i32 = xs.iter().sum();      // methods of core traits: no import
    println(f"{total}");                   // prelude: println

    let m = Mutex::new(0);                 // prelude: Mutex
    task::spawn(async { /* ... */ });      // prelude module task
    task::sleep(Duration::ms(10));         // prelude: Duration

    let n = Atomic::new(0);
    n.store(1, SeqCst);                    // variant from the parameter type
}
```

`core` 中的其他一切都透過 prelude 模組指名或匯入：

```nabi
let p: future::Poll<i32> = future::Poll::Pending;
impl ops::Add for V { ... }
use core::sync::LoadOrdering;
let h = encoding::hex::encode("nabi");
```

不在作用域內的名稱會產生錯誤，並指出應該寫什麼：可行的最短路徑（`task::sleep`、`ops::Add`、`Ordering::Less`），以及匯入它的 `use`。

## 0.2.4. 名稱衝突政策

隱含匯入的優先順序最低：解析名稱時會先取模組自身的項目及其 `use`（[refs-lang 5.7.1](../../refs-lang/005-modules/507-paths.md#571-路徑的起始識別字)）。與 prelude 名稱同名的使用者項目會在該模組中遮蔽它，而 core 的項目仍可透過路徑取得。

```nabi
struct List<T> { ... }                  // OK: shadows the prelude's List here
let xs: core::collections::List<i32> = [1, 2];   // the core List, by path
```

`use` 路徑可以用 prelude 名稱開頭：`use task::sleep;` 會匯入 `core::task::sleep`。

## 0.2.5. **不**隱含套用的層級

只有 prelude 是隱含的。`bare` 與 `std` **只能明確匯入**。

```nabi
use bare::ffi::{CStr, CString};      // unsafe: danger signal
use bare::ptr::{Ptr, ConstPtr};
use std::fs;                         // std: target-dependency signal
use std::time::SystemTime;
```

此政策與 [0.1 層級模型](001-tier-model.md)的下表一致。

| 層級            | 隱含匯入                | 理由                            |
|-----------------|-------------------------|---------------------------------|
| `core`     | ✓ prelude（0.2.3）      | 所有目標／安全                  |
| `bare`     | ✗                       | 危險訊號，只能明確匯入          |
| `std`      | ✗                       | 相依於目標，只能明確匯入          |

如此一來，從檔案開頭就能立刻看出使用者踏入了哪些範圍。

## 0.2.6. 停用：`#![no_implicit_nabi]`

在特殊情況下，可以關閉隱含匯入本身。

```nabi
#![no_implicit_nabi]   // top of the file

// no implicit import in this file
use core::prelude::*;   // the same import, written out
```

字面值由編譯器定義，仍然可以使用；運算子與 core trait 的方法也一樣（[refs-lang 6.7.8](../../refs-lang/006-meta-system/607-operator-magic.md#678-對應的可見性)）；失去的只有單獨的名稱。

這在教學或示範程式碼中很有用，可以一眼看出每個項目的來源。

標準函式庫自身的模組不匯入 prelude：每個層級都以葉名稱看到 `core` 的每個公開項目（內部的 `use core::**;`），使用者程式碼則沒有這項待遇。

## 0.2.7. 可見性

prelude 只重新匯出 `pub` 項目。內部實作（`pub(pkg)` / 私有）絕不會洩漏給使用者：`core::collections::raw` 是 `pub(pkg)`，因此 `GcBuf` 與 `buf_*` 函式只能透過明確的 `use`，以 `bare::mem::..` 取得。

名稱不會在清單之外被攤平；模組路徑保持原樣。

```nabi
let m = Mutex::new(0);              // prelude: Mutex
let s = task::spawn(async { /* */ });      // prelude module task; spawn is task::spawn

// both mean the same thing.
core::sync::Mutex::new(0);
Mutex::new(0);
```

## 0.2.8. 與其他語言的比較

| 語言     | 隱含作用域                                         | 變體                                     |
|----------|----------------------------------------------------|------------------------------------------|
| Rust     | `std::prelude::rust_20xx`，明確的清單              | 只有 `Some` `None` `Ok` `Err`            |
| Go       | universe 區塊：約 20 個內建項目與型別              | 需加限定（`time.Second`）                |
| Swift    | `Swift` 的所有公開宣告                             | 依上下文型別寫 `.less`                   |
| Kotlin   | 預設的套件匯入                                     | 依上下文解析（2.2）                      |
| **Nabi**  | `core::prelude`，明確的清單（0.2.3）          | 單獨名稱，依預期型別解析（任何列舉）     |
