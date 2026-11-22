# 3.13. `reload::`：热重载钩子与状态

`std::reload` 是程序在 `nabi dev` 下能看到的热重载部分（[refs-lang 9.14](../../refs-lang/009-tools/914-hot-reload.md)）：变更在何处生效、在变更前后运行的代码，以及跨越变更保留的状态。在 `nabi dev` 之外，`point()` 立即返回，钩子从不运行，`State` 只是一个普通的盒子。

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

程序在 `nabi dev` 下运行且通道已连接时，`active()` 为 true。

`point()` 在调用线程上应用一个已就绪的变更。保持所有类型布局不变的变更一到达就生效；改变布局的变更在第一个满足条件的停止点生效，即没有运行中的帧或旧代码的存活值依赖于被修改的类型（[refs-lang 9.14.4](../../refs-lang/009-tools/914-hot-reload.md#9144-code-that-is-still-running)），而 `point()` 通常就是满足这一条件的地方。应当在没有局部变量持有你可能修改的类型的值的位置调用它，例如帧循环的顶部。

## 3.13.2. `on_reload` / `before_reload`

`on_reload(name, f)` 在每次变更生效后运行 `f`，`before_reload(name, f)` 在生效前运行，二者都在执行 `point()` 的线程上运行，每次变更一次。以相同 `name` 再次注册的钩子会替换旧钩子，所以每次运行都注册钩子的代码不会让钩子叠加。只要注册了 `before_reload` 钩子，每个变更都会等待 `point()`。编译器没有列出函数时，`Reload.functions` 可以为空。

## 3.13.3. `State<T>`

`State<T>` 在 `T` 的字段发生变更时保留一个值。该值存放在一个固定单元格后面的独立盒子中，所以即使某个帧持有 `State`，迁移也可以重写或移动这个值；`State` 的各个副本共享同一个值。`get` 返回副本，`set` 存入值，`update(f)` 存入以该值调用 `f` 的结果。

在持续运行的帧（`main` 的循环）中，类型为 `T` 的局部变量会阻止对 `T` 的变更生效。应当在另一个函数中创建状态，让循环只提到 `State<T>`：

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

`host_main()` 是 `nabi dev` 构建并运行的程序的整个 `main`：它连接通道（`NABI_DEV`），用 [`plugin::load`](311-plugin.md) 加载每个热包的第一个版本（`NABI_DEV_PLUGINS`，每行一个路径，依赖在前），并调用最后一个包中由 `NABI_DEV_MAIN` 指定的函数。加载失败时会打印插件错误并以状态 101 退出。程序不会自己调用它。

运行时一侧是 `nabi_hot_*`（`active`、`connect`、`ready`、`point`、`events`、`hold`）。
