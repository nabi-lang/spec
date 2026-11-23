# 0.1. 三層模型

Nabi 標準函式庫分為三個層級。每個層級可以相依的範圍不同，
使用者程式碼中的匯入規則也不同。

```
bare   (base, depends only on the runtime ABI extern "nabi")
  ↓
core     (foundation, safe, all targets; uses bare internally)
  ↓
std   (target-dep; uses foundation + bare)
```

每個層級各自是一個路徑根：`bare::..`、`core::..`、`std::..`（[refs-lang 5.7.4](../../refs-lang/005-modules/507-paths.md#574-stdcorebare標準函式庫層級)）。

分成三層的理由很明確。可能破壞記憶體安全的原始操作，限制在狹小範圍內才容易
稽核；必須在每個平台上行為一致的核心抽象，只有不與依賴作業系統的程式碼混在一起，
才能保證可攜性。可用性因平台而異的功能，必須在簽章與匯入處呈現這一點，
讓使用者能一致地處理 wasm 建置與原生建置。三個層級各自負責這三項要求中的
一項。

## 0.1.1. 各層級的職責

### `bare`

最底層的 `bare` 只相依於執行環境的 `extern "nabi"` 符號。
它公開 `Ptr<T>`、`ConstPtr<T>`、`transmute`、原始配置、pin/unpin 與
volatile 等原始工具，因此使用它們一律需要明確的 `use`。明確本身就是危險訊號：
這是刻意的設計，讓危險的範圍在程式碼審查時一眼可見。

```nabi
use bare::ptr::CPtr;
use bare::alloc::{alloc, Layout};
let layout = try Layout::array::<u8>(1024);
let p: CPtr<u8> = unsafe { alloc(layout) };    // null on failure (see 1.4 alloc)
```

### `core`（基礎）

中間的基礎層級 `core` 收錄 `core::collections`、`core::iter`、`core::ops` 等模組，只在內部相依於 `bare`。日常使用語言所需的幾乎所有抽象（數值型別、算術 trait、
iter、集合型別、fmt、轉換、sync、task、future、位元組串流 I/O 等）都集中在這一層。在使用者程式碼中，
**其 prelude `use core::prelude::*;` 會自動隱含套用**（詳細政策請見 [0.2 匯入政策](002-import.md)），
因此不需要任何匯入就能直接使用。

```nabi
let xs: List<i32> = [1, 2, 3];        // List used directly, brought in by the prelude
let r = a + b;                         // Add trait automatic
let m = Mutex::new(0);                 // sync::Mutex automatic too
task::spawn(async { /* ... */ });              // task module automatic too
```

prelude 是一份固定的名稱清單（[0.2.3](002-import.md)）；`core` 的其餘部分則
透過 prelude 模組指名（`task::sleep`、`ops::Add`）或匯入。

### `std`

最上層的 `std` 視需要同時使用基礎層與 `bare`。這一層收錄
依賴平台的項目，例如 `fs`、`net`、`time`（牆上時鐘與開機時鐘）、`thread`（作業系統
執行緒）、`env`、`process`、`path` 與 `random`。由於可用性因目標而異，使用者必須一律明確匯入，清楚表明
自己踏入了哪些平台功能。

```nabi
use std::fs;
use std::time::SystemTime;
```

## 0.1.2. 相依規則（強制）

允許下列相依。

| 相依方      | 可以相依於                                                |
|-------------|-----------------------------------------------------------|
| `bare`  | 執行環境 ABI `extern "nabi"`                              |
| `core`  | `bare`                                              |
| `std`  | `core`、`bare`                                  |
| 使用者程式碼 | `core`（隱含）、`std`（明確）、`bare`（明確）     |

下列方向的相依是**禁止**的。基礎層（`core`）不得
匯入 `std`：基礎層必須能在每個目標上建置，一旦引入依賴平台的程式碼，
可攜性就會瓦解。`bare` 也不得匯入基礎層或 `std`：
低階層級一旦開始相依於高階層級，分層本身就失去
意義。最後，個別 `std::*` 模組之間禁止環狀相依，讓
每個 std 模組都保持為可以獨立建置的單元。

這條規則規範的是一個層級所寫的內容：它的 `use` 與路徑。每個層級內部的 `use core::**` 仍會提供 bare 自己的簽章與 impl 所用的語言詞彙（1.6、1.7）：0.4 的槽位項目、`List`、`Slice`、`Result`、`panic`，以及 bare 的型別所實作的 trait：`Debug` / `Formatter` / `FmtError`、`Hash` / `Hasher`、`Error`、`AsBytes`，還有 `Utf8Error` 與 `str::from_utf8`。

prelude 是 `core` 唯一指名 `std` 項目的地方：在對應的目標上，`core::prelude` 會列出 `std::os` 的 Ext trait（0.2.3），因此使用其方法不需要 `use`。這是隱含匯入（0.2），而不是 core 自身程式碼的相依；core 自身的程式碼仍然不需要 `std` 就能建置。

## 0.1.3. 建置單元

三個層級各自作為獨立的建置單元（獨立的封存檔）處理，並可在每次編譯時快取。
連結順序為 `std` → `core` → `bare` → 執行環境，使用者套件位於
所有層級之上。編譯器在首次使用時將它們建置到快取並自動連結，因此使用者
不需要把它們加入相依宣告（請見 [0.5.4 發行形式與連結機制](../../refs-lang/000-meta/005-standard-libraries-tiers.md#054-散布形式與連結機制)）。

## 0.1.4. 目標可用性

`bare` 與基礎層（`core`）可在每個[支援的目標](../../refs-lang/000-meta/006-platform-targets.md)上建置，
包括 wasm32。相對地，`std` 的可用性因目標而異。例如，
`std::thread` 無法在 `wasm32-wasip2` 或瀏覽器中啟動執行緒，因為兩者都只有一個
執行緒（`wasm32-wasip1-threads` 有執行緒）。`std::fs` 同樣無法在 wasm32-unknown-unknown 上使用，只在
WASI 目標（`wasm32-wasip2`、`wasm32-wasip1-threads`）上提供。這類可用性記載於各 `std::*` 模組規格的可用性一節。
API 仍可在每個目標上建置：目標所缺少的操作會在執行時以
`IoErrorKind::Unsupported` 失敗。在瀏覽器中，stdout / stderr 會送到主機主控台，stdin 處於 EOF。
例外是屬於單一作業系統家族的功能：它們位於 `std::os::<family>`（`unix`、`linux`、
`windows`），只存在於該家族的目標上，因此在其他地方使用會產生編譯期錯誤，
並指出所需的 `#[cfg]`（[3.10](../300-std/310-os.md)）。

## 0.1.5. 演進政策

三個層級的穩定性政策也不同。`bare` API 並不穩定，
會與編譯器及執行環境一起演進，因此建議使用者程式碼固定版本。
基礎層（`core`）的 API 以 SemVer 穩定性為目標，不相容的變更要等到破壞性
版本（1.0 之前為 `0.x.0`）。`std` API 隨各目標可用性的差異而變動，但
簽章本身保持穩定。

在同一個發行系列內，項目不會被移除，也不會以不相容的方式變更。被取代的項目
仍可繼續運作，並標記為 `#[deprecated(since = "..", note = "..")]`
（[6.2.7](../../refs-lang/006-meta-system/602-built-in-attributes.md#627-deprecated-不建議使用的標記)），
其 note 指出應改用什麼。必須改為 `unsafe` 的安全函式會標記為
`#[deprecated_safe]`（[6.2.13](../../refs-lang/006-meta-system/602-built-in-attributes.md#6213-deprecated_safe-現在安全之後-unsafe)）。
下一個破壞性版本會移除該項目，或將其改為 `unsafe`。

## 0.1.6. 每個項目只有一條路徑

每個項目都只有一條路徑。模組像命名空間一樣分成公開與私有子模組：
公開子模組是使用者會指名的命名空間（`core::num::niche`、
`core::crypto::ed25519`、`std::net::tls`、`std::os::unix::fs`），而
只存放父模組部分程式碼的子模組則是私有的（`core::io` 中的 `mod buffer;`），
由父模組以 `pub use` 重新匯出其項目。當途經的每個模組都是公開時，項目的路徑就是
其定義模組的路徑；否則就是重新匯出它的父模組的路徑：
`BufReader` 是 `core::io::BufReader`，`TcpStream` 是 `std::net::TcpStream`，`sleep` 是
`core::task::sleep`。同一項目的第二條路徑只以兩種形式存在：父模組
攤平其公開子模組（`core::collections::List` 對應
`core::collections::list::List`、`core::crypto::Secret`、`bare` 根部的強制轉型函式），
以及隱含匯入（0.2），它也會在對應的目標上指名 `std::os` 的 Ext trait
（0.1.2）。沒有任何模組會重新匯出其父模組、兄弟模組或其他層級的項目：`Duration` 就是
`core::time::Duration`，別無其他；`SeekFrom` 是 `core::io::SeekFrom`；`size_of`
是 `bare::mem::size_of`。文件與診斷顯示項目的路徑時，使用的是
最短的公開路徑，絕不經過私有模組（`std::net::WriteHalf`）。標準函式庫自身的
測試會拒絕其他任何 `pub use`。
