# 8.9. 降级：控制流与 GC 集成

本节说明 [`if` 表达式](../003-expressions/310-if-expression.md)、[`match`](../003-expressions/312-match-expression.md)、[循环](../003-expressions/313-loop-expressions.md)和[错误控制流](../003-expressions/314-error-control-flow.md)的降级规则，并定义生成 [GC](../000-meta/007-glossary.md#gc-garbage-collector) 分配 / 屏障 / 安全点代码的规则。

nabi-mir 是一张控制流图：由带块参数的基本块组成，每个块以 goto、基于 `bool` 的双路分支、返回或 `unreachable` 结束。代码生成器把它们分别生成为跳转、条件分支、返回或 trap；块参数是在跳转前向目标块的槽中进行的一次存储。在 `-O0` 以上，优化器在这张图上工作（[8.7.3](807-code-generation.md#873-优化级别)）。代码为伪代码（[8.7.1](807-code-generation.md#871-翻译流程)）。

## 8.9.1. `if` 表达式

```nabi
if cond { x } else { y }
```

这个表达式降级为一个双路分支；值作为汇合块的参数到达。

```
bb0:
    if cond then bb_then else bb_else
bb_then:
    goto bb_end(x)
bb_else:
    goto bb_end(y)
bb_end(result):
```

没有 `else` 分支、值为 unit 的 `if` 直接分支到汇合块：

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

考虑 [`if` 表达式](../003-expressions/310-if-expression.md)中的 `if let pattern = expr` 形式。

```nabi
if let v: T = compute() { process(v) } else { fallback() }
```

模式的测试按与 `match`（8.9.3）相同的方式降级：

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

## 8.9.3. `match` 决策树

[`match`](../003-expressions/312-match-expression.md) 的各分支按源代码顺序测试，每个模式是一串双路分支。

```nabi
match v {
    1 => a,
    2 => b,
    n if n > 100 => c,
    _ => d,
}
```

这个 `match` 表达式变为如下形式。

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

元组、`struct`、`enum` 变体（[8.8.6](808-lowering-types.md#886-enum-的表示)）和和类型成员以同样的方式测试其各部分；区间模式是两次比较，字符串模式是一次 `str` 相等比较（[8.7.6](807-code-generation.md#876-extern-函数运行时函数)）。match 是穷尽的，因此其最后一次测试之后的块是不可达的（一个 trap）。

不构建跳转表：在每个级别，这条链都按所写的样子编译，每个被测试的分支一次比较和分支。优化器会移除它能根据先前分支已证明的事实判定的测试。

## 8.9.4. `loop` / `while` / `for`

### `loop`

```nabi
let v = loop { ...; if cond { break x; } }
```

循环是一个头块加上一条跳回它的回边跳转；`break x` 以 `x` 为参数分支到出口块。

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

条件在头块中测试：

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

[循环表达式](../003-expressions/313-loop-expressions.md)中的脱糖把它展开为 `let it = xs.into_iter(); loop { match it.next() { ... } }`，然后经过上文所述的同一降级。对整数区间（`for i in a..b`，边界只求值一次）或数组的 `for` 直接降级为一个计数的 `while` 循环，它在函数体之前推进索引，因此 `continue` 无法跳过这一步。

## 8.9.5. `break` / `continue`

`break` 和 `continue` 是跳转到循环出口块或回边块的跳转。标签（`'outer:`）同样跳转到相应的[块](../000-meta/007-glossary.md#block块)。

## 8.9.6. `try` / `catch` / `finally`：基于值的分支

[错误控制流](../003-expressions/314-error-control-flow.md)中的错误通过[调用约定](806-calling-convention.md#865-抛出函数的返回基于值的错误传播)中定义的 `{ is_err, payload }` 返回值传播。因此 `try` 和 `catch` 都降级为普通分支。

```nabi
let v = try f(x);              // f: fn(X) throws (E) -> T
```

标有 `try` 的调用检查返回值的 `is_err`：成功时把载荷作为 `T` 取出；出错时分支到错误路径。

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

带 `catch` 子句的表达式有一个分派块，左侧表达式的所有错误路径都汇聚于此。

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

若没有子句匹配，分派块把错误交给外层 `catch` 的分派块，或者构建外围函数的错误返回值并返回。对无类型 `throws`（`any Error`）的错误所做的类型模式测试，降级为 [`any`](808-lowering-types.md) 的标签比较。

`try?` 在错误路径上产生 `none` 并转到 `bb_end`；`try!` 在错误路径上调用 `panic_fn`。

## 8.9.7. `throw`

`throw` 的用法如下。

```nabi
throw err;
```

`throw` 降级为一条无条件分支，转到错误路径。若最近的 `catch` 位于同一函数中，就转到那个分派块；否则构建外围函数的错误返回值并返回。

```
; when a catch exists in the same function
goto bb_catch_dispatch(err)

; otherwise, error return from the function
r.0 = true                ; is_err
r.1 = err                 ; the payload as E
return r
```

从函数中返回错误的路径与普通 `return` 相同，因此影子栈弹出等收尾代码照常运行。没有运行时调用。

## 8.9.8. `finally`

`finally` 的用法如下。

```nabi
try { body } finally { cleanup }
```

它的降级方式保证离开左侧表达式的每条路径都经过清理块。正常完成、经 `catch` 处理后完成、未处理错误的传播，以及每条 `return` / `break` / `continue` 路径，都先分支到清理块，再继续前往其原本的目的地。错误传播同样是普通分支，因此不需要单独的栈展开机制。

```
bb_body_ok(v):        goto bb_cleanup(K_OK, v)
bb_body_err(e):       goto bb_cleanup(K_ERR, e)
bb_body_return(rv):   goto bb_cleanup(K_RET, rv)

bb_cleanup(kind, val):
    ; cleanup
    ; depending on kind, branch to bb_end / error propagation / function return / loop destination
```

若在清理块内部抛出新的错误，它会取代原来的路径。panic 会终止进程，因此不经过清理块。

## 8.9.9. GC 分配：快速 / 慢速路径

请结合[分配](../007-runtime/702-allocation.md)阅读本节。`struct` 是值，从不分配（8.8.3）；堆对象是闭包环境、箱（`any`）、`List` / `Dict` 缓冲区以及运行时构建的字符串。

```nabi
let n = 10;
let f = |x: i32| x + n;    // env: header + an i32
```

环境在分配处以内联方式分配：

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

`t` 是线程的 TLS 基址（[8.7.7](807-code-generation.md#877-tls-访问)），`24` 是对象大小（向上取整到 8），`8` 是对齐，`TYPE_ID_ENV` 是 type_id。对象放得下时，快速路径推进 TLAB 游标，并写入头部的 `type_id` 和 `gc_word`（= 大小，[8.1.4](801-object-header-abi.md#814-gc_word偏移-8u64-或-ptr)）；头部的地址就是对象。否则调用 `nabi_gc_alloc_slow`。`gc_flags` 不写入：TLAB 内存是清零的。大小达到大对象阈值及以上的对象，或对齐超过 8 的对象，直接调用 `nabi_gc_alloc_slow`。

随后字段存储在 16 字节的头之后：

```
store.i32 [env + 16], n
```

## 8.9.10. 写屏障

这里是[屏障与根跟踪](../007-runtime/704-barriers-and-roots.md#742-写屏障对象记忆)中的对象记忆屏障。

```nabi
*p.field = q;       // write a GC pointer into a GC object
```

它降级为如下形式。

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

- 不读取被存储值 `q` 的头部。`q` 可能是 `any` 槽中的内联整数，也可能是 `str` 的内部指针，因此把它当作头部读取是不安全的。
- 快速路径是一次加载和一次分支。实际上它同时检查 `UNLOGGED | CARDED`。若为 `CARDED`（大的老对象），它不经调用，直接在卡表（其指针是紧挨在头部之前的那个字）中把被写入槽所跨越的 512 B 卡（`(slot - p) >> 9` ..= `(slot + len - 1 - p) >> 9`）标记为脏；若为 `UNLOGGED`，则如上调用 `nabi_gc_wb_slow`。因此屏障会用到槽地址和被存储值的大小（[7.4.2](../007-runtime/704-barriers-and-roots.md#742-写屏障对象记忆)）。
- `nabi_gc_wb_slow` 不是安全点。屏障与存储之间不会插入回收。
- 对刚分配对象的初始化、对 `static` 的存储、裸 `Ptr<T>` 内存，以及非引用的原子操作，不插入屏障。

## 8.9.11. 安全点轮询

另请参阅[安全点协议](804-safepoint.md)。

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

编译器在循环回边上生成这段代码，也在调用环可以不经过循环轮询就重新进入的函数中生成（递归环的成员、通过指针或从 C 调用的函数、导出函数）：对于并非由直接 Nabi 调用进入的函数，在入口处生成；对于其他函数，在每条路径的第一个安全点之前生成（[8.4.7](804-safepoint.md#847-安全点不变式)）。其他函数不在入口处轮询：由它们的调用方轮询。

## 8.9.12. 影子栈帧

另请参阅[屏障与根跟踪](../007-runtime/704-barriers-and-roots.md#741-根跟踪编译器直接生成的影子栈)。编译器直接生成栈帧的链接 / 解除链接，不需要平台的任何帮助。

下面是一个含 2 个 GC 引用字（`x`、`y`，栈槽）的函数示例。栈帧持有每个槽的地址，因此变量赋值只是一次向 `x` 的普通存储。

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

在 `-O0` 以上，栈帧布局确定后，槽会以更简短的方式列出各个字。一串 `n` 个相邻的字（三个或以上）记为条目 `(n << 1) | 1`，后跟第一个字的地址（地址是偶数，因此最低位可以区分两者）。含两个或以上条目的列表移到只读数据中：栈帧持有 `count` 0 和一个槽，即一个静态描述符 `{count: u16, entries: [u16; count]}` 的地址，其条目就是各槽的条目，只是每个地址被替换为该字距栈帧的偏移（以字计）的两倍（一个有符号 16 位值，与地址一样为偶数）。这样入口处只存储两个字，而不是整个列表。偏移放不进 16 位的列表留在栈帧中。描述符以其内容命名，链接时每种只保留一份副本。

```
; the same frame above -O0, its list in read-only data
; desc: u16 2, (x - frame) / p * 2, (y - frame) / p * 2
store.ptr [frame + 0], load.ptr [t + 0]
store.usize [frame + p], 0                 ; a descriptor follows
store.ptr [frame + 2p], &desc
store.ptr [t + 0], &frame
```

`t` 是 [8.3.8](803-tls-slots.md#838-编译器生成的形式) 中线程局部块 `nabi_tls` 的地址（[8.7.7](807-code-generation.md#877-tls-访问)）；影子栈头是其偏移 0 处的槽。错误返回同样是普通的 `return`（[8.9.6](#896-try--catch--finally基于值的分支)），因此经过同一次解除链接。链接也可以比示例中更晚，只要在每条路径的第一个安全点之前即可（[8.4.7](804-safepoint.md#847-安全点不变式)）；此时只有已链接的路径上的返回才会解除链接。只有当函数持有 GC 引用且能到达安全点时，才需要栈帧（[8.4.2](804-safepoint.md#842-生成位置)）。对从不返回的函数（panic）的调用，以及对既不分配也不阻塞的运行时函数（[8.5.2](805-extern-symbols.md#不是安全点的函数)）的调用，不计在内。

## 8.9.13. 索引

考虑[索引与切片](../003-expressions/306-indexing-and-slicing.md)中的 `arr[i]`。

```nabi
arr[5]
```

这个表达式变为对标准库函数 `Index::index(arr, 5)` 的调用，并像普通函数调用一样降级。

`Index` 调用被内联后（`-O1` 及以上），结果是边界检查加一次加载：

```
if 5 >= arr.len then bb_panic else bb_ok
bb_ok:
    val = load.T [arr.data + 5 * size_of(T)]
```

这一结果是泛型单态化和内联的自然产物。

## 8.9.14. 小结：热路径是什么样子

典型热路径（一次 [TLAB](../000-meta/007-glossary.md#tlab-thread-local-allocation-buffer) 分配加上字段写入）的开销是：

- TLAB 游标检查：load \* 2 / sub / 比较并分支
- 游标更新：store（1 条指令）
- 头部填充：store \* 2（type_id、gc_word）
- 字段填充：store \* N（字段数）
- 写屏障（如需要）：load / and / 比较并分支，+（慢速）调用
- 安全点轮询（如需要）：load / 比较并分支

慢速路径放在主路径之外。分配并填充一个小对象大约需要 10 ~ 20 条指令；在 `-O0`，每个值还要经过其栈槽，在 `-O0` 以上，放得下的值留在寄存器中。
