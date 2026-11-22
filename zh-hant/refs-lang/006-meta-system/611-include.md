# 6.11. 編譯期檔案嵌入

## 6.11.1. `include_str!` / `include_bytes!`

```nabi
const TEMPLATE: str = include_str!("templates/page.html");
static ICON: [u8; 1406] = include_bytes!("res/icon.png");
```

`include_str!("path")` 求值為檔案 `path` 的內容，型別為 `str`；`include_bytes!("path")` 求值為其位元組，型別為 `[u8; N]`（`N` 是檔案的長度；一種位元組字串，[1.7.5](../001-lexical/107-string-literals.md#175-byte-strings)，包括其 `Slice<u8>` 規則）。引數恰好是一個字串字面值。路徑相對於包含該呼叫之原始檔所在的目錄。檔案在編譯期讀取；結果是常數，可以出現在任何可以出現字面值之處，包括 `const` 與 `static` 初始化式。

若無法讀取該檔案，或對 `include_str!` 而言其內容不是有效的 UTF-8，就是編譯錯誤。被嵌入的檔案是建置的輸入：改變它會重新建置使用它的部分（[9.2.16](../009-tools/902-aot-pipeline.md#9216-incremental-build)）。

與 `cfg!`（[6.3.5](603-cfg.md#635-cfg-compile-time-bool-value)）相同，它們是編譯器的內建形式，而不是使用者定義的巨集。
