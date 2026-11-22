# 0.7. 詞彙表

同樣的概念在本規格的許多章節中反覆出現，因此把常用術語集中在一處對讀者有幫助。本文件收集了貫穿 Nabi 規格的核心術語，讓讀者在閱讀本文時一旦對某個概念不清楚，就能直接查閱。每個詞條都給出簡短的定義以及它在 Nabi 中的具體脈絡，不只說明字面意義，也指出該術語在規格中所占的位置。

## ABI（應用程式二進位介面）

ABI 是編譯後的程式碼在連結與執行時必須遵守的二進位層級契約。它包括物件配置、呼叫慣例、符號命名等。Nabi 的 ABI 定義於[第 8 章：ABI / 降階](../008-abi/README.md)。

## AOT（預先編譯）

AOT 是預先把原始碼編譯成靜態產出（`.a` 或執行檔）的模式，與 JIT 相對。Nabi 同時支援 JIT 與 AOT 引擎，兩者共用直到 nabi-mir 為止的同一條管線。

## 封存檔（archive）

封存檔是編譯後套件的產出：標準 ar 格式的 `.a` 檔案。它存放具體程式碼的目的檔；函式庫的介面（NABIHIR）則是旁邊的 `<name>.nabihir` 檔案。

## 區塊（block）

區塊是以 `{ ... }` 括起來的一組陳述式與運算式。它本身是一個運算式，其最後一個運算式成為區塊的值。

## 區塊（Immix block）

Immix 區塊是 GC 的記憶體單位，是堆積中 32 KiB 的區域。在 Immix 演算法中，它被劃分為若干行（line），並作為標記 / 清除的單位。它與上面的程式碼區塊是不同的概念。

## 具體函式 / 泛型函式（concrete fn / generic fn）

具體函式是在呼叫處所有型別都已具體的函式；泛型函式是仍有剩餘型別參數的函式。在 AOT 下，泛型函式會在呼叫處被特化為具體函式。

## 分派（dispatch）

分派是決定名稱、運算子或方法呼叫繫結到哪個實作的過程。除了 `any` 槽位中的值會在執行時依其標籤分派之外（[2.2.3](../002-types/202-any-slot.md#223-dynamic-dispatch)），分派都是靜態的；由於 trait 不能作為型別，沒有任何值帶有 vtable 指標。

## extern "C" 符號

extern "C" 符號是透過 C ABI 公開或呼叫的符號；這類符號構成編譯器與執行環境之間的介面。

## FFI（外部函式介面）

FFI 是呼叫其他語言（主要是 C）的函式，或被其他語言呼叫的介面。它以 `extern "C"` 區塊定義。

## GC（垃圾回收器）

GC 是自動記憶體管理機制。Nabi 的 GC 結合了 Immix mark-region 與 Sticky Mark-Bit 分代回收。

## 泛型（generic）

泛型是接受型別作為參數的函式或型別。它在呼叫處與實例化處被單態化。

## HIR（高階 IR）/ MIR（中階 IR）

HIR 與 MIR 是編譯管線的中間表示。整體流程是原始碼 → AST → HIR → MIR → 機器碼（Nabi 的程式碼產生器）。

## Immix

Immix 是 Blackburn 與 McKinley 於 2008 年發表的 GC 演算法。它以 32 KB 區塊與 128 B 行為單位使用 mark-region，並以機會式疏散（opportunistic evacuation）處理碎片化。

## JIT（即時編譯）

JIT 是在執行時以 Nabi 的程式碼產生器立即編譯程式碼的模式。REPL 與 Jupyter 使用這種模式；它與 AOT 相對。

## lang 槽位 / 魔法

lang 槽位是語言用來辨識特定函式庫項目的機制。`#[lang(slot_name)]` 把標準函式庫項目連接到編譯器的槽位。

## LOS（大型物件空間）

LOS 是 GC 專供大型物件使用的空間。8 KiB 以上，或對齊大於 8 位元組的物件，保存在 LOS 中且不會被移動（[7.2](../007-runtime/702-allocation.md)）。

## 單態化（monomorphization）

單態化是在呼叫處為每個具體型別分別編譯泛型函式的過程。它以編譯時間與二進位檔大小為代價，換得零間接呼叫成本。

## NABIHIR（Nabi HIR 位元組碼）

NABIHIR 是預先建置之函式庫套件的介面檔案（`<name>.nabihir`）。它位於封存檔旁邊，讓下游套件能讀取宣告並特化泛型函式。它的內容是去除了具體函式主體的套件原始碼（[9.4](../009-tools/904-nabihir-bytecode.md)）。

## niche 最佳化（niche optimization）

niche 最佳化是利用和型別或列舉中未使用的位元模式來省去標籤槽位的最佳化。典型的例子是 `Ptr | none` 中的 `none = null pointer`。

## scan_table

scan_table 是 GC 用來找出物件中哪些槽位是 GC 指標的中繼表。它以 `type_id → scan_table` 對應，並由編譯器在建置時註冊。

## 影子堆疊（shadow stack）

影子堆疊是 GC 用來追蹤存活物件的輔助堆疊。編譯器直接產生程式碼，在函式進入與離開時，把存放 GC 物件變數槽位的框架連上與解開。框架位於機器堆疊上，鏈的開頭則以每個執行緒一份的方式保存在 TLS 中。

## 特化（specialize）

特化是[單態化](#monomorphization)的另一個名稱。它被用作 AOT / JIT 管線中某個階段的名稱。

## 登錄庫套件（registry package）

發布到登錄庫的 Nabi 套件。其名稱為 `[a-z0-9_-]+`（以字母開頭），在程式碼中 `-` 寫成 `_`。登錄庫把 `-` / `_` 以及大小寫不同視為相同的名稱。在 `nabi.nabion` 中以 `Dep::version("req")` 或字串 `"req"` 引用（[9.10.4a](../009-tools/910-nabi-frontend.md#9104a-registry-packages)）。

## sticky mark-bit

sticky mark-bit 是分代 GC 的世代追蹤位元。它在第一次標記時被設定，使物件成為「老物件」。年輕代 GC 會略過 sticky 物件以降低成本。

## 和型別（sum type）

和型別是 `A | B` 形式、容納其中一個選項的型別；它是帶標籤聯集的一種形式。它是封閉的槽位：所有變體都是明確的，因此能對模式比對做窮盡性檢查。

## TLAB（執行緒區域配置緩衝區）

TLAB 是每個工作執行緒擁有的執行緒區域記憶體緩衝區。快速路徑的配置只用一個 bump 指標完成，同步成本為零。

## 工作竊取（work-stealing）

工作竊取是排程器的負載平衡機制。每個工作者都有一個區域雙端佇列，工作用完的工作者會從另一個工作者的雙端佇列尾端竊取任務。

## unsafe 區塊 / unsafe 函式

`unsafe { ... }` 區塊運算式與 `unsafe fn` 函式修飾詞，是 Nabi 用來把超出安全抽象保證的區域明顯地圈起來的唯一機制。原始指標解參照、呼叫 `unsafe fn` 等只允許在 unsafe 作用域內進行。詳見 [4.15 unsafe](../004-functions/415-unsafe.md)。

## nabic / nabi / nabils / nabir

構成工具鏈的執行檔與執行環境。`nabic` 是編譯器（相當於 rustc，[9.1](../009-tools/901-nabic-cli.md)），擁有自己的程式碼產生器與連結器；`nabi` 是類似 cargo 的前端，負責清單、相依套件、環境與工具鏈固定，呼叫 `nabic`，並執行連結了編譯器的各種工具（run、lint、REPL、Jupyter、LSP、格式化工具……）（[9.10](../009-tools/910-nabi-frontend.md)）；`nabicc` 是 C 編譯器（`nabi cc`，[9.13](../009-tools/913-c-compiler.md)）；`nabils` 是供編輯器使用的 LSP 伺服器執行檔（`nabi lsp`）；`nabir` 是執行環境（`libnabir.a`）。`$NABI_HOME/bin/` 中的執行檔是指向作用中工具鏈（`$NABI_HOME/toolchains/<version>-<triple>/`）的代理，`nabic` 則在相對於自身執行檔的 sysroot 中尋找標準函式庫與執行環境（[9.2.19](../009-tools/902-aot-pipeline.md#9219-installation-layout)）。

## NABION

Nabi Object Notation。以 Nabi 字面值語法的子集撰寫值的資料格式，副檔名為 `.nabion`。清單 `nabi.nabion` 與鎖定檔 `nabi.lock` 使用這種格式（[stdlib 3.8](../../refs-standard-libraries/300-std/308-nabion.md)）。

## packages.nabi-lang.org

Nabi 的預設套件登錄庫，位於 `https://packages.nabi-lang.org`。其索引是獨立的 git 儲存庫 `packages-index`（[9.10.4a](../009-tools/910-nabi-frontend.md#9104a-registry-packages)）。

## nabi.nabion / nabi.lock

位於套件根目錄的清單與鎖定檔。名稱的拼寫完全如此（全部小寫），兩者都使用 [NABION](#nabion) 格式。沒有相依套件的套件不需要它們（[0.3.4](003-source-units.md#034-the-manifest-nabinabion-is-optional)）。`nabi.nabion` 以單一的 `Package { .. }` 值宣告相依套件與工具鏈固定，`nabi.lock` 則固定 `nabi update` 解析出的結果。

## 環境（environment）

`nabi` 保存一組相依套件的單位。每個包含 `nabi.nabion` 的目錄都是一個環境，而且沒有啟用步驟。在專案之外，使用 `$NABI_HOME/envs/` 中具名的環境（預設為 `default`）（[9.10.5](../009-tools/910-nabi-frontend.md#9105-environments)）。

這裡收集的詞條不只是字詞的翻譯；它們也構成 Nabi 立足之基礎的精簡索引：編譯管線、記憶體管理、型別系統、並行模型等。閱讀本文時一旦概念變得模糊，回到本章就能釐清每個術語在整份規格中所占的位置。
