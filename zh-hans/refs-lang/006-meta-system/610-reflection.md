# 6.10. 反射

标准库提供两个层级的反射 trait：`Reflect` 和 `Inspect`。

```
Reflect    automatic: shallow metadata every type carries for free
Inspect    opt-in via #[derive(Inspect)]: rich field/method description
```

分为两个层级，既能控制二进制文件的大小，又能免费为每个类型提供 `any` 槽、和类型以及 REPL 自动打印所必需的信息。

## 6.10.1. `Reflect`：自动实现的 trait

`Reflect` 是自动附加在每个类型上的最小类型自省。用户无需编写 `#[derive(Reflect)]`，编译器会为每个类型合成一个 impl。

```nabi
trait Reflect {
    // static: called directly on the type
    fn type_name() -> str;
    fn size() -> usize;
    fn align() -> usize;

    // instance: runtime dispatch on any slots / sum types
    fn type_name_of_val(self) -> str;
    fn size_of_val(self) -> usize;
}
```

```nabi
i32::type_name() == "i32"
i32::size()      == 4
i32::align()     == 4

let x: any = 42i32;
x.type_name_of_val() == "i32"    // runtime type tag check

let v: i32 | f64 = pick();
v.type_name_of_val()              // "i32" or "f64" (sum type variant check)
```

由于是自动的，它不需要任何用户代码。编译器本来就知道每个类型的 `type_name`、`size` 和 `align`，以零成本合成它们。

`Reflect` 由编译器为每个类型提供；源代码中没有它的 impl。

- 手写 `impl Reflect for T` 是错误。约束 `T: Reflect` 始终成立。
- 如果类型自身有同名方法，例如 `size()` / `type_name()`，该方法会隐藏反射方法。
- 名字（`type_name`）不带模块路径（`List<i64>`、`Config`）。和类型的静态名字按规范顺序（[2.6.4](../002-types/206-sum-types.md#264-集合语义)）书写（`f64 | i32`，`none` 排在最后）。`type_name_of_val(self)` 等实例形式报告运行时所持有的成员（对和类型而言）或 `any` 的内容。
- `core::reflect::cast<T>(value: any) -> T | none` 按与 `set_field` 相同的规则把 `any` 取出为 `T`。类型的方法和字段通过 `T::methods()` / `T::fields()` 获取（[6.10.2](#6102-inspect选择启用的-trait)）；没有对应的自由函数。

## 6.10.2. `Inspect`：选择启用的 trait

`Inspect` 是通过 `#[derive(Inspect)]` 选择启用的丰富反射。它公开类型的描述（`TypeInfo`，6.10.7）、字段名、动态字段访问与修改，以及方法描述。

```nabi
trait Inspect: Reflect {
    fn type_info() -> TypeInfo;                             // the description (6.10.7)
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // private fields too
    fn set_field(ref self, name: str, value: any) -> bool;  // whether it succeeded

    // Every method has a default body over `type_info()`.
    fn fields() -> List<FieldInfo>;          // the field table of Self
    fn methods() -> List<MethodInfo>;        // the method table of Self
    fn variant_names() -> List<str>;         // empty for a struct
    fn variant_name(self) -> str;            // a struct: self.type_name_of_val()
    fn field_names_of_val(self) -> List<str>; // the fields self holds
}

struct FieldInfo {
    name: str;                 // "0", "1", .. for a tuple field
    type_name: str;            // instantiated: f64 for `v: T` in Box<f64>
    is_pub: bool;              // true only for a plain `pub` field
}

struct MethodInfo {
    name: str;
    arity: usize;              // number of arguments excluding self
    is_static: bool;           // a method that does not take self
    arg_types: List<str>;      // argument type names excluding self
    return_type: str;
}
```

- `get_field` 以 `any` 返回字段的值，名字不存在时返回 `none`。私有字段也可以读取。
- 当值的类型与字段**完全相同**，或者值是另一种整数类型的整数且**能无损容纳**时（如对 `u64` 字段执行 `u.set_field("id", 42)`），`set_field` 进行赋值并返回 `true`。否则不赋值并返回 `false`。没有其他转换（例如 `i32` → `f64`）。
- `methods()` **只返回**类型的**固有方法**（不含 trait 方法），按名字排序。泛型 `impl` 中的方法按书写形式显示类型（`T`）。
- `fields()` 按声明顺序列出每个字段及其类型名和是否为 `pub`。**私有字段也会列出**：派生 `Inspect` 即表示类型作者已经同意，与 `get_field` 相同。类型名是实例化后的名字（`Box<f64>` 报告 `v: f64`、`items: List<f64>`），不带模块路径，与 `type_name` 的写法一致。
- `enum` 在类型层面没有字段（`field_names()` 和 `fields()` 是空列表）。`variant_names()` 按声明顺序列出其变体。在值上，`variant_name()` 是它所持有的变体（`"Circle"`），`field_names_of_val()` 是该变体的字段（元组变体为 `"0"`、`"1"`，具名变体为字段名），`get_field` 读取这些字段。通过 `Inspect`，变体的载荷是只读的：在枚举上调用 `set_field` 返回 `false`。
- 在结构体上，`variant_names()` 是空列表，`variant_name()` 是其类型名（`type_name_of_val()`），`field_names_of_val()` 是其字段。
- 每个方法都有默认主体；`get_field`、`set_field` 和各名字列表读取 `type_info()`。derive 只写出 `type_info()` 所调用的访问器（6.10.7）。手写的 `impl Inspect` 可以覆盖任意方法。
- 在未派生的类型上调用 `Inspect` 方法是错误，其帮助信息会说明应写什么：``add `#[derive(Inspect)]` to `Point` ``。

用法示例：

```nabi
#[derive(Inspect)]
struct Config { host: str; port: i32; }

let c = Config { host: "localhost", port: 8080 };
Config::field_names()              // ["host", "port"]
c.get_field("port")                // 8080 inside an any
c.set_field("port", 9090)          // true (success)
c.set_field("xxx", 1)              // false (no such field)
Config::fields()[1].type_name      // "i32"
```

`Inspect` 之所以需要选择启用，是因为其元数据必须存储在二进制文件中。如果自动附加到每个类型上，二进制文件大小的代价会过高。即使有了 derive，在程序请求之前也不会存储任何内容：从不调用 `type_info()`、`info::<T>()` 或基于它们的方法的程序，构建出的代码与没有 derive 时相同。

`#[derive(Inspect)]` 由 `core::reflect` 中的[元块](608-meta-blocks.md) `#[impl_derive(Inspect)]` 实现，这是由元系统本身实现标准库 trait 的一种模式（[标准库 2.12.3](../../refs-standard-libraries/200-core/212-reflect.md#2123-inspect显式选择的字段--方法)）。

## 6.10.3. REPL / Jupyter 自动打印

REPL 打印单独的最后一个表达式时，使用 `Display`，否则使用 `Debug`；末尾带 `?` 时使用 `Debug`；`:type` 打印表达式的静态类型，它通过类型检查得到，不会运行该表达式（[9.7.7](../009-tools/907-jit-cells.md#977-可视化--输出)）。

```
>>> let x = 42i32;
>>> x
42

>>> [1, 2, 3]
[1, 2, 3]

>>> Config { host: "h", port: 80 }
Config { host: "h", port: 80 }

>>> Opaque { id: 1 }
Opaque
```

两者都没有实现的值（对末尾带 `?` 的情形而言是没有 `Debug`）只打印其类型名。REPL 和 Jupyter 单元格会话都使用这一行为（[9.7](../009-tools/907-jit-cells.md)）。

## 6.10.4. 没有全局 `typeof` / `sizeof` 函数

其他语言通过 `typeof` 或 `sizeof` 等关键字提供的自省，在 Nabi 中只以 trait 方法的形式提供。

```nabi
// nabi: method calls
T::size()
x.type_name_of_val()

// other languages: keywords
sizeof(T)        // C / C++
typeof(x)        // JS / TS
```

这样做的理由是不增加任何语言语法。单一的 `Reflect` trait 覆盖了全部自省功能。

## 6.10.5. 在 `any` 槽和和类型上的分派

`any` 槽和 `T1 | T2` 形式的和类型在运行时才确定实际类型，因此 `type_name_of_val` 调用很重要。

```nabi
fn handle(x: any) {
    if x.type_name_of_val() == "i32" { handle_int(x); }
    else if x.type_name_of_val() == "str" { handle_str(x); }
}

fn classify(v: i32 | f64) -> str {
    match v {
        n: i32 => "integer",
        f: f64 => "float",
    }
}
```

编译器把类型检查模式（`v: T => ...`）脱糖为 `Reflect` 标签检查。

## 6.10.6. 层级之间的关系

如 `Inspect: Reflect` 这一写法所示，`Inspect` 以 `Reflect` 为父 trait。因此，实现了 `Inspect` 的类型总是也实现了 `Reflect`，这一点不言而喻，因为 `Reflect` 本来就是自动的。

```
every type           Reflect          (automatic, zero cost)
+ #[derive(Inspect)] Inspect+Reflect  (opt-in, metadata embedded)
```

## 6.10.7. `TypeInfo`：类型的描述

`T::type_info()`（或对任意 `T` 使用 `core::reflect::info::<T>()`）返回一个 `TypeInfo`：编译器在只读数据中对 `T` 的描述（[8.13](../008-abi/813-type-descriptor.md)）。对于 `Inspect` 类型，它列出字段及其类型和字节偏移、枚举的变体及其字段、实例的类型实参以及运行时属性（6.10.8）；它还可以经由 `any` 访问值。

```nabi
use core::reflect::{Field, TypeInfo, TypeKind};

#[derive(Inspect)]
struct Enemy { name: str; hp: i32 = 100; }

let t = Enemy::type_info();
t.name()                           // "Enemy"
t.kind()                           // TypeKind::Struct
t.size() == Enemy::size()          // true
t.field("hp")                      // a Field: name, type_ref, offset, flags
let e: any = Enemy { name: "orc" };
let hp = t.field("hp") ?? panic("no hp");
hp.get(e)                          // 100 inside an any
```

- 读取描述不进行任何分配：`name`、`kind`、`size`、`field_count`、`field_at`、`field(name)`、`variant_at`、`attr::<A>()`。返回 `List` 的方法（`fields()`、`variants()`）会分配该列表。
- `Field::get(owner: any)` 返回字段的值，`owner` 不是该字段所属类型时返回 `none`。`Field::get_as::<T>(owner)` 以 `T` 返回该值。`Field::set(ref owner: any, value: any)` 遵循 `set_field` 的规则，并把修改后的副本存入 `owner`：结构体是值。
- 枚举的字段就是其各变体的字段（`Variant::fields()`）；`TypeInfo::variant_of(value)` 是值所持有的变体。在另一个变体的值上读取某个变体的字段得到 `none`。变体的载荷是只读的。
- `TypeInfo::build(fields: Dict<str, any>)` 根据具名字段构造一个值；省略的字段取其 `= expr` 默认值，没有默认值的字段是错误，报错会指出该字段。`Variant::build` 对枚举做同样的事。`new_default()` 是每个字段都取默认值的值，或 `none`。
- 列表、数组或字典的 `TypeInfo` 可以访问其元素：`list_items`、`list_len`、`list_get`、`list_from`、`dict_items`、`dict_from`；`element()` 和 `key()` 是元素的类型。
- `info_of(value: any)` 是 `value` 所持有类型的描述，`find(path)` 是位于 `"game::Player"` 这类路径上的类型的描述；当程序没有描述这样的类型时，两者都为 `none`。当程序请求某个类型或由它构成的类型时，该类型才会被描述。
- 两个 `TypeInfo` 描述同一类型时（按键判定，[8.13.2](../008-abi/813-type-descriptor.md#8132-键)）相等，即使来自不同的模块。
- 字段上的 `#[inspect(skip)]` 把它从描述以及 `field_names`、`get_field` 和 `set_field` 中排除。

## 6.10.8. 运行时属性

标记为 `#[runtime_attribute(name)]` 的结构体声明属性 `#[name(..)]`。把它写在类型、字段或枚举变体上时，其参数像该结构体的字面量一样进行检查，可以按字段顺序按位置给出，也可以按字段名给出，其值保存在描述中。在 `TypeInfo`、`Field` 或 `Variant` 上调用 `attr::<A>()` 可以带类型地读回它。

```nabi
/// Shown as a slider in the editor.
#[runtime_attribute(range)]
pub struct Range { pub min: f64; pub max: f64; }

#[derive(Inspect)]
struct Enemy {
    #[range(0, 100)] hp: i32 = 100;
    #[range(min = 0.5, max = 4.0)] speed: f64 = 1.0;
}

let hp = Enemy::type_info().field("hp") ?? panic("no hp");
hp.attr::<Range>()                 // Range { min: 0.0, max: 100.0 }
```

- 属性结构体的字段是数字、`bool`、`char` 或 `str`；整数字面量可以填入浮点字段。
- 参数写错是错误，报错会说明应写什么：``#[range] takes `min: f64, max: f64`; write `#[range(0, 10)]` ``。与已声明名字相近的名字会得到 ``no attribute `rnage`; did you mean `range` (`Range`)?``。
- 两个包声明同一个名字时，使用它的包所声明的那个胜出；否则该使用是错误，报错会指出两者。
- 只有以这种方式声明的属性才会被保留。`doc`、`cfg` 和其他内置属性从不保留。

字段上的 `#[renamed_from("old")]` 在其描述中记录该字段的旧名字（`Field::renamed_from`），供读取以旧名字写入的数据的工具使用。

## 6.10.9. 与插件和热重载的配合

描述位于定义该类型的模块中。每个模块在启动或被加载时把它的描述注册到运行时，`info_of` 和 `find` 在那里查找；`find` 返回最新模块对某个路径的描述。由于相等性按键判定，两个模块对同一泛型实例的描述比较时相等。
