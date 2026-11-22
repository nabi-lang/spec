# 8.4. 安全點協定

為了讓 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 進行暫時的 STW（Stop-The-World）工作，每個 mutator 執行緒都必須到達一個安全點。本文件定義該同步的 [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 協定。

## 8.4.1. `yield` 旗標檢查：快速路徑

mutator 定期檢查 [TLS 槽位](803-tls-slots.md) `nabi_tls_yield_flag`（u8）。檢查依下列方式進行。

```
poll:
    flag = TLS[yield_flag]
    if flag != 0:
        call nabi_runtime_yield()    // slow path
    // proceed normally
```

在發出的程式碼中：

```
t = <nabi_tls address>            ; the hidden last argument (8.3.8)
flag = load.u8 [t + 88]           ; yield_flag at 10p+8, p = 8
if flag != 0 then bb_slow else bb_cont

bb_slow:                          ; placed out of line
    call nabi_runtime_yield()
    goto bb_cont

bb_cont:
  ; ... continue
```

這是兩道指令的快速路徑加上一個分支，而該分支幾乎總是求值為 `false`。

## 8.4.2. 發出位置

編譯器在下列位置發出安全點輪詢。

| 位置                  | 理由                      |
|-----------------------|---------------------------|
| 函式進入（或每條路徑的第一個安全點之前） | 呼叫之後就能立即與 GC 協作 |
| 迴圈回邊              | 防止無限迴圈阻擋 GC       |
| 緊接在函式呼叫之後（選用） | 若呼叫很長，能立即協作 |

詳細的發出方針由編譯器決定；本規格只保證下列最低頻率。

- 每個迴圈回邊都有輪詢，但迭代次數為至多 64 的編譯期常數、且主體不會到達任何安全點的迴圈除外；每個呼叫循環都包含一次輪詢。
- 無法到達任何安全點的函式（除了這類有界迴圈之外沒有迴圈、沒有配置、沒有間接呼叫、沒有延遲 `static`，也沒有呼叫可能到達安全點的函式）不需要輪詢。

兩者一起限制了從 yield 請求到下一個安全點的時間。

## 8.4.3. `nabi_runtime_yield`：慢速路徑

慢速路徑的簽章如下：

```
extern "C" {
    fn nabi_runtime_yield();
}
```

mutator 呼叫這個函式以與 GC 協作。其效果如下：

1. 把 mutator 之 [mutator TLS 槽位配置](803-tls-slots.md)中的 `mutator_state` 設為 `GC_COOPERATING`。
2. 公開 GC 所需的根資訊，例如[影子堆疊](../000-meta/007-glossary.md#shadow-stack)的走訪。
3. 等待 GC 完成。
4. GC 完成時，把 `mutator_state` 設回 `RUNNING`。

`yield_flag` 在 `nabi_runtime_yield` 內自動清除，或由 GC 清除。

## 8.4.4. 設定 / 清除 `yield` 旗標

當 GC 需要 STW 暫停時：

```
gc_initiate_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 1

  // wait until every mutator yields
  for each mutator thread:
    wait until mutator_state == GC_COOPERATING
```

當 GC 完成 STW 時：

```
gc_finish_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 0
  notify all (resume)
```

## 8.4.5. 快速路徑的成本

mutator 承擔的成本是對 `yield` 旗標的一次載入與一個分支。

幾乎在所有情況下旗標都是 0，因此不會走該分支，而現代 CPU 的分支預測器能以接近零的成本處理它。

以這個成本，每個執行緒都得到一個確定性的協作同步機制。

## 8.4.6. STW 階段

STW 回收（minor、帶疏散的 full、`nabi_gc_collect`）是單一次暫停。啟動回收的執行緒停止所有其他的 mutator（`GC_COOPERATING`，或在阻塞的執行環境呼叫中時為 `PARKED`），完成回收，然後讓它們恢復。

帶並行標記的完整回收有兩次暫停，即初始暫停與最終暫停（[7.5.2](../007-runtime/705-concurrent-collection.md#752-phases)）。在兩者之間，單一的標記執行緒與 mutator 並行執行。

## 8.4.7. 安全點不變條件

到達安全點時，mutator 每個存活的 GC 物件變數都必須已註冊在影子堆疊中。

編譯器保證這一點的方式，是在函式中每條路徑的第一個安全點之前，連上[屏障與根追蹤](../007-runtime/704-barriers-and-roots.md)中的影子堆疊框架：在進入處，或在到達安全點之路徑的較後位置。不會到達任何安全點的路徑不需要連上它，而框架只在連上它的地方於回傳時解開。同樣地，只會被 Nabi 直接呼叫進入的函式，可以把其進入輪詢移到緊接在連上之後：每個呼叫循環仍包含一次輪詢，因為封閉該循環的呼叫就是安全點。從 C 或透過函式值被呼叫的函式在進入時輪詢。在安全點時容納 GC 參照的區域變數會被註冊在框架中，而被註冊的字組在每次寫入時都保持最新。只有回邊輪詢看得到的區域變數，可以改為只在執行環境呼叫期間由輪詢的慢速路徑註冊（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler) 規則 4），因此唯一的安全點就是其輪詢的迴圈，在執行期間不會連上任何框架，也不會寫入任何根字組。由於回收絕不會改寫根字組，編譯器也可以保存一份私有複本（在暫存器中）：從根直接到達的物件不會被移動（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)）。熱重載程式碼是例外（8.4.13）。

## 8.4.8. 迴圈回邊輪詢

若長時間執行的迴圈不與 GC 協作，STW 就會永遠等待。因此編譯器在每個迴圈回邊上發出輪詢（8.4.2 的有界迴圈除外）。

```
loop:
    body
    poll                              // back-edge poll
    branch loop
```

這個輪詢的成本同樣是兩道指令與一個分支，與迴圈主體相比可以忽略。

輪詢留在回邊上。其慢速路徑（`nabi_runtime_yield` 呼叫）可以分支到放在迴圈標頭之前的重新進入區塊，而不是回到該邊。

編譯器偵測每個自然迴圈，並為每個迴圈發出輪詢，但 8.4.2 的有界迴圈除外。

## 8.4.9. 內嵌函式與輪詢

小函式被內嵌時，其進入輪詢可能隨之消失。不過內嵌的結果被吸收進呼叫者的主體中，因此呼叫者的進入輪詢就足夠了。

輪詢在最佳化之後、針對被編譯的程式碼放置：內嵌主體中的迴圈，和其他迴圈一樣在呼叫者中於其回邊輪詢。

## 8.4.10. 原生呼叫

位於 C 或阻塞之執行環境呼叫中的執行緒無法到達安全點，因此它算作 `PARKED`：GC 把它視為已經在協作。

- 編譯器包住每個 `extern "C"` 呼叫：呼叫期間為 `PARKED`，之後為 `RUNNING`，每次存入之後都檢查 yield 旗標。呼叫者的引數根保留在其影子堆疊框架中。快速路徑是行內的；`nabi_enter_native` / `nabi_leave_native` 是慢速路徑（[8.5.2](805-extern-symbols.md#852-runtime-area)）。對 `#[leaf]` 宣告的呼叫沒有包住：執行緒保持 `RUNNING`，並阻擋每次回收直到呼叫回傳（[stdlib 1.7.1](../../refs-standard-libraries/100-bare/107-ffi.md#leaf---short-calls)）。
- 每個 `extern "C" fn` 定義都以 `nabi_native_reenter` 開始、以 `nabi_native_resume` 結束，因此 C 回呼以 mutator 的身分執行。
- 執行環境以相同方式包住它自己的阻塞呼叫（I/O 等待、`nabi_sync_wait`、join、sleep），以及它那些不碰任何 GC 物件、只碰呼叫者框架保持存活之緩衝區且不存入任何參照的長時間計算：`int` 核心函式（乘法、除法、gcd；從 64 Ki 個 limb 起的加法、減法、移位與單 limb 除法），以及從 64 KiB 起的 `str` 搜尋與計數、UTF-8 檢查與位元組複製（[8.5.2](805-extern-symbols.md#functions-that-are-not-safepoints)）。它們的 `extern "nabi"` 宣告不是 `#[no_safepoint]`（[5.9.2](../005-modules/509-extern-block.md#592-abi-specifiers)），因此對它們的呼叫是安全點，緩衝區也有根。
- 對 `#[no_safepoint]` 執行環境函式的呼叫不會被包住。其他每個執行緒的回收都會等到它回傳，因此這樣的函式只做有限的工作；長的輸入改交給其停駐版本。偵錯版 nabic 會以 `nabi_nosp_enter()` / `nabi_nosp_leave()` 包住它（[8.5.2](805-extern-symbols.md#852-runtime-area)），它們提高與降低執行緒的 `no_safepoint` 深度（[8.3.1](803-tls-slots.md#831-tls-slot-structure)）；若執行緒在深度不為零時要為回收而停下，偵錯版執行環境會以致命錯誤失敗。發行版 nabic 兩個呼叫都不發出。
- 透過 `nabi_*` 符號（[8.5](805-extern-symbols.md)，執行環境保留的前綴）之 `extern "C"` 宣告的呼叫不會被包住：該執行環境函式以 mutator 的身分執行。

```
call_extern_c:
    mutator_state = PARKED      // seq_cst
    if yield_flag:
        nabi_enter_native()      // wake a collection waiting for this thread
    c_function()
    mutator_state = RUNNING     // seq_cst
    if yield_flag:
        nabi_leave_native()      // wait out a running collection
```

## 8.4.11. 從使用者程式碼直接呼叫

使用者程式碼也可以（很少）直接呼叫 `nabi_runtime_yield()`。

```nabi
extern "C" {
    fn nabi_runtime_yield();
}

fn long_loop() {
    for i in 0..1_000_000 {
        ...
        if i % 1000 == 0 { unsafe { nabi_runtime_yield(); } }
    }
}
```

`nabi_runtime_yield` 是 `nabi_*` 符號，因此該呼叫不會被包住（8.4.10），執行緒以執行中之 mutator 的身分讓出。編譯器已經在迴圈回邊上發出輪詢，因此使用者幾乎永遠不需要直接呼叫它。這種模式只用於行內組合語言內部等特殊情況。

## 8.4.12. 穩定性

ABI 是 `nabi_tls` 偏移 10p+8 的槽位（`nabi_tls_yield_flag`）及其型別（u8）（[8.3.1](803-tls-slots.md#831-tls-slot-structure)）。

`nabi_runtime_yield` 遵循沒有引數也沒有回傳值的簡單呼叫慣例。

## 8.4.13. 熱重載程式碼

以 `--hot`（[9.1](../009-tools/901-nabic-cli.md)）建置的程式碼，在可能改寫根字組的熱執行環境下執行：狀態遷移會把變更之型別的物件複製到新的物件中，並更新指向它們的每個參照，包括根。因此這樣的程式碼不會跨越安全點保存 GC 參照的私有複本：跨越安全點而存活的每個參照，在其之後都從其根字組讀回。

- 熱建置是 `-O0` 建置。在那裡每個區域變數都位於其框架槽位中，而那正是影子堆疊框架所註冊的字組，因此每次使用都讀取根字組。
- 對熱套件之函式的呼叫算作安全點，無論其目前的主體做什麼：它所經過的槽位可能到達一個會停下的較新版本。呼叫者為跨越它而存活的東西建立根。
- 容納熱套件之型別之值的函式，以描述子形式連上其影子堆疊框架（[7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)），即使它沒有任何根也一樣，而其描述子之前的字組存放其框架紀錄（它所容納的型別）的位址，該位址也存放在它的某個根字組中。熱執行環境在那裡找到舊程式碼中已停下的框架。

不以 `--hot` 建置之程式碼（主機程式、C）的框架保持 8.4.7 的規則；遷移不會改寫這種框架的根。
