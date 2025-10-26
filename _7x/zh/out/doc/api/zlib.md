# Zlib

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/zlib.js -->

`node:zlib` 模块提供了使用 Gzip、Deflate/Inflate、Brotli 和 Zstd 实现的压缩功能。

要访问它：

```mjs
import zlib from 'node:zlib';
```

```cjs
const zlib = require('node:zlib');
```

压缩和解压缩是围绕 Node.js [流 API][] 构建的。

通过将源流通过 `zlib` 的 `Transform` 流管道传输到目标流，可以完成对流（例如文件）的压缩或解压缩：

```mjs
import {
  createReadStream,
  createWriteStream,
} from 'node:fs';
import process from 'node:process';
import { createGzip } from 'node:zlib';
import { pipeline } from 'node:stream';

const gzip = createGzip();
const source = createReadStream('input.txt');
const destination = createWriteStream('input.txt.gz');

pipeline(source, gzip, destination, (err) => {
  if (err) {
    console.error('An error occurred:', err);
    process.exitCode = 1;
  }
});
```

```cjs
const {
  createReadStream,
  createWriteStream,
} = require('node:fs');
const process = require('node:process');
const { createGzip } = require('node:zlib');
const { pipeline } = require('node:stream');

const gzip = createGzip();
const source = createReadStream('input.txt');
const destination = createWriteStream('input.txt.gz');

pipeline(source, gzip, destination, (err) => {
  if (err) {
    console.error('An error occurred:', err);
    process.exitCode = 1;
  }
});
```

或者，使用基于 Promise 的 `pipeline` API：

```mjs
import {
  createReadStream,
  createWriteStream,
} from 'node:fs';
import { createGzip } from 'node:zlib';
import { pipeline } from 'node:stream/promises';

async function do_gzip(input, output) {
  const gzip = createGzip();
  const source = createReadStream(input);
  const destination = createWriteStream(output);
  await pipeline(source, gzip, destination);
}

await do_gzip('input.txt', 'input.txt.gz');
```

```cjs
const {
  createReadStream,
  createWriteStream,
} = require('node:fs');
const process = require('node:process');
const { createGzip } = require('node:zlib');
const { pipeline } = require('node:stream/promises');

async function do_gzip(input, output) {
  const gzip = createGzip();
  const source = createReadStream(input);
  const destination = createWriteStream(output);
  await pipeline(source, gzip, destination);
}

do_gzip('input.txt', 'input.txt.gz')
  .catch((err) => {
    console.error('An error occurred:', err);
    process.exitCode = 1;
  });
```

也可以一步完成数据的压缩或解压缩：

```mjs
import process from 'node:process';
import { Buffer } from 'node:buffer';
import { deflate, unzip } from 'node:zlib';

const input = '.................................';
deflate(input, (err, buffer) => {
  if (err) {
    console.error('An error occurred:', err);
    process.exitCode = 1;
  }
  console.log(buffer.toString('base64'));
});

const buffer = Buffer.from('eJzT0yMAAGTvBe8=', 'base64');
unzip(buffer, (err, buffer) => {
  if (err) {
    console.error('An error occurred:', err);
    process.exitCode = 1;
  }
  console.log(buffer.toString());
});

// 或者，使用 Promise 版本

import { promisify } from 'node:util';
const do_unzip = promisify(unzip);

const unzippedBuffer = await do_unzip(buffer);
console.log(unzippedBuffer.toString());
```

```cjs
const { deflate, unzip } = require('node:zlib');

const input = '.................................';
deflate(input, (err, buffer) => {
  if (err) {
    console.error('An error occurred:', err);
    process.exitCode = 1;
  }
  console.log(buffer.toString('base64'));
});

const buffer = Buffer.from('eJzT0yMAAGTvBe8=', 'base64');
unzip(buffer, (err, buffer) => {
  if (err) {
    console.error('An error occurred:', err);
    process.exitCode = 1;
  }
  console.log(buffer.toString());
});

// 或者，使用 Promise 版本

const { promisify } = require('node:util');
const do_unzip = promisify(unzip);

do_unzip(buffer)
  .then((buf) => console.log(buf.toString()))
  .catch((err) => {
    console.error('An error occurred:', err);
    process.exitCode = 1;
  });
```

## 线程池使用和性能考虑

除了那些显式同步的 API 之外，所有 `zlib` API 都使用 Node.js 内部的线程池。这可能会导致一些应用程序中出现意想不到的效果和性能限制。

同时创建和使用大量 zlib 对象会导致显著的内存碎片。

```mjs
import zlib from 'node:zlib';
import { Buffer } from 'node:buffer';

const payload = Buffer.from('This is some data');

// 警告：不要这样做！
for (let i = 0; i < 30000; ++i) {
  zlib.deflate(payload, (err, buffer) => {});
}
```

```cjs
const zlib = require('node:zlib');

const payload = Buffer.from('This is some data');

// 警告：不要这样做！
for (let i = 0; i < 30000; ++i) {
  zlib.deflate(payload, (err, buffer) => {});
}
```

在前面的例子中，同时创建了 30,000 个 deflate 实例。由于某些操作系统处理内存分配和释放的方式，这可能导致显著的内存碎片。

强烈建议缓存压缩操作的结果，以避免重复工作。

## 压缩 HTTP 请求和响应

`node:zlib` 模块可用于实现对 [HTTP](https://tools.ietf.org/html/rfc7230#section-4.2) 定义的 `gzip`、`deflate`、`br` 和 `zstd` 内容编码机制的支持。

HTTP [`Accept-Encoding`][] 标头在 HTTP 请求中用于标识客户端接受的压缩编码。[`Content-Encoding`][] 标头用于标识实际应用于消息的压缩编码。

下面给出的示例被大幅简化以展示基本概念。使用 `zlib` 编码可能很昂贵，结果应该被缓存。有关 `zlib` 使用中涉及的速度/内存/压缩权衡的更多信息，请参阅[内存使用调优][]。

```mjs
// 客户端请求示例
import fs from 'node:fs';
import zlib from 'node:zlib';
import http from 'node:http';
import process from 'node:process';
import { pipeline } from 'node:stream';

const request = http.get({ host: 'example.com',
                           path: '/',
                           port: 80,
                           headers: { 'Accept-Encoding': 'br,gzip,deflate' } });
request.on('response', (response) => {
  const output = fs.createWriteStream('example.com_index.html');

  const onError = (err) => {
    if (err) {
      console.error('An error occurred:', err);
      process.exitCode = 1;
    }
  };

  switch (response.headers['content-encoding']) {
    case 'br':
      pipeline(response, zlib.createBrotliDecompress(), output, onError);
      break;
    // 或者，直接使用 zlib.createUnzip() 来处理以下两种情况：
    case 'gzip':
      pipeline(response, zlib.createGunzip(), output, onError);
      break;
    case 'deflate':
      pipeline(response, zlib.createInflate(), output, onError);
      break;
    default:
      pipeline(response, output, onError);
      break;
  }
});
```

```cjs
// 客户端请求示例
const zlib = require('node:zlib');
const http = require('node:http');
const fs = require('node:fs');
const { pipeline } = require('node:stream');

const request = http.get({ host: 'example.com',
                           path: '/',
                           port: 80,
                           headers: { 'Accept-Encoding': 'br,gzip,deflate,zstd' } });
request.on('response', (response) => {
  const output = fs.createWriteStream('example.com_index.html');

  const onError = (err) => {
    if (err) {
      console.error('An error occurred:', err);
      process.exitCode = 1;
    }
  };

  switch (response.headers['content-encoding']) {
    case 'br':
      pipeline(response, zlib.createBrotliDecompress(), output, onError);
      break;
    // 或者，直接使用 zlib.createUnzip() 来处理以下两种情况：
    case 'gzip':
      pipeline(response, zlib.createGunzip(), output, onError);
      break;
    case 'deflate':
      pipeline(response, zlib.createInflate(), output, onError);
      break;
    case 'zstd':
      pipeline(response, zlib.createZstdDecompress(), output, onError);
      break;
    default:
      pipeline(response, output, onError);
      break;
  }
});
```

```mjs
// 服务器示例
// 在每个请求上运行 gzip 操作相当昂贵。
// 缓存压缩后的缓冲区会更高效。
import zlib from 'node:zlib';
import http from 'node:http';
import fs from 'node:fs';
import { pipeline } from 'node:stream';

http.createServer((request, response) => {
  const raw = fs.createReadStream('index.html');
  // 存储资源的压缩和未压缩版本。
  response.setHeader('Vary', 'Accept-Encoding');
  const acceptEncoding = request.headers['accept-encoding'] || '';

  const onError = (err) => {
    if (err) {
      // 如果发生错误，我们无能为力，因为
      // 服务器已经发送了 200 响应代码，并且
      // 一定数量的数据已经发送给客户端。
      // 我们能做的最好是立即终止响应
      // 并记录错误。
      response.end();
      console.error('An error occurred:', err);
    }
  };

  // 注意：这不是一个符合规范的 accept-encoding 解析器。
  // 参见 https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.3
  if (/\bdeflate\b/.test(acceptEncoding)) {
    response.writeHead(200, { 'Content-Encoding': 'deflate' });
    pipeline(raw, zlib.createDeflate(), response, onError);
  } else if (/\bgzip\b/.test(acceptEncoding)) {
    response.writeHead(200, { 'Content-Encoding': 'gzip' });
    pipeline(raw, zlib.createGzip(), response, onError);
  } else if (/\bbr\b/.test(acceptEncoding)) {
    response.writeHead(200, { 'Content-Encoding': 'br' });
    pipeline(raw, zlib.createBrotliCompress(), response, onError);
  } else {
    response.writeHead(200, {});
    pipeline(raw, response, onError);
  }
}).listen(1337);
```

```cjs
// 服务器示例
// 在每个请求上运行 gzip 操作相当昂贵。
// 缓存压缩后的缓冲区会更高效。
const zlib = require('node:zlib');
const http = require('node:http');
const fs = require('node:fs');
const { pipeline } = require('node:stream');

http.createServer((request, response) => {
  const raw = fs.createReadStream('index.html');
  // 存储资源的压缩和未压缩版本。
  response.setHeader('Vary', 'Accept-Encoding');
  const acceptEncoding = request.headers['accept-encoding'] || '';

  const onError = (err) => {
    if (err) {
      // 如果发生错误，我们无能为力，因为
      // 服务器已经发送了 200 响应代码，并且
      // 一定数量的数据已经发送给客户端。
      // 我们能做的最好是立即终止响应
      // 并记录错误。
      response.end();
      console.error('An error occurred:', err);
    }
  };

  // 注意：这不是一个符合规范的 accept-encoding 解析器。
  // 参见 https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.3
  if (/\bdeflate\b/.test(acceptEncoding)) {
    response.writeHead(200, { 'Content-Encoding': 'deflate' });
    pipeline(raw, zlib.createDeflate(), response, onError);
  } else if (/\bgzip\b/.test(acceptEncoding)) {
    response.writeHead(200, { 'Content-Encoding': 'gzip' });
    pipeline(raw, zlib.createGzip(), response, onError);
  } else if (/\bbr\b/.test(acceptEncoding)) {
    response.writeHead(200, { 'Content-Encoding': 'br' });
    pipeline(raw, zlib.createBrotliCompress(), response, onError);
  } else if (/\bzstd\b/.test(acceptEncoding)) {
    response.writeHead(200, { 'Content-Encoding': 'zstd' });
    pipeline(raw, zlib.createZstdCompress(), response, onError);
  } else {
    response.writeHead(200, {});
    pipeline(raw, response, onError);
  }
}).listen(1337);
```

默认情况下，当解压缩截断的数据时，`zlib` 方法会抛出错误。但是，如果已知数据不完整，或者希望仅检查压缩文件的开头，可以通过更改用于解压缩最后一块输入数据的刷新方法来抑制默认的错误处理：

```js
// 这是上述示例中缓冲区的截断版本
const buffer = Buffer.from('eJzT0yMA', 'base64');

zlib.unzip(
  buffer,
  // 对于 Brotli，等效的是 zlib.constants.BROTLI_OPERATION_FLUSH。
  // 对于 Zstd，等效的是 zlib.constants.ZSTD_e_flush。
  { finishFlush: zlib.constants.Z_SYNC_FLUSH },
  (err, buffer) => {
    if (err) {
      console.error('An error occurred:', err);
      process.exitCode = 1;
    }
    console.log(buffer.toString());
  });
```

这不会改变其他抛出错误情况的行为，例如当输入数据格式无效时。使用此方法，将无法确定输入是过早结束还是缺少完整性检查，因此需要手动检查解压缩结果是否有效。

## 内存使用调优

<!--type=misc-->

### 对于基于 zlib 的流

来自 `zlib/zconf.h`，针对 Node.js 使用进行了修改：

deflate 的内存需求（以字节为单位）为：

<!-- eslint-disable @stylistic/js/semi -->

```js
(1 << (windowBits + 2)) + (1 << (memLevel + 9))
```

即：`windowBits` = 15 时 128K + `memLevel` = 8（默认值）时 128K，再加上小对象的几千字节。

例如，要将默认内存需求从 256K 减少到 128K，应设置选项为：

```js
const options = { windowBits: 14, memLevel: 7 };
```

然而，这通常会降低压缩率。

inflate 的内存需求（以字节为单位）为 `1 << windowBits`。
即，`windowBits` = 15（默认值）时 32K，再加上小对象的几千字节。

除此之外，还有一个大小为 `chunkSize` 的单个内部输出 slab 缓冲区，默认为 16K。

`zlib` 压缩的速度受 `level` 设置的影响最为显著。较高的级别将导致更好的压缩率，但需要更长时间完成。较低的级别将导致较少的压缩，但速度会快得多。

通常，更大的内存使用选项意味着 Node.js 需要调用 `zlib` 的次数更少，因为它能够在每次 `write` 操作时处理更多数据。所以，这是影响速度的另一个因素，但代价是内存使用量。

### 对于基于 Brotli 的流

对于基于 Brotli 的流，有与 zlib 选项对应的选项，尽管这些选项的范围与 zlib 不同：

* zlib 的 `level` 选项对应 Brotli 的 `BROTLI_PARAM_QUALITY` 选项。
* zlib 的 `windowBits` 选项对应 Brotli 的 `BROTLI_PARAM_LGWIN` 选项。

有关 Brotli 特定选项的更多详细信息，请参见[下文][Brotli 参数]。

### 对于基于 Zstd 的流

> Stability: 1 - Experimental

对于基于 Zstd 的流，有与 zlib 选项对应的选项，尽管这些选项的范围与 zlib 不同：

* zlib 的 `level` 选项对应 Zstd 的 `ZSTD_c_compressionLevel` 选项。
* zlib 的 `windowBits` 选项对应 Zstd 的 `ZSTD_c_windowLog` 选项。

有关 Zstd 特定选项的更多详细信息，请参见[下文][Zstd 参数]。

## 刷新

在压缩流上调用 [`.flush()`][] 将使 `zlib` 返回当前尽可能多的输出。这可能会以降低压缩质量为代价，但在数据需要尽快可用时非常有用。

在以下示例中，`flush()` 用于将压缩的部分 HTTP 响应写入客户端：

```mjs
import zlib from 'node:zlib';
import http from 'node:http';
import { pipeline } from 'node:stream';

http.createServer((request, response) => {
  // 为简单起见，省略了 Accept-Encoding 检查。
  response.writeHead(200, { 'content-encoding': 'gzip' });
  const output = zlib.createGzip();
  let i;

  pipeline(output, response, (err) => {
    if (err) {
      // 如果发生错误，我们无能为力，因为
      // 服务器已经发送了 200 响应代码，并且
      // 一定数量的数据已经发送给客户端。
      // 我们能做的最好是立即终止响应
      // 并记录错误。
      clearInterval(i);
      response.end();
      console.error('An error occurred:', err);
    }
  });

  i = setInterval(() => {
    output.write(`The current time is ${Date()}\n`, () => {
      // 数据已传递给 zlib，但压缩算法可能
      // 决定缓冲数据以实现更高效的压缩。
      // 调用 .flush() 将使数据在客户端准备接收时立即可用。
      output.flush();
    });
  }, 1000);
}).listen(1337);
```

```cjs
const zlib = require('node:zlib');
const http = require('node:http');
const { pipeline } = require('node:stream');

http.createServer((request, response) => {
  // 为简单起见，省略了 Accept-Encoding 检查。
  response.writeHead(200, { 'content-encoding': 'gzip' });
  const output = zlib.createGzip();
  let i;

  pipeline(output, response, (err) => {
    if (err) {
      // 如果发生错误，我们无能为力，因为
      // 服务器已经发送了 200 响应代码，并且
      // 一定数量的数据已经发送给客户端。
      // 我们能做的最好是立即终止响应
      // 并记录错误。
      clearInterval(i);
      response.end();
      console.error('An error occurred:', err);
    }
  });

  i = setInterval(() => {
    output.write(`The current time is ${Date()}\n`, () => {
      // 数据已传递给 zlib，但压缩算法可能
      // 决定缓冲数据以实现更高效的压缩。
      // 调用 .flush() 将使数据在客户端准备接收时立即可用。
      output.flush();
    });
  }, 1000);
}).listen(1337);
```

## 常量

<!-- YAML
added: v0.5.8
-->

<!--type=misc-->

### zlib 常量

`zlib.h` 中定义的所有常量也在 `require('node:zlib').constants` 上定义。在正常操作过程中，不需要使用这些常量。此处记录它们是为了让它们的存在不令人意外。本节几乎直接取自 [zlib 文档][]。

以前，常量可以直接从 `require('node:zlib')` 访问，例如 `zlib.Z_NO_FLUSH`。目前仍然可以直接从模块访问常量，但已弃用。

允许的刷新值。

* `zlib.constants.Z_NO_FLUSH`
* `zlib.constants.Z_PARTIAL_FLUSH`
* `zlib.constants.Z_SYNC_FLUSH`
* `zlib.constants.Z_FULL_FLUSH`
* `zlib.constants.Z_FINISH`
* `zlib.constants.Z_BLOCK`

压缩/解压缩函数的返回码。负值是错误，正值用于特殊但正常的事件。

* `zlib.constants.Z_OK`
* `zlib.constants.Z_STREAM_END`
* `zlib.constants.Z_NEED_DICT`
* `zlib.constants.Z_ERRNO`
* `zlib.constants.Z_STREAM_ERROR`
* `zlib.constants.Z_DATA_ERROR`
* `zlib.constants.Z_MEM_ERROR`
* `zlib.constants.Z_BUF_ERROR`
* `zlib.constants.Z_VERSION_ERROR`

压缩级别。

* `zlib.constants.Z_NO_COMPRESSION`
* `zlib.constants.Z_BEST_SPEED`
* `zlib.constants.Z_BEST_COMPRESSION`
* `zlib.constants.Z_DEFAULT_COMPRESSION`

压缩策略。

* `zlib.constants.Z_FILTERED`
* `zlib.constants.Z_HUFFMAN_ONLY`
* `zlib.constants.Z_RLE`
* `zlib.constants.Z_FIXED`
* `zlib.constants.Z_DEFAULT_STRATEGY`

### Brotli 常量

<!-- YAML
added:
 - v11.7.0
 - v10.16.0
-->

有多个选项和其他常量可用于基于 Brotli 的流：

#### 刷新操作

以下值是适用于基于 Brotli 的流的有效刷新操作：

* `zlib.constants.BROTLI_OPERATION_PROCESS`（所有操作的默认值）
* `zlib.constants.BROTLI_OPERATION_FLUSH`（调用 `.flush()` 时的默认值）
* `zlib.constants.BROTLI_OPERATION_FINISH`（最后一块的默认值）
* `zlib.constants.BROTLI_OPERATION_EMIT_METADATA`
  * 这个特定的操作在 Node.js 上下文中可能难以使用，因为流层使得很难知道哪些数据会最终出现在此帧中。此外，目前没有方法通过 Node.js API 使用此数据。

#### 压缩器选项

有几个选项可以设置在 Brotli 编码器上，影响压缩效率和速度。键和值都可以作为 `zlib.constants` 对象的属性访问。

最重要的选项是：

* `BROTLI_PARAM_MODE`
  * `BROTLI_MODE_GENERIC`（默认）
  * `BROTLI_MODE_TEXT`，针对 UTF-8 文本调整
  * `BROTLI_MODE_FONT`，针对 WOFF 2.0 字体调整
* `BROTLI_PARAM_QUALITY`
  * 范围从 `BROTLI_MIN_QUALITY` 到 `BROTLI_MAX_QUALITY`，默认值为 `BROTLI_DEFAULT_QUALITY`。
* `BROTLI_PARAM_SIZE_HINT`
  * 表示预期输入大小的整数值；对于未知输入大小，默认为 `0`。

以下标志可用于对压缩算法和内存使用调优进行高级控制：

* `BROTLI_PARAM_LGWIN`
  * 范围从 `BROTLI_MIN_WINDOW_BITS` 到 `BROTLI_MAX_WINDOW_BITS`，默认值为 `BROTLI_DEFAULT_WINDOW`，或者如果设置了 `BROTLI_PARAM_LARGE_WINDOW` 标志，则最大为 `BROTLI_LARGE_MAX_WINDOW_BITS`。
* `BROTLI_PARAM_LGBLOCK`
  * 范围从 `BROTLI_MIN_INPUT_BLOCK_BITS` 到 `BROTLI_MAX_INPUT_BLOCK_BITS`。
* `BROTLI_PARAM_DISABLE_LITERAL_CONTEXT_MODELING`
  * 布尔标志，以解压缩速度为代价降低压缩率。
* `BROTLI_PARAM_LARGE_WINDOW`
  * 启用“大窗口 Brotli”模式的布尔标志（与 [RFC 7932][] 标准化的 Brotli 格式不兼容）。
* `BROTLI_PARAM_NPOSTFIX`
  * 范围从 `0` 到 `BROTLI_MAX_NPOSTFIX`。
* `BROTLI_PARAM_NDIRECT`
  * 范围从 `0` 到 `15 << NPOSTFIX`，步长为 `1 << NPOSTFIX`。

#### 解压缩器选项

这些高级选项可用于控制解压缩：

* `BROTLI_DECODER_PARAM_DISABLE_RING_BUFFER_REALLOCATION`
  * 影响内部内存分配模式的布尔标志。
* `BROTLI_DECODER_PARAM_LARGE_WINDOW`
  * 启用“大窗口 Brotli”模式的布尔标志（与 [RFC 7932][] 标准化的 Brotli 格式不兼容）。

### Zstd 常量

> Stability: 1 - Experimental

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

有多个选项和其他常量可用于基于 Zstd 的流：

#### 刷新操作

以下值是适用于基于 Zstd 的流的有效刷新操作：

* `zlib.constants.ZSTD_e_continue`（所有操作的默认值）
* `zlib.constants.ZSTD_e_flush`（调用 `.flush()` 时的默认值）
* `zlib.constants.ZSTD_e_end`（最后一块的默认值）

#### 压缩器选项

有几个选项可以设置在 Zstd 编码器上，影响压缩效率和速度。键和值都可以作为 `zlib.constants` 对象的属性访问。

最重要的选项是：

* `ZSTD_c_compressionLevel`
  * 根据预定义的 cLevel 表设置压缩参数。默认级别是 ZSTD\_CLEVEL\_DEFAULT==3。
* `ZSTD_c_strategy`
  * 选择压缩策略。
  * 可能的值在下面的策略选项部分列出。

#### 策略选项

以下常量可用作 `ZSTD_c_strategy` 参数的值：

* `zlib.constants.ZSTD_fast`
* `zlib.constants.ZSTD_dfast`
* `zlib.constants.ZSTD_greedy`
* `zlib.constants.ZSTD_lazy`
* `zlib.constants.ZSTD_lazy2`
* `zlib.constants.ZSTD_btlazy2`
* `zlib.constants.ZSTD_btopt`
* `zlib.constants.ZSTD_btultra`
* `zlib.constants.ZSTD_btultra2`

示例：

```js
const stream = zlib.createZstdCompress({
  params: {
    [zlib.constants.ZSTD_c_strategy]: zlib.constants.ZSTD_btultra,
  },
});
```

#### 承诺的源大小

可以通过 `opts.pledgedSrcSize` 指定未压缩输入的预期总大小。如果大小在输入结束时不匹配，压缩将失败并显示代码 `ZSTD_error_srcSize_wrong`。

#### 解压缩器选项

这些高级选项可用于控制解压缩：

* `ZSTD_d_windowLogMax`
  * 选择一个大小限制（以 2 的幂为单位），超过该限制，流式 API 将拒绝分配内存缓冲区，以保护主机免受不合理的内存需求。

## 类：`Options`

<!-- YAML
added: v0.11.1
changes:
  - version:
    - v14.5.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/33516
    description: The `maxOutputLength` option is supported now.
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `dictionary` option can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `dictionary` option can be an `Uint8Array` now.
  - version: v5.11.0
    pr-url: https://github.com/nodejs/node/pull/6069
    description: The `finishFlush` option is supported now.
-->

<!--type=misc-->

每个基于 zlib 的类都接受一个 `options` 对象。不需要任何选项。

某些选项仅在压缩时相关，并被解压缩类忽略。

* `flush` {integer} **默认值:** `zlib.constants.Z_NO_FLUSH`
* `finishFlush` {integer} **默认值:** `zlib.constants.Z_FINISH`
* `chunkSize` {integer} **默认值:** `16 * 1024`
* `windowBits` {integer}
* `level` {integer}（仅压缩）
* `memLevel` {integer}（仅压缩）
* `strategy` {integer}（仅压缩）
* `dictionary` {Buffer|TypedArray|DataView|ArrayBuffer}（仅 deflate/inflate，默认为空字典）
* `info` {boolean}（如果为 `true`，则返回一个包含 `buffer` 和 `engine` 的对象。）
* `maxOutputLength` {integer} 限制使用[便捷方法][]时的输出大小。**默认值:** [`buffer.kMaxLength`][]

有关更多信息，请参阅 [`deflateInit2` 和 `inflateInit2`][] 文档。

## 类：`BrotliOptions`

<!-- YAML
added: v11.7.0
changes:
  - version:
    - v14.5.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/33516
    description: The `maxOutputLength` option is supported now.
-->

<!--type=misc-->

每个基于 Brotli 的类都接受一个 `options` 对象。所有选项都是可选的。

* `flush` {integer} **默认值:** `zlib.constants.BROTLI_OPERATION_PROCESS`
* `finishFlush` {integer} **默认值:** `zlib.constants.BROTLI_OPERATION_FINISH`
* `chunkSize` {integer} **默认值:** `16 * 1024`
* `params` {Object} 包含索引化 [Brotli 参数][] 的键值对象。
* `maxOutputLength` {integer} 限制使用[便捷方法][]时的输出大小。**默认值:** [`buffer.kMaxLength`][]
* `info` {boolean} 如果为 `true`，则返回一个包含 `buffer` 和 `engine` 的对象。**默认值:** `false`

例如：

```js
const stream = zlib.createBrotliCompress({
  chunkSize: 32 * 1024,
  params: {
    [zlib.constants.BROTLI_PARAM_MODE]: zlib.constants.BROTLI_MODE_TEXT,
    [zlib.constants.BROTLI_PARAM_QUALITY]: 4,
    [zlib.constants.BROTLI_PARAM_SIZE_HINT]: fs.statSync(inputFile).size,
  },
});
```

## 类：`zlib.BrotliCompress`

<!-- YAML
added:
 - v11.7.0
 - v10.16.0
-->

* 扩展自: [`ZlibBase`][]

使用 Brotli 算法压缩数据。

## 类：`zlib.BrotliDecompress`

<!-- YAML
added:
 - v11.7.0
 - v10.16.0
-->

* 扩展自: [`ZlibBase`][]

使用 Brotli 算法解压缩数据。

## 类：`zlib.Deflate`

<!-- YAML
added: v0.5.8
-->

* 扩展自: [`ZlibBase`][]

使用 deflate 压缩数据。

## 类：`zlib.DeflateRaw`

<!-- YAML
added: v0.5.8
-->

* 扩展自: [`ZlibBase`][]

使用 deflate 压缩数据，并且不附加 `zlib` 头。

## 类：`zlib.Gunzip`

<!-- YAML
added: v0.5.8
changes:
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/5883
    description: Trailing garbage at the end of the input stream will now
                 result in an `'error'` event.
  - version: v5.9.0
    pr-url: https://github.com/nodejs/node/pull/5120
    description: Multiple concatenated gzip file members are supported now.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/2595
    description: A truncated input stream will now result in an `'error'` event.
-->

* 扩展自: [`ZlibBase`][]

解压缩 gzip 流。

## 类：`zlib.Gzip`

<!-- YAML
added: v0.5.8
-->

* 扩展自: [`ZlibBase`][]

使用 gzip 压缩数据。

## 类：`zlib.Inflate`

<!-- YAML
added: v0.5.8
changes:
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/2595
    description: A truncated input stream will now result in an `'error'` event.
-->

* 扩展自: [`ZlibBase`][]

解压缩 deflate 流。

## 类：`zlib.InflateRaw`

<!-- YAML
added: v0.5.8
changes:
  - version: v6.8.0
    pr-url: https://github.com/nodejs/node/pull/8512
    description: Custom dictionaries are now supported by `InflateRaw`.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/2595
    description: A truncated input stream will now result in an `'error'` event.
-->

* 扩展自: [`ZlibBase`][]

解压缩原始 deflate 流。

## 类：`zlib.Unzip`

<!-- YAML
added: v0.5.8
-->

* 扩展自: [`ZlibBase`][]

通过自动检测头来解压缩 Gzip 或 Deflate 压缩的流。

## 类：`zlib.ZlibBase`

<!-- YAML
added: v0.5.8
changes:
  - version:
     - v11.7.0
     - v10.16.0
    pr-url: https://github.com/nodejs/node/pull/24939
    description: This class was renamed from `Zlib` to `ZlibBase`.
-->

* 扩展自: [`stream.Transform`][]

不由 `node:zlib` 模块导出。此处记录它是因为它是压缩器/解压缩器类的基类。

此类继承自 [`stream.Transform`][]，允许 `node:zlib` 对象在管道和类似的流操作中使用。

### `zlib.bytesWritten`

<!-- YAML
added: v10.0.0
-->

* 类型: {number}

`zlib.bytesWritten` 属性指定在处理字节之前（根据派生类的不同，进行压缩或解压缩）写入引擎的字节数。

### `zlib.close([callback])`

<!-- YAML
added: v0.9.4
-->

* `callback` {Function}

关闭底层句柄。

### `zlib.flush([kind, ]callback)`

<!-- YAML
added: v0.5.8
-->

* `kind` **默认值:** 对于基于 zlib 的流为 `zlib.constants.Z_FULL_FLUSH`，对于基于 Brotli 的流为 `zlib.constants.BROTLI_OPERATION_FLUSH`。
* `callback` {Function}

刷新挂起的数据。不要轻率地调用此方法，过早的刷新会对压缩算法的有效性产生负面影响。

调用此方法仅刷新内部 `zlib` 状态的数据，并不在流级别执行任何类型的刷新。相反，它的行为类似于对 `.write()` 的正常调用，即它将排队在其他挂起的写入之后，并且仅在从流中读取数据时产生输出。

### `zlib.params(level, strategy, callback)`

<!-- YAML
added: v0.11.4
-->

* `level` {integer}
* `strategy` {integer}
* `callback` {Function}

此函数仅适用于基于 zlib 的流，即不适用于 Brotli。

动态更新压缩级别和压缩策略。仅适用于 deflate 算法。

### `zlib.reset()`

<!-- YAML
added: v0.7.0
-->

将压缩器/解压缩器重置为工厂默认值。仅适用于 inflate 和 deflate 算法。

## 类：`ZstdOptions`

> Stability: 1 - Experimental

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

<!--type=misc-->

每个基于 Zstd 的类都接受一个 `options` 对象。所有选项都是可选的。

* `flush` {integer} **默认值:** `zlib.constants.ZSTD_e_continue`
* `finishFlush` {integer} **默认值:** `zlib.constants.ZSTD_e_end`
* `chunkSize` {integer} **默认值:** `16 * 1024`
* `params` {Object} 包含索引化 [Zstd 参数][] 的键值对象。
* `maxOutputLength` {integer} 限制使用[便捷方法][]时的输出大小。**默认值:** [`buffer.kMaxLength`][]
* `info` {boolean} 如果为 `true`，则返回一个包含 `buffer` 和 `engine` 的对象。**默认值:** `false`
* `dictionary` {Buffer} 用于在压缩或解压缩与字典共享常见模式的数据时提高压缩效率的可选字典。

例如：

```js
const stream = zlib.createZstdCompress({
  chunkSize: 32 * 1024,
  params: {
    [zlib.constants.ZSTD_c_compressionLevel]: 10,
    [zlib.constants.ZSTD_c_checksumFlag]: 1,
  },
});
```

## 类：`zlib.ZstdCompress`

> Stability: 1 - Experimental

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

使用 Zstd 算法压缩数据。

## 类：`zlib.ZstdDecompress`

> Stability: 1 - Experimental

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

使用 Zstd 算法解压缩数据。

## `zlib.constants`

<!-- YAML
added: v7.0.0
-->

提供一个枚举 Zlib 相关常量的对象。

## `zlib.crc32(data[, value])`

<!-- YAML
added:
  - v22.2.0
  - v20.15.0
-->

* `data` {string|Buffer|TypedArray|DataView} 当 `data` 是字符串时，它将在用于计算之前被编码为 UTF-8。
* `value` {integer} 可选的起始值。它必须是一个 32 位无符号整数。**默认值:** `0`
* 返回: {integer} 包含校验和的 32 位无符号整数。

计算 `data` 的 32 位[循环冗余校验][]校验和。如果指定了 `value`，则将其用作校验和的起始值，否则使用 0 作为起始值。

CRC 算法旨在计算校验和并检测数据传输中的错误。它不适用于加密身份验证。

为了与其他 API 保持一致，如果 `data` 是字符串，它将在用于计算之前使用 UTF-8 编码。如果用户仅使用 Node.js 来计算和匹配校验和，这与默认使用 UTF-8 编码的其他 API 配合良好。

一些第三方 JavaScript 库基于 `str.charCodeAt()` 计算字符串的校验和，以便它可以在浏览器中运行。如果用户想要匹配在浏览器中使用此类库计算的校验和，最好在 Node.js 中也使用相同的库（如果它也在 Node.js 中运行）。如果用户必须使用 `zlib.crc32()` 来匹配第三方库产生的校验和：

1. 如果该库接受 `Uint8Array` 作为输入，请在浏览器中使用 `TextEncoder` 将字符串编码为具有 UTF-8 编码的 `Uint8Array`，并在浏览器中基于 UTF-8 编码的字符串计算校验和。
2. 如果该库仅接受字符串并基于 `str.charCodeAt()` 计算数据，则在 Node.js 端，使用 `Buffer.from(str, 'utf16le')` 将字符串转换为缓冲区。

```mjs
import zlib from 'node:zlib';
import { Buffer } from 'node:buffer';

let crc = zlib.crc32('hello');  // 907060870
crc = zlib.crc32('world', crc);  // 4192936109

crc = zlib.crc32(Buffer.from('hello', 'utf16le'));  // 1427272415
crc = zlib.crc32(Buffer.from('world', 'utf16le'), crc);  // 4150509955
```

```cjs
const zlib = require('node:zlib');
const { Buffer } = require('node:buffer');

let crc = zlib.crc32('hello');  // 907060870
crc = zlib.crc32('world', crc);  // 4192936109

crc = zlib.crc32(Buffer.from('hello', 'utf16le'));  // 1427272415
crc = zlib.crc32(Buffer.from('world', 'utf16le'), crc);  // 4150509955
```

## `zlib.createBrotliCompress([options])`

<!-- YAML
added:
 - v11.7.0
 - v10.16.0
-->

* `options` {brotli options}

创建并返回一个新的 [`BrotliCompress`][] 对象。

## `zlib.createBrotliDecompress([options])`

<!-- YAML
added:
 - v11.7.0
 - v10.16.0
-->

* `options` {brotli options}

创建并返回一个新的 [`BrotliDecompress`][] 对象。

## `zlib.createDeflate([options])`

<!-- YAML
added: v0.5.8
-->

* `options` {zlib options}

创建并返回一个新的 [`Deflate`][] 对象。

## `zlib.createDeflateRaw([options])`

<!-- YAML
added: v0.5.8
-->

* `options` {zlib options}

创建并返回一个新的 [`DeflateRaw`][] 对象。

将 zlib 从 1.2.8 升级到 1.2.11 改变了原始 deflate 流在 `windowBits` 设置为 8 时的行为。如果最初设置为 8，zlib 会自动将 `windowBits` 设置为 9。新版本的 zlib 会抛出异常，因此 Node.js 恢复了将值 8 升级为 9 的原始行为，因为将 `windowBits = 9` 传递给 zlib 实际上会导致压缩流仅有效使用 8 位窗口。

## `zlib.createGunzip([options])`

<!-- YAML
added: v0.5.8
-->

* `options` {zlib options}

创建并返回一个新的 [`Gunzip`][] 对象。

## `zlib.createGzip([options])`

<!-- YAML
added: v0.5.8
-->

* `options` {zlib options}

创建并返回一个新的 [`Gzip`][] 对象。
参见[示例][zlib.createGzip 示例]。

## `zlib.createInflate([options])`

<!-- YAML
added: v0.5.8
-->

* `options` {zlib options}

创建并返回一个新的 [`Inflate`][] 对象。

## `zlib.createInflateRaw([options])`

<!-- YAML
added: v0.5.8
-->

* `options` {zlib options}

创建并返回一个新的 [`InflateRaw`][] 对象。

## `zlib.createUnzip([options])`

<!-- YAML
added: v0.5.8
-->

* `options` {zlib options}

创建并返回一个新的 [`Unzip`][] 对象。

## `zlib.createZstdCompress([options])`

> Stability: 1 - Experimental

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

* `options` {zstd options}

创建并返回一个新的 [`ZstdCompress`][] 对象。

## `zlib.createZstdDecompress([options])`

> Stability: 1 - Experimental

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

* `options` {zstd options}

创建并返回一个新的 [`ZstdDecompress`][] 对象。

## 便捷方法

<!--type=misc-->

所有这些方法都将 {Buffer}、{TypedArray}、{DataView}、{ArrayBuffer} 或字符串作为第一个参数，可选的第二个参数用于向 `zlib` 类提供选项，并将使用 `callback(error, result)` 调用提供的回调。

每个方法都有一个对应的 `*Sync` 方法，它接受相同的参数，但没有回调。

### `zlib.brotliCompress(buffer[, options], callback)`

<!-- YAML
added:
 - v11.7.0
 - v10.16.0
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {brotli options}
* `callback` {Function}

### `zlib.brotliCompressSync(buffer[, options])`

<!-- YAML
added:
 - v11.7.0
 - v10.16.0
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {brotli options}

使用 [`BrotliCompress`][] 压缩一块数据。

### `zlib.brotliDecompress(buffer[, options], callback)`

<!-- YAML
added:
 - v11.7.0
 - v10.16.0
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {brotli options}
* `callback` {Function}

### `zlib.brotliDecompressSync(buffer[, options])`

<!-- YAML
added:
 - v11.7.0
 - v10.16.0
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {brotli options}

使用 [`BrotliDecompress`][] 解压缩一块数据。

### `zlib.deflate(buffer[, options], callback)`

<!-- YAML
added: v0.6.0
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}
* `callback` {Function}

### `zlib.deflateSync(buffer[, options])`

<!-- YAML
added: v0.11.12
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}

使用 [`Deflate`][] 压缩一块数据。

### `zlib.deflateRaw(buffer[, options], callback)`

<!-- YAML
added: v0.6.0
changes:
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}
* `callback` {Function}

### `zlib.deflateRawSync(buffer[, options])`

<!-- YAML
added: v0.11.12
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}

使用 [`DeflateRaw`][] 压缩一块数据。

### `zlib.gunzip(buffer[, options], callback)`

<!-- YAML
added: v0.6.0
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}
* `callback` {Function}

### `zlib.gunzipSync(buffer[, options])`

<!-- YAML
added: v0.11.12
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}

使用 [`Gunzip`][] 解压缩一块数据。

### `zlib.gzip(buffer[, options], callback)`

<!-- YAML
added: v0.6.0
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}
* `callback` {Function}

### `zlib.gzipSync(buffer[, options])`

<!-- YAML
added: v0.11.12
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}

使用 [`Gzip`][] 压缩一块数据。

### `zlib.inflate(buffer[, options], callback)`

<!-- YAML
added: v0.6.0
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}
* `callback` {Function}

### `zlib.inflateSync(buffer[, options])`

<!-- YAML
added: v0.11.12
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}

使用 [`Inflate`][] 解压缩一块数据。

### `zlib.inflateRaw(buffer[, options], callback)`

<!-- YAML
added: v0.6.0
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}
* `callback` {Function}

### `zlib.inflateRawSync(buffer[, options])`

<!-- YAML
added: v0.11.12
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}

使用 [`InflateRaw`][] 解压缩一块数据。

### `zlib.unzip(buffer[, options], callback)`

<!-- YAML
added: v0.6.0
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}
* `callback` {Function}

### `zlib.unzipSync(buffer[, options])`

<!-- YAML
added: v0.11.12
changes:
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/16042
    description: The `buffer` parameter can be an `ArrayBuffer`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12223
    description: The `buffer` parameter can be any `TypedArray` or `DataView`.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12001
    description: The `buffer` parameter can be an `Uint8Array` now.
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zlib options}

使用 [`Unzip`][] 解压缩一块数据。

### `zlib.zstdCompress(buffer[, options], callback)`

> Stability: 1 - Experimental

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zstd options}
* `callback` {Function}

### `zlib.zstdCompressSync(buffer[, options])`

> Stability: 1 - Experimental

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zstd options}

使用 [`ZstdCompress`][] 压缩一块数据。

### `zlib.zstdDecompress(buffer[, options], callback)`

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zstd options}
* `callback` {Function}

### `zlib.zstdDecompressSync(buffer[, options])`

> Stability: 1 - Experimental

<!-- YAML
added:
  - v23.8.0
  - v22.15.0
-->

* `buffer` {Buffer|TypedArray|DataView|ArrayBuffer|string}
* `options` {zstd options}

使用 [`ZstdDecompress`][] 解压缩一块数据。

[Brotli parameters]: #brotli-constants
[Cyclic redundancy check]: https://en.wikipedia.org/wiki/Cyclic_redundancy_check
[Memory usage tuning]: #memory-usage-tuning
[RFC 7932]: https://www.rfc-editor.org/rfc/rfc7932.txt
[Streams API]: stream.md
[Zstd parameters]: #zstd-constants
[`.flush()`]: #zlibflushkind-callback
[`Accept-Encoding`]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.3
[`BrotliCompress`]: #class-zlibbrotlicompress
[`BrotliDecompress`]: #class-zlibbrotlidecompress
[`Content-Encoding`]: https://www.w3.org/Protocols/rfc2616/rfc2616-sec14.html#sec14.11
[`DeflateRaw`]: #class-zlibdeflateraw
[`Deflate`]: #class-zlibdeflate
[`Gunzip`]: #class-zlibgunzip
[`Gzip`]: #class-zlibgzip
[`InflateRaw`]: #class-zlibinflateraw
[`Inflate`]: #class-zlibinflate
[`Unzip`]: #class-zlibunzip
[`ZlibBase`]: #class-zlibzlibbase
[`ZstdCompress`]: #class-zlibzstdcompress
[`ZstdDecompress`]: #class-zlibzstddecompress
[`buffer.kMaxLength`]: buffer.md#bufferkmaxlength
[`deflateInit2` and `inflateInit2`]: https://zlib.net/manual.html#Advanced
[`stream.Transform`]: stream.md#class-streamtransform
[convenience methods]: #convenience-methods
[zlib documentation]: https://zlib.net/manual.html#Constants
[zlib.createGzip example]: #zlib
