# 1.1. Source files

This section covers the physical format of Nabi source files: how a file must be encoded on disk, and what the compiler assumes when it begins reading the first byte.

## 1.1.1. Extension

The extension of a Nabi source file is `.nabi`. Other extensions, such as `.no` or `.nabir`, are not recognized as Nabi source.

## 1.1.2. Character encoding

> [!IMPORTANT]
> Source files must be encoded in **UTF-8**.

Files written in other encodings such as UTF-16, GBK or Shift_JIS are rejected by the compiler immediately. Automatic encoding detection is not attempted, because guesses are wrong more often than not. A BOM (`U+FEFF`) at the start of a file is silently ignored; anywhere else it is an invisible character (below).

### Bidirectional control characters are forbidden

The bidirectional (bidi) control characters U+202A-202E and U+2066-2069 are a compile error anywhere in the source (including comments and string literals). This prevents attacks in which the displayed code differs from the actual token order. If a string needs one of these characters, write it with an escape such as `\u{202E}`.

### Invisible characters

A character that shows nothing is not read as code: the zero-width space, non-joiner and joiner (U+200B-200D), the left-to-right and right-to-left marks (U+200E, U+200F, U+061C), the word joiner and invisible operators (U+2060-2064), the byte order mark U+FEFF past the start of the file, the soft hyphen U+00AD, the Hangul fillers (U+115F, U+1160, U+3164, U+FFA0), and the variation selectors (U+FE00-FE0F, U+E0100-E01EF) and tags (U+E0000-E007F) when they modify nothing. In code, and inside an identifier, each one is a compile error that names the character; so is a space other than U+0020 and tab, such as U+00A0 NO-BREAK SPACE (write a plain space). Inside a string, a character literal or a comment, such a character is a `hidden_chars` warning ([6.2.9](../006-meta-system/602-built-in-attributes.md#629-allow--deny--warn-lint-control)): write it as a `\u{..}` escape so readers see it. The joiners are not reported where they belong: a ZWJ, variation selector or tag inside an emoji sequence (`👨‍👩‍👧`, `❤️`), and a ZWNJ after a letter of a script that writes one (Persian, the Indic scripts).

### Look-alike and unassigned characters

A character that only looks like an ASCII token is a compile error that names the token to write: `−` (U+2212 MINUS SIGN) for `-`, curly quotes for `"` and `'`, the fullwidth forms (`（`, `；`, `＝`) and the like. A package that registers such a character as an operator token ([1.10.4](110-punctuation-and-operator-tokens.md#1104-extension-tokens)) makes it a token of its own, and then it is not reported. A code point that Unicode 18.0.0 leaves unassigned is an error in code and a `hidden_chars` warning in strings and comments.

Diagnostics name such a character by its code point and Unicode name (`U+200B ZERO WIDTH SPACE`), and show the character itself only when it is visible.

## 1.1.3. Line terminators

All three of the following are recognized as the end of a logical line.

- LF (`U+000A`)
- CRLF (`U+000D U+000A`)
- CR (`U+000D` alone)

The compiler accepts these three forms even when mixed in one file, since it must work regardless of where a file was produced. This specification recommends LF alone.

Other line-separating code points, such as `U+2028` LINE SEPARATOR or `U+2029` PARAGRAPH SEPARATOR, are not line terminators. They appear as ordinary code points inside identifiers or strings.

## 1.1.4. Line and file size limits

This specification sets no upper bound on line length, file size or identifier length. A compiler may impose reasonable limits, but such limits are implementation decisions and not part of the specification.

## 1.1.5. Position units in diagnostics

Compiler diagnostics (errors, warnings) give positions as file, line and column.

Line numbers start at 1. The column is a 1-based index counted in code points from the start of the line (code points, not bytes). A multibyte character such as a Hangul syllable therefore occupies one column. A tab (`U+0009`) also occupies one column and does not imitate its visual width.

These rules make diagnostic positions easy for machines (editors and LSPs) to process. Visual alignment is left to the display tool.
