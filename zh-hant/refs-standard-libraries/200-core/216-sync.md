# 2.16. sync：並行原語

`core::sync` 模組定義了安全存取共用狀態所需的基礎 trait 與型別。它們用於在 Nabi 的 GC 與 M:N 排程器之下，於多個任務或執行緒之間共用資料。

`Atomic`、`Mutex`、`RwLock` 和 `Cell` 位於 [prelude](../000-meta/002-import.md) 中，不需匯入即可使用；記憶體順序列舉（`LoadOrdering`、`StoreOrdering`、`UpdateOrdering`）則不在其中，但它們的變體可由參數型別解析（`x.load(SeqCst)`）。若希望並行程式碼一眼就能看出是並行程式碼，建議使用如下的窄範圍匯入。

```nabi
use core::sync::{Atomic, Mutex, RwLock, Once, Condvar, Cell};
```

## 2.16.1. 原語概覽

每個原語各自涵蓋不同的並行情境。下表整理了各型別的意義與適用場合。

| 型別        | 意義                                    | 用途                     |
|-------------|--------------------------------------------|--------------------------------|
| `Atomic<T>` | 無鎖原子操作（T = 基本型別） | 計數器、旗標、序號 |
| `Mutex<T>`  | 互斥鎖（單一寫入者）             | 共用的可變狀態           |
| `RwLock<T>` | 多個讀取者或單一寫入者        | 以讀取為主的狀態               |
| `Cell<T>`   | 執行緒區域的內部可變性（無鎖） | 單執行緒中的可變狀態          |
| `Once`      | 只執行一次初始化（2.16.4a） | 惰性的全域設定              |
| `Condvar`   | 在 `Mutex` 之下等待條件（2.16.4a） | 生產者 / 消費者       |
| `Frozen<T>` | 以唯讀方式共用複製的值（2.16.4b） | 共用的設定    |
| `FrozenList<T>`, `FrozenDict<K, V>` | 無人修改、不需複製即可共用的集合型別（[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s唯讀集合型別)） | 表格、以讀取為主的對應 |

## 2.16.2. `Atomic<T>`：無鎖原子

`Atomic<T>` 是最輕量的原語。其內部以各 T 專用的原子指令實作，形式如下。

```nabi
struct Atomic<T> {
    // internal: a one-element cell driven by per-T atomic instructions
}

trait AtomicPrimitive { /* marker: i8/i16/i32/i64/isize, u8/u16/u32/u64/usize, bool, Ptr<T> */ }

impl<T: AtomicPrimitive + PartialEq> Atomic<T> {
    const fn new(value: T) -> Atomic<T>;

    fn load(self, ordering: LoadOrdering = SeqCst) -> T;
    fn store(self, value: T, ordering: StoreOrdering = SeqCst);
    #[discardable]
    fn swap(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn compare_exchange(
        self, current: T, new: T,
        success: UpdateOrdering, failure: LoadOrdering,
    ) -> Result<T, T>;                  // the failure result is a value too, not an error
    fn compare_exchange_weak(
        self, current: T, new: T,
        success: UpdateOrdering, failure: LoadOrdering,
    ) -> Result<T, T>;                  // the failure result is a value too, not an error

    // integers only; each fetch_* is #[discardable]
    fn fetch_add(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn fetch_sub(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn fetch_and(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn fetch_or(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
    fn fetch_xor(self, value: T, ordering: UpdateOrdering = SeqCst) -> T;
}

impl Atomic<i32> {
    fn wait(self, expected: i32);                          // waits while the value is expected
    fn wait_timeout(self, expected: i32, timeout: Duration);   // wait, at most timeout (none left: returns at once)
    fn wake(self);                                         // wakes every thread waiting on this cell
}
```

`wait` 可能虛假地（spuriously）結束等待，因此呼叫端必須重新檢查條件（futex 語意）。

`store` 是原子儲存，而不是交換。在此實作中，`compare_exchange_weak` 永遠不會虛假地失敗；其行為與 `compare_exchange` 相同。需要 `PartialEq` 是因為 `compare_exchange` 會將舊值與 `current` 比較；`Ptr<T>` 以位址比較。

### `LoadOrdering`, `StoreOrdering`, `UpdateOrdering`

原子操作有五種記憶體順序強度。每一類操作都有自己的順序型別，只包含它能接受的順序，因此無效的順序會是編譯錯誤，而非執行時檢查（沿用 Swift Atomics 的分法）。

```nabi
enum LoadOrdering   { Relaxed, Acquire, SeqCst }                     // load; a failed compare-exchange
enum StoreOrdering  { Relaxed, Release, SeqCst }                     // store
enum UpdateOrdering { Relaxed, Acquire, Release, AcqRel, SeqCst }    // swap, fetch_*, a compare-exchange that succeeds
```

| 順序  | 意義                                         |
|-----------|-------------------------------------------------|
| `Relaxed` | 不同步（計數器等）              |
| `Acquire` | 保證載入之後的同步       |
| `Release` | 保證儲存之前的同步     |
| `AcqRel`  | 讀取-修改-寫入的雙向同步         |
| `SeqCst`  | 全域循序一致性（預設，安全）   |

這些順序具有 C++20 中的意義；它們所屬的記憶體模型（happens-before、哪些函式庫操作會同步、資料競爭）見 [refs-lang 7.9](../../refs-lang/007-runtime/709-memory-model.md)。預設為安全的 `SeqCst`。載入不能 release，儲存也不能 acquire：`a.store(x, Acquire)` 是編譯錯誤（`StoreOrdering` 沒有 `Acquire`；儲存接受 `Relaxed`、`Release` 或 `SeqCst`）。compare-exchange 的失敗順序是載入順序。變體不加前綴書寫：由參數的型別決定列舉（[refs-lang 2.4.8](../../refs-lang/002-types/204-enum.md#248-依預期型別使用不帶路徑的變體)）。這三個型別不在 prelude 中，其名稱也不會與比較用的 `cmp::Ordering` 衝突。順序引數必須是編譯期常數。較弱的順序應當謹慎使用，且只用於效能關鍵之處。

### 範例

最常見的用法是遞增計數器，寫法如下。

```nabi
static COUNTER: Atomic<i32> = Atomic::new(0);

task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
task::spawn(async { COUNTER.fetch_add(1, Relaxed); });
// result: COUNTER == 2 (no race)

let v = COUNTER.load(SeqCst);     //→ 2
```

### CAS（compare-and-swap）

原子未直接提供的更新，是以 compare-and-swap 迴圈建構的。下例會保留曾經儲存過的最大值。

```nabi
fn store_max(cell: Atomic<i64>, x: i64) {
    let cur = cell.load(Acquire);
    loop {
        if cur >= x { break; }
        match cell.compare_exchange(cur, x, AcqRel, Acquire) {
            Ok(_)       => break,
            Err(actual) => cur = actual,          // another task got there first; retry with its value
        }
    }
}
```

元素型別必須是 `AtomicPrimitive`：`List<T>` 這類集合型別無法放進 `Atomic`。此 trait 是封閉的（sealed，[refs-lang 6.2.14](../../refs-lang/006-meta-system/602-built-in-attributes.md#6214-sealed-只在定義套件中實作)）：上述型別就是它全部的實作，在其他套件中撰寫 `impl AtomicPrimitive` 是錯誤。共用的集合型別應放進 `Mutex<T>`（2.16.3）。

## 2.16.3. `Mutex<T>`：互斥鎖

難以只用原子表達的共用狀態使用 `Mutex<T>`。其內部由 futex 與酬載組成；以 `using` 繫結的守衛會釋放鎖。

```nabi
struct Mutex<T> {
    // internal: futex + payload
}

impl<T> Mutex<T> {
    const fn new(value: T) -> Mutex<T>;
    const unsafe fn new_unshared(value: T) -> Mutex<T>;   // new without the hand-off copy

    fn lock(self) -> MutexGuard<T>;            // blocks the thread
    fn lock_async(self) -> LockFuture<T>;      // await m.lock_async(): suspends only the task (Future<Output = MutexGuard<T>>)
    fn try_lock(self) -> MutexGuard<T> | none; // non-blocking
    fn with<R>(self, f: fn(T) -> R) -> R;      // lock, f(value), unlock
}

struct MutexGuard<T> { /* RAII guard */ }

impl<T> Drop for MutexGuard<T> {
    fn drop(ref self) {
        // futex_wake + release the lock
    }
}

impl<T> MutexGuard<T> {
    fn get(self) -> T;                  // a scoped view of the protected value
    fn get_clone(self) -> T where T: Clone;   // a deep copy, free to outlive the lock
    fn set(self, value: T);             // replaces the value
    fn update(self, f: fn(T) -> T);     // reads, transforms and writes back
}
```

守衛不會轉發其所保護之值的方法：在 `MutexGuard<List<i32>>` 上呼叫 `g.push(1)` 是錯誤。請透過 `get`、`set` 和 `update` 存取該值。

`m.with(f)` 會上鎖、以受保護的值呼叫 `f`，並在 `f` 回傳時解鎖，回傳其結果：`log.with(|v| v.push(x))`、`let n = log.with(|v| v.len())`。該值如同 `get()` 的檢視一樣借給 `f`，遵循 `ThreadLocal::with` 的規則：它不得離開 `f`（被回傳、儲存、傳送、由被呼叫端或閉包保留），因此 `f` 不能是 `async`（E0136）；請回傳複本（`.clone()`）或由它算出的結果。若要跨陳述式或跨 `await` 持有鎖，請保留守衛（`using g = m.lock()`）。`RwLock::read_with(f)` 和 `write_with(f)` 分別在讀取鎖與寫入鎖之下做同樣的事。

### 範例

以 `using` 繫結的守衛會在離開區塊時被 `drop`，從而釋放鎖。未以 `using` 接收的守衛不會自動釋放，因此必須一律以 `using` 繫結。守衛是從呼叫、方法呼叫或 `await` 接收的。接收到的守衛只能用來初始化繫結（`let`、模式、`using`）、指派給變數，或交給呼叫端（`return`、函式的值）。把它傳給函式、在它上面呼叫方法、存入欄位或集合型別，或將其捨棄，都是編譯錯誤。以 `using` 繫結的守衛可以傳給函式；未以 `using` 繫結的守衛不會因為被傳遞而釋放。以一般的 `let`（或 `if let`）繫結守衛而始終不由 `using` 釋放，或捨棄 `lock` 回傳的守衛，都是編譯錯誤。

`Mutex::new(value)` 會接管 `value`：之後使用呼叫端的控制代碼是編譯錯誤；同樣是編譯錯誤的還有：在守衛的 `using` 區塊結束後使用 `get` 回傳的或 `set` 給出的內容；在守衛持有期間（直到 `set` 給鎖一個新值為止）交出 `get` 回傳的內容；或將其回傳、yield、作為區塊的值，或交給在區塊之後執行的協程（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）。擲回持有它的錯誤（`throw Leak { xs: g.get() };`）也是如此：攔截端在鎖釋放之後才執行。`get` 回傳的是有作用域的檢視：它與守衛一樣既不是 `Send` 也不是 `Sync`，把它傳給會保留它的被呼叫端（存入 `Cell`、欄位、擷取、執行緒區域變數）會在該呼叫處產生編譯錯誤 E0106，其說明會指向 `get_clone()`。`get_clone()` 回傳深層複本（`T: Clone`），可以比鎖存活得更久。請在 `using` 區塊內操作檢視，或保留一份 `get_clone()`。守衛是 `!Sync`，只在其 `using` 區塊內有效：在區塊結束後使用它的複本（在集合型別、閉包、欄位中）是編譯錯誤；若要在另一個執行緒上執行的閉包中使用該值，請先取得深層複本（`let v = g.get_clone();`）並擷取 `v`。`RwLockReadGuard` 和 `RwLockWriteGuard` 有相同的 `get` 與 `get_clone`。在執行時，`new`、`set` 和 `update` 會儲存值所帶入之集合型別的複本（2.13.9）；`new_unshared` 省略此複製，由呼叫端保證沒有其他東西能觸及 `value` 的集合型別（剛建立的值）。

```nabi
static STATE: Mutex<List<i32>> = Mutex::new([]);

task::spawn(async {
    using g = STATE.lock() {
        g.update(|xs| { xs.push(42); xs });
    }   // ← g.drop() → lock released ([2.13 marker](213-marker.md))
});
```

### 避免死結：`try_lock`

為避免死結，可以用 `try_lock` 進行非阻塞的嘗試，而不是阻塞。

```nabi
match STATE.try_lock() {
    v: sync::MutexGuard<List<i32>> => using g = v { g.update(|xs| { xs.push(1); xs }); },
    none => { log("locked, skip"); },
}
```

### 中毒（poisoning）：**無**

Nabi 的 Mutex 與 Rust 的 PoisonError 模型不同。即使發生 panic，鎖也會自動釋放，下一個任務可以正常進入。這樣設計有兩個理由。

1. panic 無法被攔截並會結束行程，因此其他任務永遠不會觀察到 panic 進行中的鎖狀態。唯一的例外是 C 所呼叫之函式上的 `#[on_panic]`：在那裡，被略過的框架不會執行其 drop，它們鎖上的 `Mutex` 會保持鎖定（持有鎖時請回傳錯誤值，而不要觸發 panic）。當錯誤透過 `throw` 傳播出 `using` 區塊時，[`using` 會呼叫 Drop](213-marker.md#2134-drop釋放時的清理)，因此由 `MutexGuard` 的 drop 負責釋放鎖。
2. 中毒看似保守，卻給使用者帶來沉重負擔，而且在大多數情況下是誤判。

若確實必須在帶著錯誤離開前恢復一致性，請明確處理：

```nabi
using g = MUTEX.lock() {
    try unsafe_op(g) catch e {
        g.set(known_good_state());
        throw e;
    };
}
```

## 2.16.4. `RwLock<T>`：讀寫分離

讀取遠多於寫入時，`RwLock<T>` 更合適。將多個讀取者與單一寫入者分開，可以提高吞吐量。

```nabi
struct RwLock<T> { /* ... */ }

impl<T> RwLock<T> {
    const fn new(value: T) -> RwLock<T>;

    fn read(self) -> RwLockReadGuard<T>;        // multiple readers OK
    fn write(self) -> RwLockWriteGuard<T>;      // single writer

    fn try_read(self) -> RwLockReadGuard<T> | none;
    fn try_write(self) -> RwLockWriteGuard<T> | none;

    fn read_async(self) -> ReadFuture<T>;       // await l.read_async(): suspends only the task
    fn write_async(self) -> WriteFuture<T>;

    fn read_with<R>(self, f: fn(T) -> R) -> R;  // as Mutex::with, under the read lock
    fn write_with<R>(self, f: fn(T) -> R) -> R; // as Mutex::with, under the write lock
}

struct RwLockReadGuard<T> { /* RAII guard: releases the read lock */ }

impl<T> RwLockReadGuard<T> {
    fn get(self) -> T;                  // a scoped view of the protected value
    fn get_clone(self) -> T where T: Clone;   // a deep copy, free to outlive the lock
}

struct RwLockWriteGuard<T> { /* RAII guard: releases the write lock */ }

impl<T> RwLockWriteGuard<T> {
    fn get(self) -> T;
    fn get_clone(self) -> T where T: Clone;
    fn set(self, value: T);             // replaces the value
    fn update(self, f: fn(T) -> T);     // reads, transforms and writes back
}

// both guards: Drop (releases the lock), Debug, !Sync
```

它適合設定或快取這類以讀取為主的狀態，對寫入稀少的工作負載而言，其吞吐量高於 Mutex。

```nabi
static CONFIG: RwLock<Config> = RwLock::new(Config::default());

using c = CONFIG.read() {        // concurrent with other readers OK
    process(c);
}   // ← read guard drop, lock released

using w = CONFIG.write() {       // briefly blocks all readers
    w.set(Config::load_from_file());
}
```

`RwLock<T>` 只有在 `T` 是 `Send + Sync` 時才是 `Sync`（[2.16.5](#send--sync-標記)），因此這裡的 `Config` 必須是 `Sync`。其中的集合型別要凍結（`RwLock<FrozenDict<K, V>>`）：讀取者共用它，寫入者則整個替換它（2.16.4b）。

與 `Mutex` 相同，`RwLock::new(value)` 會接管 `value`：之後使用呼叫端的控制代碼是編譯錯誤；同樣是編譯錯誤的還有：在守衛的 `using` 區塊結束後使用其 `get` 回傳的或 `set` 給出的內容；在守衛持有期間（直到 `set` 給鎖一個新值為止）交出 `get` 回傳的內容；或從區塊內部將其回傳（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）。請在 `using` 區塊內操作它，或保留一份 `clone()`。與 `Mutex` 相同，`new` 以及寫入守衛的 `set` 和 `update` 會儲存複本。

`RwLock` 的守衛也適用相同規則：以一般的 `let`（或 `if let`）繫結 `read` / `write` 的守衛而始終不由 `using` 釋放，或將其捨棄，都是編譯錯誤。

## 2.16.4a. `Once` / `Condvar`

兩個與 Rust 形式相同的原語。沒有中毒。

```nabi
struct Once { /* ... */ }
impl Once {
    const fn new() -> Once;
    fn call_once(self, f: fn());          // runs f only the first time; concurrent callers wait until it finishes
    fn is_completed(self) -> bool;        // true once a call_once has finished
}

struct Condvar { /* ... */ }
impl Condvar {
    const fn new() -> Condvar;
    fn wait<T>(self, guard: MutexGuard<T>) -> MutexGuard<T>;   // releases the lock and waits, relocks on wakeup
    fn wait_async<T>(self, guard: MutexGuard<T>) -> CondvarWait<T>;   // g = await cv.wait_async(g): suspends only the task
    fn notify_one(self);
    fn notify_all(self);
}
```

`wait` 可能虛假地喚醒，因此要在迴圈中重新檢查條件。`notify_one` 至少喚醒一個等待者，且可能喚醒不只一個非同步等待者。

### 非同步鎖與取消

`lock_async` / `read_async` / `write_async` / `wait_async` 的 future 在被取消的輪詢中不會取得鎖（[2.18.4](218-task.md#2184-協作式取消taskcancel)）。被取消的 `wait_async` 以如下方式結束。

- 若互斥鎖空閒，它會重新取得互斥鎖，因此呼叫端的 `using` 守衛照常釋放它。
- 若另一個任務持有互斥鎖，它不取得就離開，而被捨棄的守衛不會釋放該互斥鎖。

## 2.16.4b. `Frozen<T>`：唯讀共用

只建立一次、之後只被讀取的值（設定、信任錨點、表格）透過 `Frozen<T>` 共用。任何執行緒都可以不加鎖地讀取它。

```nabi
struct Frozen<T> { /* ... */ }   // Sync when T: Send

impl<T> Frozen<T> {
    const fn new(value: T) -> Frozen<T>;   // keeps a copy no one else reaches
    fn get(self) -> T;                       // the reader's own copy
}
```

對於單一的大型集合型別，請改為凍結它（[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s唯讀集合型別)）：`FrozenList` / `FrozenDict` / `FrozenSet` 由每個執行緒就地讀取，完全不需複製。`Frozen<T>` 用於由多個部分組成的值，其中一些部分是集合型別，每個讀取者可以在自己的複本中修改它們。

`new` 會保留 `value` 所觸及之集合型別的複本，就像交接（hand-off）時一樣（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)），因此外部沒有任何東西能觸及它所持有的內容。`get` 給每個讀取者一份自己的複本：修改它不會改變其他讀取者看到的任何東西。`Sync` 的部分（`Mutex`、會自行上鎖的快取、不可變的 `Sync` 型別的值）不會被複製而保持共用，因此一次 `get` 只需複製該值的其他集合型別。請讓大型的唯讀部分不進入那份複本：將它們保存為 `Sync` 型別，或放在自己的 `Frozen` 欄位中，於需要處讀取。

```nabi
struct Settings {
    name: str;
    limits: List<i64>;               // copied by each get: small
    table: Frozen<List<Entry>>;      // shared: read with table.get() where needed
}

static SETTINGS: Frozen<Settings> = Frozen::new(load());
```

標準函式庫的 TLS `Certificate` 就是這樣的型別：剖析後永不改變（其存取方法給出複本），因此 `Frozen<ClientConfig>` 的 `get` 會共用其信任錨點。

### 以讀取為主的狀態：快照

許多執行緒讀取、偶爾才改變的狀態（伺服器的路由表、快取、會重新載入的設定），是放在鎖後面的凍結快照。讀取者取得目前的快照（`Sync` 值可以離開鎖的閉包或 `using` 區塊），並在不持有鎖的情況下讀取；寫入者從複本建構下一個快照並放入。取得舊快照的讀取者會繼續讀取它，等到沒有人再讀取時，回收器就會釋放它。

```nabi
static ROUTES: Mutex<FrozenDict<str, Route>> = Mutex::new(FrozenDict::new());

fn lookup(path: str) -> Route | none {
    let routes = ROUTES.with(|r| r);          // the current snapshot; the lock is held only for this
    routes.get(path)
}

fn add_route(path: str, r: Route) {
    using g = ROUTES.lock() {
        let next = g.get().to_dict();       // a copy to change
        next[path] = r;
        g.set(next.freeze());
    }
}
```

`RwLock<FrozenDict<..>>` 搭配 `read_with` / `write` 的運作方式相同；若鎖所保護的只有取得快照這件事，`Mutex` 就足夠了。

## 2.16.5. `Cell<T>`：單執行緒內部可變性

最後，`Cell<T>` 提供只在單一任務內有效的內部可變性。

```nabi
struct Cell<T> { /* ... */ }

impl<T> Cell<T> {
    const fn new(value: T) -> Cell<T>;
    fn get(self) -> T            where T: Copy;
    fn set(ref self, value: T);
    #[discardable]
    fn replace(ref self, value: T) -> T;
    #[discardable]
    fn take(ref self) -> T       where T: Default;
}
```

`Cell<T>` 意指執行緒區域的內部可變性，因此跨執行緒並不安全。它只能在單一任務內共用，而沒有鎖使它非常快速。

```nabi
let c: Cell<i32> = Cell::new(0);
c.set(42);
let v = c.get();          //→ 42

// sharing across tasks: compile error (Cell has no Send / Sync)
task::spawn(async { c.set(100) });       // ERROR: Cell is not cross-task safe
```

### `Send` / `Sync` 標記

`Send` / `Sync` 是 [2.13.9](213-marker.md#2139-send--sync執行緒邊界標記) 的自動標記 trait。本章的型別遵循以下原則：

- `Atomic<T>`：Send + Sync，包括 `Atomic<Ptr<T>>`（該儲存格持有的是位址）
- `Mutex<T>`：`T: Send` 時為 Send + Sync
- `RwLock<T>`：`T: Send` 時為 Send；`T: Send + Sync` 時為 Sync。其讀取者同時共用該值，因此集合型別（`!Sync`）的 `RwLock` 不能在執行緒之間共用；請使用 `Mutex<T>`。
- `MutexGuard<T>`、`RwLockReadGuard<T>`、`RwLockWriteGuard<T>`：Send、`!Sync`。守衛只在其 `using` 區塊內有效，而其 `get` 回傳的檢視既不是 `Send` 也不是 `Sync`（由 E0106 追蹤，2.16.3）
- `Frozen<T>`：`T: Send` 時為 Sync（2.16.4b）
- `FrozenList<T>`、`FrozenSet<T, S>`：元素（以及雜湊器）為 `Send + Sync` 時為 Sync；`FrozenDict<K, V, S>` 則在 `K`、`V` 和 `S` 皆如此時為 Sync（[2.9.7b](209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s唯讀集合型別)）
- `Cell<T>`：兩者皆非（僅限單一任務）。上面的 `task::spawn` 之所以是錯誤，是因為 `async` 區塊擷取了 `Cell`，因而不是 `Send`

## 2.16.6. 模組對應

每個項目都定義在 `core::sync` 中，這是它們唯一的路徑（[0.1.6](../000-meta/001-tier-model.md#016-每個項目只有一條路徑)）；沒有依型別劃分的子模組。

| 型別         | 模組            | 內部結構               |
|--------------|-------------------|----------------------------------|
| `Atomic<T>`  | `core::sync` | 原子指令（`nabi.atomic.*`） |
| `Mutex<T>`   | `core::sync` | futex / 等待佇列（依目標而定）  |
| `RwLock<T>`  | `core::sync` | futex + 讀取者計數器           |
| `Cell<T>`    | `core::sync` | 單純的包裝，無鎖           |
| `MutexGuard` | `core::sync` | 透過 Drop 達成 RAII                    |
| `RwLockReadGuard`, `RwLockWriteGuard` | `core::sync` | 透過 Drop 達成 RAII |
| `LockFuture`, `ReadFuture`, `WriteFuture` | `core::sync` | `lock_async` / `read_async` / `write_async` 的 future |
| `Once`       | `core::sync` | 狀態字組 + 等待佇列          |
| `Condvar`, `CondvarWait` | `core::sync` | 等待佇列；`CondvarWait` 是 `wait_async` 的 future |
| `Frozen<T>`  | `core::sync` | 其他人無法觸及的複本       |
| `LoadOrdering`, `StoreOrdering`, `UpdateOrdering` | `core::sync` | 記憶體順序（2.16.2） |
| `AtomicPrimitive` | `core::sync` | 封閉的標記 trait          |
