# 3.5. 成員存取與方法呼叫

`.` 語彙單元（[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)）以兩種形式使用，用來存取附加在值上的項目。

```
member_expr     ::= primary "." identifier
                  | primary "." int_lit               // tuple position
method_call     ::= primary "." identifier ("::" "<" type_expr ("," type_expr)* ">")?
                    "(" arg_list? ")"
arg_list        ::= expr ("," expr)* ","?
```

## 3.5.1. 成員存取：`obj.field`

```nabi
let p = Point { x: 1.0f64, y: 2.0f64 };
let xv = p.x;               // 1.0
let yv = p.y;               // 2.0
```

這裡的 `obj` 是任何運算式，`field` 遵循[識別字](../001-lexical/104-identifiers.md)的形式。結果是該欄位的值。

### 元組 `struct` 與元組的位置存取

```nabi
let pair = Pair(1i32, 2i32);
let first = pair.0;
let second = pair.1;
```

點之後的十進位數字被解讀為位置索引；這與[數值字面值](../001-lexical/106-numeric-literals.md)中 `1.foo` 規則的切分方式相同。例如 `pair.0` 被切分為三個語彙單元：識別字 `pair`、`.` 與整數 `0`。剖析器把它們解讀為位置存取。

### 靜態解析與動態解析

若左側的靜態型別已知，欄位位置在編譯期決定，並編譯為直接的記憶體存取。若左側是 [`any` 槽位](../002-types/202-any-slot.md)，就會進行動態分派：在執行時檢查標籤並查找適當的欄位位置。

## 3.5.2. 方法呼叫：`obj.method(args)`

```nabi
let s = "hello";
let n = s.len();
let p = Point { x: 1.0, y: 2.0 };
let d = p.distance(other);
```

形式是 `expr . method_name ( arguments )`。方法解析查看左側的靜態型別，找出定義在該型別上的方法，或該型別所滿足之 trait 的方法。

### 方法解析順序

同名的方法可能定義在多個地方。編譯器依下列順序尋找候選：

1. 首先是定義在固有 `impl`（[`impl`](../002-types/209-impl.md)）中的方法。
2. 然後是 trait `impl` 的方法，只限於左側型別所滿足、且在目前模組作用域內的 trait：`core` 的 trait（其中包括 prelude 的 trait：它們是內建型別本身的方法）、以 `use` 匯入的 trait（在模組或外層區塊中）、定義在同一個模組中的 trait，以及型別參數接收者的約束。`std` 或某個套件的 trait 需要 `use`（對 `(0..6).outcomes()` 寫 `use std::random::SampleRange;`），與 Rust 相同。透過型別寫出的關聯函式（`Point::origin()`）看得到這些 trait 以及其他所有標準函式庫的 trait。meta 樣板所寫出的方法名稱，以該樣板所屬模組作用域內的 trait 查找（[6.9.9](../006-meta-system/609-template-tokens.md#展開中的名稱)）。

3. 當兩者都沒有該方法，而接收者的型別有一個不帶參數的方法 `iter()` 時，若 `iter()` 回傳的型別有 `m`，`x.m(args)` 就是 `x.iter().m(args)`。即使 `Ord::min` / `max` 存在（它們接受一個引數），不帶引數的 `x.min()` / `x.max()` 也以相同方式轉發。因此集合型別能回應迭代器方法：`xs.map(f).sum()`（[stdlib 2.7.14](../../refs-standard-libraries/200-core/207-iter.md#2714-集合型別回應迭代器方法)）。

型別為型別參數的接收者（`T: Root` 之下的 `x: T`）只具有其約束所給予的東西，因此在泛型主體中，`x.sqrt()` 對每個實例都是 `Root::sqrt`，即使是對具有固有 `sqrt` 之型別的實例（`T = Complex<f64>`）也一樣。

若有多個 trait 候選，而其中恰好一個來自模組中定義或 `use` 的 trait（固有方法一律勝出），就選那一個。否則同時存在多個候選是有歧義的編譯錯誤。此時使用者必須以[型別路徑](../002-types/213-type-paths.md)的完整限定形式明確地解決它。

只有作用域外的 trait 才提供的方法是錯誤（E0135），錯誤訊息會指出應加入的 `use`：``method `hi` comes from trait `a::Greet`, which is not in scope``，說明 ``add `use pkg::a::Greet;` ``。其他套件的 trait 以其公開所在的路徑指稱（`linalg::AsVector`，而不是宣告它的模組）。完整限定形式（`a::Greet::hi(x)`）不需要 `use`。

當一個泛型 trait 的多個 impl 都提供該方法時（`List<i64>` 的 `Conv<f64>` 與 `Conv<i64>`），由 trait 的引數選出一個：型別參數的約束（`X: Conv<f64>` 之下的 `x.conv()`，其中 `x: X`）、路徑的引數（`Conv::<f64>::conv(x)`）或限定路徑的引數（`<List<i64> as Conv<f64>>::conv(x)`）。

### 方法接收者的自動 `ref`

方法呼叫處的接收者（`obj`）會依方法的 `self` 種類（`self` / `ref self`）自動處理。呼叫處不需要寫 `ref`。

```nabi
p.distance(other);          // self
p.replace_with(other);      // ref self: caller does not write 'ref'
```

這是方法接收者專用的規則，與一般函式呼叫的明確 `ref` 規則不同。詳細內容定義於函式規格。

### 方法呼叫的去糖

```
obj.method(a, b)
```

上面的運算式在語意上等價於：

```
<typeof(obj) as TraitContaining_method>::method(obj, a, b)
```

（若定義在固有 `impl` 中，則是 `Type::method(obj, a, b)`。）

這種去糖遵循與[運算子分派](301-operator-dispatch.md)之 trait 方法對應相同的機制。`.` 語彙單元本身並不帶有 `#[infix(".")]` 之類的對應，但所產生的呼叫形狀相同。

### Turbofish：明確的泛型方法

方法的泛型引數可以用[型別路徑](../002-types/213-type-paths.md)的 turbofish 形式明確給出。

```nabi
let v = expr.parse::<i32>();
let m = arr.collect::<List<f64>>();
```

形式是 `.method::<...>(args)`：方法名稱接在 `.` 之後，接著是 turbofish，然後是引數的括號。

## 3.5.3. 與靜態項目存取的差異

`Type::item`（[型別路徑](../002-types/213-type-paths.md)）存取附加在型別上的項目。

```
Type::item       // static method / associated const / associated type (no instance)
obj.item         // member / method attached to an instance (instance required)
```

這兩種形式是完全不同種類的存取。語彙單元不同（`::` 與 `.`），意義也不同。

### 方法路徑值

不帶括號寫出的方法路徑（`T::method`、`Trait::method`、`<T as Tr>::method`、`i32::add`、`S::new`）是函式值。

| 方法種類         | 函式值的型別                                 |
|------------------|----------------------------------------------|
| 靜態方法         | `fn(args) -> R`                              |
| 實例方法         | `fn(Self, args) -> R`（`self` 是第一個引數） |

```nabi
let mk = Point::new;               // fn(f64, f64) -> Point
let add = i32::add;                // fn(i32, i32) -> i32
xs.map(Point::norm);               // fn(Point) -> f64
```

- 帶有 `ref self` 或 `ref` 參數的方法不能作為值（[4.7.6](../004-functions/407-ref-keyword.md#476-ref-與型別)）。
- 會擲回錯誤的方法給出會擲回錯誤的函式型別（`fn(..) throws (E) -> R`）。
- 對 `Tr::m`，或 `Self` 尚未決定之泛型型別的方法，函式型別由預期型別決定。沒有預期型別時，由之後的呼叫決定（`let f = Box::get; f(b)`，[2.10.2](../002-types/210-generics.md#型別引數推論)）；若仍未決定，就是「type annotations needed」錯誤。
- 泛型方法的型別引數由 turbofish 或預期的函式型別決定。
- `#[intrinsic]` 方法也可以作為值。

附加在實例上的形式，例如 `obj.method`，不是函式值（3.5.4）。

## 3.5.4. 不帶呼叫的 `.method`

不帶括號寫出 `.method` 時，Nabi 不會把它解讀為函式值。不賦予部分套用或方法參照之類的另外意義。

```
let f = obj.method;        // ERROR (E0047): not a method call; the expression is incomplete
let f = || obj.method();   // wrap in an explicit closure
```

這個決定避免了隱含部分套用的歧義，並讓使用者直接說明其意圖。

## 3.5.5. 安全的成員存取：`?.`

對可能不存在之值的點存取，以[可選鏈結與空值合併](308-optional-chaining-and-coalescing.md)中定義的可選鏈結撰寫。
