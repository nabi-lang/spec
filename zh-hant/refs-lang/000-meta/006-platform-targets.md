# 0.6. 平台目標

Nabi 可以在任何主機上（`--target`，[0.2.4](002-pipeline.md#024-程式碼產生)），為下列內建目標以及相同架構與目的檔格式的自訂目標（0.6.5）建置。每個目標都處於三種狀態之一：已支援（0.6.1）、進行中（0.6.2）或不支援（0.6.3）。

## 0.6.1. 已支援

已支援的目標是內建的：`nabic --print target-list` 會列出它們，`--target` 也接受其中每一個。

### 64 位元桌面 / 伺服器

這一組涵蓋 Linux、macOS 與 Windows 的主要架構，包括用於靜態二進位檔的 musl 變體，以及 FreeBSD。

| 目標                         | 備註                                       |
|------------------------------|--------------------------------------------|
| `x86_64-unknown-linux-gnu`   | 主要的 Linux 伺服器 / 桌面                 |
| `aarch64-apple-darwin`       | macOS Apple Silicon                        |
| `aarch64-unknown-linux-gnu`  | ARM 伺服器（AWS Graviton、Raspberry Pi 5 等） |
| `x86_64-pc-windows-msvc`     | Windows 桌面                               |
| `aarch64-pc-windows-msvc`    | Windows ARM                                |
| `x86_64-unknown-linux-musl`  | Alpine / `static` 二進位檔                 |
| `aarch64-unknown-linux-musl` | 同上，ARM                                  |
| `x86_64-unknown-freebsd`     | FreeBSD 14                                 |

### WebAssembly

支援三個 WebAssembly 目標，涵蓋瀏覽器端與伺服器端執行。WASI sockets（僅限 preview 2 元件）與 WASI threads（一項 preview 1 核心模組提案）目前無法在同一個模組中並用，因此伺服器端 WASM 有兩個目標。

| 目標                     | 備註                             |
|--------------------------|----------------------------------|
| `wasm32-unknown-unknown` | 瀏覽器環境；`target_os = "unknown"`（[6.3.2](../006-meta-system/603-cfg.md)） |
| `wasm32-wasip2` | 伺服器端 WASM：WASI preview 2 元件。以 `wasi:sockets` 提供網路；沒有執行緒（`thread::spawn` 會 panic；`thread::Builder::spawn` 擲回 `Unsupported`） |
| `wasm32-wasip1-threads` | 帶執行緒的伺服器端 WASM：使用 wasi-threads（共享記憶體）的 WASI preview 1 核心模組。網路只能用在主機傳入的監聽器上；connect / bind / DNS 擲回 `Unsupported` |

> [!NOTE]
> wasmtime 在 v46 之後移除了 wasi-threads（WAMR 的 iwasm 與 wasmer 仍有實作），因此 `wasm32-wasip1-threads` 將來會改為相依於 WASI shared-everything-threads。

這些名稱沿用生態系其他部分的用法，其中 `wasm32-wasi` 指 WASI preview 1；Nabi 沒有名為 `wasm32-wasi`、`wasm32-wasi-threads` 或 `riscv32imac-unknown-none` 的目標，對 `--target` 指定其中之一，會成為未知目標，錯誤訊息會指出最接近的內建目標（[9.1.2](../009-tools/901-nabic-cli.md#912-核心子指令)）。

### 32 位元

為了仍在實際使用中的嵌入式與舊有環境，支援下列四個 32 位元目標。

| 目標                            | 備註                                 |
|---------------------------------|--------------------------------------|
| `i686-unknown-linux-gnu`        | 32 位元 x86 Linux                    |
| `i686-pc-windows-msvc`          | 32 位元 Windows                      |
| `armv7-unknown-linux-gnueabihf` | 32 位元 ARM Linux，硬體浮點（在 32 位元作業系統上的 Raspberry Pi 2 至 4 等） |
| `armv7-unknown-linux-musleabihf` | 同上，musl：`static` 二進位檔 |

### 行動裝置

| 目標                    | 備註                                      |
|-------------------------|-------------------------------------------|
| `aarch64-apple-ios-sim` | Apple Silicon Mac 上的 iOS 模擬器         |

`aarch64-apple-ios-sim`（Apple Silicon Mac 上的 iOS 模擬器）為 iOS 13.4 以上版本建置；iOS 13.4 是第一個其載入器能讀取 chained fixups 的 iOS，而 chained fixups 正是 Apple 連結器寫出的格式。`IPHONEOS_DEPLOYMENT_TARGET`，或 nabi.nabion 中的 `deployment_target: { ios: "16.0" }`（[9.10](../009-tools/910-nabi-frontend.md)），可以提高版本；更舊的版本是錯誤，錯誤訊息會指出最舊可用的版本。連結需要 Xcode 的 iOS SDK：`SDKROOT`，否則為開發者目錄（`DEVELOPER_DIR`，否則為 `xcode-select` 指定者）中的 `iPhoneSimulator` SDK；沒有時，建置會提示安裝 Xcode。若 `SDKROOT` 是另一個平台的 SDK（依其 `SDKSettings.json` 或目錄名稱判斷，例如 iOS 目標卻給了 macOS SDK），會附帶提示略過它，改用開發者目錄的 SDK。目的檔與執行檔會標明其平台（`LC_BUILD_VERSION`：iOS 模擬器），連結器會拒絕為其他 Apple 平台建置的目的檔。基準 CPU 是 `apple-m1`。`target_os` 是 `"ios"`，`target_abi` 是 `"sim"`（[6.3.2](../006-meta-system/603-cfg.md)）。執行檔以 ad hoc 方式簽署，由模擬器執行：`nabi run`、`nabi test` 與 `nabi bench` 透過目標的 `simrun.sh` 執行，它在已開機的裝置（或 `NABI_SIM_DEVICE` 指定的裝置）上執行 `xcrun simctl spawn`，並把每個環境變數以 `SIMCTL_CHILD_<name>` 的形式傳下去，這是 `simctl` 唯一會交給程式的變數。應用程式的 Nabi 程式碼是一個由 Xcode 連結的函式庫：`--crate-type=staticlib`（[9.3.3](../009-tools/903-package-archive.md#933-靜態與共用函式庫)）建置一個封存檔，內含套件、其相依套件與執行環境，Swift 則讀取其 module map（`import <name>`）。`--crate-type=plugin` 與 `--hot` 可在模擬器上使用；REPL 不在那裡執行。維護者在自己的機器上執行模擬器的測試語料。

裝置目標 `aarch64-apple-ios`（iPhone 與 iPad）仍在進行中（0.6.2）。以下文字只適用於它。它和模擬器一樣為 iOS 13.4 以上版本建置，以 `iPhoneOS` SDK 連結，並在 `LC_BUILD_VERSION` 中標明 iOS 平台；它的基準 CPU 是 `apple-a7`（沒有 LSE atomics，與 A9 相同），也不設定 `target_abi`。裝置只有在開發者用 Xcode 為它簽署之後才會執行執行檔，因此 `nabi run`、`nabi test` 與 `nabi bench` 是錯誤，錯誤訊息會指出模擬器目標。在裝置上應用程式無法啟動行程，因此 `Command` 在那裡擲回 `Unsupported`，也沒有任何東西會在執行時載入程式碼：`--crate-type=plugin` 與 `--hot` 是錯誤。

### Android

| 目標                    | 備註                                      |
|-------------------------|-------------------------------------------|
| `x86_64-linux-android`  | Android x86-64（模擬器），API level 29 以上 |

`x86_64-linux-android` 為 Android 10（API level 29）以上版本建置執行檔與共用函式庫（`--crate-type=cdylib`，由 `System.loadLibrary` 載入）；把它們打包進應用程式是 Android 建置的工作。API level 29 是最低版本，因為 Android 的 C 函式庫 bionic 從那時起才支援 ELF 執行緒區域儲存，而執行環境把每個執行緒的狀態存放在其中。`ANDROID_PLATFORM=<n>`（或 NDK 自己的建置工具接受的 `android-<n>`）可以為更高的 level 建置；低於 29 的 level 是錯誤，錯誤訊息會提示寫 `ANDROID_PLATFORM=29` 或更高。連結時會從 Android NDK 取用該 level 的 C 執行環境目的檔與 bionic 的函式庫，而 NDK 不屬於工具鏈：`ANDROID_NDK_HOME`，否則為 `ANDROID_NDK_ROOT`，否則為 Android SDK（`ANDROID_HOME`，否則為 `ANDROID_SDK_ROOT`，否則為 Android Studio 存放它的位置）中最新的 `ndk/<version>`。沒有 NDK 時，錯誤訊息會說明找過哪些位置，並提示設定 `ANDROID_NDK_HOME`；NDK 中沒有該 level 的函式庫時是錯誤，錯誤訊息會列出它擁有的 level。`nabicc` 從同一個 NDK 取用 bionic 的標頭檔（[9.13.7](../009-tools/913-c-compiler.md#9137-標頭檔與-c-函式庫)）。`target_os` 是 `"android"`，`target_env` 不存在。核心是 Linux，因此 `std` 在 `target_os = "linux"` 底下擁有的東西（`std::os::linux`、`open` 旗標……）在 Android 上也有。執行檔是給 `/system/bin/linker64` 的 PIE。`x86_64-linux-android` 為 x86_64 Android ABI `x86-64-v2`（SSE4.2、POPCNT）以及最大 16 KiB 的分頁建置。應用程式的行程在 seccomp 過濾器下執行，對 bionic 不會發出的系統呼叫會終止行程，因此執行環境在那裡不自行發出任何系統呼叫：`ChildExt::pidfd` 擲回錯誤（`ENOSYS`），`Metadata::created` 擲回 `Unsupported`，而檔案到檔案的複製經由 `sendfile`。在 Android 14（API level 34）之前，bionic 的 `posix_spawn` 無法變更子行程的目錄，因此帶有 `current_dir` 的 `Command` 會在那裡先 fork 再 exec。

裝置目標 `aarch64-linux-android`（Android arm64，API level 29 以上）仍在進行中（0.6.2）。它的建置方式與 `x86_64-linux-android` 相同，上文也適用於它，只是它為最大 64 KiB 的分頁建置。

這些目標涵蓋了科學、資料、Jupyter 與桌面工作的大多數使用者。

### 32 位元支援

由於支援 32 位元目標，整份規格都定義為能在 32 位元與 64 位元環境中運作。物件標頭配置在兩種位元寬度下相同。64 位元整數的原子運算在 32 位元環境上使用目標原生的 64 位元原子指令（i686 上是 `cmpxchg8b`，ARMv7 上是 `ldrexd` / `strexd`），沒有這類指令時則使用系統原子函式庫（riscv32，進行中：使用執行環境自己的，因為它只有一個執行緒）。`usize` 的位元寬度自動跟隨目標原生的指標寬度。在 wasm32 上，堆積是線性記憶體，依需求成長，上限為 wasm32 的 4 GiB。在其他 32 位元目標上，垃圾回收器可用的堆積約為 3.5 GB：區塊堆積最多保留 2 GiB 的連續空間（作業系統拒絕時會更少），位址空間的其餘部分存放大型物件，而 Windows x86 映像檔是 large-address-aware 的。在 riscv32 上，堆積是板子分配出來的記憶體（0.6.2）。在 64 位元上，它實質上沒有限制。時間在 32 位元目標上也使用 64 位元介面（`time64`），因此 `SystemTime` 在任何地方都有相同的範圍。

### 測試

CI 依下表建置並測試已支援的目標。等級是 `nabic` 的最佳化等級，語料是編譯器的端對端與差分測試套組。

| 目標 | 每個 pull request | 合併佇列（合併的必要條件） | 每夜 |
|---|---|---|---|
| `x86_64-unknown-linux-gnu` | 所有測試，語料於 `-O0` | 所有測試，語料於 `-O0` | 所有測試；語料於 `-O0`、`-O1`、`-O2`、`-O3`、`-Os` 與 `-O2 -g`；GC 壓力測試於 `-O2` |
| `x86_64-pc-windows-msvc` | 所有測試，語料於 `-O0` | 所有測試，語料於 `-O0` | 所有測試；語料於 `-O0` 與 `-O2` |
| `aarch64-apple-darwin` | | 所有測試，語料於 `-O0` | 所有測試；語料於 `-O0`、`-O1`、`-O2`、`-O3`、`-Os` 與 `-O2 -g` |
| `aarch64-unknown-linux-gnu` | | 所有測試，語料於 `-O0` | 所有測試；語料於 `-O0` 與 `-O2`；GC 壓力測試於 `-O2` |
| `aarch64-pc-windows-msvc` | | | 所有測試；語料於 `-O0` 與 `-O2`，分別用 arm64 工具鏈以及在模擬下的 x64 工具鏈 |
| `x86_64-unknown-linux-musl`、`aarch64-unknown-linux-musl` | | | 語料於 `-O0` 與 `-O2`，在能執行該目標的主機上 |
| `i686-pc-windows-msvc` | | | 語料於 `-O0`、`-O2` 與 `-O2 -g` |
| `i686-unknown-linux-gnu` | | | 語料於 `-O0` 與 `-O2` |
| `armv7-unknown-linux-gnueabihf` | | | 語料於 `-O0` 與 `-O2`，在 qemu-user 下 |
| `wasm32-wasip2`、`wasm32-wasip1-threads`、`wasm32-unknown-unknown` | | | 語料於 `-O0` 與 `-O2`，在 wasmtime 與 node 下 |
| `x86_64-unknown-freebsd` | | | 語料於 `-O2`，在 FreeBSD 14 虛擬機器中 |

變更只有在合併佇列的執行通過時才會合併。每夜工作失敗時會開一個 issue。每夜執行也會在 qemu-user 下，以 `-O0` 與 `-O2` 建置三個進行中目標（0.6.2）`riscv64gc-unknown-linux-gnu`、`powerpc64le-unknown-linux-gnu` 與 `s390x-unknown-linux-gnu` 的語料。目標與最佳化等級的其他組合，在發行前於維護者自己的測試機器上執行，而不在 CI 中；`armv7-unknown-linux-gnueabihf` 與 `armv7-unknown-linux-musleabihf` 在那裡以每個等級原生執行，所用的 arm64 機器其 CPU 也能執行 32 位元 ARM 程式碼（Neoverse N1）；`x86_64-linux-android` 只在那裡於 Android 模擬器中測試，`aarch64-apple-ios-sim` 則在 iOS 模擬器中測試。`wasm32-unknown-unknown` 也在那裡於 Chromium、Firefox 與 WebKit 中以 `-O0` 與 `-O2` 執行語料，每個程式都在頁面的專屬 Worker 中執行。同樣在那裡，WASI 目標也在 wasmtime 以外的執行環境上以 `-O0` 與 `-O2` 執行語料：`wasm32-wasip2` 在 jco 下（元件轉譯成 JavaScript，在 Node.js 上），`wasm32-wasip1-threads` 在 WAMR 的 iwasm 與 wasmer 下。元件的匯出也在 jco 下被呼叫，而這些執行環境之一無法執行的案例（執行環境的錯誤或缺口）會在那裡被略過。

## 0.6.2. 進行中

下列目標是內建的，但尚未在它們所針對的硬體上測試，因此不受支援。每一個都預計在之後的版本（0.2.0 之前）獲得支援，屆時移到 0.6.1。對 `--target` 指定其中一個名稱（無論是給 `nabic`、`nabicc` 還是 `nabi`，`--print target-spec` 也一樣）是錯誤，錯誤訊息會說明該目標仍在進行中，以及目前應該改為建置什麼；`nabic --print target-list` 不會列出它們。

| 目標 | 為什麼仍在進行中 | 目前可用 |
|--------|-----------------------|---------|
| `aarch64-apple-ios`（iOS 裝置） | 可以建置與連結；尚未在 iPhone 或 iPad 上執行 | `aarch64-apple-ios-sim`，在 iOS 模擬器中 |
| `aarch64-linux-android`（Android arm64） | 已在 qemu-user 下以 Android 系統映像檔測試；尚未在裝置上測試 | `x86_64-linux-android`，在 Android 模擬器中 |
| `riscv64gc-unknown-linux-gnu`（64 位元 RISC-V Linux） | 已在 qemu-user 下測試；尚未在 RISC-V 硬體上測試 | 無 |
| `powerpc64le-unknown-linux-gnu`（IBM Power） | 已在 qemu-user 下測試；尚未在 POWER 硬體上測試 | 無 |
| `s390x-unknown-linux-gnu`（IBM Z；大端序） | 已在 qemu-user 下測試；尚未在 IBM Z 上測試 | 無 |
| `riscv32imac-unknown-none-elf`（RISC-V 嵌入式，沒有作業系統） | 已在 qemu-riscv32 下測試；尚未在開發板上測試 | 無 |

在測試期間，它們的程式碼留在工具鏈中。維護者在環境中設定 `NABI_IN_PROGRESS_TARGETS=1` 來建置它們，這讓 `--target` 與 `--print target-list` 接受它們。這是測試工具鏈用的開關，不屬於 Nabi：它不做任何承諾，也可能在任何版本中改變或消失。本頁關於這些目標的描述，說的是它們目前建置時的樣子。

### `riscv32imac-unknown-none-elf`

`riscv32imac-unknown-none-elf` 沒有作業系統。程式與執行環境連結成一個不含 C 函式庫的靜態映像檔，並在一個執行緒上執行：與 `wasm32-wasip2` 相同，`thread::spawn` 會 panic，`thread::Builder::spawn` 擲回 `Unsupported`。檔案、行程、通訊端與 C 函式庫函式都不存在，使用它們的程式無法連結；沒有環境變數（設定環境變數會擲回 `Unsupported`），`random` 則因缺少熵來源而 panic。執行環境提供映像檔的進入點 `_start` 與其堆積；`_start` 和 Linux 行程啟動時一樣從初始堆疊取得引數，設定 TLS 區塊並呼叫 `main`。記憶體、輸出、時間與程式的結束來自四個板子函式：`nabi_bare_morecore(len)`、`nabi_bare_write(fd, ptr, len)`、`nabi_bare_clock_ns(realtime)` 與 `nabi_bare_exit(code)`。執行環境自己對 `_start` 與這些函式的定義是弱符號，並發出 Linux 系統呼叫，`qemu-riscv32` 就是這樣以使用者模式執行映像檔；板子則定義自己的版本，其啟動程式碼設定 `sp` 與 `gp` 並呼叫 `nabir_bare_start(null)`（沒有引數）。浮點運算、64 位元除法與數學函式來自執行環境（compiler_builtins），因此 `exp`、`ln` 等的結果可能與 glibc 的結果在最後一個位元上不同。

## 0.6.3. 不支援

其他每個目標都不受支援。對 `--target` 指定它的名稱，會成為未知目標（[9.1.2](../009-tools/901-nabic-cli.md#912-核心子指令)）。對下列名稱，錯誤訊息會說明原因，若有替代方案也會指出該用什麼。

| 目標 | 替代方案 |
|--------|---------|
| `arm-unknown-linux-gnueabi`（ARMv6，軟體浮點：Raspberry Pi Zero / 1） | 無 |
| `armv7-linux-androideabi`（32 位元 Android） | 無；`aarch64-linux-android` 仍在進行中（0.6.2） |
| `thumbv6m-none-eabi`、`thumbv7m-none-eabi`、`thumbv7em-none-eabi`、`thumbv7em-none-eabihf`、`thumbv8m.main-none-eabi`、`thumbv8m.main-none-eabihf`（Cortex-M） | 無 |
| `x86_64-apple-ios`（Intel Mac 上的 iOS 模擬器） | `aarch64-apple-ios-sim`（0.6.1），在 Apple Silicon Mac 上 |
| `x86_64-apple-darwin`（Intel Mac） | `aarch64-apple-darwin`，在 Apple Silicon Mac 上 |

## 0.6.4. SIMD

建置以 triple 的基準 CPU 為目標，因此二進位檔是可攜的：`x86_64` 是 x86-64 基準（SSE2），`i686` 是 `pentium4`（SSE2），`aarch64-apple-darwin` 與 `aarch64-apple-ios-sim` 是 `apple-m1`，`aarch64-apple-ios` 是 `apple-a7`（NEON，沒有 LSE atomics），其他 `aarch64` 目標是 armv8-a（NEON，沒有 LSE atomics），`armv7` 是帶 NEON 的 ARMv7，`powerpc64le` 是 POWER8（VSX），`s390x` 是 z10（沒有向量設施），`wasm32` 是 SIMD128。在 `-O0` 以上，程式碼產生器在 `x86_64`、`i686`、`aarch64`、`armv7`（僅整數 lane）、`powerpc64le` 與 `wasm32` 上為該基準向量化迴圈，在 `s390x` 與 `riscv64` 上則利用它們的向量功能向量化迴圈（[8.7.8](../008-abi/807-code-generation.md#878-迴圈向量化)）：走訪串列元素的迴圈，包括融合後的點運算式所變成的那一個迴圈（[3.3.4](../003-expressions/303-broadcast-operators.md#334-自動迴圈融合)），可以用 128 位元向量執行。向量化的迴圈計算出與純量迴圈相同的結果，並在相同之處 trap。在其他目標上，每個迴圈都是純量程式碼。`-C target-cpu=<cpu>|native` 與 `-C target-feature=+avx2,...`（[9.1](../009-tools/901-nabic-cli.md)）選擇 CPU 的功能，程式碼產生器在 `x86_64` 上會使用其中一些：有 `avx2`（`x86-64-v3`、`haswell`、`znver1` 及之後的 CPU）時，向量迴圈使用 256 位元 AVX2 向量，並在由測試選出 lane 之處以遮罩儲存；有 `popcnt`、`lzcnt` 與 `bmi` 時，位元計數（`count_ones`、`leading_zeros`、`trailing_zeros`）各為一道指令；有 `fma` 時，`mul_add` 在純量與向量迴圈中都是一道 `vfmadd` 指令，但程式沒有以 `mul_add` 結合的乘法與加法絕不會被融合，因為那樣捨入結果會不同。在 `aarch64` 上它使用 `lse`，即 ARMv8.1 atomics（`apple-m1`、`cortex-a76`、`neoverse-n1` 及之後的 CPU）：原子的讀取-修改-寫入、交換或比較交換都是一道指令（`ldadd`、`swp`、`cas`……）；沒有它時，如同 `aarch64` Linux 與 Windows 的基準，每一個都是排他載入 / 儲存迴圈（`ldxr` / `stxr`，依順序要求為 `ldaxr` / `stlxr`），可在每一顆 ARMv8.0 CPU（Cortex-A53、Cortex-A72……）上執行。在 `aarch64` Linux 上的 `-O0` 以上，每一個這類運算也會測試一個旗標，該旗標由執行環境在啟動時依核心的 `AT_HWCAP` 設定，CPU 有 LSE 時就採用 LSE 指令。有 `rcpc`，即 ARMv8.3 RCpc 載入（`apple-m1`、`cortex-a76`、`neoverse-n1` 及之後的 CPU）時，要求 `Acquire` 的原子載入在 `-O0` 以上是 `ldapr`，較早的 release 儲存可以越過它，這是 acquire 載入所允許的；否則是 `ldar`。有 `dotprod`（同樣這些 CPU）時，加總位元組乘積的迴圈（`s += (a[i] as i32) * (b[i] as i32)`，`i8` 或 `u8`）一次執行十六個乘積，每個 lane 四個（`sdot`、`udot`）。`fullfp16`、`crc`、`sve` 與 `sve2` 不改變任何程式碼。在 `s390x` 上它使用 `vector`，即 `z13` 及之後 CPU 的向量設施：128 位元的向量迴圈，浮點 lane 為 8 位元組，有 `vector-enhancements-1`（`z14` 及之後）時也可為 4 位元組；基準 z10 沒有向量暫存器。在 `riscv64` 上它使用 `v`（RVV 1.0，`-C target-feature=+v`）：128 位元的向量迴圈，任何 V 機器無論向量長度為何都容納得下。在 `riscv64` 與 `riscv32` 上它使用 `zbb`：32 或 64 位元整數以常數旋轉（`rotate_left`、`rotate_right`，或拼出旋轉的兩個移位與一個 or）是一道指令（`rori`），否則是兩個移位與一個 or。在 `armv7` 上它使用 `vfp4`：`mul_add` 是一道 `vfma` 指令。`armv7` 上的 `-C target-feature=-neon`、`powerpc64le` 上的 `-vsx` 與 `wasm32` 上的 `-simd128` 會讓每個迴圈都以純量建置（預設的 `+simd128` 會重新開啟 SIMD128）。其他功能，以及 `i686` 上的每一個功能，都不改變任何程式碼：只指定這些功能的建置會發出這樣的警告。CPU 名稱代表其功能（`x86-64-v2`、`x86-64-v3`、`x86-64-v4`、`haswell`、`znver4`、`apple-m1`、`neoverse-n1`、`pwr9`、`z13`……）；未知的名稱是錯誤，錯誤訊息會列出已知的名稱。`native` 會在基準之上加入建置機器的功能（在 `nabic` 執行時偵測）：產生的二進位檔可能無法在另一台機器上執行。除了那個旗標之外，產生的程式碼不在執行時偵測功能，也沒有任何東西會被編譯成多個版本以便在執行時挑選，`par::map` 也是如此（[stdlib 2.19.6](../../refs-standard-libraries/200-core/219-par.md#2196-simd--gpu-整合)）。

可攜的 SIMD 向量，即帶有逐 lane 運算子的 `Simd<T, N>`，是 `core::simd`（[stdlib 2.24](../../refs-standard-libraries/200-core/224-simd.md)）；它們的運算在上述目標上於每個最佳化等級都是向量程式碼，在其他地方則是 lane 迴圈。

## 0.6.5. 目標規格

每個目標都由一份目標規格描述；編譯器、連結器與執行環境建置不讀取關於目標的任何其他資訊。目標以 `--target` 指定：內建名稱（`nabic --print target-list`：0.6.1 的目標），或是存放 NABION 值 `TargetSpec { .. }` 的檔案 `<name>.nabion` 的路徑（自訂目標，以檔案的主檔名命名）。

`nabic --print target-spec [--target=<t>]` 以該形式印出目標的規格，因此自訂目標通常從內建目標的複本開始。自訂檔案必須提供每個欄位；未知的欄位是錯誤。自訂目標的執行環境，以及（若目標附帶）其 C 函式庫，來自工具鏈的 `lib/nabi/targets/<name>/`；沒有該目錄時，需要連結的建置會失敗並指出它。

| 欄位 | 意義 |
|-------|---------|
| `c_target`、`deployment_target` | clang 接受的目標 triple（`NABI_CC`，以及 `nabi bindgen` 的 C 前端）；附加在其後的最低作業系統版本（Mach-O；macOS 上由 `MACOSX_DEPLOYMENT_TARGET` 覆寫，iOS 上由 `IPHONEOS_DEPLOYMENT_TARGET` 覆寫；Android：API level，由 `ANDROID_PLATFORM` 覆寫） |
| `arch`、`os`、`env`、`abi`、`family`、`endian`、`pointer_width` | `target_*` cfg 值（[6.3.2](../006-meta-system/603-cfg.md)）；空的 `env` / `abi` / `family` 不設定鍵。`abi` 在 Apple 模擬器上是 `"sim"`，否則為空 |
| `vendor` | 僅供參考 |
| `wide_align`、`max_atomic_width`、`c_abi` | `i64` / `u64` / `f64` 的對齊、最寬的原子型別、C 呼叫慣例（[8.6.3](../008-abi/806-calling-convention.md)） |
| `cpu`、`features` | 以 clang 名稱表示的預設 CPU 與功能（0.6.4 的 SIMD 基準） |
| `object_format`、`linker`、`pre_link_args`、`crt_objects`、`late_link_args` | 目的檔格式（`ObjectFormat::MachO`、`Elf`、`Coff`、`Wasm`）；`nabic` 的連結器產生什麼（`Linker::MachO`、`Elf`、`Coff`、`Wasm`，或 `WasmComponent`：把 wasm 模組做成 WASI preview 2 元件），它也決定連結引數的命令列風格；目標自己的連結器引數；C 執行環境目的檔；放在輸入之後的引數或檔案 |
| `dynamic_linking`、`dynamic_loader`、`libc` | 共用函式庫支援、ELF 直譯器、C 函式庫以及它是否隨工具鏈附帶 |
| `rust_target` | 建置執行環境所針對的 Rust 目標 |
| `exe_suffix`、`staticlib_prefix` / `staticlib_suffix`、`dylib_prefix` / `dylib_suffix`、`c_symbol_prefix` | 檔案與符號命名 |
| `stack_size`、`stack_probes`、`threads` | 主執行緒的堆疊、堆疊探測、執行緒支援 |
| `native_backend`、`repl` | 程式碼產生器接受此目標；REPL 主機在其上執行 |

規格具有穩定的雜湊；每個建置快取都以它為鍵。
