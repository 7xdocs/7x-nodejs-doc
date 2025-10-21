[文件名称]: http2.md
[文件内容开始]
# HTTP/2

<!-- YAML
added: v8.4.0
changes:
  - version:
      - v15.3.0
      - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/36070
    description: It is possible to abort a request with an AbortSignal.
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/34664
    description: Requests with the `host` header (with or without
                 `:authority`) can now be sent/received.
  - version: v10.10.0
    pr-url: https://github.com/nodejs/node/pull/22466
    description: HTTP/2 is now Stable. Previously, it had been Experimental.
-->

<!--introduced_in=v8.4.0-->

> Stability: 2 - Stable

<!-- source_link=lib/http2.js -->

`node:http2` 模块提供了 [HTTP/2][] 协议的实现。
可以通过以下方式访问：

```js
const http2 = require('node:http2');
```

## 确定加密支持是否不可用

Node.js 有可能在构建时未包含对 `node:crypto` 模块的支持。在这种情况下，尝试从 `node:http2` `import` 或调用 `require('node:http2')` 将会抛出错误。

使用 CommonJS 时，可以使用 try/catch 捕获抛出的错误：

```cjs
let http2;
try {
  http2 = require('node:http2');
} catch (err) {
  console.error('http2 support is disabled!');
}
```

使用词法 ESM `import` 关键字时，只有在任何尝试加载模块的尝试*之前*（例如，使用预加载模块）注册了 `process.on('uncaughtException')` 的处理程序时，才能捕获错误。

使用 ESM 时，如果代码可能在不启用加密支持的 Node.js 构建版本上运行，请考虑使用 [`import()`][] 函数而不是词法 `import` 关键字：

```mjs
let http2;
try {
  http2 = await import('node:http2');
} catch (err) {
  console.error('http2 support is disabled!');
}
```

## 核心 API

核心 API 提供了一个低级接口，专门围绕支持 HTTP/2 协议特性设计。它特意*不*设计为与现有的 [HTTP/1][] 模块 API 兼容。但是，[兼容性 API][] 是为此设计的。

`http2` 核心 API 在客户端和服务器之间比 `http` API 更加对称。例如，大多数事件，如 `'error'`、`'connect'` 和 `'stream'`，可以由客户端代码或服务器端代码发出。

### 服务器端示例

以下演示了使用核心 API 的简单 HTTP/2 服务器。由于没有已知的浏览器支持[未加密的 HTTP/2][HTTP/2 Unencrypted]，因此与浏览器客户端通信时，需要使用 [`http2.createSecureServer()`][]。

```mjs
import { createSecureServer } from 'node:http2';
import { readFileSync } from 'node:fs';

const server = createSecureServer({
  key: readFileSync('localhost-privkey.pem'),
  cert: readFileSync('localhost-cert.pem'),
});

server.on('error', (err) => console.error(err));

server.on('stream', (stream, headers) => {
  // stream 是一个 Duplex
  stream.respond({
    'content-type': 'text/html; charset=utf-8',
    ':status': 200,
  });
  stream.end('<h1>Hello World</h1>');
});

server.listen(8443);
```

```cjs
const http2 = require('node:http2');
const fs = require('node:fs');

const server = http2.createSecureServer({
  key: fs.readFileSync('localhost-privkey.pem'),
  cert: fs.readFileSync('localhost-cert.pem'),
});
server.on('error', (err) => console.error(err));

server.on('stream', (stream, headers) => {
  // stream 是一个 Duplex
  stream.respond({
    'content-type': 'text/html; charset=utf-8',
    ':status': 200,
  });
  stream.end('<h1>Hello World</h1>');
});

server.listen(8443);
```

要生成此示例的证书和密钥，请运行：

```bash
openssl req -x509 -newkey rsa:2048 -nodes -sha256 -subj '/CN=localhost' \
  -keyout localhost-privkey.pem -out localhost-cert.pem
```

### 客户端示例

以下演示了一个 HTTP/2 客户端：

```mjs
import { connect } from 'node:http2';
import { readFileSync } from 'node:fs';

const client = connect('https://localhost:8443', {
  ca: readFileSync('localhost-cert.pem'),
});
client.on('error', (err) => console.error(err));

const req = client.request({ ':path': '/' });

req.on('response', (headers, flags) => {
  for (const name in headers) {
    console.log(`${name}: ${headers[name]}`);
  }
});

req.setEncoding('utf8');
let data = '';
req.on('data', (chunk) => { data += chunk; });
req.on('end', () => {
  console.log(`\n${data}`);
  client.close();
});
req.end();
```

```cjs
const http2 = require('node:http2');
const fs = require('node:fs');

const client = http2.connect('https://localhost:8443', {
  ca: fs.readFileSync('localhost-cert.pem'),
});
client.on('error', (err) => console.error(err));

const req = client.request({ ':path': '/' });

req.on('response', (headers, flags) => {
  for (const name in headers) {
    console.log(`${name}: ${headers[name]}`);
  }
});

req.setEncoding('utf8');
let data = '';
req.on('data', (chunk) => { data += chunk; });
req.on('end', () => {
  console.log(`\n${data}`);
  client.close();
});
req.end();
```

### 类：`Http2Session`

<!-- YAML
added: v8.4.0
-->

* 扩展：{EventEmitter}

`http2.Http2Session` 类的实例表示 HTTP/2 客户端和服务器之间的活动通信会话。此类的实例*不*打算由用户代码直接构造。

每个 `Http2Session` 实例的行为会根据其是作为服务器还是客户端运行而略有不同。`http2session.type` 属性可用于确定 `Http2Session` 的运行模式。在服务器端，用户代码很少有机会直接使用 `Http2Session` 对象，大多数操作通常通过与 `Http2Server` 或 `Http2Stream` 对象的交互来进行。

用户代码不会直接创建 `Http2Session` 实例。服务器端的 `Http2Session` 实例是在接收到新的 HTTP/2 连接时由 `Http2Server` 实例创建的。客户端的 `Http2Session` 实例是使用 `http2.connect()` 方法创建的。

#### `Http2Session` 和套接字

每个 `Http2Session` 实例在创建时都与恰好一个 [`net.Socket`][] 或 [`tls.TLSSocket`][] 相关联。当 `Socket` 或 `Http2Session` 被销毁时，两者都将被销毁。

由于 HTTP/2 协议施加的特定序列化和处理要求，不建议用户代码从绑定到 `Http2Session` 的 `Socket` 实例读取数据或向其写入数据。这样做可能会使 HTTP/2 会话进入不确定状态，导致会话和套接字变得不可用。

一旦 `Socket` 绑定到 `Http2Session`，用户代码应完全依赖 `Http2Session` 的 API。

#### 事件：`'close'`

<!-- YAML
added: v8.4.0
-->

一旦 `Http2Session` 被销毁，就会发出 `'close'` 事件。其监听器不需要任何参数。

#### 事件：`'connect'`

<!-- YAML
added: v8.4.0
-->

* `session` {Http2Session}
* `socket` {net.Socket}

一旦 `Http2Session` 成功连接到远程对等端并且通信可以开始时，就会发出 `'connect'` 事件。

用户代码通常不会直接监听此事件。

#### 事件：`'error'`

<!-- YAML
added: v8.4.0
-->

* `error` {Error}

在处理 `Http2Session` 期间发生错误时，会发出 `'error'` 事件。

#### 事件：`'frameError'`

<!-- YAML
added: v8.4.0
-->

* `type` {integer} 帧类型。
* `code` {integer} 错误代码。
* `id` {integer} 流 ID（如果帧不与流关联，则为 `0`）。

当尝试在会话上发送帧时发生错误时，会发出 `'frameError'` 事件。如果无法发送的帧与特定的 `Http2Stream` 关联，则会尝试在 `Http2Stream` 上发出 `'frameError'` 事件。

如果 `'frameError'` 事件与流关联，则该流将在 `'frameError'` 事件之后立即关闭并销毁。如果事件不与流关联，则 `Http2Session` 将在 `'frameError'` 事件之后立即关闭。

#### 事件：`'goaway'`

<!-- YAML
added: v8.4.0
-->

* `errorCode` {number} `GOAWAY` 帧中指定的 HTTP/2 错误代码。
* `lastStreamID` {number} 远程对等端成功处理的最后一个流的 ID（如果未指定 ID，则为 `0`）。
* `opaqueData` {Buffer} 如果 `GOAWAY` 帧中包含额外的不透明数据，将传递一个包含该数据的 `Buffer` 实例。

当接收到 `GOAWAY` 帧时，会发出 `'goaway'` 事件。

当发出 `'goaway'` 事件时，`Http2Session` 实例将自动关闭。

#### 事件：`'localSettings'`

<!-- YAML
added: v8.4.0
-->

* `settings` {HTTP/2 Settings Object} 接收到的 `SETTINGS` 帧的副本。

当收到确认 `SETTINGS` 帧时，会发出 `'localSettings'` 事件。

当使用 `http2session.settings()` 提交新设置时，修改后的设置直到发出 `'localSettings'` 事件才会生效。

```js
session.settings({ enablePush: false });

session.on('localSettings', (settings) => {
  /* 使用新设置 */
});
```

#### 事件：`'ping'`

<!-- YAML
added: v10.12.0
-->

* `payload` {Buffer} `PING` 帧的 8 字节载荷

每当从连接的对等端接收到 `PING` 帧时，就会发出 `'ping'` 事件。

#### 事件：`'remoteSettings'`

<!-- YAML
added: v8.4.0
-->

* `settings` {HTTP/2 Settings Object} 接收到的 `SETTINGS` 帧的副本。

当从连接的对等端接收到新的 `SETTINGS` 帧时，会发出 `'remoteSettings'` 事件。

```js
session.on('remoteSettings', (settings) => {
  /* 使用新设置 */
});
```

#### 事件：`'stream'`

<!-- YAML
added: v8.4.0
-->

* `stream` {Http2Stream} 对流的引用
* `headers` {HTTP/2 Headers Object} 描述头部的对象
* `flags` {number} 相关的数字标志
* `rawHeaders` {HTTP/2 Raw Headers} 包含原始头部的数组

当创建新的 `Http2Stream` 时，会发出 `'stream'` 事件。

```js
session.on('stream', (stream, headers, flags) => {
  const method = headers[':method'];
  const path = headers[':path'];
  // ...
  stream.respond({
    ':status': 200,
    'content-type': 'text/plain; charset=utf-8',
  });
  stream.write('hello ');
  stream.end('world');
});
```

在服务器端，用户代码通常不会直接监听此事件，而是会为由 `http2.createServer()` 和 `http2.createSecureServer()` 分别返回的 `net.Server` 或 `tls.Server` 实例发出的 `'stream'` 事件注册处理程序，如下例所示：

```mjs
import { createServer } from 'node:http2';

// 创建一个未加密的 HTTP/2 服务器
const server = createServer();

server.on('stream', (stream, headers) => {
  stream.respond({
    'content-type': 'text/html; charset=utf-8',
    ':status': 200,
  });
  stream.on('error', (error) => console.error(error));
  stream.end('<h1>Hello World</h1>');
});

server.listen(8000);
```

```cjs
const http2 = require('node:http2');

// 创建一个未加密的 HTTP/2 服务器
const server = http2.createServer();

server.on('stream', (stream, headers) => {
  stream.respond({
    'content-type': 'text/html; charset=utf-8',
    ':status': 200,
  });
  stream.on('error', (error) => console.error(error));
  stream.end('<h1>Hello World</h1>');
});

server.listen(8000);
```

尽管 HTTP/2 流和网络套接字之间不是一一对应的关系，但网络错误将销毁每个单独的流，并且必须在流级别处理，如上所示。

#### 事件：`'timeout'`

<!-- YAML
added: v8.4.0
-->

在使用 `http2session.setTimeout()` 方法设置此 `Http2Session` 的超时时间后，如果在配置的毫秒数后 `Http2Session` 上没有活动，则会发出 `'timeout'` 事件。其监听器不需要任何参数。

```js
session.setTimeout(2000);
session.on('timeout', () => { /* .. */ });
```

#### `http2session.alpnProtocol`

<!-- YAML
added: v9.4.0
-->

* 类型：{string|undefined}

如果 `Http2Session` 尚未连接到套接字，则值为 `undefined`；如果 `Http2Session` 未连接到 `TLSSocket`，则为 `h2c`；否则将返回连接的 `TLSSocket` 自身的 `alpnProtocol` 属性值。

#### `http2session.close([callback])`

<!-- YAML
added: v9.4.0
-->

* `callback` {Function}

优雅地关闭 `Http2Session`，允许任何现有流自行完成，并防止创建新的 `Http2Stream` 实例。一旦关闭，如果没有打开的 `Http2Stream` 实例，*可能*会调用 `http2session.destroy()`。

如果指定了 `callback` 函数，它将被注册为 `'close'` 事件的处理程序。

#### `http2session.closed`

<!-- YAML
added: v9.4.0
-->

* 类型：{boolean}

如果此 `Http2Session` 实例已关闭，则为 `true`，否则为 `false`。

#### `http2session.connecting`

<!-- YAML
added: v10.0.0
-->

* 类型：{boolean}

如果此 `Http2Session` 实例仍在连接中，则为 `true`，将在发出 `connect` 事件和/或调用 `http2.connect` 回调之前设置为 `false`。

#### `http2session.destroy([error][, code])`

<!-- YAML
added: v8.4.0
-->

* `error` {Error} 如果 `Http2Session` 由于错误而被销毁，则为一个 `Error` 对象。
* `code` {number} 在最终的 `GOAWAY` 帧中发送的 HTTP/2 错误代码。如果未指定，且 `error` 不是 undefined，则默认为 `INTERNAL_ERROR`，否则默认为 `NO_ERROR`。

立即终止 `Http2Session` 和关联的 `net.Socket` 或 `tls.TLSSocket`。

一旦销毁，`Http2Session` 将发出 `'close'` 事件。如果 `error` 不是 undefined，则会在 `'close'` 事件之前立即发出 `'error'` 事件。

如果还有任何与 `Http2Session` 关联的剩余打开的 `Http2Stream`，这些也将被销毁。

#### `http2session.destroyed`

<!-- YAML
added: v8.4.0
-->

* 类型：{boolean}

如果此 `Http2Session` 实例已被销毁且不得再使用，则为 `true`，否则为 `false`。

#### `http2session.encrypted`

<!-- YAML
added: v9.4.0
-->

* 类型：{boolean|undefined}

如果 `Http2Session` 会话套接字尚未连接，则值为 `undefined`；如果 `Http2Session` 与 `TLSSocket` 连接，则为 `true`；如果 `Http2Session` 连接到任何其他类型的套接字或流，则为 `false`。

#### `http2session.goaway([code[, lastStreamID[, opaqueData]]])`

<!-- YAML
added: v9.4.0
-->

* `code` {number} 一个 HTTP/2 错误代码
* `lastStreamID` {number} 最后一个处理的 `Http2Stream` 的数字 ID
* `opaqueData` {Buffer|TypedArray|DataView} 一个包含要在 `GOAWAY` 帧中携带的额外数据的 `TypedArray` 或 `DataView` 实例。

向连接的对等端传输一个 `GOAWAY` 帧，但*不*关闭 `Http2Session`。

#### `http2session.localSettings`

<!-- YAML
added: v8.4.0
-->

* 类型：{HTTP/2 Settings Object}

一个描述此 `Http2Session` 当前本地设置的无原型对象。本地设置是*此* `Http2Session` 实例本地的。

#### `http2session.originSet`

<!-- YAML
added: v9.4.0
-->

* 类型：{string\[]|undefined}

如果 `Http2Session` 连接到 `TLSSocket`，则 `originSet` 属性将返回一个 `Array`，其中包含 `Http2Session` 可能被视为权威的来源。

`originSet` 属性仅在使用安全 TLS 连接时可用。

#### `http2session.pendingSettingsAck`

<!-- YAML
added: v8.4.0
-->

* 类型：{boolean}

指示 `Http2Session` 当前是否正在等待已发送 `SETTINGS` 帧的确认。在调用 `http2session.settings()` 方法后将为 `true`。一旦所有已发送的 `SETTINGS` 帧都被确认，将为 `false`。

#### `http2session.ping([payload, ]callback)`

<!-- YAML
added: v8.9.3
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `payload` {Buffer|TypedArray|DataView} 可选的 ping 载荷。
* `callback` {Function}
* 返回：{boolean}

向连接的 HTTP/2 对等端发送一个 `PING` 帧。必须提供一个 `callback` 函数。如果 `PING` 已发送，该方法将返回 `true`，否则返回 `false`。

未完成（未确认）的 ping 的最大数量由 `maxOutstandingPings` 配置选项决定。默认最大值为 10。

如果提供了 `payload`，它必须是一个包含 8 字节数据的 `Buffer`、`TypedArray` 或 `DataView`，这些数据将与 `PING` 一起传输，并随 ping 确认返回。

回调将以三个参数调用：一个错误参数，如果 `PING` 成功确认则为 `null`；一个 `duration` 参数，报告自 ping 发送和确认接收以来经过的毫秒数；以及一个包含 8 字节 `PING` 载荷的 `Buffer`。

```js
session.ping(Buffer.from('abcdefgh'), (err, duration, payload) => {
  if (!err) {
    console.log(`Ping acknowledged in ${duration} milliseconds`);
    console.log(`With payload '${payload.toString()}'`);
  }
});
```

如果未指定 `payload` 参数，则默认载荷将是标记 `PING` 持续时间开始的 64 位时间戳（小端序）。

#### `http2session.ref()`

<!-- YAML
added: v9.4.0
-->

在此 `Http2Session` 实例的底层 [`net.Socket`][] 上调用 [`ref()`][`net.Socket.prototype.ref()`]。

#### `http2session.remoteSettings`

<!-- YAML
added: v8.4.0
-->

* 类型：{HTTP/2 Settings Object}

一个描述此 `Http2Session` 当前远程设置的无原型对象。远程设置由*连接的* HTTP/2 对等端设置。

#### `http2session.setLocalWindowSize(windowSize)`

<!-- YAML
added:
  - v15.3.0
  - v14.18.0
-->

* `windowSize` {number}

设置本地端点的窗口大小。
`windowSize` 是要设置的总窗口大小，而不是增量。

```mjs
import { createServer } from 'node:http2';

const server = createServer();
const expectedWindowSize = 2 ** 20;
server.on('session', (session) => {

  // 设置本地窗口大小为 2 ** 20
  session.setLocalWindowSize(expectedWindowSize);
});
```

```cjs
const http2 = require('node:http2');

const server = http2.createServer();
const expectedWindowSize = 2 ** 20;
server.on('session', (session) => {

  // 设置本地窗口大小为 2 ** 20
  session.setLocalWindowSize(expectedWindowSize);
});
```

对于 http2 客户端，正确的事件是 `'connect'` 或 `'remoteSettings'`。

#### `http2session.setTimeout(msecs, callback)`

<!-- YAML
added: v8.4.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `msecs` {number}
* `callback` {Function}

用于设置一个回调函数，当在 `msecs` 毫秒后 `Http2Session` 上没有活动时调用。给定的 `callback` 被注册为 `'timeout'` 事件的监听器。

#### `http2session.socket`

<!-- YAML
added: v8.4.0
-->

* 类型：{net.Socket|tls.TLSSocket}

返回一个 `Proxy` 对象，其行为类似于 `net.Socket`（或 `tls.TLSSocket`），但将可用的方法限制为与 HTTP/2 一起使用安全的方法。

`destroy`、`emit`、`end`、`pause`、`read`、`resume` 和 `write` 将抛出一个错误代码为 `ERR_HTTP2_NO_SOCKET_MANIPULATION` 的错误。有关更多信息，请参阅 [`Http2Session` 和套接字][]。

`setTimeout` 方法将在此 `Http2Session` 上调用。

所有其他交互将直接路由到套接字。

#### `http2session.state`

<!-- YAML
added: v8.4.0
-->

提供有关 `Http2Session` 当前状态的各种信息。

* 类型：{Object}
  * `effectiveLocalWindowSize` {number} `Http2Session` 的当前本地（接收）流控制窗口大小。
  * `effectiveRecvDataLength` {number} 自上次流控制 `WINDOW_UPDATE` 以来已接收的当前字节数。
  * `nextStreamID` {number} 下次由此 `Http2Session` 创建新 `Http2Stream` 时要使用的数字标识符。
  * `localWindowSize` {number} 远程对等端可以在不接收 `WINDOW_UPDATE` 的情况下发送的字节数。
  * `lastProcStreamID` {number} 最近接收到 `HEADERS` 或 `DATA` 帧的 `Http2Stream` 的数字 id。
  * `remoteWindowSize` {number} 此 `Http2Session` 可以在不接收 `WINDOW_UPDATE` 的情况下发送的字节数。
  * `outboundQueueSize` {number} 当前在此 `Http2Session` 的出站队列中的帧数。
  * `deflateDynamicTableSize` {number} 出站头部压缩状态表的当前大小（以字节为单位）。
  * `inflateDynamicTableSize` {number} 入站头部压缩状态表的当前大小（以字节为单位）。

一个描述此 `Http2Session` 当前状态的对象。

#### `http2session.settings([settings][, callback])`

<!-- YAML
added: v8.4.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `settings` {HTTP/2 Settings Object}
* `callback` {Function} 一旦会话连接或如果会话已连接则立即调用的回调。
  * `err` {Error|null}
  * `settings` {HTTP/2 Settings Object} 更新后的 `settings` 对象。
  * `duration` {integer}

更新此 `Http2Session` 的当前本地设置，并向连接的 HTTP/2 对等端发送一个新的 `SETTINGS` 帧。

一旦调用，当会话等待远程对等端确认新设置时，`http2session.pendingSettingsAck` 属性将为 `true`。

新设置直到收到 `SETTINGS` 确认并发出 `'localSettings'` 事件后才会生效。在确认仍处于挂起状态时，可以发送多个 `SETTINGS` 帧。

#### `http2session.type`

<!-- YAML
added: v8.4.0
-->

* 类型：{number}

如果此 `Http2Session` 实例是服务器，则 `http2session.type` 将等于 `http2.constants.NGHTTP2_SESSION_SERVER`，如果实例是客户端，则等于 `http2.constants.NGHTTP2_SESSION_CLIENT`。

#### `http2session.unref()`

<!-- YAML
added: v9.4.0
-->

在此 `Http2Session` 实例的底层 [`net.Socket`][] 上调用 [`unref()`][`net.Socket.prototype.unref()`]。

### 类：`ServerHttp2Session`

<!-- YAML
added: v8.4.0
-->

* 扩展：{Http2Session}

#### `serverhttp2session.altsvc(alt, originOrStream)`

<!-- YAML
added: v9.4.0
-->

* `alt` {string} 由 [RFC 7838][] 定义的替代服务配置的描述。
* `originOrStream` {number|string|URL|Object} 要么是指定来源的 URL 字符串（或具有 `origin` 属性的 `Object`），要么是由 `http2stream.id` 属性给出的活动 `Http2Stream` 的数字标识符。

向连接的客户端提交一个 `ALTSVC` 帧（由 [RFC 7838][] 定义）。

```mjs
import { createServer } from 'node:http2';

const server = createServer();
server.on('session', (session) => {
  // 为来源 https://example.org:80 设置 altsvc
  session.altsvc('h2=":8000"', 'https://example.org:80');
});

server.on('stream', (stream) => {
  // 为特定流设置 altsvc
  stream.session.altsvc('h2=":8000"', stream.id);
});
```

```cjs
const http2 = require('node:http2');

const server = http2.createServer();
server.on('session', (session) => {
  // 为来源 https://example.org:80 设置 altsvc
  session.altsvc('h2=":8000"', 'https://example.org:80');
});

server.on('stream', (stream) => {
  // 为特定流设置 altsvc
  stream.session.altsvc('h2=":8000"', stream.id);
});
```

发送具有特定流 ID 的 `ALTSVC` 帧表示替代服务与给定 `Http2Stream` 的来源相关联。

`alt` 和来源字符串*必须*仅包含 ASCII 字节，并严格解释为 ASCII 字节序列。可以传递特殊值 `'clear'` 来清除为给定域设置的任何先前替代服务。

当为 `originOrStream` 参数传递字符串时，它将被解析为 URL，并从中派生来源。例如，HTTP URL `'https://example.org/foo/bar'` 的来源是 ASCII 字符串 `'https://example.org'`。如果给定的字符串无法解析为 URL 或无法派生有效来源，则将抛出错误。

可以传递 `URL` 对象或任何具有 `origin` 属性的对象作为 `originOrStream`，在这种情况下，将使用 `origin` 属性的值。`origin` 属性的值*必须*是正确序列化的 ASCII 来源。

#### 指定替代服务

`alt` 参数的格式由 [RFC 7838][] 严格定义为 ASCII 字符串，包含与特定主机和端口关联的“替代”协议的逗号分隔列表。

例如，值 `'h2="example.org:81"'` 表示 HTTP/2 协议在主机 `'example.org'` 上的 TCP/IP 端口 81 上可用。主机和端口*必须*包含在引号（`"`）字符内。

可以指定多个替代项，例如：`'h2="example.org:81", h2=":82"'`。

协议标识符（示例中的 `'h2'`）可以是任何有效的 [ALPN 协议 ID][]。

这些值的语法不由 Node.js 实现验证，并按用户提供或从对等端接收的方式传递。

#### `serverhttp2session.origin(...origins)`

<!-- YAML
added: v10.12.0
-->

* `origins` { string | URL | Object } 一个或多个作为单独参数传递的 URL 字符串。

向连接的客户端提交一个 `ORIGIN` 帧（由 [RFC 8336][] 定义），以通告服务器能够提供权威响应的一组来源。

```mjs
import { createSecureServer } from 'node:http2';
const options = getSecureOptionsSomehow();
const server = createSecureServer(options);
server.on('stream', (stream) => {
  stream.respond();
  stream.end('ok');
});
server.on('session', (session) => {
  session.origin('https://example.com', 'https://example.org');
});
```

```cjs
const http2 = require('node:http2');
const options = getSecureOptionsSomehow();
const server = http2.createSecureServer(options);
server.on('stream', (stream) => {
  stream.respond();
  stream.end('ok');
});
server.on('session', (session) => {
  session.origin('https://example.com', 'https://example.org');
});
```

当字符串作为 `origin` 传递时，它将被解析为 URL，并从中派生来源。例如，HTTP URL `'https://example.org/foo/bar'` 的来源是 ASCII 字符串 `'https://example.org'`。如果给定的字符串无法解析为 URL 或无法派生有效来源，则将抛出错误。

可以传递 `URL` 对象或任何具有 `origin` 属性的对象作为 `origin`，在这种情况下，将使用 `origin` 属性的值。`origin` 属性的值*必须*是正确序列化的 ASCII 来源。

或者，可以在使用 `http2.createSecureServer()` 方法创建新的 HTTP/2 服务器时使用 `origins` 选项：

```mjs
import { createSecureServer } from 'node:http2';
const options = getSecureOptionsSomehow();
options.origins = ['https://example.com', 'https://example.org'];
const server = createSecureServer(options);
server.on('stream', (stream) => {
  stream.respond();
  stream.end('ok');
});
```

```cjs
const http2 = require('node:http2');
const options = getSecureOptionsSomehow();
options.origins = ['https://example.com', 'https://example.org'];
const server = http2.createSecureServer(options);
server.on('stream', (stream) => {
  stream.respond();
  stream.end('ok');
});
```

### 类：`ClientHttp2Session`

<!-- YAML
added: v8.4.0
-->

* 扩展：{Http2Session}

#### 事件：`'altsvc'`

<!-- YAML
added: v9.4.0
-->

* `alt` {string}
* `origin` {string}
* `streamId` {number}

每当客户端接收到 `ALTSVC` 帧时，就会发出 `'altsvc'` 事件。该事件附带 `ALTSVC` 值、来源和流 ID。如果 `ALTSVC` 帧中未提供 `origin`，则 `origin` 将为空字符串。

```mjs
import { connect } from 'node:http2';
const client = connect('https://example.org');

client.on('altsvc', (alt, origin, streamId) => {
  console.log(alt);
  console.log(origin);
  console.log(streamId);
});
```

```cjs
const http2 = require('node:http2');
const client = http2.connect('https://example.org');

client.on('altsvc', (alt, origin, streamId) => {
  console.log(alt);
  console.log(origin);
  console.log(streamId);
});
```

#### 事件：`'origin'`

<!-- YAML
added: v10.12.0
-->

* `origins` {string\[]}

每当客户端接收到 `ORIGIN` 帧时，就会发出 `'origin'` 事件。该事件附带一个 `origin` 字符串数组。`http2session.originSet` 将更新以包含接收到的来源。

```mjs
import { connect } from 'node:http2';
const client = connect('https://example.org');

client.on('origin', (origins) => {
  for (let n = 0; n < origins.length; n++)
    console.log(origins[n]);
});
```

```cjs
const http2 = require('node:http2');
const client = http2.connect('https://example.org');

client.on('origin', (origins) => {
  for (let n = 0; n < origins.length; n++)
    console.log(origins[n]);
});
```

`'origin'` 事件仅在使用安全 TLS 连接时发出。

#### `clienthttp2session.request(headers[, options])`

<!-- YAML
added: v8.4.0
changes:
  - version: v24.2.0
    pr-url: https://github.com/nodejs/node/pull/58293
    description: The `weight` option is now ignored, setting it will trigger a
                 runtime warning.
  - version: v24.2.0
    pr-url: https://github.com/nodejs/node/pull/58313
    description: Following the deprecation of priority signaling as of RFC 9113,
                 `weight` option is deprecated.
  - version:
      - v24.0.0
      - v22.17.0
    pr-url: https://github.com/nodejs/node/pull/57917
    description: Allow passing headers in raw array format.
-->

* `headers` {HTTP/2 Headers Object|HTTP/2 Raw Headers}

* `options` {Object}
  * `endStream` {boolean} 如果 `Http2Stream` 的*可写*端应最初关闭，则为 `true`，例如当发送不应期望有效载荷体的 `GET` 请求时。
  * `exclusive` {boolean} 当为 `true` 且 `parent` 标识父流时，新创建的流将成为父流的唯一直接依赖项，所有其他现有依赖项将成为新创建流的依赖项。**默认值：** `false`。
  * `parent` {number} 指定新创建流所依赖的流的数字标识符。
  * `waitForTrailers` {boolean} 当为 `true` 时，`Http2Stream` 将在最终 `DATA` 帧发送后发出 `'wantTrailers'` 事件。
  * `signal` {AbortSignal} 可用于中止正在进行的请求的 AbortSignal。

* 返回：{ClientHttp2Stream}

仅适用于 HTTP/2 客户端 `Http2Session` 实例，`http2session.request()` 创建并返回一个 `Http2Stream` 实例，可用于向连接的服务器发送 HTTP/2 请求。

当首次创建 `ClientHttp2Session` 时，套接字可能尚未连接。如果在此期间调用 `clienthttp2session.request()`，则实际请求将推迟到套接字准备就绪。如果在实际请求执行之前关闭 `session`，则会抛出 `ERR_HTTP2_GOAWAY_SESSION`。

此方法仅在 `http2session.type` 等于 `http2.constants.NGHTTP2_SESSION_CLIENT` 时可用。

```mjs
import { connect, constants } from 'node:http2';
const clientSession = connect('https://localhost:1234');
const {
  HTTP2_HEADER_PATH,
  HTTP2_HEADER_STATUS,
} = constants;

const req = clientSession.request({ [HTTP2_HEADER_PATH]: '/' });
req.on('response', (headers) => {
  console.log(headers[HTTP2_HEADER_STATUS]);
  req.on('data', (chunk) => { /* .. */ });
  req.on('end', () => { /* .. */ });
});
```

```cjs
const http2 = require('node:http2');
const clientSession = http2.connect('https://localhost:1234');
const {
  HTTP2_HEADER_PATH,
  HTTP2_HEADER_STATUS,
} = http2.constants;

const req = clientSession.request({ [HTTP2_HEADER_PATH]: '/' });
req.on('response', (headers) => {
  console.log(headers[HTTP2_HEADER_STATUS]);
  req.on('data', (chunk) => { /* .. */ });
  req.on('end', () => { /* .. */ });
});
```

当设置 `options.waitForTrailers` 选项时，在排队要发送的最后一个有效载荷数据块后立即发出 `'wantTrailers'` 事件。然后可以调用 `http2stream.sendTrailers()` 方法向对等端发送尾部头部。

当设置 `options.waitForTrailers` 时，`Http2Stream` 在传输最终 `DATA` 帧时不会自动关闭。用户代码必须调用 `http2stream.sendTrailers()` 或 `http2stream.close()` 来关闭 `Http2Stream`。

当使用 `AbortSignal` 设置 `options.signal`，然后在相应的 `AbortController` 上调用 `abort` 时，请求将发出 `'error'` 事件，并附带 `AbortError` 错误。

`:method` 和 `:path` 伪头部未在 `headers` 内指定，它们分别默认为：

* `:method` = `'GET'`
* `:path` = `/`

### 类：`Http2Stream`

<!-- YAML
added: v8.4.0
-->

* 扩展：{stream.Duplex}

`Http2Stream` 类的每个实例表示在 `Http2Session` 实例上的双向 HTTP/2 通信流。任何单个 `Http2Session` 在其生命周期内最多可能有 2<sup>31</sup>-1 个 `Http2Stream` 实例。

用户代码不会直接构造 `Http2Stream` 实例。相反，这些是通过 `Http2Session` 实例创建、管理并提供给用户代码的。在服务器上，`Http2Stream` 实例要么是为了响应传入的 HTTP 请求而创建的（并通过 `'stream'` 事件交给用户代码），要么是为了响应调用 `http2stream.pushStream()` 方法而创建的。在客户端，`Http2Stream` 实例在调用 `http2session.request()` 方法时创建并返回，或者响应传入的 `'push'` 事件而创建。

`Http2Stream` 类是 [`ServerHttp2Stream`][] 和 [`ClientHttp2Stream`][] 类的基类，分别专门用于服务器或客户端。

所有 `Http2Stream` 实例都是 [`Duplex`][] 流。`Duplex` 的可写端用于向连接的对等端发送数据，而可读端用于接收连接的对等端发送的数据。

`Http2Stream` 的默认文本字符编码是 UTF-8。当使用 `Http2Stream` 发送文本时，使用 `'content-type'` 头部设置字符编码。

```js
stream.respond({
  'content-type': 'text/html; charset=utf-8',
  ':status': 200,
});
```

#### `Http2Stream` 生命周期

##### 创建

在服务器端，[`ServerHttp2Stream`][] 实例在以下情况下创建：

* 接收到具有先前未使用的流 ID 的新 HTTP/2 `HEADERS` 帧时；
* 调用 `http2stream.pushStream()` 方法时。

在客户端，当调用 `http2session.request()` 方法时，会创建 [`ClientHttp2Stream`][] 实例。

在客户端，如果父 `Http2Session` 尚未完全建立，则 `http2session.request()` 返回的 `Http2Stream` 实例可能无法立即使用。在这种情况下，在 `Http2Stream` 上调用的操作将被缓冲，直到发出 `'ready'` 事件。用户代码很少（如果有的话）需要直接处理 `'ready'` 事件。可以通过检查 `http2stream.id` 的值来确定 `Http2Stream` 的就绪状态。如果值为 `undefined`，则流尚未准备好使用。

##### 销毁

所有 [`Http2Stream`][] 实例在以下情况下被销毁：

* 接收到流的 `RST_STREAM` 帧，并且（仅适用于客户端流）已读取挂起数据。
* 调用 `http2stream.close()` 方法，并且（仅适用于客户端流）已读取挂起数据。
* 调用 `http2stream.destroy()` 或 `http2session.destroy()` 方法。

当 `Http2Stream` 实例被销毁时，将尝试向连接的对等端发送 `RST_STREAM` 帧。

当 `Http2Stream` 实例被销毁时，将发出 `'close'` 事件。因为 `Http2Stream` 是 `stream.Duplex` 的实例，如果流数据当前正在流动，也会发出 `'end'` 事件。如果 `http2stream.destroy()` 被调用并传递 `Error` 作为第一个参数，也可能发出 `'error'` 事件。

`Http2Stream` 销毁后，`http2stream.destroyed` 属性将为 `true`，`http2stream.rstCode` 属性将指定 `RST_STREAM` 错误代码。`Http2Stream` 实例一旦销毁就不再可用。

#### 事件：`'aborted'`

<!-- YAML
added: v8.4.0
-->

每当 `Http2Stream` 实例在通信过程中异常中止时，就会发出 `'aborted'` 事件。其监听器不需要任何参数。

仅当 `Http2Stream` 的可写端尚未结束时，才会发出 `'aborted'` 事件。

#### 事件：`'close'`

<!-- YAML
added: v8.4.0
-->

当 `Http2Stream` 被销毁时，会发出 `'close'` 事件。一旦发出此事件，`Http2Stream` 实例就不再可用。

关闭流时使用的 HTTP/2 错误代码可以使用 `http2stream.rstCode` 属性检索。如果代码是 `NGHTTP2_NO_ERROR` (`0`) 以外的任何值，则也会发出 `'error'` 事件。

#### 事件：`'error'`

<!-- YAML
added: v8.4.0
-->

* `error` {Error}

在处理 `Http2Stream` 期间发生错误时，会发出 `'error'` 事件。

#### 事件：`'frameError'`

<!-- YAML
added: v8.4.0
-->

* `type` {integer} 帧类型。
* `code` {integer} 错误代码。
* `id` {integer} 流 ID（如果帧不与流关联，则为 `0`）。

当尝试发送帧时发生错误时，会发出 `'frameError'` 事件。调用时，处理程序函数将接收一个标识帧类型的整数参数和一个标识错误代码的整数参数。`Http2Stream` 实例将在发出 `'frameError'` 事件后立即销毁。

#### 事件：`'ready'`

<!-- YAML
added: v8.4.0
-->

当 `Http2Stream` 已打开、已分配 `id` 并可以使用时，会发出 `'ready'` 事件。监听器不需要任何参数。

#### 事件：`'timeout'`

<!-- YAML
added: v8.4.0
-->

在此 `Http2Stream` 上使用 `http2stream.setTimeout()` 设置的毫秒数内未收到活动时，会发出 `'timeout'` 事件。其监听器不需要任何参数。

#### 事件：`'trailers'`

<!-- YAML
added: v8.4.0
-->

* `headers` {HTTP/2 Headers Object} 描述头部的对象
* `flags` {number} 相关的数字标志

当接收到与尾部头部字段关联的头部块时，会发出 `'trailers'` 事件。监听器回调传递 [HTTP/2 头部对象][] 和与头部关联的标志。

如果在接收到尾部之前调用 `http2stream.end()` 并且未读取或监听传入数据，则可能不会发出此事件。

```js
stream.on('trailers', (headers, flags) => {
  console.log(headers);
});
```

#### 事件：`'wantTrailers'`

<!-- YAML
added: v10.0.0
-->

当 `Http2Stream` 已将最终的 `DATA` 帧排队到帧上，并且 `Http2Stream` 准备好发送尾部头部时，会发出 `'wantTrailers'` 事件。在发起请求或响应时，必须设置 `waitForTrailers` 选项才能发出此事件。

#### `http2stream.aborted`

<!-- YAML
added: v8.4.0
-->

* 类型：{boolean}

如果 `Http2Stream` 实例异常中止，则设置为 `true`。当设置时，将已发出 `'aborted'` 事件。

#### `http2stream.bufferSize`

<!-- YAML
added:
 - v11.2.0
 - v10.16.0
-->

* 类型：{number}

此属性显示当前缓冲待写入的字符数。有关详细信息，请参阅 [`net.Socket.bufferSize`][]。

#### `http2stream.close(code[, callback])`

<!-- YAML
added: v8.4.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `code` {number} 标识错误代码的无符号 32 位整数。
  **默认值：** `http2.constants.NGHTTP2_NO_ERROR` (`0x00`)。
* `callback` {Function} 注册为 `'close'` 事件监听器的可选函数。

通过向连接的 HTTP/2 对等端发送 `RST_STREAM` 帧来关闭 `Http2Stream` 实例。

#### `http2stream.closed`

<!-- YAML
added: v9.4.0
-->

* 类型：{boolean}

如果 `Http2Stream` 实例已关闭，则设置为 `true`。

#### `http2stream.destroyed`

<!-- YAML
added: v8.4.0
-->

* 类型：{boolean}

如果 `Http2Stream` 实例已被销毁且不再可用，则设置为 `true`。

#### `http2stream.endAfterHeaders`

<!-- YAML
added: v10.11.0
-->

* 类型：{boolean}

如果在接收到的请求或响应 HEADERS 帧中设置了 `END_STREAM` 标志，则设置为 `true`，表示不应接收额外数据，并且 `Http2Stream` 的可读端将被关闭。

#### `http2stream.id`

<!-- YAML
added: v8.4.0
-->

* 类型：{number|undefined}

此 `Http2Stream` 实例的数字流标识符。如果尚未分配流标识符，则设置为 `undefined`。

#### `http2stream.pending`

<!-- YAML
added: v9.4.0
-->

* 类型：{boolean}

如果 `Http2Stream` 实例尚未分配数字流标识符，则设置为 `true`。

#### `http2stream.priority(options)`

<!-- YAML
added: v8.4.0
deprecated: v24.2.0
changes:
  - version: v24.2.0
    pr-url: https://github.com/nodejs/node/pull/58293
    description: This method no longer sets the priority of the stream. Using it
                 now triggers a runtime warning.
-->

> Stability: 0 - Deprecated: 对优先级信号的支持已在 [RFC 9113][] 中弃用，并且在 Node.js 中不再受支持。

空方法，仅用于保持一些向后兼容性。

#### `http2stream.rstCode`

<!-- YAML
added: v8.4.0
-->

* 类型：{number}

设置为在 `Http2Stream` 销毁后报告的 `RST_STREAM` [错误代码][]，无论是从连接的对等端接收到 `RST_STREAM` 帧、调用 `http2stream.close()` 还是 `http2stream.destroy()`。如果 `Http2Stream` 尚未关闭，则为 `undefined`。

#### `http2stream.sentHeaders`

<!-- YAML
added: v9.5.0
-->

* 类型：{HTTP/2 Headers Object}

包含为此 `Http2Stream` 发送的出站头部的对象。

#### `http2stream.sentInfoHeaders`

<!-- YAML
added: v9.5.0
-->

* 类型：{HTTP/2 Headers Object\[]}

包含为此 `Http2Stream` 发送的出站信息性（附加）头部的对象数组。

#### `http2stream.sentTrailers`

<!-- YAML
added: v9.5.0
-->

* 类型：{HTTP/2 Headers Object}

包含为此 `HttpStream` 发送的出站尾部的对象。

#### `http2stream.session`

<!-- YAML
added: v8.4.0
-->

* 类型：{Http2Session}

对拥有此 `Http2Stream` 的 `Http2Session` 实例的引用。在 `Http2Stream` 实例销毁后，该值将为 `undefined`。

#### `http2stream.setTimeout(msecs, callback)`

<!-- YAML
added: v8.4.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `msecs` {number}
* `callback` {Function}

```mjs
import { connect, constants } from 'node:http2';
const client = connect('http://example.org:8000');
const { NGHTTP2_CANCEL } = constants;
const req = client.request({ ':path': '/' });

// 如果 5 秒后没有活动，取消流
req.setTimeout(5000, () => req.close(NGHTTP2_CANCEL));
```

```cjs
const http2 = require('node:http2');
const client = http2.connect('http://example.org:8000');
const { NGHTTP2_CANCEL } = http2.constants;
const req = client.request({ ':path': '/' });

// 如果 5 秒后没有活动，取消流
req.setTimeout(5000, () => req.close(NGHTTP2_CANCEL));
```

#### `http2stream.state`

<!-- YAML
added: v8.4.0
changes:
  - version: v24.2.0
    pr-url: https://github.com/nodejs/node/pull/58293
    description: The `state.weight` property is now always set to 16 and
                 `sumDependencyWeight` is always set to 0.
  - version: v24.2.0
    pr-url: https://github.com/nodejs/node/pull/58313
    description: Following the deprecation of priority signaling as of RFC 9113,
                 `weight` and `sumDependencyWeight` options are deprecated.
-->

提供有关 `Http2Stream` 当前状态的各种信息。

* 类型：{Object}
  * `localWindowSize` {number} 连接的对等端可以为此 `Http2Stream` 发送而不接收 `WINDOW_UPDATE` 的字节数。
  * `state` {number} 由 `nghttp2` 确定的 `Http2Stream` 低级当前状态的标志。
  * `localClose` {number} 如果此 `Http2Stream` 已在本地关闭，则为 `1`。
  * `remoteClose` {number} 如果此 `Http2Stream` 已在远程关闭，则为 `1`。
  * `sumDependencyWeight` {number} 传统属性，始终设置为 `0`。
  * `weight` {number} 传统属性，始终设置为 `16`。

此 `Http2Stream` 的当前状态。

#### `http2stream.sendTrailers(headers)`

<!-- YAML
added: v10.0.0
-->

* `headers` {HTTP/2 Headers Object}

向连接的 HTTP/2 对等端发送尾部 `HEADERS` 帧。此方法将导致 `Http2Stream` 立即关闭，并且必须仅在发出 `'wantTrailers'` 事件后调用。在发送请求或发送响应时，必须设置 `options.waitForTrailers` 选项，以便在最终 `DATA` 帧之后保持 `Http2Stream` 打开，从而可以发送尾部。

```mjs
import { createServer } from 'node:http2';
const server = createServer();
server.on('stream', (stream) => {
  stream.respond(undefined, { waitForTrailers: true });
  stream.on('wantTrailers', () => {
    stream.sendTrailers({ xyz: 'abc' });
  });
  stream.end('Hello World');
});
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer();
server.on('stream', (stream) => {
  stream.respond(undefined, { waitForTrailers: true });
  stream.on('wantTrailers', () => {
    stream.sendTrailers({ xyz: 'abc' });
  });
  stream.end('Hello World');
});
```

HTTP/1 规范禁止尾部包含 HTTP/2 伪头部字段（例如 `':method'`、`':path'` 等）。

### 类：`ClientHttp2Stream`

<!-- YAML
added: v8.4.0
-->

* 扩展 {Http2Stream}

`ClientHttp2Stream` 类是 `Http2Stream` 的扩展，专门用于 HTTP/2 客户端。客户端的 `Http2Stream` 实例提供仅在客户端相关的事件，如 `'response'` 和 `'push'`。

#### 事件：`'continue'`

<!-- YAML
added: v8.5.0
-->

当服务器发送 `100 Continue` 状态时发出，通常是因为请求包含 `Expect: 100-continue`。这是一个指示客户端应发送请求体的指令。

#### 事件：`'headers'`

<!-- YAML
added: v8.4.0
-->

* `headers` {HTTP/2 Headers Object}
* `flags` {number}
* `rawHeaders` {HTTP/2 Raw Headers}

当接收到流的额外头部块时，会发出 `'headers'` 事件，例如当接收到 `1xx` 信息性头部块时。监听器回调传递 [HTTP/2 头部对象][]、与头部关联的标志以及原始格式的头部（参见 [HTTP/2 原始头部][]）。

```js
stream.on('headers', (headers, flags) => {
  console.log(headers);
});
```

#### 事件：`'push'`

<!-- YAML
added: v8.4.0
-->

* `headers` {HTTP/2 Headers Object}
* `flags` {number}

当接收到服务器推送流的响应头部时，会发出 `'push'` 事件。监听器回调传递 [HTTP/2 头部对象][] 和与头部关联的标志。

```js
stream.on('push', (headers, flags) => {
  console.log(headers);
});
```

#### 事件：`'response'`

<!-- YAML
added: v8.4.0
-->

* `headers` {HTTP/2 Headers Object}
* `flags` {number}
* `rawHeaders` {HTTP/2 Raw Headers}

当从连接的 HTTP/2 服务器接收到此流的响应 `HEADERS` 帧时，会发出 `'response'` 事件。监听器以三个参数调用：一个包含接收到的 [HTTP/2 头部对象][] 的 `Object`、与头部关联的标志以及原始格式的头部（参见 [HTTP/2 原始头部][]）。

```mjs
import { connect } from 'node:http2';
const client = connect('https://localhost');
const req = client.request({ ':path': '/' });
req.on('response', (headers, flags) => {
  console.log(headers[':status']);
});
```

```cjs
const http2 = require('node:http2');
const client = http2.connect('https://localhost');
const req = client.request({ ':path': '/' });
req.on('response', (headers, flags) => {
  console.log(headers[':status']);
});
```

### 类：`ServerHttp2Stream`

<!-- YAML
added: v8.4.0
-->

* 扩展：{Http2Stream}

`ServerHttp2Stream` 类是 [`Http2Stream`][] 的扩展，专门用于 HTTP/2 服务器。服务器上的 `Http2Stream` 实例提供了额外的方法，如 `http2stream.pushStream()` 和 `http2stream.respond()`，这些方法仅在服务器上相关。

#### `http2stream.additionalHeaders(headers)`

<!-- YAML
added: v8.4.0
-->

* `headers` {HTTP/2 Headers Object}

向连接的 HTTP/2 对等端发送额外的信息性 `HEADERS` 帧。

#### `http2stream.headersSent`

<!-- YAML
added: v8.4.0
-->

* 类型：{boolean}

如果头部已发送，则为 true，否则为 false（只读）。

#### `http2stream.pushAllowed`

<!-- YAML
added: v8.4.0
-->

* 类型：{boolean}

只读属性，映射到远程客户端最近 `SETTINGS` 帧的 `SETTINGS_ENABLE_PUSH` 标志。如果远程对等端接受推送流，则为 `true`，否则为 `false`。在同一 `Http2Session` 中的每个 `Http2Stream` 的设置是相同的。

#### `http2stream.pushStream(headers[, options], callback)`

<!-- YAML
added: v8.4.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `headers` {HTTP/2 Headers Object}
* `options` {Object}
  * `exclusive` {boolean} 当为 `true` 且 `parent` 标识父流时，新创建的流将成为父流的唯一直接依赖项，所有其他现有依赖项将成为新创建流的依赖项。**默认值：** `false`。
  * `parent` {number} 指定新创建流所依赖的流的数字标识符。
* `callback` {Function} 一旦推送流启动后调用的回调。
  * `err` {Error}
  * `pushStream` {ServerHttp2Stream} 返回的 `pushStream` 对象。
  * `headers` {HTTP/2 Headers Object} 启动 `pushStream` 的头部对象。

启动一个推送流。回调以为新推送流创建的新 `Http2Stream` 实例作为第二个参数调用，或者以作为第一个参数传递的错误调用。

```mjs
import { createServer } from 'node:http2';
const server = createServer();
server.on('stream', (stream) => {
  stream.respond({ ':status': 200 });
  stream.pushStream({ ':path': '/' }, (err, pushStream, headers) => {
    if (err) throw err;
    pushStream.respond({ ':status': 200 });
    pushStream.end('some pushed data');
  });
  stream.end('some data');
});
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer();
server.on('stream', (stream) => {
  stream.respond({ ':status': 200 });
  stream.pushStream({ ':path': '/' }, (err, pushStream, headers) => {
    if (err) throw err;
    pushStream.respond({ ':status': 200 });
    pushStream.end('some pushed data');
  });
  stream.end('some data');
});
```

在 `HEADERS` 帧中设置推送流的权重是不允许的。将 `weight` 值传递给 `http2stream.priority`，并将 `silent` 选项设置为 `true`，以启用服务器端并发流之间的带宽平衡。

不允许从推送流内部调用 `http2stream.pushStream()`，并将抛出错误。

#### `http2stream.respond([headers[, options]])`

<!-- YAML
added: v8.4.0
changes:
  - version:
    - v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59455
    description: Allow passing headers in raw array format.
  - version:
    - v14.5.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/33160
    description: Allow explicitly setting date headers.
-->

* `headers` {HTTP/2 Headers Object|HTTP/2 Raw Headers}
* `options` {Object}
  * `endStream` {boolean} 设置为 `true` 表示响应将不包含有效载荷数据。
  * `waitForTrailers` {boolean} 当为 `true` 时，`Http2Stream` 将在最终 `DATA` 帧发送后发出 `'wantTrailers'` 事件。

```mjs
import { createServer } from 'node:http2';
const server = createServer();
server.on('stream', (stream) => {
  stream.respond({ ':status': 200 });
  stream.end('some data');
});
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer();
server.on('stream', (stream) => {
  stream.respond({ ':status': 200 });
  stream.end('some data');
});
```

启动一个响应。当设置 `options.waitForTrailers` 选项时，在排队要发送的最后一个有效载荷数据块后立即发出 `'wantTrailers'` 事件。然后可以使用 `http2stream.sendTrailers()` 方法向对等端发送尾部头部字段。

当设置 `options.waitForTrailers` 时，`Http2Stream` 在传输最终 `DATA` 帧时不会自动关闭。用户代码必须调用 `http2stream.sendTrailers()` 或 `http2stream.close()` 来关闭 `Http2Stream`。

```mjs
import { createServer } from 'node:http2';
const server = createServer();
server.on('stream', (stream) => {
  stream.respond({ ':status': 200 }, { waitForTrailers: true });
  stream.on('wantTrailers', () => {
    stream.sendTrailers({ ABC: 'some value to send' });
  });
  stream.end('some data');
});
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer();
server.on('stream', (stream) => {
  stream.respond({ ':status': 200 }, { waitForTrailers: true });
  stream.on('wantTrailers', () => {
    stream.sendTrailers({ ABC: 'some value to send' });
  });
  stream.end('some data');
});
```

#### `http2stream.respondWithFD(fd[, headers[, options]])`

<!-- YAML
added: v8.4.0
changes:
  - version:
    - v14.5.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/33160
    description: Allow explicitly setting date headers.
  - version: v12.12.0
    pr-url: https://github.com/nodejs/node/pull/29876
    description: The `fd` option may now be a `FileHandle`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18936
    description: Any readable file descriptor, not necessarily for a
                 regular file, is supported now.
-->

* `fd` {number|FileHandle} 一个可读的文件描述符。
* `headers` {HTTP/2 Headers Object}
* `options` {Object}
  * `statCheck` {Function}
  * `waitForTrailers` {boolean} 当为 `true` 时，`Http2Stream` 将在最终 `DATA` 帧发送后发出 `'wantTrailers'` 事件。
  * `offset` {number} 开始读取的偏移位置。
  * `length` {number} 从 fd 发送的数据量。

启动一个响应，其数据从给定的文件描述符读取。不对给定的文件描述符执行任何验证。如果在尝试使用文件描述符读取数据时发生错误，`Http2Stream` 将使用标准 `INTERNAL_ERROR` 代码通过 `RST_STREAM` 帧关闭。

使用时，`Http2Stream` 对象的 `Duplex` 接口将自动关闭。

```mjs
import { createServer } from 'node:http2';
import { openSync, fstatSync, closeSync } from 'node:fs';

const server = createServer();
server.on('stream', (stream) => {
  const fd = openSync('/some/file', 'r');

  const stat = fstatSync(fd);
  const headers = {
    'content-length': stat.size,
    'last-modified': stat.mtime.toUTCString(),
    'content-type': 'text/plain; charset=utf-8',
  };
  stream.respondWithFD(fd, headers);
  stream.on('close', () => closeSync(fd));
});
```

```cjs
const http2 = require('node:http2');
const fs = require('node:fs');

const server = http2.createServer();
server.on('stream', (stream) => {
  const fd = fs.openSync('/some/file', 'r');

  const stat = fs.fstatSync(fd);
  const headers = {
    'content-length': stat.size,
    'last-modified': stat.mtime.toUTCString(),
    'content-type': 'text/plain; charset=utf-8',
  };
  stream.respondWithFD(fd, headers);
  stream.on('close', () => fs.closeSync(fd));
});
```

可以指定可选的 `options.statCheck` 函数，以使用户代码有机会基于给定 fd 的 `fs.Stat` 详细信息设置额外的内容头部。如果提供了 `statCheck` 函数，`http2stream.respondWithFD()` 方法将执行 `fs.fstat()` 调用来收集有关提供的文件描述符的详细信息。

`offset` 和 `length` 选项可用于将响应限制为特定的范围子集。例如，这可以用于支持 HTTP Range 请求。

当流关闭时，文件描述符或 `FileHandle` 不会关闭，因此一旦不再需要，需要手动关闭。不支持为多个流同时使用相同的文件描述符，可能会导致数据丢失。在流完成后重用文件描述符是支持的。

当设置 `options.waitForTrailers` 选项时，在排队要发送的最后一个有效载荷数据块后立即发出 `'wantTrailers'` 事件。然后可以使用 `http2stream.sendTrailers()` 方法向对等端发送尾部头部字段。

当设置 `options.waitForTrailers` 时，`Http2Stream` 在传输最终 `DATA` 帧时不会自动关闭。用户代码*必须*调用 `http2stream.sendTrailers()` 或 `http2stream.close()` 来关闭 `Http2Stream`。

```mjs
import { createServer } from 'node:http2';
import { openSync, fstatSync, closeSync } from 'node:fs';

const server = createServer();
server.on('stream', (stream) => {
  const fd = openSync('/some/file', 'r');

  const stat = fstatSync(fd);
  const headers = {
    'content-length': stat.size,
    'last-modified': stat.mtime.toUTCString(),
    'content-type': 'text/plain; charset=utf-8',
  };
  stream.respondWithFD(fd, headers, { waitForTrailers: true });
  stream.on('wantTrailers', () => {
    stream.sendTrailers({ ABC: 'some value to send' });
  });

  stream.on('close', () => closeSync(fd));
});
```

```cjs
const http2 = require('node:http2');
const fs = require('node:fs');

const server = http2.createServer();
server.on('stream', (stream) => {
  const fd = fs.openSync('/some/file', 'r');

  const stat = fs.fstatSync(fd);
  const headers = {
    'content-length': stat.size,
    'last-modified': stat.mtime.toUTCString(),
    'content-type': 'text/plain; charset=utf-8',
  };
  stream.respondWithFD(fd, headers, { waitForTrailers: true });
  stream.on('wantTrailers', () => {
    stream.sendTrailers({ ABC: 'some value to send' });
  });

  stream.on('close', () => fs.closeSync(fd));
});
```

#### `http2stream.respondWithFile(path[, headers[, options]])`

<!-- YAML
added: v8.4.0
changes:
  - version:
    - v14.5.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/33160
    description: Allow explicitly setting date headers.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18936
    description: Any readable file, not necessarily a
                 regular file, is supported now.
-->

* `path` {string|Buffer|URL}
* `headers` {HTTP/2 Headers Object}
* `options` {Object}
  * `statCheck` {Function}
  * `onError` {Function} 在发送前发生错误时调用的回调函数。
  * `waitForTrailers` {boolean} 当为 `true` 时，`Http2Stream` 将在最终 `DATA` 帧发送后发出 `'wantTrailers'` 事件。
  * `offset` {number} 开始读取的偏移位置。
  * `length` {number} 从 fd 发送的数据量。

发送一个常规文件作为响应。`path` 必须指定一个常规文件，否则将在 `Http2Stream` 对象上发出 `'error'` 事件。

使用时，`Http2Stream` 对象的 `Duplex` 接口将自动关闭。

可选的 `options.statCheck` 函数可以指定，以使用户代码有机会基于给定文件的 `fs.Stat` 详细信息设置额外的内容头部：

如果尝试读取文件数据时发生错误，`Http2Stream` 将使用标准 `INTERNAL_ERROR` 代码通过 `RST_STREAM` 帧关闭。如果定义了 `onError` 回调，则将调用它。否则流将被销毁。

使用文件路径的示例：

```mjs
import { createServer } from 'node:http2';
const server = createServer();
server.on('stream', (stream) => {
  function statCheck(stat, headers) {
    headers['last-modified'] = stat.mtime.toUTCString();
  }

  function onError(err) {
    // 如果流已被另一方销毁，stream.respond() 可能会抛出。
    try {
      if (err.code === 'ENOENT') {
        stream.respond({ ':status': 404 });
      } else {
        stream.respond({ ':status': 500 });
      }
    } catch (err) {
      // 执行实际错误处理。
      console.error(err);
    }
    stream.end();
  }

  stream.respondWithFile('/some/file',
                         { 'content-type': 'text/plain; charset=utf-8' },
                         { statCheck, onError });
});
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer();
server.on('stream', (stream) => {
  function statCheck(stat, headers) {
    headers['last-modified'] = stat.mtime.toUTCString();
  }

  function onError(err) {
    // 如果流已被另一方销毁，stream.respond() 可能会抛出。
    try {
      if (err.code === 'ENOENT') {
        stream.respond({ ':status': 404 });
      } else {
        stream.respond({ ':status': 500 });
      }
    } catch (err) {
      // 执行实际错误处理。
      console.error(err);
    }
    stream.end();
  }

  stream.respondWithFile('/some/file',
                         { 'content-type': 'text/plain; charset=utf-8' },
                         { statCheck, onError });
});
```

`options.statCheck` 函数也可用于通过返回 `false` 来取消发送操作。例如，条件请求可能会检查统计结果以确定文件是否已被修改以返回适当的 `304` 响应：

```mjs
import { createServer } from 'node:http2';
const server = createServer();
server.on('stream', (stream) => {
  function statCheck(stat, headers) {
    // 在此处检查统计信息...
    stream.respond({ ':status': 304 });
    return false; // 取消发送操作
  }
  stream.respondWithFile('/some/file',
                         { 'content-type': 'text/plain; charset=utf-8' },
                         { statCheck });
});
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer();
server.on('stream', (stream) => {
  function statCheck(stat, headers) {
    // 在此处检查统计信息...
    stream.respond({ ':status': 304 });
    return false; // 取消发送操作
  }
  stream.respondWithFile('/some/file',
                         { 'content-type': 'text/plain; charset=utf-8' },
                         { statCheck });
});
```

`content-length` 头部字段将自动设置。

`offset` 和 `length` 选项可用于将响应限制为特定的范围子集。例如，这可以用于支持 HTTP Range 请求。

`options.onError` 函数也可用于处理在文件交付开始之前可能发生的所有错误。默认行为是销毁流。

当设置 `options.waitForTrailers` 选项时，在排队要发送的最后一个有效载荷数据块后立即发出 `'wantTrailers'` 事件。然后可以使用 `http2stream.sendTrailers()` 方法向对等端发送尾部头部字段。

当设置 `options.waitForTrailers` 时，`Http2Stream` 在传输最终 `DATA` 帧时不会自动关闭。用户代码必须调用 `http2stream.sendTrailers()` 或 `http2stream.close()` 来关闭 `Http2Stream`。

```mjs
import { createServer } from 'node:http2';
const server = createServer();
server.on('stream', (stream) => {
  stream.respondWithFile('/some/file',
                         { 'content-type': 'text/plain; charset=utf-8' },
                         { waitForTrailers: true });
  stream.on('wantTrailers', () => {
    stream.sendTrailers({ ABC: 'some value to send' });
  });
});
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer();
server.on('stream', (stream) => {
  stream.respondWithFile('/some/file',
                         { 'content-type': 'text/plain; charset=utf-8' },
                         { waitForTrailers: true });
  stream.on('wantTrailers', () => {
    stream.sendTrailers({ ABC: 'some value to send' });
  });
});
```

### 类：`Http2Server`

<!-- YAML
added: v8.4.0
-->

* 扩展：{net.Server}

`Http2Server` 的实例是使用 `http2.createServer()` 函数创建的。`Http2Server` 类不是由 `node:http2` 模块直接导出的。

#### 事件：`'checkContinue'`

<!-- YAML
added: v8.5.0
-->

* `request` {http2.Http2ServerRequest}
* `response` {http2.Http2ServerResponse}

如果注册了 [`'request'`][] 监听器或向 [`http2.createServer()`][] 提供了回调函数，则每次收到带有 HTTP `Expect: 100-continue` 的请求时都会发出 `'checkContinue'` 事件。如果未监听此事件，服务器将自动响应状态 `100 Continue`。

处理此事件涉及调用 [`response.writeContinue()`][] 如果客户端应继续发送请求体，或者生成适当的 HTTP 响应（例如 400 Bad Request）如果客户端不应继续发送请求体。

当发出并处理此事件时，将不会发出 [`'request'`][] 事件。

#### 事件：`'connection'`

<!-- YAML
added: v8.4.0
-->

* `socket` {stream.Duplex}

当建立新的 TCP 流时发出此事件。`socket` 通常是 [`net.Socket`][] 类型的对象。通常用户不会想要访问此事件。

此事件也可以由用户显式发出以将连接注入 HTTP 服务器。在这种情况下，可以传递任何 [`Duplex`][] 流。

#### 事件：`'request'`

<!-- YAML
added: v8.4.0
-->

* `request` {http2.Http2ServerRequest}
* `response` {http2.Http2ServerResponse}

每次有请求时发出。每个会话可能有多个请求。请参阅 [兼容性 API][]。

#### 事件：`'session'`

<!-- YAML
added: v8.4.0
-->

* `session` {ServerHttp2Session}

当 `Http2Server` 创建新的 `Http2Session` 时，会发出 `'session'` 事件。

#### 事件：`'sessionError'`

<!-- YAML
added: v8.4.0
-->

* `error` {Error}
* `session` {ServerHttp2Session}

当与 `Http2Server` 关联的 `Http2Session` 对象发出 `'error'` 事件时，会发出 `'sessionError'` 事件。

#### 事件：`'stream'`

<!-- YAML
added: v8.4.0
-->

* `stream` {Http2Stream} 对流的引用
* `headers` {HTTP/2 Headers Object} 描述头部的对象
* `flags` {number} 相关的数字标志
* `rawHeaders` {HTTP/2 Raw Headers} 包含原始头部的数组

当与服务器关联的 `Http2Session` 发出 `'stream'` 事件时，会发出 `'stream'` 事件。

另请参阅 [`Http2Session` 的 `'stream'` 事件][]。

```mjs
import { createServer, constants } from 'node:http2';
const {
  HTTP2_HEADER_METHOD,
  HTTP2_HEADER_PATH,
  HTTP2_HEADER_STATUS,
  HTTP2_HEADER_CONTENT_TYPE,
} = constants;

const server = createServer();
server.on('stream', (stream, headers, flags) => {
  const method = headers[HTTP2_HEADER_METHOD];
  const path = headers[HTTP2_HEADER_PATH];
  // ...
  stream.respond({
    [HTTP2_HEADER_STATUS]: 200,
    [HTTP2_HEADER_CONTENT_TYPE]: 'text/plain; charset=utf-8',
  });
  stream.write('hello ');
  stream.end('world');
});
```

```cjs
const http2 = require('node:http2');
const {
  HTTP2_HEADER_METHOD,
  HTTP2_HEADER_PATH,
  HTTP2_HEADER_STATUS,
  HTTP2_HEADER_CONTENT_TYPE,
} = http2.constants;

const server = http2.createServer();
server.on('stream', (stream, headers, flags) => {
  const method = headers[HTTP2_HEADER_METHOD];
  const path = headers[HTTP2_HEADER_PATH];
  // ...
  stream.respond({
    [HTTP2_HEADER_STATUS]: 200,
    [HTTP2_HEADER_CONTENT_TYPE]: 'text/plain; charset=utf-8',
  });
  stream.write('hello ');
  stream.end('world');
});
```

#### 事件：`'timeout'`

<!-- YAML
added: v8.4.0
changes:
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/27558
    description: The default timeout changed from 120s to 0 (no timeout).
-->

当服务器在给定毫秒数内没有活动时，会发出 `'timeout'` 事件，该时间使用 `http2server.setTimeout()` 设置。**默认值：** 0（无超时）。

#### `server.close([callback])`

<!-- YAML
added: v8.4.0
-->

* `callback` {Function}

阻止服务器建立新会话。这不会阻止由于 HTTP/2 会话的持久性而创建新的请求流。要优雅地关闭服务器，请在所有活动会话上调用 [`http2session.close()`][]。

如果提供了 `callback`，则直到所有活动会话关闭后才会调用它，尽管服务器已经停止允许新会话。有关更多详细信息，请参阅 [`net.Server.close()`][]。

#### `server[Symbol.asyncDispose]()`

<!-- YAML
added: v20.4.0
changes:
 - version: v24.2.0
   pr-url: https://github.com/nodejs/node/pull/58467
   description: No longer experimental.
-->

调用 [`server.close()`][] 并返回一个在服务器关闭时履行的 promise。

#### `server.setTimeout([msecs][, callback])`

<!-- YAML
added: v8.4.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/27558
    description: The default timeout changed from 120s to 0 (no timeout).
-->

* `msecs` {number} **默认值：** 0（无超时）
* `callback` {Function}
* 返回：{Http2Server}

用于设置 http2 服务器请求的超时值，并设置一个回调函数，当在 `Http2Server` 上经过 `msecs` 毫秒后没有活动时调用。

给定的回调函数被注册为 `'timeout'` 事件的监听器。

如果 `callback` 不是函数，将抛出新的 `ERR_INVALID_ARG_TYPE` 错误。

#### `server.timeout`

<!-- YAML
added: v8.4.0
changes:
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/27558
    description: The default timeout changed from 120s to 0 (no timeout).
-->

* 类型：{number} 超时时间（毫秒）。**默认值：** 0（无超时）

在假定套接字超时之前的不活动毫秒数。

值为 `0` 将禁用传入连接的超时行为。

套接字超时逻辑在连接时设置，因此更改此值仅影响服务器的新连接，而不影响任何现有连接。

#### `server.updateSettings([settings])`

<!-- YAML
added:
  - v15.1.0
  - v14.17.0
-->

* `settings` {HTTP/2 Settings Object}

用于使用提供的设置更新服务器。

对于无效的 `settings` 值，抛出 `ERR_HTTP2_INVALID_SETTING_VALUE`。

对于无效的 `settings` 参数，抛出 `ERR_INVALID_ARG_TYPE`。

### 类：`Http2SecureServer`

<!-- YAML
added: v8.4.0
-->

* 扩展：{tls.Server}

`Http2SecureServer` 的实例是使用 `http2.createSecureServer()` 函数创建的。`Http2SecureServer` 类不是由 `node:http2` 模块直接导出的。

#### 事件：`'checkContinue'`

<!-- YAML
added: v8.5.0
-->

* `request` {http2.Http2ServerRequest}
* `response` {http2.Http2ServerResponse}

如果注册了 [`'request'`][] 监听器或向 [`http2.createSecureServer()`][] 提供了回调函数，则每次收到带有 HTTP `Expect: 100-continue` 的请求时都会发出 `'checkContinue'` 事件。如果未监听此事件，服务器将自动响应状态 `100 Continue`。

处理此事件涉及调用 [`response.writeContinue()`][] 如果客户端应继续发送请求体，或者生成适当的 HTTP 响应（例如 400 Bad Request）如果客户端不应继续发送请求体。

当发出并处理此事件时，将不会发出 [`'request'`][] 事件。

#### 事件：`'connection'`

<!-- YAML
added: v8.4.0
-->

* `socket` {stream.Duplex}

当建立新的 TCP 流时发出此事件，在 TLS 握手开始之前。`socket` 通常是 [`net.Socket`][] 类型的对象。通常用户不会想要访问此事件。

此事件也可以由用户显式发出以将连接注入 HTTP 服务器。在这种情况下，可以传递任何 [`Duplex`][] 流。

#### 事件：`'request'`

<!-- YAML
added: v8.4.0
-->

* `request` {http2.Http2ServerRequest}
* `response` {http2.Http2ServerResponse}

每次有请求时发出。每个会话可能有多个请求。请参阅 [兼容性 API][]。

#### 事件：`'session'`

<!-- YAML
added: v8.4.0
-->

* `session` {ServerHttp2Session}

当 `Http2SecureServer` 创建新的 `Http2Session` 时，会发出 `'session'` 事件。

#### 事件：`'sessionError'`

<!-- YAML
added: v8.4.0
-->

* `error` {Error}
* `session` {ServerHttp2Session}

当与 `Http2SecureServer` 关联的 `Http2Session` 对象发出 `'error'` 事件时，会发出 `'sessionError'` 事件。

#### 事件：`'stream'`

<!-- YAML
added: v8.4.0
-->

* `stream` {Http2Stream} 对流的引用
* `headers` {HTTP/2 Headers Object} 描述头部的对象
* `flags` {number} 相关的数字标志
* `rawHeaders` {HTTP/2 Raw Headers} 包含原始头部的数组

当与服务器关联的 `Http2Session` 发出 `'stream'` 事件时，会发出 `'stream'` 事件。

另请参阅 [`Http2Session` 的 `'stream'` 事件][]。

```mjs
import { createSecureServer, constants } from 'node:http2';
const {
  HTTP2_HEADER_METHOD,
  HTTP2_HEADER_PATH,
  HTTP2_HEADER_STATUS,
  HTTP2_HEADER_CONTENT_TYPE,
} = constants;

const options = getOptionsSomehow();

const server = createSecureServer(options);
server.on('stream', (stream, headers, flags) => {
  const method = headers[HTTP2_HEADER_METHOD];
  const path = headers[HTTP2_HEADER_PATH];
  // ...
  stream.respond({
    [HTTP2_HEADER_STATUS]: 200,
    [HTTP2_HEADER_CONTENT_TYPE]: 'text/plain; charset=utf-8',
  });
  stream.write('hello ');
  stream.end('world');
});
```

```cjs
const http2 = require('node:http2');
const {
  HTTP2_HEADER_METHOD,
  HTTP2_HEADER_PATH,
  HTTP2_HEADER_STATUS,
  HTTP2_HEADER_CONTENT_TYPE,
} = http2.constants;

const options = getOptionsSomehow();

const server = http2.createSecureServer(options);
server.on('stream', (stream, headers, flags) => {
  const method = headers[HTTP2_HEADER_METHOD];
  const path = headers[HTTP2_HEADER_PATH];
  // ...
  stream.respond({
    [HTTP2_HEADER_STATUS]: 200,
    [HTTP2_HEADER_CONTENT_TYPE]: 'text/plain; charset=utf-8',
  });
  stream.write('hello ');
  stream.end('world');
});
```

#### 事件：`'timeout'`

<!-- YAML
added: v8.4.0
-->

当服务器在给定毫秒数内没有活动时，会发出 `'timeout'` 事件，该时间使用 `http2secureServer.setTimeout()` 设置。**默认值：** 2 分钟。

#### 事件：`'unknownProtocol'`

<!-- YAML
added: v8.4.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44031
    description: This event will only be emitted if the client did not transmit
                 an ALPN extension during the TLS handshake.
-->

* `socket` {stream.Duplex}

当连接的客户端未能协商允许的协议（即 HTTP/2 或 HTTP/1.1）时，会发出 `'unknownProtocol'` 事件。事件处理程序接收用于处理的套接字。如果未注册此事件的监听器，则连接将终止。可以使用传递给 [`http2.createSecureServer()`][] 的 `'unknownProtocolTimeout'` 选项指定超时。

在早期版本的 Node.js 中，如果 `allowHTTP1` 为 `false`，并且在 TLS 握手期间，客户端要么不发送 ALPN 扩展，要么发送不包括 HTTP/2 (`h2`) 的 ALPN 扩展，则会发出此事件。较新版本的 Node.js 仅当 `allowHTTP1` 为 `false` 且客户端未发送 ALPN 扩展时才发出此事件。如果客户端发送不包括 HTTP/2（或如果 `allowHTTP1` 为 `true` 则不包括 HTTP/1.1）的 ALPN 扩展，TLS 握手将失败，并且不会建立安全连接。

请参阅 [兼容性 API][]。

#### `server.close([callback])`

<!-- YAML
added: v8.4.0
-->

* `callback` {Function}

阻止服务器建立新会话。这不会阻止由于 HTTP/2 会话的持久性而创建新的请求流。要优雅地关闭服务器，请在所有活动会话上调用 [`http2session.close()`][]。

如果提供了 `callback`，则直到所有活动会话关闭后才会调用它，尽管服务器已经停止允许新会话。有关更多详细信息，请参阅 [`tls.Server.close()`][]。

#### `server.setTimeout([msecs][, callback])`

<!-- YAML
added: v8.4.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `msecs` {number} **默认值：** `120000`（2 分钟）
* `callback` {Function}
* 返回：{Http2SecureServer}

用于设置 http2 安全服务器请求的超时值，并设置一个回调函数，当在 `Http2SecureServer` 上经过 `msecs` 毫秒后没有活动时调用。

给定的回调函数被注册为 `'timeout'` 事件的监听器。

如果 `callback` 不是函数，将抛出新的 `ERR_INVALID_ARG_TYPE` 错误。

#### `server.timeout`

<!-- YAML
added: v8.4.0
changes:
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/27558
    description: The default timeout changed from 120s to 0 (no timeout).
-->

* 类型：{number} 超时时间（毫秒）。**默认值：** 0（无超时）

在假定套接字超时之前的不活动毫秒数。

值为 `0` 将禁用传入连接的超时行为。

套接字超时逻辑在连接时设置，因此更改此值仅影响服务器的新连接，而不影响任何现有连接。

#### `server.updateSettings([settings])`

<!-- YAML
added:
  - v15.1.0
  - v14.17.0
-->

* `settings` {HTTP/2 Settings Object}

用于使用提供的设置更新服务器。

对于无效的 `settings` 值，抛出 `ERR_HTTP2_INVALID_SETTING_VALUE`。

对于无效的 `settings` 参数，抛出 `ERR_INVALID_ARG_TYPE`。

### `http2.createServer([options][, onRequestHandler])`

<!-- YAML
added: v8.4.0
changes:
  - version:
      - v23.0.0
      - v22.10.0
    pr-url: https://github.com/nodejs/node/pull/54875
    description: Added `streamResetBurst` and `streamResetRate`.
  - version:
      - v15.10.0
      - v14.16.0
      - v12.21.0
      - v10.24.0
    pr-url: https://github.com/nodejs-private/node-private/pull/246
    description: Added `unknownProtocolTimeout` option with a default of 10000.
  - version:
     - v14.4.0
     - v12.18.0
     - v10.21.0
    commit: 3948830ce6408be620b09a70bf66158623022af0
    pr-url: https://github.com/nodejs-private/node-private/pull/204
    description: Added `maxSettings` option with a default of 32.
  - version:
     - v13.3.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30534
    description: Added `maxSessionRejectedStreams` option with a default of 100.
  - version:
     - v13.3.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30534
    description: Added `maxSessionInvalidFrames` option with a default of 1000.
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/29144
    description: The `PADDING_STRATEGY_CALLBACK` has been made equivalent to
                 providing `PADDING_STRATEGY_ALIGNED` and `selectPadding`
                 has been removed.
  - version: v12.4.0
    pr-url: https://github.com/nodejs/node/pull/27782
    description: The `options` parameter now supports `net.createServer()`
                 options.
  - version: v9.6.0
    pr-url: https://github.com/nodejs/node/pull/15752
    description: Added the `Http1IncomingMessage` and `Http1ServerResponse`
                 option.
  - version: v8.9.3
    pr-url: https://github.com/nodejs/node/pull/17105
    description: Added the `maxOutstandingPings` option with a default limit of
                 10.
  - version: v8.9.3
    pr-url: https://github.com/nodejs/node/pull/16676
    description: Added the `maxHeaderListPairs` option with a default limit of
                 128 header pairs.
-->

* `options` {Object}
  * `maxDeflateDynamicTableSize` {number} 设置用于压缩头部字段的最大动态表大小。**默认值：** `4Kib`。
  * `maxSettings` {number} 设置每个 `SETTINGS` 帧的最大设置条目数。允许的最小值为 `1`。**默认值：** `32`。
  * `maxSessionMemory`{number} 设置 `Http2Session` 允许使用的最大内存。该值以兆字节数表示，例如 `1` 等于 1 兆字节。允许的最小值为 `1`。这是一个基于信用的限制，现有的 `Http2Stream` 可能会导致超过此限制，但在超过此限制时，新的 `Http2Stream` 实例将被拒绝。当前的 `Http2Stream` 会话数、头部压缩表的当前内存使用情况、排队等待发送的当前数据以及未确认的 `PING` 和 `SETTINGS` 帧都计入当前限制。**默认值：** `10`。
  * `maxHeaderListPairs` {number} 设置最大头部条目数。这类似于 `node:http` 模块中的 [`server.maxHeadersCount`][] 或 [`request.maxHeadersCount`][]。最小值为 `4`。**默认值：** `128`。
  * `maxOutstandingPings` {number} 设置未完成的、未确认的 ping 的最大数量。**默认值：** `10`。
  * `maxSendHeaderBlockLength` {number} 设置序列化、压缩头部块的最大允许大小。尝试发送超过此限制的头部将导致发出 `'frameError'` 事件，并且流将被关闭和销毁。虽然这设置了整个头部块的最大允许大小，但 `nghttp2`（内部 http2 库）对每个解压缩的键/值对有限制 `65536`。
  * `paddingStrategy` {number} 用于确定 `HEADERS` 和 `DATA` 帧填充量的策略。**默认值：** `http2.constants.PADDING_STRATEGY_NONE`。值可以是以下之一：
    * `http2.constants.PADDING_STRATEGY_NONE`：不应用填充。
    * `http2.constants.PADDING_STRATEGY_MAX`：应用由内部实现确定的最大填充量。
    * `http2.constants.PADDING_STRATEGY_ALIGNED`：尝试应用足够的填充以确保总帧长度（包括 9 字节头部）是 8 的倍数。对于每个帧，有一个最大允许的填充字节数，由当前流控制状态和设置确定。如果此最大值小于确保对齐所需的计算量，则使用最大值，并且总帧长度不一定在 8 字节处对齐。
  * `peerMaxConcurrentStreams` {number} 设置远程对等端的最大并发流数，就像收到了 `SETTINGS` 帧一样。如果远程对等端设置了自己的 `maxConcurrentStreams` 值，将被覆盖。**默认值：** `100`。
  * `maxSessionInvalidFrames` {integer} 设置会话关闭之前将容忍的无效帧的最大数量。**默认值：** `1000`。
  * `maxSessionRejectedStreams` {integer} 设置会话关闭之前将容忍的创建时被拒绝的流的最大数量。每个拒绝都与一个 `NGHTTP2_ENHANCE_YOUR_CALM` 错误相关联，该错误应告诉对等端不要打开更多流，因此继续打开流被视为对等端行为不当的迹象。**默认值：** `100`。
  * `settings` {HTTP/2 Settings Object} 连接时发送给远程对等端的初始设置。
  * `streamResetBurst` {number} 和 `streamResetRate` {number} 设置传入流重置（RST\_STREAM 帧）的速率限制。必须同时设置这两个设置才能生效，默认值分别为 1000 和 33。
  * `remoteCustomSettings` {Array} 整数数组确定包含在接收到的 remoteSettings 的 `CustomSettings` 属性中的设置类型。有关允许的设置类型的更多信息，请参阅 `Http2Settings` 对象的 `CustomSettings` 属性。
  * `Http1IncomingMessage` {http.IncomingMessage} 指定用于 HTTP/1 回退的 `IncomingMessage` 类。用于扩展原始的 `http.IncomingMessage`。**默认值：** `http.IncomingMessage`。
  * `Http1ServerResponse` {http.ServerResponse} 指定用于 HTTP/1 回退的 `ServerResponse` 类。用于扩展原始的 `http.ServerResponse`。**默认值：** `http.ServerResponse`。
  * `Http2ServerRequest` {http2.Http2ServerRequest} 指定要使用的 `Http2ServerRequest` 类。用于扩展原始的 `Http2ServerRequest`。**默认值：** `Http2ServerRequest`。
  * `Http2ServerResponse` {http2.Http2ServerResponse} 指定要使用的 `Http2ServerResponse` 类。用于扩展原始的 `Http2ServerResponse`。**默认值：** `Http2ServerResponse`。
  * `unknownProtocolTimeout` {number} 指定服务器在发出 [`'unknownProtocol'`][] 时应等待的毫秒超时时间。如果套接字在该时间之前未被销毁，服务器将销毁它。**默认值：** `10000`。
  * `strictFieldWhitespaceValidation` {boolean} 如果为 `true`，则根据 [RFC-9113](https://www.rfc-editor.org/rfc/rfc9113.html#section-8.2.1) 对 HTTP/2 头部字段名称和值开启严格的前导和尾随空白验证。**默认值：** `true`。
  * `...options` {Object} 可以提供任何 [`net.createServer()`][] 选项。
* `onRequestHandler` {Function} 请参阅 [兼容性 API][]
* 返回：{Http2Server}

返回一个 `net.Server` 实例，该实例创建并管理 `Http2Session` 实例。

由于没有已知的浏览器支持[未加密的 HTTP/2][HTTP/2 Unencrypted]，因此与浏览器客户端通信时，需要使用 [`http2.createSecureServer()`][]。

```mjs
import { createServer } from 'node:http2';

// 创建一个未加密的 HTTP/2 服务器。
// 由于没有已知的浏览器支持未加密的 HTTP/2，因此与浏览器客户端通信时，需要使用 `createSecureServer()`。
const server = createServer();

server.on('stream', (stream, headers) => {
  stream.respond({
    'content-type': 'text/html; charset=utf-8',
    ':status': 200,
  });
  stream.end('<h1>Hello World</h1>');
});

server.listen(8000);
```

```cjs
const http2 = require('node:http2');

// 创建一个未加密的 HTTP/2 服务器。
// 由于没有已知的浏览器支持未加密的 HTTP/2，因此与浏览器客户端通信时，需要使用 `http2.createSecureServer()`。
const server = http2.createServer();

server.on('stream', (stream, headers) => {
  stream.respond({
    'content-type': 'text/html; charset=utf-8',
    ':status': 200,
  });
  stream.end('<h1>Hello World</h1>');
});

server.listen(8000);
```

### `http2.createSecureServer(options[, onRequestHandler])`

<!-- YAML
added: v8.4.0
changes:
  - version:
      - v15.10.0
      - v14.16.0
      - v12.21.0
      - v10.24.0
    pr-url: https://github.com/nodejs-private/node-private/pull/246
    description: Added `unknownProtocolTimeout` option with a default of 10000.
  - version:
     - v14.4.0
     - v12.18.0
     - v10.21.0
    commit: 3948830ce6408be620b09a70bf66158623022af0
    pr-url: https://github.com/nodejs-private/node-private/pull/204
    description: Added `maxSettings` option with a default of 32.
  - version:
     - v13.3.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30534
    description: Added `maxSessionRejectedStreams` option with a default of 100.
  - version:
     - v13.3.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30534
    description: Added `maxSessionInvalidFrames` option with a default of 1000.
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/29144
    description: The `PADDING_STRATEGY_CALLBACK` has been made equivalent to
                 providing `PADDING_STRATEGY_ALIGNED` and `selectPadding`
                 has been removed.
  - version: v10.12.0
    pr-url: https://github.com/nodejs/node/pull/22956
    description: Added the `origins` option to automatically send an `ORIGIN`
                 frame on `Http2Session` startup.
  - version: v8.9.3
    pr-url: https://github.com/nodejs/node/pull/17105
    description: Added the `maxOutstandingPings` option with a default limit of
                 10.
  - version: v8.9.3
    pr-url: https://github.com/nodejs/node/pull/16676
    description: Added the `maxHeaderListPairs` option with a default limit of
                 128 header pairs.
-->

* `options` {Object}
  * `allowHTTP1` {boolean} 当设置为 `true` 时，不支持 HTTP/2 的传入客户端连接将降级为 HTTP/1.x。请参阅 [`'unknownProtocol'`][] 事件。请参阅 [ALPN 协商][]。**默认值：** `false`。
  * `maxDeflateDynamicTableSize` {number} 设置用于压缩头部字段的最大动态表大小。**默认值：** `4Kib`。
  * `maxSettings` {number} 设置每个 `SETTINGS` 帧的最大设置条目数。允许的最小值为 `1`。**默认值：** `32`。
  * `maxSessionMemory`{number} 设置 `Http2Session` 允许使用的最大内存。该值以兆字节数表示，例如 `1` 等于 1 兆字节。允许的最小值为 `1`。这是一个基于信用的限制，现有的 `Http2Stream` 可能会导致超过此限制，但在超过此限制时，新的 `Http2Stream` 实例将被拒绝。当前的 `Http2Stream` 会话数、头部压缩表的当前内存使用情况、排队等待发送的当前数据以及未确认的 `PING` 和 `SETTINGS` 帧都计入当前限制。**默认值：** `10`。
  * `maxHeaderListPairs` {number} 设置最大头部条目数。这类似于 `node:http` 模块中的 [`server.maxHeadersCount`][] 或 [`request.maxHeadersCount`][]。最小值为 `4`。**默认值：** `128`。
  * `maxOutstandingPings` {number} 设置未完成的、未确认的 ping 的最大数量。**默认值：** `10`。
  * `maxSendHeaderBlockLength` {number} 设置序列化、压缩头部块的最大允许大小。尝试发送超过此限制的头部将导致发出 `'frameError'` 事件，并且流将被关闭和销毁。
  * `paddingStrategy` {number} 用于确定 `HEADERS` 和 `DATA` 帧填充量的策略。**默认值：** `http2.constants.PADDING_STRATEGY_NONE`。值可以是以下之一：
    * `http2.constants.PADDING_STRATEGY_NONE`：不应用填充。
    * `http2.constants.PADDING_STRATEGY_MAX`：应用由内部实现确定的最大填充量。
    * `http2.constants.PADDING_STRATEGY_ALIGNED`：尝试应用足够的填充以确保总帧长度（包括 9 字节头部）是 8 的倍数。对于每个帧，有一个最大允许的填充字节数，由当前流控制状态和设置确定。如果此最大值小于确保对齐所需的计算量，则使用最大值，并且总帧长度不一定在 8 字节处对齐。
  * `peerMaxConcurrentStreams` {number} 设置远程对等端的最大并发流数，就像收到了 `SETTINGS` 帧一样。如果远程对等端设置了自己的 `maxConcurrentStreams` 值，将被覆盖。**默认值：** `100`。
  * `maxSessionInvalidFrames` {integer} 设置会话关闭之前将容忍的无效帧的最大数量。**默认值：** `1000`。
  * `maxSessionRejectedStreams` {integer} 设置会话关闭之前将容忍的创建时被拒绝的流的最大数量。每个拒绝都与一个 `NGHTTP2_ENHANCE_YOUR_CALM` 错误相关联，该错误应告诉对等端不要打开更多流，因此继续打开流被视为对等端行为不当的迹象。**默认值：** `100`。
  * `settings` {HTTP/2 Settings Object} 连接时发送给远程对等端的初始设置。
  * `streamResetBurst` {number} 和 `streamResetRate` {number} 设置传入流重置（RST\_STREAM 帧）的速率限制。必须同时设置这两个设置才能生效，默认值分别为 1000 和 33。
  * `remoteCustomSettings` {Array} 整数数组确定包含在接收到的 remoteSettings 的 `customSettings` 属性中的设置类型。有关允许的设置类型的更多信息，请参阅 `Http2Settings` 对象的 `customSettings` 属性。
  * `...options` {Object} 可以提供任何 [`tls.createServer()`][] 选项。对于服务器，通常需要身份选项（`pfx` 或 `key`/`cert`）。
  * `origins` {string\[]} 在新服务器 `Http2Session` 创建后立即在 `ORIGIN` 帧内发送的来源字符串数组。
  * `unknownProtocolTimeout` {number} 指定服务器在发出 [`'unknownProtocol'`][] 事件时应等待的毫秒超时时间。如果套接字在该时间之前未被销毁，服务器将销毁它。**默认值：** `10000`。
  * `strictFieldWhitespaceValidation` {boolean} 如果为 `true`，则根据 [RFC-9113](https://www.rfc-editor.org/rfc/rfc9113.html#section-8.2.1) 对 HTTP/2 头部字段名称和值开启严格的前导和尾随空白验证。**默认值：** `true`。
* `onRequestHandler` {Function} 请参阅 [兼容性 API][]
* 返回：{Http2SecureServer}

返回一个 `tls.Server` 实例，该实例创建并管理 `Http2Session` 实例。

```mjs
import { createSecureServer } from 'node:http2';
import { readFileSync } from 'node:fs';

const options = {
  key: readFileSync('server-key.pem'),
  cert: readFileSync('server-cert.pem'),
};

// 创建一个安全的 HTTP/2 服务器
const server = createSecureServer(options);

server.on('stream', (stream, headers) => {
  stream.respond({
    'content-type': 'text/html; charset=utf-8',
    ':status': 200,
  });
  stream.end('<h1>Hello World</h1>');
});

server.listen(8443);
```

```cjs
const http2 = require('node:http2');
const fs = require('node:fs');

const options = {
  key: fs.readFileSync('server-key.pem'),
  cert: fs.readFileSync('server-cert.pem'),
};

// 创建一个安全的 HTTP/2 服务器
const server = http2.createSecureServer(options);

server.on('stream', (stream, headers) => {
  stream.respond({
    'content-type': 'text/html; charset=utf-8',
    ':status': 200,
  });
  stream.end('<h1>Hello World</h1>');
});

server.listen(8443);
```

### `http2.connect(authority[, options][, listener])`

<!-- YAML
added: v8.4.0
changes:
  - version:
      - v15.10.0
      - v14.16.0
      - v12.21.0
      - v10.24.0
    pr-url: https://github.com/nodejs-private/node-private/pull/246
    description: Added `unknownProtocolTimeout` option with a default of 10000.
  - version:
     - v14.4.0
     - v12.18.0
     - v10.21.0
    commit: 3948830ce6408be620b09a70bf66158623022af0
    pr-url: https://github.com/nodejs-private/node-private/pull/204
    description: Added `maxSettings` option with a default of 32.
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/29144
    description: The `PADDING_STRATEGY_CALLBACK` has been made equivalent to
                 providing `PADDING_STRATEGY_ALIGNED` and `selectPadding`
                 has been removed.
  - version: v8.9.3
    pr-url: https://github.com/nodejs/node/pull/17105
    description: Added the `maxOutstandingPings` option with a default limit of
                 10.
  - version: v8.9.3
    pr-url: https://github.com/nodejs/node/pull/16676
    description: Added the `maxHeaderListPairs` option with a default limit of
                 128 header pairs.
-->

* `authority` {string|URL} 要连接的远程 HTTP/2 服务器。这必须是形式为最小、有效的 URL，带有 `http://` 或 `https://` 前缀、主机名和 IP 端口（如果使用非默认端口）。URL 中的用户信息（用户 ID 和密码）、路径、查询字符串和片段详细信息将被忽略。
* `options` {Object}
  * `maxDeflateDynamicTableSize` {number} 设置用于压缩头部字段的最大动态表大小。**默认值：** `4Kib`。
  * `maxSettings` {number} 设置每个 `SETTINGS` 帧的最大设置条目数。允许的最小值为 `1`。**默认值：** `32`。
  * `maxSessionMemory`{number} 设置 `Http2Session` 允许使用的最大内存。该值以兆字节数表示，例如 `1` 等于 1 兆字节。允许的最小值为 `1`。这是一个基于信用的限制，现有的 `Http2Stream` 可能会导致超过此限制，但在超过此限制时，新的 `Http2Stream` 实例将被拒绝。当前的 `Http2Stream` 会话数、头部压缩表的当前内存使用情况、排队等待发送的当前数据以及未确认的 `PING` 和 `SETTINGS` 帧都计入当前限制。**默认值：** `10`。
  * `maxHeaderListPairs` {number} 设置最大头部条目数。这类似于 `node:http` 模块中的 [`server.maxHeadersCount`][] 或 [`request.maxHeadersCount`][]。最小值为 `1`。**默认值：** `128`。
  * `maxOutstandingPings` {number} 设置未完成的、未确认的 ping 的最大数量。**默认值：** `10`。
  * `maxReservedRemoteStreams` {number} 设置客户端在任何给定时间将接受的保留推送流的最大数量。一旦当前当前保留的推送流数量达到此限制，服务器发送的新推送流将自动被拒绝。允许的最小值为 0。允许的最大值为 2<sup>32</sup>-1。负值将此选项设置为允许的最大值。**默认值：** `200`。
  * `maxSendHeaderBlockLength` {number} 设置序列化、压缩头部块的最大允许大小。尝试发送超过此限制的头部将导致发出 `'frameError'` 事件，并且流将被关闭和销毁。
  * `paddingStrategy` {number} 用于确定 `HEADERS` 和 `DATA` 帧填充量的策略。**默认值：** `http2.constants.PADDING_STRATEGY_NONE`。值可以是以下之一：
    * `http2.constants.PADDING_STRATEGY_NONE`：不应用填充。
    * `http2.constants.PADDING_STRATEGY_MAX`：应用由内部实现确定的最大填充量。
    * `http2.constants.PADDING_STRATEGY_ALIGNED`：尝试应用足够的填充以确保总帧长度（包括 9 字节头部）是 8 的倍数。对于每个帧，有一个最大允许的填充字节数，由当前流控制状态和设置确定。如果此最大值小于确保对齐所需的计算量，则使用最大值，并且总帧长度不一定在 8 字节处对齐。
  * `peerMaxConcurrentStreams` {number} 设置远程对等端的最大并发流数，就像收到了 `SETTINGS` 帧一样。如果远程对等端设置了自己的 `maxConcurrentStreams` 值，将被覆盖。**默认值：** `100`。
  * `protocol` {string} 连接时使用的协议，如果未在 `authority` 中设置。值可以是 `'http:'` 或 `'https:'`。**默认值：** `'https:'`
  * `settings` {HTTP/2 Settings Object} 连接时发送给远程对等端的初始设置。
  * `remoteCustomSettings` {Array} 整数数组确定包含在接收到的 remoteSettings 的 `CustomSettings` 属性中的设置类型。有关允许的设置类型的更多信息，请参阅 `Http2Settings` 对象的 `CustomSettings` 属性。
  * `createConnection` {Function} 一个可选的回调，接收传递给 `connect` 的 `URL` 实例和 `options` 对象，并返回任何要用作此会话连接的 [`Duplex`][] 流。
  * `...options` {Object} 可以提供任何 [`net.connect()`][] 或 [`tls.connect()`][] 选项。
  * `unknownProtocolTimeout` {number} 指定服务器在发出 [`'unknownProtocol'`][] 事件时应等待的毫秒超时时间。如果套接字在该时间之前未被销毁，服务器将销毁它。**默认值：** `10000`。
  * `strictFieldWhitespaceValidation` {boolean} 如果为 `true`，则根据 [RFC-9113](https://www.rfc-editor.org/rfc/rfc9113.html#section-8.2.1) 对 HTTP/2 头部字段名称和值开启严格的前导和尾随空白验证。**默认值：** `true`。
* `listener` {Function} 将被注册为 [`'connect'`][] 事件的一次性监听器。
* 返回：{ClientHttp2Session}

返回一个 `ClientHttp2Session` 实例。

```mjs
import { connect } from 'node:http2';
const client = connect('https://localhost:1234');

/* 使用客户端 */

client.close();
```

```cjs
const http2 = require('node:http2');
const client = http2.connect('https://localhost:1234');

/* 使用客户端 */

client.close();
```

### `http2.constants`

<!-- YAML
added: v8.4.0
-->

#### `RST_STREAM` 和 `GOAWAY` 的错误代码

| 值  | 名称                | 常量                                      |
| ------ | ------------------- | --------------------------------------------- |
| `0x00` | No Error            | `http2.constants.NGHTTP2_NO_ERROR`            |
| `0x01` | Protocol Error      | `http2.constants.NGHTTP2_PROTOCOL_ERROR`      |
| `0x02` | Internal Error      | `http2.constants.NGHTTP2_INTERNAL_ERROR`      |
| `0x03` | Flow Control Error  | `http2.constants.NGHTTP2_FLOW_CONTROL_ERROR`  |
| `0x04` | Settings Timeout    | `http2.constants.NGHTTP2_SETTINGS_TIMEOUT`    |
| `0x05` | Stream Closed       | `http2.constants.NGHTTP2_STREAM_CLOSED`       |
| `0x06` | Frame Size Error    | `http2.constants.NGHTTP2_FRAME_SIZE_ERROR`    |
| `0x07` | Refused Stream      | `http2.constants.NGHTTP2_REFUSED_STREAM`      |
| `0x08` | Cancel              | `http2.constants.NGHTTP2_CANCEL`              |
| `0x09` | Compression Error   | `http2.constants.NGHTTP2_COMPRESSION_ERROR`   |
| `0x0a` | Connect Error       | `http2.constants.NGHTTP2_CONNECT_ERROR`       |
| `0x0b` | Enhance Your Calm   | `http2.constants.NGHTTP2_ENHANCE_YOUR_CALM`   |
| `0x0c` | Inadequate Security | `http2.constants.NGHTTP2_INADEQUATE_SECURITY` |
| `0x0d` | HTTP/1.1 Required   | `http2.constants.NGHTTP2_HTTP_1_1_REQUIRED`   |

当服务器在给定毫秒数内没有活动时，会发出 `'timeout'` 事件，该时间使用 `http2server.setTimeout()` 设置。

### `http2.getDefaultSettings()`

<!-- YAML
added: v8.4.0
-->

* 返回：{HTTP/2 Settings Object}

返回一个包含 `Http2Session` 实例默认设置的对象。此方法每次调用时都会返回一个新的对象实例，因此返回的实例可以安全地修改以供使用。

### `http2.getPackedSettings([settings])`

<!-- YAML
added: v8.4.0
-->

* `settings` {HTTP/2 Settings Object}
* 返回：{Buffer}

返回一个 `Buffer` 实例，其中包含 [HTTP/2][] 规范中指定的给定 HTTP/2 设置的序列化表示。这旨在与 `HTTP2-Settings` 头部字段一起使用。

```mjs
import { getPackedSettings } from 'node:http2';

const packed = getPackedSettings({ enablePush: false });

console.log(packed.toString('base64'));
// 打印: AAIAAAAA
```

```cjs
const http2 = require('node:http2');

const packed = http2.getPackedSettings({ enablePush: false });

console.log(packed.toString('base64'));
// 打印: AAIAAAAA
```

### `http2.getUnpackedSettings(buf)`

<!-- YAML
added: v8.4.0
-->

* `buf` {Buffer|TypedArray} 打包的设置。
* 返回：{HTTP/2 Settings Object}

返回一个 [HTTP/2 设置对象][]，其中包含从 `http2.getPackedSettings()` 生成的给定 `Buffer` 反序列化的设置。

### `http2.performServerHandshake(socket[, options])`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

* `socket` {stream.Duplex}
* `options` {Object} 可以提供任何 [`http2.createServer()`][] 选项。
* 返回：{ServerHttp2Session}

从现有套接字创建 HTTP/2 服务器会话。

### `http2.sensitiveHeaders`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

* 类型：{symbol}

此符号可以设置为 HTTP/2 头部对象上的属性，其值为数组，以提供被视为敏感的头部列表。有关更多详细信息，请参阅 [敏感头部][]。

### 头部对象

头部表示为 JavaScript 对象上的自有属性。属性键将被序列化为小写。属性值应为字符串（如果不是，将被强制转换为字符串）或字符串的 `Array`（以便为每个头部字段发送多个值）。

```js
const headers = {
  ':status': '200',
  'content-type': 'text-plain',
  'ABC': ['has', 'more', 'than', 'one', 'value'],
};

stream.respond(headers);
```

传递给回调函数的头部对象将具有 `null` 原型。这意味着正常的 JavaScript 对象方法，如 `Object.prototype.toString()` 和 `Object.prototype.hasOwnProperty()` 将不起作用。

对于传入的头部：

* `:status` 头部转换为 `number`。
* `:status`、`:method`、`:authority`、`:scheme`、`:path`、`:protocol`、`age`、`authorization`、`access-control-allow-credentials`、`access-control-max-age`、`access-control-request-method`、`content-encoding`、`content-language`、`content-length`、`content-location`、`content-md5`、`content-range`、`content-type`、`date`、`dnt`、`etag`、`expires`、`from`、`host`、`if-match`、`if-modified-since`、`if-none-match`、`if-range`、`if-unmodified-since`、`last-modified`、`location`、`max-forwards`、`proxy-authorization`、`range`、`referer`、`retry-after`、`tk`、`upgrade-insecure-requests`、`user-agent` 或 `x-content-type-options` 的重复项将被丢弃。
* `set-cookie` 始终是一个数组。重复项将添加到数组中。
* 对于重复的 `cookie` 头部，值使用 '; ' 连接。
* 对于所有其他头部，值使用 ', ' 连接。

```mjs
import { createServer } from 'node:http2';
const server = createServer();
server.on('stream', (stream, headers) => {
  console.log(headers[':path']);
  console.log(headers.ABC);
});
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer();
server.on('stream', (stream, headers) => {
  console.log(headers[':path']);
  console.log(headers.ABC);
});
```

#### 原始头部

在某些 API 中，除了对象格式外，头部也可以作为原始平面数组传递或访问，保留排序和重复键的详细信息以匹配原始传输格式。

在此格式中，键和值在同一列表中。它*不是*元组列表。因此，偶数偏移是键值，奇数偏移是关联的值。重复的头部不会合并，因此每个键值对将单独出现。

这对于代理等情况很有用，其中现有头部应完全按原样转发，或者当头部已经以原始格式可用时作为性能优化。

```js
const rawHeaders = [
  ':status',
  '404',
  'content-type',
  'text/plain',
];

stream.respond(rawHeaders);
```

#### 敏感头部

HTTP2 头部可以标记为敏感，这意味着 HTTP/2 头部压缩算法永远不会索引它们。这对于具有低熵且可能被视为对攻击者有价值的头部值是有意义的，例如 `Cookie` 或 `Authorization`。为此，将头部名称作为数组添加到 `[http2.sensitiveHeaders]` 属性：

```js
const headers = {
  ':status': '200',
  'content-type': 'text-plain',
  'cookie': 'some-cookie',
  'other-sensitive-header': 'very secret data',
  [http2.sensitiveHeaders]: ['cookie', 'other-sensitive-header'],
};

stream.respond(headers);
```

对于某些头部，例如 `Authorization` 和短的 `Cookie` 头部，此标志会自动设置。

此属性也为接收到的头部设置。它将包含所有标记为敏感的头部名称，包括自动标记的头部。

对于原始头部，这仍应设置为数组上的属性，如 `rawHeadersArray[http2.sensitiveHeaders] = ['cookie']`，而不是作为数组内的单独键值对。

### 设置对象

<!-- YAML
added: v8.4.0
changes:
  - version: v12.12.0
    pr-url: https://github.com/nodejs/node/pull/29833
    description: The `maxConcurrentStreams` setting is stricter.
  - version: v8.9.3
    pr-url: https://github.com/nodejs/node/pull/16676
    description: The `maxHeaderListSize` setting is now strictly enforced.
-->

`http2.getDefaultSettings()`、`http2.getPackedSettings()`、`http2.createServer()`、`http2.createSecureServer()`、`http2session.settings()`、`http2session.localSettings` 和 `http2session.remoteSettings` API 要么返回要么接收作为输入一个对象，该对象定义 `Http2Session` 对象的配置设置。这些对象是包含以下属性的普通 JavaScript 对象。

* `headerTableSize` {number} 指定用于头部压缩的最大字节数。允许的最小值为 0。允许的最大值为 2<sup>32</sup>-1。**默认值：** `4096`。
* `enablePush` {boolean} 指定是否允许在 `Http2Session` 实例上使用 HTTP/2 推送流。**默认值：** `true`。
* `initialWindowSize` {number} 指定*发送方*的流级流控制初始窗口大小（字节）。允许的最小值为 0。允许的最大值为 2<sup>32</sup>-1。**默认值：** `65535`。
* `maxFrameSize` {number} 指定最大帧载荷大小（字节）。允许的最小值为 16384。允许的最大值为 2<sup>24</sup>-1。**默认值：** `16384`。
* `maxConcurrentStreams` {number} 指定 `Http2Session` 上允许的最大并发流数。没有默认值，这意味着理论上在 `Http2Session` 中的任何给定时间最多可以同时打开 2<sup>32</sup>-1 个流。最小值为 0。允许的最大值为 2<sup>32</sup>-1。**默认值：** `4294967295`。
* `maxHeaderListSize` {number} 指定将接受的头部列表的最大大小（未压缩的八位字节）。允许的最小值为 0。允许的最大值为 2<sup>32</sup>-1。**默认值：** `65535`。
* `maxHeaderSize` {number} `maxHeaderListSize` 的别名。
* `enableConnectProtocol`{boolean} 指定是否启用由 [RFC 8441][] 定义的“扩展连接协议”。此设置仅由服务器发送时才有意义。一旦为给定的 `Http2Session` 启用了 `enableConnectProtocol` 设置，它就不能被禁用。**默认值：** `false`。
* `customSettings` {Object} 指定 node 和底层库中尚未实现的附加设置。对象的键定义设置类型的数值（由 [RFC 7540] 建立的“HTTP/2 SETTINGS”注册表定义），值是该设置的实际数值。设置类型必须是 1 到 2^16-1 范围内的整数。它不应该是 node 已经处理的设置类型，即目前它应该大于 6，尽管这不是错误。值需要是 0 到 2^32-1 范围内的无符号整数。目前，最多支持 10 个自定义设置。它仅支持发送 SETTINGS，或用于接收在服务器或客户端对象的 `remoteCustomSettings` 选项中指定的设置值。不要将设置 ID 的 `customSettings` 机制与本机处理的设置接口混合使用，以防设置在未来的 node 版本中变得原生支持。

设置对象上的所有其他属性将被忽略。

### 错误处理

在使用 `node:http2` 模块时可能会出现几种类型的错误条件：

验证错误在传入不正确的参数、选项或设置值时发生。这些将始终通过同步 `throw` 报告。

状态错误在尝试在不正确的时间执行操作时发生（例如，在流关闭后尝试向其发送数据）。这些将使用同步 `throw` 或通过 `Http2Stream`、`Http2Session` 或 HTTP/2 服务器对象上的 `'error'` 事件报告，具体取决于错误发生的地点和时间。

内部错误在 HTTP/2 会话意外失败时发生。这些将通过 `Http2Session` 或 HTTP/2 服务器对象上的 `'error'` 事件报告。

协议错误在违反各种 HTTP/2 协议约束时发生。这些将使用同步 `throw` 或通过 `Http2Stream`、`Http2Session` 或 HTTP/2 服务器对象上的 `'error'` 事件报告，具体取决于错误发生的地点和时间。

### 头部名称和值中的无效字符处理

HTTP/2 实现比 HTTP/1 实现更严格地处理 HTTP 头部名称和值中的无效字符。

头部字段名称是*不区分大小写*的，并且在线路上严格以小写字符串传输。Node.js 提供的 API 允许将头部名称设置为混合大小写字符串（例如 `Content-Type`），但会在传输时将其转换为小写（例如 `content-type`）。

头部字段名称*必须仅*包含以下一个或多个 ASCII 字符：`a`-`z`、`A`-`Z`、`0`-`9`、`!`、`#`、`$`、`%`、`&`、`'`、`*`、`+`、`-`、`.`、`^`、`_`、`` ` ``（反引号）、`|` 和 `~`。

在 HTTP 头部字段名称中使用无效字符将导致流关闭并报告协议错误。

头部字段值的处理更为宽松，但*不应*包含换行或回车字符，并且*应*限于 US-ASCII 字符，根据 HTTP 规范的要求。

### 客户端的推送流

要在客户端接收推送流，请在 `ClientHttp2Session` 上为 `'stream'` 事件设置监听器：

```mjs
import { connect } from 'node:http2';

const client = connect('http://localhost');

client.on('stream', (pushedStream, requestHeaders) => {
  pushedStream.on('push', (responseHeaders) => {
    // 处理响应头部
  });
  pushedStream.on('data', (chunk) => { /* 处理推送的数据 */ });
});

const req = client.request({ ':path': '/' });
```

```cjs
const http2 = require('node:http2');

const client = http2.connect('http://localhost');

client.on('stream', (pushedStream, requestHeaders) => {
  pushedStream.on('push', (responseHeaders) => {
    // 处理响应头部
  });
  pushedStream.on('data', (chunk) => { /* 处理推送的数据 */ });
});

const req = client.request({ ':path': '/' });
```

### 支持 `CONNECT` 方法

`CONNECT` 方法用于允许 HTTP/2 服务器用作 TCP/IP 连接的代理。

一个简单的 TCP 服务器：

```mjs
import { createServer } from 'node:net';

const server = createServer((socket) => {
  let name = '';
  socket.setEncoding('utf8');
  socket.on('data', (chunk) => name += chunk);
  socket.on('end', () => socket.end(`hello ${name}`));
});

server.listen(8000);
```

```cjs
const net = require('node:net');

const server = net.createServer((socket) => {
  let name = '';
  socket.setEncoding('utf8');
  socket.on('data', (chunk) => name += chunk);
  socket.on('end', () => socket.end(`hello ${name}`));
});

server.listen(8000);
```

一个 HTTP/2 CONNECT 代理：

```mjs
import { createServer, constants } from 'node:http2';
const { NGHTTP2_REFUSED_STREAM, NGHTTP2_CONNECT_ERROR } = constants;
import { connect } from 'node:net';

const proxy = createServer();
proxy.on('stream', (stream, headers) => {
  if (headers[':method'] !== 'CONNECT') {
    // 仅接受 CONNECT 请求
    stream.close(NGHTTP2_REFUSED_STREAM);
    return;
  }
  const auth = new URL(`tcp://${headers[':authority']}`);
  // 验证主机名和端口是该代理应连接的内容是一个非常好的主意。
  const socket = connect(auth.port, auth.hostname, () => {
    stream.respond();
    socket.pipe(stream);
    stream.pipe(socket);
  });
  socket.on('error', (error) => {
    stream.close(NGHTTP2_CONNECT_ERROR);
  });
});

proxy.listen(8001);
```

```cjs
const http2 = require('node:http2');
const { NGHTTP2_REFUSED_STREAM } = http2.constants;
const net = require('node:net');

const proxy = http2.createServer();
proxy.on('stream', (stream, headers) => {
  if (headers[':method'] !== 'CONNECT') {
    // 仅接受 CONNECT 请求
    stream.close(NGHTTP2_REFUSED_STREAM);
    return;
  }
  const auth = new URL(`tcp://${headers[':authority']}`);
  // 验证主机名和端口是该代理应连接的内容是一个非常好的主意。
  const socket = net.connect(auth.port, auth.hostname, () => {
    stream.respond();
    socket.pipe(stream);
    stream.pipe(socket);
  });
  socket.on('error', (error) => {
    stream.close(http2.constants.NGHTTP2_CONNECT_ERROR);
  });
});

proxy.listen(8001);
```

一个 HTTP/2 CONNECT 客户端：

```mjs
import { connect, constants } from 'node:http2';

const client = connect('http://localhost:8001');

// 不得为 CONNECT 请求指定 ':path' 和 ':scheme' 头部，否则将抛出错误。
const req = client.request({
  ':method': 'CONNECT',
  ':authority': 'localhost:8000',
});

req.on('response', (headers) => {
  console.log(headers[constants.HTTP2_HEADER_STATUS]);
});
let data = '';
req.setEncoding('utf8');
req.on('data', (chunk) => data += chunk);
req.on('end', () => {
  console.log(`The server says: ${data}`);
  client.close();
});
req.end('Jane');
```

```cjs
const http2 = require('node:http2');

const client = http2.connect('http://localhost:8001');

// 不得为 CONNECT 请求指定 ':path' 和 ':scheme' 头部，否则将抛出错误。
const req = client.request({
  ':method': 'CONNECT',
  ':authority': 'localhost:8000',
});

req.on('response', (headers) => {
  console.log(headers[http2.constants.HTTP2_HEADER_STATUS]);
});
let data = '';
req.setEncoding('utf8');
req.on('data', (chunk) => data += chunk);
req.on('end', () => {
  console.log(`The server says: ${data}`);
  client.close();
});
req.end('Jane');
```

### 扩展的 `CONNECT` 协议

[RFC 8441][] 定义了 HTTP/2 的“扩展连接协议”扩展，可用于引导使用 `CONNECT` 方法作为其他通信协议（如 WebSockets）隧道的 `Http2Stream` 的使用。

通过使用 `enableConnectProtocol` 设置，HTTP/2 服务器可以启用扩展连接协议：

```mjs
import { createServer } from 'node:http2';
const settings = { enableConnectProtocol: true };
const server = createServer({ settings });
```

```cjs
const http2 = require('node:http2');
const settings = { enableConnectProtocol: true };
const server = http2.createServer({ settings });
```

一旦客户端收到来自服务器的 `SETTINGS` 帧，指示可以使用扩展的 CONNECT，它可能会发送使用 `':protocol'` HTTP/2 伪头部的 `CONNECT` 请求：

```mjs
import { connect } from 'node:http2';
const client = connect('http://localhost:8080');
client.on('remoteSettings', (settings) => {
  if (settings.enableConnectProtocol) {
    const req = client.request({ ':method': 'CONNECT', ':protocol': 'foo' });
    // ...
  }
});
```

```cjs
const http2 = require('node:http2');
const client = http2.connect('http://localhost:8080');
client.on('remoteSettings', (settings) => {
  if (settings.enableConnectProtocol) {
    const req = client.request({ ':method': 'CONNECT', ':protocol': 'foo' });
    // ...
  }
});
```

## 兼容性 API

兼容性 API 的目标是在使用 HTTP/2 时提供与 HTTP/1 类似的开发者体验，使得可以开发同时支持 [HTTP/1][] 和 HTTP/2 的应用程序。此 API 仅针对 [HTTP/1][] 的**公共 API**。然而，许多模块使用内部方法或状态，而这些*不受支持*，因为它是一个完全不同的实现。

以下示例使用兼容性 API 创建 HTTP/2 服务器：

```mjs
import { createServer } from 'node:http2';
const server = createServer((req, res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('ok');
});
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer((req, res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('ok');
});
```

为了创建混合 [HTTPS][] 和 HTTP/2 服务器，请参阅 [ALPN 协商][] 部分。不支持从非 TLS HTTP/1 服务器升级。

HTTP/2 兼容性 API 由 [`Http2ServerRequest`][] 和 [`Http2ServerResponse`][] 组成。它们旨在与 HTTP/1 实现 API 兼容，但并未隐藏协议之间的差异。例如，HTTP 代码的状态消息被忽略。



### ALPN 协商

ALPN 协商允许在同一套接字上同时支持 [HTTPS][] 和 HTTP/2。`req` 和 `res` 对象可以是 HTTP/1 或 HTTP/2，应用程序**必须**将自己限制在 [HTTP/1][] 的公共 API 内，并检测是否可以使用 HTTP/2 的更高级特性。

以下示例创建一个支持两种协议的服务器：

```mjs
import { createSecureServer } from 'node:http2';
import { readFileSync } from 'node:fs';

const cert = readFileSync('./cert.pem');
const key = readFileSync('./key.pem');

const server = createSecureServer(
  { cert, key, allowHTTP1: true },
  onRequest,
).listen(8000);

function onRequest(req, res) {
  // 检测是 HTTPS 请求还是 HTTP/2
  const { socket: { alpnProtocol } } = req.httpVersion === '2.0' ?
    req.stream.session : req;
  res.writeHead(200, { 'content-type': 'application/json' });
  res.end(JSON.stringify({
    alpnProtocol,
    httpVersion: req.httpVersion,
  }));
}
```

```cjs
const { createSecureServer } = require('node:http2');
const { readFileSync } = require('node:fs');

const cert = readFileSync('./cert.pem');
const key = readFileSync('./key.pem');

const server = createSecureServer(
  { cert, key, allowHTTP1: true },
  onRequest,
).listen(4443);

function onRequest(req, res) {
  // 检测是 HTTPS 请求还是 HTTP/2
  const { socket: { alpnProtocol } } = req.httpVersion === '2.0' ?
    req.stream.session : req;
  res.writeHead(200, { 'content-type': 'application/json' });
  res.end(JSON.stringify({
    alpnProtocol,
    httpVersion: req.httpVersion,
  }));
}
```

`'request'` 事件在 [HTTPS][] 和 HTTP/2 上的工作方式相同。

### 类：`http2.Http2ServerRequest`

<!-- YAML
added: v8.4.0
-->

* 扩展：{stream.Readable}

`Http2ServerRequest` 对象由 [`http2.Server`][] 或 [`http2.SecureServer`][] 创建，并作为第一个参数传递给 [`'request'`][] 事件。它可用于访问请求状态、头部和数据。

#### 事件：`'aborted'`

<!-- YAML
added: v8.4.0
-->

每当 `Http2ServerRequest` 实例在通信过程中异常中止时，就会发出 `'aborted'` 事件。

仅当 `Http2ServerRequest` 的可写端尚未结束时，才会发出 `'aborted'` 事件。

#### 事件：`'close'`

<!-- YAML
added: v8.4.0
-->

表示底层的 [`Http2Stream`][] 已关闭。就像 `'end'` 一样，此事件每个响应只发生一次。

#### `request.aborted`

<!-- YAML
added: v10.1.0
-->

* 类型：{boolean}

如果请求已中止，则 `request.aborted` 属性将为 `true`。

#### `request.authority`

<!-- YAML
added: v8.4.0
-->

* 类型：{string}

请求授权伪头部字段。因为 HTTP/2 允许请求设置 `:authority` 或 `host`，所以此值从 `req.headers[':authority']`（如果存在）派生。否则，从 `req.headers['host']` 派生。

#### `request.complete`

<!-- YAML
added: v12.10.0
-->

* 类型：{boolean}

如果请求已完成、中止或销毁，则 `request.complete` 属性将为 `true`。

#### `request.connection`

<!-- YAML
added: v8.4.0
deprecated: v13.0.0
-->

> Stability: 0 - Deprecated. 使用 [`request.socket`][]。

* 类型：{net.Socket|tls.TLSSocket}

参见 [`request.socket`][]。

#### `request.destroy([error])`

<!-- YAML
added: v8.4.0
-->

* `error` {Error}

在接收 [`Http2ServerRequest`][] 的 [`Http2Stream`][] 上调用 `destroy()`。如果提供了 `error`，则会发出 `'error'` 事件，并将 `error` 作为参数传递给该事件上的任何监听器。

如果流已被销毁，则不执行任何操作。

#### `request.headers`

<!-- YAML
added: v8.4.0
-->

* 类型：{Object}

请求/响应头部对象。

头部名称和值的键值对。头部名称是小写的。

```js
// 打印类似以下内容：
//
// { 'user-agent': 'curl/7.22.0',
//   host: '127.0.0.1:8000',
//   accept: '*/*' }
console.log(request.headers);
```

参见 [HTTP/2 头部对象][]。

在 HTTP/2 中，请求路径、主机名、协议和方法表示为以 `:` 字符为前缀的特殊头部（例如 `':path'`）。这些特殊头部将包含在 `request.headers` 对象中。必须注意不要无意中修改这些特殊头部，否则可能发生错误。例如，从请求中删除所有头部将导致错误：

```js
removeAllHeaders(request.headers);
assert(request.url);   // 失败，因为 :path 头部已被删除
```

#### `request.httpVersion`

<!-- YAML
added: v8.4.0
-->

* 类型：{string}

对于服务器请求，是客户端发送的 HTTP 版本。对于客户端响应，是连接到的服务器的 HTTP 版本。返回 `'2.0'`。

另外 `message.httpVersionMajor` 是第一个整数，`message.httpVersionMinor` 是第二个。

#### `request.method`

<!-- YAML
added: v8.4.0
-->

* 类型：{string}

请求方法作为字符串。只读。示例：`'GET'`、`'DELETE'`。

#### `request.rawHeaders`

<!-- YAML
added: v8.4.0
-->

* 类型：{HTTP/2 Raw Headers}

原始请求/响应头部列表，完全按接收到的样子。

```js
// 打印类似以下内容：
//
// [ 'user-agent',
//   'this is invalid because there can be only one',
//   'User-Agent',
//   'curl/7.22.0',
//   'Host',
//   '127.0.0.1:8000',
//   'ACCEPT',
//   '*/*' ]
console.log(request.rawHeaders);
```

#### `request.rawTrailers`

<!-- YAML
added: v8.4.0
-->

* 类型：{string\[]}

原始请求/响应尾部键和值，完全按接收到的样子。仅在 `'end'` 事件时填充。

#### `request.scheme`

<!-- YAML
added: v8.4.0
-->

* 类型：{string}

请求方案伪头部字段，指示目标 URL 的方案部分。

#### `request.setTimeout(msecs, callback)`

<!-- YAML
added: v8.4.0
-->

* `msecs` {number}
* `callback` {Function}
* 返回：{http2.Http2ServerRequest}

将 [`Http2Stream`][] 的超时值设置为 `msecs`。如果提供了回调，则将其添加为响应对象上 `'timeout'` 事件的监听器。

如果未向请求、响应或服务器添加 `'timeout'` 监听器，则 [`Http2Stream`][] 在超时时将被销毁。如果为请求、响应或服务器的 `'timeout'` 事件分配了处理程序，则必须显式处理超时的套接字。

#### `request.socket`

<!-- YAML
added: v8.4.0
-->

* 类型：{net.Socket|tls.TLSSocket}

返回一个 `Proxy` 对象，其行为类似于 `net.Socket`（或 `tls.TLSSocket`），但基于 HTTP/2 逻辑应用 getter、setter 和方法。

`destroyed`、`readable` 和 `writable` 属性将从 `request.stream` 检索并设置到其上。

`destroy`、`emit`、`end`、`on` 和 `once` 方法将在 `request.stream` 上调用。

`setTimeout` 方法将在 `request.stream.session` 上调用。

`pause`、`read`、`resume` 和 `write` 将抛出错误代码为 `ERR_HTTP2_NO_SOCKET_MANIPULATION` 的错误。有关更多信息，请参阅 [`Http2Session` 和套接字][]。

所有其他交互将直接路由到套接字。对于 TLS 支持，使用 [`request.socket.getPeerCertificate()`][] 获取客户端的身份验证详细信息。

#### `request.stream`

<!-- YAML
added: v8.4.0
-->

* 类型：{Http2Stream}

支持请求的 [`Http2Stream`][] 对象。

#### `request.trailers`

<!-- YAML
added: v8.4.0
-->

* 类型：{Object}

请求/响应尾部对象。仅在 `'end'` 事件时填充。

#### `request.url`

<!-- YAML
added: v8.4.0
-->

* 类型：{string}

请求 URL 字符串。这仅包含实际 HTTP 请求中存在的 URL。如果请求是：

```http
GET /status?name=ryan HTTP/1.1
Accept: text/plain
```

那么 `request.url` 将是：

<!-- eslint-disable @stylistic/js/semi -->

```js
'/status?name=ryan'
```

要解析 url 为其部分，可以使用 `new URL()`：

```console
$ node
> new URL('/status?name=ryan', 'http://example.com')
URL {
  href: 'http://example.com/status?name=ryan',
  origin: 'http://example.com',
  protocol: 'http:',
  username: '',
  password: '',
  host: 'example.com',
  hostname: 'example.com',
  port: '',
  pathname: '/status',
  search: '?name=ryan',
  searchParams: URLSearchParams { 'name' => 'ryan' },
  hash: ''
}
```

### 类：`http2.Http2ServerResponse`

<!-- YAML
added: v8.4.0
-->

* 扩展：{Stream}

此对象由 HTTP 服务器内部创建，而不是由用户创建。它作为第二个参数传递给 [`'request'`][] 事件。

#### 事件：`'close'`

<!-- YAML
added: v8.4.0
-->

表示底层的 [`Http2Stream`][] 在 [`response.end()`][] 被调用或能够刷新之前已终止。

#### 事件：`'finish'`

<!-- YAML
added: v8.4.0
-->

当响应已发送时发出。更具体地说，当响应头部和主体的最后一段已交给 HTTP/2 多路复用以通过网络传输时，会发出此事件。这并不表示客户端已收到任何内容。

在此事件之后，响应对象上不会发出更多事件。

#### `response.addTrailers(headers)`

<!-- YAML
added: v8.4.0
-->

* `headers` {Object}

此方法向响应添加 HTTP 尾部头部（消息末尾的头部）。

尝试设置包含无效字符的头部字段名称或值将导致抛出 [`TypeError`][]。

#### `response.appendHeader(name, value)`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

* `name` {string}
* `value` {string|string\[]}

向头部对象追加单个头部值。

如果值是一个数组，这相当于多次调用此方法。

如果头部之前没有值，这相当于调用 [`response.setHeader()`][]。

尝试设置包含无效字符的头部字段名称或值将导致抛出 [`TypeError`][]。

```js
// 返回包括 "set-cookie: a" 和 "set-cookie: b" 的头部
const server = http2.createServer((req, res) => {
  res.setHeader('set-cookie', 'a');
  res.appendHeader('set-cookie', 'b');
  res.writeHead(200);
  res.end('ok');
});
```

#### `response.connection`

<!-- YAML
added: v8.4.0
deprecated: v13.0.0
-->

> Stability: 0 - Deprecated. 使用 [`response.socket`][]。

* 类型：{net.Socket|tls.TLSSocket}

参见 [`response.socket`][]。

#### `response.createPushResponse(headers, callback)`

<!-- YAML
added: v8.4.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `headers` {HTTP/2 Headers Object} 描述头部的对象
* `callback` {Function} 在 `http2stream.pushStream()` 完成时调用，或者在尝试创建推送的 `Http2Stream` 失败或已被拒绝时，或者在调用 `http2stream.pushStream()` 方法之前 `Http2ServerRequest` 的状态已关闭时调用
  * `err` {Error}
  * `res` {http2.Http2ServerResponse} 新创建的 `Http2ServerResponse` 对象

使用给定的头部调用 [`http2stream.pushStream()`][]，并在成功时将给定的 [`Http2Stream`][] 包装在新创建的 `Http2ServerResponse` 上作为回调参数。当 `Http2ServerRequest` 关闭时，回调将以错误 `ERR_HTTP2_INVALID_STREAM` 调用。

#### `response.end([data[, encoding]][, callback])`

<!-- YAML
added: v8.4.0
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18780
    description: This method now returns a reference to `ServerResponse`.
-->

* `data` {string|Buffer|Uint8Array}
* `encoding` {string}
* `callback` {Function}
* 返回：{this}

此方法向服务器发出信号，表示所有响应头部和主体已发送；服务器应认为此消息已完成。方法 `response.end()` 必须在每个响应上调用。

如果指定了 `data`，则相当于调用 [`response.write(data, encoding)`][] 后跟 `response.end(callback)`。

如果指定了 `callback`，它将在响应流完成时被调用。

#### `response.finished`

<!-- YAML
added: v8.4.0
deprecated:
 - v13.4.0
 - v12.16.0
-->

> Stability: 0 - Deprecated. 使用 [`response.writableEnded`][]。

* 类型：{boolean}

指示响应是否已完成的布尔值。开始时为 `false`。在 [`response.end()`][] 执行后，值将为 `true`。

#### `response.getHeader(name)`

<!-- YAML
added: v8.4.0
-->

* `name` {string}
* 返回：{string}

读取已排队但尚未发送到客户端的头部。名称不区分大小写。

```js
const contentType = response.getHeader('content-type');
```

#### `response.getHeaderNames()`

<!-- YAML
added: v8.4.0
-->

* 返回：{string\[]}

返回包含当前传出头部的唯一名称的数组。所有头部名称都是小写的。

```js
response.setHeader('Foo', 'bar');
response.setHeader('Set-Cookie', ['foo=bar', 'bar=baz']);

const headerNames = response.getHeaderNames();
// headerNames === ['foo', 'set-cookie']
```

#### `response.getHeaders()`

<!-- YAML
added: v8.4.0
-->

* 返回：{Object}

返回当前传出头部的浅拷贝。由于使用浅拷贝，数组值可能会被修改，而无需额外调用各种与头部相关的 http 模块方法。返回对象的键是头部名称，值是相应的头部值。所有头部名称都是小写的。

`response.getHeaders()` 方法返回的对象*不*从 JavaScript `Object` 原型继承。这意味着典型的 `Object` 方法，如 `obj.toString()`、`obj.hasOwnProperty()` 等未定义且*不起作用*。

```js
response.setHeader('Foo', 'bar');
response.setHeader('Set-Cookie', ['foo=bar', 'bar=baz']);

const headers = response.getHeaders();
// headers === { foo: 'bar', 'set-cookie': ['foo=bar', 'bar=baz'] }
```

#### `response.hasHeader(name)`

<!-- YAML
added: v8.4.0
-->

* `name` {string}
* 返回：{boolean}

如果 `name` 标识的头部当前设置在传出头部中，则返回 `true`。头部名称匹配不区分大小写。

```js
const hasContentType = response.hasHeader('content-type');
```

#### `response.headersSent`

<!-- YAML
added: v8.4.0
-->

* 类型：{boolean}

如果头部已发送，则为 true，否则为 false（只读）。

#### `response.removeHeader(name)`

<!-- YAML
added: v8.4.0
-->

* `name` {string}

移除已排队等待隐式发送的头部。

```js
response.removeHeader('Content-Encoding');
```

#### `response.req`

<!-- YAML
added: v15.7.0
-->

* 类型：{http2.Http2ServerRequest}

对原始 HTTP2 `request` 对象的引用。

#### `response.sendDate`

<!-- YAML
added: v8.4.0
-->

* 类型：{boolean}

当为 true 时，如果头部中尚未存在 Date 头部，将自动生成并发送到响应中。默认为 true。

这应仅出于测试目的禁用；HTTP 要求响应中包含 Date 头部。

#### `response.setHeader(name, value)`

<!-- YAML
added: v8.4.0
-->

* `name` {string}
* `value` {string|string\[]}

为隐式头部设置单个头部值。如果此头部已存在于待发送头部中，其值将被替换。在此处使用字符串数组以发送具有相同名称的多个头部。

```js
response.setHeader('Content-Type', 'text/html; charset=utf-8');
```

或

```js
response.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
```

尝试设置包含无效字符的头部字段名称或值将导致抛出 [`TypeError`][]。

当头部已使用 [`response.setHeader()`][] 设置时，它们将与传递给 [`response.writeHead()`][] 的任何头部合并，传递给 [`response.writeHead()`][] 的头部具有优先权。

```js
// 返回 content-type = text/plain
const server = http2.createServer((req, res) => {
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('ok');
});
```

#### `response.setTimeout(msecs[, callback])`

<!-- YAML
added: v8.4.0
-->

* `msecs` {number}
* `callback` {Function}
* 返回：{http2.Http2ServerResponse}

将 [`Http2Stream`][] 的超时值设置为 `msecs`。如果提供了回调，则将其添加为响应对象上 `'timeout'` 事件的监听器。

如果未向请求、响应或服务器添加 `'timeout'` 监听器，则 [`Http2Stream`][] 在超时时将被销毁。如果为请求、响应或服务器的 `'timeout'` 事件分配了处理程序，则必须显式处理超时的套接字。

#### `response.socket`

<!-- YAML
added: v8.4.0
-->

* 类型：{net.Socket|tls.TLSSocket}

返回一个 `Proxy` 对象，其行为类似于 `net.Socket`（或 `tls.TLSSocket`），但基于 HTTP/2 逻辑应用 getter、setter 和方法。

`destroyed`、`readable` 和 `writable` 属性将从 `response.stream` 检索并设置到其上。

`destroy`、`emit`、`end`、`on` 和 `once` 方法将在 `response.stream` 上调用。

`setTimeout` 方法将在 `response.stream.session` 上调用。

`pause`、`read`、`resume` 和 `write` 将抛出错误代码为 `ERR_HTTP2_NO_SOCKET_MANIPULATION` 的错误。有关更多信息，请参阅 [`Http2Session` 和套接字][]。

所有其他交互将直接路由到套接字。

```mjs
import { createServer } from 'node:http2';
const server = createServer((req, res) => {
  const ip = req.socket.remoteAddress;
  const port = req.socket.remotePort;
  res.end(`Your IP address is ${ip} and your source port is ${port}.`);
}).listen(3000);
```

```cjs
const http2 = require('node:http2');
const server = http2.createServer((req, res) => {
  const ip = req.socket.remoteAddress;
  const port = req.socket.remotePort;
  res.end(`Your IP address is ${ip} and your source port is ${port}.`);
}).listen(3000);
```

#### `response.statusCode`

<!-- YAML
added: v8.4.0
-->

* 类型：{number}

当使用隐式头部（未显式调用 [`response.writeHead()`][]）时，此属性控制刷新头部时将发送到客户端的状态代码。

```js
response.statusCode = 404;
```

响应头部发送到客户端后，此属性指示已发送的状态代码。

#### `response.statusMessage`

<!-- YAML
added: v8.4.0
-->

* 类型：{string}

HTTP/2 不支持状态消息（RFC 7540 8.1.2.4）。它返回一个空字符串。

#### `response.stream`

<!-- YAML
added: v8.4.0
-->

* 类型：{Http2Stream}

支持响应的 [`Http2Stream`][] 对象。

#### `response.writableEnded`

<!-- YAML
added: v12.9.0
-->

* 类型：{boolean}

在 [`response.end()`][] 被调用后为 `true`。此属性不指示数据是否已刷新，为此请使用 [`writable.writableFinished`][]。

#### `response.write(chunk[, encoding][, callback])`

<!-- YAML
added: v8.4.0
-->

* `chunk` {string|Buffer|Uint8Array}
* `encoding` {string}
* `callback` {Function}
* 返回：{boolean}

如果调用此方法时尚未调用 [`response.writeHead()`][]，它将切换到隐式头部模式并刷新隐式头部。

这发送响应主体的一部分。此方法可以多次调用以提供主体的连续部分。

在 `node:http` 模块中，当请求是 HEAD 请求时，响应主体被省略。同样，`204` 和 `304` 响应*不得*包含消息主体。

`chunk` 可以是字符串或缓冲区。如果 `chunk` 是字符串，则第二个参数指定如何将其编码为字节流。默认 `encoding` 为 `'utf8'`。`callback` 将在数据块刷新时调用。

这是原始 HTTP 主体，与可能使用的更高级别的多部分主体编码无关。

第一次调用 [`response.write()`][] 时，它将发送缓冲的头部信息和主体的第一个块到客户端。第二次调用 [`response.write()`][] 时，Node.js 假定数据将被流式传输，并单独发送新数据。也就是说，响应被缓冲到主体的第一个块。

如果整个数据成功刷新到内核缓冲区，则返回 `true`。如果所有或部分数据在用户内存中排队，则返回 `false`。当缓冲区再次空闲时，将发出 `'drain'`。

#### `response.writeContinue()`

<!-- YAML
added: v8.4.0
-->

向客户端发送状态 `100 Continue`，指示应发送请求主体。请参阅 `Http2Server` 和 `Http2SecureServer` 上的 [`'checkContinue'`][] 事件。

#### `response.writeEarlyHints(hints)`

<!-- YAML
added: v18.11.0
-->

* `hints` {Object}

向客户端发送状态 `103 Early Hints` 和 Link 头部，指示用户代理可以预加载/预连接链接的资源。`hints` 是一个包含要随早期提示消息发送的头部值的对象。

**示例**

```js
const earlyHintsLink = '</styles.css>; rel=preload; as=style';
response.writeEarlyHints({
  'link': earlyHintsLink,
});

const earlyHintsLinks = [
  '</styles.css>; rel=preload; as=style',
  '</scripts.js>; rel=preload; as=script',
];
response.writeEarlyHints({
  'link': earlyHintsLinks,
});
```

#### `response.writeHead(statusCode[, statusMessage][, headers])`

<!-- YAML
added: v8.4.0
changes:
  - version:
     - v11.10.0
     - v10.17.0
    pr-url: https://github.com/nodejs/node/pull/25974
    description: Return `this` from `writeHead()` to allow chaining with
                 `end()`.
-->

* `statusCode` {number}
* `statusMessage` {string}
* `headers` {HTTP/2 Headers Object|HTTP/2 Raw Headers}
* 返回：{http2.Http2ServerResponse}

向请求发送响应头部。状态代码是 3 位 HTTP 状态代码，如 `404`。最后一个参数 `headers` 是响应头部。

返回对 `Http2ServerResponse` 的引用，以便可以链式调用。

为了与 [HTTP/1][] 兼容，可以传递人类可读的 `statusMessage` 作为第二个参数。但是，由于 `statusMessage` 在 HTTP/2 中没有意义，该参数将无效，并且会发出进程警告。

```js
const body = 'hello world';
response.writeHead(200, {
  'Content-Length': Buffer.byteLength(body),
  'Content-Type': 'text/plain; charset=utf-8',
});
```

`Content-Length` 以字节为单位，而不是字符。`Buffer.byteLength()` API 可用于确定给定编码中的字节数。在出站消息上，Node.js 不检查 Content-Length 和正在传输的主体长度是否相等。但是，在接收消息时，当 `Content-Length` 与实际有效载荷大小不匹配时，Node.js 将自动拒绝消息。

此方法在消息上最多可调用一次，然后调用 [`response.end()`][]。

如果在调用此之前调用了 [`response.write()`][] 或 [`response.end()`][]，则将计算隐式/可变头部并调用此函数。

当头部已使用 [`response.setHeader()`][] 设置时，它们将与传递给 [`response.writeHead()`][] 的任何头部合并，传递给 [`response.writeHead()`][] 的头部具有优先权。

```js
// 返回 content-type = text/plain
const server = http2.createServer((req, res) => {
  res.setHeader('Content-Type', 'text/html; charset=utf-8');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, { 'Content-Type': 'text/plain; charset=utf-8' });
  res.end('ok');
});
```

尝试设置包含无效字符的头部字段名称或值将导致抛出 [`TypeError`][]。

## 收集 HTTP/2 性能指标

[Performance Observer][] API 可用于收集每个 `Http2Session` 和 `Http2Stream` 实例的基本性能指标。

```mjs
import { PerformanceObserver } from 'node:perf_hooks';

const obs = new PerformanceObserver((items) => {
  const entry = items.getEntries()[0];
  console.log(entry.entryType);  // 打印 'http2'
  if (entry.name === 'Http2Session') {
    // 条目包含关于 Http2Session 的统计信息
  } else if (entry.name === 'Http2Stream') {
    // 条目包含关于 Http2Stream 的统计信息
  }
});
obs.observe({ entryTypes: ['http2'] });
```

```cjs
const { PerformanceObserver } = require('node:perf_hooks');

const obs = new PerformanceObserver((items) => {
  const entry = items.getEntries()[0];
  console.log(entry.entryType);  // 打印 'http2'
  if (entry.name === 'Http2Session') {
    // 条目包含关于 Http2Session 的统计信息
  } else if (entry.name === 'Http2Stream') {
    // 条目包含关于 Http2Stream 的统计信息
  }
});
obs.observe({ entryTypes: ['http2'] });
```

`PerformanceEntry` 的 `entryType` 属性将等于 `'http2'`。

`PerformanceEntry` 的 `name` 属性将等于 `'Http2Stream'` 或 `'Http2Session'`。

如果 `name` 等于 `Http2Stream`，则 `PerformanceEntry` 将包含以下附加属性：

* `bytesRead` {number} 为此 `Http2Stream` 接收的 `DATA` 帧字节数。
* `bytesWritten` {number} 为此 `Http2Stream` 发送的 `DATA` 帧字节数。
* `id` {number} 关联的 `Http2Stream` 的标识符
* `timeToFirstByte` {number} `PerformanceEntry` `startTime` 与接收第一个 `DATA` 帧之间经过的毫秒数。
* `timeToFirstByteSent` {number} `PerformanceEntry` `startTime` 与发送第一个 `DATA` 帧之间经过的毫秒数。
* `timeToFirstHeader` {number} `PerformanceEntry` `startTime` 与接收第一个头部之间经过的毫秒数。

如果 `name` 等于 `Http2Session`，则 `PerformanceEntry` 将包含以下附加属性：

* `bytesRead` {number} 为此 `Http2Session` 接收的字节数。
* `bytesWritten` {number} 为此 `Http2Session` 发送的字节数。
* `framesReceived` {number} `Http2Session` 接收的 HTTP/2 帧数。
* `framesSent` {number} `Http2Session` 发送的 HTTP/2 帧数。
* `maxConcurrentStreams` {number} `Http2Session` 生命周期内同时打开的最大流数。
* `pingRTT` {number} 自传输 `PING` 帧与其确认接收之间经过的毫秒数。仅当在 `Http2Session` 上发送了 `PING` 帧时存在。
* `streamAverageDuration` {number} 所有 `Http2Stream` 实例的平均持续时间（毫秒）。
* `streamCount` {number} `Http2Session` 处理的 `Http2Stream` 实例数。
* `type` {string} `'server'` 或 `'client'` 以标识 `Http2Session` 的类型。

## 关于 `:authority` 和 `host` 的说明

HTTP/2 要求请求具有 `:authority` 伪头部或 `host` 头部。在直接构造 HTTP/2 请求时优先使用 `:authority`，在从 HTTP/1 转换时（例如在代理中）使用 `host`。

如果 `:authority` 不存在，兼容性 API 将回退到 `host`。有关更多信息，请参阅 [`request.authority`][]。但是，如果不使用兼容性 API（或直接使用 `req.headers`），则需要自己实现任何回退行为。

[ALPN Protocol ID]: https://www.iana.org/assignments/tls-extensiontype-values/tls-extensiontype-values.xhtml#alpn-protocol-ids
[ALPN negotiation]: #alpn-negotiation
[Compatibility API]: #compatibility-api
[HTTP/1]: http.md
[HTTP/2]: https://tools.ietf.org/html/rfc7540
[HTTP/2 Headers Object]: #headers-object
[HTTP/2 Raw Headers]: #raw-headers
[HTTP/2 Settings Object]: #settings-object
[HTTP/2 Unencrypted]: https://http2.github.io/faq/#does-http2-require-encryption
[HTTPS]: https.md
[Performance Observer]: perf_hooks.md
[RFC 7838]: https://tools.ietf.org/html/rfc7838
[RFC 8336]: https://tools.ietf.org/html/rfc8336
[RFC 8441]: https://tools.ietf.org/html/rfc8441
[RFC 9113]: https://datatracker.ietf.org/doc/html/rfc9113#section-5.3.1
[Sensitive headers]: #sensitive-headers
[`'checkContinue'`]: #event-checkcontinue
[`'connect'`]: #event-connect
[`'request'`]: #event-request
[`'unknownProtocol'`]: #event-unknownprotocol
[`ClientHttp2Stream`]: #class-clienthttp2stream
[`Duplex`]: stream.md#class-streamduplex
[`Http2ServerRequest`]: #class-http2http2serverrequest
[`Http2ServerResponse`]: #class-http2http2serverresponse
[`Http2Session` and Sockets]: #http2session-and-sockets
[`Http2Session`'s `'stream'` event]: #event-stream
[`Http2Stream`]: #class-http2stream
[`ServerHttp2Stream`]: #class-serverhttp2stream
[`TypeError`]: errors.md#class-typeerror
[`http2.SecureServer`]: #class-http2secureserver
[`http2.Server`]: #class-http2server
[`http2.createSecureServer()`]: #http2createsecureserveroptions-onrequesthandler
[`http2.createServer()`]: #http2createserveroptions-onrequesthandler
[`http2session.close()`]: #http2sessionclosecallback
[`http2stream.pushStream()`]: #http2streampushstreamheaders-options-callback
[`import()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import
[`net.Server.close()`]: net.md#serverclosecallback
[`net.Socket.bufferSize`]: net.md#socketbuffersize
[`net.Socket.prototype.ref()`]: net.md#socketref
[`net.Socket.prototype.unref()`]: net.md#socketunref
[`net.Socket`]: net.md#class-netsocket
[`net.connect()`]: net.md#netconnect
[`net.createServer()`]: net.md#netcreateserveroptions-connectionlistener
[`request.authority`]: #requestauthority
[`request.maxHeadersCount`]: http.md#requestmaxheaderscount
[`request.socket.getPeerCertificate()`]: tls.md#tlssocketgetpeercertificatedetailed
[`request.socket`]: #requestsocket
[`response.end()`]: #responseenddata-encoding-callback
[`response.setHeader()`]: #responsesetheadername-value
[`response.socket`]: #responsesocket
[`response.writableEnded`]: #responsewritableended
[`response.write()`]: #responsewritechunk-encoding-callback
[`response.write(data, encoding)`]: http.md#responsewritechunk-encoding-callback
[`response.writeContinue()`]: #responsewritecontinue
[`response.writeHead()`]: #responsewriteheadstatuscode-statusmessage-headers
[`server.close()`]: #serverclosecallback
[`server.maxHeadersCount`]: http.md#servermaxheaderscount
[`tls.Server.close()`]: tls.md#serverclosecallback
[`tls.TLSSocket`]: tls.md#class-tlstlssocket
[`tls.connect()`]: tls.md#tlsconnectoptions-callback
[`tls.createServer()`]: tls.md#tlscreateserveroptions-secureconnectionlistener
[`writable.writableFinished`]: stream.md#writablewritablefinished
[error code]: #error-codes-for-rst_stream-and-goaway