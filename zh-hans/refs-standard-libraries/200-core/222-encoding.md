# 2.22. `encoding::`：hex 与 base64

`core::encoding` 用程序最常遇到的两种 RFC 4648 编码，即 hex（base16）和 base64，把字节转换为文本并转换回来。它们是纯计算，在每个目标上的行为都相同，所以放在 `core` 中（0.1）。每种编码是一个模块，即 `encoding::hex` 和 `encoding::base64`，它们有同样的四个函数；`encoding::DecodeError` 是两者都会抛出的错误。

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

字节参数是 `AsBytes`（[2.11.13](211-conversion.md#21113-asbytes-trait字节视图)）：列表、视图、数组或字符串的 UTF-8 字节。`encode` 返回新的 `str`，`decode` 返回新的 `List<u8>`。`_into` 形式写入 `out` 的开头部分，`out` 是调用方设定好大小的列表的可写视图（[2.9.7](209-collections.md)），它们返回写入的字节数；它们不做任何分配。解码后的字节如果是 UTF-8 文本，可以通过 `str::from_utf8` 变为 `str`（[2.15.6](215-string.md#2156-utf-8-不变式进程内的每个字符串)）。

```nabi
encoding::hex::encode("hi");                   //→ "6869"
encoding::hex::decode("6869");                 //→ [104, 105]
encoding::base64::encode("foob");              //→ "Zm9vYg=="
encoding::base64::decode("Zm9vYg");            //→ [102, 111, 111, 98]
```

写 `use core::encoding::base64;` 可以把调用缩短为 `base64::encode(x)`。

## 2.22.1. Hex

`hex::encode` 把每个字节写成两位数字，高半字节在前（`[0x00, 0xff]` → `"00ff"`）；数字 `a` 到 `f` 为小写，使用 `Case::Upper` 时为大写。`hex::decode` 每个字节读取两位数字，大小写都接受（`"DeadBeef"`）。文本长度必须为偶数；不接受其他任何内容：没有 `0x` 前缀，没有空格或分隔符。

`hex::encode_into` 需要 `out` 有 `2 * data.len()` 个字节，`hex::decode_into` 需要 `text.len() / 2` 个字节；`out` 更短属于调用方的 bug，会触发 panic 并给出应有的大小（``hex::encode_into: `out` holds 3 bytes but the text needs 4, two per byte; size `out` as `2 * data.len()` ``，`decode_into` 则为 ``size `out` as `text.len() / 2` ``）。

## 2.22.2. Base64

`base64::encode` 把每 3 个字节看作四个 6 位值，每个值写一个字符。最后不足 3 字节的一组（1 或 2 个字节）给出 2 或 3 个字符，可以用 `=` 填充到 4 个。`Codec` 选择字母表和填充方式：

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

`Alphabet::Standard` 是 RFC 4648 第 4 节的字母表（`A`-`Z`、`a`-`z`、`0`-`9`、`+`、`/`）；`Alphabet::UrlSafe` 是第 5 节的 URL 与文件名安全字母表，用 `-` 和 `_` 代替 `+` 和 `/`。填充方式规定了 `encode` 写出什么以及 `decode` 接受什么：

| `Padding`         | `encode` 写出 `=`   | `decode` 接受           |
|-------------------|---------------------|-------------------------|
| `Present`         | 是                  | 仅带填充的文本          |
| `Absent`          | 否                  | 仅不带填充的文本        |
| `PresentOptional` | 是                  | 带或不带填充的文本      |
| `AbsentOptional`  | 否                  | 带或不带填充的文本      |

`STANDARD` 是默认实参：编码时填充，解码时两种形式都接受。严格的解码器只需换一个编解码器：`Codec::new(Alphabet::Standard, Padding::Present)`。

| 输入       | `STANDARD`     | `STANDARD_NO_PAD` |
|------------|----------------|-------------------|
| `""`       | `""`           | `""`              |
| `"f"`      | `"Zg=="`       | `"Zg"`            |
| `"fo"`     | `"Zm8="`       | `"Zm8"`           |
| `"foo"`    | `"Zm9v"`       | `"Zm9v"`          |
| `"foobar"` | `"Zm9vYmFy"`   | `"Zm9vYmFy"`      |

存在的填充必须正确：末尾有一个或两个 `=`，使长度成为 4 的倍数。最后一个字符中剩余的位被忽略。空白和换行不会被跳过：按行折断的文本（PEM、MIME）要先拼接起来，例如用 `split_whitespace`（[2.15.2](215-string.md#2152-str不可变的-utf-8-字符串)）。

```nabi
let raw: List<u8> = [0xfb, 0xff, 0xbf];
base64::encode(raw);                           //→ "+/+/"
base64::encode(raw, base64::URL_SAFE);         //→ "-_-_"
base64::encode(raw, base64::STANDARD_NO_PAD);  //→ "+/+/"
base64::decode("-_-_", base64::URL_SAFE);      //→ [251, 255, 191]
```

`encoded_len(n, codec)` 是 `encode` 对 `n` 个字节给出的长度，`encode_into` 需要 `out` 有这么长。`decoded_len_max(n)` 是 `n` 个字符最多解码出的字节数；这个长度的 `out` 对 `decode_into` 总是足够，只有当 `out` 短于文本（去掉填充后）可能需要的长度时，`decode_into` 才会触发 panic。`out` 过短属于调用方的 bug，而不是解码错误。

```nabi
// A 32-byte key as base64 into bytes 4.. of a frame.
let end = 4 + base64::encoded_len(32);
let n = base64::encode_into(key, frame[4:end]);   // n == 44
```

## 2.22.3. 解码错误

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

解码器在遇到第一个问题时抛出 `DecodeError`。`kind` 说明问题的种类，`position` 给出问题在 `text` 中的字节偏移。消息以编码名开头，指出偏移，并说明应当改写成什么：

| 问题                                      | `kind`           | 消息示例                                                                        |
|-------------------------------------------|------------------|---------------------------------------------------------------------------------|
| 长度为奇数的 hex                          | `InvalidLength`  | `hex: odd length 3; every byte takes two digits, so write "0f" rather than "f"` |
| 不是 hex 数字                             | `InvalidByte`    | `hex: 'g' at byte 1 is not a hex digit; use 0-9, a-f or A-F`                    |
| 另一种 base64 字母表的字符                | `InvalidByte`    | `base64: '-' at byte 2 is url-safe base64; decode with base64::URL_SAFE`        |
| 不是 base64 字符                          | `InvalidByte`    | `base64: '*' at byte 1 is not base64; use A-Z, a-z, 0-9, + and /`               |
| 末尾之前出现 `=`                          | `InvalidPadding` | `base64: '=' at byte 1: '=' pads only the end of the text`                      |
| 填充没有结束一个 4 字符组                 | `InvalidPadding` | `base64: padded text must be a multiple of 4 characters long, not 7; fix the '=' count or drop the padding` |
| 编解码器不接受的填充                      | `InvalidPadding` | `base64: '=' at byte 2, but this codec takes no padding; drop the '=' or decode with base64::STANDARD` |
| 编解码器要求填充却没有填充                | `InvalidPadding` | `base64: text of 2 characters has no '=' padding, which this codec requires; pad it to a multiple of 4 or decode with base64::STANDARD` |
| 最后一组只剩一个字符                      | `InvalidLength`  | `base64: text is cut short at byte 4; a last group needs 2 to 4 characters`     |

hex 长度为奇数时，位置是最后一位数字；缺少填充时，位置是文本末尾；填充长度错误时，位置是第一个 `=`。空格或可打印 ASCII 之外的字节以其值表示（`byte 0x20`、`byte 0x0a`）。两种编码都从头扫描，先报告字符错误，再报告长度和填充错误：hex `"0g1"` 是第 1 字节处的 `InvalidByte`，而不是奇数长度。

## 2.22.4. 与其他语言的比较

| 语言     | Hex                          | Base64                                                    |
|----------|------------------------------|-----------------------------------------------------------|
| Rust     | 第三方 `hex` crate           | 第三方 `base64` crate（`STANDARD`、`URL_SAFE_NO_PAD` 等引擎） |
| Python   | `bytes.hex`、`bytes.fromhex` | `base64.b64encode`、`base64.urlsafe_b64encode` 等         |
| Go       | `encoding/hex`               | `encoding/base64`（`StdEncoding`、`RawURLEncoding` 等）   |
| Zig      | `std.fmt.hex`                | `std.base64`（`standard`、`url_safe_no_pad` 等）          |
| Nabi      | `hex::encode`、`hex::decode` | `base64::encode`、`base64::decode`（`STANDARD`、`URL_SAFE_NO_PAD` 等编解码器） |

Nabi 遵循 Go 和 Zig 的做法：每种编码一个模块，变体作为编解码器值而不是标志，常见情况用 `str` 形式，另有不做任何分配的缓冲区形式（`_into`）。填充策略同时规定编码和解码两侧，与 Kotlin 的 `PaddingOption` 一样，所以 `STANDARD` 写出填充，同时仍然接受不带填充的文本。
