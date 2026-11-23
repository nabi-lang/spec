# 1.2. 強制轉型操作：明確命名的強制轉型

指標與型別之間的轉換只以**具名函式**存在。沒有 `cast` 模組：`ptr_cast`、`align_cast`、`align_cast_unchecked` 和 `const_cast` 定義在 `bare::ptr`，`reinterpret` 和 `transmute_unchecked` 定義在 `bare::mem`，而 `bare` 根模組重新匯出這六個（`bare::ptr_cast`）以及 `Pod`（[1.2.5](#125-reinterpreta-pod-b-pod位元強制轉型安全)）。C 和 Rust 中的 `as` 強制轉型之所以如此危險，是因為轉換會在不知不覺中發生；為了避開這個陷阱，Nabi 要求每一次強制轉型都帶有**明確的名稱**。

## 1.2.1. 4 個強制轉型函式

四個基本函式如下：

| 函式                                     | 意義                                       | 風險                    |
|------------------------------------------|--------------------------------------------|-------------------------|
| `ptr_cast<U>(p: Ptr<T>)`                 | 只變更型別（保留來源 / 位址） | 對齊由呼叫端負責 |
| `align_cast<T>(p: Ptr<u8>)`              | 對齊向上轉型                               | 有檢查（未對齊時為 `none`） |
| `const_cast<T>(p: ConstPtr<T>)`          | 移除 const                                 | 非常高（由呼叫端保證） |
| `reinterpret<A: Pod, B: Pod>(a: A) -> B` | 位元層級的強制轉型（僅限 `Pod`）          | 低（由 Pod 標記保證） |

此外，還有一個在需要更深層繞過時使用的逃生門。

| 函式                                  | 意義                                            | 風險    |
|---------------------------------------|-------------------------------------------------|---------|
| `transmute_unchecked<A,B>(a: A) -> B` | **不做 Pod 檢查**的位元強制轉型（由呼叫端負責） | 最高 |

編譯器拒絕對指標使用 `as` 關鍵字，因此使用者只能使用具名的強制轉型。

## 1.2.2. `ptr_cast<U>`：指標型別轉換

```nabi
fn ptr_cast<U, T>(p: Ptr<T>) -> Ptr<U>;
```

這個函式完整保留位址與來源，只變更型別標記。**因此對齊由呼叫端負責**。它只接受 `Ptr`（Nabi 沒有多載）；`ConstPtr` 和 `ManyPtr` 以各自的 `cast` 方法用同樣方式變更型別（[1.1](101-ptr.md)）：`p.cast::<U>()`。

```nabi
let p: Ptr<u32> = ...;        // 4-byte aligned
let q: Ptr<u8> = bare::ptr_cast::<u8>(p);    // safe (down-align)
let r: Ptr<u64> = bare::ptr_cast::<u64>(p);  // dangerous: u64 needs 8-byte
                                                 // r.read() may SIGBUS (ARM)
```

在任何建置中都不會插入對齊檢查：透過未對齊的結果讀寫是 UB。請先用 `align_cast`（1.2.3）或 `is_aligned`（1.1.2）檢查。

### 安全的強制轉型：型別等價

佈局完全相同的兩個結構之間的轉換是安全的。

```nabi
struct A { x: u32 }
struct B { x: u32 }       // same layout as A

let pa: Ptr<A> = ...;
let pb: Ptr<B> = bare::ptr_cast::<B>(pa);    // safe if layout matches
```

不過，編譯器不會驗證佈局是否真的相符；這項保證必須由呼叫端承擔。

## 1.2.3. `align_cast<T>`：對齊檢查

```nabi
fn align_cast<T>(p: Ptr<u8>) -> Ptr<T> | none;
```

把 `Ptr<u8>`（1 位元組對齊）轉成對齊要求更大的 `Ptr<T>` 時，**只有在驗證實際對齊之後**才會進行轉換。

```nabi
let bytes: ManyPtr<u8> = ...;
let p: Ptr<u8> = bytes.add(offset).as_one();

if let aligned: Ptr<u32> = bare::align_cast::<u32>(p) {
    let v = unsafe { aligned.read() };       // safe (verified)
} else {
    panic("not 4-byte aligned");
}
```

`align_cast` 回傳 `T | none` 而不是 `Result`，因為兩者之中前者較輕量。

### 不檢查的變體

```nabi
fn align_cast_unchecked<T>(p: Ptr<u8>) -> Ptr<T>;     // without checking
```

只在呼叫之前已經驗證過對齊的地方使用它。這個變體即使在偵錯建置中也不做任何檢查。

## 1.2.4. `const_cast<T>`：移除 const

```nabi
fn const_cast<T>(p: ConstPtr<T>) -> Ptr<T>;
```

這個函式去除 const。其危險顯而易見：如果原始資料位於唯讀區域（永生常數），寫入那裡會立即 segfault。

```nabi
use bare::ptr::{Ptr, ConstPtr, const_cast};

let s: str = "hello";              // immortal const
let p: ConstPtr<u8> = s.as_ptr();
let m: Ptr<u8> = const_cast(p);
unsafe { m.write(72u8) };          // 'H': SEGFAULT (write to immortal const)
```

它與 C 的 `const_cast<T*>` 完全同構。其用途限於在需要可變性的地方暫時包裝唯讀 API；一般程式碼**不得使用它**。

## 1.2.5. `reinterpret<A: Pod, B: Pod>`：位元強制轉型（安全）

```nabi
unsafe trait Pod { /* marker: Plain Old Data */ }

fn reinterpret<A: Pod, B: Pod>(a: A) -> B;
```

對帶有 `Pod`（Plain Old Data）標記的型別，保證**安全的 transmute**，理由如下：

- `Pod` 型別的每一種位元模式都有效，因此永遠不會暴露未初始化的內容。
- 只允許在相同大小之間轉換。`where size_of` 約束沒有編譯期形式，因此大小條件在 `reinterpret` 執行時檢查（不相符時觸發 panic）。
- 它是值的轉換，因此對齊自然得到保證。

```nabi
let f: f32 = 3.14;
let bits: u32 = bare::reinterpret(f);     // OK: both Pod, 4 bytes
let f2: f32 = bare::reinterpret(bits);    // round-trip OK
```

### 自動衍生 `Pod`

標準函式庫自動為下列型別提供 `Pod`：

```nabi
unsafe impl Pod for i8, i16, i32, i64, u8, u16, u32, u64, usize, isize { }
unsafe impl Pod for f32, f64 { }
unsafe impl Pod for c_char { }
unsafe impl<T: Pod, const N: usize> Pod for [T; N] { }
unsafe impl<A: Pod> Pod for (A,) { }
unsafe impl<A: Pod, B: Pod> Pod for (A, B) { }
unsafe impl<A: Pod, B: Pod, C: Pod> Pod for (A, B, C) { }
unsafe impl<A: Pod, B: Pod, C: Pod, D: Pod> Pod for (A, B, C, D) { }
```

`bool` 和 `char` 不是 `Pod`：它們的大多數位元模式都不是值（`bool` 是 0 或 1，`char` 是 Unicode 純量）。請重新解讀為 `u8` / `u32`，再以帶檢查的方式轉換（`b != 0u8`、`char::from_u32`）。

`Pod` 是 `unsafe trait`（[4.15.4](../../refs-lang/004-functions/415-unsafe.md#4154-在-trait-中)）：它的實作承諾每一種位元模式都是值，而編譯器無法檢查這一點，因此手寫的實作是 `unsafe impl Pod for T {}`。`#[derive(Pod)]` 會寫出它。

下列型別不會自動取得它：

- `Ptr<T>` 帶有另外的來源資訊，無法只用位元表示。
- 使用者結構必須用 `#[derive(Pod)]` 明確選擇加入。

```nabi
#[derive(Pod)]
#[repr(C)]
struct Vec3 { x: f32; y: f32; z: f32; }

let v: Vec3 = Vec3 { x: 1.0, y: 2.0, z: 3.0 };
let bytes: [u8; 12] = bare::reinterpret(v);
let v2: Vec3 = bare::reinterpret(bytes);
```

這個設計沿用 [Project Safe Transmute（Rust RFC 2835）](https://rust-lang.github.io/rfcs/2835-project-safe-transmute.html) 模型與 [bytemuck crate](https://docs.rs/bytemuck) 的模式。

## 1.2.6. `transmute_unchecked<A, B>`：逃生門

```nabi
unsafe fn transmute_unchecked<A, B>(a: A) -> B;   // no size check either
```

這個函式原樣搬移位元，**連** `Pod` 檢查**都不做**。如名稱所示，它是最危險的強制轉型，因此是 `unsafe fn`：暴露未初始化內容、大小不符、對齊問題以及其中的 GC 參考，全都由呼叫端負責。無法配置暫存區塊時，它會以 `mem::transmute_unchecked: out of memory` 觸發 panic。大小不符不會被拒絕；值會經過一個與較大型別等大、已清零的暫存區塊，因此超出來源大小的位元組讀起來為零。

它的用途限於標準函式庫內部的**最底層**，只用於無法以其他任何工具表達的情況，例如撰寫 `Pod` 實作本身。一般使用者程式碼絕對不得使用它。

```nabi
// example inside the standard library
unsafe impl Pod for SomeNewtype {
    // empty marker
}

let s: SomeNewtype = ...;
let bits: u64 = unsafe { bare::transmute_unchecked(s) };   // code where SomeNewtype guarantees size 8
```

## 1.2.7. `as` 強制轉型適用的地方

`as` 是數值強制轉型（[refs-lang 2.14.2](../../refs-lang/002-types/214-cast-and-conversion.md#2142-as-強制轉型)）：它允許在整數型別、浮點數型別、`bool`、`char` 和 `c_char` 之中的任意兩者之間使用，遵守 refs-lang 所列的目標限制，結果與 `.wrapping_into()` 相同。有損失的配對會環繞（整數），或向零截斷並飽和（浮點數轉整數）。只有指標會被拒絕。

| 運算式             | 結果                                                   |
|--------------------|--------------------------------------------------------|
| `i32 as i64`       | OK（無損加寬）                                 |
| `i64 as i32`       | OK，環繞（要檢查或飽和請用 `try_into` / `saturating_into`） |
| `f32 as f64`       | OK（無損）                                          |
| `i32 as f32`       | OK，捨入到最接近的 `f32`                        |
| `u32 as i32`       | OK，位元相同（`0xFFFF_FFFF` 是 `-1`）              |
| `Ptr<T> as Ptr<U>` | **編譯錯誤**：`bare::ptr_cast::<U>`           |
| `Ptr<T> as usize`  | **編譯錯誤**：`.addr()` 或 `expose_provenance`    |
| `usize as Ptr<T>`  | **編譯錯誤**：`from_exposed_provenance`           |

完整的轉換矩陣請見 [2.11 conversion](../200-core/211-conversion.md)。

## 1.2.8. 政策摘要

本章的強制轉型政策歸結為四點。

- `as` 強制轉型只用於數值；指標永遠不經過 `as`，非數值的位元重新解讀則經過 `reinterpret`。
- `bare::ptr_cast` / `align_cast` / `const_cast` / `reinterpret` 這四個函式是明確強制轉型的通道。
- `Pod` 標記 trait 提供位元重新解讀的安全條件。
- `transmute_unchecked` 只作為逃生門公開。

藉由這個政策，Nabi 同時避開了 C 繁瑣的型別轉換與 Rust 在指標上的 `as` 陷阱。
