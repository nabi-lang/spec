# 1.10. Objective-C: `extern "objc"`와 `ffi::objc`

Objective-C 객체는 직접 씁니다. 메서드 호출은 메서드 시그니처의 C ABI([1.7](107-ffi.md))로 하는 `objc_msgSend` 호출이고, 셀렉터와 클래스는 이미지가 로드될 때 런타임이 고쳐 쓰는 정적 참조입니다. C 컴파일러도 shim도 끼지 않습니다. 클래스와 그 메서드는 `extern "objc"` 블록(1.10.2)에 선언합니다. 블록은 손으로 쓰거나 `ffi::objc::import`(1.10.12)로 SDK 헤더에서 읽어 옵니다. `bare::ffi::objc`에는 모든 프로그램이 함께 쓰는 것이 있습니다. `Id<T>`, `Sel`, `Class`, `IsA`, 루트 클래스 `NSObject`, `NSString`, `NSError`, autorelease pool, 블록, dispatch 큐, 실행 중에 만드는 클래스입니다.

## 1.10.1. 타깃과 링크

Objective-C는 Apple 타깃(`aarch64-apple-darwin`, `aarch64-apple-ios-sim`, 진행 중인 `aarch64-apple-ios`)에만 있고, `bare::ffi::objc`도 마찬가지입니다. 다른 타깃에서 `extern "objc"` 블록은 컴파일 오류(E0084)이고, 도움말은 블록과 그것을 쓰는 코드를 `#[cfg(any(target_os = "macos", target_os = "ios"))]` 아래에 두라고 알려 줍니다.

Objective-C 런타임을 부르는 프로그램(`extern "objc"` 블록의 메서드나 `bare::ffi::objc`의 함수를 쓰는 프로그램)은 따로 말하지 않아도 libobjc를 링크합니다. 프레임워크는 다른 라이브러리처럼 블록(또는 `extern "C"` 블록)에 `#[link(name = "Foundation", kind = "framework")]`를 붙여 링크합니다.

## 1.10.2. `extern "objc"` 블록

```
objc_block  ::= objc_attr "extern" "\"objc\"" "{" method* "}"
objc_attr   ::= "#[objc::class(" ident ("," "extends(" ident ")")? ("," "conforms(" idents ")")? ")]"
              | "#[objc::protocol(" ident ("," "extends(" ident ")")? ("," "conforms(" idents ")")? ")]"
              | "#[objc::category(" ident ")]"
method      ::= attribute* visibility? "unsafe"? "fn" ident "(" ("self" ("," params)? | params?) ")"
                ("throws" "(" type ")")? ("->" type)? ";"
```

블록 하나는 다음 세 속성 가운데 하나로 클래스 하나를 정확히 가리킵니다.

| 속성 | 선언하는 것 |
|------|-------------|
| `#[objc::class(C)]`, `#[objc::class(C, extends(S), conforms(P, Q))]` | 슈퍼클래스가 `S`(생략하면 `NSObject`)이고 프로토콜 `P`, `Q`를 따르는 클래스 `C`: 타입 `C`, 그 타입의 `Object`, 그 타입과 그것이 속하는 종류의 `IsA`, 메서드, `Id<C>::as_super() -> Id<S>` |
| `#[objc::protocol(P)]`, `#[objc::protocol(P, extends(Q), conforms(R))]` | 이름은 알 수 없지만 프로토콜 `P`를 따르는 클래스의 객체를 위한 타입 `P`(`MTLDevice`). 부모 프로토콜은 `Q`, `R`. 인스턴스 메서드만 |
| `#[objc::category(C)]` | 다른 블록이나 `bare::ffi::objc`(`NSObject`, `NSString`, `NSError`)가 선언한 클래스 `C`의 메서드 추가 |

셋 중 어느 것도 없는 블록, 두 번 선언한 클래스, 선언되지 않은 클래스나 프로토콜을 가리키는 `extends`, `conforms`, 카테고리, 블록 안의 `static`은 오류이고, 오류는 써야 할 속성을 알려 줍니다. 블록이 만드는 타입은 모두 `pub`입니다.

```nabi
use bare::ffi::objc;
use bare::ffi::objc::{Id, NSObject, NSString, ns};

#[link(name = "Foundation", kind = "framework")]
#[objc::class(NSArray)]
extern "objc" {
    pub fn array() -> Id<Self>;                       // 클래스 메서드: NSArray::array()
    pub fn count(self) -> usize;                      // 인스턴스 메서드: a.count()
    pub fn objectAtIndex(self, i: usize) -> Id<NSObject>;
}

#[objc::class(NSMutableArray, extends(NSArray))]
extern "objc" {
    pub fn addObject(self, o: Id<NSObject>);
}

#[objc::category(NSString)]
extern "objc" {
    pub fn length(self) -> usize;
}

let a = NSMutableArray::array();
a.addObject(ns("x"));
println(f"{a.count()} {ns("héllo").length()}");      // 1 5
```

`self`가 있는 메서드는 인스턴스 메서드이고, 없는 메서드는 클래스 메서드입니다. 블록의 메서드는 클래스 이름을 딴 제네릭 트레잇 둘이 됩니다. `CMethods<T>`는 인스턴스 메서드를 담고 `T: IsA<C>`인 모든 `Id<T>`에 구현되며, `CClassMethods<T>`는 클래스 메서드를 담고 그런 모든 `T`에 구현됩니다(카테고리의 트레잇은 모듈 안 블록 순서대로 `CMethods2`, `CClassMethods2`, ...). 그래서 클래스는 슈퍼클래스와 따르는 프로토콜의 메서드를, 어디에 선언되었든 물려받습니다. 위에서는 `NSMutableArray::array()`, 그리고 `Id<NSMutableArray>`의 `count`입니다. 메서드를 부르려면 다른 트레잇 메서드처럼 그 트레잇이 스코프에 있어야 합니다. `use ns::*;`는 모듈의 트레잇을 모두 가져옵니다. 조상이 이미 가진 이름으로 클래스가 다시 선언한 메서드(`init`)는 빠집니다. 같은 이름의 트레잇이 둘이면 호출이 모호해지기 때문이며, 조상의 것이 불립니다. 선언 안의 `Self`는 메서드를 부른 클래스이므로 `-> Id<Self>`가 Objective-C의 `instancetype`입니다.

## 1.10.3. 셀렉터

메서드의 셀렉터는 이름에서 `_`마다 한 부분이 끝나고 부분마다 뒤에 `:`가 붙은 것이며, 인자 하나에 부분 하나입니다. `initWithBytes_length_encoding(self, b, n, e)`는 `initWithBytes:length:encoding:`를, `count(self)`는 `count`를 보냅니다. 이름은 Objective-C 이름을 바꾸지 않은 그대로라서 Apple 문서와 일대일로 맞습니다. 이름이 다른 메서드는 `#[selector("isEqualToString:")]`로 셀렉터를 주며, `:`의 개수는 인자 개수와 같아야 합니다. 이름의 부분이 매개변수와 맞지 않으면 두 가지 쓰는 법을 모두 보여 주는 오류입니다.

## 1.10.4. 매개변수와 결과

| 선언한 타입 | Objective-C와 주고받는 형태 |
|-------------|-----------------------------|
| `Id<T>` | 객체(`T *`, nonnull). 매개변수로는 `A: IsA<T>`인 어떤 `A`의 `Id<A>`든 받습니다(서브클래스, 또는 프로토콜 `T`를 따르는 클래스) |
| `Id<T> \| none` | 객체 또는 nil. 매개변수로는 정확히 `Id<T>`(`x.upcast::<T>()`로 만듦)나 `none`을 받습니다 |
| `Id<NSObject>`(매개변수) | 아무 객체: 매개변수는 어떤 클래스 `A`든 받는 `Id<A>`입니다. `Id<NSObject> \| none`은 `x.as_object()`나 `none`을 받습니다 |
| `fn(A, ..) -> R`(매개변수) | 클로저를 부르는 블록(1.10.9) |
| `bool` | `BOOL`(arm64) |
| `Sel`, `Class` | `SEL`, `Class` |
| 그 밖의 타입 | `extern "C"` 선언([1.7](107-ffi.md))과 같습니다. 정수, 부동소수점, 원시 포인터, 값으로 넘기는 `#[repr(C)]` 구조체, NUL로 끝나는 `char *`인 `str` |

`Id<T>`로 선언한 결과가 nil로 돌아오면 panic합니다: "`-[NSArray firstObject]` returned nil, but its declaration returns an object: declare the result `Id<T> | none`". `Id`는 늘 객체를 가지므로 nil 수신자는 생기지 않습니다.

메서드 호출은 안전합니다. 다만 원시 포인터 매개변수(`Ptr`, `ConstPtr`, `ManyPtr`, `CPtr`)가 있거나 `unsafe fn`으로 선언한 메서드는 C 함수처럼 `unsafe`로 불러야 합니다. 선언 자체는 클래스와 맞는지 검사하지 않습니다. 틀린 선언은 틀린 `extern "C"` 선언처럼 정의되지 않은 동작입니다.

가변 인자 메서드, 기본 인자, 제네릭이나 `async` 메서드, `ref self`는 오류입니다.

## 1.10.5. 소유권: `Id<T>`

`Id<T>`는 클래스 `T`(또는 그 하위 클래스)의 Objective-C 객체를 가리키며 그 객체의 retain 하나를 쥡니다. GC 값이라서 `Id`의 사본은 그것을 함께 쓰고, retain 하나도 함께 씁니다.

| 셀렉터의 메서드 패밀리 | 결과 |
|------------------------|------|
| `alloc`, `new`, `copy`, `mutableCopy`, `init`(ARC가 읽듯 셀렉터의 첫 단어), 또는 `#[returns_retained]` | 이미 retain된 것(`+1`): `Id`가 넘겨받습니다 |
| 그 밖, 또는 `#[returns_not_retained]` | autorelease된 것(`+0`): 보낸 직후 `Id`가 `objc_retainAutoreleasedReturnValue`로 retain합니다 |

`init` 메서드는 Objective-C에서처럼 수신자의 retain을 넘겨받습니다. 그 뒤로는 호출한 `Id`와 그 사본이 모두 해제된 것으로 읽힙니다. 그래서 `NSArray::alloc().init()`이 남기는 retain은 결과가 쥔 하나뿐입니다.

| `Id<T>` 메서드 | 뜻 |
|----------------|----|
| `release_now()` | 지금 객체를 해제합니다. 그 뒤로 모든 사본이 해제된 것으로 읽히고, 두 번째 호출은 아무 일도 하지 않습니다 |
| `is_released() -> bool` | `release_now()`가 실행되었는지 |
| `as_raw() -> CPtr<u8>` | 객체 포인터. `release_now()` 뒤에는 panic합니다: "an `Id<NSWindow>` was used after `release_now()`; release an object only once nothing uses it any more" |
| `retained_raw() -> CPtr<u8>` | 그것을 넘겨받는 피호출자(`NS_CONSUMED`)를 위한 retain 하나 더 |
| `unsafe cast<U: Object>() -> Id<U>` | 같은 객체와 retain을 클래스 `U`로, 검사 없이 |
| `downcast<U: Object>() -> Id<U> \| none` | 객체가 `U`일 때(`isKindOfClass:`) 같은 것 |
| `as_object() -> Id<NSObject>` | 같은 객체를 `NSObject`로 |
| `unsafe Id::<T>::from_retained(p)`, `unsafe Id::<T>::retain(p)` | `+1` / `+0` 객체 포인터로 만든 `Id`. nil이면 `none` |

`Id`의 사본이 하나도 남지 않으면 수집기가 객체를 해제합니다. `Id`는 저마다 수집기에 정리 작업(cleanup)을 등록하고, 수집기는 `Id`가 죽었다고 확인한 수집 뒤에 `objc_release`를 실행합니다. 다음에 할당하는 스레드에서, 모든 락 밖에서 실행하며, 정리 작업에는 순서가 없고 끝나는 프로그램은 아무것도 실행하지 않습니다. `release_now()`는 곧바로 해제하고 정리 작업을 취소합니다. `dealloc`이 메인 스레드에서 돌아야 하는 클래스, 곧 `NSResponder`(`NSView`, `NSWindow`, ...: 타입을 선언할 때 슈퍼클래스로 정함, `Object::main_thread()`)의 객체는 메인 스레드에서 해제합니다. 다른 스레드의 정리 작업은 해제를 메인 큐로 보냅니다(`dispatch_async_f`).

`WeakId<T>`(`WeakId::new(id)`)는 객체를 살려 두지 않고 가리킵니다. `load() -> Id<T> | none`은 객체가 해제되면 `none`입니다. 약한 슬롯은 수집기가, 또는 `release_now()`가 곧바로 돌려줍니다. 자기를 쥔 객체를 쥐는 delegate처럼 Objective-C를 거치는 순환을 끊는 데 씁니다.

## 1.10.6. `Sel`, `Class`, `Object`

- `Sel`: 셀렉터. `Sel::register(name)`(`sel_registerName`), `s.name()`.
- `Class`: 클래스 객체. `Class::lookup(name) -> Class | none`(`objc_lookUpClass`), `c.name()`, `c.as_raw()`.
- `trait Object { fn name() -> str; fn class() -> Class; fn main_thread() -> bool; }`: `extern "objc"` 블록이 선언하는 모든 타입과 `NSObject`, `NSString`, `NSError`가 구현합니다. 프로토콜 타입의 `class()`는 panic합니다. 프로토콜에는 클래스 객체가 없습니다. `main_thread()`(기본값 `false`)는 `NSResponder`에서 `true`입니다(1.10.5).
- `trait IsA<C> {}`: `T: IsA<C>`인 `T`는 `C`입니다. `C` 자신, 그 슈퍼클래스, 또는 그것이나 슈퍼클래스가 따르는 프로토콜(프로토콜의 부모 포함)입니다. 블록이 구현하며, `x.upcast::<C>() -> Id<C>`에 필요합니다.
- `objc::new::<T>() -> Id<T>`: `+[T new]`. 클래스 객체에는 `c.new_object() -> Id<NSObject>`.
- `ClassBuilder`: 실행 중에 만드는 클래스(`objc_allocateClassPair`)로, delegate나 API가 서브클래스를 원하는 클래스에 씁니다. `ClassBuilder::new(name, sup) -> ClassBuilder | none`, `unsafe b.method(sel, imp, types)`(`imp`는 `extern "C" fn(this: CPtr<u8>, cmd: Sel, ..)`의 주소, `types`는 그 타입 인코딩 `"v@:@"`), `b.protocol(name)`, 그리고 `b.register() -> Class`.

## 1.10.7. `NSObject`, `NSString`, `str`

`NSObject`의 메서드는 모든 `Id`에 있습니다: `description() -> Id<NSString>`, `hash()`, `isEqual(o)`, `isKindOfClass(c)`, `respondsToSelector(s)`, `class()`, `retainCount()`. `{x}`와 `{x:?}`는 `description`을 보여 줍니다. `extern "objc"` 블록이 같은 이름으로 선언한 메서드가 우선합니다.

`NSString::from_str(s)`와 그 짧은 형태 `ns(s)`는 `str`을 새 `NSString`으로 복사합니다. 암묵 변환은 없습니다. `s.to_str()`은 텍스트를 다시 복사해 옵니다. UTF-8에 담을 수 없는 짝 없는 UTF-16 서로게이트는 U+FFFD가 됩니다. `NSString`을 쓰는 프로그램은 Foundation을 링크합니다.

## 1.10.8. 오류: `throws`로 쓰는 `NSError **`

`throws (objc::Error)`로 선언한 메서드는 선언보다 Objective-C 인자가 하나 많습니다. 마지막의 `NSError **`이고, 셀렉터는 이것도 셉니다(`contentsOfDirectoryAtPath_error(self, path)`는 `contentsOfDirectoryAtPath:error:`를 보냅니다). 결과는 `BOOL`을 돌려주는 메서드면 없음(`NO`가 오류), 객체를 돌려주는 메서드면 `Id<T>`(nil이 오류)입니다. 다른 결과나 다른 오류 타입은 오류입니다.

```nabi
let names = try fm.contentsOfDirectoryAtPath_error(ns("/nope"))
    catch e: objc::Error { println(f"{e.domain()} {e.code()}: {e}"); NSArray::array() };
```

`objc::Error`에는 `domain() -> str`, `code() -> isize`, `localizedDescription() -> str`(`Display`도 이것), `ns_error() -> Id<NSError> | none`이 있습니다. `Error`를 구현하므로 그냥 `throws`로도 전파됩니다.

Objective-C 예외는 Swift에서처럼 프로그래머 오류이고 잡지 않습니다. 런타임은 첫 `Id`를 만들 때 잡히지 않은 예외의 핸들러(`objc_setUncaughtExceptionHandler`)를 설치합니다. Objective-C 안에서 아무도 잡지 않은 예외는 `Objective-C exception <name>: <reason>`, 호출 스택, 이유가 가리키는 호출을 고치라는 안내를 출력하고 프로그램을 중단(abort)합니다.

## 1.10.9. 블록

클로저 타입 `fn(A, ..) -> R`인 매개변수는 그 클로저를 부르는 Objective-C 블록을 넘깁니다. 블록은 Apple 블록 ABI가 정한 대로 배치됩니다. `isa = _NSConcreteStackBlock`, `BLOCK_HAS_COPY_DISPOSE | BLOCK_HAS_SIGNATURE`, 클로저 타입마다 만드는 트램펄린인 `invoke`, copy·dispose 헬퍼와 타입 인코딩이 든 디스크립터, 그리고 캡처한 워드 하나입니다. 그 워드는 클로저를 담은 고정된(pinned) GC 셀의 주소이고, 고정 횟수는 셀을 쥔 블록의 수입니다. 그래서 Objective-C가 사본을 하나라도 쥐는 동안 클로저가 살아 있습니다. 메서드는 `_Block_copy`로 블록을 만들고, 보낸 뒤 자기 사본을 해제합니다.

클로저의 인자는 1.10.4의 표를 반대 방향으로 따릅니다. 객체 인자는 (retain된) `Id`로 오고, 객체 결과는 autorelease되어 돌아갑니다.

```nabi
let seen: List<str> = [];
a.enumerateObjectsUsingBlock(|o, i, stop| { seen.push(f"{i}:{o}"); });
cmd.addCompletedHandler(|buf| { done.store(buf.status() as i64); });  // Metal 스레드에서
```

블록은 다른 스레드에서 실행될 수 있습니다(완료 핸들러). 그러면 클로저는 그 스레드에서, 다른 C 콜백처럼 런타임에 붙어([1.7](107-ffi.md)) 실행되고, 캡처한 것을 함께 씁니다. 스레드에 안전한 값(`Atomic`, `Mutex`, 채널)을 쓰십시오. 블록을 받는 블록 매개변수와 블록 결과는 지원하지 않습니다.

`Block<F>`는 Nabi가 소유한 블록으로, 블록을 받는 C 함수(`dispatch_block_t`)에 씁니다. `Block::new(f: fn() + Send)`는 어느 스레드에서든 실행되고, `Block::new_main_thread(f: fn())`는 `Send`가 아니어도 되는 클로저를 받으며 메인 스레드 밖에서 실행되면 panic합니다. `b.as_raw()`는 C 매개변수로 넘길 블록이고, `b.call()`은 그것을 부르며, 블록은 수집기가 해제합니다(`_Block_release`). `BlockRef<F>`는 Objective-C가 넘겨준 블록입니다. 그 호출이 도는 동안 유효하고, `r.call()`로 부르며, `r.copy() -> Block<F>`로 간직합니다(`_Block_copy`).

`Queue::main()`과 `Queue::global()`은 Grand Central Dispatch 큐이고, `dispatch_async(queue, block)`는 `Block<fn()>`을 나중에 그 큐에서 실행합니다. 메인 큐의 블록은 메인 런 루프(`NSApplication::run`, `dispatch_main`)에서 실행됩니다.

```nabi
dispatch_async(Queue::global(), Block::new(|| { total.fetch_add(1); }));
dispatch_async(Queue::main(), Block::new_main_thread(|| { window.performClose(none); }));
```

## 1.10.10. Autorelease pool

`objc::autoreleasepool(|| ..)`는 autorelease pool(`objc_autoreleasePoolPush` / `Pop`) 안에서 클로저를 실행하고 그 값을 돌려줍니다. 그동안 Objective-C가 autorelease한 것은 돌아올 때 해제됩니다. 모든 `Id`는 자기 retain을 쥐므로 pool이 `Id`가 아직 쥔 객체를 가져가는 일은 없습니다. 메서드를 많이 부르는 루프를 감싸십시오.

## 1.10.11. 낮추기(lowering)

- 메서드는 각각 C 시그니처 `R (*)(id, SEL, A..)`(던지는 메서드는 끝에 `NSError **` 하나 더)로 선언한 `objc_msgSend`의 `extern "C"` 선언을 부르는 Nabi 함수입니다. 그래서 결과 포인터로 돌려받는 구조체 결과까지 포함해 타깃의 C ABI가 인자와 결과를 분류합니다. arm64에는 모든 반환 타입을 위한 `objc_msgSend` 하나만 있습니다. (x86_64의 `objc_msgSend_stret` / `_fpret`는 필요 없습니다. Nabi에는 x86_64 Apple 타깃이 없습니다.)
- 문자열 리터럴을 받는 `sel_registerName("..")` / `objc_getRequiredClass("..")` 호출은 Mach-O에서 정적 참조를 읽는 코드로 컴파일됩니다. `__TEXT,__objc_methname`에 이름이, `__DATA,__objc_selrefs`(`S_LITERAL_POINTERS`, dead strip 안 함)에 그것을 가리키는 포인터가 있고, 런타임이 로드할 때 등록된 셀렉터로 바꿉니다. `__DATA,__objc_classrefs`에는 `_OBJC_CLASS_$_C`를 가리키는 포인터가 있고, dyld가 프레임워크의 `.tbd`(`objc-classes`)에서 바인딩합니다. 이 둘 중 하나가 있는 오브젝트에는 `__objc_imageinfo`(버전 0, 플래그 `0x40`)도 있으며, 링커는 이미지마다 하나를 남깁니다. 만들어지는 메서드는 이 호출을 쓰므로 실행 중에 셀렉터나 클래스를 찾는 일은 없습니다.

## 1.10.12. 헤더: `ffi::objc::import`

```nabi
mod ns {
    ffi::objc::import { "Foundation/Foundation.h", "AppKit/AppKit.h" }
}
use ns::*;
```

`ffi::objc::import { "Framework/Header.h", .. }`는 toolchain의 C 컴파일러([9.13](../../refs-lang/009-tools/913-c-compiler.md))가 가진 Objective-C 프런트엔드로 SDK 헤더를 읽고, 그 바인딩(`extern "objc"` 블록과 C 선언)을 이 항목이 있는 모듈에 넣습니다. 이름은 프레임워크 헤더(`Metal/Metal.h`: SDK의 `Metal.framework/Headers/Metal.h`), SDK `usr/include` 아래의 헤더, 또는 가져오는 파일 기준의 상대 경로입니다. 어디에도 없는 헤더는 써야 할 형태를 보여 주는 오류입니다. Apple 타깃이 아니면 `extern "objc"` 블록과 같은 도움말이 붙은 오류입니다.

바인딩 대상은 지정한 헤더의 디렉터리에 선언된 모든 클래스, 프로토콜, 카테고리, 그곳의 모든 C 선언([1.7.6](107-ffi.md#176-자동-바인딩-생성)의 방식대로), 그리고 이들이 다른 곳에서 가져다 쓰는 클래스와 프로토콜(메서드 없이)입니다. 프로그램이 메서드를 부르는 프레임워크는 모두 `ffi::objc::import` 하나에서 가져오십시오. 두 import가 같은 클래스를 바인딩하면 그렇게 하라고 알려 주는 오류입니다. 모듈은 `Id`, `NSObject`, `NSString`, `NSError`도 받으므로 `use ns::*`는 바인딩이 쓰는 이름을 모두 가져옵니다.

| Objective-C | Nabi |
|-------------|------|
| `@interface C : S <P, Q>` | `#[objc::class(C, extends(S), conforms(P, Q))]`. 그 카테고리와 확장도 같은 블록에 |
| `@interface NSString (Cat)`(`NSObject`, `NSError`도) | `#[objc::category(NSString)]`: 클래스는 `bare::ffi::objc`가 선언 |
| `@protocol P <Q, R>` | `#[objc::protocol(P, extends(Q), conforms(R))]`. 필수 인스턴스 메서드. 클래스와 이름이 같은 프로토콜은 `PProtocol`(`NSObjectProtocol`) |
| `- (R)a:(A)x b:(B)y` | `fn a_b(self, x: A, y: B) -> R`. 이름으로 셀렉터를 되살릴 수 없으면(`_`가 든 부분, Nabi 키워드, 이미 쓰인 이름) `#[selector("..")]` |
| `@property (readonly, getter=isX) T x` | 게터 `isX()`, `readonly`가 아니면 `setX(v)`도. `class` 프로퍼티는 클래스 메서드 |
| `T * _Nonnull`, `T * _Nullable`, 지정 없음 | `Id<T>`, `Id<T> \| none`, `Id<T> \| none`. `NS_ASSUME_NONNULL_BEGIN/END` 안의 단일 포인터는 nonnull. 루트 클래스의 `alloc`, `new`, `init`, `copy`, `mutableCopy`는 객체를 돌려줌 |
| `id`, `id<P>`, `instancetype`, `__kindof C *` | `Id<NSObject>`, `Id<P>`, `Id<Self>`, `Id<C>` |
| `Class`, `SEL`, `BOOL` | `Class`, `Sel`, `bool` |
| `NSArray<NSString *> *`, 클래스의 타입 매개변수 `ObjectType` | `Id<NSArray>`, `Id<NSObject>`: 경량 제네릭 인자는 버림 |
| 블록 `void (^)(A)` 매개변수 | `fn(A)`(1.10.9). 블록 결과는 주석을 남기고 건너뜀 |
| `BOOL`이나 객체 결과와 마지막 `NSError **` | `throws (objc::Error)`(1.10.8) |
| `NS_ENUM(NSInteger, E)`, `NS_CLOSED_ENUM` | `Copy`, `Eq`와 값마다 `const`를 가진 `#[repr(transparent)] pub struct E(pub isize)` |
| `NS_OPTIONS(NSUInteger, O)` | 같은 구조체에 `\|`, `&`, `o.contains(flags)` |
| `NS_RETURNS_RETAINED`, `NS_RETURNS_NOT_RETAINED` | `#[returns_retained]`, `#[returns_not_retained]` |
| 11.0보다 새로운 `API_AVAILABLE(macos(13.0))` | `#[available(macos = "13.0")]` |
| `NS_UNAVAILABLE`, `API_UNAVAILABLE(macos)`, `@optional` 메서드, 가변 인자 메서드, `dealloc` / `retain` / `release` / `autorelease` | 바인딩하지 않음 |
| 객체 매개변수나 결과가 있는 C 함수(`MTLCreateSystemDefaultDevice`) | `extern "C"` 선언 위에서 `Id`를 주고받는 함수(`NS_RETURNS_RETAINED`면 `+1`, 아니면 retain) |
| `extern NSString *const NSFoo` | `fn NSFoo() -> Id<NSString>` |

Nabi 형태가 없는 타입을 쓰는 메서드는 블록 안의 `// skipped -[C sel]: why` 줄이 됩니다. 프런트엔드는 `@class`, `@protocol`, `@interface`(카테고리, 클래스 확장, ivar), 메서드, 프로퍼티, `@optional` / `@required`, `@compatibility_alias`, 소유권 한정자, nullability(`_Nonnull`과 그 친척, 문맥 단어 `nullable` / `nonnull` / `null_unspecified` / `null_resettable`, `NS_ASSUME_NONNULL`), 분산(variance)과 경계가 있는 경량 제네릭, `__kindof`, 블록 타입, 가용성·소유권·열거형 속성을 읽습니다. `__OBJC__`, `__OBJC2__`, ARC의 기능(`__has_feature(objc_arc)`), Interface Builder 표시를 미리 정의하고, `#import`는 파일마다 한 번만 읽습니다. `@implementation`이나 메서드 본문이 있는 헤더는 그것을 `.m` 파일로 옮기라는 오류입니다.

### 캐시

바인딩은 toolchain 캐시의 `objc/<SDK version>/<key>.nabi`에 남습니다. 키는 SDK, 그 버전, 타깃, 배포 타깃, 지정한 헤더, 컴파일러입니다. 다음 빌드는 헤더 대신 그 파일을 읽습니다. 그 파일이 바인딩의 소스이므로 바인딩 안의 진단은 그 파일을 가리킵니다. `nabi bindgen -x objective-c <header>`는 헤더 하나의 같은 바인딩을 파일로 씁니다([9.10.8](../../refs-lang/009-tools/910-nabi-frontend.md#nabi-bindgen-c-헤더-바인딩)).

## 1.10.13. 가용성과 배포 타깃

배포 타깃은 프로그램이 돌아가는 가장 오래된 macOS입니다. 기본값은 `11.0`이고, `nabi.nabion`의 `deployment_target: "13.0"`([9.10](../../refs-lang/009-tools/910-nabi-frontend.md))이나 그것보다 우선하는 `MACOSX_DEPLOYMENT_TARGET`으로 정합니다. `#[available(macos = "13.0")]`이 붙은 함수나 메서드(바인딩은 11.0보다 새로운 API에 이것을 붙입니다)는 macOS 13.0이 보장되는 곳에서만 부를 수 있습니다. 배포 타깃이 그 이상일 때, 또는 `if objc::available(macos: "13.0") { .. }`의 블록 안입니다. 그 밖에서는 호출이 오류입니다.

```
error[E0084]: `maximumConcurrentCompilationTaskCount` needs macOS 13.3, the deployment target is 11.0:
  wrap it in `if objc::available(macos: "13.3") { .. }`, or raise the deployment target
  (`deployment_target: "13.3"` in nabi.nabion, or `MACOSX_DEPLOYMENT_TARGET=13.3`)
```

iOS(`aarch64-apple-ios-sim`, [0.6.1](../../refs-lang/000-meta/006-platform-targets.md#061-지원함), 진행 중인 `aarch64-apple-ios`)에서는 버전이 iOS의 버전입니다. 배포 타깃은 기본값이 `13.4`이고, `nabi.nabion`의 `deployment_target: { ios: "16.0" }`이나 `IPHONEOS_DEPLOYMENT_TARGET`으로 정합니다. 바인딩은 SDK 헤더의 iOS 가용성을 읽어 13.4보다 새로운 API에 `#[available(ios = "16.4")]`을 붙이고, `if objc::available(ios: "16.4") { .. }` 밖의 호출은 iOS의 버전, 검사, 변수로 쓴 위와 같은 오류입니다. 각 플랫폼의 검사는 자기 표시와 인자만 읽습니다. `#[available(macos = ..)]` 표시와 `macos:` 인자는 macOS에서, `ios` 쪽은 iOS에서 셈하므로, `if objc::available(macos: "13.3", ios: "16.4") { .. }` 하나로 둘 다 됩니다.

`objc::available(macos: str = "", ios: str = "") -> bool`은 실행 중의 검사로, 돌고 있는 macOS(`kern.osproductversion`)나 iOS(시뮬레이터의 런타임에도 답하는 dyld의 `_availability_version_check`)가 그 버전 이상인지 알려 줍니다. 비워 둔 버전(기본값)은 그 OS의 어느 버전이든 되고, 버전 하나만 주면 macOS의 버전입니다. 컴파일러는 문자열 리터럴로 그 호출을 쓴 `if` 조건을 읽습니다. 다른 형태는 실행 중에만 검사합니다.

## 1.10.14. Swift 전용 API

Swift에만 있는 API(SwiftUI, Combine, StoreKit 2의 Swift 전용 부분)는 SDK에 Objective-C 헤더 없이 `.swiftinterface`만 있어 가져올 수 없습니다. 사용자의 Xcode 프로젝트로 빌드해 네이티브 라이브러리로 링크하는, `@objc`나 `@_cdecl`을 쓴 Swift shim으로 닿을 수 있습니다.
