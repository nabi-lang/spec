# 2.4. Compound assignment traits

The `*Assign` trait family of `core::ops` dispatches compound assignment operators such as `a += b` and `a *= b` as **in-place** updates. The syntax defined in [refs-lang 3.4 Assignment and compound assignment](../../refs-lang/003-expressions/304-assignment.md) desugars to these traits.

> [!NOTE]
> Nabi does not use a `mut` keyword in ordinary code ([refs-lang 5.1 Variable declarations](../../refs-lang/005-modules/501-variable-declaration.md)).
> Variables are always declared with `let`, and assigning to the same name again is a **reassignment**.
> In-place update of a heap object (List, Dict, etc.) is determined not by a `let mut` declarator but by
> **whether the object exposes mutating methods**. The `*Assign` traits are where those mutating methods live.

## 2.4.1. The key difference - `+=` vs `=` then `+`

```nabi
a += b;
```

This expression has two possible meanings. With **in-place dispatch**, it resolves to `<typeof(a) as AddAssign<typeof(b)>>::add_assign(ref a, b)`, which directly modifies the **contents** of the object `a` refers to and creates no new object. With the **fallback** (`Add::add` followed by reassignment), it becomes `a = a + b`, storing the result of the `Add` call back into `a` and creating a new object.

Nabi chooses between the two modes automatically.

```
Compiler behavior (on a += b):
1. If typeof(a) implements AddAssign<typeof(b)>
     → dispatch AddAssign::add_assign(ref a, b)
2. Otherwise, if typeof(a) implements Add<typeof(b)>
     → desugar to a = Add::add(a, b)
3. If neither, compile error
```

This policy enables in-place optimization of collections. For example, when `xs: List<i32>` performs `xs += y`, it pushes directly without creating a new List.

## 2.4.2. Arithmetic *Assign traits

### `AddAssign` - `+=`

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

### `SubAssign` - `-=`

```nabi
trait SubAssign<Rhs = Self> {
    #[infix("-=")]
    fn sub_assign(ref self, rhs: Rhs);
}
```

### `MulAssign` - `*=`

```nabi
trait MulAssign<Rhs = Self> {
    #[infix("*=")]
    fn mul_assign(ref self, rhs: Rhs);
}
```

### `PowAssign` - `**=`

```nabi
trait PowAssign<Rhs = Self> {
    #[infix("**=")]
    fn pow_assign(ref self, rhs: Rhs);
}
```

`x **= e` is `x = x ** e`; the exponent impls are those of `Pow` (2.3.1): `u32` on an integer, the float type or `i32` on a float.

### `DivAssign` - `/=`

```nabi
trait DivAssign<Rhs = Self> {
    #[infix("/=")]
    fn div_assign(ref self, rhs: Rhs);
}
```

### `RemAssign` - `%=`

```nabi
trait RemAssign<Rhs = Self> {
    #[infix("%=")]
    fn rem_assign(ref self, rhs: Rhs);
}
```

### `ModuloAssign` - `%%=`

```nabi
trait ModuloAssign<Rhs = Self> {
    #[infix("%%=")]
    fn modulo_assign(ref self, rhs: Rhs);
}
```

`%%=` is the compound assignment form of [`Modulo`](203-ops.md#modulo---floored-modulo-) (floored modulo, sign of result = sign of divisor). It is a separate trait from `RemAssign`, and updates in place so that, for a positive divisor, the result always lies in `[0, rhs)`.

## 2.4.3. Bitwise *Assign traits

### `BitAndAssign` - `&=`

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

### `BitOrAssign` - `|=`

```nabi
trait BitOrAssign<Rhs = Self> {
    #[infix("|=")]
    fn bitor_assign(ref self, rhs: Rhs);
}
```

### `BitXorAssign` - `^=`

```nabi
trait BitXorAssign<Rhs = Self> {
    #[infix("^=")]
    fn bitxor_assign(ref self, rhs: Rhs);
}
```

### `ShlAssign` - `<<=`

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

### `ShrAssign` - `>>=`

```nabi
trait ShrAssign<Rhs = Self> {
    #[infix(">>=")]
    fn shr_assign(ref self, rhs: Rhs);
}
```

## 2.4.4. Signature pattern - `ref self`

Every `*Assign` method takes `ref self` ([refs-lang 4.7 ref keyword](../../refs-lang/004-functions/407-ref-keyword.md)). It refers to the caller's variable binding itself, so the modification happens in place.

```nabi
fn add_assign(ref self, rhs: Rhs);
       ^^^^^
       directly modifies the object the caller's variable refers to
```

This signature is the core of in-place behavior. If it took only `self` (by value), it would modify a copy, and the caller would not see the change.

## 2.4.5. Precedence / associativity - prec=-2, right

Every `*Assign` token is fixed at the same level in the [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) table. Its precedence is `prec = -2`, lower than ordinary arithmetic (`+` is 9, `*` is 10), `..` (=0) and `|>` (=-1). It is the lowest level. Chained assignment is not allowed ([refs-lang 3.4.4](../../refs-lang/003-expressions/304-assignment.md)).

## 2.4.6. Trait implementation matrix

Every `*Assign` impl for primitive types carries `#[intrinsic("nabi.OP_assign.TYPE")]`.

| Type         | Traits                                      | Behavior                                        |
|--------------|---------------------------------------------|-------------------------------------------------|
| `i*`, `u*`   | 6 arithmetic + 5 bitwise (11 traits)        | `nabi.add_assign.i32`, `nabi.shl_assign.u32`, ... |
| `f32`, `f64` | 6 arithmetic only                           | `nabi.add_assign.f64`, ...                       |
| `bool`       | `BitAndAssign`/`BitOrAssign`/`BitXorAssign` | `nabi.and_assign.bool`, ...                      |
| `int`        | Arithmetic + bitwise; `ShlAssign<u32>` / `ShrAssign<u32>` | auto-promote logic; replaced by a new value (immutable)  |
| `float`      | Arithmetic                                  | (same as above: auto-promote)                   |
| `List<T>`    | `AddAssign<T>`, `AddAssign<List<T>>`        | `xs += x` pushes `x`; `xs += ys` pushes every element of `ys`, in place |
| `str`        | `AddAssign<str>`, `AddAssign<char>`         | `s = s + t`: a new str replaces `s` (str is immutable) |
| `Dict<K,V>`  | `AddAssign<Dict<K,V>>` (`K: Hash`)          | merge in place; the right side's values win on shared keys  |
| `Set<T>`     | `AddAssign<T>` (`T: Hash`)                  | `s += x` inserts `x`                            |
| `Deque<T>`   | `AddAssign<T>`                              | `q += x` pushes `x` at the back                 |

```nabi
let xs: List<i32> = [1];
xs += 2;              //→ [1, 2]
xs += [3, 4];         //→ [1, 2, 3, 4]

let s = "ab";
s += 'c';             //→ "abc"

let d: Dict<str, i32> = {"a": 1, "b": 2};
d += {"b": 20, "c": 3};   //→ {"a": 1, "b": 20, "c": 3}
```

`a op= b` calls `XxxAssign::xxx_assign` (`AddAssign::add_assign`, ...) when the left type implements it; otherwise it is `a = a op b`.

## 2.4.7. *Assign on collections - the main use

For mutable containers such as `List<T>` / `Dict<K,V>`, `*Assign` performs a **true in-place** operation.

```nabi
let xs: List<i32> = [1, 2, 3];
let alias = xs;             // shares the same object (Python-style, refs-lang 4.7)

xs += [4, 5];               // push onto xs (no new object)
alias;                       //→ [1, 2, 3, 4, 5]  (alias sees the change too)

// compare: xs = xs + [4, 5] creates a new List, and alias does not see the change
```

This meaning matches Nabi's automatic sharing model for heap objects ([refs-lang 4.7](../../refs-lang/004-functions/407-ref-keyword.md)) exactly.

## 2.4.8. User-defined *Assign

If a user type implements only `Add` and not `AddAssign`, the compiler uses the fallback.

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

Where performance matters, the user is advised to implement `AddAssign` explicitly.

```nabi
impl ops::AddAssign for Vec3 {
    fn add_assign(ref self, rhs: Vec3) {
        self.x += rhs.x;
        self.y += rhs.y;
        self.z += rhs.z;
    }
}
```

## 2.4.9. Supertrait relationship of *Assign - none

Nabi does **not** make `Add` a supertrait of `AddAssign`. The two traits are independent.

The reason is that some types, such as collections, have a meaningful `AddAssign` but no meaningful `Add`: `Set<T>` and `Deque<T>` implement `AddAssign<T>` (insert or push one element) and no `Add<T>`. A supertrait relationship would make it impossible for such types to implement only `AddAssign`.

## 2.4.10. Related - broadcast `*Assign`?

The broadcast tokens have no `*Assign` traits. In-place broadcast is language syntax instead: `y .= v` and `y .+= v` `.-=` `.*=` `./=` `.%=` `.%%=` `.**=` `.&=` `.|=` `.^=` `.<<=` `.>>=` write element-wise into a `List`, a fixed array or a `Broadcastable` target, fused with the expression on the right and with no allocation ([refs-lang 3.3.7](../../refs-lang/003-expressions/303-broadcast-operators.md#337-in-place-broadcast)). The comparisons and `.&&` `.||` `.??` go through `.=` (`ok .= ok .&& xs .> 0`).

```nabi
xs .+= ys;                // xs[i] = xs[i] + ys[i]; lengths must match
y .= a .* x .+ b;         // one loop into y
```

A length mismatch panics like a fused expression (`broadcast: lengths 3 and 2 differ; make both sides the same length`).
