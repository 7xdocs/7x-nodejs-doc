
### `crypto.diffieHellman(options[, callback])`

<!-- YAML
added:
 - v13.9.0
 - v12.17.0
changes:
  - version: v23.11.0
    pr-url: https://github.com/nodejs/node/pull/57274
    description: Optional callback argument added.
-->

* `options` {Object}
  * `privateKey` {KeyObject}
  * `publicKey` {KeyObject}
* `callback` {Function}
  * `err` {Error}
  * `secret` {Buffer}
* Returns: {Buffer} if the `callback` function is not provided.

Computes the Diffie-Hellman shared secret based on a `privateKey` and a `publicKey`.
Both keys must have the same `asymmetricKeyType` and must support either the DH or
ECDH operation.

If the `callback` function is provided this function uses libuv's threadpool.

### `crypto.encapsulate(key[, callback])`

<!-- YAML
added: v24.7.0
-->

> Stability: 1.2 - Release candidate

* `key` {Object|string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject} Public Key
* `callback` {Function}
  * `err` {Error}
  * `result` {Object}
    * `sharedKey` {Buffer}
    * `ciphertext` {Buffer}
* Returns: {Object} if the `callback` function is not provided.
  * `sharedKey` {Buffer}
  * `ciphertext` {Buffer}

<!--lint enable maximum-line-length remark-lint-->

Key encapsulation using a KEM algorithm with a public key.

Supported key types and their KEM algorithms are:

* `'rsa'`[^openssl30] RSA Secret Value Encapsulation
* `'ec'`[^openssl32] DHKEM(P-256, HKDF-SHA256), DHKEM(P-384, HKDF-SHA256), DHKEM(P-521, HKDF-SHA256)
* `'x25519'`[^openssl32] DHKEM(X25519, HKDF-SHA256)
* `'x448'`[^openssl32] DHKEM(X448, HKDF-SHA512)
* `'ml-kem-512'`[^openssl35] ML-KEM
* `'ml-kem-768'`[^openssl35] ML-KEM
* `'ml-kem-1024'`[^openssl35] ML-KEM

If `key` is not a [`KeyObject`][], this function behaves as if `key` had been
passed to [`crypto.createPublicKey()`][].

If the `callback` function is provided this function uses libuv's threadpool.

### `crypto.fips`

<!-- YAML
added: v6.0.0
deprecated: v10.0.0
-->

> Stability: 0 - Deprecated

Property for checking and controlling whether a FIPS compliant crypto provider
is currently in use. Setting to true requires a FIPS build of Node.js.

This property is deprecated. Please use `crypto.setFips()` and
`crypto.getFips()` instead.

### `crypto.generateKey(type, options, callback)`

<!-- YAML
added: v15.0.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `type` {string} The intended use of the generated secret key. Currently
  accepted values are `'hmac'` and `'aes'`.
* `options` {Object}
  * `length` {number} The bit length of the key to generate. This must be a
    value greater than 0.
    * If `type` is `'hmac'`, the minimum is 8, and the maximum length is
      2<sup>31</sup>-1. If the value is not a multiple of 8, the generated
      key will be truncated to `Math.floor(length / 8)`.
    * If `type` is `'aes'`, the length must be one of `128`, `192`, or `256`.
* `callback` {Function}
  * `err` {Error}
  * `key` {KeyObject}

Asynchronously generates a new random secret key of the given `length`. The
`type` will determine which validations will be performed on the `length`.

```mjs
const {
  generateKey,
} = await import('node:crypto');

generateKey('hmac', { length: 512 }, (err, key) => {
  if (err) throw err;
  console.log(key.export().toString('hex'));  // 46e..........620
});
```

```cjs
const {
  generateKey,
} = require('node:crypto');

generateKey('hmac', { length: 512 }, (err, key) => {
  if (err) throw err;
  console.log(key.export().toString('hex'));  // 46e..........620
});
```

The size of a generated HMAC key should not exceed the block size of the
underlying hash function. See [`crypto.createHmac()`][] for more information.

### `crypto.generateKeyPair(type, options, callback)`

<!-- YAML
added: v10.12.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59537
    description: Add support for SLH-DSA key pairs.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59461
    description: Add support for ML-KEM key pairs.
  - version: v24.6.0
    pr-url: https://github.com/nodejs/node/pull/59259
    description: Add support for ML-DSA key pairs.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v16.10.0
    pr-url: https://github.com/nodejs/node/pull/39927
    description: Add ability to define `RSASSA-PSS-params` sequence parameters
                 for RSA-PSS keys pairs.
  - version:
     - v13.9.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/31178
    description: Add support for Diffie-Hellman.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26960
    description: Add support for RSA-PSS key pairs.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26774
    description: Add ability to generate X25519 and X448 key pairs.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26554
    description: Add ability to generate Ed25519 and Ed448 key pairs.
  - version: v11.6.0
    pr-url: https://github.com/nodejs/node/pull/24234
    description: The `generateKeyPair` and `generateKeyPairSync` functions now
                 produce key objects if no encoding was specified.
-->

* `type` {string} The asymmetric key type to generate. See the
  supported [asymmetric key types][].
* `options` {Object}
  * `modulusLength` {number} Key size in bits (RSA, DSA).
  * `publicExponent` {number} Public exponent (RSA). **Default:** `0x10001`.
  * `hashAlgorithm` {string} Name of the message digest (RSA-PSS).
  * `mgf1HashAlgorithm` {string} Name of the message digest used by
    MGF1 (RSA-PSS).
  * `saltLength` {number} Minimal salt length in bytes (RSA-PSS).
  * `divisorLength` {number} Size of `q` in bits (DSA).
  * `namedCurve` {string} Name of the curve to use (EC).
  * `prime` {Buffer} The prime parameter (DH).
  * `primeLength` {number} Prime length in bits (DH).
  * `generator` {number} Custom generator (DH). **Default:** `2`.
  * `groupName` {string} Diffie-Hellman group name (DH). See
    [`crypto.getDiffieHellman()`][].
  * `paramEncoding` {string} Must be `'named'` or `'explicit'` (EC).
    **Default:** `'named'`.
  * `publicKeyEncoding` {Object} See [`keyObject.export()`][].
  * `privateKeyEncoding` {Object} See [`keyObject.export()`][].
* `callback` {Function}
  * `err` {Error}
  * `publicKey` {string | Buffer | KeyObject}
  * `privateKey` {string | Buffer | KeyObject}

Generates a new asymmetric key pair of the given `type`. RSA, RSA-PSS, DSA, EC,
Ed25519, Ed448, X25519, X448, and DH are currently supported.

If a `publicKeyEncoding` or `privateKeyEncoding` was specified, this function
behaves as if [`keyObject.export()`][] had been called on its result. Otherwise,
the respective part of the key is returned as a [`KeyObject`][].

It is recommended to encode public keys as `'spki'` and private keys as
`'pkcs8'` with encryption for long-term storage:

```mjs
const {
  generateKeyPair,
} = await import('node:crypto');

generateKeyPair('rsa', {
  modulusLength: 4096,
  publicKeyEncoding: {
    type: 'spki',
    format: 'pem',
  },
  privateKeyEncoding: {
    type: 'pkcs8',
    format: 'pem',
    cipher: 'aes-256-cbc',
    passphrase: 'top secret',
  },
}, (err, publicKey, privateKey) => {
  // Handle errors and use the generated key pair.
});
```

```cjs
const {
  generateKeyPair,
} = require('node:crypto');

generateKeyPair('rsa', {
  modulusLength: 4096,
  publicKeyEncoding: {
    type: 'spki',
    format: 'pem',
  },
  privateKeyEncoding: {
    type: 'pkcs8',
    format: 'pem',
    cipher: 'aes-256-cbc',
    passphrase: 'top secret',
  },
}, (err, publicKey, privateKey) => {
  // Handle errors and use the generated key pair.
});
```

On completion, `callback` will be called with `err` set to `undefined` and
`publicKey` / `privateKey` representing the generated key pair.

If this method is invoked as its [`util.promisify()`][]ed version, it returns
a `Promise` for an `Object` with `publicKey` and `privateKey` properties.

### `crypto.generateKeyPairSync(type, options)`

<!-- YAML
added: v10.12.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59537
    description: Add support for SLH-DSA key pairs.
  - version: v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59461
    description: Add support for ML-KEM key pairs.
  - version: v24.6.0
    pr-url: https://github.com/nodejs/node/pull/59259
    description: Add support for ML-DSA key pairs.
  - version: v16.10.0
    pr-url: https://github.com/nodejs/node/pull/39927
    description: Add ability to define `RSASSA-PSS-params` sequence parameters
                 for RSA-PSS keys pairs.
  - version:
     - v13.9.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/31178
    description: Add support for Diffie-Hellman.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26960
    description: Add support for RSA-PSS key pairs.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26774
    description: Add ability to generate X25519 and X448 key pairs.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26554
    description: Add ability to generate Ed25519 and Ed448 key pairs.
  - version: v11.6.0
    pr-url: https://github.com/nodejs/node/pull/24234
    description: The `generateKeyPair` and `generateKeyPairSync` functions now
                 produce key objects if no encoding was specified.
-->

* `type` {string} The asymmetric key type to generate. See the
  supported [asymmetric key types][].
* `options` {Object}
  * `modulusLength` {number} Key size in bits (RSA, DSA).
  * `publicExponent` {number} Public exponent (RSA). **Default:** `0x10001`.
  * `hashAlgorithm` {string} Name of the message digest (RSA-PSS).
  * `mgf1HashAlgorithm` {string} Name of the message digest used by
    MGF1 (RSA-PSS).
  * `saltLength` {number} Minimal salt length in bytes (RSA-PSS).
  * `divisorLength` {number} Size of `q` in bits (DSA).
  * `namedCurve` {string} Name of the curve to use (EC).
  * `prime` {Buffer} The prime parameter (DH).
  * `primeLength` {number} Prime length in bits (DH).
  * `generator` {number} Custom generator (DH). **Default:** `2`.
  * `groupName` {string} Diffie-Hellman group name (DH). See
    [`crypto.getDiffieHellman()`][].
  * `paramEncoding` {string} Must be `'named'` or `'explicit'` (EC).
    **Default:** `'named'`.
  * `publicKeyEncoding` {Object} See [`keyObject.export()`][].
  * `privateKeyEncoding` {Object} See [`keyObject.export()`][].
* Returns: {Object}
  * `publicKey` {string | Buffer | KeyObject}
  * `privateKey` {string | Buffer | KeyObject}

Generates a new asymmetric key pair of the given `type`. RSA, RSA-PSS, DSA, EC,
Ed25519, Ed448, X25519, X448, DH, and ML-DSA[^openssl35] are currently supported.

If a `publicKeyEncoding` or `privateKeyEncoding` was specified, this function
behaves as if [`keyObject.export()`][] had been called on its result. Otherwise,
the respective part of the key is returned as a [`KeyObject`][].

When encoding public keys, it is recommended to use `'spki'`. When encoding
private keys, it is recommended to use `'pkcs8'` with a strong passphrase,
and to keep the passphrase confidential.

```mjs
const {
  generateKeyPairSync,
} = await import('node:crypto');

const {
  publicKey,
  privateKey,
} = generateKeyPairSync('rsa', {
  modulusLength: 4096,
  publicKeyEncoding: {
    type: 'spki',
    format: 'pem',
  },
  privateKeyEncoding: {
    type: 'pkcs8',
    format: 'pem',
    cipher: 'aes-256-cbc',
    passphrase: 'top secret',
  },
});
```

```cjs
const {
  generateKeyPairSync,
} = require('node:crypto');

const {
  publicKey,
  privateKey,
} = generateKeyPairSync('rsa', {
  modulusLength: 4096,
  publicKeyEncoding: {
    type: 'spki',
    format: 'pem',
  },
  privateKeyEncoding: {
    type: 'pkcs8',
    format: 'pem',
    cipher: 'aes-256-cbc',
    passphrase: 'top secret',
  },
});
```

The return value `{ publicKey, privateKey }` represents the generated key pair.
When PEM encoding was selected, the respective key will be a string, otherwise
it will be a buffer containing the data encoded as DER.

### `crypto.generateKeySync(type, options)`

<!-- YAML
added: v15.0.0
-->

* `type` {string} The intended use of the generated secret key. Currently
  accepted values are `'hmac'` and `'aes'`.
* `options` {Object}
  * `length` {number} The bit length of the key to generate.
    * If `type` is `'hmac'`, the minimum is 8, and the maximum length is
      2<sup>31</sup>-1. If the value is not a multiple of 8, the generated
      key will be truncated to `Math.floor(length / 8)`.
    * If `type` is `'aes'`, the length must be one of `128`, `192`, or `256`.
* Returns: {KeyObject}

Synchronously generates a new random secret key of the given `length`. The
`type` will determine which validations will be performed on the `length`.

```mjs
const {
  generateKeySync,
} = await import('node:crypto');

const key = generateKeySync('hmac', { length: 512 });
console.log(key.export().toString('hex'));  // e89..........41e
```

```cjs
const {
  generateKeySync,
} = require('node:crypto');

const key = generateKeySync('hmac', { length: 512 });
console.log(key.export().toString('hex'));  // e89..........41e
```

The size of a generated HMAC key should not exceed the block size of the
underlying hash function. See [`crypto.createHmac()`][] for more information.

### `crypto.generatePrime(size[, options], callback)`

<!-- YAML
added: v15.8.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `size` {number} The size (in bits) of the prime to generate.
* `options` {Object}
  * `add` {ArrayBuffer|SharedArrayBuffer|TypedArray|Buffer|DataView|bigint}
  * `rem` {ArrayBuffer|SharedArrayBuffer|TypedArray|Buffer|DataView|bigint}
  * `safe` {boolean} **Default:** `false`.
  * `bigint` {boolean} When `true`, the generated prime is returned
    as a `bigint`.
* `callback` {Function}
  * `err` {Error}
  * `prime` {ArrayBuffer|bigint}

Generates a pseudorandom prime of `size` bits.

If `options.safe` is `true`, the prime will be a safe prime -- that is,
`(prime - 1) / 2` will also be a prime.

The `options.add` and `options.rem` parameters can be used to enforce additional
requirements, e.g., for Diffie-Hellman:

* If `options.add` and `options.rem` are both set, the prime will satisfy the
  condition that `prime % add = rem`.
* If only `options.add` is set and `options.safe` is not `true`, the prime will
  satisfy the condition that `prime % add = 1`.
* If only `options.add` is set and `options.safe` is set to `true`, the prime
  will instead satisfy the condition that `prime % add = 3`. This is necessary
  because `prime % add = 1` for `options.add > 2` would contradict the condition
  enforced by `options.safe`.
* `options.rem` is ignored if `options.add` is not given.

Both `options.add` and `options.rem` must be encoded as big-endian sequences
if given as an `ArrayBuffer`, `SharedArrayBuffer`, `TypedArray`, `Buffer`, or
`DataView`.

By default, the prime is encoded as a big-endian sequence of octets
in an {ArrayBuffer}. If the `bigint` option is `true`, then a {bigint}
is provided.

The `size` of the prime will have a direct impact on how long it takes to
generate the prime. The larger the size, the longer it will take. Because
we use OpenSSL's `BN_generate_prime_ex` function, which provides only
minimal control over our ability to interrupt the generation process,
it is not recommended to generate overly large primes, as doing so may make
the process unresponsive.

### `crypto.generatePrimeSync(size[, options])`

<!-- YAML
added: v15.8.0
-->

* `size` {number} The size (in bits) of the prime to generate.
* `options` {Object}
  * `add` {ArrayBuffer|SharedArrayBuffer|TypedArray|Buffer|DataView|bigint}
  * `rem` {ArrayBuffer|SharedArrayBuffer|TypedArray|Buffer|DataView|bigint}
  * `safe` {boolean} **Default:** `false`.
  * `bigint` {boolean} When `true`, the generated prime is returned
    as a `bigint`.
* Returns: {ArrayBuffer|bigint}

Generates a pseudorandom prime of `size` bits.

If `options.safe` is `true`, the prime will be a safe prime -- that is,
`(prime - 1) / 2` will also be a prime.

The `options.add` and `options.rem` parameters can be used to enforce additional
requirements, e.g., for Diffie-Hellman:

* If `options.add` and `options.rem` are both set, the prime will satisfy the
  condition that `prime % add = rem`.
* If only `options.add` is set and `options.safe` is not `true`, the prime will
  satisfy the condition that `prime % add = 1`.
* If only `options.add` is set and `options.safe` is set to `true`, the prime
  will instead satisfy the condition that `prime % add = 3`. This is necessary
  because `prime % add = 1` for `options.add > 2` would contradict the condition
  enforced by `options.safe`.
* `options.rem` is ignored if `options.add` is not given.

Both `options.add` and `options.rem` must be encoded as big-endian sequences
if given as an `ArrayBuffer`, `SharedArrayBuffer`, `TypedArray`, `Buffer`, or
`DataView`.

By default, the prime is encoded as a big-endian sequence of octets
in an {ArrayBuffer}. If the `bigint` option is `true`, then a {bigint}
is provided.

The `size` of the prime will have a direct impact on how long it takes to
generate the prime. The larger the size, the longer it will take. Because
we use OpenSSL's `BN_generate_prime_ex` function, which provides only
minimal control over our ability to interrupt the generation process,
it is not recommended to generate overly large primes, as doing so may make
the process unresponsive.

### `crypto.getCipherInfo(nameOrNid[, options])`

<!-- YAML
added: v15.0.0
-->

* `nameOrNid` {string|number} The name or nid of the cipher to query.
* `options` {Object}
  * `keyLength` {number} A test key length.
  * `ivLength` {number} A test IV length.
* Returns: {Object}
  * `name` {string} The name of the cipher
  * `nid` {number} The nid of the cipher
  * `blockSize` {number} The block size of the cipher in bytes. This property
    is omitted when `mode` is `'stream'`.
  * `ivLength` {number} The expected or default initialization vector length in
    bytes. This property is omitted if the cipher does not use an initialization
    vector.
  * `keyLength` {number} The expected or default key length in bytes.
  * `mode` {string} The cipher mode. One of `'cbc'`, `'ccm'`, `'cfb'`, `'ctr'`,
    `'ecb'`, `'gcm'`, `'ocb'`, `'ofb'`, `'stream'`, `'wrap'`, `'xts'`.

Returns information about a given cipher.

Some ciphers accept variable length keys and initialization vectors. By default,
the `crypto.getCipherInfo()` method will return the default values for these
ciphers. To test if a given key length or iv length is acceptable for given
cipher, use the `keyLength` and `ivLength` options. If the given values are
unacceptable, `undefined` will be returned.

### `crypto.getCiphers()`

<!-- YAML
added: v0.9.3
-->

* Returns: {string\[]} An array with the names of the supported cipher
  algorithms.

```mjs
const {
  getCiphers,
} = await import('node:crypto');

console.log(getCiphers()); // ['aes-128-cbc', 'aes-128-ccm', ...]
```

```cjs
const {
  getCiphers,
} = require('node:crypto');

console.log(getCiphers()); // ['aes-128-cbc', 'aes-128-ccm', ...]
```

### `crypto.getCurves()`

<!-- YAML
added: v2.3.0
-->

* Returns: {string\[]} An array with the names of the supported elliptic curves.

```mjs
const {
  getCurves,
} = await import('node:crypto');

console.log(getCurves()); // ['Oakley-EC2N-3', 'Oakley-EC2N-4', ...]
```

```cjs
const {
  getCurves,
} = require('node:crypto');

console.log(getCurves()); // ['Oakley-EC2N-3', 'Oakley-EC2N-4', ...]
```

### `crypto.getDiffieHellman(groupName)`

<!-- YAML
added: v0.7.5
-->

* `groupName` {string}
* Returns: {DiffieHellmanGroup}

Creates a predefined `DiffieHellmanGroup` key exchange object. The
supported groups are listed in the documentation for [`DiffieHellmanGroup`][].

The returned object mimics the interface of objects created by
[`crypto.createDiffieHellman()`][], but will not allow changing
the keys (with [`diffieHellman.setPublicKey()`][], for example). The
advantage of using this method is that the parties do not have to
generate nor exchange a group modulus beforehand, saving both processor
and communication time.

Example (obtaining a shared secret):

```mjs
const {
  getDiffieHellman,
} = await import('node:crypto');
const alice = getDiffieHellman('modp14');
const bob = getDiffieHellman('modp14');

alice.generateKeys();
bob.generateKeys();

const aliceSecret = alice.computeSecret(bob.getPublicKey(), null, 'hex');
const bobSecret = bob.computeSecret(alice.getPublicKey(), null, 'hex');

/* aliceSecret and bobSecret should be the same */
console.log(aliceSecret === bobSecret);
```

```cjs
const {
  getDiffieHellman,
} = require('node:crypto');

const alice = getDiffieHellman('modp14');
const bob = getDiffieHellman('modp14');

alice.generateKeys();
bob.generateKeys();

const aliceSecret = alice.computeSecret(bob.getPublicKey(), null, 'hex');
const bobSecret = bob.computeSecret(alice.getPublicKey(), null, 'hex');

/* aliceSecret and bobSecret should be the same */
console.log(aliceSecret === bobSecret);
```

### `crypto.getFips()`

<!-- YAML
added: v10.0.0
-->

* Returns: {number} `1` if and only if a FIPS compliant crypto provider is
  currently in use, `0` otherwise. A future semver-major release may change
  the return type of this API to a {boolean}.

### `crypto.getHashes()`

<!-- YAML
added: v0.9.3
-->

* Returns: {string\[]} An array of the names of the supported hash algorithms,
  such as `'RSA-SHA256'`. Hash algorithms are also called "digest" algorithms.

```mjs
const {
  getHashes,
} = await import('node:crypto');

console.log(getHashes()); // ['DSA', 'DSA-SHA', 'DSA-SHA1', ...]
```

```cjs
const {
  getHashes,
} = require('node:crypto');

console.log(getHashes()); // ['DSA', 'DSA-SHA', 'DSA-SHA1', ...]
```

### `crypto.getRandomValues(typedArray)`

<!-- YAML
added: v17.4.0
-->

* `typedArray` {Buffer|TypedArray|DataView|ArrayBuffer}
* Returns: {Buffer|TypedArray|DataView|ArrayBuffer} Returns `typedArray`.

A convenient alias for [`crypto.webcrypto.getRandomValues()`][]. This
implementation is not compliant with the Web Crypto spec, to write
web-compatible code use [`crypto.webcrypto.getRandomValues()`][] instead.

### `crypto.hash(algorithm, data[, options])`

<!-- YAML
added:
 - v21.7.0
 - v20.12.0
changes:
  - version: v24.4.0
    pr-url: https://github.com/nodejs/node/pull/58121
    description: The `outputLength` option was added for XOF hash functions.
-->

> Stability: 1.2 - Release candidate

* `algorithm` {string|undefined}
* `data` {string|Buffer|TypedArray|DataView} When `data` is a
  string, it will be encoded as UTF-8 before being hashed. If a different
  input encoding is desired for a string input, user could encode the string
  into a `TypedArray` using either `TextEncoder` or `Buffer.from()` and passing
  the encoded `TypedArray` into this API instead.
* `options` {Object|string}
  * `outputEncoding` {string} [Encoding][encoding] used to encode the
    returned digest. **Default:** `'hex'`.
  * `outputLength` {number} For XOF hash functions such as 'shake256',
    the outputLength option can be used to specify the desired output length in bytes.
* Returns: {string|Buffer}

A utility for creating one-shot hash digests of data. It can be faster than
the object-based `crypto.createHash()` when hashing a smaller amount of data
(<= 5MB) that's readily available. If the data can be big or if it is streamed,
it's still recommended to use `crypto.createHash()` instead.

The `algorithm` is dependent on the available algorithms supported by the
version of OpenSSL on the platform. Examples are `'sha256'`, `'sha512'`, etc.
On recent releases of OpenSSL, `openssl list -digest-algorithms` will
display the available digest algorithms.

If `options` is a string, then it specifies the `outputEncoding`.

Example:

```cjs
const crypto = require('node:crypto');
const { Buffer } = require('node:buffer');

// Hashing a string and return the result as a hex-encoded string.
const string = 'Node.js';
// 10b3493287f831e81a438811a1ffba01f8cec4b7
console.log(crypto.hash('sha1', string));

// Encode a base64-encoded string into a Buffer, hash it and return
// the result as a buffer.
const base64 = 'Tm9kZS5qcw==';
// <Buffer 10 b3 49 32 87 f8 31 e8 1a 43 88 11 a1 ff ba 01 f8 ce c4 b7>
console.log(crypto.hash('sha1', Buffer.from(base64, 'base64'), 'buffer'));
```

```mjs
import crypto from 'node:crypto';
import { Buffer } from 'node:buffer';

// Hashing a string and return the result as a hex-encoded string.
const string = 'Node.js';
// 10b3493287f831e81a438811a1ffba01f8cec4b7
console.log(crypto.hash('sha1', string));

// Encode a base64-encoded string into a Buffer, hash it and return
// the result as a buffer.
const base64 = 'Tm9kZS5qcw==';
// <Buffer 10 b3 49 32 87 f8 31 e8 1a 43 88 11 a1 ff ba 01 f8 ce c4 b7>
console.log(crypto.hash('sha1', Buffer.from(base64, 'base64'), 'buffer'));
```

### `crypto.hkdf(digest, ikm, salt, info, keylen, callback)`

<!-- YAML
added: v15.0.0
changes:
  - version:
    - v18.8.0
    - v16.18.0
    pr-url: https://github.com/nodejs/node/pull/44201
    description: The input keying material can now be zero-length.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `digest` {string} The digest algorithm to use.
* `ikm` {string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject} The input
  keying material. Must be provided but can be zero-length.
* `salt` {string|ArrayBuffer|Buffer|TypedArray|DataView} The salt value. Must
  be provided but can be zero-length.
* `info` {string|ArrayBuffer|Buffer|TypedArray|DataView} Additional info value.
  Must be provided but can be zero-length, and cannot be more than 1024 bytes.
* `keylen` {number} The length of the key to generate. Must be greater than 0.
  The maximum allowable value is `255` times the number of bytes produced by
  the selected digest function (e.g. `sha512` generates 64-byte hashes, making
  the maximum HKDF output 16320 bytes).
* `callback` {Function}
  * `err` {Error}
  * `derivedKey` {ArrayBuffer}

HKDF is a simple key derivation function defined in RFC 5869. The given `ikm`,
`salt` and `info` are used with the `digest` to derive a key of `keylen` bytes.

The supplied `callback` function is called with two arguments: `err` and
`derivedKey`. If an errors occurs while deriving the key, `err` will be set;
otherwise `err` will be `null`. The successfully generated `derivedKey` will
be passed to the callback as an {ArrayBuffer}. An error will be thrown if any
of the input arguments specify invalid values or types.

```mjs
import { Buffer } from 'node:buffer';
const {
  hkdf,
} = await import('node:crypto');

hkdf('sha512', 'key', 'salt', 'info', 64, (err, derivedKey) => {
  if (err) throw err;
  console.log(Buffer.from(derivedKey).toString('hex'));  // '24156e2...5391653'
});
```

```cjs
const {
  hkdf,
} = require('node:crypto');
const { Buffer } = require('node:buffer');

hkdf('sha512', 'key', 'salt', 'info', 64, (err, derivedKey) => {
  if (err) throw err;
  console.log(Buffer.from(derivedKey).toString('hex'));  // '24156e2...5391653'
});
```

### `crypto.hkdfSync(digest, ikm, salt, info, keylen)`

<!-- YAML
added: v15.0.0
changes:
  - version:
    - v18.8.0
    - v16.18.0
    pr-url: https://github.com/nodejs/node/pull/44201
    description: The input keying material can now be zero-length.
-->

* `digest` {string} The digest algorithm to use.
* `ikm` {string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject} The input
  keying material. Must be provided but can be zero-length.
* `salt` {string|ArrayBuffer|Buffer|TypedArray|DataView} The salt value. Must
  be provided but can be zero-length.
* `info` {string|ArrayBuffer|Buffer|TypedArray|DataView} Additional info value.
  Must be provided but can be zero-length, and cannot be more than 1024 bytes.
* `keylen` {number} The length of the key to generate. Must be greater than 0.
  The maximum allowable value is `255` times the number of bytes produced by
  the selected digest function (e.g. `sha512` generates 64-byte hashes, making
  the maximum HKDF output 16320 bytes).
* Returns: {ArrayBuffer}

Provides a synchronous HKDF key derivation function as defined in RFC 5869. The
given `ikm`, `salt` and `info` are used with the `digest` to derive a key of
`keylen` bytes.

The successfully generated `derivedKey` will be returned as an {ArrayBuffer}.

An error will be thrown if any of the input arguments specify invalid values or
types, or if the derived key cannot be generated.

```mjs
import { Buffer } from 'node:buffer';
const {
  hkdfSync,
} = await import('node:crypto');

const derivedKey = hkdfSync('sha512', 'key', 'salt', 'info', 64);
console.log(Buffer.from(derivedKey).toString('hex'));  // '24156e2...5391653'
```

```cjs
const {
  hkdfSync,
} = require('node:crypto');
const { Buffer } = require('node:buffer');

const derivedKey = hkdfSync('sha512', 'key', 'salt', 'info', 64);
console.log(Buffer.from(derivedKey).toString('hex'));  // '24156e2...5391653'
```

### `crypto.pbkdf2(password, salt, iterations, keylen, digest, callback)`

<!-- YAML
added: v0.5.5
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/35093
    description: The password and salt arguments can also be ArrayBuffer
                 instances.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/30578
    description: The `iterations` parameter is now restricted to positive
                 values. Earlier releases treated other values as one.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/11305
    description: The `digest` parameter is always required now.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/4047
    description: Calling this function without passing the `digest` parameter
                 is deprecated now and will emit a warning.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/5522
    description: The default encoding for `password` if it is a string changed
                 from `binary` to `utf8`.
-->

* `password` {string|ArrayBuffer|Buffer|TypedArray|DataView}
* `salt` {string|ArrayBuffer|Buffer|TypedArray|DataView}
* `iterations` {number}
* `keylen` {number}
* `digest` {string}
* `callback` {Function}
  * `err` {Error}
  * `derivedKey` {Buffer}

Provides an asynchronous Password-Based Key Derivation Function 2 (PBKDF2)
implementation. A selected HMAC digest algorithm specified by `digest` is
applied to derive a key of the requested byte length (`keylen`) from the
`password`, `salt` and `iterations`.

The supplied `callback` function is called with two arguments: `err` and
`derivedKey`. If an error occurs while deriving the key, `err` will be set;
otherwise `err` will be `null`. By default, the successfully generated
`derivedKey` will be passed to the callback as a [`Buffer`][]. An error will be
thrown if any of the input arguments specify invalid values or types.

The `iterations` argument must be a number set as high as possible. The
higher the number of iterations, the more secure the derived key will be,
but will take a longer amount of time to complete.

The `salt` should be as unique as possible. It is recommended that a salt is
random and at least 16 bytes long. See [NIST SP 800-132][] for details.

When passing strings for `password` or `salt`, please consider
[caveats when using strings as inputs to cryptographic APIs][].

```mjs
const {
  pbkdf2,
} = await import('node:crypto');

pbkdf2('secret', 'salt', 100000, 64, 'sha512', (err, derivedKey) => {
  if (err) throw err;
  console.log(derivedKey.toString('hex'));  // '3745e48...08d59ae'
});
```

```cjs
const {
  pbkdf2,
} = require('node:crypto');

pbkdf2('secret', 'salt', 100000, 64, 'sha512', (err, derivedKey) => {
  if (err) throw err;
  console.log(derivedKey.toString('hex'));  // '3745e48...08d59ae'
});
```

An array of supported digest functions can be retrieved using
[`crypto.getHashes()`][].

This API uses libuv's threadpool, which can have surprising and
negative performance implications for some applications; see the
[`UV_THREADPOOL_SIZE`][] documentation for more information.

### `crypto.pbkdf2Sync(password, salt, iterations, keylen, digest)`

<!-- YAML
added: v0.9.3
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/30578
    description: The `iterations` parameter is now restricted to positive
                 values. Earlier releases treated other values as one.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/4047
    description: Calling this function without passing the `digest` parameter
                 is deprecated now and will emit a warning.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/5522
    description: The default encoding for `password` if it is a string changed
                 from `binary` to `utf8`.
-->

* `password` {string|Buffer|TypedArray|DataView}
* `salt` {string|Buffer|TypedArray|DataView}
* `iterations` {number}
* `keylen` {number}
* `digest` {string}
* Returns: {Buffer}

Provides a synchronous Password-Based Key Derivation Function 2 (PBKDF2)
implementation. A selected HMAC digest algorithm specified by `digest` is
applied to derive a key of the requested byte length (`keylen`) from the
`password`, `salt` and `iterations`.

If an error occurs an `Error` will be thrown, otherwise the derived key will be
returned as a [`Buffer`][].

The `iterations` argument must be a number set as high as possible. The
higher the number of iterations, the more secure the derived key will be,
but will take a longer amount of time to complete.

The `salt` should be as unique as possible. It is recommended that a salt is
random and at least 16 bytes long. See [NIST SP 800-132][] for details.

When passing strings for `password` or `salt`, please consider
[caveats when using strings as inputs to cryptographic APIs][].

```mjs
const {
  pbkdf2Sync,
} = await import('node:crypto');

const key = pbkdf2Sync('secret', 'salt', 100000, 64, 'sha512');
console.log(key.toString('hex'));  // '3745e48...08d59ae'
```

```cjs
const {
  pbkdf2Sync,
} = require('node:crypto');

const key = pbkdf2Sync('secret', 'salt', 100000, 64, 'sha512');
console.log(key.toString('hex'));  // '3745e48...08d59ae'
```

An array of supported digest functions can be retrieved using
[`crypto.getHashes()`][].

### `crypto.privateDecrypt(privateKey, buffer)`

<!-- YAML
added: v0.11.14
changes:
  - version:
      - v21.6.2
      - v20.11.1
      - v18.19.1
    pr-url: https://github.com/nodejs-private/node-private/pull/515
    description: The `RSA_PKCS1_PADDING` padding was disabled unless the
                 OpenSSL build supports implicit rejection.
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/35093
    description: Added string, ArrayBuffer, and CryptoKey as allowable key
                 types. The oaepLabel can be an ArrayBuffer. The buffer can
                 be a string or ArrayBuffer. All types that accept buffers
                 are limited to a maximum of 2 ** 31 - 1 bytes.
  - version: v12.11.0
    pr-url: https://github.com/nodejs/node/pull/29489
    description: The `oaepLabel` option was added.
  - version: v12.9.0
    pr-url: https://github.com/nodejs/node/pull/28335
    description: The `oaepHash` option was added.
  - version: v11.6.0
    pr-url: https://github.com/nodejs/node/pull/24234
    description: This function now supports key objects.
-->

<!--lint disable maximum-line-length remark-lint-->

* `privateKey` {Object|string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject|CryptoKey}
  * `oaepHash` {string} The hash function to use for OAEP padding and MGF1.
    **Default:** `'sha1'`
  * `oaepLabel` {string|ArrayBuffer|Buffer|TypedArray|DataView} The label to
    use for OAEP padding. If not specified, no label is used.
  * `padding` {crypto.constants} An optional padding value defined in
    `crypto.constants`, which may be: `crypto.constants.RSA_NO_PADDING`,
    `crypto.constants.RSA_PKCS1_PADDING`, or
    `crypto.constants.RSA_PKCS1_OAEP_PADDING`.
* `buffer` {string|ArrayBuffer|Buffer|TypedArray|DataView}
* Returns: {Buffer} A new `Buffer` with the decrypted content.

<!--lint enable maximum-line-length remark-lint-->

Decrypts `buffer` with `privateKey`. `buffer` was previously encrypted using
the corresponding public key, for example using [`crypto.publicEncrypt()`][].

If `privateKey` is not a [`KeyObject`][], this function behaves as if
`privateKey` had been passed to [`crypto.createPrivateKey()`][]. If it is an
object, the `padding` property can be passed. Otherwise, this function uses
`RSA_PKCS1_OAEP_PADDING`.

Using `crypto.constants.RSA_PKCS1_PADDING` in [`crypto.privateDecrypt()`][]
requires OpenSSL to support implicit rejection (`rsa_pkcs1_implicit_rejection`).
If the version of OpenSSL used by Node.js does not support this feature,
attempting to use `RSA_PKCS1_PADDING` will fail.

### `crypto.privateEncrypt(privateKey, buffer)`

<!-- YAML
added: v1.1.0
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/35093
    description: Added string, ArrayBuffer, and CryptoKey as allowable key
                 types. The passphrase can be an ArrayBuffer. The buffer can
                 be a string or ArrayBuffer. All types that accept buffers
                 are limited to a maximum of 2 ** 31 - 1 bytes.
  - version: v11.6.0
    pr-url: https://github.com/nodejs/node/pull/24234
    description: This function now supports key objects.
-->

<!--lint disable maximum-line-length remark-lint-->

* `privateKey` {Object|string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject|CryptoKey}
  * `key` {string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject|CryptoKey}
    A PEM encoded private key.
  * `passphrase` {string|ArrayBuffer|Buffer|TypedArray|DataView} An optional
    passphrase for the private key.
  * `padding` {crypto.constants} An optional padding value defined in
    `crypto.constants`, which may be: `crypto.constants.RSA_NO_PADDING` or
    `crypto.constants.RSA_PKCS1_PADDING`.
  * `encoding` {string} The string encoding to use when `buffer`, `key`,
    or `passphrase` are strings.
* `buffer` {string|ArrayBuffer|Buffer|TypedArray|DataView}
* Returns: {Buffer} A new `Buffer` with the encrypted content.

<!--lint enable maximum-line-length remark-lint-->

Encrypts `buffer` with `privateKey`. The returned data can be decrypted using
the corresponding public key, for example using [`crypto.publicDecrypt()`][].

If `privateKey` is not a [`KeyObject`][], this function behaves as if
`privateKey` had been passed to [`crypto.createPrivateKey()`][]. If it is an
object, the `padding` property can be passed. Otherwise, this function uses
`RSA_PKCS1_PADDING`.

### `crypto.publicDecrypt(key, buffer)`

<!-- YAML
added: v1.1.0
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/35093
    description: Added string, ArrayBuffer, and CryptoKey as allowable key
                 types. The passphrase can be an ArrayBuffer. The buffer can
                 be a string or ArrayBuffer. All types that accept buffers
                 are limited to a maximum of 2 ** 31 - 1 bytes.
  - version: v11.6.0
    pr-url: https://github.com/nodejs/node/pull/24234
    description: This function now supports key objects.
-->

<!--lint disable maximum-line-length remark-lint-->

* `key` {Object|string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject|CryptoKey}
  * `passphrase` {string|ArrayBuffer|Buffer|TypedArray|DataView} An optional
    passphrase for the private key.
  * `padding` {crypto.constants} An optional padding value defined in
    `crypto.constants`, which may be: `crypto.constants.RSA_NO_PADDING` or
    `crypto.constants.RSA_PKCS1_PADDING`.
  * `encoding` {string} The string encoding to use when `buffer`, `key`,
    or `passphrase` are strings.
* `buffer` {string|ArrayBuffer|Buffer|TypedArray|DataView}
* Returns: {Buffer} A new `Buffer` with the decrypted content.

<!--lint enable maximum-line-length remark-lint-->

Decrypts `buffer` with `key`.`buffer` was previously encrypted using
the corresponding private key, for example using [`crypto.privateEncrypt()`][].

If `key` is not a [`KeyObject`][], this function behaves as if
`key` had been passed to [`crypto.createPublicKey()`][]. If it is an
object, the `padding` property can be passed. Otherwise, this function uses
`RSA_PKCS1_PADDING`.

Because RSA public keys can be derived from private keys, a private key may
be passed instead of a public key.

### `crypto.publicEncrypt(key, buffer)`

<!-- YAML
added: v0.11.14
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/35093
    description: Added string, ArrayBuffer, and CryptoKey as allowable key
                 types. The oaepLabel and passphrase can be ArrayBuffers. The
                 buffer can be a string or ArrayBuffer. All types that accept
                 buffers are limited to a maximum of 2 ** 31 - 1 bytes.
  - version: v12.11.0
    pr-url: https://github.com/nodejs/node/pull/29489
    description: The `oaepLabel` option was added.
  - version: v12.9.0
    pr-url: https://github.com/nodejs/node/pull/28335
    description: The `oaepHash` option was added.
  - version: v11.6.0
    pr-url: https://github.com/nodejs/node/pull/24234
    description: This function now supports key objects.
-->

<!--lint disable maximum-line-length remark-lint-->

* `key` {Object|string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject|CryptoKey}
  * `key` {string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject|CryptoKey}
    A PEM encoded public or private key, {KeyObject}, or {CryptoKey}.
  * `oaepHash` {string} The hash function to use for OAEP padding and MGF1.
    **Default:** `'sha1'`
  * `oaepLabel` {string|ArrayBuffer|Buffer|TypedArray|DataView} The label to
    use for OAEP padding. If not specified, no label is used.
  * `passphrase` {string|ArrayBuffer|Buffer|TypedArray|DataView} An optional
    passphrase for the private key.
  * `padding` {crypto.constants} An optional padding value defined in
    `crypto.constants`, which may be: `crypto.constants.RSA_NO_PADDING`,
    `crypto.constants.RSA_PKCS1_PADDING`, or
    `crypto.constants.RSA_PKCS1_OAEP_PADDING`.
  * `encoding` {string} The string encoding to use when `buffer`, `key`,
    `oaepLabel`, or `passphrase` are strings.
* `buffer` {string|ArrayBuffer|Buffer|TypedArray|DataView}
* Returns: {Buffer} A new `Buffer` with the encrypted content.

<!--lint enable maximum-line-length remark-lint-->

Encrypts the content of `buffer` with `key` and returns a new
[`Buffer`][] with encrypted content. The returned data can be decrypted using
the corresponding private key, for example using [`crypto.privateDecrypt()`][].

If `key` is not a [`KeyObject`][], this function behaves as if
`key` had been passed to [`crypto.createPublicKey()`][]. If it is an
object, the `padding` property can be passed. Otherwise, this function uses
`RSA_PKCS1_OAEP_PADDING`.

Because RSA public keys can be derived from private keys, a private key may
be passed instead of a public key.

### `crypto.randomBytes(size[, callback])`

<!-- YAML
added: v0.5.8
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/16454
    description: Passing `null` as the `callback` argument now throws
                 `ERR_INVALID_CALLBACK`.
-->

* `size` {number} The number of bytes to generate.  The `size` must
  not be larger than `2**31 - 1`.
* `callback` {Function}
  * `err` {Error}
  * `buf` {Buffer}
* Returns: {Buffer} if the `callback` function is not provided.

Generates cryptographically strong pseudorandom data. The `size` argument
is a number indicating the number of bytes to generate.

If a `callback` function is provided, the bytes are generated asynchronously
and the `callback` function is invoked with two arguments: `err` and `buf`.
If an error occurs, `err` will be an `Error` object; otherwise it is `null`. The
`buf` argument is a [`Buffer`][] containing the generated bytes.

```mjs
// Asynchronous
const {
  randomBytes,
} = await import('node:crypto');

randomBytes(256, (err, buf) => {
  if (err) throw err;
  console.log(`${buf.length} bytes of random data: ${buf.toString('hex')}`);
});
```

```cjs
// Asynchronous
const {
  randomBytes,
} = require('node:crypto');

randomBytes(256, (err, buf) => {
  if (err) throw err;
  console.log(`${buf.length} bytes of random data: ${buf.toString('hex')}`);
});
```

If the `callback` function is not provided, the random bytes are generated
synchronously and returned as a [`Buffer`][]. An error will be thrown if
there is a problem generating the bytes.

```mjs
// Synchronous
const {
  randomBytes,
} = await import('node:crypto');

const buf = randomBytes(256);
console.log(
  `${buf.length} bytes of random data: ${buf.toString('hex')}`);
```

```cjs
// Synchronous
const {
  randomBytes,
} = require('node:crypto');

const buf = randomBytes(256);
console.log(
  `${buf.length} bytes of random data: ${buf.toString('hex')}`);
```

The `crypto.randomBytes()` method will not complete until there is
sufficient entropy available.
This should normally never take longer than a few milliseconds. The only time
when generating the random bytes may conceivably block for a longer period of
time is right after boot, when the whole system is still low on entropy.

This API uses libuv's threadpool, which can have surprising and
negative performance implications for some applications; see the
[`UV_THREADPOOL_SIZE`][] documentation for more information.

The asynchronous version of `crypto.randomBytes()` is carried out in a single
threadpool request. To minimize threadpool task length variation, partition
large `randomBytes` requests when doing so as part of fulfilling a client
request.

### `crypto.randomFill(buffer[, offset][, size], callback)`

<!-- YAML
added:
  - v7.10.0
  - v6.13.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15231
    description: The `buffer` argument may be any `TypedArray` or `DataView`.
-->

* `buffer` {ArrayBuffer|Buffer|TypedArray|DataView} Must be supplied. The
  size of the provided `buffer` must not be larger than `2**31 - 1`.
* `offset` {number} **Default:** `0`
* `size` {number} **Default:** `buffer.length - offset`. The `size` must
  not be larger than `2**31 - 1`.
* `callback` {Function} `function(err, buf) {}`.

This function is similar to [`crypto.randomBytes()`][] but requires the first
argument to be a [`Buffer`][] that will be filled. It also
requires that a callback is passed in.

If the `callback` function is not provided, an error will be thrown.

```mjs
import { Buffer } from 'node:buffer';
const { randomFill } = await import('node:crypto');

const buf = Buffer.alloc(10);
randomFill(buf, (err, buf) => {
  if (err) throw err;
  console.log(buf.toString('hex'));
});

randomFill(buf, 5, (err, buf) => {
  if (err) throw err;
  console.log(buf.toString('hex'));
});

// The above is equivalent to the following:
randomFill(buf, 5, 5, (err, buf) => {
  if (err) throw err;
  console.log(buf.toString('hex'));
});
```

```cjs
const { randomFill } = require('node:crypto');
const { Buffer } = require('node:buffer');

const buf = Buffer.alloc(10);
randomFill(buf, (err, buf) => {
  if (err) throw err;
  console.log(buf.toString('hex'));
});

randomFill(buf, 5, (err, buf) => {
  if (err) throw err;
  console.log(buf.toString('hex'));
});

// The above is equivalent to the following:
randomFill(buf, 5, 5, (err, buf) => {
  if (err) throw err;
  console.log(buf.toString('hex'));
});
```

Any `ArrayBuffer`, `TypedArray`, or `DataView` instance may be passed as
`buffer`.

While this includes instances of `Float32Array` and `Float64Array`, this
function should not be used to generate random floating-point numbers. The
result may contain `+Infinity`, `-Infinity`, and `NaN`, and even if the array
contains finite numbers only, they are not drawn from a uniform random
distribution and have no meaningful lower or upper bounds.

```mjs
import { Buffer } from 'node:buffer';
const { randomFill } = await import('node:crypto');

const a = new Uint32Array(10);
randomFill(a, (err, buf) => {
  if (err) throw err;
  console.log(Buffer.from(buf.buffer, buf.byteOffset, buf.byteLength)
    .toString('hex'));
});

const b = new DataView(new ArrayBuffer(10));
randomFill(b, (err, buf) => {
  if (err) throw err;
  console.log(Buffer.from(buf.buffer, buf.byteOffset, buf.byteLength)
    .toString('hex'));
});

const c = new ArrayBuffer(10);
randomFill(c, (err, buf) => {
  if (err) throw err;
  console.log(Buffer.from(buf).toString('hex'));
});
```

```cjs
const { randomFill } = require('node:crypto');
const { Buffer } = require('node:buffer');

const a = new Uint32Array(10);
randomFill(a, (err, buf) => {
  if (err) throw err;
  console.log(Buffer.from(buf.buffer, buf.byteOffset, buf.byteLength)
    .toString('hex'));
});

const b = new DataView(new ArrayBuffer(10));
randomFill(b, (err, buf) => {
  if (err) throw err;
  console.log(Buffer.from(buf.buffer, buf.byteOffset, buf.byteLength)
    .toString('hex'));
});

const c = new ArrayBuffer(10);
randomFill(c, (err, buf) => {
  if (err) throw err;
  console.log(Buffer.from(buf).toString('hex'));
});
```

This API uses libuv's threadpool, which can have surprising and
negative performance implications for some applications; see the
[`UV_THREADPOOL_SIZE`][] documentation for more information.

The asynchronous version of `crypto.randomFill()` is carried out in a single
threadpool request. To minimize threadpool task length variation, partition
large `randomFill` requests when doing so as part of fulfilling a client
request.

### `crypto.randomFillSync(buffer[, offset][, size])`

<!-- YAML
added:
  - v7.10.0
  - v6.13.0
changes:
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15231
    description: The `buffer` argument may be any `TypedArray` or `DataView`.
-->

* `buffer` {ArrayBuffer|Buffer|TypedArray|DataView} Must be supplied. The
  size of the provided `buffer` must not be larger than `2**31 - 1`.
* `offset` {number} **Default:** `0`
* `size` {number} **Default:** `buffer.length - offset`. The `size` must
  not be larger than `2**31 - 1`.
* Returns: {ArrayBuffer|Buffer|TypedArray|DataView} The object passed as
  `buffer` argument.

Synchronous version of [`crypto.randomFill()`][].

```mjs
import { Buffer } from 'node:buffer';
const { randomFillSync } = await import('node:crypto');

const buf = Buffer.alloc(10);
console.log(randomFillSync(buf).toString('hex'));

randomFillSync(buf, 5);
console.log(buf.toString('hex'));

// The above is equivalent to the following:
randomFillSync(buf, 5, 5);
console.log(buf.toString('hex'));
```

```cjs
const { randomFillSync } = require('node:crypto');
const { Buffer } = require('node:buffer');

const buf = Buffer.alloc(10);
console.log(randomFillSync(buf).toString('hex'));

randomFillSync(buf, 5);
console.log(buf.toString('hex'));

// The above is equivalent to the following:
randomFillSync(buf, 5, 5);
console.log(buf.toString('hex'));
```

Any `ArrayBuffer`, `TypedArray` or `DataView` instance may be passed as
`buffer`.

```mjs
import { Buffer } from 'node:buffer';
const { randomFillSync } = await import('node:crypto');

const a = new Uint32Array(10);
console.log(Buffer.from(randomFillSync(a).buffer,
                        a.byteOffset, a.byteLength).toString('hex'));

const b = new DataView(new ArrayBuffer(10));
console.log(Buffer.from(randomFillSync(b).buffer,
                        b.byteOffset, b.byteLength).toString('hex'));

const c = new ArrayBuffer(10);
console.log(Buffer.from(randomFillSync(c)).toString('hex'));
```

```cjs
const { randomFillSync } = require('node:crypto');
const { Buffer } = require('node:buffer');

const a = new Uint32Array(10);
console.log(Buffer.from(randomFillSync(a).buffer,
                        a.byteOffset, a.byteLength).toString('hex'));

const b = new DataView(new ArrayBuffer(10));
console.log(Buffer.from(randomFillSync(b).buffer,
                        b.byteOffset, b.byteLength).toString('hex'));

const c = new ArrayBuffer(10);
console.log(Buffer.from(randomFillSync(c)).toString('hex'));
```

### `crypto.randomInt([min, ]max[, callback])`

<!-- YAML
added:
  - v14.10.0
  - v12.19.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `min` {integer} Start of random range (inclusive). **Default:** `0`.
* `max` {integer} End of random range (exclusive).
* `callback` {Function} `function(err, n) {}`.

Return a random integer `n` such that `min <= n < max`.  This
implementation avoids [modulo bias][].

The range (`max - min`) must be less than 2<sup>48</sup>. `min` and `max` must
be [safe integers][].

If the `callback` function is not provided, the random integer is
generated synchronously.

```mjs
// Asynchronous
const {
  randomInt,
} = await import('node:crypto');

randomInt(3, (err, n) => {
  if (err) throw err;
  console.log(`Random number chosen from (0, 1, 2): ${n}`);
});
```

```cjs
// Asynchronous
const {
  randomInt,
} = require('node:crypto');

randomInt(3, (err, n) => {
  if (err) throw err;
  console.log(`Random number chosen from (0, 1, 2): ${n}`);
});
```

```mjs
// Synchronous
const {
  randomInt,
} = await import('node:crypto');

const n = randomInt(3);
console.log(`Random number chosen from (0, 1, 2): ${n}`);
```

```cjs
// Synchronous
const {
  randomInt,
} = require('node:crypto');

const n = randomInt(3);
console.log(`Random number chosen from (0, 1, 2): ${n}`);
```

```mjs
// With `min` argument
const {
  randomInt,
} = await import('node:crypto');

const n = randomInt(1, 7);
console.log(`The dice rolled: ${n}`);
```

```cjs
// With `min` argument
const {
  randomInt,
} = require('node:crypto');

const n = randomInt(1, 7);
console.log(`The dice rolled: ${n}`);
```

### `crypto.randomUUID([options])`

<!-- YAML
added:
  - v15.6.0
  - v14.17.0
-->

* `options` {Object}
  * `disableEntropyCache` {boolean} By default, to improve performance,
    Node.js generates and caches enough
    random data to generate up to 128 random UUIDs. To generate a UUID
    without using the cache, set `disableEntropyCache` to `true`.
    **Default:** `false`.
* Returns: {string}

Generates a random [RFC 4122][] version 4 UUID. The UUID is generated using a
cryptographic pseudorandom number generator.

### `crypto.scrypt(password, salt, keylen[, options], callback)`

<!-- YAML
added: v10.5.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/35093
    description: The password and salt arguments can also be ArrayBuffer
                 instances.
  - version:
     - v12.8.0
     - v10.17.0
    pr-url: https://github.com/nodejs/node/pull/28799
    description: The `maxmem` value can now be any safe integer.
  - version: v10.9.0
    pr-url: https://github.com/nodejs/node/pull/21525
    description: The `cost`, `blockSize` and `parallelization` option names
                 have been added.
-->

* `password` {string|ArrayBuffer|Buffer|TypedArray|DataView}
* `salt` {string|ArrayBuffer|Buffer|TypedArray|DataView}
* `keylen` {number}
* `options` {Object}
  * `cost` {number} CPU/memory cost parameter. Must be a power of two greater
    than one. **Default:** `16384`.
  * `blockSize` {number} Block size parameter. **Default:** `8`.
  * `parallelization` {number} Parallelization parameter. **Default:** `1`.
  * `N` {number} Alias for `cost`. Only one of both may be specified.
  * `r` {number} Alias for `blockSize`. Only one of both may be specified.
  * `p` {number} Alias for `parallelization`. Only one of both may be specified.
  * `maxmem` {number} Memory upper bound. It is an error when (approximately)
    `128 * N * r > maxmem`. **Default:** `32 * 1024 * 1024`.
* `callback` {Function}
  * `err` {Error}
  * `derivedKey` {Buffer}

Provides an asynchronous [scrypt][] implementation. Scrypt is a password-based
key derivation function that is designed to be expensive computationally and
memory-wise in order to make brute-force attacks unrewarding.

The `salt` should be as unique as possible. It is recommended that a salt is
random and at least 16 bytes long. See [NIST SP 800-132][] for details.

When passing strings for `password` or `salt`, please consider
[caveats when using strings as inputs to cryptographic APIs][].

The `callback` function is called with two arguments: `err` and `derivedKey`.
`err` is an exception object when key derivation fails, otherwise `err` is
`null`. `derivedKey` is passed to the callback as a [`Buffer`][].

An exception is thrown when any of the input arguments specify invalid values
or types.

```mjs
const {
  scrypt,
} = await import('node:crypto');

// Using the factory defaults.
scrypt('password', 'salt', 64, (err, derivedKey) => {
  if (err) throw err;
  console.log(derivedKey.toString('hex'));  // '3745e48...08d59ae'
});
// Using a custom N parameter. Must be a power of two.
scrypt('password', 'salt', 64, { N: 1024 }, (err, derivedKey) => {
  if (err) throw err;
  console.log(derivedKey.toString('hex'));  // '3745e48...aa39b34'
});
```

```cjs
const {
  scrypt,
} = require('node:crypto');

// Using the factory defaults.
scrypt('password', 'salt', 64, (err, derivedKey) => {
  if (err) throw err;
  console.log(derivedKey.toString('hex'));  // '3745e48...08d59ae'
});
// Using a custom N parameter. Must be a power of two.
scrypt('password', 'salt', 64, { N: 1024 }, (err, derivedKey) => {
  if (err) throw err;
  console.log(derivedKey.toString('hex'));  // '3745e48...aa39b34'
});
```

### `crypto.scryptSync(password, salt, keylen[, options])`

<!-- YAML
added: v10.5.0
changes:
  - version:
     - v12.8.0
     - v10.17.0
    pr-url: https://github.com/nodejs/node/pull/28799
    description: The `maxmem` value can now be any safe integer.
  - version: v10.9.0
    pr-url: https://github.com/nodejs/node/pull/21525
    description: The `cost`, `blockSize` and `parallelization` option names
                 have been added.
-->

* `password` {string|Buffer|TypedArray|DataView}
* `salt` {string|Buffer|TypedArray|DataView}
* `keylen` {number}
* `options` {Object}
  * `cost` {number} CPU/memory cost parameter. Must be a power of two greater
    than one. **Default:** `16384`.
  * `blockSize` {number} Block size parameter. **Default:** `8`.
  * `parallelization` {number} Parallelization parameter. **Default:** `1`.
  * `N` {number} Alias for `cost`. Only one of both may be specified.
  * `r` {number} Alias for `blockSize`. Only one of both may be specified.
  * `p` {number} Alias for `parallelization`. Only one of both may be specified.
  * `maxmem` {number} Memory upper bound. It is an error when (approximately)
    `128 * N * r > maxmem`. **Default:** `32 * 1024 * 1024`.
* Returns: {Buffer}

Provides a synchronous [scrypt][] implementation. Scrypt is a password-based
key derivation function that is designed to be expensive computationally and
memory-wise in order to make brute-force attacks unrewarding.

The `salt` should be as unique as possible. It is recommended that a salt is
random and at least 16 bytes long. See [NIST SP 800-132][] for details.

When passing strings for `password` or `salt`, please consider
[caveats when using strings as inputs to cryptographic APIs][].

An exception is thrown when key derivation fails, otherwise the derived key is
returned as a [`Buffer`][].

An exception is thrown when any of the input arguments specify invalid values
or types.

```mjs
const {
  scryptSync,
} = await import('node:crypto');
// Using the factory defaults.

const key1 = scryptSync('password', 'salt', 64);
console.log(key1.toString('hex'));  // '3745e48...08d59ae'
// Using a custom N parameter. Must be a power of two.
const key2 = scryptSync('password', 'salt', 64, { N: 1024 });
console.log(key2.toString('hex'));  // '3745e48...aa39b34'
```

```cjs
const {
  scryptSync,
} = require('node:crypto');
// Using the factory defaults.

const key1 = scryptSync('password', 'salt', 64);
console.log(key1.toString('hex'));  // '3745e48...08d59ae'
// Using a custom N parameter. Must be a power of two.
const key2 = scryptSync('password', 'salt', 64, { N: 1024 });
console.log(key2.toString('hex'));  // '3745e48...aa39b34'
```

### `crypto.secureHeapUsed()`

<!-- YAML
added: v15.6.0
-->

* Returns: {Object}
  * `total` {number} The total allocated secure heap size as specified
    using the `--secure-heap=n` command-line flag.
  * `min` {number} The minimum allocation from the secure heap as
    specified using the `--secure-heap-min` command-line flag.
  * `used` {number} The total number of bytes currently allocated from
    the secure heap.
  * `utilization` {number} The calculated ratio of `used` to `total`
    allocated bytes.

### `crypto.setEngine(engine[, flags])`

<!-- YAML
added: v0.11.11
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53329
    description: Custom engine support in OpenSSL 3 is deprecated.
-->

* `engine` {string}
* `flags` {crypto.constants} **Default:** `crypto.constants.ENGINE_METHOD_ALL`

Load and set the `engine` for some or all OpenSSL functions (selected by flags).
Support for custom engines in OpenSSL is deprecated from OpenSSL 3.

`engine` could be either an id or a path to the engine's shared library.

The optional `flags` argument uses `ENGINE_METHOD_ALL` by default. The `flags`
is a bit field taking one of or a mix of the following flags (defined in
`crypto.constants`):

* `crypto.constants.ENGINE_METHOD_RSA`
* `crypto.constants.ENGINE_METHOD_DSA`
* `crypto.constants.ENGINE_METHOD_DH`
* `crypto.constants.ENGINE_METHOD_RAND`
* `crypto.constants.ENGINE_METHOD_EC`
* `crypto.constants.ENGINE_METHOD_CIPHERS`
* `crypto.constants.ENGINE_METHOD_DIGESTS`
* `crypto.constants.ENGINE_METHOD_PKEY_METHS`
* `crypto.constants.ENGINE_METHOD_PKEY_ASN1_METHS`
* `crypto.constants.ENGINE_METHOD_ALL`
* `crypto.constants.ENGINE_METHOD_NONE`

### `crypto.setFips(bool)`

<!-- YAML
added: v10.0.0
-->

* `bool` {boolean} `true` to enable FIPS mode.

Enables the FIPS compliant crypto provider in a FIPS-enabled Node.js build.
Throws an error if FIPS mode is not available.

### `crypto.sign(algorithm, data, key[, callback])`

<!-- YAML
added: v12.0.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59570
    description: Add support for ML-DSA, Ed448, and SLH-DSA context parameter.
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59537
    description: Add support for SLH-DSA signing.
  - version: v24.6.0
    pr-url: https://github.com/nodejs/node/pull/59259
    description: Add support for ML-DSA signing.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v15.12.0
    pr-url: https://github.com/nodejs/node/pull/37500
    description: Optional callback argument added.
  - version:
     - v13.2.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/29292
    description: This function now supports IEEE-P1363 DSA and ECDSA signatures.
-->

<!--lint disable maximum-line-length remark-lint-->

* `algorithm` {string | null | undefined}
* `data` {ArrayBuffer|Buffer|TypedArray|DataView}
* `key` {Object|string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject|CryptoKey}
* `callback` {Function}
  * `err` {Error}
  * `signature` {Buffer}
* Returns: {Buffer} if the `callback` function is not provided.

<!--lint enable maximum-line-length remark-lint-->

Calculates and returns the signature for `data` using the given private key and
algorithm. If `algorithm` is `null` or `undefined`, then the algorithm is
dependent upon the key type.

`algorithm` is required to be `null` or `undefined` for Ed25519, Ed448, and
ML-DSA.

If `key` is not a [`KeyObject`][], this function behaves as if `key` had been
passed to [`crypto.createPrivateKey()`][]. If it is an object, the following
additional properties can be passed:

* `dsaEncoding` {string} For DSA and ECDSA, this option specifies the
  format of the generated signature. It can be one of the following:
  * `'der'` (default): DER-encoded ASN.1 signature structure encoding `(r, s)`.
  * `'ieee-p1363'`: Signature format `r || s` as proposed in IEEE-P1363.
* `padding` {integer} Optional padding value for RSA, one of the following:

  * `crypto.constants.RSA_PKCS1_PADDING` (default)
  * `crypto.constants.RSA_PKCS1_PSS_PADDING`

  `RSA_PKCS1_PSS_PADDING` will use MGF1 with the same hash function
  used to sign the message as specified in section 3.1 of [RFC 4055][].
* `saltLength` {integer} Salt length for when padding is
  `RSA_PKCS1_PSS_PADDING`. The special value
  `crypto.constants.RSA_PSS_SALTLEN_DIGEST` sets the salt length to the digest
  size, `crypto.constants.RSA_PSS_SALTLEN_MAX_SIGN` (default) sets it to the
  maximum permissible value.
* `context` {ArrayBuffer|Buffer|TypedArray|DataView} For Ed448, ML-DSA, and SLH-DSA,
  this option specifies the optional context to differentiate signatures generated
  for different purposes with the same key.

If the `callback` function is provided this function uses libuv's threadpool.

### `crypto.subtle`

<!-- YAML
added: v17.4.0
-->

* Type: {SubtleCrypto}

A convenient alias for [`crypto.webcrypto.subtle`][].

### `crypto.timingSafeEqual(a, b)`

<!-- YAML
added: v6.6.0
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/35093
    description: The a and b arguments can also be ArrayBuffer.
-->

* `a` {ArrayBuffer|Buffer|TypedArray|DataView}
* `b` {ArrayBuffer|Buffer|TypedArray|DataView}
* Returns: {boolean}

This function compares the underlying bytes that represent the given
`ArrayBuffer`, `TypedArray`, or `DataView` instances using a constant-time
algorithm.

This function does not leak timing information that
would allow an attacker to guess one of the values. This is suitable for
comparing HMAC digests or secret values like authentication cookies or
[capability urls](https://www.w3.org/TR/capability-urls/).

`a` and `b` must both be `Buffer`s, `TypedArray`s, or `DataView`s, and they
must have the same byte length. An error is thrown if `a` and `b` have
different byte lengths.

If at least one of `a` and `b` is a `TypedArray` with more than one byte per
entry, such as `Uint16Array`, the result will be computed using the platform
byte order.

<strong class="critical">When both of the inputs are `Float32Array`s or
`Float64Array`s, this function might return unexpected results due to IEEE 754
encoding of floating-point numbers. In particular, neither `x === y` nor
`Object.is(x, y)` implies that the byte representations of two floating-point
numbers `x` and `y` are equal.</strong>

Use of `crypto.timingSafeEqual` does not guarantee that the _surrounding_ code
is timing-safe. Care should be taken to ensure that the surrounding code does
not introduce timing vulnerabilities.

### `crypto.verify(algorithm, data, key, signature[, callback])`

<!-- YAML
added: v12.0.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59570
    description: Add support for ML-DSA, Ed448, and SLH-DSA context parameter.
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59537
    description: Add support for SLH-DSA signature verification.
  - version: v24.6.0
    pr-url: https://github.com/nodejs/node/pull/59259
    description: Add support for ML-DSA signature verification.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v15.12.0
    pr-url: https://github.com/nodejs/node/pull/37500
    description: Optional callback argument added.
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/35093
    description: The data, key, and signature arguments can also be ArrayBuffer.
  - version:
     - v13.2.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/29292
    description: This function now supports IEEE-P1363 DSA and ECDSA signatures.
-->

<!--lint disable maximum-line-length remark-lint-->

* `algorithm` {string|null|undefined}
* `data` {ArrayBuffer| Buffer|TypedArray|DataView}
* `key` {Object|string|ArrayBuffer|Buffer|TypedArray|DataView|KeyObject|CryptoKey}
* `signature` {ArrayBuffer|Buffer|TypedArray|DataView}
* `callback` {Function}
  * `err` {Error}
  * `result` {boolean}
* Returns: {boolean} `true` or `false` depending on the validity of the
  signature for the data and public key if the `callback` function is not
  provided.

<!--lint enable maximum-line-length remark-lint-->

Verifies the given signature for `data` using the given key and algorithm. If
`algorithm` is `null` or `undefined`, then the algorithm is dependent upon the
key type.

`algorithm` is required to be `null` or `undefined` for Ed25519, Ed448, and
ML-DSA.

If `key` is not a [`KeyObject`][], this function behaves as if `key` had been
passed to [`crypto.createPublicKey()`][]. If it is an object, the following
additional properties can be passed:

* `dsaEncoding` {string} For DSA and ECDSA, this option specifies the
  format of the signature. It can be one of the following:
  * `'der'` (default): DER-encoded ASN.1 signature structure encoding `(r, s)`.
  * `'ieee-p1363'`: Signature format `r || s` as proposed in IEEE-P1363.
* `padding` {integer} Optional padding value for RSA, one of the following:

  * `crypto.constants.RSA_PKCS1_PADDING` (default)
  * `crypto.constants.RSA_PKCS1_PSS_PADDING`

  `RSA_PKCS1_PSS_PADDING` will use MGF1 with the same hash function
  used to sign the message as specified in section 3.1 of [RFC 4055][].
* `saltLength` {integer} Salt length for when padding is
  `RSA_PKCS1_PSS_PADDING`. The special value
  `crypto.constants.RSA_PSS_SALTLEN_DIGEST` sets the salt length to the digest
  size, `crypto.constants.RSA_PSS_SALTLEN_MAX_SIGN` (default) sets it to the
  maximum permissible value.
* `context` {ArrayBuffer|Buffer|TypedArray|DataView} For Ed448, ML-DSA, and SLH-DSA,
  this option specifies the optional context to differentiate signatures generated
  for different purposes with the same key.

The `signature` argument is the previously calculated signature for the `data`.

Because public keys can be derived from private keys, a private key or a public
key may be passed for `key`.

If the `callback` function is provided this function uses libuv's threadpool.

### `crypto.webcrypto`

<!-- YAML
added: v15.0.0
-->

Type: {Crypto} An implementation of the Web Crypto API standard.

See the [Web Crypto API documentation][] for details.

## Notes

### Using strings as inputs to cryptographic APIs

For historical reasons, many cryptographic APIs provided by Node.js accept
strings as inputs where the underlying cryptographic algorithm works on byte
sequences. These instances include plaintexts, ciphertexts, symmetric keys,
initialization vectors, passphrases, salts, authentication tags,
and additional authenticated data.

When passing strings to cryptographic APIs, consider the following factors.

* Not all byte sequences are valid UTF-8 strings. Therefore, when a byte
  sequence of length `n` is derived from a string, its entropy is generally
  lower than the entropy of a random or pseudorandom `n` byte sequence.
  For example, no UTF-8 string will result in the byte sequence `c0 af`. Secret
  keys should almost exclusively be random or pseudorandom byte sequences.
* Similarly, when converting random or pseudorandom byte sequences to UTF-8
  strings, subsequences that do not represent valid code points may be replaced
  by the Unicode replacement character (`U+FFFD`). The byte representation of
  the resulting Unicode string may, therefore, not be equal to the byte sequence
  that the string was created from.

  ```js
  const original = [0xc0, 0xaf];
  const bytesAsString = Buffer.from(original).toString('utf8');
  const stringAsBytes = Buffer.from(bytesAsString, 'utf8');
  console.log(stringAsBytes);
  // Prints '<Buffer ef bf bd ef bf bd>'.
  ```

  The outputs of ciphers, hash functions, signature algorithms, and key
  derivation functions are pseudorandom byte sequences and should not be
  used as Unicode strings.
* When strings are obtained from user input, some Unicode characters can be
  represented in multiple equivalent ways that result in different byte
  sequences. For example, when passing a user passphrase to a key derivation
  function, such as PBKDF2 or scrypt, the result of the key derivation function
  depends on whether the string uses composed or decomposed characters. Node.js
  does not normalize character representations. Developers should consider using
  [`String.prototype.normalize()`][] on user inputs before passing them to
  cryptographic APIs.

### Legacy streams API (prior to Node.js 0.10)

The Crypto module was added to Node.js before there was the concept of a
unified Stream API, and before there were [`Buffer`][] objects for handling
binary data. As such, many `crypto` classes have methods not
typically found on other Node.js classes that implement the [streams][stream]
API (e.g. `update()`, `final()`, or `digest()`). Also, many methods accepted
and returned `'latin1'` encoded strings by default rather than `Buffer`s. This
default was changed after Node.js v0.8 to use [`Buffer`][] objects by default
instead.

### Support for weak or compromised algorithms

The `node:crypto` module still supports some algorithms which are already
compromised and are not recommended for use. The API also allows
the use of ciphers and hashes with a small key size that are too weak for safe
use.

Users should take full responsibility for selecting the crypto
algorithm and key size according to their security requirements.

Based on the recommendations of [NIST SP 800-131A][]:

* MD5 and SHA-1 are no longer acceptable where collision resistance is
  required such as digital signatures.
* The key used with RSA, DSA, and DH algorithms is recommended to have
  at least 2048 bits and that of the curve of ECDSA and ECDH at least
  224 bits, to be safe to use for several years.
* The DH groups of `modp1`, `modp2` and `modp5` have a key size
  smaller than 2048 bits and are not recommended.

See the reference for other recommendations and details.

Some algorithms that have known weaknesses and are of little relevance in
practice are only available through the [legacy provider][], which is not
enabled by default.

### CCM mode

CCM is one of the supported [AEAD algorithms][]. Applications which use this
mode must adhere to certain restrictions when using the cipher API:

* The authentication tag length must be specified during cipher creation by
  setting the `authTagLength` option and must be one of 4, 6, 8, 10, 12, 14 or
  16 bytes.
* The length of the initialization vector (nonce) `N` must be between 7 and 13
  bytes (`7 ≤ N ≤ 13`).
* The length of the plaintext is limited to `2 ** (8 * (15 - N))` bytes.
* When decrypting, the authentication tag must be set via `setAuthTag()` before
  calling `update()`.
  Otherwise, decryption will fail and `final()` will throw an error in
  compliance with section 2.6 of [RFC 3610][].
* Using stream methods such as `write(data)`, `end(data)` or `pipe()` in CCM
  mode might fail as CCM cannot handle more than one chunk of data per instance.
* When passing additional authenticated data (AAD), the length of the actual
  message in bytes must be passed to `setAAD()` via the `plaintextLength`
  option.
  Many crypto libraries include the authentication tag in the ciphertext,
  which means that they produce ciphertexts of the length
  `plaintextLength + authTagLength`. Node.js does not include the authentication
  tag, so the ciphertext length is always `plaintextLength`.
  This is not necessary if no AAD is used.
* As CCM processes the whole message at once, `update()` must be called exactly
  once.
* Even though calling `update()` is sufficient to encrypt/decrypt the message,
  applications _must_ call `final()` to compute or verify the
  authentication tag.

```mjs
import { Buffer } from 'node:buffer';
const {
  createCipheriv,
  createDecipheriv,
  randomBytes,
} = await import('node:crypto');

const key = 'keykeykeykeykeykeykeykey';
const nonce = randomBytes(12);

const aad = Buffer.from('0123456789', 'hex');

const cipher = createCipheriv('aes-192-ccm', key, nonce, {
  authTagLength: 16,
});
const plaintext = 'Hello world';
cipher.setAAD(aad, {
  plaintextLength: Buffer.byteLength(plaintext),
});
const ciphertext = cipher.update(plaintext, 'utf8');
cipher.final();
const tag = cipher.getAuthTag();

// Now transmit { ciphertext, nonce, tag }.

const decipher = createDecipheriv('aes-192-ccm', key, nonce, {
  authTagLength: 16,
});
decipher.setAuthTag(tag);
decipher.setAAD(aad, {
  plaintextLength: ciphertext.length,
});
const receivedPlaintext = decipher.update(ciphertext, null, 'utf8');

try {
  decipher.final();
} catch (err) {
  throw new Error('Authentication failed!', { cause: err });
}

console.log(receivedPlaintext);
```

```cjs
const { Buffer } = require('node:buffer');
const {
  createCipheriv,
  createDecipheriv,
  randomBytes,
} = require('node:crypto');

const key = 'keykeykeykeykeykeykeykey';
const nonce = randomBytes(12);

const aad = Buffer.from('0123456789', 'hex');

const cipher = createCipheriv('aes-192-ccm', key, nonce, {
  authTagLength: 16,
});
const plaintext = 'Hello world';
cipher.setAAD(aad, {
  plaintextLength: Buffer.byteLength(plaintext),
});
const ciphertext = cipher.update(plaintext, 'utf8');
cipher.final();
const tag = cipher.getAuthTag();

// Now transmit { ciphertext, nonce, tag }.

const decipher = createDecipheriv('aes-192-ccm', key, nonce, {
  authTagLength: 16,
});
decipher.setAuthTag(tag);
decipher.setAAD(aad, {
  plaintextLength: ciphertext.length,
});
const receivedPlaintext = decipher.update(ciphertext, null, 'utf8');

try {
  decipher.final();
} catch (err) {
  throw new Error('Authentication failed!', { cause: err });
}

console.log(receivedPlaintext);
```

### FIPS mode

When using OpenSSL 3, Node.js supports FIPS 140-2 when used with an appropriate
OpenSSL 3 provider, such as the [FIPS provider from OpenSSL 3][] which can be
installed by following the instructions in [OpenSSL's FIPS README file][].

For FIPS support in Node.js you will need:

* A correctly installed OpenSSL 3 FIPS provider.
* An OpenSSL 3 [FIPS module configuration file][].
* An OpenSSL 3 configuration file that references the FIPS module
  configuration file.

Node.js will need to be configured with an OpenSSL configuration file that
points to the FIPS provider. An example configuration file looks like this:

```text
nodejs_conf = nodejs_init

.include /<absolute path>/fipsmodule.cnf

[nodejs_init]
providers = provider_sect

[provider_sect]
default = default_sect
# The fips section name should match the section name inside the
# included fipsmodule.cnf.
fips = fips_sect

[default_sect]
activate = 1
```

where `fipsmodule.cnf` is the FIPS module configuration file generated from the
FIPS provider installation step:

```bash
openssl fipsinstall
```

Set the `OPENSSL_CONF` environment variable to point to
your configuration file and `OPENSSL_MODULES` to the location of the FIPS
provider dynamic library. e.g.

```bash
export OPENSSL_CONF=/<path to configuration file>/nodejs.cnf
export OPENSSL_MODULES=/<path to openssl lib>/ossl-modules
```

FIPS mode can then be enabled in Node.js either by:

* Starting Node.js with `--enable-fips` or `--force-fips` command line flags.
* Programmatically calling `crypto.setFips(true)`.

Optionally FIPS mode can be enabled in Node.js via the OpenSSL configuration
file. e.g.

```text
nodejs_conf = nodejs_init

.include /<absolute path>/fipsmodule.cnf

[nodejs_init]
providers = provider_sect
alg_section = algorithm_sect

[provider_sect]
default = default_sect
# The fips section name should match the section name inside the
# included fipsmodule.cnf.
fips = fips_sect

[default_sect]
activate = 1

[algorithm_sect]
default_properties = fips=yes
```

## Crypto constants

The following constants exported by `crypto.constants` apply to various uses of
the `node:crypto`, `node:tls`, and `node:https` modules and are generally
specific to OpenSSL.

### OpenSSL options

See the [list of SSL OP Flags][] for details.

<table>
  <tr>
    <th>Constant</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><code>SSL_OP_ALL</code></td>
    <td>Applies multiple bug workarounds within OpenSSL. See
    <a href="https://www.openssl.org/docs/man3.0/man3/SSL_CTX_set_options.html">https://www.openssl.org/docs/man3.0/man3/SSL_CTX_set_options.html</a>
    for detail.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_ALLOW_NO_DHE_KEX</code></td>
    <td>Instructs OpenSSL to allow a non-[EC]DHE-based key exchange mode
    for TLS v1.3</td>
  </tr>
  <tr>
    <td><code>SSL_OP_ALLOW_UNSAFE_LEGACY_RENEGOTIATION</code></td>
    <td>Allows legacy insecure renegotiation between OpenSSL and unpatched
    clients or servers. See
    <a href="https://www.openssl.org/docs/man3.0/man3/SSL_CTX_set_options.html">https://www.openssl.org/docs/man3.0/man3/SSL_CTX_set_options.html</a>.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_CIPHER_SERVER_PREFERENCE</code></td>
    <td>Attempts to use the server's preferences instead of the client's when
    selecting a cipher. Behavior depends on protocol version. See
    <a href="https://www.openssl.org/docs/man3.0/man3/SSL_CTX_set_options.html">https://www.openssl.org/docs/man3.0/man3/SSL_CTX_set_options.html</a>.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_CISCO_ANYCONNECT</code></td>
    <td>Instructs OpenSSL to use Cisco's version identifier of DTLS_BAD_VER.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_COOKIE_EXCHANGE</code></td>
    <td>Instructs OpenSSL to turn on cookie exchange.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_CRYPTOPRO_TLSEXT_BUG</code></td>
    <td>Instructs OpenSSL to add server-hello extension from an early version
    of the cryptopro draft.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_DONT_INSERT_EMPTY_FRAGMENTS</code></td>
    <td>Instructs OpenSSL to disable a SSL 3.0/TLS 1.0 vulnerability
    workaround added in OpenSSL 0.9.6d.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_LEGACY_SERVER_CONNECT</code></td>
    <td>Allows initial connection to servers that do not support RI.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_COMPRESSION</code></td>
    <td>Instructs OpenSSL to disable support for SSL/TLS compression.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_ENCRYPT_THEN_MAC</code></td>
    <td>Instructs OpenSSL to disable encrypt-then-MAC.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_QUERY_MTU</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_RENEGOTIATION</code></td>
    <td>Instructs OpenSSL to disable renegotiation.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_SESSION_RESUMPTION_ON_RENEGOTIATION</code></td>
    <td>Instructs OpenSSL to always start a new session when performing
    renegotiation.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_SSLv2</code></td>
    <td>Instructs OpenSSL to turn off SSL v2</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_SSLv3</code></td>
    <td>Instructs OpenSSL to turn off SSL v3</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_TICKET</code></td>
    <td>Instructs OpenSSL to disable use of RFC4507bis tickets.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_TLSv1</code></td>
    <td>Instructs OpenSSL to turn off TLS v1</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_TLSv1_1</code></td>
    <td>Instructs OpenSSL to turn off TLS v1.1</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_TLSv1_2</code></td>
    <td>Instructs OpenSSL to turn off TLS v1.2</td>
  </tr>
  <tr>
    <td><code>SSL_OP_NO_TLSv1_3</code></td>
    <td>Instructs OpenSSL to turn off TLS v1.3</td>
  </tr>
  <tr>
    <td><code>SSL_OP_PRIORITIZE_CHACHA</code></td>
    <td>Instructs OpenSSL server to prioritize ChaCha20-Poly1305
    when the client does.
    This option has no effect if
    <code>SSL_OP_CIPHER_SERVER_PREFERENCE</code>
    is not enabled.</td>
  </tr>
  <tr>
    <td><code>SSL_OP_TLS_ROLLBACK_BUG</code></td>
    <td>Instructs OpenSSL to disable version rollback attack detection.</td>
  </tr>
</table>

### OpenSSL engine constants

<table>
  <tr>
    <th>Constant</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_RSA</code></td>
    <td>Limit engine usage to RSA</td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_DSA</code></td>
    <td>Limit engine usage to DSA</td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_DH</code></td>
    <td>Limit engine usage to DH</td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_RAND</code></td>
    <td>Limit engine usage to RAND</td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_EC</code></td>
    <td>Limit engine usage to EC</td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_CIPHERS</code></td>
    <td>Limit engine usage to CIPHERS</td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_DIGESTS</code></td>
    <td>Limit engine usage to DIGESTS</td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_PKEY_METHS</code></td>
    <td>Limit engine usage to PKEY_METHS</td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_PKEY_ASN1_METHS</code></td>
    <td>Limit engine usage to PKEY_ASN1_METHS</td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_ALL</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>ENGINE_METHOD_NONE</code></td>
    <td></td>
  </tr>
</table>

### Other OpenSSL constants

<table>
  <tr>
    <th>Constant</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><code>DH_CHECK_P_NOT_SAFE_PRIME</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>DH_CHECK_P_NOT_PRIME</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>DH_UNABLE_TO_CHECK_GENERATOR</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>DH_NOT_SUITABLE_GENERATOR</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>RSA_PKCS1_PADDING</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>RSA_SSLV23_PADDING</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>RSA_NO_PADDING</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>RSA_PKCS1_OAEP_PADDING</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>RSA_X931_PADDING</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>RSA_PKCS1_PSS_PADDING</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>RSA_PSS_SALTLEN_DIGEST</code></td>
    <td>Sets the salt length for <code>RSA_PKCS1_PSS_PADDING</code> to the
        digest size when signing or verifying.</td>
  </tr>
  <tr>
    <td><code>RSA_PSS_SALTLEN_MAX_SIGN</code></td>
    <td>Sets the salt length for <code>RSA_PKCS1_PSS_PADDING</code> to the
        maximum permissible value when signing data.</td>
  </tr>
  <tr>
    <td><code>RSA_PSS_SALTLEN_AUTO</code></td>
    <td>Causes the salt length for <code>RSA_PKCS1_PSS_PADDING</code> to be
        determined automatically when verifying a signature.</td>
  </tr>
  <tr>
    <td><code>POINT_CONVERSION_COMPRESSED</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>POINT_CONVERSION_UNCOMPRESSED</code></td>
    <td></td>
  </tr>
  <tr>
    <td><code>POINT_CONVERSION_HYBRID</code></td>
    <td></td>
  </tr>
</table>

### Node.js crypto constants

<table>
  <tr>
    <th>Constant</th>
    <th>Description</th>
  </tr>
  <tr>
    <td><code>defaultCoreCipherList</code></td>
    <td>Specifies the built-in default cipher list used by Node.js.</td>
  </tr>
  <tr>
    <td><code>defaultCipherList</code></td>
    <td>Specifies the active default cipher list used by the current Node.js
    process.</td>
  </tr>
</table>

[^openssl30]: Requires OpenSSL >= 3.0

[^openssl32]: Requires OpenSSL >= 3.2

[^openssl35]: Requires OpenSSL >= 3.5

[AEAD algorithms]: https://en.wikipedia.org/wiki/Authenticated_encryption
[CCM mode]: #ccm-mode
[CVE-2021-44532]: https://cve.mitre.org/cgi-bin/cvename.cgi?name=CVE-2021-44532
[Caveats]: #support-for-weak-or-compromised-algorithms
[Crypto constants]: #crypto-constants
[FIPS module configuration file]: https://www.openssl.org/docs/man3.0/man5/fips_config.html
[FIPS provider from OpenSSL 3]: https://www.openssl.org/docs/man3.0/man7/crypto.html#FIPS-provider
[HTML 5.2]: https://www.w3.org/TR/html52/changes.html#features-removed
[JWK]: https://tools.ietf.org/html/rfc7517
[Key usages]: webcrypto.md#cryptokeyusages
[NIST SP 800-131A]: https://nvlpubs.nist.gov/nistpubs/SpecialPublications/NIST.SP.800-131Ar2.pdf
[NIST SP 800-132]: https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-132.pdf
[NIST SP 800-38D]: https://nvlpubs.nist.gov/nistpubs/Legacy/SP/nistspecialpublication800-38d.pdf
[OpenSSL's FIPS README file]: https://github.com/openssl/openssl/blob/openssl-3.0/README-FIPS.md
[OpenSSL's SPKAC implementation]: https://www.openssl.org/docs/man3.0/man1/openssl-spkac.html
[RFC 1421]: https://www.rfc-editor.org/rfc/rfc1421.txt
[RFC 2409]: https://www.rfc-editor.org/rfc/rfc2409.txt
[RFC 2818]: https://www.rfc-editor.org/rfc/rfc2818.txt
[RFC 3526]: https://www.rfc-editor.org/rfc/rfc3526.txt
[RFC 3610]: https://www.rfc-editor.org/rfc/rfc3610.txt
[RFC 4055]: https://www.rfc-editor.org/rfc/rfc4055.txt
[RFC 4122]: https://www.rfc-editor.org/rfc/rfc4122.txt
[RFC 5208]: https://www.rfc-editor.org/rfc/rfc5208.txt
[RFC 5280]: https://www.rfc-editor.org/rfc/rfc5280.txt
[Web Crypto API documentation]: webcrypto.md
[`BN_is_prime_ex`]: https://www.openssl.org/docs/man1.1.1/man3/BN_is_prime_ex.html
[`Buffer`]: buffer.md
[`DH_generate_key()`]: https://www.openssl.org/docs/man3.0/man3/DH_generate_key.html
[`DiffieHellmanGroup`]: #class-diffiehellmangroup
[`KeyObject`]: #class-keyobject
[`Sign`]: #class-sign
[`String.prototype.normalize()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/normalize
[`UV_THREADPOOL_SIZE`]: cli.md#uv_threadpool_sizesize
[`Verify`]: #class-verify
[`cipher.final()`]: #cipherfinaloutputencoding
[`cipher.update()`]: #cipherupdatedata-inputencoding-outputencoding
[`crypto.createCipheriv()`]: #cryptocreatecipherivalgorithm-key-iv-options
[`crypto.createDecipheriv()`]: #cryptocreatedecipherivalgorithm-key-iv-options
[`crypto.createDiffieHellman()`]: #cryptocreatediffiehellmanprime-primeencoding-generator-generatorencoding
[`crypto.createECDH()`]: #cryptocreateecdhcurvename
[`crypto.createHash()`]: #cryptocreatehashalgorithm-options
[`crypto.createHmac()`]: #cryptocreatehmacalgorithm-key-options
[`crypto.createPrivateKey()`]: #cryptocreateprivatekeykey
[`crypto.createPublicKey()`]: #cryptocreatepublickeykey
[`crypto.createSecretKey()`]: #cryptocreatesecretkeykey-encoding
[`crypto.createSign()`]: #cryptocreatesignalgorithm-options
[`crypto.createVerify()`]: #cryptocreateverifyalgorithm-options
[`crypto.generateKey()`]: #cryptogeneratekeytype-options-callback
[`crypto.getCurves()`]: #cryptogetcurves
[`crypto.getDiffieHellman()`]: #cryptogetdiffiehellmangroupname
[`crypto.getHashes()`]: #cryptogethashes
[`crypto.privateDecrypt()`]: #cryptoprivatedecryptprivatekey-buffer
[`crypto.privateEncrypt()`]: #cryptoprivateencryptprivatekey-buffer
[`crypto.publicDecrypt()`]: #cryptopublicdecryptkey-buffer
[`crypto.publicEncrypt()`]: #cryptopublicencryptkey-buffer
[`crypto.randomBytes()`]: #cryptorandombytessize-callback
[`crypto.randomFill()`]: #cryptorandomfillbuffer-offset-size-callback
[`crypto.webcrypto.getRandomValues()`]: webcrypto.md#cryptogetrandomvaluestypedarray
[`crypto.webcrypto.subtle`]: webcrypto.md#class-subtlecrypto
[`decipher.final()`]: #decipherfinaloutputencoding
[`decipher.update()`]: #decipherupdatedata-inputencoding-outputencoding
[`diffieHellman.generateKeys()`]: #diffiehellmangeneratekeysencoding
[`diffieHellman.setPublicKey()`]: #diffiehellmansetpublickeypublickey-encoding
[`ecdh.generateKeys()`]: #ecdhgeneratekeysencoding-format
[`ecdh.setPrivateKey()`]: #ecdhsetprivatekeyprivatekey-encoding
[`hash.digest()`]: #hashdigestencoding
[`hash.update()`]: #hashupdatedata-inputencoding
[`hmac.digest()`]: #hmacdigestencoding
[`hmac.update()`]: #hmacupdatedata-inputencoding
[`import()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import
[`keyObject.export()`]: #keyobjectexportoptions
[`postMessage()`]: worker_threads.md#portpostmessagevalue-transferlist
[`sign.sign()`]: #signsignprivatekey-outputencoding
[`sign.update()`]: #signupdatedata-inputencoding
[`stream.Writable` options]: stream.md#new-streamwritableoptions
[`stream.transform` options]: stream.md#new-streamtransformoptions
[`util.promisify()`]: util.md#utilpromisifyoriginal
[`verify.update()`]: #verifyupdatedata-inputencoding
[`verify.verify()`]: #verifyverifyobject-signature-signatureencoding
[`x509.fingerprint256`]: #x509fingerprint256
[`x509.verify(publicKey)`]: #x509verifypublickey
[argon2]: https://www.rfc-editor.org/rfc/rfc9106.html
[asymmetric key types]: #asymmetric-key-types
[caveats when using strings as inputs to cryptographic APIs]: #using-strings-as-inputs-to-cryptographic-apis
[certificate object]: tls.md#certificate-object
[encoding]: buffer.md#buffers-and-character-encodings
[initialization vector]: https://en.wikipedia.org/wiki/Initialization_vector
[legacy provider]: cli.md#--openssl-legacy-provider
[list of SSL OP Flags]: https://wiki.openssl.org/index.php/List_of_SSL_OP_Flags#Table_of_Options
[modulo bias]: https://en.wikipedia.org/wiki/Fisher%E2%80%93Yates_shuffle#Modulo_bias
[safe integers]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/isSafeInteger
[scrypt]: https://en.wikipedia.org/wiki/Scrypt
[stream]: stream.md
[stream-writable-write]: stream.md#writablewritechunk-encoding-callback
