# 1.10. Objective-C - `extern "objc"` and `ffi::objc`

Objective-C objects are used directly: a method call is a call of `objc_msgSend` under the C ABI of the method's signature ([1.7](107-ffi.md)), with the selector and the class as static references the runtime fixes up when the image loads. No C compiler and no shim take part. Classes and their methods are declared in `extern "objc"` blocks (1.10.2), written by hand or read from the SDK's headers by `ffi::objc::import` (1.10.12); `bare::ffi::objc` holds what every program shares: `Id<T>`, `Sel`, `Class`, `IsA`, the root class `NSObject`, `NSString`, `NSError`, autorelease pools, blocks, dispatch queues and classes made at run time.

## 1.10.1. Targets and linking

Objective-C exists only on Apple targets (`aarch64-apple-darwin`, `aarch64-apple-ios-sim`, and `aarch64-apple-ios`, in progress), and so does `bare::ffi::objc`. On any other target an `extern "objc"` block is a compile error (E0084) whose help says to put the block, and the code using it, under `#[cfg(any(target_os = "macos", target_os = "ios"))]`.

A program that calls the Objective-C runtime (a method of an `extern "objc"` block, or a function of `bare::ffi::objc`) links libobjc without being asked. A framework is linked as any library is, with `#[link(name = "Foundation", kind = "framework")]` on the block (or on an `extern "C"` block).

## 1.10.2. `extern "objc"` blocks

```
objc_block  ::= objc_attr "extern" "\"objc\"" "{" method* "}"
objc_attr   ::= "#[objc::class(" ident ("," "extends(" ident ")")? ("," "conforms(" idents ")")? ")]"
              | "#[objc::protocol(" ident ("," "extends(" ident ")")? ("," "conforms(" idents ")")? ")]"
              | "#[objc::category(" ident ")]"
method      ::= attribute* visibility? "unsafe"? "fn" ident "(" ("self" ("," params)? | params?) ")"
                ("throws" "(" type ")")? ("->" type)? ";"
```

Each block names exactly one class with one of three attributes:

| Attribute | Declares |
|-----------|----------|
| `#[objc::class(C)]`, `#[objc::class(C, extends(S), conforms(P, Q))]` | class `C` with superclass `S` (`NSObject` when omitted) that conforms to protocols `P` and `Q`: a type `C`, `Object` for it, `IsA` for it and what it is a kind of, its methods, and `Id<C>::as_super() -> Id<S>` |
| `#[objc::protocol(P)]`, `#[objc::protocol(P, extends(Q), conforms(R))]` | a type `P` for objects of classes that are not named but conform to protocol `P` (`MTLDevice`), whose parent protocols are `Q` and `R`; instance methods only |
| `#[objc::category(C)]` | more methods of class `C`, declared by another block or by `bare::ffi::objc` (`NSObject`, `NSString`, `NSError`) |

A block without one of them, a class declared twice, an `extends`, a `conforms` or a category naming an undeclared class or protocol, and a `static` in the block are errors that name the attribute to write. Every type the block generates is `pub`.

```nabi
use bare::ffi::objc;
use bare::ffi::objc::{Id, NSObject, NSString, ns};

#[link(name = "Foundation", kind = "framework")]
#[objc::class(NSArray)]
extern "objc" {
    pub fn array() -> Id<Self>;                       // class method: NSArray::array()
    pub fn count(self) -> usize;                      // instance method: a.count()
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

A method with `self` is an instance method; one without is a class method. A block's methods are two generic traits, named after the class: `CMethods<T>` holds the instance methods and is implemented for every `Id<T>` whose `T: IsA<C>`, `CClassMethods<T>` holds the class methods and is implemented for every such `T` (a category's traits are `CMethods2`, `CClassMethods2`, ... in the order of the module's blocks). So a class gets the methods of its superclasses and of the protocols it conforms to, wherever those are declared: above, `NSMutableArray::array()` and `count` on an `Id<NSMutableArray>`. Calling a method needs its trait in scope, as for any trait method: `use ns::*;` brings every trait of a module. A method a class declares with the name of one an ancestor already has (`init`) is left out, since a second trait with the name would make the call ambiguous; the ancestor's is called. `Self` in a declaration is the class the method is called on, so `-> Id<Self>` is Objective-C's `instancetype`.

## 1.10.3. Selectors

A method's selector is its name with each `_` read as the end of a part and a `:` after every part, one part per argument: `initWithBytes_length_encoding(self, b, n, e)` sends `initWithBytes:length:encoding:`, `count(self)` sends `count`. The names are Objective-C's, not renamed, so Apple's documentation maps one to one. `#[selector("isEqualToString:")]` gives the selector of a method named otherwise; its `:` count must equal the arguments. A name whose parts do not match its parameters is an error that shows both forms.

## 1.10.4. Parameters and results

| Declared type | Passed to / returned by Objective-C as |
|---------------|-----------------------------------------|
| `Id<T>` | the object (`T *`, nonnull); as a parameter it takes an `Id<A>` of any `A: IsA<T>` (a subclass, or a class conforming to protocol `T`) |
| `Id<T> \| none` | the object or nil; as a parameter it takes exactly an `Id<T>` (`x.upcast::<T>()` makes one) or `none` |
| `Id<NSObject>` (parameter) | any object: the parameter is `Id<A>` for any class `A`; `Id<NSObject> \| none` takes `x.as_object()` or `none` |
| `fn(A, ..) -> R` (parameter) | a block that calls the closure (1.10.9) |
| `bool` | `BOOL` (arm64) |
| `Sel`, `Class` | `SEL`, `Class` |
| anything else | as in an `extern "C"` declaration ([1.7](107-ffi.md)): integers, floats, raw pointers, `#[repr(C)]` structs by value, `str` as a NUL-terminated `char *` |

A result declared `Id<T>` that comes back nil panics: "`-[NSArray firstObject]` returned nil, but its declaration returns an object: declare the result `Id<T> | none`". A nil receiver cannot occur: an `Id` always holds an object.

Calling a method is safe, except a method with a raw pointer parameter (`Ptr`, `ConstPtr`, `ManyPtr`, `CPtr`) or one declared `unsafe fn`, which is `unsafe` to call as C functions are. The declarations themselves are not checked against the class: a wrong one is undefined behavior, as a wrong `extern "C"` declaration is.

Variadic methods, default arguments, generic and `async` methods, and `ref self` are errors.

## 1.10.5. Ownership: `Id<T>`

`Id<T>` is a reference to an Objective-C object of class `T` (or a subclass) that holds one retain of it. It is a GC value: copies of an `Id` share it, and with it the one retain.

| Method family of the selector | The result |
|-------------------------------|------------|
| `alloc`, `new`, `copy`, `mutableCopy`, `init` (the first word of the selector, as ARC reads it), or `#[returns_retained]` | already retained (`+1`): the `Id` takes it over |
| any other, or `#[returns_not_retained]` | autoreleased (`+0`): the `Id` retains it with `objc_retainAutoreleasedReturnValue` right after the send |

An `init` method takes over its receiver's retain, as in Objective-C: afterwards the `Id` it was called on, and every copy of it, reads as released. `NSArray::alloc().init()` therefore leaves one retain, held by the result.

| `Id<T>` method | Meaning |
|----------------|---------|
| `release_now()` | release the object now; every copy then reads as released, a second call does nothing |
| `is_released() -> bool` | whether `release_now()` ran |
| `as_raw() -> CPtr<u8>` | the object pointer; panics after `release_now()`: "an `Id<NSWindow>` was used after `release_now()`; release an object only once nothing uses it any more" |
| `retained_raw() -> CPtr<u8>` | one more retain, for a callee that takes it over (`NS_CONSUMED`) |
| `unsafe cast<U: Object>() -> Id<U>` | the same object and retain as class `U`, unchecked |
| `downcast<U: Object>() -> Id<U> \| none` | the same, when the object is a `U` (`isKindOfClass:`) |
| `as_object() -> Id<NSObject>` | the same object as an `NSObject` |
| `unsafe Id::<T>::from_retained(p)`, `unsafe Id::<T>::retain(p)` | an `Id` from a `+1` / `+0` object pointer; `none` for nil |

The collector releases the object once no copy of the `Id` is left: each `Id` registers a cleanup with the collector, which runs `objc_release` after a collection found the `Id` dead (on the thread that next allocates, outside every lock; cleanups have no order and a program that exits runs none). `release_now()` releases it at once and cancels the cleanup. Objects of classes whose `dealloc` must run on the main thread, the `NSResponder`s (`NSView`, `NSWindow`, ..., chosen by superclass when the type is declared, `Object::main_thread()`), are released there: a cleanup on another thread sends the release to the main queue (`dispatch_async_f`).

`WeakId<T>` (`WeakId::new(id)`) refers to an object without keeping it alive: `load() -> Id<T> | none` is `none` once the object is deallocated. Its weak slot is given back by the collector, or at once by `release_now()`. It breaks a cycle through Objective-C, such as a delegate that holds the object that holds it.

## 1.10.6. `Sel`, `Class` and `Object`

- `Sel`: a selector. `Sel::register(name)` (`sel_registerName`), `s.name()`.
- `Class`: a class object. `Class::lookup(name) -> Class | none` (`objc_lookUpClass`), `c.name()`, `c.as_raw()`.
- `trait Object { fn name() -> str; fn class() -> Class; fn main_thread() -> bool; }`: implemented by every type an `extern "objc"` block declares and by `NSObject`, `NSString`, `NSError`. A protocol type's `class()` panics: a protocol has no class object. `main_thread()` (default `false`) is `true` for the `NSResponder`s (1.10.5).
- `trait IsA<C> {}`: a `T: IsA<C>` is a `C`: `C` itself, one of its superclasses, or a protocol it or a superclass conforms to (with the protocols' parents). The blocks implement it; `x.upcast::<C>() -> Id<C>` needs it.
- `objc::new::<T>() -> Id<T>`: `+[T new]`; `c.new_object() -> Id<NSObject>` for a class object.
- `ClassBuilder`: a class made at run time (`objc_allocateClassPair`), for a delegate or a class an API wants subclassed. `ClassBuilder::new(name, sup) -> ClassBuilder | none`, `unsafe b.method(sel, imp, types)` with `imp` the address of an `extern "C" fn(this: CPtr<u8>, cmd: Sel, ..)` and `types` its type encoding (`"v@:@"`), `b.protocol(name)`, then `b.register() -> Class`.

## 1.10.7. `NSObject`, `NSString` and `str`

The methods of `NSObject` are on every `Id`: `description() -> Id<NSString>`, `hash()`, `isEqual(o)`, `isKindOfClass(c)`, `respondsToSelector(s)`, `class()`, `retainCount()`. `{x}` and `{x:?}` show `description`. A method an `extern "objc"` block declares with the same name takes precedence.

`NSString::from_str(s)` and its short form `ns(s)` copy a `str` into a new `NSString`; there is no implicit conversion. `s.to_str()` copies the text back; an unpaired UTF-16 surrogate, which UTF-8 cannot hold, becomes U+FFFD. A program using `NSString` links Foundation.

## 1.10.8. Errors: `NSError **` as `throws`

A method declared `throws (objc::Error)` has one more Objective-C argument than it declares: an `NSError **` last, which the selector counts (`contentsOfDirectoryAtPath_error(self, path)` sends `contentsOfDirectoryAtPath:error:`). Its result is either nothing, for a method returning `BOOL` (`NO` is the error), or an object, `Id<T>` (nil is the error). Any other result, or another error type, is an error.

```nabi
let names = try fm.contentsOfDirectoryAtPath_error(ns("/nope"))
    catch e: objc::Error { println(f"{e.domain()} {e.code()}: {e}"); NSArray::array() };
```

`objc::Error` has `domain() -> str`, `code() -> isize`, `localizedDescription() -> str` (also its `Display`) and `ns_error() -> Id<NSError> | none`. It implements `Error`, so it also propagates through a plain `throws`.

Objective-C exceptions are programmer errors and are not caught, as in Swift. The runtime installs an uncaught exception handler (`objc_setUncaughtExceptionHandler`) when the first `Id` is made: one that nothing in Objective-C catches prints `Objective-C exception <name>: <reason>`, its call stack and a note to fix the call the reason names, then aborts.

## 1.10.9. Blocks

A parameter of closure type `fn(A, ..) -> R` passes an Objective-C block that calls the closure. The block is laid out as the Apple block ABI says: `isa = _NSConcreteStackBlock`, `BLOCK_HAS_COPY_DISPOSE | BLOCK_HAS_SIGNATURE`, a trampoline generated per closure type as `invoke`, a descriptor with copy and dispose helpers and the type encoding, and one captured word. That word is the address of a pinned GC cell holding the closure; the pin count is the number of blocks holding the cell, so the closure lives as long as any copy Objective-C keeps. The method makes the block with `_Block_copy` and releases its own copy after the send.

The closure's arguments cross as the table of 1.10.4 says, in the other direction: an object argument arrives as an `Id` (retained), an object result is returned autoreleased.

```nabi
let seen: List<str> = [];
a.enumerateObjectsUsingBlock(|o, i, stop| { seen.push(f"{i}:{o}"); });
cmd.addCompletedHandler(|buf| { done.store(buf.status() as i64); });  // on a Metal thread
```

A block may run on another thread (a completion handler); the closure then runs there, attached to the runtime as any C callback is ([1.7](107-ffi.md)), and shares what it captured: use thread-safe values (`Atomic`, `Mutex`, channels). A block parameter of a block, and a block result, are not supported.

`Block<F>` is a block Nabi owns, for a C function that takes one (`dispatch_block_t`): `Block::new(f: fn() + Send)` runs on any thread, `Block::new_main_thread(f: fn())` takes a closure that need not be `Send` and panics if it runs off the main thread; `b.as_raw()` is the block for the C parameter, `b.call()` calls it, and the collector releases it (`_Block_release`). `BlockRef<F>` is a block Objective-C passed in: valid while that call runs, `r.call()` calls it, `r.copy() -> Block<F>` keeps it (`_Block_copy`).

`Queue::main()` and `Queue::global()` are Grand Central Dispatch queues, and `dispatch_async(queue, block)` runs a `Block<fn()>` on one later; main-queue blocks run from the main run loop (`NSApplication::run`, `dispatch_main`).

```nabi
dispatch_async(Queue::global(), Block::new(|| { total.fetch_add(1); }));
dispatch_async(Queue::main(), Block::new_main_thread(|| { window.performClose(none); }));
```

## 1.10.10. Autorelease pools

`objc::autoreleasepool(|| ..)` runs the closure inside an autorelease pool (`objc_autoreleasePoolPush` / `Pop`) and returns its value: what Objective-C autoreleases meanwhile is released when it returns. Every `Id` holds its own retain, so a pool never takes an object an `Id` still holds. Wrap loops that call many methods.

## 1.10.11. Lowering

- Each method is a Nabi function calling an `extern "C"` declaration of `objc_msgSend` with the C signature `R (*)(id, SEL, A..)` (and a final `NSError **` for a throwing one), so the target's C ABI classifies arguments and results, struct results through the result pointer included. arm64 has one `objc_msgSend` for every return type. (x86_64's `objc_msgSend_stret` / `_fpret` are not needed: Nabi has no x86_64 Apple target.)
- A call of `sel_registerName("..")` / `objc_getRequiredClass("..")` with a string literal is compiled, on Mach-O, to a load of a static reference: `__TEXT,__objc_methname` holds the name and `__DATA,__objc_selrefs` (`S_LITERAL_POINTERS`, no dead strip) a pointer to it, which the runtime replaces with the registered selector at load; `__DATA,__objc_classrefs` holds a pointer to `_OBJC_CLASS_$_C`, bound by dyld from the framework's `.tbd` (`objc-classes`). An object that has either also has an `__objc_imageinfo` (version 0, flags `0x40`), and the linker keeps one per image. The generated methods use these calls, so no selector or class is looked up at run time.

## 1.10.12. Headers: `ffi::objc::import`

```nabi
mod ns {
    ffi::objc::import { "Foundation/Foundation.h", "AppKit/AppKit.h" }
}
use ns::*;
```

`ffi::objc::import { "Framework/Header.h", .. }` reads the SDK's headers with the Objective-C front end of the toolchain's C compiler ([9.13](../../refs-lang/009-tools/913-c-compiler.md)) and puts their bindings, as `extern "objc"` blocks and C declarations, in the module that holds it. A name is a framework header (`Metal/Metal.h`: `Metal.framework/Headers/Metal.h` in the SDK), a header under the SDK's `usr/include`, or a path relative to the importing file. A header found nowhere is an error that shows the form to write. The import is an error off Apple targets, with the same help as an `extern "objc"` block.

What is bound: every class, protocol and category declared in the named headers' directories, every C declaration there (as [1.7.6](107-ffi.md#176-automatic-binding-generation) binds them), and, without methods, the classes and protocols those use from elsewhere. Import every framework whose methods a program calls in one `ffi::objc::import`: a class bound by two imports is an error that says so. The module also gets `Id`, `NSObject`, `NSString` and `NSError`, so `use ns::*` brings all that the bindings name.

| Objective-C | Nabi |
|-------------|------|
| `@interface C : S <P, Q>` | `#[objc::class(C, extends(S), conforms(P, Q))]`; its categories and extensions in the same block |
| `@interface NSString (Cat)` (also `NSObject`, `NSError`) | `#[objc::category(NSString)]`: `bare::ffi::objc` declares the class |
| `@protocol P <Q, R>` | `#[objc::protocol(P, extends(Q), conforms(R))]`; required instance methods. A protocol named like a class is `PProtocol` (`NSObjectProtocol`) |
| `- (R)a:(A)x b:(B)y` | `fn a_b(self, x: A, y: B) -> R`; `#[selector("..")]` when the name cannot give the selector back (a part with `_`, a Nabi keyword, a name taken) |
| `@property (readonly, getter=isX) T x` | the getter `isX()` and, unless `readonly`, `setX(v)`; `class` properties are class methods |
| `T * _Nonnull`, `T * _Nullable`, unspecified | `Id<T>`, `Id<T> \| none`, `Id<T> \| none`; a single pointer inside `NS_ASSUME_NONNULL_BEGIN/END` is nonnull. The root class's `alloc`, `new`, `init`, `copy`, `mutableCopy` return an object |
| `id`, `id<P>`, `instancetype`, `__kindof C *` | `Id<NSObject>`, `Id<P>`, `Id<Self>`, `Id<C>` |
| `Class`, `SEL`, `BOOL` | `Class`, `Sel`, `bool` |
| `NSArray<NSString *> *`, a class's type parameter `ObjectType` | `Id<NSArray>`, `Id<NSObject>`: lightweight generic arguments are dropped |
| block `void (^)(A)` parameter | `fn(A)` (1.10.9); a block result is skipped with a comment |
| a last `NSError **` with a `BOOL` or object result | `throws (objc::Error)` (1.10.8) |
| `NS_ENUM(NSInteger, E)`, `NS_CLOSED_ENUM` | `#[repr(transparent)] pub struct E(pub isize)` with `Copy`, `Eq` and a `const` per value |
| `NS_OPTIONS(NSUInteger, O)` | the same struct with `\|`, `&` and `o.contains(flags)` |
| `NS_RETURNS_RETAINED`, `NS_RETURNS_NOT_RETAINED` | `#[returns_retained]`, `#[returns_not_retained]` |
| `API_AVAILABLE(macos(13.0))` newer than 11.0 | `#[available(macos = "13.0")]` |
| `NS_UNAVAILABLE`, `API_UNAVAILABLE(macos)`, `@optional` methods, variadic methods, `dealloc` / `retain` / `release` / `autorelease` | not bound |
| a C function with object parameters or result (`MTLCreateSystemDefaultDevice`) | a function taking and returning `Id`s (`+1` with `NS_RETURNS_RETAINED`, else retained), over its `extern "C"` declaration |
| `extern NSString *const NSFoo` | `fn NSFoo() -> Id<NSString>` |

A method whose types have no Nabi form is a `// skipped -[C sel]: why` line in its block. The front end reads `@class`, `@protocol`, `@interface` (categories, class extensions, ivars), methods, properties, `@optional` / `@required`, `@compatibility_alias`, ownership qualifiers, nullability (`_Nonnull` and its kin, the context words `nullable` / `nonnull` / `null_unspecified` / `null_resettable`, `NS_ASSUME_NONNULL`), lightweight generics with variance and bounds, `__kindof`, block types and the availability, ownership and enum attributes; it predefines `__OBJC__`, `__OBJC2__`, ARC's features (`__has_feature(objc_arc)`) and Interface Builder's markers, and reads `#import` once per file. A header with `@implementation` or a method body is an error that says to move it to a `.m` file.

### Cache

The bindings are kept in the toolchain's cache, `objc/<SDK version>/<key>.nabi`, keyed by the SDK, its version, the target, the deployment target, the headers named and the compiler: a later build reads that file instead of the headers. The file is the bindings' source, so a diagnostic in them points into it. `nabi bindgen -x objective-c <header>` writes the same bindings for one header to a file ([9.10.8](../../refs-lang/009-tools/910-nabi-frontend.md#nabi-bindgen-c-header-bindings)).

## 1.10.13. Availability and the deployment target

The deployment target is the oldest macOS a program runs on: `11.0` by default, `deployment_target: "13.0"` in `nabi.nabion` ([9.10](../../refs-lang/009-tools/910-nabi-frontend.md)) or `MACOSX_DEPLOYMENT_TARGET`, which beats it. A function or method marked `#[available(macos = "13.0")]`, as the bindings mark APIs newer than 11.0, may be called only where macOS 13.0 is known: when the deployment target is at least that, or inside the block of `if objc::available(macos: "13.0") { .. }`. Elsewhere the call is an error:

```
error[E0084]: `maximumConcurrentCompilationTaskCount` needs macOS 13.3, the deployment target is 11.0:
  wrap it in `if objc::available(macos: "13.3") { .. }`, or raise the deployment target
  (`deployment_target: "13.3"` in nabi.nabion, or `MACOSX_DEPLOYMENT_TARGET=13.3`)
```

On iOS (`aarch64-apple-ios-sim`, [0.6.1](../../refs-lang/000-meta/006-platform-targets.md#061-supported), and `aarch64-apple-ios`, in progress) the versions are iOS's: the deployment target is `13.4` by default, `deployment_target: { ios: "16.0" }` in `nabi.nabion` or `IPHONEOS_DEPLOYMENT_TARGET`; the bindings read the SDK headers' iOS availability and mark APIs newer than 13.4 `#[available(ios = "16.4")]`, and a call outside `if objc::available(ios: "16.4") { .. }` is the error above with iOS's version, guard and variable. Each platform's checks read only its own marks and arguments: a `#[available(macos = ..)]` mark and a `macos:` argument count on macOS, `ios` ones on iOS, so one guard serves both, `if objc::available(macos: "13.3", ios: "16.4") { .. }`.

`objc::available(macos: str = "", ios: str = "") -> bool` is the check at run time: whether the running macOS (`kern.osproductversion`) or iOS (dyld's `_availability_version_check`, which answers for the simulator's runtime too) is that version or newer. A version left empty, the default, is any version of that OS, and a version alone is macOS's. The compiler reads the condition of an `if` written as that call with string literals; any other form checks at run time only.

## 1.10.14. Swift-only APIs

APIs that exist only in Swift (SwiftUI, Combine, the Swift-only parts of StoreKit 2) have no Objective-C header in the SDK, only a `.swiftinterface`, and cannot be imported. A Swift shim with `@objc` or `@_cdecl`, built by the user's own Xcode project and linked as a native library, makes them reachable.
