# 2.8. `trait`

`trait` 是宣告一組共通行為的容器。實作了該行為的型別就滿足該 trait。

本文件只定義 trait 的宣告形式。實作 trait 的 `impl` 區塊定義於 [`impl`](209-impl.md)。

## 2.8.1. 基本形式

trait 宣告的文法如下：

```
trait_decl  ::= visibility? "unsafe"? "trait" identifier generics? supertraits? where_clause? "{" trait_item* "}"
supertraits ::= ":" type_expr ("+" type_expr)*
trait_item  ::= method_sig
              | assoc_type_decl
              | method_default
              | assoc_const_decl
method_sig  ::= fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? ";"
method_default ::= fn_modifiers "fn" identifier "(" param_list? ")" throws_clause? return_type? block
assoc_type_decl ::= "type" identifier (":" type_expr ("+" type_expr)*)? where_clause? ("=" type_expr)? ";"
assoc_const_decl ::= "const" identifier ":" type_expr ("=" const_expr)? ";"
```

trait 宣告的範例：

```nabi
trait Describe {
    fn describe(self) -> str;
}

trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}

trait Iterator {
    type Item;
    fn next(ref self) -> Item | none;
}
```

trait 主體以 `{ ... }` 括起來。主體可以包含方法簽章、關聯型別與預設方法主體。

## 2.8.2. 方法簽章

方法簽章的寫法如下：

```nabi
fn name(params) -> ReturnType;
```

沒有主體的簽章是 trait 要求的方法；實作必須提供主體。帶有主體時，它是預設方法，除非實作覆寫它，否則使用該主體。`self` 與 `ref self` 參數的意義定義於函式規格。

同時帶有必要方法與預設方法的範例：

```nabi
trait Greeter {
    fn greet(self) -> str;                          // required
    fn greet_loudly(self) -> str { self.greet() }    // default: may be overridden
}
```

## 2.8.3. 關聯型別

關聯型別讓 trait 在每個實作中有不同的型別。

```nabi
trait Iterator {
    type Item;
    fn next(ref self) -> Item | none;
}
```

關聯型別宣告為 `type Name;`，由實作填入具體型別。在一個實作中，關聯型別固定為恰好一個具體型別；一個實作不能同時有多種可能。

關聯型別可以用在同一個 trait 內的簽章中，例如上面的 `Item | none`。它也可以用在 throws 子句中，例如 `throws (Self::X)`，trait 與 impl 的簽章皆可。

### 預設值

關聯型別可以用 `type X: Bounds = T;` 提供預設值。impl 要麼省略該關聯型別而使用預設值，要麼以 `type X = U;` 覆寫它；沒有預設值的關聯型別必須提供（E0072）。預設值在 trait 的定義處解析，並代入 trait 的參數。

```nabi
trait TryFrom<U> {
    type Error = ConversionError;
    fn try_from(value: U) throws (Self::Error) -> Self;
}
impl TryFrom<RawPath> for Path { type Error = Utf8Error; ... }   // overridden
impl TryFrom<Celsius> for Kelvin { ... }                           // Error = ConversionError
```

在 trait 及其所有 impl（trait impl 與固有 `impl`）內，單獨的名稱 `Item` 與 `Self::Item` 都表示關聯型別，在方法簽章與主體中皆可使用。在 impl 中，它是該 impl 所選的具體型別。若關聯型別遮蔽了外部同名的型別（模組的 `struct Item`、基本型別等），單獨的名稱在那裡就有歧義：在 trait 或 impl 內使用它是編譯錯誤，其說明會要求寫 `Self::Item` 或外部型別的路徑（`pkg::Item`、`std::process::Output`）。宣告這樣的關聯型別而不使用單獨的名稱會發出警告（`assoc_type_shadowing`，[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)）。同名的 trait 不是型別，因此不會發出警告（`trait IntoFuture { type IntoFuture; }` 沒有問題）。

若一個型別實作了兩個擁有同名關聯型別的 trait，在該型別的 impl 主體中單獨的 `X` / `Self::X` 就有歧義，是錯誤。此時請以 `<Self as Tr>::X` 指明 trait。

```nabi
impl Iterator for Stream { type Item = i32; ... }
impl Source for Stream { type Item = str; ... }

impl Stream {
    fn first(self) -> Self::Item { ... }                   // ERROR: Iterator::Item? Source::Item?
    fn first(self) -> <Self as Iterator>::Item { ... }     // OK
}
```

### 從外部參照關聯型別：`T::AssocType`

要從外部參照受 trait 約束之參數的關聯型別，使用 `T::AssocType` 記法。

```nabi
fn first_item<T: Iterator>(ref it: T) -> T::Item | none { it.next() }
```

若約束是泛型 trait `T: Tr<A>`，`T::X` 是 `Tr<A>` impl 的 `X`，因為一個型別可以用不同的 `X` 實作 `Tr<A>` 與 `Tr<B>`（`<T as Tr<A>>::X` 相同，[2.13.4](213-type-paths.md#2134-fully-qualified-path)）。在約束 `T: Tr<A> + Tr<B>` 之下，或在兩個約束 trait 都宣告了 `X` 時，單獨的 `T::X` 有歧義，是錯誤；請寫 `<T as Tr<A>>::X`。關聯常數（2.8.3a）也是如此。

這個記法是使用[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)之 `::` 路徑分隔符的關聯項目存取。路徑的一般規則（模組路徑、turbofish、完整限定形式）定義於[型別路徑](213-type-paths.md)。

## 2.8.3a. 關聯常數

trait 可以用 `const N: T;` 宣告關聯常數，並以 `= value` 提供預設值。每個 `impl` 設定其值（有預設值時可以省略），而在受該 trait 約束的型別參數上，它讀作 `T::N`。

```nabi
trait Shape {
    const SIDES: u32;
    const NAME: str = "shape";
}

impl Shape for Square { const SIDES: u32 = 4; }

fn sides<T: Shape>() -> u32 { T::SIDES }      // same as <T as Shape>::SIDES
```

在針對泛型型別的 `impl` 中，關聯常數在每個實例化中可以有不同的值（`impl<const N: usize> Shape for Poly<N> { const SIDES: u32 = N as u32; }`）。有歧義時，使用完整限定形式 `<T as Tr>::N`（[2.13.4](213-type-paths.md#2134-fully-qualified-path)）。

## 2.8.4. 父 trait

trait 可以要求它只能為滿足另一個 trait 的型別實作。

```nabi
trait Eq {
    fn eq(self, other: Self) -> bool;
}

trait Ord: Eq {
    fn cmp(self, other: Self) -> Ordering;
}
```

父 trait 寫在 `:` 之後。因此實作 `Ord` 也必須實作 `Eq`。多個父 trait 以 `+` 連接，例如 `trait Foo: Bar + Baz`。泛型 impl 必須為其約束所允許的每個實例化，讓目標型別具備父 trait；這一點在 impl 處檢查。

## 2.8.5. 泛型 trait

trait 也可以接受型別參數。

```nabi
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}

trait Convert<Target> {
    fn convert(self) -> Target;
}
```

型別參數宣告在 trait 名稱之後的 `<...>` 內，參數可以有預設值，例如 `Rhs = Self`。`Self` 是表示實作該 trait 之型別的關鍵字（[關鍵字](../001-lexical/105-keywords.md)）。

泛型 trait 確切的單態化規則定義於[泛型](210-generics.md)。

## 2.8.6. where 子句：對參數的額外約束

可以在 trait 宣告或方法簽章上寫出額外的 trait 約束。

```nabi
trait Sortable<T> where T: Ord {
    fn sort(ref self);
}

trait Container {
    type Item where Item: Clone;
    fn first(self) -> Item;
}
```

約束接在 `where` 關鍵字之後，形式為 `Type: Trait`，以逗號分隔。附加在 trait 本身時，約束整個 trait；附加在方法上時，只約束該方法。

## 2.8.7. trait 方法的分派

呼叫 trait 方法時，編譯器查看呼叫處的接收者型別，並靜態地呼叫為該型別實作的方法。

```nabi
let p = Point { x: 1.0, y: 2.0 };
let s = p.describe();       // Point's Describe implementation is called statically
```

對靜態型別的接收者而言，這種分派一律是靜態的，因為[型別系統概觀](README.md)禁止把 trait 當作型別（把 trait 名稱寫成變數、參數或回傳型別）。只有對 `any Trait` 值的呼叫才會在執行時分派（[2.2.10](202-any-slot.md#2210-constrained-any-any-trait)）。

因此需要多型時，請使用泛型（[泛型](210-generics.md)）、和型別（[和型別](206-sum-types.md)）或 `any Trait`。

## 2.8.8. 可見性

trait 的項目沒有自己的可見性：它的方法、關聯型別與常數跟隨 trait 的可見性，就像 `enum` 的變體跟隨 `enum` 一樣。在 trait 項目上寫 `pub` 是錯誤（E0016），在 trait `impl` 的方法上寫 `pub` 也是（[2.9.6](209-impl.md#296-method-visibility)）。`fn_modifiers` 與函式宣告的相同（[4.1](../004-functions/401-function-declaration.md)）；`unsafe trait` 在 [4.15.4](../004-functions/415-unsafe.md#4154-in-traits) 中說明。可見性的確切規則定義於[可見性](../005-modules/506-visibility.md)。

`#[sealed]` trait 只在其定義套件中實作；其他套件仍可在約束與呼叫中使用它（[6.2.15](../006-meta-system/602-built-in-attributes.md#6215-sealed-impls-only-in-the-defining-package)）。

## 2.8.9. 同一模組內不得重複定義

在同一個模組中宣告兩個同名的 trait 是編譯錯誤。在一個 trait 中宣告兩個同名的方法是編譯錯誤（沒有多載）。
