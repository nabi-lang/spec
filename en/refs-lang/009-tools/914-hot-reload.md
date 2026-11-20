# 9.14. Hot reload

*Hot reload* applies an edit to a running program: `nabi dev` (9.10.8) rebuilds the packages whose sources changed and the program loads the new code at a safepoint, keeping its heap, its threads and its statics. It is a development tool built on plugins ([9.12](912-plugins.md)): the code of every local package lives in a plugin that can be replaced, and the rest of the program never changes while it runs.

## 9.14.1. Model

A dev session has two kinds of module:

- The *host*: the runtime, the standard library's state, and every registry or git package. It is built once per session. It is an ordinary plugin host ([9.12.1](912-plugins.md#9121-model)) whose `main` is `std::reload::host_main()`: it connects the dev channel, loads the hot plugins named by `NABI_DEV_PLUGINS` and calls the root package's `main` (`NABI_DEV_MAIN`).
- The *hot plugins*: one per local package (the root package, its `path` dependencies, the workspace members it uses), built with `nabic build --hot` ([9.1](901-nabic-cli.md)). Each save makes a new version of the packages it touches.

Under `--hot`, every call into a function of a hot package goes through a slot, keyed by the function's mangled name and its signature. Loading a version binds each slot of every loaded version to the newest body with the same key. So:

- a function whose body changed is called in its new form from everywhere, old versions' code included;
- a function whose signature changed gets a new key and a new slot: callers built against the old signature keep calling the old body, and the new callers call the new one;
- old versions stay mapped until the session ends.

A hot package's package hash ([9.5](905-symbol-mangling.md)) is pinned for the session (it is derived from the package name), so symbols keep their names from version to version. Registry and git packages keep their content hashes.

`--hot` costs a load and an indirect branch per call into a hot package, no inlining of hot functions across functions, kept frame pointers, and the root re-read rule ([8.4](../008-abi/804-safepoint.md)). A normal build (`nabi build`, `nabi run`) has none of it. `--hot` builds the dev profile only.

## 9.14.2. When a version applies

The program loads a version on its dev channel thread and applies it at a stop of every thread, the same stop a collection uses ([7.3](../007-runtime/703-gc-algorithm.md)):

- A change of code only applies at the next stop.
- A change of a type's layout applies at the first stop at which no frame holds a value of a changed type (9.14.4); then the live values are migrated (9.14.3).
- `std::reload::point()` ([3.13](../../refs-standard-libraries/300-std/313-reload.md)) asks for a stop and is the usual place where a layout change passes. Outside `nabi dev` it does nothing.

A version that cannot apply yet stays pending and is tried at every stop; the next save replaces it. A version that can never apply is refused, and the program keeps running the code it has.

## 9.14.3. What may change

The compiler compares the new version's types with the running version's (`--hot-base`, the `.layout` file written beside each version) and refuses, as a compile error, what the program could not migrate:

| Change | Result |
|--------|--------|
| A field added with a default (`mana: i32 = 0;`) | allowed; live values get the default |
| A field added without a default | error: ``new field `mana` needs a default while the program runs: write `mana: i32 = 0;` (or restart with `r`)`` |
| A field removed | allowed; its value is dropped |
| Fields reordered | allowed |
| A field renamed with `#[renamed_from("hp")] health: i32` ([6.2](../006-meta-system/602-built-in-attributes.md)) | allowed; the value is carried over. Naming a field that did not exist is an error that lists the old fields. |
| A field's type changed | error: ``field `hp` of `Player` changed from `i32` to `f64` while the program runs; give the new field its own name with a default (`health: f64 = 100.0`) to start it fresh, or restart with `r` `` |
| An enum variant added or reordered | allowed; tags are mapped by name |
| An enum variant removed | allowed at compile time; while a live value holds it the version waits: ``waiting: a live `State::Paused` exists`` |
| A variant's payload changed | as fields, per variant |
| Generic parameters changed, or a struct turned into an enum | error; restart |
| A trait impl added, removed or changed | allowed; `any Tr` dispatch entries are replaced |

Migration is the last phase of a full collection: each object of a changed type is rewritten into the new layout, kept fields by name (or `#[renamed_from]`), added ones from their defaults, removed ones dropped, enum variants matched by name. An object is rewritten in place when the new layout fits in it, else into a new object, and every heap reference to it is updated. A rooted object (a frame's local, a global root) that must grow is not moved: it holds back the change (9.14.4). A first check with the world stopped finds what holds the change back, so a migration that cannot run changes nothing.

## 9.14.4. Code that is still running

| Thing | Rule |
|-------|------|
| A frame of an old function | finishes in the old code; the calls it makes reach the new code |
| `main`'s own loop | is old code for the whole session: per-frame work belongs in functions `main` calls |
| A frame holding a value of a changed type | holds back a layout change: ``waiting: `main` (src/main.nabi:12) holds a `World` across `reload::point()`; keep it in a `reload::State`: `let world = reload::State::new(World::new());` ``. `--hot` warns at the same place when it compiles. |
| A closure | keeps the body it was made with; one whose environment holds a changed type holds back a layout change: ``waiting: a closure made in `spawn_wave` (src/wave.nabi:30) holds an old `Enemy` `` |
| A suspended coroutine or task | resumes in its old code; one holding a changed type holds back a layout change, the message naming where it was spawned |
| Statics | keep their storage and value; a new static starts at its initializer; a static whose type changed layout is migrated |
| `any Tr` values | dispatch to the newest impl |
| A rooted, pinned or address-taken object of a changed type that grows | holds back the layout change |
| An `any` holding a value of a changed type inline, or an open no-collect region | holds back the layout change |

## 9.14.5. Dev channel

`nabi dev` talks to the program over a channel it creates before it starts the program and names in the program's environment as `NABI_DEV`: a unix stream socket's path, or on Windows a named pipe's name (`\\.\pipe\nabi-dev-<pid>-<n>`). The runtime connects at startup when `NABI_DEV` is set; a program started without it never opens a channel, and `std::reload::active()` is false.

Each message is one line of UTF-8 ending in `\n`.

| From | Line | Meaning |
|------|------|---------|
| program | `hello 1 <pid>` | the first line: protocol version 1 |
| `nabi dev` | `load <path>` | load this file, a new version of one hot package. Packages that changed together come as several lines, dependencies first. |
| program | `applied <pkg> v<n>: <what>` | the version is in: `applied game v3: 4 functions, 1 type (Enemy: +mana), 120 objects migrated` |
| program | `waiting <pkg> v<n>: <why>` | not yet (9.14.4); sent again when the reason changes; a final line for the same load follows |
| program | `refused <pkg> v<n>: <why>` | the version will not apply; the program keeps its code |
| program | `superseded <pkg> v<n>` | a newer `load` of the same package replaced this one while it waited |

`<pkg>` is the package's name as code spells it, `<n>` the version the program counts (1 is the one it started with). Loads apply in the order they were sent, and a waiting load holds back the loads behind it; a newer load of the same package takes the waiting one's place. Every `load` gets exactly one final reply (`applied`, `refused` or `superseded`), and the replies of one package come in the order of its loads; those of different packages may not. `nabi dev` never waits for a reply before it sends the next `load`. An unknown line is answered `refused <line>: unknown request`. The end of the channel is the end of the program.

## 9.14.6. Not in hot reload

- Registry and git packages, the host, and `nabi.nabion`: a change to them restarts the program (9.10.8).
- Converting a field's value to a new type.
- Moving an existing closure to a new body.
- Values held in C frames: a hot frame below a C callback holds back a layout change.
- wasm targets, release builds, and the REPL.
- Reloading plugins that a shipped program loads: hot reload is for `nabi dev` only.
