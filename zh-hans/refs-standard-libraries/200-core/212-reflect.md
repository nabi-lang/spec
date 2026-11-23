# 2.12. reflect：`Reflect` / `Inspect`

`core::reflect` 模块提供运行时类型信息和**字段内省**的权威签名。详细语义由 [refs-lang 6.10 反射](../../refs-lang/006-meta-system/610-reflection.md)规定；本章只介绍标准库中 trait 的形状。

## 2.12.1. 两级 trait

反射分为两级 trait。

| trait     | 自动 / 显式选择               | 提供                              | 使用场景                                           |
|-----------|-------------------------------|-----------------------------------|----------------------------------------------------|
| `Reflect` | **自动**（所有类型）          | type_name / size / align          | `core` 的 trait，处处可见（很少直接使用）          |
| `Inspect` | 显式选择（`#[derive(Inspect)]`）| field_names / fields / variant_name / get_field / methods | trait 通过隐式 prelude 可用，derive 需显式写出 |

分为自动和显式选择两级是为了控制二进制文件的大小。如果每个结构体都带有字段反射元数据，二进制文件就会变大；显式选择的方案只为需要的类型生成元数据。

## 2.12.2. `Reflect`：静态与实例

```nabi
trait Reflect {
    fn type_name() -> str;        // static: T::type_name()
    fn size() -> usize;
    fn align() -> usize;

    fn type_name_of_val(self) -> str; // instance: x.type_name_of_val() (any / sum also OK)
    fn size_of_val(self) -> usize;
}
```

这个 trait 自动为所有类型实现；编译器自动把类型信息附加到二进制文件中。

编译器为每个类型提供 `Reflect`；源代码中没有它的实现。

- 手写 `impl Reflect for T` 是错误。约束 `T: Reflect` 总是成立。
- 如果类型自己有同名方法，例如 `size()` / `type_name()`，该方法会遮蔽反射方法。
- 名称（`type_name`）不带模块路径（`List<i64>`、`Config`），也不带末尾等于默认值的类型实参（`Dict<str, i32>`，[refs-lang 2.10.5](../../refs-lang/002-types/210-generics.md#2105-默认值)）。和类型的静态名称按规范顺序写出（[2.6.4](../../refs-lang/002-types/206-sum-types.md#264-集合语义)）（`f64 | i32`，`none` 在最后）。`type_name_of_val(self)` 等实例形式报告运行时实际存在的成员（对于和类型）或 `any` 的内容。
- `core::reflect::cast<T>(value: any) -> T | none` 按与 `set_field` 相同的规则从 `any` 中取出 `T`。没有获取类型方法或字段的自由函数：它们是 `T::methods()` / `T::fields()`，并且需要 `#[derive(Inspect)]`，即类型的许可。

### 静态与实例

一条后缀规则：普通形式是类型给出的回答，在编译时确定；`_of_val` 形式是值在运行时给出的回答。

```nabi
i32::type_name();          //→ "i32"           (static, decided at compile time)
i32::size();               //→ 4
i32::align();              //→ 4

let x: i32 = 42;
x.type_name_of_val();      //→ "i32"           (instance, usually the same as static)

let y: i32 | f64 = 42i32;
y.type_name_of_val();      //→ "i32"           (runtime dispatch, checks the sum tag)

let z: any = "hello";
z.type_name_of_val();      //→ "str"           (runtime, any slot dispatch)
```

`type_name_of_val` 是运行时分派，所以对于和类型或 `any`，它返回当前持有的变体的名称。

### 没有全局函数

```nabi
typeof(x);                 // ERROR: no global typeof
sizeof(T);                 // ERROR: no global sizeof
```

取而代之的是，一切都统一为方法。

```nabi
x.type_name_of_val()   // instance
T::type_name()             // static
T::size()
T::align()
```

这是 [refs-lang 3.5 成员访问与方法调用](../../refs-lang/003-expressions/305-member-access-and-method-call.md)的一部分：所有内省都统一为方法形式。

## 2.12.3. `Inspect`：显式选择的字段 / 方法

```nabi
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // private fields too
    fn set_field(ref self, name: str, value: any) -> bool;  // whether it succeeded

    // Default bodies; the derive overrides the value forms for an enum.
    fn fields() -> List<FieldInfo>;          // the field table of Self
    fn methods() -> List<MethodInfo>;        // the method table of Self
    fn variant_names() -> List<str>;         // empty for a struct
    fn variant_name(self) -> str;            // a struct: self.type_name_of_val()
    fn field_names_of_val(self) -> List<str>; // the fields self holds
}

#[non_exhaustive]
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
pub struct FieldInfo {
    pub name: str;                 // "0", "1", .. for a tuple field
    pub type_name: str;            // instantiated: f64 for `v: T` in Box<f64>
    pub is_pub: bool;              // true only for a plain `pub` field
}

#[non_exhaustive]
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
pub struct MethodInfo {
    pub name: str;
    pub arity: usize;              // number of arguments excluding self
    pub is_static: bool;           // method that does not take self
    pub arg_types: List<str>;      // argument type names excluding self
    pub return_type: str;
}
```

- `get_field` 以 `any` 返回字段的值，名称未知时返回 `none`。私有字段也可以读取。
- 当值的类型与字段的类型**完全相同**，或者另一种整数类型的整数**能无损放入**时（向 `u64` 字段执行 `u.set_field("id", 42)`），`set_field` 赋值并返回 `true`。否则不赋值并返回 `false`。没有其他转换（例如 `i32` → `f64`）。
- `methods()` **只返回类型的固有方法**（不含 trait 方法），按名称排序。泛型 `impl` 中的方法按书写形式显示类型（`T`）。
- `fields()` 按声明顺序列出每个字段及其类型名和是否 `pub`。**私有字段也会列出**：派生 `Inspect` 就是类型作者的许可，与 `get_field` 相同。类型名是实例化后的名称（`Box<f64>` 报告 `v: f64`、`items: List<f64>`），与 `type_name` 的写法一样不带模块路径。
- `enum` 在类型层面没有字段（`field_names()` 和 `fields()` 是空列表）。`variant_names()` 按声明顺序列出其变体。在值上，`variant_name()` 是它持有的变体（`"Circle"`），`field_names_of_val()` 是该变体的字段（元组变体为 `"0"`、`"1"`，具名变体为字段名），`get_field` 读取这些字段。通过 `Inspect` 访问时，变体的载荷是只读的：对枚举调用 `set_field` 返回 `false`。
- 在结构体上，`variant_names()` 是空列表，`variant_name()` 是其类型名（`type_name_of_val()`），`field_names_of_val()` 是其字段。
- `fields()`、`methods()`、`variant_names()`、`variant_name()` 和 `field_names_of_val()` 都有默认主体，所以手写的 `impl Inspect` 只需要前三个方法。
- 在没有派生的类型上调用 `Inspect` 方法是错误，其帮助信息会说明该写什么：``add `#[derive(Inspect)]` to `Point` ``。

### 派生示例

```nabi
#[derive(Inspect)]
struct User {
    id: u64;
    name: str;
    email: str;
}

let u = User { id: 1, name: "Alice", email: "a@b" };

User::field_names();       //→ ["id", "name", "email"]
u.get_field("name");       //→ any("Alice")
u.set_field("id", 42);     //→ true
u.id;                      //→ 42
User::fields()[0].type_name; //→ "u64"

#[derive(Inspect)]
enum Shape { Circle(f64), Rect { w: f64; h: f64; } }

let s = Shape::Rect { w: 2.0, h: 3.0 };
Shape::variant_names();    //→ ["Circle", "Rect"]
s.variant_name();          //→ "Rect"
s.field_names_of_val();    //→ ["w", "h"]
s.get_field("w");          //→ any(2.0)
```

### 使用元块

`#[derive(Inspect)]` 在 `core::reflect` 中以元块实现（[refs-lang 6.8](../../refs-lang/006-meta-system/608-meta-blocks.md)、[6.9](../../refs-lang/006-meta-system/609-template-tokens.md)）。它只编写类型描述所指向的三个访问器；其他所有方法都有基于 `type_info()` 的默认主体。概要如下：

```nabi
#[impl_derive(Inspect)]
pub @derive {
    impl core::reflect::Inspect for @Self {
        // Field `field` of `owner`: a struct's by position; an enum's as
        // `variant << 16 | index`, and `usize::MAX` for the variant index.
        fn __inspect_get(owner: any, field: usize) -> any | none {
            match core::reflect::cast::<@Self>(owner) {
                s: @Self => {
                    @if_struct {
                        @for_fields {
                            if field == @field.index { let held: any = s.@field; return held; }
                        }
                    }
                    @if_enum { /* per variant, through @variant.pattern */ }
                    return none;
                },
                none => { return none; },
            }
        }
        // A copy of `owner` with one field set (structs only).
        fn __inspect_set(owner: any, field: usize, value: any) -> any | none { /* .. */ }
        // A value from named fields; a missing field takes its default.
        fn __inspect_build(variant: usize, fields: Dict<str, any>) -> (any | none, str) {
            @if_struct {
                @for_fields {
                    let @field.binding: @field.type = /* fields[name], or @field.default */;
                }
                let built: @Self = @Self.pattern;
                let held: any = built;
                return (held, "");
            }
            /* @if_enum: the same per variant */
        }
    }
}
```

### 需要动态分派的场景

`Inspect` 用于需要按字段名访问的领域，例如配置对象、序列化器和调试器。普通代码中只用 `Reflect` 就够了。

```nabi
fn dump_struct<T: Inspect>(x: T) {
    for f in T::fields() {
        if let v: any = x.get_field(f.name) {
            println(f"{f.name}: {f.type_name} ({v.type_name_of_val()})");
        }
    }
}
```

## 2.12.4. REPL 输出

REPL 用 `Display`（没有则用 `Debug`）打印裸的最后一个表达式，用 `Debug` 打印末尾带 `?` 的表达式，两者都没有的值则用本章的 `Reflect::type_name_of_val` 打印（[2.10 fmt](210-fmt.md)、[refs-lang 6.10.3](../../refs-lang/006-meta-system/610-reflection.md#6103-repl--jupyter-自动打印)）。

```
>>> let x = 42i32;
>>> x
42

>>> [1, 2, 3]
[1, 2, 3]

>>> [1, 2, 3]?
[1, 2, 3]

>>> User { id: 1, name: "Alice" }?
User { id: 1, name: "Alice" }
```

裸表达式使用 `Display`，没有则用 `Debug`；末尾的 `?` 使用 `Debug`。两者都没有的值（对末尾的 `?` 而言是没有 `Debug`）只打印其类型名；`any` 值打印它所持有的类型的名称。

## 2.12.5. 模块对应

| 项                 | 位置                                              | 机制                              |
|--------------------|---------------------------------------------------|-----------------------------------|
| `Reflect`（自动）  | 编译器 + `core::reflect`                  | 类型信息表，附加到二进制文件中    |
| `Inspect` derive   | `core::reflect` + 元块 `#[impl_derive]`       | 由模板记号生成代码                |
| `MethodInfo`       | `core::reflect`                              | 普通结构体                        |
| `FieldInfo`       | `core::reflect`                              | 编译器字段表（实例化后的类型）    |
| `any` 类型         | 编译器内置                                        | 类型标签 + 载荷                   |

## 2.12.6. 与其他语言的比较

| 方面                | Rust                         | Java               | Python              | Nabi                                         |
|---------------------|------------------------------|--------------------|---------------------|---------------------------------------------|
| 类型名              | `std::any::type_name::<T>()` | `getClass()`       | `type(x).__name__`  | `x.type_name_of_val()` 方法                 |
| 字段内省            | **无**（`bincode` 等库）     | Reflection API     | `vars(x)` `getattr` | `x.get_field(name)`（显式选择）             |
| 运行时分派          | `dyn Trait`（vtable）        | 自动               | 自动                | 和类型标签 + `any` / `any Trait`（显式选择）|
| 自动实现            | **无**                       | 所有对象           | 所有对象            | `Reflect` 自动，`Inspect` 显式选择          |
| 内存开销            | 0                            | 每个类都有元数据   | 每个对象都有 dict   | 只有 `Reflect` 自动（显式选择分级）         |

Nabi 主要有两点不同。第一，两级划分自动提供轻量的 `Reflect`，而重量级的 `Inspect` 需要显式选择。这比 Java 或 Python 的全自动模型更节省内存，也比 Rust 的全手动模型更方便。第二，为了方法的一致性，**没有** `typeof(x)` 这样的全局函数；所有内省都以方法形式提供。

详细的反射语义由 [refs-lang 6.10](../../refs-lang/006-meta-system/610-reflection.md) 规定。

## 2.12.7. `TypeInfo`

`T::type_info()` / `info::<T>()` 返回的描述（[refs-lang 6.10.7](../../refs-lang/006-meta-system/610-reflection.md#6107-typeinfo类型的描述)，布局见 [8.13](../../refs-lang/008-abi/813-type-descriptor.md)）。返回值的访问器就地读取，不做任何分配；返回 `List` 的访问器会分配该列表。

```nabi
pub fn info<T>() -> TypeInfo;                 // any T; fields and accessors when T: Inspect
pub fn info_of(value: any) -> TypeInfo | none; // the type value holds, if described
pub fn find(path: str) -> TypeInfo | none;    // "game::Player"

pub enum TypeKind { Struct, Enum, Union, Sum, Prim, Str, List, Dict, Array, Tuple, Fn, Any, Opaque }
pub struct ReflectError { pub message: str; }  // + Display, Error

impl TypeInfo {                                 // + Clone, PartialEq (by key), Debug
    pub fn name(self) -> str;                   // as type_name: "Player", "List<f64>"
    pub fn path(self) -> str;                   // "game::Player"
    pub fn package(self) -> str;
    pub fn type_key(self) -> str;               // "Y3f8a1c2b:game::Player"
    pub fn layout_hash(self) -> u64;
    pub fn kind(self) -> TypeKind;
    pub fn size(self) -> usize;
    pub fn align(self) -> usize;
    pub fn is_inspect(self) -> bool;
    pub fn fields(self) -> List<Field>;
    pub fn field_count(self) -> usize;
    pub fn field_at(self, i: usize) -> Field;
    pub fn field(self, name: str) -> Field | none;
    pub fn field_of_val(self, value: any, name: str) -> Field | none;
    pub fn variants(self) -> List<Variant>;
    pub fn variant_count(self) -> usize;
    pub fn variant_at(self, i: usize) -> Variant;
    pub fn variant(self, name: str) -> Variant | none;
    pub fn variant_of(self, value: any) -> Variant | none;
    pub fn generics(self) -> List<TypeRef>;
    pub fn generic_count(self) -> usize;
    pub fn generic_at(self, i: usize) -> TypeRef;
    pub fn element(self) -> TypeRef | none;     // list / array element, dict value
    pub fn key(self) -> TypeRef | none;         // dict key
    pub fn attr<A>(self) -> A | none;
    pub fn attrs(self) -> List<Attr>;
    pub fn holds(self, value: any) -> bool;
    pub fn new_default(self) -> any | none;
    pub fn build(self, fields: Dict<str, any>) throws (ReflectError) -> any;
    pub fn list_len(self, value: any) -> usize;
    pub fn list_get(self, value: any, i: usize) -> any | none;
    pub fn list_items(self, value: any) -> List<any> | none;
    pub fn list_from(self, items: List<any>) throws (ReflectError) -> any;
    pub fn dict_items(self, value: any) -> List<(any, any)> | none;
    pub fn dict_from(self, items: List<(any, any)>) throws (ReflectError) -> any;
    pub unsafe fn from_desc(at: usize) -> TypeInfo; // the descriptor at address at, which must stay mapped
}

impl TypeRef { pub fn name(self) -> str; pub fn info(self) -> TypeInfo | none; }

impl Field {
    pub fn name(self) -> str;                   // "0", "1", .. in a tuple
    pub fn renamed_from(self) -> str;
    pub fn type_ref(self) -> TypeRef;
    pub fn offset(self) -> usize;               // usize::MAX: no fixed place
    pub fn index(self) -> usize;
    pub fn is_pub(self) -> bool;
    pub fn has_default(self) -> bool;
    pub fn attr<A>(self) -> A | none;
    pub fn attrs(self) -> List<Attr>;
    pub fn get(self, owner: any) -> any | none;
    pub fn get_as<T>(self, owner: any) -> T | none;
    pub fn set(self, ref owner: any, value: any) -> bool;
}

impl Variant {
    pub fn name(self) -> str;
    pub fn index(self) -> usize;
    pub fn fields(self) -> List<Field>;
    pub fn field_count(self) -> usize;
    pub fn field_at(self, i: usize) -> Field;
    pub fn field(self, name: str) -> Field | none;
    pub fn attr<A>(self) -> A | none;
    pub fn attrs(self) -> List<Attr>;
    pub fn build(self, fields: Dict<str, any>) throws (ReflectError) -> any;
}

impl Attr { pub fn type_ref(self) -> TypeRef; pub fn get<A>(self) -> A | none; }
```

`std::nabion::to_value_any` 和 `from_value_any`（[3.8.5](../300-std/308-nabion.md#385-库stdnabion)）建立在这些之上。
