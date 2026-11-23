# 3.13. `reload::`：熱重載鉤子與狀態

`std::reload` 是程式在 `nabi dev` 下所看到的熱重載（[refs-lang 9.14](../../refs-lang/009-tools/914-hot-reload.md)）：變更在哪裡套用、在變更前後執行的程式碼，以及在變更後仍存續的狀態。在 `nabi dev` 之外，`point()` 會立即回傳，鉤子永遠不會執行，而 `State` 只是一個普通的 box。

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

程式在 `nabi dev` 下執行且其通道已連線時，`active()` 為 true。

`point()` 在呼叫它的執行緒上套用一個已就緒的變更。保留所有型別佈局的變更一到達就套用；改變佈局的變更，則在第一個沒有任何執行中框架或舊程式碼的存活值相依於已變更型別的停止點套用（[refs-lang 9.14.4](../../refs-lang/009-tools/914-hot-reload.md#9144-仍在執行的程式碼)），而 `point()` 通常就是滿足這個條件的地方。應當在沒有任何區域變數持有你可能修改之型別的值的地方呼叫它：影格迴圈的開頭。

## 3.13.2. `on_reload` / `before_reload`

`on_reload(name, f)` 在每個變更套用之後執行 `f`，`before_reload(name, f)` 則在套用之前執行；兩者都在 `point()` 所在的執行緒上執行，每個變更一次。以相同 `name` 再次註冊的鉤子會取代舊的，因此每次執行時都註冊鉤子的程式碼不會讓鉤子疊加。只要註冊了 `before_reload` 鉤子，每個變更都會等待 `point()`。編譯器沒有列出函式時，`Reload.functions` 可以是空的。

## 3.13.3. `State<T>`

`State<T>` 在 `T` 的欄位變更之間保留一個值。這個值存放在固定儲存格之後自己的 box 中，因此即使有框架持有該 `State`，遷移也可以重寫或移動這個值；`State` 的副本共用同一個值。`get` 回傳一份副本，`set` 儲存值，`update(f)` 儲存 `f` 套用於該值的結果。

在持續執行的框架中（`main` 的迴圈），型別為 `T` 的區域變數會阻擋對 `T` 的變更。應當在另一個函式中建立狀態，讓迴圈只提及 `State<T>`：

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

`host_main()` 是 `nabi dev` 建置並執行的程式的完整 `main`：它連接通道（`NABI_DEV`），用 [`plugin::load`](311-plugin.md) 載入每個熱套件的第一個版本（`NABI_DEV_PLUGINS`，每行一個路徑，相依套件在前），並呼叫最後一個套件中 `NABI_DEV_MAIN` 所指名的函式。載入失敗時會印出外掛錯誤，並以結束碼 101 結束。程式不會自行呼叫它。

執行環境端是 `nabi_hot_*`（`active`、`connect`、`ready`、`point`、`events`、`hold`）。
