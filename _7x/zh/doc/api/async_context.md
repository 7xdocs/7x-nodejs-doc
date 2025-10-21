# 异步上下文追踪

<!--introduced_in=v16.4.0-->

> Stability: 2 - Stable

<!-- source_link=lib/async_hooks.js -->

## 简介

这些类用于关联状态并在回调和 Promise 链中传播它。
它们允许在整个 Web 请求或任何其他异步持续时间的内存周期内存储数据。
这类似于其他语言中的线程本地存储。

`AsyncLocalStorage` 和 `AsyncResource` 类是 `node:async_hooks` 模块的一部分：

```mjs
import { AsyncLocalStorage, AsyncResource } from 'node:async_hooks';
```

```cjs
const { AsyncLocalStorage, AsyncResource } = require('node:async_hooks');
```

## 类：`AsyncLocalStorage`

<!-- YAML
added:
 - v13.10.0
 - v12.17.0
changes:
 - version: v16.4.0
   pr-url: https://github.com/nodejs/node/pull/37675
   description: AsyncLocalStorage 现已稳定。之前它是实验性的。
-->

该类创建的存储空间在异步操作期间保持一致性。

虽然你可以在 `node:async_hooks` 模块之上创建自己的实现，但应优先使用 `AsyncLocalStorage`，因为它是一个高性能且内存安全的实现，包含了许多非显而易见的重大优化。

以下示例使用 `AsyncLocalStorage` 构建一个简单的日志记录器，该记录器为传入的 HTTP 请求分配 ID，并在每个请求内记录的消息中包含这些 ID。

```mjs
import http from 'node:http';
import { AsyncLocalStorage } from 'node:async_hooks';

const asyncLocalStorage = new AsyncLocalStorage();

function logWithId(msg) {
  const id = asyncLocalStorage.getStore();
  console.log(`${id !== undefined ? id : '-'}:`, msg);
}

let idSeq = 0;
http.createServer((req, res) => {
  asyncLocalStorage.run(idSeq++, () => {
    logWithId('start');
    // 想象这里有任何异步操作链
    setImmediate(() => {
      logWithId('finish');
      res.end();
    });
  });
}).listen(8080);

http.get('http://localhost:8080');
http.get('http://localhost:8080');
// 打印：
//   0: start
//   0: finish
//   1: start
//   1: finish
```

```cjs
const http = require('node:http');
const { AsyncLocalStorage } = require('node:async_hooks');

const asyncLocalStorage = new AsyncLocalStorage();

function logWithId(msg) {
  const id = asyncLocalStorage.getStore();
  console.log(`${id !== undefined ? id : '-'}:`, msg);
}

let idSeq = 0;
http.createServer((req, res) => {
  asyncLocalStorage.run(idSeq++, () => {
    logWithId('start');
    // 想象这里有任何异步操作链
    setImmediate(() => {
      logWithId('finish');
      res.end();
    });
  });
}).listen(8080);

http.get('http://localhost:8080');
http.get('http://localhost:8080');
// 打印：
//   0: start
//   0: finish
//   1: start
//   1: finish
```

每个 `AsyncLocalStorage` 实例维护一个独立的存储上下文。
多个实例可以安全地同时存在，而不会相互干扰数据的风险。

### `new AsyncLocalStorage([options])`

<!-- YAML
added:
 - v13.10.0
 - v12.17.0
changes:
 - version: v24.0.0
   pr-url: https://github.com/nodejs/node/pull/57766
   description: 添加了 `defaultValue` 和 `name` 选项。
 - version:
    - v19.7.0
    - v18.16.0
   pr-url: https://github.com/nodejs/node/pull/46386
   description: 移除了实验性的 onPropagate 选项。
 - version:
    - v19.2.0
    - v18.13.0
   pr-url: https://github.com/nodejs/node/pull/45386
   description: 添加了 onPropagate 选项。
-->

* `options` {Object}
  * `defaultValue` {any} 当没有提供存储时使用的默认值。
  * `name` {string} `AsyncLocalStorage` 值的名称。

创建一个新的 `AsyncLocalStorage` 实例。存储仅在 `run()` 调用内或 `enterWith()` 调用之后提供。

### 静态方法：`AsyncLocalStorage.bind(fn)`

<!-- YAML
added:
 - v19.8.0
 - v18.16.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: 标记该 API 为稳定。
-->

* `fn` {Function} 要绑定到当前执行上下文的函数。
* 返回: {Function} 一个在捕获的执行上下文中调用 `fn` 的新函数。

将给定函数绑定到当前执行上下文。

### 静态方法：`AsyncLocalStorage.snapshot()`

<!-- YAML
added:
 - v19.8.0
 - v18.16.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: 标记该 API 为稳定。
-->

* 返回: {Function} 一个具有签名 `(fn: (...args) : R, ...args) : R` 的新函数。

捕获当前执行上下文并返回一个接受函数作为参数的函数。每当调用返回的函数时，它会在捕获的上下文中调用传递给它的函数。

```js
const asyncLocalStorage = new AsyncLocalStorage();
const runInAsyncScope = asyncLocalStorage.run(123, () => AsyncLocalStorage.snapshot());
const result = asyncLocalStorage.run(321, () => runInAsyncScope(() => asyncLocalStorage.getStore()));
console.log(result);  // 返回 123
```

对于简单的异步上下文追踪目的，`AsyncLocalStorage.snapshot()` 可以替代 `AsyncResource` 的使用，例如：

```js
class Foo {
  #runInAsyncScope = AsyncLocalStorage.snapshot();

  get() { return this.#runInAsyncScope(() => asyncLocalStorage.getStore()); }
}

const foo = asyncLocalStorage.run(123, () => new Foo());
console.log(asyncLocalStorage.run(321, () => foo.get())); // 返回 123
```

### `asyncLocalStorage.disable()`

<!-- YAML
added:
 - v13.10.0
 - v12.17.0
-->

> Stability: 1 - Experimental

禁用 `AsyncLocalStorage` 实例。所有后续对 `asyncLocalStorage.getStore()` 的调用将返回 `undefined`，直到再次调用 `asyncLocalStorage.run()` 或 `asyncLocalStorage.enterWith()`。

当调用 `asyncLocalStorage.disable()` 时，所有链接到该实例的当前上下文都将被退出。

在 `asyncLocalStorage` 可以被垃圾回收之前，需要调用 `asyncLocalStorage.disable()`。这不适用于由 `asyncLocalStorage` 提供的存储，因为这些对象会随着相应的异步资源一起被垃圾回收。

当 `asyncLocalStorage` 在当前进程中不再使用时，请使用此方法。

### `asyncLocalStorage.getStore()`

<!-- YAML
added:
 - v13.10.0
 - v12.17.0
-->

* 返回: {any}

返回当前存储。
如果在通过调用 `asyncLocalStorage.run()` 或 `asyncLocalStorage.enterWith()` 初始化的异步上下文之外调用，则返回 `undefined`。

### `asyncLocalStorage.enterWith(store)`

<!-- YAML
added:
 - v13.11.0
 - v12.17.0
-->

> Stability: 1 - Experimental

* `store` {any}

在剩余的当前同步执行期间进入上下文，然后通过任何后续的异步调用持久化存储。

示例：

```js
const store = { id: 1 };
// 用给定的存储对象替换之前的存储
asyncLocalStorage.enterWith(store);
asyncLocalStorage.getStore(); // 返回存储对象
someAsyncOperation(() => {
  asyncLocalStorage.getStore(); // 返回相同的对象
});
```

此转换将持续整个_同步_执行。
这意味着，例如，如果在事件处理程序中进入上下文，则后续的事件处理程序也将在该上下文中运行，除非使用 `AsyncResource` 显式绑定到另一个上下文。这就是为什么除非有强烈理由使用后者方法，否则应优先使用 `run()` 而不是 `enterWith()`。

```js
const store = { id: 1 };

emitter.on('my-event', () => {
  asyncLocalStorage.enterWith(store);
});
emitter.on('my-event', () => {
  asyncLocalStorage.getStore(); // 返回相同的对象
});

asyncLocalStorage.getStore(); // 返回 undefined
emitter.emit('my-event');
asyncLocalStorage.getStore(); // 返回相同的对象
```

### `asyncLocalStorage.name`

<!-- YAML
added: v24.0.0
-->

* 类型: {string}

如果提供了，则为 `AsyncLocalStorage` 实例的名称。

### `asyncLocalStorage.run(store, callback[, ...args])`

<!-- YAML
added:
 - v13.10.0
 - v12.17.0
-->

* `store` {any}
* `callback` {Function}
* `...args` {any}

在上下文中同步运行一个函数并返回其返回值。存储在该回调函数外部不可访问。存储对该回调函数内创建的任何异步操作都是可访问的。

可选的 `args` 会传递给回调函数。

如果回调函数抛出错误，该错误也会被 `run()` 抛出。堆栈跟踪不受此调用的影响，并且上下文会退出。

示例：

```js
const store = { id: 2 };
try {
  asyncLocalStorage.run(store, () => {
    asyncLocalStorage.getStore(); // 返回存储对象
    setTimeout(() => {
      asyncLocalStorage.getStore(); // 返回存储对象
    }, 200);
    throw new Error();
  });
} catch (e) {
  asyncLocalStorage.getStore(); // 返回 undefined
  // 错误将在这里被捕获
}
```

### `asyncLocalStorage.exit(callback[, ...args])`

<!-- YAML
added:
 - v13.10.0
 - v12.17.0
-->

> Stability: 1 - Experimental

* `callback` {Function}
* `...args` {any}

在上下文外部同步运行一个函数并返回其返回值。存储在该回调函数内部或该回调函数内创建的异步操作中不可访问。在回调函数内进行的任何 `getStore()` 调用都将始终返回 `undefined`。

可选的 `args` 会传递给回调函数。

如果回调函数抛出错误，该错误也会被 `exit()` 抛出。堆栈跟踪不受此调用的影响，并且上下文会重新进入。

示例：

```js
// 在 run 调用内部
try {
  asyncLocalStorage.getStore(); // 返回存储对象或值
  asyncLocalStorage.exit(() => {
    asyncLocalStorage.getStore(); // 返回 undefined
    throw new Error();
  });
} catch (e) {
  asyncLocalStorage.getStore(); // 返回相同的对象或值
  // 错误将在这里被捕获
}
```

### 与 `async/await` 一起使用

如果在异步函数中，只有一个 `await` 调用需要在上下文中运行，则应使用以下模式：

```js
async function fn() {
  await asyncLocalStorage.run(new Map(), () => {
    asyncLocalStorage.getStore().set('key', value);
    return foo(); // foo 的返回值将被 await
  });
}
```

在此示例中，存储仅在回调函数和 `foo` 调用的函数中可用。在 `run` 外部，调用 `getStore` 将返回 `undefined`。

### 故障排除：上下文丢失

在大多数情况下，`AsyncLocalStorage` 工作无误。在极少数情况下，当前存储在某个异步操作中丢失。

如果你的代码是基于回调的，使用 [`util.promisify()`][] 将其 Promise 化就足以使其开始与原生 Promise 一起工作。

如果你需要使用基于回调的 API 或者你的代码假设自定义的 thenable 实现，请使用 [`AsyncResource`][] 类将异步操作与正确的执行上下文关联起来。通过在你怀疑导致丢失的调用之后记录 `asyncLocalStorage.getStore()` 的内容来找到导致上下文丢失的函数调用。当代码记录 `undefined` 时，最后调用的回调可能是导致上下文丢失的原因。

## 类：`AsyncResource`

<!-- YAML
changes:
 - version: v16.4.0
   pr-url: https://github.com/nodejs/node/pull/37675
   description: AsyncResource 现已稳定。之前它是实验性的。
-->

`AsyncResource` 类旨在由嵌入器的异步资源扩展。使用它，用户可以轻松触发自己资源的生命周期事件。

当实例化 `AsyncResource` 时，将触发 `init` 钩子。

以下是 `AsyncResource` API 的概述。

```mjs
import { AsyncResource, executionAsyncId } from 'node:async_hooks';

// AsyncResource() 旨在被扩展。实例化一个新的 AsyncResource() 也会触发 init。如果省略 triggerAsyncId，则使用 async_hook.executionAsyncId()。
const asyncResource = new AsyncResource(
  type, { triggerAsyncId: executionAsyncId(), requireManualDestroy: false },
);

// 在资源的执行上下文中运行一个函数。这将：
// * 建立资源的上下文
// * 触发 AsyncHooks before 回调
// * 使用提供的参数调用提供的函数 `fn`
// * 触发 AsyncHooks after 回调
// * 恢复原始的执行上下文
asyncResource.runInAsyncScope(fn, thisArg, ...args);

// 调用 AsyncHooks destroy 回调。
asyncResource.emitDestroy();

// 返回分配给 AsyncResource 实例的唯一 ID。
asyncResource.asyncId();

// 返回 AsyncResource 实例的触发器 ID。
asyncResource.triggerAsyncId();
```

```cjs
const { AsyncResource, executionAsyncId } = require('node:async_hooks');

// AsyncResource() 旨在被扩展。实例化一个新的 AsyncResource() 也会触发 init。如果省略 triggerAsyncId，则使用 async_hook.executionAsyncId()。
const asyncResource = new AsyncResource(
  type, { triggerAsyncId: executionAsyncId(), requireManualDestroy: false },
);

// 在资源的执行上下文中运行一个函数。这将：
// * 建立资源的上下文
// * 触发 AsyncHooks before 回调
// * 使用提供的参数调用提供的函数 `fn`
// * 触发 AsyncHooks after 回调
// * 恢复原始的执行上下文
asyncResource.runInAsyncScope(fn, thisArg, ...args);

// 调用 AsyncHooks destroy 回调。
asyncResource.emitDestroy();

// 返回分配给 AsyncResource 实例的唯一 ID。
asyncResource.asyncId();

// 返回 AsyncResource 实例的触发器 ID。
asyncResource.triggerAsyncId();
```

### `new AsyncResource(type[, options])`

* `type` {string} 异步事件的类型。
* `options` {Object}
  * `triggerAsyncId` {number} 创建此异步事件的执行上下文的 ID。**默认值:** `executionAsyncId()`。
  * `requireManualDestroy` {boolean} 如果设置为 `true`，则在对象被垃圾回收时禁用 `emitDestroy`。这通常不需要设置（即使手动调用 `emitDestroy`），除非获取了资源的 `asyncId` 并使用敏感的 API 的 `emitDestroy` 调用它。当设置为 `false` 时，垃圾回收时的 `emitDestroy` 调用仅在有至少一个活跃的 `destroy` 钩子时发生。**默认值:** `false`。

用法示例：

```js
class DBQuery extends AsyncResource {
  constructor(db) {
    super('DBQuery');
    this.db = db;
  }

  getInfo(query, callback) {
    this.db.get(query, (err, data) => {
      this.runInAsyncScope(callback, null, err, data);
    });
  }

  close() {
    this.db = null;
    this.emitDestroy();
  }
}
```

### 静态方法：`AsyncResource.bind(fn[, type[, thisArg]])`

<!-- YAML
added:
  - v14.8.0
  - v12.19.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/46432
    description: 添加到绑定函数的 `asyncResource` 属性已被弃用，并将在未来版本中移除。
  - version:
    - v17.8.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/42177
    description: 当 `thisArg` 未定义时，默认改为使用调用者的 `this`。
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/36782
    description: 添加了可选的 thisArg。
-->

* `fn` {Function} 要绑定到当前执行上下文的函数。
* `type` {string} 与底层 `AsyncResource` 关联的可选名称。
* `thisArg` {any}

将给定函数绑定到当前执行上下文。

### `asyncResource.bind(fn[, thisArg])`

<!-- YAML
added:
  - v14.8.0
  - v12.19.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/46432
    description: 添加到绑定函数的 `asyncResource` 属性已被弃用，并将在未来版本中移除。
  - version:
    - v17.8.0
    - v16.15.0
    pr-url: https://github.com/nodejs/node/pull/42177
    description: 当 `thisArg` 未定义时，默认改为使用调用者的 `this`。
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/36782
    description: 添加了可选的 thisArg。
-->

* `fn` {Function} 要绑定到当前 `AsyncResource` 的函数。
* `thisArg` {any}

将给定函数绑定到此 `AsyncResource` 的作用域中执行。

### `asyncResource.runInAsyncScope(fn[, thisArg, ...args])`

<!-- YAML
added: v9.6.0
-->

* `fn` {Function} 要在此异步资源的执行上下文中调用的函数。
* `thisArg` {any} 用于函数调用的接收器。
* `...args` {any} 传递给函数的可选参数。

在异步资源的执行上下文中使用提供的参数调用提供的函数。这将建立上下文，触发 AsyncHooks before 回调，调用函数，触发 AsyncHooks after 回调，然后恢复原始的执行上下文。

### `asyncResource.emitDestroy()`

* 返回: {AsyncResource} 对 `asyncResource` 的引用。

调用所有 `destroy` 钩子。这应该只调用一次。如果调用超过一次，将抛出错误。这**必须**手动调用。如果资源留给 GC 回收，则 `destroy` 钩子将永远不会被调用。

### `asyncResource.asyncId()`

* 返回: {number} 分配给资源的唯一 `asyncId`。

### `asyncResource.triggerAsyncId()`

* 返回: {number} 传递给 `AsyncResource` 构造函数的相同 `triggerAsyncId`。

<a id="async-resource-worker-pool"></a>

### 使用 `AsyncResource` 实现 `Worker` 线程池

以下示例展示了如何使用 `AsyncResource` 类为 [`Worker`][] 池正确提供异步追踪。其他资源池，例如数据库连接池，可以遵循类似的模型。

假设任务是两个数字相加，使用名为 `task_processor.js` 的文件，内容如下：

```mjs
import { parentPort } from 'node:worker_threads';
parentPort.on('message', (task) => {
  parentPort.postMessage(task.a + task.b);
});
```

```cjs
const { parentPort } = require('node:worker_threads');
parentPort.on('message', (task) => {
  parentPort.postMessage(task.a + task.b);
});
```

围绕它的 Worker 池可以使用以下结构：

```mjs
import { AsyncResource } from 'node:async_hooks';
import { EventEmitter } from 'node:events';
import { Worker } from 'node:worker_threads';

const kTaskInfo = Symbol('kTaskInfo');
const kWorkerFreedEvent = Symbol('kWorkerFreedEvent');

class WorkerPoolTaskInfo extends AsyncResource {
  constructor(callback) {
    super('WorkerPoolTaskInfo');
    this.callback = callback;
  }

  done(err, result) {
    this.runInAsyncScope(this.callback, null, err, result);
    this.emitDestroy();  // `TaskInfo` 仅使用一次。
  }
}

export default class WorkerPool extends EventEmitter {
  constructor(numThreads) {
    super();
    this.numThreads = numThreads;
    this.workers = [];
    this.freeWorkers = [];
    this.tasks = [];

    for (let i = 0; i < numThreads; i++)
      this.addNewWorker();

    // 每当发出 kWorkerFreedEvent 时，调度队列中挂起的下一个任务（如果有）。
    this.on(kWorkerFreedEvent, () => {
      if (this.tasks.length > 0) {
        const { task, callback } = this.tasks.shift();
        this.runTask(task, callback);
      }
    });
  }

  addNewWorker() {
    const worker = new Worker(new URL('task_processor.js', import.meta.url));
    worker.on('message', (result) => {
      // 成功情况下：调用传递给 `runTask` 的回调，移除与 Worker 关联的 `TaskInfo`，并将其标记为空闲。
      worker[kTaskInfo].done(null, result);
      worker[kTaskInfo] = null;
      this.freeWorkers.push(worker);
      this.emit(kWorkerFreedEvent);
    });
    worker.on('error', (err) => {
      // 在未捕获异常的情况下：使用错误调用传递给 `runTask` 的回调。
      if (worker[kTaskInfo])
        worker[kTaskInfo].done(err, null);
      else
        this.emit('error', err);
      // 从列表中移除该 worker 并启动一个新的 Worker 来替换当前 worker。
      this.workers.splice(this.workers.indexOf(worker), 1);
      this.addNewWorker();
    });
    this.workers.push(worker);
    this.freeWorkers.push(worker);
    this.emit(kWorkerFreedEvent);
  }

  runTask(task, callback) {
    if (this.freeWorkers.length === 0) {
      // 没有空闲线程，等待一个工作线程空闲。
      this.tasks.push({ task, callback });
      return;
    }

    const worker = this.freeWorkers.pop();
    worker[kTaskInfo] = new WorkerPoolTaskInfo(callback);
    worker.postMessage(task);
  }

  close() {
    for (const worker of this.workers) worker.terminate();
  }
}
```

```cjs
const { AsyncResource } = require('node:async_hooks');
const { EventEmitter } = require('node:events');
const path = require('node:path');
const { Worker } = require('node:worker_threads');

const kTaskInfo = Symbol('kTaskInfo');
const kWorkerFreedEvent = Symbol('kWorkerFreedEvent');

class WorkerPoolTaskInfo extends AsyncResource {
  constructor(callback) {
    super('WorkerPoolTaskInfo');
    this.callback = callback;
  }

  done(err, result) {
    this.runInAsyncScope(this.callback, null, err, result);
    this.emitDestroy();  // `TaskInfo` 仅使用一次。
  }
}

class WorkerPool extends EventEmitter {
  constructor(numThreads) {
    super();
    this.numThreads = numThreads;
    this.workers = [];
    this.freeWorkers = [];
    this.tasks = [];

    for (let i = 0; i < numThreads; i++)
      this.addNewWorker();

    // 每当发出 kWorkerFreedEvent 时，调度队列中挂起的下一个任务（如果有）。
    this.on(kWorkerFreedEvent, () => {
      if (this.tasks.length > 0) {
        const { task, callback } = this.tasks.shift();
        this.runTask(task, callback);
      }
    });
  }

  addNewWorker() {
    const worker = new Worker(path.resolve(__dirname, 'task_processor.js'));
    worker.on('message', (result) => {
      // 成功情况下：调用传递给 `runTask` 的回调，移除与 Worker 关联的 `TaskInfo`，并将其标记为空闲。
      worker[kTaskInfo].done(null, result);
      worker[kTaskInfo] = null;
      this.freeWorkers.push(worker);
      this.emit(kWorkerFreedEvent);
    });
    worker.on('error', (err) => {
      // 在未捕获异常的情况下：使用错误调用传递给 `runTask` 的回调。
      if (worker[kTaskInfo])
        worker[kTaskInfo].done(err, null);
      else
        this.emit('error', err);
      // 从列表中移除该 worker 并启动一个新的 Worker 来替换当前 worker。
      this.workers.splice(this.workers.indexOf(worker), 1);
      this.addNewWorker();
    });
    this.workers.push(worker);
    this.freeWorkers.push(worker);
    this.emit(kWorkerFreedEvent);
  }

  runTask(task, callback) {
    if (this.freeWorkers.length === 0) {
      // 没有空闲线程，等待一个工作线程空闲。
      this.tasks.push({ task, callback });
      return;
    }

    const worker = this.freeWorkers.pop();
    worker[kTaskInfo] = new WorkerPoolTaskInfo(callback);
    worker.postMessage(task);
  }

  close() {
    for (const worker of this.workers) worker.terminate();
  }
}

module.exports = WorkerPool;
```

如果没有由 `WorkerPoolTaskInfo` 对象添加的显式追踪，回调看起来似乎与单个 `Worker` 对象关联。然而，`Worker` 的创建与任务的创建无关，并且不提供有关任务何时调度的信息。

该池的使用方式如下：

```mjs
import WorkerPool from './worker_pool.js';
import os from 'node:os';

const pool = new WorkerPool(os.availableParallelism());

let finished = 0;
for (let i = 0; i < 10; i++) {
  pool.runTask({ a: 42, b: 100 }, (err, result) => {
    console.log(i, err, result);
    if (++finished === 10)
      pool.close();
  });
}
```

```cjs
const WorkerPool = require('./worker_pool.js');
const os = require('node:os');

const pool = new WorkerPool(os.availableParallelism());

let finished = 0;
for (let i = 0; i < 10; i++) {
  pool.runTask({ a: 42, b: 100 }, (err, result) => {
    console.log(i, err, result);
    if (++finished === 10)
      pool.close();
  });
}
```

### 将 `AsyncResource` 与 `EventEmitter` 集成

由 [`EventEmitter`][] 触发的事件监听器可能在不同于调用 `eventEmitter.on()` 时的执行上下文中运行。

以下示例展示了如何使用 `AsyncResource` 类正确地将事件监听器与正确的执行上下文关联。相同的方法可以应用于 [`Stream`][] 或类似的事件驱动类。

```mjs
import { createServer } from 'node:http';
import { AsyncResource, executionAsyncId } from 'node:async_hooks';

const server = createServer((req, res) => {
  req.on('close', AsyncResource.bind(() => {
    // 执行上下文绑定到当前外部作用域。
  }));
  req.on('close', () => {
    // 执行上下文绑定到触发 'close' 发射的作用域。
  });
  res.end();
}).listen(3000);
```

```cjs
const { createServer } = require('node:http');
const { AsyncResource, executionAsyncId } = require('node:async_hooks');

const server = createServer((req, res) => {
  req.on('close', AsyncResource.bind(() => {
    // 执行上下文绑定到当前外部作用域。
  }));
  req.on('close', () => {
    // 执行上下文绑定到触发 'close' 发射的作用域。
  });
  res.end();
}).listen(3000);
```

[`AsyncResource`]: #class-asyncresource
[`EventEmitter`]: events.md#class-eventemitter
[`Stream`]: stream.md#stream
[`Worker`]: worker_threads.md#class-worker
[`util.promisify()`]: util.md#utilpromisifyoriginal