# 2.4. `enum`

`enum` 定義一個值為多個變體之一的型別。它與和型別相似，但不同之處在於它是具名的容器。

## 2.4.1. 基本形式

`enum` 的文法如下：

```
enum_decl   ::= visibility? "enum" identifier generics? where_clause? "{" variant_list? "}"
variant_list ::= variant ("," variant)* ","?
variant     ::= identifier                                  // unit
              | identifier "(" type_expr ("," type_expr)* ","? ")"  // tuple
              | identifier "{" named_field_list? "}"        // struct
              | identifier "=" const_expr                   // discriminant
```

最簡單的形式如下：

```nabi
enum Direction {
    North,
    East,
    South,
    West,
}
```

主體以 `{ ... }` 括起來，變體以逗號分隔。允許尾隨逗號。

## 2.4.2. 四種變體

`enum` 的變體依其容納資料的方式分為四種。

### Unit 變體：沒有資料

最簡單的 unit 變體不容納任何資料。

```nabi
enum Color {
    Red,
    Green,
    Blue,
}
```

它只由一個不帶資料的單純標籤構成。

### 元組變體：位置欄位

元組變體具有位置欄位。

```nabi
enum Shape {
    Circle(f64),
    Rectangle(f64, f64),
}
```

它和 [`struct`](203-struct.md) 的元組 `struct` 一樣具有位置欄位。

### 具名變體：具名欄位

具名變體具有具名欄位。

```nabi
enum Event {
    Click { x: i32; y: i32; },
    Scroll { dy: i32; },
    Resize { w: i32; h: i32; },
}
```

每個變體都有形狀如具名 `struct` 的欄位。欄位以 `;` 結束，與 [`struct`](203-struct.md) 一致。

### 值變體：明確的整數值

最後，值變體直接為變體指定一個整數值。

```nabi
enum Status {
    Ok = 0,
    NotFound = 404,
    InternalError = 500,
}
```

指定的值必須是 `const` 整數運算式。兩個變體不得有相同的值，而每個值都必須能放進標籤型別（[6.2.2](../006-meta-system/602-built-in-attributes.md#622-repr-explicit-memory-representation)）。

這種形式用於 C [ABI](../000-meta/007-glossary.md#abi-application-binary-interface) 相容、對應外部協定、位元旗標等。值變體不能與其他種類的變體混用：只要有任何變體是值變體，整個 `enum` 就必須只由值變體組成。

### 強制轉型為整數

沒有酬載的 `enum`（只有 unit 變體，或只有值變體）可以用 `as` 強制轉型為整數。結果是變體的值，對於 unit 變體則是其標籤（從 0 起算的宣告索引）。

```nabi
let code = Status::NotFound as i32;   // 404
```

## 2.4.3. 混用變體種類

unit、元組與具名變體可以在同一個 `enum` 中自由混用。值變體如上所述只能單獨使用。

下面的範例同時使用三種變體。

```nabi
enum Result {
    Ok(i32),                    // tuple
    Err { code: i32; msg: str; }, // named
    Pending,                    // unit
}
```

## 2.4.4. 可見性

變體沒有自己的可見性；它們跟隨 `enum` 的可見性。`pub enum` 的所有變體都是公開的（[5.6.9](../005-modules/506-visibility.md#569-visibility-of-enum-variants)）。

## 2.4.5. 記憶體配置

預設配置交給編譯器：它可以選擇標籤與酬載的對齊與封裝。這種自由使 [niche 最佳化](207-niche-optimization.md)成為可能。

若需要與 C ABI 相容，可以用 `#[repr(...)]` 屬性強制指定表示方式。其確切意義定義於[內建屬性](../006-meta-system/602-built-in-attributes.md)。

## 2.4.6. 同一模組內不得重複定義

在同一個模組中宣告兩個同名的 `enum` 是編譯錯誤。一個 `enum` 內變體名稱重複也是錯誤。

## 2.4.7. `enum`、和型別與 `union`：預覽

三種形式的比較如下。

| 形式              | 命名                                           | 標籤 | 變體是否共用記憶體？               |
|-------------------|------------------------------------------------|------|------------------------------------|
| `enum`            | 在容器內定義的具名變體                         | 有   | 否（各自的酬載）                   |
| 和型別（`A \| B`） | 變體是既有的型別                              | 有   | 否（行內的標籤 + 酬載）            |
| `union`           | 不帶標籤；同一塊記憶體以多種型別解讀           | 無   | 是（刻意重疊）                     |

詳細差異定義於各容器的文件。

## 2.4.8. 依預期型別使用不帶路徑的變體

變體以其列舉的路徑撰寫（`Ordering::Less`、`Shape::Circle`），或在該位置預期的型別能選出該列舉時，不帶路徑地撰寫。一般查找無法解析的單獨名稱（[5.7.1](../005-modules/507-paths.md#571-starting-identifier-of-a-path)），當預期型別是具有變體 `V` 的列舉 `E`，或是恰好有一個具有變體 `V` 之列舉成員的和型別時，就指稱變體 `V`。這對任何列舉都成立，無論是使用者的還是標準函式庫的，都不需要 `use`。

預期型別來自：

- 帶型別的 `let`；
- 參數（包括方法的參數）以及參數的預設值；
- `return` 與函式的結尾運算式；
- 結構字面值中的結構欄位；
- 元素型別已知的串列或字典字面值中的元素；
- 左側型別為 `E` 的 `==` / `!=` 的右側；
- 在型別為 `E` 之受檢值上的 `match`、`if let` 與 `let` 模式（[3.11](../003-expressions/311-patterns.md)）。

```nabi
enum Shape { Circle, Square, Tri(i32) }

fn area(s: Shape = Square) -> i32 {
    match s { Circle => 1, Square => 2, Tri(n) => n }
}

fn main() {
    let s: Shape = Tri(3);
    let o: Ordering = Less;
    let n = Atomic::new(0);
    n.store(1, SeqCst);                     // StoreOrdering::SeqCst
    let lt = 1.cmp(2) == Less;
    println(f"{area()} {area(Circle)} {lt}");
}
```

區域變數、項目與 `use` 優先：預期型別只是對其他東西都無法解析之名稱的後備，因此 core 列舉之後新增的變體絕不會取代使用者的名稱。泛型列舉的實例是預期型別的實例（在 `-> Result<i32, str>` 函式中的 `Ok(1)` 是 `Result<i32, str>::Ok`）。

沒有預期型別時，不帶路徑的變體是錯誤，錯誤訊息會以其最短路徑指出它：

```nabi
let o = Less;   // ERROR: `Less` is not in scope: it is the variant `Ordering::Less`
                // help: write `Ordering::Less`
```
