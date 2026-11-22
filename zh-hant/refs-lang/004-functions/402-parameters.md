# 4.2. 參數

本節定義函式參數的形式，以及呼叫處引數配對的規則。

```
param           ::= self_param | regular_param
self_param      ::= "ref"? "self"
regular_param   ::= "ref"? identifier (":" type_expr)? ("=" const_expr)?
                  | param_pattern (":" type_expr)?
param_pattern   ::= tuple_pattern | struct_pattern
```

`param_pattern` 是取代名稱的不可反駁模式（[3.11](../003-expressions/311-patterns.md)）：`fn f((a, b): (i32, i32))`、`fn g(P { x, y }: P)`。引數在主體開頭被解構，就像 `let (a, b) = arg;` 一樣（[3.15](../003-expressions/315-destructuring.md)）：長度與參數型別不同的元組模式是 E0101，可反駁的模式是 E0100。這樣的參數對呼叫者而言沒有名稱，因此不接受預設值，也不接受具名引數。`ref` 參數一律是名稱（`ref (a, b)` 是錯誤，E0010）。

## 4.2.1. 基本形式

```nabi
fn f(x: i32, y: i32) -> i32 { x + y }
```

每個參數寫成 ` name: type`，以逗號分隔。參數名稱遵循[識別字](../001-lexical/104-identifiers.md)的形式，型別遵循[型別運算式](../002-types/201-type-expressions.md)的規則。

## 4.2.2. 省略型別

在模組私有或 `pub(sub)` 函式上，沒有型別的參數被視為泛型。詳見[泛型](../002-types/210-generics.md)的單態化規則。在 trait `impl` 中，沒有型別的方法參數取 trait 在該位置宣告的型別（代入 trait 參數與關聯型別），而不是隱含的型別參數。

```nabi
fn id(x) { x }                                   // x is generic; monomorphized at call
fn pair(x, y) { ... }                             // both parameters generic
fn mixed(a: i32, b) { ... }                       // a is i32, b is generic
```

上面的 `id` 等價於明確寫出[泛型](../002-types/210-generics.md)的形式：

```nabi
fn id<T>(x: T) -> T { x }
```

主體中使用的 trait 不會被自動合成為簽章中的 `where` 子句。每個呼叫處都會單態化，並在那裡檢查主體對 trait 的使用。

```nabi
fn compute(x, y) { x + y }                       // signature: <T, U>(x: T, y: U)
                                                 //   no where clause synthesized

compute(1i32, 2i32);                             // OK: i32 + i32 dispatch
compute(1.0f64, 2.0f64);                         // OK: separate monomorphized instance
compute(MyType, 1i32);                           // error at the call site:
                                                 //   "MyType has no Add<i32>"
                                                 //   (reported clearly where it is called,
                                                 //    not at the signature)
```

若合成一個累積了主體中每個 trait 的 `where` 子句，簽章會膨脹，而由此推論出的型別會以巨大的形式出現在工具中；因此不做任何合成。

`pub` 或 `pub(pkg)` 函式要寫出每個參數的型別（上述 trait `impl` 方法除外）。其他模組會呼叫它，而隱含參數會讓修改主體改變它們能傳入的東西。那裡未定型的參數是編譯錯誤；請寫出型別，或寫出帶有主體所需約束的型別參數。

```nabi
pub fn add(x, y) { x + y }                       // ERROR: `x` and `y` have no type
pub fn add(x: i32, y: i32) -> i32 { x + y }      // OK
pub fn add<T: ops::Add<Output = T>>(x: T, y: T) -> T { x + y }   // OK: any T with Add
```

若參數意在成為動態槽位，請明確寫出 `any`（見 [`any` 槽位](../002-types/202-any-slot.md)）。

```nabi
fn handle(x: any) { ... }                         // x is an any slot
```

## 4.2.3. 預設值

參數可以有預設值。

```nabi
fn retry(n: i32 = 3i32, delay: f64 = 0.1f64) { ... }

retry();                          // n=3, delay=0.1
retry(5i32);                       // n=5, delay=0.1
retry(5i32, 1.0f64);               // n=5, delay=1.0
```

預設值適用下列規則。只允許 `const` 運算式：字面值、`const` 變數與 `const` 函式呼叫。這與 [`struct`](../002-types/203-struct.md) 欄位預設值的限制相同。只有在呼叫處省略的參數才會被求值，而且每次呼叫都重新求值。預設值是確定性的：相同的 `const` 運算式一律產生相同的值。目前時間或隨機數之類的執行時呼叫不能使用；需要時請改用工廠函式模式。預設值中的名稱在宣告該函式之模組的作用域中解析，而不是呼叫者的作用域，另外還包括函式自己的 `const` 參數，它們由較前面的引數固定（`a: Array<f64, N>` 時的 `axis: usize = N - 1`）。預設值取其參數在引數繫結後的型別，因此 `xs: List<T> = []` 是該次呼叫之 `T` 的 `List<T>`。不是 `const` 運算式的預設值在宣告處就是錯誤。預設值位於尾端：在有預設值之參數後面出現沒有預設值的參數，在宣告處是錯誤。有兩種參數豁免：可變參數（4.2.5），以及由尾隨閉包填入、型別為 `fn` 的最後一個參數（[4.8.9](408-trailing-closure.md#489-與具名引數結合)）。

## 4.2.4. 具名引數呼叫

在呼叫處，可以依參數名稱給出引數。

```nabi
fn make_user(name: str, age: i32 = 0i32, email: str = "") -> User { ... }

make_user(name: "Alice", age: 30i32);
make_user(name: "Bob", email: "bob@example.com");      // age uses its default
make_user("Charlie", age: 25i32);                       // only the first argument positional, the rest named
```

具名引數呼叫適用下列規則。位置引數必須在前，具名引數在後；在具名引數之後出現位置引數是錯誤（E0049）。在一次呼叫中同時以位置與名稱給出同一個參數是錯誤。所有位置引數配對完畢後，剩下的參數必須由具名引數或預設值填入。

## 4.2.5. 可變參數：`...T`

接受可變數量引數的參數。

```nabi
fn sum(xs: ...i32) -> i32 {
    let total = 0i32;
    for x in xs { total += x; }
    total
}

sum();                            // xs: empty slice
sum(1i32);                         // xs = [1]
sum(1i32, 2i32, 3i32);             // xs = [1, 2, 3]
let ys: List<i32> = [4, 5];
sum(..ys);                         // xs = [4, 5] (list spread)
```

可變參數適用下列規則：

- 只有最後一個參數可以是可變參數。
- 在主體中，`xs: ...T` 是元素型別為 `T` 的唯讀切片視圖（`Slice<T>`，[stdlib 2.9.7](../../refs-standard-libraries/200-core/209-collections.md#297-slicet檢視)）。呼叫中剩餘的引數被打包成一個，長度在每次呼叫中可以不同。因此函式不會依引數數量分別單態化。
- 可以混用個別引數、元組展開與串列展開（`..xs`）（4.2.6）。
- 主體可以讀取、索引與迭代它。若需要修改，請以函式庫方法建立新的集合。

### 異質可變參數：`...any`

```nabi
fn log_all(xs: ...any) { ... }
log_all(1i32, "hi", 3.14f64);     // various types allowed
```

參數型別為 `any` 時，可變參數接受異質的引數。

## 4.2.6. 展開：`..args`

在呼叫處把既有集合展開為引數的形式。

```nabi
let args = (1i32, 2i32, 3i32);
foo(..args);                      // foo(1, 2, 3)

let xs: List<i32> = [1i32, 2i32, 3i32];
sum(..xs);                         // sum(1, 2, 3) (received as variadic)
```

展開適用下列規則：

- 元組展開在編譯期檢查其長度與型別。
- 展開到固定參數的 `List` / `Slice` 填滿直到下一個引數為止的槽位，並在執行時檢查長度。陣列展開填滿其長度。
- Dict 展開被轉換為具名引數。每個鍵都必須指稱一個未以其他方式填入的參數（在執行時檢查），其值必須具有該參數的型別。Dict 的值型別 `V` 不適合的有預設值參數取其預設值；指稱它的鍵會在執行時 panic。
- 在可變參數位置，元組、`List`、`Slice` 與陣列展開可以與一般引數混用（`f(a, ..xs, b)`）。可變參數可以原樣傳給另一個可變參數（`fn f(xs: ...i32) { g(..xs) }`）。
- 當運算元是內部沒有 `..` 的 `List` 字面值，或鍵都是字串字面值的 Dict 字面值時，編譯器也會在編譯期檢查它。長度與其所填槽位數不同，或鍵沒有指稱任何剩下待填的參數時，會產生警告 lint `literal_spread_mismatch`（`spread of 3 elements into 2 parameters; this call panics`）。執行時檢查仍然保留。
- 展開非集合的值是錯誤（E0051，``spread `f(..x)` needs a tuple, a `List`, a `Slice`, an array or a `Dict<str, V>`, found `i32` ``）。
- 以零個引數呼叫元素型別 `T` 為開放泛型的可變參數，無法決定 `T`，因此需要型別標註。

- 在 `List` 字面值中，`..x` 展開元組、陣列、`List` 或 `Slice`（`[0, ..xs, 9]`）；否則它是範圍。

```nabi
// fn train(lr: f64, momentum: f64, epochs: i32 = 10)
let opts = {"lr": 0.01, "momentum": 0.9};   // Dict<str, f64>
train(..opts);                  // train(lr: 0.01, momentum: 0.9), epochs: 10
// A key "epochs" in opts would panic: `f64` does not fit `epochs: i32`.
```

在一次呼叫中，展開與一般引數也可以自由混用。

```nabi
foo(0, ..middle, 10);
foo(..a, ..b, ..c);
```

## 4.2.7. 捨棄：`_`

明確捨棄呼叫結果的記法。

```nabi
_ = compute_with_side_effect();
```

這裡的 `_` 是[識別字](../001-lexical/104-identifiers.md)的單一底線。在左側位置，它接收一個值但不把它繫結到任何地方，表示使用者刻意忽略結果。`let _ = e;` 相同，可以在同一個作用域中使用任意多次（`_` 不是繫結，因此不是重新宣告）。

## 4.2.8. 參數求值順序

呼叫時，引數運算式依寫出的順序（由左至右）求值。方法呼叫先對接收者求值。

```nabi
foo(a(), b(), c());               // a() first, then b(), c() last
```

具名引數也依寫出的順序求值。

```nabi
make_user(name: f(), age: g());   // f() first, then g()
```

這個順序保證帶有副作用的引數以可預測的順序執行。

## 4.2.9. 參數的作用域

參數被引入函式主體的頂層作用域，並在主體內像一般變數一樣使用。

預設值運算式在沒有參數的環境中求值。因此它們不能參照同一個函式的其他參數。

```nabi
fn f(a: i32, b: i32 = a + 1) { ... }   // ERROR: default cannot refer to a
```

這項限制的動機與 [`struct`](../002-types/203-struct.md) 的欄位預設值限制相同：確定性與可預測的成本。

## 4.2.10. 靜態參數與漸進式參數

| 寫法         | 意義                                                 |
|--------------|------------------------------------------------------|
| `x: i32`     | 靜態。在呼叫處只接受 i32                             |
| `x`          | 泛型。依呼叫處的型別單態化                           |
| `x: any`     | 動態槽位。以執行時標籤接受任何型別                   |

這三種形式可以在一個函式中自由混用。
