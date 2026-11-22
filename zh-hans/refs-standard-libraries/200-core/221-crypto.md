# 2.21. `crypto::`：密码算法

`core::crypto` 以纯计算的形式提供 TLS 栈所需的算法：它们在每个目标上的行为都相同，所以放在 `core` 中（0.1）。随机性需要操作系统，属于 `std::random`（3.9）；需要随机输入的函数（密钥、nonce、封装）把随机输入作为参数接受。对机密数据的操作按 2.21.5 的定义以常数时间运行。

在 `core::crypto` 中唯一的名称（`Secret`、`constant_time_eq`、`HashAlgorithm`、`Digest`、SHA-3 和 SHAKE 状态、`Hmac`、`AeadAlgorithm`、`Aead`、`EcCurve`）可以通过 `core::crypto::` 使用；注释中写出了同样导出它们的子模块。密钥和 HKDF 只能通过其算法模块访问（`crypto::ed25519::SigningKey`、`crypto::hkdf::extract`），因为好几个模块都有 `PublicKey`。

每个字节参数都对 `AsBytes` 泛型（[2.11.13](211-conversion.md#21113-asbytes-trait字节视图)）：`str` 给出其 UTF-8 字节，`List<u8>`、`Slice<u8>` 或 `[u8; N]` 给出其字节，`Secret` 原地给出其字节，程序自己的类型给出其 `byte_slice` 返回的内容。同一个函数的多个字节参数各有自己的类型参数，因此密钥与消息不必是同一类型。

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

`Secret` 是位于机密存储（[2.21.5](#2215-安全注意事项)）中的定长字节串。与 `List` 一样，副本共享字节，它不是 `Sync`。与集合一样，交接给另一个线程时，新的所有者会在机密存储中得到自己的副本；`par` 会让到达同一个 `Secret` 的元素逐个运行（[2.13.9](213-marker.md#2139-send--sync线程边界标记)）。来自 `as_bytes` 或切片的视图直接读取存储本身：它能看到之后的写入，机密被擦除后读到的是零。`Secret` 没有 `==`；用 `constant_time_eq` 比较。

## 2.21.1. 哈希、HMAC、HKDF

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

`HashAlgorithm` 列出参与带密钥 API 和签名 API 的每一个哈希函数：`Digest`、`Hmac`、`hkdf`、ECDSA 和 RSA（PKCS #1 v1.5 和 PSS，包括 MGF1）都接受其中每个变体。它是 `#[non_exhaustive]`（[refs-lang 6.2.10](../../refs-lang/006-meta-system/602-built-in-attributes.md#6210-non_exhaustive-退出外部穷尽性)）：在 `core` 之外对它 `match` 需要 `_` 分支，后续版本可以增加函数。对于 SHA-3 函数，HMAC 的块长是海绵结构的速率（rate）。`Sha3_256`、`Sha3_512`、`Shake128` 和 `Shake256` 是低层形式：每个都在编译时固定为一个函数（SHAKE 没有 `HashAlgorithm` 变体，因为其输出长度由调用方决定）；`Sha3_256::digest(x)` 与 `HashAlgorithm::Sha3_256.digest(x)` 计算的是同一个函数。

`Digest`、`Hmac` 以及 SHA-3 / SHAKE 状态都是值：副本独立地继续计算（例如在中途读取的握手记录哈希）。连续的 `squeeze` 调用延续同一条输出流；在第一次 `squeeze` 之后调用 `update` 会触发 panic。`Hmac::verify` 以常数时间比较完整长度的标签。`hkdf::extract` 在盐为空时使用 `output_len` 个零字节；`hkdf::expand` 在 `len > 255 * output_len` 时抛出 `ValueError`。

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

`Aead` 把扩展后的密钥（AES 轮密钥和 GHASH 密钥，或 ChaCha20 密钥）保存在机密存储中；副本共享它，所以 `wipe` 会让所有副本失效。

AES-GCM 遵循 NIST SP 800-38D，使用 96 位 nonce；ChaCha20-Poly1305 遵循 RFC 8439。nonce 为 12 字节，标签为 16 字节；密钥或 nonce 长度错误是 `ValueError`，超过一个 nonce 所允许长度的消息也是（32 位块计数器不得回绕：AES-GCM 为 2^36 − 32 字节，ChaCha20-Poly1305 为 2^38 − 64 字节）。`open` 对短于一个标签的输入返回 `none`，并在释放任何明文之前以常数时间检查标签。同一个密钥下 nonce 绝不能重复；`Aead` 不跟踪 nonce。

## 2.21.3. 密钥交换

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

X25519 密钥为 32 字节（否则为 `ValueError`）；私钥在使用时进行钳位（clamp），对端的 u 坐标按 RFC 7748 第 5 节的规定读取（忽略第 255 位，≥ p 的值取模约简），因此小阶点或扭曲点不算错误：`diffie_hellman` 恰好在结果全为零时返回 `none`（RFC 7748 6.1）。ML-KEM 操作采用 FIPS 203 的确定性内部形式：调用方从随机源提供 32 字节的种子 `d`、`z` 和 `m`。解封装密钥由其种子构造（`from_seed`），或者以低层形式由其 2400 字节的扩展编码 `dk_PKE ‖ ek ‖ H(ek) ‖ z` 构造（`from_expanded_bytes`，FIPS 203 7.3），其中存储的 `H(ek)` 必须匹配（`ValueError`）。`EncapsulationKey768::from_bytes` 执行 FIPS 203 7.2 的输入检查（长度和模数，`ValueError`），`decapsulate` 执行 7.3 的密文长度检查；重新加密后与自身不一致的密文会得到隐式拒绝密钥，而不是错误。

## 2.21.4. 签名

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

私钥和签名密钥把其机密材料保存在机密存储中（[2.21.5](#2215-安全注意事项)）：副本共享这些材料，它们不是 `Sync`，`wipe` 和 `using` 释放会用零覆盖这些材料，`Debug` 不显示任何密钥字节：它以记录形式 `Type { field: .., .. }` 打印类型，只包含公开信息（长度、曲线、大小、公钥），其余部分写作 `..`（[0.5.12](../000-meta/005-notation.md#0512-trait-覆盖non_exhaustive-与-sealed)）。密钥只在创建时检查一次，所以 Ed25519 和 ECDSA 签名不会失败。密钥由字节构造；随机源由调用方提供（`SigningKey::from_seed(random::secret(32))`，3.9）。

Ed25519 验证不使用余因子（cofactorless），并拒绝 `R` 的非规范编码以及 `S ≥ L`；`ed25519::VerifyingKey::from_bytes` 拒绝非规范的 `A` 或不在曲线上的 `A`。`ecdsa::VerifyingKey::from_sec1` 拒绝不是曲线上坐标小于 p 的未压缩点的公钥，ECDSA 验证拒绝不是最小 DER 编码、或者 `r` 或 `s` 不在 `[1, n)` 中的签名。RSA 验证要求签名长度恰好等于模数长度且小于模数；PKCS #1 v1.5 比较完整重建的编码。`rsa::PublicKey::new` 接受 2048 到 4096 位的奇数模数和满足 `3 ≤ e < 2^33` 的奇数指数（忽略前导零字节）；其他任何输入都是 `ValueError`（验证时每个指数位要做一次模平方，所以不能让对端选择很长的指数）。

ECDSA 签名密钥是大端标量 `1 <= d < n`，长度不超过曲线的大小（否则为 `ValueError`）；nonce 以确定性方式派生（RFC 6979，使用 `hash` 的 HMAC-DRBG），所以签名不需要随机输入。RSA 签名使用带底数盲化（base blinding）的 CRT（盲化因子由密钥、消息以及 PSS 的 `salt` 派生，所以 PKCS #1 v1.5 签名不需要随机输入），并在返回之前用公钥检查结果；不一致的密钥是 `ValueError`。PSS 验证接受适合模数的任意盐长度（`salt_len`）；签名使用与哈希等长的盐。

## 2.21.5. 安全注意事项

**状态。** `core::crypto` 和 `net::tls`（[3.2.5](../300-std/302-net.md#325-tls-13nettls)）是实验性的：它们依据已发布的标准编写，以 NIST CAVP 和 Wycheproof 测试向量进行测试，并检查了不同机密输入之间的时间差异，但尚未经过独立的安全审计。在涉及真实机密的场合，它们不能替代经过审计的库。

**常数时间。** 在对机密进行的操作中，没有任何分支或内存索引依赖于机密数据：AES（加密和密钥编排）、GHASH、ChaCha20、Poly1305、标签比较、X25519、Ed25519 签名、ECDSA 签名和密钥派生、RSA 私钥操作、ML-KEM 密钥生成和解封装、HMAC 和 HKDF。AES 采用位切片（bitsliced）实现（不查表），GHASH 用稀疏操作数的整数乘积做乘法（无表），标量乘法通过扫描每个表项来读取其预计算表，求逆通过公开指数的幂运算（ECDSA、Ed25519）或定长二进制 GCD（RSA 盲化）完成，RSA 幂运算对每个指数位都在素数的完整宽度上做平方和乘法，ML-KEM 用乘法加移位而不是除法来约简，条件选择使用编译器无法证明为 0 或全 1 的掩码，因此编译器无法把它们变回分支。运行时间只取决于公开的值：长度、算法、曲线和密钥大小，以及本来就公开的结果（无法通过验证的标签或签名，超出范围的 RFC 6979 候选 nonce）。无论密文能否重新加密成功，ML-KEM 解封装都走同一条路径；隐式拒绝密钥通过掩码选出。

这一保证有一些界限，实现会说明而不隐瞒：
- 它假设目标的 32 × 32 → 64 位整数乘法及其移位和循环移位以常数时间运行（当前的 x86-64、AArch64、ARMv7-A 和 i686 核心如此；某些微控制器和较旧的嵌入式核心并非如此），在 wasm 目标上，它的可靠程度取决于引擎的代码生成；
- 它涵盖通过控制流和内存地址可观察到的时间差异，不涵盖功耗、电磁或故障注入攻击，也不涵盖推测执行侧信道；
- 解析（DER、PEM、密钥文件）以及对公开数据的验证是可变时间的；
- 这是由随附工具链编译出的实现所具有的性质，不是语言给出的保证：以同样风格编写的用户代码不在保证范围内。

**内存中的密钥材料。** 机密字节存放在*机密存储*中：这是收集器在大对象空间（[refs-lang 7.1.2](../../refs-lang/007-runtime/701-object-layout.md#712-内存空间)）中分配的缓冲区，从不移动或复制，并在释放内存之前用零覆盖。`Secret`（2.21）是它的公开形式。

- `Secret` 的字节在堆中只存在于一个位置；交接给另一个线程（2.13.9）会在机密存储中生成第二个 `Secret`，按同样的规则擦除。这些字节会在 `wipe` 时、在 `using` 绑定释放该 `Secret` 时（直接释放或通过 drop glue），以及最迟在释放该缓冲区的那次回收时被零覆盖。清零使用优化器不得删除的存储操作。
- 算法只在机密存储或局部变量中保存依赖于密钥的状态，从不放在普通堆对象中。这些状态包括 AES 轮密钥、GHASH 密钥和 ChaCha20 密钥（`Aead`），HMAC 和 HKDF（填充后的密钥、链值、输出块），X25519 私钥以及 X25519 和 ML-KEM 的共享密钥，ML-KEM 解封装密钥及其机密多项式、种子和消息，Ed25519 的种子、扩展密钥和 nonce 哈希，ECDSA 标量，以及 RSA 的素数、CRT 指数和 `q^-1`。结果为密钥材料的函数返回 `Secret`。机密存储中的临时值会在函数返回之前擦除，除非函数抛出错误；这时由回收来擦除。
- `net::tls`（[3.2.5](../300-std/302-net.md#325-tls-13nettls)）把其流量、握手和恢复机密、记录密钥、密钥交换机密、票据 PSK 以及 Ed25519 / EC 私钥保存在机密存储中，并在 KeyUpdate 时、握手结束时和关闭时擦除它们。

这一保证有一些界限，实现会说明而不隐瞒：

- **寄存器和栈。** 局部变量存放在寄存器和线程的栈上，不会被清除。这包括定长数组以及 `Digest`、`Hmac`、SHA-3 和密码状态等结构体值，以及 X25519、Ed25519 和 ECDSA 的域元素和标量。编译器产生的溢出（spill）也不会被清除；栈槽只在栈被复用时才被覆盖。没有清除它们的代码生成；Rust 的 `zeroize`、Go 的 `crypto/subtle` 辅助函数和 libsodium 的 `sodium_memzero` 同样只能触及交给它们的内存，触及不到编译器产生的副本。
- **普通对象。** 存储在堆对象中的值存在于该对象中：收集器可能在疏散（evacuation）期间复制它，并且不清除就释放它。这包括 `List` 元素、由 GC 持有的结构体字段、闭包环境，以及跨越挂起点的 `async` / 生成器帧（[refs-lang 7.8.2](../../refs-lang/007-runtime/708-coroutine.md#782-协程帧)）。把 `Secret` 的字节复制出来（`as_bytes().to_list()`、`str`、格式化）会产生普通副本。
- **普通输入和输出。** 不在保证范围内的有：以 `List<u8>` 或 `str` 传入的密钥、明文和密文、`Hmac` 标签、`Digest` 输出，以及解码后的密钥文件（PEM / DER 解码会先生成普通列表，然后 `PrivateKey` 才复制密钥）。
- **签名。** ECDSA 和 RSA 签名不在保证范围内：RFC 6979 状态和 `bignum` 的 limb 是普通列表，RSA 操作在普通列表上计算（`rsa::PrivateKey` 把其素数、CRT 指数和 `q^-1` 保存在机密存储中，并为每次操作重建模数）。
- **未锁定在内存中。** 机密存储没有锁定（`mlock` / `VirtualLock`）。它可能被写入交换区或休眠文件，并出现在核心转储和进程快照中。锁定以页为单位，受每进程限制（RLIMIT_MEMLOCK），也以页为单位解除，所以需要一个自己的页粒度分配器。
- **进程退出和 wasm。** 进程退出时释放的内存不会被擦除。在 wasm 目标上，宿主可以随时读取线性内存。

**随机性。** 算法把每个随机输入都作为参数接受；调用方从 `std::random`（3.9）即操作系统的 CSPRNG 获取它们。ML-KEM 种子、X25519 和 Ed25519 机密、ECDSA 私钥以及 PSS 盐必须是这种随机源的新鲜输出。
