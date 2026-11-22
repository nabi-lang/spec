# 6.9. 樣板記號與展開

本節定義可以在[元區塊](608-meta-blocks.md)主體中使用的樣板記號完整清單，以及元區塊在何時、以什麼機制展開。

## 6.9.1. 樣板記號的形式

樣板記號有下列語法形式。

```
template_token  ::= "@" identifier
                  | "@" identifier "." identifier
                  | "@" identifier "{" <template body> "}"
                  | "@" identifier "(" <args> ")" "{" <body> "}"
```

每個樣板記號都以 `@` 語彙單元（[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)）開頭。具代表性的範例：

```
@Self                   target type name
@field                  placeholder for the current field
@field.name             name of the current field
@for_fields { ... }     field iteration block
```

### 記號清單

| 記號                                                     | 值                                                     | 節 |
|----------------------------------------------------------|--------------------------------------------------------|---------|
| `@Self`                                                  | 目標型別（`Name<T, ..>`）                              | 6.9.2 |
| `@Self.name`                                             | 不帶型別引數的目標型別名稱                             | 6.9.2 |
| `@Self.is_unit` / `.is_tuple` / `.is_named`              | 結構的形狀（布林值）                                   | 6.9.2 |
| `@Self.field_count`、`@Self.variant_count`               | 結構 / 聯集的欄位數量、`enum` 的變體數量（整數字面值） | 6.9.2 |
| `@if_struct` / `@if_enum` / `@if_union { .. }`           | 只在目標種類相符時產生主體                             | 6.9.2 |
| `@for_fields { .. }`                                     | 每個欄位產生一次主體                                   | 6.9.3 |
| `@field`、`@field.name`、`@field.type`                   | 目前的欄位（存取 / 名稱 / 型別）                       | 6.9.3 |
| `@field.is_pub`、`@field.has_attr(x)`                    | 布林值                                                 | 6.9.3 |
| `@field.has_default`、`@field.default`                   | 欄位是否有 `= expr`（布林值）/ 該運算式               | 6.9.3 |
| `@field.index`、`@field.binding`                         | 欄位的位置（整數字面值）/ 其值的區域名稱               | 6.9.3 |
| `@Self.pattern`                                          | 由 `@field.binding` 區域變數建出的結構                 | 6.9.3 |
| `@for_variants { .. }`                                   | 每個 `enum` 變體產生一次主體                           | 6.9.4 |
| `@variant.name`、`@variant.pattern`                      | 目前的變體（名稱 / 比對模式）                          | 6.9.4 |
| `@variant.index`                                         | 目前變體的位置（整數字面值）                           | 6.9.4 |
| `@variant.is_unit` / `.is_tuple` / `.is_named`           | 布林值                                                 | 6.9.4 |
| `@for_variant_fields { .. }`                             | 目前變體的每個欄位產生一次主體                         | 6.9.4 |
| `@apply_if(cond) { .. }`                                 | 只在 `cond` 為真時產生主體                             | 6.9.5 |
| `@accept(kind, ..)`                                      | 限制目標種類（僅限屬性）                               | 6.9.5 |
| `@where_all(Trait)`                                      | 每個型別參數加上 `T: Trait`                            | 6.9.5 |
| `@has_derive_attr(a)`、`@derive_attr(a)`                 | 欄位的 derive 屬性（布林值 / 引數）                    | 6.9.6 |
| `@Self.has_derive_key(k)`                                | 型別的 `#[attr(..)]` 列出了鍵 `k`（布林值）            | 6.9.6 |
| `@item`、`@item.name` / `.params` / `.return_type` / `.body` | 所附加的項目及其各部分（僅限屬性）                 | 6.9.7 |
| `@attribute`                                             | 使用處的屬性引數（僅限屬性）                           | 6.9.7 |
| `<name token>.as_str()`                                  | 名稱的字串字面值                                       | 6.9.3 |
| `@use_site(path)`                                        | `path`，在使用元區塊之處查找                           | 6.9.9 |
| `@concat(a, @Self.name, ..)`                             | 由名稱、索引記號與字串接合而成的一個識別字             | 6.9.8 |

不在清單中的 `@name` 是錯誤。`@where_for_fields` 是保留名稱；使用它也是錯誤。

## 6.9.2. 型別上下文

### `@Self` 目標型別名稱

`@Self` 表示元區塊所套用之目標型別的名稱。

```nabi
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {                 // @Self → Point / the user's type name
        ...
    }
}
```

### `@if_struct { ... }` / `@if_enum { ... }` / `@if_union { ... }`

這些記號只在目標型別分別是 `struct`、`enum` 或 `union` 時，才產生其中的程式碼。

```nabi
@if_struct {
    fn struct_only_method() { ... }
}

@if_enum {
    fn enum_only_method() { ... }
}
```

若目標型別是其他種類，該區塊會被忽略，不產生任何程式碼。

### `@Self.name` / `@Self.is_unit` / `.is_tuple` / `.is_named`

`@Self.name` 是不帶型別引數的目標型別名稱（`Pair<T>` 的 `Pair`）。`@Self.is_unit` / `@Self.is_tuple` / `@Self.is_named` 是表示目標結構是 unit 結構、元組結構還是帶具名欄位之結構的布林值；它們用在 `@apply_if` 中。

`@Self.field_count` 是目標 `struct` 或 `union` 的欄位數量，`@Self.variant_count` 是目標 `enum` 的變體數量，兩者都是整數字面值（`[u8; @Self.field_count]`、`fn count() -> usize { @Self.variant_count }`）。在 `enum` 上使用 `@Self.field_count` 是錯誤（變體的欄位以 `0 @for_variant_fields { + 1 }` 計數），在不是 `enum` 的型別上使用 `@Self.variant_count` 也是錯誤（請以 `@if_enum { .. }` 保護它）。

## 6.9.3. 欄位迭代

### `@for_fields { ... }`

`@for_fields` 對 `struct` 或 `union` 的每個欄位重複該區塊。在 `@for_variants` 內，它迭代目前變體的欄位，與 `@for_variant_fields` 相同。在沒有 `@for_variants` 的 `enum` 上使用它是錯誤。

```nabi
@for_fields {
    println(self.@field);
}
```

這段程式碼為每個欄位產生一次。

### `@field`

`@field` 是目前欄位的佔位符，用在欄位存取的位置。

```nabi
self.@field                          // self.x, self.y, ...
@Self { @field: 0i32 }                // field initializer position
```

### `@field.name` 識別字語彙單元

在需要識別字之處，欄位名稱以識別字語彙單元的形式插入。

```nabi
let v = @field.name;                  // the name goes into the identifier position
fn @field.name() -> i32 { ... }       // can also be used as a function name
```

若需要字串，請附加 `.as_str()`。`@field.name.as_str()`、`@variant.name.as_str()`、`@item.name.as_str()` 與 `@Self.as_str()` 在展開時變成該名稱的字串字面值（`"x"`）。`@Self.as_str()` 是不帶型別引數的名稱。寫在字串或 f-string 內的 `@item.name` 等名稱記號會被逐字替換。

### `@field.type`

`@field.type` 表示欄位的型別。

```nabi
fn typed<T>() where T == @field.type { ... }
```

### `@field.is_pub`

若欄位是 `pub`，這個記號為真，否則為假。

```nabi
@apply_if(@field.is_pub) {
    // process only pub fields
}
```

### `@field.has_attr(X)`

若欄位上附加了 `#[X]`，就為真。

```nabi
@apply_if(@field.has_attr(skip)) {
    // process only fields with #[skip]
}
```

### `@field.has_default` / `@field.default`

若欄位有預設值（`x: i32 = 0`，[2.3.3](../002-types/203-struct.md#233-欄位預設值)），`@field.has_default` 為真，而 `@field.default` 是該 `= expr` 的運算式。在沒有預設值的欄位上使用 `@field.default` 是錯誤。

```nabi
@apply_if(@field.has_default) {
    let @field.name = @field.default;
}
```

## 6.9.4. 變體迭代

### `@for_variants { ... }`

`@for_variants` 對 `enum` 的每個變體重複該區塊。

```nabi
@for_variants {
    @variant.name => /* ... */
}
```

### `@variant.name` 變體名稱

`@variant.name` 是變體的名稱。

### `@variant.pattern` 比對模式

`@variant.pattern` 是比對模式形式的變體，例如 `Shape::Circle(r)`、`Shape::Pending` 或 `Shape::Click { x: x1, y: y1 }`。在 `r`、`x1`、`y1` 位置的繫結是編譯器產生的名稱（衛生，[6.9.9](#衛生)），其值在 `@for_variant_fields` 內以 `@field` 取得。樣板不能直接使用這些繫結名稱。

```nabi
match self {
    @for_variants {
        @variant.pattern => /* ... */
    }
}
```

`@variant.pattern` 也是一個運算式：在每個 `@field`（於 `@for_variant_fields` 中）都以 `let` 繫結之後，它會建出該變體。

### `@variant.index`、`@field.index`、`@field.binding`、`@Self.pattern`

`@variant.index` 與 `@field.index` 是目前變體與欄位依宣告順序的位置，以整數字面值表示：`c.@field.index.push(self.@field)` 存取某個欄位的元組欄。在 `@for_fields` / `@for_variant_fields` 之外的 `@field` 記號是錯誤，錯誤訊息會提示把它放進 `@for_fields { .. }` 內。`@field.binding` 是目前欄位之值的區域名稱：在結構上，`@Self.pattern` 從那些區域變數建出結構（`Name { a: <a's binding>, .. }`、`Name(..)` 或 `Name`）；在變體中，它是 `@variant.pattern` 所繫結的名稱。

```nabi
@for_fields {
    let @field.binding: @field.type = read_field(@field.name.as_str());
}
let made: @Self = @Self.pattern;
```

### `@variant.is_unit` / `@variant.is_tuple` / `@variant.is_named`

這些記號測試變體的種類。

```nabi
@apply_if(@variant.is_tuple) {
    // process only tuple variants
}
```

### `@for_variant_fields { ... }`

要迭代變體內的欄位，請使用 `@for_variant_fields`。其中可以使用 `@field.name` 等。

```nabi
@for_variants {
    @variant.pattern => {
        @for_variant_fields {
            process(@field);
        }
    }
}
```

## 6.9.5. 條件與約束

### `@apply_if(cond) { ... }`

`@apply_if` 只在 `cond` 為真時產生其中的程式碼。

```nabi
@apply_if(@field.is_pub) {
    // code
}

@apply_if(!@field.has_attr(skip)) {
    // only fields *without* skip
}

@apply_if(@field.is_pub && @field.has_attr(serialize)) {
    // both conditions true
}
```

`cond` 是以一般布林運算子（`!`、`&&`、`||`）組合樣板記號之布林值（例如 `@field.is_pub`、`@field.has_attr(...)`）的運算式，在編譯期求值。

`else` 分支不用另外的關鍵字，而是以條件相反的 `@apply_if` 表達。

### `@accept(...)`

`@accept` 限制屬性的目標種類；在 derive 中是錯誤。它接受以逗號或 `|` 分隔的項目種類清單。

```nabi
#[impl_attribute(my_wrap)] pub @attribute {
    @accept(struct, enum)                    // accepts only struct, enum; rejects union
    @item
}

#[impl_attribute(timed)] pub @attribute {
    @accept(fn)                              // accepts only fn
    fn @item.name(...) { ... }
}
```

可能的種類：

| 種類     | 意義                    |
|----------|-------------------------|
| `struct` | `struct` 項目           |
| `enum`   | `enum` 項目             |
| `union`  | `union` 項目            |
| `fn`     | 函式或方法              |
| `impl`   | `impl` 區塊             |
| `trait`  | `trait` 項目            |

以逗號或 `|` 分隔多個種類時，符合其中任何一個就是允許的。若目標不是允許的種類，就是使用處的編譯錯誤。

### `@where_all(Trait)`

`@where_all` 以自動的 where 子句加入「每個欄位型別都必須滿足 `Trait`」的條件。

```nabi
#[impl_derive(Same)] pub @derive {
    @where_all(Same)                         // OK if every field's type is Same
    impl Same for @Self {
        fn same(self, other: Self) -> bool {
            @for_fields {
                if !self.@field.same(other.@field) { return false; }
            }
            true
        }
    }
}
```

這裡的 `Same` 是使用者的 trait（`Eq` 之類的內建 derive 不能被重新定義，[6.8.11](608-meta-blocks.md#6811-與註冊--可見性的結合)）。產生的 `impl` 於是為目標型別的每個型別參數得到 `T: Same`。對具體的欄位型別不加入子句；那些欄位是否實作 `Same`，由對產生之主體的型別檢查決定。目標型別的泛型參數放在樣板的 `impl` 之後，而唯一的約束是 `@where_all` 加入的那些。

## 6.9.6. derive_attribute 記號

在 `#[impl_derive_attribute(Trait, attr)]` 內還可以使用下列記號。

### `@has_derive_attr(name)`

表示 `derive_attribute` 的目標 attr 是否附加在該欄位上。

### `@derive_attr(name)` 該 attr 的引數

`@derive_attr(name)` 表示所附加之 attr 的引數。

### `@Self.has_derive_key(k)`

型別自己的 `#[attr(..)]` 是否列出了鍵 `k`。`k` 必須以 `#[impl_derive_attribute(Trait, attr(k, ..))]` 宣告（[6.8.4](608-meta-blocks.md#684-impl_derive_attributetrait-attr-逐欄位的-derive-屬性)）；未宣告的鍵是錯誤。`#[derive(Nabion)]` 就是以這種方式讀取 `#[nabion(deny_unknown_fields)]`。

```nabi
#[impl_derive_attribute(Serialize, rename)] pub @derive {
    @for_fields {
        @apply_if(@has_derive_attr(rename)) {
            let key = @derive_attr(rename);  // the argument of #[rename("...")]
            obj.set(key, ...);
        }
    }
}
```

## 6.9.7. 屬性專用的記號

下列記號用在 `#[impl_attribute(name)]` 內。

### `@item` 重用原始項目

`@item` 原樣重用屬性所附加的整個項目。

```nabi
#[impl_attribute(deprecated_warning)] pub @attribute {
    @item                                    // the original as is
}
// the attribute above is effectively a no-op; the original passes through unchanged
```

### `@item.body`

`@item.body` 是函式主體；它只對函式項目有意義。

### `@item.name`、`@item.params`、`@item.return_type`

這些記號表示函式簽章的各部分。

```nabi
#[impl_attribute(timed)] pub @attribute {
    fn @item.name(@item.params) -> @item.return_type {
        let start = Instant::now();
        let r = { @item.body };
        log(f"@item.name took {Instant::now() - start}");
        r
    }
}
```

### `@attribute` `#[...]` 的引數

`@attribute` 表示在附加處寫出的引數。若使用處把以 `#[impl_attribute(name)]` 定義的屬性套用為 `#[name(args)]`，`@attribute` 就是那些 args。

```nabi
#[impl_attribute(log_prefix)] pub @attribute {
    fn @item.name() {
        let p = @attribute;                  // the use site's argument
        log(f"{p}: {@item.name}");
        @item.body
    }
}

// use site
#[log_prefix("INFO")]
fn run() { ... }
// → generated.
//   fn run() {
//       let p = "INFO";
//       log(f"{p}: run");
//       ...
//   }
```

## 6.9.8. 呼叫處（巨集）記號

上述記號繫結定義所附加的型別或項目。巨集（[6.8.12](608-meta-blocks.md#6812-巨集形式)）則透過匹配器繫結呼叫者的語彙單元，並使用下列記號；`@for` 也只用於巨集：在 `@attribute` 或 `@derive` 區塊中它是錯誤（它們以 `@for_fields` / `@for_variants` 重複）。`@concat` 在三種形式中都可以使用。反過來，形狀記號（`@apply_if` 以及它所測試的 `@Self` / `@field` / `@variant` 記號）在巨集中是錯誤：巨集沒有可以測試的型別形狀。

| 記號                                   | 在匹配器 / 樣板中                                              | 節 |
|----------------------------------------|----------------------------------------------------------------|---------|
| `@name: <fragment>`                    | 匹配器：擷取一個片段（`expr`、`stmt`、`block`、`ty`、`pat`、`path`、`ident`、`literal`、`item`、`tt`） | 6.9.8 |
| `@name`                                | 樣板：擷取的片段，或 `@for` 中目前的元素                       | 6.9.8 |
| `@rep[sep = ",", trailing] { .. }`     | 匹配器：重複的子模式，可帶分隔符與尾隨分隔符                   | 6.9.8 |
| `@for @name { .. }`                    | 樣板：為擷取之序列的每個元素產生一次主體（多個序列同步）       | 6.9.3 |
| `@concat(a, @name, ..)`                | 樣板：把名稱語彙單元 / 字串字面值接合成一個被索引的識別字（在 `@derive` / `@attribute` 中也可以） | 6.9.8 |

### `@name: <fragment>` 片段擷取

在匹配器中，`@name: <fragment>` 以原始語彙單元的形式擷取呼叫的一部分；比對時不剖析任何東西，因此會選擇第一個字面語彙單元與語彙單元數量都相符的規則（[6.8.12](608-meta-blocks.md#匹配器與片段)）。`ident`、`literal` 與 `tt` 分別取一個識別字、一個字面值語彙單元與一個語彙單元樹（一個語彙單元或一個平衡的分組）；其他片段取到匹配器的下一個字面語彙單元為止的語彙單元序列。展開在轉錄之後被當作一般程式碼剖析，因此被擷取的程式碼在那裡進行型別檢查，工具也能解析其中的名稱；在樣板所放置之處不相符的引數，是在呼叫處回報的語法錯誤。`tt` 是用於非 Nabi 之 DSL 的片段。匹配器中其他地方的字面語彙單元必須原樣出現。在樣板中，`@name` 替換為擷取的內容，並保留呼叫處的衛生性（[6.9.9](#699-展開機制)）。

### `@rep` / `@for` 重複

`@rep[sep = ",", trailing] { <sub-pattern> }` 比對子模式零次以上；在其中擷取的後設變數繫結一個序列。`@for @name { <body> }` 與 `@for_fields`（[6.9.3](#693-欄位迭代)）是同一種迭代器，這裡是走訪那個序列；來自同一個 `@rep` 的後設變數同步迭代，長度必須相等。`@rep` 與 `@for` 可以巢狀。

### `@concat(..)` 名稱合成

`@concat(..)` 把名稱語彙單元（`@name`、單獨的識別字）與字串字面值接合成單一識別字，只能用在定義或路徑位置。它所形成的識別字會被記錄，因此符號搜尋與跳到定義能到達所產生的項目（[6.8.12](608-meta-blocks.md#工具與衛生)）；非名稱的片段，或在名稱位置之外使用，是錯誤。

在 `@derive` 或 `@attribute` 區塊中，其片段也可以是目標的名稱與索引記號（`@Self.name`、`@field.name`、`@variant.name`、`@field.index`、`@variant.index`、`@item.name`），因此 derive 可以宣告一個具名的配套項目。不以字母或 `_` 開頭的結果是錯誤。

```nabi
#[impl_derive(Soa)] pub @derive {
    pub struct @concat(@Self.name, Columns) {
        @for_fields { pub @field.name: List<@field.type>; }
    }
}

#[derive(Soa)]
struct Point { x: f32; y: f32; }    // also declares `PointColumns { x: List<f32>; y: List<f32>; }`
```

## 6.9.9. 展開機制

### 收集與展開

meta 展開依下列方式進行。

1. 剖析之前的預處理，從函式庫與使用者套件的每個檔案收集元區塊（[6.8.11](608-meta-blocks.md#6811-與註冊--可見性的結合)）。
2. 當剖析器遇到帶有 `#[derive(X)]` 或 `#[x]` 的項目時，它以該項目的值替換該元區塊主體中的樣板記號，並重新剖析結果。
3. 產生的項目會再次進行 meta 處理，重複直到到達固定點。
4. 循環（`A → B → A`）是錯誤（[6.8.8](608-meta-blocks.md#688-固定點展開)）。

### 與 `cfg` 的關係

元區塊本身上的 `#[cfg(..)]` 決定該區塊的**註冊**。cfg 的求值先於元區塊的收集，因此 cfg 為假之區塊的 derive / 屬性不會被註冊。因此兩個區塊可以在互斥的 cfg 之下定義同一個名稱。包圍該區塊之模組（行內 `mod` 與檔案模組）的 `#[cfg]` 也適用於註冊。檔案內的 `#![cfg]` 尚不適用。

項目上的 [`#[cfg]`](603-cfg.md) 會原樣複製到由它產生的每個項目上。因此 cfg 為假的項目會連同其產生的輸出一起消失；整體順序如下：

```
parse → cfg eval → meta expansion (fixed-point) → name resolution → type checking → ...
```

### 衛生

樣板所繫結的區域名稱（`let` / `for` 繫結與閉包參數，包括 f-string 的 `{..}` 內）在展開時會被改名為不衝突的名稱。因此樣板的區域變數既不會隱藏也不會擷取來自項目的名稱（`@item.body`、`@item.params`、`let @field.name = ..`）。

```nabi
#[impl_derive(Foo)] pub @derive {
    impl Foo for @Self {
        fn foo(self) -> i32 {
            let temp = compute();             // renamed at expansion
            temp * 2
        }
    }
}

// use site
#[derive(Foo)]
struct S { temp: str; }                       // the use site's 'temp' field

let s = S { temp: "hi" };
let r = s.foo();                              // no collision with the meta-generated 'temp'
```

由 `match` 分支模式與 `S { x }` 簡寫所繫結的名稱不會被改名。樣板在那些位置不使用可能與項目衝突的名稱。`@variant.pattern` 的繫結是編譯器產生的名稱，不會衝突。

### 展開中的名稱

展開中的每個語彙單元，要麼來自**定義**（樣板，或編譯器所寫的程式碼），要麼來自**使用**（`@Self`、`@field`、`@variant`、`@item`、`@attribute`、`@derive_attr` 與巨集擷取所替換的東西）。路徑取其第一個片段所在的那一側；其泛型引數保持它們自己的一側。

來自定義的路徑，就像寫在包含該元區塊的模組中一樣解析並接受存取檢查：先是展開本身宣告的名稱，然後是該模組的作用域（其項目與 `use`、`pkg::`、其套件的相依套件、prelude 的名稱）。它可以指稱該模組的私有項目以及其套件的 `pub(pkg)` 項目。來自定義的方法名稱，以那裡作用域內的 trait 查找（[3.5](../003-expressions/305-member-access-and-method-call.md)）。來自使用的路徑在使用處解析並接受檢查，因此傳入 meta 定義的程式碼不會因此獲得存取權。樣板中的 `pkg::` 指稱定義該元區塊的套件，無論它在哪裡展開。

編譯器所寫的程式碼（內建 derive、`#[literal]` 處理器呼叫）從其所指稱之東西的模組來看：處理器就像從它自己的模組被呼叫一樣，因此它可以有任何可見性。

產生的項目放在使用處的模組中，它們所宣告的名稱在那裡可見。樣板的區域變數會被改名（見上文的衛生）。

`@use_site(path)` 讓樣板路徑在使用處解析，用於預期由使用者定義的名稱。其引數必須是路徑。

```nabi
// package mylib
pub(pkg) fn combine(h: u64, x: u64) -> u64 { .. }

#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self {
        fn my_hash(self) -> u64 {
            let h: u64 = 0u64;
            @for_fields { h = combine(h, self.@field.my_hash()); }   // mylib's, though pub(pkg)
            @use_site(check)(self);                                    // the user's `check`
            h
        }
    }
}
```

定義模組中沒有的、來自定義的名稱，是樣板處的錯誤，並以使用處作為附註，說明會指出 `@use_site(..)`。建置一個套件時，其 `pub` 元區塊中的每個有根路徑（`pkg::..`、`std::..`、相依套件的路徑）都必須能從該區塊的模組解析，無論是否被使用，因此錯誤由作者負責。

### 診斷位置

展開期間的錯誤以及展開結果的剖析錯誤，會指向樣板的位置，並附上使用處（`#[derive(X)]` / `#[x]`）作為附註。透過樣板記號來自項目的語彙單元，則指向該項目的位置。

## 6.9.10. 偵錯

`nabic build --emit=expansions`（[9.9.10](../009-tools/909-debug-tools.md#9910-meta-區塊偵錯)）把使用者套件的展開結果以原始碼文字寫到 `target/<profile>/build/<name>.expanded.nabi`（或 `-o` 路徑）。每個展開是一行 ``// expanded from this `#[derive(X)]` (file:line:col)`` 註解，後接產生的原始碼；若輸出又被展開，它接在外層展開之後。

## 6.9.11. 總結

樣板記號是 Nabi 元系統的程式碼產生詞彙。它們雖然簡單，卻涵蓋了實用 derive 與屬性的所有核心情況。

Nabi 的選擇是以限制語彙單元轉換的彈性，換取可除錯性、工具友善性與有保證的衛生性。

第 6 章（元系統）到此結束。
