# 2. Core 層級

core 層級（`core`）是安全的標準函式庫：Nabi 在每個目標上都能建置的實用介面。使用者程式碼約有 90% 使用這個層級的項目，因此它是決定 Nabi 在日常程式設計中樣貌的一層。

## 位置

規格文件位於：

- 規格：`spec/en/refs-standard-libraries/200-core/`

## 相依關係

往上，core 層級在內部使用上相依於 `bare`；往下，`std` 和使用者程式碼相依於它。因此 core 層級位於安全區域的中心，介於 `bare` 和 `std` 之間。

## 使用者匯入政策

`use core::prelude::*;` 是**隱含套用**的：編譯器會把它加在每個使用者檔案的開頭。prelude 中的名稱（List、Dict、Display、Iterator、Mutex、task……）不需明確匯入即可使用；core 的其餘部分則透過 prelude 模組（`ops::Add`、`task::sleep`）或 `use` 取用。

需要時也可以使用範圍較窄的明確匯入。

```nabi
use core::collections::List;           // narrow, explicit
use core::ops::{Add, Sub};           // operator traits are not in the prelude
```

> [!NOTE]
> prelude 的清單是規範性的，且不含任何列舉變體：裸變體從預期型別解析
>（[refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-依預期型別使用不帶路徑的變體)）。

完整政策請見 [0.2 匯入政策](../000-meta/002-import.md)。

## 模組索引

core 層級提供的模組，依領域分組。每個項目最後列出它所規範的 `core` 模組。

### 數值／niche

- [2.1. 數值型別（固定寬度、`Int`、`Float`）](201-numbers.md)：`num`
- [2.2. Niche 包裝型別（`NonZero`、`NonNan`、`Finite`、`Positive`）](202-non-zero.md)：`num`

### 運算子 trait

- [2.3. 算術／位元／邏輯（`Add`、`Sub`、`Mul`、`Div`、`Rem`、`Neg`、`Not`、`BitNot`、`BitAnd`、`BitOr`、`BitXor`、`Shl`、`Shr`、`LogicalAnd`、`LogicalOr`、`Index`）](203-ops.md)：`ops`、`index`
- [2.4. 複合指派（`AddAssign`、`SubAssign`、……、`ShrAssign`）](204-ops-assign.md)：`ops`
- [2.5. 比較（`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash`、`Hasher`）](205-cmp.md)：`cmp`、`hash`
- [2.6. 廣播（`DotAdd`、`DotMul`、……、`DotShr`）](206-broadcast.md)：`ops`

### iter／stream

- [2.7. iter（`Iterator`、`IntoIterator`、`Range`、`RangeInclusive`）](207-iter.md)：`iter`、`range`
- [2.8. `Stream`（非同步迭代器）](208-stream.md)：`stream`

### 集合型別

- [2.9. 集合型別（`List`、`Dict`、`Set`、`Slice`、`Deque`、`BinaryHeap`、`SortedDict` / `SortedSet`、陣列、元組）](209-collections.md)：`collections`、`array`、`tuple`

### 格式化／轉換

- [2.10. fmt（`Display`、`Debug`、`Formatter`、`print`）](210-fmt.md)：`fmt`、`print`
- [2.11. 轉換（`From`、`Into`、`TryFrom`、`TryInto`、`saturating_into`、`wrapping_into`）](211-conversion.md)：`convert`

### 元資訊／標記

- [2.12. 反射（`Reflect`、`Inspect`）](212-reflect.md)：`reflect`
- [2.13. marker（`Clone`、`Copy`、`Drop`、`NoneSentinel`）](213-marker.md)：`marker`

### 錯誤／字串

- [2.14. 錯誤（`Error`、`Result`、`TypeError`、`ValueError`、`OverflowError`、`panic`）](214-error.md)：`error`、`panic`
- [2.15. 字串（`str`、`StrBuf`、`char`、`c_char`、Unicode 正規化與切分）](215-string.md)：`str`

### 並行／非同步

- [2.16. 同步基本元件（`Atomic`、`Mutex`、`RwLock`、`Cell`）](216-sync.md)：`sync`
- [2.17. `Channel`：訊息傳遞](217-channel.md)：`channel`
- [2.18. `task::`：M:N 排程器任務 API](218-task.md)：`task`
- `time`：`Duration` / `Instant`，與 std 的時鐘一起在 [3.3](../300-std/303-time.md) 中規範
- [2.19. `par::`：平行處理 API](219-par.md)：`par`
- [2.20. `Future` / `await` 整合](220-future.md)：`future`

### 密碼學

- [2.21. `crypto::`：雜湊、MAC、AEAD、金鑰交換、簽章](221-crypto.md)：`crypto`

### 編碼

- [2.22. `encoding::`：hex 與 base64](222-encoding.md)：`encoding`

### I/O

- [2.23. io（`Read`、`BufRead`、`Write`、`Seek`、`AsyncRead`、`AsyncWrite`、`BufReader`、`Cursor`、stdio）](223-io.md)：`io`

### SIMD

- [2.24. simd（`Simd`、`Mask`、`f32x4`……）](224-simd.md)：`simd`

### 基準測試／最佳化器提示

- `bench`：`Bench`，`#[bench]` 函式收到的控制代碼。量測方式規範於 [refs-lang 9.9.11](../../refs-lang/009-tools/909-debug-tools.md#9911-效能測試)。

  ```nabi
  pub struct Bench { /* private fields */ }   // Debug: `Bench { name: "parse_1kb", .. }`
  impl Bench {
      pub fn iter<T>(ref self, f: fn() -> T);  // calibrate, warm up, sample and report f
      pub fn set_bytes(ref self, n: i64);      // bytes per call: adds a throughput column
      pub fn set_items(ref self, n: i64);      // items per call: adds a throughput column
  }
  ```
- `hint`：`black_box<T>(x: T) -> T`：原樣回傳 `x`，但最佳化器無法看穿它，因此產生 `x` 的計算和對結果的使用都不會被摺疊消除。基準測試以它包住計時區段的輸入和結果。

> [!NOTE]
> 可攜式 SIMD 向量屬於 core 層級（[2.24](224-simd.md)）。張量、自動微分和 GPU 等科學計算領域不屬於 core 層級，
> 而是拆分為**外部函式庫**。外部函式庫透過後端介面接觸編譯器的後端，該介面規劃於之後的版本推出。

## Core 的設計原則

core 層級的設計遵循四個軸向。

第一，trait 優先。每個語意上的決定都由 trait 方法分派做出：`+` 解析為 `Add::add`，`[i]` 索引解析為 `Index::index`。

第二，轉換是明確的。可能遺失資訊的轉換（例如 `i64 → i32`）會以編譯錯誤或執行時錯誤拒絕，而飽和或環繞之類的策略必須在每個運算上明確指定（[2.11](211-conversion.md)）。

第三，此層級以**對 GC 友善**為目標。集合型別和並行物件由 GC 管理。`Drop` 資源是仿射的（affine）：它在最後一個擁有者離開作用域處，或在其 `using` 區塊結束時釋放（[2.13.4](213-marker.md)）。

最後，保證**目標無關性**。每個項目在每個支援的目標上行為都相同，依賴目標的項目則拆分到 `std`。
