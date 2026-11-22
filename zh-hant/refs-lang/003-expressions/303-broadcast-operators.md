# 3.3. 廣播運算子

[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)中定義的帶點前綴運算子（`.+`、`.*`、`.==` 等）是用於逐元素運算（廣播）的獨立語彙單元。它們透過與一般運算子相同的機制運作（去糖為 trait 方法呼叫），但對應到另一組 trait。

## 3.3.1. 去糖模式

帶點前綴的運算子去糖如下。

```
a .+ b      →   <typeof(a) as DotAdd<typeof(b)>>::dot_add(a, b)
a .* b      →   <typeof(a) as DotMul<typeof(b)>>::dot_mul(a, b)
a .== b     →   <typeof(a) as DotEq<typeof(b)>>::dot_eq(a, b)
```

這與[運算子分派](301-operator-dispatch.md)中一般對應的形式相同；只有目標 trait 不同。帶點前綴的運算子對應到名稱以 **Dot** 開頭的另一個 trait 家族，如下所示。

| 帶點前綴的語彙單元 | 不帶點的形式      | 帶點前綴的 trait   |
|--------------------|-------------------|--------------------|
| `.+`         | `+` ↔ `Add`       | `DotAdd`        |
| `.-`         | `-` ↔ `Sub`       | `DotSub`        |
| `.*`         | `*` ↔ `Mul`       | `DotMul`        |
| `.**`        | `**` ↔ `Pow`      | `DotPow`        |
| `./`         | `/` ↔ `Div`       | `DotDiv`        |
| `.%`         | `%` ↔ `Rem`       | `DotRem`        |
| `.%%`        | `%%` ↔ `Modulo`   | `DotModulo`     |
| `.==`        | `==` ↔ `PartialEq::eq` | `DotEq`    |
| `.!=`        | `!=` ↔ `PartialEq::ne` | `DotNe`    |
| `.<`         | `<` ↔ `PartialOrd::lt` | `DotLt`    |
| `.>`         | `>` ↔ `PartialOrd::gt` | `DotGt`    |
| `.<=`        | `<=` ↔ `PartialOrd::le` | `DotLe`   |
| `.>=`        | `>=` ↔ `PartialOrd::ge` | `DotGe`   |
| `.&&`        | `&&`（短路）      | `DotLogicalAnd` |
| `.\|\|`        | `\|\|`（短路）      | `DotLogicalOr`  |
| `.&`         | `&`               | `DotBitAnd`     |
| `.\|`         | `\|`               | `DotBitOr`      |
| `.^`         | `^`               | `DotBitXor`     |
| `.<<`        | `<<`              | `DotShl`        |
| `.>>`        | `>>`              | `DotShr`        |

這些 trait 的確切簽章以及哪些型別實作它們，由標準函式庫定義。本文件只負責語言把帶點前綴的語彙單元去糖到另一個 trait 家族這一事實。

> [!IMPORTANT]
> **帶點前綴的邏輯運算子不短路**：與不帶點的形式不同，`.&&` 與 `.||` **不會**短路。由於廣播被降階為融合的迴圈，兩個運算元一律都會被求值；在語意上，它是對 bool 容器的逐元素邏輯組合。放在任一運算元中、帶有副作用的函式呼叫一律會執行。

## 3.3.2. 優先順序與結合性

帶點前綴的運算子與其不帶點的對應形式具有相同的優先順序與結合性。

```
a .+ b .* c     →   a .+ (b .* c)              // .* binds tighter than .+ (same as * > +)
a .+ b .+ c     →   (a .+ b) .+ c              // left-associative
```

## 3.3.3. 函式與方法廣播

在函式或方法之後附加 `.`，會把它套用到每個元素上。

```nabi
sin.(x)            // apply sin to each element of x
f64::sqrt.(arr)    // method form also allowed
g.(a, b)           // multiple arguments
```

這種形式回傳一個新的集合，把一般函式或方法呼叫的結果對應到每個元素上：一個 `List`，或在每個容器引數都是固定長度陣列時，一個固定長度陣列。它去糖為對標準函式庫 `Iterator::map`（或等價 trait）的呼叫，並收集其結果。

```
f.(x)        →   x.iter().map(f)                                  // collected
g.(a, b)     →   a.iter().zip(b.iter()).map(|(x, y)| g(x, y))     // collected
```

函式與方法廣播沒有自己的 trait；它是迭代器轉換之上的語法糖。這正是它與二元帶點前綴運算子不同之處。

## 3.3.4. 自動迴圈融合

由帶點前綴運算子與函式廣播構成的運算式樹會被融合為一個迴圈。

```
sin.(a) .+ cos.(b) .* 2.0f64
```

當容器運算元全都是 core 的 `List`、`Slice` 或固定長度陣列，或全都是 `Broadcastable`（[stdlib 2.6.6](../../refs-standard-libraries/200-core/206-broadcast.md#266-迴圈融合)），而點運算子的其他每個運算元都是基本型別、元素型別，或（與自訂的 `Broadcastable` 型別並列時）`BroadcastScalar` 時，保證會融合：結果在一次走訪中建立，沒有中間集合。每個運算元依原始碼順序在迴圈之前各求值一次。

- 對 core 集合而言，長度在第一個元素之前檢查（固定長度陣列在編譯期檢查）。結果是 `List`，或在每個容器運算元都是陣列時是陣列。
- 對 `Broadcastable` 運算元（`Array`，或自訂型別）而言，每個運算元都被視為陣列，並在第一個元素之前依廣播規則（[stdlib 2.6.8](../../refs-standard-libraries/200-core/206-broadcast.md#268-numpy-風格的廣播規則)）把這些視圖伸展為同一個形狀。結果透過 `Materialize` 建立：對 `Array` 而言是主導運算元（秩最高者中的第一個）的，對自訂型別而言是點 trait 所給出之型別的，因此融合後的迴圈具有運算子逐一套用時會得到的型別（stdlib 2.6.6）。
- 混用兩類運算元的樹是錯誤，其說明會把串列轉換過來（`Array::from_list(xs, [xs.len()])`）。

其他任何樹，即含有既非容器、也非基本型別或元素型別之運算元的樹，不會被融合：每個運算子呼叫其點 trait（3.3.1）並產生自己的結果。

融合後的迴圈是一般的迴圈，和其他迴圈一樣編譯（[0.6.4](../000-meta/006-platform-targets.md#064-simd)）。

## 3.3.5. 與一般運算子的語意差異

> [!IMPORTANT]
> 帶點前綴的運算子之所以是獨立的 trait，是因為同樣的資料可以帶有數學上不同的意義。

```
m * n           // matrix product (Mul impl, user-defined)
m .* n          // element-wise product (Hadamard product, DotMul)
```

為了避免 NumPy 的 `*` 隱含為逐元素運算所造成的混淆，Nabi 一律在視覺上把兩種意義分開。

這種區分直接套用了[語言與標準函式庫的邊界](../000-meta/004-language-vs-standard-libraries.md)中「語法屬於語言，意義屬於函式庫」的原則：兩種不同的意義對應到兩種不同的語法。

## 3.3.6. 使用者定義的 trait 實作

要讓型別具有帶點前綴的運算，請以與一般運算子相同的方式實作 **Dot** trait 家族。

```nabi
impl ops::DotAdd for MyVector {
    type Output = MyVector;
    fn dot_add(self, rhs: MyVector) -> MyVector { ... }
}
```

帶點前綴的運算透過同樣的模式支援，不需要額外的 trait 或巨集。改為實作 `Broadcastable` 的型別則得到融合的運算式與原地廣播（3.3.4、3.3.7）；此時它的點 trait impl 用於 trait 約束與以名稱呼叫。

## 3.3.7. 原地廣播

`target .= value` 把 `value`（點運算式、容器或純量）的元素逐一寫入 `target`；對 `+ - * / % %% ** & | ^ << >>` 中的 `op`，`target .op= value` 是 `target .= target .op value`，且 `target` 只求值一次。它們和 `=` 一樣是陳述式。

```nabi
y .= a .* x .+ b;          // one loop, no array allocated
y .+= 1.0;                 // y .= y .+ 1.0
m[:, 0] .= 0.0;            // a view is a target: the column of m
xs .*= ys;                 // a List, lengths equal
```

- `.=` 先依原始碼順序對運算元求值，再對目標求值；`.op=` 先對目標求值（它是第一個運算元），再對其他運算元求值（[3.4.7](304-assignment.md#347-求值順序)）。
- `List` 或固定長度陣列目標接受其長度的運算元（在第一個元素之前檢查，固定長度陣列在編譯期檢查）；每個元素透過 `IndexSet` 存入。`Slice` 是唯讀視圖，不能作為目標。
- `Broadcastable` 目標保持其形狀：運算元伸展為它的形狀（stdlib 2.6.8），秩較高的運算元是編譯錯誤；結果形狀不同時會 panic（`broadcast: the result has shape [2, 3], which does not fit the destination of shape [1, 3]; the destination is not stretched`）。元素透過目標的視圖寫入，因此陣列的切片（`a[1:]`）會寫入該陣列。
- 在其他位置與目標共用緩衝區的運算元（`v .= v[::-1]`），從迴圈之前製作的複本讀取；同一個視圖（`v .+= v`）或與之分開的緩衝區則原地讀取。結果就像每個元素都在任何元素被存入之前就已計算完畢。
- 比較以及 `.&&` `.||` `.??` 透過 `.=` 使用：`ok .= ok .&& xs .> 0`。
- 其他任何目標都是錯誤，其說明會把運算展開寫出（`x = x .+ ..;`）。

## 3.3.8. 廣播合併 `.??`

`a .?? b` 是逐元素的 `??`：`a` 中每個 `none` 元素都變成 `b` 的對應元素，若 `b` 是純量則變成 `b` 本身。它與樹的其餘部分融合；其結合方式與 `??` 相同（等級 1，右結合）。其結果的元素是各元素的 `??`（`List<f64 | none> .?? 0.0` 是 `List<f64>`）。沒有容器運算元時，它是錯誤，其說明是 `x ?? y`。

`none` 與 NaN 保持區分：`.??` 只取代 `none`，NaN 元素是會被傳遞下去的值（`[1.0, none, NaN] .?? 0.0` 是 `[1.0, 0.0, NaN]`）。`f64 | none` 不會以 NaN 作為其 `none` 來儲存：NaN 是浮點數算術的一般結果，把兩者混為一談會遺失缺失值或憑空造出缺失值。要把 NaN 當作缺失值，請遮罩它：`a[f64::is_nan.(a)] = 0.0`。
