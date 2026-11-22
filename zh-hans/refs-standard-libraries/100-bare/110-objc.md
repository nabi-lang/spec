# 1.10. Objective-C：`extern "objc"` 与 `ffi::objc`

Objective-C 对象可以直接使用：方法调用就是按方法签名的 C ABI 调用 `objc_msgSend`（[1.7](107-ffi.md)），选择器（selector）和类是运行时在映像加载时修正的静态引用。整个过程不需要 C 编译器，也不需要垫片（shim）。类及其方法在 `extern "objc"` 块中声明（1.10.2），可以手写，也可以由 `ffi::objc::import` 从 SDK 的头文件读取（1.10.12）；`bare::ffi::objc` 包含每个程序共用的部分：`Id<T>`、`Sel`、`Class`、`IsA`、根类 `NSObject`、`NSString`、`NSError`、自动释放池、块、调度队列和运行时创建的类。

## 1.10.1. 目标与链接

Objective-C 只存在于 Apple 目标上（`aarch64-apple-darwin`、`aarch64-apple-ios-sim`，以及进行中的 `aarch64-apple-ios`），`bare::ffi::objc` 也是如此。在其他目标上，`extern "objc"` 块是编译错误（E0084），其帮助信息会提示把该块以及使用它的代码放在 `#[cfg(any(target_os = "macos", target_os = "ios"))]` 之下。

调用 Objective-C 运行时（`extern "objc"` 块的方法或 `bare::ffi::objc` 的函数）的程序会自动链接 libobjc。框架与其他库一样链接，在块（或 `extern "C"` 块）上写 `#[link(name = "Foundation", kind = "framework")]`。

## 1.10.2. `extern "objc"` 块

```
objc_block  ::= objc_attr "extern" "\"objc\"" "{" method* "}"
objc_attr   ::= "#[objc::class(" ident ("," "extends(" ident ")")? ("," "conforms(" idents ")")? ")]"
              | "#[objc::protocol(" ident ("," "extends(" ident ")")? ("," "conforms(" idents ")")? ")]"
              | "#[objc::category(" ident ")]"
method      ::= attribute* visibility? "unsafe"? "fn" ident "(" ("self" ("," params)? | params?) ")"
                ("throws" "(" type ")")? ("->" type)? ";"
```

每个块用以下三种属性之一指名恰好一个类：

| 属性 | 声明的内容 |
|-----------|----------|
| `#[objc::class(C)]`、`#[objc::class(C, extends(S), conforms(P, Q))]` | 超类为 `S`（省略时为 `NSObject`）且遵循协议 `P` 和 `Q` 的类 `C`：类型 `C`、它的 `Object`、它及其所属种类的 `IsA`、它的方法，以及 `Id<C>::as_super() -> Id<S>` |
| `#[objc::protocol(P)]`、`#[objc::protocol(P, extends(Q), conforms(R))]` | 类型 `P`，表示未指名但遵循协议 `P` 的类的对象（`MTLDevice`），其父协议为 `Q` 和 `R`；只有实例方法 |
| `#[objc::category(C)]` | 类 `C` 的更多方法，该类由另一个块或 `bare::ffi::objc`（`NSObject`、`NSString`、`NSError`）声明 |

缺少这些属性的块、重复声明的类、`extends`、`conforms` 或类别指名了未声明的类或协议，以及块中的 `static`，都是错误，错误信息会指出应当写的属性。块生成的每个类型都是 `pub`。

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

带 `self` 的方法是实例方法，不带的是类方法。块的方法构成两个以类命名的泛型 trait：`CMethods<T>` 包含实例方法，为每个满足 `T: IsA<C>` 的 `Id<T>` 实现；`CClassMethods<T>` 包含类方法，为每个这样的 `T` 实现（类别的 trait 按模块中块的顺序命名为 `CMethods2`、`CClassMethods2`……）。因此，无论超类和所遵循的协议在哪里声明，类都会获得它们的方法：上例中 `Id<NSMutableArray>` 上有 `NSMutableArray::array()` 和 `count`。与任何 trait 方法一样，调用方法需要它的 trait 在作用域中：`use ns::*;` 会引入一个模块的所有 trait。如果类声明的方法与祖先已有的方法同名（`init`），该方法会被略去，因为同名的第二个 trait 会使调用产生歧义；调用的是祖先的方法。声明中的 `Self` 是调用该方法的类，所以 `-> Id<Self>` 就是 Objective-C 的 `instancetype`。

## 1.10.3. 选择器

方法的选择器由它的名称得到：每个 `_` 读作一个部分的结尾，每个部分后加 `:`，每个实参对应一个部分：`initWithBytes_length_encoding(self, b, n, e)` 发送 `initWithBytes:length:encoding:`，`count(self)` 发送 `count`。名称沿用 Objective-C 的，不做重命名，因此与 Apple 的文档一一对应。`#[selector("isEqualToString:")]` 为另行命名的方法给出选择器；其中 `:` 的个数必须等于实参个数。名称的各部分与参数不匹配是错误，错误信息会同时显示两种形式。

## 1.10.4. 参数与结果

| 声明的类型 | 与 Objective-C 之间传递或返回的形式 |
|---------------|-----------------------------------------|
| `Id<T>` | 对象（`T *`，非空）；作为参数时接受任何 `A: IsA<T>` 的 `Id<A>`（子类，或遵循协议 `T` 的类） |
| `Id<T> \| none` | 对象或 nil；作为参数时只接受 `Id<T>`（可以用 `x.upcast::<T>()` 得到）或 `none` |
| `Id<NSObject>`（参数） | 任何对象：该参数对任何类 `A` 都是 `Id<A>`；`Id<NSObject> \| none` 接受 `x.as_object()` 或 `none` |
| `fn(A, ..) -> R`（参数） | 调用该闭包的块（1.10.9） |
| `bool` | `BOOL`（arm64） |
| `Sel`、`Class` | `SEL`、`Class` |
| 其他类型 | 与 `extern "C"` 声明中相同（[1.7](107-ffi.md)）：整数、浮点数、裸指针、按值传递的 `#[repr(C)]` 结构体，`str` 作为以 NUL 结尾的 `char *` |

声明为 `Id<T>` 的结果如果返回 nil，会触发 panic：“`-[NSArray firstObject]` returned nil, but its declaration returns an object: declare the result `Id<T> | none`”。接收者不可能是 nil：`Id` 总是持有一个对象。

调用方法是安全的，但带有裸指针参数（`Ptr`、`ConstPtr`、`ManyPtr`、`CPtr`）或声明为 `unsafe fn` 的方法除外，它们与 C 函数一样必须在 `unsafe` 中调用。声明本身不会与类核对：错误的声明是未定义行为，与错误的 `extern "C"` 声明一样。

可变参数方法、默认实参、泛型方法和 `async` 方法，以及 `ref self`，都是错误。

## 1.10.5. 所有权：`Id<T>`

`Id<T>` 是对类 `T`（或其子类）的 Objective-C 对象的引用，持有它的一次 retain。它是 GC 值：`Id` 的各个副本共享该对象，也共享这一次 retain。

| 选择器的方法族 | 结果 |
|-------------------------------|------------|
| `alloc`、`new`、`copy`、`mutableCopy`、`init`（按 ARC 的读法，取选择器的第一个词），或 `#[returns_retained]` | 已被 retain（`+1`）：由 `Id` 接管 |
| 其他，或 `#[returns_not_retained]` | 已被 autorelease（`+0`）：`Id` 在发送后立即用 `objc_retainAutoreleasedReturnValue` retain 它 |

与 Objective-C 一样，`init` 方法会接管其接收者的 retain：之后，调用它的那个 `Id` 及其所有副本都读作已释放。因此 `NSArray::alloc().init()` 只留下一次 retain，由结果持有。

| `Id<T>` 的方法 | 含义 |
|----------------|---------|
| `release_now()` | 立即释放对象；之后所有副本都读作已释放，再次调用不做任何事 |
| `is_released() -> bool` | `release_now()` 是否已运行 |
| `as_raw() -> CPtr<u8>` | 对象指针；在 `release_now()` 之后调用会触发 panic：“an `Id<NSWindow>` was used after `release_now()`; release an object only once nothing uses it any more” |
| `retained_raw() -> CPtr<u8>` | 多一次 retain，供接管它的被调用方使用（`NS_CONSUMED`） |
| `unsafe cast<U: Object>() -> Id<U>` | 同一个对象和 retain，视为类 `U`，不做检查 |
| `downcast<U: Object>() -> Id<U> \| none` | 同上，但仅当对象是 `U` 时（`isKindOfClass:`） |
| `as_object() -> Id<NSObject>` | 同一个对象，视为 `NSObject` |
| `unsafe Id::<T>::from_retained(p)`、`unsafe Id::<T>::retain(p)` | 由 `+1` / `+0` 对象指针得到的 `Id`；nil 时为 `none` |

当 `Id` 的副本一个都不剩时，回收器会释放该对象：每个 `Id` 都向回收器注册一个清理操作，回收发现该 `Id` 已死后，回收器运行 `objc_release`（在下一个进行分配的线程上，在所有锁之外；清理操作没有顺序，程序退出时一个也不运行）。`release_now()` 立即释放对象并取消该清理操作。`dealloc` 必须在主线程上运行的类的对象，即 `NSResponder` 系列（`NSView`、`NSWindow` 等，声明类型时按超类选定，`Object::main_thread()`），会在主线程上释放：在其他线程上运行的清理操作会把释放发送到主队列（`dispatch_async_f`）。

`WeakId<T>`（`WeakId::new(id)`）引用一个对象但不使它保持存活：对象被 dealloc 后，`load() -> Id<T> | none` 为 `none`。它的弱引用槽由回收器归还，或由 `release_now()` 立即归还。它可以打破经由 Objective-C 的循环，例如 delegate 持有了持有它的对象。

## 1.10.6. `Sel`、`Class` 与 `Object`

- `Sel`：选择器。`Sel::register(name)`（`sel_registerName`），`s.name()`。
- `Class`：类对象。`Class::lookup(name) -> Class | none`（`objc_lookUpClass`），`c.name()`，`c.as_raw()`。
- `trait Object { fn name() -> str; fn class() -> Class; fn main_thread() -> bool; }`：由 `extern "objc"` 块声明的每个类型以及 `NSObject`、`NSString`、`NSError` 实现。协议类型的 `class()` 会触发 panic：协议没有类对象。`main_thread()`（默认为 `false`）对 `NSResponder` 系列为 `true`（1.10.5）。
- `trait IsA<C> {}`：`T: IsA<C>` 表示 `T` 是一个 `C`：即 `C` 本身、它的某个超类，或它或其超类遵循的协议（包括协议的父协议）。各个块实现它；`x.upcast::<C>() -> Id<C>` 需要它。
- `objc::new::<T>() -> Id<T>`：`+[T new]`；对类对象使用 `c.new_object() -> Id<NSObject>`。
- `ClassBuilder`：在运行时创建的类（`objc_allocateClassPair`），用于 delegate 或 API 要求子类化的类。`ClassBuilder::new(name, sup) -> ClassBuilder | none`，`unsafe b.method(sel, imp, types)`，其中 `imp` 是 `extern "C" fn(this: CPtr<u8>, cmd: Sel, ..)` 的地址，`types` 是它的类型编码（`"v@:@"`），`b.protocol(name)`，最后 `b.register() -> Class`。

## 1.10.7. `NSObject`、`NSString` 与 `str`

`NSObject` 的方法在每个 `Id` 上都可用：`description() -> Id<NSString>`、`hash()`、`isEqual(o)`、`isKindOfClass(c)`、`respondsToSelector(s)`、`class()`、`retainCount()`。`{x}` 和 `{x:?}` 显示 `description`。`extern "objc"` 块声明的同名方法优先。

`NSString::from_str(s)` 及其简写 `ns(s)` 把 `str` 复制到新的 `NSString` 中；不存在隐式转换。`s.to_str()` 把文本复制回来；UTF-8 无法容纳的未配对 UTF-16 代理项会变成 U+FFFD。使用 `NSString` 的程序会链接 Foundation。

## 1.10.8. 错误：作为 `throws` 的 `NSError **`

声明为 `throws (objc::Error)` 的方法比它声明的多一个 Objective-C 实参：最后一个 `NSError **`，选择器会把它计入（`contentsOfDirectoryAtPath_error(self, path)` 发送 `contentsOfDirectoryAtPath:error:`）。它的结果要么为空，对应返回 `BOOL` 的方法（`NO` 表示错误），要么是对象 `Id<T>`（nil 表示错误）。其他结果或其他错误类型都是错误。

```nabi
let names = try fm.contentsOfDirectoryAtPath_error(ns("/nope"))
    catch e: objc::Error { println(f"{e.domain()} {e.code()}: {e}"); NSArray::array() };
```

`objc::Error` 有 `domain() -> str`、`code() -> isize`、`localizedDescription() -> str`（也是它的 `Display`）和 `ns_error() -> Id<NSError> | none`。它实现了 `Error`，所以也可以经由普通的 `throws` 传播。

Objective-C 异常属于程序员错误，与 Swift 一样不会被捕获。运行时在创建第一个 `Id` 时安装未捕获异常处理器（`objc_setUncaughtExceptionHandler`）：Objective-C 中无人捕获的异常会打印 `Objective-C exception <name>: <reason>`、它的调用栈，以及一条提示修正原因所指调用的说明，然后中止程序。

## 1.10.9. 块

闭包类型 `fn(A, ..) -> R` 的参数传递一个调用该闭包的 Objective-C 块。块按 Apple 块 ABI 的规定布局：`isa = _NSConcreteStackBlock`，`BLOCK_HAS_COPY_DISPOSE | BLOCK_HAS_SIGNATURE`，按闭包类型生成的跳板函数作为 `invoke`，带有复制和销毁辅助函数及类型编码的描述符，以及一个捕获字。该字是持有闭包的、被固定的 GC 单元的地址；固定计数等于持有该单元的块的数量，因此只要 Objective-C 保留着任何副本，闭包就一直存活。方法用 `_Block_copy` 创建块，并在发送后释放自己的副本。

闭包的实参按 1.10.4 的表格反方向传递：对象实参以 `Id`（已 retain）的形式到达，对象结果以 autorelease 的形式返回。

```nabi
let seen: List<str> = [];
a.enumerateObjectsUsingBlock(|o, i, stop| { seen.push(f"{i}:{o}"); });
cmd.addCompletedHandler(|buf| { done.store(buf.status() as i64); });  // on a Metal thread
```

块可能在另一个线程上运行（完成处理程序）；这时闭包在那里运行，与任何 C 回调一样附加到运行时（[1.7](107-ffi.md)），并共享它捕获的内容：应当使用线程安全的值（`Atomic`、`Mutex`、通道）。不支持块的块参数，也不支持块结果。

`Block<F>` 是 Nabi 拥有的块，用于接受块的 C 函数（`dispatch_block_t`）：`Block::new(f: fn() + Send)` 可在任何线程上运行；`Block::new_main_thread(f: fn())` 接受不必是 `Send` 的闭包，在主线程之外运行时会触发 panic；`b.as_raw()` 是传给 C 参数的块，`b.call()` 调用它，回收器负责释放它（`_Block_release`）。`BlockRef<F>` 是 Objective-C 传入的块：在该次调用期间有效，`r.call()` 调用它，`r.copy() -> Block<F>` 保留它（`_Block_copy`）。

`Queue::main()` 和 `Queue::global()` 是 Grand Central Dispatch 队列，`dispatch_async(queue, block)` 稍后在其中一个队列上运行 `Block<fn()>`；主队列上的块由主运行循环（`NSApplication::run`、`dispatch_main`）运行。

```nabi
dispatch_async(Queue::global(), Block::new(|| { total.fetch_add(1); }));
dispatch_async(Queue::main(), Block::new_main_thread(|| { window.performClose(none); }));
```

## 1.10.10. 自动释放池

`objc::autoreleasepool(|| ..)` 在自动释放池（`objc_autoreleasePoolPush` / `Pop`）内运行闭包并返回它的值：期间 Objective-C autorelease 的对象在返回时释放。每个 `Id` 都持有自己的 retain，所以池永远不会拿走 `Id` 仍持有的对象。调用大量方法的循环应当用它包裹。

## 1.10.11. 降级

- 每个方法都是一个 Nabi 函数，它调用 `objc_msgSend` 的 `extern "C"` 声明，C 签名为 `R (*)(id, SEL, A..)`（会抛出错误的方法末尾还有一个 `NSError **`），因此由目标的 C ABI 对实参和结果分类，包括经由结果指针返回的结构体结果。arm64 对所有返回类型只有一个 `objc_msgSend`。（不需要 x86_64 的 `objc_msgSend_stret` / `_fpret`：Nabi 没有 x86_64 的 Apple 目标。）
- 以字符串字面量调用 `sel_registerName("..")` / `objc_getRequiredClass("..")`，在 Mach-O 上会编译为对静态引用的加载：`__TEXT,__objc_methname` 存放名称，`__DATA,__objc_selrefs`（`S_LITERAL_POINTERS`，不做死代码剥离）存放指向名称的指针，运行时在加载时把它替换为已注册的选择器；`__DATA,__objc_classrefs` 存放指向 `_OBJC_CLASS_$_C` 的指针，由 dyld 依据框架的 `.tbd`（`objc-classes`）绑定。含有其中任一项的目标文件还带有 `__objc_imageinfo`（版本 0，标志 `0x40`），链接器为每个映像保留一份。生成的方法使用这些调用，因此运行时不会查找任何选择器或类。

## 1.10.12. 头文件：`ffi::objc::import`

```nabi
mod ns {
    ffi::objc::import { "Foundation/Foundation.h", "AppKit/AppKit.h" }
}
use ns::*;
```

`ffi::objc::import { "Framework/Header.h", .. }` 用工具链 C 编译器的 Objective-C 前端（[9.13](../../refs-lang/009-tools/913-c-compiler.md)）读取 SDK 的头文件，并把它们的绑定以 `extern "objc"` 块和 C 声明的形式放进包含它的模块。名称可以是框架头文件（`Metal/Metal.h`：SDK 中的 `Metal.framework/Headers/Metal.h`）、SDK 的 `usr/include` 下的头文件，或相对于导入文件的路径。到处都找不到的头文件是错误，错误信息会显示应当写的形式。在 Apple 以外的目标上导入是错误，帮助信息与 `extern "objc"` 块相同。

绑定的内容：所指名头文件所在目录中声明的每个类、协议和类别，那里的每个 C 声明（按 [1.7.6](107-ffi.md#176-自动生成绑定) 的方式绑定），以及它们从别处用到的类和协议（不含方法）。程序调用其方法的所有框架都应当在同一个 `ffi::objc::import` 中导入：被两次导入绑定的类是错误，错误信息会说明这一点。模块还会获得 `Id`、`NSObject`、`NSString` 和 `NSError`，所以 `use ns::*` 会引入绑定中提到的所有名称。

| Objective-C | Nabi |
|-------------|------|
| `@interface C : S <P, Q>` | `#[objc::class(C, extends(S), conforms(P, Q))]`；它的类别和扩展放在同一个块中 |
| `@interface NSString (Cat)`（`NSObject`、`NSError` 也一样） | `#[objc::category(NSString)]`：类由 `bare::ffi::objc` 声明 |
| `@protocol P <Q, R>` | `#[objc::protocol(P, extends(Q), conforms(R))]`；必需的实例方法。与类同名的协议命名为 `PProtocol`（`NSObjectProtocol`） |
| `- (R)a:(A)x b:(B)y` | `fn a_b(self, x: A, y: B) -> R`；名称无法还原出选择器时（某部分含 `_`、是 Nabi 关键字、名称已被占用）使用 `#[selector("..")]` |
| `@property (readonly, getter=isX) T x` | getter `isX()`，非 `readonly` 时还有 `setX(v)`；`class` 属性是类方法 |
| `T * _Nonnull`、`T * _Nullable`、未指定 | `Id<T>`、`Id<T> \| none`、`Id<T> \| none`；`NS_ASSUME_NONNULL_BEGIN/END` 内的单个指针是非空的。根类的 `alloc`、`new`、`init`、`copy`、`mutableCopy` 返回对象 |
| `id`、`id<P>`、`instancetype`、`__kindof C *` | `Id<NSObject>`、`Id<P>`、`Id<Self>`、`Id<C>` |
| `Class`、`SEL`、`BOOL` | `Class`、`Sel`、`bool` |
| `NSArray<NSString *> *`，类的类型参数 `ObjectType` | `Id<NSArray>`、`Id<NSObject>`：轻量泛型实参被丢弃 |
| 块 `void (^)(A)` 参数 | `fn(A)`（1.10.9）；块结果会被跳过并留下注释 |
| 带 `BOOL` 或对象结果的末尾 `NSError **` | `throws (objc::Error)`（1.10.8） |
| `NS_ENUM(NSInteger, E)`、`NS_CLOSED_ENUM` | `#[repr(transparent)] pub struct E(pub isize)`，带 `Copy`、`Eq`，每个值一个 `const` |
| `NS_OPTIONS(NSUInteger, O)` | 同样的结构体，带 `\|`、`&` 和 `o.contains(flags)` |
| `NS_RETURNS_RETAINED`、`NS_RETURNS_NOT_RETAINED` | `#[returns_retained]`、`#[returns_not_retained]` |
| 比 11.0 新的 `API_AVAILABLE(macos(13.0))` | `#[available(macos = "13.0")]` |
| `NS_UNAVAILABLE`、`API_UNAVAILABLE(macos)`、`@optional` 方法、可变参数方法、`dealloc` / `retain` / `release` / `autorelease` | 不绑定 |
| 带对象参数或对象结果的 C 函数（`MTLCreateSystemDefaultDevice`） | 基于其 `extern "C"` 声明、接受并返回 `Id` 的函数（带 `NS_RETURNS_RETAINED` 时为 `+1`，否则会 retain） |
| `extern NSString *const NSFoo` | `fn NSFoo() -> Id<NSString>` |

类型没有 Nabi 形式的方法在其块中是一行 `// skipped -[C sel]: why`。前端读取 `@class`、`@protocol`、`@interface`（类别、类扩展、实例变量）、方法、属性、`@optional` / `@required`、`@compatibility_alias`、所有权限定符、可空性（`_Nonnull` 及其同类、上下文关键字 `nullable` / `nonnull` / `null_unspecified` / `null_resettable`、`NS_ASSUME_NONNULL`）、带型变和约束的轻量泛型、`__kindof`、块类型，以及可用性、所有权和枚举属性；它预定义 `__OBJC__`、`__OBJC2__`、ARC 的特性（`__has_feature(objc_arc)`）和 Interface Builder 的标记，并且每个文件只读取一次 `#import`。含有 `@implementation` 或方法体的头文件是错误，错误信息会提示把它移到 `.m` 文件中。

### 缓存

绑定保存在工具链的缓存 `objc/<SDK version>/<key>.nabi` 中，以 SDK、其版本、目标、部署目标、所指名的头文件和编译器为键：之后的构建读取该文件而不是头文件。该文件就是绑定的源代码，所以绑定中的诊断会指向它。`nabi bindgen -x objective-c <header>` 把单个头文件的相同绑定写入文件（[9.10.8](../../refs-lang/009-tools/910-nabi-frontend.md#nabi-bindgenc-头文件绑定)）。

## 1.10.13. 可用性与部署目标

部署目标是程序可以运行的最旧的 macOS：默认为 `11.0`，可在 `nabi.nabion` 中用 `deployment_target: "13.0"` 设置（[9.10](../../refs-lang/009-tools/910-nabi-frontend.md)），或用优先于它的 `MACOSX_DEPLOYMENT_TARGET` 设置。标记为 `#[available(macos = "13.0")]` 的函数或方法（绑定会这样标记比 11.0 新的 API）只能在已知 macOS 13.0 可用的地方调用：部署目标至少为该版本时，或在 `if objc::available(macos: "13.0") { .. }` 的块内。在其他地方调用是错误：

```
error[E0084]: `maximumConcurrentCompilationTaskCount` needs macOS 13.3, the deployment target is 11.0:
  wrap it in `if objc::available(macos: "13.3") { .. }`, or raise the deployment target
  (`deployment_target: "13.3"` in nabi.nabion, or `MACOSX_DEPLOYMENT_TARGET=13.3`)
```

在 iOS 上（`aarch64-apple-ios-sim`，[0.6.1](../../refs-lang/000-meta/006-platform-targets.md#061-已支持)，以及进行中的 `aarch64-apple-ios`），版本是 iOS 的版本：部署目标默认为 `13.4`，可在 `nabi.nabion` 中用 `deployment_target: { ios: "16.0" }` 设置，或用 `IPHONEOS_DEPLOYMENT_TARGET` 设置；绑定读取 SDK 头文件中的 iOS 可用性，把比 13.4 新的 API 标记为 `#[available(ios = "16.4")]`，在 `if objc::available(ios: "16.4") { .. }` 之外调用会得到上面的错误，只是换成 iOS 的版本、守卫和变量。每个平台的检查只读取自己的标记和实参：在 macOS 上看 `#[available(macos = ..)]` 标记和 `macos:` 实参，在 iOS 上看 `ios` 的，所以一个守卫可以同时用于两者：`if objc::available(macos: "13.3", ios: "16.4") { .. }`。

`objc::available(macos: str = "", ios: str = "") -> bool` 是运行时检查：正在运行的 macOS（`kern.osproductversion`）或 iOS（dyld 的 `_availability_version_check`，它也能回答模拟器运行时的版本）是否为该版本或更新。留空的版本（默认）表示该操作系统的任何版本，只给一个版本时表示 macOS 的版本。编译器会读取写成该调用且实参为字符串字面量的 `if` 条件；其他形式只在运行时检查。

## 1.10.14. 仅限 Swift 的 API

只存在于 Swift 中的 API（SwiftUI、Combine、StoreKit 2 中仅限 Swift 的部分）在 SDK 中没有 Objective-C 头文件，只有 `.swiftinterface`，因此无法导入。可以用带 `@objc` 或 `@_cdecl` 的 Swift 垫片，由用户自己的 Xcode 项目构建并作为原生库链接，使它们可以访问。
