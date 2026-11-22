# 1. 語彙結構

本章涵蓋把 Nabi 原始碼切分為語彙單元的規則。它從檔案與行開始，接著是識別字、關鍵字、字面值與運算子語彙單元，逐一檢視語彙分析過程中遇到的每個元素。

型別、運算式與陳述式等較高層接收的是已經切分好的語彙單元，因此語彙分析期間適用的所有規則都集中在這一章。

## 目錄

- [1.1. 原始檔](101-source-files.md)。副檔名、編碼、行終止符。
- [1.2. 語彙結構](102-lexical-structure.md)。語彙單元、空白、註解。
- [1.3. 陳述式終止符](103-statement-terminator.md)。`;` 以及可以省略它的位置。
- [1.4. 識別字](104-identifiers.md)。名稱格式。
- [1.5. 關鍵字](105-keywords.md)。保留字清單。
- [1.6. 數值字面值](106-numeric-literals.md)。
- [1.7. 字串字面值](107-string-literals.md)。
- [1.8. 字元字面值](108-character-literals.md)。
- [1.9. 布林值與哨兵值字面值](109-other-literals.md)。`true`、`false`、`none`。
- [1.10. 標點與運算子語彙單元](110-punctuation-and-operator-tokens.md)。
