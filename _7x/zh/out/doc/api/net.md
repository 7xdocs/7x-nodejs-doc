# Net

<!--introduced_in=v0.10.0-->

<!--lint disable maximum-line-length-->

> Stability: 2 - Stable

<!-- source_link=lib/net.js -->

`node:net` 模块提供了异步网络 API，用于创建基于流的 TCP 或 [IPC][] 服务器（[`net.createServer()`][]）和客户端（[`net.createConnection()`][]）。

可以通过以下方式访问：

```mjs
import net from 'node:net';
```

```cjs
const net = require('node:net');
```

## IPC 支持

<!-- YAML
changes:
  - version: v20.8.0
    pr-url: https://github.com/nodejs/node/pull/49667
    description: Support binding to abstract Unix domain socket path like `\0abstract`.
                 We can bind '\0' for Node.js `< v20.4.0`.
-->

`node:net` 模块在 Windows 上支持使用命名管道的 IPC，在其他操作系统上支持 Unix 域套接字。

### 识别 IPC 连接的路径

[`net.connect()`][]、[`net.createConnection()`][]、[`server.listen()`][] 和 [`socket.connect()`][] 接受一个 `path` 参数来标识 IPC 端点。

在 Unix 上，本地域也称为 Unix 域。路径是文件系统路径名。当路径名的长度大于 `sizeof(sockaddr_un.sun_path)` 的长度时，将抛出错误。典型值在 Linux 上为 107 字节，在 macOS 上为 103 字节。如果 Node.js API 抽象创建了 Unix 域套接字，它也会取消链接该 Unix 域套接字。例如，[`net.createServer()`][] 可能创建一个 Unix 域套接字，而 [`server.close()`][] 将取消链接它。但如果用户在这些抽象之外创建了 Unix 域套接字，则需要用户手动删除它。同样的情况也适用于 Node.js API 创建了 Unix 域套接字但程序随后崩溃的情况。简而言之，Unix 域套接字在文件系统中可见，并且会持续存在直到被取消链接。在 Linux 上，你可以通过在路径开头添加 `\0` 来使用 Unix 抽象套接字，例如 `\0abstract`。Unix 抽象套接字的路径在文件系统中不可见，并且当所有对套接字的打开引用关闭时，它会自动消失。

在 Windows 上，本地域是使用命名管道实现的。路径 _必须_ 引用 `\\?\pipe\` 或 `\\.\pipe\` 中的条目。允许任何字符，但后者可能会对管道名称进行一些处理，例如解析 `..` 序列。尽管看起来可能如此，但管道命名空间是平坦的。管道 _不会持久化_。当对它们的最后一个引用关闭时，它们将被移除。与 Unix 域套接字不同，Windows 会在拥有进程退出时关闭并移除管道。

JavaScript 字符串转义要求路径使用额外的反斜杠转义来指定，例如：

```js
net.createServer().listen(
  path.join('\\\\?\\pipe', process.cwd(), 'myctl'));
```

## 类：`net.BlockList`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

`BlockList` 对象可以与一些网络 API 一起使用，以指定禁用特定 IP 地址、IP 范围或 IP 子网的入站或出站访问的规则。

### `blockList.addAddress(address[, type])`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

* `address` {string|net.SocketAddress} 一个 IPv4 或 IPv6 地址。
* `type` {string} 可以是 `'ipv4'` 或 `'ipv6'`。**默认值：** `'ipv4'`。

添加一条规则来阻止给定的 IP 地址。

### `blockList.addRange(start, end[, type])`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

* `start` {string|net.SocketAddress} 范围内的起始 IPv4 或 IPv6 地址。
* `end` {string|net.SocketAddress} 范围内的结束 IPv4 或 IPv6 地址。
* `type` {string} 可以是 `'ipv4'` 或 `'ipv6'`。**默认值：** `'ipv4'`。

添加一条规则来阻止从 `start`（包含）到 `end`（包含）的 IP 地址范围。

### `blockList.addSubnet(net, prefix[, type])`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

* `net` {string|net.SocketAddress} 网络 IPv4 或 IPv6 地址。
* `prefix` {number} CIDR 前缀位数。对于 IPv4，必须是 `0` 到 `32` 之间的值。对于 IPv6，必须是 `0` 到 `128` 之间的值。
* `type` {string} 可以是 `'ipv4'` 或 `'ipv6'`。**默认值：** `'ipv4'`。

添加一条规则来阻止指定为子网掩码的 IP 地址范围。

### `blockList.check(address[, type])`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

* `address` {string|net.SocketAddress} 要检查的 IP 地址
* `type` {string} 可以是 `'ipv4'` 或 `'ipv6'`。**默认值：** `'ipv4'`。
* 返回：{boolean}

如果给定的 IP 地址匹配添加到 `BlockList` 的任何规则，则返回 `true`。

```js
const blockList = new net.BlockList();
blockList.addAddress('123.123.123.123');
blockList.addRange('10.0.0.1', '10.0.0.10');
blockList.addSubnet('8592:757c:efae:4e45::', 64, 'ipv6');

console.log(blockList.check('123.123.123.123'));  // 打印：true
console.log(blockList.check('10.0.0.3'));  // 打印：true
console.log(blockList.check('222.111.111.222'));  // 打印：false

// IPv4 地址的 IPv6 表示法也有效：
console.log(blockList.check('::ffff:7b7b:7b7b', 'ipv6')); // 打印：true
console.log(blockList.check('::ffff:123.123.123.123', 'ipv6')); // 打印：true
```

### `blockList.rules`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

* 类型：{string\[]}

添加到阻止列表中的规则列表。

### `BlockList.isBlockList(value)`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
-->

* `value` {any} 任何 JS 值
* 如果 `value` 是 `net.BlockList`，则返回 `true`。

### `blockList.fromJSON(value)`

> Stability: 1 - Experimental

 <!-- YAML
added: v24.5.0
-->

```js
const blockList = new net.BlockList();
const data = [
  'Subnet: IPv4 192.168.1.0/24',
  'Address: IPv4 10.0.0.5',
  'Range: IPv4 192.168.2.1-192.168.2.10',
  'Range: IPv4 10.0.0.1-10.0.0.10',
];
blockList.fromJSON(data);
blockList.fromJSON(JSON.stringify(data));
```

* `value` Blocklist.rules

### `blockList.toJSON()`

> Stability: 1 - Experimental

 <!-- YAML
added: v24.5.0
-->

* 返回 Blocklist.rules

## 类：`net.SocketAddress`

<!-- YAML
added:
  - v15.14.0
  - v14.18.0
-->

### `new net.SocketAddress([options])`

<!-- YAML
added:
  - v15.14.0
  - v14.18.0
-->

* `options` {Object}
  * `address` {string} 网络地址，可以是 IPv4 或 IPv6 字符串。
    **默认值**：如果 `family` 是 `'ipv4'`，则为 `'127.0.0.1'`；如果 `family` 是 `'ipv6'`，则为 `'::'`。
  * `family` {string} 可以是 `'ipv4'` 或 `'ipv6'` 之一。
    **默认值**：`'ipv4'`。
  * `flowlabel` {number} 仅当 `family` 为 `'ipv6'` 时使用的 IPv6 流标签。
  * `port` {number} IP 端口。

### `socketaddress.address`

<!-- YAML
added:
  - v15.14.0
  - v14.18.0
-->

* 类型：{string}

### `socketaddress.family`

<!-- YAML
added:
  - v15.14.0
  - v14.18.0
-->

* 类型：{string} 可以是 `'ipv4'` 或 `'ipv6'`。

### `socketaddress.flowlabel`

<!-- YAML
added:
  - v15.14.0
  - v14.18.0
-->

* 类型：{number}

### `socketaddress.port`

<!-- YAML
added:
  - v15.14.0
  - v14.18.0
-->

* 类型：{number}

### `SocketAddress.parse(input)`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
-->

* `input` {string} 包含 IP 地址和可选端口的输入字符串，例如 `123.1.2.3:1234` 或 `[1::1]:1234`。
* 返回：{net.SocketAddress} 如果解析成功，返回一个 `SocketAddress`。否则返回 `undefined`。

## 类：`net.Server`

<!-- YAML
added: v0.1.90
-->

* 扩展：{EventEmitter}

此类用于创建 TCP 或 [IPC][] 服务器。

### `new net.Server([options][, connectionListener])`

* `options` {Object} 参见 [`net.createServer([options][, connectionListener])`][`net.createServer()`]。
* `connectionListener` {Function} 自动设置为 [`'connection'`][] 事件的监听器。
* 返回：{net.Server}

`net.Server` 是一个 [`EventEmitter`][]，具有以下事件：

### 事件：`'close'`

<!-- YAML
added: v0.5.0
-->

当服务器关闭时触发。如果存在连接，则直到所有连接都结束后才会触发此事件。

### 事件：`'connection'`

<!-- YAML
added: v0.1.90
-->

* 类型：{net.Socket} 连接对象

当建立新连接时触发。`socket` 是 `net.Socket` 的一个实例。

### 事件：`'error'`

<!-- YAML
added: v0.1.90
-->

* 类型：{Error}

当发生错误时触发。与 [`net.Socket`][] 不同，除非手动调用 [`server.close()`][]，否则在此事件之后 **不会** 直接触发 [`'close'`][] 事件。请参见 [`server.listen()`][] 讨论中的示例。

### 事件：`'listening'`

<!-- YAML
added: v0.1.90
-->

在调用 [`server.listen()`][] 后，服务器已绑定时触发。

### 事件：`'drop'`

<!-- YAML
added:
  - v18.6.0
  - v16.17.0
-->

当连接数达到 `server.maxConnections` 的阈值时，服务器将丢弃新连接并触发 `'drop'` 事件。如果是 TCP 服务器，参数如下，否则参数为 `undefined`。

* `data` {Object} 传递给事件监听器的参数。
  * `localAddress` {string} 本地地址。
  * `localPort` {number} 本地端口。
  * `localFamily` {string} 本地协议族。
  * `remoteAddress` {string} 远程地址。
  * `remotePort` {number} 远程端口。
  * `remoteFamily` {string} 远程 IP 协议族。`'IPv4'` 或 `'IPv6'`。

### `server.address()`

<!-- YAML
added: v0.1.90
changes:
  - version: v18.4.0
    pr-url: https://github.com/nodejs/node/pull/43054
    description: The `family` property now returns a string instead of a number.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41431
    description: The `family` property now returns a number instead of a string.
-->

* 返回：{Object|string|null}

返回服务器绑定的 `address`、地址 `family` 名称和 `port`，如操作系统所报告（对于监听 IP 套接字很有用，可以找到在获取操作系统分配的地址时分配了哪个端口）：
`{ port: 12346, family: 'IPv4', address: '127.0.0.1' }`。

对于监听管道或 Unix 域套接字的服务器，名称作为字符串返回。

```js
const server = net.createServer((socket) => {
  socket.end('goodbye\n');
}).on('error', (err) => {
  // 在这里处理错误。
  throw err;
});

// 获取任意未使用的端口。
server.listen(() => {
  console.log('opened server on', server.address());
});
```

在 `'listening'` 事件触发之前或调用 `server.close()` 之后，`server.address()` 返回 `null`。

### `server.close([callback])`

<!-- YAML
added: v0.1.90
-->

* `callback` {Function} 当服务器关闭时调用。
* 返回：{net.Server}

停止服务器接受新连接并保持现有连接。此函数是异步的，当所有连接都结束且服务器触发 [`'close'`][] 事件时，服务器最终关闭。可选的 `callback` 将在 `'close'` 事件发生时调用一次。与该事件不同，如果服务器在关闭时未打开，它将使用一个 `Error` 作为其唯一参数调用。

### `server[Symbol.asyncDispose]()`

<!-- YAML
added:
 - v20.5.0
 - v18.18.0
changes:
 - version: v24.2.0
   pr-url: https://github.com/nodejs/node/pull/58467
   description: No longer experimental.
-->

调用 [`server.close()`][] 并返回一个在服务器关闭时完成的 promise。

### `server.getConnections(callback)`

<!-- YAML
added: v0.9.7
-->

* `callback` {Function}
* 返回：{net.Server}

异步获取服务器上的并发连接数。当套接字被发送到分支时也有效。

回调应接受两个参数 `err` 和 `count`。

### `server.listen()`

启动服务器监听连接。`net.Server` 可以是 TCP 或 [IPC][] 服务器，具体取决于它监听的内容。

可能的签名：

* [`server.listen(handle[, backlog][, callback])`][`server.listen(handle)`]
* [`server.listen(options[, callback])`][`server.listen(options)`]
* [`server.listen(path[, backlog][, callback])`][`server.listen(path)`] 用于 [IPC][] 服务器
* [`server.listen([port[, host[, backlog]]][, callback])`][`server.listen(port)`] 用于 TCP 服务器

此函数是异步的。当服务器开始监听时，将触发 [`'listening'`][] 事件。最后一个参数 `callback` 将被添加为 [`'listening'`][] 事件的监听器。

所有 `listen()` 方法都可以接受一个 `backlog` 参数来指定待处理连接队列的最大长度。实际长度由操作系统通过 sysctl 设置确定，例如 Linux 上的 `tcp_max_syn_backlog` 和 `somaxconn`。此参数的默认值为 511（不是 512）。

所有 [`net.Socket`][] 都设置为 `SO_REUSEADDR`（有关详细信息，请参见 [`socket(7)`][]）。

当且仅当第一次 `server.listen()` 调用期间发生错误或已调用 `server.close()` 时，可以再次调用 `server.listen()` 方法。否则，将抛出 `ERR_SERVER_ALREADY_LISTEN` 错误。

监听时最常见的错误之一是 `EADDRINUSE`。当另一个服务器已经在请求的 `port`/`path`/`handle` 上监听时，会发生这种情况。处理此问题的一种方法是在一段时间后重试：

```js
server.on('error', (e) => {
  if (e.code === 'EADDRINUSE') {
    console.error('Address in use, retrying...');
    setTimeout(() => {
      server.close();
      server.listen(PORT, HOST);
    }, 1000);
  }
});
```

#### `server.listen(handle[, backlog][, callback])`

<!-- YAML
added: v0.5.10
-->

* `handle` {Object}
* `backlog` {number} [`server.listen()`][] 函数的通用参数
* `callback` {Function}
* 返回：{net.Server}

启动服务器监听给定 `handle` 上的连接，该 `handle` 已经绑定到端口、Unix 域套接字或 Windows 命名管道。

`handle` 对象可以是服务器、套接字（任何具有底层 `_handle` 成员的对象）或具有有效文件描述符的 `fd` 成员的对象。

在 Windows 上不支持监听文件描述符。

#### `server.listen(options[, callback])`

<!-- YAML
added: v0.11.14
changes:
  - version:
    - v23.1.0
    - v22.12.0
    pr-url: https://github.com/nodejs/node/pull/55408
    description: The `reusePort` option is supported.
  - version: v15.6.0
    pr-url: https://github.com/nodejs/node/pull/36623
    description: AbortSignal support was added.
  - version: v11.4.0
    pr-url: https://github.com/nodejs/node/pull/23798
    description: The `ipv6Only` option is supported.
-->

* `options` {Object} 必需。支持以下属性：
  * `backlog` {number} [`server.listen()`][] 函数的通用参数。
  * `exclusive` {boolean} **默认值：** `false`
  * `host` {string}
  * `ipv6Only` {boolean} 对于 TCP 服务器，将 `ipv6Only` 设置为 `true` 将禁用双栈支持，即绑定到主机 `::` 不会绑定 `0.0.0.0`。**默认值：** `false`。
  * `reusePort` {boolean} 对于 TCP 服务器，将 `reusePort` 设置为 `true` 允许同一主机上的多个套接字绑定到同一端口。传入连接由操作系统分配到监听套接字。此选项仅在部分平台上可用，例如 Linux 3.9+、DragonFlyBSD 3.6+、FreeBSD 12.0+、Solaris 11.4 和 AIX 7.2.5+。**默认值：** `false`。
  * `path` {string} 如果指定了 `port`，则忽略此选项。参见 [Identifying paths for IPC connections][]。
  * `port` {number}
  * `readableAll` {boolean} 对于 IPC 服务器，使管道对所有用户可读。**默认值：** `false`。
  * `signal` {AbortSignal} 可用于关闭监听服务器的 AbortSignal。
  * `writableAll` {boolean} 对于 IPC 服务器，使管道对所有用户可写。**默认值：** `false`。
* `callback` {Function}
  函数。
* 返回：{net.Server}

如果指定了 `port`，其行为与 [`server.listen([port[, host[, backlog]]][, callback])`][`server.listen(port)`] 相同。否则，如果指定了 `path`，其行为与 [`server.listen(path[, backlog][, callback])`][`server.listen(path)`] 相同。如果都未指定，将抛出错误。

如果 `exclusive` 为 `false`（默认），则集群工作进程将使用相同的底层句柄，允许共享连接处理职责。当 `exclusive` 为 `true` 时，句柄不共享，尝试共享端口将导致错误。下面显示了一个监听独占端口的示例。

```js
server.listen({
  host: 'localhost',
  port: 80,
  exclusive: true,
});
```

当 `exclusive` 为 `true` 且底层句柄共享时，多个工作进程可能使用不同的 backlog 查询句柄。在这种情况下，将使用传递给主进程的第一个 `backlog`。

以 root 身份启动 IPC 服务器可能导致服务器路径对无特权用户不可访问。使用 `readableAll` 和 `writableAll` 将使服务器对所有用户可访问。

如果启用了 `signal` 选项，在相应的 `AbortController` 上调用 `.abort()` 类似于在服务器上调用 `.close()`：

```js
const controller = new AbortController();
server.listen({
  host: 'localhost',
  port: 80,
  signal: controller.signal,
});
// 之后，当你想关闭服务器时。
controller.abort();
```

#### `server.listen(path[, backlog][, callback])`

<!-- YAML
added: v0.1.90
-->

* `path` {string} 服务器应监听的路径。参见 [Identifying paths for IPC connections][]。
* `backlog` {number} [`server.listen()`][] 函数的通用参数。
* `callback` {Function}。
* 返回：{net.Server}

启动 [IPC][] 服务器监听给定 `path` 上的连接。

#### `server.listen([port[, host[, backlog]]][, callback])`

<!-- YAML
added: v0.1.90
-->

* `port` {number}
* `host` {string}
* `backlog` {number} [`server.listen()`][] 函数的通用参数。
* `callback` {Function}。
* 返回：{net.Server}

启动 TCP 服务器监听给定 `port` 和 `host` 上的连接。

如果省略 `port` 或为 0，操作系统将分配一个任意未使用的端口，可以在 [`'listening'`][] 事件触发后使用 `server.address().port` 检索。

如果省略 `host`，当 IPv6 可用时，服务器将接受 [未指定的 IPv6 地址][]（`::`）上的连接，否则接受 [未指定的 IPv4 地址][]（`0.0.0.0`）上的连接。

在大多数操作系统中，监听 [未指定的 IPv6 地址][]（`::`）可能导致 `net.Server` 也监听 [未指定的 IPv4 地址][]（`0.0.0.0`）。

### `server.listening`

<!-- YAML
added: v5.7.0
-->

* 类型：{boolean} 指示服务器是否正在监听连接。

### `server.maxConnections`

<!-- YAML
added: v0.2.0
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/48276
    description: Setting `maxConnections` to `0` drops all the incoming
                 connections. Previously, it was interpreted as `Infinity`.
-->

* 类型：{integer}

当连接数达到 `server.maxConnections` 阈值时：

1. 如果进程未在集群模式下运行，Node.js 将关闭连接。

2. 如果进程在集群模式下运行，默认情况下，Node.js 会将连接路由到另一个工作进程。要改为关闭连接，请设置 \[`server.dropMaxConnection`]\[] 为 `true`。

一旦套接字通过 [`child_process.fork()`][] 发送给子进程，不建议使用此选项。

### `server.dropMaxConnection`

<!-- YAML
added:
  - v23.1.0
  - v22.12.0
-->

* 类型：{boolean}

将此属性设置为 `true`，以便在连接数达到 \[`server.maxConnections`]\[] 阈值时开始关闭连接。此设置仅在集群模式下有效。

### `server.ref()`

<!-- YAML
added: v0.9.1
-->

* 返回：{net.Server}

与 `unref()` 相反，在先前 `unref` 的服务器上调用 `ref()` 将 _不会_ 让程序退出（如果它是唯一的剩余服务器，这是默认行为）。如果服务器已经是 `ref` 状态，再次调用 `ref()` 将无效。

### `server.unref()`

<!-- YAML
added: v0.9.1
-->

* 返回：{net.Server}

在服务器上调用 `unref()` 将允许程序退出，如果这是事件系统中唯一的活动服务器。如果服务器已经是 `unref` 状态，再次调用 `unref()` 将无效。

## 类：`net.Socket`

<!-- YAML
added: v0.3.4
-->

* 扩展：{stream.Duplex}

此类是 TCP 套接字或流式 [IPC][] 端点（在 Windows 上使用命名管道，在其他系统上使用 Unix 域套接字）的抽象。它也是一个 [`EventEmitter`][]。

`net.Socket` 可以由用户创建并直接用于与服务器交互。例如，它由 [`net.createConnection()`][] 返回，因此用户可以使用它与服务器通信。

它也可以由 Node.js 创建并在接收到连接时传递给用户。例如，它被传递给 [`net.Server`][] 上触发的 [`'connection'`][] 事件的监听器，因此用户可以使用它与客户端交互。

### `new net.Socket([options])`

<!-- YAML
added: v0.3.4
changes:
  - version: v15.14.0
    pr-url: https://github.com/nodejs/node/pull/37735
    description: AbortSignal support was added.
  - version: v12.10.0
    pr-url: https://github.com/nodejs/node/pull/25436
    description: Added `onread` option.
-->

* `options` {Object} 可用选项有：
  * `allowHalfOpen` {boolean} 如果设置为 `false`，则当可读端结束时，套接字将自动结束可写端。有关详细信息，请参见 [`net.createServer()`][] 和 [`'end'`][] 事件。**默认值：** `false`。
  * `fd` {number} 如果指定，则包装具有给定文件描述符的现有套接字，否则将创建新套接字。
  * `onread` {Object} 如果指定，传入数据存储在单个 `buffer` 中，并在数据到达套接字时传递给提供的 `callback`。这将导致流功能不提供任何数据。套接字将像通常一样触发 `'error'`、`'end'` 和 `'close'` 事件。像 `pause()` 和 `resume()` 这样的方法也会按预期行为。
    * `buffer` {Buffer|Uint8Array|Function} 用于存储传入数据的可重用内存块或返回此类内存块的函数。
    * `callback` {Function} 此函数针对每个传入数据块调用。传递给它两个参数：写入 `buffer` 的字节数和 `buffer` 的引用。从此函数返回 `false` 以隐式 `pause()` 套接字。此函数将在全局上下文中执行。
  * `readable` {boolean} 当传递 `fd` 时允许在套接字上读取，否则忽略。**默认值：** `false`。
  * `signal` {AbortSignal} 可用于销毁套接字的 Abort 信号。
  * `writable` {boolean} 当传递 `fd` 时允许在套接字上写入，否则忽略。**默认值：** `false`。
* 返回：{net.Socket}

创建一个新的套接字对象。

新创建的套接字可以是 TCP 套接字或流式 [IPC][] 端点，具体取决于它 [`connect()`][`socket.connect()`] 到什么。

### 事件：`'close'`

<!-- YAML
added: v0.1.90
-->

* `hadError` {boolean} 如果套接字有传输错误，则为 `true`。

当套接字完全关闭时触发一次。参数 `hadError` 是一个布尔值，表示套接字是否因传输错误而关闭。

### 事件：`'connect'`

<!-- YAML
added: v0.1.90
-->

当套接字连接成功建立时触发。参见 [`net.createConnection()`][]。

### 事件：`'connectionAttempt'`

<!-- YAML
added:
  - v21.6.0
  - v20.12.0
-->

* `ip` {string} 套接字尝试连接到的 IP。
* `port` {number} 套接字尝试连接到的端口。
* `family` {number} IP 的协议族。可以是 IPv6 的 `6` 或 IPv4 的 `4`。

当开始新的连接尝试时触发。如果在 [`socket.connect(options)`][] 中启用了协议族自动选择算法，则可能多次触发此事件。

### 事件：`'connectionAttemptFailed'`

<!-- YAML
added:
  - v21.6.0
  - v20.12.0
-->

* `ip` {string} 套接字尝试连接到的 IP。
* `port` {number} 套接字尝试连接到的端口。
* `family` {number} IP 的协议族。可以是 IPv6 的 `6` 或 IPv4 的 `4`。
* `error` {Error} 与失败相关的错误。

当连接尝试失败时触发。如果在 [`socket.connect(options)`][] 中启用了协议族自动选择算法，则可能多次触发此事件。

### 事件：`'connectionAttemptTimeout'`

<!-- YAML
added:
  - v21.6.0
  - v20.12.0
-->

* `ip` {string} 套接字尝试连接到的 IP。
* `port` {number} 套接字尝试连接到的端口。
* `family` {number} IP 的协议族。可以是 IPv6 的 `6` 或 IPv4 的 `4`。

当连接尝试超时时触发。仅当在 [`socket.connect(options)`][] 中启用了协议族自动选择算法时，才会触发此事件（并且可能多次触发）。

### 事件：`'data'`

<!-- YAML
added: v0.1.90
-->

* 类型：{Buffer|string}

当接收到数据时触发。参数 `data` 将是 `Buffer` 或 `String`。数据的编码由 [`socket.setEncoding()`][] 设置。

当 `Socket` 触发 `'data'` 事件时没有监听器，数据将丢失。

### 事件：`'drain'`

<!-- YAML
added: v0.1.90
-->

当写入缓冲区变空时触发。可用于限制上传速度。

另请参见：`socket.write()` 的返回值。

### 事件：`'end'`

<!-- YAML
added: v0.1.90
-->

当套接字的另一端发出传输结束信号时触发，从而结束套接字的可读端。

默认情况下（`allowHalfOpen` 为 `false`），套接字将发送一个传输结束包回去，并在写出其待处理写入队列后销毁其文件描述符。但是，如果 `allowHalfOpen` 设置为 `true`，套接字将不会自动 [`end()`][`socket.end()`] 其可写端，允许用户写入任意数量的数据。用户必须显式调用 [`end()`][`socket.end()`] 来关闭连接（即发送 FIN 包回去）。

### 事件：`'error'`

<!-- YAML
added: v0.1.90
-->

* 类型：{Error}

当发生错误时触发。`'close'` 事件将在此事件之后直接调用。

### 事件：`'lookup'`

<!-- YAML
added: v0.11.3
changes:
  - version: v5.10.0
    pr-url: https://github.com/nodejs/node/pull/5598
    description: The `host` parameter is supported now.
-->

在解析主机名之后但在连接之前触发。不适用于 Unix 套接字。

* `err` {Error|null} 错误对象。参见 [`dns.lookup()`][]。
* `address` {string} IP 地址。
* `family` {number|null} 地址类型。参见 [`dns.lookup()`][]。
* `host` {string} 主机名。

### 事件：`'ready'`

<!-- YAML
added: v9.11.0
-->

当套接字准备就绪可供使用时触发。

在 `'connect'` 之后立即触发。

### 事件：`'timeout'`

<!-- YAML
added: v0.1.90
-->

如果套接字因不活动而超时，则触发。这只是通知套接字已空闲。用户必须手动关闭连接。

另请参见：[`socket.setTimeout()`][]。

### `socket.address()`

<!-- YAML
added: v0.1.90
changes:
  - version: v18.4.0
    pr-url: https://github.com/nodejs/node/pull/43054
    description: The `family` property now returns a string instead of a number.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41431
    description: The `family` property now returns a number instead of a string.
-->

* 返回：{Object}

返回套接字绑定的 `address`、地址 `family` 名称和 `port`，如操作系统所报告：
`{ port: 12346, family: 'IPv4', address: '127.0.0.1' }`

### `socket.autoSelectFamilyAttemptedAddresses`

<!-- YAML
added:
 - v19.4.0
 - v18.18.0
-->

* 类型：{string\[]}

此属性仅当在 [`socket.connect(options)`][] 中启用了协议族自动选择算法时才存在，并且它是一个已尝试地址的数组。

每个地址都是 `$IP:$PORT` 形式的字符串。如果连接成功，则最后一个地址是套接字当前连接到的地址。

### `socket.bufferSize`

<!-- YAML
added: v0.3.8
deprecated:
  - v14.6.0
-->

> Stability: 0 - Deprecated: Use [`writable.writableLength`][] instead.

* 类型：{integer}

此属性显示为写入缓冲的字符数。缓冲区可能包含编码后长度尚未知的字符串。因此，此数字只是缓冲区中字节数的近似值。

`net.Socket` 具有 `socket.write()` 始终有效的特性。这是为了帮助用户快速上手。计算机可能无法始终跟上写入套接字的数据量。网络连接可能太慢。Node.js 将在内部排队写入套接字的数据，并在可能时通过网络发送。

这种内部缓冲的后果是内存可能会增长。遇到较大或不断增长的 `bufferSize` 的用户应尝试通过 [`socket.pause()`][] 和 [`socket.resume()`][] 来“限制”其程序中的数据流。

### `socket.bytesRead`

<!-- YAML
added: v0.5.3
-->

* 类型：{integer}

接收到的字节数。

### `socket.bytesWritten`

<!-- YAML
added: v0.5.3
-->

* 类型：{integer}

发送的字节数。

### `socket.connect()`

在给定套接字上发起连接。

可能的签名：

* [`socket.connect(options[, connectListener])`][`socket.connect(options)`]
* [`socket.connect(path[, connectListener])`][`socket.connect(path)`] 用于 [IPC][] 连接。
* [`socket.connect(port[, host][, connectListener])`][`socket.connect(port)`] 用于 TCP 连接。
* 返回：{net.Socket} 套接字本身。

此函数是异步的。当连接建立时，将触发 [`'connect'`][] 事件。如果连接出现问题，将触发 [`'error'`][] 事件（而不是 [`'connect'`][] 事件），并将错误传递给 [`'error'`][] 监听器。最后一个参数 `connectListener`（如果提供）将在 **一次** 被添加为 [`'connect'`][] 事件的监听器。

此函数仅应在 `'close'` 事件触发后重新连接套接字时使用，否则可能导致未定义行为。

#### `socket.connect(options[, connectListener])`

<!-- YAML
added: v0.1.90
changes:
  - version:
      - v20.0.0
      - v18.18.0
    pr-url: https://github.com/nodejs/node/pull/46790
    description: The default value for the autoSelectFamily option is now true.
                 The `--enable-network-family-autoselection` CLI flag has been renamed
                 to `--network-family-autoselection`. The old name is now an
                 alias but it is discouraged.
  - version: v19.4.0
    pr-url: https://github.com/nodejs/node/pull/45777
    description: The default value for autoSelectFamily option can be changed
                 at runtime using `setDefaultAutoSelectFamily` or via the
                 command line option `--enable-network-family-autoselection`.
  - version:
      - v19.3.0
      - v18.13.0
    pr-url: https://github.com/nodejs/node/pull/44731
    description: Added the `autoSelectFamily` option.
  - version:
    - v17.7.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/41310
    description: The `noDelay`, `keepAlive`, and `keepAliveInitialDelay`
                 options are supported now.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/6021
    description: The `hints` option defaults to `0` in all cases now.
                 Previously, in the absence of the `family` option it would
                 default to `dns.ADDRCONFIG | dns.V4MAPPED`.
  - version: v5.11.0
    pr-url: https://github.com/nodejs/node/pull/6000
    description: The `hints` option is supported now.
-->

* `options` {Object}
* `connectListener` {Function} [`socket.connect()`][] 方法的通用参数。将在一次被添加为 [`'connect'`][] 事件的监听器。
* 返回：{net.Socket} 套接字本身。

在给定套接字上发起连接。通常不需要此方法，应使用 [`net.createConnection()`][] 创建和打开套接字。仅在实现自定义套接字时使用此方法。

对于 TCP 连接，可用的 `options` 有：

* `autoSelectFamily` {boolean}: 如果设置为 `true`，则启用一个协议族自动检测算法，该算法松散地实现了 [RFC 8305][] 的第 5 节。传递给查找的 `all` 选项设置为 `true`，套接字尝试按顺序连接到所有获取的 IPv6 和 IPv4 地址，直到建立连接。首先尝试第一个返回的 AAAA 地址，然后是第一个返回的 A 地址，然后是第二个返回的 AAAA 地址，依此类推。每个连接尝试（除了最后一个）在超时并尝试下一个地址之前，会被赋予 `autoSelectFamilyAttemptTimeout` 选项指定的时间量。如果 `family` 选项不是 `0` 或设置了 `localAddress`，则忽略此选项。如果至少有一个连接成功，则不会发出连接错误。如果所有连接尝试都失败，则发出一个包含所有失败尝试的 `AggregateError`。**默认值：** [`net.getDefaultAutoSelectFamily()`][]。
* `autoSelectFamilyAttemptTimeout` {number}: 当使用 `autoSelectFamily` 选项时，在尝试下一个地址之前等待连接尝试完成的时间量（以毫秒为单位）。如果设置为小于 `10` 的正整数，则将使用值 `10`。**默认值：** [`net.getDefaultAutoSelectFamilyAttemptTimeout()`][]。
* `family` {number}: IP 栈的版本。必须为 `4`、`6` 或 `0`。值 `0` 表示允许 IPv4 和 IPv6 地址。**默认值：** `0`。
* `hints` {number} 可选的 [`dns.lookup()` hints][]。
* `host` {string} 套接字应连接到的目标主机。**默认值：** `'localhost'`。
* `keepAlive` {boolean} 如果设置为 `true`，则在连接建立后立即在套接字上启用 keep-alive 功能，类似于在 [`socket.setKeepAlive()`][] 中所做的操作。**默认值：** `false`。
* `keepAliveInitialDelay` {number} 如果设置为正数，则设置在空闲套接字上发送第一个 keepalive 探测之前的初始延迟。**默认值：** `0`。
* `localAddress` {string} 套接字应连接来自的本地地址。
* `localPort` {number} 套接字应连接来自的本地端口。
* `lookup` {Function} 自定义查找函数。**默认值：** [`dns.lookup()`][]。
* `noDelay` {boolean} 如果设置为 `true`，则在套接字建立后立即禁用 Nagle 算法的使用。**默认值：** `false`。
* `port` {number} 必需。套接字应连接到的目标端口。
* `blockList` {net.BlockList} `blockList` 可用于禁用对特定 IP 地址、IP 范围或 IP 子网的出站访问。

对于 [IPC][] 连接，可用的 `options` 有：

* `path` {string} 必需。客户端应连接到的路径。参见 [Identifying paths for IPC connections][]。如果提供，则忽略上述 TCP 特定选项。

#### `socket.connect(path[, connectListener])`

* `path` {string} 客户端应连接到的路径。参见 [Identifying paths for IPC connections][]。
* `connectListener` {Function} [`socket.connect()`][] 方法的通用参数。将在一次被添加为 [`'connect'`][] 事件的监听器。
* 返回：{net.Socket} 套接字本身。

在给定套接字上发起 [IPC][] 连接。

是 [`socket.connect(options[, connectListener])`][`socket.connect(options)`] 的别名，调用时 `options` 为 `{ path: path }`。

#### `socket.connect(port[, host][, connectListener])`

<!-- YAML
added: v0.1.90
-->

* `port` {number} 客户端应连接到的目标端口。
* `host` {string} 客户端应连接到的目标主机。
* `connectListener` {Function} [`socket.connect()`][] 方法的通用参数。将在一次被添加为 [`'connect'`][] 事件的监听器。
* 返回：{net.Socket} 套接字本身。

在给定套接字上发起 TCP 连接。

是 [`socket.connect(options[, connectListener])`][`socket.connect(options)`] 的别名，调用时 `options` 为 `{port: port, host: host}`。

### `socket.connecting`

<!-- YAML
added: v6.1.0
-->

* 类型：{boolean}

如果为 `true`，则表示 [`socket.connect(options[, connectListener])`][`socket.connect(options)`] 已被调用且尚未完成。它将保持 `true` 直到套接字连接成功，然后设置为 `false` 并触发 `'connect'` 事件。请注意，[`socket.connect(options[, connectListener])`][`socket.connect(options)`] 的回调是 `'connect'` 事件的监听器。

### `socket.destroy([error])`

<!-- YAML
added: v0.1.90
-->

* `error` {Object}
* 返回：{net.Socket}

确保在此套接字上不再发生 I/O 活动。销毁流并关闭连接。

有关详细信息，请参见 [`writable.destroy()`][]。

### `socket.destroyed`

* 类型：{boolean} 指示连接是否已销毁。一旦连接被销毁，就无法使用它传输更多数据。

有关详细信息，请参见 [`writable.destroyed`][]。

### `socket.destroySoon()`

<!-- YAML
added: v0.3.4
-->

在所有数据写入后销毁套接字。如果 `'finish'` 事件已经触发，则套接字立即销毁。如果套接字仍然可写，则隐式调用 `socket.end()`。

### `socket.end([data[, encoding]][, callback])`

<!-- YAML
added: v0.1.90
-->

* `data` {string|Buffer|Uint8Array}
* `encoding` {string} 仅当 data 为 `string` 时使用。**默认值：** `'utf8'`。
* `callback` {Function} 套接字完成时的可选回调。
* 返回：{net.Socket} 套接字本身。

半关闭套接字。即，它发送一个 FIN 包。服务器可能仍会发送一些数据。

有关详细信息，请参见 [`writable.end()`][]。

### `socket.localAddress`

<!-- YAML
added: v0.9.6
-->

* 类型：{string}

远程客户端正在连接的本地 IP 地址的字符串表示形式。例如，在监听 `'0.0.0.0'` 的服务器中，如果客户端在 `'192.168.1.1'` 上连接，则 `socket.localAddress` 的值将为 `'192.168.1.1'`。

### `socket.localPort`

<!-- YAML
added: v0.9.6
-->

* 类型：{integer}

本地端口的数字表示形式。例如，`80` 或 `21`。

### `socket.localFamily`

<!-- YAML
added:
  - v18.8.0
  - v16.18.0
-->

* 类型：{string}

本地 IP 协议族的字符串表示形式。`'IPv4'` 或 `'IPv6'`。

### `socket.pause()`

* 返回：{net.Socket} 套接字本身。

暂停数据读取。也就是说，不会触发 [`'data'`][] 事件。可用于限制上传速度。

### `socket.pending`

<!-- YAML
added:
 - v11.2.0
 - v10.16.0
-->

* 类型：{boolean}

如果套接字尚未连接，则为 `true`，要么是因为尚未调用 `.connect()`，要么是因为它仍在连接过程中（参见 [`socket.connecting`][]）。

### `socket.ref()`

<!-- YAML
added: v0.9.1
-->

* 返回：{net.Socket} 套接字本身。

与 `unref()` 相反，在先前 `unref` 的套接字上调用 `ref()` 将 _不会_ 让程序退出（如果它是唯一的剩余套接字，这是默认行为）。如果套接字已经是 `ref` 状态，再次调用 `ref` 将无效。

### `socket.remoteAddress`

<!-- YAML
added: v0.5.10
-->

* 类型：{string}

远程 IP 地址的字符串表示形式。例如，`'74.125.127.100'` 或 `'2001:4860:a005::68'`。如果套接字被销毁（例如，如果客户端断开连接），值可能为 `undefined`。

### `socket.remoteFamily`

<!-- YAML
added: v0.11.14
-->

* 类型：{string}

远程 IP 协议族的字符串表示形式。`'IPv4'` 或 `'IPv6'`。如果套接字被销毁（例如，如果客户端断开连接），值可能为 `undefined`。

### `socket.remotePort`

<!-- YAML
added: v0.5.10
-->

* 类型：{integer}

远程端口的数字表示形式。例如，`80` 或 `21`。如果套接字被销毁（例如，如果客户端断开连接），值可能为 `undefined`。

### `socket.resetAndDestroy()`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

* 返回：{net.Socket}

通过发送 RST 包关闭 TCP 连接并销毁流。如果此 TCP 套接字处于连接状态，它将在连接后发送 RST 包并销毁此 TCP 套接字。否则，它将使用 `ERR_SOCKET_CLOSED` 错误调用 `socket.destroy`。如果这不是 TCP 套接字（例如，管道），调用此方法将立即抛出 `ERR_INVALID_HANDLE_TYPE` 错误。

### `socket.resume()`

* 返回：{net.Socket} 套接字本身。

在调用 [`socket.pause()`][] 后恢复读取。

### `socket.setEncoding([encoding])`

<!-- YAML
added: v0.1.90
-->

* `encoding` {string}
* 返回：{net.Socket} 套接字本身。

将套接字的编码设置为 [可读流][]。有关更多信息，请参见 [`readable.setEncoding()`][]。

### `socket.setKeepAlive([enable][, initialDelay])`

<!-- YAML
added: v0.1.92
changes:
  - version:
    - v13.12.0
    - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32204
    description: New defaults for `TCP_KEEPCNT` and `TCP_KEEPINTVL` socket options were added.
-->

* `enable` {boolean} **默认值：** `false`
* `initialDelay` {number} **默认值：** `0`
* 返回：{net.Socket} 套接字本身。

启用/禁用 keep-alive 功能，并可选择设置在空闲套接字上发送第一个 keepalive 探测之前的初始延迟。

设置 `initialDelay`（以毫秒为单位）以设置接收到的最后一个数据包与第一个 keepalive 探测之间的延迟。将 `initialDelay` 设置为 `0` 将保持默认（或先前）设置不变。

启用 keep-alive 功能将设置以下套接字选项：

* `SO_KEEPALIVE=1`
* `TCP_KEEPIDLE=initialDelay`
* `TCP_KEEPCNT=10`
* `TCP_KEEPINTVL=1`

### `socket.setNoDelay([noDelay])`

<!-- YAML
added: v0.1.90
-->

* `noDelay` {boolean} **默认值：** `true`
* 返回：{net.Socket} 套接字本身。

启用/禁用 Nagle 算法的使用。

创建 TCP 连接时，将启用 Nagle 算法。

Nagle 算法在数据通过网络发送之前延迟数据。它试图以延迟为代价来优化吞吐量。

为 `noDelay` 传递 `true` 或不传递参数将禁用套接字的 Nagle 算法。为 `noDelay` 传递 `false` 将启用 Nagle 算法。

### `socket.setTimeout(timeout[, callback])`

<!-- YAML
added: v0.1.90
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `timeout` {number}
* `callback` {Function}
* 返回：{net.Socket} 套接字本身。

将套接字设置为在套接字不活动 `timeout` 毫秒后超时。默认情况下 `net.Socket` 没有超时。

当触发空闲超时时，套接字将收到 [`'timeout'`][] 事件，但连接不会断开。用户必须手动调用 [`socket.end()`][] 或 [`socket.destroy()`][] 来结束连接。

```js
socket.setTimeout(3000);
socket.on('timeout', () => {
  console.log('socket timeout');
  socket.end();
});
```

如果 `timeout` 为 0，则禁用现有的空闲超时。

可选的 `callback` 参数将作为一次性监听器添加到 [`'timeout'`][] 事件。

### `socket.timeout`

<!-- YAML
added: v10.7.0
-->

* 类型：{number|undefined}

由 [`socket.setTimeout()`][] 设置的套接字超时时间（以毫秒为单位）。如果未设置超时，则为 `undefined`。

### `socket.unref()`

<!-- YAML
added: v0.9.1
-->

* 返回：{net.Socket} 套接字本身。

在套接字上调用 `unref()` 将允许程序退出，如果这是事件系统中唯一的活动套接字。如果套接字已经是 `unref` 状态，再次调用 `unref()` 将无效。

### `socket.write(data[, encoding][, callback])`

<!-- YAML
added: v0.1.90
-->

* `data` {string|Buffer|Uint8Array}
* `encoding` {string} 仅当 data 为 `string` 时使用。**默认值：** `utf8`。
* `callback` {Function}
* 返回：{boolean}

在套接字上发送数据。第二个参数在字符串的情况下指定编码。默认为 UTF8 编码。

如果所有数据成功刷新到内核缓冲区，则返回 `true`。如果所有或部分数据在用户内存中排队，则返回 `false`。当缓冲区再次空闲时，将触发 [`'drain'`][]。

可选的 `callback` 参数将在数据最终写出时执行，这可能不是立即的。

有关更多信息，请参见 `Writable` 流 [`write()`][stream_writable_write] 方法。

### `socket.readyState`

<!-- YAML
added: v0.5.0
-->

* 类型：{string}

此属性以字符串形式表示连接的状态。

* 如果流正在连接，`socket.readyState` 为 `opening`。
* 如果流可读且可写，则为 `open`。
* 如果流可读但不可写，则为 `readOnly`。
* 如果流不可读但可写，则为 `writeOnly`。

## `net.connect()`

是 [`net.createConnection()`][`net.createConnection()`] 的别名。

可能的签名：

* [`net.connect(options[, connectListener])`][`net.connect(options)`]
* [`net.connect(path[, connectListener])`][`net.connect(path)`] 用于 [IPC][] 连接。
* [`net.connect(port[, host][, connectListener])`][`net.connect(port, host)`] 用于 TCP 连接。

### `net.connect(options[, connectListener])`

<!-- YAML
added: v0.7.0
-->

* `options` {Object}
* `connectListener` {Function}
* 返回：{net.Socket}

是 [`net.createConnection(options[, connectListener])`][`net.createConnection(options)`] 的别名。

### `net.connect(path[, connectListener])`

<!-- YAML
added: v0.1.90
-->

* `path` {string}
* `connectListener` {Function}
* 返回：{net.Socket}

是 [`net.createConnection(path[, connectListener])`][`net.createConnection(path)`] 的别名。

### `net.connect(port[, host][, connectListener])`

<!-- YAML
added: v0.1.90
-->

* `port` {number}
* `host` {string}
* `connectListener` {Function}
* 返回：{net.Socket}

是 [`net.createConnection(port[, host][, connectListener])`][`net.createConnection(port, host)`] 的别名。

## `net.createConnection()`

一个工厂函数，它创建一个新的 [`net.Socket`][]，立即使用 [`socket.connect()`][] 发起连接，然后返回启动连接的 `net.Socket`。

当连接建立时，将在返回的套接字上触发 [`'connect'`][] 事件。最后一个参数 `connectListener`（如果提供）将在 **一次** 被添加为 [`'connect'`][] 事件的监听器。

可能的签名：

* [`net.createConnection(options[, connectListener])`][`net.createConnection(options)`]
* [`net.createConnection(path[, connectListener])`][`net.createConnection(path)`] 用于 [IPC][] 连接。
* [`net.createConnection(port[, host][, connectListener])`][`net.createConnection(port, host)`] 用于 TCP 连接。

[`net.connect()`][] 函数是此函数的别名。

### `net.createConnection(options[, connectListener])`

<!-- YAML
added: v0.1.90
-->

* `options` {Object} 必需。将传递给 [`new net.Socket([options])`][`new net.Socket(options)`] 调用和 [`socket.connect(options[, connectListener])`][`socket.connect(options)`] 方法。
* `connectListener` {Function} [`net.createConnection()`][] 函数的通用参数。如果提供，将在一次被添加为返回套接字上的 [`'connect'`][] 事件的监听器。
* 返回：{net.Socket} 用于启动连接的新创建套接字。

有关可用选项，请参见 [`new net.Socket([options])`][`new net.Socket(options)`] 和 [`socket.connect(options[, connectListener])`][`socket.connect(options)`]。

其他选项：

* `timeout` {number} 如果设置，将在套接字创建之后、开始连接之前用于调用 [`socket.setTimeout(timeout)`][]。

以下是 [`net.createServer()`][] 部分中描述的 echo 服务器客户端的示例：

```mjs
import net from 'node:net';
const client = net.createConnection({ port: 8124 }, () => {
  // 'connect' 监听器。
  console.log('connected to server!');
  client.write('world!\r\n');
});
client.on('data', (data) => {
  console.log(data.toString());
  client.end();
});
client.on('end', () => {
  console.log('disconnected from server');
});
```

```cjs
const net = require('node:net');
const client = net.createConnection({ port: 8124 }, () => {
  // 'connect' 监听器。
  console.log('connected to server!');
  client.write('world!\r\n');
});
client.on('data', (data) => {
  console.log(data.toString());
  client.end();
});
client.on('end', () => {
  console.log('disconnected from server');
});
```

要连接到套接字 `/tmp/echo.sock`：

```js
const client = net.createConnection({ path: '/tmp/echo.sock' });
```

以下是使用 `port` 和 `onread` 选项的客户端示例。在这种情况下，`onread` 选项将仅用于调用 `new net.Socket([options])`，而 `port` 选项将用于调用 `socket.connect(options[, connectListener])`。

```mjs
import net from 'node:net';
import { Buffer } from 'node:buffer';
net.createConnection({
  port: 8124,
  onread: {
    // 为从套接字读取的每次操作重用 4KiB Buffer。
    buffer: Buffer.alloc(4 * 1024),
    callback: function(nread, buf) {
      // 接收到的数据在 `buf` 中从 0 到 `nread` 可用。
      console.log(buf.toString('utf8', 0, nread));
    },
  },
});
```

```cjs
const net = require('node:net');
net.createConnection({
  port: 8124,
  onread: {
    // 为从套接字读取的每次操作重用 4KiB Buffer。
    buffer: Buffer.alloc(4 * 1024),
    callback: function(nread, buf) {
      // 接收到的数据在 `buf` 中从 0 到 `nread` 可用。
      console.log(buf.toString('utf8', 0, nread));
    },
  },
});
```

### `net.createConnection(path[, connectListener])`

<!-- YAML
added: v0.1.90
-->

* `path` {string} 套接字应连接到的路径。将传递给 [`socket.connect(path[, connectListener])`][`socket.connect(path)`]。参见 [Identifying paths for IPC connections][]。
* `connectListener` {Function} [`net.createConnection()`][] 函数的通用参数，是启动套接字上 `'connect'` 事件的“一次性”监听器。将传递给 [`socket.connect(path[, connectListener])`][`socket.connect(path)`]。
* 返回：{net.Socket} 用于启动连接的新创建套接字。

发起 [IPC][] 连接。

此函数创建一个所有选项设置为默认值的新 [`net.Socket`][]，立即使用 [`socket.connect(path[, connectListener])`][`socket.connect(path)`] 发起连接，然后返回启动连接的 `net.Socket`。

### `net.createConnection(port[, host][, connectListener])`

<!-- YAML
added: v0.1.90
-->

* `port` {number} 套接字应连接到的目标端口。将传递给 [`socket.connect(port[, host][, connectListener])`][`socket.connect(port)`]。
* `host` {string} 套接字应连接到的目标主机。将传递给 [`socket.connect(port[, host][, connectListener])`][`socket.connect(port)`]。**默认值：** `'localhost'`。
* `connectListener` {Function} [`net.createConnection()`][] 函数的通用参数，是启动套接字上 `'connect'` 事件的“一次性”监听器。将传递给 [`socket.connect(port[, host][, connectListener])`][`socket.connect(port)`]。
* 返回：{net.Socket} 用于启动连接的新创建套接字。

发起 TCP 连接。

此函数创建一个所有选项设置为默认值的新 [`net.Socket`][]，立即使用 [`socket.connect(port[, host][, connectListener])`][`socket.connect(port)`] 发起连接，然后返回启动连接的 `net.Socket`。

## `net.createServer([options][, connectionListener])`

<!-- YAML
added: v0.5.0
changes:
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/47405
    description: The `highWaterMark` option is supported now.
  - version:
    - v17.7.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/41310
    description: The `noDelay`, `keepAlive`, and `keepAliveInitialDelay`
                 options are supported now.
-->

* `options` {Object}
  * `allowHalfOpen` {boolean} 如果设置为 `false`，则当可读端结束时，套接字将自动结束可写端。
    **默认值：** `false`。
  * `highWaterMark` {number} 可选地覆盖所有 [`net.Socket`][] 的 `readableHighWaterMark` 和 `writableHighWaterMark`。
    **默认值：** 参见 [`stream.getDefaultHighWaterMark()`][]。
  * `keepAlive` {boolean} 如果设置为 `true`，则在接收到新的传入连接后立即在套接字上启用 keep-alive 功能，类似于在 [`socket.setKeepAlive()`][] 中所做的操作。**默认值：**
    `false`。
  * `keepAliveInitialDelay` {number} 如果设置为正数，则设置在空闲套接字上发送第一个 keepalive 探测之前的初始延迟。
    **默认值：** `0`。
  * `noDelay` {boolean} 如果设置为 `true`，则在接收到新的传入连接后立即禁用 Nagle 算法的使用。
    **默认值：** `false`。
  * `pauseOnConnect` {boolean} 指示是否应在传入连接上暂停套接字。**默认值：** `false`。
  * `blockList` {net.BlockList} `blockList` 可用于禁用对特定 IP 地址、IP 范围或 IP 子网的入站访问。如果服务器位于反向代理、NAT 等后面，则此方法无效，因为检查阻止列表的地址是代理的地址或 NAT 指定的地址。

* `connectionListener` {Function} 自动设置为 [`'connection'`][] 事件的监听器。

* 返回：{net.Server}

创建一个新的 TCP 或 [IPC][] 服务器。

如果 `allowHalfOpen` 设置为 `true`，当套接字的另一端发出传输结束信号时，服务器只有在显式调用 [`socket.end()`][] 时才会发回传输结束信号。例如，在 TCP 的上下文中，当接收到 FIN 包时，只有在显式调用 [`socket.end()`][] 时才会发回 FIN 包。在此之前，连接处于半关闭状态（不可读但仍可写）。有关更多信息，请参见 [`'end'`][] 事件和 [RFC 1122][half-closed]（第 4.2.2.13 节）。

如果 `pauseOnConnect` 设置为 `true`，则与每个传入连接关联的套接字将被暂停，并且不会从其句柄读取任何数据。这允许连接在进程之间传递，而原始进程不会读取任何数据。要从暂停的套接字开始读取数据，请调用 [`socket.resume()`][]。

服务器可以是 TCP 服务器或 [IPC][] 服务器，具体取决于它 [`listen()`][`server.listen()`] 到什么。

以下是一个 TCP echo 服务器的示例，它监听 8124 端口上的连接：

```mjs
import net from 'node:net';
const server = net.createServer((c) => {
  // 'connection' 监听器。
  console.log('client connected');
  c.on('end', () => {
    console.log('client disconnected');
  });
  c.write('hello\r\n');
  c.pipe(c);
});
server.on('error', (err) => {
  throw err;
});
server.listen(8124, () => {
  console.log('server bound');
});
```

```cjs
const net = require('node:net');
const server = net.createServer((c) => {
  // 'connection' 监听器。
  console.log('client connected');
  c.on('end', () => {
    console.log('client disconnected');
  });
  c.write('hello\r\n');
  c.pipe(c);
});
server.on('error', (err) => {
  throw err;
});
server.listen(8124, () => {
  console.log('server bound');
});
```

使用 `telnet` 进行测试：

```bash
telnet localhost 8124
```

要监听套接字 `/tmp/echo.sock`：

```js
server.listen('/tmp/echo.sock', () => {
  console.log('server bound');
});
```

使用 `nc` 连接到 Unix 域套接字服务器：

```bash
nc -U /tmp/echo.sock
```

## `net.getDefaultAutoSelectFamily()`

<!-- YAML
added: v19.4.0
-->

获取 [`socket.connect(options)`][] 的 `autoSelectFamily` 选项的当前默认值。初始默认值为 `true`，除非提供了命令行选项 `--no-network-family-autoselection`。

* 返回：{boolean} `autoSelectFamily` 选项的当前默认值。

## `net.setDefaultAutoSelectFamily(value)`

<!-- YAML
added: v19.4.0
-->

设置 [`socket.connect(options)`][] 的 `autoSelectFamily` 选项的默认值。

* `value` {boolean} 新的默认值。
  初始默认值为 `true`，除非提供了命令行选项 `--no-network-family-autoselection`。

## `net.getDefaultAutoSelectFamilyAttemptTimeout()`

<!-- YAML
added:
 - v19.8.0
 - v18.18.0
-->

获取 [`socket.connect(options)`][] 的 `autoSelectFamilyAttemptTimeout` 选项的当前默认值。初始默认值为 `250` 或通过命令行选项 `--network-family-autoselection-attempt-timeout` 指定的值。

* 返回：{number} `autoSelectFamilyAttemptTimeout` 选项的当前默认值。

## `net.setDefaultAutoSelectFamilyAttemptTimeout(value)`

<!-- YAML
added:
 - v19.8.0
 - v18.18.0
-->

设置 [`socket.connect(options)`][] 的 `autoSelectFamilyAttemptTimeout` 选项的默认值。

* `value` {number} 新的默认值，必须为正数。如果数字小于 `10`，则改用值 `10`。初始默认值为 `250` 或通过命令行选项 `--network-family-autoselection-attempt-timeout` 指定的值。

## `net.isIP(input)`

<!-- YAML
added: v0.3.0
-->

* `input` {string}
* 返回：{integer}

如果 `input` 是 IPv6 地址，则返回 `6`。如果 `input` 是 [点分十进制表示法][] 中没有前导零的 IPv4 地址，则返回 `4`。否则，返回 `0`。

```js
net.isIP('::1'); // 返回 6
net.isIP('127.0.0.1'); // 返回 4
net.isIP('127.000.000.001'); // 返回 0
net.isIP('127.0.0.1/24'); // 返回 0
net.isIP('fhqwhgads'); // 返回 0
```

## `net.isIPv4(input)`

<!-- YAML
added: v0.3.0
-->

* `input` {string}
* 返回：{boolean}

如果 `input` 是 [点分十进制表示法][] 中没有前导零的 IPv4 地址，则返回 `true`。否则，返回 `false`。

```js
net.isIPv4('127.0.0.1'); // 返回 true
net.isIPv4('127.000.000.001'); // 返回 false
net.isIPv4('127.0.0.1/24'); // 返回 false
net.isIPv4('fhqwhgads'); // 返回 false
```

## `net.isIPv6(input)`

<!-- YAML
added: v0.3.0
-->

* `input` {string}
* 返回：{boolean}

如果 `input` 是 IPv6 地址，则返回 `true`。否则，返回 `false`。

```js
net.isIPv6('::1'); // 返回 true
net.isIPv6('fhqwhgads'); // 返回 false
```

[IPC]: #ipc-support
[Identifying paths for IPC connections]: #identifying-paths-for-ipc-connections
[RFC 8305]: https://www.rfc-editor.org/rfc/rfc8305.txt
[可读流]: stream.md#class-streamreadable
[`'close'`]: #event-close
[`'connect'`]: #event-connect
[`'connection'`]: #event-connection
[`'data'`]: #event-data
[`'drain'`]: #event-drain
[`'end'`]: #event-end
[`'error'`]: #event-error_1
[`'listening'`]: #event-listening
[`'timeout'`]: #event-timeout
[`EventEmitter`]: events.md#class-eventemitter
[`child_process.fork()`]: child_process.md#child_processforkmodulepath-args-options
[`dns.lookup()`]: dns.md#dnslookuphostname-options-callback
[`dns.lookup()` hints]: dns.md#supported-getaddrinfo-flags
[`net.Server`]: #class-netserver
[`net.Socket`]: #class-netsocket
[`net.connect()`]: #netconnect
[`net.connect(options)`]: #netconnectoptions-connectlistener
[`net.connect(path)`]: #netconnectpath-connectlistener
[`net.connect(port, host)`]: #netconnectport-host-connectlistener
[`net.createConnection()`]: #netcreateconnection
[`net.createConnection(options)`]: #netcreateconnectionoptions-connectlistener
[`net.createConnection(path)`]: #netcreateconnectionpath-connectlistener
[`net.createConnection(port, host)`]: #netcreateconnectionport-host-connectlistener
[`net.createServer()`]: #netcreateserveroptions-connectionlistener
[`net.getDefaultAutoSelectFamily()`]: #netgetdefaultautoselectfamily
[`net.getDefaultAutoSelectFamilyAttemptTimeout()`]: #netgetdefaultautoselectfamilyattempttimeout
[`new net.Socket(options)`]: #new-netsocketoptions
[`readable.setEncoding()`]: stream.md#readablesetencodingencoding
[`server.close()`]: #serverclosecallback
[`server.listen()`]: #serverlisten
[`server.listen(handle)`]: #serverlistenhandle-backlog-callback
[`server.listen(options)`]: #serverlistenoptions-callback
[`server.listen(path)`]: #serverlistenpath-backlog-callback
[`server.listen(port)`]: #serverlistenport-host-backlog-callback
[`socket(7)`]: https://man7.org/linux/man-pages/man7/socket.7.html
[`socket.connect()`]: #socketconnect
[`socket.connect(options)`]: #socketconnectoptions-connectlistener
[`socket.connect(path)`]: #socketconnectpath-connectlistener
[`socket.connect(port)`]: #socketconnectport-host-connectlistener
[`socket.connecting`]: #socketconnecting
[`socket.destroy()`]: #socketdestroyerror
[`socket.end()`]: #socketenddata-encoding-callback
[`socket.pause()`]: #socketpause
[`socket.resume()`]: #socketresume
[`socket.setEncoding()`]: #socketsetencodingencoding
[`socket.setKeepAlive()`]: #socketsetkeepaliveenable-initialdelay
[`socket.setTimeout()`]: #socketsettimeouttimeout-callback
[`socket.setTimeout(timeout)`]: #socketsettimeouttimeout-callback
[`stream.getDefaultHighWaterMark()`]: stream.md#streamgetdefaulthighwatermarkobjectmode
[`writable.destroy()`]: stream.md#writabledestroyerror
[`writable.destroyed`]: stream.md#writabledestroyed
[`writable.end()`]: stream.md#writableendchunk-encoding-callback
[`writable.writableLength`]: stream.md#writablewritablelength
[点分十进制表示法]: https://en.wikipedia.org/wiki/Dot-decimal_notation
[half-closed]: https://tools.ietf.org/html/rfc1122
[stream_writable_write]: stream.md#writablewritechunk-encoding-callback
[未指定的 IPv4 地址]: https://en.wikipedia.org/wiki/0.0.0.0
[未指定的 IPv6 地址]: https://en.wikipedia.org/wiki/IPv6_address#Unspecified_address