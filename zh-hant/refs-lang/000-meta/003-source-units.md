# 0.3. 原始碼單元

Nabi 的編譯單元是**套件**。檔案系統上的套件依固定慣例（`src/main.nabi` / `src/lib.nabi`）辨識。清單 `nabi.nabion` 只在宣告相依套件等資訊時才存在；沒有相依套件的套件不需要它（0.3.4）。

## 0.3.1. 執行檔與函式庫

套件包含一個執行檔、一個函式庫，或兩者兼有。包含哪一種，只由其原始碼目錄 `src/` 中的根檔案決定：`src/main.nabi` 是執行檔的根，`src/lib.nabi` 是函式庫的根。

套件目錄在 `src/` 中存放套件的原始碼，其餘一切放在它旁邊：清單 `nabi.nabion` 與鎖定檔 `nabi.lock`（0.3.4）、`target/`（建置產出，[9.2.15](../009-tools/902-aot-pipeline.md#9215-輸出)），以及 `README`、`LICENSE` 或測試資料等檔案。只有 `src/` 底下的檔案才是套件的原始碼；`src/` 之外的 `.nabi` 檔案不屬於套件（它不會一起編譯，不計入套件識別碼（0.3.3），單獨執行時它是一個匿名套件，[4.16.8](../004-functions/416-entry-point.md#4168-單一檔案執行)）。唯一的例外是 `tests/` 與 `benches/` 目錄，它們只在 `nabic test` / `nabic bench` 時編譯（[0.3.5](#035-測試與基準測試目錄)）。

### 可執行套件：`src/main.nabi`

包含 `src/main.nabi` 的目錄被辨識為可執行套件。

```
my_app/
├── nabi.nabion     <- manifest (optional, 0.3.4)
├── src/
│   ├── main.nabi   <- entry point
│   └── ...         <- other module files
└── target/         <- build output
```

`main.nabi` 是進入模組，以兩種形式之一提供進入點：明確的 `fn main` 或頂層陳述式。進入點的確切形式與頂層陳述式的意義定義於 [4.16 進入點](../004-functions/416-entry-point.md)。執行不屬於任何套件的單一檔案時，該檔案被視為匿名套件的 `main.nabi`（[4.16.8](../004-functions/416-entry-point.md#4168-單一檔案執行)）。

### 函式庫套件：`src/lib.nabi`

包含 `src/lib.nabi` 的目錄被辨識為函式庫套件。

```
my_lib/
├── nabi.nabion
└── src/
    ├── lib.nabi    <- library root
    └── ...
```

函式庫套件沒有進入點；它作為一組供外部使用的定義。除了 `main.nabi` 之外的每個模組檔案（包括 `lib.nabi`）只能包含項目，不得包含頂層陳述式，因此匯入模組絕不會執行任何程式碼（[4.16.6](../004-functions/416-entry-point.md#4166-模組只含定義)）。

### 同一套件中的函式庫與執行檔

套件可以同時有 `src/main.nabi` 和 `src/lib.nabi`。它們是從同一個 `src/` 建置出的兩個目標：函式庫，以及使用該函式庫的執行檔。執行檔把函式庫看成以套件命名的相依套件（`use my_app::parse;`；即 [9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock) 的程式碼名稱，`-` 換成 `_`）。函式庫與執行檔是不同的套件：在執行檔中只看得到它的 `pub` 項目，看不到 `pub(pkg)`、`pub(sub)` 或私有項目（[5.7.14](../005-modules/507-paths.md#5714-套件之間的路徑)），而且在執行檔 `use` 它之前，它的任何項目都不在作用域內。兩個目標中的 `pkg::` 各自是該目標自己的根。每個目標有自己的模組樹，因此兩者都可以為同一個 `src/util.nabi` 宣告 `mod util;`，並各自得到一份複本。相依於這個套件的套件只會得到函式庫。

## 0.3.2. 檔案與模組

套件內的額外模組以 `src/` 底下檔案的兩種形式之一存在。

```
foo.nabi            <- single-file module (no submodules)
```

或者可以附帶一個資料夾：

```
foo.nabi            <- module body
foo/                <- foo's submodules
├── bar.nabi        foo::bar submodule
└── baz.nabi        foo::baz submodule
```

在資料夾形式中，主體檔案（`foo.nabi`）與同名資料夾（`foo/`）並排。只有在主體中以 `mod name;` 宣告的子模組才會對應到資料夾中的檔案。這與 Rust 2018+ 的規則相同；詳細的對應規則定義於[模組](../005-modules/504-modules.md)。

此外，也可以不用另外的檔案，而以 `mod name { ... }` 或 `pub mod name { ... }` 的形式在主體中放置行內模組。它的行為與 Rust 的行內模組相同；其詳細形式與對應同樣定義於[模組](../005-modules/504-modules.md)。

## 0.3.3. 套件識別碼

每個套件都被賦予一個基於內容的雜湊。雜湊的輸入是套件所有原始檔的串接，依其相對於 `src/` 的路徑（`/` 分隔）的位元組順序排序，每個檔案序列化為 `path + \0 + contents + \0`。取其 SHA-256 的前 8 個字元作為識別碼。

這個雜湊具有下列性質：內容相同的套件無論在哪裡建置都產生相同的雜湊，而即使只改動一個字元也會產生新的雜湊。

雜湊用於產出的符號名稱與快取鍵；具體用途定義於後續文件。

## 0.3.4. 清單（`nabi.nabion`）是選用的

套件由什麼構成，只由檔案是否存在決定。若目錄中存在 `src/main.nabi` 或 `src/lib.nabi`，該目錄就是一個套件，而其中哪些存在決定它包含執行檔、函式庫，或兩者兼有。清單不會改變這個慣例。

只有在需要相依套件、開發用相依套件或工具鏈固定等資訊時，才在套件根目錄（`src/` 旁邊）放置清單 `nabi.nabion`（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）；工具解析出的結果記錄在鎖定檔 `nabi.lock` 中。這兩個檔名的拼寫完全如此（全部小寫）。大小寫不同的名稱（`Nabi.nabion` 等）會產生警告。兩個檔案都使用 [NABION](../../refs-standard-libraries/300-std/308-nabion.md) 格式；清單只會被讀取為單一的 `Package { .. }` 值，絕不會被編譯或執行。

這個決定帶來兩件事。第一，編譯器只看檔案就能處理沒有相依套件的套件。第二，清單純粹是附加的，絕不會破壞既有的套件定義。

## 0.3.5. 測試與基準測試目錄

套件可以在 `src/` 旁邊放置 `tests/` 與 `benches/` 目錄。與 `src/` 之外的其他檔案（0.3.1）不同，它們會被編譯，但只用於對應的指令：`tests/` 用於 `nabic test`，`benches/` 用於 `nabic bench`。它們絕不會出現在一般建置、函式庫介面（`.nabihir`）或套件識別碼（0.3.3）中，這些全都只涵蓋 `src/`。

每個目錄都掛載為一個合成的根模組（`tests/` 位於模組 `tests` 之下，`benches/` 位於 `benches` 之下），依[模組](../005-modules/504-modules.md#544-套件根)的一般規則配置：`tests/parse.nabi` 是 `tests::parse`，其旁邊的 `tests/parse/` 資料夾存放該模組的子模組。沒有 `mod.nabi`（[5.4.6](../005-modules/504-modules.md#546-自訂路徑path)）；命名方式與 `src/` 底下相同。

這些程式碼屬於套件內部：它與套件一起編譯，並透過 `pkg::` 存取套件，具有與 `src/` 底下模組相同的套件內可見性（而非相依套件那種只有 `pub` 的視角），因此可以測試 `pub(pkg)` 項目以及套件自身的非公開模組。它可以使用清單的 `dev_deps`（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)）。在同時有 `src/lib.nabi` 與 `src/main.nabi` 的套件中，這兩個目錄針對函式庫建置；執行檔自己的測試則以行內方式留在 `src/` 底下。

這裡的檔案隱含地只用於測試或基準測試，因此不需要 `#[cfg(test)]`。以 `#[test]` 或 `#[bench]` 標記要執行或量測的函式（[6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-測試函式標記)）；目錄中的其他項目是共用的輔助程式。

```
my_lib/
├── nabi.nabion
├── src/
│   └── lib.nabi
├── tests/              <- nabic test only
│   ├── parse.nabi      tests::parse   (uses pkg::...)
│   └── common.nabi     tests::common  (shared helper, no #[test])
└── benches/            <- nabic bench only
    └── parse.nabi      benches::parse
```
