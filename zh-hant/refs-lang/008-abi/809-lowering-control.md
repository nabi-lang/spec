# 8.9. 降階：控制流程與 GC 整合

本節涵蓋 [`if` 運算式](../003-expressions/310-if-expression.md)、[`match`](../003-expressions/312-match-expression.md)、[迴圈](../003-expressions/313-loop-expressions.md)與[錯誤控制流程](../003-expressions/314-error-control-flow.md)的降階規則，也定義發出 [GC](../000-meta/007-glossary.md#gc垃圾回收器) 配置 / 屏障 / 安全點程式碼的規則。

nabi-mir 是一個控制流程圖：帶有區塊引數的基本區塊，每個都以 goto、對 `bool` 的雙向分支、回傳或 `unreachable` 結束。程式碼產生器把每一種發出為跳躍、條件分支、回傳或 trap；區塊引數是在跳躍之前對目標區塊之槽位的存入。在 `-O0` 以上，最佳化器在這個圖上運作（[8.7.3](807-code-generation.md#873-最佳化等級)）。程式碼是虛擬碼（[8.7.1](807-code-generation.md#871-轉譯流程)）。

## 8.9.1. `if` 運算式

```nabi
if cond { x } else { y }
```

這個運算式被降階為雙向分支；值作為匯合區塊的引數到達。

```
bb0:
    if cond then bb_then else bb_else
bb_then:
    goto bb_end(x)
bb_else:
    goto bb_end(y)
bb_end(result):
```

沒有 `else` 分支、值為 unit 的 `if` 直接分支到匯合區塊：

```nabi
if cond { do_thing(); }
```

```
bb0:
    if cond then bb_then else bb_end
bb_then:
    call do_thing(tls)
    goto bb_end
bb_end:
```

## 8.9.2. `if let`

考慮 [`if` 運算式](../003-expressions/310-if-expression.md)中的 `if let pattern = expr` 形式。

```nabi
if let v: T = compute() { process(v) } else { fallback() }
```

模式的測試與 `match`（8.9.3）中一樣降階：

```
bb0:
    opt = call compute(tls)            ; (i32 tag, payload)
    if opt.0 != TAG_NONE then bb_then else bb_else
bb_then:
    v = opt.1 as T
    call process(v, tls)
    goto bb_end
bb_else:
    call fallback(tls)
    goto bb_end
```

## 8.9.3. `match` 決策樹

[`match`](../003-expressions/312-match-expression.md) 的分支依原始碼順序測試，每個模式是一串雙向分支。

```nabi
match v {
    1 => a,
    2 => b,
    n if n > 100 => c,
    _ => d,
}
```

這個 `match` 運算式變成下列內容。

```
bb0:
    if v == 1 then bb_a else bb_test2
bb_test2:
    if v == 2 then bb_b else bb_guard
bb_guard:
    n = v
    if n > 100 then bb_c else bb_d

bb_a:
    goto bb_end(a)
bb_b:
    goto bb_end(b)
bb_c:
    goto bb_end(c)
bb_d:
    goto bb_end(d)

bb_end(result):
    ...
```

元組、`struct`、`enum` 變體（[8.8.6](808-lowering-types.md#886-enum-的表示方式)）與和型別成員以相同方式測試其各部分；範圍模式是兩次比較，字串模式是一次 `str` 相等比較（[8.7.6](807-code-generation.md#876-extern-函式執行環境函式)）。match 是窮盡的，因此其最後一次測試之後的區塊是無法到達的（一個 trap）。

不會建立跳躍表：在每個等級，這串鏈都依寫出的樣子編譯，每個被測試的分支一次比較與分支。最佳化器會移除它能從前面分支所證明的事情決定的測試。

## 8.9.4. `loop` / `while` / `for`

### `loop`

```nabi
let v = loop { ...; if cond { break x; } }
```

迴圈是一個標頭區塊以及跳回它的回邊跳躍；`break x` 以 `x` 作為引數分支到離開區塊。

```
    goto bb_head
bb_head:
    ; body
    if cond then bb_exit(x) else bb_latch
bb_latch:
    ; safepoint poll (8.9.11)
    goto bb_head
bb_exit(v):
```

### `while`

```nabi
while cond { body }
```

條件在標頭中測試：

```
    goto bb_head
bb_head:
    c = ...
    if c then bb_body else bb_exit
bb_body:
    ; body, then the back-edge poll
    goto bb_head
bb_exit:
```

### `for`

```nabi
for x in xs { body }
```

[迴圈運算式](../003-expressions/313-loop-expressions.md)中的去語法糖把它展開為 `let it = xs.into_iter(); loop { match it.next() { ... } }`，然後經過上述相同的降階。對整數範圍（`for i in a..b`，邊界只求值一次）或陣列的 `for`，直接降階為一個在主體之前推進索引的計數 `while` 迴圈，因此 `continue` 無法略過它。

## 8.9.5. `break` / `continue`

`break` 與 `continue` 是跳到迴圈之離開區塊或其回邊區塊的跳躍。標籤（`'outer:`）同樣跳到適當的[區塊](../000-meta/007-glossary.md#區塊block)。

## 8.9.6. `try` / `catch` / `finally`：以值為基礎的分支

[錯誤控制流程](../003-expressions/314-error-control-flow.md)的錯誤透過[呼叫慣例](806-calling-convention.md#865-會擲回錯誤之函式的回傳以值為基礎的錯誤傳播)中定義的 `{ is_err, payload }` 回傳值傳播。因此 `try` 與 `catch` 都被降階為一般的分支。

```nabi
let v = try f(x);              // f: fn(X) throws (E) -> T
```

以 `try` 標示的呼叫檢查回傳值的 `is_err`：成功時以 `T` 取出酬載；錯誤時分支到錯誤路徑。

```
bb0:
    r = call f(x, tls)                 ; (is_err, payload)
    if r.0 then bb_err else bb_ok

bb_ok:
    v = r.1 as T
    ...

bb_err:
    e = r.1 as E
    ; if there is an enclosing catch, go to its dispatch block;
    ; otherwise build the enclosing function's return value { 1, e } and return it
    goto bb_catch_dispatch(e)
```

帶有 `catch` 子句的運算式有單一的分派區塊，左側運算式的所有錯誤路徑都在那裡匯合。

```nabi
try body catch e: A { handler_a } catch _ { handler_any }
```

```
bb_catch_dispatch(e):
    ; test patterns in clause order (same scheme as the decision tree of 8.9.3)
    if <e is A> then bb_handler_a(e) else bb_handler_any(e)

bb_handler_a(ea):
    ; handler_a
    goto bb_end(result_a)

bb_handler_any(ex):
    ; handler_any
    goto bb_end(result_any)

bb_end(result):
    ...
```

若沒有任何子句相符，分派區塊會把錯誤傳給外層 `catch` 的分派區塊，或建立外層函式的錯誤回傳值並回傳它。對未定型 `throws`（`any Error`）之錯誤的型別模式測試，降階為 [`any`](808-lowering-types.md) 的標籤比較。

`try?` 在錯誤路徑上產生 `none` 並前往 `bb_end`；`try!` 在錯誤路徑上呼叫 `panic_fn`。

## 8.9.7. `throw`

`throw` 的用法如下。

```nabi
throw err;
```

`throw` 被降階為無條件分支到錯誤路徑。若最近的 `catch` 位於同一個函式中，就前往該分派區塊；否則建立外層函式的錯誤回傳值並回傳它。

```
; when a catch exists in the same function
goto bb_catch_dispatch(err)

; otherwise, error return from the function
r.0 = true                ; is_err
r.1 = err                 ; the payload as E
return r
```

把錯誤回傳出函式的路徑與一般的 `return` 相同，因此影子堆疊彈出之類的收尾程式碼會照常執行。沒有執行環境呼叫。

## 8.9.8. `finally`

`finally` 的用法如下。

```nabi
try { body } finally { cleanup }
```

它被降階為讓離開左側運算式的每條路徑都經過清理區塊。正常完成、經 `catch` 處理之後完成、未處理之錯誤的傳播，以及每條 `return` / `break` / `continue` 路徑，都先分支到清理區塊，再繼續前往其原本的目的地。錯誤傳播也是一般的分支，因此不需要另外的展開機制。

```
bb_body_ok(v):        goto bb_cleanup(K_OK, v)
bb_body_err(e):       goto bb_cleanup(K_ERR, e)
bb_body_return(rv):   goto bb_cleanup(K_RET, rv)

bb_cleanup(kind, val):
    ; cleanup
    ; depending on kind, branch to bb_end / error propagation / function return / loop destination
```

若在清理區塊內擲回新的錯誤，它會取代原本的路徑。panic 會終止行程，因此不會經過清理區塊。

## 8.9.9. GC 配置：快速 / 慢速路徑

請一併參考[記憶體配置](../007-runtime/702-allocation.md)。`struct` 是值，絕不會被配置（8.8.3）；堆積物件是閉包環境、box（`any`）、`List` / `Dict` 緩衝區以及在執行時建立的字串。

```nabi
let n = 10;
let f = |x: i32| x + n;    // env: header + an i32
```

環境在配置處以行內方式配置：

```
cur = load.ptr [t + p]                 ; tlab_cursor
lim = load.ptr [t + 2p]                ; tlab_limit
if 24 <= lim - cur then bb_fast else bb_slow
bb_fast:
    store.ptr [t + p], cur + 24        ; bump the cursor
    store.u32 [cur + 0], TYPE_ID_ENV   ; type_id
    store.u64 [cur + 8], 24            ; gc_word
    env = cur
    goto bb_done
bb_slow:                               ; placed out of line
    env = call nabi_gc_alloc_slow(24, 8, TYPE_ID_ENV)
    goto bb_done
bb_done:
```

`t` 是執行緒的 TLS 基址（[8.7.7](807-code-generation.md#877-tls-存取)），`24` 是物件大小（向上取整為 8），`8` 是對齊，`TYPE_ID_ENV` 是 type_id。物件放得下時，快速路徑推進 TLAB 游標，並寫入標頭的 `type_id` 與 `gc_word`（= 大小，[8.1.4](801-object-header-abi.md#814-gc_word偏移-8u64-或-ptr)）；標頭的位址就是該物件。否則它呼叫 `nabi_gc_alloc_slow`。不會寫入 `gc_flags`：TLAB 記憶體已清零。大型物件大小以上，或對齊超過 8 的物件，直接呼叫 `nabi_gc_alloc_slow`。

然後在 16 位元組的標頭之後存入各欄位：

```
store.i32 [env + 16], n
```

## 8.9.10. 寫入屏障

[屏障與根追蹤](../007-runtime/704-barriers-and-roots.md#742-寫入屏障物件記錄)中的物件記錄屏障。

```nabi
*p.field = q;       // write a GC pointer into a GC object
```

這被降階為下列內容。

```
; 1. write barrier: checks only the header of the destination p (may precede the store)
flags = load.u32 [p + 4]               ; gc_flags
if flags & UNLOGGED != 0 then bb_log else bb_store

bb_log:                                ; placed out of line
    call nabi_gc_wb_slow(p, slot, len) ; atomically clear UNLOGGED and add to the remembered set
    goto bb_store

bb_store:
    ; 2. ordinary store
    store.ptr [p + offset_of(field)], q
```

- 不會讀取被存入之值 `q` 的標頭。`q` 可能是 `any` 槽位的行內整數，或 `str` 的內部指標，因此把它當作標頭讀取是不安全的。
- 快速路徑是一次載入與一個分支。實際上它一起檢查 `UNLOGGED | CARDED`。若是 `CARDED`（大型的老物件），它不經呼叫地在卡片表（其指標是緊接在標頭之前的字組）中，把被寫入之槽位所跨越的 512 B 卡片（`(slot - p) >> 9` ..= `(slot + len - 1 - p) >> 9`）標記為髒；若是 `UNLOGGED`，它如上所述呼叫 `nabi_gc_wb_slow`。因此屏障使用槽位位址與被存入之值的大小（[7.4.2](../007-runtime/704-barriers-and-roots.md#742-寫入屏障物件記錄)）。
- `nabi_gc_wb_slow` 不是安全點。在屏障與存入之間不會插入任何回收。
- 對剛配置之物件的初始化、對 `static` 的存入、原始的 `Ptr<T>` 記憶體，以及非參照的 atomics，不插入屏障。

## 8.9.11. 安全點輪詢

也請參考[安全點協定](804-safepoint.md)。

```
; safepoint poll (t: the TLS base, 8.7.7)
flag = load.u8 [t + 10p+8]             ; yield_flag
if flag != 0 then bb_slow else bb_cont

bb_slow:                               ; placed out of line
    call nabi_runtime_yield()
    goto bb_cont

bb_cont:
    ; ... continue ...
```

編譯器在迴圈回邊上，以及在呼叫循環可以不經過迴圈輪詢就重新進入的函式中（遞迴循環的成員、透過指標或從 C 呼叫的函式、匯出的函式）發出這段程式碼：對於不是由直接的 Nabi 呼叫進入的函式，在進入處發出；對於其他函式，在每條路徑的第一個安全點之前發出（[8.4.7](804-safepoint.md#847-安全點不變條件)）。其他函式不在進入處輪詢：由它們的呼叫者輪詢。

## 8.9.12. 影子堆疊框架

也請參考[屏障與根追蹤](../007-runtime/704-barriers-and-roots.md#741-根追蹤由編譯器直接發出的影子堆疊)。編譯器直接發出框架的連上 / 解開，不需要平台的任何協助。

以一個有 2 個 GC 參照字組（`x`、`y`，堆疊槽位）的函式為例。框架容納每個槽位的位址，因此變數指派只是對 `x` 的一般存入。

```
; entry: zero-initialize the words, link the frame
; frame: a stack slot (prev: ptr, count: usize, slots: [ptr; 2])
store.ptr [x], null
store.ptr [y], null
store.ptr [frame + 0], load.ptr [t + 0]    ; prev = the shadow stack head (offset 0 of the TLS block)
store.usize [frame + p], 2                 ; count
store.ptr [frame + 2p], &x                 ; the address, not the value
store.ptr [frame + 3p], &y
store.ptr [t + 0], &frame                  ; link

; variable assignment: no frame update
store.ptr [x], obj

; immediately before every return (including error returns): unlink
store.ptr [t + 0], load.ptr [frame + 0]
```

在 `-O0` 以上，一旦框架配置好，槽位會以更簡短的方式列出字組。`n` 個相鄰字組（三個以上）的一段是項目 `(n << 1) | 1`，其後接第一個字組的位址（位址是偶數，因此以最低位元區分兩者）。兩個以上項目的清單會移到唯讀資料中：框架容納 `count` 0 以及一個槽位，即一個靜態描述子 `{count: u16, entries: [u16; count]}` 的位址，其項目就是槽位的項目，只是每個位址換成字組相對於框架、以字組為單位之偏移的兩倍（一個有號的 16 位元值，與位址一樣是偶數）。進入時於是只存入兩個字組，而不是整份清單。偏移放不進 16 位元的清單會留在框架中。描述子以其內容命名，連結時每一種只保留一份。

```
; the same frame above -O0, its list in read-only data
; desc: u16 2, (x - frame) / p * 2, (y - frame) / p * 2
store.ptr [frame + 0], load.ptr [t + 0]
store.usize [frame + p], 0                 ; a descriptor follows
store.ptr [frame + 2p], &desc
store.ptr [t + 0], &frame
```

`t` 是 [8.3.8](803-tls-slots.md#838-編譯器發出的形式) 中執行緒區域區塊 `nabi_tls` 的位址（[8.7.7](807-code-generation.md#877-tls-存取)）；影子堆疊的開頭是它在偏移 0 的槽位。錯誤回傳也是一般的 `return`（[8.9.6](#896-try--catch--finally以值為基礎的分支)），因此它經過相同的解開。連上也可以比圖示更晚，在每條路徑的第一個安全點之前（[8.4.7](804-safepoint.md#847-安全點不變條件)）；此時只有已連上之路徑的回傳才會解開。函式只有在容納 GC 參照、而且可能到達安全點時（[8.4.2](804-safepoint.md#842-發出位置)）才需要框架。對永不回傳之函式（panic）以及既不配置記憶體也不阻塞之執行環境函式（[8.5.2](805-extern-symbols.md#不是安全點的函式)）的呼叫不算在內。

## 8.9.13. 索引

考慮[索引與切片](../003-expressions/306-indexing-and-slicing.md)中的 `arr[i]`。

```nabi
arr[5]
```

這個運算式變成對標準函式庫函式 `Index::index(arr, 5)` 的呼叫，並像一般函式呼叫一樣降階。

當 `Index` 呼叫被內嵌時（`-O1` 以上），結果是邊界檢查加上一次載入：

```
if 5 >= arr.len then bb_panic else bb_ok
bb_ok:
    val = load.T [arr.data + 5 * size_of(T)]
```

這個結果自然來自泛型單態化與內嵌。

## 8.9.14. 總結：熱路徑的樣子

典型的熱路徑，即 [TLAB](../000-meta/007-glossary.md#tlab執行緒區域配置緩衝區) 配置加上欄位寫入，其成本如下：

- TLAB 游標檢查：load \* 2 / sub / 比較與分支
- 游標更新：store（1 道指令）
- 填寫標頭：store \* 2（type_id、gc_word）
- 填寫欄位：store \* N（欄位數量）
- 寫入屏障（若需要）：load / and / 比較與分支，+（慢速）呼叫
- 安全點輪詢（若需要）：load / 比較與分支

慢速路徑放在主線之外。配置並填寫一個小物件大約需要 10 ~ 20 道指令；在 `-O0`，每個值還會經過其堆疊槽位，而在 `-O0` 以上，放得下的值會留在暫存器中。
