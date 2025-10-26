# 文件系统

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!--name=fs-->

<!-- source_link=lib/fs.js -->

`node:fs` 模块提供了以标准 POSIX 函数为模型的方式与文件系统进行交互。

要使用基于 Promise 的 API：

```mjs
import * as fs from 'node:fs/promises';
```

```cjs
const fs = require('node:fs/promises');
```

要使用回调和同步 API：

```mjs
import * as fs from 'node:fs';
```

```cjs
const fs = require('node:fs');
```

所有文件系统操作都有同步、回调和基于 Promise 的形式，并且可以通过 CommonJS 语法和 ES6 模块（ESM）访问。

## Promise 示例

基于 Promise 的操作返回一个 Promise，当异步操作完成时该 Promise 会被兑现。

```mjs
import { unlink } from 'node:fs/promises';

try {
  await unlink('/tmp/hello');
  console.log('successfully deleted /tmp/hello');
} catch (error) {
  console.error('there was an error:', error.message);
}
```

```cjs
const { unlink } = require('node:fs/promises');

(async function(path) {
  try {
    await unlink(path);
    console.log(`successfully deleted ${path}`);
  } catch (error) {
    console.error('there was an error:', error.message);
  }
})('/tmp/hello');
```

## 回调示例

回调形式将完成回调函数作为其最后一个参数，并异步调用操作。传递给完成回调的参数取决于方法，但第一个参数始终保留给异常。如果操作成功完成，则第一个参数为 `null` 或 `undefined`。

```mjs
import { unlink } from 'node:fs';

unlink('/tmp/hello', (err) => {
  if (err) throw err;
  console.log('successfully deleted /tmp/hello');
});
```

```cjs
const { unlink } = require('node:fs');

unlink('/tmp/hello', (err) => {
  if (err) throw err;
  console.log('successfully deleted /tmp/hello');
});
```

当需要最大性能（无论是执行时间还是内存分配方面）时，基于回调的 `node:fs` 模块 API 比使用 Promise API 更可取。

## 同步示例

同步 API 会阻塞 Node.js 事件循环和进一步的 JavaScript 执行，直到操作完成。异常会立即抛出，可以使用 `try…catch` 处理，或者允许冒泡。

```mjs
import { unlinkSync } from 'node:fs';

try {
  unlinkSync('/tmp/hello');
  console.log('successfully deleted /tmp/hello');
} catch (err) {
  // 处理错误
}
```

```cjs
const { unlinkSync } = require('node:fs');

try {
  unlinkSync('/tmp/hello');
  console.log('successfully deleted /tmp/hello');
} catch (err) {
  // 处理错误
}
```

## Promises API

<!-- YAML
added: v10.0.0
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31553
    description: Exposed as `require('fs/promises')`.
  - version:
    - v11.14.0
    - v10.17.0
    pr-url: https://github.com/nodejs/node/pull/26581
    description: This API is no longer experimental.
  - version: v10.1.0
    pr-url: https://github.com/nodejs/node/pull/20504
    description: The API is accessible via `require('fs').promises` only.
-->

`fs/promises` API 提供返回 Promise 的异步文件系统方法。

Promise API 使用底层的 Node.js 线程池在事件循环线程之外执行文件系统操作。这些操作不是同步的，也不是线程安全的。在对同一文件执行多个并发修改时必须小心，否则可能发生数据损坏。

### 类：`FileHandle`

<!-- YAML
added: v10.0.0
-->

{FileHandle} 对象是数字文件描述符的对象包装器。

{FileHandle} 对象的实例由 `fsPromises.open()` 方法创建。

所有 {FileHandle} 对象都是 {EventEmitter}。

如果 {FileHandle} 没有使用 `filehandle.close()` 方法关闭，它将尝试自动关闭文件描述符并发出进程警告，有助于防止内存泄漏。请不要依赖此行为，因为它可能不可靠，并且文件可能不会关闭。相反，应始终显式关闭 {FileHandle}。Node.js 将来可能会更改此行为。

#### 事件：`'close'`

<!-- YAML
added: v15.4.0
-->

当 {FileHandle} 已关闭且无法再使用时，会发出 `'close'` 事件。

#### `filehandle.appendFile(data[, options])`

<!-- YAML
added: v10.0.0
changes:
  - version:
    - v21.1.0
    - v20.10.0
    pr-url: https://github.com/nodejs/node/pull/50095
    description: The `flush` option is now supported.
  - version:
      - v15.14.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37490
    description: The `data` argument supports `AsyncIterable`, `Iterable`, and `Stream`.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `data` parameter won't coerce unsupported input to
                 strings anymore.
-->

* `data` {string|Buffer|TypedArray|DataView|AsyncIterable|Iterable|Stream}
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `signal` {AbortSignal|undefined} 允许中止正在进行的 writeFile。**默认值:** `undefined`
* 返回: {Promise} 成功时使用 `undefined` 兑现。

[`filehandle.writeFile()`][] 的别名。

在文件句柄上操作时，模式无法从 [`fsPromises.open()`][] 设置的模式更改。因此，这等同于 [`filehandle.writeFile()`][]。

#### `filehandle.chmod(mode)`

<!-- YAML
added: v10.0.0
-->

* `mode` {integer} 文件模式位掩码。
* 返回: {Promise} 成功时使用 `undefined` 兑现。

修改文件的权限。参见 chmod(2)。

#### `filehandle.chown(uid, gid)`

<!-- YAML
added: v10.0.0
-->

* `uid` {integer} 文件新所有者的用户 ID。
* `gid` {integer} 文件新组的组 ID。
* 返回: {Promise} 成功时使用 `undefined` 兑现。

更改文件的所有权。chown(2) 的包装器。

#### `filehandle.close()`

<!-- YAML
added: v10.0.0
-->

* 返回: {Promise} 成功时使用 `undefined` 兑现。

在等待句柄上任何待处理的操作完成后关闭文件句柄。

```mjs
import { open } from 'node:fs/promises';

let filehandle;
try {
  filehandle = await open('thefile.txt', 'r');
} finally {
  await filehandle?.close();
}
```

#### `filehandle.createReadStream([options])`

<!-- YAML
added: v16.11.0
-->

* `options` {Object}
  * `encoding` {string} **默认值:** `null`
  * `autoClose` {boolean} **默认值:** `true`
  * `emitClose` {boolean} **默认值:** `true`
  * `start` {integer}
  * `end` {integer} **默认值:** `Infinity`
  * `highWaterMark` {integer} **默认值:** `64 * 1024`
  * `signal` {AbortSignal|undefined} **默认值:** `undefined`
* 返回: {fs.ReadStream}

`options` 可以包含 `start` 和 `end` 值，以从文件中读取一个字节范围而不是整个文件。`start` 和 `end` 都包含在内，从 0 开始计数，允许的值在 \[0, [`Number.MAX_SAFE_INTEGER`][]] 范围内。如果省略或 `undefined` `start`，`filehandle.createReadStream()` 将从当前文件位置顺序读取。`encoding` 可以是 {Buffer} 接受的任何编码之一。

如果 `FileHandle` 指向仅支持阻塞读取的字符设备（如键盘或声卡），则读取操作在数据可用之前不会完成。这可能会阻止进程退出和流自然关闭。

默认情况下，流在销毁后会发出 `'close'` 事件。将 `emitClose` 选项设置为 `false` 可以更改此行为。

```mjs
import { open } from 'node:fs/promises';

const fd = await open('/dev/input/event0');
// 从某个字符设备创建流。
const stream = fd.createReadStream();
setTimeout(() => {
  stream.close(); // 这可能不会关闭流。
  // 人工标记流结束，就好像底层资源自身指示了文件结束，允许流关闭。
  // 这不会取消待处理的读取操作，如果有这样的操作，进程可能仍然无法成功退出，直到它完成。
  stream.push(null);
  stream.read(0);
}, 100);
```

如果 `autoClose` 为 false，那么即使有错误，文件描述符也不会关闭。应用程序有责任关闭它并确保没有文件描述符泄漏。如果 `autoClose` 设置为 true（默认行为），在 `'error'` 或 `'end'` 时，文件描述符将自动关闭。

读取一个 100 字节长的文件的最后 10 字节的示例：

```mjs
import { open } from 'node:fs/promises';

const fd = await open('sample.txt');
fd.createReadStream({ start: 90, end: 99 });
```

#### `filehandle.createWriteStream([options])`

<!-- YAML
added: v16.11.0
changes:
  - version:
    - v21.0.0
    - v20.10.0
    pr-url: https://github.com/nodejs/node/pull/50093
    description: The `flush` option is now supported.
-->

* `options` {Object}
  * `encoding` {string} **默认值:** `'utf8'`
  * `autoClose` {boolean} **默认值:** `true`
  * `emitClose` {boolean} **默认值:** `true`
  * `start` {integer}
  * `highWaterMark` {number} **默认值:** `16384`
  * `flush` {boolean} 如果为 `true`，则在关闭底层文件描述符之前会刷新它。**默认值:** `false`。
* 返回: {fs.WriteStream}

`options` 还可以包含 `start` 选项，以允许在文件开头之后的某个位置写入数据，允许的值在 \[0, [`Number.MAX_SAFE_INTEGER`][]] 范围内。修改文件而不是替换它可能需要将 `flags` `open` 选项设置为 `r+` 而不是默认的 `r`。`encoding` 可以是 {Buffer} 接受的任何编码之一。

如果 `autoClose` 设置为 true（默认行为），在 `'error'` 或 `'finish'` 时，文件描述符将自动关闭。如果 `autoClose` 为 false，那么即使有错误，文件描述符也不会关闭。应用程序有责任关闭它并确保没有文件描述符泄漏。

默认情况下，流在销毁后会发出 `'close'` 事件。将 `emitClose` 选项设置为 `false` 可以更改此行为。

#### `filehandle.datasync()`

<!-- YAML
added: v10.0.0
-->

* 返回: {Promise} 成功时使用 `undefined` 兑现。

强制所有当前与文件关联的排队 I/O 操作到操作系统的同步 I/O 完成状态。有关详细信息，请参阅 POSIX fdatasync(2) 文档。

与 `filehandle.sync` 不同，此方法不会刷新修改的元数据。

#### `filehandle.fd`

<!-- YAML
added: v10.0.0
-->

* 类型: {number} 由 {FileHandle} 对象管理的数字文件描述符。

#### `filehandle.read(buffer, offset, length, position)`

<!-- YAML
added: v10.0.0
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/42835
    description: Accepts bigint values as `position`.
-->

* `buffer` {Buffer|TypedArray|DataView} 将用读取的文件数据填充的缓冲区。
* `offset` {integer} 缓冲区中开始填充的位置。**默认值:** `0`
* `length` {integer} 要读取的字节数。**默认值:** `buffer.byteLength - offset`
* `position` {integer|bigint|null} 从文件中开始读取数据的位置。如果 `null` 或 `-1`，将从当前文件位置读取数据，并且位置将被更新。如果 `position` 是非负整数，则当前文件位置将保持不变。**默认值:** `null`
* 返回: {Promise} 成功时使用具有两个属性的对象兑现：
  * `bytesRead` {integer} 读取的字节数
  * `buffer` {Buffer|TypedArray|DataView} 对传入的 `buffer` 参数的引用。

从文件中读取数据并将其存储在给定的缓冲区中。

如果文件没有被并发修改，当读取的字节数为零时达到文件末尾。

#### `filehandle.read([options])`

<!-- YAML
added:
 - v13.11.0
 - v12.17.0
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/42835
    description: Accepts bigint values as `position`.
-->

* `options` {Object}
  * `buffer` {Buffer|TypedArray|DataView} 将用读取的文件数据填充的缓冲区。**默认值:** `Buffer.alloc(16384)`
  * `offset` {integer} 缓冲区中开始填充的位置。**默认值:** `0`
  * `length` {integer} 要读取的字节数。**默认值:** `buffer.byteLength - offset`
  * `position` {integer|bigint|null} 从文件中开始读取数据的位置。如果 `null` 或 `-1`，将从当前文件位置读取数据，并且位置将被更新。如果 `position` 是非负整数，则当前文件位置将保持不变。**默认值:** `null`
* 返回: {Promise} 成功时使用具有两个属性的对象兑现：
  * `bytesRead` {integer} 读取的字节数
  * `buffer` {Buffer|TypedArray|DataView} 对传入的 `buffer` 参数的引用。

从文件中读取数据并将其存储在给定的缓冲区中。

如果文件没有被并发修改，当读取的字节数为零时达到文件末尾。

#### `filehandle.read(buffer[, options])`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/42835
    description: Accepts bigint values as `position`.
-->

* `buffer` {Buffer|TypedArray|DataView} 将用读取的文件数据填充的缓冲区。
* `options` {Object}
  * `offset` {integer} 缓冲区中开始填充的位置。**默认值:** `0`
  * `length` {integer} 要读取的字节数。**默认值:** `buffer.byteLength - offset`
  * `position` {integer|bigint|null} 从文件中开始读取数据的位置。如果 `null` 或 `-1`，将从当前文件位置读取数据，并且位置将被更新。如果 `position` 是非负整数，则当前文件位置将保持不变。**默认值:** `null`
* 返回: {Promise} 成功时使用具有两个属性的对象兑现：
  * `bytesRead` {integer} 读取的字节数
  * `buffer` {Buffer|TypedArray|DataView} 对传入的 `buffer` 参数的引用。

从文件中读取数据并将其存储在给定的缓冲区中。

如果文件没有被并发修改，当读取的字节数为零时达到文件末尾。

#### `filehandle.readableWebStream([options])`

<!-- YAML
added: v17.0.0
changes:
  - version: v24.2.0
    pr-url: https://github.com/nodejs/node/pull/58548
    description: Added the `autoClose` option.
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57513
    description: Marking the API stable.
  - version:
    - v23.8.0
    - v22.15.0
    pr-url: https://github.com/nodejs/node/pull/55461
    description: Removed option to create a 'bytes' stream. Streams are now always 'bytes' streams.
  - version:
    - v20.0.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/46933
    description: Added option to create a 'bytes' stream.
-->

* `options` {Object}
  * `autoClose` {boolean} 当为 true 时，导致 {FileHandle} 在流关闭时被关闭。**默认值:** `false`
* 返回: {ReadableStream}

返回一个面向字节的 `ReadableStream`，可用于读取文件的内容。

如果此方法被多次调用或在 `FileHandle` 关闭或正在关闭后调用，将抛出错误。

```mjs
import {
  open,
} from 'node:fs/promises';

const file = await open('./some/file/to/read');

for await (const chunk of file.readableWebStream())
  console.log(chunk);

await file.close();
```

```cjs
const {
  open,
} = require('node:fs/promises');

(async () => {
  const file = await open('./some/file/to/read');

  for await (const chunk of file.readableWebStream())
    console.log(chunk);

  await file.close();
})();
```

虽然 `ReadableStream` 会读取文件直到完成，但它不会自动关闭 `FileHandle`。用户代码仍必须调用 `fileHandle.close()` 方法。

#### `filehandle.readFile(options)`

<!-- YAML
added: v10.0.0
-->

* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `null`
  * `signal` {AbortSignal} 允许中止正在进行的 readFile
* 返回: {Promise} 成功读取时使用文件的内容兑现。如果未指定编码（使用 `options.encoding`），则数据作为 {Buffer} 对象返回。否则，数据将是一个字符串。

异步读取文件的全部内容。

如果 `options` 是字符串，则它指定 `encoding`。

{FileHandle} 必须支持读取。

如果在文件句柄上进行了一个或多个 `filehandle.read()` 调用，然后进行了 `filehandle.readFile()` 调用，则将从当前位置读取数据直到文件末尾。它并不总是从文件开头读取。

#### `filehandle.readLines([options])`

<!-- YAML
added: v18.11.0
-->

* `options` {Object}
  * `encoding` {string} **默认值:** `null`
  * `autoClose` {boolean} **默认值:** `true`
  * `emitClose` {boolean} **默认值:** `true`
  * `start` {integer}
  * `end` {integer} **默认值:** `Infinity`
  * `highWaterMark` {integer} **默认值:** `64 * 1024`
* 返回: {readline.InterfaceConstructor}

创建一个 `readline` 接口并流式传输文件的便捷方法。有关选项，请参见 [`filehandle.createReadStream()`][]。

```mjs
import { open } from 'node:fs/promises';

const file = await open('./some/file/to/read');

for await (const line of file.readLines()) {
  console.log(line);
}
```

```cjs
const { open } = require('node:fs/promises');

(async () => {
  const file = await open('./some/file/to/read');

  for await (const line of file.readLines()) {
    console.log(line);
  }
})();
```

#### `filehandle.readv(buffers[, position])`

<!-- YAML
added:
 - v13.13.0
 - v12.17.0
-->

* `buffers` {Buffer\[]|TypedArray\[]|DataView\[]}
* `position` {integer|null} 从文件开头开始读取数据的偏移量。如果 `position` 不是 `number`，则将从当前位置读取数据。**默认值:** `null`
* 返回: {Promise} 成功时使用包含两个属性的对象兑现：
  * `bytesRead` {integer} 读取的字节数
  * `buffers` {Buffer\[]|TypedArray\[]|DataView\[]} 包含对 `buffers` 输入引用的属性。

从文件读取并写入到 {ArrayBufferView} 数组。

#### `filehandle.stat([options])`

<!-- YAML
added: v10.0.0
changes:
  - version: v10.5.0
    pr-url: https://github.com/nodejs/node/pull/20220
    description: Accepts an additional `options` object to specify whether
                 the numeric values returned should be bigint.
-->

* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.Stats} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* 返回: {Promise} 使用文件的 {fs.Stats} 兑现。

#### `filehandle.sync()`

<!-- YAML
added: v10.0.0
-->

* 返回: {Promise} 成功时使用 `undefined` 兑现。

请求将打开文件描述符的所有数据刷新到存储设备。具体实现取决于操作系统和设备。有关更多细节，请参阅 POSIX fsync(2) 文档。

#### `filehandle.truncate(len)`

<!-- YAML
added: v10.0.0
-->

* `len` {integer} **默认值:** `0`
* 返回: {Promise} 成功时使用 `undefined` 兑现。

截断文件。

如果文件大于 `len` 字节，则仅保留文件中的前 `len` 字节。

以下示例仅保留文件的前四个字节：

```mjs
import { open } from 'node:fs/promises';

let filehandle = null;
try {
  filehandle = await open('temp.txt', 'r+');
  await filehandle.truncate(4);
} finally {
  await filehandle?.close();
}
```

如果文件先前短于 `len` 字节，则会被扩展，扩展部分用空字节（`'\0'`）填充：

如果 `len` 为负数，则将使用 `0`。

#### `filehandle.utimes(atime, mtime)`

<!-- YAML
added: v10.0.0
-->

* `atime` {number|string|Date}
* `mtime` {number|string|Date}
* 返回: {Promise}

更改 {FileHandle} 引用的对象的文件系统时间戳，然后在成功时使用无参数兑现 Promise。

#### `filehandle.write(buffer, offset[, length[, position]])`

<!-- YAML
added: v10.0.0
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `buffer` parameter won't coerce unsupported input to
                 buffers anymore.
-->

* `buffer` {Buffer|TypedArray|DataView}
* `offset` {integer} `buffer` 中要写入的数据开始的位置。
* `length` {integer} 要从 `buffer` 写入的字节数。**默认值:** `buffer.byteLength - offset`
* `position` {integer|null} 从文件开头开始写入 `buffer` 数据的偏移量。如果 `position` 不是 `number`，数据将写入当前位置。有关更多细节，请参阅 POSIX pwrite(2) 文档。**默认值:** `null`
* 返回: {Promise}

将 `buffer` 写入文件。

Promise 使用包含两个属性的对象兑现：

* `bytesWritten` {integer} 写入的字节数
* `buffer` {Buffer|TypedArray|DataView} 对写入的 `buffer` 的引用。

在同一文件上多次使用 `filehandle.write()` 而不等待 Promise 兑现（或拒绝）是不安全的。对于这种情况，请使用 [`filehandle.createWriteStream()`][]。

在 Linux 上，当文件以追加模式打开时，位置写入不起作用。内核会忽略位置参数，始终将数据追加到文件末尾。

#### `filehandle.write(buffer[, options])`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

* `buffer` {Buffer|TypedArray|DataView}
* `options` {Object}
  * `offset` {integer} **默认值:** `0`
  * `length` {integer} **默认值:** `buffer.byteLength - offset`
  * `position` {integer|null} **默认值:** `null`
* 返回: {Promise}

将 `buffer` 写入文件。

类似于上面的 `filehandle.write` 函数，此版本接受一个可选的 `options` 对象。如果未指定 `options` 对象，它将使用上述值默认。

#### `filehandle.write(string[, position[, encoding]])`

<!-- YAML
added: v10.0.0
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `string` parameter won't coerce unsupported input to
                 strings anymore.
-->

* `string` {string}
* `position` {integer|null} 从文件开头开始写入 `string` 数据的偏移量。如果 `position` 不是 `number`，数据将写入当前位置。有关更多细节，请参阅 POSIX pwrite(2) 文档。**默认值:** `null`
* `encoding` {string} 预期的字符串编码。**默认值:** `'utf8'`
* 返回: {Promise}

将 `string` 写入文件。如果 `string` 不是字符串，Promise 将被拒绝并返回错误。

Promise 使用包含两个属性的对象兑现：

* `bytesWritten` {integer} 写入的字节数
* `buffer` {string} 对写入的 `string` 的引用。

在同一文件上多次使用 `filehandle.write()` 而不等待 Promise 兑现（或拒绝）是不安全的。对于这种情况，请使用 [`filehandle.createWriteStream()`][]。

在 Linux 上，当文件以追加模式打开时，位置写入不起作用。内核会忽略位置参数，始终将数据追加到文件末尾。

#### `filehandle.writeFile(data, options)`

<!-- YAML
added: v10.0.0
changes:
  - version:
      - v15.14.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37490
    description: The `data` argument supports `AsyncIterable`, `Iterable`, and `Stream`.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `data` parameter won't coerce unsupported input to
                 strings anymore.
-->

* `data` {string|Buffer|TypedArray|DataView|AsyncIterable|Iterable|Stream}
* `options` {Object|string}
  * `encoding` {string|null} 当 `data` 是字符串时的预期字符编码。**默认值:** `'utf8'`
  * `signal` {AbortSignal|undefined} 允许中止正在进行的 writeFile。**默认值:** `undefined`
* 返回: {Promise}

异步将数据写入文件，如果文件已存在则替换该文件。`data` 可以是字符串、缓冲区、{AsyncIterable} 或 {Iterable} 对象。Promise 在成功时使用无参数兑现。

如果 `options` 是字符串，则它指定 `encoding`。

{FileHandle} 必须支持写入。

在同一文件上多次使用 `filehandle.writeFile()` 而不等待 Promise 兑现（或拒绝）是不安全的。

如果在文件句柄上进行了一个或多个 `filehandle.write()` 调用，然后进行了 `filehandle.writeFile()` 调用，则将从当前位置写入数据直到文件末尾。它并不总是从文件开头写入。

#### `filehandle.writev(buffers[, position])`

<!-- YAML
added: v12.9.0
-->

* `buffers` {Buffer\[]|TypedArray\[]|DataView\[]}
* `position` {integer|null} 从文件开头开始写入 `buffers` 数据的偏移量。如果 `position` 不是 `number`，数据将写入当前位置。**默认值:** `null`
* 返回: {Promise}

将 {ArrayBufferView} 数组写入文件。

Promise 使用包含两个属性的对象兑现：

* `bytesWritten` {integer} 写入的字节数
* `buffers` {Buffer\[]|TypedArray\[]|DataView\[]} 对 `buffers` 输入的引用。

在同一文件上多次调用 `writev()` 而不等待 Promise 兑现（或拒绝）是不安全的。

在 Linux 上，当文件以追加模式打开时，位置写入不起作用。内核会忽略位置参数，始终将数据追加到文件末尾。

#### `filehandle[Symbol.asyncDispose]()`

<!-- YAML
added:
 - v20.4.0
 - v18.18.0
changes:
 - version: v24.2.0
   pr-url: https://github.com/nodejs/node/pull/58467
   description: No longer experimental.
-->

调用 `filehandle.close()` 并返回一个在文件句柄关闭时兑现的 Promise。

### `fsPromises.access(path[, mode])`

<!-- YAML
added: v10.0.0
-->

* `path` {string|Buffer|URL}
* `mode` {integer} **默认值:** `fs.constants.F_OK`
* 返回: {Promise} 成功时使用 `undefined` 兑现。

测试用户对 `path` 指定的文件或目录的权限。`mode` 参数是一个可选的整数，指定要执行的可访问性检查。`mode` 应该是值 `fs.constants.F_OK` 或由 `fs.constants.R_OK`、`fs.constants.W_OK` 和 `fs.constants.X_OK` 中任何值的按位或组成的掩码（例如 `fs.constants.W_OK | fs.constants.R_OK`）。有关 `mode` 的可能值，请检查 [文件访问常量][]。

如果可访问性检查成功，Promise 使用无值兑现。如果任何可访问性检查失败，Promise 将被拒绝并返回 {Error} 对象。以下示例检查文件 `/etc/passwd` 是否可以被当前进程读取和写入。

```mjs
import { access, constants } from 'node:fs/promises';

try {
  await access('/etc/passwd', constants.R_OK | constants.W_OK);
  console.log('can access');
} catch {
  console.error('cannot access');
}
```

在调用 `fsPromises.open()` 之前使用 `fsPromises.access()` 检查文件的可访问性是不推荐的。这样做会引入竞争条件，因为其他进程可能会在两个调用之间更改文件的状态。相反，用户代码应直接打开/读取/写入文件，并处理如果文件不可访问时引发的错误。

### `fsPromises.appendFile(path, data[, options])`

<!-- YAML
added: v10.0.0
changes:
  - version:
    - v21.1.0
    - v20.10.0
    pr-url: https://github.com/nodejs/node/pull/50095
    description: The `flush` option is now supported.
-->

* `path` {string|Buffer|URL|FileHandle} 文件名或 {FileHandle}
* `data` {string|Buffer}
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `mode` {integer} **默认值:** `0o666`
  * `flag` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'a'`。
  * `flush` {boolean} 如果为 `true`，则在关闭底层文件描述符之前会刷新它。**默认值:** `false`。
* 返回: {Promise} 成功时使用 `undefined` 兑现。

异步将数据追加到文件，如果文件尚不存在则创建该文件。`data` 可以是字符串或 {Buffer}。

如果 `options` 是字符串，则它指定 `encoding`。

`mode` 选项仅影响新创建的文件。有关更多细节，请参见 [`fs.open()`][]。

`path` 可以指定为已打开用于追加的 {FileHandle}（使用 `fsPromises.open()`）。

### `fsPromises.chmod(path, mode)`

<!-- YAML
added: v10.0.0
-->

* `path` {string|Buffer|URL}
* `mode` {string|integer}
* 返回: {Promise} 成功时使用 `undefined` 兑现。

更改文件的权限。

### `fsPromises.chown(path, uid, gid)`

<!-- YAML
added: v10.0.0
-->

* `path` {string|Buffer|URL}
* `uid` {integer}
* `gid` {integer}
* 返回: {Promise} 成功时使用 `undefined` 兑现。

更改文件的所有权。

### `fsPromises.copyFile(src, dest[, mode])`

<!-- YAML
added: v10.0.0
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/27044
    description: Changed `flags` argument to `mode` and imposed
                 stricter type validation.
-->

* `src` {string|Buffer|URL} 要复制的源文件名
* `dest` {string|Buffer|URL} 复制操作的目标文件名
* `mode` {integer} 复制操作的可选修饰符。可以创建由两个或多个值的按位或组成的掩码（例如 `fs.constants.COPYFILE_EXCL | fs.constants.COPYFILE_FICLONE`）**默认值:** `0`。
  * `fs.constants.COPYFILE_EXCL`：如果 `dest` 已存在，复制操作将失败。
  * `fs.constants.COPYFILE_FICLONE`：复制操作将尝试创建写时复制 reflink。如果平台不支持写时复制，则使用回退复制机制。
  * `fs.constants.COPYFILE_FICLONE_FORCE`：复制操作将尝试创建写时复制 reflink。如果平台不支持写时复制，则操作将失败。
* 返回: {Promise} 成功时使用 `undefined` 兑现。

异步地将 `src` 复制到 `dest`。默认情况下，如果 `dest` 已存在，则会被覆盖。

不保证复制操作的原子性。如果在目标文件已打开进行写入后发生错误，将尝试删除目标文件。

```mjs
import { copyFile, constants } from 'node:fs/promises';

try {
  await copyFile('source.txt', 'destination.txt');
  console.log('source.txt was copied to destination.txt');
} catch {
  console.error('The file could not be copied');
}

// 通过使用 COPYFILE_EXCL，如果 destination.txt 存在，操作将失败。
try {
  await copyFile('source.txt', 'destination.txt', constants.COPYFILE_EXCL);
  console.log('source.txt was copied to destination.txt');
} catch {
  console.error('The file could not be copied');
}
```

### `fsPromises.cp(src, dest[, options])`

<!-- YAML
added: v16.7.0
changes:
  - version: v22.3.0
    pr-url: https://github.com/nodejs/node/pull/53127
    description: This API is no longer experimental.
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/47084
    description: Accept an additional `mode` option to specify
                 the copy behavior as the `mode` argument of `fs.copyFile()`.
  - version:
    - v17.6.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/41819
    description: Accepts an additional `verbatimSymlinks` option to specify
                 whether to perform path resolution for symlinks.
-->

* `src` {string|URL} 要复制的源路径。
* `dest` {string|URL} 要复制到的目标路径。
* `options` {Object}
  * `dereference` {boolean} 取消引用符号链接。**默认值:** `false`。
  * `errorOnExist` {boolean} 当 `force` 为 `false` 且目标已存在时，抛出错误。**默认值:** `false`。
  * `filter` {Function} 过滤要复制的文件/目录的函数。返回 `true` 复制项目，`false` 忽略它。当忽略目录时，其所有内容也将被跳过。也可以返回一个解析为 `true` 或 `false` 的 `Promise` **默认值:** `undefined`。
    * `src` {string} 要复制的源路径。
    * `dest` {string} 要复制到的目标路径。
    * 返回: {boolean|Promise} 可强制转换为 `boolean` 的值或使用此类值履行的 `Promise`。
  * `force` {boolean} 覆盖现有文件或目录。如果将此设置为 false 且目标存在，复制操作将忽略错误。使用 `errorOnExist` 选项更改此行为。**默认值:** `true`。
  * `mode` {integer} 复制操作的修饰符。**默认值:** `0`。参见 [`fsPromises.copyFile()`][] 的 `mode` 标志。
  * `preserveTimestamps` {boolean} 当为 `true` 时，将保留 `src` 的时间戳。**默认值:** `false`。
  * `recursive` {boolean} 递归复制目录 **默认值:** `false`
  * `verbatimSymlinks` {boolean} 当为 `true` 时，将跳过符号链接的路径解析。**默认值:** `false`
* 返回: {Promise} 成功时使用 `undefined` 兑现。

异步地将整个目录结构从 `src` 复制到 `dest`，包括子目录和文件。

当将一个目录复制到另一个目录时，不支持通配符，行为类似于 `cp dir1/ dir2/`。

### `fsPromises.glob(pattern[, options])`

<!-- YAML
added: v22.0.0
changes:
  - version: v24.1.0
    pr-url: https://github.com/nodejs/node/pull/58182
    description: Add support for `URL` instances for `cwd` option.
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57513
    description: Marking the API stable.
  - version:
    - v23.7.0
    - v22.14.0
    pr-url: https://github.com/nodejs/node/pull/56489
    description: Add support for `exclude` option to accept glob patterns.
  - version: v22.2.0
    pr-url: https://github.com/nodejs/node/pull/52837
    description: Add support for `withFileTypes` as an option.
-->

* `pattern` {string|string\[]}
* `options` {Object}
  * `cwd` {string|URL} 当前工作目录。**默认值:** `process.cwd()`
  * `exclude` {Function|string\[]} 过滤掉文件/目录的函数或要排除的全局模式列表。如果提供了函数，返回 `true` 排除项目，`false` 包含它。**默认值:** `undefined`。如果提供了字符串数组，每个字符串应是指定要排除路径的全局模式。注意：不支持否定模式（例如 '!foo.js'）。
  * `withFileTypes` {boolean} 如果为 `true`，全局应返回路径作为 Dirent，否则为 `false`。**默认值:** `false`。
* 返回: {AsyncIterator} 一个产生匹配模式的文件路径的 AsyncIterator。

```mjs
import { glob } from 'node:fs/promises';

for await (const entry of glob('**/*.js'))
  console.log(entry);
```

```cjs
const { glob } = require('node:fs/promises');

(async () => {
  for await (const entry of glob('**/*.js'))
    console.log(entry);
})();
```

### `fsPromises.lchmod(path, mode)`

<!-- YAML
deprecated: v10.0.0
-->

> Stability: 0 - Deprecated

* `path` {string|Buffer|URL}
* `mode` {integer}
* 返回: {Promise} 成功时使用 `undefined` 兑现。

更改符号链接的权限。

此方法仅在 macOS 上实现。

### `fsPromises.lchown(path, uid, gid)`

<!-- YAML
added: v10.0.0
changes:
  - version: v10.6.0
    pr-url: https://github.com/nodejs/node/pull/21498
    description: This API is no longer deprecated.
-->

* `path` {string|Buffer|URL}
* `uid` {integer}
* `gid` {integer}
* 返回: {Promise} 成功时使用 `undefined` 兑现。

更改符号链接的所有权。

### `fsPromises.lutimes(path, atime, mtime)`

<!-- YAML
added:
  - v14.5.0
  - v12.19.0
-->

* `path` {string|Buffer|URL}
* `atime` {number|string|Date}
* `mtime` {number|string|Date}
* 返回: {Promise} 成功时使用 `undefined` 兑现。

以与 [`fsPromises.utimes()`][] 相同的方式更改文件的访问和修改时间，不同之处在于如果路径引用符号链接，则不会取消引用该链接：而是更改符号链接本身的时间戳。

### `fsPromises.link(existingPath, newPath)`

<!-- YAML
added: v10.0.0
-->

* `existingPath` {string|Buffer|URL}
* `newPath` {string|Buffer|URL}
* 返回: {Promise} 成功时使用 `undefined` 兑现。

从 `existingPath` 创建到 `newPath` 的新链接。有关更多细节，请参阅 POSIX link(2) 文档。

### `fsPromises.lstat(path[, options])`

<!-- YAML
added: v10.0.0
changes:
  - version: v10.5.0
    pr-url: https://github.com/nodejs/node/pull/20220
    description: Accepts an additional `options` object to specify whether
                 the numeric values returned should be bigint.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.Stats} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* 返回: {Promise} 使用给定符号链接 `path` 的 {fs.Stats} 对象兑现。

等同于 [`fsPromises.stat()`][]，除非 `path` 引用符号链接，在这种情况下，链接本身是 stat-ed，而不是它引用的文件。有关更多细节，请参阅 POSIX lstat(2) 文档。

### `fsPromises.mkdir(path[, options])`

<!-- YAML
added: v10.0.0
-->

* `path` {string|Buffer|URL}
* `options` {Object|integer}
  * `recursive` {boolean} **默认值:** `false`
  * `mode` {string|integer} 在 Windows 上不支持。**默认值:** `0o777`。
* 返回: {Promise} 成功时，如果 `recursive` 为 `false`，则使用 `undefined` 兑现，或者如果 `recursive` 为 `true`，则使用创建的第一个目录路径兑现。

异步创建目录。

可选的 `options` 参数可以是指定 `mode`（权限和粘滞位）的整数，或者是具有 `mode` 属性和 `recursive` 属性的对象，指示是否应创建父目录。当 `path` 是已存在的目录时，调用 `fsPromises.mkdir()` 仅当 `recursive` 为 false 时会导致拒绝。

```mjs
import { mkdir } from 'node:fs/promises';

try {
  const projectFolder = new URL('./test/project/', import.meta.url);
  const createDir = await mkdir(projectFolder, { recursive: true });

  console.log(`created ${createDir}`);
} catch (err) {
  console.error(err.message);
}
```

```cjs
const { mkdir } = require('node:fs/promises');
const { join } = require('node:path');

async function makeDirectory() {
  const projectFolder = join(__dirname, 'test', 'project');
  const dirCreation = await mkdir(projectFolder, { recursive: true });

  console.log(dirCreation);
  return dirCreation;
}

makeDirectory().catch(console.error);
```

### `fsPromises.mkdtemp(prefix[, options])`

<!-- YAML
added: v10.0.0
changes:
  - version:
    - v20.6.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/48828
    description: The `prefix` parameter now accepts buffers and URL.
  - version:
      - v16.5.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/39028
    description: The `prefix` parameter now accepts an empty string.
-->

* `prefix` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* 返回: {Promise} 使用包含新创建的临时目录的文件系统路径的字符串兑现。

创建唯一的临时目录。通过将六个随机字符附加到提供的 `prefix` 末尾来生成唯一的目录名。由于平台不一致，避免在 `prefix` 中使用尾随 `X` 字符。某些平台，特别是 BSD，可以返回超过六个随机字符，并用随机字符替换 `prefix` 中的尾随 `X` 字符。

可选的 `options` 参数可以是指定编码的字符串，或者是具有指定要使用的字符编码的 `encoding` 属性的对象。

```mjs
import { mkdtemp } from 'node:fs/promises';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

try {
  await mkdtemp(join(tmpdir(), 'foo-'));
} catch (err) {
  console.error(err);
}
```

`fsPromises.mkdtemp()` 方法将直接将六个随机选择的字符附加到 `prefix` 字符串。例如，给定目录 `/tmp`，如果意图是在 `/tmp` 内创建临时目录，则 `prefix` 必须以尾随的平台特定路径分隔符（`require('node:path').sep`）结尾。

### `fsPromises.mkdtempDisposable(prefix[, options])`

<!-- YAML
added: v24.4.0
-->

* `prefix` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* 返回: {Promise} 使用一个异步可处置对象的 Promise 兑现：
  * `path` {string} 创建的目录的路径。
  * `remove` {AsyncFunction} 移除创建的目录的函数。
  * `[Symbol.asyncDispose]` {AsyncFunction} 与 `remove` 相同。

生成的 Promise 持有一个异步可处置对象，其 `path` 属性持有创建的目录路径。当对象被处置时，如果目录仍然存在，它将异步移除目录及其内容。如果目录无法删除，处置将抛出错误。对象有一个异步 `remove()` 方法，将执行相同的任务。

此函数和结果对象上的处置函数都是异步的，因此应与 `await` + `await using` 一起使用，如 `await using dir = await fsPromises.mkdtempDisposable('prefix')`。

有关详细信息，请参阅 [`fsPromises.mkdtemp()`][] 的文档。

可选的 `options` 参数可以是指定编码的字符串，或者是具有指定要使用的字符编码的 `encoding` 属性的对象。

### `fsPromises.open(path, flags[, mode])`

<!-- YAML
added: v10.0.0
changes:
  - version: v11.1.0
    pr-url: https://github.com/nodejs/node/pull/23767
    description: The `flags` argument is now optional and defaults to `'r'`.
-->

* `path` {string|Buffer|URL}
* `flags` {string|number} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'r'`。
* `mode` {string|integer} 如果创建文件，设置文件模式（权限和粘滞位）。**默认值:** `0o666`（可读和可写）
* 返回: {Promise} 使用 {FileHandle} 对象兑现。

打开一个 {FileHandle}。

有关更多细节，请参阅 POSIX open(2) 文档。

在 Windows 上，某些字符（`< > : " / \ | ? *`）是保留的，如 [命名文件、路径和命名空间][] 所述。在 NTFS 下，如果文件名包含冒号，Node.js 将打开一个文件系统流，如 [此 MSDN 页面][MSDN-Using-Streams] 所述。

### `fsPromises.opendir(path[, options])`

<!-- YAML
added: v12.12.0
changes:
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/41439
    description: Added `recursive` option.
  - version:
     - v13.1.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30114
    description: The `bufferSize` option was introduced.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `bufferSize` {number} 从目录读取时内部缓冲的目录条目数。较高的值导致更好的性能但更高的内存使用。**默认值:** `32`
  * `recursive` {boolean} 解析的 `Dir` 将是一个包含所有子文件和目录的 {AsyncIterable}。**默认值:** `false`
* 返回: {Promise} 使用 {fs.Dir} 兑现。

异步打开目录以进行迭代扫描。有关更多细节，请参阅 POSIX opendir(3) 文档。

创建一个 {fs.Dir}，其中包含所有用于从目录读取和清理的进一步函数。

`encoding` 选项在打开目录及后续读取操作时设置 `path` 的编码。

使用异步迭代的示例：

```mjs
import { opendir } from 'node:fs/promises';

try {
  const dir = await opendir('./');
  for await (const dirent of dir)
    console.log(dirent.name);
} catch (err) {
  console.error(err);
}
```

当使用异步迭代器时，{fs.Dir} 对象将在迭代器退出后自动关闭。

### `fsPromises.readdir(path[, options])`

<!-- YAML
added: v10.0.0
changes:
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/41439
    description: Added `recursive` option.
  - version: v10.11.0
    pr-url: https://github.com/nodejs/node/pull/22020
    description: New option `withFileTypes` was added.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
  * `withFileTypes` {boolean} **默认值:** `false`
  * `recursive` {boolean} 如果为 `true`，则递归读取目录的内容。在递归模式下，它将列出所有文件、子文件和目录。**默认值:** `false`。
* 返回: {Promise} 使用目录中文件名称的数组兑现，不包括 `'.'` 和 `'..'`。

读取目录的内容。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定用于文件名的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的文件名将作为 {Buffer} 对象传递。

如果 `options.withFileTypes` 设置为 `true`，返回的数组将包含 {fs.Dirent} 对象。

```mjs
import { readdir } from 'node:fs/promises';

try {
  const files = await readdir(path);
  for (const file of files)
    console.log(file);
} catch (err) {
  console.error(err);
}
```

### `fsPromises.readFile(path[, options])`

<!-- YAML
added: v10.0.0
changes:
  - version:
    - v15.2.0
    - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/35911
    description: The options argument may include an AbortSignal to abort an
                 ongoing readFile request.
-->

* `path` {string|Buffer|URL|FileHandle} 文件名或 `FileHandle`
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `null`
  * `flag` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'r'`。
  * `signal` {AbortSignal} 允许中止正在进行的 readFile
* 返回: {Promise} 使用文件内容兑现。

异步读取文件的全部内容。

如果未指定编码（使用 `options.encoding`），则数据作为 {Buffer} 对象返回。否则，数据将是一个字符串。

如果 `options` 是字符串，则它指定编码。

当 `path` 是目录时，`fsPromises.readFile()` 的行为是特定于平台的。在 macOS、Linux 和 Windows 上，Promise 将被拒绝并返回错误。在 FreeBSD 上，将返回目录内容的表示。

读取位于运行代码同一目录中的 `package.json` 文件的示例：

```mjs
import { readFile } from 'node:fs/promises';
try {
  const filePath = new URL('./package.json', import.meta.url);
  const contents = await readFile(filePath, { encoding: 'utf8' });
  console.log(contents);
} catch (err) {
  console.error(err.message);
}
```

```cjs
const { readFile } = require('node:fs/promises');
const { resolve } = require('node:path');
async function logFile() {
  try {
    const filePath = resolve('./package.json');
    const contents = await readFile(filePath, { encoding: 'utf8' });
    console.log(contents);
  } catch (err) {
    console.error(err.message);
  }
}
logFile();
```

可以使用 {AbortSignal} 中止正在进行的 `readFile`。如果请求被中止，返回的 Promise 将被拒绝并返回 `AbortError`：

```mjs
import { readFile } from 'node:fs/promises';

try {
  const controller = new AbortController();
  const { signal } = controller;
  const promise = readFile(fileName, { signal });

  // 在 Promise 结算之前中止请求。
  controller.abort();

  await promise;
} catch (err) {
  // 当请求被中止时 - err 是 AbortError
  console.error(err);
}
```

中止正在进行的请求不会中止单个操作系统请求，而是中止 `fs.readFile` 执行的内部缓冲。

任何指定的 {FileHandle} 必须支持读取。

### `fsPromises.readlink(path[, options])`

<!-- YAML
added: v10.0.0
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* 返回: {Promise} 成功时使用 `linkString` 兑现。

读取 `path` 引用的符号链接的内容。有关更多细节，请参阅 POSIX readlink(2) 文档。Promise 在成功时使用 `linkString` 兑现。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定用于返回的链接路径的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的链接路径将作为 {Buffer} 对象传递。

### `fsPromises.realpath(path[, options])`

<!-- YAML
added: v10.0.0
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* 返回: {Promise} 成功时使用解析的路径兑现。

使用与 `fs.realpath.native()` 函数相同的语义确定 `path` 的实际位置。

仅支持可以转换为 UTF8 字符串的路径。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定用于路径的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的路径将作为 {Buffer} 对象传递。

在 Linux 上，当 Node.js 链接到 musl libc 时，procfs 文件系统必须挂载在 `/proc` 上才能使此函数工作。Glibc 没有此限制。

### `fsPromises.rename(oldPath, newPath)`

<!-- YAML
added: v10.0.0
-->

* `oldPath` {string|Buffer|URL}
* `newPath` {string|Buffer|URL}
* 返回: {Promise} 成功时使用 `undefined` 兑现。

将 `oldPath` 重命名为 `newPath`。

### `fsPromises.rmdir(path[, options])`

<!-- YAML
added: v10.0.0
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37216
    description: "Using `fsPromises.rmdir(path, { recursive: true })` on a `path`
                 that is a file is no longer permitted and results in an
                 `ENOENT` error on Windows and an `ENOTDIR` error on POSIX."
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37216
    description: "Using `fsPromises.rmdir(path, { recursive: true })` on a `path`
                 that does not exist is no longer permitted and results in a
                 `ENOENT` error."
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37302
    description: The `recursive` option is deprecated, using it triggers a
                 deprecation warning.
  - version: v14.14.0
    pr-url: https://github.com/nodejs/node/pull/35579
    description: The `recursive` option is deprecated, use `fsPromises.rm` instead.
  - version:
     - v13.3.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30644
    description: The `maxBusyTries` option is renamed to `maxRetries`, and its
                 default is 0. The `emfileWait` option has been removed, and
                 `EMFILE` errors use the same retry logic as other errors. The
                 `retryDelay` option is now supported. `ENFILE` errors are now
                 retried.
  - version: v12.10.0
    pr-url: https://github.com/nodejs/node/pull/29168
    description: The `recursive`, `maxBusyTries`, and `emfileWait` options are
                  now supported.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `maxRetries` {integer} 如果遇到 `EBUSY`、`EMFILE`、`ENFILE`、`ENOTEMPTY` 或 `EPERM` 错误，Node.js 将以每次重试线性退避等待 `retryDelay` 毫秒的方式重试操作。此选项表示重试次数。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `0`。
  * `recursive` {boolean} 如果为 `true`，则执行递归目录移除。在递归模式下，操作会在失败时重试。**默认值:** `false`。**已弃用。**
  * `retryDelay` {integer} 重试之间等待的时间量（毫秒）。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `100`。
* 返回: {Promise} 成功时使用 `undefined` 兑现。

移除由 `path` 标识的目录。

在文件（非目录）上使用 `fsPromises.rmdir()` 会导致在 Windows 上拒绝 Promise 并返回 `ENOENT` 错误，在 POSIX 上返回 `ENOTDIR` 错误。

要获得类似于 `rm -rf` Unix 命令的行为，请使用 [`fsPromises.rm()`][] 并设置选项 `{ recursive: true, force: true }`。

### `fsPromises.rm(path[, options])`

<!-- YAML
added: v14.14.0
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `force` {boolean} 当为 `true` 时，如果 `path` 不存在，异常将被忽略。**默认值:** `false`。
  * `maxRetries` {integer} 如果遇到 `EBUSY`、`EMFILE`、`ENFILE`、`ENOTEMPTY` 或 `EPERM` 错误，Node.js 将以每次重试线性退避等待 `retryDelay` 毫秒的方式重试操作。此选项表示重试次数。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `0`。
  * `recursive` {boolean} 如果为 `true`，则执行递归目录移除。在递归模式下，操作会在失败时重试。**默认值:** `false`。
  * `retryDelay` {integer} 重试之间等待的时间量（毫秒）。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `100`。
* 返回: {Promise} 成功时使用 `undefined` 兑现。

移除文件和目录（基于标准 POSIX `rm` 实用程序建模）。

### `fsPromises.stat(path[, options])`

<!-- YAML
added: v10.0.0
changes:
  - version: v10.5.0
    pr-url: https://github.com/nodejs/node/pull/20220
    description: Accepts an additional `options` object to specify whether
                 the numeric values returned should be bigint.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.Stats} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* 返回: {Promise} 使用给定 `path` 的 {fs.Stats} 对象兑现。

### `fsPromises.statfs(path[, options])`

<!-- YAML
added:
  - v19.6.0
  - v18.15.0
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.StatFs} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* 返回: {Promise} 使用给定 `path` 的 {fs.StatFs} 对象兑现。

### `fsPromises.symlink(target, path[, type])`

<!-- YAML
added: v10.0.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/42894
    description: If the `type` argument is `null` or omitted, Node.js will
                 autodetect `target` type and automatically
                 select `dir` or `file`.

-->

* `target` {string|Buffer|URL}
* `path` {string|Buffer|URL}
* `type` {string|null} **默认值:** `null`
* 返回: {Promise} 成功时使用 `undefined` 兑现。

创建符号链接。

`type` 参数仅在 Windows 平台上使用，可以是 `'dir'`、`'file'` 或 `'junction'` 之一。如果 `type` 参数为 `null`，Node.js 将自动检测 `target` 类型并使用 `'file'` 或 `'dir'`。如果 `target` 不存在，将使用 `'file'`。Windows 连接点要求目标路径是绝对的。当使用 `'junction'` 时，`target` 参数将自动规范化为绝对路径。NTFS 卷上的连接点只能指向目录。

### `fsPromises.truncate(path[, len])`

<!-- YAML
added: v10.0.0
-->

* `path` {string|Buffer|URL}
* `len` {integer} **默认值:** `0`
* 返回: {Promise} 成功时使用 `undefined` 兑现。

将 `path` 处的内容截断（缩短或扩展长度）为 `len` 字节。

### `fsPromises.unlink(path)`

<!-- YAML
added: v10.0.0
-->

* `path` {string|Buffer|URL}
* 返回: {Promise} 成功时使用 `undefined` 兑现。

如果 `path` 引用符号链接，则移除该链接而不影响该链接引用的文件或目录。如果 `path` 引用的文件路径不是符号链接，则删除该文件。有关更多细节，请参阅 POSIX unlink(2) 文档。

### `fsPromises.utimes(path, atime, mtime)`

<!-- YAML
added: v10.0.0
-->

* `path` {string|Buffer|URL}
* `atime` {number|string|Date}
* `mtime` {number|string|Date}
* 返回: {Promise} 成功时使用 `undefined` 兑现。

更改 `path` 引用的对象的文件系统时间戳。

`atime` 和 `mtime` 参数遵循以下规则：

* 值可以是代表 Unix 纪元时间的数字、`Date` 或数字字符串，如 `'123456789.0'`。
* 如果值无法转换为数字，或者是 `NaN`、`Infinity` 或 `-Infinity`，将抛出 `Error`。

### `fsPromises.watch(filename[, options])`

<!-- YAML
added:
  - v15.9.0
  - v14.18.0
-->

* `filename` {string|Buffer|URL}
* `options` {string|Object}
  * `persistent` {boolean} 指示只要正在监视文件，进程是否应继续运行。**默认值:** `true`。
  * `recursive` {boolean} 指示是否应监视所有子目录，或仅当前目录。这在指定目录时适用，并且仅在支持的平台上（参见 [注意事项][]）。**默认值:** `false`。
  * `encoding` {string} 指定传递给监听器的文件名使用的字符编码。**默认值:** `'utf8'`。
  * `signal` {AbortSignal} 用于指示监视器应停止的 {AbortSignal}。
  * `maxQueue` {number} 指定在 {AsyncIterator} 返回的迭代之间排队的事件数。**默认值:** `2048`。
  * `overflow` {string} 当排队事件超过 `maxQueue` 允许时的处理方式，可以是 `'ignore'` 或 `'throw'`。`'ignore'` 表示溢出事件被丢弃并发出警告，而 `'throw'` 表示抛出异常。**默认值:** `'ignore'`。
* 返回: {AsyncIterator} 具有以下属性的对象：
  * `eventType` {string} 更改类型
  * `filename` {string|Buffer|null} 更改的文件名

返回一个异步迭代器，监视 `filename` 上的更改，其中 `filename` 是文件或目录。

```js
const { watch } = require('node:fs/promises');

const ac = new AbortController();
const { signal } = ac;
setTimeout(() => ac.abort(), 10000);

(async () => {
  try {
    const watcher = watch(__filename, { signal });
    for await (const event of watcher)
      console.log(event);
  } catch (err) {
    if (err.name === 'AbortError')
      return;
    throw err;
  }
})();
```

在大多数平台上，每当文件名在目录中出现或消失时，都会发出 `'rename'`。

`fs.watch()` 的所有 [注意事项][] 也适用于 `fsPromises.watch()`。

### `fsPromises.writeFile(file, data[, options])`

<!-- YAML
added: v10.0.0
changes:
  - version:
    - v21.0.0
    - v20.10.0
    pr-url: https://github.com/nodejs/node/pull/50009
    description: The `flush` option is now supported.
  - version:
      - v15.14.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37490
    description: The `data` argument supports `AsyncIterable`, `Iterable`, and `Stream`.
  - version:
      - v15.2.0
      - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/35993
    description: The options argument may include an AbortSignal to abort an
                 ongoing writeFile request.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `data` parameter won't coerce unsupported input to
                 strings anymore.
-->

* `file` {string|Buffer|URL|FileHandle} 文件名或 `FileHandle`
* `data` {string|Buffer|TypedArray|DataView|AsyncIterable|Iterable|Stream}
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `mode` {integer} **默认值:** `0o666`
  * `flag` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'w'`。
  * `flush` {boolean} 如果所有数据成功写入文件，并且 `flush` 为 `true`，则使用 `filehandle.sync()` 刷新数据。**默认值:** `false`。
  * `signal` {AbortSignal} 允许中止正在进行的 writeFile
* 返回: {Promise} 成功时使用 `undefined` 兑现。

异步将数据写入文件，如果文件已存在则替换该文件。`data` 可以是字符串、缓冲区、{AsyncIterable} 或 {Iterable} 对象。

如果 `data` 是缓冲区，则忽略 `encoding` 选项。

如果 `options` 是字符串，则它指定编码。

`mode` 选项仅影响新创建的文件。有关更多细节，请参见 [`fs.open()`][]。

任何指定的 {FileHandle} 必须支持写入。

在同一文件上多次使用 `fsPromises.writeFile()` 而不等待 Promise 结算是不安全的。

类似于 `fsPromises.readFile` - `fsPromises.writeFile` 是一个便捷方法，它在内部执行多个 `write` 调用来写入传递给它的缓冲区。对于性能敏感的代码，请考虑使用 [`fs.createWriteStream()`][] 或 [`filehandle.createWriteStream()`][]。

可以使用 {AbortSignal} 取消 `fsPromises.writeFile()`。取消是“尽力而为”，某些数据可能仍然会被写入。

```mjs
import { writeFile } from 'node:fs/promises';
import { Buffer } from 'node:buffer';

try {
  const controller = new AbortController();
  const { signal } = controller;
  const data = new Uint8Array(Buffer.from('Hello Node.js'));
  const promise = writeFile('message.txt', data, { signal });

  // 在 Promise 结算之前中止请求。
  controller.abort();

  await promise;
} catch (err) {
  // 当请求被中止时 - err 是 AbortError
  console.error(err);
}
```

中止正在进行的请求不会中止单个操作系统请求，而是中止 `fs.writeFile` 执行的内部缓冲。

### `fsPromises.constants`

<!-- YAML
added:
  - v18.4.0
  - v16.17.0
-->

* 类型: {Object}

返回一个包含文件系统操作常用常量的对象。该对象与 `fs.constants` 相同。有关更多细节，请参见 [FS 常量][]。

## 回调 API

回调 API 异步执行所有操作，不阻塞事件循环，然后在完成或错误时调用回调函数。

回调 API 使用底层的 Node.js 线程池在事件循环线程之外执行文件系统操作。这些操作不是同步的，也不是线程安全的。在对同一文件执行多个并发修改时必须小心，否则可能发生数据损坏。

### `fs.access(path[, mode], callback)`

<!-- YAML
added: v0.11.15
changes:
  - version: v20.8.0
    pr-url: https://github.com/nodejs/node/pull/49683
    description: The constants `fs.F_OK`, `fs.R_OK`, `fs.W_OK` and `fs.X_OK`
                 which were present directly on `fs` are deprecated.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/6534
    description: The constants like `fs.R_OK`, etc which were present directly
                 on `fs` were moved into `fs.constants` as a soft deprecation.
                 Thus for Node.js `< v6.3.0` use `fs`
                 to access those constants, or
                 do something like `(fs.constants || fs).R_OK` to work with all
                 versions.
-->

* `path` {string|Buffer|URL}
* `mode` {integer} **默认值:** `fs.constants.F_OK`
* `callback` {Function}
  * `err` {Error}

测试用户对 `path` 指定的文件或目录的权限。`mode` 参数是一个可选的整数，指定要执行的可访问性检查。`mode` 应该是值 `fs.constants.F_OK` 或由 `fs.constants.R_OK`、`fs.constants.W_OK` 和 `fs.constants.X_OK` 中任何值的按位或组成的掩码（例如 `fs.constants.W_OK | fs.constants.R_OK`）。有关 `mode` 的可能值，请检查 [文件访问常量][]。

最后一个参数 `callback` 是一个回调函数，调用时可能带有错误参数。如果任何可访问性检查失败，错误参数将是一个 `Error` 对象。以下示例检查 `package.json` 是否存在，以及是否可读或可写。

```mjs
import { access, constants } from 'node:fs';

const file = 'package.json';

// 检查文件是否在当前目录中存在。
access(file, constants.F_OK, (err) => {
  console.log(`${file} ${err ? 'does not exist' : 'exists'}`);
});

// 检查文件是否可读。
access(file, constants.R_OK, (err) => {
  console.log(`${file} ${err ? 'is not readable' : 'is readable'}`);
});

// 检查文件是否可写。
access(file, constants.W_OK, (err) => {
  console.log(`${file} ${err ? 'is not writable' : 'is writable'}`);
});

// 检查文件是否可读和可写。
access(file, constants.R_OK | constants.W_OK, (err) => {
  console.log(`${file} ${err ? 'is not' : 'is'} readable and writable`);
});
```

不要使用 `fs.access()` 在调用 `fs.open()`、`fs.readFile()` 或 `fs.writeFile()` 之前检查文件的可访问性。这样做会引入竞争条件，因为其他进程可能会在两个调用之间更改文件的状态。相反，用户代码应直接打开/读取/写入文件，并处理如果文件不可访问时引发的错误。

**写入（不推荐）**

```mjs
import { access, open, close } from 'node:fs';

access('myfile', (err) => {
  if (!err) {
    console.error('myfile already exists');
    return;
  }

  open('myfile', 'wx', (err, fd) => {
    if (err) throw err;

    try {
      writeMyData(fd);
    } finally {
      close(fd, (err) => {
        if (err) throw err;
      });
    }
  });
});
```

**写入（推荐）**

```mjs
import { open, close } from 'node:fs';

open('myfile', 'wx', (err, fd) => {
  if (err) {
    if (err.code === 'EEXIST') {
      console.error('myfile already exists');
      return;
    }

    throw err;
  }

  try {
    writeMyData(fd);
  } finally {
    close(fd, (err) => {
      if (err) throw err;
    });
  }
});
```

**读取（不推荐）**

```mjs
import { access, open, close } from 'node:fs';
access('myfile', (err) => {
  if (err) {
    if (err.code === 'ENOENT') {
      console.error('myfile does not exist');
      return;
    }

    throw err;
  }

  open('myfile', 'r', (err, fd) => {
    if (err) throw err;

    try {
      readMyData(fd);
    } finally {
      close(fd, (err) => {
        if (err) throw err;
      });
    }
  });
});
```

**读取（推荐）**

```mjs
import { open, close } from 'node:fs';

open('myfile', 'r', (err, fd) => {
  if (err) {
    if (err.code === 'ENOENT') {
      console.error('myfile does not exist');
      return;
    }

    throw err;
  }

  try {
    readMyData(fd);
  } finally {
    close(fd, (err) => {
      if (err) throw err;
    });
  }
});
```

上面的“不推荐”示例检查可访问性，然后使用文件；“推荐”示例更好，因为它们直接使用文件并处理错误（如果有）。

通常，仅当文件不直接使用时才检查文件的可访问性，例如当其可访问性是来自另一个进程的信号时。

在 Windows 上，目录上的访问控制策略（ACL）可能会限制对文件或目录的访问。但是，`fs.access()` 函数不检查 ACL，因此即使 ACL 限制用户读取或写入，它也可能报告路径可访问。

### `fs.appendFile(path, data[, options], callback)`

<!-- YAML
added: v0.6.7
changes:
  - version:
    - v21.1.0
    - v20.10.0
    pr-url: https://github.com/nodejs/node/pull/50095
    description: The `flush` option is now supported.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7831
    description: The passed `options` object will never be modified.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/3163
    description: The `file` parameter can be a file descriptor now.
-->

* `path` {string|Buffer|URL|number} 文件名或文件描述符
* `data` {string|Buffer}
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `mode` {integer} **默认值:** `0o666`
  * `flag` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'a'`。
  * `flush` {boolean} 如果为 `true`，则在关闭底层文件描述符之前会刷新它。**默认值:** `false`。
* `callback` {Function}
  * `err` {Error}

异步将数据追加到文件，如果文件尚不存在则创建该文件。`data` 可以是字符串或 {Buffer}。

`mode` 选项仅影响新创建的文件。有关更多细节，请参见 [`fs.open()`][]。

```mjs
import { appendFile } from 'node:fs';

appendFile('message.txt', 'data to append', (err) => {
  if (err) throw err;
  console.log('The "data to append" was appended to file!');
});
```

如果 `options` 是字符串，则它指定编码：

```mjs
import { appendFile } from 'node:fs';

appendFile('message.txt', 'data to append', 'utf8', callback);
```

`path` 可以指定为已打开用于追加的数字文件描述符（使用 `fs.open()` 或 `fs.openSync()`）。文件描述符不会自动关闭。

```mjs
import { open, close, appendFile } from 'node:fs';

function closeFd(fd) {
  close(fd, (err) => {
    if (err) throw err;
  });
}

open('message.txt', 'a', (err, fd) => {
  if (err) throw err;

  try {
    appendFile(fd, 'data to append', 'utf8', (err) => {
      closeFd(fd);
      if (err) throw err;
    });
  } catch (err) {
    closeFd(fd);
    throw err;
  }
});
```

### `fs.chmod(path, mode, callback)`

<!-- YAML
added: v0.1.30
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `path` {string|Buffer|URL}
* `mode` {string|integer}
* `callback` {Function}
  * `err` {Error}

异步更改文件的权限。除了可能的异常外，不会向完成回调提供任何参数。

有关更多细节，请参阅 POSIX chmod(2) 文档。

```mjs
import { chmod } from 'node:fs';

chmod('my_file.txt', 0o775, (err) => {
  if (err) throw err;
  console.log('The permissions for file "my_file.txt" have been changed!');
});
```

#### 文件模式

`fs.chmod()` 和 `fs.chmodSync()` 方法中使用的 `mode` 参数是使用以下常量的逻辑或创建的数字位掩码：

| 常量 | 八进制 | 描述 |
| ---------------------- | ------- | ------------------------ |
| `fs.constants.S_IRUSR` | `0o400` | 所有者可读 |
| `fs.constants.S_IWUSR` | `0o200` | 所有者可写 |
| `fs.constants.S_IXUSR` | `0o100` | 所有者可执行/搜索 |
| `fs.constants.S_IRGRP` | `0o40` | 组可读 |
| `fs.constants.S_IWGRP` | `0o20` | 组可写 |
| `fs.constants.S_IXGRP` | `0o10` | 组可执行/搜索 |
| `fs.constants.S_IROTH` | `0o4` | 其他人可读 |
| `fs.constants.S_IWOTH` | `0o2` | 其他人可写 |
| `fs.constants.S_IXOTH` | `0o1` | 其他人可执行/搜索 |

构建 `mode` 的一种更简单的方法是使用三个八进制数字的序列（例如 `765`）。最左边的数字（示例中的 `7`）指定文件所有者的权限。中间的数字（示例中的 `6`）指定组的权限。最右边的数字（示例中的 `5`）指定其他人的权限。

| 数字 | 描述 |
| ------ | ------------------------ |
| `7` | 读、写和执行 |
| `6` | 读和写 |
| `5` | 读和执行 |
| `4` | 只读 |
| `3` | 写和执行 |
| `2` | 只写 |
| `1` | 只执行 |
| `0` | 无权限 |

例如，八进制值 `0o765` 表示：

* 所有者可以读、写和执行文件。
* 组可以读和写文件。
* 其他人可以读和执行文件。

在使用文件模式期望的原始数字时，任何大于 `0o777` 的值都可能导致特定于平台的行为，这些行为不支持一致工作。因此，像 `S_ISVTX`、`S_ISGID` 或 `S_ISUID` 这样的常量在 `fs.constants` 中未公开。

注意：在 Windows 上，只能更改写权限，并且组、所有者或其他人的权限之间的区别未实现。

### `fs.chown(path, uid, gid, callback)`

<!-- YAML
added: v0.1.97
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `path` {string|Buffer|URL}
* `uid` {integer}
* `gid` {integer}
* `callback` {Function}
  * `err` {Error}

异步更改文件的所有者和组。除了可能的异常外，不会向完成回调提供任何参数。

有关更多细节，请参阅 POSIX chown(2) 文档。

### `fs.close(fd[, callback])`

<!-- YAML
added: v0.0.2
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version:
      - v15.9.0
      - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/37174
    description: A default callback is now used if one is not provided.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `fd` {integer}
* `callback` {Function}
  * `err` {Error}

关闭文件描述符。除了可能的异常外，不会向完成回调提供任何参数。

在任何当前通过任何其他 `fs` 操作使用的文件描述符（`fd`）上调用 `fs.close()` 可能导致未定义的行为。

有关更多细节，请参阅 POSIX close(2) 文档。

### `fs.copyFile(src, dest[, mode], callback)`

<!-- YAML
added: v8.5.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/27044
    description: Changed `flags` argument to `mode` and imposed
                 stricter type validation.
-->

* `src` {string|Buffer|URL} 要复制的源文件名
* `dest` {string|Buffer|URL} 复制操作的目标文件名
* `mode` {integer} 复制操作的修饰符。**默认值:** `0`。
* `callback` {Function}
  * `err` {Error}

异步地将 `src` 复制到 `dest`。默认情况下，如果 `dest` 已存在，则会被覆盖。除了可能的异常外，不会向回调函数提供任何参数。Node.js 不保证复制操作的原子性。如果在目标文件已打开进行写入后发生错误，Node.js 将尝试删除目标文件。

`mode` 是一个可选整数，指定复制操作的行为。可以创建由两个或多个值的按位或组成的掩码（例如 `fs.constants.COPYFILE_EXCL | fs.constants.COPYFILE_FICLONE`）。

* `fs.constants.COPYFILE_EXCL`：如果 `dest` 已存在，复制操作将失败。
* `fs.constants.COPYFILE_FICLONE`：复制操作将尝试创建写时复制 reflink。如果平台不支持写时复制，则使用回退复制机制。
* `fs.constants.COPYFILE_FICLONE_FORCE`：复制操作将尝试创建写时复制 reflink。如果平台不支持写时复制，则操作将失败。

```mjs
import { copyFile, constants } from 'node:fs';

function callback(err) {
  if (err) throw err;
  console.log('source.txt was copied to destination.txt');
}

// 默认情况下，destination.txt 将被创建或覆盖。
copyFile('source.txt', 'destination.txt', callback);

// 通过使用 COPYFILE_EXCL，如果 destination.txt 存在，操作将失败。
copyFile('source.txt', 'destination.txt', constants.COPYFILE_EXCL, callback);
```

### `fs.cp(src, dest[, options], callback)`

<!-- YAML
added: v16.7.0
changes:
  - version: v22.3.0
    pr-url: https://github.com/nodejs/node/pull/53127
    description: This API is no longer experimental.
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/47084
    description: Accept an additional `mode` option to specify
                 the copy behavior as the `mode` argument of `fs.copyFile()`.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version:
    - v17.6.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/41819
    description: Accepts an additional `verbatimSymlinks` option to specify
                 whether to perform path resolution for symlinks.
-->

* `src` {string|URL} 要复制的源路径。
* `dest` {string|URL} 要复制到的目标路径。
* `options` {Object}
  * `dereference` {boolean} 取消引用符号链接。**默认值:** `false`。
  * `errorOnExist` {boolean} 当 `force` 为 `false` 且目标已存在时，抛出错误。**默认值:** `false`。
  * `filter` {Function} 过滤要复制的文件/目录的函数。返回 `true` 复制项目，`false` 忽略它。当忽略目录时，其所有内容也将被跳过。也可以返回一个解析为 `true` 或 `false` 的 `Promise` **默认值:** `undefined`。
    * `src` {string} 要复制的源路径。
    * `dest` {string} 要复制到的目标路径。
    * 返回: {boolean|Promise} 可强制转换为 `boolean` 的值或使用此类值履行的 `Promise`。
  * `force` {boolean} 覆盖现有文件或目录。如果将此设置为 false 且目标存在，复制操作将忽略错误。使用 `errorOnExist` 选项更改此行为。**默认值:** `true`。
  * `mode` {integer} 复制操作的修饰符。**默认值:** `0`。参见 [`fs.copyFile()`][] 的 `mode` 标志。
  * `preserveTimestamps` {boolean} 当为 `true` 时，将保留 `src` 的时间戳。**默认值:** `false`。
  * `recursive` {boolean} 递归复制目录 **默认值:** `false`
  * `verbatimSymlinks` {boolean} 当为 `true` 时，将跳过符号链接的路径解析。**默认值:** `false`
* `callback` {Function}
  * `err` {Error}

异步地将整个目录结构从 `src` 复制到 `dest`，包括子目录和文件。

当将一个目录复制到另一个目录时，不支持通配符，行为类似于 `cp dir1/ dir2/`。

### `fs.createReadStream(path[, options])`

<!-- YAML
added: v0.1.31
changes:
  - version: v16.10.0
    pr-url: https://github.com/nodejs/node/pull/40013
    description: The `fs` option does not need `open` method if an `fd` was provided.
  - version: v16.10.0
    pr-url: https://github.com/nodejs/node/pull/40013
    description: The `fs` option does not need `close` method if `autoClose` is `false`.
  - version: v15.5.0
    pr-url: https://github.com/nodejs/node/pull/36431
    description: Add support for `AbortSignal`.
  - version:
     - v15.4.0
    pr-url: https://github.com/nodejs/node/pull/35922
    description: The `fd` option accepts FileHandle arguments.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31408
    description: Change `emitClose` default to `true`.
  - version:
     - v13.6.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/29083
    description: The `fs` options allow overriding the used `fs`
                 implementation.
  - version: v12.10.0
    pr-url: https://github.com/nodejs/node/pull/29212
    description: Enable `emitClose` option.
  - version: v11.0.0
    pr-url: https://github.com/nodejs/node/pull/19898
    description: Impose new restrictions on `start` and `end`, throwing
                 more appropriate errors in cases when we cannot reasonably
                 handle the input values.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using
                 `file:` protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7831
    description: The passed `options` object will never be modified.
  - version: v2.3.0
    pr-url: https://github.com/nodejs/node/pull/1845
    description: The passed `options` object can be a string now.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `flags` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'r'`。
  * `encoding` {string} **默认值:** `null`
  * `fd` {integer|FileHandle} **默认值:** `null`
  * `mode` {integer} **默认值:** `0o666`
  * `autoClose` {boolean} **默认值:** `true`
  * `emitClose` {boolean} **默认值:** `true`
  * `start` {integer}
  * `end` {integer} **默认值:** `Infinity`
  * `highWaterMark` {integer} **默认值:** `64 * 1024`
  * `fs` {Object|null} **默认值:** `null`
  * `signal` {AbortSignal|null} **默认值:** `null`
* 返回: {fs.ReadStream}

`options` 可以包含 `start` 和 `end` 值，以从文件中读取一个字节范围而不是整个文件。`start` 和 `end` 都包含在内，从 0 开始计数，允许的值在 \[0, [`Number.MAX_SAFE_INTEGER`][]] 范围内。如果指定了 `fd` 并且省略或 `undefined` `start`，`fs.createReadStream()` 将从当前文件位置顺序读取。`encoding` 可以是 {Buffer} 接受的任何编码之一。

如果指定了 `fd`，`ReadStream` 将忽略 `path` 参数并使用指定的文件描述符。这意味着不会发出 `'open'` 事件。`fd` 应该是阻塞的；非阻塞 `fd` 应该传递给 {net.Socket}。

如果 `fd` 指向仅支持阻塞读取的字符设备（如键盘或声卡），则读取操作在数据可用之前不会完成。这可能会阻止进程退出和流自然关闭。

默认情况下，流在销毁后会发出 `'close'` 事件。将 `emitClose` 选项设置为 `false` 可以更改此行为。

通过提供 `fs` 选项，可以覆盖相应的 `fs` 实现以进行 `open`、`read` 和 `close`。当提供 `fs` 选项时，需要覆盖 `read`。如果未提供 `fd`，则还需要覆盖 `open`。如果 `autoClose` 为 `true`，则还需要覆盖 `close`。

```mjs
import { createReadStream } from 'node:fs';

// 从某个字符设备创建流。
const stream = createReadStream('/dev/input/event0');
setTimeout(() => {
  stream.close(); // 这可能不会关闭流。
  // 人工标记流结束，就好像底层资源自身指示了文件结束，允许流关闭。
  // 这不会取消待处理的读取操作，如果有这样的操作，进程可能仍然无法成功退出，直到它完成。
  stream.push(null);
  stream.read(0);
}, 100);
```

如果 `autoClose` 为 false，那么即使有错误，文件描述符也不会关闭。应用程序有责任关闭它并确保没有文件描述符泄漏。如果 `autoClose` 设置为 true（默认行为），在 `'error'` 或 `'end'` 时，文件描述符将自动关闭。

`mode` 设置文件模式（权限和粘滞位），但仅当文件被创建时。

读取一个 100 字节长的文件的最后 10 字节的示例：

```mjs
import { createReadStream } from 'node:fs';

createReadStream('sample.txt', { start: 90, end: 99 });
```

如果 `options` 是字符串，则它指定编码。

### `fs.createWriteStream(path[, options])`

<!-- YAML
added: v0.1.31
changes:
  - version:
    - v21.0.0
    - v20.10.0
    pr-url: https://github.com/nodejs/node/pull/50093
    description: The `flush` option is now supported.
  - version: v16.10.0
    pr-url: https://github.com/nodejs/node/pull/40013
    description: The `fs` option does not need `open` method if an `fd` was provided.
  - version: v16.10.0
    pr-url: https://github.com/nodejs/node/pull/40013
    description: The `fs` option does not need `close` method if `autoClose` is `false`.
  - version: v15.5.0
    pr-url: https://github.com/nodejs/node/pull/36431
    description: Add support for `AbortSignal`.
  - version:
     - v15.4.0
    pr-url: https://github.com/nodejs/node/pull/35922
    description: The `fd` option accepts FileHandle arguments.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31408
    description: Change `emitClose` default to `true`.
  - version:
     - v13.6.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/29083
    description: The `fs` options allow overriding the used `fs`
                 implementation.
  - version: v12.10.0
    pr-url: https://github.com/nodejs/node/pull/29212
    description: Enable `emitClose` option.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using
                 `file:` protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7831
    description: The passed `options` object will never be modified.
  - version: v5.5.0
    pr-url: https://github.com/nodejs/node/pull/3679
    description: The `autoClose` option is supported now.
  - version: v2.3.0
    pr-url: https://github.com/nodejs/node/pull/1845
    description: The passed `options` object can be a string now.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `flags` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'w'`。
  * `encoding` {string} **默认值:** `'utf8'`
  * `fd` {integer|FileHandle} **默认值:** `null`
  * `mode` {integer} **默认值:** `0o666`
  * `autoClose` {boolean} **默认值:** `true`
  * `emitClose` {boolean} **默认值:** `true`
  * `start` {integer}
  * `fs` {Object|null} **默认值:** `null`
  * `signal` {AbortSignal|null} **默认值:** `null`
  * `highWaterMark` {number} **默认值:** `16384`
  * `flush` {boolean} 如果为 `true`，则在关闭底层文件描述符之前会刷新它。**默认值:** `false`。
* 返回: {fs.WriteStream}

`options` 还可以包含 `start` 选项，以允许在文件开头之后的某个位置写入数据，允许的值在 \[0, [`Number.MAX_SAFE_INTEGER`][]] 范围内。修改文件而不是替换它可能需要将 `flags` 选项设置为 `r+` 而不是默认的 `w`。`encoding` 可以是 {Buffer} 接受的任何编码之一。

如果 `autoClose` 设置为 true（默认行为），在 `'error'` 或 `'finish'` 时，文件描述符将自动关闭。如果 `autoClose` 为 false，那么即使有错误，文件描述符也不会关闭。应用程序有责任关闭它并确保没有文件描述符泄漏。

默认情况下，流在销毁后会发出 `'close'` 事件。将 `emitClose` 选项设置为 `false` 可以更改此行为。

通过提供 `fs` 选项，可以覆盖相应的 `fs` 实现以进行 `open`、`write`、`writev` 和 `close`。覆盖 `write()` 而不覆盖 `writev()` 会降低性能，因为某些优化（`_writev()`）将被禁用。当提供 `fs` 选项时，至少需要覆盖 `write` 和 `writev` 之一。如果未提供 `fd` 选项，则还需要覆盖 `open`。如果 `autoClose` 为 `true`，则还需要覆盖 `close`。

与 {fs.ReadStream} 类似，如果指定了 `fd`，{fs.WriteStream} 将忽略 `path` 参数并使用指定的文件描述符。这意味着不会发出 `'open'` 事件。`fd` 应该是阻塞的；非阻塞 `fd` 应该传递给 {net.Socket}。

如果 `options` 是字符串，则它指定编码。

### `fs.exists(path, callback)`

<!-- YAML
added: v0.0.2
deprecated: v1.0.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using
                 `file:` protocol.
-->

> Stability: 0 - Deprecated: Use [`fs.stat()`][] or [`fs.access()`][] instead.

* `path` {string|Buffer|URL}
* `callback` {Function}
  * `exists` {boolean}

通过检查文件系统来测试给定 `path` 的元素是否存在。然后使用 true 或 false 调用 `callback` 参数：

```mjs
import { exists } from 'node:fs';

exists('/etc/passwd', (e) => {
  console.log(e ? 'it exists' : 'no passwd!');
});
```

**此回调的参数与其他 Node.js 回调不一致。** 通常，Node.js 回调的第一个参数是 `err` 参数，可选地后跟其他参数。`fs.exists()` 回调只有一个布尔参数。这是推荐使用 `fs.access()` 而不是 `fs.exists()` 的原因之一。

如果 `path` 是符号链接，则跟踪它。因此，如果 `path` 存在但指向不存在的元素，回调将收到值 `false`。

不推荐在调用 `fs.open()`、`fs.readFile()` 或 `fs.writeFile()` 之前使用 `fs.exists()` 检查文件的存在性。这样做会引入竞争条件，因为其他进程可能会在两个调用之间更改文件的状态。相反，用户代码应直接打开/读取/写入文件，并处理如果文件不存在时引发的错误。

**写入（不推荐）**

```mjs
import { exists, open, close } from 'node:fs';

exists('myfile', (e) => {
  if (e) {
    console.error('myfile already exists');
  } else {
    open('myfile', 'wx', (err, fd) => {
      if (err) throw err;

      try {
        writeMyData(fd);
      } finally {
        close(fd, (err) => {
          if (err) throw err;
        });
      }
    });
  }
});
```

**写入（推荐）**

```mjs
import { open, close } from 'node:fs';
open('myfile', 'wx', (err, fd) => {
  if (err) {
    if (err.code === 'EEXIST') {
      console.error('myfile already exists');
      return;
    }

    throw err;
  }

  try {
    writeMyData(fd);
  } finally {
    close(fd, (err) => {
      if (err) throw err;
    });
  }
});
```

**读取（不推荐）**

```mjs
import { open, close, exists } from 'node:fs';

exists('myfile', (e) => {
  if (e) {
    open('myfile', 'r', (err, fd) => {
      if (err) throw err;

      try {
        readMyData(fd);
      } finally {
        close(fd, (err) => {
          if (err) throw err;
        });
      }
    });
  } else {
    console.error('myfile does not exist');
  }
});
```

**读取（推荐）**

```mjs
import { open, close } from 'node:fs';

open('myfile', 'r', (err, fd) => {
  if (err) {
    if (err.code === 'ENOENT') {
      console.error('myfile does not exist');
      return;
    }

    throw err;
  }

  try {
    readMyData(fd);
  } finally {
    close(fd, (err) => {
      if (err) throw err;
    });
  }
});
```

上面的“不推荐”示例检查存在性，然后使用文件；“推荐”示例更好，因为它们直接使用文件并处理错误（如果有）。

通常，仅当文件不直接使用时才检查文件的存在性，例如当其存在性是来自另一个进程的信号时。

### `fs.fchmod(fd, mode, callback)`

<!-- YAML
added: v0.4.7
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `fd` {integer}
* `mode` {string|integer}
* `callback` {Function}
  * `err` {Error}

设置文件的权限。除了可能的异常外，不会向完成回调提供任何参数。

有关更多细节，请参阅 POSIX fchmod(2) 文档。

### `fs.fchown(fd, uid, gid, callback)`

<!-- YAML
added: v0.4.7
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `fd` {integer}
* `uid` {integer}
* `gid` {integer}
* `callback` {Function}
  * `err` {Error}

设置文件的所有者。除了可能的异常外，不会向完成回调提供任何参数。

有关更多细节，请参阅 POSIX fchown(2) 文档。

### `fs.fdatasync(fd, callback)`

<!-- YAML
added: v0.1.96
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `fd` {integer}
* `callback` {Function}
  * `err` {Error}

强制所有当前与文件关联的排队 I/O 操作到操作系统的同步 I/O 完成状态。有关详细信息，请参阅 POSIX fdatasync(2) 文档。除了可能的异常外，不会向完成回调提供任何参数。

### `fs.fstat(fd[, options], callback)`

<!-- YAML
added: v0.1.95
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.5.0
    pr-url: https://github.com/nodejs/node/pull/20220
    description: Accepts an additional `options` object to specify whether
                 the numeric values returned should be bigint.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `fd` {integer}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.Stats} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* `callback` {Function}
  * `err` {Error}
  * `stats` {fs.Stats}

使用文件描述符调用回调并返回 {fs.Stats}。

有关更多细节，请参阅 POSIX fstat(2) 文档。

### `fs.fsync(fd, callback)`

<!-- YAML
added: v0.1.96
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `fd` {integer}
* `callback` {Function}
  * `err` {Error}

请求将打开文件描述符的所有数据刷新到存储设备。具体实现取决于操作系统和设备。有关更多细节，请参阅 POSIX fsync(2) 文档。除了可能的异常外，不会向完成回调提供任何参数。

### `fs.ftruncate(fd[, len], callback)`

<!-- YAML
added: v0.8.6
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `fd` {integer}
* `len` {integer} **默认值:** `0`
* `callback` {Function}
  * `err` {Error}

截断文件描述符。除了可能的异常外，不会向完成回调提供任何参数。

有关更多细节，请参阅 POSIX ftruncate(2) 文档。

如果文件描述符引用的文件大于 `len` 字节，则仅保留文件中的前 `len` 字节。

例如，以下程序仅保留文件的前四个字节：

```mjs
import { open, close, ftruncate } from 'node:fs';

function closeFd(fd) {
  close(fd, (err) => {
    if (err) throw err;
  });
}

open('temp.txt', 'r+', (err, fd) => {
  if (err) throw err;

  try {
    ftruncate(fd, 4, (err) => {
      closeFd(fd);
      if (err) throw err;
    });
  } catch (err) {
    closeFd(fd);
    if (err) throw err;
  }
});
```

如果文件先前短于 `len` 字节，则会被扩展，扩展部分用空字节（`'\0'`）填充：

如果 `len` 为负数，则将使用 `0`。

### `fs.futimes(fd, atime, mtime, callback)`

<!-- YAML
added: v0.4.2
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v4.1.0
    pr-url: https://github.com/nodejs/node/pull/2387
    description: Numeric strings, `NaN`, and `Infinity` are now allowed
                 time specifiers.
-->

* `fd` {integer}
* `atime` {number|string|Date}
* `mtime` {number|string|Date}
* `callback` {Function}
  * `err` {Error}

更改由提供的文件描述符引用的对象的文件系统时间戳。参见 [`fs.utimes()`][]。

### `fs.glob(pattern[, options], callback)`

<!-- YAML
added: v22.0.0
changes:
  - version: v24.1.0
    pr-url: https://github.com/nodejs/node/pull/58182
    description: Add support for `URL` instances for `cwd` option.
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57513
    description: Marking the API stable.
  - version:
    - v23.7.0
    - v22.14.0
    pr-url: https://github.com/nodejs/node/pull/56489
    description: Add support for `exclude` option to accept glob patterns.
  - version: v22.2.0
    pr-url: https://github.com/nodejs/node/pull/52837
    description: Add support for `withFileTypes` as an option.
-->

* `pattern` {string|string\[]}

* `options` {Object}
  * `cwd` {string|URL} 当前工作目录。**默认值:** `process.cwd()`
  * `exclude` {Function|string\[]} 过滤掉文件/目录的函数或要排除的全局模式列表。如果提供了函数，返回 `true` 排除项目，`false` 包含它。**默认值:** `undefined`。
  * `withFileTypes` {boolean} 如果为 `true`，全局应返回路径作为 Dirent，否则为 `false`。**默认值:** `false`。

* `callback` {Function}
  * `err` {Error}

* 检索匹配指定模式的文件。

```mjs
import { glob } from 'node:fs';

glob('**/*.js', (err, matches) => {
  if (err) throw err;
  console.log(matches);
});
```

```cjs
const { glob } = require('node:fs');

glob('**/*.js', (err, matches) => {
  if (err) throw err;
  console.log(matches);
});
```

### `fs.lchmod(path, mode, callback)`

<!-- YAML
deprecated: v0.4.7
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37460
    description: The error returned may be an `AggregateError` if more than one
                 error is returned.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

> Stability: 0 - Deprecated

* `path` {string|Buffer|URL}
* `mode` {integer}
* `callback` {Function}
  * `err` {Error|AggregateError}

更改符号链接的权限。除了可能的异常外，不会向完成回调提供任何参数。

此方法仅在 macOS 上实现。

有关更多细节，请参阅 POSIX lchmod(2) 文档。

### `fs.lchown(path, uid, gid, callback)`

<!-- YAML
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.6.0
    pr-url: https://github.com/nodejs/node/pull/21498
    description: This API is no longer deprecated.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v0.4.7
    description: Documentation-only deprecation.
-->

* `path` {string|Buffer|URL}
* `uid` {integer}
* `gid` {integer}
* `callback` {Function}
  * `err` {Error}

设置符号链接的所有者。除了可能的异常外，不会向完成回调提供任何参数。

有关更多细节，请参阅 POSIX lchown(2) 文档。

### `fs.lutimes(path, atime, mtime, callback)`

<!-- YAML
added:
  - v14.5.0
  - v12.19.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `path` {string|Buffer|URL}
* `atime` {number|string|Date}
* `mtime` {number|string|Date}
* `callback` {Function}
  * `err` {Error}

以与 [`fs.utimes()`][] 相同的方式更改文件的访问和修改时间，不同之处在于如果路径引用符号链接，则不会取消引用该链接：而是更改符号链接本身的时间戳。

除了可能的异常外，不会向完成回调提供任何参数。

### `fs.link(existingPath, newPath, callback)`

<!-- YAML
added: v0.1.31
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `existingPath` and `newPath` parameters can be WHATWG
                 `URL` objects using `file:` protocol. Support is currently
                 still *experimental*.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `existingPath` {string|Buffer|URL}
* `newPath` {string|Buffer|URL}
* `callback` {Function}
  * `err` {Error}

从 `existingPath` 创建到 `newPath` 的新链接。有关更多细节，请参阅 POSIX link(2) 文档。除了可能的异常外，不会向完成回调提供任何参数。

### `fs.lstat(path[, options], callback)`

<!-- YAML
added: v0.1.30
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.5.0
    pr-url: https://github.com/nodejs/node/pull/20220
    description: Accepts an additional `options` object to specify whether
                 the numeric values returned should be bigint.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.Stats} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* `callback` {Function}
  * `err` {Error}
  * `stats` {fs.Stats}

检索 `path` 引用的符号链接的 {fs.Stats}。回调获得两个参数 `(err, stats)`，其中 `stats` 是 {fs.Stats} 对象。`lstat()` 与 `stat()` 相同，除了如果 `path` 是符号链接，则链接本身是 stat-ed，而不是它引用的文件。

有关更多细节，请参阅 POSIX lstat(2) 文档。

### `fs.mkdir(path[, options], callback)`

<!-- YAML
added: v0.1.8
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version:
     - v13.11.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/31530
    description: In `recursive` mode, the callback now receives the first
                 created path as an argument.
  - version: v10.12.0
    pr-url: https://github.com/nodejs/node/pull/21875
    description: The second argument can now be an `options` object with
                 `recursive` and `mode` properties.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `path` {string|Buffer|URL}
* `options` {Object|integer}
  * `recursive` {boolean} **默认值:** `false`
  * `mode` {string|integer} 在 Windows 上不支持。**默认值:** `0o777`。
* `callback` {Function}
  * `err` {Error}
  * `path` {string|undefined} 仅当使用 `recursive` 设置为 `true` 创建目录时存在。

异步创建目录。

回调获得一个可能的异常，如果 `recursive` 为 `true`，则获得第一个创建的目录路径，`(err[, path])`。当 `recursive` 为 `true` 时，如果未创建目录（例如，如果之前创建了），`path` 仍然可以是 `undefined`。

可选的 `options` 参数可以是指定 `mode`（权限和粘滞位）的整数，或者是具有 `mode` 属性和 `recursive` 属性的对象，指示是否应创建父目录。当 `path` 是已存在的目录时，调用 `fs.mkdir()` 仅当 `recursive` 为 false 时会导致错误。如果 `recursive` 为 false 且目录存在，会发生 `EEXIST` 错误。

```mjs
import { mkdir } from 'node:fs';

// 创建 ./tmp/a/apple，无论 ./tmp 和 ./tmp/a 是否存在。
mkdir('./tmp/a/apple', { recursive: true }, (err) => {
  if (err) throw err;
});
```

在 Windows 上，即使在根目录上使用 `fs.mkdir()` 递归也会导致错误：

```mjs
import { mkdir } from 'node:fs';

mkdir('/', { recursive: true }, (err) => {
  // => [Error: EPERM: operation not permitted, mkdir 'C:\']
});
```

有关更多细节，请参阅 POSIX mkdir(2) 文档。

### `fs.mkdtemp(prefix[, options], callback)`

<!-- YAML
added: v5.10.0
changes:
  - version:
    - v20.6.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/48828
    description: The `prefix` parameter now accepts buffers and URL.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version:
      - v16.5.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/39028
    description: The `prefix` parameter now accepts an empty string.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v6.2.1
    pr-url: https://github.com/nodejs/node/pull/6828
    description: The `callback` parameter is optional now.
-->

* `prefix` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* `callback` {Function}
  * `err` {Error}
  * `directory` {string}

创建唯一的临时目录。

生成六个随机字符附加到所需的 `prefix` 后面以创建唯一的临时目录。由于平台不一致，避免在 `prefix` 中使用尾随 `X` 字符。某些平台，特别是 BSD，可以返回超过六个随机字符，并用随机字符替换 `prefix` 中的尾随 `X` 字符。

创建的目录路径作为字符串传递给回调的第二个参数。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定要使用的字符编码。

```mjs
import { mkdtemp } from 'node:fs';
import { join } from 'node:path';
import { tmpdir } from 'node:os';

mkdtemp(join(tmpdir(), 'foo-'), (err, directory) => {
  if (err) throw err;
  console.log(directory);
  // 打印: /tmp/foo-itXde2 或 C:\Users\...\AppData\Local\Temp\foo-itXde2
});
```

`fs.mkdtemp()` 方法将直接将六个随机选择的字符附加到 `prefix` 字符串。例如，给定目录 `/tmp`，如果意图是在 `/tmp` 内创建临时目录，则 `prefix` 必须以尾随的平台特定路径分隔符（`require('node:path').sep`）结尾。

```mjs
import { tmpdir } from 'node:os';
import { mkdtemp } from 'node:fs';

// 新临时目录的父目录
const tmpDir = tmpdir();

// 此方法是 *不正确* 的：
mkdtemp(tmpDir, (err, directory) => {
  if (err) throw err;
  console.log(directory);
  // 将打印类似 `/tmpabc123` 的内容。
  // 在文件系统根目录而不是 /tmp 目录内创建新的临时目录。
});

// 此方法是 *正确* 的：
import { sep } from 'node:path';
mkdtemp(`${tmpDir}${sep}`, (err, directory) => {
  if (err) throw err;
  console.log(directory);
  // 将打印类似 `/tmp/abc123` 的内容。
  // 在 /tmp 目录内创建新的临时目录。
});
```

### `fs.open(path[, flags[, mode]], callback)`

<!-- YAML
added: v0.0.2
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v11.1.0
    pr-url: https://github.com/nodejs/node/pull/23767
    description: The `flags` argument is now optional and defaults to `'r'`.
  - version: v9.9.0
    pr-url: https://github.com/nodejs/node/pull/18801
    description: The `as` and `as+` flags are supported now.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `flags` {string|number} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'r'`。
* `mode` {string|integer} **默认值:** `0o666`（可读和可写）
* `callback` {Function}
  * `err` {Error}
  * `fd` {integer}

异步文件打开。有关更多细节，请参阅 POSIX open(2) 文档。

`mode` 设置文件模式（权限和粘滞位），但仅当文件被创建时。在 Windows 上，只能操作写权限；参见 [`fs.chmod()`][]。

回调获得两个参数 `(err, fd)`。

在 Windows 上，某些字符（`< > : " / \ | ? *`）是保留的，如 [命名文件、路径和命名空间][] 所述。在 NTFS 下，如果文件名包含冒号，Node.js 将打开一个文件系统流，如 [此 MSDN 页面][MSDN-Using-Streams] 所述。

基于 `fs.open()` 的函数也表现出此行为：`fs.writeFile()`、`fs.readFile()` 等。

### `fs.openAsBlob(path[, options])`

<!-- YAML
added: v19.8.0
changes:
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57513
    description: Marking the API stable.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `type` {string} Blob 的可选 MIME 类型。
* 返回: {Promise} 成功时使用 {Blob} 兑现。

返回一个 {Blob}，其数据由给定文件支持。

创建 {Blob} 后不得修改文件。任何修改将导致读取 {Blob} 数据失败并返回 `DOMException` 错误。在创建 `Blob` 时以及每次读取之前对文件进行同步 stat 操作，以检测文件数据是否在磁盘上被修改。

```mjs
import { openAsBlob } from 'node:fs';

const blob = await openAsBlob('the.file.txt');
const ab = await blob.arrayBuffer();
blob.stream();
```

```cjs
const { openAsBlob } = require('node:fs');

(async () => {
  const blob = await openAsBlob('the.file.txt');
  const ab = await blob.arrayBuffer();
  blob.stream();
})();
```

### `fs.opendir(path[, options], callback)`

<!-- YAML
added: v12.12.0
changes:
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/41439
    description: Added `recursive` option.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version:
     - v13.1.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30114
    description: The `bufferSize` option was introduced.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `bufferSize` {number} 从目录读取时内部缓冲的目录条目数。较高的值导致更好的性能但更高的内存使用。**默认值:** `32`
  * `recursive` {boolean} 解析的 {fs.Dir} 将是一个包含所有子文件和目录的 {AsyncIterable}。**默认值:** `false`
* `callback` {Function}
  * `err` {Error}
  * `dir` {fs.Dir}

异步打开目录以进行迭代扫描。有关更多细节，请参阅 POSIX opendir(3) 文档。

创建一个 {fs.Dir}，其中包含所有用于从目录读取和清理的进一步函数。

`encoding` 选项在打开目录及后续读取操作时设置 `path` 的编码。

回调获得两个参数 `(err, dir)`。

### `fs.read(fd, buffer, offset, length, position, callback)`

<!-- YAML
added: v0.0.2
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/42835
    description: Accepts bigint values as `position`.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `buffer` parameter won't coerce unsupported input to
                 buffers anymore.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.4.0
    pr-url: https://github.com/nodejs/node/pull/10382
    description: The `position` parameter is optional now.
  - version: v7.2.0
    pr-url: https://github.com/nodejs/node/pull/7856
    description: The `offset` and `length` parameters are optional now.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/4518
    description: The `length` parameter can now be `0`.
-->

* `fd` {integer}
* `buffer` {Buffer|TypedArray|DataView} 将用读取的文件数据填充的缓冲区。
* `offset` {integer} `buffer` 中开始填充的位置。
* `length` {integer} 要读取的字节数。
* `position` {integer|bigint|null} 从文件中开始读取数据的位置。如果 `null` 或 `-1`，将从当前文件位置读取数据，并且位置将被更新。如果 `position` 是非负整数，则当前文件位置将保持不变。
* `callback` {Function}
  * `err` {Error}
  * `bytesRead` {integer}
  * `buffer` {Buffer}

从 `fd` 指定的文件中读取数据。

回调获得三个参数 `(err, bytesRead, buffer)`。

如果文件没有被并发修改，当读取的字节数为零时达到文件末尾。

如果此方法作为其 [`util.promisify()`][] 版本调用，则返回一个具有 `bytesRead` 和 `buffer` 属性的 `Object` 的 promise。

```mjs
import { read } from 'node:fs';

function readFileContents(fd, buf, bytesToRead, position, callback) {
  read(fd, buf, 0, bytesToRead, position, (err, bytesRead, buffer) => {
    if (err) {
      console.error('Error reading file:', err);
      return;
    }
    console.log(`Read ${bytesRead} bytes from file:`, buffer.toString());
    callback();
  });
}
```

### `fs.read(fd, [options,] callback)`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/42835
    description: Accepts bigint values as `position`.
-->

* `fd` {integer}
* `options` {Object}
  * `buffer` {Buffer|TypedArray|DataView} **默认值:** `Buffer.alloc(16384)`
  * `offset` {integer} **默认值:** `0`
  * `length` {integer} **默认值:** `buffer.byteLength - offset`
  * `position` {integer|bigint|null} **默认值:** `null`
* `callback` {Function}
  * `err` {Error}
  * `bytesRead` {integer}
  * `buffer` {Buffer}

类似于上面的 `fs.read` 函数，此版本接受一个可选的 `options` 对象。如果未指定 `options` 对象，它将使用上述值默认。

### `fs.read(fd, buffer[, options], callback)`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/42835
    description: Accepts bigint values as `position`.
-->

* `fd` {integer}
* `buffer` {Buffer|TypedArray|DataView} 将用读取的文件数据填充的缓冲区。
* `options` {Object}
  * `offset` {integer} **默认值:** `0`
  * `length` {integer} **默认值:** `buffer.byteLength - offset`
  * `position` {integer|bigint|null} **默认值:** `null`
* `callback` {Function}
  * `err` {Error}
  * `bytesRead` {integer}
  * `buffer` {Buffer}

类似于上面的 `fs.read` 函数，此版本接受一个可选的 `options` 对象。如果未指定 `options` 对象，它将使用上述值默认。

### `fs.readdir(path[, options], callback)`

<!-- YAML
added: v0.1.8
changes:
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/41439
    description: Added `recursive` option.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.10.0
    pr-url: https://github.com/nodejs/node/pull/22020
    description: New option `withFileTypes` was added.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/5616
    description: The `options` parameter was added.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
  * `withFileTypes` {boolean} **默认值:** `false`
  * `recursive` {boolean} 如果为 `true`，则递归读取目录的内容。在递归模式下，它将列出所有文件、子文件和目录。**默认值:** `false`。
* `callback` {Function}
  * `err` {Error}
  * `files` {string\[]|Buffer\[]|fs.Dirent\[]}

异步读取目录的内容。回调获得两个参数 `(err, files)`，其中 `files` 是目录中文件的名称数组，不包括 `'.'` 和 `'..'`。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定用于文件名的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的文件名将作为 {Buffer} 对象传递。

如果 `options.withFileTypes` 设置为 `true`，`files` 数组将包含 {fs.Dirent} 对象。

```mjs
import { readdir } from 'node:fs';

readdir('path/to/directory', (err, files) => {
  if (err) throw err;
  console.log(files);
});
```

### `fs.readFile(path[, options], callback)`

<!-- YAML
added: v0.1.29
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version:
    - v15.2.0
    - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/35911
    description: The options argument may include an AbortSignal to abort an
                 ongoing readFile request.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v5.1.0
    pr-url: https://github.com/nodejs/node/pull/3740
    description: The `callback` will always be called with `null` as the `error`
                 parameter in case of success.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/3163
    description: The `path` parameter can be a file descriptor now.
-->

* `path` {string|Buffer|URL|integer} 文件名或文件描述符
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `null`
  * `flag` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'r'`。
  * `signal` {AbortSignal} 允许中止正在进行的 readFile
* `callback` {Function}
  * `err` {Error}
  * `data` {string|Buffer}

异步读取文件的全部内容。

```mjs
import { readFile } from 'node:fs';

readFile('/etc/passwd', (err, data) => {
  if (err) throw err;
  console.log(data);
});
```

回调获得两个参数 `(err, data)`，其中 `data` 是文件的内容。

如果未指定编码，则返回原始缓冲区。

如果 `options` 是字符串，则它指定编码：

```mjs
import { readFile } from 'node:fs';

readFile('/etc/passwd', 'utf8', callback);
```

当 `path` 是目录时，`fs.readFile()` 的行为是特定于平台的。在 macOS、Linux 和 Windows 上，回调将收到错误。在 FreeBSD 上，将返回目录内容的表示。

此函数可以中止，使用 `AbortSignal`。如果请求被中止，回调将使用 `AbortError` 调用：

```mjs
import { readFile } from 'node:fs';

const controller = new AbortController();
const { signal } = controller;
readFile(fileInfo[0].name, { signal }, (err, buf) => {
  // ...
});
// 当您想要中止请求时
controller.abort();
```

中止正在进行的请求不会中止单个操作系统请求，而是中止 `fs.readFile` 执行的内部缓冲。

任何指定的 {FileHandle} 必须支持读取。

### `fs.readlink(path[, options], callback)`

<!-- YAML
added: v0.1.31
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* `callback` {Function}
  * `err` {Error}
  * `linkString` {string|Buffer}

读取 `path` 引用的符号链接的内容。回调获得两个参数 `(err, linkString)`。

有关更多细节，请参阅 POSIX readlink(2) 文档。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定返回的链接路径的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的链接路径将作为 {Buffer} 对象传递。

### `fs.readv(fd, buffers[, position], callback)`

<!-- YAML
added:
  - v13.13.0
  - v12.17.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `fd` {integer}
* `buffers` {ArrayBufferView\[]}
* `position` {integer|null} 从文件开头开始读取数据的偏移量。如果 `position` 不是 `number`，数据将从当前位置读取。**默认值:** `null`
* `callback` {Function}
  * `err` {Error}
  * `bytesRead` {integer}
  * `buffers` {ArrayBufferView\[]}

从 `fd` 指定的文件中读取，并使用 `readv()` 写入 `ArrayBufferView` 数组。

`position` 是从文件开头开始读取数据的偏移量。如果 `typeof position !== 'number'`，数据将从当前位置读取。

回调将获得三个参数：`err`、`bytesRead` 和 `buffers`。`bytesRead` 是从文件中读取的字节数。

如果此方法作为其 [`util.promisify()`][] 版本调用，则返回一个具有 `bytesRead` 和 `buffers` 属性的 `Object` 的 promise。

### `fs.realpath(path[, options], callback)`

<!-- YAML
added: v0.1.31
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v6.4.0
    pr-url: https://github.com/nodejs/node/pull/7899
    description: Calling `realpath` now works again for various edge cases
                 on Windows.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/3594
    description: The `cache` parameter was removed.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* `callback` {Function}
  * `err` {Error}
  * `resolvedPath` {string|Buffer}

使用与 `fs.realpath.native()` 函数相同的语义计算 `path` 的实际位置。

仅支持可以转换为 UTF8 字符串的路径。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定用于传递给回调的路径的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的路径将作为 {Buffer} 对象传递。

### `fs.realpath.native(path[, options], callback)`

<!-- YAML
added: v9.2.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* `callback` {Function}
  * `err` {Error}
  * `resolvedPath` {string|Buffer}

异步的 realpath(3)。

`callback` 获得两个参数 `(err, resolvedPath)`。

仅支持可以转换为 UTF8 字符串的路径。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定用于传递给回调的路径的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的路径将作为 {Buffer} 对象传递。

### `fs.rename(oldPath, newPath, callback)`

<!-- YAML
added: v0.0.2
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `oldPath` and `newPath` parameters can be WHATWG `URL`
                 objects using `file:` protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `oldPath` {string|Buffer|URL}
* `newPath` {string|Buffer|URL}
* `callback` {Function}
  * `err` {Error}

异步地将 `oldPath` 处的文件重命名为 `newPath` 提供的文件名。如果 `newPath` 已存在，则将被覆盖。如果 `newPath` 是目录，则会出现错误。除了可能的异常外，不会向完成回调提供任何参数。

另请参阅：rename(2)。

```mjs
import { rename } from 'node:fs';

rename('oldFile.txt', 'newFile.txt', (err) => {
  if (err) throw err;
  console.log('Rename complete!');
});
```

### `fs.rmdir(path[, options], callback)`

<!-- YAML
added: v0.0.2
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37216
    description: "Using `fs.rmdir(path, { recursive: true })` on a `path`
                 that is a file is no longer permitted and results in an
                 `ENOENT` error on Windows and an `ENOTDIR` error on POSIX."
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37216
    description: "Using `fs.rmdir(path, { recursive: true })` on a `path`
                 that does not exist is no longer permitted and results in a
                 `ENOENT` error."
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37302
    description: The `recursive` option is deprecated, using it triggers a
                 deprecation warning.
  - version: v14.14.0
    pr-url: https://github.com/nodejs/node/pull/35579
    description: The `recursive` option is deprecated, use `fs.rm` instead.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version:
     - v13.3.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30644
    description: The `maxBusyTries` option is renamed to `maxRetries`, and its
                 default is 0. The `emfileWait` option has been removed, and
                 `EMFILE` errors use the same retry logic as other errors. The
                 `retryDelay` option is now supported. `ENFILE` errors are now
                 retried.
  - version: v12.10.0
    pr-url: https://github.com/nodejs/node/pull/29168
    description: The `recursive`, `maxBusyTries`, and `emfileWait` options are
                  now supported.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `maxRetries` {integer} 如果遇到 `EBUSY`、`EMFILE`、`ENFILE`、`ENOTEMPTY` 或 `EPERM` 错误，Node.js 将以每次重试线性退避等待 `retryDelay` 毫秒的方式重试操作。此选项表示重试次数。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `0`。
  * `recursive` {boolean} 如果为 `true`，则执行递归目录移除。在递归模式下，操作会在失败时重试。**默认值:** `false`。**已弃用。**
  * `retryDelay` {integer} 重试之间等待的时间量（毫秒）。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `100`。
* `callback` {Function}
  * `err` {Error}

异步的 rmdir(2)。除了可能的异常外，不会向完成回调提供任何参数。

在文件（非目录）上使用 `fs.rmdir()` 会导致在 Windows 上回调错误 `ENOENT`，在 POSIX 上回调错误 `ENOTDIR`。要获得类似于 `rm -rf` Unix 命令的行为，请使用 [`fs.rm()`][] 并设置选项 `{ recursive: true, force: true }`。

### `fs.rm(path[, options], callback)`

<!-- YAML
added: v14.14.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `force` {boolean} 当为 `true` 时，如果 `path` 不存在，异常将被忽略。**默认值:** `false`。
  * `maxRetries` {integer} 如果遇到 `EBUSY`、`EMFILE`、`ENFILE`、`ENOTEMPTY` 或 `EPERM` 错误，Node.js 将以每次重试线性退避等待 `retryDelay` 毫秒的方式重试操作。此选项表示重试次数。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `0`。
  * `recursive` {boolean} 如果为 `true`，则执行递归目录移除。在递归模式下，操作会在失败时重试。**默认值:** `false`。
  * `retryDelay` {integer} 重试之间等待的时间量（毫秒）。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `100`。
* `callback` {Function}
  * `err` {Error}

异步地移除文件和目录（基于标准 POSIX `rm` 实用程序建模）。

回调在完成时调用，可能带有异常。

### `fs.stat(path[, options], callback)`

<!-- YAML
added: v0.0.2
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.5.0
    pr-url: https://github.com/nodejs/node/pull/20220
    description: Accepts an additional `options` object to specify whether
                 the numeric values returned should be bigint.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.Stats} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* `callback` {Function}
  * `err` {Error}
  * `stats` {fs.Stats}

异步的 stat(2)。回调获得两个参数 `(err, stats)`，其中 `stats` 是 {fs.Stats} 对象。

如果发生错误，`err.code` 将是 [常见系统错误][] 之一。

不建议在调用 `fs.open()`、`fs.readFile()` 或 `fs.writeFile()` 之前使用 `fs.stat()` 检查文件的存在性。相反，用户代码应直接打开/读取/写入文件，并处理如果文件不存在时引发的错误。

要检查文件是否存在而不对其进行操作，建议使用 [`fs.access()`][]。

有关更多细节，请参阅 stat(2)。

### `fs.statfs(path[, options], callback)`

<!-- YAML
added:
  - v19.6.0
  - v18.15.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.StatFs} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* `callback` {Function}
  * `err` {Error}
  * `stats` {fs.StatFs}

异步的 statfs(2)。返回有关包含 `path` 的已挂载文件系统的信息。回调获得两个参数 `(err, stats)`，其中 `stats` 是 {fs.StatFs} 对象。

如果发生错误，`err.code` 将是 [常见系统错误][] 之一。

### `fs.symlink(target, path[, type], callback)`

<!-- YAML
added: v0.1.31
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/42894
    description: If the `type` argument is `null` or omitted, Node.js will
                 autodetect `target` type and automatically
                 select `dir` or `file`.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/23724
    description: If the `type` argument is not a string, Node.js autodetects
                 `target` type and automatically selects `dir` or `file`.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `target` and `path` parameters can be WHATWG `URL` objects
                 using `file:` protocol. Support is currently still
                 *experimental*.
-->

* `target` {string|Buffer|URL}
* `path` {string|Buffer|URL}
* `type` {string|null} **默认值:** `null`
* `callback` {Function}
  * `err` {Error}

异步的 symlink(2)，创建名为 `path` 指向 `target` 的链接。除了可能的异常外，不会向完成回调提供任何参数。

`type` 参数仅在 Windows 平台上使用，可以是 `'dir'`、`'file'` 或 `'junction'` 之一。如果 `type` 参数为 `null`，Node.js 将自动检测 `target` 类型并使用 `'file'` 或 `'dir'`。如果 `target` 不存在，将使用 `'file'`。Windows 连接点要求目标路径是绝对的。当使用 `'junction'` 时，`target` 参数将自动规范化为绝对路径。NTFS 卷上的连接点只能指向目录。

相对目标是相对于链接的父目录。

```mjs
import { symlink } from 'node:fs';

symlink('./mew', './mewtwo', callback);
```

上面的示例创建了一个符号链接 `mewtwo`，指向同一目录中的 `mew`：

```bash
$ tree .
.
├── mew
└── mewtwo -> ./mew
```

### `fs.truncate(path[, len], callback)`

<!-- YAML
added: v0.8.6
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `path` {string|Buffer|URL}
* `len` {integer} **默认值:** `0`
* `callback` {Function}
  * `err` {Error}

异步的 truncate(2)。除了可能的异常外，不会向完成回调提供任何参数。文件描述符也可以作为第一个参数传递。在这种情况下，`fs.ftruncate()` 被调用。

传递文件描述符已弃用，将来可能导致抛出错误。

### `fs.unlink(path, callback)`

<!-- YAML
added: v0.0.2
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
-->

* `path` {string|Buffer|URL}
* `callback` {Function}
  * `err` {Error}

异步的 unlink(2)。除了可能的异常外，不会向完成回调提供任何参数。

### `fs.unwatchFile(filename[, listener])`

<!-- YAML
added: v0.1.31
-->

* `filename` {string|Buffer|URL}
* `listener` {Function} 可选，先前使用 `fs.watchFile()` 附加的监听器。

停止监视 `filename` 的更改。如果指定了 `listener`，则仅移除该特定监听器。否则，*所有* 监听器将被移除，从而有效地停止监视 `filename`。

使用未被监视的 `filename` 调用 `fs.unwatchFile()` 是无操作，而不是错误。

使用 [`fs.watch()`][] 比 `fs.watchFile()` 和 `fs.unwatchFile()` 更高效。应尽可能使用 `fs.watch()` 而不是 `fs.watchFile()` 和 `fs.unwatchFile()`。

### `fs.utimes(path, atime, mtime, callback)`

<!-- YAML
added: v0.4.2
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v4.1.0
    pr-url: https://github.com/nodejs/node/pull/2387
    description: Numeric strings, `NaN`, and `Infinity` are now allowed
                 time specifiers.
-->

* `path` {string|Buffer|URL}
* `atime` {number|string|Date}
* `mtime` {number|string|Date}
* `callback` {Function}
  * `err` {Error}

更改 `path` 引用的对象的文件系统时间戳。

`atime` 和 `mtime` 参数遵循以下规则：

* 值可以是代表 Unix 纪元时间的数字、`Date` 或数字字符串，如 `'123456789.0'`。
* 如果值无法转换为数字，或者是 `NaN`、`Infinity` 或 `-Infinity`，将抛出 `Error`。

### `fs.watch(filename[, options][, listener])`

<!-- YAML
added: v0.5.10
changes:
  - version: v22.0.0
    pr-url: https://github.com/nodejs/node/pull/46284
    description: The `recursive` option is now supported on macOS.
  - version: v19.4.0
    pr-url: https://github.com/nodejs/node/pull/46089
    description: The `recursive` option is now supported on Windows.
  - version: v18.13.0
    pr-url: https://github.com/nodejs/node/pull/44912
    description: The `recursive` option is now supported on IBMi.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `listener` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version:
     - v14.3.0
     - v12.20.0
    pr-url: https://github.com/nodejs/node/pull/33170
    description: The `recursive` option is now supported on Linux.
  - version:
    - v14.14.0
    - v12.20.0
    pr-url: https://github.com/nodejs/node/pull/35611
    description: The `recursive` option is now supported on AIX.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `filename` parameter can be a WHATWG `URL` object using
                 `file:` protocol.
-->

* `filename` {string|Buffer|URL}
* `options` {string|Object}
  * `persistent` {boolean} 指示只要正在监视文件，进程是否应继续运行。**默认值:** `true`。
  * `recursive` {boolean} 指示是否应监视所有子目录，或仅当前目录。这在指定目录时适用，并且仅在支持的平台上（参见 [注意事项][]）。**默认值:** `false`。
  * `encoding` {string} 指定传递给监听器的文件名使用的字符编码。**默认值:** `'utf8'`。
  * `signal` {AbortSignal} 允许使用 AbortSignal 关闭监视器。
* `listener` {Function|undefined} **默认值:** `undefined`
  * `eventType` {string}
  * `filename` {string|Buffer}
* 返回: {fs.FSWatcher}

监视 `filename` 的更改，其中 `filename` 是文件或目录。

第二个参数是可选的。如果 `options` 作为字符串提供，则它指定 `encoding`。否则，`options` 应作为对象传递。

监听器回调获得两个参数 `(eventType, filename)`。`eventType` 是 `'rename'` 或 `'change'`，`filename` 是触发事件的文件的名称。

在大多数平台上，每当文件名在目录中出现或消失时，都会发出 `'rename'`。

监听器回调附加到由 `fs.FSWatcher` 发出的 `'change'` 事件，但它与 `eventType` 的 `'change'` 值不同。

如果传递了 `signal`，则中止相应的 AbortController 将关闭返回的 {fs.FSWatcher}。

#### 注意事项

`fs.watch` API 在不同平台上并非 100% 一致，并且在某些情况下不可用。

递归选项仅在 macOS、Windows、Linux、AIX 和 IBMi 上受支持。当在不支持该选项的平台上使用该选项时，将抛出异常。

在 Windows 上，如果监视的目录被移动或重命名，则不会触发任何事件。当监视的目录被删除时，会报告 `EPERM` 错误。

##### 可用性

此功能依赖于底层操作系统提供文件更改通知的方式。

* 在 Linux 系统上，这使用 [`inotify(7)`]。
* 在 BSD 系统上，这使用 [`kqueue(2)`]。
* 在 macOS 上，这使用 [`kqueue(2)`] 用于文件，但在目录上使用 [`FSEvents`]。
* 在 SunOS 系统（包括 Solaris 和 SmartOS）上，这使用 [`event ports`]。
* 在 Windows 系统上，此功能依赖于 [`ReadDirectoryChangesW`]。
* 在 IBM i 系统上，此功能不受支持。

如果底层功能由于某种原因不可用，则 `fs.watch()` 将无法工作并抛出异常。例如，在使用虚拟化软件（如 Vagrant 或 Docker）时，在网络文件系统（NFS、SMB 等）或主机文件系统上监视文件或目录可能不可靠，有时甚至不可能。

仍然可以使用 `fs.watchFile()`，它使用 stat 轮询，但这种方法较慢且可靠性较低。

##### 索引节点

在 Linux 和 macOS 系统上，`fs.watch()` 解析路径到 [索引节点][] 并监视该索引节点。如果监视的路径被删除并重新创建，则分配一个新的索引节点。监视将发出删除事件，但继续监视*原始*索引节点。不会发出新索引节点的事件。这是预期行为。

AIX 文件在文件的生命周期内保留相同的索引节点。在 AIX 上保存和关闭监视的文件将触发两个通知（一个用于添加新内容，另一个用于截断）。

##### 文件名参数

仅在 Linux、macOS、Windows 和 AIX 上支持在回调中提供 `filename` 参数。即使在支持的平台上，也不能保证 `filename` 总是被提供。因此，不要假设 `filename` 参数总是在回调中提供，如果它为 `null`，则有一些回退逻辑。

```mjs
import { watch } from 'node:fs';
watch('somedir', (eventType, filename) => {
  console.log(`event type is: ${eventType}`);
  if (filename) {
    console.log(`filename provided: ${filename}`);
  } else {
    console.log('filename not provided');
  }
});
```

### `fs.watchFile(filename[, options], listener)`

<!-- YAML
added: v0.1.31
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `listener` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `filename` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} **默认值:** `false`
  * `persistent` {boolean} **默认值:** `true`
  * `interval` {integer} **默认值:** `5007`
* `listener` {Function}
  * `current` {fs.Stats}
  * `previous` {fs.Stats}

监视 `filename` 的更改。每次访问文件时都会调用 `listener` 回调。

`options` 参数可以省略。如果提供，它应该是一个对象。`options` 对象可以包含一个名为 `persistent` 的布尔值，指示当文件被监视时进程是否应该继续运行。`options` 对象可以指定一个 `interval` 属性，指示目标应以多高的频率轮询（以毫秒为单位）。

`listener` 获得两个参数，当前 stat 对象和上一个 stat 对象：

```mjs
import { watchFile } from 'node:fs';

watchFile('message.text', (curr, prev) => {
  console.log(`the current mtime is: ${curr.mtime}`);
  console.log(`the previous mtime was: ${prev.mtime}`);
});
```

这些 stat 对象是 `fs.Stat` 的实例。如果 `bigint` 选项为 `true`，则这些对象中的数值值指定为 `BigInt`。

要在文件被修改而不仅仅是访问时得到通知，需要比较 `curr.mtimeMs` 和 `prev.mtimeMs`。

当 `fs.watchFile` 操作导致 `ENOENT` 错误时，它将调用监听器一次，所有字段都为零（或者，对于日期，是 Unix 纪元）。如果文件是之后创建的，监听器将再次被调用，并显示最新的 stat 对象。这是自 v0.10 以来的功能变化。

使用 [`fs.watch()`][] 比 `fs.watchFile` 和 `fs.unwatchFile` 更高效。应尽可能使用 `fs.watch` 而不是 `fs.watchFile` 和 `fs.unwatchFile`。

当 `fs.watchFile()` 正在监视的文件消失并重新出现时，第二个回调事件（文件重新出现）中 `previous` 的内容将与第一个回调事件（文件消失）中 `previous` 的内容相同。

这种情况发生在：

* 文件被删除，然后恢复
* 文件被重命名，然后再次重命名为其原始名称

### `fs.write(fd, buffer, offset[, length[, position]], callback)`

<!-- YAML
added: v0.0.2
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `buffer` parameter won't coerce unsupported input to
                 buffers anymore.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.4.0
    pr-url: https://github.com/nodejs/node/pull/10382
    description: The `position` parameter is optional now.
  - version: v7.2.0
    pr-url: https://github.com/nodejs/node/pull/7856
    description: The `offset` and `length` parameters are optional now.
-->

* `fd` {integer}
* `buffer` {Buffer|TypedArray|DataView}
* `offset` {integer}
* `length` {integer}
* `position` {integer}
* `callback` {Function}
  * `err` {Error}
  * `bytesWritten` {integer}
  * `buffer` {Buffer|TypedArray|DataView}

将 `buffer` 写入 `fd` 指定的文件。

`offset` 确定要写入的缓冲区部分，`length` 是一个整数，指定要写入的字节数。

`position` 指从文件开头数据应被写入的偏移量。如果 `typeof position !== 'number'`，数据将被写入当前位置。参见 pwrite(2)。

回调将获得三个参数 `(err, bytesWritten, buffer)`，其中 `bytesWritten` 指定从 `buffer` 写入了多少字节。

如果此方法作为其 [`util.promisify()`][] 版本调用，则返回一个具有 `bytesWritten` 和 `buffer` 属性的 `Object` 的 promise。

在同一文件上多次使用 `fs.write()` 而不等待回调是不安全的。对于这种情况，建议使用 [`fs.createWriteStream()`][]。

在 Linux 上，当文件以追加模式打开时，位置写入不起作用。内核会忽略位置参数，始终将数据追加到文件末尾。

### `fs.write(fd, buffer[, options], callback)`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

* `fd` {integer}
* `buffer` {Buffer|TypedArray|DataView}
* `options` {Object}
  * `offset` {integer} **默认值:** `0`
  * `length` {integer} **默认值:** `buffer.byteLength - offset`
  * `position` {integer} **默认值:** `null`
* `callback` {Function}
  * `err` {Error}
  * `bytesWritten` {integer}
  * `buffer` {Buffer|TypedArray|DataView}

类似于上面的 `fs.write` 函数，此版本接受一个可选的 `options` 对象。如果未指定 `options` 对象，它将使用上述值默认。

### `fs.write(fd, string[, position[, encoding]], callback)`

<!-- YAML
added: v0.11.5
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `string` parameter won't coerce unsupported input to
                 strings anymore.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.2.0
    pr-url: https://github.com/nodejs/node/pull/7856
    description: The `position` parameter is optional now.
-->

* `fd` {integer}
* `string` {string}
* `position` {integer}
* `encoding` {string} **默认值:** `'utf8'`
* `callback` {Function}
  * `err` {Error}
  * `written` {integer}
  * `string` {string}

将 `string` 写入 `fd` 指定的文件。如果 `string` 不是字符串，则抛出异常。

`position` 指从文件开头数据应被写入的偏移量。如果 `typeof position !== 'number'`，数据将被写入当前位置。参见 pwrite(2)。

`encoding` 是预期的字符串编码。

回调将接收参数 `(err, written, string)`，其中 `written` 指定传入的字符串需要写入多少字节。写入的字节数不一定与字符串字符数相同。参见 [`Buffer.byteLength`][]。

与 `fs.write()` 处理缓冲区的方式不同，整个字符串必须被写入。不能指定子字符串。这是因为字节偏移量可能与字符串偏移量不同。

在同一文件上多次使用 `fs.write()` 而不等待回调是不安全的。对于这种情况，建议使用 [`fs.createWriteStream()`][]。

在 Linux 上，当文件以追加模式打开时，位置写入不起作用。内核会忽略位置参数，始终将数据追加到文件末尾。

在 Windows 上，如果文件描述符连接到控制台（例如 `fd == 1` 或 `stdout`），则默认情况下无论使用何种编码，包含非 ASCII 字符的字符串都无法正确渲染。通过使用 `chcp 65001` 命令更改活动代码页来配置控制台以支持 UTF-8。有关更多细节，请参阅 [chcp][] 文档。

### `fs.writeFile(file, data[, options], callback)`

<!-- YAML
added: v0.1.29
changes:
  - version:
    - v21.0.0
    - v20.10.0
    pr-url: https://github.com/nodejs/node/pull/50009
    description: The `flush` option is now supported.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37490
    description: The `data` argument now accepts `AsyncIterable` and `Iterable`.
  - version:
    - v15.2.0
    - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/35993
    description: The options argument may include an AbortSignal to abort an
                 ongoing writeFile request.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `data` parameter won't coerce unsupported input to
                 strings anymore.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.4.0
    pr-url: https://github.com/nodejs/node/pull/10382
    description: The `data` parameter can now be a `Uint8Array`.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/3163
    description: The `file` parameter can be a file descriptor now.
-->

* `file` {string|Buffer|URL|integer} 文件名或文件描述符
* `data` {string|Buffer|TypedArray|DataView|AsyncIterable|Iterable|Stream}
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `mode` {integer} **默认值:** `0o666`
  * `flag` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'w'`。
  * `flush` {boolean} 如果所有数据成功写入文件，并且 `flush` 为 `true`，则使用 `fs.fsync()` 刷新数据。**默认值:** `false`。
  * `signal` {AbortSignal} 允许中止正在进行的 writeFile
* `callback` {Function}
  * `err` {Error}

异步地将数据写入文件，如果文件已存在则替换该文件。`data` 可以是字符串、缓冲区、{AsyncIterable} 或 {Iterable} 对象。

`encoding` 选项在 `data` 是字符串时适用。默认值为 `'utf8'`。

`mode` 选项仅影响新创建的文件。有关更多细节，请参见 [`fs.open()`][]。

如果 `data` 是缓冲区，则忽略 `encoding` 选项。

如果 `data` 是普通的对象，则它必须具有自身的（不是继承的）`toString` 函数属性。

```mjs
import { writeFile } from 'node:fs';
import { Buffer } from 'node:buffer';

const data = new Uint8Array(Buffer.from('Hello Node.js'));
writeFile('message.txt', data, (err) => {
  if (err) throw err;
  console.log('The file has been saved!');
});
```

如果 `options` 是字符串，则它指定编码：

```mjs
import { writeFile } from 'node:fs';

writeFile('message.txt', 'Hello Node.js', 'utf8', callback);
```

在同一个文件上多次使用 `fs.writeFile()` 而不等待回调是不安全的。对于这种情况，建议使用 [`fs.createWriteStream()`][]。

与 `fs.readFile` 类似 - `fs.writeFile` 是一个便捷方法，它在内部执行多个 `write` 调用来写入传递给它的缓冲区。对于性能敏感的代码，请考虑使用 [`fs.createWriteStream()`][]。

可以使用 {AbortSignal} 取消 `fs.writeFile()`。取消是“尽力而为”，某些数据可能仍然会被写入。

```mjs
import { writeFile } from 'node:fs';
import { Buffer } from 'node:buffer';

const controller = new AbortController();
const { signal } = controller;
const data = new Uint8Array(Buffer.from('Hello Node.js'));
writeFile('message.txt', data, { signal }, (err) => {
  // 当请求被中止时 - err 是 AbortError
});
// 当您想要中止请求时
controller.abort();
```

中止正在进行的请求不会中止单个操作系统请求，而是中止 `fs.writeFile` 执行的内部缓冲。

#### 使用 `fs.writeFile()` 与文件描述符

当 `file` 是文件描述符时，行为类似于直接调用 `fs.write()`（推荐）。请参见以下关于使用文件描述符的说明：

```mjs
import { write } from 'node:fs';
import { Buffer } from 'node:buffer';

write(fd, Buffer.from(data, options.encoding), callback);
```

与直接调用 `fs.write()` 不同，在某些异常情况下，`fs.writeFile()` 可能会多次尝试写入文件描述符，然后才返回错误。

如果要管理原始文件描述符，则关闭文件描述符取决于用户。

```mjs
import { open, close, writeFile } from 'node:fs';

open('data.txt', 'w', (err, fd) => {
  if (err) throw err;

  try {
    writeFile(fd, 'data to write', 'utf8', (err) => {
      close(fd, (err) => {
        if (err) throw err;
      });
      if (err) throw err;
    });
  } catch (err) {
    close(fd, (err) => {
      if (err) throw err;
    });
    throw err;
  }
});
```

### `fs.writev(fd, buffers[, position], callback)`

<!-- YAML
added:
  - v12.9.0
  - v10.17.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `fd` {integer}
* `buffers` {ArrayBufferView\[]}
* `position` {integer|null} 从文件开头开始写入 `buffers` 数据的偏移量。如果 `position` 不是 `number`，数据将写入当前位置。**默认值:** `null`
* `callback` {Function}
  * `err` {Error}
  * `bytesWritten` {integer}
  * `buffers` {ArrayBufferView\[]}

将 `buffers` 数组写入 `fd`。

`position` 是从文件开头开始写入数据的偏移量。如果 `typeof position !== 'number'`，数据将写入当前位置。

回调将获得三个参数：`err`、`bytesWritten` 和 `buffers`。`bytesWritten` 是从 `buffers` 写入的字节数。

如果此方法作为其 [`util.promisify()`][] 版本调用，则返回一个具有 `bytesWritten` 和 `buffers` 属性的 `Object` 的 promise。

在同一文件上多次使用 `fs.writev()` 而不等待回调是不安全的。对于这种情况，请使用 [`fs.createWriteStream()`][]。

在 Linux 上，当文件以追加模式打开时，位置写入不起作用。内核会忽略位置参数，始终将数据追加到文件末尾。

## 同步 API

同步 API 同步执行所有操作，阻塞事件循环，直到操作完成或失败。

### `fs.accessSync(path[, mode])`

<!-- YAML
added: v0.11.15
changes:
  - version: v20.8.0
    pr-url: https://github.com/nodejs/node/pull/49683
    description: The constants `fs.F_OK`, `fs.R_OK`, `fs.W_OK` and `fs.X_OK`
                 which were present directly on `fs` are deprecated.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/6534
    description: The constants like `fs.R_OK`, etc which were present directly
                 on `fs` were moved into `fs.constants` as a soft deprecation.
                 Thus for Node.js `< v6.3.0` use `fs`
                 to access those constants, or
                 do something like `(fs.constants || fs).R_OK` to work with all
                 versions.
-->

* `path` {string|Buffer|URL}
* `mode` {integer} **默认值:** `fs.constants.F_OK`

同步测试用户对 `path` 指定的文件或目录的权限。`mode` 参数是一个可选的整数，指定要执行的可访问性检查。`mode` 应该是值 `fs.constants.F_OK` 或由 `fs.constants.R_OK`、`fs.constants.W_OK` 和 `fs.constants.X_OK` 中任何值的按位或组成的掩码（例如 `fs.constants.W_OK | fs.constants.R_OK`）。有关 `mode` 的可能值，请检查 [文件访问常量][]。

如果任何可访问性检查失败，将抛出 {Error}。否则，该方法将返回 `undefined`。

```mjs
import { accessSync, constants } from 'node:fs';

try {
  accessSync('etc/passwd', constants.R_OK | constants.W_OK);
  console.log('can read/write');
} catch (err) {
  console.error('no access!');
}
```

### `fs.appendFileSync(path, data[, options])`

<!-- YAML
added: v0.6.7
changes:
  - version:
    - v21.1.0
    - v20.10.0
    pr-url: https://github.com/nodejs/node/pull/50095
    description: The `flush` option is now supported.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/12562
    description: The `callback` parameter is no longer optional. Not passing
                 it will throw a `TypeError` at runtime.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/3163
    description: The `file` parameter can be a file descriptor now.
-->

* `path` {string|Buffer|URL|number} 文件名或文件描述符
* `data` {string|Buffer}
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `mode` {integer} **默认值:** `0o666`
  * `flag` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'a'`。
  * `flush` {boolean} 如果为 `true`，则在关闭底层文件描述符之前会刷新它。**默认值:** `false`。

同步地将数据追加到文件，如果文件尚不存在则创建该文件。`data` 可以是字符串或 {Buffer}。

`mode` 选项仅影响新创建的文件。有关更多细节，请参见 [`fs.open()`][]。

```mjs
import { appendFileSync } from 'node:fs';

try {
  appendFileSync('message.txt', 'data to append');
  console.log('The "data to append" was appended to file!');
} catch (err) {
  /* 处理错误 */
}
```

如果 `options` 是字符串，则它指定编码：

```mjs
import { appendFileSync } from 'node:fs';

appendFileSync('message.txt', 'data to append', 'utf8');
```

`path` 可以指定为已打开用于追加的数字文件描述符（使用 `fs.open()` 或 `fs.openSync()`）。文件描述符不会自动关闭。

```mjs
import { openSync, closeSync, appendFileSync } from 'node:fs';

let fd;

try {
  fd = openSync('message.txt', 'a');
  appendFileSync(fd, 'data to append', 'utf8');
} catch (err) {
  /* 处理错误 */
} finally {
  if (fd !== undefined)
    closeSync(fd);
}
```

### `fs.chmodSync(path, mode)`

<!-- YAML
added: v0.6.7
changes:
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `mode` {string|integer}

有关详细信息，请参阅此 API 的异步版本文档：[`fs.chmod()`][]。

另请参阅：chmod(2)。

### `fs.chownSync(path, uid, gid)`

<!-- YAML
added: v0.1.97
changes:
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `uid` {integer}
* `gid` {integer}

同步更改文件的所有者和组。返回 `undefined`。这是 [`fs.chown()`][] 的同步版本。

另请参阅：chown(2)。

### `fs.closeSync(fd)`

<!-- YAML
added: v0.1.21
-->

* `fd` {integer}

关闭文件描述符。返回 `undefined`。

在任何当前通过任何其他 `fs` 操作使用的文件描述符（`fd`）上调用 `fs.closeSync()` 可能导致未定义的行为。

另请参阅：close(2)。

### `fs.copyFileSync(src, dest[, mode])`

<!-- YAML
added: v8.5.0
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/27044
    description: Changed `flags` argument to `mode` and imposed
                 stricter type validation.
-->

* `src` {string|Buffer|URL} 要复制的源文件名
* `dest` {string|Buffer|URL} 复制操作的目标文件名
* `mode` {integer} 复制操作的修饰符。**默认值:** `0`。

同步地将 `src` 复制到 `dest`。默认情况下，如果 `dest` 已存在，则会被覆盖。返回 `undefined`。Node.js 不保证复制操作的原子性。如果在目标文件已打开进行写入后发生错误，Node.js 将尝试删除目标文件。

`mode` 是一个可选整数，指定复制操作的行为。可以创建由两个或多个值的按位或组成的掩码（例如 `fs.constants.COPYFILE_EXCL | fs.constants.COPYFILE_FICLONE`）。

* `fs.constants.COPYFILE_EXCL`：如果 `dest` 已存在，复制操作将失败。
* `fs.constants.COPYFILE_FICLONE`：复制操作将尝试创建写时复制 reflink。如果平台不支持写时复制，则使用回退复制机制。
* `fs.constants.COPYFILE_FICLONE_FORCE`：复制操作将尝试创建写时复制 reflink。如果平台不支持写时复制，则操作将失败。

```mjs
import { copyFileSync, constants } from 'node:fs';

// 默认情况下，destination.txt 将被创建或覆盖。
copyFileSync('source.txt', 'destination.txt');
console.log('source.txt was copied to destination.txt');

// 通过使用 COPYFILE_EXCL，如果 destination.txt 存在，操作将失败。
copyFileSync('source.txt', 'destination.txt', constants.COPYFILE_EXCL);
```

### `fs.cpSync(src, dest[, options])`

<!-- YAML
added: v16.7.0
changes:
  - version: v22.3.0
    pr-url: https://github.com/nodejs/node/pull/53127
    description: This API is no longer experimental.
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/47084
    description: Accept an additional `mode` option to specify
                 the copy behavior as the `mode` argument of `fs.copyFile()`.
  - version:
    - v17.6.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/41819
    description: Accepts an additional `verbatimSymlinks` option to specify
                 whether to perform path resolution for symlinks.
-->

* `src` {string|URL} 要复制的源路径。
* `dest` {string|URL} 要复制到的目标路径。
* `options` {Object}
  * `dereference` {boolean} 取消引用符号链接。**默认值:** `false`。
  * `errorOnExist` {boolean} 当 `force` 为 `false` 且目标已存在时，抛出错误。**默认值:** `false`。
  * `filter` {Function} 过滤要复制的文件/目录的函数。返回 `true` 复制项目，`false` 忽略它。当忽略目录时，其所有内容也将被跳过。也可以返回一个解析为 `true` 或 `false` 的 `Promise` **默认值:** `undefined`。
    * `src` {string} 要复制的源路径。
    * `dest` {string} 要复制到的目标路径。
    * 返回: {boolean|Promise} 可强制转换为 `boolean` 的值或使用此类值履行的 `Promise`。
  * `force` {boolean} 覆盖现有文件或目录。如果将此设置为 false 且目标存在，复制操作将忽略错误。使用 `errorOnExist` 选项更改此行为。**默认值:** `true`。
  * `mode` {integer} 复制操作的修饰符。**默认值:** `0`。参见 [`fs.copyFile()`][] 的 `mode` 标志。
  * `preserveTimestamps` {boolean} 当为 `true` 时，将保留 `src` 的时间戳。**默认值:** `false`。
  * `recursive` {boolean} 递归复制目录 **默认值:** `false`
  * `verbatimSymlinks` {boolean} 当为 `true` 时，将跳过符号链接的路径解析。**默认值:** `false`

同步地将整个目录结构从 `src` 复制到 `dest`，包括子目录和文件。

当将一个目录复制到另一个目录时，不支持通配符，行为类似于 `cp dir1/ dir2/`。

### `fs.existsSync(path)`

<!-- YAML
added: v0.1.21
changes:
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using
                 `file:` protocol.
-->

* `path` {string|Buffer|URL}
* 返回: {boolean}

如果路径存在，则返回 `true`，否则返回 `false`。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.exists()`][]。

`fs.exists()` 已弃用，但 `fs.existsSync()` 不是。`fs.exists()` 的 `callback` 参数接受与其他 Node.js 回调不一致的参数。`fs.existsSync()` 不使用回调。

```mjs
import { existsSync } from 'node:fs';

if (existsSync('/etc/passwd'))
  console.log('The path exists.');
```

### `fs.fchmodSync(fd, mode)`

<!-- YAML
added: v0.4.7
-->

* `fd` {integer}
* `mode` {string|integer}

设置文件的权限。返回 `undefined`。这是 [`fs.fchmod()`][] 的同步版本。

另请参阅：chmod(2)。

### `fs.fchownSync(fd, uid, gid)`

<!-- YAML
added: v0.4.7
-->

* `fd` {integer}
* `uid` {integer}
* `gid` {integer}

设置文件的所有者。返回 `undefined`。这是 [`fs.fchown()`][] 的同步版本。

另请参阅：chown(2)。

### `fs.fdatasyncSync(fd)`

<!-- YAML
added: v0.1.96
-->

* `fd` {integer}

强制所有当前与文件关联的排队 I/O 操作到操作系统的同步 I/O 完成状态。返回 `undefined`。有关详细信息，请参阅 POSIX fdatasync(2) 文档。

### `fs.fstatSync(fd[, options])`

<!-- YAML
added: v0.1.95
changes:
  - version: v10.5.0
    pr-url: https://github.com/nodejs/node/pull/20220
    description: Accepts an additional `options` object to specify whether
                 the numeric values returned should be bigint.
-->

* `fd` {integer}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.Stats} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* 返回: {fs.Stats}

检索文件描述符的 {fs.Stats}。

有关详细信息，请参阅 POSIX fstat(2) 文档。

### `fs.fsyncSync(fd)`

<!-- YAML
added: v0.1.96
-->

* `fd` {integer}

请求将打开文件描述符的所有数据刷新到存储设备。返回 `undefined`。有关详细信息，请参阅 POSIX fsync(2) 文档。

### `fs.ftruncateSync(fd[, len])`

<!-- YAML
added: v0.8.6
-->

* `fd` {integer}
* `len` {integer} **默认值:** `0`

截断文件描述符。返回 `undefined`。有关详细信息，请参阅 POSIX ftruncate(2) 文档。

### `fs.futimesSync(fd, atime, mtime)`

<!-- YAML
added: v0.4.2
changes:
  - version: v4.1.0
    pr-url: https://github.com/nodejs/node/pull/2387
    description: Numeric strings, `NaN`, and `Infinity` are now allowed
                 time specifiers.
-->

* `fd` {integer}
* `atime` {number|string|Date}
* `mtime` {number|string|Date}

同步的 [`fs.futimes()`][]。返回 `undefined`。

### `fs.globSync(pattern[, options])`

<!-- YAML
added: v22.0.0
changes:
  - version: v24.1.0
    pr-url: https://github.com/nodejs/node/pull/58182
    description: Add support for `URL` instances for `cwd` option.
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57513
    description: Marking the API stable.
  - version:
    - v23.7.0
    - v22.14.0
    pr-url: https://github.com/nodejs/node/pull/56489
    description: Add support for `exclude` option to accept glob patterns.
  - version: v22.2.0
    pr-url: https://github.com/nodejs/node/pull/52837
    description: Add support for `withFileTypes` as an option.
-->

* `pattern` {string|string\[]}
* `options` {Object}
  * `cwd` {string|URL} 当前工作目录。**默认值:** `process.cwd()`
  * `exclude` {Function|string\[]} 过滤掉文件/目录的函数或要排除的全局模式列表。如果提供了函数，返回 `true` 排除项目，`false` 包含它。**默认值:** `undefined`。
  * `withFileTypes` {boolean} 如果为 `true`，全局应返回路径作为 Dirent，否则为 `false`。**默认值:** `false`。
* 返回: {string\[]|Buffer\[]|fs.Dirent\[]}

同步的 [`fs.glob()`][]。

### `fs.lchmodSync(path, mode)`

<!-- YAML
deprecated: v0.4.7
-->

> Stability: 0 - Deprecated

* `path` {string|Buffer|URL}
* `mode` {integer}

更改符号链接的权限。返回 `undefined`。这是 [`fs.lchmod()`][] 的同步版本。

### `fs.lchownSync(path, uid, gid)`

<!-- YAML
added: v0.4.7
changes:
  - version: v10.6.0
    pr-url: https://github.com/nodejs/node/pull/21498
    description: This API is no longer deprecated.
-->

* `path` {string|Buffer|URL}
* `uid` {integer}
* `gid` {integer}

设置符号链接的所有者。返回 `undefined`。这是 [`fs.lchown()`][] 的同步版本。

### `fs.lutimesSync(path, atime, mtime)`

<!-- YAML
added:
  - v14.5.0
  - v12.19.0
-->

* `path` {string|Buffer|URL}
* `atime` {number|string|Date}
* `mtime` {number|string|Date}

以与 [`fs.utimes()`][] 相同的方式更改文件的访问和修改时间，不同之处在于如果路径引用符号链接，则不会取消引用该链接：而是更改符号链接本身的时间戳。返回 `undefined`。

### `fs.linkSync(existingPath, newPath)`

<!-- YAML
added: v0.1.31
changes:
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `existingPath` and `newPath` parameters can be WHATWG
                 `URL` objects using `file:` protocol. Support is currently
                 still *experimental*.
-->

* `existingPath` {string|Buffer|URL}
* `newPath` {string|Buffer|URL}

从 `existingPath` 创建到 `newPath` 的新链接。有关更多细节，请参阅 POSIX link(2) 文档。返回 `undefined`。

### `fs.lstatSync(path[, options])`

<!-- YAML
added: v0.1.30
changes:
  - version: v10.5.0
    pr-url: https://github.com/nodejs/node/pull/20220
    description: Accepts an additional `options` object to specify whether
                 the numeric values returned should be bigint.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.Stats} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* 返回: {fs.Stats}

检索 `path` 引用的符号链接的 {fs.Stats}。

有关更多细节，请参阅 POSIX lstat(2) 文档。

### `fs.mkdirSync(path[, options])`

<!-- YAML
added: v0.1.21
changes:
  - version:
     - v13.11.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/31530
    description: In `recursive` mode, the first created path is returned now.
  - version: v10.12.0
    pr-url: https://github.com/nodejs/node/pull/21875
    description: The second argument can now be an `options` object with
                 `recursive` and `mode` properties.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `options` {Object|integer}
  * `recursive` {boolean} **默认值:** `false`
  * `mode` {string|integer} 在 Windows 上不支持。**默认值:** `0o777`。
* 返回: {string|undefined}

同步创建目录。返回 `undefined`，或者如果 `recursive` 为 `true`，则返回第一个创建的目录路径。这是 [`fs.mkdir()`][] 的同步版本。

有关更多细节，请参阅 POSIX mkdir(2) 文档。

### `fs.mkdtempSync(prefix[, options])`

<!-- YAML
added: v5.10.0
changes:
  - version:
    - v20.6.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/48828
    description: The `prefix` parameter now accepts buffers and URL.
  - version:
      - v16.5.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/39028
    description: The `prefix` parameter now accepts an empty string.
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/7897
    description: The `callback` parameter is no longer optional. Not passing
                 it will emit a deprecation warning with id DEP0013.
  - version: v6.2.1
    pr-url: https://github.com/nodejs/node/pull/6828
    description: The `callback` parameter is optional now.
-->

* `prefix` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* 返回: {string}

返回创建的目录路径。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.mkdtemp()`][]。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定要使用的字符编码。

### `fs.mkdtempDisposableSync(prefix[, options])`

<!-- YAML
added: v24.4.0
-->

* `prefix` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* 返回: {Disposable}

返回一个同步可处置对象，其 `path` 属性持有创建的目录路径。当对象被处置时，如果目录仍然存在，它将同步移除目录及其内容。如果目录无法删除，处置将抛出错误。对象有一个同步 `remove()` 方法，将执行相同的任务。

此函数和结果对象上的处置函数都是同步的，因此应与 `using` 一起使用，如 `using dir = fs.mkdtempDisposableSync('prefix')`。

有关详细信息，请参阅 [`fs.mkdtempSync()`][] 的文档。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定要使用的字符编码。

### `fs.opendirSync(path[, options])`

<!-- YAML
added: v12.12.0
changes:
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/41439
    description: Added `recursive` option.
  - version:
     - v13.1.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30114
    description: The `bufferSize` option was introduced.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `bufferSize` {number} 从目录读取时内部缓冲的目录条目数。较高的值导致更好的性能但更高的内存使用。**默认值:** `32`
  * `recursive` {boolean} 解析的 {fs.Dir} 将是一个包含所有子文件和目录的 {Iterable}。**默认值:** `false`
* 返回: {fs.Dir}

同步打开目录。参见 opendir(3)。

创建一个 {fs.Dir}，其中包含所有用于从目录读取和清理的进一步函数。

`encoding` 选项在打开目录及后续读取操作时设置 `path` 的编码。

### `fs.openSync(path[, flags[, mode]])`

<!-- YAML
added: v0.1.21
changes:
  - version: v11.1.0
    pr-url: https://github.com/nodejs/node/pull/23767
    description: The `flags` argument is now optional and defaults to `'r'`.
  - version: v9.9.0
    pr-url: https://github.com/nodejs/node/pull/18801
    description: The `as` and `as+` flags are supported now.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `flags` {string|number} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'r'`。
* `mode` {string|integer} **默认值:** `0o666`（可读和可写）
* 返回: {integer}

返回表示文件描述符的整数。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.open()`][]。

### `fs.readdirSync(path[, options])`

<!-- YAML
added: v0.1.21
changes:
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/41439
    description: Added `recursive` option.
  - version: v10.10.0
    pr-url: https://github.com/nodejs/node/pull/22020
    description: New option `withFileTypes` was added.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
  * `withFileTypes` {boolean} **默认值:** `false`
  * `recursive` {boolean} 如果为 `true`，则递归读取目录的内容。在递归模式下，它将列出所有文件、子文件和目录。**默认值:** `false`。
* 返回: {string\[]|Buffer\[]|fs.Dirent\[]}

读取目录的内容。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.readdir()`][]。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定用于文件名的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的文件名将作为 {Buffer} 对象传递。

如果 `options.withFileTypes` 设置为 `true`，结果将包含 {fs.Dirent} 对象。

### `fs.readFileSync(path[, options])`

<!-- YAML
added: v0.1.8
changes:
  - version:
    - v7.6.0
    - v6.5.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/3163
    description: The `path` parameter can be a file descriptor now.
-->

* `path` {string|Buffer|URL|integer} 文件名或文件描述符
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `null`
  * `flag` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'r'`。
* 返回: {string|Buffer}

返回 `path` 的内容。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.readFile()`][]。

如果指定了 `encoding` 选项，则此函数返回字符串。否则返回缓冲区。

与 [`fs.readFile()`][] 类似，当路径是目录时，`fs.readFileSync()` 的行为是特定于平台的。

```mjs
import { readFileSync } from 'node:fs';

// 在 macOS、Linux 和 Windows 上：
readFileSync('<directory>');
// => [Error: EISDIR: illegal operation on a directory, read <directory>]

// 在 FreeBSD 和 NetBSD 上：
readFileSync('<directory>'); // => <data>
```

### `fs.readlinkSync(path[, options])`

<!-- YAML
added: v0.1.31
changes:
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* 返回: {string|Buffer}

返回符号链接的字符串值。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.readlink()`][]。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定返回的链接路径的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的链接路径将作为 {Buffer} 对象传递。

### `fs.readSync(fd, buffer, offset, length[, position])`

<!-- YAML
added: v0.1.21
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/42835
    description: Accepts bigint values as `position`.
  - version: v10.10.0
    pr-url: https://github.com/nodejs/node/pull/22020
    description: The `length` parameter is optional now.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/4518
    description: The `length` parameter can now be `0`.
-->

* `fd` {integer}
* `buffer` {Buffer|TypedArray|DataView} 将用读取的文件数据填充的缓冲区。
* `offset` {integer} `buffer` 中开始填充的位置。
* `length` {integer} 要读取的字节数。
* `position` {integer|bigint|null} 从文件中开始读取数据的位置。如果 `null` 或 `-1`，将从当前文件位置读取数据，并且位置将被更新。如果 `position` 是非负整数，则当前文件位置将保持不变。
* 返回: {integer}

返回 `bytesRead` 的数量。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.read()`][]。

### `fs.readSync(fd, buffer, [options])`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/42835
    description: Accepts bigint values as `position`.
-->

* `fd` {integer}
* `buffer` {Buffer|TypedArray|DataView} 将用读取的文件数据填充的缓冲区。
* `options` {Object}
  * `offset` {integer} **默认值:** `0`
  * `length` {integer} **默认值:** `buffer.byteLength - offset`
  * `position` {integer|bigint|null} **默认值:** `null`
* 返回: {integer}

返回 `bytesRead` 的数量。

类似于上面的 `fs.readSync` 函数，此版本接受一个可选的 `options` 对象。如果未指定 `options` 对象，它将使用上述值默认。

### `fs.readvSync(fd, buffers[, position])`

<!-- YAML
added:
  - v13.13.0
  - v12.17.0
-->

* `fd` {integer}
* `buffers` {ArrayBufferView\[]}
* `position` {integer|null} 从文件开头开始读取数据的偏移量。如果 `position` 不是 `number`，数据将从当前位置读取。**默认值:** `null`
* 返回: {integer} 读取的字节数。

从 `fd` 读取并使用 `readv()` 写入 `buffers`。

`position` 是从文件开头开始读取数据的偏移量。如果 `typeof position !== 'number'`，数据将从当前位置读取。

### `fs.realpathSync(path[, options])`

<!-- YAML
added: v0.1.31
changes:
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* 返回: {string|Buffer}

返回解析的路径名。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.realpath()`][]。

### `fs.realpathSync.native(path[, options])`

<!-- YAML
added: v9.2.0
-->

* `path` {string|Buffer|URL}
* `options` {string|Object}
  * `encoding` {string} **默认值:** `'utf8'`
* 返回: {string|Buffer}

同步的 realpath(3)。

仅支持可以转换为 UTF8 字符串的路径。

可选的 `options` 参数可以是指定编码的字符串，或者是具有 `encoding` 属性的对象，指定用于返回的路径的字符编码。如果 `encoding` 设置为 `'buffer'`，返回的路径将作为 {Buffer} 对象传递。

在 Linux 上，当 Node.js 链接到 musl libc 时，procfs 文件系统必须挂载在 `/proc` 上才能使此函数工作。Glibc 没有此限制。

### `fs.renameSync(oldPath, newPath)`

<!-- YAML
added: v0.1.21
changes:
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `oldPath` and `newPath` parameters can be WHATWG `URL`
                 objects using `file:` protocol.
-->

* `oldPath` {string|Buffer|URL}
* `newPath` {string|Buffer|URL}

将 `oldPath` 重命名为 `newPath`。返回 `undefined`。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.rename()`][]。

### `fs.rmdirSync(path[, options])`

<!-- YAML
added: v0.1.21
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37216
    description: "Using `fs.rmdirSync(path, { recursive: true })` on a `path`
                 that is a file is no longer permitted and results in an
                 `ENOENT` error on Windows and an `ENOTDIR` error on POSIX."
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37216
    description: "Using `fs.rmdirSync(path, { recursive: true })` on a `path`
                 that does not exist is no longer permitted and results in a
                 `ENOENT` error."
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37302
    description: The `recursive` option is deprecated, using it triggers a
                 deprecation warning.
  - version: v14.14.0
    pr-url: https://github.com/nodejs/node/pull/35579
    description: The `recursive` option is deprecated, use `fs.rmSync` instead.
  - version:
     - v13.3.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30644
    description: The `maxBusyTries` option is renamed to `maxRetries`, and its
                 default is 0. The `emfileWait` option has been removed, and
                 `EMFILE` errors use the same retry logic as other errors. The
                 `retryDelay` option is now supported. `ENFILE` errors are now
                 retried.
  - version: v12.10.0
    pr-url: https://github.com/nodejs/node/pull/29168
    description: The `recursive`, `maxBusyTries`, and `emfileWait` options are
                  now supported.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `maxRetries` {integer} 如果遇到 `EBUSY`、`EMFILE`、`ENFILE`、`ENOTEMPTY` 或 `EPERM` 错误，Node.js 将以每次重试线性退避等待 `retryDelay` 毫秒的方式重试操作。此选项表示重试次数。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `0`。
  * `recursive` {boolean} 如果为 `true`，则执行递归目录移除。在递归模式下，操作会在失败时重试。**默认值:** `false`。**已弃用。**
  * `retryDelay` {integer} 重试之间等待的时间量（毫秒）。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `100`。

同步的 rmdir(2)。返回 `undefined`。

在文件（非目录）上使用 `fs.rmdirSync()` 会导致在 Windows 上抛出 `ENOENT` 错误，在 POSIX 上抛出 `ENOTDIR` 错误。要获得类似于 `rm -rf` Unix 命令的行为，请使用 [`fs.rmSync()`][] 并设置选项 `{ recursive: true, force: true }`。

### `fs.rmSync(path[, options])`

<!-- YAML
added: v14.14.0
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `force` {boolean} 当为 `true` 时，如果 `path` 不存在，异常将被忽略。**默认值:** `false`。
  * `maxRetries` {integer} 如果遇到 `EBUSY`、`EMFILE`、`ENFILE`、`ENOTEMPTY` 或 `EPERM` 错误，Node.js 将以每次重试线性退避等待 `retryDelay` 毫秒的方式重试操作。此选项表示重试次数。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `0`。
  * `recursive` {boolean} 如果为 `true`，则执行递归目录移除。在递归模式下，操作会在失败时重试。**默认值:** `false`。
  * `retryDelay` {integer} 重试之间等待的时间量（毫秒）。如果 `recursive` 选项不为 `true`，则忽略此选项。**默认值:** `100`。

同步地移除文件和目录（基于标准 POSIX `rm` 实用程序建模）。返回 `undefined`。

### `fs.statSync(path[, options])`

<!-- YAML
added: v0.1.21
changes:
  - version: v10.5.0
    pr-url: https://github.com/nodejs/node/pull/20220
    description: Accepts an additional `options` object to specify whether
                 the numeric values returned should be bigint.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.Stats} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* 返回: {fs.Stats}

检索 `path` 的 {fs.Stats}。

### `fs.statfsSync(path[, options])`

<!-- YAML
added:
  - v19.6.0
  - v18.15.0
-->

* `path` {string|Buffer|URL}
* `options` {Object}
  * `bigint` {boolean} 返回的 {fs.StatFs} 对象中的数值是否应为 `bigint`。**默认值:** `false`。
* 返回: {fs.StatFs}

同步的 statfs(2)。返回包含 `path` 的已挂载文件系统的信息的 {fs.StatFs}。

如果发生错误，`err.code` 将是 [常见系统错误][] 之一。

### `fs.symlinkSync(target, path[, type])`

<!-- YAML
added: v0.1.31
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/42894
    description: If the `type` argument is `null` or omitted, Node.js will
                 autodetect `target` type and automatically
                 select `dir` or `file`.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/23724
    description: If the `type` argument is not a string, Node.js autodetects
                 `target` type and automatically selects `dir` or `file`.
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `target` and `path` parameters can be WHATWG `URL` objects
                 using `file:` protocol. Support is currently still
                 *experimental*.
-->

* `target` {string|Buffer|URL}
* `path` {string|Buffer|URL}
* `type` {string|null} **默认值:** `null`

返回 `undefined`。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.symlink()`][]。

### `fs.truncateSync(path[, len])`

<!-- YAML
added: v0.8.6
-->

* `path` {string|Buffer|URL}
* `len` {integer} **默认值:** `0`

截断文件。返回 `undefined`。文件描述符也可以作为第一个参数传递。在这种情况下，`fs.ftruncateSync()` 被调用。

传递文件描述符已弃用，将来可能导致抛出错误。

### `fs.unlinkSync(path)`

<!-- YAML
added: v0.1.21
changes:
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
-->

* `path` {string|Buffer|URL}

同步的 unlink(2)。返回 `undefined`。

### `fs.utimesSync(path, atime, mtime)`

<!-- YAML
added: v0.4.2
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/11919
    description: "`NaN`, `Infinity`, and `-Infinity` are no longer valid time
                 specifiers."
  - version: v7.6.0
    pr-url: https://github.com/nodejs/node/pull/10739
    description: The `path` parameter can be a WHATWG `URL` object using `file:`
                 protocol.
  - version: v4.1.0
    pr-url: https://github.com/nodejs/node/pull/2387
    description: Numeric strings, `NaN`, and `Infinity` are now allowed
                 time specifiers.
-->

* `path` {string|Buffer|URL}
* `atime` {number|string|Date}
* `mtime` {number|string|Date}

返回 `undefined`。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.utimes()`][]。

### `fs.writeFileSync(file, data[, options])`

<!-- YAML
added: v0.1.29
changes:
  - version:
    - v21.0.0
    - v20.10.0
    pr-url: https://github.com/nodejs/node/pull/50009
    description: The `flush` option is now supported.
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37490
    description: The `data` argument now accepts `AsyncIterable` and `Iterable`.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `data` parameter won't coerce unsupported input to
                 strings anymore.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/3163
    description: The `file` parameter can be a file descriptor now.
-->

* `file` {string|Buffer|URL|integer} 文件名或文件描述符
* `data` {string|Buffer|TypedArray|DataView|AsyncIterable|Iterable}
* `options` {Object|string}
  * `encoding` {string|null} **默认值:** `'utf8'`
  * `mode` {integer} **默认值:** `0o666`
  * `flag` {string} 参见 [文件系统 `flags` 的支持][]。**默认值:** `'w'`。
  * `flush` {boolean} 如果所有数据成功写入文件，并且 `flush` 为 `true`，则使用 `fs.fsyncSync()` 刷新数据。**默认值:** `false`。

返回 `undefined`。

`mode` 选项仅影响新创建的文件。有关更多细节，请参见 [`fs.open()`][]。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.writeFile()`][]。

### `fs.writeSync(fd, buffer, offset[, length[, position]])`

<!-- YAML
added: v0.1.21
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `buffer` parameter won't coerce unsupported input to
                 buffers anymore.
  - version: v10.10.0
    pr-url: https://github.com/nodejs/node/pull/22120
    description: The `length` parameter is now optional.
  - version: v7.4.0
    pr-url: https://github.com/nodejs/node/pull/10382
    description: The `position` parameter is optional now.
  - version: v7.2.0
    pr-url: https://github.com/nodejs/node/pull/7856
    description: The `offset` and `length` parameters are optional now.
-->

* `fd` {integer}
* `buffer` {Buffer|TypedArray|DataView}
* `offset` {integer}
* `length` {integer}
* `position` {integer}
* 返回: {integer}

有关详细信息，请参阅此 API 的异步版本文档：[`fs.write(fd, buffer...)`][]。

### `fs.writeSync(fd, buffer[, options])`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

* `fd` {integer}
* `buffer` {Buffer|TypedArray|DataView}
* `options` {Object}
  * `offset` {integer} **默认值:** `0`
  * `length` {integer} **默认值:** `buffer.byteLength - offset`
  * `position` {integer} **默认值:** `null`
* 返回: {integer}

有关详细信息，请参阅此 API 的异步版本文档：[`fs.write(fd, buffer...)`][]。

### `fs.writeSync(fd, string[, position[, encoding]])`

<!-- YAML
added: v0.11.5
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/31030
    description: The `string` parameter won't coerce unsupported input to
                 strings anymore.
  - version: v7.2.0
    pr-url: https://github.com/nodejs/node/pull/7856
    description: The `position` parameter is optional now.
-->

* `fd` {integer}
* `string` {string}
* `position` {integer}
* `encoding` {string} **默认值:** `'utf8'`
* 返回: {integer}

有关详细信息，请参阅此 API 的异步版本文档：[`fs.write(fd, string...)`][]。

### `fs.writevSync(fd, buffers[, position])`

<!-- YAML
added:
  - v12.9.0
  - v10.17.0
-->

* `fd` {integer}
* `buffers` {ArrayBufferView\[]}
* `position` {integer|null} 从文件开头开始写入 `buffers` 数据的偏移量。如果 `position` 不是 `number`，数据将写入当前位置。**默认值:** `null`
* 返回: {integer} 写入的字节数。

有关详细信息，请参阅此 API 的异步版本文档：[`fs.writev()`][]。

## 常见对象

常见对象由所有文件系统 API 变体（promise、回调和同步）共享。

### 类：`fs.Dir`

表示目录流的类。

由 [`fs.opendir()`][]、[`fs.opendirSync()`][] 或 [`fsPromises.opendir()`][] 创建。

```mjs
import { opendir } from 'node:fs/promises';

try {
  const dir = await opendir('./');
  for await (const dirent of dir)
    console.log(dirent.name);
} catch (err) {
  console.error(err);
}
```

当使用异步迭代器时，`fs.Dir` 对象将在迭代器退出后自动关闭。

#### `dir.close()`

* 返回: {Promise}

异步关闭目录的底层资源句柄。后续读取将导致错误。

返回一个 promise，将在资源关闭后兑现。

#### `dir.close(callback)`

* `callback` {Function}
  * `err` {Error}

异步关闭目录的底层资源句柄。后续读取将导致错误。

关闭资源句柄后，将调用 `callback`。

#### `dir.closeSync()`

同步关闭目录的底层资源句柄。后续读取将导致错误。

#### `dir.path`

此目录的只读路径，如提供给 [`fs.opendir()`][]、[`fs.opendirSync()`][] 或 [`fsPromises.opendir()`][]。

#### `dir.read()`

* 返回: {Promise} 使用 {fs.Dirent} 或 `null`（如果目录中没有更多内容可读取）兑现。

通过 readdir(3) 异步读取下一个目录条目作为 {fs.Dirent}。

创建后，首次调用 `dir.read()` 将读取目录中的第一个条目。

如果目录中没有更多内容可读取，则返回 `null`。

#### `dir.read(callback)`

* `callback` {Function}
  * `err` {Error}
  * `dirent` {fs.Dirent|null}

通过 readdir(3) 异步读取下一个目录条目作为 {fs.Dirent}。

创建后，首次调用 `dir.read()` 将读取目录中的第一个条目。

读取完成后，将调用 `callback`，并传入 {fs.Dirent} 或 `null`（如果目录中没有更多内容可读取）。

#### `dir.readSync()`

* 返回: {fs.Dirent|null}

通过 readdir(3) 同步读取下一个目录条目作为 {fs.Dirent}。

创建后，首次调用 `dir.readSync()` 将读取目录中的第一个条目。

如果目录中没有更多内容可读取，则返回 `null`。

#### `dir[Symbol.asyncIterator]()`

* 返回: {AsyncIterator} 的 {fs.Dirent}

异步遍历目录，直到所有条目都被读取完毕。

有关详细信息，请参阅 [异步迭代器][] 的文档。

`dir` 在异步迭代器退出后自动关闭。

```mjs
import { opendir } from 'node:fs/promises';

try {
  const dir = await opendir('./');
  for await (const dirent of dir)
    console.log(dirent.name);
} catch (err) {
  console.error(err);
}
```

### 类：`fs.Dirent`

目录条目（可以是文件或目录）的表示，通过从 {fs.Dir} 读取返回。目录条目是文件名和文件类型对的组合。

此外，当调用 [`fs.readdir()`][] 或 [`fs.readdirSync()`][] 并设置 `withFileTypes` 选项为 `true` 时，结果数组将填充 {fs.Dirent} 对象，而不是字符串或 {Buffer}。

#### `dirent.isBlockDevice()`

* 返回: {boolean}

如果 {fs.Dirent} 对象描述块设备，则返回 `true`。

#### `dirent.isCharacterDevice()`

* 返回: {boolean}

如果 {fs.Dirent} 对象描述字符设备，则返回 `true`。

#### `dirent.isDirectory()`

* 返回: {boolean}

如果 {fs.Dirent} 对象描述文件系统目录，则返回 `true`。

#### `dirent.isFIFO()`

* 返回: {boolean}

如果 {fs.Dirent} 对象描述先进先出（FIFO）管道，则返回 `true`。

#### `dirent.isFile()`

* 返回: {boolean}

如果 {fs.Dirent} 对象描述常规文件，则返回 `true`。

#### `dirent.isSocket()`

* 返回: {boolean}

如果 {fs.Dirent} 对象描述套接字，则返回 `true`。

#### `dirent.isSymbolicLink()`

* 返回: {boolean}

如果 {fs.Dirent} 对象描述符号链接，则返回 `true`。

#### `dirent.name`

此 {fs.Dirent} 对象引用的文件名。此值的类型由传递给 [`fs.readdir()`][] 或 [`fs.readdirSync()`][] 的 `options.encoding` 决定。

### 类：`fs.FSWatcher`

* 继承 {EventEmitter}

成功调用 [`fs.watch()`][] 方法将返回一个新的 {fs.FSWatcher} 对象。

所有 {fs.FSWatcher} 对象在关联的监视器检测到更改时都会发出 `'change'` 事件。`'change'` 事件在每次文件访问时触发，而不仅仅是当文件被修改时。例如，即使文件内容保持不变，在打开文件进行写入时也会触发事件。在 macOS 上，保存文件可能会触发多个事件。例如，在使用常见的编辑器时，可能会触发多个事件。

`'change'` 事件的处理程序接收触发的事件的类型和触发事件的文件名：

```mjs
import { watch } from 'node:fs';
watch('./tmp', { encoding: 'buffer' }, (eventType, filename) => {
  if (filename) {
    console.log(filename);
    // 打印: <Buffer ...>
  }
});
```

根据系统支持，处理程序还会接收触发事件的文件名。

#### 事件：`'change'`

* `eventType` {string} 发生的更改事件的类型
* `filename` {string|Buffer} 更改的文件名（如果相关/可用）

当监视的目录或文件发生更改时触发。

#### 事件：`'close'`

当监视器停止监视更改时触发。关闭的 {fs.FSWatcher} 对象不再可用。

#### 事件：`'error'`

* `error` {Error}

在监视文件时发生错误时触发。出错的 {fs.FSWatcher} 对象不再可用。

#### `watcher.close()`

停止监视给定 {fs.FSWatcher} 上的更改。一旦停止，{fs.FSWatcher} 对象将不再可用。

#### `watcher.ref()`

* 返回: {fs.FSWatcher}

调用时，请求 Node.js 事件循环*不*退出，只要 {fs.FSWatcher} 处于活动状态。多次调用 `watcher.ref()` 将不起作用。

默认情况下，所有 {fs.FSWatcher} 对象都是“引用”的，通常不需要调用 `watcher.ref()`，除非之前调用了 `watcher.unref()`。

#### `watcher.unref()`

* 返回: {fs.FSWatcher}

调用时，活动的 {fs.FSWatcher} 对象将不需要 Node.js 事件循环保持活动状态。如果没有其他活动保持事件循环运行，则进程可能在调用 {fs.FSWatcher} 对象的 `'close'` 事件之前退出。多次调用 `watcher.unref()` 将不起作用。

### 类：`fs.StatWatcher`

* 继承 {EventEmitter}

成功调用 [`fs.watchFile()`][] 方法将返回一个新的 {fs.StatWatcher} 对象。

#### `watcher.ref()`

* 返回: {fs.StatWatcher}

调用时，请求 Node.js 事件循环*不*退出，只要 {fs.StatWatcher} 处于活动状态。多次调用 `watcher.ref()` 将不起作用。

默认情况下，所有 {fs.StatWatcher} 对象都是“引用”的，通常不需要调用 `watcher.ref()`，除非之前调用了 `watcher.unref()`。

#### `watcher.unref()`

* 返回: {fs.StatWatcher}

调用时，活动的 {fs.StatWatcher} 对象将不需要 Node.js 事件循环保持活动状态。如果没有其他活动保持事件循环运行，则进程可能在调用 {fs.StatWatcher} 对象的 `'close'` 事件之前退出。多次调用 `watcher.unref()` 将不起作用。

### 类：`fs.ReadStream`

* 继承 {stream.Readable}

成功调用 [`fs.createReadStream()`][] 将返回一个新的 {fs.ReadStream} 对象。

#### 事件：`'close'`

当 {fs.ReadStream} 的底层文件描述符已关闭时触发。

#### 事件：`'open'`

* `fd` {integer} {fs.ReadStream} 使用的整数文件描述符。

当 {fs.ReadStream} 的文件描述符打开时触发。

#### 事件：`'ready'`

当 {fs.ReadStream} 准备好使用时触发。

在 `'open'` 之后立即触发。

#### `readStream.bytesRead`

到目前为止已读取的字节数。

#### `readStream.path`

流正在读取的文件的路径，如 `fs.createReadStream()` 的第一个参数中所指定。如果 `path` 作为字符串传递，则 `readStream.path` 将是字符串。如果 `path` 作为 {Buffer} 传递，则 `readStream.path` 将是 {Buffer}。如果指定了 `fd`，则 `readStream.path` 将是 `undefined`。

#### `readStream.pending`

如果底层文件尚未打开，即在触发 `'ready'` 事件之前，则此属性为 `true`。

### 类：`fs.Stats`

{fs.Stats} 对象提供关于文件的信息。

从 [`fs.stat()`][]、[`fs.lstat()`][]、[`fs.fstat()`][] 及其同步对应方法返回的对象属于此类型。如果传递给这些方法的 `options` 中的 `bigint` 为 true，则数值将为 `bigint` 而不是 `number`。

```console
Stats {
  dev: 2114,
  ino: 48064969,
  mode: 33188,
  nlink: 1,
  uid: 85,
  gid: 100,
  rdev: 0,
  size: 527,
  blksize: 4096,
  blocks: 8,
  atimeMs: 1318289051000.1,
  mtimeMs: 1318289051000.1,
  ctimeMs: 1318289051000.1,
  birthtimeMs: 1318289051000.1,
  atime: Mon, 10 Oct 2011 23:24:11 GMT,
  mtime: Mon, 10 Oct 2011 23:24:11 GMT,
  ctime: Mon, 10 Oct 2011 23:24:11 GMT,
  birthtime: Mon, 10 Oct 2011 23:24:11 GMT }
```

`bigint` 版本：

```console
Stats {
  dev: 2114n,
  ino: 48064969n,
  mode: 33188n,
  nlink: 1n,
  uid: 85n,
  gid: 100n,
  rdev: 0n,
  size: 527n,
  blksize: 4096n,
  blocks: 8n,
  atimeMs: 1318289051000n,
  mtimeMs: 1318289051000n,
  ctimeMs: 1318289051000n,
  birthtimeMs: 1318289051000n,
  atime: Mon, 10 Oct 2011 23:24:11 GMT,
  mtime: Mon, 10 Oct 2011 23:24:11 GMT,
  ctime: Mon, 10 Oct 2011 23:24:11 GMT,
  birthtime: Mon, 10 Oct 2011 23:24:11 GMT }
```

#### `stats.isBlockDevice()`

* 返回: {boolean}

如果 {fs.Stats} 对象描述块设备，则返回 `true`。

#### `stats.isCharacterDevice()`

* 返回: {boolean}

如果 {fs.Stats} 对象描述字符设备，则返回 `true`。

#### `stats.isDirectory()`

* 返回: {boolean}

如果 {fs.Stats} 对象描述文件系统目录，则返回 `true`。

#### `stats.isFIFO()`

* 返回: {boolean}

如果 {fs.Stats} 对象描述先进先出（FIFO）管道，则返回 `true`。

#### `stats.isFile()`

* 返回: {boolean}

如果 {fs.Stats} 对象描述常规文件，则返回 `true`。

#### `stats.isSocket()`

* 返回: {boolean}

如果 {fs.Stats} 对象描述套接字，则返回 `true`。

#### `stats.isSymbolicLink()`

* 返回: {boolean}

如果 {fs.Stats} 对象描述符号链接，则返回 `true`。

此方法仅在使用 [`fs.lstat()`][] 时有效。

#### `stats.dev`

包含文件的设备的数字标识符。

#### `stats.ino`

文件的文件系统特定索引节点编号。

#### `stats.mode`

描述文件类型和模式的位字段。

#### `stats.nlink`

文件存在的硬链接数。

#### `stats.uid`

文件所有者的数字用户标识符（POSIX）。

#### `stats.gid`

文件所有者的数字组标识符（POSIX）。

#### `stats.rdev`

如果文件表示设备，则为此文件的数字设备标识符。

#### `stats.size`

文件的大小（以字节为单位）。

#### `stats.blksize`

用于 I/O 操作的文件系统块大小。

#### `stats.blocks`

为此文件分配的块数。

#### `stats.atimeMs`

指示上次访问此文件的时间戳，以毫秒为单位，自 POSIX 纪元以来。

#### `stats.mtimeMs`

指示上次修改此文件的时间戳，以毫秒为单位，自 POSIX 纪元以来。

#### `stats.ctimeMs`

指示上次更改文件状态的时间戳，以毫秒为单位，自 POSIX 纪元以来。

#### `stats.birthtimeMs`

指示此文件创建时间的时间戳，以毫秒为单位，自 POSIX 纪元以来。

#### `stats.atime`

指示上次访问此文件的时间戳，自 POSIX 纪元以来。

#### `stats.mtime`

指示上次修改此文件的时间戳，自 POSIX 纪元以来。

#### `stats.ctime`

指示上次更改文件状态的时间戳，自 POSIX 纪元以来。

#### `stats.birthtime`

指示此文件创建时间的时间戳，自 POSIX 纪元以来。

### 类：`fs.StatFs`

提供有关已挂载文件系统的信息。

从 [`fs.statfs()`][]、[`fs.statfsSync()`][]、[`fsPromises.statfs()`][] 返回的对象属于此类型。

`bigint` 版本：

```console
StatFs {
  type: 1397114950,
  bsize: 4096,
  blocks: 121938943,
  bfree: 61058895,
  bavail: 61058895,
  files: 999,
  ffree: 1000000
}
```

如果传递给这些方法的 `options` 中的 `bigint` 为 true，则数值将为 `bigint` 而不是 `number`。

#### `statfs.bavail`

非特权用户的可用块数。

#### `statfs.bfree`

文件系统中的空闲块数。

#### `statfs.blocks`

文件系统中的总数据块数。

#### `statfs.bsize`

最佳传输块大小。

#### `statfs.ffree`

文件系统中的空闲文件节点数。

#### `statfs.files`

文件系统中的总文件节点数。

#### `statfs.type`

文件系统的类型。

### 类：`fs.WriteStream`

* 继承 {stream.Writable}

{fs.WriteStream} 的实例是通过 [`fs.createWriteStream()`][] 创建和返回的。

#### 事件：`'close'`

当 {fs.WriteStream} 的底层文件描述符已关闭时触发。

#### 事件：`'open'`

* `fd` {integer} {fs.WriteStream} 使用的整数文件描述符。

当 {fs.WriteStream} 的文件打开时触发。

#### 事件：`'ready'`

当 {fs.WriteStream} 准备好使用时触发。

在 `'open'` 之后立即触发。

#### `writeStream.bytesWritten`

到目前为止写入的字节数。不包括仍在排队等待写入的数据。

#### `writeStream.close([callback])`

* `callback` {Function}
  * `err` {Error}

如果 `writeStream` 是使用 `autoClose: false` 创建的，则底层文件描述符将保持打开状态。应用程序有责任关闭它。

如果 `writeStream` 是使用 `autoClose: true` 创建的（默认行为），则底层文件描述符将在 `'finish'` 事件或 `'error'` 事件（如果有）触发时自动关闭。

`close()` 方法用于在流仍处于打开状态时关闭流。注册的 `callback` 将在底层文件描述符关闭后触发，除非流是使用 `autoClose: false` 创建的，在这种情况下，回调将不会自动调用。

#### `writeStream.path`

流正在写入的文件的路径，如 `fs.createWriteStream()` 的第一个参数中所指定。如果 `path` 作为字符串传递，则 `writeStream.path` 将是字符串。如果 `path` 作为 {Buffer} 传递，则 `writeStream.path` 将是 {Buffer}。如果指定了 `fd`，则 `writeStream.path` 将是 `undefined`。

#### `writeStream.pending`

如果底层文件尚未打开，即在触发 `'ready'` 事件之前，则此属性为 `true`。

### `fs.constants`

* 返回: {Object}

返回一个包含文件系统操作常用常量的对象。

#### FS 常量

以下常量由 `fs.constants` 导出。

并非每个常量在每个操作系统上都可用；这对于在 Windows 上使用尤其重要，因为许多 POSIX 特定定义不可用。对于可移植应用程序，建议在使用前检查其是否存在。

要使用多个常量，请使用按位或 `|` 运算符。

示例：

```mjs
import { open, constants } from 'node:fs';

const {
  O_RDWR, O_CREAT, O_EXCL
} = constants;

open('/path/to/my/file', O_RDWR | O_CREAT | O_EXCL, (err, fd) => {
  // ...
});
```

##### 文件访问常量

以下常量用作 [`fs.access()`][] 的 `mode` 参数。

| 常量 | 描述 |
| ----------------- | ----------------------------- |
| `F_OK` | 指示文件对调用进程可见的标志。这对于确定文件是否存在很有用，但不对 `rwx` 权限提供任何指示。如果未指定模式，则此为默认值。 |
| `R_OK` | 指示文件可以被调用进程读取的标志。 |
| `W_OK` | 指示文件可以被调用进程写入的标志。 |
| `X_OK` | 指示文件可以被调用进程执行的标志。这在 Windows 上无效（行为类似于 `fs.constants.F_OK`）。 |

##### 文件复制常量

以下常量与 [`fs.copyFile()`][] 一起使用。

| 常量 | 描述 |
| ---------------------- | ----------------------------- |
| `COPYFILE_EXCL` | 如果存在，如果目标路径已存在，复制操作将失败并显示错误。 |
| `COPYFILE_FICLONE` | 如果存在，复制操作将尝试创建写时复制 reflink。如果平台不支持写时复制，则使用回退复制机制。 |
| `COPYFILE_FICLONE_FORCE` | 如果存在，复制操作将尝试创建写时复制 reflink。如果平台不支持写时复制，则操作将失败并显示错误。 |

##### 文件打开常量

以下常量由 `fs.open()` 使用。

| 常量 | 描述 |
| ----------------- | ----------------------------- |
| `O_RDONLY` | 指示打开文件以进行只读访问的标志。 |
| `O_WRONLY` | 指示打开文件以进行只写访问的标志。 |
| `O_RDWR` | 指示打开文件以进行读写访问的标志。 |
| `O_CREAT` | 指示如果文件不存在则创建文件的标志。 |
| `O_EXCL` | 指示如果设置了 `O_CREAT` 标志且文件已存在，则打开文件应失败。 |
| `O_NOCTTY` | 指示如果路径标识终端设备，则打开路径不应导致该终端成为进程的控制终端（如果进程尚未有一个）。 |
| `O_TRUNC` | 指示如果文件存在并且是常规文件，并且文件成功打开以进行写入访问，则其长度应被截断为零。 |
| `O_APPEND` | 指示数据将追加到文件末尾的标志。 |
| `O_DIRECTORY` | 指示如果路径不是目录，则打开应失败。 |
| `O_NOATIME` | 指示文件系统访问将不再导致与文件关联的 `atime` 信息更新的标志。此标志仅在 Linux 操作系统上可用。 |
| `O_NOFOLLOW` | 指示如果路径是符号链接，则打开应失败。 |
| `O_SYNC` | 指示文件打开以进行同步 I/O 的标志。 |
| `O_DSYNC` | 指示文件打开以进行同步 I/O 的标志，写操作等待数据完整性。 |
| `O_SYMLINK` | 指示打开符号链接本身，而不是它指向的资源的标志。 |
| `O_DIRECT` | 设置后，将尝试最小化文件 I/O 的缓存效果。 |
| `O_NONBLOCK` | 指示在可能的情况下以非阻塞模式打开文件的标志。 |
| `O_EVTONLY` | 指示打开文件用于事件通知 only 的标志。 |
| `O_PATH` | 指示文件描述符的获取应用于执行文件系统操作，这些操作使用文件描述符本身而不是其连接的文件。 |

在 Windows 上，仅 `O_APPEND`、`O_CREAT`、`O_EXCL`、`O_RDONLY`、`O_RDWR`、`O_TRUNC`、`O_WRONLY` 和 `UV_FS_O_FILEMAP` 可用。

##### 文件类型常量

以下常量由 {fs.Stats} 对象的 `mode` 属性用于确定文件的类型。

| 常量 | 描述 |
| ----------------- | ----------------------------- |
| `S_IFMT` | 用于提取文件类型代码的位掩码。 |
| `S_IFREG` | 常规文件的文件类型常量。 |
| `S_IFDIR` | 目录的文件类型常量。 |
| `S_IFCHR` | 面向字符的设备文件的文件类型常量。 |
| `S_IFBLK` | 面向块的设备文件的文件类型常量。 |
| `S_IFIFO` | FIFO/管道的文件类型常量。 |
| `S_IFLNK` | 符号链接的文件类型常量。 |
| `S_IFSOCK` | 套接字的文件类型常量。 |

在 Windows 上，只有 `S_IFCHR`、`S_IFDIR`、`S_IFLNK`、`S_IFMT` 和 `S_IFREG` 可用。

##### 文件模式常量

以下常量由 {fs.Stats} 对象的 `mode` 属性用于确定文件的访问权限。

| 常量 | 描述 |
| ----------------- | ----------------------------- |
| `S_IRWXU` | 文件模式指示所有者可读、可写和可执行。 |
| `S_IRUSR` | 文件模式指示所有者可读。 |
| `S_IWUSR` | 文件模式指示所有者可写。 |
| `S_IXUSR` | 文件模式指示所有者可执行。 |
| `S_IRWXG` | 文件模式指示组可读、可写和可执行。 |
| `S_IRGRP` | 文件模式指示组可读。 |
| `S_IWGRP` | 文件模式指示组可写。 |
| `S_IXGRP` | 文件模式指示组可执行。 |
| `S_IRWXO` | 文件模式指示其他人可读、可写和可执行。 |
| `S_IROTH` | 文件模式指示其他人可读。 |
| `S_IWOTH` | 文件模式指示其他人可写。 |
| `S_IXOTH` | 文件模式指示其他人可执行。 |

在 Windows 上，只有 `S_IRUSR` 和 `S_IWUSR` 可用。

## 注意事项

### 有序异步操作

由于它们是由底层线程池异步执行的，因此无法保证顺序。此外，因为每个操作都由一个或多个事件循环迭代执行，所以在操作之间执行其他操作是可能的。

例如，以下操作容易出错，因为 `fs.stat()` 操作可能在 `fs.rename()` 操作之前完成：

```mjs
import { rename, stat } from 'node:fs';

rename('/tmp/hello', '/tmp/world', (err) => {
  if (err) throw err;
  console.log('renamed complete');
});
stat('/tmp/world', (err, stats) => {
  if (err) throw err;
  console.log(`stats: ${JSON.stringify(stats)}`);
});
```

通过在调用下一个操作之前等待前一个操作的结果，可以正确地排序操作：

```mjs
import { rename, stat } from 'node:fs';

rename('/tmp/hello', '/tmp/world', (err) => {
  if (err) throw err;
  stat('/tmp/world', (err, stats) => {
    if (err) throw err;
    console.log(`stats: ${JSON.stringify(stats)}`);
  });
});
```

或者，使用基于 promise 的 API：

```mjs
import { rename, stat } from 'node:fs/promises';

const oldPath = '/tmp/hello';
const newPath = '/tmp/world';

try {
  await rename(oldPath, newPath);
  const stats = await stat(newPath);
  console.log(`stats: ${JSON.stringify(stats)}`);
} catch (error) {
  console.error('there was an error:', error.message);
}
```

### 文件路径

大多数 `fs` 操作接受的文件路径可以指定为字符串、{Buffer} 或使用 `file:` 协议的 {URL} 对象。

#### 字符串路径

字符串路径被解释为标识绝对或相对文件名的 UTF-8 字符序列。相对路径将相对于通过调用 `process.cwd()` 确定的当前工作目录进行解析。

在 POSIX 上使用绝对路径的示例：

```mjs
import { open } from 'node:fs/promises';

let fd;
try {
  fd = await open('/open/some/file.txt', 'r');
  // 对文件进行操作
} finally {
  await fd?.close();
}
```

在 POSIX 上使用相对路径的示例（相对于 `process.cwd()`）：

```mjs
import { open } from 'node:fs/promises';

let fd;
try {
  fd = await open('file.txt', 'r');
  // 对文件进行操作
} finally {
  await fd?.close();
}
```

#### 文件 URL 路径

<!-- YAML
added: v7.6.0
-->

对于大多数 `node:fs` 模块函数，`path` 或 `filename` 参数可以作为使用 `file:` 协议的 {URL} 对象传递。

```mjs
import { readFileSync } from 'node:fs';

readFileSync(new URL('file:///tmp/hello'));
```

`file:` URL 始终是绝对路径。

##### 平台特定注意事项

在 Windows 上，带有主机名的 `file:` {URL} 会转换为 UNC 路径，而带有驱动器号的 `file:` {URL} 会转换为本地绝对路径。没有主机名和驱动器号的 `file:` {URL} 会导致错误：

```mjs
import { readFileSync } from 'node:fs';
// 在 Windows 上：

// - 带有主机名的 WHATWG 文件 URL 转换为 UNC 路径
// file://hostname/p/a/t/h/file => \\hostname\p\a\t\h\file
readFileSync(new URL('file://hostname/p/a/t/h/file'));

// - 带有驱动器号的 WHATWG 文件 URL 转换为绝对路径
// file:///C:/tmp/hello => C:\tmp\hello
readFileSync(new URL('file:///C:/tmp/hello'));

// - 没有主机名的 WHATWG 文件 URL 必须包含驱动器号
readFileSync(new URL('file:///notdriveletter/p/a/t/h/file'));
readFileSync(new URL('file:///c/p/a/t/h/file'));
// TypeError [ERR_INVALID_FILE_URL_PATH]: File URL path must be absolute
```

带有驱动器号的 `file:` {URL} 必须使用 `:` 作为驱动器号后的分隔符。使用其他分隔符会导致错误。

在所有其他平台上，不支持带有主机名的 `file:` {URL} 并会导致错误：

```mjs
import { readFileSync } from 'node:fs';
// 在其他平台上：

// - 不支持带有主机名的 WHATWG 文件 URL
// file://hostname/p/a/t/h/file => 抛出错误！
readFileSync(new URL('file://hostname/p/a/t/h/file'));
// TypeError [ERR_INVALID_FILE_URL_PATH]: must be absolute

// - WHATWG 文件 URL 转换为绝对路径
// file:///tmp/hello => /tmp/hello
readFileSync(new URL('file:///tmp/hello'));
```

在所有平台上，包含编码斜杠字符的 `file:` {URL} 会导致错误：

```mjs
import { readFileSync } from 'node:fs';

// 在 Windows 上
readFileSync(new URL('file:///C:/p/a/t/h/%2F'));
readFileSync(new URL('file:///C:/p/a/t/h/%2f'));
/* TypeError [ERR_INVALID_FILE_URL_PATH]: File URL path must not include encoded
\ or / characters */

// 在 POSIX 上
readFileSync(new URL('file:///p/a/t/h/%2F'));
readFileSync(new URL('file:///p/a/t/h/%2f'));
/* TypeError [ERR_INVALID_FILE_URL_PATH]: File URL path must not include encoded
/ characters */
```

在 Windows 上，包含编码反斜杠的 `file:` {URL} 会导致错误：

```mjs
import { readFileSync } from 'node:fs';

// 在 Windows 上
readFileSync(new URL('file:///C:/path/%5C'));
readFileSync(new URL('file:///C:/path/%5c'));
/* TypeError [ERR_INVALID_FILE_URL_PATH]: File URL path must not include encoded
\ or / characters */
```

#### Buffer 路径

使用 {Buffer} 指定的路径主要在某些将文件路径视为不透明字节序列的 POSIX 操作系统上有用。在此类系统上，单个文件路径可能包含使用多种字符编码的子序列。与字符串路径一样，{Buffer} 路径可以是相对的或绝对的：

在 POSIX 上使用绝对路径的示例：

```mjs
import { open } from 'node:fs/promises';
import { Buffer } from 'node:buffer';

let fd;
try {
  fd = await open(Buffer.from('/open/some/file.txt'), 'r');
  // 对文件进行操作
} finally {
  await fd?.close();
}
```

#### Windows 上按驱动器的工作目录

在 Windows 上，Node.js 遵循按驱动器工作目录的概念。当使用不带反斜杠的驱动器路径时，可以观察到这种行为。例如，`fs.readdirSync('C:\\')` 可能返回与 `fs.readdirSync('C:')` 不同的结果。更多信息请参阅 [此 MSDN 页面][MSDN-Rel-Path]。

### 文件描述符

在 POSIX 系统上，内核为每个进程维护一个当前打开文件和资源的表格。每个打开的文件被分配一个称为_文件描述符_的简单数字标识符。在系统级别，所有文件系统操作都使用这些文件描述符来识别和跟踪每个特定文件。Windows 系统使用不同但在概念上类似的机制来跟踪资源。为了简化用户操作，Node.js 抽象了操作系统之间的差异，并为所有打开的文件分配一个数字文件描述符。

基于回调的 `fs.open()` 和同步的 `fs.openSync()` 方法打开文件并分配一个新的文件描述符。一旦分配，文件描述符可用于从文件读取数据、向文件写入数据或请求有关文件的信息。

操作系统限制了在任何给定时间可能打开的文件描述符数量，因此在操作完成时关闭描述符至关重要。否则将导致内存泄漏，最终导致应用程序崩溃。

```mjs
import { open, close, fstat } from 'node:fs';

function closeFd(fd) {
  close(fd, (err) => {
    if (err) throw err;
  });
}

open('/open/some/file.txt', 'r', (err, fd) => {
  if (err) throw err;
  try {
    fstat(fd, (err, stat) => {
      if (err) {
        closeFd(fd);
        throw err;
      }

      // 使用 stat

      closeFd(fd);
    });
  } catch (err) {
    closeFd(fd);
    throw err;
  }
});
```

基于 Promise 的 API 使用 {FileHandle} 对象代替数字文件描述符。这些对象由系统更好地管理，以确保资源不会泄漏。但是，在操作完成时仍然需要关闭它们：

```mjs
import { open } from 'node:fs/promises';

let file;
try {
  file = await open('/open/some/file.txt', 'r');
  const stat = await file.stat();
  // 使用 stat
} finally {
  await file.close();
}
```

### 线程池使用

所有基于回调和 Promise 的文件系统 API（除了 `fs.FSWatcher()`）都使用 libuv 的线程池。这对某些应用程序可能产生令人惊讶和负面的性能影响。更多信息请参阅 [`UV_THREADPOOL_SIZE`][] 文档。

### 文件系统标志

以下标志在 `flag` 选项接受字符串的任何地方可用。

* `'a'`: 打开文件进行追加。如果文件不存在，则创建该文件。

* `'ax'`: 类似于 `'a'`，但如果路径存在则失败。

* `'a+'`: 打开文件进行读取和追加。如果文件不存在，则创建该文件。

* `'ax+'`: 类似于 `'a+'`，但如果路径存在则失败。

* `'as'`: 以同步模式打开文件进行追加。如果文件不存在，则创建该文件。

* `'as+'`: 以同步模式打开文件进行读取和追加。如果文件不存在，则创建该文件。

* `'r'`: 打开文件进行读取。如果文件不存在，则抛出异常。

* `'rs'`: 以同步模式打开文件进行读取。如果文件不存在，则抛出异常。

* `'r+'`: 打开文件进行读取和写入。如果文件不存在，则抛出异常。

* `'rs+'`: 以同步模式打开文件进行读取和写入。指示操作系统绕过本地文件系统缓存。

  这主要用于在 NFS 挂载上打开文件，因为它允许跳过可能过时的本地缓存。这对 I/O 性能有非常实际的影响，因此除非需要，否则不建议使用此标志。

  这不会将 `fs.open()` 或 `fsPromises.open()` 变为同步阻塞调用。如果需要同步操作，应使用类似 `fs.openSync()` 的方法。

* `'w'`: 打开文件进行写入。如果文件不存在则创建，如果存在则截断。

* `'wx'`: 类似于 `'w'`，但如果路径存在则失败。

* `'w+'`: 打开文件进行读取和写入。如果文件不存在则创建，如果存在则截断。

* `'wx+'`: 类似于 `'w+'`，但如果路径存在则失败。

`flag` 也可以是 open(2) 文档中记录的数字；常用常量可从 `fs.constants` 获取。在 Windows 上，标志会转换为等效的标志（如果适用），例如 `O_WRONLY` 转换为 `FILE_GENERIC_WRITE`，或 `O_EXCL|O_CREAT` 转换为 `CREATE_NEW`，如 `CreateFileW` 所接受。

独占标志 `'x'`（open(2) 中的 `O_EXCL` 标志）导致操作在路径已存在时返回错误。在 POSIX 上，如果路径是符号链接，使用 `O_EXCL` 会返回错误，即使链接指向不存在的路径。独占标志可能不适用于网络文件系统。

在 Linux 上，当文件以追加模式打开时，位置写入不起作用。内核忽略位置参数，始终将数据追加到文件末尾。

修改文件而不是替换它可能需要将 `flag` 选项设置为 `'r+'` 而不是默认的 `'w'`。

某些标志的行为是特定于平台的。因此，在 macOS 和 Linux 上使用 `'a+'` 标志打开目录（如下例所示）将返回错误。相比之下，在 Windows 和 FreeBSD 上，将返回文件描述符或 `FileHandle`。

```js
// macOS 和 Linux
fs.open('<directory>', 'a+', (err, fd) => {
  // => [Error: EISDIR: illegal operation on a directory, open <directory>]
});

// Windows 和 FreeBSD
fs.open('<directory>', 'a+', (err, fd) => {
  // => null, <fd>
});
```

在 Windows 上，使用 `'w'` 标志（通过 `fs.open()`、`fs.writeFile()` 或 `fsPromises.open()`）打开现有的隐藏文件将失败并显示 `EPERM`。可以使用 `'r+'` 标志打开现有的隐藏文件进行写入。

调用 `fs.ftruncate()` 或 `filehandle.truncate()` 可用于重置文件内容。

[#25741]: https://github.com/nodejs/node/issues/25741
[Common System Errors]: errors.md#common-system-errors
[FS constants]: #fs-constants
[File access constants]: #file-access-constants
[MDN-Date]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Date
[MDN-Number]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Data_structures#Number_type
[MSDN-Rel-Path]: https://docs.microsoft.com/en-us/windows/desktop/FileIO/naming-a-file#fully-qualified-vs-relative-paths
[MSDN-Using-Streams]: https://docs.microsoft.com/en-us/windows/desktop/FileIO/using-streams
[Naming Files, Paths, and Namespaces]: https://docs.microsoft.com/en-us/windows/desktop/FileIO/naming-a-file
[`AHAFS`]: https://developer.ibm.com/articles/au-aix_event_infrastructure/
[`Buffer.byteLength`]: buffer.md#static-method-bufferbytelengthstring-encoding
[`FSEvents`]: https://developer.apple.com/documentation/coreservices/file_system_events
[`Number.MAX_SAFE_INTEGER`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Number/MAX_SAFE_INTEGER
[`ReadDirectoryChangesW`]: https://docs.microsoft.com/en-us/windows/desktop/api/winbase/nf-winbase-readdirectorychangesw
[`UV_THREADPOOL_SIZE`]: cli.md#uv_threadpool_sizesize
[`event ports`]: https://illumos.org/man/port_create
[`filehandle.createReadStream()`]: #filehandlecreatereadstreamoptions
[`filehandle.createWriteStream()`]: #filehandlecreatewritestreamoptions
[`filehandle.writeFile()`]: #filehandlewritefiledata-options
[`fs.access()`]: #fsaccesspath-mode-callback
[`fs.accessSync()`]: #fsaccesssyncpath-mode
[`fs.chmod()`]: #fschmodpath-mode-callback
[`fs.chown()`]: #fschownpath-uid-gid-callback
[`fs.copyFile()`]: #fscopyfilesrc-dest-mode-callback
[`fs.copyFileSync()`]: #fscopyfilesyncsrc-dest-mode
[`fs.createReadStream()`]: #fscreatereadstreampath-options
[`fs.createWriteStream()`]: #fscreatewritestreampath-options
[`fs.exists()`]: #fsexistspath-callback
[`fs.fstat()`]: #fsfstatfd-options-callback
[`fs.ftruncate()`]: #fsftruncatefd-len-callback
[`fs.futimes()`]: #fsfutimesfd-atime-mtime-callback
[`fs.lstat()`]: #fslstatpath-options-callback
[`fs.lutimes()`]: #fslutimespath-atime-mtime-callback
[`fs.mkdir()`]: #fsmkdirpath-options-callback
[`fs.mkdtemp()`]: #fsmkdtempprefix-options-callback
[`fs.open()`]: #fsopenpath-flags-mode-callback
[`fs.opendir()`]: #fsopendirpath-options-callback
[`fs.opendirSync()`]: #fsopendirsyncpath-options
[`fs.read()`]: #fsreadfd-buffer-offset-length-position-callback
[`fs.readFile()`]: #fsreadfilepath-options-callback
[`fs.readFileSync()`]: #fsreadfilesyncpath-options
[`fs.readdir()`]: #fsreaddirpath-options-callback
[`fs.readdirSync()`]: #fsreaddirsyncpath-options
[`fs.readv()`]: #fsreadvfd-buffers-position-callback
[`fs.realpath()`]: #fsrealpathpath-options-callback
[`fs.rm()`]: #fsrmpath-options-callback
[`fs.rmSync()`]: #fsrmsyncpath-options
[`fs.rmdir()`]: #fsrmdirpath-options-callback
[`fs.stat()`]: #fsstatpath-options-callback
[`fs.statfs()`]: #fsstatfspath-options-callback
[`fs.symlink()`]: #fssymlinktarget-path-type-callback
[`fs.utimes()`]: #fsutimespath-atime-mtime-callback
[`fs.watch()`]: #fswatchfilename-options-listener
[`fs.write(fd, buffer...)`]: #fswritefd-buffer-offset-length-position-callback
[`fs.write(fd, string...)`]: #fswritefd-string-position-encoding-callback
[`fs.writeFile()`]: #fswritefilefile-data-options-callback
[`fs.writev()`]: #fswritevfd-buffers-position-callback
[`fsPromises.access()`]: #fspromisesaccesspath-mode
[`fsPromises.copyFile()`]: #fspromisescopyfilesrc-dest-mode
[`fsPromises.mkdtemp()`]: #fspromisesmkdtempprefix-options
[`fsPromises.open()`]: #fspromisesopenpath-flags-mode
[`fsPromises.opendir()`]: #fspromisesopendirpath-options
[`fsPromises.rm()`]: #fspromisesrmpath-options
[`fsPromises.stat()`]: #fspromisesstatpath-options
[`fsPromises.utimes()`]: #fspromisesutimespath-atime-mtime
[`inotify(7)`]: https://man7.org/linux/man-pages/man7/inotify.7.html
[`kqueue(2)`]: https://www.freebsd.org/cgi/man.cgi?query=kqueue&sektion=2
[`util.promisify()`]: util.md#utilpromisifyoriginal
[bigints]: https://tc39.github.io/proposal-bigint
[caveats]: #caveats
[chcp]: https://ss64.com/nt/chcp.html
[inode]: https://en.wikipedia.org/wiki/Inode
[support of file system `flags`]: #file-system-flags