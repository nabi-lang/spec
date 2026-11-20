# 3.11. `plugin::` - 플러그인 로드

`std::plugin`은 플러그인, 즉 `lib: { kind: "plugin" }`으로 빌드한 라이브러리 패키지를 실행 중인 프로그램에 로드하고 그 함수를 함수 값으로 얻습니다([9.12](../../refs-lang/009-tools/912-plugins.md)). 플러그인은 프로그램의 런타임 위에서 돌므로, 주고받는 것은 프로그램의 평범한 값입니다.

```nabi
use std::plugin;

let p = try plugin::load(plugin::beside("enemies"));
let spawn = try p.get::<fn(i32) -> List<str>>("enemies::spawn");
println(spawn(3));
match p.unload() {
    plugin::Unload::Done => {},
    plugin::Unload::Pending(why) => println(f"later: {why}"),
    plugin::Unload::Kept(why) => println(f"kept: {why}"),
}
```

## 3.11.1. 함수

```nabi
fn load(path: str) throws (PluginError) -> Plugin;  // 파일을 로드 (이름만 쓰면 현재 디렉터리). 이미 로드된 것이면 그 플러그인
fn file(name: str) -> str;                          // "libname.dylib" / "libname.so" / "name.dll"
fn beside(name: str) -> str;                        // 실행 파일이 있는 디렉터리의 file(name)
fn loaded() -> List<Plugin>;                        // 로드한 순서, 언로드한 것은 빠짐
fn wait_unloaded(p: Plugin, ms: u64) -> bool;       // p가 없어질 때까지, 최대 ms 밀리초 수거
```

`file`은 이 타깃에서 `nabi build`가 플러그인 패키지에 쓰는 파일의 이름이고, `beside`는 `plugin: true` 의존성이 놓이는 곳입니다(9.10.3).

## 3.11.2. `Plugin`

```nabi
struct Plugin;                       // Copy. 사본은 같은 플러그인을 가리킴
fn name(self) -> str;                // 패키지 이름
fn version(self) -> str;             // 패키지 버전 (모르면 "")
fn path(self) -> str;                // 로드한 파일
fn get<F>(self, path: str) throws (PluginError) -> F;
fn get_static<T>(self, path: str) throws (PluginError) -> T;
fn types(self) -> List<TypeInfo>;    // 타입 디스크립터 (core::reflect)
fn unload(self) -> Unload;
```

`get::<F>(path)`는 플러그인 패키지의 제네릭이 아닌 `pub` 함수를 경로로 찾아 `F` 값으로 돌려줍니다. `F`는 그 함수의 정확한 타입이어야 합니다. 시그니처는 타입 키로 쓴 텍스트의 해시로 비교하므로, 다른 타입이면 `WrongType`이며, 그 `have`는 함수가 선언된 타입입니다. `F`는 함수 타입이어야 하며, 아니면 함수 타입을 쓰는 법이나 `get_static`을 알려 주는 컴파일 에러입니다. 제네릭 함수는 내보내지 않으므로, 그것을 부르는 구체적인 `pub fn`을 내보내십시오. `get_static::<T>(path)`는 static `path`를 읽습니다.

`types()`는 플러그인의 코드가 `core::reflect`에 묻는 타입과 그것을 이루는 타입의 설명([2.12.7](../200-core/212-reflect.md#2127-typeinfo))을 돌려줍니다. 설명은 플러그인 안을 가리키므로 플러그인이 언로드된 뒤에는 쓰지 마십시오.

`unload`는 미뤄집니다(9.12.5).

```nabi
enum Unload {
    Done,            // 언로드됨. 살아 있는 것이 없었음
    Pending(str),    // 더는 찾지 못함. 살아 있는 것이 없어지면 내려감 (무엇이 붙잡고 있는지)
    Kept(str),       // 로드된 채 남음 (왜)
}
```

## 3.11.3. `PluginError`

```nabi
#[non_exhaustive]
enum PluginError {
    NotFound { path: str; },
    NotAPlugin { path: str; },
    Toolchain { plugin: str; program: str; },
    Target { plugin: str; program: str; },
    MissingPackages { names: List<str>; },
    MissingSymbols { names: List<str>; },
    Layout { type_path: str; },
    NoSuchItem { path: str; near: List<str>; },
    WrongType { path: str; want: str; have: str; },
    NotLoaded { name: str; },
    Os { message: str; },
}
fn message(self) -> str;   // Display도 같음
```

메시지마다 할 일을 알려 줍니다([9.12.6](../../refs-lang/009-tools/912-plugins.md#9126-안정성과-오류)의 표).

## 3.11.4. 타깃

플러그인은 [9.12.7](../../refs-lang/009-tools/912-plugins.md#9127-타깃)의 타깃에서 로드됩니다. wasm과 정적으로 링크하는 musl 타깃에서는 `std::plugin`에 닿는 프로그램이 링크되지 않으며, 프로그램 하나로 빌드하라는 에러가 납니다.
