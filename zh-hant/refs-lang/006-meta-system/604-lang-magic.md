# 6.4. `#[lang(...)]` 魔法

`#[lang(slot_name)]` 屬性把標準函式庫的項目連接到編譯器直接辨識的槽位。它是編譯器與執行環境得知哪個項目帶有哪種意義的管道。

## 6.4.1. 動機

如[語言與標準函式庫的邊界](../000-meta/004-language-vs-standard-libraries.md)所述，語言只提供 `+` 與 `-` 之類的語法，由標準函式庫決定其意義。要讓這個模型運作，編譯器必須知道哪個函式庫 trait 或型別連接到哪種語言意義。

```
compiler slot           ← language-known anchor
                          (e.g. "add trait", "option slot", "iterator trait")

library item            ← trait / type / fn defined by standard libraries

  ←── #[lang(slot)] connects the two sides ──→
```

## 6.4.2. 形式

```
lang_attr       ::= "#" "[" "lang" "(" string_lit ")" "]"
```

```nabi
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

#[lang("iterator_trait")]
trait Iterator { ... }

#[lang("display_trait")]
trait Display { ... }              // the default f"{x}" format
```

它寫成 `#[lang("slot_name")]`，以字串字面值引數接受槽位名稱。它可以附加在 trait、型別、函式、`const` 與 `static` 等項目上。

## 6.4.3. 槽位名稱的分類

槽位名稱構成編譯器事先知道的固定集合；使用者不能建立新的槽位。

具代表性的槽位：

| 槽位                    | 所連接項目的種類       | 編譯器的用途                      |
|-------------------------|------------------------|-----------------------------------|
| `"add_trait"`           | trait                  | `+` 運算子去語法糖                |
| `"sub_trait"`           | trait                  | `-` 運算子去語法糖                |
| `"mul_trait"`           | trait                  | `*` 運算子去語法糖                |
| ...                     | ...                    | ...                               |
| `"none_type"`           | 型別                   | `none` 字面值的型別               |
| `"iterator_trait"`      | trait                  | `for` 迴圈去語法糖                |
| `"into_iterator_trait"` | trait                  | `for` 迴圈去語法糖                |
| `"future_trait"`        | trait                  | `await` 去語法糖                  |
| `"index_trait"`         | trait                  | `[i]` 索引去語法糖                |
| `"slice_trait"`         | trait                  | `[1:3]` 切片去語法糖              |
| `"display_trait"`       | trait                  | `f"{x}"` 格式化                   |
| `"debug_trait"`         | trait                  | `f"{x:?}"` 偵錯格式               |
| `"clone_trait"`         | trait                  | `#[derive(Clone)]`                |
| `"eq_trait"`            | trait                  | `==` / `!=` 運算子（`PartialEq`） |
| ...                     | ...                    | ...                               |

完整的槽位清單以及附加於各槽位的項目，以[標準函式庫 0.4 `#[lang]` 槽位對應表](../../refs-standard-libraries/000-meta/004-lang-slot-table.md)為權威；上表只是摘錄。槽位集合的變更從 0.4 的表格開始。

## 6.4.4. 對應的效果

當某個 trait 帶有 `#[lang("add_trait")]` 時，編譯器把該 trait 連接到加法的意義。之後的 `+` 運算子去語法糖如下：

```
a + b    →    <typeof(a) as the_trait<typeof(b)>>::add(a, b)
```

這裡的 `the_trait` 是帶有 `#[lang("add_trait")]` 的 trait。

## 6.4.5. 在使用者程式碼中的使用（罕見）

`#[lang]` 幾乎只由標準函式庫使用。使用者程式碼只有在 _no_std_ 環境中建立自己的函式庫，或在嵌入式系統或作業系統等必須重新定義語言核心的環境中，才會直接附加它。一般使用者永遠不需要直接附加 `#[lang]`。

## 6.4.6. 每個槽位一個項目

把同一個槽位名稱附加在多個項目上是編譯錯誤。

```nabi
#[lang("add_trait")] trait MyAdd { ... }
#[lang("add_trait")] trait OtherAdd { ... }    // ERROR: duplicate slot
```

每個槽位恰好對應一個項目。這種唯一性在整個程式範圍內（跨所有連結的套件）成立。因此使用者套件不能重新附加並取代 `core` 所定義的槽位（`panic_fn`、`oom_fn`、`unreachable_fn` 等）；那樣做是編譯錯誤。

## 6.4.7. 槽位的簽章檢查

每個槽位都預先決定了它所接受之項目的簽章。

```nabi
#[lang("add_trait")]
trait Add<Rhs = Self> {              // the form the slot requires
    type Output;
    fn add(self, rhs: Rhs) -> Output;
}
```

若簽章與槽位要求的形式不符，就是編譯錯誤。

```nabi
#[lang("add_trait")]
trait BadAdd { fn run() }            // ERROR: does not match the form of the 'add_trait' slot
```

## 6.4.8. 未知的槽位名稱

```nabi
#[lang("nonexistent_slot")]
trait T { ... }
```

使用未知的槽位名稱同樣是編譯錯誤。

## 6.4.9. 槽位的穩定性

槽位名稱的集合本身是語言規格的一部分，因此新增、變更或移除槽位會改變語言版本（SemVer）。

## 6.4.10. 魔法與一般屬性的差異

| 面向          | `#[derive]`、`#[inline]` 等             | `#[lang]`                       |
|---------------|-----------------------------------------|---------------------------------|
| 由誰附加      | 使用者                                  | 標準函式庫（大多數）            |
| 效果          | 程式碼產生或編譯器提示                  | 繫結到語言意義                  |
| 槽位或鍵      | 自由（以屬性名稱識別）                  | 固定的槽位集合                  |

`#[lang]` 是元系統中最具特權的屬性。其他魔法（`#[literal]`、`#[keyword]` 與運算子魔法）都可以視為 `#[lang]` 的細化形式。

## 6.4.11. `#[lang]` 與其他魔法的關係

運算子魔法（`#[infix]` 等）通常附加在帶有 `#[lang(...)]` 之 trait 的方法上。

```nabi
#[lang("add_trait")]
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}
```

這裡的 `#[lang("add_trait")]` 宣告這個 trait 是加法意義所在之處，而 `#[infix("+", ...)]` 把該方法標示為 `+` 語彙單元的分派目標。兩種魔法一起運作，雙向連接語言與函式庫。

`#[literal]` 與 `#[keyword]` 同樣可以理解為分別連接到字面值位置與關鍵字註冊槽位的工具。
