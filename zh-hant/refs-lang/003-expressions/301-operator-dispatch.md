# 3.1. 運算子分派

本章定義一般原則：[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)中定義的運算子語彙單元，如何被轉換成 trait 方法呼叫。哪個運算子對應到哪個 trait 的哪個方法，其具體矩陣由標準函式庫定義。

## 3.1.1. 每個運算子都是 trait 方法

如[語言與標準函式庫的邊界](../000-meta/004-language-vs-standard-libraries.md)所確立的，語言提供運算子語法，但行為由標準函式庫的 trait 決定。

```
a + b         →   <typeof(a) as Add<typeof(b)>>::add(a, b)
a - b         →   <typeof(a) as Sub<typeof(b)>>::sub(a, b)
a * b         →   <typeof(a) as Mul<typeof(b)>>::mul(a, b)
a < b         →   <typeof(a) as PartialOrd<typeof(b)>>::lt(a, b)
-a            →   <typeof(a) as Neg>::neg(a)
!a            →   <typeof(a) as Not>::not(a)
```

每個運算子都遵循相同的模式：語言辨識語彙單元，並把它轉換成 trait 方法呼叫。

## 3.1.2. 運算子 → trait 的對應位於標準函式庫

各運算子語彙單元對應到哪個 trait 方法的確切表格，由標準函式庫定義。本文件只負責對應的存在以及分派機制。

標準函式庫透過元系統屬性 `#[infix]` / `#[prefix]` / `#[postfix]` 把對應附加到 trait 方法上。確切的機制定義於 [`#[infix]` / `#[prefix]` / `#[postfix]` 魔法](../006-meta-system/607-operator-magic.md)。

```
trait Add<Rhs = Self> {
    type Output;
    #[infix("+")]
    fn add(self, rhs: Rhs) -> Output;
}

trait Neg {
    type Output;
    #[prefix("-")]
    fn neg(self) -> Output;
}
```

這個範例展示 trait 與方法簽章的形狀；它們的意義來自第 2 章的 trait 規格與第 6 章的魔法規格的結合。第 3 章只定義語言會把語彙單元轉換成如上的呼叫。

## 3.1.3. 不經 vtable 的分派

依照 [trait 不能作為型別](../002-types/212-trait-as-type-banned.md)，運算子呼叫的分派被設計為不經 vtable 間接即可解析。

- 對靜態槽位，trait 方法在編譯期決定，並編譯為直接的函式呼叫。
- 對 `any` 槽位，trait 方法由 [`any` 槽位](../002-types/202-any-slot.md)的執行時標籤檢查決定，然後直接呼叫。成本是一次標籤檢查加上一次直接呼叫。

兩種情況都沒有因 vtable 查找而多出的間接呼叫。讓科學與數值程式碼的熱迴圈免於這種成本，是 [trait 不能作為型別](../002-types/212-trait-as-type-banned.md)的動機，而這條分派規則是它的結果。

### 選擇 impl

二元運算 `a op b` 從左運算元型別的該運算子 trait 的 impl 中選擇。

- 若一個型別有同一個運算子 trait 的多個 impl（`Mul<i64>`、`Mul<f64>` 等），選擇 `Rhs` 與右運算元型別相符的 impl。當泛型 impl 的參數能繫結到該型別時，它就相符（`impl<const N: usize> Call<Array<f64, N>>` 以 `N = 2` 接受 `Array<f64, 2>`）；型別與運算元完全相同的 impl 優先於泛型 impl。當有多個泛型 impl 都接受該運算元時，選擇其約束對所繫結者成立的那一個：在 `impl<D: Distribution> Tilde<D> for S` 與 `impl<D: Distribution> Tilde<Iid<D>> for S` 之下，不是 `Distribution` 的 `Iid<Normal>` 採用第二個。
- 若左側是基本型別而右側是結構，使用為該基本型別宣告的 impl（`2.0 * d` 是 `impl Mul<Duration> for f64`）。
- 省略了有預設值之 trait 引數的 impl（`impl<T> Add for W<T>`），以預設值解讀，其中 `Self` 是該 impl 的型別（`Add<W<T>>`）。無後綴字面值仍開放的右運算元（`let v = W::new(1.0); m * v`），取存在 impl 的那一個數字型別。
- 當 trait 有左運算元的 impl，但沒有任何一個接受右運算元時，錯誤訊息會指出兩個型別以及缺少的 impl（`` operator `*` cannot be applied to `Vec3` and `f64`: there is no `impl ops::Mul<f64> for Vec3` ``），說明則提示應寫什麼：兩側使用同一種元素型別（`Vec3::<f64>::X`），或轉換那個數字（`x as f32`）。

同一條規則不限於運算子。凡是依填入 trait 型別參數之值的型別來選擇 impl 之處都適用：運算子的 `Rhs`、`T::from(x)` / `T::try_from(x)` / `Tr::f(x)` 的引數、在已知型別位置上 `.into()` / `.try_into()` 的接收者，以及當各 impl 在該參數上不同時，方法呼叫 `x.m(v)` 或路徑呼叫 `T::m(x, v)` 的引數。若該值是無後綴字面值，選擇依下列順序進行：

1. 字面值預設型別的 impl（[1.6.3](../001-lexical/106-numeric-literals.md#163-無後綴的字面值)：若它填入某個參數，就是該參數的預設值，否則是 `i64` / `f64`），前提是它存在且該值能放進該型別。
2. 否則，參數型別能容納該值的唯一 impl（對整數而言，是範圍內的整數型別或 `int`；對實數而言，是能有限表示該值的 `f32` / `f64`，或 `float`）。`d * 2` 是 `Duration` 的 `Mul<i64>`。沒有整數 impl 的整數，會像它填入浮點數槽位時一樣（1.6.3），採用能精確容納它的浮點數 impl：在 `Call<f64>` 與 `Call<Vector<f64>>` 之下，`s(1)` 是 `s(1.0)`。
3. 若有多個這樣的 impl，就是要求加上後綴的編譯錯誤。

串列字面值 `[..]` 若有 `List` impl 就採用它，否則採用與字面值巢狀深度相符的 `Array` impl：在 `Call<f64>`、`Call<Vector<f64>>` 與 `Call<Matrix<f64>>` 之下，`s([1.0, 2.0])` 是 `Vector` 的，`s([[1.0], [2.0]])` 是 `Matrix` 的，而泛型的 `Call<Array<f64, N>>` 以深度決定 `N` 而接受兩者。

這條規則只適用於單獨的字面值與帶正負號的字面值（`-1`），不適用於 `1 + 1` 這類運算式。以 `Tr::f(..)` 呼叫的靜態 trait 方法，其 `Self` 由預期型別決定。當沒有任何引數填入 trait 的參數時，呼叫的預期型別會選出回傳型別與之相符的 impl（或約束）；否則呼叫有歧義（[4.9.6](../004-functions/409-method.md#496-自由函式與方法的呼叫形式)）。

## 3.1.4. 沒有對應的運算

若某一對型別不存在 trait 實作，運算子呼叫就是編譯錯誤。

```
let a: i32 = 1i32;
let b: str = "hi";
let c = a + b;          // ERROR: no Add impl for (i32, str)
```

診斷會指出運算子與兩個運算元型別。

```
error[E0054]: operator `+` cannot be applied to `i32` and `str`
```

## 3.1.5. 一元、二元、後綴

運算子依位置分為三種之一。

| 位置                  | 形式     | 對應機制           |
|-----------------------|----------|--------------------|
| 二元（中綴）          | `a OP b` | `#[infix("OP")]`   |
| 前綴一元              | `OP a`   | `#[prefix("OP")]`  |
| 後綴一元              | `a OP`   | `#[postfix("OP")]` |

同一個語彙單元可以出現在多個位置。例如 `-` 既用作二元（`a - b`），也用作前綴一元（`-a`）。位置由語彙單元出現的上下文決定：只有其後有運算元時是前綴；只有其前有運算元時是後綴；兩側都有運算元時是二元。

## 3.1.6. 優先順序與結合性

除了依位置分派之外，運算子還有優先順序與結合性。它們定義於[優先順序與結合性](302-precedence-and-associativity.md)。

優先順序與結合性不屬於語彙單元 → trait 的對應。3.2 的表格在語言層級固定它們；`#[infix("+")]` 之類的註冊只把語彙單元連接到 trait 方法。

## 3.1.7. 使用者定義的運算子

標準函式庫透過 `#[infix]` / `#[prefix]` / `#[postfix]` 賦予基本運算子意義。使用者程式碼可以用相同的機制，為額外的 trait 附加新的對應。

```
#[infix("⊕", like = "+")]          // the new token follows the level of an existing token
fn combine(self, rhs: Rhs) -> Output;
```

同一組（語彙單元、位置、型別配對）的對應必須唯一；衝突是編譯錯誤。

要把不在[標點與運算子語彙單元](../001-lexical/110-punctuation-and-operator-tokens.md)固定語彙單元表中的字元序列註冊為新的運算子，必須透過相同的機制註冊該語彙單元本身。確切的步驟定義於 [`#[infix]` / `#[prefix]` / `#[postfix]` 魔法](../006-meta-system/607-operator-magic.md)。

## 3.1.8. Unicode 運算子

`÷ ≡ ≠ ≤ ≥` 是 `/ == != <= >=` 的另一種拼法：語彙分析器把它們讀作那些語彙單元（[1.10.3](../001-lexical/110-punctuation-and-operator-tokens.md#1103-固定語彙單元表)），因此它們分派到相同的 trait 方法。ASCII 範圍以外的其他運算子（`×`、`⋅`、`∘`、`√`、`∈`、`∉`、`⊂`、`⊆`、`∪`、`∩`）是各自獨立的語彙單元，和任何運算子一樣以 `#[infix]` / `#[prefix]` 繫結到 trait 方法；`×` 是 `ops::Cross`：

```
trait Cross<Rhs = Self> {
    type Output;
    #[infix("×")]
    fn cross(self, rhs: Rhs) -> Output;
}
```

一個方法也可以帶有多個 `#[infix]` 註冊；此時每個語彙單元都對應到同一個方法，行為完全相同。
