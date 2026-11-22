# 1.2. 語彙結構

本章概述字元串流如何變成語彙單元串流。各種語彙單元的詳細格式定義於後續文件；這裡只給出輪廓。

```
token           ::= identifier
                  | keyword
                  | numeric_lit
                  | string_lit
                  | byte_string_lit
                  | fstring_tokens                // f"..." / f"""...""", see 1.2.1
                  | char_lit
                  | sentinel_lit                  // true | false | none
                  | label                         // 'name
                  | punctuation
label           ::= "'" identifier                // no closing quote
whitespace      ::= space_char+
                  | line_comment
                  | block_comment
space_char      ::= U+0009 | U+000A | U+000B | U+000C | U+000D | U+0020
line_comment    ::= "//" <any char except line_terminator>* line_terminator
block_comment   ::= "/*" <any char>* "*/"          // recursive
shebang_line    ::= "#!" <any char except line_terminator>*   // first line only, see 1.2.3
line_terminator ::= U+000A | U+000D U+000A? | U+000D
```

## 1.2.1. 語彙單元

編譯器的語彙分析器（**lexer**）讀取原始檔並產生語彙單元串流。語彙單元分為下列幾類。

| 類別                         | 範例                                |
|------------------------------|-------------------------------------|
| 識別字                       | `count`、`α`、`Δt`                  |
| 關鍵字                       | `let`、`if`、`fn`                   |
| 數值字面值                   | `42`、`3.14`、`0xFF`、`1_000`       |
| 字串字面值                   | `"hello"`、`r"raw\n"`、`f"x = {x}"`、`b"bytes"` |
| 字元字面值                   | `'A'`、`c'A'`                       |
| 布林值 / 哨兵值字面值        | `true`、`false`、`none`             |
| 標籤                         | `'outer`                            |
| 運算子 / 標點                | `+`、`==`、`..`、`[`、`;`           |

標籤是 `'` 後接一個識別字，沒有結尾引號（`'outer`）；它為 `break` / `continue` 命名一個迴圈或區塊（[3.13](../003-expressions/313-loop-expressions.md)）。帶有結尾引號的 `'A'` 則是字元字面值。

格式字串不是一個語彙單元，而是一族語彙單元。`f"`（或 `f"""`）開啟它，字面文字段與洞（hole）交替出現，結尾的 `"`（或 `"""`）結束它。一個洞由一個 `{` 語彙單元、其運算式的一般語彙單元、可選的 `:` 與格式規範文字（其中可以含有巢狀的 `{...}` 或 `${...}` 洞），以及一個 `}` 語彙單元組成（[1.7.3](107-string-literals.md#173-format-strings)）。

各類別的確切格式規則依序定義於後續文件。

## 1.2.2. 空白

空白字元只作為語彙單元之間的分隔才有意義，本身不構成語彙單元。

下列碼位被辨識為空白。

| 碼位       | 名稱                 |
|------------|----------------------|
| `U+0009`   | 水平定位（TAB）      |
| `U+000A`   | 換行（LF）           |
| `U+000B`   | 垂直定位（VT）       |
| `U+000C`   | 換頁（FF）           |
| `U+000D`   | 歸位（CR）           |
| `U+0020`   | 空格                 |

上表以外的 Unicode 類空格字元（`U+00A0` NO-BREAK SPACE、`U+200B` ZERO WIDTH SPACE 等）不是空白。它們只能以一般碼位的身分出現在字串與註解中；出現在其他任何位置都是語彙錯誤。

縮排沒有意義。Nabi 不採用 Python 式有意義的縮排。

## 1.2.3. 註解

Nabi 支援三種註解：行註解、區塊註解與文件註解。檔案第一行的 shebang 行會像註解一樣被忽略。

### Shebang 行

若檔案的第一行（BOM 之後）以 `#!` 開頭，該行就是 shebang 行，會像註解一樣被忽略，因此帶有 `#!/usr/bin/env nabi` 的指令碼可以用 `./script.nabi` 執行（[9.10.1](../009-tools/910-nabi-frontend.md#9101-subcommands-and-global-options)）。例外是 `#!` 之後、略過空白與文件註解以外的註解後的第一個語彙單元是 `[` 時：此時該行不是 shebang 行，`#!` 開啟一個內部屬性（`#![allow(unused)]`）。屬性的 `#![` 是一個內部不含空白的語彙單元（[1.10.3](110-punctuation-and-operator-tokens.md)），因此 `#! [allow(unused)]` 是錯誤。之後任何一行上的 `#!` 都不是 shebang 行。

### 行註解

```nabi
// comment to the end of this line
```

從 `//` 到該行的行終止符是註解。換行本身不屬於註解。

### 區塊註解

```nabi
/* spanning
   multiple lines */
```

`/*` 與 `*/` 之間的文字是註解，而且允許巢狀。外層註解只有在內層的 `/*`/`*/` 成對正確關閉之後才會關閉（與 Rust 的規則相同），這對暫時停用大段程式碼很有用。

```nabi
/* outer
   /* inner */
   still a comment */
```

### 文件註解

文件註解有四種形式，與 Rust 相同。

```nabi
/// outer doc comment (the following item)
//! inner doc comment (the enclosing file, module or function)
/** outer block doc comment */
/*! inner block doc comment */
```

- 行文件註解延伸到行終止符；`/// text` 是 `#[doc = "text"]`（標記之後的一個空格會被去掉），`//! text` 是 `#![doc = "text"]`（[6.2.8](../006-meta-system/602-built-in-attributes.md#628-doc---doc-comment-form)）。以 `////` 開頭的行是一般註解。
- 區塊文件註解延伸到與它配對的 `*/`，並和區塊註解一樣可以巢狀。`/**/`、`/***/` 以及以 `/***` 開頭的註解是一般註解。其文字是一個 `#[doc]` / `#![doc]` 屬性。空白的第一行與空白的最後一行會被去掉。當註解跨越多行，且第一行之後的每一行都以可選的空白與 `*` 開頭時，該前綴會被移除。接著移除非空白行共有的縮排，以及 `*/` 之前的空白。換行是 LF。
- 內部文件註解只出現在檔案、模組主體或函式主體的開頭。出現在其他任何位置都是錯誤。
- 外部文件註解為其後的項目撰寫文件。不為任何東西撰寫文件的外部文件註解（位於 `}`、`)`、`]` 或檔案結尾之前）是錯誤，位於函式參數上的也是錯誤。位於陳述式、match 分支、結構字面值欄位、元組欄位或運算式上的外部文件註解不會被任何工具保留，並觸發 `unused_doc_comments` lint（[6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)）。
- 預先建置之套件的 NABIHIR 介面文字（[9.4](../009-tools/904-nabihir-bytecode.md)）會保留文件註解。

## 1.2.4. 編譯器輸出

語彙分析器的輸出由語彙單元串流與位置資訊組成。位置資訊遵循[原始檔](101-source-files.md)的診斷單位規則。

這個輸出是下一個階段，即剖析器的唯一輸入；剖析器不會再看原始的原始碼。
