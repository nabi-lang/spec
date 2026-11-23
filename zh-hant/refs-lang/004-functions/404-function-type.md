# 4.4. 函式型別

函式與閉包是值，而這些值有型別。本文件詳細涵蓋[型別運算式](../002-types/201-type-expressions.md)的函式型別形式。

## 4.4.1. 形式

函式型別的形狀如下：

```nabi
fn(ParamTypes) -> ReturnType
fn(i32, i32) -> i32
fn(str) -> bool
fn() -> ()
```

`fn` 關鍵字之後接括號中的參數型別清單，然後是 `->` 與回傳型別。不寫參數名稱，只寫型別。省略回傳型別時是 unit，因此 `fn(i32, i32)` 與 `fn(i32, i32) -> ()` 被視為相同。

可能擲回錯誤之函式的型別，在與函式宣告相同的位置寫出 [`throws`](414-throws.md)。

```nabi
fn(str) throws -> i32               // untyped: throws (any Error)
fn(str) throws (ParseError) -> i32  // typed
fn(Path) throws (IoError)           // return type omitted: unit
```

`unsafe fn(..) -> R` 是呼叫時需要 `unsafe` 之函式的型別。`unsafe fn` 與 C 函式作為值時具有這種型別。安全的 `fn` 值可以用在預期 `unsafe fn` 之處，反之則不行（[4.15.3](415-unsafe.md#unsafe-fn-型別)）。

`throws` 是函式型別的一部分，因此呼叫會擲回錯誤之函式型別的值也必須以 `try` 標示。`rethrows` 只用於函式宣告，絕不用於函式型別。

## 4.4.2. 函式值與閉包值的型別

函式與閉包都以相同形式的函式型別表達。

```nabi
fn add(x: i32, y: i32) -> i32 { x + y }
let f = add;                                    // f: fn(i32, i32) -> i32

let g = |x: i32| -> i32 { x * 2 };              // g: fn(i32) -> i32
let h: fn(i32) -> i32 = |x| x + 1;              // explicit type
```

由於函式與閉包具有相同的函式型別，在任何需要函式型別之處，兩者都適用。

需要 trait 之處也是如此。閉包無論是否擷取，都是其 `fn(..) -> R` 型別的值，因此它滿足為該型別實作的 trait：它可以是約束 `F: Tr` 的引數、`Tr` 方法的接收者以及其運算子的運算元，而 blanket `impl<F: Tr> Other for F` 也涵蓋它。當約束的 trait 為閉包參數數量的某一個 `fn` 型別實作時，該型別為閉包未標註的參數提供型別。閉包也能透過其 `fn` 成員填入 `fn(..) -> R | none` 槽位。

```nabi
trait Integrand { fn integral(self, a: f64, b: f64) -> f64; }
impl Integrand for fn(f64) -> f64 {
    fn integral(self, a: f64, b: f64) -> f64 { (self(a) + self(b)) * 0.5 * (b - a) }
}
fn integrate<F: Integrand>(f: F, a: f64, b: f64) -> f64 { f.integral(a, b) }

let k = 3.0;
integrate(|x: f64| -> f64 { k * x }, 0.0, 2.0);     // 6: the closure as fn(f64) -> f64
integrate(|x| x * 2.0, 0.0, 2.0);                   // 4: x: f64 from the one fn impl
```

加上 `+ Send` / `+ Sync` 的約束或槽位，會如下文所述對擷取提出要求；錯誤訊息會指出該擷取以及應改寫成什麼。

## 4.4.3. 指派給變數

函式或閉包可以指派給具有明確函式型別的變數。

```nabi
let f: fn(i32) -> i32 = |x| x * 2;
let g: fn(str, i32) -> str = format;
```

函式與閉包都可以指派給函式型別的槽位。

## 4.4.4. 作為函式引數與回傳值

函式型別主要用作其他函式的引數與回傳值。

```nabi
fn apply(f: fn(i32) -> i32, x: i32) -> i32 { f(x) }
fn make_adder(n: i32) -> fn(i32) -> i32 { |x| x + n }
```

第一種模式是接受函式作為引數的高階函式；第二種把函式作為其結果回傳。這兩種模式是函式型別的核心用法。

## 4.4.5. 集合中的函式

和其他型別一樣，函式型別可以作為集合的元素型別。

```nabi
let handlers: List<fn(Event) -> ()> = [...];
let dispatch: Dict<str, fn(Request) -> Response> = {...};
```

把函式型別存放在集合中，就能把行為當作資料處理。

## 4.4.6. 帶有擷取的閉包

帶有擷取的閉包以相同的函式型別表達。對使用者而言，它們呈現相同的介面。

```nabi
let n = 10;
let f: fn(i32) -> i32 = |x| x + n;     // captures n, but the type is fn(i32) -> i32
```

擷取在內部附加於閉包物件上，並在呼叫時自動使用。因此使用者程式碼不需要在意是否有擷取，擷取也不會出現在型別中。

nabi 沒有 Rust 中 `Fn`、`FnMut` 與 `FnOnce` trait 的區分。擷取預設是依值的，這降低了語意的複雜度。詳細的擷取語意定義於[擷取語意](406-capture-semantics.md)。

### `Send` / `Sync` 約束

在函式型別後附加 `+ Send` / `+ Sync`，會在型別中記錄該值可以跨越執行緒邊界（[stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync執行緒邊界標記)）。

```nabi
fn(i32) -> i32 + Send
fn() + Send + Sync
```

- 當閉包字面值出現在 `+ Send` 位置時，每個一般擷取都必須是 `Send + Sync`：一般擷取是建立者會繼續使用的控制代碼，因此兩個執行緒都能存取它。`[move x]` 擷取只屬於該閉包（[4.6.11](406-capture-semantics.md#4611-移動擷取move-x-擷取清單)），因此只需要 `Send`。
- `+ Sync` 位置要求每個擷取都是 `Sync`，無論是否移動。`fn(..) + Send` 值不是 `Sync`，除非其型別這樣寫：它背後的閉包可能擁有一個不是 `Sync` 的移動擷取。會被多個執行緒同時呼叫的值（`static`、`par::*` 閉包）具有型別 `fn(..) + Send + Sync`。
- 產生執行緒 / 任務的函式（`task::spawn_blocking`、`thread::spawn`、`par::Scope::spawn`）接受 `+ Send` 閉包，因此被產生的程式碼不與其建立者共享任何非 `Sync` 的值；`[move x]` 則改為交給它一個值。`par::for_each`、`par::map` 與 `par::filter` 接受 `+ Send + Sync` 閉包，由每個工作者呼叫。
- 要使用 `fn` 型別的變數，該變數的型別必須已經具有相同的約束。`fn() -> T` 值不適合 `fn() -> T + Send` 位置。擷取了 `fn(..) + Send` 值的閉包，只有在把該值移入（`[move f]`），或該值的型別是 `+ Send + Sync` 時，才能跨越執行緒。
- 具名函式沒有擷取，因此一律是 `Send + Sync`。
- 約束是型別的一部分：分別以 `fn(i32) + Send + Sync` 值與 `fn(i32)` 值呼叫的泛型函式有兩個實例。`fn(i32)` 上的 impl 服務兩者。

```nabi
let c: Cell<i32> = Cell::new(0);
let xs: List<i32> = [1, 2];
let f: fn() + Send = || print("hi");             // OK: no captures
let g: fn() + Send = || c.set(1);                // ERROR: Cell is not Send
let h: fn() = || print("hi");
let k: fn() + Send = h;                          // ERROR: h's type has no + Send
let m: fn() + Send = || print(xs.len());         // ERROR: List is not Sync
let n: fn() + Send = [move xs] || print(xs.len());  // OK: the closure owns xs
```

## 4.4.7. 漸進式參數與回傳

這種記法無法直接表達函式型別中的泛型參數。泛型函式的值在使用處被單態化為具有具體型別的函式值。

```nabi
fn id<T>(x: T) -> T { x }

let f = id;                       // ERROR: id itself is generic; T is not determined
let f: fn(i32) -> i32 = id;       // OK: monomorphized to i32
let g = id::<i32>;                // OK: monomorphized with turbofish
```

要在函式型別中放入動態槽位，請為參數或回傳寫 `any`。

```nabi
fn(any) -> any
```

這是接受任何型別、也能回傳任何型別之函式的型別。它會產生動態分派成本。

## 4.4.8. 發散函式的型別

[回傳](403-return.md)中所涵蓋之發散函式的型別，在回傳型別位置寫 `!`（`fn() -> !`）。`fn(A) -> !` 值可以用在任何參數相同的 `fn(A) -> R` 位置，而會擲回錯誤的 `fn(A) throws (E) -> !` 適合 `fn(A) throws (E) -> R`。

```nabi
let f: fn() -> i32 = || panic("never");     // fn() -> ! value → fn() -> i32 position
```

### 函式值的表示方式

函式值是 `{code, env}` 組合：`code` 是程式碼指標，`env` 是擷取環境。對具名函式的值而言，`env` 是 null。`extern "C"` 宣告中函式型別的參數具有型別 `extern "C" fn(..)`，即沒有 `env` 的 C 函式指標，它與 Nabi 函式值之間不能互相轉換（[5.9.4](../005-modules/509-extern-block.md#c-函式指標型別)）。方法路徑（`S::new`、`i32::add`）也是函式值（[3.5.3](../003-expressions/305-member-access-and-method-call.md#方法路徑值)）。

## 4.4.9. 函式型別之間的子型別關係（無）

> [!IMPORTANT]
> Rust 或 TypeScript 這類語言中的函式型別子型別關係，在 nabi 中不存在。函式型別不會被自動轉換為更一般或更特定的函式型別。參數型別與回傳型別必須完全相符。

```nabi
let f: fn(i32 | f64) -> i32 = |x: i32| x;     // ERROR: parameter type does not match exactly
```

函式型別中的參數型別與回傳型別必須完全相符；它們不會被放寬為和型別。

唯一的例外是 4.4.8 的發散回傳以及錯誤通道。若參數型別與回傳型別相同，擲回較少錯誤的函式值可以用在預期能擲回較多錯誤之函式型別的位置。這種轉換（不擲回 → 擲回、較窄的錯誤 → 較寬的錯誤）可以由編譯器產生的配接器實作。

```nabi
fn parse_typed(s: str) throws (ParseError) -> i32 { ... }

let p: fn(str) throws (ParseError) -> i32 = |s| s.len() as i32;   // OK: non-throwing → throwing
let q: fn(str) throws -> i32 = parse_typed;   // OK: throws (ParseError) → throws (any Error)
let r: fn(str) -> i32 = parse_typed;          // ERROR: throwing → non-throwing
```

`throws (A)` 函式值可以用在 `throws (A | B)` 位置，而若 `A` 實作了 `Error`，也可以用在未定型的 `throws` 位置。

這種轉換適用於所有函式值，包括函式型別的變數與帶有擷取的閉包。編譯器建立一個包裝原始值的配接器 `{code, env}`。

```nabi
let g: fn(str) throws (ParseError) -> i32 = parse_typed;
let h: fn(str) throws -> i32 = g;             // OK: wrapped in an adapter
```

動機是明確性。Nabi 的方針是函式的轉換由使用者以[轉換](../002-types/214-cast-and-conversion.md)的明確形式撰寫。

## 4.4.10. 函式型別的求值

函式型別是不產生值的型別。它只是說明接受何種形狀之函式的規格，本身沒有求值行為。

呼叫函式遵循[成員存取與方法呼叫](../003-expressions/305-member-access-and-method-call.md)中所述的一般呼叫規則（`f(...)`）。

## 4.4.11. 沒有參數名稱

函式型別不含參數名稱。定義（`fn name(x: i32)`）有名稱，但型別記法（`fn(i32)`）只保留位置。

```nabi
fn(i32, i32) -> i32                // OK
fn(x: i32, y: i32) -> i32          // ERROR: function types have no names
```

呼叫時的具名引數（[參數](402-parameters.md)）指稱函式定義中的名稱。因此透過函式型別呼叫時，具名引數不起作用；透過值的呼叫只接受位置引數。

```nabi
fn make_user(name: str, age: i32) -> User { ... }
let f: fn(str, i32) -> User = make_user;

make_user(name: "A", age: 30i32);    // named OK: uses the definition's names
f("A", 30i32);                        // positional OK
f(name: "A", age: 30i32);             // ERROR: the type has no names
```
