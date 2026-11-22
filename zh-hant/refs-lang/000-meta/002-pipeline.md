# 0.2. 編譯管線

本規格把 Nabi 編譯器稱為 `nabic`。`nabic` 在同一個行程內執行從原始碼到連結完成之二進位檔的整條管線：它以自己的程式碼產生器產生機器碼，並以自己的連結器連結。

## 0.2.1. 階段

整體流程如下。

```
  source code
        |
        v
   nabi-hir         (High-level IR: pre type inference, user code structure preserved)
        |
        v
   nabi-mir         (Mid-level IR: monomorphized / operators desugared, all types fixed)
        |
        v
   machine code     (Nabi's code generator: optimized at -O1 and above)
        |
        v
   object -> linked by nabic's linker
```

> nabi-hir 是高階 IR；它位於型別推論之前，原樣保留使用者程式碼的結構。nabi-mir 是中階 IR，其中單態化與運算子去糖已經完成，所有型別都已確定。
> 程式碼產生器把 nabi-mir 轉成目標的機器碼與目的檔；連結器再從這些目的檔產生執行檔。

## 0.2.2. 單向可見性

每個階段只看緊接在它之前的階段。`nabi-hir` 產生器只參照原始碼。`nabi-mir` 產生器只參照 `nabi-hir`。程式碼產生器只參照 `nabi-mir`。

因此，較早階段的結果絕不會再被較晚的階段查閱。規格文件也遵循相同的規則：編號較小的檔案不參照編號較大的檔案。

## 0.2.3. 自給自足的工具鏈

`nabic` 不執行任何外部編譯器、組譯器或連結器，也不需要在它旁邊安裝任何工具。唯一需要主動選用的例外是 `NABI_CC`，它指定一個 C 編譯器驅動程式來代替 `nabic` 的連結器進行最終連結（[9.1](../009-tools/901-nabic-cli.md)）。每一種中間形式都是 Nabi 自己的：nabi-hir、nabi-mir，以及位於 nabi-mir 之下、程式碼產生器的低階形式。程式碼產生器與連結器只為 Nabi 的執行環境 ABI（[8](../008-abi/README.md)）及其目的檔格式而寫。

因此，把 Nabi 的高階語意降階為機器運算的責任在編譯器，具體而言在程式碼產生階段（[8.7](../008-abi/807-code-generation.md)）。

## 0.2.4. 程式碼產生

`nabic` 只有一個程式碼產生器，用於每一個最佳化等級以及 REPL 儲存格。它讀取 nabi-mir（在 `-O0` 以上會先最佳化），寫出遵循執行環境 ABI（[8](../008-abi/README.md)）的 Mach-O、ELF、COFF 或 wasm 目的檔，因此以不同等級建置的目的檔可以連結在一起。

連結器也是 `nabic` 的一部分：Mach-O 執行檔（附帶 ad-hoc 程式碼簽章，以及存放偵錯資訊的 `.dSYM` 套組）、ELF 執行檔、COFF / PE 執行檔（附帶 `nabic` 產生的匯入函式庫，偵錯資訊則為 PDB），以及 wasm 模組，`wasm32-wasip2` 建置會再把它轉成 WASI preview 2 元件。交叉編譯是內建的：`--target <triple>` 在任何主機上都能使用，工具鏈也為其支援的 Linux 目標附帶 C 函式庫（musl、glibc 連結用 stub），因此為 Linux 目標建置 Nabi 程式碼不需要 sysroot。C 程式碼由工具鏈的 C 編譯器 `nabicc` 以相同的程式碼產生器與連結器編譯；它讀取工具鏈為目標附帶的 C 標頭檔，或系統的標頭檔（[9.13](../009-tools/913-c-compiler.md)）。Windows 目標在 UCRT 之上，連結從工具鏈所帶的模組定義建出的匯入函式庫，不需要 MSVC 工具或 Windows SDK。Apple 目標需要 macOS SDK（Xcode Command Line Tools），它不可再散布（[9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables)、[0.6](006-platform-targets.md)）。`--emit=mir` 傾印最佳化後的 nabi-mir，`--emit=obj` 傾印目的檔（[9.9.1](../009-tools/909-debug-tools.md#991-intermediate-output---emit)）。
