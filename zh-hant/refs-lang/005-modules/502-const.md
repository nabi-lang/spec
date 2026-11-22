# 5.2. `const`

`const` 關鍵字（上下文關鍵字，只在項目開頭是關鍵字，[1.5.4](../001-lexical/105-keywords.md#154-contextual-keywords)）宣告在編譯期決定的值。文法如下：

```
const_decl      ::= "const" identifier ":" type_expr "=" const_expr ";"
const_fn        ::= "const" fn_decl
const_block     ::= "const" block
```

## 5.2.1. 基本形式

```nabi
const PI: f64 = 3.14159f64;
const MAX_RETRY: i32 = 3i32;
const NAME: str = "nabi";
```

宣告依序由 `const` 關鍵字、名稱、`:`、型別、`=` 與一個 `const` 運算式組成。型別必須明確寫出；與 `let` 不同，它不會被推論。值的位置只接受 `const` 運算式，也就是能在編譯期求值的運算式。

## 5.2.2. 允許的位置

`const` 可以在下列所有位置宣告。

```nabi
// module level
const PI: f64 = 3.14159f64;

// inside a function body
fn area(r: f64) -> f64 {
    const FACTOR: f64 = 3.14159f64;
    FACTOR * r * r
}

// inside an impl block
impl Circle {
    const UNIT_AREA: f64 = 3.14159f64;
    fn unit() -> Circle { Circle { r: 1.0f64 } }
}

// inside a trait: associated const
trait Numbers {
    const ZERO: Self;
    const ONE: Self;
}
```

函式主體中的 `const` 是其區塊的項目（[3.9.1](../003-expressions/309-block-expressions.md#391-form)）：在整個區塊及其中宣告的項目內可見。它不讀取周圍程式碼的任何區域變數或參數；錯誤訊息會提示改寫成 `let`（`let half = n / 2;`）。它可以讀取其所在之泛型函式或 `impl` 的 const 參數，以及其型別參數的關聯常數與 `T::size()`：此時它在每個實例中各有一個值（5.2.7）。

```nabi
fn mid<const N: usize>(xs: [i32; N]) -> i32 {
    const HALF: usize = N / 2;      // one value per instance
    xs[HALF]
}
```

## 5.2.3. `const` 的意義：在每次使用處內嵌

`const` 是值而不是位址。其值在每個使用之處內嵌。

```nabi
const X: i32 = 42i32;

fn f() {
    let a = X;                    // a = 42i32 (X is inlined)
    let b = X + 1i32;              // b = 43i32 (X is inlined, then evaluated)
}
```

`const` 的結果不存在於任何記憶體位置，因此無法取得其位址。

```nabi
let p = &X;                       // ERROR (E0012): Nabi has no `&` borrows
```

Nabi 沒有 `&` 運算子，因此沒有任何運算式會取得 `const` 的位址。用於 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 的原始指標由[型別運算式](../002-types/201-type-expressions.md)中的指標型別定義。

## 5.2.4. 常數運算式與求值器

`const` 項目、陣列長度、const 泛型引數、`struct` 欄位預設值、`static` 初始化式，以及接收常數引數的 `const fn` 字面值處理器（[6.5.8](../006-meta-system/605-literal-magic.md#658-const-compatibility-of-literal-handlers)），全都使用同一個編譯期求值器。

能求值的東西：

- 值：各種寬度的整數、浮點數、`bool`、`char`、`str`、元組、`struct`、`enum`、和型別、固定長度陣列。
- 運算式：字面值、對其他 `const` 的參照、const 泛型參數、運算子、`const fn` 呼叫。
- 控制流程：`if`、`match`、`loop` / `while` / `for`、帶標籤的 `break`、`return`、`throw`。允許遞迴與泛型 `const fn`。

不能求值的東西：呼叫不是 `const fn` 的函式（I/O、時間、隨機數等）、讀取 `static`、參照外部的 `let` 變數。

下列情況是編譯錯誤：

- 整數溢位（包括摺疊只由字面值與常數構成之運算式的結果，`let m: u8 = 2 * 200`）、除以零、`MIN / -1`、移位位元寬度以上，以及超出範圍的浮點數 → 整數轉換。回繞算術只能透過 `wrapping_*`。
  被常數條件排除的程式碼永遠不會執行，也不受檢查：在 `N = 0` 的泛型實例中，`if N == 0 { 0 } else { a[N - 1] }` 的 `N - 1`、`if N == 0 { return ..; }` 之後區塊其餘部分中的 `N - 1`，以及 `N > 0 && a[N - 1] == x` 中的 `N - 1`，都不是錯誤。實例中的溢位會連同建立該實例的呼叫及其 const 引數一起回報。
- 求值期間的 panic 或未捕捉的 `throw`。
- 超出求值步數或呼叫深度的限制。限制由實作定義（nabi：4,000,000 步，深度 128）。
- `const` 之間的循環參照。

## 5.2.5. `const fn`：能在編譯期求值的函式

```nabi
const fn double(x: i32) -> i32 {
    x * 2i32
}

const fn gcd(a: i64, b: i64) -> i64 {
    if b == 0i64 { a } else { gcd(b, a % b) }
}

const TWO: i32 = double(1i32);                  // compile time
const TWELVE: i64 = gcd(24i64, 36i64);          // compile time

let y = double(user_input);                      // runtime call
```

- `const fn` 只能呼叫 `const fn`。`const fn` 內透過 `fn` 值或閉包的呼叫只在求值時才檢查，因為靜態檢查無法追蹤它們。標準函式庫把純函式標示為 `const fn`。
- 主體必須是純的。I/O 與存取可變的全域變數是錯誤。
- 在編譯期成功求值的結果，與相同引數在執行時的結果相等。在執行時會溢位的運算，在編譯期是錯誤（5.2.4）。
- `const fn` 不能接受型別作為引數（`const fn sizeof(T: type)` 是錯誤；說明會寫出 `const fn sizeof<T>() -> usize { T::size() }`）。型別內省屬於元區塊。
- `const fn` 也可以像一般函式一樣在執行時呼叫。

### 泛型 `const fn`

```nabi
const fn id<T>(x: T) -> T { x }

const A: i32 = id(42i32);                        // evaluated after monomorphization with T = i32
```

### 使用堆積的結果

若結果包含堆積配置（`List`、緩衝區、閉包等），它不會被嵌入二進位檔中。這樣的 `const` 在編譯期求值並檢查，在執行時於每次使用處建構。

每次使用都是新的值，因此對 `const` 的變更呼叫（對 `List` 的 `XS.push(4)`，或對 `const` 結構呼叫 `ref self` 方法）只會改變一份隨即被捨棄的複本。這是編譯錯誤：請把值複製到變數中再改變它（`let xs = XS; xs.push(4);`）。

## 5.2.6. `const` 與其他機制的正交性

```
generic <T>     = type parameterization (call-site specialize)
meta block      = code generation (impl, struct, etc. at AST level)
const fn        = value computation (compile-time pure function)
```

這三種機制是正交的。它們的領域不重疊，因此可以自由組合。

## 5.2.7. `const` 何時求值

模組層級、函式主體與 `impl` 層級的 `const` 在套件建置時求值一次。相依於泛型參數的 `const`（泛型 `impl` 的關聯 `const`、讀取 const 參數的函式主體 `const` 或 `const { .. }` 區塊）依每次單態化求值。不使用堆積的結果會被嵌入二進位檔，沒有執行時成本。

## 5.2.8. 與 `struct` 欄位預設值的關係

[`struct`](../002-types/203-struct.md) 的欄位預設值是常數運算式（5.2.4）。

## 5.2.9. 與 `const` 泛型的關係

[const 泛型](../002-types/211-const-generics.md)的引數是成為型別一部分的常數，由同一個求值器求值（5.2.4）。

## 5.2.10. `const` 區塊

`const { .. }` 是由求值器（5.2.4）在編譯期求值的運算式。其值是該區塊型別的常數；型別為 `()` 的區塊是一項檢查。

```nabi
fn chunk<const N: usize>(xs: [i32; N]) -> usize {
    const { assert(N > 0, "chunk needs at least one element") };
    const { N / 2 }
}
```

- 它讀取的東西與函式主體 `const` 相同：常數、`const fn` 呼叫、周圍程式碼的 const 參數，以及其型別參數的關聯常數與 `T::size()` / `T::align()`。`let` 變數是錯誤，其說明會拿掉 `const`。
- 在泛型函式中，它在實例的引數已知之後，對每個實例求值一次。泛型主體在其定義處的檢查（[2.10.9](../002-types/210-generics.md#2109-body-checking)）會對區塊做型別檢查，但不求值它。
- 對它求值時發生 panic 或 `assert` 失敗，是在建立該實例之呼叫處的編譯錯誤，錯誤訊息會指出該實例（`` const block failed in `chunk::<0>`: chunk needs at least one element ``），並以該 `assert` 作為次要位置。`nabi check` 也會回報它：它和 `nabi build` 一樣，會實例化程式所到達的東西。
- 通過的區塊在執行時沒有任何成本；其值和任何 `const` 一樣被嵌入。
- 求值器知道一般型別（數字、`bool`、`char`、`str`、`()` 或由它們組成的陣列）的 `T::size()`；對其他型別則是錯誤，其訊息會提示使用 const 參數或 `List`。
