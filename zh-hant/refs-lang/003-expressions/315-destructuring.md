# 3.15. 解構指派

在 `let` 的左側寫出[模式](311-patterns.md)中不可反駁的模式，就能把一個運算式的結果拆開放進多個變數。

## 3.15.1. 形式

解構指派的文法如下：

```
destructure_let ::= "let" irrefutable_pattern (":" type_expr)? "=" expr ";"
```

```nabi
let pattern = expr;
```

這種形式把運算式的結果與模式比對，並一次宣告模式中所有的繫結變數。

## 3.15.2. 元組解構

最基本的形式，元組解構，用法如下。

```nabi
let (a, b) = (1, 2);                    // a = 1, b = 2
let (q, r) = divmod(10, 3);             // receive two values at once
let (q, _) = divmod(10, 3);             // discard the second
```

元素數量必須完全相符；否則就是編譯錯誤。

```nabi
let (a, b) = (1, 2, 3);                 // ERROR (E0101): length mismatch
```

## 3.15.3. 剩餘模式

要一起接收剩下的元素，請使用帶 `..` 的剩餘模式。

```nabi
let (first, ..rest) = (1, 2, 3, 4);     // rest = (2, 3, 4)
let (..init, last) = (1, 2, 3, 4);      // init = (1, 2, 3), last = 4
let (head, ..mid, tail) = (1, 2, 3, 4, 5);  // mid = (2, 3, 4)
let (..) = (1, 2, 3);                    // discard everything
```

`..` 在一個模式中最多出現一次，用來把剩餘部分打包成一個變數，或捨棄它。

## 3.15.4. 巢狀解構

解構模式可以巢狀到任意深度，因此複雜的結構可以一次拆開。

```nabi
let ((a, b), c) = ((1, 2), 3);          // a = 1, b = 2, c = 3
let (Point { x, y }, scale) = (origin, 2.0f64);
```

解構的形狀只需與值的結構相符。

## 3.15.5. 結構解構

對結構值，可以直接使用[模式](311-patterns.md)的 `struct` 模式。

```nabi
let Point { x, y } = origin;            // x = origin.x, y = origin.y
let Point { x, .. } = origin;           // discard other fields
let Point { x: px, y: py } = origin;    // rename
```

這可以一起取出多個欄位、只接收部分欄位而捨棄其餘，或為它們改名。

## 3.15.6. 型別標註

可以為解構的每個部分給出型別，也可以一次為整個模式給出型別。

```nabi
let (a: i32, b: str) = (1i32, "hi");    // type on each position
let (a, b): (i32, str) = (1i32, "hi");  // type on the whole pattern
```

標註的型別是槽位型別，不是執行時的型別檢查。它必須與值的型別相符；否則就是編譯錯誤。

```nabi
let (a: i32, b: str) = (1, 2);          // ERROR (E0040): second type mismatch
```

## 3.15.7. 只允許不可反駁的模式

`let pattern = expr;` 的模式必須是不可反駁的，也就是必須一律比對成功。

允許的模式：

- 識別字、萬用字元。
- 元組、結構與元組結構的完整比對（沒有遺漏的欄位或位置）。
- 上述模式的巢狀組合。

不允許的模式：

- 字面值模式（`1`、`"hello"` 等）。
- 只比對特定 `enum` 變體的模式（`Circle(r)`、`Click { x, y }` 等）。
- 型別檢查模式（`v: i32`，用在和型別或 `any` 槽位上）。
- 範圍模式。

這些形式可能比對失敗，因此以 `let` 接收它們是不安全的。請改用 [`let-else`](316-let-else.md)、[`match` 運算式](312-match-expression.md)或 `if let`（[`if` 運算式](310-if-expression.md)）。

## 3.15.8. 預設值（無）

不採用解構位置中的預設值。

```
let (a, b = 0i32) = ...;        // not adopted
```

函式參數的預設值被認為已經足夠。要在解構位置處理可能不存在的值，請使用 `match` 或 `let-else`。

## 3.15.9. 解構之後變數的可見性

由解構宣告的每個變數，都被引入 `let` 所在的作用域。

```nabi
let (a, b) = (1, 2);
// a and b are both visible here
```

由同一個 `let` 宣告的變數同時變得可見；沒有任何一個比另一個先可見。

## 3.15.10. 被比對之運算式的求值次數

解構右側的運算式恰好求值一次，並把結果與模式比對。因此帶有副作用的運算式保證只被呼叫一次。

## 3.15.11. 多重指派的一般化

解構指派可以視為多重指派的一般化。

```nabi
let (q, r) = divmod(num, den);          // two results
let (head, ..tail) = items;             // split
```

解構 `let` 宣告的是新變數；它不會寫入既有的變數。`let (a, b) = (b, a);` 會在其區塊的其餘部分遮蔽 `a` 與 `b`，因此在迴圈主體內，外層的 `a` 與 `b` 保持其值。要重新指派既有變數，請寫出不帶 `let` 的元組（3.15.12）：`(a, b) = (b, a + b);`。

## 3.15.12. 不帶 `let`：解構指派

既有位置組成的元組不帶 `let` 地指派：

```nabi
(a, b) = (b, a + b);             // reassigns a and b
```

`let` 宣告，單獨的元組目標重新指派，這與 `let x = ..` 和 `x = ..` 的區分相同。規則見 [3.4.6](304-assignment.md#346-destructuring-assignment)。

## 3.15.13. 參數中的模式

函式或閉包的參數可以是不可反駁的模式，而不是名稱（[4.2](../004-functions/402-parameters.md)、[4.5](../004-functions/405-closure.md)）：

```nabi
fn index(self, (i, j): (usize, usize)) -> f64 { self.data[i * self.cols + j] }
pairs.map(|(k, v)| f"{k}={v}")
```

該模式在主體開頭依本節的規則解構引數。
