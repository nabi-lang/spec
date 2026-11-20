# 8.4. Safepoint protocol

For the [GC](../000-meta/007-glossary.md#gc-garbage-collector) to perform temporary STW (Stop-The-World) work, every mutator thread must have reached a safe point. This document defines the [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) protocol for that synchronization.

## 8.4.1. `yield` flag check: fast path

Mutators periodically check the [TLS slot](803-tls-slots.md) `nabi_tls_yield_flag` (u8). The check proceeds as follows.

```
poll:
    flag = TLS[yield_flag]
    if flag != 0:
        call nabi_runtime_yield()    // slow path
    // proceed normally
```

In the emitted code:

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

This is a two-instruction fast path plus one branch, and the branch almost always evaluates to `false`.

## 8.4.2. Emission sites

The compiler emits safepoint polls at the following sites.

| Site                  | Reason                    |
|-----------------------|---------------------------|
| Function entry (or before the first safepoint of each path) | Can cooperate with the GC right after a call |
| Loop back-edge        | Prevents an infinite loop from blocking the GC |
| Right after a function call (optional) | Cooperate immediately if the call was long |

The detailed emission policy is up to the compiler; this specification guarantees only the following minimum frequency.

- Every loop back-edge has a poll, except a loop with a compile-time constant trip count of at most 64 whose body reaches no safepoint; every cycle of calls contains a poll.
- A function that can reach no safepoint (no loop but such bounded loops, no allocation, indirect call, lazy `static`, or call to a function that may reach one) needs none.

Together these bound the time from a yield request to the next safepoint.

## 8.4.3. `nabi_runtime_yield`: slow path

The slow path signature is:

```
extern "C" {
    fn nabi_runtime_yield();
}
```

A mutator calls this function to cooperate with the GC. Its effects are:

1. Sets `mutator_state` in the mutator's [Mutator TLS slot layout](803-tls-slots.md) to `GC_COOPERATING`.
2. Exposes the root information the GC needs, such as the [shadow stack](../000-meta/007-glossary.md#shadow-stack) walk.
3. Waits until the GC finishes.
4. When the GC finishes, sets `mutator_state` back to `RUNNING`.

`yield_flag` is cleared automatically while inside `nabi_runtime_yield`, or is cleared by the GC.

## 8.4.4. Setting / clearing the `yield` flag

When the GC needs an STW pause:

```
gc_initiate_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 1

  // wait until every mutator yields
  for each mutator thread:
    wait until mutator_state == GC_COOPERATING
```

When the GC finishes STW:

```
gc_finish_stw():
  for each mutator thread:
    atomic store TLS[yield_flag] = 0
  notify all (resume)
```

## 8.4.5. Fast path cost

The cost borne by a mutator is a single load of the `yield` flag and one branch.

In almost all cases the flag is 0, so the branch is not taken, and the branch predictor of modern CPUs handles it at nearly zero cost.

For that cost, every thread gets a deterministic cooperative synchronization mechanism.

## 8.4.6. STW phases

STW collection (minor, full with evacuation, `nabi_gc_collect`) is a single pause. The thread that started the collection stops all other mutators (`GC_COOPERATING`, or `PARKED` if inside a blocking runtime call), finishes the collection and resumes them.

A full collection with concurrent mark has two pauses, an initial and a final one ([7.5.2](../007-runtime/705-concurrent-collection.md#752-phases)). In between, a single marker thread runs alongside the mutators.

## 8.4.7. Safepoint invariant

When a safepoint is reached, every live GC object variable of the mutator must be registered in the shadow stack.

The compiler guarantees this by linking the shadow stack frame from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md) before the first safepoint on each path through the function: at entry, or later on a path that reaches one. A path that reaches no safepoint need not link it, and a frame is unlinked on return only where it was linked. Likewise, a function entered only by direct Nabi calls may move its entry poll to just after the link: every cycle of calls still contains a poll, since the call that closes the cycle is a safepoint. A function called from C or through a function value polls at entry. A local holding GC references at a safepoint is registered in the frame, and the registered word is kept current at every write. A local only back-edge polls see may instead be registered by the poll's slow path just for the runtime call ([7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler) rule 4), so a loop whose only safepoints are its polls links no frame and writes no root word while it runs. The compiler may also keep a private copy (in a register), since a collection never rewrites a root word: objects reached directly from roots are not moved ([7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)). Hot-reload code is the exception (8.4.13).

## 8.4.8. Loop back-edge poll

If a long loop does not cooperate with the GC, STW waits forever. The compiler therefore emits a poll on every loop back-edge (8.4.2's bounded loops aside).

```
loop:
    body
    poll                              // back-edge poll
    branch loop
```

The cost of this poll is again two instructions and one branch, negligible compared to the loop body.

The poll stays on the back edge. Its slow path (the `nabi_runtime_yield` call) may branch to a re-entry block placed before the loop header instead of returning into the edge.

The compiler detects every natural loop and emits a poll for each but the bounded loops of 8.4.2.

## 8.4.9. Inline functions and polls

When a small function is inlined, its entry poll may disappear with it. The inlined result is absorbed into the caller's body, however, so the caller's entry poll suffices.

Polls are placed after optimization, on the code that is compiled: a loop of an inlined body polls on its back edge in the caller like any other loop.

## 8.4.10. Native calls

A thread inside C or a blocking runtime call cannot reach a safepoint, so it counts as `PARKED`: the GC treats it as already cooperating.

- The compiler brackets every `extern "C"` call: `PARKED` for the duration of the call, `RUNNING` after it, each store followed by a yield-flag check. The caller's argument roots stay in its shadow-stack frame. The fast path is inline; `nabi_enter_native` / `nabi_leave_native` are the slow paths ([8.5.2](805-extern-symbols.md#852-runtime-area)). A call to a `#[leaf]` declaration has no bracket: the thread stays `RUNNING`, and holds back every collection until the call returns ([stdlib 1.7.1](../../refs-standard-libraries/100-bare/107-ffi.md#leaf---short-calls)).
- Every `extern "C" fn` definition starts with `nabi_native_reenter` and ends with `nabi_native_resume`, so a C callback runs as a mutator.
- The runtime brackets its own blocking calls (I/O wait, `nabi_sync_wait`, join, sleep) the same way, and its long computations that touch no GC object but buffers the caller's frame keeps alive and store no reference: the `int` kernels (multiplication, division, gcd; addition, subtraction, shifts and single-limb division from 64 Ki limbs), and `str` searches and counts, UTF-8 checks and byte copies from 64 KiB ([8.5.2](805-extern-symbols.md#functions-that-are-not-safepoints)). Their `extern "nabi"` declarations are not `#[no_safepoint]` ([5.9.2](../005-modules/509-extern-block.md#592-abi-specifiers)), so a call to them is a safepoint and the buffers are rooted.
- A call to a `#[no_safepoint]` runtime function is not bracketed. Every other thread's collection waits until it returns, so such a function does a bounded amount of work; a long input goes to its parked variant. A debug build of nabic wraps it in `nabi_nosp_enter()` / `nabi_nosp_leave()` ([8.5.2](805-extern-symbols.md#852-runtime-area)), which raise and lower the thread's `no_safepoint` depth ([8.3.1](803-tls-slots.md#831-tls-slot-structure)); a debug runtime fails with a fatal error if the thread would stop for a collection while the depth is nonzero. A release nabic emits neither call.
- A call through an `extern "C"` declaration of a `nabi_*` symbol ([8.5](805-extern-symbols.md), the runtime's reserved prefix) is not bracketed: the runtime function runs as a mutator.

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

## 8.4.11. Direct calls from user code

User code may also (rarely) call `nabi_runtime_yield()` directly.

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

`nabi_runtime_yield` is a `nabi_*` symbol, so the call is not bracketed (8.4.10) and the thread yields as a running mutator. The compiler already emits polls on loop back-edges, so users almost never need to call it directly. This pattern is used only in special situations such as inside inline assembly.

## 8.4.12. Stability

The ABI is the slot at offset 10p+8 of `nabi_tls` (`nabi_tls_yield_flag`) and its type (u8) ([8.3.1](803-tls-slots.md#831-tls-slot-structure)).

`nabi_runtime_yield` follows a simple calling convention with no arguments and no return value.

## 8.4.13. Hot-reload code

Code built with `--hot` ([9.1](../009-tools/901-nabic-cli.md)) runs under a hot runtime that may rewrite root words: a state migration copies objects of a changed type into new ones and updates every reference to them, roots included. Such code therefore keeps no private copy of a GC reference across a safepoint: every reference live across one is read back from its root word after it.

- Hot builds are `-O0` builds. There every local lives in its frame slot, which is the word the shadow-stack frame registers, so each use reads the root word.
- A call to a function of a hot package counts as a safepoint, whatever its current body does: the slot it goes through may reach a later version that stops. The caller roots what is live across it.
- A function that holds a value of a type of a hot package links its shadow-stack frame in the descriptor form ([7.4.1](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)), also when it roots nothing, and the word before its descriptor holds the address of its frame record (the types it holds), which one of its root words holds too. The hot runtime finds the stopped frames of old code there.

Frames of code built without `--hot` (the host program, C) keep the rule of 8.4.7; a migration does not rewrite the roots of such a frame.
