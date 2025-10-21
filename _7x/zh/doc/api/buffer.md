# Buffer

<!--introduced_in=v0.1.90-->

> Stability: 2 - Stable

<!-- source_link=lib/buffer.js -->

`Buffer` 对象用于表示固定长度的字节序列。许多 Node.js API 都支持 `Buffer`。

`Buffer` 类是 JavaScript 的 {Uint8Array} 类的子类，并使用覆盖更多用例的方法对其进行了扩展。Node.js API 在支持 `Buffer` 的地方也接受普通的 {Uint8Array}。

虽然 `Buffer` 类在全局作用域内可用，但仍然建议通过 import 或 require 语句显式地引用它。

```mjs
import { Buffer } from 'node:buffer';

// 创建一个长度为 10、用零填充的 Buffer。
const buf1 = Buffer.alloc(10);

// 创建一个长度为 10、
// 且所有字节的值都为 `1` 的 Buffer。
const buf2 = Buffer.alloc(10, 1);

// 创建一个长度为 10 的未初始化的 buffer。
// 这比调用 Buffer.alloc() 更快，但返回的
// Buffer 实例可能包含需要被
// 使用 fill()、write() 或其他填充 Buffer 内容的函数
// 覆盖的旧数据。
const buf3 = Buffer.allocUnsafe(10);

// 创建一个包含字节 [1, 2, 3] 的 Buffer。
const buf4 = Buffer.from([1, 2, 3]);

// 创建一个包含字节 [1, 1, 1, 1] 的 Buffer——所有条目
// 都使用 `(value & 255)` 截断以适合 0–255 的范围。
const buf5 = Buffer.from([257, 257.5, -255, '1']);

// 创建一个包含字符串 'tést' 的 UTF-8 编码字节的 Buffer：
// [0x74, 0xc3, 0xa9, 0x73, 0x74]（十六进制表示法）
// [116, 195, 169, 115, 116]（十进制表示法）
const buf6 = Buffer.from('tést');

// 创建一个包含 Latin-1 字节 [0x74, 0xe9, 0x73, 0x74] 的 Buffer。
const buf7 = Buffer.from('tést', 'latin1');
```

```cjs
const { Buffer } = require('node:buffer');

// 创建一个长度为 10、用零填充的 Buffer。
const buf1 = Buffer.alloc(10);

// 创建一个长度为 10、
// 且所有字节的值都为 `1` 的 Buffer。
const buf2 = Buffer.alloc(10, 1);

// 创建一个长度为 10 的未初始化的 buffer。
// 这比调用 Buffer.alloc() 更快，但返回的
// Buffer 实例可能包含需要被
// 使用 fill()、write() 或其他填充 Buffer 内容的函数
// 覆盖的旧数据。
const buf3 = Buffer.allocUnsafe(10);

// 创建一个包含字节 [1, 2, 3] 的 Buffer。
const buf4 = Buffer.from([1, 2, 3]);

// 创建一个包含字节 [1, 1, 1, 1] 的 Buffer——所有条目
// 都使用 `(value & 255)` 截断以适合 0–255 的范围。
const buf5 = Buffer.from([257, 257.5, -255, '1']);

// 创建一个包含字符串 'tést' 的 UTF-8 编码字节的 Buffer：
// [0x74, 0xc3, 0xa9, 0x73, 0x74]（十六进制表示法）
// [116, 195, 169, 115, 116]（十进制表示法）
const buf6 = Buffer.from('tést');

// 创建一个包含 Latin-1 字节 [0x74, 0xe9, 0x73, 0x74] 的 Buffer。
const buf7 = Buffer.from('tést', 'latin1');
```

## Buffer 和字符编码

<!-- YAML
changes:
  - version:
      - v15.7.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/36952
    description: Introduced `base64url` encoding.
  - version: v6.4.0
    pr-url: https://github.com/nodejs/node/pull/7111
    description: Introduced `latin1` as an alias for `binary`.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/2859
    description: Removed the deprecated `raw` and `raws` encodings.
-->

在 `Buffer` 和字符串之间进行转换时，可以指定字符编码。如果未指定字符编码，则默认使用 UTF-8。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('hello world', 'utf8');

console.log(buf.toString('hex'));
// 打印: 68656c6c6f20776f726c64
console.log(buf.toString('base64'));
// 打印: aGVsbG8gd29ybGQ=

console.log(Buffer.from('fhqwhgads', 'utf8'));
// 打印: <Buffer 66 68 71 77 68 67 61 64 73>
console.log(Buffer.from('fhqwhgads', 'utf16le'));
// 打印: <Buffer 66 00 68 00 71 00 77 00 68 00 67 00 61 00 64 00 73 00>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('hello world', 'utf8');

console.log(buf.toString('hex'));
// 打印: 68656c6c6f20776f726c64
console.log(buf.toString('base64'));
// 打印: aGVsbG8gd29ybGQ=

console.log(Buffer.from('fhqwhgads', 'utf8'));
// 打印: <Buffer 66 68 71 77 68 67 61 64 73>
console.log(Buffer.from('fhqwhgads', 'utf16le'));
// 打印: <Buffer 66 00 68 00 71 00 77 00 68 00 67 00 61 00 64 00 73 00>
```

Node.js buffer 接受它们收到的编码字符串的所有大小写变体。例如，UTF-8 可以指定为 `'utf8'`、`'UTF8'` 或 `'uTf8'`。

Node.js 当前支持的字符编码如下：

* `'utf8'`（别名：`'utf-8'`）：多字节编码的 Unicode 字符。许多网页和其他文档格式使用 [UTF-8][]。这是默认的字符编码。当将 `Buffer` 解码为不完全是有效 UTF-8 数据的字符串时，Unicode 替换字符 `U+FFFD` � 将用于表示这些错误。

* `'utf16le'`（别名：`'utf-16le'`）：多字节编码的 Unicode 字符。与 `'utf8'` 不同，字符串中的每个字符将使用 2 或 4 个字节进行编码。Node.js 仅支持 [UTF-16][] 的[小端序][endianness]变体。

* `'latin1'`：Latin-1 代表 [ISO-8859-1][]。此字符编码仅支持从 `U+0000` 到 `U+00FF` 的 Unicode 字符。每个字符使用单个字节进行编码。不适合该范围的字符将被截断并映射到该范围内的字符。

使用上述之一将 `Buffer` 转换为字符串称为解码，将字符串转换为 `Buffer` 称为编码。

Node.js 还支持以下二进制到文本的编码。对于二进制到文本的编码，命名约定是相反的：将 `Buffer` 转换为字符串通常称为编码，将字符串转换为 `Buffer` 称为解码。

* `'base64'`：[Base64][] 编码。从字符串创建 `Buffer` 时，此编码也将正确接受 [RFC 4648，第 5 节][]中指定的“URL 和文件名安全字母表”。base64 编码字符串中包含的空格、制表符和换行符等空白字符将被忽略。

* `'base64url'`：[base64url][] 编码，如 [RFC 4648，第 5 节][]中所指定。从字符串创建 `Buffer` 时，此编码也将正确接受常规的 base64 编码字符串。将 `Buffer` 编码为字符串时，此编码将省略填充。

* `'hex'`：将每个字节编码为两个十六进制字符。解码不完全由偶数个十六进制字符组成的字符串时，可能会发生数据截断。请参阅下面的示例。

还支持以下传统字符编码：

* `'ascii'`：仅用于 7 位 [ASCII][] 数据。将字符串编码为 `Buffer` 时，这等效于使用 `'latin1'`。将 `Buffer` 解码为字符串时，使用此编码将在解码为 `'latin1'` 之前额外取消设置每个字节的最高位。
  通常，没有理由使用此编码，因为 `'utf8'`（或者，如果已知数据始终为纯 ASCII，则使用 `'latin1'`）在编码或解码纯 ASCII 文本时将是更好的选择。仅为传统兼容性提供。

* `'binary'`：`'latin1'` 的别名。
  此编码的名称可能非常具有误导性，因为此处列出的所有编码都是在字符串和二进制数据之间进行转换。对于在字符串和 `Buffer` 之间进行转换，通常 `'utf8'` 是正确的选择。

* `'ucs2'`、`'ucs-2'`：`'utf16le'` 的别名。UCS-2 过去指的是不支持码点大于 U+FFFF 的字符的 UTF-16 变体。在 Node.js 中，始终支持这些码点。

```mjs
import { Buffer } from 'node:buffer';

Buffer.from('1ag123', 'hex');
// 打印 <Buffer 1a>，当遇到第一个非十六进制值
// ('g') 时数据被截断。

Buffer.from('1a7', 'hex');
// 打印 <Buffer 1a>，当数据以单个数字 ('7') 结尾时数据被截断。

Buffer.from('1634', 'hex');
// 打印 <Buffer 16 34>，所有数据都被表示。
```

```cjs
const { Buffer } = require('node:buffer');

Buffer.from('1ag123', 'hex');
// 打印 <Buffer 1a>，当遇到第一个非十六进制值
// ('g') 时数据被截断。

Buffer.from('1a7', 'hex');
// 打印 <Buffer 1a>，当数据以单个数字 ('7') 结尾时数据被截断。

Buffer.from('1634', 'hex');
// 打印 <Buffer 16 34>，所有数据都被表示。
```

现代 Web 浏览器遵循 [WHATWG 编码标准][]，该标准将 `'latin1'` 和 `'ISO-8859-1'` 都别名为 `'win-1252'`。这意味着在执行诸如 `http.get()` 之类的操作时，如果返回的字符集是 WHATWG 规范中列出的字符集之一，则服务器实际上可能返回了 `'win-1252'` 编码的数据，并且使用 `'latin1'` 编码可能会错误地解码字符。

## Buffer 和 TypedArray

<!-- YAML
changes:
  - version: v3.0.0
    pr-url: https://github.com/nodejs/node/pull/2002
    description: The `Buffer` class now inherits from `Uint8Array`.
-->

`Buffer` 实例也是 JavaScript {Uint8Array} 和 {TypedArray} 实例。所有 {TypedArray} 方法在 `Buffer` 上都可用。但是，`Buffer` API 和 {TypedArray} API 之间存在细微的不兼容性。

特别是：

* 虽然 [`TypedArray.prototype.slice()`][] 创建了 `TypedArray` 的一部分的副本，但 [`Buffer.prototype.slice()`][`buf.slice()`] 在不复制的情况下在现有 `Buffer` 上创建视图。此行为可能令人惊讶，并且仅出于传统兼容性而存在。 [`TypedArray.prototype.subarray()`][] 可用于在 `Buffer` 和其他 `TypedArray` 上实现 [`Buffer.prototype.slice()`][`buf.slice()`] 的行为，并且应优先使用。
* [`buf.toString()`][] 与其 `TypedArray` 等效项不兼容。
* 许多方法，例如 [`buf.indexOf()`][]，支持附加参数。

有两种方法可以从 `Buffer` 创建新的 {TypedArray} 实例：

* 将 `Buffer` 传递给 {TypedArray} 构造函数将复制 `Buffer` 的内容，解释为整数数组，而不是目标类型的字节序列。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([1, 2, 3, 4]);
const uint32array = new Uint32Array(buf);

console.log(uint32array);

// 打印: Uint32Array(4) [ 1, 2, 3, 4 ]
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([1, 2, 3, 4]);
const uint32array = new Uint32Array(buf);

console.log(uint32array);

// 打印: Uint32Array(4) [ 1, 2, 3, 4 ]
```

* 传递 `Buffer` 的底层 {ArrayBuffer} 将创建一个与 `Buffer` 共享其内存的 {TypedArray}。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('hello', 'utf16le');
const uint16array = new Uint16Array(
  buf.buffer,
  buf.byteOffset,
  buf.length / Uint16Array.BYTES_PER_ELEMENT);

console.log(uint16array);

// 打印: Uint16Array(5) [ 104, 101, 108, 108, 111 ]
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('hello', 'utf16le');
const uint16array = new Uint16Array(
  buf.buffer,
  buf.byteOffset,
  buf.length / Uint16Array.BYTES_PER_ELEMENT);

console.log(uint16array);

// 打印: Uint16Array(5) [ 104, 101, 108, 108, 111 ]
```

可以通过以相同方式使用 `TypedArray` 对象的 `.buffer` 属性来创建一个新的 `Buffer`，该 `Buffer` 与 {TypedArray} 实例共享相同的已分配内存。在这种情况下，[`Buffer.from()`][`Buffer.from(arrayBuf)`] 的行为类似于 `new Uint8Array()`。

```mjs
import { Buffer } from 'node:buffer';

const arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// 复制 `arr` 的内容。
const buf1 = Buffer.from(arr);

// 与 `arr` 共享内存。
const buf2 = Buffer.from(arr.buffer);

console.log(buf1);
// 打印: <Buffer 88 a0>
console.log(buf2);
// 打印: <Buffer 88 13 a0 0f>

arr[1] = 6000;

console.log(buf1);
// 打印: <Buffer 88 a0>
console.log(buf2);
// 打印: <Buffer 88 13 70 17>
```

```cjs
const { Buffer } = require('node:buffer');

const arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// 复制 `arr` 的内容。
const buf1 = Buffer.from(arr);

// 与 `arr` 共享内存。
const buf2 = Buffer.from(arr.buffer);

console.log(buf1);
// 打印: <Buffer 88 a0>
console.log(buf2);
// 打印: <Buffer 88 13 a0 0f>

arr[1] = 6000;

console.log(buf1);
// 打印: <Buffer 88 a0>
console.log(buf2);
// 打印: <Buffer 88 13 70 17>
```

当使用 {TypedArray} 的 `.buffer` 创建 `Buffer` 时，可以通过传入 `byteOffset` 和 `length` 参数来仅使用底层 {ArrayBuffer} 的一部分。

```mjs
import { Buffer } from 'node:buffer';

const arr = new Uint16Array(20);
const buf = Buffer.from(arr.buffer, 0, 16);

console.log(buf.length);
// 打印: 16
```

```cjs
const { Buffer } = require('node:buffer');

const arr = new Uint16Array(20);
const buf = Buffer.from(arr.buffer, 0, 16);

console.log(buf.length);
// 打印: 16
```

`Buffer.from()` 和 [`TypedArray.from()`][] 具有不同的签名和实现。具体来说，{TypedArray} 变体接受第二个参数，该参数是在类型化数组的每个元素上调用的映射函数：

* [`TypedArray.from(source[, mapFn[, thisArg]])`][`TypedArray.from()`]

但是，`Buffer.from()` 方法不支持使用映射函数：

* [`Buffer.from(array)`][]
* [`Buffer.from(buffer)`][]
* [`Buffer.from(arrayBuffer[, byteOffset[, length]])`][`Buffer.from(arrayBuf)`]
* [`Buffer.from(string[, encoding])`][`Buffer.from(string)`]

## Buffer 和迭代

可以使用 `for..of` 语法迭代 `Buffer` 实例：

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([1, 2, 3]);

for (const b of buf) {
  console.log(b);
}
// 打印:
//   1
//   2
//   3
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([1, 2, 3]);

for (const b of buf) {
  console.log(b);
}
// 打印:
//   1
//   2
//   3
```

此外，[`buf.values()`][]、[`buf.keys()`][] 和 [`buf.entries()`][] 方法可用于创建迭代器。

## 类：`Blob`

<!-- YAML
added:
  - v15.7.0
  - v14.18.0
changes:
  - version:
    - v18.0.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/41270
    description: No longer experimental.
-->

{Blob} 封装了不可变的原始数据，可以安全地在多个工作线程之间共享。

### `new buffer.Blob([sources[, options]])`

<!-- YAML
added:
  - v15.7.0
  - v14.18.0
changes:
  - version: v16.7.0
    pr-url: https://github.com/nodejs/node/pull/39708
    description: Added the standard `endings` option to replace line-endings,
                 and removed the non-standard `encoding` option.
-->

* `sources` {string\[]|ArrayBuffer\[]|TypedArray\[]|DataView\[]|Blob\[]} 一个字符串、{ArrayBuffer}、{TypedArray}、{DataView} 或 {Blob} 对象的数组，或此类对象的任何混合，将存储在 `Blob` 中。
* `options` {Object}
  * `endings` {string} `'transparent'` 或 `'native'` 之一。当设置为 `'native'` 时，字符串源部分中的行结尾将转换为 `require('node:os').EOL` 指定的平台本机行结尾。
  * `type` {string} Blob 内容类型。目的是让 `type` 传达数据的 MIME 媒体类型，但不执行类型格式的验证。

创建一个新的 `Blob` 对象，其中包含给定源的连接。

{ArrayBuffer}、{TypedArray}、{DataView} 和 {Buffer} 源被复制到 'Blob' 中，因此可以在创建 'Blob' 后安全地修改它们。

字符串源被编码为 UTF-8 字节序列并复制到 Blob 中。每个字符串部分中不匹配的代理对将被 Unicode U+FFFD 替换字符替换。

### `blob.arrayBuffer()`

<!-- YAML
added:
  - v15.7.0
  - v14.18.0
-->

* 返回：{Promise}

返回一个 promise，该 promise 使用包含 `Blob` 数据副本的 {ArrayBuffer} 来履行。

#### `blob.bytes()`

<!-- YAML
added:
  - v22.3.0
  - v20.16.0
-->

`blob.bytes()` 方法将 `Blob` 对象的字节作为 `Promise<Uint8Array>` 返回。

```js
const blob = new Blob(['hello']);
blob.bytes().then((bytes) => {
  console.log(bytes); // 输出: Uint8Array(5) [ 104, 101, 108, 108, 111 ]
});
```

### `blob.size`

<!-- YAML
added:
  - v15.7.0
  - v14.18.0
-->

`Blob` 的总大小（以字节为单位）。

### `blob.slice([start[, end[, type]]])`

<!-- YAML
added:
  - v15.7.0
  - v14.18.0
-->

* `start` {number} 起始索引。
* `end` {number} 结束索引。
* `type` {string} 新 `Blob` 的内容类型

创建并返回一个新的 `Blob`，其中包含此 `Blob` 对象数据的子集。原始 `Blob` 不会被更改。

### `blob.stream()`

<!-- YAML
added: v16.7.0
-->

* 返回：{ReadableStream}

返回一个新的 `ReadableStream`，允许读取 `Blob` 的内容。

### `blob.text()`

<!-- YAML
added:
  - v15.7.0
  - v14.18.0
-->

* 返回：{Promise}

返回一个 promise，该 promise 使用 `Blob` 的内容作为 UTF-8 字符串解码来履行。

### `blob.type`

<!-- YAML
added:
  - v15.7.0
  - v14.18.0
-->

* 类型：{string}

`Blob` 的内容类型。

### `Blob` 对象和 `MessageChannel`

一旦创建了 {Blob} 对象，它就可以通过 `MessagePort` 发送到多个目的地，而无需传输或立即复制数据。仅当调用 `arrayBuffer()` 或 `text()` 方法时，才会复制 `Blob` 包含的数据。

```mjs
import { Blob } from 'node:buffer';
import { setTimeout as delay } from 'node:timers/promises';

const blob = new Blob(['hello there']);

const mc1 = new MessageChannel();
const mc2 = new MessageChannel();

mc1.port1.onmessage = async ({ data }) => {
  console.log(await data.arrayBuffer());
  mc1.port1.close();
};

mc2.port1.onmessage = async ({ data }) => {
  await delay(1000);
  console.log(await data.arrayBuffer());
  mc2.port1.close();
};

mc1.port2.postMessage(blob);
mc2.port2.postMessage(blob);

// 发布后 Blob 仍然可用。
blob.text().then(console.log);
```

```cjs
const { Blob } = require('node:buffer');
const { setTimeout: delay } = require('node:timers/promises');

const blob = new Blob(['hello there']);

const mc1 = new MessageChannel();
const mc2 = new MessageChannel();

mc1.port1.onmessage = async ({ data }) => {
  console.log(await data.arrayBuffer());
  mc1.port1.close();
};

mc2.port1.onmessage = async ({ data }) => {
  await delay(1000);
  console.log(await data.arrayBuffer());
  mc2.port1.close();
};

mc1.port2.postMessage(blob);
mc2.port2.postMessage(blob);

// 发布后 Blob 仍然可用。
blob.text().then(console.log);
```

## 类：`Buffer`

`Buffer` 类是用于直接处理二进制数据的全局类型。它可以通过多种方式构造。

### 静态方法：`Buffer.alloc(size[, fill[, encoding]])`

<!-- YAML
added: v5.10.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/45796
    description: Throw ERR_INVALID_ARG_TYPE or ERR_OUT_OF_RANGE instead of
                 ERR_INVALID_ARG_VALUE for invalid input arguments.
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/34682
    description: Throw ERR_INVALID_ARG_VALUE instead of ERR_INVALID_OPT_VALUE
                 for invalid input arguments.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18129
    description: Attempting to fill a non-zero length buffer with a zero length
                 buffer triggers a thrown exception.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/17427
    description: Specifying an invalid string for `fill` triggers a thrown
                 exception.
  - version: v8.9.3
    pr-url: https://github.com/nodejs/node/pull/17428
    description: Specifying an invalid string for `fill` now results in a
                 zero-filled buffer.
-->

* `size` {integer} 新 `Buffer` 的所需长度。
* `fill` {string|Buffer|Uint8Array|integer} 用于预填充新 `Buffer` 的值。
  **默认值：** `0`。
* `encoding` {string} 如果 `fill` 是字符串，则这是它的编码。
  **默认值：** `'utf8'`。
* 返回：{Buffer}

分配一个大小为 `size` 字节的新 `Buffer`。如果 `fill` 是 `undefined`，则 `Buffer` 将被零填充。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.alloc(5);

console.log(buf);
// 打印: <Buffer 00 00 00 00 00>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.alloc(5);

console.log(buf);
// 打印: <Buffer 00 00 00 00 00>
```

如果 `size` 大于 [`buffer.constants.MAX_LENGTH`][] 或小于 0，则会抛出 [`ERR_OUT_OF_RANGE`][]。

如果指定了 `fill`，则分配的 `Buffer` 将通过调用 [`buf.fill(fill)`][`buf.fill()`] 来初始化。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.alloc(5, 'a');

console.log(buf);
// 打印: <Buffer 61 61 61 61 61>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.alloc(5, 'a');

console.log(buf);
// 打印: <Buffer 61 61 61 61 61>
```

如果同时指定了 `fill` 和 `encoding`，则分配的 `Buffer` 将通过调用 [`buf.fill(fill, encoding)`][`buf.fill()`] 来初始化。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.alloc(11, 'aGVsbG8gd29ybGQ=', 'base64');

console.log(buf);
// 打印: <Buffer 68 65 6c 6c 6f 20 77 6f 72 6c 64>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.alloc(11, 'aGVsbG8gd29ybGQ=', 'base64');

console.log(buf);
// 打印: <Buffer 68 65 6c 6c 6f 20 77 6f 72 6c 64>
```

调用 [`Buffer.alloc()`][] 可能比替代方法 [`Buffer.allocUnsafe()`][] 慢得多，但确保新创建的 `Buffer` 实例内容永远不会包含来自先前分配的敏感数据，包括可能尚未为 `Buffer` 分配的数据。

如果 `size` 不是数字，则会抛出 `TypeError`。

### 静态方法：`Buffer.allocUnsafe(size)`

<!-- YAML
added: v5.10.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/45796
    description: Throw ERR_INVALID_ARG_TYPE or ERR_OUT_OF_RANGE instead of
                 ERR_INVALID_ARG_VALUE for invalid input arguments.
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/34682
    description: Throw ERR_INVALID_ARG_VALUE instead of ERR_INVALID_OPT_VALUE
                 for invalid input arguments.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7079
    description: Passing a negative `size` will now throw an error.
-->

* `size` {integer} 新 `Buffer` 的所需长度。
* 返回：{Buffer}

分配一个大小为 `size` 字节的新 `Buffer`。如果 `size` 大于 [`buffer.constants.MAX_LENGTH`][] 或小于 0，则会抛出 [`ERR_OUT_OF_RANGE`][]。

以这种方式创建的 `Buffer` 实例的底层内存*未初始化*。新创建的 `Buffer` 的内容未知且*可能包含敏感数据*。使用 [`Buffer.alloc()`][] 将 `Buffer` 实例初始化为零。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(10);

console.log(buf);
// 打印（内容可能不同）: <Buffer a0 8b 28 3f 01 00 00 00 50 32>

buf.fill(0);

console.log(buf);
// 打印: <Buffer 00 00 00 00 00 00 00 00 00 00>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(10);

console.log(buf);
// 打印（内容可能不同）: <Buffer a0 8b 28 3f 01 00 00 00 50 32>

buf.fill(0);

console.log(buf);
// 打印: <Buffer 00 00 00 00 00 00 00 00 00 00>
```

如果 `size` 不是数字，则会抛出 `TypeError`。

`Buffer` 模块预分配了一个大小为 [`Buffer.poolSize`][] 的内部 `Buffer` 实例，该实例用作池，用于快速分配使用 [`Buffer.allocUnsafe()`][]、[`Buffer.from(array)`][]、[`Buffer.from(string)`][] 和 [`Buffer.concat()`][] 创建的新 `Buffer` 实例，仅当 `size` 小于 `Buffer.poolSize >>> 1`（[`Buffer.poolSize`][] 除以二的下取整）时。

使用此预分配的内部内存池是调用 `Buffer.alloc(size, fill)` 与 `Buffer.allocUnsafe(size).fill(fill)` 之间的关键区别。具体来说，`Buffer.alloc(size, fill)` 将*永远不会*使用内部 `Buffer` 池，而 `Buffer.allocUnsafe(size).fill(fill)` 将*会*使用内部 `Buffer` 池（如果 `size` 小于或等于 [`Buffer.poolSize`][] 的一半）。这种差异很微妙，但当应用程序需要 [`Buffer.allocUnsafe()`][] 提供的额外性能时，可能很重要。

### 静态方法：`Buffer.allocUnsafeSlow(size)`

<!-- YAML
added: v5.12.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/45796
    description: Throw ERR_INVALID_ARG_TYPE or ERR_OUT_OF_RANGE instead of
                 ERR_INVALID_ARG_VALUE for invalid input arguments.
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/34682
    description: Throw ERR_INVALID_ARG_VALUE instead of ERR_INVALID_OPT_VALUE
                 for invalid input arguments.
-->

* `size` {integer} 新 `Buffer` 的所需长度。
* 返回：{Buffer}

分配一个大小为 `size` 字节的新 `Buffer`。如果 `size` 大于 [`buffer.constants.MAX_LENGTH`][] 或小于 0，则会抛出 [`ERR_OUT_OF_RANGE`][]。如果 `size` 为 0，则创建零长度的 `Buffer`。

以这种方式创建的 `Buffer` 实例的底层内存*未初始化*。新创建的 `Buffer` 的内容未知且*可能包含敏感数据*。使用 [`buf.fill(0)`][`buf.fill()`] 将此类 `Buffer` 实例初始化为零。

当使用 [`Buffer.allocUnsafe()`][] 分配新的 `Buffer` 实例时，小于 `Buffer.poolSize >>> 1`（使用默认 poolSize 时为 4KiB）的分配是从单个预分配的 `Buffer` 中切分的。这允许应用程序避免创建许多单独分配的 `Buffer` 实例的垃圾收集开销。这种方法通过消除跟踪和清理尽可能多的单个 `ArrayBuffer` 对象的需要，提高了性能和内存使用率。

但是，在开发人员可能需要从池中保留一小块内存一段不确定时间的情况下，使用 `Buffer.allocUnsafeSlow()` 创建一个非池化的 `Buffer` 实例，然后复制出相关位可能是合适的。

```mjs
import { Buffer } from 'node:buffer';

// 需要保留几小块内存。
const store = [];

socket.on('readable', () => {
  let data;
  while (null !== (data = readable.read())) {
    // 为保留的数据分配。
    const sb = Buffer.allocUnsafeSlow(10);

    // 将数据复制到新分配中。
    data.copy(sb, 0, 0, 10);

    store.push(sb);
  }
});
```

```cjs
const { Buffer } = require('node:buffer');

// 需要保留几小块内存。
const store = [];

socket.on('readable', () => {
  let data;
  while (null !== (data = readable.read())) {
    // 为保留的数据分配。
    const sb = Buffer.allocUnsafeSlow(10);

    // 将数据复制到新分配中。
    data.copy(sb, 0, 0, 10);

    store.push(sb);
  }
});
```

如果 `size` 不是数字，则会抛出 `TypeError`。

### 静态方法：`Buffer.byteLength(string[, encoding])`

<!-- YAML
added: v0.1.90
changes:
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/8946
    description: Passing invalid input will now throw an error.
  - version: v5.10.0
    pr-url: https://github.com/nodejs/node/pull/5255
    description: The `string` parameter can now be any `TypedArray`, `DataView`
                 or `ArrayBuffer`.
-->

* `string` {string|Buffer|TypedArray|DataView|ArrayBuffer|SharedArrayBuffer} 要计算长度的值。
* `encoding` {string} 如果 `string` 是字符串，则这是它的编码。
  **默认值：** `'utf8'`。
* 返回：{integer} `string` 中包含的字节数。

返回使用 `encoding` 编码时字符串的字节长度。这与 [`String.prototype.length`][] 不同，后者不考虑用于将字符串转换为字节的编码。

对于 `'base64'`、`'base64url'` 和 `'hex'`，此函数假定输入有效。对于包含非 base64/十六进制编码数据（例如空格）的字符串，返回值可能大于从字符串创建的 `Buffer` 的长度。

```mjs
import { Buffer } from 'node:buffer';

const str = '\u00bd + \u00bc = \u00be';

console.log(`${str}: ${str.length} 个字符, ` +
            `${Buffer.byteLength(str, 'utf8')} 个字节`);
// 打印: ½ + ¼ = ¾: 9 个字符, 12 个字节
```

```cjs
const { Buffer } = require('node:buffer');

const str = '\u00bd + \u00bc = \u00be';

console.log(`${str}: ${str.length} 个字符, ` +
            `${Buffer.byteLength(str, 'utf8')} 个字节`);
// 打印: ½ + ¼ = ¾: 9 个字符, 12 个字节
```

当 `string` 是 {Buffer|DataView|TypedArray|ArrayBuffer|SharedArrayBuffer} 时，返回 `.byteLength` 报告的字节长度。

### 静态方法：`Buffer.compare(buf1, buf2)`

<!-- YAML
added: v0.11.13
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10236
    description: The arguments can now be `Uint8Array`s.
-->

* `buf1` {Buffer|Uint8Array}
* `buf2` {Buffer|Uint8Array}
* 返回：{integer} `-1`、`0` 或 `1`，具体取决于比较结果。有关详细信息，请参阅 [`buf.compare()`][]。

将 `buf1` 与 `buf2` 进行比较，通常用于对 `Buffer` 实例数组进行排序。这等效于调用 [`buf1.compare(buf2)`][`buf.compare()`]。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from('1234');
const buf2 = Buffer.from('0123');
const arr = [buf1, buf2];

console.log(arr.sort(Buffer.compare));
// 打印: [ <Buffer 30 31 32 33>, <Buffer 31 32 33 34> ]
//（此结果等于：[buf2, buf1]。）
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from('1234');
const buf2 = Buffer.from('0123');
const arr = [buf1, buf2];

console.log(arr.sort(Buffer.compare));
// 打印: [ <Buffer 30 31 32 33>, <Buffer 31 32 33 34> ]
//（此结果等于：[buf2, buf1]。）
```

### 静态方法：`Buffer.concat(list[, totalLength])`

<!-- YAML
added: v0.7.11
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10236
    description: The elements of `list` can now be `Uint8Array`s.
-->

* `list` {Buffer\[] | Uint8Array\[]} 要连接的 `Buffer` 或 {Uint8Array} 实例的列表。
* `totalLength` {integer} 连接时 `list` 中 `Buffer` 实例的总长度。
* 返回：{Buffer}

返回一个新的 `Buffer`，它是 `list` 中所有 `Buffer` 实例连接的结果。

如果列表没有项目，或者 `totalLength` 为 0，则返回一个新的零长度 `Buffer`。

如果未提供 `totalLength`，则通过将 `list` 中的 `Buffer` 实例的长度相加来计算。

如果提供了 `totalLength`，则将其强制转换为无符号整数。如果 `list` 中 `Buffer` 的组合长度超过 `totalLength`，则结果将被截断为 `totalLength`。如果 `list` 中 `Buffer` 的组合长度小于 `totalLength`，则剩余空间将用零填充。

```mjs
import { Buffer } from 'node:buffer';

// 从三个 `Buffer` 实例的列表创建一个 `Buffer`。

const buf1 = Buffer.alloc(10);
const buf2 = Buffer.alloc(14);
const buf3 = Buffer.alloc(18);
const totalLength = buf1.length + buf2.length + buf3.length;

console.log(totalLength);
// 打印: 42

const bufA = Buffer.concat([buf1, buf2, buf3], totalLength);

console.log(bufA);
// 打印: <Buffer 00 00 00 00 ...>
console.log(bufA.length);
// 打印: 42
```

```cjs
const { Buffer } = require('node:buffer');

// 从三个 `Buffer` 实例的列表创建一个 `Buffer`。

const buf1 = Buffer.alloc(10);
const buf2 = Buffer.alloc(14);
const buf3 = Buffer.alloc(18);
const totalLength = buf1.length + buf2.length + buf3.length;

console.log(totalLength);
// 打印: 42

const bufA = Buffer.concat([buf1, buf2, buf3], totalLength);

console.log(bufA);
// 打印: <Buffer 00 00 00 00 ...>
console.log(bufA.length);
// 打印: 42
```

`Buffer.concat()` 也可能像 [`Buffer.allocUnsafe()`][] 一样使用内部 `Buffer` 池。

### 静态方法：`Buffer.copyBytesFrom(view[, offset[, length]])`

<!-- YAML
added:
 - v19.8.0
 - v18.16.0
-->

* `view` {TypedArray} 要复制的 {TypedArray}。
* `offset` {integer} `view` 中的起始偏移量。**默认值：** `0`。
* `length` {integer} 要从 `view` 复制的元素数。
  **默认值：** `view.length - offset`。
* 返回：{Buffer}

将 `view` 的底层内存复制到一个新的 `Buffer` 中。

```js
const u16 = new Uint16Array([0, 0xffff]);
const buf = Buffer.copyBytesFrom(u16, 1, 1);
u16[1] = 0;
console.log(buf.length); // 2
console.log(buf[0]); // 255
console.log(buf[1]); // 255
```

### 静态方法：`Buffer.from(array)`

<!-- YAML
added: v5.10.0
-->

* `array` {integer\[]}
* 返回：{Buffer}

使用 `0` – `255` 范围内的字节数组分配一个新的 `Buffer`。超出该范围的数组条目将被截断以适合它。

```mjs
import { Buffer } from 'node:buffer';

// 创建一个包含字符串 'buffer' 的 UTF-8 字节的新 Buffer。
const buf = Buffer.from([0x62, 0x75, 0x66, 0x66, 0x65, 0x72]);
```

```cjs
const { Buffer } = require('node:buffer');

// 创建一个包含字符串 'buffer' 的 UTF-8 字节的新 Buffer。
const buf = Buffer.from([0x62, 0x75, 0x66, 0x66, 0x65, 0x72]);
```

如果 `array` 是一个类似 `Array` 的对象（即具有 `number` 类型的 `length` 属性），则将其视为数组，除非它是 `Buffer` 或 `Uint8Array`。这意味着所有其他 `TypedArray` 变体都被视为 `Array`。要从支持 `TypedArray` 的字节创建 `Buffer`，请使用 [`Buffer.copyBytesFrom()`][]。

如果 `array` 不是 `Array` 或另一种适合 `Buffer.from()` 变体的类型，则会抛出 `TypeError`。

`Buffer.from(array)` 和 [`Buffer.from(string)`][] 也可能像 [`Buffer.allocUnsafe()`][] 一样使用内部 `Buffer` 池。

### 静态方法：`Buffer.from(arrayBuffer[, byteOffset[, length]])`

<!-- YAML
added: v5.10.0
-->

* `arrayBuffer` {ArrayBuffer|SharedArrayBuffer} 一个 {ArrayBuffer}、{SharedArrayBuffer}，例如 {TypedArray} 的 `.buffer` 属性。
* `byteOffset` {integer} 要暴露的第一个字节的索引。**默认值：** `0`。
* `length` {integer} 要暴露的字节数。
  **默认值：** `arrayBuffer.byteLength - byteOffset`。
* 返回：{Buffer}

这创建了 {ArrayBuffer} 的视图，而不复制底层内存。例如，当传递对 {TypedArray} 实例的 `.buffer` 属性的引用时，新创建的 `Buffer` 将与 {TypedArray} 的底层 `ArrayBuffer` 共享相同的已分配内存。

```mjs
import { Buffer } from 'node:buffer';

const arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// 与 `arr` 共享内存。
const buf = Buffer.from(arr.buffer);

console.log(buf);
// 打印: <Buffer 88 13 a0 0f>

// 更改原始 Uint16Array 也会更改 Buffer。
arr[1] = 6000;

console.log(buf);
// 打印: <Buffer 88 13 70 17>
```

```cjs
const { Buffer } = require('node:buffer');

const arr = new Uint16Array(2);

arr[0] = 5000;
arr[1] = 4000;

// 与 `arr` 共享内存。
const buf = Buffer.from(arr.buffer);

console.log(buf);
// 打印: <Buffer 88 13 a0 0f>

// 更改原始 Uint16Array 也会更改 Buffer。
arr[1] = 6000;

console.log(buf);
// 打印: <Buffer 88 13 70 17>
```

可选的 `byteOffset` 和 `length` 参数指定 `arrayBuffer` 中将由 `Buffer` 共享的内存范围。

```mjs
import { Buffer } from 'node:buffer';

const ab = new ArrayBuffer(10);
const buf = Buffer.from(ab, 0, 2);

console.log(buf.length);
// 打印: 2
```

```cjs
const { Buffer } = require('node:buffer');

const ab = new ArrayBuffer(10);
const buf = Buffer.from(ab, 0, 2);

console.log(buf.length);
// 打印: 2
```

如果 `arrayBuffer` 不是 {ArrayBuffer} 或 {SharedArrayBuffer} 或另一种适合 `Buffer.from()` 变体的类型，则会抛出 `TypeError`。

重要的是要记住，后备 `ArrayBuffer` 可以覆盖超出 `TypedArray` 视图边界的内存范围。使用 `TypedArray` 的 `buffer` 属性创建的新的 `Buffer` 可能会超出 `TypedArray` 的范围：

```mjs
import { Buffer } from 'node:buffer';

const arrA = Uint8Array.from([0x63, 0x64, 0x65, 0x66]); // 4 个元素
const arrB = new Uint8Array(arrA.buffer, 1, 2); // 2 个元素
console.log(arrA.buffer === arrB.buffer); // true

const buf = Buffer.from(arrB.buffer);
console.log(buf);
// 打印: <Buffer 63 64 65 66>
```

```cjs
const { Buffer } = require('node:buffer');

const arrA = Uint8Array.from([0x63, 0x64, 0x65, 0x66]); // 4 个元素
const arrB = new Uint8Array(arrA.buffer, 1, 2); // 2 个元素
console.log(arrA.buffer === arrB.buffer); // true

const buf = Buffer.from(arrB.buffer);
console.log(buf);
// 打印: <Buffer 63 64 65 66>
```

### 静态方法：`Buffer.from(buffer)`

<!-- YAML
added: v5.10.0
-->

* `buffer` {Buffer|Uint8Array} 要从中复制数据的现有 `Buffer` 或 {Uint8Array}。
* 返回：{Buffer}

将传递的 `buffer` 数据复制到新的 `Buffer` 实例中。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from('buffer');
const buf2 = Buffer.from(buf1);

buf1[0] = 0x61;

console.log(buf1.toString());
// 打印: auffer
console.log(buf2.toString());
// 打印: buffer
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from('buffer');
const buf2 = Buffer.from(buf1);

buf1[0] = 0x61;

console.log(buf1.toString());
// 打印: auffer
console.log(buf2.toString());
// 打印: buffer
```

如果 `buffer` 不是 `Buffer` 或另一种适合 `Buffer.from()` 变体的类型，则会抛出 `TypeError`。

### 静态方法：`Buffer.from(object[, offsetOrEncoding[, length]])`

<!-- YAML
added: v8.2.0
-->

* `object` {Object} 支持 `Symbol.toPrimitive` 或 `valueOf()` 的对象。
* `offsetOrEncoding` {integer|string} 字节偏移量或编码。
* `length` {integer} 长度。
* 返回：{Buffer}

对于 `valueOf()` 函数返回的值不严格等于 `object` 的对象，返回 `Buffer.from(object.valueOf(), offsetOrEncoding, length)`。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from(new String('this is a test'));
// 打印: <Buffer 74 68 69 73 20 69 73 20 61 20 74 65 73 74>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from(new String('this is a test'));
// 打印: <Buffer 74 68 69 73 20 69 73 20 61 20 74 65 73 74>
```

对于支持 `Symbol.toPrimitive` 的对象，返回 `Buffer.from(object[Symbol.toPrimitive]('string'), offsetOrEncoding)`。

```mjs
import { Buffer } from 'node:buffer';

class Foo {
  [Symbol.toPrimitive]() {
    return 'this is a test';
  }
}

const buf = Buffer.from(new Foo(), 'utf8');
// 打印: <Buffer 74 68 69 73 20 69 73 20 61 20 74 65 73 74>
```

```cjs
const { Buffer } = require('node:buffer');

class Foo {
  [Symbol.toPrimitive]() {
    return 'this is a test';
  }
}

const buf = Buffer.from(new Foo(), 'utf8');
// 打印: <Buffer 74 68 69 73 20 69 73 20 61 20 74 65 73 74>
```

如果 `object` 没有提到的方法或不是另一种适合 `Buffer.from()` 变体的类型，则会抛出 `TypeError`。

### 静态方法：`Buffer.from(string[, encoding])`

<!-- YAML
added: v5.10.0
-->

* `string` {string} 要编码的字符串。
* `encoding` {string} `string` 的编码。**默认值：** `'utf8'`。
* 返回：{Buffer}

创建一个包含 `string` 的新 `Buffer`。`encoding` 参数标识将 `string` 转换为字节时要使用的字符编码。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from('this is a tést');
const buf2 = Buffer.from('7468697320697320612074c3a97374', 'hex');

console.log(buf1.toString());
// 打印: this is a tést
console.log(buf2.toString());
// 打印: this is a tést
console.log(buf1.toString('latin1'));
// 打印: this is a tÃ©st
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from('this is a tést');
const buf2 = Buffer.from('7468697320697320612074c3a97374', 'hex');

console.log(buf1.toString());
// 打印: this is a tést
console.log(buf2.toString());
// 打印: this is a tést
console.log(buf1.toString('latin1'));
// 打印: this is a tÃ©st
```

如果 `string` 不是字符串或另一种适合 `Buffer.from()` 变体的类型，则会抛出 `TypeError`。

[`Buffer.from(string)`][] 也可能像 [`Buffer.allocUnsafe()`][] 一样使用内部 `Buffer` 池。

### 静态方法：`Buffer.isBuffer(obj)`

<!-- YAML
added: v0.1.101
-->

* `obj` {Object}
* 返回：{boolean}

如果 `obj` 是 `Buffer`，则返回 `true`，否则返回 `false`。

```mjs
import { Buffer } from 'node:buffer';

Buffer.isBuffer(Buffer.alloc(10)); // true
Buffer.isBuffer(Buffer.from('foo')); // true
Buffer.isBuffer('a string'); // false
Buffer.isBuffer([]); // false
Buffer.isBuffer(new Uint8Array(1024)); // false
```

```cjs
const { Buffer } = require('node:buffer');

Buffer.isBuffer(Buffer.alloc(10)); // true
Buffer.isBuffer(Buffer.from('foo')); // true
Buffer.isBuffer('a string'); // false
Buffer.isBuffer([]); // false
Buffer.isBuffer(new Uint8Array(1024)); // false
```

### 静态方法：`Buffer.isEncoding(encoding)`

<!-- YAML
added: v0.9.1
-->

* `encoding` {string} 要检查的字符编码名称。
* 返回：{boolean}

如果 `encoding` 是受支持的字符编码的名称，则返回 `true`，否则返回 `false`。

```mjs
import { Buffer } from 'node:buffer';

console.log(Buffer.isEncoding('utf8'));
// 打印: true

console.log(Buffer.isEncoding('hex'));
// 打印: true

console.log(Buffer.isEncoding('utf/8'));
// 打印: false

console.log(Buffer.isEncoding(''));
// 打印: false
```

```cjs
const { Buffer } = require('node:buffer');

console.log(Buffer.isEncoding('utf8'));
// 打印: true

console.log(Buffer.isEncoding('hex'));
// 打印: true

console.log(Buffer.isEncoding('utf/8'));
// 打印: false

console.log(Buffer.isEncoding(''));
// 打印: false
```

### `Buffer.poolSize`

<!-- YAML
added: v0.11.3
-->

* 类型：{integer} **默认值：** `8192`

这是用于池化的预分配内部 `Buffer` 实例的大小（以字节为单位）。可以修改此值。

### `buf[index]`

* `index` {integer}

索引运算符 `[index]` 可用于获取和设置 `buf` 中位置 `index` 处的八位字节。值指的是单个字节，因此合法值范围在 `0x00` 和 `0xFF`（十六进制）或 `0` 和 `255`（十进制）之间。

此运算符继承自 `Uint8Array`，因此其对越界访问的行为与 `Uint8Array` 相同。换句话说，当 `index` 为负数或大于或等于 `buf.length` 时，`buf[index]` 返回 `undefined`，并且当 `index` 为负数或 `>= buf.length` 时，`buf[index] = value` 不会修改缓冲区。

```mjs
import { Buffer } from 'node:buffer';

// 将 ASCII 字符串逐字节复制到 `Buffer` 中。
//（这仅适用于仅包含 ASCII 的字符串。通常，应该使用
// `Buffer.from()` 来执行此转换。）

const str = 'Node.js';
const buf = Buffer.allocUnsafe(str.length);

for (let i = 0; i < str.length; i++) {
  buf[i] = str.charCodeAt(i);
}

console.log(buf.toString('utf8'));
// 打印: Node.js
```

```cjs
const { Buffer } = require('node:buffer');

// 将 ASCII 字符串逐字节复制到 `Buffer` 中。
//（这仅适用于仅包含 ASCII 的字符串。通常，应该使用
// `Buffer.from()` 来执行此转换。）

const str = 'Node.js';
const buf = Buffer.allocUnsafe(str.length);

for (let i = 0; i < str.length; i++) {
  buf[i] = str.charCodeAt(i);
}

console.log(buf.toString('utf8'));
// 打印: Node.js
```

### `buf.buffer`

* 类型：{ArrayBuffer} 此 `Buffer` 对象基于的底层 `ArrayBuffer` 对象。

不保证此 `ArrayBuffer` 与原始 `Buffer` 完全对应。有关 `buf.byteOffset` 的说明，请参阅注释。

```mjs
import { Buffer } from 'node:buffer';

const arrayBuffer = new ArrayBuffer(16);
const buffer = Buffer.from(arrayBuffer);

console.log(buffer.buffer === arrayBuffer);
// 打印: true
```

```cjs
const { Buffer } = require('node:buffer');

const arrayBuffer = new ArrayBuffer(16);
const buffer = Buffer.from(arrayBuffer);

console.log(buffer.buffer === arrayBuffer);
// 打印: true
```

### `buf.byteOffset`

* 类型：{integer} `Buffer` 的底层 `ArrayBuffer` 对象的 `byteOffset`。

当在 `Buffer.from(ArrayBuffer, byteOffset, length)` 中设置 `byteOffset`，或者有时在分配小于 `Buffer.poolSize` 的 `Buffer` 时，缓冲区不会从底层 `ArrayBuffer` 的零偏移开始。

当使用 `buf.buffer` 直接访问底层 `ArrayBuffer` 时，这可能会导致问题，因为 `ArrayBuffer` 的其他部分可能与 `Buffer` 对象本身无关。

创建与 `Buffer` 共享其内存的 `TypedArray` 对象时，一个常见问题是需要正确指定 `byteOffset`：

```mjs
import { Buffer } from 'node:buffer';

// 创建一个小于 `Buffer.poolSize` 的缓冲区。
const nodeBuffer = Buffer.from([0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);

// 将 Node.js Buffer 转换为 Int8Array 时，使用 byteOffset
// 仅引用包含 `nodeBuffer` 内存的 `nodeBuffer.buffer` 部分。
new Int8Array(nodeBuffer.buffer, nodeBuffer.byteOffset, nodeBuffer.length);
```

```cjs
const { Buffer } = require('node:buffer');

// 创建一个小于 `Buffer.poolSize` 的缓冲区。
const nodeBuffer = Buffer.from([0, 1, 2, 3, 4, 5, 6, 7, 8, 9]);

// 将 Node.js Buffer 转换为 Int8Array 时，使用 byteOffset
// 仅引用包含 `nodeBuffer` 内存的 `nodeBuffer.buffer` 部分。
new Int8Array(nodeBuffer.buffer, nodeBuffer.byteOffset, nodeBuffer.length);
```

### `buf.compare(target[, targetStart[, targetEnd[, sourceStart[, sourceEnd]]]])`

<!-- YAML
added: v0.11.13
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10236
    description: The `target` parameter can now be a `Uint8Array`.
  - version: v5.11.0
    pr-url: https://github.com/nodejs/node/pull/5880
    description: Additional parameters for specifying offsets are supported now.
-->

* `target` {Buffer|Uint8Array} 要与 `buf` 比较的 `Buffer` 或 {Uint8Array}。
* `targetStart` {integer} `target` 中开始比较的偏移量。**默认值：** `0`。
* `targetEnd` {integer} `target` 中结束比较的偏移量（不包括）。**默认值：** `target.length`。
* `sourceStart` {integer} `buf` 中开始比较的偏移量。
  **默认值：** `0`。
* `sourceEnd` {integer} `buf` 中结束比较的偏移量（不包括）。
  **默认值：** [`buf.length`][]。
* 返回：{integer}

将 `buf` 与 `target` 进行比较，并返回一个数字，指示 `buf` 在排序顺序中是在 `target` 之前、之后还是相同。比较基于每个 `Buffer` 中的实际字节序列。

* 如果 `target` 与 `buf` 相同，则返回 `0`
* 如果排序时 `target` 应该出现在 `buf` *之前*，则返回 `1`。
* 如果排序时 `target` 应该出现在 `buf` *之后*，则返回 `-1`。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from('ABC');
const buf2 = Buffer.from('BCD');
const buf3 = Buffer.from('ABCD');

console.log(buf1.compare(buf1));
// 打印: 0
console.log(buf1.compare(buf2));
// 打印: -1
console.log(buf1.compare(buf3));
// 打印: -1
console.log(buf2.compare(buf1));
// 打印: 1
console.log(buf2.compare(buf3));
// 打印: 1
console.log([buf1, buf2, buf3].sort(Buffer.compare));
// 打印: [ <Buffer 41 42 43>, <Buffer 41 42 43 44>, <Buffer 42 43 44> ]
//（此结果等于：[buf1, buf3, buf2]。）
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from('ABC');
const buf2 = Buffer.from('BCD');
const buf3 = Buffer.from('ABCD');

console.log(buf1.compare(buf1));
// 打印: 0
console.log(buf1.compare(buf2));
// 打印: -1
console.log(buf1.compare(buf3));
// 打印: -1
console.log(buf2.compare(buf1));
// 打印: 1
console.log(buf2.compare(buf3));
// 打印: 1
console.log([buf1, buf2, buf3].sort(Buffer.compare));
// 打印: [ <Buffer 41 42 43>, <Buffer 41 42 43 44>, <Buffer 42 43 44> ]
//（此结果等于：[buf1, buf3, buf2]。）
```

可选的 `targetStart`、`targetEnd`、`sourceStart` 和 `sourceEnd` 参数可用于分别将比较限制在 `target` 和 `buf` 中的特定范围内。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from([1, 2, 3, 4, 5, 6, 7, 8, 9]);
const buf2 = Buffer.from([5, 6, 7, 8, 9, 1, 2, 3, 4]);

console.log(buf1.compare(buf2, 5, 9, 0, 4));
// 打印: 0
console.log(buf1.compare(buf2, 0, 6, 4));
// 打印: -1
console.log(buf1.compare(buf2, 5, 6, 5));
// 打印: 1
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from([1, 2, 3, 4, 5, 6, 7, 8, 9]);
const buf2 = Buffer.from([5, 6, 7, 8, 9, 1, 2, 3, 4]);

console.log(buf1.compare(buf2, 5, 9, 0, 4));
// 打印: 0
console.log(buf1.compare(buf2, 0, 6, 4));
// 打印: -1
console.log(buf1.compare(buf2, 5, 6, 5));
// 打印: 1
```

如果 `targetStart < 0`、`sourceStart < 0`、`targetEnd > target.byteLength` 或 `sourceEnd > source.byteLength`，则会抛出 [`ERR_OUT_OF_RANGE`][]。

### `buf.copy(target[, targetStart[, sourceStart[, sourceEnd]]])`

<!-- YAML
added: v0.1.90
-->

* `target` {Buffer|Uint8Array} 要复制到的 `Buffer` 或 {Uint8Array}。
* `targetStart` {integer} `target` 中开始写入的偏移量。**默认值：** `0`。
* `sourceStart` {integer} `buf` 中开始复制的偏移量。
  **默认值：** `0`。
* `sourceEnd` {integer} `buf` 中停止复制的偏移量（不包括）。
  **默认值：** [`buf.length`][]。
* 返回：{integer} 复制的字节数。

将数据从 `buf` 的一个区域复制到 `target` 的一个区域，即使 `target` 内存区域与 `buf` 重叠。

[`TypedArray.prototype.set()`][] 执行相同的操作，并且可用于所有 TypedArray，包括 Node.js `Buffer`，尽管它采用不同的函数参数。

```mjs
import { Buffer } from 'node:buffer';

// 创建两个 `Buffer` 实例。
const buf1 = Buffer.allocUnsafe(26);
const buf2 = Buffer.allocUnsafe(26).fill('!');

for (let i = 0; i < 26; i++) {
  // 97 是 'a' 的十进制 ASCII 值。
  buf1[i] = i + 97;
}

// 将 `buf1` 的字节 16 到 19 复制到 `buf2` 中，从 `buf2` 的字节 8 开始。
buf1.copy(buf2, 8, 16, 20);
// 这等效于：
// buf2.set(buf1.subarray(16, 20), 8);

console.log(buf2.toString('ascii', 0, 25));
// 打印: !!!!!!!!qrst!!!!!!!!!!!!!
```

```cjs
const { Buffer } = require('node:buffer');

// 创建两个 `Buffer` 实例。
const buf1 = Buffer.allocUnsafe(26);
const buf2 = Buffer.allocUnsafe(26).fill('!');

for (let i = 0; i < 26; i++) {
  // 97 是 'a' 的十进制 ASCII 值。
  buf1[i] = i + 97;
}

// 将 `buf1` 的字节 16 到 19 复制到 `buf2` 中，从 `buf2` 的字节 8 开始。
buf1.copy(buf2, 8, 16, 20);
// 这等效于：
// buf2.set(buf1.subarray(16, 20), 8);

console.log(buf2.toString('ascii', 0, 25));
// 打印: !!!!!!!!qrst!!!!!!!!!!!!!
```

```mjs
import { Buffer } from 'node:buffer';

// 创建一个 `Buffer` 并将数据从一个区域复制到同一 `Buffer` 中的重叠区域。

const buf = Buffer.allocUnsafe(26);

for (let i = 0; i < 26; i++) {
  // 97 是 'a' 的十进制 ASCII 值。
  buf[i] = i + 97;
}

buf.copy(buf, 0, 4, 10);

console.log(buf.toString());
// 打印: efghijghijklmnopqrstuvwxyz
```

```cjs
const { Buffer } = require('node:buffer');

// 创建一个 `Buffer` 并将数据从一个区域复制到同一 `Buffer` 中的重叠区域。

const buf = Buffer.allocUnsafe(26);

for (let i = 0; i < 26; i++) {
  // 97 是 'a' 的十进制 ASCII 值。
  buf[i] = i + 97;
}

buf.copy(buf, 0, 4, 10);

console.log(buf.toString());
// 打印: efghijghijklmnopqrstuvwxyz
```

### `buf.entries()`

<!-- YAML
added: v1.1.0
-->

* 返回：{Iterator}

从 `buf` 的内容创建并返回一个 `[index, byte]` 对的[迭代器][]。

```mjs
import { Buffer } from 'node:buffer';

// 记录 `Buffer` 的整个内容。

const buf = Buffer.from('buffer');

for (const pair of buf.entries()) {
  console.log(pair);
}
// 打印:
//   [0, 98]
//   [1, 117]
//   [2, 102]
//   [3, 102]
//   [4, 101]
//   [5, 114]
```

```cjs
const { Buffer } = require('node:buffer');

// 记录 `Buffer` 的整个内容。

const buf = Buffer.from('buffer');

for (const pair of buf.entries()) {
  console.log(pair);
}
// 打印:
//   [0, 98]
//   [1, 117]
//   [2, 102]
//   [3, 102]
//   [4, 101]
//   [5, 114]
```

### `buf.equals(otherBuffer)`

<!-- YAML
added: v0.11.13
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10236
    description: The arguments can now be `Uint8Array`s.
-->

* `otherBuffer` {Buffer|Uint8Array} 要与 `buf` 比较的 `Buffer` 或 {Uint8Array}。
* 返回：{boolean}

如果 `buf` 和 `otherBuffer` 具有完全相同的字节，则返回 `true`，否则返回 `false`。等效于 [`buf.compare(otherBuffer) === 0`][`buf.compare()`]。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from('ABC');
const buf2 = Buffer.from('414243', 'hex');
const buf3 = Buffer.from('ABCD');

console.log(buf1.equals(buf2));
// 打印: true
console.log(buf1.equals(buf3));
// 打印: false
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from('ABC');
const buf2 = Buffer.from('414243', 'hex');
const buf3 = Buffer.from('ABCD');

console.log(buf1.equals(buf2));
// 打印: true
console.log(buf1.equals(buf3));
// 打印: false
```

### `buf.fill(value[, offset[, end]][, encoding])`

<!-- YAML
added: v0.5.0
changes:
  - version: v11.0.0
    pr-url: https://github.com/nodejs/node/pull/22969
    description: Throws `ERR_OUT_OF_RANGE` instead of `ERR_INDEX_OUT_OF_RANGE`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18790
    description: Negative `end` values throw an `ERR_INDEX_OUT_OF_RANGE` error.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18129
    description: Attempting to fill a non-zero length buffer with a zero length
                 buffer triggers a thrown exception.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/17427
    description: Specifying an invalid string for `value` triggers a thrown
                 exception.
  - version: v5.7.0
    pr-url: https://github.com/nodejs/node/pull/4935
    description: The `encoding` parameter is supported now.
-->

* `value` {string|Buffer|Uint8Array|integer} 用于填充 `buf` 的值。
  空值（字符串、Uint8Array、Buffer）被强制转换为 `0`。
* `offset` {integer} 在开始填充 `buf` 之前要跳过的字节数。
  **默认值：** `0`。
* `end` {integer} 停止填充 `buf` 的位置（不包括）。**默认值：**
  [`buf.length`][]。
* `encoding` {string} 如果 `value` 是字符串，则这是它的编码。
  **默认值：** `'utf8'`。
* 返回：{Buffer} 对 `buf` 的引用。

用指定的 `value` 填充 `buf`。如果未给出 `offset` 和 `end`，则将填充整个 `buf`：

```mjs
import { Buffer } from 'node:buffer';

// 用 ASCII 字符 'h' 填充 `Buffer`。

const b = Buffer.allocUnsafe(50).fill('h');

console.log(b.toString());
// 打印: hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh

// 用空字符串填充缓冲区
const c = Buffer.allocUnsafe(5).fill('');

console.log(c.fill(''));
// 打印: <Buffer 00 00 00 00 00>
```

```cjs
const { Buffer } = require('node:buffer');

// 用 ASCII 字符 'h' 填充 `Buffer`。

const b = Buffer.allocUnsafe(50).fill('h');

console.log(b.toString());
// 打印: hhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhhh

// 用空字符串填充缓冲区
const c = Buffer.allocUnsafe(5).fill('');

console.log(c.fill(''));
// 打印: <Buffer 00 00 00 00 00>
```

如果 `value` 不是字符串、`Buffer` 或整数，则将其强制转换为 `uint32` 值。如果结果整数大于 `255`（十进制），则 `buf` 将用 `value & 255` 填充。

如果 `fill()` 操作的最终写入落在多字节字符上，则仅写入适合 `buf` 的该字符的字节：

```mjs
import { Buffer } from 'node:buffer';

// 用在 UTF-8 中占用两个字节的字符填充 `Buffer`。

console.log(Buffer.allocUnsafe(5).fill('\u0222'));
// 打印: <Buffer c8 a2 c8 a2 c8>
```

```cjs
const { Buffer } = require('node:buffer');

// 用在 UTF-8 中占用两个字节的字符填充 `Buffer`。

console.log(Buffer.allocUnsafe(5).fill('\u0222'));
// 打印: <Buffer c8 a2 c8 a2 c8>
```

如果 `value` 包含无效字符，则它将被截断；如果没有有效的填充数据剩余，则会抛出异常：

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(5);

console.log(buf.fill('a'));
// 打印: <Buffer 61 61 61 61 61>
console.log(buf.fill('aazz', 'hex'));
// 打印: <Buffer aa aa aa aa aa>
console.log(buf.fill('zz', 'hex'));
// 抛出异常。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(5);

console.log(buf.fill('a'));
// 打印: <Buffer 61 61 61 61 61>
console.log(buf.fill('aazz', 'hex'));
// 打印: <Buffer aa aa aa aa aa>
console.log(buf.fill('zz', 'hex'));
// 抛出异常。
```

### `buf.includes(value[, byteOffset][, encoding])`

<!-- YAML
added: v5.3.0
-->

* `value` {string|Buffer|Uint8Array|integer} 要搜索的内容。
* `byteOffset` {integer} 在 `buf` 中开始搜索的位置。如果为负数，则从 `buf` 的末尾计算偏移量。**默认值：** `0`。
* `encoding` {string} 如果 `value` 是字符串，则这是它的编码。
  **默认值：** `'utf8'`。
* 返回：{boolean} 如果在 `buf` 中找到 `value`，则为 `true`，否则为 `false`。

等效于 [`buf.indexOf() !== -1`][`buf.indexOf()`]。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('this is a buffer');

console.log(buf.includes('this'));
// 打印: true
console.log(buf.includes('is'));
// 打印: true
console.log(buf.includes(Buffer.from('a buffer')));
// 打印: true
console.log(buf.includes(97));
// 打印: true (97 是 'a' 的十进制 ASCII 值)
console.log(buf.includes(Buffer.from('a buffer example')));
// 打印: false
console.log(buf.includes(Buffer.from('a buffer example').slice(0, 8)));
// 打印: true
console.log(buf.includes('this', 4));
// 打印: false
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('this is a buffer');

console.log(buf.includes('this'));
// 打印: true
console.log(buf.includes('is'));
// 打印: true
console.log(buf.includes(Buffer.from('a buffer')));
// 打印: true
console.log(buf.includes(97));
// 打印: true (97 是 'a' 的十进制 ASCII 值)
console.log(buf.includes(Buffer.from('a buffer example')));
// 打印: false
console.log(buf.includes(Buffer.from('a buffer example').slice(0, 8)));
// 打印: true
console.log(buf.includes('this', 4));
// 打印: false
```

### `buf.indexOf(value[, byteOffset][, encoding])`

<!-- YAML
added: v1.5.0
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10236
    description: The `value` can now be a `Uint8Array`.
  - version:
    - v5.7.0
    - v4.4.0
    pr-url: https://github.com/nodejs/node/pull/4803
    description: When `encoding` is being passed, the `byteOffset` parameter
                 is no longer required.
-->

* `value` {string|Buffer|Uint8Array|integer} 要搜索的内容。
* `byteOffset` {integer} 在 `buf` 中开始搜索的位置。如果为负数，则从 `buf` 的末尾计算偏移量。**默认值：** `0`。
* `encoding` {string} 如果 `value` 是字符串，则这是用于确定将在 `buf` 中搜索的字符串的二进制表示的编码。**默认值：** `'utf8'`。
* 返回：{integer} `buf` 中 `value` 第一次出现的索引，如果 `buf` 不包含 `value`，则为 `-1`。

如果 `value` 是：

* 字符串，则根据 `encoding` 中的字符编码解释 `value`。
* `Buffer` 或 {Uint8Array}，则将完全使用 `value`。要比较部分 `Buffer`，请使用 [`buf.subarray`][]。
* 数字，则 `value` 将被解释为介于 `0` 和 `255` 之间的无符号 8 位整数值。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('this is a buffer');

console.log(buf.indexOf('this'));
// 打印: 0
console.log(buf.indexOf('is'));
// 打印: 2
console.log(buf.indexOf(Buffer.from('a buffer')));
// 打印: 8
console.log(buf.indexOf(97));
// 打印: 8 (97 是 'a' 的十进制 ASCII 值)
console.log(buf.indexOf(Buffer.from('a buffer example')));
// 打印: -1
console.log(buf.indexOf(Buffer.from('a buffer example').slice(0, 8)));
// 打印: 8

const utf16Buffer = Buffer.from('\u039a\u0391\u03a3\u03a3\u0395', 'utf16le');

console.log(utf16Buffer.indexOf('\u03a3', 0, 'utf16le'));
// 打印: 4
console.log(utf16Buffer.indexOf('\u03a3', -4, 'utf16le'));
// 打印: 6
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('this is a buffer');

console.log(buf.indexOf('this'));
// 打印: 0
console.log(buf.indexOf('is'));
// 打印: 2
console.log(buf.indexOf(Buffer.from('a buffer')));
// 打印: 8
console.log(buf.indexOf(97));
// 打印: 8 (97 是 'a' 的十进制 ASCII 值)
console.log(buf.indexOf(Buffer.from('a buffer example')));
// 打印: -1
console.log(buf.indexOf(Buffer.from('a buffer example').slice(0, 8)));
// 打印: 8

const utf16Buffer = Buffer.from('\u039a\u0391\u03a3\u03a3\u0395', 'utf16le');

console.log(utf16Buffer.indexOf('\u03a3', 0, 'utf16le'));
// 打印: 4
console.log(utf16Buffer.indexOf('\u03a3', -4, 'utf16le'));
// 打印: 6
```

如果 `value` 不是字符串、数字或 `Buffer`，则此方法将抛出 `TypeError`。如果 `value` 是数字，则它将被强制转换为有效的字节值，即 0 到 255 之间的整数。

如果 `byteOffset` 不是数字，则它将被强制转换为数字。如果强制转换的结果是 `NaN` 或 `0`，则将搜索整个缓冲区。此行为匹配 [`String.prototype.indexOf()`][]。

```mjs
import { Buffer } from 'node:buffer';

const b = Buffer.from('abcdef');

// 传递一个值是数字但无效的字节。
// 打印: 2，等效于搜索 99 或 'c'。
console.log(b.indexOf(99.9));
console.log(b.indexOf(256 + 99));

// 传递强制转换为 NaN 或 0 的 byteOffset。
// 打印: 1，搜索整个缓冲区。
console.log(b.indexOf('b', undefined));
console.log(b.indexOf('b', {}));
console.log(b.indexOf('b', null));
console.log(b.indexOf('b', []));
```

```cjs
const { Buffer } = require('node:buffer');

const b = Buffer.from('abcdef');

// 传递一个值是数字但无效的字节。
// 打印: 2，等效于搜索 99 或 'c'。
console.log(b.indexOf(99.9));
console.log(b.indexOf(256 + 99));

// 传递强制转换为 NaN 或 0 的 byteOffset。
// 打印: 1，搜索整个缓冲区。
console.log(b.indexOf('b', undefined));
console.log(b.indexOf('b', {}));
console.log(b.indexOf('b', null));
console.log(b.indexOf('b', []));
```

如果 `value` 是空字符串或空 `Buffer` 且 `byteOffset` 小于 `buf.length`，则将返回 `byteOffset`。如果 `value` 为空且 `byteOffset` 至少为 `buf.length`，则将返回 `buf.length`。

### `buf.keys()`

<!-- YAML
added: v1.1.0
-->

* 返回：{Iterator}

创建并返回 `buf` 键（索引）的[迭代器][]。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('buffer');

for (const key of buf.keys()) {
  console.log(key);
}
// 打印:
//   0
//   1
//   2
//   3
//   4
//   5
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('buffer');

for (const key of buf.keys()) {
  console.log(key);
}
// 打印:
//   0
//   1
//   2
//   3
//   4
//   5
```

### `buf.lastIndexOf(value[, byteOffset][, encoding])`

<!-- YAML
added: v6.0.0
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10236
    description: The `value` can now be a `Uint8Array`.
-->

* `value` {string|Buffer|Uint8Array|integer} 要搜索的内容。
* `byteOffset` {integer} 在 `buf` 中开始搜索的位置。如果为负数，则从 `buf` 的末尾计算偏移量。**默认值：**
  `buf.length - 1`。
* `encoding` {string} 如果 `value` 是字符串，则这是用于确定将在 `buf` 中搜索的字符串的二进制表示的编码。**默认值：** `'utf8'`。
* 返回：{integer} `buf` 中 `value` 最后一次出现的索引，如果 `buf` 不包含 `value`，则为 `-1`。

与 [`buf.indexOf()`][] 相同，除了找到 `value` 的最后一次出现而不是第一次出现。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('this buffer is a buffer');

console.log(buf.lastIndexOf('this'));
// 打印: 0
console.log(buf.lastIndexOf('buffer'));
// 打印: 17
console.log(buf.lastIndexOf(Buffer.from('buffer')));
// 打印: 17
console.log(buf.lastIndexOf(97));
// 打印: 15 (97 是 'a' 的十进制 ASCII 值)
console.log(buf.lastIndexOf(Buffer.from('yolo')));
// 打印: -1
console.log(buf.lastIndexOf('buffer', 5));
// 打印: 5
console.log(buf.lastIndexOf('buffer', 4));
// 打印: -1

const utf16Buffer = Buffer.from('\u039a\u0391\u03a3\u03a3\u0395', 'utf16le');

console.log(utf16Buffer.lastIndexOf('\u03a3', undefined, 'utf16le'));
// 打印: 6
console.log(utf16Buffer.lastIndexOf('\u03a3', -5, 'utf16le'));
// 打印: 4
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('this buffer is a buffer');

console.log(buf.lastIndexOf('this'));
// 打印: 0
console.log(buf.lastIndexOf('buffer'));
// 打印: 17
console.log(buf.lastIndexOf(Buffer.from('buffer')));
// 打印: 17
console.log(buf.lastIndexOf(97));
// 打印: 15 (97 是 'a' 的十进制 ASCII 值)
console.log(buf.lastIndexOf(Buffer.from('yolo')));
// 打印: -1
console.log(buf.lastIndexOf('buffer', 5));
// 打印: 5
console.log(buf.lastIndexOf('buffer', 4));
// 打印: -1

const utf16Buffer = Buffer.from('\u039a\u0391\u03a3\u03a3\u0395', 'utf16le');

console.log(utf16Buffer.lastIndexOf('\u03a3', undefined, 'utf16le'));
// 打印: 6
console.log(utf16Buffer.lastIndexOf('\u03a3', -5, 'utf16le'));
// 打印: 4
```

如果 `value` 不是字符串、数字或 `Buffer`，则此方法将抛出 `TypeError`。如果 `value` 是数字，则它将被强制转换为有效的字节值，即 0 到 255 之间的整数。

如果 `byteOffset` 不是数字，则它将被强制转换为数字。任何强制转换为 `NaN` 的参数，例如 `{}` 或 `undefined`，都将搜索整个缓冲区。此行为匹配 [`String.prototype.lastIndexOf()`][]。

```mjs
import { Buffer } from 'node:buffer';

const b = Buffer.from('abcdef');

// 传递一个值是数字但无效的字节。
// 打印: 2，等效于搜索 99 或 'c'。
console.log(b.lastIndexOf(99.9));
console.log(b.lastIndexOf(256 + 99));

// 传递强制转换为 NaN 的 byteOffset。
// 打印: 1，搜索整个缓冲区。
console.log(b.lastIndexOf('b', undefined));
console.log(b.lastIndexOf('b', {}));

// 传递强制转换为 0 的 byteOffset。
// 打印: -1，等效于传递 0。
console.log(b.lastIndexOf('b', null));
console.log(b.lastIndexOf('b', []));
```

```cjs
const { Buffer } = require('node:buffer');

const b = Buffer.from('abcdef');

// 传递一个值是数字但无效的字节。
// 打印: 2，等效于搜索 99 或 'c'。
console.log(b.lastIndexOf(99.9));
console.log(b.lastIndexOf(256 + 99));

// 传递强制转换为 NaN 的 byteOffset。
// 打印: 1，搜索整个缓冲区。
console.log(b.lastIndexOf('b', undefined));
console.log(b.lastIndexOf('b', {}));

// 传递强制转换为 0 的 byteOffset。
// 打印: -1，等效于传递 0。
console.log(b.lastIndexOf('b', null));
console.log(b.lastIndexOf('b', []));
```

如果 `value` 是空字符串或空 `Buffer`，则将返回 `byteOffset`。

### `buf.length`

<!-- YAML
added: v0.1.90
-->

* 类型：{integer}

返回 `buf` 中的字节数。

```mjs
import { Buffer } from 'node:buffer';

// 创建一个 `Buffer` 并使用 UTF-8 向其写入较短的字符串。

const buf = Buffer.alloc(1234);

console.log(buf.length);
// 打印: 1234

buf.write('some string', 0, 'utf8');

console.log(buf.length);
// 打印: 1234
```

```cjs
const { Buffer } = require('node:buffer');

// 创建一个 `Buffer` 并使用 UTF-8 向其写入较短的字符串。

const buf = Buffer.alloc(1234);

console.log(buf.length);
// 打印: 1234

buf.write('some string', 0, 'utf8');

console.log(buf.length);
// 打印: 1234
```

### `buf.parent`

<!-- YAML
deprecated: v8.0.0
-->

> Stability: 0 - 已弃用：改用 [`buf.buffer`][]。

`buf.parent` 属性是 `buf.buffer` 的已弃用别名。

### `buf.readBigInt64BE([offset])`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足：`0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{bigint}

从 `buf` 中指定的 `offset` 读取有符号的大端序 64 位整数。

从 `Buffer` 读取的整数被解释为二进制补码有符号值。

### `buf.readBigInt64LE([offset])`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足：`0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{bigint}

从 `buf` 中指定的 `offset` 读取有符号的小端序 64 位整数。

从 `Buffer` 读取的整数被解释为二进制补码有符号值。

### `buf.readBigUInt64BE([offset])`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
changes:
  - version:
    - v14.10.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34960
    description: This function is also available as `buf.readBigUint64BE()`.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足：`0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{bigint}

从 `buf` 中指定的 `offset` 读取无符号的大端序 64 位整数。

此函数也可用作 `readBigUint64BE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x00, 0x00, 0x00, 0x00, 0xff, 0xff, 0xff, 0xff]);

console.log(buf.readBigUInt64BE(0));
// 打印: 4294967295n
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x00, 0x00, 0x00, 0x00, 0xff, 0xff, 0xff, 0xff]);

console.log(buf.readBigUInt64BE(0));
// 打印: 4294967295n
```

### `buf.readBigUInt64LE([offset])`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
changes:
  - version:
    - v14.10.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34960
    description: This function is also available as `buf.readBigUint64LE()`.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足：`0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{bigint}

从 `buf` 中指定的 `offset` 读取无符号的小端序 64 位整数。

此函数也可用作 `readBigUint64LE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x00, 0x00, 0x00, 0x00, 0xff, 0xff, 0xff, 0xff]);

console.log(buf.readBigUInt64LE(0));
// 打印: 18446744069414584320n
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x00, 0x00, 0x00, 0x00, 0xff, 0xff, 0xff, 0xff]);

console.log(buf.readBigUInt64LE(0));
// 打印: 18446744069414584320n
```

### `buf.readDoubleBE([offset])`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{number}

从 `buf` 中指定的 `offset` 读取 64 位大端序双精度浮点数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([1, 2, 3, 4, 5, 6, 7, 8]);

console.log(buf.readDoubleBE(0));
// 打印: 8.20788039913184e-304
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([1, 2, 3, 4, 5, 6, 7, 8]);

console.log(buf.readDoubleBE(0));
// 打印: 8.20788039913184e-304
```

### `buf.readDoubleLE([offset])`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{number}

从 `buf` 中指定的 `offset` 读取 64 位小端序双精度浮点数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([1, 2, 3, 4, 5, 6, 7, 8]);

console.log(buf.readDoubleLE(0));
// 打印: 5.447603722011605e-270
console.log(buf.readDoubleLE(1));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([1, 2, 3, 4, 5, 6, 7, 8]);

console.log(buf.readDoubleLE(0));
// 打印: 5.447603722011605e-270
console.log(buf.readDoubleLE(1));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readFloatBE([offset])`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{number}

从 `buf` 中指定的 `offset` 读取 32 位大端序浮点数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([1, 2, 3, 4]);

console.log(buf.readFloatBE(0));
// 打印: 2.387939260590663e-38
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([1, 2, 3, 4]);

console.log(buf.readFloatBE(0));
// 打印: 2.387939260590663e-38
```

### `buf.readFloatLE([offset])`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{number}

从 `buf` 中指定的 `offset` 读取 32 位小端序浮点数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([1, 2, 3, 4]);

console.log(buf.readFloatLE(0));
// 打印: 1.539989614439558e-36
console.log(buf.readFloatLE(1));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([1, 2, 3, 4]);

console.log(buf.readFloatLE(0));
// 打印: 1.539989614439558e-36
console.log(buf.readFloatLE(1));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readInt8([offset])`

<!-- YAML
added: v0.5.0
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 1`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取有符号的 8 位整数。

从 `Buffer` 读取的整数被解释为二进制补码有符号值。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([-1, 5]);

console.log(buf.readInt8(0));
// 打印: -1
console.log(buf.readInt8(1));
// 打印: 5
console.log(buf.readInt8(2));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([-1, 5]);

console.log(buf.readInt8(0));
// 打印: -1
console.log(buf.readInt8(1));
// 打印: 5
console.log(buf.readInt8(2));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readInt16BE([offset])`

<!-- YAML
added: v0.5.5
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 2`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取有符号的大端序 16 位整数。

从 `Buffer` 读取的整数被解释为二进制补码有符号值。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0, 5]);

console.log(buf.readInt16BE(0));
// 打印: 5
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0, 5]);

console.log(buf.readInt16BE(0));
// 打印: 5
```

### `buf.readInt16LE([offset])`

<!-- YAML
added: v0.5.5
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 2`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取有符号的小端序 16 位整数。

从 `Buffer` 读取的整数被解释为二进制补码有符号值。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0, 5]);

console.log(buf.readInt16LE(0));
// 打印: 1280
console.log(buf.readInt16LE(1));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0, 5]);

console.log(buf.readInt16LE(0));
// 打印: 1280
console.log(buf.readInt16LE(1));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readInt32BE([offset])`

<!-- YAML
added: v0.5.5
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取有符号的大端序 32 位整数。

从 `Buffer` 读取的整数被解释为二进制补码有符号值。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0, 0, 0, 5]);

console.log(buf.readInt32BE(0));
// 打印: 5
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0, 0, 0, 5]);

console.log(buf.readInt32BE(0));
// 打印: 5
```

### `buf.readInt32LE([offset])`

<!-- YAML
added: v0.5.5
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取有符号的小端序 32 位整数。

从 `Buffer` 读取的整数被解释为二进制补码有符号值。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0, 0, 0, 5]);

console.log(buf.readInt32LE(0));
// 打印: 83886080
console.log(buf.readInt32LE(1));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0, 0, 0, 5]);

console.log(buf.readInt32LE(0));
// 打印: 83886080
console.log(buf.readInt32LE(1));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readIntBE(offset, byteLength)`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 and `byteLength` to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - byteLength`。
* `byteLength` {integer} 要读取的字节数。必须满足 `0 < byteLength <= 6`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取 `byteLength` 个字节，并将结果解释为支持高达 48 位精度的大端序二进制补码有符号值。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

console.log(buf.readIntBE(0, 6).toString(16));
// 打印: 1234567890ab
console.log(buf.readIntBE(1, 6).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
console.log(buf.readIntBE(1, 0).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

console.log(buf.readIntBE(0, 6).toString(16));
// 打印: 1234567890ab
console.log(buf.readIntBE(1, 6).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
console.log(buf.readIntBE(1, 0).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readIntLE(offset, byteLength)`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 and `byteLength` to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - byteLength`。
* `byteLength` {integer} 要读取的字节数。必须满足 `0 < byteLength <= 6`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取 `byteLength` 个字节，并将结果解释为支持高达 48 位精度的小端序二进制补码有符号值。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

console.log(buf.readIntLE(0, 6).toString(16));
// 打印: -546f87a9cbee
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

console.log(buf.readIntLE(0, 6).toString(16));
// 打印: -546f87a9cbee
```

### `buf.readUInt8([offset])`

<!-- YAML
added: v0.5.0
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.readUint8()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 1`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取无符号的 8 位整数。

此函数也可用作 `readUint8` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([1, -2]);

console.log(buf.readUInt8(0));
// 打印: 1
console.log(buf.readUInt8(1));
// 打印: 254
console.log(buf.readUInt8(2));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([1, -2]);

console.log(buf.readUInt8(0));
// 打印: 1
console.log(buf.readUInt8(1));
// 打印: 254
console.log(buf.readUInt8(2));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readUInt16BE([offset])`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.readUint16BE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 2`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取无符号的大端序 16 位整数。

此函数也可用作 `readUint16BE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x12, 0x34, 0x56]);

console.log(buf.readUInt16BE(0).toString(16));
// 打印: 1234
console.log(buf.readUInt16BE(1).toString(16));
// 打印: 3456
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x12, 0x34, 0x56]);

console.log(buf.readUInt16BE(0).toString(16));
// 打印: 1234
console.log(buf.readUInt16BE(1).toString(16));
// 打印: 3456
```

### `buf.readUInt16LE([offset])`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.readUint16LE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 2`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取无符号的小端序 16 位整数。

此函数也可用作 `readUint16LE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x12, 0x34, 0x56]);

console.log(buf.readUInt16LE(0).toString(16));
// 打印: 3412
console.log(buf.readUInt16LE(1).toString(16));
// 打印: 5634
console.log(buf.readUInt16LE(2).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x12, 0x34, 0x56]);

console.log(buf.readUInt16LE(0).toString(16));
// 打印: 3412
console.log(buf.readUInt16LE(1).toString(16));
// 打印: 5634
console.log(buf.readUInt16LE(2).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readUInt32BE([offset])`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.readUint32BE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取无符号的大端序 32 位整数。

此函数也可用作 `readUint32BE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78]);

console.log(buf.readUInt32BE(0).toString(16));
// 打印: 12345678
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78]);

console.log(buf.readUInt32BE(0).toString(16));
// 打印: 12345678
```

### `buf.readUInt32LE([offset])`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.readUint32LE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取无符号的小端序 32 位整数。

此函数也可用作 `readUint32LE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78]);

console.log(buf.readUInt32LE(0).toString(16));
// 打印: 78563412
console.log(buf.readUInt32LE(1).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78]);

console.log(buf.readUInt32LE(0).toString(16));
// 打印: 78563412
console.log(buf.readUInt32LE(1).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readUIntBE(offset, byteLength)`

<!-- YAML
added: v0.11.15
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.readUintBE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 and `byteLength` to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - byteLength`。
* `byteLength` {integer} 要读取的字节数。必须满足 `0 < byteLength <= 6`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取 `byteLength` 个字节，并将结果解释为支持高达 48 位精度的无符号大端序整数。

此函数也可用作 `readUintBE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

console.log(buf.readUIntBE(0, 6).toString(16));
// 打印: 1234567890ab
console.log(buf.readUIntBE(1, 6).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

console.log(buf.readUIntBE(0, 6).toString(16));
// 打印: 1234567890ab
console.log(buf.readUIntBE(1, 6).toString(16));
// 抛出 ERR_OUT_OF_RANGE。
```

### `buf.readUIntLE(offset, byteLength)`

<!-- YAML
added: v0.11.15
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.readUintLE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 and `byteLength` to `uint32` anymore.
-->

* `offset` {integer} 在开始读取之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - byteLength`。
* `byteLength` {integer} 要读取的字节数。必须满足 `0 < byteLength <= 6`。
* 返回：{integer}

从 `buf` 中指定的 `offset` 读取 `byteLength` 个字节，并将结果解释为支持高达 48 位精度的无符号小端序整数。

此函数也可用作 `readUintLE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

console.log(buf.readUIntLE(0, 6).toString(16));
// 打印: ab9078563412
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x12, 0x34, 0x56, 0x78, 0x90, 0xab]);

console.log(buf.readUIntLE(0, 6).toString(16));
// 打印: ab9078563412
```

### `buf.subarray([start[, end]])`

<!-- YAML
added: v3.0.0
-->

* `start` {integer} 新 `Buffer` 将开始的位置。**默认值：** `0`。
* `end` {integer} 新 `Buffer` 将结束的位置（不包括）。
  **默认值：** [`buf.length`][]。
* 返回：{Buffer}

返回一个新的 `Buffer`，它引用与原始缓冲区相同的内存，但由 `start` 和 `end` 索引偏移和裁剪。

指定 `end` 大于 [`buf.length`][] 将返回与 `end` 等于 [`buf.length`][] 相同的结果。

此方法继承自 [`TypedArray.prototype.subarray()`][]。

修改新的 `Buffer` 切片将修改原始 `Buffer` 中的内存，因为两个对象的分配内存重叠。

```mjs
import { Buffer } from 'node:buffer';

// 创建一个带有 ASCII 字母表的 `Buffer`，取一个切片，并修改原始 `Buffer` 中的一个字节。

const buf1 = Buffer.allocUnsafe(26);

for (let i = 0; i < 26; i++) {
  // 97 是 'a' 的十进制 ASCII 值。
  buf1[i] = i + 97;
}

const buf2 = buf1.subarray(0, 3);

console.log(buf2.toString('ascii', 0, buf2.length));
// 打印: abc

buf1[0] = 33;

console.log(buf2.toString('ascii', 0, buf2.length));
// 打印: !bc
```

```cjs
const { Buffer } = require('node:buffer');

// 创建一个带有 ASCII 字母表的 `Buffer`，取一个切片，并修改原始 `Buffer` 中的一个字节。

const buf1 = Buffer.allocUnsafe(26);

for (let i = 0; i < 26; i++) {
  // 97 是 'a' 的十进制 ASCII 值。
  buf1[i] = i + 97;
}

const buf2 = buf1.subarray(0, 3);

console.log(buf2.toString('ascii', 0, buf2.length));
// 打印: abc

buf1[0] = 33;

console.log(buf2.toString('ascii', 0, buf2.length));
// 打印: !bc
```

指定负索引会导致切片相对于 `buf` 的末尾而不是开头生成。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('buffer');

console.log(buf.subarray(-6, -1).toString());
// 打印: buffe
// (等同于 buf.subarray(0, 5)。)

console.log(buf.subarray(-6, -2).toString());
// 打印: buff
// (等同于 buf.subarray(0, 4)。)

console.log(buf.subarray(-5, -2).toString());
// 打印: uff
// (等同于 buf.subarray(1, 4)。)
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('buffer');

console.log(buf.subarray(-6, -1).toString());
// 打印: buffe
// (等同于 buf.subarray(0, 5)。)

console.log(buf.subarray(-6, -2).toString());
// 打印: buff
// (等同于 buf.subarray(0, 4)。)

console.log(buf.subarray(-5, -2).toString());
// 打印: uff
// (等同于 buf.subarray(1, 4)。)
```

### `buf.slice([start[, end]])`

<!-- YAML
added: v0.3.0
changes:
  - version:
    - v17.5.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/41596
    description: The buf.slice() method has been deprecated.
  - version:
    - v7.1.0
    - v6.9.2
    pr-url: https://github.com/nodejs/node/pull/9341
    description: Coercing the offsets to integers now handles values outside
                 the 32-bit integer range properly.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/9101
    description: All offsets are now coerced to integers before doing any
                 calculations with them.
-->

* `start` {integer} 新 `Buffer` 将开始的位置。**默认值：** `0`。
* `end` {integer} 新 `Buffer` 将结束的位置（不包括）。
  **默认值：** [`buf.length`][]。
* 返回：{Buffer}

> Stability: 0 - 已弃用：改用 [`buf.subarray`][]。

返回一个新的 `Buffer`，它引用与原始缓冲区相同的内存，但由 `start` 和 `end` 索引偏移和裁剪。

此方法与 `Uint8Array.prototype.slice()` 不兼容，后者是 `Buffer` 的超类。要复制切片，请使用 `Uint8Array.prototype.slice()`。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('buffer');

const copiedBuf = Uint8Array.prototype.slice.call(buf);
copiedBuf[0]++;
console.log(copiedBuf.toString());
// 打印: cuffer

console.log(buf.toString());
// 打印: buffer

// 使用 buf.slice()，原始缓冲区会被修改。
const notReallyCopiedBuf = buf.slice();
notReallyCopiedBuf[0]++;
console.log(notReallyCopiedBuf.toString());
// 打印: cuffer
console.log(buf.toString());
// 也打印: cuffer (!)
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('buffer');

const copiedBuf = Uint8Array.prototype.slice.call(buf);
copiedBuf[0]++;
console.log(copiedBuf.toString());
// 打印: cuffer

console.log(buf.toString());
// 打印: buffer

// 使用 buf.slice()，原始缓冲区会被修改。
const notReallyCopiedBuf = buf.slice();
notReallyCopiedBuf[0]++;
console.log(notReallyCopiedBuf.toString());
// 打印: cuffer
console.log(buf.toString());
// 也打印: cuffer (!)
```

### `buf.swap16()`

<!-- YAML
added: v5.10.0
-->

* 返回：{Buffer} 对 `buf` 的引用。

将 `buf` 解释为无符号 16 位整数数组，并就地交换字节顺序。如果 [`buf.length`][] 不是 2 的倍数，则抛出 [`ERR_INVALID_BUFFER_SIZE`][]。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8]);

console.log(buf1);
// 打印: <Buffer 01 02 03 04 05 06 07 08>

buf1.swap16();

console.log(buf1);
// 打印: <Buffer 02 01 04 03 06 05 08 07>

const buf2 = Buffer.from([0x1, 0x2, 0x3]);

buf2.swap16();
// 抛出 ERR_INVALID_BUFFER_SIZE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8]);

console.log(buf1);
// 打印: <Buffer 01 02 03 04 05 06 07 08>

buf1.swap16();

console.log(buf1);
// 打印: <Buffer 02 01 04 03 06 05 08 07>

const buf2 = Buffer.from([0x1, 0x2, 0x3]);

buf2.swap16();
// 抛出 ERR_INVALID_BUFFER_SIZE。
```

`buf.swap16()` 的一个方便用法是在 UTF-16 小端序和 UTF-16 大端序之间执行快速就地转换：

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('This is little-endian UTF-16', 'utf16le');
buf.swap16(); // 转换为大端序 UTF-16 文本。
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('This is little-endian UTF-16', 'utf16le');
buf.swap16(); // 转换为大端序 UTF-16 文本。
```

### `buf.swap32()`

<!-- YAML
added: v5.10.0
-->

* 返回：{Buffer} 对 `buf` 的引用。

将 `buf` 解释为无符号 32 位整数数组，并就地交换字节顺序。如果 [`buf.length`][] 不是 4 的倍数，则抛出 [`ERR_INVALID_BUFFER_SIZE`][]。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8]);

console.log(buf1);
// 打印: <Buffer 01 02 03 04 05 06 07 08>

buf1.swap32();

console.log(buf1);
// 打印: <Buffer 04 03 02 01 08 07 06 05>

const buf2 = Buffer.from([0x1, 0x2, 0x3]);

buf2.swap32();
// 抛出 ERR_INVALID_BUFFER_SIZE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8]);

console.log(buf1);
// 打印: <Buffer 01 02 03 04 05 06 07 08>

buf1.swap32();

console.log(buf1);
// 打印: <Buffer 04 03 02 01 08 07 06 05>

const buf2 = Buffer.from([0x1, 0x2, 0x3]);

buf2.swap32();
// 抛出 ERR_INVALID_BUFFER_SIZE。
```

### `buf.swap64()`

<!-- YAML
added: v6.3.0
-->

* 返回：{Buffer} 对 `buf` 的引用。

将 `buf` 解释为 64 位数字数组，并就地交换字节顺序。如果 [`buf.length`][] 不是 8 的倍数，则抛出 [`ERR_INVALID_BUFFER_SIZE`][]。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8]);

console.log(buf1);
// 打印: <Buffer 01 02 03 04 05 06 07 08>

buf1.swap64();

console.log(buf1);
// 打印: <Buffer 08 07 06 05 04 03 02 01>

const buf2 = Buffer.from([0x1, 0x2, 0x3]);

buf2.swap64();
// 抛出 ERR_INVALID_BUFFER_SIZE。
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5, 0x6, 0x7, 0x8]);

console.log(buf1);
// 打印: <Buffer 01 02 03 04 05 06 07 08>

buf1.swap64();

console.log(buf1);
// 打印: <Buffer 08 07 06 05 04 03 02 01>

const buf2 = Buffer.from([0x1, 0x2, 0x3]);

buf2.swap64();
// 抛出 ERR_INVALID_BUFFER_SIZE。
```

### `buf.toJSON()`

<!-- YAML
added: v0.9.2
-->

* 返回：{Object}

返回 `buf` 的 JSON 表示。 [`JSON.stringify()`][] 在字符串化 `Buffer` 实例时隐式调用此函数。

`Buffer.from()` 接受从此方法返回的格式的对象。特别是，`Buffer.from(buf.toJSON())` 的工作方式类似于 `Buffer.from(buf)`。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5]);
const json = JSON.stringify(buf);

console.log(json);
// 打印: {"type":"Buffer","data":[1,2,3,4,5]}

const copy = JSON.parse(json, (key, value) => {
  return value && value.type === 'Buffer' ?
    Buffer.from(value) :
    value;
});

console.log(copy);
// 打印: <Buffer 01 02 03 04 05>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([0x1, 0x2, 0x3, 0x4, 0x5]);
const json = JSON.stringify(buf);

console.log(json);
// 打印: {"type":"Buffer","data":[1,2,3,4,5]}

const copy = JSON.parse(json, (key, value) => {
  return value && value.type === 'Buffer' ?
    Buffer.from(value) :
    value;
});

console.log(copy);
// 打印: <Buffer 01 02 03 04 05>
```

### `buf.toString([encoding[, start[, end]]])`

<!-- YAML
added: v0.1.90
-->

* `encoding` {string} 要使用的字符编码。**默认值：** `'utf8'`。
* `start` {integer} 开始解码的字节偏移量。**默认值：** `0`。
* `end` {integer} 停止解码的字节偏移量（不包括）。
  **默认值：** [`buf.length`][]。
* 返回：{string}

根据 `encoding` 中指定的字符编码将 `buf` 解码为字符串。可以传递 `start` 和 `end` 以仅解码 `buf` 的子集。

如果 `encoding` 是 `'utf8'` 并且输入中的字节序列不是有效的 UTF-8，则每个无效字节将被替换字符 `U+FFFD` � 替换。

字符串实例的最大长度（以 UTF-16 代码单元计）可用作 [`buffer.constants.MAX_STRING_LENGTH`][]。

```mjs
import { Buffer } from 'node:buffer';

const buf1 = Buffer.allocUnsafe(26);

for (let i = 0; i < 26; i++) {
  // 97 是 'a' 的十进制 ASCII 值。
  buf1[i] = i + 97;
}

console.log(buf1.toString('utf8'));
// 打印: abcdefghijklmnopqrstuvwxyz
console.log(buf1.toString('utf8', 0, 5));
// 打印: abcde

const buf2 = Buffer.from('tést');

console.log(buf2.toString('hex'));
// 打印: 74c3a97374
console.log(buf2.toString('utf8', 0, 3));
// 打印: té
console.log(buf2.toString(undefined, 0, 3));
// 打印: té
```

```cjs
const { Buffer } = require('node:buffer');

const buf1 = Buffer.allocUnsafe(26);

for (let i = 0; i < 26; i++) {
  // 97 是 'a' 的十进制 ASCII 值。
  buf1[i] = i + 97;
}

console.log(buf1.toString('utf8'));
// 打印: abcdefghijklmnopqrstuvwxyz
console.log(buf1.toString('utf8', 0, 5));
// 打印: abcde

const buf2 = Buffer.from('tést');

console.log(buf2.toString('hex'));
// 打印: 74c3a97374
console.log(buf2.toString('utf8', 0, 3));
// 打印: té
console.log(buf2.toString(undefined, 0, 3));
// 打印: té
```

### `buf.values()`

<!-- YAML
added: v1.1.0
-->

* 返回：{Iterator}

为 `buf` 值（字节）创建并返回一个[迭代器][]。当在 `for..of` 语句中使用 `Buffer` 时，会自动调用此函数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('buffer');

for (const value of buf.values()) {
  console.log(value);
}
// 打印:
//   98
//   117
//   102
//   102
//   101
//   114

for (const value of buf) {
  console.log(value);
}
// 打印:
//   98
//   117
//   102
//   102
//   101
//   114
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('buffer');

for (const value of buf.values()) {
  console.log(value);
}
// 打印:
//   98
//   117
//   102
//   102
//   101
//   114

for (const value of buf) {
  console.log(value);
}
// 打印:
//   98
//   117
//   102
//   102
//   101
//   114
```

### `buf.write(string[, offset[, length]][, encoding])`

<!-- YAML
added: v0.1.90
-->

* `string` {string} 要写入 `buf` 的字符串。
* `offset` {integer} 在开始写入 `string` 之前要跳过的字节数。
  **默认值：** `0`。
* `length` {integer} 要写入的最大字节数（写入的字节数不会超过 `buf.length - offset`）。**默认值：** `buf.length - offset`。
* `encoding` {string} `string` 的字符编码。**默认值：** `'utf8'`。
* 返回：{integer} 写入的字节数。

在 `offset` 处根据 `encoding` 中的字符编码将 `string` 写入 `buf`。`length` 参数是要写入的字节数。如果 `buf` 没有足够的空间来容纳整个字符串，则只会写入 `string` 的一部分。但是，不会写入部分编码的字符。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.alloc(256);

const len = buf.write('\u00bd + \u00bc = \u00be', 0);

console.log(`${len} 个字节: ${buf.toString('utf8', 0, len)}`);
// 打印: 12 个字节: ½ + ¼ = ¾

const buffer = Buffer.alloc(10);

const length = buffer.write('abcd', 8);

console.log(`${length} 个字节: ${buffer.toString('utf8', 8, 10)}`);
// 打印: 2 个字节 : ab
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.alloc(256);

const len = buf.write('\u00bd + \u00bc = \u00be', 0);

console.log(`${len} 个字节: ${buf.toString('utf8', 0, len)}`);
// 打印: 12 个字节: ½ + ¼ = ¾

const buffer = Buffer.alloc(10);

const length = buffer.write('abcd', 8);

console.log(`${length} 个字节: ${buffer.toString('utf8', 8, 10)}`);
// 打印: 2 个字节 : ab
```

### `buf.writeBigInt64BE(value[, offset])`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
-->

* `value` {bigint} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足：`0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以大端序将 `value` 写入 `buf` 中指定的 `offset`。

`value` 被解释并写入为二进制补码有符号整数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(8);

buf.writeBigInt64BE(0x0102030405060708n, 0);

console.log(buf);
// 打印: <Buffer 01 02 03 04 05 06 07 08>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(8);

buf.writeBigInt64BE(0x0102030405060708n, 0);

console.log(buf);
// 打印: <Buffer 01 02 03 04 05 06 07 08>
```

### `buf.writeBigInt64LE(value[, offset])`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
-->

* `value` {bigint} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足：`0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以小端序将 `value` 写入 `buf` 中指定的 `offset`。

`value` 被解释并写入为二进制补码有符号整数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(8);

buf.writeBigInt64LE(0x0102030405060708n, 0);

console.log(buf);
// 打印: <Buffer 08 07 06 05 04 03 02 01>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(8);

buf.writeBigInt64LE(0x0102030405060708n, 0);

console.log(buf);
// 打印: <Buffer 08 07 06 05 04 03 02 01>
```

### `buf.writeBigUInt64BE(value[, offset])`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
changes:
  - version:
    - v14.10.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34960
    description: This function is also available as `buf.writeBigUint64BE()`.
-->

* `value` {bigint} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足：`0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以大端序将 `value` 写入 `buf` 中指定的 `offset`。

此函数也可用作 `writeBigUint64BE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(8);

buf.writeBigUInt64BE(0xdecafafecacefaden, 0);

console.log(buf);
// 打印: <Buffer de ca fa fe ca ce fa de>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(8);

buf.writeBigUInt64BE(0xdecafafecacefaden, 0);

console.log(buf);
// 打印: <Buffer de ca fa fe ca ce fa de>
```

### `buf.writeBigUInt64LE(value[, offset])`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
changes:
  - version:
    - v14.10.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34960
    description: This function is also available as `buf.writeBigUint64LE()`.
-->

* `value` {bigint} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足：`0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以小端序将 `value` 写入 `buf` 中指定的 `offset`。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(8);

buf.writeBigUInt64LE(0xdecafafecacefaden, 0);

console.log(buf);
// 打印: <Buffer de fa ce ca fe fa ca de>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(8);

buf.writeBigUInt64LE(0xdecafafecacefaden, 0);

console.log(buf);
// 打印: <Buffer de fa ce ca fe fa ca de>
```

此函数也可用作 `writeBigUint64LE` 别名。

### `buf.writeDoubleBE(value[, offset])`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {number} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的 64 位双精度浮点数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(8);

buf.writeDoubleBE(123.456, 0);

console.log(buf);
// 打印: <Buffer 40 5e dd 2f 1a 9f be 77>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(8);

buf.writeDoubleBE(123.456, 0);

console.log(buf);
// 打印: <Buffer 40 5e dd 2f 1a 9f be 77>
```

### `buf.writeDoubleLE(value[, offset])`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {number} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 8`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的 64 位双精度浮点数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(8);

buf.writeDoubleLE(123.456, 0);

console.log(buf);
// 打印: <Buffer 77 be 9f 1a 2f dd 5e 40>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(8);

buf.writeDoubleLE(123.456, 0);

console.log(buf);
// 打印: <Buffer 77 be 9f 1a 2f dd 5e 40>
```

### `buf.writeFloatBE(value[, offset])`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {number} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的 32 位浮点数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(4);

buf.writeFloatBE(0xcafebabe, 0);

console.log(buf);
// 打印: <Buffer 4f 4a fe bb>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(4);

buf.writeFloatBE(0xcafebabe, 0);

console.log(buf);
// 打印: <Buffer 4f 4a fe bb>
```

### `buf.writeFloatLE(value[, offset])`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {number} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的 32 位浮点数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(4);

buf.writeFloatLE(0xcafebabe, 0);

console.log(buf);
// 打印: <Buffer bb fe 4a 4f>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(4);

buf.writeFloatLE(0xcafebabe, 0);

console.log(buf);
// 打印: <Buffer bb fe 4a 4f>
```

### `buf.writeInt8(value[, offset])`

<!-- YAML
added: v0.5.0
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 1`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的有符号 8 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

`value` 被解释并写入为二进制补码有符号整数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(2);

buf.writeInt8(2, 0);
buf.writeInt8(-2, 1);

console.log(buf);
// 打印: <Buffer 02 fe>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(2);

buf.writeInt8(2, 0);
buf.writeInt8(-2, 1);

console.log(buf);
// 打印: <Buffer 02 fe>
```

### `buf.writeInt16BE(value[, offset])`

<!-- YAML
added: v0.5.5
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 2`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以大端序将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的有符号 16 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

`value` 被解释并写入为二进制补码有符号整数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(2);

buf.writeInt16BE(0x0102, 0);

console.log(buf);
// 打印: <Buffer 01 02>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(2);

buf.writeInt16BE(0x0102, 0);

console.log(buf);
// 打印: <Buffer 01 02>
```

### `buf.writeInt16LE(value[, offset])`

<!-- YAML
added: v0.5.5
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 2`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以小端序将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的有符号 16 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

`value` 被解释并写入为二进制补码有符号整数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(2);

buf.writeInt16LE(0x0304, 0);

console.log(buf);
// 打印: <Buffer 04 03>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(2);

buf.writeInt16LE(0x0304, 0);

console.log(buf);
// 打印: <Buffer 04 03>
```

### `buf.writeInt32BE(value[, offset])`

<!-- YAML
added: v0.5.5
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以大端序将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的有符号 32 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

`value` 被解释并写入为二进制补码有符号整数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(4);

buf.writeInt32BE(0x01020304, 0);

console.log(buf);
// 打印: <Buffer 01 02 03 04>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(4);

buf.writeInt32BE(0x01020304, 0);

console.log(buf);
// 打印: <Buffer 01 02 03 04>
```

### `buf.writeInt32LE(value[, offset])`

<!-- YAML
added: v0.5.5
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以小端序将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的有符号 32 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

`value` 被解释并写入为二进制补码有符号整数。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(4);

buf.writeInt32LE(0x05060708, 0);

console.log(buf);
// 打印: <Buffer 08 07 06 05>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(4);

buf.writeInt32LE(0x05060708, 0);

console.log(buf);
// 打印: <Buffer 08 07 06 05>
```

### `buf.writeIntBE(value, offset, byteLength)`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 and `byteLength` to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - byteLength`。
* `byteLength` {integer} 要写入的字节数。必须满足 `0 < byteLength <= 6`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `byteLength` 个字节的 `value` 写入 `buf` 中指定的 `offset`。支持高达 48 位精度。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(6);

buf.writeIntBE(0x1234567890ab, 0, 6);

console.log(buf);
// 打印: <Buffer 12 34 56 78 90 ab>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(6);

buf.writeIntBE(0x1234567890ab, 0, 6);

console.log(buf);
// 打印: <Buffer 12 34 56 78 90 ab>
```

### `buf.writeIntLE(value, offset, byteLength)`

<!-- YAML
added: v0.11.15
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 and `byteLength` to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - byteLength`。
* `byteLength` {integer} 要写入的字节数。必须满足 `0 < byteLength <= 6`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `byteLength` 个字节的 `value` 写入 `buf` 中指定的 `offset`。支持高达 48 位精度。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(6);

buf.writeIntLE(0x1234567890ab, 0, 6);

console.log(buf);
// 打印: <Buffer ab 90 78 56 34 12>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(6);

buf.writeIntLE(0x1234567890ab, 0, 6);

console.log(buf);
// 打印: <Buffer ab 90 78 56 34 12>
```

### `buf.writeUInt8(value[, offset])`

<!-- YAML
added: v0.5.0
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.writeUint8()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 1`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的无符号 8 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

此函数也可用作 `writeUint8` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(4);

buf.writeUInt8(0x3, 0);
buf.writeUInt8(0x4, 1);
buf.writeUInt8(0x23, 2);
buf.writeUInt8(0x42, 3);

console.log(buf);
// 打印: <Buffer 03 04 23 42>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(4);

buf.writeUInt8(0x3, 0);
buf.writeUInt8(0x4, 1);
buf.writeUInt8(0x23, 2);
buf.writeUInt8(0x42, 3);

console.log(buf);
// 打印: <Buffer 03 04 23 42>
```

### `buf.writeUInt16BE(value[, offset])`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.writeUint16BE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 2`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以大端序将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的无符号 16 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

此函数也可用作 `writeUint16BE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(4);

buf.writeUInt16BE(0xdead, 0);
buf.writeUInt16BE(0xbeef, 2);

console.log(buf);
// 打印: <Buffer de ad be ef>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(4);

buf.writeUInt16BE(0xdead, 0);
buf.writeUInt16BE(0xbeef, 2);

console.log(buf);
// 打印: <Buffer de ad be ef>
```

### `buf.writeUInt16LE(value[, offset])`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.writeUint16LE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 2`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以小端序将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的无符号 16 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

此函数也可用作 `writeUint16LE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(4);

buf.writeUInt16LE(0xdead, 0);
buf.writeUInt16LE(0xbeef, 2);

console.log(buf);
// 打印: <Buffer ad de ef be>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(4);

buf.writeUInt16LE(0xdead, 0);
buf.writeUInt16LE(0xbeef, 2);

console.log(buf);
// 打印: <Buffer ad de ef be>
```

### `buf.writeUInt32BE(value[, offset])`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.writeUint32BE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以大端序将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的无符号 32 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

此函数也可用作 `writeUint32BE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(4);

buf.writeUInt32BE(0xfeedface, 0);

console.log(buf);
// 打印: <Buffer fe ed fa ce>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(4);

buf.writeUInt32BE(0xfeedface, 0);

console.log(buf);
// 打印: <Buffer fe ed fa ce>
```

### `buf.writeUInt32LE(value[, offset])`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.writeUint32LE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - 4`。**默认值：** `0`。
* 返回：{integer} `offset` 加上写入的字节数。

以小端序将 `value` 写入 `buf` 中指定的 `offset`。`value` 必须是有效的无符号 32 位整数。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

此函数也可用作 `writeUint32LE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(4);

buf.writeUInt32LE(0xfeedface, 0);

console.log(buf);
// 打印: <Buffer ce fa ed fe>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(4);

buf.writeUInt32LE(0xfeedface, 0);

console.log(buf);
// 打印: <Buffer ce fa ed fe>
```

### `buf.writeUIntBE(value, offset, byteLength)`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.writeUintBE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 and `byteLength` to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - byteLength`。
* `byteLength` {integer} 要写入的字节数。必须满足 `0 < byteLength <= 6`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `byteLength` 个字节的 `value` 写入 `buf` 中指定的 `offset`。支持高达 48 位精度。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

此函数也可用作 `writeUintBE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(6);

buf.writeUIntBE(0x1234567890ab, 0, 6);

console.log(buf);
// 打印: <Buffer 12 34 56 78 90 ab>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(6);

buf.writeUIntBE(0x1234567890ab, 0, 6);

console.log(buf);
// 打印: <Buffer 12 34 56 78 90 ab>
```

### `buf.writeUIntLE(value, offset, byteLength)`

<!-- YAML
added: v0.5.5
changes:
  - version:
    - v14.9.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34729
    description: This function is also available as `buf.writeUintLE()`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18395
    description: Removed `noAssert` and no implicit coercion of the offset
                 and `byteLength` to `uint32` anymore.
-->

* `value` {integer} 要写入 `buf` 的数字。
* `offset` {integer} 在开始写入之前要跳过的字节数。必须满足 `0 <= offset <= buf.length - byteLength`。
* `byteLength` {integer} 要写入的字节数。必须满足 `0 < byteLength <= 6`。
* 返回：{integer} `offset` 加上写入的字节数。

将 `byteLength` 个字节的 `value` 写入 `buf` 中指定的 `offset`。支持高达 48 位精度。当 `value` 是 JavaScript 数字以外的任何值时，行为未定义。

此函数也可用作 `writeUintLE` 别名。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.allocUnsafe(6);

buf.writeUIntLE(0x1234567890ab, 0, 6);

console.log(buf);
// 打印: <Buffer ab 90 78 56 34 12>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.allocUnsafe(6);

buf.writeUIntLE(0x1234567890ab, 0, 6);

console.log(buf);
// 打印: <Buffer ab 90 78 56 34 12>
```

## `buffer.INSPECT_MAX_BYTES`

<!-- YAML
added: v0.5.4
-->

* 类型：{integer} **默认值：** `50`

返回调用 `buf.inspect()` 时将返回的最大字节数。这可以被用户模块覆盖。有关 `buf.inspect()` 行为的更多详细信息，请参阅 [`util.inspect()`]。

该值是在 `require('node:buffer')` 时返回的 `buffer` 模块上的属性。通过 `import` 或 `require` 属性访问它，将返回 `Buffer` 基类。

## `buffer.kMaxLength`

<!-- YAML
added: v3.0.0
-->

* 类型：{integer} 单个 `Buffer` 实例允许的最大大小。

在 32 位架构上，此值为 `(2^30)-1`（~1GiB）。
在 64 位架构上，此值为 `(2^31)-1`（~2GiB）。

该值是在 `require('node:buffer')` 时返回的 `buffer` 模块上的属性。通过 `import` 或 `require` 属性访问它，将返回 `Buffer` 基类。

## `buffer.transcode(source, fromEnc, toEnc)`

<!-- YAML
added: v7.1.0
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10236
    description: The `source` parameter can now be a `Uint8Array`.
-->

* `source` {Buffer|Uint8Array} 一个 `Buffer` 或 `Uint8Array` 实例。
* `fromEnc` {string} 当前编码。
* `toEnc` {string} 目标编码。
* 返回：{Buffer}

将给定的 `Buffer` 或 `Uint8Array` 实例从一种字符编码重新编码为另一种。返回一个新的 `Buffer` 实例。

如果 `fromEnc` 或 `toEnc` 指定了无效的字符编码，或者从 `fromEnc` 到 `toEnc` 不允许转换，则抛出错误。

如果给定的字节序列无法用目标编码充分表示，则 `buffer.transcode()` 支持转义字符的替代字符编码将执行替换。例如：

```mjs
import { Buffer, transcode } from 'node:buffer';

const newBuf = transcode(Buffer.from('€'), 'utf8', 'ascii');
console.log(newBuf.toString('ascii'));
// 打印: '?'
```

```cjs
const { Buffer, transcode } = require('node:buffer');

const newBuf = transcode(Buffer.from('€'), 'utf8', 'ascii');
console.log(newBuf.toString('ascii'));
// 打印: '?'
```

因为欧元符号（`€`）在 US-ASCII 中无法表示，所以在转码后的 `Buffer` 中，它被替换为 `?`。

该函数是在 `require('node:buffer')` 时返回的 `buffer` 模块上的属性。通过 `import` 或 `require` 属性访问它，将返回 `Buffer` 基类。

## `Buffer` 常量

<!-- YAML
changes:
  - version:
    - v9.9.0
    - v8.17.0
    pr-url: https://github.com/nodejs/node/pull/17688
    description: Added `buffer.constants.MAX_LENGTH`.
  - version:
    - v9.9.0
    - v8.17.0
    pr-url: https://github.com/nodejs/node/pull/17688
    description: Added `buffer.constants.MAX_STRING_LENGTH`.
-->

这些是在 `buffer.constants` 上定义的，并且特定于 Node.js 的 `Buffer` 实现。它们也可通过 `Buffer.constants` 或 `buffer.constants` 获得。

### `buffer.constants.MAX_LENGTH`

<!-- YAML
added:
 - v8.2.0
 - v6.5.0
-->

* 类型：{integer} 单个 `Buffer` 实例允许的最大大小。

在 32 位架构上，此值为 `(2^30)-1`（~1GiB）。
在 64 位架构上，此值为 `(2^31)-1`（~2GiB）。

该值也可用作 [`buffer.kMaxLength`][]。

### `buffer.constants.MAX_STRING_LENGTH`

<!-- YAML
added:
 - v8.2.0
 - v6.5.0
-->

* 类型：{integer} 单个 `string` 实例允许的最大长度（以 UTF-16 代码单元计）。

表示 `string` 原语可以增长到的最大 `length`，以 UTF-16 代码单元计。

该值可能取决于正在使用的 JS 引擎。

## 类：`File`

<!-- YAML
added:
  - v20.0.0
  - v19.0.0
-->

> Stability: 1 - 实验性

[`Blob`](#class-blob) 的扩展，用于支持用户操作系统上的文件。有关更多详细信息，请参阅 [`File` Web API][]。

### `new buffer.File(fileBits, fileName[, options])`

<!-- YAML
added:
  - v20.0.0
  - v19.0.0
-->

* `fileBits` {Array} 一个包含 {ArrayBuffer}、{TypedArray}、{DataView}、{Blob}、字符串或这些类型的混合的数组，这些数据将构成文件的内容。
* `fileName` {string} 文件名。
* `options` {Object}
  * `endings` {string} `'transparent'` 或 `'native'` 之一。当设置为 `'native'` 时，字符串源部分中的行结尾将转换为 `require('node:os').EOL` 指定的平台本机行结尾。
  * `type` {string} 文件内容类型。目的是让 `type` 传达数据的 MIME 媒体类型，但不执行类型格式的验证。
  * `lastModified` {number} 文件最后修改的时间戳。**默认值：** `Date.now()`。

### `file.name`

<!-- YAML
added:
  - v20.0.0
  - v19.0.0
-->

* 类型：{string}

文件的名称。

### `file.lastModified`

<!-- YAML
added:
  - v20.0.0
  - v19.0.0
-->

* 类型：{number}

文件的最后修改时间。

## 性能注意事项

<!-- YAML
added: v0.1.90
-->

### `new Buffer(size)`

从 Node.js 8.0.0 开始，使用 `new Buffer(size)` 分配内存已被弃用。对于零填充内存，请改用 [`Buffer.alloc(size)`][`Buffer.alloc()`]。对于未初始化的内存，请改用 [`Buffer.allocUnsafe(size)`][`Buffer.allocUnsafe()`]。

`Buffer.alloc()` 和 `Buffer.allocUnsafe()` 之间的区别在于，虽然 `Buffer.alloc(size)` 和 `Buffer.alloc(size, 0)` 都会返回零填充的 `Buffer`，但 `Buffer.alloc(size, 0)` 执行此操作的速度较慢。虽然这看起来违反直觉，但它是对性能优化的结果。

当开发人员使用 `new Buffer(size)` 时，他们通常希望内存是零填充的。为了改进这一点，`Buffer.alloc(size)` 和 `Buffer.allocUnsafe(size).fill(0)` 现在是唯一明确用于零填充新 `Buffer` 分配的方法。由于 `Buffer.alloc(size, 0)` 比 `Buffer.allocUnsafe(size).fill(0)` 慢，我们鼓励开发人员在零填充不是绝对必要时使用 `Buffer.allocUnsafe(size)`。

当应用程序需要额外性能时，使用 `Buffer.allocUnsafe()` 分配未初始化的内存段可能是合适的。此类分配必须非常小心，以避免从 `Buffer` 读取未初始化的内存。

### 缓冲区和字符编码

当在 `Buffer` 和字符串之间进行转换时，可以传递字符编码。如果未指定字符编码，则默认使用 UTF-8。

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from('hello world', 'utf8');

console.log(buf.toString('hex'));
// 打印: 68656c6c6f20776f726c64
console.log(buf.toString('base64'));
// 打印: aGVsbG8gd29ybGQ=

console.log(Buffer.from('fhqwhgads', 'utf8'));
// 打印: <Buffer 66 68 71 77 68 67 61 64 73>
console.log(Buffer.from('fhqwhgads', 'utf16le'));
// 打印: <Buffer 66 00 68 00 71 00 77 00 68 00 67 00 61 00 64 00 73 00>
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from('hello world', 'utf8');

console.log(buf.toString('hex'));
// 打印: 68656c6c6f20776f726c64
console.log(buf.toString('base64'));
// 打印: aGVsbG8gd29ybGQ=

console.log(Buffer.from('fhqwhgads', 'utf8'));
// 打印: <Buffer 66 68 71 77 68 67 61 64 73>
console.log(Buffer.from('fhqwhgads', 'utf16le'));
// 打印: <Buffer 66 00 68 00 71 00 77 00 68 00 67 00 61 00 64 00 73 00>
```

虽然 Node.js 支持将其他基于 Latin-1 和 ISO-8859-1 的字符编码转换为字符串，但不鼓励使用它们。当将非 ASCII 字符转换为这些编码时，它们可能会被损坏。为了正确处理非 ASCII 字符，请使用 UTF-8。

### 缓冲区和 TypedArray

`Buffer` 实例也是 {Uint8Array} 实例。但是，与 {TypedArray} 相比，`Buffer` API 存在细微的不兼容性。例如，虽然 [`ArrayBuffer.prototype.slice()`][] 创建了 `ArrayBuffer` 的一部分的副本，但 [`Buffer.prototype.slice()`][`buf.slice()`] 在不复制的情况下在现有 `Buffer` 上创建视图。可以通过使用 `TypedArray.prototype.subarray()` 在 `Buffer` 和其他 {TypedArray} 上实现 [`Buffer.prototype.slice()`][`buf.slice()`] 的行为。

此外，虽然 `Buffer` 实例的行为类似于 {Uint8Array}，但某些方法在两种类型上的行为不同。具体来说，虽然 `Uint8Array.prototype.subarray()` 在修改时会更改原始 {TypedArray}，但 `Buffer.prototype.subarray()` 在不修改原始 `Buffer` 的情况下返回一个新的 `Buffer` 副本。

在可能的情况下，开发人员应避免使用导致 `Buffer` 和其他 {TypedArray} 之间行为差异的 API。有关更多信息，请参阅 [`Buffer` 和 `TypedArray`](#buffer-and-typedarray)。

### 缓冲区和迭代

可以使用 `for..of` 语法迭代 `Buffer` 实例：

```mjs
import { Buffer } from 'node:buffer';

const buf = Buffer.from([1, 2, 3]);

for (const b of buf) {
  console.log(b);
}
// 打印:
//   1
//   2
//   3
```

```cjs
const { Buffer } = require('node:buffer');

const buf = Buffer.from([1, 2, 3]);

for (const b of buf) {
  console.log(b);
}
// 打印:
//   1
//   2
//   3
```

此外，[`buf.values()`][]、[`buf.keys()`][] 和 [`buf.entries()`][] 方法可用于创建迭代器。

## `Buffer` 模块 API

虽然 `Buffer` 对象可作为全局对象使用，但还有其他与 `Buffer` 相关的 API 仅可通过使用 `require('node:buffer')` 访问的 `buffer` 模块获得。

### `buffer.atob(data)`

<!-- YAML
added:
  - v21.5.0
  - v20.12.0
-->

> Stability: 3 - 旧版。改用 `Buffer.from(data, 'base64')`。

* `data` {any} Base64 编码的输入字符串。

解码 `data` 从 Base64 转换为字符串。`data` 可以是任何 JavaScript 值，可以强制转换为字符串。由于此函数仅处理由 `'a'` 到 `'z'`、`'A'` 到 `'Z'`、`'0'` 到 `'9'`、`'+'`、`'/'` 和 `'='` 组成的字符，因此该函数会忽略字符串中可能存在的任何其他字符。

返回包含解码二进制数据的字符串。

此函数提供了与浏览器中的 `atob()` 函数的兼容性。有关此函数的使用和注意事项的详细信息，请参阅 [`atob` MDN 文档][]。

### `buffer.btoa(data)`

<!-- YAML
added:
  - v21.5.0
  - v20.12.0
-->

> Stability: 3 - 旧版。改用 `buf.toString('base64')`。

* `data` {any} 二进制数据输入字符串。

将 `data` 从二进制字符串编码为 Base64 字符串。`data` 可以是任何 JavaScript 值，可以强制转换为字符串。由于此函数仅处理由 `'a'` 到 `'z'`、`'A'` 到 `'Z'`、`'0'` 到 `'9'`、`'+'`、`'/'` 和 `'='` 组成的字符，因此该函数会忽略字符串中可能存在的任何其他字符。

返回包含 Base64 表示的字符串。

此函数提供了与浏览器中的 `btoa()` 函数的兼容性。有关此函数的使用和注意事项的详细信息，请参阅 [`btoa` MDN 文档][]。

### `buffer.isAscii(input)`

<!-- YAML
added: v20.12.0
-->

* `input` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* 返回：{boolean}

如果 `input` 仅包含有效的 ASCII 编码数据，包括输入为空的情况，则返回 `true`。

### `buffer.isUtf8(input)`

<!-- YAML
added: v20.12.0
-->

* `input` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* 返回：{boolean}

如果 `input` 仅包含有效的 UTF-8 编码数据，包括输入为空的情况，则返回 `true`。

### `buffer.transcode(source, fromEnc, toEnc)`

有关详细信息，请参阅 [`buffer.transcode()`]。

### `Blob` 类

有关详细信息，请参阅 [`Blob` 类]。

### `File` 类

有关详细信息，请参阅 [`File` 类]。

### `buffer.resolveObjectURL(id)`

<!-- YAML
added:
  - v20.0.0
  - v19.0.0
-->

> Stability: 1 - 实验性

* `id` {string} 先前调用 `URL.createObjectURL()` 返回的“blob:nodedata:...” URL 字符串。
* 返回：{Blob}

解析“blob:nodedata:...” URL 字符串（由 Node.js 的 `URL.createObjectURL()` 实现返回）到表示 URL 所引用的对象的 {Blob}。

### `buffer.isEncoding(encoding)`

<!-- YAML
added: v0.9.1
-->

* `encoding` {string} 要检查的字符编码名称。
* 返回：{boolean}

如果 `encoding` 是受支持的字符编码的名称，则返回 `true`，否则返回 `false`。

```mjs
import { Buffer } from 'node:buffer';

console.log(Buffer.isEncoding('utf8'));
// 打印: true

console.log(Buffer.isEncoding('hex'));
// 打印: true

console.log(Buffer.isEncoding('utf/8'));
// 打印: false

console.log(Buffer.isEncoding(''));
// 打印: false
```

```cjs
const { Buffer } = require('node:buffer');

console.log(Buffer.isEncoding('utf8'));
// 打印: true

console.log(Buffer.isEncoding('hex'));
// 打印: true

console.log(Buffer.isEncoding('utf/8'));
// 打印: false

console.log(Buffer.isEncoding(''));
// 打印: false
```

### `Buffer` 常量

这些是在 `buffer.constants` 上定义的，并且特定于 Node.js 的 `Buffer` 实现。它们也可通过 `Buffer.constants` 或 `buffer.constants` 获得。

#### `buffer.constants.MAX_LENGTH`

<!-- YAML
added:
 - v8.2.0
 - v6.5.0
-->

* 类型：{integer} 单个 `Buffer` 实例允许的最大大小。

在 32 位架构上，此值为 `(2^30)-1`（~1GiB）。
在 64 位架构上，此值为 `(2^31)-1`（~2GiB）。

该值也可用作 [`buffer.kMaxLength`][]。

#### `buffer.constants.MAX_STRING_LENGTH`

<!-- YAML
added:
 - v8.2.0
 - v6.5.0
-->

* 类型：{integer} 单个 `string` 实例允许的最大长度（以 UTF-16 代码单元计）。

表示 `string` 原语可以增长到的最大 `length`，以 UTF-16 代码单元计。

该值可能取决于正在使用的 JS 引擎。

## 注意

### `Buffer.from()`、`Buffer.alloc()` 和 `Buffer.allocUnsafe()`

在 Node.js 6.0.0 之前的版本中，`Buffer` 实例是使用 `Buffer` 构造函数创建的，它根据提供的参数以不同方式分配返回的 `Buffer`：

* 将数字作为第一个参数传递给 `Buffer()`（例如 `new Buffer(10)`）会分配指定大小的新 `Buffer` 对象。在 Node.js 8.0.0 之前，为这样的 `Buffer` 实例分配的内存*未初始化*，并且*可能包含敏感数据*。此类 `Buffer` 实例*必须*随后通过使用 [`buf.fill(0)`][`buf.fill()`] 或写入整个 `Buffer` 来初始化。虽然此行为是*为了提高性能*而有意为之，但开发经验表明，在快速创建和慢速初始化之间需要更明确的区分。从 Node.js 8.0.0 开始，`Buffer(num)` 和 `new Buffer(num)` 将返回具有初始化内存的 `Buffer`。
* 传递字符串、数组或 `Buffer` 作为第一个参数会将传递的对象的数据复制到 `Buffer` 中。
* 传递 {ArrayBuffer} 或 {SharedArrayBuffer} 会返回与给定 {ArrayBuffer} 共享分配内存的 `Buffer`。

由于 `new Buffer()` 的行为因第一个参数的类型而异，因此当未执行参数验证或初始化时，可能会无意中在应用程序中引入安全性和可靠性问题。

例如，如果攻击者可以使应用程序接收到期望字符串的数字，则应用程序可能会调用 `new Buffer(100)` 而不是 `new Buffer("100")`，从而导致它分配 100 字节的缓冲区而不是分配内容为 `"100"` 的 3 字节缓冲区。这通常可以使用 JSON API 调用实现。由于 JSON 区分数字和字符串类型，因此它允许在不进行任何强制转换的情况下注入数字，其中应用程序逻辑可能期望始终接收字符串。在 Node.js 8.0.0 之前，100 字节的缓冲区可能包含任意预先存在的内存数据，因此可用于向远程攻击者公开内存机密。从 Node.js 8.0.0 开始，公开内存不会发生，因为数据是零填充的。但是，其他攻击仍然存在，例如导致服务器分配非常大缓冲区，导致性能下降或崩溃。

为了使 `Buffer` 实例的创建更可靠且不易出错，各种形式的 `new Buffer()` 构造函数已被**弃用**，并由单独的方法 `Buffer.from()`、`Buffer.alloc()` 和 `Buffer.allocUnsafe()` 替换。

*开发者应将所有现有的 `new Buffer()` 构造函数调用迁移到这些新 API 之一。*

* [`Buffer.from(array)`][] 返回一个新的 `Buffer`，其中包含提供的八位字节数组的副本。
* [`Buffer.from(arrayBuffer[, byteOffset[, length]])`][`Buffer.from(arrayBuf)`] 返回一个新的 `Buffer`，它与给定的 {ArrayBuffer} 共享相同的分配内存。
* [`Buffer.from(buffer)`][] 返回一个新的 `Buffer`，其中包含给定 `Buffer` 的内容的副本。
* [`Buffer.from(string[, encoding])`][`Buffer.from(string)`] 返回一个新的 `Buffer`，其中包含给定字符串的副本。
* [`Buffer.alloc(size[, fill[, encoding]])`][`Buffer.alloc()`] 返回一个指定大小的新 `Buffer`，该 `Buffer` 已填充。如果未指定 `fill`，则 `Buffer` 将被零填充。
* [`Buffer.allocUnsafe(size)`][`Buffer.allocUnsafe()`] 和 [`Buffer.allocUnsafeSlow(size)`][`Buffer.allocUnsafeSlow()`] 各自返回一个指定 `size` 的新 `Buffer`，但其内容*必须*使用 [`buf.fill(0)`][`buf.fill()`] 或通过完全写入 `Buffer` 来初始化。

如果 `size` 小于或等于 [`Buffer.poolSize`][] 的一半，则 `Buffer.allocUnsafe()` 返回的 `Buffer` 实例*可能*从共享内部内存池中分配。`Buffer.allocUnsafeSlow()` 返回的实例*从不*使用共享内部内存池。

#### `--zero-fill-buffers` 命令行选项

Node.js 可以使用 `--zero-fill-buffers` 命令行选项启动，以强制所有新分配的 `Buffer` 实例在创建时默认使用零填充，包括由 `new Buffer(size)`、`Buffer.allocUnsafe()`、`Buffer.allocUnsafeSlow()` 和 `new SlowBuffer(size)` 返回的实例。使用此标志可以更改这些方法的默认行为，并影响性能敏感的应用程序。建议仅在必要时使用 `--zero-fill-buffers` 选项来强制所有新分配的 `Buffer` 实例在创建时使用零填充。

```bash
$ node --zero-fill-buffers
> Buffer.allocUnsafe(5);
<Buffer 00 00 00 00 00>
```

#### `Buffer.allocUnsafe()` 和 `Buffer.allocUnsafeSlow()` 的安全性

当调用 `Buffer.allocUnsafe()` 和 `Buffer.allocUnsafeSlow()` 时，分配的内存段*未初始化*（未清零）。虽然这种设计使内存分配非常快，但分配的内存段可能包含可能敏感的旧数据。使用由 `Buffer.allocUnsafe()` 创建的 `Buffer` 而不完全覆盖内存*可能*允许在读取 `Buffer` 内存时泄露这些旧数据。

虽然使用 `Buffer.allocUnsafe()` 有明显的性能优势，但*必须*额外小心，以避免将安全漏洞引入应用程序。

如果应用程序对性能敏感，并且 `Buffer.allocUnsafe()` 用于频繁分配小 `Buffer`，则建议将应用程序的一部分移至本机插件，以便可以更快地分配内存。

开发人员在使用 `Buffer.allocUnsafe()` 时应始终牢记安全性和性能之间的权衡。

### 缓冲区和 TypedArray

`Buffer` 实例也是 {Uint8Array} 实例。但是，与 {TypedArray} 相比，`Buffer` API 存在细微的不兼容性。例如，虽然 [`ArrayBuffer.prototype.slice()`][] 创建了 `ArrayBuffer` 的一部分的副本，但 [`Buffer.prototype.slice()`][`buf.slice()`] 在不复制的情况下在现有 `Buffer` 上创建视图。可以通过使用 `TypedArray.prototype.subarray()` 在 `Buffer` 和其他 {TypedArray} 上实现 [`Buffer.prototype.slice()`][`buf.slice()`] 的行为。

此外，虽然 `Buffer` 实例的行为类似于 {Uint8Array}，但某些方法在两种类型上的行为不同。具体来说，虽然 `Uint8Array.prototype.subarray()` 在修改时会更改原始 {TypedArray}，但 `Buffer.prototype.subarray()` 在不修改原始 `Buffer` 的情况下返回一个新的 `Buffer` 副本。

在可能的情况下，开发人员应避免使用导致 `Buffer` 和其他 {TypedArray} 之间行为差异的 API。有关更多信息，请参阅 [`Buffer` 和 `TypedArray`](#buffer-and-typedarray)。

[ASCII]: https://en.wikipedia.org/wiki/ASCII
[Base64]: https://en.wikipedia.org/wiki/Base64
[ISO-8859-1]: https://en.wikipedia.org/wiki/ISO-8859-1
[RFC 4648, Section 5]: https://tools.ietf.org/html/rfc4648#section-5
[UTF-8]: https://en.wikipedia.org/wiki/UTF-8
[UTF-16]: https://en.wikipedia.org/wiki/UTF-16
[WHATWG 编码标准]: https://encoding.spec.whatwg.org/
[`ArrayBuffer.prototype.slice()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer/slice
[`Blob` 类]: #class-blob
[`Buffer.alloc()`]: #static-method-bufferallocsize-fill-encoding
[`Buffer.allocUnsafe()`]: #static-method-bufferallocunsafesize
[`Buffer.allocUnsafeSlow()`]: #static-method-bufferallocunsafeslowsize
[`Buffer.from(array)`]: #static-method-bufferfromarray
[`Buffer.from(arrayBuf)`]: #static-method-bufferfromarraybuffer-byteoffset-length
[`Buffer.from(buffer)`]: #static-method-bufferfrombuffer
[`Buffer.from(string)`]: #static-method-bufferfromstring-encoding
[`Buffer.poolSize`]: #bufferpoolsize
[`File` 类]: #class-file
[`JSON.stringify()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify
[`TypedArray.from()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/from
[`TypedArray.prototype.slice()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/slice
[`TypedArray.prototype.subarray()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/subarray
[`atob` MDN 文档]: https://developer.mozilla.org/en-US/docs/Web/API/atob
[`btoa` MDN 文档]: https://developer.mozilla.org/en-US/docs/Web/API/btoa
[`buf.buffer`]: #bufbuffer
[`buf.byteOffset`]: #bufbyteoffset
[`buf.compare()`]: #bufcomparetarget-targetstart-targetend-sourcestart-sourceend
[`buf.entries()`]: #bufentries
[`buf.fill()`]: #buffillvalue-offset-end-encoding
[`buf.indexOf()`]: #bufindexofvalue-byteoffset-encoding
[`buf.keys()`]: #bufkeys
[`buf.length`]: #buflength
[`buf.slice()`]: #bufslicestart-end
[`buf.subarray`]: #bufsubarraystart-end
[`buf.toString()`]: #buftostringencoding-start-end
[`buf.values()`]: #bufvalues
[`buffer.constants.MAX_LENGTH`]: #bufferconstantsmax_length
[`buffer.constants.MAX_STRING_LENGTH`]: #bufferconstantsmax_string_length
[`buffer.kMaxLength`]: #bufferkmaxlength
[`buffer.transcode()`]: #buffertranscodesource-fromenc-toenc
[`buf.slice()`]: #bufslicestart-end
[`ERR_INVALID_BUFFER_SIZE`]: errors.md#err_invalid_buffer_size
[`ERR_OUT_OF_RANGE`]: errors.md#err_out_of_range
[`File` Web API]: https://developer.mozilla.org/en-US/docs/Web/API/File
[`String.prototype.indexOf()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/indexOf
[`String.prototype.lastIndexOf()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/String/lastIndexOf
[`TypedArray.prototype.set()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/TypedArray/set
[`util.inspect()`]: util.md#utilinspectobject-options
[base64url]: https://tools.ietf.org/html/rfc4648#section-5
[endianness]: https://en.wikipedia.org/wiki/Endianness
[迭代器]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Iteration_protocols