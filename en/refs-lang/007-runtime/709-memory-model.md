# 7.9. Memory model

This section defines what a thread may see of memory that other threads write. Nabi adopts the memory model of C++20 (ISO/IEC 14882:2020, [intro.multithread] and [atomics.order]), which Rust also uses; the terms below have their C++20 meaning. On top of it, this section states which library operations synchronize and what safe Nabi guarantees. Unlike 7.1 to 7.5, this section is normative.

"Thread" here means an OS thread. Tasks ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)) run on worker threads and may resume on another worker after any suspension point; the scheduler orders a task's own steps, so code in one task sees its own earlier writes, wherever it resumes. Between two tasks the rules below apply as between two threads.

## 7.9.1. Relations

- **Sequenced before**: within one thread, an evaluation sequenced before another by the order of evaluation ([3. Expressions](../003-expressions/README.md)).
- **Modification order**: every atomic object has a single total order of all modifications to it, which every thread observes consistently (C++20 coherence rules).
- **Synchronizes with**: a release operation on an atomic object synchronizes with an acquire operation on the same object that reads the value it wrote, or a later value in its release sequence. The library operations of 7.9.3 also synchronize as stated there.
- **Happens before**: the transitive closure of sequenced before and synchronizes with (C++20's simply-happens-before; Nabi has no `consume` ordering, so the two notions coincide).

A non-atomic read sees the last write to that location that happens before it, provided no other write to it is unordered with the read.

## 7.9.2. Atomic orderings

`Atomic<T>` ([stdlib 2.16.2](../../refs-standard-libraries/200-core/216-sync.md#2162-atomict---lock-free-atomics)) takes the five orderings of C++20, with their C++20 meaning:

| Ordering  | C++20                        | Meaning |
|-----------|------------------------------|---------|
| `Relaxed` | `memory_order_relaxed`       | Atomic and in the modification order; no synchronization. |
| `Acquire` | `memory_order_acquire`       | A load (or the load half of a read-modify-write) that synchronizes with the release it reads from. |
| `Release` | `memory_order_release`       | A store (or the store half of a read-modify-write) that heads a release sequence. |
| `AcqRel`  | `memory_order_acq_rel`       | A read-modify-write that is both. |
| `SeqCst`  | `memory_order_seq_cst`       | Acquire for loads, release for stores, and part of the single total order of all `SeqCst` operations. The default. |

A failed compare-exchange is a load with its failure ordering. Each operation's ordering type admits only the orderings it can take (2.16.2), so `consume` and an acquire store do not exist. Atomic operations on one object never tear and never form a data race with each other, whatever their orderings.

Nabi has no stand-alone fence: every ordering is attached to an atomic operation. `volatile` accesses ([stdlib 1.5](../../refs-standard-libraries/100-bare/105-volatile.md)) are not atomic and order nothing between threads.

## 7.9.3. Operations that synchronize

Each of the following makes the earlier operation synchronize with the later one, so everything sequenced before the first happens before everything sequenced after the second.

| Earlier | Later |
|---------|-------|
| `Mutex` unlock (a guard's release) | the next lock of that `Mutex` that succeeds (`lock`, `try_lock`, `lock_async`) |
| `RwLock` write unlock | the next `read` / `write` acquisition |
| `RwLock` read unlock | the next `write` acquisition |
| `Condvar::wait` releasing its lock | as a `Mutex` unlock; its relock as a `Mutex` lock |
| the `f` of a `Once::call_once` returning | every `call_once` of that `Once` returning, and an `is_completed` that returns `true` |
| a lazily initialized `static`'s initializer finishing | every access of that `static` ([5.3.9](../005-modules/503-static.md#539-initializers-compile-time-or-first-use)) |
| `send` / `try_send` / `send_async` of a value on a channel | the `recv` that receives that value ([stdlib 2.17](../../refs-standard-libraries/200-core/217-channel.md)) |
| `thread::spawn` / `Builder::spawn` | the first step of the new thread |
| the last step of a thread | the `join` (or `await` of its `JoinHandle`) that returns its result ([stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)) |
| `task::spawn` / `spawn_blocking` | the first step of the task |
| the last step of a task | the `await` of its `Task` that returns the result ([stdlib 2.18](../../refs-standard-libraries/200-core/218-task.md)) |
| the start of a `par` call or `par::scope` | the first step of each of its jobs |
| the last step of each `par` job | the return of the `par` call, or the end of the scope ([stdlib 2.19](../../refs-standard-libraries/200-core/219-par.md)) |

The locks, `Once` and the channels are built on `SeqCst` atomics in the reference library, so the orders above are also part of the `SeqCst` total order. `Atomic<i32>::wait` / `wake` only block and wake: they synchronize nothing by themselves, and the value that `wait` compares is read with the atomic's own ordering.

The collector, its write barrier ([7.4.2](704-barriers-and-roots.md#742-write-barrier-object-remembering)) and the stop-the-world pauses synchronize with the collector only. They give user code no happens-before edges, even where they happen to fence the hardware.

## 7.9.4. Data races

Two accesses to the same memory location **conflict** when at least one is a write and they are not both atomic. A program has a **data race** when two conflicting accesses happen in different threads and neither happens before the other. A data race is undefined behavior ([4.15.8](../004-functions/415-unsafe.md#4158-undefined-behavior)).

A data race can only happen through code that `unsafe` lets in. Safe Nabi has none, for these reasons:

- A `static` is reachable from every thread, so its type must be `Sync` ([5.3.1](../005-modules/503-static.md#531-basic-form)); `static mut` exists only in `extern "C"` blocks, and touching one needs `unsafe`.
- Collections (`List`, `Dict`, `StrBuf`, ...) are `!Sync`, and so are `Cell<T>`, raw pointers and guards ([stdlib 2.13.9](../../refs-standard-libraries/200-core/213-marker.md#2139-send--sync---thread-boundary-markers)). Shared mutable state crosses threads only inside `Mutex`, `RwLock`, `Atomic` or a channel. A frozen collection (`xs.freeze()`, [stdlib 2.9.7b](../../refs-standard-libraries/200-core/209-collections.md#297b-frozenlistt--frozendictk-v-s--frozensett-s---read-only-collections)) is `Sync` because nothing writes it: its storage is reached by no changeable handle.
- Handing a value to another owner (`Mutex::new`, `send`, `task::spawn`, `[move x]`, a thread's result, ...) gives the new owner its own copy of every collection the value reaches, and using the sender's value afterwards is a compile error (2.13.9). A plain capture of a spawned closure must be `Send + Sync`.
- `par` checks at run time that two items do not reach one collection that is not `Sync`, and runs such items one at a time ([stdlib 2.19.1](../../refs-standard-libraries/200-core/219-par.md#2191-core-api)). Items may write one list through views (`chunks_mut`) only where no other item's view or slice overlaps the part they write.

What makes a data race possible, and so needs the author's care:

- reads and writes through `Ptr<T>`, `ManyPtr<T>`, `CPtr<T>` and `GcBuf<T>` ([stdlib 1.1](../../refs-standard-libraries/100-bare/101-ptr.md), [stdlib 1.3](../../refs-standard-libraries/100-bare/103-mem.md));
- an `unsafe impl Send` or `unsafe impl Sync` ([4.15.4](../004-functions/415-unsafe.md#4154-in-traits)) on a type whose sharing it does not actually protect;
- `Mutex::new_unshared` on a value that something else still reaches ([stdlib 2.16.3](../../refs-standard-libraries/200-core/216-sync.md#2163-mutext---exclusive-lock));
- the hand-off checks skipped inside `unsafe` (2.13.9);
- an `extern static mut` and foreign code reached through FFI ([5.9](../005-modules/509-extern-block.md)), including C code that calls `getenv` / `setenv` while `env::set_var` runs ([stdlib 3.5](../../refs-standard-libraries/300-std/305-env.md)).

## 7.9.5. GC references

A GC reference is one aligned machine word, read and written with one access. The collector relies on that: it never meets a half-written reference, and it treats every reference word it scans as an object or null. This does not make a racing reference access defined. A data race on a reference is undefined behavior like any other, and a `str` (pointer and length, two words) or a struct can tear when raced.
