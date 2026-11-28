# 2.12. reflect: `Reflect` / `Inspect`

`core::reflect` 모듈은 런타임 타입 정보와 **필드 인트로스펙션(introspection)**의 권위 시그니처를 정의합니다. 자세한 의미는 [refs-lang 6.10 reflection](../../refs-lang/006-meta-system/610-reflection.md)이 정합니다. 이 장은 표준 라이브러리 쪽 트레잇 형태만 다룹니다.

## 2.12.1. 두 단계 트레잇

리플렉션은 트레잇 두 단계로 나뉩니다.

| 트레잇    | 자동 구현 / opt-in            | 제공 항목                         | 사용 자리                                        |
|-----------|-------------------------------|-----------------------------------|--------------------------------------------------|
| `Reflect` | **자동** (모든 타입)          | type_name / size / align          | `core` 트레잇이라 어디서나 쓸 수 있음(드물게 씀)        |
| `Inspect` | opt-in (`#[derive(Inspect)]`) | field_names / fields / variant_name / get_field / methods | 묵시 prelude로 트레잇을 쓸 수 있음, derive는 명시 |

자동과 opt-in으로 나눈 이유는 바이너리 크기입니다. 모든 struct에 필드 리플렉션 메타데이터가 붙으면 바이너리가 커집니다. 그래서 opt-in 방식으로 필요한 타입에만 메타데이터를 만듭니다.

## 2.12.2. `Reflect`: 정적 + 인스턴스 모두

```nabi
trait Reflect {
    fn type_name() -> str;        // 정적: T::type_name()
    fn size() -> usize;
    fn align() -> usize;

    fn type_name_of_val(self) -> str; // 인스턴스: x.type_name_of_val() (any / sum 도 OK)
    fn size_of_val(self) -> usize;
}
```

이 트레잇은 모든 타입에 자동으로 구현됩니다. 컴파일러가 타입 정보를 바이너리에 자동으로 넣습니다.

컴파일러가 모든 타입에 `Reflect`를 구현하므로 소스에는 구현 코드가 없습니다.

- 손으로 쓴 `impl Reflect for T`는 에러입니다. `T: Reflect` 바운드는 늘 성립합니다.
- 타입에 `size()` / `type_name()`처럼 이름이 같은 메서드가 있으면 그 메서드가 리플렉션 메서드를 가립니다.
- 이름(`type_name`)에는 모듈 경로도, 기본값과 같은 뒤쪽 타입 인자도 붙지 않습니다(`List<i64>`, `Config`, `Dict<str, i32>`, [refs-lang 2.10.5](../../refs-lang/002-types/210-generics.md#2105-기본값)). 합 타입의 정적 이름은 정규 순서([2.6.4](../../refs-lang/002-types/206-sum-types.md#264-집합-의미론))로 적습니다(`f64 | i32`, `none`은 마지막). `type_name_of_val(self)` 같은 인스턴스 형태는 런타임에 실제로 든 항(합 타입일 때)이나 `any`의 내용을 보고합니다.
- `core::reflect::cast<T>(value: any) -> T | none`은 `set_field`와 같은 규칙으로 `any`를 `T`로 꺼냅니다. `get_field`나 `Field::get`으로 읽은 `T | none` 필드처럼 합 타입 그대로 담긴 값은 그 값이 가진 멤버를 줍니다. `"x"`를 가진 필드 `a: str | none`의 `cast::<str>`는 `"x"`입니다. 합 타입 `T`는 그 멤버 하나도 받으므로, `T | none` 필드의 `set_field`는 `T`를 받습니다. 타입의 메서드나 필드를 돌려주는 자유 함수는 없습니다. `T::methods()` / `T::fields()`로 얻으며, 타입의 동의인 `#[derive(Inspect)]`가 있어야 합니다.

### 정적과 인스턴스의 차이

접미사 규칙은 하나입니다. 접미사 없는 형태는 타입의 답으로 컴파일 시점에 정해지고, `_of_val` 형태는 실행 중 값의 답입니다.

```nabi
i32::type_name();          //→ "i32"           (정적, 컴파일 타임 결정)
i32::size();               //→ 4
i32::align();              //→ 4

let x: i32 = 42;
x.type_name_of_val();      //→ "i32"           (인스턴스, 보통 정적과 같음)

let y: i32 | f64 = 42i32;
y.type_name_of_val();      //→ "i32"           (런타임 dispatch, sum tag 검사)

let z: any = "hello";
z.type_name_of_val();      //→ "str"           (런타임, any slot dispatch)
```

`type_name_of_val`는 런타임 디스패치이므로 합 타입이나 any 슬롯에서는 지금 들어 있는 변형의 이름을 반환합니다.

### 전역 함수 없음

```nabi
typeof(x);                 // ERROR: 글로벌 typeof 없음
sizeof(T);                 // ERROR: 글로벌 sizeof 없음
```

대신 모두 메서드입니다.

```nabi
x.type_name_of_val()       // 인스턴스
T::type_name()             // 정적
T::size()
T::align()
```

이 규칙은 [refs-lang 3.5 멤버 접근과 메서드 호출](../../refs-lang/003-expressions/305-member-access-and-method-call.md)의 일부입니다. 모든 인트로스펙션은 메서드 형태입니다.

## 2.12.3. `Inspect`: opt-in 필드 / 메서드

```nabi
trait Inspect: Reflect {
    fn field_names() -> List<str>;
    fn get_field(self, name: str) -> any | none;            // 비공개 필드도
    fn set_field(ref self, name: str, value: any) -> bool;  // 성공 여부

    // 기본 본문이 있으며, enum 에서는 derive 가 덮어씁니다.
    fn fields() -> List<FieldInfo>;          // the field table of Self
    fn methods() -> List<MethodInfo>;        // the method table of Self
    fn variant_names() -> List<str>;         // struct 는 빈 목록
    fn variant_name(self) -> str;            // struct: self.type_name_of_val()
    fn field_names_of_val(self) -> List<str>; // self 가 지닌 필드
}

#[non_exhaustive]
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
pub struct FieldInfo {
    pub name: str;                 // 튜플 필드는 "0", "1", ..
    pub type_name: str;            // 인스턴스화된 타입: Box<f64> 의 `v: T` 는 f64
    pub is_pub: bool;              // 그냥 `pub` 인 필드만 true
}

#[non_exhaustive]
#[derive(Clone, PartialEq, Eq, Hash, Debug)]
pub struct MethodInfo {
    pub name: str;
    pub arity: usize;              // self 를 뺀 인자 수
    pub is_static: bool;           // self 를 받지 않는 메서드
    pub arg_types: List<str>;      // self 를 뺀 인자 타입 이름
    pub return_type: str;
}
```

- `get_field`는 그 필드의 값을 `any`로 돌려주고, 없는 이름이면 `none`을 돌려줍니다. 비공개 필드도 꺼낼 수 있습니다.
- `set_field`는 값의 타입이 필드 타입과 **정확히 같으면** 대입하고 `true`를 돌려줍니다. 정수는 타입이 달라도 **손실 없이 들어가면** 대입합니다(`u.set_field("id", 42)`가 `u64` 필드에). 그 밖에는 대입하지 않고 `false`를 돌려줍니다. 다른 변환(예: `i32` → `f64`)은 없습니다.
- `methods()`는 트레잇 메서드를 빼고 그 타입의 **고유 메서드만** 이름순으로 돌려줍니다. 제네릭 `impl`의 메서드는 타입을 적힌 그대로(`T`) 보여 줍니다.
- `fields()`는 모든 필드를 선언 순서대로 돌려줍니다. 항목마다 타입 이름과 `pub` 여부가 있습니다. **비공개 필드도 나옵니다**. `get_field`와 마찬가지로, 타입 작성자가 `Inspect`를 derive로 구현한 것을 동의로 봅니다. 타입 이름은 인스턴스화된 이름입니다(`Box<f64>`는 `v: f64`, `items: List<f64>`). `type_name`처럼 모듈 경로는 붙지 않습니다.
- `enum`에는 타입 수준 필드가 없습니다(`field_names()`와 `fields()`는 빈 목록). `variant_names()`는 변형을 선언 순서대로 돌려줍니다. 값에서 `variant_name()`은 지금 든 변형(`"Circle"`)입니다. `field_names_of_val()`는 그 변형의 필드입니다. 튜플 변형은 `"0"`, `"1"`이고 이름 있는 변형은 필드 이름입니다. 이 필드는 `get_field`로 읽습니다. 변형의 페이로드는 `Inspect`로 읽기만 할 수 있습니다. enum에서 `set_field`는 `false`를 돌려줍니다.
- struct에서 `variant_names()`는 빈 목록이고, `variant_name()`은 그 타입 이름(`type_name_of_val()`)이며, `field_names_of_val()`는 그 필드입니다.
- `fields()`, `methods()`, `variant_names()`, `variant_name()`, `field_names_of_val()`에는 기본 본문이 있습니다. 그래서 손으로 쓰는 `impl Inspect`는 앞의 세 메서드만 구현하면 됩니다.
- derive가 없는 타입에서 `Inspect` 메서드를 부르면 에러입니다. 도움말이 무엇을 쓸지 알려 줍니다(``add `#[derive(Inspect)]` to `Point` ``).

### derive 예

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

### 메타 블록 활용

`#[derive(Inspect)]`는 `core::reflect`의 메타 블록으로 구현합니다([refs-lang 6.8](../../refs-lang/006-meta-system/608-meta-blocks.md), [6.9](../../refs-lang/006-meta-system/609-template-tokens.md)). 메타 블록은 타입의 설명이 가리키는 접근 함수 세 개만 씁니다. 다른 메서드는 모두 `type_info()`를 읽는 기본 본문이 있습니다. 요약하면 다음과 같습니다.

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

### 동적 디스패치가 필요한 자리

`Inspect`는 설정 객체, 직렬화기, 디버거처럼 필드 이름으로 접근해야 하는 곳에서 씁니다. 일반 코드에서는 `Reflect`만으로 충분합니다.

```nabi
fn dump_struct<T: Inspect>(x: T) {
    for f in T::fields() {
        if let v: any = x.get_field(f.name) {
            println(f"{f.name}: {f.type_name} ({v.type_name_of_val()})");
        }
    }
}
```

## 2.12.4. REPL 출력

REPL은 평범한 마지막 식을 `Display`로, 없으면 `Debug`로 출력합니다. 뒤에 `?`를 붙이면 `Debug`로 출력합니다. 그 트레잇이 없는 값은 이 장의 `Reflect::type_name_of_val`로 출력합니다([2.10 fmt](210-fmt.md), [refs-lang 6.10.3](../../refs-lang/006-meta-system/610-reflection.md#6103-repl--jupyter-자동-출력)).

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

평범한 식은 `Display`를, 없으면 `Debug`를 쓰고, 뒤에 `?`가 붙으면 `Debug`를 씁니다. 둘 다 없는 값은 타입 이름만 출력합니다. 뒤에 `?`가 붙은 식은 `Debug`가 없을 때 그렇습니다. `any` 값은 담은 타입의 이름을 출력합니다.

## 2.12.5. 모듈 매핑

| 항목             | 위치                                             | 메커니즘                      |
|------------------|--------------------------------------------------|-------------------------------|
| `Reflect` 자동   | 컴파일러 + `core::reflect`                   | 타입 정보 테이블, 바이너리에 부착  |
| `Inspect` derive | `core::reflect` + 메타 블록 `#[impl_derive]` | 템플릿 토큰 기반 코드 생성 |
| `MethodInfo`     | `core::reflect`                              | 단순 struct                   |
| `FieldInfo`      | `core::reflect`                              | 컴파일러 필드 테이블 (인스턴스화된 타입) |
| `any` 타입       | 컴파일러 내장                                     | 타입 태그 + 페이로드            |

## 2.12.6. 다른 언어와 비교

| 측면               | Rust                         | Java             | Python              | Nabi                                         |
|--------------------|------------------------------|------------------|---------------------|---------------------------------------------|
| 타입 이름          | `std::any::type_name::<T>()` | `getClass()`     | `type(x).__name__`  | `x.type_name_of_val()` 메서드                   |
| 필드 introspection | **없음** (lib `bincode` 등)  | Reflection API   | `vars(x)` `getattr` | `x.get_field(name)` (opt-in)                |
| 런타임 디스패치    | `dyn Trait` (vtable)         | 자동             | 자동                | 합 타입 태그 + `any` / `any Trait` (opt-in) |
| 자동 구현          | **없음**                     | 모든 객체        | 모든 객체           | `Reflect` 자동, `Inspect` opt-in            |
| 메모리 비용        | 0                            | 모든 클래스 메타 | 모든 객체 dict      | `Reflect`만 자동 (opt-in 분리)             |

Nabi는 크게 두 가지가 다릅니다. 첫째, 트레잇을 두 단계로 나눠 가벼운 `Reflect`는 자동으로, 무거운 `Inspect`는 opt-in으로 둡니다. 그래서 Java나 Python처럼 모두 자동인 모델보다 메모리를 덜 쓰고, Rust처럼 모두 수동인 모델보다 쓰기 편합니다. 둘째, 메서드 일관성을 위해 `typeof(x)` 같은 글로벌 함수가 **없습니다**. 모든 인트로스펙션은 메서드 형태입니다.

리플렉션의 자세한 의미는 [refs-lang 6.10](../../refs-lang/006-meta-system/610-reflection.md)이 정합니다.

## 2.12.7. `TypeInfo`

`T::type_info()` / `info::<T>()`가 돌려주는 설명입니다([refs-lang 6.10.7](../../refs-lang/006-meta-system/610-reflection.md#6107-typeinfo-타입의-설명), 레이아웃 [8.13](../../refs-lang/008-abi/813-type-descriptor.md)). 값을 돌려주는 접근자는 설명을 그 자리에서 읽고 메모리를 할당하지 않습니다. `List`를 돌려주는 접근자는 그 리스트를 할당합니다.

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
    pub unsafe fn from_desc(at: usize) -> TypeInfo; // 주소 at의 디스크립터. 매핑된 채로 있어야 합니다
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

`std::nabion::to_value_any`와 `from_value_any`([3.8.5](../300-std/308-nabion.md#385-라이브러리-stdnabion))는 이것들 위에 만들어졌습니다.
