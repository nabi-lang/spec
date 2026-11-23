# 2.22. `encoding::`：hex 與 base64

`core::encoding` 以程式最常遇到的兩種 RFC 4648 編碼，把位元組轉成文字再轉回來：hex（base16）與 base64。它們是純計算，在每個目標上的行為完全相同，因此位於 `core`（0.1）。每種編碼各是一個模組（`encoding::hex` 與 `encoding::base64`），各有相同的四個函式；`encoding::DecodeError` 是兩者擲回的錯誤。

```nabi
// core::encoding::hex
enum Case { Lower, Upper }
fn encode<B: AsBytes>(data: B, case: Case = Case::Lower) -> str;
fn decode<B: AsBytes>(text: B) throws (DecodeError) -> List<u8>;
fn encode_into<B: AsBytes>(data: B, out: SliceMut<u8>, case: Case = Case::Lower) -> usize;
fn decode_into<B: AsBytes>(text: B, out: SliceMut<u8>) throws (DecodeError) -> usize;

// core::encoding::base64
fn encode<B: AsBytes>(data: B, codec: Codec = STANDARD) -> str;
fn decode<B: AsBytes>(text: B, codec: Codec = STANDARD) throws (DecodeError) -> List<u8>;
fn encode_into<B: AsBytes>(data: B, out: SliceMut<u8>, codec: Codec = STANDARD) -> usize;
fn decode_into<B: AsBytes>(text: B, out: SliceMut<u8>, codec: Codec = STANDARD) throws (DecodeError) -> usize;
const fn encoded_len(n: usize, codec: Codec = STANDARD) -> usize;
const fn decoded_len_max(n: usize) -> usize;
```

位元組引數是 `AsBytes`（[2.11.13](211-conversion.md#21113-asbytes-trait位元組作為檢視)）：串列、檢視、陣列或字串的 UTF-8 位元組。`encode` 回傳新的 `str`，`decode` 回傳新的 `List<u8>`。`_into` 形式寫入 `out` 的開頭，`out` 是呼叫端已決定大小之串列的可寫入檢視（[2.9.7](209-collections.md)），並回傳寫入的位元組數；它們不配置任何記憶體。解碼後的位元組若是 UTF-8 文字，可以透過 `str::from_utf8` 成為 `str`（[2.15.6](215-string.md#2156-utf-8-不變式行程內的每個字串)）。

```nabi
encoding::hex::encode("hi");                   //→ "6869"
encoding::hex::decode("6869");                 //→ [104, 105]
encoding::base64::encode("foob");              //→ "Zm9vYg=="
encoding::base64::decode("Zm9vYg");            //→ [102, 111, 111, 98]
```

`use core::encoding::base64;` 可以把呼叫縮短為 `base64::encode(x)`。

## 2.22.1. Hex

`hex::encode` 把每個位元組寫成兩個十六進位數字，高四位元在前（`[0x00, 0xff]` → `"00ff"`）；數字 `a` 到 `f` 為小寫，使用 `Case::Upper` 時為大寫。`hex::decode` 每個位元組讀取兩個數字，大小寫皆接受（`"DeadBeef"`）。文字長度必須為偶數；除此之外不接受任何東西：沒有 `0x` 前綴，也沒有空格或分隔符。

`hex::encode_into` 需要 `out` 有 `2 * data.len()` 個位元組，`hex::decode_into` 需要 `text.len() / 2` 個；較短的 `out` 是呼叫端的錯誤，會觸發 panic 並指出應給的大小（``hex::encode_into: `out` holds 3 bytes but the text needs 4, two per byte; size `out` as `2 * data.len()` ``，`decode_into` 則為 ``size `out` as `text.len() / 2` ``）。

## 2.22.2. Base64

`base64::encode` 把每 3 個位元組視為四個 6 位元的值，每個值寫一個字元。最後剩 1 或 2 個位元組的組會產生 2 或 3 個字元，可以用 `=` 填補到 4 個。`Codec` 選擇字母表與填補方式：

```nabi
#[non_exhaustive]
enum Alphabet { Standard, UrlSafe }
enum Padding { Present, Absent, PresentOptional, AbsentOptional }
struct Codec { .. }                    // Copy, Eq, Hash, Debug
impl Codec {
    const fn new(alphabet: Alphabet, padding: Padding) -> Codec;
    const fn alphabet(self) -> Alphabet;
    const fn padding(self) -> Padding;
}
const STANDARD: Codec;                 // Standard, PresentOptional
const STANDARD_NO_PAD: Codec;          // Standard, Absent
const URL_SAFE: Codec;                 // UrlSafe, PresentOptional
const URL_SAFE_NO_PAD: Codec;          // UrlSafe, Absent
```

`Alphabet::Standard` 是 RFC 4648 第 4 節（`A`-`Z`、`a`-`z`、`0`-`9`、`+`、`/`）；`Alphabet::UrlSafe` 是第 5 節的 URL 與檔名安全字母表，以 `-` 和 `_` 取代 `+` 和 `/`。填補方式指出 `encode` 寫出什麼，以及 `decode` 接受什麼：

| `Padding`         | `encode` 寫出 `=` | `decode` 接受          |
|-------------------|---------------------|-------------------------|
| `Present`         | 是                 | 只接受有填補的文字        |
| `Absent`          | 否                  | 只接受無填補的文字      |
| `PresentOptional` | 是                 | 有填補或無填補的文字 |
| `AbsentOptional`  | 否                  | 有填補或無填補的文字 |

`STANDARD` 是預設引數：編碼時填補，解碼時兩種形式都接受。要嚴格的解碼器，換一個 codec 即可：`Codec::new(Alphabet::Standard, Padding::Present)`。

| 輸入      | `STANDARD`     | `STANDARD_NO_PAD` |
|------------|----------------|-------------------|
| `""`       | `""`           | `""`              |
| `"f"`      | `"Zg=="`       | `"Zg"`            |
| `"fo"`     | `"Zm8="`       | `"Zm8"`           |
| `"foo"`    | `"Zm9v"`       | `"Zm9v"`          |
| `"foobar"` | `"Zm9vYmFy"`   | `"Zm9vYmFy"`      |

若有填補，就必須正確：結尾有一或兩個 `=`，使長度成為 4 的倍數。最後一個字元中剩餘的位元會被忽略。空白字元與換行不會被略過：分行的文字（PEM、MIME）要先合併，例如使用 `split_whitespace`（[2.15.2](215-string.md#2152-str不可變的-utf-8-字串)）。

```nabi
let raw: List<u8> = [0xfb, 0xff, 0xbf];
base64::encode(raw);                           //→ "+/+/"
base64::encode(raw, base64::URL_SAFE);         //→ "-_-_"
base64::encode(raw, base64::STANDARD_NO_PAD);  //→ "+/+/"
base64::decode("-_-_", base64::URL_SAFE);      //→ [251, 255, 191]
```

`encoded_len(n, codec)` 是 `encode` 對 `n` 個位元組產生的長度，`encode_into` 需要 `out` 有這麼多空間。`decoded_len_max(n)` 是 `n` 個字元最多能解碼成的位元組數；這個長度的 `out` 對 `decode_into` 永遠足夠，`decode_into` 只有在 `out` 短於文字（扣除填補）可能需要的長度時才會觸發 panic。過短的 `out` 是呼叫端的錯誤，不是解碼錯誤。

```nabi
// A 32-byte key as base64 into bytes 4.. of a frame.
let end = 4 + base64::encoded_len(32);
let n = base64::encode_into(key, frame[4:end]);   // n == 44
```

## 2.22.3. 解碼錯誤

```nabi
// core::encoding
struct DecodeError { .. }
impl DecodeError {
    fn kind(self) -> DecodeErrorKind;
    fn position(self) -> usize;        // byte offset in the text
    fn message(self) -> str;           // the Display text
}
#[non_exhaustive]
enum DecodeErrorKind { InvalidByte, InvalidLength, InvalidPadding }   // Copy, Eq, Hash, Debug
impl Display for DecodeError; impl Debug for DecodeError; impl Error for DecodeError;
impl Clone for DecodeError; impl PartialEq for DecodeError; impl Eq for DecodeError;
```

解碼器在遇到第一個問題時擲回 `DecodeError`。`kind` 說明問題的種類，`position` 指出問題在 `text` 中的位元組偏移。訊息以編碼名稱開頭，指出偏移，並說明應該改寫成什麼：

| 問題                                   | `kind`           | 訊息範例                                                                 |
|-------------------------------------------|------------------|---------------------------------------------------------------------------------|
| 長度為奇數的 hex                         | `InvalidLength`  | `hex: odd length 3; every byte takes two digits, so write "0f" rather than "f"` |
| 不是 hex 數字                           | `InvalidByte`    | `hex: 'g' at byte 1 is not a hex digit; use 0-9, a-f or A-F`                    |
| 另一種 base64 字母表的字元  | `InvalidByte`    | `base64: '-' at byte 2 is url-safe base64; decode with base64::URL_SAFE`        |
| 不是 base64 字元                    | `InvalidByte`    | `base64: '*' at byte 1 is not base64; use A-Z, a-z, 0-9, + and /`               |
| 結尾之前出現 `=`                        | `InvalidPadding` | `base64: '=' at byte 1: '=' pads only the end of the text`                      |
| 填補沒有結束一個 4 字元的組    | `InvalidPadding` | `base64: padded text must be a multiple of 4 characters long, not 7; fix the '=' count or drop the padding` |
| codec 不接受的填補           | `InvalidPadding` | `base64: '=' at byte 2, but this codec takes no padding; drop the '=' or decode with base64::STANDARD` |
| codec 要求填補卻沒有填補    | `InvalidPadding` | `base64: text of 2 characters has no '=' padding, which this codec requires; pad it to a multiple of 4 or decode with base64::STANDARD` |
| 最後一組只剩一個字元      | `InvalidLength`  | `base64: text is cut short at byte 4; a last group needs 2 to 4 characters`     |

奇數長度 hex 的位置是它的最後一個數字；缺少填補時是文字的結尾；填補長度錯誤時是第一個 `=`。空格或可列印 ASCII 範圍之外的位元組以其值顯示（`byte 0x20`、`byte 0x0a`）。在兩種編碼中，都從開頭掃描，字元錯誤會先於長度與填補錯誤回報：hex `"0g1"` 是位元組 1 的 `InvalidByte`，而不是奇數長度。

## 2.22.4. 與其他語言的比較

| 語言 | Hex                          | Base64                                                    |
|----------|------------------------------|-----------------------------------------------------------|
| Rust     | 第三方 `hex` crate      | 第三方 `base64` crate（`STANDARD`、`URL_SAFE_NO_PAD` 等 engine） |
| Python   | `bytes.hex`、`bytes.fromhex` | `base64.b64encode`、`base64.urlsafe_b64encode` 等       |
| Go       | `encoding/hex`               | `encoding/base64`（`StdEncoding`、`RawURLEncoding` 等）  |
| Zig      | `std.fmt.hex`                | `std.base64`（`standard`、`url_safe_no_pad` 等）         |
| Nabi      | `hex::encode`、`hex::decode` | `base64::encode`、`base64::decode`（`STANDARD`、`URL_SAFE_NO_PAD` 等 codec） |

Nabi 遵循 Go 與 Zig：每種編碼一個模組，變體以 codec 值而非旗標表示，常見情況使用 `str` 形式，另有不配置記憶體的緩衝區形式（`_into`）。填補方針同時指明兩端，如同 Kotlin 的 `PaddingOption`，因此 `STANDARD` 會寫出填補，但仍接受沒有填補的文字。
