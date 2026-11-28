# 2.12. reflect：`Reflect` / `Inspect`

`core::reflect` 模組提供執行時型別資訊與**欄位內省**（field introspection）的權威簽章。詳細語意由 [refs-lang 6.10 反射](../../refs-lang/006-meta-system/610-reflection.md) 規定；本章只涵蓋標準函式庫的 trait 形狀。

## 2.12.1. 兩層 trait

反射分成兩層 trait。

| Trait     | 自動 / 選擇性加入            | 提供                          | 使用場合                                         |
|-----------|-------------------------------|-----------------------------------|----------------------------------------------------|
| `Reflect` | **自動**（每個型別）    | type_name / size / align          | `core` 的 trait，在任何地方都在作用域內（很少用到） |
| `Inspect` | 選擇性加入（`#[derive(Inspect)]`） | field_names / fields / variant_name / get_field / methods | trait 透過隱含的 prelude 提供，derive 必須明確寫出 |

分成自動與選擇性加入兩層，是為了控制二進位檔大小。如果每個結構都帶著欄位反射中繼資料，二進位檔就會變大；選擇性加入的機制只為需要的型別產生中繼資料。

## 2.12.2. `Reflect`：靜態與實例兩種形式

```nabi
trait Reflect {
    fn type_name() -> str;        // static: T::type_name()
    fn size() -> usize;
    fn align() -> usize;

    fn type_name_of_val(self) -> str; // instance: x.type_name_of_val() (any / sum also OK)
    fn size_of_val(self) -> usize;
}
```

這個 trait 會為每個型別自動實作；編譯器會自動把型別資訊附加到二進位檔中。

編譯器為每個型別提供 `Reflect`；原始碼中沒有對應的 impl。

- 手寫 `impl Reflect for T` 是錯誤。約束 `T: Reflect` 永遠成立。
- 如果型別有同名的自有方法，例如 `size()` / `type_name()`，該方法會遮蔽反射方法。
- 名稱（`type_name`）不帶模組路徑（`List<i64>`、`Config`），也不帶等於其預設值的尾端型別引數（`Dict<str, i32>`，[refs-lang 2.10.5](../../refs-lang/002-types/210-generics.md#2105-預設值)）。和型別的靜態名稱依正規順序（[2.6.4](../../refs-lang/002-types/206-sum-types.md#264-集合語意)）寫出（`f64 | i32`，`none` 放在最後）。`type_name_of_val(self)` 等實例形式回報執行時實際存在的成員（對和型別而言），或 `any` 的內容。
- `core::reflect::cast<T>(value: any) -> T | none` 依照與 `set_field` 相同的規則，從 `any` 取出 `T`。整體以和型別存放的值（例如透過 `get_field` 或 `Field::get` 讀出的 `T | none` 欄位）會給出它持有的成員：持有 `"x"` 的欄位 `a: str | none` 的 `cast::<str>` 是 `"x"`。和型別 `T` 也接受它的某個成員，因此 `T | none` 欄位的 `set_field` 接受 `T`。型別的方法或欄位沒有對應的自由函式：它們是 `T::methods()` / `T::fields()`，並且需要 `#[derive(Inspect)]`，也就是型別的同意。

### 靜態與實例

只有一條後綴規則：一般形式是型別的答案，在編譯時就已確定；`_of_val` 形式是值在執行時的答案。

```nabi
i32::type_name();          //→ "i32"           (static, decided at compile time)
i32::size();               //→ 4
i32::align();              //→ 4

let x: i32 = 42;
x.type_name_of_val();      //→ "i32"           (instance, usually the same as static)

let y: i32 | f64 = 42i32;
y.type_name_of_val();      //→ "i32"           (runtime dispatch, checks the sum tag)

let z: any = "hello";
z.type_name_of_val();      //→ "str"           (runtime, any slot dispatch)
```

`type_name_of_val` 是執行時分派，因此對和型別或 `any`，它回傳目前持有的變體的名稱。

### 沒有全域函式

```nabi
typeof(x);                 // ERROR: no global typeof
sizeof(T);                 // ERROR: no global sizeof
```

取而代之的是，一切都統一成方法。

```nabi
x.type_name_of_val()   // instance
T::type_name()             // static
T::size()
T::align()
```

這是 [refs-lang 3.5 成員存取與方法呼叫](../../refs-lang/003-expressions/305-member-access-and-method-call.md) 的一部分：所有內省都統一成方法形式。

## 2.12.3. `Inspect`：選擇性加入的欄位 / 方法

```nabi
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // private fields too
    fn set_field(ref self, name: str, value: any) -> bool;  // whether it succeeded

    // Default bodies; the derive overrides the value forms for an enum.
    fn fields() -> List<FieldInfo>;          // the field table of Self
    fn methods() -> List<MethodInfo>;        // the method table of Self
    fn variant_names() -> List<str>;         // empty for a struct
    fn variant_name(self) -> str;            // a struct: self.type_name_of_val()
    fn field_names_of_val(self) -> List<str>; // the fields self holds
}

#[non_exhaustive]
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
pub struct FieldInfo {
    pub name: str;                 // "0", "1", .. for a tuple field
    pub type_name: str;            // instantiated: f64 for `v: T` in Box<f64>
    pub is_pub: bool;              // true only for a plain `pub` field
}

#[non_exhaustive]
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
pub struct MethodInfo {
    pub name: str;
    pub arity: usize;              // number of arguments excluding self
    pub is_static: bool;           // method that does not take self
    pub arg_types: List<str>;      // argument type names excluding self
    pub return_type: str;
}
```

- `get_field` 以 `any` 回傳欄位的值，名稱不存在時回傳 `none`。私有欄位也可以讀取。
- 當值的型別與欄位的型別**完全相同**，或者值是另一種整數型別且**可以無損放入**時（例如以 `u.set_field("id", 42)` 寫入 `u64` 欄位），`set_field` 會指派並回傳 `true`。否則不指派並回傳 `false`。沒有其他轉換（例如 `i32` → `f64`）。
- `methods()` **只回傳型別的固有方法**（不含 trait 方法），依名稱排序。泛型 `impl` 的方法依寫法顯示型別（`T`）。
- `fields()` 依宣告順序列出每個欄位，附上型別名稱與是否為 `pub`。**私有欄位也會列出**：衍生 `Inspect` 就是型別作者的同意，與 `get_field` 相同。型別名稱是實例化後的名稱（`Box<f64>` 回報 `v: f64`、`items: List<f64>`），不帶模組路徑，寫法與 `type_name` 相同。
- `enum` 在型別層級沒有欄位（`field_names()` 和 `fields()` 是空串列）。`variant_names()` 依宣告順序列出其變體。在值上，`variant_name()` 是它持有的變體（`"Circle"`），`field_names_of_val()` 是該變體的欄位（元組變體為 `"0"`、`"1"`，具名變體為欄位名稱），`get_field` 讀取它們。透過 `Inspect`，變體的酬載是唯讀的：對列舉呼叫 `set_field` 得到 `false`。
- 在結構上，`variant_names()` 是空串列，`variant_name()` 是它的型別名稱（`type_name_of_val()`），`field_names_of_val()` 是它的欄位。
- `fields()`、`methods()`、`variant_names()`、`variant_name()` 和 `field_names_of_val()` 有預設主體，因此手寫的 `impl Inspect` 只需要前三個方法。
- 在沒有衍生的型別上呼叫 `Inspect` 方法是錯誤，其說明會指出該寫什麼：``add `#[derive(Inspect)]` to `Point` ``。

### 衍生範例

```nabi
#[derive(Inspect)]
struct User {
    id: u64;
    name: str;
    email: str;
}

let u = User { id: 1, name: "Alice", email: "a@b" };

User::field_names();       //→ ["id", "name", "email"]
u.get_field("name");       //→ any("Alice")
u.set_field("id", 42);     //→ true
u.id;                      //→ 42
User::fields()[0].type_name; //→ "u64"

#[derive(Inspect)]
enum Shape { Circle(f64), Rect { w: f64; h: f64; } }

let s = Shape::Rect { w: 2.0, h: 3.0 };
Shape::variant_names();    //→ ["Circle", "Rect"]
s.variant_name();          //→ "Rect"
s.field_names_of_val();    //→ ["w", "h"]
s.get_field("w");          //→ any(2.0)
```

### 使用元區塊

`#[derive(Inspect)]` 以 `core::reflect` 中的元區塊實作（[refs-lang 6.8](../../refs-lang/006-meta-system/608-meta-blocks.md)、[6.9](../../refs-lang/006-meta-system/609-template-tokens.md)）。它只寫出型別描述所指向的三個存取器；其他每個方法都有建立在 `type_info()` 之上的預設主體。概要如下：

```nabi
#[impl_derive(Inspect)]
pub @derive {
    impl core::reflect::Inspect for @Self {
        // Field `field` of `owner`: a struct's by position; an enum's as
        // `variant << 16 | index`, and `usize::MAX` for the variant index.
        fn __inspect_get(owner: any, field: usize) -> any | none {
            match core::reflect::cast::<@Self>(owner) {
                s: @Self => {
                    @if_struct {
                        @for_fields {
                            if field == @field.index { let held: any = s.@field; return held; }
                        }
                    }
                    @if_enum { /* per variant, through @variant.pattern */ }
                    return none;
                },
                none => { return none; },
            }
        }
        // A copy of `owner` with one field set (structs only).
        fn __inspect_set(owner: any, field: usize, value: any) -> any | none { /* .. */ }
        // A value from named fields; a missing field takes its default.
        fn __inspect_build(variant: usize, fields: Dict<str, any>) -> (any | none, str) {
            @if_struct {
                @for_fields {
                    let @field.binding: @field.type = /* fields[name], or @field.default */;
                }
                let built: @Self = @Self.pattern;
                let held: any = built;
                return (held, "");
            }
            /* @if_enum: the same per variant */
        }
    }
}
```

### 需要動態分派的地方

`Inspect` 用在需要依欄位名稱存取的領域，例如設定物件、序列化器和偵錯工具。在一般程式碼中，只用 `Reflect` 就夠了。

```nabi
fn dump_struct<T: Inspect>(x: T) {
    for f in T::fields() {
        if let v: any = x.get_field(f.name) {
            println(f"{f.name}: {f.type_name} ({v.type_name_of_val()})");
        }
    }
}
```

## 2.12.4. REPL 輸出

REPL 以 `Display` 印出單獨的最後一個運算式，沒有 `Display` 時用 `Debug`；結尾帶 `?` 時用 `Debug`；兩者都沒有的值則用本章的 `Reflect::type_name_of_val` 印出（[2.10 fmt](210-fmt.md)、[refs-lang 6.10.3](../../refs-lang/006-meta-system/610-reflection.md#6103-repl--jupyter-的自動印出)）。

```
>>> let x = 42i32;
>>> x
42

>>> [1, 2, 3]
[1, 2, 3]

>>> [1, 2, 3]?
[1, 2, 3]

>>> User { id: 1, name: "Alice" }?
User { id: 1, name: "Alice" }
```

單獨的運算式使用 `Display`，沒有時用 `Debug`；結尾的 `?` 使用 `Debug`。兩者都沒有的值（對結尾的 `?` 而言是沒有 `Debug`）只印出其型別名稱；`any` 值則印出它所持有的型別的名稱。

## 2.12.5. 模組對應

| 項目               | 位置                                          | 機制                         |
|--------------------|---------------------------------------------------|-----------------------------------|
| `Reflect`（自動） | 編譯器 + `core::reflect`                | 型別資訊表，附加在二進位檔中  |
| `Inspect` derive   | `core::reflect` + 元區塊 `#[impl_derive]` | 由樣板記號產生程式碼 |
| `MethodInfo`       | `core::reflect`                              | 一般結構                      |
| `FieldInfo`       | `core::reflect`                              | 編譯器欄位表（實例化後的型別） |
| `any` 型別         | 編譯器內建                                 | 型別標籤 + 酬載                |

## 2.12.6. 與其他語言的比較

| 面向              | Rust                         | Java               | Python              | Nabi                                         |
|---------------------|------------------------------|--------------------|---------------------|---------------------------------------------|
| 型別名稱           | `std::any::type_name::<T>()` | `getClass()`       | `type(x).__name__`  | `x.type_name_of_val()` 方法                   |
| 欄位內省 | **無**（由 `bincode` 等函式庫提供） | Reflection API | `vars(x)` `getattr` | `x.get_field(name)`（選擇性加入）          |
| 執行時分派    | `dyn Trait`（vtable）         | 自動          | 自動           | 和型別標籤 + `any` / `any Trait`（選擇性加入） |
| 自動實作      | **無**                     | 每個物件       | 每個物件        | `Reflect` 自動，`Inspect` 選擇性加入       |
| 記憶體成本         | 0                            | 每個類別都有中繼資料 | 每個物件都有 dict | 只有 `Reflect` 是自動的（選擇性加入的分層） |

Nabi 主要有兩點不同。第一，兩層分割讓輕量的 `Reflect` 自動提供，重量級的 `Inspect` 則選擇性加入。這比 Java 或 Python 全自動的模型更節省記憶體，也比 Rust 全手動的模型更好用。第二，為了方法的一致性，**沒有** `typeof(x)` 這類全域函式；所有內省都以方法形式提供。

詳細的反射語意由 [refs-lang 6.10](../../refs-lang/006-meta-system/610-reflection.md) 規定。

## 2.12.7. `TypeInfo`

`T::type_info()` / `info::<T>()` 回傳的描述（[refs-lang 6.10.7](../../refs-lang/006-meta-system/610-reflection.md#6107-typeinfo型別的描述)，佈局見 [8.13](../../refs-lang/008-abi/813-type-descriptor.md)）。回傳值的存取器就地讀取，不做任何配置；回傳 `List` 的存取器會配置該串列。

```nabi
pub fn info<T>() -> TypeInfo;                 // any T; fields and accessors when T: Inspect
pub fn info_of(value: any) -> TypeInfo | none; // the type value holds, if described
pub fn find(path: str) -> TypeInfo | none;    // "game::Player"

pub enum TypeKind { Struct, Enum, Union, Sum, Prim, Str, List, Dict, Array, Tuple, Fn, Any, Opaque }
pub struct ReflectError { pub message: str; }  // + Display, Error

impl TypeInfo {                                 // + Clone, PartialEq (by key), Debug
    pub fn name(self) -> str;                   // as type_name: "Player", "List<f64>"
    pub fn path(self) -> str;                   // "game::Player"
    pub fn package(self) -> str;
    pub fn type_key(self) -> str;               // "Y3f8a1c2b:game::Player"
    pub fn layout_hash(self) -> u64;
    pub fn kind(self) -> TypeKind;
    pub fn size(self) -> usize;
    pub fn align(self) -> usize;
    pub fn is_inspect(self) -> bool;
    pub fn fields(self) -> List<Field>;
    pub fn field_count(self) -> usize;
    pub fn field_at(self, i: usize) -> Field;
    pub fn field(self, name: str) -> Field | none;
    pub fn field_of_val(self, value: any, name: str) -> Field | none;
    pub fn variants(self) -> List<Variant>;
    pub fn variant_count(self) -> usize;
    pub fn variant_at(self, i: usize) -> Variant;
    pub fn variant(self, name: str) -> Variant | none;
    pub fn variant_of(self, value: any) -> Variant | none;
    pub fn generics(self) -> List<TypeRef>;
    pub fn generic_count(self) -> usize;
    pub fn generic_at(self, i: usize) -> TypeRef;
    pub fn element(self) -> TypeRef | none;     // list / array element, dict value
    pub fn key(self) -> TypeRef | none;         // dict key
    pub fn attr<A>(self) -> A | none;
    pub fn attrs(self) -> List<Attr>;
    pub fn holds(self, value: any) -> bool;
    pub fn new_default(self) -> any | none;
    pub fn build(self, fields: Dict<str, any>) throws (ReflectError) -> any;
    pub fn list_len(self, value: any) -> usize;
    pub fn list_get(self, value: any, i: usize) -> any | none;
    pub fn list_items(self, value: any) -> List<any> | none;
    pub fn list_from(self, items: List<any>) throws (ReflectError) -> any;
    pub fn dict_items(self, value: any) -> List<(any, any)> | none;
    pub fn dict_from(self, items: List<(any, any)>) throws (ReflectError) -> any;
    pub unsafe fn from_desc(at: usize) -> TypeInfo; // the descriptor at address at, which must stay mapped
}

impl TypeRef { pub fn name(self) -> str; pub fn info(self) -> TypeInfo | none; }

impl Field {
    pub fn name(self) -> str;                   // "0", "1", .. in a tuple
    pub fn renamed_from(self) -> str;
    pub fn type_ref(self) -> TypeRef;
    pub fn offset(self) -> usize;               // usize::MAX: no fixed place
    pub fn index(self) -> usize;
    pub fn is_pub(self) -> bool;
    pub fn has_default(self) -> bool;
    pub fn attr<A>(self) -> A | none;
    pub fn attrs(self) -> List<Attr>;
    pub fn get(self, owner: any) -> any | none;
    pub fn get_as<T>(self, owner: any) -> T | none;
    pub fn set(self, ref owner: any, value: any) -> bool;
}

impl Variant {
    pub fn name(self) -> str;
    pub fn index(self) -> usize;
    pub fn fields(self) -> List<Field>;
    pub fn field_count(self) -> usize;
    pub fn field_at(self, i: usize) -> Field;
    pub fn field(self, name: str) -> Field | none;
    pub fn attr<A>(self) -> A | none;
    pub fn attrs(self) -> List<Attr>;
    pub fn build(self, fields: Dict<str, any>) throws (ReflectError) -> any;
}

impl Attr { pub fn type_ref(self) -> TypeRef; pub fn get<A>(self) -> A | none; }
```

`std::nabion::to_value_any` 和 `from_value_any`（[3.8.5](../300-std/308-nabion.md#385-函式庫stdnabion)）建立在這些之上。
