# 8.11. 程式碼產生器契約

本節彙整程式碼產生器必須保證的事項，好讓它寫出的目的檔能與 `nabic` 的連結器連結、在執行環境上執行，並與其他程式碼產生器寫出的目的檔混用：包括 `nabic` 自己的（[8.7](807-code-generation.md)），以及另外安裝的（8.11.13）。其中大部分定義於本章的其他地方；本節為每項要求命名，並連結到其定義。當 `nabic` 的程式碼產生器做的比契約要求的更多時，內文會註明。

> [!NOTE]
> 0.2 的草案。在 0.1 中，`nabic` 自己的程式碼產生器是唯一的一個，下列每項要求都是它所做的事。8.11.13 的行程介面以及它所讀取的檔案格式（[9.11](../009-tools/911-mir-format.md)）尚未實作：0.1 的 `nabic` 不會執行另外的程式碼產生器，`-C codegen-backend` 是未知的 codegen 選項。

## 8.11.1. 分工

程式碼產生器把一個 codegen 單元（[9.2.16](../009-tools/902-aot-pipeline.md#codegen-units)）的函式轉成一個可重定位的目的檔。在任何程式碼產生器執行之前，`nabic` 決定一切需要多個目的檔一致同意的事，並把結果連同該單元的 nabi-mir 交出：

| `nabic` 決定 | 程式碼產生器決定 |
|-----------------|----------------------------|
| nabi-mir 的最佳化（內嵌，[8.7.3](807-code-generation.md#873-optimization-levels)）；協程狀態機（[8.10](810-lowering-coroutine-generic.md)） | 指令選擇、暫存器配置、排程、單一函式內的框架配置 |
| 各單元，以及每個單元定義哪些函式 | 在 8.11.6 與 8.11.7 的規則之內它自己的最佳化 |
| 每個函式與 static 的符號、其連結性與可見性（8.11.3） | 它自己的區域符號（8.11.3） |
| 每個型別的配置（8.11.5） | |
| GC 型別 id 及其 ScanTable（[8.2](802-scan-table.md)） | |
| 哪些函式永不到達安全點、哪些在進入處輪詢、哪些回邊輪詢（8.11.7） | 輪詢在被輪詢之邊上的位置 |
| 共用單元：C 的 `main`、static、`nabi.register_types`、lazy static 的初始化器（[9.2.16](../009-tools/902-aot-pipeline.md#codegen-units)） | |
| 連結（[9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables)） | |

共用單元一律由 `nabic` 自己的程式碼產生器編譯。同一個程式中不同程式碼產生器的目的檔，以及由另一個程式碼產生器重新編譯之同一單元的目的檔，都必須能夠連結，而且行為相同。

## 8.11.2. 輸入

輸入是一個單元最佳化後的 nabi-mir（[9.11](../009-tools/911-mir-format.md)）：該單元所定義的函式及其主體；它們所參照之每個函式與 static 的宣告；它們所使用的型別、配置與 GC 型別；目標；以及建置的選項（`-O` 等級、溢位檢查、`-g`）。

這個 nabi-mir 就是 `--emit=mir` 所印出的那個（[9.9.1](../009-tools/909-debug-tools.md#991-intermediate-output---emit)）。它沒有協程暫停點（此時協程已是作用於框架物件的狀態機，[8.10.1](810-lowering-coroutine-generic.md#8101-state-machines)），也沒有向下取整的 `%%`（它已被展開）。程式碼產生器以每個陳述式在 nabi-mir 中的意義編譯它，並依程式順序保留每個值、每個 trap 與每個副作用，與 [8.7.3](807-code-generation.md#873-optimization-levels) 對最佳化器的要求相同。

## 8.11.3. 符號與連結性

每個函式與 static 在輸入中都帶有其符號：修飾後的名稱（[9.5](../009-tools/905-symbol-mangling.md)）、`#[export_name]`，或 `extern` 宣告的 C 名稱（在 i686 Windows 上帶有其 `@N` stdcall 後綴）。程式碼產生器原樣使用這些名稱，不進行修飾。它完全依照輸入所說的，把符號定義為 global、weak 或 hidden；其他單元會呼叫的函式是 global，另一個套件之程式碼的私有複本命名為 `<symbol>.copy_Y<hash>_`，且只在該套件的單元內是 global。

`nabic` 與其程式碼產生器也使用下列衍生符號：

| 符號 | 定義者 | 它是什麼 |
|--------|-----------|------------|
| Nabi 函式的 `<symbol>` | 定義該函式的單元 | 主體，採用帶有隱藏 `nabi_tls` 引數的 Nabi 慣例（[8.6.6](806-calling-convention.md#866-nabi-to-nabi-calling-convention)） |
| `<symbol>.c` | 定義沒有 `#[export_name]` 之 `extern "C"` 函式的單元（有的話則為匯出名稱） | 其 C 進入點（8.11.8） |
| `<symbol>.v` | 每個取得該函式位址的目的檔，作為區域符號 | 函式值所呼叫的 thunk：沒有 `nabi_tls` 引數；它讀取區塊的位址並呼叫 `<symbol>`（[8.6.11](806-calling-convention.md#8611-function-pointer-abi)） |
| `<static>.state`、`<static>.init` | 共用單元 | lazy static 的狀態位元組與初始化器（8.11.10） |
| `_Y<pkg>_gctype.<hash>` | 單元建置中的共用單元 | 容納 GC 型別 id 的 `u32`（8.11.6） |
| `nabi.scan.<hash>` | 共用單元，weak 且 hidden | ScanTable 的位元組，以其內容命名（[8.2.13](802-scan-table.md#8213-registration-time)） |
| `nabi.register_types`、`nabi.register_types.<lib>` | 共用單元 | GC 型別與 static 根的註冊（[8.2.13](802-scan-table.md#8213-registration-time)） |
| `nabi.any.<hash>` | `any` 分派器的單元；在函式庫中為 weak | `any` 方法分派器（[9.4.8](../009-tools/904-nabihir-bytecode.md#948-rules-for-separate-compilation)） |
| `nabi.str.<hash>` | 任何目的檔，作為區域符號 | 字串字面值的位元組，其後接一個 NUL 位元組 |
| `nabi_tls` | 執行環境 | 執行緒區域區塊（[8.3](803-tls-slots.md)） |

程式碼產生器自己產生的其他任何符號都是其目的檔的區域符號。`nabic` 自己的這類符號名稱以 `nabi.` 開頭（`nabi.i386.udivmod64`……）。

每個符號名稱的位元組都必須原樣、未經壓縮地出現在目的檔中：`nabic` 以套件雜湊的預留位置寫出單元的目的檔，並在連結之前把目的檔位元組中的 `_Y<placeholder>_` 換成 `_Y<hash>_`（[9.2.16](../009-tools/902-aot-pipeline.md#codegen-units)）。

字串字面值的位元組以一個不屬於 `str` 的 NUL 位元組結尾：擲回處（[8.11.11](#81111-errors-panics-and-unwinding)）以及作為 `char*` 傳遞的 `str` 都讀到它為止。

## 8.11.4. 呼叫慣例

- Nabi 函式之間：[8.6.6](806-calling-convention.md#866-nabi-to-nabi-calling-convention)。Nabi 函式可能從其他目的檔收到的每次呼叫都完全遵循它，包括作為最後一個引數的隱藏 `nabi_tls` 位址；超過 64 位元組的聚合型別以位址傳遞，並透過開頭的結果指標傳回。只有其他目的檔無法到達的函式（區域的、從未作為值使用）才可以使用自己的慣例。
- `extern "C"` 宣告與定義，以及經由 C 函式指標的呼叫：平台的 C ABI，包括聚合型別分類（[8.6.1](806-calling-convention.md#861-default-platform-c-abi) ~ [8.6.3](806-calling-convention.md#863-struct-arguments--return-small-struct-rule)）、窄整數擴充、i686 Windows 上 `extern "system"` 的 stdcall，以及 C 可變參數（[8.6.10](806-calling-convention.md#8610-variadic-functions)）。輸入會給出 `nabic` 對每個這類簽章所做的分類（[9.11.9](../009-tools/911-mir-format.md#9119-functions)）；程式碼產生器放置已分類的各部分，不重新分類。
- `extern "C"` 呼叫的 `str` 引數以 `char*` 傳遞：程式碼在呼叫之前呼叫 `nabi_str_cstr(s)`，傳遞它所回傳的指標，並在呼叫之後呼叫 `nabi_str_cstr_done(s, p)`（當 `s` 的位元組之後接著 NUL 位元組時，該指標就是 `s` 自己的，否則是一份釘住的複本）。
- `extern "nabi"` 執行環境宣告：C ABI，`str` 作為以值傳遞的 `(ptr, len)` C 結構（[8.5](805-extern-symbols.md)）。
- 函式值：`{code, env}`，環境在前，沒有 `nabi_tls` 引數（[8.6.11](806-calling-convention.md#8611-function-pointer-abi)、[8.6.12](806-calling-convention.md#8612-closure-calls)）。
- 會擲回錯誤的函式回傳其承載結構 `(is_err, payload)`（[8.6.5](806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)）；在 nabi-mir 中，那是輸入所給之型別的一般回傳值。
- 堆疊探測遵循目標的 `stack_probes`（[0.6.5](../000-meta/006-platform-targets.md#065-target-specifications)）：一頁以上的框架依序碰觸每一頁，以行內方式或經由 `__chkstk`（Windows；由 `nabic` 的啟動目的檔定義）。執行環境把保護範圍內的錯誤回報為堆疊溢位（[8.5.7](805-extern-symbols.md#stack-overflow)）。

## 8.11.5. 資料配置

每個 nabi-mir 型別的配置由目標固定，而輸入會給出每個型別的大小、對齊與欄位偏移（[9.11.6](../009-tools/911-mir-format.md#9116-types)）。程式碼產生器原樣使用它們：

| nabi-mir 型別 | 配置 |
|---------------|--------|
| `()` | 大小 0，對齊 1 |
| `i8` `u8` `bool` `c_char` | 1 位元組；`bool` 是 0 或 1 |
| `i16` `u16` | 2 位元組 |
| `i32` `u32` `f32` `char` | 4 位元組；`char` 是 Unicode 純量值 |
| `i64` `u64` `f64` | 8 位元組，對齊到目標的 `wide_align`（i686 Linux 上為 4，其他為 8） |
| `isize` `usize`、GC 參照、函式指標、未解析的字組 | 一個指標寬度的字組 |
| `str` | 兩個字組 `(ptr, len)`；`ptr` 可以指向 GC 物件內部 |
| 元組、結構 | 依序排列各欄位的 C 配置；`#[repr(packed)]`：沒有填充，對齊 1；`#[repr(align(N))]`：至少 `N` |
| `[T; N]` | `N` 個 `T` 大小的元素 |
| `n` 個字組的 union | `n` 個字組，對齊到其對齊最大的變體，且至少對齊到一個字組 |
| 和型別、列舉 | `(i32 tag, payload)` 或 niche 配置（[8.8.6](808-lowering-types.md#886-representation-of-enum)、[8.8.8](808-lowering-types.md#888-representation-of-sum-types-a--b)） |
| `any Tr` | `(u32 tag, payload word)`（[8.8.9](808-lowering-types.md#889-representation-of-any-slots)） |
| 函式值 | `(code, env)` |

除了 `s390x` 以外，所有目標都是小端序。GC 物件是 16 位元組的標頭（[8.1](801-object-header-abi.md)）後接其酬載；其大小如 [8.1.7](801-object-header-abi.md#817-total-size-of-header--payload) 所述。

## 8.11.6. GC：配置、根與屏障

**配置。** 程式碼產生器所配置的每個 GC 物件（nabi-mir 中的 `HeapBox`、`HeapAlloc`、`BufAlloc`）都取得輸入隨之給出的型別 id：一個常數，或在單元建置中位於 `_Y<pkg>_gctype.<hash>` 的 `u32`。不比字組寬的數字、`bool` 或 `char` 的 `HeapBox` 不配置任何東西：酬載字組就是該值（[8.2.7](802-scan-table.md#827-any-slots)）。配置是 [8.9.9](809-lowering-control.md#899-gc-alloc-fast--slow-path) 的 TLAB 快速路徑，或對 `nabi_gc_alloc_slow(size, align, type_id)` 的呼叫；`LOS_THRESHOLD`（8 KiB）以上或對齊超過 8 的物件一律呼叫它。快速路徑寫入 `type_id` 與 `gc_word`，並讓 `gc_flags` 保持為零；酬載為零。程式碼產生器一律可以改為呼叫 `nabi_gc_alloc_slow`。

**根。** 在每個安全點（8.11.7），每個跨越該安全點仍存活之區域變數中可能存有 GC 參照的每個字組，都位於記憶體中，並註冊在連結到 `nabi_tls` 偏移 0 的影子堆疊框架中（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)、[8.9.12](809-lowering-control.md#8912-shadow-stack-frame)）：

- 框架是位於機器堆疊（在 wasm32 上是線性記憶體）上的 `{prev: ptr, count: usize, slots: [ptr; count]}`；每個槽位容納一個字組的位址，或是一個槽位 `(n << 1) | 1` 加上下一個槽位存放 `n` 個相鄰字組中第一個的位址。`count` 為 0 時，框架有一個槽位：唯讀資料中一個描述子 `{count: u16, entries: [u16; count]}` 的位址，其項目就是槽位的項目，只是每個位址換成字組相對於框架、以字組為單位之偏移的兩倍，有號（[8.9.12](809-lowering-control.md#8912-shadow-stack-frame)）。區域變數所需的字組，就是輸入作為該型別之根偏移所給出的那些：任何變體可能存有參照的每個字組、`any` 的酬載字組、`str` 的指標字組。
- 框架在每條路徑的第一個安全點之前連上，並在已連上它之路徑的每次回傳之前解開；在路徑匯合處，要嘛全部都已連上，要嘛全都沒有。只要回收可能讀取已註冊的字組，它就是零或有效的值：在連上之前清零，並在每次寫入時保持最新。
- 跨越安全點仍存活的東西：在它之後仍存活的區域變數；在呼叫執行環境函式或 C 時，還包括其運算元；在呼叫 Nabi 函式時，只包括呼叫者之後會讀取的東西（被呼叫者自己為其參數建立根）。位址被取得的區域變數（`AddrOf`），以及函式列為釘住的每個區域變數，在其 `StorageDead` 之前都保持註冊。
- 只有回邊輪詢會看到的區域變數，可以改為只由輪詢的慢速路徑註冊（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler) 規則 4）。`nabic` 的程式碼產生器在進入處連上一個框架，並把每個這類區域變數都註冊在其中。
- 已死亡之區域變數的字組可以清為零（`StorageDead`，即其最後一次使用的結尾）。這避免它留住垃圾；這不是必要的。

**暫存器與移動的物件。** 完整回收可能會移動物件（疏散，[7.3.1](../007-runtime/703-gc-algorithm.md#731-immix-mark-region)）；它改寫堆積中每個精確的參照，但絕不改寫根字組。當已註冊的字組、模稜兩可的字組或釘選到達某個物件時，該物件留在原處，而大型物件絕不移動。因此：

- GC 參照只有作為跨越該安全點已註冊之字組的複本時，才可以跨越安全點留在暫存器中；指向該物件內部的指標也是如此。
- 其他任何 GC 參照或指向 GC 物件內部的指標，例如從堆積物件欄位載入的參照，都不會跨越安全點存活：之後會重新載入它。程式碼產生器不得把這樣的值保留在跨越安全點時未註冊的暫存器、溢出槽位或堆疊槽位中。
- `nabic` 的程式碼產生器不把任何根保留在暫存器中：跨越安全點容納 GC 參照的變數，存在於其已註冊的堆疊槽位中。

**寫入屏障。** 每次把帶有 GC 字組的值存入 GC 物件（堆積物件的欄位、緩衝區元素、協程框架槽位），都帶有 [8.9.10](809-lowering-control.md#8910-write-barrier) 的屏障：測試目的物件的 `gc_flags & (UNLOGGED | CARDED)`；若為 `UNLOGGED`，呼叫 `nabi_gc_wb_slow(obj, slot, len)`（槽位的位址與被存入之值的大小）；若為 `CARDED`，在 `card_table + ((slot - obj) >> 9)` 處把 `1` 存入被寫入之槽位所跨越的卡片位元組，卡片表的位址是標頭之前的字組。屏障與其存入之間沒有安全點。在下一個安全點之前對剛配置之物件的存入、對 static 的存入、對原始記憶體的存入，以及沒有 GC 字組之值的存入，都不需要屏障。屏障絕不讀取被存入之值的標頭。

回收器對編譯後的程式碼沒有其他要求（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)）：沒有堆疊對應表，也沒有暫存器對應表。

## 8.11.7. 安全點與輪詢

安全點是執行緒可能為了回收而停下的點：對輸入未標示為永不到達安全點之 Nabi 函式的呼叫、經由函式值的呼叫、對非 `#[no_safepoint]` 之執行環境函式的呼叫、對 C 的呼叫（它會停駐，8.11.8）、配置、對 lazy static 的讀取、寫入或取址，以及輪詢。

- **輪詢。** 載入位於 `nabi_tls + 10p+8` 的 `u8`（`yield_flag`）；若它非零，呼叫 `nabi_runtime_yield()`（[8.4.1](804-safepoint.md#841-yield-flag-check-fast-path)）。
- **回邊。** 輸入標示為被輪詢的每個回邊上都有輪詢（[8.4.8](804-safepoint.md#848-loop-back-edge-poll)）。未標示的屬於最多 64 次迭代、不到達任何安全點的迴圈（[8.4.2](804-safepoint.md#842-emission-sites)）。
- **進入處。** 輸入標示為可重入的函式，在每條路徑的第一個安全點之前輪詢；而當它也會以直接的 Nabi 呼叫以外的方式進入（從 C、經由函式值、匯出、`any` 分派器），且確實會到達安全點時，在進入處輪詢（[8.4.7](804-safepoint.md#847-safepoint-invariant)、[8.9.11](809-lowering-control.md#8911-safepoint-poll)）。
- 輸入標示為永不到達安全點的函式不需要框架，也不需要輪詢，而對它的呼叫不是安全點。

允許額外的輪詢。向量化的迴圈內部沒有輪詢，最多執行 1024 次迭代，然後純量迴圈的輪詢會再次執行（[8.7.8](807-code-generation.md#878-loop-vectorization)）。

## 8.11.8. 呼叫 C 與從 C 呼叫

- **呼叫 C。** 在每次呼叫符號不以 `nabi_` 開頭的 `extern "C"` 函式前後：之前，以循序一致的方式把 `2`（PARKED）存入 `mutator_state`（`nabi_tls + 10p+9`），然後載入 `yield_flag`，若非零則呼叫 `nabi_enter_native()`；之後，存入 `0`（RUNNING），進行相同的載入，若非零則呼叫 `nabi_leave_native()`（[8.4.10](804-safepoint.md#8410-native-calls)）。呼叫者的根在呼叫期間保持註冊。
- **從 C 呼叫。** `extern "C"` 定義有一個 C 進入點（`<symbol>.c` 或其匯出名稱），它接受 C 引數，呼叫 `nabi_native_reenter()`（它也會附加執行環境不認識的執行緒，[8.3.8](803-tls-slots.md#838-form-of-compiler-emission)），以這個執行緒的 `nabi_tls` 位址呼叫主體，呼叫 `nabi_native_resume(<reenter 所回傳的值>)`，然後以 C 的方式回傳。在 wasm32 上，當函式被匯出時，它會先呼叫 `nabi_init_once()`。
- 對 `#[no_safepoint]` 執行環境函式以及 `nabi_*` 符號的呼叫不加上這些前後處理。

## 8.11.9. 執行緒區域存取

Nabi 函式從它作為最後一個引數收到的位址，在其偏移上讀取 `nabi_tls` 槽位（[8.3.1](803-tls-slots.md#831-tls-slot-structure)）。從 Nabi 之外進入的程式碼（C 進入點、函式值 thunk、匯出的 Nabi 函式、`any` 分派器）計算該位址一次（[8.7.7](807-code-generation.md#877-tls-access)）。存取序列，因而包括重定位，取決於目的檔格式：

| 目標 | 模型 | `nabic` 使用的重定位 |
|--------|-------|--------------------------|
| ELF x86_64 / aarch64 / i686 / armv7 / riscv64 / powerpc64le / s390x | initial exec，執行緒指標加上來自 GOT 槽位的偏移 | `R_X86_64_GOTTPOFF`、`R_AARCH64_TLSIE_*`、`R_386_TLS_GOTIE`、`R_ARM_TLS_IE32`、`R_RISCV_TLS_GOT_HI20`、`R_PPC64_GOT_TPREL16_*`、`R_390_TLS_IEENT` |
| ELF riscv32（沒有作業系統） | local exec | `R_RISCV_TPREL_HI20` / `_ADD` / `_LO12_I` |
| Mach-O arm64 | 執行緒區域變數描述子，經由其 getter 呼叫 | `ARM64_RELOC_TLVP_LOAD_PAGE21` / `_PAGEOFF12` |
| COFF | 位於 `_tls_index` 的 TEB TLS 陣列，加上相對於區段的偏移 | `SECREL`（arm64：`SECREL_HIGH12A` / `_LOW12A`） |
| 有執行緒的 wasm32 | `__tls_base` 加上偏移 | `R_WASM_MEMORY_ADDR_TLS_SLEB` |
| 沒有執行緒的 wasm32 | 一般的 global | 記憶體位址 |

執行環境一律以靜態方式連結，因此這些模型就足夠了；在連結器能解析之處，程式碼產生器可以改用 general dynamic。

## 8.11.10. static

static 的儲存空間、初始位元組與根註冊屬於共用單元。程式碼產生器經由其符號讀寫 static；當目的檔格式有 GOT 且 static 定義在程式自己的目的檔之外（在預先建置的函式庫或 C 函式庫中）時，經由 GOT；在 Windows 上，對 DLL 所定義的 static 則經由 `__imp_<symbol>`。對 static 的存入沒有寫入屏障（static 是根）。

在第一次讀取、寫入或取得 lazy static 的位址之前，程式碼以原子方式載入其狀態位元組 `<static>.state`，除非它是 `2`（已初始化），否則以存取處的原始碼位置（`path:line:col`）呼叫 `<static>.init(at: str)`，採用不帶 `nabi_tls` 引數的 Nabi 慣例。這是一個安全點。

## 8.11.11. 錯誤、panic 與展開

Nabi 程式中沒有任何東西會展開機器堆疊。

- `throw` 與 `try` 是對承載結構的一般回傳與分支（[8.6.5](806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation)、[8.9.6](809-lowering-control.md#896-try--catch--finally-value-based-branching)）；每個框架都經由它自己的回傳路徑離開，並解開其影子堆疊框架。
- `throw` 以呼叫 `nabi_throw_site(at: str)` 記錄其位置；`nabic` 的程式碼產生器則改為把字面值的位址存入執行環境所讀取之由實作定義的槽位（[8.3.1](803-tls-slots.md#831-tls-slot-structure)），程式碼產生器只有搭配與它一起發行的執行環境時才可以這樣做。
- panic 是對不回傳之函式（`nabi_panic_at`……）的呼叫；它結束行程，不執行任何 `finally`。程式碼產生器自己發出的檢查經由 `nabi_panic_at(msg, at)` panic，`at` 是正在執行之陳述式的位置（`path:line:col`），或在 `#[track_caller]` 函式中是其最後一個參數；執行環境依 [4.14.13](../004-functions/414-throws.md#41413-panics-and-fatal-errors) 所示的方式印出它。檢查及其訊息：

| 檢查 | 時機 | 訊息 |
|-------|------|---------|
| 整數 `+` `-` `*` 溢位 | 開啟溢位檢查、不是 `wrapping_*`、運算元不是常數安全的 | `attempt to add with overflow`、`attempt to subtract with overflow`、`attempt to multiply with overflow` |
| 對最小值的整數取負 | 開啟溢位檢查 | `attempt to negate with overflow` |
| 除法與餘數 | 一律 | `attempt to divide by zero`、`attempt to divide with overflow`、`attempt to calculate the remainder with a divisor of zero`、`attempt to calculate the remainder with overflow` |
| 移位達寬度以上 | 一律 | `attempt to shift left with overflow`、`attempt to shift right with overflow` |
| 陣列索引（`ArrayIndex`、`ArrayWrite`） | 一律，除非索引是範圍內的常數 | `index out of bounds` |

其他檢查（切片、`List`、`unwrap`）在 nabi-mir 中已經是呼叫。

展開表是給偵錯工具、效能分析工具與作業系統用的，不是給程式用的。程式碼產生器為每個函式發出其目的檔格式的展開資訊：ELF 上的 `.eh_frame` FDE（armv7：`.ARM.exidx` 項目）、Mach-O 上的 compact unwind 或 `__eh_frame`、COFF x86_64 與 arm64 上的 `.pdata` / `.xdata`（i686 沒有；框架經由 `ebp` 走訪）。沒有任何 CIE 指名 personality 常式，也不發出 LSDA；在 armv7 上，精簡的 `.ARM.exidx` 項目隱含 `__aeabi_unwind_cpp_pr0`，由執行環境以弱符號定義。`nabic` 的程式碼產生器在每個函式中都保留框架指標，因此效能分析工具不需要展開表也能走訪堆疊。

## 8.11.12. 目的檔

程式碼產生器以目標的格式寫出一個可重定位的目的檔，由 `nabic` 的連結器讀取（[9.2.14](../009-tools/902-aot-pipeline.md#9214-linking-for-executables)）：

| 格式 | 要求 |
|--------|--------------|
| ELF | `ET_REL`，目標 ABI 的機器與旗標（powerpc64le：ELFv2，`e_flags` 2；armv7：EABI5 硬體浮點；riscv：RVC，在 riscv64 上還有雙精度浮點 ABI）。64 位元目標：只用 RELA 重定位；32 位元：REL 或 RELA。沒有壓縮的已配置區段（`SHF_COMPRESSED` 偵錯區段會被捨棄）。一個不帶 `SHF_EXECINSTR` 的 `.note.GNU-stack` 區段 |
| Mach-O | `MH_OBJECT`，僅限 arm64，`MH_SUBSECTIONS_VIA_SYMBOLS` |
| COFF | 目標的機器；i686 的 C 名稱帶有 `_` 前綴；超過 65279 個區段時使用 `/bigobj` 格式 |
| wasm | 遵循 WebAssembly 工具慣例（`linking` 區段第 2 版、`reloc.CODE` / `reloc.DATA`）的可重定位模組，從 `env` 匯入 `__linear_memory`、`__stack_pointer`、`__indirect_function_table`，以及有執行緒時的 `__tls_base`；每個函式的型別是其慣例所給出的型別，`nabi_tls` 位址是最後一個 `i32` 參數 |

- 每個函式與每個資料物件都應當位於自己的區段（ELF）、COMDAT（COFF）、子區段（Mach-O）或自己的區段（wasm 的 segment）中：連結器會捨棄未被參照的部分，並在 `release` 建置中合併相同的函式（[9.2.15](../009-tools/902-aot-pipeline.md#9215-output)）。共用一個區段的東西會一起保留或捨棄，且絕不被合併。
- 只能出現連結器能解析的重定位種類：`nabic` 的程式碼產生器所寫出的，以及工具鏈所附之執行環境函式庫與 C 函式庫中的。其他任何種類都是指出其名稱的連結錯誤（`unsupported relocation ..`）。
- 指令集所缺少的常式是對平台支援常式的呼叫，由執行環境函式庫提供：在 armv7 上是 `__aeabi_idiv`、`__aeabi_ldivmod`、`__aeabi_l2d`、`__mulodi4`……；在 riscv32 上是軟體浮點常式與 `__divdi3`……（[8.7.4](807-code-generation.md#874-operations)）。
- `sqrt` 以外的浮點數方法是輸入中已有的 `nabi_f64_*` / `nabi_f32_*` 呼叫；程式碼產生器不在編譯期摺疊它們。
- 輸出是確定性的：相同的輸入產生相同的位元組。

**偵錯資訊**是選用的。當建置有 `-g` 時，程式碼產生器應當像 [9.9.6](../009-tools/909-debug-tools.md#996-debugger-integration-gdb--lldb) 那樣描述其函式：以其 Nabi 路徑命名的函式（[9.5.10](../009-tools/905-symbol-mangling.md#9510-debug-information)）、來自陳述式位置的行號表，以及帶有輸入所描述之型別的參數與具名區域變數。連結器讀取 32 位元格式的 DWARF 2 到 4（`.dSYM` 套組會略過帶有 DWARF 5 或 64 位元 DWARF 的目的檔），在 COFF 上則讀取 `.debug$S` / `.debug$T` 中的 CodeView C13，並把它合併進 PDB。偵錯資訊不得改變程式碼。

## 8.11.13. 另外安裝的程式碼產生器

> [!NOTE]
> 0.2 的草案。本小節的內容皆尚未實作。

`nabic` 自己以外的程式碼產生器是另外的執行檔，即以 `nabi component add <component>` 安裝的工具鏈元件：`codegen-llvm` 安裝 `nabic-codegen-llvm`，其中包含它自己以靜態方式連結、固定版本的 LLVM 複本。`nabic` 本身不包含其他程式碼產生器，也不連結任何外部程式碼產生器（[9.2.17](../009-tools/902-aot-pipeline.md#9217-separation-of-compiler-and-runtime)）。

**選擇。** `-C codegen-backend=<name>`（`native` 是預設值，或元件的名稱，例如 `llvm`），或 `nabi.nabion` 之設定檔中的 `codegen_backend: "<name>"`（[9.10.3](../009-tools/910-nabi-frontend.md#9103-nabinabion--nabilock)），選項會覆寫它。這個選擇適用於建置所編譯的每個單元；預先建置的函式庫保留其既有的目的檔，而共用單元、REPL 儲存格與 Jupyter 儲存格一律是 `native`。其元件未安裝的名稱是錯誤，錯誤訊息會指出安裝它的指令：``codegen backend `llvm` is not installed; run `nabi component add codegen-llvm` ``。元件為 `nabic` 支援的每個目標編譯，或說明它不支援哪些（`E0129`）。

**核心元件。** 元件可以只編譯部分函式：`codegen-mlir` 編譯函式庫為它標示的函式，而單元的其餘部分仍由所選的程式碼產生器處理。`nabic` 把這類函式放進它們自己的單元。函式如何被標示尚未規定。

**尋找它。** `nabic` 從它自己的 `bin/` 目錄（工具鏈的目錄，[9.2.19](../009-tools/902-aot-pipeline.md#9219-installation-layout)）執行 `nabic-codegen-<name>`；它不搜尋 `PATH`。元件依工具鏈安裝，其版本就是工具鏈的版本。

**呼叫。** 一個行程編譯一個單元：

```
nabic-codegen-<name> <unit>.nabimir -o <unit>.o
```

- `<unit>.nabimir` 是該單元以 [9.11](../009-tools/911-mir-format.md) 格式表示的 nabi-mir。建置的選項都在其中；沒有其他引數。
- `-o` 指名要寫出的目的檔。`nabic` 只在結束狀態為 0 之後才讀取它；在其他任何狀態下，它忽略已寫出的任何內容。
- `nabic` 同時執行與其核心數一樣多的行程，並以 `--keep` 保留失敗單元的輸入（[9.1.2](../009-tools/901-nabic-cli.md#nabic-build-aot-build)）。
- 工作目錄是建置的目錄；環境是 `nabic` 的環境。

**診斷。** 行程把其診斷寫到標準輸出，每行一個 JSON 物件，格式與 `--message-format=json` 相同（schema 1，[9.9.4](../009-tools/909-debug-tools.md#json-format-tool-integration)）。其範圍是輸入之陳述式的位置。`nabic` 把它們當作自己的診斷，以建置所要求的格式印出。它無法編譯的構造是指出其名稱的錯誤 `E0129`，位於使用它的陳述式；不會默默改用另一個程式碼產生器。標準錯誤原樣傳遞。

**結束狀態。**

| 狀態 | 意義 |
|--------|---------|
| `0` | 已寫出目的檔 |
| `1` | 已在標準輸出上回報錯誤 |
| `2` | 用法錯誤，或元件不讀取的輸入格式版本（它會指出兩個版本） |
| `70` | 元件的內部錯誤 |

**版本。** `nabic-codegen-<name> --version` 在第一行印出 `nabic-codegen-<name> <version>`，並印出一行 `mir: <major>.<minor>` 指出它所讀取的輸入格式版本；`nabic` 拒絕其他工具鏈版本的元件，錯誤訊息會指出重新安裝它的指令。元件的身分（其版本行與其檔案的戳記，與 [9.4.4](../009-tools/904-nabihir-bytecode.md#944-header-and-producer) 中的 `nabic` 相同）是每個單元之鍵的一部分（[9.2.16](../009-tools/902-aot-pipeline.md#codegen-units)），因此元件變更時會重新編譯它所編譯過的單元。

## 8.11.14. 驗證

程式碼產生器以 [8.7.9](807-code-generation.md#879-verifying-the-translation) 的端對端測試檢查，測試時程式中包含其目的檔，並使用 GC 壓力設定（`NABIR_GC_STRESS`、`NABIR_GC_EVAC=always`，[9.9.8](../009-tools/909-debug-tools.md#998-memory--gc-tools)），在這些設定下每次配置都可能進行回收，每次完整回收都會移動它所能移動的每個物件。連結兩個程式碼產生器之目的檔（每個單元來自其中任一個）的程式，其行為與由單一程式碼產生器建置的程式相同。
