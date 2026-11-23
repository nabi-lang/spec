# 3.9. 區塊運算式

以 `{ ... }` 括起來的一組陳述式與運算式，是 Nabi 中最基本的運算式單位。本文件定義區塊的語彙形式及其值的求值規則。

## 3.9.1. 形式

區塊的文法如下：

```
block       ::= label? "{" stmt* tail_expr? "}"
label       ::= label_token ":"              // label_token: 1.2
stmt        ::= expr ";"
              | block_like_expr ";"?
              | let_stmt
              | item_decl
item_decl   ::= fn_decl | const_decl | use_stmt | struct_decl | enum_decl
              | union_decl | trait_decl | impl_block | type_alias | static_decl
type_alias  ::= "type" identifier generics? where_clause? "=" type_expr ";"
block_like_expr ::= block | if_expr | match_expr
              | loop_expr       // loop, while, for (3.13)
              | unsafe_block    // 4.15
tail_expr   ::= expr            // no trailing ";"
```

區塊可以宣告 `fn`（[4.10](../004-functions/410-nested-functions.md)）、`const`（[5.2.2](../005-modules/502-const.md#522-允許的位置)）、`use`、`struct`、`enum`、`union`、`trait`、`impl`、`type` 與 `static` 項目。`mod` 與 `extern` 項目只能在模組層級宣告（[5.4.10](../005-modules/504-modules.md#5410-模組宣告可以出現的位置)、[5.9.8](../005-modules/509-extern-block.md#598-extern-區塊本身的位置)）；在區塊中宣告它們是編譯錯誤（E0017）。

在區塊中宣告的項目遵循下列規則：

- **作用域。** 項目的名稱在整個區塊中可見，無論在其宣告之前或之後，也在該區塊中宣告的項目內部可見。在區塊內，它會隱藏在區塊外宣告的同名、同命名空間項目；內層區塊中的 `use` 又會隱藏它。在區塊之外無法指稱它。同一個區塊中同名、同命名空間的兩個項目，或一個項目與一個同名的 `use`，是編譯錯誤（E0026 與 E0032，與模組層級相同）。
- **它看得到什麼。** 區塊中的項目看得到其模組的項目，以及其區塊和外圍區塊的項目與 `use` 宣告。它不能使用周圍程式碼的區域變數或參數（`let`、函式或閉包參數、模式繫結）或泛型參數；那樣做是編譯錯誤（E0039）。值透過欄位與參數傳給它；型別透過它自己的泛型參數傳給它。區塊中的 glob `use p::*`，在那裡宣告之項目的標頭（欄位型別、簽章、`const` / `static` 初始化式、別名、變體）與其主體中都看得到，優先順序與函式主體中相同：模組自己的項目勝過 glob，內層區塊的 glob 勝過外層的，而泛型參數勝過兩者。
- **一個項目。** 區塊中的項目無論其區塊執行多少次，都是一個項目：泛型函式中的項目不會隨該函式的參數成為泛型；區塊中的 `static` 是一個槽位，只初始化一次；區塊中的 `const` 只有一個值。
- **一致性。** 區塊中的 `impl` 與同一模組的模組層級 `impl` 一樣參與一致性與孤兒規則（[2.9.5](../002-types/209-impl.md#295-孤兒規則)）；其方法與 trait 實作在任何地方都成立。它可以為外部型別實作外部 trait。
- **可見性。** 區塊中的項目沒有自己的可見性：在其上寫 `pub` 是編譯錯誤（E0016）。在區塊中宣告之型別的欄位與方法，在其模組內遵循一般規則。
- **名稱。** `type_name`、`Debug` 與偵錯資訊以寫出的名稱指稱區塊中的項目（`S`）；診斷也是如此，而當有另一個項目同名時，會加上宣告它的函式（`f::S`）。

```nabi
struct S { a: i32; }

fn f() -> i32 {
    use core::fmt::Debug;
    const SCALE: i32 = 2;
    #[derive(Debug)]
    struct S { x: i32; }               // hides the module's `S` in this block
    impl S { fn scaled(self) -> i32 { self.x * SCALE } }   // OK: `SCALE` is an item
    S { x: 1 }.scaled()
}

fn g(n: i32) {
    struct P { v: i32 = n; }           // ERROR: `n` is a local of `g`
}
```

示意如下：

```nabi
{
    statement1;
    statement2;
    ...
    last_expr     // no terminator, becomes the block's value
}
```

主體可以包含零或多個陳述式，以及零或一個最後的運算式。若最後的運算式不帶終止符，它的值就是區塊的值；若帶有終止符（`expr;`），區塊的值是 unit。

```nabi
let a = {
    let x = 1;
    let y = 2;
    x + y               // block value = 3
};

let b = {
    let x = 1;
    x + 2;              // terminator present → result discarded
                        // block value = unit
};

let c = {
    let x = 1;
    return x;           // return exits the function itself (block value never reached)
};
```

## 3.9.2. 用途

區塊可以出現在任何可以出現運算式之處。

```nabi
let v = { ... };                  // variable initializer
fn_call({ ... }, other);          // function argument
if { ... > 0 } { ... }            // condition (rare but allowed)
```

其他控制結構（`if`、`match`、`loop`、函式主體）的主體也是區塊。

## 3.9.3. 作用域

區塊建立一個新的作用域。在區塊內以 `let` 宣告的變數只在該區塊內可見，離開區塊時被捨棄。外層的變數在區塊內仍然可見，除非被內層同名的 `let` 遮蔽。

內層同名的 `let` 是另一個繫結，不影響外層的繫結。確切規則定義於[變數宣告](../005-modules/501-variable-declaration.md)。

## 3.9.4. 空區塊

```nabi
{}
```

空區塊的值是 unit。

## 3.9.5. 區塊中的最後一個運算式與終止符

下列兩個輸入的求值結果不同。

```nabi
let a = { x };          // a is the value of x
let b = { x; };         // b is unit
```

不帶終止符結束的運算式成為區塊的值，而帶有終止符結束的運算式其結果會被捨棄，區塊的值是 unit。這種區分與[陳述式終止符](../001-lexical/103-statement-terminator.md)的規則一致。

- 區塊中間不帶 `;` 的類區塊陳述式必須具有型別 `()`。
- `}` 之前的最後一個陳述式可以省略其 `;`，意義與寫出相同（[1.3.4](../001-lexical/103-statement-terminator.md#134-類區塊運算式陳述式)）：`{ x += 1 }` 與 `{ let y = x }` 的值是 unit，與 `{ x += 1; }` 和 `{ let y = x; }` 相同。
- `;;` 是空陳述式。
- 類區塊運算式之後的成員存取（`.`）與 `?` 會延續運算式（`match v { .. }.len();`）。在陳述式位置，其 `}` 之後的 `(` 或 `[` 會開始一個新的運算式（[1.3.4](../001-lexical/103-statement-terminator.md#134-類區塊運算式陳述式)）；在較大的運算式內部，例如 `let f = if c { g } else { h }(x);`，它是呼叫或索引。

### 等價形式：最後一個運算式與明確的 `return` / `break`

不帶終止符的最後一個運算式，等價於適合其上下文的明確離開陳述式。

```nabi
// function body: last expression == return value
fn f() -> i32 { 42 }
fn g() -> i32 { return 42; }      // same meaning

// labeled block: last expression == break 'label value
let x = 'b: { 42 };
let y = 'b: { break 'b 42; };     // same meaning

// loop: a loop without break is divergent. There is no last-expression position.
```

最後一個運算式的位置，就是值浮現出來的地方。在函式主體中，它表示函式的回傳；在帶標籤的區塊中，它表示 break 到該標籤。無論寫成哪種形式，編譯器都產生相同的結果。

## 3.9.6. 與函式主體的關係

函式主體也是區塊，其最後一個運算式的值就是函式的回傳值。因此 `return` 關鍵字並非必要。

```nabi
fn add(x: i32, y: i32) -> i32 {
    x + y               // no terminator, returned automatically
}

fn other(x: i32) -> i32 {
    let r = x + 1;
    return r;           // explicit return also allowed, same meaning as above
}

fn early(x: i32) -> i32 {
    if x < 0 { return 0; }   // explicit return for early exit
    x * 2                    // last expression of normal flow, returned automatically
}
```

這三個函式都能運作。使用哪種形式是風格問題；規格完整支援兩者。

### 建議的用法

使用最後一個運算式（不寫 `return`）適合一般流程的單一回傳點。明確的 `return` 適合提前離開、從函式中間回傳，以及為了可讀性而標示。

函式主體、`return` 關鍵字與回傳型別規則詳細定義於 [4. 函式](../004-functions/README.md)。

## 3.9.7. 區塊的型別

區塊的型別是其最後一個運算式的型別（若沒有最後一個運算式則是 unit）。

```
{ 1i32 }                // i32
{ "hi" }                 // str
{ 1i32; }                // unit (expression ends with a terminator → result discarded)
{ }                      // unit (empty block)
```

這種型別判定與[運算式概觀](README.md)的分支結果型別規則結合。若同一位置的多個區塊運算式產生不同的型別，接收的一方必須有明確的型別。

## 3.9.8. 發散的區塊

有些區塊因為 `return`、`break`、`continue`、`throw`、`loop { /* no break */ }` 等而不產生值。

```nabi
let x = {
    return 0;       // exits the function
};
```

這樣的區塊被歸類為發散的（永不完成）。發散區塊的型別與任何型別相容，並通過分支結果型別檢查。

```nabi
let x: i32 = if cond {
    42i32
} else {
    return -1i32;       // divergent, does not affect branch agreement
};
```

偵測發散的確切規則定義於函式與控制流程規格。

## 3.9.9. 區塊的求值順序

區塊中的陳述式與最後一個運算式依寫出的順序（由上到下）求值，副作用也遵循該順序。

若在求值期間發生 `return`、`break`、`continue`、擲回的錯誤等，區塊的求值就在那一點停止。

## 3.9.10. 帶標籤的區塊：`'label: { ... }`

帶標籤的區塊可以用在任何可以出現區塊運算式之處：`let` 初始化式、函式主體、`if` / `else` 分支、`match` 分支主體、另一個區塊的結尾運算式等。沒有額外的位置限制；標籤只指定該區塊的離開點。

在區塊前加上 `'name:` 形式的標籤，就能在區塊內以 `break 'name value;` 提前離開，同時設定區塊的值。

```nabi
let result = 'b: {
    if early_check_fails() { break 'b -1; }
    if shortcut_taken()    { break 'b  0; }
    let x = compute();
    let y = refine(x);
    x + y                    // last expression of normal flow: the value of 'b
};
```

標籤語彙單元的形式（`'identifier:`）與 [3.13.4 標籤](313-loop-expressions.md#3134-標籤控制巢狀迴圈)中的相同。`break 'name value` 以給出的值離開該帶標籤的區塊。若 `continue 'name` 的 `'name` 指向一個**區塊**標籤，就是編譯錯誤（E0059），因為 `continue` 只適用於迴圈標籤。此外，不帶標籤的 `break` / `continue` 不能以區塊為目標（它們只以最內層的**迴圈**為目標）。

與其為了提前離開而寫一個小的輔助函式，帶標籤的區塊能乾淨地表達帶著值提前離開區塊。

## 3.9.11. `unsafe { ... }` 區塊

在區塊之前放上 `unsafe` 關鍵字，就使它成為進入不安全作用域的區塊。與一般區塊相同，最後一個運算式的值就是區塊的值，而且可以進行超出安全抽象保證的操作（原始指標解參照、呼叫 `unsafe fn` 等）。

```nabi
let n: usize = unsafe { strlen(cs.as_ptr()) };
```

詳細的意義與責任定義於 [4.15 unsafe](../004-functions/415-unsafe.md)。

## 3.9.12. 作用域結束時的釋放順序

大多數值不需要釋放：GC 會回收它們，沒有確定的順序，也不會執行任何程式碼。型別具有 `Drop` 的值（[stdlib 2.13.4](../../refs-standard-libraries/200-core/213-marker.md#2134-drop釋放時的清理)）會在確定的時間點被釋放。下列規則收集自定義它們的各節：

1. 值在其擁有者離開作用域之處被釋放，任何離開方式皆然：正常完成、`return`、`break` / `continue`、`?`、`throw` 以及錯誤傳播。移動到新擁有者（呼叫引數、欄位、通道）的值，改由該擁有者釋放。
2. 在同一個作用域內，擁有者依宣告的相反順序離開（最後宣告者最先釋放）；內層區塊的擁有者先於外層區塊的離開。
3. `using` 在其區塊結束時釋放其繫結，最後一個繫結最先（[3.17.4](317-using-expression.md#3174-多個資源)）。
4. 釋放一個值會先執行它自己的 `drop`，然後依宣告的相反順序執行其欄位的 drop glue、依相反順序執行元組元素的 drop glue，以及列舉或和型別目前所容納之酬載的 drop glue。
5. 重新宣告名稱的 `let`（[5.1.4](../005-modules/501-variable-declaration.md#514-同一作用域中的重新宣告)）不釋放任何東西：舊的值在舊繫結本應離開作用域之處被釋放。
6. 關閉可能失敗的值（`io::Close`）絕不會被自動關閉：讓它在開啟狀態下離開其作用域是編譯錯誤，沒有任何擁有者的該型別暫存值也是（E0134）；請以 `using` 繫結它。
7. panic 會結束行程，不會執行任何 `drop`。
