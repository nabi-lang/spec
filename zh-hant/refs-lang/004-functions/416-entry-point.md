# 4.16. 進入點

本節定義可執行套件啟動時執行哪些程式碼、以什麼順序執行。這正是[原始碼單元](../000-meta/003-source-units.md)以「進入點的確切形式定義於函式規格」留下待定之處。

## 4.16.1. 進入模組：`main.nabi`

可執行套件的根檔案 `src/main.nabi`（[0.3.1](../000-meta/003-source-units.md#可執行套件srcmainnabi)）是**進入模組**。進入模組以下列兩種形式中的**恰好一種**提供進入點。

| 形式                  | 進入點                                               | 用途                             |
|-----------------------|------------------------------------------------------|----------------------------------|
| 明確的 `fn main`      | 使用者宣告的 `fn main` 函式                          | 一般的應用程式                   |
| 頂層陳述式            | 以頂層陳述式為主體的**隱含 `main`**                  | 指令碼、範例、短程式             |

```ebnf
main_file   ::= item* fn_main item*            // form 1: explicit main
              | (item | top_stmt)*             // form 2: top-level statements (no fn main)
top_stmt    ::= let_stmt | let_else_stmt | expr ";" | block_like_expr ";"?   // 3.9.1
module_file ::= item*                          // lib.nabi and every module file other than main.nabi
```

`item` 是[模組中的項目](../005-modules/504-modules.md#547-模組內的項目)（`fn`、`struct`、`enum`、`trait`、`impl`、`const`、`static`、`mod`、`use` 等），而 `let_stmt` / `let_else_stmt` / `expr ";"` 與[區塊運算式](../003-expressions/309-block-expressions.md)的陳述式相同。控制流程（`if`、`match`、`for`、`while`、`loop`、`try`）由運算式構成，因此包含在 `expr ";"` 中。與函式主體不同，這裡沒有不帶終止符的最後運算式（`tail_expr`）：隱含 `main` 不回傳任何值。只有項目而沒有陳述式（也沒有 `fn main`）的 `main.nabi` 會得到一個空的隱含 `main`，但匯出 WIT 的套件除外：那種套件是沒有進入點的 reactor（[stdlib 1.8.2](../../refs-standard-libraries/100-bare/108-wit.md#匯出)）。

> [!IMPORTANT]
> 若單一 `main.nabi` **同時**包含頂層陳述式與 `fn main` 宣告，就是**編譯錯誤**（E0036）。這排除了讀者必須猜測兩個進入點中哪一個會執行的任何情況。

```nabi
// main.nabi: ERROR
fn main() { run(); }
run();                      // ERROR: top-level statement in a file with fn main
```

## 4.16.2. 頂層陳述式的意義：隱含 `main`

沒有 `fn main` 之 `main.nabi` 的頂層陳述式，會**依原始碼順序**收集到一個函式主體中，成為編譯器合成之隱含 `main` 的主體。項目不會被執行，也不會放進主體；即使項目與陳述式交錯出現，有意義的也只有陳述式之間的相對順序。

```nabi
// main.nabi
let names = ["ada", "grace"];
for n in names {
    greet(n);               // greet is declared below; visible regardless of order
}

fn greet(name: str) {
    println(f"hello, {name}");
}

let total = names.len();
println(f"{total} greeted");
```

上面的檔案與下列意義相同：

```nabi
fn greet(name: str) {
    println(f"hello, {name}");
}

fn main() throws {          // form of the implicit main (see 4.16.4)
    let names = ["ada", "grace"];
    for n in names {
        greet(n);
    }
    let total = names.len();
    println(f"{total} greeted");
}
```

從這種等價關係可以得出下列規則。

- **項目無論順序都可見。** `main.nabi` 中的項目與一般模組項目一樣屬於模組作用域，因此頂層陳述式可以參照在它之後宣告的 `fn` / `struct` 等。
- **頂層的 `let` 是隱含 `main` 的區域變數。** 它不是全域變數，只在其宣告之後的頂層陳述式中可見（[變數宣告](../005-modules/501-variable-declaration.md)的區塊作用域規則原樣適用）。
- **頂層的 `let` 在項目主體內不可見。** 在上面的範例中，若 `greet` 的主體參照 `names` 或 `total`，就是名稱解析錯誤。以閉包擷取該區域變數是可能的（`let shout = |s| println(f"{s}! ({total})");`）。閉包是陳述式，在隱含 `main` 的主體內建立。

> [!NOTE]
> **為什麼頂層的 `let` 對項目不可見。** 項目的存在與順序無關，但 `let` 只有在執行時才得到其值。若 `fn greet` 看得到頂層的 `let`，就必須為 `greet` 在該 `let` **之前**被呼叫的情況（例如在第一行呼叫 `greet()`）定義意義，這等於把「全域初始化順序」的語意帶進語言中。Nabi 的原則是沒有全域初始化順序（4.16.6），因此頂層的 `let` 只限於隱含 `main` 的區域變數。必須在項目之間共享的狀態，以參數傳遞或宣告為 `static`。

## 4.16.3. 明確 `main` 的簽章

明確的 `fn main` 沒有參數，也沒有泛型參數，而且必須是下列形式之一。其他任何形式都是編譯錯誤（E0036）。

| 簽章                              | 正常結束時的結束碼       | 備註                                   |
|-----------------------------------|--------------------------|----------------------------------------|
| `fn main()`                       | `0`                      | 隱含地 throws：未處理的例外 → 4.16.4   |
| `fn main() -> i32`                | 回傳值                   | 回傳值就是行程的結束碼                 |
| `fn main() throws` / `throws (E)` | `0`                      | 未處理的例外 → 4.16.4                  |
| 上述三種形式加上 `async`          | 同上                     | 4.16.5                                 |

沒有回傳 `Result` 的 `main`，也沒有同時帶有 `throws` 與 `-> i32` 的 `main`。若錯誤時需要特定的結束碼，請在 `catch` 中呼叫 [`std::process::exit(code)`](../../refs-standard-libraries/300-std/306-process.md)。

命令列引數與環境變數透過標準函式庫 [`std::env`](../../refs-standard-libraries/300-std/305-env.md) 取得，而不是透過參數。`unsafe` 與 `yield` 修飾詞不能套用於 `main`。可見性（`pub`）不影響它是否為進入點。

在 `lib.nabi` 或一般模組中宣告的 `fn main` 不是進入點；它只是一個名為 `main` 的一般函式。

## 4.16.4. 結束碼與未處理的例外

行程的結束碼由進入點的結束方式決定。

- **正常結束**：依 4.16.3 的表格，為 `0` 或回傳的 `i32`。
- **未處理的例外**：若在會 throws 的 main（包括沒有 `->` 的 `fn main()`）中有錯誤傳播出進入點，執行環境會把 ``error: `main` returned an uncaught error: `` 及其後的例外值印到標準錯誤（stderr），並以結束碼 **`1`** 結束。輸出格式在值實作了 [`Display`](../../refs-standard-libraries/200-core/210-fmt.md) 時是 `Display`，否則是 `Debug`，兩者都沒有時則是型別名稱（[`Error`](../../refs-standard-libraries/200-core/214-error.md) trait 要求 `Display`，因此一般的錯誤型別以 `Display` 印出）。下一行以 `  --> path:line:col` 給出錯誤離開 `main` 的位置。當引發它的 `throw` 已知且位於他處時，最後一行以 `  raised at path:line:col` 給出其位置（指出哪個 `throw` 遵循失敗的 `try!` 的規則，[3.14.8](../003-expressions/314-error-control-flow.md#3148-panic-與致命錯誤)）。
- **panic**：panic 與致命錯誤不會被 `catch` 捕捉；執行環境把訊息印到標準錯誤，並以結束碼 **`101`** 結束；在 C 所呼叫的函式之下則改為中止（[4.14.13](414-throws.md#41413-panic-與致命錯誤)）。
- **明確結束**：[`std::process::exit(code)`](../../refs-standard-libraries/300-std/306-process.md) 從任何位置立即終止行程。這條路徑不會執行 `finally` 或 `Drop`。

沒有 `->` 也沒有 `throws` 的明確 `fn main()`，與隱含 `main` 一樣被讀作 `fn main() throws`（未定型）：指令碼的 `try` 不需要 `main` 上的 `throws`。`fn main() -> i32` 不會擲回錯誤（帶 `-> i32` 的 `throws` 不是進入點的形式）。

**隱含 `main`** 被視為未定型的 `fn main() throws`（錯誤型別為 `any Error`）（若主體含有 `await`，則為 `async fn main() throws`，4.16.5）。因此頂層陳述式中以 `try` 標示之會擲回錯誤的呼叫，其錯誤不需要另外的 `catch` 就可以傳播出進入點，而未處理的例外依上述規則得到結束碼 `1`。若執行正常地跑到結尾，結束碼是 `0`。

在 `wasm32-wasip2`（preview 2 元件）上，`wasi:cli/exit` 只能傳遞成功或失敗，因此任何非零結束碼（包括 `1` 與 `101`）都以失敗（狀態 1）到達主機。`wasm32-wasip1-threads`（preview 1）傳遞完整的結束碼，但主機可能把 126 以上的 `proc_exit` 轉成失敗（狀態 1）。`abort` 之類的 trap 以 134 結束。

> [!IMPORTANT]
> **頂層的 `return;` 結束程式。** 在 `main.nabi` 的頂層陳述式中（包括巢狀區塊與 `let .. else`），單獨的 `return;` 從隱含 `main` 回傳：`finally` 區塊與 `using` 的釋放會執行，結束碼是 `0`。這讓指令碼能提前停止，例如 `let x = opt else { return; };`。頂層陳述式不回傳任何東西，因此那裡的 `return v;` 是編譯錯誤（E0018），其說明會指出 `return;` 與 `process::exit(v)`。要使用其他結束碼，請呼叫 `process::exit(code)`，或使用明確的 `fn main() -> i32` 形式。

## 4.16.5. 非同步 `main`

`async fn main()` 作為執行環境[排程器](../007-runtime/706-scheduler.md)上的頂層任務執行。平台的進入包裝程式初始化執行環境，把 `main` 建立的 Future 交給排程器，等待該 Future 完成，然後依 4.16.4 的規則決定結束碼（與 [`task::block_on`](../../refs-standard-libraries/200-core/220-future.md) 的行為相同）。`main` 結束時仍在執行的其他任務不會被等待。

當 `await` 直接出現在頂層陳述式中（而不是在另一個 `async` 區塊或閉包內）時，隱含 `main` 成為 `async fn main() throws`；否則它是同步的 `fn main() throws`。

```nabi
// main.nabi: implicit async main
use std::fs;
let body = try await fs::read_to_str_async("notes.txt");
println(body.len());
```

## 4.16.6. 模組只含定義

頂層陳述式**只允許出現在進入模組 `main.nabi` 中**。`lib.nabi` 以及 `main.nabi` 以外的每個模組檔案（以及行內 `mod { ... }` 主體）只能包含項目（`module_file ::= item*`）；那裡的頂層陳述式是**編譯錯誤**。對於行內 `mod x { stmt }` 主體中的陳述式，會發出指向本節的專用診斷。

結果是，**匯入模組（`mod`、`use`）不會執行任何程式碼。**這與 Python 刻意不同。在 Python 中，`import` 會執行模組檔案的頂層程式碼，因此同時用作函式庫與指令碼的檔案必須以 `if __name__ == "__main__":` 守衛區分兩種用途。Nabi 不需要這種守衛：作為指令碼執行的程式碼只存在於 `main.nabi` 中，而其他每個檔案在匯入時都不會造成任何事。

這個決定的理由：

- **AOT 是意義的基準**（[0.1.4](../000-meta/001-execution-modes.md#014-語意基準)）。要在 AOT 產出中支援匯入時執行，就必須在連結時合成每個模組的初始化函式及其呼叫順序。只含定義的模組不需要這種機制。
- **沒有初始化順序的語意。** 跨模組的匯入時程式碼，需要定義「哪個模組先初始化」以及「循環匯入期間看得到什麼」。Nabi 確保這些問題永遠不會出現。全域狀態的初始值在編譯期決定（4.16.7）。
- **快取與增量建置。** 由於模組的意義只由其定義決定，以內容雜湊為基礎的快取（[0.3.3](../000-meta/003-source-units.md#033-套件識別碼)）與增量建置不需要考慮執行的副作用。
- **套件種類由檔案是否存在決定**（[0.3.4](../000-meta/003-source-units.md#034-清單nabinabion是選用的)）。可執行的程式碼恰好位於 `src/main.nabi`，因此「這是可執行套件嗎？」與「可執行的程式碼在哪裡？」的答案永遠一致。

## 4.16.7. 全域狀態

模組層級的全域狀態只能以 [`static`](../005-modules/503-static.md) 宣告。`const` 初始化式在編譯期求值；其他任何初始化式在第一次存取其 `static` 時執行，絕不會在進入點之前執行（[5.3.9](../005-modules/503-static.md#539-初始化式編譯期或第一次使用)）。因此程式的執行一律從進入點的第一個陳述式開始，在它之前不會執行任何使用者程式碼。

## 4.16.8. 單一檔案執行

不屬於任何套件的單一 `.nabi` 檔案可以直接執行。

```
nabi run hello.nabi
nabic build hello.nabi
```

若作為引數給出的檔案不在某個套件的 `src/` 中，編譯器會建構一個只由該檔案組成的**匿名可執行套件**，並把該檔案視為該套件的 `main.nabi`。檔名沒有意義，進入點規則（4.16.1 到 4.16.5）與 `main.nabi` 完全相同地適用。位於套件 `src/` 旁邊的 `.nabi` 檔案，例如其根目錄中的指令碼，也以這種方式執行。匿名套件可以有檔案子模組：`mod foo;` 載入該檔案旁邊的 `foo.nabi`（其子模組位於 `foo/`），與套件 `src/` 中完全相同（[5.4.4](../005-modules/504-modules.md#544-套件根)）。只有沒有檔案的原始碼（標準輸入、REPL 工作階段）不行；它們使用行內模組。若需要相依套件，就在該檔案及其任何子模組檔案開頭的行內清單區塊中宣告（[9.10.7](../009-tools/910-nabi-frontend.md#9107-單檔指令碼的行內資訊清單)）。詳細的指令形式定義於 [`nabic` CLI](../009-tools/901-nabic-cli.md) 與 [`nabi run`](../009-tools/910-nabi-frontend.md#nabi-run建置並執行)。

## 4.16.9. 沒有進入點屬性

Nabi **沒有**用來指定進入點的屬性（例如 `#[main]`）。根檔案 `src/main.nabi` 已經指定了進入模組，而名稱 `main`（或頂層陳述式）指定了進入函式，因此沒有第二種表達相同資訊的手段。只保留單一機制來決定進入點，表示「哪個函式會執行」只看檔案結構就能決定。`main` 不是保留字，語言也不保留 `#[main]` 這個名稱。

## 4.16.10. 與 JIT / REPL 的關係

REPL 輸入與 Jupyter 儲存格（[JIT 儲存格](../009-tools/907-jit-cells.md)）的意義，等同於**把頂層陳述式與項目依序附加到隱含 `main.nabi` 的末尾**。儲存格的 `let` 是隱含 `main` 的區域變數，在之後的儲存格中可見；儲存格的 `fn` / `struct` 成為模組項目。4.16.2 的規則原樣適用，因此在儲存格中宣告的 `fn`，其主體不能參照先前儲存格的頂層 `let`（可以用閉包擷取）。

由於這種等價關係，[0.1.1](../000-meta/001-execution-modes.md#語意相同) 中「JIT 與 AOT 共用相同語意」的原則對進入點也成立。把筆記本的儲存格依序串接起來所得的檔案，是一個產生相同結果的 `main.nabi`。不過，下列三項是只存在於互動式工作階段中的**工作階段操作**，而不是語言語意的擴充：

- 刪除與重新執行儲存格（[9.7.4](../009-tools/907-jit-cells.md#974-復原--重新執行儲存格)）。
- 自動顯示不帶終止符之最後運算式的值（[9.7.7](../009-tools/907-jit-cells.md#977-視覺化--輸出)）。在檔案中，這等同於在該運算式後面加上 `;`。
- 該最後運算式結尾的 `?`（`xs?`），它顯示 `Debug` 形式（[9.7.7](../009-tools/907-jit-cells.md#977-視覺化--輸出)）。在檔案中，結尾的 `?` 是 [`?` 運算子](../003-expressions/314-error-control-flow.md#31410--運算子提前回傳-none)。
