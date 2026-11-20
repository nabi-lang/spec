# 1.6. `Pinned<T>` - GC pinning

The `bare::gc` module defines a mechanism for temporarily fixing an object on the GC heap so that it cannot move. When a raw `Ptr<T>` points to a GC object and the object moves, the pointer becomes invalid; this chapter covers how that conflict is resolved.

This is the most important chapter of Nabi's unsafe tier: it explains head-on how the conflict between pointers and the GC is reconciled.

## 1.6.1. Motivation - why pinning is needed

The Immix mark-region GC of [refs-lang 7.3 GC algorithm](../../refs-lang/007-runtime/703-gc-algorithm.md) performs **evacuation** (moving objects by copying). An object's address can therefore change freely between GC cycles.

```
  GC cycle start:    obj at 0x1000
  GC evacuation:     obj at 0x2000  (copied, then forwarded)
  GC cycle end:      obj at 0x2000
```

For managed code (`GcPtr<T>`), the GC traces roots and updates every ptr automatically. A raw `Ptr<T>`, however, is invisible to the GC, so it is not updated and becomes a **stale ptr**.

```nabi
use bare::gc;
use bare::ptr::Ptr;

let xs: List<i32> = [1, 2, 3];                 // GcPtr (updated automatically)
let raw: Ptr<i32> = Ptr::dangling();
using pin = gc::pin(xs) {
    raw = pin.as_data_ptr();                    // raw ptr, valid inside the pin
}                                               // unpinned here
gc::collect();                                  // xs may move
unsafe { raw.read() };                          // UB: raw is stale
```

The solution is **pinning**: explicitly telling the GC "do not move this object".

## 1.6.2. `Pinned<T>` guard

```nabi
struct Pinned<T> {
    obj: T;
    // internal: registered in the GC's pin set
}

#[sealed]
trait Pinnable {
    // the GC object holding the value's data; none when nothing can move
    #[internal("write `pin(x)` to keep a value in place")]
    fn gc_object(self) -> GcBuf<u8> | none;
}
impl<T> Pinnable for List<T> { /* the list's buffer; none when empty */ }
impl<T> Pinnable for Slice<T> { /* the buffer it views; none when empty */ }
impl<T, const N: usize> Pinnable for Array<T, N> { /* the element buffer; none when empty (2.9.14) */ }
impl Pinnable for str { /* none (2.15.2) */ }
impl Pinnable for StrBuf { /* the byte buffer; none while it has no room (2.15.2) */ }
impl<T> Pinnable for GcPtr<T> { /* the cell */ }

fn pin<T: Pinnable>(obj: T) -> Pinned<T>;
fn collect();                           // a full collection now; may move every unpinned object
fn stats() -> GcStats;                  // the collector's counters (1.6.10)
fn collect_young();                     // a minor collection now (1.6.11)
fn collect_young_within(ms: f64) -> bool;
fn set_pause_goal(ms: f64);             // 1.6.12
fn no_collect<R>(f: fn() -> R) -> R;    // 1.6.13
fn set_heap_limit(bytes: u64);          // 1.6.14
struct Arena<T>;                        // 1.6.15

impl<T> Pinned<T> {
    fn obj(self) -> T;
}

impl<T> Drop for Pinned<T> {
    fn drop(ref self) {
        // removed from the GC pin set
    }
}
```

It is a guard. While the `Pinned<T>` returned by `pin()` lives, the GC does not move the object. `Pinned<T>` is a `Drop` value, released once ([2.13.4](../200-core/213-marker.md#2134-drop---cleanup-on-release)): `Pinned::drop` runs and unpins the object where the guard's owner leaves scope, or at the end of the `using` block that binds it. Pins count: an object moves again only after as many unpins. `pin` pins the object `gc_object` names, and pins nothing when it returns `none`. `GcBuf<u8>` is the type-erased GC buffer of [1.3.9](103-mem.md#139-gcbuft-and-the-buf_-primitives). `Pinnable` is sealed and `gc_object` is internal to the toolchain ([6.2.16](../../refs-lang/006-meta-system/602-built-in-attributes.md#6216-internal-shared-by-the-toolchains-packages)): a program pins with `pin(x)` and does not call `gc_object`.

### Example - FFI call

```nabi
let buf: List<u8> = read_input();

using pin = bare::gc::pin(buf) {
    let raw: Ptr<u8> = pin.as_data_ptr();
    unsafe {
        ffi::process(raw, buf.len());          // FFI uses buf's data
    }
}     // ← automatic unpin at end of scope
```

The `using` block here follows the RAII form defined by [refs-lang 3.17 using expression](../../refs-lang/003-expressions/317-using-expression.md).

## 1.6.3. `GcPtr<T>` vs `Ptr<T>` - separation at the type level

```nabi
struct GcPtr<T> { /* compiler-managed; roots traced automatically */ }
struct Ptr<T>   { /* raw; no root tracking */ }
```

| Aspect     | `GcPtr<T>`                    | `Ptr<T>`                                         |
|------------|-------------------------------|--------------------------------------------------|
| GC root    | **Traced automatically**      | **Not traced**                                   |
| On move    | **Updated automatically**     | **Stale**                                        |
| Used in    | Ordinary code (`List`/`Dict`/...) | unsafe + FFI only                            |
| Created by | GC alloc (`List::new()` etc.) | `gc::pin().as_data_ptr()` or `alloc::alloc(layout)`   |
| Valid for  | As long as the object is reachable | Only within the pin guard's scope           |

Because the type system separates the two kinds, user code normally deals only with `GcPtr` (invisibly wrapped in `List<T>` etc.). `Ptr<T>` appears explicitly only after entering unsafe.

### Operations

```nabi
impl<T> GcPtr<T> {
    fn new(value: T) -> GcPtr<T>;           // a new GC cell holding value
    fn get(self) -> T;
    fn set(self, value: T);                 // every copy of the GcPtr sees it
    fn same(self, other: GcPtr<T>) -> bool; // identity: the same cell
}
impl<T> !Sync for GcPtr<T> {}
impl<T> Pinnable for GcPtr<T> {}
```

### Internal implementation - the form of GcPtr

```nabi
struct GcPtr<T> {
    // internal: usually a plain `*T`; the GC traces it externally via the root set
    // for a local variable, kept in a shadow stack frame slot emitted by the compiler
}
```

Nabi's GC traces the root set through a **per-thread shadow stack** emitted directly by the compiler ([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)).

## 1.6.4. GC-side mechanism of pin / unpin

```
GC pin set structure.
  - hash table: Ptr<Object> → pin count
  - during evacuation the GC _skips_ objects with pin count > 0

pin(obj):
  1. check the pin bit in obj's header (or register in the pin set)
  2. count++
  3. return Pinned<T>

drop(Pinned):
  1. count--
  2. if count == 0, clear the pin bit
```

This structure is effectively isomorphic to Go 1.21's `runtime.Pinner`. See [pkg.go.dev/runtime#Pinner](https://pkg.go.dev/runtime#Pinner) for details.

### overhead

The costs are:

- One pin is just an O(1) hash insert
- One unpin is likewise an O(1) hash remove
- A pin set check is added within a GC cycle, but its cost is small

In most cases this is not on a hot path, because FFI calls themselves are not that frequent.

## 1.6.5. When a pin is needed

Every `extern "C"` call parks the calling thread ([1.7.1](107-ffi.md#171-extern-c-block)): while C runs, the collector does not wait for the thread, so a collection may run, and move objects, during any C call, however short. A pointer into a GC object that C uses during the call is valid only when:

- the object is pinned for the whole call (`gc::pin`, 1.6.2), and for as long after it as C keeps the pointer; or
- the pointer is to the object a root names itself: a `str` argument, which the caller's frame roots until the call returns ([refs-lang 7.4.1](../../refs-lang/007-runtime/704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)), and `str::as_ptr` / `CStr::as_ptr` under the rule of [2.15.2](../200-core/215-string.md#2152-str---immutable-utf-8-string).

Every other pointer into a GC object, a list's buffer (`as_data_ptr`), a slice's elements, a `StrBuf`'s bytes, a `GcPtr`'s value, needs a pin. The compiler never leaves a pin out on its own, whatever the call: the program writes it. `#[leaf]` ([1.7.1](107-ffi.md#171-extern-c-block)) makes a call cheaper and does not change this rule. A value C keeps between calls goes in a `Handle` ([1.7.10](107-ffi.md#1710-handlet---nabi-values-c-holds)), which pins it until it is released.

Code that holds no raw pointer needs no pin:

| Place            | Reason                                  |
|------------------|-----------------------------------------|
| Ordinary method calls | GcPtr is updated automatically; no raw ptr is created |
| Operations between GcPtrs | All traced by the GC           |
| A `str` passed to an `extern "C"` function | The argument's root keeps the string in place for the call |

### A pinned `List` or `StrBuf` cannot grow

Growing a list or a `StrBuf` past its capacity replaces its buffer, and C may hold the address of the old one. While the buffer is pinned, growth (`reserve`, and through it `push`, `push_str`, `extend` and the like) panics instead:

```text
cannot grow a list while it is pinned (C may hold its address): call `xs.reserve(n)` before `gc::pin(xs)`
cannot grow a `StrBuf` while it is pinned (C may hold its address): call `b.reserve(n)` before `gc::pin(b)`
```

Room reserved before the pin can be filled while the pin lasts.

```nabi
use bare::gc::pin;

let ys: List<i64> = [1];
ys.reserve(7);                     // room for 8
let py = pin(ys);
for i in 0..7 { ys.push(i); }      // fits: the buffer stays
ys.push(9);                        // panics: the buffer would have to grow
```

## 1.6.6. Extraction methods

```nabi
impl<T> Pinned<List<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // start of the List's buffer
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl<T> Pinned<Slice<T>> {
    fn as_data_ptr(self) -> Ptr<T>;          // the slice's first element; dangling when empty
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl<T, const N: usize> Pinned<Array<T, N>> {
    fn as_data_ptr(self) -> Ptr<T>;          // the first element; steps are strides(), in elements
    fn as_data_many_ptr(self) -> ManyPtr<T>; // many ptr
}

impl<T> Pinned<GcPtr<T>> {
    fn as_value_ptr(self) -> Ptr<T>;         // the cell's value
}

impl Pinned<str> {
    fn as_bytes_ptr(self) -> ConstPtr<u8>;   // UTF-8 bytes
    fn as_c_ptr(self) -> ConstPtr<c_char>;   // null termination not guaranteed; convert separately
}

impl Pinned<StrBuf> {
    fn as_bytes_ptr(self) -> Ptr<u8>;        // the first byte (no NUL promised); dangling while there is no room
}
```

These container methods are the extraction API, valid for as long as the `Pinned` is held. (`str::as_ptr` and `CStr::as_ptr` give an address without a pin, under the narrower rule of [2.15.2](../200-core/215-string.md#2152-str---immutable-utf-8-string).) There is no generic `as_ptr`; `as_value_ptr` is named apart from `List`'s `as_data_ptr`. Each container gets dedicated extraction methods, so data can be taken out in a form matching each container's layout, such as the buffer start or a null-terminated form.

## 1.6.7. GC runtime linkage

Each item maps to GC runtime calls as follows.

| Item             | Runtime call                   |
|------------------|--------------------------------|
| `Pinned<T>`      | `nabi_gc_pin` / `nabi_gc_unpin`  |
| `pin<T>()`       | (same as above)                |
| `collect()`      | `nabi_gc_collect`               |
| `stats()`        | `nabi_gc_stats` (1.6.10)        |
| `collect_young()` / `collect_young_within(ms)` | `nabi_gc_collect_young` / `nabi_gc_collect_young_within` (1.6.11) |
| `set_pause_goal(ms)` | `nabi_gc_set_pause_goal` (1.6.12) |
| `no_collect(f)`  | `nabi_gc_no_collect` (1.6.13)   |
| `set_heap_limit(bytes)` | `nabi_gc_set_heap_limit` (1.6.14) |
| `Arena<T>`       | `nabi_gc_arena_epoch` (1.6.15)  |
| `GcPtr<T>`       | shadow stack frame slot (compiler-emitted) |
| `as_data_ptr` etc. | depends on each container's layout |
| `Weak<T>`        | `nabi_gc_weak_new` / `nabi_gc_weak_get` (1.6.9) |
| growth check of a pinned buffer (1.6.5) | `nabi_gc_pinned` |

## 1.6.8. Comparison with other languages

| Aspect           | Java JNI                       | .NET fixed             | Go 1.21 Pinner           | Nabi                       |
|------------------|--------------------------------|------------------------|--------------------------|---------------------------|
| pin API          | `Get*Critical`                 | `fixed (T* p = ...)`   | `runtime.Pinner`         | `bare::gc::pin(obj)`  |
| RAII guard       | explicit `Release`             | block scope            | `defer pin.Unpin()`      | `using` block (automatic drop) |
| Type separation  | **none** (all `jobject`)       | **none**               | **none** (all ptrs alike) | `GcPtr<T>` vs `Ptr<T>`   |
| Expiry check after pin | **none**                 | compiler (block scope) | **none** (manual)        | type system + scope       |
| Multiple pins    | overlapping OK                 | **restricted**         | overlapping OK           | overlapping OK (count)    |

Nabi diverges from other languages at two main points.

1. The type system enforces it. `GcPtr<T>` and `Ptr<T>` are distinct types and automatic conversion is rejected; the only way across is through a pin guard.
2. Release is based on the `using` block. `Pinned<T>` is dropped on leaving the `using` block, so an unpin is never missed. This is safer than Go's manual `defer` approach.

> This policy is the backbone of Nabi's pointer model: the point where the two domains, unsafe and the GC, meet safely.

## 1.6.9. `Weak<T>` - weak references

```nabi
// bare::gc
struct Weak<T> { /* opaque */ }

impl<T> Weak<T> {
    fn new(target: GcPtr<T>) -> Weak<T>;
    fn get(self) -> GcPtr<T> | none;        // none once the target is freed
}
impl<T> !Sync for Weak<T> {}
impl<T> !Send for Weak<T> {}    // a hand-off cannot copy what it names (2.13.9)
```

A weak reference does not keep its object alive. Once a collection frees the object, every `Weak` to it reads `none`; a collection that moves the object updates the `Weak`. This is the library interface of the `WEAK_REF` scan kind ([refs-lang 8.2.8](../../refs-lang/008-abi/802-scan-table.md#828-weak_ref)).

At run time a `Weak` is a GC cell of type id `3` ([refs-lang 8.2.12](../../refs-lang/008-abi/802-scan-table.md#8212-runtime-reserved-type-ids)) holding one `WEAK_REF` word, made by `nabi_gc_weak_new` and read by `nabi_gc_weak_get` ([refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-auxiliary-symbols-appendix-non-normative)).

## 1.6.10. `stats` - collector counters

```nabi
// bare::gc
struct GcStats {
    pub collections: u64;      // every kind
    pub minor: u64;            // minor collections
    pub full: u64;             // full collections that stopped the world throughout
    pub concurrent: u64;       // full collections marked concurrently (7.5)
    pub pause_total_ns: u64;   // time the program stood stopped, all pauses together
    pub pause_max_ns: u64;     // the longest pause
    pub live_bytes: u64;       // alive at the last collection, plus allocated since
    pub heap_bytes: u64;       // held from the operating system (blocks and large objects)
    pub moved_bytes: u64;      // moved by evacuation
    pub assists: u64;          // times an allocating thread marked for a concurrent mark (7.5.4)
    pub grown_bytes: u64;      // bytes no_collect regions grew the heap by (1.6.13)
    pub limit_collections: u64; // collections the heap limit brought forward (1.6.14)
    pub young_budget: u64;     // bytes allocated between two minor collections now
}

fn stats() -> GcStats;
```

`stats()` reads the counters since the program started, for metrics or a frame budget. It takes the collector's lock for a moment, so it is meant to be read now and then (an export, once per frame), not in a hot loop. A pause counts from the stop request, so it includes the time to safepoint ([refs-lang 9.9.8](../../refs-lang/009-tools/909-debug-tools.md#998-memory--gc-tools)).

`collect()` stops every thread for a whole full collection, longer than the collections the runtime starts by itself (a minor one, or a full one marked concurrently). To keep pauses short, leave collecting to the runtime, or call `collect_young` between frames (1.6.11).

## 1.6.11. `collect_young` - a minor collection on demand

```nabi
// bare::gc
fn collect_young();
fn collect_young_within(ms: f64) -> bool;
```

`collect_young()` runs a minor collection now: it looks at the objects allocated since the last collection, so it costs about what survived of them, not the size of the heap. Between two frames or two requests is the cheap moment to collect. While a concurrent mark runs ([refs-lang 7.5](../../refs-lang/007-runtime/705-concurrent-collection.md)) it does nothing: the mark finishes on its own, without a long pause.

`collect_young_within(ms)` runs it only when the last minor collections say it takes at most `ms` milliseconds (the time left of the frame), and says whether it ran. The estimate is the cost per young byte of recent minor collections times the young bytes now, plus a fixed part; with no minor collection seen yet, it runs.

## 1.6.12. `set_pause_goal` - a pause goal

```nabi
// bare::gc
fn set_pause_goal(ms: f64);   // 0: no goal (the default)
```

With a goal the collector slices its work to keep each pause near `ms` milliseconds:

- The young budget follows the goal: three quarters of what the recent minor collections say fits in it, from 1 MiB up to the default budget ([refs-lang 7.2.8](../../refs-lang/007-runtime/702-allocation.md#728-gc-trigger)). `GcStats::young_budget` shows the current value.
- Every full collection marks concurrently, whatever the heap size, and the heap is never evacuated on its own: only `collect()` compacts it.
- Sweeping after a concurrent mark and its mark assists already run outside pauses ([refs-lang 7.5.2](../../refs-lang/007-runtime/705-concurrent-collection.md#752-phases)).

It is a goal, not a promise: a pause can still be longer, for example while the live heap grows fast or right after a concurrent mark whose allocation was large.

## 1.6.13. `no_collect` - a region without collections

```nabi
// bare::gc
fn no_collect<R>(f: fn() -> R) -> R;
```

`no_collect(f)` calls `f`, and while it runs no allocation on any thread starts a collection: no minor, no full, no stress collection (`NABIR_GC_STRESS`). The region closes where `f` returns, however it leaves. Regions nest and may be open on several threads at once; collections resume when the last one closes.

- When the young budget runs out inside a region, the heap grows instead, at least 1 MiB at a time; `GcStats::grown_bytes` counts it. The collection comes after the region, on the next allocation that needs one.
- A concurrent mark that is running keeps marking during the region and finishes (its final pause) after it.
- The heap limit (1.6.14) still holds: an allocation that would pass it collects, inside a region too.
- `collect()` and `collect_young()` called inside the region still collect.

## 1.6.14. `set_heap_limit` - a heap limit

```nabi
// bare::gc
fn set_heap_limit(bytes: u64);   // 0: no limit (the default)
```

The budget of each cycle is capped at `bytes`: collections come sooner as the heap nears the limit, and a collection the limit brings forward is a full one (counted in `GcStats::limit_collections`). The limit counts the bytes of GC objects, the young bytes handed out included, not the process's whole memory. Live data beyond the limit is not an error: the cap is then the live bytes plus an eighth (at least 4 MiB), so the heap grows past the limit a little at a time and each step collects fully. A concurrent mark that would pass the limit finishes in a pause.

## 1.6.15. `Arena<T>` - values for one frame

```nabi
// bare::gc
struct Arena<T> { /* shared by copies */ }
struct ArenaRef<T> { /* plain data: an index and a generation */ }
impl<T> Copy for ArenaRef<T> {}

impl<T> Arena<T> {
    fn new() -> Arena<T>;
    fn with_capacity(n: usize) -> Arena<T>;
    fn alloc(self, value: T) -> ArenaRef<T>;   // append in place
    fn get(self, r: ArenaRef<T>) -> T;         // panics for a stale reference
    fn try_get(self, r: ArenaRef<T>) -> T | none;
    fn set(self, r: ArenaRef<T>, value: T);    // panics for a stale reference
    fn holds(self, r: ArenaRef<T>) -> bool;
    fn len(self) -> usize;
    fn is_empty(self) -> bool;
    fn capacity(self) -> usize;
    fn reset(self);                            // empty at once; every ArenaRef so far is stale
    fn values(self) -> List<T>;                // a new list, in allocation order
}
impl<T> !Sync for Arena<T> {}
```

An `Arena<T>` holds the values of one frame (or one request) in one buffer: `alloc` appends a value in place, so the collector neither allocates nor traces an object per value, and `reset` empties the arena at once while keeping its buffer for the next frame. Copies of an `Arena` share it.

Memory safety rests on two rules:

- The buffer is an ordinary GC object. What the values reference stays alive while they are in the arena, and nothing in the arena can dangle. (After `reset`, a value of a type that holds GC references is cleared; one of a type that holds none is overwritten by the next `alloc`s.)
- An `ArenaRef` is plain data, an index and a generation. Each `reset` takes a generation no arena of the program used before ([refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-auxiliary-symbols-appendix-non-normative)), so a reference from before a reset, or from another arena, fails the check in `get` and `set` and panics (`try_get` gives `none`, `holds` says `false`). It can be kept anywhere, in a GC object too: using it late is a defined panic, never a read of the next frame's value.

So the arena needs no lifetimes, no branded scope and no rule on what `T` may hold; the price is one integer compare per access.

```nabi
use bare::gc;
use bare::gc::Arena;

struct Particle { x: f64; y: f64; }

let particles: Arena<Particle> = Arena::with_capacity(4096);
loop {
    let p = particles.alloc(Particle { x: 0.0, y: 0.0 });
    update(particles, p);
    draw(particles);
    particles.reset();                         // the frame's particles are gone
    gc::collect_young_within(frame_left_ms());
}
```
