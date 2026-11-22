# 9.8. Jupyter 核心整合

本節定義 [Jupyter 介面](../000-meta/001-execution-modes.md)的 Nabi 核心。

## 9.8.1. 核心

Jupyter 以另一個行程執行 Nabi 核心，並經由以 ZeroMQ 為基礎的 Jupyter 線路協定通訊。核心宣告的 `protocol_version` 為 `"5.3"`（`jupyter_client` 所期望的值；可搭配 5.3 以上的用戶端運作）。在這種形式下，所有作業系統資源，即檔案、網路與外部行程，都可以使用。

核心在與終端機 REPL **相同的儲存格工作階段**上執行，並遵循 [9.7.0](907-jit-cells.md#970-儲存格工作階段的規則) 的規則：每個儲存格恰好執行一次，繫結與項目延續到之後的儲存格，失敗的儲存格不會在繫結上留下痕跡。核心的引擎與 REPL 是同一個 [JIT 工作階段](906-jit-engine.md)：頂層的 `await` 與 `it` 的運作方式與那裡相同（[9.7.1](907-jit-cells.md#971-儲存格的種類)、[9.7.7](907-jit-cells.md#977-視覺化--輸出)）。

### 輸入

儲存格經由用戶端讀取 stdin。當 `execute_request` 帶有 `allow_stdin: true` 時，執行環境對 stdin 的每次讀取都會在 stdin 頻道上向該請求的用戶端傳送一個 `input_request`（`prompt` 為空，`password` 為 false）；儲存格在讀取之前所寫的東西，例如它印出的提示，已先以 `stream` 傳送。`input_reply` 的 `value` 加上一個換行，就是讀取所回傳的內容。沒有 `allow_stdin`，或對於 `silent` 請求，讀取就是輸入結束。儲存格等待回覆時發生的中斷會結束該儲存格（9.8.4）。

### 筆記本儲存格與 REPL 儲存格

筆記本儲存格依終端機 REPL 拆分輸入的相同規則拆成多個 REPL 儲存格，並依序執行。與 IPython 一樣，**只有最後一個值**會作為結果顯示。若筆記本儲存格較後面的部分失敗，較前面部分（先前的 REPL 儲存格）的效果仍會保留。

REPL 儲存格在執行時位置（`<repl cell N>`，[9.7.5](907-jit-cells.md#位置)）中的編號取自執行計數：只是一個 REPL 儲存格的筆記本儲存格是 `N`，拆成多個的則把它們編號為 `N.1`、`N.2` 等。不推進計數的請求（`silent`，或 `store_history: false`）把其 REPL 儲存格接在目前計數下最後一個編號之後編號。

## 9.8.2. 本機原生核心：運作

當使用者選擇「Nabi」作為筆記本的核心時，Jupyter 以另一個行程執行 kernelspec（9.8.3）的指令。

```
nabi jupyter --connection-file <path>        # resolves dependencies; also accepts build options such as --features
```

核心從連線檔讀取 ZMQ 連接埠與金鑰，然後在其五個通訊端上繫結並監聽。五個都接受連線之後，它在開始擷取輸出之前，把一行 `nabi jupyter: ready` 寫到其 stderr。只支援 `tcp` 傳輸（不支援 `ipc`），也不使用 CURVE 加密。訊息以連線檔的金鑰進行 HMAC 簽章。

核心與 `nabi repl .` 一樣（[9.7.10](907-jit-cells.md#9710-載入程式repl-path)），載入它所啟動之處的套件（目前目錄，或其上方最近一個含有 `src/main.nabi` 或 `src/lib.nabi` 的目錄）：函式庫套件是一個相依套件，並以 `use <name>::*;` 作為第一個儲存格；程式套件則會被載入並執行。這在開始擷取輸出之後發生；它印出的東西會與第一個執行之儲存格的輸出一起傳送。在套件之外，工作階段以空的狀態開始。

### 通訊端頻道

| 頻道        | 用途                                                       |
|-------------|------------------------------------------------------------|
| `shell`     | 請求（execute、kernel_info、complete、is_complete 等）     |
| `iopub`     | 輸出與狀態廣播（status、stream、execute_result 等）        |
| `stdin`     | 供讀取 stdin 之儲存格使用的 `input_request` / `input_reply`（9.8.1） |
| `control`   | interrupt、shutdown                                        |
| `heartbeat` | 把收到的任何東西傳回去（echo）                             |

### 訊息格式

每則訊息序列化為六個以上的 ZMQ 框。

```
[b"u-u-i-d",         # zmq identity(ies)
 b"<IDS|MSG>",       # delimiter
 b"baddad42",         # HMAC signature (with key)
 b"{header}",          # header JSON
 b"{parent_header}",   # parent header JSON
 b"{metadata}",        # metadata JSON
 b"{content}",         # content JSON
]
```

詳細格式遵循 Jupyter 線路協定標準。每個請求都包在 iopub 的 `status: busy` 與 `status: idle` 之間。

### 支援的訊息

| 請求                   | 回覆                                                                                   |
|------------------------|----------------------------------------------------------------------------------------|
| `kernel_info_request`  | `language_info`，其 name 為 `nabi`、mimetype 為 `text/x-nabi`、file_extension 為 `.nabi`、codemirror_mode 與 pygments_lexer 為 `rust`（在前端認識 Nabi 之前用於語法標示）；`help_links` 指向文件（`https://docs.nabi-lang.org/latest/`）與本 REPL 章節；`banner` 為 REPL 的工具鏈行（[9.7.8](907-jit-cells.md#行編輯)）；`protocol_version` 為 `"5.3"` |
| `execute_request`      | 見下文的「執行」                                                                       |
| `is_complete_request`  | 依 REPL 的延續規則決定的 `complete` / `incomplete`；`indent` 是每個仍未關閉的括號四個空白 |
| `complete_request`     | REPL 的補全候選項（9.8.5）                                                             |
| `inspect_request`      | 對游標處的名稱回覆 `found: true` 並附上 `text/plain` 說明（9.8.6），否則為 `false`     |
| `history_request`      | 這個核心所執行過的儲存格                                                               |
| `comm_info_request`    | 空清單                                                                                 |
| `shutdown_request`     | 在 shell 或 control 上。回覆之後結束                                                   |
| `interrupt_request`    | 在 control 上（9.8.4）                                                                 |

### 執行

收到 `execute_request` 時，核心傳送：

1. iopub `execute_input`（程式碼與執行計數）。
2. 儲存格的輸出，**在寫出時即傳送**：執行中的儲存格寫到 stdout / stderr 的東西，在核心讀取到時就以 iopub `stream`（`stdout` / `stderr`）傳送。兩者是分開的管道，因此其中一者的各區塊保持其順序，而兩者的區塊依讀取的順序到達。在執行之外寫出的輸出（診斷、`:` 指令）在下一次執行之前或儲存格結束時傳送。
3. 若有值，iopub `execute_result`：`text/plain` 容納與 REPL 相同的輸出（9.7.7）。其型別實作了 `RichDisplay`（[stdlib 2.10](../../refs-standard-libraries/200-core/210-fmt.md)）的值改為經由 `display` 顯示：iopub `display_data`，其 `data` 是它的 `mime_bundle()`（`image/svg+xml`、`image/png`、`text/html`……），且沒有 `execute_result`。
4. 失敗時，iopub `error`：`evalue` 是尚未傳送之 stderr 的第一行（無法編譯之儲存格的診斷，或執行如何結束的報告），`traceback` 是其各行。儲存格本身寫到 stderr 的東西，例如 panic 訊息，已先以串流傳送。
5. shell `execute_reply`（包括 `execution_count`，以及成功 / 失敗狀態）。

輸出的 MIME 種類是 `text/plain` 以及 `RichDisplay` bundle 中的那些。`display(x)` 在儲存格中途傳送 `display_data`，位於儲存格串流輸出中它所在的位置：它把一個加框的 bundle 寫到 stdout（核心為它所執行的儲存格設定 `NABI_JUPYTER`），由核心從文字中取出。不支援就地更新顯示（`update_display_data`）。

## 9.8.3. 安裝 kernelspec

Jupyter 只辨識安裝在 kernelspec 目錄中的核心。

```
nabi jupyter install [--env <name>]
```

不帶引數的 `nabi jupyter` 是用法錯誤（結束碼 `2`），錯誤訊息會說明改為執行什麼：`` Jupyter starts the kernel itself; to add it, run `nabi jupyter install` ``。

kernelspec 是下列目錄之下的 `kernels/nabi/kernel.json`。以 `--env <name>` 安裝時，它是 `kernels/nabi-<name>/`，顯示名稱為 `nabi (<name>)`，指令為 `nabi --env <name> jupyter ...`。

1. 有設定時為 `$JUPYTER_DATA_DIR`
2. 在 macOS 上為 `~/Library/Jupyter`
3. 在 Windows 上為 `%APPDATA%\jupyter`
4. 否則為 `$XDG_DATA_HOME/jupyter`，未設定時為 `~/.local/share/jupyter`

它旁邊是 `logo-32x32.png`、`logo-64x64.png` 與 `logo-svg.svg`，即前端為該核心顯示的 Nabi 標誌。`kernel.json` 執行 `nabi` 代理程式（[9.10.6](910-nabi-frontend.md#9106-toolchain)），並把安裝時的 `NABI_HOME` 與 `PATH` 記錄在 `env` 中。

```json
{
  "argv": ["/home/u/.nabi/bin/nabi", "jupyter", "--connection-file", "{connection_file}"],
  "display_name": "nabi",
  "language": "nabi",
  "interrupt_mode": "message",
  "env": { "NABI_HOME": "/home/u/.nabi", "PATH": "<PATH at install time>" }
}
```

安裝之後，Nabi 核心就可以從 Jupyter、VS Code、nteract 等使用。

## 9.8.4. 中斷 / 關閉

### 中斷

由於 `interrupt_mode` 是 `"message"`，中斷以 control 通訊端上的 `interrupt_request` 到達。執行中的儲存格會被終止，工作階段從該儲存格之前的狀態繼續（與失敗的儲存格相同，[9.7.0](907-jit-cells.md#970-儲存格工作階段的規則)）。等待 `input_reply` 的儲存格也會結束。仍在編譯中的儲存格無法被中斷。

### 關閉（重新啟動核心）

在 shell 或 control 通訊端上回覆 `shutdown_request` 之後，核心清理工作階段並結束行程。重新啟動時，Jupyter 會自動再次啟動它。

## 9.8.5. 程式碼補全

`complete_request` 把游標前的片段與終端機 REPL 之 Tab 補全的候選項比對（[9.7.8](907-jit-cells.md#行編輯)）：工作階段的繫結與項目、core 函式庫的名稱、工作階段之 `use` 項目所匯入的名稱，以及關鍵字；`.` 之後的成員與 `::` 之後的項目。它以 `{ matches, cursor_start, cursor_end }` 回覆。

## 9.8.6. inspect_request：說明

`inspect_request`（Shift+Tab）以 `found: true` 回覆，並以 `text/plain` 提供游標處的名稱，寫成 `name: type`，以及其簽章與文件註解（[語彙結構](../001-lexical/102-lexical-structure.md)的 `///` / `//!`）。對未知的名稱則是 `found: false`。

## 9.8.7. REPL 與 Jupyter 之間的程式碼共用

在 REPL 中寫的程式碼可以原封不動地移到 Jupyter 儲存格，反之亦然，因為兩者遵循相同的儲存格工作階段與相同的語意（[9.7.0](907-jit-cells.md#970-儲存格工作階段的規則)）。

### 差異

兩種介面在少數幾個方面不同，摘要如下。

| 面向        | REPL              | Jupyter                      |
|-------------|-------------------|------------------------------|
| 輸入        | 一行 / 一個運算式 | 一個儲存格（多行）           |
| 輸出        | 只有文字（`display(x)` 印出其 `text/plain`） | 文字，以及 `RichDisplay` 的各種 MIME 內容 |
| stdin       | 終端機；經由管道時為空 | 帶有 `allow_stdin` 的 `input_request` |
| 狀態        | 結束時捨棄        | 儲存格保留在筆記本檔案中     |

## 9.8.8. 安全性 / 隔離

核心以使用者的權限執行任意程式碼。不進行任何隔離。

## 9.8.9. 接下來

接下來的[偵錯 / 診斷工具](909-debug-tools.md)定義對 gdb 友善的功能、效能分析、`--emit` 等。
