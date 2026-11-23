# 2.9. 集合型別：`List` / `Dict` / `Set` / `Tuple` / `Slice`

`core::collections` 模組提供通用的容器。所有集合型別都配置在 GC 堆積上，而 `List`、`Dict` 和 `Set` 可以直接透過以 `#[literal(...)]` 繫結的字面值語法建立。

本章定義下列集合型別的權威簽章，以及 `Deque<T>` 和 `BinaryHeap<T>`。

| 集合型別            | 字面值語法                 | 意義                            |
|---------------------|----------------------------|---------------------------------|
| `List<T>`           | `[1, 2, 3]`                | 動態陣列（插入順序） |
| `Dict<K, V, S>`     | `{"a": 1, "b": 2}`         | 雜湊對應（插入順序） |
| `Set<T, S>`         | `{1, 2, 3}`                | 雜湊集合（插入順序）      |
| `Tuple<...>`        | `(1, "a", true)`           | 固定長度、異質        |
| `Slice<T>`          | `xs[1:3]`                  | 容器的零複製檢視   |
| `SliceMut<T>`       | 預期型別為 `SliceMut` 時的 `xs[1:3]` | 串列的可寫入檢視 |
| `FrozenList<T>`、`FrozenDict<K, V, S>`、`FrozenSet<T, S>` | `xs.freeze()` | 唯讀，可在執行緒之間共用 |

## 2.9.1. 字面值語法：`#[literal(query)]` 繫結

字面值語法以 [refs-lang 6.5 `#[literal]`](../../refs-lang/006-meta-system/605-literal-magic.md) 的**查詢 DSL** 定義如下。

```nabi
#[literal("[" {expr} % "," * "]")]
fn list_literal<T>(items: ...T) -> List<T> { ... }

#[literal("{" {expr ":" expr} % "," * "}")]
fn dict_literal<K: Hash, V>(pairs: ...(K, V)) -> Dict<K, V> { ... }

#[literal("{" {expr} % "," + "}")]
fn set_literal<T: Hash>(items: ...T) -> Set<T> { ... }
```

每個處理函式都是 `core::collections` 的套件私有函式。元組 `(1, "a", true)` 和固定長度陣列由編譯器直接建立，不經過處理函式（[refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md)）。

字面值查詢語法的意義如下：`{...} % sep *` 是以 sep 分隔的零次或多次重複，`{...} % sep +` 則是一次或多次。單一元素的 Set 寫成帶尾隨逗號的 `{1,}`，以便和區塊 `{1}` 區分。在比對語彙單元序列時，詞法分析器直接區分語彙單元，再由剖析器進行查詢比對。

### 解決 Dict / Set 的歧義

```nabi
{"a": 1}      // Dict: `:` pattern
{1, 2}        // Set: values only
{1,}          // Set (single element, trailing comma)
{1}           // always a block, even when the slot is a Set
{ x = 1 }     // block expression: the token after `{` is statement-like
```

剖析器只依內容判斷：`expr ":"` 形式是 Dict，有逗號就是 Set，其餘都是區塊。接收槽位的型別不參與判斷。

### `[...]` 的型別

沒有預期型別時，`[1, 2, 3]` 是 `List<T>`。只有在預期型別寫成 `[T; N]` 時，才會建立固定長度陣列 `[T; N]`，例如 `let a: [i32; 3] = [1, 2, 3];`。固定長度陣列由編譯器直接建立，不經過處理函式。

沒有預期的元素型別時，各元素的型別以與分支相同的方式取得一致（[refs-lang 3](../../refs-lang/003-expressions/README.md)）：所有元素是同一個型別，或者某個元素的和型別包含其他所有元素的型別，取先成立者（當 `x: f64 | none` 時，`[none, x]` 和 `[x, none]` 都是 `List<f64 | none>`）。不會為不同型別的元素自行建構和型別：`[0.5, none]` 是錯誤（E0040），其說明會要求寫出串列的型別，即 `let xs: List<f64 | none> = [0.5, none];`。

重複字面值 `[v; N]` 一律是固定長度陣列。它只對 `v` 求值一次，並讓每個元素各自持有一份副本：`Copy` 或一般值的元素採逐位元複製，持有集合型別的元素則採 `v.clone()`，因此 `[List::<i32>::new(); 3]` 是三個各自獨立的串列（[refs-lang 2.11.6](../../refs-lang/002-types/211-const-generics.md#2116-參數推論)）。`[T; N]` 的 `Default` 同樣對每個元素各呼叫一次 `T::default()`。

當元素實作 `Clone` / `Copy` / `Default` / `Debug` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` 時，固定長度陣列也實作這些 trait。排序採字典序，因此可以直接使用比較運算子（`==`、`<` 等）。其方法如下：

```nabi
impl<T, const N: usize> [T; N] {
    fn len(self) -> usize;                       // N
    fn is_empty(self) -> bool;
    fn iter(self) -> SliceIter<T>;               // over a copy (arrays are values)
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn map<U>(self, f: fn(T) -> U) -> [U; N];    // a new array
    fn fill(ref self, x: T);                     // in place
}

impl<T: PartialEq, const N: usize> [T; N] {
    fn contains(self, x: T) -> bool;
}
```

`array::from_fn::<T, N>(f: fn(usize) -> T) -> [T; N]` 建立一個固定長度陣列，其第 `i` 個元素是 `f(i)`，依序呼叫；`N` 取自預期型別或型別引數。每個元素都是獨立的值，因此它可以建立 `[v; N]` 無法複製的元素所組成的陣列：未實作 `Clone` 的型別，或只有一個擁有者的資源。

```nabi
let bags = array::from_fn::<Bag, 3>(|i| Bag { xs: [i] });
let files: [File; 2] = array::from_fn(|i| try! File::open(paths[i]));
```

```nabi
let a: [i32; 3] = [1, 2, 3];
a.len();                 //→ 3
a == [1, 2, 3];          //→ true
a < [1, 2, 4];           //→ true (lexicographic)
```

### 參照控制代碼

`List`、`Dict` 和 `Set` 的值是指向 GC 堆積物件的參照控制代碼。`let ys = xs;` 和引數傳遞複製的是控制代碼，因此共用同一個集合。所以修改集合的方法接收 `self`，而不是 `ref self`。需要獨立的副本時，呼叫 `clone()`。結構元素是值：`xs[i]` 和 `d[k]` 回傳副本，修改要以 `xs[i] = v` 存回（[refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-值語意)）。`iter` / `keys` / `values` 回傳不複製集合的惰性迭代器。

## 2.9.2. `List<T>`：動態陣列

```nabi
struct List<T> {
    // internal: ptr + len + capacity
}
```

內部由一塊連續的記憶體緩衝區加上 `len` / `capacity` 組成，提供攤銷 O(1) 的 push/pop 和 O(1) 的隨機存取。

### 核心方法

```nabi
impl<T> List<T> {
    fn new() -> List<T>;
    fn with_capacity(cap: usize) -> List<T>;
    fn filled(n: usize, x: T) -> List<T>;        // n copies of x

    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;

    fn push(self, x: T);
    fn pop(self) -> T | none;
    fn insert(self, idx: usize, x: T);
    fn remove(self, idx: usize) -> T;
    fn clear(self);
    fn truncate(self, len: usize);              // keeps the first len elements; no change if fewer
    fn drain(self, r: Range<usize>) -> List<T>; // removes and returns r's elements in order; panics if r is reversed or ends past len()

    fn get(self, idx: usize) -> T | none;       // safe: out-of-bounds → none
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn swap(self, i: usize, j: usize);          // panics when either is out of bounds
    fn fill(self, x: T);                         // every element becomes x

    fn extend<I: IntoIterator<Item = T>>(self, iter: I);
    fn retain(self, keep: fn(T) -> bool);        // in place, order kept
    fn sort(self) where T: Ord;                  // ascending, stable
    fn sort_by(self, cmp: fn(T, T) -> Ordering); // stable
    fn sort_by_key<K: Ord>(self, key: fn(T) -> K); // ascending by key, stable
    fn reserve(self, additional: usize);
    fn shrink_to_fit(self);
    unsafe fn set_len(self, n: usize);           // n > capacity() panics
    fn reverse(self);                            // in place

    fn sort_unstable_by(self, cmp: fn(T, T) -> Ordering);   // not stable, in place, no extra memory; O(n log n)
    fn sort_unstable_by_key<K: Ord>(self, key: fn(T) -> K);
    fn binary_search_by(self, f: fn(T) -> Ordering) -> usize | none;          // f(x) says where x is from the target
    fn binary_search_by_key<K: Ord>(self, k: K, key: fn(T) -> K) -> usize | none;
    fn rposition(self, pred: fn(T) -> bool) -> usize | none;  // the last index pred accepts
    fn rotate_left(self, k: usize);              // the first k move to the back; k > len panics
    fn rotate_right(self, k: usize);             // the last k move to the front
    fn split_off(self, at: usize) -> List<T>;    // removes and returns [at, len); at > len panics
    fn dedup_by(self, same: fn(T, T) -> bool);   // drops x when same(kept, x)
    fn dedup_by_key<K: PartialEq>(self, key: fn(T) -> K);

    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);   // views of [0, mid) and [mid, len); mid > len panics
    fn windows(self, size: usize) -> slice::Windows<T>;   // self[:].windows(size)
    fn chunks(self, size: usize) -> slice::Chunks<T>;     // self[:].chunks(size)
    fn split_at_mut(self, mid: usize) -> (SliceMut<T>, SliceMut<T>);   // writable views of [0, mid) and [mid, len)
    fn chunks_mut(self, size: usize) -> slice::ChunksMut<T>;   // writable views of size elements, the last shorter
    fn partition_point(self, pred: fn(T) -> bool) -> usize;   // self[:].partition_point(pred)

    fn iter(self) -> ListIter<T>;
    fn into_iter(self) -> ListIntoIter<T>;
}

impl<T: PartialEq> List<T> {
    fn contains(self, x: T) -> bool;
    fn dedup(self);                              // drops each element equal to the one before it
}

impl<T: Ord> List<T> {
    fn binary_search(self, x: T) -> usize | none;   // ascending list: the first index of x, O(log n)
    fn sort_unstable(self);                          // ascending, not stable
}

impl<T> List<List<T>> {
    fn concat(self) -> List<T>;                      // every list's elements in order, one new list
}

impl<T: Send + Sync> List<T> {
    fn freeze(self) -> FrozenList<T>;                // read-only and Sync, O(1); the list is handed over (2.9.7b)
}
```

`sort_unstable` 是內省排序（introsort）：以三數取中為樞紐的快速排序，元素少於 20 個時改用插入排序，深度超過 2 log2(n) 時改用堆積排序，因此在任何情況下都是 O(n log n)，且不配置任何記憶體；相等的元素可能改變順序。`sort`（合併排序）是穩定排序，使用 n 個額外的槽位。`_by` 系列搜尋採用與 `sort_by` 相同的 `Ordering` 慣例：`binary_search_by(|x| x.cmp(target))` 在 `f` 視為遞增的串列中，找出第一個 `f` 判為 `Equal` 的元素。

依位置讀取的操作有 `T | none` 形式（`get`、`first`、`last`、`pop`）；其他依位置的操作在超出範圍時一律觸發 panic（`xs[i]`、`xs[i] = v`、`insert`、`remove`、`swap`、`drain`、`split_at`）。沒有 `set`：寫入就是 `xs[i] = v`，要略過超出範圍的索引時，先檢查再寫入（`if i < xs.len() { xs[i] = v }`）。

`set_len(n)` 設定長度而不寫入元素，用於就地填入的緩衝區（檢視的 `raw_parts`、執行環境呼叫）。它是 `unsafe`：超過舊長度的元素保有儲存空間裡原有的內容（從未寫入或已騰空的槽位為零），不一定是有效的 `T`；呼叫者必須在讀取前寫入它們。以此方式縮短時，被截去的元素仍留在緩衝區中。`n` 超過容量時觸發 panic；請先 `reserve`。

`partition_point(pred)` 是第一個不滿足 `pred` 的元素索引；`pred` 必須對串列的某個前綴成立，對其餘部分不成立；O(log n)。`x` 在遞增串列中的插入點是 `xs.partition_point(|e| e < x)`，依鍵則是 `xs.partition_point(|e| key(e) < k)`。`binary_search(x)` 回傳第一個等於 `x` 的元素索引（下界），或 `none`。

```nabi
impl List<str> {
    fn join(self, sep: str) -> str;              // joins the elements with sep
}

impl List<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // self[:].find(needle)
}
```

### 範例

```nabi
let xs: List<i32> = [];
xs.push(1);
xs.push(2);
xs.push(3);

let ys: List<i32> = [10, 20, 30];
ys[0];                    //→ 10  (Index trait)
ys[100];                  //→ panic: index 100 out of bounds for length 3
ys.get(100);              //→ none

for x in ys {
    process(x);               //→ 10, 20, 30
}

let total: i32 = ys.into_iter().sum();   //→ 60

let zs: List<i32> = [3, 1, 1, 2, 2, 4];
zs.dedup();                                  //→ [3, 1, 2, 4]
zs.retain(|x| x != 3);                       //→ [1, 2, 4]
zs.binary_search(4);                         //→ 2
zs.partition_point(|x| x < 3);               //→ 2 (where 3 would go)
let (a, b) = zs.split_at(1);                 //→ views of [1] and [2, 4]
```

### trait 實作

| trait                 | 條件                | 意義                            |
|-----------------------|---------------------|---------------------------------|
| `IntoIterator`        | 一律                | `for x in xs`                   |
| `FromIterator<T>`     | 一律                | `(0..5).collect::<List<i32>>()` |
| `Index<usize>`        | 一律                | `xs[i]`                         |
| `IndexSet<usize, T>`  | 一律                | `xs[i] = v`                    |
| `AddAssign<T>`        | 一律                | `xs += x` 推入 `x`            |
| `AddAssign<List<T>>`  | 一律                | `xs += ys` 附加 `ys` 的每個元素 |
| `Sliceable<Range<usize>>` | 一律            | `xs[1:3]`                       |
| `Display`             | **否**              | **建議自行輸出**   |
| `Debug`               | `T: Debug`          | `f"{xs:?}"` → `[1, 2, 3]`       |
| `Default`             | 一律                | `List::new()`                   |
| `Clone`               | `T: Clone`          | 深層複製                       |
| `PartialEq`           | `T: PartialEq`      | 逐元素                    |
| `PartialOrd`          | `T: PartialOrd`     | 字典序；前綴排在前面 |
| `Eq`/`Ord`/`Hash`     | T 實作相同的 trait 時 | 字典序             |

### 比較 / 雜湊 / Debug：共同規則

- `List<T>` 依序逐元素實作 `PartialEq` / `Eq` / `Hash`。
- `Dict<K, V, S>` 和 `Set<T, S>` 的比較與雜湊與順序無關。持有相同元素（鍵和值）時即相等。每個項目各自以行程金鑰雜湊，再將結果相加（[2.5.6](205-cmp.md#256-hash雜湊eq-一致性承諾)），因此雜湊值也不取決於 `S`。
- 這三種集合型別在元素實作 `Debug` 時都實作 `Debug`，並依迭代順序輸出。
- 因此，元素為 `Eq + Hash` 的集合型別可以作為 `Set` 的元素或 `Dict` 的鍵（`Set<List<i32>>`）。

### 容量與移除：共同規則

- 索引超出範圍時觸發 panic，並指出索引和長度：`index 100 out of bounds for length 3`。方法會先寫出自己的名稱（`List::remove: index 5 out of bounds for length 3`、`List::drain: range 2..4 out of bounds for length 3`），panic 指向呼叫者。
- `reserve`、`with_capacity`、`repeat` 或擴充時的大小溢位，在任何最佳化等級下都會以 "capacity overflow" 觸發 panic。
- 移除元素（`pop`、`remove`、`clear`、`truncate`、`drain` 等）會釋放參考：GC 不再透過該集合看到它。

## 2.9.3. `Dict<K, V, S>`：雜湊對應

```nabi
struct Dict<K: Hash + Eq, V, S = RandomState> {
    // internal: compact entries in insertion order, plus an index of
    // linear-probed slots (none for a small table)
}
```

它提供平均 O(1) 的查詢 / 插入 / 移除。迭代依插入順序進行：已存在的鍵再次插入時保有原本的位置，移除後再插入的鍵則排到最後。

`S` 負責建立雜湊器（[2.5.11](205-cmp.md#2511-corehash雜湊器)）。預設的 `RandomState` 使用每個行程只抽取一次的金鑰，因此鍵在表中的位置會隨每次執行而改變；迭代順序則不會。在常數求值期間建立的字典，會在執行時第一次使用時以行程金鑰重新雜湊。`Dict<K, V>` 就是 `Dict<K, V, RandomState>`，每個 `{k: v}` 字面值也是。

### 核心方法

```nabi
impl<K: Hash + Eq, V> Dict<K, V> {
    fn new() -> Dict<K, V>;
    fn with_capacity(cap: usize) -> Dict<K, V>;
}

impl<K: Hash + Eq, V, S: BuildHasher> Dict<K, V, S> {
    fn with_hasher(s: S) -> Dict<K, V, S>;
    fn with_capacity_and_hasher(cap: usize, s: S) -> Dict<K, V, S>;
    fn hasher(self) -> S;

    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, k: K, v: V) -> V | none;       // returns the previous V / none if absent
    fn remove(self, k: K) -> V | none;
    fn remove_entry(self, k: K) -> (K, V) | none;  // the stored key too
    fn pop_last(self) -> (K, V) | none;            // removes the entry inserted last; Python d.popitem()
    fn pop_first(self) -> (K, V) | none;           // removes the entry inserted first
    fn drain(self) -> List<(K, V)>;                // removes every entry, returned in insertion order
    fn get(self, k: K) -> V | none;
    fn get_or(self, k: K, default: V) -> V;        // `default` when absent; no insert
    fn get_or_insert(self, k: K, default: V) -> V; // inserts `default` when absent
    fn contains_key(self, k: K) -> bool;
    fn clear(self);
    fn retain(self, keep: fn(K, V) -> bool);      // in place
    fn extend<I: IntoIterator<Item = (K, V)>>(self, pairs: I);   // a later pair's value wins

    fn iter(self) -> DictIter<K, V, S>;              // (K, V) pairs
    fn keys(self) -> Keys<K, V, S>;
    fn values(self) -> Values<K, V, S>;

    fn entry(self, k: K) -> Entry<K, V, S>;       // the slot for that key (present or not)
}

struct Entry<K, V, S = RandomState> { /* dict + key */ }

impl<K: Hash + Eq, V, S: BuildHasher> Entry<K, V, S> {
    fn key(self) -> K;
    fn or_insert(self, v: V) -> V;                   // inserts v if absent; returns the value
    fn or_insert_with(self, f: fn() -> V) -> V;      // inserts f() if absent; returns the value
    fn and_modify(self, f: fn(V) -> V) -> Entry<K, V, S>;   // if present, replaces it with f(value)
}

impl<K: Hash + Eq, V: Default, S: BuildHasher> Entry<K, V, S> {
    fn or_default(self) -> V;                         // inserts V::default() if absent; returns the value
}
```

`d.freeze()`（`K`、`V` 和 `S` 皆為 `Send + Sync`）把字典交接給 `FrozenDict<K, V, S>`，讓多個執行緒不需鎖即可共用（[2.9.7b](#297b-frozenlistt--frozendictk-v-s--frozensett-s唯讀集合型別)）。

`Dict` 實作 `IntoIterator`、`Index<K>`、`IndexSet<K, V>`、`AddAssign<Dict<K, V, S>>`、`PartialEq` / `Eq` / `Hash`（依 `V`）、`Debug`（`K: Debug, V: Debug`）、`Clone`（`K: Clone, V: Clone`；複製鍵和值，使用相同的雜湊器）、`Default`、`FromIterator<(K, V)>` 和 `From<List<(K, V)>>`。`Default`、`FromIterator` 和 `From<List<(K, V)>>` 需要 `S: BuildHasher + Default`；其他實作接受任何 `S: BuildHasher`。

```nabi
use core::hash::BuildHasherDefault;

let fixed: Dict<str, i32, BuildHasherDefault<Fnv>> = Dict::with_hasher(BuildHasherDefault::new());
```

```nabi
let counts: Dict<str, i32> = Dict::new();   // `{}` is an empty block, not an empty Dict
for w in words {
    counts.entry(w).and_modify(|n| n + 1).or_insert(1);
}
```

### 範例

```nabi
let d: Dict<str, i32> = {"a": 1, "b": 2};
d["a"];                   //→ 1
d["c"];                   //→ panic: key "c" not found; use `d.get(k)` to get `i32 | none`
d.get("c");               //→ none

d.insert("c", 3);
d.contains_key("c");      //→ true

for (k, v) in d.iter() {
    log("{k} = {v}");
}
```

### `Index<K>`

```nabi
impl<K: Hash + Eq, V> Index<K> for Dict<K, V> {
    type Output = V;
    fn index(self, k: K) -> V {
        self.get(k) ?? panic(f"key {k:?} not found; use `d.get(k)` to get `V | none`")
    }
}
```

`d[k]` 寫法假定鍵存在；需要安全存取時，請用 `.get(k)`。panic 訊息以 `{k:?}` 的方式顯示鍵，`K` 沒有 `Debug` 時則顯示為 `<K>`（型別名稱），`V` 則顯示為值型別的名稱。

`d += e`（`AddAssign<Dict<K, V>>`）把 `e` 的每個項目就地插入 `d`；鍵相同時，以 `e` 的值為準。

`Dict::from(pairs)`（`From<List<(K, V)>>`）從配對串列建立字典；鍵重複時，以較後面的值為準。`K` 和 `V` 取自引數，因此 `let d = Dict::from([("a", 1)]);` 不需要型別標註。

### 惰性檢視

`iter` / `keys` / `values` 是目前表格的惰性檢視，而不是副本。在迭代期間修改字典會得到未指定的結果（元素可能被遺漏或出現兩次），但記憶體安全性仍然維持。

## 2.9.4. `Set<T, S>`：雜湊集合

```nabi
struct Set<T: Hash + Eq, S = RandomState> {
    // internal: Dict<T, bool, S>
}
```

它和 `Dict` 一樣依插入順序迭代；平均效能為 O(1)。`S` 和 `Dict` 一樣負責建立雜湊器。`s.freeze()`（`T` 和 `S` 皆為 `Send + Sync`）把集合交接給 `FrozenSet<T, S>`（[2.9.7b](#297b-frozenlistt--frozendictk-v-s--frozensett-s唯讀集合型別)）。

```nabi
impl<T: Hash + Eq> Set<T> {
    fn new() -> Set<T>;
}

impl<T: Hash + Eq, S: BuildHasher> Set<T, S> {
    fn with_hasher(s: S) -> Set<T, S>;
    fn with_capacity_and_hasher(cap: usize, s: S) -> Set<T, S>;
    fn hasher(self) -> S;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;

    fn insert(self, x: T) -> bool;        // true if newly inserted
    fn remove(self, x: T) -> bool;
    fn pop(self) -> T | none;             // removes the member inserted last
    fn drain(self) -> List<T>;            // removes every member, returned in insertion order
    fn contains(self, x: T) -> bool;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);         // in place
    fn extend<I: IntoIterator<Item = T>>(self, items: I);

    fn union(self, other: Set<T, S>) -> Set<T, S>;
    fn intersection(self, other: Set<T, S>) -> Set<T, S>;
    fn difference(self, other: Set<T, S>) -> Set<T, S>;
    fn symmetric_difference(self, other: Set<T, S>) -> Set<T, S>;

    fn is_subset(self, other: Set<T, S>) -> bool;
    fn is_superset(self, other: Set<T, S>) -> bool;
    fn is_disjoint(self, other: Set<T, S>) -> bool;

    fn iter(self) -> SetIter<T, S>;
    fn to_list(self) -> List<T>;
}
```

`Set` 實作 `IntoIterator`、`AddAssign<T>`（插入）、`PartialEq` / `Eq` / `Hash`、`Debug`（`T: Debug`）、`Clone`（`T: Clone`；複製元素，使用相同的雜湊器）、`Default`、`FromIterator<T>` 和 `From<List<T>>`；後三者需要 `S: BuildHasher + Default`。

```nabi
let s: Set<i32> = {1, 2, 3};
s.contains(2);            //→ true
s.insert(4);

let a: Set<i32> = {1, 2, 3};
let b: Set<i32> = {2, 3, 4};
let u = a.union(b);       //→ {1, 2, 3, 4}
let i = a.intersection(b);//→ {2, 3}
```

`Set::from(xs)`（`From<List<T>>`）從串列建立集合，並去除重複；`T` 取自引數（`let s = Set::from([1, 2, 2]);`）。`s += x`（`AddAssign<T>`）插入 `x`。

## 2.9.5. 插入順序

`Dict` 和 `Set` 依插入順序迭代，如同 Python 3.7+ 的 `dict` 或 Java 的 `LinkedHashMap`，因此沒有另外的有序型別。`iter`、`keys`、`values`、`to_list`、`Debug` 和 `for` 都遵循這個順序：

- 已存在的鍵再次插入時保有原本的位置（只有值改變）；
- 移除後再插入的鍵排到最後；
- `retain` 保留存留項目的順序，`extend` 和 `d += e` 依序插入（新鍵排到最後）；
- `pop_last` / `Set::pop` 取出最新的項目，`pop_first` 取出最舊的項目，平均皆為 O(1)（它們留下的空洞在數量超過項目數時會被一次擠除）；
- 集合運算的結果先依左運算元的順序排列。

相等性和雜湊忽略順序（[2.9.2](#292-listt動態陣列)），因此 `{"a": 1, "b": 2} == {"b": 2, "a": 1}`。

## 2.9.5a. `Deque<T>`：雙端佇列

建構在環形緩衝區上的雙端佇列。

```nabi
impl<T> Deque<T> {
    fn new() -> Deque<T>;
    fn with_capacity(cap: usize) -> Deque<T>;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;
    fn push_back(self, x: T);
    fn push_front(self, x: T);
    fn pop_front(self) -> T | none;
    fn pop_back(self) -> T | none;
    fn get(self, i: usize) -> T | none;
    fn front(self) -> T | none;
    fn back(self) -> T | none;
    fn clear(self);
    fn iter(self) -> DequeIter<T>;             // front to back
}
```

沒有 `set`：寫入就是 `d[i] = v`。`capacity` 是環形緩衝區在擴充前能容納的元素數。它實作 `d[i]` / `d[i] = v`（超出範圍時和 `List` 一樣觸發 panic：`index 5 out of bounds for length 3`）、`IntoIterator`、`FromIterator`（從前到後）、`Default`、`Clone`（`T: Clone`）、`PartialEq` / `Eq`（逐元素，從前到後）和 `Debug`。`d += x`（`AddAssign<T>`）把 `x` 推入尾端。

## 2.9.5b. `BinaryHeap<T>`：優先佇列

```nabi
impl<T: Ord> BinaryHeap<T> {
    fn new() -> BinaryHeap<T>;                 // max-heap: largest first
}
impl<T> BinaryHeap<T> {
    fn by_ordering(cmp: fn(T, T) -> Ordering + Send) -> BinaryHeap<T>; // the greatest by cmp first, as new() by Ord::cmp
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn peek(self) -> T | none;
    fn push(self, x: T);
    fn pop(self) -> T | none;
    fn clear(self);
    fn retain(self, keep: fn(T) -> bool);
    fn iter(self) -> HeapIter<T>;              // unordered
}
```

`retain` 就地篩選並恢復堆積順序，O(n)。`HeapIter<T>` 是 `ExactSizeIterator`。堆積實作 `IntoIterator`（無序，同 `iter`）、`Debug`（儲存順序，頂端在前）和 `Clone`（`T: Clone`；副本保有堆積的順序）；在 `T: Ord` 時，`Default` 和 `FromIterator` 和 `new` 一樣建立最大堆積。

比較器和 `List::sort_by` 的一樣回傳 `Ordering`：`by_ordering(|a, b| b.cmp(a))` 會先彈出最小者。

比較器是 `fn .. + Send`，因此 `T` 是 `Send` 時堆積也是 `Send`，可以透過通道傳送或放進 `Mutex`。擷取非 `Send + Sync` 值（例如集合型別）的比較器是編譯錯誤，其說明會要求去掉該擷取：在閉包內計算該值，或改為擷取陣列。

## 2.9.5c. `SortedDict<K, V>` / `SortedSet<T>`：有序對應與集合

`core::collections::{SortedDict, SortedSet}` 以 B 樹依鍵的順序（`K: Ord`）保存項目：查詢、插入和移除皆為 O(log n)，迭代、`range`、`first` / `last` 和 `floor` / `ceil` 都遵循此順序。它們不在 prelude 中：`use core::collections::SortedDict;`。

```nabi
impl<K: Ord, V> SortedDict<K, V> {
    fn new() -> SortedDict<K, V>;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn insert(self, k: K, v: V) -> V | none;       // the previous value / none if new
    fn remove(self, k: K) -> V | none;
    fn get(self, k: K) -> V | none;
    fn get_or(self, k: K, default: V) -> V;
    fn contains_key(self, k: K) -> bool;
    fn clear(self);

    fn first(self) -> (K, V) | none;               // the smallest key's entry
    fn last(self) -> (K, V) | none;                // the largest key's entry
    fn pop_first(self) -> (K, V) | none;
    fn pop_last(self) -> (K, V) | none;
    fn floor(self, k: K) -> (K, V) | none;         // the largest key <= k
    fn ceil(self, k: K) -> (K, V) | none;          // the smallest key >= k
    fn lower(self, k: K) -> (K, V) | none;         // the largest key < k
    fn higher(self, k: K) -> (K, V) | none;        // the smallest key > k

    fn iter(self) -> SortedIter<K, V>;             // (K, V) in key order
    fn iter_rev(self) -> SortedIter<K, V>;         // descending
    fn keys(self) -> SortedKeys<K, V>;
    fn values(self) -> SortedValues<K, V>;
    fn range<R: KeyRange<K>>(self, r: R) -> SortedIter<K, V>;      // keys in r, ascending
    fn range_rev<R: KeyRange<K>>(self, r: R) -> SortedIter<K, V>;  // keys in r, descending
}

pub trait KeyRange<K> {                            // a..b, a..=b, a.., ..b, ..=b, ..
    fn lower(self) -> (K, bool) | none;            // the bound and whether it is included; none when open
    fn upper(self) -> (K, bool) | none;
}
```

`SortedSet<T>`（內部為 `SortedDict<T, bool>`）具有 `new`、`len`、`is_empty`、`insert(x) -> bool`（新插入時為 true）、`remove(x) -> bool`、`contains`、`clear`、`first`、`last`、`pop_first`、`pop_last`、`floor`、`ceil`、`lower`、`higher`（皆為 `T | none`）、`iter`、`iter_rev`、`range`、`range_rev`、`to_list`，以及 `Set` 的集合運算（`union`、`intersection`、`difference`、`symmetric_difference`、`is_subset`、`is_superset`、`is_disjoint`），其結果為有序集合。

`range` 接受所有範圍形式（[refs-lang 3.7](../../refs-lang/003-expressions/307-range-expressions.md)）；以 `..` 開頭的形式作為引數時要加括號（`d.range((..10))`）。找到第一個項目為 O(log n)，之後每一個平均為 O(1)。空的或反向的範圍不產生任何項目。

兩者都實作 `IntoIterator`（鍵的順序）、`FromIterator`、`From<List<..>>`（以較後面的配對的值為準）、`Default`、`Clone`（`K: Clone, V: Clone`；複製項目）、`PartialEq` / `Eq`（相同的項目；順序即鍵的順序）、`Hash`（先長度，再依序雜湊每個項目）和 `Debug`（依序輸出 `{k: v, ..}` / `{a, b, ..}`）。`SortedDict` 實作 `d[k]`（不存在時觸發 panic：`key 7 not found; use `d.get(k)` to get `V | none``）和 `d[k] = v`；`SortedSet` 實作 `s += x`。副本共用同一個集合，而且兩者都和 `Dict` 一樣是 `!Sync`。

在迭代器走訪有序字典或集合期間修改它，會在迭代器的下一步觸發 panic，並提示先收集鍵（`SortedDict changed while it was walked; ..`）；替換已存在鍵的值不算修改（`for (k, v) in d { d[k] = v + 1; }` 沒有問題）。

```nabi
use core::collections::SortedDict;

let events: SortedDict<i64, str> = SortedDict::new();
events[1700] = "boot";
events[1650] = "power";
events[1720] = "login";
for (t, what) in events.range(1600..1710) {   // 1650, 1700 in time order
    println(f"{t} {what}");
}
let before = events.floor(1710);              //→ (1700, "boot")
let latest = events.last();                   //→ (1720, "login")
```

**實作**。最小度數為 16：每個節點持有 15 到 31 個鍵（根節點可以更少），節點依節點編號存放在扁平的串列中（鍵、值和子節點編號各占一個串列），因此每次查詢在每一層只觸及一個串列槽位。插入在向下途中分裂已滿的節點；移除（CLRS）在向下之前，從兄弟節點補足鍵少於 16 個的子節點，或合併兩個子節點。騰空的槽位會被清零，因此已移除的項目不會讓任何東西繼續存活。

## 2.9.6. `Tuple<...>`：固定長度、異質

```nabi
struct Tuple { /* heterogeneous, determined at compile time */ }
```

元組是語言內建的，不能註冊為字面值處理函式（[refs-lang 6.5.7](../../refs-lang/006-meta-system/605-literal-magic.md#657-使用者定義的字面值)）。元組運算式必須至少含有一個逗號。`(e)` 是分組運算式；單一元素的元組是 `(e,)`。

每個位置可以有不同的型別，長度在編譯時固定。

```nabi
let t: Tuple<i32, str, bool> = (42, "hello", true);
let n = t.0;              //→ 42
let s = t.1;              //→ "hello"
let b = t.2;              //→ true

// pattern destructuring
let (n, s, b) = t;
```

### trait 實作

| trait        | 條件                          |
|--------------|-------------------------------|
| `Clone`      | 每個成分皆為 `Clone`       |
| `Copy`       | 每個成分皆為 `Copy`        |
| `Debug`      | 每個成分皆為 `Debug`       |
| `PartialEq`  | 每個成分皆為 `PartialEq`   |
| `Eq`         | 每個成分皆為 `Eq`          |
| `PartialOrd` | 每個成分皆為 `PartialOrd`  |
| `Ord`        | 每個成分皆為 `Ord`         |
| `Hash`       | 每個成分皆為 `Hash`        |

表中的 trait 由 `core::tuple` 為 1 到 12 個元素的元組提供。`PartialOrd` / `Ord` 採字典序（從第一個元素開始，由第一個不同的元素決定），因此可以直接對元組使用 `<` / `<=` / `>` / `>=`（`(1, "b") < (1, "c")`）。

`Display` 不會自動衍生；建議在需要的地方直接實作。

### 多重回傳值

[refs-lang 4.3.5 多重回傳](../../refs-lang/004-functions/403-return.md#435-多個回傳值元組)的多重回傳以回傳 Tuple 的方式處理。

```nabi
fn divmod(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

let (q, r) = divmod(17, 5);     //→ q = 3, r = 2
```

## 2.9.7. `Slice<T>`：檢視

```nabi
struct Slice<T> {
    // GC buffer + start position + length + stride (1 = contiguous, k = step, negative = reversed)
}
```

它是 `xs[1:3]` / `xs[::2]` / `xs[1:]` 等運算式的結果（[2.3.4 `Sliceable`](203-ops.md#234-索引-trait)）。檢視不複製元素；它指向原物件的 GC 緩衝區，只要檢視存活，該緩衝區就保持存活。

| 來源       | 檢視看到的內容                                                                  |
|------------|---------------------------------------------------------------------------------|
| `List<T>`  | 串列目前的緩衝區。元素的寫入可見；串列擴充並搬移緩衝區之後的變更則不可見 |
| `Slice<T>` | 同一個緩衝區（檢視的檢視）                                              |
| `[T; N]`   | 陣列的副本（陣列是值）                                         |

### 核心方法

```nabi
impl<T> Slice<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn step(self, k: isize) -> Slice<T>;     // every k-th; from the end if k < 0. k == 0 panics
    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);   // [0, mid) and [mid, len); mid > len panics
    fn windows(self, size: usize) -> slice::Windows<T>;   // overlapping views of size elements; size == 0 panics
    fn chunks(self, size: usize) -> slice::Chunks<T>;     // views of size elements, the last shorter; size == 0 panics
    fn iter(self) -> SliceIter<T>;
    fn to_list(self) -> List<T>;             // allocates a new List
    unsafe fn raw_parts(self) -> (GcBuf<T>, usize, usize);   // (buffer, start, len); a strided view is copied first
    unsafe fn from_raw_parts(buf: GcBuf<T>, start: usize, len: usize) -> Slice<T>;   // len elements of buf from start
}

impl<T> Slice<T> {
    fn partition_point(self, pred: fn(T) -> bool) -> usize;   // the first index whose element fails pred, O(log n)
}

impl<T: PartialEq> Slice<T> {
    fn contains(self, x: T) -> bool;
}

impl<T: Ord> Slice<T> {
    fn binary_search(self, x: T) -> usize | none;   // ascending view: the first index of x, O(log n)
}

impl Slice<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // offset of the first occurrence; an empty needle is at 0
}

impl<T> Index<usize> for Slice<T> { ... }               // s[i]: panics when out of range (`slice index 5 out of bounds for length 3`)
impl<T> Sliceable<Range<usize>> for Slice<T> { ... }    // s[a:b]: a view of a view; the range is clamped to the length
impl<T> IntoIterator for Slice<T> { ... }
impl<T: PartialEq> PartialEq for Slice<T> { ... }
impl<T: Debug> Debug for Slice<T> { ... }
```

`raw_parts` 把所檢視的元素以緩衝區中一段連續的區間提供，用於對某個區域進行的執行環境呼叫。有步幅的檢視（`xs[::2]`）會先複製到新的緩衝區，因此透過該區間的寫入不會反映到檢視或其串列。`from_raw_parts(buf, start, len)` 是其反向操作。兩者都是 `unsafe`：不檢查 `start + len` 是否位於 `buf` 之內，也不檢查其元素是否已初始化。

`slice::Windows<T>` 和 `slice::Chunks<T>` 是 `Item = Slice<T>` 的 `ExactSizeIterator`：每個視窗或區塊都是同一個緩衝區的檢視，而不是副本。`Iterator::windows` / `chunks`（[2.7.1](207-iter.md#271-iterator步進)）則改為收集成串列，供沒有緩衝區的迭代器使用。

`find` 的搜尋目標可以是任何 `AsBytes` 型別（[2.11.13](211-conversion.md#21113-asbytes-trait位元組作為檢視)）：

```nabi
trait AsBytes {
    fn byte_slice(self) -> Slice<u8>;      // the bytes as a view, no copy
}
```

它為 `str`（其 UTF-8 位元組）、`List<u8>`、`Slice<u8>` 和 `[u8; N]` 實作，因此位元組字串可以作為搜尋目標傳入（依其 `Slice<u8>` 規則，[refs-lang 1.7.5](../../refs-lang/001-lexical/107-string-literals.md#175-位元組字串)）。

`Slice` 是唯讀的（沒有 `IndexSet`）；可寫入串列的檢視是 `SliceMut<T>`（[2.9.7a](#297a-slicemutt可寫入檢視)）。負索引從尾端算起（[refs-lang 3.6.1](../../refs-lang/003-expressions/306-indexing-and-slicing.md#361-索引arri)）。

### 範例

```nabi
let xs: List<i32> = [1, 2, 3, 4, 5];
let s1: Slice<i32> = xs[1:4];           //→ view of [2, 3, 4]
let s2: Slice<i32> = xs[::2];           //→ view of [1, 3, 5]
let s3: Slice<i32> = xs[..];            //→ view of all

s1.len();                               //→ 3
let t: i32 = s1.iter().sum();           //→ 9
xs[2] = 30;
s1[1];                                  //→ 30 (the list's element write is visible)

let owned: List<i32> = s1.to_list();    //→ [2, 30, 4] (new List)
```

### 與可變引數的整合

`fn f(xs: ...T)` 的 `xs` 是 `Slice<T>` 檢視（[refs-lang 4.2.5](../../refs-lang/004-functions/402-parameters.md#425-可變參數t)）。個別引數和串列展開（`f(..ys)`）都以相同的形式接收。

```nabi
fn sum(xs: ...i32) -> i32 {
    let total = 0;
    for x in xs {            // xs: Slice<i32>
        total = total + x;   // reassignment without `let`
    }
    total
}

sum(1, 2, 3);                //→ 6
sum(..[4, 5]);               //→ 9 (list spread)
```

## 2.9.7a. `SliceMut<T>`：可寫入檢視

```nabi
struct SliceMut<T> {
    // the list's buffer + start + length, and the list with its version when the view was made
}
```

`SliceMut<T>` 是可寫入其所檢視串列的連續檢視，類似 Rust 的 `&mut [T]`。填入或重新排列串列某一部分的函式接收它（`fn zero(buf: SliceMut<u8>)`）。它在預期型別為 `SliceMut<T>` 的地方建立（引數、有型別標註的 `let`、回傳值），也由 `split_at_mut` 和 `chunks_mut` 建立。它在 prelude 中。

| 該處寫的是                      | 檢視                                                                     |
|---------------------------------|--------------------------------------------------------------------------|
| `xs`（`List<T>`）               | 整個串列（[refs-lang 2.14.6](../../refs-lang/002-types/214-cast-and-conversion.md#2146-依上下文的自動轉換)） |
| `xs[a:b]`、`xs[a..b]`、`xs[..]` | 串列的該範圍，邊界和 `Slice` 一樣被夾限（[refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-切片arrstartend)） |
| `s[a:b]`（`SliceMut<T>`）       | 該檢視的可寫入檢視                                              |

在其他任何地方，`xs[a:b]` 仍是唯讀的 `Slice<T>`。有步長的範圍（`xs[a:b:k]`）永遠不是 `SliceMut`：它不連續。

### 核心方法

```nabi
impl<T> SliceMut<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn fill(self, x: T);
    fn swap(self, i: usize, j: usize);                 // out of range panics
    fn reverse(self);
    fn copy_from_slice(self, src: Slice<T>);           // src.len() != len panics; src may overlap the view
    fn split_at(self, mid: usize) -> (SliceMut<T>, SliceMut<T>);   // [0, mid) and [mid, len), both writable; mid > len panics
    fn chunks_mut(self, size: usize) -> ChunksMut<T>;  // writable views of size elements in order, the last shorter; size 0 panics
    fn sort_by(self, cmp: fn(T, T) -> Ordering);       // stable
    fn sort_by_key<K: Ord>(self, key: fn(T) -> K);     // stable
    fn as_slice(self) -> Slice<T>;                     // the same elements, read-only
    fn iter(self) -> SliceMutIter<T>;
    fn to_list(self) -> List<T>;                       // allocates a new List
    unsafe fn raw_parts(self) -> (GcBuf<T>, usize, usize);   // (buffer, start, len), for runtime calls that fill a region
}

impl<T: Ord> SliceMut<T> {
    fn sort(self);                                     // ascending, stable
}

impl<T: PartialEq> SliceMut<T> {
    fn contains(self, x: T) -> bool;
}

impl<T> Index<usize> for SliceMut<T> { ... }              // s[i]: panics when out of range
impl<T> IndexSet<usize, T> for SliceMut<T> { ... }           // s[i] = x writes element i of the list
impl<T> Sliceable<Range<usize>> for SliceMut<T> { ... }   // s[a:b]: a writable view of the view; the range is clamped
impl<T> IntoIterator for SliceMut<T> { ... }
impl<T: PartialEq> PartialEq for SliceMut<T> { ... }
impl<T: Debug> Debug for SliceMut<T> { ... }
```

`SliceMutIter<T>` 是 `Item = T` 的 `DoubleEndedIterator` 和 `ExactSizeIterator`；`ChunksMut<T>` 是 `Item = SliceMut<T>` 的 `ExactSizeIterator`。和 `List` 一樣，`SliceMut` 不是 `Sync`（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）；交接時會複製其串列，並讓檢視指向該副本。`chunks_mut` 和 `split_at_mut` 的檢視彼此不重疊，因此 `par::for_each` 可以把每一個交給各自的工作者，由它就地寫入串列中自己的部分（[2.19.1](219-par.md#2191-核心-api)）。負索引和串列一樣從尾端算起。

### 檢視存活期間調整串列大小

調整大小可能把串列的元素搬到新的緩衝區，因此串列調整大小之後，不得再使用其檢視。調整大小是指任何改變長度或緩衝區的 `List` 方法：`push`、`pop`、`insert`、`remove`、`clear`、`truncate`、`drain`、`set_len`、`reserve`、`shrink_to_fit`、`extend`、`retain`、`dedup`、`freeze` 和 `+=`。元素的寫入（`xs[i] = x`、`fill`、`swap`、`sort`、`reverse`）不是調整大小。

- 當同一個函式中同時出現兩者時，在串列調整大小後使用其檢視是編譯錯誤 `E0137`（[refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-診斷--錯誤訊息)）。檢視會透過串列的副本（`let ys = xs`）、檢視的檢視（`s[1:]`、`split_at`）、走訪它的 `for` 迴圈、分支以及迴圈的下一輪被追蹤。重新指派檢視（`s = xs[..]`）會使它再次可用，而位於離開路徑（`return`、`break`）上的調整大小不會影響其後的程式碼。
- 其他所有情況都在執行時檢查：被呼叫函式內的調整大小、存放在結構中的檢視、在呼叫者調整大小時等待的 `async fn`。串列帶有一個版本號，每次調整大小都會遞增，而在遞增之前建立的檢視，其方法會觸發 panic（`SliceMut used after its list was resized (push, insert, clear, ...): make the view again after the resize`）；走訪它的 `for` 或 `for ref` 迴圈的每一步、它的交接，以及項目中含有它的 `par` 呼叫也是如此。元素的讀取和寫入（`s[i]`、`s[i] = x`、`get`、`first`、`last`）不做任何檢查，因此走訪檢視的迴圈成本與走訪串列相同：對於過時的檢視，它們觸及的是建立檢視時的儲存空間。若調整大小後保留了該儲存空間，它仍屬於串列，否則就不再屬於串列；其他執行緒不會觸及它（`freeze` 會複製曾有檢視的串列）。

```nabi
let xs: List<i32> = [1, 2, 3];
let s: SliceMut<i32> = xs[1:];
xs.push(4);
s[0] = 9;          // error[E0137]: `s` views `xs`, which `xs.push(..)` resized at line 3
```

執行時檢查正是讓 `SliceMut` 能作為 `async fn` 的引數並跨越 `await` 存活的原因。

### 範例

```nabi
fn zero(buf: SliceMut<u8>) {
    buf.fill(0);
}

let xs: List<i32> = [5, 4, 3, 2, 1];
let mid: SliceMut<i32> = xs[1:4];
mid.sort();                             // xs is [5, 2, 3, 4, 1]
mid[0] = 20;                            // xs is [5, 20, 3, 4, 1]
let (a, b) = mid.split_at(1);
b.fill(0);                              // xs is [5, 20, 0, 0, 1]

let bytes: List<u8> = [1, 2, 3];
zero(bytes[1:]);                        // [1, 0, 0]
zero(bytes);                            // the whole list: [0, 0, 0]
```

## 2.9.7b. `FrozenList<T>` / `FrozenDict<K, V, S>` / `FrozenSet<T, S>`：唯讀集合型別

只建立一次、之後只會被讀取的集合型別（查詢表、系統讀取的元件陣列、伺服器據以回應的對應）會被凍結：`xs.freeze()` 把 `List`、`Dict` 或 `Set` 轉為任何東西都無法修改的集合型別，所有執行緒都能不加鎖、不複製地讀取它。

```nabi
struct FrozenList<T> { /* the list's storage, never changed */ }
struct FrozenDict<K, V, S = RandomState> { /* ... */ }
struct FrozenSet<T, S = RandomState> { /* ... */ }

impl<T> FrozenList<T> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, i: usize) -> T | none;
    fn first(self) -> T | none;
    fn last(self) -> T | none;
    fn iter(self) -> ListIter<T>;
    fn split_at(self, mid: usize) -> (Slice<T>, Slice<T>);
    fn windows(self, size: usize) -> slice::Windows<T>;
    fn chunks(self, size: usize) -> slice::Chunks<T>;
    fn to_list(self) -> List<T>;                       // a new list holding the elements, free to change
}
impl<T: PartialEq> FrozenList<T> { fn contains(self, x: T) -> bool; }
impl<T: Ord> FrozenList<T> { fn binary_search(self, x: T) -> usize | none; }
impl<T: Send + Sync> FrozenList<T> { fn new() -> FrozenList<T>; }   // empty

impl<K: Hash + Eq, V, S: BuildHasher> FrozenDict<K, V, S> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn get(self, k: K) -> V | none;
    fn get_or(self, k: K, default: V) -> V;
    fn contains_key(self, k: K) -> bool;
    fn keys(self) -> Keys<K, V, S>;
    fn values(self) -> Values<K, V, S>;
    fn iter(self) -> DictIter<K, V, S>;
    fn hasher(self) -> S;
    fn to_dict(self) -> Dict<K, V, S>;                 // a new dict, free to change
}

impl<T: Hash + Eq, S: BuildHasher> FrozenSet<T, S> {
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn contains(self, x: T) -> bool;
    fn iter(self) -> Keys<T, bool, S>;
    fn hasher(self) -> S;
    fn to_set(self) -> Set<T, S>;                      // a new set, free to change
}
```

`fr[i]`（超出範圍時觸發 panic）和 `fr[a:b]`（唯讀的 `Slice<T>`，不複製）的用法與串列相同，`fd[k]` 的用法與字典相同；沒有 `IndexSet`，也沒有任何會改變內容的方法。三者都實作 `IntoIterator`（依凍結前集合型別的順序）、`Clone`（同一個集合：它無法改變，因此共用它與複製無異）、`PartialEq`、`Debug`（與未凍結的集合型別相同）和 `Default`（空）；`FrozenList` 另外實作 `Eq` 和 `Hash`。

**凍結**。只有在元素（對字典而言是鍵、值和雜湊器）為 `Send + Sync` 時才有 `freeze`：執行緒讀取的內容也不得改變，而串列的凍結串列會讓每個執行緒拿到同一批可變的內層串列（`List<List<i32>>::freeze` 是錯誤 `E0069`，其說明會要求先凍結每個元素，或透過 `Mutex` 共用該集合）。`freeze` 會交接集合（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）：之後使用 `xs` 是編譯錯誤 `E0105`，並和任何交接一樣透過副本、欄位和呼叫追蹤；請改為讀取凍結後的值，或凍結 `clone()` 以保留 `xs`。在執行時，凍結後的集合以 O(1) 接管儲存空間並讓 `xs` 變為空，因此檢查器看不到的控制代碼（`unsafe` 中的程式碼）修改的是一個新的空集合，絕不是讀取者所讀的內容。曾有 `SliceMut` 的串列則改為提供其元素的副本（O(n)）：過時的檢視仍可能寫入舊的儲存空間。在常數求值期間建立的字典會在凍結時重新雜湊，因此查詢絕不會寫入。要修改凍結的集合，請建立可變的副本（`to_list()`、`to_dict()`、`to_set()`），修改後再次凍結。

**共用**。凍結型別是 `Sync`，因此可以放到集合型別無法進入的地方：多個工作者執行的閉包（`par::map(0..n, |i| table[i])`）、執行緒的閉包、`static`（`static PRIMES: FrozenList<i64> = [2, 3, 5, 7].freeze();`）、多方讀取的 `RwLock`、使結構因而成為 `Sync` 的欄位。交接（`send`、`Mutex::new`、`[move x]`、任務的結果）不會複製凍結的集合。它們的元素本身是 `Sync`，因此 `FrozenList<Atomic<i64>>` 是一個可供多個執行緒累加的計數器陣列。

```nabi
let table = load_table().freeze();                  // FrozenList<Entry>
let hits = par::map(queries, |q| table[q.slot]);    // every worker reads the one table

static NAMES: FrozenDict<str, i32> = {"two": 2, "seven": 7}.freeze();

let hist = List::from_fn(16, |_| Atomic::new(0i64)).freeze();
par::for_each(samples, |x| { hist[bucket(x)].fetch_add(1, Relaxed); });
```

## 2.9.8. `Index` / `IndexSet` / `Sliceable` trait 對應

[2.3 ops](203-ops.md) 的 trait 定義容器上 `[]`、`[i] = v` 和 `[a:b]` 的意義。

| 容器         | `Index<I>`                     | `IndexSet<I, V>`          | `Sliceable<R>`                          |
|--------------|--------------------------------|---------------------------|-----------------------------------------|
| `List<T>`    | `Index<usize>` → T             | `IndexSet<usize, T>` | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `Dict<K, V>` | `Index<K>` → V                 | `IndexSet<K, V>` | **無**                                |
| `Set<T>`     | **無**                       | **無**                  | **無**                                |
| `Tuple<...>` | **編譯器直接處理**（`.0`/.1）  | **編譯器直接處理**       | **無**                                |
| `[T; N]`     | `Index<usize>` → T             | `IndexSet<usize, T>` | `Sliceable<Range<usize>>` → `Slice<T>`（複製） |
| `Slice<T>`   | `Index<usize>` → T             | **無**（唯讀檢視） | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `SliceMut<T>` | `Index<usize>` → T            | `IndexSet<usize, T>` | `Sliceable<Range<usize>>` → `SliceMut<T>` |
| `FrozenList<T>` | `Index<usize>` → T          | **無**（唯讀）      | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `FrozenDict<K, V>` | `Index<K>` → V           | **無**（唯讀）      | **無**                                |
| `str`        | **無**（拒絕直接存取位元組） | **無**          | `Sliceable<Range<usize>>` → `str`（複製，檢查 UTF-8 邊界） |
| `Array<T, N>` | 軸元組（2.9.14）→ `T` 或檢視 | 軸元組，`V` = `T` 或檢視形狀的 `Array` | 透過 `Index<SliceArg>`（refs-lang 3.6.3） |

為了 UTF-8 多位元組字元的安全，刻意不提供 `str` 上的 `s[10]`。逐字元存取請用 `s.chars()`，擷取請用 `s[10..15]`（作為切片，會驗證多位元組邊界）。

## 2.9.9. `IntoIterator` / `FromIterator`

所有容器都實作 `IntoIterator`。

| 容器                | `Item`              | 順序                              |
|---------------------|---------------------|-----------------------------------|
| `List<T>`           | `T`                 | 第一個 → 最後一個                 |
| `Dict<K, V>`        | `(K, V)`            | 插入順序                          |
| `Set<T>`            | `T`                 | 插入順序                          |
| `Deque<T>`          | `T`                 | 前端 → 尾端                       |
| `BinaryHeap<T>`     | `T`                 | **無序**（不是彈出順序）     |
| `Tuple<T...>`       | **未實作**          | （異質，沒有 Iterator 意義）  |
| `Slice<T>`          | `T`                 | 檢視順序（套用步幅）       |
| `SliceMut<T>`       | `T`                 | 檢視順序                          |
| `FrozenList<T>`     | `T`                 | 第一個 → 最後一個                 |
| `FrozenDict<K, V>`  | `(K, V)`            | 插入順序                          |
| `FrozenSet<T>`      | `T`                 | 插入順序                          |

`FromIterator<T>` 決定 `collect::<C>()` 建立哪一種容器。

```nabi
let v: List<i32> = (0..5).collect();
let s: Set<i32> = (0..5).collect();
let d: Dict<str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
```

## 2.9.10. 模組對應

| 容器          | 模組                                     | 內部結構                            |
|---------------|------------------------------------------|-------------------------------------|
| `List<T>`     | `core::collections::list`            | 參考 Rust 的 `Vec`              |
| `Dict<K, V, S>` | `core::collections::dict`          | 緊湊項目 + 線性探測    |
| `Set<T, S>`   | `core::collections::set`             | 建構在 `Dict<T, bool, S>` 之上        |
| `Deque<T>`    | `core::collections::deque`           | 環形緩衝區                         |
| `BinaryHeap<T>` | `core::collections::heap`          | 建構在 `List` 上的隱式二元樹    |
| `SortedDict<K, V>` / `SortedSet<T>` | `core::collections::sorted` | B 樹，節點存放於扁平串列 |
| `Tuple`       | **編譯器內建** + `core::tuple` 的實作 | 由編譯器建立，沒有字面值處理函式 |
| `Slice<T>`    | `core::collections::slice`           | 結構 ptr + len + stride           |
| `SliceMut<T>` | `core::collections::slice`           | 緩衝區 + start + len，串列 + 版本號 |
| `FrozenList<T>` / `FrozenDict<K, V, S>` / `FrozenSet<T, S>` | `core::collections::frozen` | 被凍結集合的儲存空間 |

## 2.9.11. 與其他語言的比較

| 面向            | Rust                 | Python                     | Nabi                                    |
|-----------------|----------------------|----------------------------|----------------------------------------|
| 動態陣列        | `Vec<T>`             | `list`                     | `List<T>`                              |
| 雜湊對應        | `HashMap<K, V>`      | `dict`（插入順序） | `Dict<K, V>`（插入順序）       |
| 雜湊集合        | `HashSet<T>`         | `set`                      | `Set<T>`（插入順序）           |
| 固定異質        | `(T1, T2)` 元組     | `tuple`                    | `Tuple<T1, T2>`                        |
| 檢視            | `&[T]` / `&str`      | **slice 物件**           | `Slice<T>`                             |
| 可寫入檢視      | `&mut [T]`           | **無**（切片指派） | `SliceMut<T>`                         |
| 唯讀、可在執行緒之間共用 | `Arc<[T]>`、`&'static` | `tuple`、`frozenset` | `FrozenList<T>`、`FrozenDict<K, V>`、`FrozenSet<T>`（`freeze`） |
| 字面值          | `vec![...]`（巨集）  | `[...]` `{...}` `(...)`    | `[...]` `{...}` `(...)`（`#[literal]`） |
| 保持插入順序的對應 | `IndexMap`（crate） | **預設的 dict**       | **預設的 `Dict`**                     |
| 有序對應 / 集合 | `BTreeMap` / `BTreeSet` | 無（`sortedcontainers` 套件） | `SortedDict` / `SortedSet`，獨立的型別；`Dict` 不是 |

Nabi 最主要的差異如下。第一，`List`、`Dict`、`Set`、`Tuple` 和 `Slice` 具有一級的字面值語法，因此不使用 `vec![]` 之類的巨集。第二，`Dict` / `Set` 和 Python 3.7+ 的 `dict` 一樣依插入順序迭代，因此不需要另外保持插入順序的型別。不過插入順序不是鍵的順序：範圍查詢、下限 / 上限查詢以及依鍵順序迭代需要有序對應，而 `Dict` 不是；那是另外的 `SortedDict` / `SortedSet`（[2.9.5c](#295c-sorteddictk-v--sortedsett有序對應與集合)）。第三，`Slice<T>` 是帶有步幅的檢視，因此即使是 `xs[::2]` 這樣的運算式也是零複製。

## 2.9.12. `GcBuf<T>`：原始 GC 緩衝區

`GcBuf<T>`（`bare::mem::GcBuf`）是 `List` / `Dict` 等集合型別實作所使用的固定長度 GC 緩衝區，是編譯器知道的型別。它唯一的公開路徑是 `bare::mem`（[0.1.6](../000-meta/001-tier-model.md#016-每個項目只有一條路徑)）；集合型別透過 core 內部的模組取得它。不含 GC 參考的 `T`（數值、`bool`、`char`、`Pod` 結構）的緩衝區是垃圾回收器從不掃描的原始資料（型別 ID `1`，[refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-執行環境保留的型別-id)）；含有參考的 `T` 的緩衝區則透過 `T` 的陣列 `ScanTable` 掃描，只掃描其參考槽位。

```nabi
struct GcBuf<T> { /* representation known to the compiler */ }

impl<T> GcBuf<T> {
    unsafe fn new(len: usize) -> GcBuf<T>;           // every byte set to 0 (references null)
    unsafe fn get(self, i: usize) -> T;        // no bounds check (the owner tracks the length)
    unsafe fn set(self, i: usize, v: T);       // includes the write barrier
}

#[intrinsic("nabi.buf_new")] unsafe fn buf_new<T>(len: usize) -> GcBuf<T>;   // GcBuf::new
#[intrinsic("nabi.buf_get")] unsafe fn buf_get<T>(buf: GcBuf<T>, i: usize) -> T;
#[intrinsic("nabi.buf_set")] unsafe fn buf_set<T>(buf: GcBuf<T>, i: usize, v: T);
```

元素緊密排列：第 `i` 個元素位於酬載的第 `i * size_of::<T>()` 個位元組，採用 `T` 本身的對齊。8 KiB 以上的緩衝區是大型物件：其酬載以 64 位元組對齊，在可達期間絕不移動（[refs-lang 7.2.9](../../refs-lang/007-runtime/702-allocation.md#729-大型物件的-los-配置)）。一般的使用者程式碼使用 `List<T>` 或 `Array<T, N>`（2.9.14）；`GcBuf<T>` 用於實作集合型別。

`GcBuf<T>` 及其基本操作（`bare::mem::{buf_new, buf_get, buf_set}`、`new` / `get` / `set`、對 `GcBuf` 進行索引）是 `unsafe`：它們略過集合型別所保有的元素型別與初始化檢查。

```nabi
#[intrinsic("nabi.buf_same")]
fn buf_same<T>(a: GcBuf<T>, b: GcBuf<T>) -> bool;   // whether they are the same buffer (identity, not contents). Pins nothing
```

## 2.9.13. 預設值、運算子與建構子

**`d[k, default: v]`**。帶有具名部分 `default:` 的兩段式索引讀取 `d.get_or(k, v)`：`k` 不存在時為 `v`，字典不變。作為 `=` 或複合指派的目標、作為欄位 / 索引目標的基底（`d[k, default: v].f += 1`），以及作為方法呼叫的接收者（`d[k, default: []].push(x)`）時，它先執行 `d.get_or_insert(k, v)`，因此 `k` 不存在時會存入 `v`，然後再對 `d[k]` 進行操作。`d` 和 `k` 只求值一次；`v` 在運算式每次執行時都會求值（Swift 的 `subscript(_:default:)`；Python 的 `defaultdict` 每次讀取都會插入，Nabi 的讀取則不會）。

```nabi
counts[w, default: 0] += 1;
groups[sale.region, default: Group {}].orders += 1;
by_len[w.len(), default: []].push(w);
let n = counts["zz", default: 0];        // no entry made
```

**`Dict<K, usize>::most_common(n: usize | none = none) -> List<(K, usize)>`**。最大的 `n` 個計數（`none` 時為全部），由大到小，計數相同者依首次出現的順序（Python 的 `Counter.most_common`）。

**集合運算子**。`a | b` 是 `a.union(b)`，`a & b` 是 `a.intersection(b)`，`a - b` 是 `a.difference(b)`，`a ^ b` 是 `a.symmetric_difference(b)`；每個運算都建立新的集合，兩個運算元皆不變（[2.3](203-ops.md)）。

**`List + List`**。`xs + ys` 是先放 `xs` 的項目、再放 `ys` 的項目的新串列；兩者皆不變（`xs += ys` 則就地附加）。逐元素加法是廣播的 `.+`（[2.6](206-broadcast.md)）。

**`List::from_fn(n: usize, f: fn(usize) -> T) -> List<T>`** 建立 `n` 個項目，第 `i` 個項目為 `f(i)`；`List::filled(n, x)` 建立 `n` 份 `x` 的副本。

## 2.9.14. `Array<T, const N: usize>`：n 維陣列

`core::array::Array<T, N>` 是標準的 n 維陣列：`N` 是秩，元素緩衝區由各副本共用（與 `List` 相同），形狀和各軸的步幅指出每個元素的位置。core 持有容器及其協定；數值演算法（線性代數、統計、FFT）則來自建構在其上的套件。

```nabi
pub struct Array<T, const N: usize> { /* buffer, offset, shape: [usize; N], strides: [isize; N] */ }
pub type Vector<T> = Array<T, 1>;
pub type Matrix<T> = Array<T, 2>;
```

**儲存**。緩衝區是 `GcBuf<T>`（2.9.12）：緊密排列，`T` 不含參考時垃圾回收器從不掃描，以 64 位元組對齊，8 KiB 起絕不移動（[refs-lang 7.2.9](../../refs-lang/007-runtime/702-allocation.md#729-大型物件的-los-配置)）。新建立的陣列是列優先（最後一軸連續）。任何軸的長度都不超過 `isize::MAX`：含有更長軸的形狀會觸發 panic（`capacity overflow`），即使另一軸的長度為 0 也一樣。`Array` 是 `!Sync`（其副本共用未加鎖的緩衝區）。

**字面值**。在 `Array<T, N>` 槽位中，巢狀串列字面值會建立陣列：巢狀深度即為秩，形狀取自巢狀結構，葉節點是依列優先順序排列的元素，每個都以 `T` 為型別。長度不同的列是編譯錯誤（E0142），其說明會要求讓它們等長，或保留為 `List<List<T>>`；應為一列卻不是 `[..]` 字面值的元素也是同一個錯誤。在沒有 `List` 成員的和型別槽位中，字面值會建立其中唯一的 `Array` 成員：傳給 `x: Vector<f64> | none` 的 `[1.0, 2.0]` 是 `Vector<f64>`（有 `List` 成員時，優先取 `List` 成員）。`N` 未定的泛型函式的參數 `x: Array<f64, N>` 從字面值取得秩：對 `fn sum<const N: usize>(x: Array<f64, N>)` 而言，`sum([[1.0, 2.0]])` 的 `N = 2`。

```nabi
let m: Matrix<f64> = [[1, 2, 3],
                      [4, 5, 6]];          // shape [2, 3]
let e: Array<f64, 2> = [[]];               // shape [1, 0]
```

**建構**。

| 項目 | 意義 |
|------|---------|
| `Array::full(shape, v)` | 每個元素皆為 `v` |
| `Array::zeros(shape)` / `Array::ones(shape)` | `T: Zero` / `T: One` |
| `Array::from_fn(shape, f)` | 索引 `idx` 處的元素為 `f(idx)`，`idx: [usize; N]` |
| `Array::from_list(xs, shape)` | 串列的元素依列優先順序排列（複製）；數量不一致時觸發 panic |
| `Array::view_list(xs, shape)` | 同上，但不複製而共用串列的緩衝區：只要串列保有該緩衝區，透過任一方的寫入都會反映在另一方（推入超過容量會把串列搬到新的緩衝區，與 `xs[a:b]` 相同） |
| `From<List<T>> for Array<T, 1>` | `Array::from(xs)` |

**形狀與檢視**。`shape() -> [usize; N]`、`strides() -> [isize; N]`（以元素為單位）、`ndim()`、`len()`（元素數）、`is_empty()`、`is_contiguous()`。下列操作回傳相同元素的檢視，不複製：`t()`（軸反轉）、`permute(axes)`、`broadcast_to(shape)`（長度為 1 的軸以步幅 0 重複）、`reshape(shape)`（連續時為檢視，否則為連續副本的檢視；數量不一致時觸發 panic），以及所有基本索引（見下文）。`copy()` 建立擁有自己緩衝區的連續陣列；`clone()` 另外也複製元素。

**索引**。`Array` 是 `AxisIndexed`（[refs-lang 3.6.3](../../refs-lang/003-expressions/306-indexing-and-slicing.md#363-多軸索引)）：方括號中的每個部分都是一個軸引數。位置（任何型別的整數，視為 `isize`（[refs-lang 3.6.3](../../refs-lang/003-expressions/306-indexing-and-slicing.md#363-多軸索引)）；負值從尾端算起，超出範圍時觸發 panic 並指出該軸）會去掉其軸，切片（`SliceArg`，每軸依 Python 的規則）以步幅保留其軸，`..`（`Rest`）代表其他部分剩下的軸，`none`（`NewAxis`）加入長度為 1 的軸，沒有任何部分指名的軸則整個取用。位置數量與秩相同時得到元素 `T`（秩 1 到 4）；其他情況得到檢視 `Array<T, M>`，`M = N - positions + new axes`。與陣列形狀相同的遮罩 `Array<bool, N>` 選取其值為 `true` 的元素，結果為秩 1 的副本。這些實作涵蓋任意種類的至多兩個部分，以及由位置、切片和一個 `..` 組成的三個部分；對同一個陣列使用更多部分，則是缺少 `Index` 實作。

```nabi
m[1, 2]          // 6.0
m[1]             // [4.0, 5.0, 6.0] (a view)
m[:, ::-1]       // columns reversed
m[m .> 2.0]      // [3.0, 4.0, 5.0, 6.0] (a copy)
m[:, 0] = 0.0;   // IndexSet: a scalar fills the view
m[0, :] = row;   // an array stretches to the view's shape
m[m .< 0.0] = 0.0;
```

| 方法 | 意義 |
|--------|---------|
| `get(idx: [usize; N]) -> T \| none` | 該元素，超出某軸時為 `none` |
| `at(idx)` / `set(idx, v)` | 該元素；超出某軸時觸發 panic |
| `fill(v)` | 每個元素（檢視也可以） |
| `assign(src)` | 複製 `src` 進來，延展為此形狀 |
| `masked(mask)`、`fill_masked(mask, v)`、`assign_masked(mask, vals)` | `a[mask]` 和 `a[mask] = v` 所呼叫的方法 |
| `iter()`、`IntoIterator` | 依列優先順序的元素 |
| `to_list()`、`map(f)`、`cast::<U>()` | 依列優先順序的串列；由 `f(x)` 組成的新陣列；每個元素無損轉換（`U: From<T>`，即 NumPy 的 `astype`；有損轉換請用帶 `as` 的 `map`） |
| `as_list() -> List<T> \| none` | 當元素從陣列起點開始連續時（新建立的陣列、`view_list` 建立的陣列），以共用此陣列緩衝區、不複製的串列提供元素；否則為 `none`（`to_list()` 會複製）。串列第一次推入時會搬到自己的緩衝區 |
| `unsafe fn as_ptr() -> ConstPtr<T>` | 第一個元素；8 KiB 起在可達期間保持穩定（較小者請固定：`Array` 是 `Pinnable`，[stdlib 1.6](../100-bare/106-gc-pin.md)） |

**歸約**。每種歸約都有對整個陣列、依列優先順序走訪元素的形式，以及 `_along(axis)` 形式；後者沿 `axis` 歸約每條線道並去掉該軸：`[2, 3]` 矩陣的 `m.sum_along(0)` 是由各行總和組成的 `[3]` 向量（NumPy 的 `m.sum(axis=0)`）。超過秩的軸會觸發 panic（`sum_along: axis 2 is past the last axis of a rank-2 array; pass an axis in 0..2`）。檢視（轉置、跳步、反轉）依其元素的讀取順序歸約。

| 方法 | 約束 | 整個陣列 | 意義 |
|--------|-------|-------------|---------|
| `sum()`、`sum_along(k)` | `T: Add + Zero` | `T`（空時為 `0`） | 成對求和：每 128 個為一段，以八個累加器加總，再兩半相加，因此浮點數總和的誤差隨 `log2 n` 增長 |
| `prod()`、`prod_along(k)` | `T: Mul + One` | `T`（空時為 `1`） | 依序 |
| `cumsum()`、`cumprod()` | 同 `sum` / `prod` | `Array<T, 1>` | 依列優先順序的累積和 / 累積積 |
| `cumsum_along(k)`、`cumprod_along(k)` | | `Array<T, N>` | 沿每條線道累積，形狀相同 |
| `min()`、`max()` | `T: PartialEq + PartialOrd` | `T \| none`（空時為 `none`） | 任何位置有 NaN 都得到 NaN |
| `argmin()`、`argmax()` | | `usize \| none` | 第一個最小 / 最大值，或第一個 NaN 的列優先位置；`unravel(i)` 提供各軸的索引 |
| `nanmin()`、`nanmax()`、`nanargmin()`、`nanargmax()` | | `T \| none` / `usize \| none` | 同上，但只考慮不是 NaN 的元素 |
| `min_along(k)`、`max_along(k)`、`nanmin_along(k)`、`nanmax_along(k)` | | `Array<T, N - 1>` | 每條線道；長度為 0 的軸觸發 panic |
| `argmin_along(k)`、`argmax_along(k)` | | `Array<usize, N - 1>` | 沿該軸的位置 |
| `mean()`、`mean_along(k)` | `T: FloatingPoint` | `T`（空時為 NaN） | 成對求和除以數量，再加上一次加入平均殘差的走訪 |
| `var(ddof = 0)`、`std(ddof = 0)`、`var_along(k, ddof)`、`std_along(k, ddof)` | | `T` | 修正的兩次走訪演算法（Chan、Golub 與 LeVeque），除以 `n - ddof`；`n <= ddof` 時為 NaN |
| `nansum()`、`nanmean()`、`nansum_along(k)`、`nanmean_along(k)` | | `T` | 只考慮不是 NaN 的元素（沒有這種元素時，`nansum` 為 0，`nanmean` 為 NaN） |
| `any()`、`all()`、`count()` 及其 `_along(k)` | `Array<bool, N>` | `bool` / `usize` | 空陣列的 `any` 為 `false`，`all` 為 `true`；`count` 是 `true` 的數量 |

NaN 是任何不等於自身的元素，因此 NaN 規則適用於所有 `PartialEq` 元素型別，且對整數沒有任何成本。搭配點比較時，布林歸約可以測試條件：`(a .> 0.0).any()`、`(a .< lo).count_along(1)`。

**合併、選取與排序**。

```nabi
fn concat<T, const N: usize>(parts: List<Array<T, N>>, axis: usize = 0) -> Array<T, N>;
fn stack<T, const N: usize>(parts: List<Array<T, N>>, axis: usize = 0) -> Array<T, N + 1>;
fn where<T, const N: usize>(mask: Array<bool, N>, a: Array<T, N>, b: Array<T, N>) -> Array<T, N>;
fn linspace<T: FloatingPoint>(start: T, stop: T, n: usize) -> Array<T, 1>;
fn arange<T: FloatingPoint>(start: T, stop: T, step: T) -> Array<T, 1>;
```

`concat` 沿既有的軸合併陣列（其他每一軸都必須一致，否則觸發 panic 並指出兩個形狀）；`stack` 在 `axis` 處的新軸上合併同形狀的陣列；兩者遇到空串列時都會觸發 panic。`where(mask, a, b)` 在 `mask` 為 `true` 處取 `a` 的元素，其他位置取 `b` 的元素，三者廣播為同一形狀（NumPy 的 `np.where`）；若一側是純量，請對副本使用 `fill_masked`。`linspace` 和 `arange` 是以向量形式提供的 `num::linspace` / `num::arange`（2.1.11）。全部都在 `core::array` 中。

對 `T: PartialEq + PartialOrd`，`sorted_along(k)` 回傳一份副本，其中沿 `k` 的每條線道都以穩定排序遞增排列，NaN 在最後；`sort_along(k)` 就地排序（檢視也可以）；`sorted()` 和 `sort()` 作用於最後一軸（向量的元素）；`argsort_along(k) -> Array<usize, N>` 提供將每條線道排序的位置。

**容差比較**。對 `T: FloatingPoint`，`a.is_close(b, rel = none, abs = 0.0, equal_nan = false) -> Array<bool, N>` 逐元素套用 `f64::is_close`（2.1.11），兩者廣播為同一形狀；`all_close(b, ..) -> bool` 就是 `is_close(..).all()`。`rel` 預設為元素型別的值（`f64`：1e-9，`f32`：2^-20）。此規則預設是對稱且相對的，與 NumPy 的 `allclose`（絕對 1e-8 加上第二個運算元的相對 1e-5）不同；接近零的值必須相符時，請傳入 `abs`。

```nabi
use core::array::{Matrix, concat, where};

let m: Matrix<f64> = [[1, 2, 3], [4, 5, 6]];
m.sum_along(0);              //→ [5.0, 7.0, 9.0]
m.mean_along(1);             //→ [2.0, 5.0]
m.max();                     //→ 6.0
m.argmax_along(1);           //→ [2, 2]
(m .> 2.5).count();          //→ 4
concat([m, m]).shape();      //→ [4, 3]
where(m .> 2.0, m, m .* 0.0); //→ [[0.0, 0.0, 3.0], [4.0, 5.0, 6.0]]
(m .* 0.1 .* 10.0).all_close(m);   //→ true
```

**逐元素運算子**。每個點運算子（2.6）都接受同秩的陣列與陣列、陣列與純量，以及純量與陣列；算術運算得到 `Array<T, N>`，比較得到 `Array<bool, N>`，`.&&` / `.||` 組合 `Array<bool, N>`。兩個陣列依 `broadcast_shape(a, b)` 延展為同一形狀：每一軸上，長度相等則保留，長度為 1 則延展為另一方的長度；其他組合會觸發 panic 並列出兩個形狀（`broadcast: shapes [2, 3] and [2, 2] differ on axis 1; ...`）。在點運算式中，秩較低的陣列對齊到最後幾軸（2.6.8），整個運算式以單一迴圈執行，不產生中間陣列：`Array` 是 `Broadcastable` 和 `Materialize`（2.6.6）。`.=` 和 `.+=` 等就地寫入陣列或陣列的檢視（`a[:, 0] .= 0.0`，[refs-lang 3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-原地廣播)）。`PartialEq` 比較形狀和元素；`Debug` / `Display` 以巢狀方括號輸出。
