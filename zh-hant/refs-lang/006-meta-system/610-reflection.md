# 6.10. 反射

標準函式庫提供兩個層級的反射 trait：`Reflect` 與 `Inspect`。

```
Reflect    automatic: shallow metadata every type carries for free
Inspect    opt-in via #[derive(Inspect)]: rich field/method description
```

分成兩個層級，是為了在控制二進位檔大小的同時，仍然免費給予每個型別 `any` 槽位、和型別與 REPL 自動印出所必需的東西。

## 6.10.1. `Reflect`：自動實作的 trait

`Reflect` 是自動附加在每個型別上的最小型別內省。使用者不需要寫 `#[derive(Reflect)]`，編譯器就會為每個型別合成一個 impl。

```nabi
trait Reflect {
    // static: called directly on the type
    fn type_name() -> str;
    fn size() -> usize;
    fn align() -> usize;

    // instance: runtime dispatch on any slots / sum types
    fn type_name_of_val(self) -> str;
    fn size_of_val(self) -> usize;
}
```

```nabi
i32::type_name() == "i32"
i32::size()      == 4
i32::align()     == 4

let x: any = 42i32;
x.type_name_of_val() == "i32"    // runtime type tag check

let v: i32 | f64 = pick();
v.type_name_of_val()              // "i32" or "f64" (sum type variant check)
```

由於是自動的，它不需要使用者程式碼。編譯器本來就知道每個型別的 `type_name`、`size` 與 `align`，並以零成本合成它們。

`Reflect` 由編譯器為每個型別提供；原始碼中沒有 impl。

- 手寫的 `impl Reflect for T` 是錯誤。約束 `T: Reflect` 一律成立。
- 若型別有自己的同名方法，例如 `size()` / `type_name()`，該方法會隱藏反射方法。
- 名稱（`type_name`）不帶模組路徑（`List<i64>`、`Config`）。和型別的靜態名稱依標準順序（[2.6.4](../002-types/206-sum-types.md#264-集合語意)）寫出（`f64 | i32`，`none` 在最後）。`type_name_of_val(self)` 之類的實例形式，回報執行時所容納的成員（對和型別而言）或 `any` 的內容。
- `core::reflect::cast<T>(value: any) -> T | none` 依與 `set_field` 相同的規則，把 `any` 取出為 `T`。型別的方法與欄位是 `T::methods()` / `T::fields()`（[6.10.2](#6102-inspect選用的-trait)）；沒有取得它們的自由函式。

## 6.10.2. `Inspect`：選用的 trait

`Inspect` 是以 `#[derive(Inspect)]` 選用的豐富反射。它公開型別的描述（`TypeInfo`，6.10.7）、欄位名稱、動態的欄位存取與修改，以及方法描述。

```nabi
trait Inspect: Reflect {
    fn type_info() -> TypeInfo;                             // the description (6.10.7)
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // private fields too
    fn set_field(ref self, name: str, value: any) -> bool;  // whether it succeeded

    // Every method has a default body over `type_info()`.
    fn fields() -> List<FieldInfo>;          // the field table of Self
    fn methods() -> List<MethodInfo>;        // the method table of Self
    fn variant_names() -> List<str>;         // empty for a struct
    fn variant_name(self) -> str;            // a struct: self.type_name_of_val()
    fn field_names_of_val(self) -> List<str>; // the fields self holds
}

struct FieldInfo {
    name: str;                 // "0", "1", .. for a tuple field
    type_name: str;            // instantiated: f64 for `v: T` in Box<f64>
    is_pub: bool;              // true only for a plain `pub` field
}

struct MethodInfo {
    name: str;
    arity: usize;              // number of arguments excluding self
    is_static: bool;           // a method that does not take self
    arg_types: List<str>;      // argument type names excluding self
    return_type: str;
}
```

- `get_field` 以 `any` 回傳欄位的值，名稱不存在時回傳 `none`。私有欄位也可以讀取。
- 當值的型別與欄位**完全相同**，或不同整數型別的整數**能無損地放進**時（例如以 `u.set_field("id", 42)` 設定 `u64` 欄位），`set_field` 會指派並回傳 `true`。否則它不指派並回傳 `false`。沒有其他轉換（例如 `i32` → `f64`）。
- `methods()` 只回傳型別的**固有方法**（不含 trait 方法），依名稱排序。泛型 `impl` 的方法依寫法顯示型別（`T`）。
- `fields()` 依宣告順序列出每個欄位及其型別名稱與是否為 `pub`。**私有欄位也會列出**：衍生 `Inspect` 就表示型別作者同意公開它們，與 `get_field` 相同。型別名稱是實例化後的名稱（`Box<f64>` 回報 `v: f64`、`items: List<f64>`），不帶模組路徑，與 `type_name` 的寫法相同。
- `enum` 在型別層級沒有欄位（`field_names()` 與 `fields()` 是空串列）。`variant_names()` 依宣告順序列出其變體。在值上，`variant_name()` 是它所容納的變體（`"Circle"`），`field_names_of_val()` 是該變體的欄位（元組變體為 `"0"`、`"1"`，具名變體為其名稱），而 `get_field` 讀取它們。變體的酬載透過 `Inspect` 是唯讀的：對列舉使用 `set_field` 是 `false`。
- 在結構上，`variant_names()` 是空串列，`variant_name()` 是其型別名稱（`type_name_of_val()`），`field_names_of_val()` 是其欄位。
- 每個方法都有預設主體；`get_field`、`set_field` 與名稱清單讀取 `type_info()`。derive 只寫出 `type_info()` 所呼叫的存取函式（6.10.7）。手寫的 `impl Inspect` 可以覆寫任何方法。
- 對沒有該 derive 的型別呼叫 `Inspect` 方法是錯誤，其說明會指出應寫什麼：``add `#[derive(Inspect)]` to `Point` ``。

用法範例：

```nabi
#[derive(Inspect)]
struct Config { host: str; port: i32; }

let c = Config { host: "localhost", port: 8080 };
Config::field_names()              // ["host", "port"]
c.get_field("port")                // 8080 inside an any
c.set_field("port", 9090)          // true (success)
c.set_field("xxx", 1)              // false (no such field)
Config::fields()[1].type_name      // "i32"
```

`Inspect` 之所以是選用的，是因為其中繼資料必須儲存在二進位檔中。自動把它附加在每個型別上，會帶來過大的二進位檔大小成本。即使有了 derive，在程式要求之前也不會儲存任何東西：從不呼叫 `type_info()`、`info::<T>()` 或建立在它們之上之方法的程式，建置出的程式碼與沒有該 derive 的程式相同。

`#[derive(Inspect)]` 由 `core::reflect` 中的[元區塊](608-meta-blocks.md) `#[impl_derive(Inspect)]` 實作，這是元系統本身實作標準函式庫 trait 的一種模式（[stdlib 2.12.3](../../refs-standard-libraries/200-core/212-reflect.md#2123-inspect選擇性加入的欄位--方法)）。

## 6.10.3. REPL / Jupyter 的自動印出

當 REPL 印出單獨的最後一個運算式時，它使用 `Display`，否則使用 `Debug`；結尾的 `?` 使用 `Debug`；`:type` 印出運算式的靜態型別，由型別檢查得出而不執行它（[9.7.7](../009-tools/907-jit-cells.md#977-視覺化--輸出)）。

```
>>> let x = 42i32;
>>> x
42

>>> [1, 2, 3]
[1, 2, 3]

>>> Config { host: "h", port: 80 }
Config { host: "h", port: 80 }

>>> Opaque { id: 1 }
Opaque
```

兩者都沒有的值（對結尾的 `?` 而言是沒有 `Debug`）只印出其型別名稱。REPL 與 Jupyter 儲存格工作階段使用這種行為（[9.7](../009-tools/907-jit-cells.md)）。

## 6.10.4. 沒有全域的 `typeof` / `sizeof` 函式

其他語言透過 `typeof` 或 `sizeof` 等關鍵字公開的內省，在 Nabi 中只以 trait 方法提供。

```nabi
// nabi: method calls
T::size()
x.type_name_of_val()

// other languages: keywords
sizeof(T)        // C / C++
typeof(x)        // JS / TS
```

理由是這樣不會增加任何語言語法。單一的 `Reflect` trait 就涵蓋了整個內省介面。

## 6.10.5. `any` 槽位與和型別上的分派

`any` 槽位與 `T1 | T2` 形式的和型別在執行時才決定實際型別，因此 `type_name_of_val` 呼叫很重要。

```nabi
fn handle(x: any) {
    if x.type_name_of_val() == "i32" { handle_int(x); }
    else if x.type_name_of_val() == "str" { handle_str(x); }
}

fn classify(v: i32 | f64) -> str {
    match v {
        n: i32 => "integer",
        f: f64 => "float",
    }
}
```

編譯器把型別測試模式（`v: T => ...`）去語法糖為 `Reflect` 的標籤檢查。

## 6.10.6. 層級之間的關係

如記法 `Inspect: Reflect` 所示，`Inspect` 以 `Reflect` 作為父 trait。因此實作了 `Inspect` 的型別一律也實作 `Reflect`，這是理所當然的，因為 `Reflect` 本來就是自動的。

```
every type           Reflect          (automatic, zero cost)
+ #[derive(Inspect)] Inspect+Reflect  (opt-in, metadata embedded)
```

## 6.10.7. `TypeInfo`：型別的描述

`T::type_info()`（或對任何 `T` 使用 `core::reflect::info::<T>()`）回傳一個 `TypeInfo`：編譯器在唯讀資料中對 `T` 的描述（[8.13](../008-abi/813-type-descriptor.md)）。對 `Inspect` 型別而言，它列出各欄位及其型別與位元組偏移、列舉的各變體及其欄位、實例的型別引數，以及執行時屬性（6.10.8）；它也能透過 `any` 存取值。

```nabi
use core::reflect::{Field, TypeInfo, TypeKind};

#[derive(Inspect)]
struct Enemy { name: str; hp: i32 = 100; }

let t = Enemy::type_info();
t.name()                           // "Enemy"
t.kind()                           // TypeKind::Struct
t.size() == Enemy::size()          // true
t.field("hp")                      // a Field: name, type_ref, offset, flags
let e: any = Enemy { name: "orc" };
let hp = t.field("hp") ?? panic("no hp");
hp.get(e)                          // 100 inside an any
```

- 讀取描述不會配置任何東西：`name`、`kind`、`size`、`field_count`、`field_at`、`field(name)`、`variant_at`、`attr::<A>()`。回傳 `List` 的方法（`fields()`、`variants()`）會配置該串列。
- `Field::get(owner: any)` 回傳欄位的值，當 `owner` 不是該欄位所屬的型別時回傳 `none`。`Field::get_as::<T>(owner)` 是以 `T` 表示的值。`Field::set(ref owner: any, value: any)` 遵循 `set_field` 的規則，並把修改後的複本存入 `owner`：結構是值。
- 列舉的欄位是其變體的欄位（`Variant::fields()`）；`TypeInfo::variant_of(value)` 是某個值所容納的變體。變體的欄位在另一個變體的值上讀出 `none`。變體的酬載是唯讀的。
- `TypeInfo::build(fields: Dict<str, any>)` 從具名欄位建立一個值；省略的欄位取其 `= expr` 預設值，沒有預設值的欄位是錯誤，錯誤訊息會指出它。`Variant::build` 對列舉做相同的事。`new_default()` 是每個欄位都取預設值的值，或 `none`。
- 串列、陣列或字典的 `TypeInfo` 能存取其項目：`list_items`、`list_len`、`list_get`、`list_from`、`dict_items`、`dict_from`；`element()` 與 `key()` 是項目的型別。
- `info_of(value: any)` 是 `value` 所容納之型別的描述，`find(path)` 是 `"game::Player"` 這類路徑上之型別的描述；當程式沒有描述這樣的型別時，兩者都是 `none`。當程式要求某個型別或由它構成的型別時，該型別就會被描述。
- 當兩個 `TypeInfo` 描述同一個型別時（依鍵，[8.13.2](../008-abi/813-type-descriptor.md#8132-鍵)），它們相等，即使來自不同的模組。
- 欄位上的 `#[inspect(skip)]` 讓它不出現在描述中，也不出現在 `field_names`、`get_field` 與 `set_field` 中。

## 6.10.8. 執行時屬性

標示了 `#[runtime_attribute(name)]` 的結構宣告屬性 `#[name(..)]`。把它寫在型別、欄位或列舉變體上時，其引數像該結構的字面值一樣接受檢查，依欄位順序以位置給出或以欄位名稱給出，而其值保存在描述中。`TypeInfo`、`Field` 或 `Variant` 上的 `attr::<A>()` 以有型別的方式讀回它。

```nabi
/// Shown as a slider in the editor.
#[runtime_attribute(range)]
pub struct Range { pub min: f64; pub max: f64; }

#[derive(Inspect)]
struct Enemy {
    #[range(0, 100)] hp: i32 = 100;
    #[range(min = 0.5, max = 4.0)] speed: f64 = 1.0;
}

let hp = Enemy::type_info().field("hp") ?? panic("no hp");
hp.attr::<Range>()                 // Range { min: 0.0, max: 100.0 }
```

- 屬性結構的欄位是數字、`bool`、`char` 或 `str`；整數字面值可以填入浮點數欄位。
- 錯誤的引數是錯誤，錯誤訊息會說明應寫什麼：``#[range] takes `min: f64, max: f64`; write `#[range(0, 10)]` ``。與某個已宣告名稱相近的名稱會得到 ``no attribute `rnage`; did you mean `range` (`Range`)?``。
- 當兩個套件宣告同一個名稱時，使用它的套件中的那一個勝出；否則該使用是錯誤，錯誤訊息會指出兩者。
- 只有以這種方式宣告的屬性會被保留。`doc`、`cfg` 以及其他內建屬性永遠不會。

欄位上的 `#[renamed_from("old")]` 在其描述中記錄該欄位先前的名稱（`Field::renamed_from`），供讀取以舊名稱寫出之資料的工具使用。

## 6.10.9. 搭配外掛與熱重載

描述位於定義該型別的模組中。每個模組在啟動或被載入時都會向執行環境註冊其描述，而 `info_of` 與 `find` 在那裡查找它們；`find` 回傳最新模組對某個路徑的描述。由於相等性依鍵判斷，兩個模組對同一個泛型實例的描述比較起來相等。
