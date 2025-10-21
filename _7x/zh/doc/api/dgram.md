# UDP/datagram sockets

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- name=dgram -->

<!-- source_link=lib/dgram.js -->

`node:dgram` 模块提供了 UDP 数据报套接字的实现。

```mjs
import dgram from 'node:dgram';

const server = dgram.createSocket('udp4');

server.on('error', (err) => {
  console.error(`server error:\n${err.stack}`);
  server.close();
});

server.on('message', (msg, rinfo) => {
  console.log(`server got: ${msg} from ${rinfo.address}:${rinfo.port}`);
});

server.on('listening', () => {
  const address = server.address();
  console.log(`server listening ${address.address}:${address.port}`);
});

server.bind(41234);
// Prints: server listening 0.0.0.0:41234
```

```cjs
const dgram = require('node:dgram');
const server = dgram.createSocket('udp4');

server.on('error', (err) => {
  console.error(`server error:\n${err.stack}`);
  server.close();
});

server.on('message', (msg, rinfo) => {
  console.log(`server got: ${msg} from ${rinfo.address}:${rinfo.port}`);
});

server.on('listening', () => {
  const address = server.address();
  console.log(`server listening ${address.address}:${address.port}`);
});

server.bind(41234);
// Prints: server listening 0.0.0.0:41234
```

## 类：`dgram.Socket`

<!-- YAML
added: v0.1.99
-->

* 继承：{EventEmitter}

封装了数据报功能。

使用 [`dgram.createSocket()`][] 创建 `dgram.Socket` 的新实例。不应使用 `new` 关键字来创建 `dgram.Socket` 实例。

### 事件：`'close'`

<!-- YAML
added: v0.1.99
-->

在使用 [`close()`][] 关闭套接字后，会发出 `'close'` 事件。一旦触发，此套接字上不会再发出新的 `'message'` 事件。

### 事件：`'connect'`

<!-- YAML
added: v12.0.0
-->

当套接字因成功调用 [`connect()`][] 而与远程地址关联后，会发出 `'connect'` 事件。

### 事件：`'error'`

<!-- YAML
added: v0.1.99
-->

* `exception` {Error}

每当发生任何错误时，都会发出 `'error'` 事件。事件处理函数会传入一个 `Error` 对象。

### 事件：`'listening'`

<!-- YAML
added: v0.1.99
-->

一旦 `dgram.Socket` 可寻址并能接收数据，就会发出 `'listening'` 事件。这可以通过显式调用 `socket.bind()` 或隐式地在首次使用 `socket.send()` 发送数据时发生。在 `dgram.Socket` 开始监听之前，底层系统资源不存在，并且调用诸如 `socket.address()` 和 `socket.setTTL()` 等方法将会失败。

### 事件：`'message'`

<!-- YAML
added: v0.1.99
changes:
  - version: v18.4.0
    pr-url: https://github.com/nodejs/node/pull/43054
    description: The `family` property now returns a string instead of a number.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41431
    description: The `family` property now returns a number instead of a string.
-->

当套接字上有新的数据报可用时，会发出 `'message'` 事件。事件处理函数会传入两个参数：`msg` 和 `rinfo`。

* `msg` {Buffer} 消息。
* `rinfo` {Object} 远程地址信息。
  * `address` {string} 发送方地址。
  * `family` {string} 地址族（`'IPv4'` 或 `'IPv6'`）。
  * `port` {number} 发送方端口。
  * `size` {number} 消息大小。

如果传入数据包的源地址是 IPv6 链路本地地址，则接口名称会添加到 `address` 中。例如，在 `en0` 接口上接收的数据包可能将地址字段设置为 `'fe80::2618:1234:ab11:3b9c%en0'`，其中 `'%en0'` 是作为区域 ID 后缀的接口名称。

### `socket.addMembership(multicastAddress[, multicastInterface])`

<!-- YAML
added: v0.6.9
-->

* `multicastAddress` {string}
* `multicastInterface` {string}

告诉内核使用 `IP_ADD_MEMBERSHIP` 套接字选项在给定的 `multicastAddress` 和 `multicastInterface` 上加入多播组。如果未指定 `multicastInterface` 参数，操作系统将选择一个接口并加入其成员资格。要加入每个可用接口的成员资格，请多次调用 `addMembership`，每个接口一次。

在未绑定的套接字上调用此方法时，该方法将隐式绑定到随机端口，监听所有接口。

在多个 `cluster` 工作进程之间共享 UDP 套接字时，必须仅调用一次 `socket.addMembership()` 函数，否则将发生 `EADDRINUSE` 错误：

```mjs
import cluster from 'node:cluster';
import dgram from 'node:dgram';

if (cluster.isPrimary) {
  cluster.fork(); // 工作正常。
  cluster.fork(); // 因 EADDRINUSE 失败。
} else {
  const s = dgram.createSocket('udp4');
  s.bind(1234, () => {
    s.addMembership('224.0.0.114');
  });
}
```

```cjs
const cluster = require('node:cluster');
const dgram = require('node:dgram');

if (cluster.isPrimary) {
  cluster.fork(); // 工作正常。
  cluster.fork(); // 因 EADDRINUSE 失败。
} else {
  const s = dgram.createSocket('udp4');
  s.bind(1234, () => {
    s.addMembership('224.0.0.114');
  });
}
```

### `socket.addSourceSpecificMembership(sourceAddress, groupAddress[, multicastInterface])`

<!-- YAML
added:
 - v13.1.0
 - v12.16.0
-->

* `sourceAddress` {string}
* `groupAddress` {string}
* `multicastInterface` {string}

告诉内核使用 `IP_ADD_SOURCE_MEMBERSHIP` 套接字选项，在给定的 `sourceAddress` 和 `groupAddress` 上加入特定源多播通道，并使用 `multicastInterface`。如果未指定 `multicastInterface` 参数，操作系统将选择一个接口并加入其成员资格。要加入每个可用接口的成员资格，请多次调用 `socket.addSourceSpecificMembership()`，每个接口一次。

在未绑定的套接字上调用此方法时，该方法将隐式绑定到随机端口，监听所有接口。

### `socket.address()`

<!-- YAML
added: v0.1.99
-->

* 返回：{Object}

返回一个包含套接字地址信息的对象。对于 UDP 套接字，此对象将包含 `address`、`family` 和 `port` 属性。

如果在未绑定的套接字上调用此方法，则会抛出 `EBADF`。

### `socket.bind([port][, address][, callback])`

<!-- YAML
added: v0.1.99
changes:
  - version: v0.9.1
    commit: 332fea5ac1816e498030109c4211bca24a7fa667
    description: The method was changed to an asynchronous execution model.
                 Legacy code would need to be changed to pass a callback
                 function to the method call.
-->

* `port` {integer}
* `address` {string}
* `callback` {Function} 无参数。绑定完成时调用。

对于 UDP 套接字，使 `dgram.Socket` 在指定的 `port` 和可选的 `address` 上监听数据报消息。如果未指定 `port` 或为 `0`，操作系统将尝试绑定到随机端口。如果未指定 `address`，操作系统将尝试在所有地址上监听。一旦绑定完成，将发出 `'listening'` 事件，并调用可选的 `callback` 函数。

同时指定 `'listening'` 事件监听器并将 `callback` 传递给 `socket.bind()` 方法并无害处，但不是很实用。

绑定的数据报套接字会保持 Node.js 进程运行以接收数据报消息。

如果绑定失败，会生成 `'error'` 事件。在极少数情况下（例如尝试使用已关闭的套接字进行绑定），可能会抛出 [`Error`][]。

监听端口 41234 的 UDP 服务器示例：

```mjs
import dgram from 'node:dgram';

const server = dgram.createSocket('udp4');

server.on('error', (err) => {
  console.error(`server error:\n${err.stack}`);
  server.close();
});

server.on('message', (msg, rinfo) => {
  console.log(`server got: ${msg} from ${rinfo.address}:${rinfo.port}`);
});

server.on('listening', () => {
  const address = server.address();
  console.log(`server listening ${address.address}:${address.port}`);
});

server.bind(41234);
// Prints: server listening 0.0.0.0:41234
```

```cjs
const dgram = require('node:dgram');
const server = dgram.createSocket('udp4');

server.on('error', (err) => {
  console.error(`server error:\n${err.stack}`);
  server.close();
});

server.on('message', (msg, rinfo) => {
  console.log(`server got: ${msg} from ${rinfo.address}:${rinfo.port}`);
});

server.on('listening', () => {
  const address = server.address();
  console.log(`server listening ${address.address}:${address.port}`);
});

server.bind(41234);
// Prints: server listening 0.0.0.0:41234
```

### `socket.bind(options[, callback])`

<!-- YAML
added: v0.11.14
-->

* `options` {Object} 必需。支持以下属性：
  * `port` {integer}
  * `address` {string}
  * `exclusive` {boolean}
  * `fd` {integer}
* `callback` {Function}

对于 UDP 套接字，使 `dgram.Socket` 在作为第一个参数传递的 `options` 对象的属性中指定的命名 `port` 和可选 `address` 上监听数据报消息。如果未指定 `port` 或为 `0`，操作系统将尝试绑定到随机端口。如果未指定 `address`，操作系统将尝试在所有地址上监听。一旦绑定完成，将发出 `'listening'` 事件，并调用可选的 `callback` 函数。

`options` 对象可能包含 `fd` 属性。当设置了大于 `0` 的 `fd` 时，它将包装具有给定文件描述符的现有套接字。在这种情况下，将忽略 `port` 和 `address` 的属性。

同时指定 `'listening'` 事件监听器并将 `callback` 传递给 `socket.bind()` 方法并无害处，但不是很实用。

`options` 对象可能包含一个额外的 `exclusive` 属性，该属性在与 [`cluster`][] 模块一起使用 `dgram.Socket` 对象时使用。当 `exclusive` 设置为 `false`（默认值）时，集群工作进程将使用相同的底层套接字句柄，允许共享连接处理职责。然而，当 `exclusive` 为 `true` 时，句柄不共享，尝试共享端口会导致错误。创建 `dgram.Socket` 时将 `reusePort` 选项设置为 `true` 会导致在调用 `socket.bind()` 时 `exclusive` 始终为 `true`。

绑定的数据报套接字会保持 Node.js 进程运行以接收数据报消息。

如果绑定失败，会生成 `'error'` 事件。在极少数情况下（例如尝试使用已关闭的套接字进行绑定），可能会抛出 [`Error`][]。

下面显示了一个监听独占端口的套接字示例。

```js
socket.bind({
  address: 'localhost',
  port: 8000,
  exclusive: true,
});
```

### `socket.close([callback])`

<!-- YAML
added: v0.1.99
-->

* `callback` {Function} 当套接字关闭时调用。

关闭底层套接字并停止在其上监听数据。如果提供了回调函数，它会被添加为 [`'close'`][] 事件的监听器。

### `socket[Symbol.asyncDispose]()`

<!-- YAML
added:
 - v20.5.0
 - v18.18.0
changes:
 - version: v24.2.0
   pr-url: https://github.com/nodejs/node/pull/58467
   description: No longer experimental.
-->

调用 [`socket.close()`][] 并返回一个在套接字关闭时完成的 promise。

### `socket.connect(port[, address][, callback])`

<!-- YAML
added: v12.0.0
-->

* `port` {integer}
* `address` {string}
* `callback` {Function} 当连接完成或出错时调用。

将 `dgram.Socket` 与远程地址和端口关联。此句柄发送的每条消息都会自动发送到该目的地。此外，套接字将仅接收来自该远程对等方的消息。尝试在已连接的套接字上调用 `connect()` 将导致 [`ERR_SOCKET_DGRAM_IS_CONNECTED`][] 异常。如果未提供 `address`，将默认使用 `'127.0.0.1'`（对于 `udp4` 套接字）或 `'::1'`（对于 `udp6` 套接字）。连接完成后，会发出 `'connect'` 事件，并调用可选的 `callback` 函数。如果失败，将调用 `callback`，或者，如果失败，将发出 `'error'` 事件。

### `socket.disconnect()`

<!-- YAML
added: v12.0.0
-->

一个同步函数，用于将已连接的 `dgram.Socket` 与其远程地址解除关联。尝试在未绑定或已断开连接的套接字上调用 `disconnect()` 将导致 [`ERR_SOCKET_DGRAM_NOT_CONNECTED`][] 异常。

### `socket.dropMembership(multicastAddress[, multicastInterface])`

<!-- YAML
added: v0.6.9
-->

* `multicastAddress` {string}
* `multicastInterface` {string}

指示内核使用 `IP_DROP_MEMBERSHIP` 套接字选项在 `multicastAddress` 上离开多播组。当套接字关闭或进程终止时，内核会自动调用此方法，因此大多数应用程序永远没有理由调用此方法。

如果未指定 `multicastInterface`，操作系统将尝试在所有有效接口上删除成员资格。

### `socket.dropSourceSpecificMembership(sourceAddress, groupAddress[, multicastInterface])`

<!-- YAML
added:
 - v13.1.0
 - v12.16.0
-->

* `sourceAddress` {string}
* `groupAddress` {string}
* `multicastInterface` {string}

指示内核使用 `IP_DROP_SOURCE_MEMBERSHIP` 套接字选项在给定的 `sourceAddress` 和 `groupAddress` 上离开特定源多播通道。当套接字关闭或进程终止时，内核会自动调用此方法，因此大多数应用程序永远没有理由调用此方法。

如果未指定 `multicastInterface`，操作系统将尝试在所有有效接口上删除成员资格。

### `socket.getRecvBufferSize()`

<!-- YAML
added: v8.7.0
-->

* 返回：{number} `SO_RCVBUF` 套接字接收缓冲区大小（以字节为单位）。

如果在未绑定的套接字上调用此方法，则会抛出 [`ERR_SOCKET_BUFFER_SIZE`][]。

### `socket.getSendBufferSize()`

<!-- YAML
added: v8.7.0
-->

* 返回：{number} `SO_SNDBUF` 套接字发送缓冲区大小（以字节为单位）。

如果在未绑定的套接字上调用此方法，则会抛出 [`ERR_SOCKET_BUFFER_SIZE`][]。

### `socket.getSendQueueSize()`

<!-- YAML
added:
  - v18.8.0
  - v16.19.0
-->

* 返回：{number} 排队等待发送的字节数。

### `socket.getSendQueueCount()`

<!-- YAML
added:
  - v18.8.0
  - v16.19.0
-->

* 返回：{number} 当前队列中等待处理的发送请求数。

### `socket.ref()`

<!-- YAML
added: v0.9.1
-->

* 返回：{dgram.Socket}

默认情况下，绑定套接字将导致 Node.js 进程在套接字打开时保持运行。`socket.unref()` 方法可用于将套接字从保持 Node.js 进程活动的引用计数中排除。`socket.ref()` 方法将套接字添加回引用计数并恢复默认行为。

多次调用 `socket.ref()` 不会有额外效果。

`socket.ref()` 方法返回对套接字的引用，因此可以链式调用。

### `socket.remoteAddress()`

<!-- YAML
added: v12.0.0
-->

* 返回：{Object}

返回一个包含远程端点的 `address`、`family` 和 `port` 的对象。如果套接字未连接，此方法将抛出 [`ERR_SOCKET_DGRAM_NOT_CONNECTED`][] 异常。

### `socket.send(msg[, offset, length][, port][, address][, callback])`

<!-- YAML
added: v0.1.99
changes:
  - version: v17.0.0
    pr-url: https://github.com/nodejs/node/pull/39190
    description: The `address` parameter now only accepts a `string`, `null`
                 or `undefined`.
  - version:
    - v14.5.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/22413
    description: The `msg` parameter can now be any `TypedArray` or `DataView`.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26871
    description: Added support for sending data on connected sockets.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/11985
    description: The `msg` parameter can be an `Uint8Array` now.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10473
    description: The `address` parameter is always optional now.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/5929
    description: On success, `callback` will now be called with an `error`
                 argument of `null` rather than `0`.
  - version: v5.7.0
    pr-url: https://github.com/nodejs/node/pull/4374
    description: The `msg` parameter can be an array now. Also, the `offset`
                 and `length` parameters are optional now.
-->

* `msg` {Buffer|TypedArray|DataView|string|Array} 要发送的消息。
* `offset` {integer} 缓冲区中消息开始的位置。
* `length` {integer} 消息中的字节数。
* `port` {integer} 目标端口。
* `address` {string} 目标主机名或 IP 地址。
* `callback` {Function} 当消息已发送时调用。

在套接字上广播数据报。对于无连接套接字，必须指定目标 `port` 和 `address`。相反，已连接的套接字将使用其关联的远程端点，因此不得设置 `port` 和 `address` 参数。

`msg` 参数包含要发送的消息。根据其类型，可能适用不同的行为。如果 `msg` 是 `Buffer`、任何 `TypedArray` 或 `DataView`，则 `offset` 和 `length` 分别指定 `Buffer` 中消息开始的偏移量和消息中的字节数。如果 `msg` 是 `String`，则它会自动转换为使用 `'utf8'` 编码的 `Buffer`。对于包含多字节字符的消息，`offset` 和 `length` 将根据[字节长度][byte length]计算，而不是字符位置。如果 `msg` 是数组，则不得指定 `offset` 和 `length`。

`address` 参数是一个字符串。如果 `address` 的值是主机名，则将使用 DNS 来解析主机的地址。如果未提供 `address` 或为 nullish，将默认使用 `'127.0.0.1'`（对于 `udp4` 套接字）或 `'::1'`（对于 `udp6` 套接字）。

如果套接字之前未通过调用 `bind` 进行绑定，则将为套接字分配一个随机端口号，并绑定到“所有接口”地址（对于 `udp4` 套接字为 `'0.0.0.0'`，对于 `udp6` 套接字为 `'::0'`）。

可以指定可选的 `callback` 函数作为报告 DNS 错误或确定何时可以安全重用 `buf` 对象的一种方式。DNS 查找会延迟发送时间至少一个 Node.js 事件循环滴答。

确定数据报已发送的唯一方法是使用 `callback`。如果发生错误并且提供了 `callback`，则错误将作为第一个参数传递给 `callback`。如果未提供 `callback`，则错误将作为 `'error'` 事件在 `socket` 对象上发出。

偏移量和长度是可选的，但如果使用了其中一个，则两者_必须_都设置。仅当第一个参数是 `Buffer`、`TypedArray` 或 `DataView` 时才支持它们。

如果在未绑定的套接字上调用此方法，则会抛出 [`ERR_SOCKET_BAD_PORT`][]。

向 `localhost` 上的端口发送 UDP 数据包的示例；

```mjs
import dgram from 'node:dgram';
import { Buffer } from 'node:buffer';

const message = Buffer.from('Some bytes');
const client = dgram.createSocket('udp4');
client.send(message, 41234, 'localhost', (err) => {
  client.close();
});
```

```cjs
const dgram = require('node:dgram');
const { Buffer } = require('node:buffer');

const message = Buffer.from('Some bytes');
const client = dgram.createSocket('udp4');
client.send(message, 41234, 'localhost', (err) => {
  client.close();
});
```

向 `127.0.0.1` 上的端口发送由多个缓冲区组成的 UDP 数据包的示例；

```mjs
import dgram from 'node:dgram';
import { Buffer } from 'node:buffer';

const buf1 = Buffer.from('Some ');
const buf2 = Buffer.from('bytes');
const client = dgram.createSocket('udp4');
client.send([buf1, buf2], 41234, (err) => {
  client.close();
});
```

```cjs
const dgram = require('node:dgram');
const { Buffer } = require('node:buffer');

const buf1 = Buffer.from('Some ');
const buf2 = Buffer.from('bytes');
const client = dgram.createSocket('udp4');
client.send([buf1, buf2], 41234, (err) => {
  client.close();
});
```

发送多个缓冲区可能会更快或更慢，具体取决于应用程序和操作系统。运行基准测试以根据具体情况确定最佳策略。然而，一般来说，发送多个缓冲区更快。

使用连接到 `localhost` 上端口的套接字发送 UDP 数据包的示例：

```mjs
import dgram from 'node:dgram';
import { Buffer } from 'node:buffer';

const message = Buffer.from('Some bytes');
const client = dgram.createSocket('udp4');
client.connect(41234, 'localhost', (err) => {
  client.send(message, (err) => {
    client.close();
  });
});
```

```cjs
const dgram = require('node:dgram');
const { Buffer } = require('node:buffer');

const message = Buffer.from('Some bytes');
const client = dgram.createSocket('udp4');
client.connect(41234, 'localhost', (err) => {
  client.send(message, (err) => {
    client.close();
  });
});
```

#### 关于 UDP 数据报大小的说明

IPv4/v6 数据报的最大大小取决于 `MTU`（最大传输单元）和 `Payload Length` 字段大小。

* `Payload Length` 字段为 16 位宽，这意味着正常有效载荷不能超过 64K 八位字节，包括互联网头和数据（65,507 字节 = 65,535 − 8 字节 UDP 头 − 20 字节 IP 头）；这对于环回接口通常成立，但如此长的数据报消息对于大多数主机和网络来说不切实际。

* `MTU` 是给定链路层技术可以支持的数据报消息的最大大小。对于任何链路，IPv4 强制要求最小 `MTU` 为 68 个八位字节，而 IPv4 的推荐 `MTU` 为 576（通常推荐作为拨号类型应用程序的 `MTU`），无论它们是完整到达还是分段到达。

  对于 IPv6，最小 `MTU` 为 1280 个八位字节。但是，强制的最小分段重组缓冲区大小为 1500 个八位字节。68 个八位字节的值非常小，因为大多数当前的链路层技术（如以太网）的最小 `MTU` 为 1500。

无法预先知道数据包可能经过的每个链路的 MTU。发送大于接收方 `MTU` 的数据报将不起作用，因为数据包将被静默丢弃，而不会通知源数据未到达其预期的接收者。

### `socket.setBroadcast(flag)`

<!-- YAML
added: v0.6.9
-->

* `flag` {boolean}

设置或清除 `SO_BROADCAST` 套接字选项。当设置为 `true` 时，UDP 数据包可以发送到本地接口的广播地址。

如果在未绑定的套接字上调用此方法，则会抛出 `EBADF`。

### `socket.setMulticastInterface(multicastInterface)`

<!-- YAML
added: v8.6.0
-->

* `multicastInterface` {string}

_本节中所有对范围的引用都是指 [IPv6 区域索引][]，这些索引由 [RFC 4007][] 定义。在字符串形式中，带有范围索引的 IP 写为 `'IP%scope'`，其中 scope 是接口名称或接口编号。_

将套接字的默认传出多播接口设置为选择的接口或恢复为系统接口选择。`multicastInterface` 必须是套接字族 IP 的有效字符串表示形式。

对于 IPv4 套接字，这应该是为所需物理接口配置的 IP。发送到套接字上多播的所有数据包将在最近一次成功使用此调用确定的接口上发送。

对于 IPv6 套接字，`multicastInterface` 应包含一个范围以指示接口，如下例所示。在 IPv6 中，各个 `send` 调用也可以在地址中使用显式范围，因此只有发送到未指定显式范围的多播地址的数据包才会受到最近一次成功使用此调用的影响。

如果在未绑定的套接字上调用此方法，则会抛出 `EBADF`。

#### 示例：IPv6 传出多播接口

在大多数系统上，范围格式使用接口名称：

```js
const socket = dgram.createSocket('udp6');

socket.bind(1234, () => {
  socket.setMulticastInterface('::%eth1');
});
```

在 Windows 上，范围格式使用接口编号：

```js
const socket = dgram.createSocket('udp6');

socket.bind(1234, () => {
  socket.setMulticastInterface('::%2');
});
```

#### 示例：IPv4 传出多播接口

所有系统都使用所需物理接口上的主机 IP：

```js
const socket = dgram.createSocket('udp4');

socket.bind(1234, () => {
  socket.setMulticastInterface('10.0.0.2');
});
```

#### 调用结果

在未准备好发送或不再打开的套接字上调用可能会抛出 _Not running_ [`Error`][]。

如果 `multicastInterface` 无法解析为 IP，则会抛出 _EINVAL_ [`System Error`][]。

在 IPv4 上，如果 `multicastInterface` 是有效地址但与任何接口不匹配，或者地址与族不匹配，则会抛出 [`System Error`][]，例如 `EADDRNOTAVAIL` 或 `EPROTONOSUP`。

在 IPv6 上，指定或省略范围的大多数错误将导致套接字继续使用（或恢复为）系统的默认接口选择。

套接字地址族的 ANY 地址（IPv4 `'0.0.0.0'` 或 IPv6 `'::'`）可用于将套接字的默认传出接口的控制权返回给系统，以供将来的多播数据包使用。

### `socket.setMulticastLoopback(flag)`

<!-- YAML
added: v0.3.8
-->

* `flag` {boolean}

设置或清除 `IP_MULTICAST_LOOP` 套接字选项。当设置为 `true` 时，多播数据包也将在本地接口上接收。

如果在未绑定的套接字上调用此方法，则会抛出 `EBADF`。

### `socket.setMulticastTTL(ttl)`

<!-- YAML
added: v0.3.8
-->

* `ttl` {integer}

设置 `IP_MULTICAST_TTL` 套接字选项。虽然 TTL 通常代表“生存时间”，但在此上下文中，它指定数据包允许经过的 IP 跳数，特别是多播流量。转发数据包的每个路由器或网关都会递减 TTL。如果路由器将 TTL 递减到 0，则不会转发。

`ttl` 参数可能在 0 到 255 之间。大多数系统上的默认值为 `1`。

如果在未绑定的套接字上调用此方法，则会抛出 `EBADF`。

### `socket.setRecvBufferSize(size)`

<!-- YAML
added: v8.7.0
-->

* `size` {integer}

设置 `SO_RCVBUF` 套接字选项。设置最大套接字接收缓冲区（以字节为单位）。

如果在未绑定的套接字上调用此方法，则会抛出 [`ERR_SOCKET_BUFFER_SIZE`][]。

### `socket.setSendBufferSize(size)`

<!-- YAML
added: v8.7.0
-->

* `size` {integer}

设置 `SO_SNDBUF` 套接字选项。设置最大套接字发送缓冲区（以字节为单位）。

如果在未绑定的套接字上调用此方法，则会抛出 [`ERR_SOCKET_BUFFER_SIZE`][]。

### `socket.setTTL(ttl)`

<!-- YAML
added: v0.1.101
-->

* `ttl` {integer}

设置 `IP_TTL` 套接字选项。虽然 TTL 通常代表“生存时间”，但在此上下文中，它指定数据包允许经过的 IP 跳数。转发数据包的每个路由器或网关都会递减 TTL。如果路由器将 TTL 递减到 0，则不会转发。更改 TTL 值通常用于网络探测或多播。

`ttl` 参数可能在 1 到 255 之间。大多数系统上的默认值为 64。

如果在未绑定的套接字上调用此方法，则会抛出 `EBADF`。

### `socket.unref()`

<!-- YAML
added: v0.9.1
-->

* 返回：{dgram.Socket}

默认情况下，绑定套接字将导致 Node.js 进程在套接字打开时保持运行。`socket.unref()` 方法可用于将套接字从保持 Node.js 进程活动的引用计数中排除，允许进程退出，即使套接字仍在监听。

多次调用 `socket.unref()` 不会有额外效果。

`socket.unref()` 方法返回对套接字的引用，因此可以链式调用。

## `node:dgram` 模块函数

### `dgram.createSocket(options[, callback])`

<!-- YAML
added: v0.11.13
changes:
  - version:
    - v23.1.0
    - v22.12.0
    pr-url: https://github.com/nodejs/node/pull/55403
    description: The `reusePort` option is supported.
  - version: v15.8.0
    pr-url: https://github.com/nodejs/node/pull/37026
    description: AbortSignal support was added.
  - version: v11.4.0
    pr-url: https://github.com/nodejs/node/pull/23798
    description: The `ipv6Only` option is supported.
  - version: v8.7.0
    pr-url: https://github.com/nodejs/node/pull/13623
    description: The `recvBufferSize` and `sendBufferSize` options are
                 supported now.
  - version: v8.6.0
    pr-url: https://github.com/nodejs/node/pull/14560
    description: The `lookup` option is supported.
-->

* `options` {Object} 可用选项有：
  * `type` {string} 套接字的族。必须是 `'udp4'` 或 `'udp6'`。必需。
  * `reuseAddr` {boolean} 当为 `true` 时，[`socket.bind()`][] 将重用地址，即使另一个进程已经在其上绑定了套接字，但只有一个套接字可以接收数据。**默认值：** `false`。
  * `reusePort` {boolean} 当为 `true` 时，[`socket.bind()`][] 将重用端口，即使另一个进程已经在其上绑定了套接字。传入的数据报会分发给监听的套接字。此选项仅在部分平台上可用，例如 Linux 3.9+、DragonFlyBSD 3.6+、FreeBSD 12.0+、Solaris 11.4 和 AIX 7.2.5+。在不支持的平台上，此选项在绑定套接字时会引发错误。**默认值：** `false`。
  * `ipv6Only` {boolean} 将 `ipv6Only` 设置为 `true` 将禁用双栈支持，即绑定到地址 `::` 不会绑定 `0.0.0.0`。**默认值：** `false`。
  * `recvBufferSize` {number} 设置 `SO_RCVBUF` 套接字值。
  * `sendBufferSize` {number} 设置 `SO_SNDBUF` 套接字值。
  * `lookup` {Function} 自定义查找函数。**默认值：** [`dns.lookup()`][]。
  * `signal` {AbortSignal} 可用于关闭套接字的 AbortSignal。
  * `receiveBlockList` {net.BlockList} `receiveBlockList` 可用于丢弃到特定 IP 地址、IP 范围或 IP 子网的入站数据报。如果服务器在反向代理、NAT 等后面，则此方法不起作用，因为检查阻止列表的地址是代理的地址或 NAT 指定的地址。
  * `sendBlockList` {net.BlockList} `sendBlockList` 可用于禁用对特定 IP 地址、IP 范围或 IP 子网的出站访问。
* `callback` {Function} 作为 `'message'` 事件的监听器附加。可选。
* 返回：{dgram.Socket}

创建一个 `dgram.Socket` 对象。一旦创建了套接字，调用 [`socket.bind()`][] 将指示套接字开始监听数据报消息。当 `address` 和 `port` 未传递给 [`socket.bind()`][] 时，该方法会将套接字绑定到随机端口上的“所有接口”地址（它对 `udp4` 和 `udp6` 套接字都执行正确的操作）。绑定的地址和端口可以使用 [`socket.address().address`][] 和 [`socket.address().port`][] 检索。

如果启用了 `signal` 选项，在相应的 `AbortController` 上调用 `.abort()` 类似于在套接字上调用 `.close()`：

```js
const controller = new AbortController();
const { signal } = controller;
const server = dgram.createSocket({ type: 'udp4', signal });
server.on('message', (msg, rinfo) => {
  console.log(`server got: ${msg} from ${rinfo.address}:${rinfo.port}`);
});
// 稍后，当您想关闭服务器时。
controller.abort();
```

### `dgram.createSocket(type[, callback])`

<!-- YAML
added: v0.1.99
-->

* `type` {string} `'udp4'` 或 `'udp6'`。
* `callback` {Function} 作为 `'message'` 事件的监听器附加。
* 返回：{dgram.Socket}

创建指定 `type` 的 `dgram.Socket` 对象。

一旦创建了套接字，调用 [`socket.bind()`][] 将指示套接字开始监听数据报消息。当 `address` 和 `port` 未传递给 [`socket.bind()`][] 时，该方法会将套接字绑定到随机端口上的“所有接口”地址（它对 `udp4` 和 `udp6` 套接字都执行正确的操作）。绑定的地址和端口可以使用 [`socket.address().address`][] 和 [`socket.address().port`][] 检索。

[IPv6 区域索引]: https://en.wikipedia.org/wiki/IPv6_address#Scoped_literal_IPv6_addresses
[RFC 4007]: https://tools.ietf.org/html/rfc4007
[`'close'`]: #event-close
[`ERR_SOCKET_BAD_PORT`]: errors.md#err_socket_bad_port
[`ERR_SOCKET_BUFFER_SIZE`]: errors.md#err_socket_buffer_size
[`ERR_SOCKET_DGRAM_IS_CONNECTED`]: errors.md#err_socket_dgram_is_connected
[`ERR_SOCKET_DGRAM_NOT_CONNECTED`]: errors.md#err_socket_dgram_not_connected
[`Error`]: errors.md#class-error
[`System Error`]: errors.md#class-systemerror
[`close()`]: #socketclosecallback
[`cluster`]: cluster.md
[`connect()`]: #socketconnectport-address-callback
[`dgram.createSocket()`]: #dgramcreatesocketoptions-callback
[`dns.lookup()`]: dns.md#dnslookuphostname-options-callback
[`socket.address().address`]: #socketaddress
[`socket.address().port`]: #socketaddress
[`socket.bind()`]: #socketbindport-address-callback
[`socket.close()`]: #socketclosecallback
[字节长度]: buffer.md#static-method-bufferbytelengthstring-encoding