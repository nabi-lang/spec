# 9.1. `nabic` CLI 介面

本節定義編譯器 `nabic` 的命令列介面。工具鏈由四個二進位檔組成。

| 二進位檔 | 角色                                                                                  |
|----------|---------------------------------------------------------------------------------------|
| `nabic`   | 編譯器（相當於 rustc）。建置、檢查與輸出套件，並建置測試與效能測試的 harness（本節） |
| `nabi`    | Cargo 風格的前端。負責資訊清單、相依套件、環境與工具鏈釘選，呼叫 `nabic`，並在連結了編譯器的情況下執行各工具（run、lint、REPL、Jupyter、LSP、格式化工具、文件……）（[9.10](910-nabi-frontend.md)） |
| `nabicc`  | C 編譯器。與 `nabi cc` 是同一個編譯器（[9.13](913-c-compiler.md)）                 |
| `nabils`  | 供編輯器使用的 LSP 伺服器執行檔。與 `nabi lsp` 是同一個伺服器                       |

執行環境是 `libnabir.a`（`nabir`）。沒有另外的工具（`nabi-fmt` 等）。`nabic` 不解析相依套件；相依套件由 `nabi` 經由 `--extern` 傳入（[9.2.3](902-aot-pipeline.md#923-相依套件解析)）。

## 9.1.1. 一般形式

`nabic` 以下列一般形式呼叫。

```
nabic <subcommand> [options] [args]
```

以文法規則表示：

```
cli_invocation  ::= "nabic" subcommand option* arg*
subcommand      ::= "build" | "emit" | "test" | "bench" | "check"
                  | "version" | "help"
option          ::= "--" identifier ("=" value)?
                  | "-" letter
```

`--name value` 等同於 `--name=value`。全域選項（9.1.5）可以出現在子指令之前或之後。

子指令、選項、其值與路徑都必須是有效的 UTF-8。不是的引數屬於用法錯誤（9.1.6），錯誤訊息會指出其位置：``argument 3 (`b�c`) is not valid UTF-8; only a program's arguments, after `--`, may be other bytes``。`--` 之後的引數則不必是：`nabi run` 把它們依系統給出的樣子傳給程式（在 Windows 上也包括含有未配對代理字元的 UTF-16；[9.10.8](910-nabi-frontend.md#nabi-run建置並執行)）。

| 領域                | 子指令                                                |
|---------------------|-------------------------------------------------------|
| 建置                | `build`、`emit`、`test`、`bench`、`check`             |
| 其他                | `version`、`help`                                     |

各工具（`run`、`lint`、`repl`、`jupyter`、`lsp`、`fmt`、`doc`、`clean`、`bindgen`、`inspect`、`demangle`、`completions`）是 `nabi` 的子指令（[9.10.8](910-nabi-frontend.md#9108-工具)）。

### 套件選擇（`path`）

`build` / `emit` / `test` / `bench` / `check`（以及 `nabi` 的各工具）的 `[path]` 是下列其中之一。省略時為目前目錄。

| `path`                       | 套件                                                                                                    |
|------------------------------|---------------------------------------------------------------------------------------------------------|
| 套件目錄                     | 該目錄。若它沒有 `src/main.nabi` 或 `src/lib.nabi`，搜尋會往上移動（9.2.2）                             |
| 套件之 `src/` 中的檔案       | 該套件：從該檔案往上走，第一個含有 `src/main.nabi` / `src/lib.nabi` 的目錄                               |
| 其他任何檔案                 | 由該單一檔案組成的匿名可執行套件（[4.16.8](../004-functions/416-entry-point.md#4168-單一檔案執行)），包括套件目錄中位於其 `src/` 之外的 `.nabi` 檔案 |

若給出目錄而往上走也找不到套件，則是錯誤。匿名套件的檔案子模組（`mod foo;`）位於其檔案旁邊（[4.16.8](../004-functions/416-entry-point.md#4168-單一檔案執行)）。

套件名稱（輸出名稱）是套件目錄的名稱；對匿名套件而言，是去掉 `.nabi` 的檔案名稱。

## 9.1.2. 核心子指令

### `nabic build`：[AOT](../000-meta/007-glossary.md#aot預先編譯) 建置

`nabic build` 以 AOT 方式建置套件。

```
nabic build [path]
```

`[path]` 遵循上述的套件選擇規則。輸出放在 [9.2.15](902-aot-pipeline.md#9215-輸出) 中依設定檔區分的 `target/` 內。`src/main.nabi` 產生 `target/<profile>/<name>`；`src/lib.nabi` 產生 `target/<profile>/lib<name>.a` 以及介面 `target/<profile>/<name>.nabihir`（[9.4](904-nabihir-bytecode.md)）；兩者都有的套件兩者都產生。`-O2` / `-O3` / `-Os`（包括 `--release`）的設定檔是 `release`，其他是 `debug`；有 `--target=<target>` 時目錄是 `target/<target>/<profile>/`。若設定了 `--target-dir=<dir>` 或 `NABI_TARGET_DIR`，則改用該目錄而非 `target/`。

單獨的檔案（[4.16.8](../004-functions/416-entry-point.md#4168-單一檔案執行)）會在暫存目錄中以匿名套件建置，因此不會在檔案旁邊建立 `target/`，而其執行檔會以其主檔名加上目標的執行檔後綴複製到檔案旁邊（`hello.nabi` 產生 `hello`，在 Windows 上是 `hello.exe`），除非 `-o` 指定了另一個檔案。若該路徑上已有一個不是該腳本先前之建置所寫出的檔案，就不會動它：建置失敗，並提示 `-o <file>`。

選項：

- `--release` 在 `release` 設定檔中進行最佳化建置（等同於 `-O2`）；預設是在 `debug` 設定檔中進行偵錯建置（`-O0`）。各設定檔的差異如 [9.2.15](902-aot-pipeline.md#9215-輸出) 所述。
- `--target-dir=<dir>` 改變 `target/` 的位置（等同於 `NABI_TARGET_DIR`）。
- `--crate-type=lib|cdylib|staticlib|plugin` 選擇函式庫套件的輸出。`lib`（預設）是靜態封存檔及其 NABIHIR；`cdylib` 還會在它們旁邊連結一個供 C 程式碼使用的共用函式庫，即 `lib<name>.so`、`lib<name>.dylib`，或 `<name>.dll` 加上 `<name>.lib`（[9.3.3](903-package-archive.md#933-靜態與共用函式庫)）；`-o` 指定該檔案。它也會寫出函式庫的 C 標頭檔 `target/<profile>/include/<name>.h`（[9.3.3](903-package-archive.md#933-靜態與共用函式庫)）。`staticlib` 產生一個 C、Swift 與 Xcode 可以單獨連結的靜態函式庫 `static/lib<name>.a`（在 Windows 上是 `static/<name>.lib`；由 `-o` 指定），並附上標頭檔與 `include/module.modulemap`（[9.3.3](903-package-archive.md#933-靜態與共用函式庫)）；對 wasm 目標是錯誤。搭配它時，`--xcframework` 還會把該目標的切片加入 `<profile>/<name>.xcframework`（Apple 目標）。`plugin` 則連結一個由 Nabi 程式載入的外掛，檔案名稱相同，但沒有 `.lib`，也沒有標頭檔（[9.12](912-plugins.md)）；對 `aarch64-apple-ios`（進行中，[0.6.2](../000-meta/006-platform-targets.md#062-進行中)），由於其裝置只載入簽署進 app 的程式碼，這是錯誤，錯誤訊息會提示 `aarch64-apple-ios-sim`。對沒有 `src/lib.nabi` 的套件使用它是錯誤，對同時有 `src/main.nabi` 的套件，或對 wasm 或 musl 目標使用 `plugin` 也是錯誤。
- `--lib-version <x.y.z>` 給予 `cdylib` 其版本：在 Linux 上其 soname 是 `lib<name>.so.<x>`，低於 1.0 時是 `lib<name>.so.0.<y>`，並在函式庫旁邊建立該名稱的連結；在 macOS 上它設定其 install name 的目前版本（`x.y.z`）與相容版本（`x.0.0`，低於 1.0 時是 `0.y.0`）。沒有它時，soname 是檔案名稱，兩個版本都是 1.0.0。`nabi build` 會傳入套件的 `version`（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。不是三個數字的值是無效的選項。
- `--bundle=<dep>`（搭配 `--crate-type=plugin`）把相依套件 `<dep>` 連結進外掛，而不是從載入它的程式取得。
- `--hot` 建置一個熱重載外掛，即 `nabi dev` 載入執行中程式的東西（隱含 `--crate-type=plugin`）。只有 `src/main.nabi` 的套件建置為根外掛，其 `main` 以 `<name>::main` 匯出；兩個根檔案都有的套件是錯誤。對熱套件之函式的每次呼叫都經過熱執行環境會重新繫結的槽位，其 static 繫結到第一個版本的儲存空間，其型別被完整描述，而其套件雜湊對該套件的每次建置都相同（取其名稱的雜湊），因此新版本保留舊版本的名稱。編譯器還會在外掛旁邊寫出 `<output stem>.layout`：版本編號以及套件每個型別的配置。`--hot` 只以 `-O0` 建置：`--release`、`-O1` .. `-O3` 與 `-Os` 都是錯誤（[8.4.13](../008-abi/804-safepoint.md#8413-熱重載程式碼)）。
- `--hot-base=<file>`（搭配 `--hot`）指定程式正在執行之版本的 `.layout`。新版本的編號接在它之後，並與它對照檢查：新增了沒有預設值的欄位、欄位的型別改變、`#[renamed_from]` 沒有指名任何舊欄位，或型別參數或種類改變，都是錯誤（E0149），錯誤訊息會說明應改寫成什麼。
- `--hot-dep=<dep>`（搭配 `--hot`）表示相依套件 `<dep>` 也是熱套件：其函式經由它自己之外掛的槽位呼叫，而不是被編譯進來，其套件雜湊也以相同方式固定。
- `--extern <name>=<dir | src/lib.nabi | <name>.nabihir | lib<name>.a>` 把一個相依套件掛載在頂層路徑 `name`（[9.2.3](902-aot-pipeline.md#923-相依套件解析)）。原始碼（`dir` / `src/lib.nabi`）在同一個建置中一起編譯；預先建置的套件（`.nabihir` / `.a`；給出其中一個時，會找到它旁邊的另一個）會讀取其 NABIHIR 並連結其封存檔。可以給出多次，通常由 `nabi` 填入。`name` 是識別字：把套件名稱中的 `-` 換成 `_`（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。`std`、`core` 與 `bare` 屬於標準函式庫：以這些名稱命名的函式庫套件會被拒絕，而 `--extern std=<path>` 是用法錯誤，除非之前的某個 `--extern` 指名了同一個路徑，使它成為該相依套件的另一個名稱（`--extern helper=p --extern std=p`，程式碼寫成 `std::..`；`::std::..` 仍是標準函式庫）。關鍵字與路徑起點（`pkg`、`sup`）也是如此，無論作為 `name`，或作為函式庫或帶有函式庫之套件的名稱（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。
- `--extern-feature <name>=<f>` 以 `cfg(feature = "<f>")` 建置原始碼相依套件 `name`。可重複，每個 feature 一次。相依套件只看得到這些：`--cfg` 的 `feature` 鍵屬於正在編譯的套件。通常由 `nabi` 填入。
- `--extern-dep <package>=<name>`：套件 `<package>`（一個 `--extern` 名稱，或以 `pkg` 表示正在建置的套件）相依於 extern `<name>`。可重複。有任何一個時，每個套件只看得到它所相依的 extern（[9.2.3](902-aot-pipeline.md#923-相依套件解析)）；沒有時，每個套件都看得到每個 extern。通常由 `nabi` 填入。
- `--name <n>` 設定套件名稱（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock) 的命名規則）。輸出是 `lib<n>.a` / `<n>.nabihir`，掛載名稱是把 `-` 換成 `_` 的名稱。預設是目錄名稱。`--extern a-b=..` 同樣繫結為 `a_b`。
- `--bootstrap <file>` 給出要預先載入的標準函式庫根（可重複）；`--nabir <lib>` 給出要連結的執行環境。預設是 sysroot 的 `lib/nabi/{core,std,bare}/src/lib.nabi` 與 `lib/libnabir.a`（[9.2.19](902-aot-pipeline.md#9219-安裝配置)）。bootstrap 根的層級是其 `src/` 上一層目錄的名稱；給出 `core/src/lib.nabi` 而沒有 `bare` 根時，也會載入它旁邊的 `bare/src/lib.nabi`。找不到時是錯誤，錯誤訊息會列出搜尋過的路徑。
- `-O0` / `-O1` / `-O2` / `-O3` / `-Os` 明確選擇最佳化等級（[9.2.10](902-aot-pipeline.md#9210-最佳化等級)）：`-O0` 不最佳化；`-O1` 迴圈與 `-O2` 一樣最佳化，其他地方幾乎不增加程式碼，帶有偵錯資訊；`-O2` 追求速度；`-O3` 以更多程式碼換取更多速度（迴圈中更大的內嵌、展開純量迴圈）；`-Os` 追求大小，迴圈仍會計數，短的迴圈仍會向量化。
- `--target=<target>` 選擇目標平台：內建名稱，即[平台目標](../000-meta/006-platform-targets.md#061-已支援)中的已支援目標之一（進行中的目標只能搭配維護者的測試開關，[0.6.2](../000-meta/006-platform-targets.md#062-進行中)），或一個 `<name>.nabion` 目標規格檔（[0.6.5](../000-meta/006-platform-targets.md#065-目標規格)）。任何目標都可以從任何主機建置（[0.2.4](../000-meta/002-pipeline.md#024-程式碼產生)）。其他任何名稱都是錯誤。對進行中或不支援的目標（[0.6.2](../000-meta/006-platform-targets.md#062-進行中)、[0.6.3](../000-meta/006-platform-targets.md#063-不支援)），錯誤訊息會說明這一點，並在有替代方案時說明改用什麼（``target `x86_64-apple-darwin` is not supported (Intel Macs); use `aarch64-apple-darwin` ``、``target `aarch64-apple-ios` (iOS devices) is in progress: it is not tested on real hardware yet and comes in a later release; for now build for the simulator with `--target aarch64-apple-ios-sim` ``）；對其他名稱，在有最接近的內建目標時會指出它（``unknown target `wasm32-wasi`; did you mean `wasm32-wasip2`?``）。
- `-L <dir>` 為 `#[link]`（[5.9.9](../005-modules/509-extern-block.md#599-linklibname指名函式庫)）新增一個函式庫搜尋目錄，在 `LIBRARY_PATH` 之前搜尋。可重複。
- `-l [static=|dylib=|framework=]<name>` 連結函式庫 `<name>`（與 `#[link]` 一樣搜尋），或者搭配 `framework=` 時，如 `#[link(name = "<name>", kind = "framework")]` 一樣連結 Apple framework `<name>`；`-F <dir>` 新增一個 framework 搜尋目錄，在 SDK 的目錄之前搜尋。`--native-runtime <file>` 指定一個要與執行檔一起發行的共用函式庫（[9.2.14](902-aot-pipeline.md#9214-連結執行檔)）。以上全都可以重複；`nabi` 為原生套件與系統函式庫傳入它們（[9.10.4b](910-nabi-frontend.md#9104b-原生套件)）。在 Apple 以外的目標上使用 framework 是連結錯誤，並提示把 `extern` 區塊放在 `#[cfg(target_os = "macos")]` 之下。
- `--native-object <file>` 把一個目的檔或封存檔原樣連結進執行檔或共用函式庫（可重複）；`nabi` 傳入套件之 C 原始碼的目的檔（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）。
- `-o <path>` / `--output=<path>` 選擇輸出位置。
- `--keep` 保留建置的暫存目錄（目的檔與其他中間檔案）而不移除，並在標準錯誤上指出它（`nabic: kept temporaries in <dir>`）。
- `--cfg <flag>` / `--cfg <key>=<value>` 為[條件編譯](../006-meta-system/603-cfg.md)設定一個 cfg 旗標。可重複。
- `--emit=<kind>[,<kind>..]` 把中間輸出（`source`、`hir`、`mir`、`obj`、`asm`、`listing`、`expansions`、`c-header`）寫到 `target/<profile>/build/<name>.<ext>`（[9.9.1](909-debug-tools.md#991-中間輸出--emit)）。可以用逗號分隔一次給出多種；此時不能使用 `-o`。有 `--emit` 時不會產生執行檔或封存檔。
- `-g` 為 `release` 設定檔的建置加上偵錯資訊：DWARF，或對 Windows 目標是 CodeView 與 PDB（[9.9.6](909-debug-tools.md#996-偵錯工具整合gdb--lldb)）。它不改變產生的程式碼：帶 `-g` 的最佳化建置與不帶時有相同的機器碼，其偵錯資訊描述最佳化後的程式碼把每個變數放在哪裡（[9.2.10](902-aot-pipeline.md#9210-最佳化等級)）。`debug` 設定檔的建置一律帶有它。
- `--remap-path-prefix <from>=<to>` 在建置所產生的東西中，把目錄 `<from>` 下的原始碼檔案路徑寫成位於 `<to>` 下：panic 位置、偵錯資訊與 `.nabihir` 檔案。診斷保留真實路徑。相對路徑會以從工作目錄起算的絕對路徑比對，前綴以完整的路徑組成部分比對，其餘部分以 `/` 接到 `<to>` 之後（空的 `<to>` 會讓它保持相對），而當多條規則相符時，最後給出的勝出。可重複。標準函式庫的原始碼一律寫在 `/nabi/library` 下（`/nabi/library/core/src/...`），因此程式在每台機器與每次安裝上都指名相同的路徑。套件的 `release` 設定檔建置也會把套件自己的檔案寫成相對於其目錄（`src/main.nabi`），把原始碼相依套件的檔案寫成位於其名稱之下（`json/src/lib.nabi`），因此在兩個地方建置同一個套件會產生相同的位元組；`--remap-path-prefix` 規則在這些之後套用，並且勝出。
- `-C <option>` 也可以寫成 `--codegen <option>`。
- `-C overflow-checks[=yes|no]` 開啟或關閉固定寬度整數算術的溢位檢查（panic）。每個設定檔中預設都是開啟；`nabi.nabion` 的設定檔可以把它關閉（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)），而 `-C overflow-checks` 會覆寫它。省略值表示 `yes`。對不會溢位之常數運算元的運算會略過檢查。
- `-C icf[=yes|no]` 開啟或關閉連結時的相同程式碼合併（[9.2.15](902-aot-pipeline.md#9215-輸出)）。預設在 `release` 設定檔中開啟，在 `debug` 設定檔中關閉。省略值表示 `yes`。
- `-C target-cpu=<cpu>|native` 指定一個 CPU，取代三元組的基準（[0.6.4](../000-meta/006-platform-targets.md#064-simd)）；`-C target-feature=+f1,-f2,..` 開啟或關閉個別功能（`+avx2`……），每個寫成 `+feature` 或 `-feature`，連同它們所需要的功能（`+avx2` 會開啟 `avx`）或需要它們的功能（`-avx` 會關閉 `avx2`）。程式碼產生器在 `x86_64` 上使用 `avx2`、`popcnt`、`lzcnt` 與 `bmi`，在 `aarch64` 上使用 `lse`、`rcpc` 與 `dotprod`，在 `s390x` 上使用 `vector` 與 `vector-enhancements-1`，在 `riscv64` 上使用 `v`，在 `riscv64` 與 `riscv32` 上使用 `zbb`，並在 `armv7` 上對 `-neon`、在 `powerpc64le` 上對 `-vsx`、在 `wasm32` 上對 `-simd128` 關閉向量迴圈（[8.7.8](../008-abi/807-code-generation.md#878-迴圈向量化)）；以不改變上述任何一項之選項產生程式碼的建置或執行會發出警告說明這一點。未知的 CPU 名稱是錯誤，錯誤訊息會列出已知的名稱。`native` 只在目標是建置機器的平台時才被接受，並加入這台機器所具有的功能。
- `-C subsystem=console|windows` 選擇 Windows 子系統：`console`（預設，`mainCRTStartup`）或 `windows`（GUI，`WinMainCRTStartup`；[9.5.9](905-symbol-mangling.md#959-main-函式的符號)）。
- 其他任何 `-C` 選項都是無效的選項（E0132，見下文）：``unknown codegen option `<option>` ``，在有最接近的選項時會指出它，並附上列出各選項的說明。

`nabic build` 與 `nabic check` 一樣回報套件的 lint 診斷（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）：警告會被印出，建置繼續進行；`deny` 的 lint 會讓它失敗。若自上次建置以來沒有任何改變，`nabic build` 會印出 `Fresh`，然後再次印出那次建置的警告，並略過建置（[9.2.16](902-aot-pipeline.md#9216-增量建置)）。

### `nabic emit`：中間輸出

```
nabic emit [path] [--emit=<kind>[,<kind>..]]
```

等同於 `nabic build --emit=..`。省略 `--emit` 時為 `mir`。

### `nabic test`：測試

`nabic test` 收集測試函式並建置測試 harness；`nabi test`（[9.10](910-nabi-frontend.md)）執行它。

```
nabic test [path] [filter]
```

它收集標示了 `#[test]` 的函式，並建置一個測試執行檔（harness）。`#[test]` 隱含 `#[cfg(test)]`（[6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-測試函式標記)），因此測試函式不屬於一般的建置。測試同時從 `src/`（在 `#[cfg(test)]` 之下）與 `tests/` 目錄（[0.3.5](../000-meta/003-source-units.md#035-測試與基準測試目錄)）收集。測試建置使用一個選擇測試的進入點，取代套件的進入點。

`nabic` 不執行 harness。`--emit-harness <file>` 把建置出的執行檔以及其中的測試（`nabi test` 所執行的清單）寫到 `<file>`；其第一行是 `version 2`，即清單格式。執行器會拒絕較新的版本：``the test harness is from a newer nabic (format version N); run the nabi of the same toolchain, e.g. `nabi +<version> test` ``。沒有 `--emit-harness` 時，建置出的執行檔會在標準錯誤上列出。執行 harness（下文所述的選擇測試、行程隔離、平行化與回報）是 `nabi test` 執行器的工作（[9.10](910-nabi-frontend.md)）。

若只有一個引數且它不是既有的路徑，就把它當作 `filter`。`filter` 比對包括模組路徑在內的測試名稱（`math::adds`）。

- 它是一個使用 `^` `$` `.` `*` `+` `?` 與 `\` 跳脫的小型正規表示式。搜尋不錨定，因此沒有 `^` / `$` 時它可以比對名稱的任何部分。
- 沒有這些字元時，它等同於單純的子字串搜尋。

**每個測試都在自己的行程中執行。** 若某個測試 panic（結束碼 101）或以未捕捉的錯誤結束（結束碼 1），只有該測試失敗，其餘的繼續執行。被訊號終止也是失敗，執行超過時間限制（`--timeout`，預設 600 秒）的測試也是：它會被終止並回報為逾時。`#[should_panic]` 測試只在 panic 時通過；`#[ignore]` 測試回報為已忽略，不會執行（[6.2.5](../006-meta-system/602-built-in-attributes.md#625-test-測試函式標記)）。

#### 文件測試

對帶有函式庫的套件，`nabic test` 也會建置其文件測試：函式庫之 `///` 與 `//!` 文件註解中的圍欄程式碼區塊，以及套件根目錄 `README.md` 中的 ```` ```nabi ```` 區塊。在文件註解中，沒有語言的圍欄也是 Nabi（與 `nabi doc` 的顯示方式相同）；其他任何語言的區塊（```` ```text ````、```` ```nabion ````）都不是測試。位於建置之 `#[cfg]` 排除之模組中的區塊，或位於被排除之項目上的區塊（沒有該 feature 時的 `#[cfg(feature = "table")]`），也不是測試。

每個區塊都是使用該函式庫的程式碼，就像相依於它的程式一樣：函式庫以套件名稱作為它的相依套件（`stats::mean(x)`），而每個區塊開頭就如同有 `use stats::*;`，因此函式庫的公開項目不需要路徑（`mean(x)`）。區塊開頭的 `use` 項目留在模組層級；其餘部分是一個可以擲回錯誤之函式的主體（`try` 不需要 `catch` 也能運作），其中的項目是區域項目。宣告了 `fn main` 的區塊會呼叫它。圍欄中 `nabi` 之後、以 `,` 或空白分隔的字詞會改變通過的意義：

| 圍欄 | 區塊 |
|-------|-----------|
| ```` ```nabi ```` | 會被建置並執行；回傳時即通過 |
| ```` ```nabi,norun ```` | 會被建置，但絕不執行（讀取檔案、等待或需要網路的程式碼） |
| ```` ```nabi,should_panic ```` | 只在 panic 時通過 |
| ```` ```nabi,compile_fail ```` | 只在無法建置時通過；`compile_fail,E0040` 還要求該錯誤碼 |
| ```` ```nabi,ignore ```` | 既不建置也不執行，回報為已忽略 |

其他字詞會引發警告，並列出可用的字詞。以 `# `（或單獨的 `#`）開頭的行是測試的一部分，但 `nabi doc` 不會顯示它，用於讀者不需要看到的準備工作：

````nabi
/// ```nabi
/// # let data = [1.0, 2.0, 3.0];
/// assert(stats::mean(data) == 2.0);
/// ```
````

文件測試的名稱是 `doc <file>:<line>`，檔案相對於套件，行號是其開頭圍欄的行號，當區塊位於某個項目上時，後面接著被文件化的項目：`doc src/stats.nabi:12 (mean)`、`doc README.md:20`。篩選器、`--list` 與各紀錄像對待其他名稱一樣看待這些名稱。一個套件的所有區塊建置為一個執行檔，排在函式庫自己的測試之後；每個 `compile_fail` 區塊在建置 harness 時各自檢查。區塊的程式碼保留其行號，因此其中的錯誤會指出被文件化之檔案的行號。無法建置的區塊作為獨立的案例失敗，其錯誤與該失敗一起顯示，其他區塊則在沒有它的情況下建置並執行。

`nabic test --doc` 只建置文件測試，`--no-doc` 一個都不建置；`nabi test` 會把兩者都傳下去。

選項是 `nabic build` 的選項加上 `--emit-harness`。同時執行多少個測試、通過的測試是否顯示其輸出，以及輸出形式，是執行器 `nabi test` 的選項（[9.10.1](910-nabi-frontend.md#9101-子指令與全域選項)）。

執行器以下列形式印到 stdout。

```
running 3 tests
test math::adds ... ok
test math::subs ... FAILED
test util::trims ... ok

failures:

---- math::subs ----
<stdout / stderr of that test>
(panicked (exit 101))

failures:
    math::subs

test result: FAILED. 2 passed; 1 failed; 0 ignored; 0 filtered out; finished in 0.12s
```

被忽略的測試顯示為 `test <name> ... ignored`（有原因時加上 `, <reason>`）。若有任何測試失敗，`nabi test` 以 `1` 結束。

在兩個根檔案都有的套件中，先執行函式庫的測試，再執行執行檔的測試（對照不含其測試的函式庫建置），然後是文件測試；失敗的執行會結束該指令。`bench` 的運作方式相同。

### `nabic bench`：效能測試

```
nabic bench [path] [filter]
```

收集標示了 `#[bench]` 的函式，並以最佳化建置建置它們（除非給出 `-O`，否則為 `-O2`）；明確的 `-O0` 或 `-O1` 會與 `nabic test` 一樣，以帶有偵錯資訊的 `debug` 設定檔建置。`nabi bench`（[9.10](910-nabi-frontend.md)）執行 harness 並量測它們（[9.9.11](909-debug-tools.md#9911-效能測試)）。與 `#[test]` 一樣，`#[bench]` 隱含 `#[cfg(test)]`，效能測試同時從 `src/` 與 `benches/` 目錄（[0.3.5](../000-meta/003-source-units.md#035-測試與基準測試目錄)）收集。`path`、`filter` 與 `--emit-harness` 的規則與 `nabic test` 相同。

效能測試接受一個 `Bench` 控制代碼並對一個閉包計時，因此準備工作不會被算入量測：

```nabi
#[bench]
fn parse_1kb(b: bench::Bench) {
    let input = sample(1024);                   // setup: run once, not timed
    b.set_bytes(1024);                          // optional: report throughput
    b.iter(|| parse(hint::black_box(input)));   // the timed region
}
```

`b.iter(f)` 執行整個量測（校準、暖身、取樣）；其結果會被黑箱化。`hint::black_box`（[`core::hint`](../../refs-standard-libraries/200-core/README.md#benchmarks--optimizer-hints)）是最佳化器無法看穿的恆等函式。把它套用到計時區域的輸入與結果，好讓工作既不會被提升到外面，也不會被摺疊掉。`b.set_bytes(n)` / `b.set_items(n)` 會加上一個吞吐量欄位。`Bench`、`black_box` 與統計是函式庫 API；其方法定義於 [9.9.11](909-debug-tools.md#9911-效能測試)。

每個樣本的呼叫次數、樣本數量與輸出形式是執行器 `nabi bench` 的選項（[9.10.1](910-nabi-frontend.md#9101-子指令與全域選項)）。

效能測試一次執行一個。無論是否輸出到終端機，每個都以人類易讀的單位印出一行對齊的內容：

```
  json::parse_1kb     1.84 us +/- 0.03 us   543 MiB/s   [2 outliers, 0 gc]
```

`nabi bench --format=nabibench` 則改為印出供回歸工具使用的穩定機器形式，每個效能測試一行：`#nabibench name=<name> median_ns=<f> mad_ns=<f> samples=<n> outliers=<n> gc=<n> bytes=<n> items=<n>`。這一行原樣凍結：不會新增、移除或重新排列任何鍵，因此為它撰寫的讀取程式能持續運作。新的欄位只會放在 `--format=json` 的 JSON 紀錄中，它也會把效能測試自己的輸出與這些紀錄分開（使用 `nabibench` 時，它與這些行共用 stdout）。

從未呼叫 `b.iter` 的效能測試沒有量測到任何東西。這不是失敗：它的那一行是 `` <name>  warning: measured nothing: time the code with `b.iter(|| ..)` ``，摘要把它算作 `<n> measured nothing`，而不是已執行。

### `nabic check`：診斷

```
nabic check [path | -] [--check-library]
```

執行前端直到型別檢查與 MIR 建立，加上 lint，然後執行不含原生建置（沒有目的檔、不連結）的程式碼產生，並回報診斷。選項與 `nabic build` 相同。`--check-library` 不論是否可到達，都降階並檢查函式庫的每個主體，並把每個泛型主體對照其約束檢查；`nabic build --check-library` 還會以建置的等級最佳化並編譯每一個主體到執行檔中（否則建置只會編譯程式所觸及的函式庫程式碼，[9.3.6](903-package-archive.md#936-標準函式庫)）（[2.10.9](../002-types/210-generics.md#2109-主體檢查)）；此時其 lint 也涵蓋標準函式庫自己的程式碼，且 `missing_docs` 為 `deny`（[6.2.9](../006-meta-system/602-built-in-attributes.md)）。兩個根檔案都有的套件會先檢查其函式庫，然後若函式庫沒有錯誤，再檢查編譯進該函式庫的執行檔；每個警告只回報一次。

## 9.1.3. `--print` 選項

### `nabic --print cfg`：作用中的 cfg

```
nabic --print cfg [--target=<target>] [-O<n>] [--cfg ..]
```

依每行一項印出選項所描述之建置的 cfg（[6.3](../006-meta-system/603-cfg.md)）：旗標印成 `flag`，鍵印成 `key="value"`（與 rustc 相同）。鍵包括 `target_env`，而 `debug_assertions` 在 `-O0` 時設定。

```
debug_assertions
target_arch="aarch64"
target_endian="little"
target_family="unix"
target_os="macos"
target_pointer_width="64"
```

### `nabic --print target-list` / `target-spec`：目標

```
nabic --print target-list
nabic --print target-spec [--target=<target>]
```

`target-list` 依每行一個印出已支援的目標名稱；進行中的目標（[0.6.2](../000-meta/006-platform-targets.md#062-進行中)）不列入。`target-spec` 以 NABION 印出目標的規格（[0.6.5](../000-meta/006-platform-targets.md#065-目標規格)）；沒有 `--target` 時是主機的規格。

### `nabic --print lldbinit`：lldb 格式化器

印出一行 `command script import '<path>'`，用以載入這個工具鏈的 lldb 格式化器（[9.9.6](909-debug-tools.md#格式化器)）：`<sysroot>/lib/nabi/lldb/nabi.py`。載入它們也會把標準函式庫的 `/nabi/library` 路徑對應到工具鏈的函式庫原始碼（`target.source-map`）。

`--print gdbinit` 印出 `source <path>`，即載入 gdb pretty-printer `<sysroot>/lib/nabi/gdb/nabi.py` 的 gdb 指令（它們以 `set substitute-path` 對應函式庫路徑）。`--print natvis` 印出 natvis 視覺化工具的路徑 `<sysroot>/lib/nabi/natvis/nabi.natvis`。

## 9.1.4. 輔助子指令

### `nabic version`：版本

```
nabic version
nabic --version            # = -V
nabic -V -v                # adds host line
```

在 stdout 印出一行 `nabic <version>`，當 `nabic` 是從 git checkout 建置時，後面接著 ` (<commit> <date>)`。有 `-v` 時，會附加一行 `host: <triple>`。

### `nabic help`：說明

```
nabic help [subcommand]
nabic <subcommand> --help
```

沒有子指令時，印出完整的參考：每個子指令、選項與環境變數。給出子指令時，印出該子指令自己的說明：其用法形式、簡短描述、一個以上的範例，以及它所接受的選項。兩者都不會印出屬於其他子指令的文字。

## 9.1.5. 全域選項

全域選項如下所列。它們可以出現在子指令之前或之後。

| 選項                             | 意義                                                        |
|----------------------------------|-------------------------------------------------------------|
| `--verbose` / `-v`               | 顯示各階段的耗時（[9.9.5](909-debug-tools.md#995-記錄--進度顯示)） |
| `-vv`                            | 更詳細的記錄：`Reading <file>` 與 `Specialized <path>(sig)` 行（[9.9.5](909-debug-tools.md#995-記錄--進度顯示)） |
| `--quiet` / `-q`                 | 不印出進度行（仍會印出診斷）                                |
| `--color=auto\|always\|never`    | stdout 與 stderr 上的色彩（9.1.8）                          |
| `--no-color`                     | 等同於 `--color=never`                                      |
| `--cwd=<path>`                   | 執行之前變更工作目錄                                        |
| `--print cfg\|target-list\|target-spec\|lldbinit\|gdbinit\|natvis` | 印出作用中的 cfg、已支援的目標名稱、目標的規格，或載入 Nabi 偵錯工具格式化器的內容（lldb、gdb、natvis；9.1.3） |
| `--message-format=human\|json`   | 診斷格式。`json` 對每個診斷在 stdout 寫出一行（9.1.8）      |
| `--explain <code>`               | 印出錯誤碼（`E0040`）的說明頁：其意義以及應改寫成什麼，然後結束（[9.9.4](909-debug-tools.md#錯誤碼)） |
| `-A` / `-W` / `-D <lint \| group>` | lint 等級（`--allow` / `--warn` / `--deny`）。適用於每個會編譯套件的指令（`build`、`test`、`bench`、`emit`、`check`，以及 `nabi` 的 `run`、`lint`、`repl`、`jupyter`、`lsp`）。可重複；較後者勝出 |

`nabic help` 把 `--print` 的各種形式與建置選項列在一起，因為它們印出的內容取決於 `--target` 與 `--cfg`；它們不需要子指令。

lint 等級依下列順序決定：lint 的預設等級 < `-A` / `-W` / `-D` < 程式碼中的 `#[allow(..)]` / `#[warn(..)]` / `#[deny(..)]`（最內層者勝出）。`nabi` 把資訊清單的 `lints` 作為這些旗標加在前面（[9.10.1](910-nabi-frontend.md#9101-子指令與全域選項)）。

色彩對 stdout 與 stderr 分別依下列順序決定：`--color=never` 關閉它，`--color=always` 開啟它；否則（`--color=auto`，預設）非空的 `NO_COLOR` 關閉它，非空且不是 `0` 的 `CLICOLOR_FORCE` 開啟它，`TERM=dumb` 關閉它，其他情況下串流是終端機時就加上色彩。在 Windows 上，不接受跳脫序列的主控台會得到純文字。

範例：

```
nabic --verbose build
nabic test -q
nabic --cwd=examples/hello check
```

## 9.1.6. 結束碼

結束碼定義如下。

| 碼    | 意義                                                 |
|-------|------------------------------------------------------|
| `0`   | 成功                                                 |
| `1`   | 使用者程式碼錯誤（編譯錯誤 / 測試失敗等）            |
| `2`   | 用法錯誤（無效的選項、未知的子指令等）               |
| `70`  | nabic 內部錯誤（panic）                               |

用法錯誤會印出 `nabic: <message>`，然後是一行 ``run `nabic help <subcommand>` for usage``（沒有給出已知的子指令時是 ``run `nabic help` for usage``）。訊息會說明應改寫成什麼：未知的子指令或選項在有最接近者時會指出它（``unknown option `--relase`; did you mean `--release`?``），而看起來是檔案的未知子指令（含有 `/` 或以 `.nabi` 結尾）會建議 `nabic build <file>`。當 `__NABI_CLI` 是 `nabi` 時（9.1.7），兩行都會寫 `nabi` 而不是 `nabic`。

已知選項被給予它不接受的值時，會回報為診斷 `E0132`（[9.9.4](909-debug-tools.md#錯誤碼)），結束碼仍是 `2`：未知的 `-C` 選項或 `--emit` 種類、缺少值的選項、未知的 `-C target-cpu`、`--print`、`--color` 或 `--message-format` 不接受的值、格式錯誤的 `-l`、`--extern`、`--extern-feature` 或 `--extern-dep`、`--explain` 沒有的錯誤碼，或 `-o` 搭配多種 `--emit` 種類。其 `= help:` 行會說明該寫什麼（``error[E0132]: unknown emit kind `mirr`; did you mean `mir`?``），其後接著 `run .. for usage` 行。無論 `--color` 與 `--message-format` 位於命令列的何處，它都依它們的指定印出：在 `--message-format=json` 下，它是標準輸出上的一行 JSON，不含那一行。

sysroot 不來自 `NABI_HOME`：它相對於執行檔。`nabic` 讀取 `NABI_HOME` 只是為了把已擷取之相依套件的路徑顯示在 `$NABI_HOME/cache/` 下。若 nabic 本身 panic（編譯器內部錯誤），它會印出 panic 訊息，然後是 `nabic: internal compiler error` 以及三行 `note:`：說明這是 nabi 的錯誤，以及該在哪裡連同觸發它的程式碼回報；版本與主機（`nabic <version> (<commit> <date>) on <triple>`）；以及命令列；當它當時位於某個編譯器階段中時，第四行 `note: in pass: <pass> > <pass>` 會指出那些階段（名稱與 `NABI_TIME_PASSES` 所用的相同）。它以 `70`（`EX_SOFTWARE`）結束，有別於程式的 panic，後者以 `101` 結束。`nabi` 以相同方式回報它自己的內部錯誤（`nabi: internal error`，結束碼 `70`）。

若標準輸出或標準錯誤是讀取端已離開的管道，`nabic` 會不帶任何訊息地停止：在 Unix 上它被 `SIGPIPE` 終止（shell 回報 `141`）；在 Windows 上它以 `1` 結束。這不是內部錯誤。

## 9.1.7. 環境變數

影響 `nabic` 的環境變數如下：

| 變數              | 意義                                                                                     |
|-------------------|------------------------------------------------------------------------------------------|
| `NABI_CC`          | 選用的 C 編譯器驅動程式，取代 `nabic` 的連結器進行最終連結（[9.2.14](902-aot-pipeline.md#9214-連結執行檔)） |
| `NABI_VERBOSE`     | 等同於 `--verbose`                                                                       |
| `NABI_TIME_PASSES` | 為 `1` 時，在 stderr 上印出各階段的實際耗時（[9.9.5](909-debug-tools.md#nabi_time_passes)） |
| `__NABI_ENV`、`__NABI_ENV_LIBS` | 當 `nabi` 從某個環境執行腳本時由它設定（內部用）：環境的名稱，以及安裝在那裡、以逗號分隔的函式庫。使用其中之一卻未宣告它的腳本會得到指出它的提示 |
| `__NABI_CLI`       | 當 `nabi` 執行 `nabic` 時由 `nabi` 設為 `nabi`（內部用）：用法錯誤（9.1.6）、`--help` 以及錯誤與警告行的 `nabic:` 前綴會寫 `nabi`，而不是 `nabic` |
| `NABI_TARGET_DIR`  | `target/` 的位置（預設：在套件內）                                                       |
| `NABI_INCREMENTAL` | 前端保存套件之剖析結果與降階後主體的位置（預設 `target/<profile>/incremental/`；`0`：不保存）；函式庫所收集的項目保存在 `lib/nabi-cache/library/`（[9.2.16](902-aot-pipeline.md#9216-增量建置)） |
| `NABI_INCREMENTAL_VERIFY` | `1`：重新降階每個主體，有差異時失敗（[9.2.16](902-aot-pipeline.md#9216-增量建置)） |
| `NABI_FN_CACHE` | `0`：既不讀取也不寫入工具鏈的函式快取，即建置所保存的標準函式庫程式碼（[9.2.16](902-aot-pipeline.md#9216-增量建置)） |
| `NABI_CACHE_MAX_SIZE` | 工具鏈快取的上限：位元組數，或帶 `K`、`M` 或 `G` 的數字（預設 `5G`）；`off`：沒有上限（[9.2.16](902-aot-pipeline.md#9216-增量建置)） |
| `NABI_HOME`        | 已擷取之相依套件的顯示位置（9.1.6；預設 `~/.nabi`）                                      |
| `NABI_RUNNER`      | `<program> [args]`：`nabi run` / `test` / `bench` 執行其他目標之執行檔時所透過的程式（[9.10.8](910-nabi-frontend.md#其他目標的執行檔)） |
| `LIBRARY_PATH`    | 原生函式庫的搜尋目錄，在 `-L` 之後                                                       |
| `NO_COLOR`        | 非空：`--color=auto` 不使用色彩（9.1.5）                                                 |
| `CLICOLOR_FORCE`  | 非空且不是 `0`：即使串流不是終端機，`--color=auto` 也使用色彩，除非設定了 `NO_COLOR`（9.1.5） |
| `COLORTERM`       | `truecolor` 或 `24bit`：色彩使用 24 位元值（9.1.8）                                      |
| `TERM`            | `dumb`：`--color=auto` 不使用色彩；含有 `256color`：色彩使用 256 色調色盤（9.1.8）       |

未設定或為空的變數採用其預設值。所列值以外的值會在 stderr 上印出一行警告，指出該變數、該值與可接受的值，而該變數採用其預設值。

`nabi` 把內部變數 `__NABI_CLI`、`__NABI_ENV` 與 `__NABI_ENV_LIBS` 交給 `nabic`；`nabic` 不讀取它們的其他拼法。`nabi run`、`nabi test` 與 `nabi bench` 會從它們所啟動之程式的環境中移除名稱以 `__NABI` 開頭的每個變數（在 Windows 上不論名稱的大小寫）；程式看到的其餘環境保持不變。

## 9.1.8. 輸出格式

### 診斷訊息

診斷訊息以下列格式印出。

```
error[E0027]: cannot find value `count` in this scope
  --> src/main.nabi:5:13
   |
 5 |     let x = count + 1;
   |             ^^^^^
   |
   = help: did you mean `counter`?
```

診斷使用[原始碼檔案](../001-lexical/101-source-files.md)中定義的檔案 + 行 + 欄單位。

標頭是 `<severity>[<code>]: <message>`，而 `--> path:line:col` 給出主要位置。主要範圍以 `^` 標底線，次要範圍以 `-` 標示；位於另一個檔案中的次要範圍以 `::: path:line:col` 引入。結尾的 `= note:` 行補充脈絡，`= help:` 行建議修正方式。訊息不引用本規格的章節；JSON 格式會列出它們（[9.9.4](909-debug-tools.md#994-診斷--錯誤訊息)）。

### 建置進度

進度以靠右對齊的動詞顯示在 stderr 上。有 `-q` 時不印出任何東西。

```
   Compiling json v0.1.0
   Compiling foo (path/to/foo)
    Building target/debug/foo
    Finished in 1.23s
     Running `target/debug/foo`
```

| 動詞        | 時機                                          |
|-------------|-----------------------------------------------|
| `Compiling` | 前端開始（套件名稱與目錄）；`check` 與 `lint` 時為 `Checking`，`doc` 時為 `Documenting`。`nabi` 所擷取的相依套件（位於 `$NABI_HOME/cache/` 下）改為顯示 `<name> v<version>` |
| `Building`  | 產生執行檔 / 封存檔（`build`）                |
| `Built`     | 單獨檔案的執行檔已複製到其旁邊（`build`）：`Built hello (use -o to put it elsewhere)`。在沒有 `--emit-harness` 的情況下建置了測試或效能測試 harness（`test`、`bench`），後面接著一則提示，指出以 `nabi test` 或 `nabi bench` 執行它 |
| `Wrote`     | 已寫出一個 `--emit` 輸出                      |
| `Reused`    | 有 `-v` 時，在前端之後：有多少檔案的剖析結果與多少主體的降階結果來自增量快取（[9.2.16](902-aot-pipeline.md#9216-增量建置)） |
| `Units`     | 有 `-v` 時，在程式碼產生期間：有多少程式碼單元被重複使用（來自增量快取或函式快取），有多少被編譯 |
| `Finished`  | 建置完成（自指令開始以來的時間）              |
| `Running`   | 執行程式（`run`）：相對於目前目錄的執行檔；單獨的檔案以其檔案名稱表示（`Running hello.nabi`），有 `-v` 時是快取的執行檔 |
| `Created`   | 已建立套件（`new`），然後是接下來要在其下執行的指令（[9.10.2](910-nabi-frontend.md#9102-nabi-new--nabi-init建立套件)） |
| `Removed`   | 已刪除 `target/`（`clean`）                   |
| `Fresh`     | 已是最新；接著是那次建置的警告                |
| `Checked`   | `check` 完成（錯誤與警告數量）                |
| `Generated` | 已寫出文件（`doc`）                           |
| `Formatted` | 已改寫檔案（`fmt`）：在結尾顯示其數量（[`nabi fmt`](910-nabi-frontend.md#nabi-fmt格式化工具)） |

在前端失敗的建置（`build`、`run`、`test`、`bench`、`doc`），在診斷之後以 `error: could not build <name> (<n> error(s), <m> warning(s))` 結束，計數方式與 `check` 相同。`-q` 與 `--message-format=json` 會省略它。

`nabi` 以相同方式印出它自己的進度：`Added`、`Installed`、`Updated`、`Downloading` 等（[9.10](910-nabi-frontend.md)）。

### 色彩

色彩開啟時（9.1.5），輸出使用這個調色盤。其色深跟隨終端機：`COLORTERM` 為 `truecolor` 或 `24bit` 時使用 24 位元值，否則在 `TERM` 含有 `256color` 時使用 256 色調色盤，否則使用 16 種標準色彩。

| 部分                                  | 色彩                     | 24 位元     | 256   | 16             |
|---------------------------------------|--------------------------|-------------|-------|----------------|
| 狀態動詞（`Compiling`）、清單標記 `*` | 天藍色，粗體             | `#38bdf8`   | `81`  | 亮青色         |
| `error`                               | 紅色，粗體               | `#ff5f5f`   | `203` | 亮紅色         |
| `warning`                             | 琥珀色，粗體             | `#e0b057`   | `179` | 黃色           |
| `note`                                | 天藍色                   | `#38bdf8`   | `81`  | 亮青色         |
| `help`                                | 淺粉紅色                 | `#f9a8d4`   | `218` | 亮洋紅色       |
| 診斷的邊欄（`-->`、`\|`、行號）       | 天藍色，粗體             | `#38bdf8`   | `81`  | 亮青色         |

色彩絕不單獨承載意義：標籤（`error:`、`warning:`、`note:`、`help:`）與狀態動詞無論有無色彩都會印出。標示其中一個項目的清單（`nabi toolchain list`、`nabi env list`）在該項目之前加上 `*`，之後加上 `(default)` 或 `(active)`。經由管道的輸出、`--message-format=json` 與 `-q` 不受影響。

### JSON 輸出（供工具整合）

為了工具整合，輸出也可以 JSON 形式產生。

```
nabic build --message-format=json
```

在這個模式下，每個診斷在 stdout 上成為一行 JSON 物件，便於 IDE 與 CI 工具剖析。物件的結構定義於 [9.9.4](909-debug-tools.md#994-診斷--錯誤訊息)。

## 9.1.9. 標準輸入

在引數位置寫 `-`，如 `nabic check -`，表示標準輸入，供管道或腳本使用。來自標準輸入的原始碼會被當作匿名可執行套件的 `main.nabi` 檢查。

## 9.1.10. 與外部工具的相容性

`nabic` 是低階的編譯器；相依套件解析是前端 `nabi` 的工作（[9.10](910-nabi-frontend.md)）。`nabi` 呼叫 `nabic`，經由 `--extern` 傳入已解析之相依套件的根（[9.2.3](902-aot-pipeline.md#923-相依套件解析)）。其他建置系統也可以用相同方式直接呼叫 `nabic` 來整合。破壞 CLI 相容性的變更依 SemVer 規則管理。
