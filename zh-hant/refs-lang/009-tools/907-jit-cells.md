# 9.7. JIT 儲存格的執行流程與狀態管理

本節詳細定義 [JIT 引擎](906-jit-engine.md)如何處理使用者輸入（儲存格）。

## 9.7.0. 儲存格工作階段的規則

儲存格工作階段遵守下列規則。其實作是 [9.6 JIT 引擎](906-jit-engine.md)。

1. **只執行一次。** 每個儲存格的程式碼恰好執行一次。先前儲存格的副作用（I/O、時間、亂數）不會重複。
2. **狀態會保留。** 儲存格的頂層 `let` 繫結、`static` 的狀態以及 `any` 值（包括 `any Tr`）會延續到之後的儲存格。繫結可以被指派，而同名的新 `let` 會以新的繫結遮蔽它（9.7.3）。
3. **項目會保留，並可以重新定義。** `fn` / `struct` / `impl` 等項目會保留。若同一個項目再次被定義，從下一個儲存格起使用新的定義。即使是在重新定義之前編譯的程式碼，當之後的儲存格再次到達它時，也會使用新的定義（呼叫被重新定義之 `f` 的 `g` 會呼叫新的 `f`）。重新定義 `static` 會重新開始其狀態。其型別被重新定義的繫結不能再使用；嘗試使用它會被回報，並附上說明。一個儲存格可以定義多個項目。新的 `impl Tr for T` 取代舊的；新的固有 `impl T` 取代先前定義了其任一方法的固有 `impl`。會讓先前的項目無法再編譯的重新定義會被拒絕：錯誤指向該項目內部，指出它及其儲存格，並說明要在同一個儲存格中重新定義它，或使用 `:reset`。
4. **`any` 的同一性。** `any` 的型別標籤在工作階段的所有儲存格中都相同。在先前儲存格中存入的 `any` 值，可以在之後的儲存格中以 `is` / 模式取出。
5. **失敗的儲存格。** panic、終止行程或死亡的儲存格不會在繫結上留下任何部分效果。工作階段從該儲存格之前的狀態繼續。已經發生的外部效果（I/O 等）不會被撤銷。先前儲存格的執行緒：失敗的儲存格之後，工作階段只保留回到原處的執行緒（Windows），或一個都不保留（fork 主機）。執行環境會重新啟動它自己的執行器執行緒；join 一個已消失的使用者執行緒會回報它已遺失（`JoinErrorKind::Lost`，[stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)）。工作階段在靜止的時刻儲存：沒有任務正在輪詢中，也沒有阻塞工作正在執行中（主機在儲存格之前等待它們；中斷這個等待會取消該儲存格），因此任務與工作絕不會遺失。已消失之執行緒在阻塞時持有的鎖會被釋放。
6. **顯示。** 運算式儲存格以 `Display` 印出其值，否則以 `Debug`，否則只印出其型別名稱；結尾的 `?` 印出 `Debug` 形式，否則印出型別名稱（9.7.7）。

不支援下列繫結與效果。

- 無法在頂層命名之型別的繫結（閉包、私有型別等）。它們在建立它們的儲存格中就會被拒絕。
- 在儲存格內呼叫 `exit(..)`。該儲存格被當作失敗的儲存格處理（規則 5）。（單獨的 `exit` 或 `quit` 會結束工作階段，[9.9.9](909-debug-tools.md#999-repl--jupyter-debugging)。）
- 由失敗的儲存格產生的執行緒。它們不會在它之後存活（規則 5）。

## 9.7.1. 儲存格的種類

REPL 的一行或 Jupyter 的一個儲存格依其內容以不同方式處理。

```
1. expression                → evaluate + bind as `it` + print
2. statement (let / call)    → execute (return value is unit)
3. definition (fn / struct)  → add to the session's items
4. import (use / mod)        → update the session's visible names
5. meta / attribute          → handle as appropriate
```

五種都遵循相同的儲存格處理流程；只有最後的輸出不同。運算式儲存格所顯示的值會繫結到 `it`，供之後的儲存格使用（9.7.7）。運算式儲存格會印出其值（`Display`，否則 `Debug`，否則其型別名稱）；`let` 儲存格不印出任何東西。發散的運算式儲存格（[4.3.7](../004-functions/403-return.md#437-divergent-functions)；`panic(..)`、`exit(..)`、`throw`）作為陳述式執行。

儲存格可以在頂層陳述式中 `await`，就像隱含的 `main` 一樣（[4.16.5](../004-functions/416-entry-point.md#4165-async-main)）：此時該儲存格在執行器上執行直到完成。其 `let` 繫結與其值就像其他任何儲存格的一樣延續到之後的儲存格。

> [!IMPORTANT]
> 就語言語意而言，一連串的儲存格等同於把頂層陳述式與項目附加到一個隱含之 `main.nabi` 的結尾（[4.16.10](../004-functions/416-entry-point.md#41610-relationship-to-jit--repl)）。儲存格的 `let` 是隱含 `main` 的區域變數，因此之後儲存格的陳述式看得到它，但在儲存格中宣告之 `fn` 的主體內看不到。下文 9.7.3 的全域儲存是讓區域變數跨越儲存格邊界保持存活的實作技術，不會改變這個可見性規則。儲存格重新執行（9.7.4）、自動顯示最後一個運算式的值及其繫結 `it`（9.7.7）是只存在於互動式介面上的工作階段操作。

## 9.7.2. 執行流程：詳細說明

```
1. parse → AST                      (error: display, discard cell)
2. lower to HIR, type check         (error: display, discard cell)
3. MIR → native machine code        (reuse / redefinition handling, 9.6.3 / 9.6.4)
4. host: load in a process forked from the snapshot, run the cell entry function
5. success: new snapshot, apply bindings and items to the session, print result
   failure: continue from the previous snapshot, display error
```

每個步驟都是交易式的；失敗時會回復到先前的狀態。

當 REPL 讀取終端機時，儲存格的標準輸入就是終端機，否則為空（[9.7.10](#9710-loading-a-program-repl-path)）；Jupyter 儲存格經由用戶端讀取（[9.8.1](908-jupyter-integration.md#981-kernel)）。儲存格的標準輸出在每個換行時都會清空緩衝，即使它不是終端機也一樣，因此其輸出會在寫出時就顯示。無論使用者的環境怎麼說，REPL 主機都會在執行環境啟動之前設定 `NABIR_GC_CONCURRENT=off`：並行標記的執行緒不會被納入快照，因此工作階段中的完整回收在沒有它的情況下標記（[7.5](../007-runtime/705-concurrent-collection.md)）。

## 9.7.3. 頂層變數的儲存

變數必須在 REPL 與 Jupyter 的儲存格之間共用，因此頂層變數必須在工作階段持續期間保持存活。

```nabi
[1] let x = 42i32;
[2] let y = x + 1;       // uses the earlier x
```

### 儲存形式

```
let x = 42                  → global of the inferred type (i32)
let x: i64 = 42             → global of the written type (i64)
heap payload                → register GC root
                             (compiler-emitted shadow stack frame,
                              linked into nabir shadow stack chain)
```

每個頂層變數都以其型別（寫出的或推論的）的匯出全域變數存放在主機中。當之後的儲存格參照同一個變數時，經由該全域變數的位址找到它。

### GC 根的註冊

指向堆積物件的頂層變數必須作為根被 GC 追蹤，因此它們被連結到[影子堆疊](../000-meta/007-glossary.md#shadow-stack)鏈中。

詳細的機制請見[屏障與根追蹤](../007-runtime/704-barriers-and-roots.md)中影子堆疊的部分。

### 同名的重新宣告

```nabi
[1] let x = 1i32;            // x: i32
[2] let x = "hi";            // redeclaration at cell top level: new binding (x: str)
```

對已在儲存格頂層繫結之名稱的 `let` 是重新宣告，與原始碼檔案中相同（[5.1.4](../005-modules/501-variable-declaration.md#514-redeclaration-in-the-same-scope)）。它建立一個新的繫結，其型別可以與先前的繫結不同。之後的儲存格看到新的繫結。

既有的全域變數於是被覆寫：其記憶體留在原處，只有名稱槽位被重新導向到新的定義。

這支援重新定義變數這種常見的 REPL 用法。

## 9.7.4. 復原 / 重新執行儲存格

### 在 Jupyter 中重新執行儲存格

當儲存格在 Jupyter 中被編輯並再次執行時，它作為新的儲存格執行。它所重新定義的項目與繫結，從之後的儲存格起以其新的形式使用（9.7.0 規則 2、3）。先前執行過之儲存格的效果不會被抹除。

### 復原的意義

REPL 沒有明確的復原；工作以覆寫最後一個儲存格或新增儲存格的方式進行。

在 Jupyter 中也一樣，刪除或重新執行儲存格不會復原先前儲存格的效果。要回到初始狀態，請重新啟動工作階段（REPL 中的 `:reset`）。

## 9.7.5. 錯誤模式

### 儲存格編譯錯誤

```
parse error / type error / semantic error.
  - the cell is not added.
  - the session's bindings and items are unchanged.
  - reuse information for already compiled functions is kept.
  - the error is shown to the user.
```

這個步驟完全是交易式的，因此錯誤發生前一刻的狀態會被保留。若編譯器本身在編譯儲存格時 panic（編譯器內部錯誤），`nabi lsp`、`nabi repl` 與 `nabi jupyter` 都能撐過去：語言伺服器把它回報為被分析之緩衝區的錯誤診斷，REPL / 核心讓該儲存格失敗；兩者都繼續提供服務。

### 儲存格執行時錯誤

```
panic / uncaught throw / process exit / segfault during execution.
  - the cell is not added; bindings and items stay as they were before the cell.
  - the error is shown to the user.
  - the next cell can proceed.
```

錯誤會說明儲存格如何結束，以及工作階段未被改變：

```
panic             the panic's report, then: the cell panicked; the session is unchanged
uncaught throw    the cell threw `<error>`; the session is unchanged
                    --> <where the error left the cell>
                    raised at <the throw that raised it, when known and elsewhere>
exit(n)           the cell called exit(n); the session is unchanged (`exit` or `:quit` ends the session)
Ctrl-C            interrupted (Ctrl-C); the session is unchanged
other (a fault)   the cell failed; the session is unchanged
```

`<error>` 是錯誤的描述，與從 `main` 逸出之未捕捉錯誤的描述相同（[4.16.4](../004-functions/416-entry-point.md)）。沒有任何訊息會指名儲存格所產生的進入函式。

繫結與項目會回復到該儲存格之前的狀態（9.7.0 規則 5），但已經發生的外部效果（輸出、檔案寫入等）不會被撤銷。失敗的儲存格不會保留在工作階段中，因此可以再次輸入修正後的儲存格。沒有 fork 的主機（Windows）在儲存格之前儲存工作階段的記憶體與執行緒，並在它失敗時把它們放回去，可觀察到的結果與 fork 出的快照相同。

### JIT 失敗

若主機無法載入或連結儲存格的程式碼（未知的 `extern` 符號等），就與儲存格執行時錯誤一樣處理。執行該儲存格的 fork 行程結束，由先前的快照接手。

### 位置

正在輸入之儲存格中的診斷指名 `<repl>:line:col`，以儲存格自己的文字計算。先前儲存格之項目中的診斷，以及執行中程式碼所回報的位置（panic、未捕捉的錯誤、`try!`），指名 `<repl cell N>:line:col`：儲存格 N 是工作階段中輸入的第 N 個儲存格（`:type` 等指令不是儲存格；失敗的儲存格也算在內），行與欄以其文字計算。在 Jupyter 核心中，N 是筆記本儲存格的執行計數，而拆成多個 REPL 儲存格的筆記本儲存格會把它們編號為 `N.1`、`N.2` 等（[9.8.1](908-jupyter-integration.md#notebook-cells-and-repl-cells)）。在先前儲存格中定義的函式會回報在該儲存格中的位置。

```
>>> fn boom(n: i32) {
...     panic(f"boom {n}");
... }
>>> boom(5)
thread 'main' panicked at <repl cell 1>:2:5:
boom 5
error: the cell panicked; the session is unchanged
```

## 9.7.6. 快取

在一個工作階段內，主機會重複使用它已經編譯過的函式（9.6.4）。沒有跨工作階段的磁碟快取。

## 9.7.7. 視覺化 / 輸出

### REPL 輸出

```
>>> 1 + 2
3
>>> "hi"
hi
```

最後一個運算式的值以 `Display` 印出，否則以 `Debug`。結尾的 `?`（如 `xs?`）則印出 `Debug` 形式，而 `:type xs` 只印出型別。沒有 `Display` 與 `Debug`（對 `?` 而言是 `Debug`）的值只印出其型別名稱（`type_name_of_val`，[stdlib 2.12.4](../../refs-standard-libraries/200-core/212-reflect.md#2124-repl-output)：`any` 值會指出它所容納的型別）。`let` 儲存格不印出任何東西。在 Jupyter 筆記本中，帶有 `RichDisplay` 的值改為顯示其 MIME bundle（[9.8.2](908-jupyter-integration.md#982-local-native-kernel-operation)）。

```
>>> let xs = [1, 2, 3];
>>> xs
[1, 2, 3]
>>> "hi"?
"hi"
>>> struct Opaque { id: i32 }
>>> Opaque { id: 1 }
Opaque
```

### `it`：最後顯示的值

運算式儲存格所顯示的值會繫結到 `it`，它和其他任何繫結一樣是工作階段的頂層繫結：之後的儲存格可以讀取並指派它，`:vars` 也會列出它。不顯示任何東西的儲存格（陳述式、unit 值、失敗的儲存格）讓 `it` 保持原樣；其型別無法在頂層命名的值也是如此（9.7.0）。

```
>>> 2 + 3
5
>>> it * 10
50
>>> it
50
```

使用者定義的名稱勝出：一旦某個儲存格以 `let` 繫結 `it`，或定義了名為 `it` 的項目，運算式儲存格就不再繫結它，直到 `:reset`。在 Jupyter 工作階段中也是如此。

### Jupyter 輸出

每個儲存格的結果以只有單一 `text/plain` 項目的 `execute_result` 傳送，內容與 REPL 印出的文字相同。不會產生其他種類的 MIME 內容。

Jupyter 整合的細節定義於 [Jupyter 整合](908-jupyter-integration.md)。

## 9.7.8. 輸入單位：REPL 的辨識

REPL 以完整的運算式或陳述式為單位處理輸入。

```
>>> 1 + 2
3
>>> let x =
... 42i32;          (continued input)
>>> x
42
```

提示符號是 `>>>`，儲存格尚未結束時是 `...`。儲存格在行尾結束，以語彙單元判斷：當括號、字串或區塊註解仍未關閉時，當最後一個語彙單元無法結束陳述式時（二元運算子、`.`、`,`、`=`、`->`、`|`，或 `else`、`in` 等關鍵字），或當輸入以沒有主體的項目標頭結束時（`fn g(x: i32)`，而 `-> i32 { .. }` 在下一行），輸入會以 `...` 繼續。字元字面值、字串與註解中的括號不算在內。其他任何行都會結束儲存格。`///` 文件註解行也會讓輸入繼續：它屬於其後各行上的項目。以 `:` 開頭的行是指令，在該行結束。當儲存格的最後一個頂層陳述式是沒有 `;` 的 `let` 或指派（如 `let x = 1` 或 `k += 2`）時，會補上 `;`：`let` 照常繫結，而儲存格不顯示任何東西。其他陳述式不會改變，診斷以輸入時的樣子引用儲存格。在終端機上，每個提示符號都從新的一行開始，即使在沒有以換行結尾的輸出之後也是，而且不會畫出任何標記；經由管道的輸入沒有提示符號。當標準輸入在一個不完整的儲存格中結束時，該儲存格不會執行，會回報錯誤，`nabi repl` 以 1 結束。失敗的儲存格不會改變結束碼。

### 行編輯

在終端機上，REPL 自己編輯每一行；經由管道的輸入原樣讀取。

- **啟動。** 當標準輸入與輸出都是終端機時，REPL 在第一個提示符號之前印出兩行：`Nabi <version> (<commit> <date>) on <host triple>`，其中版本文字與 `nabic --version` 印出的相同，然後是 `Type :help for help, :exit to quit.`。`-q` 會省略兩者；經由管道的輸入絕不會得到它們。
- **按鍵。** 方向鍵、Home / End 與 Emacs 按鍵用於移動與編輯：`C-a` / `C-e` 移到行首與行尾，`C-b` / `C-f` 與 `M-b` / `M-f`（也可以是 Alt 或 Ctrl 搭配方向鍵）以字元或單字移動，`C-k` / `C-u` / `C-w` / `M-Backspace` 刪除到行尾、到行首或前一個單字，`C-d` 刪除下一個字元，或在空行上結束輸入，`C-l` 清除畫面，`C-c` 捨棄該行以及未結束的儲存格。在 Unix 上，`C-z` 把 REPL 作為工作暫停。
- **顯示。** 長行會水平捲動。欄位以顯示寬度計算：East Asian Wide 與 Fullwidth 字元以及表情符號占兩欄，組合用附加符號與其他零寬度字元不占欄。輸入依語彙單元類別上色，與文件相同：關鍵字與屬性為淺粉紅色，型別為天藍色，字串為綠色，數字為琥珀色，註解為灰色，使用 [9.1.8](901-nabic-cli.md#colour) 的調色盤與色深。標準輸出上的色彩遵循 [9.1.5](901-nabic-cli.md#915-global-options)：`--color=never`、`NO_COLOR` 或 `TERM=dumb` 會關閉它。
- **補全。** Tab 把游標前的單字補全到各候選項的最長共同前綴，若這樣沒有增加任何東西，就列出它們。候選項是游標處範圍內的區域變數、工作階段的繫結與項目、prelude 的名稱（`use core::prelude::*;`）、工作階段之 `use` 項目所匯入的名稱，以及關鍵字；在 `x.` 之後是 `x` 之型別的欄位與方法（對標準函式庫的型別只有其公開者），在 `a::` 之後是 `a` 之下的模組與項目。在行首按 Tab 會縮排四個空白。Jupyter 核心從相同的候選項補全（[9.8.5](908-jupyter-integration.md#985-code-completion)），語言伺服器也使用相同的引擎（[9.9.13](909-debug-tools.md#9913-lsp-server)）。
- **貼上。** REPL 開啟 bracketed paste：貼上的區塊，包括換行與 Tab，會作為文字插入，並在按下 Enter 時成為一個儲存格，而不是逐行執行。編輯時，輸入中的換行顯示為 `↵`；輸入之後，儲存格顯示為 `...` 之後的各行。
- **歷史。** 歷史容納整個儲存格：跨多行輸入或貼上的儲存格是一個項目。Up / Down（`C-p` / `C-n`）走訪它；已輸入文字時，Up 與 Down 只走訪以該文字開頭的項目。`C-r` 向後漸進式搜尋：輸入的文字會縮小搜尋範圍，再按 `C-r` 會找到更舊的相符項目，Enter 執行該相符項目，其他按鍵會編輯它，`C-g` 回到原本的行。歷史在啟動時載入，並保存在 `$NABI_HOME/repl_history` 中；它保留最後 1000 個儲存格，在容納 2000 個時捨棄較舊的，並且不儲存超過 64 KiB 的儲存格。
- **Windows。** 主控台會切換為 VT 輸入與輸出。在支援它的終端機（例如 Windows Terminal）中，bracketed paste 可以運作。

## 9.7.9. 輸入單位：Jupyter 儲存格

Jupyter 儲存格是一次處理的多行輸入。

```
[Cell 1]:
let x = 42i32;
let y = x * 2i32;
y                       # last expression = cell result
```

儲存格的最後一個運算式是儲存格的結果；其他一切都被當作副作用（定義或呼叫）。

## 9.7.10. 載入程式（`repl <path>`）

帶有程式路徑的 `nabi repl <path>` 把程式載入新的工作階段，然後在該工作階段上繼續 REPL。帶有函式庫套件路徑時，該套件會被加為相依套件，並以 `use <name>::*;` 作為第一個儲存格執行（`:reset` 之後也會再次執行）；不會載入其他任何東西。

- **先檢查。** 在執行任何東西之前，會檢查整個程式。錯誤以檔案自己的位置回報；不會執行任何東西，工作階段以空的狀態開始。
- **項目。** 項目與匯入加入工作階段。
- **陳述式。** 頂層陳述式作為儲存格執行。每個儲存格在一個頂層 `let`（或最後一個陳述式）處結束。
- **失敗。** 發生 panic、`exit()` 或逃逸的錯誤時，已完成之儲存格的繫結保留其在該儲存格結束時的值；失敗區段中的指派會遺失。程式的其餘部分不會執行，工作階段繼續。
- **無法使用的繫結。** 之後的儲存格無法使用的繫結（私有或無法命名的型別），若之後沒有任何陳述式指名它，就會被捨棄；否則到其最後一次使用為止的陳述式作為一個儲存格執行。
- **`fn main`。** 帶有 `fn main` 的程式以 `main` 作為進入點執行；之後 `main` 離開工作階段，因此其區域變數不可見。
- **位置。** 診斷與 panic 使用檔案自己的位置。
- **輸入。** 程式與任何儲存格一樣讀取 stdin。在終端機上，執行中的儲存格讀取終端機（控制終端機；在 Windows 上是主控台 `CONIN$`）；儲存格結束之後，行編輯器以儲存格之前的模式取回終端機。經由管道的輸入屬於 REPL：儲存格讀取空的 stdin（`/dev/null`，在 Windows 上是 `NUL`），並立即看到輸入結束，而其後的各行仍作為儲存格讀取。
- **`:load`、`:reload`。** `:load <file>` 依相同規則把程式載入執行中的工作階段；其項目取代工作階段中同名的項目。`:reload` 再次載入最後載入的程式（[9.9.9](909-debug-tools.md#999-repl--jupyter-debugging)）。由 `:save` 寫出的腳本不會作為程式載入：其儲存格會一個一個地再次執行，就像被輸入一樣，因此再次宣告的名稱保留其最後的繫結。

## 9.7.11. 接下來

[Jupyter 整合](908-jupyter-integration.md)定義經由本機 ZMQ 的 Jupyter 核心整合。
