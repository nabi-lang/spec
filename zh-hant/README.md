# Nabi 語言規格

[English](../en/README.md) · [한국어](../ko/README.md) · **繁體中文**

> **狀態**：進行中的草稿，描述 Nabi 0.1。規格以三個版本發布：英文（`en/`，預設）、韓文（[`ko/`](../ko/README.md)）與繁體中文（`zh-hant/`）。各版本以相同的節號涵蓋相同的內容。

Nabi 是一門程式語言，本儲存庫是它的形式規格。從語彙結構與型別系統開始，它在同一處定義語言的每一層：運算式、執行環境模型、ABI，以及隨規格附帶的工具。編譯器與標準函式庫的實作位於另一個儲存庫；本儲存庫只存放「Nabi 是什麼」這個問題的規範性答案。

由 [Han Damin](https://github.com/miniex)（`@miniex`）設計與維護。

| 項目                 | 位置                                                    |
|----------------------|---------------------------------------------------------|
| GitHub 組織          | [nabi-lang](https://github.com/nabi-lang)                 |
| 規格儲存庫           | <https://github.com/nabi-lang/spec>                      |
| 網站                 | <https://nabi-lang.org>                                  |
| 文件                 | <https://docs.nabi-lang.org>                             |
| 套件登錄庫           | [packages.nabi-lang.org](https://packages.nabi-lang.org)        |
| 登錄庫索引           | <https://github.com/nabi-lang/packages-index>          |

## Nabi 為何存在

Nabi 從作者想寫的一種程式碼出發。它不鎖定特定的使用者族群，也不主張某種學術立場。每個設計決定的核心都是一個單純的動機：「一門能做到作者想做的一切的語言」。這句話大致就是 Nabi 的全部。

## 哲學

出於個人動機而誕生的語言，需要在每次決定時都能依靠的原則，才能保持一致。兩句標語引導 Nabi 的設計。

> [!IMPORTANT]
> **「核心窄小，函式庫豐富。」**

第一個原則是關於規格大小的承諾。Nabi 規格定義得越少越好。幾乎所有塑造語言*手感*的東西（數值語意、轉換、集合型別、I/O、並行）都在標準函式庫中。小的核心更容易演進、驗證與教學；每個領域的重量則改由豐富的函式庫承擔。

> [!IMPORTANT]
> **「規則越清楚，在規則之內就越自由。」**

第二個原則關乎自由與限制的關係。Nabi 不迴避限制。靜態型別、封閉的和型別、trait 不能作為型別、沒有隱含提升：每一個決定都刻意縮小使用者能走的路。在那個較窄的空間裡，程式碼變得可預測，也能深入控制。

控制與自由之間的界線是清楚的。能一次決定、不必在每一行重新決定的事（型別、分派、記憶體模型、求值順序），由語言一次全部決定。必然因人而異的事（函式庫設計、領域建模、演算法、程式碼結構）留給使用者，而元系統讓語言能擴充自身，使函式庫可以決定意義。

因此自由與控制並不對立。Nabi 的立場是：自由的領域只有在控制良好的核心之上才能安全地存續。兩句標語的完整討論延續於 [`refs-lang/000-meta/`](refs-lang/000-meta/)。

## 結構

儲存庫有固定的形狀。頂層目錄依語言版本劃分，每個版本包含兩份參考：

- [`refs-lang/`](refs-lang/)：語言參考，涵蓋語彙結構、型別、運算式、執行環境模型、ABI、工具。
- [`refs-standard-libraries/`](refs-standard-libraries/)：標準函式庫參考。

每個版本都遵循這個結構：`en/refs-lang/`、`ko/refs-lang/` 等，檔案與節號都相同。

## 版本

規格有三種語言版本：

- **English**（`en/`）：預設版本。
- **한국어**（[`ko/`](../ko/README.md)）：韓文。
- **繁體中文**（`zh-hant/`）：臺灣使用的繁體中文。

各版本使用相同的檔案結構與節號（`3.11.4`），因此同一個節號在任何版本中都指同一條規則。若版本之間不一致，以英文版為準，而該差異是一個應當在所有版本中修正的錯誤。

## 閱讀順序

規格的寫法是從第 0 章讀到第 9 章。新概念一律只用截至當時已引入的概念來解釋，而這個不變條件在每一章都成立。無論打開哪一章，它需要的概念都來自該章本身或之前的章節。有些交叉引用指向後面完整討論該主題之處；它們僅供參考，章節並不相依於它們。初次閱讀者應當從 [`refs-lang/README.md`](refs-lang/README.md) 開始。

## 慣例

程式碼圍欄內的 ASCII 與框線圖使用英文標籤，並在其下方的文字中說明。程式碼範例在各版本中完全相同，只有註解、附帶的字串，以及代表使用者自行撰寫部分的文法佔位符（後設變數）可能經過翻譯。本繁體中文版的程式碼區塊與英文版逐位元組相同。

## 術語

下列術語在英文版中一貫使用（列出繁體中文版的對應譯名，供譯者參考）。[詞彙表](refs-lang/000-meta/007-glossary.md)定義核心術語。

| English | 繁體中文 |
|---|---|
| specification, chapter, section | 規格、章、節 |
| language reference / standard library reference | 語言參考 / 標準函式庫參考 |
| sum type | 和型別 |
| variant | 變體 |
| trait; trait-as-type ban | trait；trait 不能作為型別 |
| associated type / constant | 關聯型別 / 常數 |
| monomorphization; concrete | 單態化；具體的 |
| slot; `any` slot; lang slot ("magic") | 槽位；`any` 槽位；lang 槽位（魔法） |
| implicit promotion / conversion | 隱含提升 / 隱含轉換 |
| conversion / cast | 轉換 / 強制轉型 |
| expression / statement; statement terminator | 運算式 / 陳述式；陳述式終止符 |
| block; body | 區塊；主體 |
| arm (of `match`) / branch | 分支 / 分支 |
| pattern; binding; destructuring (assignment) | 模式；繫結；解構（指派） |
| optional chaining / coalescing | 可選鏈結 / 空值合併 |
| broadcast | 廣播 |
| literal; identifier; keyword; reserved word | 字面值；識別字；關鍵字；保留字 |
| boolean; sentinel; floating-point; integer | 布林值；哨兵值；浮點數；整數 |
| punctuation; trailing comma | 標點；尾隨逗號 |
| parameter / argument | 參數 / 引數 |
| closure; capture; coroutine; async | 閉包；擷取；協程；非同步 |
| entry point | 進入點 |
| module; visibility; path; item | 模組；可見性；路徑；項目 |
| package; registry | 套件；登錄庫 |
| manifest; lockfile; environment; dependency; pin | 清單；鎖定檔；環境；相依套件；固定 |
| meta-system; attribute; template tokens | 元系統；屬性；樣板記號 |
| runtime; object; root; barrier; generation | 執行環境；物件；根；屏障；世代 |
| lowering; dialect; symbol | 降階（lowering）；方言；符號 |
| error; "is an error" / "is a compile error"; diagnostic | 錯誤；「是錯誤」/「是編譯錯誤」；診斷 |
| raw pointer; collection; iterator | 原始指標；集合型別；迭代器 |
| evaluation order; incremental build | 求值順序；增量建置 |

已經是英文的名稱在各版本中保持原樣：NABION、`nabi.nabion`、`nabi.lock`、`nabic`、`nabi`、`nabicc`、`nabils`、`nabir`、NABIHIR、HIR、MIR、LIR、AOT、JIT、GC、Immix、LOS、TLAB、shadow stack、`scan_table`、niche optimization、sticky mark-bit、work-stealing、ABI、FFI、toolchain、sysroot。

## 授權條款

本規格以 [Creative Commons Attribution 4.0 International (CC BY 4.0)](../LICENSE) 散布。

Copyright © 2026 DaminStudio and the Nabi contributors

只要註明出處，即可自由分享、改編與翻譯本文。完整的授權條款見 [LICENSE](../LICENSE)；易讀的摘要見 <https://creativecommons.org/licenses/by/4.0/>。

## 商標

「Nabi」是本專案的名稱，也是正式規格的識別。CC BY 4.0 只涵蓋**文字**：只要註明出處，文字本身可以自由複製、翻譯與改編。不過，請不要把「Nabi」這個名稱用於分叉的規格、替代規格或相互競爭的實作。請使用不同的名稱，以免與正式專案混淆。

這是一項**請求**，而非註冊商標。若專案成長到需要正式保護的程度，將會考慮註冊。
