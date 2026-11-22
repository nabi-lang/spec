# 3.6. 索引與切片

以方括號（`[`、`]`）括起來的運算式以兩種形式使用，用來存取值的一部分：索引，取出指定位置的元素；以及切片，切出一個區域。

```
index_expr      ::= primary "[" part ("," part)* ","? "]"
part            ::= expr | slice_arg
slice_arg       ::= expr? ":" expr? (":" expr?)?
```

只容納一個 `slice_arg` 的方括號是切片（3.6.2）；其他每個方括號都是索引，以逗號分隔的每個部分是一個引數（3.6.1、3.6.3）。

## 3.6.1. 索引：`arr[i]`

索引在方括號內寫一個索引運算式，以取得該位置的值。

```nabi
let v = arr[0];
let last = arr[arr.len() - 1];
let m = matrix[i, j];        // multidimensional, indices separated by commas
```

左側可以是任何運算式；方括號內放一個或多個以逗號分隔的索引運算式。結果是索引所指位置的值。

### 去糖

索引運算式去糖為下列 trait 呼叫。

```
arr[i]                 →   <typeof(arr) as Index<typeof(i)>>::index(arr, i)
arr[i] = v;            →   <typeof(arr) as IndexSet<typeof(i)>>::index_set(ref arr, i, v)
matrix[i, j]           →   <typeof(matrix) as Index<(typeof(i), typeof(j))>>::index(matrix, (i, j))
```

具名的最後一個部分 `default:` 給出帶預設值的索引 `d[k, default: v]`：

```
d[k, default: v]                  →   d.get_or(k, v)                      // a read; d unchanged
d[k, default: v] = e;             →   { let t = d; let i = k; t.get_or_insert(i, v); t[i] = e; }
d[k, default: v] op= e;           →   the same, with t[i] op= e
d[k, default: v].f op= e;         →   the same, with t[i].f op= e
d[k, default: v].m(args)          →   d.get_or_insert(k, v).m(args)
```

`d` 與 `k` 只求值一次；`v` 則在每次執行該運算式時求值。型別只要有 `get_or` 與 `get_or_insert` 就能使用這種形式（`Dict`，[stdlib 2.9.13](../../refs-standard-libraries/200-core/209-collections.md#2913-預設值運算子與建構函式)）。

讀取與寫入對應到不同的 trait，因此嘗試寫入唯讀資料是編譯錯誤。多維索引去糖為單一的索引 trait 呼叫，其索引被打包成一個元組：`g[i, j]` 是透過 `Index<(I, J)>` 的 `g[(i, j)]`。當一個型別有多個 `Index<K>` impl 時，由引數型別選出一個。

### 負索引

對於以位置索引的型別（`List`、`Slice`、固定長度陣列 `[T; N]`，以及 `str` / `StrBuf` 的切片邊界），負的**有號整數**索引從末尾算起。實際位置是 `len + i`。

這是關於索引之值的規則，而不是關於寫法的規則：在執行時計算出的負值和字面值一樣從末尾算起，也沒有另外的「從末尾算起」形式。在 `i = 0` 時，`xs[i - 1]` 是最後一個元素，而不是 panic。

```nabi
let xs = [10, 20, 30];
xs[-1];                // 30
let i = -2;
xs[i];                 // 20 (variables too, not just literals)
let j = 0;
xs[j - 1];             // 30 (a computed -1 is the last element)
xs[-4];                // panic: index -4 out of bounds for length 3
```

- 無號整數索引就是原樣的位置，不需要為從末尾算起付出額外成本。
- 無型別的負字面值索引是 `isize`。
- 若 `len + i < 0`，會 panic（`index -4 out of bounds for length 3`）。超過末尾的非負索引也會 panic（`xs[3]`）。
- 不得從末尾算起的索引要使用無號型別（`usize`），或在使用前先檢查。
- 以鍵索引的型別（`Dict`）不受影響。負的鍵就只是一個鍵。
- 轉換使用 `core::index::index_from_end(i: isize, len: usize) -> usize`。

讀取時先對目標求值，再對索引求值；目標只求值一次。指派則先對值求值，再對目標與索引求值（[3.4.7](304-assignment.md#347-求值順序)）。

### 邊界檢查與失敗

方括號索引在有效範圍之外可能失敗；失敗如何表現由 `Index` 的實作決定。通常是 panic 或錯誤。

要在不 panic 的情況下得到 `T | none` 值，請使用函式庫提供的方法，例如 `.get(i)`。

## 3.6.2. 切片：`arr[start:end]`

切片是方括號內含有帶冒號之切片形式時所使用的形式。

```nabi
let part = arr[1:3];
let head = arr[:5];
let tail = arr[5:];
let all = arr[:];
let stepped = arr[::2];
let reverse = arr[::-1];
let full = arr[a:b:s];
let same = arr[1..3];        // a range also works (same as arr[1:3])
let whole = arr[..];         // same as arr[:]
```

方括號內的切片形式依下列 BNF 作為切片處理。

```
slice_expr  := expr? ':' expr? (':' expr?)?
              | range_expr                      // arr[a..b], arr[..]
              | expr ',' expr ',' ...           // a comma means indexing
```

基本形式 `[start : end]` 表示從 start（含）到 end（不含）的區域。可以加上步長：`[start : end : step]`。其語意**與 Python 的切片相同**。

- 邊界會被夾在長度之內。無論以哪種形式寫出邊界（`a:b` 或範圍 `a..b`），切片絕不會因範圍而 panic。`xs[5:100]` 盡可能多取；`start >= end`（正步長）得到空的結果。
- 負的有號整數邊界從末尾算起（`len + i`），然後再夾住。`xs[-10:]` 是全部；`xs[:-10]` 是空的。
- 正步長時，省略的 `start` 是 0，省略的 `end` 是長度。省略的步長是 1。
- 負步長時，邊界的讀法與 Python 相同：從 `start` 往回走到 `end` 之前為止。省略的 `start` 是最後一個元素；省略的 `end` 是第一個元素之前。
- 步長為 0 時會 panic（`slice step must not be zero`）。

```nabi
let xs = [1, 2, 3, 4, 5];
xs[-2:];               // [4, 5]
xs[1:-1];              // [2, 3, 4]
xs[5:100];             // [] (clamp)
xs[-10:];              // [1, 2, 3, 4, 5]
xs[:-10];              // []
xs[3:1];               // [] (start >= end)
xs[1..10];             // [2, 3, 4, 5] (a range clamps too)
xs[3..1];              // []
"abc"[1..10];          // "bc"
xs[::-1];              // [5, 4, 3, 2, 1]
xs[-1:0:-1];           // [5, 4, 3, 2]
xs[3:1:-1];            // [4, 3] (positions 3, 2)
xs[1:4:-2];            // []
xs[::0];               // panic: step 0
```

單一元素的索引（3.6.1）不同：負數從末尾算起，但超出範圍會 panic。兩條規則都是刻意的：索引指名一個元素，該元素必須存在；切片指名一個區域，該區域可以是空的。

`str` / `StrBuf` 的邊界是位元組偏移；它們先被夾住，若所得位置落在 UTF-8 字元的中間，就會 panic。`str` 的切片沒有步長（`s[::2]` 是錯誤，E0056）。

### 去糖

切片去糖為一個範圍以及對 `Sliceable` trait 的呼叫（[stdlib 2.3.4](../../refs-standard-libraries/200-core/203-ops.md#234-索引-trait)）。

```
arr[a : b]            →   arr.slice(a..b)                       // unsigned bounds, no step
arr[a : b]            →   arr.slice(slice_bound(a, n)..slice_bound(b, n))   // signed bounds
arr[a : b : k]        →   arr.slice(slice_range(a, b, k, n)).step(k)
arr[: b] / arr[a :]   →   omitted bounds use the defaults above (none for slice_range when a step is present)
arr[a..b] / arr[..]   →   arr.slice(a..b) / arr.slice(0..n)
arr[a : b]            →   slice_mut(arr, a..b)                  // a List where a SliceMut is expected
```

`n` 是 `arr.len()`，而 `arr` 只求值一次。每個 `Sliceable<Range<usize>>` 實作都以 `clamp_range` 夾住範圍，因此直接呼叫 `arr.slice(9..2)` 也不會 panic。所使用的輔助函式位於 `core::index`；它們是 `pub(pkg)`，由降階後的程式碼呼叫，不屬於 API（從使用者程式碼呼叫是 E0034）。

| 輔助函式                                                       | 意義                                                               |
|----------------------------------------------------------------|--------------------------------------------------------------------|
| `clamp_range(r: Range<usize>, n: usize) -> Range<usize>`       | 把終點夾在 `n`，起點夾在終點                                       |
| `slice_bound(i: isize, len: usize) -> usize`                   | 為負時是 `len + i`；低於 0 時變成 0                                |
| `slice_range(start: isize \| none, stop: isize \| none, step: isize, len: usize) -> Range<usize>` | 等價於 Python `slice.indices` 的正向範圍。附加 `.step(step)` 後得到 `arr[start:stop:step]` |
| `index_from_end(i: isize, len: usize) -> usize`                | 用於單一索引。為負時是 `len + i`；溢位時 panic                     |
| `slice_mut(xs: List<T>, r: Range<usize> \| none) -> SliceMut<T>` | `xs[r]` 的可寫視圖（`none`：整個 `xs`），`r` 會被夾住；僅供降階後的程式碼使用 |

### `Slice` 結果的種類

結果型別由 `Sliceable` 的關聯型別決定。在標準函式庫中如下（[stdlib 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet檢視)）。

| 目標                 | 結果                                                                 |
|----------------------|----------------------------------------------------------------------|
| `List<T>`            | `Slice<T>`：串列緩衝區的視圖（元素的寫入看得到）                     |
| `Slice<T>`           | `Slice<T>`：視圖的視圖                                               |
| `[T; N]`             | `Slice<T>`：陣列複本的視圖（陣列是值）                               |
| 在預期 `SliceMut<T>` 之處的 `List<T>` | `SliceMut<T>`：串列的可寫視圖（[stdlib 2.9.7a](../../refs-standard-libraries/200-core/209-collections.md#297a-slicemutt可寫入檢視)）；沒有步長 |
| `SliceMut<T>`        | `SliceMut<T>`：該視圖的可寫視圖                                      |
| `str`、`StrBuf`      | 新的 `str`：位元組複本（因為 `str` 必須保持為 C 字串），並檢查 UTF-8 邊界 |

### 切片指派

切片是位置：`arr[a:b:k] = v;` 透過它存入。

```
xs[a : b] = ys;       →   { let t = xs; slice_mut(t, a..b).copy_from_slice(ys[:]) }   // a List
m[a : b : k] = v;     →   IndexSet::index_set(m, SliceArg::new(a, b, k), v)           // an AxisIndexed type (3.6.3)
```

- 在 `List` 上，值來自長度完全相同的 `List` / `Slice`（否則 `copy_from_slice` 會 panic；改變長度要用 `splice`）。串列帶步長的切片不能被指派（錯誤，說明會提示逐一存入元素）。
- 在 `Array` 這類以軸索引的型別上，選取的部分是視圖，而值要麼是填滿它的純量（`a[1:3] = 0.0`），要麼是伸展為其形狀的陣列（`a[1:3] = b`），取決於哪個 `IndexSet` impl 接受該值的型別。
- `str`、`StrBuf`、`Slice` 與固定長度陣列不接受切片指派（錯誤）；`SliceMut` 視圖則逐元素寫入，或以 `copy_from_slice` 寫入。

## 3.6.3. 多軸索引

方括號中以逗號分隔的每個部分是一個引數，而部分可以是切片：`m[1:, ::2]`、`m[i, :]`。各部分以一個元組透過 `Index` / `IndexSet`（3.6.1）傳遞，切片部分以值 `core::index::SliceArg { start, stop, step }` 傳遞（各為 `isize | none`，缺少的部分為 `none`）：

```
m[1:, ::2]          →   Index::index(m, (SliceArg::new(1, none, none), SliceArg::new(none, none, 2)))
m[i, :]             →   Index::index(m, (i, SliceArg::FULL))
m[1:3, :] = x       →   IndexSet::index_set(m, (SliceArg::new(1, 3, none), SliceArg::FULL), x)
```

`SliceArg::indices(len)` 對一個軸套用 3.6.2 的規則，並回傳 `(first, count, step)`。

實作了標記 trait `core::index::AxisIndexed` 的型別（`Array` 有實作，[stdlib 2.9.14](../../refs-standard-libraries/200-core/209-collections.md#2914-arrayt-const-n-usizen-維陣列)），會把每個部分當作*軸引數*接收，因此一組針對 `isize` 的 impl 就能服務索引的每一種寫法：

| 部分                        | 軸引數                           | 意義（對 `Array` 而言）                       |
|-----------------------------|----------------------------------|-----------------------------------------------|
| 任何型別的整數              | `isize`（字面值就是 `isize`；型別只由字面值給出的變數，例如 `for i in 0..4` 的 `i`，是其 `i64`；`usize`、`u64`、`i64` 與 `u32` 會被檢查，其他型別則放寬） | 一個位置；該軸被去掉；負數從末尾算起（3.6.1） |
| `a:b:k`                     | `SliceArg`                       | 該軸帶步長的範圍（3.6.2 的規則）              |
| 與其他部分並列的 `..`       | `core::index::Rest`         | 其他部分留下的每個軸，整個                     |
| `none`                      | `core::index::NewAxis`      | 長度為 1 的新軸                               |
| 其他任何東西（遮罩）        | 原樣                             | 由 impl 決定（`Array<bool, N>`：遮罩）        |

```nabi
use core::array::Array;

let a: Array<f64, 2> = [[1.0, 2.0, 3.0], [4.0, 5.0, 6.0]];
a[1, :]          // [4.0, 5.0, 6.0]: row 1, a view
a[:, -1]         // [3.0, 6.0]: the last column
a[::-1, 1:]      // rows reversed, columns 1..
a[.., 0]         // [1.0, 4.0]
a[:, none]       // shape [2, 1, 3]
a[a .> 2.0]      // [3.0, 4.0, 5.0, 6.0]: a mask selects a copy
a[:, 0] = 0.0;   // fills a column
```

單獨的 `..`（`a[..]`）仍是整個範圍（3.6.2）。不是 `AxisIndexed` 的型別以各部分自己的型別接收它們（整數字面值取其預設型別），由其 `Index` impl 決定接受什麼。

在沿單一軸索引的型別（`List`、`Slice`、固定長度陣列、`str`）上，部分之中含有切片的多部分索引是錯誤（E0143），說明會提示在分開的方括號中切片與索引（`m[1:][0]`），或把資料放在 `Array` 中。

## 3.6.4. 空的方括號與格式錯誤的形式

下列全都是格式錯誤的。

```
arr[]               // ERROR: empty brackets
arr[a:b:c:d]        // ERROR: three or more colons
arr[: : :]          // ERROR: three colons
```

這些形式在語彙與語法階段就會被拒絕。

## 3.6.5. 索引與切片取決於 `trait`

語言本身不決定哪些型別支援索引或切片。只有在標準函式庫為某個型別實作了 `Index`、`IndexSet` 與 `Sliceable` 之後，`arr[i]` 與 `arr[1:3]` 才能對該型別運作。

語言定義方括號語彙單元的形式與去糖規則；意義屬於函式庫。
