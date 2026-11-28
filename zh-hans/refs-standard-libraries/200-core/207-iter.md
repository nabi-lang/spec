# 2.7. iter：迭代 trait

`core::iter` 模块是 `for x in xs` 的**脱糖依据**。每个 `for` 循环都分派到本章的 trait。

[refs-lang 3.13 循环表达式](../../refs-lang/003-expressions/313-loop-expressions.md) 中的 `for` 语法脱糖为 `IntoIterator::into_iter` 和 `Iterator::next` 调用的组合。

## 2.7.1. `Iterator`：步进

第一个 trait 是 `Iterator`，签名如下。

```nabi
#[lang("iterator_trait")]
trait Iterator {
    type Item;

    fn next(ref self) -> Item | none;
}
```

核心行为很简单：每次调用 `next` 返回下一个元素，没有更多元素时返回 `none`。

签名是 `ref self`：迭代器自身持有**状态**，每次调用时就地更新。迭代器是值：复制出的 `ListIter` 独立地继续，`next(ref self)` 只推进那一个副本。

### 示例（用户代码）

直接调用的样子如下。

```nabi
let xs: List<i32> = [10, 20, 30];
let it = xs.into_iter();         // iterator created by IntoIterator
let v1 = it.next();              //→ 10
let v2 = it.next();              //→ 20
let v3 = it.next();              //→ 30
let v4 = it.next();              //→ none
```

在实际代码中，`for` 比直接调用更常见。

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

### 方法：丰富的默认实现

用户为 `Iterator` 必须实现的方法只有 `next(ref self)`。其他所有方法都以 `next` 为基础，作为默认实现提供。

> [!NOTE]
> 下面签名中的 `fn(Item) -> U` 是 [refs-lang 4.4 函数类型](../../refs-lang/004-functions/404-function-type.md) 定义的统一可调用类型，**函数和闭包都可以接受**。`it.map(some_fn)` 和 `it.map(|x| x * 2)` 都适用于同一位置。这是单一的形式，不同于 Rust 的 `fn`/`Fn`/`FnMut`/`FnOnce`。

```nabi
impl Iterator {
    fn next(ref self) -> Item | none;       // user-defined

    // transforming adapters (lazy)
    fn map<U>(self, f: fn(Item) -> U) -> Map<Self, U>;
    fn filter(self, pred: fn(Item) -> bool) -> Filter<Self>;
    fn filter_map<U>(self, f: fn(Item) -> (U | none)) -> FilterMap<Self, U>;   // the f(item) that are not none
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

### `sum` / `product` 与 `Zero` / `One`

`sum` 从 `Zero::zero()` 开始用 `+` 累加；`product` 从 `One::one()` 开始用 `*` 累乘。因此空迭代器的和是 0，积是 1。`Zero` / `One` 是 `core::num` 的 trait（[2.1](201-numbers.md)）。

### `min` / `max` 与浮点数

`min` 和 `max` 需要 `PartialOrd`，所以浮点迭代器也有它们：与自身无序的项（NaN）就是结果（2.7.15）。`min_by_key` 和 `max_by_key` 需要 `K: Ord`，即全序；`f32` 和 `f64` 不是 `Ord`，因为 NaN 无序，浮点键是错误（E0069），其帮助信息给出 `reduce` 的写法。浮点列表按全序排序：`xs.sort_by(f64::total_cmp)`。

### 停止、运行状态、分组

- `take_while(pred)` 产出各项，直到 `pred` 第一次拒绝某一项。该项从内部迭代器取出后被丢弃；之后的项不再取出，所以停止点之后的副作用不会发生。`skip_while(pred)` 丢弃各项，直到 `pred` 第一次拒绝某一项，然后产出该项以及之后的所有项，不再检查。
- `scan(init, f)` 产出运行状态：`f(init, x0)`，然后是 `f` 作用于该结果和 `x1`，依此类推。它就是产出每一步的 `fold`，所以产出的项数与内部迭代器相同。
- `step_by(step)` 产出第一项，然后每隔 `step` 项产出一项。`step` 为 0 时触发 panic。
- `peekable()` 返回一个 `Peekable<Self>`，其 `peek(ref self) -> Item | none` 给出下一项，并把它留给接下来的 `next`。
- `inspect(f)` 把每一项交给 `f`，并原样产出。
- `cycle()` 每当耗尽时，就从 `cycle` 收到时的迭代器副本重新开始，所以它是无限的；只有迭代器为空时例外，此时什么也不产出。生成器的副本共享同一个位置（2.7.10a），所以对生成器使用 cycle 不会重复它。
- `chunks(size)` 依次产出含 `size` 项的列表，项用完时最后一个较短。`windows(size)` 产出含 `size` 个连续项、每次滑动一项的列表，项数少于 `size` 时什么也不产出。两者在 `size` 为 0 时都触发 panic。`List` 和 `Slice` 有自己产出视图的 `chunks` / `windows`（[2.9.7](209-collections.md#297-slicet视图)）。
- `try_fold` 和 `try_for_each` 接受会抛出错误的 `f`：第一个错误停止循环，之后的项不再访问，错误抛给调用方。
- `partition(pred)` 以两个列表返回 `pred` 接受的项和其余的项，各自保持顺序。`unzip()` 把由对 `(A, B)` 组成的迭代器拆成一个 `List<A>` 和一个 `List<B>`。它的项必须是对：`A` 和 `B` 从 `Item = (A, B)` 推断，其他形式的项在调用处报错。

```nabi
let xs = [1, 2, 3, 4];
let small: List<i32> = xs.iter().take_while(|x| x < 3).collect();   //→ [1, 2]
let sums: List<i32> = xs.iter().scan(0, |acc, x| acc + x).collect(); //→ [1, 3, 6, 10]
let pairs: List<List<i32>> = xs.iter().windows(2).collect();        //→ [[1, 2], [2, 3], [3, 4]]
let (even, odd) = xs.iter().partition(|x| x % 2 == 0);              //→ [2, 4], [1, 3]
let (ns, ss) = [(1, "a"), (2, "b")].iter().unzip();                 //→ [1, 2], ["a", "b"]
```

### 惰性与急切

这些方法分为两组。`map`、`filter`、`take` 等变换适配器是**惰性的**：它们只返回一个新的迭代器，不执行任何计算。只有调用 `count`、`sum`、`collect` 或 `for_each` 等终结方法时，才开始步进。

```nabi
let xs = [1, 2, 3, 4, 5];
let r = xs.into_iter()
    .map(|x| x * 2)            // lazy: nothing computed yet
    .filter(|x| x > 4)         // lazy
    .sum();                    // eager: stepping happens here
//→ 6 + 8 + 10 = 24
```

### 用户定义的 Iterator

直接定义 Iterator 的示例：

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

## 2.7.2. `IntoIterator`：`for` 的起点

接下来是 `IntoIterator` trait，它是 `for` 循环的起点。

```nabi
#[lang("into_iterator_trait")]
trait IntoIterator {
    type Item;
    type IntoIter: Iterator<Item = Item>;

    fn into_iter(self) -> IntoIter;
}
```

`IntoIterator` 是让集合类型创建迭代器的适配器。`for x in xs` 并不把 **xs 本身**当作迭代器；它用 `xs.into_iter()` 创建一个迭代器，并对它调用 `next`。

### 为什么二者分开

两个 trait 分开，是因为两种视角根本不同。

- 集合类型本身（List、Dict 等）是无状态的容器，不知道自己指向哪个位置。
- 迭代器带有进度状态，记录已经走到哪里。

因此同一个集合迭代两次，每次都需要一个新的迭代器，而 `into_iter` 就是创建它的地方。

### 自动实现

`Iterator` 本身自动获得 `IntoIterator` 的实现，其 `into_iter` 原样返回 self。

```nabi
// every Iterator is automatically an IntoIterator
impl<I: Iterator> IntoIterator for I {
    type Item = I::Item;
    type IntoIter = I;
    fn into_iter(self) -> I { self }
}
```

所以 `for x in iter` 可以直接使用：如果 iter 已经是 Iterator，`into_iter` 就返回 self。

### `for` 脱糖的确切形式

更准确地说，`for` 循环的脱糖如下：

```nabi
for x in expr {
    body
}
```

脱糖为：

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

详细的脱糖规则见 [refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md)。

## 2.7.3. `Step`：单步抽象

对 `Range<T>` 的迭代建立在“移到下一个值”这一简单操作之上。它在含义上不同于一般的算术（`Add`）：例如 `char` 的下一个值是码位 + 1，但 `char + char` 本身没有定义。因此 Nabi 把步进放在单独的 trait 中。

```nabi
// core::range::Step
trait Step: PartialOrd + Clone {
    fn forward_checked(self, count: usize) -> Self | none;
    fn backward_checked(self, count: usize) -> Self | none;
    fn steps_between(start: Self, end: Self) -> usize | none;
}
```

`Step` 位于 `core::range`，不在 prelude 中：用户写 `impl Step for T` 需要 `use core::range::Step;`。

各方法的含义：

- `forward_checked(self, n)`：返回从 `self` 向前 `n` 步的值；超出类型范围时返回 `none`。
- `backward_checked(self, n)`：向后移动后的值；超出范围时同样返回 `none`。
- `steps_between(start, end)`：计算从 `start` 到 `end` 有多少步；无法表示时（例如为负或溢出）返回 `none`。

### 为什么与算术 trait 分开

`Step` 与 `Add` 分开有三个原因。

- **含义分离**：`+1` 是简单的递增，而不是任意的算术。`MyVec3 + MyVec3` 有意义，但 `Range<MyVec3>` 没有；没有单独的 `Step` 实现时它会被拒绝。
- **排除浮点数**：`f32` / `f64` 没有 `Step` 实现，所以 `Range<f64>` 不是迭代器：`for x in 0.0..1.0` 是编译错误（E0068）。`0.0..1.0` 仍可用于判断成员关系（`x in 0.0..1.0`）。
- **高效跳过**：`Range`、`RangeInclusive` 和 `RangeFrom` 用一次 `forward_checked(_, n)` 调用重写了 `Iterator::nth(n)`，所以 `nth(n)` 和 `step_by(n)`（通过 `nth` 步进）在区间上以 O(1) 运行。如果只有 `succ`，同样的操作是 O(n)。

### Step 实现矩阵

标准库只为整数原始类型和 `char` 实现 `Step`。

| 类型                 | 实现     | 说明                                                |
|----------------------|----------|------------------------------------------------------|
| `i8`..`i64`、`isize` | **有**  | 溢出时 `forward_checked` 返回 `none`         |
| `u8`..`u64`、`usize` | **有**  | 下溢时 `backward_checked` 返回 `none`       |
| `int`                | **无**   | 没有标准库实现                             |
| `char`               | **有**  | 按码位步进，跳过代理项（`0xD800..=0xDFFF`） |
| `bool`               | **无**   | 没有标准库实现                             |
| `f32`、`f64`         | **无**   | `Range<f64>` 不是迭代器：步长有歧义     |
| `float`              | **无**   | （同上）                                      |

用户定义的类型可以自由添加 `impl Step for MyType`，这会立即使 `Range<MyType>` 可用。

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

[refs-lang 3.7 区间表达式](../../refs-lang/003-expressions/307-range-expressions.md) 中的 `a..b` 语法脱糖为这些标准库类型。

### `Range<T>`：`a..b`（不含终点）

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

每种区间形式都由 `core::range` 的一个 `#[literal]` 处理函数注册（构建 `Range` 的函数上带有 `#[literal(expr ".." expr)]`，每种形式一个处理函数），所以 `0..5` 这样的写法脱糖为 `Range` 结构体（以 [refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md) 为准）。编译器通过 `range_type` lang 槽找到 `Range`。

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

`start` 和 `end` 以外的字段（例如标记已到达终点的内部标志）属于实现细节，是私有的。即使 `end` 是该类型的最大值，`a..=b` 也恰好产出到 `b` 为止。

```nabi
for i in 0..=5 {
    process(i);          //→ 0, 1, 2, 3, 4, 5
}
```

### `RangeFrom<T>`：`a..`（无限）

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

`RangeFrom` 不产出该类型的最大值。由于最大值没有下一个值，迭代在到达最大值时、产出它之前结束（`(254u8..)` 只产出 `254` 就结束）。

### `RangeTo<T>` / `RangeToInclusive<T>` / `RangeFull`

```nabi
struct RangeTo<T> { pub end: T; }

struct RangeToInclusive<T> { pub end: T; }               // ..=b

struct RangeFull;
```

`..b` / `..=b`（无起点）和 `..`（两端都没有）只在切片或索引上下文中有意义，不创建迭代器。

```nabi
let xs = [1, 2, 3, 4, 5];
let s = xs[..3];          //→ [1, 2, 3]  (Slice: the Slice trait of 2.3 ops)
```

## 2.7.5. `FromIterator<T>`：`collect` 的分派

负责 `collect` 分派的 trait 是 `FromIterator<T>`。

```nabi
trait FromIterator<T> {
    fn from_iter<I: IntoIterator<Item = T>>(iter: I) -> Self;
}
```

调用 `Iterator::collect` 时，会调用目标集合的 `from_iter`。

```nabi
let v: List<i32> = (0..5).collect();
// = List::from_iter((0..5).into_iter())
```

每个集合类型（`List`、`Dict`、`Set`、`str` 等）都实现 `FromIterator`。

## 2.7.6. 各集合类型的 IntoIterator 实现

各集合类型实现 `IntoIterator` 的方式：

| 集合类型          | `Item` 类型 | 行为                                  |
|---------------------|-------------|-------------------------------------------|
| `List<T>`           | `T`         | 从第一个元素到最后一个，按顺序 |
| `Dict<K, V>`        | `(K, V)`    | 插入顺序                           |
| `Set<T>`            | `T`         | 插入顺序                           |
| `Tuple<A, B, C>`    | （特殊）   | **不创建迭代器**；使用模式匹配  |
| `Range<T>`          | `T`         | 从 start 到 end-1                       |
| `Slice<T>`          | `T`         | 从视图的起点到终点     |

### Dict / Set 的迭代顺序

`Dict<K, V>` 和 `Set<T>` 按插入顺序迭代，每次运行都相同：已存在的键再次插入时保持原位，删除后再插入的键排到最后。哈希种子改变的是键在表中的位置，而不是顺序（[2.9](209-collections.md)）。

## 2.7.7. 无限迭代器与 `take`

无限迭代器（例如 `RangeFrom` 或用户定义的迭代器）直接与急切方法组合会无限循环。

```nabi norun
(0..).count();         // infinite loop: count runs to the end
(0..).sum();           // infinite loop
```

用 `take(n)` 使它有限。

```nabi
(0..).take(100).count();     //→ 100
(0..).take(10).sum();        //→ 0+1+...+9 = 45
```

`Iterator` 不检测无限性。用户必须有意识地组合惰性适配器。

## 2.7.8. 函数广播：`f.(x)` 的脱糖

[refs-lang](../../refs-lang/003-expressions/303-broadcast-operators.md) 中的函数广播（`sin.(xs)`、`f.(a, b)`）脱糖为 `Iterator::map`，没有自己的 trait。

```nabi
sin.(xs)   ≡   xs.into_iter().map(sin).collect()
g.(a, b)   ≡   a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

## 2.7.9. `DoubleEndedIterator`：双向步进

可以双向步进的迭代器由 `DoubleEndedIterator` trait 建模。

```nabi
trait DoubleEndedIterator: Iterator {
    fn next_back(ref self) -> Item | none;

    // provided
    fn rfold<B>(self, init: B, f: fn(B, Item) -> B) -> B;     // fold from the back
    fn rfind(self, pred: fn(Item) -> bool) -> Item | none;    // the last item pred accepts
}
```

这个 trait 表示可以从后端步进的迭代器；`rev()` 适配器、`rfold` 和 `rfind` 需要它。

`List<T>` 和 `Slice<T>` 的迭代器以及下列类型必须实现这个 trait。

- `Range<T>`、`RangeInclusive<T>`
- `Rev<I>`（`I` 为双向时）、`Map<I, U>`、`Filter<I>`、`FilterMap<I, U>`、`Chain<A, B>`、`InspectIter<I>`（组成它们的迭代器为双向时）
- `Enumerate<I>`、`Zip<A, B>`、`Take<I>`、`Skip<I>`、`StepBy<I>`：组成它们的迭代器是 `DoubleEndedIterator + ExactSizeIterator` 时，因为从后端计数需要知道长度。

`Dict<K, V>` 和 `Set<T>` 不实现它。

```nabi
let r: List<i32> = (0..5).rev().collect();    //→ [4, 3, 2, 1, 0]
```

## 2.7.10. `ExactSizeIterator`：长度已知的迭代器

在步进前就知道元素个数的迭代器由 `ExactSizeIterator` 表示。

```nabi
trait ExactSizeIterator: Iterator {
    fn len(self) -> usize;
}
```

`len()` 是剩余元素的个数。`Iterator::exact_len()` 在无需步进就能知道该个数时返回它，否则返回 `none`（默认）；标准库的 `ExactSizeIterator` 类型返回它们的 `len()`，下面列出的适配器建立在它们之上时也是如此。默认的 `count()` 在 `exact_len()` 不为 `none` 时返回它，所以 `count()` 不调用 `map` 的函数：`xs.iter().map(f).take(3).count()` 调用 `f` 零次。`scan` 和 `inspect` 的函数同样如此。长度未知的迭代器，例如 `Filter`、`TakeWhile`、`SkipWhile` 或 `(0..).take(100)`，会被步进。`List`、`Slice` 和 `Range` 的迭代器，`List` 和 `Slice` 的 `windows` / `chunks` 视图，以及组成迭代器全是 `ExactSizeIterator` 时的 `Enumerate`、`Zip`、`Take`、`Skip`、`StepBy`、`Rev`、`Map`、`Scan`、`InspectIter`、`Chunks`，都实现这个 trait。`Windows` 在定长迭代器之上有 `exact_len`，而 `Cycle` 只在其迭代器为空时有 `exact_len`（0）。

## 2.7.10a. `GenIter<T>` / `TryGenIter<T, E>`：生成器的值

它们是 `yield fn` 调用和 `yield { .. }` 块的值（[refs-lang 4.1.11](../../refs-lang/004-functions/401-function-declaration.md#协程函数的结果类型)）。

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

`TryGenIter` 是 `TryIterator`（[2.7.13](#2713-tryiterator可能失败的迭代)），而不是 `Iterator`，因为它的 `next` 会抛出错误。它用 `try for`、`try g.next()` 或它的适配器步进。抛出错误后生成器就结束了：下一次 `next` 给出 `none`。私有字段不作规定。副本共享同一个协程帧，因此共享进度位置。

## 2.7.11. 模块对应关系

各 trait 与其 lang 槽及标准库位置的对应关系：

| 项                  | lang 槽             | 标准库位置 |
|-----------------------|-----------------------|---------------------------|
| `Iterator` trait      | `iterator_trait`      | `core::iter`         |
| `IntoIterator` trait  | `into_iterator_trait` | `core::iter`         |
| `Step` trait          | （无）                | `core::range`        |
| `FromIterator` trait  | （无）                | `core::iter`         |
| `DoubleEndedIterator` | （无）                | `core::iter`         |
| `ExactSizeIterator`   | （无）                | `core::iter`         |
| `TryIterator` trait   | `try_iterator_trait`  | `core::iter`         |
| `Range<T>`            | `range_type`          | `core::range`        |
| 其余 5 个区间类型 | （无）              | `core::range`        |

原始类型本身没有 `Iterator` 实现（i32 不能作为 `for` 的目标）。只有集合类型和 Range 可以。

## 2.7.12. async：`Stream` trait

异步迭代器见 [2.8 stream](208-stream.md)。`for await x in stream` 语法（[refs-lang 3.13](../../refs-lang/003-expressions/313-loop-expressions.md)）分派到 `Stream::poll_next`。

## 2.7.13. `TryIterator`：可能失败的迭代

`next` 可能失败的迭代器是 `TryIterator`。它的错误是 `next` 的一个类型化通道，而不是项的一部分：`next` 给出一项、在结尾给出 `none`，或者抛出 `Error`。

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

`TryIterator` 在 prelude 中（[0.2](../000-meta/002-import.md)）。它是一个单独的 trait，而不是 `Iterator` 上的错误参数：一个值要么是这个，要么是那个。

- **抛出错误之后**。抛出错误后的 `next` 做什么，由各类型在文档中选择。类型可以继续给出下一项（`io::Bytes`），也可以结束，此后一直给出 `none`（`TryGenIter`）。
- **适配器函数抛出 `Error`**。传给 `map`、`filter`、`fold` 等的函数可以抛出迭代器自身的 `Error`；不抛出错误的函数也适用（[refs-lang 4.14.8](../../refs-lang/004-functions/414-throws.md#4148-函数类型中的-throws)）。抛出其他类型的函数在调用处是类型错误：先用 `.map_err(..)` 转换，或在函数内用 `catch` 处理错误。`map_err` 是改变 `Error` 的唯一方式：`.map_err(|e| -> IoError | ParseError { e })` 扩大它，`.map_err(|e| -> any Error { e })` 擦除它。
- **没有 `try_fold` / `try_for_each`**。`fold` 和 `for_each` 已经接受会抛出错误的函数。`Iterator::try_fold` / `try_for_each`（2.7.1）仍然保留：它们为不抛出错误的迭代器加上抛出。
- **`collect`** 构建任何 `FromIterator` 集合类型（2.7.5）；没有单独的 trait。它在第一个错误处停止并抛出该错误。
- **命名**。适配器类型带 `Try` 前缀，与 `Map`、`Filter` 等一起位于 `core::iter`；`MapErr` 和 `Results` 只存在于这里。
- **`for`**。`TryIterator` 用 `try for x in it { .. }` 步进（[refs-lang 3.13.3](../../refs-lang/003-expressions/313-loop-expressions.md#3133-for基于迭代器的重复)）。`try` 覆盖可迭代对象和每次 `next`，而不覆盖循环体。对 `TryIterator` 使用普通的 `for` 是编译错误，错误会提示改用 `try for` 或 `.results()`。

### 转换

| 从 | 到 | 写法 |
|---|---|---|
| `TryIterator` | `Iterator<Item = Result<T, E>>` | `it.results()` |
| `IntoIterator<Item = Result<T, E>>` | `TryIterator<Item = T, Error = E>` | `iter::from_results(xs)`（`FromResults<I, T, E>`） |
| 普通的 `IntoIterator` | 从不抛出的 `TryIterator<Error = E>` | `iter::infallible::<E, J>(xs)`（`Infallible<J::IntoIter, E>`） |
| 函数 | `TryIterator` | `iter::try_from_fn(f)`，`f: fn() throws (E) -> (T \| none)`（`TryFromFn<T, E>`） |
| 生成器 | `TryIterator` | `yield fn .. throws (E) -> T` 给出 `TryGenIter<T, E>`（2.7.10a） |

`results()` 在出错后继续调用 `next`，所以会继续运行的源的每个错误都变成一个 `Err` 项，出错后就结束的源则最后给出那个 `Err`。`iter::from_results` 抛出每个 `Err` 项并继续。

### 写法

| 层次 | 写法 |
|---|---|
| 手动步进 | `while let x: T = try it.next() { .. }` |
| 跳过并继续 | `for r in it.results() { match r { Result::Ok(x) => .., Result::Err(e) => .. } }` |
| 循环 | `try for x in it { .. }` |
| 管道 | `let ns: List<i64> = try lines.filter(\|l\| !l.is_empty()).map(\|l\| try i64::from_str(l) catch _ { 0 }).collect();` |
| 实现一个 | `yield fn .. throws (E) -> T { .. }`，或 `impl TryIterator for ..` |

```nabi
fn total(path: str) throws (IoError | ParseError) -> i64 {
    let n: i64 = 0;
    try for line in read_lines(path) {      // IoError from each next
        n += try i64::from_str(line);       // the body marks its own call
    }
    n
}
```

## 2.7.14. 集合类型响应迭代器方法

对于方法调用 `x.m(args)`，如果接收者类型没有方法 `m`，但有一个不带参数的方法 `iter()`，且 `iter()` 返回的类型有 `m`，那么该调用就是 `x.iter().m(args)`。零参数的 `x.min()` / `x.max()` 也会被转发，因为 `Ord::min` / `max` 需要一个参数。其他情况下，接收者自己的方法优先（`List::contains`、`List::windows`、`Set::to_list`）。转发的调用与 `iter()` 形式完全相同：惰性适配器保持惰性，不复制任何内容。

```nabi
let xs = [3, 1, 2];
let total = xs.sum();                          // xs.iter().sum()
let evens = xs.filter(|x| x % 2 == 0).to_list();
let big = xs.max();                            // xs.iter().max(), 3
let d = counts.map(|p| p.1).sum();             // a Dict iterates (k, v) pairs
```

没有期望类型、之后也没有确定 `C` 的用法时，`collect()` 收集为 `List<Item>`（其类型参数的默认值）。

## 2.7.15. 物化、排序、计数

- `to_list()`、`to_set()`、`to_dict()` 把各项收集到一个新的集合中；`to_dict` 接受 `(K, V)` 项，后出现的键的值替换先出现的。
- `sorted(reverse:)` 返回一个新的已排序 `List`（稳定排序；`reverse: true` 把最大的放在最前）。`sorted_by_key(key, reverse:)` 按 `key` 排序，在两个方向上键相等的项都保持原有顺序；`sorted_by(cmp)` 按比较函数排序。Python 的 `sorted(xs, key=f, reverse=True)` 即 `xs.sorted_by_key(f, reverse: true)`；Nabi 像 `sort_by_key` 和 `min_by_key` 一样，把按键的形式命名为 `_by_key`。
- `counts()` 把每一项映射到它出现的次数，键按首次出现的顺序排列（Python 的 `Counter`）；`Dict<K, usize>::most_common(n)` 对结果排名（[2.9.3](209-collections.md#293-dictk-v-s哈希映射)）。`group_by(key)` 把每个键映射到它的项组成的列表，保持顺序；与 Python 的 `itertools.groupby` 不同，相等的键不必相邻。
- `enumerate(start: 1)` 从 `start` 开始计数。
- `pairwise()` 把每一项与下一项一起产出。`combinations(k)` 把每种选取 `k` 项的组合作为列表产出，`permutations(k)` 产出在不同位置上 `k` 项的每种排列（`k` 为 `none` 时取全部项），两者都在收集各项之后按位置的字典序产出（Python 的 `itertools`）。
- `min()` / `max()` 需要 `PartialOrd`，所以浮点迭代器也有它们：与自身无序的项（NaN）就是结果，与 IEEE 754 的 `minimum` / `maximum` 和 NumPy 相同。`min_by_key` / `max_by_key` 仍要求 `K: Ord`。
- `f32` / `f64` 项的 `sum()` 采用成对求和：每 128 个连续项为一块，从左到右相加；各块的和按二进制计数器的方式合并（每个新的和与之前块数相同的和合并，`left + right`），剩余的部分和从最后一个到第一个相加（`p0 + (p1 + (.. + pn))`）。顺序只取决于项数，所以结果在每个目标、线程数和优化级别下都相同。其他项类型从左到右相加。

```nabi
let words = ["b", "a", "c", "a"];
words.sorted()                         //→ ["a", "a", "b", "c"]
words.sorted_by_key(|w| w.len(), reverse: true)
words.counts().most_common(1)          //→ [("a", 2)]
[1, 4, 9].pairwise().map(|p| p.1 - p.0).to_list()   //→ [3, 5]
["a", "b", "c"].combinations(2).count()             //→ 3
```
