# 2.21. `crypto::`：密碼學演算法

`core::crypto` 以純計算的形式提供 TLS 堆疊所需的演算法：它們在每個目標上的行為完全相同，因此位於 `core`（0.1）。亂數需要作業系統，屬於 `std::random`（3.9）；需要隨機輸入的函式（金鑰、隨機數（nonce）、封裝）以引數接受它。對秘密資料的運算以 2.21.5 定義的常數時間執行。

在 `core::crypto` 中唯一的名稱（`Secret`、`constant_time_eq`、`HashAlgorithm`、`Digest`、SHA-3 與 SHAKE 的狀態、`Hmac`、`AeadAlgorithm`、`Aead`、`EcCurve`）可以透過 `core::crypto::` 取得；註解指出同樣匯出它們的子模組。金鑰與 HKDF 只能透過各自的演算法模組取得（`crypto::ed25519::SigningKey`、`crypto::hkdf::extract`），因為有好幾個模組都有 `PublicKey`。

每個位元組引數都對 `AsBytes` 泛型（[2.11.13](211-conversion.md#21113-asbytes-trait位元組作為檢視)）：`str` 提供其 UTF-8 位元組，`List<u8>`、`Slice<u8>` 或 `[u8; N]` 提供其位元組，`Secret` 就地提供其位元組，程式自己的型別則提供其 `byte_slice` 回傳的內容。同一個函式的多個位元組引數各有自己的型別參數，因此金鑰與訊息不必是相同型別。

```nabi
struct Secret {                                     // crypto::secret
    fn zeroed(len: usize) -> Secret;
    fn new<B: AsBytes>(bytes: B) -> Secret;                 // a copy
    fn len(self) -> usize;
    fn as_bytes(self) -> Slice<u8>;                 // the bytes in place, not a copy
    fn wipe(self);                                  // overwrite with zeros now
}
impl Index<usize> for Secret; impl IndexSet<usize, u8> for Secret;   // s[i], s[i] = b (bounds-checked)
impl Sliceable<Range<usize>> for Secret;           // s[a:b]: a view in place
impl Drop for Secret;                               // using release wipes
impl Debug for Secret;                              // "Secret { len: n, .. }", never the bytes
impl AsBytes for Secret;                            // byte_slice: as_bytes (2.11.13)
fn constant_time_eq<A: AsBytes, B: AsBytes>(a: A, b: B) -> bool;   // time depends only on the lengths
```

`Secret` 是位於秘密儲存區（[2.21.5](#2215-安全性考量)）中的固定長度位元組字串。與 `List` 一樣，複本共用位元組，而且它不是 `Sync`。與集合型別相同，移交給另一個執行緒時，新的擁有者會在秘密儲存區中取得自己的複本，而 `par` 會逐一執行觸及同一個 `Secret` 的項目（[2.13.9](213-marker.md#2139-send--sync執行緒邊界標記)）。由 `as_bytes` 或切片取得的檢視直接讀取儲存區本身：它看得到之後的寫入，且秘密被抹除後讀到的是零。`Secret` 沒有 `==`；請用 `constant_time_eq` 比較。

## 2.21.1. 雜湊、HMAC、HKDF

```nabi
#[non_exhaustive]
enum HashAlgorithm { Sha256, Sha384, Sha512, Sha3_256, Sha3_512 }   // crypto (FIPS 180-4, FIPS 202); Copy, Eq, Hash, Debug
impl HashAlgorithm {
    fn output_len(self) -> usize;                   // 32, 48, 64, 32, 64
    fn block_len(self) -> usize;                    // 64, 128, 128; SHA-3: the rate, 136 / 72
    fn digest<B: AsBytes>(self, data: B) -> List<u8>;
}
struct Digest {                                     // crypto::sha2; a running hash, a copy forks it
    fn new(alg: HashAlgorithm) -> Digest;
    fn algorithm(self) -> HashAlgorithm;
    fn update<B: AsBytes>(ref self, data: B);
    fn finish(self) -> List<u8>;
}
struct Sha3_256 / Sha3_512 {                        // crypto::sha3 (FIPS 202), the low-level form
    const OUTPUT_LEN: usize;                        // 32 / 64
    fn new() -> Self; fn update<B: AsBytes>(ref self, data: B); fn finish(self) -> List<u8>;
    fn digest<B: AsBytes>(data: B) -> List<u8>;
}
struct Shake128 / Shake256 {
    fn new() -> Self; fn update<B: AsBytes>(ref self, data: B);
    fn squeeze(ref self, n: usize) -> List<u8>;     // the next n output bytes
}
struct Hmac {                                       // crypto::hmac (RFC 2104)
    fn new<B: AsBytes>(alg: HashAlgorithm, key: B) -> Hmac;
    fn update<B: AsBytes>(ref self, data: B);
    fn finish(self) -> List<u8>;
    fn mac<K: AsBytes, D: AsBytes>(alg: HashAlgorithm, key: K, data: D) -> List<u8>;
    fn verify<K: AsBytes, D: AsBytes, T: AsBytes>(alg: HashAlgorithm, key: K, data: D, tag: T) -> bool;   // constant time
}
mod hkdf {                                          // crypto::hkdf (RFC 5869)
    fn extract<S: AsBytes, I: AsBytes>(alg: HashAlgorithm, salt: S, ikm: I) -> Secret;
    fn expand<P: AsBytes, I: AsBytes>(alg: HashAlgorithm, prk: P, info: I, len: usize) throws (ValueError) -> Secret;
}
```

`HashAlgorithm` 列出參與金鑰化與簽章 API 的每個雜湊函式：`Digest`、`Hmac`、`hkdf`、ECDSA 與 RSA（PKCS #1 v1.5 與 PSS，包含 MGF1）都接受每個變體。它是 `#[non_exhaustive]`（[refs-lang 6.2.10](../../refs-lang/006-meta-system/602-built-in-attributes.md#6210-non_exhaustive-opting-out-of-external-exhaustiveness)）：在 `core` 之外對它進行 `match` 需要 `_` 分支，之後的版本可以加入新的函式。對於 SHA-3 函式，HMAC 的區塊是海綿結構的速率（rate）。`Sha3_256`、`Sha3_512`、`Shake128` 與 `Shake256` 是低階形式：每個都是在編譯時固定的一個函式（SHAKE 沒有 `HashAlgorithm` 變體，因為它的輸出長度由呼叫端決定）；`Sha3_256::digest(x)` 與 `HashAlgorithm::Sha3_256.digest(x)` 計算的是同一個函式。

`Digest`、`Hmac` 以及 SHA-3 / SHAKE 的狀態都是值：複本會各自獨立地繼續（例如在中途讀取的對話紀錄雜湊）。連續的 `squeeze` 呼叫延續同一個輸出串流；在第一次 `squeeze` 之後呼叫 `update` 會觸發 panic。`Hmac::verify` 以常數時間比較完整長度的標籤。`hkdf::extract` 在 salt 為空時使用 `output_len` 個零位元組；`hkdf::expand` 在 `len > 255 * output_len` 時擲回 `ValueError`。

## 2.21.2. AEAD

```nabi
#[non_exhaustive]
enum AeadAlgorithm { Aes128Gcm, Aes256Gcm, ChaCha20Poly1305 }   // crypto::aead; Copy, Eq, Hash, Debug
impl AeadAlgorithm { fn key_len(self) -> usize; fn nonce_len(self) -> usize; fn tag_len(self) -> usize; }
struct Aead {
    fn new<B: AsBytes>(alg: AeadAlgorithm, key: B) throws (ValueError) -> Aead;
    fn algorithm(self) -> AeadAlgorithm;
    fn seal<N: AsBytes, A: AsBytes, P: AsBytes>(self, nonce: N, aad: A, plaintext: P) throws (ValueError) -> List<u8>;   // ciphertext ‖ 16-byte tag
    fn open<N: AsBytes, A: AsBytes, C: AsBytes>(self, nonce: N, aad: A, ciphertext_and_tag: C) throws (ValueError) -> List<u8> | none;  // none: not authentic
    fn wipe(self);                                  // overwrite the key material with zeros
}
impl Drop for Aead;
```

`Aead` 把展開後的金鑰（AES 輪金鑰與 GHASH 金鑰，或 ChaCha20 金鑰）保存在秘密儲存區中；複本共用它，因此 `wipe` 會使所有複本失效。

AES-GCM 遵循 NIST SP 800-38D 並使用 96 位元的 nonce，ChaCha20-Poly1305 遵循 RFC 8439。nonce 為 12 個位元組，標籤為 16 個位元組；金鑰或 nonce 的長度錯誤會是 `ValueError`，訊息長度超過單一 nonce 所容許的範圍也是（32 位元的區塊計數器不得環繞：AES-GCM 為 2^36 − 32 個位元組，ChaCha20-Poly1305 為 2^38 − 64）。`open` 對短於一個標籤的輸入回傳 `none`，並在釋出任何明文之前以常數時間檢查標籤。同一把金鑰下 nonce 絕不能重複；`Aead` 不追蹤 nonce。

## 2.21.3. 金鑰交換

```nabi
mod x25519 {                                        // crypto::x25519 (RFC 7748)
    struct PrivateKey {                             // 32 bytes in secret storage
        fn from_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> PrivateKey;
        fn to_secret(self) -> Secret;               // a copy of the 32 bytes
        fn public_key(self) -> PublicKey;
        fn diffie_hellman(self, peer: PublicKey) -> Secret | none;   // none: all-zero result
        fn wipe(self);
    }
    impl Drop for PrivateKey; impl Debug for PrivateKey;   // release wipes; Debug: PrivateKey { .. }
    struct PublicKey {                              // Copy, Eq, Hash, Debug
        fn from_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> PublicKey;
        fn to_bytes(self) -> [u8; 32];
    }
}
mod mlkem {                                         // crypto::mlkem (FIPS 203)
    const SHARED_KEY_LEN: usize = 32;
    struct DecapsulationKey768 {                    // in secret storage
        fn from_seed<D: AsBytes, Z: AsBytes>(d: D, z: Z) throws (ValueError) -> DecapsulationKey768;   // 32-byte seeds
        fn from_expanded_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> DecapsulationKey768;   // 2400 bytes
        fn to_expanded_bytes(self) -> Secret;
        fn encapsulation_key(self) -> EncapsulationKey768;
        fn decapsulate<C: AsBytes>(self, c: C) throws (ValueError) -> Secret;   // implicit rejection
        fn wipe(self);
    }
    impl Drop for DecapsulationKey768; impl Debug for DecapsulationKey768;   // DecapsulationKey768 { .. }
    struct EncapsulationKey768 {                    // Clone, Eq, Hash, Debug
        const LEN: usize = 1184;
        const CIPHERTEXT_LEN: usize = 1088;
        fn from_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> EncapsulationKey768;
        fn to_bytes(self) -> List<u8>;
        fn encapsulate<M: AsBytes>(self, m: M) throws (ValueError) -> (Secret, List<u8>);   // (key, ciphertext) from 32-byte m
    }
}
```

X25519 金鑰為 32 個位元組（否則為 `ValueError`）；私密金鑰在使用時會被夾限（clamp），對方的 u 座標依照 RFC 7748 第 5 節的規定解讀（忽略第 255 位元，大於或等於 p 的值會被約化），因此小階點或扭曲曲線上的點不是錯誤：`diffie_hellman` 恰好在結果全為零時回傳 `none`（RFC 7748 6.1）。ML-KEM 運算是 FIPS 203 的確定性內部形式：呼叫端從亂數來源提供 32 位元組的種子 `d`、`z` 與 `m`。解封裝金鑰由其種子建立（`from_seed`），或以低階形式由其 2400 位元組的展開編碼 `dk_PKE ‖ ek ‖ H(ek) ‖ z` 建立（`from_expanded_bytes`，FIPS 203 7.3），其中儲存的 `H(ek)` 必須相符（`ValueError`）。`EncapsulationKey768::from_bytes` 執行 FIPS 203 7.2 的輸入檢查（長度與模數，`ValueError`），`decapsulate` 執行 7.3 的密文長度檢查；無法重新加密回自身的密文會得到隱式拒絕（implicit rejection）金鑰，而不是錯誤。

## 2.21.4. 簽章

```nabi
mod ed25519 {                                       // crypto::ed25519 (RFC 8032)
    struct SigningKey {                             // the seed and its expansion in secret storage
        fn from_seed<B: AsBytes>(seed: B) throws (ValueError) -> SigningKey;   // 32 bytes
        fn seed(self) -> Secret;                    // a copy
        fn verifying_key(self) -> VerifyingKey;
        fn sign<B: AsBytes>(self, msg: B) -> List<u8>;   // 64 bytes, deterministic
        fn wipe(self);
    }
    impl Drop for SigningKey; impl Debug for SigningKey;   // SigningKey { verifying_key: .., .. }
    struct VerifyingKey {                           // Copy, Eq, Hash, Debug
        fn from_bytes<B: AsBytes>(bytes: B) throws (ValueError) -> VerifyingKey;   // 32 bytes, a canonical point
        fn to_bytes(self) -> [u8; 32];
        fn verify<M: AsBytes, S: AsBytes>(self, msg: M, sig: S) -> bool;
    }
}
#[non_exhaustive]
enum EcCurve { P256, P384 }                         // crypto::ecdsa (FIPS 186-5); Copy, Eq, Hash, Debug
mod ecdsa {
    struct SigningKey {                             // the scalar in secret storage
        fn from_bytes<B: AsBytes>(curve: EcCurve, bytes: B) throws (ValueError) -> SigningKey;   // big-endian scalar
        fn curve(self) -> EcCurve;
        fn verifying_key(self) -> VerifyingKey;
        fn sign<M: AsBytes>(self, hash: HashAlgorithm, msg: M) -> List<u8>;   // DER, RFC 6979 nonce
        fn wipe(self);
    }
    impl Drop for SigningKey; impl Debug for SigningKey;   // SigningKey { curve: P256, .. }
    struct VerifyingKey {                           // Copy, Eq, Hash, Debug
        fn from_sec1<B: AsBytes>(curve: EcCurve, bytes: B) throws (ValueError) -> VerifyingKey;   // SEC1 uncompressed
        fn curve(self) -> EcCurve;
        fn to_sec1(self) -> List<u8>;
        fn verify<M: AsBytes, S: AsBytes>(self, hash: HashAlgorithm, msg: M, signature: S) -> bool;   // DER signature
    }
}
mod rsa {                                           // crypto::rsa (RFC 8017), 2048..4096 bits
    struct PublicKey {                              // Clone, Eq, Hash, Debug
        fn new<N: AsBytes, E: AsBytes>(n: N, e: E) throws (ValueError) -> PublicKey;
        fn from_pkcs1_der<B: AsBytes>(der: B) throws (ValueError) -> PublicKey;
        fn bits(self) -> usize;
        fn modulus(self) -> List<u8>;               // big-endian, no leading zero bytes
        fn exponent(self) -> List<u8>;
        fn verify_pkcs1v15<M: AsBytes, S: AsBytes>(self, hash: HashAlgorithm, msg: M, signature: S) -> bool;
        fn verify_pss<M: AsBytes, S: AsBytes>(self, hash: HashAlgorithm, msg: M, signature: S, salt_len: usize | none = none) -> bool;  // MGF1(hash); none: salt length = hash length
    }
    struct PrivateKey {
        fn from_pkcs1_der<B: AsBytes>(der: B) throws (ValueError) -> PrivateKey;   // RSAPrivateKey, two primes
        fn public_key(self) -> PublicKey;
        fn bits(self) -> usize;
        fn sign_pkcs1v15<B: AsBytes>(self, hash: HashAlgorithm, msg: B) throws (ValueError) -> List<u8>;  // deterministic
        fn sign_pss<M: AsBytes, S: AsBytes>(self, hash: HashAlgorithm, msg: M, salt: S) throws (ValueError) -> List<u8>;  // salt: fresh random, hash length
        fn wipe(self);
    }
    impl Drop for PrivateKey; impl Debug for PrivateKey;   // PrivateKey { bits: 2048, .. }
}
```

私密金鑰與簽署金鑰把秘密材料保存在秘密儲存區中（[2.21.5](#2215-安全性考量)）：複本共用它，它們不是 `Sync`，`wipe` 與 `using` 的釋放會用零覆寫它，而 `Debug` 不顯示任何金鑰位元組：它以記錄形式 `Type { field: .., .. }` 印出型別，只包含公開的資訊（長度、曲線、大小、公開金鑰），其餘以 `..` 表示（[0.5.12](../000-meta/005-notation.md#0512-trait-涵蓋範圍non_exhaustive-與-sealed)）。金鑰在建立時檢查一次，因此 Ed25519 與 ECDSA 的簽署不會失敗。金鑰由位元組建立；亂數來源由呼叫端提供（`SigningKey::from_seed(random::secret(32))`，3.9）。

Ed25519 驗證不使用餘因子（cofactorless），並拒絕 `R` 的非正規編碼以及 `S ≥ L`；`ed25519::VerifyingKey::from_bytes` 拒絕非正規的 `A` 或不在曲線上的 `A`。`ecdsa::VerifyingKey::from_sec1` 拒絕不是曲線上座標小於 p 的未壓縮點的公開金鑰，而 ECDSA 驗證拒絕不是最小 DER 編碼、或 `r`、`s` 不在 `[1, n)` 範圍內的簽章。RSA 驗證要求簽章長度恰好等於模數長度且小於模數；PKCS #1 v1.5 會比較整個重建的編碼。`rsa::PublicKey::new` 接受 2048 到 4096 位元的奇數模數，以及滿足 `3 ≤ e < 2^33` 的奇數指數（忽略開頭的零位元組）；其他情況都是 `ValueError`（驗證時每個指數位元要做一次模平方，所以不得讓對方選擇過長的指數）。

ECDSA 簽署金鑰是不超過曲線大小的大端序純量 `1 <= d < n`（否則為 `ValueError`）；nonce 以確定性方式導出（RFC 6979，以 `hash` 進行 HMAC-DRBG），因此簽署不需要隨機輸入。RSA 簽署使用 CRT 並搭配底數盲化（盲化因子由金鑰、訊息以及 PSS 的 `salt` 導出，因此 PKCS #1 v1.5 簽署不需要隨機輸入），並在回傳結果之前以公開金鑰檢查結果；不一致的金鑰是 `ValueError`。PSS 驗證接受任何能容納於模數的 salt 長度（`salt_len`）；簽署使用與雜湊等長的 salt。

## 2.21.5. 安全性考量

**狀態**。`core::crypto` 與 `net::tls`（[3.2.5](../300-std/302-net.md#325-tls-13nettls)）是實驗性的：它們依照公開的標準撰寫，以 NIST CAVP 與 Wycheproof 的測試向量測試，並檢查過秘密輸入之間的時間差異，但尚未經過獨立的安全稽核。在涉及真實秘密的場合，它們不能取代經過稽核的函式庫。

**常數時間**。在對秘密的運算中，沒有任何分支或記憶體索引依賴秘密資料：AES（加密與金鑰排程）、GHASH、ChaCha20、Poly1305、標籤比較、X25519、Ed25519 簽署、ECDSA 簽署與金鑰導出、RSA 私密金鑰運算、ML-KEM 金鑰產生與解封裝、HMAC 與 HKDF。AES 以位元切片（bitslice）實作（沒有查表），GHASH 以稀疏運算元的整數乘積進行乘法（沒有表格），純量乘法會掃描每個項目來讀取預先計算的表格，反元素以公開指數的冪運算（ECDSA、Ed25519）或固定長度的二進位 GCD（RSA 盲化）計算，RSA 冪運算對每個指數位元都在質數的完整寬度上做平方與乘法，ML-KEM 以乘法加位移而非除法進行約化，而條件選擇則使用編譯器無法證明為 0 或全為 1 的遮罩，因此編譯器無法把它們變回分支。執行時間只依賴公開的值：長度、演算法、曲線與金鑰大小，以及本來就是公開的結果（無法驗證的標籤或簽章、超出範圍的 RFC 6979 候選 nonce）。無論密文是否能重新加密，ML-KEM 解封裝都走相同的路徑；隱式拒絕金鑰以遮罩選取。

這項保證有其限制，實作會明確說明而不隱藏：
- 它假設目標的 32 × 32 → 64 位元整數乘法以及位移與旋轉以常數時間執行（目前的 x86-64、AArch64、ARMv7-A 與 i686 核心皆是如此；部分微控制器與較舊的嵌入式核心則否），而在 wasm 目標上，它的可靠程度只取決於引擎的程式碼產生；
- 它涵蓋透過控制流程與記憶體位址可見的時間差異，不涵蓋功耗、電磁或故障注入攻擊，也不涵蓋推測執行的旁路通道；
- 剖析（DER、PEM、金鑰檔案）與公開資料的驗證是可變時間的；
- 它是由隨附工具鏈編譯出的實作所具有的性質，不是語言提供的保證：以相同風格撰寫的使用者程式碼不在涵蓋範圍內。

**記憶體中的金鑰材料**。秘密位元組存放在*秘密儲存區*中：這是垃圾回收器在大型物件空間（[refs-lang 7.1.2](../../refs-lang/007-runtime/701-object-layout.md#712-memory-spaces)）中配置的緩衝區，從不移動或複製，並在釋放記憶體之前用零覆寫。`Secret`（2.21）是它的公開形式。

- `Secret` 的位元組在堆積中只存在於一個地方；移交給另一個執行緒（2.13.9）時會在秘密儲存區中建立第二個 `Secret`，並依相同的規則抹除。它們會被 `wipe` 用零覆寫、在 `using` 繫結釋放 `Secret` 時（直接或透過 drop glue）被覆寫，最遲也會在釋放該緩衝區的回收中被覆寫。歸零使用的是最佳化器不得移除的儲存操作。
- 演算法只把依賴金鑰的狀態保存在秘密儲存區或區域變數中，從不放在普通的堆積物件裡。這些狀態包括 AES 輪金鑰、GHASH 金鑰與 ChaCha20 金鑰（`Aead`），HMAC 與 HKDF（填補後的金鑰、鏈結值、輸出區塊），X25519 私密金鑰以及 X25519 與 ML-KEM 的共用金鑰，ML-KEM 解封裝金鑰及其秘密多項式、種子與訊息，Ed25519 的種子、展開金鑰與 nonce 雜湊，ECDSA 純量，以及 RSA 的質數、CRT 指數與 `q^-1`。結果為金鑰材料的函式回傳 `Secret`。秘密儲存區中的暫存值在函式回傳前抹除，除非函式擲回錯誤；此時由回收來抹除它們。
- `net::tls`（[3.2.5](../300-std/302-net.md#325-tls-13nettls)）把它的流量、交握與恢復秘密，記錄金鑰，金鑰交換秘密，票證 PSK 以及 Ed25519 / EC 私密金鑰保存在秘密儲存區中，並在 KeyUpdate、交握結束與關閉時抹除它們。

這項保證有其限制，實作會明確說明而不隱藏：

- **暫存器與堆疊**。區域變數存放在暫存器與執行緒的堆疊上，不會被清除。這包括固定長度陣列與結構值，例如 `Digest`、`Hmac`、SHA-3 與加密器的狀態，以及 X25519、Ed25519 與 ECDSA 的體元素與純量。編譯器產生的溢出（spill）也不會被清除；槽位只有在堆疊被重複使用時才會被覆寫。沒有清除它們的程式碼產生機制，而 Rust 的 `zeroize`、Go 的 `crypto/subtle` 輔助函式以及 libsodium 的 `sodium_memzero` 同樣只能觸及交給它們的記憶體，無法觸及編譯器產生的複本。
- **普通物件**。存放在堆積物件中的值就存在於該物件中：垃圾回收器可能在疏散時複製它，並在不清除的情況下釋放它。這涵蓋 `List` 的元素、GC 持有之結構的欄位、閉包環境，以及跨越暫停點的 `async` / 產生器框架（[refs-lang 7.8.2](../../refs-lang/007-runtime/708-coroutine.md#782-coroutine-frame)）。把 `Secret` 的位元組複製出來（`as_bytes().to_list()`、`str`、格式化）會產生普通的複本。
- **普通的輸入與輸出**。不在涵蓋範圍內的有：以 `List<u8>` 或 `str` 傳入的金鑰、明文與密文、`Hmac` 標籤、`Digest` 的輸出，以及解碼後的金鑰檔案（PEM / DER 解碼在 `PrivateKey` 複製金鑰之前會產生普通串列）。
- **簽署**。ECDSA 與 RSA 簽署不在涵蓋範圍內：RFC 6979 的狀態與 `bignum` 的 limb 是普通串列，而 RSA 運算是在普通串列上計算（`rsa::PrivateKey` 把質數、CRT 指數與 `q^-1` 保存在秘密儲存區中，並在每次運算時重建模數）。
- **未鎖定在 RAM 中**。秘密儲存區沒有被鎖定（`mlock` / `VirtualLock`）。它可能被寫入置換空間或休眠檔案，並出現在核心傾印與行程快照中。鎖定以分頁為單位、每個行程有上限（RLIMIT_MEMLOCK），解除也以分頁為單位，因此需要一個自己的分頁粒度配置器。
- **行程結束與 wasm**。因行程結束而釋放的記憶體不會被抹除。在 wasm 目標上，主機可以隨時讀取線性記憶體。

**亂數**。演算法以引數接受每一個隨機輸入；呼叫端從作業系統的 CSPRNG `std::random`（3.9）取得它。ML-KEM 的種子、X25519 與 Ed25519 的秘密、ECDSA 私密金鑰以及 PSS 的 salt 都必須是這類來源新產生的輸出。
