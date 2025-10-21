# Cluster 集群

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/cluster.js -->

Node.js 的进程集群可用于运行多个 Node.js 实例，这些实例可以分配工作负载 among their application threads。当不需要进程隔离时，改用 [`worker_threads`][] 模块，它允许在单个 Node.js 实例中运行多个应用程序线程。

cluster 模块允·许轻松创建共享服务器端口的子进程。

```mjs
import cluster from 'node:cluster';
import http from 'node:http';
import { availableParallelism } from 'node:os';
import process from 'node:process';

const numCPUs = availableParallelism();

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  // Fork workers.
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`worker ${worker.process.pid} died`);
  });
} else {
  // Workers can share any TCP connection
  // In this case it is an HTTP server
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end('hello world\n');
  }).listen(8000);

  console.log(`Worker ${process.pid} started`);
}
```

```cjs
const cluster = require('node:cluster');
const http = require('node:http');
const numCPUs = require('node:os').availableParallelism();
const process = require('node:process');

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  // Fork workers.
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('exit', (worker, code, signal) => {
    console.log(`worker ${worker.process.pid} died`);
  });
} else {
  // Workers can share any TCP connection
  // In this case it is an HTTP server
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end('hello world\n');
  }).listen(8000);

  console.log(`Worker ${process.pid} started`);
}
```

现在运行 Node.js 将在工作进程之间共享端口 8000：

```console
$ node server.js
Primary 3596 is running
Worker 4324 started
Worker 4520 started
Worker 6056 started
Worker 5644 started
```

在 Windows 上，尚无法在工作进程中设置命名管道服务器。

## 工作原理

<!--type=misc-->

工作进程使用 [`child_process.fork()`][] 方法生成，因此它们可以通过 IPC 与父进程通信并在来回传递服务器句柄。

cluster 模块支持两种分发传入连接的方法。

第一种（也是除 Windows 外所有平台上的默认方法）是轮询方法，其中主进程监听一个端口，接受新连接并以轮询方式将它们分发给工作进程，同时内置了一些智能机制以避免工作进程过载。

第二种方法是主进程创建监听套接字并将其发送给感兴趣的工作进程。然后工作进程直接接受传入连接。

理论上，第二种方法应该能提供最佳性能。然而，实际上，由于操作系统调度程序的变幻莫测，分发往往非常不平衡。据观察，在总共八个进程的情况下，超过 70% 的连接最终只集中在两个进程中。

因为 `server.listen()` 将大部分工作交给了主进程，所以在以下三种情况下，普通 Node.js 进程和集群工作进程的行为会有所不同：

1.  `server.listen({fd: 7})` 因为消息被传递给主进程，所以将监听**父进程**中的文件描述符 7，并将句柄传递给工作进程，而不是监听工作进程所认为的文件描述符 7 所引用的内容。
2.  `server.listen(handle)` 显式地监听句柄将导致工作进程使用提供的句柄，而不是与主进程通信。
3.  `server.listen(0)` 通常，这会导致服务器监听一个随机端口。然而，在集群中，每个工作进程在每次执行 `listen(0)` 时都会收到相同的"随机"端口。本质上，端口在第一次是随机的，但此后是可预测的。要监听唯一端口，请基于集群工作进程 ID 生成端口号。

Node.js 不提供路由逻辑。因此，设计应用程序时，重要的一点是不要过于依赖内存中的数据对象来处理会话和登录等事务。

因为工作进程都是独立的进程，它们可以根据程序的需要被杀死或重新生成，而不会影响其他工作进程。只要还有工作进程存活，服务器就会继续接受连接。如果没有工作进程存活，现有连接将被丢弃，新连接将被拒绝。然而，Node.js 不会自动管理工作进程的数量。应用程序有责任根据自己的需要管理工作进程池。

尽管 `node:cluster` 模块的一个主要用例是网络，但它也可以用于其他需要工作进程的用例。

## 类：`Worker`

<!-- YAML
added: v0.7.0
-->

* 继承自：{EventEmitter}

`Worker` 对象包含关于工作进程的所有公共信息和方法。在主进程中，可以通过 `cluster.workers` 获取。在工作进程中，可以通过 `cluster.worker` 获取。

### 事件：`'disconnect'`

<!-- YAML
added: v0.7.7
-->

类似于 `cluster.on('disconnect')` 事件，但特定于此工作进程。

```js
cluster.fork().on('disconnect', () => {
  // Worker has disconnected
});
```

### 事件：`'error'`

<!-- YAML
added: v0.7.3
-->

此事件与 [`child_process.fork()`][] 提供的事件相同。

在工作进程中，也可以使用 `process.on('error')`。

### 事件：`'exit'`

<!-- YAML
added: v0.11.2
-->

* `code` {number} 退出代码，如果正常退出。
* `signal` {string} 导致进程被终止的信号名称（例如 `'SIGHUP'`）。

类似于 `cluster.on('exit')` 事件，但特定于此工作进程。

```mjs
import cluster from 'node:cluster';

if (cluster.isPrimary) {
  const worker = cluster.fork();
  worker.on('exit', (code, signal) => {
    if (signal) {
      console.log(`worker was killed by signal: ${signal}`);
    } else if (code !== 0) {
      console.log(`worker exited with error code: ${code}`);
    } else {
      console.log('worker success!');
    }
  });
}
```

```cjs
const cluster = require('node:cluster');

if (cluster.isPrimary) {
  const worker = cluster.fork();
  worker.on('exit', (code, signal) => {
    if (signal) {
      console.log(`worker was killed by signal: ${signal}`);
    } else if (code !== 0) {
      console.log(`worker exited with error code: ${code}`);
    } else {
      console.log('worker success!');
    }
  });
}
```

### 事件：`'listening'`

<!-- YAML
added: v0.7.0
-->

* `address` {Object}

类似于 `cluster.on('listening')` 事件，但特定于此工作进程。

```mjs
cluster.fork().on('listening', (address) => {
  // Worker is listening
});
```

```cjs
cluster.fork().on('listening', (address) => {
  // Worker is listening
});
```

此事件不会在工作进程中触发。

### 事件：`'message'`

<!-- YAML
added: v0.7.0
-->

* `message` {Object}
* `handle` {undefined|Object}

类似于 `cluster` 的 `'message'` 事件，但特定于此工作进程。

在工作进程中，也可以使用 `process.on('message')`。

参见 [`process` event: `'message'`][]。

下面是一个使用消息系统的例子。它在主进程中记录工作进程接收到的 HTTP 请求数量：

```mjs
import cluster from 'node:cluster';
import http from 'node:http';
import { availableParallelism } from 'node:os';
import process from 'node:process';

if (cluster.isPrimary) {

  // Keep track of http requests
  let numReqs = 0;
  setInterval(() => {
    console.log(`numReqs = ${numReqs}`);
  }, 1000);

  // Count requests
  function messageHandler(msg) {
    if (msg.cmd && msg.cmd === 'notifyRequest') {
      numReqs += 1;
    }
  }

  // Start workers and listen for messages containing notifyRequest
  const numCPUs = availableParallelism();
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  for (const id in cluster.workers) {
    cluster.workers[id].on('message', messageHandler);
  }

} else {

  // Worker processes have a http server.
  http.Server((req, res) => {
    res.writeHead(200);
    res.end('hello world\n');

    // Notify primary about the request
    process.send({ cmd: 'notifyRequest' });
  }).listen(8000);
}
```

```cjs
const cluster = require('node:cluster');
const http = require('node:http');
const numCPUs = require('node:os').availableParallelism();
const process = require('node:process');

if (cluster.isPrimary) {

  // Keep track of http requests
  let numReqs = 0;
  setInterval(() => {
    console.log(`numReqs = ${numReqs}`);
  }, 1000);

  // Count requests
  function messageHandler(msg) {
    if (msg.cmd && msg.cmd === 'notifyRequest') {
      numReqs += 1;
    }
  }

  // Start workers and listen for messages containing notifyRequest
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  for (const id in cluster.workers) {
    cluster.workers[id].on('message', messageHandler);
  }

} else {

  // Worker processes have a http server.
  http.Server((req, res) => {
    res.writeHead(200);
    res.end('hello world\n');

    // Notify primary about the request
    process.send({ cmd: 'notifyRequest' });
  }).listen(8000);
}
```

### 事件：`'online'`

<!-- YAML
added: v0.7.0
-->

类似于 `cluster.on('online')` 事件，但特定于此工作进程。

```js
cluster.fork().on('online', () => {
  // Worker is online
});
```

此事件不会在工作进程中触发。

### `worker.disconnect()`

<!-- YAML
added: v0.7.7
changes:
  - version: v7.3.0
    pr-url: https://github.com/nodejs/node/pull/10019
    description: This method now returns a reference to `worker`.
-->

* 返回：{cluster.Worker} 对 `worker` 的引用。

在工作进程中，此函数将关闭所有服务器，等待这些服务器上的 `'close'` 事件，然后断开 IPC 通道。

在主进程中，会向工作进程发送一个内部消息，使其调用自身的 `.disconnect()`。

会导致 `.exitedAfterDisconnect` 被设置。

服务器关闭后，它将不再接受新连接，但新连接可能被任何其他正在监听的工作进程接受。现有连接将被允许正常关闭。当不再存在连接时（参见 [`server.close()`][]），到工作进程的 IPC 通道将关闭，允许其优雅地退出。

以上内容*仅*适用于服务器连接，客户端连接不会由工作进程自动关闭，并且在退出前，disconnect 不会等待它们关闭。

在工作进程中，`process.disconnect` 存在，但它不是此函数；它是 [`disconnect()`][]。

由于长时间存活的服务器连接可能会阻止工作进程断开连接，发送一条消息可能会有用，以便可以采取应用程序特定的操作来关闭它们。实现超时机制也可能有用，如果在一定时间后仍未发出 `'disconnect'` 事件，则杀死工作进程。

```js
if (cluster.isPrimary) {
  const worker = cluster.fork();
  let timeout;

  worker.on('listening', (address) => {
    worker.send('shutdown');
    worker.disconnect();
    timeout = setTimeout(() => {
      worker.kill();
    }, 2000);
  });

  worker.on('disconnect', () => {
    clearTimeout(timeout);
  });

} else if (cluster.isWorker) {
  const net = require('node:net');
  const server = net.createServer((socket) => {
    // Connections never end
  });

  server.listen(8000);

  process.on('message', (msg) => {
    if (msg === 'shutdown') {
      // Initiate graceful close of any connections to server
    }
  });
}
```

### `worker.exitedAfterDisconnect`

<!-- YAML
added: v6.0.0
-->

* 类型：{boolean}

如果工作进程由于 `.disconnect()` 而退出，则此属性为 `true`。如果工作进程以任何其他方式退出，则为 `false`。如果工作进程尚未退出，则为 `undefined`。

布尔值 [`worker.exitedAfterDisconnect`][] 允许区分自愿退出和意外退出，主进程可以根据此值选择不重新生成工作进程。

```js
cluster.on('exit', (worker, code, signal) => {
  if (worker.exitedAfterDisconnect === true) {
    console.log('Oh, it was just voluntary – no need to worry');
  }
});

// kill worker
worker.kill();
```

### `worker.id`

<!-- YAML
added: v0.8.0
-->

* 类型：{integer}

每个新工作进程都会被赋予自己唯一的 id，这个 id 存储在 `id` 中。

当工作进程存活时，这是在 `cluster.workers` 中索引它的键。

### `worker.isConnected()`

<!-- YAML
added: v0.11.14
-->

如果工作进程通过其 IPC 通道连接到其主进程，则此函数返回 `true`，否则返回 `false`。工作进程在创建后即连接到其主进程。在 `'disconnect'` 事件发出后，它会断开连接。

### `worker.isDead()`

<!-- YAML
added: v0.11.14
-->

如果工作进程的进程已终止（无论是由于退出还是被发送信号），则此函数返回 `true`。否则，返回 `false`。

```mjs
import cluster from 'node:cluster';
import http from 'node:http';
import { availableParallelism } from 'node:os';
import process from 'node:process';

const numCPUs = availableParallelism();

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  // Fork workers.
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('fork', (worker) => {
    console.log('worker is dead:', worker.isDead());
  });

  cluster.on('exit', (worker, code, signal) => {
    console.log('worker is dead:', worker.isDead());
  });
} else {
  // Workers can share any TCP connection. In this case, it is an HTTP server.
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Current process\n ${process.pid}`);
    process.kill(process.pid);
  }).listen(8000);
}
```

```cjs
const cluster = require('node:cluster');
const http = require('node:http');
const numCPUs = require('node:os').availableParallelism();
const process = require('node:process');

if (cluster.isPrimary) {
  console.log(`Primary ${process.pid} is running`);

  // Fork workers.
  for (let i = 0; i < numCPUs; i++) {
    cluster.fork();
  }

  cluster.on('fork', (worker) => {
    console.log('worker is dead:', worker.isDead());
  });

  cluster.on('exit', (worker, code, signal) => {
    console.log('worker is dead:', worker.isDead());
  });
} else {
  // Workers can share any TCP connection. In this case, it is an HTTP server.
  http.createServer((req, res) => {
    res.writeHead(200);
    res.end(`Current process\n ${process.pid}`);
    process.kill(process.pid);
  }).listen(8000);
}
```

### `worker.kill([signal])`

<!-- YAML
added: v0.9.12
-->

* `signal` {string} 要发送给工作进程的终止信号名称。**默认值:** `'SIGTERM'`

此函数将杀死工作进程。在主工作进程中，它通过断开 `worker.process` 的连接来实现，一旦断开连接，就用 `signal` 杀死。在工作进程中，它通过用 `signal` 杀死进程来实现。

`kill()` 函数在不等待优雅断开连接的情况下杀死工作进程，它具有与 `worker.process.kill()` 相同的行为。

此方法别名为 `worker.destroy()` 以保持向后兼容性。

在工作进程中，`process.kill()` 存在，但它不是此函数；它是 [`kill()`][]。

### `worker.process`

<!-- YAML
added: v0.7.0
-->

* 类型：{ChildProcess}

所有工作进程都是使用 [`child_process.fork()`][] 创建的，该函数返回的对象存储为 `.process`。在工作进程中，全局 `process` 被存储。

参见：[Child Process module][]。

如果 `process` 上发生 `'disconnect'` 事件且 `.exitedAfterDisconnect` 不为 `true`，工作进程将调用 `process.exit(0)`。这可以防止意外断开连接。

### `worker.send(message[, sendHandle[, options]][, callback])`

<!-- YAML
added: v0.7.0
changes:
  - version: v4.0.0
    pr-url: https://github.com/nodejs/node/pull/2620
    description: The `callback` parameter is supported now.
-->

* `message` {Object}
* `sendHandle` {Handle}
* `options` {Object} 如果存在 `options` 参数，它是一个用于参数化某些类型句柄的传递的对象。`options` 支持以下属性：
  * `keepOpen` {boolean} 当传递 `net.Socket` 实例时可以使用的值。当为 `true` 时，套接字在发送进程中保持打开状态。**默认值:** `false`。
* `callback` {Function}
* 返回：{boolean}

向工作进程或主进程发送消息，可以选择附带一个句柄。

在主进程中，这会向特定工作进程发送消息。它与 [`ChildProcess.send()`][] 相同。

在工作进程中，这会向主进程发送消息。它与 `process.send()` 相同。

这个例子将回显来自主进程的所有消息：

```js
if (cluster.isPrimary) {
  const worker = cluster.fork();
  worker.send('hi there');

} else if (cluster.isWorker) {
  process.on('message', (msg) => {
    process.send(msg);
  });
}
```

## 事件：`'disconnect'`

<!-- YAML
added: v0.7.9
-->

* `worker` {cluster.Worker}

在工作进程 IPC 通道断开连接后发出。这可能发生在工作进程优雅退出、被杀死或手动断开连接（例如使用 `worker.disconnect()`）时。

在 `'disconnect'` 和 `'exit'` 事件之间可能会有延迟。这些事件可用于检测进程是否在清理过程中卡住，或者是否存在长时间存活的连接。

```js
cluster.on('disconnect', (worker) => {
  console.log(`The worker #${worker.id} has disconnected`);
});
```

## 事件：`'exit'`

<!-- YAML
added: v0.7.9
-->

* `worker` {cluster.Worker}
* `code` {number} 退出代码，如果正常退出。
* `signal` {string} 导致进程被终止的信号名称（例如 `'SIGHUP'`）。

当任何工作进程死亡时，cluster 模块将发出 `'exit'` 事件。

这可用于通过再次调用 [`.fork()`][] 来重新启动工作进程。

```js
cluster.on('exit', (worker, code, signal) => {
  console.log('worker %d died (%s). restarting...',
              worker.process.pid, signal || code);
  cluster.fork();
});
```

参见 [`child_process` event: `'exit'`][]。

## 事件：`'fork'`

<!-- YAML
added: v0.7.0
-->

* `worker` {cluster.Worker}

当新的工作进程被 fork 时，cluster 模块将发出 `'fork'` 事件。这可用于记录工作进程活动，并创建自定义超时。

```js
const timeouts = [];
function errorMsg() {
  console.error('Something must be wrong with the connection ...');
}

cluster.on('fork', (worker) => {
  timeouts[worker.id] = setTimeout(errorMsg, 2000);
});
cluster.on('listening', (worker, address) => {
  clearTimeout(timeouts[worker.id]);
});
cluster.on('exit', (worker, code, signal) => {
  clearTimeout(timeouts[worker.id]);
  errorMsg();
});
```

## 事件：`'listening'`

<!-- YAML
added: v0.7.0
-->

* `worker` {cluster.Worker}
* `address` {Object}

当从工作进程调用 `listen()` 后，在服务器上发出 `'listening'` 事件时，在主进程的 `cluster` 上也会发出 `'listening'` 事件。

事件处理程序使用两个参数执行，`worker` 包含工作进程对象，`address` 对象包含以下连接属性：`address`、`port` 和 `addressType`。如果工作进程正在监听多个地址，这将非常有用。

```js
cluster.on('listening', (worker, address) => {
  console.log(
    `A worker is now connected to ${address.address}:${address.port}`);
});
```

`addressType` 是以下之一：

* `4` (TCPv4)
* `6` (TCPv6)
* `-1` (Unix 域套接字)
* `'udp4'` 或 `'udp6'` (UDPv4 或 UDPv6)

## 事件：`'message'`

<!-- YAML
added: v2.5.0
changes:
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/5361
    description: The `worker` parameter is passed now; see below for details.
-->

* `worker` {cluster.Worker}
* `message` {Object}
* `handle` {undefined|Object}

当集群主进程从任何工作进程接收到消息时发出。

参见 [`child_process` event: `'message'`][]。

## 事件：`'online'`

<!-- YAML
added: v0.7.0
-->

* `worker` {cluster.Worker}

在 fork 新的工作进程后，工作进程应回复一条在线消息。当主进程收到在线消息时，将发出此事件。`'fork'` 和 `'online'` 之间的区别在于，fork 是在主进程 fork 工作进程时发出的，而 `'online'` 是在工作进程运行时发出的。

```js
cluster.on('online', (worker) => {
  console.log('Yay, the worker responded after it was forked');
});
```

## 事件：`'setup'`

<!-- YAML
added: v0.7.1
-->

* `settings` {Object}

每次调用 [`.setupPrimary()`][] 时发出。

`settings` 对象是 [`.setupPrimary()`][] 调用时的 `cluster.settings` 对象，并且仅是建议性的，因为可以在单个 tick 内多次调用 [`.setupPrimary()`][]。

如果准确性很重要，请使用 `cluster.settings`。

## `cluster.disconnect([callback])`

<!-- YAML
added: v0.7.7
-->

* `callback` {Function} 当所有工作进程都断开连接且句柄关闭时调用。

在 `cluster.workers` 中的每个工作进程上调用 `.disconnect()`。

当它们都断开连接后，所有内部句柄都将关闭，如果没有其他事件在等待，则允许主进程优雅退出。

该方法接受一个可选的回调参数，该参数在完成时调用。

这只能从主进程调用。

## `cluster.fork([env])`

<!-- YAML
added: v0.6.0
-->

* `env` {Object} 要添加到工作进程环境中的键/值对。
* 返回：{cluster.Worker}

生成一个新的工作进程。

这只能从主进程调用。

## `cluster.isMaster`

<!-- YAML
added: v0.8.1
deprecated: v16.0.0
-->

> Stability: 0 - Deprecated

[`cluster.isPrimary`][] 的已弃用别名。

## `cluster.isPrimary`

<!-- YAML
added: v16.0.0
-->

* 类型：{boolean}

如果进程是主进程，则为 true。这是由 `process.env.NODE_UNIQUE_ID` 决定的。如果 `process.env.NODE_UNIQUE_ID` 未定义，则 `isPrimary` 为 `true`。

## `cluster.isWorker`

<!-- YAML
added: v0.6.0
-->

* 类型：{boolean}

如果进程不是主进程（它是 `cluster.isPrimary` 的否定），则为 true。

## `cluster.schedulingPolicy`

<!-- YAML
added: v0.11.2
-->

调度策略，可以是用于轮询的 `cluster.SCHED_RR`，或者是留给操作系统的 `cluster.SCHED_NONE`。这是一个全局设置，并且在第一个工作进程生成或调用 [`.setupPrimary()`][] 时（以先发生者为准）有效冻结。

除 Windows 外，所有操作系统上默认都是 `SCHED_RR`。一旦 libuv 能够有效地分发 IOCP 句柄而不会导致大的性能损失，Windows 将更改为 `SCHED_RR`。

`cluster.schedulingPolicy` 也可以通过 `NODE_CLUSTER_SCHED_POLICY` 环境变量设置。有效值为 `'rr'` 和 `'none'`。

## `cluster.settings`

<!-- YAML
added: v0.7.1
changes:
  - version:
     - v13.2.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30162
    description: The `serialization` option is supported now.
  - version: v9.5.0
    pr-url: https://github.com/nodejs/node/pull/18399
    description: The `cwd` option is supported now.
  - version: v9.4.0
    pr-url: https://github.com/nodejs/node/pull/17412
    description: The `windowsHide` option is supported now.
  - version: v8.2.0
    pr-url: https://github.com/nodejs/node/pull/14140
    description: The `inspectPort` option is supported now.
  - version: v6.4.0
    pr-url: https://github.com/nodejs/node/pull/7838
    description: The `stdio` option is supported now.
-->

* 类型：{Object}
  * `execArgv` {string\[]} 传递给 Node.js 可执行文件的字符串参数列表。**默认值:** `process.execArgv`。
  * `exec` {string} 工作进程文件的路径。**默认值:** `process.argv[1]`。
  * `args` {string\[]} 传递给工作进程的字符串参数。**默认值:** `process.argv.slice(2)`。
  * `cwd` {string} 工作进程的当前工作目录。**默认值:** `undefined`（继承自父进程）。
  * `serialization` {string} 指定用于在进程之间发送消息的序列化类型。可能的值是 `'json'` 和 `'advanced'`。有关更多详细信息，请参阅 [Advanced serialization for `child_process`][]。**默认值:** `false`。
  * `silent` {boolean} 是否将输出发送到父进程的 stdio。**默认值:** `false`。
  * `stdio` {Array} 配置 fork 进程的 stdio。因为 cluster 模块依赖 IPC 来工作，此配置必须包含一个 `'ipc'` 条目。当提供此选项时，它会覆盖 `silent`。参见 [`child_process.spawn()`][] 的 [`stdio`][]。
  * `uid` {number} 设置进程的用户标识。（参见 setuid(2)。）
  * `gid` {number} 设置进程的组标识。（参见 setgid(2)。）
  * `inspectPort` {number|Function} 设置工作进程的检查器端口。这可以是一个数字，也可以是一个不带参数并返回数字的函数。默认情况下，每个工作进程获取自己的端口，从主进程的 `process.debugPort` 递增。
  * `windowsHide` {boolean} 隐藏通常在 Windows 系统上创建的 fork 进程的控制台窗口。**默认值:** `false`。

在调用 [`.setupPrimary()`][]（或 [`.fork()`][]）之后，此设置对象将包含设置，包括默认值。

此对象不打算手动更改或设置。

## `cluster.setupMaster([settings])`

<!-- YAML
added: v0.7.1
deprecated: v16.0.0
changes:
  - version: v6.4.0
    pr-url: https://github.com/nodejs/node/pull/7838
    description: The `stdio` option is supported now.
-->

> Stability: 0 - Deprecated

[`.setupPrimary()`][] 的已弃用别名。

## `cluster.setupPrimary([settings])`

<!-- YAML
added: v16.0.0
-->

* `settings` {Object} 参见 [`cluster.settings`][]。

`setupPrimary` 用于更改默认的 'fork' 行为。一旦调用，设置将出现在 `cluster.settings` 中。

任何设置更改仅影响未来对 [`.fork()`][] 的调用，对已运行的工作进程没有影响。

工作进程的唯一不能通过 `.setupPrimary()` 设置的属性是传递给 [`.fork()`][] 的 `env`。

上面的默认值仅适用于第一次调用；后续调用的默认值是调用 `cluster.setupPrimary()` 时的当前值。

```mjs
import cluster from 'node:cluster';

cluster.setupPrimary({
  exec: 'worker.js',
  args: ['--use', 'https'],
  silent: true,
});
cluster.fork(); // https worker
cluster.setupPrimary({
  exec: 'worker.js',
  args: ['--use', 'http'],
});
cluster.fork(); // http worker
```

```cjs
const cluster = require('node:cluster');

cluster.setupPrimary({
  exec: 'worker.js',
  args: ['--use', 'https'],
  silent: true,
});
cluster.fork(); // https worker
cluster.setupPrimary({
  exec: 'worker.js',
  args: ['--use', 'http'],
});
cluster.fork(); // http worker
```

这只能从主进程调用。

## `cluster.worker`

<!-- YAML
added: v0.7.0
-->

* 类型：{Object}

对当前工作进程对象的引用。在主进程中不可用。

```mjs
import cluster from 'node:cluster';

if (cluster.isPrimary) {
  console.log('I am primary');
  cluster.fork();
  cluster.fork();
} else if (cluster.isWorker) {
  console.log(`I am worker #${cluster.worker.id}`);
}
```

```cjs
const cluster = require('node:cluster');

if (cluster.isPrimary) {
  console.log('I am primary');
  cluster.fork();
  cluster.fork();
} else if (cluster.isWorker) {
  console.log(`I am worker #${cluster.worker.id}`);
}
```

## `cluster.workers`

<!-- YAML
added: v0.7.0
-->

* 类型：{Object}

一个存储活动工作进程对象的哈希，以 `id` 字段为键。这使得遍历所有工作进程变得容易。它仅在主进程中可用。

在工作进程断开连接*并*退出后，会从 `cluster.workers` 中移除工作进程。这两个事件之间的顺序无法提前确定。但是，可以保证从 `cluster.workers` 列表中移除的操作发生在最后一个 `'disconnect'` 或 `'exit'` 事件发出之前。

```mjs
import cluster from 'node:cluster';

for (const worker of Object.values(cluster.workers)) {
  worker.send('big announcement to all workers');
}
```

```cjs
const cluster = require('node:cluster');

for (const worker of Object.values(cluster.workers)) {
  worker.send('big announcement to all workers');
}
```

[Advanced serialization for `child_process`]: child_process.md#advanced-serialization
[Child Process module]: child_process.md#child_processforkmodulepath-args-options
[`.fork()`]: #clusterforkenv
[`.setupPrimary()`]: #clustersetupprimarysettings
[`ChildProcess.send()`]: child_process.md#subprocesssendmessage-sendhandle-options-callback
[`child_process.fork()`]: child_process.md#child_processforkmodulepath-args-options
[`child_process.spawn()`]: child_process.md#child_processspawncommand-args-options
[`child_process` event: `'exit'`]: child_process.md#event-exit
[`child_process` event: `'message'`]: child_process.md#event-message
[`cluster.isPrimary`]: #clusterisprimary
[`cluster.settings`]: #clustersettings
[`disconnect()`]: child_process.md#subprocessdisconnect
[`kill()`]: process.md#processkillpid-signal
[`process` event: `'message'`]: process.md#event-message
[`server.close()`]: net.md#event-close
[`stdio`]: child_process.md#optionsstdio
[`worker.exitedAfterDisconnect`]: #workerexitedafterdisconnect
[`worker_threads`]: worker_threads.md