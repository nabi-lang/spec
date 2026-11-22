# 1.5. 揮發性記憶體存取

`bare::volatile` 模組提供編譯器不會最佳化掉的記憶體讀取 / 寫入。在硬體暫存器、MMIO、訊號處理常式、DMA 緩衝區等每一次存取都必須照寫法執行的地方，這個通道不可或缺。

## 1.5.1. 動機

編譯器可以自由地移除、重新排列或合併一般的讀取 / 寫入。請看下面的程式碼。

```nabi
let p: Ptr<u32> = mmio_register;
unsafe {
    p.write(1);                // compiler: possible dead store (overwritten by the next write)
    p.write(2);                // the actual hardware must receive both, 1 → 2
}
```

編譯器可能把第一次寫入視為無效儲存（dead store）而刪除，但有些硬體必須先收到 1 再收到 2 才能正確運作。`volatile` 拒絕這類最佳化，強制存取完全照寫法發生。

```nabi
unsafe {
    bare::volatile::write(p, 1);    // always writes 1 to the hardware
    bare::volatile::write(p, 2);    // always writes 2
}
```

## 1.5.2. API

API 只有讀取和寫入兩個函式。

```nabi
// bare::volatile
unsafe fn read<T>(src: ConstPtr<T>) -> T;
unsafe fn write<T>(dst: Ptr<T>, value: T);
```

同樣的操作也可以透過方法 `Ptr<T>::read_volatile` / `write_volatile` 使用，讀起來往往更自然（[1.1.2](101-ptr.md#112-ptrt嚴格來源-api)）。`bare::mem` 沒有揮發性函式：這兩個就是自由函式的路徑。

## 1.5.3. 用途

需要 `volatile` 的地方通常屬於以下幾種模式。

| 位置                   | 原因                                  |
|------------------------|---------------------------------------|
| MMIO 暫存器            | 硬體即使在讀取時也有副作用 |
| 記憶體映射裝置         | DMA 等                                |
| 訊號處理常式的變數     | 在訊號中變更，必須在外部可見 |
| 自旋鎖重試迴圈         | 重複讀取另一個執行緒變更的值 |

## 1.5.4. 注意：與原子操作不同

常見的誤解：`volatile` 只阻擋編譯器最佳化，不提供任何原子性保證。

- 執行緒之間仍然會發生競爭條件
- 不強制任何記憶體順序

因此執行緒之間共用的變數必須使用 [`Atomic<T>`](../200-core/216-sync.md#2162-atomict無鎖原子)，`volatile` 應當只保留作為**硬體 MMIO** 的工具。

## 1.5.5. 降階

`volatile::read` 和 `volatile::write` 降階為一次以指向型別進行的載入 / 儲存，最佳化器將其視為副作用：它永遠不會被移除、合併，也不會被移到另一個記憶體存取的另一側；它們呼叫 `ConstPtr` / `Ptr` 的方法 `read_volatile` / `write_volatile`，其內建函式為 `nabi.ptr.read_volatile` 和 `nabi.ptr.write_volatile`。

原子操作在 [2.16 sync](../200-core/216-sync.md) 中詳細說明；本章只對非原子的情況具權威性。
