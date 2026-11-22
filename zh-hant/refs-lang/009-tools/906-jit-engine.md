# 9.6. JIT 引擎

本節定義 REPL 與 Jupyter 所使用的 JIT 執行引擎。它是[執行引擎與使用者介面](../000-meta/001-execution-modes.md)中定義之 JIT 引擎的實作。儲存格工作階段的規則由 [9.7.0](907-jit-cells.md#970-rules-of-the-cell-session) 定義。

## 9.6.1. 結構：nabic 與 JIT 主機

工作階段在另一個**主機行程**中執行，它包含執行環境，並在工作階段期間持續存在。主機以 `nabic` 預先建置，並由 `nabic` 的連結器連結；它自己載入儲存格的目的檔。

```
nabic (front end)                         JIT host (one per session)
  cell → parse → HIR → MIR                   all of libnabir.a
     → code generator                        runtime initialized once
     → machine code + relocations ───────►   load into executable memory, resolve symbols
                                             → run the cell entry function once
```

- 儲存格是程式碼產生器（[9.2.11](902-aot-pipeline.md#9211-code-generator)）以 `:opt` 所設定之等級產生、由主機對應並連結的目的檔；其 nabi-mir 不會被最佳化，因為之後的儲存格可能重新定義它會內嵌的東西（[9.2.10](902-aot-pipeline.md#9210-optimization-levels)）。REPL 在規格表明支援它的目標上執行（[0.6.5](../000-meta/006-platform-targets.md#065-target-specifications)，`repl`）：`x86_64` 與 `aarch64` 的 Linux（glibc）、macOS 與 Windows。
- 主機把程式碼放進可執行記憶體，對照執行環境、libc 與先前的儲存格解析其重定位，並執行儲存格的進入函式一次。
- 主機匯出執行環境（`libnabir.a`）的每個符號，儲存格程式碼在主機行程中找到執行環境與 libc 的符號。

## 9.6.2. 程式碼載入

儲存格是一個目的檔。其程式碼與位置無關，帶有重定位。主機為工作階段保有一個符號表；儲存格的定義在儲存格載入之後加入其中。無法解析的符號會讓儲存格失敗，錯誤訊息會指出它（[9.7](907-jit-cells.md)）。

## 9.6.3. 符號與重新定義

工作階段使用一個符號表。所有儲存格的定義都累積在那裡。

- 當儲存格重新定義一個已定義的符號時，主機把該定義重新命名為 `<name>.r<cell>`。之後儲存格中的參照連結到最新的名稱。
- 重新定義從下一個儲存格起生效（[9.7.0](907-jit-cells.md#970-rules-of-the-cell-session) 規則 3）。
- 相依於被重新定義之符號的函式由 `nabic` 重新編譯。因此，即使是在重新定義之前編譯的程式碼，當之後的儲存格再次到達它時，也會使用新的定義（呼叫被重新定義之 `f` 的 `g`）。
- 儲存格程式碼與舊定義在工作階段結束之前都不會被釋放。

## 9.6.4. 重複使用函式庫程式碼

標準函式庫與相依套件的程式碼在每個工作階段中只編譯一次。

- `nabic` 記住主機已經載入的函式及其 MIR。若儲存格的函式與某個已載入的函式相同，就只傳遞參照而不傳遞程式碼。
- 跨儲存格的名稱是固定的，因此相同的 MIR 表示相同的意義。字串字面值的全域變數以其內容的雜湊命名，GC 型別 id 則以 [ScanTable](../008-abi/802-scan-table.md) 的內容為鍵，在工作階段中固定。
- 直接或間接呼叫這個儲存格新定義之東西（改變的函式、`static`、繫結槽位）的函式會被重新編譯。
- 儲存格的目的檔只容納該儲存格新增的東西：它新定義的函式，以及對於其程式碼所讀取之先前儲存格的 static，它們的初始化防護。它只指名其程式碼所參照的符號。

## 9.6.5. 執行流程（一行 REPL / 一個 Jupyter 儲存格）

```
1. nabic: input → parse → HIR (on error the cell is discarded; session state unchanged)
2. nabic: cell → MIR → code generator → machine code
         (already loaded functions as references, redefinitions under new names)
3. host: load the code in a process forked from the session snapshot and run the cell once
4. success: that process becomes the new snapshot. If there is a value, print the value (`Display`)
   failure (panic, exit, signal, load failure): the previous snapshot takes over (9.7.0 rule 5)
```

步驟 3 在 Unix 上會 fork。沒有 fork 的主機（Windows）在儲存格之前儲存工作階段的記憶體與執行緒，並在它失敗時把它們放回去，可觀察到的結果與 fork 出的快照相同。

一個儲存格的延遲大約是 0.1 秒，其中大部分花在前端（參考資訊）。

## 9.6.6. JIT 與 AOT 的差異

| 面向        | JIT（REPL / Jupyter）                        | AOT                               |
|-------------|----------------------------------------------|-----------------------------------|
| 管線        | 到 nabi-mir 為止相同                          | 相同                              |
| 後端        | Nabi 的程式碼產生器，載入主機                | Nabi 的程式碼產生器（`-O0` 以上會最佳化），然後是 `nabic` 的連結器 |
| 單位        | 儲存格（函式庫每個工作階段一次）             | 套件                              |
| 結果        | 主機記憶體                                   | 執行檔 / `.a`                     |

兩個引擎共用相同的單態化、nabi-mir 與程式碼產生器，並遵循相同的 ABI，因此其語意相同。

## 9.6.7. 接下來：儲存格的執行 / 狀態 / 錯誤

[JIT 儲存格的執行流程與狀態管理](907-jit-cells.md)定義各儲存格的行為（頂層變數、重新定義、錯誤）。
