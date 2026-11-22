# 2.5. `union`

`union` 是不帶標籤的容器，其欄位共用同一塊記憶體。語言不追蹤任何時刻哪個欄位的值是有意義的；那是使用者的責任。

```
union_decl      ::= "union" identifier generics? where_clause? "{" named_field_list? "}"
```

## 2.5.1. 基本形式

最簡單的 `union` 宣告如下：

```nabi
union EpollData {
    ptr: usize;
    fd: i32;
    val: u64;
}
```

主體與欄位語法和 [`struct`](203-struct.md) 的具名 `struct` 相同。差異在於關鍵字 `union` 與語意：所有欄位共用同一個記憶體位置。

### 泛型 union

`union` 可以帶型別參數（[2.10](210-generics.md)），並和 `struct` 一樣有約束、`where` 子句與預設值。每個實例都是有自己配置的獨立型別：其大小與對齊依該組引數的欄位型別而定。

```nabi
union Bits<T> { v: T; raw: u64; }
union Small<T = u16> { v: T; raw: u8; }

let a = Bits::<f64> { raw: 0x3FF0000000000000u64 };
let b = Bits { v: 7i32 };              // Bits<i32>, inferred from the field given
let x = unsafe { a.v };                // 1.0 (the bits of 1.0f64)
```

在 union 字面值中，引數由給出的那一個欄位推論。約束依實例檢查，診斷會指出該實例（``reading field `v` of union `Bits<i32>` needs an `unsafe` block``）。`union` 至少宣告一個欄位。

## 2.5.2. 語意

下面的範例展示 `union` 的行為。

```nabi
let x = EpollData { fd: 5i32 };
x.fd = 7i32;                    // write: safe
let y = unsafe { x.val };       // read: requires unsafe; reinterprets x's bit pattern as u64
```

以某個欄位建立實例，會把該欄位的位元寫入記憶體。之後透過另一個欄位讀取，會把相同的記憶體位元重新解讀為該欄位的型別。寫入欄位是安全的，但讀取欄位只允許在 [`unsafe`](../004-functions/415-unsafe.md#4152-unsafe------block-expression) 內進行，因為語言不保證讀出的位元是該型別的有效值（例如 `bool`、`char`、指標）。

## 2.5.3. 安全性在哪裡

`union` 的位元重新解讀是語言定義的行為，但結果的意義是程式領域的責任。例如，把 `1.0` 寫入 `f32` 欄位再讀取 `i32` 欄位，會得到 `1065353216`（`0x3F800000`）；反過來，把 `i32` 位元當作 `f32` 讀取，會得到由 IEEE 754 位元表示給出的實數值。語言不檢查這個值對領域而言是否有意義。

這個模型與 C 的 `union` 不同。在 C 中，從不同的成員讀取是未定義行為；在 Nabi 中，它被定義為位元重新解讀。由於結果不保證是有效值，讀取被標示為 `unsafe`。

## 2.5.4. 預設配置交給編譯器

`union` 的預設配置交給編譯器。它只需滿足最大欄位的大小與對齊；確切的欄位對齊與填充由編譯器決定。

若需要與 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 相容的 `union`，請加上 `#[repr(C)]`。

```nabi
#[repr(C)]
union CUnion {
    a: i32;
    b: f32;
}
```

帶有 `#[repr(C)]` 的 `union` 與 C 的 `union` 位元相容，用於 [FFI](../000-meta/007-glossary.md#ffi-foreign-function-interface) 介面。確切的意義定義於[內建屬性](../006-meta-system/602-built-in-attributes.md)。

## 2.5.5. `union` 與 `#[repr(C)] union`

兩種形式的差異如下。

| 形式                         | 用於        | 配置              | 位元相容性                |
|------------------------------|-------------|-------------------|---------------------------|
| `union T { ... }`            | 一般程式碼  | 交給編譯器        | 只在 Nabi 內保證          |
| `#[repr(C)] union T { ... }` | FFI         | 與 C ABI 相同     | 與 C 程式碼相容           |

## 2.5.6. 可見性 / 重複定義

適用與 [`struct`](203-struct.md) 相同的規則。

> [!WARNING]
> 讀取 `union` 欄位是位元重新解讀，只允許在 `unsafe` 內進行（寫入是安全的）。讀出的位元是否為該型別的有效值，以及它對領域是否有意義，是作者的責任。

## 2.5.7. `union` 與和型別

`union` 與和型別看起來可能相似，但它們是完全不同的工具。

```nabi
union  Foo { a: i32; b: f32; }    // same memory, no tag
type   Bar = i32 | f32;            // separate memory, tagged
```

各面向的差異：

| 面向                           | `union`                          | 和型別（`A \| B`）             |
|--------------------------------|----------------------------------|-------------------------------|
| 追蹤哪個欄位是作用中的         | 否                               | 由標籤追蹤                    |
| 欄位之間的轉換                 | 位元重新解讀                     | 變體改變（值被重建）          |
| 記憶體用量                     | 最大欄位的大小                   | 標籤 + 最大變體的大小         |
| 讀取欄位                       | 需要 `unsafe`（位元重新解讀）    | 安全，透過模式 / `is`         |

兩者的用途完全不同：`union` 在位元層級提供不同的解讀，和型別則在值層級表達不同的種類。不得將兩者混淆。
