# 3.12. `panic::`: panic hook

`std::panic`은 프로세스가 끝나기 전에 프로그램이 모든 panic을 보게 합니다. 기록을 남기거나, 파일을 flush하거나, 호스트에 알리는 데 씁니다. panic은 잡을 수 없습니다([refs-lang 4.14.13](../../refs-lang/004-functions/414-throws.md#41413-panic과-fatal-에러)). hook은 나가는 길에 돕니다.

```nabi
// std::panic
pub struct PanicInfo {
    pub message: str;           // panic 의 메시지
    pub location: str | none;   // 알면 "path:line:col"
}

pub fn set_hook(hook: fn(PanicInfo));
pub fn take_hook();
```

## 3.12.1. `set_hook` / `take_hook`

`set_hook(hook)`은 그 뒤로 프로세스의 모든 panic에서 `hook`을 돌게 합니다. panic한 스레드에서, 보고를 출력한 뒤, 프로세스가 끝나기(코드 101), abort하기(C 프레임 아래의 panic, [refs-lang 5.9.12](../../refs-lang/005-modules/509-extern-block.md#5912-ffi의-안전성-책임)), 복구하기(`#[on_panic]`, [refs-lang 5.9.10](../../refs-lang/005-modules/509-extern-block.md#on_panic-abort-대신-c에-줄-값)) 전에 돕니다. hook은 클로저를 포함한 어떤 `fn(PanicInfo)` 값이든 됩니다. 나중의 `set_hook`이 hook을 바꾸고, `take_hook()`이 hook을 없앱니다. hook 안에서 난 panic은 보고되지만 hook에 다시 넘어가지 않습니다. hook이 보는 메시지는 앞의 1024바이트까지입니다. `PanicInfo`는 `Debug`를 구현합니다.

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

Python, R 같은 호스트가 불러오는 공유 라이브러리(`--crate-type=cdylib`)는 panic에서 exit하지 않고 abort합니다([refs-lang 5.9.12](../../refs-lang/005-modules/509-extern-block.md#5912-ffi의-안전성-책임)). hook은 그 전에 알릴 기회입니다.

런타임 쪽은 `nabi_panic_set_hook`입니다([refs-lang 8.5.12](../../refs-lang/008-abi/805-extern-symbols.md#8512-보조-심볼부록-비규범)).
