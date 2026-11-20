# 3.13. `reload::`: 핫 리로드 hook과 상태

`std::reload`는 `nabi dev` 아래의 핫 리로드([refs-lang 9.14](../../refs-lang/009-tools/914-hot-reload.md)) 가운데 프로그램이 보는 부분입니다. 변경이 적용되는 곳, 그 앞뒤로 도는 코드, 변경을 거쳐 살아남는 상태를 다룹니다. `nabi dev` 밖에서는 `point()`가 바로 돌아오고, hook은 돌지 않고, `State`는 보통 box입니다.

```nabi
// std::reload
pub fn active() -> bool;
pub fn point();
pub fn on_reload(name: str, f: fn(Reload) + Send + Sync);
pub fn before_reload(name: str, f: fn(Reload) + Send + Sync);

pub struct Reload {
    pub package: str;           // 바뀐 패키지
    pub version: u64;           // 새 버전
    pub functions: List<str>;   // 코드가 바뀐 함수, `game::spawn_wave` 꼴
    pub types: List<str>;       // 필드가 바뀐 타입, `game::Enemy` 꼴
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

`active()`는 프로그램이 `nabi dev` 아래에서 채널이 연결된 채로 돌 때 참입니다.

`point()`는 준비된 변경을 호출한 스레드에서 적용합니다. 모든 타입의 레이아웃을 그대로 두는 변경은 도착하는 대로 적용됩니다. 레이아웃을 바꾸는 변경은 옛 코드의 실행 중인 frame이나 살아 있는 값이 바뀐 타입에 기대지 않는 첫 정지에서 적용되며([refs-lang 9.14.4](../../refs-lang/009-tools/914-hot-reload.md#9144-아직-실행-중인-코드)), 보통 `point()`가 그곳입니다. 편집할 수 있는 타입의 값을 지역 변수가 담지 않는 곳, 곧 frame 루프의 맨 위에서 부르세요.

## 3.13.2. `on_reload` / `before_reload`

`on_reload(name, f)`는 변경이 적용된 뒤에, `before_reload(name, f)`는 적용되기 전에 `f`를 돌립니다. 둘 다 `point()` 안의 스레드에서 변경마다 한 번 돕니다. 같은 `name`으로 다시 등록한 hook은 옛 hook을 대신하므로, 돌 때마다 hook을 등록하는 코드도 hook이 쌓이지 않습니다. `before_reload` hook이 등록되어 있는 동안에는 모든 변경이 `point()`를 기다립니다. 컴파일러가 함수를 적어 주지 않았으면 `Reload.functions`는 빌 수 있습니다.

## 3.13.3. `State<T>`

`State<T>`는 `T`의 필드가 바뀌어도 값을 지킵니다. 값은 고정된 cell 뒤의 자기 box에 살므로, frame이 `State`를 쥐고 있어도 migration이 값을 다시 쓰거나 옮길 수 있습니다. `State`의 복사본은 값을 함께 씁니다. `get`은 복사본을 돌려주고, `set`은 저장하고, `update(f)`는 값에 `f`를 적용한 결과를 저장합니다.

계속 도는 frame(`main`의 루프)에 `T` 타입 지역 변수가 있으면 `T`의 변경이 기다립니다. 상태는 다른 함수에서 만들어, 루프가 `State<T>`만 쓰게 하세요.

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

`host_main()`은 `nabi dev`가 빌드해서 돌리는 프로그램의 `main` 전부입니다. 채널(`NABI_DEV`)에 연결하고, 핫 패키지마다 첫 버전(`NABI_DEV_PLUGINS`, 한 줄에 경로 하나, 의존성 먼저)을 [`plugin::load`](311-plugin.md)로 올리고, 마지막 것에서 `NABI_DEV_MAIN`이 가리키는 함수를 부릅니다. 올리기에 실패하면 plugin 에러를 출력하고 상태 101로 끝납니다. 프로그램이 직접 부르지는 않습니다.

런타임 쪽은 `nabi_hot_*`(`active`, `connect`, `ready`, `point`, `events`, `hold`)입니다.
