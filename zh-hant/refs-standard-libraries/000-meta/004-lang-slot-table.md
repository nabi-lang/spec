# 0.4. `#[lang]` 槽位對應表

`#[lang(slot_name)]` 屬性（[refs-lang 6.4](../../refs-lang/006-meta-system/604-lang-magic.md)）把標準函式庫的項目登記到編譯器可以查詢的**槽位**中。本章以表格形式，把每一個槽位名稱、其意義以及它所附加的項目集中在一處。

槽位表是規格中的權威位置；編譯器、標準函式庫和使用者都參照它以保持一致。槽位名稱一旦走樣，語言與函式庫之間的契約就會瓦解，因此任何變更都從這張表開始。

## 0.4.1. 槽位分類

100 個槽位依職責分為四組。下表一目了然地列出各組的角色與槽位數，指出後續各節的詳細表格屬於哪一類。

| 組別    | 職責                                                                        | 槽位數  |
|---------|-----------------------------------------------------------------------------|---------|
| 型別    | Nabi 視為**具體型別**的標準函式庫型別                                        | 20      |
| Trait   | Nabi 用於**運算子 / 控制流程去語法糖（desugaring）**、衍生以及執行緒邊界檢查的 trait | 37 |
| 方法    | Nabi **自動呼叫**的方法（drop）                                            | 1       |
| 函式    | Nabi 以**內建函式呼叫**方式呼叫的標準函式庫函式，以及降階後程式碼所呼叫的鉤子 | 42 |
| 合計    |                                                                             | **100** |

## 0.4.2. 型別槽位（20）

型別槽位指出編譯器遇到字面值或關鍵字時要降階成的具體型別。這一組再細分為兩種。第一種是以小寫識別字直接公開的語言預設型別。它們不需要另外登記關鍵字，是使用者每天都會碰到的基本型別。

語言預設（內建基本型別，不需要 `#[keyword]`）：

| 槽位名稱     | 意義                         | 標準函式庫項目           | 參照 |
|--------------|------------------------------|--------------------------|------|
| `i32_type`   | `i32` 基本型別               | 內建（方法：`core::num`） | 2.1  |
| `i64_type`   | `i64` 基本型別               | 內建（方法：`core::num`） | 2.1  |
| `f32_type`   | `f32` 基本型別               | 內建（方法：`core::num`） | 2.1  |
| `f64_type`   | `f64` 基本型別               | 內建（方法：`core::num`） | 2.1  |
| `usize_type` | `usize` 基本型別             | 內建（方法：`core::num`） | 2.1  |
| `isize_type` | `isize` 基本型別             | 內建（方法：`core::num`） | 2.1  |
| `bool_type`  | `bool` 基本型別              | 內建（方法：`core::num`） | 2.1  |
| `cchar_type` | `c_char` 基本型別（FFI）     | 內建（實作：`core::num`） | 2.15 |
| `str_type`   | `str` 基本型別（UTF-8 檢視） | 內建（方法：`core::str`） | 2.15 |
| `char_type`  | `char` 基本型別（Unicode 純量） | 內建（方法：`core::str`） | 2.15 |

第二種是與語言語法綁在一起的槽位：會自動提升的 `int` 和 `float`、儲存用浮點數 `f16` 和 `bf16`、`none` 哨兵值，以及 `Range` 字面值。`int`、`float`、`f16` 和 `bf16` 是 PascalCase 的函式庫結構，透過 `#[keyword(...)]` 公開小寫名稱；`none` 是內建的，函式庫以 `pub type NoneSentinel = none` 為其型別命名為 `NoneSentinel`。

與語法綁在一起的槽位：

| 槽位名稱     | 意義                          | 標準函式庫項目                                            | 參照 |
|--------------|-------------------------------|-----------------------------------------------------------|------|
| `none_type`  | `none` 字面值的型別           | 內建（別名：`core::marker::NoneSentinel`） | 2.13 |
| `int_type`   | `int` 自動提升型別            | `core::num::bigint::Int` + `#[keyword("int")]` | 2.1  |
| `float_type` | `float` 自動提升型別          | `core::num::bigfloat::Float` + `#[keyword("float")]` | 2.1  |
| `f16_type`   | `f16` 儲存用浮點數（IEEE 754 binary16） | `core::num::half::F16` + `#[keyword("f16")]` | 2.1  |
| `bf16_type`  | `bf16` 儲存用浮點數（bfloat16） | `core::num::half::BF16` + `#[keyword("bf16")]` | 2.1  |
| `range_type` | `Range<T>`（`a..b`）           | `core::range::Range` + `#[literal( expr ".." expr )]` | 2.7  |
| `slice_arg_type` | 多部分索引中的切片部分 `a:b:k` | `core::index::SliceArg` | 2.3  |
| `rest_axis_type` | 軸索引中與其他部分並列的 `..` | `core::index::Rest` | 2.3  |
| `new_axis_type`  | 軸索引中的 `none`         | `core::index::NewAxis` | 2.3  |
| `ndarray_type`   | 位於其槽位中的巢狀串列字面值 | `core::array::Array` | 2.9.14 |

內建提供基本型別的編譯器可以不附加 `i32_type` ... `str_type`、`none_type` 和 `char_type`。參考編譯器就是如此：這些槽位名稱會被接受，但函式庫不附加任何項目（`str` 和 `c_char` 不是函式庫結構，`NoneSentinel` 是別名而非 `#[lang]` 項目）。在 0.4.6 查詢其中之一時，編譯器使用其內建型別。

## 0.4.3. trait 槽位（37）

Trait 槽位指出：使用者撰寫的語法被去語法糖為標準函式庫 trait 呼叫時，編譯器要去找哪一個 trait。`for` 迴圈、索引、切片、格式化和 `await` 等核心語法都經由這裡登記的 trait 解析，因此這張表是語言語法與函式庫之間最主要的橋樑。

| 槽位名稱              | 意義                            | 標準函式庫 trait                | 參照 |
|-----------------------|---------------------------------|---------------------------------|------|
| `iterator_trait`      | `for x in xs` 中的逐步推進      | `core::iter::Iterator`     | 2.7  |
| `into_iterator_trait` | `for x in xs` 的開始            | `core::iter::IntoIterator` | 2.7  |
| `try_iterator_trait`  | `try for x in xs` 中的逐步推進  | `core::iter::TryIterator`  | 2.7.13 |
| `index_trait`         | `arr[i]` 讀取                   | `core::index::Index` | 2.3  |
| `index_set_trait`     | `arr[i] = v` 寫入               | `core::index::IndexSet` | 2.3  |
| `slice_trait`         | `s[1:3]` 切片                   | `core::index::Sliceable` | 2.3  |
| `display_trait`       | `f"{x}"` 預設格式               | `core::fmt::Display`       | 2.10 |
| `debug_trait`         | `f"{x:?}"` 偵錯格式             | `core::fmt::Debug`         | 2.10 |
| `future_trait`        | `await expr` 中的逐步推進       | `core::future::Future`     | 2.20 |
| `axis_indexed_trait`  | 以索引部分作為軸引數            | `core::index::AxisIndexed` | 2.3  |
| `broadcastable_trait` | 融合點運算式的運算元            | `core::ops::Broadcastable` | 2.6.6 |
| `broadcast_scalar_trait` | 融合點運算式中的純量包裝器   | `core::ops::BroadcastScalar` | 2.6.6 |
| `materialize_trait`   | 融合點運算式的結果              | `core::ops::Materialize`   | 2.6.6 |
| `close_trait`         | 可能失敗的關閉：由擁有者關閉它  | `core::io::Close`     | 2.23 |

接著是運算子 trait 槽位。運算子語彙單元的分派目標由附加在各 trait 方法上的 `#[infix]` / `#[prefix]`（[refs-lang 6.7](../../refs-lang/006-meta-system/607-operator-magic.md)）決定，優先順序由 [refs-lang 3.2](../../refs-lang/003-expressions/302-precedence-and-associativity.md) 的表格固定；下列槽位是固定位置，讓編譯器在基本型別降階和診斷時能以名稱找到「加法 trait」、「比較 trait」等。複合指派（`AddAssign` 等，2.4）、廣播（`DotAdd` 等，2.6）、乘冪（`Pow`，`**`）和向下取整模數（`Modulo`，`%%`）的 trait 只需要 `#[infix]` 登記，沒有槽位：降階和診斷從不以名稱查詢它們。在基本型別上，`**` 呼叫一般方法（`pow`、`powi`），`%%` 是 `#[intrinsic]` 方法（[0.3](003-intrinsics.md)），因此它們的降階也不需要槽位。

| 槽位名稱            | 意義                                     | 標準函式庫 trait             | 參照 |
|---------------------|------------------------------------------|------------------------------|------|
| `add_trait`         | `a + b`                                  | `core::ops::Add`        | 2.3  |
| `sub_trait`         | `a - b`                                  | `core::ops::Sub`        | 2.3  |
| `mul_trait`         | `a * b`                                  | `core::ops::Mul`        | 2.3  |
| `div_trait`         | `a / b`                                  | `core::ops::Div`        | 2.3  |
| `rem_trait`         | `a % b`                                  | `core::ops::Rem`        | 2.3  |
| `neg_trait`         | `-a`（前綴）                            | `core::ops::Neg`        | 2.3  |
| `not_trait`         | `!a`                                     | `core::ops::Not`        | 2.3  |
| `bitnot_trait`      | `~a`                                     | `core::ops::BitNot`     | 2.3  |
| `bitand_trait`      | `a & b`                                  | `core::ops::BitAnd`     | 2.3  |
| `bitor_trait`       | `a \| b`                                 | `core::ops::BitOr`      | 2.3  |
| `bitxor_trait`      | `a ^ b`                                  | `core::ops::BitXor`     | 2.3  |
| `shl_trait`         | `a << b`                                 | `core::ops::Shl`        | 2.3  |
| `shr_trait`         | `a >> b`                                 | `core::ops::Shr`        | 2.3  |
| `logical_and_trait` | `a && b`                                 | `core::ops::LogicalAnd` | 2.3  |
| `logical_or_trait`  | `a \|\| b`                               | `core::ops::LogicalOr`  | 2.3  |
| `call_trait`        | 對非函式的值進行 `v(a)`                  | `core::ops::Call`       | 2.3  |
| `eq_trait`          | `==` / `!=`、字面值模式                  | `core::ops::PartialEq` | 2.5  |
| `cmp_trait`         | `<` / `>` / `<=` / `>=`、範圍模式        | `core::ops::PartialOrd` | 2.5  |
| `ord_trait`         | 全序（`#[derive(Ord)]`）                | `core::cmp::Ord`        | 2.5  |
| `hash_trait`        | 雜湊（`#[derive(Hash)]`）               | `core::cmp::Hash`       | 2.5  |
| `clone_trait`       | 複製（`#[derive(Clone)]`）              | `core::marker::Clone`   | 2.13 |
| `send_trait`        | 自動標記：可安全地跨執行緒移動           | `core::marker::Send`   | 2.13 |
| `sync_trait`        | 自動標記：可安全地跨執行緒共用           | `core::marker::Sync`  | 2.13 |

> [!NOTE]
> 比較用的槽位名稱與 trait 名稱並非一一對應。`eq_trait` 附加在定義相等運算子的 `PartialEq` 上，`cmp_trait` 附加在定義順序運算子的 `PartialOrd` 上。標記 trait `Eq`、`Copy` 和 `Default` 沒有槽位：和型別以結構方式取得它們（`T` 是 `Eq` 時 `T | none` 也是 `Eq`；其 `Default` 是 `none`），因此編譯器以它們在 `core` 中的固定路徑找到它們。

## 0.4.4. 函式槽位（42）

函式槽位用於編譯器像內建函式一樣直接呼叫的標準函式庫函式。它們主要涵蓋非正常終止的路徑（`throw`、OOM、不可到達），用在一般流程之外需要統一進入點的地方。

| 槽位名稱         | 意義                                     | 標準函式庫函式                  | 參照 |
|------------------|------------------------------------------|---------------------------------|------|
| `panic_fn`       | panic 時呼叫的中止函式                   | `core::panic::panic`       | 2.14 |
| `oom_fn`         | OOM 中止（執行環境配置失敗時直接呼叫 `nabi_oom`） | `core::panic::oom` | 2.14 |
| `unreachable_fn` | 編譯器在不可到達之處呼叫                 | `core::panic::unreachable` | 2.14 |

其餘三十九個是**鉤子**：由編譯器所寫的程式碼呼叫的函式，用於負索引、切片邊界、可寫入檢視、展開、f-string 填空、取消點、對集合型別的廣播、巢狀陣列字面值以及融合點運算式。降階透過槽位而非路徑找到每一個鉤子，因此鉤子不必對它被降階進去的程式碼可見。鉤子不是 API：每一個在 `core` 中都是 `pub(pkg)`，在其他套件中手寫的呼叫是錯誤。

| 槽位名稱             | 呼叫時機                                               | 標準函式庫函式                            | 參照 |
|----------------------|--------------------------------------------------------|-------------------------------------------|------|
| `spread_len_fn`      | `List` / `Slice` 展開 `f(..xs)` 檢查其長度             | `core::index::spread_len`            | 2.3  |
| `spread_names_fn`    | `Dict` 展開 `f(..d)` 檢查其鍵                          | `core::index::spread_names`          | 2.3  |
| `spread_misfit_fn`   | `Dict` 展開中無法填入參數的鍵                          | `core::index::spread_misfit`         | 2.3  |
| `index_from_end_fn`  | 有號索引 `xs[i]`，負值表示從尾端起算                   | `core::index::index_from_end`        | 2.3  |
| `slice_bound_fn`     | 有號切片邊界 `xs[i:]`                                  | `core::index::slice_bound`           | 2.3  |
| `slice_range_fn`     | 帶步長的切片 `xs[a:b:k]`                               | `core::index::slice_range`           | 2.3  |
| `slice_mut_fn`       | 在預期 `SliceMut` 的地方出現串列的 `xs` 或 `xs[a:b]` | `core::index::slice_mut`（`pub(pkg)`） | 2.9  |
| `view_live_fn`       | 對串列或檢視進行 `for ref` 迴圈的每一步：檢視過期時觸發 panic | `core::index::view_live`（`pub(pkg)`） | 2.9  |
| `fstr_display_fn`    | f-string 填空 `{v}`                                    | `core::fmt::fstr_display`            | 2.10 |
| `fstr_debug_fn`      | f-string 填空 `{v:?}`                                  | `core::fmt::fstr_debug`              | 2.10 |
| `fstr_num_fn`        | 帶格式規範的 `int` 或 `float` 填空                     | `core::fmt::fstr_num`                | 2.10 |
| `fstr_pack_fn`       | 格式規範由計算得出的填空 `{v:{spec}}`                  | `core::fmt::fstr_pack`               | 2.10 |
| `fstr_text_any_fn`   | 對非數字的值使用計算得出的格式規範                     | `core::fmt::fstr_text_any`           | 2.10 |
| `fstr_args_fn`       | `Arguments` f-string：涵蓋其各片段的閉包               | `core::fmt::fstr_args`               | 2.10 |
| `fstr_put_str_fn`    | `Arguments` f-string：字面片段，或已轉成文字的填空     | `core::fmt::fstr_put_str` | 2.10 |
| `fstr_put_display_fn` | `Arguments` f-string：填空 `{v}`                      | `core::fmt::fstr_put_display`        | 2.10 |
| `fstr_put_debug_fn`  | `Arguments` f-string：填空 `{v:?}`                     | `core::fmt::fstr_put_debug`          | 2.10 |
| `cancel_point_fn`    | 輪詢得到 `Pending` 的 `await`（取消點）                | `core::task::cancel_point`           | 2.18 |
| `cancel_left_fn`     | 輪詢結果為 `Ready` 的 `await`                          | `core::task::cancel_left`            | 2.18 |
| `cleanup_running_fn` | 因取消而離開的框架進入其清理程式                       | `core::task::cleanup_running`        | 2.18 |
| `cleanup_done_fn`    | 該框架完成其清理程式                                   | `core::task::cleanup_done`           | 2.18 |
| `broadcast_len_fn`   | 廣播 `xs .+ ys` 檢查長度是否一致                       | `core::collections::broadcast::check_len` | 2.6 |
| `axis_from_u64_fn`   | 軸索引中的無號位置                                     | `core::index::axis_from_u64`         | 2.3  |
| `axis_from_i64_fn`   | 軸索引中的寬有號位置                                   | `core::index::axis_from_i64`         | 2.3  |
| `array_literal_fn`   | 位於 `Array` 槽位中的巢狀串列字面值                    | `core::array::array_literal`         | 2.9.14 |
| `fused_shape_fn` | 融合點運算式：運算元延展到的形狀 | `core::array::fused_shape` | 2.6.6 |
| `fused_view_fn` | 融合點運算式：以該形狀檢視的運算元 | `core::array::fused_view` | 2.6.6 |
| `fused_alloc_fn` | 融合點運算式：新建立的結果陣列 | `core::array::fused_alloc` | 2.6.6 |
| `fused_flat_fn` | 融合點運算式：某個檢視是否連續 | `core::array::fused_flat` | 2.6.6 |
| `fused_inner_fn` | 融合點運算式：一列（最後一軸）的長度 | `core::array::fused_inner` | 2.6.6 |
| `fused_rows_fn` | 融合點運算式：列數 | `core::array::fused_rows` | 2.6.6 |
| `fused_row_fn` | 融合點運算式：某一列在緩衝區中的位置 | `core::array::fused_row` | 2.6.6 |
| `fused_step_fn` | 融合點運算式：沿最後一軸的步長 | `core::array::fused_step` | 2.6.6 |
| `fused_at_fn` | 融合點運算式：連續區段中的一個元素 | `core::array::fused_at` | 2.6.6 |
| `fused_put_fn` | 融合點運算式：儲存到連續區段 | `core::array::fused_put` | 2.6.6 |
| `fused_get_fn` | 融合點運算式：帶步幅列中的一個元素 | `core::array::fused_get` | 2.6.6 |
| `fused_set_fn` | 融合點運算式：儲存到帶步幅的列 | `core::array::fused_set` | 2.6.6 |
| `fused_dest_fn` | 融合點運算式：就地結果符合其目標 | `core::array::fused_dest` | 2.6.6 |
| `fused_source_fn` | 融合點運算式：與目標重疊的就地來源會被複製 | `core::array::fused_source` | 2.6.6 |
| `fused_list_alloc_fn` | 對串列的融合點運算式：依索引填入的結果串列 | `core::collections::broadcast::fused_list_alloc` | 2.6.6 |

### 各層級共用的項目：`#[internal]`

有少數 `pub` 項目不是由編譯器呼叫，而是由工具鏈的另一個層級呼叫：`std` 從 `core` 使用它們，`core` 從 `bare` 使用它們。它們標記為 `#[internal]`（[refs-lang 6.2.15](../../refs-lang/006-meta-system/602-built-in-attributes.md#6215-internal-由工具鏈的套件共用)）：`bare`、`core` 和 `std` 可以使用它們，其他任何套件會得到錯誤「internal to the Nabi toolchain」，而文件和語言伺服器會略過它們。

| 項目                                   | 使用者                                                    | 參照 |
|----------------------------------------|-----------------------------------------------------------|------|
| `core::marker::hand_over`         | `std::thread`（交出執行緒的結果）      | 2.13 |
| `core::marker::Handoff::new`     | `std::thread`（執行緒的結果跨回到 join 的一方） | 2.13 |
| `core::marker::view`              | `std::net`（listener 的 `incoming` 檢視）             | 2.13 |
| `core::task::wake_io`             | `std::net`（在關閉或 shutdown 之後喚醒等待者） | 2.18 |
| `core::task::host_enter`          | `std::wit`（元件匯出進入主機迴圈）  | 2.18 |
| `core::task::host_run`            | `std::wit`（主機迴圈的一輪）                | 2.18 |
| `core::task::host_leave`          | `std::wit`（結束 `host_enter`）                        | 2.18 |
| `core::task::readable` / `writable` | `std::net`（等待通訊端就緒）          | 2.18 |
| `core::io::IoError::retold`       | `std::fs`（針對某個路徑重新陳述的錯誤）            | 2.23 |
| `core::io::IoError::resolver`     | `std::net`（名稱查詢失敗）                     | 2.23 |
| `core::io::Owned::of`            | `std::net`（`TcpListener::into_incoming`）             | 2.23 |
| `core::io::Read::copy_handle` / `Write::copy_handle` | `std` 的 fs、net、process 和 `os::unix::net`（`io::copy` 交給核心的 OS 控制代碼） | 2.23 |
| `core::num::bigint::Int::from_literal` / `bigfloat::Float::from_literal` | `std::nabion`（數字字面值的位數） | 2.1 |
| `bare::gc::Pinnable::gc_object`   | `core::marker`（複製 `GcPtr` 的交接）         | 1.6  |

## 0.4.5. 方法槽位（1）

方法槽位用於編譯器在使用者沒有寫出呼叫時自動呼叫的方法。只有一個：`using` 繫結被釋放時的處置。`?` 運算子由編譯器針對 `T | none` 直接展開（[refs-lang 3.14.10](../../refs-lang/003-expressions/314-error-control-flow.md#31410--運算子提前回傳-none)），因此沒有方法槽位。值的複製（`Clone::clone`）永遠不會自動呼叫（[2.13.2](../200-core/213-marker.md#2132-clone深層複製)），因此它登記為 trait 槽位 `clone_trait`，而不是方法槽位。

| 槽位名稱      | 意義                                 | 標準函式庫方法                  | 參照 |
|---------------|--------------------------------------|---------------------------------|------|
| `drop_method` | 在值被釋放的位置呼叫（`using`，或其擁有者離開作用域） | `core::marker::Drop::drop` | 2.13 |

## 0.4.6. 槽位的使用者：編譯器在哪裡查詢槽位

前面各節列出「有哪些槽位」，本節則反過來彙整「編譯器何時查詢哪個槽位」。要追查一行使用者語法會解析到哪個標準函式庫項目時，下表就是起點。

| 編譯器動作                                         | 查詢的槽位                               |
|----------------------------------------------------|------------------------------------------|
| 降階 `none` 字面值                                 | `none_type`                              |
| `42` 的字面值型別（沒有預期型別）                  | `i64_type`                               |
| `3.14` 的字面值型別（沒有預期型別）                | `f64_type`                               |
| 預期型別為 `int`、`float`、`f16` 或 `bf16` 的數字字面值，以及 `f16` / `bf16` 字面值後綴 | `int_type`、`float_type`、`f16_type`、`bf16_type` |
| `"hello"` 的字面值型別                             | `str_type`                               |
| 去語法糖 `for x in xs`                                 | `into_iterator_trait`、`iterator_trait`、`try_iterator_trait` |
| 去語法糖 `arr[i]`                                      | `index_trait`                            |
| 去語法糖 `arr[i] = v`                                  | `index_set_trait`                        |
| 去語法糖 `s[1:3]`                                      | `slice_trait`                            |
| 去語法糖預設的 `f"{x}"` 填空                           | `display_trait`                          |
| 去語法糖 `f"{x:?}"`                                    | `debug_trait`                            |
| 去語法糖 `await expr`                                  | `future_trait`                           |
| 算術 / 位元 / 邏輯運算子的基本型別降階             | `add_trait` ... `logical_or_trait` |
| `==` / `!=`、字面值模式比對                        | `eq_trait`                               |
| `<` / `>` / `<=` / `>=`、範圍模式比對              | `cmp_trait`                              |
| 展開 `#[derive(Clone / Ord / Hash)]`               | `clone_trait`、`ord_trait`、`hash_trait` |
| 降階 panic（包括 `try!` 失敗）                     | `panic_fn`                               |
| 發生 OOM 時                                        | `oom_fn`                                 |
| `match` 對所有分支完成窮盡檢查後的不可到達處       | `unreachable_fn`                 |
| 負索引或切片邊界、帶步長的切片                     | `index_from_end_fn`、`slice_bound_fn`、`slice_range_fn` |
| 在預期 `SliceMut` 的地方出現串列或其範圍           | `slice_mut_fn`                          |
| 對串列或檢視進行 `for ref` 迴圈的每一步            | `view_live_fn`                         |
| 展開 `f(..xs)` / `f(..d)`                          | `spread_len_fn`、`spread_names_fn`、`spread_misfit_fn` |
| f-string 填空                                      | `fstr_display_fn` ... `fstr_text_any_fn` |
| `await`，以及取消時執行的清理程式                  | `cancel_point_fn`、`cancel_left_fn`、`cleanup_running_fn`、`cleanup_done_fn` |
| 對兩個以上集合型別的廣播                           | `broadcast_len_fn`                       |
| 釋放值（`using` 區塊結束，或其最後一個擁有者離開作用域） | `drop_method`              |
| 自動實作 `Send` / `Sync`、`+ Send` 檢查            | `send_trait`、`sync_trait`        |

## 0.4.7. 重複 / 缺漏偵測

如果兩個標準函式庫項目附加到同一個槽位，編譯器會拒絕（refs-lang 6.4.6 的重複偵測規則）。反之，如果缺少 `iterator_trait` 這類必要的槽位，依賴它的功能（這裡是 `for` 迴圈）的編譯會被拒絕。兩者都是強制槽位契約只有單一事實來源的機制。

實際附加的項目寫在 [200-core/](../200-core/) 的各章中。每一個有附加的 trait、方法、型別和中止函式槽位，其 `#[lang(...)]` 附加位置都在「參照」欄所指的章節中。鉤子 `_fn` 槽位（`spread_len_fn` ... `fused_list_alloc_fn`）只供編譯器使用，只在這張表中列名；0.4.2 未附加的內建型別槽位則沒有附加位置。依 [refs-lang 6.4.8](../../refs-lang/006-meta-system/604-lang-magic.md)，不在這張表中的槽位名稱是編譯錯誤。例如 `Stream` 沒有槽位（[2.8.10](../200-core/208-stream.md)）。

## 0.4.8. 槽位命名政策

最後是新增槽位或重新命名現有槽位時應遵守的命名規則。名稱一致，就能只靠瀏覽表格猜出槽位的種類，因此這項政策並非小事。

名稱以 snake_case 撰寫：型別槽位加 `_type` 後綴，trait 槽位加 `_trait` 後綴，函式槽位加 `_fn` 後綴，方法槽位加 `_method` 後綴（`drop_method`）。指定整個 trait 的槽位，即使編譯器處理該 trait 的多個方法或衍生，也統一為單一的 `_trait`；方法槽位只限於編譯器**自動呼叫某一個特定方法**的情況。依此準則，`Clone` 登記為 `clone_trait`，而不是 `clone_method`。後綴也決定附加位置：`_trait` 槽位附加在 **trait 宣告本身**，`_method` 槽位附加在 **trait 內對應的方法**，`_type` 槽位附加在型別宣告，`_fn` 槽位附加在函式宣告；附加到其他地方會被 [refs-lang 6.4.7](../../refs-lang/006-meta-system/604-lang-magic.md) 的簽章檢查拒絕。
