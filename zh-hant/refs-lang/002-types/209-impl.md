# 2.9. `impl`

`impl` 區塊是把方法附加到型別上，或為型別實作 trait 的容器。

## 2.9.1. 兩種形式

`impl` 區塊的文法有兩種形式：

```
impl_block      ::= "impl" generics? type_expr "{" impl_item* "}"
                  | "unsafe"? "impl" generics? type_expr "for" type_expr where_clause? "{" impl_item* "}"
                  | "impl" generics? "!" type_expr "for" type_expr where_clause? "{" "}"
impl_item       ::= fn_decl
                  | "type" identifier "=" type_expr ";"
                  | const_decl
```

在 `impl <` 中，只有在其後接著泛型參數時（`<>`、`<T>`、`<T:`、`<T,`、`<T =`、`<const N`），`<` 才開啟 `generics`；否則它是目標型別的開頭，例如固有 `impl <A as P>::Out { .. }`。

### 固有 `impl`：附加方法

首先是固有 `impl`：

```nabi
impl Point {
    fn origin() -> Point { Point { x: 0.0f64, y: 0.0f64 } }
    fn distance(self, other: Point) -> f64 { ... }
}
```

這種形式是 `impl Type { ... }`，主體是一組函式定義。以這種方式定義的函式成為該型別的方法。

固定長度陣列與元組也可以有固有 `impl`。標準函式庫利用這一點為陣列提供 `len()` / `is_empty()`。

```nabi
impl<T, const N: usize> [T; N] {
    fn len(self) -> usize { N }
}

impl<A, B> (A, B) {
    fn swap(self) -> (B, A) { (self.1, self.0) }
}
```

### trait `impl`：實作 trait

其次是 trait `impl`：

```nabi
impl Describe for Point {
    fn describe(self) -> str { f"({self.x}, {self.y})" }
}

impl ops::Add for Point {
    type Output = Point;
    fn add(self, rhs: Point) -> Point { ... }
}
```

這種形式是 `impl Trait for Type { ... }`。主體包含 [`trait`](208-trait.md) 所要求之方法的主體，以及關聯型別的具體定義。trait 提供了預設方法的項目可以從主體中省略，此時使用預設主體。trait 宣告了但沒有預設值的關聯型別或常數必須定義；省略它是錯誤（E0072），其說明會寫出要加入的那一行，若 impl 的簽章中提到了該型別，就使用簽章所給的型別（`type Item = i32;`）。

impl 方法必須：與 trait 宣告有相同的參數數量；除接收者以外的每個參數有相同的 `ref`；有相同的 `unsafe`（[4.15.4](../004-functions/415-unsafe.md#4154-in-traits)）；在它寫出型別的每個參數上使用宣告的型別（經過下述替換；未寫型別的參數取該型別，[4.1](../004-functions/401-function-declaration.md)）；並且恰好在宣告有回傳值時回傳一個值，其型別是把 `Self`、trait 的參數及其關聯型別替換為 impl 的對應項後的宣告型別（沒有更窄或共變的回傳）；否則就是 impl 處的錯誤，錯誤訊息會顯示該宣告。impl 方法的 `throws` 必須與 trait 方法的 `throws` 相同；沒有 `throws` 的 impl 方法取用 trait 方法的 `throws`。impl 不能加入 `throws`，也不能放寬錯誤型別（[4.14.11](../004-functions/414-throws.md#41411-throws-on-methods)）。

`Send` / `Sync` 是自動的標記 trait，有兩種手動宣告方式（[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)）。

- 選入：`unsafe impl Send for X {}`。`unsafe impl` 只能用於 `Send` / `Sync` 以及 `unsafe trait`（[4.15.4](../004-functions/415-unsafe.md#4154-in-traits)），而手動實作這些 trait 時必須使用它。
- 選出：`impl !Send for X {}`、`impl<T> !Send for Cell<T> {}`。它阻止自動實作，主體必須為空。`!` 只能與 `Send` / `Sync` 一起使用。

### `impl` 中的常數

`impl` 區塊內的 `const N: T = v;` 是該型別的關聯常數。從外部讀作 `Type::N`，在 `impl` 內讀作 `Self::N`。trait 宣告的關聯常數（[2.8.3a](208-trait.md#283a-associated-constants)）在 trait `impl` 中取得其值。

```nabi
impl Circle {
    const UNIT: f64 = 1.0;
    fn unit() -> Circle { Circle { r: Self::UNIT } }
}
let r = Circle::UNIT;
```

`impl` 的目標或 trait 引數可以是投影（`<T as Tr>::X`），也可以指稱之後才宣告的項目；它在所有項目註冊完畢之後才解析。

`impl` 的目標可以指稱型別別名，無論是否為泛型。別名會被展開，該 `impl` 就是以被取別名之型別撰寫的 `impl`：在 `type Al<T> = S<T>;` 之下，`impl<T> Al<T> { .. }` 是 `impl<T> S<T> { .. }`，`impl<T> Tr for Al<T>` 是 `impl<T> Tr for S<T>`。固定了部分引數的別名（`type Names<V> = Dict<str, V>;`）所給出的 `impl` 只涵蓋那些實例。

## 2.9.2. 方法的 `self` 慣例

主體中的函式要成為實例方法，其第一個參數必須是 `self`（或 `ref self`）。

```nabi
impl Point {
    fn origin() -> Point { ... }                    // static: called as Type::method()
    fn distance(self, other: Point) -> f64 { ... }  // instance: called as instance.method()
    fn replace_with(ref self, other: Point) { ... } // ref self: may reassign the caller's variable
}
```

`self` 參數的確切意義與 `ref self` 的行為定義於函式規格。本文件只指出 `self` 的有無區分了靜態方法與實例方法。

## 2.9.3. 一個型別的多個 `impl` 區塊

同一個型別的固有 `impl` 可以分成多個區塊。

```nabi
impl Point {
    fn origin() -> Point { ... }
}

impl Point {
    fn distance(self, other: Point) -> f64 { ... }
}
```

這只是視覺上與組織上的分組；結果與合併成一個主體相同。

不過，對同一個型別，某個 trait 只能 `impl` 一次；第二個 impl 無論是否為空都是錯誤（E0075），錯誤訊息會顯示第一個。

```nabi
impl Display for Point { ... }
impl Display for Point { ... }   // ERROR: duplicate implementation (E0075)
```

## 2.9.4. 泛型 `impl`

`impl` 也可以宣告為泛型。

```nabi
impl<T> List<T> {
    fn len(self) -> usize { ... }
}

impl<T> ops::Add for Pair<T> where T: ops::Add<T> {
    type Output = Pair<T::Output>;
    fn add(self, rhs: Pair<T>) -> Pair<T::Output> { ... }
}
```

型別參數以 `impl` 之後的 `<...>` 宣告，可以出現在目標型別、trait 引數與關聯型別中。可以用 `where` 子句加入額外的約束，方式與 trait 宣告相同。

泛型 `impl` 涵蓋其參數能產生的目標實例：`impl<V> P<str, V>` 為 `P<str, i32>` 提供方法，但不為 `P<i32, i32>` 提供。

## 2.9.5. 孤兒規則

為另一個套件定義的型別實作某個套件定義的 trait 是禁止的。

```nabi
// trait Display and type i32 are both defined in the standard library (another package)
impl Display for i32 { ... }     // ERROR: both from external packages (E0076)
```

因此至少必須滿足下列一項：

1. trait 定義於目前的套件；
2. 目標型別定義於目前的套件；
3. trait 是泛型的，而在依序排列的目標型別與 trait 型別引數中（`impl Tr<T1, .., Tn> for T0`：`T0`、`T1`、..、`Tn`），目前套件定義的型別出現在 `impl` 的任何單獨型別參數之前（Rust 的規則，RFC 2451）。

規則 3 讓套件能把純量放在自己型別之運算子的左側：

```nabi
// in a package defining `Matrix`
impl Mul<Matrix> for f64 { type Output = Matrix; ... }      // OK: `Matrix` is local (rule 3)
impl DotAdd<Matrix> for f64 { type Output = Matrix; ... }   // OK
impl<T> Mul<Matrix> for T { ... }        // ERROR: the bare `T` comes first (E0076)
impl Add<List<Matrix>> for i32 { ... }   // ERROR: `List<Matrix>` is `core`'s type (E0076)
```

單獨的參數出現在區域型別之前之所以被拒絕，是因為另一個套件可以為自己的型別在該位置寫出相同的 `impl`；只巢狀在外部型別中的區域型別（`List<Matrix>`）不算數。內建型別（基本型別、元組、陣列、函式型別）視為定義於 `core`。

這條規則的單位是[原始碼單元](../000-meta/003-source-units.md)的套件。因此同一個套件內的不同模組可以自由地 `impl`。

這條規則防止兩個套件同時為同一組（trait, 型別）建立相互衝突的 `impl`，確保一致性。

### 一致性

一組（trait, 型別）只有一個 `impl`，而且沒有特化。針對某一型別的快速路徑，以型別測試的方式放在泛型 `impl` 中：`if Self is f32 { .. }`（[2.10.13](210-generics.md#21013-type-tests-on-type-parameters)）。

- 泛型 `impl` 針對每個使用它的型別單態化。
- 具體 `impl` 只有在該具體型別滿足泛型 `impl` 的約束時，才與泛型 `impl` 重疊。
- 若兩個泛型 `impl` 的目標（以及 trait 引數）能統一，它們就重疊；這會在宣告時回報為錯誤。統一之後仍為泛型的參數上的約束不會把它們分開。即使目前還沒有任何型別同時滿足兩個約束，這也是錯誤（與 Rust 的一致性相同）。
- 就重疊而言，`T?` 與 `T | none` 是同一個型別。
- `const` 引數（[2.11](211-const-generics.md)）依值比較：`Buf<3>` 與 `Buf<{1 + 2}>` 是同一個型別。由 `const` 參數乘以整數再加上常數構成的和（`Buf<{N + 1}>`、`Buf<{2 * N}>`），只有在兩者對某些整數值可能相等時才與另一個引數重疊，而無號型別的參數不取負值。因此 `impl<const N: usize> X for Buf<{N + 1}>` 與 `impl X for Buf<0>` 不重疊，`Buf<{2 * N}>` 與 `Buf<{2 * M + 1}>` 也不重疊。其他任何形式（`{N * M}`）都假設為重疊。
- 否定推理只針對已知型別進行。
  1. 經由統一而成為具體型別的參數，會檢查其約束。`impl<U: Num> X for Pair<U, i32>` 與 `impl<U> X for Pair<str, U>` 以 `U = str` 統一，但 `str: Num` 不成立，因此它們不重疊。
  2. 若某個 trait 的任何 impl 都不以某個已知的型別建構子或單獨參數為目標，則該建構子上的該 trait 約束（`List<T>: Iterator`）會把兩者分開。因此 `impl<I: Iterator> IntoIterator for I` 與 `impl<T> IntoIterator for List<T>` 可以並存。trait 引數也是如此：只要沒有任何 `Dist` impl 以 `Iid` 為目標，`impl<D: Dist> Tilde<D> for S` 與 `impl<D: Dist> Tilde<Iid<D>> for S` 就能並存，而呼叫或運算子會採用其引數滿足約束的那一個（`s ∼ Iid { .. }` 採用第二個）。

```nabi
impl<T: Display> Describe for List<T> { ... }
impl<T: Hash> Describe for List<T> { ... }   // ERROR: i32 satisfies both Display and Hash
impl<T: Shape> Area for T { ... }
impl<T: Color> Area for T { ... }            // ERROR: T and T unify (bounds irrelevant)
```

```nabi
impl<T: Display> Describe for T { ... }
impl Describe for i32 { ... }        // ERROR: overlaps the impl above since i32: Display
```

## 2.9.6. 方法的可見性

固有 `impl` 的方法預設為模組私有。在同一個模組內可以自由呼叫；要對外公開則需要明確的 `pub`。

```nabi
impl Account {
    pub fn balance(self) -> f64 { self.amount }      // public
        fn _internal_check(self) { ... }              // module-internal only
}
```

這與 [`struct`](203-struct.md) 和 [`enum`](204-enum.md) 的可見性模型一致。`impl` 區塊本身只是視覺上的分組，不是可見性的單位；可見性的單位一律是模組。

trait `impl` 的方法跟隨 trait 的可見性，不能另外標記。

## 2.9.7. 固有 `impl` 與 trait `impl` 之間的方法衝突

若同一個方法名稱同時存在於固有 `impl` 與 trait `impl` 中，呼叫處必須無歧義地選擇。確切的解析規則定義於[方法](../004-functions/409-method.md#name-conflicts-between-traits)。

## 2.9.8. 不得重複定義同一個方法

為同一個型別定義兩個同名的方法是編譯錯誤，無論是在一個 `impl` 區塊中，還是在多個固有 `impl` 區塊中（[2.9.3](#293-multiple-impl-blocks-for-one-type)），或是在一個 trait `impl` 中（沒有多載）。來自不同 trait 的同名方法是允許的（[2.9.7](#297-method-conflicts-between-inherent-and-trait-impls)）。
