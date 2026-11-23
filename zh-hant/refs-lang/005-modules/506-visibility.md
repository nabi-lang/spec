# 5.6. 可見性

為每個項目（函式、型別、trait、模組、`const`、`static`）賦予可見性，可以控制它公開到什麼程度。

```
visibility      ::= "pub" ("(" reach ")")?
reach           ::= "pkg" | "sub"
```

## 5.6.1. 四個等級

可見性有四個等級：

| 關鍵字     | 意義                                                      |
|------------|-----------------------------------------------------------|
| （無）     | 模組私有。只在同一個模組內。                              |
| `pub(sub)` | 只在同一棵模組樹的子模組內。                              |
| `pub(pkg)` | 只在同一個套件內。                                        |
| `pub`      | 外部套件也可以。                                          |

最窄的範圍是預設的私有；最寬的是 `pub`。工具鏈自己的套件之間也共用一些標示為 `#[internal]` 的 `pub` 項目，其他套件不能使用它們（[6.2.15](../006-meta-system/602-built-in-attributes.md#6215-internal-由工具鏈的套件共用)）；這個標記不是可見性等級。

`pkg` 與 `sub` 是上下文關鍵字，只在 `pub(...)` 的括號內作為限定詞；在其他位置它們是一般識別字（見 [1.5.4 可見性限定詞](../001-lexical/105-keywords.md#可見性限定詞pkg--sub)）。

## 5.6.2. 預設：沒有可見性關鍵字

沒有可見性關鍵字的項目是模組私有的，只在同一個模組內可見。

```nabi
mod m {
    fn helper() -> i32 { 42i32 }   // module-private (default)
    pub fn api() -> i32 { helper() }
}

let r = m::api();                   // OK
let h = m::helper();                // ERROR: module-private
```

私有的 `mod` 從宣告它的父模組以及該父模組的子模組中可見。

## 5.6.3. `pub`：最寬的公開

要最廣泛地公開，請加上 `pub`。

```nabi
pub fn calculate(x: i32) -> i32 { ... }
pub struct Point { pub x: f64; pub y: f64; }
pub trait Display { ... }
```

以這種方式宣告的項目可以用 `use` 匯入，並從其他套件使用。

## 5.6.4. `pub(pkg)`：在套件內

`pub(pkg)` 表示在套件內公開。

```nabi
pub(pkg) fn internal_helper() -> i32 { ... }
```

它從同一個套件中的任何模組可見，但從外部套件不可見。相依套件以及標準函式庫的每個層級也都是各自獨立的套件（[5.7.14](507-paths.md#5714-套件之間的路徑)）。

這個等級用於套件內部的函式庫工具：不是公開 API、但必須在多個模組之間共用的函式與型別。

## 5.6.5. `pub(sub)`：子模組

`pub(sub)` 用於只在模組樹的一個分支內共用。

```nabi
mod parent {
    pub(sub) fn shared_within_subtree() -> i32 { ... }

    pub mod child {
        pub fn use_it() -> i32 { sup::shared_within_subtree() }   // OK
    }
}
```

它只在 `parent` 以及 `parent` 的所有後代模組中可見，在 `parent` 的祖先或其兄弟中不可見。

這個等級用於一組模組內部的工具，只在模組樹的一個分支內共用。

## 5.6.6. 等級之間的包含關係

可見性構成單調遞增的格。

```
pub  ⊇  pub(pkg)  ⊇  pub(sub)  ⊇  private
```

`pub(sub)` 的包含關係只在相關的模組之間有意義，也就是定義它的模組及其後代。

- `pub` 對每個呼叫者可見。
- `pub(pkg)` 只在同一個套件內可見。
- `pub(sub)` 只在同一棵模組樹內可見。
- 私有只在同一個模組內可見。

## 5.6.7. glob 匯入與可見性

glob 匯入（`*`、`**`，包括隱含的 `use core::prelude::*`）只匯入從匯入模組可見的項目。只有可及範圍包括該模組的項目才會被拉進來，而不可見的項目既不會隱藏名稱，也不會造成衝突。沒有逐項目的 glob 阻擋標記；要縮小對外的介面，請建立另外的 `prelude` 模組，並以 `pub use` 精選其內容。

## 5.6.8. `struct` 欄位的可見性

每個欄位都可以被賦予自己的可見性。

```nabi
struct Account {
    pub id: i64;                    // readable externally
    pub balance: f64;               // readable externally
        secret: str;                // module-private
}
```

欄位的可見性與 `struct` 本身的可見性獨立運作。

```nabi
pub struct Counter {                // the struct itself is visible externally
        count: i32 = 0i32;          // the field is module-private (default)
}

let c = Counter { count: 0i32 };    // ERROR: private field, cannot construct directly from outside
let c = Counter::new();             // OK: factory method (if any)
```

這是公開一個型別、同時防止直接實例化的標準模式（newtype、不透明型別）。

結構字面值會檢查它所寫之欄位的可見性。只有當使用處看不到的欄位有預設值時（[2.3.3](../002-types/203-struct.md#233-欄位預設值)），字面值才可以省略它；沒有預設值的私有欄位，會讓其範圍之外的每個字面值都成為錯誤，此時改以其模組的函式建立該值。

```nabi
pub struct Opts {
    pub depth: i32 = 4;
        seed: i64 = 7;              // private, with a default
}
pub struct Keyed {
    pub label: str = "k";
        key: i64;                   // private, no default
}

let o = Opts { depth: 1 };          // OK: `seed` takes its default
let k = Keyed { label: "x" };       // ERROR outside the module: `key` has no default
```

### 元組結構的欄位

元組結構的每個欄位也有自己的可見性，預設為私有。

```nabi
pub struct S(pub i32, i32);         // field 0 is pub, field 1 is private
```

- 以 `S(a, b)` 建構需要每個欄位在使用處都可見。若有任何欄位是私有的，在其可見範圍之外要使用建構函式（`NonZero::new`）。
- 讀取或指派 `x.N` 需要欄位 N 可見。
- 在模式 `S(p0, p1, ..)` 中，子模式不是 `_` 的每個位置都必須可見。帶具名欄位的結構模式也是如此（只檢查觸及的欄位）。
- 緊接在 `pub` 之後的 `(` 被讀作可見性限制（`pub(pkg)` 等）。要讓元組型別的欄位公開，請再加一對括號：`struct S(pub ((i32, i32)));`。

## 5.6.9. `enum` 變體的可見性

變體沒有另外的可見性；它們跟隨 `enum` 的可見性。在變體之前寫可見性關鍵字是編譯錯誤。

```nabi
pub enum Shape {
    Circle(f64),                    // pub (follows Shape)
    Square(f64),
}

pub enum Bad {
    pub A,                          // ERROR: no visibility on variants
}
```

變體的酬載欄位也沒有逐欄位的可見性；它們跟隨 `enum`。與 `struct` 欄位不同，隱藏單一變體會讓外部的 `match` 無法窮盡，因此有這條規則。依這條規則，標準函式庫的 `pub enum Result<T, E> { Ok(T), Err(E) }` 讓 `Ok` / `Err` 公開。

## 5.6.10. `impl` 方法的可見性

固有 `impl`（[`impl`](../002-types/209-impl.md)）的方法預設為模組私有；要公開它們，請明確標示 `pub`。

```nabi
impl Account {
    pub fn balance(self) -> f64 { self.amount }
        fn _internal_check(self) { ... }
}
```

trait `impl` 的方法跟隨 trait 的可見性，不能另外標示。

## 5.6.11. `use` 的可見性

透過 [`use`](505-use.md) 的匯入也有可見性。

```nabi
pub use foo::bar;                   // re-export: visible externally too
pub(pkg) use foo::bar;              // within the package only

use foo::bar;                       // module-private (default)
```

重新匯出與 [`use`](505-use.md) 的可見性一節結合，決定對外的介面。

## 5.6.12. 可見性的位置

可見性關鍵字作為項目宣告的第一個語彙單元出現。

```nabi
pub fn f() { ... }
pub(pkg) fn g() { ... }
pub struct S { ... }
pub mod m { ... }
pub const X: i32 = 0;
pub static Y: i32 = 0;
pub use foo::bar;
pub trait T { ... }
```

它可以附加在 `fn`、`struct`、`enum`、`union`、`trait`、`impl`（不是在 `impl` 本身上，而是在每個方法上）、`mod`、`const`、`static`、`use` 與 `type`（別名）的每一種宣告形式上。

## 5.6.13. 可見性與名稱解析

與 [`use`](505-use.md) 結合時，項目只有在下列兩者都成立時，才在目前模組中可見。

1. 項目的可見性到達目前的模組。路徑的每個模組片段也都必須可見（`a::b::f` 要求 `a`、`b` 與 `f` 都可見）。
2. 項目直接可見（子模組，或以 `use` 匯入的名稱）。

兩者都成立時，該項目就可以用其簡短名稱使用。

對於可見性不足的項目，編譯器診斷會說明它為何不可見，並指出能到達該使用處的可見性：在其套件內任何地方使用要 `pub(pkg)`，在項目所屬模組之下的模組使用也是（私有項目對子模組不可見），從另一個套件使用要 `pub`。

meta 樣板或編譯器所寫出的程式碼，從其定義所在的模組檢查，而不是從其展開之處（[6.9.9](../006-meta-system/609-template-tokens.md#展開中的名稱)）：樣板可以呼叫其自身模組的私有項目以及其套件的 `pub(pkg)` 項目，而使用處傳入的東西則在使用處檢查。

```text
error[E0034]: function `foo::secret` is private to module `foo` and cannot be used from `<root>` (add `pub(pkg)` to expose it)
```
