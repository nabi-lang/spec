# 2.12. trait 不能作為型別

> [!IMPORTANT]
> 禁止直接在型別運算式位置寫出 trait 名稱。需要實作型別在執行時才決定的值時，請寫 `any Trait`，它的成本在語法中可見（2.12.3）。

## 2.12.1. 禁止的內容

```nabi
let a: Number = 32i32;                          // ERROR: Number is a trait
fn process(x: Display) -> str { ... }            // ERROR
struct Holder { item: Iterator; }                // ERROR
```

如這些範例所示，trait 名稱直接作為變數型別、參數型別、回傳型別或結構欄位型別出現，是編譯錯誤。

## 2.12.2. 理由

這項禁止針對的是**隱藏的**動態分派。直接在型別位置寫出 trait 名稱，乍看之下與一般的靜態型別無法區分，實際上卻會產生執行時分派。那種分派有實際的成本。實作在執行時才選定並呼叫，因此在分支預測失誤時，可能比直接呼叫慢兩倍以上。它需要額外存取分派資料的記憶體，損害快取友善性。最重要的是，它阻礙編譯器的內嵌與最佳化。

為了讓科學與資料程式碼中的核心數值迴圈不會在不知情的情況下承擔這種成本，Nabi 不允許把單獨的 trait 名稱當作型別。執行時多型只存在於寫了 `any` 之處。`any Trait` 之所以被允許，是因為它的動態分派成本透過 `any` 關鍵字在語法中可見。

## 2.12.3. 需要多型時：三種替代方案

需要多型時，Nabi 提供三種替代方案。

### 1. 封閉的候選集合使用和型別

若候選集合事先固定，請使用和型別。

```nabi
let a: i32 | f32 | f64 = 32i32;
fn process(x: i32 | f32) -> str { ... }
```

這適合候選在編譯期就固定的情況。記憶體與分派成本是明確的（見[和型別](206-sum-types.md)），模式比對也能檢查窮盡性。

### 2. 開放的多型使用泛型

若候選是開放的，請使用泛型。

```nabi
fn process<T: Display>(x: T) -> str { ... }
struct Holder<T: Iterator> { item: T; }
```

這適合候選在編譯期不固定，或因呼叫者而異的情況。單態化產生直接呼叫，因此成本接近於零（見[泛型](210-generics.md)），而且可以透過 trait 約束呼叫方法。

### 3. 實作型別在執行時決定時使用 `any Trait`

若必須在同一個槽位或集合中混合不同實作型別的值，請使用受約束的 `any`。

```nabi
let shapes: List<any Shape> = [Circle { r: 1.0 }, Square { side: 2.0 }];
fn report(e: any Error) -> str { f"{e}" }
struct ConfigError { msg: str; cause: any Error | none; }
```

`any Trait` 槽位只接受實作了 `Trait` 之型別的值（在編譯期檢查），並且可以在其值上呼叫 `Trait` 及其父 trait 的方法。呼叫依執行時標籤分派，成本模型與 [`any` 槽位](202-any-slot.md)相同。確切規則定義於 [2.2.10](202-any-slot.md#2210-受約束的-anyany-trait)。

### 4. 函式隱藏的回傳型別使用 `some Trait`

函式可以回傳一個具體型別，而其呼叫者只能透過 trait 看到它：`-> some Iterator<Item = i32>`（[4.1.13](../004-functions/401-function-declaration.md#4113-不透明回傳型別some-trait)）；參數也可以接受一個這樣的型別：`x: some Display`（[4.1.14](../004-functions/401-function-declaration.md#4114-some-參數)）。要把該型別存放在欄位中或在簽章中指稱它，請為它命名：`type Evens = some Iterator<Item = i32>;`（[4.1.15](../004-functions/401-function-declaration.md#4115-具名不透明型別type-x--some-trait)）。分派和泛型一樣是靜態的：`some Tr` 是一個被隱藏的靜態型別，而 `any Tr` 容納任何具有 `Tr` 之型別的值，並在執行時分派。

```nabi
pub fn evens(n: i32) -> some Iterator<Item = i32> { (0..n).filter(|x| x % 2 == 0) }
```

## 2.12.4. 選擇準則

請依下列準則在替代方案中做選擇。

| 情況                                                          | 選擇        |
|---------------------------------------------------------------|-------------|
| 候選是固定的小集合（3-5 個）                                  | 和型別      |
| 候選可由使用者 / 函式庫擴充                                   | 泛型        |
| 函式回傳一個呼叫者不需要指稱的型別                            | `-> some Trait` |
| 該隱藏型別要存放在欄位中或在簽章中指稱                        | `type X = some Trait;` |
| 候選只在執行時決定（例如外掛、異質集合）                      | `any Trait` |
| 必須容納任何東西，沒有約束                                    | `any`       |

在需要以執行時決定之型別實現多型的第三種情況下，請使用 `any Trait`。它會產生動態分派成本，但該成本明確地集中在寫了 `any` 之處。

## 2.12.5. 父 trait 與約束運算式是允許的

需要說清楚的是：被禁止的只是 trait 名稱出現在型別位置。下列全都是允許的。

```nabi
trait Ord: Eq { ... }                            // supertrait declaration
fn f<T: ops::Add + Display>(x: T) { ... }              // trait bound
fn g<T>(x: T) where T: Ord, T: Clone { ... }      // where clause
impl Display for Point { ... }                    // trait position of an impl
```

在這些位置，trait 名稱表達的是約束，而不是型別。

## 2.12.6. 編譯錯誤的形式

當 trait 名稱出現在型別位置時，編譯器在該型別處回報 E0071：

```text
error[E0071]: trait `core::fmt::Display` cannot be used as a type directly
  = help: use `any Display` for a value of any type implementing it, or a generic parameter `<T: Display>`
```

說明列出上述替代方案中的兩個；由具體型別組成的和型別（`SomeType | OtherType`）是第三個。
