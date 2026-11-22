# 8.8. 降階：資料型別與運算式

本節定義 Nabi 的值、型別與運算式在 `nabic` 產生的程式碼中如何表示。配置以 Nabi 型別寫出：`(A, B)` 是位於其自然偏移的一對欄位，`[T; n]` 是陣列，`ptr` 是指標寬度的位址，`p` 是以位元組計的指標寬度。程式碼是虛擬碼（[8.7.1](807-code-generation.md#871-translation-flow)）。

## 8.8.1. 基本的整數 / 浮點數

基本數值型別是其寬度的機器值。

| Nabi 型別         | 表示方式                                                                |
|-------------------|-------------------------------------------------------------------------|
| `i8` ~ `i64`      | 1 ~ 8 位元組，二補數                                                    |
| `u8` ~ `u64`      | 1 ~ 8 位元組；由運算決定是否有號                                        |
| `usize` / `isize` | 目標的指標寬度                                                          |
| `f32` / `f64`     | IEEE 754 binary32 / binary64                                            |
| `bool`            | 一個位元組，`0` 或 `1`                                                  |
| `char`            | 4 位元組（Unicode 純量值）                                              |
| `c_char`          | 一個位元組                                                              |

`str` 是以值傳遞的不可變組合 `(ptr, usize)`（一個指標與位元組長度）。字面值指向永不消失的常數；在執行時建立的字串指向某個 GC 物件（型別 id `1`，[8.2.12](802-scan-table.md#8212-runtime-reserved-type-ids)）的酬載。

- `s[a:b]` 是 `(ptr + a, b - a)`，並共用原本的位元組（O(1)）。
- `ptr` 可以指向 GC 物件的任何位元組（包括結尾的 NUL），並讓整個物件保持存活。GC 把它當作內部指標處理，疏散會保留偏移地改寫它。
- 字面值以及執行環境新建立的字串在其位元組之後有一個 NUL。切片則可能沒有。

## 8.8.2. 算術運算

在基本型別上，[運算子分派](../003-expressions/301-operator-dispatch.md)的運算子是運算元寬度的單一指令：加、減、乘、依運算元型別的有號或無號除法與餘數，以及有號、無號或有序的浮點數比較。

```
a + b       ->  add.i32 a, b            ; with overflow checks: add, then branch to the panic on overflow
a / b       ->  div.s32 a, b            ; after the checks for b == 0 and MIN / -1
a < b       ->  lt.s32 a, b             ; lt.u32 for unsigned, lt.f64 (ordered) for floats
```

受檢的運算在結果放不下時分支到其 panic（`-C overflow-checks`，[9.1.2](../009-tools/901-nabic-cli.md#nabic-build-aot-build)）；最佳化建置會移除它能證明不會失敗的檢查（[8.7.3](807-code-generation.md#873-optimization-levels)）。使用者定義的 trait 實作在泛型單態化之後是函式呼叫。

## 8.8.3. `struct` 的表示方式

`struct` 是值型別（[2.3.8](../002-types/203-struct.md#238-value-semantics)），沒有物件標頭。

```nabi
struct Point { x: f64; y: f64; }
```

其表示方式就是以 [2.3.6](../002-types/203-struct.md#236-memory-layout) 之配置排列的各欄位，包括 `#[repr]` 的對齊與偏移。

```
(f64, f64)        ; x at 0, y at 8; size 16, align 8
```

具有相同欄位的兩個結構有相同的配置。結構值存在於暫存器、堆疊槽位中，或以行內方式存在於其容器中：另一個結構的欄位、`List` 緩衝區的元素、閉包環境或協程框架的槽位。指派與引數傳遞會複製各欄位。只有在 box 無法被觀察到之處，編譯器才可以在內部把值裝箱。

### 欄位存取

`p.x` 讀取位於其偏移的欄位：從結構的槽位或行內位置載入，或是容納該欄位的暫存器。

```
val = load.f64 [p + 0]
```

### 欄位寫入

`p.x = 1.0f64` 更新變數自己的複本：存入其槽位中該欄位的偏移。

```
store.f64 [p + 0], 1.0
```

當結構以行內方式存放在 GC 堆積物件中，且該欄位容納 GC 參照時，存入會經過寫入屏障（[屏障與根](../007-runtime/704-barriers-and-roots.md)）。堆疊槽位中之結構的指標欄位是 GC 根（[7.4](../007-runtime/704-barriers-and-roots.md)）。

## 8.8.4. 元組 `struct` 的表示方式

考慮元組 `struct`：

```nabi
struct Pair(i32, i32);
```

它的表示方式與一般的 `struct` 相同，只是欄位以位置索引而非名稱存取。

## 8.8.5. 元組型別

元組型別

```nabi
(i32, str, bool)
```

配置為 `(i32, (ptr, usize), bool)`：`str` 無論出現在哪裡都是其組合（8.8.1）。和 `struct` 一樣，它是沒有標頭的值。

## 8.8.6. `enum` 的表示方式

[`enum`](../002-types/204-enum.md) 中的 `enum` 是沒有物件標頭的值，表示為帶標籤的 `union`：與和型別（8.8.8）相同的 `(i32 tag, payload)` 配置。

```nabi
enum Result {
    Ok(i32),
    Err(str),
    Pending,
}
```

它表示為：

```
(i32, [u64; 2])    ; 64-bit target: tag at 0, payload at 8 (16 bytes, 8-aligned)
```

標籤容納變體的索引（`Ok`=0、`Err`=1、`Pending`=2）。酬載是帶資料之變體的 union：大小與對齊都與其中最大者相同，每個變體的酬載以其自己的配置存放在其開頭。可能容納 GC 參照的酬載是一串指標寬度的字組，由回收器依標籤掃描（[8.2](802-scan-table.md)，`SUM_PAYLOAD`）。

沒有欄位的列舉就只有其標籤：一個 `i32`，或其 `#[repr(..)]` 的整數。

### 建構變體

`Result::Ok(42)` 在暫存器中或目的地的槽位中建立該值：不進行配置。

```
store.i32 [v + 0], 0        ; tag: Ok
store.i32 [v + 8], 42       ; the payload in its own type
```

從較短變體之酬載建立的 union 值，在該酬載之後的位元組中容納零，因此從它複製出的根字組是 null 或參照，絕不會是殘留的堆疊內容（[8.9.12](809-lowering-control.md#8912-shadow-stack-frame)）。

### 比對變體

比對變體：

```nabi
match v {
    Ok(n) => ...,
    Err(s) => ...,
    Pending => ...,
}
```

依分支順序把標籤與每個分支之變體的索引比較（8.9.3）；沒有任何分支相符的值會到達一個無法到達的區塊。

```
bb0:
    tag = v.0
    if tag == 0 then bb_ok else bb1
bb1:
    if tag == 1 then bb_err else bb2
bb2:
    if tag == 2 then bb_pending else bb_unreachable
bb_ok:
    n = v.1 as i32          ; the payload read as the variant's type
```

## 8.8.7. （不帶標籤之）`union` 的表示方式

考慮 [`union`](../002-types/205-union.md) 中的 `union`。

```nabi
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

它的大小與對齊都與其最大的欄位相同（這裡是 8 位元組），每個欄位都從偏移 0 開始。欄位存取以該欄位的型別讀寫同一塊記憶體。在原始碼中，欄位讀取必須位於 `unsafe` 內（[2.5](../002-types/205-union.md)）。

## 8.8.8. 和型別（`A | B`）的表示方式

[和型別](../002-types/206-sum-types.md)中的和型別處理如下。

```nabi
type X = i32 | f64
```

這表示為 `(i32, u64)`，即一個標籤與一個酬載槽位，建立方式與 `enum` 相同（8.8.6）：這裡的酬載容納 `i32` 或 `f64`。`none` 成員是標籤 0，不帶酬載，因此清零的記憶體讀回來是 `none`。

若適用 niche 最佳化（[niche 最佳化](../002-types/207-niche-optimization.md)），標籤槽位就會消失，只剩下酬載的位元模式。

```nabi
Ptr<u8> | none
```

這個型別是單一指標：以 null 表示 `none` 的自動 niche。

## 8.8.9. `any` 槽位的表示方式

考慮 [`any` 槽位](../002-types/202-any-slot.md)中的動態槽位。

```nabi
let x: any = ...
```

它是一個 `(u32 tag, payload)` 組合。`any Tr` 與單獨的 `any` 有相同的表示方式。

```
(u32, ptr)
```

標籤的低 31 位元識別值的型別：型別名稱的 31 位元 FNV-1a 雜湊，因此分開編譯的套件會一致（一個建置中的衝突會取下一個空閒的值）。不比指標寬的數字、`bool` 與 `char` 值直接以其位元存放在酬載字組中。其他的值是 GC 堆積上的 box，酬載就是該 box，並設定標籤的位元 31（`ANY_BOXED`）；GC 在掃描 `any` 槽位時讀取該位元（[8.2.7](802-scan-table.md)）。

### `any` 上的型別測試

對 `any` 值的型別模式

```nabi
match v {
    n: i32 => n + 1,
    s: str => s.len() as i32,
    _ => 0,
}
```

會遮掉 `ANY_BOXED`，並把標籤與每個模式型別的標籤比較，形成一串雙向分支：

```
bb0:
    t = v.0 & 0x7fffffff
    if t == TAG_I32 then bb_i32 else bb1
bb_i32:
    n = v.1 as i32          ; the payload word's low 32 bits
bb1:
    if t == TAG_STR then bb_str else bb_other
bb_str:
    s = load.str [v.1 + 16] ; a boxed `str`: the pair from the box payload
```

這些執行時的測試就是 `any` 槽位的成本。

## 8.8.10. 陣列型別：`[T; N]`

考慮[型別運算式](../002-types/201-type-expressions.md)中的固定長度陣列。

```nabi
[i32; 8]
```

它是連續的 8 個 `i32` 元素，32 位元組。在 GC 物件內，16 位元組的標頭位於陣列之前。

## 8.8.11. 函式型別：`fn(...) -> ...`

考慮[函式型別](../004-functions/404-function-type.md)。

```nabi
fn(i32, i32) -> i32
```

每個函式值，無論是具名函式還是閉包，都是一個 `{code, env}` 組合：

```
(ptr, ptr)        ; code, env
```

`code` 以 `env` 作為第一個引數，然後是宣告的引數，沒有 TLS 位址（[8.6.11](806-calling-convention.md#8611-function-pointer-abi)）。作為值使用的具名函式有 null 的 `env`，以及指向捨棄它之 thunk 的程式碼指標。只有 C 函式型別（`extern "C" fn(..)`）是單純的程式碼指標。

## 8.8.12. 閉包的表示方式

考慮擷取變數的[閉包](../004-functions/405-closure.md)。

```nabi
let n = 10;
let f = |x| x + n;       // captures n
```

它是一個函式值（8.8.11），其 `env` 是容納各擷取的 GC 物件，每個擷取一個欄位，排列在 16 位元組標頭之後的結構中：

```
env object: header (16 bytes), then (i32)     ; a copy of n
f = (code of the closure body, env)

; on call
r = call f.0(f.1, x)
```

不擷取任何東西的閉包有 null 的 `env`，不配置任何東西。

## 8.8.13. 識別字 / 變數存取

考慮[變數宣告](../005-modules/501-variable-declaration.md)中的區域變數。

```nabi
let counter = 0i32;
counter = counter + 1i32;
```

每個區域變數都有一個堆疊槽位，而指派就是對它的存入：

```
store.i32 [counter], 0
c = load.i32 [counter]
c2 = add.i32 c, 1
store.i32 [counter], c2
```

在 `-O0`，每個值都留在其槽位中。在 `-O0` 以上，從未被取址、從未被整體複製、也沒有向偵錯工具描述的區域變數，可以改為存在暫存器中，而最佳化器會把已知的值轉送給它的讀取（[8.7.3](807-code-generation.md#873-optimization-levels)）。容納 GC 參照的區域變數一律保有其槽位：影子堆疊框架容納該槽位的位址（8.9.12）。

## 8.8.14. 字面值的去糖

考慮 [`#[literal]` 魔法](../006-meta-system/605-literal-magic.md)中的集合字面值。

```nabi
[1, 2, 3]
```

它去糖為對標準函式庫函式 `list_literal(1, 2, 3)` 的呼叫，並遵循一般函式呼叫的降階。

字串與數值字面值被當作存放在 .rodata 中的編譯期 `const` 處理，然後作為指標或值使用。
