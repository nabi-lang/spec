# 2.10. 泛型

泛型是為函式、型別、trait 與 `impl` 區塊提供型別參數的機制。在呼叫處或實例化處，會為每一組具體型別產生各自的編譯結果；這個過程稱為單態化。

```
generics        ::= "<" generic_param ("," generic_param)* ","? ">"
generic_param   ::= type_param | const_param
type_param      ::= identifier (":" type_bounds)? ("=" type_expr)?
const_param     ::= "const" identifier ":" type_expr ("=" const_expr)?
type_bounds     ::= type_expr ("+" type_expr)*
where_clause    ::= "where" where_pred ("," where_pred)* ","?
where_pred      ::= type_expr ":" type_bounds
```

> [!NOTE]
> `where` 是上下文關鍵字，只在函式簽章、`impl` 標頭，或 `struct` / `enum` / `union` / 型別別名標頭之後才是關鍵字（[1.5.4](../001-lexical/105-keywords.md#154-上下文關鍵字)）。

## 2.10.1. 參數宣告的位置

泛型參數出現在下列位置：

```nabi
fn id<T>(x: T) -> T { x }                       // function
struct Pair<A, B> { first: A; second: B; }      // struct
enum Result<T, E> { Ok(T), Err(E) }              // enum
union Bits<T> { v: T; raw: u64; }               // union
trait Convert<Target> { ... }                    // trait
impl<T> List<T> { ... }                          // impl
type Maybe<T> = T | none;                        // type alias
```

在每個位置，參數宣告的形式都相同：在名稱之後的 `<...>` 中列出參數名稱，以逗號分隔。

寫在 `impl` 區塊中的方法，不得宣告與該 impl 的泛型參數同名的泛型參數：`impl<K> W<K> { fn get<K>(..) }` 是錯誤（E0141），因為在方法內 `K` 可能指兩者之一；請為方法的參數改名（`fn get<K2>`），或使用 impl 的參數。trait 的預設方法寫在 trait 中而不是 impl 中，因此其參數會遮蔽實作它的 impl 中的同名參數：`impl<K, V> Iterator for DictIter<K, V>` 上的 `Iterator::min_by_key<K>` 有它自己的 `K`。

## 2.10.2. 參數的種類

### 型別參數

```nabi
fn first<T>(pair: (T, T)) -> T { pair.0 }
```

參數名稱遵循[識別字](../001-lexical/104-identifiers.md)的形式。依慣例使用 `T`、`U`、`K`、`V` 這類單一大寫字母或簡短的 PascalCase 名稱，但這不是強制的。

### `const` 參數

這類參數以值作為參數。確切規則另外定義於 [`const` 泛型](211-const-generics.md)。

### 型別引數推論

若呼叫處沒有寫出型別引數，由編譯器推論。除了引數型別之外，預期型別（`let` 的型別標註、引數位置的參數型別、回傳位置的回傳型別）也用於推論。

```nabi
fn empty<T>() -> List<T> { [] }

let xs: List<i32> = empty();        // T = i32, from the let's type annotation
takes_strs(empty());                // T = str, from the parameter type List<str>
```

沒有預期型別的 `let` 由之後的使用推論。在不帶標註的 `let x = f();` 中，若 `f` 的泛型回傳型別無法單靠呼叫決定，就由之後的使用決定。簽章一律是明確的。

- **範圍**：變數的作用域，即區塊的其餘部分，包括其中的閉包與 async 主體。
- **確定型別的使用**：把值傳到型別已固定之位置的使用：函式 / 方法引數、指派、有標註的 `let`、結構欄位、`return`、主體的值。運算子會依其另一個運算元確定開放字面值運算元的型別（`c += x`）；除此之外，運算子、f-string 的洞、模式與欄位讀取都不會確定型別。
- **空的建構式**：`[]`、`Dict::new()`、`Set::new()`、`channel()`、`Mutex::new([])` 的型別引數以相同方式確定，也可以透過 `d[k] = v` 以及元組 `let`（`let (tx, rx) = channel(); tx.send(1)`）確定。
- **相互確定**：多個開放的 `let` 可以相互確定（`let a = List::new(); let b = List::new(); a.push(b); b.push(1);` → `b: List<i32>`、`a: List<List<i32>>`）。
- **字面值的預設型別**：`i64` / `f64` 會讓位給之後同類（整數 / 實數）的具體使用。在 `ys.push(1); sum_i32(ys)` 中，`ys` 是 `List<i32>`。由 `let` 本身繫結的字面值（`let c = 0`）也同樣是開放的（[1.6.3](../001-lexical/106-numeric-literals.md#163-無後綴的字面值)）。
- **衝突**：兩個不同的使用不會形成和型別；那是錯誤（`xs.push(none); xs.push(1)`，與 [5.1.7](../005-modules/501-variable-declaration.md#517-型別標註與型別推論) 的分支規則相同）。單獨的 `let v = none;` 仍然是錯誤。
- 沒有上下文的閉包參數（[4.5.2](../004-functions/405-closure.md)）與方法路徑值（[3.5.3](../003-expressions/305-member-access-and-method-call.md#方法路徑值)），同樣由之後的呼叫確定（`let f = |x| x * 2; f(4)`、`let f = Box::get; f(b)`）。

診斷：

| 情況                 | 位置                                                      | 訊息範例                                                                   |
|----------------------|-----------------------------------------------------------|----------------------------------------------------------------------------|
| 始終無法決定         | 該 `let`                                                  | `` type annotations needed: cannot infer the type of `d` (`Dict<_, _>`) `` |
| 衝突                 | 使用處，並以確定型別之處作為次要標籤                      | `` conflicting uses of `c` (`List<_>`): `str` here, but `i32` fixed at 8:5 `` |
| 字面值被當作浮點數使用 | 使用處，並以宣告處作為次要標籤                          | `` `t` is used as `f64` here, but it holds an integer (`i64`) from `0` at 3:13 ``，說明 `` make it a float at the declaration: `let t = 0.0;` `` |

```nabi
let xs = empty();                   // still List<?>
takes_ints(xs);                     // T = i32 here (parameter List<i32>)
let zs = List::new();
zs.push(1);                         // zs: List<i32>, from a later method argument
let ys = empty();                   // ERROR: type annotations needed (no concrete use)
```

沒有引數的靜態 trait 方法以相同方式決定 `Self`。對型別參數的呼叫 `T::zero()` 與 trait 路徑呼叫 `Tr::m()` 從預期型別取得 `Self`。

```nabi
let z: i32 = num::Zero::zero();     // Self = i32 (core::num::Zero)
fn sum<T: num::Zero + ops::Add<Output = T>>(xs: List<T>) -> T {
    let acc: T = T::zero();          // Self = T
    ...
}
```

**關聯型別繫結。** 帶有繫結的約束 `T: Add<U, Output = V>`，讓 `V` 取得它所選之 impl 的關聯型別。該 impl 由 `T` 以及 trait 的引數（`U`）選出，因此只有在從引數或預期型別得知 `T` 與每個 trait 引數之後，才會讀取該繫結；在那之前 `V` 保持開放。沒有任何東西決定的 trait 引數會讓 `V` 無法決定（``type annotations needed``）：編譯器絕不會在某個型別對該 trait 的多個 impl 之間猜測。

```nabi
fn plus<T, U, V>(a: T, b: U) -> V where T: Add<U, Output = V> { a + b }
plus(7.0, 0.5)        // U = f64 from 0.5, so Add<f64> for f64: V = f64
plus(3.0, 4.0i)       // U = Complex<f64>: V = Complex<f64>
```

泛型 `enum` 的變體建構子，先從其引數決定實例，再從型別參數的預設值，再從它所繫結的 `let` 推論變數；否則就是錯誤 ``type annotations needed``。

```nabi
enum Pair<A, B> { One(A), Two(A, B) }
let p = Pair::Two(1, "a");          // Pair<i32, str>, from the arguments
```

## 2.10.3. trait 約束

約束說明參數必須滿足的 trait。

```nabi
fn add_all<T: ops::Add<T>>(items: List<T>) -> T { ... }
fn print_all<T: Display>(items: List<T>) { ... }
fn sort_all<T: Ord + Clone>(ref items: List<T>) { ... }
```

單一約束寫成 `Param: Trait`；多個約束以 `+` 連接。沒有約束時，只有每個型別都支援的東西適用於該參數：移動、`Reflect`、存入 `any`、與 `none` 組成和型別，以及 `is`（2.10.9）。

在約束、`where` 子句、`impl`、父 trait 清單或 `any Tr` 中提到的 trait，和其他名稱一樣解析：其他模組的 trait 需要 `use` 或路徑（[5.5](../005-modules/505-use.md)）。否則就是找不到名稱的錯誤，錯誤訊息會指出它寫在哪裡：``cannot find trait `Shape` (named in a bound)``、`(named in an impl)` 或 `(named as a supertrait)`，並附上應加入的 `use`。

## 2.10.4. where 子句

當約束變長時，最好把它們移到 `where` 子句中。

```nabi
fn complex<T, U, V>(a: T, b: U) -> V
where
    T: ops::Add<U, Output = V>,
    V: Display + Clone,
{
    ...
}
```

`where` 關鍵字放在參數宣告與主體之間，其後接以逗號分隔的約束。關聯型別上的約束也可以寫在這裡（例如 `T::Output: Foo`）。

出現在關聯型別繫結中的型別參數（`I: Iterator<Item = (A, B)>`，或 `where` 子句中的 `Self: Iterator<Item = (A, B)>`），會從引數的該關聯型別推論（對 `Self` 而言則是接收者的）。繫結的型別可以有結構：`(A, B)`、`(A, List<B>)` 與 `fn(A) -> B` 會繫結其中包含的每個參數。若關聯型別是其他形式，呼叫就是錯誤，錯誤訊息會說明項目必須長什麼樣子：``` `unzip` needs `Item` of the form `(A, B)`, but `ListIter<i32>` has `Item = i32` ```。

```nabi
fn unzip2<I: Iterator<Item = (A, B)>, A, B>(it: I) -> (List<A>, List<B>) { ... }
unzip2([(1, "a"), (2, "b")].iter());   // A = i32, B = str
```

型別別名的 `where` 子句（[2.6.9](206-sum-types.md#269-與型別別名結合)）在別名的每次使用處都成立；型別參數透過該使用處作用域內的約束滿足它。

## 2.10.5. 預設值

型別參數可以有預設值。

```nabi
trait Add<Rhs = Self> {
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

沒有預設值的參數不能接在有預設值的參數之後；預設值必須位於尾端。

在型別位置，不寫出有預設值之引數的泛型型別就是預設實例（`x: Vec3` 是 `x: Vec3<f32>`）。在運算式路徑中（`Vec3::new(..)`、`Vec3 { .. }`、`Vec3::X`、`zero()`），省略的引數和其他引數一樣推論：從引數、預期型別與之後的使用；只有在沒有其他東西確定它時才套用其預設值，而填入它的無後綴字面值會把該預設值當作自己的預設型別（[1.6.3](../001-lexical/106-numeric-literals.md#163-無後綴的字面值)）。透過這樣的路徑讀取的關聯項目，從預期型別取得其實例，否則從定義它的唯一 impl，否則取預設值：`let v: DVec3 = Vec3::X` 讀取 `Vec3<f64>` 的 `X`；`let v = Vec3::X` 讀取 `Vec3<f32>` 的。函式中不是數字型別的預設值（`C = List<Self::Item>`）在呼叫處取用。

```nabi
struct Vec3<T = f32> { x: T; y: T; z: T; }
fn zero<T: Num = f32>() -> T { T::zero() }

let a = Vec3::new(1.0, 2.0, 3.0);    // Vec3<f32>: the literals take T's default
let b: Vec3<f64> = a;                // a later use fixes it first: a is Vec3<f64>
let r = zero();                      // f32 unless a later use says otherwise
takes_f64(r);                        // r is f64
```

改變公開型別之參數的預設值，會改變使用它的程式碼中推論出的型別。

在顯示型別之處（診斷、說明文字、`type_name`），等於其參數預設值的尾端引數會被省略：對 `struct W<T = i32>` 而言，`W<i32>` 顯示為 `W`，`Dict<str, i32, RandomState>` 顯示為 `Dict<str, i32>`。

### 引數數量

泛型的 `struct`、`union`、`enum` 或型別別名，接受的引數最多與其參數一樣多，最少與其沒有預設值的參數一樣多。其他情況都是錯誤，錯誤訊息會指出該宣告：

```nabi
struct P<T, U> { a: T; b: U; }
struct D<T, U = i32> { a: T; b: U; }

let p: P<i64> = P { a: 1, b: 2 };          // ERROR: `P` takes 2 generic arguments but 1 was given
                                           //   help: `P` is declared `P<T, U>`; add one for `U`
let d: D<i32, i32, i32> = D { a: 1, b: 2 }; // ERROR: `D` takes 1 to 2 generic arguments but 3 were given
let e: D<i32> = D { a: 1, b: 2 };          // OK: `U` is `i32`
```

函式或方法可以接受比其參數少的明確引數（`f::<i32>(..)`）；其餘的由推論決定。給出更多是同樣的錯誤，給 trait 約束比該 trait 更多的引數也是（`` `Tr` takes at most 1 generic argument but 2 were given ``）。寫在列舉路徑上的引數指定其實例：`E::<i64>::A(1)` 是 `E<i64>`。

在預期型別之處寫出不帶引數的泛型型別（對 `struct S<T>` 寫 `x: S`），當某個參數沒有預設值時是錯誤（所有參數都有預設值的型別可以不帶引數撰寫）：`` `S` needs generic arguments ``，說明 `` `S` is declared `S<T>`; write it with them, e.g. `S<i32>` ``。

## 2.10.6. 單態化

泛型呼叫與實例化為每一組具體型別產生各自的編譯結果。

```nabi
fn id<T>(x: T) -> T { x }

let a = id(42i32);          // id<i32> is generated: an i32 → i32 function
let b = id(3.14f64);        // id<f64> is generated: an f64 → f64 function
let c = id("hello");         // id<str> is generated
```

具有相同型別組合的呼叫共用同一個結果，因此只編譯一次。結果的效能與直接呼叫相同，沒有間接呼叫成本。代價是編譯時間與二進位檔大小，大多會因捨棄無法到達的實例以及連結時的無用程式碼移除而抵銷。

### 單態化快取模型

[AOT](../000-meta/007-glossary.md#aot預先編譯) 建置收集呼叫處，並一次全部[特化](../000-meta/007-glossary.md#特化specialize)。在 [JIT](../000-meta/007-glossary.md#jit即時編譯) 模式中，一組型別在第一次呼叫時特化，並快取結果。兩種模式共用同一套特化機制，它是[執行引擎與使用者介面](../000-meta/001-execution-modes.md)之共通管線的一部分。

## 2.10.7. 與漸進式型別結合

[`any` 槽位](202-any-slot.md)中所述函式參數的漸進性處理如下。

```nabi
fn compute(x, y) { ... }
```

這段程式碼被轉成隱含的泛型 `fn compute<T, U>(x: T, y: U)`。不會合成任何約束（[4.2.2](../004-functions/402-parameters.md#422-省略型別)）：每個實例都在其呼叫處單態化，其主體也在那裡檢查，而錯誤也會指向該呼叫。

若在參數上寫出明確的 `any`，它就不會被單態化，而是成為執行時的動態槽位。這個差異是漸進式模式的核心。

## 2.10.8. 泛型參數的一致性

一個泛型參數在一次呼叫中只表示一個型別。它在不同的呼叫中可以表示不同的型別，但在同一次呼叫中其型別不能改變。

```nabi
fn pair<T>(a: T, b: T) -> (T, T) { (a, b) }

let p = pair(1i32, 2i32);     // OK: T = i32 for both arguments
let q = pair(1i32, 3.14f64);  // ERROR: ambiguous whether T is i32 or f64
```

多個參數（`pair<A, B>`）各自獨立推論。

## 2.10.9. 主體檢查

泛型函式的主體只能使用透過其約束得知的行為。

```nabi
fn double<T>(x: T) -> T {
    x + x       // ERROR: T has no Add bound
}

fn double<T: ops::Add<T, Output = T>>(x: T) -> T {
    x + x       // OK
}
```

由於這條規則，泛型主體的檢查與呼叫處無關，因此無論呼叫傳入什麼具體型別，主體都保證能編譯。這條規則擴充如下。

- trait 預設方法的主體是泛型主體：`Self` 是不透明的並受該 trait 約束，trait 參數也是不透明的。
- 每個型別（無論是否受約束）都支援移動、`Reflect`（[6.10.1](../006-meta-system/610-reflection.md#6101-reflect自動實作的-trait)）、存入 `any`、與 `none` 組成和型別，以及 `is`。
- 帶有任何隱含參數的函式（[4.2.2](../004-functions/402-parameters.md#422-省略型別)）整個豁免。
- const 參數在其值未知的情況下檢查；當型別的 const 運算式作為值相等時，型別就相等（[2.11.5](211-const-generics.md#2115-const-運算式中的算術)）。
- 非參數型別上的 `where` 子句在主體中被假設成立。
- 主體檢查失敗之主體的實例，不會再增加自己的錯誤。
- 對型別涉及型別參數之值的型別測試（`is`、型別模式），被當作可反駁的來檢查，並依實例決定。
- 對參數本身的型別測試（`if T is f32 { .. }`，2.10.13）以所測試的型別取代 `T` 來檢查其分支一次；每個實例只降階其型別所走的那個分支。

## 2.10.10. 泛型與 trait 作為型別

泛型是編譯期多型，透過單態化直接呼叫。

trait 作為型別是使用 vtable 分派的執行時多型，Nabi 在[型別系統概觀](README.md)中禁止它。

因此每當需要多型時，請使用泛型（本文件）或和型別（[和型別](206-sum-types.md)）。

## 2.10.11. 變異性

Nabi 在泛型實例之間沒有子型別關係：每個型別參數都是不變的。`List<i64>` 不是 `List<i64 | none>`，`Box<Cat>` 不是 `Box<any>`，`fn(i64 | none)` 也不是 `fn(i64)`，即使 `i64` 值能放進 `i64 | none` 槽位。放寬為和型別（[2.6.4](206-sum-types.md)）與放寬為 `any`（[2.2](202-any-slot.md)）適用於整個值，那是編譯器能包裝它之處；它絕不會深入泛型型別內部，因為 `List<i64>` 與 `List<i64 | none>` 以不同的配置存放其元素，並在其所有控制代碼之間共用一個緩衝區。要得到較寬的型別，請在建立值之處寫出它（`let xs: List<i64 | none> = [1, 2]`），或把元素複製到較寬型別的新集合中（`let ys: List<i64 | none> = []; for x in xs { ys.push(x) }`）。

唯一的放寬是在函式約束上：`fn(..) -> R + Send` 值可以放進 `fn(..) -> R` 槽位（[4.4.9](../004-functions/404-function-type.md#449-函式型別之間的子型別關係無)）。

## 2.10.12. 推論與預設值一覽

型別推論侷限於函式主體；簽章一律寫出。各條規則與其相關的結構放在一起；這份清單把它們收集起來。

- **泛型引數**來自呼叫的引數型別與預期型別（2.10.2，「型別引數推論」）；`const` 引數也一樣（[2.11.6](211-const-generics.md#2116-參數推論)）。沒有任何東西確定時，以 `::<..>` 寫出；省略是錯誤。
- **不帶標註的 `let`** 從其初始化式取得型別，而初始化式留下的開放部分由其作用域內之後的使用確定（2.10.2；[5.1.7](../005-modules/501-variable-declaration.md#517-型別標註與型別推論)）。兩個不一致的使用是錯誤；它們不會形成和型別。
- **和型別**以集合的方式比對：以 `T | none` 比對 `i32 | str | none` 得到 `T = i32 | str`（[2.6.4](206-sum-types.md#264-集合語意)）。分支結果不會被放寬為新的和型別；分支型別不同是錯誤，除非預期型別是和型別，或每個分支型別都是某個分支之和型別的成員。
- **字面值預設值**：沒有預期型別的整數字面值是 `i64`，浮點數字面值是 `f64`（[1.6.3](../001-lexical/106-numeric-literals.md#163-無後綴的字面值)），除非它填入一個有宣告預設值的參數；之後同類（整數、實數）的使用會取代預設值（2.10.2）。
- **型別參數的預設值**（`Rhs = Self`，2.10.5）只在參數沒有寫出且沒有任何東西推論它時才適用；填入該參數的字面值把該預設值當作自己的預設型別（[1.6.3](../001-lexical/106-numeric-literals.md#163-無後綴的字面值)），因此順序是：寫出的引數，然後是預期型別與使用，然後是參數的預設值，然後是 `i64` / `f64`。
- 沒有參數型別的**閉包**從預期的 `fn` 型別取得參數型別，否則從之後的呼叫（[4.5.2](../004-functions/405-closure.md)）。

沒有其他東西有預設值：不會退回到 `any`、`()` 或和型別。

## 2.10.13. 對型別參數的型別測試

`T is X`（其中 `T` 指稱作用域內的型別參數，或是 trait 預設方法或 impl 中的 `Self`）測試的是型別本身，不涉及任何值。它在每個實例中都是一個常數 `bool`：當該實例之 `T` 的每個值都通過 `v is X` 時為真。對單一型別 `X` 而言，這表示 `T` 就是 `X`；對和型別 `X` 而言，表示 `T` 是 `X` 的成員或其子和型別（[2.6.4](206-sum-types.md#264-集合語意)）。`T = any` 只通過 `T is any`。

```nabi
fn sum<T: num::Num + Clone>(xs: List<T>) -> T {
    if T is f32 {
        return sum_f32(xs);          // xs: List<f32>; the f32 result is a T
    } else if T is f64 {
        return sum_f64(xs);
    }
    let acc = T::zero();             // generic fallback, checked against the bounds
    for x in xs { acc = acc + x; }
    acc
}
```

- **細化。** 在單一型別 `X`（不是和型別，也不是 `any`）的 `if T is X { B }` 中，`B` 在定義處以 `X` 取代作用域內所有的 `T` 檢查一次：參數、在分支外宣告的區域變數、其中的閉包以及回傳型別。`xs: List<T>` 在 `B` 中是 `List<f32>`，而 `f32` 是一個 `T`。型別測試的 `&&` 鏈（`if T is f32 && U is i32`）會細化每個參數；其他條件可以加入該鏈。和型別 `X` 不細化任何東西。
- **其他地方不細化。** `else` 分支、`if` 之後的程式碼、型別測試的 `||` 以及否定的測試（`!(T is f32)`）都使用一般的泛型檢查。若型別測試的 `||` 所在分支需要 `T` 是單一型別，就是錯誤，錯誤訊息會提示為每個型別各寫一個分支。
- **依實例。** 一個實例只降階其型別所走的分支：另一個分支永遠不會為它降階，因此在那裡它可以是型別錯誤的。快速路徑在執行時不需要任何測試，也不需要經過 `any` 來回轉換。
- **不繫結。** `is` 之後的型別指稱封閉的型別或作用域中已有的參數；沒有宣告 `U` 的 `T is List<U>` 是錯誤，`T is v: X` 也是錯誤：沒有值可以繫結。
- **作為值。** `let fast = T is f32;` 在每個實例中是一個 `bool` 常數；只有 `if` 條件會細化。
- 型別以 `is` 比較；`T == f32` 是錯誤，其修正建議是 `is` 形式。
- 值測試與型別測試不同：在 `T = i32 | str` 時，`x is i32` 問的是這個值是否為 `i32`，而 `T is i32` 為假，因為 `T` 的值並非全都是 `i32`。

當型別的集合是開放的時候，請使用 trait。對保留泛型後備路徑的快速路徑，請使用 `if T is`：後備路徑會 panic 或為 `unreachable` 的鏈是封閉的型別集合，`closed_type_switch` lint 會建議改用 trait，並為每個型別各寫一個 impl。與 blanket impl 重疊的具體 impl（E0075）寫成 blanket impl 中的 `if Self is X` 分支。
