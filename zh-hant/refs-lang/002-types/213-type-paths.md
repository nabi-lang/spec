# 2.13. 型別路徑

以 `::` 語彙單元（[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)）連接的一串識別字，是表示某個項目的路徑。本文件只定義型別位置的路徑與關聯項目存取；模組匯入的確切意義另見 [`use` 宣告](../005-modules/505-use.md)。

```
type_path       ::= path_root ("::" path_segment)*
path_root       ::= identifier | "pkg" | "sup"
path_segment    ::= identifier type_args?
                  | identifier "::" "<" type_expr ">" "::" identifier  // assoc item
```

## 2.13.1. 模組路徑

若型別名稱定義於另一個模組，請把通往該位置的模組路徑與型別名稱一起寫出。

```nabi
core::time::Duration
core::collections::List
pkg::geom::Point
```

每個識別字指稱一個模組或項目。最後一個識別字是型別名稱；前面的部分是說明在哪裡能找到該型別的路徑。

## 2.13.2. 路徑的起點

路徑的第一個識別字必須是目前模組中可見的名稱：其子模組之一的名稱、以 `use` 匯入的名稱，或上下文關鍵字。上下文關鍵字是表示套件根的 `pkg` 與表示父模組的 `sup`；它們的確切意義定義於[路徑](../005-modules/507-paths.md)。

## 2.13.3. 關聯項目存取

`::` 也用來存取附加在型別上的項目。

```nabi
i32::MAX                  // const item of an integer type
Point::origin()           // static method
List::<i32>::new()        // generic given with turbofish
T::Output                 // associated type of a trait parameter
```

關聯項目存取有幾種形式。靜態方法以 `Type::method(args)` 呼叫。關聯 `const` 以 `Type::CONSTANT` 參照，trait 參數的關聯型別以 `T::AssocType` 參照。

經由型別別名的路徑指稱被取別名之型別的項目。在 `type B = S<i32>;` 之下，`B::mk(x)` 是 `S::<i32>::mk(x)`；在 `type Al<T> = S<T>;` 之下，`Al::mk(x)` 是引數經推論的 `S::mk(x)`，`Al::<i32>::mk(x)` 是 `S::<i32>::mk(x)`。不帶引數指稱的泛型別名，只有在被取別名之型別的引數是該別名的相異參數時，才把引數交給推論；否則，由與別名的固定引數一致、型別為被取別名之型別的有型別槽位提供引數（在 `type Matrix<T> = Array<T, 2>;` 之下的 `let m: Matrix<f64> = Matrix::zeros([2, 3]);`），或者寫出引數（對 `type Names<V> = Dict<str, V>;` 寫 `Names::<i32>::new()`）。

經由別名的結構字面值與結構模式指稱被取別名的結構。在 `type Q = P<f64>;` 之下，`Q { x: 2.0 }` 是 `P::<f64> { x: 2.0 }`，模式 `Q { x }` 比對 `P<f64>`。不帶引數寫出的泛型別名依相同規則推論引數（對 `type A<T> = P<T>;` 寫 `A { x: 3 }`）；否則寫出引數（對 `type D<T> = P<List<T>>;` 寫 `D<i64> { x: [1] }`）。

## 2.13.4. 完整限定路徑

當同一個方法名稱存在於多個 trait 與固有 `impl` 中而產生歧義，或必須說明關聯型別是透過哪個 trait 參照時，請使用完整限定形式。

```nabi
<i32 as Clone>::clone(42i32)
<T as ops::Add<U>>::Output
<Self as Iterator>::next(ref self)
```

完整限定形式是 `<Type as Trait>::item`。在角括號內寫出 `Type as Trait`，其後接 `::` 以及項目名稱或方法呼叫。這裡的 `as` 是一種關鍵字用法，說明型別是透過哪個 trait 來看待。同一個 `as` 語彙單元在運算式位置（二元用法）有不同的意義，定義於運算式規格。

完整限定形式在型別位置（作為關聯型別的 `<T as Tr>::X`）與呼叫位置（`<T as Tr>::m(x)`）都有效。對泛型 trait 而言，會選出與引數相符的 impl。`<T as Tr<A>>::X` 以及在約束 `T: Tr<A>` 之下的 `T::X`，是 `Tr<A>` impl 的 `X`，它可能與同一型別之 `Tr<B>` impl 的 `X` 不同。`<Sq as Shape>::Out` 這樣的投影在函式簽章、`static` 型別、結構欄位、列舉酬載與型別別名（無論是否為泛型）中都有效，不論型別、trait 與 impl 的宣告順序為何。在限定的關聯型別之後的呼叫 `<T as Tr>::X::m()` 在運算式中是有效的。

完整限定形式在一般使用中很少出現。只有在需要消除歧義或自動推論失敗時才會明確使用。

## 2.13.5. Turbofish：運算式位置的泛型

要在值位置給出泛型引數，使用 `::<...>` 形式。

```nabi
parse::<i32>("42")
collect::<List<f64>>(iter)
```

也就是說，在函式或方法名稱之後附加 `::<...>`，以在該位置給出泛型引數。在型別位置（`:` 之後、`->` 之後、`as` 之後等），直接寫 `<...>` 而不需要 `::`。

存在這個差異，是因為在值位置 `<` 會與比較運算子產生歧義。

## 2.13.6. 絕對型別路徑

從根開始的絕對路徑以 `pkg::` 撰寫（[路徑](../005-modules/507-paths.md)）。Nabi 沒有以 `::` 開頭的形式。

## 2.13.7. 路徑與和型別結合

路徑本身是[型別運算式](201-type-expressions.md)的簡單名稱或路徑形式，可以與和型別（`|`）、泛型套用（`<...>`）等其他組合子自由混用。

```nabi
core::io::Lines<T>
pkg::Foo | std::Bar
List<core::range::Range<i32>>
```

優先順序依[型別運算式](201-type-expressions.md)中的定義適用：路徑（`::`）與泛型套用（`<...>`）的結合比和型別（`|`）緊。

## 2.13.8. 空路徑與單獨的 `::`

空路徑（只有 `::` 而沒有識別字）一律是編譯錯誤。同樣地，識別字之間連續的 `::`（`A::::B`）也是錯誤。
