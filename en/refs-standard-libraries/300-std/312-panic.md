# 3.12. `panic::` - the panic hook

`std::panic` lets a program see every panic before the process ends: to log it, flush a file, or tell a host. A panic cannot be caught ([refs-lang 4.14.13](../../refs-lang/004-functions/414-throws.md#41413-panics-and-fatal-errors)); the hook runs on the way out.

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

`set_hook(hook)` makes `hook` run on every panic in the process from then on, on the panicking thread, after the report is printed and before the process exits (code 101), aborts (a panic under C frames, [refs-lang 5.9.12](../../refs-lang/005-modules/509-extern-block.md#5912-responsibility-for-ffi-safety)) or recovers (`#[on_panic]`, [refs-lang 5.9.10](../../refs-lang/005-modules/509-extern-block.md#on_panic---a-value-for-c-instead-of-an-abort)). The hook is any `fn(PanicInfo)` value, a closure included. A later `set_hook` replaces it, and `take_hook()` removes it. A panic inside the hook is reported but not passed to the hook again. The hook sees at most the first 1024 bytes of the message. `PanicInfo` implements `Debug`.

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

A shared library (`--crate-type=cdylib`) that Python, R or another host loads aborts on a panic instead of exiting ([refs-lang 5.9.12](../../refs-lang/005-modules/509-extern-block.md#5912-responsibility-for-ffi-safety)): the hook is its chance to report first.

The runtime side is `nabi_panic_set_hook` ([refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-auxiliary-symbols-appendix-non-normative)).
