# Nabi 標準函式庫參考

本文件是 Nabi 標準函式庫的正式規格。語言規格定義語法與語意機制，標準函式庫規格則提供這些機制運作所需的型別、trait 與函式的意義。運算子解析為哪個 trait、集合型別有哪些介面、非同步與 I/O 依據哪些保證運作，都在這裡統一定義。

語言規格另外收錄在 [`refs-lang/`](../refs-lang/README.md)；本文件只涵蓋規格。函式庫的實際實作不在規格範圍內。

> [!IMPORTANT]
> **語言規格與標準函式庫規格的界線**
>
> [refs-lang](../refs-lang/README.md) 定義**語法與語意機制**（例如 `+` 語彙單元對應到哪個 trait
> 方法）。[refs-standard-libraries](README.md)（本文件）定義
> **這些 trait 本身的簽章與意義**（例如 `Add::add` 套用哪種數值
> 轉換）。
>
> 兩份規格遵循相同的[表示法慣例](../refs-lang/000-meta/008-notation.md)。

---

## 三層模型

標準函式庫不是單一的扁平平面，而是分成三層。幾乎所有系統程式碼都寫在安全的抽象之上，但要實作這些抽象，終究得在某處處理原始記憶體與 FFI。此外，檔案或網路這類只有在主機環境下才有意義的功能，在嵌入式這類沒有該環境的目標上必須完全不存在。為了把這些差異直接呈現在相依關係圖中，函式庫採用三層結構，每一層都是一個路徑根：`bare` / `core` / `std`。

```
bare   (base, depends only on the runtime ABI extern "nabi")
  ↓
core   (foundation, safe, all targets; uses bare internally)
  ↓
std   (target-dep; uses core + bare)
```

`bare` 位於最底層，只公開原始能力；`core` 在其上提供安全的基礎，在每個目標上都相同。`std` 在 `core` 之上加入相依於主機的功能，補齊一般應用程式預期的介面。日常程式碼只需要 `core`；需要作業系統資源時才用到 `std`，只有在必須直接跨越抽象界線時才明確匯入 `bare`。詳細的相依規則請見 [0.1 層級模型](000-meta/001-tier-model.md)。

| 層級        | 路徑           | 目錄                               | 範圍                                           |
|-------------|----------------|------------------------------------|------------------------------------------------|
| `bare` | `bare::*` | [`100-bare/`](100-bare/README.md)  | `Ptr`、`ConstPtr`、`reinterpret`、原始配置、固定、FFI |
| `core` | `core::*` | [`200-core/`](200-core/README.md)  | 數值、運算子、iter、集合型別、fmt、sync、task、io、crypto |
| `std` | `std::*` | [`300-std/`](300-std/README.md)    | fs、net、time、thread、env、process、path、nabion、random、os |

---

## 章節

### 0. Meta：[000-meta/](000-meta/README.md)

適用於整個函式庫的政策與表格。

- [0.1. 三層模型](000-meta/001-tier-model.md)
- [0.2. 匯入政策（prelude `core::prelude`）](000-meta/002-import.md)
- [0.3. 內建函式機制：編譯器 ↔ 標準函式庫的界線](000-meta/003-intrinsics.md)
- [0.4. `#[lang]` 槽位對應表](000-meta/004-lang-slot-table.md)
- [0.5. 表示法](000-meta/005-notation.md)

### 1. Bare：[100-bare/](100-bare/README.md)

低階／危險的範圍。必須明確匯入。

- 1.1. [指標家族（`Ptr`/`ConstPtr`/`ManyPtr`/`CPtr`：Zig 的 arity 編碼 + Rust 的 strict provenance）](100-bare/101-ptr.md)
- 1.2. [強制轉型操作（`ptr_cast`/`align_cast`/`const_cast`/`reinterpret`）](100-bare/102-cast.md)
- 1.3. [`MaybeUninit<T>` + `Pod` + 原始記憶體操作](100-bare/103-mem.md)
- 1.4. [原始配置](100-bare/104-alloc.md)
- 1.5. [揮發性記憶體存取](100-bare/105-volatile.md)
- 1.6. [`Pinned<T>`：GC 固定（Go 1.21 Pinner 模型）](100-bare/106-gc-pin.md)
- 1.7. [FFI：`extern "C"` / `CStr` / `CString`](100-bare/107-ffi.md)
- 1.8. [WASM Component Model（`ffi::wit`）](100-bare/108-wit.md)

### 2. Core：[200-core/](200-core/README.md)

安全的範圍。適用所有目標。其 prelude `use core::prelude::*;` 會隱含套用。

- 2.1. [數值型別（固定寬度、`Int`、`Float`）](200-core/201-numbers.md)
- 2.2. [Niche 包裝型別（`NonZero`、`NonNan`、`Finite`、`Positive`）](200-core/202-non-zero.md)
- 2.3. [算術／位元運算 trait（`Add`、`Sub`、...、`Shl`、`Shr`）](200-core/203-ops.md)
- 2.4. [複合指派 trait（`AddAssign`、...）](200-core/204-ops-assign.md)
- 2.5. [比較 trait（`PartialEq`、`Eq`、`Ord`、`PartialOrd`、`Hash`）](200-core/205-cmp.md)
- 2.6. [廣播 trait（`DotAdd`、`DotMul`、...）](200-core/206-broadcast.md)
- 2.7. [Iter trait（`Iterator`、`IntoIterator`）](200-core/207-iter.md)
- 2.8. [`Stream`（非同步迭代器）](200-core/208-stream.md)
- 2.9. [集合型別（`List`、`Dict`、`Set`、`Slice`、`Deque`、`BinaryHeap`、`SortedDict` / `SortedSet`、陣列、元組）](200-core/209-collections.md)
- 2.10. [fmt（`Display`、`Debug`、`Formatter`）](200-core/210-fmt.md)
- 2.11. [轉換（`From`、`Into`、`TryFrom`、`saturating_into`、...）](200-core/211-conversion.md)
- 2.12. [反射（`Reflect`、`Inspect`）](200-core/212-reflect.md)
- 2.13. [標記 trait（`Clone`、`Copy`、`Drop`）](200-core/213-marker.md)
- 2.14. [錯誤型別（`TypeError`、`ValueError`、...）](200-core/214-error.md)
- 2.15. [字串（`str`、`StrBuf`、`char`、`c_char`、Unicode 正規化與分段）](200-core/215-string.md)
- 2.16. [同步基本元件（`Atomic`、`Mutex`、`RwLock`、`Cell`）](200-core/216-sync.md)
- 2.17. [`Channel`](200-core/217-channel.md)
- 2.18. [`task::` API](200-core/218-task.md)
- 2.19. [`par::` API](200-core/219-par.md)
- 2.20. [`Future` / `await` 整合](200-core/220-future.md)
- 2.21. [`crypto::`：雜湊、MAC、AEAD、金鑰交換、簽章](200-core/221-crypto.md)
- 2.22. [`encoding::`：十六進位與 base64](200-core/222-encoding.md)
- 2.23. [`io::`：位元組串流與標準 I/O（`Read`、`BufRead`、`Write`、`Seek`、非同步對應版本）](200-core/223-io.md)
- 2.24. [`simd::`：可攜式 SIMD 向量（`Simd`、`Mask`、`f32x4`、...）](200-core/224-simd.md)

> [!NOTE]
> 張量、自動微分、GPU 等科學計算／AI 領域不屬於標準函式庫，而是拆分為外部函式庫。外部函式庫透過後端介面接觸編譯器的後端，該介面預計在之後的版本推出。

### 3. Std：[300-std/](300-std/README.md)

相依於目標的範圍。必須明確匯入。

- 3.1. [`fs::`：檔案系統（原子操作、Dir 能力、reflink）](300-std/301-fs.md)
- 3.2. [`net::`：TCP/UDP/Unix + TLS 1.3（HTTP 為外部函式庫）](300-std/302-net.md)
- 3.3. [`time::`：時鐘、日曆日期與時區（搭配 `core::time` 的 `Instant` / `Duration`）](300-std/303-time.md)
- 3.4. [`thread::`：作業系統執行緒（1:1，與任務分開）](300-std/304-thread.md)
- 3.5. [`env::`：環境變數／引數／目錄](300-std/305-env.md)
- 3.6. [`process::`：`Command` / pidfd / kill_on_drop](300-std/306-process.md)
- 3.7. [`path::`：UTF-8 `Path` + `/` 運算子（Python pathlib 風格）](300-std/307-path.md)
- 3.8. [`nabion::`：NABION 資料格式（`.nabion`）](300-std/308-nabion.md)
- 3.9. [`random::`：亂數位元組、產生器與分布](300-std/309-random.md)
- 3.10. [`os::`：只有部分作業系統才有的功能（`unix` / `linux` / `windows`）](300-std/310-os.md)
- 3.12. [`panic::`：panic 鉤子（`set_hook` / `take_hook`）](300-std/312-panic.md)
- `wit::`：產生的 WIT 繫結所呼叫的 component model 非同步 ABI（[1.8](100-bare/108-wit.md)）；屬於繫結內部，不在此規範。

---

## 閱讀方式

### 初次閱讀

初次接觸時，建議先依照 [`refs-lang`](../refs-lang/README.md) 的第 0 至 6 章理解語言的機制，再以此為基礎閱讀本文件。本文件為 refs-lang 定義的機制賦予意義；若對這些機制沒有概念，trait 與簽章讀起來容易像是懸在半空中。把兩份文件並排、來回對照閱讀，是最有效率的方式。

### 作為參考

每個項目都以穩定的節號（`<chapter>.<section>`）識別。內部連結與外部引用使用相同的節號，因此即使檔案移動，也能可靠地指向該處。

```
refs-standard-libraries 2.3        →  core ops (Add/Sub/...)
refs-standard-libraries 1.1        →  bare Ptr<T>
refs-standard-libraries 3.1.2      →  std fs (specific subsection)
```

### 搜尋

要快速找到特定的 trait 或函式，最輕量的方式是用 `ripgrep` 縮小目錄範圍後搜尋。

```bash
rg "trait Add" spec/en/refs-standard-libraries/200-core/
rg "fn into" spec/en/refs-standard-libraries/
```

---

## 目錄／檔案命名

本文件採用與 `refs-lang` 不同的目錄慣例，因為兩份規格的形狀不同：refs-lang 是扁平結構，第 0 至 9 章權威性相同；refs-standard-libraries 則是階層結構，三個層級是交互參照中的關鍵資訊。這個結構差異直接反映在目錄名稱的位數上。

| 規格                    | 前綴                   | 理由                                                            |
|-------------------------|------------------------|-----------------------------------------------------------------|
| refs-lang               | 一位數（`002-`）       | 10 章（0-9），權威性相同；扁平形狀                              |
| refs-standard-libraries | 三位數（`200-`）       | 3 個層級 × N 個模組；層級是交互參照的關鍵資訊                   |

具體的檔案命名規則如下。層級目錄的前綴以 100 為單位，其中的模組檔案以三位數前綴開頭，由一位層級數字加兩位模組數字組成。

| 位置     | 形式                                   | 範例                              |
|----------|----------------------------------------|-----------------------------------|
| 層級     | `<TNN>-<descriptive-name>/`            | `100-bare/`、`200-core/`          |
| 模組     | `<tier><module>-<name>.md`             | `203-ops.md`、`211-conversion.md` |
|          | （一位層級數字 + 兩位模組數字）        |                                   |
| README   | 每個層級／章節中的 `README.md`         | 目錄 + 概觀                       |

雖然層級資訊已編碼在檔案路徑本身，但在外部引用本文件時，穩定的節號（`2.3.1` 等）才是正式的表示法。檔案路徑只是為了方便在 IDE 與 GitHub 中瀏覽的輔助表示法。
