# 7.3. GC algorithm

Nabi's GC is a collector that layers sticky mark-bit generational collection[^whippet] on top of [Immix](../000-meta/007-glossary.md#immix) mark-region[^immix]. Minor collection and evacuation are stop-the-world, with mutators stopped at safepoints ([8.4](../008-abi/804-safepoint.md)). The mark phase of a full collection may run concurrently ([7.5](705-concurrent-collection.md)). Roots and the write barrier are defined in [7.4](704-barriers-and-roots.md).

> [!NOTE]
> This section is informative: it describes the reference runtime's GC, which may change in any release. Only the object layout fixed as ABI in [8.1](../008-abi/801-object-header-abi.md) is normative.

## 7.3.1. Immix mark-region

### Blocks and lines

```
block (32 KiB) = line (128 B) × 256
```

[Blocks](../000-meta/007-glossary.md#block) are obtained from the OS (mmap). Objects may cross line boundaries. Mark marks the object together with the lines it spans, and sweep reclaims unmarked lines. Small gaps between live objects are reused at line granularity.

### Sweep: block classification

| Class      | State                         | Handling                         |
|------------|-------------------------------|----------------------------------|
| clean      | All lines empty               | To the free pool                 |
| recyclable | Only some lines live          | Free lines reused for allocation |
| full       | Nearly all lines live         | Left as is                       |

Sweep does not zero dead objects, except secret buffers (`WIPE`, [7.1.2](701-object-layout.md#712-memory-spaces)). The allocator zeroes lines when it fills a TLAB.

### Opportunistic evacuation

A full collection may move live objects out of fragmented blocks into other blocks.

- Candidates are chosen from the line state left by the previous sweep (blocks at most half full). Lines handed out since count as occupied: their objects' liveness is not known yet. Objects are moved only when a fragmentation threshold is exceeded.
- A forwarding pointer is left at a moved object's old location, and all precise references are updated to the new location within the same collection.
- Objects not moved: objects pointed to directly by roots ([7.4.1](704-barriers-and-roots.md#objects-pointed-to-by-roots-are-not-moved)), objects pointed to by ambiguous words (`AMBIGUOUS`: type id `2` buffers, untagged `union`s), `PINNED` / `NOMOVE` objects, and LOS objects.
- Free blocks beyond the next cycle's allocation budget are returned to the OS.
- For testing, `NABIR_GC_EVAC=always` evacuates on every full collection ([9.9.8](../009-tools/909-debug-tools.md#998-memory--gc-tools)).

## 7.3.2. Sticky mark-bit generational collection

There is no separate nursery space; generations are distinguished by the header's `STICKY` bit ([8.1.3](../008-abi/801-object-header-abi.md#813-gc_flags-offset-4-u32)). An object that survives a collection gets `STICKY` set and becomes old. Objects stay in place; there is no promotion copy.

| Collection    | Mark starting points                             | Scope                                         |
|---------------|--------------------------------------------------|-----------------------------------------------|
| minor (young) | Roots, remembered set, dirty cards ([7.4.2](704-barriers-and-roots.md#742-write-barrier-object-remembering)) | Does not enter old objects. Sweep covers only the blocks handed out since the last collection (and, after a concurrent mark, those it handed out lines in) |
| full (old)    | Roots                                            | Clears all `STICKY` and does a full mark / sweep |

- Young lines are parsed only in blocks reached by an object lookup.
- A minor collection's bookkeeping follows what changed, not the heap: the lists of blocks with free lines stay from one collection to the next, and only the blocks handed out since are swept and listed again. Dirty cards are scanned a word (8 cards) at a time.
- A full collection happens when the bytes of old objects reach `max(4 MiB, live after the previous full collection × 2)`, or when `nabi_gc_collect` is called.

## 7.3.3. Mark procedure

1. Start from the roots ([7.4.1](704-barriers-and-roots.md#741-root-tracking-a-shadow-stack-emitted-directly-by-the-compiler)).
2. For each reached object, set `MARK` in the header, mark the lines it spans, and follow its children through the [ScanTable](../008-abi/802-scan-table.md) of its `type_id`.
3. Finish when the queue is empty.

In a stop-the-world pause the mark runs on several threads once the queue holds enough to share (`NABIR_GC_WORKERS` helper threads, [9.9.8](../009-tools/909-debug-tools.md#998-memory--gc-tools)): each marks from a stack of its own and passes work through a shared pool, `MARK` is set with an atomic `fetch_or` so each object is scanned once, and a block's young lines are parsed by one thread while the others wait. The concurrent mark ([7.5](705-concurrent-collection.md)) runs on its marker thread and the threads that assist it.

ScanTables are registered with `nabi_gc_register_type` ([8.5.1](../008-abi/805-extern-symbols.md#851-gc-area)).

## 7.3.4. Migration (hot reload)

Under `nabi dev` ([9.14](../009-tools/914-hot-reload.md)), a new version of a hot package may change the layout of a type. Its objects are rewritten in a migration: a full stop-the-world collection whose last phase, after the mark and sweep (and evacuation, if any), rewrites every live object of a changed type.

- Which objects: a hot build gives each box and buffer that holds a hot type by value its own type id, so the type id names the type and the version. The affected types are the changed ones and every type that holds one by value.
- How: kept fields are copied by name (or `#[renamed_from]`), added fields take the values their defaults made before the pause, removed fields are dropped, enum variants are matched by name ([9.14.3](../009-tools/914-hot-reload.md#9143-what-may-change)). An object whose new layout fits is rewritten in place; another is copied into a new object and forwarded as in evacuation, and every heap reference to it is fixed, interior references into buffers and weak cells included. Root words are never rewritten.
- A dry pass first: the migration does nothing, and the change waits, when a rooted, `PINNED` or `NOMOVE` object of a changed type must grow, an ambiguous word points into the middle of one that moves, an `any` holds a value of a changed type inline, a closure, coroutine or function value of old code holds or uses a changed type, a live enum value holds a variant the new version removed, or a no-collect region is open.
- A migration counts as a full collection in the GC statistics.

## References

[^immix]: Blackburn & McKinley, _Immix: A Mark-Region Garbage Collector with Space Efficiency, Fast Collection, and Mutator Performance_, PLDI 2008.

[^whippet]: Wingo, Whippet: a sub-toolkit for embeddable garbage collection. <https://wingolog.org/tags/whippet>
