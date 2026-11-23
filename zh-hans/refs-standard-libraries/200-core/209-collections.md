# 2.9. 集合类型：`List` / `Dict` / `Set` / `Tuple` / `Slice`

`core::collections` 模块提供通用容器。每个集合类型都分配在 GC 堆上，`List`、`Dict` 和 `Set` 可以通过以 `#[literal(...)]` 绑定的字面量语法直接创建。

本章定义下列集合类型的权威签名，另外还有 `Deque<T>` 和 `BinaryHeap<T>`。

| 集合类型            | 字面量语法                 | 含义                            |
|---------------------|----------------------------|---------------------------------|
| `List<T>`           | `[1, 2, 3]`                | 动态数组（保持插入顺序）        |
| `Dict<K, V, S>`     | `{"a": 1, "b": 2}`         | 哈希映射（保持插入顺序） |
| `Set<T, S>`         | `{1, 2, 3}`                | 哈希集合（保持插入顺序）        |
| `Tuple<...>`        | `(1, "a", true)`           | 定长异构                        |
| `Slice<T>`          | `xs[1:3]`                  | 容器的零拷贝视图                |
| `SliceMut<T>`       | 期望 `SliceMut` 处的 `xs[1:3]` | 列表的可写视图 |
| `FrozenList<T>`、`FrozenDict<K, V, S>`、`FrozenSet<T, S>` | `xs.freeze()` | 只读，在线程之间共享 |

## 2.9.1. 字面量语法：`#[literal(query)]` 绑定

字面量语法用 [refs-lang 6.5 `#[literal]`](../../refs-lang/006-meta-system/605-literal-magic.md) 的**查询 DSL** 定义如下。

```nabi
#[literal("[" {expr} % "," * "]")]
fn list_literal<T>(items: ...T) -> List<T> { ... }

#[literal("{" {expr ":" expr} % "," * "}")]
fn dict_literal<K: Hash, V>(pairs: ...(K, V)) -> Dict<K, V> { ... }

#[literal("{" {expr} % "," + "}")]
fn set_literal<T: Hash>(items: ...T) -> Set<T> { ... }
```

每个处理函数都是 `core::collections` 的包私有函数。元组 `(1, "a", true)` 和定长数组由编译器直接构造，不经过处理函数（[refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md)）。

字面量查询语法的含义是：`{...} % sep *` 表示以 sep 分隔的零次或多次重复，`{...} % sep +` 表示一次或多次。单元素的 Set 写作带尾随逗号的 `{1,}`，以区别于块 `{1}`。在匹配词法单元序列时，词法分析器直接区分词法单元，然后由解析器进行查询匹配。

### 消除 Dict / Set 的歧义

```nabi
{"a": 1}      // Dict: `:` pattern
{1, 2}        // Set: values only
{1,}          // Set (single element, trailing comma)
{1}           // always a block, even when the slot is a Set
{ x = 1 }     // block expression: the token after `{` is statement-like
```

解析器只根据内容判断：`expr ":"` 形式是 Dict，有逗号是 Set，其他情况是块。接收它的槽的类型不参与判断。

### `[...]` 的类型

没有期望类型时，`[1, 2, 3]` 是 `List<T>`。只有当期望类型写作 `[T; N]` 时才会创建定长数组 `[T; N]`，例如 `let a: [i32; 3] = [1, 2, 3];`。定长数组由编译器直接构造，不经过处理函数。

没有期望的元素类型时，各元素像分支一样统一类型（[refs-lang 3](../../refs-lang/003-expressions/README.md)）：它们具有同一个类型，或者某个元素的和类型包含其他所有元素的类型，取先满足的那一种（`x: f64 | none` 时，`[none, x]` 和 `[x, none]` 都是 `List<f64 | none>`）。不会为不同类型的元素凭空构造一个和类型：`[0.5, none]` 是错误（E0040），其帮助信息要求写出列表的类型，即 `let xs: List<f64 | none> = [0.5, none];`。

重复字面量 `[v; N]` 总是定长数组。它只对 `v` 求值一次，并给每个元素各自的副本：元素为 `Copy` 或普通值时按位复制，元素持有集合类型时调用 `v.clone()`，因此 `[List::<i32>::new(); 3]` 是三个互相独立的列表（[refs-lang 2.11.6](../../refs-lang/002-types/211-const-generics.md#2116-参数推断)）。`[T; N]` 的 `Default` 同样对每个元素调用一次 `T::default()`。

当元素实现 `Clone` / `Copy` / `Default` / `Debug` / `PartialEq` / `Eq` / `PartialOrd` / `Ord` / `Hash` 时，定长数组也实现它们。排序按字典序，因此可以直接使用比较运算符（`==`、`<` 等）。它的方法如下：

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

`array::from_fn::<T, N>(f: fn(usize) -> T) -> [T; N]` 创建一个定长数组，其元素 `i` 为 `f(i)`，按顺序调用；`N` 来自期望类型或类型实参。每个元素都是独立的值，因此它可以构造 `[v; N]` 无法复制的元素组成的数组：不是 `Clone` 的类型，或者只有一个所有者的资源。

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

### 引用句柄

`List`、`Dict` 和 `Set` 的值是指向 GC 堆对象的引用句柄。`let ys = xs;` 和参数传递复制的是句柄，因此共享同一个集合。所以修改集合的方法接收 `self`，而不是 `ref self`。需要独立副本时调用 `clone()`。结构体元素是值：`xs[i]` 和 `d[k]` 返回副本，修改要用 `xs[i] = v` 存回（[refs-lang 2.3.8](../../refs-lang/002-types/203-struct.md#238-值语义)）。`iter` / `keys` / `values` 返回不复制集合的惰性迭代器。

## 2.9.2. `List<T>`：动态数组

```nabi
struct List<T> {
    // internal: ptr + len + capacity
}
```

内部由一段连续的内存缓冲区加上 `len` / `capacity` 构成，提供均摊 O(1) 的 push/pop 和 O(1) 的随机访问。

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

`sort_unstable` 是内省排序（introsort）：以三数取中为枢轴的快速排序，少于 20 个元素时用插入排序，深度超过 2 log2(n) 时用堆排序，因此在任何情况下都是 O(n log n)，且不分配内存；相等的元素可能改变顺序。`sort`（归并排序）是稳定的，使用 n 个额外槽位。`_by` 系列搜索采用与 `sort_by` 相同的 `Ordering` 约定：在 `f` 看来升序的列表中，`binary_search_by(|x| x.cmp(target))` 找到 `f` 判为 `Equal` 的第一个元素。

按位置读取有 `T | none` 形式（`get`、`first`、`last`、`pop`）；其他按位置的操作越界时都会 panic（`xs[i]`、`xs[i] = v`、`insert`、`remove`、`swap`、`drain`、`split_at`）。没有 `set`：写入用 `xs[i] = v`，需要跳过越界索引的写入应先检查（`if i < xs.len() { xs[i] = v }`）。

`set_len(n)` 设置长度而不写入元素，用于就地填充的缓冲区（视图的 `raw_parts`、运行时调用）。它是 `unsafe` 的：超过旧长度的元素保存的是存储中原有的内容（从未写入或已腾空的槽位为零），不一定是有效的 `T`；调用者必须在读取前写入它们。用这种方式缩短时，被截掉的元素仍留在缓冲区中。`n` 超过容量时 panic；应先调用 `reserve`。

`partition_point(pred)` 是第一个不满足 `pred` 的元素的索引，`pred` 必须对列表的某个前缀成立、对其余部分不成立；O(log n)。在升序列表中，`x` 的插入点是 `xs.partition_point(|e| e < x)`，按键则是 `xs.partition_point(|e| key(e) < k)`。`binary_search(x)` 返回等于 `x` 的第一个元素的索引（下界），或 `none`。

```nabi
impl List<str> {
    fn join(self, sep: str) -> str;              // joins the elements with sep
}

impl List<u8> {
    fn find<B: AsBytes>(self, needle: B) -> usize | none;   // self[:].find(needle)
}
```

### 示例

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

### trait 实现

| trait                 | 条件                | 含义                            |
|-----------------------|---------------------|---------------------------------|
| `IntoIterator`        | 总是                | `for x in xs`                   |
| `FromIterator<T>`     | 总是                | `(0..5).collect::<List<i32>>()` |
| `Index<usize>`        | 总是                | `xs[i]`                         |
| `IndexSet<usize, T>`  | 总是                | `xs[i] = v`                    |
| `AddAssign<T>`        | 总是                | `xs += x` 推入 `x`              |
| `AddAssign<List<T>>`  | 总是                | `xs += ys` 追加 `ys` 的每个元素 |
| `Sliceable<Range<usize>>` | 总是            | `xs[1:3]`                       |
| `Display`             | **否**              | **建议手动输出**                |
| `Debug`               | `T: Debug`          | `f"{xs:?}"` → `[1, 2, 3]`       |
| `Default`             | 总是                | `List::new()`                   |
| `Clone`               | `T: Clone`          | 深拷贝                          |
| `PartialEq`           | `T: PartialEq`      | 逐元素                          |
| `PartialOrd`          | `T: PartialOrd`     | 字典序；前缀排在前面            |
| `Eq`/`Ord`/`Hash`     | T 有相同的 trait 时 | 字典序                          |

### 比较 / 哈希 / Debug：通用规则

- `List<T>` 按顺序逐元素实现 `PartialEq` / `Eq` / `Hash`。
- `Dict<K, V, S>` 和 `Set<T, S>` 的比较和哈希与顺序无关。只要持有相同的元素（键和值），它们就相等。每个条目都单独用进程密钥求哈希，再把结果相加（[2.5.6](205-cmp.md#256-hash哈希与-eq-一致的承诺)），因此哈希也不依赖于 `S`。
- 当元素实现 `Debug` 时，这三种集合类型都实现 `Debug`，按迭代顺序打印。
- 因此，元素为 `Eq + Hash` 的集合类型可以作为 `Set` 的元素或 `Dict` 的键（`Set<List<i32>>`）。

### 容量与移除：通用规则

- 索引越界时 panic，并给出索引和长度：`index 100 out of bounds for length 3`。方法会先报出自己的名字（`List::remove: index 5 out of bounds for length 3`、`List::drain: range 2..4 out of bounds for length 3`），且 panic 指向调用者。
- 在 `reserve`、`with_capacity`、`repeat` 或扩容中大小溢出时，在任何优化级别下都以 “capacity overflow” panic。
- 移除元素（`pop`、`remove`、`clear`、`truncate`、`drain` 等）会释放引用：GC 不再通过该集合看到它。

## 2.9.3. `Dict<K, V, S>`：哈希映射

```nabi
struct Dict<K: Hash + Eq, V, S = RandomState> {
    // internal: compact entries in insertion order, plus an index of
    // linear-probed slots (none for a small table)
}
```

它提供平均 O(1) 的查找 / 插入 / 移除。迭代按插入顺序进行：已存在时再次插入的键保持原位，移除后再插入的键排到最后。

`S` 构造哈希器（[2.5.11](205-cmp.md#2511-corehash哈希器)）。默认的 `RandomState` 使用每个进程抽取一次的密钥，因此键在表中的位置在每次运行之间会变化；迭代顺序不变。常量求值期间构造的字典在运行时首次使用时会用进程密钥重新哈希。`Dict<K, V>` 就是 `Dict<K, V, RandomState>`，每个 `{k: v}` 字面量也是如此。

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

`d.freeze()`（`K`、`V` 和 `S` 为 `Send + Sync`）把字典移交给一个 `FrozenDict<K, V, S>`，线程之间无需锁即可共享它（[2.9.7b](#297b-frozenlistt--frozendictk-v-s--frozensett-s只读集合类型)）。

`Dict` 实现 `IntoIterator`、`Index<K>`、`IndexSet<K, V>`、`AddAssign<Dict<K, V, S>>`、`PartialEq` / `Eq` / `Hash`（针对 `V`）、`Debug`（`K: Debug, V: Debug`）、`Clone`（`K: Clone, V: Clone`；克隆键和值，哈希器相同）、`Default`、`FromIterator<(K, V)>` 和 `From<List<(K, V)>>`。`Default`、`FromIterator` 和 `From<List<(K, V)>>` 需要 `S: BuildHasher + Default`；其他实现接受任意 `S: BuildHasher`。

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

### 示例

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

`d[k]` 写法假定键存在；需要安全访问时使用 `.get(k)`。panic 信息按 `{k:?}` 的方式显示键，`K` 没有 `Debug` 时显示为 `<K>`（类型名），`V` 显示为值类型的名称。

`d += e`（`AddAssign<Dict<K, V>>`）把 `e` 的每个条目就地插入 `d`；键相同时以 `e` 中的值为准。

`Dict::from(pairs)`（`From<List<(K, V)>>`）从键值对列表构造字典；键重复时以后面的值为准。`K` 和 `V` 来自实参，因此 `let d = Dict::from([("a", 1)]);` 不需要类型标注。

### 惰性视图

`iter` / `keys` / `values` 是当前表上的惰性视图，而不是副本。在迭代期间修改字典，结果未作规定（元素可能被漏掉或被看到两次），但内存安全得以保持。

## 2.9.4. `Set<T, S>`：哈希集合

```nabi
struct Set<T: Hash + Eq, S = RandomState> {
    // internal: Dict<T, bool, S>
}
```

它与 `Dict` 一样按插入顺序迭代；平均性能为 O(1)。与 `Dict` 一样，`S` 构造哈希器。`s.freeze()`（`T` 和 `S` 为 `Send + Sync`）把集合移交给一个 `FrozenSet<T, S>`（[2.9.7b](#297b-frozenlistt--frozendictk-v-s--frozensett-s只读集合类型)）。

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

`Set` 实现 `IntoIterator`、`AddAssign<T>`（插入）、`PartialEq` / `Eq` / `Hash`、`Debug`（`T: Debug`）、`Clone`（`T: Clone`；克隆元素，哈希器相同）、`Default`、`FromIterator<T>` 和 `From<List<T>>`；后三者需要 `S: BuildHasher + Default`。

```nabi
let s: Set<i32> = {1, 2, 3};
s.contains(2);            //→ true
s.insert(4);

let a: Set<i32> = {1, 2, 3};
let b: Set<i32> = {2, 3, 4};
let u = a.union(b);       //→ {1, 2, 3, 4}
let i = a.intersection(b);//→ {2, 3}
```

`Set::from(xs)`（`From<List<T>>`）从列表构造集合，并去掉重复项；`T` 来自实参（`let s = Set::from([1, 2, 2]);`）。`s += x`（`AddAssign<T>`）插入 `x`。

## 2.9.5. 插入顺序

`Dict` 和 `Set` 按插入顺序迭代，就像 Python 3.7+ 的 `dict` 或 Java 的 `LinkedHashMap`，因此没有单独的有序类型。`iter`、`keys`、`values`、`to_list`、`Debug` 和 `for` 都遵循这一顺序：

- 已存在时再次插入的键保持原位（只有值改变）；
- 移除后再插入的键排到最后；
- `retain` 保持保留下来的元素的顺序，`extend` 和 `d += e` 按顺序插入（新键排到最后）；
- `pop_last` / `Set::pop` 取出最新的条目，`pop_first` 取出最旧的条目，平均都是 O(1)（它们留下的空洞数量超过条目数时会被压缩掉）；
- 集合运算的结果先按左操作数的顺序排列。

相等性和哈希忽略顺序（[2.9.2](#292-listt动态数组)），因此 `{"a": 1, "b": 2} == {"b": 2, "a": 1}`。

## 2.9.5a. `Deque<T>`：双端队列

基于环形缓冲区的双端队列。

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

没有 `set`：写入用 `d[i] = v`。`capacity` 是环形缓冲区在扩容前能容纳的元素数。它实现 `d[i]` / `d[i] = v`（越界时 panic，与 `List` 相同：`index 5 out of bounds for length 3`）、`IntoIterator`、`FromIterator`（从前到后）、`Default`、`Clone`（`T: Clone`）、`PartialEq` / `Eq`（从前到后逐元素）和 `Debug`。`d += x`（`AddAssign<T>`）把 `x` 推入尾部。

## 2.9.5b. `BinaryHeap<T>`：优先队列

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

`retain` 就地过滤并恢复堆序，O(n)。`HeapIter<T>` 是 `ExactSizeIterator`。堆实现 `IntoIterator`（无序，与 `iter` 相同）、`Debug`（存储顺序，堆顶在前）和 `Clone`（`T: Clone`；副本保持堆的顺序）；当 `T: Ord` 时，`Default` 和 `FromIterator` 与 `new` 一样构造最大堆。

比较器与 `List::sort_by` 的比较器一样返回 `Ordering`：`by_ordering(|a, b| b.cmp(a))` 先弹出最小的元素。

比较器是 `fn .. + Send`，因此当 `T` 是 `Send` 时堆也是 `Send`，可以通过通道发送或放入 `Mutex`。捕获了非 `Send + Sync` 值（例如集合类型）的比较器是编译错误，其帮助信息建议去掉该捕获：在闭包内部计算该值，或者捕获数组。

## 2.9.5c. `SortedDict<K, V>` / `SortedSet<T>`：有序映射与有序集合

`core::collections::{SortedDict, SortedSet}` 用 B 树按键的顺序（`K: Ord`）保存条目：查找、插入和移除都是 O(log n)，迭代、`range`、`first` / `last` 和 `floor` / `ceil` 都遵循该顺序。它们不在 prelude 中：`use core::collections::SortedDict;`。

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

`SortedSet<T>`（内部是 `SortedDict<T, bool>`）有 `new`、`len`、`is_empty`、`insert(x) -> bool`（新插入时为 true）、`remove(x) -> bool`、`contains`、`clear`、`first`、`last`、`pop_first`、`pop_last`、`floor`、`ceil`、`lower`、`higher`（均返回 `T | none`）、`iter`、`iter_rev`、`range`、`range_rev`、`to_list`，以及 `Set` 的集合运算（`union`、`intersection`、`difference`、`symmetric_difference`、`is_subset`、`is_superset`、`is_disjoint`），运算结果是有序集合。

`range` 接受所有区间形式（[refs-lang 3.7](../../refs-lang/003-expressions/307-range-expressions.md)）；以 `..` 开头的形式作为实参时要写在括号里（`d.range((..10))`）。找到第一个条目是 O(log n)，之后每个条目平均 O(1)。空区间或反向区间不产出任何内容。

两者都实现 `IntoIterator`（键的顺序）、`FromIterator`、`From<List<..>>`（以后出现的键值对的值为准）、`Default`、`Clone`（`K: Clone, V: Clone`；克隆条目）、`PartialEq` / `Eq`（条目相同；顺序即键的顺序）、`Hash`（先是长度，再按顺序哈希每个条目）和 `Debug`（按顺序输出 `{k: v, ..}` / `{a, b, ..}`）。`SortedDict` 实现 `d[k]`（键不存在时 panic：`key 7 not found; use `d.get(k)` to get `V | none``）和 `d[k] = v`；`SortedSet` 实现 `s += x`。副本共享同一个集合，两者与 `Dict` 一样都是 `!Sync`。

在迭代器遍历有序字典或有序集合期间修改它，会在迭代器的下一步 panic，提示先收集键（`SortedDict changed while it was walked; ..`）；替换已存在键的值不算修改（`for (k, v) in d { d[k] = v + 1; }` 没有问题）。

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

**实现**。最小度为 16：一个节点保存 15 到 31 个键（根节点可以更少），节点按节点编号存放在扁平的列表中（键、值和子节点编号各占一个列表），因此一次查找在每一层只访问一个列表槽位。插入在向下的过程中分裂满节点；移除（CLRS）在下降之前，从兄弟节点为键少于 16 个的子节点补充键，或者合并两个子节点。腾空的槽位会被清零，因此被移除的条目不会让任何对象继续存活。

## 2.9.6. `Tuple<...>`：定长异构

```nabi
struct Tuple { /* heterogeneous, determined at compile time */ }
```

元组是语言内建的，不能注册为字面量处理函数（[refs-lang 6.5.7](../../refs-lang/006-meta-system/605-literal-magic.md#657-用户定义的字面量)）。元组表达式必须至少包含一个逗号。`(e)` 是分组表达式；单元素元组写作 `(e,)`。

每个位置可以有不同的类型，长度在编译时固定。

```nabi
let t: Tuple<i32, str, bool> = (42, "hello", true);
let n = t.0;              //→ 42
let s = t.1;              //→ "hello"
let b = t.2;              //→ true

// pattern destructuring
let (n, s, b) = t;
```

### trait 实现

| trait        | 条件                          |
|--------------|-------------------------------|
| `Clone`      | 每个分量都是 `Clone`          |
| `Copy`       | 每个分量都是 `Copy`           |
| `Debug`      | 每个分量都是 `Debug`          |
| `PartialEq`  | 每个分量都是 `PartialEq`      |
| `Eq`         | 每个分量都是 `Eq`             |
| `PartialOrd` | 每个分量都是 `PartialOrd`     |
| `Ord`        | 每个分量都是 `Ord`            |
| `Hash`       | 每个分量都是 `Hash`           |

表中的 trait 由 `core::tuple` 为 1 到 12 个元素的元组提供。`PartialOrd` / `Ord` 按字典序（从第一个元素开始；第一个不同的元素决定结果），因此可以直接对元组使用 `<` / `<=` / `>` / `>=`（`(1, "b") < (1, "c")`）。

`Display` 不会自动派生；建议在需要的地方直接实现它。

### 多返回值

[refs-lang 4.3.5 多返回值](../../refs-lang/004-functions/403-return.md#435-多个返回值元组) 的多值返回按 Tuple 返回处理。

```nabi
fn divmod(a: i32, b: i32) -> (i32, i32) {
    (a / b, a % b)
}

let (q, r) = divmod(17, 5);     //→ q = 3, r = 2
```

## 2.9.7. `Slice<T>`：视图

```nabi
struct Slice<T> {
    // GC buffer + start position + length + stride (1 = contiguous, k = step, negative = reversed)
}
```

它是 `xs[1:3]` / `xs[::2]` / `xs[1:]` 等表达式的结果（[2.3.4 `Sliceable`](203-ops.md#234-索引-trait)）。视图不复制元素；它指向原对象的 GC 缓冲区，只要视图存活，该缓冲区就保持存活。

| 来源       | 视图看到的内容                                                                  |
|------------|---------------------------------------------------------------------------------|
| `List<T>`  | 列表当前的缓冲区。元素的写入可见；列表扩容并移动缓冲区之后的修改不可见 |
| `Slice<T>` | 同一个缓冲区（视图的视图）                                                      |
| `[T; N]`   | 数组的副本（数组是值）                                                          |

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

`raw_parts` 把视图所见的元素作为缓冲区中一段连续的区域给出，用于对一块区域进行运行时调用。带步长的视图（`xs[::2]`）会先复制到新缓冲区，因此通过这段区域的写入不会到达该视图或其列表。`from_raw_parts(buf, start, len)` 是它的逆操作。两者都是 `unsafe` 的：不会检查 `start + len` 是否位于 `buf` 之内，也不会检查其元素是否已初始化。

`slice::Windows<T>` 和 `slice::Chunks<T>` 是 `Item = Slice<T>` 的 `ExactSizeIterator`：每个窗口或块都是同一缓冲区的视图，而不是副本。`Iterator::windows` / `chunks`（[2.7.1](207-iter.md#271-iterator步进)）则收集为列表，用于没有缓冲区的迭代器。

`find` 的 needle 可以是任何 `AsBytes` 类型（[2.11.13](211-conversion.md#21113-asbytes-trait字节视图)）：

```nabi
trait AsBytes {
    fn byte_slice(self) -> Slice<u8>;      // the bytes as a view, no copy
}
```

它为 `str`（其 UTF-8 字节）、`List<u8>`、`Slice<u8>` 和 `[u8; N]` 实现，因此字节串可以作为 needle 传入（依据其 `Slice<u8>` 规则，[refs-lang 1.7.5](../../refs-lang/001-lexical/107-string-literals.md#175-字节串)）。

`Slice` 是只读的（没有 `IndexSet`）；写入列表的视图是 `SliceMut<T>`（[2.9.7a](#297a-slicemutt可写视图)）。负索引从末尾开始计数（[refs-lang 3.6.1](../../refs-lang/003-expressions/306-indexing-and-slicing.md#361-索引arri)）。

### 示例

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

### 与可变参数的集成

`fn f(xs: ...T)` 中的 `xs` 是 `Slice<T>` 视图（[refs-lang 4.2.5](../../refs-lang/004-functions/402-parameters.md#425-可变参数t)）。单独传入的实参和列表展开（`f(..ys)`）以相同的形式接收。

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

## 2.9.7a. `SliceMut<T>`：可写视图

```nabi
struct SliceMut<T> {
    // the list's buffer + start + length, and the list with its version when the view was made
}
```

`SliceMut<T>` 是一个可以写入其所视列表的连续视图，类似 Rust 的 `&mut [T]`。填充或重排列表一部分的函数接收它（`fn zero(buf: SliceMut<u8>)`）。它在期望 `SliceMut<T>` 的地方（实参、带类型标注的 `let`、返回值）创建，也由 `split_at_mut` 和 `chunks_mut` 创建。它在 prelude 中。

| 写在该处的内容                  | 视图                                                                     |
|---------------------------------|--------------------------------------------------------------------------|
| `xs`（`List<T>`）               | 整个列表（[refs-lang 2.14.6](../../refs-lang/002-types/214-cast-and-conversion.md#2146-基于上下文的自动转换)） |
| `xs[a:b]`、`xs[a..b]`、`xs[..]` | 列表的该区间，边界与 `Slice` 一样被截断到有效范围（[refs-lang 3.6.2](../../refs-lang/003-expressions/306-indexing-and-slicing.md#362-切片arrstartend)） |
| `s[a:b]`（`SliceMut<T>`）       | 该视图的可写视图                                                         |

在其他任何地方，`xs[a:b]` 仍然是只读的 `Slice<T>`。带步长的区间（`xs[a:b:k]`）永远不是 `SliceMut`：它不连续。

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

`SliceMutIter<T>` 是 `Item = T` 的 `DoubleEndedIterator` 和 `ExactSizeIterator`；`ChunksMut<T>` 是 `Item = SliceMut<T>` 的 `ExactSizeIterator`。与 `List` 一样，`SliceMut` 不是 `Sync`（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）；移交时会复制其列表，并让视图指向副本。`chunks_mut` 和 `split_at_mut` 给出的视图互不重叠，因此 `par::for_each` 可以把每个视图交给各自的工作线程，由它就地写入列表中属于自己的部分（[2.19.1](219-par.md#2191-核心-api)）。负索引与列表上一样从末尾开始计数。

### 视图存活期间调整列表大小

调整大小可能把列表的元素移到新的缓冲区，因此在列表调整大小之后不得再使用其视图。调整大小是指任何改变长度或缓冲区的 `List` 方法：`push`、`pop`、`insert`、`remove`、`clear`、`truncate`、`drain`、`set_len`、`reserve`、`shrink_to_fit`、`extend`、`retain`、`dedup`、`freeze` 和 `+=`。元素写入（`xs[i] = x`、`fill`、`swap`、`sort`、`reverse`）不是调整大小。

- 当同一个函数中同时出现两者时，在其列表调整大小之后使用视图是编译错误 `E0137`（[refs-lang 9.9.4](../../refs-lang/009-tools/909-debug-tools.md#994-诊断--错误消息)）。编译器会通过列表的副本（`let ys = xs`）、视图的视图（`s[1:]`、`split_at`）、遍历它的 `for` 循环、分支以及循环的下一轮来跟踪视图。重新给视图赋值（`s = xs[..]`）会使它重新可用，在离开的路径上（`return`、`break`）发生的调整大小不会影响其后的代码。
- 其他所有情况都在运行时检查：在被调用的函数中调整大小、保存在结构体中的视图、在调用者调整大小时处于等待中的 `async fn`。列表带有一个版本号，每次调整大小都会递增；在递增之前创建的视图，其方法会 panic（`SliceMut used after its list was resized (push, insert, clear, ...): make the view again after the resize`），对它进行的 `for` 或 `for ref` 循环的每一步、它的移交以及把它列为条目之一的 `par` 调用也会 panic。元素读写（`s[i]`、`s[i] = x`、`get`、`first`、`last`）不做任何检查，因此遍历视图的循环与遍历列表的循环开销相同：在过期视图上，它们访问的是创建视图时所用的存储。如果调整大小后列表保留了该存储，它仍属于列表，否则就不再属于列表；其他线程都不会访问它（`freeze` 会复制曾经有过视图的列表）。

```nabi
let xs: List<i32> = [1, 2, 3];
let s: SliceMut<i32> = xs[1:];
xs.push(4);
s[0] = 9;          // error[E0137]: `s` views `xs`, which `xs.push(..)` resized at line 3
```

正是运行时检查使 `SliceMut` 可以作为 `async fn` 的实参，并跨越 `await` 存活。

### 示例

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

## 2.9.7b. `FrozenList<T>` / `FrozenDict<K, V, S>` / `FrozenSet<T, S>`：只读集合类型

只创建一次、之后只读的集合（查找表、系统读取的组件数组、服务器据以应答的映射）会被冻结：`xs.freeze()` 把 `List`、`Dict` 或 `Set` 变成一个任何代码都无法修改的集合，所有线程都可以在无锁、无复制的情况下读取它。

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

`fr[i]`（越界时 panic）和 `fr[a:b]`（只读的 `Slice<T>`，不复制）与列表上的用法相同，`fd[k]` 与字典上的用法相同；没有 `IndexSet`，也没有任何修改内容的方法。三者都实现 `IntoIterator`（按冻结前集合的顺序）、`Clone`（同一个集合：它不会改变，所以共享它与复制它效果相同）、`PartialEq`、`Debug`（与未冻结的集合相同）和 `Default`（空集合）；`FrozenList` 还实现 `Eq` 和 `Hash`。

**冻结**。只有当元素（对于字典是键、值和哈希器）为 `Send + Sync` 时才有 `freeze`：线程读取的内容同样不能改变，而冻结的列表的列表会让每个线程拿到同一批可修改的内层列表（`List<List<i32>>::freeze` 是错误 `E0069`，其帮助信息建议先冻结每个元素，或者通过 `Mutex` 共享该集合）。`freeze` 会移交集合（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）：之后使用 `xs` 是编译错误 `E0105`，与任何移交一样会通过副本、字段和调用进行跟踪；应改为读取冻结后的值，或者冻结一个 `clone()` 以保留 `xs`。在运行时，冻结后的集合以 O(1) 接管存储并使 `xs` 变为空，因此检查器看不到的句柄（`unsafe` 中的代码）修改的是一个新的空集合，而永远不是读取者读取的内容。曾经有过 `SliceMut` 的列表则改为给出其元素的副本（O(n)）：过期视图仍可能写入旧的存储。常量求值期间构造的字典在冻结时会重新哈希，因此查找永远不会写入。要修改冻结的集合，应创建一个可修改的副本（`to_list()`、`to_dict()`、`to_set()`），修改后再重新冻结。

**共享**。冻结类型是 `Sync`，因此它们可以放在集合类型不能去的地方：多个工作线程运行的闭包（`par::map(0..n, |i| table[i])`）、线程的闭包、`static`（`static PRIMES: FrozenList<i64> = [2, 3, 5, 7].freeze();`）、被多方读取的 `RwLock`、随之成为 `Sync` 的结构体的字段。移交（`send`、`Mutex::new`、`[move x]`、任务的结果）不会复制冻结的集合。它们的元素本身是 `Sync`，因此 `FrozenList<Atomic<i64>>` 是一个可供多个线程累加的计数器数组。

```nabi
let table = load_table().freeze();                  // FrozenList<Entry>
let hits = par::map(queries, |q| table[q.slot]);    // every worker reads the one table

static NAMES: FrozenDict<str, i32> = {"two": 2, "seven": 7}.freeze();

let hist = List::from_fn(16, |_| Atomic::new(0i64)).freeze();
par::for_each(samples, |x| { hist[bucket(x)].fetch_add(1, Relaxed); });
```

## 2.9.8. `Index` / `IndexSet` / `Sliceable` trait 对应关系

[2.3 ops](203-ops.md) 的 trait 定义了容器上 `[]`、`[i] = v` 和 `[a:b]` 的含义。

| 容器         | `Index<I>`                     | `IndexSet<I, V>`          | `Sliceable<R>`                          |
|--------------|--------------------------------|---------------------------|-----------------------------------------|
| `List<T>`    | `Index<usize>` → T             | `IndexSet<usize, T>` | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `Dict<K, V>` | `Index<K>` → V                 | `IndexSet<K, V>` | **无**                                  |
| `Set<T>`     | **无**                         | **无**                    | **无**                                  |
| `Tuple<...>` | **编译器直接处理**（`.0`/.1）  | **编译器直接处理**        | **无**                                  |
| `[T; N]`     | `Index<usize>` → T             | `IndexSet<usize, T>` | `Sliceable<Range<usize>>` → `Slice<T>`（副本） |
| `Slice<T>`   | `Index<usize>` → T             | **无**（只读视图）        | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `SliceMut<T>` | `Index<usize>` → T            | `IndexSet<usize, T>` | `Sliceable<Range<usize>>` → `SliceMut<T>` |
| `FrozenList<T>` | `Index<usize>` → T          | **无**（只读）            | `Sliceable<Range<usize>>` → `Slice<T>`  |
| `FrozenDict<K, V>` | `Index<K>` → V           | **无**（只读）            | **无**                                  |
| `str`        | **无**（拒绝直接按字节访问）   | **无**                    | `Sliceable<Range<usize>>` → `str`（副本，检查 UTF-8 边界） |
| `Array<T, N>` | 轴元组（2.9.14）→ `T` 或视图  | 轴元组，`V` = `T` 或视图形状的 `Array` | 通过 `Index<SliceArg>`（refs-lang 3.6.3） |

为了 UTF-8 多字节字符的安全，`str` 上刻意不提供 `s[10]`。按字符访问使用 `s.chars()`，截取使用 `s[10..15]`（作为切片时会验证多字节边界）。

## 2.9.9. `IntoIterator` / `FromIterator`

每个容器都实现 `IntoIterator`。

| 容器                | `Item`              | 顺序                              |
|---------------------|---------------------|-----------------------------------|
| `List<T>`           | `T`                 | 第一个 → 最后一个                 |
| `Dict<K, V>`        | `(K, V)`            | 插入顺序                          |
| `Set<T>`            | `T`                 | 插入顺序                          |
| `Deque<T>`          | `T`                 | 前 → 后                           |
| `BinaryHeap<T>`     | `T`                 | **无序**（不是弹出顺序）          |
| `Tuple<T...>`       | **未实现**          | （异构，没有迭代器的含义）        |
| `Slice<T>`          | `T`                 | 视图顺序（应用步长）              |
| `SliceMut<T>`       | `T`                 | 视图顺序                          |
| `FrozenList<T>`     | `T`                 | 第一个 → 最后一个                 |
| `FrozenDict<K, V>`  | `(K, V)`            | 插入顺序                          |
| `FrozenSet<T>`      | `T`                 | 插入顺序                          |

`FromIterator<T>` 决定 `collect::<C>()` 构造哪种容器。

```nabi
let v: List<i32> = (0..5).collect();
let s: Set<i32> = (0..5).collect();
let d: Dict<str, i32> = [("a", 1), ("b", 2)].into_iter().collect();
```

## 2.9.10. 模块对应关系

| 容器          | 模块                                     | 内部结构                            |
|---------------|------------------------------------------|-------------------------------------|
| `List<T>`     | `core::collections::list`            | 借鉴 Rust 的 `Vec`                  |
| `Dict<K, V, S>` | `core::collections::dict`          | 紧凑条目 + 线性探测                 |
| `Set<T, S>`   | `core::collections::set`             | 建立在 `Dict<T, bool, S>` 之上      |
| `Deque<T>`    | `core::collections::deque`           | 环形缓冲区                          |
| `BinaryHeap<T>` | `core::collections::heap`          | `List` 上的隐式二叉树               |
| `SortedDict<K, V>` / `SortedSet<T>` | `core::collections::sorted` | B 树，节点存放在扁平列表中 |
| `Tuple`       | **编译器内建** + `core::tuple` 的实现 | 编译器构造，没有字面量处理函数 |
| `Slice<T>`    | `core::collections::slice`           | 结构体 ptr + len + stride           |
| `SliceMut<T>` | `core::collections::slice`           | 缓冲区 + start + len，列表 + 版本号 |
| `FrozenList<T>` / `FrozenDict<K, V, S>` / `FrozenSet<T, S>` | `core::collections::frozen` | 被冻结集合的存储 |

## 2.9.11. 与其他语言的比较

| 方面            | Rust                 | Python                     | Nabi                                    |
|-----------------|----------------------|----------------------------|----------------------------------------|
| 动态数组        | `Vec<T>`             | `list`                     | `List<T>`                              |
| 哈希映射        | `HashMap<K, V>`      | `dict`（保持插入顺序）     | `Dict<K, V>`（保持插入顺序）           |
| 哈希集合        | `HashSet<T>`         | `set`                      | `Set<T>`（保持插入顺序）               |
| 定长异构        | `(T1, T2)` 元组      | `tuple`                    | `Tuple<T1, T2>`                        |
| 视图            | `&[T]` / `&str`      | **切片对象**               | `Slice<T>`                             |
| 可写视图        | `&mut [T]`           | **无**（切片赋值）         | `SliceMut<T>`                         |
| 只读，在线程之间共享 | `Arc<[T]>`、`&'static` | `tuple`、`frozenset` | `FrozenList<T>`、`FrozenDict<K, V>`、`FrozenSet<T>`（`freeze`） |
| 字面量          | `vec![...]`（宏）    | `[...]` `{...}` `(...)`    | `[...]` `{...}` `(...)`（`#[literal]`） |
| 保持插入顺序的映射 | `IndexMap`（crate） | **默认的 dict**          | **默认的 `Dict`**                      |
| 有序映射 / 集合 | `BTreeMap` / `BTreeSet` | 无（`sortedcontainers` 包） | `SortedDict` / `SortedSet`，独立的类型；`Dict` 不是有序映射 |

Nabi 最主要的不同在于以下几点。第一，`List`、`Dict`、`Set`、`Tuple` 和 `Slice` 有一等的字面量语法，因此不使用 `vec![]` 这样的宏。第二，`Dict` / `Set` 与 Python 3.7+ 的 `dict` 一样按插入顺序迭代，因此不需要单独的保持插入顺序的类型。不过插入顺序不是键的顺序：区间查询、floor / ceiling 查找和按键顺序迭代需要有序映射，而 `Dict` 不是有序映射；有序映射是独立的 `SortedDict` / `SortedSet`（[2.9.5c](#295c-sorteddictk-v--sortedsett有序映射与有序集合)）。第三，`Slice<T>` 是带步长的视图，因此即使是 `xs[::2]` 这样的表达式也是零拷贝的。

## 2.9.12. `GcBuf<T>`：原始 GC 缓冲区

`GcBuf<T>`（`bare::mem::GcBuf`）是 `List` / `Dict` 等集合类型的实现所用的定长 GC 缓冲区，是编译器已知的类型。它唯一的公开路径是 `bare::mem`（[0.1.6](../000-meta/001-tier-model.md#016-每个项只有一条路径)）；集合类型通过 core 内部的模块访问它。元素类型 `T` 不含 GC 引用（数值、`bool`、`char`、`Pod` 结构体）的缓冲区是回收器从不扫描的原始数据（类型 id `1`，[refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-运行时保留的类型-id)）；元素类型 `T` 含有引用的缓冲区则通过 `T` 的数组 `ScanTable` 扫描，只扫描其引用槽位。

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

元素紧密排列：元素 `i` 位于载荷的第 `i * size_of::<T>()` 字节处，按 `T` 自身的对齐方式对齐。8 KiB 或更大的缓冲区是大对象：其载荷按 64 字节对齐，并且在可达期间永不移动（[refs-lang 7.2.9](../../refs-lang/007-runtime/702-allocation.md#729-大对象的-los-分配)）。普通用户代码使用 `List<T>` 或 `Array<T, N>`（2.9.14）；`GcBuf<T>` 用于实现集合类型。

`GcBuf<T>` 及其原语（`bare::mem::{buf_new, buf_get, buf_set}`、`new` / `get` / `set`、对 `GcBuf` 的索引）是 `unsafe` 的：它们跳过集合类型所保留的元素类型和初始化检查。

```nabi
#[intrinsic("nabi.buf_same")]
fn buf_same<T>(a: GcBuf<T>, b: GcBuf<T>) -> bool;   // whether they are the same buffer (identity, not contents). Pins nothing
```

## 2.9.13. 默认值、运算符与构造函数

**`d[k, default: v]`**。带具名部分 `default:` 的两部分索引读取 `d.get_or(k, v)`：`k` 不存在时得到 `v`，字典不变。当它作为 `=` 或复合赋值的目标、作为字段 / 索引目标的基础（`d[k, default: v].f += 1`）或作为方法调用的接收者（`d[k, default: []].push(x)`）时，会先执行 `d.get_or_insert(k, v)`，因此 `k` 不存在时会存入 `v`，然后再对 `d[k]` 操作。`d` 和 `k` 只求值一次；`v` 在表达式每次执行时求值（对应 Swift 的 `subscript(_:default:)`；Python 的 `defaultdict` 每次读取都会插入，Nabi 的读取不会）。

```nabi
counts[w, default: 0] += 1;
groups[sale.region, default: Group {}].orders += 1;
by_len[w.len(), default: []].push(w);
let n = counts["zz", default: 0];        // no entry made
```

**`Dict<K, usize>::most_common(n: usize | none = none) -> List<(K, usize)>`**。计数最大的 `n` 项（为 `none` 时返回全部），从大到小排列，计数相同时按首次出现的顺序（对应 Python 的 `Counter.most_common`）。

**集合运算符**。`a | b` 是 `a.union(b)`，`a & b` 是 `a.intersection(b)`，`a - b` 是 `a.difference(b)`，`a ^ b` 是 `a.symmetric_difference(b)`；每个运算都创建新集合，两个操作数都保持不变（[2.3](203-ops.md)）。

**`List + List`**。`xs + ys` 是由 `xs` 的项再加上 `ys` 的项组成的新列表；两者都不改变（`xs += ys` 就地追加）。逐元素加法是广播运算 `.+`（[2.6](206-broadcast.md)）。

**`List::from_fn(n: usize, f: fn(usize) -> T) -> List<T>`** 创建 `n` 个项，第 `i` 项为 `f(i)`；`List::filled(n, x)` 创建 `n` 个 `x` 的副本。

## 2.9.14. `Array<T, const N: usize>`：n 维数组

`core::array::Array<T, N>` 是标准的 n 维数组：`N` 是秩（rank），元素缓冲区由副本共享（与 `List` 相同），形状和各轴步长指明每个元素的位置。core 只提供容器及其协议；数值算法（线性代数、统计、FFT）由建立在它之上的包提供。

```nabi
pub struct Array<T, const N: usize> { /* buffer, offset, shape: [usize; N], strides: [isize; N] */ }
pub type Vector<T> = Array<T, 1>;
pub type Matrix<T> = Array<T, 2>;
```

**存储**。缓冲区是 `GcBuf<T>`（2.9.12）：紧密排列，当 `T` 不含引用时回收器从不扫描它，从 8 KiB 起按 64 字节对齐且永不移动（[refs-lang 7.2.9](../../refs-lang/007-runtime/702-allocation.md#729-大对象的-los-分配)）。新创建的数组是行优先的（最后一个轴连续）。任何轴的长度都不超过 `isize::MAX`：形状中有更长的轴时 panic（`capacity overflow`），即使另一个轴的长度为 0 也是如此。`Array` 是 `!Sync`（它的副本共享一个不加锁的缓冲区）。

**字面量**。在 `Array<T, N>` 槽中，嵌套的列表字面量会构造数组：嵌套深度即秩，形状来自嵌套结构，叶子是按行优先顺序排列的元素，每个元素的类型为 `T`。长度不同的行是编译错误（E0142），其帮助信息建议使它们等长，或者保留为 `List<List<T>>`；应为一行但不是 `[..]` 字面量的元素也是同一个错误。在没有 `List` 成员的和类型槽中，字面量构造其唯一的 `Array` 成员：传给 `x: Vector<f64> | none` 的 `[1.0, 2.0]` 是 `Vector<f64>`（如果有 `List` 成员，则优先取它）。在 `N` 未确定的泛型函数中，形参 `x: Array<f64, N>` 从字面量取得秩：对 `fn sum<const N: usize>(x: Array<f64, N>)` 调用 `sum([[1.0, 2.0]])` 时 `N = 2`。

```nabi
let m: Matrix<f64> = [[1, 2, 3],
                      [4, 5, 6]];          // shape [2, 3]
let e: Array<f64, 2> = [[]];               // shape [1, 0]
```

**构造**。

| 项 | 含义 |
|------|---------|
| `Array::full(shape, v)` | 每个元素都是 `v` |
| `Array::zeros(shape)` / `Array::ones(shape)` | `T: Zero` / `T: One` |
| `Array::from_fn(shape, f)` | 索引 `idx` 处的元素为 `f(idx)`，`idx: [usize; N]` |
| `Array::from_list(xs, shape)` | 列表的元素按行优先顺序排列（副本）；数量不一致时 panic |
| `Array::view_list(xs, shape)` | 同上，但与列表共享缓冲区而不复制：只要列表保留该缓冲区，通过任何一方的写入在另一方都可见（推入超过容量会使列表移到新的缓冲区，与 `xs[a:b]` 的情况相同） |
| `From<List<T>> for Array<T, 1>` | `Array::from(xs)` |

**形状与视图**。`shape() -> [usize; N]`、`strides() -> [isize; N]`（以元素为单位）、`ndim()`、`len()`（元素数量）、`is_empty()`、`is_contiguous()`。下列操作返回相同元素的视图，不复制：`t()`（轴反转）、`permute(axes)`、`broadcast_to(shape)`（长度为 1 的轴以步长 0 重复）、`reshape(shape)`（连续时是视图，否则是连续副本的视图；数量不一致时 panic），以及每种基本索引（见下文）。`copy()` 创建一个拥有自己缓冲区的连续数组；`clone()` 还会克隆元素。

**索引**。`Array` 是 `AxisIndexed`（[refs-lang 3.6.3](../../refs-lang/003-expressions/306-indexing-and-slicing.md#363-多轴索引)）：方括号的每个部分都是一个轴参数。位置（任意类型的整数，按 `isize` 处理（[refs-lang 3.6.3](../../refs-lang/003-expressions/306-indexing-and-slicing.md#363-多轴索引)）；负数从末尾计数，越界时 panic 并指明轴）会去掉其轴，切片（`SliceArg`，每个轴遵循 Python 的规则）保留其轴并带上步长，`..`（`Rest`）代表其他部分剩下的轴，`none`（`NewAxis`）添加一个长度为 1 的轴，没有任何部分指明的轴则整体取用。位置数量等于秩时得到元素 `T`（秩 1 到 4）；其他情况得到视图 `Array<T, M>`，`M = N - positions + new axes`。形状相同的掩码 `Array<bool, N>` 选出其为 `true` 处的元素，结果是秩为 1 的副本。这些实现覆盖任意种类的至多两个部分，以及由位置、切片和一个 `..` 组成的三个部分；对一个数组使用更多部分属于缺少 `Index` 实现。

```nabi
m[1, 2]          // 6.0
m[1]             // [4.0, 5.0, 6.0] (a view)
m[:, ::-1]       // columns reversed
m[m .> 2.0]      // [3.0, 4.0, 5.0, 6.0] (a copy)
m[:, 0] = 0.0;   // IndexSet: a scalar fills the view
m[0, :] = row;   // an array stretches to the view's shape
m[m .< 0.0] = 0.0;
```

| 方法 | 含义 |
|--------|---------|
| `get(idx: [usize; N]) -> T \| none` | 该元素，超出某个轴时为 `none` |
| `at(idx)` / `set(idx, v)` | 该元素；超出某个轴时 panic |
| `fill(v)` | 每个元素（对视图也适用） |
| `assign(src)` | 把 `src` 拉伸到本数组的形状后复制进来 |
| `masked(mask)`、`fill_masked(mask, v)`、`assign_masked(mask, vals)` | `a[mask]` 和 `a[mask] = v` 所调用的方法 |
| `iter()`、`IntoIterator` | 按行优先顺序的元素 |
| `to_list()`、`map(f)`、`cast::<U>()` | 按行优先顺序的列表；由 `f(x)` 组成的新数组；无损地转换每个元素（`U: From<T>`，对应 NumPy 的 `astype`；有损转换用带 `as` 的 `map`） |
| `as_list() -> List<T> \| none` | 当元素从数组起始处连续排列时（新创建的数组、`view_list` 创建的数组），把元素作为与本数组共享缓冲区的列表返回，不复制；否则为 `none`（`to_list()` 会复制）。该列表第一次推入时会移到自己的缓冲区 |
| `unsafe fn as_ptr() -> ConstPtr<T>` | 第一个元素；从 8 KiB 起在可达期间保持稳定（较小的数组需要固定：`Array` 是 `Pinnable`，[stdlib 1.6](../100-bare/106-gc-pin.md)） |

**归约**。每种归约都有一个整个数组的形式，按行优先顺序遍历元素；还有一个 `_along(axis)` 形式，沿 `axis` 归约每条通道并去掉该轴：`[2, 3]` 矩阵的 `m.sum_along(0)` 是由各列之和组成的 `[3]` 向量（对应 NumPy 的 `m.sum(axis=0)`）。超出秩的轴会 panic（`sum_along: axis 2 is past the last axis of a rank-2 array; pass an axis in 0..2`）。视图（转置、带步长、反转）按其元素的读取顺序归约。

| 方法 | 约束 | 整个数组 | 含义 |
|--------|-------|-------------|---------|
| `sum()`、`sum_along(k)` | `T: Add + Zero` | `T`（为空时为 `0`） | 成对求和：以 8 个累加器对每 128 个元素一段求和，再把两半相加，因此浮点求和的误差按 `log2 n` 增长 |
| `prod()`、`prod_along(k)` | `T: Mul + One` | `T`（为空时为 `1`） | 按顺序 |
| `cumsum()`、`cumprod()` | 同 `sum` / `prod` | `Array<T, 1>` | 按行优先顺序的累积和 / 累积积 |
| `cumsum_along(k)`、`cumprod_along(k)` | | `Array<T, N>` | 沿每条通道累积，形状不变 |
| `min()`、`max()` | `T: PartialEq + PartialOrd` | `T \| none`（为空时为 `none`） | 任何位置有 NaN 都得到 NaN |
| `argmin()`、`argmax()` | | `usize \| none` | 第一个最小 / 最大元素的行优先位置，或第一个 NaN 的位置；`unravel(i)` 给出各轴上的索引 |
| `nanmin()`、`nanmax()`、`nanargmin()`、`nanargmax()` | | `T \| none` / `usize \| none` | 同上，但只针对不是 NaN 的元素 |
| `min_along(k)`、`max_along(k)`、`nanmin_along(k)`、`nanmax_along(k)` | | `Array<T, N - 1>` | 按通道；长度为 0 的轴会 panic |
| `argmin_along(k)`、`argmax_along(k)` | | `Array<usize, N - 1>` | 沿该轴的位置 |
| `mean()`、`mean_along(k)` | `T: FloatingPoint` | `T`（为空时为 NaN） | 成对求和除以数量，再加一遍平均残差的修正 |
| `var(ddof = 0)`、`std(ddof = 0)`、`var_along(k, ddof)`、`std_along(k, ddof)` | | `T` | 修正的两遍算法（Chan、Golub 和 LeVeque），除以 `n - ddof`；`n <= ddof` 时为 NaN |
| `nansum()`、`nanmean()`、`nansum_along(k)`、`nanmean_along(k)` | | `T` | 针对不是 NaN 的元素（没有这样的元素时 `nansum` 为 0，`nanmean` 为 NaN） |
| `any()`、`all()`、`count()` 及其 `_along(k)` | `Array<bool, N>` | `bool` / `usize` | 空数组的 `any` 为 `false`，`all` 为 `true`；`count` 是 `true` 的个数 |

NaN 是任何不等于自身的元素，因此 NaN 规则适用于每种 `PartialEq` 元素类型，对整数没有任何开销。配合点运算比较，布尔归约可以检验条件：`(a .> 0.0).any()`、`(a .< lo).count_along(1)`。

**拼接、选择与排序**。

```nabi
fn concat<T, const N: usize>(parts: List<Array<T, N>>, axis: usize = 0) -> Array<T, N>;
fn stack<T, const N: usize>(parts: List<Array<T, N>>, axis: usize = 0) -> Array<T, N + 1>;
fn where<T, const N: usize>(mask: Array<bool, N>, a: Array<T, N>, b: Array<T, N>) -> Array<T, N>;
fn linspace<T: FloatingPoint>(start: T, stop: T, n: usize) -> Array<T, 1>;
fn arange<T: FloatingPoint>(start: T, stop: T, step: T) -> Array<T, 1>;
```

`concat` 沿已有的轴拼接数组（其他所有轴必须一致，否则 panic 并给出两个形状）；`stack` 在 `axis` 处新建一个轴来拼接形状相同的数组；列表为空时两者都会 panic。`where(mask, a, b)` 在 `mask` 为 `true` 处取 `a` 的元素，其余位置取 `b` 的元素，三者广播到同一形状（对应 NumPy 的 `np.where`）；如果一侧是标量，对副本使用 `fill_masked`。`linspace` 和 `arange` 是向量形式的 `num::linspace` / `num::arange`（2.1.11）。它们都在 `core::array` 中。

对于 `T: PartialEq + PartialOrd`，`sorted_along(k)` 返回一个副本，其中沿 `k` 的每条通道都按升序稳定排序，NaN 排在最后；`sort_along(k)` 就地排序（对视图也适用）；`sorted()` 和 `sort()` 针对最后一个轴（向量的元素）；`argsort_along(k) -> Array<usize, N>` 给出使每条通道有序的位置。

**容差比较**。对于 `T: FloatingPoint`，`a.is_close(b, rel = none, abs = 0.0, equal_nan = false) -> Array<bool, N>` 逐元素应用 `f64::is_close`（2.1.11），两者广播到同一形状；`all_close(b, ..) -> bool` 即 `is_close(..).all()`。`rel` 默认取元素类型的值（`f64`：1e-9，`f32`：2^-20）。该规则是对称的，默认为相对误差，与 NumPy 的 `allclose`（绝对误差 1e-8 加上第二个操作数的相对误差 1e-5）不同；需要接近零的值也匹配时，传入 `abs`。

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

**逐元素运算符**。每个点运算符（2.6）都接受秩相同的数组与数组、数组与标量以及标量与数组；算术运算得到 `Array<T, N>`，比较运算得到 `Array<bool, N>`，`.&&` / `.||` 组合 `Array<bool, N>`。两个数组通过 `broadcast_shape(a, b)` 拉伸到同一形状：在每个轴上，长度相等则保持不变，长度为 1 则拉伸到另一方的长度；其他任何组合都会 panic 并给出两个形状（`broadcast: shapes [2, 3] and [2, 2] differ on axis 1; ...`）。在点运算表达式中，秩较低的数组与最后几个轴对齐（2.6.8），整个表达式作为一个循环执行，没有中间数组：`Array` 是 `Broadcastable` 和 `Materialize`（2.6.6）。`.=` 和 `.+=` 等就地写入数组或数组的视图（`a[:, 0] .= 0.0`，[refs-lang 3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-原地广播)）。`PartialEq` 比较形状和元素；`Debug` / `Display` 打印嵌套的方括号。
