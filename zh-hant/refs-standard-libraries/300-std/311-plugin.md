# 3.11. `plugin::`：載入外掛

`std::plugin` 把外掛，也就是以 `lib: { kind: "plugin" }` 建置的函式庫套件，載入正在執行的程式，並把其中的函式取得為函式值（[9.12](../../refs-lang/009-tools/912-plugins.md)）。外掛在程式的執行環境上執行，所以它接收與回傳的都是程式的一般值。

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

## 3.11.1. 函式

```nabi
fn load(path: str) throws (PluginError) -> Plugin;  // load the file (a bare name: in the current directory); a loaded one gives that plugin
fn file(name: str) -> str;                          // "libname.dylib" / "libname.so" / "name.dll"
fn beside(name: str) -> str;                        // file(name) in the running executable's directory
fn loaded() -> List<Plugin>;                        // in load order, unloaded ones left out
fn wait_unloaded(p: Plugin, ms: u64) -> bool;       // collect until p is gone, at most ms milliseconds
```

`file` 給出 `nabi build` 在這個目標上為外掛套件寫出的檔名，`beside` 給出 `plugin: true` 相依套件所放的位置（9.10.3）。

## 3.11.2. `Plugin`

```nabi
struct Plugin;                       // Copy; copies name one plugin
fn name(self) -> str;                // its package name
fn version(self) -> str;             // its package version ("" when unknown)
fn path(self) -> str;                // the file it was loaded from
fn get<F>(self, path: str) throws (PluginError) -> F;
fn get_static<T>(self, path: str) throws (PluginError) -> T;
fn types(self) -> List<TypeInfo>;    // its type descriptors (core::reflect)
fn unload(self) -> Unload;
```

`get::<F>(path)` 依路徑在外掛的套件中尋找一個 `pub` 的非泛型函式，並以 `F` 的值回傳；`F` 必須正好是該函式的型別：簽章以「用型別鍵寫出的文字」的雜湊比較，所以其他型別會得到 `WrongType`，其 `have` 是函式宣告的型別。`F` 必須是函式型別；其他型別是編譯錯誤，錯誤會說明如何寫出函式型別，或改用 `get_static`。泛型函式不會被匯出：請匯出一個呼叫它的具體 `pub fn`。`get_static::<T>(path)` 讀取靜態變數 `path`。

`types()` 給出外掛程式碼向 `core::reflect` 查詢的型別，以及組成這些型別的型別的描述（[2.12.7](../200-core/212-reflect.md#2127-typeinfo)）。這些描述指向外掛內部：外掛卸載後不得再使用。

`unload` 是延後的（9.12.5）：

```nabi
enum Unload {
    Done,            // unloaded: nothing of it was alive
    Pending(str),    // no longer found; it goes once nothing of it is alive (what still holds it)
    Kept(str),       // stays loaded (why)
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
fn message(self) -> str;   // also its Display
```

每則訊息都會說明該怎麼做（[9.12.6](../../refs-lang/009-tools/912-plugins.md#9126-穩定性與錯誤) 的表格）。

## 3.11.4. 目標

外掛可以在 [9.12.7](../../refs-lang/009-tools/912-plugins.md#9127-目標) 列出的目標上載入。在 wasm 以及靜態連結的 musl 目標上，用到 `std::plugin` 的程式無法連結，錯誤會說明應建置成單一程式。
