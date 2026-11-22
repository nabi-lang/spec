# 9.13. `nabicc`：C 編譯器

工具鏈以它自己的 C 編譯器編譯 C。它以與 `nabi bindgen`（[9.10.8](910-nabi-frontend.md#nabi-bindgenc-標頭檔繫結)）相同的前端讀取 C，把它降階為 nabi-mir，並以 `nabic` 所使用的程式碼產生器與連結器編譯並連結它（[0.2.4](../000-meta/002-pipeline.md#024-程式碼產生)）。沒有其他 C 編譯器、組譯器或連結器參與，也不需要安裝任何一個。

前端也讀取 Objective-C 標頭檔，供 `ffi::objc::import` 與 `nabi bindgen -x objective-c` 使用（[stdlib 1.10.12](../../refs-standard-libraries/100-bare/110-objc.md#11012-headers-ffiobjcimport)）：只有宣告（`@interface`、`@protocol`、方法、屬性），預先定義 `__OBJC__`、`__OBJC2__` 與 ARC 的功能，`#import` 每個檔案只讀取一次。`nabicc` 本身只編譯 C；`.m` 檔案不是 C 原始碼。

## 9.13.1. `nabicc` 與 `nabi cc`

C 編譯器有兩種行為相同的形式：

- `nabicc`，獨立的執行檔。它安裝在工具鏈的 `bin/` 中，並作為代理程式安裝在 `$NABI_HOME/bin/` 中，位於 `nabi`、`nabic` 與 `nabils` 旁邊（[9.10.6](910-nabi-frontend.md#9106-toolchain)）。
- `nabi cc`。`nabi` 以 `cc` 之後的引數原樣執行它自己工具鏈的 `nabicc`，就像它委派給 `nabic` 一樣（[9.10.1](910-nabi-frontend.md#9101-子指令與全域選項)）。`nabi` 的全域選項可以出現在 `cc` 之前；其後的每個字詞都屬於 C 編譯器。

```
nabicc [options] <file>...
nabi cc [options] <file>...
```

命令列與 gcc 及 clang 所接受的相同，因此兩種形式都可以作為建置系統直接替換的 C 編譯器：`CC=nabicc` 與 `CC="nabi cc"` 都能搭配 make、CMake、autoconf 與 Meson 運作。

```
make CC="nabi cc"
cmake -DCMAKE_C_COMPILER=nabicc -B build
```

訊息會指出所執行的形式：`nabi cc: error: ..` 或 `nabicc: error: ..`。成功時結束碼為 `0`，原始碼編譯失敗、選項錯誤或連結失敗時為 `1`。編譯器內部錯誤以 [9.1.6](901-nabic-cli.md#916-結束碼) 的結束碼結束。

## 9.13.2. 語言

C89、C99、C11、C17 與 C23 被當作一種方言讀取：為其中任何一個所寫的程式都能編譯，而較新的語法（`_Static_assert`、`typeof`、`[[attributes]]`、作為關鍵字的 `bool`、二進位字面值、數字分隔符）不論標準為何都會被接受。`-std=` 選擇程式經由其巨集所看到的標準：

| `-std=` | `__STDC_VERSION__` |
|---------|--------------------|
| `c89`、`c90`、`gnu89`、`gnu90`（也包括 `-ansi`） | 未定義 |
| `c99`、`gnu99` | `199901L` |
| `c11`、`gnu11` | `201112L` |
| `c17`、`c18`、`gnu17`、`gnu18` | `201710L` |
| `c23`、`c2x`、`gnu23`、`gnu2x` | `202311L` |

也接受 `iso9899:<year>` 的拼法。預設為 `gnu17`。`c` 形式（不是 `gnu`）還會定義 `__STRICT_ANSI__`。其他任何標準都是錯誤，錯誤訊息會列出這些標準。

會讀取系統標頭檔所使用的 GNU、Clang、Apple 與 Microsoft 擴充：`__attribute__((..))`、`__declspec(..)`、`__asm__("symbol")` 標籤、`__extension__`、`typeof`、陳述式運算式、`__builtin_*` 呼叫（9.13.4）、nullability 限定詞、匿名 struct 與 union、`#pragma pack`、`#include_next`、`__has_include`、`__has_attribute`、`__has_builtin` 與 `__has_feature`。型別依目標的 C ABI 配置，與 clang 為該目標配置的方式相同。

預先定義的巨集是 clang 為該目標定義的巨集（`__x86_64__`、`__aarch64__`、`__linux__`、`__APPLE__`、`_WIN32`、`__SIZEOF_LONG__`、非 MSVC 目標上的 `__GNUC__`、MSVC 目標上的 `_MSC_VER`……），再加上 `__nabicc__`（`1`）。向量擴充及其功能巨集（`__SSE__`、`__SSE2__` .. `__AVX512F__`、`__ARM_NEON`、`__ARM_FEATURE_*`、`__wasm_simd128__`、`__riscv_vector`、`__ALTIVEC__`……）沒有定義，`__SIZEOF_INT128__` 也沒有：目前還沒有向量內建函式，因此檢查這些巨集的函式庫會走其純量路徑。

## 9.13.3. 它拒絕什麼

下列構造是位於其 `file:line:col` 的錯誤。每個錯誤都會說明應改寫成什麼。在 `inline` 或 `static` 函式的主體中，只有在該單元呼叫該函式時才是錯誤：沒有任何程式碼使用的標頭檔 inline 函式（C 函式庫的 SIMD `wmemchr`、`cpuid` 輔助函式）不會阻止建置。

| 構造 | 改寫為 |
|-----------|---------------|
| 行內組合語言（`asm`、`__asm__`），但空的屏障（`asm("")`、`asm volatile("" ::: "memory")`）以及不帶運算元的單獨自旋提示（`"pause"`、`"yield"`、`"nop"`）除外，它們會編譯為什麼都沒有 | 以 C 撰寫的程式碼，或選擇函式庫可攜程式碼路徑的巨集 |
| 標籤的位址（`&&label`）與計算式 `goto *p` | `switch` |
| 可變長度陣列（大小不是常數的陣列），但參數除外，它是指標 | 固定大小，或 `malloc` 與 `free` |
| `alloca`（`__builtin_alloca`、`_alloca`） | `malloc` 與 `free`，或固定大小的陣列 |
| 對 `_Complex` 值的算術（`__real__`、`__imag__`、運算子） | 放在兩個變數中的實部與虛部 |
| `__int128` 與 `unsigned __int128` | 兩個 64 位元的半部 |
| `long double` 比 `double` 寬之處的 `long double` 值（x86 與 x86-64 Linux 與 FreeBSD 上的 80 位元 x87；aarch64、riscv64、powerpc64le 與 s390x Linux、Android、riscv32 與 wasm32 上的 128 位元） | `double`（許多函式庫接受 `-DLONGDOUBLE_TYPE=double` 之類的選項） |
| 對 `_Float16`、`__fp16` 與 `__bf16` 的算術 | 先轉換為 `float` |
| 定義執行緒區域變數（`_Thread_local`、`thread_local`、`__thread`、`__declspec(thread)`）；其 `extern` 宣告，例如 wasi-libc 的 `errno`，會被讀取並使用 | 經由參數傳遞的狀態，或保存在每個執行緒各有一份的結構中 |
| `aarch64-pc-windows-msvc` 上的可變參數函式定義（其他目標會編譯它們） | 一個 `va_list` 版本 `int vf(const char *fmt, va_list ap)`，由非可變參數的包裝函式呼叫 |
| Windows 結構化例外處理（`__try`、`__except`、`__finally`、`__leave`） | 不使用它建置的函式庫（sqlite：`-DSQLITE_OMIT_SEH`） |
| 向量型別（`__attribute__((vector_size(N)))`、`__m128`、`float32x4_t`、SVE 型別） | 函式庫的純量程式碼路徑（其 `NO_SIMD` / `NO_SSE` 選項） |
| `__builtin_frame_address`、`__builtin_return_address` 以及 9.13.4 未列出的內建函式 | 以 C 撰寫的運算，或 C 函式庫的函式 |
| K&R 風格的定義（`int f(a, b) int a; ..`） | 原型：`int f(int a, int b)` |
| 隱含的 `int`（`static x;`） | 型別：`static int x;` |
| 呼叫未宣告的函式 | 宣告它之標頭檔的 `#include`，或在呼叫之前寫出其原型 |

在 Apple 目標、Windows 與 32 位元 Arm 上，由於目標的 C ABI 讓它是 64 位元，`long double` 就是 `double`；在那裡它會以 `double` 編譯。

被拒絕的選項與輸入：

| 選項或輸入 | 錯誤 |
|-----------------|-------|
| `-shared` | `nabi cc` 連結執行檔；請以 `ar` 建置靜態函式庫（或對套件執行 `nabi build`） |
| `.s`、`.S` 或 `.asm` 原始碼 | `nabi cc` 不進行組譯；請以 C 撰寫該函式，或連結在別處建置的目的檔 |
| C++ 或 Objective-C 原始碼（`.cc`、`.cpp`、`.cxx`、`.C`、`.m`、`.mm`），或語言不是 `c`、`c-header` 或 `none` 的 `-x` | `nabi cc` 只編譯 C；C++ 與 Objective-C 需要它們自己的編譯器 |

## 9.13.4. 內建函式

這些內建函式不需要宣告就能編譯。每個都帶有 `__builtin_` 前綴（`__builtin_expect`）；最後幾列中的名稱有它們自己的前綴。

| 內建函式 | 意義 |
|---------|---------|
| `va_start`、`va_arg`、`va_end`、`va_copy` | 依目標之 C ABI 的可變引數，與 `<stdarg.h>` 使用它們的方式相同。在沒有 `...` 的函式中使用 `va_start` 是錯誤 |
| `expect`、`expect_with_probability` | 該值；提示會被忽略 |
| `assume`、`assume_aligned`、`prefetch` | 該值，或什麼都沒有；對程式碼沒有影響 |
| `unreachable` | 若到達則為未定義行為 |
| `trap`、`debugtrap`（也包括 `__debugbreak`） | 以 trap 指令停止程式 |
| `constant_p` | 對整數或浮點數常數為 `1`，否則為 `0` |
| `object_size`、`dynamic_object_size` | 未知：type `0` 與 `1` 為 `(size_t)-1`，type `2` 與 `3` 為 `0` |
| `clz`、`ctz`、`popcount`（各自帶有 `l` 與 `ll`）、`bswap16`、`bswap32`、`bswap64` | 位元計數與位元組交換 |
| `huge_val`、`inf`、`nan`、`nans`（各自帶有 `f` 與 `l`） | 浮點數常數 |
| `isnan`、`isinf`、`isinf_sign`、`isfinite`、`isnormal`、`signbit`、`fpclassify`、`isgreater`、`isgreaterequal`、`isless`、`islessequal`、`islessgreater`、`isunordered` | 分類與比較，供 `<math.h>` 巨集使用 |
| `add_overflow`、`sub_overflow`、`mul_overflow` 以及帶型別的形式（`sadd_overflow`、`umulll_overflow`……） | 存入回繞結果並回傳是否溢位的算術 |
| 記憶體、字串與數學函式（`memcpy`、`memset`、`strlen`、`strcmp`、`malloc`、`abs`、`sqrt`、`sin`、`pow`、`fabsf`……） | 對 C 函式庫中去掉前綴之同名函式的呼叫 |
| 強化版呼叫（`__builtin___memcpy_chk`、`__builtin___snprintf_chk`……） | 對一般函式（`memcpy`、`snprintf`）的呼叫：物件大小未知（見上文），因此檢查一定會通過 |
| `offsetof`（`__builtin_offsetof`） | 成員的偏移，一個常數運算式 |
| `__atomic_*`、`__sync_*`、`__c11_atomic_*`、`_InterlockedIncrement`、`_InterlockedDecrement`、`_InterlockedExchange`、`_InterlockedExchangeAdd`、`_InterlockedCompareExchange`、`_ReadWriteBarrier` | 具有各內建函式之意義的載入、存入、交換、比較並交換以及取值並運算；對最多兩個指標大小的 2 的冪次大小，`__atomic_is_lock_free` / `__atomic_always_lock_free` 為 `1` |

`__func__`、`__FUNCTION__` 與 `__PRETTY_FUNCTION__` 是外圍函式的名稱。

> [!WARNING]
> 原子內建函式會編譯為一般的（volatile）載入與存入，以及一般的讀取-修改-寫入序列，而柵欄（`__atomic_thread_fence`、`__sync_synchronize`、`_ReadWriteBarrier`）會編譯為什麼都沒有。它們在單一執行緒中給出各內建函式的結果，但在執行緒之間不是原子的。經由它們在執行緒之間共用記憶體的 C 程式碼，在 `nabicc` 下目前還不正確。

## 9.13.5. 選項

| 選項 | 意義 |
|--------|---------|
| `-c` | 把每個原始碼編譯為目的檔；不連結 |
| `-S` | 把每個原始碼編譯為組合語言文字（`<stem>.s`） |
| `-E` | 只進行前置處理，輸出到標準輸出或 `-o` |
| `-o <file>` | 輸出。搭配 `-c`、`-S` 或 `-E` 時只接受一個原始碼 |
| `-I <dir>`、`-isystem <dir>`、`-iquote <dir>`、`-idirafter <dir>` | 標頭檔搜尋目錄，搜尋方式與 gcc 相同 |
| `-include <file>` | 在原始碼之前讀取 `<file>` |
| `--sysroot <dir>`、`-isysroot <dir>` | 目標的標頭檔與 C 函式庫（9.13.7） |
| `-nostdinc`、`-nostdlibinc`、`-nobuiltininc` | 排除系統標頭檔、C 函式庫的標頭檔或內建標頭檔 |
| `-D <name>[=<value>]`、`-U <name>` | 定義或移除巨集 |
| `-Wp,<opts>` | 本表中的前置處理器選項，以逗號分隔 |
| `-std=<std>` | 標準（9.13.2） |
| `-O0`、`-O1`、`-O2`、`-O3`、`-Os`、`-Oz` | 最佳化等級（預設 `-O0`）；`-O` 是 `-O1`，`-Og` 是 `-O1`，`-Ofast` 是 `-O3` |
| `-g`、`-g0` | 開啟或關閉偵錯資訊（`-ggdb`、`-g2`、`-g3`、`-gdwarf*`、`-gcodeview` 會開啟它） |
| `-l<lib>`、`-L<dir>`、`-Wl,<opts>` | 連結用的函式庫、函式庫目錄與連結器選項 |
| `-M`、`-MM` | 印出原始碼及其所讀取之標頭檔的 make 規則，不進行編譯；`-MM` 排除系統標頭檔 |
| `-MD`、`-MMD` | 在編譯時把該規則寫到 `<output or stem>.d`；`-MMD` 排除系統標頭檔 |
| `-MF <file>` | 把規則寫到 `<file>` |
| `-MT <target>`、`-MQ <target>` | 規則的目標（可重複）；預設為目的檔 |
| `-MP` | 為每個標頭檔加上一條空規則，讓被刪除的標頭檔不會讓 make 停止 |
| `--target=<triple>`、`-target <triple>` | 為另一個目標編譯（9.13.7） |
| `-x c` | 不論副檔名為何，把之後的輸入當作 C 讀取 |
| `-dumpmachine` | 印出目標的 C 三元組 |
| `-dumpversion`、`-dumpfullversion` | 印出工具鏈的版本 |
| `--version` | 印出 `nabicc <version>`；有 `-v` 時，還會印出 `host: <triple>` |
| `-v` | 詳細模式（搭配 `--version`） |
| `--help`、`-h` | 印出用法 |

`--target` 接受 Nabi 的目標名稱（`nabi --print target-list`，[0.6](../000-meta/006-platform-targets.md)），以及 clang 與 gcc 為它們所接受的三元組：`x86_64-linux-gnu` 是 `x86_64-unknown-linux-gnu`，`arm64-apple-macos` 是 `aarch64-apple-darwin`，`aarch64-linux-musl` 是 `aarch64-unknown-linux-musl`，`i386-linux-gnu` 是 `i686-unknown-linux-gnu`，`armv7-linux-gnueabihf` 是 `armv7-unknown-linux-gnueabihf`，`armv7-linux-musleabihf` 是 `armv7-unknown-linux-musleabihf`，`riscv64-linux-gnu` 是 `riscv64gc-unknown-linux-gnu`。不屬於這些的名稱是錯誤。

下列選項會被接受但沒有效果：其他每個 `-W*`（警告）、`-f*` 與 `-m*` 選項、`--param`、`-fPIC`、`-fpic`、`-fPIE`、`-fpie`、`-pie`、`-no-pie`、`-rdynamic`、`-pthread`、`-pedantic`、`-pedantic-errors`、`-w`、`-pipe`、`-s`、`-static` 與 `-Qunused-arguments`。程式碼一律與位置無關。其他任何選項都是錯誤，錯誤訊息會列出上述選項。

以 `.c`、`.h` 或 `.i` 結尾的輸入是 C 原始碼；其他每個輸入（目的檔、封存檔、`.lib` 檔案）都依其在命令列上的位置加入連結。各原始碼平行編譯。沒有 `-o` 時，執行檔是 `a.out`（Windows 目標為 `a.exe`），目的檔是目前目錄中的 `<stem>.o`（Windows 目標為 `<stem>.obj`）。沒有原始碼也沒有連結輸入時是錯誤（`no input files`）。

## 9.13.6. 程式碼產生

每個 C 函式都被降階為 nabi-mir 中的 C 函式：以其 C 名稱、採用 C ABI 的主體，與帶有 `#[export_name]` 的 `extern "C" fn` 相同（[5.9.10](../005-modules/509-extern-block.md#5910-extern-fn-定義對外公開)）。Nabi 的原生程式碼產生器編譯它，在 `-O1` 以上會套用 nabi-mir 最佳化器與程式碼產生器的最佳化（[9.2.10](902-aot-pipeline.md#9210-最佳化等級)）。整數算術遵循 C：沒有溢位檢查。

呼叫慣例是目標的 C ABI，與 Nabi 的 `extern "C"` 所使用的相同（[8.6](../008-abi/806-calling-convention.md)），因此從 Nabi 呼叫由 `nabicc` 編譯的 C，或從該 C 呼叫 Nabi 的 `extern "C" fn`，在構造上就會相符。浮點數運算不會被縮約：`a * b + c` 是一次乘法與一次加法，絕不是融合乘加，與 Nabi 程式碼相同。

`-g` 發出 DWARF（Linux、Apple、wasm32）或 CodeView（Windows）的行號表與型別。在 Apple 目標上，連結會把 DWARF 彙整成執行檔旁邊的 `.dSYM` 套組；在 Windows 上，連結器會寫出 PDB（[9.2.14](902-aot-pipeline.md#9214-連結執行檔)）。

執行檔由 Nabi 的連結器對照目標的 C 函式庫（9.13.7）連結。在 ELF 目標上一律會加上 `-lm`。`-lc`、`-lpthread`、`-ldl` 與 `-lrt` 指名的是連結已經擁有的 C 函式庫部分，會被捨棄，`-nostdlib`、`-nodefaultlibs` 與 `-nostartfiles` 也是。在 Windows 上，工具鏈的啟動程式碼呼叫 C 的 `main(argc, argv, envp)`（不會呼叫 `wmain`），而連結提供 `atexit`。在 32 位元 Arm 與 wasm32 上，連結從工具鏈為該目標提供的執行環境封存檔取得編譯器輔助函式（`__aeabi_idiv`、`__multi3`……）。

## 9.13.7. 標頭檔與 C 函式庫

標頭檔與 C 函式庫來自目標：

| 目標 | 標頭檔 | C 函式庫 |
|--------|---------|-----------|
| Apple | macOS SDK：`SDKROOT`，否則為 Xcode 的或 Command Line Tools 的 | libSystem |
| Linux、glibc | 工具鏈為該目標提供的 sysroot `lib/nabi/targets/<target>/sysroot`：來自與其所連結之 C 執行環境（[9.2.14](902-aot-pipeline.md#9214-連結執行檔)）相同之固定版本 Debian 套件的 glibc 2.36 標頭檔 | 隨附的 glibc 連結 stub |
| Linux、musl | 目前尚未隨附 sysroot：在 musl 主機上使用主機自己的，否則使用 `--sysroot <dir>` | 隨附的 musl |
| Windows | 在 Windows 主機上，Windows SDK 的 UCRT 標頭檔與 Visual Studio 的 C 標頭檔：`INCLUDE`，否則為 `vswhere` 所找到的最新 Visual Studio 與 Windows SDK | UCRT，經由工具鏈自己的匯入函式庫與啟動程式碼；不連結任何 MSVC 函式庫 |
| wasm32 | wasi-libc 的標頭檔，作為該目標的 sysroot 隨附 | 隨附的 wasi-libc |
| Android | Android NDK 的 sysroot（與連結時相同的方式尋找，[0.6.1](../000-meta/006-platform-targets.md#android)），使用其針對該目標的目錄 | bionic：NDK 中該 API 等級的 crt 目的檔與函式庫 |
| riscv32（沒有作業系統） | 只有獨立式標頭檔（見下文） | 無 |

當目標就是主機，而工具鏈沒有為它隨附 sysroot 時，使用主機的 `/usr/include`。不屬於這些任何一種的目標是錯誤，錯誤訊息會指出缺少的目錄以及 `--sysroot <dir>`。`--sysroot <dir>`（或 `-isysroot <dir>`）會覆寫此表：改為搜尋其下的標頭檔（`usr/include`，在 Linux 上搭配目標的 multiarch 目錄；在 wasm32 上為 `include`）。

在 C 函式庫的標頭檔之前是內建標頭檔，即代替 clang 自己之標頭檔的獨立式標頭檔：`stddef.h`、`stdint.h`、`stdarg.h`、`stdbool.h`、`stdalign.h`、`stdnoreturn.h`、`stdatomic.h`、`limits.h`、`float.h`、`inttypes.h`、`iso646.h`，以及 [std 1.7.6](../../refs-standard-libraries/100-bare/107-ffi.md#the-c-bindgen-reads) 的內建函式標頭檔，其函式在這裡沒有定義。`-nobuiltininc` 會排除它們；`-nostdinc` 會排除每個系統目錄。

## 9.13.8. 套件的 C 原始碼

套件在 `nabi.nabion` 的 `c` 欄位中列出其 C 原始碼（[9.10.3](910-nabi-frontend.md#9103-nabinabion--nabilock)）：

```nabion
Package {
    name: "notes",
    c: {
        sources: ["vendor/sqlite3.c"],
        include: ["vendor"],
        define: { "SQLITE_THREADSAFE": "0", "NDEBUG": "" },
        std: "c17",
        flags: ["-Wall"],
    },
}
```

| 欄位 | 意義 |
|-------|---------|
| `sources` | C 檔案，相對於資訊清單 |
| `include` | 標頭檔目錄，相對於資訊清單（`-I`）；接著是套件之 `Dep::system` 函式庫的目錄 |
| `define` | 巨集（`-D<name>=<value>`）；空值會把巨集定義為空 |
| `std` | 標準：`c89`、`c99`、`c11`、`c17`、`c23` 或其 `gnu` 版本（預設 `gnu17`） |
| `flags` | 其他選項：只限 `-O*`、`-g`、`-W*`、`-f*`（不包括 `-flto`）與 `-m*` |

內建編譯器以建置的最佳化等級（`-O0`，或有 `--release` 時為 `-O2`，或所給出的 `-O`）編譯每個原始碼；`flags` 中的 `-O` 會取代它。目的檔放到 `target/<profile>/c/<package>/`，只有其輸入（它以及它所讀取的標頭檔）或選項改變的原始碼才會重新編譯。它們在套件被連結的任何地方都與套件一起連結：連結進其執行檔與測試 harness，以及相依於它之套件的執行檔。其 Nabi 程式碼經由 `extern "C"` 區塊（[5.9](../005-modules/509-extern-block.md)）呼叫 C 函式；`--target` 也會為該目標編譯 C。

`flags` 中這些以外的選項，在讀取資訊清單時是錯誤，錯誤訊息會列出 `flags` 所接受的選項。不存在的原始碼或目錄是指出其名稱的錯誤。編譯失敗的 C 原始碼會以編譯器的錯誤讓建置失敗。絕不會執行任何外部 C 編譯器，也不會有套件的程式碼在建置時執行（9.10.3）。
