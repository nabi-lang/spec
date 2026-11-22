# 2.4. 複合指派 trait

`core::ops` 的 `*Assign` trait 家族把 `a += b`、`a *= b` 這類複合指派運算子分派為**就地**更新。[refs-lang 3.4 指派與複合指派](../../refs-lang/003-expressions/304-assignment.md)所定義的語法會去語法糖為這些 trait。

> [!NOTE]
> Nabi 在一般程式碼中不使用 `mut` 關鍵字（[refs-lang 5.1 變數宣告](../../refs-lang/005-modules/501-variable-declaration.md)）。
> 變數一律以 `let` 宣告，再次指派給同一名稱是**重新指派**。
> 堆積物件（List、Dict 等）的就地更新，不是由 `let mut` 宣告子決定，而是由
> **該物件是否公開可變更的方法**決定。`*Assign` trait 就是這些可變更方法所在之處。

## 2.4.1. 關鍵差異：`+=` 與先 `=` 再 `+`

```nabi
a += b;
```

這個運算式有兩種可能的意義。採用**就地分派**時，它解析為 `<typeof(a) as AddAssign<typeof(b)>>::add_assign(ref a, b)`，直接修改 `a` 所參照物件的**內容**，不會建立新物件。採用**後備**（先 `Add::add` 再重新指派）時，它成為 `a = a + b`，把 `Add` 呼叫的結果存回 `a`，並建立新物件。

Nabi 會自動在兩種模式之間選擇。

```
Compiler behavior (on a += b):
1. If typeof(a) implements AddAssign<typeof(b)>
     → dispatch AddAssign::add_assign(ref a, b)
2. Otherwise, if typeof(a) implements Add<typeof(b)>
     → desugar to a = Add::add(a, b)
3. If neither, compile error
```

這項政策讓集合型別得以就地最佳化。例如當 `xs: List<i32>` 執行 `xs += y` 時，會直接 push，而不建立新的 List。

## 2.4.2. 算術 *Assign trait

### `AddAssign`：`+=`

```nabi
trait AddAssign<Rhs = Self> {
    #[infix("+=")]
    fn add_assign(ref self, rhs: Rhs);
}
```

```nabi
let counter: i32 = 0;
counter += 5;       //→ 5 (counter is updated in place)
counter += 3;       //→ 8

let xs: List<i32> = [1, 2];
xs += [3, 4];       //→ [1, 2, 3, 4]  (List's add_assign: efficient push)
```

### `SubAssign`：`-=`

```nabi
trait SubAssign<Rhs = Self> {
    #[infix("-=")]
    fn sub_assign(ref self, rhs: Rhs);
}
```

### `MulAssign`：`*=`

```nabi
trait MulAssign<Rhs = Self> {
    #[infix("*=")]
    fn mul_assign(ref self, rhs: Rhs);
}
```

### `PowAssign`：`**=`

```nabi
trait PowAssign<Rhs = Self> {
    #[infix("**=")]
    fn pow_assign(ref self, rhs: Rhs);
}
```

`x **= e` 即 `x = x ** e`；指數的實作與 `Pow`（2.3.1）相同：整數上為 `u32`，浮點數上為該浮點型別或 `i32`。

### `DivAssign`：`/=`

```nabi
trait DivAssign<Rhs = Self> {
    #[infix("/=")]
    fn div_assign(ref self, rhs: Rhs);
}
```

### `RemAssign`：`%=`

```nabi
trait RemAssign<Rhs = Self> {
    #[infix("%=")]
    fn rem_assign(ref self, rhs: Rhs);
}
```

### `ModuloAssign`：`%%=`

```nabi
trait ModuloAssign<Rhs = Self> {
    #[infix("%%=")]
    fn modulo_assign(ref self, rhs: Rhs);
}
```

`%%=` 是 [`Modulo`](203-ops.md#modulo向下取整模數)（向下取整的模數，結果的正負號等於除數的正負號）的複合指派形式。它是與 `RemAssign` 不同的 trait，並以就地方式更新，使得除數為正時，結果一定落在 `[0, rhs)` 之內。

## 2.4.3. 位元 *Assign trait

### `BitAndAssign`：`&=`

```nabi
trait BitAndAssign<Rhs = Self> {
    #[infix("&=")]
    fn bitand_assign(ref self, rhs: Rhs);
}
```

```nabi
let flags: u32 = 0xFF;
flags &= 0x0F;     //→ 0x0F
```

### `BitOrAssign`：`|=`

```nabi
trait BitOrAssign<Rhs = Self> {
    #[infix("|=")]
    fn bitor_assign(ref self, rhs: Rhs);
}
```

### `BitXorAssign`：`^=`

```nabi
trait BitXorAssign<Rhs = Self> {
    #[infix("^=")]
    fn bitxor_assign(ref self, rhs: Rhs);
}
```

### `ShlAssign`：`<<=`

```nabi
trait ShlAssign<Rhs = Self> {
    #[infix("<<=")]
    fn shl_assign(ref self, rhs: Rhs);
}
```

```nabi
let n: i32 = 1;
n <<= 4;       //→ 16
```

### `ShrAssign`：`>>=`

```nabi
trait ShrAssign<Rhs = Self> {
    #[infix(">>=")]
    fn shr_assign(ref self, rhs: Rhs);
}
```

## 2.4.4. 簽章模式：`ref self`

每個 `*Assign` 方法都接受 `ref self`（[refs-lang 4.7 ref 關鍵字](../../refs-lang/004-functions/407-ref-keyword.md)）。它參照呼叫端的變數繫結本身，因此修改會就地發生。

```nabi
fn add_assign(ref self, rhs: Rhs);
       ^^^^^
       directly modifies the object the caller's variable refers to
```

這個簽章是就地行為的核心。如果它只接受 `self`（以值傳遞），修改的就會是副本，呼叫端看不到變更。

## 2.4.5. 優先順序／結合性：prec=-2，右結合

每個 `*Assign` 語彙單元在 [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) 表格中都固定在同一層級。其優先順序為 `prec = -2`，低於一般算術（`+` 為 9，`*` 為 10）、`..`（=0）和 `|>`（=-1）。這是最低的層級。不允許鏈式指派（[refs-lang 3.4.4](../../refs-lang/003-expressions/304-assignment.md)）。

## 2.4.6. Trait 實作矩陣

基本型別的每個 `*Assign` 實作都帶有 `#[intrinsic("nabi.OP_assign.TYPE")]`。

| 型別         | Trait                                      | 行為                                        |
|--------------|---------------------------------------------|-------------------------------------------------|
| `i*`, `u*`   | 6 個算術 + 5 個位元（共 11 個 trait）        | `nabi.add_assign.i32`、`nabi.shl_assign.u32`…… |
| `f32`, `f64` | 只有 6 個算術                           | `nabi.add_assign.f64`……                       |
| `bool`       | `BitAndAssign`/`BitOrAssign`/`BitXorAssign` | `nabi.and_assign.bool`……                      |
| `int`        | 算術 + 位元；`ShlAssign<u32>` / `ShrAssign<u32>` | 自動提升邏輯；以新值取代（不可變）  |
| `float`      | 算術                                  | （同上：自動提升）                   |
| `List<T>`    | `AddAssign<T>`, `AddAssign<List<T>>`        | `xs += x` push `x`；`xs += ys` 就地 push `ys` 的每個元素 |
| `str`        | `AddAssign<str>`, `AddAssign<char>`         | `s = s + t`：以新的 str 取代 `s`（str 不可變） |
| `Dict<K,V>`  | `AddAssign<Dict<K,V>>`（`K: Hash`）          | 就地合併；共同的鍵以右側的值為準  |
| `Set<T>`     | `AddAssign<T>`（`T: Hash`）                  | `s += x` 插入 `x`                            |
| `Deque<T>`   | `AddAssign<T>`                              | `q += x` 把 `x` push 到尾端                 |

```nabi
let xs: List<i32> = [1];
xs += 2;              //→ [1, 2]
xs += [3, 4];         //→ [1, 2, 3, 4]

let s = "ab";
s += 'c';             //→ "abc"

let d: Dict<str, i32> = {"a": 1, "b": 2};
d += {"b": 20, "c": 3};   //→ {"a": 1, "b": 20, "c": 3}
```

左側型別實作了 `XxxAssign` 時，`a op= b` 呼叫 `XxxAssign::xxx_assign`（`AddAssign::add_assign`……）；否則即為 `a = a op b`。

## 2.4.7. 集合型別上的 *Assign：主要用途

對於 `List<T>` / `Dict<K,V>` 這類可變容器，`*Assign` 執行**真正的就地**運算。

```nabi
let xs: List<i32> = [1, 2, 3];
let alias = xs;             // shares the same object (Python-style, refs-lang 4.7)

xs += [4, 5];               // push onto xs (no new object)
alias;                       //→ [1, 2, 3, 4, 5]  (alias sees the change too)

// compare: xs = xs + [4, 5] creates a new List, and alias does not see the change
```

這個意義與 Nabi 對堆積物件的自動共用模型（[refs-lang 4.7](../../refs-lang/004-functions/407-ref-keyword.md)）完全一致。

## 2.4.8. 使用者定義的 *Assign

如果使用者型別只實作 `Add` 而未實作 `AddAssign`，編譯器會使用後備方式。

```nabi
struct Vec3 { x: f64; y: f64; z: f64; }

impl ops::Add for Vec3 {
    type Output = Vec3;
    fn add(self, rhs: Vec3) -> Vec3 { ... }
}
// AddAssign not defined

let v = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
v += Vec3 { x: 0.5, y: 0.5, z: 0.5 };
// → compiler: no AddAssign, Add exists → desugar 'v = v + (...)'
// Vec3 is immutable, so a new object is created and reassigned to v
```

在效能重要的地方，建議使用者明確實作 `AddAssign`。

```nabi
impl ops::AddAssign for Vec3 {
    fn add_assign(ref self, rhs: Vec3) {
        self.x += rhs.x;
        self.y += rhs.y;
        self.z += rhs.z;
    }
}
```

## 2.4.9. *Assign 的父 trait 關係：無

Nabi **不**把 `Add` 設為 `AddAssign` 的父 trait。這兩個 trait 彼此獨立。

原因是某些型別（例如集合型別）有意義的 `AddAssign`，卻沒有有意義的 `Add`：`Set<T>` 和 `Deque<T>` 實作 `AddAssign<T>`（插入或 push 一個元素），但沒有 `Add<T>`。父 trait 關係會讓這類型別無法只實作 `AddAssign`。

## 2.4.10. 相關：廣播的 `*Assign`？

廣播語彙單元沒有 `*Assign` trait。就地廣播改為語言語法：`y .= v` 以及 `y .+= v` `.-=` `.*=` `./=` `.%=` `.%%=` `.**=` `.&=` `.|=` `.^=` `.<<=` `.>>=` 逐元素寫入 `List`、固定長度陣列或 `Broadcastable` 目標，與右側的運算式融合，且不做任何配置（[refs-lang 3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-in-place-broadcast)）。比較運算以及 `.&&` `.||` `.??` 則透過 `.=`（`ok .= ok .&& xs .> 0`）。

```nabi
xs .+= ys;                // xs[i] = xs[i] + ys[i]; lengths must match
y .= a .* x .+ b;         // one loop into y
```

長度不相符時，與融合運算式一樣觸發 panic（`broadcast: lengths 3 and 2 differ; make both sides the same length`）。
