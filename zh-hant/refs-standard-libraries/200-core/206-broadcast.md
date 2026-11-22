# 2.6. Broadcast：逐元素運算子

`core::ops` 中的廣播 trait 是 Julia 風格點運算子的 trait 依據。點運算子（`.+` / `.*` / `.==` / `f.(x)`）在語法上就讓逐元素的意義一目了然，因此不會與使用者定義的 `*`（例如矩陣乘法）混淆。

本章定義 20 個廣播 trait，以及函式廣播的去語法糖規則。

這些 trait 宣告在 `core::ops`，這是它們唯一的路徑（[0.1.6](../000-meta/001-tier-model.md#016-每個項目只有一條路徑)）；沒有 `broadcast` 子模組。`Dot`，即 [2.3.10](203-ops.md#2310-unicode-數學運算子-trait) 中 `⋅` 內積的 trait，與它們並列。`Dot` 不屬於這 20 個廣播 trait：`a ⋅ b` 把兩個運算元歸約為一個值（`fn dot(self, rhs: Rhs) -> Output`），而不是逐元素運算，且標準函式庫沒有為任何型別實作它。

## 2.6.1. 動機：為何要分開點運算子

在 NumPy 和 MATLAB 中，`*` 對陣列是逐元素、對矩陣卻是乘法，這種**無聲的歧義**長久以來一直是錯誤的來源（PEP 465 引入 `@` 也是出於同樣的考量）。

相較之下，Nabi 的方針很明確。

```nabi
m * n       // user-defined Mul: matrix multiplication, dot product, etc.
m .* n      // element-wise (Hadamard product), always broadcast
```

點前綴（`.`）是表明逐元素意圖的視覺訊號，由編譯器強制執行。在任何容器（List、Vec、Matrix 等）中，`.+` 一律分派到 `DotAdd::dot_add`。

## 2.6.2. 20 個 trait：運算子對應

完整的集合是算術（7）＋比較（6）＋邏輯（2）＋位元（3）＋位移（2）＝ **20** 個 trait。

> [!NOTE]
> 本章定義全部 20 個 trait：7 個算術、6 個比較、2 個邏輯、3 個位元與 2 個位移。點比較（`.==`/`.!=`/`.<`/`.>`/`.<=`/`.>=`）在 [2.5.8 廣播比較](205-cmp.md#258-廣播比較-trait)中只給出**簡要簽章**；
> 本章才是**完整的依據**。

### 算術（7）

| Trait       | 運算子 | 優先順序 | 意義                       |
|-------------|----------|------------|-------------------------------|
| `DotAdd`    | `.+`     | 9（左）   | 逐元素加法         |
| `DotSub`    | `.-`     | 9（左）   | 逐元素減法      |
| `DotMul`    | `.*`     | 10（左）  | 逐元素乘積（Hadamard） |
| `DotDiv`    | `./`     | 10（左）  | 逐元素除法         |
| `DotRem`    | `.%`     | 10（左）  | 逐元素餘數（Rem）  |
| `DotModulo` | `.%%`    | 10（左）  | 逐元素向下取整模數   |
| `DotPow`    | `.**`    | 13（右） | 逐元素冪（`**`，2.3.1） |

### 比較（6）

| Trait   | 運算子 | 優先順序 | 意義                             |
|---------|----------|------------|-------------------------------------|
| `DotEq` | `.==`    | 4（無）   | 逐元素 == → `Container<bool>` |
| `DotNe` | `.!=`    | 4（無）   | 逐元素 !=                     |
| `DotLt` | `.<`     | 4（無）   | 逐元素 <                      |
| `DotGt` | `.>`     | 4（無）   | 逐元素 >                      |
| `DotLe` | `.<=`    | 4（無）   | 逐元素 <=                     |
| `DotGe` | `.>=`    | 4（無）   | 逐元素 >=                     |

### 邏輯（2）

| Trait           | 運算子 | 優先順序 | 意義                                  |
|-----------------|----------|------------|------------------------------------------|
| `DotLogicalAnd` | `.&&`    | 3（左）   | 逐元素邏輯 AND（不短路） |
| `DotLogicalOr`  | `.\|\|`    | 2（左）   | 逐元素邏輯 OR（不短路）  |

> [!IMPORTANT]
> **`.&&` / `.||` 不會短路**。與一般的 `&&` / `||` 不同，廣播會降階為融合後的迴圈，因此兩個運算元一定都會求值。語意上它是 bool 容器的逐元素邏輯組合。放在任一運算元中、帶有副作用的函式呼叫一定會執行。

### 位元（3）

| Trait       | 運算子 | 優先順序 | 意義        |
|-------------|----------|------------|----------------|
| `DotBitAnd` | `.&`     | 7（左）   | 逐元素 & |
| `DotBitOr`  | `.\|`     | 5（左）   | 逐元素 \| |
| `DotBitXor` | `.^`     | 6（左）   | 逐元素 ^ |

### 位移（2）

| Trait    | 運算子 | 優先順序 | 意義         |
|----------|----------|------------|-----------------|
| `DotShl` | `.<<`    | 8（左）   | 逐元素 << |
| `DotShr` | `.>>`    | 8（左）   | 逐元素 >> |

優先順序與結合性和一般的 trait 相同（[2.3 ops](203-ops.md)、[2.5 cmp](205-cmp.md)）。

## 2.6.3. Trait 簽章：統一的形式

每個點 trait 都依循相同的統一形式。

```nabi
trait DotAdd<Rhs = Self> {
    type Output;
    #[infix(".+")]
    fn dot_add(self, rhs: Rhs) -> Output;
}

trait DotMul<Rhs = Self> {
    type Output;
    #[infix(".*")]
    fn dot_mul(self, rhs: Rhs) -> Output;
}

trait DotModulo<Rhs = Self> {
    type Output;
    #[infix(".%%")]
    fn dot_modulo(self, rhs: Rhs) -> Output;
}

trait DotEq<Rhs = Self> {
    type Output;
    #[infix(".==")]
    fn dot_eq(self, rhs: Rhs) -> Output;
}

trait DotLogicalAnd<Rhs = Self> {
    type Output;
    #[infix(".&&")]
    fn dot_logical_and(self, rhs: Rhs) -> Output;
}
```

算術點 trait 與 `Add`/`Sub`/`Mul`/`Div`/`Rem`/`Modulo` 完全對稱：一個 `Output` 關聯型別加上一個中綴屬性。六個比較點 trait（`DotEq`、`DotNe`、`DotLt`、`DotGt`、`DotLe`、`DotGe`）形式相同，各有自己的 `Output`，通常是 `Container<bool>`。邏輯點 trait（`DotLogicalAnd` / `DotLogicalOr`）通常兩側都接受 `Container<bool>` 並回傳 `Container<bool>`；由於不短路，兩個容器都會求值到最後。

### 函式廣播：沒有 trait

函式廣播的寫法如下。

```nabi
f.(x)              // function broadcast
f.(a, b)           // multiple arguments
```

函式廣播沒有自己的 trait；它的去語法糖方式如下。

```nabi
f.(x)        →  x.into_iter().map(f).collect()
g.(a, b)     →  a.into_iter().zip(b.into_iter()).map(|(x, y)| g(x, y)).collect()
```

它直接分派到 [2.7 iter](207-iter.md) 的 `map`/`zip`；廣播模組不另外定義 trait。差別在於函式本身是可呼叫物，而不是分派的目標。

## 2.6.4. 語意：逐元素

### 算術點運算：兩個形狀相同的容器

對兩個形狀相同的容器做算術點運算，行為如下。

```nabi
let a: List<i32> = [1, 2, 3];
let b: List<i32> = [10, 20, 30];

let c = a .+ b;          //→ [11, 22, 33]
let d = a .* b;          //→ [10, 40, 90]
let e = b .- a;          //→ [9, 18, 27]
let f = b ./ a;          //→ [10, 10, 10]
```

長度不符時會在執行時觸發 panic，並指出方法名稱（`List::dot_add: lengths 3 and 2 differ; make both sides the same length`）；融合成一個迴圈的運算子運算式（2.6.6）可能含有多個運算子，此時訊息改以 `broadcast:` 開頭。若大小在編譯時已知，則會以編譯錯誤攔下。

```nabi
let x: [i32; 3] = [1, 2, 3];
let y: [i32; 4] = [1, 2, 3, 4];
let z = x .+ y;          // ERROR (compile: fixed sizes differ)

let p: List<i32> = [1, 2, 3];
let q: List<i32> = [1, 2];
let r = p .+ q;          // panic at runtime: length mismatch
```

### 純量廣播

純量與容器組合時，行為如下。

```nabi
let a: List<i32> = [1, 2, 3];
let s = a .+ 10;         //→ [11, 12, 13]  (scalar broadcast)
let m = 2 .* a;          //→ [2, 4, 6]
let t = a ./ 2;          //→ [0, 1, 1]    (i32 truncate)
```

這裡由各自獨立的 impl（`DotAdd<T> for List<T>`，以及純量在左側時的 `DotAdd<List<T>> for T`）處理純量這一側。`Slice<T>` 也有同樣的一對；固定長度陣列 `[T; N]` 沒有純量在左的 impl。函式庫的容器也可以用同樣方式為自己的型別取得純量在左的形式：孤兒規則允許在定義 `Matrix` 的套件中寫 `impl DotMul<Matrix> for f64`，因為區域的 `Matrix` 出現在任何型別參數之前（[2.9.5](../../refs-lang/002-types/209-impl.md#295-孤兒規則)），所以 `2.0 .* m` 與 `m .* 2.0` 都可以使用。

### 比較點運算：回傳 `Container<bool>`

比較點運算子回傳 `Container<bool>`。

```nabi
let a: List<i32> = [1, 5, 3];
let b: List<i32> = [2, 5, 1];

a .== b;                 //→ [false, true, false]
a .< b;                  //→ [true, false, false]
a .>= b;                 //→ [false, true, true]

a .< 4;                  //→ [true, false, true]   (scalar broadcast)
```

### 餘數點運算：`.%` 與 `.%%`

餘數點運算子把一般 `%` / `%%` 的意義逐元素套用。

```nabi
let a: List<i32> = [-7, 5, -3, 8];
let b: List<i32> = [3, 3, 3, 3];

a .% b;                  //→ [-1, 2, 0, 2]   (Rem: sign follows the left operand)
a .%% b;                 //→ [2, 2, 0, 2]    (Modulo: always in [0, 3))
```

### 邏輯點運算：`.&&` 與 `.||`

邏輯點運算子產生兩個 `Container<bool>` 的逐元素邏輯組合。與不加點的形式不同，它不短路，所以兩個容器一定都會求值到最後。

```nabi
let p: List<bool> = [true, false, true, false];
let q: List<bool> = [true, true, false, false];

p .&& q;                 //→ [true, false, false, false]
p .|| q;                 //→ [true, true, true, false]

// combining comparison results: the most common use
let xs: List<i32> = [1, 5, 3, 8];
(xs .>= 2) .&& (xs .< 6);  //→ [false, true, true, false]
```

### 位元／位移點運算

位元與位移點運算子的運作方式相同。

```nabi
let xs: List<u8> = [0b1100, 0b1010, 0b0011];
let ys: List<u8> = [0b1010, 0b1010, 0b1111];

xs .& ys;                //→ [0b1000, 0b1010, 0b0011]
xs .| ys;                //→ [0b1110, 0b1010, 0b1111]
xs .^ ys;                //→ [0b0110, 0b0000, 0b1100]

let ns: List<u32> = [1, 2, 4];
ns .<< 2;                //→ [4, 8, 16]
```

## 2.6.5. 函式廣播：`f.(x)`

函式廣播的寫法是在一般函式呼叫處加上一個點。

```nabi
let xs: List<f64> = [0.0, 0.5, 1.0];

let s = f64::sin.(xs);               //→ [0.0, 0.479..., 0.841...]
let r = f64::sqrt.(xs);              //→ [0.0, 0.707..., 1.0]
let g = (|x: f64| x * 2.0).(xs);     //→ [0.0, 1.0, 2.0]
```

可以同時廣播多個引數。

```nabi
let a: List<f64> = [1.0, 2.0, 3.0];
let b: List<f64> = [10.0, 20.0, 30.0];

let r = f64::pow.(a, b);             //→ [1.0, 1_048_576.0, ...]
let m = f64::max.(a, b);             //→ [10.0, 20.0, 30.0]
```

型別為 `List`、`Slice` 或陣列的引數會一起迭代；其他引數則原樣傳給每一次呼叫（`f64::pow.(xs, 2.0)`）。被迭代的引數必須長度相同，在執行時檢查。引數數量**沒有上限**，因此任何 N 元函式都適用。

## 2.6.6. 迴圈融合

編譯器把連續的點運算融合成單一迴圈。

```nabi
let result = f64::sin.(a) .+ f64::cos.(b) .* 2.0;
```

單純的降階需要 3 次暫存 List 配置與 3 個迴圈。Nabi 保證在 `List`、`Slice` 與陣列上，以及任何 `Broadcastable` 型別上，只用一個迴圈且不產生中間集合（確切條件見 [3.3.4](../../refs-lang/003-expressions/303-broadcast-operators.md#334-自動迴圈融合)）。原地形式（`y .= ..`、`y .+= ..`，[3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-原地廣播)）直接寫入目標，完全不做配置。

```nabi
// conceptually equivalent to this single elementwise call
let result = elementwise(a, b, |x, y| x.sin() + y.cos() * 2.0);
```

### `Broadcastable`、`BroadcastScalar` 與 `Materialize`

融合協定由 `core::ops` 的三個 trait 組成。

```nabi
#[lang("broadcastable_trait")]
pub trait Broadcastable<const N: usize> {
    type Elem;
    /// The elements as an array: a view sharing the storage.
    fn broadcast_view(self) -> Array<Elem, N>;
}

#[lang("broadcast_scalar_trait")]
pub trait BroadcastScalar {
    type Elem;
    /// The plain value the loop applies to every element.
    fn broadcast_scalar(self) -> Elem;
}

#[lang("materialize_trait")]
pub trait Materialize<U, const N: usize> {
    type Output;
    /// The result built around `out`, a fresh row-major array.
    fn materialize(out: Array<U, N>) -> Output;
}
```

`Array`（[2.9.14](209-collections.md#2914-arrayt-const-n-usizen-維陣列)）由緩衝區、位移量、形狀與步幅組成，因此能描述任何有步幅的儲存空間；自訂的型別（套件的 `Tensor`、資料框的欄）回傳它所持有的 `Array` 來實作 `Broadcastable`，並實作 `Materialize` 以便從結果的元素建構出自己。`Array` 兩者都實作：它的檢視就是它自己，而 `materialize` 回傳傳入的 `Array<U, N>`。實作 `Broadcastable` 代表承諾：該型別的點運算子作用在檢視中的元素上時，就如同作用在一般的值上；只包裝一個數的型別（帶單位的量）則實作 `BroadcastScalar`，以純量的身分與它們並列。

```nabi
struct Signal { data: Array<f64, 1>; }
impl Broadcastable<1> for Signal {
    type Elem = f64;
    fn broadcast_view(self) -> Array<f64, 1> { self.data }
}
impl Materialize<f64, 1> for Signal {
    type Output = Signal;
    fn materialize(out: Array<f64, 1>) -> Signal { Signal { data: out } }
}
let t = s .* 2.0 .+ k;      // one loop; a Signal
```

融合後的運算式會取得每個運算元的檢視（`broadcast_view`；純量包裝型別則用 `broadcast_scalar`），把各檢視延展成同一個形狀（2.6.8），以一個迴圈寫入一個結果形狀的新陣列，再交給結果型別的 `materialize`。迴圈直接走訪緩衝區：當結果與所有檢視都是連續的，就是一段連續的位置；否則就沿最後一軸逐列走訪，各檢視依自己的步幅前進。

由哪個型別建構結果：

- 若運算式樹中的容器全都是 `Array`，採用領頭運算元的型別（最高秩中的第一個）：`Array` 的 `materialize` 產生元素型別為迴圈所計算之 `U` 的 `Array<U, N>`（`a .< b` 是 `Array<bool, N>`）。
- 若運算式樹涉及自訂型別，或有 `BroadcastScalar` 運算元，則先透過它的點 trait 逐個運算子定出型別，就如同沒有融合一樣。若該型別是此樹之秩的 `Broadcastable`，且有其元素型別的 `Materialize`，就由它建構結果；否則此樹不融合，改經由 trait 執行。因此 trait 的型別對融合後的迴圈同樣成立：若某個量的型別其 `DotDiv` 產生 `Quantity<Array<f64, R>, { L - L2 }, ..>`，則 `d ./ t .+ v0` 是一個迴圈，結果是速度；而 `d .+ t` 是與不融合時相同的編譯錯誤。
- 若自訂型別沒有實作樹中任何點 trait，則它的運算子僅來自 `Broadcastable`：由領頭運算元的型別建構結果。

領頭運算元沒有 `Materialize` 時是編譯錯誤，錯誤中會指出該 trait。這類型別上的原地形式（`v .= d ./ t`）也以同樣方式定出型別，若值的型別與目標型別不同，則是編譯錯誤（`` `.=` writes a `Q<Array<f64, 1>, 1>` into a `Q<Array<f64, 1>, -1>` ``）。

## 2.6.7. Trait 實作矩陣

點 trait 的 impl 對應到各容器型別的方式如下。

| 容器        | DotAdd   | DotMul   | DotEq       | 備註                                   |
|------------------|----------|----------|-------------|-----------------------------------------|
| `List<T>`        | T:Add    | T:Mul    | T:PartialEq | 逐元素，長度必須相符        |
| `[T; N]`（固定） | T:Add    | T:Mul    | T:PartialEq | 固定 N，編譯時                   |
| `Slice<T>`       | T:Add    | T:Mul    | T:PartialEq | 檢視，逐元素                      |
| `i32`/`f64`/等 | 純量在左 | 純量在左 | 純量在左 | 沒有純量對純量的 impl（用一般的 `+`/`*`/`==`）；`T` 實作 `DotAdd<List<T>>` / `DotAdd<Slice<T>>` 作為純量在左的形式（2.6.4） |
| `T \| none`       | （運算子） | （運算子） | （運算子） | 可選值傳播（`none .+ x = none`） |

`T | none` 的點運算意義是可選值傳播：任一側為 `none` 時，結果就是 `none`。這在運算子層級處理，沒有函式庫 impl：整棵點運算式樹的運算元依原始碼順序各求值一次；接著檢查每個 `T | none` 運算元，遇到第一個 `none` 時，整棵樹的值就是 `none`，不再執行。否則就以 `T` 值執行此樹，依 2.6.6 所述以融合方式或透過點 trait 執行，因此不會以 `none` 呼叫任何點 trait。值的型別是此樹的型別 `| none`。

core 的 n 維 `Array<T, N>`（[2.9.14](209-collections.md#2914-arrayt-const-n-usizen-維陣列)）實作所有點 trait：同秩的陣列與陣列、陣列與純量、純量與陣列，依 2.6.8 的規則。函式庫的數值容器（`Tensor<T>`、資料框的欄）直接實作相同的 trait，包含純量在左的形式：孤兒規則允許在定義 `Tensor` 的套件中寫 `impl DotMul<Tensor> for f64`（[refs-lang 2.9.5](../../refs-lang/002-types/209-impl.md#295-孤兒規則)）。對這類型別的點運算式會逐個運算子呼叫 trait，除非該型別是 `Broadcastable`（2.6.6）；`Array` 是，所以它的點運算式會融合，而它的 trait impl 則用於約束與以名稱呼叫的情況。

## 2.6.8. NumPy 風格的廣播規則

只有一條規則，定義在 core，讓所有陣列型別一致：`core::array::broadcast_shape(a, b)` 接受兩個同秩的形狀，逐軸比較，長度相等就保留，長度為 1 就延展為另一側的長度。其他任何組合都會觸發 panic，並指出兩個形狀與該軸（`broadcast: shapes [2, 3] and [2, 2] differ on axis 1; each axis must match or have length 1`）。純量套用到每個元素。在點運算式中，不同秩的運算元以最後幾軸對齊：較短的一方視為前面補上長度為 1 的軸（`[3]` 向量與 `[2, 3]` 矩陣得到 `[2, 3]`），結果取最高的秩，這會反映在型別上。點 trait 的 impl 本身只接受單一秩，所以泛型程式碼與以名稱呼叫時要明確對齊秩（`v[none, :]` 把 `[3]` 向量變成 `[1, 3]` 的列）。`Array::broadcast_to(shape)` 直接產生延展後的檢視（延展的軸步幅為 0，不複製）。

一維容器（`List<T>`、`[T; N]`、`Slice<T>`）只有一軸，也沒有長度 1 的延展：它們的長度必須相等（2.6.4）。

## 2.6.9. 與其他語言的比較

與其他主要語言相比：

| 面向             | NumPy                | Julia                      | MATLAB           | Nabi                           |
|--------------------|----------------------|----------------------------|------------------|-------------------------------|
| 逐元素       | `*`（無聲）         | `.*`（明確）            | `.*`（明確）  | `.*`（明確）               |
| 矩陣乘法         | `@`（PEP 465）        | `*`                        | `*`              | `*`（使用者定義的 Mul）        |
| 逐元素加法   | `+`（無聲）         | `.+`                       | `.+`             | `.+`                          |
| 函式廣播 | `np.vectorize(f)(x)` | `f.(x)`                    | `arrayfun(f, x)` | `f.(x)`                       |
| 迴圈融合        | **無**             | `@.` 巨集（自動）     | **無**         | 點運算鏈自動融合        |
| 比較         | `==` 回傳陣列   | `.==`                      | `==`（易混淆） | `.==`                         |
| 獨立的點 trait | **無**            | **無**（只有 `*` 多載） | **無**       | **DotAdd/DotMul/...**（20 個）    |

Nabi 與其他語言最大的差異有兩點。第一，每個點運算子都有自己的 trait。Julia 把點語法疊加在多載的 `+` 之上，而在 Nabi 中 `Add` 與 `DotAdd` 是不同的 trait，`Matrix * Matrix` 與 `Matrix .* Matrix` 走的是完全不同的分派。第二，編譯器強制使用點運算。透過一般的 `*` 進行廣播是**不可能的**，這保證了一種不依賴使用者程式碼的確定性區分。
