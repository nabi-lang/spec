# 3.10. `if` 運算式

`if` 與 `else` 構成依布林值分支的運算式。

## 3.10.1. 形式

```
if_expr     ::= "if" expr block ("else" (if_expr | block))?
              | "if" "let" pattern "=" expr block ("else" (if_expr | block))?
```

```nabi
if cond { ... }
if cond { ... } else { ... }
if cond1 { ... } else if cond2 { ... } else { ... }
```

這裡的 `cond` 是型別為 `bool` 的運算式，主體與 `else` 主體都是[區塊運算式](309-block-expressions.md)。`else if` 是 `else { if ... }` 的簡寫；它不是另外的語彙單元，而是自然的組合。

## 3.10.2. 作為運算式的值

`if` 只有在帶有 `else` 子句時，才會產生 unit 以外的值。

```nabi
let a = if cond { 1i32 } else { 2i32 };          // OK, i32
let b = if cond { 1i32 };                          // OK, but b has type unit (no else)
```

沒有 `else` 的 `if` 只有一個分支，因此不回傳任何值，其結果是 unit。帶有 `else` 的 `if` 把兩個分支的值合併為其結果。

### 分支結果型別

依[運算式概觀](README.md)的分支結果規則，若所有分支型別相同，該型別就是 `if` 運算式的結果型別；某個分支的和型別若容納另一個分支的型別，也是如此（在 `x: i32 | none` 之下，`if c { none } else { x }` 是 `i32 | none`）。若分支型別以其他方式不同，接收的一方必須有明確的型別，且每個分支都必須與之相容。和型別不會被自動推論。接收槽位的型別遵循[變數宣告](../005-modules/501-variable-declaration.md)的規則。

```nabi
// no type on the slot → inferred from the expression's result type
let x = if cond { 1i32 } else { 2i32 };          // expression result i32, slot x: i32
let y = if cond { 1i32 } else { 2.0f64 };        // ERROR: branch type mismatch; type annotation required

// sum type on the slot → member check
let w: i32 | f64 = if cond { 1i32 } else { 2.0f64 };   // OK

// single type on the slot → the expression's result must be that type
let z: i32 = if cond { 1i32 } else { 2.0f64 };   // ERROR: f64 branch does not fit i32
```

`else if` 鏈的每個分支（每個 `else if` 與最後的 `else`）都受到這項檢查。

## 3.10.3. 條件運算式的位置

```nabi
if cond { ... }
```

任何 `bool` 運算式都可以出現在條件位置 `cond`：呼叫、方法呼叫、比較、邏輯組合等。

```nabi
if user.is_admin() && !user.is_locked { ... }
if items.len() > 0 { ... }
```

非 `bool` 型別是編譯錯誤（E0040）。沒有把 0 或空字串視為 `false` 的隱含真值轉換。需要布林值之處，請寫出明確的轉換或比較。

```
if x { ... }              // ERROR if x: i32 (i32 is not bool)
if x != 0i32 { ... }      // OK
```

測試型別參數的條件（`if T is f32 { .. }`）在每個實例中是常數；其分支把 `T` 視為所測試的型別，而一個實例只降階它所走的分支（[2.10.13](../002-types/210-generics.md#21013-type-tests-on-type-parameters)）。

### 條件位置的結構字面值

在下列位置，結構字面值不能出現在運算式的頂層，因為它的 `{` 會被讀作主體的開頭。

- `if` / `while` / `if let` / `while let` 的條件
- `match` 的受檢值（[3.12](312-match-expression.md)）
- `for` 的迭代目標（[3.13](313-loop-expressions.md)）
- `using` 的資源運算式（[3.17](317-using-expression.md)）

在分隔符內（`( )`、`[ ]`、呼叫 / 方法引數、陣列、元組、索引、字典字面值、區塊 `{ }`）是允許的。要在頂層使用，請以括號包起來。

```nabi
if S { a: 1 }.ok() { .. }       // ERROR (E0013): the `{` after `S` is the body
if (S { a: 1 }).ok() { .. }     // OK
if check(S { a: 1 }) { .. }     // OK: inside an argument
```

## 3.10.4. 發散的分支

[區塊運算式](309-block-expressions.md)的發散區塊在分支結果型別檢查中與任何型別相容。

```nabi
let x: i32 = if cond {
    42i32
} else {
    return -1i32;          // divergent, OK even in an i32 position
};
```

這條規則讓提前離開能自然地表達。

## 3.10.5. `if let`：與模式比對結合

`if` 的條件位置也可以與模式比對結合。

```nabi
if let v = compute() {             // compute(): User | none; v: User
    consume(v);
} else {
    fallback();
}
```

```
if_let ::= "if" "let" pattern (":" type_expr)? "=" expr block ("else" (block | if_expr))?
```

- 沒有型別時，運算式的值與 `pattern` 比對。當值是可選的（含有 `none` 的和型別）且 `pattern` 是單純的名稱時，該名稱比對的是去掉 `none` 之後的值：`if let x = xs.pop()` 對 `T | none` 繫結 `x: T`，對 `A | B | none` 繫結 `x: A | B`。只會去掉一層：在 `(T | none)?`（可選呼叫的 `try?`）上，`x` 是 `T | none`。元組與結構模式本來就只比對其形狀的成員。
- 有 `: T` 時，值先被收窄為 `T`，再由 `pattern` 解構。此時 `pattern` 對 `T` 必須是不可反駁的（`if let (a, b): (i32, i32) = v { .. }`）。
- 比對成功時，執行主體，模式的繫結在其中可見。否則執行 `else`。

`if let` 的模式語法定義於[模式](311-patterns.md)。

## 3.10.6. 鏈式 `if let`（無）

不採用 Rust RFC 2497 風格、在一行上串接多個 `if let` 的寫法。

```
if let A(a) = x && let B(b) = y { ... }     // not adopted
```

替代方式是使用[模式](311-patterns.md)的元組模式，或巢狀的 `if let`。

## 3.10.7. `else` 主體形式的限制

`else` 之後只能接區塊（`{ ... }`）或另一個 `if` 運算式。不允許直接接一般運算式。

```
if a { 1 } else 2          // ERROR: block required after else
if a { 1 } else { 2 }      // OK
if a { 1 } else if b { 2 } else { 3 }    // OK, if after else
```

這項限制防止了有歧義的懸置 `else`。

## 3.10.8. 求值順序

`if` 的求值如下。

1. 對 `cond` 求值。
2. 若 `cond` 為 `true`，對 `then` 主體求值；若為 `false`，對 `else` 主體求值。

求值是短路的。一旦 `cond` 確定，就只對一側求值，另一側的副作用不會發生。`else if` 鏈由上到下檢查，第一個條件為 `true` 的分支會執行。
