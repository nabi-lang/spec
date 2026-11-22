# 0.3. 內建函式機制：編譯器 ↔ 標準函式庫的界線

標準函式庫 trait 方法的主體，例如 `i32::add`，最終必須降階為機器的 `add` 指令。本章定義標準函式庫方法主體連接到編譯器後端（PrimOp / MIR / 程式碼產生）的途徑。

## 0.3.1. 問題：自舉環狀相依

問題的核心是標準函式庫與編譯器之間的自舉環狀相依。下面的流程說明卡在哪裡。

```
user code:         let r = a + b;        (a, b: i32)
↓
compiler:          dispatch Add::add(a, b) call     (refs-lang 3.1)
↓
standard library:  impl Add for i32 { fn add(self, rhs) { ... } }
              ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
              what goes in the "..." here?
              without an intrinsic, "self + rhs" would dispatch Add::add again → infinite recursion
              a raw machine `add` cannot be written either (not part of the user language surface)
↓
machine code:      add w0, w0, w1          (aarch64)
```

因此需要一條途徑，讓標準函式庫的主體不呼叫自身，直接通往編譯器後端的 `add` 指令。

## 0.3.2. 解法：`#[intrinsic("nabi.OP.TYPE")]` 屬性

解法是在標準函式庫的方法主體上附加編譯器能辨識的屬性。編譯器遇到這種呼叫時，會直接把它轉換為 PrimOp / MIR，而不是降階其主體。

```nabi
impl Add for i32 {
    type Output = i32;

    #[intrinsic("nabi.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // an empty body is OK; the compiler uses the intrinsic path
        // even if a body exists, the compiler takes precedence
    }
}
```

自由函式（`mem::size_of`、`ptr::addr_of`）與固有方法（`MaybeUninit::as_ptr`）也以相同方式帶有 `#[intrinsic]`；對它們的呼叫就是內建函式。

編譯器的處理步驟如下。分派 `Add::add(a, b)` 呼叫時，它在 method_table 中找到 `i32::add`，接著檢查該方法是否附加了 `#[intrinsic("nabi.add.i32")]`。若有，就直接產生 `PrimOp::Add { ty: i32 }`，而不降階主體；之後它會經過 MIR 與程式碼產生，最終到達目標的 32 位元 `add` 指令。

此機制把編譯器的攔截吸收進 trait 分派之中。

## 0.3.3. 內建函式命名慣例

內建函式名稱採用以下三種形式之一：

- `nabi.<OP>.<TYPE>`：針對單一型別的基本運算（`nabi.add.i32`），包括複合指派 `nabi.<OP>_assign.<TYPE>`（`nabi.add_assign.i32`）；
- `nabi.<area>.<op>`：某個範圍（例如 `ptr` 或 `atomic`）的運算（`nabi.ptr.read`、`nabi.atomic.cas`）；
- `nabi.<name>`：名稱中不含型別的泛用運算（`nabi.size_of`、`nabi.align_of`、`nabi.addr_of`、`nabi.drop_in_place`、`nabi.hand_over`），包括 GC 緩衝區與交接家族 `nabi.buf_<op>` 與 `nabi.transfer_<op>`（`nabi.buf_get`、`nabi.transfer_keep`）。

基本形式：

```
nabi.add.i32        i32 + i32
nabi.add.i64
nabi.add.f32
nabi.add.f64
nabi.sub.i32
nabi.mul.i32
nabi.div.i32
nabi.rem.i32
nabi.and.i32        bitwise AND
nabi.or.i32
nabi.xor.i32
nabi.shl.i32
nabi.shr.i32
nabi.eq.i32
nabi.lt.i32
...
nabi.neg.i32        unary
nabi.not.bool       logical not (bool)
nabi.not.i32        bitwise not (integer)
nabi.cast.i32.i64   type cast
```

基本形式的每個 OP 都與編譯器的 `PrimOp` 列舉一一對應。名稱本身沿用常見的指令名稱與 Rust 的內建函式命名慣例。

## 0.3.4. 主體後備政策

附加了內建函式的方法仍然可以有主體。下面的範例說明這一點。

```nabi
impl Add for BigInt {
    type Output = BigInt;

    // BigInt is not fixed-width, so no intrinsic; the body is authoritative
    fn add(self, rhs: BigInt) -> BigInt {
        // actual BigInt addition implementation
        ...
    }
}

impl Add for i32 {
    type Output = i32;

    #[intrinsic("nabi.add.i32")]
    fn add(self, rhs: i32) -> i32 {
        // body: runs when taken out as an fn value or in environments that do not know the intrinsic
        self + rhs                    // primitive operators lower directly to machine operations
    }
}
```

規則整理如下。在啟用內建函式的一般編譯途徑上，主體會被忽略；在不認得該內建函式的實驗性後端或直譯環境中，則實際使用主體。若主體為空，就視為 stub `panic("intrinsic not available")`。

當方法被取出作為 `fn` 值（`let f = i32::add;`）時，呼叫該值會執行主體。內建函式只在直接呼叫處生效，因此內建方法的主體也必須是意義相同的正確實作。在主體內，對基本型別寫 `self + rhs` 本身就是對此內建函式的直接呼叫，因此會降階為機器運算，而不會再回到主體。

內建函式也可以是沒有主體的宣告（`pub fn size_of<T>() -> usize;`）。它沒有後備：內建函式就是它唯一的實作，而且只用於直接呼叫。

## 0.3.5. 使用者程式碼中的內建函式

`#[intrinsic(...)]` 是只在標準函式庫與編譯器耦合的界線上運作的機制，因此使用者程式碼不能直接附加此屬性：無論是函式、泛型函式或方法都一樣。

```nabi
// user code: rejected.
#[intrinsic("nabi.add.i32")]
fn my_add(a: i32, b: i32) -> i32 { ... }
// error[E0115]: `#[intrinsic]` on `my_add`: the intrinsic attribute is for the standard library only
```

此限制避免開出一道逃生門，讓人能從標準函式庫 trait 之外任意呼叫編譯器後端。

## 0.3.6. 與其他機制的比較

內建函式只是連接編譯器與標準函式庫的幾種魔法之一；下表整理各自的角色。

| 機制                  | 用途                                                             | 規格                                                                   |
|-----------------------|------------------------------------------------------------------|------------------------------------------------------------------------|
| `#[intrinsic("...")]` | 標準函式庫方法 ↔ 編譯器後端 PrimOp                               | 本文件                                                                 |
| `#[lang(...)]`        | 標準函式庫項目 ↔ 編譯器槽位（none、iter、index 等）              | [refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)     |
| `#[infix(...)]`       | 語彙單元 ↔ 標準函式庫 trait 方法                                 | [refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md) |
| `#[literal(...)]`     | 字面值形式 ↔ 標準函式庫處理器                                    | [refs-lang 6.5](../../refs-lang/006-meta-system/605-literal-magic.md)  |

這四種魔法彼此正交。`#[intrinsic]` 負責與編譯器後端的耦合，其他三種則負責與語言語法的耦合。
