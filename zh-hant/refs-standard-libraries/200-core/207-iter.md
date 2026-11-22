# 2.7. iter：迭代 trait

`core::iter` 模組是 `for x in xs` 的**去語法糖依據**。每個 `for` 迴圈都分派到本章的 trait。

[refs-lang 3.13 迴圈運算式](../../refs-lang/003-expressions/313-loop-expressions.md)的 `for` 語法，去語法糖為 `IntoIterator::into_iter` 與 `Iterator::next` 呼叫的組合。

## 2.7.1. `Iterator`：步進

第一個 trait 是 `Iterator`，簽章如下。

```nabi
#[lang("iterator_trait")]
trait Iterator {
    type Item;

    fn next(ref self) -> Item | none;
}
```

核心行為很簡單：每次呼叫 `next` 回傳下一個元素，沒有更多元素時回傳 `none`。

簽章是 `ref self`：迭代器本身持有**狀態**，每次呼叫都就地更新。迭代器是值：複製出來的 `ListIter` 會獨立地繼續，`next(ref self)` 只推進那一份複本。

### 範例（使用者程式碼）

直接呼叫的樣子如下。

```nabi
let xs: List<i32> = [10, 20, 30];
let it = xs.into_iter();         // iterator created by IntoIterator
let v1 = it.next();              //→ 10
let v2 = it.next();              //→ 20
let v3 = it.next();              //→ 30
let v4 = it.next();              //→ none
```

實際的程式碼中，`for` 比直接呼叫更常見。

```nabi
for x in xs {
    process(x);
}
// → the compiler desugars this to:
//   let __it = xs.into_iter();
//   loop {
//       match __it.next() {
//           v: i32 => process(v),
//           none => break,
//       }
//   }
```

### 方法：豐富的預設 impl

使用者必須為 `Iterator` 實作的方法只有 `next(ref self)`。其他所有方法都以 `next` 為基礎，作為預設 impl 提供。

> [!NOTE]
> 下列簽章中的 `fn(Item) -> U` 是 [refs-lang 4.4 函式型別](../../refs-lang/004-functions/404-function-type.md)所定義的統一可呼叫型別，**函式與閉包都接受**。`it.map(some_fn)` 與 `it.map(|x| x * 2)` 都放得進同一個位置。這是單一的形式，不同於 Rust 的 `fn`/`Fn`/`FnMut`/`FnOnce`。

```nabi
impl Iterator {
    fn next(ref self) -> Item | none;       // user-defined

    // transforming adapters (lazy)
    fn map<U>(self, f: fn(Item) -> U) -> Map<Self, U>;
    fn filter(self, pred: fn(Item) -> bool) -> Filter<Self>;
    fn enumerate(self, start: usize = 0) -> Enumerate<Self>;   // (index, item) pairs, from `start`
    fn zip<J: IntoIterator>(self, other: J) -> Zip<Self, J::IntoIter>;    // Item = (Item, J::Item)
    fn chain<J: IntoIterator<Item = Item>>(self, other: J) -> Chain<Self, J::IntoIter>;
    fn take(self, n: usize) -> Take<Self>;
    fn skip(self, n: usize) -> Skip<Self>;
    fn step_by(self, step: usize) -> StepBy<Self>;
    fn rev(self) -> Rev<Self>;                    // DoubleEndedIterator only
    fn flatten(self) -> Flatten<Self> where Item: IntoIterator;
    fn flat_map<U: IntoIterator>(self, f: fn(Item) -> U) -> FlatMap<Self, U>;
    fn peekable(self) -> Peekable<Self>;          // peek(ref self) -> Item | none
    fn take_while(self, pred: fn(Item) -> bool) -> TakeWhile<Self>;
    fn skip_while(self, pred: fn(Item) -> bool) -> SkipWhile<Self>;
    fn scan<S>(self, init: S, f: fn(S, Item) -> S) -> Scan<Self, S>;   // Item = S
    fn inspect(self, f: fn(Item)) -> InspectIter<Self>;    // f sees each item
    fn cycle(self) -> Cycle<Self>;                // endless repeat
    fn chunks(self, size: usize) -> Chunks<Self>;     // Item = List<Item>
    fn windows(self, size: usize) -> Windows<Self>;   // Item = List<Item>

    fn exact_len(self) -> usize | none;           // length without stepping, else none

    // terminal methods (eager)
    fn count(self) -> usize;
    fn sum(self) -> Item where Item: Add<Output = Item> + Zero;     // 0 if empty; floats pairwise (2.7.15)
    fn product(self) -> Item where Item: Mul<Output = Item> + One;  // 1 if empty
    fn fold<B>(self, init: B, f: fn(B, Item) -> B) -> B;
    fn try_fold<B, E>(self, init: B, f: fn(B, Item) throws (E) -> B) throws (E) -> B;
    fn try_for_each<E>(self, f: fn(Item) throws (E)) throws (E);
    fn partition(self, pred: fn(Item) -> bool) -> (List<Item>, List<Item>);
    fn unzip<A, B>(self) -> (List<A>, List<B>) where Self: Iterator<Item = (A, B)>;
    fn reduce(self, f: fn(Item, Item) -> Item) -> Item | none;
    fn any(self, pred: fn(Item) -> bool) -> bool;
    fn all(self, pred: fn(Item) -> bool) -> bool;
    fn find(self, pred: fn(Item) -> bool) -> Item | none;
    fn position(self, pred: fn(Item) -> bool) -> usize | none;
    fn min(self) -> Item | none where Item: PartialOrd;      // the first of equal minima; NaN propagates
    fn max(self) -> Item | none where Item: PartialOrd;      // the last of equal maxima; NaN propagates
    fn min_by_key<K: Ord>(self, key: fn(Item) -> K) -> Item | none;  // the first of equals
    fn max_by_key<K: Ord>(self, key: fn(Item) -> K) -> Item | none;  // the last of equals
    fn collect<C: FromIterator<Item> = List<Item>>(self) -> C;   // a List when nothing picks C

    // materializing, sorting, counting, combinatorics (2.7.15)
    fn to_list(self) -> List<Item>;
    fn to_set(self) -> Set<Item> where Item: Hash;
    fn to_dict<K: Hash, V>(self) -> Dict<K, V> where Self: Iterator<Item = (K, V)>;
    fn sorted(self, reverse: bool = false) -> List<Item> where Item: Ord;
    fn sorted_by_key<K: Ord>(self, key: fn(Item) -> K, reverse: bool = false) -> List<Item>;
    fn sorted_by(self, cmp: fn(Item, Item) -> Ordering) -> List<Item>;
    fn counts(self) -> Dict<Item, usize> where Item: Hash;
    fn group_by<K: Hash>(self, key: fn(Item) -> K) -> Dict<K, List<Item>>;
    fn pairwise(self) -> Pairwise<Self>;                    // Item = (Item, Item)
    fn combinations(self, k: usize) -> Combinations<Item>;  // Item = List<Item>
    fn permutations(self, k: usize | none = none) -> Permutations<Item>;
    fn for_each(self, f: fn(Item));               // side effects only
    fn last(self) -> Item | none;
    fn nth(ref self, n: usize) -> Item | none;
}
```

### `sum` / `product` 與 `Zero` / `One`

`sum` 從 `Zero::zero()` 開始，以 `+` 累加；`product` 從 `One::one()` 開始，以 `*` 累乘。因此空迭代器的總和為 0，乘積為 1。`Zero` / `One` 是 `core::num` 的 trait（[2.1](201-numbers.md)）。

### `min` / `max` 與浮點數

`min` 與 `max` 需要 `PartialOrd`，因此浮點數迭代器也有這兩個方法：與自身無法排序的項目（NaN）就是結果（2.7.15）。`min_by_key` 與 `max_by_key` 需要 `K: Ord`，也就是全序；`f32` 與 `f64` 不是 `Ord`，因為 NaN 無法排序，以浮點數作為鍵是錯誤（E0069），其說明會給出 `reduce` 的寫法。浮點數串列依全序排序：`xs.sort_by(f64::total_cmp)`。

### 停止、累進狀態、分組

- `take_while(pred)` 產出項目，直到 `pred` 第一次拒絕某個項目為止。該項目會從內部迭代器取出並丟棄；之後的項目都不會取出，所以停止點之後的副作用不會執行。`skip_while(pred)` 丟棄項目，直到 `pred` 第一次拒絕某個項目，然後產出該項目以及之後所有項目，不再檢查。
- `scan(init, f)` 產出累進的狀態：`f(init, x0)`，然後是以該值與 `x1` 呼叫 `f` 的結果，依此類推。它就是會產出每一步的 `fold`，所以產出的項目數與內部迭代器相同。
- `step_by(step)` 產出第一個項目，之後每隔 `step` 個產出一個。`step` 為 0 時觸發 panic。
- `peekable()` 回傳 `Peekable<Self>`，其 `peek(ref self) -> Item | none` 給出下一個項目，並把它留給接下來的 `next`。
- `inspect(f)` 把每個項目傳給 `f`，並原樣產出。
- `cycle()` 每當迭代器耗盡時，就從 `cycle` 接收當時的迭代器複本重新開始，因此除非迭代器為空，否則永無止境；迭代器為空時則什麼都不產出。產生器的複本共用同一個位置（2.7.10a），所以對產生器使用 `cycle` 不會重複。
- `chunks(size)` 依序產出每 `size` 個項目一組的串列，項目用盡時最後一組較短。`windows(size)` 產出 `size` 個連續項目的串列，每次滑動一格；項目少於 `size` 個時什麼都不產出。兩者在 `size` 為 0 時都會觸發 panic。`List` 與 `Slice` 有自己的 `chunks` / `windows`，產出的是檢視（[2.9.7](209-collections.md#297-slicet檢視)）。
- `try_fold` 與 `try_for_each` 接受會擲回錯誤的 `f`：第一個錯誤就停止迴圈，之後的項目都不會走訪，該錯誤擲回給呼叫端。
- `partition(pred)` 以兩個串列回傳 `pred` 接受的項目與其餘項目，各自保持原順序。`unzip()` 把成對 `(A, B)` 的迭代器拆成 `List<A>` 與 `List<B>`。它的項目必須是成對的：`A` 與 `B` 由 `Item = (A, B)` 推論，其他形式的項目在呼叫處是錯誤。

```nabi
let xs = [1, 2, 3, 4];
let small: List<i32> = xs.iter().take_while(|x| x < 3).collect();   //→ [1, 2]
let sums: List<i32> = xs.iter().scan(0, |acc, x| acc + x).collect(); //→ [1, 3, 6, 10]
let pairs: List<List<i32>> = xs.iter().windows(2).collect();        //→ [[1, 2], [2, 3], [3, 4]]
let (even, odd) = xs.iter().partition(|x| x % 2 == 0);              //→ [2, 4], [1, 3]
let (ns, ss) = [(1, "a"), (2, "b")].iter().unzip();                 //→ [1, 2], ["a", "b"]
```

### 惰性與及早

這些方法分成兩組。`map`、`filter`、`take` 等轉換配接器是**惰性**的：它們只回傳新的迭代器，不做任何計算。只有呼叫 `count`、`sum`、`collect`、`for_each` 等終端方法時才開始步進。

```nabi
let xs = [1, 2, 3, 4, 5];
let r = xs.into_iter()
    .map(|x| x * 2)            // lazy: nothing computed yet
    .filter(|x| x > 4)         // lazy
    .sum();                    // eager: stepping happens here
//→ 6 + 8 + 10 = 24
```

### 使用者定義的 Iterator

直接定義 Iterator 的範例：

```nabi
struct Counter { n: i32; }

impl Iterator for Counter {
    type Item = i32;

    fn next(ref self) -> i32 | none {
        if self.n < 5 {
            let cur = self.n;
            self.n = self.n + 1;
            cur
        } else {
            none
        }
    }
}

let c = Counter { n: 0 };
for x in c {
    nabi_print_i32(x);          //→ 0, 1, 2, 3, 4
}
```

## 2.7.2. `IntoIterator`：`for` 的起點

接下來是 `IntoIterator` trait，它是 `for` 迴圈的起點。

```nabi
#[lang("into_iterator_trait")]
trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Item>;

    fn into_iter(self) -> IntoIter;
}
```

`IntoIterator` 是讓集合型別建立迭代器的配接器。`for x in xs` 不會把 **xs 本身**當作迭代器；它以 `xs.into_iter()` 建立迭代器，再對它呼叫 `next`。

### 為何分開

這兩個 trait 分開，是因為兩者的觀點根本不同。

- 集合型別本身（List、Dict 等）是無狀態的容器，不知道自己指向哪個位置。
- 迭代器帶有進度狀態，追蹤自己走到了哪裡。

因此同一個集合型別迭代兩次，每次都需要新的迭代器，而 `into_iter` 就是建立它的地方。

### 自動 impl

`Iterator` 本身自動得到 `IntoIterator` 的 impl，其 `into_iter` 原樣回傳 self。

```nabi
// every Iterator is automatically an IntoIterator
impl<I: Iterator> IntoIterator for I {
    type Item = I::Item;
    type IntoIter = I;
    fn into_iter(self) -> I { self }
}
```

所以 `for x in iter` 可以直接使用：若 iter 已經是 Iterator，`into_iter` 就回傳 self。

### `for` 去語法糖的確切形式

更精確地說，`for` 迴圈的去語法糖如下：

```nabi
for x in expr {
    body
}
```

去語法糖為：

```nabi
let __it = expr.into_iter();
loop {
    match __it.next() {
        v: T => {
            let x = v;
            body;
        }
        none => break,
    }
}
```

詳細的去語法糖規則見 [refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md)。

## 2.7.3. `Step`：單步抽象

對 `Range<T>` 的迭代建立在「移到下一個值」這個簡單操作上。這與一般的算術（`Add`）意義不同：例如 `char` 的下一個值是碼位 + 1，但 `char + char` 本身沒有定義。因此 Nabi 把步進放在獨立的 trait 中。

```nabi
// core::range::Step
trait Step: PartialOrd + Clone {
    fn forward_checked(self, count: usize) -> Self | none;
    fn backward_checked(self, count: usize) -> Self | none;
    fn steps_between(start: Self, end: Self) -> usize | none;
}
```

`Step` 位於 `core::range`，不在 prelude 中：使用者撰寫 `impl Step for T` 需要 `use core::range::Step;`。

各方法的意義：

- `forward_checked(self, n)`：回傳從 `self` 往前 `n` 步的值；超出型別範圍時回傳 `none`。
- `backward_checked(self, n)`：往後移動的值；同樣在超出範圍時回傳 `none`。
- `steps_between(start, end)`：計算從 `start` 到 `end` 有幾步；無法表示時（例如為負、溢位）回傳 `none`。

### 為何與算術 trait 分開

`Step` 與 `Add` 分開有三個理由。

- **意義分離**：`+1` 是單純的遞增，不是任意的算術。`MyVec3 + MyVec3` 有意義，但 `Range<MyVec3>` 沒有；沒有另外的 `Step` impl 就會被拒絕。
- **排除浮點數**：`f32` / `f64` 沒有 `Step` impl，所以 `Range<f64>` 不是迭代器：`for x in 0.0..1.0` 是編譯錯誤（E0068）。`0.0..1.0` 仍可用來測試成員關係（`x in 0.0..1.0`）。
- **有效率的跳躍**：`Range`、`RangeInclusive` 與 `RangeFrom` 以單次 `forward_checked(_, n)` 呼叫覆寫 `Iterator::nth(n)`，所以在範圍上 `nth(n)` 與 `step_by(n)`（透過 `nth` 步進）以 O(1) 執行。若只有 `succ`，同樣的操作會是 O(n)。

### Step impl 矩陣

標準函式庫只為整數基本型別與 `char` 實作 `Step`。

| 型別                 | impl     | 備註                                                |
|----------------------|----------|------------------------------------------------------|
| `i8`..`i64`, `isize` | **有**  | `forward_checked` 在溢位時回傳 `none`         |
| `u8`..`u64`, `usize` | **有**  | `backward_checked` 在下溢時回傳 `none`       |
| `int`                | **無**   | 標準函式庫沒有 impl                             |
| `char`               | **有**  | 以碼位步進，跳過代理碼位（`0xD800..=0xDFFF`） |
| `bool`               | **無**   | 標準函式庫沒有 impl                             |
| `f32`, `f64`         | **無**   | `Range<f64>` 不是迭代器：步長有歧義     |
| `float`              | **無**   | （同上）                                      |

使用者定義的型別可以自由加上 `impl Step for MyType`，立即就能使用 `Range<MyType>`。

```nabi
use core::range::Step;

enum Weekday { Mon, Tue, Wed, Thu, Fri, Sat, Sun }

impl Step for Weekday {
    fn forward_checked(self, n: usize) -> Weekday | none { ... }
    fn backward_checked(self, n: usize) -> Weekday | none { ... }
    fn steps_between(s: Weekday, e: Weekday) -> usize | none { ... }
}

for d in Weekday::Mon..=Weekday::Fri {
    schedule(d);     // the 5 weekdays
}
```

## 2.7.4. `Range<T>` / `RangeInclusive<T>` / `RangeFrom<T>` / `RangeTo<T>` / `RangeToInclusive<T>` / `RangeFull`

[refs-lang 3.7 範圍運算式](../../refs-lang/003-expressions/307-range-expressions.md)的 `a..b` 語法，去語法糖為這些標準函式庫型別。

### `Range<T>`：`a..b`（不含終點）

```nabi
#[lang("range_type")]
struct Range<T> {
    pub start: T;
    pub end: T;
}

impl<T: Step> Iterator for Range<T> {
    type Item = T;

    fn next(ref self) -> T | none {
        if self.start < self.end {
            let cur = self.start.clone();
            self.start = self.start.forward_checked(1)?;   // overflow → none
            cur
        } else {
            none
        }
    }
}
```

每種範圍形式都由 `core::range` 的 `#[literal]` 處理器註冊（在建構 `Range` 的函式上標註 `#[literal(expr ".." expr)]`，每種形式一個處理器），所以 `0..5` 這類寫法會去語法糖為 `Range` 結構（以 [refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md) 為準）。編譯器透過 `range_type` lang 槽位找到 `Range`。

```nabi
for i in 0..5 {
    process(i);          //→ 0, 1, 2, 3, 4
}

let v: List<i32> = (0..10).collect();   //→ [0, 1, ..., 9]
```

### `RangeInclusive<T>`：`a..=b`

```nabi
struct RangeInclusive<T> {
    pub start: T;
    pub end: T;
}

impl<T: Step> Iterator for RangeInclusive<T> { ... }
```

`start` 與 `end` 以外的欄位（例如標記已到達終點的內部旗標）屬於實作細節，是私有的。即使 `end` 是該型別的最大值，`a..=b` 也恰好產出到 `b` 為止。

```nabi
for i in 0..=5 {
    process(i);          //→ 0, 1, 2, 3, 4, 5
}
```

### `RangeFrom<T>`：`a..`（無限）

```nabi
struct RangeFrom<T> { pub start: T; }

impl<T: Step> Iterator for RangeFrom<T> {
    type Item = T;
    fn next(ref self) -> T | none {
        let cur = self.start.clone();
        self.start = self.start.forward_checked(1)?;  // overflow → none (e.g. reaching i32::MAX)
        cur
    }
}
```

```nabi
let it = (10..).take(3);     //→ [10, 11, 12] (take is a lazy adapter)
```

`RangeFrom` 不會產出型別的最大值。由於最大值沒有下一個值，迭代在到達最大值時、產出它之前就結束（`(254u8..)` 只產出 `254` 就結束）。

### `RangeTo<T>` / `RangeToInclusive<T>` / `RangeFull`

```nabi
struct RangeTo<T> { pub end: T; }

struct RangeToInclusive<T> { pub end: T; }               // ..=b

struct RangeFull;
```

`..b` / `..=b`（沒有起點）與 `..`（兩者皆無）只在切片或索引的上下文中有意義，不會建立迭代器。

```nabi
let xs = [1, 2, 3, 4, 5];
let s = xs[..3];          //→ [1, 2, 3]  (Slice: the Slice trait of 2.3 ops)
```

## 2.7.5. `FromIterator<T>`：`collect` 的分派

負責 `collect` 分派的 trait 是 `FromIterator<T>`。

```nabi
trait FromIterator<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self;
}
```

呼叫 `Iterator::collect` 時，會呼叫目標集合型別的 `from_iter`。

```nabi
let v: List<i32> = (0..5).collect();
// = List::from_iter((0..5).into_iter())
```

每個集合型別（`List`、`Dict`、`Set`、`str` 等）都實作 `FromIterator`。

## 2.7.6. 各集合型別的 IntoIterator impl

各集合型別實作 `IntoIterator` 的方式：

| 集合型別          | `Item` 型別 | 行為                                  |
|---------------------|-------------|-------------------------------------------|
| `List<T>`           | `T`         | 從第一個元素依序到最後一個 |
| `Dict<K, V>`        | `(K, V)`    | 插入順序                           |
| `Set<T>`            | `T`         | 插入順序                           |
| `Tuple<A, B, C>`    | （特殊）   | **不建立迭代器**；請使用模式比對  |
| `Range<T>`          | `T`         | 從 start 到 end-1                       |
| `Slice<T>`          | `T`         | 從檢視的開頭到結尾     |

### Dict / Set 的迭代順序

`Dict<K, V>` 與 `Set<T>` 依插入順序迭代，每次執行都相同：已存在的鍵再次插入時保留原位，移除後再插入的鍵則排到最後。雜湊種子改變的是鍵在表中的位置，而不是順序（[2.9](209-collections.md)）。

## 2.7.7. 無限迭代器與 `take`

無限迭代器（例如 `RangeFrom` 或使用者定義的迭代器）直接與及早方法組合時，會永遠迴圈下去。

```nabi norun
(0..).count();         // infinite loop: count runs to the end
(0..).sum();           // infinite loop
```

用 `take(n)` 讓它變成有限。

```nabi
(0..).take(100).count();     //→ 100
(0..).take(10).sum();        //→ 0+1+...+9 = 45
```

`Iterator` 不會偵測無限。使用者必須刻意組合惰性配接器。

## 2.7.8. 函式廣播：`f.(x)` 的去語法糖

[refs-lang](../../refs-lang/003-expressions/303-broadcast-operators.md) 中的函式廣播（`sin.(xs)`、`f.(a, b)`）去語法糖為 `Iterator::map`，沒有自己的 trait。

```nabi
sin.(xs)   ≡   xs.into_iter().map(sin).collect()
g.(a, b)   ≡   a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

## 2.7.9. `DoubleEndedIterator`：雙向步進

可以雙向步進的迭代器以 `DoubleEndedIterator` trait 表示。

```nabi
trait DoubleEndedIterator: Iterator {
    fn next_back(ref self) -> Item | none;

    // provided
    fn rfold<B>(self, init: B, f: fn(B, Item) -> B) -> B;     // fold from the back
    fn rfind(self, pred: fn(Item) -> bool) -> Item | none;    // the last item pred accepts
}
```

這個 trait 代表可以從後端步進的迭代器；`rev()` 配接器、`rfold` 與 `rfind` 都需要它。

`List<T>` 與 `Slice<T>` 的迭代器，以及下列型別，必須實作這個 trait。

- `Range<T>`、`RangeInclusive<T>`
- `Rev<I>`（當 `I` 為雙向時）、`Map<I, U>`、`Filter<I>`、`Chain<A, B>`、`InspectIter<I>`（當組成的迭代器為雙向時）
- `Enumerate<I>`、`Zip<A, B>`、`Take<I>`、`Skip<I>`、`StepBy<I>`：當組成的迭代器為 `DoubleEndedIterator + ExactSizeIterator` 時，因為從後端計數需要知道長度。

`Dict<K, V>` 與 `Set<T>` 不實作它。

```nabi
let r: List<i32> = (0..5).rev().collect();    //→ [4, 3, 2, 1, 0]
```

## 2.7.10. `ExactSizeIterator`：已知長度的迭代器

步進前就知道元素數量的迭代器，以 `ExactSizeIterator` 表示。

```nabi
trait ExactSizeIterator: Iterator {
    fn len(self) -> usize;
}
```

`len()` 是剩餘的元素數量。`Iterator::exact_len()` 在不步進就能得知該數量時回傳它，否則回傳 `none`（預設）；標準函式庫的 `ExactSizeIterator` 型別回傳它們的 `len()`，以它們為基礎建構的下列配接器也一樣。預設的 `count()` 在 `exact_len()` 不是 `none` 時直接回傳它，所以 `count()` 不會呼叫 `map` 的函式：`xs.iter().map(f).take(3).count()` 呼叫 `f` 零次。`scan` 與 `inspect` 的函式也是如此。長度未知的迭代器，例如 `Filter`、`TakeWhile`、`SkipWhile` 或 `(0..).take(100)`，則會實際步進。`List`、`Slice` 與 `Range` 的迭代器，`List` 與 `Slice` 的 `windows` / `chunks` 檢視，以及在組成的迭代器全都是 `ExactSizeIterator` 時的 `Enumerate`、`Zip`、`Take`、`Skip`、`StepBy`、`Rev`、`Map`、`Scan`、`InspectIter`、`Chunks`，都實作這個 trait。`Windows` 在已知大小的迭代器之上有 `exact_len`，而 `Cycle` 只有在其迭代器為空時才有（值為 0）。

## 2.7.10a. `GenIter<T>` / `TryGenIter<T, E>`：產生器的值

這些是 `yield fn` 呼叫與 `yield { .. }` 區塊的值（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#協程函式的結果型別)）。

```nabi
struct GenIter<T> { /* private */ }
impl<T> Iterator for GenIter<T> {
    type Item = T;
    fn next(ref self) -> T | none;          // runs the body to the next yield
}

struct TryGenIter<T, E> { /* private */ }    // yield fn .. throws (E)
impl<T, E> TryIterator for TryGenIter<T, E> {
    type Item = T;
    type Error = E;
    fn next(ref self) throws (E) -> T | none;   // throws what the body throws
}
```

`TryGenIter` 是 `TryIterator`（[2.7.13](#2713-tryiterator可能失敗的迭代)），而不是 `Iterator`，因為它的 `next` 會擲回錯誤。它以 `try for`、`try g.next()` 或其配接器步進。擲回錯誤之後產生器就結束了：下一次 `next` 給出 `none`。私有欄位不作規定。複本共用同一個協程框架，因此共用進度位置。

## 2.7.11. 模組對應

各 trait 對應的 lang 槽位與標準函式庫位置：

| 項目                  | lang 槽位             | 標準函式庫位置 |
|-----------------------|-----------------------|---------------------------|
| `Iterator` trait      | `iterator_trait`      | `core::iter`         |
| `IntoIterator` trait  | `into_iterator_trait` | `core::iter`         |
| `Step` trait          | （無）                | `core::range`        |
| `FromIterator` trait  | （無）                | `core::iter`         |
| `DoubleEndedIterator` | （無）                | `core::iter`         |
| `ExactSizeIterator`   | （無）                | `core::iter`         |
| `TryIterator` trait   | `try_iterator_trait`  | `core::iter`         |
| `Range<T>`            | `range_type`          | `core::range`        |
| 其他 5 種範圍型別 | （無）              | `core::range`        |

基本型別本身沒有 `Iterator` impl（i32 不能作為 `for` 的迭代目標）。只有集合型別與 Range 才有。

## 2.7.12. async：`Stream` trait

非同步迭代器在 [2.8 stream](208-stream.md) 中說明。`for await x in stream` 語法（[refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md)）分派到 `Stream::poll_next`。

## 2.7.13. `TryIterator`：可能失敗的迭代

`next` 可能失敗的迭代器是 `TryIterator`。它的錯誤是 `next` 的具型別通道，而不是項目的一部分：`next` 給出一個項目、在結尾給出 `none`，或擲回 `Error`。

```nabi
#[lang("try_iterator_trait")]
trait TryIterator {
    type Item;
    type Error;
    fn next(ref self) throws (Error) -> Item | none;

    // lazy adapters; each function may throw `Error`
    fn map<U>(self, f: fn(Item) throws (Error) -> U) -> TryMap<Self, U>;
    fn filter(self, pred: fn(Item) throws (Error) -> bool) -> TryFilter<Self>;
    fn map_err<E2>(self, f: fn(Error) -> E2) -> MapErr<Self, E2>;
    fn enumerate(self, start: usize = 0) -> TryEnumerate<Self>;
    fn zip<J: IntoIterator>(self, other: J) -> TryZip<Self, J::IntoIter>;
    fn chain<J: TryIterator<Item = Item, Error = Error>>(self, other: J) -> TryChain<Self, J>;
    fn take(self, n: usize) -> TryTake<Self>;
    fn skip(self, n: usize) -> TrySkip<Self>;
    fn step_by(self, step: usize) -> TryStepBy<Self>;    // step 0 panics
    fn take_while(self, pred: fn(Item) throws (Error) -> bool) -> TryTakeWhile<Self>;
    fn skip_while(self, pred: fn(Item) throws (Error) -> bool) -> TrySkipWhile<Self>;
    fn flat_map<U: IntoIterator>(self, f: fn(Item) throws (Error) -> U) -> TryFlatMap<Self, U>;
    fn scan<S>(self, init: S, f: fn(S, Item) throws (Error) -> S) -> TryScan<Self, S>;
    fn inspect(self, f: fn(Item) throws (Error)) -> TryInspect<Self>;
    fn peekable(self) -> TryPeekable<Self>;      // peek(ref self) throws (Error) -> Item | none
    fn chunks(self, size: usize) -> TryChunks<Self>;     // size 0 panics

    // to a plain iterator of results
    fn results(self) -> Results<Self>;           // Iterator<Item = Result<Item, Error>>

    // eager: the first error stops the loop and is thrown
    fn collect<C: FromIterator<Item>>(self) throws (Error) -> C;
    fn count(self) throws (Error) -> usize;
    fn last(self) throws (Error) -> Item | none;
    fn nth(ref self, n: usize) throws (Error) -> Item | none;
    fn fold<B>(self, init: B, f: fn(B, Item) throws (Error) -> B) throws (Error) -> B;
    fn for_each(self, f: fn(Item) throws (Error)) throws (Error);
    fn reduce(self, f: fn(Item, Item) throws (Error) -> Item) throws (Error) -> Item | none;
    fn any(self, pred: fn(Item) throws (Error) -> bool) throws (Error) -> bool;
    fn all(self, pred: fn(Item) throws (Error) -> bool) throws (Error) -> bool;
    fn find(self, pred: fn(Item) throws (Error) -> bool) throws (Error) -> Item | none;
    fn position(self, pred: fn(Item) throws (Error) -> bool) throws (Error) -> usize | none;
    fn partition(self, pred: fn(Item) throws (Error) -> bool) throws (Error) -> (List<Item>, List<Item>);
    fn sum(self) throws (Error) -> Item where Item: Add<Output = Item> + Zero;
    fn product(self) throws (Error) -> Item where Item: Mul<Output = Item> + One;
    fn min(self) throws (Error) -> Item | none where Item: Ord;     // the first of equal minima
    fn max(self) throws (Error) -> Item | none where Item: Ord;     // the last of equal maxima
    fn min_by_key<K: Ord>(self, key: fn(Item) throws (Error) -> K) throws (Error) -> Item | none;  // the first of equals
    fn max_by_key<K: Ord>(self, key: fn(Item) throws (Error) -> K) throws (Error) -> Item | none;  // the last of equals
}
```

`TryIterator` 在 prelude 中（[0.2](../000-meta/002-import.md)）。它是獨立的 trait，而不是 `Iterator` 上的錯誤參數：一個值只會是其中之一。

- **擲回之後。** 擲回錯誤之後再呼叫 `next` 會怎樣，由各型別在文件中自行選擇。型別可以繼續給出下一個項目（`io::Bytes`），也可以結束，從此給出 `none`（`TryGenIter`）。
- **配接器的函式擲回 `Error`。** 傳給 `map`、`filter`、`fold` 等的函式可以擲回迭代器自身的 `Error`；不會擲回錯誤的函式也適用（[refs-lang 4.14.8](../../refs-lang/004-functions/414-throws.md#4148-函式型別中的-throws)）。擲回其他型別的函式在呼叫處是型別錯誤：先用 `.map_err(..)` 轉換，或在函式內以 `catch` 處理錯誤。`map_err` 是改變 `Error` 的唯一方式：`.map_err(|e| -> IoError | ParseError { e })` 加寬它，`.map_err(|e| -> any Error { e })` 抹除它。
- **沒有 `try_fold` / `try_for_each`。** `fold` 與 `for_each` 本來就接受會擲回錯誤的函式。`Iterator::try_fold` / `try_for_each`（2.7.1）仍然保留：它們為不會擲回錯誤的迭代器加上擲回。
- **`collect`** 建構任何 `FromIterator` 集合型別（2.7.5）；沒有另外的 trait。它在第一個錯誤處停止並擲回該錯誤。
- **命名。** 配接器型別加上 `Try` 前綴，與 `Map`、`Filter` 等一起位於 `core::iter`；`MapErr` 與 `Results` 只存在於這裡。
- **`for`。** `TryIterator` 以 `try for x in it { .. }` 步進（[refs-lang 3.13.3](../../refs-lang/003-expressions/313-loop-expressions.md#3133-for以迭代器為基礎的重複)）。`try` 涵蓋可迭代物與每次 `next`，不涵蓋主體。對 `TryIterator` 使用一般的 `for` 是編譯錯誤，錯誤會要求改用 `try for` 或 `.results()`。

### 轉換

| 來源 | 目標 | 寫法 |
|---|---|---|
| `TryIterator` | `Iterator<Item = Result<T, E>>` | `it.results()` |
| `IntoIterator<Item = Result<T, E>>` | `TryIterator<Item = T, Error = E>` | `iter::from_results(xs)`（`FromResults<I, T, E>`） |
| 一般的 `IntoIterator` | 永不擲回錯誤的 `TryIterator<Error = E>` | `iter::infallible::<E, J>(xs)`（`Infallible<J::IntoIter, E>`） |
| 函式 | `TryIterator` | `iter::try_from_fn(f)`，`f: fn() throws (E) -> (T \| none)`（`TryFromFn<T, E>`） |
| 產生器 | `TryIterator` | `yield fn .. throws (E) -> T` 給出 `TryGenIter<T, E>`（2.7.10a） |

`results()` 在錯誤之後仍繼續呼叫 `next`，所以會繼續的來源，其每個錯誤都會成為一個 `Err` 項目；而在錯誤之後結束的來源，最後給出的就是那個 `Err`。`iter::from_results` 擲回每個 `Err` 項目並繼續。

### 寫法

| 層次 | 寫法 |
|---|---|
| 手動步進 | `while let x: T = try it.next() { .. }` |
| 略過並繼續 | `for r in it.results() { match r { Result::Ok(x) => .., Result::Err(e) => .. } }` |
| 迴圈 | `try for x in it { .. }` |
| 管線 | `let ns: List<i64> = try lines.filter(\|l\| !l.is_empty()).map(\|l\| try i64::from_str(l) catch _ { 0 }).collect();` |
| 自行實作 | `yield fn .. throws (E) -> T { .. }`，或 `impl TryIterator for ..` |

```nabi
fn total(path: str) throws (IoError | ParseError) -> i64 {
    let n: i64 = 0;
    try for line in read_lines(path) {      // IoError from each next
        n += try i64::from_str(line);       // the body marks its own call
    }
    n
}
```

## 2.7.14. 集合型別回應迭代器方法

方法呼叫 `x.m(args)` 中，若接收者型別沒有方法 `m`，但有不接受引數的方法 `iter()`，且 `iter()` 回傳的型別有 `m`，則它就是 `x.iter().m(args)`。零引數的 `x.min()` / `x.max()` 也會轉送，因為 `Ord::min` / `max` 需要一個引數。其他情況則以接收者自身的方法優先（`List::contains`、`List::windows`、`Set::to_list`）。轉送的呼叫與 `iter()` 形式完全相同：惰性配接器仍是惰性的，也不做任何複製。

```nabi
let xs = [3, 1, 2];
let total = xs.sum();                          // xs.iter().sum()
let evens = xs.filter(|x| x % 2 == 0).to_list();
let big = xs.max();                            // xs.iter().max(), 3
let d = counts.map(|p| p.1).sum();             // a Dict iterates (k, v) pairs
```

沒有預期型別、之後也沒有用法能確定 `C` 的 `collect()`，會收集成 `List<Item>`（其型別參數的預設值）。

## 2.7.15. 實體化、排序、計數

- `to_list()`、`to_set()`、`to_dict()` 把項目收集到新的集合型別中；`to_dict` 接受 `(K, V)` 項目，後出現的鍵的值會取代先前的值。
- `sorted(reverse:)` 回傳一個新的已排序 `List`（穩定排序；`reverse: true` 把最大的放在最前面）。`sorted_by_key(key, reverse:)` 依 `key` 排序，鍵相等的項目在兩個方向上都保持原順序；`sorted_by(cmp)` 依比較函式排序。Python 的 `sorted(xs, key=f, reverse=True)` 就是 `xs.sorted_by_key(f, reverse: true)`；Nabi 把鍵的形式命名為 `_by_key`，與 `sort_by_key` 和 `min_by_key` 一致。
- `counts()` 把每個項目對應到它出現的次數，鍵依首次出現的順序排列（Python 的 `Counter`）；`Dict<K, usize>::most_common(n)` 為結果排名（[2.9.3](209-collections.md#293-dictk-v-s雜湊對應)）。`group_by(key)` 把每個鍵對應到其項目的串列，保持原順序；與 Python 的 `itertools.groupby` 不同，相等的鍵不必相鄰。
- `enumerate(start: 1)` 從 `start` 開始計數。
- `pairwise()` 產出每個項目與其下一個項目。`combinations(k)` 以串列產出每一種選取 `k` 個項目的組合，`permutations(k)` 產出 `k` 個項目在不同位置上的每一種排列（`k` 為 `none` 時取全部項目），兩者都在收集完項目之後，依位置的字典序產出（Python 的 `itertools`）。
- `min()` / `max()` 需要 `PartialOrd`，因此浮點數迭代器也有這兩個方法：與自身無法排序的項目（NaN）就是結果，與 IEEE 754 的 `minimum` / `maximum` 及 NumPy 相同。`min_by_key` / `max_by_key` 仍要求 `K: Ord`。
- `f32` / `f64` 項目的 `sum()` 採成對加總：每 128 個連續項目為一個區塊，由左至右相加；區塊總和以二進位計數器的方式合併（每個新總和與先前區塊數相同的總和合併，`left + right`），剩下的部分和則從最後一個加到第一個（`p0 + (p1 + (.. + pn))`）。順序只取決於項目數量，所以在任何目標、執行緒數量與最佳化等級下結果都相同。其他項目型別由左至右相加。

```nabi
let words = ["b", "a", "c", "a"];
words.sorted()                         //→ ["a", "a", "b", "c"]
words.sorted_by_key(|w| w.len(), reverse: true)
words.counts().most_common(1)          //→ [("a", 2)]
[1, 4, 9].pairwise().map(|p| p.1 - p.0).to_list()   //→ [3, 5]
["a", "b", "c"].combinations(2).count()             //→ 3
```
