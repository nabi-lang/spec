# 6. 元系統概觀

這一部分定義 Nabi 的元系統：屬性、魔法與元[區塊](../000-meta/007-glossary.md#block)。元系統是實現[語言與標準函式庫之邊界](../000-meta/004-language-vs-standard-libraries.md)模型的核心機制：語言提供小的核心，標準函式庫填入其意義。

## 目錄

- [6.1. 屬性語法](601-attribute-syntax.md)
- [6.2. 內建屬性](602-built-in-attributes.md)
- [6.3. `#[cfg]` 條件式編譯](603-cfg.md)
- [6.4. `#[lang(...)]` 魔法](604-lang-magic.md)
- [6.5. `#[literal(...)]` 魔法](605-literal-magic.md)
- [6.6. `#[keyword(...)]` 魔法](606-keyword-magic.md)
- [6.7. 運算子魔法](607-operator-magic.md)
- [6.8. 使用者定義的 meta 定義（元區塊與巨集）](608-meta-blocks.md)
- [6.9. 樣板記號與展開](609-template-tokens.md)
- [6.10. 反射](610-reflection.md)
- [6.11. 編譯期檔案嵌入](611-include.md)

## 三層結構

元系統有三個層級：

```
attribute
  ├── plain attribute: hints / metadata for compiler / tools
  └── magic: channel where the library binds to language meaning

meta definition: user-defined code generator, one template engine (6.8)
  ├── derive / attribute / per-field  (triggered by #[..] on a type or item)
  └── macro                           (triggered by a call, name!(..))
      └── template token (@Self, @field, ..) + matcher (typed fragments)
```

### 屬性

屬性看起來像這樣：

```nabi
#[derive(Debug, Clone)]
struct Point { x: f64; y: f64; }

#[inline(always)]
fn fast() { ... }

#[cfg(target_os = "linux")]
fn linux_only() { ... }
```

屬性以 `#[...]` 或 `#![...]` 的形式附加在項目上。編譯器辨識的內建屬性（例如 `#[derive]`、`#[repr]`、`#[cfg]`）與使用者定義的屬性共用相同的形式。詳細形式定義於[屬性語法](601-attribute-syntax.md)與[內建屬性](602-built-in-attributes.md)。

### 魔法

魔法指的是連接語言與函式庫的特殊屬性。

```nabi
#[lang("slot_name")]            // connects a standard library item ↔ a compiler slot
#[literal("...")]                // registers literal syntax
#[keyword("name")]               // registers a new keyword
```

此外還有運算子魔法家族。

```nabi
#[infix("+")]
#[prefix("-")]
#[infix("⊕", like = "+")]
```

這些魔法把 `1 + 2` 之類的語言語彙單元連接到函式庫的 trait 方法呼叫。它們從 [`#[literal]` 魔法](605-literal-magic.md)到[運算子魔法](607-operator-magic.md)逐一定義。

### 元區塊

meta 定義是程式碼產生器。有三種形式讓使用者定義自己的 derive 或屬性（取代 Rust 的 derive 與屬性 proc macro）；第四種是[巨集](608-meta-blocks.md#6812-the-macro-form)，由呼叫 `name!(..)` 觸發，並從呼叫者寫出的內容產生程式碼（類函式的情況）。四種形式共用一個樣板引擎。

```nabi
#[impl_derive(MyTrait)] pub @derive {
    impl MyTrait for @Self {
        @for_fields {
            // per-field code generation
        }
    }
}
```

共有四種形式：由屬性觸發的 `#[impl_derive]`、`#[impl_attribute]` 與 `#[impl_derive_attribute]`，以及由呼叫觸發的 `#[impl_macro(name)] @macro { .. }`（以 `name!(..)` 叫用）。它們的寫法都相同：`#[impl_<kind>(name)]`、可見性、`@` 引導符與一個區塊。其主體以 `@Self`、`@field` 與 `@for_fields` 等樣板記號撰寫程式碼產生規則，而巨集還多了由型別化片段組成的匹配器（`@x: expr`）。不提供 proc macro 風格的語彙單元串流操作；單靠樣板記號就有足夠的表達力。完整定義見 [meta 定義](608-meta-blocks.md)。

## 核心原則

元系統建立在四項核心原則之上。

### 1. 不操作語彙單元串流

Nabi 沒有 Rust 的 `syn` 或 `quote` 那種過程式（procedural）、操作語彙單元的巨集。所有使用者程式碼產生，無論是由屬性觸發的元區塊，還是由呼叫觸發的巨集（[6.8.12](608-meta-blocks.md#6812-the-macro-form)），都只依靠模式與樣板：匹配器繫結輸入的部分，樣板把它們轉錄出來。沒有任何步驟會執行任意程式碼或改寫原始的語彙單元串流。

理由是讓產生的程式碼保持在編譯器、格式化工具與語言伺服器都能追蹤的狀態。允許任意的語彙單元轉換會帶來取捨：不可預測、難以除錯，以及工具介面的爆炸。巨集在遵守這條規則的同時，加入了呼叫處的程式碼產生（其他形式無法做到的事）。

### 2. 魔法由標準函式庫使用

`#[lang]`、`#[literal]`、`#[keyword]` 與運算子魔法主要由標準函式庫使用；使用者程式碼很少直接使用它們。

這些魔法是標準函式庫與語言繫結以產生意義的機制；[語言與標準函式庫的邊界](../000-meta/004-language-vs-standard-libraries.md)透過這個元系統實現。

### 3. 巨集衛生

元區塊中的程式碼產生是衛生的。樣板寫出的名稱在定義處、在元區塊所屬的模組中解析；使用處傳入的名稱在使用處解析；樣板的區域變數會被改名，因此絕不會擷取或遮蔽使用者的名稱。展開所產生的項目放在使用處的模組中，在那裡它們的名稱是可見的（[6.9.9](609-template-tokens.md#names-in-an-expansion)）。

這防止了使用者程式碼與元區塊之間不可預測的衝突。

### 4. 只在編譯期

所有 meta 處理都在編譯期完成。不支援在執行時註冊新的 derive 或注入新的魔法。
