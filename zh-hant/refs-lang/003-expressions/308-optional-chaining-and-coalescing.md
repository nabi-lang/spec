# 3.8. 可選鏈結與空值合併

為了簡潔地表達對[和型別](../002-types/206-sum-types.md)之 `T | none` 值的鏈結與預設處理，定義了兩種記法。

```
optional_chain  ::= primary "?." identifier                       // optional field
                  | primary "?." int_literal                      // optional tuple field
                  | primary "?." identifier "(" arg_list? ")"     // optional method
coalesce_expr   ::= expr "??" expr
```

## 3.8.1. 可選鏈結：`?.`

```nabi
x?.field
x?.method()
```

`?.` 語彙單元定義於[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)。當左側 `x` 的型別為 `T | none` 且存在值時，就進行點存取或方法呼叫；沒有值（`none`）時，整個結果是 `none`。

### 去糖

```
x?.field            →   match x { v: T => v.field, none => none }
x?.method(args)     →   match x { v: T => v.method(args), none => none }
```

這裡的 `T` 表示左側和型別中 `none` 以外的變體。型別測試模式（`v: T`）取出並收窄和型別中非 none 的分支，然後進行點存取或方法呼叫。

### 非可選的接收者

若左側型別不包含 `none`，`?.` 是編譯錯誤。請改用 `.`。

```nabi
let p: Point = ...;
p?.x;          // ERROR: p is not optional; use .
```

## 3.8.2. 鏈結

`?.` 可以鏈結。

```nabi
let name = user?.profile?.name;
```

若任何一步得到 `none`，整個運算式就在那一點以 `none` 結束。若 `user` 是 `none`，整個結果是 `none`；若 `user.profile` 是 `none`，整個結果也是 `none`；只有在每個值都存在時，才會產生 `name` 的值。

## 3.8.3. 結果型別

`?.` 的結果型別傳播到鏈的末端。即使最後的運算式是 `T`，鏈的結果型別也一律是 `T | none`。

```nabi
let user: User | none = ...;
let name = user?.profile?.name;
// name: str | none
```

最後一步的攤平型別（`str`）不會被自動提升為和型別（`str | none`）；是鏈本身產生結果型別 `T | none`。

## 3.8.4. 空值合併：`??`

```nabi
x ?? y
```

當左側 `x` 是 `T | none` 時，`??` 在值存在時回傳該值，否則回傳 `y`。

### 去糖

```
x ?? y          →   match x { v: T => v, none => y }
```

### 結果型別

`??` 的結果型別遵循[運算式概觀](README.md#branch-result-types)的分支結果規則。若兩個分支（`T` 與 `typeof(y)`）型別相同，該型別就是結果型別。若不同，接收的一方必須有型別；不會自動放寬為和型別。

```nabi
let a = foo() ?? 0;                           // i32 (foo(): i32 | none)
let r = foo() ?? "default";                   // ERROR: i32 and str, no receiving type
let r2: i32 | str = foo() ?? "default";       // OK
```

若 `y` 也是 `T | none`，結果是 `T | none`（`T | (T | none)` 的聯集）。因此在鏈結 `??` 時，只有當最後一個替代值不是 `none` 時，`none` 才會消失。

```nabi
let a: i32 | none = ...;
let b: i32 | none = ...;
let r = a ?? b;              // r: i32 | none
let s = a ?? b ?? 0i32;      // s: i32
```

## 3.8.5. 優先順序

在[優先順序與結合性](302-precedence-and-associativity.md)的等級中，`?.` 與成員存取（`.`）具有相同的優先順序，結合非常緊。`??`（prec=1）位於邏輯 OR（`||`）之下、範圍運算子（`..`，prec=0）之上，並且是右結合的。

```
a ?? b ?? c     →   a ?? (b ?? c)              // right-associative
foo() ?? bar    →   (foo()) ?? (bar)           // a call binds tighter than ??
a + b ?? c      →   (a + b) ?? c               // + binds tighter than ??
a ?? 0..3       →   (a ?? 0)..3                // ?? binds tighter than ..
```

## 3.8.6. 結合鏈結與合併

```nabi
let name = user?.profile?.name ?? "anonymous";
```

這裡的 `user?.profile?.name` 得到 `str | none`，其後的 `?? "anonymous"` 填補 `none` 的情況，因此最終結果是 `str`。這是在套用預設值的同時，安全地取出可能不存在之值的標準形式。

## 3.8.7. 區分 `?.` 與 `?`

`?.` 是可選鏈結；`T | none` 值上的後綴 `?` 定義於 [3.14.10](314-error-control-flow.md)。

`?.` 與 `?` `.` 是不同的語彙單元，這由[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)的最長匹配規則保證。

## 3.8.8. 可選鏈結的短路求值

```
x?.foo().bar()
```

在這個運算式中，若 `x` 是 `none`，`foo()` 與 `bar()` 都不會被呼叫。短路求值是有保證的。

這是因為去糖後的 `match` 在 `none` 分支中立即回傳 `none`。

因此，當鏈中放了帶有副作用的方法時，接收者為 `none` 時它不會被呼叫。

鏈是 `?.` 之後的一串後綴運算：欄位存取、方法呼叫與索引（`x?.a[i]`）。`?.` 之後不接 `[`：沒有 `x?.[i]`；請寫 `x?.get(i)`。括號會結束鏈：在 `(x?.a).b` 中，`.b` 套用於 `x?.a` 的結果，而它可能是 `none`。

## 3.8.9. 深層點存取的替代方式

當鏈結變得又深又複雜時，把它與 `??` 結合，或使用明確的比對（[`match` 運算式](312-match-expression.md)）可能會更清楚。

```nabi
// chaining
let x = a?.b?.c?.d ?? default;

// explicit (for a: A | none, b: B | none, c: C | none)
let x = match a {
    av: A => match av.b {
        bv: B => match bv.c {
            cv: C => cv.d,
            none => default,
        },
        none => default,
    },
    none => default,
};
```

兩種形式在語意上完全相同；依可讀性選擇。
