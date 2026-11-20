# 3.13. `reload::` - hot reload hooks and state

`std::reload` is what a program sees of hot reload under `nabi dev` ([refs-lang 9.14](../../refs-lang/009-tools/914-hot-reload.md)): where a change applies, code that runs around it, and state that lives through it. Outside `nabi dev` `point()` returns at once, the hooks never run and a `State` is a plain box.

```nabi
// std::reload
pub fn active() -> bool;
pub fn point();
pub fn on_reload(name: str, f: fn(Reload) + Send + Sync);
pub fn before_reload(name: str, f: fn(Reload) + Send + Sync);

pub struct Reload {
    pub package: str;           // the package that changed
    pub version: u64;           // its new version
    pub functions: List<str>;   // functions whose code changed, as `game::spawn_wave`
    pub types: List<str>;       // types whose fields changed, as `game::Enemy`
}

pub struct State<T> { .. }
impl<T> State<T> {
    pub fn new(v: T) -> State<T>;
    pub fn get(self) -> T;
    pub fn set(self, v: T);
    pub fn update(self, f: fn(T) -> T);
}

pub fn host_main();
```

## 3.13.1. `active` / `point`

`active()` is true while the program runs under `nabi dev` with its channel connected.

`point()` applies a change that is ready, on the calling thread. A change that keeps every type's layout applies as soon as it arrives; one that changes a layout applies at the first stop where no running frame or live value of old code depends on a changed type ([refs-lang 9.14.4](../../refs-lang/009-tools/914-hot-reload.md#9144-code-that-is-still-running)), and `point()` is where that usually holds. Call it where no local holds a value of a type you may edit: the top of a frame loop.

## 3.13.2. `on_reload` / `before_reload`

`on_reload(name, f)` runs `f` after each change applies, `before_reload(name, f)` before it, both on the thread in `point()`, once per change. A hook registered again under the same `name` replaces the old one, so code that registers its hooks each time it runs does not stack them. While a `before_reload` hook is registered, every change waits for `point()`. `Reload.functions` may be empty when the compiler did not list them.

## 3.13.3. `State<T>`

A `State<T>` keeps a value across changes to `T`'s fields. The value lives in a box of its own behind a fixed cell, so a migration may rewrite or move it while a frame holds the `State`; copies of a `State` share the value. `get` returns a copy, `set` stores, `update(f)` stores `f` of the value.

A local of type `T` in a frame that keeps running (`main`'s loop) holds a change to `T` back. Make the state in another function, so the loop only names `State<T>`:

```nabi
fn main() {
    let state = new_state();
    loop {
        step(state);
        reload::point();
    }
}

fn new_state() -> reload::State<World> { reload::State::new(World::new()) }
```

## 3.13.4. `host_main`

`host_main()` is the whole `main` of the program `nabi dev` builds and runs: it connects the channel (`NABI_DEV`), loads the first version of each hot package (`NABI_DEV_PLUGINS`, one path a line, dependencies first) with [`plugin::load`](311-plugin.md) and calls the function `NABI_DEV_MAIN` names in the last one. A load that fails prints the plugin error and exits with status 101. Programs do not call it themselves.

The runtime side is `nabi_hot_*` (`active`, `connect`, `ready`, `point`, `events`, `hold`).
