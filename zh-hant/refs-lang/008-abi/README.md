# 8. 執行環境 ABI / 降階概觀

這一部分定義編譯器（`nabic`）與執行環境（`libnabir.a`；在 Windows MSVC 目標上為 `nabir.lib`）之間的共同契約。編譯器與執行環境透過本規格的文字連接。它們共用的程式碼是一個兩者都會建置、沒有相依的 crate，即 `nabir_abi`（`runtime/nabir_abi`），因此雙方使用同一份定義：GC ABI 常數（`nabir_abi::gc`，[8.1](801-object-header-abi.md)）、TLS 槽位配置（`nabir_abi::tls`，[8.3](803-tls-slots.md)）、模組與型別描述子（`nabir_abi::module`、`nabir_abi::typedesc`，[8.12](812-module-abi.md)、[8.13](813-type-descriptor.md)），以及常數求值必須與執行環境一致的格式規範、字串與 Unicode 常式。

## 目錄

- [8.1. 物件標頭 ABI](801-object-header-abi.md)
- [8.2. ScanTable 序列化格式](802-scan-table.md)
- [8.3. mutator 的 TLS 槽位配置](803-tls-slots.md)
- [8.4. 安全點協定](804-safepoint.md)
- [8.5. Extern "C" 符號](805-extern-symbols.md)
- [8.6. 呼叫慣例](806-calling-convention.md)
- [8.7. 程式碼產生](807-code-generation.md)
- [8.8. 降階：資料型別與運算式](808-lowering-types.md)
- [8.9. 降階：控制流程與 GC 整合](809-lowering-control.md)
- [8.10. 降階：協程與泛型](810-lowering-coroutine-generic.md)
- [8.11. 程式碼產生器契約](811-backend-contract.md)
- [8.12. 模組描述子 ABI](812-module-abi.md)
- [8.13. 型別描述子 ABI](813-type-descriptor.md)

## 兩種契約

### 1. 執行環境 ABI

編譯器產生的程式碼與 `libnabir.a` 一起連結並執行時所使用的介面。物件的記憶體配置有明確規定，讓 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 能精確地解讀編譯器所建立的物件。[Extern "C" 符號](../000-meta/007-glossary.md#extern-c-symbol)是編譯器可以呼叫的執行環境函式；每個符號的執行緒安全類別（`thread-safe` / `caller-thread-only` / `process-init` 等）定義於 [8.5.0 執行緒安全類別](805-extern-symbols.md#850-thread-safety-classes)。TLS 槽位是由編譯器發出的行內程式碼讀寫的執行緒區域位置。呼叫慣例規定函式呼叫中暫存器與堆疊的使用方式。

### 2. 降階

在[編譯管線](../000-meta/002-pipeline.md)的程式碼產生階段套用、把 Nabi 的高階語意轉譯為機器碼的規則：每個值的表示方式，以及 `tagged union`、`any` 槽位、模式比對、try/catch、協程與泛型如何變成載入、存入、分支與呼叫。

## 為什麼本規格的文字具權威性

除了那些共用的原始檔之外，編譯器與執行環境不共用任何程式碼或標頭檔。兩者遵循同一個 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 的保證，來自本規格的文字以及整合測試中的端對端驗證。

這個模型採用與 Swift ABI Stability 相同的做法：規格具權威性，實作遵循它。

## 穩定性

這一部分的契約，包括符號修飾（[9.5](../009-tools/905-symbol-mangling.md)），是工具鏈版本的一部分。在 1.0 之前，ABI 可以在每個 0.x.0 版本中改變；0.x.y 版本不會破壞它。從 1.0 起，次版本 / 修訂版本只能新增而不得破壞相容性，主版本則可以以套件管理器能偵測的方式破壞它。

預先建置之套件的 NABIHIR 標頭會記錄建置它的 `nabic`（[9.4](../009-tools/904-nabihir-bytecode.md)）。使用由不同 `nabic` 建置的套件是錯誤，錯誤訊息會要求重新建置。
