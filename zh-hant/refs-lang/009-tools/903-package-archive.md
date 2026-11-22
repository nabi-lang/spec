# 9.3. 套件封存檔（.a）格式

函式庫套件 `foo` 的建置輸出是兩個檔案（[9.2.15](902-aot-pipeline.md#9215-輸出)）。

| 檔案        | 內容                                                                          |
|-------------|-------------------------------------------------------------------------------|
| `libfoo.a`  | 容納套件之目的檔的標準 `ar` 封存檔，每個 codegen 單元一個（[9.2.16](902-aot-pipeline.md#9216-增量建置)） |
| `foo.nabihir`  | 下游用於單態化與名稱解析的介面（[9.4](904-nabihir-bytecode.md)） |

兩個檔案位於同一個目錄。`--extern foo=libfoo.a` 與 `--extern foo=foo.nabihir` 都會找到另一個檔案（[9.2.3](902-aot-pipeline.md#923-相依套件解析)）。版本與產生者之類的中繼資料位於 NABIHIR 標頭中（[9.4](904-nabihir-bytecode.md)）。

## 9.3.1. 標準 ar 封存檔

`libfoo.a` 是標準的 Unix `ar` 封存檔。`ar`、`ranlib`、`nm` 與 C 連結器不需轉換就能處理它。

```
ar t libfoo.a
part0.o
part1.o
...
```

## 9.3.2. 目的檔

每個目的檔都是 [AOT 建置管線](902-aot-pipeline.md)為一個 codegen 單元產生的標準 ELF / Mach-O / COFF 目的檔。它們合起來只包含套件的[具體 fn](../000-meta/007-glossary.md#具體函式--泛型函式concrete-fn--generic-fn) 中，從套件的根可以到達的那些（[9.2.8](902-aot-pipeline.md#928-specialize單態化)）。

| 區段      | 內容                                                        |
|-----------|-------------------------------------------------------------|
| `.text`   | 具體 `fn` 的程式碼                                          |
| `.rodata` | `const`、字面值、[ScanTable](../008-abi/802-scan-table.md)   |
| `.data`   | `static` 的初始值                                           |
| `.bss`    | 以零初始化的 `static`                                       |

ScanTable 是一般的常數資料；產生的註冊程式碼在行程啟動時經由 `nabi_gc_register_type` 把它們註冊到執行環境（[8.2.13](../008-abi/802-scan-table.md#8213-註冊時機)）。沒有 nabi 專屬的區段。

## 9.3.3. 靜態與共用函式庫

函式庫的輸出是靜態的 `.a`；相依套件從封存檔連結進程式，而程式所觸及的標準函式庫程式碼與它一起編譯（[9.3.6](#936-標準函式庫)）。`--crate-type=cdylib`（[9.1.2](901-nabic-cli.md#nabic-buildaot-建置)）還會把函式庫套件連結成一個由 C 程式碼（或 Python、R、Julia）在執行時載入的共用函式庫：`lib<name>.so`（Linux）、`lib<name>.dylib`（macOS，install name 為 `@rpath/lib<name>.dylib`），或 `<name>.dll` 加上其匯入函式庫 `<name>.lib`（Windows）。它容納該套件、標準函式庫與執行環境，並且只匯出套件的 `#[export] extern "C" fn` 與 `#[export_name("sym")] extern "C" fn`（[5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-定義對外公開)），名稱為它們自己的名稱或 `sym`；其他每個符號都在函式庫內部繫結。

- 載入函式庫會啟動其執行環境：載入器執行 `nabi_library_init`（`DT_INIT`、一個 `__mod_init_func` 項目，或在行程附加時執行 DLL 進入點 `nabi_dll_main`），它在載入的執行緒上設定執行環境、註冊函式庫的 GC 型別（[8.2.13](../008-abi/802-scan-table.md#8213-註冊時機)），並把該執行緒作為位於 C 程式碼中的執行緒停駐。在任何執行緒上被呼叫的匯出函式會先附加該執行緒（[8.5](../008-abi/805-extern-symbols.md)），因此 C 程式碼不需要初始化呼叫。
- 每個共用函式庫都有自己的執行環境與堆積：GC 物件不會從一個 Nabi 函式庫跨到另一個，也不會跨到載入它的 Nabi 程式。只有 C 的值會跨越匯出函式。
- 在 Linux 上，函式庫的執行緒區域變數使用 initial-exec 模型（`DF_STATIC_TLS`）：glibc 把它們放在其靜態 TLS 區域中，那裡也有空間給以 `dlopen` 開啟的函式庫。一個函式庫占用其中 160 位元組，即執行環境的執行緒狀態（[8.3](../008-abi/803-tls-slots.md)）；glibc 為之後開啟的函式庫保留 512 位元組（`glibc.rtld.optional_static_tls`），因此預設情況下一個行程能容納三個 Nabi 函式庫。更多的會以 `cannot allocate memory in static TLS block` 載入失敗；`GLIBC_TUNABLES=glibc.rtld.optional_static_tls=4096` 可以騰出空間。
- 版本：`--lib-version <x.y.z>`（[9.1.2](901-nabic-cli.md#nabic-buildaot-建置)），`nabi build` 從資訊清單的 `version` 取得它，它把 ELF soname 設為 `lib<name>.so.<x>`（低於 1.0 時是 `lib<name>.so.0.<y>`，此時次版本號會造成破壞），並在檔案旁邊建立該名稱的連結，並把 Mach-O 的目前版本與相容版本設為 `x.y.z` 與 `x.0.0`（低於 1.0 時是 `0.y.0`）。
- C 標頭檔：建置還會寫出 `target/<profile>/include/<name>.h`，與 `--emit=c-header` 相同。它以 `#[export_name]` 的名稱宣告套件中每個帶有該屬性的 `extern "C"` 函式，以及它們的簽章所觸及的型別：`#[repr(C)]` struct 或 union 宣告為依序列出其欄位的 `typedef struct`（`align(N)` 用 `_Alignas`，`packed` 用 `#pragma pack`），`#[repr(transparent)]` struct 宣告為其欄位的 `typedef`，`#[repr(iN)]` enum 宣告為其標籤型別的 `typedef`，每個變體一個 `#define <ENUM>_<VARIANT> ((<Enum>)<value>)`。基本型別對應到 `<stdint.h>` / `<stddef.h>` / `<stdbool.h>` 的型別（`usize` 是 `size_t`，`isize` 是 `ptrdiff_t`，`c_char` 是 `char`，`char` 是 `uint32_t`，C ABI 的 `str` 是 `const char *`），`ConstPtr<T>` 對應到 `const T *`，`Ptr<T>` / `ManyPtr<T>` 對應到 `T *`，`CPtr<T>` 對應到帶有 `/* nullable */` 註解的 `T *`，`T | none` 也是，`extern "C" fn` 型別對應到函式指標，`extern "system"` 函式對應到 `NABI_SYSTEM`（32 位元 Windows 上的 `__stdcall`），永不回傳的函式對應到 `_Noreturn`。每個項目的 `///` 文件註解會變成 C 註解。其型別無法在 C 中命名的函式（泛型 struct、沒有 `#[repr(C)]` 的 struct、帶資料的 enum）會被省略，並留下一行 `/* skipped <name>: <why> */`。標頭檔帶有 `#pragma once`、include guard 以及供 C++ 使用的 `extern "C"`。
- 目標：`aarch64-apple-darwin`、`x86_64-unknown-linux-gnu`、`aarch64-unknown-linux-gnu`、`x86_64-pc-windows-msvc` 與 `aarch64-pc-windows-msvc`，使用 Nabi 自己的連結器（不是 `NABI_CC`）。其他目標是錯誤，錯誤訊息會說明改以 C 工具鏈連結靜態函式庫。
- 對其他共用函式庫的 [FFI](../000-meta/007-glossary.md#ffi外部函式介面) 呼叫在每個目標上都能運作。

`--crate-type=staticlib` 則產生一個 C、Swift 與 Xcode 可以單獨連結的靜態函式庫：`target/<profile>/static/lib<name>.a`（由 `-o` 指定）容納該套件、其相依套件、它所觸及的標準函式庫以及執行環境，每個封存檔的成員都重新命名為 `<n>-<member>`，因此彼此不會遮蔽。執行環境在第一個匯出函式被呼叫時啟動，並與共用函式庫的匯出函式一樣附加其執行緒；GC 物件同樣不會跨越匯出函式。建置還會寫出 C 標頭檔（如上所述）以及 `target/<profile>/include/module.modulemap`，即一個以套件命名的 clang 模組，它連結該函式庫以及套件所指名的系統函式庫與 framework，因此 Swift 寫 `import <name>`：

```
module geo {
    header "geo.h"
    link "geo"
    link framework "UIKit"
    export *
}
```

有 `--xcframework` 時，建置還會把函式庫及其標頭檔放進 `target/<profile>/<name>.xcframework`，即與 `xcodebuild -create-xcframework` 所寫出的相同的 XCFramework：每個建置過的 Apple 目標一個切片（`ios-arm64`、`ios-arm64-simulator`、`macos-arm64`），每個都帶有 `lib<name>.a` 與 `Headers/`，以及列出它們的 `Info.plist`。每次建置都會新增或取代其目標的切片，因此為 `aarch64-apple-ios` 建置一次、為 `aarch64-apple-ios-sim` 建置一次，就會產生一個 Xcode 對兩者都接受的 XCFramework；裝置目標仍在進行中（[0.6.2](../000-meta/006-platform-targets.md#062-進行中)），因此目前 XCFramework 只容納模擬器的切片。沒有 `--crate-type=staticlib` 的 `--xcframework`，或用於非 Apple 的目標，是錯誤，錯誤訊息會說明該寫什麼。

這就是 iOS app 取用 Nabi 程式碼的方式（[0.6.1](../000-meta/006-platform-targets.md#061-已支援)）：一個 `aarch64-apple-ios-sim` 的封存檔，以及在裝置目標獲得支援之後一個 `aarch64-apple-ios` 的封存檔，加入 Xcode 專案，並把 `include` 目錄加到其標頭檔搜尋路徑上。在 wasm 目標上，`--crate-type=staticlib` 是錯誤。

## 9.3.4. 版本相容性檢查

讀取相依套件的 NABIHIR 時，會把其 `version` 與產生者（產生該 NABIHIR 的 `nabic`）與目前的 `nabic` 比較。若兩者不同，就是說明需要重新建置什麼的錯誤（[9.4.9](904-nabihir-bytecode.md#949-讀取與拒絕)）。

## 9.3.5. 符號公開

封存檔所公開的符號是其目的檔的全域符號，遵循[符號修飾](905-symbol-mangling.md)。

```
nm libfoo.a
0000000000000000 T _Y3f8a1c2b_foo_geom_distance_a7c91d8e44b2f190
0000000000000040 T _Y3f8a1c2b_foo_geom_Point_origin_b1234567...
```

## 9.3.6. 標準函式庫

標準函式庫（`bare` / `core` / `std`）不以封存檔散布，也沒有任何建置會連結預先建置的標準函式庫。每次建置都與套件一起從原始碼（`--bootstrap`）編譯它，而且只編譯程式所觸及的部分：從程式的根（`main`、harness 的 `#[test]` 與 `#[bench]` 函式、`#[export_name]` 與 WIT 匯出、編譯器與執行環境所呼叫的函式，以及每個 static 的初始化器）出發，前端追蹤呼叫、函式值、它所使用之型別的 `Drop` 實作，以及 `any Trait` 值所分派到的實作，並對它所遇到的每個函式進行型別檢查與降階（[9.2.6](902-aot-pipeline.md#926-型別檢查)）。對於其餘部分，它只讀取名稱與簽章，供名稱解析、方法查找與診斷使用。`--check-library` 則會檢查並編譯函式庫的每個主體（[9.1.2](901-nabic-cli.md#nabic-buildaot-建置)）。

程式所觸及的標準函式庫函式以 codegen 單元編譯，每個模組一個（9.2.16）。每個函式最佳化後的程式碼與每個單元的目的檔，以涵蓋它們所依賴之一切的鍵保存在工具鏈的函式快取（`lib/nabi-cache/fn/`，[9.2.19](902-aot-pipeline.md#9219-安裝配置)）中，而該工具鏈之後每個鍵相同的建置，無論它建置哪個套件，都從那裡取用它們。最佳化器仍會看到整個程式，包括標準函式庫的函式，因此它會像內嵌套件自己的函式一樣內嵌它們（9.2.10）。

函式庫套件的封存檔容納它所觸及之標準函式庫程式碼的私有複本，就像它的泛型實例一樣。標準函式庫的 static 是例外：函式庫以它們自己的符號將它們定義為 weak，因此程式以及連結進其中的每個函式庫都共用每個 static 的同一份。

## 9.3.7. 連結

```
cc main.o libfoo.a libnabir.a -o my_binary
```

連結器只從封存檔的目的檔中連結需要的符號。以 `#[export_name]` 公開的函式可以直接從 C 呼叫。

```c
extern uint32_t nabi_compute(int32_t a, int32_t b);   // #[export_name("nabi_compute")]
int main(void) { return (int)nabi_compute(1, 2); }
```

```
cc main.c libfoo.a libnabir.a -o my_binary
```

使用 `--crate-type=staticlib`（9.3.3）時，一個封存檔就足夠了：

```
cc main.c target/release/static/libfoo.a -o my_binary
```

## 9.3.8. 穩定性

封存檔格式遵循 [ABI 穩定性方針](../008-abi/README.md)；NABIHIR 格式遵循 [9.4](904-nabihir-bytecode.md)。
