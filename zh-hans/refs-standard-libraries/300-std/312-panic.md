# 3.12. `panic::`：panic 钩子

`std::panic` 让程序在进程结束前看到每一个 panic：用来记录日志、刷新文件或通知宿主。panic 无法被捕获（[refs-lang 4.14.13](../../refs-lang/004-functions/414-throws.md#41413-panics-and-fatal-errors)）；钩子在退出的路上运行。

```nabi
// std::panic
pub struct PanicInfo {
    pub message: str;           // the panic's message
    pub location: str | none;   // "path:line:col", when known
}

pub fn set_hook(hook: fn(PanicInfo));
pub fn take_hook();
```

## 3.12.1. `set_hook` / `take_hook`

`set_hook(hook)` 使 `hook` 从此在进程中的每个 panic 上运行：在触发 panic 的线程上，在报告打印之后、进程退出（退出码 101）、中止（C 帧下的 panic，[refs-lang 5.9.12](../../refs-lang/005-modules/509-extern-block.md#5912-responsibility-for-ffi-safety)）或恢复（`#[on_panic]`，[refs-lang 5.9.10](../../refs-lang/005-modules/509-extern-block.md#on_panic---a-value-for-c-instead-of-an-abort)）之前。钩子可以是任何 `fn(PanicInfo)` 值，包括闭包。之后的 `set_hook` 会替换它，`take_hook()` 会移除它。钩子内部的 panic 会被报告，但不会再次传给钩子。钩子最多看到消息的前 1024 字节。`PanicInfo` 实现了 `Debug`。

```nabi
fn main() {
    std::panic::set_hook(|info| {
        eprintln(f"hook: {info.message}");
    });
    let xs: List<i32> = [];
    println(f"{xs[9]}");
}
```

```text
thread 'main' panicked at main.nabi:6:16:
index 9 out of bounds for length 0
hook: index 9 out of bounds for length 0
```

由 Python、R 或其他宿主加载的共享库（`--crate-type=cdylib`）遇到 panic 时会中止而不是退出（[refs-lang 5.9.12](../../refs-lang/005-modules/509-extern-block.md#5912-responsibility-for-ffi-safety)）：钩子是它先报告的机会。

运行时一侧是 `nabi_panic_set_hook`（[refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-auxiliary-symbols-appendix-non-normative)）。
