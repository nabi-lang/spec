# 9.12. 外掛

*外掛*是建置為共用函式庫、由 Nabi 程式在執行中載入的函式庫套件（[3.11 `std::plugin`](../../refs-standard-libraries/300-std/311-plugin.md)）。與 `cdylib`（9.3.3）不同，外掛沒有自己的執行環境：它在載入它之程式的執行環境上執行，因此其值就是該程式的值（一個 GC 堆積、一個排程器、每個 static 一份），而任何 `pub` Nabi 函式都可以帶著任何 Nabi 值跨越邊界呼叫。

## 9.12.1. 模型

一個行程只有一個執行環境，位於*主機*中：即載入外掛的程式。外掛不連結任何執行環境，也不連結它從主機取得的任何套件。它改為攜帶一個模組描述子（[8.12](../008-abi/812-module-abi.md)），其匯入表指名它所使用的每個執行環境進入點、標準函式庫 static 以及相依套件符號；Nabi 模組載入器在外掛載入時以名稱把每一個繫結到主機（或先前載入的外掛）。作業系統的載入器只對應檔案並解析 C 符號（C 函式庫與 `#[link]` 函式庫）。

當程式觸及 `std::plugin` 時，它就是主機；不需要宣告任何東西。主機保留其完整的執行環境，並在匯出表中列出其 Nabi 符號，這對不載入任何外掛的程式沒有任何成本，因為只有主機才會有匯出表。

| | cdylib | 外掛 |
|-|--------|--------|
| 載入者 | 任何 C 主機 | Nabi 程式 |
| 執行環境 | 自己的一份，載入時啟動 | 主機的 |
| 介面 | `pub extern "C" fn`、`#[repr(C)]` 型別 | 每個 `pub fn`、任何 Nabi 值 |
| GC 堆積、排程器、static | 私有 | 與主機共用 |

## 9.12.2. 建置

```nabion
Package {
    name: "enemies",
    version: "0.1.0",
    lib: { kind: "plugin" },
    deps: {
        "engine": "^0.3",                             // taken from the host
        "noise": Dep::version("^1.2", bundle: true),  // linked into the plugin
    },
}
```

- `lib: { kind: "plugin" }` 讓 `nabi build` 傳入 `--crate-type=plugin`（[9.1](901-nabic-cli.md)）。輸出是 `target/<profile>/libenemies.dylib`（macOS）、`libenemies.so`（Linux）或 `enemies.dll`（Windows），與任何函式庫一樣位於 `libenemies.a` 與 `enemies.nabihir` 旁邊。`std::plugin::file("enemies")` 給出該目標的檔案名稱。
- 外掛套件有 `src/lib.nabi` 而沒有 `src/main.nabi`（`` a plugin is loaded by a program, so move main.nabi to a package of its own that loads this one with `std::plugin::load` ``）。對它執行 `nabi run` 是錯誤。
- 預設情況下，相依套件從主機取得：外掛對照它編譯，其 static 是主機的，描述子把它列在 `needs` 之下。`bundle: true` 則把相依套件連結進外掛：其 static 是外掛自己的，並列在 `provides` 之下。在不是外掛之套件的相依套件上使用 `bundle` 是錯誤。
- 程式的相依套件 `Dep::path("../physics", plugin: true)` 會作為外掛建置到程式的目標目錄中（因此位於程式旁邊，使用程式的設定檔與目標），且不會被連結進來；程式以 `plugin::load(plugin::beside("physics"))` 載入它。在同一個相依套件上同時使用 `plugin: true` 與 `bundle: true` 是錯誤。
- 外掛所觸及的標準函式庫程式碼會被編譯進其中（與程式一樣只編譯觸及的部分，[9.3.6](903-package-archive.md#936-standard-library)）；其 static 是主機的。重複的程式碼無害，因為一個行程的每個模組都由同一個工具鏈建置（9.12.6）。
- 外掛的泛型實例被編譯進需要它們的模組：`List<Orc>` 位於定義 `Orc` 的外掛中。泛型程式碼沒有自己的 static，因此其各份複本可以互換。
- 對外掛與主機，codegen 單元（[9.2.16](902-aot-pipeline.md#9216-incremental-build)）是關閉的：它們的描述子容納真正的套件雜湊。

## 9.12.3. 同一性

當一個型別在兩個模組中的*型別鍵*相同時，它就是同一個型別：套件的內容雜湊、路徑以及泛型引數的鍵，例如 `Y3f8a1c2b:engine::Body`（[8.13](../008-abi/813-type-descriptor.md)）。套件雜湊是內容雜湊（[9.5](905-symbol-mangling.md)），因此對照與主機不同之 `engine` 原始碼建置的外掛會有另一個鍵，載入會以缺少套件的錯誤失敗，而不會混用兩種 `Body` 配置。

- 符號以修飾後的名稱繫結。外掛對 `_Y3f8a1c2b_engine_spawn_..` 的匯入只會繫結到提供套件 `3f8a1c2b` 的模組。
- `any` 標籤依行程給予：主機註冊其編譯期標籤，而外掛的程式碼從其描述子的標籤表讀取其標籤，由載入器填入（8.12.6）。行程已見過的鍵保留其標籤，因此外掛存入 `any` 的 `engine::Body` 就是主機的 `engine::Body`。
- GC 型別 id 是結構性的且依模組而定；載入器給每個外掛一段範圍。
- 兩個模組都描述的型別（`Inspect` 型別、泛型實例）必須有相同的配置雜湊。若有差異，通常是以其他 feature 建置的相依套件，會拒絕載入：`` engine::Body is laid out differently in the program and the plugin; build both with the same features of its package ``。

## 9.12.4. 載入與呼叫

`plugin::load(path)` 對應檔案，並在每個執行緒短暫停止期間註冊其描述子：它檢查工具鏈與目標、`needs` 之下的每個套件都已被提供（所有缺少的套件會在一個錯誤中回報）、繫結匯入、給予 GC 型別 id 與 `any` 標籤、註冊根、型別描述子與 `any Tr` 方法主體，然後發布該外掛。外掛沒有初始化函式；其 static 與其他地方一樣在第一次使用時初始化。

`p.get::<F>("enemies::spawn")` 以路徑查找一個 `pub` 非泛型函式，並檢查其簽章恰好是 `F`：雙方都對以型別鍵寫出的簽章文字（`fn({engine::World},i32)->List<..>`）進行雜湊，因此不符時是 `WrongType`，而不是當機。結果是一個一般的函式值。對泛型函式的 `get` 找不到任何東西（`NoSuchItem`，其訊息會說明要匯出一個具體的 `pub fn`）。`p.get_static::<T>(path)` 讀取一個 static。

跨越邊界時：

- `str`、`List`、`Dict`、閉包以及其他每個 GC 值都是指向唯一堆積的參照，原樣雙向傳遞。
- 閉包會讓建立它的外掛保持對應，直到閉包被回收（9.12.5）。
- `any` 與 `any Tr` 原樣傳遞。`any Tr` 方法的分派器知道它自己模組的型別；對它不認識的標籤，它會詢問行程（`nabi_any_dispatch`，8.12.6），由行程呼叫存入該值之模組的分派器。不載入任何外掛的程式保留 [2.2.10](../002-types/202-any-slot.md) 的封閉分派器。
- 和型別、列舉與 `throws` 錯誤是帶有結構性標籤的值，不經改變地跨越。外掛中的 panic 與其他任何地方一樣會結束行程。

## 9.12.5. 卸載

`p.unload()` 是延後進行的：

1. 若另一個已載入的外掛從 `p` 匯入，它回傳 `Pending`，並附上使用它的東西。若 `p` 交出了它自己程式碼的 C 回呼（`NO_UNLOAD`），它回傳 `Kept`：這樣的外掛會保持對應。
2. 否則 `p` 會立即離開查找（`get` 與 `plugin::loaded()` 不再看到它），並被*退役*。
3. 之後的完整回收會檢查已退役的外掛是否仍然存活：屬於其某個 GC 型別的物件、其程式碼位於其中的函式值或協程、堆疊上屬於它的框架。當都不存在時，其根、型別與方法主體會被捨棄，檔案會被解除對應；若已經如此，`unload` 回傳 `Done`。

`plugin::wait_unloaded(p, ms)` 會持續回收，直到該外掛消失或時間用完（供測試與工具使用）。

## 9.12.6. 穩定性與錯誤

沒有跨工具鏈建置穩定的外掛 ABI。一個行程的每個模組都必須來自同一個工具鏈建置：描述子的 `toolchain` 是編譯器 id，以完全相同比較（`` the plugin was built by nabic 0.1.3 (a1b2c3), this program by nabic 0.1.4 (d4e5f6); rebuild the plugin with this program's toolchain ``）。

| `PluginError` | 時機 | 訊息說明該怎麼做 |
|---|---|---|
| `NotFound` | 該路徑沒有檔案 | 給出 `nabi build` 所寫出的檔案 |
| `NotAPlugin` | 沒有 Nabi 描述子的檔案 | 以 `lib: { kind: "plugin" }` 建置該套件，或以 `bare::ffi` 載入 C 函式庫 |
| `Toolchain` / `Target` | 另一個編譯器 id / 目標 | 重新建置外掛 |
| `MissingPackages` | 行程缺少 `needs` 之下的某個套件 | 把它加到程式中，或在外掛中使用 `bundle: true` |
| `MissingSymbols` | 沒有任何東西提供的匯入 | 對照程式的套件建置外掛 |
| `Layout` | 一個鍵，兩種配置 | 以相同的 feature 建置兩者 |
| `NoSuchItem` | 該路徑沒有 `pub fn` / static | 接近它的名稱；匯出一個具體的函式 |
| `WrongType` | 另一個簽章 | 該寫的型別 |
| `NotLoaded` | 外掛已被卸載 | 再次載入它 |
| `Os` | 系統載入器失敗 | 其訊息 |

## 9.12.7. 目標

| 目標 | 檔案 | 狀態 |
|---|---|---|
| aarch64-apple-darwin | `MH_DYLIB`，install name 為 `@rpath/lib<name>.dylib`，ad hoc 簽章，只匯出 `_nabi_module` | 已支援；hardened-runtime 主機需要 `disable-library-validation` entitlement 才能載入他人簽章的外掛 |
| x86_64-unknown-linux-gnu、aarch64-unknown-linux-gnu | `ET_DYN`，只匯出 `nabi_module` | 已支援 |
| x86_64-pc-windows-msvc、aarch64-pc-windows-msvc | 匯出 `nabi_module` 的 DLL，帶有自己的進入點；以 `LoadLibraryExW` 載入 | 已支援；已載入之 DLL 的檔案會被鎖定 |
| *-linux-musl | | 錯誤：C 函式庫以靜態方式連結，無法載入程式碼（`` build plugins and the program for `x86_64-unknown-linux-gnu`, or build one program ``） |
| wasm32-* | | 錯誤：`` `--crate-type=plugin` needs a native target ``；觸及 `std::plugin` 的 wasm 程式也會以相同方式連結失敗（`` on wasm, build one program: make the plugin's package an ordinary dependency ``） |
| 其他目標 | | 尚未連結 |

`NABI_CC` 無法連結外掛或主機：它們需要 Nabi 自己的連結器，由它產生匯入 stub 與各種表。
