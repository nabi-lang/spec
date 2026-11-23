# 1.6. `Pinned<T>`：GC 固定

`bare::gc` 模組定義一種機制，可暫時把 GC 堆積上的物件固定住，使其無法移動。當原始 `Ptr<T>` 指向 GC 物件而該物件移動時，指標就會失效；本章說明如何解決這個衝突。

這是 Nabi 不安全層級中最重要的一章：它正面說明指標與 GC 之間的衝突如何調和。

## 1.6.1. 動機：為什麼需要固定

[refs-lang 7.3 GC 演算法](../../refs-lang/007-runtime/703-gc-algorithm.md)的 Immix mark-region GC 會進行**疏散**（evacuation，以複製的方式移動物件）。因此，物件的位址在 GC 週期之間可以任意改變。

```
  GC cycle start:    obj at 0x1000
  GC evacuation:     obj at 0x2000  (copied, then forwarded)
  GC cycle end:      obj at 0x2000
```

對於受管理的程式碼（`GcPtr<T>`），GC 會追蹤根並自動更新每個 ptr。然而原始 `Ptr<T>` 對 GC 不可見，因此不會被更新，而成為**過時的 ptr**（stale ptr）。

```nabi
use bare::gc;
use bare::ptr::Ptr;

let xs: List<i32> = [1, 2, 3];                 // GcPtr (updated automatically)
let raw: Ptr<i32> = Ptr::dangling();
using pin = gc::pin(xs) {
    raw = pin.as_data_ptr();                    // raw ptr, valid inside the pin
}                                               // unpinned here
gc::collect();                                  // xs may move
unsafe { raw.read() };                          // UB: raw is stale
```

解決方法是**固定**（pinning）：明確告訴 GC「不要移動這個物件」。

## 1.6.2. `Pinned<T>` 守衛

```nabi
struct Pinned<T> {
    obj: T;
    // internal: registered in the GC's pin set
}

#[sealed]
trait Pinnable {
    // the GC object holding the value's data; none when nothing can move
    #[internal("write `pin(x)` to keep a value in place")]
    fn gc_object(self) -> GcBuf<u8> | none;
}
impl<T> Pinnable for List<T> { /* the list's buffer; none when empty */ }
impl<T> Pinnable for Slice<T> { /* the buffer it views; none when empty */ }
impl<T, const N: usize> Pinnable for Array<T, N> { /* the element buffer; none when empty (2.9.14) */ }
impl Pinnable for str { /* none (2.15.2) */ }
impl Pinnable for StrBuf { /* the byte buffer; none while it has no room (2.15.2) */ }
impl<T> Pinnable for GcPtr<T> { /* the cell */ }

fn pin<T: Pinnable>(obj: T) -> Pinned<T>;
fn collect();                           // a full collection now; may move every unpinned object
fn stats() -> GcStats;                  // the collector's counters (1.6.10)
fn collect_young();                     // a minor collection now (1.6.11)
fn collect_young_within(ms: f64) -> bool;
fn set_pause_goal(ms: f64);             // 1.6.12
fn no_collect<R>(f: fn() -> R) -> R;    // 1.6.13
fn set_heap_limit(bytes: u64);          // 1.6.14
struct Arena<T>;                        // 1.6.15

impl<T> Pinned<T> {
    fn obj(self) -> T;
}

impl<T> Drop for Pinned<T> {
    fn drop(ref self) {
        // removed from the GC pin set
    }
}
```

它是一個守衛（guard）。只要 `pin()` 回傳的 `Pinned<T>` 還存活，GC 就不會移動該物件。`Pinned<T>` 是 `Drop` 值，只會釋放一次（[2.13.4](../200-core/213-marker.md#2134-drop釋放時的清理)）：在守衛的擁有者離開作用域之處，或在繫結它的 `using` 區塊結束時，`Pinned::drop` 會執行並解除對物件的固定。固定會計數：物件必須經過同樣次數的解除固定之後才會再次移動。`pin` 會固定 `gc_object` 所指出的物件；當它回傳 `none` 時則不固定任何東西。`GcBuf<u8>` 是 [1.3.9](103-mem.md#139-gcbuft-與-buf_-基本操作) 中抹除型別的 GC 緩衝區。`Pinnable` 是封閉的，`gc_object` 則是工具鏈內部的項目（[6.2.15](../../refs-lang/006-meta-system/602-built-in-attributes.md#6215-internal-由工具鏈的套件共用)）：程式以 `pin(x)` 固定，不呼叫 `gc_object`。

### 範例：FFI 呼叫

```nabi
let buf: List<u8> = read_input();

using pin = bare::gc::pin(buf) {
    let raw: Ptr<u8> = pin.as_data_ptr();
    unsafe {
        ffi::process(raw, buf.len());          // FFI uses buf's data
    }
}     // ← automatic unpin at end of scope
```

這裡的 `using` 區塊遵循 [refs-lang 3.17 using 運算式](../../refs-lang/003-expressions/317-using-expression.md)所定義的 RAII 形式。

## 1.6.3. `GcPtr<T>` 與 `Ptr<T>`：在型別層級分離

```nabi
struct GcPtr<T> { /* compiler-managed; roots traced automatically */ }
struct Ptr<T>   { /* raw; no root tracking */ }
```

| 面向       | `GcPtr<T>`                    | `Ptr<T>`                                         |
|------------|-------------------------------|--------------------------------------------------|
| GC 根      | **自動追蹤**                  | **不追蹤**                                       |
| 移動時     | **自動更新**                  | **過時**                                         |
| 使用於     | 一般程式碼（`List`/`Dict`/...） | 僅限 unsafe + FFI                              |
| 建立方式   | GC 配置（`List::new()` 等）   | `gc::pin().as_data_ptr()` 或 `alloc::alloc(layout)`   |
| 有效期間   | 只要物件可達                  | 僅在固定守衛的作用域內                           |

由於型別系統把這兩種區分開來，使用者程式碼通常只處理 `GcPtr`（以不可見的方式包在 `List<T>` 等之中）。`Ptr<T>` 只有在進入 unsafe 之後才會明確出現。

### 操作

```nabi
impl<T> GcPtr<T> {
    fn new(value: T) -> GcPtr<T>;           // a new GC cell holding value
    fn get(self) -> T;
    fn set(self, value: T);                 // every copy of the GcPtr sees it
    fn same(self, other: GcPtr<T>) -> bool; // identity: the same cell
}
impl<T> !Sync for GcPtr<T> {}
impl<T> Pinnable for GcPtr<T> {}
```

### 內部實作：GcPtr 的形式

```nabi
struct GcPtr<T> {
    // internal: usually a plain `*T`; the GC traces it externally via the root set
    // for a local variable, kept in a shadow stack frame slot emitted by the compiler
}
```

Nabi 的 GC 透過編譯器直接產生的**每執行緒影子堆疊**（shadow stack）追蹤根集合（[refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-根追蹤由編譯器直接發出的影子堆疊)）。

## 1.6.4. pin / unpin 在 GC 端的機制

```
GC pin set structure.
  - hash table: Ptr<Object> → pin count
  - during evacuation the GC _skips_ objects with pin count > 0

pin(obj):
  1. check the pin bit in obj's header (or register in the pin set)
  2. count++
  3. return Pinned<T>

drop(Pinned):
  1. count--
  2. if count == 0, clear the pin bit
```

這個結構實際上與 Go 1.21 的 `runtime.Pinner` 同構。詳見 [pkg.go.dev/runtime#Pinner](https://pkg.go.dev/runtime#Pinner)。

### 額外負擔

成本如下：

- 一次固定只是一次 O(1) 的雜湊插入
- 一次解除固定同樣是一次 O(1) 的雜湊移除
- GC 週期內會多一次固定集合檢查，但成本很小

在大多數情況下這不在熱路徑上，因為 FFI 呼叫本身並不那麼頻繁。

## 1.6.5. 何時需要固定

每個 `extern "C"` 呼叫都會讓呼叫端執行緒進入停駐狀態（[1.7.1](107-ffi.md#171-extern-c-區塊)）：C 執行期間，回收器不等待該執行緒，因此在任何 C 呼叫期間，不論多短，都可能進行回收並移動物件。C 在呼叫期間使用的、指向 GC 物件的指標，只有在以下情況才有效：

- 該物件在整個呼叫期間都已固定（`gc::pin`，1.6.2），且在 C 保留該指標的期間之後也持續固定；或
- 該指標指向某個根本身所指出的物件：`str` 引數（呼叫端的框架會將其作為根，直到呼叫回傳，[refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#被根指向的物件不會被移動)），以及在 [2.15.2](../200-core/215-string.md#2152-str不可變的-utf-8-字串) 規則下的 `str::as_ptr` / `CStr::as_ptr`。

其他所有指向 GC 物件的指標，例如串列的緩衝區（`as_data_ptr`）、切片的元素、`StrBuf` 的位元組、`GcPtr` 的值，都需要固定。不論是什麼呼叫，編譯器都不會自行省略固定：固定由程式撰寫。`#[leaf]`（[1.7.1](107-ffi.md#171-extern-c-區塊)）讓呼叫更便宜，但不改變這條規則。C 在多次呼叫之間保留的值要放進 `Handle`（[1.7.10](107-ffi.md#1710-handletc-持有的-nabi-值)），它會固定該值直到被釋放。

不持有原始指標的程式碼不需要固定：

| 場合             | 原因                                    |
|------------------|-----------------------------------------|
| 一般方法呼叫 | GcPtr 會自動更新；不會建立原始 ptr |
| GcPtr 之間的操作 | 全部由 GC 追蹤                   |
| 傳給 `extern "C"` 函式的 `str` | 引數的根在呼叫期間讓字串保持原位 |

### 已固定的 `List` 或 `StrBuf` 不能成長

讓串列或 `StrBuf` 成長超過其容量會替換它的緩衝區，而 C 可能持有舊緩衝區的位址。緩衝區固定期間，成長（`reserve`，以及透過它的 `push`、`push_str`、`extend` 等）會改為觸發 panic：

```text
cannot grow a list while it is pinned (C may hold its address): call `xs.reserve(n)` before `gc::pin(xs)`
cannot grow a `StrBuf` while it is pinned (C may hold its address): call `b.reserve(n)` before `gc::pin(b)`
```

固定前預留的空間可以在固定期間填入。

```nabi
use bare::gc::pin;

let ys: List<i64> = [1];
ys.reserve(7);                     // room for 8
let py = pin(ys);
for i in 0..7 { ys.push(i); }      // fits: the buffer stays
ys.push(9);                        // panics: the buffer would have to grow
```

## 1.6.6. 取出方法

```nabi
impl<T> Pinned<List<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // start of the List's buffer
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl<T> Pinned<Slice<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // the slice's first element; dangling when empty
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl<T, const N: usize> Pinned<Array<T, N>> {
    fn as_data_ptr(self) -> Ptr<T>;          // the first element; steps are strides(), in elements
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl<T> Pinned<GcPtr<T>> {
    fn as_value_ptr(self) -> Ptr<T>;         // the cell's value
}

impl Pinned<str> {
    fn as_bytes_ptr(self) -> ConstPtr<u8>;   // UTF-8 bytes
    fn as_c_ptr(self) -> ConstPtr<c_char>;   // null termination not guaranteed; convert separately
}

impl Pinned<StrBuf> {
    fn as_bytes_ptr(self) -> Ptr<u8>;        // the first byte (no NUL promised); dangling while there is no room
}
```

這些容器方法是取出 API，在持有 `Pinned` 的期間有效。（`str::as_ptr` 與 `CStr::as_ptr` 在 [2.15.2](../200-core/215-string.md#2152-str不可變的-utf-8-字串) 較窄的規則下，不需固定即可給出位址。）沒有泛用的 `as_ptr`；`as_value_ptr` 的命名與 `List` 的 `as_data_ptr` 區隔開來。每個容器都有專用的取出方法，因此可以用符合各容器佈局的形式取出資料，例如緩衝區起點或以 null 結尾的形式。

## 1.6.7. 與 GC 執行環境的連結

各項目對應的 GC 執行環境呼叫如下。

| 項目             | 執行環境呼叫                   |
|------------------|--------------------------------|
| `Pinned<T>`      | `nabi_gc_pin` / `nabi_gc_unpin`  |
| `pin<T>()`       | （同上）                       |
| `collect()`      | `nabi_gc_collect`               |
| `stats()`        | `nabi_gc_stats`（1.6.10）       |
| `collect_young()` / `collect_young_within(ms)` | `nabi_gc_collect_young` / `nabi_gc_collect_young_within`（1.6.11） |
| `set_pause_goal(ms)` | `nabi_gc_set_pause_goal`（1.6.12） |
| `no_collect(f)`  | `nabi_gc_no_collect`（1.6.13）  |
| `set_heap_limit(bytes)` | `nabi_gc_set_heap_limit`（1.6.14） |
| `Arena<T>`       | `nabi_gc_arena_epoch`（1.6.15） |
| `GcPtr<T>`       | 影子堆疊框架槽位（由編譯器產生） |
| `as_data_ptr` 等 | 取決於各容器的佈局             |
| `Weak<T>`        | `nabi_gc_weak_new` / `nabi_gc_weak_get`（1.6.9） |
| 已固定緩衝區的成長檢查（1.6.5） | `nabi_gc_pinned` |

## 1.6.8. 與其他語言的比較

| 面向             | Java JNI                       | .NET fixed             | Go 1.21 Pinner           | Nabi                       |
|------------------|--------------------------------|------------------------|--------------------------|---------------------------|
| 固定 API         | `Get*Critical`                 | `fixed (T* p = ...)`   | `runtime.Pinner`         | `bare::gc::pin(obj)`  |
| RAII 守衛        | 明確呼叫 `Release`             | 區塊作用域             | `defer pin.Unpin()`      | `using` 區塊（自動 drop） |
| 型別分離         | **無**（全是 `jobject`）       | **無**                 | **無**（所有 ptr 都相同） | `GcPtr<T>` 與 `Ptr<T>`   |
| 固定後的失效檢查 | **無**                         | 編譯器（區塊作用域）   | **無**（手動）           | 型別系統 + 作用域         |
| 多重固定         | 可重疊                         | **受限**               | 可重疊                   | 可重疊（計數）            |

Nabi 與其他語言主要在兩點上不同。

1. 由型別系統強制。`GcPtr<T>` 與 `Ptr<T>` 是不同的型別，自動轉換會被拒絕；唯一的轉換途徑是透過固定守衛。
2. 以 `using` 區塊為基礎釋放。`Pinned<T>` 在離開 `using` 區塊時被 drop，因此絕不會漏掉解除固定。這比 Go 手動 `defer` 的做法更安全。

> 這項方針是 Nabi 指標模型的骨幹：unsafe 與 GC 這兩個領域安全交會之處。

## 1.6.9. `Weak<T>`：弱參考

```nabi
// bare::gc
struct Weak<T> { /* opaque */ }

impl<T> Weak<T> {
    fn new(target: GcPtr<T>) -> Weak<T>;
    fn get(self) -> GcPtr<T> | none;        // none once the target is freed
}
impl<T> !Sync for Weak<T> {}
impl<T> !Send for Weak<T> {}    // a hand-off cannot copy what it names (2.13.9)
```

弱參考不會讓它的物件保持存活。一旦某次回收釋放了該物件，所有指向它的 `Weak` 都會讀到 `none`；移動該物件的回收則會更新 `Weak`。這是 `WEAK_REF` 掃描種類（[refs-lang 8.2.8](../../refs-lang/008-abi/802-scan-table.md#828-weak_ref)）的函式庫介面。

執行時，`Weak` 是型別 ID 為 `3` 的 GC 儲存格（[refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-執行環境保留的型別-id)），內含一個 `WEAK_REF` 字組，由 `nabi_gc_weak_new` 建立、由 `nabi_gc_weak_get` 讀取（[refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-輔助符號附錄非規範性)）。

## 1.6.10. `stats`：回收器計數器

```nabi
// bare::gc
struct GcStats {
    pub collections: u64;      // every kind
    pub minor: u64;            // minor collections
    pub full: u64;             // full collections that stopped the world throughout
    pub concurrent: u64;       // full collections marked concurrently (7.5)
    pub pause_total_ns: u64;   // time the program stood stopped, all pauses together
    pub pause_max_ns: u64;     // the longest pause
    pub live_bytes: u64;       // alive at the last collection, plus allocated since
    pub heap_bytes: u64;       // held from the operating system (blocks and large objects)
    pub moved_bytes: u64;      // moved by evacuation
    pub assists: u64;          // times an allocating thread marked for a concurrent mark (7.5.4)
    pub grown_bytes: u64;      // bytes no_collect regions grew the heap by (1.6.13)
    pub limit_collections: u64; // collections the heap limit brought forward (1.6.14)
    pub young_budget: u64;     // bytes allocated between two minor collections now
}

fn stats() -> GcStats;
```

`stats()` 讀取程式啟動以來的計數器，供指標統計或影格預算使用。它會短暫取得回收器的鎖，因此是用來偶爾讀取的（一次匯出、每個影格一次），而不是在熱迴圈中讀取。暫停從停止請求起算，因此包含抵達安全點的時間（[refs-lang 9.9.8](../../refs-lang/009-tools/909-debug-tools.md#998-記憶體--gc-工具)）。

`collect()` 在整個完整回收期間停止所有執行緒，比執行環境自行啟動的回收（次要回收，或並行標記的完整回收）更久。要讓暫停保持短暫，就把回收交給執行環境，或在影格之間呼叫 `collect_young`（1.6.11）。

## 1.6.11. `collect_young`：依需求進行次要回收

```nabi
// bare::gc
fn collect_young();
fn collect_young_within(ms: f64) -> bool;
```

`collect_young()` 立即進行一次次要回收：它檢查上次回收以來配置的物件，因此成本大約取決於其中存活下來的部分，而不是堆積的大小。兩個影格或兩個請求之間是回收成本低的時機。並行標記（[refs-lang 7.5](../../refs-lang/007-runtime/705-concurrent-collection.md)）執行期間，它什麼也不做：標記會自行完成，不會造成長時間暫停。

`collect_young_within(ms)` 只在最近幾次次要回收顯示其耗時至多 `ms` 毫秒（影格剩餘的時間）時才執行，並回報是否有執行。估計值是最近次要回收每個年輕位元組的成本乘以目前的年輕位元組數，再加上一個固定部分；尚未觀察到任何次要回收時，它會執行。

## 1.6.12. `set_pause_goal`：暫停目標

```nabi
// bare::gc
fn set_pause_goal(ms: f64);   // 0: no goal (the default)
```

設定目標後，回收器會切分工作，讓每次暫停接近 `ms` 毫秒：

- 年輕預算跟隨目標：取最近次要回收顯示能放進目標的量的四分之三，範圍從 1 MiB 到預設預算（[refs-lang 7.2.8](../../refs-lang/007-runtime/702-allocation.md#728-gc-觸發)）。`GcStats::young_budget` 顯示目前的值。
- 不論堆積大小，每次完整回收都並行標記，而且堆積絕不會自行疏散：只有 `collect()` 會壓縮它。
- 並行標記之後的清掃及其標記協助本來就在暫停之外執行（[refs-lang 7.5.2](../../refs-lang/007-runtime/705-concurrent-collection.md#752-階段)）。

這是目標，而非保證：暫停仍可能更長，例如在存活堆積快速成長時，或在配置量很大的並行標記剛結束後。

## 1.6.13. `no_collect`：不進行回收的區域

```nabi
// bare::gc
fn no_collect<R>(f: fn() -> R) -> R;
```

`no_collect(f)` 呼叫 `f`，在它執行期間，任何執行緒上的配置都不會啟動回收：沒有次要回收、沒有完整回收，也沒有壓力測試回收（`NABIR_GC_STRESS`）。區域在 `f` 回傳之處關閉，不論它以何種方式離開。區域可以巢狀，也可以同時在多個執行緒上開啟；最後一個區域關閉時，回收才會恢復。

- 區域內年輕預算用盡時，堆積會改為成長，每次至少 1 MiB；`GcStats::grown_bytes` 會計入。回收延到區域之後，在下一次需要回收的配置時進行。
- 正在執行的並行標記在區域期間繼續標記，並在區域之後完成（其最後一次暫停）。
- 堆積上限（1.6.14）仍然有效：會超過上限的配置會進行回收，在區域內也是如此。
- 在區域內呼叫的 `collect()` 與 `collect_young()` 仍會進行回收。

## 1.6.14. `set_heap_limit`：堆積上限

```nabi
// bare::gc
fn set_heap_limit(bytes: u64);   // 0: no limit (the default)
```

每個週期的預算上限為 `bytes`：堆積接近上限時回收會提早進行，而由上限提前的回收一律是完整回收（計入 `GcStats::limit_collections`）。上限計算的是 GC 物件的位元組數，包含已分出的年輕位元組，而不是行程的全部記憶體。存活資料超過上限並不是錯誤：此時上限變成存活位元組數再加八分之一（至少 4 MiB），因此堆積會一點一點地超過上限成長，每一步都進行完整回收。會超過上限的並行標記會在一次暫停中完成。

## 1.6.15. `Arena<T>`：一個影格的值

```nabi
// bare::gc
struct Arena<T> { /* shared by copies */ }
struct ArenaRef<T> { /* plain data: an index and a generation */ }
impl<T> Copy for ArenaRef<T> {}

impl<T> Arena<T> {
    fn new() -> Arena<T>;
    fn with_capacity(n: usize) -> Arena<T>;
    fn alloc(self, value: T) -> ArenaRef<T>;   // append in place
    fn get(self, r: ArenaRef<T>) -> T;         // panics for a stale reference
    fn try_get(self, r: ArenaRef<T>) -> T | none;
    fn set(self, r: ArenaRef<T>, value: T);    // panics for a stale reference
    fn holds(self, r: ArenaRef<T>) -> bool;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;
    fn reset(self);                            // empty at once; every ArenaRef so far is stale
    fn values(self) -> List<T>;                // a new list, in allocation order
}
impl<T> !Sync for Arena<T> {}
```

`Arena<T>` 把一個影格（或一個請求）的值放在同一個緩衝區中：`alloc` 就地附加一個值，因此回收器既不必為每個值配置物件，也不必逐一追蹤；`reset` 則立即清空 arena，同時保留緩衝區供下一個影格使用。`Arena` 的複本共用同一個 arena。

記憶體安全建立在兩條規則上：

- 緩衝區是一般的 GC 物件。值在 arena 中的期間，它們參考的東西都保持存活，arena 中沒有任何東西會成為懸置參考。（`reset` 之後，持有 GC 參考的型別的值會被清除；不持有 GC 參考的型別的值則由後續的 `alloc` 覆寫。）
- `ArenaRef` 是純資料：一個索引和一個世代。每次 `reset` 都取得程式中任何 arena 都未曾用過的世代（[refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-輔助符號附錄非規範性)），因此 reset 之前的參考，或來自另一個 arena 的參考，都無法通過 `get` 和 `set` 中的檢查而觸發 panic（`try_get` 給出 `none`，`holds` 回傳 `false`）。它可以存放在任何地方，包括 GC 物件中：晚了才使用它是有定義的 panic，絕不會讀到下一個影格的值。

因此 arena 不需要生命週期、不需要帶標記的作用域，也不需要限制 `T` 可以持有什麼；代價是每次存取一次整數比較。

```nabi
use bare::gc;
use bare::gc::Arena;

struct Particle { x: f64; y: f64; }

let particles: Arena<Particle> = Arena::with_capacity(4096);
loop {
    let p = particles.alloc(Particle { x: 0.0, y: 0.0 });
    update(particles, p);
    draw(particles);
    particles.reset();                         // the frame's particles are gone
    gc::collect_young_within(frame_left_ms());
}
```
