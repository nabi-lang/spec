# 3.12. `match` 運算式

`match` 是把一個值與多個模式比較，並對第一個比對成功的分支求值的運算式。

## 3.12.1. 形式

```
match_expr  ::= "match" expr "{" match_arm ("," match_arm)* ","? "}"
match_arm   ::= pattern ("if" expr)? "=>" expr
```

```nabi
match expr {
    pattern1 => expr1,
    pattern2 => expr2,
    ...
    patternN => exprN,
}
```

`match` 運算式由 `match` 關鍵字、受檢值以及 `{ ... }` 主體組成。結構字面值不能出現在受檢值的頂層（[3.10.3](310-if-expression.md#條件位置的結構字面值)）。主體包含零或多個以逗號分隔的分支；每個分支由一個模式、`=>` 語彙單元（[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)）與一個運算式組成。允許尾隨逗號。

## 3.12.2. 分支主體

```nabi
match v {
    1 => "one",
    2 => "two",
    _ => "other",
}
```

`=>` 之後接單一運算式。若該運算式是[區塊運算式](309-block-expressions.md)，它可以包含多個陳述式。

```nabi
match v {                        // v: i32 | none
    n: i32 => {
        let x = n * 2;
        x + 1
    },
    none => 0,
}
```

區塊分支中最後一個運算式的值就是該分支的結果。

## 3.12.3. 比對順序

分支由上到下檢查，第一個比對成功的分支會被求值。該分支運算式的值就是整個 `match` 運算式的結果。

## 3.12.4. 分支結果型別

依[運算式概觀](README.md)的分支結果規則，若所有分支結果運算式的型別相同，該型別就是 `match` 運算式的結果型別；某個分支的和型別若容納其他每個分支的型別，無論分支順序為何，也是如此。若分支型別以其他方式不同，接收的一方必須有明確的型別；和型別不會被自動推論。接收槽位的型別遵循[變數宣告](../005-modules/501-variable-declaration.md)的規則。

```nabi
// no type on the slot → inferred from the expression's result type
let x = match v {
    1 => "one",
    _ => "other",
};                      // expression result str, slot x: str

let y = match v {
    1 => 1i32,
    _ => "other",
};                      // ERROR (E0041): arm type mismatch; type annotation required

// sum type on the slot → member check
let w: i32 | str = match v {
    1 => 1i32,
    _ => "other",
};                      // OK

// single type on the slot → the expression's result must be that type
let z: i32 = match v {
    1 => 1i32,
    _ => "other",
};                      // ERROR (E0040): str arm does not fit i32
```

## 3.12.5. 窮盡性檢查

分支模式的集合必須涵蓋受檢值可能具有的每一個值；否則就是編譯錯誤。

```nabi
let x: bool = ...;
match x {
    true => 1,
    // ERROR (E0098): false is missing
}
```

規則如下。涵蓋封閉和型別（`A | B`）的每個成員是窮盡的。對 `enum` 而言，涵蓋每個變體是窮盡的。`bool` 在同時涵蓋 `true` 與 `false` 時是窮盡的。串列、切片或陣列在切片模式涵蓋每個長度時是窮盡的（3.11.1）。整數、`char`、字串與浮點數的字面值與範圍模式被視為開放集合。即使範圍涵蓋了該型別的每個值（`0..=255` 涵蓋 `u8` 的全部），也不被視為窮盡，因此需要萬用字元（`_`）或識別字模式（`x`）之類的全收分支。非窮盡的錯誤會列出遺漏的值（`` `false` not covered ``），並建議為它們加上分支。

對 [`any` 槽位](../002-types/202-any-slot.md)之動態槽位的比對一律需要全收分支，因為編譯器無法知道會來什麼型別。

## 3.12.6. 無法到達的分支

依比對順序永遠無法到達的分支是編譯錯誤（而非警告）。整數與 `char` 的字面值 / 範圍模式以區間比較，因此已被前面分支的區間涵蓋的範圍也是無法到達的（`0..=9` 之後的 `3..=5`）。範圍不會建立窮盡性（3.12.5）。

```nabi
match v {
    _ => 0,
    1 => 1,         // ERROR (E0099): the _ above always matches
}
```

## 3.12.7. 或模式

[模式](311-patterns.md)的或模式（`|`）把共用同一個主體的多個模式組在一起。

```nabi
match v {
    1 | 2 | 3 => "small",
    4 | 5 | 6 => "medium",
    _         => "large",
}
```

## 3.12.8. 守衛

分支可以附加[模式](311-patterns.md)的守衛（`if condition`）。

```nabi
match (x, y) {
    (a, b) if a == b => "equal",
    (a, b) if a > b  => "first bigger",
    _                => "second bigger",
}
```

帶有守衛的分支在窮盡性檢查中不算作全收分支，因為守衛可能為 `false`。因此只靠帶守衛的分支不能保證窮盡性。

```nabi
let x: bool = ...;
match x {
    a if a == true  => 1,
    a if a == false => 2,
    // ERROR (E0098): guarded arms alone do not guarantee exhaustiveness; _ required
}
```

## 3.12.9. 依比對結果收窄型別

在由型別模式或 `enum` 模式比對成功的分支內，編譯器會收窄受檢值的型別。

```nabi
let v: i32 | str = ...;
match v {
    n: i32 => use_int(n),       // in this arm n: i32, static dispatch possible
    s: str => use_str(s),
}
```

這種收窄以與 [`any` 槽位](../002-types/202-any-slot.md)和[和型別](../002-types/206-sum-types.md)之型別收窄相同的機制運作。

## 3.12.10. 求值順序

`match` 依下列順序求值：

1. 對受檢值求值（一次）。
2. 由上到下檢查分支。
3. 對第一個比對成功之分支的運算式求值。

受檢值的任何副作用都恰好發生一次。帶守衛之分支的守衛，只有在到達該分支時才會求值。

## 3.12.11. match 運算式本身的結果

`match` 運算式本身的結果遵循上述的分支結果規則（若所有分支型別相同，就是該型別，或容納每個分支型別的某個分支之和型別；否則接收的一方必須有明確的型別）。

若受檢值是發散的（例如 `match return 0 { ... }`），`match` 也是發散的。

## 3.12.12. 發散的分支

若分支的運算式是[區塊運算式](309-block-expressions.md)的發散形式（`return`、`break`、`throw` 等），該分支在分支結果型別檢查中被視為與任何型別相容。

```nabi
let x: i32 = match v {           // v: i32 | none
    n: i32  => n,
    none    => return 0,         // divergent, OK in an i32 position
};
```
