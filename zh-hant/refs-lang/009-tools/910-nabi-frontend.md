# 9.10. `nabi`：套件 / 建置前端與工具

`nabi` 是 cargo 風格的前端。它負責資訊清單 `nabi.nabion` 與鎖定檔 `nabi.lock`、相依套件、環境、工具鏈釘選以及腳本的相依套件，並把編譯委派給編譯器 `nabic`（[9.1](901-nabic-cli.md)）。各工具（run、lint、REPL、Jupyter、LSP、格式化工具、文件……）是 `nabi` 本身的一部分（9.10.8）。執行環境是 `nabir`（`libnabir.a`）。

沒有相依套件的套件只憑檔案慣例就能建置，不需要資訊清單（[0.3.4](../000-meta/003-source-units.md#034-清單nabinabion是選用的)）。此時 `nabi build` 等同於 `nabic build`。

## 9.10.1. 子指令與全域選項

| 子指令                                                            | 動作                                    |
|-------------------------------------------------------------------|-----------------------------------------|
| `new`、`init`                                                     | 建立套件（9.10.2）                      |
| `add`、`remove`、`update`、`deps`                                 | 相依套件（9.10.4）                      |
| `search`、`publish`、`yank`、`login`、`logout`                    | 登錄庫套件（9.10.4a）                   |
| `env`                                                             | 環境（9.10.5）                          |
| `component`                                                       | 選用元件：官方套件，供沒有網路的建置使用（9.10.4a） |
| `toolchain`、`cache`                                              | 工具鏈及其建置快取（9.10.6）            |
| `install`、`uninstall`                                            | 供 REPL / Jupyter 使用的函式庫，以及執行檔（9.10.5a） |
| `build`、`check`、`emit`                                          | 附上相依套件後委派給 `nabic`            |
| `test`、`bench`                                                   | `nabic` 附上相依套件建置 harness；`nabi` 執行它（[9.1](901-nabic-cli.md)） |
| `run`、`dev`、`lint`、`fix`、`repl [path]`、`jupyter`、`lsp`、`doc` | 工具，附上相依套件（9.10.8）          |
| `fmt`、`clean`、`bindgen`、`inspect`、`demangle`、`completions <shell>` | 工具，不附相依套件（9.10.8）     |
| `cc`                                                              | C 編譯器 `nabicc`，引數原樣傳入（[9.13](913-c-compiler.md)） |
| `version`、`help`                                                 | 版本；說明（見下文）                    |
| （無）                                                            | 與 `nabi repl` 相同：啟動 REPL          |

委派的子指令執行同名的 `nabic` 子指令；工具則在 `nabi` 自己的行程中、連結了編譯器的情況下執行（9.10.8）。對兩者，`nabi` 只消耗它自己的全域選項（`--env` / `--offline` / `--locked` / `--frozen` / `--cwd` / `--color`，位於 `--` 之前的任何位置），把其餘引數原樣傳給編譯器，並加上下列內容。對 `test` 與 `bench`，`nabi` 讓 `nabic` 建置 harness（`--emit-harness`）而不是執行它，然後自己執行 harness（[9.1](901-nabic-cli.md)）；執行器不屬於編譯器，其選項（見下文）是 `nabi` 的選項。

- 每個相依套件一個 `--extern name=<source directory>`。它們依照 `nabi.lock` 準備，相依套件排在其相依者之前（即 `nabi deps` 印出的順序）。預先建置由 `nabic` 進行（[9.2.3](902-aot-pipeline.md#預先建置的相依套件)）。
- 每個套件的每個相依套件一個 `--extern-dep <package>=<name>`（`pkg` 表示正在建置的套件），讓每個套件只看得到它自己的相依套件（[5.7.14](../005-modules/507-paths.md#5714-套件之間的路徑)）。
- 套件自己的 feature 以 `--cfg feature=<f>` 傳入，每個相依套件的 feature 以 `--extern-feature <name>=<f>` 傳入（9.10.4c）。build / run / test / bench / check / lint / fix / doc / emit / repl / lsp / jupyter / deps 的 feature 選項 `--features <f,..>`（可重複；也可以是 `<dep>/<f>`）、`--all-features` 與 `--no-default-features` 由 `nabi` 消耗（位於 `--` 之前）。
- 資訊清單的 `lints` 以 `-A` / `-W` / `-D` 傳入（位於引數之前）。資訊清單是路徑所選擇之專案的資訊清單：對 `.nabi` 檔案而言，是該檔案所在或其上方最近一個含有 `src/main.nabi`、`src/lib.nabi` 或 `nabi.nabion` 的目錄。位於套件 `src/` 中的檔案選擇該套件（[9.1.1](901-nabic-cli.md#911-一般形式)）。位於某個有 `nabi.nabion` 之目錄中其他地方的單獨檔案（套件 `src/` 旁邊的腳本，或只有 `nabi.nabion` 的目錄），會以該資訊清單的 `lints`（以及 `lints_from`）與相依套件，作為單一檔案套件編譯。
- 在套件之外（環境、腳本），`--target-dir` 設為該環境或腳本的 `target/`。

`nabi cc` 既不是委派的子指令也不是工具：它以 `cc` 之後的每個引數原樣執行工具鏈的 `nabicc`，不加上任何東西（[9.13.1](913-c-compiler.md#9131-nabicc-與-nabi-cc)）。

`nabi repl <path>` 與 `run` 一樣，從該路徑的專案解析相依套件與 lint，並使用資訊清單的名稱（`--name`）。

`nabi test` 的執行器接受下列選項（位於 `--` 之前）；它們不會傳給 `nabic`：

- `--threads=N` 設定同時執行的測試行程數量。預設為 `NABI_TEST_THREADS`，未設定時為 CPU 數量；不是正數的 `NABI_TEST_THREADS` 值會引發警告（`nabi: warning: ignoring NABI_TEST_THREADS=0 (expected a positive number)`）並被忽略。
- `--show-output` 也顯示通過之測試的 stdout / stderr。失敗之測試的輸出一律會顯示。
- `--format=human|json` 選擇輸出形式（預設 `human`）。`json` 在 stdout 上為每個測試寫出一筆 JSON 紀錄以及一個摘要，人類可讀的文字則寫到 stderr（[9.9.12](909-debug-tools.md#9912-ci--測試整合)）。
- `--exact` 讓篩選器與 `--skip` 比對完整的測試名稱：`nabi test --exact math::adds` 執行該測試，而不執行 `math::adds_wide`。
- `--skip <pat>` 排除 `<pat>` 所比對到的測試，規則與篩選器相同（可重複）。
- `--ignored` 只執行 `#[ignore]` 測試；`--include-ignored` 把它們與其餘測試一起執行。
- `--timeout=SECS` 終止執行時間更長的測試，並將其判為逾時失敗。預設為 `NABI_TEST_TIMEOUT`，未設定時為 600；`0` 表示沒有限制。不是整數秒的 `NABI_TEST_TIMEOUT` 值會引發警告並被忽略。
- `--list` 依每行一個印出篩選器所保留之測試的名稱，不執行任何測試（結束碼 `0`）；`#[ignore]` 測試後面接著 ` (ignored)`。有 `--format=json` 時，每個名稱改為一筆紀錄：`{"schema":1,"type":"listed","kind":"test","name":"adds"}`，被忽略者帶有 `"ignored":true`。

`--doc` 與 `--no-doc` 傳給 `nabic`：只有文件測試，或一個都沒有（[9.1.2](901-nabic-cli.md#文件測試)）。

`nabi bench` 的執行器接受下列選項：

- `--iters=N` 固定每個樣本的呼叫次數，而不進行校準。
- `--samples=N` 是量測的樣本數（預設 10）。
- `--format=human|json|nabibench` 選擇輸出形式（預設 `human`）：`json` 與測試相同，`nabibench` 是 [9.1.2](901-nabic-cli.md#nabic-bench效能測試) 的穩定行。
- `--list` 與測試一樣印出篩選器所保留之效能測試的名稱，不執行任何效能測試。

兩者都經由其執行器執行另一個 `--target` 的執行檔（[其他目標的執行檔](#其他目標的執行檔)）。

錯誤的值是用法錯誤，錯誤訊息會指出該寫什麼：`` `--threads` takes a positive number, not `0` ``、`` `--timeout` takes whole seconds (0: no limit), not `1.5`: `--timeout=120` ``、`` `--format` takes human or json, not `xml` ``。執行器印出的內容與 `nabic test` 及 `nabic bench` 一起說明（[9.1.2](901-nabic-cli.md#nabic-test測試)）。

選項、輸出與結束碼都遵循 `nabic` 的規定（[9.1](901-nabic-cli.md)）。`nabi` 處理已關閉之標準輸出或標準錯誤的方式與 `nabic` 相同（[9.1.6](901-nabic-cli.md#916-結束碼)）。`nabils` 是等同於 `nabi lsp` 的獨立執行檔。

`nabi help`（也可以是 `nabi --help`）印出一頁簡短說明：分組列出的子指令，每個一行，以及全域選項。`nabi help <subcommand>`（也可以是 `nabi <subcommand> --help`）印出該子指令自己的說明：其用法形式、描述、範例與其選項。對委派的子指令，它是以 `nabi` 改寫的 `nabic` 說明，加上 `nabi` 的 feature 選項，對 `test` 與 `bench` 還會加上執行器的選項。

未知的子指令或選項是用法錯誤（結束碼 2）。訊息在有最接近的子指令或選項時會指出它，包括常見的其他名稱（`compile` 對應 `build`、`rm` 對應 `remove`），其後接著 ``run `nabi help <subcommand>` for usage``。索引中沒有的套件或原生套件會指出那裡最接近的一個，以及要執行的 `nabi search`。`nabi` 為它所執行的 `nabic` 設定 `__NABI_CLI=nabi`，因此 `nabic` 的用法錯誤、說明以及其錯誤與警告行的前綴也會寫 `nabi`（[9.1.7](901-nabic-cli.md#917-環境變數)）。

只執行 `nabi [options]` 而不帶子指令，會像 `nabi repl` 一樣啟動 REPL。

```
nabi                 # = nabi repl
nabi --env sci       # = nabi repl --env sci
```

`nabi [options] <file> [args..]` 執行一個腳本：它就是 `nabi run -q <file> -- args..`。檔案之後的每個引數都原樣傳給程式，結束碼就是程式的結束碼。當該字詞含有 `/`（在 Windows 上也包括 `\`）或副檔名為 `.nabi` 時，它就是檔案；沒有任何子指令符合這些條件，因此子指令絕不會被當作檔案。搭配 shebang 行（[1.2.3](../001-lexical/102-lexical-structure.md#shebang-行)），`./script.nabi args` 會執行該腳本。

委派的指令或工具的路徑，是其第一個既不是選項也不是選項之值的引數，因此選項可以出現在它之前（`nabi run --release t.nabi`）。不存在的路徑是錯誤 `` `<path>` does not exist ``，在其他任何事情之前回報（而不是「not inside a package」）；對 `test` 與 `bench`，由於其第一個字詞可能是篩選器，只有以 `.nabi` 結尾的字詞才會被當作路徑。

`nabi` 的全域選項如下：

| 選項             | 意義                                                              |
|------------------|-------------------------------------------------------------------|
| `--env <name>`   | 在該具名環境中工作（REPL、Jupyter、`nabi install`）；用在作用於套件的指令上是錯誤（[9.10.5](#9105-環境)） |
| `--offline`      | 不使用網路。git 相依套件必須已在快取中                            |
| `--locked`       | 若 `nabi.lock` 需要改變，就不改變它並失敗（供 CI 使用）           |
| `--frozen`       | 同時為 `--locked` 與 `--offline`                                  |
| `--cwd <dir>`    | 執行之前變更目錄                                                  |
| `-q`、`--quiet`  | 減少輸出                                                          |
| `-v`、`--verbose` | 更多輸出（也會傳給 `nabic`）                                     |
| `--color <when>`、`--no-color` | `auto`、`always` 或 `never`：色彩，決定方式與 `nabic` 相同（[9.1.5](901-nabic-cli.md#915-全域選項)）；以 `--color=<when>` 傳給它所執行的 `nabic` |

全域選項也可以接在子指令之後：`nabi new app -q` 就是 `nabi -q new app`。

環境變數有 `NABI_HOME`（預設 `~/.nabi`）、`NABI_ENV`（9.10.5）、`NABI_TOOLCHAIN`（9.10.6）、`NABI_REGISTRY_TOKEN`（預設登錄庫的 API 權杖，9.10.4a）以及 `NABI_DIST`（發行伺服器，9.10.6）。`NO_COLOR`、`CLICOLOR_FORCE`、`COLORTERM` 與 `TERM` 設定色彩的方式與 `nabic` 相同（[9.1.7](901-nabic-cli.md#917-環境變數)）。

## 9.10.2. `nabi new` / `nabi init`：建立套件

```
nabi new [--lib] [--bin] <name>
nabi init [--lib] [--bin] [dir]
nabi init --script <file.nabi>
```

`nabi new` 建立目錄 `<name>` 並填入預設檔案。`--bin`（預設）建立會印出問候語的 `src/main.nabi`；`--lib` 建立帶有一個範例函式與一個 `#[test]` 的 `src/lib.nabi`；兩者一起使用時兩者都建立，此時 `src/main.nabi` 經由 `use <name>::add;` 呼叫函式庫的函式（[0.3.1](../000-meta/003-source-units.md#031-執行檔與函式庫)）。若 `<name>` 已存在，則是錯誤。

`nabi init` 把既有的目錄（省略時為目前目錄）變成套件。若該目錄已含有 `src/main.nabi` 或 `src/lib.nabi`，則是錯誤。

`nabi init --script <file.nabi>` 把一個腳本（[9.10.7](#9107-單一檔案腳本的行內資訊清單)）變成位於該腳本目錄中、以檔案名稱（去掉 `.nabi`）命名的套件。`nabi.nabion` 取得腳本各區塊合併後的 `deps`（以及 `native`），路徑改寫為相對於它；腳本變成 `src/main.nabi`，其 `mod` 所載入的每個檔案都以相同的相對路徑移到 `src/` 之下，各自去掉其區塊。若腳本位於套件內、被移動的檔案或 `nabi.nabion` 已存在，或某個子模組檔案位於腳本目錄之外，則是錯誤。

兩個指令也都會以標準形式（[NABION 3.8.4](../../refs-standard-libraries/300-std/308-nabion.md#384-正規形式)）寫出 `nabi.nabion`，其 `nabi_version` 是寫出它的 `nabi` 的版本（`nabi_version: "0.1.3"`，省略其預發行標記），並釘選在該 `nabi` 的系列上（任何 0.1.x 都寫 `toolchain: "0.1"`，9.10.6），讓套件在更新的系列成為預設時仍停留在該系列；還會寫出內容為 `target/` 的 `.gitignore`，並在不位於 git 儲存庫內時執行 `git init`。`init` 使用依命名規則（9.10.3）調整過的目錄名稱作為套件名稱：Windows 裝置名稱會加上前綴 `pkg-`（`nul` → `pkg-nul`），名稱會截短為 64 個字元。

在 `Created` 行之後，兩者都會在訊息下方印出接下來要執行的指令：`cd <dir> && nabi run`，或對只有函式庫的套件印出 `nabi test`；當套件就是目前目錄時，不帶 `cd`。在 Windows 上，由於 Windows PowerShell 沒有 `&&`，`cd <dir>` 與該指令分成兩行。`-q` 兩者都不印出。

```
     Created executable package (hello)
             cd hello && nabi run
```

在 Windows 上：

```
     Created executable package (hello)
             cd hello
             nabi run
```

```
hello/
├── .gitignore        <- target/
├── nabi.nabion
└── src/
    └── main.nabi
```

## 9.10.3. `nabi.nabion` / `nabi.lock`

套件根部的 `nabi.nabion` 是資訊清單，同一位置的 `nabi.lock` 是鎖定檔。兩者都是 [NABION](../../refs-standard-libraries/300-std/308-nabion.md) 格式。名稱的拼寫完全如此（全部小寫）。只有大小寫不同的檔案（`Nabi.nabion` 等）會被忽略並發出警告，不會被讀取。

資訊清單是單一的 NABION 值，即一個 `Package { .. }` 結構（根名稱 `Package` 可以省略）。它只會被讀取，絕不會被編譯或執行。

```nabion
Package {
    name: "geom",                    // string, optional
    version: "0.3.1",                // string, optional
    nabi_version: "0.4.1",           // string, optional (oldest nabi it builds with; `nabi publish` needs it)
    toolchain: "0.4",                // string, optional (toolchain pin, 9.10.6)
    description: "2D geometry",      // string, optional
    license: "MIT OR Apache-2.0",    // string, optional (SPDX expression)
    repository: "https://example.com/geom.git",   // string, optional
    authors: ["Ann Lee <ann@example.com>"],       // list of strings, optional
    homepage: "https://example.com/geom",         // string, optional
    documentation: "https://example.com/geom/doc",   // string, optional
    readme: "README.md",             // string, optional (path relative to nabi.nabion)
    keywords: ["geometry", "math"],  // list of strings, optional (at most 5)
    categories: ["mathematics"],     // list of strings, optional (at most 5)
    license_file: "LICENSE",         // string, optional (path relative to nabi.nabion)
    include: ["src/**", "README.md"],   // list of globs, optional (what the package holds)
    exclude: ["bench/**"],           // list of globs, optional (left out of it)
    links: "z",                      // string, optional (the native library it links)
    default_run: "geom",             // string, optional (the executable `nabi run` runs)
    deployment_target: "13.0",       // string, optional (the oldest macOS the program runs on;
                                     // { macos: "13.0", ios: "16.0" } for iOS too)
    metadata: { docs: { all_features: true } },   // table, optional (for other tools)
    publish: true,                   // bool, optional (false: `nabi publish` refuses)
    deps: {                          // dependency name (string) -> Dep
        "util": Dep::path("../util"),
        "parser": Dep::git("https://example.com/parser.git", tag: "v1.2"),
        "http": "0.9",                                   // from the index (= Dep::version("0.9"))
        "json": Dep::version("^1.4", optional: true),    // only with a feature (9.10.4c)
        "csv": Dep::version("^0.3", registry: "https://example.com/index"),
        "j5": Dep::version("^0.2", package: "json5"),    // the package `json5`, `j5` in code
        "nix": Dep::version("^0.4", target: "cfg(target_family = \"unix\")"),
    },
    dev_deps: {                      // same form as deps. test / bench only
        "fixtures": Dep::path("tests/fixtures"),
    },
    features: { "default": ["json"], "json": ["dep:json"], "tls": ["http/tls"], "full": ["json", "tls"] },
    native: {                        // C libraries (9.10.4b)
        "zlib": "^1.3",                                  // a prebuilt native package
        "png": Dep::native(url: "vendor/png.nabion", sha256: "<64 hex>"),
        "sqlite3": Dep::system(brew: "sqlite", apt: "libsqlite3-dev"),   // installed on the machine
        "Foundation": Dep::framework(),                  // an Apple framework
    },
    lib: { kind: "cdylib" },         // optional (also a shared library for C, 9.3.3; "plugin": 9.12)
    c: { sources: ["c/shim.c"], include: ["c"], define: { "N": "1" }, std: "c17" },   // optional
    lints_from: "..",                // string, optional (starts from ../nabi.nabion's lints)
    lints: { "dead_code": "allow", "nonstandard_style": "deny" },
    profile: { "release": { overflow_checks: false } },   // per build profile
}
```

- 值是 NABION 字面值（字串、`{ "k": v }` 字典、空字典 `{}`）以及 `Dep::path("..")` / `Dep::git("url", rev | tag | branch: "..")` / `Dep::version("req", registry: "url")`。`Dep::git` 最多接受 `rev` / `tag` / `branch` 其中之一，`Dep::version` 的 `registry` 是選用的。只是一個字串的相依套件值是 `Dep::version` 的簡寫。
- `Dep::path("..", version: "req")` 為路徑相依套件加上版本需求。建置使用該路徑，而那裡的套件必須有需求所接受的版本（`` dependency `linalg` (path ..) is version 0.2.0, which its `version: "^0.1"` does not accept; .. ``）；`nabi publish` 會在其位置寫入該需求（9.10.4a）。並行開發的套件就是以這種方式發布的。
- `Dep::path` / `Dep::git` / `Dep::version` 也接受 `optional: true`、`features: ["<f>", ..]` 與 `default_features: false`（9.10.4c）；選用的開發用相依套件是錯誤。
- `package: "<name>"` 重新命名相依套件：`<name>` 是套件，鍵是這個套件的程式碼對它的稱呼（`"j": Dep::version("^1", package: "json5")` 就是 `json5`，以 `use j::..` 使用）。每個相依者選擇自己的名稱，而且只有它自己的程式碼看得到，因此兩個套件可以用不同的名稱稱呼同一個套件；它仍是同一個套件，只建置一次，其型別在每個名稱下都相同。一個相依者以兩個鍵相依於同一個套件是錯誤（`` the root package depends on package `json5` twice, as `j` and `json5`; depend on it once ``）。
- `target: "<target>"` 把相依套件限制在對相符目標的建置：目標名稱（`x86_64-unknown-linux-gnu`），或是對 [6.3.2](../006-meta-system/603-cfg.md#632-述詞的種類) 之目標鍵（`target_os`、`target_arch`、`target_family`、`target_env`、`target_endian`、`target_pointer_width`）的 `cfg(<predicate>)`，搭配 `all(..)`、`any(..)`、`not(..)`。其他任何東西在讀取資訊清單時都是錯誤。解析與鎖定檔對每個目標都涵蓋該相依套件；對其他目標的建置會省略它，連同它會啟用的 feature。
- 相依套件上的 `plugin: true` 把它建置為程式旁邊的外掛，不連結它（[9.12.2](912-plugins.md#9122-建置)）；外掛之相依套件上的 `bundle: true` 把它連結進外掛，而不是從程式取得。在不是外掛的套件中使用 `bundle` 是錯誤（`` `bundle: true` on `noise` belongs in a plugin's dependencies: give this package `lib: { kind: "plugin" }`, or remove `bundle: true` ``），同時帶有兩者的相依套件也是錯誤。
- `Dep::registry` 只用於 `nabi.lock`；在 `nabi.nabion` 中使用它是錯誤。
- 套件名稱（`deps` 的鍵、`name`）是以字母開頭的小寫 ASCII `[a-z0-9_-]+`（以數字、`-` 或 `_` 開頭是錯誤）。名稱 `std`、`core` 與 `bare` 是標準函式庫的各層級，隨工具鏈提供：套件、原生套件或登錄庫的名稱是其中之一是錯誤（`` `std` is the standard library, which ships with the toolchain; name the package something else, e.g. `std_util` ``），以它執行 `nabi new` / `nabi init` 也是；`nabi add std` 會說明沒有東西可以新增。`deps` 的鍵是其中之一是錯誤，其說明為 `` std ships with the toolchain; remove this line ``，除非它帶有 `package: "<name>"`：此時它是以該層級之名稱稱呼的那個套件（[5.7.4](../005-modules/507-paths.md#574-stdcorebare標準函式庫層級)）。`package: "std"` 是相同的錯誤。程式碼也必須能夠指名該套件，因此其程式碼名稱（見下文）不得是關鍵字（1.5：`fn`、`match`、`self`……）或路徑起點（5.7.1：`pkg`、`sup`）；執行檔中的 `use pkg::add;` 一律表示它自己的根（`` `pkg` already names a package's own root (`pkg::`), so code can't name a package called that; name the package something else, e.g. `pkg_util` ``）。鍵是字串（`"nabi-syntax": Dep::version("^0.1")`）。
- 套件名稱也必須是每個支援的作業系統都能作為檔案或目錄儲存的名稱：不是 Windows 裝置名稱（`con`、`prn`、`aux`、`nul`、`com1`-`com9`、`lpt1`-`lpt9`，不論大小寫），且最多 64 個字元（`` `nul` is a device name on Windows, so no file can be called that; name the package something else, e.g. `my_nul` ``）。`nabi new` / `nabi init --script` / `nabi add` / `nabi publish` 會拒絕這樣的名稱。在被讀取的 `nabi.nabion` 中（其 `name` 以及 `deps`、`dev_deps`、`native` 的鍵），它是帶有 `file:line:col` 的警告，建置繼續進行。
- `version` 是一個版本（`major.minor.patch`，9.10.4a）。不是版本的值在讀取資訊清單時是警告（`` `banana` is not a version (major.minor.patch): it needs three numbers; `nabi publish` needs one, e.g. `version: "0.1.0"` ``），在 `nabi publish` 時是錯誤。
- `nabi_version` 是套件能用以建置的最舊工具鏈，是完整的版本：`x.y.z`，不多也不少（讀取資訊清單時為 `` `nabi_version: "0.1"` is not a toolchain version; write all three parts, e.g. `nabi_version: "0.1.3"` ``）。它接受該工具鏈以及其系列中較新的工具鏈（[9.10.6 相容性](#相容性)）：`"0.1.3"` 接受 0.1.3 到最後一個 0.1.x，`"1.4.2"` 接受 1.4.2 到最後一個 1.x。`nabi publish` 需要它（9.10.4a），而解析依它挑選版本。它不是 `toolchain`，後者釘選這個專案用以建置的工具鏈（9.10.6）。
- 在程式碼中，把 `-` 換成 `_` 的名稱是指稱該套件的頂層路徑（`nabi-syntax` → `use nabi_syntax::..`、`--extern nabi_syntax=..`）。
- 在登錄庫以及同一個相依圖內，名稱的比較把 `-` 與 `_` 視為相同並忽略大小寫。因此 `nabi-syntax` 與 `nabi_syntax` 無法共存。
- 相依套件必須有 `src/lib.nabi`；該函式庫就是其相依者所得到的東西，無論套件是否也有 `src/main.nabi`。只有 `src/main.nabi` 的相依套件是錯誤，錯誤訊息會說明它是程式，應使用 `nabi install`。間接相依套件是相依套件的 `deps`（`dev_deps` 不會傳遞）。
- `native` 把原生套件名稱（自成一個命名空間：套件與原生套件可以同名）對應到 `Dep::native("req"[, registry: "url"])`、需求字串，或 `Dep::native(url: "<url or path relative to the manifest>", sha256: "<64 hex digits>")`（9.10.4b）。它也把名稱對應到 `Dep::system(..)`，即安裝在機器上的函式庫，以及 `Dep::framework()`，即 Apple framework，這些在建置時尋找（9.10.4b）。在 `deps` / `dev_deps` 中使用 `Dep::native` 是錯誤。圖中每個套件的原生相依套件都會連結進執行檔（只經由 `dev_deps` 到達者，只用於 test / bench）。
- `lib: { kind: "nabi" | "cdylib" | "staticlib" | "plugin" }`（該紀錄可以命名為 `Lib`）說明函式庫建置出什麼。`"nabi"` 是預設值，即其他 Nabi 套件所使用的函式庫；使用 `"cdylib"` 時，`nabi build` 會傳入 `--crate-type=cdylib` 以及帶有套件 `version` 的 `--lib-version`（[9.1.2](901-nabic-cli.md#nabic-buildaot-建置)），因此建置也會產生共用函式庫及其 C 標頭檔（[9.3.3](903-package-archive.md#933-靜態與共用函式庫)），除非命令列給出了 `--crate-type`。使用 `"staticlib"` 時，它傳入 `--crate-type=staticlib`：建置也會產生一個 C 與 Swift 可以單獨連結的靜態函式庫，連同其標頭檔與模組對應檔（9.3.3）；在它旁邊的 `xcframework: true` 還會傳入 `--xcframework`，用在其他種類上則是錯誤。使用 `"plugin"` 時，它傳入 `--crate-type=plugin`：該套件是由 Nabi 程式載入的共用函式庫（[9.12](912-plugins.md)）；帶有 `src/main.nabi` 的 `"plugin"` 套件是錯誤，對它執行 `nabi run` 也是。其他種類是錯誤，錯誤訊息會列出這四種。
- `c: { sources, include, define, std, flags }`（該紀錄可以命名為 `C`）是由 nabicc，即 Nabi 自己的 C 編譯器（絕不是系統的 C 編譯器），建置進套件的 C 檔案：`sources`（相對於資訊清單的路徑，至少一個）、`include`（標頭檔目錄，然後是套件之 `Dep::system` 函式庫的目錄）、`define`（`{ "NAME": "value" }`）、`std`（`c89`、`c99`、`c11`、`c17`、`c23` 或其 `gnu` 版本）以及 `flags`，後者只接受 `-O*`、`-g`、`-W*`、`-f*`（不包括 `-flto`）與 `-m*`：其他任何東西都是錯誤，錯誤訊息會列出它們。目的檔放到 `target/<profile>/c/<name>/` 並加入連結（`--native-object`）。只有其輸入（它以及它所讀取的標頭檔）或選項改變的原始檔才會重新編譯（[9.13.8](913-c-compiler.md#9138-套件的-c-原始碼)）。
- `lints` 設定 lint 等級（lint 或群組名稱 → `"allow"` | `"warn"` | `"deny"`，[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）。它們被當作加在 `nabi` 所執行之每個編譯指令（委派給 `nabic` 的，或工具）的 CLI 旗標之前處理，因此 CLI 的 `-A/-W/-D` 與程式碼中的屬性會覆寫它們。
- `lints_from` 指定一個目錄（相對於這個資訊清單的目錄，或絕對路徑），其 `nabi.nabion` 提供起始的 lint 等級。該資訊清單的 `lints`（在其自己的 `lints_from` 鏈之後）先套用，接著是這個資訊清單的 `lints`，因此這裡的項目會覆寫鏈中相同的 lint 或群組（所有項目都依此順序以 `-A`/`-W`/`-D` 傳入；較後的旗標勝出）。它只適用於正在建置或檢查的套件（以及環境的資訊清單）；絕不會參考相依套件之資訊清單的 lint。若該目錄不存在、不含 `nabi.nabion`，或該鏈回到鏈中已有的資訊清單（循環），則是錯誤。`nabi env import` 會把相對的 `lints_from` 改寫為絕對路徑。腳本的 `//! ```nabion` 區塊不得使用它。
- `profile` 把建置設定檔（`"debug"` 或 `"release"`，[9.2.15](902-aot-pipeline.md#9215-輸出)）對應到其設定 `{ overflow_checks: bool }`（該紀錄可以命名為 `Profile`）。`overflow_checks: false` 在該設定檔中關閉整數溢位檢查，與 `-C overflow-checks=no` 相同（[9.1.2](901-nabic-cli.md#nabic-buildaot-建置)）；命令列上的 `-C overflow-checks` 會覆寫它。其他設定檔名稱或設定是錯誤。只會讀取正在建置的套件（或環境）；不讀取相依套件的設定檔，腳本的 `//! ```nabion` 區塊也不得使用它。
- 套件中繼資料：`authors` 是 `"Name <email>"` 字串，`nabi new` / `nabi init` 不會填入它們。`keywords` 與 `categories` 各自最多容納 5 個項目。`authors`、`keywords`、`categories`、`include` 或 `exclude` 中的空項目是錯誤。`readme` 與 `license_file` 是套件內、相對於資訊清單目錄的路徑（絕對路徑，或其 `..` 離開套件的路徑，是錯誤）；沒有 `readme` 時，若有 `README.md`，它就是 readme。`metadata` 是 `nabi` 不讀取的表（`{ .. }`）；它依寫出的樣子送到登錄庫。`nabi publish` 會檢查 readme 與 `license_file` 存在、`default_run` 指名套件的某個執行檔，並拒絕帶有 `publish: false` 的套件（9.10.4a）。
- `include` / `exclude` 是針對所發布之提交中檔案路徑的 glob（`*` 與 `?` 在一個名稱內，`**` 跨越名稱；不含 `/` 的模式比對任何深度的名稱，比對到目錄的模式也比對其中的內容）。當 `include` 為空或比對到某個檔案，且 `exclude` 沒有比對到它時，套件就容納該檔案；`nabi.nabion` 一律包含在內。它們決定 `nabi publish` 送出的打包大小（9.10.4a）；原始碼仍是整個提交。
- `links` 指名套件所連結的原生函式庫。同一個建置中兩個套件有相同的 `links` 是錯誤，錯誤訊息會指出兩者。
- `default_run` 指名 `nabi run` 所執行的執行檔；一個套件最多有一個執行檔（`src/main.nabi`，以套件命名），因此它就是那個名稱。
- `deployment_target` 是套件的程式能執行的最舊 macOS，以數字表示（`"13.0"`；其他形式是會顯示它的錯誤）；省略時為 `11.0`。紀錄形式 `{ macos: "13.0", ios: "16.0" }` 也給出最舊的 iOS（省略時為 `13.4`；更舊的是錯誤），每個欄位都是選用的。`nabi` 把它們以 `MACOSX_DEPLOYMENT_TARGET` 與 `IPHONEOS_DEPLOYMENT_TARGET` 傳給 `nabic`，除非其中之一已經設定，此時以已設定的為準。比它更新的 Objective-C API 只在 `objc::available` 之下呼叫（[std 1.10.13](../../refs-standard-libraries/100-bare/110-objc.md#11013-可用性與部署目標)）。
- `nabi` 不認識的欄位是帶有 `file:line:col` 的警告（`` unknown field `licence` is ignored; did you mean `license`? ``，有接近的欄位時附上建議），資訊清單在沒有它的情況下被讀取；`nabi publish` 會把它依寫出的樣子放在 `meta` 中送出（9.10.4a），讓較新的登錄庫可以使用它。
- 錯誤：未知的 `Dep` 種類 / 具名引數、重複、違反 9.10.4c 規則的 `features`、違反規則的套件名稱、不是 NABION 的語法、單一 `Package { .. }` 以外的任何東西、結尾多餘的語彙單元。診斷以 `file:line:col` 與插入符號顯示位置。

| 相依套件來源 | 形式                                                         | 在 `nabi.lock` 中釘選 |
|-------------------|--------------------------------------------------------------|----------------------|
| path        | `Dep::path(".."[, version: "req"])`（相對於資訊清單的路徑）  | 否（一律為目前的內容） |
| git         | `Dep::git("url")`，可選擇 `rev:` / `tag:` / `branch:` 其中之一 | 提交與 tree 雜湊     |
| registry    | `Dep::version("req"[, registry: "url"])` 或字串 `"req"`（[9.10.4a](#9104a-登錄庫套件)） | 版本、提交、tree 雜湊（連同索引，絕不是鏡像站） |
| native      | `Dep::native("req")` / `Dep::native(url:, sha256:)`（[9.10.4b](#9104b-原生套件)） | 版本與項目 SHA-256（索引）；無（由雜湊釘選） |
| system      | `Dep::system(..)` / `Dep::framework()`（[9.10.4b](#9104b-原生套件)） | 否（每次建置時尋找） |

Nabi 沒有建置腳本：套件的程式碼不會在建置時執行。C 函式庫以預先建置的原生套件、安裝在機器上的函式庫，或由 nabicc 編譯的 C 原始碼（`c`，[9.13.8](913-c-compiler.md#9138-套件的-c-原始碼)）的形式引入，全都在資訊清單中宣告（9.10.4b）。

在一個建置內，一個名稱就是一個套件。若同一個名稱從不同的來源被要求（不同的路徑，或不同的 url / 提交），就是錯誤，錯誤訊息會顯示兩個要求者。兩個相依者把同一個程式碼名稱給了不同的套件也是錯誤（`` `hello` is package `other` for the root package and package `greet` for `mid` ``）。一個套件可以有多個程式碼名稱（`package`）；`nabi` 以相同的路徑把每一個都作為 `--extern` 傳入。

### 工作區

一起開發的套件可以共用一個鎖定檔與一個建置目錄。**工作區**的根容納一個 `nabi.nabion`，其值是 `Workspace { .. }` 而不是套件：

```nabion
Workspace {
    members: ["app", "libs/*"],   // directories under the root; `dir/*`: each package in `dir`
}
```

- `members` 列出相對於根的目錄；`dir/*` 代表 `dir` 的每個含有 `nabi.nabion` 的直接子目錄。絕對路徑、`..` 步驟或其他的 `*` 都是錯誤（`` `../x` is not a member: write a directory under the workspace root, or `dir/*` for each package in `dir` ``），沒有 `nabi.nabion` 的成員目錄也是（`` member `gone` has no nabi.nabion (..); create the package with `nabi new gone`, or remove it from `members` ``）。每個成員都需要 `name`；兩個成員同名是錯誤。`Workspace` 不接受其他欄位。
- 當其目錄上方的工作區根列出某個套件時，它就是成員；位於根之下但未被列出的套件仍是獨立的。
- 各成員共用根部的 `nabi.lock`：它們的相依套件（包括開發用的）一起解析，因此每個成員都對照相同的版本建置，成員自己的 `nabi.lock` 不會被讀取。它們也共用根部的 `target/`（`NABI_TARGET_DIR` 與 `--target-dir` 仍然勝出）。成員之間以 `Dep::path` 相依。
- `build`、`check`、`test`、`bench`、`lint`、`fmt`、`doc`、`clean` 與 `run` 接受 `-p <name>` / `--package <name>`（可重複：這些成員）與 `--workspace`（每個成員）。在根部，由於那裡沒有套件，這些指令會在每個成員上執行；`run` 需要恰好一個（`` `nabi run` runs one package; name it: `nabi run -p <member>` (members: ..) ``）。每個成員都在自己的目錄中依 `members` 順序執行，有多個時，前面加上 `Member <name> (<dir>)`；失敗之後其他成員仍會執行，指令以第一個失敗的結束碼結束，並指出失敗的成員。不是成員的名稱是錯誤，錯誤訊息會列出各成員；在工作區之外使用 `-p` 是錯誤。
- 在根部執行的其他指令（`emit`、`add`……）是錯誤，錯誤訊息會說明這是工作區根，並提示 `-p <member>` 或某個成員的目錄；在那裡的 `nabi update` 與 `nabi deps` 作用於共用的鎖定檔。語言伺服器從共用的鎖定檔解析每個檔案的套件，包括成員。

### `nabi.lock` 格式

`nabi.lock` 是由 `nabi` 以確定性方式寫出的 NABION（標準形式）。在一段標頭註解之後，它依名稱順序為每個 git 或登錄庫套件容納一個 `Locked`。路徑相依套件不會被釘選，也不會出現。其欄位名稱遵循資訊清單的命名。

```nabion
// nabi.lock: generated by `nabi`; do not edit. Commit it with your sources.
Lock {
    version: 1,
    packages: [
        Locked {
            name: "parser",
            source: Dep::git("https://example.com/parser.git", tag: "v1.2"),
            commit: "4f0c2a9e1b7d3c58a6e09f21d4b8c7a3e5f60123",
            tree: "9a1e7c3b5d2f4a6081c9e3b7d5f2a4c6e8b0d1f3",
        },
        Locked {
            name: "json",
            version: "1.4.2",
            source: Dep::registry("https://github.com/nabi-lang/packages-index"),
            commit: "0b6d..",
            tree: "77e1..",
        },
    ],
}
```

| 欄位      | 意義                                                                 |
|-----------|----------------------------------------------------------------------|
| `version` | 在 `Lock` 中，是鎖定檔格式版本（`1`）；在 `Locked` 中，僅限登錄庫套件：解析出的版本 |
| `name`    | 套件的名稱（`deps` 的鍵，重新命名時則為其 `package`）                |
| `source`  | 對 git 而言，是與 `nabi.nabion` 中寫法完全相同的 `Dep::git(..)`；對登錄庫套件而言，是 `Dep::registry("<index url>")`：即索引，絕不是擷取它所用的鏡像站（9.10.4a） |
| `commit`  | 解析出的提交（40 個十六進位數字）                                    |
| `tree`    | 該提交的 git tree 雜湊。用於驗證快取中的 checkout                    |

- `Locked` 的 `source` 是 `Dep::git(..)` 或 `Dep::registry(..)`。
- 缺少 `packages` 欄位時讀取為空清單。
- 不記錄 feature：鎖定檔釘選圖中的每個套件，包括選用的，因此它不取決於建置所啟用的 feature。
- 來自索引的原生套件釘選在選用的 `natives: [LockedNative { name, version, source: Dep::registry("<index>"), sha256 }]` 中，依名稱排序，只在非空時寫出。`sha256` 是該項目以 `yanked: false` 表示之標準索引行的 SHA-256；改變的項目是「run `nabi update`」錯誤。
- `nabi` 寫出鎖定檔格式 1。
- `version` 在其他欄位之前讀取。大於 `1` 的 `version` 是由較新的 `nabi` 寫出的鎖定檔，這是一個錯誤，錯誤訊息會指出應改用的工具鏈，而不是刪除它：``nabi.lock was written by a newer nabi (lock version 2); run that nabi's toolchain here: set `toolchain` in nabi.nabion to its version, or run `nabi +<version> ..` ``。
- `1` 以外的其他任何 `version` 是「unsupported lockfile version」錯誤。
- 讀取鎖定檔時引發的其他每個錯誤都會附加「(delete nabi.lock and run `nabi update` to make it again)」。

## 9.10.4. 相依套件：`nabi add` / `remove` / `update` / `deps`

```
nabi add <name> --path <dir> [--dev]
nabi add <name> --git <url> [--rev <rev> | --tag <tag> | --branch <branch>] [--dev]
nabi add <name>[@<req>] [--registry <url>] [--dev]      # from the index
nabi add <name> .. [--features <f,..>] [--no-default-features] [--optional]   # with any of the above
nabi remove <name>
nabi add .. <file.nabi>                    # into a script's //! nabion block
nabi remove [--native] <name> <file.nabi>
nabi update [name...]
nabi deps [--dev] [--features <f,..>] [--all-features] [--no-default-features] [--format json]
```

- `add` / `remove` 編輯套件的 `nabi.nabion`，並保留註解與版面。`--dev` 新增到 `dev_deps`。以腳本檔案作為最後一個引數時，`add` / `remove` 改為編輯該檔案的 `//! nabion` 區塊（9.10.7）：缺少的區塊會建立在 shebang 行與開頭的 `//!` 行之後（以標準形式寫出；既有的區塊保留其版面與註解），變成空的區塊會被移除，路徑寫成相對於腳本，而 `--dev` / `--optional` 是錯誤。位於套件內的腳本是錯誤（它使用套件的 `nabi.nabion`）。在套件之外又沒有腳本時，或搭配 `--env` 時，它們是錯誤，錯誤訊息會提示 `nabi add <name> <file.nabi>` 以及 `nabi install` / `nabi uninstall`（[9.10.5a](#9105a-nabi-install--nabi-uninstall)）。新增不帶需求的套件會寫入 `^<newest version>`。`--features`、`--no-default-features` 與 `--optional` 會寫進 `Dep::..`；`--optional` 還會新增 feature `"<name>": ["dep:<name>"]`，除非套件已有同名的 feature。
- 對套件執行 `add` 會回報 `Added <name> v<version>`，即相依套件之 `nabi.nabion` 中的 `version`（對尚未解析的登錄庫套件，是被釘選的版本），沒有版本時則回報 `Added <name>`；`--dev` 會加上 ` (dev_deps)`。
- 建置使用 `nabi.lock` 中的提交與版本。新宣告的相依套件會加入鎖定檔，被移除的會被清理，但只有 `update` 會移動已經釘選的東西。給出名稱時，`update` 只重新解析那些套件；否則它重新解析所有 git 與登錄庫相依套件。
- `deps` 以 `<code name>=<path of its src/lib.nabi>` 行印出已解析的相依套件（程式碼名稱中的 `-` 換成 `_`），相依套件在前（`--dev` 包括開發用相依套件）。相依者以其他名稱（`package`）稱呼的套件，每個名稱多一行這樣的內容。每個套件的行之後接著其 `--extern-feature=<code name>=<f>` 行；`deps` 接受 feature 選項與 `--dev`。對每個原生套件（相依者在前，針對主機），它還會印出 `# native <name> <version>`，其後依每行一個列出其 `nabic` 引數。沒有相依套件時，它不印出任何東西。
- `deps --format json` 則依相同順序每行印出一個 JSON 物件，每個都帶有 `schema`（目前是 `1`）與 `type`。`package` 紀錄有 `name`、`version`（登錄庫套件解析出的版本，否則是相依套件之 `nabi.nabion` 的 `version`，否則為 `null`）、`source`（`path+<dir>`、`git+<url>#<commit>` 或 `registry+<index url>`）、`path`（其 `src/lib.nabi`）以及 `features`。`native` 紀錄有 `name`、`version` 與 `args`（其 `nabic` 引數）。最後一筆 `summary` 紀錄計算 `packages` 與 `natives` 的數量，因此沒有相依套件的套件只會印出它。紀錄遵循 JSON 診斷的 `schema` 規則（[9.9.4](909-debug-tools.md#994-診斷--錯誤訊息)）。`--format human` 是預設值；其他值是 CLI 錯誤。
- `nabi.lock` 要提交到版本控制中。
- `update` 為鎖定檔中的每項改變印出一行 `Updated <name> v0.1.0 -> v0.1.1`（git 相依套件以其短提交表示；新釘選的以 `Locked` 表示，被捨棄的以 `Removed` 表示），或 `note: all dependencies are up to date`；沒有相依套件時為 `` note: no dependencies to update; add one: `nabi add <name>` ``。在套件之外且沒有 `--env` 時，若在套件之外使用的環境（9.10.5）有函式庫，它會更新該環境（有改變時以 `` Updated environment `<name>` `` 結尾）；否則是錯誤：`` not inside a package (..): make one with `nabi init`; for an environment's libraries: `nabi update --env <name>` ``。
- `nabi update` 重新解析 `nabi.lock`；`nabi toolchain update` 取代工具鏈（9.10.6）。

### 快取

擷取的 git 原始碼保存在 `$NABI_HOME/cache/` 中，由所有專案共用。

| 路徑                                       | 內容                                                    |
|--------------------------------------------|---------------------------------------------------------|
| `cache/git/db/<repo>-<hash>/`              | 儲存庫的 bare clone（每個 url 一個）                    |
| `cache/registry/db/<repo>-<hash>/`         | 登錄庫索引的 bare clone（每個索引一個）                 |
| `cache/registry/src/<sha256>/`             | 來自登錄庫之該 SHA-256 封存檔的登錄庫套件檔案（9.10.4a）；絕不會再次擷取 |
| `cache/git/checkouts/<commit>/`            | 某個提交的 checkout。使用前以 tree 雜湊驗證，不同時會重新解開 |
| `cache/scripts/<hash>/`                    | 腳本的 `nabi.lock` 與 `target/`；`<hash>` 涵蓋腳本的路徑，以及它所到達之每個檔案的路徑與區塊（[9.10.7](#9107-單一檔案腳本的行內資訊清單)） |
| `cache/native/<sha256>/`                   | 原生封存檔，在其 SHA-256 相符之後解開；絕不會再次擷取（9.10.4b） |
| `cache/native/meta/<sha256>.nabion`         | `Dep::native(url:)` 檔案                                |

## 9.10.4a. 登錄庫套件

發布到登錄庫的套件是**登錄庫套件**。預設登錄庫是 `https://packages.nabi-lang.org`，其索引是另一個 git 儲存庫 `https://github.com/nabi-lang/packages-index`。

### 版本需求

與 Cargo 相同。

| 需求                     | 意義                                            |
|--------------------------|-------------------------------------------------|
| `1.2.3`、`^1.2.3`        | `>=1.2.3, <2.0.0`（`^0.2.3` 是 `<0.3.0`，`^0.0.3` 是 `<0.0.4`） |
| `~1.2.3`                 | `>=1.2.3, <1.3.0`                               |
| `=1.2.3`                 | 完全相同                                        |
| `>=` `>` `<` `<=`        | 比較                                            |
| `*`、`1.*`、`1.2.x`      | 萬用字元                                        |
| `>=1.2, <1.5`            | 逗號表示全部都必須滿足                          |

預發行版本（`1.3.0-beta.1`）只在需求指名了相同 major.minor.patch 的預發行版本時才會相符。

### 哪一個索引

不帶 `registry:` 的需求（單純的需求）一律表示預設登錄庫的索引 `https://github.com/nabi-lang/packages-index`。`Dep::version` 的 `registry:` 與 `nabi add --registry` 為該單一相依套件設定索引；`nabi publish`、`yank`、`login` 與 `logout` 接受 `--registry <url>` 以使用其他索引。

`$NABI_HOME/sources.nabion` 取代擷取索引的來源（來源替換）：

```nabion
Sources { replace: { "https://github.com/nabi-lang/packages-index": "https://mirror.example.com/packages-index" } }
```

- 每個鍵是索引 url（忽略結尾的 `/`），其值是內容相同之鏡像站的 git url。`nabi` 從鏡像站擷取索引；其他一切，包括鎖定檔（9.10.3）、權杖與訊息，都指名該索引。缺少該檔案表示沒有替換；`nabi` 不認識的欄位會被略過。

### 索引格式

索引儲存庫的根部有一個 `config.nabion`，即 `Config { api: "<url>" }`：登錄庫 API 的基底 URL（預設登錄庫為 `https://packages.nabi-lang.org`）。它必須是 `https://`，`http://127.0.0.1` 與 `http://localhost` 除外。一個套件是位於其發布時小寫名稱（`-` 保持原樣）之分片路徑上的一個檔案。

| 名稱長度    | 路徑                     |
|-------------|--------------------------|
| 1           | `1/a`                    |
| 2           | `2/ab`                   |
| 3           | `3/a/abc`                |
| 4 以上      | `ab/cd/abcd..`           |

檔案的每一行是一個版本，以標準形式的單行 NABION 表示。較舊的版本在前。一行只容納解析所需的東西，使用資訊清單的欄位名稱；描述性欄位送到登錄庫 API（`meta`，見下文）。

```nabion
Package { name: "json", version: "1.4.2", deps: [{ name: "util", req: "^0.3", kind: "normal" }], source: Dep::git("https://example.com/json.git", rev: "0b6d.."), tree: "77e1..", yanked: false, nabi_version: "0.1.0" }
```

| 欄位        | 意義 |
|-------------|---------|
| `name`      | 套件發布時的名稱 |
| `version`   | 其版本 |
| `deps`      | 相依套件：`{ name, req, kind, registry?, optional?, features?, default_features?, package?, target? }` |
| `features`  | 選用：feature（9.10.4c） |
| `source`    | `Dep::git("<url>", rev: "<commit>")` |
| `tree`      | 該提交的 git tree 雜湊 |
| `archive`   | 選用，由登錄庫寫入：它為該提交保存之原始碼封存檔的 SHA-256（64 個小寫十六進位數字） |
| `yanked`    | 是否已撤回 |
| `nabi_version` | 能建置它的最舊工具鏈，`x.y.z`（9.10.3） |
| `links`     | 選用：它所連結的原生函式庫（9.10.3） |
| `official`  | 選用，由登錄庫寫入：對 Nabi 專案所發布的套件為 `true`（見下文） |

- `deps` 中的 `kind` 是 `"normal"`、`"dev"` 或 `"native"`；`registry` 是選用的。被重新命名的相依套件以 `name` 表示程式碼名稱，以 `package` 表示套件的名稱；限定目標的相依套件帶有與資訊清單相同的 `target`，由用戶端在每次建置時求值（9.10.3）。
- 帶有 feature 的套件在 `deps` 之後有 `features: { "<name>": ["<entry>", ..] }`，依 9.10.4c 檢查。`normal` 相依套件可以有 `optional: true`、`features: [..]` 與 `default_features: false`（只在不是預設值時寫入）；`dev` / `native` 相依套件不得是選用的，原生相依套件沒有 feature。`nabi publish` 從 `nabi.nabion` 寫入它們；解析從項目中讀取它們。checkout 之後，項目的 `features` 必須等於該提交時套件之 `nabi.nabion` 的 `features`（順序不拘）；否則是錯誤。
- `native/<shard path>` 容納原生套件，每行一個 `Native { .. }`（[9.10.4b](#9104b-原生套件)）。
- `nabi_version` 就是資訊清單的 `nabi_version`（9.10.3）。`nabi publish` 需要它：沒有它時的錯誤是 `` nabi.nabion has no `nabi_version`, the oldest nabi this package builds with; the registry needs it ``，說明為 `` add `nabi_version: "0.1.3"` (this nabi), or the oldest one you have built it with ``，而不接受進行發布之 `nabi` 的值是錯誤，錯誤訊息會指出它所接受的工具鏈。登錄庫會拒絕沒有它，或其值不是 `x.y.z` 的行。讀取者會遵守它（解析，見下文）；在自己的索引中，沒有它的行適用於任何工具鏈。
- 標準函式庫層級的檔案（`3/s/std`、`co/re/core`、`ba/re/bare`）容納一行 `Library { name: "std", official: true }`：登錄庫中對隨工具鏈提供、由 Nabi 專案擁有、沒有版本也沒有封存檔之函式庫的項目。進行解析的讀取者會略過它，也無法以其名稱發布任何版本。
- 這是唯一的行格式：每一行沒有格式版本。
- 讀取者會略過索引行中它不認識的欄位（`Package`、其 `deps` 項目、`Native`、`Artifact`）。自己的 `Native { .. }` 檔案（`Dep::native(url:)`、`nabi publish --native`）仍會拒絕它們。登錄庫會嚴格檢查各行。
- 套件位於 `source` 儲存庫的根部。`tree` 是該提交的 git tree 雜湊，在 checkout 之後驗證。
- 版本發布時，登錄庫保存該提交的 gzip 壓縮 tar（`git archive`，每個路徑都位於 `<name>-<version>/` 之下），在 `<api>/api/v1/packages/<name>/<version>/download` 提供它，並在 `tree` 之後把其 SHA-256 寫成 `archive`；`nabi publish` 絕不送出 `archive`。不是 64 個小寫十六進位數字的值被讀取為不存在。
- 登錄庫套件依序來自：對預設索引而言，工具鏈自己的複本（官方套件，見下文）；快取中已有的其提交的 git checkout；其封存檔（有給出 `archive` 時），來自 `cache/registry/src/<sha256>/`，或從索引之 `config.nabion` 的 `api` 下載（`Downloading <name> <version>`）；其 git `source`。SHA-256 不是 `archive` 的下載是錯誤（`` package `num` 1.2.0: <url> has SHA-256 .., but the index says ..; the archive or the index is damaged .. ``），建置停止。取得不到封存檔（沒有保存、無法連線、沒有 `api`，或 `--offline` 而快取中沒有）是一則附註，改為擷取 git 原始碼。有封存檔時，既不需要 git，也不需要套件的儲存庫；無論哪種方式，鎖定檔都釘選相同的提交與 tree。

### 解析

- 對一個套件的每項需求都算數：所選的版本滿足所有需求。套件依其第一次被要求的順序挑選，每個都先嘗試鎖定檔中釘選的版本（只要它滿足每項需求就保留，即使之後被撤回），然後由高到低嘗試未撤回的版本。當某個選擇讓某個套件沒有滿足其需求的版本時，搜尋會回溯，並嘗試在它之前挑選之套件的下一個版本，因此一個相依者的 `>=0.1` 與另一個相依者的 `^0.1` 會解析為最高的 `0.1.x`。
- 新的選擇（以及不帶需求的 `nabi add`）會略過其 `nabi_version` 不接受正在執行之 `nabi`（[9.10.6 相容性](#相容性)），或 `nabi` 無法讀取的項目。當這樣隱藏了最高的相符版本時，一則附註會指出該版本、它所需要的工具鏈（`` package `num` 2.0.0 needs nabi 0.2.1 or a later 0.2.x (this is nabi 0.1.5) ``）、如何使用它（`nabi +0.2.1 ..`，或釘選該工具鏈）以及改為選擇的版本；當沒有其他版本相符時，錯誤會把該附註作為說明。鎖定檔中釘選的版本無論如何都會保留。
- 對預設索引，工具鏈所攜帶的版本（官方套件，見下文）在鎖定檔中釘選的版本之後、其他版本之前嘗試：`^0.2` 會挑選攜帶的 `0.2.0` 而不是索引中的 `0.2.5`。它們的行不需要索引，因此當搜尋中的每個套件都有攜帶且沒有快取的索引時，不會擷取任何索引；當攜帶的版本讓某項需求無法滿足時，會擷取索引並重新進行搜尋。`nabi update` 由高到低嘗試各版本，不論是否有攜帶。
- 在一個相依圖中，一個名稱就是一個版本。間接相依套件遵循索引項目的 `deps`。路徑與 git 套件是固定的：它們的登錄庫需求與根的需求一樣算數。
- 當沒有任何一組版本可行時，錯誤會指出一個沒有任何版本能滿足其需求的套件、每項需求及其要求者（`` `^2` (required by `a` 1.4.0) ``）、索引中有的版本、每個提出要求的套件如何進入建置（`` `a` 1.4.0: the root package requires `a` `^1.0` ``），以及是否嘗試過它們的其他版本。其說明會指出該寫什麼：滿足其他需求的該套件版本（`` change the root package's to it, `"b": "^2.3.0"` ``），或與它們相容的提出要求之套件的版本（`` change the root package's requirement on it, `"c": "^0.4.0"` ``），否則放寬其中一項需求。沒有任何版本能滿足的單一需求是單純的「no version of package .. matches ..」錯誤。搜尋在 100000 次選擇之後放棄，並建議釘選主要的相依套件。
- 鎖定檔會對照索引檢查。若某個被釘選之版本的 `rev` 已改變，就是「run `nabi update`」錯誤。
- `--offline` 使用快取的索引，並遵守 `--locked`。其原始碼兩個快取中都沒有的版本（沒有 git checkout，也沒有封存檔）是錯誤（`` package `json` 0.2.5 is not in the cache, and --offline is set: run the command once without --offline; `nabi component add packages` (or `update`) keeps the official packages for offline use ``）；後半段只針對官方套件。

### 官方套件

Nabi 專案所發布的套件在索引中標示為 `official: true`（由登錄庫寫入）。程式碼可以不先新增就使用它：當對套件或腳本執行 `build`、`run`、`check`、`test`、`bench`、`repl` 或 `jupyter` 時，發現某個作為路徑根使用的名稱（`use json::parse;`、`json::parse(s)`）不是該套件或腳本的相依套件（建置中另一個套件所相依的不算），程式碼本身也沒有宣告它（模組、項目、匯入、繫結），而預設索引表示該名稱是官方的（依其最新版本的行，即未撤回的最高版本；查找名稱時 `_` 與 `-` 視為相同，因此 `nabi_syntax::..` 就是 `nabi-syntax`，以其索引名稱新增），`nabi` 會：

- 挑選工具鏈所攜帶的版本（見下文），否則挑選這個工具鏈能建置的最高未撤回版本（其 `nabi_version` 接受這個工具鏈），並把 `"<name>": "^<major>.<minor>"`（低於 0.1 時為 `^<version>`）寫進套件的 `nabi.nabion`（工作區成員自己的；腳本的 `//! nabion` 區塊，缺少時會建立），解析並更新 `nabi.lock`，印出 `Added <name> ^x.y (official)`，然後繼續執行該指令；
- 對工具鏈所攜帶的套件，不讀取任何索引；否則讀取快取的索引，只有在沒有快取時才擷取它（擷取失敗，或 `--offline` 而沒有快取時，不會新增任何東西）；
- 有 `--locked` 或 `--frozen` 時（在錯誤中以其名稱指出），或 `CI` 設為非空值時，絕不編輯資訊清單：`` `json` is an official package the code uses, but it is not in nabi.nabion, and --locked never changes it; add it: `nabi add json` ``；
- 離線且兩個快取中都沒有該套件時：`` .. it is not in the cache and --offline is set: run `nabi add json` online, or keep the official packages for offline use with `nabi component add packages` ``。

根 `std`、`core`、`bare`、`nabi`、`pkg`、`sup`、`crate`、`super`、`self` 以及首字母大寫的名稱絕不是套件。不是官方的名稱保留編譯器的錯誤（`` `geom` is not a dependency of this package: add it to nabi.nabion (`nabi add geom`) ``）。REPL 與 Jupyter 核心對每個儲存格做相同的事（在套件的工作階段中是 `nabi add`，在環境的工作階段中是 `nabi install --env`）。語言伺服器絕不編輯資訊清單：它提供 `nabi add <name>` 快速修正（9.9.13）。

### `nabi component`

```
nabi component list
nabi component add packages [--bundle <dir>]
nabi component update [packages]
nabi component remove packages
```

`packages` 是預設索引中每個官方套件、這個工具鏈能建置（其 `nabi_version` 接受這個工具鏈）之最高未撤回版本的登錄庫封存檔（9.10.4a 索引格式）。

- 發行版工具鏈會攜帶它（發行，9.10.6）：`bin/` 旁邊的 `lib/nabi/packages/` 在 `index` 中容納那些版本的索引行（每行一個，與索引檔案相同），並在 `src/<sha256>/` 中容納解開後的每個封存檔。除了 `remove` 之外，沒有任何東西會寫入那裡；解析對預設索引先讀取它（解析，見上文），而其各行指名的提交與 tree 與索引相同，因此鎖定檔與下載所產生的相同。
- `add` 擷取索引，並把每個封存檔下載到 `cache/registry/src/<sha256>/`，對照該行的 `archive` 檢查（`Downloading json 0.2.0` ..，然後是 `Added component packages (<n> official packages)`），並把它們記錄在 `$NABI_HOME/components.nabion` 中；元件已新增或由工具鏈攜帶時，再次 `add` 是一則提示 `update` 的附註。`--bundle <dir>` 則把它們作為工具鏈的 `lib/nabi/packages` 寫到 `<dir>`（取代它），與發行版建置的做法相同（`Bundled <n> official packages into <dir>`）。
- `update` 重新擷取索引，並把這個工具鏈能建置的最新版本下載到快取中；工具鏈所攜帶的東西保持原樣。
- `remove` 刪除工具鏈所攜帶的東西、記錄的封存檔以及該紀錄；兩者都沒有時，是錯誤。
- `list` 顯示每個元件以及它是否已新增：`added: <n> package(s) with the toolchain`，當 `add` / `update` 也下載了一些時，加上 `, <m> downloaded`。

沒有 `archive` 的版本是一則附註，在使用時從 git 擷取。對 `add` 與 `update` 使用 `--offline` 是錯誤；其他名稱是錯誤，錯誤訊息會列出各元件。

元件只會填入快取：版本仍由資訊清單與鎖定檔挑選。釘選了工具鏈與快取都沒有之版本的鎖定檔，在線上時會下載它，離線時則是上述錯誤。

### `nabi search` / `publish` / `yank` / `login` / `logout`

```
nabi search [--native] <query>
nabi login [--registry <url>] [<token>]
nabi logout [--registry <url> | --all]
nabi publish [--dry-run] [--registry <url> | --index-dir <checkout>]
nabi yank [--native] <name>@<version> [--undo] [--message <why>] [--registry <url> | --index-dir <checkout>]
```

- `search` 依名稱搜尋索引，並為每個相符項目印出一行：`<name> <newest version> - <description>`。描述來自登錄庫 API（`GET <api>/api/v1/packages/<name>`，最新版本的 `meta.description`）；API 沒有描述或沒有回應時，以及在 `--offline` 下，會省略它。索引查找接受 `-` 或 `_` 任一種拼法。
- `publish` 進行發布（與 `cargo publish` 相同）；`publish --dry-run` 是試執行。帶有 `publish: false` 的套件會被拒絕，錯誤訊息會指出要移除的那一行。readme（9.10.3）必須存在。它要求乾淨的工作樹、位於儲存庫根部的套件，以及 `src/lib.nabi`、`src/main.nabi` 或兩者。只有 `src/main.nabi` 的套件是程式：`nabi install` 會建置它，它不能作為相依套件。一般的路徑相依套件以其 `version:` 需求發布，沒有時會被拒絕（`` dependency `linalg` is a path: .. add the version it needs, published from that path: `Dep::path("../linalg", version: "^0.1")` ``）；一般的 git 相依套件會被拒絕，沒有 `version:` 的路徑 / git 開發用相依套件會被捨棄。它會檢查與相同版本的衝突，以及與在 `-` / `_` 與大小寫折疊下相同之名稱的衝突。已以不同拼法被擁有的名稱會被拒絕。沒有 `--index-dir` 的試執行只會印出項目；有 `--index-dir` 時也會對照該 checkout 檢查。項目的來源是所設定之 git remote `origin` 的 url（在 `insteadOf` 改寫之前）；它必須是 `https://`，`http://127.0.0.1` 與 `http://localhost` 除外。
- 沒有 `--dry-run` 時，`publish` 新增項目：有 `--index-dir` 時，作為該 checkout 中的一個提交「Publish <name> <version>」（不推送）；沒有時，經由 `--registry <url>` 所指名之索引的登錄庫 API，否則經由預設登錄庫的（「哪一個索引」），由它把項目加入索引。`--index-dir` 與 `--registry` 不能同時使用。用戶端絕不會自己寫入索引儲存庫。
- `yank` 為該版本設定（或以 `--undo` 清除）`yanked`：有 `--index-dir` 時，作為該 checkout 中的一個提交；否則經由 `--registry <url>` 所指名之索引的登錄庫 API，否則經由預設登錄庫的。
- `--commit`（`publish`）、`--index`（`publish` 與 `yank`）與 `--vers`（`yank`）是錯誤，錯誤訊息會指出應寫的形式：單純的 `publish`、`--index-dir <dir>` 與 `yank <name>@<version>`。
- `yank --message <why>` 說明原因（最多 512 位元組）；登錄庫保存並顯示它。它只能搭配撤回：搭配 `--undo`，或搭配 `--index-dir`（checkout 沒有可以放它的地方），是錯誤。
- 登錄庫 API 位於索引之 `config.nabion` 的 `api`。請求帶有 `Authorization: Bearer <token>`，主體是單行 NABION（`application/nabion; charset=utf-8`）：
  - `GET <api>/api/v1/me` → `Me { login: "..", publish: <bool> }`
  - `POST <api>/api/v1/publish`，主體 `Publish { entry: <index line>, meta: { .. } }`（該行是 `Package { .. }` / `Native { .. }`）→ `Published { name, version, file, warnings: ["..", ..] }`；`nabi` 印出每則警告。
  - `POST <api>/api/v1/yank`，主體 `Yank { name, version, undo: <bool>, native: <bool>, message: ".." }`（`message` 只在有時寫入）→ `Yanked { name, version, yanked }`
  - `GET <api>/api/v1/packages/<name>` 與 `/api/v1/native/<name>`，不需權杖 → `Package { name, native, owners, versions: [Version { version, yanked, meta, yank_message, has_readme, downloads, published_by, yanked_by, unyanked_by }, ..] }`；`GET <api>/api/v1/packages/<name>/<version>/readme` → 以 markdown 表示的 README。

  失敗是帶有 `Error { message: ".." }` 的 4xx / 5xx 狀態，`nabi` 會回報其訊息；401 表示權杖無效。`nabi` 會略過它不認識的回覆欄位。
- `meta` 容納描述套件、不參與解析的內容，來自資訊清單（9.10.3）與套件。沒有值的欄位會被省略。

  | 欄位 | 來源 |
  |-------|------|
  | `description`、`license`、`license_file`、`authors`、`homepage`、`documentation`、`repository`、`readme`（路徑）、`keywords`、`categories`、`include`、`exclude`、`publish`、`metadata`、`links`、`default_run` | 資訊清單中同名的欄位 |
  | `has_lib` | 套件是否有 `src/lib.nabi` |
  | `bin_names` | 其執行檔：有 `src/main.nabi` 時為 `[<name>]`，否則為 `[]` |
  | `readme_content` | readme 的文字（9.10.3），最多 524288 位元組；更長的在 `nabi publish` 時是錯誤 |
  | `size` | 打包大小：套件所容納之 HEAD 檔案的位元組數（`include` / `exclude`，9.10.3） |

  `nabi` 不認識的資訊清單欄位會依寫出的樣子放在 `meta` 中送出，讓較新的登錄庫可以儲存它們。對 `publish --native`，`meta` 是該檔案的 `description`、`license` 與 `repository`。
- 登錄庫 API 需要權杖；`--index-dir` checkout 不需要。預設登錄庫的權杖是有設定且非空時的 `NABI_REGISTRY_TOKEN`，否則是 `nabi login` 所儲存的權杖。對其他索引，是 `nabi login --registry <url>` 所儲存的權杖；`NABI_REGISTRY_TOKEN` 絕不會送到那裡（有設定時，會警告一次：``NABI_REGISTRY_TOKEN goes only to https://github.com/nabi-lang/packages-index; for <url> run `nabi login --registry <url>` ``）。沒有權杖時，是錯誤「run `nabi login` first」（對其他索引是 ``run `nabi login --registry <url>` first``）。
- `nabi login` 接受一個登錄庫帳號權杖，在網站上建立（`<api>/me`，以 GitHub 登入；目前只有登錄庫的維護者能建立）。省略權杖時，會從標準輸入讀取，不回顯（按 Ctrl-C 時會恢復回顯）。`login` 會詢問 `/api/v1/me`：它拒絕 API 不接受的權杖，以及其 `publish` 為 false 的帳號（「this account cannot publish yet: publishing is open to the registry's maintainers only」），不儲存任何東西。否則它把該索引（`--registry <url>`，否則為預設登錄庫）的權杖以 `Tokens { tokens: { "<index url>": "<token>" } }` 儲存在 `$NABI_HOME/tokens.nabion` 中（鍵不帶結尾的 `/`；已有的欄位與其他索引會保留），並印出「Logged in as <login>」。權杖只會送到它自己之索引的登錄庫 API。
- `tokens.nabion` 是唯一的權杖檔案。
- 在 Unix 上，`tokens.nabion` 以模式 0600 寫入。在 Windows 上，它取得位於使用者設定檔之下的 `$NABI_HOME` 的預設權限。
- `nabi logout` 從 `tokens.nabion` 移除預設登錄庫的權杖；`--registry <url>` 移除該索引的項目；`--all` 移除每個權杖。
- 權杖只經由 `curl` 的標準輸入交給它：絕不出現在命令列或環境中，也絕不被印出（錯誤顯示 `***`）。

## 9.10.4b. 原生套件

原生套件是每個目標三元組一個的預先建置封存檔，加上一個值：

```nabion
Native {
    name: "png", version: "1.6.43",
    deps: { "zlib": Dep::native("^1.3") },
    targets: {
        "x86_64-unknown-linux-gnu": Artifact {
            url: "https://example.com/png-1.6.43-x86_64-linux.tar.gz",
            sha256: "<64 hex>",
            links: { "png16": "static" },
            runtime: [],
        },
    },
    yanked: false,
}
```

該值是一行索引行，或是供 `Dep::native(url:, sha256:)` 使用的檔案，其產出物 url 可以相對於該檔案。`nabi publish --native` 所讀取的檔案也可以有 `description`、`license` 與 `repository`；它們放在 `meta` 中送到登錄庫 API，不會放進索引行。

- 封存檔（任何 tar 可讀取的格式）容納 `lib/`：`lib<n>.a`、`lib<n>.dylib` / `lib<n>.so`、Windows 的 `<n>.lib`（靜態或匯入函式庫）。它也容納 `runtime` 檔案與授權檔案。其中的任何東西都不會被執行。
- 雜湊是必要的。封存檔在解開之前會被檢查，若含有絕對路徑或 `..` 路徑、離開它的連結，或缺少所指名的函式庫或 runtime 檔案，就會被拒絕。
- 在 Apple 目標上，runtime dylib 必須有 `@rpath/` 的 install name。
- 建置使用目標三元組的產出物；缺少時是錯誤，錯誤訊息會列出它所擁有的三元組。
- `nabi` 傳給 `nabic` `-L <archive>/lib`、每個連結一個 `-l kind=name`，以及每個 runtime 檔案一個 `--native-runtime`，相依者在相依套件之前。

```
nabi add --native <name>[@<req>]
nabi remove --native <name>
nabi search --native <query>
nabi publish --native <file> [--dry-run] [--registry <url> | --index-dir <dir>]
nabi yank --native <name>@<version> [--undo] [--registry <url> | --index-dir <dir>]
```

`publish --native` 會擷取並檢查每個目標，並要求絕對 url、授權檔案，以及只來自索引的相依套件。發布它以及 `yank --native` 遵循登錄庫套件的規則：`--index-dir` checkout，否則是帶有權杖的登錄庫 API（`Yank` 中的 `native: true`）。

- 在 ELF 目標上，runtime 函式庫的 soname 必須等於其檔案名稱。對 `-musl` 目標的 `dylib` 連結是錯誤。
- `repl` / `jupyter` 會載入原生套件：共用函式庫經由系統載入器，靜態封存檔則連結進工作階段。`check` / `lint` / `doc` 不連結任何東西。
- 執行檔旁邊的複本是建置輸出。

### 系統函式庫與 framework

```nabion
native: {
    "z": Dep::system(pkg_config: "zlib", brew: "zlib", vcpkg: "zlib", apt: "zlib1g-dev"),
    "png": Dep::system(pkg_config: "libpng16", libs: ["png16"], static: true),
    "Metal": Dep::framework(),
}
```

`Dep::system` 指名安裝在機器上的 C 函式庫。其所有引數都是選用的具名引數：`pkg_config`（`.pc` 模組；預設為鍵）、`brew`（Homebrew formula）、`vcpkg`（vcpkg port）、`apt`（錯誤訊息告訴 Linux 使用者要安裝的套件）、`libs`（沒有 `.pc` 檔案說明時要連結的東西；預設為鍵），以及 `static: true`（連結靜態封存檔，連同 `.pc` 檔案的 private 部分）。每次建置都依下列順序尋找它，不執行任何程式：

1. pkg-config `.pc` 檔案，在程式內部讀取：`PKG_CONFIG_PATH`，然後是有設定時的 `PKG_CONFIG_LIBDIR`，否則為目標的 multiarch 目錄（`/usr/lib/x86_64-linux-gnu/pkgconfig`……）、`/usr/local/lib/pkgconfig`、`/usr/local/share/pkgconfig`、`/usr/lib64/pkgconfig`、`/usr/lib/pkgconfig`、`/usr/share/pkgconfig`，在 macOS 上還有 Homebrew 的 `lib/pkgconfig`。會追蹤變數（`${prefix}`、`${pcfiledir}`）與 `Requires`（有 `static` 時還有 `Requires.private` 與 `Libs.private`）；`PKG_CONFIG_SYSROOT_DIR` 會加在它們所給出的路徑之前。
2. Homebrew，在 macOS 上（或設定了 `HOMEBREW_PREFIX` 時）：`$HOMEBREW_PREFIX/opt/<formula>`（預設 `/opt/homebrew`），其自己的 `.pc` 檔案，否則為其 `lib/` 與 `include/`。
3. vcpkg，設定了 `VCPKG_ROOT`（或 `VCPKG_INSTALLATION_ROOT`）時：`installed/<triplet>`，triplet 取自目標（`x64-windows`、`arm64-windows`、`static` 時為 `x64-windows-static`、`x64-linux`、`arm64-osx`……），除非 `VCPKG_DEFAULT_TRIPLET` 指名了一個；在 Windows 上，其 `bin/` 中的 DLL 會放到執行檔旁邊。
4. macOS SDK 自己的函式庫（`usr/lib/lib<name>.tbd`：`z`、`sqlite3`……）。

交叉建置只在目標的檔案所在之處尋找：`PKG_CONFIG_PATH`、`PKG_CONFIG_LIBDIR`、目標的 multiarch 目錄，以及有 `PKG_CONFIG_SYSROOT_DIR` 時該 sysroot 的預設目錄；絕不會找主機的 Homebrew。`nabi` 把它找到的東西以 `-L` 與 `-l` 傳入；標頭檔目錄交給 `c` 原始碼。找不到時是錯誤，錯誤訊息會指出它找過的地方以及主機的安裝指令：`` system library `png` (pkg-config `libpng16`) not found for x86_64-unknown-linux-gnu (looked in pkg-config (libpng16.pc)): install it with `sudo apt install libpng-dev` (or your distribution's package of it), or set PKG_CONFIG_PATH to the directory holding libpng16.pc ``（在 macOS 上為 `brew install`，在 Windows 上為 `vcpkg install <port>:<triplet>`）。

`Dep::framework()` 不接受任何引數：鍵就是 framework 的名稱（`Foundation`，保持 Apple 的拼法），以 `#[link(name = "Foundation", kind = "framework")]` 的方式連結（[5.9.9](../005-modules/509-extern-block.md#599-linklibname指名函式庫)）。對 Apple 以外之目標的建置會省略它。

圖中每個套件的系統函式庫都會被連結，每個名稱一個；登錄庫套件的系統函式庫來自其 `nabi.nabion`（索引行中沒有）。它們不在 `nabi.lock` 中：它們就是機器所擁有的東西。

## 9.10.4c. feature

套件在 `features` 中宣告 feature，即一個從 feature 名稱對應到該 feature 所啟用之東西清單的字典。名稱是 `[a-z0-9][a-z0-9_-]*`，以字詞說明它們是什麼（`http1`、`http2`、`tls`）。每個項目是下列其中之一：

| 項目        | 啟用                                                                 |
|-------------|----------------------------------------------------------------------|
| `<f>`       | 這個套件的 feature `f`                                               |
| `dep:<d>`   | 選用的相依套件 `d`                                                   |
| `<d>/<f>`   | 相依套件 `d` 的 feature `f`，若 `d` 是選用的，也啟用 `d` 本身        |
| `<d>?/<f>`  | 選用相依套件 `d` 的 feature `f`，只在有其他東西啟用 `d` 時           |

- `default` 是一個一般的 feature，除非被關閉，否則是開啟的：對正在建置的套件是 `--no-default-features`，對相依套件是 `default_features: false`。
- 選用的相依套件（`optional: true`）只在某個已啟用之 feature 的 `dep:<d>` 或 `<d>/<f>` 項目啟用它時才會在建置中。沒有以其名稱命名的隱含 feature，沒有任何項目啟用它是錯誤。
- 若項目指名了套件沒有的 feature 或（`deps` 的）相依套件、若 `dep:` 或 `?` 指名了非選用的相依套件，或若 feature 形成循環，則是錯誤。診斷顯示 `file:line:col`。要求相依套件提供它沒有的 feature 是錯誤。
- 解析：鎖定檔涵蓋整個圖。接著建置從根套件的 feature（除非被關閉否則為 `default`、`--features`，或有 `--all-features` 時為每個 feature）開始，追蹤每個已啟用的相依套件，連同它對該相依套件要求的 feature 加上其 `default`。feature 是累加且統一的：每個套件在整個圖中只有一組 feature。根的 `dev_deps` 以及它們所要求的 feature 只在 `test` / `bench` 時算數。結果不取決於任何東西的順序。
- `--features` 接受根的 feature 名稱，以及其直接相依套件的 `<dep>/<f>`。
- 每個套件都恰好以其已啟用之 feature 的 `cfg(feature = "<f>")` 編譯（[6.3.6](../006-meta-system/603-cfg.md#636-參數的來源)）。原生套件沒有 feature。
- 腳本的 ```` ```nabion ```` 區塊沒有 `features`（9.10.7）。

## 9.10.5. 環境

環境取代 conda / venv。沒有啟用步驟。

- 套件（含有 `src/main.nabi`、`src/lib.nabi` 或 `nabi.nabion` 的目錄）使用它自己之 `nabi.nabion` 的相依套件，而不使用其他任何東西；它絕不會看到安裝到環境中的函式庫。腳本也不會。作用於套件的指令搭配 `--env` 是錯誤。
- 具名環境把 `nabi.nabion`、`nabi.lock` 與 `target/` 保存在 `$NABI_HOME/envs/<name>/` 中。其 `deps` 是以 `nabi install` 安裝的函式庫（[9.10.5a](#9105a-nabi-install--nabi-uninstall)），供 REPL 與 Jupyter 使用；只有 `nabi install` / `nabi uninstall` 與 `nabi env import` 會改變它們。其中的路徑相依套件是絕對路徑。已安裝的函式庫會預先建置到其 `target/` 中。
- 在套件之外使用的環境（REPL、Jupyter、語言伺服器、`nabi install`）依 `--env <name>` > `NABI_ENV` > `nabi env use`（`$NABI_HOME/envs/.current`）> `default` 的順序選擇；`default` 由第一次對它執行的 `nabi install` 建立。在套件內，不帶路徑的 `nabi repl` / `jupyter` / `lsp` 使用該套件，搭配 `--env` 時則使用該環境。
- 環境名稱由 ASCII 字母、數字、`_`、`-` 與 `.` 組成，不以 `.` 開頭，最多 64 個字元，且不是 Windows 裝置名稱（`con`、`nul.txt`……）。`nabi env create` 會拒絕另一個環境以不同大小寫擁有的名稱（`Foo` 與 `foo` 並存）：Windows 與 macOS 把它們視為同一個目錄。
- 名稱的每個來源（`--env`、`NABI_ENV`、`nabi env use` 檔案、`nabi env` 的引數）都會被檢查。帶有分隔符號（`/`、`\`、`:`）、`.` 或 `..` 的名稱在任何地方都是錯誤（`` `../p` is a path, not an environment name; use a name from `nabi env list`, e.g. `default` ``）。`nabi env create` 會拒絕的其他任何名稱也是錯誤，除非該環境存在（其目錄是手動建立的）：此時是警告，指令繼續執行。`nabi jupyter install --env <name>` 要求該環境存在，`default` 除外。
- `nabi env use` 把名稱寫到 `$NABI_HOME/envs/.current`。只讀取第一行。對所選環境執行 `nabi env remove` 會移除該檔案，因此就沒有選擇。

### 相依套件從哪裡來

| 編譯為 | 相依套件 | 鎖定檔 | 預先建置到 |
|---|---|---|---|
| 套件 | 其 `nabi.nabion` 的 `deps`（test / bench 時為 `dev_deps`） | `<package>/nabi.lock` | `<package>/target/<profile>/deps/` |
| 腳本 | 它以及其 `mod` 所載入之檔案的 `//! nabion` 區塊合併後的 `deps` | `cache/scripts/<hash>/nabi.lock` | `cache/scripts/<hash>/target/` |
| 套件之外的 REPL / Jupyter | 作用中環境的已安裝函式庫 | 該環境的 | 該環境的 `target/` |

```
nabi env create <name>
nabi env list                      # marks the current environment with * and (active); with none, says how to make one
nabi env remove <name>
nabi env use <name>                # changes the default environment outside projects
nabi env export <name> <dir>       # writes nabi.nabion / nabi.lock to <dir>
nabi env import <name> <dir | nabi.nabion>    # creates from a project or export (relative paths made absolute)
nabi env shell [name]              # $SHELL with NABI_ENV=<name>
nabi repl --env <name>
```

REPL 中的 `:add <dep> ..` 對目前的環境執行 `nabi install <dep> ..`，或在套件的工作階段中執行 `nabi add`；然後它重新讀取相依套件，並把它們附加到工作階段。

## 9.10.5a. `nabi install` / `nabi uninstall`

```
nabi install <name>[@<req>] [--registry <url>]
nabi install [<name>] --path <dir | file.nabi>
nabi install [<name>] --git <url> [--rev <rev> | --tag <tag> | --branch <branch>]
nabi install .. [--features <f,..>] [--no-default-features] [--lib | --bin] [--force]
nabi install --native <name>[@<req>] [--registry <url>]
nabi install --list
nabi uninstall [--native] <name>..
```

- 位於來源的套件會先擷取到快取中（[9.10.4](#9104-相依套件nabi-add--remove--update--deps)）。帶有 `src/main.nabi` 的套件，或腳本檔案，是執行檔：它以 `--release` 建置，其執行檔放在 `$NABI_HOME/bin/<name>`，與 `cargo install` 相同。git 或登錄庫套件從其 checkout 的暫存複本建置；路徑套件就地建置；腳本檔案也就地建置。`$NABI_HOME/installs.nabion`（`Package { deps: { "<name>": <source> } }`）記錄每個執行檔及其確切來源（套件寫成 `=<version>`，git 套件帶有其 `rev:`）。當套件也有 `src/lib.nabi` 時，`nabi install` 需要 `--bin` 或 `--lib`：兩者都沒有時是錯誤，錯誤訊息會指出執行檔用 `--bin`、函式庫用 `--lib`，且不會安裝任何東西。對沒有 `src/main.nabi` 的套件使用 `--bin` 是錯誤，同時使用 `--bin` 與 `--lib` 也是。
- 只有 `src/lib.nabi` 的套件，或給出 `--lib` 的任何帶有 `src/lib.nabi` 的套件，是函式庫，會與 `pip install` 一樣放進作用中的環境，供 `nabi repl`、`nabi jupyter` 與 `:add` 使用（[9.10.5](#9105-環境)）：它被寫進環境的 `nabi.nabion`（不帶需求的登錄庫套件寫成 `^<newest version>`，路徑寫成絕對路徑），解析到其 `nabi.lock` 中，並預先建置到其 `target/` 中。腳本看不到它：腳本以 `nabi add <name> <file.nabi>` 取得函式庫；輸出行會指出這一點。若它無法建置，就不會安裝任何東西。對沒有 `src/lib.nabi` 的套件或對腳本使用 `--lib` 是錯誤。
- `<name>` 是函式庫的頂層名稱，或執行檔的檔案名稱。省略時，它是套件的 `name`，否則是目錄名稱（路徑）、去掉 `.git` 的儲存庫名稱（git）或檔案主檔名（腳本）。它遵循 [9.10.3](#9103-nabinabion--nabilock) 的命名規則。
- `--features` / `--no-default-features` 放進函式庫的 `Dep`，或交給執行檔的建置；`--dev` / `--optional` 是錯誤。`--native` 把原生套件加入環境的 `native`（[9.10.4b](#9104b-原生套件)）。
- 代理程式 `nabi`、`nabic`、`nabicc`、`nabils`（[9.10.6](#9106-toolchain)）絕不會被取代，`$NABI_HOME/bin` 中不是由 `nabi install` 放進去的檔案也不會（這是錯誤）。再次安裝執行檔時：來自相同登錄庫版本或 git 提交且 feature 相同者會被保留，不重新建置（`` `tool` 0.1.0 is installed already (<path>); to rebuild it, run `nabi install tool --force` ``），而 `--force` 會重新建置它（`Reinstalled`）；另一個版本會取代它（`Upgraded tool from 0.1.0 to 0.2.0`、`Downgraded ...`），另一個 git 提交也是（`Replaced <url> at <commit> with ...`）。來自路徑或腳本檔案者一律重新建置，與 `cargo install --path` 相同。函式庫的安裝不變。
- `--list` 印出 ``libraries (environment `<env>`):`` 與 `executables (<bin dir>):`，各自之後依安裝順序接著 `  <name> <source>` 行。
- `nabi uninstall <name>..` 從作用中環境的函式庫中移除每一個（重新解析其鎖定檔），否則移除 `nabi install` 放在 `$NABI_HOME/bin` 中的執行檔；`--native` 從環境中移除原生套件。兩者都不是的名稱是錯誤。
- 除了 `nabi install` 之外，只有 `nabi toolchain`（工具鏈、代理程式、`config.nabion`）、`nabi login`（`tokens.nabion`）、`nabi env`（環境）以及各快取會寫入 `$NABI_HOME` 之下。

## 9.10.6. toolchain

`$NABI_HOME/bin/` 中的 `nabi` / `nabic` / `nabicc` / `nabils` 是執行作用中工具鏈之同名執行檔的代理程式（rustup 風格）。作用中的工具鏈依下列順序選擇：

1. 命令列上的 `+<version>`（`nabi +0.4 build`）
2. 環境變數 `NABI_TOOLCHAIN`
3. 所執行之腳本區塊中的 `toolchain: "X"`（[9.10.7](#9107-單一檔案腳本的行內資訊清單)），適用於 `nabi <file>`，以及對套件之外的 `.nabi` 檔案執行的 `nabi run` / `build` / `check` / `test`（代理程式讀取該檔案的前 64 KiB）
4. 資訊清單中的 `toolchain: "X"`（最近的 `nabi.nabion`；代理程式只讀取這個欄位，因此它不認識的欄位不影響釘選）
5. 預設值（`$NABI_HOME/config.nabion` 中的 `Config { toolchain }`）

釘選是下列其中之一：

- 一個版本或版本的前綴，比對其下的版本（`0.3` 是最高的 `0.3.*`，`0.3.1` 是 `0.3.1`）；
- 9.10.4a 中的需求（`^0.3.1`、`~0.3.1`、`=0.3.1`、`>=0.3, <0.5`），比對它所接受的最高已安裝版本；
- 工具鏈目錄名稱（`0.3.1-<triple>`），比對該目錄。

相符的最高已安裝正式版勝出；預發行版（`0.3.1-rc.1`）只在沒有正式版相符時才勝出，因此 `0.3` 會選擇 `0.3.0` 而不是 `0.3.1-rc.1`，而只安裝了候選版的系列會選擇其中最新的。確切指名預發行版的釘選（`0.3.1-rc.1`）只比對它。不屬於這些的資訊清單釘選是警告，會說明該寫什麼（`` `toolchain: "banana"`: write a version (`0.1`), a requirement (`^0.1.0`) or a toolchain directory name ``）。若沒有任何工具鏈相符，就是錯誤，錯誤訊息會指出該釘選及其來源，以及安裝它的指令：完整版本時為 `nabi toolchain install <pin>`，否則為帶有該釘選所接受之完整版本的 `nabi toolchain install <version>`，或 `--from`。

一個系列是一個破壞性變更槽位所涵蓋的各個版本：主版本為 0 時是 `0.MINOR`（`0.1`），之後是 `MAJOR`（`1`）。只有最新的系列會得到修正與安全性版本；較舊的系列仍可下載與安裝，但不受支援。

### 相容性

工具鏈能建置其系列中較早工具鏈所建置的程式碼：在 1.0 之前，`x.y.z` 中 `y` 的改變可能破壞程式碼（`0.1.7` 到 `0.2.0`），從 1.0 起，`x` 的改變會破壞（`1.9.0` 到 `2.0.0`），而較後的 `z`（1.0 之前）或 `y`（從 1.0 起）則不會。套件的 `nabi_version`（9.10.3）遵循這個規則：`"0.1.3"` 接受 0.1.3 到最後一個 0.1.x，`"1.4.2"` 接受 1.4.2 到最後一個 1.x。預發行版的工具鏈算作其正式版（`0.1.4-dev` 算作 `0.1.4`）。

代理程式是最新已安裝工具鏈之 `nabi` 的複本：這個主機的最高版本，只有在沒有安裝正式版時才是預發行版。較新的 `nabi` 以這種方式持續執行較舊的工具鏈，因此代理程式的契約在整個 0.x 中是固定的：上述的順序，然後以相同的引數執行 `toolchains/<dir>/bin/<tool>`，設定 `NABI_HOME`，並把 `NABI_TOOLCHAIN` 設為代理程式所取下的 `+<version>`。

每個工具鏈的 `nabi` 只執行它自己的 `nabic`，即它旁邊的那一個，因此 `nabi` 與 `nabic` 之間的旗標、環境變數與檔案絕不是跨版本的契約。直接執行的 `nabi`（開發建置、套件管理器的、工具鏈的 `bin/nabi`），在 `+<version>`、`NABI_TOOLCHAIN` 或資訊清單的 `toolchain` 指名另一個已安裝的工具鏈時，會把指令交給該工具鏈的 `nabi`：它從啟動所在的目錄以相同的引數執行它，設定 `NABI_HOME`，並以其狀態結束（在 Unix 上它取代自己；在 Windows 上它等待）。對會編譯的指令（委派給 `nabic` 的，或會編譯的工具），資訊清單是該指令所作用之套件的資訊清單（`nabi build ../app` 讀取 `../app/nabi.nabion`）；對其他指令則是最近的那一個。當釘選與它自己的版本相符時、當該工具鏈就是它執行所在的目錄時（以實際路徑比較），或當該工具鏈的 `nabi --version` 印出與它自己相同的行時（同一個建置），它不會交出。沒有釘選時，預設值不適用：直接執行的 `nabi` 保留該指令。`nabi toolchain`、`help`、`version`、`completions` 與 `--version` 絕不交出。被交出指令的工具鏈若又把它交回，是錯誤，錯誤訊息會指出兩個執行檔，絕不會形成迴圈。對會編譯的指令，指名了未安裝工具鏈的釘選是上述的錯誤；其他指令則繼續執行。

```
nabi toolchain list                         # marks the default with * and (default)
nabi toolchain install <version> [--from <dir | archive | url>]
nabi toolchain default [<version>]          # without an argument, prints the current default
nabi toolchain remove <version>
nabi toolchain update [<version>]
nabi toolchain clean                        # remove every installed toolchain but the default
```

- 安裝是 `$NABI_HOME/toolchains/<version>-<triple>/`，是自給自足的（[9.2.19](902-aot-pipeline.md#9219-安裝配置)）。
- `install --from` 從目錄、封存檔或其 URL 安裝；`<version>` 是工具鏈必須相符的釘選。沒有 `--from` 時，`install` 會下載一個發行版（發行，見下文）：`<version>` 是完整的版本（`0.1.0`）；其他任何東西都是錯誤，錯誤訊息會說明要寫完整版本（`` nabi toolchain install: `0.1` is not a full version; write one like `0.1.0` ``）。已安裝的工具鏈不會再次下載（`toolchain <dir> is already installed`）。它不改變預設值。
- `install` 以 `bin/nabic --version --verbose` 驗證；版本與主機必須相符。
- 第一個安裝的工具鏈成為預設值，並在那時建立 `bin/` 代理程式。
- `install`、`default`、`update`、`remove` 與 `clean` 在某個 `bin/` 代理程式缺少或不是最新工具鏈的 `nabi` 時改寫它們，否則保持不動。把較舊的工具鏈設為預設值會保留較新的代理程式；移除最新的工具鏈會讓它們變成次新的工具鏈的。
- `nabi toolchain update` 取代 `$NABI_HOME` 之下的預設工具鏈。不帶引數時，它安裝最新的發行版（`latest/download/nabi-version`），即使它位於另一個系列也是（rustup 風格）。當預設值移到另一個系列時，說明舊工具鏈仍保持安裝的附註也會說明只有最新的系列會得到修正，因此專案應當遷移，以及專案在遷移之前如何停留在舊系列：``note: nabi <old version> stays installed, but <old series> gets no more fixes; move projects to <new version>. Until a project moves it can pin <old series> (`toolchain: "<old series>"` in nabi.nabion or a script's `//! nabion` block) or run `nabi +<old series> ..` ``。帶有引數時，它安裝該發行版：引數是完整的版本（`update 0.1.1`），沒有系列選擇；其他任何東西都是錯誤，錯誤訊息會說明要寫完整版本（`` nabi toolchain update: `0.1` is not a full version; write one like `0.1.0`, or nothing for the newest release ``）。安裝的工具鏈成為預設值，當最新者改變時會改寫 `bin/` 代理程式；在 Windows 上，執行中的代理程式會在其複本取代它之前被重新命名。它下載發行版封存檔並驗證其 SHA-256（發行，見下文），在別處解開後移到定位，然後檢查代理程式能執行它；失敗時會還原先前的預設值與代理程式，並移除新的工具鏈。較早的工具鏈保持安裝。已安裝的版本會在不下載的情況下設為預設值。`nabi update` 重新解析 `nabi.lock`；`nabi toolchain update` 取代工具鏈（9.10.4）。
- 在 `nabi toolchain default <version>` 或 `nabi toolchain update` 之後，安裝程式所寫出的補全腳本（發行，見下文）會從新的預設工具鏈的 `nabi completions` 改寫。只會改寫已存在的檔案；不會建立任何檔案，沒有 `completions` 的工具鏈會讓它們保持原樣。這不會印出任何東西；`-v` 會指出被改寫的檔案。
- 在 `nabi toolchain default <version>` 或 `nabi toolchain update` 之後，`nabi` 會在 `PATH` 上查找 `nabi`（在 Windows 上搭配每個 `PATHEXT` 副檔名）。當找到的第一個不是 `$NABI_HOME/bin/nabi`（以實際路徑比較）時，它會以該檔案及其 `--version` 所印出的版本發出警告：`warning: your shell runs <path> (nabi X) before ~/.nabi/bin/nabi; put ~/.nabi/bin first on PATH or remove the other one`。`-q` 會省略這則警告（安裝程式就是這樣執行它的）。
- `nabi --version -v`（或 `nabi version -v`）在版本行之後印出 `binary`（這個執行檔）、`toolchain`（作用中的工具鏈及其原因：命令列上的 `+`、`NABI_TOOLCHAIN`、釘選它的 `nabi.nabion`，或預設值；沒有時為 `none`）、`nabic`（在這裡執行的指令所執行的 `nabic`：它自己的，或交出時被釘選之工具鏈的）以及 `host`，每個一行 `key: value`，值對齊，形式與 `nabic --version -v`（`host: <triple>`）相同。`--format json`（`nabi version -v --format json`、`nabi --version --format=json`）則印出一個 JSON 物件：`schema`（目前是 `1`）、`version`、`line`（版本行中 `nabi ` 之後的文字），以及有 `-v` 時的那四個鍵。
- `nabi toolchain clean` 移除目前預設值以外的每個已安裝工具鏈，這是針對 `update` 所留下之較早版本的選擇性整理。專案所釘選的版本也會被移除，因此若建置需要它，請重新安裝；沒有設定預設值時是錯誤。

### 建置快取

```
nabi cache dir                              # print the toolchain's cache directory
nabi cache prune [--unused <days>] [--max-size <size>]
nabi cache clean                            # remove everything in it
```

快取是作用中工具鏈的 `lib/nabi-cache/`（[9.2.19](902-aot-pipeline.md#9219-安裝配置)）：函式快取，內含各建置所編譯的標準函式庫程式碼，以及函式庫所收集的項目（[9.2.16](902-aot-pipeline.md#9216-增量建置)）。其中的任何東西都不是必要的；建置會重新計算缺少的東西。

- `prune` 移除 `--unused` 天內（預設 30，即各建置每天自行移除一次的天數）沒有任何建置讀取過的項目（函式快取與函式庫階段的），然後移除最久沒有被讀取的項目，直到剩下最多 `--max-size`（位元組數，或帶 `K`、`M` 或 `G` 的數字；預設為上限 `NABI_CACHE_MAX_SIZE`，[9.2.16](902-aot-pipeline.md#9216-增量建置)），並印出它移除與保留了多少。
- `clean` 移除快取目錄中的一切。

### 跨工具鏈共用的檔案

安裝在同一個 `$NABI_HOME` 中的每個工具鏈都讀寫那裡相同的檔案：`config.nabion`、`tokens.nabion`、`sources.nabion`、`installs.nabion` 與 `envs/.current`。較新的 `nabi` 可能在其中寫入了較舊者不認識的欄位。

- 讀取它們時，`nabi` 會略過它不認識的欄位，在 `installs.nabion` 中還會略過它不認識的 `Dep` 引數與項目種類（它不認識之種類的項目不會被列出）。有 `-v` 時，它每個檔案指出一次它略過了什麼：``note: <file>: skipped `x`, written by a newer nabi (kept in the file)``。
- 寫入它們時，它會保留它不認識的東西：`nabi toolchain default` 設定 `config.nabion` 的那一個欄位，`nabi install` / `nabi uninstall` 設定或移除 `installs.nabion` 的那一個項目，`nabi login` 在 `tokens.nabion` 中設定其索引的項目，而 `nabi logout` 只移除該項目。`sources.nabion` 只會被讀取。
- `nabi` 不認識的 `nabi.nabion` 欄位是警告（[9.10.3](#9103-nabinabion--nabilock)）；`nabi` 認識之版本的 `nabi.lock` 仍是嚴格的。
- 較新的 `nabi` 以新的檔案新增設定，而不是作為這些檔案的新欄位，讓較舊的工具鏈能在它旁邊持續運作。

### 發行

- 發行版封存檔是 `nabi-<version>-<triple>.tar.gz`（Windows 為 `.zip`），內含工具鏈配置；在 Windows 以外，旁邊還有一個檔案相同的 `nabi-<version>-<triple>.tar.xz`。每個封存檔旁邊都有 `<archive>.sha256`：以十六進位表示的封存檔 SHA-256，後面可以接著檔案名稱（sha256sum 格式）。
- `install.sh`、`install.ps1` 與 `nabi toolchain install --from <url>` 從同一個地方下載 `<url>.sha256`，並拒絕 SHA-256 不同的封存檔；缺少 `.sha256` 是錯誤。以路徑給出的封存檔，在 `<path>.sha256` 存在時會對照它檢查。未定義簽章；由 HTTPS 擔保來源。
- `install.sh`（<https://nabi.sh>）從封存檔或從 `nabi-lang` 組織的 GitHub releases 安裝到 `$NABI_HOME`。在 tar 能讀取 `.tar.xz` 之處（macOS，或 `PATH` 上有 `xz`）且發行版有它時，它下載 `.tar.xz`，否則下載 `.tar.gz`。`nabi toolchain install` 下載 `.tar.gz`（在 Windows 上是 `.zip`）。
- 接著 `install.sh` 與 rustup 一樣設定 shell。它寫出 `$NABI_HOME/env`（sh）與 `$NABI_HOME/env.fish`，它們會把 `$NABI_HOME/bin` 放在 `PATH` 的最前面（若尚未在那裡），並把帶有防護的一行 `[ -f "$HOME/.nabi/env" ] && . "$HOME/.nabi/env"`（針對該 `$NABI_HOME`）加到已存在的 `~/.profile`、`~/.bashrc`、`~/.bash_profile` 與 `~/.zshenv` 中各一次，也加到登入 shell 的檔案中，缺少時會建立（bash：`~/.bashrc`，以及除非 `~/.bash_profile` 或 `~/.bash_login` 存在，否則還有 `~/.profile`；zsh：`$ZDOTDIR/.zshenv`；其他 shell：`~/.profile`）。當 fish 是登入 shell 或其設定目錄存在時，會得到 `~/.config/fish/conf.d/nabi.fish`（設定了 `$XDG_CONFIG_HOME` 時位於其下）。`nabi` 的補全腳本放到 `${XDG_DATA_HOME:-~/.local/share}/bash-completion/completions/`、`$NABI_HOME/zsh/`（`env` 會把它加到 zsh 的 `fpath`），以及對 fish 而言的 `~/.config/fish/completions/`。它會指出它所改變的檔案；新的 shell 會套用這些改變，再次執行它不會改變任何東西。沒有 `$NABI_HOME` 時，帶有防護的那一行不會做任何事。
- `install.ps1` 把 `$NABI_HOME\bin` 放在使用者 `Path` 的最前面，並寫出 `$NABI_HOME\completions\nabi.ps1`，由 PowerShell 設定檔中一行帶有防護的程式碼載入，該行只會加入一次。
- `install.sh` 與 `install.ps1` 與 `nabi` 一樣在 stderr 上印出狀態行，使用相同的調色盤與相同的規則（[9.1.5](901-nabic-cli.md#915-全域選項)、[9.1.8](901-nabic-cli.md#色彩)），不使用 `tput`。`install.sh` 會測試其 stderr，因此在終端機上執行 `curl .. | sh` 會帶有色彩。
- `--no-modify-path`（`install.ps1 -NoModifyPath`），或 `NABI_NO_MODIFY_PATH=1`，不會動 shell 檔案、`Path` 與設定檔，並印出要加到 `PATH` 的內容。
- 發行版封存檔在 `lib/nabi/packages/` 中攜帶官方套件（`nabi component`，9.10.4a），每個都是其 `nabi_version` 接受該發行版的最新版本，因此使用它們的程式碼不需要網路就能建置。`--no-packages`（`install.ps1 -NoPackages`），或 `NABI_NO_PACKAGES=1`，會在安裝之後對已安裝的工具鏈執行 `nabi component remove packages`；若失敗，工具鏈仍會連同它們保持安裝，腳本會說明這一點。
- 發行版由 `$NABI_DIST`（預設 `https://github.com/nabi-lang/nabi/releases`）提供：`<dist>/download/v<version>/<archive>` 及其 `.sha256`，以及 `<dist>/latest/download/nabi-version`，即指名任何系列之最新發行版的一行，由單純的 `nabi toolchain update` 讀取。候選發行版（`<version>-rc.N`）是預發行版：`latest/` 絕不會指名它，因此只能以其完整版本安裝。
- `nabi toolchain update <version>` / `install <version>`、`install.sh --version <version>` 與 `install.ps1 -Version <version>` 接受完整版本；最新者以外的發行版以這種方式安裝。不會發布各系列專屬的檔案。

### 安裝程式契約

每個已發行工具鏈的 `nabi toolchain update` / `install` 以及每個發行版的安裝腳本，都必須能夠安裝之後的每個發行版，因此發行版封存檔會保留它們所依賴的東西。這份契約是凍結的；破壞它的發行版不會被發布。

- 名稱：`nabi-<version>-<triple>.tar.gz`（Windows 為 `nabi-<version>-<triple>.zip`），在 Windows 以外還有 `.tar.xz`，依上述發行一節的方式提供，各自帶有其 `.sha256`。
- 指標：`<dist>/latest/download/nabi-version`，即指名最新發行版的一行，讓它們單純的 `nabi toolchain update` 能到達它。
- 配置：一個頂層目錄 `nabi-<version>-<triple>/`，容納 `bin/nabi` 與 `bin/nabic`（Windows 為 `bin\nabi.exe` 與 `bin\nabic.exe`）、`lib/<runtime>`（`lib/libnabir.a`；Windows 為 `lib\nabir.lib`）以及 `lib/nabi/core/src/lib.nabi`。
- `bin/nabic --version --verbose` 在第一行印出 `nabic <version>`（後面可以接著一個空白與更多文字），並印出一行 `host: <triple>`。

## 9.10.7. 單一檔案腳本的行內資訊清單

匿名套件（[4.16.8](../004-functions/416-entry-point.md#4168-單一檔案執行)）只從其檔案以及其 `mod` 所載入之每個檔案最開頭（在 shebang 行之後）的 `//!` 行內、容納 `Package { deps: { .. } }`（以及 `native`）的 ```` ```nabion ```` 區塊取得其相依套件（類似 uv 的行內腳本中繼資料）。安裝到環境中的函式庫絕不會被使用。區塊只能包含 `deps`、`native` 與 `toolchain`（沒有 `features`，因此沒有選用的相依套件）。`toolchain` 釘選執行該腳本的工具鏈（9.10.6），只允許出現在被執行之檔案的區塊中；在模組的區塊中是錯誤，錯誤訊息會指出被執行的檔案。直接執行的 `nabi` 會與對套件一樣，把腳本交給被釘選之工具鏈的 `nabi`。其他語言的程式碼區塊會被忽略。區塊中的路徑相對於容納它的檔案。各區塊會被合併；在兩個檔案中被給予不同項目的同一個函式庫是錯誤，錯誤訊息會指出兩個檔案與兩個項目。直接執行子模組檔案時，只使用該執行所到達之檔案的區塊。`nabi add <name> .. <file.nabi>` / `nabi remove <name> <file.nabi>` 寫入區塊（[9.10.4](#9104-相依套件nabi-add--remove--update--deps)），`nabi init --script` 把腳本變成套件（[9.10.2](#9102-nabi-new--nabi-init建立套件)）。腳本把其鎖定檔與建置保存在 `cache/scripts/<hash>/` 中，以其路徑以及它所到達之每個檔案的路徑與區塊作為鍵。腳本的執行檔以建置它的一切作為快取鍵，包括它所引入的每個檔案及其函式庫。腳本對它所缺少之套件的 `use x::..` 是錯誤，其說明是要執行的指令 ``run `nabi add x <file>` ``，當作用中的環境有 `x` 時附上一則附註；在套件中，它建議 `nabi add x`。`import x`（以及 `include` / `require`）是錯誤，當 `x.nabi` 位於檔案旁邊時，其說明是 `mod x;`，當 `x` 是路徑根或檔案所宣告的模組時是 `use`，否則是 `nabi add x <file>`。

````nabi
//! ```nabion
//! Package {
//!     deps: { "util": Dep::git("https://example.com/util.git", tag: "v1") },
//! }
//! ```

use util::greet;
greet();
````

執行單一檔案腳本會重複使用其執行檔。匿名套件的 `nabi run` 把執行檔保存在 `<target>/[<triple>/]<profile>/scripts/<key>/` 中，其中 `<target>` 是腳本的 `target/`（位於 `cache/scripts/<hash>/` 之下，在其 `nabi.lock` 旁邊）；由 `nabi.nabion` 使其成為單一檔案套件的檔案（9.10.1），以及由 `nabic` 建置的單獨檔案，則改用單獨檔案快取（[9.2.16](902-aot-pipeline.md#9216-增量建置)）。`<key>` 是建置所依據之一切的雜湊：工具鏈、建置選項、lint 等級與目標、原始碼及其引入的每個檔案、工具鏈為該目標隨附的執行環境與 C 函式庫，以及傳給它的相依套件及其預先建置的函式庫。當鍵相符時，不會編譯任何東西，並再次印出產生該執行檔之建置的警告。新的執行檔以暫存名稱寫出，再重新命名就位，因此同一個腳本的並行執行是安全的。一天未使用的項目會在建置另一個項目時被移除。

## 9.10.8. 工具

各工具是 `nabi` 自己的子指令。`nabi` 在它自己的行程中、連結了編譯器的情況下執行它們；會編譯的工具接受 `nabic build` 的編譯選項（[9.1.2](901-nabic-cli.md#nabic-buildaot-建置)），並依 9.10.1 所述取得套件的相依套件、feature 與 lint。它們的診斷、進度行與色彩遵循 [9.1.8](901-nabic-cli.md#918-輸出格式)，用法錯誤與結束碼則遵循 [9.1.6](901-nabic-cli.md#916-結束碼)，除非工具另有說明。`nabi lsp`、`nabi repl` 與 `nabi jupyter` 能撐過編譯器內部錯誤：語言伺服器把它回報為被分析之緩衝區的錯誤診斷，REPL / 核心讓該儲存格失敗（[9.7.5](907-jit-cells.md#975-錯誤模式)）；兩者都繼續提供服務。

### `nabi run`：建置並執行

```
nabi run [path] [build options] [-- args...]
```

`nabi run` 與 `nabic build` 一樣建置可執行套件，並立即執行它（另一個 `--target` 的執行檔經由其執行器執行，見下文）；`--` 之後的引數原樣傳給執行檔，可以是任何位元組（[9.1.1](901-nabic-cli.md#911-一般形式)）。選項就是 `nabic build` 的選項。函式庫套件沒有東西可以執行，因此是錯誤。

套件建置到 `target/<profile>/<name>`；單獨的 `.nabi` 檔案作為匿名套件建置，並從 9.10.7 的快取執行，因此不會在檔案旁邊建立 `target/`。`nabic build <file>.nabi` 則會建置它並把執行檔複製到檔案旁邊。若自上次建置以來沒有任何改變，`nabi run` 會印出 `Fresh`，然後再次印出那次建置的警告，並略過建置（[9.2.16](902-aot-pipeline.md#9216-增量建置)）。

```
nabi run hello.nabi -- arg1 arg2
```

`nabi run` 的結束碼就是程式的結束碼。若程式被訊號 `n` 終止，則為 `128 + n`。

程式不會在 `nabi run` 結束之後繼續存活。送給 `nabi` 的 `SIGTERM`、`SIGHUP`、`SIGINT` 或 `SIGQUIT` 會轉送給程式（終端機送給其前景群組的訊號本來就會到達它），而若程式因該訊號結束，`nabi run` 也會因它結束。若 `nabi run` 本身被終止，程式也會一起被終止；在 Windows 上，程式在一個與 `nabi` 一起結束的 job object 中執行。

### `nabi dev`：以熱重載執行

```
nabi dev [path] [-p <member>] [--bin <name>] [build options] [-- args...]
```

`nabi dev` 為熱重載（[9.14](914-hot-reload.md)）建置套件的程式，執行它，並監看其原始碼。儲存會重新建置它所觸及的本地套件，程式在執行中取用新的程式碼，並保留其狀態。

- *主機*（執行環境以及每個登錄庫或 git 套件）只建置一次，即 `target/dev/<name>-host`。每個本地套件（該套件、其 `path` 相依套件、它所使用的工作區成員）都是熱外掛，以 `nabic build --hot` 建置為 `target/dev/<package>.<n>.dylib`（`.so`、`.dll`），建置編號 `n` 從 1 開始，旁邊是其配置 `target/dev/<package>.<n>.layout`。工作階段開始時會清空 `target/dev/`。
- 每 100 ms 輪詢一次本地套件的 `src/` 樹（每個 `.nabi` 檔案的修改時間與大小）。在檔案靜止 50 ms 之後才採納改變，因此一次儲存多個檔案算作一個改變。它增量地（[9.2.16](902-aot-pipeline.md#9216-增量建置)）重新建置檔案有改變的套件以及相依於它們的每個本地套件，相依套件在前，並把新版本送給程式（[9.14.5](914-hot-reload.md#9145-dev-頻道)）。
- 程式的回覆以狀態行印出：`Applied`、`Waiting`、`Refused`（附上該怎麼做）、`Superseded`。
- 編譯錯誤與 `nabi build` 一樣印出，其後接著 `` Kept the running code of `game`: fix the error and save ``。程式以其舊程式碼繼續執行。
- 對 `nabi.nabion`（套件的、本地相依套件的或工作區根的）、`nabi.lock`，或工具鏈之 `nabi` 或 `nabic` 的改變，會在一行說明原因的訊息之後重新啟動程式：`` Restarting nabi.nabion changed ``。重新啟動會重新解析相依套件，並重新建置一切；程式的狀態會遺失。
- 按鍵：`r` 重新啟動，`q` 結束。在終端機上，按鍵一按下就作用（`nabi dev` 會關閉行編輯與回顯，並在結束時恢復它們）；否則以每行的第一個字母為準。程式的標準輸入是空的；其標準輸出與錯誤就是 `nabi dev` 的。
- 當程式自行結束時，`nabi dev` 會連同其結束碼說明這一點並等待：儲存或 `r` 會再次執行它，`q` 結束。其熱外掛無法載入的程式會在載入器的錯誤之後以結束碼 101 結束（[9.12.6](912-plugins.md#9126-穩定性與錯誤)）。
- Ctrl-C（或 `SIGTERM`、`SIGHUP`）結束工作階段：程式被終止並回收，`nabi dev` 以 130 結束。`q` 以 0 結束。
- `--` 之後的引數在每次啟動時都傳給程式。

`-p <member>` 執行一個工作區成員；`--bin <name>` 指名套件的執行檔（一個套件有一個，以套件命名）。建置選項是 `nabic build` 的選項，僅適用於這台機器上的 dev 設定檔：`--release` 或其他 `-O` 等級是錯誤（``hot reload runs debug builds; drop `--release` ``），`--target`、`-o` 與 `--crate-type` 也是（``` `nabi dev` builds for this machine only; drop `--target` ```）。沒有 `src/main.nabi` 的套件是錯誤，錯誤訊息會說明要在程式的套件中執行它。

### 其他目標的執行檔

搭配這台機器本身無法執行之 `--target` 的 `nabi run`、`nabi test` 與 `nabi bench`，會經由目標執行器啟動每個執行檔，即一個依序接受執行檔及其引數的程式：

- `NABI_RUNNER=<program> [args]`（以空白分割）是主機以外每個目標的執行器。
- 沒有它時，`wasm32-wasip2` 與 `wasm32-wasip1-threads` 模組在 `PATH` 上的 `wasmtime` 之下執行：`wasmtime run`，帶有主機的環境（在 `wasm32-wasip2` 上還有其網路與名稱查找）、8 MiB 的引擎堆疊（與原生主執行緒一樣深），以及位於 `/` 的主機檔案系統。WASI 沒有工作目錄，因此相對路徑從 `/` 解析。
- `wasm32-unknown-unknown` 模組在 `PATH` 上的 `node` 之下，以工具鏈的 JS 主機執行，即 `lib/nabi/targets/wasm32-unknown-unknown/` 中的 `nabi.mjs`。
- 在 Linux 上，另一種 CPU 的 Linux 目標在 `PATH` 上 qemu-user 的 `qemu-<cpu>` 之下執行（`qemu-aarch64`、`qemu-arm`、`qemu-riscv64`、`qemu-ppc64le`、`qemu-s390x`、`qemu-i386`……），當 `/usr/<triple>`（`/usr/aarch64-linux-gnu`）目錄存在且未設定 `QEMU_LD_PREFIX` 時，帶有 `-L /usr/<triple>`；沒有 `qemu-<cpu>` 時，若 binfmt_misc 有 `qemu-<cpu>` 處理器，就直接執行。
- 主機作業系統之目標、針對主機能執行之另一種 CPU 時，會在尋找 qemu-user 之前直接執行：x86_64 上的 32 位元 x86，以及當 CPU 能在 EL0 執行 AArch32 程式碼時（Neoverse N1、Cortex-A72；不包括 Neoverse N2 或 Apple silicon），arm64 Linux 上的 32 位元 ARM。沒有任何暫存器會告訴程式這件事，因此 `nabi` 會執行一個極小的靜態 32 位元 ARM 執行檔一次來找出答案。

否則，或當該程式不存在時，指令會以錯誤失敗，錯誤訊息會指出要安裝什麼，或 `NABI_RUNNER`：``cannot run a wasm32-wasip2 module here: `wasmtime` is not on PATH; install it (https://wasmtime.dev), or set NABI_RUNNER to a program that runs it``。`nabi run` 在建置之前檢查；`nabi test` 與 `nabi bench` 在 harness 建置之後檢查，而 `--list` 不需要執行器就能列出。結束碼是執行器的結束碼，對上述所有情況而言就是程式的結束碼。

```
nabi test --target wasm32-wasip2
NABI_RUNNER="qemu-aarch64 -L /opt/sysroot" nabi run --target aarch64-unknown-linux-gnu
```

### `nabi lint`：linter

```
nabi lint [path] [--list] [--check-library] [-A|-W|-D <lint | group>..]
```

`nabi lint` 執行前端與 lint，不進行程式碼產生（`nabic check` 會加上程式碼產生，[9.1.2](901-nabic-cli.md#nabic-check診斷)）。每個會編譯的指令都回報相同的 lint（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）；`lint` 只執行它們與前端。診斷格式見 [9.1.8](901-nabic-cli.md#918-輸出格式)；lint 診斷的錯誤碼是 `E0131`。lint 與群組的清單見 [6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)。`--list` 印出每個 lint 及其預設等級與加入它的版本，然後是各群組。`--check-library` 與 `nabic check` 相同。

lint 等級由 `-A` / `-W` / `-D`（[9.1.5](901-nabic-cli.md#915-全域選項)）以及程式碼中的屬性設定。若有任何 `deny` 等級的 lint 觸發，結束碼是 `1`。

### `nabi fix`：套用修正

```
nabi fix [path] [--allow-dirty] [--diff] [-A|-W|-D <lint | group>..]
```

`nabi fix` 與 `nabi lint` 一樣檢查套件，並套用發現項目所帶的修正：恰好帶有一個編輯的 lint 發現項目或編譯器警告（帶有多個替代方案的診斷留給編輯器的快速修正）。只要找到編輯，它就會再次檢查並重複，最多 8 輪；與同一輪中另一個編輯重疊的編輯會等到下一輪。只會改變套件的檔案；絕不會改變相依套件與標準函式庫的檔案。帶有修正的 lint：

| lint | 修正 |
|------|-----|
| `unused_variables` | `let` 名稱前面加上 `_`（`let _y = 2;`） |
| `unused_imports` | 當 `use` 只匯入該名稱、單獨占一行，且上方沒有屬性時，刪除該 `use` |
| `bool_comparison` | `x == true` 變成 `x`，`x == false` / `x != true` 變成 `!x`（運算子周圍為 `!(x)`） |
| `while_true` | `while true` 變成 `loop` |
| `redundant_cast` | `x as T` 變成 `x` |
| `redundant_clone` | 刪除 `.clone()` |
| `missing_f_prefix` | 字串加上 `f` 前綴 |

lint 等級（`-A` / `-W` / `-D`、資訊清單的 `lints`、屬性）決定回報什麼，因而也決定修正什麼：`nabi fix -A unused_imports` 會保留未使用的匯入。當套件位於其目錄下有未提交變更的 git 工作樹中時，`nabi fix` 會拒絕（``the work tree has uncommitted changes (src/main.nabi); commit them first so the fixes can be reviewed and undone, or run `nabi fix --allow-dirty` ``）；`--allow-dirty` 仍會修正。在 git 之外沒有這項檢查。`--diff` 與 `nabi fmt --diff` 一樣印出第一輪的編輯，不寫入任何東西。沒有任何修正能解決的錯誤會以結束碼 `1` 與 ``N errors to fix by hand first; nothing was changed`` 停止它（在某一輪之後則是 ``the files fixed so far are kept``）。它以每個被改變之檔案的 `Fixed <file> (N fixes)`，或 `<name>: nothing to fix` 結束。

### `nabi repl`：終端機 REPL

`nabi repl` 在終端機中啟動互動式 shell。不帶子指令的 `nabi` 與它相同（9.10.1）。

```
nabi repl [path] [-q]
```

儲存格語意遵循 [9.7.0](907-jit-cells.md#970-儲存格工作階段的規則) 的規則：每個儲存格恰好執行一次，頂層的 `let` 繫結與項目延續到之後的儲存格，失敗的儲存格不會在繫結上留下痕跡。運算式儲存格印出其值（`Display`，否則 `Debug`，否則其型別名稱）。

- 預設模式以 Nabi 的程式碼產生器編譯每個儲存格，並把程式碼交給在工作階段期間持續存在的 JIT 主機行程；主機載入它並執行一次（[9.6](906-jit-engine.md)）。繫結、`static` 狀態與 `any` 值在儲存格之間延續。其型別無法在頂層命名的繫結（閉包、私有型別等）在建立時就會被拒絕。
- 診斷以儲存格自己的文字指名 `<repl>:line:col`；先前儲存格的項目與執行時位置指名 `<repl cell N>:line:col`（[9.7.5](907-jit-cells.md#975-錯誤模式)）。
- 在終端機上，輸入具有行編輯、Tab 補全、色彩以及整個儲存格的歷史（[9.7.8](907-jit-cells.md#行編輯)）；歷史保存在 `$NABI_HOME/repl_history` 中。
- 在終端機上，工作階段以工具鏈行與一則提示開始（[9.7.8](907-jit-cells.md#行編輯)）；`-q` 會省略它們。
- 沒有路徑時，工作階段以空的狀態開始。程式路徑（單獨的 `.nabi` 檔案或帶有 `src/main.nabi` 的套件）會被載入並執行，頂層陳述式作為儲存格，`fn main` 作為進入點，然後工作階段繼續（[9.7.10](907-jit-cells.md#9710-載入程式repl-path)）。函式庫套件路徑（`src/lib.nabi`）會被加為相依套件，並以 `use <name>::*;` 作為第一個儲存格，`:reset` 之後也是。
- 在套件內而沒有路徑時，不會載入該套件。找不到但由套件定義的名稱會得到提示 ``load this package: `nabi repl .` ``。
- 多於一個路徑是用法錯誤。不接受 `-`；標準輸入就是儲存格的輸入。
- `NABI_REPL_TIME=1` 在 stderr 上印出儲存格每個階段的時間（[9.9.9](909-debug-tools.md#999-repl--jupyter-偵錯)）。

兩個根檔案都有的套件是程式路徑；其函式庫是工作階段的相依套件。`nabi repl <path>` 從該路徑的專案解析相依套件與 lint（9.10.1）；在套件之外，工作階段使用作用中環境的函式庫（9.10.5）。

REPL 指令（`:type`、`:doc`、`:time`、`:vars`、`:load`、`:reload`、`:edit`、`:save`、`:opt`、`:reset`、`:add`、`:help`、`:quit` / `:exit`，以及單獨的 `exit` / `quit`）與偵錯輔助見 [9.9.9](909-debug-tools.md#999-repl--jupyter-偵錯)。

### `nabi jupyter`：Jupyter 核心

```
nabi jupyter install [--env <name>]
nabi jupyter --connection-file <path>
```

`nabi jupyter install` 註冊核心（[9.8.3](908-jupyter-integration.md#983-安裝-kernelspec)）；kernelspec 執行 `nabi jupyter --connection-file {connection_file}`。

`nabi jupyter --connection-file <path>` 就是核心。由 Jupyter 啟動它，因此使用者不直接執行它。`--connection-file` 接受 Jupyter 所提供之 ZMQ 連線檔的路徑；單純的 `nabi jupyter` 是指向 `nabi jupyter install` 的用法錯誤。也接受 `--features` 等建置選項。核心在與 REPL 相同的儲存格工作階段上執行（[9.7.0](907-jit-cells.md#970-儲存格工作階段的規則)），並與 `nabi repl .` 一樣載入它所啟動之處的套件；一旦開始監聽，它會在 stderr 上印出 `nabi jupyter: ready`（[9.8.2](908-jupyter-integration.md#982-本機原生核心運作)）。整合細節定義於 [Jupyter 整合](908-jupyter-integration.md)。

### `nabi lsp`：LSP 伺服器

```
nabi lsp
```

`nabi lsp` 在標準輸入與輸出上執行 Language Server Protocol 伺服器。它由 VS Code、Vim 與 Emacs 等 LSP 用戶端呼叫。獨立的編輯器二進位檔 `nabils` 是同一個伺服器。`.nabion` 緩衝區只檢查語法。在兩個根檔案都有的套件中，函式庫之模組樹所到達的檔案作為函式庫的一部分檢查，其他檔案則作為執行檔的一部分檢查，後者看得到函式庫的開啟緩衝區。各功能列於 [9.9.13](909-debug-tools.md#9913-lsp-伺服器)；檢查在背景中執行，期間的查詢從最後一次成功的檢查回應。

### `nabi fmt`：格式化工具

```
nabi fmt [path..] [--check] [--diff]
```

`[path]` 接受檔案或目錄（可以有多個），並就地把它們改寫為[標準格式](#標準格式)。屬於套件的目錄（[9.2.2](902-aot-pipeline.md#922-套件探索)）會以其 `src/`、`tests/`、`benches/` 與 `examples/` 目錄以及其 `nabi.nabion` 進行格式化；其中的其他檔案不會被動到。沒有路徑時，`fmt` 格式化目前目錄所在的套件：它所在或其上方最近一個含有 `src/main.nabi`、`src/lib.nabi` 或 `nabi.nabion` 的目錄。在套件之外，沒有路徑是錯誤（結束碼 `1`）：`` not inside a package; pass a file or directory: `nabi fmt <path>` ``。目錄會被遞迴搜尋 `.nabi` 與 `.nabion` 檔案，略過 `target/`、`node_modules/`、以 `.` 開頭的目錄，以及被搜尋目錄或其套件根部之 `.gitignore` 所忽略的東西。只讀取下列 `.gitignore` 行：`name`（其下任何位置的該名稱）、`name/`（僅限目錄）、`/name`（直接位於 `.gitignore` 所在目錄中）以及 `*suffix`；註解、`!` 行、其他萬用字元以及內部帶有 `/` 的路徑會被略過。在終端機上，耗時超過一秒的搜尋會在標準錯誤顯示 `Scanning <n> files…`。`.nabion` 檔案（[NABION](../../refs-standard-libraries/300-std/308-nabion.md#384-正規形式)）會被檢查是否為 NABION，並且只重新縮排（它們不會被改寫為標準形式，因此註解會被保留）。

在結尾，除非有 `-q`，標準錯誤會說明發生了什麼：`Formatted <n> files`（若有些已經格式化，加上 `, <m> already formatted`）、`note: <n> files already formatted`，或在 `--check` / `--diff` 時為 `` note: <n> files need formatting; `nabi fmt` without `--check` rewrites them ``。沒有東西可以格式化的目錄會給出 `note: no .nabi or .nabion files under <dir>`（結束碼 `0`）。

選項：

- `--check` 不修改檔案；它印出會改變的檔案及其差異。若有任何檔案會改變，結束碼是 `1`；適合 CI 使用。
- `--diff` 不修改檔案，只印出差異。

有詞法錯誤的檔案不會被動到，並會回報該錯誤，結束碼為 `1`。

#### 標準格式

`nabi fmt` 的標準格式是一種**保留語彙單元的版面規則**。它不會改變一行內的語彙單元或註解；它設定各行的縮排與空行，並排列模組頂部的 `mod` / `use` 項目。`nabi fmt` 刻意沒有任何設定：所有 Nabi 程式碼只有一種標準格式。

1. 縮排是每個括號（`(` `[` `{` `#[`）深度 4 個空白。以關閉括號開頭的行取得開啟它之那一行的深度。Tab 變成空白。
2. 延續行，即其前一個語彙單元不是 `;` `,` 或括號的行（例如拆分長運算式的行），保留它在原始檔案中超出區塊縮排的縮排量。
3. 移除結尾的空白。
4. 連續的空行壓縮為一行，並移除檔案開頭的空行。
5. 檔案恰好以一個換行結尾。
6. 檔案頂部以及行內 `mod name { }` 頂部（在其第一個其他項目之前）的 `mod name;` 與 `use` 項目，依 `pub mod`、`mod`、`pub use`、`use` 的順序分組，其中任何 `pub(..)` 都算作 `pub`。每組都會排序（`mod` 依名稱，`use` 依路徑逐段排序，`self`、`sup` 與 `pkg` 在前），各組之間以一個空行分隔，組內沒有空行。緊接在項目上方的註解、文件註解與屬性會隨它移動；第一個項目上方的註解留在原處。與另一個項目共用一行的項目會結束頂部區域。

除了規則 6 之外，格式化工具不會移動行、不會拆分或合併行，也不會改變一行內的間距。因此註解會完全保持寫出的樣子，而除了規則 6 所排序之項目的順序之外，語彙單元序列也不會改變。多行字串與區塊註解的內部也保持原樣。

這個版面規則在 1.0 之前一直是標準格式：在那之前，不會有重新格式化程式碼（移動語彙單元、拆分或合併行）的 pretty-printer 取代它，因此 `nabi fmt --check` 所接受的檔案在之後的 0.x 工具鏈上也會持續通過。

### `nabi doc`：產生文件

```
nabi doc [path] [--std] [--open]
```

從套件之函式庫（沒有函式庫時為其執行檔）及其相依套件的公開項目與文件（`///`、`//!`、`#[doc]`，[6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---文件註解形式)），在 `target/doc/` 中產生 HTML API 文件。

```
target/doc/
├── <pkg>/index.html           # a package's root module
├── <pkg>/<mod>.html           # one page per public module
├── search-index.js            # search index
├── search.js, keys.js         # search, and its `/` shortcut
├── theme.js                   # light / dark theme
├── nabi.css, style.css        # style sheets
└── fonts/, favicon.svg
```

`<pkg>` 是套件的名稱（資訊清單的 `name`）；標準函式庫的層級就是它自己的名稱（`core`、`std`、`bare`）。當模組路徑上的每個模組都是 `pub` 且不是 `#[doc(hidden)]` 時，該模組就是公開的；每個公開模組有一個頁面：根的頁面是 `<pkg>/index.html`，`a::b` 的頁面是 `<pkg>/a/b.html`（根的名為 `index` 的子模組是 `<pkg>/index/index.html`）。套件的根頁面之上沒有頁面：每個頁面的標頭顯示其套件的名稱（連結到該套件的根）與版本，其側邊欄列出套件的相依套件（連結到它們的根頁面），並與其麵包屑一起通往它自己套件的根，而搜尋從任何頁面都涵蓋每個有文件的套件。較早的工具鏈留下的 `target/doc/index.html` 會被移除；`target/doc/` 中的其他檔案都不會。頁面不需要網路：樣式表、腳本與字型都在輸出中。

模組的頁面顯示其文件（其 `//!` 行，接在其 `mod` 項目上的 `///` 之後；一般的 `//` 註解不是文件註解，而 `nabi lint` 會對開啟函式庫或公開模組之檔案的這種註解發出警告，[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-控制)）、其公開的子模組，然後依種類顯示其項目：重新匯出、結構（其中包括 union）、列舉、trait、型別別名、函式、常數、static，最後是在別處宣告之型別的實作（`impl str { .. }`）。項目是不是 `#[doc(hidden)]` 或 `#[internal]` 的 `pub` 項目，每個都帶有其寫出的簽章、其文件、其成員（公開欄位、變體、trait 項目、其固有 impl 的公開方法與常數）以及其 trait 實作。頁面顯示什麼由下列規則決定：

- `pub use` 從私有模組，或只經由私有模組引入的項目，會在重新匯出的頁面上像在那裡宣告一樣顯示，使用它被重新匯出時的名稱，並先顯示 `pub use` 的文件。`pub use private::*` 的每個項目也是如此。
- 對某個公開模組頁面所顯示之項目的重新匯出，保持為一個重新匯出項目，連結到該頁面。
- 在私有模組中寫的 `impl` 的方法，會在其上方最近之公開模組的頁面上加入其型別，否則加入該頁面的實作。
- 每個項目都有錨點 `#<kind>.<name>`，其中 `<kind>` 是 `struct`（union 也是）、`enum`、`trait`、`type`、`fn`、`const` 或 `static`（`app/index.html#struct.Point`）；成員為 `#<Type>.<member>`（`#Point.x`、`#Shape.Circle`、`#Point.origin`）；實作為 `#impl.<Type>`；重新匯出為 `#reexport.<names>`，即它所引入的名稱以 `.` 串接，而其中的每個名稱是它所指名之項目的 `#<kind>.<name>`，因此 `#struct.Event` 會在重新匯出 `Event` 的頁面上找到它。
- 簽章中的型別，以及行內程式碼中指名有文件之項目的名稱，會連結到顯示它的頁面，無論位於套件中或相依套件中；沒有任何頁面顯示的名稱保持為純文字。每個連結都指向存在的頁面與錨點。

docs.nabi-lang.org 上的標準函式庫 API 就是依這些規則產生的：每個模組相同的頁面（那裡是 `std/core/io.html`，這裡是 `core/io.html`），帶有相同的錨點，而 `nabi doc` 的頁面共用其樣式表與版面。

`--std` 也會把標準函式庫各層級的文件產生到 `core/`、`std/`、`bare/` 中；沒有路徑時，只產生它們的文件，放到 `./target/doc/`。

`nabi doc` 印出套件的根頁面（`Generated ./target/doc/<pkg>/index.html`）；有 `--std` 而沒有路徑時是 `std/index.html`。接著 `--open` 會以設定了的 `$BROWSER` 開啟該頁面，否則以系統的開啟程式（macOS 上的 `open`、Linux 上的 `xdg-open`、Windows 上的 `start`）開啟。若無法啟動，會有一則警告指出要手動開啟的檔案；結束碼不受影響。

文件文字被解讀為 markdown 的一個子集（標題、圍欄程式碼區塊、行內程式碼、連結、`**bold**`、清單、段落）。在 Nabi 程式碼區塊中，以 `# `（或單獨的 `#`）開頭的行不會顯示：它是該區塊之文件測試（[9.1.2](901-nabic-cli.md#文件測試)）的準備工作，由 `nabi test` 執行。

### `nabi clean`：移除建置輸出

```
nabi clean [path]
```

刪除套件的整個 `target/` 目錄（包括所有設定檔與交叉建置；有設定時為 `--target-dir` / `NABI_TARGET_DIR` 目錄）。不存在時不做任何事。

### `nabi bindgen`：C 標頭檔繫結

```
nabi bindgen <header> [-x objective-c] [-o <file>] [--target <t>] [--c-shim <file.c>] [-- <C compiler args>]
```

把 `<header>` 的 [std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#176-自動產生繫結) 宣告寫到標準輸出（或 `-o <file>`）；其第一行以 ``// Generated by `nabi bindgen` `` 開頭。`nabi` 以工具鏈之 C 編譯器（[9.13](913-c-compiler.md)）的前端讀取標頭檔；它不需要其他 C 編譯器，也不需要 libclang。`--` 之後的引數是會改變標頭檔所宣告內容的 C 編譯器選項：`-I <dir>`、`-isystem <dir>`、`-iquote <dir>`、`-idirafter <dir>`、`-D <name>[=<value>]`、`-U <name>`、`-include <file>`、`--sysroot <dir>`、`-isysroot <dir>`、`-nostdinc`、`-nostdlibinc` 與 `-nobuiltininc`。`-std=` 與警告選項會被接受並忽略；其他任何選項都是錯誤，錯誤訊息會列出這些選項。前端拒絕的標頭檔是位於其 `file:line:col` 的錯誤（結束碼 1），錯誤訊息會說明應改寫成什麼（[std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#bindgen-讀取的-c) 列出它所讀取的內容）。

使用 `-x objective-c` 時（Apple 目標），標頭檔以 Objective-C 讀取，其類別、協定與 category 會與 `ffi::objc::import` 的做法一樣變成 `extern "objc"` 區塊（[std 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-標頭檔ffiobjcimport)）；`-x c` 是預設值。

`static inline` 函式沒有可以連結的符號，因此會被略過並附上註解。使用 `--c-shim <file.c>` 時，它則會被繫結：`<file.c>` 為每個這類函式取得一個 C 包裝函式，即位於 `#include "<header file name>"` 之後的 `R nabi_shim_<f>(A a0, ..) { return <f>(a0, ..); }`，而繫結連結到該包裝函式（`#[link_name = "nabi_shim_<f>"]`）。套件從其資訊清單的 `c.sources` 編譯 `<file.c>`，並把標頭檔的目錄放在 `c.include` 中（9.10.3）。可變參數的 inline 函式仍會被略過。

### `nabi inspect`：檢視套件

`nabi inspect` 提供封存檔內容的概觀。

```
nabi inspect path/to/libfoo.a        # or foo.nabihir
nabi inspect --dump path/to/foo.nabihir   # or libfoo.a; reads the sibling foo.nabihir
```

它顯示套件名稱與雜湊、nabic 版本、NABIHIR 版本、GC 型別 id 範圍、匯出、[泛型](../000-meta/007-glossary.md#泛型generic)項目，以及封存檔成員及其大小（[9.9.14](909-debug-tools.md#9914-檢視套件)）。

有 `--dump` 時，它則以文字印出預先建置之函式庫的 NABIHIR：標頭欄位、產生者、meta 與介面檔案；格式說明見 [NABIHIR 格式](904-nabihir-bytecode.md)。

### `nabi demangle`：符號解修飾

`nabi demangle` 把修飾後的符號解碼為人類易讀的形式。

```
nabi demangle [-v] <symbol>...
nabi demangle
```

對每個符號，它依原始碼的寫法每行印出一個名稱；有 `-v` 時，印出 [9.5.4](905-symbol-mangling.md) 格式的區塊，以空行分隔（[9.9.3](909-debug-tools.md#993-符號解修飾)）。若有任何引數不是 Nabi 符號，結束碼是 `1`。沒有符號時，它把標準輸入過濾到標準輸出，把文字中的每個 Nabi 符號都換成其名稱（`perf report --stdio | nabi demangle`）。

### `nabi completions`：shell 補全

```
nabi completions bash|zsh|fish|powershell
```

在標準輸出上印出該 shell 的補全腳本。它從與 `nabi help` 相同的指令與選項文字產生（對委派的子指令則還有 `nabic` 的），因此兩者一致。它補全子指令及其選項、以已安裝的工具鏈補全 `+<version>` 以及 `toolchain default` / `toolchain remove` 的引數、補全 `env use` / `remove` / `shell` / `export` 與 `--env` 的環境名稱，並以 `nabic --print target-list` 所印出的名稱補全 `--target` 的值。這些名稱在補全時藉由執行 `nabi __complete toolchains` / `envs`（每行一個）讀取，因此它們跟隨 `$NABI_HOME`。其他任何 shell，或沒有 shell，是用法錯誤（結束碼 2）。`install.sh` 與 `install.ps1` 會安裝這些腳本，而 `nabi toolchain default` / `update` 會改寫它們（9.10.6）。
