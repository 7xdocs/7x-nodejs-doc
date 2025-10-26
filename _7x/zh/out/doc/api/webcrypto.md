# Web Crypto API

<!-- YAML
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59647
    description: KMAC algorithms are now supported.
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59544
    description: Argon2 algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59539
    description: AES-OCB algorithm is now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59569
    description: ML-KEM algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ChaCha20-Poly1305 algorithm is now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHA-3 algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHAKE algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ML-DSA algorithms are now supported.
  - version:
    - v23.5.0
    - v22.13.0
    pr-url: https://github.com/nodejs/node/pull/56142
    description: Algorithms `Ed25519` and `X25519` are now stable.
  - version:
    - v20.0.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/46067
    description: Arguments are now coerced and validated as per their WebIDL
      definitions like in other Web Crypto API implementations.
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44897
    description: No longer experimental except for the `Ed25519`, `Ed448`,
      `X25519`, and `X448` algorithms.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/43310
    description: Removed proprietary `'node.keyObject'` import/export format.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/43310
    description: Removed proprietary `'NODE-DSA'`, `'NODE-DH'`,
      and `'NODE-SCRYPT'` algorithms.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42507
    description: Added `'Ed25519'`, `'Ed448'`, `'X25519'`, and `'X448'`
      algorithms.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42507
    description: Removed proprietary `'NODE-ED25519'` and `'NODE-ED448'`
      algorithms.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42507
    description: Removed proprietary `'NODE-X25519'` and `'NODE-X448'` named
      curves from the `'ECDH'` algorithm.
-->

<!-- introduced_in=v15.0.0 -->

> Stability: 2 - Stable

Node.js 提供了 [Web Crypto API][] 标准的实现。

使用 `globalThis.crypto` 或 `require('node:crypto').webcrypto` 来访问此模块。

```js
const { subtle } = globalThis.crypto;

(async function() {

  const key = await subtle.generateKey({
    name: 'HMAC',
    hash: 'SHA-256',
    length: 256,
  }, true, ['sign', 'verify']);

  const enc = new TextEncoder();
  const message = enc.encode('I love cupcakes');

  const digest = await subtle.sign({
    name: 'HMAC',
  }, key, message);

})();
```

## Web 加密 API 中的现代算法

> Stability: 1.1 - Active development

Node.js 提供了以下来自
[Web 加密 API 中的现代算法](https://wicg.github.io/webcrypto-modern-algos/)
WICG 提案的特性的实现：

算法：

* `'AES-OCB'`[^openssl30]
* `'Argon2d'`[^openssl32]
* `'Argon2i'`[^openssl32]
* `'Argon2id'`[^openssl32]
* `'ChaCha20-Poly1305'`
* `'cSHAKE128'`
* `'cSHAKE256'`
* `'KMAC128'`[^openssl30]
* `'KMAC256'`[^openssl30]
* `'ML-DSA-44'`[^openssl35]
* `'ML-DSA-65'`[^openssl35]
* `'ML-DSA-87'`[^openssl35]
* `'ML-KEM-512'`[^openssl35]
* `'ML-KEM-768'`[^openssl35]
* `'ML-KEM-1024'`[^openssl35]
* `'SHA3-256'`
* `'SHA3-384'`
* `'SHA3-512'`

密钥格式：

* `'raw-public'`
* `'raw-secret'`
* `'raw-seed'`

方法：

* [`subtle.decapsulateBits()`][]
* [`subtle.decapsulateKey()`][]
* [`subtle.encapsulateBits()`][]
* [`subtle.encapsulateKey()`][]
* [`subtle.getPublicKey()`][]
* [`SubtleCrypto.supports()`][]

## Web 加密 API 中的安全曲线

> Stability: 1.1 - Active development

Node.js 提供了以下来自
[Web 加密 API 中的安全曲线](https://wicg.github.io/webcrypto-secure-curves/)
WICG 提案的特性的实现：

算法：

* `'Ed448'`
* `'X448'`

## 示例

### 生成密钥

{SubtleCrypto} 类可用于生成对称（秘密）密钥或非对称密钥对（公钥和私钥）。

#### AES 密钥

```js
const { subtle } = globalThis.crypto;

async function generateAesKey(length = 256) {
  const key = await subtle.generateKey({
    name: 'AES-CBC',
    length,
  }, true, ['encrypt', 'decrypt']);

  return key;
}
```

#### ECDSA 密钥对

```js
const { subtle } = globalThis.crypto;

async function generateEcKey(namedCurve = 'P-521') {
  const {
    publicKey,
    privateKey,
  } = await subtle.generateKey({
    name: 'ECDSA',
    namedCurve,
  }, true, ['sign', 'verify']);

  return { publicKey, privateKey };
}
```

#### Ed25519/X25519 密钥对

```js
const { subtle } = globalThis.crypto;

async function generateEd25519Key() {
  return subtle.generateKey({
    name: 'Ed25519',
  }, true, ['sign', 'verify']);
}

async function generateX25519Key() {
  return subtle.generateKey({
    name: 'X25519',
  }, true, ['deriveKey']);
}
```

#### HMAC 密钥

```js
const { subtle } = globalThis.crypto;

async function generateHmacKey(hash = 'SHA-256') {
  const key = await subtle.generateKey({
    name: 'HMAC',
    hash,
  }, true, ['sign', 'verify']);

  return key;
}
```

#### RSA 密钥对

```js
const { subtle } = globalThis.crypto;
const publicExponent = new Uint8Array([1, 0, 1]);

async function generateRsaKey(modulusLength = 2048, hash = 'SHA-256') {
  const {
    publicKey,
    privateKey,
  } = await subtle.generateKey({
    name: 'RSASSA-PKCS1-v1_5',
    modulusLength,
    publicExponent,
    hash,
  }, true, ['sign', 'verify']);

  return { publicKey, privateKey };
}
```

### 加密和解密

```js
const crypto = globalThis.crypto;

async function aesEncrypt(plaintext) {
  const ec = new TextEncoder();
  const key = await generateAesKey();
  const iv = crypto.getRandomValues(new Uint8Array(16));

  const ciphertext = await crypto.subtle.encrypt({
    name: 'AES-CBC',
    iv,
  }, key, ec.encode(plaintext));

  return {
    key,
    iv,
    ciphertext,
  };
}

async function aesDecrypt(ciphertext, key, iv) {
  const dec = new TextDecoder();
  const plaintext = await crypto.subtle.decrypt({
    name: 'AES-CBC',
    iv,
  }, key, ciphertext);

  return dec.decode(plaintext);
}
```

### 导出和导入密钥

```js
const { subtle } = globalThis.crypto;

async function generateAndExportHmacKey(format = 'jwk', hash = 'SHA-512') {
  const key = await subtle.generateKey({
    name: 'HMAC',
    hash,
  }, true, ['sign', 'verify']);

  return subtle.exportKey(format, key);
}

async function importHmacKey(keyData, format = 'jwk', hash = 'SHA-512') {
  const key = await subtle.importKey(format, keyData, {
    name: 'HMAC',
    hash,
  }, true, ['sign', 'verify']);

  return key;
}
```

### 包装和解包密钥

```js
const { subtle } = globalThis.crypto;

async function generateAndWrapHmacKey(format = 'jwk', hash = 'SHA-512') {
  const [
    key,
    wrappingKey,
  ] = await Promise.all([
    subtle.generateKey({
      name: 'HMAC', hash,
    }, true, ['sign', 'verify']),
    subtle.generateKey({
      name: 'AES-KW',
      length: 256,
    }, true, ['wrapKey', 'unwrapKey']),
  ]);

  const wrappedKey = await subtle.wrapKey(format, key, wrappingKey, 'AES-KW');

  return { wrappedKey, wrappingKey };
}

async function unwrapHmacKey(
  wrappedKey,
  wrappingKey,
  format = 'jwk',
  hash = 'SHA-512') {

  const key = await subtle.unwrapKey(
    format,
    wrappedKey,
    wrappingKey,
    'AES-KW',
    { name: 'HMAC', hash },
    true,
    ['sign', 'verify']);

  return key;
}
```

### 签名和验证

```js
const { subtle } = globalThis.crypto;

async function sign(key, data) {
  const ec = new TextEncoder();
  const signature =
    await subtle.sign('RSASSA-PKCS1-v1_5', key, ec.encode(data));
  return signature;
}

async function verify(key, signature, data) {
  const ec = new TextEncoder();
  const verified =
    await subtle.verify(
      'RSASSA-PKCS1-v1_5',
      key,
      signature,
      ec.encode(data));
  return verified;
}
```

### 派生比特和密钥

```js
const { subtle } = globalThis.crypto;

async function pbkdf2(pass, salt, iterations = 1000, length = 256) {
  const ec = new TextEncoder();
  const key = await subtle.importKey(
    'raw',
    ec.encode(pass),
    'PBKDF2',
    false,
    ['deriveBits']);
  const bits = await subtle.deriveBits({
    name: 'PBKDF2',
    hash: 'SHA-512',
    salt: ec.encode(salt),
    iterations,
  }, key, length);
  return bits;
}

async function pbkdf2Key(pass, salt, iterations = 1000, length = 256) {
  const ec = new TextEncoder();
  const keyMaterial = await subtle.importKey(
    'raw',
    ec.encode(pass),
    'PBKDF2',
    false,
    ['deriveKey']);
  const key = await subtle.deriveKey({
    name: 'PBKDF2',
    hash: 'SHA-512',
    salt: ec.encode(salt),
    iterations,
  }, keyMaterial, {
    name: 'AES-GCM',
    length,
  }, true, ['encrypt', 'decrypt']);
  return key;
}
```

### 摘要

```js
const { subtle } = globalThis.crypto;

async function digest(data, algorithm = 'SHA-512') {
  const ec = new TextEncoder();
  const digest = await subtle.digest(algorithm, ec.encode(data));
  return digest;
}
```

### 检查运行时算法支持

[`SubtleCrypto.supports()`][] 允许在 Web Crypto API 中进行特性检测，可用于检测给定的算法标识符（包括其参数）是否支持给定的操作。

此示例在可用时使用 Argon2 从密码派生密钥，否则使用 PBKDF2；然后使用 AES-OCB（如果可用）或 AES-GCM（否则）加密和解密一些文本。

```mjs
const { SubtleCrypto, crypto } = globalThis;

const password = 'correct horse battery staple';
const derivationAlg =
  SubtleCrypto.supports?.('importKey', 'Argon2id') ?
    'Argon2id' :
    'PBKDF2';
const encryptionAlg =
  SubtleCrypto.supports?.('importKey', 'AES-OCB') ?
    'AES-OCB' :
    'AES-GCM';
const passwordKey = await crypto.subtle.importKey(
  derivationAlg === 'Argon2id' ? 'raw-secret' : 'raw',
  new TextEncoder().encode(password),
  derivationAlg,
  false,
  ['deriveKey'],
);
const nonce = crypto.getRandomValues(new Uint8Array(16));
const derivationParams =
  derivationAlg === 'Argon2id' ?
    {
      nonce,
      parallelism: 4,
      memory: 2 ** 21,
      passes: 1,
    } :
    {
      salt: nonce,
      iterations: 100_000,
      hash: 'SHA-256',
    };
const key = await crypto.subtle.deriveKey(
  {
    name: derivationAlg,
    ...derivationParams,
  },
  passwordKey,
  {
    name: encryptionAlg,
    length: 256,
  },
  false,
  ['encrypt', 'decrypt'],
);
const plaintext = 'Hello, world!';
const iv = crypto.getRandomValues(new Uint8Array(16));
const encrypted = await crypto.subtle.encrypt(
  { name: encryptionAlg, iv },
  key,
  new TextEncoder().encode(plaintext),
);
const decrypted = new TextDecoder().decode(await crypto.subtle.decrypt(
  { name: encryptionAlg, iv },
  key,
  encrypted,
));
```

## 算法矩阵

下表详细说明了 Node.js Web Crypto API 实现支持的算法以及每种算法支持的 API：

### 密钥管理 API

| 算法                            | [`subtle.generateKey()`][] | [`subtle.exportKey()`][] | [`subtle.importKey()`][] | [`subtle.getPublicKey()`][] |
| ------------------------------ | -------------------------- | ------------------------ | ------------------------ | --------------------------- |
| `'AES-CBC'`                    | ✔                          | ✔                        | ✔                        |                             |
| `'AES-CTR'`                    | ✔                          | ✔                        | ✔                        |                             |
| `'AES-GCM'`                    | ✔                          | ✔                        | ✔                        |                             |
| `'AES-KW'`                     | ✔                          | ✔                        | ✔                        |                             |
| `'AES-OCB'`                    | ✔                          | ✔                        | ✔                        |                             |
| `'Argon2d'`                    |                            |                          | ✔                        |                             |
| `'Argon2i'`                    |                            |                          | ✔                        |                             |
| `'Argon2id'`                   |                            |                          | ✔                        |                             |
| `'ChaCha20-Poly1305'`[^modern-algos] | ✔                          | ✔                        | ✔                        |                             |
| `'ECDH'`                       | ✔                          | ✔                        | ✔                        | ✔                           |
| `'ECDSA'`                      | ✔                          | ✔                        | ✔                        | ✔                           |
| `'Ed25519'`                    | ✔                          | ✔                        | ✔                        | ✔                           |
| `'Ed448'`[^secure-curves]      | ✔                          | ✔                        | ✔                        | ✔                           |
| `'HKDF'`                       |                            |                          | ✔                        |                             |
| `'HMAC'`                       | ✔                          | ✔                        | ✔                        |                             |
| `'KMAC128'`[^modern-algos]     | ✔                          | ✔                        | ✔                        |                             |
| `'KMAC256'`[^modern-algos]     | ✔                          | ✔                        | ✔                        |                             |
| `'ML-DSA-44'`[^modern-algos]   | ✔                          | ✔                        | ✔                        | ✔                           |
| `'ML-DSA-65'`[^modern-algos]   | ✔                          | ✔                        | ✔                        | ✔                           |
| `'ML-DSA-87'`[^modern-algos]   | ✔                          | ✔                        | ✔                        | ✔                           |
| `'ML-KEM-512'`[^modern-algos]  | ✔                          | ✔                        | ✔                        | ✔                           |
| `'ML-KEM-768'`[^modern-algos]  | ✔                          | ✔                        | ✔                        | ✔                           |
| `'ML-KEM-1024'`[^modern-algos] | ✔                          | ✔                        | ✔                        | ✔                           |
| `'PBKDF2'`                     |                            |                          | ✔                        |                             |
| `'RSA-OAEP'`                   | ✔                          | ✔                        | ✔                        | ✔                           |
| `'RSA-PSS'`                    | ✔                          | ✔                        | ✔                        | ✔                           |
| `'RSASSA-PKCS1-v1_5'`          | ✔                          | ✔                        | ✔                        | ✔                           |
| `'X25519'`                     | ✔                          | ✔                        | ✔                        | ✔                           |
| `'X448'`[^secure-curves]       | ✔                          | ✔                        | ✔                        | ✔                           |

### 加密操作 API

**列说明：**

* **加密**: [`subtle.encrypt()`][] / [`subtle.decrypt()`][]
* **签名和 MAC**: [`subtle.sign()`][] / [`subtle.verify()`][]
* **密钥或比特派生**: [`subtle.deriveBits()`][] / [`subtle.deriveKey()`][]
* **密钥包装**: [`subtle.wrapKey()`][] / [`subtle.unwrapKey()`][]
* **密钥封装**: [`subtle.encapsulateBits()`][] / [`subtle.decapsulateBits()`][] /
  [`subtle.encapsulateKey()`][] / [`subtle.decapsulateKey()`][]
* **摘要**: [`subtle.digest()`][]

| 算法                            | 加密 | 签名和 MAC | 密钥或比特派生 | 密钥包装 | 密钥封装 | 摘要 |
| ------------------------------ | ---- | ---------- | -------------- | -------- | -------- | ---- |
| `'AES-CBC'`                    | ✔    |            |                | ✔        |          |      |
| `'AES-CTR'`                    | ✔    |            |                | ✔        |          |      |
| `'AES-GCM'`                    | ✔    |            |                | ✔        |          |      |
| `'AES-KW'`                     |      |            |                | ✔        |          |      |
| `'AES-OCB'`                    | ✔    |            |                | ✔        |          |      |
| `'Argon2d'`                    |      |            | ✔              |          |          |      |
| `'Argon2i'`                    |      |            | ✔              |          |          |      |
| `'Argon2id'`                   |      |            | ✔              |          |          |      |
| `'ChaCha20-Poly1305'`[^modern-algos] | ✔    |            |                | ✔        |          |      |
| `'cSHAKE128'`[^modern-algos]   |      |            |                |          |          | ✔    |
| `'cSHAKE256'`[^modern-algos]   |      |            |                |          |          | ✔    |
| `'ECDH'`                       |      |            | ✔              |          |          |      |
| `'ECDSA'`                      |      | ✔          |                |          |          |      |
| `'Ed25519'`                    |      | ✔          |                |          |          |      |
| `'Ed448'`[^secure-curves]      |      | ✔          |                |          |          |      |
| `'HKDF'`                       |      |            | ✔              |          |          |      |
| `'HMAC'`                       |      | ✔          |                |          |          |      |
| `'KMAC128'`[^modern-algos]     |      | ✔          |                |          |          |      |
| `'KMAC256'`[^modern-algos]     |      | ✔          |                |          |          |      |
| `'ML-DSA-44'`[^modern-algos]   |      | ✔          |                |          |          |      |
| `'ML-DSA-65'`[^modern-algos]   |      | ✔          |                |          |          |      |
| `'ML-DSA-87'`[^modern-algos]   |      | ✔          |                |          |          |      |
| `'ML-KEM-512'`[^modern-algos]  |      |            |                |          | ✔        |      |
| `'ML-KEM-768'`[^modern-algos]  |      |            |                |          | ✔        |      |
| `'ML-KEM-1024'`[^modern-algos] |      |            |                |          | ✔        |      |
| `'PBKDF2'`                     |      |            | ✔              |          |          |      |
| `'RSA-OAEP'`                   | ✔    |            |                | ✔        |          |      |
| `'RSA-PSS'`                    |      | ✔          |                |          |          |      |
| `'RSASSA-PKCS1-v1_5'`          |      | ✔          |                |          |          |      |
| `'SHA-1'`                      |      |            |                |          |          | ✔    |
| `'SHA-256'`                    |      |            |                |          |          | ✔    |
| `'SHA-384'`                    |      |            |                |          |          | ✔    |
| `'SHA-512'`                    |      |            |                |          |          | ✔    |
| `'SHA3-256'`[^modern-algos]    |      |            |                |          |          | ✔    |
| `'SHA3-384'`[^modern-algos]    |      |            |                |          |          | ✔    |
| `'SHA3-512'`[^modern-algos]    |      |            |                |          |          | ✔    |
| `'X25519'`                     |      |            | ✔              |          |          |      |
| `'X448'`[^secure-curves]       |      |            | ✔              |          |          |      |

## 类：`Crypto`

<!-- YAML
added: v15.0.0
-->

`globalThis.crypto` 是 `Crypto` 类的一个实例。`Crypto` 是一个单例，提供对加密 API 其余部分的访问。

### `crypto.subtle`

<!-- YAML
added: v15.0.0
-->

* 类型：{SubtleCrypto}

提供对 `SubtleCrypto` API 的访问。

### `crypto.getRandomValues(typedArray)`

<!-- YAML
added: v15.0.0
-->

* `typedArray` {Buffer|TypedArray}
* 返回：{Buffer|TypedArray}

生成密码学强随机值。给定的 `typedArray` 会被随机值填充，并返回对 `typedArray` 的引用。

给定的 `typedArray` 必须是基于整数的 {TypedArray} 实例，即不接受 `Float32Array` 和 `Float64Array`。

如果给定的 `typedArray` 大于 65,536 字节，将抛出错误。

### `crypto.randomUUID()`

<!-- YAML
added: v16.7.0
-->

* 返回：{string}

生成一个随机的 [RFC 4122][] 版本 4 UUID。UUID 是使用密码学伪随机数生成器生成的。

## 类：`CryptoKey`

<!-- YAML
added: v15.0.0
-->

### `cryptoKey.algorithm`

<!-- YAML
added: v15.0.0
-->

<!--lint disable maximum-line-length remark-lint-->

* 类型：{KeyAlgorithm|RsaHashedKeyAlgorithm|EcKeyAlgorithm|AesKeyAlgorithm|HmacKeyAlgorithm|KmacKeyAlgorithm}

<!--lint enable maximum-line-length remark-lint-->

一个对象，详细说明了密钥可以使用的算法以及额外的算法特定参数。

只读。

### `cryptoKey.extractable`

<!-- YAML
added: v15.0.0
-->

* 类型：{boolean}

当为 `true` 时，可以使用 [`subtle.exportKey()`][] 或 [`subtle.wrapKey()`][] 提取 {CryptoKey}。

只读。

### `cryptoKey.type`

<!-- YAML
added: v15.0.0
-->

* 类型：{string} 其中之一：`'secret'`、`'private'` 或 `'public'`。

一个字符串，标识密钥是对称密钥（`'secret'`）还是非对称密钥（`'private'` 或 `'public'`）。

### `cryptoKey.usages`

<!-- YAML
added: v15.0.0
-->

* 类型：{string\[]}

一个字符串数组，标识密钥可以用于的操作。

可能的用途包括：

* `'encrypt'` - 允许使用密钥与 [`subtle.encrypt()`][]
* `'decrypt'` - 允许使用密钥与 [`subtle.decrypt()`][]
* `'sign'` - 允许使用密钥与 [`subtle.sign()`][]
* `'verify'` - 允许使用密钥与 [`subtle.verify()`][]
* `'deriveKey'` - 允许使用密钥与 [`subtle.deriveKey()`][]
* `'deriveBits'` - 允许使用密钥与 [`subtle.deriveBits()`][]
* `'encapsulateBits'` - 允许使用密钥与 [`subtle.encapsulateBits()`][]
* `'decapsulateBits'` - 允许使用密钥与 [`subtle.decapsulateBits()`][]
* `'encapsulateKey'` - 允许使用密钥与 [`subtle.encapsulateKey()`][]
* `'decapsulateKey'` - 允许使用密钥与 [`subtle.decapsulateKey()`][]
* `'wrapKey'` - 允许使用密钥与 [`subtle.wrapKey()`][]
* `'unwrapKey'` - 允许使用密钥与 [`subtle.unwrapKey()`][]

有效的密钥用途取决于密钥算法（由 `cryptokey.algorithm.name` 标识）。

**列说明：**

* **加密**: [`subtle.encrypt()`][] / [`subtle.decrypt()`][]
* **签名和 MAC**: [`subtle.sign()`][] / [`subtle.verify()`][]
* **密钥或比特派生**: [`subtle.deriveBits()`][] / [`subtle.deriveKey()`][]
* **密钥包装**: [`subtle.wrapKey()`][] / [`subtle.unwrapKey()`][]
* **密钥封装**: [`subtle.encapsulateBits()`][] / [`subtle.decapsulateBits()`][] /
  [`subtle.encapsulateKey()`][] / [`subtle.decapsulateKey()`][]

| 支持的密钥算法              | 加密 | 签名和 MAC | 密钥或比特派生 | 密钥包装 | 密钥封装 |
| -------------------------- | ---- | ---------- | -------------- | -------- | -------- |
| `'AES-CBC'`                | ✔    |            |                | ✔        |          |
| `'AES-CTR'`                | ✔    |            |                | ✔        |          |
| `'AES-GCM'`                | ✔    |            |                | ✔        |          |
| `'AES-KW'`                 |      |            |                | ✔        |          |
| `'AES-OCB'`                | ✔    |            |                | ✔        |          |
| `'Argon2d'`                |      |            | ✔              |          |          |
| `'Argon2i'`                |      |            | ✔              |          |          |
| `'Argon2id'`               |      |            | ✔              |          |          |
| `'ChaCha20-Poly1305'`[^modern-algos] | ✔    |            |                | ✔        |          |
| `'ECDH'`                   |      |            | ✔              |          |          |
| `'ECDSA'`                  |      | ✔          |                |          |          |
| `'Ed25519'`                |      | ✔          |                |          |          |
| `'Ed448'`[^secure-curves]  |      | ✔          |                |          |          |
| `'HDKF'`                   |      |            | ✔              |          |          |
| `'HMAC'`                   |      | ✔          |                |          |          |
| `'KMAC128'`[^modern-algos] |      | ✔          |                |          |          |
| `'KMAC256'`[^modern-algos] |      | ✔          |                |          |          |
| `'ML-DSA-44'`[^modern-algos] |      | ✔          |                |          |          |
| `'ML-DSA-65'`[^modern-algos] |      | ✔          |                |          |          |
| `'ML-DSA-87'`[^modern-algos] |      | ✔          |                |          |          |
| `'ML-KEM-512'`[^modern-algos] |      |            |                |          | ✔        |
| `'ML-KEM-768'`[^modern-algos] |      |            |                |          | ✔        |
| `'ML-KEM-1024'`[^modern-algos] |      |            |                |          | ✔        |
| `'PBKDF2'`                 |      |            | ✔              |          |          |
| `'RSA-OAEP'`               | ✔    |            |                | ✔        |          |
| `'RSA-PSS'`                |      | ✔          |                |          |          |
| `'RSASSA-PKCS1-v1_5'`      |      | ✔          |                |          |          |
| `'X25519'`                 |      |            | ✔              |          |          |
| `'X448'`[^secure-curves]   |      |            | ✔              |          |          |

## 类：`CryptoKeyPair`

<!-- YAML
added: v15.0.0
-->

`CryptoKeyPair` 是一个简单的字典对象，具有 `publicKey` 和 `privateKey` 属性，表示一个非对称密钥对。

### `cryptoKeyPair.privateKey`

<!-- YAML
added: v15.0.0
-->

* 类型：{CryptoKey} 一个 {CryptoKey}，其 `type` 将为 `'private'`。

### `cryptoKeyPair.publicKey`

<!-- YAML
added: v15.0.0
-->

* 类型：{CryptoKey} 一个 {CryptoKey}，其 `type` 将为 `'public'`。

## 类：`SubtleCrypto`

<!-- YAML
added: v15.0.0
-->

### 静态方法：`SubtleCrypto.supports(operation, algorithm[, lengthOrAdditionalAlgorithm])`

<!-- YAML
added: v24.7.0
-->

> Stability: 1.1 - Active development

<!--lint disable maximum-line-length remark-lint-->

* `operation` {string} "encrypt", "decrypt", "sign", "verify", "digest", "generateKey", "deriveKey", "deriveBits", "importKey", "exportKey", "getPublicKey", "wrapKey", "unwrapKey", "encapsulateBits", "encapsulateKey", "decapsulateBits", 或 "decapsulateKey"
* `algorithm` {string|Algorithm}
* `lengthOrAdditionalAlgorithm` {null|number|string|Algorithm|undefined} 根据操作，这要么被忽略，要么是当操作为 "deriveBits" 时的长度参数值，当操作为 "deriveKey" 时要派生的密钥的算法，当操作为 "wrapKey" 时包装前要导出的密钥的算法，当操作为 "unwrapKey" 时解包后要导入的密钥的算法，或者当操作为 "encapsulateKey" 或 "decapsulateKey" 时封装/解封装密钥后要导入的密钥的算法。**默认值：** 当操作为 "deriveBits" 时为 `null`，否则为 `undefined`。
* 返回：{boolean} 指示实现是否支持给定的操作

<!--lint enable maximum-line-length remark-lint-->

允许在 Web Crypto API 中进行特性检测，可用于检测给定的算法标识符（包括其参数）是否支持给定的操作。

有关此方法的使用示例，请参阅[检查运行时算法支持][]。

### `subtle.decapsulateBits(decapsulationAlgorithm, decapsulationKey, ciphertext)`

<!-- YAML
added: v24.7.0
-->

> Stability: 1.1 - Active development

* `decapsulationAlgorithm` {string|Algorithm}
* `decapsulationKey` {CryptoKey}
* `ciphertext` {ArrayBuffer|TypedArray|DataView|Buffer}
* 返回：{Promise} 成功时以 {ArrayBuffer} 完成。

消息接收者使用其非对称私钥解密“封装密钥”（密文），从而恢复一个临时对称密钥（表示为 {ArrayBuffer}），然后使用该密钥解密消息。

当前支持的算法包括：

* `'ML-KEM-512'`[^modern-algos]
* `'ML-KEM-768'`[^modern-algos]
* `'ML-KEM-1024'`[^modern-algos]

### `subtle.decapsulateKey(decapsulationAlgorithm, decapsulationKey, ciphertext, sharedKeyAlgorithm, extractable, usages)`

<!-- YAML
added: v24.7.0
-->

> Stability: 1.1 - Active development

* `decapsulationAlgorithm` {string|Algorithm}
* `decapsulationKey` {CryptoKey}
* `ciphertext` {ArrayBuffer|TypedArray|DataView|Buffer}
* `sharedKeyAlgorithm` {string|Algorithm|HmacImportParams|AesDerivedKeyParams|KmacImportParams}
* `extractable` {boolean}
* `usages` {string\[]} 参见 [密钥用途][]。
* 返回：{Promise} 成功时以 {CryptoKey} 完成。

消息接收者使用其非对称私钥解密“封装密钥”（密文），从而恢复一个临时对称密钥（表示为 {CryptoKey}），然后使用该密钥解密消息。

当前支持的算法包括：

* `'ML-KEM-512'`[^modern-algos]
* `'ML-KEM-768'`[^modern-algos]
* `'ML-KEM-1024'`[^modern-algos]

### `subtle.decrypt(algorithm, key, data)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59539
    description: AES-OCB algorithm is now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ChaCha20-Poly1305 algorithm is now supported.
-->

* `algorithm` {RsaOaepParams|AesCtrParams|AesCbcParams|AeadParams}
* `key` {CryptoKey}
* `data` {ArrayBuffer|TypedArray|DataView|Buffer}
* 返回：{Promise} 成功时以 {ArrayBuffer} 完成。

使用 `algorithm` 中指定的方法和参数以及 `key` 提供的密钥材料，此方法尝试解密提供的 `data`。如果成功，返回的 Promise 将以包含明文结果的 {ArrayBuffer} 解析。

当前支持的算法包括：

* `'AES-CBC'`
* `'AES-CTR'`
* `'AES-GCM'`
* `'AES-OCB'`[^modern-algos]
* `'ChaCha20-Poly1305'`[^modern-algos]
* `'RSA-OAEP'`

### `subtle.deriveBits(algorithm, baseKey[, length])`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59544
    description: Argon2 algorithms are now supported.
  - version:
    - v22.5.0
    - v20.17.0
    - v18.20.5
    pr-url: https://github.com/nodejs/node/pull/53601
    description: The length parameter is now optional for `'ECDH'`, `'X25519'`,
                 and `'X448'`.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42507
    description: Added `'X25519'`, and `'X448'` algorithms.
-->

<!--lint disable maximum-line-length remark-lint-->

* `algorithm` {EcdhKeyDeriveParams|HkdfParams|Pbkdf2Params|Argon2Params}
* `baseKey` {CryptoKey}
* `length` {number|null} **默认值：** `null`
* 返回：{Promise} 成功时以 {ArrayBuffer} 完成。

<!--lint enable maximum-line-length remark-lint-->

使用 `algorithm` 中指定的方法和参数以及 `baseKey` 提供的密钥材料，此方法尝试生成 `length` 比特。

当 `length` 未提供或为 `null` 时，将生成给定算法的最大比特数。这允许用于 `'ECDH'`、`'X25519'` 和 `'X448'`[^secure-curves] 算法，对于其他算法，`length` 必须是一个数字。

如果成功，返回的 Promise 将以包含生成数据的 {ArrayBuffer} 解析。

当前支持的算法包括：

* `'Argon2d'`[^modern-algos]
* `'Argon2i'`[^modern-algos]
* `'Argon2id'`[^modern-algos]
* `'ECDH'`
* `'HKDF'`
* `'PBKDF2'`
* `'X25519'`
* `'X448'`[^secure-curves]

### `subtle.deriveKey(algorithm, baseKey, derivedKeyAlgorithm, extractable, keyUsages)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59544
    description: Argon2 algorithms are now supported.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42507
    description: Added `'X25519'`, and `'X448'` algorithms.
-->

<!--lint disable maximum-line-length remark-lint-->

* `algorithm` {EcdhKeyDeriveParams|HkdfParams|Pbkdf2Params|Argon2Params}
* `baseKey` {CryptoKey}
* `derivedKeyAlgorithm` {string|Algorithm|HmacImportParams|AesDerivedKeyParams|KmacImportParams}
* `extractable` {boolean}
* `keyUsages` {string\[]} 参见 [密钥用途][]。
* 返回：{Promise} 成功时以 {CryptoKey} 完成。

<!--lint enable maximum-line-length remark-lint-->

使用 `algorithm` 中指定的方法和参数以及 `baseKey` 提供的密钥材料，此方法尝试基于 `derivedKeyAlgorithm` 中的方法和参数生成一个新的 {CryptoKey}。

调用此方法等效于调用 [`subtle.deriveBits()`][] 生成原始密钥材料，然后将结果传入 [`subtle.importKey()`][] 方法，使用 `deriveKeyAlgorithm`、`extractable` 和 `keyUsages` 参数作为输入。

当前支持的算法包括：

* `'Argon2d'`[^modern-algos]
* `'Argon2i'`[^modern-algos]
* `'Argon2id'`[^modern-algos]
* `'ECDH'`
* `'HKDF'`
* `'PBKDF2'`
* `'X25519'`
* `'X448'`[^secure-curves]

### `subtle.digest(algorithm, data)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHA-3 algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHAKE algorithms are now supported.
-->

* `algorithm` {string|Algorithm|CShakeParams}
* `data` {ArrayBuffer|TypedArray|DataView|Buffer}
* 返回：{Promise} 成功时以 {ArrayBuffer} 完成。

使用 `algorithm` 标识的方法，此方法尝试生成 `data` 的摘要。如果成功，返回的 Promise 将以包含计算摘要的 {ArrayBuffer} 解析。

如果 `algorithm` 以 {string} 形式提供，则必须是以下之一：

* `'cSHAKE128'`[^modern-algos]
* `'cSHAKE256'`[^modern-algos]
* `'SHA-1'`
* `'SHA-256'`
* `'SHA-384'`
* `'SHA-512'`
* `'SHA3-256'`[^modern-algos]
* `'SHA3-384'`[^modern-algos]
* `'SHA3-512'`[^modern-algos]

如果 `algorithm` 以 {Object} 形式提供，则必须具有 `name` 属性，其值为上述之一。

### `subtle.encapsulateBits(encapsulationAlgorithm, encapsulationKey)`

<!-- YAML
added: v24.7.0
-->

> Stability: 1.1 - Active development

* `encapsulationAlgorithm` {string|Algorithm}
* `encapsulationKey` {CryptoKey}
* 返回：{Promise} 成功时以 {EncapsulatedBits} 完成。

使用消息接收者的非对称公钥加密临时对称密钥。此加密密钥是“封装密钥”，表示为 {EncapsulatedBits}。

当前支持的算法包括：

* `'ML-KEM-512'`[^modern-algos]
* `'ML-KEM-768'`[^modern-algos]
* `'ML-KEM-1024'`[^modern-algos]

### `subtle.encapsulateKey(encapsulationAlgorithm, encapsulationKey, sharedKeyAlgorithm, extractable, usages)`

<!-- YAML
added: v24.7.0
-->

> Stability: 1.1 - Active development

* `encapsulationAlgorithm` {string|Algorithm}
* `encapsulationKey` {CryptoKey}
* `sharedKeyAlgorithm` {string|Algorithm|HmacImportParams|AesDerivedKeyParams|KmacImportParams}
* `extractable` {boolean}
* `usages` {string\[]} 参见 [密钥用途][]。
* 返回：{Promise} 成功时以 {EncapsulatedKey} 完成。

使用消息接收者的非对称公钥加密临时对称密钥。此加密密钥是“封装密钥”，表示为 {EncapsulatedKey}。

当前支持的算法包括：

* `'ML-KEM-512'`[^modern-algos]
* `'ML-KEM-768'`[^modern-algos]
* `'ML-KEM-1024'`[^modern-algos]

### `subtle.encrypt(algorithm, key, data)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59539
    description: AES-OCB algorithm is now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ChaCha20-Poly1305 algorithm is now supported.
-->

* `algorithm` {RsaOaepParams|AesCtrParams|AesCbcParams|AeadParams}
* `key` {CryptoKey}
* `data` {ArrayBuffer|TypedArray|DataView|Buffer}
* 返回：{Promise} 成功时以 {ArrayBuffer} 完成。

使用 `algorithm` 指定的方法和参数以及 `key` 提供的密钥材料，此方法尝试加密 `data`。如果成功，返回的 Promise 将以包含加密结果的 {ArrayBuffer} 解析。

当前支持的算法包括：

* `'AES-CBC'`
* `'AES-CTR'`
* `'AES-GCM'`
* `'AES-OCB'`[^modern-algos]
* `'ChaCha20-Poly1305'`[^modern-algos]
* `'RSA-OAEP'`

### `subtle.exportKey(format, key)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59647
    description: KMAC algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59569
    description: ML-KEM algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ChaCha20-Poly1305 algorithm is now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ML-DSA algorithms are now supported.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42507
    description: Added `'Ed25519'`, `'Ed448'`, `'X25519'`, and `'X448'`
      algorithms.
  - version: v15.9.0
    pr-url: https://github.com/nodejs/node/pull/37203
    description: Removed `'NODE-DSA'` JWK export.
-->

* `format` {string} 必须是 `'raw'`、`'pkcs8'`、`'spki'`、`'jwk'`、`'raw-secret'`[^modern-algos]、`'raw-public'`[^modern-algos] 或 `'raw-seed'`[^modern-algos] 之一。
* `key` {CryptoKey}
* 返回：{Promise} 成功时以 {ArrayBuffer|Object} 完成。

如果支持，将给定密钥导出为指定格式。

如果 {CryptoKey} 不可提取，返回的 Promise 将被拒绝。

当 `format` 是 `'pkcs8'` 或 `'spki'` 且导出成功时，返回的 Promise 将以包含导出密钥数据的 {ArrayBuffer} 解析。

当 `format` 是 `'jwk'` 且导出成功时，返回的 Promise 将以符合 [JSON Web Key][] 规范的 JavaScript 对象解析。

| 支持的密钥算法              | `'spki'` | `'pkcs8'` | `'jwk'` | `'raw'` | `'raw-secret'` | `'raw-public'` | `'raw-seed'` |
| -------------------------- | -------- | --------- | ------- | ------- | -------------- | -------------- | ------------ |
| `'AES-CBC'`                |          |           | ✔       | ✔       | ✔              |                |              |
| `'AES-CTR'`                |          |           | ✔       | ✔       | ✔              |                |              |
| `'AES-GCM'`                |          |           | ✔       | ✔       | ✔              |                |              |
| `'AES-KW'`                 |          |           | ✔       | ✔       | ✔              |                |              |
| `'AES-OCB'`[^modern-algos] |          |           | ✔       |         | ✔              |                |              |
| `'ChaCha20-Poly1305'`[^modern-algos] |          |           | ✔       |         | ✔              |                |              |
| `'ECDH'`                   | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |
| `'ECDSA'`                  | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |
| `'Ed25519'`                | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |
| `'Ed448'`[^secure-curves]  | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |
| `'HMAC'`                   |          |           | ✔       | ✔       | ✔              |                |              |
| `'KMAC128'`[^modern-algos] |          |           | ✔       |         | ✔              |                |              |
| `'KMAC256'`[^modern-algos] |          |           | ✔       |         | ✔              |                |              |
| `'ML-DSA-44'`[^modern-algos] | ✔        | ✔         | ✔       |         |                | ✔              | ✔            |
| `'ML-DSA-65'`[^modern-algos] | ✔        | ✔         | ✔       |         |                | ✔              | ✔            |
| `'ML-DSA-87'`[^modern-algos] | ✔        | ✔         | ✔       |         |                | ✔              | ✔            |
| `'ML-KEM-512'`[^modern-algos] | ✔        | ✔         |         |         |                | ✔              | ✔            |
| `'ML-KEM-768'`[^modern-algos] | ✔        | ✔         |         |         |                | ✔              | ✔            |
| `'ML-KEM-1024'`[^modern-algos] | ✔        | ✔         |         |         |                | ✔              | ✔            |
| `'RSA-OAEP'`               | ✔        | ✔         | ✔       |         |                |                |              |
| `'RSA-PSS'`                | ✔        | ✔         | ✔       |         |                |                |              |
| `'RSASSA-PKCS1-v1_5'`      | ✔        | ✔         | ✔       |         |                |                |              |

### `subtle.getPublicKey(key, keyUsages)`

<!-- YAML
added: v24.7.0
-->

> Stability: 1.1 - Active development

* `key` {CryptoKey} 用于派生相应公钥的私钥。
* `keyUsages` {string\[]} 参见 [密钥用途][]。
* 返回：{Promise} 成功时以 {CryptoKey} 完成。

从给定的私钥派生公钥。

### `subtle.generateKey(algorithm, extractable, keyUsages)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59647
    description: KMAC algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59569
    description: ML-KEM algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ChaCha20-Poly1305 algorithm is now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ML-DSA algorithms are now supported.
-->

<!--lint disable maximum-line-length remark-lint-->

* `algorithm` {string|Algorithm|RsaHashedKeyGenParams|EcKeyGenParams|HmacKeyGenParams|AesKeyGenParams|KmacKeyGenParams}

<!--lint enable maximum-line-length remark-lint-->

* `extractable` {boolean}
* `keyUsages` {string\[]} 参见 [密钥用途][]。
* 返回：{Promise} 成功时以 {CryptoKey|CryptoKeyPair} 完成。

使用 `algorithm` 中提供的参数，此方法尝试生成新的密钥材料。根据使用的算法，会生成单个 {CryptoKey} 或一个 {CryptoKeyPair}。

生成 {CryptoKeyPair}（公钥和私钥）的算法包括：

* `'ECDH'`
* `'ECDSA'`
* `'Ed25519'`
* `'Ed448'`[^secure-curves]
* `'ML-DSA-44'`[^modern-algos]
* `'ML-DSA-65'`[^modern-algos]
* `'ML-DSA-87'`[^modern-algos]
* `'ML-KEM-512'`[^modern-algos]
* `'ML-KEM-768'`[^modern-algos]
* `'ML-KEM-1024'`[^modern-algos]
* `'RSA-OAEP'`
* `'RSA-PSS'`
* `'RSASSA-PKCS1-v1_5'`
* `'X25519'`
* `'X448'`[^secure-curves]

生成 {CryptoKey}（秘密密钥）的算法包括：

* `'AES-CBC'`
* `'AES-CTR'`
* `'AES-GCM'`
* `'AES-KW'`
* `'AES-OCB'`[^modern-algos]
* `'ChaCha20-Poly1305'`[^modern-algos]
* `'HMAC'`
* `'KMAC128'`[^modern-algos]
* `'KMAC256'`[^modern-algos]

### `subtle.importKey(format, keyData, algorithm, extractable, keyUsages)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59647
    description: KMAC algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59569
    description: ML-KEM algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ChaCha20-Poly1305 algorithm is now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ML-DSA algorithms are now supported.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42507
    description: Added `'Ed25519'`, `'Ed448'`, `'X25519'`, and `'X448'`
      algorithms.
  - version: v15.9.0
    pr-url: https://github.com/nodejs/node/pull/37203
    description: Removed `'NODE-DSA'` JWK import.
-->

* `format` {string} 必须是 `'raw'`、`'pkcs8'`、`'spki'`、`'jwk'`、`'raw-secret'`[^modern-algos]、`'raw-public'`[^modern-algos] 或 `'raw-seed'`[^modern-algos] 之一。
* `keyData` {ArrayBuffer|TypedArray|DataView|Buffer|Object}

<!--lint disable maximum-line-length remark-lint-->

* `algorithm` {string|Algorithm|RsaHashedImportParams|EcKeyImportParams|HmacImportParams|KmacImportParams}

<!--lint enable maximum-line-length remark-lint-->

* `extractable` {boolean}
* `keyUsages` {string\[]} 参见 [密钥用途][]。
* 返回：{Promise} 成功时以 {CryptoKey} 完成。

此方法尝试将提供的 `keyData` 解释为给定的 `format`，以使用提供的 `algorithm`、`extractable` 和 `keyUsages` 参数创建 {CryptoKey} 实例。如果导入成功，返回的 Promise 将以密钥材料的 {CryptoKey} 表示形式解析。

如果导入 KDF 算法密钥，`extractable` 必须为 `false`。

当前支持的算法包括：

| 支持的密钥算法              | `'spki'` | `'pkcs8'` | `'jwk'` | `'raw'` | `'raw-secret'` | `'raw-public'` | `'raw-seed'` |
| -------------------------- | -------- | --------- | ------- | ------- | -------------- | -------------- | ------------ |
| `'AES-CBC'`                |          |           | ✔       | ✔       | ✔              |                |              |
| `'AES-CTR'`                |          |           | ✔       | ✔       | ✔              |                |              |
| `'AES-GCM'`                |          |           | ✔       | ✔       | ✔              |                |              |
| `'AES-KW'`                 |          |           | ✔       | ✔       | ✔              |                |              |
| `'AES-OCB'`[^modern-algos] |          |           | ✔       |         | ✔              |                |              |
| `'Argon2d'`[^modern-algos] |          |           |         |         | ✔              |                |              |
| `'Argon2i'`[^modern-algos] |          |           |         |         | ✔              |                |              |
| `'Argon2id'`[^modern-algos] |          |           |         |         | ✔              |                |              |
| `'ChaCha20-Poly1305'`[^modern-algos] |          |           | ✔       |         | ✔              |                |              |
| `'ECDH'`                   | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |
| `'ECDSA'`                  | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |
| `'Ed25519'`                | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |
| `'Ed448'`[^secure-curves]  | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |
| `'HDKF'`                   |          |           |         | ✔       | ✔              |                |              |
| `'HMAC'`                   |          |           | ✔       | ✔       | ✔              |                |              |
| `'KMAC128'`[^modern-algos] |          |           | ✔       |         | ✔              |                |              |
| `'KMAC256'`[^modern-algos] |          |           | ✔       |         | ✔              |                |              |
| `'ML-DSA-44'`[^modern-algos] | ✔        | ✔         | ✔       |         |                | ✔              | ✔            |
| `'ML-DSA-65'`[^modern-algos] | ✔        | ✔         | ✔       |         |                | ✔              | ✔            |
| `'ML-DSA-87'`[^modern-algos] | ✔        | ✔         | ✔       |         |                | ✔              | ✔            |
| `'ML-KEM-512'`[^modern-algos] | ✔        | ✔         |         |         |                | ✔              | ✔            |
| `'ML-KEM-768'`[^modern-algos] | ✔        | ✔         |         |         |                | ✔              | ✔            |
| `'ML-KEM-1024'`[^modern-algos] | ✔        | ✔         |         |         |                | ✔              | ✔            |
| `'PBKDF2'`                 |          |           |         | ✔       | ✔              |                |              |
| `'RSA-OAEP'`               | ✔        | ✔         | ✔       |         |                |                |              |
| `'RSA-PSS'`                | ✔        | ✔         | ✔       |         |                |                |              |
| `'RSASSA-PKCS1-v1_5'`      | ✔        | ✔         | ✔       |         |                |                |              |
| `'X25519'`                 | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |
| `'X448'`[^secure-curves]   | ✔        | ✔         | ✔       | ✔       |                | ✔              |              |

### `subtle.sign(algorithm, key, data)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59647
    description: KMAC algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ML-DSA algorithms are now supported.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42507
    description: Added `'Ed25519'`, and `'Ed448'` algorithms.
-->

<!--lint disable maximum-line-length remark-lint-->

* `algorithm` {string|Algorithm|RsaPssParams|EcdsaParams|ContextParams|KmacParams}
* `key` {CryptoKey}
* `data` {ArrayBuffer|TypedArray|DataView|Buffer}
* 返回：{Promise} 成功时以 {ArrayBuffer} 完成。

<!--lint enable maximum-line-length remark-lint-->

使用 `algorithm` 给出的方法和参数以及 `key` 提供的密钥材料，此方法尝试生成 `data` 的密码学签名。如果成功，返回的 Promise 将以包含生成签名的 {ArrayBuffer} 解析。

当前支持的算法包括：

* `'ECDSA'`
* `'Ed25519'`
* `'Ed448'`[^secure-curves]
* `'HMAC'`
* `'KMAC128'`[^modern-algos]
* `'KMAC256'`[^modern-algos]
* `'ML-DSA-44'`[^modern-algos]
* `'ML-DSA-65'`[^modern-algos]
* `'ML-DSA-87'`[^modern-algos]
* `'RSA-PSS'`
* `'RSASSA-PKCS1-v1_5'`

### `subtle.unwrapKey(format, wrappedKey, unwrappingKey, unwrapAlgo, unwrappedKeyAlgo, extractable, keyUsages)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59539
    description: AES-OCB algorithm is now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ChaCha20-Poly1305 algorithm is now supported.
-->

* `format` {string} 必须是 `'raw'`、`'pkcs8'`、`'spki'`、`'jwk'`、`'raw-secret'`[^modern-algos]、`'raw-public'`[^modern-algos] 或 `'raw-seed'`[^modern-algos] 之一。
* `wrappedKey` {ArrayBuffer|TypedArray|DataView|Buffer}
* `unwrappingKey` {CryptoKey}

<!--lint disable maximum-line-length remark-lint-->

* `unwrapAlgo` {string|Algorithm|RsaOaepParams|AesCtrParams|AesCbcParams|AeadParams}
* `unwrappedKeyAlgo` {string|Algorithm|RsaHashedImportParams|EcKeyImportParams|HmacImportParams|KmacImportParams}

<!--lint enable maximum-line-length remark-lint-->

* `extractable` {boolean}
* `keyUsages` {string\[]} 参见 [密钥用途][]。
* 返回：{Promise} 成功时以 {CryptoKey} 完成。

在密码学中，“包装密钥”指的是导出然后加密密钥材料。此方法尝试解密已包装的密钥并创建 {CryptoKey} 实例。它等效于首先在加密的密钥数据上调用 [`subtle.decrypt()`][]（使用 `wrappedKey`、`unwrapAlgo` 和 `unwrappingKey` 参数作为输入），然后将结果传递给 [`subtle.importKey()`][] 方法，使用 `unwrappedKeyAlgo`、`extractable` 和 `keyUsages` 参数作为输入。如果成功，返回的 Promise 将以 {CryptoKey} 对象解析。

当前支持的包装算法包括：

* `'AES-CBC'`
* `'AES-CTR'`
* `'AES-GCM'`
* `'AES-KW'`
* `'AES-OCB'`[^modern-algos]
* `'ChaCha20-Poly1305'`[^modern-algos]
* `'RSA-OAEP'`

解包后的密钥算法包括：

* `'AES-CBC'`
* `'AES-CTR'`
* `'AES-GCM'`
* `'AES-KW'`
* `'AES-OCB'`[^modern-algos]
* `'ChaCha20-Poly1305'`[^modern-algos]
* `'ECDH'`
* `'ECDSA'`
* `'Ed25519'`
* `'Ed448'`[^secure-curves]
* `'HMAC'`
* `'KMAC128'`[^secure-curves]
* `'KMAC256'`[^secure-curves]
* `'ML-DSA-44'`[^modern-algos]
* `'ML-DSA-65'`[^modern-algos]
* `'ML-DSA-87'`[^modern-algos]
* `'ML-KEM-512'`[^modern-algos]
* `'ML-KEM-768'`[^modern-algos]
* `'ML-KEM-1024'`[^modern-algos]v
* `'RSA-OAEP'`
* `'RSA-PSS'`
* `'RSASSA-PKCS1-v1_5'`
* `'X25519'`
* `'X448'`[^secure-curves]

### `subtle.verify(algorithm, key, signature, data)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59647
    description: KMAC algorithms are now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ML-DSA algorithms are now supported.
  - version:
    - v18.4.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42507
    description: Added `'Ed25519'`, and `'Ed448'` algorithms.
-->

<!--lint disable maximum-line-length remark-lint-->

* `algorithm` {string|Algorithm|RsaPssParams|EcdsaParams|ContextParams|KmacParams}
* `key` {CryptoKey}
* `signature` {ArrayBuffer|TypedArray|DataView|Buffer}
* `data` {ArrayBuffer|TypedArray|DataView|Buffer}
* 返回：{Promise} 成功时以 {boolean} 完成。

<!--lint enable maximum-line-length remark-lint-->

使用 `algorithm` 中给出的方法和参数以及 `key` 提供的密钥材料，此方法尝试验证 `signature` 是否是 `data` 的有效密码学签名。返回的 Promise 以 `true` 或 `false` 解析。

当前支持的算法包括：

* `'ECDSA'`
* `'Ed25519'`
* `'Ed448'`[^secure-curves]
* `'HMAC'`
* `'KMAC128'`[^secure-curves]
* `'KMAC256'`[^secure-curves]
* `'ML-DSA-44'`[^modern-algos]
* `'ML-DSA-65'`[^modern-algos]
* `'ML-DSA-87'`[^modern-algos]
* `'RSA-PSS'`
* `'RSASSA-PKCS1-v1_5'`

### `subtle.wrapKey(format, key, wrappingKey, wrapAlgo)`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59539
    description: AES-OCB algorithm is now supported.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: ChaCha20-Poly1305 algorithm is now supported.
-->

<!--lint disable maximum-line-length remark-lint-->

* `format` {string} 必须是 `'raw'`、`'pkcs8'`、`'spki'`、`'jwk'`、`'raw-secret'`[^modern-algos]、`'raw-public'`[^modern-algos] 或 `'raw-seed'`[^modern-algos] 之一。
* `key` {CryptoKey}
* `wrappingKey` {CryptoKey}
* `wrapAlgo` {string|Algorithm|RsaOaepParams|AesCtrParams|AesCbcParams|AeadParams}
* 返回：{Promise} 成功时以 {ArrayBuffer} 完成。

<!--lint enable maximum-line-length remark-lint-->

在密码学中，“包装密钥”指的是导出然后加密密钥材料。此方法将密钥材料导出为 `format` 标识的格式，然后使用 `wrapAlgo` 指定的方法和参数以及 `wrappingKey` 提供的密钥材料对其进行加密。它等效于使用 `format` 和 `key` 作为参数调用 [`subtle.exportKey()`][]，然后将结果传递给 [`subtle.encrypt()`][] 方法，使用 `wrappingKey` 和 `wrapAlgo` 作为输入。如果成功，返回的 Promise 将以包含加密密钥数据的 {ArrayBuffer} 解析。

当前支持的包装算法包括：

* `'AES-CBC'`
* `'AES-CTR'`
* `'AES-GCM'`
* `'AES-KW'`
* `'AES-OCB'`[^modern-algos]
* `'ChaCha20-Poly1305'`[^modern-algos]
* `'RSA-OAEP'`

## 算法参数

算法参数对象定义了各种 {SubtleCrypto} 方法使用的方法和参数。虽然在这里描述为“类”，但它们是简单的 JavaScript 字典对象。

### 类：`Algorithm`

<!-- YAML
added: v15.0.0
-->

#### `Algorithm.name`

<!-- YAML
added: v15.0.0
-->

* 类型：{string}

### 类：`AeadParams`

<!-- YAML
added: v15.0.0
-->

#### `aeadParams.additionalData`

<!-- YAML
added: v15.0.0
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer|undefined}

不加密但包含在数据认证中的额外输入。`additionalData` 的使用是可选的。

#### `aeadParams.iv`

<!-- YAML
added: v15.0.0
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer}

初始化向量对于使用给定密钥的每个加密操作必须是唯一的。

#### `aeadParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型：{string} 必须是 `'AES-GCM'`、`'AES-OCB'` 或 `'ChaCha20-Poly1305'`。

#### `aeadParams.tagLength`

<!-- YAML
added: v15.0.0
-->

* 类型：{number} 生成的认证标签的长度，以比特为单位。

### 类：`AesDerivedKeyParams`

<!-- YAML
added: v15.0.0
-->

#### `aesDerivedKeyParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型：{string} 必须是 `'AES-CBC'`、`'AES-CTR'`、`'AES-GCM'`、`'AES-OCB'` 或 `'AES-KW'` 之一

#### `aesDerivedKeyParams.length`

<!-- YAML
added: v15.0.0
-->

* 类型：{number}

要派生的 AES 密钥的长度。必须是 `128`、`192` 或 `256`。

### 类：`AesCbcParams`

<!-- YAML
added: v15.0.0
-->

#### `aesCbcParams.iv`

<!-- YAML
added: v15.0.0
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer}

提供初始化向量。它必须恰好为 16 字节长，并且应该是不可预测且密码学随机的。

#### `aesCbcParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型：{string} 必须是 `'AES-CBC'`。

### 类：`AesCtrParams`

<!-- YAML
added: v15.0.0
-->

#### `aesCtrParams.counter`

<!-- YAML
added: v15.0.0
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer}

计数器块的初始值。必须恰好为 16 字节长。

`AES-CTR` 方法使用块的最右边 `length` 比特作为计数器，其余比特作为随机数。

#### `aesCtrParams.length`

<!-- YAML
added: v15.0.0
-->

* 类型：{number} `aesCtrParams.counter` 中用作计数器的比特数。

#### `aesCtrParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型：{string} 必须是 `'AES-CTR'`。

### 类：`AesKeyAlgorithm`

<!-- YAML
added: v15.0.0
-->

#### `aesKeyAlgorithm.length`

<!-- YAML
added: v15.0.0
-->

* 类型：{number}

AES 密钥的长度，以比特为单位。

#### `aesKeyAlgorithm.name`

<!-- YAML
added: v15.0.0
-->

* 类型：{string}

### 类：`AesKeyGenParams`

<!-- YAML
added: v15.0.0
-->

#### `aesKeyGenParams.length`

<!-- YAML
added: v15.0.0
-->

* 类型：{number}

要生成的 AES 密钥的长度。必须是 `128`、`192` 或 `256`。

#### `aesKeyGenParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型：{string} 必须是 `'AES-CBC'`、`'AES-CTR'`、`'AES-GCM'` 或 `'AES-KW'` 之一

### 类：`Argon2Params`

<!-- YAML
added: v24.8.0
-->

#### `argon2Params.associatedData`

<!-- YAML
added: v24.8.0
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer}

表示可选的关联数据。

#### `argon2Params.memory`

<!-- YAML
added: v24.8.0
-->

* 类型：{number}

表示内存大小，以 kibibytes 为单位。必须至少是并行度的 8 倍。

#### `argon2Params.name`

<!-- YAML
added: v24.8.0
-->

* 类型：{string} 必须是 `'Argon2d'`、`'Argon2i'` 或 `'Argon2id'` 之一。

#### `argon2Params.nonce`

<!-- YAML
added: v24.8.0
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer}

表示随机数，对于密码哈希应用来说是盐值。

#### `argon2Params.parallelism`

<!-- YAML
added: v24.8.0
-->

* 类型：{number}

表示并行度。

#### `argon2Params.passes`

<!-- YAML
added: v24.8.0
-->

* 类型：{number}

表示轮数。

#### `argon2Params.secretValue`

<!-- YAML
added: v24.8.0
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer}

表示可选的秘密值。

#### `argon2Params.version`

<!-- YAML
added: v24.8.0
-->

* 类型：{number}

表示 Argon2 版本号。默认且当前唯一定义的版本是 `19` (`0x13`)。

### 类：`ContextParams`

<!-- YAML
added: v24.7.0
-->

#### `contextParams.name`

<!-- YAML
added: v24.7.0
-->

* 类型：{string} 必须是 `Ed448`[^secure-curves]、`'ML-DSA-44'`[^modern-algos]、`'ML-DSA-65'`[^modern-algos] 或 `'ML-DSA-87'`[^modern-algos]。

#### `contextParams.context`

<!-- YAML
added: v24.7.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59570
    description: Non-empty context is now supported.
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer|undefined}

`context` 成员表示要与消息关联的可选上下文数据。

### 类：`CShakeParams`

<!-- YAML
added: v24.7.0
-->

#### `cShakeParams.customization`

<!-- YAML
added: v24.7.0
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer|undefined}

`customization` 成员表示定制字符串。
Node.js Web Crypto API 实现仅支持零长度定制，这相当于根本不提供定制。

#### `cShakeParams.functionName`

<!-- YAML
added: v24.7.0
-->

* 类型：{ArrayBuffer|TypedArray|DataView|Buffer|undefined}

`functionName` 成员表示函数名称，由 NIST 用于定义基于 cSHAKE 的函数。
Node.js Web Crypto API 实现仅支持零长度函数名称，这相当于根本不提供函数名称。

#### `cShakeParams.length`

<!-- YAML
added: v24.7.0
-->

* 类型：{number} 表示请求的输出长度，以比特为单位。

#### `cShakeParams.name`

<!-- YAML
added: v24.7.0
-->

* 类型：{string} 必须是 `'cSHAKE128'`[^modern-algos] 或 `'cSHAKE256'`[^modern-algos]

### 类：`EcdhKeyDeriveParams`

<!-- YAML
added: v15.0.0
-->

#### `ecdhKeyDeriveParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型：{string} 必须是 `'ECDH'`、`'X25519'` 或 `'X448'`[^secure-curves]。

#### `ecdhKeyDeriveParams.public`

<!-- YAML
added: v15.0.0
-->

* 类型：{CryptoKey}

ECDH 密钥派生通过输入一方的私钥和另一方的公钥来操作——使用两者生成共同的共享秘密。`ecdhKeyDeriveParams.public` 属性设置为另一方的公钥。

### 类：`EcdsaParams`

<!-- YAML
added: v15.0.0
-->

#### `ecdsaParams.hash`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHA-3 algorithms are now supported.
-->

* 类型：{string|Algorithm}

如果表示为 {string}，值必须是以下之一：

* `'SHA-1'`
* `'SHA-256'`
* `'SHA-384'`
* `'SHA-512'`
* `'SHA3-256'`[^modern-algos]
* `'SHA3-384'`[^modern-algos]
* `'SHA3-512'`[^modern-algos]

如果表示为 {Algorithm}，对象的 `name` 属性必须是上述列出的

#### `ecdsaParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'ECDSA'`

### 类: `EcKeyAlgorithm`

<!-- YAML
added: v15.0.0
-->

#### `ecKeyAlgorithm.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string}

#### `ecKeyAlgorithm.namedCurve`

<!-- YAML
added: v15.0.0
-->

* 类型: {string}

### 类: `EcKeyGenParams`

<!-- YAML
added: v15.0.0
-->

#### `ecKeyGenParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'ECDSA'` 或 `'ECDH'` 之一

#### `ecKeyGenParams.namedCurve`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'P-256'`, `'P-384'`, `'P-521'` 之一

### 类: `EcKeyImportParams`

<!-- YAML
added: v15.0.0
-->

#### `ecKeyImportParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'ECDSA'` 或 `'ECDH'` 之一

#### `ecKeyImportParams.namedCurve`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'P-256'`, `'P-384'`, `'P-521'` 之一

### 类: `EncapsulatedBits`

<!-- YAML
added: v24.7.0
-->

用于消息加密的临时对称密钥（表示为 {ArrayBuffer}）以及由此共享密钥加密的密文（可随消息一起传输给消息接收方）。接收方使用其私钥确定共享密钥，从而能够解密消息。

#### `encapsulatedBits.ciphertext`

<!-- YAML
added: v24.7.0
-->

* 类型: {ArrayBuffer}

#### `encapsulatedBits.sharedKey`

<!-- YAML
added: v24.7.0
-->

* 类型: {ArrayBuffer}

### 类: `EncapsulatedKey`

<!-- YAML
added: v24.7.0
-->

用于消息加密的临时对称密钥（表示为 {CryptoKey}）以及由此共享密钥加密的密文（可随消息一起传输给消息接收方）。接收方使用其私钥确定共享密钥，从而能够解密消息。

#### `encapsulatedKey.ciphertext`

<!-- YAML
added: v24.7.0
-->

* 类型: {ArrayBuffer}

#### `encapsulatedKey.sharedKey`

<!-- YAML
added: v24.7.0
-->

* 类型: {CryptoKey}

### 类: `HkdfParams`

<!-- YAML
added: v15.0.0
-->

#### `hkdfParams.hash`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHA-3 算法现已支持
-->

* 类型: {string|Algorithm}

如果表示为 {string}，该值必须为以下之一:

* `'SHA-1'`
* `'SHA-256'`
* `'SHA-384'`
* `'SHA-512'`
* `'SHA3-256'`[^modern-algos]
* `'SHA3-384'`[^modern-algos]
* `'SHA3-512'`[^modern-algos]

如果表示为 {Algorithm}，对象的 `name` 属性必须为上述列出的值之一。

#### `hkdfParams.info`

<!-- YAML
added: v15.0.0
-->

* 类型: {ArrayBuffer|TypedArray|DataView|Buffer}

向 HKDF 算法提供应用特定的上下文输入。可以是零长度但必须提供。

#### `hkdfParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'HKDF'`

#### `hkdfParams.salt`

<!-- YAML
added: v15.0.0
-->

* 类型: {ArrayBuffer|TypedArray|DataView|Buffer}

盐值显著提高了 HKDF 算法的强度。它应该是随机或伪随机的，并且长度应与摘要函数的输出长度相同（例如，如果使用 `'SHA-256'` 作为摘要，盐值应为 256 位的随机数据）。

### 类: `HmacImportParams`

<!-- YAML
added: v15.0.0
-->

#### `hmacImportParams.hash`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHA-3 算法现已支持
-->

* 类型: {string|Algorithm}

如果表示为 {string}，该值必须为以下之一:

* `'SHA-1'`
* `'SHA-256'`
* `'SHA-384'`
* `'SHA-512'`
* `'SHA3-256'`[^modern-algos]
* `'SHA3-384'`[^modern-algos]
* `'SHA3-512'`[^modern-algos]

如果表示为 {Algorithm}，对象的 `name` 属性必须为上述列出的值之一。

#### `hmacImportParams.length`

<!-- YAML
added: v15.0.0
-->

* 类型: {number}

HMAC 密钥的位长度（可选）。在大多数情况下此项可选且应省略。

#### `hmacImportParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'HMAC'`

### 类: `HmacKeyAlgorithm`

<!-- YAML
added: v15.0.0
-->

#### `hmacKeyAlgorithm.hash`

<!-- YAML
added: v15.0.0
-->

* 类型: {Algorithm}

#### `hmacKeyAlgorithm.length`

<!-- YAML
added: v15.0.0
-->

* 类型: {number}

HMAC 密钥的位长度。

#### `hmacKeyAlgorithm.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string}

### 类: `HmacKeyGenParams`

<!-- YAML
added: v15.0.0
-->

#### `hmacKeyGenParams.hash`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHA-3 算法现已支持
-->

* 类型: {string|Algorithm}

如果表示为 {string}，该值必须为以下之一:

* `'SHA-1'`
* `'SHA-256'`
* `'SHA-384'`
* `'SHA-512'`
* `'SHA3-256'`[^modern-algos]
* `'SHA3-384'`[^modern-algos]
* `'SHA3-512'`[^modern-algos]

如果表示为 {Algorithm}，对象的 `name` 属性必须为上述列出的值之一。

#### `hmacKeyGenParams.length`

<!-- YAML
added: v15.0.0
-->

* 类型: {number}

为 HMAC 密钥生成的位长度。如果省略，长度将由使用的哈希算法确定。在大多数情况下此项可选且应省略。

#### `hmacKeyGenParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'HMAC'`

### 类: `KeyAlgorithm`

<!-- YAML
added: v15.0.0
-->

#### `keyAlgorithm.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string}

### 类: `KmacImportParams`

<!-- YAML
added: v24.8.0
-->

#### `kmacImportParams.length`

<!-- YAML
added: v24.8.0
-->

* 类型: {number}

KMAC 密钥的位长度（可选）。在大多数情况下此项可选且应省略。

#### `kmacImportParams.name`

<!-- YAML
added: v24.8.0
-->

* 类型: {string} 必须为 `'KMAC128'` 或 `'KMAC256'`

### 类: `KmacKeyAlgorithm`

<!-- YAML
added: v24.8.0
-->

#### `kmacKeyAlgorithm.length`

<!-- YAML
added: v24.8.0
-->

* 类型: {number}

KMAC 密钥的位长度。

#### `kmacKeyAlgorithm.name`

<!-- YAML
added: v24.8.0
-->

* 类型: {string}

### 类: `KmacKeyGenParams`

<!-- YAML
added: v24.8.0
-->

#### `kmacKeyGenParams.length`

<!-- YAML
added: v24.8.0
-->

* 类型: {number}

为 KMAC 密钥生成的位长度。如果省略，长度将由使用的 KMAC 算法确定。在大多数情况下此项可选且应省略。

#### `kmacKeyGenParams.name`

<!-- YAML
added: v24.8.0
-->

* 类型: {string} 必须为 `'KMAC128'` 或 `'KMAC256'`

### 类: `KmacParams`

<!-- YAML
added: v24.8.0
-->

#### `kmacParams.algorithm`

<!-- YAML
added: v24.8.0
-->

* 类型: {string} 必须为 `'KMAC128'` 或 `'KMAC256'`

#### `kmacParams.customization`

<!-- YAML
added: v24.8.0
-->

* 类型: {ArrayBuffer|TypedArray|DataView|Buffer|undefined}

`customization` 成员表示可选的定制字符串。

#### `kmacParams.length`

<!-- YAML
added: v24.8.0
-->

* 类型: {number}

输出的字节长度。必须为正整数。

### 类: `Pbkdf2Params`

<!-- YAML
added: v15.0.0
-->

#### `pbkdf2Params.hash`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHA-3 算法现已支持
-->

* 类型: {string|Algorithm}

如果表示为 {string}，该值必须为以下之一:

* `'SHA-1'`
* `'SHA-256'`
* `'SHA-384'`
* `'SHA-512'`
* `'SHA3-256'`[^modern-algos]
* `'SHA3-384'`[^modern-algos]
* `'SHA3-512'`[^modern-algos]

如果表示为 {Algorithm}，对象的 `name` 属性必须为上述列出的值之一。

#### `pbkdf2Params.iterations`

<!-- YAML
added: v15.0.0
-->

* 类型: {number}

PBKDF2 算法在派生位时应进行的迭代次数。

#### `pbkdf2Params.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'PBKDF2'`

#### `pbkdf2Params.salt`

<!-- YAML
added: v15.0.0
-->

* 类型: {ArrayBuffer|TypedArray|DataView|Buffer}

应至少为 16 个随机或伪随机字节。

### 类: `RsaHashedImportParams`

<!-- YAML
added: v15.0.0
-->

#### `rsaHashedImportParams.hash`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHA-3 算法现已支持
-->

* 类型: {string|Algorithm}

如果表示为 {string}，该值必须为以下之一:

* `'SHA-1'`
* `'SHA-256'`
* `'SHA-384'`
* `'SHA-512'`
* `'SHA3-256'`[^modern-algos]
* `'SHA3-384'`[^modern-algos]
* `'SHA3-512'`[^modern-algos]

如果表示为 {Algorithm}，对象的 `name` 属性必须为上述列出的值之一。

#### `rsaHashedImportParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'RSASSA-PKCS1-v1_5'`, `'RSA-PSS'` 或 `'RSA-OAEP'` 之一

### 类: `RsaHashedKeyAlgorithm`

<!-- YAML
added: v15.0.0
-->

#### `rsaHashedKeyAlgorithm.hash`

<!-- YAML
added: v15.0.0
-->

* 类型: {Algorithm}

#### `rsaHashedKeyAlgorithm.modulusLength`

<!-- YAML
added: v15.0.0
-->

* 类型: {number}

RSA 模数的位长度。

#### `rsaHashedKeyAlgorithm.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string}

#### `rsaHashedKeyAlgorithm.publicExponent`

<!-- YAML
added: v15.0.0
-->

* 类型: {Uint8Array}

RSA 公钥指数。

### 类: `RsaHashedKeyGenParams`

<!-- YAML
added: v15.0.0
-->

#### `rsaHashedKeyGenParams.hash`

<!-- YAML
added: v15.0.0
changes:
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59365
    description: SHA-3 算法现已支持
-->

* 类型: {string|Algorithm}

如果表示为 {string}，该值必须为以下之一:

* `'SHA-1'`
* `'SHA-256'`
* `'SHA-384'`
* `'SHA-512'`
* `'SHA3-256'`[^modern-algos]
* `'SHA3-384'`[^modern-algos]
* `'SHA3-512'`[^modern-algos]

如果表示为 {Algorithm}，对象的 `name` 属性必须为上述列出的值之一。

#### `rsaHashedKeyGenParams.modulusLength`

<!-- YAML
added: v15.0.0
-->

* 类型: {number}

RSA 模数的位长度。作为最佳实践，该值应至少为 `2048`。

#### `rsaHashedKeyGenParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'RSASSA-PKCS1-v1_5'`, `'RSA-PSS'` 或 `'RSA-OAEP'` 之一

#### `rsaHashedKeyGenParams.publicExponent`

<!-- YAML
added: v15.0.0
-->

* 类型: {Uint8Array}

RSA 公钥指数。必须是一个包含大端无符号整数的 {Uint8Array}，且必须适应 32 位。{Uint8Array} 可以包含任意数量的前导零位。该值必须是质数。除非有理由使用不同的值，否则应使用 `new Uint8Array([1, 0, 1])` (65537) 作为公钥指数。

### 类: `RsaOaepParams`

<!-- YAML
added: v15.0.0
-->

#### `rsaOaepParams.label`

<!-- YAML
added: v15.0.0
-->

* 类型: {ArrayBuffer|TypedArray|DataView|Buffer}

不会加密但会绑定到生成的密文的额外字节集合。

`rsaOaepParams.label` 参数是可选的。

#### `rsaOaepParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'RSA-OAEP'`

### 类: `RsaPssParams`

<!-- YAML
added: v15.0.0
-->

#### `rsaPssParams.name`

<!-- YAML
added: v15.0.0
-->

* 类型: {string} 必须为 `'RSA-PSS'`

#### `rsaPssParams.saltLength`

<!-- YAML
added: v15.0.0
-->

* 类型: {number}

要使用的随机盐的长度（字节）。

[^secure-curves]: 参见 [Web Cryptography API 中的安全曲线][]

[^modern-algos]: 参见 [Web Cryptography API 中的现代算法][]

[^openssl30]: 需要 OpenSSL >= 3.0

[^openssl32]: 需要 OpenSSL >= 3.2

[^openssl35]: 需要 OpenSSL >= 3.5

[检查运行时算法支持]: #checking-for-runtime-algorithm-support
[JSON Web Key]: https://tools.ietf.org/html/rfc7517
[密钥用途]: #cryptokeyusages
[Web Cryptography API 中的现代算法]: #modern-algorithms-in-the-web-cryptography-api
[RFC 4122]: https://www.rfc-editor.org/rfc/rfc4122.txt
[Web Cryptography API 中的安全曲线]: #secure-curves-in-the-web-cryptography-api
[Web Crypto API]: https://www.w3.org/TR/WebCryptoAPI/
[`SubtleCrypto.supports()`]: #static-method-subtlecryptosupportsoperation-algorithm-lengthoradditionalalgorithm
[`subtle.decapsulateBits()`]: #subtledecapsulatebitsdecapsulationalgorithm-decapsulationkey-ciphertext
[`subtle.decapsulateKey()`]: #subtledecapsulatekeydecapsulationalgorithm-decapsulationkey-ciphertext-sharedkeyalgorithm-extractable-usages
[`subtle.decrypt()`]: #subtledecryptalgorithm-key-data
[`subtle.deriveBits()`]: #subtlederivebitsalgorithm-basekey-length
[`subtle.deriveKey()`]: #subtlederivekeyalgorithm-basekey-derivedkeyalgorithm-extractable-keyusages
[`subtle.digest()`]: #subtledigestalgorithm-data
[`subtle.encapsulateBits()`]: #subtleencapsulatebitsencapsulationalgorithm-encapsulationkey
[`subtle.encapsulateKey()`]: #subtleencapsulatekeyencapsulationalgorithm-encapsulationkey-sharedkeyalgorithm-extractable-usages
[`subtle.encrypt()`]: #subtleencryptalgorithm-key-data
[`subtle.exportKey()`]: #subtleexportkeyformat-key
[`subtle.generateKey()`]: #subtlegeneratekeyalgorithm-extractable-keyusages
[`subtle.getPublicKey()`]: #subtlegetpublickeykey-keyusages
[`subtle.importKey()`]: #subtleimportkeyformat-keydata-algorithm-extractable-keyusages
[`subtle.sign()`]: #subtlesignalgorithm-key-data
[`subtle.unwrapKey()`]: #subtleunwrapkeyformat-wrappedkey-unwrappingkey-unwrapalgo-unwrappedkeyalgo-extractable-keyusages
[`subtle.verify()`]: #subtleverifyalgorithm-key-signature-data
[`subtle.wrapKey()`]: #subtlewrapkeyformat-key-wrappingkey-wrapalgo