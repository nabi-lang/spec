# 8.13. 型別描述子 ABI

*型別描述子*（`TypeDesc`）是一個型別的唯讀紀錄。反射會讀取它（[6.10](../006-meta-system/610-reflection.md)），模組載入器用它檢查兩個模組對某個型別的看法是否一致（[8.12](812-module-abi.md)）。其配置在 `nabir_abi::typedesc` 中定義一次；適用 [8.12.2](812-module-abi.md#8122-layout-rules) 的配置規則。

## 8.13.1. 描述子何時存在

只有在程式要求時，編譯器才會為型別發出描述子：`Inspect` 型別的 `T::type_info()` 或 `core::reflect::info::<T>()`，以及遞迴地，其欄位、變體與型別引數所指名之型別的描述子。從不要求的程式沒有任何描述子，其物件與沒有反射時逐位元組相同。

每個描述子都是唯讀資料中的隱藏弱符號，因此同一個程式的兩個目的檔所發出的複本會合併為一份。當程式有描述子時，其 `nabi.register_types` 會在啟動時把它們連同其編譯期的 `any` 標籤傳給 `nabi_types_register`。

## 8.13.2. 鍵

型別的*鍵*跨模組地為它命名：

- struct、enum 或 union：`Y<pkg8>:` 加上其完整路徑，例如 `Y3f8a1c2b:game::Player`，其中 `pkg8` 是定義它之套件的雜湊，與 [9.5](../009-tools/905-symbol-mangling.md) 相同；
- 泛型實例：基底鍵加上 `<..>` 中的引數鍵，例如 `Y..:core::collections::List<Y..:game::Player>`；
- 基本型別、`str`、`any` 與 `none`：其名稱；
- 元組、和型別、陣列與函式型別：以其各部分的鍵依原始碼的寫法寫出，和型別的成員依標準順序排列。

`key_hash` 是鍵之 SHA-256 的前 8 個位元組，以小端序解讀。兩個描述子恰好在其鍵相等時屬於同一個型別。

## 8.13.3. `TypeDesc`

| 欄位 | 型別 | 意義 |
|-------|------|---------|
| `magic` | `u32` | 以小端序 `u32` 表示的 `"NABT"` |
| `version` | `u16` | 1 |
| `kind` | `u16` | `STRUCT` 1、`TUPLE_STRUCT` 2、`UNIT` 3、`ENUM` 4、`UNION` 5、`SUM` 6、`PRIM` 7、`STR` 8、`LIST` 9、`DICT` 10、`ARRAY` 11、`TUPLE` 12、`FN` 13、`ANY` 14、`OPAQUE` 15 |
| `flags` | `u32` | `INSPECT` 1（已填入欄位、屬性與 `ops`）、`LAYOUT_ONLY` 2、`INSTANCE` 4、`HAS_DEFAULT` 8 |
| `tag_index` | `u32` | 模組之 `Tags` 中的索引（[8.12.5](812-module-abi.md#8125-tags)）；沒有時為 `u32::MAX` |
| `key_hash` | `u64` | 8.13.2 |
| `layout_hash` | `u64` | 大小、對齊、欄位偏移與型別鍵、變體標籤以及 GC 字組的雜湊 |
| `size` | `u64` | 與 `T::size()` 相同 |
| `align` | `u32` | 與 `T::align()` 相同 |
| `reserved0` | `u32` | 0 |
| `key` | `Str` | 8.13.2 |
| `name` | `Str` | 與 `T::type_name()` 相同：`Player`、`List<f64>` |
| `path` | `Str` | `game::Player`；沒有路徑的型別為空 |
| `fields` | `Slice<FieldDesc>` | 結構依宣告順序排列的欄位；enum 為空 |
| `variants` | `Slice<VariantDesc>` | 列舉依宣告順序排列的變體 |
| `generics` | `Slice<TypeRef>` | 實例的型別引數；和型別的成員；元組的各部分；串列或陣列的元素；字典的鍵，然後是值 |
| `attrs` | `Slice<AttrDesc>` | 型別上的執行時屬性（[6.10.8](../006-meta-system/610-reflection.md#6108-runtime-attributes)） |
| `ops` | `*const TypeOps` | 動態存取（8.13.5），或 null |
| `reserved1` | `usize` | 0 |

## 8.13.4. 組成部分

| 結構 | 欄位 |
|-----------|--------|
| `TypeRef` | `name: Str`、`key_hash: u64`、`desc: *const TypeDesc`（沒有發出時為 null）、`kind: usize` |
| `FieldDesc` | `name: Str`（在元組結構或變體中為 `"0"`、`"1"`……）、`renamed_from: Str`（除非有 `#[renamed_from(..)]`，否則為空）、`ty: TypeRef`、`attrs: Slice<AttrDesc>`、`offset: u64`（從值的開頭算起；欄位沒有固定位置時為 `u64::MAX`）、`flags: u32`（`PUB` 1、`HAS_DEFAULT` 2、`SKIP` 4）、`index: u32`、`init_default: FnValue`（僅限熱建置） |
| `VariantDesc` | `name: Str`、`fields: Slice<FieldDesc>`、`tag: u32`（存放的標籤）、`index: u32`、`attrs: Slice<AttrDesc>` |
| `AttrDesc` | `ty: TypeRef`（屬性結構）、`value: *const u8`（以該結構之配置表示的常數）、`reserved: usize` |
| `FnValue` | `code: usize`、`env: usize`：一個 Nabi 函式值（[8.8](808-lowering-types.md)）；`env` 為 0 |

## 8.13.5. `TypeOps`

`TypeOps` 容納 Nabi 函式值（型別沒有該項時為 `{0, 0}`），以 Nabi 呼叫慣例呼叫：

| 欄位 | 簽章 | 意義 |
|-------|-----------|---------|
| `get` | `fn(owner: any, field: usize) -> any \| none` | 欄位的值，或在 `owner` 不屬於這個型別時為 `none`。列舉的欄位編號是 `variant << 16 \| index`，對其他變體回答 `none`；`usize::MAX` 以 `usize` 回答 `owner` 所容納之變體的索引。 |
| `set` | `fn(owner: any, field: usize, value: any) -> any \| none` | 設定了該欄位的 `owner` 複本，或在 `owner` 或 `value` 不相符時為 `none` |
| `build` | `fn(variant: usize, fields: Dict<str, any>) -> (any \| none, str)` | 從具名欄位建立的新值（列舉的 `variant`，結構為 0），缺少的欄位取其預設值；或 `none` 加上指出問題所在的訊息 |
| `new_default` | `fn() -> any \| none` | 保留（`{0, 0}`）；不帶任何欄位的 `build` 會建立每個欄位都是預設值的值 |
| `list` | `*const ListOps` | 串列與陣列：`len: fn(any) -> usize`、`get: fn(any, usize) -> any \| none`、`items: fn(any) -> List<any> \| none`、`from: fn(List<any>) -> any \| none` |
| `dict` | `*const DictOps` | 字典：`len: fn(any) -> usize`、`items: fn(any) -> List<(any, any)> \| none`、`from: fn(List<(any, any)>) -> any \| none`、`reserved` |
