# 7.5. Concurrent mark

A full collection may run its mark phase concurrently with mutators, and its block sweep after it. Minor collection, explicit `nabi_gc_collect`, and evacuation are always stop-the-world ([7.3](703-gc-algorithm.md)).

> [!NOTE]
> This section is informative: it describes the reference runtime's GC, which may change in any release. Only the object layout fixed as ABI in [8.1](../008-abi/801-object-header-abi.md) is normative.

## 7.5.1. When

| `NABIR_GC_CONCURRENT` | Full collection                                    |
|----------------------|----------------------------------------------------|
| `auto` (default)    | Concurrent mark if old objects total 32 MiB or more, or a pause goal is set ([bare 1.6.12](../../refs-standard-libraries/100-bare/106-gc-pin.md#1612-set_pause_goal---a-pause-goal)) |
| `always`            | Always concurrent mark                             |
| `off`               | Always STW                                         |

An unset or empty variable has its default. A value outside those listed prints one warning line on stderr naming the variable, the value and the accepted values, and the variable has its default.

## 7.5.2. Phases

1. **Initial pause (STW).** Run one minor collection and take a snapshot of the roots at a safepoint. Set the global `MARKING` flag.
2. **Concurrent mark.** A single marker thread marks starting from the snapshot. Mutators keep running; one whose allocation outruns the marker marks too (7.5.4).
3. **Final pause (STW).** Drain the SATB buffers, rescan the roots, and sweep the large objects. Clear `MARKING`.
4. **Lazy sweep.** The blocks are swept after the final pause while mutators run: by the marker thread, by an allocating thread that finds no free block (it sweeps a few blocks instead of growing the heap), and by the next pause for whatever is left, which every pause finishes first. A block is handed out again as soon as it is swept; free blocks beyond the next cycle's budget go back to the OS first.

## 7.5.3. SATB barrier

Snapshot-at-the-beginning (Yuasa), per object: before an object's first store during mark, the references it held at that moment are logged, so everything reachable when mark began stays marked. It needs no barrier of its own. The generational write barrier ([7.4.2](704-barriers-and-roots.md#742-write-barrier-object-remembering)) already takes its slow path on the first store into an object, and that slow path does the logging. The generational barrier remembers the object stored into; it is not an incremental-update barrier, which would record the stored value instead.

- The slow path on the first write to an `UNLOGGED` object, when `MARKING` is set, records all of that object's current references in the SATB buffer (TLS `nabi_tls_satb_cursor` / `nabi_tls_satb_limit`, [8.3](../008-abi/803-tls-slots.md)).
- While recording, it sets the header's `SATB` bit. A competing store to the same object waits until that bit is cleared.
- A large (LOS) object is logged one 512 B card at a time: the first store into each card records the references that card holds (the slow path receives the slot and the stored size), and two bits per card, after the card table, say whether a card is being or has been recorded. A store waits for at most one card, never for the whole object; the object stays `UNLOGGED` for the rest of the mark.
- So the barrier code the compiler emits is the same whether or not concurrent mark is in use.

## 7.5.4. Allocation and collection

- Allocation during mark is black at line granularity (objects in that line are treated as live). Objects allocated during mark are young after mark finishes.
- No minor collection happens during mark. A mark has twice the young budget to allocate in; past it, the allocating thread marks a share of what is left itself (a mark assist, 4096 objects) and earns more room in proportion: the young budget over the objects the last mark marked, per object marked. Only past four times the budget, or at the heap limit ([bare 1.6.14](../../refs-standard-libraries/100-bare/106-gc-pin.md#1614-set_heap_limit---a-heap-limit)), does allocation finish the mark in a pause. When an assist finds nothing left to mark, it finishes the mark (a short final pause).
- Evacuation happens only in STW full collections. If the lazy sweep finds the heap fragmented, the next full collection is done as STW with evacuation (not with a pause goal). The lines allocated during the mark count as occupied in that test ([7.3.1](703-gc-algorithm.md#731-immix-mark-region)), so allocation alone never makes the next full collection stop the world.
- While a `no_collect` region is open ([bare 1.6.13](../../refs-standard-libraries/100-bare/106-gc-pin.md#1613-no_collect---a-region-without-collections)), the marker keeps marking but waits with the final pause until the region closes.
