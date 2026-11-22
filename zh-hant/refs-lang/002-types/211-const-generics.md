# 2.11. `const` 泛型

`const` 泛型在[泛型](210-generics.md)參數位置接受一個值。在編譯期確定的整數、布林值與簡短運算式都可以作為參數。

## 2.11.1. 動機

它們把編譯期確定的值（陣列長度、矩陣維度、嵌入式記憶體池大小）作為型別的一部分攜帶。

```nabi
struct Matrix<T, const M: usize, const N: usize> {
    data: [T; M * N];
}

fn dot<T: Num, const N: usize>(a: [T; N], b: [T; N]) -> T { ... }
```

由於陣列長度成為型別的一部分，呼叫時的長度檢查不需要任何成本。編譯器也能套用取決於長度的最佳化，例如 SIMD 與展開，而矩陣乘法這類維度相容性很重要的運算可以在編譯期驗證。

## 2.11.2. 宣告形式

```nabi
const ParamName : Type
```

`const` 關鍵字之後依序接參數名稱、`:` 與型別。型別必須是下文所述允許的 `const` 型別之一。

多個參數以逗號分隔。型別參數與 `const` 參數可以自由混用。

```nabi
fn replicate<T, const N: usize>(value: T) -> [T; N] { ... }
struct Tagged<T, const TAG: u32> { value: T; }
```

## 2.11.3. 允許的型別

`const` 參數的型別是整數型別之一（`i8` 到 `i64`、`u8` 到 `u64`、`usize`、`isize`）、`bool`、`char` 或 `c_char`。

浮點數型別、`str`、和型別與使用者定義的結構不能作為 `const` 參數；這樣的參數在其宣告處就是錯誤。浮點數型別被排除，是因為其位元表示具有非確定性；使用者定義的型別被排除，則是因為相等比較的一致性問題。

## 2.11.4. 實例化時的引數

在呼叫處或實例化處，為 `const` 參數傳入一個 `const` 運算式。

```nabi
let m: Matrix<f64, 3, 4> = ...;
let v: [i32; 16] = ...;
let arr = replicate::<f64, 8>(0.0f64);
```

引數必須能在編譯期求值。允許的有字面值、`const` 變數、`const` 函式呼叫、其他 `const` 參數，以及它們的簡單算術組合。執行時的變數不能用作 `const` 引數；那樣做是編譯錯誤。引數的型別必須是參數的型別：對 `const X: usize` 而言，`S<true>` 與 `S<-1>` 是錯誤。

在泛型引數清單中，單獨的引數是字面值、取負的字面值（`-1`）或名稱（`N`、某個 `const` 變數）。其他任何東西（包括算術）都要以大括號包起來：

```nabi
fn grow<const N: usize>(b: Buf<N>) -> Buf<{ N + 1 }> { ... }
let b: Buf<{ 2 * 8 }> = ...;
```

陣列長度（`[T; M * N]`）本來就是運算式位置，不需要大括號。

## 2.11.5. `const` 運算式中的算術

型別內可以使用 `const` 參數之間的簡單算術。

```nabi
struct Matrix<T, const M: usize, const N: usize> {
    data: [T; M * N];
}

fn flatten<T, const M: usize, const N: usize>(
    m: Matrix<T, M, N>
) -> [T; M * N] { ... }
```

允許的運算子是 `+`、`-`、`*`、`/`、`%` 與一元 `-`。目前的規格不允許比較與條件；只允許簡單算術。型別系統確實能辨識 `M * N` 與 `N * M` 是相同的值（交換律）。

更複雜的 `const` 運算式，例如條件或遞迴，由 `const` 函式規格涵蓋。

## 2.11.6. 參數推論

若 `const` 引數能從引數型別、結構欄位型別或預期型別（`let` 標註、參數型別、回傳型別）推論出來，就可以省略。

```nabi
fn replicate<T, const N: usize>(value: T) -> [T; N] { ... }

let arr: [i32; 8] = replicate(0i32);     // N = 8 inferred
let arr2 = replicate::<i32, 8>(0i32);    // explicit
```

推論的方式與型別參數相同（[2.10](210-generics.md)）：每個引數與預期型別都有貢獻，而當有多個來源確定 `N` 時，它們必須一致，否則呼叫就是型別錯誤。沒有任何東西確定的 `const` 引數必須寫出（`replicate::<i32, 8>(0i32)`）；省略它是編譯錯誤。沒有預設值，各個來源之間也沒有優先順序。

標頭可以從參數計算出引數：`impl<const A: i32> Root for Q<{ 2 * A }>` 以 `A = 1` 適用於 `Q<2>`，而不適用於 `Q<3>`。當參數只出現一次、只經過 `+`、`-` 與乘以常數以及取負，而且值恰好能精確求出時，參數才會被解出；其他任何標頭運算式都不適用於任何型別。

### 重複字面值 `[v; N]`

`[v; N]` 是由 `N` 個元素組成的 `[T; N]` 陣列，每個元素都是 `v` 的複本。`N` 是 `const` 運算式，可以是 `const` 參數。

`v` 恰好被求值一次，在寫入任何元素之前，因此其副作用只發生一次（`N` 為 0 時也是如此）。之後每個元素都是各自獨立的值：

- 若 `T` 是 `Copy`，或是其中沒有任何東西參照共享儲存空間的值型別（數字、`bool`、`char`、`str`、`none`、函式，以及由這些組成的陣列、元組、結構與列舉），每個元素都是 `v` 的逐位元複本，不會呼叫 `clone`。
- 否則元素 `0` 到 `N - 2` 是 `v.clone()`，依索引順序呼叫，最後一個元素是 `v` 本身。`List` 元素、串列的陣列，或存放 `Dict` 的結構，會產生 `N` 個各自獨立的集合，因此透過一個元素寫入不會改變其他元素。
- 若 `T` 不是 `Clone`，該字面值是錯誤（E0069），其說明會寫出 `array::from_fn::<T, N>(|i| ..)`（[stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md)），它以各自的值建立每個元素；`List` 重複字面值的說明則寫出 `List::from_fn(N, |_| ..)`。型別參數需要 `T: Clone` 約束。
- 若 `T` 存放只有一個擁有者的值（必須關閉的型別，例如 `File`，或 `Drop` 型別，在任何陣列深度），該字面值是錯誤，說明相同：`N` 個元素不能共用一個擁有者。

```nabi
fn zeros<const N: usize>() -> [f64; N] { [0.0; N] }
let z: [f64; 4] = zeros();              // N = 4, from the expected type
let w = [1u8; 16];                       // [u8; 16]

let g = [[0; 3]; 2];
g[0][1] = 5;                             // g[1] is still [0, 0, 0]
let rows = [List::<i32>::new(); 3];
rows[0].push(1);                         // rows[1] and rows[2] stay empty
let fs = [try File::open("a"); 2];       // ERROR: a `File` has one owner (E0069)
```

## 2.11.7. 單態化

`const` 泛型的運作方式與[泛型](210-generics.md)的單態化相同。每一種 `const` 引數組合都會產生各自的編譯結果。

```nabi
let a = replicate(0i32);    // N = ?, determined at inference time
```

具有相同 `const` 引數組合的呼叫共用結果。

## 2.11.8. 相等性

`const` 引數值相同的兩個型別是同一個型別。

```nabi
let a: [i32; 4] = ...;
let b: [i32; 2 + 2] = ...;
// a and b have the same type
```

這種相等性之所以可能，是因為 `const` 運算式的求值是確定性的。

## 2.11.9. 可以定義的位置

`const` 泛型可以在下列所有位置以相同語法使用：函式與方法；結構、`enum` 與 `union`；trait、`impl` 區塊與型別別名。

這種一致性與[泛型](210-generics.md)中的一般泛型相同，因此沒有額外的規則需要記。

## 2.11.10. 作為索引使用

`const` 參數可以像一般的 `const` 變數一樣，直接用在值位置。

```nabi
fn first<T, const N: usize>(arr: [T; N]) -> T {
    arr[0]
}

fn middle<T, const N: usize>(arr: [T; N]) -> T {
    arr[N / 2]
}
```

在值位置，`N` 是其宣告型別（這裡是 `usize`）的常數，與 `const` 項目相同（[5.2](../005-modules/502-const.md)）：它可以作為索引、決定重複字面值 `[v; N]` 的大小、出現在常數運算式中（2.11.5），並作為 `const` 引數往下傳遞。它不是位置（place），因此不能被指派。

函式主體中的 `const` 與 `const { .. }` 區塊可以讀取它，此時它們在每個實例中各有一個值（[5.2.10](../005-modules/502-const.md#5210-const-區塊)）。對引數的檢查放在這樣的區塊中，因此錯誤的實例會在其呼叫處成為編譯錯誤，而不是在執行時 panic：

```nabi
fn halves<const N: usize>(arr: [i32; N]) -> usize {
    const { assert(N % 2 == 0, "halves needs an even length") };
    const HALF: usize = N / 2;
    HALF
}
```

陣列長度只能讀取一般型別的大小：在 `T = i32` 時，`[u8; 4 * T::size()]` 是 `[u8; 16]`；但若 `T` 是泛型主體中的型別參數或結構，它就是錯誤，其說明會提示使用 const 參數（`const N: usize`）或 `List<u8>`。
