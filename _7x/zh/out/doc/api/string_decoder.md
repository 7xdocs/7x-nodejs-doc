# String decoder

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/string_decoder.js -->

`node:string_decoder` 模块提供了一个 API，用于将以编码的多字节 UTF-8 和 UTF-16 字符方式存储的 `Buffer` 对象解码为字符串。可以通过以下方式访问：

```mjs
import { StringDecoder } from 'node:string_decoder';
```

```cjs
const { StringDecoder } = require('node:string_decoder');
```

以下示例展示了 `StringDecoder` 类的基本用法。

```mjs
import { StringDecoder } from 'node:string_decoder';
import { Buffer } from 'node:buffer';
const decoder = new StringDecoder('utf8');

const cent = Buffer.from([0xC2, 0xA2]);
console.log(decoder.write(cent)); // Prints: Â˘

const euro = Buffer.from([0xE2, 0x82, 0xAC]);
console.log(decoder.write(euro)); // Prints: â‚¬
```

```cjs
const { StringDecoder } = require('node:string_decoder');
const decoder = new StringDecoder('utf8');

const cent = Buffer.from([0xC2, 0xA2]);
console.log(decoder.write(cent)); // Prints: Â˘

const euro = Buffer.from([0xE2, 0x82, 0xAC]);
console.log(decoder.write(euro)); // Prints: â‚¬
```

当向 `StringDecoder` 实例写入 `Buffer` 实例时，会使用一个内部缓冲区来确保解码后的字符串不包含任何不完整的多字节字符。这些字符会保留在缓冲区中，直到下一次调用 `stringDecoder.write()` 或调用 `stringDecoder.end()`。

在下面的示例中，欧洲欧元符号 (`€`) 的三个 UTF-8 编码字节分三次单独的写入操作完成：

```mjs
import { StringDecoder } from 'node:string_decoder';
import { Buffer } from 'node:buffer';
const decoder = new StringDecoder('utf8');

decoder.write(Buffer.from([0xE2]));
decoder.write(Buffer.from([0x82]));
console.log(decoder.end(Buffer.from([0xAC]))); // Prints: €
```

```cjs
const { StringDecoder } = require('node:string_decoder');
const decoder = new StringDecoder('utf8');

decoder.write(Buffer.from([0xE2]));
decoder.write(Buffer.from([0x82]));
console.log(decoder.end(Buffer.from([0xAC]))); // Prints: €
```

## 类：`StringDecoder`

### `new StringDecoder([encoding])`

<!-- YAML
added: v0.1.99
-->

* `encoding` {string} `StringDecoder` 将要使用的字符[编码][]。**默认值:** `'utf8'`。

创建一个新的 `StringDecoder` 实例。

### `stringDecoder.end([buffer])`

<!-- YAML
added: v0.9.3
-->

* `buffer` {string|Buffer|TypedArray|DataView} 要解码的字节。
* 返回: {string}

返回内部缓冲区中存储的任何剩余输入作为字符串。表示不完整的 UTF-8 和 UTF-16 字符的字节将被替换为适用于字符编码的替换字符。

如果提供了 `buffer` 参数，则在返回剩余输入之前会执行一次最终的 `stringDecoder.write()` 调用。
在调用 `end()` 之后，`stringDecoder` 对象可以被重新用于新的输入。

### `stringDecoder.write(buffer)`

<!-- YAML
added: v0.1.99
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/9618
    description: Each invalid character is now replaced by a single replacement
                 character instead of one for each individual byte.
-->

* `buffer` {string|Buffer|TypedArray|DataView} 要解码的字节。
* 返回: {string}

返回一个解码后的字符串，确保位于 `Buffer`、`TypedArray` 或 `DataView` 末尾的任何不完整多字节字符从返回的字符串中省略，并存储在内部缓冲区中，用于下一次调用 `stringDecoder.write()` 或 `stringDecoder.end()`。

[encoding]: buffer.md#buffers-and-character-encodings
