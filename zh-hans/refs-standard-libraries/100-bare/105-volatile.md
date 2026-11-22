# 1.5. volatile 内存访问

`bare::volatile` 模块提供编译器不会优化掉的内存读写。在硬件寄存器、MMIO、信号处理函数和 DMA 缓冲区等每一次访问都必须按原样执行的地方，这条通道必不可少。

## 1.5.1. 动机

编译器可以自由地删除、重排或合并普通的读写。看下面的代码。

```nabi
let p: Ptr<u32> = mmio_register;
unsafe {
    p.write(1);                // compiler: possible dead store (overwritten by the next write)
    p.write(2);                // the actual hardware must receive both, 1 → 2
}
```

编译器可以把第一次写入视为死存储并删除，而有些硬件只有先收到 1 再收到 2 才能正确工作。`volatile` 拒绝这类优化，强制访问严格按写法发生。

```nabi
unsafe {
    bare::volatile::write(p, 1);    // always writes 1 to the hardware
    bare::volatile::write(p, 2);    // always writes 2
}
```

## 1.5.2. API

API 只有读和写两个函数。

```nabi
// bare::volatile
unsafe fn read<T>(src: ConstPtr<T>) -> T;
unsafe fn write<T>(dst: Ptr<T>, value: T);
```

同样的操作也可以通过方法 `Ptr<T>::read_volatile` / `write_volatile` 使用，这种写法往往更自然（[1.1.2](101-ptr.md#112-ptrtstrict-provenance-api)）。`bare::mem` 没有 volatile 函数：这两个函数就是自由函数路径。

## 1.5.3. 用途

需要 `volatile` 的地方一般属于以下几种模式。

| 场合                   | 原因                                  |
|------------------------|---------------------------------------|
| MMIO 寄存器            | 硬件即使在读取时也有副作用            |
| 内存映射设备           | DMA 等                                |
| 信号处理函数中的变量   | 在信号中被修改，在外部必须可见        |
| 自旋锁重试循环         | 反复读取被另一个线程修改的值          |

## 1.5.4. 注意：不等同于原子操作

一个常见的误解：`volatile` 只阻止编译器优化，不提供任何原子性保证。

- 跨线程时仍会发生竞态条件
- 不强制任何内存顺序

因此线程之间共享的变量必须使用 [`Atomic<T>`](../200-core/216-sync.md#2162-atomict无锁原子操作)，`volatile` 应当只作为**硬件 MMIO** 的工具使用。

## 1.5.5. 降级

`volatile::read` 和 `volatile::write` 降级为对所指类型的一次 load / store，优化器把它视为副作用：它永远不会被删除、合并或移过另一次内存访问；它们调用 `ConstPtr` / `Ptr` 的方法 `read_volatile` / `write_volatile`，其内建函数是 `nabi.ptr.read_volatile` 和 `nabi.ptr.write_volatile`。

原子操作在 [2.16 sync](../200-core/216-sync.md) 中详细介绍；本章只对非原子的情况具有权威性。
