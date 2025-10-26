# Inspector

<!--introduced_in=v8.0.0-->

> Stability: 2 - Stable

<!-- source_link=lib/inspector.js -->

`node:inspector` 模块提供了与 V8 检查器交互的 API。

可以通过以下方式访问：

```mjs
import * as inspector from 'node:inspector/promises';
```

```cjs
const inspector = require('node:inspector/promises');
```

或

```mjs
import * as inspector from 'node:inspector';
```

```cjs
const inspector = require('node:inspector');
```

## Promises API

<!-- YAML
added: v19.0.0
-->

> Stability: 1 - Experimental

### 类：`inspector.Session`

* 继承自：{EventEmitter}

`inspector.Session` 用于向 V8 检查器后端分发消息并接收消息响应和通知。

#### `new inspector.Session()`

<!-- YAML
added: v8.0.0
-->

创建一个新的 `inspector.Session` 类实例。检查器会话需要通过 [`session.connect()`][] 连接后才能将消息分派到检查器后端。

当使用 `Session` 时，控制台 API 输出的对象不会被释放，除非我们手动执行了 `Runtime.DiscardConsoleEntries` 命令。

#### 事件：`'inspectorNotification'`

<!-- YAML
added: v8.0.0
-->

* 类型：{Object} 通知消息对象

当接收到来自 V8 检查器的任何通知时触发。

```js
session.on('inspectorNotification', (message) => console.log(message.method));
// Debugger.paused
// Debugger.resumed
```

> **注意** 不推荐使用同线程会话设置断点，请参阅 [断点支持][support of breakpoints]。

也可以只订阅特定方法的通知：

#### 事件：`<inspector-protocol-method>`

<!-- YAML
added: v8.0.0
-->

* 类型：{Object} 通知消息对象

当接收到一个检查器通知，并且其 method 字段设置为 `<inspector-protocol-method>` 值时触发。

以下代码片段在 [`'Debugger.paused'`][] 事件上安装了一个监听器，每当程序执行被暂停时（例如通过断点），就会打印程序暂停的原因：

```js
session.on('Debugger.paused', ({ params }) => {
  console.log(params.hitBreakpoints);
});
// [ '/the/file/that/has/the/breakpoint.js:11:0' ]
```

> **注意** 不推荐使用同线程会话设置断点，请参阅 [断点支持][support of breakpoints]。

#### `session.connect()`

<!-- YAML
added: v8.0.0
-->

将会话连接到检查器后端。

#### `session.connectToMainThread()`

<!-- YAML
added: v12.11.0
-->

将会话连接到主线程检查器后端。如果此 API 未在 Worker 线程上调用，将抛出异常。

#### `session.disconnect()`

<!-- YAML
added: v8.0.0
-->

立即关闭会话。所有挂起的消息回调都将被调用并带有错误。需要再次调用 [`session.connect()`][] 才能重新发送消息。重新连接的会话将丢失所有检查器状态，例如已启用的代理或配置的断点。

#### `session.post(method[, params])`

<!-- YAML
added: v19.0.0
-->

* `method` {string}
* `params` {Object}
* 返回：{Promise}

向检查器后端发送消息。

```mjs
import { Session } from 'node:inspector/promises';
try {
  const session = new Session();
  session.connect();
  const result = await session.post('Runtime.evaluate', { expression: '2 + 2' });
  console.log(result);
} catch (error) {
  console.error(error);
}
// 输出：{ result: { type: 'number', value: 4, description: '4' } }
```

V8 检查器协议的最新版本发布在 [Chrome DevTools Protocol Viewer][] 上。

Node.js 检查器支持 V8 声明的所有 Chrome DevTools Protocol 域。Chrome DevTools Protocol 域提供了一个接口，用于与用于检查应用程序状态和监听运行时事件的运行时代理之一进行交互。

#### 用法示例

除了调试器之外，各种 V8 分析器也可以通过 DevTools 协议使用。

##### CPU 分析器

以下示例展示如何使用 [CPU Profiler][]：

```mjs
import { Session } from 'node:inspector/promises';
import fs from 'node:fs';
const session = new Session();
session.connect();

await session.post('Profiler.enable');
await session.post('Profiler.start');
// 在此处调用需要测量的业务逻辑...

// 一段时间后...
const { profile } = await session.post('Profiler.stop');

// 将分析文件写入磁盘、上传等。
fs.writeFileSync('./profile.cpuprofile', JSON.stringify(profile));
```

##### 堆分析器

以下示例展示如何使用 [Heap Profiler][]：

```mjs
import { Session } from 'node:inspector/promises';
import fs from 'node:fs';
const session = new Session();

const fd = fs.openSync('profile.heapsnapshot', 'w');

session.connect();

session.on('HeapProfiler.addHeapSnapshotChunk', (m) => {
  fs.writeSync(fd, m.params.chunk);
});

const result = await session.post('HeapProfiler.takeHeapSnapshot', null);
console.log('HeapProfiler.takeHeapSnapshot done:', result);
session.disconnect();
fs.closeSync(fd);
```

## Callback API

### 类：`inspector.Session`

* 继承自：{EventEmitter}

`inspector.Session` 用于向 V8 检查器后端分发消息并接收消息响应和通知。

#### `new inspector.Session()`

<!-- YAML
added: v8.0.0
-->

创建一个新的 `inspector.Session` 类实例。检查器会话需要通过 [`session.connect()`][] 连接后才能将消息分派到检查器后端。

当使用 `Session` 时，控制台 API 输出的对象不会被释放，除非我们手动执行了 `Runtime.DiscardConsoleEntries` 命令。

#### 事件：`'inspectorNotification'`

<!-- YAML
added: v8.0.0
-->

* 类型：{Object} 通知消息对象

当接收到来自 V8 检查器的任何通知时触发。

```js
session.on('inspectorNotification', (message) => console.log(message.method));
// Debugger.paused
// Debugger.resumed
```

> **注意** 不推荐使用同线程会话设置断点，请参阅 [断点支持][support of breakpoints]。

也可以只订阅特定方法的通知：

#### 事件：`<inspector-protocol-method>`;

<!-- YAML
added: v8.0.0
-->

* 类型：{Object} 通知消息对象

当接收到一个检查器通知，并且其 method 字段设置为 `<inspector-protocol-method>` 值时触发。

以下代码片段在 [`'Debugger.paused'`][] 事件上安装了一个监听器，每当程序执行被暂停时（例如通过断点），就会打印程序暂停的原因：

```js
session.on('Debugger.paused', ({ params }) => {
  console.log(params.hitBreakpoints);
});
// [ '/the/file/that/has/the/breakpoint.js:11:0' ]
```

> **注意** 不推荐使用同线程会话设置断点，请参阅 [断点支持][support of breakpoints]。

#### `session.connect()`

<!-- YAML
added: v8.0.0
-->

将会话连接到检查器后端。

#### `session.connectToMainThread()`

<!-- YAML
added: v12.11.0
-->

将会话连接到主线程检查器后端。如果此 API 未在 Worker 线程上调用，将抛出异常。

#### `session.disconnect()`

<!-- YAML
added: v8.0.0
-->

立即关闭会话。所有挂起的消息回调都将被调用并带有错误。需要再次调用 [`session.connect()`][] 才能重新发送消息。重新连接的会话将丢失所有检查器状态，例如已启用的代理或配置的断点。

#### `session.post(method[, params][, callback])`

<!-- YAML
added: v8.0.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `method` {string}
* `params` {Object}
* `callback` {Function}

向检查器后端发送消息。当收到响应时，将通知 `callback`。`callback` 是一个接受两个可选参数的函数：错误和消息特定结果。

```js
session.post('Runtime.evaluate', { expression: '2 + 2' },
             (error, { result }) => console.log(result));
// 输出：{ type: 'number', value: 4, description: '4' }
```

V8 检查器协议的最新版本发布在 [Chrome DevTools Protocol Viewer][] 上。

Node.js 检查器支持 V8 声明的所有 Chrome DevTools Protocol 域。Chrome DevTools Protocol 域提供了一个接口，用于与用于检查应用程序状态和监听运行时事件的运行时代理之一进行交互。

向 V8 发送 `HeapProfiler.takeHeapSnapshot` 或 `HeapProfiler.stopTrackingHeapObjects` 命令时，不能将 `reportProgress` 设置为 `true`。

#### 用法示例

除了调试器之外，各种 V8 分析器也可以通过 DevTools 协议使用。

##### CPU 分析器

以下示例展示如何使用 [CPU Profiler][]：

```js
const inspector = require('node:inspector');
const fs = require('node:fs');
const session = new inspector.Session();
session.connect();

session.post('Profiler.enable', () => {
  session.post('Profiler.start', () => {
    // 在此处调用需要测量的业务逻辑...

    // 一段时间后...
    session.post('Profiler.stop', (err, { profile }) => {
      // 将分析文件写入磁盘、上传等。
      if (!err) {
        fs.writeFileSync('./profile.cpuprofile', JSON.stringify(profile));
      }
    });
  });
});
```

##### 堆分析器

以下示例展示如何使用 [Heap Profiler][]：

```js
const inspector = require('node:inspector');
const fs = require('node:fs');
const session = new inspector.Session();

const fd = fs.openSync('profile.heapsnapshot', 'w');

session.connect();

session.on('HeapProfiler.addHeapSnapshotChunk', (m) => {
  fs.writeSync(fd, m.params.chunk);
});

session.post('HeapProfiler.takeHeapSnapshot', null, (err, r) => {
  console.log('HeapProfiler.takeHeapSnapshot done:', err, r);
  session.disconnect();
  fs.closeSync(fd);
});
```

## 常用对象

### `inspector.close()`

<!-- YAML
added: v9.0.0
changes:
  - version: v18.10.0
    pr-url: https://github.com/nodejs/node/pull/44489
    description: The API is exposed in the worker threads.
-->

尝试关闭所有剩余的连接，阻塞事件循环直到所有连接都关闭。一旦所有连接关闭，停用检查器。

### `inspector.console`

* 类型：{Object} 用于向远程检查器控制台发送消息的对象。

```js
require('node:inspector').console.log('a message');
```

检查器控制台与 Node.js 控制台没有 API 对等功能。

### `inspector.open([port[, host[, wait]]])`

<!-- YAML
changes:
  - version: v20.6.0
    pr-url: https://github.com/nodejs/node/pull/48765
    description: inspector.open() now returns a `Disposable` object.
-->

* `port` {number} 监听检查器连接的端口。可选。**默认值:** CLI 上指定的值。
* `host` {string} 监听检查器连接的主机。可选。**默认值:** CLI 上指定的值。
* `wait` {boolean} 阻塞直到客户端连接。可选。**默认值:** `false`。
* 返回：{Disposable} 一个调用 [`inspector.close()`][] 的 Disposable 对象。

在主机和端口上激活检查器。等同于 `node --inspect=[[host:]port]`，但可以在 node 启动后通过编程方式完成。

如果 wait 是 `true`，将阻塞直到客户端连接到检查端口并且流控制已传递给调试器客户端。

有关 `host` 参数使用的 [安全警告][security warning]，请参阅相关说明。

### `inspector.url()`

* 返回：{string|undefined}

返回活动检查器的 URL，如果没有则返回 `undefined`。

```console
$ node --inspect -p 'inspector.url()'
Debugger listening on ws://127.0.0.1:9229/166e272e-7a30-4d09-97ce-f1c012b43c34
For help, see: https://nodejs.org/en/docs/inspector
ws://127.0.0.1:9229/166e272e-7a30-4d09-97ce-f1c012b43c34

$ node --inspect=localhost:3000 -p 'inspector.url()'
Debugger listening on ws://localhost:3000/51cf8d0e-3c36-4c59-8efd-54519839e56a
For help, see: https://nodejs.org/en/docs/inspector
ws://localhost:3000/51cf8d0e-3c36-4c59-8efd-54519839e56a

$ node -p 'inspector.url()'
undefined
```

### `inspector.waitForDebugger()`

<!-- YAML
added: v12.7.0
-->

阻塞直到客户端（现有的或后来连接的）发送了 `Runtime.runIfWaitingForDebugger` 命令。

如果没有活动的检查器，将抛出异常。

## 与 DevTools 集成

> Stability: 1.1 - Active development

`node:inspector` 模块提供了与支持 Chrome DevTools Protocol 的 devtools 集成的 API。
连接到正在运行的 Node.js 实例的 DevTools 前端可以捕获从该实例发出的协议事件，并相应地显示它们以方便调试。
以下方法向所有连接的前端广播协议事件。
根据协议，传递给方法的 `params` 可以是可选的。

```js
// 将触发 `Network.requestWillBeSent` 事件。
inspector.Network.requestWillBeSent({
  requestId: 'request-id-1',
  timestamp: Date.now() / 1000,
  wallTime: Date.now(),
  request: {
    url: 'https://nodejs.org/en',
    method: 'GET',
  },
});
```

### `inspector.Network.dataReceived([params])`

<!-- YAML
added: v24.2.0
-->

* `params` {Object}

此功能仅在启用 `--experimental-network-inspection` 标志时可用。

向连接的前端广播 `Network.dataReceived` 事件，或者如果尚未为给定请求调用 `Network.streamResourceContent` 命令，则缓冲数据。

同时启用 `Network.getResponseBody` 命令来检索响应数据。

### `inspector.Network.dataSent([params])`

<!-- YAML
added: v24.3.0
-->

* `params` {Object}

此功能仅在启用 `--experimental-network-inspection` 标志时可用。

启用 `Network.getRequestPostData` 命令来检索请求数据。

### `inspector.Network.requestWillBeSent([params])`

<!-- YAML
added:
 - v22.6.0
 - v20.18.0
-->

* `params` {Object}

此功能仅在启用 `--experimental-network-inspection` 标志时可用。

向连接的前端广播 `Network.requestWillBeSent` 事件。此事件表明应用程序即将发送 HTTP 请求。

### `inspector.Network.responseReceived([params])`

<!-- YAML
added:
 - v22.6.0
 - v20.18.0
-->

* `params` {Object}

此功能仅在启用 `--experimental-network-inspection` 标志时可用。

向连接的前端广播 `Network.responseReceived` 事件。此事件表明 HTTP 响应已可用。

### `inspector.Network.loadingFinished([params])`

<!-- YAML
added:
 - v22.6.0
 - v20.18.0
-->

* `params` {Object}

此功能仅在启用 `--experimental-network-inspection` 标志时可用。

向连接的前端广播 `Network.loadingFinished` 事件。此事件表明 HTTP 请求已完成加载。

### `inspector.Network.loadingFailed([params])`

<!-- YAML
added:
 - v22.7.0
 - v20.18.0
-->

* `params` {Object}

此功能仅在启用 `--experimental-network-inspection` 标志时可用。

向连接的前端广播 `Network.loadingFailed` 事件。此事件表明 HTTP 请求加载失败。

### `inspector.Network.webSocketCreated([params])`

<!-- YAML
added:
  - v24.7.0
-->

* `params` {Object}

此功能仅在启用 `--experimental-network-inspection` 标志时可用。

向连接的前端广播 `Network.webSocketCreated` 事件。此事件表明已发起 WebSocket 连接。

### `inspector.Network.webSocketHandshakeResponseReceived([params])`

<!-- YAML
added:
  - v24.7.0
-->

* `params` {Object}

此功能仅在启用 `--experimental-network-inspection` 标志时可用。

向连接的前端广播 `Network.webSocketHandshakeResponseReceived` 事件。
此事件表明已收到 WebSocket 握手响应。

### `inspector.Network.webSocketClosed([params])`

<!-- YAML
added:
  - v24.7.0
-->

* `params` {Object}

此功能仅在启用 `--experimental-network-inspection` 标志时可用。

向连接的前端广播 `Network.webSocketClosed` 事件。
此事件表明 WebSocket 连接已关闭。

### `inspector.NetworkResources.put`

<!-- YAML
added:
  - v24.5.0
-->

> Stability: 1.1 - Active Development

此功能仅在启用 `--experimental-inspector-network-resource` 标志时可用。

inspector.NetworkResources.put 方法用于为通过 Chrome DevTools Protocol (CDP) 发出的 loadNetworkResource 请求提供响应。
这通常在通过 URL 指定源映射，并且 DevTools 前端（例如 Chrome）请求该资源以检索源映射时触发。

此方法允许开发者预定义要为此类 CDP 请求服务的资源内容。

```js
const inspector = require('node:inspector');
// 通过预先调用 put 来注册资源，当从前端发出 loadNetworkResource 请求时，可以解析源映射。
async function setNetworkResources() {
  const mapUrl = 'http://localhost:3000/dist/app.js.map';
  const tsUrl = 'http://localhost:3000/src/app.ts';
  const distAppJsMap = await fetch(mapUrl).then((res) => res.text());
  const srcAppTs = await fetch(tsUrl).then((res) => res.text());
  inspector.NetworkResources.put(mapUrl, distAppJsMap);
  inspector.NetworkResources.put(tsUrl, srcAppTs);
};
setNetworkResources().then(() => {
  require('./dist/app');
});
```

有关更多详细信息，请参阅官方 CDP 文档：[Network.loadNetworkResource](https://chromedevtools.github.io/devtools-protocol/tot/Network/#method-loadNetworkResource)

## 断点支持

Chrome DevTools Protocol 的 [`Debugger` 域][`Debugger` domain] 允许 `inspector.Session` 附加到程序并设置断点以单步执行代码。

但是，应避免使用通过 [`session.connect()`][] 连接的同线程 `inspector.Session` 设置断点，因为被附加和暂停的程序正是调试器本身。相反，尝试通过 [`session.connectToMainThread()`][] 连接到主线程并在 worker 线程中设置断点，或者通过 WebSocket 连接与 [Debugger][] 程序连接。

[CPU Profiler]: https://chromedevtools.github.io/devtools-protocol/v8/Profiler
[Chrome DevTools Protocol Viewer]: https://chromedevtools.github.io/devtools-protocol/v8/
[Debugger]: debugger.md
[Heap Profiler]: https://chromedevtools.github.io/devtools-protocol/v8/HeapProfiler
[`'Debugger.paused'`]: https://chromedevtools.github.io/devtools-protocol/v8/Debugger#event-paused
[`Debugger` domain]: https://chromedevtools.github.io/devtools-protocol/v8/Debugger
[`inspector.close()`]: #inspectorclose
[`session.connect()`]: #sessionconnect
[`session.connectToMainThread()`]: #sessionconnecttomainthread
[security warning]: cli.md#warning-binding-inspector-to-a-public-ipport-combination-is-insecure
[support of breakpoints]: #support-of-breakpoints
