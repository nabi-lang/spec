# 8.12. 模組描述子 ABI

*模組*是行程中一個可載入的 Nabi 程式碼單位：程式本身（*主機*）或一個外掛。每個模組以唯讀資料中的一個*模組描述子*向執行環境描述自己。載入器讀取它，以檢查模組是否適合該行程、繫結它所匯入的符號，並註冊其 GC 型別、`any` 標籤、根與型別描述子（[8.13](813-type-descriptor.md)）。其配置在 `nabir_abi::module` 中定義一次，編譯器與執行環境都會建置它。

## 8.12.1. 符號

| 符號 | 定義者 | 它是什麼 |
|--------|-----------|------------|
| `nabi_module`（Mach-O 為 `_nabi_module`） | 外掛 | 其描述子，外掛唯一匯出的符號 |
| `nabi_host_module` | 主機 | 其描述子，由執行環境直接取用 |

## 8.12.2. 配置規則

所有指標都是絕對位址，由作業系統的重定位固定。`Str` 是覆蓋 UTF-8 位元組的 `{ptr, len: usize}`，`Slice<T>` 是覆蓋 `len` 個項目的 `{ptr, len: usize}`。在兩種指標寬度下，每個結構的配置都沒有隱含的填充：`u64` 位於 8 的倍數處，而指標大小的字組成對出現。因此 `u64` 對齊為 4（i686）與 8（wasm32）的 32 位元目標在每個偏移上都一致。欄位只會被附加；`version` 說明讀取者可以使用哪些欄位。

## 8.12.3. `ModuleDesc`

| 欄位 | 型別 | 意義 |
|-------|------|---------|
| `magic` | `u32` | 以小端序 `u32` 表示的 `"NABM"` |
| `version` | `u16` | 1 |
| `flags` | `u16` | `HOST` 1、`PLUGIN` 2、`HOT` 4（熱重載建置）、`NO_UNLOAD` 8（已把它自己程式碼的 C 回呼交出去） |
| `hot_version` | `u64` | 除非 `HOT`，否則為 0 |
| `text_lo`、`text_hi` | `usize` | 模組的程式碼是 `[text_lo, text_hi)` |
| `toolchain` | `Str` | 編譯器 id；一個行程的每個模組都相同 |
| `target` | `Str` | 目標三元組 |
| `name` | `Str` | 外掛的套件名稱，或主機的程式名稱 |
| `provides` | `Slice<PkgDesc>` | 其程式碼與 static 位於這個模組中的套件 |
| `needs` | `Slice<PkgDesc>` | 它從行程匯入的套件 |
| `imports` | `Slice<Import>` | 它從行程取得的符號 |
| `exports` | `Slice<Export>` | 它提供的符號，依 `sym` 排序 |
| `gc_types` | `GcTypes` | 其 ScanTable（[8.2](802-scan-table.md)）以及載入器給它們的 id |
| `tags` | `Tags` | 其程式碼所使用之 `any` 標籤的型別鍵，以及載入器給它們的標籤 |
| `roots` | `Slice<Root>` | 其 static 根 |
| `types` | `Slice<*const TypeDesc>` | 其型別描述子（[8.13](813-type-descriptor.md)） |
| `dispatch` | `Slice<Dispatch>` | 其 `any Tr` 方法主體，依方法與標籤排列 |
| `functions` | `Slice<FnRange>` | 僅限 `HOT`：每個函式的範圍與框架資訊 |

組成部分：

| 結構 | 欄位 |
|-----------|--------|
| `PkgDesc` | `name: Str`、`version: Str`、`pkg8: [u8; 8]`（套件雜湊在其 `_Y` 符號中的十六進位數字，[9.5](../009-tools/905-symbol-mangling.md)）、`sha256: [u8; 32]`（套件的完整內容雜湊） |
| `Import` | `sym: Str`（修飾後的名稱）、`sig: u64`、`slot: *mut usize`、`kind: usize`（`FN` 0、`DATA` 1、`HOT_FN` 2） |
| `Export` | `sym: Str`、`path: Str`（`pub` 非泛型函式或 static 為 `enemies::spawn`，否則為空）、`sig: u64`、`addr: usize`、`kind: usize`（與 `Import` 相同，另加 `FN_VALUE` 3：函式的值 thunk）、`sig_text: Str`（`sig` 所雜湊的型別，依 `type_name` 的寫法，例如 `fn(i32) -> List<str>`，供錯誤訊息使用；`sig` 為 0 時為空） |
| `GcTypes` | `tables: *const *const u8`、`ids: *mut u32`、`n: u32`、`reserved: u32` |
| `TagKey` | `key: Str`（型別鍵，8.13.2）、`hash: u64`（`key` 的 FNV-1a 64） |
| `Tags` | `keys: *const TagKey`、`tags: *mut u32`、`n: u32`、`reserved: u32` |
| `Root` | `word: *const usize`、`any: usize`（1：`any` 的酬載字組，其標籤是前一個字組） |
| `Dispatch` | `method: u64`、`code: usize`、`tag_index: usize`（`tags` 中的索引） |
| `FnRange` | `start: u32`（從 `text_lo` 算起）、`len: u32`、`frame: *const u8`、`reserved: usize` |

`sig` 是以型別鍵寫出之標準簽章文字的 SHA-256 的前 64 位元，例如 `fn(Y3f8a1c2b:game::Player, i32) -> str`。對資料而言，它是 static 之型別鍵的雜湊。`FN` 匯入經由一個透過 `slot` 跳躍的 stub 呼叫；`DATA` 匯入的 `slot` 容納資料的位址。

## 8.12.4. 執行環境進入點

這些進入點擴充 [8.5](805-extern-symbols.md)。全部都是 `thread-safe`。

| 進入點 | 意義 |
|-------|---------|
| `nabi_types_register(descs: *const *const TypeDesc, n: usize, tags: *const TagKey, out_tags: *mut u32, nt: usize) -> i32` | 註冊 `n` 個型別描述子並解析 `nt` 個標籤（8.12.5）。回傳 0，或在 `descs[i]` 的鍵已以另一個配置雜湊註冊時回傳 `1 + i`；此時不會註冊任何東西。 |
| `nabi_types_by_tag(tag: u32) -> *const TypeDesc` | `any` 標籤（忽略其 `ANY_BOXED` 位元）的描述子，或 null |
| `nabi_types_by_path(p: *const u8, len: usize) -> *const TypeDesc` | 某個路徑（例如 `game::Player`）最新註冊的描述子，或 null |
| `nabi_types_unregister(descs: *const *const TypeDesc, n: usize)` | 在其模組被釋放時撤銷註冊 |
| `nabi_module_register(m: *const ModuleDesc) -> i32` | 在短暫的 stop-the-world 中檢查並繫結一個模組，並註冊其表；成功時為 0 |
| `nabi_any_dispatch(method: u64, tag: u32) -> usize` | 來自已載入模組、針對 `tag` 之值的 `method` 程式碼，或 0 |
| `nabi_tls_get() -> *mut u8` | 呼叫端執行緒的 `nabi_tls` 區塊（[8.3](803-tls-slots.md)） |

型別登錄的讀取者不取得任何鎖，也不配置任何東西。

## 8.12.5. 標籤

模組的程式碼從 `Tags.tags` 讀取它所使用之每個型別的 `any` 標籤。對每個鍵，`nabi_types_register` 保留 `out_tags` 中已有的標籤（不是 `u32::MAX`）：主機以及沒有外掛的程式以這種方式傳入其編譯期標籤。之前已註冊的鍵取得相同的標籤，這正是讓一個型別跨模組只有一個標籤的方式。新的鍵取得該鍵之 FNV-1a 雜湊的低 31 位元，並移到所有使用中的標籤之後。

## 8.12.6. 編譯器發出什麼

外掛的黏合程式碼（[8.7](807-code-generation.md)）發出 `nabi_module` 以及它所指向的表；主機的黏合程式碼發出 `nabi_host_module`。既不觸及 `std::plugin` 也不使用 `--crate-type=plugin` 的程式兩者都不發出，其程式碼與沒有外掛時相同。

| 欄位 | 外掛 | 主機 |
|---|---|---|
| `flags` | `PLUGIN` | `HOST` |
| `text_lo`、`text_hi` | 指向 `nabi.module.text_lo` / `nabi.module.text_hi` 的重定位，由連結器定義；載入器從對應的映像取得程式碼的範圍 | 相同 |
| `provides` | 它自己的套件以及它所打包的套件（[9.12.2](../009-tools/912-plugins.md#9122-建置)） | 它所連結的每個套件，標準函式庫除外 |
| `needs` | 建置中的其他每個套件，標準函式庫除外 | 空 |
| `imports` | 指向 `nabi.module.imports` 的重定位，即連結器從保持未定義的符號（每個 `nabi_*` 進入點與 `_Y` 符號）建立的表；連結器把其長度寫入描述子；`sig` 為 0 | 空 |
| `exports` | 依 `sym` 排序：其套件的每個函式為 `FN`（`pub` 非泛型者設定 `path`），其值 thunk 為 `FN_VALUE`（設定 `path`、`sig` 與 `sig_text`），其 static 為 `DATA`（設定 `path`），以及它從行程取得之 static 的自有複本（見下文） | 指向 `nabi.module.exports` 的重定位，即連結器建立的表（每個已定義的 `nabi_*` 進入點與 `_Y` 符號，`sig` 為 0）；連結器寫入其長度 |
| `gc_types` | 其程式碼所使用的每個 ScanTable；`ids` 初始為 0 | 空：主機與任何程式一樣以 `nabi_gc_register_type` 註冊其表 |
| `tags` | 其程式碼所使用之每個標籤的鍵；`tags` 初始為 `u32::MAX` | 帶有編譯期標籤的鍵 |
| `roots` | 它所定義之每個 static 的每個參照字組（`any` 為 0） | 空：與任何程式一樣註冊 |
| `types` | 其型別描述子（[8.13](813-type-descriptor.md)），其 `tag_index` 索引其 `tags` | 空：它與任何程式一樣以 `nabi_types_register` 註冊其描述子 |
| `dispatch` | 每個 `any Tr` 分派器以及其分支所接受的每個標籤：方法的 `<trait path>.<method>` 之 FNV-1a 64 雜湊、分派器的值程式碼 | 主機的分派器也相同 |

主機的 `nabi.register_types`（[8.5](805-extern-symbols.md)）以 `nabi_module_register(&nabi_host_module)` 結束，因此其標籤、分派項目與匯出表在 `main` 之前、也在任何外掛之前就已存在於行程中。

外掛的程式碼與程式的程式碼有何不同：

- **GC 型別 id。** 配置會讀取 `nabi.module.gc_ids[0]` 並加上表的索引：載入器給模組一段連續的範圍，因此 `ids[i] = ids[0] + i`。
- **`any` 標籤。** 存入 `any` 以及測試 `any` 的標籤時，從 `nabi.module.tags` 載入標籤（型別在模組標籤清單中的索引所對應的 `tags[i]`），而不是使用常數；`ANY_BOXED` 在執行時以 or 加入。
- **TLS。** 會取得 `nabi_tls` 位址的程式碼（[8.3](803-tls-slots.md)），即函式值的 thunk 或 C 進入點，會呼叫匯入的 `nabi_tls_get()`。Nabi 對 Nabi 的呼叫與其他地方一樣，以隱藏的最後一個引數傳遞該區塊，因此大部分程式碼從不需要詢問。
- **它從行程取得的 static。** 外掛既不擁有也不打包之套件的 static（包括標準函式庫的）經由目標的 GOT 形式取用，連結器讓它指向匯入的槽位。外掛也把自己的複本定義為 `<sym>.own`（延遲初始化者還有 `<sym>.state.own`、`<sym>.init.own`），並以 `<sym>` 匯出。載入器把匯入繫結到主機的 static，否則到之前已載入之外掛的 static，否則到這份複本，因此每個 static 在行程中恰好只有一個實例。
- **分派器。** 分派器針對它不認識之標籤的分支會呼叫 `nabi_any_dispatch(method, tag)`，並把結果當作分派器自身型別的函式值呼叫。在不是主機的程式中，該分支仍是 [8.8](808-lowering-types.md) 的 trap。
- **簽章。** 匯出的 `sig`，以及 `Plugin::get::<F>` 所傳遞的值，是以型別鍵寫出之簽章文字（`fn(Y3f8a1c2b:engine::World,i32)->str`）的 SHA-256 的前 8 個位元組（小端序）。雙方在編譯期都從相同的文字計算它，因此雜湊相等就表示型別相同。
