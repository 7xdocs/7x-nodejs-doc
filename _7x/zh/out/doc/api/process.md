[文件名称]: process.md
[文件内容开始]
# Process

<!-- introduced_in=v0.10.0 -->

<!-- type=global -->

<!-- source_link=lib/process.js -->

`process` 对象提供有关当前 Node.js 进程的信息并对其进行控制。

```mjs
import process from 'node:process';
```

```cjs
const process = require('node:process');
```

## Process 事件

`process` 对象是 [`EventEmitter`][] 的一个实例。

### 事件: `'beforeExit'`

<!-- YAML
added: v0.11.12
-->

当 Node.js 清空其事件循环并且没有额外的工作需要调度时，会发出 `'beforeExit'` 事件。通常，当没有安排工作时 Node.js 进程会退出，但是在 `'beforeExit'` 事件上注册的监听器可以进行异步调用，从而使 Node.js 进程继续。

监听器回调函数被调用时，会将 [`process.exitCode`][] 的值作为唯一参数传入。

对于导致显式终止的条件，例如调用 [`process.exit()`][] 或未捕获的异常，不会发出 `'beforeExit'` 事件。

除非意图是调度额外的工作，否则不应将 `'beforeExit'` 用作 `'exit'` 事件的替代方案。

```mjs
import process from 'node:process';

process.on('beforeExit', (code) => {
  console.log('Process beforeExit event with code: ', code);
});

process.on('exit', (code) => {
  console.log('Process exit event with code: ', code);
});

console.log('This message is displayed first.');

// 打印:
// This message is displayed first.
// Process beforeExit event with code: 0
// Process exit event with code: 0
```

```cjs
const process = require('node:process');

process.on('beforeExit', (code) => {
  console.log('Process beforeExit event with code: ', code);
});

process.on('exit', (code) => {
  console.log('Process exit event with code: ', code);
});

console.log('This message is displayed first.');

// 打印:
// This message is displayed first.
// Process beforeExit event with code: 0
// Process exit event with code: 0
```

### 事件: `'disconnect'`

<!-- YAML
added: v0.7.7
-->

如果 Node.js 进程是使用 IPC 通道生成的（请参阅 [子进程][] 和 [集群][] 文档），则当 IPC 通道关闭时，将发出 `'disconnect'` 事件。

### 事件: `'exit'`

<!-- YAML
added: v0.1.7
-->

* `code` {integer}

当 Node.js 进程由于以下原因之一即将退出时，会发出 `'exit'` 事件：

* 显式调用 `process.exit()` 方法；
* Node.js 事件循环不再需要执行任何额外的工作。

此时无法阻止事件循环的退出，并且一旦所有 `'exit'` 监听器运行完毕，Node.js 进程将终止。

监听器回调函数被调用时，会传入由 [`process.exitCode`][] 属性指定的退出码，或者传给 [`process.exit()`][] 方法的 `exitCode` 参数。

```mjs
import process from 'node:process';

process.on('exit', (code) => {
  console.log(`About to exit with code: ${code}`);
});
```

```cjs
const process = require('node:process');

process.on('exit', (code) => {
  console.log(`About to exit with code: ${code}`);
});
```

监听器函数**必须**只执行**同步**操作。Node.js 进程将在调用 `'exit'` 事件监听器后立即退出，导致事件循环中仍然排队的任何额外工作被丢弃。
例如，在以下示例中，超时将永远不会发生：

```mjs
import process from 'node:process';

process.on('exit', (code) => {
  setTimeout(() => {
    console.log('This will not run');
  }, 0);
});
```

```cjs
const process = require('node:process');

process.on('exit', (code) => {
  setTimeout(() => {
    console.log('This will not run');
  }, 0);
});
```

### 事件: `'message'`

<!-- YAML
added: v0.5.10
-->

* `message` { Object | boolean | number | string | null } 解析后的 JSON 对象或可序列化的原始值。
* `sendHandle` {net.Server|net.Socket} 一个 [`net.Server`][] 或 [`net.Socket`][] 对象，或 undefined。

如果 Node.js 进程是使用 IPC 通道生成的（请参阅 [子进程][] 和 [集群][] 文档），则当子进程收到父进程使用 [`childprocess.send()`][] 发送的消息时，会发出 `'message'` 事件。

消息会经过序列化和解析。结果消息可能与最初发送的消息不同。

如果在生成进程时将 `serialization` 选项设置为 `advanced`，则 `message` 参数可以包含 JSON 无法表示的数据。
有关更多详细信息，请参阅 [`child_process` 的高级序列化][]。

### 事件: `'multipleResolves'`

<!-- YAML
added: v10.12.0
deprecated:
  - v17.6.0
  - v16.15.0
-->

> Stability: 0 - Deprecated

* `type` {string} 决议类型。`'resolve'` 或 `'reject'` 之一。
* `promise` {Promise} 多次 resolved 或 rejected 的 promise。
* `value` {any} 在原始 resolve 之后，promise 被 resolve 或 reject 时所带的值。

每当一个 `Promise` 满足以下情况时，就会发出 `'multipleResolves'` 事件：

* 被 resolve 多次。
* 被 reject 多次。
* 在 resolve 之后被 reject。
* 在 reject 之后被 resolve。

这对于在使用 `Promise` 构造函数时跟踪应用程序中的潜在错误非常有用，因为多次决议会被静默吞掉。但是，此事件的发生并不一定表示错误。例如，[`Promise.race()`][] 可以触发 `'multipleResolves'` 事件。

由于在上述 [`Promise.race()`][] 示例等情况下该事件的不可靠性，它已被弃用。

```mjs
import process from 'node:process';

process.on('multipleResolves', (type, promise, reason) => {
  console.error(type, promise, reason);
  setImmediate(() => process.exit(1));
});

async function main() {
  try {
    return await new Promise((resolve, reject) => {
      resolve('First call');
      resolve('Swallowed resolve');
      reject(new Error('Swallowed reject'));
    });
  } catch {
    throw new Error('Failed');
  }
}

main().then(console.log);
// resolve: Promise { 'First call' } 'Swallowed resolve'
// reject: Promise { 'First call' } Error: Swallowed reject
//     at Promise (*)
//     at new Promise (<anonymous>)
//     at main (*)
// First call
```

```cjs
const process = require('node:process');

process.on('multipleResolves', (type, promise, reason) => {
  console.error(type, promise, reason);
  setImmediate(() => process.exit(1));
});

async function main() {
  try {
    return await new Promise((resolve, reject) => {
      resolve('First call');
      resolve('Swallowed resolve');
      reject(new Error('Swallowed reject'));
    });
  } catch {
    throw new Error('Failed');
  }
}

main().then(console.log);
// resolve: Promise { 'First call' } 'Swallowed resolve'
// reject: Promise { 'First call' } Error: Swallowed reject
//     at Promise (*)
//     at new Promise (<anonymous>)
//     at main (*)
// First call
```

### 事件: `'rejectionHandled'`

<!-- YAML
added: v1.4.1
-->

* `promise` {Promise} 被延迟处理的 promise。

每当一个 `Promise` 被 reject 并且错误处理程序（例如使用 [`promise.catch()`][]）附加到它的时间晚于 Node.js 事件循环的一个回合时，就会发出 `'rejectionHandled'` 事件。

该 `Promise` 对象之前应该在 `'unhandledRejection'` 事件中发出过，但在处理过程中获得了 rejection 处理程序。

对于 `Promise` 链，没有顶层概念可以在那里始终处理 rejections。由于其固有的异步性质，`Promise` rejection 可以在未来的某个时间点被处理，可能比发出 `'unhandledRejection'` 事件的事件循环回合要晚得多。

另一种说法是，与同步代码中存在不断增长的未处理异常列表不同，对于 Promise，可能存在一个增长和缩小的未处理 rejection 列表。

在同步代码中，当未处理异常列表增长时，会发出 `'uncaughtException'` 事件。

在异步代码中，当未处理 rejection 列表增长时，会发出 `'unhandledRejection'` 事件，而当未处理 rejection 列表缩小时，会发出 `'rejectionHandled'` 事件。

```mjs
import process from 'node:process';

const unhandledRejections = new Map();
process.on('unhandledRejection', (reason, promise) => {
  unhandledRejections.set(promise, reason);
});
process.on('rejectionHandled', (promise) => {
  unhandledRejections.delete(promise);
});
```

```cjs
const process = require('node:process');

const unhandledRejections = new Map();
process.on('unhandledRejection', (reason, promise) => {
  unhandledRejections.set(promise, reason);
});
process.on('rejectionHandled', (promise) => {
  unhandledRejections.delete(promise);
});
```

在这个例子中，`unhandledRejections` `Map` 会随着时间的推移而增长和缩小，反映了开始时未处理然后变为已处理的 rejections。可以定期（对于长时间运行的应用程序来说可能最好）或在进程退出时（对于脚本来说可能最方便）在错误日志中记录此类错误。

### 事件: `'workerMessage'`

<!-- YAML
added:
- v22.5.0
- v20.19.0
-->

* `value` {any} 使用 [`postMessageToThread()`][] 传输的值。
* `source` {number} 发送方工作线程的 ID，主线程为 `0`。

对于另一方使用 [`postMessageToThread()`][] 发送的任何传入消息，都会发出 `'workerMessage'` 事件。

### 事件: `'uncaughtException'`

<!-- YAML
added: v0.1.18
changes:
  - version:
     - v12.0.0
     - v10.17.0
    pr-url: https://github.com/nodejs/node/pull/26599
    description: Added the `origin` argument.
-->

* `err` {Error} 未捕获的异常。
* `origin` {string} 指示异常是源自未处理的 rejection 还是源自同步错误。可以是 `'uncaughtException'` 或 `'unhandledRejection'`。后者在基于 `Promise` 的异步上下文中发生异常（或者如果 `Promise` 被 reject）且 [`--unhandled-rejections`][] 标志设置为 `strict` 或 `throw`（这是默认值）并且 rejection 未被处理时使用，或者在命令行入口点的 ES 模块静态加载阶段发生 rejection 时使用。

当未捕获的 JavaScript 异常一直冒泡回到事件循环时，会发出 `'uncaughtException'` 事件。默认情况下，Node.js 通过将堆栈跟踪打印到 `stderr` 并以代码 1 退出来处理此类异常，覆盖任何先前设置的 [`process.exitCode`][]。
为 `'uncaughtException'` 事件添加处理程序会覆盖此默认行为。或者，在 `'uncaughtException'` 处理程序中更改 [`process.exitCode`][]，这将导致进程以提供的退出码退出。否则，在存在此类处理程序的情况下，进程将以 0 退出。

```mjs
import process from 'node:process';
import fs from 'node:fs';

process.on('uncaughtException', (err, origin) => {
  fs.writeSync(
    process.stderr.fd,
    `Caught exception: ${err}\n` +
    `Exception origin: ${origin}\n`,
  );
});

setTimeout(() => {
  console.log('This will still run.');
}, 500);

// 故意引发异常，但不捕获它。
nonexistentFunc();
console.log('This will not run.');
```

```cjs
const process = require('node:process');
const fs = require('node:fs');

process.on('uncaughtException', (err, origin) => {
  fs.writeSync(
    process.stderr.fd,
    `Caught exception: ${err}\n` +
    `Exception origin: ${origin}\n`,
  );
});

setTimeout(() => {
  console.log('This will still run.');
}, 500);

// 故意引发异常，但不捕获它。
nonexistentFunc();
console.log('This will not run.');
```

可以通过安装 `'uncaughtExceptionMonitor'` 监听器来监视 `'uncaughtException'` 事件，而不覆盖退出进程的默认行为。

#### 警告：正确使用 `'uncaughtException'`

`'uncaughtException'` 是一种粗糙的异常处理机制，仅打算作为最后手段使用。该事件*不应*用作 `On Error Resume Next` 的等效物。未处理的异常本质上意味着应用程序处于未定义状态。尝试在不从异常中正确恢复的情况下恢复应用程序代码可能导致额外的不可预见和不可预测的问题。

从事件处理程序中抛出的异常将不会被捕获。相反，进程将以非零退出码退出，并打印堆栈跟踪。这是为了避免无限递归。

在未捕获异常后尝试正常恢复类似于在升级计算机时拔掉电源线。十次中有九次，什么也不会发生。但第十次，系统会损坏。

`'uncaughtException'` 的正确用法是在关闭进程之前对已分配的资源（例如文件描述符、句柄等）执行同步清理。**在 `'uncaughtException'` 之后恢复正常操作是不安全的。**

为了以更可靠的方式重启崩溃的应用程序，无论是否发出 `'uncaughtException'`，都应在单独的进程中使用外部监视器来检测应用程序故障并根据需要恢复或重启。

### 事件: `'uncaughtExceptionMonitor'`

<!-- YAML
added:
 - v13.7.0
 - v12.17.0
-->

* `err` {Error} 未捕获的异常。
* `origin` {string} 指示异常是源自未处理的 rejection 还是源自同步错误。可以是 `'uncaughtException'` 或 `'unhandledRejection'`。后者在基于 `Promise` 的异步上下文中发生异常（或者如果 `Promise` 被 reject）且 [`--unhandled-rejections`][] 标志设置为 `strict` 或 `throw`（这是默认值）并且 rejection 未被处理时使用，或者在命令行入口点的 ES 模块静态加载阶段发生 rejection 时使用。

在发出 `'uncaughtException'` 事件或调用通过 [`process.setUncaughtExceptionCaptureCallback()`][] 安装的钩子之前，会发出 `'uncaughtExceptionMonitor'` 事件。

安装 `'uncaughtExceptionMonitor'` 监听器不会改变一旦发出 `'uncaughtException'` 事件时的行为。如果未安装 `'uncaughtException'` 监听器，进程仍然会崩溃。

```mjs
import process from 'node:process';

process.on('uncaughtExceptionMonitor', (err, origin) => {
  MyMonitoringTool.logSync(err, origin);
});

// 故意引发异常，但不捕获它。
nonexistentFunc();
// 仍然会使 Node.js 崩溃
```

```cjs
const process = require('node:process');

process.on('uncaughtExceptionMonitor', (err, origin) => {
  MyMonitoringTool.logSync(err, origin);
});

// 故意引发异常，但不捕获它。
nonexistentFunc();
// 仍然会使 Node.js 崩溃
```

### 事件: `'unhandledRejection'`

<!-- YAML
added: v1.4.1
changes:
  - version: v7.0.0
    pr-url: https://github.com/nodejs/node/pull/8217
    description: Not handling `Promise` rejections is deprecated.
  - version: v6.6.0
    pr-url: https://github.com/nodejs/node/pull/8223
    description: Unhandled `Promise` rejections will now emit
                 a process warning.
-->

* `reason` {Error|any} promise 被 reject 时传入的对象（通常是 [`Error`][] 对象）。
* `promise` {Promise} 被 reject 的 promise。

每当一个 `Promise` 被 reject 并且在事件循环的一个回合内没有错误处理程序附加到该 promise 时，就会发出 `'unhandledRejection'` 事件。在使用 Promise 编程时，异常被封装为 "rejected promises"。Rejections 可以使用 [`promise.catch()`][] 捕获和处理，并通过 `Promise` 链传播。`'unhandledRejection'` 事件对于检测和跟踪那些被 reject 且其 rejection 尚未处理的 promise 非常有用。

```mjs
import process from 'node:process';

process.on('unhandledRejection', (reason, promise) => {
  console.log('Unhandled Rejection at:', promise, 'reason:', reason);
  // 应用程序特定的日志记录、抛出错误或其他逻辑在这里
});

somePromise.then((res) => {
  return reportToUser(JSON.pasre(res)); // 注意拼写错误 (`pasre`)
}); // 没有 `.catch()` 或 `.then()`
```

```cjs
const process = require('node:process');

process.on('unhandledRejection', (reason, promise) => {
  console.log('Unhandled Rejection at:', promise, 'reason:', reason);
  // 应用程序特定的日志记录、抛出错误或其他逻辑在这里
});

somePromise.then((res) => {
  return reportToUser(JSON.pasre(res)); // 注意拼写错误 (`pasre`)
}); // 没有 `.catch()` 或 `.then()`
```

以下情况也会触发发出 `'unhandledRejection'` 事件：

```mjs
import process from 'node:process';

function SomeResource() {
  // 最初将加载状态设置为一个被 reject 的 promise
  this.loaded = Promise.reject(new Error('Resource not yet loaded!'));
}

const resource = new SomeResource();
// resource.loaded 上至少一个回合内没有 .catch 或 .then
```

```cjs
const process = require('node:process');

function SomeResource() {
  // 最初将加载状态设置为一个被 reject 的 promise
  this.loaded = Promise.reject(new Error('Resource not yet loaded!'));
}

const resource = new SomeResource();
// resource.loaded 上至少一个回合内没有 .catch 或 .then
```

在这个例子中，可以将 rejection 作为开发人员错误进行跟踪，这通常是其他 `'unhandledRejection'` 事件的情况。为了解决此类故障，可以将一个非操作性的 [`.catch(() => { })`][`promise.catch()`] 处理程序附加到 `resource.loaded`，这将防止发出 `'unhandledRejection'` 事件。

如果发出 `'unhandledRejection'` 事件但未处理，它将被作为未捕获的异常抛出。这种行为以及其他 `'unhandledRejection'` 事件的行为可以通过 [`--unhandled-rejections`][] 标志更改。

### 事件: `'warning'`

<!-- YAML
added: v6.0.0
-->

* `warning` {Error} 警告的关键属性包括：
  * `name` {string} 警告的名称。**默认值:** `'Warning'`。
  * `message` {string} 系统提供的警告描述。
  * `stack` {string} 代码中发出警告位置的堆栈跟踪。

每当 Node.js 发出进程警告时，就会发出 `'warning'` 事件。

进程警告类似于错误，因为它描述了引起用户注意的异常情况。但是，警告不是 Node.js 和 JavaScript 错误处理流程的正常组成部分。每当 Node.js 检测到可能导致次优应用程序性能、错误或安全漏洞的不良编码实践时，它都会发出警告。

```mjs
import process from 'node:process';

process.on('warning', (warning) => {
  console.warn(warning.name);    // 打印警告名称
  console.warn(warning.message); // 打印警告消息
  console.warn(warning.stack);   // 打印堆栈跟踪
});
```

```cjs
const process = require('node:process');

process.on('warning', (warning) => {
  console.warn(warning.name);    // 打印警告名称
  console.warn(warning.message); // 打印警告消息
  console.warn(warning.stack);   // 打印堆栈跟踪
});
```

默认情况下，Node.js 会将进程警告打印到 `stderr`。`--no-warnings` 命令行选项可用于抑制默认控制台输出，但 `process` 对象仍会发出 `'warning'` 事件。目前，除了弃用警告之外，还无法抑制特定的警告类型。要抑制弃用警告，请查看 [`--no-deprecation`][] 标志。

以下示例说明了当向事件添加了过多监听器时打印到 `stderr` 的警告：

```console
$ node
> events.defaultMaxListeners = 1;
> process.on('foo', () => {});
> process.on('foo', () => {});
> (node:38638) MaxListenersExceededWarning: Possible EventEmitter memory leak
detected. 2 foo listeners added. Use emitter.setMaxListeners() to increase limit
```

相反，以下示例关闭了默认警告输出，并向 `'warning'` 事件添加了自定义处理程序：

```console
$ node --no-warnings
> const p = process.on('warning', (warning) => console.warn('Do not do that!'));
> events.defaultMaxListeners = 1;
> process.on('foo', () => {});
> process.on('foo', () => {});
> Do not do that!
```

`--trace-warnings` 命令行选项可用于使警告的默认控制台输出包含警告的完整堆栈跟踪。

使用 `--throw-deprecation` 命令行标志启动 Node.js 将导致自定义弃用警告被作为异常抛出。

使用 `--trace-deprecation` 命令行标志将导致自定义弃用信息与堆栈跟踪一起打印到 `stderr`。

使用 `--no-deprecation` 命令行标志将抑制所有自定义弃用信息的报告。

`*-deprecation` 命令行标志仅影响使用名称 `'DeprecationWarning'` 的警告。

#### 发出自定义警告

请参阅 [`process.emitWarning()`][process_emit_warning] 方法以发出自定义或应用程序特定的警告。

#### Node.js 警告名称

Node.js 发出的警告类型（由 `name` 属性标识）没有严格的指导方针。可以随时添加新类型的警告。最常见的一些警告类型包括：

* `'DeprecationWarning'` - 表示使用了已弃用的 Node.js API 或功能。此类警告必须包含一个 `'code'` 属性来标识[弃用代码][]。
* `'ExperimentalWarning'` - 表示使用了实验性的 Node.js API 或功能。此类功能必须谨慎使用，因为它们可能随时更改，并且不受与受支持功能相同的严格语义版本控制和长期支持策略的约束。
* `'MaxListenersExceededWarning'` - 表示在 `EventEmitter` 或 `EventTarget` 上为给定事件注册了过多监听器。这通常是内存泄漏的迹象。
* `'TimeoutOverflowWarning'` - 表示向 `setTimeout()` 或 `setInterval()` 函数提供了无法容纳在 32 位有符号整数内的数值。
* `'TimeoutNegativeWarning'` - 表示向 `setTimeout()` 或 `setInterval()` 函数提供了负数。
* `'TimeoutNaNWarning'` - 表示向 `setTimeout()` 或 `setInterval()` 函数提供了非数字值。
* `'UnsupportedWarning'` - 表示使用了不受支持的选项或功能，该选项或功能将被忽略而不是被视为错误。一个例子是在使用 HTTP/2 兼容性 API 时使用 HTTP 响应状态消息。

### 事件: `'worker'`

<!-- YAML
added:
  - v16.2.0
  - v14.18.0
-->

* `worker` {Worker} 已创建的 {Worker}。

在创建新的 {Worker} 线程后，会发出 `'worker'` 事件。

### 信号事件

<!--type=event-->

<!--name=SIGINT, SIGHUP, etc.-->

当 Node.js 进程收到信号时，会发出信号事件。请参阅 signal(7) 以获取标准 POSIX 信号名称的列表，例如 `'SIGINT'`、`'SIGHUP'` 等。

信号在 [`Worker`][] 线程中不可用。

信号处理程序将接收信号的名称（`'SIGINT'`、`'SIGTERM'` 等）作为第一个参数。

每个事件的名称将是大写的信号通用名称（例如，对于 `SIGINT` 信号，事件名为 `'SIGINT'`）。

```mjs
import process from 'node:process';

// 开始从 stdin 读取，这样进程就不会退出。
process.stdin.resume();

process.on('SIGINT', () => {
  console.log('Received SIGINT. Press Control-D to exit.');
});

// 使用单个函数处理多个信号
function handle(signal) {
  console.log(`Received ${signal}`);
}

process.on('SIGINT', handle);
process.on('SIGTERM', handle);
```

```cjs
const process = require('node:process');

// 开始从 stdin 读取，这样进程就不会退出。
process.stdin.resume();

process.on('SIGINT', () => {
  console.log('Received SIGINT. Press Control-D to exit.');
});

// 使用单个函数处理多个信号
function handle(signal) {
  console.log(`Received ${signal}`);
}

process.on('SIGINT', handle);
process.on('SIGTERM', handle);
```

* `'SIGUSR1'` 由 Node.js 保留用于启动[调试器][]。可以安装监听器，但这样做可能会干扰调试器。
* `'SIGTERM'` 和 `'SIGINT'` 在非 Windows 平台上有默认处理程序，这些处理程序在退出前重置终端模式，退出码为 `128 + 信号编号`。如果这些信号之一安装了监听器，则其默认行为将被移除（Node.js 将不再退出）。
* `'SIGPIPE'` 默认被忽略。它可以安装监听器。
* `'SIGHUP'` 在 Windows 上当控制台窗口关闭时生成，在其他平台上的各种类似条件下也会生成。参见 signal(7)。它可以安装监听器，但是 Node.js 将在大约 10 秒后被 Windows 无条件终止。在非 Windows 平台上，`SIGHUP` 的默认行为是终止 Node.js，但一旦安装了监听器，其默认行为将被移除。
* `'SIGTERM'` 在 Windows 上不受支持，可以监听它。
* 来自终端的 `'SIGINT'` 在所有平台上都受支持，通常可以通过 <kbd>Ctrl</kbd>+<kbd>C</kbd> 生成（尽管这可能是可配置的）。当[终端原始模式][]启用并使用 <kbd>Ctrl</kbd>+<kbd>C</kbd> 时，不会生成它。
* `'SIGBREAK'` 在 Windows 上当按下 <kbd>Ctrl</kbd>+<kbd>Break</kbd> 时传递。在非 Windows 平台上，可以监听它，但无法发送或生成它。
* `'SIGWINCH'` 在控制台调整大小时传递。在 Windows 上，这只会发生在写入控制台且光标移动时，或者在原始模式下使用可读 tty 时。
* `'SIGKILL'` 无法安装监听器，它将在所有平台上无条件终止 Node.js。
* `'SIGSTOP'` 无法安装监听器。
* `'SIGBUS'`、`'SIGFPE'`、`'SIGSEGV'` 和 `'SIGILL'`，当不是使用 kill(2) 人为引发时，本质上会使进程处于无法安全调用 JS 监听器的状态。这样做可能导致进程停止响应。
* `0` 可以发送用于测试进程是否存在，如果进程存在则没有效果，但如果进程不存在则会抛出错误。

Windows 不支持信号，因此没有等同于通过信号终止的方式，但 Node.js 通过 [`process.kill()`][] 和 [`subprocess.kill()`][] 提供了一些模拟：

* 发送 `SIGINT`、`SIGTERM` 和 `SIGKILL` 将导致目标进程无条件终止，之后子进程将报告该进程被信号终止。
* 发送信号 `0` 可以作为跨平台独立方式测试进程是否存在。

## `process.abort()`

<!-- YAML
added: v0.7.0
-->

`process.abort()` 方法导致 Node.js 进程立即退出并生成核心文件。

此功能在 [`Worker`][] 线程中不可用。

## `process.allowedNodeEnvironmentFlags`

<!-- YAML
added: v10.10.0
-->

* 类型: {Set}

`process.allowedNodeEnvironmentFlags` 属性是一个特殊的、只读的 `Set`，包含 [`NODE_OPTIONS`][] 环境变量中允许的标志。

`process.allowedNodeEnvironmentFlags` 扩展了 `Set`，但覆盖了 `Set.prototype.has` 以识别几种不同的可能标志表示形式。在以下情况下，`process.allowedNodeEnvironmentFlags.has()` 将返回 `true`：

* 标志可以省略前导单 (`-`) 或双 (`--`) 破折号；例如，`inspect-brk` 代表 `--inspect-brk`，或 `r` 代表 `-r`。
* 传递给 V8 的标志（如 `--v8-options` 中所列）可以用一个或多个_非前导_下划线替换破折号，反之亦然；例如，`--perf_basic_prof`、`--perf-basic-prof`、`--perf_basic-prof` 等。
* 标志可以包含一个或多个等号 (`=`) 字符；第一个等号之后的所有字符（包括第一个等号）将被忽略；例如，`--stack-trace-limit=100`。
* 标志*必须*在 [`NODE_OPTIONS`][] 中允许。

当遍历 `process.allowedNodeEnvironmentFlags` 时，标志只会出现*一次*；每个标志将以一个或多个破折号开头。传递给 V8 的标志将包含下划线而不是非前导破折号：

```mjs
import { allowedNodeEnvironmentFlags } from 'node:process';

allowedNodeEnvironmentFlags.forEach((flag) => {
  // -r
  // --inspect-brk
  // --abort_on_uncaught_exception
  // ...
});
```

```cjs
const { allowedNodeEnvironmentFlags } = require('node:process');

allowedNodeEnvironmentFlags.forEach((flag) => {
  // -r
  // --inspect-brk
  // --abort_on_uncaught_exception
  // ...
});
```

`process.allowedNodeEnvironmentFlags` 的 `add()`、`clear()` 和 `delete()` 方法不执行任何操作，并且会静默失败。

如果 Node.js 编译时*没有* [`NODE_OPTIONS`][] 支持（在 [`process.config`][] 中显示），`process.allowedNodeEnvironmentFlags` 将包含*本来*允许的内容。

## `process.arch`

<!-- YAML
added: v0.5.0
-->

* 类型: {string}

运行 Node.js 二进制文件的操作系统 CPU 架构。可能的值有：`'arm'`、`'arm64'`、`'ia32'`、`'loong64'`、`'mips'`、`'mipsel'`、`'ppc64'`、`'riscv64'`、`'s390'`、`'s390x'` 和 `'x64'`。

```mjs
import { arch } from 'node:process';

console.log(`This processor architecture is ${arch}`);
```

```cjs
const { arch } = require('node:process');

console.log(`This processor architecture is ${arch}`);
```

## `process.argv`

<!-- YAML
added: v0.1.27
-->

* 类型: {string\[]}

`process.argv` 属性返回一个数组，其中包含启动 Node.js 进程时传递的命令行参数。第一个元素将是 [`process.execPath`][]。如果需要访问 `argv[0]` 的原始值，请参阅 `process.argv0`。第二个元素将是正在执行的 JavaScript 文件的路径。其余元素将是任何其他命令行参数。

例如，假设以下 `process-args.js` 脚本：

```mjs
import { argv } from 'node:process';

// 打印 process.argv
argv.forEach((val, index) => {
  console.log(`${index}: ${val}`);
});
```

```cjs
const { argv } = require('node:process');

// 打印 process.argv
argv.forEach((val, index) => {
  console.log(`${index}: ${val}`);
});
```

启动 Node.js 进程如下：

```bash
node process-args.js one two=three four
```

将生成输出：

```text
0: /usr/local/bin/node
1: /Users/mjr/work/node/process-args.js
2: one
3: two=three
4: four
```

## `process.argv0`

<!-- YAML
added: v6.4.0
-->

* 类型: {string}

`process.argv0` 属性存储了 Node.js 启动时传递的 `argv[0]` 原始值的只读副本。

```console
$ bash -c 'exec -a customArgv0 ./node'
> process.argv[0]
'/Volumes/code/external/node/out/Release/node'
> process.argv0
'customArgv0'
```

## `process.availableMemory()`

<!-- YAML
added:
  - v22.0.0
  - v20.13.0
changes:
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57765
    description: Change stability index for this feature from Experimental to Stable.
-->

* 类型: {number}

获取进程仍然可用的空闲内存量（以字节为单位）。

有关更多信息，请参阅 [`uv_get_available_memory`][uv_get_available_memory]。

## `process.channel`

<!-- YAML
added: v7.1.0
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/30165
    description: The object no longer accidentally exposes native C++ bindings.
-->

* 类型: {Object}

如果 Node.js 进程是使用 IPC 通道生成的（请参阅[子进程][]文档），则 `process.channel` 属性是对 IPC 通道的引用。如果不存在 IPC 通道，则此属性为 `undefined`。

### `process.channel.ref()`

<!-- YAML
added: v7.1.0
-->

如果之前调用过 `.unref()`，此方法会使 IPC 通道保持进程的事件循环运行。

通常，这是通过 `process` 对象上的 `'disconnect'` 和 `'message'` 监听器的数量来管理的。但是，此方法可用于显式请求特定行为。

### `process.channel.unref()`

<!-- YAML
added: v7.1.0
-->

此方法使 IPC 通道不保持进程的事件循环运行，并允许其在通道打开时完成。

通常，这是通过 `process` 对象上的 `'disconnect'` 和 `'message'` 监听器的数量来管理的。但是，此方法可用于显式请求特定行为。

## `process.chdir(directory)`

<!-- YAML
added: v0.1.17
-->

* `directory` {string}

`process.chdir()` 方法更改 Node.js 进程的当前工作目录，如果失败（例如，如果指定的 `directory` 不存在）则抛出异常。

```mjs
import { chdir, cwd } from 'node:process';

console.log(`Starting directory: ${cwd()}`);
try {
  chdir('/tmp');
  console.log(`New directory: ${cwd()}`);
} catch (err) {
  console.error(`chdir: ${err}`);
}
```

```cjs
const { chdir, cwd } = require('node:process');

console.log(`Starting directory: ${cwd()}`);
try {
  chdir('/tmp');
  console.log(`New directory: ${cwd()}`);
} catch (err) {
  console.error(`chdir: ${err}`);
}
```

此功能在 [`Worker`][] 线程中不可用。

## `process.config`

<!-- YAML
added: v0.7.7
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/43627
    description: The `process.config` object is now frozen.
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/36902
    description: Modifying process.config has been deprecated.
-->

* 类型: {Object}

`process.config` 属性返回一个冻结的 `Object`，其中包含用于编译当前 Node.js 可执行文件的配置选项的 JavaScript 表示形式。这与运行 `./configure` 脚本时生成的 `config.gypi` 文件相同。

可能的输出示例看起来像：

<!-- eslint-skip -->

```js
{
  target_defaults:
   { cflags: [],
     default_configuration: 'Release',
     defines: [],
     include_dirs: [],
     libraries: [] },
  variables:
   {
     host_arch: 'x64',
     napi_build_version: 5,
     node_install_npm: 'true',
     node_prefix: '',
     node_shared_cares: 'false',
     node_shared_http_parser: 'false',
     node_shared_libuv: 'false',
     node_shared_zlib: 'false',
     node_use_openssl: 'true',
     node_shared_openssl: 'false',
     target_arch: 'x64',
     v8_use_snapshot: 1
   }
}
```

## `process.connected`

<!-- YAML
added: v0.7.2
-->

* 类型: {boolean}

如果 Node.js 进程是使用 IPC 通道生成的（请参阅[子进程][]和[集群][]文档），只要 IPC 通道连接，`process.connected` 属性将返回 `true`，并在调用 `process.disconnect()` 后返回 `false`。

一旦 `process.connected` 为 `false`，就无法再使用 `process.send()` 通过 IPC 通道发送消息。

## `process.constrainedMemory()`

<!-- YAML
added:
  - v19.6.0
  - v18.15.0
changes:
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57765
    description: Change stability index for this feature from Experimental to Stable.
  - version:
    - v22.0.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52039
    description: Aligned return value with `uv_get_constrained_memory`.
-->

* 类型: {number}

根据操作系统施加的限制，获取进程可用的内存量（以字节为单位）。如果没有这样的约束，或者约束未知，则返回 `0`。

有关更多信息，请参阅 [`uv_get_constrained_memory`][uv_get_constrained_memory]。

## `process.cpuUsage([previousValue])`

<!-- YAML
added: v6.1.0
-->

* `previousValue` {Object} 之前调用 `process.cpuUsage()` 的返回值
* 返回: {Object}
  * `user` {integer}
  * `system` {integer}

`process.cpuUsage()` 方法返回当前进程的用户和系统 CPU 时间使用情况，在一个具有 `user` 和 `system` 属性的对象中，这些属性的值是以微秒（百万分之一秒）为单位的时间值。这些值分别测量在用户和系统代码中花费的时间，如果多个 CPU 核心为此进程执行工作，则可能最终大于实际经过的时间。

可以将之前调用 `process.cpuUsage()` 的结果作为参数传递给该函数，以获取差值读数。

```mjs
import { cpuUsage } from 'node:process';

const startUsage = cpuUsage();
// { user: 38579, system: 6986 }

// 让 CPU 旋转 500 毫秒
const now = Date.now();
while (Date.now() - now < 500);

console.log(cpuUsage(startUsage));
// { user: 514883, system: 11226 }
```

```cjs
const { cpuUsage } = require('node:process');

const startUsage = cpuUsage();
// { user: 38579, system: 6986 }

// 让 CPU 旋转 500 毫秒
const now = Date.now();
while (Date.now() - now < 500);

console.log(cpuUsage(startUsage));
// { user: 514883, system: 11226 }
```

## `process.cwd()`

<!-- YAML
added: v0.1.8
-->

* 返回: {string}

`process.cwd()` 方法返回 Node.js 进程的当前工作目录。

```mjs
import { cwd } from 'node:process';

console.log(`Current directory: ${cwd()}`);
```

```cjs
const { cwd } = require('node:process');

console.log(`Current directory: ${cwd()}`);
```

## `process.debugPort`

<!-- YAML
added: v0.7.2
-->

* 类型: {number}

启用时 Node.js 调试器使用的端口。

```mjs
import process from 'node:process';

process.debugPort = 5858;
```

```cjs
const process = require('node:process');

process.debugPort = 5858;
```

## `process.disconnect()`

<!-- YAML
added: v0.7.2
-->

如果 Node.js 进程是使用 IPC 通道生成的（请参阅[子进程][]和[集群][]文档），`process.disconnect()` 方法将关闭到父进程的 IPC 通道，允许子进程在没有任何其他连接保持其活动状态时正常退出。

调用 `process.disconnect()` 的效果与从父进程调用 [`ChildProcess.disconnect()`][] 相同。

如果 Node.js 进程不是使用 IPC 通道生成的，`process.disconnect()` 将是 `undefined`。

## `process.dlopen(module, filename[, flags])`

<!-- YAML
added: v0.1.16
changes:
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/12794
    description: Added support for the `flags` argument.
-->

* `module` {Object}
* `filename` {string}
* `flags` {os.constants.dlopen} **默认值:** `os.constants.dlopen.RTLD_LAZY`

`process.dlopen()` 方法允许动态加载共享对象。它主要由 `require()` 用于加载 C++ 插件，不应直接使用，除非在特殊情况下。换句话说，[`require()`][] 应优先于 `process.dlopen()`，除非有特定原因，例如自定义 dlopen 标志或从 ES 模块加载。

`flags` 参数是一个整数，允许指定 dlopen 行为。有关详细信息，请参阅 [`os.constants.dlopen`][] 文档。

调用 `process.dlopen()` 的一个重要要求是必须传递 `module` 实例。然后通过 `module.exports` 可以访问 C++ 插件导出的函数。

下面的示例显示了如何加载一个名为 `local.node` 的 C++ 插件，该插件导出一个 `foo` 函数。通过传递 `RTLD_NOW` 常量，所有符号在调用返回之前加载。在此示例中，假定该常量可用。

```mjs
import { dlopen } from 'node:process';
import { constants } from 'node:os';
import { fileURLToPath } from 'node:url';

const module = { exports: {} };
dlopen(module, fileURLToPath(new URL('local.node', import.meta.url)),
       constants.dlopen.RTLD_NOW);
module.exports.foo();
```

```cjs
const { dlopen } = require('node:process');
const { constants } = require('node:os');
const { join } = require('node:path');

const module = { exports: {} };
dlopen(module, join(__dirname, 'local.node'), constants.dlopen.RTLD_NOW);
module.exports.foo();
```

## `process.emitWarning(warning[, options])`

<!-- YAML
added: v8.0.0
-->

* `warning` {string|Error} 要发出的警告。
* `options` {Object}
  * `type` {string} 当 `warning` 是 `String` 时，`type` 是用于发出的警告*类型*的名称。**默认值:** `'Warning'`。
  * `code` {string} 正在发出的警告实例的唯一标识符。
  * `ctor` {Function} 当 `warning` 是 `String` 时，`ctor` 是一个可选函数，用于限制生成的堆栈跟踪。**默认值:** `process.emitWarning`。
  * `detail` {string} 要包含在错误中的附加文本。

`process.emitWarning()` 方法可用于发出自定义或应用程序特定的进程警告。可以通过向 [`'warning'`][process_warning] 事件添加处理程序来监听这些警告。

```mjs
import { emitWarning } from 'node:process';

// 发出带有代码和附加详细信息的警告。
emitWarning('Something happened!', {
  code: 'MY_WARNING',
  detail: 'This is some additional information',
});
// 发出:
// (node:56338) [MY_WARNING] Warning: Something happened!
// This is some additional information
```

```cjs
const { emitWarning } = require('node:process');

// 发出带有代码和附加详细信息的警告。
emitWarning('Something happened!', {
  code: 'MY_WARNING',
  detail: 'This is some additional information',
});
// 发出:
// (node:56338) [MY_WARNING] Warning: Something happened!
// This is some additional information
```

在此示例中，`process.emitWarning()` 在内部生成一个 `Error` 对象，并将其传递给 [`'warning'`][process_warning] 处理程序。

```mjs
import process from 'node:process';

process.on('warning', (warning) => {
  console.warn(warning.name);    // 'Warning'
  console.warn(warning.message); // 'Something happened!'
  console.warn(warning.code);    // 'MY_WARNING'
  console.warn(warning.stack);   // 堆栈跟踪
  console.warn(warning.detail);  // 'This is some additional information'
});
```

```cjs
const process = require('node:process');

process.on('warning', (warning) => {
  console.warn(warning.name);    // 'Warning'
  console.warn(warning.message); // 'Something happened!'
  console.warn(warning.code);    // 'MY_WARNING'
  console.warn(warning.stack);   // 堆栈跟踪
  console.warn(warning.detail);  // 'This is some additional information'
});
```

如果 `warning` 作为 `Error` 对象传递，则忽略 `options` 参数。

## `process.emitWarning(warning[, type[, code]][, ctor])`

<!-- YAML
added: v6.0.0
-->

* `warning` {string|Error} 要发出的警告。
* `type` {string} 当 `warning` 是 `String` 时，`type` 是用于发出的警告*类型*的名称。**默认值:** `'Warning'`。
* `code` {string} 正在发出的警告实例的唯一标识符。
* `ctor` {Function} 当 `warning` 是 `String` 时，`ctor` 是一个可选函数，用于限制生成的堆栈跟踪。**默认值:** `process.emitWarning`。

`process.emitWarning()` 方法可用于发出自定义或应用程序特定的进程警告。可以通过向 [`'warning'`][process_warning] 事件添加处理程序来监听这些警告。

```mjs
import { emitWarning } from 'node:process';

// 使用字符串发出警告。
emitWarning('Something happened!');
// 发出: (node: 56338) Warning: Something happened!
```

```cjs
const { emitWarning } = require('node:process');

// 使用字符串发出警告。
emitWarning('Something happened!');
// 发出: (node: 56338) Warning: Something happened!
```

```mjs
import { emitWarning } from 'node:process';

// 使用字符串和类型发出警告。
emitWarning('Something Happened!', 'CustomWarning');
// 发出: (node:56338) CustomWarning: Something Happened!
```

```cjs
const { emitWarning } = require('node:process');

// 使用字符串和类型发出警告。
emitWarning('Something Happened!', 'CustomWarning');
// 发出: (node:56338) CustomWarning: Something Happened!
```

```mjs
import { emitWarning } from 'node:process';

emitWarning('Something happened!', 'CustomWarning', 'WARN001');
// 发出: (node:56338) [WARN001] CustomWarning: Something happened!
```

```cjs
const { emitWarning } = require('node:process');

process.emitWarning('Something happened!', 'CustomWarning', 'WARN001');
// 发出: (node:56338) [WARN001] CustomWarning: Something happened!
```

在每个先前的示例中，`process.emitWarning()` 在内部生成一个 `Error` 对象，并将其传递给 [`'warning'`][process_warning] 处理程序。

```mjs
import process from 'node:process';

process.on('warning', (warning) => {
  console.warn(warning.name);
  console.warn(warning.message);
  console.warn(warning.code);
  console.warn(warning.stack);
});
```

```cjs
const process = require('node:process');

process.on('warning', (warning) => {
  console.warn(warning.name);
  console.warn(warning.message);
  console.warn(warning.code);
  console.warn(warning.stack);
});
```

如果 `warning` 作为 `Error` 对象传递，它将未经修改地传递给 `'warning'` 事件处理程序（可选的 `type`、`code` 和 `ctor` 参数将被忽略）：

```mjs
import { emitWarning } from 'node:process';

// 使用 Error 对象发出警告。
const myWarning = new Error('Something happened!');
// 使用 Error name 属性指定类型名称
myWarning.name = 'CustomWarning';
myWarning.code = 'WARN001';

emitWarning(myWarning);
// 发出: (node:56338) [WARN001] CustomWarning: Something happened!
```

```cjs
const { emitWarning } = require('node:process');

// 使用 Error 对象发出警告。
const myWarning = new Error('Something happened!');
// 使用 Error name 属性指定类型名称
myWarning.name = 'CustomWarning';
myWarning.code = 'WARN001';

emitWarning(myWarning);
// 发出: (node:56338) [WARN001] CustomWarning: Something happened!
```

如果 `warning` 不是字符串或 `Error` 对象，则抛出 `TypeError`。

虽然进程警告使用 `Error` 对象，但进程警告机制**不是**正常错误处理机制的替代品。

如果警告 `type` 为 `'DeprecationWarning'`，则实施以下附加处理：

* 如果使用了 `--throw-deprecation` 命令行标志，则弃用警告将作为异常抛出，而不是作为事件发出。
* 如果使用了 `--no-deprecation` 命令行标志，则弃用警告被抑制。
* 如果使用了 `--trace-deprecation` 命令行标志，则弃用警告将连同完整堆栈跟踪一起打印到 `stderr`。

### 避免重复警告

作为最佳实践，每个进程只应发出一次警告。为此，请将 `emitWarning()` 放在布尔值后面。

```mjs
import { emitWarning } from 'node:process';

function emitMyWarning() {
  if (!emitMyWarning.warned) {
    emitMyWarning.warned = true;
    emitWarning('Only warn once!');
  }
}
emitMyWarning();
// 发出: (node: 56339) Warning: Only warn once!
emitMyWarning();
// 不发出任何内容
```

```cjs
const { emitWarning } = require('node:process');

function emitMyWarning() {
  if (!emitMyWarning.warned) {
    emitMyWarning.warned = true;
    emitWarning('Only warn once!');
  }
}
emitMyWarning();
// 发出: (node: 56339) Warning: Only warn once!
emitMyWarning();
// 不发出任何内容
```

## `process.env`

<!-- YAML
added: v0.1.27
changes:
  - version: v11.14.0
    pr-url: https://github.com/nodejs/node/pull/26544
    description: Worker threads will now use a copy of the parent thread's
                 `process.env` by default, configurable through the `env`
                 option of the `Worker` constructor.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18990
    description: Implicit conversion of variable value to string is deprecated.
-->

* 类型: {Object}

`process.env` 属性返回包含用户环境的对象。参见 environ(7)。

此对象的示例看起来像：

<!-- eslint-skip -->

```js
{
  TERM: 'xterm-256color',
  SHELL: '/usr/local/bin/bash',
  USER: 'maciej',
  PATH: '~/.bin/:/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin',
  PWD: '/Users/maciej',
  EDITOR: 'vim',
  SHLVL: '1',
  HOME: '/Users/maciej',
  LOGNAME: 'maciej',
  _: '/usr/local/bin/node'
}
```

可以修改此对象，但此类修改不会反映到 Node.js 进程之外，或者（除非明确请求）到其他 [`Worker`][] 线程。
换句话说，以下示例将不起作用：

```bash
node -e 'process.env.foo = "bar"' && echo $foo
```

而以下示例会：

```mjs
import { env } from 'node:process';

env.foo = 'bar';
console.log(env.foo);
```

```cjs
const { env } = require('node:process');

env.foo = 'bar';
console.log(env.foo);
```

在 `process.env` 上分配属性会隐式地将值转换为字符串。**此行为已弃用。** 未来版本的 Node.js 可能在值不是字符串、数字或布尔值时抛出错误。

```mjs
import { env } from 'node:process';

env.test = null;
console.log(env.test);
// => 'null'
env.test = undefined;
console.log(env.test);
// => 'undefined'
```

```cjs
const { env } = require('node:process');

env.test = null;
console.log(env.test);
// => 'null'
env.test = undefined;
console.log(env.test);
// => 'undefined'
```

使用 `delete` 从 `process.env` 中删除属性。

```mjs
import { env } from 'node:process';

env.TEST = 1;
delete env.TEST;
console.log(env.TEST);
// => undefined
```

```cjs
const { env } = require('node:process');

env.TEST = 1;
delete env.TEST;
console.log(env.TEST);
// => undefined
```

在 Windows 操作系统上，环境变量不区分大小写。

```mjs
import { env } from 'node:process';

env.TEST = 1;
console.log(env.test);
// => 1
```

```cjs
const { env } = require('node:process');

env.TEST = 1;
console.log(env.test);
// => 1
```

除非在创建 [`Worker`][] 实例时明确指定，否则每个 [`Worker`][] 线程都有自己的 `process.env` 副本，基于其父线程的 `process.env`，或者作为 `env` 选项传递给 [`Worker`][] 构造函数的任何内容。对 `process.env` 的更改在 [`Worker`][] 线程之间不可见，只有主线程可以进行对操作系统或原生插件可见的更改。在 Windows 上，[`Worker`][] 实例上的 `process.env` 副本以区分大小写的方式运行，与主线程不同。

## `process.execArgv`

<!-- YAML
added: v0.7.7
-->

* 类型: {string\[]}

`process.execArgv` 属性返回在启动 Node.js 进程时传递的一组 Node.js 特定的命令行选项。这些选项不会出现在 [`process.argv`][] 属性返回的数组中，并且不包括 Node.js 可执行文件、脚本名称或脚本名称之后的任何选项。这些选项对于生成具有与父进程相同执行环境的子进程非常有用。

```bash
node --icu-data-dir=./foo --require ./bar.js script.js --version
```

结果在 `process.execArgv` 中：

```json
["--icu-data-dir=./foo", "--require", "./bar.js"]
```

而 `process.argv` 中：

<!-- eslint-disable @stylistic/js/semi -->

```js
['/usr/local/bin/node', 'script.js', '--version']
```

有关工作线程与此属性的详细行为，请参阅 [`Worker` 构造函数][]。

## `process.execPath`

<!-- YAML
added: v0.1.100
-->

* 类型: {string}

`process.execPath` 属性返回启动 Node.js 进程的可执行文件的绝对路径名。符号链接（如果有）会被解析。

<!-- eslint-disable @stylistic/js/semi -->

```js
'/usr/local/bin/node'
```

## `process.execve(file[, args[, env]])`

<!-- YAML
added:
  - v23.11.0
  - v22.15.0
-->

> Stability: 1 - Experimental

* `file` {string} 要运行的可执行文件的名称或路径。
* `args` {string\[]} 字符串参数列表。任何参数都不能包含空字节 (`\u0000`)。
* `env` {Object} 环境键值对。
  任何键或值都不能包含空字节 (`\u0000`)。
  **默认值:** `process.env`。

用新进程替换当前进程。

这是通过使用 `execve` POSIX 函数实现的，因此不会保留当前进程的任何内存或其他资源，除了标准输入、标准输出和标准错误文件描述符。

当进程被交换时，系统会丢弃所有其他资源，不会触发任何退出或关闭事件，也不会运行任何清理处理程序。

除非发生错误，否则此函数永远不会返回。

此函数在 Windows 或 IBM i 上不可用。

## `process.exit([code])`

<!-- YAML
added: v0.1.13
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/43716
    description: Only accepts a code of type number, or of type string if it
                 represents an integer.
-->

* `code` {integer|string|null|undefined} 退出码。对于字符串类型，只允许整数字符串（例如，'1'）。**默认值:** `0`。

`process.exit()` 方法指示 Node.js 以 `code` 的退出状态同步终止进程。如果省略 `code`，则退出使用 'success' 代码 `0` 或 `process.exitCode` 的值（如果已设置）。在所有 [`'exit'`][] 事件监听器被调用之前，Node.js 不会终止。

以 'failure' 代码退出：

```mjs
import { exit } from 'node:process';

exit(1);
```

```cjs
const { exit } = require('node:process');

exit(1);
```

执行 Node.js 的 shell 应将退出码视为 `1`。

调用 `process.exit()` 将强制进程尽快退出，即使仍有尚未完全完成的异步操作挂起，包括对 `process.stdout` 和 `process.stderr` 的 I/O 操作。

在大多数情况下，实际上不需要显式调用 `process.exit()`。Node.js 进程将在事件循环中没有额外工作挂起时自行退出。可以设置 `process.exitCode` 属性来告诉进程在正常退出时使用哪个退出码。

例如，以下示例说明了 _误用_ `process.exit()` 方法可能导致打印到 stdout 的数据被截断和丢失：

```mjs
import { exit } from 'node:process';

// 这是一个 *不要* 做的示例：
if (someConditionNotMet()) {
  printUsageToStdout();
  exit(1);
}
```

```cjs
const { exit } = require('node:process');

// 这是一个 *不要* 做的示例：
if (someConditionNotMet()) {
  printUsageToStdout();
  exit(1);
}
```

这是因为写入 `process.stdout` 在 Node.js 中有时是 _异步的_，并且可能发生在 Node.js 事件循环的多个时钟周期内。然而，调用 `process.exit()` 会强制进程在那些额外的写入到 `stdout` 能够执行之前退出。

与其直接调用 `process.exit()`，代码 _应该_ 设置 `process.exitCode` 并通过避免为事件循环调度任何额外工作来允许进程自然退出：

```mjs
import process from 'node:process';

// 如何正确设置退出码，同时让进程优雅退出。
if (someConditionNotMet()) {
  printUsageToStdout();
  process.exitCode = 1;
}
```

```cjs
const process = require('node:process');

// 如何正确设置退出码，同时让进程优雅退出。
if (someConditionNotMet()) {
  printUsageToStdout();
  process.exitCode = 1;
}
```

如果由于错误条件需要终止 Node.js 进程，抛出 _未捕获_ 错误并允许进程相应地终止比调用 `process.exit()` 更安全。

在 [`Worker`][] 线程中，此函数停止当前线程而不是当前进程。

## `process.exitCode`

<!-- YAML
added: v0.11.8
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/43716
    description: Only accepts a code of type number, or of type string if it
                 represents an integer.
-->

* 类型: {integer|string|null|undefined} 退出码。对于字符串类型，只允许整数字符串（例如，'1'）。**默认值:** `undefined`。

当进程要么优雅退出，要么通过 [`process.exit()`][] 退出而未指定代码时，将作为进程退出码的数字。

可以通过分配值给 `process.exitCode` 或通过传递参数给 [`process.exit()`][] 来更新 `process.exitCode` 的值：

```console
$ node -e 'process.exitCode = 9'; echo $?
9
$ node -e 'process.exit(42)'; echo $?
42
$ node -e 'process.exitCode = 9; process.exit(42)'; echo $?
42
```

当发生不可恢复的错误时，Node.js 也可以隐式地设置该值（例如，遇到未解决的顶级 await）。然而显式操作退出码总是优先于隐式操作：

```console
$ node --input-type=module -e 'await new Promise(() => {})'; echo $?
13
$ node --input-type=module -e 'process.exitCode = 9; await new Promise(() => {})'; echo $?
9
```

## `process.features.cached_builtins`

<!-- YAML
added: v12.0.0
-->

* 类型: {boolean}

如果当前 Node.js 构建缓存了内置模块，则为 `true` 的布尔值。

## `process.features.debug`

<!-- YAML
added: v0.5.5
-->

* 类型: {boolean}

如果当前 Node.js 构建是调试构建，则为 `true` 的布尔值。

## `process.features.inspector`

<!-- YAML
added: v11.10.0
-->

* 类型: {boolean}

如果当前 Node.js 构建包含检查器，则为 `true` 的布尔值。

## `process.features.ipv6`

<!-- YAML
added: v0.5.3
deprecated:
  - v23.4.0
  - v22.13.0
-->

> Stability: 0 - Deprecated. This property is always true, and any checks based on it are
> redundant.

* 类型: {boolean}

如果当前 Node.js 构建包含对 IPv6 的支持，则为 `true` 的布尔值。

由于所有 Node.js 构建都具有 IPv6 支持，此值始终为 `true`。

## `process.features.require_module`

<!-- YAML
added:
 - v23.0.0
 - v22.10.0
 - v20.19.0
-->

* 类型: {boolean}

如果当前 Node.js 构建支持[使用 `require()` 加载 ECMAScript 模块][]，则为 `true` 的布尔值。

## `process.features.tls`

<!-- YAML
added: v0.5.3
-->

* 类型: {boolean}

如果当前 Node.js 构建包含对 TLS 的支持，则为 `true` 的布尔值。

## `process.features.tls_alpn`

<!-- YAML
added: v4.8.0
deprecated:
  - v23.4.0
  - v22.13.0
-->

> Stability: 0 - Deprecated. Use `process.features.tls` instead.

* 类型: {boolean}

如果当前 Node.js 构建包含对 TLS 中 ALPN 的支持，则为 `true` 的布尔值。

在 Node.js 11.0.0 及更高版本中，OpenSSL 依赖项具有无条件的 ALPN 支持。因此，此值与 `process.features.tls` 的值相同。

## `process.features.tls_ocsp`

<!-- YAML
added: v0.11.13
deprecated:
  - v23.4.0
  - v22.13.0
-->

> Stability: 0 - Deprecated. Use `process.features.tls` instead.

* 类型: {boolean}

如果当前 Node.js 构建包含对 TLS 中 OCSP 的支持，则为 `true` 的布尔值。

在 Node.js 11.0.0 及更高版本中，OpenSSL 依赖项具有无条件的 OCSP 支持。因此，此值与 `process.features.tls` 的值相同。

## `process.features.tls_sni`

<!-- YAML
added: v0.5.3
deprecated:
  - v23.4.0
  - v22.13.0
-->

> Stability: 0 - Deprecated. Use `process.features.tls` instead.

* 类型: {boolean}

如果当前 Node.js 构建包含对 TLS 中 SNI 的支持，则为 `true` 的布尔值。

在 Node.js 11.0.0 及更高版本中，OpenSSL 依赖项具有无条件的 SNI 支持。因此，此值与 `process.features.tls` 的值相同。

## `process.features.typescript`

<!-- YAML
added:
 - v23.0.0
 - v22.10.0
-->

> Stability: 1.2 - Release candidate

* 类型: {boolean|string}

默认情况下为 `"strip"` 的值，如果 Node.js 使用 `--experimental-transform-types` 运行，则为 `"transform"`，如果 Node.js 使用 `--no-experimental-strip-types` 运行，则为 `false`。

## `process.features.uv`

<!-- YAML
added: v0.5.3
deprecated:
  - v23.4.0
  - v22.13.0
-->

> Stability: 0 - Deprecated. This property is always true, and any checks based on it are
> redundant.

* 类型: {boolean}

如果当前 Node.js 构建包含对 libuv 的支持，则为 `true` 的布尔值。

由于不可能在没有 libuv 的情况下构建 Node.js，此值始终为 `true`。

## `process.finalization.register(ref, callback)`

<!-- YAML
added: v22.5.0
-->

> Stability: 1.1 - Active Development

* `ref` {Object | Function} 正在被跟踪的资源的引用。
* `callback` {Function} 当资源被终结时要调用的回调函数。
  * `ref` {Object | Function} 正在被跟踪的资源的引用。
  * `event` {string} 触发终结的事件。默认为 'exit'。

如果 `ref` 对象在进程发出 `exit` 事件时未被垃圾回收，则此函数注册一个回调以被调用。如果对象 `ref` 在发出 `exit` 事件之前被垃圾回收，则回调将从终结注册表中移除，并且不会在进程退出时被调用。

在回调内部，你可以释放由 `ref` 对象分配的资源。请注意，应用于 `beforeExit` 事件的所有限制也适用于 `callback` 函数，这意味着在特殊情况下回调可能不会被调用。

这个函数的想法是帮助你在进程开始退出时释放资源，但如果对象不再被使用，也让对象被垃圾回收。

例如：你可以注册一个包含缓冲区的对象，你希望确保在进程退出时释放该缓冲区，但如果对象在进程退出之前被垃圾回收，我们就不再需要释放缓冲区，所以在这种情况下，我们只需从终结注册表中移除回调。

```cjs
const { finalization } = require('node:process');

// 请确保传递给 finalization.register() 的函数不会围绕不必要的对象创建闭包。
function onFinalize(obj, event) {
  // 你可以对对象做任何你想做的事情
  obj.dispose();
}

function setup() {
  // 这个对象可以被安全地垃圾回收，
  // 并且产生的关闭函数将不会被调用。
  // 没有泄漏。
  const myDisposableObject = {
    dispose() {
      // 同步释放你的资源
    },
  };

  finalization.register(myDisposableObject, onFinalize);
}

setup();
```

```mjs
import { finalization } from 'node:process';

// 请确保传递给 finalization.register() 的函数不会围绕不必要的对象创建闭包。
function onFinalize(obj, event) {
  // 你可以对对象做任何你想做的事情
  obj.dispose();
}

function setup() {
  // 这个对象可以被安全地垃圾回收，
  // 并且产生的关闭函数将不会被调用。
  // 没有泄漏。
  const myDisposableObject = {
    dispose() {
      // 同步释放你的资源
    },
  };

  finalization.register(myDisposableObject, onFinalize);
}

setup();
```

上面的代码依赖于以下假设：

* 避免使用箭头函数
* 建议常规函数在全局上下文（根）中

常规函数 _可能_ 引用 `obj` 所在的上下文，使得 `obj` 无法被垃圾回收。

箭头函数将持有先前的上下文。例如，考虑：

```js
class Test {
  constructor() {
    finalization.register(this, (ref) => ref.dispose());

    // 即使是这样的做法也强烈不鼓励
    // finalization.register(this, () => this.dispose());
  }
  dispose() {}
}
```

这个对象被垃圾回收的可能性很小（不是不可能），但如果没有被回收，当调用 `process.exit` 时，`dispose` 将被调用。

请小心，不要依赖此功能来处理关键资源的处置，因为不能保证在所有情况下都会调用回调。

## `process.finalization.registerBeforeExit(ref, callback)`

<!-- YAML
added: v22.5.0
-->

> Stability: 1.1 - Active Development

* `ref` {Object | Function} 正在被跟踪的资源的引用。
* `callback` {Function} 当资源被终结时要调用的回调函数。
  * `ref` {Object | Function} 正在被跟踪的资源的引用。
  * `event` {string} 触发终结的事件。默认为 'beforeExit'。

此函数的行为与 `register` 完全相同，只是如果 `ref` 对象未被垃圾回收，则当进程发出 `beforeExit` 事件时将调用回调。

请注意，应用于 `beforeExit` 事件的所有限制也适用于 `callback` 函数，这意味着在特殊情况下回调可能不会被调用。

## `process.finalization.unregister(ref)`

<!-- YAML
added: v22.5.0
-->

> Stability: 1.1 - Active Development

* `ref` {Object | Function} 先前注册的资源的引用。

此函数从终结注册表中移除对象的注册，因此回调将不再被调用。

```cjs
const { finalization } = require('node:process');

// 请确保传递给 finalization.register() 的函数不会围绕不必要的对象创建闭包。
function onFinalize(obj, event) {
  // 你可以对对象做任何你想做的事情
  obj.dispose();
}

function setup() {
  // 这个对象可以被安全地垃圾回收，
  // 并且产生的关闭函数将不会被调用。
  // 没有泄漏。
  const myDisposableObject = {
    dispose() {
      // 同步释放你的资源
    },
  };

  finalization.register(myDisposableObject, onFinalize);

  // 做一些事情

  myDisposableObject.dispose();
  finalization.unregister(myDisposableObject);
}

setup();
```

```mjs
import { finalization } from 'node:process';

// 请确保传递给 finalization.register() 的函数不会围绕不必要的对象创建闭包。
function onFinalize(obj, event) {
  // 你可以对对象做任何你想做的事情
  obj.dispose();
}

function setup() {
  // 这个对象可以被安全地垃圾回收，
  // 并且产生的关闭函数将不会被调用。
  // 没有泄漏。
  const myDisposableObject = {
    dispose() {
      // 同步释放你的资源
    },
  };

  // 请确保传递给 finalization.register() 的函数不会围绕不必要的对象创建闭包。
  function onFinalize(obj, event) {
    // 你可以对对象做任何你想做的事情
    obj.dispose();
  }

  finalization.register(myDisposableObject, onFinalize);

  // 做一些事情

  myDisposableObject.dispose();
  finalization.unregister(myDisposableObject);
}

setup();
```

## `process.getActiveResourcesInfo()`

<!-- YAML
added:
  - v17.3.0
  - v16.14.0
changes:
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57765
    description: Change stability index for this feature from Experimental to Stable.
-->

* 返回: {string\[]}

`process.getActiveResourcesInfo()` 方法返回一个字符串数组，包含当前保持事件循环活动的活动资源的类型。

```mjs
import { getActiveResourcesInfo } from 'node:process';
import { setTimeout } from 'node:timers';

console.log('Before:', getActiveResourcesInfo());
setTimeout(() => {}, 1000);
console.log('After:', getActiveResourcesInfo());
// 打印:
//   Before: [ 'CloseReq', 'TTYWrap', 'TTYWrap', 'TTYWrap' ]
//   After: [ 'CloseReq', 'TTYWrap', 'TTYWrap', 'TTYWrap', 'Timeout' ]
```

```cjs
const { getActiveResourcesInfo } = require('node:process');
const { setTimeout } = require('node:timers');

console.log('Before:', getActiveResourcesInfo());
setTimeout(() => {}, 1000);
console.log('After:', getActiveResourcesInfo());
// 打印:
//   Before: [ 'TTYWrap', 'TTYWrap', 'TTYWrap' ]
//   After: [ 'TTYWrap', 'TTYWrap', 'TTYWrap', 'Timeout' ]
```

## `process.getBuiltinModule(id)`

<!-- YAML
added:
- v22.3.0
- v20.16.0
-->

* `id` {string} 被请求的内置模块的 ID。
* 返回: {Object|undefined}

`process.getBuiltinModule(id)` 提供了一种在全局可用函数中加载内置模块的方法。需要支持其他环境的 ES 模块可以使用它在 Node.js 中运行时有条件地加载 Node.js 内置模块，而无需处理在非 Node.js 环境中 `import` 可能引发的解析错误，或者使用动态 `import()`，这要么将模块转换为异步模块，要么将同步 API 转换为异步 API。

```mjs
if (globalThis.process?.getBuiltinModule) {
  // 在 Node.js 中运行，使用 Node.js fs 模块。
  const fs = globalThis.process.getBuiltinModule('fs');
  // 如果需要 `require()` 来加载用户模块，使用 createRequire()
  const module = globalThis.process.getBuiltinModule('module');
  const require = module.createRequire(import.meta.url);
  const foo = require('foo');
}
```

如果 `id` 指定了当前 Node.js 进程中可用的内置模块，`process.getBuiltinModule(id)` 方法返回相应的内置模块。如果 `id` 不对应任何内置模块，则返回 `undefined`。

`process.getBuiltinModule(id)` 接受 [`module.isBuiltin(id)`][] 识别的内置模块 ID。一些内置模块必须使用 `node:` 前缀加载，请参阅[带有强制 `node:` 前缀的内置模块][]。
`process.getBuiltinModule(id)` 返回的引用总是指向对应于 `id` 的内置模块，即使用户修改了 [`require.cache`][] 使得 `require(id)` 返回其他内容。

## `process.getegid()`

<!-- YAML
added: v2.0.0
-->

`process.getegid()` 方法返回 Node.js 进程的数字有效组标识。（参见 getegid(2)。）

```mjs
import process from 'node:process';

if (process.getegid) {
  console.log(`Current gid: ${process.getegid()}`);
}
```

```cjs
const process = require('node:process');

if (process.getegid) {
  console.log(`Current gid: ${process.getegid()}`);
}
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。

## `process.geteuid()`

<!-- YAML
added: v2.0.0
-->

* 返回: {Object}

`process.geteuid()` 方法返回进程的数字有效用户标识。（参见 geteuid(2)。）

```mjs
import process from 'node:process';

if (process.geteuid) {
  console.log(`Current uid: ${process.geteuid()}`);
}
```

```cjs
const process = require('node:process');

if (process.geteuid) {
  console.log(`Current uid: ${process.geteuid()}`);
}
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。

## `process.getgid()`

<!-- YAML
added: v0.1.31
-->

* 返回: {Object}

`process.getgid()` 方法返回进程的数字组标识。（参见 getgid(2)。）

```mjs
import process from 'node:process';

if (process.getgid) {
  console.log(`Current gid: ${process.getgid()}`);
}
```

```cjs
const process = require('node:process');

if (process.getgid) {
  console.log(`Current gid: ${process.getgid()}`);
}
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。

## `process.getgroups()`

<!-- YAML
added: v0.9.4
-->

* 返回: {integer\[]}

`process.getgroups()` 方法返回一个包含补充组 ID 的数组。POSIX 未指定是否包含有效组 ID，但 Node.js 确保它始终包含。

```mjs
import process from 'node:process';

if (process.getgroups) {
  console.log(process.getgroups()); // [ 16, 21, 297 ]
}
```

```cjs
const process = require('node:process');

if (process.getgroups) {
  console.log(process.getgroups()); // [ 16, 21, 297 ]
}
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。

## `process.getuid()`

<!-- YAML
added: v0.1.28
-->

* 返回: {integer}

`process.getuid()` 方法返回进程的数字用户标识。（参见 getuid(2)。）

```mjs
import process from 'node:process';

if (process.getuid) {
  console.log(`Current uid: ${process.getuid()}`);
}
```

```cjs
const process = require('node:process');

if (process.getuid) {
  console.log(`Current uid: ${process.getuid()}`);
}
```

此函数在 Windows 上不可用。

## `process.hasUncaughtExceptionCaptureCallback()`

<!-- YAML
added: v9.3.0
-->

* 返回: {boolean}

指示是否已使用 [`process.setUncaughtExceptionCaptureCallback()`][] 设置了回调。

## `process.hrtime([time])`

<!-- YAML
added: v0.7.6
-->

> Stability: 3 - Legacy. Use [`process.hrtime.bigint()`][] instead.

* `time` {integer\[]} 之前调用 `process.hrtime()` 的结果
* 返回: {integer\[]}

这是在 JavaScript 中引入 `bigint` 之前 [`process.hrtime.bigint()`][] 的旧版本。

`process.hrtime()` 方法返回当前高分辨率实时时间，单位为 `[seconds, nanoseconds]` 元组 `Array`，其中 `nanoseconds` 是实时中无法以秒精度表示的剩余部分。

`time` 是一个可选参数，必须是先前 `process.hrtime()` 调用的结果，用于与当前时间进行差异比较。如果传入的参数不是元组 `Array`，将抛出 `TypeError`。传入用户定义的数组而不是先前调用 `process.hrtime()` 的结果将导致未定义的行为。

这些时间相对于过去的任意时间，与一天中的时间无关，因此不受时钟漂移的影响。主要用途是测量时间间隔之间的性能：

```mjs
import { hrtime } from 'node:process';

const NS_PER_SEC = 1e9;
const time = hrtime();
// [ 1800216, 25 ]

setTimeout(() => {
  const diff = hrtime(time);
  // [ 1, 552 ]

  console.log(`Benchmark took ${diff[0] * NS_PER_SEC + diff[1]} nanoseconds`);
  // Benchmark took 1000000552 nanoseconds
}, 1000);
```

```cjs
const { hrtime } = require('node:process');

const NS_PER_SEC = 1e9;
const time = hrtime();
// [ 1800216, 25 ]

setTimeout(() => {
  const diff = hrtime(time);
  // [ 1, 552 ]

  console.log(`Benchmark took ${diff[0] * NS_PER_SEC + diff[1]} nanoseconds`);
  // Benchmark took 1000000552 nanoseconds
}, 1000);
```

## `process.hrtime.bigint()`

<!-- YAML
added: v10.7.0
-->

* 返回: {bigint}

[`process.hrtime()`][] 方法的 `bigint` 版本，以纳秒为单位返回当前高分辨率实时时间作为 `bigint`。

与 [`process.hrtime()`][] 不同，它不支持额外的 `time` 参数，因为差值可以直接通过两个 `bigint` 的减法计算得出。

```mjs
import { hrtime } from 'node:process';

const start = hrtime.bigint();
// 191051479007711n

setTimeout(() => {
  const end = hrtime.bigint();
  // 191052633396993n

  console.log(`Benchmark took ${end - start} nanoseconds`);
  // Benchmark took 1154389282 nanoseconds
}, 1000);
```

```cjs
const { hrtime } = require('node:process');

const start = hrtime.bigint();
// 191051479007711n

setTimeout(() => {
  const end = hrtime.bigint();
  // 191052633396993n

  console.log(`Benchmark took ${end - start} nanoseconds`);
  // Benchmark took 1154389282 nanoseconds
}, 1000);
```

## `process.initgroups(user, extraGroup)`

<!-- YAML
added: v0.9.4
-->

* `user` {string|number} 用户名或数字标识符。
* `extraGroup` {string|number} 组名或数字标识符。

`process.initgroups()` 方法读取 `/etc/group` 文件并初始化组访问列表，使用用户所属的所有组。这是一个特权操作，要求 Node.js 进程具有 `root` 访问权限或 `CAP_SETGID` 能力。

删除权限时要小心：

```mjs
import { getgroups, initgroups, setgid } from 'node:process';

console.log(getgroups());         // [ 0 ]
initgroups('nodeuser', 1000);     // 切换用户
console.log(getgroups());         // [ 27, 30, 46, 1000, 0 ]
setgid(1000);                     // 删除 root gid
console.log(getgroups());         // [ 27, 30, 46, 1000 ]
```

```cjs
const { getgroups, initgroups, setgid } = require('node:process');

console.log(getgroups());         // [ 0 ]
initgroups('nodeuser', 1000);     // 切换用户
console.log(getgroups());         // [ 27, 30, 46, 1000, 0 ]
setgid(1000);                     // 删除 root gid
console.log(getgroups());         // [ 27, 30, 46, 1000 ]
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。
此功能在 [`Worker`][] 线程中不可用。

## `process.kill(pid[, signal])`

<!-- YAML
added: v0.0.6
-->

* `pid` {number} 进程 ID
* `signal` {string|number} 要发送的信号，可以是字符串或数字。**默认值:** `'SIGTERM'`。

`process.kill()` 方法将 `signal` 发送给由 `pid` 标识的进程。

信号名称是字符串，例如 `'SIGINT'` 或 `'SIGHUP'`。有关更多信息，请参阅[信号事件][]和 kill(2)。

如果目标 `pid` 不存在，此方法将抛出错误。作为特殊情况，可以使用 `0` 信号来测试进程是否存在。Windows 平台如果 `pid` 用于杀死进程组，将抛出错误。

尽管此函数的名称是 `process.kill()`，但它实际上只是一个信号发送器，就像 `kill` 系统调用一样。发送的信号可能对目标进程执行除杀死之外的其他操作。

```mjs
import process, { kill } from 'node:process';

process.on('SIGHUP', () => {
  console.log('Got SIGHUP signal.');
});

setTimeout(() => {
  console.log('Exiting.');
  process.exit(0);
}, 100);

kill(process.pid, 'SIGHUP');
```

```cjs
const process = require('node:process');

process.on('SIGHUP', () => {
  console.log('Got SIGHUP signal.');
});

setTimeout(() => {
  console.log('Exiting.');
  process.exit(0);
}, 100);

process.kill(process.pid, 'SIGHUP');
```

当 Node.js 进程收到 `SIGUSR1` 时，Node.js 将启动调试器。请参阅[信号事件][]。

## `process.loadEnvFile(path)`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
changes:
  - version: v24.10.0
    pr-url: https://github.com/nodejs/node/pull/59925
    description: This API is no longer experimental.
-->

* `path` {string | URL | Buffer | undefined}. **默认值:** `'./.env'`

将 `.env` 文件加载到 `process.env` 中。在 `.env` 文件中使用 `NODE_OPTIONS` 不会对 Node.js 产生任何影响。

```cjs
const { loadEnvFile } = require('node:process');
loadEnvFile();
```

```mjs
import { loadEnvFile } from 'node:process';
loadEnvFile();
```

## `process.mainModule`

<!-- YAML
added: v0.1.17
deprecated: v14.0.0
-->

> Stability: 0 - Deprecated: Use [`require.main`][] instead.

* 类型: {Object}

`process.mainModule` 属性提供了一种检索 [`require.main`][] 的替代方式。区别在于，如果主模块在运行时发生更改，[`require.main`][] 可能仍然引用在更改发生之前所需模块中的原始主模块。通常，可以安全地假设两者引用的是同一个模块。

与 [`require.main`][] 一样，如果没有入口脚本，`process.mainModule` 将是 `undefined`。

## `process.memoryUsage()`

<!-- YAML
added: v0.1.16
changes:
  - version:
     - v13.9.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/31550
    description: Added `arrayBuffers` to the returned object.
  - version: v7.2.0
    pr-url: https://github.com/nodejs/node/pull/9587
    description: Added `external` to the returned object.
-->

* 返回: {Object}
  * `rss` {integer}
  * `heapTotal` {integer}
  * `heapUsed` {integer}
  * `external` {integer}
  * `arrayBuffers` {integer}

返回一个描述 Node.js 进程内存使用情况的对象，单位为字节。

```mjs
import { memoryUsage } from 'node:process';

console.log(memoryUsage());
// 打印:
// {
//  rss: 4935680,
//  heapTotal: 1826816,
//  heapUsed: 650472,
//  external: 49879,
//  arrayBuffers: 9386
// }
```

```cjs
const { memoryUsage } = require('node:process');

console.log(memoryUsage());
// 打印:
// {
//  rss: 4935680,
//  heapTotal: 1826816,
//  heapUsed: 650472,
//  external: 49879,
//  arrayBuffers: 9386
// }
```

* `heapTotal` 和 `heapUsed` 指的是 V8 的内存使用情况。
* `external` 指的是绑定到由 V8 管理的 JavaScript 对象的 C++ 对象的内存使用情况。
* `rss`，常驻集大小，是进程在主内存设备中占据的空间量（这是总分配内存的一个子集），包括所有 C++ 和 JavaScript 对象和代码。
* `arrayBuffers` 指的是为 `ArrayBuffer` 和 `SharedArrayBuffer` 分配的内存，包括所有 Node.js [`Buffer`][]。这也包含在 `external` 值中。当 Node.js 作为嵌入式库使用时，此值可能为 `0`，因为在这种情况下可能不会跟踪 `ArrayBuffer` 的分配。

当使用 [`Worker`][] 线程时，`rss` 将是整个进程的有效值，而其他字段仅引用当前线程。

`process.memoryUsage()` 方法遍历每个页面以收集有关内存使用情况的信息，这取决于程序内存分配可能会很慢。

### 关于 process.memoryUsage 的说明

在 Linux 或其他通常使用 glibc 的系统上，应用程序可能尽管 `heapTotal` 稳定，但 `rss` 持续增长，这是由于 glibc `malloc` 实现导致的碎片化。请参阅 [nodejs/node#21973][] 了解如何切换到替代的 `malloc` 实现以解决性能问题。

## `process.memoryUsage.rss()`

<!-- YAML
added:
  - v15.6.0
  - v14.18.0
-->

* 返回: {integer}

`process.memoryUsage.rss()` 方法返回一个整数，表示常驻集大小（RSS），单位为字节。

常驻集大小是进程在主内存设备中占据的空间量（这是总分配内存的一个子集），包括所有 C++ 和 JavaScript 对象和代码。

这与 `process.memoryUsage()` 提供的 `rss` 属性值相同，但 `process.memoryUsage.rss()` 更快。

```mjs
import { memoryUsage } from 'node:process';

console.log(memoryUsage.rss());
// 35655680
```

```cjs
const { memoryUsage } = require('node:process');

console.log(memoryUsage.rss());
// 35655680
```

## `process.nextTick(callback[, ...args])`

<!-- YAML
added: v0.1.26
changes:
  - version:
    - v22.7.0
    - v20.18.0
    pr-url: https://github.com/nodejs/node/pull/51280
    description: Changed stability to Legacy.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v1.8.1
    pr-url: https://github.com/nodejs/node/pull/1077
    description: Additional arguments after `callback` are now supported.
-->

> Stability: 3 - Legacy. Use [`queueMicrotask()`][] instead.

* `callback` {Function}
* `...args` {any} 调用 `callback` 时传入的额外参数

`process.nextTick()` 将 `callback` 添加到 "next tick queue"。这个队列在当前 JavaScript 栈上的操作运行完成后、事件循环继续之前被完全排空。如果递归调用 `process.nextTick()`，可能会创建无限循环。有关更多背景信息，请参阅[事件循环][]指南。

```mjs
import { nextTick } from 'node:process';

console.log('start');
nextTick(() => {
  console.log('nextTick callback');
});
console.log('scheduled');
// 输出:
// start
// scheduled
// nextTick callback
```

```cjs
const { nextTick } = require('node:process');

console.log('start');
nextTick(() => {
  console.log('nextTick callback');
});
console.log('scheduled');
// 输出:
// start
// scheduled
// nextTick callback
```

这在开发 API 时非常重要，以便让用户有机会在对象构造之后、任何 I/O 发生之前分配事件处理程序：

```mjs
import { nextTick } from 'node:process';

function MyThing(options) {
  this.setupOptions(options);

  nextTick(() => {
    this.startDoingStuff();
  });
}

const thing = new MyThing();
thing.getReadyForStuff();

// thing.startDoingStuff() 现在被调用，而不是之前。
```

```cjs
const { nextTick } = require('node:process');

function MyThing(options) {
  this.setupOptions(options);

  nextTick(() => {
    this.startDoingStuff();
  });
}

const thing = new MyThing();
thing.getReadyForStuff();

// thing.startDoingStuff() 现在被调用，而不是之前。
```

对于 API，要么 100% 同步，要么 100% 异步，这一点非常重要。考虑这个例子：

```js
// 警告！不要使用！不安全的危险！
function maybeSync(arg, cb) {
  if (arg) {
    cb();
    return;
  }

  fs.stat('file', cb);
}
```

这个 API 是危险的，因为在以下情况下：

```js
const maybeTrue = Math.random() > 0.5;

maybeSync(maybeTrue, () => {
  foo();
});

bar();
```

不清楚是 `foo()` 还是 `bar()` 会先被调用。

以下方法要好得多：

```mjs
import { nextTick } from 'node:process';

function definitelyAsync(arg, cb) {
  if (arg) {
    nextTick(cb);
    return;
  }

  fs.stat('file', cb);
}
```

```cjs
const { nextTick } = require('node:process');

function definitelyAsync(arg, cb) {
  if (arg) {
    nextTick(cb);
    return;
  }

  fs.stat('file', cb);
}
```

### 何时使用 `queueMicrotask()` 与 `process.nextTick()`

[`queueMicrotask()`][] API 是 `process.nextTick()` 的替代方案，它不使用 "next tick queue"，而是使用用于执行已解析 promise 的 then、catch 和 finally 处理程序的相同微任务队列来延迟函数的执行。

在 Node.js 中，每次排空 "next tick queue" 后，微任务队列会立即被排空。

所以在 CJS 模块中，`process.nextTick()` 回调总是在 `queueMicrotask()` 回调之前运行。然而，由于 ESM 模块已经作为微任务队列的一部分被处理，在那里 `queueMicrotask()` 回调总是在 `process.nextTick()` 回调之前执行，因为 Node.js 已经在排空微任务队列的过程中。

```mjs
import { nextTick } from 'node:process';

Promise.resolve().then(() => console.log('resolve'));
queueMicrotask(() => console.log('microtask'));
nextTick(() => console.log('nextTick'));
// 输出:
// resolve
// microtask
// nextTick
```

```cjs
const { nextTick } = require('node:process');

Promise.resolve().then(() => console.log('resolve'));
queueMicrotask(() => console.log('microtask'));
nextTick(() => console.log('nextTick'));
// 输出:
// nextTick
// resolve
// microtask
```

对于_大多数_用户层面的用例，`queueMicrotask()` API 提供了一种可移植且可靠的延迟执行机制，适用于多个 JavaScript 平台环境，应优先于 `process.nextTick()`。在简单场景中，`queueMicrotask()` 可以作为 `process.nextTick()` 的直接替代品。

```js
console.log('start');
queueMicrotask(() => {
  console.log('microtask callback');
});
console.log('scheduled');
// 输出:
// start
// scheduled
// microtask callback
```

两个 API 之间一个值得注意的区别是 `process.nextTick()` 允许指定额外的值，这些值将在延迟函数被调用时作为参数传递给该函数。使用 `queueMicrotask()` 实现相同的结果需要使用闭包或绑定函数：

```js
function deferred(a, b) {
  console.log('microtask', a + b);
}

console.log('start');
queueMicrotask(deferred.bind(undefined, 1, 2));
console.log('scheduled');
// 输出:
// start
// scheduled
// microtask 3
```

在 next tick 队列和微任务队列中引发的错误处理方式有细微差别。在排队的微任务回调中引发的错误应在排队的回调中尽可能处理。如果未处理，可以使用 `process.on('uncaughtException')` 事件处理程序来捕获和处理错误。

如果有疑问，除非需要 `process.nextTick()` 的特定功能，否则请使用 `queueMicrotask()`。

## `process.noDeprecation`

<!-- YAML
added: v0.8.0
-->

* 类型: {boolean}

`process.noDeprecation` 属性指示是否在当前 Node.js 进程上设置了 `--no-deprecation` 标志。有关此标志行为的更多信息，请参阅 [`'warning'`][process_warning] 事件和 [`emitWarning()` 方法][process_emit_warning] 的文档。

## `process.permission`

<!-- YAML
added: v20.0.0
-->

* 类型: {Object}

此 API 可通过 [`--permission`][] 标志使用。

`process.permission` 是一个对象，其方法用于管理当前进程的权限。其他文档可在[权限模型][]中找到。

### `process.permission.has(scope[, reference])`

<!-- YAML
added: v20.0.0
-->

* `scope` {string}
* `reference` {string}
* 返回: {boolean}

验证进程能够访问给定的作用域和引用。如果未提供引用，则假定为全局作用域，例如，`process.permission.has('fs.read')` 将检查进程是否具有所有文件系统读取权限。

引用的含义基于提供的作用域。例如，当作用域为文件系统时，引用表示文件和文件夹。

可用的作用域有：

* `fs` - 所有文件系统
* `fs.read` - 文件系统读取操作
* `fs.write` - 文件系统写入操作
* `child` - 子进程生成操作
* `worker` - 工作线程生成操作

```js
// 检查进程是否有权限读取 README 文件
process.permission.has('fs.read', './README.md');
// 检查进程是否有读取权限操作
process.permission.has('fs.read');
```

## `process.pid`

<!-- YAML
added: v0.1.15
-->

* 类型: {integer}

`process.pid` 属性返回进程的 PID。

```mjs
import { pid } from 'node:process';

console.log(`This process is pid ${pid}`);
```

```cjs
const { pid } = require('node:process');

console.log(`This process is pid ${pid}`);
```

## `process.platform`

<!-- YAML
added: v0.1.16
-->

* 类型: {string}

`process.platform` 属性返回一个字符串，标识为其编译 Node.js 二进制文件的操作系统平台。

当前可能的值有：

* `'aix'`
* `'darwin'`
* `'freebsd'`
* `'linux'`
* `'openbsd'`
* `'sunos'`
* `'win32'`

```mjs
import { platform } from 'node:process';

console.log(`This platform is ${platform}`);
```

```cjs
const { platform } = require('node:process');

console.log(`This platform is ${platform}`);
```

如果 Node.js 是在 Android 操作系统上构建的，也可能返回 `'android'` 值。然而，Node.js 中的 Android 支持[是实验性的][Android building]。

## `process.ppid`

<!-- YAML
added:
  - v9.2.0
  - v8.10.0
  - v6.13.0
-->

* 类型: {integer}

`process.ppid` 属性返回当前进程的父进程的 PID。

```mjs
import { ppid } from 'node:process';

console.log(`The parent process is pid ${ppid}`);
```

```cjs
const { ppid } = require('node:process');

console.log(`The parent process is pid ${ppid}`);
```

## `process.ref(maybeRefable)`

<!-- YAML
added:
  - v23.6.0
  - v22.14.0
-->

> Stability: 1 - Experimental

* `maybeRefable` {any} 一个可能是 "可引用" 的对象。

如果一个对象实现了 Node.js "可引用协议"，那么它就是 "可引用" 的。具体来说，这意味着该对象实现了 `Symbol.for('nodejs.ref')` 和 `Symbol.for('nodejs.unref')` 方法。"已引用" 的对象将保持 Node.js 事件循环活动，而 "未引用" 的对象则不会。历史上，这是通过在对象上直接使用 `ref()` 和 `unref()` 方法来实现的。然而，这种模式正在被弃用，转而支持 "可引用协议"，以便更好地支持那些 API 无法修改以添加 `ref()` 和 `unref()` 方法但仍需要支持该行为的 Web 平台 API 类型。

## `process.release`

<!-- YAML
added: v3.0.0
changes:
  - version: v4.2.0
    pr-url: https://github.com/nodejs/node/pull/3212
    description: The `lts` property is now supported.
-->

* 类型: {Object}

`process.release` 属性返回一个 `Object`，包含与当前发布相关的元数据，包括源代码 tarball 和仅头文件 tarball 的 URL。

`process.release` 包含以下属性：

* `name` {string} 一个始终为 `'node'` 的值。
* `sourceUrl` {string} 指向包含当前发布源代码的 _`.tar.gz`_ 文件的绝对 URL。
* `headersUrl`{string} 指向仅包含当前发布源代码头文件的 _`.tar.gz`_ 文件的绝对 URL。此文件比完整的源代码文件小得多，可用于编译 Node.js 原生插件。
* `libUrl` {string|undefined} 指向与当前发布架构和版本匹配的 _`node.lib`_ 文件的绝对 URL。此文件用于编译 Node.js 原生插件。_此属性仅存在于 Node.js 的 Windows 构建上，在所有其他平台上将缺失。_
* `lts` {string|undefined} 一个字符串标签，标识此发布的 [LTS][] 标签。此属性仅存在于 LTS 发布中，对于所有其他发布类型（包括 _Current_ 发布）为 `undefined`。有效值包括 LTS 发布代码名称（包括不再支持的代码名称）。
  * `'Fermium'` 用于从 14.15.0 开始的 14.x LTS 线。
  * `'Gallium'` 用于从 16.13.0 开始的 16.x LTS 线。
  * `'Hydrogen'` 用于从 18.12.0 开始的 18.x LTS 线。
    对于其他 LTS 发布代码名称，请参阅 [Node.js 更新日志存档](https://github.com/nodejs/node/blob/HEAD/doc/changelogs/CHANGELOG_ARCHIVE.md)

<!-- eslint-skip -->

```js
{
  name: 'node',
  lts: 'Hydrogen',
  sourceUrl: 'https://nodejs.org/download/release/v18.12.0/node-v18.12.0.tar.gz',
  headersUrl: 'https://nodejs.org/download/release/v18.12.0/node-v18.12.0-headers.tar.gz',
  libUrl: 'https://nodejs.org/download/release/v18.12.0/win-x64/node.lib'
}
```

在来自非发布版本源代码树的自定义构建中，可能只有 `name` 属性存在。不应依赖其他属性的存在。

## `process.report`

<!-- YAML
added: v11.8.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This API is no longer experimental.
-->

* 类型: {Object}

`process.report` 是一个对象，其方法用于为当前进程生成诊断报告。其他文档可在[报告文档][]中找到。

### `process.report.compact`

<!-- YAML
added:
 - v13.12.0
 - v12.17.0
-->

* 类型: {boolean}

以紧凑格式编写报告，单行 JSON，比设计用于人类阅读的默认多行格式更易于日志处理系统使用。

```mjs
import { report } from 'node:process';

console.log(`Reports are compact? ${report.compact}`);
```

```cjs
const { report } = require('node:process');

console.log(`Reports are compact? ${report.compact}`);
```

### `process.report.directory`

<!-- YAML
added: v11.12.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This API is no longer experimental.
-->

* 类型: {string}

报告写入的目录。默认值为空字符串，表示报告写入 Node.js 进程的当前工作目录。

```mjs
import { report } from 'node:process';

console.log(`Report directory is ${report.directory}`);
```

```cjs
const { report } = require('node:process');

console.log(`Report directory is ${report.directory}`);
```

### `process.report.filename`

<!-- YAML
added: v11.12.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This API is no longer experimental.
-->

* 类型: {string}

报告写入的文件名。如果设置为空字符串，输出文件名将由时间戳、PID 和序列号组成。默认值为空字符串。

如果 `process.report.filename` 的值设置为 `'stdout'` 或 `'stderr'`，则报告将分别写入进程的 stdout 或 stderr。

```mjs
import { report } from 'node:process';

console.log(`Report filename is ${report.filename}`);
```

```cjs
const { report } = require('node:process');

console.log(`Report filename is ${report.filename}`);
```

### `process.report.getReport([err])`

<!-- YAML
added: v11.8.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This API is no longer experimental.
-->

* `err` {Error} 用于报告 JavaScript 栈的自定义错误。
* 返回: {Object}

返回运行进程诊断报告的 JavaScript 对象表示形式。报告的 JavaScript 栈跟踪取自 `err`（如果存在）。

```mjs
import { report } from 'node:process';
import util from 'node:util';

const data = report.getReport();
console.log(data.header.nodejsVersion);

// 类似于 process.report.writeReport()
import fs from 'node:fs';
fs.writeFileSync('my-report.log', util.inspect(data), 'utf8');
```

```cjs
const { report } = require('node:process');
const util = require('node:util');

const data = report.getReport();
console.log(data.header.nodejsVersion);

// 类似于 process.report.writeReport()
const fs = require('node:fs');
fs.writeFileSync('my-report.log', util.inspect(data), 'utf8');
```

其他文档可在[报告文档][]中找到。

### `process.report.reportOnFatalError`

<!-- YAML
added: v11.12.0
changes:
  - version:
     - v15.0.0
     - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/35654
    description: This API is no longer experimental.
-->

* 类型: {boolean}

如果为 `true`，则在发生致命错误（如内存不足错误或失败的 C++ 断言）时生成诊断报告。

```mjs
import { report } from 'node:process';

console.log(`Report on fatal error: ${report.reportOnFatalError}`);
```

```cjs
const { report } = require('node:process');

console.log(`Report on fatal error: ${report.reportOnFatalError}`);
```

### `process.report.reportOnSignal`

<!-- YAML
added: v11.12.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This API is no longer experimental.
-->

* 类型: {boolean}

如果为 `true`，则当进程收到 `process.report.signal` 指定的信号时生成诊断报告。

```mjs
import { report } from 'node:process';

console.log(`Report on signal: ${report.reportOnSignal}`);
```

```cjs
const { report } = require('node:process');

console.log(`Report on signal: ${report.reportOnSignal}`);
```

### `process.report.reportOnUncaughtException`

<!-- YAML
added: v11.12.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This API is no longer experimental.
-->

* 类型: {boolean}

如果为 `true`，则在发生未捕获异常时生成诊断报告。

```mjs
import { report } from 'node:process';

console.log(`Report on exception: ${report.reportOnUncaughtException}`);
```

```cjs
const { report } = require('node:process');

console.log(`Report on exception: ${report.reportOnUncaughtException}`);
```

### `process.report.excludeEnv`

<!-- YAML
added:
  - v23.3.0
  - v22.13.0
-->

* 类型: {boolean}

如果为 `true`，则生成不包含环境变量的诊断报告。

### `process.report.signal`

<!-- YAML
added: v11.12.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This API is no longer experimental.
-->

* 类型: {string}

用于触发创建诊断报告的信号。默认为 `'SIGUSR2'`。

```mjs
import { report } from 'node:process';

console.log(`Report signal: ${report.signal}`);
```

```cjs
const { report } = require('node:process');

console.log(`Report signal: ${report.signal}`);
```

### `process.report.writeReport([filename][, err])`

<!-- YAML
added: v11.8.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This API is no longer experimental.
-->

* `filename` {string} 报告写入的文件名。这应该是一个相对路径，将附加到 `process.report.directory` 指定的目录，或者如果未指定，则附加到 Node.js 进程的当前工作目录。

* `err` {Error} 用于报告 JavaScript 栈的自定义错误。

* 返回: {string} 返回生成报告的文件名。

将诊断报告写入文件。如果未提供 `filename`，默认文件名包括日期、时间、PID 和序列号。报告的 JavaScript 栈跟踪取自 `err`（如果存在）。

如果 `filename` 的值设置为 `'stdout'` 或 `'stderr'`，则报告将分别写入进程的 stdout 或 stderr。

```mjs
import { report } from 'node:process';

report.writeReport();
```

```cjs
const { report } = require('node:process');

report.writeReport();
```

其他文档可在[报告文档][]中找到。

## `process.resourceUsage()`

<!-- YAML
added: v12.6.0
-->

* 返回: {Object} 当前进程的资源使用情况。所有这些值都来自 `uv_getrusage` 调用，该调用返回一个 [`uv_rusage_t` 结构体][uv_rusage_t]。
  * `userCPUTime` {integer} 映射到以微秒计算的 `ru_utime`。它与 [`process.cpuUsage().user`][process.cpuUsage] 的值相同。
  * `systemCPUTime` {integer} 映射到以微秒计算的 `ru_stime`。它与 [`process.cpuUsage().system`][process.cpuUsage] 的值相同。
  * `maxRSS` {integer} 映射到 `ru_maxrss`，即使用的最大常驻集大小，单位为千字节（1024 字节）。
  * `sharedMemorySize` {integer} 映射到 `ru_ixrss`，但不受任何平台支持。
  * `unsharedDataSize` {integer} 映射到 `ru_idrss`，但不受任何平台支持。
  * `unsharedStackSize` {integer} 映射到 `ru_isrss`，但不受任何平台支持。
  * `minorPageFault` {integer} 映射到 `ru_minflt`，即进程的次要页面错误次数，有关更多详细信息，请参阅[此文章][wikipedia_minor_fault]。
  * `majorPageFault` {integer} 映射到 `ru_majflt`，即进程的主要页面错误次数，有关更多详细信息，请参阅[此文章][wikipedia_major_fault]。此字段在 Windows 上不受支持。
  * `swappedOut` {integer} 映射到 `ru_nswap`，但不受任何平台支持。
  * `fsRead` {integer} 映射到 `ru_inblock`，即文件系统必须执行输入的次数。
  * `fsWrite` {integer} 映射到 `ru_oublock`，即文件系统必须执行输出的次数。
  * `ipcSent` {integer} 映射到 `ru_msgsnd`，但不受任何平台支持。
  * `ipcReceived` {integer} 映射到 `ru_msgrcv`，但不受任何平台支持。
  * `signalsCount` {integer} 映射到 `ru_nsignals`，但不受任何平台支持。
  * `voluntaryContextSwitches` {integer} 映射到 `ru_nvcsw`，即由于进程在其时间片完成之前自愿放弃处理器（通常是为了等待资源可用）而导致的 CPU 上下文切换次数。此字段在 Windows 上不受支持。
  * `involuntaryContextSwitches` {integer} 映射到 `ru_nivcsw`，即由于更高优先级的进程变得可运行或因为当前进程超出其时间片而导致的 CPU 上下文切换次数。此字段在 Windows 上不受支持。

```mjs
import { resourceUsage } from 'node:process';

console.log(resourceUsage());
/*
  将输出:
  {
    userCPUTime: 82872,
    systemCPUTime: 4143,
    maxRSS: 33164,
    sharedMemorySize: 0,
    unsharedDataSize: 0,
    unsharedStackSize: 0,
    minorPageFault: 2469,
    majorPageFault: 0,
    swappedOut: 0,
    fsRead: 0,
    fsWrite: 8,
    ipcSent: 0,
    ipcReceived: 0,
    signalsCount: 0,
    voluntaryContextSwitches: 79,
    involuntaryContextSwitches: 1
  }
*/
```

```cjs
const { resourceUsage } = require('node:process');

console.log(resourceUsage());
/*
  将输出:
  {
    userCPUTime: 82872,
    systemCPUTime: 4143,
    maxRSS: 33164,
    sharedMemorySize: 0,
    unsharedDataSize: 0,
    unsharedStackSize: 0,
    minorPageFault: 2469,
    majorPageFault: 0,
    swappedOut: 0,
    fsRead: 0,
    fsWrite: 8,
    ipcSent: 0,
    ipcReceived: 0,
    signalsCount: 0,
    voluntaryContextSwitches: 79,
    involuntaryContextSwitches: 1
  }
*/
```

## `process.send(message[, sendHandle[, options]][, callback])`

<!-- YAML
added: v0.5.9
-->

* `message` {Object}
* `sendHandle` {net.Server|net.Socket}
* `options` {Object} 用于参数化某些类型句柄的发送。`options` 支持以下属性：
  * `keepOpen` {boolean} 在传递 `net.Socket` 实例时可使用的值。当为 `true` 时，套接字在发送进程中保持打开状态。**默认值:** `false`。
* `callback` {Function}
* 返回: {boolean}

如果 Node.js 是使用 IPC 通道生成的，则 `process.send()` 方法可用于向父进程发送消息。消息将作为父进程的 [`ChildProcess`][] 对象上的 [`'message'`][] 事件接收。

如果 Node.js 不是使用 IPC 通道生成的，`process.send` 将是 `undefined`。

消息经过序列化和解析。结果消息可能与最初发送的消息不同。

## `process.setegid(id)`

<!-- YAML
added: v2.0.0
-->

* `id` {string|number} 组名或 ID

`process.setegid()` 方法设置进程的有效组标识。（参见 setegid(2)。）`id` 可以作为数字 ID 或组名字符串传递。如果指定了组名，则此方法在解析关联的数字 ID 时会阻塞。

```mjs
import process from 'node:process';

if (process.getegid && process.setegid) {
  console.log(`Current gid: ${process.getegid()}`);
  try {
    process.setegid(501);
    console.log(`New gid: ${process.getegid()}`);
  } catch (err) {
    console.error(`Failed to set gid: ${err}`);
  }
}
```

```cjs
const process = require('node:process');

if (process.getegid && process.setegid) {
  console.log(`Current gid: ${process.getegid()}`);
  try {
    process.setegid(501);
    console.log(`New gid: ${process.getegid()}`);
  } catch (err) {
    console.error(`Failed to set gid: ${err}`);
  }
}
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。
此功能在 [`Worker`][] 线程中不可用。

## `process.seteuid(id)`

<!-- YAML
added: v2.0.0
-->

* `id` {string|number} 用户名或 ID

`process.seteuid()` 方法设置进程的有效用户标识。（参见 seteuid(2)。）`id` 可以作为数字 ID 或用户名字符串传递。如果指定了用户名，则该方法在解析关联的数字 ID 时会阻塞。

```mjs
import process from 'node:process';

if (process.geteuid && process.seteuid) {
  console.log(`Current uid: ${process.geteuid()}`);
  try {
    process.seteuid(501);
    console.log(`New uid: ${process.geteuid()}`);
  } catch (err) {
    console.error(`Failed to set uid: ${err}`);
  }
}
```

```cjs
const process = require('node:process');

if (process.geteuid && process.seteuid) {
  console.log(`Current uid: ${process.geteuid()}`);
  try {
    process.seteuid(501);
    console.log(`New uid: ${process.geteuid()}`);
  } catch (err) {
    console.error(`Failed to set uid: ${err}`);
  }
}
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。
此功能在 [`Worker`][] 线程中不可用。

## `process.setgid(id)`

<!-- YAML
added: v0.1.31
-->

* `id` {string|number} 组名或 ID

`process.setgid()` 方法设置进程的组标识。（参见 setgid(2)。）`id` 可以作为数字 ID 或组名字符串传递。如果指定了组名，则此方法在解析关联的数字 ID 时会阻塞。

```mjs
import process from 'node:process';

if (process.getgid && process.setgid) {
  console.log(`Current gid: ${process.getgid()}`);
  try {
    process.setgid(501);
    console.log(`New gid: ${process.getgid()}`);
  } catch (err) {
    console.error(`Failed to set gid: ${err}`);
  }
}
```

```cjs
const process = require('node:process');

if (process.getgid && process.setgid) {
  console.log(`Current gid: ${process.getgid()}`);
  try {
    process.setgid(501);
    console.log(`New gid: ${process.getgid()}`);
  } catch (err) {
    console.error(`Failed to set gid: ${err}`);
  }
}
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。
此功能在 [`Worker`][] 线程中不可用。

## `process.setgroups(groups)`

<!-- YAML
added: v0.9.4
-->

* `groups` {integer\[]}

`process.setgroups()` 方法设置 Node.js 进程的补充组 ID。这是一个特权操作，要求 Node.js 进程具有 `root` 或 `CAP_SETGID` 能力。

`groups` 数组可以包含数字组 ID、组名或两者。

```mjs
import process from 'node:process';

if (process.getgroups && process.setgroups) {
  try {
    process.setgroups([501]);
    console.log(process.getgroups()); // 新的组
  } catch (err) {
    console.error(`Failed to set groups: ${err}`);
  }
}
```

```cjs
const process = require('node:process');

if (process.getgroups && process.setgroups) {
  try {
    process.setgroups([501]);
    console.log(process.getgroups()); // 新的组
  } catch (err) {
    console.error(`Failed to set groups: ${err}`);
  }
}
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。
此功能在 [`Worker`][] 线程中不可用。

## `process.setuid(id)`

<!-- YAML
added: v0.1.28
-->

* `id` {integer | string}

`process.setuid(id)` 方法设置进程的用户标识。（参见 setuid(2)。）`id` 可以作为数字 ID 或用户名字符串传递。如果指定了用户名，则该方法在解析关联的数字 ID 时会阻塞。

```mjs
import process from 'node:process';

if (process.getuid && process.setuid) {
  console.log(`Current uid: ${process.getuid()}`);
  try {
    process.setuid(501);
    console.log(`New uid: ${process.getuid()}`);
  } catch (err) {
    console.error(`Failed to set uid: ${err}`);
  }
}
```

```cjs
const process = require('node:process');

if (process.getuid && process.setuid) {
  console.log(`Current uid: ${process.getuid()}`);
  try {
    process.setuid(501);
    console.log(`New uid: ${process.getuid()}`);
  } catch (err) {
    console.error(`Failed to set uid: ${err}`);
  }
}
```

此函数仅在 POSIX 平台上可用（即不可在 Windows 或 Android 上使用）。
此功能在 [`Worker`][] 线程中不可用。

## `process.setSourceMapsEnabled(val)`

<!-- YAML
added:
  - v16.6.0
  - v14.18.0
-->

> Stability: 1 - Experimental: Use [`module.setSourceMapsSupport()`][] instead.

* `val` {boolean}

此函数启用或禁用栈跟踪的[源映射][]支持。

它提供与使用命令行选项 `--enable-source-maps` 启动 Node.js 进程相同的功能。

只有在启用源映射后加载的 JavaScript 文件中的源映射才会被解析和加载。

这意味着使用选项 `{ nodeModules: true, generatedCode: true }` 调用 `module.setSourceMapsSupport()`。

## `process.setUncaughtExceptionCaptureCallback(fn)`

<!-- YAML
added: v9.3.0
-->

* `fn` {Function|null}

`process.setUncaughtExceptionCaptureCallback()` 函数设置一个函数，当发生未捕获的异常时将调用该函数，该函数将接收异常值本身作为其第一个参数。

如果设置了这样的函数，则不会发出 [`'uncaughtException'`][] 事件。如果从命令行传递了 `--abort-on-uncaught-exception` 或通过 [`v8.setFlagsFromString()`][] 设置，则进程不会中止。为异常配置的操作（例如报告生成）也会受到影响

要取消设置捕获函数，可以使用 `process.setUncaughtExceptionCaptureCallback(null)`。使用非 `null` 参数调用此方法而另一个捕获函数已设置时，将抛出错误。

使用此函数与使用已弃用的 [`domain`][] 内置模块互斥。

## `process.sourceMapsEnabled`

<!-- YAML
added:
  - v20.7.0
  - v18.19.0
-->

> Stability: 1 - Experimental: Use [`module.getSourceMapsSupport()`][] instead.

* 类型: {boolean}

`process.sourceMapsEnabled` 属性返回是否启用了栈跟踪的[源映射][]支持。

## `process.stderr`

* 类型: {Stream}

`process.stderr` 属性返回连接到 `stderr`（fd `2`）的流。它是一个 [`net.Socket`][]（一个[双工][]流），除非 fd `2` 引用一个文件，在这种情况下它是一个[可写][]流。

`process.stderr` 在其他重要的方面与其他 Node.js 流不同。有关更多信息，请参阅[关于进程 I/O 的说明][]。

### `process.stderr.fd`

* 类型: {number}

此属性引用 `process.stderr` 的底层文件描述符的值。该值固定为 `2`。在 [`Worker`][] 线程中，此字段不存在。

## `process.stdin`

* 类型: {Stream}

`process.stdin` 属性返回连接到 `stdin`（fd `0`）的流。它是一个 [`net.Socket`][]（一个[双工][]流），除非 fd `0` 引用一个文件，在这种情况下它是一个[可读][]流。

有关如何从 `stdin` 读取的详细信息，请参阅 [`readable.read()`][]。

作为一个[双工][]流，`process.stdin` 也可以在 "old" 模式下使用，该模式兼容为 v0.10 之前的 Node.js 编写的脚本。有关更多信息，请参阅[流兼容性][]。

在 "old" 流模式下，`stdin` 流默认是暂停的，因此必须调用 `process.stdin.resume()` 来从中读取。另请注意，调用 `process.stdin.resume()` 本身会将流切换到 "old" 模式。

### `process.stdin.fd`

* 类型: {number}

此属性引用 `process.stdin` 的底层文件描述符的值。该值固定为 `0`。在 [`Worker`][] 线程中，此字段不存在。

## `process.stdout`

* 类型: {Stream}

`process.stdout` 属性返回连接到 `stdout`（fd `1`）的流。它是一个 [`net.Socket`][]（一个[双工][]流），除非 fd `1` 引用一个文件，在这种情况下它是一个[可写][]流。

例如，要将 `process.stdin` 复制到 `process.stdout`：

```mjs
import { stdin, stdout } from 'node:process';

stdin.pipe(stdout);
```

```cjs
const { stdin, stdout } = require('node:process');

stdin.pipe(stdout);
```

`process.stdout` 在其他重要的方面与其他 Node.js 流不同。有关更多信息，请参阅[关于进程 I/O 的说明][]。

### `process.stdout.fd`

* 类型: {number}

此属性引用 `process.stdout` 的底层文件描述符的值。该值固定为 `1`。在 [`Worker`][] 线程中，此字段不存在。

### 关于进程 I/O 的说明

`process.stdout` 和 `process.stderr` 在重要方面与其他 Node.js 流不同：

1. 它们分别由 [`console.log()`][] 和 [`console.error()`][] 内部使用。
2. 写入可能是同步的，取决于流连接到的对象以及系统是 Windows 还是 POSIX：
   * 文件：在 Windows 和 POSIX 上都是_同步的_
   * TTY（终端）：在 Windows 上是_异步的_，在 POSIX 上是_同步的_
   * 管道（和套接字）：在 Windows 上是_同步的_，在 POSIX 上是_异步的_

这些行为部分是由于历史原因，因为更改它们会产生向后不兼容性，但某些用户也期望这些行为。

同步写入可以避免诸如使用 `console.log()` 或 `console.error()` 编写的输出意外交错的问题，或者如果在异步写入完成之前调用 `process.exit()` 则根本不写入的问题。有关更多信息，请参阅 [`process.exit()`][]。

_**警告**_：同步写入会阻塞事件循环，直到写入完成。在输出到文件的情况下，这可能是瞬间的，但在高系统负载、接收端未被读取的管道或慢速终端或文件系统的情况下，事件循环可能经常被阻塞足够长的时间，从而产生严重的负面性能影响。在写入交互式终端会话时这可能不是问题，但在进行生产日志记录到进程输出流时要特别小心这一点。

要检查流是否连接到 [TTY][] 上下文，请检查 `isTTY` 属性。

例如：

```console
$ node -p "Boolean(process.stdin.isTTY)"
true
$ echo "foo" | node -p "Boolean(process.stdin.isTTY)"
false
$ node -p "Boolean(process.stdout.isTTY)"
true
$ node -p "Boolean(process.stdout.isTTY)" | cat
false
```

有关更多信息，请参阅 [TTY][] 文档。

## `process.throwDeprecation`

<!-- YAML
added: v0.9.12
-->

* 类型: {boolean}

`process.throwDeprecation` 的初始值指示是否在当前 Node.js 进程上设置了 `--throw-deprecation` 标志。`process.throwDeprecation` 是可变的，因此是否弃用警告会导致错误可能在运行时被改变。有关更多信息，请参阅 [`'warning'`][process_warning] 事件和 [`emitWarning()` 方法][process_emit_warning] 的文档。

```console
$ node --throw-deprecation -p "process.throwDeprecation"
true
$ node -p "process.throwDeprecation"
undefined
$ node
> process.emitWarning('test', 'DeprecationWarning');
undefined
> (node:26598) DeprecationWarning: test
> process.throwDeprecation = true;
true
> process.emitWarning('test', 'DeprecationWarning');
Thrown:
[DeprecationWarning: test] { name: 'DeprecationWarning' }
```

## `process.threadCpuUsage([previousValue])`

<!-- YAML
added: v23.9.0
-->

* `previousValue` {Object} 之前调用 `process.threadCpuUsage()` 的返回值
* 返回: {Object}
  * `user` {integer}
  * `system` {integer}

`process.threadCpuUsage()` 方法返回当前工作线程的用户和系统 CPU 时间使用情况，在一个具有 `user` 和 `system` 属性的对象中，这些属性的值是以微秒（百万分之一秒）为单位的时间值。

可以将之前调用 `process.threadCpuUsage()` 的结果作为参数传递给该函数，以获取差值读数。

## `process.title`

<!-- YAML
added: v0.1.104
-->

* 类型: {string}

`process.title` 属性返回当前进程标题（即返回 `ps` 的当前值）。将新值赋给 `process.title` 会修改 `ps` 的当前值。

当赋新值时，不同的平台会对标题施加不同的最大长度限制。通常这样的限制是相当有限的。例如，在 Linux 和 macOS 上，`process.title` 限制为二进制名称的大小加上命令行参数的长度，因为设置 `process.title` 会覆盖进程的 `argv` 内存。Node.js v0.8 允许更长的进程标题字符串，也覆盖了 `environ` 内存，但这在某些（相当模糊的）情况下可能不安全和令人困惑。

将值赋给 `process.title` 可能不会在进程管理器应用程序（如 macOS Activity Monitor 或 Windows Services Manager）中产生准确的标签。

## `process.traceDeprecation`

<!-- YAML
added: v0.8.0
-->

* 类型: {boolean}

`process.traceDeprecation` 属性指示是否在当前 Node.js 进程上设置了 `--trace-deprecation` 标志。有关此标志行为的更多信息，请参阅 [`'warning'`][process_warning] 事件和 [`emitWarning()` 方法][process_emit_warning] 的文档。

## `process.umask()`

<!-- YAML
added: v0.1.19
changes:
  - version:
    - v14.0.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/32499
    description: Calling `process.umask()` with no arguments is deprecated.
-->

> Stability: 0 - Deprecated. Calling `process.umask()` with no argument causes
> the process-wide umask to be written twice. This introduces a race condition
> between threads, and is a potential security vulnerability. There is no safe,
> cross-platform alternative API.

`process.umask()` 返回 Node.js 进程的文件模式创建掩码。子进程从父进程继承掩码。

## `process.umask(mask)`

<!-- YAML
added: v0.1.19
-->

* `mask` {string|integer}

`process.umask(mask)` 设置 Node.js 进程的文件模式创建掩码。子进程从父进程继承掩码。返回之前的掩码。

```mjs
import { umask } from 'node:process';

const newmask = 0o022;
const oldmask = umask(newmask);
console.log(
  `Changed umask from ${oldmask.toString(8)} to ${newmask.toString(8)}`,
);
```

```cjs
const { umask } = require('node:process');

const newmask = 0o022;
const oldmask = umask(newmask);
console.log(
  `Changed umask from ${oldmask.toString(8)} to ${newmask.toString(8)}`,
);
```

在 [`Worker`][] 线程中，`process.umask(mask)` 将抛出异常。

## `process.unref(maybeRefable)`

<!-- YAML
added:
  - v23.6.0
  - v22.14.0
-->

> Stability: 1 - Experimental

* `maybeUnfefable` {any} 一个可能是 "未引用" 的对象。

如果一个对象实现了 Node.js "可引用协议"，那么它就是 "可取消引用" 的。具体来说，这意味着该对象实现了 `Symbol.for('nodejs.ref')` 和 `Symbol.for('nodejs.unref')` 方法。"已引用" 的对象将保持 Node.js 事件循环活动，而 "未引用" 的对象则不会。历史上，这是通过在对象上直接使用 `ref()` 和 `unref()` 方法来实现的。然而，这种模式正在被弃用，转而支持 "可引用协议"，以便更好地支持那些 API 无法修改以添加 `ref()` 和 `unref()` 方法但仍需要支持该行为的 Web 平台 API 类型。

## `process.uptime()`

<!-- YAML
added: v0.5.0
-->

* 返回: {number}

`process.uptime()` 方法返回当前 Node.js 进程已运行的秒数。

返回值包括秒的小数部分。使用 `Math.floor()` 获取整秒数。

## `process.version`

<!-- YAML
added: v0.1.3
-->

* 类型: {string}

`process.version` 属性包含 Node.js 版本字符串。

```mjs
import { version } from 'node:process';

console.log(`Version: ${version}`);
// Version: v14.8.0
```

```cjs
const { version } = require('node:process');

console.log(`Version: ${version}`);
// Version: v14.8.0
```

要获取没有前缀 _v_ 的版本字符串，请使用 `process.versions.node`。

## `process.versions`

<!-- YAML
added: v0.2.0
changes:
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15785
    description: The `v8` property now includes a Node.js specific suffix.
  - version: v4.2.0
    pr-url: https://github.com/nodejs/node/pull/3102
    description: The `icu` property is now supported.
-->

* 类型: {Object}

`process.versions` 属性返回一个对象，列出 Node.js 及其依赖项的版本字符串。`process.versions.modules` 指示当前的 ABI 版本，每当 C++ API 更改时都会增加。Node.js 将拒绝加载针对不同模块 ABI 版本编译的模块。

```mjs
import { versions } from 'node:process';

console.log(versions);
```

```cjs
const { versions } = require('node:process');

console.log(versions);
```

将生成类似于以下的对象：

```console
{ node: '23.0.0',
  acorn: '8.11.3',
  ada: '2.7.8',
  ares: '1.28.1',
  base64: '0.5.2',
  brotli: '1.1.0',
  cjs_module_lexer: '1.2.2',
  cldr: '45.0',
  icu: '75.1',
  llhttp: '9.2.1',
  modules: '127',
  napi: '9',
  nghttp2: '1.61.0',
  nghttp3: '0.7.0',
  ngtcp2: '1.3.0',
  openssl: '3.0.13+quic',
  simdjson: '3.8.0',
  simdutf: '5.2.4',
  sqlite: '3.46.0',
  tz: '2024a',
  undici: '6.13.0',
  unicode: '15.1',
  uv: '1.48.0',
  uvwasi: '0.0.20',
  v8: '12.4.254.14-node.11',
  zlib: '1.3.0.1-motley-7d77fb7' }
```

## 退出码

当没有更多异步操作挂起时，Node.js 通常会以 `0` 状态码退出。在其他情况下使用以下状态码：

* `1` **未捕获的致命异常**：存在未捕获的异常，并且未被域或 [`'uncaughtException'`][] 事件处理程序处理。
* `2`：未使用（由 Bash 保留用于内置误用）
* `3` **内部 JavaScript 解析错误**：Node.js 引导过程中的 JavaScript 源代码内部导致解析错误。这极为罕见，通常只能在 Node.js 本身的开发过程中发生。
* `4` **内部 JavaScript 评估失败**：Node.js 引导过程中的 JavaScript 源代码在评估时未能返回函数值。这极为罕见，通常只能在 Node.js 本身的开发过程中发生。
* `5` **致命错误**：V8 中存在致命不可恢复的错误。通常会打印一条消息到 stderr，前缀为 `FATAL ERROR`。
* `6` **非函数的内部异常处理程序**：存在未捕获的异常，但内部致命异常处理函数不知何故被设置为非函数，无法调用。
* `7` **内部异常处理程序运行时失败**：存在未捕获的异常，并且内部致命异常处理函数本身在尝试处理它时抛出错误。例如，如果 [`'uncaughtException'`][] 或 `domain.on('error')` 处理程序抛出错误，就会发生这种情况。
* `8`：未使用。在 Node.js 的早期版本中，退出码 8 有时表示未捕获的异常。
* `9` **无效参数**：指定了未知选项，或需要值的选项未提供值。
* `10` **内部 JavaScript 运行时失败**：Node.js 引导过程中的 JavaScript 源代码在调用引导函数时抛出错误。这极为罕见，通常只能在 Node.js 本身的开发过程中发生。
* `12` **无效的调试参数**：设置了 `--inspect` 和/或 `--inspect-brk` 选项，但选择的端口号无效或不可用。
* `13` **未解决的顶级 Await**：在顶层代码的函数外部使用了 `await`，但传递的 `Promise` 从未解决。
* `14` **快照失败**：Node.js 启动构建 V8 启动快照但失败，因为应用程序状态的某些要求未满足。
* `>128` **信号退出**：如果 Node.js 收到致命信号，例如 `SIGKILL` 或 `SIGHUP`，则其退出码将为 `128` 加上信号代码的值。这是标准的 POSIX 实践，因为退出码被定义为 7 位整数，而信号退出设置高位，然后包含信号代码的值。例如，信号 `SIGABRT` 的值为 `6`，因此预期的退出码将为 `128` + `6`，即 `134`。

[Advanced serialization for `child_process`]: child_process.md#advanced-serialization
[Android building]: https://github.com/nodejs/node/blob/HEAD/BUILDING.md#android
[Child Process]: child_process.md
[Cluster]: cluster.md
[Duplex]: stream.md#duplex-and-transform-streams
[Event Loop]: https://nodejs.org/en/learn/asynchronous-work/event-loop-timers-and-nexttick#understanding-processnexttick
[LTS]: https://github.com/nodejs/Release
[Permission Model]: permissions.md#permission-model
[Readable]: stream.md#readable-streams
[Signal Events]: #signal-events
[Source Map]: https://tc39.es/ecma426/
[Stream compatibility]: stream.md#compatibility-with-older-nodejs-versions
[TTY]: tty.md#tty
[Writable]: stream.md#writable-streams
[`'exit'`]: #event-exit
[`'message'`]: child_process.md#event-message
[`'uncaughtException'`]: #event-uncaughtexception
[`--no-deprecation`]: cli.md#--no-deprecation
[`--permission`]: cli.md#--permission
[`--unhandled-rejections`]: cli.md#--unhandled-rejectionsmode
[`Buffer`]: buffer.md
[`ChildProcess.disconnect()`]: child_process.md#subprocessdisconnect
[`ChildProcess.send()`]: child_process.md#subprocesssendmessage-sendhandle-options-callback
[`ChildProcess`]: child_process.md#class-childprocess
[`Error`]: errors.md#class-error
[`EventEmitter`]: events.md#class-eventemitter
[`NODE_OPTIONS`]: cli.md#node_optionsoptions
[`Promise.race()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/race
[`Worker`]: worker_threads.md#class-worker
[`Worker` constructor]: worker_threads.md#new-workerfilename-options
[`console.error()`]: console.md#consoleerrordata-args
[`console.log()`]: console.md#consolelogdata-args
[`domain`]: domain.md
[`module.getSourceMapsSupport()`]: module.md#modulegetsourcemapssupport
[`module.isBuiltin(id)`]: module.md#moduleisbuiltinmodulename
[`module.setSourceMapsSupport()`]: module.md#modulesetsourcemapssupportenabled-options
[`net.Server`]: net.md#class-netserver
[`net.Socket`]: net.md#class-netsocket
[`os.constants.dlopen`]: os.md#dlopen-constants
[`postMessageToThread()`]: worker_threads.md#workerpostmessagetothreadthreadid-value-transferlist-timeout
[`process.argv`]: #processargv
[`process.config`]: #processconfig
[`process.execPath`]: #processexecpath
[`process.exit()`]: #processexitcode
[`process.exitCode`]: #processexitcode_1
[`process.hrtime()`]: #processhrtimetime
[`process.hrtime.bigint()`]: #processhrtimebigint
[`process.kill()`]: #processkillpid-signal
[`process.setUncaughtExceptionCaptureCallback()`]: #processsetuncaughtexceptioncapturecallbackfn
[`promise.catch()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Promise/catch
[`queueMicrotask()`]: globals.md#queuemicrotaskcallback
[`readable.read()`]: stream.md#readablereadsize
[`require()`]: globals.md#require
[`require.cache`]: modules.md#requirecache
[`require.main`]: modules.md#accessing-the-main-module
[`subprocess.kill()`]: child_process.md#subprocesskillsignal
[`v8.setFlagsFromString()`]: v8.md#v8setflagsfromstringflags
[built-in modules with mandatory `node:` prefix]: modules.md#built-in-modules-with-mandatory-node-prefix
[debugger]: debugger.md
[deprecation code]: deprecations.md
[loading ECMAScript modules using `require()`]: modules.md#loading-ecmascript-modules-using-require
[nodejs/node#21973]: https://github.com/nodejs/node/issues/21973
[note on process I/O]: #a-note-on-process-io
[process.cpuUsage]: #processcpuusagepreviousvalue
[process_emit_warning]: #processemitwarningwarning-type-code-ctor
[process_warning]: #event-warning
[report documentation]: report.md
[terminal raw mode]: tty.md#readstreamsetrawmodemode
[uv_get_available_memory]: https://docs.libuv.org/en/v1.x/misc.html#c.uv_get_available_memory
[uv_get_constrained_memory]: https://docs.libuv.org/en/v1.x/misc.html#c.uv_get_constrained_memory
[uv_rusage_t]: https://docs.libuv.org/en/v1.x/misc.html#c.uv_rusage_t
[wikipedia_major_fault]: https://en.wikipedia.org/wiki/Page_fault#Major
[wikipedia_minor_fault]: https://en.wikipedia.org/wiki/Page_fault#Minor
