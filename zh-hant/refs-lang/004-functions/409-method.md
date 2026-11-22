# 4.9. 方法

本文件詳細定義在 [`impl`](../002-types/209-impl.md) 中定義之方法的參數與呼叫慣例。它是函式規格的一部分，著重於方法與自由函式之間的差異。

## 4.9.1. 方法定義的位置

方法定義在 [`impl`](../002-types/209-impl.md) 區塊內。

```nabi
impl Point {
    fn origin() -> Point { Point { x: 0.0f64, y: 0.0f64 } }
    fn distance(self, other: Point) -> f64 { ... }
    fn translate(ref self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }
}
```

## 4.9.2. 依第一個參數形式區分的方法種類

### 靜態方法：沒有 `self`

```nabi
impl Point {
    fn origin() -> Point { ... }              // static
    fn from_polar(r: f64, theta: f64) -> Point { ... }
}

let p = Point::origin();                       // Type::method() call
let q = Point::from_polar(1.0f64, 0.0f64);
```

靜態方法是第一個參數既不是 `self` 也不是 `ref self` 的函式。它以 `Type::method(args)` 呼叫，這是透過[型別路徑](../002-types/213-type-paths.md)的靜態項目存取。

### 實例方法：`self`

```nabi
impl Point {
    fn distance(self, other: Point) -> f64 { ... }
    fn norm(self) -> f64 { ... }
}

let d = p.distance(q);                         // instance.method() call
let n = p.norm();
```

在這種形式中，第一個參數是 `self`。`self` 表示實例的值（對 `struct` 而言是一份複本，[2.3.8](../002-types/203-struct.md#238-值語意)），主體以 `self.field` 存取欄位。它以 `instance.method(args)` 呼叫。

### 實例方法：`ref self`

```nabi
impl Point {
    fn translate(ref self, dx: f64, dy: f64) {
        self.x = self.x + dx;
        self.y = self.y + dy;
    }

    fn replace_with(ref self, other: Point) {
        self = other;                          // the caller's variable becomes other
    }
}

let p = Point { x: 1.0, y: 2.0 };
p.translate(5.0f64, 0.0f64);                   // called without explicit ref
p.replace_with(other);                         // likewise
```

在這種形式中，第一個參數是 `ref self`。在主體內，`self` 及其欄位可以被指派，結果會寫回呼叫者的變數（見 [`ref` 關鍵字](407-ref-keyword.md)）。呼叫者不寫 `ref`，因為方法呼叫的接收者會被自動處理。

## 4.9.3. `self` 關鍵字的兩種用法

```nabi
impl Point {
    fn distance(self, other: Point) -> f64 {
        let dx = self.x - other.x;             // uses the instance self
        let dy = self.y - other.y;
        (dx * dx + dy * dy).sqrt()
    }
}
```

`self`（[關鍵字](../001-lexical/105-keywords.md)）用在兩個位置。在參數位置，它是接收實例的參數；在主體內，它是指稱該參數的識別字。

參數位置的 `self` 不寫型別。它的型別自動是 `Self`，即目前 `impl` 的目標型別。

## 4.9.4. `Self` 關鍵字：型別位置

`Self`（[關鍵字](../001-lexical/105-keywords.md)，大寫）是指稱目前 `impl` 之目標型別的型別名稱。

```nabi
impl Point {
    fn new(x: f64, y: f64) -> Self {           // Self == Point
        Self { x, y }                          // Self can also construct instances
    }
}
```

兩者必須清楚區分：小寫的 `self` 用作參數與實例識別字；大寫的 `Self` 用作型別名稱。

## 4.9.5. 方法呼叫接收者的自動 `ref`

對 `ref self` 方法而言，即使呼叫者在呼叫時沒有寫，也會自動套用 `ref`。

```nabi
impl Counter {
    fn increment(ref self) { self.count = self.count + 1; }
}

let c = Counter { count: 0 };
c.increment();                                 // no explicit 'ref'
```

這是 [`ref` 關鍵字](407-ref-keyword.md)中一般函式呼叫規則（必須明確寫出 `ref`）的例外。理由是：方法呼叫語法 `x.method()` 本身就是接收者的[分派](../000-meta/007-glossary.md#分派dispatch)機制，而且無論接收者是 `self` 還是 `ref self`，呼叫形式都相同。額外明確寫出的 `ref` 只會是視覺上的雜訊。

這與 Rust、C# 和 Swift 的方法呼叫慣例一致。不過，以路徑呼叫同一個方法時，接收者是一般的引數，因此要寫 `ref`（`Counter::increment(ref c)`，[4.7.3](407-ref-keyword.md#473-呼叫時明確寫出c-風格)）。

## 4.9.6. 自由函式與方法的呼叫形式

| 形式            | 呼叫                    |
|-----------------|-------------------------|
| 自由函式        | `function_name(args)`   |
| 靜態方法        | `Type::method(args)`    |
| 實例方法        | `instance.method(args)` |

呼叫形式清楚區分了函式的意義。`x.f(args)` 一律指稱方法：它絕不會呼叫欄位 `f`，即使該欄位容納 `fn` 值、閉包或具有 `ops::Call` 的值也一樣。這樣的欄位要以括號包起來呼叫，即 `(x.f)(args)`，與 Rust 相同。當 `x` 沒有方法 `f` 但有可呼叫的欄位 `f` 時，錯誤的說明會提示寫 `(x.f)(..)`。

### trait 之間的名稱衝突

當 `x.method()` 有多個候選時，依下列順序選出一個：

1. 固有 `impl` 的方法優先於 trait 方法。
2. 在 trait 方法中，目前套件定義的或明確 `use` 的 trait 優先於 `core` 的 trait，後者在任何地方都在方法作用域內。目前套件是寫出該呼叫的實際套件。除非經過 `use`，相依套件的 trait 不會優先於 core 的 trait。
3. 若仍有多個候選，就是歧義錯誤，診斷會建議寫成 `Trait::method(x)`。同一個 trait 在型別參數上不同的多個 impl（`Call<f64>`、`Call<Array<f64, N>>`）不算衝突：由引數選出 impl（[3.1.3](../003-expressions/301-operator-dispatch.md#313-不經-vtable-的分派)）。

```nabi
use pkg::geom::Union;              // user trait: union method
a.union(b);                        // pkg::geom::Union takes precedence over core's Union
core::ops::Union::union(a, b) // explicit to use the core one
```

## 4.9.7. 在方法內呼叫其他方法

```nabi
impl Point {
    fn distance(self, other: Point) -> f64 { ... }

    fn distance_squared(self, other: Point) -> f64 {
        let d = self.distance(other);          // another method of the same instance
        d * d
    }
}
```

同一個實例的其他方法可以用 `self.method()` 呼叫。適用與一般實例呼叫相同的規則。

## 4.9.8. 方法的可見性

方法的可見性遵循 [`impl`](../002-types/209-impl.md) 中定義的模組私有預設。

```nabi
impl Account {
    pub fn balance(self) -> f64 { ... }        // public
    fn _internal(self) { ... }                  // module-private
}
```

可見性等級的確切規則定義於[可見性](../005-modules/506-visibility.md)。

## 4.9.9. 方法簽章的自由度

方法的參數、回傳與主體遵循[參數](402-parameters.md)與[回傳](403-return.md)的一般函式規則。具體而言，預設值、具名引數、可變參數與展開都可以在方法中自由使用。主體一律經由 `throw` 或無限迴圈離開的發散方法也是可能的，而泛型方法可以同時使用 `impl` 區塊的泛型參數與方法自己的泛型。

```nabi
impl<T> List<T> {
    fn map<U>(self, f: fn(T) -> U) -> List<U> { ... }
}
```

方法自己的泛型接在函式名稱之後（`fn map<U>(...)`）。在 trait `impl` 中，沒有型別的方法參數取 trait 在該位置宣告的型別（代入 trait 參數與關聯型別），而不是隱含的型別參數。

## 4.9.10. 方法中 `self` 的總結

| 第一個參數      | 意義                      | 呼叫時                               | 能否在主體中重新指派 `self`？         |
|-----------------|---------------------------|--------------------------------------|---------------------------------------|
| （無）          | 靜態方法                  | `Type::method()`                     | -                                     |
| `self`          | 實例方法（值）            | `instance.method()`                  | 否（只影響區域）                      |
| `ref self`      | 實例方法（`ref`）         | `instance.method()`（自動 `ref`）    | 是（反映到呼叫者）                    |

這三種形式是方法僅有的種類。
