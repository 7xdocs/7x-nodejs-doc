# HTTP

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/http.js -->

该模块包含客户端和服务器，可以通过 `require('node:http')` (CommonJS) 或 `import * as http from 'node:http'` (ES module) 导入。

Node.js 中的 HTTP 接口旨在支持该协议许多传统上难以使用的特性。特别是大的、可能分块编码的消息。该接口小心地避免缓冲整个请求或响应，因此用户能够流式传输数据。

HTTP 消息头由一个类似这样的对象表示：

```json
{
  "content-length": "123",
  "content-type": "text/plain",
  "connection": "keep-alive",
  "host": "example.com",
  "accept": "*/*"
}
```

键是小写的。值不会被修改。

为了支持所有可能的 HTTP 应用，Node.js HTTP API 非常底层。它只处理流处理和消息解析。它将消息解析为头部和主体，但不解析实际的头部或主体。

有关重复头如何处理，请参见 [`message.headers`][]。

接收到的原始头部保留在 `rawHeaders` 属性中，这是一个 `[key, value, key2, value2, ...]` 的数组。例如，前面的消息头对象可能有一个 `rawHeaders` 列表，如下所示：

<!-- eslint-disable @stylistic/js/semi -->

```js
[
  'ConTent-Length',
  '123456',
  'content-LENGTH',
  '123',
  'content-type',
  'text/plain',
  'CONNECTION',
  'keep-alive',
  'Host',
  'example.com',
  'accepT',
  '*/*',
];
```

## 类：`http.Agent`

<!-- YAML
added: v0.3.4
-->

`Agent` 负责管理 HTTP 客户端的连接持久化和复用。它为给定的主机和端口维护一个待处理请求队列，为每个请求复用单个 socket 连接，直到队列为空，此时 socket 要么被销毁，要么放入池中以便再次用于相同主机和端口的请求。它是被销毁还是放入池中取决于 `keepAlive` [选项](#new-agentoptions)。

池化的连接为其启用了 TCP Keep-Alive，但服务器仍可能关闭空闲连接，在这种情况下，它们将从池中移除，并在为该主机和端口发出新的 HTTP 请求时建立新连接。服务器也可能拒绝允许同一连接上的多个请求，在这种情况下，必须为每个请求重新建立连接，并且无法池化。`Agent` 仍将向该服务器发出请求，但每个请求将通过新连接进行。

当连接被客户端或服务器关闭时，它将从池中移除。池中任何未使用的 socket 将被取消引用，以便在没有未完成请求时不保持 Node.js 进程运行（参见 [`socket.unref()`][]）。

当不再使用时，最好 [`destroy()`][] 一个 `Agent` 实例，因为未使用的 socket 会消耗操作系统资源。

当 socket 发出 `'close'` 事件或 `'agentRemove'` 事件时，它会从代理中移除。当打算长时间保持一个 HTTP 请求打开而不将其保留在代理中时，可以执行类似以下的操作：

```js
http
  .get(options, (res) => {
    // 处理事情
  })
  .on('socket', (socket) => {
    socket.emit('agentRemove');
  });
```

代理也可以用于单个请求。通过向 `http.get()` 或 `http.request()` 函数提供 `{agent: false}` 作为选项，将使用具有默认选项的一次性 `Agent` 进行客户端连接。

`agent:false`:

```js
http.get(
  {
    hostname: 'localhost',
    port: 80,
    path: '/',
    agent: false, // 为此请求创建一个新的代理
  },
  (res) => {
    // 处理响应
  }
);
```

### `new Agent([options])`

<!-- YAML
added: v0.3.4
changes:
  - version:
    - v24.7.0
    pr-url: https://github.com/nodejs/node/pull/59315
    description: 添加对 `agentKeepAliveTimeoutBuffer` 的支持。
  - version:
    - v24.5.0
    pr-url: https://github.com/nodejs/node/pull/58980
    description: 添加对 `proxyEnv` 的支持。
  - version:
    - v24.5.0
    pr-url: https://github.com/nodejs/node/pull/58980
    description: 添加对 `defaultPort` 和 `protocol` 的支持。
  - version:
      - v15.6.0
      - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/36685
    description: 默认调度从 'fifo' 更改为 'lifo'。
  - version:
    - v14.5.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/33617
    description: 向代理构造函数添加 `maxTotalSockets` 选项。
  - version:
      - v14.5.0
      - v12.20.0
    pr-url: https://github.com/nodejs/node/pull/33278
    description: 添加 `scheduling` 选项以指定空闲 socket 调度策略。
-->

- `options` {Object} 要在代理上设置的可配置选项集合。
  可以包含以下字段：
  - `keepAlive` {boolean} 即使没有未完成的请求，也保持 socket 存在，以便它们可以用于未来的请求，而无需重新建立 TCP 连接。不要与 `Connection` 头的 `keep-alive` 值混淆。使用代理时，除了显式指定 `Connection` 头，或者当 `keepAlive` 和 `maxSockets` 选项分别设置为 `false` 和 `Infinity` 时，将始终发送 `Connection: keep-alive` 头，在这种情况下将使用 `Connection: close`。**默认值：** `false`。
  - `keepAliveMsecs` {number} 当使用 `keepAlive` 选项时，指定 TCP Keep-Alive 数据包的[初始延迟][initial delay]。当 `keepAlive` 选项为 `false` 或 `undefined` 时忽略。**默认值：** `1000`。
  - `agentKeepAliveTimeoutBuffer` {number} 在确定 socket 过期时间时，从服务器提供的 `keep-alive: timeout=...` 提示中减去的毫秒数。此缓冲区有助于确保代理在服务器关闭 socket 之前稍早关闭 socket，减少在即将被服务器关闭的 socket 上发送请求的机会。
    **默认值：** `1000`。
  - `maxSockets` {number} 每个主机允许的最大 socket 数量。
    如果同一主机打开多个并发连接，每个请求将使用新的 socket，直到达到 `maxSockets` 值。
    如果主机尝试打开比 `maxSockets` 更多的连接，额外的请求将进入待处理请求队列，并在现有连接终止时进入活动连接状态。
    这确保在任何时间点，来自给定主机的活动连接最多为 `maxSockets` 个。
    **默认值：** `Infinity`。
  - `maxTotalSockets` {number} 所有主机总共允许的最大 socket 数量。每个请求将使用新的 socket，直到达到最大值。
    **默认值：** `Infinity`。
  - `maxFreeSockets` {number} 每个主机在空闲状态下保持打开的 socket 最大数量。仅在 `keepAlive` 设置为 `true` 时相关。
    **默认值：** `256`。
  - `scheduling` {string} 选择下一个要使用的空闲 socket 时应用的调度策略。可以是 `'fifo'` 或 `'lifo'`。
    两种调度策略的主要区别在于，`'lifo'` 选择最近使用的 socket，而 `'fifo'` 选择最近最少使用的 socket。
    在每秒请求率较低的情况下，`'lifo'` 调度将降低选择可能因不活动而被服务器关闭的 socket 的风险。
    在每秒请求率较高的情况下，`'fifo'` 调度将最大化打开的 socket 数量，而 `'lifo'` 调度将使其尽可能低。
    **默认值：** `'lifo'`。
  - `timeout` {number} socket 超时（毫秒）。
    这将在创建 socket 时设置超时。
  - `proxyEnv` {Object|undefined} 代理配置的环境变量。
    有关详细信息，请参见[内置代理支持][Built-in Proxy Support]。**默认值：** `undefined`
    - `HTTP_PROXY` {string|undefined} HTTP 请求应使用的代理服务器的 URL。
      如果未定义，则 HTTP 请求不使用代理。
    - `HTTPS_PROXY` {string|undefined} HTTPS 请求应使用的代理服务器的 URL。
      如果未定义，则 HTTPS 请求不使用代理。
    - `NO_PROXY` {string|undefined} 指定不应通过代理路由的端点的模式。
    - `http_proxy` {string|undefined} 与 `HTTP_PROXY` 相同。如果两者都设置，`http_proxy` 优先。
    - `https_proxy` {string|undefined} 与 `HTTPS_PROXY` 相同。如果两者都设置，`https_proxy` 优先。
    - `no_proxy` {string|undefined} 与 `NO_PROXY` 相同。如果两者都设置，`no_proxy` 优先。
  - `defaultPort` {number} 当请求中未指定端口时使用的默认端口。**默认值：** `80`。
  - `protocol` {string} 代理使用的协议。**默认值：** `'http:'`。

[`socket.connect()`][] 中的 `options` 也受支持。

要配置它们中的任何一个，必须创建自定义的 [`http.Agent`][] 实例。

```mjs
import { Agent, request } from 'node:http';
const keepAliveAgent = new Agent({ keepAlive: true });
options.agent = keepAliveAgent;
request(options, onResponseCallback);
```

```cjs
const http = require('node:http');
const keepAliveAgent = new http.Agent({ keepAlive: true });
options.agent = keepAliveAgent;
http.request(options, onResponseCallback);
```

### `agent.createConnection(options[, callback])`

<!-- YAML
added: v0.11.4
-->

- `options` {Object} 包含连接详细信息的选项。检查
  [`net.createConnection()`][] 了解选项的格式
- `callback` {Function} 接收创建的 socket 的回调函数
- 返回：{stream.Duplex}

生成用于 HTTP 请求的 socket/流。

默认情况下，此函数与 [`net.createConnection()`][] 相同。但是，自定义代理可能覆盖此方法，以便获得更大的灵活性。

可以通过两种方式之一提供 socket/流：通过从此函数返回 socket/流，或者通过将 socket/流传递给 `callback`。

此方法保证返回 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

`callback` 的签名为 `(err, stream)`。

### `agent.keepSocketAlive(socket)`

<!-- YAML
added: v8.1.0
-->

- `socket` {stream.Duplex}

当 `socket` 从请求中分离并可以被 `Agent` 持久化时调用。默认行为是：

```js
socket.setKeepAlive(true, this.keepAliveMsecs);
socket.unref();
return true;
```

此方法可以被特定的 `Agent` 子类覆盖。如果此方法返回假值，socket 将被销毁而不是持久化以供下一个请求使用。

`socket` 参数可以是 {net.Socket} 的实例，即 {stream.Duplex} 的子类。

### `agent.reuseSocket(socket, request)`

<!-- YAML
added: v8.1.0
-->

- `socket` {stream.Duplex}
- `request` {http.ClientRequest}

当 `socket` 由于 keep-alive 选项被持久化后附加到 `request` 时调用。默认行为是：

```js
socket.ref();
```

此方法可以被特定的 `Agent` 子类覆盖。

`socket` 参数可以是 {net.Socket} 的实例，即 {stream.Duplex} 的子类。

### `agent.destroy()`

<!-- YAML
added: v0.11.4
-->

销毁代理当前正在使用的任何 socket。

通常不需要这样做。但是，如果使用启用了 `keepAlive` 的代理，那么当代理不再需要时，最好显式关闭代理。否则，socket 可能会在服务器终止它们之前保持打开很长时间。

### `agent.freeSockets`

<!-- YAML
added: v0.11.4
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/36409
    description: The property now has a `null` prototype.
-->

- 类型：{Object}

一个对象，包含当 `keepAlive` 启用时当前等待代理使用的 socket 数组。请勿修改。

`freeSockets` 列表中的 socket 将在 `'timeout'` 时自动销毁并从数组中移除。

### `agent.getName([options])`

<!-- YAML
added: v0.11.4
changes:
  - version:
    - v17.7.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/41906
    description: The `options` parameter is now optional.
-->

- `options` {Object} 一组提供名称生成信息的选项
  - `host` {string} 发出请求的服务器的域名或 IP 地址
  - `port` {number} 远程服务器的端口
  - `localAddress` {string} 发出请求时用于网络连接的本地接口
  - `family` {integer} 如果此值不等于 `undefined`，则必须为 4 或 6。
- 返回：{string}

获取一组请求选项的唯一名称，以确定连接是否可以复用。对于 HTTP 代理，这将返回 `host:port:localAddress` 或 `host:port:localAddress:family`。对于 HTTPS 代理，名称包括 CA、证书、密码和其他确定 socket 可复用性的 HTTPS/TLS 特定选项。

### `agent.maxFreeSockets`

<!-- YAML
added: v0.11.7
-->

- 类型：{number}

默认设置为 256。对于启用 `keepAlive` 的代理，这设置了在空闲状态下将保持打开的最大 socket 数量。

### `agent.maxSockets`

<!-- YAML
added: v0.3.6
-->

- 类型：{number}

默认设置为 `Infinity`。确定代理每个源可以打开的并发 socket 数量。源是 [`agent.getName()`][] 的返回值。

### `agent.maxTotalSockets`

<!-- YAML
added:
  - v14.5.0
  - v12.19.0
-->

- 类型：{number}

默认设置为 `Infinity`。确定代理可以打开的并发 socket 总数。与 `maxSockets` 不同，此参数适用于所有源。

### `agent.requests`

<!-- YAML
added: v0.5.9
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/36409
    description: The property now has a `null` prototype.
-->

- 类型：{Object}

一个包含尚未分配给 socket 的请求队列的对象。请勿修改。

### `agent.sockets`

<!-- YAML
added: v0.3.6
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/36409
    description: The property now has a `null` prototype.
-->

- 类型：{Object}

一个包含代理当前正在使用的 socket 数组的对象。请勿修改。

## 类：`http.ClientRequest`

<!-- YAML
added: v0.1.17
-->

- 扩展：{http.OutgoingMessage}

此对象在内部创建并从 [`http.request()`][] 返回。它代表一个*进行中*的请求，其头部已排队。头部仍然可以使用 [`setHeader(name, value)`][]、[`getHeader(name)`][]、[`removeHeader(name)`][] API 进行修改。实际头部将在第一个数据块发送时或调用 [`request.end()`][] 时发送。

要获取响应，请向请求对象添加 [`'response'`][] 的监听器。当响应头已接收时，将从请求对象发出 [`'response'`][]。[`'response'`][] 事件执行时带有一个参数，该参数是 [`http.IncomingMessage`][] 的实例。

在 [`'response'`][] 事件期间，可以向响应对象添加监听器；特别是监听 `'data'` 事件。

如果未添加 [`'response'`][] 处理程序，则响应将被完全丢弃。但是，如果添加了 [`'response'`][] 事件处理程序，则**必须**消耗响应对象中的数据，可以通过在每次有 `'readable'` 事件时调用 `response.read()`，或通过添加 `'data'` 处理程序，或通过调用 `.resume()` 方法。直到数据被消耗，`'end'` 事件不会触发。此外，在数据被读取之前，它将消耗内存，最终可能导致"进程内存不足"错误。

为了向后兼容，`res` 只有在注册了 `'error'` 监听器时才会发出 `'error'`。

设置 `Content-Length` 头以限制响应体大小。
如果 [`response.strictContentLength`][] 设置为 `true`，不匹配 `Content-Length` 头值将导致抛出 `Error`，标识为 `code:` [`'ERR_HTTP_CONTENT_LENGTH_MISMATCH'`][]。

`Content-Length` 值应以字节为单位，而不是字符。使用 [`Buffer.byteLength()`][] 来确定主体的字节长度。

### 事件：`'abort'`

<!-- YAML
added: v1.4.1
deprecated:
  - v17.0.0
  - v16.12.0
-->

> Stability: 0 - 已弃用。请监听 `'close'` 事件。

当请求被客户端中止时触发。此事件仅在第一次调用 `abort()` 时触发。

### 事件：`'close'`

<!-- YAML
added: v0.5.4
-->

指示请求已完成，或其底层连接过早终止（在响应完成之前）。

### 事件：`'connect'`

<!-- YAML
added: v0.7.0
-->

- `response` {http.IncomingMessage}
- `socket` {stream.Duplex}
- `head` {Buffer}

每次服务器响应带有 `CONNECT` 方法的请求时触发。如果未监听此事件，接收到 `CONNECT` 方法的客户端将关闭其连接。

此事件保证传递一个 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

演示如何监听 `'connect'` 事件的客户端和服务器对：

```mjs
import { createServer, request } from 'node:http';
import { connect } from 'node:net';
import { URL } from 'node:url';

// 创建一个 HTTP 隧道代理
const proxy = createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('okay');
});
proxy.on('connect', (req, clientSocket, head) => {
  // 连接到源服务器
  const { port, hostname } = new URL(`http://${req.url}`);
  const serverSocket = connect(port || 80, hostname, () => {
    clientSocket.write(
      'HTTP/1.1 200 Connection Established\r\n' +
        'Proxy-agent: Node.js-Proxy\r\n' +
        '\r\n'
    );
    serverSocket.write(head);
    serverSocket.pipe(clientSocket);
    clientSocket.pipe(serverSocket);
  });
});

// 现在代理正在运行
proxy.listen(1337, '127.0.0.1', () => {
  // 向隧道代理发出请求
  const options = {
    port: 1337,
    host: '127.0.0.1',
    method: 'CONNECT',
    path: 'www.google.com:80',
  };

  const req = request(options);
  req.end();

  req.on('connect', (res, socket, head) => {
    console.log('got connected!');

    // 通过 HTTP 隧道发出请求
    socket.write(
      'GET / HTTP/1.1\r\n' +
        'Host: www.google.com:80\r\n' +
        'Connection: close\r\n' +
        '\r\n'
    );
    socket.on('data', (chunk) => {
      console.log(chunk.toString());
    });
    socket.on('end', () => {
      proxy.close();
    });
  });
});
```

```cjs
const http = require('node:http');
const net = require('node:net');
const { URL } = require('node:url');

// 创建一个 HTTP 隧道代理
const proxy = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('okay');
});
proxy.on('connect', (req, clientSocket, head) => {
  // 连接到源服务器
  const { port, hostname } = new URL(`http://${req.url}`);
  const serverSocket = net.connect(port || 80, hostname, () => {
    clientSocket.write(
      'HTTP/1.1 200 Connection Established\r\n' +
        'Proxy-agent: Node.js-Proxy\r\n' +
        '\r\n'
    );
    serverSocket.write(head);
    serverSocket.pipe(clientSocket);
    clientSocket.pipe(serverSocket);
  });
});

// 现在代理正在运行
proxy.listen(1337, '127.0.0.1', () => {
  // 向隧道代理发出请求
  const options = {
    port: 1337,
    host: '127.0.0.1',
    method: 'CONNECT',
    path: 'www.google.com:80',
  };

  const req = http.request(options);
  req.end();

  req.on('connect', (res, socket, head) => {
    console.log('got connected!');

    // 通过 HTTP 隧道发出请求
    socket.write(
      'GET / HTTP/1.1\r\n' +
        'Host: www.google.com:80\r\n' +
        'Connection: close\r\n' +
        '\r\n'
    );
    socket.on('data', (chunk) => {
      console.log(chunk.toString());
    });
    socket.on('end', () => {
      proxy.close();
    });
  });
});
```

### 事件：`'continue'`

<!-- YAML
added: v0.3.2
-->

当服务器发送 '100 Continue' HTTP 响应时触发，通常是因为请求包含 'Expect: 100-continue'。这是一个指示客户端应发送请求体的指令。

### 事件：`'finish'`

<!-- YAML
added: v0.3.6
-->

当请求已发送时触发。更具体地说，当响应头和主体的最后一段已移交操作系统通过网络传输时触发此事件。这并不意味着服务器已收到任何内容。

### 事件：`'information'`

<!-- YAML
added: v10.0.0
-->

- `info` {Object}
  - `httpVersion` {string}
  - `httpVersionMajor` {integer}
  - `httpVersionMinor` {integer}
  - `statusCode` {integer}
  - `statusMessage` {string}
  - `headers` {Object}
  - `rawHeaders` {string\[]}

当服务器发送 1xx 中间响应（不包括 101 Upgrade）时触发。此事件的监听器将接收一个包含 HTTP 版本、状态码、状态消息、键值头对象以及原始头名称数组及其各自值的对象。

```mjs
import { request } from 'node:http';

const options = {
  host: '127.0.0.1',
  port: 8080,
  path: '/length_request',
};

// 发出请求
const req = request(options);
req.end();

req.on('information', (info) => {
  console.log(`在主响应之前获得信息: ${info.statusCode}`);
});
```

```cjs
const http = require('node:http');

const options = {
  host: '127.0.0.1',
  port: 8080,
  path: '/length_request',
};

// 发出请求
const req = http.request(options);
req.end();

req.on('information', (info) => {
  console.log(`在主响应之前获得信息: ${info.statusCode}`);
});
```

101 Upgrade 状态不会触发此事件，因为它们脱离了传统的 HTTP 请求/响应链，例如 WebSocket、就地 TLS 升级或 HTTP 2.0。要接收 101 Upgrade 通知，请监听 [`'upgrade'`][] 事件。

### 事件：`'response'`

<!-- YAML
added: v0.1.0
-->

- `response` {http.IncomingMessage}

当收到此请求的响应时触发。此事件仅触发一次。

### 事件：`'socket'`

<!-- YAML
added: v0.5.3
-->

- `socket` {stream.Duplex}

此事件保证传递一个 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

### 事件：`'timeout'`

<!-- YAML
added: v0.7.8
-->

当底层 socket 因不活动而超时时触发。这只通知 socket 已空闲。请求必须手动销毁。

另请参见：[`request.setTimeout()`][]。

### 事件：`'upgrade'`

<!-- YAML
added: v0.1.94
-->

- `response` {http.IncomingMessage}
- `socket` {stream.Duplex}
- `head` {Buffer}

每次服务器响应带有升级的请求时触发。如果未监听此事件且响应状态码为 101 Switching Protocols，接收到升级头的客户端将关闭其连接。

此事件保证传递一个 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

演示如何监听 `'upgrade'` 事件的客户端服务器对。

```mjs
import http from 'node:http';
import process from 'node:process';

// 创建一个 HTTP 服务器
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('okay');
});
server.on('upgrade', (req, socket, head) => {
  socket.write(
    'HTTP/1.1 101 Web Socket Protocol Handshake\r\n' +
      'Upgrade: WebSocket\r\n' +
      'Connection: Upgrade\r\n' +
      '\r\n'
  );

  socket.pipe(socket); // 回显
});

// 现在服务器正在运行
server.listen(1337, '127.0.0.1', () => {
  // 发出请求
  const options = {
    port: 1337,
    host: '127.0.0.1',
    headers: {
      Connection: 'Upgrade',
      Upgrade: 'websocket',
    },
  };

  const req = http.request(options);
  req.end();

  req.on('upgrade', (res, socket, upgradeHead) => {
    console.log('got upgraded!');
    socket.end();
    process.exit(0);
  });
});
```

```cjs
const http = require('node:http');

// 创建一个 HTTP 服务器
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('okay');
});
server.on('upgrade', (req, socket, head) => {
  socket.write(
    'HTTP/1.1 101 Web Socket Protocol Handshake\r\n' +
      'Upgrade: WebSocket\r\n' +
      'Connection: Upgrade\r\n' +
      '\r\n'
  );

  socket.pipe(socket); // 回显
});

// 现在服务器正在运行
server.listen(1337, '127.0.0.1', () => {
  // 发出请求
  const options = {
    port: 1337,
    host: '127.0.0.1',
    headers: {
      Connection: 'Upgrade',
      Upgrade: 'websocket',
    },
  };

  const req = http.request(options);
  req.end();

  req.on('upgrade', (res, socket, upgradeHead) => {
    console.log('got upgraded!');
    socket.end();
    process.exit(0);
  });
});
```

### `request.abort()`

<!-- YAML
added: v0.3.8
deprecated:
  - v14.1.0
  - v13.14.0
-->

> Stability: 0 - 已弃用：使用 [`request.destroy()`][] 代替。

将请求标记为中止。调用此方法将导致响应中的剩余数据被丢弃，并且 socket 被销毁。

### `request.aborted`

<!-- YAML
added: v0.11.14
deprecated:
  - v17.0.0
  - v16.12.0
changes:
  - version: v11.0.0
    pr-url: https://github.com/nodejs/node/pull/20230
    description: The `aborted` property is no longer a timestamp number.
-->

> Stability: 0 - 已弃用。检查 [`request.destroyed`][] 代替。

- 类型：{boolean}

如果请求已中止，`request.aborted` 属性将为 `true`。

### `request.connection`

<!-- YAML
added: v0.3.0
deprecated: v13.0.0
-->

> Stability: 0 - 已弃用。使用 [`request.socket`][]。

- 类型：{stream.Duplex}

参见 [`request.socket`][]。

### `request.cork()`

<!-- YAML
added:
 - v13.2.0
 - v12.16.0
-->

参见 [`writable.cork()`][]。

### `request.end([data[, encoding]][, callback])`

<!-- YAML
added: v0.1.90
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/33155
    description: The `data` parameter can now be a `Uint8Array`.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18780
    description: This method now returns a reference to `ClientRequest`.
-->

- `data` {string|Buffer|Uint8Array}
- `encoding` {string}
- `callback` {Function}
- 返回：{this}

完成发送请求。如果主体的任何部分未发送，它将将它们刷新到流中。如果请求是分块的，这将发送终止的 `'0\r\n\r\n'`。

如果指定了 `data`，则相当于调用 [`request.write(data, encoding)`][] 后跟 `request.end(callback)`。

如果指定了 `callback`，它将在请求流完成时被调用。

### `request.destroy([error])`


<!-- YAML
added: v0.3.0
changes:
  - version: v14.5.0
    pr-url: https://github.com/nodejs/node/pull/32789
    description: The function returns `this` for consistency with other Readable
                 streams.
-->

- `error` {Error} 可选，与 `'error'` 事件一起发出的错误。
- 返回：{this}

销毁请求。可选地发出 `'error'` 事件，并发出 `'close'` 事件。调用此方法将导致响应中的剩余数据被丢弃，并且 socket 被销毁。

有关更多详细信息，请参见 [`writable.destroy()`][]。

#### `request.destroyed`

<!-- YAML
added:
  - v14.1.0
  - v13.14.0
-->

- 类型：{boolean}

在调用 [`request.destroy()`][] 后为 `true`。

有关更多详细信息，请参见 [`writable.destroyed`][]。

### `request.finished`

<!-- YAML
added: v0.0.1
deprecated:
 - v13.4.0
 - v12.16.0
-->

> Stability: 0 - 已弃用。使用 [`request.writableEnded`][]。

- 类型：{boolean}

如果已调用 [`request.end()`][]，`request.finished` 属性将为 `true`。如果请求是通过 [`http.get()`][] 发起的，`request.end()` 将自动调用。

### `request.flushHeaders()`

<!-- YAML
added: v1.6.0
-->

刷新请求头。

出于效率原因，Node.js 通常缓冲请求头，直到调用 `request.end()` 或写入第一个请求数据块。然后它尝试将请求头和数据打包到单个 TCP 数据包中。

这通常是需要的（它节省了一次 TCP 往返），但当第一个数据可能直到很晚才发送时则不需要。`request.flushHeaders()` 绕过优化并启动请求。

### `request.getHeader(name)`

<!-- YAML
added: v1.6.0
-->

- `name` {string}
- 返回：{any}

读取请求上的一个头。名称不区分大小写。返回值的类型取决于提供给 [`request.setHeader()`][] 的参数。

```js
request.setHeader('content-type', 'text/html');
request.setHeader('Content-Length', Buffer.byteLength(body));
request.setHeader('Cookie', ['type=ninja', 'language=javascript']);
const contentType = request.getHeader('Content-Type');
// 'contentType' 是 'text/html'
const contentLength = request.getHeader('Content-Length');
// 'contentLength' 是数字类型
const cookie = request.getHeader('Cookie');
// 'cookie' 是字符串数组类型
```

### `request.getHeaderNames()`

<!-- YAML
added: v7.7.0
-->

- 返回：{string\[]}

返回包含当前传出头的唯一名称的数组。所有头名称都是小写的。

```js
request.setHeader('Foo', 'bar');
request.setHeader('Cookie', ['foo=bar', 'bar=baz']);

const headerNames = request.getHeaderNames();
// headerNames === ['foo', 'cookie']
```

### `request.getHeaders()`

<!-- YAML
added: v7.7.0
-->

- 返回：{Object}

返回当前传出头的浅拷贝。由于使用了浅拷贝，数组值可以在不调用各种头相关 http 模块方法的情况下被修改。返回对象的键是头名称，值是相应的头值。所有头名称都是小写的。

`request.getHeaders()` 方法返回的对象*不*从 JavaScript `Object` 原型继承。这意味着典型的 `Object` 方法，如 `obj.toString()`、`obj.hasOwnProperty()` 等未定义且*不起作用*。

```js
request.setHeader('Foo', 'bar');
request.setHeader('Cookie', ['foo=bar', 'bar=baz']);

const headers = request.getHeaders();
// headers === { foo: 'bar', 'cookie': ['foo=bar', 'bar=baz'] }
```

### `request.getRawHeaderNames()`

<!-- YAML
added:
  - v15.13.0
  - v14.17.0
-->

- 返回：{string\[]}

返回包含当前传出原始头的唯一名称的数组。头名称以其设置的确切大小写返回。

```js
request.setHeader('Foo', 'bar');
request.setHeader('Set-Cookie', ['foo=bar', 'bar=baz']);

const headerNames = request.getRawHeaderNames();
// headerNames === ['Foo', 'Set-Cookie']
```

### `request.hasHeader(name)`

<!-- YAML
added: v7.7.0
-->

- `name` {string}
- 返回：{boolean}

如果由 `name` 标识的头当前设置在传出头中，则返回 `true`。头名称匹配不区分大小写。

```js
const hasContentType = request.hasHeader('content-type');
```

### `request.maxHeadersCount`

- 类型：{number} **默认值：** `2000`

限制最大响应头数量。如果设置为 0，则不应用限制。

### `request.path`

<!-- YAML
added: v0.4.0
-->

- 类型：{string} 请求路径。

### `request.method`

<!-- YAML
added: v0.1.97
-->

- 类型：{string} 请求方法。

### `request.host`

<!-- YAML
added:
  - v14.5.0
  - v12.19.0
-->

- 类型：{string} 请求主机。

### `request.protocol`

<!-- YAML
added:
  - v14.5.0
  - v12.19.0
-->

- 类型：{string} 请求协议。

### `request.removeHeader(name)`

<!-- YAML
added: v1.6.0
-->

- `name` {string}

移除已定义到头对象中的头。

```js
request.removeHeader('Content-Type');
```

### `request.reusedSocket`

<!-- YAML
added:
 - v13.0.0
 - v12.16.0
-->

- 类型：{boolean} 请求是否通过复用的 socket 发送。

当通过启用 keep-alive 的代理发送请求时，底层 socket 可能会被复用。但如果服务器在不幸的时间关闭连接，客户端可能会遇到 'ECONNRESET' 错误。

```mjs
import http from 'node:http';

// 服务器默认有 5 秒的 keep-alive 超时
http
  .createServer((req, res) => {
    res.write('hello\n');
    res.end();
  })
  .listen(3000);

setInterval(() => {
  // 适配 keep-alive 代理
  http.get('http://localhost:3000', { agent }, (res) => {
    res.on('data', (data) => {
      // 什么都不做
    });
  });
}, 5000); // 以 5 秒间隔发送请求，因此很容易达到空闲超时
```

```cjs
const http = require('node:http');

// 服务器默认有 5 秒的 keep-alive 超时
http
  .createServer((req, res) => {
    res.write('hello\n');
    res.end();
  })
  .listen(3000);

setInterval(() => {
  // 适配 keep-alive 代理
  http.get('http://localhost:3000', { agent }, (res) => {
    res.on('data', (data) => {
      // 什么都不做
    });
  });
}, 5000); // 以 5 秒间隔发送请求，因此很容易达到空闲超时
```

通过标记请求是否复用了 socket，我们可以基于此进行自动错误重试。

```mjs
import http from 'node:http';
const agent = new http.Agent({ keepAlive: true });

function retriableRequest() {
  const req = http
    .get('http://localhost:3000', { agent }, (res) => {
      // ...
    })
    .on('error', (err) => {
      // 检查是否需要重试
      if (req.reusedSocket && err.code === 'ECONNRESET') {
        retriableRequest();
      }
    });
}

retriableRequest();
```

```cjs
const http = require('node:http');
const agent = new http.Agent({ keepAlive: true });

function retriableRequest() {
  const req = http
    .get('http://localhost:3000', { agent }, (res) => {
      // ...
    })
    .on('error', (err) => {
      // 检查是否需要重试
      if (req.reusedSocket && err.code === 'ECONNRESET') {
        retriableRequest();
      }
    });
}

retriableRequest();
```

### `request.setHeader(name, value)`

<!-- YAML
added: v1.6.0
-->

- `name` {string}
- `value` {any}

为头对象设置单个头值。如果此头在要发送的头中已存在，其值将被替换。在此处使用字符串数组发送具有相同名称的多个头。非字符串值将不经修改存储。因此，[`request.getHeader()`][] 可能返回非字符串值。但是，非字符串值将转换为字符串以进行网络传输。

```js
request.setHeader('Content-Type', 'application/json');
```

或

```js
request.setHeader('Cookie', ['type=ninja', 'language=javascript']);
```

当值为字符串时，如果它包含 `latin1` 编码之外的字符，将抛出异常。

如果您需要在值中传递 UTF-8 字符，请使用 [RFC 8187][] 标准对值进行编码。

```js
const filename = 'Rock 🎵.txt';
request.setHeader(
  'Content-Disposition',
  `attachment; filename*=utf-8''${encodeURIComponent(filename)}`
);
```

### `request.setNoDelay([noDelay])`

<!-- YAML
added: v0.5.9
-->

- `noDelay` {boolean}

一旦 socket 分配给此请求并连接，将调用 [`socket.setNoDelay()`][]。

### `request.setSocketKeepAlive([enable][, initialDelay])`

<!-- YAML
added: v0.5.9
-->

- `enable` {boolean}
- `initialDelay` {number}

一旦 socket 分配给此请求并连接，将调用 [`socket.setKeepAlive()`][]。

### `request.setTimeout(timeout[, callback])`

<!-- YAML
added: v0.5.9
changes:
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/8895
    description: Consistently set socket timeout only when the socket connects.
-->

- `timeout` {number} 请求超前的毫秒数。
- `callback` {Function} 超时发生时调用的可选函数。与绑定到 `'timeout'` 事件相同。
- 返回：{http.ClientRequest}

一旦 socket 分配给此请求并连接，将调用 [`socket.setTimeout()`][]。

### `request.socket`

<!-- YAML
added: v0.3.0
-->

- 类型：{stream.Duplex}

对底层 socket 的引用。通常用户不希望访问此属性。特别是，socket 不会发出 `'readable'` 事件，因为协议解析器如何附加到 socket。

```mjs
import http from 'node:http';
const options = {
  host: 'www.google.com',
};
const req = http.get(options);
req.end();
req.once('response', (res) => {
  const ip = req.socket.localAddress;
  const port = req.socket.localPort;
  console.log(`Your IP address is ${ip} and your source port is ${port}.`);
  // 消耗响应对象
});
```

```cjs
const http = require('node:http');
const options = {
  host: 'www.google.com',
};
const req = http.get(options);
req.end();
req.once('response', (res) => {
  const ip = req.socket.localAddress;
  const port = req.socket.localPort;
  console.log(`Your IP address is ${ip} and your source port is ${port}.`);
  // 消耗响应对象
});
```

此属性保证是 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

### `request.uncork()`

<!-- YAML
added:
 - v13.2.0
 - v12.16.0
-->

参见 [`writable.uncork()`][]。

### `request.writableEnded`

<!-- YAML
added: v12.9.0
-->

- 类型：{boolean}

在调用 [`request.end()`][] 后为 `true`。此属性不指示数据是否已刷新，为此请使用 [`request.writableFinished`][]。

### `request.writableFinished`

<!-- YAML
added: v12.7.0
-->

- 类型：{boolean}

如果所有数据都已刷新到底层系统，则在 [`'finish'`][] 事件发出之前立即为 `true`。

### `request.write(chunk[, encoding][, callback])`

<!-- YAML
added: v0.1.29
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/33155
    description: The `chunk` parameter can now be a `Uint8Array`.
-->

- `chunk` {string|Buffer|Uint8Array}
- `encoding` {string}
- `callback` {Function}
- 返回：{boolean}

发送一个主体块。此方法可以多次调用。如果未设置 `Content-Length`，数据将自动以 HTTP Chunked 传输编码编码，以便服务器知道数据何时结束。添加 `Transfer-Encoding: chunked` 头。调用 [`request.end()`][] 是完成发送请求所必需的。

`encoding` 参数是可选的，仅当 `chunk` 是字符串时适用。默认为 `'utf8'`。

`callback` 参数是可选的，将在数据块刷新时调用，但仅当数据块非空时。

如果所有数据都成功刷新到内核缓冲区，则返回 `true`。如果所有或部分数据在用户内存中排队，则返回 `false`。当缓冲区再次空闲时将发出 `'drain'`。

当使用空字符串或缓冲区调用 `write` 函数时，它不执行任何操作并等待更多输入。

## 类：`http.Server`

<!-- YAML
added: v0.1.17
-->

- 扩展：{net.Server}

### 事件：`'checkContinue'`

<!-- YAML
added: v0.3.0
-->

- `request` {http.IncomingMessage}
- `response` {http.ServerResponse}

每次收到带有 HTTP `Expect: 100-continue` 的请求时触发。如果未监听此事件，服务器将自动响应 `100 Continue`。

处理此事件涉及调用 [`response.writeContinue()`][] 如果客户端应继续发送请求体，或者生成适当的 HTTP 响应（例如 400 Bad Request）如果客户端不应继续发送请求体。

当此事件被触发和处理时，[`'request'`][] 事件将不会触发。

### 事件：`'checkExpectation'`

<!-- YAML
added: v5.5.0
-->

- `request` {http.IncomingMessage}
- `response` {http.ServerResponse}

每次收到带有 HTTP `Expect` 头的请求时触发，其中值不是 `100-continue`。如果未监听此事件，服务器将自动响应 `417 Expectation Failed`。

当此事件被触发和处理时，[`'request'`][] 事件将不会触发。

### 事件：`'clientError'`

<!-- YAML
added: v0.1.94
changes:
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/25605
    description: The default behavior will return a 431 Request Header
                 Fields Too Large if a HPE_HEADER_OVERFLOW error occurs.
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/17672
    description: The `rawPacket` is the current buffer that just parsed. Adding
                 this buffer to the error object of `'clientError'` event is to
                 make it possible that developers can log the broken packet.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/4557
    description: The default action of calling `.destroy()` on the `socket`
                 will no longer take place if there are listeners attached
                 for `'clientError'`.
-->

- `exception` {Error}
- `socket` {stream.Duplex}

如果客户端连接发出 `'error'` 事件，它将被转发到这里。此事件的监听器负责关闭/销毁底层 socket。例如，可能希望使用自定义 HTTP 响应更优雅地关闭 socket，而不是突然断开连接。监听器结束前**必须关闭或销毁** socket。

此事件保证传递一个 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

默认行为是尝试使用 HTTP '400 Bad Request' 关闭 socket，或者在 [`HPE_HEADER_OVERFLOW`][] 错误的情况下使用 HTTP '431 Request Header Fields Too Large'。如果 socket 不可写或当前附加的 [`http.ServerResponse`][] 的头已发送，则立即销毁它。

`socket` 是错误起源的 [`net.Socket`][] 对象。

```mjs
import http from 'node:http';

const server = http.createServer((req, res) => {
  res.end();
});
server.on('clientError', (err, socket) => {
  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});
server.listen(8000);
```

```cjs
const http = require('node:http');

const server = http.createServer((req, res) => {
  res.end();
});
server.on('clientError', (err, socket) => {
  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});
server.listen(8000);
```

当发生 `'clientError'` 事件时，没有 `request` 或 `response` 对象，因此任何发送的 HTTP 响应，包括响应头和有效载荷，*必须*直接写入 `socket` 对象。必须注意确保响应是正确格式化的 HTTP 响应消息。

`err` 是 `Error` 的实例，有两个额外的列：

- `bytesParsed`：Node.js 可能正确解析的请求数据包的字节数；
- `rawPacket`：当前请求的原始数据包。

在某些情况下，客户端已经收到响应和/或 socket 已经被销毁，例如在 `ECONNRESET` 错误的情况下。在尝试向 socket 发送数据之前，最好检查它是否仍然可写。

```js
server.on('clientError', (err, socket) => {
  if (err.code === 'ECONNRESET' || !socket.writable) {
    return;
  }

  socket.end('HTTP/1.1 400 Bad Request\r\n\r\n');
});
```

### 事件：`'close'`

<!-- YAML
added: v0.1.4
-->

当服务器关闭时触发。

### 事件：`'connect'`

<!-- YAML
added: v0.7.0
-->

- `request` {http.IncomingMessage} HTTP 请求的参数，如 [`'request'`][] 事件中所示
- `socket` {stream.Duplex} 服务器和客户端之间的网络 socket
- `head` {Buffer} 隧道流的第一个数据包（可能为空）

每次客户端请求 HTTP `CONNECT` 方法时触发。如果未监听此事件，则请求 `CONNECT` 方法的客户端将关闭其连接。

此事件保证传递一个 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

此事件触发后，请求的 socket 将没有 `'data'` 事件监听器，意味着需要绑定才能处理发送到该 socket 上的服务器的数据。

### 事件：`'connection'`

<!-- YAML
added: v0.1.0
-->

- `socket` {stream.Duplex}

当建立新的 TCP 流时触发此事件。`socket` 通常是 [`net.Socket`][] 类型的对象。通常用户不希望访问此事件。特别是，socket 不会发出 `'readable'` 事件，因为协议解析器如何附加到 socket。`socket` 也可以在 `request.socket` 处访问。

此事件也可以由用户显式触发以将连接注入 HTTP 服务器。在这种情况下，可以传递任何 [`Duplex`][] 流。

如果在此处调用 `socket.setTimeout()`，则当 socket 服务请求时（如果 `server.keepAliveTimeout` 非零），超时将被 `server.keepAliveTimeout` 替换。

此事件保证传递一个 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

### 事件：`'dropRequest'`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
-->

- `request` {http.IncomingMessage} HTTP 请求的参数，如 [`'request'`][] 事件中所示
- `socket` {stream.Duplex} 服务器和客户端之间的网络 socket

当 socket 上的请求数达到 `server.maxRequestsPerSocket` 的阈值时，服务器将丢弃新请求并触发 `'dropRequest'` 事件，然后向客户端发送 `503`。

### 事件：`'request'`

<!-- YAML
added: v0.1.0
-->

- `request` {http.IncomingMessage}
- `response` {http.ServerResponse}

每次有请求时触发。每个连接可能有多个请求（在 HTTP Keep-Alive 连接的情况下）。

### 事件：`'upgrade'`

<!-- YAML
added: v0.1.94
changes:
  - version: v24.9.0
    pr-url: https://github.com/nodejs/node/pull/59824
    description: Whether this event is fired can now be controlled by the
                 `shouldUpgradeCallback` and sockets will be destroyed
                 if upgraded while no event handler is listening.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19981
    description: Not listening to this event no longer causes the socket
                 to be destroyed if a client sends an Upgrade header.
-->


- `request` {http.IncomingMessage} HTTP 请求的参数，如 [`'request'`][] 事件中所示
- `socket` {stream.Duplex} 服务器和客户端之间的网络 socket
- `head` {Buffer} 升级流的第一个数据包（可能为空）

每次客户端的 HTTP 升级请求被接受时触发。默认情况下，所有 HTTP 升级请求都被忽略（即仅发出常规 `'request'` 事件，坚持正常的 HTTP 请求/响应流），除非您监听此事件，在这种情况下它们都被接受（即发出 `'upgrade'` 事件，未来的通信必须直接通过原始 socket 处理）。您可以使用服务器 `shouldUpgradeCallback` 选项更精确地控制这一点。

监听此事件是可选的，客户端不能坚持协议更改。

此事件触发后，请求的 socket 将没有 `'data'` 事件监听器，意味着需要绑定才能处理发送到该 socket 上的服务器的数据。

如果升级被 `shouldUpgradeCallback` 接受但没有注册事件处理程序，则 socket 被销毁，导致客户端立即关闭连接。

此事件保证传递一个 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

### `server.close([callback])`

<!-- YAML
added: v0.1.90
changes:
  - version:
      - v19.0.0
    pr-url: https://github.com/nodejs/node/pull/43522
    description: 该方法在返回前关闭空闲连接。

-->

- `callback` {Function}

停止服务器接受新连接，并关闭所有连接到该服务器但未发送请求或等待响应的连接。
参见 [`net.Server.close()`][]。

```js
const http = require('node:http');

const server = http.createServer({ keepAliveTimeout: 60000 }, (req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(
    JSON.stringify({
      data: 'Hello World!',
    })
  );
});

server.listen(8000);
// 10 秒后关闭服务器
setTimeout(() => {
  server.close(() => {
    console.log('server on port 8000 closed successfully');
  });
}, 10000);
```

### `server.closeAllConnections()`

<!-- YAML
added: v18.2.0
-->

关闭所有连接到该服务器的已建立 HTTP(S) 连接，包括连接到该服务器且正在发送请求或等待响应的活动连接。这*不*会销毁升级到不同协议的 socket，例如 WebSocket 或 HTTP/2。

> 这是关闭所有连接的强制方式，应谨慎使用。每当与 `server.close` 一起使用此方法时，建议在 `server.close` 之后调用此方法，以避免在此调用和 `server.close` 调用之间创建新连接的竞争条件。

```js
const http = require('node:http');

const server = http.createServer({ keepAliveTimeout: 60000 }, (req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(
    JSON.stringify({
      data: 'Hello World!',
    })
  );
});

server.listen(8000);
// 10 秒后关闭服务器
setTimeout(() => {
  server.close(() => {
    console.log('server on port 8000 closed successfully');
  });
  // 关闭所有连接，确保服务器成功关闭
  server.closeAllConnections();
}, 10000);
```

### `server.closeIdleConnections()`

<!-- YAML
added: v18.2.0
-->

关闭所有连接到该服务器但未发送请求或等待响应的连接。

> 从 Node.js 19.0.0 开始，不需要在与 `server.close` 结合使用此方法来回收 `keep-alive` 连接。使用它不会造成任何损害，并且对于需要支持早于 19.0.0 版本的库和应用程序来说，它可以确保向后兼容性。每当与 `server.close` 一起使用此方法时，建议在 `server.close` 之后调用此方法，以避免在此调用和 `server.close` 调用之间创建新连接的竞争条件。

```js
const http = require('node:http');

const server = http.createServer({ keepAliveTimeout: 60000 }, (req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(
    JSON.stringify({
      data: 'Hello World!',
    })
  );
});

server.listen(8000);
// 10 秒后关闭服务器
setTimeout(() => {
  server.close(() => {
    console.log('server on port 8000 closed successfully');
  });
  // 关闭空闲连接，例如 keep-alive 连接。一旦剩余的活动连接终止，服务器将关闭
  server.closeIdleConnections();
}, 10000);
```

### `server.headersTimeout`

<!-- YAML
added:
 - v11.3.0
 - v10.14.0
changes:
  - version:
    - v19.4.0
    - v18.14.0
    pr-url: https://github.com/nodejs/node/pull/45778
    description: 默认值现在设置为 [`server.requestTimeout`][] 或 `60000` 中的最小值。
-->

- 类型：{number} **默认值：** [`server.requestTimeout`][] 或 `60000` 中的最小值。

限制解析器等待接收完整 HTTP 头的时间。

如果超时，服务器以状态 408 响应，而不将请求转发到请求监听器，然后关闭连接。

必须设置为非零值（例如 120 秒）以保护在服务器部署时没有反向代理在前面的情况下免受潜在的拒绝服务攻击。

### `server.listen()`

启动 HTTP 服务器监听连接。
此方法与 [`net.Server`][] 的 [`server.listen()`][] 相同。

### `server.listening`

<!-- YAML
added: v5.7.0
-->

- 类型：{boolean} 指示服务器是否正在监听连接。

### `server.maxHeadersCount`

<!-- YAML
added: v0.7.0
-->

- 类型：{number} **默认值：** `2000`

限制传入头的最大数量。如果设置为 0，则不应用限制。

### `server.requestTimeout`

<!-- YAML
added: v14.11.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41263
    description: 默认请求超时从无超时更改为 300 秒（5 分钟）。
-->

- 类型：{number} **默认值：** `300000`

设置从客户端接收整个请求的超时值（毫秒）。

如果超时，服务器以状态 408 响应，而不将请求转发到请求监听器，然后关闭连接。

必须设置为非零值（例如 120 秒）以保护在服务器部署时没有反向代理在前面的情况下免受潜在的拒绝服务攻击。

### `server.setTimeout([msecs][, callback])`

<!-- YAML
added: v0.9.12
changes:
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/27558
    description: 默认超时从 120 秒更改为 0（无超时）。
-->

- `msecs` {number} **默认值：** 0（无超时）
- `callback` {Function}
- 返回：{http.Server}

设置 socket 的超时值，并在服务器对象上发出 `'timeout'` 事件，将 socket 作为参数传递，如果发生超时。

如果服务器对象上有 `'timeout'` 事件监听器，则它将使用超时的 socket 作为参数调用。

默认情况下，服务器不会使 socket 超时。但是，如果为服务器的 `'timeout'` 事件分配了回调，则必须显式处理超时。

### `server.maxRequestsPerSocket`

<!-- YAML
added: v16.10.0
-->

- 类型：{number} 每个 socket 的请求数。**默认值：** 0（无限制）

socket 在关闭 keep-alive 连接之前可以处理的最大请求数。

值为 `0` 将禁用限制。

当达到限制时，它将设置 `Connection` 头值为 `close`，但不会实际关闭连接，在达到限制后发送的后续请求将得到 `503 Service Unavailable` 作为响应。

### `server.timeout`

<!-- YAML
added: v0.9.12
changes:
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/27558
    description: 默认超时从 120 秒更改为 0（无超时）。
-->

- 类型：{number} 超时（毫秒）。**默认值：** 0（无超时）

socket 被假定为超时之前的不活动毫秒数。

值为 `0` 将禁用传入连接的超时行为。

socket 超时逻辑在连接时设置，因此更改此值仅影响服务器的新连接，而不影响任何现有连接。

### `server.keepAliveTimeout`

<!-- YAML
added: v8.0.0
-->

- 类型：{number} 超时（毫秒）。**默认值：** `5000`（5 秒）。

服务器在完成写入最后一个响应后需要等待额外传入数据的不活动毫秒数，然后 socket 将被销毁。

此超时值与 [`server.keepAliveTimeoutBuffer`][] 选项结合以确定实际的 socket 超时，计算为：
socketTimeout = keepAliveTimeout + keepAliveTimeoutBuffer
如果服务器在 keep-alive 超时触发之前收到新数据，它将重置常规不活动超时，即 [`server.timeout`][]。

值为 `0` 将禁用传入连接的 keep-alive 超时行为。
值为 `0` 使 HTTP 服务器的行为类似于 8.0.0 之前的 Node.js 版本，这些版本没有 keep-alive 超时。

socket 超时逻辑在连接时设置，因此更改此值仅影响服务器的新连接，而不影响任何现有连接。

### `server.keepAliveTimeoutBuffer`

<!-- YAML
added: v24.6.0
-->

- 类型：{number} 超时（毫秒）。**默认值：** `1000`（1 秒）。

添加到 [`server.keepAliveTimeout`][] 的额外缓冲时间，以延长内部 socket 超时。

此缓冲区通过将 socket 超时稍微超出广告的 keep-alive 超时来帮助减少连接重置（`ECONNRESET`）错误。

此选项仅适用于新的传入连接。

### `server[Symbol.asyncDispose]()`

<!-- YAML
added: v20.4.0
changes:
 - version: v24.2.0
   pr-url: https://github.com/nodejs/node/pull/58467
   description: 不再实验性。
-->

调用 [`server.close()`][] 并返回一个在服务器关闭时完成的 promise。

## 类：`http.ServerResponse`

<!-- YAML
added: v0.1.17
-->

- 扩展：{http.OutgoingMessage}

此对象由 HTTP 服务器在内部创建，而不是由用户创建。它作为第二个参数传递给 [`'request'`][] 事件。

### 事件：`'close'`

<!-- YAML
added: v0.6.7
-->

指示响应已完成，或其底层连接过早终止（在响应完成之前）。

### 事件：`'finish'`

<!-- YAML
added: v0.3.6
-->

当响应已发送时触发。更具体地说，当响应头和主体的最后一段已移交操作系统通过网络传输时触发此事件。这并不意味着客户端已收到任何内容。

### `response.addTrailers(headers)`

<!-- YAML
added: v0.3.0
-->

- `headers` {Object}

此方法向响应添加 HTTP 尾部头（消息末尾的头）。

仅当响应使用分块编码时才会发出尾部；否则（例如，如果请求是 HTTP/1.0），它们将被静默丢弃。

HTTP 要求发送 `Trailer` 头以发出尾部，其值中包含头字段列表。例如，

```js
response.writeHead(200, {
  'Content-Type': 'text/plain',
  Trailer: 'Content-MD5',
});
response.write(fileData);
response.addTrailers({ 'Content-MD5': '7895bf4b8828b55ceaf47747b4bca667' });
response.end();
```

尝试设置包含无效字符的头字段名称或值将导致抛出 [`TypeError`][]。

### `response.connection`

<!-- YAML
added: v0.3.0
deprecated: v13.0.0
-->

> Stability: 0 - 已弃用。使用 [`response.socket`][]。

- 类型：{stream.Duplex}

参见 [`response.socket`][]。

### `response.cork()`

<!-- YAML
added:
 - v13.2.0
 - v12.16.0
-->

参见 [`writable.cork()`][]。

### `response.end([data[, encoding]][, callback])`

<!-- YAML
added: v0.1.90
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/33155
    description: `data` 参数现在可以是 `Uint8Array`。
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18780
    description: 此方法现在返回对 `ServerResponse` 的引用。
-->

- `data` {string|Buffer|Uint8Array}
- `encoding` {string}
- `callback` {Function}
- 返回：{this}

此方法向服务器发出信号，表明所有响应头和主体都已发送；服务器应认为此消息已完成。方法 `response.end()` 必须在每个响应上调用。

如果指定了 `data`，则相当于调用 [`response.write(data, encoding)`][] 后跟 `response.end(callback)`。

如果指定了 `callback`，它将在响应流完成时调用。

### `response.finished`

<!-- YAML
added: v0.0.2
deprecated:
 - v13.4.0
 - v12.16.0
-->

> Stability: 0 - 已弃用。使用 [`response.writableEnded`][]。

- 类型：{boolean}

如果已调用 [`response.end()`][]，`response.finished` 属性将为 `true`。

### `response.flushHeaders()`

<!-- YAML
added: v1.6.0
-->

刷新响应头。另请参见：[`request.flushHeaders()`][]。

### `response.getHeader(name)`

<!-- YAML
added: v0.4.0
-->

- `name` {string}
- 返回：{number | string | string\[] | undefined}

读取已排队但尚未发送到客户端的头。名称不区分大小写。返回值的类型取决于提供给 [`response.setHeader()`][] 的参数。

```js
response.setHeader('Content-Type', 'text/html');
response.setHeader('Content-Length', Buffer.byteLength(body));
response.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
const contentType = response.getHeader('content-type');
// contentType 是 'text/html'
const contentLength = response.getHeader('Content-Length');
// contentLength 是数字类型
const setCookie = response.getHeader('set-cookie');
// setCookie 是字符串数组类型
```

### `response.getHeaderNames()`

<!-- YAML
added: v7.7.0
-->

- 返回：{string\[]}

返回包含当前传出头的唯一名称的数组。所有头名称都是小写的。

```js
response.setHeader('Foo', 'bar');
response.setHeader('Set-Cookie', ['foo=bar', 'bar=baz']);

const headerNames = response.getHeaderNames();
// headerNames === ['foo', 'set-cookie']
```

### `response.getHeaders()`

<!-- YAML
added: v7.7.0
-->

- 返回：{Object}

返回当前传出头的浅拷贝。由于使用了浅拷贝，数组值可以在不调用各种头相关 http 模块方法的情况下被修改。返回对象的键是头名称，值是相应的头值。所有头名称都是小写的。

`response.getHeaders()` 方法返回的对象*不*从 JavaScript `Object` 原型继承。这意味着典型的 `Object` 方法，如 `obj.toString()`、`obj.hasOwnProperty()` 等未定义且*不起作用*。

```js
response.setHeader('Foo', 'bar');
response.setHeader('Set-Cookie', ['foo=bar', 'bar=baz']);

const headers = response.getHeaders();
// headers === { foo: 'bar', 'set-cookie': ['foo=bar', 'bar=baz'] }
```

### `response.hasHeader(name)`

<!-- YAML
added: v7.7.0
-->

- `name` {string}
- 返回：{boolean}

如果由 `name` 标识的头当前设置在传出头中，则返回 `true`。头名称匹配不区分大小写。

```js
const hasContentType = response.hasHeader('content-type');
```

### `response.headersSent`

<!-- YAML
added: v0.9.3
-->

- 类型：{boolean}

布尔值（只读）。如果头已发送，则为 true，否则为 false。

### `response.removeHeader(name)`

<!-- YAML
added: v0.4.0
-->

- `name` {string}

移除排队等待隐式发送的头。

```js
response.removeHeader('Content-Encoding');
```

### `response.req`

<!-- YAML
added: v15.7.0
-->

- 类型：{http.IncomingMessage}

对原始 HTTP `request` 对象的引用。

### `response.sendDate`

<!-- YAML
added: v0.7.5
-->

- 类型：{boolean}

当为 true 时，如果头中尚未存在 Date 头，则会自动生成并在响应中发送。默认为 true。

这应仅用于测试；HTTP 要求在响应中包含 Date 头。

### `response.setHeader(name, value)`

<!-- YAML
added: v0.4.0
-->

- `name` {string}
- `value` {number | string | string\[]}
- 返回：{http.ServerResponse}

返回响应对象。

为隐式头设置单个头值。如果此头在要发送的头中已存在，其值将被替换。在此处使用字符串数组发送具有相同名称的多个头。非字符串值将不经修改存储。因此，[`response.getHeader()`][] 可能返回非字符串值。但是，非字符串值将转换为字符串以进行网络传输。相同的响应对象返回给调用者，以启用调用链。

```js
response.setHeader('Content-Type', 'text/html');
```

或

```js
response.setHeader('Set-Cookie', ['type=ninja', 'language=javascript']);
```

尝试设置包含无效字符的头字段名称或值将导致抛出 [`TypeError`][]。

当头已使用 [`response.setHeader()`][] 设置时，它们将与传递给 [`response.writeHead()`][] 的任何头合并，传递给 [`response.writeHead()`][] 的头优先。

```js
// 返回 content-type = text/plain
const server = http.createServer((req, res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('ok');
});
```

如果调用 [`response.writeHead()`][] 方法且未调用此方法，它将直接在不缓存内部的情况下将提供的头值写入网络通道，并且在头上的 [`response.getHeader()`][] 不会产生预期结果。如果希望逐步填充头并可能未来检索和修改，请使用 [`response.setHeader()`][] 而不是 [`response.writeHead()`][]。

### `response.setTimeout(msecs[, callback])`

<!-- YAML
added: v0.9.12
-->

- `msecs` {number}
- `callback` {Function}
- 返回：{http.ServerResponse}

将 Socket 的超时值设置为 `msecs`。如果提供了回调，则将其添加为响应对象上 `'timeout'` 事件的监听器。

如果未将 `'timeout'` 监听器添加到请求、响应或服务器，则 socket 在超时会被销毁。如果为请求、响应或服务器的 `'timeout'` 事件分配了处理程序，则必须显式处理超时的 socket。

### `response.socket`

<!-- YAML
added: v0.3.0
-->

- 类型：{stream.Duplex}

对底层 socket 的引用。通常用户不希望访问此属性。特别是，socket 不会发出 `'readable'` 事件，因为协议解析器如何附加到 socket。在 `response.end()` 之后，此属性为 null。

```mjs
import http from 'node:http';
const server = http
  .createServer((req, res) => {
    const ip = res.socket.remoteAddress;
    const port = res.socket.remotePort;
    res.end(`Your IP address is ${ip} and your source port is ${port}.`);
  })
  .listen(3000);
```

```cjs
const http = require('node:http');
const server = http
  .createServer((req, res) => {
    const ip = res.socket.remoteAddress;
    const port = res.socket.remotePort;
    res.end(`Your IP address is ${ip} and your source port is ${port}.`);
  })
  .listen(3000);
```

此属性保证是 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型。

### `response.statusCode`

<!-- YAML
added: v0.4.0
-->

- 类型：{number} **默认值：** `200`

当使用隐式头（不显式调用 [`response.writeHead()`][]）时，此属性控制头刷新时将发送到客户端的状态码。

```js
response.statusCode = 404;
```

响应头发送到客户端后，此属性指示已发送的状态码。

### `response.statusMessage`

<!-- YAML
added: v0.11.8
-->

- 类型：{string}

当使用隐式头（不显式调用 [`response.writeHead()`][]）时，此属性控制头刷新时将发送到客户端的状态消息。如果此项保留为 `undefined`，则将使用状态码的标准消息。

```js
response.statusMessage = 'Not found';
```

响应头发送到客户端后，此属性指示已发送的状态消息。

### `response.strictContentLength`

<!-- YAML
added:
  - v18.10.0
  - v16.18.0
-->

- 类型：{boolean} **默认值：** `false`

如果设置为 `true`，Node.js 将检查 `Content-Length` 头值和主体的字节大小是否相等。不匹配 `Content-Length` 头值将导致抛出 `Error`，标识为 `code:` [`'ERR_HTTP_CONTENT_LENGTH_MISMATCH'`][]。

### `response.uncork()`

<!-- YAML
added:
 - v13.2.0
 - v12.16.0
-->

参见 [`writable.uncork()`][]。

### `response.writableEnded`

<!-- YAML
added: v12.9.0
-->

- 类型：{boolean}

在调用 [`response.end()`][] 后为 `true`。此属性不指示数据是否已刷新，为此请使用 [`response.writableFinished`][]。

### `response.writableFinished`

<!-- YAML
added: v12.7.0
-->

- 类型：{boolean}

如果所有数据都已刷新到底层系统，则在 [`'finish'`][] 事件发出之前立即为 `true`。

### `response.write(chunk[, encoding][, callback])`

<!-- YAML
added: v0.1.29
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/33155
    description: `chunk` 参数现在可以是 `Uint8Array`。
-->

- `chunk` {string|Buffer|Uint8Array}
- `encoding` {string} **默认值：** `'utf8'`
- `callback` {Function}
- 返回：{boolean}

如果调用此方法且尚未调用 [`response.writeHead()`][]，它将切换到隐式头模式并刷新隐式头。

这将发送一个响应主体块。此方法可以多次调用以提供连续的主体部分。

如果在 `createServer` 中将 `rejectNonStandardBodyWrites` 设置为 true，则当请求方法或响应状态不支持内容时，不允许写入主体。如果尝试为 HEAD 请求或作为 `204` 或 `304` 响应的一部分写入主体，将抛出带有代码 `ERR_HTTP_BODY_NOT_ALLOWED` 的同步 `Error`。

`chunk` 可以是字符串或缓冲区。如果 `chunk` 是字符串，第二个参数指定如何将其编码为字节流。`callback` 将在数据块刷新时调用。

这是原始 HTTP 主体，与可能使用的更高级别的多部分主体编码无关。

第一次调用 [`response.write()`][] 时，它将发送缓冲的头信息和主体的第一个块到客户端。第二次调用 [`response.write()`][] 时，Node.js 假定数据将被流式传输，并单独发送新数据。也就是说，响应被缓冲到主体的第一个块。

如果所有数据都成功刷新到内核缓冲区，则返回 `true`。如果所有或部分数据在用户内存中排队，则返回 `false`。当缓冲区再次空闲时将发出 `'drain'`。

### `response.writeContinue()`

<!-- YAML
added: v0.3.0
-->

向客户端发送 HTTP/1.1 100 Continue 消息，指示应发送请求体。参见 `Server` 上的 [`'checkContinue'`][] 事件。

### `response.writeEarlyHints(hints[, callback])`

<!-- YAML
added: v18.11.0
changes:
  - version: v18.11.0
    pr-url: https://github.com/nodejs/node/pull/44820
    description: 允许将 hints 作为对象传递。
-->

- `hints` {Object}
- `callback` {Function}

向客户端发送带有 Link 头的 HTTP/1.1 103 Early Hints 消息，指示用户代理可以预加载/预连接链接的资源。`hints` 是一个包含要随早期提示消息发送的头值的对象。可选的 `callback` 参数将在响应消息写入后调用。

**示例**

```js
const earlyHintsLink = '</styles.css>; rel=preload; as=style';
response.writeEarlyHints({
  link: earlyHintsLink,
});

const earlyHintsLinks = [
  '</styles.css>; rel=preload; as=style',
  '</scripts.js>; rel=preload; as=script',
];
response.writeEarlyHints({
  link: earlyHintsLinks,
  'x-trace-id': 'id for diagnostics',
});

const earlyHintsCallback = () => console.log('early hints message sent');
response.writeEarlyHints(
  {
    link: earlyHintsLinks,
  },
  earlyHintsCallback
);
```

### `response.writeHead(statusCode[, statusMessage][, headers])`

<!-- YAML
added: v0.1.30
changes:
  - version: v14.14.0
    pr-url: https://github.com/nodejs/node/pull/35274
    description: 允许将 headers 作为数组传递。
  - version:
     - v11.10.0
     - v10.17.0
    pr-url: https://github.com/nodejs/node/pull/25974
    description: 从 `writeHead()` 返回 `this` 以允许与 `end()` 链式调用。
  - version:
    - v5.11.0
    - v4.4.5
    pr-url: https://github.com/nodejs/node/pull/6291
    description: 如果 `statusCode` 不是 `[100, 999]` 范围内的数字，则抛出 `RangeError`。
-->

- `statusCode` {number}
- `statusMessage` {string}
- `headers` {Object|Array}
- 返回：{http.ServerResponse}

向请求发送响应头。状态码是一个 3 位 HTTP 状态码，如 `404`。最后一个参数 `headers` 是响应头。可以选择性地将人类可读的 `statusMessage` 作为第二个参数给出。

`headers` 可以是一个 `Array`，其中键和值在同一个列表中。它*不是*元组列表。因此，偶数偏移是键值，奇数偏移是关联的值。数组的格式与 `request.rawHeaders` 相同。

返回对 `ServerResponse` 的引用，以便可以链式调用。

```js
const body = 'hello world';
response
  .writeHead(200, {
    'Content-Length': Buffer.byteLength(body),
    'Content-Type': 'text/plain',
  })
  .end(body);
```

此方法在消息上必须仅调用一次，并且必须在调用 [`response.end()`][] 之前调用。

如果在调用此方法之前调用了 [`response.write()`][] 或 [`response.end()`][]，则将计算隐式/可变头并调用此函数。

当头已使用 [`response.setHeader()`][] 设置时，它们将与传递给 [`response.writeHead()`][] 的任何头合并，传递给 [`response.writeHead()`][] 的头优先。

如果调用此方法且尚未调用 [`response.setHeader()`][]，它将直接在不缓存内部的情况下将提供的头值写入网络通道，并且在头上的 [`response.getHeader()`][] 不会产生预期结果。如果希望逐步填充头并可能未来检索和修改，请使用 [`response.setHeader()`][] 代替。

```js
// 返回 content-type = text/plain
const server = http.createServer((req, res) => {
  res.setHeader('Content-Type', 'text/html');
  res.setHeader('X-Foo', 'bar');
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('ok');
});
```

`Content-Length` 以字节读取，而不是字符。使用 [`Buffer.byteLength()`][] 来确定主体的字节长度。Node.js 将检查 `Content-Length` 和已传输的主体长度是否相等。

尝试设置包含无效字符的头字段名称或值将导致抛出 [`TypeError`][]。

### `response.writeProcessing()`

<!-- YAML
added: v10.0.0
-->

向客户端发送 HTTP/1.1 102 Processing 消息，指示应发送请求体。

## 类：`http.IncomingMessage`

<!-- YAML
added: v0.1.17
changes:
  - version: v15.5.0
    pr-url: https://github.com/nodejs/node/pull/33035
    description: 在传入数据被消耗后，`destroyed` 值返回 `true`。
  - version:
     - v13.1.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30135
    description: `readableHighWaterMark` 值反映 socket 的值。
-->

- 扩展：{stream.Readable}

`IncomingMessage` 对象由 [`http.Server`][] 或 [`http.ClientRequest`][] 创建，并分别作为第一个参数传递给 [`'request'`][] 和 [`'response'`][] 事件。它可用于访问响应状态、头和数据。

与其 `socket` 值（它是 {stream.Duplex} 的子类）不同，`IncomingMessage` 本身扩展了 {stream.Readable}，并单独创建以解析和发出传入的 HTTP 头和有效载荷，因为底层 socket 在 keep-alive 的情况下可能会被多次重用。

### 事件：`'aborted'`

<!-- YAML
added: v0.3.8
deprecated:
  - v17.0.0
  - v16.12.0
-->

> Stability: 0 - 已弃用。请监听 `'close'` 事件。

当请求被中止时触发。

### 事件：`'close'`

<!-- YAML
added: v0.4.2
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/33035
    description: 现在在请求完成时发出 close 事件，而不是在底层 socket 关闭时。
-->

当请求完成时触发。

### `message.aborted`

<!-- YAML
added: v10.1.0
deprecated:
  - v17.0.0
  - v16.12.0
-->

> Stability: 0 - 已弃用。从 {stream.Readable} 检查 `message.destroyed`。

- 类型：{boolean}

如果请求已中止，`message.aborted` 属性将为 `true`。

### `message.complete`

<!-- YAML
added: v0.3.0
-->

- 类型：{boolean}

如果已接收并成功解析完整的 HTTP 消息，`message.complete` 属性将为 `true`。

此属性特别适用于确定客户端或服务器在连接终止之前是否完全传输了消息：

```js
const req = http.request(
  {
    host: '127.0.0.1',
    port: 8080,
    method: 'POST',
  },
  (res) => {
    res.resume();
    res.on('end', () => {
      if (!res.complete)
        console.error(
          'The connection was terminated while the message was still being sent'
        );
    });
  }
);
```

### `message.connection`

<!-- YAML
added: v0.1.90
deprecated: v16.0.0
 -->

> Stability: 0 - 已弃用。使用 [`message.socket`][]。

[`message.socket`][] 的别名。

### `message.destroy([error])`

<!-- YAML
added: v0.3.0
changes:
  - version:
    - v14.5.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/32789
    description: 该函数返回 `this` 以与其他 Readable 流保持一致。
-->

- `error` {Error}
- 返回：{this}

在接收 `IncomingMessage` 的 socket 上调用 `destroy()`。如果提供了 `error`，将在 socket 上发出 `'error'` 事件，并将 `error` 作为参数传递给该事件上的任何监听器。

### `message.headers`

<!-- YAML
added: v0.1.5
changes:
  - version:
    - v19.5.0
    - v18.14.0
    pr-url: https://github.com/nodejs/node/pull/45982
    description: >-
     `http.request()` 和 `http.createServer()` 函数中的 `joinDuplicateHeaders` 选项确保重复头不会被丢弃，而是使用逗号分隔符合并，符合 RFC 9110 第 5.3 节。
  - version: v15.1.0
    pr-url: https://github.com/nodejs/node/pull/35281
    description: >-
      `message.headers` 现在使用原型上的访问器属性延迟计算，并且不再可枚举。
-->

- 类型：{Object}

请求/响应头对象。

头名称和值的键值对。头名称是小写的。

```js
// 打印类似以下内容：
//
// { 'user-agent': 'curl/7.22.0',
//   host: '127.0.0.1:8000',
//   accept: '*/*' }
console.log(request.headers);
```

原始头中的重复项根据头名称以以下方式处理：

- `age`、`authorization`、`content-length`、`content-type`、`etag`、`expires`、`from`、`host`、`if-modified-since`、`if-unmodified-since`、`last-modified`、`location`、`max-forwards`、`proxy-authorization`、`referer`、`retry-after`、`server` 或 `user-agent` 的重复项将被丢弃。
  要允许将上述列表中的头的重复值连接起来，请在 [`http.request()`][] 和 [`http.createServer()`][] 中使用 `joinDuplicateHeaders` 选项。有关更多信息，请参见 RFC 9110 第 5.3 节。
- `set-cookie` 始终是一个数组。重复项将添加到数组中。
- 对于重复的 `cookie` 头，值使用 `; ` 连接。
- 对于所有其他头，值使用 `, ` 连接。

### `message.headersDistinct`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

- 类型：{Object}

类似于 [`message.headers`][]，但没有连接逻辑，值始终是字符串数组，即使头只接收一次。

```js
// 打印类似以下内容：
//
// { 'user-agent': ['curl/7.22.0'],
//   host: ['127.0.0.1:8000'],
//   accept: ['*/*'] }
console.log(request.headersDistinct);
```

### `message.httpVersion`

<!-- YAML
added: v0.1.1
-->

- 类型：{string}

在服务器请求的情况下，客户端发送的 HTTP 版本。在客户端响应的情况下，连接到的服务器的 HTTP 版本。可能是 `'1.1'` 或 `'1.0'`。

还有 `message.httpVersionMajor` 是第一个整数，`message.httpVersionMinor` 是第二个。

### `message.method`

<!-- YAML
added: v0.1.1
-->

- 类型：{string}

**仅对从 [`http.Server`][] 获取的请求有效。**

请求方法作为字符串。只读。示例：`'GET'`、`'DELETE'`。

### `message.rawHeaders`

<!-- YAML
added: v0.11.6
-->

- 类型：{string\[]}

原始请求/响应头列表，完全按照接收到的样子。

键和值在同一个列表中。它*不是*元组列表。因此，偶数偏移是键值，奇数偏移是关联的值。

头名称不是小写的，重复项不会合并。

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

### `message.rawTrailers`

<!-- YAML
added: v0.11.6
-->

- 类型：{string\[]}

原始请求/响应尾部键和值，完全按照接收到的样子。仅在 `'end'` 事件时填充。

### `message.setTimeout(msecs[, callback])`

<!-- YAML
added: v0.5.9
-->

- `msecs` {number}
- `callback` {Function}
- 返回：{http.IncomingMessage}

调用 `message.socket.setTimeout(msecs, callback)`。

### `message.socket`

<!-- YAML
added: v0.3.0
-->

- 类型：{stream.Duplex}

与连接关联的 [`net.Socket`][] 对象。

使用 HTTPS 支持时，使用 [`request.socket.getPeerCertificate()`][] 获取客户端的身份验证详细信息。

此属性保证是 {net.Socket} 类的实例，它是 {stream.Duplex} 的子类，除非用户指定了 {net.Socket} 以外的 socket 类型或在内部为 null。

### `message.statusCode`

<!-- YAML
added: v0.1.1
-->

- 类型：{number}

**仅对从 [`http.ClientRequest`][] 获取的响应有效。**

3 位 HTTP 响应状态码。例如 `404`。

### `message.statusMessage`

<!-- YAML
added: v0.11.10
-->

- 类型：{string}

**仅对从 [`http.ClientRequest`][] 获取的响应有效。**

HTTP 响应状态消息（原因短语）。例如 `OK` 或 `Internal Server Error`。

### `message.trailers`

<!-- YAML
added: v0.3.0
-->

- 类型：{Object}

请求/响应尾部对象。仅在 `'end'` 事件时填充。

### `message.trailersDistinct`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

- 类型：{Object}

类似于 [`message.trailers`][]，但没有连接逻辑，值始终是字符串数组，即使头只接收一次。仅在 `'end'` 事件时填充。

### `message.url`

<!-- YAML
added: v0.1.90
-->

- 类型：{string}

**仅对从 [`http.Server`][] 获取的请求有效。**

请求 URL 字符串。这仅包含实际 HTTP 请求中存在的 URL。以以下请求为例：

```http
GET /status?name=ryan HTTP/1.1
Accept: text/plain
```

要将 URL 解析为其部分：

```js
new URL(`http://${process.env.HOST ?? 'localhost'}${request.url}`);
```

当 `request.url` 是 `'/status?name=ryan'` 且 `process.env.HOST` 未定义时：

```console
$ node
> new URL(`http://${process.env.HOST ?? 'localhost'}${request.url}`);
URL {
  href: 'http://localhost/status?name=ryan',
  origin: 'http://localhost',
  protocol: 'http:',
  username: '',
  password: '',
  host: 'localhost',
  hostname: 'localhost',
  port: '',
  pathname: '/status',
  search: '?name=ryan',
  searchParams: URLSearchParams { 'name' => 'ryan' },
  hash: ''
}
```

确保将 `process.env.HOST` 设置为服务器的主机名，或考虑完全替换此部分。如果使用 `req.headers.host`，请确保使用适当的验证，因为客户端可能指定自定义 `Host` 头。

## 类：`http.OutgoingMessage`

<!-- YAML
added: v0.1.17
-->

- 扩展：{Stream}

此类用作 [`http.ClientRequest`][] 和 [`http.ServerResponse`][] 的父类。从 HTTP 事务参与者的角度来看，它是一个抽象的传出消息。

### 事件：`'drain'`

<!-- YAML
added: v0.3.6
-->

当消息的缓冲区再次空闲时触发。

### 事件：`'finish'`

<!-- YAML
added: v0.1.17
-->

当传输成功完成时触发。

### 事件：`'prefinish'`

<!-- YAML
added: v0.11.6
-->

在调用 `outgoingMessage.end()` 后触发。当事件触发时，所有数据已被处理但不一定完全刷新。

### `outgoingMessage.addTrailers(headers)`

<!-- YAML
added: v0.3.0
-->

- `headers` {Object}

向消息添加 HTTP 尾部头（消息末尾的头）。

尾部**仅**在消息使用分块编码时才会发出。否则，尾部将被静默丢弃。

HTTP 要求发送 `Trailer` 头以发出尾部，其值中包含头字段列表，例如：

```js
message.writeHead(200, {
  'Content-Type': 'text/plain',
  Trailer: 'Content-MD5',
});
message.write(fileData);
message.addTrailers({ 'Content-MD5': '7895bf4b8828b55ceaf47747b4bca667' });
message.end();
```

尝试设置包含无效字符的头字段名称或值将导致抛出 `TypeError`。

### `outgoingMessage.appendHeader(name, value)`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

- `name` {string} 头名称
- `value` {string|string\[]} 头值
- 返回：{this}

向头对象追加单个头值。

如果值是一个数组，这相当于多次调用此方法。

如果头没有先前的值，这相当于调用 [`outgoingMessage.setHeader(name, value)`][]。

根据创建客户端请求或服务器时 `options.uniqueHeaders` 的值，这将导致头被多次发送或使用 `; ` 连接值发送一次。

### `outgoingMessage.connection`

<!-- YAML
added: v0.3.0
deprecated:
  - v15.12.0
  - v14.17.1
-->

> Stability: 0 - 已弃用：使用 [`outgoingMessage.socket`][] 代替。

[`outgoingMessage.socket`][] 的别名。

### `outgoingMessage.cork()`

<!-- YAML
added:
  - v13.2.0
  - v12.16.0
-->

参见 [`writable.cork()`][]。

### `outgoingMessage.destroy([error])`

<!-- YAML
added: v0.3.0
-->

- `error` {Error} 可选，与 `error` 事件一起发出的错误
- 返回：{this}

销毁消息。一旦 socket 与消息关联并连接，该 socket 也将被销毁。

### `outgoingMessage.end(chunk[, encoding][, callback])`

<!-- YAML
added: v0.1.90
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/33155
    description: `chunk` 参数现在可以是 `Uint8Array`。
  - version: v0.11.6
    description: 添加 `callback` 参数。
-->

- `chunk` {string|Buffer|Uint8Array}
- `encoding` {string} 可选，**默认值：** `utf8`
- `callback` {Function} 可选
- 返回：{this}

完成传出消息。如果主体的任何部分未发送，它将将它们刷新到底层系统。如果消息是分块的，它将发送终止块 `0\r\n\r\n`，并发送尾部（如果有）。

如果指定了 `chunk`，则相当于调用 `outgoingMessage.write(chunk, encoding)`，后跟 `outgoingMessage.end(callback)`。

如果提供了 `callback`，它将在消息完成时调用（相当于 `'finish'` 事件的监听器）。

### `outgoingMessage.flushHeaders()`

<!-- YAML
added: v1.6.0
-->

刷新消息头。

出于效率原因，Node.js 通常缓冲消息头，直到调用 `outgoingMessage.end()` 或写入第一个消息数据块。然后它尝试将头和数据打包到单个 TCP 数据包中。

这通常是需要的（它节省了一次 TCP 往返），但当第一个数据可能直到很晚才发送时则不需要。`outgoingMessage.flushHeaders()` 绕过优化并启动消息。

### `outgoingMessage.getHeader(name)`

<!-- YAML
added: v0.4.0
-->

- `name` {string} 头名称
- 返回：{number | string | string\[] | undefined}

获取给定名称的 HTTP 头的值。如果未设置该头，则返回的值将是 `undefined`。

### `outgoingMessage.getHeaderNames()`

<!-- YAML
added: v7.7.0
-->

- 返回：{string\[]}

返回包含当前传出头的唯一名称的数组。所有名称都是小写的。

### `outgoingMessage.getHeaders()`

<!-- YAML
added: v7.7.0
-->

- 返回：{Object}

返回当前传出头的浅拷贝。由于使用了浅拷贝，数组值可以在不调用各种头相关 HTTP 模块方法的情况下被修改。返回对象的键是头名称，值是相应的头值。所有头名称都是小写的。

`outgoingMessage.getHeaders()` 方法返回的对象不从 JavaScript `Object` 原型继承。这意味着典型的 `Object` 方法，如 `obj.toString()`、`obj.hasOwnProperty()` 等未定义且不起作用。

```js
outgoingMessage.setHeader('Foo', 'bar');
outgoingMessage.setHeader('Set-Cookie', ['foo=bar', 'bar=baz']);

const headers = outgoingMessage.getHeaders();
// headers === { foo: 'bar', 'set-cookie': ['foo=bar', 'bar=baz'] }
```

### `outgoingMessage.hasHeader(name)`

<!-- YAML
added: v7.7.0
-->

- `name` {string}
- 返回：{boolean}

如果由 `name` 标识的头当前设置在传出头中，则返回 `true`。头名称匹配不区分大小写。

```js
const hasContentType = outgoingMessage.hasHeader('content-type');
```

### `outgoingMessage.headersSent`

<!-- YAML
added: v0.9.3
-->

- 类型：{boolean}

只读。如果头已发送，则为 `true`，否则为 `false`。

### `outgoingMessage.pipe()`

<!-- YAML
added: v9.0.0
-->

覆盖从旧版 `Stream` 类继承的 `stream.pipe()` 方法，该类是 `http.OutgoingMessage` 的父类。

调用此方法将抛出 `Error`，因为 `outgoingMessage` 是只写流。

### `outgoingMessage.removeHeader(name)`

<!-- YAML
added: v0.4.0
-->

- `name` {string} 头名称

移除排队等待隐式发送的头。

```js
outgoingMessage.removeHeader('Content-Encoding');
```

### `outgoingMessage.setHeader(name, value)`

<!-- YAML
added: v0.4.0
-->

- `name` {string} 头名称
- `value` {number | string | string\[]} 头值
- 返回：{this}

设置单个头值。如果头在要发送的头中已存在，其值将被替换。使用字符串数组发送具有相同名称的多个头。

### `outgoingMessage.setHeaders(headers)`

<!-- YAML
added:
  - v19.6.0
  - v18.15.0
-->

- `headers` {Headers|Map}
- 返回：{this}

为隐式头设置多个头值。`headers` 必须是 [`Headers`][] 或 `Map` 的实例，如果头在要发送的头中已存在，其值将被替换。

```js
const headers = new Headers({ foo: 'bar' });
outgoingMessage.setHeaders(headers);
```

或

```js
const headers = new Map([['foo', 'bar']]);
outgoingMessage.setHeaders(headers);
```

当头已使用 [`outgoingMessage.setHeaders()`][] 设置时，它们将与传递给 [`response.writeHead()`][] 的任何头合并，传递给 [`response.writeHead()`][] 的头优先。

```js
// 返回 content-type = text/plain
const server = http.createServer((req, res) => {
  const headers = new Headers({ 'Content-Type': 'text/html' });
  res.setHeaders(headers);
  res.writeHead(200, { 'Content-Type': 'text/plain' });
  res.end('ok');
});
```

### `outgoingMessage.setTimeout(msecs[, callback])`

<!-- YAML
added: v0.9.12
-->

- `msecs` {number}
- `callback` {Function} 超时发生时调用的可选函数。与绑定到 `timeout` 事件相同。
- 返回：{this}

一旦 socket 与消息关联并连接，将使用 `msecs` 作为第一个参数调用 [`socket.setTimeout()`][]。

### `outgoingMessage.socket`

<!-- YAML
added: v0.3.0
-->

- 类型：{stream.Duplex}

对底层 socket 的引用。通常，用户不希望访问此属性。

在调用 `outgoingMessage.end()` 后，此属性将为 null。

### `outgoingMessage.uncork()`

<!-- YAML
added:
  - v13.2.0
  - v12.16.0
-->

参见 [`writable.uncork()`][]

### `outgoingMessage.writableCorked`

<!-- YAML
added:
  - v13.2.0
  - v12.16.0
-->

- 类型：{number}

`outgoingMessage.cork()` 被调用的次数。

### `outgoingMessage.writableEnded`

<!-- YAML
added: v12.9.0
-->

- 类型：{boolean}

如果 `outgoingMessage.end()` 已被调用，则为 `true`。此属性不指示数据是否已刷新。为此目的，请使用 `message.writableFinished`。

### `outgoingMessage.writableFinished`

<!-- YAML
added: v12.7.0
-->

- 类型：{boolean}

如果所有数据都已刷新到底层系统，则为 `true`。

### `outgoingMessage.writableHighWaterMark`

<!-- YAML
added: v12.9.0
-->

- 类型：{number}

如果分配了底层 socket 的 `highWaterMark`。否则，当 [`writable.write()`][] 开始返回 false 时的默认缓冲区级别（`16384`）。

### `outgoingMessage.writableLength`

<!-- YAML
added: v12.9.0
-->

- 类型：{number}

缓冲的字节数。

### `outgoingMessage.writableObjectMode`

<!-- YAML
added: v12.9.0
-->

- 类型：{boolean}

始终为 `false`。

### `outgoingMessage.write(chunk[, encoding][, callback])`

<!-- YAML
added: v0.1.29
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/33155
    description: `chunk` 参数现在可以是 `Uint8Array`。
  - version: v0.11.6
    description: 添加了 `callback` 参数。
-->

- `chunk` {string|Buffer|Uint8Array}
- `encoding` {string} **默认值：** `utf8`
- `callback` {Function}
- 返回：{boolean}

发送一个主体块。此方法可以多次调用。

`encoding` 参数仅当 `chunk` 是字符串时相关。默认为 `'utf8'`。

`callback` 参数是可选的，将在数据块刷新时调用。

如果所有数据都成功刷新到内核缓冲区，则返回 `true`。如果所有或部分数据在用户内存中排队，则返回 `false`。当缓冲区再次空闲时将发出 `'drain'` 事件。

## `http.METHODS`

<!-- YAML
added: v0.11.8
-->

- 类型：{string\[]}

解析器支持的 HTTP 方法列表。

## `http.STATUS_CODES`

<!-- YAML
added: v0.1.22
-->

- 类型：{Object}

所有标准 HTTP 响应状态码及其简短描述的集合。例如，`http.STATUS_CODES[404] === 'Not Found'`。

## `http.createServer([options][, requestListener])`

<!-- YAML
added: v0.1.13
changes:
  - version: v24.9.0
    pr-url: https://github.com/nodejs/node/pull/59824
    description: 现在支持 `shouldUpgradeCallback` 选项。
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/47405
    description: 现在支持 `highWaterMark` 选项。
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41263
    description: 现在支持 `requestTimeout`、`headersTimeout`、`keepAliveTimeout` 和 `connectionsCheckingInterval` 选项。
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/42163
    description: `noDelay` 选项现在默认为 `true`。
  - version:
    - v17.7.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/41310
    description: 现在支持 `noDelay`、`keepAlive` 和 `keepAliveInitialDelay` 选项。
  - version:
     - v13.8.0
     - v12.15.0
     - v10.19.0
    pr-url: https://github.com/nodejs/node/pull/31448
    description: 现在支持 `insecureHTTPParser` 选项。
  - version: v13.3.0
    pr-url: https://github.com/nodejs/node/pull/30570
    description: 现在支持 `maxHeaderSize` 选项。
  - version:
    - v9.6.0
    - v8.12.0
    pr-url: https://github.com/nodejs/node/pull/15752
    description: 现在支持 `options` 参数。
-->

- `options` {Object}

  - `connectionsCheckingInterval`：设置以毫秒为单位的间隔值，以检查不完整请求中的请求和头超时。
    **默认值：** `30000`。
  - `headersTimeout`：设置从客户端接收完整 HTTP 头的超时值（毫秒）。
    有关更多信息，请参见 [`server.headersTimeout`][]。
    **默认值：** `60000`。
  - `highWaterMark` {number} 可选地覆盖所有 `socket` 的 `readableHighWaterMark` 和 `writableHighWaterMark`。这会影响 `IncomingMessage` 和 `ServerResponse` 的 `highWaterMark` 属性。
    **默认值：** 参见 [`stream.getDefaultHighWaterMark()`][]。
  - `insecureHTTPParser` {boolean} 如果设置为 `true`，它将使用具有宽松标志的 HTTP 解析器。应避免使用不安全的解析器。
    有关更多信息，请参见 [`--insecure-http-parser`][]。
    **默认值：** `false`。
  - `IncomingMessage` {http.IncomingMessage} 指定要使用的 `IncomingMessage` 类。对于扩展原始 `IncomingMessage` 很有用。
    **默认值：** `IncomingMessage`。
  - `joinDuplicateHeaders` {boolean} 如果设置为 `true`，此选项允许将请求中多个头的字段行值使用逗号（`, `）连接，而不是丢弃重复项。
    有关更多信息，请参见 [`message.headers`][]。
    **默认值：** `false`。
  - `keepAlive` {boolean} 如果设置为 `true`，它会在接收到新的传入连接后立即在 socket 上启用 keep-alive 功能，类似于在 \[`socket.setKeepAlive([enable][, initialDelay])`]\[`socket.setKeepAlive(enable, initialDelay)`] 中所做的操作。
    **默认值：** `false`。
  - `keepAliveInitialDelay` {number} 如果设置为正数，它设置在空闲 socket 上发送第一个 keepalive 探测之前的初始延迟。
    **默认值：** `0`。
  - `keepAliveTimeout`：服务器在完成写入最后一个响应后需要等待额外传入数据的不活动毫秒数，然后 socket 将被销毁。
    有关更多信息，请参见 [`server.keepAliveTimeout`][]。
    **默认值：** `5000`。
  - `maxHeaderSize` {number} 可选地覆盖此服务器接收的请求的 [`--max-http-header-size`][] 值，即请求头的最大长度（字节）。
    **默认值：** 16384 (16 KiB)。
  - `noDelay` {boolean} 如果设置为 `true`，它会在接收到新的传入连接后立即禁用 Nagle 算法。
    **默认值：** `true`。
  - `requestTimeout`：设置从客户端接收整个请求的超时值（毫秒）。
    有关更多信息，请参见 [`server.requestTimeout`][]。
    **默认值：** `300000`。
  - `requireHostHeader` {boolean} 如果设置为 `true`，它强制服务器对任何缺少 Host 头的 HTTP/1.1 请求消息响应 400 (Bad Request) 状态码（根据规范要求）。
    **默认值：** `true`。
  - `ServerResponse` {http.ServerResponse} 指定要使用的 `ServerResponse` 类。对于扩展原始 `ServerResponse` 很有用。**默认值：**
    `ServerResponse`。
  - `shouldUpgradeCallback(request)` {Function} 一个回调，接收传入请求并返回布尔值，以控制应接受哪些升级尝试。接受的升级将触发 `'upgrade'` 事件（或者如果没有注册监听器，它们的 socket 将被销毁），而拒绝的升级将像任何非升级请求一样触发 `'request'` 事件。此选项默认为
    `() => server.listenerCount('upgrade') > 0`。
  - `uniqueHeaders` {Array} 应仅发送一次的响应头列表。如果头的值是数组，则项将使用 `; ` 连接。
  - `rejectNonStandardBodyWrites` {boolean} 如果设置为 `true`，当写入没有主体的 HTTP 响应时会抛出错误。
    **默认值：** `false`。

- `requestListener` {Function}

- 返回：{http.Server}

返回 [`http.Server`][] 的新实例。

`requestListener` 是一个自动添加到 [`'request'`][] 事件的函数。

```mjs
import http from 'node:http';

// 创建本地服务器以接收数据
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(
    JSON.stringify({
      data: 'Hello World!',
    })
  );
});

server.listen(8000);
```

```cjs
const http = require('node:http');

// 创建本地服务器以接收数据
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(
    JSON.stringify({
      data: 'Hello World!',
    })
  );
});

server.listen(8000);
```

```mjs
import http from 'node:http';

// 创建本地服务器以接收数据
const server = http.createServer();

// 监听请求事件
server.on('request', (request, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(
    JSON.stringify({
      data: 'Hello World!',
    })
  );
});

server.listen(8000);
```

```cjs
const http = require('node:http');

// 创建本地服务器以接收数据
const server = http.createServer();

// 监听请求事件
server.on('request', (request, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(
    JSON.stringify({
      data: 'Hello World!',
    })
  );
});

server.listen(8000);
```

## `http.get(options[, callback])`

## `http.get(url[, options][, callback])`

<!-- YAML
added: v0.3.6
changes:
  - version: v10.9.0
    pr-url: https://github.com/nodejs/node/pull/21616
    description: `url` 参数现在可以与单独的 `options` 对象一起传递。
  - version: v7.5.0
    pr-url: https://github.com/nodejs/node/pull/10638
    description: `options` 参数可以是 WHATWG `URL` 对象。
-->

- `url` {string | URL}
- `options` {Object} 接受与 [`http.request()`][] 相同的 `options`，方法默认设置为 GET。
- `callback` {Function}
- 返回：{http.ClientRequest}

由于大多数请求是没有主体的 GET 请求，Node.js 提供了此便捷方法。此方法与 [`http.request()`][] 的唯一区别在于它默认将方法设置为 GET 并自动调用 `req.end()`。回调必须注意消耗响应数据，原因在 [`http.ClientRequest`][] 部分说明。

`callback` 使用单个参数调用，该参数是 [`http.IncomingMessage`][] 的实例。

JSON 获取示例：

```js
http
  .get('http://localhost:8000/', (res) => {
    const { statusCode } = res;
    const contentType = res.headers['content-type'];

    let error;
    // 任何 2xx 状态码表示成功的响应，但这里我们只检查 200。
    if (statusCode !== 200) {
      error = new Error('Request Failed.\n' + `Status Code: ${statusCode}`);
    } else if (!/^application\/json/.test(contentType)) {
      error = new Error(
        'Invalid content-type.\n' +
          `Expected application/json but received ${contentType}`
      );
    }
    if (error) {
      console.error(error.message);
      // 消耗响应数据以释放内存
      res.resume();
      return;
    }

    res.setEncoding('utf8');
    let rawData = '';
    res.on('data', (chunk) => {
      rawData += chunk;
    });
    res.on('end', () => {
      try {
        const parsedData = JSON.parse(rawData);
        console.log(parsedData);
      } catch (e) {
        console.error(e.message);
      }
    });
  })
  .on('error', (e) => {
    console.error(`Got error: ${e.message}`);
  });

// 创建本地服务器以接收数据
const server = http.createServer((req, res) => {
  res.writeHead(200, { 'Content-Type': 'application/json' });
  res.end(
    JSON.stringify({
      data: 'Hello World!',
    })
  );
});

server.listen(8000);
```

## `http.globalAgent`

<!-- YAML
added: v0.5.9
changes:
  - version:
      - v19.0.0
    pr-url: https://github.com/nodejs/node/pull/43522
    description: 代理现在默认使用 HTTP Keep-Alive 和 5 秒超时。
-->

- 类型：{http.Agent}

`Agent` 的全局实例，用作所有 HTTP 客户端请求的默认值。与默认 `Agent` 配置的不同之处在于启用了 `keepAlive` 并具有 5 秒的 `timeout`。

## `http.maxHeaderSize`

<!-- YAML
added:
 - v11.6.0
 - v10.15.0
-->

- 类型：{number}

只读属性，指定 HTTP 头的最大允许大小（字节）。默认为 16 KiB。可使用 [`--max-http-header-size`][] CLI 选项配置。

可以通过传递 `maxHeaderSize` 选项为服务器和客户端请求覆盖此值。

## `http.request(options[, callback])`

## `http.request(url[, options][, callback])`

<!-- YAML
added: v0.3.6
changes:
  - version:
      - v16.7.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/39310
    description: 当使用 `URL` 对象时，解析的用户名和密码现在将正确进行 URI 解码。
  - version:
      - v15.3.0
      - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/36048
    description: 可以使用 AbortSignal 中止请求。
  - version:
     - v13.8.0
     - v12.15.0
     - v10.19.0
    pr-url: https://github.com/nodejs/node/pull/31448
    description: 现在支持 `insecureHTTPParser` 选项。
  - version: v13.3.0
    pr-url: https://github.com/nodejs/node/pull/30570
    description: 现在支持 `maxHeaderSize` 选项。
  - version: v10.9.0
    pr-url: https://github.com/nodejs/node/pull/21616
    description: `url` 参数现在可以与单独的 `options` 对象一起传递。
  - version: v7.5.0
    pr-url: https://github.com/nodejs/node/pull/10638
    description: `options` 参数可以是 WHATWG `URL` 对象。
-->

- `url` {string | URL}
- `options` {Object}
  - `agent` {http.Agent | boolean} 控制 [`Agent`][] 行为。可能的值：
    - `undefined` (默认)：对此主机和端口使用 [`http.globalAgent`][]。
    - `Agent` 对象：显式使用传入的 `Agent`。
    - `false`：导致使用具有默认值的新 `Agent`。
  - `auth` {string} 基本身份验证（`'user:password'`）以计算 Authorization 头。
  - `createConnection` {Function} 当未使用 `agent` 选项时，产生用于请求的 socket/流的函数。这可用于避免仅为了覆盖默认 `createConnection` 函数而创建自定义 `Agent` 类。有关更多详细信息，请参见 [`agent.createConnection()`][]。任何 [`Duplex`][] 流都是有效的返回值。
  - `defaultPort` {number} 协议的默认端口。**默认值：**
    如果使用 `Agent`，则为 `agent.defaultPort`，否则为 `undefined`。
  - `family` {number} 解析 `host` 或 `hostname` 时使用的 IP 地址族。有效值为 `4` 或 `6`。未指定时，将同时使用 IP v4 和 v6。
  - `headers` {Object|Array} 包含请求头的对象或字符串数组。数组的格式与 [`message.rawHeaders`][] 相同。
  - `hints` {number} 可选的 [`dns.lookup()` hints][]。
  - `host` {string} 发出请求的服务器的域名或 IP 地址。**默认值：** `'localhost'`。
  - `hostname` {string} `host` 的别名。为了支持 [`url.parse()`][]，如果同时指定了 `host` 和 `hostname`，则将使用 `hostname`。
  - `insecureHTTPParser` {boolean} 如果设置为 `true`，它将使用具有宽松标志的 HTTP 解析器。应避免使用不安全的解析器。
    有关更多信息，请参见 [`--insecure-http-parser`][]。
    **默认值：** `false`
  - `joinDuplicateHeaders` {boolean} 它将请求中多个头的字段行值使用 `, ` 连接，而不是丢弃重复项。有关更多信息，请参见 [`message.headers`][]。
    **默认值：** `false`。
  - `localAddress` {string} 用于网络连接的本地接口。
  - `localPort` {number} 连接来源的本地端口。
  - `lookup` {Function} 自定义查找函数。**默认值：** [`dns.lookup()`][]。
  - `maxHeaderSize` {number} 可选地覆盖从服务器接收的响应的 [`--max-http-header-size`][] 值（响应头的最大长度，字节）。
    **默认值：** 16384 (16 KiB)。
  - `method` {string} 指定 HTTP 请求方法的字符串。**默认值：**
    `'GET'`。
  - `path` {string} 请求路径。应包括查询字符串（如果有）。
    例如 `'/index.html?page=12'`。当请求路径包含非法字符时抛出异常。当前仅拒绝空格，但未来可能会更改。**默认值：** `'/'`。
  - `port` {number} 远程服务器的端口。**默认值：** 如果设置了 `defaultPort`，则为 `defaultPort`，否则为 `80`。
  - `protocol` {string} 要使用的协议。**默认值：** `'http:'`。
  - `setDefaultHeaders` {boolean}：指定是否自动添加默认头，如 `Connection`、`Content-Length`、`Transfer-Encoding` 和 `Host`。如果设置为 `false`，则必须手动添加所有必要的头。默认为 `true`。
  - `setHost` {boolean}：指定是否自动添加 `Host` 头。如果提供，则覆盖 `setDefaultHeaders`。默认为 `true`。
  - `signal` {AbortSignal}：可用于中止正在进行的请求的 AbortSignal。
  - `socketPath` {string} Unix 域 socket。如果指定了 `host` 或 `port` 之一，则不能使用，因为它们指定了 TCP Socket。
  - `timeout` {number}：指定 socket 超时（毫秒）的数字。
    这将在 socket 连接之前设置超时。
  - `uniqueHeaders` {Array} 应仅发送一次的请求头列表。如果头的值是数组，则项将使用 `; ` 连接。
- `callback` {Function}
- 返回：{http.ClientRequest}

[`socket.connect()`][] 中的 `options` 也受支持。

Node.js 为每个服务器维护多个连接以发出 HTTP 请求。此函数允许透明地发出请求。

`url` 可以是字符串或 [`URL`][] 对象。如果 `url` 是字符串，则使用 [`new URL()`][] 自动解析。如果是 [`URL`][] 对象，它将自动转换为普通 `options` 对象。

如果同时指定了 `url` 和 `options`，则对象会合并，`options` 属性优先。

可选的 `callback` 参数将作为一次性监听器添加到 [`'response'`][] 事件。

`http.request()` 返回 [`http.ClientRequest`][] 类的实例。`ClientRequest` 实例是可写流。如果需要使用 POST 请求上传文件，则写入 `ClientRequest` 对象。

```mjs
import http from 'node:http';
import { Buffer } from 'node:buffer';

const postData = JSON.stringify({
  msg: 'Hello World!',
});

const options = {
  hostname: 'www.google.com',
  port: 80,
  path: '/upload',
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Content-Length': Buffer.byteLength(postData),
  },
};

const req = http.request(options, (res) => {
  console.log(`STATUS: ${res.statusCode}`);
  console.log(`HEADERS: ${JSON.stringify(res.headers)}`);
  res.setEncoding('utf8');
  res.on('data', (chunk) => {
    console.log(`BODY: ${chunk}`);
  });
  res.on('end', () => {
    console.log('No more data in response.');
  });
});

req.on('error', (e) => {
  console.error(`problem with request: ${e.message}`);
});

// 将数据写入请求体
req.write(postData);
req.end();
```

```cjs
const http = require('node:http');

const postData = JSON.stringify({
  msg: 'Hello World!',
});

const options = {
  hostname: 'www.google.com',
  port: 80,
  path: '/upload',
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'Content-Length': Buffer.byteLength(postData),
  },
};

const req = http.request(options, (res) => {
  console.log(`STATUS: ${res.statusCode}`);
  console.log(`HEADERS: ${JSON.stringify(res.headers)}`);
  res.setEncoding('utf8');
  res.on('data', (chunk) => {
    console.log(`BODY: ${chunk}`);
  });
  res.on('end', () => {
    console.log('No more data in response.');
  });
});

req.on('error', (e) => {
  console.error(`problem with request: ${e.message}`);
});

// 将数据写入请求体
req.write(postData);
req.end();
```

在示例中调用了 `req.end()`。使用 `http.request()` 时必须始终调用 `req.end()` 以表示请求结束 - 即使没有数据写入请求体。

如果在请求期间遇到任何错误（无论是 DNS 解析、TCP 级别错误还是实际 HTTP 解析错误），都会在返回的请求对象上发出 `'error'` 事件。与所有 `'error'` 事件一样，如果未注册监听器，错误将被抛出。

有几个特殊的头需要注意。

- 发送 'Connection: keep-alive' 将通知 Node.js 与服务器的连接应持久化直到下一个请求。

- 发送 'Content-Length' 头将禁用默认的分块编码。

- 发送 'Expect' 头将立即发送请求头。
  通常，当发送 'Expect: 100-continue' 时，应设置超时和 `'continue'` 事件的监听器。有关更多信息，请参见 RFC 2616 第 8.2.3 节。

- 发送 Authorization 头将覆盖使用 `auth` 选项计算基本身份验证。

使用 [`URL`][] 作为 `options` 的示例：

```js
const options = new URL('http://abc:xyz@example.com');

const req = http.request(options, (res) => {
  // ...
});
```

在成功的请求中，将按以下顺序发出以下事件：

- `'socket'`
- `'response'`
  - `'data'` 任意次数，在 `res` 对象上
    （如果响应体为空，例如在大多数重定向中，则根本不会发出 `'data'`）
  - `'end'` 在 `res` 对象上
- `'close'`

在连接错误的情况下，将发出以下事件：

- `'socket'`
- `'error'`
- `'close'`

在响应接收之前过早关闭连接的情况下，将按以下顺序发出以下事件：

- `'socket'`
- `'error'` 带有消息 `'Error: socket hang up'` 和代码 `'ECONNRESET'` 的错误
- `'close'`

在响应接收后过早关闭连接的情况下，将按以下顺序发出以下事件：

- `'socket'`
- `'response'`
  - `'data'` 任意次数，在 `res` 对象上
- (连接在此处关闭)
- `'aborted'` 在 `res` 对象上
- `'close'`
- `'error'` 在 `res` 对象上，带有消息 `'Error: aborted'` 和代码 `'ECONNRESET'` 的错误
- `'close'` 在 `res` 对象上

如果在分配 socket 之前调用 `req.destroy()`，将按以下顺序发出以下事件：

- (`req.destroy()` 在此处调用)
- `'error'` 带有消息 `'Error: socket hang up'` 和代码 `'ECONNRESET'` 的错误，或 `req.destroy()` 调用的错误
- `'close'`

如果在连接成功之前调用 `req.destroy()`，将按以下顺序发出以下事件：

- `'socket'`
- (`req.destroy()` 在此处调用)
- `'error'` 带有消息 `'Error: socket hang up'` 和代码 `'ECONNRESET'` 的错误，或 `req.destroy()` 调用的错误
- `'close'`

如果在响应接收后调用 `req.destroy()`，将按以下顺序发出以下事件：

- `'socket'`
- `'response'`
  - `'data'` 任意次数，在 `res` 对象上
- (`req.destroy()` 在此处调用)
- `'aborted'` 在 `res` 对象上
- `'close'`
- `'error'` 在 `res` 对象上，带有消息 `'Error: aborted'` 和代码 `'ECONNRESET'` 的错误，或 `req.destroy()` 调用的错误
- `'close'` 在 `res` 对象上

如果在分配 socket 之前调用 `req.abort()`，将按以下顺序发出以下事件：

- (`req.abort()` 在此处调用)
- `'abort'`
- `'close'`

如果在连接成功之前调用 `req.abort()`，将按以下顺序发出以下事件：

- `'socket'`
- (`req.abort()` 在此处调用)
- `'abort'`
- `'error'` 带有消息 `'Error: socket hang up'` 和代码 `'ECONNRESET'` 的错误
- `'close'`

如果在响应接收后调用 `req.abort()`，将按以下顺序发出以下事件：

- `'socket'`
- `'response'`
  - `'data'` 任意次数，在 `res` 对象上
- (`req.abort()` 在此处调用)
- `'abort'`
- `'aborted'` 在 `res` 对象上
- `'error'` 在 `res` 对象上，带有消息 `'Error: aborted'` 和代码 `'ECONNRESET'` 的错误。
- `'close'`
- `'close'` 在 `res` 对象上

设置 `timeout` 选项或使用 `setTimeout()` 函数不会中止请求或执行任何操作，除了添加 `'timeout'` 事件。

传递 `AbortSignal` 然后在相应的 `AbortController` 上调用 `abort()` 的行为与在请求上调用 `.destroy()` 的方式相同。具体来说，将发出 `'error'` 事件，其中包含消息 `'AbortError: The operation was aborted'`、代码 `'ABORT_ERR'` 和 `cause`（如果提供）。

## `http.validateHeaderName(name[, label])`

<!-- YAML
added: v14.3.0
changes:
  - version:
    - v19.5.0
    - v18.14.0
    pr-url: https://github.com/nodejs/node/pull/46143
    description: 添加了 `label` 参数。
-->

- `name` {string}
- `label` {string} 错误消息的标签。**默认值：** `'Header name'`。

执行在调用 `res.setHeader(name, value)` 时完成的低级验证。

传递非法值作为 `name` 将导致抛出 [`TypeError`][]，标识为 `code: 'ERR_INVALID_HTTP_TOKEN'`。

在将头传递给 HTTP 请求或响应之前，不需要使用此方法。HTTP 模块将自动验证此类头。

示例：

```mjs
import { validateHeaderName } from 'node:http';

try {
  validateHeaderName('');
} catch (err) {
  console.error(err instanceof TypeError); // --> true
  console.error(err.code); // --> 'ERR_INVALID_HTTP_TOKEN'
  console.error(err.message); // --> 'Header name must be a valid HTTP token [""]'
}
```

```cjs
const { validateHeaderName } = require('node:http');

try {
  validateHeaderName('');
} catch (err) {
  console.error(err instanceof TypeError); // --> true
  console.error(err.code); // --> 'ERR_INVALID_HTTP_TOKEN'
  console.error(err.message); // --> 'Header name must be a valid HTTP token [""]'
}
```

## `http.validateHeaderValue(name, value)`

<!-- YAML
added: v14.3.0
-->

- `name` {string}
- `value` {any}

执行在调用 `res.setHeader(name, value)` 时完成的低级验证。

传递非法值作为 `value` 将导致抛出 [`TypeError`][]。

- 未定义值错误标识为 `code: 'ERR_HTTP_INVALID_HEADER_VALUE'`。
- 无效值字符错误标识为 `code: 'ERR_INVALID_CHAR'`。

在将头传递给 HTTP 请求或响应之前，不需要使用此方法。HTTP 模块将自动验证此类头。

示例：

```mjs
import { validateHeaderValue } from 'node:http';

try {
  validateHeaderValue('x-my-header', undefined);
} catch (err) {
  console.error(err instanceof TypeError); // --> true
  console.error(err.code === 'ERR_HTTP_INVALID_HEADER_VALUE'); // --> true
  console.error(err.message); // --> 'Invalid value "undefined" for header "x-my-header"'
}

try {
  validateHeaderValue('x-my-header', 'oʊmɪɡə');
} catch (err) {
  console.error(err instanceof TypeError); // --> true
  console.error(err.code === 'ERR_INVALID_CHAR'); // --> true
  console.error(err.message); // --> 'Invalid character in header content ["x-my-header"]'
}
```

```cjs
const { validateHeaderValue } = require('node:http');

try {
  validateHeaderValue('x-my-header', undefined);
} catch (err) {
  console.error(err instanceof TypeError); // --> true
  console.error(err.code === 'ERR_HTTP_INVALID_HEADER_VALUE'); // --> true
  console.error(err.message); // --> 'Invalid value "undefined" for header "x-my-header"'
}

try {
  validateHeaderValue('x-my-header', 'oʊmɪɡə');
} catch (err) {
  console.error(err instanceof TypeError); // --> true
  console.error(err.code === 'ERR_INVALID_CHAR'); // --> true
  console.error(err.message); // --> 'Invalid character in header content ["x-my-header"]'
}
```

## `http.setMaxIdleHTTPParsers(max)`

<!-- YAML
added:
  - v18.8.0
  - v16.18.0
-->

- `max` {number} **默认值：** `1000`。

设置空闲 HTTP 解析器的最大数量。

## 类：`WebSocket`

<!-- YAML
added:
  - v22.5.0
-->

{WebSocket} 的浏览器兼容实现。

## 内置代理支持

<!-- YAML
added: v24.5.0
-->

> Stability: 1.1 - 积极开发

当 Node.js 创建全局代理时，如果 `NODE_USE_ENV_PROXY` 环境变量设置为 `1` 或启用了 `--use-env-proxy`，则全局代理将使用 `proxyEnv: process.env` 构建，从而基于环境变量启用代理支持。自定义代理也可以通过在建理代理时传递 `proxyEnv` 选项来创建代理支持。值可以是 `process.env`，如果它们只想从环境变量继承配置，或者是一个具有特定设置覆盖环境的对象。

检查 `proxyEnv` 的以下属性以配置代理支持。

- `HTTP_PROXY` 或 `http_proxy`：HTTP 请求的代理服务器 URL。如果两者都设置，`http_proxy` 优先。
- `HTTPS_PROXY` 或 `https_proxy`：HTTPS 请求的代理服务器 URL。如果两者都设置，`https_proxy` 优先。
- `NO_PROXY` 或 `no_proxy`：逗号分隔的应绕过代理的主机列表。如果两者都设置，`no_proxy` 优先。

如果请求发送到 Unix 域 socket，则代理设置将被忽略。

### 代理 URL 格式

代理 URL 可以使用 HTTP 或 HTTPS 协议：

- HTTP 代理：`http://proxy.example.com:8080`
- HTTPS 代理：`https://proxy.example.com:8080`
- 带身份验证的代理：`http://username:password@proxy.example.com:8080`

### `NO_PROXY` 格式

`NO_PROXY` 环境变量支持几种格式：

- `*` - 为所有主机绕过代理
- `example.com` - 精确主机名匹配
- `.example.com` - 域名后缀匹配（匹配 `sub.example.com`）
- `*.example.com` - 通配符域名匹配
- `192.168.1.100` - 精确 IP 地址匹配
- `192.168.1.1-192.168.1.100` - IP 地址范围
- `example.com:8080` - 带有特定端口的主机名

多个条目应使用逗号分隔。

### 示例

要启动具有代理支持的 Node.js 进程，以便通过默认全局代理发送的所有请求都启用代理支持，可以使用 `NODE_USE_ENV_PROXY` 环境变量：

```console
NODE_USE_ENV_PROXY=1 HTTP_PROXY=http://proxy.example.com:8080 NO_PROXY=localhost,127.0.0.1 node client.js
```

或 `--use-env-proxy` 标志。

```console
HTTP_PROXY=http://proxy.example.com:8080 NO_PROXY=localhost,127.0.0.1 node --use-env-proxy client.js
```

要创建具有内置代理支持的自定义代理：

```cjs
const http = require('node:http');

// 创建具有自定义代理支持的自定义代理。
const agent = new http.Agent({
  proxyEnv: { HTTP_PROXY: 'http://proxy.example.com:8080' },
});

http.request(
  {
    hostname: 'www.example.com',
    port: 80,
    path: '/',
    agent,
  },
  (res) => {
    // 此请求将通过 HTTP 协议通过 proxy.example.com:8080 代理。
    console.log(`STATUS: ${res.statusCode}`);
  }
);
```

或者，以下也有效：

```cjs
const http = require('node:http');
// 使用小写选项名称。
const agent1 = new http.Agent({
  proxyEnv: { http_proxy: 'http://proxy.example.com:8080' },
});
// 使用从环境变量继承的值，如果进程以 HTTP_PROXY=http://proxy.example.com:8080 启动，则将使用 process.env.HTTP_PROXY 中指定的代理服务器。
const agent2 = new http.Agent({ proxyEnv: process.env });
```

[内置代理支持]: #内置代理支持
[RFC 8187]: https://www.rfc-editor.org/rfc/rfc8187.txt
[`'ERR_HTTP_CONTENT_LENGTH_MISMATCH'`]: errors.md#err_http_content_length_mismatch
[`'checkContinue'`]: #event-checkcontinue
[`'finish'`]: #event-finish
[`'request'`]: #event-request
[`'response'`]: #event-response
[`'upgrade'`]: #event-upgrade
[`--insecure-http-parser`]: cli.md#--insecure-http-parser
[`--max-http-header-size`]: cli.md#--max-http-header-sizesize
[`Agent`]: #class-httpagent
[`Buffer.byteLength()`]: buffer.md#static-method-bufferbytelengthstring-encoding
[`Duplex`]: stream.md#class-streamduplex
[`HPE_HEADER_OVERFLOW`]: errors.md#hpe_header_overflow
[`Headers`]: globals.md#class-headers
[`TypeError`]: errors.md#class-typeerror
[`URL`]: url.md#the-whatwg-url-api
[`agent.createConnection()`]: #agentcreateconnectionoptions-callback
[`agent.getName()`]: #agentgetnameoptions
[`destroy()`]: #agentdestroy
[`dns.lookup()`]: dns.md#dnslookuphostname-options-callback
[`dns.lookup()` hints]: dns.md#supported-getaddrinfo-flags
[`getHeader(name)`]: #requestgetheadername
[`http.Agent`]: #class-httpagent
[`http.ClientRequest`]: #class-httpclientrequest
[`http.IncomingMessage`]: #class-httpincomingmessage
[`http.ServerResponse`]: #class-httpserverresponse
[`http.Server`]: #class-httpserver
[`http.createServer()`]: #httpcreateserveroptions-requestlistener
[`http.get()`]: #httpgetoptions-callback
[`http.globalAgent`]: #httpglobalagent
[`http.request()`]: #httprequestoptions-callback
[`message.headers`]: #messageheaders
[`message.rawHeaders`]: #messagerawheaders
[`message.socket`]: #messagesocket
[`message.trailers`]: #messagetrailers
[`net.Server.close()`]: net.md#serverclosecallback
[`net.Server`]: net.md#class-netserver
[`net.Socket`]: net.md#class-netsocket
[`net.createConnection()`]: net.md#netcreateconnectionoptions-connectlistener
[`new URL()`]: url.md#new-urlinput-base
[`outgoingMessage.setHeader(name, value)`]: #outgoingmessagesetheadername-value
[`outgoingMessage.setHeaders()`]: #outgoingmessagesetheadersheaders
[`outgoingMessage.socket`]: #outgoingmessagesocket
[`removeHeader(name)`]: #requestremoveheadername
[`request.destroy()`]: #requestdestroyerror
[`request.destroyed`]: #requestdestroyed
[`request.end()`]: #requestenddata-encoding-callback
[`request.flushHeaders()`]: #requestflushheaders
[`request.getHeader()`]: #requestgetheadername
[`request.setHeader()`]: #requestsetheadername-value
[`request.setTimeout()`]: #requestsettimeouttimeout-callback
[`request.socket.getPeerCertificate()`]: tls.md#tlssocketgetpeercertificatedetailed
[`request.socket`]: #requestsocket
[`request.writableEnded`]: #requestwritableended
[`request.writableFinished`]: #requestwritablefinished
[`request.write(data, encoding)`]: #requestwritechunk-encoding-callback
[`response.end()`]: #responseenddata-encoding-callback
[`response.getHeader()`]: #responsegetheadername
[`response.setHeader()`]: #responsesetheadername-value
[`response.socket`]: #responsesocket
[`response.strictContentLength`]: #responsestrictcontentlength
[`response.writableEnded`]: #responsewritableended
[`response.writableFinished`]: #responsewritablefinished
[`response.write()`]: #responsewritechunk-encoding-callback
[`response.write(data, encoding)`]: #responsewritechunk-encoding-callback
[`response.writeContinue()`]: #responsewritecontinue
[`response.writeHead()`]: #responsewriteheadstatuscode-statusmessage-headers
[`server.close()`]: #serverclosecallback
[`server.headersTimeout`]: #serverheaderstimeout
[`server.keepAliveTimeoutBuffer`]: #serverkeepalivetimeoutbuffer
[`server.keepAliveTimeout`]: #serverkeepalivetimeout
[`server.listen()`]: net.md#serverlisten
[`server.requestTimeout`]: #serverrequesttimeout
[`server.timeout`]: #servertimeout
[`setHeader(name, value)`]: #requestsetheadername-value
[`socket.connect()`]: net.md#socketconnectoptions-connectlistener
[`socket.setKeepAlive()`]: net.md#socketsetkeepaliveenable-initialdelay
[`socket.setNoDelay()`]: net.md#socketsetnodelaynodelay
[`socket.setTimeout()`]: net.md#socketsettimeouttimeout-callback
[`socket.unref()`]: net.md#socketunref
[`stream.getDefaultHighWaterMark()`]: stream.md#streamgetdefaulthighwatermarkobjectmode
[`url.parse()`]: url.md#urlparseurlstring-parsequerystring-slashesdenotehost
[`writable.cork()`]: stream.md#writablecork
[`writable.destroy()`]: stream.md#writabledestroyerror
[`writable.destroyed`]: stream.md#writabledestroyed
[`writable.uncork()`]: stream.md#writableuncork
[`writable.write()`]: stream.md#writablewritechunk-encoding-callback
[initial delay]: net.md#socketsetkeepaliveenable-initialdelay
