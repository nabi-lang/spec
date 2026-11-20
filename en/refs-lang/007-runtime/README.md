# 7. Runtime overview

This block defines the implementation specification of the Nabi runtime system ([GC](../000-meta/007-glossary.md#gc-garbage-collector), task scheduler, coroutines). The system works identically under both engines ([JIT](../000-meta/007-glossary.md#jit-just-in-time) / [AOT](../000-meta/007-glossary.md#aot-ahead-of-time)) of [Execution engines and the user surface](../000-meta/001-execution-modes.md).

## Contents

The sections of this chapter are:

- [7.1. Object layout and memory spaces](701-object-layout.md)
- [7.2. Allocation](702-allocation.md)
- [7.3. GC algorithm](703-gc-algorithm.md)
- [7.4. Barriers and root tracking](704-barriers-and-roots.md)
- [7.5. Concurrent mark](705-concurrent-collection.md)
- [7.6. Scheduler](706-scheduler.md)
- [7.7. I/O driver and timers](707-io-and-timer.md)
- [7.8. Coroutines](708-coroutine.md)
- [7.9. Memory model](709-memory-model.md)

## Key decisions

The decisions for the runtime's core components are summarized as:

```
GC          in-house Immix + Sticky Mark-Bit generations
Scheduler   M:N work-stealing
Coroutine   compiler-generated state machine (stackless)
```

All three are implemented in-house, with zero dependencies on external libraries.

## Normative status

The GC sections, 7.1 to 7.5, are informative: they describe how the reference runtime allocates and collects today, and a later runtime may use other algorithms, spaces or barriers. What is normative is the object layout fixed as ABI in [8.1](../008-abi/801-object-header-abi.md): a 16-byte header before every GC object's payload. Library code depends on it: `Pinned::as_data_ptr` and `as_value_ptr` ([stdlib 1.6.6](../../refs-standard-libraries/100-bare/106-gc-pin.md#166-extraction-methods)) compute a payload address as the object's address plus 16. The memory model of [7.9](709-memory-model.md) is normative too: it is what programs and every backend rely on.

## Zero-dependency principle

The runtime does not depend on external libraries. Specifically, it does not use:

- external GC libraries (Boehm GC, MMTk, etc.);
- external async runtimes;
- external coroutine libraries.

Platform system interfaces (`libc`, OS headers, etc.) do not count as dependencies, since they are an essential part of the OS interface.

## M:N model

The scheduler follows an M:N hybrid model.

```
OS Thread (worker) x N  ──── nabi task / coroutine x M
                                   (typically M >> N)
```

The default N is the number of CPU cores and can be adjusted by runtime configuration (`NABIR_WORKERS`, [7.6.8](706-scheduler.md#768-thread-per-core-option)). Each worker thread is both a scheduler worker and a mutator, and a stop-the-world collection runs on the thread that started it ([7.6.10](706-scheduler.md#7610-integrated-gc-worker-role)). The I/O driver is a single separate reactor thread ([7.7](707-io-and-timer.md)), and concurrent mark runs on a separate marker thread ([7.5](705-concurrent-collection.md)).

## Single .a output

The runtime is distributed as a single staticlib, `libnabir.a`. Building a user package links `libnabir.a`, producing an executable binary with no further external dependencies.

## Runtime error: stack overflow

When a thread's stack overflows, the runtime flushes stdout (if possible), prints the following to stderr, and aborts (SIGABRT). Unlike a panic, there is no unwinding and `finally` does not run.

```
thread '<name>' has overflowed its stack
fatal runtime error: stack overflow, aborting
```

`<name>` is the thread's name: the name given with `thread::Builder::name` ([stdlib 3.4](../../refs-standard-libraries/300-std/304-thread.md)), which is also set as the OS thread name. The main thread is `main`, and an unnamed thread is `<unnamed>`. This applies to the main thread, `thread::spawn` threads, and executor workers (including deep `await` chains). The detection mechanism (guard range and `sigaltstack`) and TLS slots are in [8.5.7](../008-abi/805-extern-symbols.md#stack-overflow). On wasm a module cannot catch the trap; the host reports it (the shipped JS host prints the message above).

## Interface with the compiler

The runtime communicates with compiler-generated code through a fixed [ABI](../000-meta/007-glossary.md#abi-application-binary-interface). The interface consists of three elements:

- The object layout (16-byte header + payload).
- `extern "C"` functions such as `nabi_gc_alloc_slow`, `nabi_runtime_yield`, `nabi_thread_spawn`.
- Coroutines are compiler-generated state machines ([7.8](708-coroutine.md)). For GC roots, the compiler emits shadow stack frames directly ([7.4.1](704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)).

This ABI is defined in detail in [8. Runtime ABI / lowering](../008-abi/README.md); this chapter covers only internal behavior.
