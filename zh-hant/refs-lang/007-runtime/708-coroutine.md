# 7.8. 協程

[產生器](../004-functions/411-generators.md)、[非同步](../004-functions/412-async.md)與[串流](../004-functions/413-stream.md)使用相同的協程降階。

## 7.8.1. 狀態機降階

Nabi 的協程是無堆疊的，被降階為由編譯器自己建構的狀態機：編譯器必須知道框架配置，才能建構框架的 ScanTable。沒有逐架構的情境切換程式碼；在每個支援的目標上都相同。

```
yield / async / async yield block
  → lifted into a body function (HIR, like a closure)
  → state machine (MIR): resume function + frame type
  → ordinary code generation
```

`yield fn` / `async fn` 是把其主體包在這樣的區塊中並回傳一個包裝型別的 `fn`。

## 7.8.2. 協程框架

```
Frame {
    state  : i32        // suspend point to continue from next
    locals : ...        // only locals live across suspend points
}
```

- 框架是一般的 GC 物件，由配置的快速 / 慢速路徑建立（[7.2](702-allocation.md)）。每個型別都有一個框架 [ScanTable](../008-abi/802-scan-table.md)。
- 暫停期間，框架的 GC 參照由 ScanTable 追蹤；恢復函式執行期間，其區域變數由影子堆疊追蹤（[7.4.1](704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)）。
- 長期存活之任務的框架透過 sticky mark-bit 變成老物件（[7.3.2](703-gc-algorithm.md#732-sticky-mark-bit-generational-collection)）。
- 呼叫的結果（`GenIter<T>`、`AsyncFuture<T>`、`AsyncStream<T>` 及其 `Try*` 變體）是包裝框架的值（[4.1.11](../004-functions/401-function-declaration.md#result-types-of-coroutine-functions)）。

## 7.8.3. 產生器

```
1. Call count() → allocate frame, return wrapper.
2. next() → resume function continues from state. At yield, saves the value and state and returns.
3. Next next() → continues from the saved state. When finished, none.
```

由呼叫者驅動進展（`for x in g { .. }`）。提前離開產生器的 `for` 會關閉它：產生器以關閉模式恢復，並在它所等待的 `yield` 處像 `return` 一樣離開，執行其清理（[4.11.10](../004-functions/411-generators.md#41110-generator-termination)）。

## 7.8.4. 非同步

```
1. Call fetch() → allocate frame, return Future.
2. poll → resume function proceeds. If an awaited future is Pending, saves state and returns Pending.
3. Wake (parking registry, I/O driver, timer) → the task re-enters the queue and a worker polls it.
```

由排程器與執行環境驅動進展（[7.6](706-scheduler.md)）。

## 7.8.5. 串流

`async yield` 結合了兩者。在同一個框架內，`yield` 產生一個值，`await` 則以 Pending 暫停。

## 7.8.6. 配置消除

每次協程呼叫都會配置一個框架。對於由其呼叫者執行到完成的協程，沒有專用的消除；從不逃逸的框架可以像任何配置一樣被移除（[7.2.12](702-allocation.md#7212-allocation-elision)）。

## 7.8.7. `yield` / `await` 的降階

| 關鍵字         | 降階                                                                      |
|----------------|---------------------------------------------------------------------------|
| `yield expr;`  | 儲存狀態 + 回傳值（Iterator）/ 回傳 `Ready(value)`（Stream）              |
| `await expr;`  | 輪詢目標 future。若為 Pending，儲存狀態 + 回傳 `Pending`                  |
| `return expr;` | 結束狀態機 + 回傳結果                                                     |

## 7.8.8. 執行環境符號

`await` 的等待與喚醒由 `core::task` 的停駐登錄處理（由 `core::task` 透過 `nabi_task_hooks_install` 安裝的掛鉤，[8.5.4](../008-abi/805-extern-symbols.md#854-scheduler-area)）。執行環境的 `nabi_coro_frame_alloc` / `nabi_coro_park` / `nabi_coro_unpark` 供 C 與低階程式碼使用；編譯器不使用它們（[8.5](../008-abi/805-extern-symbols.md)）。

## 7.8.9. 堆疊深度

巢狀的 `await` 透過原生呼叫進行，其中外層的恢復函式呼叫內層 future 的輪詢。因此 `await` 遞迴的深度受原生堆疊限制，非常深的遞迴會造成堆疊溢位（[執行時錯誤](README.md#runtime-error-stack-overflow)）。若需要深層的遞迴，請把它拆成多個任務。

## 7.8.10. 協程框架的生命週期

框架與其他任何 GC 物件一樣，只要可到達就會存活。執行中的任務由任務佇列參照，等待中的任務由停駐登錄參照，而等待結果者則由它自己的框架參照。當沒有任何東西參照它時，GC 就會回收它。沒有明確的釋放。
