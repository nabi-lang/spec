# 2.6. 和型別（`A | B`）

和型別是值為多個既有型別之一的帶標籤聯集。它與 `enum` 相似，但其變體沒有另外命名；既有的型別直接作為變體使用。

## 2.6.1. 形式

和型別的範例：

```nabi
i32 | f32
T | none
A | B | C
i32 | str | bool
```

各項以[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)的 `|` 語彙單元分隔。一項可以是任何型別運算式：簡單名稱、路徑、泛型，或另一個和型別。

## 2.6.2. 意義

和型別的值屬於其中一項，而它屬於哪一項由標籤追蹤。

```nabi
let x: i32 | f32 = 42i32;       // tag is at the i32 position
let y: i32 | f32 = 3.14f32;     // tag is at the f32 position
```

## 2.6.3. 記憶體表示

預設的表示方式把標籤與酬載行內放置。

```
+------+----------------------+
| tag  | payload (size of     |
|      |  largest variant)    |
+------+----------------------+
```

標籤識別哪一項是作用中的。酬載的大小取所有項中最大者，對齊則取所有項中最大的對齊。

這種表示方式適用 [niche 最佳化](207-niche-optimization.md)，因此標籤槽位可能消失。

### 值遞迴與大小限制

- 以值的方式包含自身的遞迴型別（即使經由結構、列舉、元組、和型別、陣列或泛型實例）在宣告處就是錯誤。遞迴請使用 GC 參照型別（`List<T>` 等）。
- Nabi 的實作限制（由實作定義，[0.4.4](../000-meta/004-language-vs-standard-libraries.md#044-實作限制)）：超過 65536 個攤平後純量槽位或 4 GiB 的值型別是錯誤。

確切的放置方式由實作決定，但保證一件事：以零填滿的 `T | none` 儲存空間（`GcBuf<T | none>` 與集合型別緩衝區的元素、新配置之 GC 物件的欄位）讀出來是 `none`。

### 確切的位元寬度

標籤的位元寬度由編譯器決定，依項數可以是 1、2、4、8 位元等。使用者程式碼無法直接存取標籤位元。

## 2.6.4. 集合語意

> [!IMPORTANT]
> 和型別具有集合運算的語意。

### 正規化

和型別一律以正規形式處理。

- 巢狀的和型別會被攤平。`(A | B) | C`、`A | (B | C)` 與 `A | B | C` 是同一個型別。
- 重複會被移除。`A | A | B` 是 `A | B`。
- 項的順序無關緊要。`A | B` 與 `B | A` 是同一個型別。
- 若只剩一項，就是該型別本身。`A | A` 是 `A`。

展開型別別名的結果會再次正規化（在 `type N = i32 | f64;` 之下，`N | i32` 是 `f64 | i32`）。泛型參數的代入遵循下文的「泛型和型別的分層」。

因此相同形狀的和型別是同一個型別，使用者不必在意寫的是 `i32 | f32` 還是 `f32 | i32`。

### 泛型和型別的分層

帶有泛型項的和型別 `T | X1 | ... | Xn`（例如 `T | none`、`T | ParseErr`），在語意上**自成一層**。`X1 ... Xn` 是寫在該和型別中的非泛型項。泛型項有下列四種：

- 型別參數（`T`）
- `Self`
- 關聯型別（`Self::X`、`T::X`、`<T as Tr>::X`）
- 泛型型別別名的參數

分層規則適用於泛型項被實際型別取代之時。若代入的型別與同時寫出的項有重疊，就形成巢狀的層；若沒有重疊，就攤平。下文以 `T` 代表一個泛型項。

**保留分層的情況。** 若代入 `T` 的型別與 `X1 ... Xn` 有任何共同項，代入後的 `T` 保持為單一項，成為**巢狀的層**。若沒有重疊，就攤平。攤平不會遺失區分值的資訊，因此結果看起來就是攤平的和型別，也被當作攤平的和型別處理。

| 泛型和型別           | `T` 的代入           | 結果                                                |
|----------------------|----------------------|-----------------------------------------------------|
| `T \| none`          | `i32 \| str`         | `i32 \| str \| none`（沒有重疊，攤平）               |
| `T \| none`          | `i32 \| none`        | `(i32 \| none)?`（兩層）                             |
| `T \| ParseErr`      | `i32 \| ParseErr`    | `(i32 \| ParseErr) \| ParseErr`                      |
| `T \| str \| none`   | `i32 \| str`         | `(i32 \| str) \| str \| none`（`str` 重疊）          |

- 重疊計入任何項，不只是 `none`。
- 只由兩個參數組成的和型別 `T | U`，兩者都被代入，因此即使重疊也會攤平，而不會形成巢狀。若必須區分這兩個值，請使用 `enum`。
- 每一層各自正規化，因此 `A | B == B | A` 在每一層都成立。
- 在原始碼中以具體型別寫出的和型別一律攤平。寫成 `(i32 | none) | none` 得到的是 `i32 | none`。
- 非泛型的型別別名是寫出的型別，因此一律攤平。在 `type Opt = i32 | none; type Flat = Opt | none;` 之下，`Flat` 是 `i32 | none`。
- 泛型別名的參數是泛型項，遵循上述規則（見下文的「撰寫巢狀型別」）。

**撰寫巢狀型別。** 可選型別以後綴 `?` 撰寫（[2.1.5a](201-type-expressions.md#215a-可選型別t)）。`T?` 與 `Maybe<T>` 相同，因此其運算元是一個泛型項，適用相同的分層規則。其他巢狀形式（`(E | i32) | E` 等）以泛型別名或 `Self::X` 撰寫。

```nabi
type Maybe<T> = T | none;
let r: (i32 | none)? = xs.get(0);       // two layers
let q: Maybe<i32 | none> = xs.get(0);   // same type
let p: i32? = 1;                         // i32 | none (no overlap)
let f: (i32 | str)? = none;              // i32 | str | none
```

**存入值。** 把值存入巢狀的和型別時，屬於 `T` 某一項型別的值放進內層。屬於外層某一項型別的值（`none` 字面值、`ParseErr` 值等）是外層的值。要把外層項型別的值放進內層，請先把它放進型別為 `T` 的槽位，再存入。

**轉換。** 巢狀型別不會被隱含轉換為攤平的型別；那是型別錯誤。

```nabi
let v: i32 | none = xs.get(0);    // ERROR: xs: List<i32 | none>
// mismatched types: `let v` expects `i32 | none`, found `(i32 | none)?`
```

**運算。** 模式、`is`、`?.`、`??` 與 `?` 作用於**最外層**。外層的另一側繫結到內層的 `T`。在涵蓋了外層所有項的分支之後，繫結模式繫結的是內層。

**診斷記法。** 診斷以括號表示巢狀的層。若外層只有 `none`，寫成 `(i32 | none)?`；否則寫成 `(E | i32) | E`、`(i32 | none) | str | none`。`(i32 | none)?` 也是意義相同的有效原始碼。

```nabi
struct ParseErr { msg: str; }

fn check<T>(x: T) -> T | ParseErr {
    if bad(x) { return ParseErr { msg: "bad" }; }   // ParseErr of the outer layer
    x                                                // a value of T: inner layer
}

let a = check(1i32);                  // T = i32, no overlap: ParseErr | i32
let v: i32 | ParseErr = ParseErr { msg: "earlier" };
let r = check(v);                     // T = ParseErr | i32, overlap: (ParseErr | i32) | ParseErr
match r {
    e: ParseErr => print(f"check failed: {e.msg}"),        // outer layer: error made by check
    inner => match inner {                                // inner: ParseErr | i32
        e: ParseErr => print(f"original value was an error: {e.msg}"), // inner layer: error that was in v
        n: i32 => print(n),
    },
}
```

**關聯型別。** 在 trait 及其所有 impl 內，單獨的名稱 `X` 與 `Self::X` 都表示關聯型別（[2.8.3](208-trait.md#283-關聯型別)）。關聯型別是泛型項，因此適用相同的規則。當 `Item` 含有 `none` 時，`fn next(ref self) -> Self::Item?` 會形成巢狀，因此產生 `i32 | none` 元素的使用者迭代器，也會把存放的 `none` 當作元素迭代。

`none` 的情況（巢狀可選型別）見 2.6.5 的範例。

### 標準順序

正規形式的各項有確定的順序。

- `none` 一律在最後。
- 其餘各項依結構鍵排序：基本型別依其名稱，結構或列舉依其完整路徑，泛型實例依 `path<arg keys>`。巢狀的和型別會被攤平（見上文的正規化），因此和型別絕不會是和型別的一項；泛型引數內的和型別以其依標準順序排列的各項為鍵（`List<A | B>` 的鍵是 `List<key(A)|key(B)>`）。由「泛型和型別的分層」保留下來的層是單一項，其鍵是 `(` + 依標準順序以 `|` 連接的各項之鍵 + `)`。這個順序與寫出的順序無關。
- 這個順序（以及標籤）絕不取決於建置內部的型別編號，因此預先建置的函式庫與其使用者的看法一致（[9.4.8](../009-tools/904-nabihir-bytecode.md#948-分開編譯的規則)）。

標準順序固定的是表示方式（標籤）；它不為值排序。只有在除 `none` 之外最多只有一項的和型別中，值才有順序（[stdlib 2.5.7](../../refs-standard-libraries/200-core/205-cmp.md#257-implementation-matrix)）：

- 當 `T` 有 `PartialOrd` / `Ord` 時，`T | none`（以及內層為單一項的巢狀 `(T | none)?`）也有。`none` 小於任何 `T` 值，與 Rust 和 Haskell 相同；兩個 `T` 值依 `T` 的順序比較。`min` / `max` / `clamp` 與排序都遵循這個順序。
- 除 `none` 之外有兩項以上的和型別（`i32 | str`、`i32 | str | none`）沒有 `PartialOrd` / `Ord`。對這種值使用 `<`、`>`、`<=`、`>=`、`cmp`、`min` / `max` / `clamp`，以及對這種值的集合使用 `sort` 或 `max`，都是編譯錯誤。請用 `sort_by_key` / `sort_by` / `min_by_key` 依某個鍵為這些值排序，或對各項進行 `match`。

```nabi
let xs: List<i32 | none> = [3, none, 1];
xs.sort();                       // [none, 1, 3]
let ys: List<i32 | str> = [1, "a"];
ys.sort();                       // ERROR: `i32 | str` has no order
ys.sort_by_key(|v| match v { n: i32 => (0, n, ""), s: str => (1, 0, s) });
```

### 子集

- 和型別的值可以原樣放進包含其所有項的較大和型別。`i32 | none` 值可以放在 `i32 | str | none` 的位置。單一項（`i32`）的值也是如此。
- 型別模式 `v: A | B` 可以比對較大和型別各項的子集。在該分支中，`v` 是 `A | B`。

```nabi
let v: i32 | str | f64 | none = ...;
match v {
    n: i32 | f64 => use_num(n),     // n: f64 | i32
    s: str => use_str(s),
    none => {},
}
```

### 泛型推論

泛型推論把和型別當作集合處理。以 `T | none` 比對 `i32 | str | none` 時，會收集剩下的項：`T = i32 | str`。

```nabi
fn unwrap_or<T>(x: T | none, d: T) -> T { ... }
let v: i32 | str | none = ...;
unwrap_or(v, 0i32);       // T = i32 | str
```

## 2.6.5. 與 `none` 結合

[布林值與哨兵值字面值](../001-lexical/109-other-literals.md)的 `none` 最常作為和型別的一項出現。

```nabi
T | none
```

這種形式表達值可能存在也可能不存在，相當於其他語言中的 `Option<T>`、`T?` 或 `Maybe T`。

與 niche 最佳化結合（例如對類指標的 T），null 可以在沒有記憶體額外負擔的情況下表示 `none`。確切規則定義於 [niche 最佳化](207-niche-optimization.md)。

### 巢狀可選型別

這是把 2.6.4 的「泛型和型別的分層」規則套用到 `none`。若泛型 `T | none` 的 `T` 已經含有 `none`，外層的 `none` 層會保留下來，成為**巢狀可選型別**（類似 Swift 的 `T??`）。`Maybe<i32 | none>`，即以 `T = i32 | none` 代入 `Maybe<T> = T | none` 所得的 `(i32 | none)?`，有兩層。`none` 模式、`is none`、`?.`、`??` 與 `?` 作用於外層，非 none 的一側繫結到內層的 `T`（`i32 | none`）。

因此 `List<i32 | none>` 會把存放的 `none` 當作元素迭代，而 `get` / `pop` / `Dict` 查詢能區分「不存在」與「存放了 none」。

```nabi
let xs: List<i32 | none> = [1, none, 3];

for x in xs { print(f"{x:?}"); }  // 1, none, 3 (stored none is also an element)

match xs.get(1) {                // get: (i32 | none) | none (two layers)
    none => print("out of range"),     // outer none: no element
    v => match v {               // v: i32 | none (stored value)
        none => print("stored none"),
        n: i32 => print(n),
    },
}

let d: Dict<str, i32 | none> = {"a": none};
d.get("a") ?? 0i32;              // none: key present, value is none (?? strips only the outer layer)
d.get("b") ?? 0i32;              // 0: key absent
```

## 2.6.6. 與 `any` 的差異

[`any` 槽位](202-any-slot.md)的 `any` 是開放的槽位：任何型別都可能，變體可以自由增加。

相對地，和型別（`A | B`）是封閉的槽位：只有列出的變體是可能的。作為回報，編譯器知道每一個可能的項，並能檢查模式比對的窮盡性。

| 面向            | `any`                                  | `A \| B`                                       |
|-----------------|----------------------------------------|-----------------------------------------------|
| 可能的型別      | 所有型別                               | 只有列出的項                                  |
| 分派            | 執行時標籤檢查                         | 執行時標籤檢查（項較少，分支較少）            |
| 窮盡性          | 不可能                                 | 可能                                          |
| 記憶體          | 標籤 + 酬載（通用）                    | 標籤 + 最大的項（較緊湊）                     |
| 漸進性          | 漸進式（新型別不需修改程式碼即可）     | 靜態（增加變體會改變型別）                    |

## 2.6.7. 與 `enum` 的差異

在 `enum` 中，具名變體定義在容器內，變體是 `enum` 的一部分。

和型別直接使用既有的型別作為變體，因此沒有另外的變體名稱。

```nabi
enum Result { Ok(i32), Err(str) }       // variant names Ok / Err are defined inside the enum
type Either = i32 | str;                  // no variant names: just i32 or str
```

當變體帶有不同的意義而需要名稱時，選擇 `enum`。當原樣組合既有型別最自然時，選擇和型別。

## 2.6.8. 型別收窄

和型別的值可以透過模式比對或 `is` 檢查收窄為其中一項，方式與 [`any` 槽位](202-any-slot.md)相同。

```nabi
let v: i32 | str = ...;
match v {
    n: i32 => ...,           // inside this arm v is i32
    s: str => ...,           // inside this arm v is str
}
```

確切的收窄語法定義於模式與運算式規格。

## 2.6.9. 與型別別名結合

較長的和型別可以用 `type` 取別名（`type Num = i32 | f64;`）。循環的型別別名（即使經由泛型引數）是錯誤。型別別名的 `where` 放在被取別名的型別之後（`type A<T> = B<T> where T: X;`）。省略的引數取其參數的預設值，因此所有參數都有預設值的別名可以不帶引數撰寫：在 `type Vel<V = f64> = Q<V, 1, -1>;` 之下，`Vel` 是 `Vel<f64>`。診斷以程式碼為泛型型別所寫的別名指稱它（`expects Vel, found Len`，而不是 `Q<f64, 1, -1>`）；正在建置之套件所寫的別名，優先於相依套件所寫的別名。別名的 `where` 在每次使用該別名時都會檢查。當某次使用的引數指稱型別參數時，由使用處作用域內的約束決定，與泛型主體相同（[2.10.9](210-generics.md#2109-主體檢查)）：外層函式、`impl` 或 trait 的參數約束與 `where` 子句，以及 trait 的父 trait 與關聯型別約束。在 `type W<T> = S<T> where T: Num;` 之下，`fn get<T: Num>(x: W<T>)` 與 `impl<T: Num> W<T> { .. }` 是有效的；`fn get<T>(x: W<T>)` 即使從未被呼叫也是錯誤。宣告中的型別也一樣：`struct`、`enum` 或 `union` 的欄位、trait `impl` 的 trait 引數與關聯型別，以及 trait 方法的簽章，當別名的 `where` 不滿足時都會被拒絕，即使沒有任何東西使用它們；說明會指出應加上約束的泛型（``add `T: Num` to `F`'s generics``）。`any Tr` 的別名（`type Shown = any Display;`）也是這樣的別名：它代表該動態槽位。`type X = some Tr;` 不是別名，而是具名的不透明型別，即由其模組的函式固定的一個靜態型別（[4.1.15](../004-functions/401-function-declaration.md#4115-具名不透明型別type-x--some-trait)）。`type` 是上下文關鍵字，只有在項目開頭且後面接著識別字與 `=` 或 `<` 時才是關鍵字（[1.5.4](../001-lexical/105-keywords.md#154-上下文關鍵字)）。確切的別名語法定義於型別別名規格。

```nabi
type Number = i32 | i64 | f32 | f64;
type Maybe<T> = T | none;
```
