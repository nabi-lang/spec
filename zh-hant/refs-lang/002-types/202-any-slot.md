# 2.2. `any` 槽位

`any` 是表示可以容納任何型別之值的變數槽位的型別名稱（它是上下文關鍵字，只在型別位置是關鍵字，[1.5.4](../001-lexical/105-keywords.md#154-上下文關鍵字)；宣告名為 `any` 的型別是錯誤）。它是 Nabi 漸進式型別的核心概念。

> [!IMPORTANT]
> `any` 槽位只存在於寫了 `any` 之處。省略型別標註的變數，會成為由其初始化式型別推論出的靜態槽位（[5.1.7](../005-modules/501-variable-declaration.md#517-型別標註與型別推論)）。Nabi 的漸進式型別由一行規則決定：「預設為靜態，只有寫了 `any` 之處才是動態。」

## 2.2.1. 與靜態槽位的對比

首先，靜態槽位與 `any` 槽位的差異如下：

```nabi
let x: i32 = 42i32;     // x is i32; cannot hold a value of another type
let y: any = 42i32;     // y is an any slot; any type allowed
let z = 42i32;          // z is i32; static slot inferred from the initializer
```

在這三行中，`x` 與 `z` 意義相同；`y` 是唯一的 `any` 槽位。省略型別標註時，編譯器把初始化式的型別當作槽位型別。這讓 REPL 與筆記本的使用者可以不寫標註而快速輸入，同時仍得到靜態分派，也讓會產生動態分派成本的地方在宣告語法中一目了然。

## 2.2.2. 槽位的記憶體表示

`any` 槽位中的值以一個標籤與一個酬載的組合表示。

```
+------+----------+
| tag  | payload  |
+------+----------+
```

標籤識別酬載是哪個型別；酬載就是值本身：小的值（基本型別等）直接存放，大的值則以對堆積物件的參照存放。

標籤與酬載確切的位元寬度、對齊與編碼定義於 [8. 執行環境 ABI](../008-abi/README.md)；本文件只保證其形狀（兩個部分：標籤 + 酬載）。

## 2.2.3. 動態分派

對 `any` 槽位中的值套用運算時，執行環境會檢查標籤並呼叫適當的 trait 方法。

```nabi
let v: any = 42i32;
let w = v + 1i32;           // at runtime, looks at v's tag and calls i32::add
```

這種分派的成本明顯高於靜態呼叫，因此效能關鍵的程式碼應當使用靜態槽位。

### `any` 上的運算子

對 `any` 運算元套用運算子時，規則如下：

- 候選是擁有該運算子的基本型別，而兩個運算元必須是相同型別。
- 若一個運算元具有靜態型別，該型別把候選固定為一個。若靜態運算元不是基本型別，就是編譯錯誤。
- 無後綴的字面值運算元不是靜態型別：當它能取另一個運算元的執行時標籤所表示的型別時，就取該型別（當 `v` 存放 `i32` 時，`v + 1` 中的 `1` 是 `i32`；存放 `f32` 時則是 `f32`）。若標籤是字面值無法取的型別（`300` 對上 `u8`、`0.5` 對上 `i32`），程式會 panic 並指出兩者（`` `+` on an `any` holding `u8`: `300` does not fit `u8` ``）。
- 若執行時運算元型別不相符，程式會 panic（結束碼 101）。
- 若每個候選的結果都是 `bool`，結果就是 `bool`；否則是裝箱的 `any`。

```nabi
let v: any = 42i32;
let w = v + 1i32;           // w: any (i32 + i32)
let b = v == 1i32;          // b: bool
let c = v + 1;              // 1 is i32 here: it takes the tag's type
let p = v + Point {};       // ERROR: Point is not a primitive type
```

## 2.2.4. 重新指派：可以改變型別

`any` 槽位可以自由地以不同型別的值重新指派。

```nabi
let s: any = 42i32;
s = "hello";        // OK: the slot is any
s = 3.14f64;        // OK
```

相對地，靜態槽位只允許以相同型別重新指派。

```nabi
let s: i32 = 42i32;
s = 10i32;          // OK
s = "hello";        // ERROR
```

## 2.2.5. 檢查槽位目前的型別

`any` 槽位中之值目前的型別可以用後綴運算子 `is` 檢查。`e is T` 的結果是 `bool`，其優先順序與 `as` 相同（[3.2.3](../003-expressions/302-precedence-and-associativity.md#323-優先順序與結合性表)）。

```nabi
if v is i32 { ... }
```

這個檢查是一次執行時標籤比較。在 `any` 上，當存放的值是 `A` 或 `B`，以及存放的是整個和型別 `A | B` 的值時，`is (A | B)` 與模式 `x: A | B` 都會匹配。繫結會被重新標記為和型別 `A | B`。若運算元的靜態型別既不是 `any` 也不是和型別，型別檢查 `e is T` 是一個編譯期常數。沒有被先前的使用確定型別的無後綴字面值，在那裡是其預設型別（[1.6.3](../001-lexical/106-numeric-literals.md#163-無後綴的字面值)）：在 `let v = 3;` 之後，`v is i64` 是 `true`。目標為和型別時，它測試的是成員關係：對於普通型別 `A` 的值，`e is (A | B)` 是 `true`。

對於靜態型別為 `enum` 的值，`e is Enum::Variant` 檢查變體。它是一次執行時的變體比較，對該變體而言無論酬載為何都是 `true`。它不會收窄型別；需要酬載時，請使用 `match` 或 `if let`。

```nabi
if shape is Shape::Circle { count += 1; }        // checks only whether it is Circle(_)
```

## 2.2.6. 型別收窄

在 `is` 檢查或模式比對成功的分支內，編譯器把槽位型別視為已收窄為所檢查的型別，因此在該分支內可以進行靜態分派。

```nabi
let v: any = ...;
if v is i32 {
    let w = v + 1i32;       // inside this branch v is i32: static dispatch
}
```

收窄規則如下：

- 收窄條件是 `x is T`，其中 `x` 必須是區域變數。它可以是整個條件，或是 `a && b && ..` 鏈中的一項。對於可選的 `x`（含有 `none` 的和型別），`x != none` 與 `!(x is none)` 把 `x` 收窄為 `none` 以外的成員。
- 在其後的 `&&` 項、`if` 的 `then` 區塊以及 `while` 的主體中，`x` 是 `T`。
- `x == none`（或 `x is none`）作為整個條件，或作為 `a || b || ..` 鏈中的一項時，會在 `else` 區塊中把 `x` 收窄為 `none` 以外的成員。
- 在型別除了其他成員之外還含有 `none` 的任何值上，`x == none`、`x != none`、`none == x` 與 `none != x` 都是 `none` 測試，與 `x is none` 相同：其他成員不需要 `PartialEq`，因此 `fn has<T>(v: T | none) -> bool { v != none }` 不需要對 `T` 加約束。在不含 `none` 的型別上，或在型別 `none` 本身上，這個比較是錯誤（E0054）。
- 在一個其中某個分支一定會離開（`return`、`break`、`continue`、`throw`、`!` 呼叫）的 `if` 陳述式之後，另一個分支的收窄在外層區塊的其餘部分持續有效：`if x == none { return; }` 讓其下方的 `x` 不是 `none`。
- 除此之外，在 `||` 或 `!` 之下、在 `else` 區塊中，或在欄位路徑上（`p.x is T`）都不會收窄。
- 變體檢查 `x is Enum::Variant`（2.2.5）不會收窄。
- 當運算元的靜態型別是所檢查和型別的普通成員 `A` 時，`x is (A | B)` 不會收窄；`x` 保持型別 `A`。
- 當 `x` 被 `[ref x]` 擷取時（[4.6.2](../004-functions/406-capture-semantics.md#462-參照擷取ref-x-擷取清單)），收窄作用域內每次讀取 `x` 都會再次檢查共享的儲存格；若某個閉包存入了不是 `T` 的值，該次讀取會 panic。
- 在收窄作用域內，`x` 的型別是 `T`。對 `x`（或其欄位 / 元素）的指派會被轉換回去並寫入原本的槽位；作為 `ref` 引數傳遞的 `x`（或其欄位）的最終值，在呼叫回傳時也是如此。指派一個屬於宣告型別但不是 `T` 的值，會在該作用域的其餘部分結束收窄：`x` 重新具有其宣告型別，並繼續一般的流程定型。指派宣告型別之外的值仍然是型別錯誤。

```nabi
let v: any = 1i32;
if v is i32 && v > 0i32 {      // v: i32 in the second term
    v = v + 1i32;              // written to the original any slot
    v = "x";                   // ends the narrowing: v is any again
}
if v is i32 || flag { ... }    // no narrowing

fn first(xs: List<i32>) -> i32 {
    let x = xs.pop();          // i32 | none
    if x == none { return 0; }
    x + 1                      // x: i32 for the rest of the block
}
```

關於以模式收窄，請見[模式](../003-expressions/311-patterns.md)中的型別檢查模式。

## 2.2.7. 漸進式型別的邊界

`any` 槽位只存在於寫了 `any` 之處。省略型別標註的意義依位置定義如下。

### 沒有型別標註的變數

```nabi
let n = 42i32;          // n: i32
let s = "hi";           // s: the string type chosen by the literal handler
let n2: any = 42i32;    // dynamic slot only when written explicitly
```

`let` 遵循區域型別推論。初始化式的型別成為槽位的型別，之後的重新指派都對照它檢查。無法從初始化式決定型別時的規則定義於 [5.1.7](../005-modules/501-variable-declaration.md#517-型別標註與型別推論)。

### 沒有型別標註的函式參數

```nabi
fn compute(x, y) { ... }
```

這是一個泛型函式。和變數一樣，它遵循「省略型別 = 由編譯器靜態決定」的規則。它依呼叫處的型別單態化：若某個呼叫者傳入兩個 `i32`，該呼叫就被編譯為接受兩個 `i32` 的函式；若另一個呼叫者傳入兩個 `f64`，就會編譯出另一個函式。

要讓參數成為動態槽位，必須明確寫出 `any`。

```nabi
fn compute(x: any, y: any) { ... }    // dynamic slots
```

### 結構欄位

結構欄位必須寫出型別。

```nabi
struct Point { x: f64; y: f64; }      // OK
struct Mixed { name; value; }          // ERROR: missing field types
```

要讓欄位成為動態的，同樣使用明確的 `any`。

```nabi
struct Cell { tag: i32; payload: any; }   // OK
```

這個要求讓編譯器能決定結構的記憶體配置。

## 2.2.8. 槽位種類由宣告固定

變數的槽位是動態還是靜態，光看宣告語法就能知道。寫出 `: any` 構成動態槽位；其他任何寫法（明確的或推論的型別）構成靜態槽位。一旦固定，槽位的種類絕不會改變。重新宣告該名稱的 `let` 會建立一個具有自己種類的新槽位（[5.1.4](../005-modules/501-variable-declaration.md#514-同一作用域中的重新宣告)）；舊的槽位保持其種類。

```nabi
let x = 1;              // static slot (inferred from initializer)
x = "hi";               // ERROR: x is a static slot
let x: any = 5i32;      // a new binding: a dynamic slot
x = "hi";               // OK
```

詳細內容定義於[變數宣告](../005-modules/501-variable-declaration.md)。

## 2.2.9. 對編譯器 / 工具的友善程度

含有 `any` 槽位的程式碼，型別檢查較弱。編譯器無法事先知道槽位的實際型別，因此工具（LSP、靜態分析）只能在這個限制內盡力而為。漸進式與靜態之間的選擇是使用者的明確決定，可從是否寫了 `any` 看出，而規格完整支援兩種模式。

## 2.2.10. 受約束的 `any`：`any Trait`

在 `any` 之後寫一個 trait，就構成只容納實作了該 trait 之型別的值的動態槽位。

```nabi
let e: any Error = ParseError::new(s, 0, "empty");
let shapes: List<any Shape> = [Circle { r: 1.0 }, Square { side: 2.0 }];
let item: any (Display + Hash) = 42i32;
```

共有三種形式：單獨的 `any`、帶一個 trait 的 `any Trait`，以及在括號中帶多個 trait 的 `any (A + B)`。單獨的 `any` 等同於沒有約束的 `any ()`（文法：[型別運算式](201-type-expressions.md)中的 `any_type`）。`any` 只繫結緊接在它之後的單一 trait，因此 `any Error | none` 是 `(any Error) | none`。

在型別位置寫出單獨的 trait 名稱仍然是禁止的（[trait 不能作為型別](212-trait-as-type-banned.md)）。`any Trait` 之所以被允許，是因為動態分派的成本由 `any` 關鍵字明顯地標示出來。

### 存入時的檢查

把值存入 `any Tr` 槽位時，編譯器在編譯期檢查該值的靜態型別是否實作了 `Tr`。`any (A + B)` 要求同時實作 `A` 與 `B`。

```nabi
let a: any Display = 42i32;          // OK: i32: Display
let b: any Display = Opaque {};      // ERROR: Opaque does not implement Display

let v: any = 42i32;
let c: any Display = v;              // ERROR: no implicit any → any Display conversion
if v is i32 {
    let d: any Display = v;          // OK: after narrowing, the static type is i32
}
```

`any` 型別之間的轉換只允許朝放寬的方向。

- 當 `B` 的每個約束都被 `A` 的某個約束涵蓋時，允許 `any A` → `any B`。涵蓋是指相同的 trait，或能沿著父 trait（經過多步）到達。`Tr<args>` 涵蓋 `Tr`。
- `any X` → 單獨的 `any` 一律允許。
- 其他任何方向（例如 `any` → `any Tr` 這種收窄的轉換）都是錯誤。這是為了避免隱藏執行時檢查的隱含轉換；請先以 `is` 檢查或型別模式取出具體型別，再存入。

```nabi
let e: any Error = ...;
let d: any Display = e;              // OK: Display is a supertrait of Error
let x: any = e;                      // OK
let e2: any Error = x;               // ERROR: narrowing conversion
```

### 使用

在 `any Tr` 值上，只能呼叫 `Tr` 及其父 trait 的方法，而呼叫在執行時依標籤分派。不過，若方法的簽章（不含 `self`）提到下列任何一項，就不能呼叫：

- `Self`
- 未被約束的引數固定的 trait 型別參數
- 未被約束的 `X = ..` 繫結固定的關聯型別

約束可以省略引數與繫結；省略者被視為不受約束。在 `any Tr<A, X = B>` 上的呼叫，會以代入了 `A` 與 `B` 的 trait 宣告簽章檢查。因此即使 `Tr` 只由泛型 impl 實作，也能呼叫。例如，`next` 不能在 `any Iterator` 上呼叫（`Item` 未固定），但可以在 `any Iterator<Item = i32>` 上呼叫。成本模型與 2.2.3 中的 `any` 相同。

分派涵蓋程式存入 `any` 槽位的型別。載入外掛的程式（[9.12](../009-tools/912-plugins.md)）也會接收只有外掛知道的型別的值：對於分派不認得的標籤，它會呼叫存入該值的模組所註冊的方法主體。不載入任何外掛的程式沒有這個步驟。

擁有自身型別參數的方法（`fn write<B: AsBytes>(ref self, buf: B) -> usize`）也可以呼叫。每次呼叫都和泛型呼叫一樣固定它們：從 `::<..>`，否則從引數型別，否則從結果的預期型別；沒有任何一項能固定的參數是錯誤，錯誤訊息會要求寫 `::<..>`。它們的約束在呼叫處檢查。每個用到的實例化都有自己針對實作者的分派，每個實作者呼叫的是針對這些型別單態化的方法。

```nabi
fn log(w: any Write) throws (IoError) {
    try w.write("line\n");                 // B = str
    try w.write([0u8, 1]);                  // B = [u8; 2]
}
```

要使用其他方法或欄位，請如 2.2.5-2.2.6 所述，以 `is` 檢查或型別模式收窄為具體型別。

```nabi
fn describe(e: any Error) -> str {
    let msg = f"{e}";                       // Display (supertrait of Error)
    match e {
        p: ParseError => f"{msg} at {p.pos}",   // narrowed to the concrete type
        _ => msg,
    }
}
```

### 表示方式

`any Tr` 值與 `any`（2.2.2）具有相同的標籤 + 酬載表示方式。trait 方法的分派表在連結時，從程式中存入 `any` 槽位的所有型別建出（封閉世界，假設為單一套件編譯）。透過泛型 impl 或 blanket impl 實作該 trait 的型別也包含在內。因此沒有任何值帶有自己的 vtable 指標。
