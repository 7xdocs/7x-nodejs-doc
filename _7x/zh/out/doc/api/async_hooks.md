# 异步钩子

<!--introduced_in=v8.1.0-->

> Stability: 1 - 实验性。如果可能，请迁移远离此 API。
> 我们不建议使用 [`createHook`][]、[`AsyncHook`][] 和
> [`executionAsyncResource`][] API，因为它们存在可用性问题、安全风险
> 和性能影响。异步上下文跟踪用例更适合使用
> 稳定的 [`AsyncLocalStorage`][] API。如果你有超出 [`AsyncLocalStorage`][]
> 解决的上下文跟踪需求或 [Diagnostics Channel][] 当前提供的诊断数据
> 之外的 `createHook`、`AsyncHook` 或 `executionAsyncResource` 用例，
> 请在 <https://github.com/nodejs/node/issues> 描述你的用例，
> 以便我们可以创建更专注的 API。

<!-- source_link=lib/async_hooks.js -->

我们强烈反对使用 `async_hooks` API。
可以覆盖其大部分用例的其他 API 包括：

* [`AsyncLocalStorage`][] 跟踪异步上下文
* [`process.getActiveResourcesInfo()`][] 跟踪活动资源

`node:async_hooks` 模块提供了一个用于跟踪异步资源的 API。
可以通过以下方式访问：

```mjs
import async_hooks from 'node:async_hooks';
```

```cjs
const async_hooks = require('node:async_hooks');
```

## 术语

异步资源表示一个具有关联回调的对象。
此回调可能被多次调用，例如 `net.createServer()` 中的 `'connection'` 事件，
或者仅调用一次，如 `fs.open()`。资源也可能在回调调用之前关闭。
`AsyncHook` 不会明确区分这些不同情况，而是将它们表示为资源这一抽象概念。

如果使用了 [`Worker`][]，每个线程都有独立的 `async_hooks` 接口，
并且每个线程将使用一组新的异步 ID。

## 概述

以下是公共 API 的简单概述。

```mjs
import async_hooks from 'node:async_hooks';

// 返回当前执行上下文的 ID。
const eid = async_hooks.executionAsyncId();

// 返回负责触发当前执行范围回调调用的句柄的 ID。
const tid = async_hooks.triggerAsyncId();

// 创建一个新的 AsyncHook 实例。所有这些回调都是可选的。
const asyncHook =
    async_hooks.createHook({ init, before, after, destroy, promiseResolve });

// 允许此 AsyncHook 实例的回调调用。这不是运行构造函数后的隐式
// 操作，必须显式运行以开始执行回调。
asyncHook.enable();

// 禁用监听新的异步事件。
asyncHook.disable();

//
// 以下是可以传递给 createHook() 的回调。
//

// init() 在对象构造期间调用。此回调运行时，资源可能尚未
// 完成构造。因此，由 "asyncId" 引用的资源的所有字段可能尚未填充。
function init(asyncId, type, triggerAsyncId, resource) { }

// before() 在资源回调即将调用之前调用。对于句柄（如 TCPWrap），
// 它可以被调用 0-N 次，对于请求（如 FSReqCallback），它将恰好被调用 1 次。
function before(asyncId) { }

// after() 在资源回调刚刚完成后调用。
function after(asyncId) { }

// destroy() 在资源被销毁时调用。
function destroy(asyncId) { }

// promiseResolve() 仅针对 promise 资源调用，当
// 传递给 Promise 构造函数的 resolve() 函数被调用时
//（直接或通过其他解析 promise 的方式）。
function promiseResolve(asyncId) { }
```

```cjs
const async_hooks = require('node:async_hooks');

// 返回当前执行上下文的 ID。
const eid = async_hooks.executionAsyncId();

// 返回负责触发当前执行范围回调调用的句柄的 ID。
const tid = async_hooks.triggerAsyncId();

// 创建一个新的 AsyncHook 实例。所有这些回调都是可选的。
const asyncHook =
    async_hooks.createHook({ init, before, after, destroy, promiseResolve });

// 允许此 AsyncHook 实例的回调调用。这不是运行构造函数后的隐式
// 操作，必须显式运行以开始执行回调。
asyncHook.enable();

// 禁用监听新的异步事件。
asyncHook.disable();

//
// 以下是可以传递给 createHook() 的回调。
//

// init() 在对象构造期间调用。此回调运行时，资源可能尚未
// 完成构造。因此，由 "asyncId" 引用的资源的所有字段可能尚未填充。
function init(asyncId, type, triggerAsyncId, resource) { }

// before() 在资源回调即将调用之前调用。对于句柄（如 TCPWrap），
// 它可以被调用 0-N 次，对于请求（如 FSReqCallback），它将恰好被调用 1 次。
function before(asyncId) { }

// after() 在资源回调刚刚完成后调用。
function after(asyncId) { }

// destroy() 在资源被销毁时调用。
function destroy(asyncId) { }

// promiseResolve() 仅针对 promise 资源调用，当
// 传递给 Promise 构造函数的 resolve() 函数被调用时
//（直接或通过其他解析 promise 的方式）。
function promiseResolve(asyncId) { }
```

## `async_hooks.createHook(callbacks)`

<!-- YAML
added: v8.1.0
-->

* `callbacks` {Object} 要注册的 [Hook 回调][Hook Callbacks]
  * `init` {Function} [`init` 回调][`init` callback]。
  * `before` {Function} [`before` 回调][`before` callback]。
  * `after` {Function} [`after` 回调][`after` callback]。
  * `destroy` {Function} [`destroy` 回调][`destroy` callback]。
  * `promiseResolve` {Function} [`promiseResolve` 回调][`promiseResolve` callback]。
* 返回：{AsyncHook} 用于禁用和启用钩子的实例

注册函数以在每个异步操作的不同生命周期事件中调用。

回调 `init()`/`before()`/`after()`/`destroy()` 在资源生命周期中相应的异步事件期间被调用。

所有回调都是可选的。例如，如果只需要跟踪资源清理，则只需传递 `destroy` 回调。可以传递给 `callbacks` 的所有函数的详细信息在 [Hook 回调][Hook Callbacks] 部分。

```mjs
import { createHook } from 'node:async_hooks';

const asyncHook = createHook({
  init(asyncId, type, triggerAsyncId, resource) { },
  destroy(asyncId) { },
});
```

```cjs
const async_hooks = require('node:async_hooks');

const asyncHook = async_hooks.createHook({
  init(asyncId, type, triggerAsyncId, resource) { },
  destroy(asyncId) { },
});
```

回调将通过原型链继承：

```js
class MyAsyncCallbacks {
  init(asyncId, type, triggerAsyncId, resource) { }
  destroy(asyncId) {}
}

class MyAddedCallbacks extends MyAsyncCallbacks {
  before(asyncId) { }
  after(asyncId) { }
}

const asyncHook = async_hooks.createHook(new MyAddedCallbacks());
```

因为 promise 是通过 async hooks 机制跟踪其生命周期的异步资源，
所以 `init()`、`before()`、`after()` 和 `destroy()` 回调 _不得_ 是返回 promise 的异步函数。

### 错误处理

如果任何 `AsyncHook` 回调抛出错误，应用程序将打印堆栈跟踪并退出。退出路径遵循未捕获异常的处理方式，但所有 `'uncaughtException'` 监听器都会被移除，从而强制进程退出。除非应用程序使用 `--abort-on-uncaught-exception` 运行，否则 `'exit'` 回调仍将被调用，在这种情况下，将打印堆栈跟踪并且应用程序退出，留下核心文件。

这种错误处理行为的原因在于这些回调在对象生命周期的潜在不稳定点运行，例如在类构造和销毁期间。因此，认为有必要快速终止进程以防止将来出现意外中止。如果执行了全面分析以确保异常可以遵循正常的控制流而不会产生意外的副作用，这一点在未来可能会改变。

### 在 `AsyncHook` 回调中打印

因为打印到控制台是异步操作，`console.log()` 会导致 `AsyncHook` 回调被调用。在 `AsyncHook` 回调函数内部使用 `console.log()` 或类似的异步操作将导致无限递归。在调试时，一个简单的解决方法是使用同步日志操作，例如 `fs.writeFileSync(file, msg, flag)`。这将打印到文件，并且不会递归调用 `AsyncHook`，因为它是同步的。

```mjs
import { writeFileSync } from 'node:fs';
import { format } from 'node:util';

function debug(...args) {
  // 在 AsyncHook 回调内部调试时使用类似此函数的函数
  writeFileSync('log.out', `${format(...args)}\n`, { flag: 'a' });
}
```

```cjs
const fs = require('node:fs');
const util = require('node:util');

function debug(...args) {
  // 在 AsyncHook 回调内部调试时使用类似此函数的函数
  fs.writeFileSync('log.out', `${util.format(...args)}\n`, { flag: 'a' });
}
```

如果日志记录需要异步操作，可以使用 `AsyncHook` 本身提供的信息来跟踪是什么导致了异步操作。然后，当日志记录本身导致调用 `AsyncHook` 回调时，应跳过日志记录。通过这样做，打破了原本的无限递归。

## 类：`AsyncHook`

`AsyncHook` 类公开了一个用于跟踪异步操作生命周期事件的接口。

### `asyncHook.enable()`

* 返回：{AsyncHook} 对 `asyncHook` 的引用

启用给定 `AsyncHook` 实例的回调。如果未提供回调，则启用是无操作。

`AsyncHook` 实例默认是禁用的。如果 `AsyncHook` 实例应在创建后立即启用，可以使用以下模式。

```mjs
import { createHook } from 'node:async_hooks';

const hook = createHook(callbacks).enable();
```

```cjs
const async_hooks = require('node:async_hooks');

const hook = async_hooks.createHook(callbacks).enable();
```

### `asyncHook.disable()`

* 返回：{AsyncHook} 对 `asyncHook` 的引用

从要执行的全局 `AsyncHook` 回调池中禁用给定 `AsyncHook` 实例的回调。一旦钩子被禁用，除非重新启用，否则不会再次调用。

为了 API 一致性，`disable()` 也返回 `AsyncHook` 实例。

### 钩子回调

异步事件生命周期中的关键事件已分为四个领域：实例化、回调调用之前/之后以及实例销毁时。

#### `init(asyncId, type, triggerAsyncId, resource)`

* `asyncId` {number} 异步资源的唯一 ID。
* `type` {string} 异步资源的类型。
* `triggerAsyncId` {number} 在其执行上下文中创建此异步资源的异步资源的唯一 ID。
* `resource` {Object} 代表异步操作的资源的引用，需要在 _destroy_ 期间释放。

当构造一个有可能发出异步事件的类时调用。这 _并不_ 意味着实例必须在 `destroy` 调用之前调用 `before`/`after`，只表示存在这种可能性。

可以通过执行诸如打开资源然后在资源可以使用之前关闭它来观察此行为。以下代码片段演示了这一点。

```mjs
import { createServer } from 'node:net';

createServer().listen(function() { this.close(); });
// 或
clearTimeout(setTimeout(() => {}, 10));
```

```cjs
require('node:net').createServer().listen(function() { this.close(); });
// 或
clearTimeout(setTimeout(() => {}, 10));
```

每个新资源都被分配一个在当前 Node.js 实例范围内唯一的 ID。

##### `type`

`type` 是一个字符串，标识导致调用 `init` 的资源类型。通常，它对应于资源构造函数的名称。

由 Node.js 本身创建的资源的 `type` 在任何 Node.js 版本中都可能更改。有效值包括 `TLSWRAP`、`TCPWRAP`、`TCPSERVERWRAP`、`GETADDRINFOREQWRAP`、`FSREQCALLBACK`、`Microtask` 和 `Timeout`。检查使用的 Node.js 版本的源代码以获取完整列表。

此外，[`AsyncResource`][] 的用户创建独立于 Node.js 本身的异步资源。

还有 `PROMISE` 资源类型，用于跟踪 `Promise` 实例和由它们调度的异步工作。

用户在使用公共嵌入器 API 时能够定义自己的 `type`。

类型名称可能发生冲突。鼓励嵌入器使用唯一前缀，例如 npm 包名称，以防止在监听钩子时发生冲突。

##### `triggerAsyncId`

`triggerAsyncId` 是导致（或“触发”）新资源初始化并导致 `init` 调用的资源的 `asyncId`。这与 `async_hooks.executionAsyncId()` 不同，后者仅显示资源 _何时_ 创建，而 `triggerAsyncId` 显示资源 _为什么_ 创建。

以下是 `triggerAsyncId` 的简单演示：

```mjs
import { createHook, executionAsyncId } from 'node:async_hooks';
import { stdout } from 'node:process';
import net from 'node:net';
import fs from 'node:fs';

createHook({
  init(asyncId, type, triggerAsyncId) {
    const eid = executionAsyncId();
    fs.writeSync(
      stdout.fd,
      `${type}(${asyncId}): trigger: ${triggerAsyncId} execution: ${eid}\n`);
  },
}).enable();

net.createServer((conn) => {}).listen(8080);
```

```cjs
const { createHook, executionAsyncId } = require('node:async_hooks');
const { stdout } = require('node:process');
const net = require('node:net');
const fs = require('node:fs');

createHook({
  init(asyncId, type, triggerAsyncId) {
    const eid = executionAsyncId();
    fs.writeSync(
      stdout.fd,
      `${type}(${asyncId}): trigger: ${triggerAsyncId} execution: ${eid}\n`);
  },
}).enable();

net.createServer((conn) => {}).listen(8080);
```

使用 `nc localhost 8080` 访问服务器时的输出：

```console
TCPSERVERWRAP(5): trigger: 1 execution: 1
TCPWRAP(7): trigger: 5 execution: 0
```

`TCPSERVERWRAP` 是接收连接的服务器。

`TCPWRAP` 是来自客户端的新连接。当建立新连接时，立即构造 `TCPWrap` 实例。这发生在任何 JavaScript 堆栈之外。（`executionAsyncId()` 为 `0` 意味着它正在从 C++ 执行，上方没有 JavaScript 堆栈。）仅凭这些信息，不可能在导致它们创建的原因方面将资源链接在一起，因此 `triggerAsyncId` 被赋予传播负责新资源存在的资源的任务。

##### `resource`

`resource` 是一个对象，代表已初始化的实际异步资源。访问该对象的 API 可能由资源的创建者指定。由 Node.js 本身创建的资源是内部的，可能随时更改。因此没有为这些资源指定 API。

在某些情况下，为了性能原因，资源对象会被重用，因此将其用作 `WeakMap` 中的键或向其添加属性是不安全的。

##### 异步上下文示例

上下文跟踪用例由稳定的 API [`AsyncLocalStorage`][] 覆盖。此示例仅说明 async hooks 的操作，但 [`AsyncLocalStorage`][] 更适合此用例。

以下是一个示例，提供了关于在 `before` 和 `after` 调用之间对 `init` 调用的附加信息，特别是 `listen()` 回调的样子。输出格式稍微复杂一些，以便更容易查看调用上下文。

```mjs
import async_hooks from 'node:async_hooks';
import fs from 'node:fs';
import net from 'node:net';
import { stdout } from 'node:process';
const { fd } = stdout;

let indent = 0;
async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) {
    const eid = async_hooks.executionAsyncId();
    const indentStr = ' '.repeat(indent);
    fs.writeSync(
      fd,
      `${indentStr}${type}(${asyncId}):` +
      ` trigger: ${triggerAsyncId} execution: ${eid}\n`);
  },
  before(asyncId) {
    const indentStr = ' '.repeat(indent);
    fs.writeSync(fd, `${indentStr}before:  ${asyncId}\n`);
    indent += 2;
  },
  after(asyncId) {
    indent -= 2;
    const indentStr = ' '.repeat(indent);
    fs.writeSync(fd, `${indentStr}after:  ${asyncId}\n`);
  },
  destroy(asyncId) {
    const indentStr = ' '.repeat(indent);
    fs.writeSync(fd, `${indentStr}destroy:  ${asyncId}\n`);
  },
}).enable();

net.createServer(() => {}).listen(8080, () => {
  // 在记录服务器启动之前等待 10 毫秒。
  setTimeout(() => {
    console.log('>>>', async_hooks.executionAsyncId());
  }, 10);
});
```

```cjs
const async_hooks = require('node:async_hooks');
const fs = require('node:fs');
const net = require('node:net');
const { fd } = process.stdout;

let indent = 0;
async_hooks.createHook({
  init(asyncId, type, triggerAsyncId) {
    const eid = async_hooks.executionAsyncId();
    const indentStr = ' '.repeat(indent);
    fs.writeSync(
      fd,
      `${indentStr}${type}(${asyncId}):` +
      ` trigger: ${triggerAsyncId} execution: ${eid}\n`);
  },
  before(asyncId) {
    const indentStr = ' '.repeat(indent);
    fs.writeSync(fd, `${indentStr}before:  ${asyncId}\n`);
    indent += 2;
  },
  after(asyncId) {
    indent -= 2;
    const indentStr = ' '.repeat(indent);
    fs.writeSync(fd, `${indentStr}after:  ${asyncId}\n`);
  },
  destroy(asyncId) {
    const indentStr = ' '.repeat(indent);
    fs.writeSync(fd, `${indentStr}destroy:  ${asyncId}\n`);
  },
}).enable();

net.createServer(() => {}).listen(8080, () => {
  // 在记录服务器启动之前等待 10 毫秒。
  setTimeout(() => {
    console.log('>>>', async_hooks.executionAsyncId());
  }, 10);
});
```

仅启动服务器时的输出：

```console
TCPSERVERWRAP(5): trigger: 1 execution: 1
TickObject(6): trigger: 5 execution: 1
before:  6
  Timeout(7): trigger: 6 execution: 6
after:   6
destroy: 6
before:  7
>>> 7
  TickObject(8): trigger: 7 execution: 7
after:   7
before:  8
after:   8
```

如示例所示，`executionAsyncId()` 和 `execution` 分别指定当前执行上下文的值；该值由对 `before` 和 `after` 的调用划定。

仅使用 `execution` 来图形化资源分配结果如下：

```console
  root(1)
     ^
     |
TickObject(6)
     ^
     |
 Timeout(7)
```

`TCPSERVERWRAP` 不是此图的一部分，即使它是调用 `console.log()` 的原因。这是因为在没有主机名的情况下绑定到端口是 _同步_ 操作，但为了维护完全异步的 API，用户的回调被放置在 `process.nextTick()` 中。这就是为什么 `TickObject` 出现在输出中并且是 `.listen()` 回调的“父级”。

该图仅显示资源 _何时_ 创建，而不是 _为什么_，因此要跟踪 _为什么_，请使用 `triggerAsyncId`。这可以用以下图表示：

```console
 bootstrap(1)
     |
     ˅
TCPSERVERWRAP(5)
     |
     ˅
 TickObject(6)
     |
     ˅
  Timeout(7)
```

#### `before(asyncId)`

* `asyncId` {number}

当异步操作启动（例如 TCP 服务器接收新连接）或完成（例如将数据写入磁盘）时，会调用回调以通知用户。`before` 回调就在所述回调执行之前调用。`asyncId` 是分配给即将执行回调的资源的唯一标识符。

`before` 回调将被调用 0 到 N 次。如果异步操作被取消，或者例如 TCP 服务器没有接收到连接，`before` 回调通常会被调用 0 次。持久的异步资源（如 TCP 服务器）通常会多次调用 `before` 回调，而其他操作如 `fs.open()` 只会调用一次。

#### `after(asyncId)`

* `asyncId` {number}

在 `before` 中指定的回调完成后立即调用。

如果在回调执行期间发生未捕获的异常，则 `after` 将在 `'uncaughtException'` 事件发出或 `domain` 的处理程序运行 _之后_ 运行。

#### `destroy(asyncId)`

* `asyncId` {number}

在与 `asyncId` 对应的资源被销毁后调用。它也从嵌入器 API `emitDestroy()` 异步调用。

某些资源依赖垃圾收集进行清理，因此如果对传递给 `init` 的 `resource` 对象进行了引用，则 `destroy` 可能永远不会被调用，导致应用程序中的内存泄漏。如果资源不依赖垃圾收集，则这将不是问题。

使用 destroy 钩子会导致额外的开销，因为它通过垃圾收集器启用对 `Promise` 实例的跟踪。

#### `promiseResolve(asyncId)`

<!-- YAML
added: v8.6.0
-->

* `asyncId` {number}

当传递给 `Promise` 构造函数的 `resolve` 函数被调用时调用（直接或通过其他解析 promise 的方式）。

`resolve()` 不会执行任何可观察的同步工作。

此时 `Promise` 不一定已兑现或拒绝，如果 `Promise` 是通过假设另一个 `Promise` 的状态来解析的。

```js
new Promise((resolve) => resolve(true)).then((a) => {});
```

调用以下回调：

```text
init for PROMISE with id 5, trigger id: 1
  promise resolve 5      # 对应于 resolve(true)
init for PROMISE with id 6, trigger id: 5  # then() 返回的 Promise
  before 6               # 进入 then() 回调
  promise resolve 6      # then() 回调通过返回解析 promise
  after 6
```

### `async_hooks.executionAsyncResource()`

<!-- YAML
added:
 - v13.9.0
 - v12.17.0
-->

* 返回：{Object} 代表当前执行的资源。用于在资源内存储数据。

由 `executionAsyncResource()` 返回的资源对象通常是具有未文档化 API 的内部 Node.js 句柄对象。使用对象上的任何函数或属性很可能导致应用程序崩溃，应避免。

在顶级执行上下文中使用 `executionAsyncResource()` 将返回一个空对象，因为没有句柄或请求对象可用，但拥有一个代表顶级的对象可能会有所帮助。

```mjs
import { open } from 'node:fs';
import { executionAsyncId, executionAsyncResource } from 'node:async_hooks';

console.log(executionAsyncId(), executionAsyncResource());  // 1 {}
open(new URL(import.meta.url), 'r', (err, fd) => {
  console.log(executionAsyncId(), executionAsyncResource());  // 7 FSReqWrap
});
```

```cjs
const { open } = require('node:fs');
const { executionAsyncId, executionAsyncResource } = require('node:async_hooks');

console.log(executionAsyncId(), executionAsyncResource());  // 1 {}
open(__filename, 'r', (err, fd) => {
  console.log(executionAsyncId(), executionAsyncResource());  // 7 FSReqWrap
});
```

这可用于实现连续本地存储，而无需使用跟踪 `Map` 来存储元数据：

```mjs
import { createServer } from 'node:http';
import {
  executionAsyncId,
  executionAsyncResource,
  createHook,
} from 'node:async_hooks';
const sym = Symbol('state'); // 私有符号以避免污染

createHook({
  init(asyncId, type, triggerAsyncId, resource) {
    const cr = executionAsyncResource();
    if (cr) {
      resource[sym] = cr[sym];
    }
  },
}).enable();

const server = createServer((req, res) => {
  executionAsyncResource()[sym] = { state: req.url };
  setTimeout(function() {
    res.end(JSON.stringify(executionAsyncResource()[sym]));
  }, 100);
}).listen(3000);
```

```cjs
const { createServer } = require('node:http');
const {
  executionAsyncId,
  executionAsyncResource,
  createHook,
} = require('node:async_hooks');
const sym = Symbol('state'); // 私有符号以避免污染

createHook({
  init(asyncId, type, triggerAsyncId, resource) {
    const cr = executionAsyncResource();
    if (cr) {
      resource[sym] = cr[sym];
    }
  },
}).enable();

const server = createServer((req, res) => {
  executionAsyncResource()[sym] = { state: req.url };
  setTimeout(function() {
    res.end(JSON.stringify(executionAsyncResource()[sym]));
  }, 100);
}).listen(3000);
```

### `async_hooks.executionAsyncId()`

<!-- YAML
added: v8.1.0
changes:
  - version: v8.2.0
    pr-url: https://github.com/nodejs/node/pull/13490
    description: Renamed from `currentId`.
-->

* 返回：{number} 当前执行上下文的 `asyncId`。用于跟踪何时调用某些内容。

```mjs
import { executionAsyncId } from 'node:async_hooks';
import fs from 'node:fs';

console.log(executionAsyncId());  // 1 - 引导
const path = '.';
fs.open(path, 'r', (err, fd) => {
  console.log(executionAsyncId());  // 6 - open()
});
```

```cjs
const async_hooks = require('node:async_hooks');
const fs = require('node:fs');

console.log(async_hooks.executionAsyncId());  // 1 - 引导
const path = '.';
fs.open(path, 'r', (err, fd) => {
  console.log(async_hooks.executionAsyncId());  // 6 - open()
});
```

从 `executionAsyncId()` 返回的 ID 与执行时序相关，而不是因果关系（由 `triggerAsyncId()` 覆盖）：

```js
const server = net.createServer((conn) => {
  // 返回服务器的 ID，而不是新连接的 ID，因为
  // 回调在服务器的 MakeCallback() 执行范围内运行。
  async_hooks.executionAsyncId();

}).listen(port, () => {
  // 返回 TickObject (process.nextTick()) 的 ID，因为所有
  // 传递给 .listen() 的回调都包装在 nextTick() 中。
  async_hooks.executionAsyncId();
});
```

Promise 上下文默认可能无法获得精确的 `executionAsyncId`。请参阅 [promise 执行跟踪][promise execution tracking] 部分。

### `async_hooks.triggerAsyncId()`

* 返回：{number} 负责调用当前正在执行的回调的资源的 ID。

```js
const server = net.createServer((conn) => {
  // 导致（或触发）此回调被调用的资源
  // 是新连接的资源。因此 triggerAsyncId() 的返回值
  // 是 "conn" 的 asyncId。
  async_hooks.triggerAsyncId();

}).listen(port, () => {
  // 即使所有传递给 .listen() 的回调都包装在 nextTick() 中
  // 回调本身存在是因为对服务器的 .listen() 调用
  // 被发出。所以返回值将是服务器的 ID。
  async_hooks.triggerAsyncId();
});
```

Promise 上下文默认可能无法获得有效的 `triggerAsyncId`。请参阅 [promise 执行跟踪][promise execution tracking] 部分。

### `async_hooks.asyncWrapProviders`

<!-- YAML
added:
  - v17.2.0
  - v16.14.0
-->

* 返回：提供程序类型到相应数字 id 的映射。此映射包含 `async_hooks.init()` 事件可能发出的所有事件类型。

此特性抑制了已弃用的 `process.binding('async_wrap').Providers` 的使用。参见：[DEP0111][]

## Promise 执行跟踪

默认情况下，由于 V8 提供的 [promise 内省 API][PromiseHooks] 相对昂贵，不会为 promise 执行分配 `asyncId`。这意味着使用 promise 或 `async`/`await` 的程序默认不会为 promise 回调上下文获得正确的执行和触发 id。

```mjs
import { executionAsyncId, triggerAsyncId } from 'node:async_hooks';

Promise.resolve(1729).then(() => {
  console.log(`eid ${executionAsyncId()} tid ${triggerAsyncId()}`);
});
// 产生：
// eid 1 tid 0
```

```cjs
const { executionAsyncId, triggerAsyncId } = require('node:async_hooks');

Promise.resolve(1729).then(() => {
  console.log(`eid ${executionAsyncId()} tid ${triggerAsyncId()}`);
});
// 产生：
// eid 1 tid 0
```

观察到 `then()` 回调声称在外部范围的上下文中执行，即使涉及异步跳转。此外，`triggerAsyncId` 值为 `0`，这意味着我们缺少关于导致（触发）`then()` 回调执行的资源的上下文。

通过 `async_hooks.createHook` 安装 async hooks 可以启用 promise 执行跟踪：

```mjs
import { createHook, executionAsyncId, triggerAsyncId } from 'node:async_hooks';
createHook({ init() {} }).enable(); // 强制启用 PromiseHooks。
Promise.resolve(1729).then(() => {
  console.log(`eid ${executionAsyncId()} tid ${triggerAsyncId()}`);
});
// 产生：
// eid 7 tid 6
```

```cjs
const { createHook, executionAsyncId, triggerAsyncId } = require('node:async_hooks');

createHook({ init() {} }).enable(); // 强制启用 PromiseHooks。
Promise.resolve(1729).then(() => {
  console.log(`eid ${executionAsyncId()} tid ${triggerAsyncId()}`);
});
// 产生：
// eid 7 tid 6
```

在此示例中，添加任何实际的钩子函数启用了对 promise 的跟踪。上面示例中有两个 promise；由 `Promise.resolve()` 创建的 promise 和调用 `then()` 返回的 promise。在上面的示例中，第一个 promise 获得了 `asyncId` `6`，后者获得了 `asyncId` `7`。在执行 `then()` 回调期间，我们正在 `asyncId` 为 `7` 的 promise 的上下文中执行。此 promise 由异步资源 `6` 触发。

关于 promise 的另一个微妙之处是，`before` 和 `after` 回调仅在链式 promise 上运行。这意味着不是由 `then()`/`catch()` 创建的 promise 不会在其上触发 `before` 和 `after` 回调。有关更多详细信息，请参阅 V8 [PromiseHooks][] API 的详细信息。

## JavaScript 嵌入器 API

处理自己的异步资源执行 I/O、连接池或管理回调队列等任务的库开发人员可以使用 `AsyncResource` JavaScript API，以便调用所有适当的回调。

### 类：`AsyncResource`

此类的文档已移至 [`AsyncResource`][]。

## 类：`AsyncLocalStorage`

此类的文档已移至 [`AsyncLocalStorage`][]。

[DEP0111]: deprecations.md#dep0111-processbinding
[Diagnostics Channel]: diagnostics_channel.md
[Hook Callbacks]: #hook-callbacks
[PromiseHooks]: https://docs.google.com/document/d/1rda3yKGHimKIhg5YeoAmCOtyURgsbTH_qaYR79FELlk/edit
[`AsyncHook`]: #class-asynchook
[`AsyncLocalStorage`]: async_context.md#class-asynclocalstorage
[`AsyncResource`]: async_context.md#class-asyncresource
[`Worker`]: worker_threads.md#class-worker
[`after` callback]: #afterasyncid
[`before` callback]: #beforeasyncid
[`createHook`]: #async_hookscreatehookcallbacks
[`destroy` callback]: #destroyasyncid
[`executionAsyncResource`]: #async_hooksexecutionasyncresource
[`init` callback]: #initasyncid-type-triggerasyncid-resource
[`process.getActiveResourcesInfo()`]: process.md#processgetactiveresourcesinfo
[`promiseResolve` callback]: #promiseresolveasyncid
[promise execution tracking]: #promise-execution-tracking