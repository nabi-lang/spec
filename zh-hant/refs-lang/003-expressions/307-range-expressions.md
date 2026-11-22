# 3.7. 範圍運算式

範圍運算式使用 `..` 或 `..=` 語彙單元（[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)），產生一個有起點與終點的範圍值。

```
range_expr      ::= expr ".." expr        // half-open
                  | expr "..=" expr       // inclusive
                  | ".." expr             // from start
                  | "..=" expr            // from start, inclusive
                  | expr ".."             // unbounded end
                  | ".."                  // full
```

## 3.7.1. 六種形式

範圍運算式可以寫成六種形式。

```nabi
0..10           // [0, 1, ..., 9]      exclusive end
0..=10          // [0, 1, ..., 10]     inclusive end
..10            // same as 0..10       start omitted (prefix)
..=10           // same as 0..=10      start omitted, inclusive end
10..            // [10, 11, ...]       end omitted, infinite
..              // full                both omitted, mainly for slicing
```

`..` 產生不含終點的半開區間；`..=` 包含終點。省略起點表示 0 或依資料而定的預設值；省略終點表示無限或資料的末端。

## 3.7.2. 去糖

各形式去糖為下列標準函式庫型別。

```
a .. b          →   Range { start: a, end: b }
a ..= b         →   RangeInclusive { start: a, end: b }
.. b            →   RangeTo { end: b }
..= b           →   RangeToInclusive { end: b }
a ..            →   RangeFrom { start: a }
..              →   RangeFull
```

這些型別位於 `core::range`，由該模組末端的 `#[literal]` 處理器建立。這個轉換所使用的確切標準函式庫 trait 與型別，定義於函式庫規格。本文件只負責語言把六種形式辨識為不同的運算式，並把它們去糖為標準函式庫型別的實例這一事實。

## 3.7.3. 優先順序

在[優先順序與結合性](302-precedence-and-associativity.md)的等級中，範圍運算子（`..`、`..=`）是 prec=0 的不結合運算子：比包括空值合併（`??`，prec=1）在內的每個二元運算子都低，只比指派高。

```
0..n + 1            →   0 .. (n + 1)        // + binds tighter than ..
a ?? 0..3           →   (a ?? 0) .. 3       // ?? binds tighter than ..
a..b..c             // ERROR: range operators are non-associative
```

由於它們在二元運算子中結合最弱，起點與終點中的算術運算式讀起來很自然。

### 呼叫引數中的 `..x`

在呼叫引數位置，`..x` 不是範圍，而是展開（[4.2.6](../004-functions/402-parameters.md#426-展開args)）。因此 `f(..5)` 是展開 `5` 的呼叫；要傳遞 `RangeTo`，請以括號包起來（`f((..5))`）。`..=x` 不是展開，因此即使在引數位置也是範圍（`f(..=5)` 傳遞 `RangeToInclusive`）。

## 3.7.4. 用途

### 迭代

範圍值主要用於迴圈。

```nabi
for i in 0..10 { ... }
```

這裡[迴圈運算式](313-loop-expressions.md)的 `for` 迴圈接收範圍值並迭代它。`Range` 與 `RangeInclusive` 實作了 `Iterator` trait（[`trait`](../002-types/208-trait.md)）時會自動迭代。

### 與切片的區別

切片與範圍運算式是不同的記法。

```
arr[1:3]            // a slice of [indexing and slicing](306-indexing-and-slicing.md), uses a colon (`:`)
0..3                // a range value, uses two dots (`..`)
```

兩者是意義不同的不同語彙單元。切片是索引運算式的內部形式；範圍則是一個值（Range 型別的實例）。

### 模式比對

範圍模式（[模式](311-patterns.md)）使用相同的語彙單元。其意義定義於該處。

```
match x {
    1..=5 => ...,
    _ => ...,
}
```

## 3.7.5. 步長是方法

步長不屬於記法。它改由標準函式庫方法 `.step_by(n)` 提供。

```nabi
(0..100).step_by(2)         // [0, 2, 4, ..., 98]
```

這與 Nabi 的整體方向一致：保持語言語法簡單，額外功能以方法提供。切片的 `[::2]` 是另一種記法，之所以引入，是因為在那裡步長很自然。

## 3.7.6. 反向

要反向迭代，請使用 `.rev()` 方法。

```nabi
(0..10).rev()
```

範圍記法本身沒有負的步長。

## 3.7.7. 空範圍

若起點大於終點，範圍就是空的。

```nabi
5..3            // empty range, valid expression, iterates 0 times
5..=3           // empty range
```

這不是編譯錯誤，而是迭代零次的有效值。

## 3.7.8. 使用無限範圍

`for` 迴圈可以走訪 `0..` 這樣的開放範圍，在迴圈內以 `break` 結束。若值超過型別的最大值，就和算術溢位一樣處理（啟用檢查時會 panic）。

```nabi
for i in 0.. {
    if condition(i) { break; }
}
```

無限迭代本身等價於 [`loop { ... }`](313-loop-expressions.md)；唯一的差別在於方便。
