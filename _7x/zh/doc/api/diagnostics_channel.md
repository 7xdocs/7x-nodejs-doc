# Diagnostics Channel

<!-- YAML
added:
  - v15.1.0
  - v14.17.0
changes:
  - version:
      - v19.2.0
      - v18.13.0
    pr-url: https://github.com/nodejs/node/pull/45290
    description: diagnostics_channel is now Stable.
-->

<!--introduced_in=v15.1.0-->

> Stability: 2 - Stable

<!-- source_link=lib/diagnostics_channel.js -->

`node:diagnostics_channel` 模块提供了一个 API 来创建命名通道，用于诊断目的报告任意消息数据。

可以通过以下方式访问：

```mjs
import diagnostics_channel from 'node:diagnostics_channel';
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');
```

模块开发者若想报告诊断消息，可以创建一个或多个顶层通道来报告消息。也可以在运行时获取通道，但由于这样做会产生额外开销，因此不鼓励。为了方便，通道可以被导出，但只要知道通道名称，就可以在任何地方获取。

如果你打算让你的模块产生供他人使用的诊断数据，建议你包含使用了哪些命名通道以及消息数据结构的文档。通道名称通常应包含模块名称，以避免与其他模块的数据发生冲突。

## 公共 API

### 概述

以下是公共 API 的简单概述。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

// 获取一个可重用的通道对象
const channel = diagnostics_channel.channel('my-channel');

function onMessage(message, name) {
  // 接收数据
}

// 订阅通道
diagnostics_channel.subscribe('my-channel', onMessage);

// 检查通道是否有活跃订阅者
if (channel.hasSubscribers) {
  // 发布数据到通道
  channel.publish({
    some: 'data',
  });
}

// 取消订阅通道
diagnostics_channel.unsubscribe('my-channel', onMessage);
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

// 获取一个可重用的通道对象
const channel = diagnostics_channel.channel('my-channel');

function onMessage(message, name) {
  // 接收数据
}

// 订阅通道
diagnostics_channel.subscribe('my-channel', onMessage);

// 检查通道是否有活跃订阅者
if (channel.hasSubscribers) {
  // 发布数据到通道
  channel.publish({
    some: 'data',
  });
}

// 取消订阅通道
diagnostics_channel.unsubscribe('my-channel', onMessage);
```

#### `diagnostics_channel.hasSubscribers(name)`

<!-- YAML
added:
 - v15.1.0
 - v14.17.0
-->

* `name` {string|symbol} 通道名称
* 返回: {boolean} 是否有活跃订阅者

检查指定名称的通道是否有活跃订阅者。如果你要发送的消息准备成本很高，这个 API 会很有帮助。

此 API 是可选的，但在尝试从性能非常敏感的代码发布消息时很有用。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

if (diagnostics_channel.hasSubscribers('my-channel')) {
  // 有订阅者，准备并发布消息
}
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

if (diagnostics_channel.hasSubscribers('my-channel')) {
  // 有订阅者，准备并发布消息
}
```

#### `diagnostics_channel.channel(name)`

<!-- YAML
added:
 - v15.1.0
 - v14.17.0
-->

* `name` {string|symbol} 通道名称
* 返回: {Channel} 命名通道对象

这是想要发布到命名通道的主要入口点。它产生一个通道对象，该对象经过优化以尽可能减少发布时的开销。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channel = diagnostics_channel.channel('my-channel');
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channel = diagnostics_channel.channel('my-channel');
```

#### `diagnostics_channel.subscribe(name, onMessage)`

<!-- YAML
added:
 - v18.7.0
 - v16.17.0
-->

* `name` {string|symbol} 通道名称
* `onMessage` {Function} 接收通道消息的处理程序
  * `message` {any} 消息数据
  * `name` {string|symbol} 通道名称

注册一个消息处理程序来订阅此通道。每当有消息发布到通道时，此消息处理程序将同步运行。消息处理程序中抛出的任何错误都将触发 [`'uncaughtException'`][]。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

diagnostics_channel.subscribe('my-channel', (message, name) => {
  // 接收数据
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

diagnostics_channel.subscribe('my-channel', (message, name) => {
  // 接收数据
});
```

#### `diagnostics_channel.unsubscribe(name, onMessage)`

<!-- YAML
added:
 - v18.7.0
 - v16.17.0
-->

* `name` {string|symbol} 通道名称
* `onMessage` {Function} 要移除的先前订阅的处理程序
* 返回: {boolean} 如果找到处理程序则为 `true`，否则为 `false`。

移除先前使用 [`diagnostics_channel.subscribe(name, onMessage)`][] 注册到此通道的消息处理程序。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

function onMessage(message, name) {
  // 接收数据
}

diagnostics_channel.subscribe('my-channel', onMessage);

diagnostics_channel.unsubscribe('my-channel', onMessage);
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

function onMessage(message, name) {
  // 接收数据
}

diagnostics_channel.subscribe('my-channel', onMessage);

diagnostics_channel.unsubscribe('my-channel', onMessage);
```

#### `diagnostics_channel.tracingChannel(nameOrChannels)`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

> Stability: 1 - Experimental

* `nameOrChannels` {string|TracingChannel} 通道名称或包含所有 [TracingChannel Channels][] 的对象
* 返回: {TracingChannel} 用于跟踪的通道集合

为给定的 [TracingChannel Channels][] 创建一个 [`TracingChannel`][] 包装器。如果给定名称，则相应的跟踪通道将以 `tracing:${name}:${eventType}` 的形式创建，其中 `eventType` 对应于 [TracingChannel Channels][] 的类型。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channelsByName = diagnostics_channel.tracingChannel('my-channel');

// 或者...

const channelsByCollection = diagnostics_channel.tracingChannel({
  start: diagnostics_channel.channel('tracing:my-channel:start'),
  end: diagnostics_channel.channel('tracing:my-channel:end'),
  asyncStart: diagnostics_channel.channel('tracing:my-channel:asyncStart'),
  asyncEnd: diagnostics_channel.channel('tracing:my-channel:asyncEnd'),
  error: diagnostics_channel.channel('tracing:my-channel:error'),
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channelsByName = diagnostics_channel.tracingChannel('my-channel');

// 或者...

const channelsByCollection = diagnostics_channel.tracingChannel({
  start: diagnostics_channel.channel('tracing:my-channel:start'),
  end: diagnostics_channel.channel('tracing:my-channel:end'),
  asyncStart: diagnostics_channel.channel('tracing:my-channel:asyncStart'),
  asyncEnd: diagnostics_channel.channel('tracing:my-channel:asyncEnd'),
  error: diagnostics_channel.channel('tracing:my-channel:error'),
});
```

### 类：`Channel`

<!-- YAML
added:
 - v15.1.0
 - v14.17.0
-->

`Channel` 类表示数据管道中的单个命名通道。它用于跟踪订阅者，并在有订阅者存在时发布消息。它作为一个独立的对象存在，以避免在发布时查找通道，从而实现非常快的发布速度，并允许大量使用而产生非常小的成本。通道是使用 [`diagnostics_channel.channel(name)`][] 创建的，不支持直接使用 `new Channel(name)` 构造通道。

#### `channel.hasSubscribers`

<!-- YAML
added:
 - v15.1.0
 - v14.17.0
-->

* 返回: {boolean} 是否有活跃订阅者

检查此通道是否有活跃订阅者。如果你要发送的消息准备成本很高，这个 API 会很有帮助。

此 API 是可选的，但在尝试从性能非常敏感的代码发布消息时很有用。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channel = diagnostics_channel.channel('my-channel');

if (channel.hasSubscribers) {
  // 有订阅者，准备并发布消息
}
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channel = diagnostics_channel.channel('my-channel');

if (channel.hasSubscribers) {
  // 有订阅者，准备并发布消息
}
```

#### `channel.publish(message)`

<!-- YAML
added:
 - v15.1.0
 - v14.17.0
-->

* `message` {any} 发送给通道订阅者的消息

向通道的任何订阅者发布消息。这将同步触发消息处理程序，因此它们将在同一上下文中执行。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channel = diagnostics_channel.channel('my-channel');

channel.publish({
  some: 'message',
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channel = diagnostics_channel.channel('my-channel');

channel.publish({
  some: 'message',
});
```

#### `channel.subscribe(onMessage)`

<!-- YAML
added:
 - v15.1.0
 - v14.17.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59758
    description: Deprecation revoked.
  - version:
    - v18.7.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/44943
    description: Documentation-only deprecation.
-->

* `onMessage` {Function} 接收通道消息的处理程序
  * `message` {any} 消息数据
  * `name` {string|symbol} 通道名称

注册一个消息处理程序来订阅此通道。每当有消息发布到通道时，此消息处理程序将同步运行。消息处理程序中抛出的任何错误都将触发 [`'uncaughtException'`][]。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channel = diagnostics_channel.channel('my-channel');

channel.subscribe((message, name) => {
  // 接收数据
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channel = diagnostics_channel.channel('my-channel');

channel.subscribe((message, name) => {
  // 接收数据
});
```

#### `channel.unsubscribe(onMessage)`

<!-- YAML
added:
 - v15.1.0
 - v14.17.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59758
    description: Deprecation revoked.
  - version:
    - v18.7.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/44943
    description: Documentation-only deprecation.
  - version:
    - v17.1.0
    - v16.14.0
    - v14.19.0
    pr-url: https://github.com/nodejs/node/pull/40433
    description: Added return value. Added to channels without subscribers.
-->

* `onMessage` {Function} 要移除的先前订阅的处理程序
* 返回: {boolean} 如果找到处理程序则为 `true`，否则为 `false`。

移除先前使用 [`channel.subscribe(onMessage)`][] 注册到此通道的消息处理程序。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channel = diagnostics_channel.channel('my-channel');

function onMessage(message, name) {
  // 接收数据
}

channel.subscribe(onMessage);

channel.unsubscribe(onMessage);
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channel = diagnostics_channel.channel('my-channel');

function onMessage(message, name) {
  // 接收数据
}

channel.subscribe(onMessage);

channel.unsubscribe(onMessage);
```

#### `channel.bindStore(store[, transform])`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

> Stability: 1 - Experimental

* `store` {AsyncLocalStorage} 要绑定上下文数据的存储
* `transform` {Function} 在设置存储上下文之前转换上下文数据

当调用 [`channel.runStores(context, ...)`][] 时，给定的上下文数据将应用于绑定到通道的任何存储。如果存储已经被绑定，先前的 `transform` 函数将被新的替换。可以省略 `transform` 函数，将给定的上下文数据直接设置为上下文。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';
import { AsyncLocalStorage } from 'node:async_hooks';

const store = new AsyncLocalStorage();

const channel = diagnostics_channel.channel('my-channel');

channel.bindStore(store, (data) => {
  return { data };
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');
const { AsyncLocalStorage } = require('node:async_hooks');

const store = new AsyncLocalStorage();

const channel = diagnostics_channel.channel('my-channel');

channel.bindStore(store, (data) => {
  return { data };
});
```

#### `channel.unbindStore(store)`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

> Stability: 1 - Experimental

* `store` {AsyncLocalStorage} 要从通道解绑的存储。
* 返回: {boolean} 如果找到存储则为 `true`，否则为 `false`。

移除先前使用 [`channel.bindStore(store)`][] 注册到此通道的消息处理程序。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';
import { AsyncLocalStorage } from 'node:async_hooks';

const store = new AsyncLocalStorage();

const channel = diagnostics_channel.channel('my-channel');

channel.bindStore(store);
channel.unbindStore(store);
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');
const { AsyncLocalStorage } = require('node:async_hooks');

const store = new AsyncLocalStorage();

const channel = diagnostics_channel.channel('my-channel');

channel.bindStore(store);
channel.unbindStore(store);
```

#### `channel.runStores(context, fn[, thisArg[, ...args]])`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

> Stability: 1 - Experimental

* `context` {any} 发送给订阅者并绑定到存储的消息
* `fn` {Function} 在进入的存储上下文中运行的处理程序
* `thisArg` {any} 函数调用的接收器。
* `...args` {any} 传递给函数的可选参数。

在给定函数的持续时间内，将给定的数据应用于绑定到通道的任何 AsyncLocalStorage 实例，然后在该数据应用于存储的范围内发布到通道。

如果给定了 [`channel.bindStore(store)`][] 一个转换函数，它将在消息数据成为存储的上下文值之前应用于转换消息数据。在需要上下文链接的情况下，可以从转换函数内部访问先前的存储上下文。

应用于存储的上下文应该可以在从给定函数期间开始的任何异步代码中访问，但是在某些情况下可能会发生 [上下文丢失][]。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';
import { AsyncLocalStorage } from 'node:async_hooks';

const store = new AsyncLocalStorage();

const channel = diagnostics_channel.channel('my-channel');

channel.bindStore(store, (message) => {
  const parent = store.getStore();
  return new Span(message, parent);
});
channel.runStores({ some: 'message' }, () => {
  store.getStore(); // Span({ some: 'message' })
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');
const { AsyncLocalStorage } = require('node:async_hooks');

const store = new AsyncLocalStorage();

const channel = diagnostics_channel.channel('my-channel');

channel.bindStore(store, (message) => {
  const parent = store.getStore();
  return new Span(message, parent);
});
channel.runStores({ some: 'message' }, () => {
  store.getStore(); // Span({ some: 'message' })
});
```

### 类：`TracingChannel`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

> Stability: 1 - Experimental

`TracingChannel` 类是一个 [TracingChannel Channels][] 的集合，它们共同表达一个可跟踪的操作。它用于形式化和简化生成跟踪应用程序流事件的过程。[`diagnostics_channel.tracingChannel()`][] 用于构造 `TracingChannel`。与 `Channel` 一样，建议在文件的顶层创建并重用单个 `TracingChannel`，而不是动态创建它们。

#### `tracingChannel.subscribe(subscribers)`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

* `subscribers` {Object} [TracingChannel Channels][] 订阅者集合
  * `start` {Function} [`start` event][] 订阅者
  * `end` {Function} [`end` event][] 订阅者
  * `asyncStart` {Function} [`asyncStart` event][] 订阅者
  * `asyncEnd` {Function} [`asyncEnd` event][] 订阅者
  * `error` {Function} [`error` event][] 订阅者

辅助函数，将一组函数订阅到相应的通道。这与在每个通道上单独调用 [`channel.subscribe(onMessage)`][] 相同。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.subscribe({
  start(message) {
    // 处理 start 消息
  },
  end(message) {
    // 处理 end 消息
  },
  asyncStart(message) {
    // 处理 asyncStart 消息
  },
  asyncEnd(message) {
    // 处理 asyncEnd 消息
  },
  error(message) {
    // 处理 error 消息
  },
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.subscribe({
  start(message) {
    // 处理 start 消息
  },
  end(message) {
    // 处理 end 消息
  },
  asyncStart(message) {
    // 处理 asyncStart 消息
  },
  asyncEnd(message) {
    // 处理 asyncEnd 消息
  },
  error(message) {
    // 处理 error 消息
  },
});
```

#### `tracingChannel.unsubscribe(subscribers)`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

* `subscribers` {Object} [TracingChannel Channels][] 订阅者集合
  * `start` {Function} [`start` event][] 订阅者
  * `end` {Function} [`end` event][] 订阅者
  * `asyncStart` {Function} [`asyncStart` event][] 订阅者
  * `asyncEnd` {Function} [`asyncEnd` event][] 订阅者
  * `error` {Function} [`error` event][] 订阅者
* 返回: {boolean} 如果所有处理程序都成功取消订阅则为 `true`，否则为 `false`。

辅助函数，从相应的通道取消订阅一组函数。这与在每个通道上单独调用 [`channel.unsubscribe(onMessage)`][] 相同。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.unsubscribe({
  start(message) {
    // 处理 start 消息
  },
  end(message) {
    // 处理 end 消息
  },
  asyncStart(message) {
    // 处理 asyncStart 消息
  },
  asyncEnd(message) {
    // 处理 asyncEnd 消息
  },
  error(message) {
    // 处理 error 消息
  },
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.unsubscribe({
  start(message) {
    // 处理 start 消息
  },
  end(message) {
    // 处理 end 消息
  },
  asyncStart(message) {
    // 处理 asyncStart 消息
  },
  asyncEnd(message) {
    // 处理 asyncEnd 消息
  },
  error(message) {
    // 处理 error 消息
  },
});
```

#### `tracingChannel.traceSync(fn[, context[, thisArg[, ...args]]])`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

* `fn` {Function} 要围绕其进行跟踪的函数
* `context` {Object} 用于关联事件的共享对象
* `thisArg` {any} 函数调用的接收器
* `...args` {any} 传递给函数的可选参数
* 返回: {any} 给定函数的返回值

跟踪同步函数调用。这总是在执行前后产生 [`start` event][] 和 [`end` event][]，如果给定函数抛出错误，可能会产生 [`error` event][]。这将在 `start` 通道上使用 [`channel.runStores(context, ...)`][] 运行给定函数，确保所有事件都应具有任何绑定的存储以匹配此跟踪上下文。

为确保仅形成正确的跟踪图，只有在跟踪开始之前存在订阅者时才会发布事件。在跟踪开始后添加的订阅者将不会收到该跟踪的未来事件，只有未来的跟踪才会被看到。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.traceSync(() => {
  // 做一些事情
}, {
  some: 'thing',
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.traceSync(() => {
  // 做一些事情
}, {
  some: 'thing',
});
```

#### `tracingChannel.tracePromise(fn[, context[, thisArg[, ...args]]])`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

* `fn` {Function} 要围绕其进行跟踪的返回 Promise 的函数
* `context` {Object} 用于关联跟踪事件的共享对象
* `thisArg` {any} 函数调用的接收器
* `...args` {any} 传递给函数的可选参数
* 返回: {Promise} 链接自给定函数返回的 promise

跟踪返回 promise 的函数调用。这总是在函数执行的同步部分前后产生 [`start` event][] 和 [`end` event][]，并在达到 promise 延续时产生 [`asyncStart` event][] 和 [`asyncEnd` event][]。如果给定函数抛出错误或返回的 promise 被拒绝，它也可能产生 [`error` event][]。这将在 `start` 通道上使用 [`channel.runStores(context, ...)`][] 运行给定函数，确保所有事件都应具有任何绑定的存储以匹配此跟踪上下文。

为确保仅形成正确的跟踪图，只有在跟踪开始之前存在订阅者时才会发布事件。在跟踪开始后添加的订阅者将不会收到该跟踪的未来事件，只有未来的跟踪才会被看到。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.tracePromise(async () => {
  // 做一些事情
}, {
  some: 'thing',
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.tracePromise(async () => {
  // 做一些事情
}, {
  some: 'thing',
});
```

#### `tracingChannel.traceCallback(fn[, position[, context[, thisArg[, ...args]]]])`

<!-- YAML
added:
 - v19.9.0
 - v18.19.0
-->

* `fn` {Function} 要围绕其进行跟踪的使用回调的函数
* `position` {number} 预期回调的从零开始的参数位置（如果传递 `undefined` 则默认为最后一个参数）
* `context` {Object} 用于关联跟踪事件的共享对象（如果传递 `undefined` 则默认为 `{}`）
* `thisArg` {any} 函数调用的接收器
* `...args` {any} 传递给函数的参数（必须包括回调）
* 返回: {any} 给定函数的返回值

跟踪接收回调的函数调用。回调应遵循通常使用的 error-first 约定。这总是在函数执行的同步部分前后产生 [`start` event][] 和 [`end` event][]，并在回调执行前后产生 [`asyncStart` event][] 和 [`asyncEnd` event][]。如果给定函数抛出或传递给回调的第一个参数被设置，它也可能产生 [`error` event][]。这将在 `start` 通道上使用 [`channel.runStores(context, ...)`][] 运行给定函数，确保所有事件都应具有任何绑定的存储以匹配此跟踪上下文。

为确保仅形成正确的跟踪图，只有在跟踪开始之前存在订阅者时才会发布事件。在跟踪开始后添加的订阅者将不会收到该跟踪的未来事件，只有未来的跟踪才会被看到。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.traceCallback((arg1, callback) => {
  // 做一些事情
  callback(null, 'result');
}, 1, {
  some: 'thing',
}, thisArg, arg1, callback);
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channels = diagnostics_channel.tracingChannel('my-channel');

channels.traceCallback((arg1, callback) => {
  // 做一些事情
  callback(null, 'result');
}, 1, {
  some: 'thing',
}, thisArg, arg1, callback);
```

回调也将使用 [`channel.runStores(context, ...)`][] 运行，这在某些情况下可以实现上下文丢失恢复。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';
import { AsyncLocalStorage } from 'node:async_hooks';

const channels = diagnostics_channel.tracingChannel('my-channel');
const myStore = new AsyncLocalStorage();

// start 通道将初始存储数据设置为某个值
// 并将该存储数据值存储在跟踪上下文对象上
channels.start.bindStore(myStore, (data) => {
  const span = new Span(data);
  data.span = span;
  return span;
});

// 然后 asyncStart 可以从之前存储的数据中恢复
channels.asyncStart.bindStore(myStore, (data) => {
  return data.span;
});
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');
const { AsyncLocalStorage } = require('node:async_hooks');

const channels = diagnostics_channel.tracingChannel('my-channel');
const myStore = new AsyncLocalStorage();

// start 通道将初始存储数据设置为某个值
// 并将该存储数据值存储在跟踪上下文对象上
channels.start.bindStore(myStore, (data) => {
  const span = new Span(data);
  data.span = span;
  return span;
});

// 然后 asyncStart 可以从之前存储的数据中恢复
channels.asyncStart.bindStore(myStore, (data) => {
  return data.span;
});
```

#### `tracingChannel.hasSubscribers`

<!-- YAML
added:
 - v22.0.0
 - v20.13.0
-->

* 返回: {boolean} 如果任何单个通道有订阅者则为 `true`，否则为 `false`。

这是 [`TracingChannel`][] 实例上可用的辅助方法，用于检查任何 [TracingChannel Channels][] 是否有订阅者。如果其中任何一个至少有一个订阅者，则返回 `true`，否则返回 `false`。

```mjs
import diagnostics_channel from 'node:diagnostics_channel';

const channels = diagnostics_channel.tracingChannel('my-channel');

if (channels.hasSubscribers) {
  // 做一些事情
}
```

```cjs
const diagnostics_channel = require('node:diagnostics_channel');

const channels = diagnostics_channel.tracingChannel('my-channel');

if (channels.hasSubscribers) {
  // 做一些事情
}
```

### TracingChannel Channels

TracingChannel 是几个 diagnostics_channel 的集合，代表单个可跟踪操作的执行生命周期中的特定点。行为分为五个 diagnostics_channel，包括 `start`、`end`、`asyncStart`、`asyncEnd` 和 `error`。单个可跟踪操作将在所有事件之间共享相同的事件对象，这有助于通过 weakmap 管理关联。

这些事件对象将在任务"完成"时使用 `result` 或 `error` 值进行扩展。对于同步任务，`result` 将是返回值，`error` 将是函数抛出的任何东西。对于基于回调的异步函数，`result` 将是回调的第二个参数，而 `error` 要么是 `end` 事件中可见的抛出错误，要么是 `asyncStart` 或 `asyncEnd` 事件中的第一个回调参数。

为确保仅形成正确的跟踪图，只有在跟踪开始之前存在订阅者时才会发布事件。在跟踪开始后添加的订阅者不应收到该跟踪的未来事件，只有未来的跟踪才会被看到。

跟踪通道应遵循以下命名模式：

* `tracing:module.class.method:start` 或 `tracing:module.function:start`
* `tracing:module.class.method:end` 或 `tracing:module.function:end`
* `tracing:module.class.method:asyncStart` 或 `tracing:module.function:asyncStart`
* `tracing:module.class.method:asyncEnd` 或 `tracing:module.function:asyncEnd`
* `tracing:module.class.method:error` 或 `tracing:module.function:error`

#### `start(event)`

* 名称: `tracing:${name}:start`

`start` 事件表示函数被调用的点。此时，事件数据可能包含函数参数或在函数执行开始时可用的任何内容。

#### `end(event)`

* 名称: `tracing:${name}:end`

`end` 事件表示函数调用返回值的点。对于异步函数，这是返回 promise 的时候，而不是函数内部进行返回语句的时候。此时，如果被跟踪的函数是同步的，`result` 字段将设置为函数的返回值。或者，`error` 字段可能存在以表示任何抛出的错误。

建议专门监听 `error` 事件来跟踪错误，因为一个可跟踪操作可能产生多个错误。例如，一个内部启动的异步任务可能失败，然后任务的同步部分抛出错误。

#### `asyncStart(event)`

* 名称: `tracing:${name}:asyncStart`

`asyncStart` 事件表示到达可跟踪函数的回调或延续。此时，回调参数之类的东西可能可用，或者表达操作"结果"的任何其他内容。

对于基于回调的函数，回调的第一个参数将被分配给 `error` 字段（如果不是 `undefined` 或 `null`），第二个参数将被分配给 `result` 字段。

对于 promise，`resolve` 路径的参数将被分配给 `result`，`reject` 路径的参数将被分配给 `error`。

建议专门监听 `error` 事件来跟踪错误，因为一个可跟踪操作可能产生多个错误。例如，一个内部启动的异步任务可能失败，然后任务的同步部分抛出错误。

#### `asyncEnd(event)`

* 名称: `tracing:${name}:asyncEnd`

`asyncEnd` 事件表示异步函数的回调返回。在 `asyncStart` 事件之后，事件数据不太可能改变，但查看回调完成的点可能很有用。

#### `error(event)`

* 名称: `tracing:${name}:error`

`error` 事件表示可跟踪函数同步或异步产生的任何错误。如果在被跟踪函数的同步部分抛出错误，错误将被分配给事件的 `error` 字段并触发 `error` 事件。如果通过回调或 promise 拒绝异步接收错误，它也将被分配给事件的 `error` 字段并触发 `error` 事件。

单个可跟踪函数调用可能多次产生错误，因此在消费此事件时应考虑这一点。例如，如果内部触发了另一个失败的异步任务，然后函数的同步部分抛出错误，则将发出两个 `error` 事件，一个用于同步错误，一个用于异步错误。

### 内置通道

#### Console

> Stability: 1 - Experimental

##### 事件: `'console.log'`

* `args` {any\[]}

当 `console.log()` 被调用时触发。接收传递给 `console.log()` 的参数数组。

##### 事件: `'console.info'`

* `args` {any\[]}

当 `console.info()` 被调用时触发。接收传递给 `console.info()` 的参数数组。

##### 事件: `'console.debug'`

* `args` {any\[]}

当 `console.debug()` 被调用时触发。接收传递给 `console.debug()` 的参数数组。

##### 事件: `'console.warn'`

* `args` {any\[]}

当 `console.warn()` 被调用时触发。接收传递给 `console.warn()` 的参数数组。

##### 事件: `'console.error'`

* `args` {any\[]}

当 `console.error()` 被调用时触发。接收传递给 `console.error()` 的参数数组。

#### HTTP

> Stability: 1 - Experimental

##### 事件: `'http.client.request.created'`

* `request` {http.ClientRequest}

当客户端创建请求对象时触发。
与 `http.client.request.start` 不同，此事件在请求发送之前触发。

##### 事件: `'http.client.request.start'`

* `request` {http.ClientRequest}

当客户端开始请求时触发。

##### 事件: `'http.client.request.error'`

* `request` {http.ClientRequest}
* `error` {Error}

当客户端请求期间发生错误时触发。

##### 事件: `'http.client.response.finish'`

* `request` {http.ClientRequest}
* `response` {http.IncomingMessage}

当客户端收到响应时触发。

##### 事件: `'http.server.request.start'`

* `request` {http.IncomingMessage}
* `response` {http.ServerResponse}
* `socket` {net.Socket}
* `server` {http.Server}

当服务器收到请求时触发。

##### 事件: `'http.server.response.created'`

* `request` {http.IncomingMessage}
* `response` {http.ServerResponse}

当服务器创建响应时触发。
此事件在响应发送之前触发。

##### 事件: `'http.server.response.finish'`

* `request` {http.IncomingMessage}
* `response` {http.ServerResponse}
* `socket` {net.Socket}
* `server` {http.Server}

当服务器发送响应时触发。

#### HTTP/2

> Stability: 1 - Experimental

##### 事件: `'http2.client.stream.created'`

* `stream` {ClientHttp2Stream}
* `headers` {HTTP/2 Headers Object}

当在客户端创建流时触发。

##### 事件: `'http2.client.stream.start'`

* `stream` {ClientHttp2Stream}
* `headers` {HTTP/2 Headers Object}

当在客户端启动流时触发。

##### 事件: `'http2.client.stream.error'`

* `stream` {ClientHttp2Stream}
* `error` {Error}

当在客户端处理流期间发生错误时触发。

##### 事件: `'http2.client.stream.finish'`

* `stream` {ClientHttp2Stream}
* `headers` {HTTP/2 Headers Object}
* `flags` {number}

当在客户端收到流时触发。

##### 事件: `'http2.client.stream.close'`

* `stream` {ClientHttp2Stream}

当在客户端关闭流时触发。关闭流时使用的 HTTP/2 错误代码可以使用 `stream.rstCode` 属性检索。

##### 事件: `'http2.server.stream.created'`

* `stream` {ServerHttp2Stream}
* `headers` {HTTP/2 Headers Object}

当在服务器创建流时触发。

##### 事件: `'http2.server.stream.start'`

* `stream` {ServerHttp2Stream}
* `headers` {HTTP/2 Headers Object}

当在服务器启动流时触发。

##### 事件: `'http2.server.stream.error'`

* `stream` {ServerHttp2Stream}
* `error` {Error}

当在服务器处理流期间发生错误时触发。

##### 事件: `'http2.server.stream.finish'`

* `stream` {ServerHttp2Stream}
* `headers` {HTTP/2 Headers Object}
* `flags` {number}

当在服务器发送流时触发。

##### 事件: `'http2.server.stream.close'`

* `stream` {ServerHttp2Stream}

当在服务器关闭流时触发。关闭流时使用的 HTTP/2 错误代码可以使用 `stream.rstCode` 属性检索。

#### Modules

> Stability: 1 - Experimental

##### 事件: `'module.require.start'`

* `event` {Object} 包含以下属性
  * `id` 传递给 `require()` 的参数。模块名称。
  * `parentFilename` 尝试 require(id) 的模块名称。

当 `require()` 执行时触发。参见 [`start` event][]。

##### 事件: `'module.require.end'`

* `event` {Object} 包含以下属性
  * `id` 传递给 `require()` 的参数。模块名称。
  * `parentFilename` 尝试 require(id) 的模块名称。

当 `require()` 调用返回时触发。参见 [`end` event][]。

##### 事件: `'module.require.error'`

* `event` {Object} 包含以下属性
  * `id` 传递给 `require()` 的参数。模块名称。
  * `parentFilename` 尝试 require(id) 的模块名称。
* `error` {Error}

当 `require()` 抛出错误时触发。参见 [`error` event][]。

##### 事件: `'module.import.asyncStart'`

* `event` {Object} 包含以下属性
  * `id` 传递给 `import()` 的参数。模块名称。
  * `parentURL` 尝试 import(id) 的模块的 URL 对象。

当 `import()` 被调用时触发。参见 [`asyncStart` event][]。

##### 事件: `'module.import.asyncEnd'`

* `event` {Object} 包含以下属性
  * `id` 传递给 `import()` 的参数。模块名称。
  * `parentURL` 尝试 import(id) 的模块的 URL 对象。

当 `import()` 完成时触发。参见 [`asyncEnd` event][]。

##### 事件: `'module.import.error'`

* `event` {Object} 包含以下属性
  * `id` 传递给 `import()` 的参数。模块名称。
  * `parentURL` 尝试 import(id) 的模块的 URL 对象。
* `error` {Error}

当 `import()` 抛出错误时触发。参见 [`error` event][]。

#### NET

> Stability: 1 - Experimental

##### 事件: `'net.client.socket'`

* `socket` {net.Socket|tls.TLSSocket}

当创建新的 TCP 或管道客户端套接字连接时触发。

##### 事件: `'net.server.socket'`

* `socket` {net.Socket}

当收到新的 TCP 或管道连接时触发。

##### 事件: `'tracing:net.server.listen:asyncStart'`

* `server` {net.Server}
* `options` {Object}

当调用 [`net.Server.listen()`][] 时触发，在实际设置端口或管道之前。

##### 事件: `'tracing:net.server.listen:asyncEnd'`

* `server` {net.Server}

当 [`net.Server.listen()`][] 完成并且服务器准备好接受连接时触发。

##### 事件: `'tracing:net.server.listen:error'`

* `server` {net.Server}
* `error` {Error}

当 [`net.Server.listen()`][] 返回错误时触发。

#### UDP

> Stability: 1 - Experimental

##### 事件: `'udp.socket'`

* `socket` {dgram.Socket}

当创建新的 UDP 套接字时触发。

#### Process

> Stability: 1 - Experimental

<!-- YAML
added: v16.18.0
-->

##### 事件: `'child_process'`

* `process` {ChildProcess}

当创建新进程时触发。

##### 事件: `'execve'`

* `execPath` {string}
* `args` {string\[]}
* `env` {string\[]}

当调用 [`process.execve()`][] 时触发。

#### Worker Thread

> Stability: 1 - Experimental

<!-- YAML
added: v16.18.0
-->

##### 事件: `'worker_threads'`

* `worker` {Worker}

当创建新线程时触发。

[TracingChannel Channels]: #tracingchannel-channels
[`'uncaughtException'`]: process.md#event-uncaughtexception
[`TracingChannel`]: #class-tracingchannel
[`asyncEnd` event]: #asyncendevent
[`asyncStart` event]: #asyncstartevent
[`channel.bindStore(store)`]: #channelbindstorestore-transform
[`channel.runStores(context, ...)`]: #channelrunstorescontext-fn-thisarg-args
[`channel.subscribe(onMessage)`]: #channelsubscribeonmessage
[`channel.unsubscribe(onMessage)`]: #channelunsubscribeonmessage
[`diagnostics_channel.channel(name)`]: #diagnostics_channelchannelname
[`diagnostics_channel.subscribe(name, onMessage)`]: #diagnostics_channelsubscribename-onmessage
[`diagnostics_channel.tracingChannel()`]: #diagnostics_channeltracingchannelnameorchannels
[`end` event]: #endevent
[`error` event]: #errorevent
[`net.Server.listen()`]: net.md#serverlisten
[`process.execve()`]: process.md#processexecvefile-args-env
[`start` event]: #startevent
[context loss]: async_context.md#troubleshooting-context-loss