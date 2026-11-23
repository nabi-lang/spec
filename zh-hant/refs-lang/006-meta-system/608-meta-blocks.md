# 6.8. 使用者定義的 meta 定義

**meta 定義**是使用者定義的程式碼產生器。它們全都是同一種機制：定義繫結輸入的各部分，並轉錄由 Nabi 程式碼與樣板記號（[6.9](609-template-tokens.md)）構成的樣板，以衛生的方式（[6.8.9](#689-不操作語彙單元串流)）進行到固定點（[6.8.8](#688-固定點展開)）。它們的差別只在於如何被*觸發*以及繫結什麼輸入。共有四種形式；前三種由項目上的屬性觸發，第四種由呼叫觸發：

| 形式                                    | 定義為                             | 觸發方式               | 繫結                                    |
|-----------------------------------------|------------------------------------|------------------------|-----------------------------------------|
| derive                                  | `#[impl_derive(T)] @derive {}`     | 型別上的 `#[derive(T)]` | 型別的形狀（`@Self`、`@for_fields`）  |
| 屬性                                    | `#[impl_attribute(n)] @attribute {}` | 項目上的 `#[n]`      | 該項目（`@item`）                       |
| 逐欄位的 derive 屬性                    | `#[impl_derive_attribute(T, a)] @derive {}` | `#[derive(T)]` + 欄位上的 `#[a]` | 形狀 + 欄位屬性        |
| **巨集**                                | `#[impl_macro(n)] @macro { (m) => {t} }` | `n!(..)` 呼叫      | 呼叫的語彙單元（`@x: expr`、`@rep`）    |

前三種*從作者附加它們的型別或項目*產生程式碼；巨集則*從呼叫者寫出的內容*產生程式碼，這是其他形式無法滿足的唯一需求（[6.8.12](#6812-巨集形式)）。它們共用樣板引擎、`@` 記號、衛生性以及下文的工具保證；本章與 [6.9](609-template-tokens.md) 會指出所繫結的輸入在何處不同。這四種形式是 Nabi 全部的超程式設計介面：它們一起取代了 Rust 的 derive、屬性與類函式巨集，而不需要過程式（procedural）程式碼或原始的語彙單元操作。

```
meta_block      ::= meta_attr visibility? meta_kind block
meta_attr       ::= "#" "[" meta_kind_attr "(" meta_args ")" "]"
meta_kind_attr  ::= "impl_derive" | "impl_attribute" | "impl_derive_attribute" | "impl_macro"
meta_kind       ::= "@derive" | "@attribute" | "@macro"
meta_args       ::= identifier ("," identifier)*

# the @macro block holds rules instead of a flat template:
macro_body      ::= macro_rule+
macro_rule      ::= "(" matcher ")" "=>" block
```

## 6.8.1. 由屬性觸發的形式

三種由屬性觸發的形式：

```nabi
#[impl_derive(TraitName)] pub @derive {
    impl TraitName for @Self { ... }
}

#[impl_attribute(name)] pub @attribute {
    @item                                    // reuse the original item
    fn additional_helper() { ... }            // generate an additional item
}

#[impl_derive_attribute(Trait, attr)] pub @derive {
    @for_fields {
        @apply_if(@has_derive_attr(attr)) {
            // process only fields carrying that attr
        }
    }
}
```

各形式的用途：

| 形式                                    | 用途                                       |
|-----------------------------------------|--------------------------------------------|
| `#[impl_derive(TraitName)]`             | 使用者定義的 derive                        |
| `#[impl_attribute(name)]`               | 使用者定義的屬性（項目包裝）               |
| `#[impl_derive_attribute(Trait, attr)]` | 逐欄位的 derive 屬性                       |

第四種，由呼叫觸發的 `macro`，定義於 [6.8.12](#6812-巨集形式)。

## 6.8.2. `#[impl_derive(TraitName)]` 使用者定義的 derive

使用者定義之 derive 的範例：

```nabi
#[impl_derive(MyHash)] pub @derive {
    impl MyHash for @Self {
        fn my_hash(self) -> u64 {
            let h: u64 = 0u64;
            @for_fields {
                h = combine_hash(h, self.@field.my_hash());
            }
            h
        }
    }
}

// use site
#[derive(MyHash)]
struct Point { x: f64; y: f64; }
// → generated automatically.
//   impl MyHash for Point {
//       fn my_hash(self) -> u64 {
//           let h: u64 = 0u64;
//           h = combine_hash(h, self.x.my_hash());
//           h = combine_hash(h, self.y.my_hash());
//           h
//       }
//   }
```

這種形式由 `#[impl_derive(name)]` 與一個主體（樣板區塊）組成。當使用處出現 `#[derive(name)]` 時，樣板會針對該型別展開以產生程式碼。

## 6.8.3. `#[impl_attribute(name)]` 使用者定義的屬性

使用者定義之屬性的範例：

```nabi
#[impl_attribute(timed)] pub @attribute {
    @accept(fn)                                  // attachable only to fn items

    fn @item.name(@item.params) -> @item.return_type {
        let start = Instant::now();
        let result = {
            @item.body
        };
        log(f"@item.name took {Instant::now() - start}");
        result
    }
}

// use site
#[timed]
fn process(x: i32) -> i32 {
    expensive(x)
}
// → generated automatically.
//   fn process(x: i32) -> i32 {
//       let start = Instant::now();
//       let result = { expensive(x) };
//       log(f"process took {Instant::now() - start}");
//       result
//   }
```

這是項目的包裝：它重用原始項目（`@item`）的內容與簽章，產生轉換後的形式。它不只能附加在函式上，也能附加在 `struct`、`enum` 與 `impl` 上；目標的限制以 `@accept` 表達。

## 6.8.4. `#[impl_derive_attribute(Trait, attr)]` 逐欄位的 derive 屬性

逐欄位 derive 屬性的範例：

```nabi
#[impl_derive_attribute(Serialize, rename)] pub @derive {
    impl Serialize for @Self {
        fn serialize(self) -> Json {
            let obj = JsonObj::new();
            @for_fields {
                @apply_if(@has_derive_attr(rename)) {
                    obj.set(@derive_attr(rename), self.@field.serialize());
                }
                @apply_if(!@has_derive_attr(rename)) {
                    obj.set(@field.name.as_str(), self.@field.serialize());
                }
            }
            obj.into()
        }
    }
}

// use site
#[derive(Serialize)]
struct User {
    #[rename("user_name")]
    name: str;
    age: i32;
}
// → generated automatically.
//   impl Serialize for User {
//       fn serialize(self) -> Json {
//           let obj = JsonObj::new();
//           obj.set("user_name", self.name.serialize());
//           obj.set("age", self.age.serialize());
//           obj.into()
//       }
//   }
```

這種形式接受兩個引數：被衍生的 Trait，以及標示逐欄位資訊的屬性。它在 derive 主體內讀取欄位的屬性資訊來分支。

該屬性可以以 `attr(k, ..)` 的形式宣告它接受的鍵：`#[impl_derive_attribute(Nabion, nabion(deny_unknown_fields))]`。此時型別、其變體與其欄位上的每個 `#[attr(..)]` 都只能列出已宣告的鍵；其他任何東西都是錯誤，錯誤訊息會指出有效的鍵。`@Self.has_derive_key(k)` 從型別讀取某個鍵（[6.9.6](609-template-tokens.md#696-derive_attribute-記號)）。

## 6.8.5. 主體（樣板區塊）

在每種形式中，主體都是同時含有一般 Nabi 程式碼與樣板記號的樣板區塊（由屬性觸發的形式用 `@Self`、`@field`、`@for_fields`；巨集用 `@x`、`@rep`、`@for`、`@concat`，[6.8.12](#6812-巨集形式)）。

樣板記號的完整清單定義於[樣板記號與展開](609-template-tokens.md)。

## 6.8.6. `pub` / 可見性

可見性的寫法如下：

```nabi
pub @derive { ... }
pub(pkg) @attribute { ... }
@derive { ... }                  // private
```

可見性關鍵字附加在樣板區塊本身上。其效果：

| 可見性                        | 可以使用的位置                                             |
|-------------------------------|------------------------------------------------------------|
| `pub`（包括 `pub(pkg)`）      | 整個建置。其他套件以本套件的名稱撰寫它：`#[derive(dep::X)]` / `#[dep::x]` / `dep::x!(..)`（6.8.11） |
| 私有                          | 定義該元區塊的檔案內                                       |

meta 名稱沒有模組路徑：名稱屬於註冊它的元區塊所在的套件，在該套件中不帶前綴撰寫，在其他套件中則加上該套件的名稱（6.8.11）。元區塊的樣板看得到其模組看得到的東西：它所寫出的名稱在那裡解析並接受存取檢查（[6.9.9](609-template-tokens.md#展開中的名稱)）。元區塊或關鍵字處理器的 `#[cfg]` 以定義它之套件的 cfg 求值（其自身的功能特性，[9.10.4c](../009-tools/910-nabi-frontend.md#9104c-feature)）。

## 6.8.7. 主體引導符（`@derive` / `@attribute`）

主體引導符表示樣板的種類。

| 引導符       | 用途                                                      |
|--------------|-----------------------------------------------------------|
| `@derive`    | derive 定義（`impl_derive`、`impl_derive_attribute`）     |
| `@attribute` | 屬性定義（`impl_attribute`）                              |
| `@macro`     | 巨集定義（`impl_macro`）；其區塊容納規則（[6.8.12](#6812-巨集形式)） |

引導符決定哪些樣板記號在主體中有意義。例如 `@item` 只用於屬性，而匹配器記號（`@x: expr`、`@rep`）只用於 `@macro`。

## 6.8.8. 固定點展開

若樣板展開的結果產生了帶有另一個 derive 或屬性的項目，該項目會被遞迴地展開。

```nabi
#[impl_attribute(adds_helper)] pub @attribute {
    @item                                       // the original item as is

    #[derive(MyHelper)]                          // new item: derive processed again
    struct Helper { ... }
}
```

套用這個屬性時，主體中的 `#[derive(MyHelper)]` 會在下一個展開階段處理。

當展開到達沒有任何 derive 或屬性需要展開的固定點時，展開就結束。

循環展開（A 叫用 B，而 B 又叫用 A）會被編譯器偵測到，是錯誤。訊息會顯示循環路徑（`A → B → A`）。

## 6.8.9. 不操作語彙單元串流

每個 meta 定義（三種由屬性觸發的形式以及巨集）都只用樣板記號產生程式碼。Nabi 沒有 Rust 的 `syn` 或 `quote` 那樣的任意語彙單元轉換功能，也沒有在編譯期執行程式碼的過程式形式。

這項限制保證 meta 定義的輸出是編譯器能理解的 Nabi 程式碼，同時確保工具友善性與可偵錯性。

## 6.8.10. 與 proc macro 的比較

比較 Rust 的 proc macro 與 Nabi 的元區塊：

| 面向            | Rust proc macro                | Nabi 元區塊                                      |
|-----------------|--------------------------------|-------------------------------------------------|
| 輸入            | 任意的語彙單元串流             | 項目的結構資訊（欄位、變體等）                  |
| 輸出            | 任意的語彙單元串流             | Nabi 程式碼與樣板記號                            |
| 衛生            | 取決於 proc macro              | 自動（衛生的）                                  |
| 偵錯            | 檢視輸出的語彙單元             | 檢視展開後的 AST                                |
| 表達力          | 非常高                         | 有限                                            |

Nabi 的選擇以表達力換取可預測性與偵錯的便利。

## 6.8.11. 與註冊 / 可見性的結合

帶有 `#[impl_derive(TraitName)]` 的元區塊註冊名為 `TraitName` 的 derive，`#[impl_attribute(name)]` 註冊 `#[name]` 屬性，`#[impl_macro(name)]` 註冊巨集 `name!`。編譯器在剖析之前從函式庫與使用者套件的每個檔案收集元區塊，因此無論定義位置或檔案順序為何都可以使用它們。若元區塊上的 `#[cfg]` 為假，該區塊不會被註冊（[6.9.9](609-template-tokens.md#與-cfg-的關係)）。

meta 名稱屬於註冊它的元區塊所在的套件。在該套件中，它不帶前綴撰寫（`#[derive(MyHash)]`、`#[x]`、`x!(..)`），或帶 `pkg::` 撰寫。其他套件在它之前寫出定義套件的名稱，即與該套件之路徑相同的根：`#[derive(dep::MyHash)]`、`#[dep::x]`、`dep::x!(..)`。限定詞只有套件；元區塊不是命名空間中的項目，也沒有模組路徑。

模組層級的 `use dep::x!;` 匯入巨集 `x!`，因此該檔案以 `x!(..)` 呼叫它。它不匯入任何項目，也不能被改名（`as`）或重新匯出（`pub use`）。對與某個 derive 同名之項目（通常是其 trait）的 `use`（`use dep::MyHash;`）也會匯入該 derive，而 `use dep::*;` 匯入 `dep` 的每個 meta 名稱。

```nabi
// package json
#[impl_macro(log)] pub @macro { ... }
#[impl_derive(MyHash)] pub @derive {
    impl pkg::MyHash for @Self { ... }
}

// a package that depends on json
use json::log!;

#[derive(json::MyHash)]
struct Foo { ... }

fn main() {
    json::log!("start");
    log!("imported");
}
```

單獨的名稱解析為本套件的區塊，否則是 `use` 所匯入的那一個，否則是標準函式庫的（其 meta 名稱保持不帶前綴：`#[derive(Inspect)]`）。否則就是錯誤，錯誤訊息會列出限定形式：`` `log!` is package `json`'s macro, which a bare name does not reach ``，說明 ``write `json::log!`, or add `use json::log!;` ``。兩個套件的 `use` 都匯入該名稱時就有歧義，也是錯誤。在展開內，樣板所寫出的 meta 名稱從定義它的套件與檔案依相同規則解析，因此相依套件的巨集無論在哪裡展開都能繼續運作；使用處傳入的名稱則從使用處解析（[6.9.9](609-template-tokens.md#展開中的名稱)）。

下列是錯誤：

- 一個套件的兩個元區塊註冊相同的 derive、屬性或巨集名稱（一個巨集是容納其所有規則的一個 `@macro` 區塊）。兩個不同的套件可以各自註冊同一個名稱。
- 被指名的套件沒有註冊的限定巨集、屬性或 derive（`dep::x!`、`#[dep::x]`、`#[derive(dep::X)]`）。
- 元區塊重新定義內建 derive（[6.2.1](602-built-in-attributes.md)：`Clone`、`Copy`、`Default`、`Debug`、`PartialEq`、`Eq`、`PartialOrd`、`Ord`、`Hash`、`Error`）或內建屬性（`cfg`、`derive`、`test`、`inline`、`repr`、`keyword`、`bench`、`path`，以及 6.2 到 6.7 的其他屬性）的名稱。說明會建議一個新名稱（`#[impl_attribute(my_bench)]`）。

## 6.8.12. 巨集形式

**巨集**是由呼叫觸發的 meta 定義。derive 或屬性從它所附加的型別或項目產生程式碼，而巨集像 `name!(..)` 那樣被*叫用*，並在該位置展開為程式碼，從呼叫者寫出的內容產生程式碼。這是由屬性觸發的形式無法滿足的唯一需求：由呼叫而非由型別的欄位驅動的重複或產生程式碼。

### 定義

和其他形式一樣，巨集由 `#[impl_macro(name)]`、可見性、`@macro` 引導符與一個區塊組成。區塊容納一條或多條**規則**，而不是單一的樣板：每條規則是括號中的匹配器、`=>`（與 [`match`](../003-expressions/312-match-expression.md) 分支相同的箭頭），以及大括號中的樣板。

```nabi
#[impl_macro(name)] pub @macro {
    ( <matcher> ) => { <template> }
    ( <matcher> ) => { <template> }
    ..
}
```

`#[impl_macro(name)]` 註冊巨集 `name!`，與 `#[impl_derive(T)]` 註冊 derive `T` 完全相同。呼叫時，規則由上到下嘗試，第一個匹配器相符的規則會被展開；若沒有任何一個相符，就是呼叫處的編譯錯誤。與 derive 或屬性的名稱一樣，巨集的名稱屬於其套件：在那裡是 `name!`，在其他套件中則是 `dep::name!` 或 `use dep::name!;`（[6.8.11](#6811-與註冊--可見性的結合)）。每次呼叫結尾的 `!` 標示它是巨集而不是函式。

```nabi
#[impl_macro(twice)] pub @macro {
    ( @e: expr ) => { { let v = @e; (v, v) } }
}

let p = twice!(compute());      // (the value, the same value)
```

### 叫用

巨集以 `name!( .. )`、`name![ .. ]` 或 `name!{ .. }` 呼叫，從另一個套件則以 `dep::name!( .. )` 呼叫；分隔符可以互換，不改變意義。引數是任意的**語彙單元樹**：帶有平衡之 `()` `[]` `{}` 的任何語彙單元，因此即使不認識該巨集，語彙分析器與格式化工具也能切分呼叫。巨集可以在下列位置展開，取決於其樣板產生的是哪一種：

| 位置       | 範例                                      | 樣板產生                |
|------------|-------------------------------------------|-------------------------|
| 運算式     | `let xs = grid![1, 2; 3, 4];`             | 一個運算式              |
| 陳述式     | `debug!("here");`                         | 零或多個陳述式          |
| 項目       | `int_impl!(i8, i16, i32);`（模組層級）    | 零或多個項目            |
| 型別       | `let f: fn_ptr!(i32, i32) = ..;`          | 一個型別                |
| 模式       | `match x { pair!(a, b) => .. }`           | 一個模式                |

在運算式位置，呼叫是基本運算式：其展開是一個運算元，就像放在括號中一樣，後綴形式接在它之後（`grid![1, 2; 3, 4].len()`、`table!{ .. }.head(5)`、`first!(xs)?`）。

### 匹配器與片段

匹配器是語彙單元模式。字面語彙單元必須原樣出現在呼叫中；**後設變數** `@name: <fragment>` 為樣板擷取呼叫的一部分。片段決定擷取多少個語彙單元：`ident` 取一個識別字，`literal` 取一個字面值語彙單元，`tt` 取一個語彙單元樹（一個語彙單元或一個平衡的分組）。`expr`、`stmt`、`block`、`ty`、`pat`、`path` 與 `item` 取到匹配器在括號深度 0 的下一個字面語彙單元為止（或到呼叫或外圍分組的結尾為止）的非空語彙單元序列。匹配器中括號括起的分組，`( .. )`、`[ .. ]` 或 `{ .. }`，比對呼叫中的平衡分組，而其內部作為自己的匹配器比對：`[ @rep[sep = ","] { @x: expr } ]` 比對 `[5, 6]`。每個片段都以原始的語彙單元擷取，比對時不剖析；整個展開在轉錄之後才剖析，因此在樣板所放置之處無效的引數，是展開中的語法錯誤，在呼叫處回報。

```nabi
#[impl_macro(clamp)] pub @macro {
    ( @x: expr , @lo: expr , @hi: expr ) => {
        { let v = @x; if v < @lo { @lo } else if v > @hi { @hi } else { v } }
    }
}
```

### 重複

`@rep` 比對子模式零次以上。`@for` 為每個元素轉錄一次；它與 `@for_fields`（[6.9.3](609-template-tokens.md#693-欄位迭代)）是同一種迭代器，這裡是走訪擷取的序列。

```nabi
#[impl_macro(list)] pub @macro {
    ( @rep[sep = ","] { @x: expr } ) => {
        { let acc = List::new(); @for @x { acc.push(@x); } acc }
    }
}

let xs = list![a, b, c];        // acc.push(a); acc.push(b); acc.push(c);
```

`@rep[sep = ","] { <sub-pattern> }` 比對以 `sep` 語彙單元分隔的重複子模式（`sep` 是選用的；`trailing` 也允許一個尾隨的分隔符）。在 `@rep` 內擷取的後設變數繫結一個序列。`@for @x { <body> }` 為每個元素發出主體；來自同一個 `@rep` 的多個後設變數同步迭代，且長度必須相等。`@rep` 與 `@for` 可以巢狀：位於兩層 `@rep` 內的後設變數繫結序列的序列，而內層的 `@for @x` 走訪外層目前的元素（`(@rep[sep = ";"] { @rep[sep = ","] { @x: expr } }) => { [@for @x { [@for @x { @x, }], }] }` 把 `1, 2; 3` 轉成 `[[1, 2], [3]]`）。形狀記號（`@Self`、`@field`、`@apply_if`……）屬於由型別驅動的形式，而不屬於巨集：巨集沒有可以測試的形狀，因此使用它們是錯誤。

### 名稱合成

`@concat(..)` 把名稱語彙單元與字串字面值接合成一個識別字，因此巨集可以產生各自不同、具名的項目：

```nabi
#[impl_macro(accessors)] pub @macro {
    ( @s: ident ; @rep[sep = ","] { @f: ident : @t: ty } ) => {
        impl @s { @for @f { fn @concat(get_, @f)(self) -> @t { self.@f } } }
    }
}
// accessors!{ P; x: i32, y: i32 } -> impl P { fn get_x(self) -> i32 { self.x } .. }  (and get_y)
```

由 `@concat` 產生的名稱會被**記錄**：編譯器以其產生的名稱為每個合成的項目建立索引，因此符號搜尋、跳到定義與尋找參照都能到達它。`@concat` 只在定義或路徑位置形成識別字；其片段是名稱語彙單元與字串字面值，它不能組裝任意的語法。

### 函式庫的樣板程式碼

使用巨集的典型理由，是在作者寫出的固定清單上重複一個定義，這是任何由屬性觸發的形式都做不到的。它取代了 [`core::num`](../../refs-standard-libraries/200-core/README.md) 中手動複製的逐型別 impl：

```nabi
#[impl_macro(nonzero_div)] pub @macro {
    ( @rep[sep = ","] { @t: ty } ) => {
        @for @t {
            impl Div<NonZero<@t>> for @t {
                type Output = @t;
                const fn div(self, rhs: NonZero<@t>) -> @t { self / rhs.0 }
            }
        }
    }
}

nonzero_div!(i8, i16, i32, i64, u8, u16, u32, u64);
```

### 遞迴

樣板可以呼叫其他巨集；它們的呼叫會依序展開。巨集不得呼叫自己：在其自身的展開內（直接或經由其他巨集）呼叫該巨集，會立即以指出該呼叫的編譯錯誤拒絕（`` `name!` expands too deeply or cyclically ``）。不同巨集的巢狀展開限制在 256 層，超過時是同樣的錯誤。要在清單上重複，請使用 `@rep` / `@for` 而不是遞迴。

### 內建的 `!` 形式是保留的

`cfg!`（[6.3](603-cfg.md)）與 `include_str!` / `include_bytes!`（[6.11](611-include.md)）以 `!` 結尾，但它們是固定的編譯器內建形式而不是巨集，不能被重新定義、遮蔽或用作巨集名稱。它們是編譯器提供的僅有 `!` 形式；其他每個 `name!` 都是以 Nabi 定義的巨集。

### 工具與衛生

巨集保有每個 meta 定義的保證（[6.8.9](#689-不操作語彙單元串流)），並加上呼叫處所帶來的細節：

- **衛生**是自動的：樣板所繫結的名稱（`let` 或 `for` 迴圈的變數）是全新的，不能擷取呼叫處的名稱，也不能被呼叫處擷取；而透過後設變數傳入的名稱在呼叫處解析。樣板所寫出的其他任何名稱，例如 `impl Tr for E` 標頭中的型別或匯入的 `Complex`，都指向巨集定義處的項目。
- **位置範圍**：每個展開後的語彙單元都帶有呼叫的位置範圍，因此診斷會指向該呼叫，並把展開作為附註顯示。
- **解析**：展開被當作一般程式碼剖析，合成的名稱也會被建立索引，因此懸停、跳到定義與尋找參照都能到達巨集所產生的項目。
- **依要求展開**：語言伺服器可以就地展開一個呼叫（[9.9.13](../009-tools/909-debug-tools.md)）。
- **格式化**：當呼叫的引數是 Nabi 片段時，格式化工具會格式化它們；原始的 `tt` DSL 則在其平衡的分隔符內保持原樣。

## 6.8.13. 總結

這四種形式（derive、屬性、逐欄位的 derive 屬性以及由呼叫觸發的巨集）構成 Nabi 全部的超程式設計介面，建立在同一個樣板引擎之上。它們缺少過程式巨集的表達力，但涵蓋了核心的使用情境：derive、屬性、逐欄位屬性以及呼叫處的程式碼產生，每一種都是衛生的，而且工具都能追蹤。

[樣板記號與展開](609-template-tokens.md)定義樣板記號的確切清單（包括由型別驅動與由呼叫驅動的輸入）以及展開機制。
