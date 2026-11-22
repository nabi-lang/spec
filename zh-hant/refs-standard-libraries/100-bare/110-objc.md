# 1.10. Objective-C：`extern "objc"` 與 `ffi::objc`

Objective-C 物件可以直接使用：方法呼叫就是在該方法簽章的 C ABI 下呼叫 `objc_msgSend`（[1.7](107-ffi.md)），選擇器（selector）和類別則是靜態參照，由執行環境在映像載入時修正。過程中沒有 C 編譯器，也沒有 shim。類別及其方法宣告在 `extern "objc"` 區塊中（1.10.2），可以手寫，也可以由 `ffi::objc::import` 從 SDK 的標頭檔讀取（1.10.12）；`bare::ffi::objc` 存放每個程式共用的部分：`Id<T>`、`Sel`、`Class`、`IsA`、根類別 `NSObject`、`NSString`、`NSError`、autorelease pool、block、dispatch 佇列，以及執行時建立的類別。

## 1.10.1. 目標與連結

Objective-C 只存在於 Apple 目標上（`aarch64-apple-darwin`、`aarch64-apple-ios-sim`，以及進行中的 `aarch64-apple-ios`），`bare::ffi::objc` 也是如此。在其他任何目標上，`extern "objc"` 區塊都是編譯錯誤（E0084），其說明會指示把該區塊和使用它的程式碼放到 `#[cfg(any(target_os = "macos", target_os = "ios"))]` 之下。

呼叫 Objective-C 執行環境的程式（呼叫 `extern "objc"` 區塊的方法，或 `bare::ffi::objc` 的函式）不需另外要求就會連結 libobjc。framework 的連結方式與任何函式庫相同，在區塊（或 `extern "C"` 區塊）上加上 `#[link(name = "Foundation", kind = "framework")]`。

## 1.10.2. `extern "objc"` 區塊

```
objc_block  ::= objc_attr "extern" "\"objc\"" "{" method* "}"
objc_attr   ::= "#[objc::class(" ident ("," "extends(" ident ")")? ("," "conforms(" idents ")")? ")]"
              | "#[objc::protocol(" ident ("," "extends(" ident ")")? ("," "conforms(" idents ")")? ")]"
              | "#[objc::category(" ident ")]"
method      ::= attribute* visibility? "unsafe"? "fn" ident "(" ("self" ("," params)? | params?) ")"
                ("throws" "(" type ")")? ("->" type)? ";"
```

每個區塊以三種屬性之一，恰好指定一個類別：

| 屬性 | 宣告的內容 |
|-----------|----------|
| `#[objc::class(C)]`, `#[objc::class(C, extends(S), conforms(P, Q))]` | 類別 `C`，其父類別為 `S`（省略時為 `NSObject`），並遵循協定 `P` 和 `Q`：一個型別 `C`、它的 `Object`、它及其所屬種類的 `IsA`、它的方法，以及 `Id<C>::as_super() -> Id<S>` |
| `#[objc::protocol(P)]`, `#[objc::protocol(P, extends(Q), conforms(R))]` | 一個型別 `P`，代表未具名但遵循協定 `P` 的類別的物件（`MTLDevice`），其父協定為 `Q` 和 `R`；只有實例方法 |
| `#[objc::category(C)]` | 類別 `C` 的更多方法，該類別由另一個區塊或 `bare::ffi::objc`（`NSObject`、`NSString`、`NSError`）宣告 |

缺少上述屬性的區塊、重複宣告的類別、`extends`、`conforms` 或 category 指向未宣告的類別或協定，以及區塊中的 `static`，都是錯誤，錯誤訊息會指出應寫的屬性。區塊產生的每個型別都是 `pub`。

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

帶有 `self` 的方法是實例方法；沒有 `self` 的是類別方法。區塊的方法構成兩個以類別命名的泛型 trait：`CMethods<T>` 存放實例方法，並為每個 `T: IsA<C>` 的 `Id<T>` 實作；`CClassMethods<T>` 存放類別方法，並為每個這樣的 `T` 實作（category 的 trait 依模組中區塊的順序命名為 `CMethods2`、`CClassMethods2`……）。因此一個類別會得到其父類別及其所遵循協定的方法，不論它們宣告在哪裡：上例中，`Id<NSMutableArray>` 上有 `NSMutableArray::array()` 和 `count`。和任何 trait 方法一樣，呼叫方法需要其 trait 在作用域中：`use ns::*;` 會帶入一個模組的所有 trait。如果類別宣告的方法與祖先已有的方法同名（`init`），該方法會被省略，因為同名的第二個 trait 會讓呼叫產生歧義；實際呼叫的是祖先的方法。宣告中的 `Self` 是被呼叫方法的那個類別，所以 `-> Id<Self>` 就是 Objective-C 的 `instancetype`。

## 1.10.3. 選擇器

方法的選擇器就是其名稱，每個 `_` 視為一個部分的結尾，每個部分後面加上 `:`，每個引數對應一個部分：`initWithBytes_length_encoding(self, b, n, e)` 傳送 `initWithBytes:length:encoding:`，`count(self)` 傳送 `count`。名稱沿用 Objective-C 的，不重新命名，因此 Apple 的文件可以一對一對應。`#[selector("isEqualToString:")]` 為另取名稱的方法指定選擇器；其中 `:` 的數量必須等於引數數量。名稱的部分與參數不相符時是錯誤，錯誤訊息會同時列出兩種形式。

## 1.10.4. 參數與結果

| 宣告的型別 | 傳給 Objective-C／由 Objective-C 回傳的形式 |
|---------------|-----------------------------------------|
| `Id<T>` | 物件（`T *`，nonnull）；作為參數時，接受任何 `A: IsA<T>` 的 `Id<A>`（子類別，或遵循協定 `T` 的類別） |
| `Id<T> \| none` | 物件或 nil；作為參數時，恰好接受 `Id<T>`（可用 `x.upcast::<T>()` 產生）或 `none` |
| `Id<NSObject>`（參數） | 任何物件：參數是任何類別 `A` 的 `Id<A>`；`Id<NSObject> \| none` 接受 `x.as_object()` 或 `none` |
| `fn(A, ..) -> R`（參數） | 呼叫該閉包的 block（1.10.9） |
| `bool` | `BOOL`（arm64） |
| `Sel`, `Class` | `SEL`, `Class` |
| 其他任何型別 | 與 `extern "C"` 宣告相同（[1.7](107-ffi.md)）：整數、浮點數、原始指標、以值傳遞的 `#[repr(C)]` 結構、以 NUL 結尾的 `char *` 形式的 `str` |

宣告為 `Id<T>` 的結果若回傳 nil，會觸發 panic：「`-[NSArray firstObject]` returned nil, but its declaration returns an object: declare the result `Id<T> | none`」。接收者不可能是 nil：`Id` 一定持有物件。

呼叫方法是安全的，但帶有原始指標參數（`Ptr`、`ConstPtr`、`ManyPtr`、`CPtr`）或宣告為 `unsafe fn` 的方法除外，這類方法和 C 函式一樣必須在 `unsafe` 中呼叫。宣告本身不會與類別比對檢查：錯誤的宣告是未定義行為，與錯誤的 `extern "C"` 宣告相同。

可變引數方法、預設引數、泛型和 `async` 方法，以及 `ref self`，都是錯誤。

## 1.10.5. 所有權：`Id<T>`

`Id<T>` 是對類別 `T`（或其子類別）的 Objective-C 物件的參照，並持有該物件的一次 retain。它是 GC 值：`Id` 的副本共用它，也共用那一次 retain。

| 選擇器的方法家族 | 結果 |
|-------------------------------|------------|
| `alloc`、`new`、`copy`、`mutableCopy`、`init`（選擇器的第一個單字，與 ARC 的判讀方式相同），或 `#[returns_retained]` | 已 retain（`+1`）：由 `Id` 接管 |
| 其他任何方法，或 `#[returns_not_retained]` | 已 autorelease（`+0`）：傳送後 `Id` 立即以 `objc_retainAutoreleasedReturnValue` retain 它 |

`init` 方法會接管其接收者的 retain，與 Objective-C 相同：之後，被呼叫的那個 `Id` 及其所有副本都會讀作已釋放。因此 `NSArray::alloc().init()` 只留下一次 retain，由結果持有。

| `Id<T>` 方法 | 意義 |
|----------------|---------|
| `release_now()` | 立即釋放物件；之後所有副本都讀作已釋放，第二次呼叫不做任何事 |
| `is_released() -> bool` | `release_now()` 是否已執行 |
| `as_raw() -> CPtr<u8>` | 物件指標；在 `release_now()` 之後觸發 panic：「an `Id<NSWindow>` was used after `release_now()`; release an object only once nothing uses it any more」 |
| `retained_raw() -> CPtr<u8>` | 多一次 retain，給會接管它的被呼叫端（`NS_CONSUMED`） |
| `unsafe cast<U: Object>() -> Id<U>` | 同一個物件和 retain，視為類別 `U`，不檢查 |
| `downcast<U: Object>() -> Id<U> \| none` | 同上，但只在物件是 `U` 時（`isKindOfClass:`） |
| `as_object() -> Id<NSObject>` | 同一個物件，視為 `NSObject` |
| `unsafe Id::<T>::from_retained(p)`, `unsafe Id::<T>::retain(p)` | 從 `+1`／`+0` 物件指標建立 `Id`；nil 時為 `none` |

當 `Id` 的副本都不存在時，回收器會釋放物件：每個 `Id` 都向回收器登記一個清理動作，回收器在某次回收發現該 `Id` 已死亡後執行 `objc_release`（在下一個進行配置的執行緒上、所有鎖之外執行；清理動作沒有順序，程式結束時不會執行任何清理）。`release_now()` 會立即釋放並取消該清理動作。`dealloc` 必須在主執行緒上執行的類別的物件，也就是 `NSResponder` 系列（`NSView`、`NSWindow`……，在宣告型別時依父類別判定，即 `Object::main_thread()`），會在主執行緒上釋放：在其他執行緒上的清理動作會把釋放傳送到主佇列（`dispatch_async_f`）。

`WeakId<T>`（`WeakId::new(id)`）參照物件但不讓它存活：物件被 dealloc 後，`load() -> Id<T> | none` 為 `none`。它的 weak 槽位由回收器歸還，或由 `release_now()` 立即歸還。它可以打斷經過 Objective-C 的環狀參照，例如一個 delegate 持有持有它的那個物件。

## 1.10.6. `Sel`、`Class` 與 `Object`

- `Sel`：選擇器。`Sel::register(name)`（`sel_registerName`）、`s.name()`。
- `Class`：類別物件。`Class::lookup(name) -> Class | none`（`objc_lookUpClass`）、`c.name()`、`c.as_raw()`。
- `trait Object { fn name() -> str; fn class() -> Class; fn main_thread() -> bool; }`：由 `extern "objc"` 區塊宣告的每個型別，以及 `NSObject`、`NSString`、`NSError` 實作。協定型別的 `class()` 會觸發 panic：協定沒有類別物件。`main_thread()`（預設為 `false`）對 `NSResponder` 系列為 `true`（1.10.5）。
- `trait IsA<C> {}`：`T: IsA<C>` 表示 `T` 是一種 `C`：`C` 本身、它的某個父類別，或它或某個父類別所遵循的協定（連同這些協定的父協定）。由區塊實作；`x.upcast::<C>() -> Id<C>` 需要它。
- `objc::new::<T>() -> Id<T>`：`+[T new]`；對類別物件則用 `c.new_object() -> Id<NSObject>`。
- `ClassBuilder`：在執行時建立的類別（`objc_allocateClassPair`），用於 delegate 或 API 要求衍生子類別的類別。`ClassBuilder::new(name, sup) -> ClassBuilder | none`、`unsafe b.method(sel, imp, types)`，其中 `imp` 是 `extern "C" fn(this: CPtr<u8>, cmd: Sel, ..)` 的位址，`types` 是其型別編碼（`"v@:@"`）、`b.protocol(name)`，最後 `b.register() -> Class`。

## 1.10.7. `NSObject`、`NSString` 與 `str`

`NSObject` 的方法存在於每個 `Id` 上：`description() -> Id<NSString>`、`hash()`、`isEqual(o)`、`isKindOfClass(c)`、`respondsToSelector(s)`、`class()`、`retainCount()`。`{x}` 和 `{x:?}` 顯示 `description`。`extern "objc"` 區塊宣告的同名方法優先。

`NSString::from_str(s)` 及其簡寫 `ns(s)` 會把 `str` 複製到新的 `NSString`；沒有隱含轉換。`s.to_str()` 把文字複製回來；UTF-8 無法容納的不成對 UTF-16 代理字元會變成 U+FFFD。使用 `NSString` 的程式會連結 Foundation。

## 1.10.8. 錯誤：以 `throws` 表示 `NSError **`

宣告為 `throws (objc::Error)` 的方法，在 Objective-C 端比宣告多一個引數：最後一個 `NSError **`，選擇器會計入它（`contentsOfDirectoryAtPath_error(self, path)` 傳送 `contentsOfDirectoryAtPath:error:`）。其結果要麼為空，對應回傳 `BOOL` 的方法（`NO` 表示錯誤），要麼是物件 `Id<T>`（nil 表示錯誤）。其他任何結果或其他錯誤型別都是錯誤。

```nabi
let names = try fm.contentsOfDirectoryAtPath_error(ns("/nope"))
    catch e: objc::Error { println(f"{e.domain()} {e.code()}: {e}"); NSArray::array() };
```

`objc::Error` 有 `domain() -> str`、`code() -> isize`、`localizedDescription() -> str`（也是它的 `Display`）和 `ns_error() -> Id<NSError> | none`。它實作 `Error`，因此也能透過一般的 `throws` 傳播。

Objective-C 例外屬於程式設計者的錯誤，不會被攔截，與 Swift 相同。執行環境在建立第一個 `Id` 時安裝未攔截例外處理常式（`objc_setUncaughtExceptionHandler`）：Objective-C 中沒有任何地方攔截的例外，會印出 `Objective-C exception <name>: <reason>`、其呼叫堆疊，以及一則提示要修正 reason 所指的呼叫，然後中止。

## 1.10.9. Block

閉包型別 `fn(A, ..) -> R` 的參數會傳遞一個呼叫該閉包的 Objective-C block。block 依照 Apple block ABI 佈局：`isa = _NSConcreteStackBlock`、`BLOCK_HAS_COPY_DISPOSE | BLOCK_HAS_SIGNATURE`、每種閉包型別各產生一個 trampoline 作為 `invoke`、一個帶有 copy 與 dispose 輔助函式和型別編碼的描述元，以及一個擷取的字組。該字組是存放閉包的已固定 GC 儲存格的位址；固定計數等於持有該儲存格的 block 數量，因此只要 Objective-C 保留任何副本，閉包就會存活。方法以 `_Block_copy` 建立 block，並在傳送後釋放自己的副本。

閉包的引數依 1.10.4 的表格跨越邊界，但方向相反：物件引數以 `Id`（已 retain）傳入，物件結果以 autorelease 方式回傳。

```nabi
let seen: List<str> = [];
a.enumerateObjectsUsingBlock(|o, i, stop| { seen.push(f"{i}:{o}"); });
cmd.addCompletedHandler(|buf| { done.store(buf.status() as i64); });  // on a Metal thread
```

block 可能在其他執行緒上執行（completion handler）；此時閉包在該執行緒上執行，與任何 C 回呼一樣附加到執行環境（[1.7](107-ffi.md)），並共用它所擷取的內容：請使用執行緒安全的值（`Atomic`、`Mutex`、通道）。不支援 block 的 block 參數，也不支援 block 結果。

`Block<F>` 是 Nabi 擁有的 block，用於接受 block 的 C 函式（`dispatch_block_t`）：`Block::new(f: fn() + Send)` 可在任何執行緒上執行，`Block::new_main_thread(f: fn())` 接受不必是 `Send` 的閉包，若在主執行緒以外執行則觸發 panic；`b.as_raw()` 是給 C 參數的 block，`b.call()` 呼叫它，回收器負責釋放它（`_Block_release`）。`BlockRef<F>` 是 Objective-C 傳入的 block：在該次呼叫執行期間有效，`r.call()` 呼叫它，`r.copy() -> Block<F>` 保留它（`_Block_copy`）。

`Queue::main()` 和 `Queue::global()` 是 Grand Central Dispatch 佇列，`dispatch_async(queue, block)` 會稍後在其中一個佇列上執行 `Block<fn()>`；主佇列的 block 從主 run loop 執行（`NSApplication::run`、`dispatch_main`）。

```nabi
dispatch_async(Queue::global(), Block::new(|| { total.fetch_add(1); }));
dispatch_async(Queue::main(), Block::new_main_thread(|| { window.performClose(none); }));
```

## 1.10.10. Autorelease pool

`objc::autoreleasepool(|| ..)` 在 autorelease pool（`objc_autoreleasePoolPush` / `Pop`）內執行閉包並回傳其值：期間 Objective-C autorelease 的物件會在它回傳時釋放。每個 `Id` 都持有自己的 retain，所以 pool 永遠不會拿走 `Id` 仍持有的物件。呼叫大量方法的迴圈應當以它包住。

## 1.10.11. 降階

- 每個方法都是一個 Nabi 函式，以 C 簽章 `R (*)(id, SEL, A..)`（會擲回錯誤的方法最後還有一個 `NSError **`）呼叫 `objc_msgSend` 的 `extern "C"` 宣告，因此由目標的 C ABI 分類引數與結果，包括透過結果指標回傳的結構結果。arm64 對所有回傳型別只有一個 `objc_msgSend`。（不需要 x86_64 的 `objc_msgSend_stret` / `_fpret`：Nabi 沒有 x86_64 Apple 目標。）
- 以字串字面值呼叫 `sel_registerName("..")` / `objc_getRequiredClass("..")`，在 Mach-O 上會編譯成載入一個靜態參照：`__TEXT,__objc_methname` 存放名稱，`__DATA,__objc_selrefs`（`S_LITERAL_POINTERS`，不做 dead strip）存放指向它的指標，執行環境在載入時把它換成已登記的選擇器；`__DATA,__objc_classrefs` 存放指向 `_OBJC_CLASS_$_C` 的指標，由 dyld 依 framework 的 `.tbd`（`objc-classes`）繫結。含有其中任一者的目的檔也會有 `__objc_imageinfo`（版本 0，旗標 `0x40`），連結器在每個映像中保留一份。產生的方法使用這些呼叫，因此執行時不會查找任何選擇器或類別。

## 1.10.12. 標頭檔：`ffi::objc::import`

```nabi
mod ns {
    ffi::objc::import { "Foundation/Foundation.h", "AppKit/AppKit.h" }
}
use ns::*;
```

`ffi::objc::import { "Framework/Header.h", .. }` 以工具鏈 C 編譯器（[9.13](../../refs-lang/009-tools/913-c-compiler.md)）的 Objective-C 前端讀取 SDK 的標頭檔，並把它們的繫結以 `extern "objc"` 區塊和 C 宣告的形式放入包含它的模組。名稱可以是 framework 標頭檔（`Metal/Metal.h`：SDK 中的 `Metal.framework/Headers/Metal.h`）、SDK 的 `usr/include` 底下的標頭檔，或相對於匯入檔案的路徑。到處都找不到的標頭檔是錯誤，錯誤訊息會列出應寫的形式。在非 Apple 目標上，匯入是錯誤，說明與 `extern "objc"` 區塊相同。

會繫結的內容：在指定標頭檔所在目錄中宣告的每個類別、協定和 category，其中的每個 C 宣告（依 [1.7.6](107-ffi.md#176-自動產生繫結) 的方式繫結），以及這些內容從其他地方用到的類別和協定（不含方法）。程式會呼叫其方法的每個 framework，都要在同一個 `ffi::objc::import` 中匯入：被兩次匯入繫結的類別是錯誤，錯誤訊息會說明這一點。模組也會得到 `Id`、`NSObject`、`NSString` 和 `NSError`，所以 `use ns::*` 會帶入繫結所用到的所有名稱。

| Objective-C | Nabi |
|-------------|------|
| `@interface C : S <P, Q>` | `#[objc::class(C, extends(S), conforms(P, Q))]`；它的 category 和 extension 在同一個區塊中 |
| `@interface NSString (Cat)`（`NSObject`、`NSError` 亦同） | `#[objc::category(NSString)]`：類別由 `bare::ffi::objc` 宣告 |
| `@protocol P <Q, R>` | `#[objc::protocol(P, extends(Q), conforms(R))]`；必要的實例方法。與類別同名的協定為 `PProtocol`（`NSObjectProtocol`） |
| `- (R)a:(A)x b:(B)y` | `fn a_b(self, x: A, y: B) -> R`；名稱無法還原出選擇器時（某部分含 `_`、是 Nabi 關鍵字、名稱已被占用）加上 `#[selector("..")]` |
| `@property (readonly, getter=isX) T x` | getter `isX()`，以及非 `readonly` 時的 `setX(v)`；`class` 屬性是類別方法 |
| `T * _Nonnull`, `T * _Nullable`, 未指定 | `Id<T>`、`Id<T> \| none`、`Id<T> \| none`；`NS_ASSUME_NONNULL_BEGIN/END` 內的單一指標為 nonnull。根類別的 `alloc`、`new`、`init`、`copy`、`mutableCopy` 回傳物件 |
| `id`, `id<P>`, `instancetype`, `__kindof C *` | `Id<NSObject>`, `Id<P>`, `Id<Self>`, `Id<C>` |
| `Class`, `SEL`, `BOOL` | `Class`, `Sel`, `bool` |
| `NSArray<NSString *> *`、類別的型別參數 `ObjectType` | `Id<NSArray>`、`Id<NSObject>`：輕量泛型引數會被捨棄 |
| block `void (^)(A)` 參數 | `fn(A)`（1.10.9）；block 結果會以註解標示略過 |
| 最後一個 `NSError **`，搭配 `BOOL` 或物件結果 | `throws (objc::Error)`（1.10.8） |
| `NS_ENUM(NSInteger, E)`, `NS_CLOSED_ENUM` | `#[repr(transparent)] pub struct E(pub isize)`，具備 `Copy`、`Eq`，每個值一個 `const` |
| `NS_OPTIONS(NSUInteger, O)` | 同樣的結構，另有 `\|`、`&` 和 `o.contains(flags)` |
| `NS_RETURNS_RETAINED`, `NS_RETURNS_NOT_RETAINED` | `#[returns_retained]`, `#[returns_not_retained]` |
| 比 11.0 新的 `API_AVAILABLE(macos(13.0))` | `#[available(macos = "13.0")]` |
| `NS_UNAVAILABLE`、`API_UNAVAILABLE(macos)`、`@optional` 方法、可變引數方法、`dealloc` / `retain` / `release` / `autorelease` | 不繫結 |
| 帶有物件參數或結果的 C 函式（`MTLCreateSystemDefaultDevice`） | 接受並回傳 `Id` 的函式（有 `NS_RETURNS_RETAINED` 時為 `+1`，否則 retain），建立在其 `extern "C"` 宣告之上 |
| `extern NSString *const NSFoo` | `fn NSFoo() -> Id<NSString>` |

型別沒有 Nabi 形式的方法，會在其區塊中成為一行 `// skipped -[C sel]: why`。前端讀取 `@class`、`@protocol`、`@interface`（category、class extension、ivar）、方法、屬性、`@optional` / `@required`、`@compatibility_alias`、所有權限定詞、可空性（`_Nonnull` 及其同類、上下文關鍵字 `nullable` / `nonnull` / `null_unspecified` / `null_resettable`、`NS_ASSUME_NONNULL`）、帶有變異性與界限的輕量泛型、`__kindof`、block 型別，以及可用性、所有權和列舉屬性；它預先定義 `__OBJC__`、`__OBJC2__`、ARC 的功能（`__has_feature(objc_arc)`）和 Interface Builder 的標記，並且每個檔案只讀取一次 `#import`。含有 `@implementation` 或方法主體的標頭檔是錯誤，錯誤訊息會指示把它移到 `.m` 檔案。

### 快取

繫結保存在工具鏈的快取 `objc/<SDK version>/<key>.nabi` 中，以 SDK、其版本、目標、部署目標、指定的標頭檔和編譯器為鍵：之後的建置會讀取該檔案，而不是標頭檔。該檔案就是繫結的原始碼，因此繫結中的診斷會指向它。`nabi bindgen -x objective-c <header>` 會把單一標頭檔的相同繫結寫入檔案（[9.10.8](../../refs-lang/009-tools/910-nabi-frontend.md#nabi-bindgen-c-header-bindings)）。

## 1.10.13. 可用性與部署目標

部署目標是程式可執行的最舊 macOS：預設為 `11.0`，可在 `nabi.nabion`（[9.10](../../refs-lang/009-tools/910-nabi-frontend.md)）中寫 `deployment_target: "13.0"`，或使用優先於它的 `MACOSX_DEPLOYMENT_TARGET`。標示為 `#[available(macos = "13.0")]` 的函式或方法（繫結會這樣標示比 11.0 新的 API），只能在已知 macOS 13.0 的地方呼叫：部署目標至少為該版本時，或在 `if objc::available(macos: "13.0") { .. }` 的區塊內。在其他地方呼叫是錯誤：

```
error[E0084]: `maximumConcurrentCompilationTaskCount` needs macOS 13.3, the deployment target is 11.0:
  wrap it in `if objc::available(macos: "13.3") { .. }`, or raise the deployment target
  (`deployment_target: "13.3"` in nabi.nabion, or `MACOSX_DEPLOYMENT_TARGET=13.3`)
```

在 iOS 上（`aarch64-apple-ios-sim`，[0.6.1](../../refs-lang/000-meta/006-platform-targets.md#061-supported)，以及進行中的 `aarch64-apple-ios`），版本是 iOS 的版本：部署目標預設為 `13.4`，可在 `nabi.nabion` 中寫 `deployment_target: { ios: "16.0" }`，或使用 `IPHONEOS_DEPLOYMENT_TARGET`；繫結讀取 SDK 標頭檔中的 iOS 可用性，把比 13.4 新的 API 標示為 `#[available(ios = "16.4")]`，而在 `if objc::available(ios: "16.4") { .. }` 之外的呼叫，就是上述錯誤，只是換成 iOS 的版本、守衛和變數。每個平台的檢查只讀取自己的標示和引數：`#[available(macos = ..)]` 標示和 `macos:` 引數只在 macOS 上算數，`ios` 的只在 iOS 上算數，所以一個守衛可以同時用於兩者：`if objc::available(macos: "13.3", ios: "16.4") { .. }`。

`objc::available(macos: str = "", ios: str = "") -> bool` 是執行時的檢查：正在執行的 macOS（`kern.osproductversion`）或 iOS（dyld 的 `_availability_version_check`，模擬器的執行環境也由它回答）是否為該版本或更新版本。留空的版本（預設值）表示該作業系統的任何版本，只給一個版本時則是 macOS 的版本。編譯器會讀取寫成該呼叫且引數為字串字面值的 `if` 條件；其他任何形式只在執行時檢查。

## 1.10.14. 僅限 Swift 的 API

只存在於 Swift 中的 API（SwiftUI、Combine、StoreKit 2 中僅限 Swift 的部分）在 SDK 中沒有 Objective-C 標頭檔，只有 `.swiftinterface`，因此無法匯入。由使用者自己的 Xcode 專案建置、以原生函式庫連結、帶有 `@objc` 或 `@_cdecl` 的 Swift shim，可以讓它們變得可用。
