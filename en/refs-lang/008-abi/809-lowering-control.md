# 8.9. Lowering: control flow and GC integration

This section covers the lowering rules for [`if` expressions](../003-expressions/310-if-expression.md), [`match`](../003-expressions/312-match-expression.md), [loops](../003-expressions/313-loop-expressions.md) and [error control flow](../003-expressions/314-error-control-flow.md), and also defines the rules for emitting [GC](../000-meta/007-glossary.md#gc-garbage-collector) alloc / barrier / safepoint code.

nabi-mir is a control-flow graph: basic blocks with block arguments, each ending in a goto, a two-way branch on a `bool`, a return or `unreachable`. The code generator emits each as a jump, a conditional branch, a return or a trap; a block argument is a store to the target block's slot before the jump. The optimizer works on this graph above `-O0` ([8.7.3](807-code-generation.md#873-optimization-levels)). Code is pseudo-code ([8.7.1](807-code-generation.md#871-translation-flow)).

## 8.9.1. `if` expressions

```nabi
if cond { x } else { y }
```

This expression is lowered to a two-way branch; the value arrives as the argument of the join block.

```
bb0:
    if cond then bb_then else bb_else
bb_then:
    goto bb_end(x)
bb_else:
    goto bb_end(y)
bb_end(result):
```

An `if` with no `else` branch and a unit value branches straight to the join block:

```nabi
if cond { do_thing(); }
```

```
bb0:
    if cond then bb_then else bb_end
bb_then:
    call do_thing(tls)
    goto bb_end
bb_end:
```

## 8.9.2. `if let`

Consider the `if let pattern = expr` form from [`if` expressions](../003-expressions/310-if-expression.md).

```nabi
if let v: T = compute() { process(v) } else { fallback() }
```

The pattern's test is lowered as in a `match` (8.9.3):

```
bb0:
    opt = call compute(tls)            ; (i32 tag, payload)
    if opt.0 != TAG_NONE then bb_then else bb_else
bb_then:
    v = opt.1 as T
    call process(v, tls)
    goto bb_end
bb_else:
    call fallback(tls)
    goto bb_end
```

## 8.9.3. `match` decision tree

The arms of a [`match`](../003-expressions/312-match-expression.md) are tested in source order, each pattern a sequence of two-way branches.

```nabi
match v {
    1 => a,
    2 => b,
    n if n > 100 => c,
    _ => d,
}
```

This `match` expression becomes the following.

```
bb0:
    if v == 1 then bb_a else bb_test2
bb_test2:
    if v == 2 then bb_b else bb_guard
bb_guard:
    n = v
    if n > 100 then bb_c else bb_d

bb_a:
    goto bb_end(a)
bb_b:
    goto bb_end(b)
bb_c:
    goto bb_end(c)
bb_d:
    goto bb_end(d)

bb_end(result):
    ...
```

Tuples, `struct`s, `enum` variants ([8.8.6](808-lowering-types.md#886-representation-of-enum)) and sum members test their parts the same way; a range pattern is two comparisons, a string pattern a `str` equality ([8.7.6](807-code-generation.md#876-extern-functions-runtime-functions)). A match is exhaustive, so the block after its last test is unreachable (a trap).

No jump table is built: the chain is compiled as written at every level, one comparison and branch per tested arm. The optimizer removes a test it can decide from what earlier branches proved.

## 8.9.4. `loop` / `while` / `for`

### `loop`

```nabi
let v = loop { ...; if cond { break x; } }
```

A loop is a header block and a back-edge jump to it; `break x` branches to the exit block with `x` as its argument.

```
    goto bb_head
bb_head:
    ; body
    if cond then bb_exit(x) else bb_latch
bb_latch:
    ; safepoint poll (8.9.11)
    goto bb_head
bb_exit(v):
```

### `while`

```nabi
while cond { body }
```

The condition is tested in the header:

```
    goto bb_head
bb_head:
    c = ...
    if c then bb_body else bb_exit
bb_body:
    ; body, then the back-edge poll
    goto bb_head
bb_exit:
```

### `for`

```nabi
for x in xs { body }
```

The desugaring from [Loop expressions](../003-expressions/313-loop-expressions.md) expands it to `let it = xs.into_iter(); loop { match it.next() { ... } }`, which then goes through the same lowering described above. A `for` over an integer range (`for i in a..b`, bounds evaluated once) or an array is lowered directly to a counting `while` loop that steps the index before the body, so `continue` cannot skip it.

## 8.9.5. `break` / `continue`

`break` and `continue` are jumps to the loop's exit block or to its back-edge block. Labels (`'outer:`) likewise jump to the appropriate [block](../000-meta/007-glossary.md#block).

## 8.9.6. `try` / `catch` / `finally`: value-based branching

Errors from [error control flow](../003-expressions/314-error-control-flow.md) are propagated through the `{ is_err, payload }` return value defined in [Calling convention](806-calling-convention.md#865-return-of-throwing-functions-value-based-error-propagation). `try` and `catch` are therefore both lowered to ordinary branches.

```nabi
let v = try f(x);              // f: fn(X) throws (E) -> T
```

A call marked with `try` checks `is_err` of the return value: on success it extracts the payload as `T`; on error it branches to the error path.

```
bb0:
    r = call f(x, tls)                 ; (is_err, payload)
    if r.0 then bb_err else bb_ok

bb_ok:
    v = r.1 as T
    ...

bb_err:
    e = r.1 as E
    ; if there is an enclosing catch, go to its dispatch block;
    ; otherwise build the enclosing function's return value { 1, e } and return it
    goto bb_catch_dispatch(e)
```

An expression with `catch` clauses has a single dispatch block where all error paths of the left-hand expression converge.

```nabi
try body catch e: A { handler_a } catch _ { handler_any }
```

```
bb_catch_dispatch(e):
    ; test patterns in clause order (same scheme as the decision tree of 8.9.3)
    if <e is A> then bb_handler_a(e) else bb_handler_any(e)

bb_handler_a(ea):
    ; handler_a
    goto bb_end(result_a)

bb_handler_any(ex):
    ; handler_any
    goto bb_end(result_any)

bb_end(result):
    ...
```

If no clause matches, the dispatch block passes the error to the dispatch block of the outer `catch`, or builds the enclosing function's error return value and returns it. Type pattern tests on errors of untyped `throws` (`any Error`) are lowered to tag comparisons of [`any`](808-lowering-types.md).

`try?` produces `none` on the error path and goes to `bb_end`; `try!` calls `panic_fn` on the error path.

## 8.9.7. `throw`

`throw` is used as follows.

```nabi
throw err;
```

`throw` is lowered to an unconditional branch to the error path. If the nearest `catch` is in the same function, it goes to that dispatch block; otherwise it builds the enclosing function's error return value and returns it.

```
; when a catch exists in the same function
goto bb_catch_dispatch(err)

; otherwise, error return from the function
r.0 = true                ; is_err
r.1 = err                 ; the payload as E
return r
```

The path returning an error out of the function is the same as an ordinary `return`, so epilogues such as the shadow stack pop run as usual. There is no runtime call.

## 8.9.8. `finally`

`finally` is used as follows.

```nabi
try { body } finally { cleanup }
```

It is lowered so that every path leaving the left-hand expression passes through the cleanup block. Normal completion, completion after `catch` handling, propagation of an unhandled error, and each `return` / `break` / `continue` path branch to the cleanup block and then continue to their original destination. Error propagation is also an ordinary branch, so no separate unwind mechanism is needed.

```
bb_body_ok(v):        goto bb_cleanup(K_OK, v)
bb_body_err(e):       goto bb_cleanup(K_ERR, e)
bb_body_return(rv):   goto bb_cleanup(K_RET, rv)

bb_cleanup(kind, val):
    ; cleanup
    ; depending on kind, branch to bb_end / error propagation / function return / loop destination
```

If a new error is thrown inside the cleanup block, it replaces the original path. A panic terminates the process, so it does not pass through the cleanup block.

## 8.9.9. GC alloc: fast / slow path

Consider [Allocation](../007-runtime/702-allocation.md) together with this. A `struct` is a value and is never allocated (8.8.3); heap objects are closure environments, boxes (`any`), `List` / `Dict` buffers and strings built at run time.

```nabi
let n = 10;
let f = |x: i32| x + n;    // env: header + an i32
```

The environment is allocated inline, at the allocation site:

```
cur = load.ptr [t + p]                 ; tlab_cursor
lim = load.ptr [t + 2p]                ; tlab_limit
if 24 <= lim - cur then bb_fast else bb_slow
bb_fast:
    store.ptr [t + p], cur + 24        ; bump the cursor
    store.u32 [cur + 0], TYPE_ID_ENV   ; type_id
    store.u64 [cur + 8], 24            ; gc_word
    env = cur
    goto bb_done
bb_slow:                               ; placed out of line
    env = call nabi_gc_alloc_slow(24, 8, TYPE_ID_ENV)
    goto bb_done
bb_done:
```

`t` is the thread's TLS base ([8.7.7](807-code-generation.md#877-tls-access)), `24` the object size (rounded up to 8), `8` the alignment, `TYPE_ID_ENV` the type_id. The fast path bumps the TLAB cursor when the object fits and writes the header's `type_id` and `gc_word` (= size, [8.1.4](801-object-header-abi.md#814-gc_word-offset-8-u64-or-ptr)); the header's address is the object. Otherwise it calls `nabi_gc_alloc_slow`. `gc_flags` is not written: TLAB memory is zeroed. Objects of the large-object size and above, or aligned past 8, call `nabi_gc_alloc_slow` directly.

The fields are then stored after the 16-byte header:

```
store.i32 [env + 16], n
```

## 8.9.10. Write barrier

The object-remembering barrier from [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md#742-write-barrier-object-remembering).

```nabi
*p.field = q;       // write a GC pointer into a GC object
```

This is lowered to the following.

```
; 1. write barrier: checks only the header of the destination p (may precede the store)
flags = load.u32 [p + 4]               ; gc_flags
if flags & UNLOGGED != 0 then bb_log else bb_store

bb_log:                                ; placed out of line
    call nabi_gc_wb_slow(p, slot, len) ; atomically clear UNLOGGED and add to the remembered set
    goto bb_store

bb_store:
    ; 2. ordinary store
    store.ptr [p + offset_of(field)], q
```

- The header of the stored value `q` is not read. `q` may be an inline integer of an `any` slot or an interior pointer of a `str`, so reading it as a header is unsafe.
- The fast path is one load and one branch. In practice it checks `UNLOGGED | CARDED` together. If `CARDED` (a large old object), it marks dirty, without a call, the 512 B cards spanned by the written slot (`(slot - p) >> 9` ..= `(slot + len - 1 - p) >> 9`) in the card table (whose pointer is the word immediately before the header); if `UNLOGGED`, it calls `nabi_gc_wb_slow` as above. The barrier therefore uses the slot address and the size of the stored value ([7.4.2](../007-runtime/704-barriers-and-roots.md#742-write-barrier-object-remembering)).
- `nabi_gc_wb_slow` is not a safepoint. No collection intervenes between the barrier and the store.
- No barrier is inserted for initialization of a just-allocated object, stores into a `static`, raw `Ptr<T>` memory, or non-reference atomics.

## 8.9.11. Safepoint poll

Consider also the [Safepoint protocol](804-safepoint.md).

```
; safepoint poll (t: the TLS base, 8.7.7)
flag = load.u8 [t + 10p+8]             ; yield_flag
if flag != 0 then bb_slow else bb_cont

bb_slow:                               ; placed out of line
    call nabi_runtime_yield()
    goto bb_cont

bb_cont:
    ; ... continue ...
```

The compiler emits this code on loop back-edges, and in the functions a call cycle can re-enter without passing a loop poll (members of a recursive cycle, functions called through a pointer or from C, exported functions): at entry for those entered other than by a direct Nabi call, before the first safepoint of each path for the others ([8.4.7](804-safepoint.md#847-safepoint-invariant)). Other functions do not poll at entry: their callers do.

## 8.9.12. Shadow stack frame

See also [Barriers and root tracking](../007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler). The compiler emits the link / unlink of frames directly, with no help from the platform.

Example of a function with 2 GC reference words (`x`, `y`, stack slots). The frame holds the address of each slot, so a variable assignment is just an ordinary store into `x`.

```
; entry: zero-initialize the words, link the frame
; frame: a stack slot (prev: ptr, count: usize, slots: [ptr; 2])
store.ptr [x], null
store.ptr [y], null
store.ptr [frame + 0], load.ptr [t + 0]    ; prev = the shadow stack head (offset 0 of the TLS block)
store.usize [frame + p], 2                 ; count
store.ptr [frame + 2p], &x                 ; the address, not the value
store.ptr [frame + 3p], &y
store.ptr [t + 0], &frame                  ; link

; variable assignment: no frame update
store.ptr [x], obj

; immediately before every return (including error returns): unlink
store.ptr [t + 0], load.ptr [frame + 0]
```

Above `-O0` the slots list the words more briefly, once the frame is laid out. A run of `n` adjacent words (three or more) is the entry `(n << 1) | 1` followed by the first word's address (an address is even, so the low bit tells the two apart). A list of two entries or more moves to read-only data: the frame holds `count` 0 and one slot, the address of a static descriptor `{count: u16, entries: [u16; count]}`, whose entries are those of the slots with each address replaced by twice the word's offset from the frame in words (a signed 16-bit value, even like an address). Entry then stores two words instead of the whole list. A list whose offsets do not fit 16 bits stays in the frame. A descriptor is named by its contents and the link keeps one copy of each.

```
; the same frame above -O0, its list in read-only data
; desc: u16 2, (x - frame) / p * 2, (y - frame) / p * 2
store.ptr [frame + 0], load.ptr [t + 0]
store.usize [frame + p], 0                 ; a descriptor follows
store.ptr [frame + 2p], &desc
store.ptr [t + 0], &frame
```

`t` is the address of the thread-local block `nabi_tls` of [8.3.8](803-tls-slots.md#838-form-of-compiler-emission) ([8.7.7](807-code-generation.md#877-tls-access)); the shadow stack head is its slot at offset 0. An error return is also an ordinary `return` ([8.9.6](#896-try--catch--finally-value-based-branching)), so it passes through the same unlink. The link may also come later than shown, before the first safepoint of each path ([8.4.7](804-safepoint.md#847-safepoint-invariant)); then only the returns of paths that linked unlink. A function needs a frame only if it holds GC references and can reach a safepoint ([8.4.2](804-safepoint.md#842-emission-sites)). Calls to functions that never return (panics) and to runtime functions that neither allocate nor block ([8.5.2](805-extern-symbols.md#functions-that-are-not-safepoints)) do not count.

## 8.9.13. Indexing

Consider `arr[i]` from [Indexing and slicing](../003-expressions/306-indexing-and-slicing.md).

```nabi
arr[5]
```

This expression becomes a call to the standard library function `Index::index(arr, 5)` and is lowered like an ordinary function call.

With the `Index` call inlined (`-O1` and above), the result is the bounds check and one load:

```
if 5 >= arr.len then bb_panic else bb_ok
bb_ok:
    val = load.T [arr.data + 5 * size_of(T)]
```

This result falls out of generic monomorphization and inlining.

## 8.9.14. Summary: what the hot path looks like

The cost of the typical hot path, a [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) alloc plus field writes, is:

- TLAB cursor check: load \* 2 / sub / compare and branch
- cursor update: store (1 instruction)
- header fill: store \* 2 (type_id, gc_word)
- field fill: store \* N (number of fields)
- write barrier (if needed): load / and / compare and branch, + (slow) call
- safepoint poll (if needed): load / compare and branch

The slow paths are placed out of line. Allocating and filling a small object takes roughly 10 ~ 20 instructions; at `-O0` each value also goes through its stack slot, and above `-O0` the values that fit stay in registers.
