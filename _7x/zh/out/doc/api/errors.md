# Errors（错误）

<!--introduced_in=v4.0.0-->

<!--type=misc-->

在 Node.js 中运行的应用程序通常会遇到以下几类错误：

* 标准 JavaScript 错误，例如 {EvalError}、{SyntaxError}、{RangeError}、{ReferenceError}、{TypeError} 和 {URIError}。
* 标准 `DOMException` 错误。
* 由底层操作系统约束触发的系统错误，例如尝试打开不存在的文件或尝试通过已关闭的套接字发送数据。
* `AssertionError` 是一类特殊的错误，当 Node.js 检测到不应发生的异常逻辑违规时会触发。这些通常由 `node:assert` 模块引发。
* 由应用程序代码触发的用户指定错误。

Node.js 引发的所有 JavaScript 和系统错误都继承自标准 JavaScript {Error} 类，或者是该类的实例，并保证 _至少_ 提供该类上可用的属性。

Node.js 引发的错误的 [`error.message`][] 属性可能在任意版本中更改。请使用 [`error.code`][] 来识别错误。对于 `DOMException`，使用 [`domException.name`][] 来识别其类型。

## 错误传播与拦截

<!--type=misc-->

Node.js 支持多种机制来传播和处理应用程序运行时发生的错误。这些错误的报告和处理方式完全取决于 `Error` 的类型和所调用 API 的风格。

所有 JavaScript 错误都作为异常处理，使用标准的 JavaScript `throw` 机制 _立即_ 生成并抛出错误。这些错误使用 JavaScript 语言提供的 [`try…catch` 结构][try-catch] 来处理。

```js
// 由于 z 未定义，抛出 ReferenceError。
try {
  const m = 1;
  const n = m + z;
} catch (err) {
  // 在此处理错误。
}
```

任何使用 JavaScript `throw` 机制的操作都会引发异常，该异常 _必须_ 被处理，否则 Node.js 进程将立即退出。

除了少数例外，_同步_ API（任何不返回 {Promise} 也不接受 `callback` 函数的阻塞方法，例如 [`fs.readFileSync`][]）将使用 `throw` 来报告错误。

在 _异步 API_ 中发生的错误可能通过多种方式报告：

* 一些异步方法返回 {Promise}，你应该始终考虑到它可能会被拒绝。有关进程对未处理的 Promise 拒绝如何反应，请参见 [`--unhandled-rejections`][] 标志。

  <!-- eslint-disable no-useless-return -->

  ```js
  const fs = require('node:fs/promises');

  (async () => {
    let data;
    try {
      data = await fs.readFile('a file that does not exist');
    } catch (err) {
      console.error('读取文件时出错！', err);
      return;
    }
    // 否则处理数据
  })();
  ```

* 大多数接受 `callback` 函数的异步方法会将 `Error` 对象作为该函数的第一个参数传递。如果第一个参数不是 `null` 且是 `Error` 的实例，则表示发生了应处理的错误。

  <!-- eslint-disable no-useless-return -->

  ```js
  const fs = require('node:fs');
  fs.readFile('a file that does not exist', (err, data) => {
    if (err) {
      console.error('读取文件时出错！', err);
      return;
    }
    // 否则处理数据
  });
  ```

* 在作为 [`EventEmitter`][] 的对象上调用异步方法时，错误可以被路由到该对象的 `'error'` 事件。

  ```js
  const net = require('node:net');
  const connection = net.connect('localhost');

  // 向流添加 'error' 事件处理程序：
  connection.on('error', (err) => {
    // 如果连接被服务器重置，或根本无法连接，或连接遇到任何错误，错误将发送到此。
    console.error(err);
  });

  connection.pipe(process.stdout);
  ```

* Node.js API 中少数通常为异步的方法可能仍使用 `throw` 机制来引发必须使用 `try…catch` 处理的异常。没有此类方法的完整列表；请参考每个方法的文档以确定所需的适当错误处理机制。

`'error'` 事件机制的使用在[基于流的][stream-based]和[基于事件发射器的][event emitter-based] API 中最常见，这些 API 本身代表一系列随时间推移的异步操作（与可能通过或失败的单一操作相对）。

对于 _所有_ [`EventEmitter`][] 对象，如果未提供 `'error'` 事件处理程序，错误将被抛出，导致 Node.js 进程报告未捕获的异常并崩溃，除非：要么已经为 [`'uncaughtException'`][] 事件注册了处理程序，要么使用了已弃用的 [`node:domain`][domains] 模块。

```js
const EventEmitter = require('node:events');
const ee = new EventEmitter();

setImmediate(() => {
  // 这将使进程崩溃，因为未添加 'error' 事件处理程序。
  ee.emit('error', new Error('这将导致崩溃'));
});
```

以这种方式产生的错误 _无法_ 使用 `try…catch` 拦截，因为它们在调用代码已经退出 _之后_ 抛出。

开发人员必须参考每个方法的文档以确切了解这些方法引发的错误是如何传播的。

## 类：`Error`

<!--type=class-->

通用的 JavaScript {Error} 对象，不表示错误发生的任何具体情况。`Error` 对象捕获了一个“栈追踪”，详细说明了 `Error` 被实例化的代码点，并可能提供错误的文本描述。

Node.js 生成的所有错误，包括所有系统和 JavaScript 错误，都将是 `Error` 类的实例或继承自该类。

### `new Error(message[, options])`

* `message` {string}
* `options` {Object}
  * `cause` {any} 导致新创建错误的原因错误。

创建一个新的 `Error` 对象并将 `error.message` 属性设置为提供的文本消息。如果传递了一个对象作为 `message`，则通过调用 `String(message)` 生成文本消息。如果提供了 `cause` 选项，它将被赋值给 `error.cause` 属性。`error.stack` 属性将表示在代码中调用 `new Error()` 的点。栈追踪依赖于 [V8 的栈追踪 API][]。栈追踪仅扩展到 (a) _同步代码执行_ 的开始，或 (b) 属性 `Error.stackTraceLimit` 给出的帧数，以较小者为准。

### `Error.captureStackTrace(targetObject[, constructorOpt])`

* `targetObject` {Object}
* `constructorOpt` {Function}

在 `targetObject` 上创建一个 `.stack` 属性，当访问时返回一个字符串，表示在代码中调用 `Error.captureStackTrace()` 的位置。

```js
const myObject = {};
Error.captureStackTrace(myObject);
myObject.stack;  // 类似于 `new Error().stack`
```

追踪的第一行将以 `${myObject.name}: ${myObject.message}` 为前缀。

可选的 `constructorOpt` 参数接受一个函数。如果给定，则 `constructorOpt` 以上的所有帧，包括 `constructorOpt`，将从生成的栈追踪中省略。

`constructorOpt` 参数对于向用户隐藏错误生成的实现细节非常有用。例如：

```js
function a() {
  b();
}

function b() {
  c();
}

function c() {
  // 创建一个没有栈追踪的错误以避免计算两次栈追踪。
  const { stackTraceLimit } = Error;
  Error.stackTraceLimit = 0;
  const error = new Error();
  Error.stackTraceLimit = stackTraceLimit;

  // 捕获函数 b 以上的栈追踪
  Error.captureStackTrace(error, b); // 函数 c 和 b 都不包含在栈追踪中
  throw error;
}

a();
```

### `Error.stackTraceLimit`

* 类型：{number}

`Error.stackTraceLimit` 属性指定了栈追踪收集的栈帧数量（无论是通过 `new Error().stack` 还是 `Error.captureStackTrace(obj)` 生成）。

默认值为 `10`，但可以设置为任何有效的 JavaScript 数字。更改将影响在值更改 _之后_ 捕获的任何栈追踪。

如果设置为非数字值，或设置为负数，栈追踪将不会捕获任何帧。

### `error.cause`

<!-- YAML
added: v16.9.0
-->

* 类型：{any}

如果存在，`error.cause` 属性是 `Error` 的根本原因。它在捕获错误并抛出具有不同消息或代码的新错误时使用，以便仍然能够访问原始错误。

`error.cause` 属性通常通过调用 `new Error(message, { cause })` 来设置。如果未提供 `cause` 选项，则构造函数不会设置此属性。

此属性允许错误链式连接。当序列化 `Error` 对象时，如果设置了 `error.cause`，[`util.inspect()`][] 会递归地序列化它。

```js
const cause = new Error('远程 HTTP 服务器响应了 500 状态码');
const symptom = new Error('消息发送失败', { cause });

console.log(symptom);
// 打印：
//   Error: 消息发送失败
//       at REPL2:1:17
//       at Script.runInThisContext (node:vm:130:12)
//       ... 7 行匹配原因栈追踪 ...
//       at [_line] [as _line] (node:internal/readline/interface:886:18) {
//     [cause]: Error: 远程 HTTP 服务器响应了 500 状态码
//         at REPL1:1:15
//         at Script.runInThisContext (node:vm:130:12)
//         at REPLServer.defaultEval (node:repl:574:29)
//         at bound (node:domain:426:15)
//         at REPLServer.runBound [as eval] (node:domain:437:12)
//         at REPLServer.onLine (node:repl:902:10)
//         at REPLServer.emit (node:events:549:35)
//         at REPLServer.emit (node:domain:482:12)
//         at [_onLine] [as _onLine] (node:internal/readline/interface:425:12)
//         at [_line] [as _line] (node:internal/readline/interface:886:18)
```

### `error.code`

* 类型：{string}

`error.code` 属性是一个字符串标签，用于标识错误的类型。`error.code` 是识别错误最稳定的方式。它只会在 Node.js 的主要版本之间更改。相比之下，`error.message` 字符串可能在 Node.js 的任何版本之间更改。有关特定代码的详细信息，请参见 [Node.js 错误代码][]。

### `error.message`

* 类型：{string}

`error.message` 属性是通过调用 `new Error(message)` 设置的错误字符串描述。传递给构造函数的 `message` 也会出现在 `Error` 栈追踪的第一行，但是在 `Error` 对象创建后更改此属性 _可能不会_ 更改栈追踪的第一行（例如，在更改此属性之前读取了 `error.stack`）。

```js
const err = new Error('消息');
console.error(err.message);
// 打印：消息
```

### `error.stack`

* 类型：{string}

`error.stack` 属性是一个字符串，描述了实例化 `Error` 的代码点。

```console
Error: Things keep happening!
   at /home/gbusey/file.js:525:2
   at Frobnicator.refrobulate (/home/gbusey/business-logic.js:424:21)
   at Actor.<anonymous> (/home/gbusey/actors.js:400:8)
   at increaseSynergy (/home/gbusey/actors.js:701:6)
```

第一行格式为 `<error class name>: <error message>`，后跟一系列栈帧（每行以 "at " 开头）。每帧描述了导致错误生成的代码中的调用站点。V8 尝试为每个函数显示一个名称（通过变量名、函数名或对象方法名），但偶尔它无法找到合适的名称。如果 V8 无法确定函数的名称，则仅显示该帧的位置信息。否则，确定的函数名将显示，并在括号中附加位置信息。

帧仅针对 JavaScript 函数生成。例如，如果执行同步地通过一个名为 `cheetahify` 的 C++ 插件函数，而该函数本身调用了一个 JavaScript 函数，则代表 `cheetahify` 调用的帧将不会出现在栈追踪中：

```js
const cheetahify = require('./native-binding.node');

function makeFaster() {
  // `cheetahify()` *同步地* 调用 speedy。
  cheetahify(function speedy() {
    throw new Error('oh no!');
  });
}

makeFaster();
// 将抛出：
//   /home/gbusey/file.js:6
//       throw new Error('oh no!');
//           ^
//   Error: oh no!
//       at speedy (/home/gbusey/file.js:6:11)
//       at makeFaster (/home/gbusey/file.js:5:3)
//       at Object.<anonymous> (/home/gbusey/file.js:10:1)
//       at Module._compile (module.js:456:26)
//       at Object.Module._extensions..js (module.js:474:10)
//       at Module.load (module.js:356:32)
//       at Function.Module._load (module.js:312:12)
//       at Function.Module.runMain (module.js:497:10)
//       at startup (node.js:119:16)
//       at node.js:906:3
```

位置信息将是以下之一：

* `native`，如果帧表示对 V8 内部的调用（如 `[].forEach`）。
* `plain-filename.js:line:column`，如果帧表示对 Node.js 内部的调用。
* `/absolute/path/to/file.js:line:column`，如果帧表示用户程序（使用 CommonJS 模块系统）或其依赖项中的调用。
* `<transport-protocol>:///url/to/module/file.mjs:line:column`，如果帧表示用户程序（使用 ES 模块系统）或其依赖项中的调用。

表示栈追踪的字符串在 **访问** `error.stack` 属性时 **延迟生成**。

栈追踪捕获的帧数受 `Error.stackTraceLimit` 或当前事件循环滴答中可用帧数的较小值限制。

## 类：`AssertionError`

* 扩展自：{errors.Error}

表示断言失败。有关详细信息，请参见 [`Class: assert.AssertionError`][]。

## 类：`RangeError`

* 扩展自：{errors.Error}

表示提供的参数不在函数可接受值的集合或范围内；无论是数值范围，还是给定函数参数选项集之外。

```js
require('node:net').connect(-1);
// 抛出 "RangeError: "port" 选项应 >= 0 且 < 65536: -1"
```

Node.js 将 _立即_ 生成并抛出 `RangeError` 实例作为参数验证的一种形式。

## 类：`ReferenceError`

* 扩展自：{errors.Error}

表示尝试访问未定义的变量。此类错误通常表示代码中的拼写错误或其他程序损坏。

虽然客户端代码可能会生成和传播这些错误，但在实践中，只有 V8 会这样做。

```js
doesNotExist;
// 抛出 ReferenceError，doesNotExist 不是此程序中的变量。
```

除非应用程序动态生成并运行代码，否则 `ReferenceError` 实例表示代码或其依赖项中存在错误。

## 类：`SyntaxError`

* 扩展自：{errors.Error}

表示程序不是有效的 JavaScript。这些错误可能仅作为代码评估的结果生成和传播。代码评估可能由于 `eval`、`Function`、`require` 或 [vm][] 而发生。这些错误几乎总是表明程序已损坏。

```js
try {
  require('node:vm').runInThisContext('binary ! isNotOk');
} catch (err) {
  // 'err' 将是 SyntaxError。
}
```

`SyntaxError` 实例在创建它们的上下文中是不可恢复的——它们只能被其他上下文捕获。

## 类：`SystemError`

* 扩展自：{errors.Error}

当异常发生在 Node.js 运行时环境中时，Node.js 会生成系统错误。这通常发生在应用程序违反操作系统约束时。例如，如果应用程序尝试读取不存在的文件，将发生系统错误。

* `address` {string} 如果存在，网络连接失败的地址
* `code` {string} 字符串错误代码
* `dest` {string} 如果存在，报告文件系统错误时的文件路径目标
* `errno` {number} 系统提供的错误号
* `info` {Object} 如果存在，关于错误条件的额外详细信息
* `message` {string} 系统提供的错误描述（人类可读）
* `path` {string} 如果存在，报告文件系统错误时的文件路径
* `port` {number} 如果存在，不可用的网络连接端口
* `syscall` {string} 触发错误的系统调用的名称

### `error.address`

* 类型：{string}

如果存在，`error.address` 是一个字符串，描述网络连接失败的地址。

### `error.code`

* 类型：{string}

`error.code` 属性是一个字符串，表示错误代码。

### `error.dest`

* 类型：{string}

如果存在，`error.dest` 是报告文件系统错误时的文件路径目标。

### `error.errno`

* 类型：{number}

`error.errno` 属性是一个负数，对应于 [`libuv Error handling`][] 中定义的错误代码。

在 Windows 上，系统提供的错误号将由 libuv 规范化。

要获取错误代码的字符串表示，请使用 [`util.getSystemErrorName(error.errno)`][]。

### `error.info`

* 类型：{Object}

如果存在，`error.info` 是一个包含错误条件详细信息的对象。

### `error.message`

* 类型：{string}

`error.message` 是系统提供的错误描述（人类可读）。

### `error.path`

* 类型：{string}

如果存在，`error.path` 是一个包含相关无效路径名的字符串。

### `error.port`

* 类型：{number}

如果存在，`error.port` 是不可用的网络连接端口。

### `error.syscall`

* 类型：{string}

`error.syscall` 属性是一个字符串，描述失败的 [syscall][]。

### 常见系统错误

这是在编写 Node.js 程序时经常遇到的系统错误列表。有关完整列表，请参见 [`errno`(3) man page][]。

* `EACCES`（权限被拒绝）：尝试以文件访问权限禁止的方式访问文件。

* `EADDRINUSE`（地址已被使用）：尝试将服务器（[`net`][]、[`http`][] 或 [`https`][]）绑定到本地地址失败，因为本地系统上的另一个服务器已经占用了该地址。

* `ECONNREFUSED`（连接被拒绝）：无法建立连接，因为目标机器主动拒绝。这通常是由于尝试连接到外部主机上未激活的服务所致。

* `ECONNRESET`（连接被对端重置）：连接被对端强制关闭。这通常是由于远程套接字因超时或重启导致连接丢失。通常通过 [`http`][] 和 [`net`][] 模块遇到。

* `EEXIST`（文件已存在）：现有文件是要求目标不存在的操作的目标。

* `EISDIR`（是一个目录）：操作期望一个文件，但给定的路径名是一个目录。

* `EMFILE`（系统中打开的文件过多）：系统允许的[文件描述符][]的最大数量已达到，在至少关闭一个之前，无法满足另一个描述符的请求。这在同时并行打开许多文件时遇到，特别是在那些进程文件描述符限制较低的系统（特别是 macOS）上。要补救低限制，请在运行 Node.js 进程的同一 shell 中运行 `ulimit -n 2048`。

* `ENOENT`（没有这样的文件或目录）：通常由 [`fs`][] 操作引发，表示指定路径名的组件不存在。给定路径找不到任何实体（文件或目录）。

* `ENOTDIR`（不是目录）：给定路径名的组件存在，但不是预期的目录。通常由 [`fs.readdir`][] 引发。

* `ENOTEMPTY`（目录非空）：包含条目的目录是需要空目录的操作的目标，通常是 [`fs.unlink`][]。

* `ENOTFOUND`（DNS 查找失败）：表示 `EAI_NODATA` 或 `EAI_NONAME` 的 DNS 失败。这不是标准的 POSIX 错误。

* `EPERM`（操作不被允许）：尝试执行需要提升权限的操作。

* `EPIPE`（管道破裂）：在管道、套接字或 FIFO 上进行写入，但没有进程读取数据。通常在 [`net`][] 和 [`http`][] 层遇到，表示写入的流的远程端已关闭。

* `ETIMEDOUT`（操作超时）：连接或发送请求失败，因为连接方在一段时间后未正确响应。通常由 [`http`][] 或 [`net`][] 遇到。通常表示未正确调用 `socket.end()`。

## 类：`TypeError`

* 扩展自 {errors.Error}

表示提供的参数不是允许的类型。例如，将函数传递给期望字符串的参数将是一个 `TypeError`。

```js
require('node:url').parse(() => { });
// 抛出 TypeError，因为它期望一个字符串。
```

Node.js 将 _立即_ 生成并抛出 `TypeError` 实例作为参数验证的一种形式。

## 异常与错误

<!--type=misc-->

JavaScript 异常是由于无效操作或作为 `throw` 语句的目标而抛出的值。虽然不要求这些值是 `Error` 的实例或继承自 `Error`，但 Node.js 或 JavaScript 运行时抛出的所有异常 _都将是_ `Error` 的实例。

一些异常在 JavaScript 层是 _不可恢复的_。此类异常将 _总是_ 导致 Node.js 进程崩溃。示例包括 C++ 层中的 `assert()` 检查或 `abort()` 调用。

## OpenSSL 错误

源自 `crypto` 或 `tls` 的错误属于 `Error` 类，除了标准的 `.code` 和 `.message` 属性外，可能还有一些额外的 OpenSSL 特定属性。

### `error.opensslErrorStack`

一个错误数组，可以提供错误在 OpenSSL 库中起源位置的上下文。

### `error.function`

错误起源的 OpenSSL 函数。

### `error.library`

错误起源的 OpenSSL 库。

### `error.reason`

描述错误原因的人类可读字符串。

<a id="nodejs-error-codes"></a>

## Node.js 错误代码

<a id="ABORT_ERR"></a>

### `ABORT_ERR`

<!-- YAML
added: v15.0.0
-->

当操作被中止时使用（通常使用 `AbortController`）。

_不_ 使用 `AbortSignal` 的 API 通常不会引发带有此代码的错误。

此代码不使用 Node.js 错误使用的常规 `ERR_*` 约定，以便与 Web 平台的 `AbortError` 兼容。

<a id="ERR_ACCESS_DENIED"></a>

### `ERR_ACCESS_DENIED`

一种特殊类型的错误，每当 Node.js 尝试访问受[权限模型][Permission Model]限制的资源时触发。

<a id="ERR_AMBIGUOUS_ARGUMENT"></a>

### `ERR_AMBIGUOUS_ARGUMENT`

函数参数的使用方式暗示函数签名可能被误解。当 `node:assert` 模块中的 `assert.throws(block, message)` 的 `message` 参数与 `block` 抛出的错误消息匹配时，会抛出此错误，因为这种用法表明用户认为 `message` 是预期消息，而不是如果 `block` 未抛出时 `AssertionError` 将显示的消息。

<a id="ERR_ARG_NOT_ITERABLE"></a>

### `ERR_ARG_NOT_ITERABLE`

需要可迭代参数（即适用于 `for...of` 循环的值），但未提供给 Node.js API。

<a id="ERR_ASSERTION"></a>

### `ERR_ASSERTION`

一种特殊类型的错误，每当 Node.js 检测到不应发生的异常逻辑违规时可以触发。这些通常由 `node:assert` 模块引发。

<a id="ERR_ASYNC_CALLBACK"></a>

### `ERR_ASYNC_CALLBACK```

尝试将不是函数的内容注册为 `AsyncHooks` 回调。

<a id="ERR_ASYNC_TYPE"></a>

### `ERR_ASYNC_TYPE`

异步资源的类型无效。如果使用公共嵌入器 API，用户也能够定义自己的类型。

<a id="ERR_BROTLI_COMPRESSION_FAILED"></a>

### `ERR_BROTLI_COMPRESSION_FAILED`

传递给 Brotli 流的数据未能成功压缩。

<a id="ERR_BROTLI_INVALID_PARAM"></a>

### `ERR_BROTLI_INVALID_PARAM`

在构建 Brotli 流期间传递了无效的参数键。

<a id="ERR_BUFFER_CONTEXT_NOT_AVAILABLE"></a>

### `ERR_BUFFER_CONTEXT_NOT_AVAILABLE```

尝试从插件或嵌入器代码创建 Node.js `Buffer` 实例，但所在的 JS 引擎上下文与 Node.js 实例无关。传递给 `Buffer` 方法的数据将在方法返回时被释放。

遇到此错误时，创建 `Buffer` 实例的一个可能替代方法是创建普通的 `Uint8Array`，它仅在结果对象的原型上有所不同。`Uint8Array` 通常在所有接受 `Buffer` 的 Node.js 核心 API 中被接受；它们在所有上下文中都可用。

<a id="ERR_BUFFER_OUT_OF_BOUNDS"></a>

### `ERR_BUFFER_OUT_OF_BOUNDS```

尝试在 `Buffer` 的边界之外进行操作。

<a id="ERR_BUFFER_TOO_LARGE"></a>

### `ERR_BUFFER_TOO_LARGE```

尝试创建超过最大允许大小的 `Buffer`。

<a id="ERR_CANNOT_WATCH_SIGINT"></a>

### `ERR_CANNOT_WATCH_SIGINT```

Node.js 无法监视 `SIGINT` 信号。

<a id="ERR_CHILD_CLOSED_BEFORE_REPLY"></a>

### `ERR_CHILD_CLOSED_BEFORE_REPLY```

子进程在父进程收到回复之前关闭。

<a id="ERR_CHILD_PROCESS_IPC_REQUIRED"></a>

### `ERR_CHILD_PROCESS_IPC_REQUIRED```

当派生子进程时未指定 IPC 通道时使用。

<a id="ERR_CHILD_PROCESS_STDIO_MAXBUFFER"></a>

### `ERR_CHILD_PROCESS_STDIO_MAXBUFFER```

当主进程尝试从子进程的 STDERR/STDOUT 读取数据，且数据长度超过 `maxBuffer` 选项时使用。

<a id="ERR_CLOSED_MESSAGE_PORT"></a>

### `ERR_CLOSED_MESSAGE_PORT`

<!-- YAML
added: v10.5.0
changes:
  - version:
      - v16.2.0
      - v14.17.1
    pr-url: https://github.com/nodejs/node/pull/38510
    description: The error message was reintroduced.
  - version: v11.12.0
    pr-url: https://github.com/nodejs/node/pull/26487
    description: The error message was removed.
-->

尝试在关闭状态下使用 `MessagePort` 实例，通常在调用 `.close()` 之后。

<a id="ERR_CONSOLE_WRITABLE_STREAM"></a>

### `ERR_CONSOLE_WRITABLE_STREAM```

`Console` 实例化时没有 `stdout` 流，或者 `Console` 的 `stdout` 或 `stderr` 流不可写。

<a id="ERR_CONSTRUCT_CALL_INVALID"></a>

### `ERR_CONSTRUCT_CALL_INVALID`

<!-- YAML
added: v12.5.0
-->

调用了不可调用的类构造函数。

<a id="ERR_CONSTRUCT_CALL_REQUIRED"></a>

### `ERR_CONSTRUCT_CALL_REQUIRED```

类的构造函数被调用时没有使用 `new`。

<a id="ERR_CONTEXT_NOT_INITIALIZED"></a>

### `ERR_CONTEXT_NOT_INITIALIZED```

传递给 API 的 vm 上下文尚未初始化。这可能在上下文创建期间发生错误（并被捕获）时发生，例如，在分配失败或创建上下文时达到最大调用栈大小。

<a id="ERR_CPU_PROFILE_ALREADY_STARTED"></a>

### `ERR_CPU_PROFILE_ALREADY_STARTED`

<!-- YAML
added: v24.8.0
-->

具有给定名称的 CPU 分析已启动。

<a id="ERR_CPU_PROFILE_NOT_STARTED"></a>

### `ERR_CPU_PROFILE_NOT_STARTED`

<!-- YAML
added: v24.8.0
-->

具有给定名称的 CPU 分析未启动。

<a id="ERR_CPU_PROFILE_TOO_MANY"></a>

### `ERR_CPU_PROFILE_TOO_MANY`

<!-- YAML
added: v24.8.0
-->

正在收集的 CPU 分析过多。

<a id="ERR_CRYPTO_ARGON2_NOT_SUPPORTED"></a>

### `ERR_CRYPTO_ARGON2_NOT_SUPPORTED```

当前使用的 OpenSSL 版本不支持 Argon2。

<a id="ERR_CRYPTO_CUSTOM_ENGINE_NOT_SUPPORTED"></a>

### `ERR_CRYPTO_CUSTOM_ENGINE_NOT_SUPPORTED```

请求了当前使用的 OpenSSL 版本不支持的 OpenSSL 引擎（例如，通过 `clientCertEngine` 或 `privateKeyEngine` TLS 选项），可能是由于编译时标志 `OPENSSL_NO_ENGINE`。

<a id="ERR_CRYPTO_ECDH_INVALID_FORMAT"></a>

### `ERR_CRYPTO_ECDH_INVALID_FORMAT```

向 `crypto.ECDH()` 类的 `getPublicKey()` 方法传递了无效的 `format` 参数值。

<a id="ERR_CRYPTO_ECDH_INVALID_PUBLIC_KEY"></a>

### `ERR_CRYPTO_ECDH_INVALID_PUBLIC_KEY```

向 `crypto.ECDH()` 类的 `computeSecret()` 方法传递了无效的 `key` 参数值。这意味着公钥位于椭圆曲线之外。

<a id="ERR_CRYPTO_ENGINE_UNKNOWN"></a>

### `ERR_CRYPTO_ENGINE_UNKNOWN```

向 [`require('node:crypto').setEngine()`][] 传递了无效的加密引擎标识符。

<a id="ERR_CRYPTO_FIPS_FORCED"></a>

### `ERR_CRYPTO_FIPS_FORCED```

使用了 [`--force-fips`][] 命令行参数，但尝试在 `node:crypto` 模块中启用或禁用 FIPS 模式。

<a id="ERR_CRYPTO_FIPS_UNAVAILABLE"></a>

### `ERR_CRYPTO_FIPS_UNAVAILABLE```

尝试启用或禁用 FIPS 模式，但 FIPS 模式不可用。

<a id="ERR_CRYPTO_HASH_FINALIZED"></a>

### `ERR_CRYPTO_HASH_FINALIZED```

[`hash.digest()`][] 被多次调用。`hash.digest()` 方法在每个 `Hash` 对象实例中最多只能调用一次。

<a id="ERR_CRYPTO_HASH_UPDATE_FAILED"></a>

### `ERR_CRYPTO_HASH_UPDATE_FAILED```

[`hash.update()`][] 因任何原因失败。这应该很少发生。

<a id="ERR_CRYPTO_INCOMPATIBLE_KEY"></a>

### `ERR_CRYPTO_INCOMPATIBLE_KEY```

给定的加密密钥与尝试的操作不兼容。

<a id="ERR_CRYPTO_INCOMPATIBLE_KEY_OPTIONS"></a>

### `ERR_CRYPTO_INCOMPATIBLE_KEY_OPTIONS```

选择的公钥或私钥编码与其他选项不兼容。

<a id="ERR_CRYPTO_INITIALIZATION_FAILED"></a>

### `ERR_CRYPTO_INITIALIZATION_FAILED`

<!-- YAML
added: v15.0.0
-->

加密子系统初始化失败。

<a id="ERR_CRYPTO_INVALID_AUTH_TAG"></a>

### `ERR_CRYPTO_INVALID_AUTH_TAG`

<!-- YAML
added: v15.0.0
-->

提供了无效的身份验证标签。

<a id="ERR_CRYPTO_INVALID_COUNTER"></a>

### `ERR_CRYPTO_INVALID_COUNTER`

<!-- YAML
added: v15.0.0
-->

为计数器模式密码提供了无效的计数器。

<a id="ERR_CRYPTO_INVALID_CURVE"></a>

### `ERR_CRYPTO_INVALID_CURVE`

<!-- YAML
added: v15.0.0
-->

提供了无效的椭圆曲线。

<a id="ERR_CRYPTO_INVALID_DIGEST"></a>

### `ERR_CRYPTO_INVALID_DIGEST```

指定了无效的[加密摘要算法][crypto digest algorithm]。

<a id="ERR_CRYPTO_INVALID_IV"></a>

### `ERR_CRYPTO_INVALID_IV`

<!-- YAML
added: v15.0.0
-->

提供了无效的初始化向量。

<a id="ERR_CRYPTO_INVALID_JWK"></a>

### `ERR_CRYPTO_INVALID_JWK`

<!-- YAML
added: v15.0.0
-->

提供了无效的 JSON Web Key。

<a id="ERR_CRYPTO_INVALID_KEYLEN"></a>

### `ERR_CRYPTO_INVALID_KEYLEN`

<!-- YAML
added: v15.0.0
-->

提供了无效的密钥长度。

<a id="ERR_CRYPTO_INVALID_KEYPAIR"></a>

### `ERR_CRYPTO_INVALID_KEYPAIR`

<!-- YAML
added: v15.0.0
-->

提供了无效的密钥对。

<a id="ERR_CRYPTO_INVALID_KEYTYPE"></a>

### `ERR_CRYPTO_INVALID_KEYTYPE`

<!-- YAML
added: v15.0.0
-->

提供了无效的密钥类型。

<a id="ERR_CRYPTO_INVALID_KEY_OBJECT_TYPE"></a>

### `ERR_CRYPTO_INVALID_KEY_OBJECT_TYPE```

给定的加密密钥对象的类型对于尝试的操作无效。

<a id="ERR_CRYPTO_INVALID_MESSAGELEN"></a>

### `ERR_CRYPTO_INVALID_MESSAGELEN`

<!-- YAML
added: v15.0.0
-->

提供了无效的消息长度。

<a id="ERR_CRYPTO_INVALID_SCRYPT_PARAMS"></a>

### `ERR_CRYPTO_INVALID_SCRYPT_PARAMS`

<!-- YAML
added: v15.0.0
-->

一个或多个 [`crypto.scrypt()`][] 或 [`crypto.scryptSync()`][] 参数超出其合法范围。

<a id="ERR_CRYPTO_INVALID_STATE"></a>

### `ERR_CRYPTO_INVALID_STATE```

在处于无效状态的对象上使用了加密方法。例如，在调用 `cipher.final()` 之前调用 [`cipher.getAuthTag()`][]。

<a id="ERR_CRYPTO_INVALID_TAG_LENGTH"></a>

### `ERR_CRYPTO_INVALID_TAG_LENGTH`

<!-- YAML
added: v15.0.0
-->

提供了无效的身份验证标签长度。

<a id="ERR_CRYPTO_JOB_INIT_FAILED"></a>

### `ERR_CRYPTO_JOB_INIT_FAILED`

<!-- YAML
added: v15.0.0
-->

异步加密操作初始化失败。

<a id="ERR_CRYPTO_JWK_UNSUPPORTED_CURVE"></a>

### `ERR_CRYPTO_JWK_UNSUPPORTED_CURVE```

密钥的椭圆曲线未在 [JSON Web Key Elliptic Curve Registry][] 中注册使用。

<a id="ERR_CRYPTO_JWK_UNSUPPORTED_KEY_TYPE"></a>

### `ERR_CRYPTO_JWK_UNSUPPORTED_KEY_TYPE```

密钥的非对称密钥类型未在 [JSON Web Key Types Registry][] 中注册使用。

<a id="ERR_CRYPTO_KEM_NOT_SUPPORTED"></a>

### `ERR_CRYPTO_KEM_NOT_SUPPORTED`

<!-- YAML
added: v24.7.0
-->

尝试使用 KEM 操作，但 Node.js 编译时未使用支持 KEM 的 OpenSSL。

<a id="ERR_CRYPTO_OPERATION_FAILED"></a>

### `ERR_CRYPTO_OPERATION_FAILED`

<!-- YAML
added: v15.0.0
-->

加密操作因其他未指定的原因失败。

<a id="ERR_CRYPTO_PBKDF2_ERROR"></a>

### `ERR_CRYPTO_PBKDF2_ERROR```

PBKDF2 算法因未指定的原因失败。OpenSSL 不提供更多细节，因此 Node.js 也不提供。

<a id="ERR_CRYPTO_SCRYPT_NOT_SUPPORTED"></a>

### `ERR_CRYPTO_SCRYPT_NOT_SUPPORTED```

Node.js 编译时未包含 `scrypt` 支持。官方发布版本不可能发生，但可能发生在自定义构建中，包括发行版构建。

<a id="ERR_CRYPTO_SIGN_KEY_REQUIRED"></a>

### `ERR_CRYPTO_SIGN_KEY_REQUIRED```

未向 [`sign.sign()`][] 方法提供签名 `key`。

<a id="ERR_CRYPTO_TIMING_SAFE_EQUAL_LENGTH"></a>

### `ERR_CRYPTO_TIMING_SAFE_EQUAL_LENGTH```

使用不同长度的 `Buffer`、`TypedArray` 或 `DataView` 参数调用了 [`crypto.timingSafeEqual()`][]。

<a id="ERR_CRYPTO_UNKNOWN_CIPHER"></a>

### `ERR_CRYPTO_UNKNOWN_CIPHER```

指定了未知的密码。

<a id="ERR_CRYPTO_UNKNOWN_DH_GROUP"></a>

### `ERR_CRYPTO_UNKNOWN_DH_GROUP```

给出了未知的 Diffie-Hellman 组名。有关有效组名列表，请参见 [`crypto.getDiffieHellman()`][]。

<a id="ERR_CRYPTO_UNSUPPORTED_OPERATION"></a>

### `ERR_CRYPTO_UNSUPPORTED_OPERATION`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

尝试调用不受支持的加密操作。

<a id="ERR_DEBUGGER_ERROR"></a>

### `ERR_DEBUGGER_ERROR`

<!-- YAML
added:
  - v16.4.0
  - v14.17.4
-->

[调试器][debugger]发生错误。

<a id="ERR_DEBUGGER_STARTUP_ERROR"></a>

### `ERR_DEBUGGER_STARTUP_ERROR`

<!-- YAML
added:
  - v16.4.0
  - v14.17.4
-->

[调试器][debugger]在等待所需的主机/端口空闲时超时。

<a id="ERR_DIR_CLOSED"></a>

### `ERR_DIR_CLOSED```

[`fs.Dir`][] 先前已关闭。

<a id="ERR_DIR_CONCURRENT_OPERATION"></a>

### `ERR_DIR_CONCURRENT_OPERATION`

<!-- YAML
added: v14.3.0
-->

尝试在具有正在进行异步操作的 [`fs.Dir`][] 上进行同步读取或关闭调用。

<a id="ERR_DLOPEN_DISABLED"></a>

### `ERR_DLOPEN_DISABLED`

<!-- YAML
added:
  - v16.10.0
  - v14.19.0
-->

已使用 [`--no-addons`][] 禁用加载原生插件。

<a id="ERR_DLOPEN_FAILED"></a>

### `ERR_DLOPEN_FAILED`

<!-- YAML
added: v15.0.0
-->

调用 `process.dlopen()` 失败。

<a id="ERR_DNS_SET_SERVERS_FAILED"></a>

### `ERR_DNS_SET_SERVERS_FAILED```

`c-ares` 未能设置 DNS 服务器。

<a id="ERR_DOMAIN_CALLBACK_NOT_AVAILABLE"></a>

### `ERR_DOMAIN_CALLBACK_NOT_AVAILABLE```

`node:domain` 模块不可用，因为它无法建立所需的错误处理钩子，因为 [`process.setUncaughtExceptionCaptureCallback()`][] 在更早的时间点已被调用。

<a id="ERR_DOMAIN_CANNOT_SET_UNCAUGHT_EXCEPTION_CAPTURE"></a>

### `ERR_DOMAIN_CANNOT_SET_UNCAUGHT_EXCEPTION_CAPTURE```

无法调用 [`process.setUncaughtExceptionCaptureCallback()`][]，因为 `node:domain` 模块在更早的时间点已被加载。

栈追踪被扩展以包含加载 `node:domain` 模块的时间点。

<a id="ERR_DUPLICATE_STARTUP_SNAPSHOT_MAIN_FUNCTION"></a>

### `ERR_DUPLICATE_STARTUP_SNAPSHOT_MAIN_FUNCTION```

无法调用 [`v8.startupSnapshot.setDeserializeMainFunction()`][]，因为它之前已被调用过。

<a id="ERR_ENCODING_INVALID_ENCODED_DATA"></a>

### `ERR_ENCODING_INVALID_ENCODED_DATA```

提供给 `TextDecoder()` API 的数据根据提供的编码无效。

<a id="ERR_ENCODING_NOT_SUPPORTED"></a>

### `ERR_ENCODING_NOT_SUPPORTED```

提供给 `TextDecoder()` API 的编码不是 [WHATWG 支持的编码][WHATWG Supported Encodings] 之一。

<a id="ERR_EVAL_ESM_CANNOT_PRINT"></a>

### `ERR_EVAL_ESM_CANNOT_PRINT```

`--print` 不能与 ESM 输入一起使用。

<a id="ERR_EVENT_RECURSION"></a>

### `ERR_EVENT_RECURSION```

尝试在 `EventTarget` 上递归分发事件时抛出。

<a id="ERR_EXECUTION_ENVIRONMENT_NOT_AVAILABLE"></a>

### `ERR_EXECUTION_ENVIRONMENT_NOT_AVAILABLE```

JS 执行上下文与 Node.js 环境无关。这可能发生在 Node.js 被用作嵌入式库且未正确设置 JS 引擎的一些钩子时。

<a id="ERR_FALSY_VALUE_REJECTION"></a>

### `ERR_FALSY_VALUE_REJECTION```

通过 `util.callbackify()` 回调化的 `Promise` 被拒绝，且拒绝值为假值。

<a id="ERR_FEATURE_UNAVAILABLE_ON_PLATFORM"></a>

### `ERR_FEATURE_UNAVAILABLE_ON_PLATFORM`

<!-- YAML
added: v14.0.0
-->

当使用对运行 Node.js 的当前平台不可用的功能时使用。

<a id="ERR_FS_CP_DIR_TO_NON_DIR"></a>

### `ERR_FS_CP_DIR_TO_NON_DIR`

<!-- YAML
added: v16.7.0
-->

尝试使用 [`fs.cp()`][] 将目录复制到非目录（文件、符号链接等）。

<a id="ERR_FS_CP_EEXIST"></a>

### `ERR_FS_CP_EEXIST`

<!-- YAML
added: v16.7.0
-->

尝试使用 [`fs.cp()`][] 覆盖已存在的文件，且 `force` 和 `errorOnExist` 设置为 `true`。

<a id="ERR_FS_CP_EINVAL"></a>

### `ERR_FS_CP_EINVAL`

<!-- YAML
added: v16.7.0
-->

使用 [`fs.cp()`][] 时，`src` 或 `dest` 指向无效路径。

<a id="ERR_FS_CP_FIFO_PIPE"></a>

### `ERR_FS_CP_FIFO_PIPE`

<!-- YAML
added: v16.7.0
-->

尝试使用 [`fs.cp()`][] 复制命名管道。

<a id="ERR_FS_CP_NON_DIR_TO_DIR"></a>

### `ERR_FS_CP_NON_DIR_TO_DIR`

<!-- YAML
added: v16.7.0
-->

尝试使用 [`fs.cp()`][] 将非目录（文件、符号链接等）复制到目录。

<a id="ERR_FS_CP_SOCKET"></a>

### `ERR_FS_CP_SOCKET`

<!-- YAML
added: v16.7.0
-->

尝试使用 [`fs.cp()`][] 复制到套接字。

<a id="ERR_FS_CP_SYMLINK_TO_SUBDIRECTORY"></a>

### `ERR_FS_CP_SYMLINK_TO_SUBDIRECTORY`

<!-- YAML
added: v16.7.0
-->

使用 [`fs.cp()`][] 时，`dest` 中的符号链接指向 `src` 的子目录。

<a id="ERR_FS_CP_UNKNOWN"></a>

### `ERR_FS_CP_UNKNOWN`

<!-- YAML
added: v16.7.0
-->

尝试使用 [`fs.cp()`][] 复制到未知文件类型。

<a id="ERR_FS_EISDIR"></a>

### `ERR_FS_EISDIR```

路径是目录。

<a id="ERR_FS_FILE_TOO_LARGE"></a>

### `ERR_FS_FILE_TOO_LARGE```

尝试读取大于 `fs.readFile()` 支持的 2 GiB 限制的文件。这不是 `Buffer` 的限制，而是内部 I/O 约束。要处理更大的文件，请考虑使用 `fs.createReadStream()` 分块读取文件。

<a id="ERR_FS_WATCH_QUEUE_OVERFLOW"></a>

### `ERR_FS_WATCH_QUEUE_OVERFLOW```

未处理的文件系统事件队列数量超过了 `fs.watch()` 中 `maxQueue` 指定的大小。

<a id="ERR_HTTP2_ALTSVC_INVALID_ORIGIN"></a>

### `ERR_HTTP2_ALTSVC_INVALID_ORIGIN```

HTTP/2 ALTSVC 帧需要有效的来源。

<a id="ERR_HTTP2_ALTSVC_LENGTH"></a>

### `ERR_HTTP2_ALTSVC_LENGTH```

HTTP/2 ALTSVC 帧限制为最多 16,382 个有效载荷字节。

<a id="ERR_HTTP2_CONNECT_AUTHORITY"></a>

### `ERR_HTTP2_CONNECT_AUTHORITY```

对于使用 `CONNECT` 方法的 HTTP/2 请求，需要 `:authority` 伪头部。

<a id="ERR_HTTP2_CONNECT_PATH"></a>

### `ERR_HTTP2_CONNECT_PATH```

对于使用 `CONNECT` 方法的 HTTP/2 请求，禁止使用 `:path` 伪头部。

<a id="ERR_HTTP2_CONNECT_SCHEME"></a>

### `ERR_HTTP2_CONNECT_SCHEME```

对于使用 `CONNECT` 方法的 HTTP/2 请求，禁止使用 `:scheme` 伪头部。

<a id="ERR_HTTP2_ERROR"></a>

### `ERR_HTTP2_ERROR```

发生了非特定的 HTTP/2 错误。

<a id="ERR_HTTP2_GOAWAY_SESSION"></a>

### `ERR_HTTP2_GOAWAY_SESSION```

在 `Http2Session` 从连接的对端接收到 `GOAWAY` 帧后，可能无法打开新的 HTTP/2 流。

<a id="ERR_HTTP2_HEADERS_AFTER_RESPOND"></a>

### `ERR_HTTP2_HEADERS_AFTER_RESPOND```

在 HTTP/2 响应启动后指定了额外的头部。

<a id="ERR_HTTP2_HEADERS_SENT"></a>

### `ERR_HTTP2_HEADERS_SENT```

尝试发送多个响应头部。

<a id="ERR_HTTP2_HEADER_SINGLE_VALUE"></a>

### `ERR_HTTP2_HEADER_SINGLE_VALUE```

为需要只有一个值的 HTTP/2 头部字段提供了多个值。

<a id="ERR_HTTP2_INFO_STATUS_NOT_ALLOWED"></a>

### `ERR_HTTP2_INFO_STATUS_NOT_ALLOWED```

信息性 HTTP 状态码（`1xx`）不能设置为 HTTP/2 响应的响应状态码。

<a id="ERR_HTTP2_INVALID_CONNECTION_HEADERS"></a>

### `ERR_HTTP2_INVALID_CONNECTION_HEADERS```

HTTP/1 连接特定的头部禁止在 HTTP/2 请求和响应中使用。

<a id="ERR_HTTP2_INVALID_HEADER_VALUE"></a>

### `ERR_HTTP2_INVALID_HEADER_VALUE```

指定了无效的 HTTP/2 头部值。

<a id="ERR_HTTP2_INVALID_INFO_STATUS"></a>

### `ERR_HTTP2_INVALID_INFO_STATUS```

指定了无效的 HTTP 信息状态码。信息状态码必须是 `100` 到 `199`（包含）之间的整数。

<a id="ERR_HTTP2_INVALID_ORIGIN"></a>

### `ERR_HTTP2_INVALID_ORIGIN```

HTTP/2 `ORIGIN` 帧需要有效的来源。

<a id="ERR_HTTP2_INVALID_PACKED_SETTINGS_LENGTH"></a>

### `ERR_HTTP2_INVALID_PACKED_SETTINGS_LENGTH```

传递给 `http2.getUnpackedSettings()` API 的输入 `Buffer` 和 `Uint8Array` 实例的长度必须是六的倍数。

<a id="ERR_HTTP2_INVALID_PSEUDOHEADER"></a>

### `ERR_HTTP2_INVALID_PSEUDOHEADER```

只能使用有效的 HTTP/2 伪头部（`:status`、`:path`、`:authority`、`:scheme` 和 `:method`）。

<a id="ERR_HTTP2_INVALID_SESSION"></a>

### `ERR_HTTP2_INVALID_SESSION```

在已经销毁的 `Http2Session` 对象上执行了操作。

<a id="ERR_HTTP2_INVALID_SETTING_VALUE"></a>

### `ERR_HTTP2_INVALID_SETTING_VALUE```

为 HTTP/2 设置指定了无效的值。

<a id="ERR_HTTP2_INVALID_STREAM"></a>

### `ERR_HTTP2_INVALID_STREAM```

在已经销毁的流上执行了操作。

<a id="ERR_HTTP2_MAX_PENDING_SETTINGS_ACK"></a>

### `ERR_HTTP2_MAX_PENDING_SETTINGS_ACK```

每当 HTTP/2 `SETTINGS` 帧发送到连接的对端时，对端需要发送确认，表示已接收并应用新的 `SETTINGS`。默认情况下，在任何给定时间可以发送的最大未确认 `SETTINGS` 帧数量是有限的。当达到该限制时，使用此错误代码。

<a id="ERR_HTTP2_NESTED_PUSH"></a>

### `ERR_HTTP2_NESTED_PUSH```

尝试从推送流内部启动新的推送流。不允许嵌套推送流。

<a id="ERR_HTTP2_NO_MEM"></a>

### `ERR_HTTP2_NO_MEM```

使用 `http2session.setLocalWindowSize(windowSize)` API 时内存不足。

<a id="ERR_HTTP2_NO_SOCKET_MANIPULATION"></a>

### `ERR_HTTP2_NO_SOCKET_MANIPULATION```

尝试直接操作（读取、写入、暂停、恢复等）附加到 `Http2Session` 的套接字。

<a id="ERR_HTTP2_ORIGIN_LENGTH"></a>

### `ERR_HTTP2_ORIGIN_LENGTH```

HTTP/2 `ORIGIN` 帧限制为 16382 字节的长度。

<a id="ERR_HTTP2_OUT_OF_STREAMS"></a>

### `ERR_HTTP2_OUT_OF_STREAMS```

在单个 HTTP/2 会话上创建的流数量达到最大限制。

<a id="ERR_HTTP2_PAYLOAD_FORBIDDEN"></a>

### `ERR_HTTP2_PAYLOAD_FORBIDDEN```

为禁止有效载荷的 HTTP 响应代码指定了消息有效载荷。

<a id="ERR_HTTP2_PING_CANCEL"></a>

### `ERR_HTTP2_PING_CANCEL```

HTTP/2 ping 被取消。

<a id="ERR_HTTP2_PING_LENGTH"></a>

### `ERR_HTTP2_PING_LENGTH```

HTTP/2 ping 有效载荷必须正好是 8 字节长。

<a id="ERR_HTTP2_PSEUDOHEADER_NOT_ALLOWED"></a>

### `ERR_HTTP2_PSEUDOHEADER_NOT_ALLOWED```

HTTP/2 伪头部使用不当。伪头部是以 `:` 前缀开头的头部键名。

<a id="ERR_HTTP2_PUSH_DISABLED"></a>

### `ERR_HTTP2_PUSH_DISABLED```

尝试创建推送流，但已被客户端禁用。

<a id="ERR_HTTP2_SEND_FILE"></a>

### `ERR_HTTP2_SEND_FILE```

尝试使用 `Http2Stream.prototype.responseWithFile()` API 发送目录。

<a id="ERR_HTTP2_SEND_FILE_NOSEEK"></a>

### `ERR_HTTP2_SEND_FILE_NOSEEK```

尝试使用 `Http2Stream.prototype.responseWithFile()` API 发送非普通文件的内容，但提供了 `offset` 或 `length` 选项。

<a id="ERR_HTTP2_SESSION_ERROR"></a>

### `ERR_HTTP2_SESSION_ERROR```

`Http2Session` 以非零错误代码关闭。

<a id="ERR_HTTP2_SETTINGS_CANCEL"></a>

### `ERR_HTTP2_SETTINGS_CANCEL```

`Http2Session` 设置被取消。

<a id="ERR_HTTP2_SOCKET_BOUND"></a>

### `ERR_HTTP2_SOCKET_BOUND```

尝试将 `Http2Session` 对象连接到已绑定到另一个 `Http2Session` 对象的 `net.Socket` 或 `tls.TLSSocket`。

<a id="ERR_HTTP2_SOCKET_UNBOUND"></a>

### `ERR_HTTP2_SOCKET_UNBOUND```

尝试使用已经关闭的 `Http2Session` 的 `socket` 属性。

<a id="ERR_HTTP2_STATUS_101"></a>

### `ERR_HTTP2_STATUS_101```

在 HTTP/2 中禁止使用 `101` 信息状态码。

<a id="ERR_HTTP2_STATUS_INVALID"></a>

### `ERR_HTTP2_STATUS_INVALID```

指定了无效的 HTTP 状态码。状态码必须是 `100` 到 `599`（包含）之间的整数。

<a id="ERR_HTTP2_STREAM_CANCEL"></a>

### `ERR_HTTP2_STREAM_CANCEL```

在有任何数据传输到连接的对端之前，`Http2Stream` 被销毁。

<a id="ERR_HTTP2_STREAM_ERROR"></a>

### `ERR_HTTP2_STREAM_ERROR```

在 `RST_STREAM` 帧中指定了非零错误代码。

<a id="ERR_HTTP2_STREAM_SELF_DEPENDENCY"></a>

### `ERR_HTTP2_STREAM_SELF_DEPENDENCY```

当设置 HTTP/2 流的优先级时，流可能被标记为父流的依赖项。当尝试将流标记为其自身的依赖项时，使用此错误代码。

<a id="ERR_HTTP2_TOO_MANY_CUSTOM_SETTINGS"></a>

### `ERR_HTTP2_TOO_MANY_CUSTOM_SETTINGS```

支持的自定义设置数量（10）已超出。

<a id="ERR_HTTP2_TOO_MANY_INVALID_FRAMES"></a>

### `ERR_HTTP2_TOO_MANY_INVALID_FRAMES`

<!-- YAML
added: v15.14.0
-->

对端发送的不可接受无效 HTTP/2 协议帧的数量已超过通过 `maxSessionInvalidFrames` 选项指定的限制。

<a id="ERR_HTTP2_TRAILERS_ALREADY_SENT"></a>

### `ERR_HTTP2_TRAILERS_ALREADY_SENT```

尾部头部已在 `Http2Stream` 上发送。

<a id="ERR_HTTP2_TRAILERS_NOT_READY"></a>

### `ERR_HTTP2_TRAILERS_NOT_READY```

在 `Http2Stream` 对象上发出 `'wantTrailers'` 事件之前，不能调用 `http2stream.sendTrailers()` 方法。仅当为 `Http2Stream` 设置了 `waitForTrailers` 选项时，才会发出 `'wantTrailers'` 事件。

<a id="ERR_HTTP2_UNSUPPORTED_PROTOCOL"></a>

### `ERR_HTTP2_UNSUPPORTED_PROTOCOL```

`http2.connect()` 传递了使用 `http:` 或 `https:` 以外协议的 URL。

<a id="ERR_HTTP_BODY_NOT_ALLOWED"></a>

### `ERR_HTTP_BODY_NOT_ALLOWED```

当写入不允许内容的 HTTP 响应时抛出错误。

<a id="ERR_HTTP_CONTENT_LENGTH_MISMATCH"></a>

### `ERR_HTTP_CONTENT_LENGTH_MISMATCH```

响应体大小与指定的 content-length 头部值不匹配。

<a id="ERR_HTTP_HEADERS_SENT"></a>

### `ERR_HTTP_HEADERS_SENT```

在头部已经发送后尝试添加更多头部。

<a id="ERR_HTTP_INVALID_HEADER_VALUE"></a>

### `ERR_HTTP_INVALID_HEADER_VALUE```

指定了无效的 HTTP 头部值。

<a id="ERR_HTTP_INVALID_STATUS_CODE"></a>

### `ERR_HTTP_INVALID_STATUS_CODE```

状态码超出常规状态码范围（100-999）。

<a id="ERR_HTTP_REQUEST_TIMEOUT"></a>

### `ERR_HTTP_REQUEST_TIMEOUT```

客户端未在允许的时间内发送完整请求。

<a id="ERR_HTTP_SOCKET_ASSIGNED"></a>

### `ERR_HTTP_SOCKET_ASSIGNED```

给定的 [`ServerResponse`][] 已被分配套接字。

<a id="ERR_HTTP_SOCKET_ENCODING"></a>

### `ERR_HTTP_SOCKET_ENCODING```

根据 [RFC 7230 Section 3][]，不允许更改套接字编码。

<a id="ERR_HTTP_TRAILER_INVALID"></a>

### `ERR_HTTP_TRAILER_INVALID```

即使传输编码不支持，也设置了 `Trailer` 头部。

<a id="ERR_ILLEGAL_CONSTRUCTOR"></a>

### `ERR_ILLEGAL_CONSTRUCTOR```

尝试使用非公共构造函数构造对象。

<a id="ERR_IMPORT_ATTRIBUTE_MISSING"></a>

### `ERR_IMPORT_ATTRIBUTE_MISSING`

<!-- YAML
added:
  - v21.1.0
-->

缺少导入属性，导致无法导入指定模块。

<a id="ERR_IMPORT_ATTRIBUTE_TYPE_INCOMPATIBLE"></a>

### `ERR_IMPORT_ATTRIBUTE_TYPE_INCOMPATIBLE`

<!-- YAML
added:
  - v21.1.0
-->

提供了导入 `type` 属性，但指定模块的类型不同。

<a id="ERR_IMPORT_ATTRIBUTE_UNSUPPORTED"></a>

### `ERR_IMPORT_ATTRIBUTE_UNSUPPORTED`

<!-- YAML
added:
  - v21.0.0
  - v20.10.0
  - v18.19.0
-->

此版本的 Node.js 不支持导入属性。

<a id="ERR_INCOMPATIBLE_OPTION_PAIR"></a>

### `ERR_INCOMPATIBLE_OPTION_PAIR```

选项对彼此不兼容，不能同时使用。

<a id="ERR_INPUT_TYPE_NOT_ALLOWED"></a>

### `ERR_INPUT_TYPE_NOT_ALLOWED```

`--input-type` 标志用于尝试执行文件。此标志只能与通过 `--eval`、`--print` 或 `STDIN` 的输入一起使用。

<a id="ERR_INSPECTOR_ALREADY_ACTIVATED"></a>

### `ERR_INSPECTOR_ALREADY_ACTIVATED```

在使用 `node:inspector` 模块时，尝试在检查器已经开始监听端口时激活它。在另一个地址激活之前，使用 `inspector.close()`。

<a id="ERR_INSPECTOR_ALREADY_CONNECTED"></a>

### `ERR_INSPECTOR_ALREADY_CONNECTED```

在使用 `node:inspector` 模块时，尝试在检查器已经连接时连接。

<a id="ERR_INSPECTOR_CLOSED"></a>

### `ERR_INSPECTOR_CLOSED```

在使用 `node:inspector` 模块时，尝试在会话已经关闭后使用检查器。

<a id="ERR_INSPECTOR_COMMAND"></a>

### `ERR_INSPECTOR_COMMAND```

通过 `node:inspector` 模块发出命令时发生错误。

<a id="ERR_INSPECTOR_NOT_ACTIVE"></a>

### `ERR_INSPECTOR_NOT_ACTIVE```

调用 `inspector.waitForDebugger()` 时检查器未激活。

<a id="ERR_INSPECTOR_NOT_AVAILABLE"></a>

### `ERR_INSPECTOR_NOT_AVAILABLE```

`node:inspector` 模块不可用。

<a id="ERR_INSPECTOR_NOT_CONNECTED"></a>

### `ERR_INSPECTOR_NOT_CONNECTED```

在使用 `node:inspector` 模块时，尝试在检查器连接之前使用它。

<a id="ERR_INSPECTOR_NOT_WORKER"></a>

### `ERR_INSPECTOR_NOT_WORKER```

在主线程上调用了只能从工作线程使用的 API。

<a id="ERR_INTERNAL_ASSERTION"></a>

### `ERR_INTERNAL_ASSERTION```

Node.js 内部存在错误或错误使用。要修复错误，请在 <https://github.com/nodejs/node/issues> 开一个问题。

<a id="ERR_INVALID_ADDRESS"></a>

### `ERR_INVALID_ADDRESS```

提供的地址不被 Node.js API 理解。

<a id="ERR_INVALID_ADDRESS_FAMILY"></a>

### `ERR_INVALID_ADDRESS_FAMILY```

提供的地址族不被 Node.js API 理解。

<a id="ERR_INVALID_ARG_TYPE"></a>

### `ERR_INVALID_ARG_TYPE```

向 Node.js API 传递了错误类型的参数。

<a id="ERR_INVALID_ARG_VALUE"></a>

### `ERR_INVALID_ARG_VALUE```

为给定参数传递了无效或不支持的值。

<a id="ERR_INVALID_ASYNC_ID"></a>

### `ERR_INVALID_ASYNC_ID```

使用 `AsyncHooks` 传递了无效的 `asyncId` 或 `triggerAsyncId`。id 小于 -1 的情况不应发生。

<a id="ERR_INVALID_BUFFER_SIZE"></a>

### `ERR_INVALID_BUFFER_SIZE```

在 `Buffer` 上执行了交换，但其大小与操作不兼容。

<a id="ERR_INVALID_CHAR"></a>

### `ERR_INVALID_CHAR```

在头部中检测到无效字符。

<a id="ERR_INVALID_CURSOR_POS"></a>

### `ERR_INVALID_CURSOR_POS```

无法在未指定列的情况下将给定流上的光标移动到指定行。

<a id="ERR_INVALID_FD"></a>

### `ERR_INVALID_FD```

文件描述符（'fd'）无效（例如，它是负值）。

<a id="ERR_INVALID_FD_TYPE"></a>

### `ERR_INVALID_FD_TYPE```

文件描述符（'fd'）类型无效。

<a id="ERR_INVALID_FILE_URL_HOST"></a>

### `ERR_INVALID_FILE_URL_HOST```

使用 `file:` URL 的 Node.js API（例如 [`fs`][] 模块中的某些函数）遇到了具有不兼容主机的文件 URL。这种情况只能发生在类似 Unix 的系统上，其中只支持 `localhost` 或空主机。

<a id="ERR_INVALID_FILE_URL_PATH"></a>

### `ERR_INVALID_FILE_URL_PATH```

使用 `file:` URL 的 Node.js API（例如 [`fs`][] 模块中的某些函数）遇到了具有不兼容路径的文件 URL。确定路径是否可用的确切语义取决于平台。

抛出的错误对象包括一个 `input` 属性，其中包含无效 `file:` URL 的 URL 对象。

<a id="ERR_INVALID_HANDLE_TYPE"></a>

### `ERR_INVALID_HANDLE_TYPE```

尝试通过 IPC 通信通道向子进程发送不受支持的“句柄”。有关更多信息，请参见 [`subprocess.send()`][] 和 [`process.send()`][]。

<a id="ERR_INVALID_HTTP_TOKEN"></a>

### `ERR_INVALID_HTTP_TOKEN```

提供了无效的 HTTP token。

<a id="ERR_INVALID_IP_ADDRESS"></a>

### `ERR_INVALID_IP_ADDRESS```

IP 地址无效。

<a id="ERR_INVALID_MIME_SYNTAX"></a>

### `ERR_INVALID_MIME_SYNTAX```

MIME 的语法无效。

<a id="ERR_INVALID_MODULE"></a>

### `ERR_INVALID_MODULE`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

尝试加载不存在或无效的模块。

<a id="ERR_INVALID_MODULE_SPECIFIER"></a>

### `ERR_INVALID_MODULE_SPECIFIER```

导入的模块字符串是无效的 URL、包名或包子路径说明符。

<a id="ERR_INVALID_OBJECT_DEFINE_PROPERTY"></a>

### `ERR_INVALID_OBJECT_DEFINE_PROPERTY```

在设置对象属性的无效属性时发生错误。

<a id="ERR_INVALID_PACKAGE_CONFIG"></a>

### `ERR_INVALID_PACKAGE_CONFIG```

无效的 [`package.json`][] 文件解析失败。

<a id="ERR_INVALID_PACKAGE_TARGET"></a>

### `ERR_INVALID_PACKAGE_TARGET```

`package.json` [`"exports"`][] 字段包含针对尝试的模块解析的无效目标映射值。

<a id="ERR_INVALID_PROTOCOL"></a>

### `ERR_INVALID_PROTOCOL```

向 `http.request()` 传递了无效的 `options.protocol`。

<a id="ERR_INVALID_REPL_EVAL_CONFIG"></a>

### `ERR_INVALID_REPL_EVAL_CONFIG```

在 [`REPL`][] 配置中同时设置了 `breakEvalOnSigint` 和 `eval` 选项，这是不支持的。

<a id="ERR_INVALID_REPL_INPUT"></a>

### `ERR_INVALID_REPL_INPUT```

输入不能在 [`REPL`][] 中使用。使用此错误的条件在 [`REPL`][] 文档中描述。

<a id="ERR_INVALID_RETURN_PROPERTY"></a>

### `ERR_INVALID_RETURN_PROPERTY```

在函数选项执行时未为其返回的对象属性之一提供有效值时抛出。

<a id="ERR_INVALID_RETURN_PROPERTY_VALUE"></a>

### `ERR_INVALID_RETURN_PROPERTY_VALUE```

在函数选项执行时未为其返回的对象属性之一提供预期的值类型时抛出。

<a id="ERR_INVALID_RETURN_VALUE"></a>

### `ERR_INVALID_RETURN_VALUE```

在函数选项执行时未返回预期的值类型时抛出，例如当函数预期返回 Promise 时。

<a id="ERR_INVALID_STATE"></a>

### `ERR_INVALID_STATE`

<!-- YAML
added: v15.0.0
-->

表示由于无效状态而无法完成操作。例如，对象可能已经被销毁，或者可能正在执行另一个操作。

<a id="ERR_INVALID_SYNC_FORK_INPUT"></a>

### `ERR_INVALID_SYNC_FORK_INPUT```

向异步 fork 提供了 `Buffer`、`TypedArray`、`DataView` 或 `string` 作为 stdio 输入。有关更多信息，请参见 [`child_process`][] 模块的文档。

<a id="ERR_INVALID_THIS"></a>

### `ERR_INVALID_THIS```

使用不兼容的 `this` 值调用了 Node.js API 函数。

```js
const urlSearchParams = new URLSearchParams('foo=bar&baz=new');

const buf = Buffer.alloc(1);
urlSearchParams.has.call(buf, 'foo');
// 抛出 TypeError，代码为 'ERR_INVALID_THIS'
```

<a id="ERR_INVALID_TUPLE"></a>

### `ERR_INVALID_TUPLE```

提供给 [WHATWG][WHATWG URL API] [`URLSearchParams` 构造函数][`new URLSearchParams(iterable)`] 的 `iterable` 中的元素不表示 `[name, value]` 元组——也就是说，如果元素不可迭代，或者不正好由两个元素组成。

<a id="ERR_INVALID_TYPESCRIPT_SYNTAX"></a>

### `ERR_INVALID_TYPESCRIPT_SYNTAX`

<!-- YAML
added:
 - v23.0.0
 - v22.10.0
changes:
    - version:
      - v23.7.0
      - v22.14.0
      pr-url: https://github.com/nodejs/node/pull/56610
      description: This error is no longer thrown on valid yet unsupported syntax.
-->

提供的 TypeScript 语法无效。

<a id="ERR_INVALID_URI"></a>

### `ERR_INVALID_URI```

传递了无效的 URI。

<a id="ERR_INVALID_URL"></a>

### `ERR_INVALID_URL```

向 [WHATWG][WHATWG URL API] [`URL` 构造函数][`new URL(input)`] 或传统的 [`url.parse()`][] 传递了无效的 URL 进行解析。抛出的错误对象通常有一个额外的属性 `'input'`，其中包含解析失败的 URL。

<a id="ERR_INVALID_URL_PATTERN"></a>

### `ERR_INVALID_URL_PATTERN```

向 [WHATWG][WHATWG URL API] [`URLPattern` 构造函数][`new URLPattern(input)`] 传递了无效的 URLPattern 进行解析。

<a id="ERR_INVALID_URL_SCHEME"></a>

### `ERR_INVALID_URL_SCHEME```

尝试使用不兼容方案（协议）的 URL 用于特定目的。它仅在 [`fs`][] 模块中的 [WHATWG URL API][] 支持中使用（该模块仅接受具有 `'file'` 方案的 URL），但将来也可能在其他 Node.js API 中使用。

<a id="ERR_IPC_CHANNEL_CLOSED"></a>

### `ERR_IPC_CHANNEL_CLOSED```

尝试使用已经关闭的 IPC 通信通道。

<a id="ERR_IPC_DISCONNECTED"></a>

### `ERR_IPC_DISCONNECTED```

尝试断开已经断开的 IPC 通信通道。有关更多信息，请参见 [`child_process`][] 模块的文档。

<a id="ERR_IPC_ONE_PIPE"></a>

### `ERR_IPC_ONE_PIPE```

尝试使用多个 IPC 通信通道创建子 Node.js 进程。有关更多信息，请参见 [`child_process`][] 模块的文档。

<a id="ERR_IPC_SYNC_FORK"></a>

### `ERR_IPC_SYNC_FORK```

尝试与同步派生的 Node.js 进程打开 IPC 通信通道。有关更多信息，请参见 [`child_process`][] 模块的文档。

<a id="ERR_IP_BLOCKED"></a>

### `ERR_IP_BLOCKED```

IP 被 `net.BlockList` 阻止。

<a id="ERR_LOADER_CHAIN_INCOMPLETE"></a>

### `ERR_LOADER_CHAIN_INCOMPLETE`

<!-- YAML
added:
  - v18.6.0
  - v16.17.0
-->

ESM 加载器钩子返回时没有调用 `next()` 且没有明确发出短路信号。

<a id="ERR_LOAD_SQLITE_EXTENSION"></a>

### `ERR_LOAD_SQLITE_EXTENSION`

<!-- YAML
added:
  - v23.5.0
  - v22.13.0
-->

加载 SQLite 扩展时发生错误。

<a id="ERR_MEMORY_ALLOCATION_FAILED"></a>

### `ERR_MEMORY_ALLOCATION_FAILED```

尝试分配内存（通常在 C++ 层）但失败。

<a id="ERR_MESSAGE_TARGET_CONTEXT_UNAVAILABLE"></a>

### `ERR_MESSAGE_TARGET_CONTEXT_UNAVAILABLE`

<!-- YAML
added:
  - v14.5.0
  - v12.19.0
-->

发布到 [`MessagePort`][] 的消息无法在目标 [vm][] `Context` 中反序列化。目前，并非所有 Node.js 对象都可以在任何上下文中成功实例化，尝试使用 `postMessage()` 传输它们在这种情况下可能会在接收端失败。

<a id="ERR_METHOD_NOT_IMPLEMENTED"></a>

### `ERR_METHOD_NOT_IMPLEMENTED```

需要方法但未实现。

<a id="ERR_MISSING_ARGS"></a>

### `ERR_MISSING_ARGS```

未传递 Node.js API 的必需参数。这仅用于严格符合 API 规范（在某些情况下可能接受 `func(undefined)` 但不接受 `func()`）。在大多数原生 Node.js API 中，`func(undefined)` 和 `func()` 被视为相同，并且可能改用 [`ERR_INVALID_ARG_TYPE`][] 错误代码。

<a id="ERR_MISSING_OPTION"></a>

### `ERR_MISSING_OPTION```

对于接受选项对象的 API，某些选项可能是强制性的。如果缺少必需选项，则抛出此代码。

<a id="ERR_MISSING_PASSPHRASE"></a>

### `ERR_MISSING_PASSPHRASE```

尝试读取加密密钥但未指定密码。

<a id="ERR_MISSING_PLATFORM_FOR_WORKER"></a>

### `ERR_MISSING_PLATFORM_FOR_WORKER```

此 Node.js 实例使用的 V8 平台不支持创建 Workers。这是由于缺少对 Workers 的嵌入器支持。特别是，标准构建的 Node.js 不会发生此错误。

<a id="ERR_MODULE_LINK_MISMATCH"></a>

### `ERR_MODULE_LINK_MISMATCH```

模块无法链接，因为其中的相同模块请求未解析为同一模块。

<a id="ERR_MODULE_NOT_FOUND"></a>

### `ERR_MODULE_NOT_FOUND```

在尝试 `import` 操作或加载程序入口点时，ECMAScript 模块加载器无法解析模块文件。

<a id="ERR_MULTIPLE_CALLBACK"></a>

### `ERR_MULTIPLE_CALLBACK```

回调被多次调用。

回调几乎总是意味着只被调用一次，因为查询要么被满足，要么被拒绝，但不能同时发生。后者可能通过多次调用回调而发生。

<a id="ERR_NAPI_CONS_FUNCTION"></a>

### `ERR_NAPI_CONS_FUNCTION```

在使用 `Node-API` 时，传递的构造函数不是函数。

<a id="ERR_NAPI_INVALID_DATAVIEW_ARGS"></a>

### `ERR_NAPI_INVALID_DATAVIEW_ARGS```

调用 `napi_create_dataview()` 时，给定的 `offset` 超出 dataview 的边界，或 `offset + length` 大于给定 `buffer` 的长度。

<a id="ERR_NAPI_INVALID_TYPEDARRAY_ALIGNMENT"></a>

### `ERR_NAPI_INVALID_TYPEDARRAY_ALIGNMENT```

调用 `napi_create_typedarray()` 时，提供的 `offset` 不是元素大小的倍数。

<a id="ERR_NAPI_INVALID_TYPEDARRAY_LENGTH"></a>

### `ERR_NAPI_INVALID_TYPEDARRAY_LENGTH```

调用 `napi_create_typedarray()` 时，`(length * size_of_element) + byte_offset` 大于给定 `buffer` 的长度。

<a id="ERR_NAPI_TSFN_CALL_JS"></a>

### `ERR_NAPI_TSFN_CALL_JS```

调用线程安全函数的 JavaScript 部分时发生错误。

<a id="ERR_NAPI_TSFN_GET_UNDEFINED"></a>

### `ERR_NAPI_TSFN_GET_UNDEFINED```

尝试检索 JavaScript `undefined` 值时发生错误。

<a id="ERR_NON_CONTEXT_AWARE_DISABLED"></a>

### `ERR_NON_CONTEXT_AWARE_DISABLED```

在禁止非上下文感知原生插件的进程中加载了非上下文感知原生插件。

<a id="ERR_NOT_BUILDING_SNAPSHOT"></a>

### `ERR_NOT_BUILDING_SNAPSHOT```

尝试使用只能在构建 V8 启动快照时使用的操作，但 Node.js 并未构建快照。

<a id="ERR_NOT_IN_SINGLE_EXECUTABLE_APPLICATION"></a>

### `ERR_NOT_IN_SINGLE_EXECUTABLE_APPLICATION`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

当不在单可执行应用程序中时，无法执行该操作。

<a id="ERR_NOT_SUPPORTED_IN_SNAPSHOT"></a>

### `ERR_NOT_SUPPORTED_IN_SNAPSHOT```

尝试执行在构建启动快照时不支持的操作。

<a id="ERR_NO_CRYPTO"></a>

### `ERR_NO_CRYPTO```

尝试使用加密功能，但 Node.js 编译时未包含 OpenSSL 加密支持。

<a id="ERR_NO_ICU"></a>

### `ERR_NO_ICU```

尝试使用需要 [ICU][] 的功能，但 Node.js 编译时未包含 ICU 支持。

<a id="ERR_NO_TYPESCRIPT"></a>

### `ERR_NO_TYPESCRIPT`

<!-- YAML
added:
  - v23.0.0
  - v22.12.0
-->

尝试使用需要 [原生 TypeScript 支持][Native TypeScript support] 的功能，但 Node.js 编译时未包含 TypeScript 支持。

<a id="ERR_OPERATION_FAILED"></a>

### `ERR_OPERATION_FAILED`

<!-- YAML
added: v15.0.0
-->

操作失败。这通常用于表示异步操作的一般失败。

<a id="ERR_OPTIONS_BEFORE_BOOTSTRAPPING"></a>

### `ERR_OPTIONS_BEFORE_BOOTSTRAPPING`

<!-- YAML
added: v23.10.0
-->

尝试在引导完成之前获取选项。

<a id="ERR_OUT_OF_RANGE"></a>

### `ERR_OUT_OF_RANGE```

给定的值超出可接受范围。

<a id="ERR_PACKAGE_IMPORT_NOT_DEFINED"></a>

### `ERR_PACKAGE_IMPORT_NOT_DEFINED```

`package.json` [`"imports"`][] 字段未定义给定的内部包说明符映射。

<a id="ERR_PACKAGE_PATH_NOT_EXPORTED"></a>

### `ERR_PACKAGE_PATH_NOT_EXPORTED```

`package.json` [`"exports"`][] 字段未导出请求的子路径。由于导出是封装的，未导出的私有内部模块无法通过包解析导入，除非使用绝对 URL。

<a id="ERR_PARSE_ARGS_INVALID_OPTION_VALUE"></a>

### `ERR_PARSE_ARGS_INVALID_OPTION_VALUE`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

当 `strict` 设置为 `true` 时，如果为 {string} 类型的选项提供了 {boolean} 值，或为 {boolean} 类型的选项提供了 {string} 值，则由 [`util.parseArgs()`][] 抛出。

<a id="ERR_PARSE_ARGS_UNEXPECTED_POSITIONAL"></a>

### `ERR_PARSE_ARGS_UNEXPECTED_POSITIONAL`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

当提供了位置参数且 `allowPositionals` 设置为 `false` 时，由 [`util.parseArgs()`][] 抛出。

<a id="ERR_PARSE_ARGS_UNKNOWN_OPTION"></a>

### `ERR_PARSE_ARGS_UNKNOWN_OPTION`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

当 `strict` 设置为 `true` 时，如果参数未在 `options` 中配置，则由 [`util.parseArgs()`][] 抛出。

<a id="ERR_PERFORMANCE_INVALID_TIMESTAMP"></a>

### `ERR_PERFORMANCE_INVALID_TIMESTAMP```

为性能标记或测量提供了无效的时间戳值。

<a id="ERR_PERFORMANCE_MEASURE_INVALID_OPTIONS"></a>

### `ERR_PERFORMANCE_MEASURE_INVALID_OPTIONS```

为性能测量提供了无效的选项。

<a id="ERR_PROTO_ACCESS"></a>

### `ERR_PROTO_ACCESS```

使用 [`--disable-proto=throw`][] 禁止访问 `Object.prototype.__proto__`。应使用 [`Object.getPrototypeOf`][] 和 [`Object.setPrototypeOf`][] 来获取和设置对象的原型。

<a id="ERR_PROXY_INVALID_CONFIG"></a>

### `ERR_PROXY_INVALID_CONFIG```

由于代理配置无效，无法代理请求。

<a id="ERR_PROXY_TUNNEL"></a>

### `ERR_PROXY_TUNNEL```

当启用 `NODE_USE_ENV_PROXY` 或 `--use-env-proxy` 时，无法建立代理隧道。

<a id="ERR_QUIC_APPLICATION_ERROR"></a>

### `ERR_QUIC_APPLICATION_ERROR`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
-->

> Stability: 1 - Experimental

发生 QUIC 应用程序错误。

<a id="ERR_QUIC_CONNECTION_FAILED"></a>

### `ERR_QUIC_CONNECTION_FAILED`

<!-- YAML
added:
 - v23.0.0
 - v22.10.0
-->

> Stability: 1 - Experimental

建立 QUIC 连接失败。

<a id="ERR_QUIC_ENDPOINT_CLOSED"></a>

### `ERR_QUIC_ENDPOINT_CLOSED`

<!-- YAML
added:
 - v23.0.0
 - v22.10.0
-->

> Stability: 1 - Experimental

QUIC Endpoint 关闭并出现错误。

<a id="ERR_QUIC_OPEN_STREAM_FAILED"></a>

### `ERR_QUIC_OPEN_STREAM_FAILED`

<!-- YAML
added:
 - v23.0.0
 - v22.10.0
-->

> Stability: 1 - Experimental

打开 QUIC 流失败。

<a id="ERR_QUIC_TRANSPORT_ERROR"></a>

### `ERR_QUIC_TRANSPORT_ERROR`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
-->

> Stability: 1 - Experimental

发生 QUIC 传输错误。

<a id="ERR_QUIC_VERSION_NEGOTIATION_ERROR"></a>

### `ERR_QUIC_VERSION_NEGOTIATION_ERROR`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
-->

> Stability: 1 - Experimental

QUIC 会话失败，因为需要版本协商。

<a id="ERR_REQUIRE_ASYNC_MODULE"></a>

### `ERR_REQUIRE_ASYNC_MODULE```

> Stability: 1 - Experimental

尝试 `require()` 一个 [ES 模块][ES Module] 时，该模块是异步的。也就是说，它包含顶级 await。

要查看顶级 await 的位置，请使用 `--experimental-print-required-tla`（这将在查找顶级 await 之前执行模块）。

<a id="ERR_REQUIRE_CYCLE_MODULE"></a>

### `ERR_REQUIRE_CYCLE_MODULE```

> Stability: 1 - Experimental

尝试 `require()` 一个 [ES 模块][ES Module] 时，CommonJS 到 ESM 或 ESM 到 CommonJS 的边参与了一个即时循环。
这是不允许的，因为 ES 模块在已经正在评估时无法被评估。

为了避免循环，参与循环的 `require()` 调用不应发生在 ES 模块（通过 `createRequire()`）或 CommonJS 模块的顶级，而应在内部函数中延迟完成。

<a id="ERR_REQUIRE_ESM"></a>

### `ERR_REQUIRE_ESM`

<!-- YAML
changes:
  - version:
    - v23.0.0
    - v22.12.0
    - v20.19.0
    pr-url: https://github.com/nodejs/node/pull/55085
    description: require() now supports loading synchronous ES modules by default.
-->

> Stability: 0 - Deprecated

尝试 `require()` 一个 [ES 模块][ES Module]。

此错误已被弃用，因为 `require()` 现在支持加载同步 ES 模块。当 `require()` 遇到包含顶级 `await` 的 ES 模块时，它将抛出 [`ERR_REQUIRE_ASYNC_MODULE`][] 代替。

<a id="ERR_SCRIPT_EXECUTION_INTERRUPTED"></a>

### `ERR_SCRIPT_EXECUTION_INTERRUPTED```

脚本执行被 `SIGINT` 中断（例如，按下了 <kbd>Ctrl</kbd>+<kbd>C</kbd>）。

<a id="ERR_SCRIPT_EXECUTION_TIMEOUT"></a>

### `ERR_SCRIPT_EXECUTION_TIMEOUT```

脚本执行超时，可能是由于正在执行的脚本中存在错误。

<a id="ERR_SERVER_ALREADY_LISTEN"></a>

### `ERR_SERVER_ALREADY_LISTEN```

在 `net.Server` 已经在监听时调用了 [`server.listen()`][] 方法。这适用于所有 `net.Server` 实例，包括 HTTP、HTTPS 和 HTTP/2 `Server` 实例。

<a id="ERR_SERVER_NOT_RUNNING"></a>

### `ERR_SERVER_NOT_RUNNING```

在 `net.Server` 未运行时调用了 [`server.close()`][] 方法。这适用于所有 `net.Server` 实例，包括 HTTP、HTTPS 和 HTTP/2 `Server` 实例。

<a id="ERR_SINGLE_EXECUTABLE_APPLICATION_ASSET_NOT_FOUND"></a>

### `ERR_SINGLE_EXECUTABLE_APPLICATION_ASSET_NOT_FOUND`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

向单可执行应用程序 API 传递了用于标识资源的键，但找不到匹配项。

<a id="ERR_SOCKET_ALREADY_BOUND"></a>

### `ERR_SOCKET_ALREADY_BOUND```

尝试绑定已经绑定的套接字。

<a id="ERR_SOCKET_BAD_BUFFER_SIZE"></a>

### `ERR_SOCKET_BAD_BUFFER_SIZE```

在 [`dgram.createSocket()`][] 中为 `recvBufferSize` 或 `sendBufferSize` 选项传递了无效（负）大小。

<a id="ERR_SOCKET_BAD_PORT"></a>

### `ERR_SOCKET_BAD_PORT```

期望端口 >= 0 且 < 65536 的 API 函数收到了无效值。

<a id="ERR_SOCKET_BAD_TYPE"></a>

### `ERR_SOCKET_BAD_TYPE```

期望套接字类型（`udp4` 或 `udp6`）的 API 函数收到了无效值。

<a id="ERR_SOCKET_BUFFER_SIZE"></a>

### `ERR_SOCKET_BUFFER_SIZE```

在使用 [`dgram.createSocket()`][] 时，无法确定接收或发送 `Buffer` 的大小。

<a id="ERR_SOCKET_CLOSED"></a>

### `ERR_SOCKET_CLOSED```

尝试对已经关闭的套接字进行操作。

<a id="ERR_SOCKET_CLOSED_BEFORE_CONNECTION"></a>

### `ERR_SOCKET_CLOSED_BEFORE_CONNECTION```

在连接套接字上调用 [`net.Socket.write()`][] 时，套接字在连接建立之前关闭。

<a id="ERR_SOCKET_CONNECTION_TIMEOUT"></a>

### `ERR_SOCKET_CONNECTION_TIMEOUT```

在使用族自动选择算法时，套接字无法在允许的超时时间内连接到 DNS 返回的任何地址。

<a id="ERR_SOCKET_DGRAM_IS_CONNECTED"></a>

### `ERR_SOCKET_DGRAM_IS_CONNECTED```

在已经连接的套接字上调用了 [`dgram.connect()`][]。

<a id="ERR_SOCKET_DGRAM_NOT_CONNECTED"></a>

### `ERR_SOCKET_DGRAM_NOT_CONNECTED```

在断开的套接字上调用了 [`dgram.disconnect()`][] 或 [`dgram.remoteAddress()`][]。

<a id="ERR_SOCKET_DGRAM_NOT_RUNNING"></a>

### `ERR_SOCKET_DGRAM_NOT_RUNNING```

进行了调用，但 UDP 子系统未运行。

<a id="ERR_SOURCE_MAP_CORRUPT"></a>

### `ERR_SOURCE_MAP_CORRUPT```

无法解析源映射，因为它不存在或已损坏。

<a id="ERR_SOURCE_MAP_MISSING_SOURCE"></a>

### `ERR_SOURCE_MAP_MISSING_SOURCE```

未找到从源映射导入的文件。

<a id="ERR_SOURCE_PHASE_NOT_DEFINED"></a>

### `ERR_SOURCE_PHASE_NOT_DEFINED`

<!-- YAML
added: v24.0.0
-->

提供的模块导入未为源阶段导入语法 `import source x from 'x'` 或 `import.source(x)` 提供源阶段导入表示。

<a id="ERR_SQLITE_ERROR"></a>

### `ERR_SQLITE_ERROR`

<!-- YAML
added: v22.5.0
-->

从 [SQLite][] 返回错误。

<a id="ERR_SRI_PARSE"></a>

### `ERR_SRI_PARSE```

为子资源完整性检查提供了字符串，但无法解析。通过查看[子资源完整性规范][Subresource Integrity specification]检查完整性属性的格式。

<a id="ERR_STREAM_ALREADY_FINISHED"></a>

### `ERR_STREAM_ALREADY_FINISHED```

调用了无法完成的流方法，因为流已结束。

<a id="ERR_STREAM_CANNOT_PIPE"></a>

### `ERR_STREAM_CANNOT_PIPE```

尝试在 [`Writable`][] 流上调用 [`stream.pipe()`][]。

<a id="ERR_STREAM_DESTROYED"></a>

### `ERR_STREAM_DESTROYED```

调用了无法完成的流方法，因为流已使用 `stream.destroy()` 销毁。

<a id="ERR_STREAM_NULL_VALUES"></a>

### `ERR_STREAM_NULL_VALUES```

尝试使用 `null` 块调用 [`stream.write()`][]。

<a id="ERR_STREAM_PREMATURE_CLOSE"></a>

### `ERR_STREAM_PREMATURE_CLOSE```

由 `stream.finished()` 和 `stream.pipeline()` 返回的错误，当流或管道非正常结束且没有显式错误时。

<a id="ERR_STREAM_PUSH_AFTER_EOF"></a>

### `ERR_STREAM_PUSH_AFTER_EOF```

在将 `null`(EOF) 推送到流后尝试调用 [`stream.push()`][]。

<a id="ERR_STREAM_UNABLE_TO_PIPE"></a>

### `ERR_STREAM_UNABLE_TO_PIPE```

尝试在管道中向已关闭或销毁的流进行管道传输。

<a id="ERR_STREAM_UNSHIFT_AFTER_END_EVENT"></a>

### `ERR_STREAM_UNSHIFT_AFTER_END_EVENT```

在发出 `'end'` 事件后尝试调用 [`stream.unshift()`][]。

<a id="ERR_STREAM_WRAP"></a>

### `ERR_STREAM_WRAP```

如果在 Socket 上设置了字符串解码器，或者解码器处于 `objectMode`，则防止中止。

```js
const Socket = require('node:net').Socket;
const instance = new Socket();

instance.setEncoding('utf8');
```

<a id="ERR_STREAM_WRITE_AFTER_END"></a>

### `ERR_STREAM_WRITE_AFTER_END```

在调用 `stream.end()` 后尝试调用 [`stream.write()`][]。

<a id="ERR_STRING_TOO_LONG"></a>

### `ERR_STRING_TOO_LONG```

尝试创建超过最大允许长度的字符串。

<a id="ERR_SYNTHETIC"></a>

### `ERR_SYNTHETIC```

用于捕获诊断报告调用栈的人工错误对象。

<a id="ERR_SYSTEM_ERROR"></a>

### `ERR_SYSTEM_ERROR```

在 Node.js 进程内发生了未指定或非特定的系统错误。错误对象将具有一个 `err.info` 对象属性，其中包含其他详细信息。

<a id="ERR_TEST_FAILURE"></a>

### `ERR_TEST_FAILURE```

此错误表示测试失败。有关失败的更多信息可通过 `cause` 属性获得。`failureType` 属性指定失败发生时测试正在执行的操作。

<a id="ERR_TLS_ALPN_CALLBACK_INVALID_RESULT"></a>

### `ERR_TLS_ALPN_CALLBACK_INVALID_RESULT```

当 `ALPNCallback` 返回的值不在客户端提供的 ALPN 协议列表中时抛出此错误。

<a id="ERR_TLS_ALPN_CALLBACK_WITH_PROTOCOLS"></a>

### `ERR_TLS_ALPN_CALLBACK_WITH_PROTOCOLS```

如果在创建 `TLSServer` 时 TLS 选项同时包含 `ALPNProtocols` 和 `ALPNCallback`，则抛出此错误。这些选项是互斥的。

<a id="ERR_TLS_CERT_ALTNAME_FORMAT"></a>

### `ERR_TLS_CERT_ALTNAME_FORMAT```

如果用户提供的 `subjectaltname` 属性违反编码规则，则由 `checkServerIdentity` 抛出。Node.js 本身生成的证书对象始终符合编码规则，永远不会导致此错误。

<a id="ERR_TLS_CERT_ALTNAME_INVALID"></a>

### `ERR_TLS_CERT_ALTNAME_INVALID```

在使用 TLS 时，对端的主机名/IP 与其证书中的任何 `subjectAltNames` 都不匹配。

<a id="ERR_TLS_DH_PARAM_SIZE"></a>

### `ERR_TLS_DH_PARAM_SIZE```

在使用 TLS 时，为 Diffie-Hellman (`DH`) 密钥协商协议提供的参数太小。默认情况下，密钥长度必须大于或等于 1024 位以避免漏洞，尽管强烈建议使用 2048 位或更大以获得更强的安全性。

<a id="ERR_TLS_HANDSHAKE_TIMEOUT"></a>

### `ERR_TLS_HANDSHAKE_TIMEOUT```

TLS/SSL 握手超时。在这种情况下，服务器也必须中止连接。

<a id="ERR_TLS_INVALID_CONTEXT"></a>

### `ERR_TLS_INVALID_CONTEXT`

<!-- YAML
added: v13.3.0
-->

上下文必须是 `SecureContext`。

<a id="ERR_TLS_INVALID_PROTOCOL_METHOD"></a>

### `ERR_TLS_INVALID_PROTOCOL_METHOD```

指定的 `secureProtocol` 方法无效。它要么未知，要么由于不安全而被禁用。

<a id="ERR_TLS_INVALID_PROTOCOL_VERSION"></a>

### `ERR_TLS_INVALID_PROTOCOL_VERSION```

有效的 TLS 协议版本是 `'TLSv1'`、`'TLSv1.1'` 或 `'TLSv1.2'`。

<a id="ERR_TLS_INVALID_STATE"></a>

### `ERR_TLS_INVALID_STATE`

<!-- YAML
added:
 - v13.10.0
 - v12.17.0
-->

TLS 套接字必须已连接并安全建立。确保在继续之前发出 'secure' 事件。

<a id="ERR_TLS_PROTOCOL_VERSION_CONFLICT"></a>

### `ERR_TLS_PROTOCOL_VERSION_CONFLICT```

尝试设置 TLS 协议 `minVersion` 或 `maxVersion` 与显式设置 `secureProtocol` 的尝试冲突。使用一种机制或另一种。

<a id="ERR_TLS_PSK_SET_IDENTITY_HINT_FAILED"></a>

### `ERR_TLS_PSK_SET_IDENTITY_HINT_FAILED```

设置 PSK 身份提示失败。提示可能太长。

<a id="ERR_TLS_RENEGOTIATION_DISABLED"></a>

### `ERR_TLS_RENEGOTIATION_DISABLED```

尝试在禁用重新协商的套接字实例上重新协商 TLS。

<a id="ERR_TLS_REQUIRED_SERVER_NAME"></a>

### `ERR_TLS_REQUIRED_SERVER_NAME```

在使用 TLS 时，调用 `server.addContext()` 方法时未在第一个参数中提供主机名。

<a id="ERR_TLS_SESSION_ATTACK"></a>

### `ERR_TLS_SESSION_ATTACK```

检测到过多的 TLS 重新协商，这是拒绝服务攻击的潜在载体。

<a id="ERR_TLS_SNI_FROM_SERVER"></a>

### `ERR_TLS_SNI_FROM_SERVER```

尝试从 TLS 服务器端套接字发出服务器名称指示，这仅对客户端有效。

<a id="ERR_TRACE_EVENTS_CATEGORY_REQUIRED"></a>

### `ERR_TRACE_EVENTS_CATEGORY_REQUIRED```

`trace_events.createTracing()` 方法需要至少一个跟踪事件类别。

<a id="ERR_TRACE_EVENTS_UNAVAILABLE"></a>

### `ERR_TRACE_EVENTS_UNAVAILABLE```

无法加载 `node:trace_events` 模块，因为 Node.js 是使用 `--without-v8-platform` 标志编译的。

<a id="ERR_TRAILING_JUNK_AFTER_STREAM_END"></a>

### `ERR_TRAILING_JUNK_AFTER_STREAM_END```

在压缩流末尾之后发现尾随垃圾。
当在压缩流（例如，在 zlib 或 gzip 解压缩中）末尾检测到额外的意外数据时抛出此错误。

<a id="ERR_TRANSFORM_ALREADY_TRANSFORMING"></a>

### `ERR_TRANSFORM_ALREADY_TRANSFORMING```

`Transform` 流在仍在转换时结束。

<a id="ERR_TRANSFORM_WITH_LENGTH_0"></a>

### `ERR_TRANSFORM_WITH_LENGTH_0```

`Transform` 流结束时写缓冲区中仍有数据。

<a id="ERR_TTY_INIT_FAILED"></a>

### `ERR_TTY_INIT_FAILED```

由于系统错误，TTY 初始化失败。

<a id="ERR_UNAVAILABLE_DURING_EXIT"></a>

### `ERR_UNAVAILABLE_DURING_EXIT```

在 [`process.on('exit')`][] 处理程序中调用了不应在 [`process.on('exit')`][] 处理程序中调用的函数。

<a id="ERR_UNCAUGHT_EXCEPTION_CAPTURE_ALREADY_SET"></a>

### `ERR_UNCAUGHT_EXCEPTION_CAPTURE_ALREADY_SET```

[`process.setUncaughtExceptionCaptureCallback()`][] 被调用了两次，但未首先将回调重置为 `null`。

此错误旨在防止意外覆盖从另一个模块注册的回调。

<a id="ERR_UNESCAPED_CHARACTERS"></a>

### `ERR_UNESCAPED_CHARACTERS```

收到了包含未转义字符的字符串。

<a id="ERR_UNHANDLED_ERROR"></a>

### `ERR_UNHANDLED_ERROR```

发生未处理的错误（例如，当 [`EventEmitter`][] 发出 `'error'` 事件但未注册 `'error'` 处理程序时）。

<a id="ERR_UNKNOWN_BUILTIN_MODULE"></a>

### `ERR_UNKNOWN_BUILTIN_MODULE```

用于识别特定类型的内部 Node.js 错误，通常不应由用户代码触发。此错误的实例指向 Node.js 二进制文件本身的内部错误。

<a id="ERR_UNKNOWN_CREDENTIAL"></a>

### `ERR_UNKNOWN_CREDENTIAL```

传递了不存在的 Unix 组或用户标识符。

<a id="ERR_UNKNOWN_ENCODING"></a>

### `ERR_UNKNOWN_ENCODING```

向 API 传递了无效或未知的编码选项。

<a id="ERR_UNKNOWN_FILE_EXTENSION"></a>

### `ERR_UNKNOWN_FILE_EXTENSION```

尝试加载具有未知或不支持文件扩展名的模块。

<a id="ERR_UNKNOWN_MODULE_FORMAT"></a>

### `ERR_UNKNOWN_MODULE_FORMAT```

尝试加载具有未知或不支持格式的模块。

<a id="ERR_UNKNOWN_SIGNAL"></a>

### `ERR_UNKNOWN_SIGNAL```

向期望有效信号的 API 传递了无效或未知的进程信号（例如 [`subprocess.kill()`][]）。

<a id="ERR_UNSUPPORTED_DIR_IMPORT"></a>

### `ERR_UNSUPPORTED_DIR_IMPORT```

`import` 目录 URL 不受支持。相反，使用[其名称自引用包][self-reference a package using its name]并在 [`package.json`][] 文件的 [`"exports"`][] 字段中[定义自定义子路径][define a custom subpath]。

```mjs
import './'; // 不支持
import './index.js'; // 支持
import 'package-name'; // 支持
```

<a id="ERR_UNSUPPORTED_ESM_URL_SCHEME"></a>

### `ERR_UNSUPPORTED_ESM_URL_SCHEME```

不支持 `file` 和 `data` 以外的 URL 方案的 `import`。

<a id="ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING"></a>

### `ERR_UNSUPPORTED_NODE_MODULES_TYPE_STRIPPING`

<!-- YAML
added: v22.6.0
-->

不支持对 `node_modules` 目录后代文件进行类型剥离。

<a id="ERR_UNSUPPORTED_RESOLVE_REQUEST"></a>

### `ERR_UNSUPPORTED_RESOLVE_REQUEST```

尝试解析无效的模块引用者。这可能在以下情况下发生：

* 从 URL 方案不是 `file` 的模块导入或调用 `import.meta.resolve()` 时使用裸说明符。
* 从 URL 方案不是[特殊方案][special scheme]的模块使用[相对 URL][relative URL]。

```mjs
try {
  // 尝试从 `data:` URL 模块导入包 'bare-specifier'：
  await import('data:text/javascript,import "bare-specifier"');
} catch (e) {
  console.log(e.code); // ERR_UNSUPPORTED_RESOLVE_REQUEST
}
```

<a id="ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX"></a>

### `ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`

<!-- YAML
added:
  - v23.7.0
  - v22.14.0
-->

提供的 TypeScript 语法不受支持。
这可能在使用需要[类型剥离][type-stripping]进行转换的 TypeScript 语法时发生。

<a id="ERR_USE_AFTER_CLOSE"></a>

### `ERR_USE_AFTER_CLOSE```

尝试使用已经关闭的内容。

<a id="ERR_VALID_PERFORMANCE_ENTRY_TYPE"></a>

### `ERR_VALID_PERFORMANCE_ENTRY_TYPE```

在使用性能计时 API (`perf_hooks`) 时，未找到有效的性能条目类型。

<a id="ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING"></a>

### `ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING```

未指定动态导入回调。

<a id="ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING_FLAG"></a>

### `ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING_FLAG```

在没有 `--experimental-vm-modules` 的情况下调用了动态导入回调。

<a id="ERR_VM_MODULE_ALREADY_LINKED"></a>

### `ERR_VM_MODULE_ALREADY_LINKED```

尝试链接的模块不符合链接条件，原因如下：

* 它已经被链接（`linkingStatus` 是 `'linked'`）
* 它正在被链接（`linkingStatus` 是 `'linking'`）
* 此模块的链接失败（`linkingStatus` 是 `'errored'`）

<a id="ERR_VM_MODULE_CACHED_DATA_REJECTED"></a>

### `ERR_VM_MODULE_CACHED_DATA_REJECTED```

传递给模块构造函数的 `cachedData` 选项无效。

<a id="ERR_VM_MODULE_CANNOT_CREATE_CACHED_DATA"></a>

### `ERR_VM_MODULE_CANNOT_CREATE_CACHED_DATA```

无法为已经评估的模块创建缓存数据。

<a id="ERR_VM_MODULE_DIFFERENT_CONTEXT"></a>

### `ERR_VM_MODULE_DIFFERENT_CONTEXT```

从链接器函数返回的模块与父模块来自不同的上下文。链接的模块必须共享相同的上下文。

<a id="ERR_VM_MODULE_LINK_FAILURE"></a>

### `ERR_VM_MODULE_LINK_FAILURE```

由于失败，模块无法链接。

<a id="ERR_VM_MODULE_NOT_MODULE"></a>

### `ERR_VM_MODULE_NOT_MODULE```

链接承诺的履行值不是 `vm.Module` 对象。

<a id="ERR_VM_MODULE_STATUS"></a>

### `ERR_VM_MODULE_STATUS```

当前模块的状态不允许此操作。错误的具体含义取决于特定函数。

<a id="ERR_WASI_ALREADY_STARTED"></a>

### `ERR_WASI_ALREADY_STARTED```

WASI 实例已经启动。

<a id="ERR_WASI_NOT_STARTED"></a>

### `ERR_WASI_NOT_STARTED```

WASI 实例尚未启动。

<a id="ERR_WEBASSEMBLY_RESPONSE"></a>

### `ERR_WEBASSEMBLY_RESPONSE`

<!-- YAML
added: v18.1.0
-->

传递给 `WebAssembly.compileStreaming` 或 `WebAssembly.instantiateStreaming` 的 `Response` 不是有效的 WebAssembly 响应。

<a id="ERR_WORKER_INIT_FAILED"></a>

### `ERR_WORKER_INIT_FAILED```

`Worker` 初始化失败。

<a id="ERR_WORKER_INVALID_EXEC_ARGV"></a>

### `ERR_WORKER_INVALID_EXEC_ARGV```

传递给 `Worker` 构造函数的 `execArgv` 选项包含无效标志。

<a id="ERR_WORKER_MESSAGING_ERRORED"></a>

### `ERR_WORKER_MESSAGING_ERRORED`

<!-- YAML
added: v22.5.0
-->

> Stability: 1.1 - Active development

目标线程在处理通过 [`postMessageToThread()`][] 发送的消息时抛出错误。

<a id="ERR_WORKER_MESSAGING_FAILED"></a>

### `ERR_WORKER_MESSAGING_FAILED`

<!-- YAML
added: v22.5.0
-->

> Stability: 1.1 - Active development

[`postMessageToThread()`][] 中请求的线程无效或没有 `workerMessage` 监听器。

<a id="ERR_WORKER_MESSAGING_SAME_THREAD"></a>

### `ERR_WORKER_MESSAGING_SAME_THREAD`

<!-- YAML
added: v22.5.0
-->

> Stability: 1.1 - Active development

[`postMessageToThread()`][] 中请求的线程 ID 是当前线程 ID。

<a id="ERR_WORKER_MESSAGING_TIMEOUT"></a>

### `ERR_WORKER_MESSAGING_TIMEOUT`

<!-- YAML
added: v22.5.0
-->

> Stability: 1.1 - Active development

通过 [`postMessageToThread()`][] 发送消息超时。

<a id="ERR_WORKER_NOT_RUNNING"></a>

### `ERR_WORKER_NOT_RUNNING```

操作失败，因为 `Worker` 实例当前未运行。

<a id="ERR_WORKER_OUT_OF_MEMORY"></a>

### `ERR_WORKER_OUT_OF_MEMORY```

`Worker` 实例因达到其内存限制而终止。

<a id="ERR_WORKER_PATH"></a>

### `ERR_WORKER_PATH```

工作器主脚本的路径既不是绝对路径，也不是以 `./` 或 `../` 开头的相对路径。

<a id="ERR_WORKER_UNSERIALIZABLE_ERROR"></a>

### `ERR_WORKER_UNSERIALIZABLE_ERROR```

所有尝试序列化工作线程中未捕获异常的操作都失败了。

<a id="ERR_WORKER_UNSUPPORTED_OPERATION"></a>

### `ERR_WORKER_UNSUPPORTED_OPERATION```

请求的功能在工作线程中不受支持。

<a id="ERR_ZLIB_INITIALIZATION_FAILED"></a>

### `ERR_ZLIB_INITIALIZATION_FAILED```

由于配置不正确，创建 [`zlib`][] 对象失败。

<a id="ERR_ZSTD_INVALID_PARAM"></a>

### `ERR_ZSTD_INVALID_PARAM```

在构建 Zstd 流期间传递了无效的参数键。

<a id="HPE_CHUNK_EXTENSIONS_OVERFLOW"></a>

### `HPE_CHUNK_EXTENSIONS_OVERFLOW`

<!-- YAML
added:
 - v21.6.2
 - v20.11.1
 - v18.19.1
-->

为块扩展接收了太多数据。为了保护免受恶意或配置错误的客户端的影响，如果接收到超过 16 KiB 的数据，则将发出带有此代码的 `Error`。

<a id="HPE_HEADER_OVERFLOW"></a>

### `HPE_HEADER_OVERFLOW`

<!-- YAML
changes:
  - version:
     - v11.4.0
     - v10.15.0
    commit: 186035243fad247e3955f
    pr-url: https://github.com/nodejs-private/node-private/pull/143
    description: Max header size in `http_parser` was set to 8 KiB.
-->

接收了太多的 HTTP 头部数据。为了保护免受恶意或配置错误的客户端的影响，如果接收到超过 `maxHeaderSize` 的 HTTP 头部数据，则 HTTP 解析将中止，不会创建请求或响应对象，并将发出带有此代码的 `Error`。

<a id="HPE_UNEXPECTED_CONTENT_LENGTH"></a>

### `HPE_UNEXPECTED_CONTENT_LENGTH```

服务器同时发送 `Content-Length` 头部和 `Transfer-Encoding: chunked`。

`Transfer-Encoding: chunked` 允许服务器为动态生成的内容维护 HTTP 持久连接。
在这种情况下，不能使用 `Content-Length` HTTP 头部。

使用 `Content-Length` 或 `Transfer-Encoding: chunked`。

<a id="MODULE_NOT_FOUND"></a>

### `MODULE_NOT_FOUND`

<!-- YAML
changes:
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/25690
    description: Added `requireStack` property.
-->

在尝试 [`require()`][] 操作或加载程序入口点时，CommonJS 模块加载器无法解析模块文件。

## 传统 Node.js 错误代码

> Stability: 0 - Deprecated. 这些错误代码要么不一致，要么已被移除。

<a id="ERR_CANNOT_TRANSFER_OBJECT"></a>

### `ERR_CANNOT_TRANSFER_OBJECT`

<!-- YAML
added: v10.5.0
removed: v12.5.0
-->

传递给 `postMessage()` 的值包含不支持传输的对象。

<a id="ERR_CPU_USAGE"></a>

### `ERR_CPU_USAGE`

<!-- YAML
removed: v15.0.0
-->

来自 `process.cpuUsage` 的本机调用无法处理。

<a id="ERR_CRYPTO_HASH_DIGEST_NO_UTF16"></a>

### `ERR_CRYPTO_HASH_DIGEST_NO_UTF16`

<!-- YAML
added: v9.0.0
removed: v12.12.0
-->

UTF-16 编码与 [`hash.digest()`][] 一起使用。虽然 `hash.digest()` 方法允许传递 `encoding` 参数，导致该方法返回字符串而不是 `Buffer`，但不支持 UTF-16 编码（例如 `ucs` 或 `utf16le`）。

<a id="ERR_CRYPTO_SCRYPT_INVALID_PARAMETER"></a>

### `ERR_CRYPTO_SCRYPT_INVALID_PARAMETER`

<!-- YAML
removed: v23.0.0
-->

向 [`crypto.scrypt()`][] 或 [`crypto.scryptSync()`][] 传递了不兼容的选项组合。新版本的 Node.js 使用错误代码 [`ERR_INCOMPATIBLE_OPTION_PAIR`][] 代替，这与其他 API 一致。

<a id="ERR_FS_INVALID_SYMLINK_TYPE"></a>

### `ERR_FS_INVALID_SYMLINK_TYPE`

<!-- YAML
removed: v23.0.0
-->

向 [`fs.symlink()`][] 或 [`fs.symlinkSync()`][] 方法传递了无效的符号链接类型。

<a id="ERR_HTTP2_FRAME_ERROR"></a>

### `ERR_HTTP2_FRAME_ERROR`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

在 HTTP/2 会话上发送单个帧时发生失败时使用。

<a id="ERR_HTTP2_HEADERS_OBJECT"></a>

### `ERR_HTTP2_HEADERS_OBJECT`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

需要 HTTP/2 头部对象时使用。

<a id="ERR_HTTP2_HEADER_REQUIRED"></a>

### `ERR_HTTP2_HEADER_REQUIRED`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

当 HTTP/2 消息中缺少必需头部时使用。

<a id="ERR_HTTP2_INFO_HEADERS_AFTER_RESPOND"></a>

### `ERR_HTTP2_INFO_HEADERS_AFTER_RESPOND`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

HTTP/2 信息头部必须仅在调用 `Http2Stream.prototype.respond()` 方法 _之前_ 发送。

<a id="ERR_HTTP2_STREAM_CLOSED"></a>

### `ERR_HTTP2_STREAM_CLOSED`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

在已经关闭的 HTTP/2 流上执行操作时使用。

<a id="ERR_HTTP_INVALID_CHAR"></a>

### `ERR_HTTP_INVALID_CHAR`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

在 HTTP 响应状态消息（原因短语）中发现无效字符时使用。

<a id="ERR_IMPORT_ASSERTION_TYPE_FAILED"></a>

### `ERR_IMPORT_ASSERTION_TYPE_FAILED`

<!-- YAML
added:
  - v17.1.0
  - v16.14.0
removed: v21.1.0
-->

导入断言失败，阻止导入指定模块。

<a id="ERR_IMPORT_ASSERTION_TYPE_MISSING"></a>

### `ERR_IMPORT_ASSERTION_TYPE_MISSING`

<!-- YAML
added:
  - v17.1.0
  - v16.14.0
removed: v21.1.0
-->

缺少导入断言，阻止导入指定模块。

<a id="ERR_IMPORT_ASSERTION_TYPE_UNSUPPORTED"></a>

### `ERR_IMPORT_ASSERTION_TYPE_UNSUPPORTED`

<!-- YAML
added:
  - v17.1.0
  - v16.14.0
removed: v21.1.0
-->

此版本的 Node.js 不支持导入属性。

<a id="ERR_INDEX_OUT_OF_RANGE"></a>

### `ERR_INDEX_OUT_OF_RANGE`

<!-- YAML
  added: v10.0.0
  removed: v11.0.0
-->

给定的索引超出可接受范围（例如，负偏移）。

<a id="ERR_INVALID_OPT_VALUE"></a>

### `ERR_INVALID_OPT_VALUE`

<!-- YAML
added: v8.0.0
removed: v15.0.0
-->

在选项对象中传递了无效或意外的值。

<a id="ERR_INVALID_OPT_VALUE_ENCODING"></a>

### `ERR_INVALID_OPT_VALUE_ENCODING`

<!-- YAML
added: v9.0.0
removed: v15.0.0
-->

传递了无效或未知的文件编码。

<a id="ERR_INVALID_PERFORMANCE_MARK"></a>

### `ERR_INVALID_PERFORMANCE_MARK`

<!-- YAML
added: v8.5.0
removed: v16.7.0
-->

在使用性能计时 API (`perf_hooks`) 时，性能标记无效。

<a id="ERR_INVALID_TRANSFER_OBJECT"></a>

### `ERR_INVALID_TRANSFER_OBJECT`

<!-- YAML
removed: v21.0.0
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/47839
    description: A `DOMException` is thrown instead.
-->

向 `postMessage()` 传递了无效的传输对象。

<a id="ERR_MANIFEST_ASSERT_INTEGRITY"></a>

### `ERR_MANIFEST_ASSERT_INTEGRITY`

<!-- YAML
removed: v22.2.0
-->

尝试加载资源，但资源与策略清单定义的完整性不匹配。有关策略清单的更多信息，请参见文档。

<a id="ERR_MANIFEST_DEPENDENCY_MISSING"></a>

### `ERR_MANIFEST_DEPENDENCY_MISSING`

<!-- YAML
removed: v22.2.0
-->

尝试加载资源，但资源未列为尝试加载它的位置的依赖项。有关策略清单的更多信息，请参见文档。

<a id="ERR_MANIFEST_INTEGRITY_MISMATCH"></a>

### `ERR_MANIFEST_INTEGRITY_MISMATCH`

<!-- YAML
removed: v22.2.0
-->

尝试加载策略清单，但清单中资源的多个条目不匹配。更新清单条目以匹配以解决此错误。有关策略清单的更多信息，请参见文档。

<a id="ERR_MANIFEST_INVALID_RESOURCE_FIELD"></a>

### `ERR_MANIFEST_INVALID_RESOURCE_FIELD`

<!-- YAML
removed: v22.2.0
-->

策略清单资源的某个字段具有无效值。更新清单条目以匹配以解决此错误。有关策略清单的更多信息，请参见文档。

<a id="ERR_MANIFEST_INVALID_SPECIFIER"></a>

### `ERR_MANIFEST_INVALID_SPECIFIER`

<!-- YAML
removed: v22.2.0
-->

策略清单资源的某个依赖映射具有无效值。更新清单条目以匹配以解决此错误。有关策略清单的更多信息，请参见文档。

<a id="ERR_MANIFEST_PARSE_POLICY"></a>

### `ERR_MANIFEST_PARSE_POLICY`

<!-- YAML
removed: v22.2.0
-->

尝试加载策略清单，但无法解析清单。有关策略清单的更多信息，请参见文档。

<a id="ERR_MANIFEST_TDZ"></a>

### `ERR_MANIFEST_TDZ`

<!-- YAML
removed: v22.2.0
-->

尝试从策略清单读取，但清单初始化尚未发生。这可能是 Node.js 中的错误。

<a id="ERR_MANIFEST_UNKNOWN_ONERROR"></a>

### `ERR_MANIFEST_UNKNOWN_ONERROR`

<!-- YAML
removed: v22.2.0
-->

加载了策略清单，但其 "onerror" 行为具有未知值。有关策略清单的更多信息，请参见文档。

<a id="ERR_MISSING_MESSAGE_PORT_IN_TRANSFER_LIST"></a>

### `ERR_MISSING_MESSAGE_PORT_IN_TRANSFER_LIST`

<!-- YAML
removed: v15.0.0
-->

此错误代码在 Node.js v15.0.0 中被 [`ERR_MISSING_TRANSFERABLE_IN_TRANSFER_LIST`][] 替换，因为它不再准确，因为现在也存在其他类型的可传输对象。

<a id="ERR_MISSING_TRANSFERABLE_IN_TRANSFER_LIST"></a>

### `ERR_MISSING_TRANSFERABLE_IN_TRANSFER_LIST`

<!-- YAML
added: v15.0.0
removed: v21.0.0
changes:
  - version: v21.0.0
    pr-url: https://github.com/nodejs/node/pull/47839
    description: A `DOMException` is thrown instead.
-->

需要显式列在 `transferList` 参数中的对象位于传递给 [`postMessage()`][] 调用的对象中，但未在该调用的 `transferList` 中提供。通常，这是一个 `MessagePort`。

在 Node.js v15.0.0 之前的版本中，这里使用的错误代码是 [`ERR_MISSING_MESSAGE_PORT_IN_TRANSFER_LIST`][]。但是，可传输对象类型的集合已扩展以覆盖更多类型，而不仅仅是 `MessagePort`。

<a id="ERR_NAPI_CONS_PROTOTYPE_OBJECT"></a>

### `ERR_NAPI_CONS_PROTOTYPE_OBJECT`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

当 `Constructor.prototype` 不是对象时，由 `Node-API` 使用。

<a id="ERR_NAPI_TSFN_START_IDLE_LOOP"></a>

### `ERR_NAPI_TSFN_START_IDLE_LOOP`

<!-- YAML
added:
  - v10.6.0
  - v8.16.0
removed:
  - v14.2.0
  - v12.17.0
-->

在主线程上，值在与线程安全函数关联的队列中的空闲循环中被移除。此错误表示尝试启动循环时发生错误。

<a id="ERR_NAPI_TSFN_STOP_IDLE_LOOP"></a>

### `ERR_NAPI_TSFN_STOP_IDLE_LOOP`

<!-- YAML
added:
  - v10.6.0
  - v8.16.0
removed:
  - v14.2.0
  - v12.17.0
-->

一旦队列中没有更多项，必须暂停空闲循环。此错误表示空闲循环未能停止。

<a id="ERR_NO_LONGER_SUPPORTED"></a>

### `ERR_NO_LONGER_SUPPORTED```

以不支持的方式调用了 Node.js API，例如 `Buffer.write(string, encoding, offset[, length])`。

<a id="ERR_OUTOFMEMORY"></a>

### `ERR_OUTOFMEMORY`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

通常用于标识操作导致内存不足的情况。

<a id="ERR_PARSE_HISTORY_DATA"></a>

### `ERR_PARSE_HISTORY_DATA`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

`node:repl` 模块无法从 REPL 历史文件中解析数据。

<a id="ERR_SOCKET_CANNOT_SEND"></a>

### `ERR_SOCKET_CANNOT_SEND`

<!-- YAML
added: v9.0.0
removed: v14.0.0
-->

无法在套接字上发送数据。

<a id="ERR_STDERR_CLOSE"></a>

### `ERR_STDERR_CLOSE`

<!-- YAML
removed: v10.12.0
changes:
  - version: v10.12.0
    pr-url: https://github.com/nodejs/node/pull/23053
    description: Rather than emitting an error, `process.stderr.end()` now
                 only closes the stream side but not the underlying resource,
                 making this error obsolete.
-->

尝试关闭 `process.stderr` 流。根据设计，Node.js 不允许用户代码关闭 `stdout` 或 `stderr` 流。

<a id="ERR_STDOUT_CLOSE"></a>

### `ERR_STDOUT_CLOSE`

<!-- YAML
removed: v10.12.0
changes:
  - version: v10.12.0
    pr-url: https://github.com/nodejs/node/pull/23053
    description: Rather than emitting an error, `process.stderr.end()` now
                 only closes the stream side but not the underlying resource,
                 making this error obsolete.
-->

尝试关闭 `process.stdout` 流。根据设计，Node.js 不允许用户代码关闭 `stdout` 或 `stderr` 流。

<a id="ERR_STREAM_READ_NOT_IMPLEMENTED"></a>

### `ERR_STREAM_READ_NOT_IMPLEMENTED`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

尝试使用未实现 [`readable._read()`][] 的可读流时使用。

<a id="ERR_TAP_LEXER_ERROR"></a>

### `ERR_TAP_LEXER_ERROR```

表示失败词法分析器状态的错误。

<a id="ERR_TAP_PARSER_ERROR"></a>

### `ERR_TAP_PARSER_ERROR```

表示失败解析器状态的错误。有关导致错误的 token 的更多信息可通过 `cause` 属性获得。

<a id="ERR_TAP_VALIDATION_ERROR"></a>

### `ERR_TAP_VALIDATION_ERROR```

此错误表示 TAP 验证失败。

<a id="ERR_TLS_RENEGOTIATION_FAILED"></a>

### `ERR_TLS_RENEGOTIATION_FAILED`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

当 TLS 重新协商请求以非特定方式失败时使用。

<a id="ERR_TRANSFERRING_EXTERNALIZED_SHAREDARRAYBUFFER"></a>

### `ERR_TRANSFERRING_EXTERNALIZED_SHAREDARRAYBUFFER`

<!-- YAML
added: v10.5.0
removed: v14.0.0
-->

在序列化过程中遇到了内存不由 JavaScript 引擎或 Node.js 管理的 `SharedArrayBuffer`。此类 `SharedArrayBuffer` 无法序列化。

这只能在本机插件在“外部化”模式下创建 `SharedArrayBuffer`，或将现有 `SharedArrayBuffer` 置于外部化模式时发生。

<a id="ERR_UNKNOWN_STDIN_TYPE"></a>

### `ERR_UNKNOWN_STDIN_TYPE`

<!-- YAML
added: v8.0.0
removed: v11.7.0
-->

尝试启动具有未知 `stdin` 文件类型的 Node.js 进程。此错误通常表示 Node.js 本身存在错误，尽管用户代码也可能触发它。

<a id="ERR_UNKNOWN_STREAM_TYPE"></a>

### `ERR_UNKNOWN_STREAM_TYPE`

<!-- YAML
added: v8.0.0
removed: v11.7.0
-->

尝试启动具有未知 `stdout` 或 `stderr` 文件类型的 Node.js 进程。此错误通常表示 Node.js 本身存在错误，尽管用户代码也可能触发它。

<a id="ERR_V8BREAKITERATOR"></a>

### `ERR_V8BREAKITERATOR```

使用了 V8 `BreakIterator` API，但未安装完整的 ICU 数据集。

<a id="ERR_VALUE_OUT_OF_RANGE"></a>

### `ERR_VALUE_OUT_OF_RANGE`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

当给定值超出可接受范围时使用。

<a id="ERR_VM_MODULE_LINKING_ERRORED"></a>

### `ERR_VM_MODULE_LINKING_ERRORED`

<!-- YAML
added: v10.0.0
removed:
  - v18.1.0
  - v16.17.0
-->

链接器函数返回了一个链接失败的模块。

<a id="ERR_VM_MODULE_NOT_LINKED"></a>

### `ERR_VM_MODULE_NOT_LINKED```

在实例化之前必须成功链接模块。

<a id="ERR_WORKER_UNSUPPORTED_EXTENSION"></a>

### `ERR_WORKER_UNSUPPORTED_EXTENSION`

<!-- YAML
added: v11.0.0
removed: v16.9.0
-->

工作器主脚本的路径名具有未知的文件扩展名。

<a id="ERR_ZLIB_BINDING_CLOSED"></a>

### `ERR_ZLIB_BINDING_CLOSED`

<!-- YAML
added: v9.0.0
removed: v10.0.0
-->

尝试在 `zlib` 对象已经关闭后使用它时使用。

<a id="openssl-error-codes"></a>

## OpenSSL 错误代码

<a id="Time Validity Errors"></a>

### 时间有效性错误

<a id="CERT_NOT_YET_VALID"></a>

#### `CERT_NOT_YET_VALID`

证书尚未生效：notBefore 日期在当前时间之后。

<a id="CERT_HAS_EXPIRED"></a>

#### `CERT_HAS_EXPIRED`

证书已过期：notAfter 日期在当前时间之前。

<a id="CRL_NOT_YET_VALID"></a>

#### `CRL_NOT_YET_VALID`

证书吊销列表 (CRL) 具有未来的发布日期。

<a id="CRL_HAS_EXPIRED"></a>

#### `CRL_HAS_EXPIRED`

证书吊销列表 (CRL) 已过期。

<a id="CERT_REVOKED"></a>

#### `CERT_REVOKED`

证书已被吊销；它在证书吊销列表 (CRL) 上。

<a id="Trust or Chain Related Errors"></a>

### 信任或链相关错误

<a id="UNABLE_TO_GET_ISSUER_CERT"></a>

#### `UNABLE_TO_GET_ISSUER_CERT`

查找证书的颁发者证书找不到。这通常意味着受信任证书列表不完整。

<a id="UNABLE_TO_GET_ISSUER_CERT_LOCALLY"></a>

#### `UNABLE_TO_GET_ISSUER_CERT_LOCALLY`

证书的颁发者未知。如果颁发者未包含在受信任证书列表中，则会出现这种情况。

<a id="DEPTH_ZERO_SELF_SIGNED_CERT"></a>

#### `DEPTH_ZERO_SELF_SIGNED_CERT`

传递的证书是自签名的，并且在受信任证书列表中找不到相同的证书。

<a id="SELF_SIGNED_CERT_IN_CHAIN"></a>

#### `SELF_SIGNED_CERT_IN_CHAIN`

证书的颁发者未知。如果颁发者未包含在受信任证书列表中，则会出现这种情况。

<a id="CERT_CHAIN_TOO_LONG"></a>

#### `CERT_CHAIN_TOO_LONG`

证书链长度大于最大深度。

<a id="UNABLE_TO_GET_CRL"></a>

#### `UNABLE_TO_GET_CRL`

证书引用的 CRL 找不到。

<a id="UNABLE_TO_VERIFY_LEAF_SIGNATURE"></a>

#### `UNABLE_TO_VERIFY_LEAF_SIGNATURE`

无法验证任何签名，因为链中只包含一个证书且它不是自签名的。

<a id="CERT_UNTRUSTED"></a>

#### `CERT_UNTRUSTED`

根证书颁发机构 (CA) 未标记为受信任用于指定目的。

<a id="Basic Extension Errors"></a>

### 基本扩展错误

<a id="INVALID_CA"></a>

#### `INVALID_CA`

CA 证书无效。要么它不是 CA，要么其扩展与提供的目的不一致。

<a id="PATH_LENGTH_EXCEEDED"></a>

#### `PATH_LENGTH_EXCEEDED`

已超过 basicConstraints pathlength 参数。

<a id="Name Related Errors"></a>

### 名称相关错误

<a id="HOSTNAME_MISMATCH"></a>

#### `HOSTNAME_MISMATCH`

证书与提供的名称不匹配。

<a id="Usage and Policy Errors"></a>

### 使用和策略错误

<a id="INVALID_PURPOSE"></a>

#### `INVALID_PURPOSE`

提供的证书不能用于指定目的。

<a id="CERT_REJECTED"></a>

#### `CERT_REJECTED`

根 CA 被标记为拒绝指定目的。

<a id="Formatting Errors"></a>

### 格式化错误

<a id="CERT_SIGNATURE_FAILURE"></a>

#### `CERT_SIGNATURE_FAILURE`

证书的签名无效。

<a id="CRL_SIGNATURE_FAILURE"></a>

#### `CRL_SIGNATURE_FAILURE`

证书吊销列表 (CRL) 的签名无效。

<a id="ERROR_IN_CERT_NOT_BEFORE_FIELD"></a>

#### `ERROR_IN_CERT_NOT_BEFORE_FIELD`

证书 notBefore 字段包含无效时间。

<a id="ERROR_IN_CERT_NOT_AFTER_FIELD"></a>

#### `ERROR_IN_CERT_NOT_AFTER_FIELD`

证书 notAfter 字段包含无效时间。

<a id="ERROR_IN_CRL_LAST_UPDATE_FIELD"></a>

#### `ERROR_IN_CRL_LAST_UPDATE_FIELD`

CRL lastUpdate 字段包含无效时间。

<a id="ERROR_IN_CRL_NEXT_UPDATE_FIELD"></a>

#### `ERROR_IN_CRL_NEXT_UPDATE_FIELD`

CRL nextUpdate 字段包含无效时间。

<a id="UNABLE_TO_DECRYPT_CERT_SIGNATURE"></a>

#### `UNABLE_TO_DECRYPT_CERT_SIGNATURE`

证书签名无法解密。这意味着无法确定实际签名值，而不是它与期望值不匹配，这仅对 RSA 密钥有意义。

<a id="UNABLE_TO_DECRYPT_CRL_SIGNATURE"></a>

#### `UNABLE_TO_DECRYPT_CRL_SIGNATURE`

证书吊销列表 (CRL) 签名无法解密：这意味着无法确定实际签名值，而不是它与期望值不匹配。

<a id="UNABLE_TO_DECODE_ISSUER_PUBLIC_KEY"></a>

#### `UNABLE_TO_DECODE_ISSUER_PUBLIC_KEY`

无法读取证书 SubjectPublicKeyInfo 中的公钥。

<a id="Other OpenSSL Errors"></a>

### 其他 OpenSSL 错误

<a id="OUT_OF_MEM"></a>

#### `OUT_OF_MEM`

尝试分配内存时发生错误。这不应发生。

[ES Module]: esm.md
[ICU]: intl.md#internationalization-support
[JSON Web Key Elliptic Curve Registry]: https://www.iana.org/assignments/jose/jose.xhtml#web-key-elliptic-curve
[JSON Web Key Types Registry]: https://www.iana.org/assignments/jose/jose.xhtml#web-key-types
[Native TypeScript support]: typescript.md#type-stripping
[Node.js error codes]: #nodejs-error-codes
[Permission Model]: permissions.md#permission-model
[RFC 7230 Section 3]: https://tools.ietf.org/html/rfc7230#section-3
[SQLite]: sqlite.md
[Subresource Integrity specification]: https://www.w3.org/TR/SRI/#the-integrity-attribute
[V8's stack trace API]: https://v8.dev/docs/stack-trace-api
[WHATWG Supported Encodings]: util.md#whatwg-supported-encodings
[WHATWG URL API]: url.md#the-whatwg-url-api
[`"exports"`]: packages.md#exports
[`"imports"`]: packages.md#imports
[`'uncaughtException'`]: process.md#event-uncaughtexception
[`--disable-proto=throw`]: cli.md#--disable-protomode
[`--force-fips`]: cli.md#--force-fips
[`--no-addons`]: cli.md#--no-addons
[`--unhandled-rejections`]: cli.md#--unhandled-rejectionsmode
[`Class: assert.AssertionError`]: assert.md#class-assertassertionerror
[`ERR_INCOMPATIBLE_OPTION_PAIR`]: #err_incompatible_option_pair
[`ERR_INVALID_ARG_TYPE`]: #err_invalid_arg_type
[`ERR_MISSING_MESSAGE_PORT_IN_TRANSFER_LIST`]: #err_missing_message_port_in_transfer_list
[`ERR_MISSING_TRANSFERABLE_IN_TRANSFER_LIST`]: #err_missing_transferable_in_transfer_list
[`ERR_REQUIRE_ASYNC_MODULE`]: #err_require_async_module
[`EventEmitter`]: events.md#class-eventemitter
[`MessagePort`]: worker_threads.md#class-messageport
[`Object.getPrototypeOf`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/getPrototypeOf
[`Object.setPrototypeOf`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/setPrototypeOf
[`REPL`]: repl.md
[`ServerResponse`]: http.md#class-httpserverresponse
[`Writable`]: stream.md#class-streamwritable
[`child_process`]: child_process.md
[`cipher.getAuthTag()`]: crypto.md#ciphergetauthtag
[`crypto.getDiffieHellman()`]: crypto.md#cryptogetdiffiehellmangroupname
[`crypto.scrypt()`]: crypto.md#cryptoscryptpassword-salt-keylen-options-callback
[`crypto.scryptSync()`]: crypto.md#cryptoscryptsyncpassword-salt-keylen-options
[`crypto.timingSafeEqual()`]: crypto.md#cryptotimingsafeequala-b
[`dgram.connect()`]: dgram.md#socketconnectport-address-callback
[`dgram.createSocket()`]: dgram.md#dgramcreatesocketoptions-callback
[`dgram.disconnect()`]: dgram.md#socketdisconnect
[`dgram.remoteAddress()`]: dgram.md#socketremoteaddress
[`domException.name`]: https://developer.mozilla.org/en-US/docs/Web/API/DOMException/name
[`errno`(3) man page]: https://man7.org/linux/man-pages/man3/errno.3.html
[`error.code`]: #errorcode
[`error.message`]: #errormessage
[`fs.Dir`]: fs.md#class-fsdir
[`fs.cp()`]: fs.md#fscpsrc-dest-options-callback
[`fs.readFileSync`]: fs.md#fsreadfilesyncpath-options
[`fs.readdir`]: fs.md#fsreaddirpath-options-callback
[`fs.symlink()`]: fs.md#fssymlinktarget-path-type-callback
[`fs.symlinkSync()`]: fs.md#fssymlinksynctarget-path-type
[`fs.unlink`]: fs.md#fsunlinkpath-callback
[`fs`]: fs.md
[`hash.digest()`]: crypto.md#hashdigestencoding
[`hash.update()`]: crypto.md#hashupdatedata-inputencoding
[`http`]: http.md
[`https`]: https.md
[`libuv Error handling`]: https://docs.libuv.org/en/v1.x/errors.html
[`net.Socket.write()`]: net.md#socketwritedata-encoding-callback
[`net`]: net.md
[`new URL(input)`]: url.md#new-urlinput-base
[`new URLPattern(input)`]: url.md#new-urlpatternstring-baseurl-options
[`new URLSearchParams(iterable)`]: url.md#new-urlsearchparamsiterable
[`package.json`]: packages.md#nodejs-packagejson-field-definitions
[`postMessage()`]: worker_threads.md#portpostmessagevalue-transferlist
[`postMessageToThread()`]: worker_threads.md#workerpostmessagetothreadthreadid-value-transferlist-timeout
[`process.on('exit')`]: process.md#event-exit
[`process.send()`]: process.md#processsendmessage-sendhandle-options-callback
[`process.setUncaughtExceptionCaptureCallback()`]: process.md#processsetuncaughtexceptioncapturecallbackfn
[`readable._read()`]: stream.md#readable_readsize
[`require('node:crypto').setEngine()`]: crypto.md#cryptosetengineengine-flags
[`require()`]: modules.md#requireid
[`server.close()`]: net.md#serverclosecallback
[`server.listen()`]: net.md#serverlisten
[`sign.sign()`]: crypto.md#signsignprivatekey-outputencoding
[`stream.pipe()`]: stream.md#readablepipedestination-options
[`stream.push()`]: stream.md#readablepushchunk-encoding
[`stream.unshift()`]: stream.md#readableunshiftchunk-encoding
[`stream.write()`]: stream.md#writablewritechunk-encoding-callback
[`subprocess.kill()`]: child_process.md#subprocesskillsignal
[`subprocess.send()`]: child_process.md#subprocesssendmessage-sendhandle-options-callback
[`url.parse()`]: url.md#urlparseurlstring-parsequerystring-slashesdenotehost
[`util.getSystemErrorName(error.errno)`]: util.md#utilgetsystemerrornameerr
[`util.inspect()`]: util.md#utilinspectobject-options
[`util.parseArgs()`]: util.md#utilparseargsconfig
[`v8.startupSnapshot.setDeserializeMainFunction()`]: v8.md#v8startupsnapshotsetdeserializemainfunctioncallback-data
[`zlib`]: zlib.md
[crypto digest algorithm]: crypto.md#cryptogethashes
[debugger]: debugger.md
[define a custom subpath]: packages.md#subpath-exports
[domains]: domain.md
[event emitter-based]: events.md#class-eventemitter
[file descriptors]: https://en.wikipedia.org/wiki/File_descriptor
[relative URL]: https://url.spec.whatwg.org/#relative-url-string
[self-reference a package using its name]: packages.md#self-referencing-a-package-using-its-name
[special scheme]: https://url.spec.whatwg.org/#special-scheme
[stream-based]: stream.md
[syscall]: https://man7.org/linux/man-pages/man2/syscalls.2.html
[try-catch]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/try...catch
[type-stripping]: typescript.md#type-stripping
[vm]: vm.md
