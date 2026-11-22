# 9.2. AOT 建置管線

本章定義 `nabic build` 的逐步流程。它以指令與檔案流向，具體化[執行引擎與使用者介面](../000-meta/001-execution-modes.md)與[編譯管線](../000-meta/002-pipeline.md)中給出的整體樣貌。

## 9.2.1. 步驟

建置分 11 個步驟進行，全部都在 `nabic` 內。

```
1. package discovery       (find package root)
2. dependency resolve      (resolve deps to sources or prebuilt .nabihir + .a)
3. source parse            (.nabi → AST)
4. name / visibility       (resolve)
5. type check              (type check)
6. nabi-hir build           (HIR build)
7. specialize              (collect call-sites + monomorphize)
8. nabi-mir build
9. code generation         (optimized at -O1 and above)
10. archive / link         (nabic's linker)
11. artifact .a / executable
```

> 依序為：套件探索、相依套件解析、原始碼剖析、名稱 / 可見性解析、型別檢查、nabi-hir 建立、特化（收集呼叫處與單態化）、nabi-mir 建立、產生目的檔、封存或連結，最後是輸出：`.a` 封存檔或執行檔。

每個步驟的輸入、輸出與責任都清楚分開。

## 9.2.2. 套件探索

從 `nabic build [path]` 的路徑開始往上搜尋，找出含有 `src/main.nabi` 或 `src/lib.nabi` 的目錄（若 path 是檔案，則從其所在的目錄開始）；搜尋每次往上移動一層目錄。

找到時，該目錄就是套件根，其 `src/` 是套件的原始碼（見[原始碼單元](../000-meta/003-source-units.md)）。其他情況：

- 若 path 是位於所找到之套件 `src/` 之外的檔案（套件原始碼旁邊的腳本或工具），或往上走也找不到套件，它就是由該單一檔案組成的匿名可執行套件（[4.16.8](../004-functions/416-entry-point.md#4168-single-file-execution)）。其檔案子模組位於它旁邊。
- 若 path 是目錄且沒有套件，則是錯誤。

套件名稱是套件目錄的名稱（對匿名套件而言，是去掉 `.nabi` 的檔案名稱），並成為輸出名稱（9.2.15）。兩個根檔案都有的套件建置為兩個目標：先是其函式庫，然後是其執行檔，後者把該函式庫掛載為相依套件 `<name>`（[0.3.1](../000-meta/003-source-units.md#library-and-executable-in-one-package)）。

## 9.2.3. 相依套件解析

這個步驟處理[路徑](../005-modules/507-paths.md)與 [`use`](../005-modules/505-use.md) 中對外部套件的參照（`use external_pkg::...`）。

`nabic` 本身不解析相依套件。前端 `nabi`（[9.10](910-nabi-frontend.md)）依照 `nabi.nabion` / `nabi.lock` 準備路徑 / git 相依套件，並經由 `--extern` 把每個套件的根傳給 `nabic`。

```
nabic build --extern util=../util --extern parser=target/debug/deps/libparser.a
```

`--extern name=<path>` 把一個相依套件掛載在頂層路徑 `name`。`name` 是把 `nabi.nabion` 中 `deps` 的鍵裡的 `-` 換成 `_` 所得到的識別字。路徑有兩種形式。

| 路徑                              | 相依套件                                                                     |
|-----------------------------------|------------------------------------------------------------------------------|
| `<dir>` / `<dir>/src/lib.nabi`     | 原始碼。該 `src/lib.nabi` 在同一個建置中一起編譯                              |
| `<name>.nabihir` / `lib<name>.a`     | 預先建置的套件。給出其中一個時，會找到同一目錄中的另一個檔案。讀取 NABIHIR（[9.4](904-nabihir-bytecode.md)）取得宣告與泛型，並連結封存檔 |

- 使用者程式碼經由 `use name::..` 使用該套件的項目。
- 相依套件內部的 `pkg::` 指稱該相依套件本身。
- 相依套件是獨立的套件。只有 `pub` 項目可以從外部使用；`pub(pkg)` / `pub(sub)` / 私有項目不行（[5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)）。
- 使用者套件根部的模組與建置中某個套件同名是錯誤。
- 路徑與先前某個 `--extern` 相同的後續 `--extern`，是同一個套件的另一個名稱（各套件可以用不同的名稱稱呼一個相依套件，[9.10.2](910-nabi-frontend.md)）。該套件只以第一個名稱建置並掛載一次；以另一個名稱開頭的路徑（`other::..`，在以該名稱稱呼它之套件的程式碼與 meta 區塊範本中）指稱它，因此其項目與型別在每個名稱下都相同。
- `--extern-dep <package>=<name>` 表示套件 `<package>`（一個 `--extern` 名稱，或以 `pkg` 表示正在建置的套件）相依於 extern `<name>`；`nabi` 為每個套件的每個相依套件傳入一個。有任何一個時，每個套件只看得到它所相依的 extern，並以它給它們的名稱稱呼（[5.7.14](../005-modules/507-paths.md#5714-paths-between-packages)）；沒有任何 `--extern-dep` 指名的 extern（套件自己的函式庫，由 `nabic` 掛載）屬於正在建置的套件，並相依於它所相依的東西。沒有任何一個時，每個套件都看得到每個 extern。

對有 `src/lib.nabi` 的套件，`nabic build` 產生 `lib<name>.a` 與 `<name>.nabihir`；有 `src/main.nabi` 時產生執行檔 `<name>`；兩者都有的套件產生全部三個，執行檔對照該函式庫連結。`--name <n>` 設定套件名稱（輸出名稱與掛載名稱）；預設是目錄名稱。

### 預先建置的相依套件

`nabic` 依給出的順序，把每個原始碼 `--extern` 預先建置**一次**到 `target/[<triple>/]<profile>/deps/`。每個相依套件在它之前的那些之後建置，並看得到它所相依的那些（`--extern-dep`；沒有時則是全部）。後續的編譯使用這些 `lib<name>.a` 與 `<name>.nabihir`，而不是原始碼。

```
target/<profile>/deps/
├── lib<name>.a
├── <name>.nabihir
└── <name>.fp               # fingerprint
```

若指紋相符，就不會重新建置。指紋包括：

- `nabic` 的身分（與 NABIHIR 產生者相同的值）
- 建置選項：`--release` / `-O*`、`-g`、`--target`、`--cfg`（不含其 `feature` 鍵）、`--bootstrap`
- 該相依套件的 feature（`--extern-feature`），以及它和在它之前建置之套件的 `--extern-dep`
- 該相依套件 `src/` 下的所有檔案，以及其 `nabi.nabion`
- 在它之前建置之相依套件的指紋

預先建置之套件的介面以它建置時的 feature 讀取（其 NABIHIR meta，[9.4.6](904-nabihir-bytecode.md#946-meta)），而不是使用端的 feature。

每個會編譯的指令都使用這條路徑：`build` / `run` / `test` / `bench` / `check` / `lint` / `doc` / `lsp` / `repl` / `jupyter`、單一檔案腳本，以及具名環境。REPL 主機也會連結預先建置的封存檔。若既沒有套件也沒有目標目錄（`--target-dir`、`NABI_TARGET_DIR`），它們會放在系統暫存目錄中的單獨檔案快取下（9.2.16）。

標準函式庫在每次建置時都從原始碼編譯，只編譯程式所觸及的部分；其已編譯的函式保存在工具鏈的函式快取中（[9.3.6](903-package-archive.md#936-standard-library)）。

## 9.2.4. 原始碼剖析

所有 `.nabi` 檔案依照從[原始碼檔案](../001-lexical/101-source-files.md)到[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)的詞法 / 語法規則剖析成 AST。

[模組](../005-modules/504-modules.md)中的目錄模組把其所有 .nabi 檔案合併為一個模組的主體。

套件根部的資訊清單 `nabi.nabion`（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）不是模組檔案。資訊清單被讀取為單一的 [NABION](../../refs-standard-libraries/300-std/308-nabion.md) 值（`Package { .. }`）；頂層陳述式的規則不適用於它，它也不會被編譯。

### 掃描器預先處理

[`#[keyword]` 魔法](../006-meta-system/606-keyword-magic.md)的掃描器預先處理也在這個步驟進行。所有 .nabi 檔案先被淺層掃描，收集 `#[keyword(...)]`、`#[infix(...)]`、`#[literal(...)]` 的註冊，再把這些註冊交給真正的詞法分析器 / 剖析器。

## 9.2.5. 名稱 / 可見性解析

每個路徑與識別字都依照 [`use`（匯入）](../005-modules/505-use.md)、[可見性](../005-modules/506-visibility.md)、[路徑](../005-modules/507-paths.md)與[重新匯出（`pub use`）](../005-modules/508-re-export.md)的規則對應到一個項目。

這個步驟套用隱含的 `core::prelude` 匯入（見[標準函式庫層級](../000-meta/005-standard-libraries-tiers.md)；`bare::*` 與 `std::*` 需要明確匯入，因此排除在外），進行可見性檢查並把不當的存取回報為錯誤，並檢查匯入衝突。

## 9.2.6. 型別檢查

每個運算式的型別都依照 [2. 型別系統](../002-types/README.md)的規則決定。

這包括決定漸進式槽位與靜態槽位（見 [`any` 槽位](../002-types/202-any-slot.md)）、檢查 trait 約束（見[泛型](../002-types/210-generics.md)），以及固定各分支的結果型別（見 [3. 運算式](../003-expressions/README.md)）。和型別與 `any` 槽位的型別收窄也在這裡處理。

## 9.2.7. nabi-hir 建立

經過型別檢查的 AST 被轉換成 nabi-hir（[HIR](../000-meta/007-glossary.md#hir-high-level-ir--mir-mid-level-ir)）資料結構。

HIR 是富含型別資訊的中間表示，反映上述步驟 4-5 的所有結果。它在結構上經過壓縮，讓相同的型別與識別字共用同一個表示。

## 9.2.8. specialize：單態化

進行[泛型](../002-types/210-generics.md)與[常數泛型](../002-types/211-const-generics.md)的單態化。

```
1. Collect all call sites.
2. Determine the concrete type tuple of each call.
3. Create a separate specialized HIR per (item_id, type_tuple).
4. Cache. Each (item_id, type_tuple) only once.
```

從程式的根（`main`、匯出的符號、`#[export_name]` 項目、執行環境所參照的符號）無法到達的函式不會被發出。函式庫建置也是如此：從套件的 `pub` 項目與那些根無法到達的函式不會放進封存檔。

這個步驟讓 [AOT](../000-meta/007-glossary.md#aot-ahead-of-time) 能夠不經 vtable 進行直接呼叫（見[禁止把 trait 當作型別](../002-types/212-trait-as-type-banned.md)）。

## 9.2.9. nabi-mir 建立

[特化](../000-meta/007-glossary.md#specialize)後的 HIR 被轉換成 nabi-mir（MIR）。

MIR 是比 HIR 低一層的 IR。在這個步驟中，運算子去糖完成（見[運算子分派](../003-expressions/301-operator-dispatch.md)），模式比對的決策樹被決定（見 [`match` 運算式](../003-expressions/312-match-expression.md)），閉包與協程的框架配置也被固定。

## 9.2.10. 最佳化等級

最佳化等級決定 nabi-mir 在程式碼產生之前如何最佳化（[8.7.3](../008-abi/807-code-generation.md#873-optimization-levels)）；程式碼產生器在每個等級都相同（[0.2.4](../000-meta/002-pipeline.md#024-code-generation)）。

| 等級          | 效果 |
|---------------|--------|
| `-O0`         | 不最佳化；每個值都在其堆疊槽位中 |
| `-O1`         | 值經過簡化並放在暫存器中；迴圈與 `-O2` 一樣最佳化（其中的呼叫與 `-O2` 一樣內嵌，因此 `for` 迴圈的迭代器會變成計數迴圈，經過分段並向量化），其他地方內嵌小函式與 `#[inline]` 函式，並在各處內嵌只從一處呼叫的函式；帶有偵錯資訊（`debug` 設定檔） |
| `-O2`         | 另外在迴圈之外也內嵌較大的主體 |
| `-O3`         | 另外在迴圈中內嵌更大的主體，並展開保持純量的迴圈：保有浮點數總和或呼叫函式之迴圈的完整一段，每 2 或 4 次迭代才測試一次其計數器 |
| `-Os`         | 只在主體大約與呼叫一樣小之處（在迴圈中則是其數倍）內嵌，迴圈仍會變成計數迴圈並分段，主體短時會向量化 |

`-g` 既不改變最佳化也不改變程式碼（8.7.3）：偵錯資訊跟隨最佳化後的程式碼，內嵌的主體作為框架，每個變數位於暫存器、其堆疊槽位中，或被最佳化掉（[9.9.6](909-debug-tools.md#996-debugger-integration-gdb--lldb)）。最佳化器在套件被分割成 codegen 單元（9.2.16）之前一次看到整個套件，因此它會跨模組內嵌，也會內嵌進入與移出標準函式庫的函式，而標準函式庫在每次建置時都與程式一起編譯（[9.3.6](903-package-archive.md#936-standard-library)）。以不同等級建置的目的檔遵循相同的執行環境 ABI（[8](../008-abi/README.md)），並能連結在一起，包括以其他等級建置的預先建置相依套件。REPL 以 `:opt` 所設定的等級編譯儲存格（[9.9.9](909-debug-tools.md#999-repl--jupyter-debugging)），不最佳化其 nabi-mir。

## 9.2.11. 程式碼產生器

程式碼產生器（[8.7](../008-abi/807-code-generation.md)）經由所有目標共用的一種低階形式，直接從 nabi-mir 產生機器碼與目的檔，支援 `x86_64`、`aarch64`、i686、armv7、riscv64、riscv32、powerpc64le、s390x 與 wasm32。有 `-g` 時，它發出 DWARF：函式、行號表、參數、區域變數及其型別，以及在最佳化後的程式碼中，被內嵌的主體與每個變數依程式碼範圍所在的位置（[9.9.6](909-debug-tools.md#996-debugger-integration-gdb--lldb)）；對 Windows 目標，它在目的檔中以 CodeView（`.debug$S`、`.debug$T`）發出相同的內容，由連結器彙整成 PDB（9.2.14）。使用了它尚不支援之東西的程式碼是指出其名稱的 `E0129` 錯誤。

## 9.2.12. 目的檔

每個目標的目的檔都由 `nabic` 寫出：Apple 目標是 Mach-O，Linux、FreeBSD 與 riscv32 是 ELF，Windows 是 COFF，wasm32 是 wasm 目的檔。建置的 codegen 單元（9.2.16）平行編譯，每個單元一個目的檔。`--emit=obj` 寫出目的檔而不連結它（[9.9.1](909-debug-tools.md#991-intermediate-output---emit)）。

```
target/debug/build/foo.o
```

## 9.2.13. 封存檔

函式庫套件把其目的檔（每個 codegen 單元一個，9.2.16）放進由 `nabic` 自己寫出的標準 `ar` 封存檔，並在其旁邊寫出介面檔 `foo.nabihir`（[NABIHIR 格式](904-nabihir-bytecode.md)）。

```
target/debug/libfoo.a
target/debug/foo.nabihir
```

## 9.2.14. 連結（執行檔）

執行檔由 `nabic` 自己的連結器連結：

```
foo.o + <deps>/lib*.a + <sysroot>/lib/libnabir.a + target libc  →  target/debug/foo
```

Mach-O 執行檔對照 macOS SDK（Xcode Command Line Tools）連結。Windows 執行檔對照 `nabic` 從它所攜帶之模組定義（衍生自 mingw-w64）建立的匯入函式庫連結，並在 UCRT 之上使用 Nabi 自己的啟動程式碼；不需要 MSVC 工具或 Windows SDK。Nabi 的 Windows 啟動程式碼在 `main` 之前執行 `.CRT$XCA`..`.CRT$XCZ` 中的 C 初始化器。對 x86 Windows，匯入名稱來自 mingw-w64 的 `lib32` 定義，帶有 stdcall 的 `@N` 修飾，映像以 `/SAFESEH:NO` 連結。Linux 目標，無論原生或交叉，一律連結 `<sysroot>/lib/nabi/targets/<triple>/` 中隨附的 C 函式庫：musl 以靜態方式，glibc 經由連結 stub（執行檔需要 glibc 2.34 以上）。其他函式庫經由 `-L` 與 `LIBRARY_PATH` 尋找。32 位元 glibc 目標所指定的動態載入器是 `/lib/ld-linux.so.2`（i686）與 `/lib/ld-linux-armhf.so.3`（armv7hf）。riscv32 映像（沒有作業系統，進行中）是靜態的，不連結任何 C 函式庫：執行環境自帶其進入點與堆積（[0.6.2](../000-meta/006-platform-targets.md#riscv32imac-unknown-none-elf)）。若設定了 `NABI_CC`，則改由該 C 編譯器驅動程式進行最終連結（`$NABI_CC <objects> <archives> -o <out>`）；這是選用的覆寫。以 `--native-runtime` 給出的共用函式庫會複製到執行檔旁邊，執行檔取得 rpath `@loader_path`（Mach-O）/ `$ORIGIN`（ELF）；Windows 從執行檔的目錄載入 DLL。執行檔會經過 dead-strip：移除未被參照的函式與資料；`release` 建置還會合併相同的函式並合併相同的字串字面值（[9.2.15](#9215-output)）。Mach-O 執行檔帶有 ad-hoc 程式碼簽章，這是 arm64 macOS 執行它所必需的。當 Apple 目標上的執行檔帶有偵錯資訊時，`nabic` 把目的檔的 DWARF 彙整成執行檔旁邊的 `.dSYM` 套組；在 Windows 目標上，連結器寫出一個 PDB（MSF 7.0）：合併後的目的檔 CodeView 型別、每個目的檔一個帶有其符號與行號表的模組、全域與公開符號，以及區段貢獻；執行檔的偵錯目錄會指名它（一筆帶有 PDB 之 GUID 與 age 的 RSDS 紀錄）。`wasm32-wasip2` 執行檔先連結為核心 wasm 模組，再以工具鏈為該目標隨附的 preview 1 轉接器做成 WASI preview 2 元件；`ffi::wit` world（[std 1.8](../../refs-standard-libraries/100-bare/108-wit.md)）決定其匯入與匯出的形狀。

## 9.2.15. 輸出

輸出與 cargo 一樣放在**依設定檔區分**的目錄中。設定檔有 `debug`（預設）與 `release`；最佳化等級為 `-O2` / `-O3` / `-Os`（包括 `--release`）時，設定檔是 `release`。

設定檔的差異與 cargo 相同：

| 設定檔 | 最佳化 | 偵錯資訊 | 連結 |
|---|---|---|---|
| `debug` | `-O0`、`-O1` | 有 | 移除未被參照的程式碼與資料 |
| `release` | `-O2`、`-O3`、`-Os` | 只在有 `-g` 時 | 另外合併相同的函式並合併相同的字串字面值；沒有 `-g` 時，除了載入器所需之外沒有任何符號 |

相同程式碼合併以函式的機器碼與重定位比較函式：當兩個函式的位元組相同，而且每個參照都指向相同的位置，或指向本身會合併在一起的函式時，兩者合併為一。位址被取得（被儲存、比較、匯出、放進表中）的函式絕不會被合併，因此不同的函式保持不同的位址；只有被呼叫與被跳躍到的函式才會被合併，函式值所呼叫的 thunk（`<symbol>.v`，[8.11](../008-abi/811-backend-contract.md)）也是：函式值無法比較，因此兩個函式值共用其程式碼不會改變程式能看到的任何東西。被合併之複本的展開項目與偵錯資訊會與被移除的程式碼一樣被捨棄：偵錯工具、效能分析工具或回溯會指出留下來的那份複本。位元組相同的字串字面值共用一份複本（它們的位址沒有意義）。`-C icf=no` 在 `release` 建置中關閉這項功能，`-C icf` 在 `debug` 建置中開啟它（[9.1](901-nabic-cli.md)）。

偵錯資訊屬於建置所產生的執行檔或封存檔（`build`、`run`、`test`，以及明確 `-O0` / `-O1` 的 `bench`），不屬於 `--emit` 輸出或 REPL 與 Jupyter 儲存格。panic 在兩種設定檔中都會指出其原始碼位置。建置的目的檔從輸出旁邊的同一個位置連結，因此相同的輸入產生相同的執行檔。溢位檢查在兩種設定檔中都是開啟的；`nabi.nabion` 中的 `profile` 可以為其中一個關閉它們（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。設定檔沒有其他設定。

```
target/
├── debug/                      # profile (default)
│   ├── foo                     # executable (src/main.nabi)
│   ├── libfoo.a                # library archive (src/lib.nabi)
│   ├── foo.nabihir              # library interface (9.4)
│   ├── build/                  # intermediates and --emit output (foo.mir, ...)
│   └── deps/                   # prebuilt dependencies (lib<dep>.a, <dep>.nabihir, <dep>.fp)
├── release/                    # --release
│   └── ...
├── aarch64-unknown-linux-gnu/  # cross build (--target=<triple>)
│   └── debug/ ...
└── doc/                        # nabi doc
```

| 輸出            | 路徑                                                              |
|-----------------|-------------------------------------------------------------------|
| 執行檔          | `target/<profile>/<name>`                                         |
| 函式庫          | `target/<profile>/lib<name>.a`                                    |
| `--emit=<kind>` | `target/<profile>/build/<name>.<ext>`（`ast.nabi` `hir` `mir` `o` `s` `lst` `expanded.nabi`） |
| 中間檔案        | `target/<profile>/build/`                                         |
| 相依套件        | `target/<profile>/deps/`（`lib<dep>.a`、`<dep>.nabihir`、`<dep>.fp`，9.2.3） |
| 交叉建置        | `target/<triple>/<profile>/...`（結構與上述相同）                 |
| 文件            | `target/doc/`                                                     |
| 增量            | `target/<profile>/incremental/<name>/`（執行檔的 codegen 單元）、`.../lib<name>/`（函式庫的），以及每個目標根各一個的 `.../<name>-<hash>/`（前端），9.2.16 |

兩個根檔案都有的套件兩種輸出都有，即 `target/<profile>/<name>` 與 `target/<profile>/lib<name>.a`（+ `<name>.nabihir`）；`-o` 指定執行檔。

名稱 `foo` 是套件名稱（9.2.2）。`-o` 改變單一輸出的路徑；`--target-dir=<dir>` 與環境變數 `NABI_TARGET_DIR` 改變 `target/` 本身的位置。`nabi clean` 刪除整個 `target/`。函式庫的 NABIHIR 是封存檔旁邊的 `foo.nabihir`（[9.4](904-nabihir-bytecode.md)）。

## 9.2.16. 增量建置

增量建置在三個層級上運作：輸入沒有改變的套件完全不建置；需要建置的套件從上次建置取得其未改變之檔案與函式的剖析結果與降階後主體（前端，見下文）；而其程式碼以 codegen 單元編譯，其中只有改變的單元會重新編譯。若上次建置的指紋等於目前的指紋，`nabic build` / `nabi run` 會印出 `Fresh` 並略過建置。指紋包括：

- 編譯器的身分（與 NABIHIR 產生者相同的值）、建置選項、診斷格式（`--message-format`、色彩）、輸入 / 輸出路徑、有設定時的 `NABI_CC`
- 套件及其相依套件 `src/` 下的檔案、它們的 `nabi.nabion`，以及建置所讀取的其他每個檔案（`src/` 之外的 `#[path]` / `include_str!`）
- 由 `include_str!` / `include_bytes!` 讀取的檔案（[6.11](../006-meta-system/611-include.md)），連同其內容
- lint 等級（`-A` / `-W` / `-D`，以及經由它們的資訊清單 `lints`）
- 執行環境函式庫、工具鏈為目標隨附的 C 執行環境目的檔與 C 函式庫（[9.2.19](#9219-installation-layout)），以及預先建置相依套件的 `.a`，各自依其大小、修改時間與檔案身分（inode 與狀態變更時間；在 Windows 上是其建立時間），因此保留修改時間的複本也算是改變
- 輸出的大小、修改時間與檔案身分

建置所發出的警告，包括 lint 警告與後端的警告，都會被記錄，而 `Fresh` 建置會再次印出它們。

每次建置，無論原生或交叉，都會編譯程式所觸及的標準函式庫函式（[9.3.6](903-package-archive.md#936-standard-library)）；其最佳化後的程式碼及其單元的目的檔保存在工具鏈的函式快取 `lib/nabi-cache/fn/`（9.2.19）中，供該工具鏈的每次建置使用，單獨檔案的建置也是：

- 項目的鍵涵蓋其結果所依賴的東西：對最佳化後的程式碼而言，是該步驟之前的函式程式碼、等級、目標，以及該步驟從它所呼叫之函式讀取的內容，全都以名稱表示；對目的檔而言，是單元的鍵（見下文）；兩者都包含編譯器的身分。套件把標準函式庫的降階後主體與它自己的一起保存（前端，見下文）。建置取用鍵相符的項目，並寫出它所產生的項目；輸出與以空快取進行的建置所產生的相同。
- 項目是完整寫出後再重新命名就位的檔案，因此同時執行的建置能安全地共用快取。
- 建置每天最多一次移除 30 天內沒有任何建置讀取過的項目；`nabi cache prune` 會移除更多（9.10.6）。快取目錄為唯讀且非空的工具鏈（建置時已預熱的沙箱映像）會讀取其項目，但不寫入任何項目。
- 快取有上限 `NABI_CACHE_MAX_SIZE`（位元組數，或帶 `K`、`M` 或 `G` 的數字；預設 `5G`；`off`：沒有上限），涵蓋其函式快取與其函式庫階段（見下文）。寫出了項目的建置會把它們的大小加到快取目錄中的累計總量；當總量超過上限時，一次只有一個建置會量測快取，若超過上限，就移除最久以前使用的項目，直到剩下最多上限的 80%。項目的使用是其最後一次寫入或讀取，每小時最多記錄一次。最近一小時內使用過的項目，以及仍在寫入中的檔案，絕不會以這種方式移除，因此同時執行的建置會保有它們所使用的東西；當它們讓快取保持在上限的 80% 以上時，下一次量測會等到再寫入上限的五分之一之後。被移除的項目會由下一個需要它的建置重新計算。
- `NABI_FN_CACHE=0` 既不讀取也不寫入快取。

### 前端

套件的建置（`check`、`build`、`run`、`test`、`bench`、`doc`、`lint`）把其前端所計算的東西保存在 `target/<profile>/incremental/<name>-<hash>/` 下，其中 `<hash>` 指稱套件根（兩個根檔案都有之套件的每個目標各有自己的一個，`<hash>` 指稱其根），每種建置（目標、cfg、是否為測試、哪些函式庫是預先建置的）各有一個子目錄。下一次相同種類的建置從其中取用：

- 文字與剖析輸入（建置的擴充運算子、套件的關鍵字、meta 區塊與 `#[literal]` 模式，以及 cfg）都未改變之每個檔案的剖析結果。
- 每個一般函式主體的降階（名稱解析、型別檢查、泛型實例化）：非泛型、非 `const` 的函式，或非泛型 `impl` 的方法。當它自己的文字與位置未改變、程式的介面未改變、降階所讀取的每個簽章、實例與其他事實現在都相同，而且其結果在主體之外所指出之每個位置之前的文字都未改變時，它會被重複使用。
- 介面是除了寫出回傳型別（或 `()`）之一般主體，以及一般函式的參數與回傳型別以外的每個項目；回傳型別是推論出來的主體屬於介面（其呼叫者會讀取該型別）。編輯主體會重新降階該主體，而編輯回傳型別是推論出來的主體，還會重新降階讀取其型別的主體；編輯一般函式的簽章會重新降階它以及讀取其簽章的主體；對項目的任何其他改變（型別、trait、`impl` 標頭、泛型函式、`use`）都會重新降階每個主體。
- 重複使用的降階所產生的函式、型別與實例，其編號與乾淨建置中的相同；診斷、程式及其符號都與乾淨建置的相同。

簽章、`impl` 註冊、一致性以及全程式檢查在每次建置中都會執行。收集函式庫之項目（標準函式庫的與相依套件的）所產生的東西，以及函式庫檔案的預先處理剖析結果，保存在工具鏈的快取中（`lib/nabi-cache/library/`，工具鏈、建置旗標與函式庫原始碼的每個雜湊一個檔案），供每次建置使用，單獨檔案的建置也是；`NABI_INCREMENTAL` 既不會移動也不會停用它們。保存的東西都不是必要的：遺失、損壞或過時的項目會重新計算。位於任何套件之外的單獨檔案不保存它自己的剖析結果或主體，除非 `NABI_INCREMENTAL` 指定了一個目錄。`NABI_INCREMENTAL=<dir>` 把結果保存在 `<dir>/<name>-<hash>/` 下；`NABI_INCREMENTAL=0` 關閉它們。`NABI_INCREMENTAL_VERIFY=1` 還會重新降階每個主體，若重複使用的降階有所不同，則讓建置失敗。`-v` 回報有多少檔案的剖析結果與多少主體被重複使用。

### Codegen 單元

不是 `Fresh` 的建置以 codegen 單元重複使用已編譯的程式碼。nabic 把套件的程式碼（執行檔的或函式庫的）分割成單元：套件的每個模組是一個單元，它所編譯之函式或實例所屬的標準函式庫模組與相依套件模組也各是一個（實例可能依其型別引數分散到多個單元）；另有一個單元容納程式只有一份的東西（static、C 的 `main`、GC 型別註冊）。

- 只有在單元的鍵自上次建置到同一個目標目錄以來改變時，才會編譯該單元。否則重複使用保存在 `target/<profile>/incremental/<name>/`（函式庫的：`lib<name>/`）下的結果；然後連結一切。標準函式庫的單元則改為保存在函式快取中（見上文），供每個套件的建置使用；單獨的檔案也會使用它，雖然它不保存自己的單元。預先建置的相依套件（9.2.3）編譯為一個目的檔，不分單元。
- 匿名檔案的單元保存在沒有套件時使用的目標目錄中（`--target-dir`、`NABI_TARGET_DIR`，否則為單獨檔案快取），位於 `[<triple>/]<profile>/incremental/<stem>-<hash of its path>/` 下。單獨檔案快取在 Unix 上是 `<system temporary directory>/nabic-target-<uid>/`，`<uid>` 是有效使用者 ID，在 Windows 上是 `%TEMP%\nabic-target\`，其暫存目錄是使用者自己的。它是一個快取：nabic 每天最多一次移除其中一週內沒有任何建置寫入過的檔案。
- 單獨檔案的執行檔保存在同一個目標目錄中，位於 `[<triple>/]<profile>/scripts/<key>/` 下，在其鍵相符期間會被重複使用；一天未使用的項目會在建置另一個項目時被移除（[9.10.7](910-nabi-frontend.md#9107-inline-manifest-for-single-file-scripts)）。`nabi run` 執行它，`nabic build` 把它複製到檔案旁邊。對腳本而言，`nabi` 以 `$NABI_HOME/cache/scripts/<hash>/` 下該腳本自己的 `target/` 傳入 `--target-dir`；單獨檔案快取容納其餘的部分：由 `nabic` 本身建置的單獨檔案，以及位於其 `nabi.nabion` 使它成為單一檔案套件之目錄中的檔案（9.10.1）。
- 單獨檔案快取容納會被執行的執行檔，因此在 Unix 上它是私有的：編譯器以模式 `0700` 建立它，並在使用前檢查它是一個目錄（不是符號連結）、由有效使用者擁有，而且沒有群組或其他人的權限位元。否則指令以結束碼 `1` 失敗，並印出 ``nabic: error: <dir> is not private to you (owner uid N, mode 0755); remove it or set NABI_TARGET_DIR``。`--target-dir` 與 `NABI_TARGET_DIR` 原樣使用。
- 單元的鍵涵蓋其函式的程式碼、它從其他單元所使用之東西的簽章與程式碼產生資訊（函式的簽章改變會重新編譯其呼叫者的單元）、它所使用的 static 與 GC 型別、目標、最佳化等級、程式碼產生選項、編譯器的身分，以及在 `-g` 下其偵錯資訊所指名之原始碼檔案的路徑（新增檔案只會重新編譯參照它的單元）。
- 符號與 GC 型別 id 不取決於其他單元包含什麼（單元從套件之共用單元所定義的符號讀取 GC 型別 id；[9.5.7](905-symbol-mangling.md#957-collision-avoidance-typehash-stability)），因此輸出與乾淨建置所產生的相同。
- 最佳化建置仍會跨單元內嵌：套件在被分割之前整體最佳化（9.2.10）。
- `-v` 回報有多少單元被重複使用與編譯；`-vv` 指出被編譯的單元。

## 9.2.17. 編譯器與執行環境的分離

`nabic` 不使用任何第三方 crate，也不連結任何外部程式碼產生器。

管線讓編譯器與執行環境保持分離：`nabic` 在建置時不參照執行環境（`libnabir.a`）。

它只發出對 `extern "C"` 符號的呼叫（[Extern "C" 符號](../008-abi/805-extern-symbols.md)），而在連結時，使用者的建置會一併連結 `libnabir.a`。

這種分離讓 `nabic` 與 `libnabir.a` 能在各自獨立的 SemVer 下演進。

## 9.2.18. 自給自足的工具鏈

`nabic` 是自給自足的：其程式碼產生器與連結器都是它自己的（[0.2.4](../000-meta/002-pipeline.md#024-code-generation)），它不執行任何外部編譯器、組譯器或連結器。交叉編譯是內建的：工具鏈為其支援的 Linux 目標隨附 C 函式庫（musl、glibc 連結 stub）。

| 需要系統提供的東西     | 目標                                     |
|------------------------|------------------------------------------|
| macOS SDK（Xcode Command Line Tools；不可重新散布） | Apple 目標 |
| 無（隨附的 C 函式庫）   | Linux 目標                               |
| 無（從隨附之模組定義建立的匯入函式庫；UCRT） | Windows 目標 |
| `NABI_CC`（選用）       | 覆寫最終連結（9.2.14）                   |

C 編譯器 `nabicc` 還需要目標的 C 標頭檔：工具鏈為其 Linux 與 wasm32 目標隨附它們，Apple 目標從 macOS SDK 取得，Windows 目標則在 Windows 主機上從 Windows SDK 與 Visual Studio 取得（[9.13.7](913-c-compiler.md#9137-headers-and-c-library)）。

## 9.2.19. 安裝配置

安裝配置遵循 rustup / cargo。`NABI_HOME` 的預設值是 `~/.nabi`（在 Windows 上是 `%USERPROFILE%\.nabi`）。

```
$NABI_HOME/
├── bin/                         # proxies (the newest toolchain's nabi): run the active toolchain's same-named executable
│   ├── nabi
│   ├── nabic
│   ├── nabicc
│   ├── nabils
│   └── <name>                   # an executable installed by nabi install (9.10.5a)
├── toolchains/
│   └── <version>-<triple>/      # self-contained toolchain
│       ├── bin/{nabi, nabic, nabicc, nabils}
│       └── lib/
│           ├── libnabir.a        # runtime
│           ├── nabi/lldb/nabi.py           # lldb formatters for -g programs (9.9.6)
│           ├── nabi/targets/<triple>/   # what --target links: the runtime (libnabir.a / nabir.lib), C library,
│           │                           #   its headers (sysroot/, for nabicc),
│           │                           #   and C runtime objects, wasm32-wasip2's preview 1 adapter; the host's included
│           ├── nabi/packages/          # the official packages it carries, read only: index, src/<sha256>/ (9.10.4a)
│           └── nabi/{core, std, bare}/src/   # standard library sources: each tier is laid out as a package, root src/lib.nabi (5.4)
├── cache/                       # git, registry, native, scripts (9.10.4)
├── envs/<name>/, envs/.current  # named environments: nabi.nabion, nabi.lock, target/ (9.10.5)
├── installs.nabion               # what nabi install put in bin/, and where from
├── tokens.nabion                 # registry API tokens, written by nabi login (9.10.4a)
├── sources.nabion                # index mirrors, only read (9.10.4a)
└── config.nabion                 # Config { toolchain }
```

`nabic` 相對於它自己的執行檔尋找標準函式庫與執行環境（sysroot = 執行檔的 `..`），不查看 `NABI_HOME`。標準函式庫的每個層級都像函式庫套件一樣配置：其根模組是 `lib/nabi/<tier>/src/lib.nabi`，其模組位於 `lib/nabi/<tier>/src/` 下。在實作儲存庫中建置的 `nabic` 從儲存庫的 `library/<tier>/src/` 讀取相同的配置。它自己的快取是工具鏈的 `lib/nabi-cache/`（儲存庫建置使用 `target/<profile>/nabi-cache/`）；以唯讀方式安裝的工具鏈（系統套件）則改用使用者快取目錄中的 `nabi/nabic-<hash of the sysroot>/`。它在 `fn/` 中容納函式快取，即各建置所編譯的標準函式庫函式（9.2.16），並在 `library/` 中容納函式庫所收集的項目與預先處理剖析結果（9.2.16）。`nabi cache` 顯示、修剪並清空它（9.10.6）。作用中工具鏈的選擇定義於 [9.10.6](910-nabi-frontend.md#9106-toolchain)。

Linux（glibc）工具鏈需要 glibc 2.34 以上，與它所建置的執行檔相同（9.2.14）。
