# 2.3. `struct`

`struct` 是把多個欄位組成一個的容器語法，用於定義積型別。本章涵蓋 `struct` 可以採取的形式，以及結構的各項規則：建立實例、預設值、展開、可見性與記憶體配置。

## 2.3.1. 三種形式

`struct` 宣告遵循下列文法。

```
struct_decl     ::= "struct" identifier generics? where_clause? "{" named_field_list? "}"
                  | "struct" identifier generics? "(" tuple_field_list? ")" where_clause? ";"   // tuple struct: where comes after the field list
                  | "struct" identifier generics? where_clause? ";"                              // unit struct
named_field_list ::= named_field (";" named_field)* ";"?
named_field     ::= identifier ":" type_expr
tuple_field_list ::= type_expr ("," type_expr)* ","?
```

主體的形狀決定它是三種形式中的哪一種，下文逐一說明。

### 具名 `struct`：具名欄位

最常見的形式是具名 `struct`，其中每個欄位都有名稱。

```nabi
struct Point {
    x: f64;
    y: f64;
}

struct User {
    id: i64;
    name: str;
    age: i32;
}
```

規則：主體以 `{ ... }` 括起來。每個欄位寫成 `name: Type;`，欄位結尾的 `;` 與[陳述式終止符](../001-lexical/103-statement-terminator.md)是同一個終止符。欄位型別是必要的，因為如 [`any` 槽位](202-any-slot.md)所確立的，結構欄位不允許漸進式型別。

### 元組 `struct`：位置欄位

其次，元組 `struct` 只以位置識別欄位，沒有名稱。

```nabi
struct Pair(i32, i32);
struct Wrapper(str);
struct Color(u8, u8, u8);
```

在這種形式中，主體以 `( ... )` 括起來，每個元素只有型別而沒有名稱。欄位以 `.0`、`.1` 等位置索引存取；確切語法在運算式規格中說明。也允許單一欄位的形式 `struct W(T);`，可用於 newtype 模式。

### Unit `struct`：沒有欄位的標記

最後，unit `struct` 完全沒有欄位。

```nabi
struct Marker;
struct Sentinel;
```

這種形式沒有主體，在宣告之後立即以 `;` 結束。它主要用於只有單一實例才有意義的標記型別。

unit 結構可以帶型別參數（`struct Tag<T>;`），作為攜帶型別的標記。它的值要麼寫出引數 `Tag::<i64>`，要麼交給它所放入的槽位決定（`let t: Tag<i64> = Tag;`，或在 `fn f(t: Tag<str>)` 之下的 `f(Tag)`）。兩者都沒有時，`let t = Tag;` 是錯誤，錯誤訊息會要求寫出型別（`let t: Tag<i64> = Tag;`）。

## 2.3.2. 建立實例：預覽

各形式建立實例的語法正式定義於運算式規格。本文件只預覽各形式的樣子。

```nabi
let p = Point { x: 1.0f64, y: 2.0f64 };    // named
let q = Pair(1i32, 2i32);                   // tuple
let m = Marker;                              // unit
let t = Tag::<i64>;                          // generic unit
```

## 2.3.3. 欄位預設值

具名 `struct` 的欄位可以有預設值。

```nabi
struct Config {
    retry: i32 = 3i32;
    timeout: f64 = 30.0f64;
    verbose: bool = false;
}
```

預設值受到下列限制。只允許 `const` 運算式：字面值、`const` 變數、`const` 函式呼叫，以及由這類值構成的 `List` / `Dict` 字面值（`[]`、`[1, 2]`、`{"a": 1}`）。集合型別的預設值在編譯期檢查，並為每個實例重新建立，因此實例之間絕不會共用它。`const` 運算式定義於 [`const`](../005-modules/502-const.md)。需要執行時呼叫的運算式，例如目前時間或隨機數，不能使用；這樣的程式碼是編譯錯誤。參照同一個結構的其他欄位也是禁止的。預設值只在建立實例時省略該欄位才會求值，而且每個實例都會求值一次。由於求值是確定性的，編譯器可以套用快取最佳化。

這些限制確保了確定性、可預測的成本以及 SemVer 安全性。若需要執行時初始化，請使用另外的工廠函式，即 `fn new() -> T` 模式。欄位預設值中的名稱在宣告該結構的模組作用域中解析，而不是在建立值的地方。衍生的 `Default` 也使用宣告的預設值（[stdlib 2.13.10](../../refs-standard-libraries/200-core/213-marker.md#21310-default預設值)）。

在結構所屬模組之外的字面值，只有在私有欄位有預設值時才能省略它（[5.6.8](../005-modules/506-visibility.md#568-struct-欄位的可見性)）。`#[non_exhaustive]` 結構會為第一個版本之後新增的每個欄位提供預設值，讓其他套件中的字面值能繼續編譯（[6.2.10](../006-meta-system/602-built-in-attributes.md#6210-non_exhaustive-退出外部的窮盡性)）。

## 2.3.4. 欄位展開

有時新的實例應當複製既有實例的欄位，只改變其中一些。

```nabi
let base = Point { x: 0.0f64, y: 0.0f64 };
let p = Point { x: 5.0f64, ..base };
```

這裡的 `..base` 表示原樣取用 base 的所有欄位，並以新值覆寫明確寫出的欄位。

在語法上：`..` 語彙單元（[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)）之後接一個實例值。`..` 在建立實例的運算式中只能出現一次；不允許兩個以上的展開。把展開放在明確欄位之後是慣用的風格。寫出的欄位先依欄位順序求值，再求值 base。

## 2.3.5. 可見性

每個欄位都可以用[關鍵字](../001-lexical/105-keywords.md)中定義的可見性關鍵字控制其對外公開程度。

```nabi
struct Account {
    pub id: i64;        // readable from outside
    pub balance: f64;
        secret: str;    // only within the same module (default = module-private)
}
```

元組結構的欄位也各自有自己的可見性（`struct S(pub i32, i32);`，預設為私有）。建構、`x.N` 存取與模式的規則，以及可見性等級的意義，見 [5.6.8](../005-modules/506-visibility.md#568-struct-欄位的可見性)。

## 2.3.6. 記憶體配置

欄位依宣告順序以自然對齊配置，與 `#[repr(C)]` 相同（[6.2.2](../006-meta-system/602-built-in-attributes.md#622-repr-明確的記憶體表示)）。`packed` 與 `align(N)` 會改變這一點。

## 2.3.7. 同一模組內不得重複定義

在同一個模組中宣告兩個同名的結構是編譯錯誤。一個結構內欄位名稱重複也是錯誤。這條規則不論可見性等級都適用。元組結構或 unit 結構的建構子是一個值，位於函式、`const` 與 `static` 的命名空間中（[4.1.10](../004-functions/401-function-declaration.md#4110-同一模組內不得重複定義)）：同一個模組中的 `struct Point(i32, i32);` 與名為 `Point` 的 `fn`、`const` 或 `static` 構成重複定義（`E0026`）。

## 2.3.8. 值語意

`struct` 是值型別。指派、傳遞引數、回傳以及讀取元素（`dict[k]`、`list[i]`、`entry(k).or_insert_with(..)`）都會複製結構。變更只有透過 `ref` 參數或 `ref self`（[4.7](../004-functions/407-ref-keyword.md)），或把值寫回去，才會在其他地方看得到。

```nabi
let a = Point { x: 1.0, y: 2.0 };
let b = a;
b.x = 5.0;                        // a.x unchanged

let ps: List<Point> = [a];
let q = ps[0];
q.x = 9.0;                        // ps[0].x unchanged
ps[0] = q;                        // written back
```

對之後沒有任何東西讀取的複本進行欄位寫入，是編譯錯誤 `E0057`：那個變更會遺失。受檢查的複本包括以值傳遞的參數或 `self`、閉包以值傳遞的參數（`xs.for_each(|p| { p.x = 0.0; })`）、`for` 繫結，以及元素的 `let`（`let q = xs[i];`）。當之後沒有任何讀取、迴圈迭代、回傳或呼叫能看到該變更時，寫入就是錯誤；只用來餵給對同一變數另一次寫入的讀取不算數，而被閉包或協程區塊擷取的變數不受檢查。說明會指出應改寫成什麼：`ref self` 或 `ref` 參數（[4.7](../004-functions/407-ref-keyword.md)）、`for ref`（[3.13.3](../003-expressions/313-loop-expressions.md#for-ref元素作為位置)）、回傳閉包的引數並使用結果，或把元素寫回去。

```nabi
impl Counter {
    fn inc(self) { self.n += 1; }   // error[E0057]: `self.n` is set on a copy of the receiver; write `fn inc(ref self)`
}
for p in ps { p.x = 0.0; }        // error[E0057]: `p.x` is set on a copy of the element; write `for ref p in ps`
for ref p in ps { p.x = 0.0; }    // writes every element of ps
ps.for_each(|p| { p.x = 0.0; });  // error[E0057]: `p.x` is set on a copy of the closure's argument
let qs = ps.map(|p| { p.x = 0.0; p }).to_list();  // returned: kept
let c = self;                     // a local copy on purpose: change `c`, then read or return it
```

共享要透過參照控制代碼：集合型別（`List`、`Dict`、`Set`、`StrBuf`、`Deque`……，[stdlib 2.9](../../refs-standard-libraries/200-core/209-collections.md#參考控制代碼)）、`Mutex`、`RwLock`、atomics、通道，以及 `[ref x]` 擷取儲存格。複製控制代碼會共享物件。

|                         | 值型別（`struct`、`enum`、元組、基本型別）      | 參照控制代碼（`List`、`Dict`、`Mutex`……）     |
|-------------------------|-------------------------------------------------|-----------------------------------------------|
| `let b = a;`            | 獨立的複本                                      | 同一個物件                                    |
| 欄位 / 元素寫入         | 只影響該複本                                    | 透過每個控制代碼都看得到                      |
| 獨立的複本              | 指派本身                                        | `.clone()`                                    |

存放控制代碼的結構欄位複製的是控制代碼：複製該結構會共享那個物件。
