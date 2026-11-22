# 3.4. 指派與複合指派

指派（`=`）與複合指派（`+=`、`-=` 等）更新變數、欄位或索引位置的值。兩者都是陳述式；簡單指派也可以以 3.4.5 的形式出現在運算式位置，而複合指派不行。文法如下：

```
assign_stmt     ::= lvalue "=" expr ";"
compound_stmt   ::= lvalue compound_op expr ";"
assign_expr     ::= lvalue "=" expr                    // in parentheses or an arm body (3.4.5)
compound_op     ::= "+=" | "-=" | "*=" | "**=" | "/=" | "%=" | "%%=" | "&=" | "|="
                  | "^=" | "<<=" | ">>="
assign_target   ::= lvalue | tuple_target
tuple_target    ::= "(" target_item ("," target_item)+ ","? ")"
target_item     ::= lvalue | tuple_target | "_"
lvalue          ::= identifier
                  | lvalue "." identifier
                  | lvalue "." int_lit                 // tuple position
                  | lvalue "[" expr "]"                // index
```

`assign_stmt` 的左側接受 `assign_target`：由位置組成的元組會指派其中每一個（3.4.6）。其他形式接受 `lvalue`。

## 3.4.1. 簡單指派

簡單指派的形狀如下。

```nabi
x = expr;
obj.field = expr;
arr[i] = expr;
```

左側（lvalue）是可以重新指派的位置，右側（rvalue）是任何運算式，結果值的型別是 unit（`()`）。指派通常是陳述式；當它出現在運算式位置（3.4.5）時，其結果是 unit。結果是，像 C 的 `if (x = read())` 這種很少需要的模式，被 unit 結果擋住了。

### lvalue 的種類

左側允許的形式及其意義：

| 形式             | 範例              | 意義                                             |
|------------------|-------------------|--------------------------------------------------|
| 簡單變數         | `x = ...`         | 把值寫入變數的槽位                               |
| 欄位存取         | `obj.field = ...` | 把值寫入物件的欄位                               |
| 元組位置         | `pair.0 = ...`    | 把值寫入元組 `struct` / 元組的某個位置           |
| 索引             | `arr[i] = ...`    | 把值寫入索引所指的位置                           |

索引指派去糖為[索引與切片](306-indexing-and-slicing.md)的寫入 trait。切片不是 lvalue：`arr[1:3] = other;` 會被拒絕（[3.6](306-indexing-and-slicing.md)）。

### 不能作為 lvalue 的東西

下列不能作為 lvalue。

```
1 = 2;                    // a literal is not an lvalue
foo() = 2;                // a call result is not an lvalue
foo().x = 2;              // nor is a field of one
(x + 1) = 2;              // an arithmetic result is not an lvalue
(a, 1) = (b, 2);          // each element of a tuple target is a place or `_` (3.4.6)
```

`List`、`Dict` 或其他共享集合的元素，即使該集合來自呼叫，也是位置：`rows()[0] = 1;` 寫入 `rows()` 回傳的串列。呼叫結果的欄位不是位置，作為呼叫結果的固定長度陣列的元素也不是。

某個東西能否作為 lvalue，在編譯期依其形式以及被索引之集合的型別檢查。不做任何執行時檢查。指派給不是位置的東西是錯誤（E0140），無論是 `=` 還是複合指派（3.4.3）：寫入會落在暫存值上而遺失。

## 3.4.2. 宣告新變數與重新指派

與 `let` [關鍵字](../001-lexical/105-keywords.md)結合時，區分兩種形式。

```nabi
let x = 1;       // new declaration, 'let' required
x = 2;           // reassignment, without 'let'
```

建立新變數需要 `let`；再次寫入既有變數的槽位則不寫 `let`。對尚未宣告的名稱不帶 `let` 地指派是編譯錯誤。在同一個作用域中對已宣告的名稱寫 `let` 是重新宣告：它建立一個新的繫結，可以有另一個型別，並讓舊的槽位保持原樣（[5.1.4](../005-modules/501-variable-declaration.md#514-redeclaration-in-the-same-scope)）。確切規則定義於[變數宣告](../005-modules/501-variable-declaration.md)。

```nabi
let n = "42";
let n: i32 = try! n.parse();   // redeclaration: a new binding of another type
n = n + 1;                     // reassignment of the new binding
```

```
counter = counter + 1;     // OK if counter is declared
coutner = ...;             // ERROR: `coutner` is not declared; a variable is created only by `let`
```

這條規則擋住了打錯字而默默建立新變數的陷阱（Julia 與 Python 中常見的錯誤）。

## 3.4.3. 複合指派

複合指派把運算與指派結合在一個形式中。

```nabi
x += 1;            // semantically the same as x = x + 1
x -= 1;
x *= 2;
arr[i] += 1;
```

[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)的所有複合指派語彙單元（`+=`、`-=`、`*=`、`**=`、`/=`、`%=`、`%%=`、`&=`、`|=`、`^=`、`<<=`、`>>=`）都遵循相同的模式。複合指派是陳述式，以 `;` 結束（在 `}` 之前可以省略，[1.3.4](../001-lexical/103-statement-terminator.md#134-block-like-expression-statements)）；它沒有值，不能出現在運算式位置（`let y = (x += 1);` 無法剖析）。原地廣播形式 `.=` `.+=` `.-=` `.*=` `./=` `.%=` `.%%=` `.**=` `.&=` `.|=` `.^=` `.<<=` `.>>=` 是同類的陳述式，有其自身的意義（[3.3.7](303-broadcast-operators.md#337-in-place-broadcast)）。

### 去糖與 lvalue 的求值

複合指派去糖如下。若型別實作了 `OpAssign` trait，`a += b` 就呼叫它；否則就是 `a = a + b`（[stdlib 2.4.1](../../refs-standard-libraries/200-core/204-ops-assign.md)）。每個運算子都是如此。

```
lvalue OP= rhs       →   OpAssign::op_assign(ref lvalue, rhs)   // if implemented
                     →   lvalue = (lvalue OP rhs)               // otherwise
```

編譯器保證 lvalue 不會被求值兩次。

```nabi
arr[expensive_index()] += 1;
// expensive_index() is called only once
```

若 lvalue 是單純的變數或欄位，就原樣參照兩次；若它包含呼叫或計算，該部分會繫結到暫存值並只求值一次。這項保證防止帶有副作用的索引或方法被無意間呼叫兩次。lvalue 與 `rhs` 何者先求值取決於運算元型別（3.4.7）。

lvalue 必須是位置（3.4.1）。`f() += 1` 與 `d.entry(k).or_insert(0) += 1` 是錯誤（E0140）：兩者都會更新暫存值。透過鎖守衛的 `get()` 寫入（`g.get().total += 1`）也是，因為 `get()` 回傳的是複本；說明會指出 `g.update(|v| { v.total += 1; v })`，或是整個值用 `g.set(v)`。要在字典中計數，請以帶預設值的方式索引（[3.6.1](306-indexing-and-slicing.md#361-indexing-arri)）：

```nabi
counts[w, default: 0] += 1;
```

### trait 對應

每個複合指派都對應到另一個 **Assign** trait 家族。

```
x += y       →   <typeof(x) as AddAssign<typeof(y)>>::add_assign(ref x, y)
x -= y       →   <typeof(x) as SubAssign<typeof(y)>>::sub_assign(ref x, y)
```

（有 `AddAssign`、`SubAssign`、`MulAssign`、`DivAssign`……等等。）

這些 trait 可以實作為原地更新，從而得到比 `x = x + y` 更有效率的實作。例如，對大型集合附加元素可以不必建立新的集合。

實作 `AddAssign` 是選用的：沒有它時，`x += y` 退回為 `x = x + y`。`x + y` 本身一律需要 `Add`。

### 簽章

`AddAssign` 等的簽章形狀如下。

```nabi
trait AddAssign<Rhs = Self> {
    #[infix("+=")]
    fn add_assign(ref self, rhs: Rhs);
}
```

和其他運算子一樣，`*Assign` trait 透過 `#[infix("op=")]` 註冊（[3.1](301-operator-dispatch.md)）。`ref self` 指稱 lvalue 的槽位；沒有回傳型別，結果是 unit。`ref self` 以及更新 lvalue 槽位的確切意義定義於函式規格。

## 3.4.4. 鏈式指派

nabi 不允許 C 與 Python 風格的鏈式指派（`a = b = c = 1`）。

```
a = b = c = 1;             // ERROR
```

由於指派的結果是 unit，`b = c = 1` 求值為 unit，整體就變成沒有意義的 `a = (unit)`。請把預期的意義寫成分開的指派：

```nabi
c = 1; b = 1; a = 1;
```

## 3.4.5. 運算式內的指派

簡單指派可以放在運算式位置，此時其結果是 unit，但這樣做幾乎沒有價值。

```nabi
let x = (y = 1);           // x = unit (nearly meaningless)
```

不鼓勵這種模式，工具或 linter 可能會對它發出警告。

在陳述式位置之外，指派要寫在括號中 `(y = 1)`，或作為 match 分支的主體。不支援沒有括號的 `let x = y = 1;`。複合指派沒有運算式形式。

## 3.4.6. 解構指派

`=` 左側由位置組成的元組，會從右側的對應元素指派每個位置：

```nabi
(a, b) = (b, a);                    // swap
for _ in 0..n {
    (x, y) = (y, x + y);            // Fibonacci step
}
(xs[i], xs[j]) = (xs[j], xs[i]);    // swap two elements
(p.x, _, p.y) = (1, 2, 3);          // `_` discards
(c, (d, e)) = t;                    // nested
```

右側先完整求值，然後由左至右指派每個位置；位置自身的接收者或索引在它被指派時才求值（3.4.7）。因此 `(a, b) = (b, a + b)` 讀取的是舊的 `a` 與 `b`。每個位置都必須已經存在：指派不會宣告（3.4.2），尚未宣告的名稱是 E0057，其說明會提示先以 `let` 宣告。兩側的形狀必須相符（E0101）。其意義等同於

```nabi
{ let (t0, t1) = (b, a + b); a = t0; b = t1; }
```

只是使用隱藏的暫存值。以 `let` 解構則是宣告新的變數（[3.15](315-destructuring.md)）；在迴圈內，`let (a, b) = (b, a + b);` 會為該次迭代宣告新的 `a` 與 `b`，並讓外層的保持原樣。

## 3.4.7. 求值順序

指派先對其值求值，再對其位置求值，與 Rust 相同。位置的各部分（它作為欄位或元素所屬的接收者，然後是其索引）在值之後由左至右各求值一次，最後才存入。

| 形式 | 順序 |
|------|-------|
| `place = value` | 先 `value`，再 `place` |
| `place op= value`，兩個運算元都是基本型別（整數、浮點數、`bool`、`char`、`str`） | 先 `value`，再 `place` |
| `place op= value`，其他任何運算元 | 先 `place`，再 `value` |
| `(p0, p1) = value`（3.4.6） | 先完整的 `value`，再 `p0`，再 `p1` |
| `place .= value`（[3.3.7](303-broadcast-operators.md#337-in-place-broadcast)） | 先依原始碼順序的 `value` 各運算元，再 `place` |
| `place .op= value` | 先 `place`，再 `value` 的各運算元 |

```nabi
xs[f()] = g();          // g(), then f()
xs[f()] += g();         // xs: List<i64>: g(), then f()
ms[f()] += g();         // ms: List<Money> with AddAssign: f(), then g()
(xs[f()], ys[h()]) = (g(), k());   // g(), k(), then f(), then h()
```

其他運算元的複合指派是對該位置呼叫 `OpAssign` 方法（3.4.3），因此它的位置先求值，就像任何方法呼叫中的接收者一樣（[4.2.8](../004-functions/402-parameters.md#428-parameter-evaluation-order)）；`.op=` 把其目標當作 `target .op value` 的第一個運算元讀取。值先求值時，會 panic 或擲回錯誤的值不會對位置的任何部分求值，而改變了位置所讀取之變數的值（透過 `[ref]` 擷取的索引、透過 `ref self` 方法的欄位），其改變會被位置看到：

```nabi
let i = 0;
let bump = [ref i] || -> i64 { i += 1; return 1; };
zs[i] = bump();         // stores at zs[1]
```
