# 8.3. mutator 的 TLS 槽位配置

每個作業系統執行緒（= 排程器工作者 = mutator）都在執行緒區域儲存（TLS）的固定位置保存執行環境資訊。編譯器發出的程式碼與執行環境讀寫相同的 TLS 槽位，因此本文件定義每個槽位的偏移與型別。

## 8.3.1. TLS 槽位結構

TLS 槽位是一個由執行環境定義並匯出的 `thread_local` 區塊 `nabi_tls`；每個槽位位於其中固定的偏移。編譯後的程式碼以相對於該區塊位址的偏移存取每個槽位，而 Nabi 函式從其呼叫者接收該位址（8.3.8）。

`p` 是以位元組計的指標寬度。

| 偏移     | 槽位                        | 型別 | 意義                                    |
|----------|-----------------------------|------|-----------------------------------------|
| 0        | `nabi_tls_shadow_stack_top`  | ptr  | 影子堆疊鏈的開頭                        |
| p        | `nabi_tls_tlab_cursor`       | ptr  | TLAB 中下一個配置位置                   |
| 2p       | `nabi_tls_tlab_limit`        | ptr  | TLAB 的末端                             |
| 3p       | `nabi_tls_wb_log_cursor`     | ptr  | 在寫入屏障記錄中的位置                  |
| 4p       | `nabi_tls_wb_log_limit`      | ptr  | 寫入屏障記錄的末端                      |
| 5p       | `nabi_tls_satb_cursor`       | ptr  | 在 SATB 緩衝區中的位置（並行標記期間）  |
| 6p       | `nabi_tls_satb_limit`        | ptr  | SATB 緩衝區的末端                       |
| 7p       | `nabi_tls_task_worker`       | ptr  | 這個執行緒的執行器工作者（GC 物件，若無則為 0） |
| 8p       | `nabi_tls_stack_guard`       | [usize; 2] | `[lo, hi)`：在此範圍內的錯誤是這個執行緒的堆疊溢位（`[0, 0)` 表示未知，[8.5.7](805-extern-symbols.md#stack-overflow)） |
| 10p      | `nabi_tls_worker_id`         | u64  | 工作者 id                               |
| 10p+8    | `nabi_tls_yield_flag`        | u8   | 是否要在安全點 `yield`                  |
| 10p+9    | `nabi_tls_mutator_state`     | u8   | mutator 狀態                            |

**ABI 契約**是區塊符號 `nabi_tls`，以及每個槽位的偏移、型別與意義。槽位名稱僅供參考；它們不是符號。該區塊以 8 對齊，至少 10p+10 位元組；最後一個 ABI 槽位之後可以接由實作定義的槽位。

**由實作定義的槽位**：執行環境可以在 ABI 槽位之後或其他地方，保存更多的執行緒區域狀態，用於排程、統計、效能剖析、偵錯等。編譯器與外部套件不得參照它們。

> [!NOTE]
> Nabi 的執行環境把下列槽位放在 ABI 部分之後。該區塊為 16p+16 位元組。
>
> - 10p+10：`no_safepoint`（`u8`）。偵錯版 nabic 以 `nabi_nosp_enter` / `nabi_nosp_leave` 包住的 `#[no_safepoint]` 呼叫深度（[8.4.10](804-safepoint.md#8410-native-calls)）。
> - 10p+11：`attached`（`u8`）。執行緒如何加入執行環境：0 表示尚未或已不再加入，1 表示 `nabi_thread_init` 或由執行環境啟動的執行緒，2 表示在 `extern "C"` 定義的 C 進入點附加（8.3.8）。
> - 10p+16：`task_lane`
> - 11p+16：`static_inits`
> - 12p+16：`held_locks`。這個執行緒持有的鎖。在工作階段主機中，已經不存在的執行緒在阻塞時持有的鎖會被釋放。
> - 13p+16：`thread_name`（`{ptr, len}`）。報告中執行緒的名稱；未命名時為 null。
> - 15p+16：`throw_site`。其錯誤正在傳遞中之 `throw` 的、以 NUL 結尾之位置的位址，由編譯後的程式碼在 `throw` 處存入；未知時為 null 或空。

## 8.3.2. 存取機制

每種作業系統與架構都有自己的 TLS 存取機制。

| 平台           | 機制                                  |
|----------------|---------------------------------------|
| Linux x86_64   | 執行緒指標 `%fs:0` + 偏移             |
| Linux aarch64  | 執行緒指標 `tpidr_el0` + 偏移         |
| macOS aarch64  | TLV 描述子，透過其 getter 呼叫        |
| Windows x86_64 | TEB（Thread Environment Block）在 `_tls_index` 的 TLS 陣列 + 偏移 |
| `wasm32-wasip1-threads` | `__tls_base` 全域變數 + 偏移   |
| 其他 WASM（沒有執行緒） | 該區塊是一般的全域變數       |

編譯器依目標與 TLS 模型發出適當的指令（8.3.8）。

## 8.3.3. 各槽位的意義

### `nabi_tls_yield_flag`（u8）

[GC](../000-meta/007-glossary.md#gc-garbage-collector) 為安全點所設定的旗標；mutator 在輪詢時檢查它。位元 0 是執行環境的停止請求，由執行環境設定與清除；其他位元屬於嵌入它的主機（例如 REPL 的中斷），執行環境不會碰它們。只要旗標不為零，輪詢就會呼叫 `nabi_runtime_yield`。

詳細機制定義於[屏障與根](../007-runtime/704-barriers-and-roots.md)中的安全點輪詢。

### `nabi_tls_tlab_cursor`（ptr）

指向[記憶體配置](../007-runtime/702-allocation.md)中 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 下一次配置發生之處。

```
allocate(size):
    tmp = tlab_cursor
    new_cursor = tmp + size
    if new_cursor > tlab_limit:
        slow path
    tlab_cursor = new_cursor
    return tmp
```

這四道指令構成 TLAB 快速路徑，由編譯器以行內方式發出。

### `nabi_tls_tlab_limit`（ptr）

指向 TLAB 的末端。當游標超過這個值時，就進入慢速路徑。

### `nabi_tls_shadow_stack_top`（ptr）

影子堆疊鏈的開頭。詳細定義於 8.3.8。

### `nabi_tls_satb_cursor` / `nabi_tls_satb_limit`（ptr）

並行標記期間由寫入屏障慢速路徑記錄的 SATB 緩衝區（[7.5](../007-runtime/705-concurrent-collection.md)）。滿了之後交給回收器。標記之外不使用。

### `nabi_tls_wb_log_cursor` / `nabi_tls_wb_log_limit`（ptr）

[屏障與根追蹤](../007-runtime/704-barriers-and-roots.md)中寫入屏障記錄緩衝區的游標與上限。

```
write_barrier(p, q):
    if !needs_log: return
    if wb_log_cursor >= wb_log_limit: flush
    *wb_log_cursor++ = p
```

大多數操作都是非原子且執行緒區域的。

這兩個槽位指向一個每個執行緒一份、512 字組的記錄區塊（在 64 位元目標上為 4 KiB，在 32 位元上為 2 KiB）。已滿的區塊會被推入回收器會取出的無鎖堆疊（[7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-object-remembering)）。

### `nabi_tls_mutator_state`（u8）

mutator 目前的狀態。

| 值    | 意義                          |
|-------|-------------------------------|
| `0`   | RUNNING。正常執行             |
| `1`   | GC_COOPERATING。與 GC 協作中  |
| `2`   | PARKED。在睡眠 / 等待中       |
| `3` ~ | 保留                          |

這個值用於 GC 週期的同步與安全點協作。

### `nabi_tls_worker_id`（u64）

執行緒的工作者 id（在排程器內部使用），用於[工作竊取](../000-meta/007-glossary.md#work-stealing)、統計等。

## 8.3.4. 槽位的寬度與對齊

每個槽位都有其型別的寬度，並位於以其對齊的偏移上。

## 8.3.5. 32 位元與 64 位元

指標槽位是 p 位元組（64 位元上為 8，32 位元上為 4），因此偏移在 32 位元與 64 位元之間依表格而不同。

## 8.3.6. TLS 槽位的初始化

工作者啟動時，槽位依下列方式初始化。

```
nabi_thread_init():
  nabi_tls_yield_flag = 0
  nabi_tls_tlab_cursor = NULL          // filled by the slow path on first allocation
  nabi_tls_tlab_limit = NULL
  nabi_tls_shadow_stack_top = NULL     // empty shadow stack chain (8.3.8)
  nabi_tls_wb_log_cursor = wb_log_base
  nabi_tls_wb_log_limit = wb_log_base + LOG_SIZE
  nabi_tls_mutator_state = RUNNING
  nabi_tls_worker_id = (allocated)
  nabi_tls_task_worker = 0
  nabi_tls_stack_guard = [guard_lo, guard_hi)   // guard range below the stack
```

排程器在工作者啟動時呼叫 `nabi_thread_init`；這個函式是 [extern 符號](805-extern-symbols.md)的一部分。

## 8.3.7. TLS 槽位的拆除

工作者結束時，會經過下列步驟。

```
nabi_thread_shutdown():
  // log flush
  // shadow stack walk (and a final mark, if possible)
  // return worker_id
```

## 8.3.8. 編譯器發出的形式

該區塊是由執行環境定義並匯出的 `thread_local` 全域變數。編譯器以外部宣告參照它，並以相對於該區塊位址的偏移載入槽位。Nabi 函式以隱藏的最後一個引數從其呼叫者取得該位址（8.6.6），因此 Nabi 函式之間的呼叫不會讀取 TLS。從 Nabi 之外進入的程式碼自己取得該位址一次，並往下傳遞：C 的 `main`、`extern "C"` 定義的 C 進入點、`#[export_name]` 的 Nabi 函式、`any` 分派器、延遲 `static` 的初始化式，以及函式值所經過的 thunk（8.6.11）。

`extern "C"` 定義的 C 進入點，可能在由 C 程式碼啟動、執行環境不認識的執行緒上執行。在這樣的執行緒上，它會先附加該執行緒：像 `nabi_thread_init` 一樣初始化槽位（8.3.6），並把該執行緒註冊為 mutator，然後才執行主體。在這類呼叫之間回到 C 時，該執行緒算作 `PARKED`（[8.4.10](804-safepoint.md#8410-native-calls)），因此回收不會等待它；而當該執行緒結束時，它會像 `nabi_thread_shutdown` 一樣離開登錄（8.3.7）。無法以這種方式附加執行緒的目標（WebAssembly）則改為中止，並印出 `nabi: callback on a thread nabi does not know; call nabi_thread_init() first`。

```
; nabi_tls: defined by the runtime; compiler output refers to it as an undefined symbol
t = <address of nabi_tls>         ; once, where code enters nabi
                                  ; (a nabi function: its last argument)

; safepoint poll (yield_flag at 10p+8)
flag = load.u8 [t + 88]           ; 10p+8 with p = 8
if flag != 0 then bb_slow else bb_cont
```

在 ELF 上，`nabi_tls` 使用 initial-exec TLS 模型（編譯器以這個模型發出存取程式碼，不依賴連結器的放寬處理）。執行環境一律靜態連結進執行檔，因此這個模型就足夠了。Mach-O 使用平台的 TLV 描述子機制，Windows 則使用執行緒在 `_tls_index` 的 TLS 陣列。無法選擇 TLS 模型的實作可以改用 general-dynamic。

### 影子堆疊的開頭

`nabi_tls` 偏移 0 的槽位（`nabi_tls_shadow_stack_top`）是[屏障與根追蹤](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)中[影子堆疊](../000-meta/007-glossary.md#shadow-stack)鏈的開頭，即最內層 `ShadowFrame` 的位址（若無則為 null）。它是唯一的鏈開頭。編譯器發出的程式碼在函式中每條路徑的第一個安全點之前連上框架，並在連上了它之路徑的每個 `return` 之前立即解開它（8.4.7）。

## 8.3.9. 穩定性

8.3.1 表中的偏移、型別與意義是 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 的一部分；其變更遵循[概觀](README.md#stability)中的穩定性方針。改變偏移，或改變既有槽位的型別或意義，會破壞 ABI（1.0 之前是 0.x.0 版本，從 1.0 起是主版本）。新的槽位取既有槽位之後的偏移，這是相容的變更。

由實作定義的槽位不屬於 ABI，可能在任何版本中改變。
