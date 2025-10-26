[文件名称]: util.md

[文件内容开始]

# Util

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/util.js -->

`node:util` 模块支持 Node.js 内部 API 的需求。许多实用工具对应用程序和模块开发者也很有用。要访问它：

```mjs
import util from 'node:util';
```

```cjs
const util = require('node:util');
```

## `util.callbackify(original)`

<!-- YAML
added: v8.2.0
-->

- `original` {Function} 一个 `async` 函数
- 返回: {Function} 一个遵循错误优先回调风格的函数

接受一个 `async` 函数（或返回 `Promise` 的函数），并返回一个遵循错误优先回调风格的函数，即将 `(err, value) => ...` 回调作为最后一个参数。在回调中，第一个参数将是拒绝原因（如果 `Promise` 解决则为 `null`），第二个参数将是解决的值。

```mjs
import { callbackify } from 'node:util';

async function fn() {
  return 'hello world';
}
const callbackFunction = callbackify(fn);

callbackFunction((err, ret) => {
  if (err) throw err;
  console.log(ret);
});
```

```cjs
const { callbackify } = require('node:util');

async function fn() {
  return 'hello world';
}
const callbackFunction = callbackify(fn);

callbackFunction((err, ret) => {
  if (err) throw err;
  console.log(ret);
});
```

将打印：

```text
hello world
```

回调是异步执行的，并且将具有有限的堆栈跟踪。如果回调抛出异常，进程将发出 [`'uncaughtException'`][] 事件，如果未处理则退出。

由于 `null` 作为回调的第一个参数具有特殊含义，如果包装函数拒绝一个 `Promise` 且原因是一个假值，则该值将被包装在一个 `Error` 中，原始值存储在一个名为 `reason` 的字段中。

```js
function fn() {
  return Promise.reject(null);
}
const callbackFunction = util.callbackify(fn);

callbackFunction((err, ret) => {
  // 当 Promise 被 `null` 拒绝时，它会被包装在一个 Error 中，并且原始值存储在 `reason` 中。
  err && Object.hasOwn(err, 'reason') && err.reason === null; // true
});
```

## `util.debuglog(section[, callback])`

<!-- YAML
added: v0.11.3
-->

- `section` {string} 一个标识应用程序部分的字符串，正在为其创建 `debuglog` 函数。
- `callback` {Function} 第一次调用日志函数时调用的回调，带有一个函数参数，该参数是一个更优化的日志函数。
- 返回: {Function} 日志函数

`util.debuglog()` 方法用于创建一个函数，根据 `NODE_DEBUG` 环境变量的存在情况，有条件地将调试消息写入 `stderr`。如果 `section` 名称出现在该环境变量的值中，则返回的函数操作类似于 [`console.error()`][]。否则，返回的函数是一个空操作。

```mjs
import { debuglog } from 'node:util';
const log = debuglog('foo');

log('hello from foo [%d]', 123);
```

```cjs
const { debuglog } = require('node:util');
const log = debuglog('foo');

log('hello from foo [%d]', 123);
```

如果在环境中运行此程序时设置了 `NODE_DEBUG=foo`，则它将输出类似以下内容：

```console
FOO 3245: hello from foo [123]
```

其中 `3245` 是进程 ID。如果没有设置该环境变量运行，则它不会打印任何内容。

`section` 也支持通配符：

```mjs
import { debuglog } from 'node:util';
const log = debuglog('foo');

log("hi there, it's foo-bar [%d]", 2333);
```

```cjs
const { debuglog } = require('node:util');
const log = debuglog('foo');

log("hi there, it's foo-bar [%d]", 2333);
```

如果在环境中使用 `NODE_DEBUG=foo*` 运行，则它将输出类似以下内容：

```console
FOO-BAR 3257: hi there, it's foo-bar [2333]
```

可以在 `NODE_DEBUG` 环境变量中指定多个逗号分隔的 `section` 名称：`NODE_DEBUG=fs,net,tls`。

可选的 `callback` 参数可用于将日志函数替换为另一个没有任何初始化或不必要包装的函数。

```mjs
import { debuglog } from 'node:util';
let log = debuglog('internals', (debug) => {
  // 替换为一个优化掉检查该部分是否启用的日志函数
  log = debug;
});
```

```cjs
const { debuglog } = require('node:util');
let log = debuglog('internals', (debug) => {
  // 替换为一个优化掉检查该部分是否启用的日志函数
  log = debug;
});
```

### `debuglog().enabled`

<!-- YAML
added: v14.9.0
-->

- 类型: {boolean}

`util.debuglog().enabled` getter 用于创建一个测试，该测试可用于基于 `NODE_DEBUG` 环境变量存在性的条件判断。如果 `section` 名称出现在该环境变量的值中，则返回的值为 `true`。否则，返回的值为 `false`。

```mjs
import { debuglog } from 'node:util';
const enabled = debuglog('foo').enabled;
if (enabled) {
  console.log('hello from foo [%d]', 123);
}
```

```cjs
const { debuglog } = require('node:util');
const enabled = debuglog('foo').enabled;
if (enabled) {
  console.log('hello from foo [%d]', 123);
}
```

如果在环境中运行此程序时设置了 `NODE_DEBUG=foo`，则它将输出类似以下内容：

```console
hello from foo [123]
```

## `util.debug(section)`

<!-- YAML
added: v14.9.0
-->

`util.debuglog` 的别名。用法允许在仅使用 `util.debuglog().enabled` 时提高可读性，而不暗示正在记录日志。

## `util.deprecate(fn, msg[, code])`

<!-- YAML
added: v0.8.0
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/16393
    description: Deprecation warnings are only emitted once for each code.
-->

- `fn` {Function} 被弃用的函数。
- `msg` {string} 调用已弃用函数时显示的警告消息。
- `code` {string} 弃用代码。有关代码列表，请参阅 [已弃用 API 列表][]。
- 返回: {Function} 包装后的已弃用函数，会发出警告。

`util.deprecate()` 方法包装 `fn`（可以是函数或类），使其被标记为已弃用。

```mjs
import { deprecate } from 'node:util';

export const obsoleteFunction = deprecate(() => {
  // 在这里做一些事情。
}, 'obsoleteFunction() is deprecated. Use newShinyFunction() instead.');
```

```cjs
const { deprecate } = require('node:util');

exports.obsoleteFunction = deprecate(() => {
  // 在这里做一些事情。
}, 'obsoleteFunction() is deprecated. Use newShinyFunction() instead.');
```

调用时，`util.deprecate()` 将返回一个函数，该函数将使用 [`'warning'`][] 事件发出 `DeprecationWarning`。警告将在第一次调用返回的函数时发出并打印到 `stderr`。警告发出后，将调用包装函数而不发出警告。

如果在多次调用 `util.deprecate()` 时提供了相同的可选 `code`，则对于该 `code` 只会发出一次警告。

```mjs
import { deprecate } from 'node:util';

const fn1 = deprecate(() => 'a value', 'deprecation message', 'DEP0001');
const fn2 = deprecate(
  () => 'a  different value',
  'other dep message',
  'DEP0001'
);
fn1(); // 发出带有代码 DEP0001 的弃用警告
fn2(); // 不发出弃用警告，因为它具有相同的代码
```

```cjs
const { deprecate } = require('node:util');

const fn1 = deprecate(
  function () {
    return 'a value';
  },
  'deprecation message',
  'DEP0001'
);
const fn2 = deprecate(
  function () {
    return 'a  different value';
  },
  'other dep message',
  'DEP0001'
);
fn1(); // 发出带有代码 DEP0001 的弃用警告
fn2(); // 不发出弃用警告，因为它具有相同的代码
```

如果使用了 `--no-deprecation` 或 `--no-warnings` 命令行标志，或者在第一次弃用警告 _之前_ 将 `process.noDeprecation` 属性设置为 `true`，则 `util.deprecate()` 方法不执行任何操作。

如果设置了 `--trace-deprecation` 或 `--trace-warnings` 命令行标志，或者将 `process.traceDeprecation` 属性设置为 `true`，则在第一次调用已弃用函数时，会将警告和堆栈跟踪打印到 `stderr`。

如果设置了 `--throw-deprecation` 命令行标志，或者将 `process.throwDeprecation` 属性设置为 `true`，则在调用已弃用函数时将抛出异常。

`--throw-deprecation` 命令行标志和 `process.throwDeprecation` 属性优先于 `--trace-deprecation` 和 `process.traceDeprecation`。

## `util.diff(actual, expected)`

<!-- YAML
added:
  - v23.11.0
  - v22.15.0
-->

> Stability: 1 - Experimental

- `actual` {Array|string} 要比较的第一个值
- `expected` {Array|string} 要比较的第二个值
- 返回: {Array} 一个差异条目数组。每个条目是一个包含两个元素的数组：

  - `0` {number} 操作码：`-1` 表示删除，`0` 表示无操作/未更改，`1` 表示插入
  - `1` {string} 与操作关联的值

- 算法复杂度：O(N\*D)，其中：

- N 是两个序列的总长度（N = actual.length + expected.length）

- D 是编辑距离（将一个序列转换为另一个序列所需的最少操作次数）。

[`util.diff()`][] 比较两个字符串或数组值，并返回一个差异条目数组。
它使用 Myers diff 算法来计算最小差异，该算法与断言错误消息内部使用的算法相同。

如果值相等，则返回一个空数组。

```js
const { diff } = require('node:util');

// 比较字符串
const actualString = '12345678';
const expectedString = '12!!5!7!';
console.log(diff(actualString, expectedString));
// [
//   [0, '1'],
//   [0, '2'],
//   [1, '3'],
//   [1, '4'],
//   [-1, '!'],
//   [-1, '!'],
//   [0, '5'],
//   [1, '6'],
//   [-1, '!'],
//   [0, '7'],
//   [1, '8'],
//   [-1, '!'],
// ]
// 比较数组
const actualArray = ['1', '2', '3'];
const expectedArray = ['1', '3', '4'];
console.log(diff(actualArray, expectedArray));
// [
//   [0, '1'],
//   [1, '2'],
//   [0, '3'],
//   [-1, '4'],
// ]
// 相等的值返回空数组
console.log(diff('same', 'same'));
// []
```

## `util.format(format[, ...args])`

<!-- YAML
added: v0.5.3
changes:
  - version: v12.11.0
    pr-url: https://github.com/nodejs/node/pull/29606
    description: The `%c` specifier is ignored now.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/23162
    description: The `format` argument is now only taken as such if it actually
                 contains format specifiers.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/23162
    description: If the `format` argument is not a format string, the output
                 string's formatting is no longer dependent on the type of the
                 first argument. This change removes previously present quotes
                 from strings that were being output when the first argument
                 was not a string.
  - version: v11.4.0
    pr-url: https://github.com/nodejs/node/pull/23708
    description: The `%d`, `%f`, and `%i` specifiers now support Symbols
                 properly.
  - version: v11.4.0
    pr-url: https://github.com/nodejs/node/pull/24806
    description: The `%o` specifier's `depth` has default depth of 4 again.
  - version: v11.0.0
    pr-url: https://github.com/nodejs/node/pull/17907
    description: The `%o` specifier's `depth` option will now fall back to the
                 default depth.
  - version: v10.12.0
    pr-url: https://github.com/nodejs/node/pull/22097
    description: The `%d` and `%i` specifiers now support BigInt.
  - version: v8.4.0
    pr-url: https://github.com/nodejs/node/pull/14558
    description: The `%o` and `%O` specifiers are supported now.
-->

- `format` {string} 一个类似 `printf` 的格式字符串。

`util.format()` 方法使用第一个参数作为类似 `printf` 的格式字符串（可以包含零个或多个格式说明符）返回一个格式化字符串。每个说明符都会被对应参数转换后的值替换。支持的说明符有：

- `%s`：`String` 将用于转换除 `BigInt`、`Object` 和 `-0` 之外的所有值。`BigInt` 值将用 `n` 表示，而既没有用户定义 `toString` 函数也没有 `Symbol.toPrimitive` 函数的对象将使用 `util.inspect()` 和选项 `{ depth: 0, colors: false, compact: 3 }` 进行检查。
- `%d`：`Number` 将用于转换除 `BigInt` 和 `Symbol` 之外的所有值。
- `%i`：`parseInt(value, 10)` 用于除 `BigInt` 和 `Symbol` 之外的所有值。
- `%f`：`parseFloat(value)` 用于除 `Symbol` 之外的所有值。
- `%j`：JSON。如果参数包含循环引用，则替换为字符串 `'[Circular]'`。
- `%o`：`Object`。具有通用 JavaScript 对象格式的对象字符串表示形式。类似于使用选项 `{ showHidden: true, showProxy: true }` 的 `util.inspect()`。这将显示包括不可枚举属性和代理在内的完整对象。
- `%O`：`Object`。具有通用 JavaScript 对象格式的对象字符串表示形式。类似于不使用选项的 `util.inspect()`。这将显示不包括不可枚举属性和代理的完整对象。
- `%c`：`CSS`。此说明符被忽略，并将跳过任何传入的 CSS。
- `%%`：单个百分号（`'%'`）。这不消耗参数。
- 返回: {string} 格式化后的字符串

如果说明符没有对应的参数，则不会被替换：

```js
util.format('%s:%s', 'foo');
// 返回: 'foo:%s'
```

不属于格式字符串的值如果其类型不是 `string`，则使用 `util.inspect()` 进行格式化。

如果传递给 `util.format()` 方法的参数数量多于说明符的数量，则额外的参数将连接到返回的字符串，以空格分隔：

```js
util.format('%s:%s', 'foo', 'bar', 'baz');
// 返回: 'foo:bar baz'
```

如果第一个参数不包含有效的格式说明符，则 `util.format()` 返回一个所有参数以空格分隔连接而成的字符串：

```js
util.format(1, 2, 3);
// 返回: '1 2 3'
```

如果只向 `util.format()` 传递一个参数，则它按原样返回，不进行任何格式化：

```js
util.format('%% %s');
// 返回: '%% %s'
```

`util.format()` 是一个同步方法，旨在用作调试工具。某些输入值可能会产生显著的性能开销，从而阻塞事件循环。请谨慎使用此函数，切勿在热点代码路径中使用。

## `util.formatWithOptions(inspectOptions, format[, ...args])`

<!-- YAML
added: v10.0.0
-->

- `inspectOptions` {Object}
- `format` {string}

此函数与 [`util.format()`][] 相同，只是它接受一个 `inspectOptions` 参数，该参数指定传递给 [`util.inspect()`][] 的选项。

```js
util.formatWithOptions({ colors: true }, 'See object %O', { foo: 42 });
// 返回 'See object { foo: 42 }'，其中 `42` 在打印到终端时被着色为数字。
```

## `util.getCallSites([frameCount][, options])`

<!-- YAML
added: v22.9.0
changes:
  - version:
    - v23.7.0
    - v22.14.0
    pr-url: https://github.com/nodejs/node/pull/56584
    description: Property `column` is deprecated in favor of `columnNumber`.
  - version:
    - v23.7.0
    - v22.14.0
    pr-url: https://github.com/nodejs/node/pull/56551
    description: Property `CallSite.scriptId` is exposed.
  - version:
    - v23.3.0
    - v22.12.0
    pr-url: https://github.com/nodejs/node/pull/55626
    description: The API is renamed from `util.getCallSite` to `util.getCallSites()`.
-->

> Stability: 1.1 - Active development

- `frameCount` {number} 要捕获为调用站点对象的帧数。
  **默认值:** `10`。允许范围在 1 到 200 之间。
- `options` {Object} 可选
  - `sourceMap` {boolean} 从源映射中重建堆栈跟踪中的原始位置。
    使用 `--enable-source-maps` 标志时默认启用。
- 返回: {Object\[]} 调用站点对象的数组
  - `functionName` {string} 返回与此调用站点关联的函数名称。
  - `scriptName` {string} 返回包含此调用站点函数脚本的资源名称。
  - `scriptId` {string} 返回脚本的唯一 ID，如 Chrome DevTools 协议 [`Runtime.ScriptId`][] 中所示。
  - `lineNumber` {number} 返回 JavaScript 脚本行号（从 1 开始）。
  - `columnNumber` {number} 返回 JavaScript 脚本列号（从 1 开始）。

返回一个包含调用者函数堆栈的调用站点对象数组。

```mjs
import { getCallSites } from 'node:util';

function exampleFunction() {
  const callSites = getCallSites();

  console.log('Call Sites:');
  callSites.forEach((callSite, index) => {
    console.log(`CallSite ${index + 1}:`);
    console.log(`Function Name: ${callSite.functionName}`);
    console.log(`Script Name: ${callSite.scriptName}`);
    console.log(`Line Number: ${callSite.lineNumber}`);
    console.log(`Column Number: ${callSite.column}`);
  });
  // CallSite 1:
  // Function Name: exampleFunction
  // Script Name: /home/example.js
  // Line Number: 5
  // Column Number: 26

  // CallSite 2:
  // Function Name: anotherFunction
  // Script Name: /home/example.js
  // Line Number: 22
  // Column Number: 3

  // ...
}

// 模拟另一个堆栈层的函数
function anotherFunction() {
  exampleFunction();
}

anotherFunction();
```

```cjs
const { getCallSites } = require('node:util');

function exampleFunction() {
  const callSites = getCallSites();

  console.log('Call Sites:');
  callSites.forEach((callSite, index) => {
    console.log(`CallSite ${index + 1}:`);
    console.log(`Function Name: ${callSite.functionName}`);
    console.log(`Script Name: ${callSite.scriptName}`);
    console.log(`Line Number: ${callSite.lineNumber}`);
    console.log(`Column Number: ${callSite.column}`);
  });
  // CallSite 1:
  // Function Name: exampleFunction
  // Script Name: /home/example.js
  // Line Number: 5
  // Column Number: 26

  // CallSite 2:
  // Function Name: anotherFunction
  // Script Name: /home/example.js
  // Line Number: 22
  // Column Number: 3

  // ...
}

// 模拟另一个堆栈层的函数
function anotherFunction() {
  exampleFunction();
}

anotherFunction();
```

可以通过将选项 `sourceMap` 设置为 `true` 来重建原始位置。
如果源映射不可用，原始位置将与当前位置相同。
当启用 `--enable-source-maps` 标志时，例如在使用 `--experimental-transform-types` 时，`sourceMap` 将默认为 true。

```ts
import { getCallSites } from 'node:util';

interface Foo {
  foo: string;
}

const callSites = getCallSites({ sourceMap: true });

// 使用 sourceMap：
// Function Name: ''
// Script Name: example.js
// Line Number: 7
// Column Number: 26

// 不使用 sourceMap：
// Function Name: ''
// Script Name: example.js
// Line Number: 2
// Column Number: 26
```

```cjs
const { getCallSites } = require('node:util');

const callSites = getCallSites({ sourceMap: true });

// 使用 sourceMap：
// Function Name: ''
// Script Name: example.js
// Line Number: 7
// Column Number: 26

// 不使用 sourceMap：
// Function Name: ''
// Script Name: example.js
// Line Number: 2
// Column Number: 26
```

## `util.getSystemErrorName(err)`

<!-- YAML
added: v9.7.0
-->

- `err` {number}
- 返回: {string}

返回来自 Node.js API 的数字错误代码对应的字符串名称。错误代码和错误名称之间的映射取决于平台。
有关常见错误的名称，请参阅 [常见系统错误][]。

```js
fs.access('file/that/does/not/exist', (err) => {
  const name = util.getSystemErrorName(err.errno);
  console.error(name); // ENOENT
});
```

## `util.getSystemErrorMap()`

<!-- YAML
added:
  - v16.0.0
  - v14.17.0
-->

- 返回: {Map}

返回一个 Map，包含来自 Node.js API 的所有可用系统错误代码。
错误代码和错误名称之间的映射取决于平台。
有关常见错误的名称，请参阅 [常见系统错误][]。

```js
fs.access('file/that/does/not/exist', (err) => {
  const errorMap = util.getSystemErrorMap();
  const name = errorMap.get(err.errno);
  console.error(name); // ENOENT
});
```

## `util.getSystemErrorMessage(err)`

<!-- YAML
added:
  - v23.1.0
  - v22.12.0
-->

- `err` {number}
- 返回: {string}

返回来自 Node.js API 的数字错误代码对应的字符串消息。
错误代码和字符串消息之间的映射取决于平台。

```js
fs.access('file/that/does/not/exist', (err) => {
  const message = util.getSystemErrorMessage(err.errno);
  console.error(message); // No such file or directory
});
```

## `util.setTraceSigInt(enable)`

<!-- YAML
added: v24.6.0
-->

- `enable` {boolean}

启用或禁用打印 `SIGINT` 的堆栈跟踪。该 API 仅在主线程上可用。

## `util.inherits(constructor, superConstructor)`

<!-- YAML
added: v0.3.0
changes:
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/3455
    description: The `constructor` parameter can refer to an ES6 class now.
-->

> Stability: 3 - Legacy: 使用 ES2015 class 语法和 `extends` 关键字代替。

- `constructor` {Function}
- `superConstructor` {Function}

不鼓励使用 `util.inherits()`。请使用 ES6 `class` 和 `extends` 关键字来获得语言级别的继承支持。另请注意，这两种风格是 [语义不兼容的][]。

将一个 [构造函数][] 的原型方法继承到另一个构造函数。`constructor` 的原型将被设置为从 `superConstructor` 创建的新对象。

这主要是在 `Object.setPrototypeOf(constructor.prototype, superConstructor.prototype)` 的基础上增加了一些输入验证。
作为额外的便利，`superConstructor` 可以通过 `constructor.super_` 属性访问。

```js
const util = require('node:util');
const EventEmitter = require('node:events');

function MyStream() {
  EventEmitter.call(this);
}

util.inherits(MyStream, EventEmitter);

MyStream.prototype.write = function (data) {
  this.emit('data', data);
};

const stream = new MyStream();

console.log(stream instanceof EventEmitter); // true
console.log(MyStream.super_ === EventEmitter); // true

stream.on('data', (data) => {
  console.log(`Received data: "${data}"`);
});
stream.write('It works!'); // Received data: "It works!"
```

使用 `class` 和 `extends` 的 ES6 示例：

```mjs
import EventEmitter from 'node:events';

class MyStream extends EventEmitter {
  write(data) {
    this.emit('data', data);
  }
}

const stream = new MyStream();

stream.on('data', (data) => {
  console.log(`Received data: "${data}"`);
});
stream.write('With ES6');
```

```cjs
const EventEmitter = require('node:events');

class MyStream extends EventEmitter {
  write(data) {
    this.emit('data', data);
  }
}

const stream = new MyStream();

stream.on('data', (data) => {
  console.log(`Received data: "${data}"`);
});
stream.write('With ES6');
```

## `util.inspect(object[, options])`

## `util.inspect(object[, showHidden[, depth[, colors]]])`

<!-- YAML
added: v0.3.0
changes:
  - version:
    - v17.3.0
    - v16.14.0
    pr-url: https://github.com/nodejs/node/pull/41003
    description: The `numericSeparator` option is supported now.
  - version: v16.18.0
    pr-url: https://github.com/nodejs/node/pull/43576
    description: add support for `maxArrayLength` when inspecting `Set` and `Map`.
  - version:
    - v14.6.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/33690
    description: If `object` is from a different `vm.Context` now, a custom
                 inspection function on it will not receive context-specific
                 arguments anymore.
  - version:
     - v13.13.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32392
    description: The `maxStringLength` option is supported now.
  - version:
     - v13.5.0
     - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30768
    description: User defined prototype properties are inspected in case
                 `showHidden` is `true`.
  - version: v13.0.0
    pr-url: https://github.com/nodejs/node/pull/27685
    description: Circular references now include a marker to the reference.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/27109
    description: The `compact` options default is changed to `3` and the
                 `breakLength` options default is changed to `80`.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/24971
    description: Internal properties no longer appear in the context argument
                 of a custom inspection function.
  - version: v11.11.0
    pr-url: https://github.com/nodejs/node/pull/26269
    description: The `compact` option accepts numbers for a new output mode.
  - version: v11.7.0
    pr-url: https://github.com/nodejs/node/pull/25006
    description: ArrayBuffers now also show their binary contents.
  - version: v11.5.0
    pr-url: https://github.com/nodejs/node/pull/24852
    description: The `getters` option is supported now.
  - version: v11.4.0
    pr-url: https://github.com/nodejs/node/pull/24326
    description: The `depth` default changed back to `2`.
  - version: v11.0.0
    pr-url: https://github.com/nodejs/node/pull/22846
    description: The `depth` default changed to `20`.
  - version: v11.0.0
    pr-url: https://github.com/nodejs/node/pull/22756
    description: The inspection output is now limited to about 128 MiB. Data
                 above that size will not be fully inspected.
  - version: v10.12.0
    pr-url: https://github.com/nodejs/node/pull/22788
    description: The `sorted` option is supported now.
  - version: v10.6.0
    pr-url: https://github.com/nodejs/node/pull/20725
    description: Inspecting linked lists and similar objects is now possible
                 up to the maximum call stack size.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19259
    description: The `WeakMap` and `WeakSet` entries can now be inspected
                 as well.
  - version: v9.9.0
    pr-url: https://github.com/nodejs/node/pull/17576
    description: The `compact` option is supported now.
  - version: v6.6.0
    pr-url: https://github.com/nodejs/node/pull/8174
    description: Custom inspection functions can now return `this`.
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/7499
    description: The `breakLength` option is supported now.
  - version: v6.1.0
    pr-url: https://github.com/nodejs/node/pull/6334
    description: The `maxArrayLength` option is supported now; in particular,
                 long arrays are truncated by default.
  - version: v6.1.0
    pr-url: https://github.com/nodejs/node/pull/6465
    description: The `showProxy` option is supported now.
-->

- `object` {any} 任何 JavaScript 原始值或 `Object`。
- `options` {Object}
  - `showHidden` {boolean} 如果为 `true`，则 `object` 的不可枚举符号和属性将包含在格式化结果中。{WeakMap} 和 {WeakSet} 条目以及用户定义的原型属性（不包括方法属性）也将被包含。**默认值:** `false`。
  - `depth` {number} 指定格式化 `object` 时递归的次数。这对于检查大型对象很有用。要递归到最大调用堆栈大小，请传递 `Infinity` 或 `null`。
    **默认值:** `2`。
  - `colors` {boolean} 如果为 `true`，则输出使用 ANSI 颜色代码进行样式设置。颜色是可定制的。参见 [自定义 `util.inspect` 颜色][]。
    **默认值:** `false`。
  - `customInspect` {boolean} 如果为 `false`，则不调用 `[util.inspect.custom](depth, opts, inspect)` 函数。
    **默认值:** `true`。
  - `showProxy` {boolean} 如果为 `true`，则 `Proxy` 检查包括 [`target` 和 `handler`][] 对象。**默认值:** `false`。
  - `maxArrayLength` {integer} 指定格式化时要包含的 `Array`、{TypedArray}、{Map}、{WeakMap} 和 {WeakSet} 元素的最大数量。设置为 `null` 或 `Infinity` 可显示所有元素。设置为 `0` 或负数则不显示任何元素。**默认值:** `100`。
  - `maxStringLength` {integer} 指定格式化时要包含的最大字符数。设置为 `null` 或 `Infinity` 可显示所有元素。设置为 `0` 或负数则不显示任何字符。**默认值:** `10000`。
  - `breakLength` {integer} 输入值被分割成多行的长度。设置为 `Infinity` 可将输入格式化为单行（结合 `compact` 设置为 `true` 或任何大于等于 `1` 的数字）。
    **默认值:** `80`。
  - `compact` {boolean|integer} 将其设置为 `false` 会导致每个对象键显示在新行上。它将在比 `breakLength` 长的文本中的新行处换行。如果设置为数字，则最多 `n` 个内部元素将合并为单行，只要所有属性都适合 `breakLength`。短数组元素也会分组在一起。有关更多信息，请参见下面的示例。**默认值:** `3`。
  - `sorted` {boolean|Function} 如果设置为 `true` 或函数，则对象的所有属性以及 `Set` 和 `Map` 的条目将在结果字符串中排序。如果设置为 `true`，则使用 [默认排序][]。如果设置为函数，则将其用作 [比较函数][]。
  - `getters` {boolean|string} 如果设置为 `true`，则检查 getter。如果设置为 `'get'`，则仅检查没有相应 setter 的 getter。如果设置为 `'set'`，则仅检查有相应 setter 的 getter。这可能会根据 getter 函数产生副作用。
    **默认值:** `false`。
  - `numericSeparator` {boolean} 如果设置为 `true`，则所有 bigints 和数字中每三位数字用一个下划线分隔。
    **默认值:** `false`。
- 返回: {string} `object` 的表示形式。

`util.inspect()` 方法返回一个 `object` 的字符串表示形式，用于调试。`util.inspect` 的输出可能会随时更改，不应以编程方式依赖。可以传递额外的 `options` 来改变结果。
`util.inspect()` 将使用构造函数的名称和/或 `Symbol.toStringTag` 属性为被检查的值创建一个可识别的标签。

```js
class Foo {
  get [Symbol.toStringTag]() {
    return 'bar';
  }
}

class Bar {}

const baz = Object.create(null, { [Symbol.toStringTag]: { value: 'foo' } });

util.inspect(new Foo()); // 'Foo [bar] {}'
util.inspect(new Bar()); // 'Bar {}'
util.inspect(baz); // '[foo] {}'
```

循环引用通过使用引用索引指向其锚点：

```mjs
import { inspect } from 'node:util';

const obj = {};
obj.a = [obj];
obj.b = {};
obj.b.inner = obj.b;
obj.b.obj = obj;

console.log(inspect(obj));
// <ref *1> {
//   a: [ [Circular *1] ],
//   b: <ref *2> { inner: [Circular *2], obj: [Circular *1] }
// }
```

```cjs
const { inspect } = require('node:util');

const obj = {};
obj.a = [obj];
obj.b = {};
obj.b.inner = obj.b;
obj.b.obj = obj;

console.log(inspect(obj));
// <ref *1> {
//   a: [ [Circular *1] ],
//   b: <ref *2> { inner: [Circular *2], obj: [Circular *1] }
// }
```

以下示例检查 `util` 对象的所有属性：

```mjs
import util from 'node:util';

console.log(util.inspect(util, { showHidden: true, depth: null }));
```

```cjs
const util = require('node:util');

console.log(util.inspect(util, { showHidden: true, depth: null }));
```

以下示例突出了 `compact` 选项的效果：

```mjs
import { inspect } from 'node:util';

const o = {
  a: [
    1,
    2,
    [
      [
        'Lorem ipsum dolor sit amet,\nconsectetur adipiscing elit, sed do ' +
          'eiusmod \ntempor incididunt ut labore et dolore magna aliqua.',
        'test',
        'foo',
      ],
    ],
    4,
  ],
  b: new Map([
    ['za', 1],
    ['zb', 'test'],
  ]),
};
console.log(inspect(o, { compact: true, depth: 5, breakLength: 80 }));

// { a:
//   [ 1,
//     2,
//     [ [ 'Lorem ipsum dolor sit amet,\nconsectetur [...]', // 长行
//           'test',
//           'foo' ] ],
//     4 ],
//   b: Map(2) { 'za' => 1, 'zb' => 'test' } }

// 将 `compact` 设置为 false 或整数会创建更具可读性的输出。
console.log(inspect(o, { compact: false, depth: 5, breakLength: 80 }));

// {
//   a: [
//     1,
//     2,
//     [
//       [
//         'Lorem ipsum dolor sit amet,\n' +
//           'consectetur adipiscing elit, sed do eiusmod \n' +
//           'tempor incididunt ut labore et dolore magna aliqua.',
//         'test',
//         'foo'
//       ]
//     ],
//     4
//   ],
//   b: Map(2) {
//     'za' => 1,
//     'zb' => 'test'
//   }
// }

// 将 `breakLength` 设置为例如 150，将在单行中打印 "Lorem ipsum" 文本。
```

```cjs
const { inspect } = require('node:util');

const o = {
  a: [
    1,
    2,
    [
      [
        'Lorem ipsum dolor sit amet,\nconsectetur adipiscing elit, sed do ' +
          'eiusmod \ntempor incididunt ut labore et dolore magna aliqua.',
        'test',
        'foo',
      ],
    ],
    4,
  ],
  b: new Map([
    ['za', 1],
    ['zb', 'test'],
  ]),
};
console.log(inspect(o, { compact: true, depth: 5, breakLength: 80 }));

// { a:
//   [ 1,
//     2,
//     [ [ 'Lorem ipsum dolor sit amet,\nconsectetur [...]', // 长行
//           'test',
//           'foo' ] ],
//     4 ],
//   b: Map(2) { 'za' => 1, 'zb' => 'test' } }

// 将 `compact` 设置为 false 或整数会创建更具可读性的输出。
console.log(inspect(o, { compact: false, depth: 5, breakLength: 80 }));

// {
//   a: [
//     1,
//     2,
//     [
//       [
//         'Lorem ipsum dolor sit amet,\n' +
//           'consectetur adipiscing elit, sed do eiusmod \n' +
//           'tempor incididunt ut labore et dolore magna aliqua.',
//         'test',
//         'foo'
//       ]
//     ],
//     4
//   ],
//   b: Map(2) {
//     'za' => 1,
//     'zb' => 'test'
//   }
// }

// 将 `breakLength` 设置为例如 150，将在单行中打印 "Lorem ipsum" 文本。
```

`showHidden` 选项允许检查 {WeakMap} 和 {WeakSet} 条目。如果条目数超过 `maxArrayLength`，则无法保证显示哪些条目。这意味着两次检索相同的 {WeakSet} 条目可能会导致不同的输出。此外，没有剩余强引用的条目可能随时被垃圾回收。

```mjs
import { inspect } from 'node:util';

const obj = { a: 1 };
const obj2 = { b: 2 };
const weakSet = new WeakSet([obj, obj2]);

console.log(inspect(weakSet, { showHidden: true }));
// WeakSet { { a: 1 }, { b: 2 } }
```

```cjs
const { inspect } = require('node:util');

const obj = { a: 1 };
const obj2 = { b: 2 };
const weakSet = new WeakSet([obj, obj2]);

console.log(inspect(weakSet, { showHidden: true }));
// WeakSet { { a: 1 }, { b: 2 } }
```

`sorted` 选项确保对象的属性插入顺序不会影响 `util.inspect()` 的结果。

```mjs
import { inspect } from 'node:util';
import assert from 'node:assert';

const o1 = {
  b: [2, 3, 1],
  a: '`a` comes before `b`',
  c: new Set([2, 3, 1]),
};
console.log(inspect(o1, { sorted: true }));
// { a: '`a` comes before `b`', b: [ 2, 3, 1 ], c: Set(3) { 1, 2, 3 } }
console.log(inspect(o1, { sorted: (a, b) => b.localeCompare(a) }));
// { c: Set(3) { 3, 2, 1 }, b: [ 2, 3, 1 ], a: '`a` comes before `b`' }

const o2 = {
  c: new Set([2, 1, 3]),
  a: '`a` comes before `b`',
  b: [2, 3, 1],
};
assert.strict.equal(
  inspect(o1, { sorted: true }),
  inspect(o2, { sorted: true })
);
```

```cjs
const { inspect } = require('node:util');
const assert = require('node:assert');

const o1 = {
  b: [2, 3, 1],
  a: '`a` comes before `b`',
  c: new Set([2, 3, 1]),
};
console.log(inspect(o1, { sorted: true }));
// { a: '`a` comes before `b`', b: [ 2, 3, 1 ], c: Set(3) { 1, 2, 3 } }
console.log(inspect(o1, { sorted: (a, b) => b.localeCompare(a) }));
// { c: Set(3) { 3, 2, 1 }, b: [ 2, 3, 1 ], a: '`a` comes before `b`' }

const o2 = {
  c: new Set([2, 1, 3]),
  a: '`a` comes before `b`',
  b: [2, 3, 1],
};
assert.strict.equal(
  inspect(o1, { sorted: true }),
  inspect(o2, { sorted: true })
);
```

`numericSeparator` 选项在所有数字中每三位数字添加一个下划线。

```mjs
import { inspect } from 'node:util';

const thousand = 1000;
const million = 1000000;
const bigNumber = 123456789n;
const bigDecimal = 1234.12345;

console.log(inspect(thousand, { numericSeparator: true }));
// 1_000
console.log(inspect(million, { numericSeparator: true }));
// 1_000_000
console.log(inspect(bigNumber, { numericSeparator: true }));
// 123_456_789n
console.log(inspect(bigDecimal, { numericSeparator: true }));
// 1_234.123_45
```

```cjs
const { inspect } = require('node:util');

const thousand = 1000;
const million = 1000000;
const bigNumber = 123456789n;
const bigDecimal = 1234.12345;

console.log(inspect(thousand, { numericSeparator: true }));
// 1_000
console.log(inspect(million, { numericSeparator: true }));
// 1_000_000
console.log(inspect(bigNumber, { numericSeparator: true }));
// 123_456_789n
console.log(inspect(bigDecimal, { numericSeparator: true }));
// 1_234.123_45
```

`util.inspect()` 是一个用于调试的同步方法。其最大输出长度约为 128 MiB。导致更长输出的输入将被截断。

### 自定义 `util.inspect` 颜色

<!-- type=misc -->

`util.inspect` 的颜色输出（如果启用）可以通过 `util.inspect.styles` 和 `util.inspect.colors` 属性全局自定义。

`util.inspect.styles` 是一个将样式名称映射到 `util.inspect.colors` 中的颜色的映射。

默认样式和关联的颜色如下：

- `bigint`: `yellow`
- `boolean`: `yellow`
- `date`: `magenta`
- `module`: `underline`
- `name`: (无样式)
- `null`: `bold`
- `number`: `yellow`
- `regexp`: `red`
- `special`: `cyan` (例如，`Proxies`)
- `string`: `green`
- `symbol`: `green`
- `undefined`: `grey`

颜色样式使用 ANSI 控制代码，可能并非所有终端都支持。要验证颜色支持，请使用 [`tty.hasColors()`][]。

预定义的控制代码如下（分组为“修饰符”、“前景色”和“背景色”）。

#### 修饰符

不同终端对修饰符的支持各不相同。如果不支持，它们大多会被忽略。

- `reset` - 将所有（颜色）修饰符重置为默认值
- **bold** - 使文本加粗
- _italic_ - 使文本斜体
- <span style="border-bottom: 1px solid;">underline</span> - 给文本添加下划线
- ~~strikethrough~~ - 在文本中央画一条水平线（别名：`strikeThrough`、`crossedout`、`crossedOut`）
- `hidden` - 打印文本，但使其不可见（别名：`conceal`）
- <span style="opacity: 0.5;">dim</span> - 降低颜色强度（别名：`faint`）
- <span style="border-top: 1px solid;">overlined</span> - 给文本添加上划线
- blink - 以间隔隐藏和显示文本
- <span style="filter: invert(100%);">inverse</span> - 交换前景色和背景色（别名：`swapcolors`、`swapColors`）
- <span style="border-bottom: 1px double;">doubleunderline</span> - 给文本添加双下划线（别名：`doubleUnderline`）
- <span style="border: 1px solid;">framed</span> - 在文本周围绘制一个框

#### 前景色

- `black`
- `red`
- `green`
- `yellow`
- `blue`
- `magenta`
- `cyan`
- `white`
- `gray` (别名：`grey`、`blackBright`)
- `redBright`
- `greenBright`
- `yellowBright`
- `blueBright`
- `magentaBright`
- `cyanBright`
- `whiteBright`

#### 背景色

- `bgBlack`
- `bgRed`
- `bgGreen`
- `bgYellow`
- `bgBlue`
- `bgMagenta`
- `bgCyan`
- `bgWhite`
- `bgGray` (别名：`bgGrey`、`bgBlackBright`)
- `bgRedBright`
- `bgGreenBright`
- `bgYellowBright`
- `bgBlueBright`
- `bgMagentaBright`
- `bgCyanBright`
- `bgWhiteBright`

### 对象上的自定义检查函数

<!-- type=misc -->

<!-- YAML
added: v0.1.97
changes:
  - version:
      - v17.3.0
      - v16.14.0
    pr-url: https://github.com/nodejs/node/pull/41019
    description: The inspect argument is added for more interoperability.
-->

对象也可以定义自己的 [`[util.inspect.custom](depth, opts, inspect)`][util.inspect.custom] 函数，`util.inspect()` 将在检查该对象时调用并使用其结果。

```mjs
import { inspect } from 'node:util';

class Box {
  constructor(value) {
    this.value = value;
  }

  [inspect.custom](depth, options, inspect) {
    if (depth < 0) {
      return options.stylize('[Box]', 'special');
    }

    const newOptions = Object.assign({}, options, {
      depth: options.depth === null ? null : options.depth - 1,
    });

    // 五个空格填充，因为这是 "Box< " 的大小。
    const padding = ' '.repeat(5);
    const inner = inspect(this.value, newOptions).replace(
      /\n/g,
      `\n${padding}`
    );
    return `${options.stylize('Box', 'special')}< ${inner} >`;
  }
}

const box = new Box(true);

console.log(inspect(box));
// "Box< true >"
```

```cjs
const { inspect } = require('node:util');

class Box {
  constructor(value) {
    this.value = value;
  }

  [inspect.custom](depth, options, inspect) {
    if (depth < 0) {
      return options.stylize('[Box]', 'special');
    }

    const newOptions = Object.assign({}, options, {
      depth: options.depth === null ? null : options.depth - 1,
    });

    // 五个空格填充，因为这是 "Box< " 的大小。
    const padding = ' '.repeat(5);
    const inner = inspect(this.value, newOptions).replace(
      /\n/g,
      `\n${padding}`
    );
    return `${options.stylize('Box', 'special')}< ${inner} >`;
  }
}

const box = new Box(true);

console.log(inspect(box));
// "Box< true >"
```

自定义的 `[util.inspect.custom](depth, opts, inspect)` 函数通常返回一个字符串，但可以返回任何类型的值，该值将由 `util.inspect()` 相应地格式化。

```mjs
import { inspect } from 'node:util';

const obj = { foo: 'this will not show up in the inspect() output' };
obj[inspect.custom] = (depth) => {
  return { bar: 'baz' };
};

console.log(inspect(obj));
// "{ bar: 'baz' }"
```

```cjs
const { inspect } = require('node:util');

const obj = { foo: 'this will not show up in the inspect() output' };
obj[inspect.custom] = (depth) => {
  return { bar: 'baz' };
};

console.log(inspect(obj));
// "{ bar: 'baz' }"
```

### `util.inspect.custom`

<!-- YAML
added: v6.6.0
changes:
  - version: v10.12.0
    pr-url: https://github.com/nodejs/node/pull/20857
    description: This is now defined as a shared symbol.
-->

- 类型: {symbol} 可用于声明自定义检查函数的符号。

除了可以通过 `util.inspect.custom` 访问外，此符号还在 [全局符号注册表][global symbol registry] 中注册，可以在任何环境中作为 `Symbol.for('nodejs.util.inspect.custom')` 访问。

使用此符号可以编写可移植的代码，以便在 Node.js 环境中使用自定义检查函数，而在浏览器中忽略它。`util.inspect()` 函数本身作为第三个参数传递给自定义检查函数，以允许进一步的可移植性。

```js
const customInspectSymbol = Symbol.for('nodejs.util.inspect.custom');

class Password {
  constructor(value) {
    this.value = value;
  }

  toString() {
    return 'xxxxxxxx';
  }

  [customInspectSymbol](depth, inspectOptions, inspect) {
    return `Password <${this.toString()}>`;
  }
}

const password = new Password('r0sebud');
console.log(password);
// 打印 Password <xxxxxxxx>
```

有关更多详细信息，请参阅 [对象上的自定义检查函数][]。

### `util.inspect.defaultOptions`

<!-- YAML
added: v6.4.0
-->

`defaultOptions` 值允许自定义 `util.inspect` 使用的默认选项。这对于像 `console.log` 或 `util.format` 这样隐式调用 `util.inspect` 的函数很有用。它应设置为包含一个或多个有效 [`util.inspect()`][] 选项的对象。直接设置选项属性也受支持。

```mjs
import { inspect } from 'node:util';
const arr = Array(156).fill(0);

console.log(arr); // 记录被截断的数组
inspect.defaultOptions.maxArrayLength = null;
console.log(arr); // 记录完整的数组
```

```cjs
const { inspect } = require('node:util');
const arr = Array(156).fill(0);

console.log(arr); // 记录被截断的数组
inspect.defaultOptions.maxArrayLength = null;
console.log(arr); // 记录完整的数组
```

## `util.isDeepStrictEqual(val1, val2[, options])`

<!-- YAML
added: v9.0.0
changes:
  - version: v24.9.0
    pr-url: https://github.com/nodejs/node/pull/59762
    description: Added `options` parameter to allow skipping prototype comparison.
-->

- `val1` {any}
- `val2` {any}
- `skipPrototype` {boolean} 如果为 `true`，则在深度严格相等性检查期间跳过原型和构造函数的比较。**默认值:** `false`。
- 返回: {boolean}

如果 `val1` 和 `val2` 之间存在深度严格相等性，则返回 `true`。否则，返回 `false`。

默认情况下，深度严格相等性包括对象原型和构造函数的比较。当 `skipPrototype` 为 `true` 时，具有不同原型或构造函数的对象如果其可枚举属性深度严格相等，仍可被视为相等。

```js
const util = require('node:util');

class Foo {
  constructor(a) {
    this.a = a;
  }
}

class Bar {
  constructor(a) {
    this.a = a;
  }
}

const foo = new Foo(1);
const bar = new Bar(1);

// 不同的构造函数，相同的属性
console.log(util.isDeepStrictEqual(foo, bar));
// false

console.log(util.isDeepStrictEqual(foo, bar, true));
// true
```

有关深度严格相等的更多信息，请参阅 [`assert.deepStrictEqual()`][]。

## 类：`util.MIMEType`

<!-- YAML
added:
  - v19.1.0
  - v18.13.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[MIMEType 类](https://bmeck.github.io/node-proposal-mime-api/) 的一个实现。

根据浏览器约定，`MIMEType` 对象的所有属性都作为类原型上的 getter 和 setter 实现，而不是作为对象本身的数据属性。

MIME 字符串是一个包含多个有意义组件的结构化字符串。解析时，会返回一个 `MIMEType` 对象，其中包含每个这些组件的属性。

### `new MIMEType(input)`

- `input` {string} 要解析的输入 MIME

通过解析 `input` 创建一个新的 `MIMEType` 对象。

```mjs
import { MIMEType } from 'node:util';

const myMIME = new MIMEType('text/plain');
```

```cjs
const { MIMEType } = require('node:util');

const myMIME = new MIMEType('text/plain');
```

如果 `input` 不是有效的 MIME，将抛出 `TypeError`。请注意，将尝试将给定值强制转换为字符串。例如：

```mjs
import { MIMEType } from 'node:util';
const myMIME = new MIMEType({ toString: () => 'text/plain' });
console.log(String(myMIME));
// 打印: text/plain
```

```cjs
const { MIMEType } = require('node:util');
const myMIME = new MIMEType({ toString: () => 'text/plain' });
console.log(String(myMIME));
// 打印: text/plain
```

### `mime.type`

- 类型: {string}

获取和设置 MIME 的类型部分。

```mjs
import { MIMEType } from 'node:util';

const myMIME = new MIMEType('text/javascript');
console.log(myMIME.type);
// 打印: text
myMIME.type = 'application';
console.log(myMIME.type);
// 打印: application
console.log(String(myMIME));
// 打印: application/javascript
```

```cjs
const { MIMEType } = require('node:util');

const myMIME = new MIMEType('text/javascript');
console.log(myMIME.type);
// 打印: text
myMIME.type = 'application';
console.log(myMIME.type);
// 打印: application
console.log(String(myMIME));
// 打印: application/javascript
```

### `mime.subtype`

- 类型: {string}

获取和设置 MIME 的子类型部分。

```mjs
import { MIMEType } from 'node:util';

const myMIME = new MIMEType('text/ecmascript');
console.log(myMIME.subtype);
// 打印: ecmascript
myMIME.subtype = 'javascript';
console.log(myMIME.subtype);
// 打印: javascript
console.log(String(myMIME));
// 打印: text/javascript
```

```cjs
const { MIMEType } = require('node:util');

const myMIME = new MIMEType('text/ecmascript');
console.log(myMIME.subtype);
// 打印: ecmascript
myMIME.subtype = 'javascript';
console.log(myMIME.subtype);
// 打印: javascript
console.log(String(myMIME));
// 打印: text/javascript
```

### `mime.essence`

- 类型: {string}

获取 MIME 的实质部分。此属性是只读的。
使用 `mime.type` 或 `mime.subtype` 来更改 MIME。

```mjs
import { MIMEType } from 'node:util';

const myMIME = new MIMEType('text/javascript;key=value');
console.log(myMIME.essence);
// 打印: text/javascript
myMIME.type = 'application';
console.log(myMIME.essence);
// 打印: application/javascript
console.log(String(myMIME));
// 打印: application/javascript;key=value
```

```cjs
const { MIMEType } = require('node:util');

const myMIME = new MIMEType('text/javascript;key=value');
console.log(myMIME.essence);
// 打印: text/javascript
myMIME.type = 'application';
console.log(myMIME.essence);
// 打印: application/javascript
console.log(String(myMIME));
// 打印: application/javascript;key=value
```

### `mime.params`

- 类型: {MIMEParams}

获取表示 MIME 参数的 [`MIMEParams`][] 对象。此属性是只读的。有关详细信息，请参阅 [`MIMEParams`][] 文档。

### `mime.toString()`

- 返回: {string}

`MIMEType` 对象上的 `toString()` 方法返回序列化的 MIME。

由于需要符合标准，此方法不允许用户自定义 MIME 的序列化过程。

### `mime.toJSON()`

- 返回: {string}

[`mime.toString()`][] 的别名。

当使用 [`JSON.stringify()`][] 序列化 `MIMEType` 对象时，此方法会自动调用。

```mjs
import { MIMEType } from 'node:util';

const myMIMES = [new MIMEType('image/png'), new MIMEType('image/gif')];
console.log(JSON.stringify(myMIMES));
// 打印: ["image/png", "image/gif"]
```

```cjs
const { MIMEType } = require('node:util');

const myMIMES = [new MIMEType('image/png'), new MIMEType('image/gif')];
console.log(JSON.stringify(myMIMES));
// 打印: ["image/png", "image/gif"]
```

## 类：`util.MIMEParams`

<!-- YAML
added:
  - v19.1.0
  - v18.13.0
-->

`MIMEParams` API 提供对 `MIMEType` 参数的读写访问。

### `new MIMEParams()`

创建一个具有空参数的新 `MIMEParams` 对象

```mjs
import { MIMEParams } from 'node:util';

const myParams = new MIMEParams();
```

```cjs
const { MIMEParams } = require('node:util');

const myParams = new MIMEParams();
```

### `mimeParams.delete(name)`

- `name` {string}

删除所有名称为 `name` 的名称-值对。

### `mimeParams.entries()`

- 返回: {Iterator}

返回参数中每个名称-值对的迭代器。迭代器的每个项都是一个 JavaScript `Array`。数组的第一项是 `name`，第二项是 `value`。

### `mimeParams.get(name)`

- `name` {string}
- 返回: {string | null} 一个字符串，如果没有名称为 `name` 的名称-值对，则为 `null`。

返回第一个名称为 `name` 的名称-值对的值。如果没有这样的对，则返回 `null`。

### `mimeParams.has(name)`

- `name` {string}
- 返回: {boolean}

如果至少有一个名称为 `name` 的名称-值对，则返回 `true`。

### `mimeParams.keys()`

- 返回: {Iterator}

返回每个名称-值对名称的迭代器。

```mjs
import { MIMEType } from 'node:util';

const { params } = new MIMEType('text/plain;foo=0;bar=1');
for (const name of params.keys()) {
  console.log(name);
}
// 打印:
//   foo
//   bar
```

```cjs
const { MIMEType } = require('node:util');

const { params } = new MIMEType('text/plain;foo=0;bar=1');
for (const name of params.keys()) {
  console.log(name);
}
// 打印:
//   foo
//   bar
```

### `mimeParams.set(name, value)`

- `name` {string}
- `value` {string}

将 `MIMEParams` 对象中与 `name` 关联的值设置为 `value`。如果存在任何名称为 `name` 的预先存在的名称-值对，则将第一个这样的对的值设置为 `value`。

```mjs
import { MIMEType } from 'node:util';

const { params } = new MIMEType('text/plain;foo=0;bar=1');
params.set('foo', 'def');
params.set('baz', 'xyz');
console.log(params.toString());
// 打印: foo=def;bar=1;baz=xyz
```

```cjs
const { MIMEType } = require('node:util');

const { params } = new MIMEType('text/plain;foo=0;bar=1');
params.set('foo', 'def');
params.set('baz', 'xyz');
console.log(params.toString());
// 打印: foo=def;bar=1;baz=xyz
```

### `mimeParams.values()`

- 返回: {Iterator}

返回每个名称-值对值的迭代器。

### `mimeParams[Symbol.iterator]()`

- 返回: {Iterator}

[`mimeParams.entries()`][] 的别名。

```mjs
import { MIMEType } from 'node:util';

const { params } = new MIMEType('text/plain;foo=bar;xyz=baz');
for (const [name, value] of params) {
  console.log(name, value);
}
// 打印:
//   foo bar
//   xyz baz
```

```cjs
const { MIMEType } = require('node:util');

const { params } = new MIMEType('text/plain;foo=bar;xyz=baz');
for (const [name, value] of params) {
  console.log(name, value);
}
// 打印:
//   foo bar
//   xyz baz
```

## `util.parseArgs([config])`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53107
    description: add support for allowing negative options in input `config`.
  - version:
    - v20.0.0
    pr-url: https://github.com/nodejs/node/pull/46718
    description: The API is no longer experimental.
  - version:
    - v18.11.0
    - v16.19.0
    pr-url: https://github.com/nodejs/node/pull/44631
    description: Add support for default values in input `config`.
  - version:
    - v18.7.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/43459
    description: add support for returning detailed parse information
                 using `tokens` in input `config` and returned properties.
-->

- `config` {Object} 用于提供解析参数并配置解析器。`config` 支持以下属性：

  - `args` {string\[]} 参数字符串数组。**默认值:** `process.argv`，移除了 `execPath` 和 `filename`。
  - `options` {Object} 用于描述解析器已知的参数。
    `options` 的键是选项的长名称，值是一个 {Object}，接受以下属性：
    - `type` {string} 参数的类型，必须是 `boolean` 或 `string`。
    - `multiple` {boolean} 此选项是否可以提供多次。如果为 `true`，则所有值将收集在一个数组中。如果为 `false`，则选项的值是最后出现的值。**默认值:** `false`。
    - `short` {string} 选项的单字符别名。
    - `default` {string | boolean | string\[] | boolean\[]} 如果选项不出现在要解析的参数中，则为该选项分配的值。该值必须与 `type` 属性指定的类型匹配。如果 `multiple` 为 `true`，则它必须是一个数组。当选项确实出现在要解析的参数中时，不应用默认值，即使提供的值是假值。
  - `strict` {boolean} 当遇到未知参数或传递的参数与 `options` 中配置的 `type` 不匹配时，是否应抛出错误。
    **默认值:** `true`。
  - `allowPositionals` {boolean} 此命令是否接受位置参数。
    **默认值:** 如果 `strict` 为 `true` 则为 `false`，否则为 `true`。
  - `allowNegative` {boolean} 如果为 `true`，允许通过在以 `--no-` 为前缀的选项名称中明确将布尔选项设置为 `false`。
    **默认值:** `false`。
  - `tokens` {boolean} 返回解析后的令牌。这对于扩展内置行为非常有用，从添加额外检查到以不同方式重新处理令牌。
    **默认值:** `false`。

- 返回: {Object} 解析后的命令行参数：
  - `values` {Object} 解析后的选项名称及其 {string} 或 {boolean} 值的映射。
  - `positionals` {string\[]} 位置参数。
  - `tokens` {Object\[] | undefined} 请参阅 [parseArgs 令牌](#parseargs-tokens) 部分。仅在 `config` 包含 `tokens: true` 时返回。

提供了一种比直接与 `process.argv` 交互更高级别的命令行参数解析 API。接受预期参数的规范，并返回一个包含已解析选项和位置的结构化对象。

```mjs
import { parseArgs } from 'node:util';
const args = ['-f', '--bar', 'b'];
const options = {
  foo: {
    type: 'boolean',
    short: 'f',
  },
  bar: {
    type: 'string',
  },
};
const { values, positionals } = parseArgs({ args, options });
console.log(values, positionals);
// 打印: [Object: null prototype] { foo: true, bar: 'b' } []
```

```cjs
const { parseArgs } = require('node:util');
const args = ['-f', '--bar', 'b'];
const options = {
  foo: {
    type: 'boolean',
    short: 'f',
  },
  bar: {
    type: 'string',
  },
};
const { values, positionals } = parseArgs({ args, options });
console.log(values, positionals);
// 打印: [Object: null prototype] { foo: true, bar: 'b' } []
```

### `parseArgs` `tokens`

通过指定配置中的 `tokens: true`，可以提供详细的解析信息以添加自定义行为。
返回的令牌具有描述以下内容的属性：

- 所有令牌
  - `kind` {string} 'option'、'positional' 或 'option-terminator' 之一。
  - `index` {number} `args` 中包含令牌的元素的索引。因此，令牌的源参数是 `args[token.index]`。
- 选项令牌
  - `name` {string} 选项的长名称。
  - `rawName` {string} 在 args 中使用的选项，如 `-f` 或 `--foo`。
  - `value` {string | undefined} 在 args 中指定的选项值。布尔选项未定义。
  - `inlineValue` {boolean | undefined} 选项值是否内联指定，如 `--foo=bar`。
- 位置令牌
  - `value` {string} 位置参数在 args 中的值（即 `args[index]`）。
- 选项终止符令牌

返回的令牌按照在输入 args 中遇到的顺序排列。选项在 args 中出现多次会产生每次使用的令牌。短选项组如 `-xy` 扩展为每个选项的令牌。所以 `-xxx` 产生三个令牌。

例如，要添加对像 `--no-color` 这样的否定选项的支持（当选项是 `boolean` 类型时，`allowNegative` 支持此功能），可以重新处理返回的令牌以更改为否定选项存储的值。

```mjs
import { parseArgs } from 'node:util';

const options = {
  color: { type: 'boolean' },
  'no-color': { type: 'boolean' },
  logfile: { type: 'string' },
  'no-logfile': { type: 'boolean' },
};
const { values, tokens } = parseArgs({ options, tokens: true });

// 重新处理选项令牌并覆盖返回的值。
tokens
  .filter((token) => token.kind === 'option')
  .forEach((token) => {
    if (token.name.startsWith('no-')) {
      // 为 --no-foo 存储 foo:false
      const positiveName = token.name.slice(3);
      values[positiveName] = false;
      delete values[token.name];
    } else {
      // 重新保存值，以便如果同时出现 --foo 和 --no-foo，最后一个获胜。
      values[token.name] = token.value ?? true;
    }
  });

const color = values.color;
const logfile = values.logfile ?? 'default.log';

console.log({ logfile, color });
```

```cjs
const { parseArgs } = require('node:util');

const options = {
  color: { type: 'boolean' },
  'no-color': { type: 'boolean' },
  logfile: { type: 'string' },
  'no-logfile': { type: 'boolean' },
};
const { values, tokens } = parseArgs({ options, tokens: true });

// 重新处理选项令牌并覆盖返回的值。
tokens
  .filter((token) => token.kind === 'option')
  .forEach((token) => {
    if (token.name.startsWith('no-')) {
      // 为 --no-foo 存储 foo:false
      const positiveName = token.name.slice(3);
      values[positiveName] = false;
      delete values[token.name];
    } else {
      // 重新保存值，以便如果同时出现 --foo 和 --no-foo，最后一个获胜。
      values[token.name] = token.value ?? true;
    }
  });

const color = values.color;
const logfile = values.logfile ?? 'default.log';

console.log({ logfile, color });
```

显示否定选项的用法示例，以及当选项以多种方式使用时最后一个获胜的情况。

```console
$ node negate.js
{ logfile: 'default.log', color: undefined }
$ node negate.js --no-logfile --no-color
{ logfile: false, color: false }
$ node negate.js --logfile=test.log --color
{ logfile: 'test.log', color: true }
$ node negate.js --no-logfile --logfile=test.log --color --no-color
{ logfile: 'test.log', color: false }
```

## `util.parseEnv(content)`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
changes:
  - version: v24.10.0
    pr-url: https://github.com/nodejs/node/pull/59925
    description: This API is no longer experimental.
-->

- `content` {string}

.env 文件的原始内容。

- 返回: {Object}

给定一个示例 .env 文件：

```cjs
const { parseEnv } = require('node:util');

parseEnv('HELLO=world\nHELLO=oh my\n');
// 返回: { HELLO: 'oh my' }
```

```mjs
import { parseEnv } from 'node:util';

parseEnv('HELLO=world\nHELLO=oh my\n');
// 返回: { HELLO: 'oh my' }
```

## `util.promisify(original)`

<!-- YAML
added: v8.0.0
changes:
  - version: v20.8.0
    pr-url: https://github.com/nodejs/node/pull/49647
    description: Calling `promisify` on a function that returns a `Promise` is
                 deprecated.
-->

- `original` {Function}
- 返回: {Function}

接受一个遵循常见的错误优先回调风格的函数，即最后一个参数是 `(err, value) => ...` 回调，并返回一个返回 promise 的版本。

```mjs
import { promisify } from 'node:util';
import { stat } from 'node:fs';

const promisifiedStat = promisify(stat);
promisifiedStat('.')
  .then((stats) => {
    // 使用 `stats` 做一些事情
  })
  .catch((error) => {
    // 处理错误。
  });
```

```cjs
const { promisify } = require('node:util');
const { stat } = require('node:fs');

const promisifiedStat = promisify(stat);
promisifiedStat('.')
  .then((stats) => {
    // 使用 `stats` 做一些事情
  })
  .catch((error) => {
    // 处理错误。
  });
```

或者，等效地使用 `async function`：

```mjs
import { promisify } from 'node:util';
import { stat } from 'node:fs';

const promisifiedStat = promisify(stat);

async function callStat() {
  const stats = await promisifiedStat('.');
  console.log(`This directory is owned by ${stats.uid}`);
}

callStat();
```

```cjs
const { promisify } = require('node:util');
const { stat } = require('node:fs');

const promisifiedStat = promisify(stat);

async function callStat() {
  const stats = await promisifiedStat('.');
  console.log(`This directory is owned by ${stats.uid}`);
}

callStat();
```

如果存在 `original[util.promisify.custom]` 属性，`promisify` 将返回其值，请参阅 [自定义 promise 化函数][]。

`promisify()` 假设在所有情况下 `original` 都是一个将回调作为其最终参数的函数。如果 `original` 不是函数，`promisify()` 将抛出错误。如果 `original` 是函数但其最后一个参数不是错误优先回调，则它仍然会传递一个错误优先回调作为其最后一个参数。

在类方法或其他使用 `this` 的方法上使用 `promisify()` 可能无法按预期工作，除非特殊处理：

```mjs
import { promisify } from 'node:util';

class Foo {
  constructor() {
    this.a = 42;
  }

  bar(callback) {
    callback(null, this.a);
  }
}

const foo = new Foo();

const naiveBar = promisify(foo.bar);
// TypeError: Cannot read properties of undefined (reading 'a')
// naiveBar().then(a => console.log(a));

naiveBar.call(foo).then((a) => console.log(a)); // '42'

const bindBar = naiveBar.bind(foo);
bindBar().then((a) => console.log(a)); // '42'
```

```cjs
const { promisify } = require('node:util');

class Foo {
  constructor() {
    this.a = 42;
  }

  bar(callback) {
    callback(null, this.a);
  }
}

const foo = new Foo();

const naiveBar = promisify(foo.bar);
// TypeError: Cannot read properties of undefined (reading 'a')
// naiveBar().then(a => console.log(a));

naiveBar.call(foo).then((a) => console.log(a)); // '42'

const bindBar = naiveBar.bind(foo);
bindBar().then((a) => console.log(a)); // '42'
```

### 自定义 promise 化函数

使用 `util.promisify.custom` 符号可以覆盖 [`util.promisify()`][] 的返回值：

```mjs
import { promisify } from 'node:util';

function doSomething(foo, callback) {
  // ...
}

doSomething[promisify.custom] = (foo) => {
  return getPromiseSomehow();
};

const promisified = promisify(doSomething);
console.log(promisified === doSomething[promisify.custom]);
// 打印 'true'
```

```cjs
const { promisify } = require('node:util');

function doSomething(foo, callback) {
  // ...
}

doSomething[promisify.custom] = (foo) => {
  return getPromiseSomehow();
};

const promisified = promisify(doSomething);
console.log(promisified === doSomething[promisify.custom]);
// 打印 'true'
```

这对于原始函数不遵循将错误优先回调作为最后一个参数的标准格式的情况非常有用。

例如，对于接受 `(foo, onSuccessCallback, onErrorCallback)` 的函数：

```js
doSomething[util.promisify.custom] = (foo) => {
  return new Promise((resolve, reject) => {
    doSomething(foo, resolve, reject);
  });
};
```

如果 `promisify.custom` 被定义但不是函数，`promisify()` 将抛出错误。

### `util.promisify.custom`

<!-- YAML
added: v8.0.0
changes:
  - version:
      - v13.12.0
      - v12.16.2
    pr-url: https://github.com/nodejs/node/pull/31672
    description: This is now defined as a shared symbol.
-->

- 类型: {symbol} 可用于声明函数的自定义 promise 化变体的符号，请参阅 [自定义 promise 化函数][]。

除了可以通过 `util.promisify.custom` 访问外，此符号还在 [全局符号注册表][global symbol registry] 中注册，可以在任何环境中作为 `Symbol.for('nodejs.util.promisify.custom')` 访问。

例如，对于接受 `(foo, onSuccessCallback, onErrorCallback)` 的函数：

```js
const kCustomPromisifiedSymbol = Symbol.for('nodejs.util.promisify.custom');

doSomething[kCustomPromisifiedSymbol] = (foo) => {
  return new Promise((resolve, reject) => {
    doSomething(foo, resolve, reject);
  });
};
```

## `util.stripVTControlCharacters(str)`

<!-- YAML
added: v16.11.0
-->

- `str` {string}
- 返回: {string}

返回删除了所有 ANSI 转义码的 `str`。

```js
console.log(util.stripVTControlCharacters('\u001B[4mvalue\u001B[0m'));
// 打印 "value"
```

## `util.styleText(format, text[, options])`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
changes:
  - version: v24.2.0
    pr-url: https://github.com/nodejs/node/pull/58437
    description: Added the `'none'` format as a non-op format.
  - version:
    - v23.5.0
    - v22.13.0
    pr-url: https://github.com/nodejs/node/pull/56265
    description: styleText is now stable.
  - version:
    - v22.8.0
    - v20.18.0
    pr-url: https://github.com/nodejs/node/pull/54389
    description: Respect isTTY and environment variables
      such as NO_COLOR, NODE_DISABLE_COLORS, and FORCE_COLOR.
-->

- `format` {string | Array} `util.inspect.colors` 中定义的文本格式或文本格式数组。
- `text` {string} 要格式化的文本。
- `options` {Object}
  - `validateStream` {boolean} 当为 true 时，检查 `stream` 是否可以处理颜色。**默认值:** `true`。
  - `stream` {Stream} 将验证其是否可以着色的流。**默认值:** `process.stdout`。

此函数返回考虑传递的 `format` 的格式化文本，以便在终端中打印。它知道终端的能力，并根据通过 `NO_COLOR`、`NODE_DISABLE_COLORS` 和 `FORCE_COLOR` 环境变量设置的配置进行操作。

```mjs
import { styleText } from 'node:util';
import { stderr } from 'node:process';

const successMessage = styleText('green', 'Success!');
console.log(successMessage);

const errorMessage = styleText(
  'red',
  'Error! Error!',
  // 验证 process.stderr 是否有 TTY
  { stream: stderr }
);
console.error(errorMessage);
```

```cjs
const { styleText } = require('node:util');
const { stderr } = require('node:process');

const successMessage = styleText('green', 'Success!');
console.log(successMessage);

const errorMessage = styleText(
  'red',
  'Error! Error!',
  // 验证 process.stderr 是否有 TTY
  { stream: stderr }
);
console.error(errorMessage);
```

`util.inspect.colors` 还提供了文本格式，如 `italic` 和 `underline`，你可以同时使用两者：

```cjs
console.log(
  util.styleText(['underline', 'italic'], 'My italic underlined message')
);
```

当传递格式数组时，应用的格式顺序是从左到右，因此后续的格式可能会覆盖前一个。

```cjs
console.log(
  util.styleText(['red', 'green'], 'text') // green
);
```

特殊格式值 `none` 不对文本应用任何额外的样式。

格式的完整列表可以在 [修饰符][] 中找到。

## 类：`util.TextDecoder`

<!-- YAML
added: v8.3.0
changes:
  - version: v11.0.0
    pr-url: https://github.com/nodejs/node/pull/22281
    description: The class is now available on the global object.
-->

[WHATWG 编码标准][] `TextDecoder` API 的一个实现。

```js
const decoder = new TextDecoder();
const u8arr = new Uint8Array([72, 101, 108, 108, 111]);
console.log(decoder.decode(u8arr)); // Hello
```

### WHATWG 支持的编码

根据 [WHATWG 编码标准][]，`TextDecoder` API 支持的编码如下表所述。对于每种编码，可以使用一个或多个别名。

不同的 Node.js 构建配置支持不同的编码集。（参见 [国际化][]）

#### 默认支持的编码（具有完整的 ICU 数据）

| 编码               | 别名                                                                                                                                                                                                                                |
| ------------------ | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `'ibm866'`         | `'866'`、`'cp866'`、`'csibm866'`                                                                                                                                                                                                    |
| `'iso-8859-2'`     | `'csisolatin2'`、`'iso-ir-101'`、`'iso8859-2'`、`'iso88592'`、`'iso_8859-2'`、`'iso_8859-2:1987'`、`'l2'`、`'latin2'`                                                                                                               |
| `'iso-8859-3'`     | `'csisolatin3'`、`'iso-ir-109'`、`'iso8859-3'`、`'iso88593'`、`'iso_8859-3'`、`'iso_8859-3:1988'`、`'l3'`、`'latin3'`                                                                                                               |
| `'iso-8859-4'`     | `'csisolatin4'`、`'iso-ir-110'`、`'iso8859-4'`、`'iso88594'`、`'iso_8859-4'`、`'iso_8859-4:1988'`、`'l4'`、`'latin4'`                                                                                                               |
| `'iso-8859-5'`     | `'csisolatincyrillic'`、`'cyrillic'`、`'iso-ir-144'`、`'iso8859-5'`、`'iso88595'`、`'iso_8859-5'`、`'iso_8859-5:1988'`                                                                                                              |
| `'iso-8859-6'`     | `'arabic'`、`'asmo-708'`、`'csiso88596e'`、`'csiso88596i'`、`'csisolatinarabic'`、`'ecma-114'`、`'iso-8859-6-e'`、`'iso-8859-6-i'`、`'iso-ir-127'`、`'iso8859-6'`、`'iso88596'`、`'iso_8859-6'`、`'iso_8859-6:1987'`                |
| `'iso-8859-7'`     | `'csisolatingreek'`、`'ecma-118'`、`'elot_928'`、`'greek'`、`'greek8'`、`'iso-ir-126'`、`'iso8859-7'`、`'iso88597'`、`'iso_8859-7'`、`'iso_8859-7:1987'`、`'sun_eu_greek'`                                                          |
| `'iso-8859-8'`     | `'csiso88598e'`、`'csisolatinhebrew'`、`'hebrew'`、`'iso-8859-8-e'`、`'iso-ir-138'`、`'iso8859-8'`、`'iso88598'`、`'iso_8859-8'`、`'iso_8859-8:1988'`、`'visual'`                                                                   |
| `'iso-8859-8-i'`   | `'csiso88598i'`、`'logical'`                                                                                                                                                                                                        |
| `'iso-8859-10'`    | `'csisolatin6'`、`'iso-ir-157'`、`'iso8859-10'`、`'iso885910'`、`'l6'`、`'latin6'`                                                                                                                                                  |
| `'iso-8859-13'`    | `'iso8859-13'`、`'iso885913'`                                                                                                                                                                                                       |
| `'iso-8859-14'`    | `'iso8859-14'`、`'iso885914'`                                                                                                                                                                                                       |
| `'iso-8859-15'`    | `'csisolatin9'`、`'iso8859-15'`、`'iso885915'`、`'iso_8859-15'`、`'l9'`                                                                                                                                                             |
| `'koi8-r'`         | `'cskoi8r'`、`'koi'`、`'koi8'`、`'koi8_r'`                                                                                                                                                                                          |
| `'koi8-u'`         | `'koi8-ru'`                                                                                                                                                                                                                         |
| `'macintosh'`      | `'csmacintosh'`、`'mac'`、`'x-mac-roman'`                                                                                                                                                                                           |
| `'windows-874'`    | `'dos-874'`、`'iso-8859-11'`、`'iso8859-11'`、`'iso885911'`、`'tis-620'`                                                                                                                                                            |
| `'windows-1250'`   | `'cp1250'`、`'x-cp1250'`                                                                                                                                                                                                            |
| `'windows-1251'`   | `'cp1251'`、`'x-cp1251'`                                                                                                                                                                                                            |
| `'windows-1252'`   | `'ansi_x3.4-1968'`、`'ascii'`、`'cp1252'`、`'cp819'`、`'csisolatin1'`、`'ibm819'`、`'iso-8859-1'`、`'iso-ir-100'`、`'iso8859-1'`、`'iso88591'`、`'iso_8859-1'`、`'iso_8859-1:1987'`、`'l1'`、`'latin1'`、`'us-ascii'`、`'x-cp1252'` |
| `'windows-1253'`   | `'cp1253'`、`'x-cp1253'`                                                                                                                                                                                                            |
| `'windows-1254'`   | `'cp1254'`、`'csisolatin5'`、`'iso-8859-9'`、`'iso-ir-148'`、`'iso8859-9'`、`'iso88599'`、`'iso_8859-9'`、`'iso_8859-9:1989'`、`'l5'`、`'latin5'`、`'x-cp1254'`                                                                     |
| `'windows-1255'`   | `'cp1255'`、`'x-cp1255'`                                                                                                                                                                                                            |
| `'windows-1256'`   | `'cp1256'`、`'x-cp1256'`                                                                                                                                                                                                            |
| `'windows-1257'`   | `'cp1257'`、`'x-cp1257'`                                                                                                                                                                                                            |
| `'windows-1258'`   | `'cp1258'`、`'x-cp1258'`                                                                                                                                                                                                            |
| `'x-mac-cyrillic'` | `'x-mac-ukrainian'`                                                                                                                                                                                                                 |
| `'gbk'`            | `'chinese'`、`'csgb2312'`、`'csiso58gb231280'`、`'gb2312'`、`'gb_2312'`、`'gb_2312-80'`、`'iso-ir-58'`、`'x-gbk'`                                                                                                                   |
| `'gb18030'`        |                                                                                                                                                                                                                                     |
| `'big5'`           | `'big5-hkscs'`、`'cn-big5'`、`'csbig5'`、`'x-x-big5'`                                                                                                                                                                               |
| `'euc-jp'`         | `'cseucpkdfmtjapanese'`、`'x-euc-jp'`                                                                                                                                                                                               |
| `'iso-2022-jp'`    | `'csiso2022jp'`                                                                                                                                                                                                                     |
| `'shift_jis'`      | `'csshiftjis'`、`'ms932'`、`'ms_kanji'`、`'shift-jis'`、`'sjis'`、`'windows-31j'`、`'x-sjis'`                                                                                                                                       |
| `'euc-kr'`         | `'cseuckr'`、`'csksc56011987'`、`'iso-ir-149'`、`'korean'`、`'ks_c_5601-1987'`、`'ks_c_5601-1989'`、`'ksc5601'`、`'ksc_5601'`、`'windows-949'`                                                                                      |

#### 当 Node.js 使用 `small-icu` 选项构建时支持的编码

| 编码         | 别名                            |
| ------------ | ------------------------------- |
| `'utf-8'`    | `'unicode-1-1-utf-8'`、`'utf8'` |
| `'utf-16le'` | `'utf-16'`                      |
| `'utf-16be'` |                                 |

#### 当 ICU 被禁用时支持的编码

| 编码         | 别名                            |
| ------------ | ------------------------------- |
| `'utf-8'`    | `'unicode-1-1-utf-8'`、`'utf8'` |
| `'utf-16le'` | `'utf-16'`                      |

[WHATWG 编码标准][] 中列出的 `'iso-8859-16'` 编码不受支持。

### `new TextDecoder([encoding[, options]])`

- `encoding` {string} 标识此 `TextDecoder` 实例支持的 `encoding`。**默认值:** `'utf-8'`。
- `options` {Object}
  - `fatal` {boolean} `true` 表示解码失败是致命的。
    当 ICU 被禁用时，此选项不受支持（参见 [国际化][]）。**默认值:** `false`。
  - `ignoreBOM` {boolean} 当为 `true` 时，`TextDecoder` 将在解码结果中包含字节顺序标记。当为 `false` 时，字节顺序标记将从输出中移除。此选项仅在 `encoding` 为 `'utf-8'`、`'utf-16be'` 或 `'utf-16le'` 时使用。**默认值:** `false`。

创建一个新的 `TextDecoder` 实例。`encoding` 可以指定一种支持的编码或别名。

`TextDecoder` 类在全局对象上也可用。

### `textDecoder.decode([input[, options]])`

- `input` {ArrayBuffer|DataView|TypedArray} 包含编码数据的 `ArrayBuffer`、`DataView` 或 `TypedArray` 实例。
- `options` {Object}
  - `stream` {boolean} `true` 表示期望有额外的数据块。**默认值:** `false`。
- 返回: {string}

解码 `input` 并返回一个字符串。如果 `options.stream` 为 `true`，则发生在 `input` 末尾的任何不完整的字节序列将在内部缓冲，并在下次调用 `textDecoder.decode()` 后发出。

如果 `textDecoder.fatal` 为 `true`，发生的解码错误将导致抛出 `TypeError`。

### `textDecoder.encoding`

- 类型: {string}

`TextDecoder` 实例支持的编码。

### `textDecoder.fatal`

- 类型: {boolean}

如果解码错误导致抛出 `TypeError`，则该值为 `true`。

### `textDecoder.ignoreBOM`

- 类型: {boolean}

如果解码结果将包含字节顺序标记，则该值为 `true`。

## 类：`util.TextEncoder`

<!-- YAML
added: v8.3.0
changes:
  - version: v11.0.0
    pr-url: https://github.com/nodejs/node/pull/22281
    description: The class is now available on the global object.
-->

[WHATWG 编码标准][] `TextEncoder` API 的一个实现。所有 `TextEncoder` 实例仅支持 UTF-8 编码。

```js
const encoder = new TextEncoder();
const uint8array = encoder.encode('this is some data');
```

`TextEncoder` 类在全局对象上也可用。

### `textEncoder.encode([input])`

- `input` {string} 要编码的文本。**默认值:** 空字符串。
- 返回: {Uint8Array}

将 `input` 字符串进行 UTF-8 编码，并返回一个包含编码字节的 `Uint8Array`。

### `textEncoder.encodeInto(src, dest)`

<!-- YAML
added: v12.11.0
-->

- `src` {string} 要编码的文本。
- `dest` {Uint8Array} 用于保存编码结果的数组。
- 返回: {Object}
  - `read` {number} 读取的 src 的 Unicode 代码单元数。
  - `written` {number} 写入 dest 的 UTF-8 字节数。

将 `src` 字符串进行 UTF-8 编码到 `dest` Uint8Array 中，并返回一个包含读取的 Unicode 代码单元数和写入的 UTF-8 字节数的对象。

```js
const encoder = new TextEncoder();
const src = 'this is some data';
const dest = new Uint8Array(10);
const { read, written } = encoder.encodeInto(src, dest);
```

### `textEncoder.encoding`

- 类型: {string}

`TextEncoder` 实例支持的编码。始终设置为 `'utf-8'`。

## `util.toUSVString(string)`

<!-- YAML
added:
  - v16.8.0
  - v14.18.0
-->

- `string` {string}

在将任何代理代码点（或等效地，任何未配对的代理代码单元）替换为 Unicode "替换字符" U+FFFD 后返回 `string`。

## `util.transferableAbortController()`

<!-- YAML
added: v18.11.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

创建并返回一个 {AbortController} 实例，其 {AbortSignal} 被标记为可转移，并可与 `structuredClone()` 或 `postMessage()` 一起使用。

## `util.transferableAbortSignal(signal)`

<!-- YAML
added: v18.11.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

- `signal` {AbortSignal}
- 返回: {AbortSignal}

将给定的 {AbortSignal} 标记为可转移，以便它可以与 `structuredClone()` 和 `postMessage()` 一起使用。

```js
const signal = transferableAbortSignal(AbortSignal.timeout(100));
const channel = new MessageChannel();
channel.port2.postMessage(signal, [signal]);
```

## `util.aborted(signal, resource)`

<!-- YAML
added:
 - v19.7.0
 - v18.16.0
changes:
 - version: v24.0.0
   pr-url: https://github.com/nodejs/node/pull/57765
   description: Change stability index for this feature from Experimental to Stable.
-->

- `signal` {AbortSignal}
- `resource` {Object} 任何与可中止操作关联的非空对象，并被弱持有。
  如果在 `signal` 中止之前 `resource` 被垃圾回收，则 promise 保持挂起状态，允许 Node.js 停止跟踪它。
  这有助于防止长时间运行或不可取消操作中的内存泄漏。
- 返回: {Promise}

监听提供的 `signal` 上的中止事件，并返回一个在 `signal` 被中止时解决的 promise。
如果提供了 `resource`，它会弱引用操作的关联对象，
因此如果在 `signal` 中止之前 `resource` 被垃圾回收，
则返回的 promise 将保持挂起状态。
这可以防止长时间运行或不可取消操作中的内存泄漏。

```cjs
const { aborted } = require('node:util');

// 获取一个具有可中止信号的对象，例如自定义资源或操作。
const dependent = obtainSomethingAbortable();

// 将 `dependent` 作为资源传递，指示 promise 仅当 `dependent` 在信号中止时仍在内存中时才应解决。
aborted(dependent.signal, dependent).then(() => {
  // 当 `dependent` 被中止时，此代码运行。
  console.log('Dependent resource was aborted.');
});

// 模拟触发中止的事件。
dependent.on('event', () => {
  dependent.abort(); // 这将导致 `aborted` promise 解决。
});
```

```mjs
import { aborted } from 'node:util';

// 获取一个具有可中止信号的对象，例如自定义资源或操作。
const dependent = obtainSomethingAbortable();

// 将 `dependent` 作为资源传递，指示 promise 仅当 `dependent` 在信号中止时仍在内存中时才应解决。
aborted(dependent.signal, dependent).then(() => {
  // 当 `dependent` 被中止时，此代码运行。
  console.log('Dependent resource was aborted.');
});

// 模拟触发中止的事件。
dependent.on('event', () => {
  dependent.abort(); // 这将导致 `aborted` promise 解决。
});
```

## `util.types`

<!-- YAML
added: v10.0.0
changes:
  - version: v15.3.0
    pr-url: https://github.com/nodejs/node/pull/34055
    description: Exposed as `require('util/types')`.
-->

`util.types` 为不同类型的内置对象提供类型检查。与 `instanceof` 或 `Object.prototype.toString.call(value)` 不同，这些检查不检查从 JavaScript 可访问的对象属性（如它们的原型），并且通常具有调用 C++ 的开销。

结果通常不保证值在 JavaScript 中公开哪些属性或行为。它们主要对偏好用 JavaScript 进行类型检查的插件开发者有用。

该 API 可通过 `require('node:util').types` 或 `require('node:util/types')` 访问。

### `util.types.isAnyArrayBuffer(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {ArrayBuffer} 或 {SharedArrayBuffer} 实例，则返回 `true`。

另请参阅 [`util.types.isArrayBuffer()`][] 和 [`util.types.isSharedArrayBuffer()`][]。

```js
util.types.isAnyArrayBuffer(new ArrayBuffer()); // 返回 true
util.types.isAnyArrayBuffer(new SharedArrayBuffer()); // 返回 true
```

### `util.types.isArrayBufferView(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是 {ArrayBuffer} 视图之一（例如类型化数组对象或 {DataView}）的实例，则返回 `true`。等价于 [`ArrayBuffer.isView()`][]。

```js
util.types.isArrayBufferView(new Int8Array()); // true
util.types.isArrayBufferView(Buffer.from('hello world')); // true
util.types.isArrayBufferView(new DataView(new ArrayBuffer(16))); // true
util.types.isArrayBufferView(new ArrayBuffer()); // false
```

### `util.types.isArgumentsObject(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是 `arguments` 对象，则返回 `true`。

<!-- eslint-disable prefer-rest-params -->

```js
function foo() {
  util.types.isArgumentsObject(arguments); // 返回 true
}
```

### `util.types.isArrayBuffer(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {ArrayBuffer} 实例，则返回 `true`。
这不包括 {SharedArrayBuffer} 实例。通常，需要同时测试两者；有关此情况，请参阅 [`util.types.isAnyArrayBuffer()`][]。

```js
util.types.isArrayBuffer(new ArrayBuffer()); // 返回 true
util.types.isArrayBuffer(new SharedArrayBuffer()); // 返回 false
```

### `util.types.isAsyncFunction(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是 [异步函数][]，则返回 `true`。
这仅报告 JavaScript 引擎所看到的内容；特别是，返回值可能与原始源代码不匹配，如果使用了转译工具。

```js
util.types.isAsyncFunction(function foo() {}); // 返回 false
util.types.isAsyncFunction(async function foo() {}); // 返回 true
```

### `util.types.isBigInt64Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是 `BigInt64Array` 实例，则返回 `true`。

```js
util.types.isBigInt64Array(new BigInt64Array()); // 返回 true
util.types.isBigInt64Array(new BigUint64Array()); // 返回 false
```

### `util.types.isBigIntObject(value)`

<!-- YAML
added: v10.4.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是 BigInt 对象，例如由 `Object(BigInt(123))` 创建，则返回 `true`。

```js
util.types.isBigIntObject(Object(BigInt(123))); // 返回 true
util.types.isBigIntObject(BigInt(123)); // 返回 false
util.types.isBigIntObject(123); // 返回 false
```

### `util.types.isBigUint64Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是 `BigUint64Array` 实例，则返回 `true`。

```js
util.types.isBigUint64Array(new BigInt64Array()); // 返回 false
util.types.isBigUint64Array(new BigUint64Array()); // 返回 true
```

### `util.types.isBooleanObject(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是布尔对象，例如由 `new Boolean()` 创建，则返回 `true`。

```js
util.types.isBooleanObject(false); // 返回 false
util.types.isBooleanObject(true); // 返回 false
util.types.isBooleanObject(new Boolean(false)); // 返回 true
util.types.isBooleanObject(new Boolean(true)); // 返回 true
util.types.isBooleanObject(Boolean(false)); // 返回 false
util.types.isBooleanObject(Boolean(true)); // 返回 false
```

### `util.types.isBoxedPrimitive(value)`

<!-- YAML
added: v10.11.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是任何被包装的原始对象，例如由 `new Boolean()`、`new String()` 或 `Object(Symbol())` 创建，则返回 `true`。

例如：

```js
util.types.isBoxedPrimitive(false); // 返回 false
util.types.isBoxedPrimitive(new Boolean(false)); // 返回 true
util.types.isBoxedPrimitive(Symbol('foo')); // 返回 false
util.types.isBoxedPrimitive(Object(Symbol('foo'))); // 返回 true
util.types.isBoxedPrimitive(Object(BigInt(5))); // 返回 true
```

### `util.types.isCryptoKey(value)`

<!-- YAML
added: v16.2.0
-->

- `value` {Object}
- 返回: {boolean}

如果 `value` 是 {CryptoKey}，则返回 `true`，否则返回 `false`。

### `util.types.isDataView(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {DataView} 实例，则返回 `true`。

```js
const ab = new ArrayBuffer(20);
util.types.isDataView(new DataView(ab)); // 返回 true
util.types.isDataView(new Float64Array()); // 返回 false
```

另请参阅 [`ArrayBuffer.isView()`][]。

### `util.types.isDate(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Date} 实例，则返回 `true`。

```js
util.types.isDate(new Date()); // 返回 true
```

### `util.types.isExternal(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是原生 `External` 值，则返回 `true`。

原生 `External` 值是一种特殊类型的对象，包含一个原始 C++ 指针（`void*`）供原生代码访问，并且没有其他属性。此类对象由 Node.js 内部或原生插件创建。在 JavaScript 中，它们是 [冻结的][`Object.freeze()`] 具有 `null` 原型的对象。

```c
#include <js_native_api.h>
#include <stdlib.h>
napi_value result;
static napi_value MyNapi(napi_env env, napi_callback_info info) {
  int* raw = (int*) malloc(1024);
  napi_status status = napi_create_external(env, (void*) raw, NULL, NULL, &result);
  if (status != napi_ok) {
    napi_throw_error(env, NULL, "napi_create_external failed");
    return NULL;
  }
  return result;
}
...
DECLARE_NAPI_PROPERTY("myNapi", MyNapi)
...
```

```mjs
import native from 'napi_addon.node';
import { types } from 'node:util';

const data = native.myNapi();
types.isExternal(data); // 返回 true
types.isExternal(0); // 返回 false
types.isExternal(new String('foo')); // 返回 false
```

```cjs
const native = require('napi_addon.node');
const { types } = require('node:util');

const data = native.myNapi();
types.isExternal(data); // 返回 true
types.isExternal(0); // 返回 false
types.isExternal(new String('foo')); // 返回 false
```

有关 `napi_create_external` 的更多信息，请参阅 [`napi_create_external()`][]。

### `util.types.isFloat16Array(value)`

<!-- YAML
added: v24.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Float16Array} 实例，则返回 `true`。

```js
util.types.isFloat16Array(new ArrayBuffer()); // 返回 false
util.types.isFloat16Array(new Float16Array()); // 返回 true
util.types.isFloat16Array(new Float32Array()); // 返回 false
```

### `util.types.isFloat32Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Float32Array} 实例，则返回 `true`。

```js
util.types.isFloat32Array(new ArrayBuffer()); // 返回 false
util.types.isFloat32Array(new Float32Array()); // 返回 true
util.types.isFloat32Array(new Float64Array()); // 返回 false
```

### `util.types.isFloat64Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Float64Array} 实例，则返回 `true`。

```js
util.types.isFloat64Array(new ArrayBuffer()); // 返回 false
util.types.isFloat64Array(new Uint8Array()); // 返回 false
util.types.isFloat64Array(new Float64Array()); // 返回 true
```

### `util.types.isGeneratorFunction(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是生成器函数，则返回 `true`。
这仅报告 JavaScript 引擎所看到的内容；特别是，返回值可能与原始源代码不匹配，如果使用了转译工具。

```js
util.types.isGeneratorFunction(function foo() {}); // 返回 false
util.types.isGeneratorFunction(function* foo() {}); // 返回 true
```

### `util.types.isGeneratorObject(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是从内置生成器函数返回的生成器对象，则返回 `true`。
这仅报告 JavaScript 引擎所看到的内容；特别是，返回值可能与原始源代码不匹配，如果使用了转译工具。

```js
function* foo() {}
const generator = foo();
util.types.isGeneratorObject(generator); // 返回 true
```

### `util.types.isInt8Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Int8Array} 实例，则返回 `true`。

```js
util.types.isInt8Array(new ArrayBuffer()); // 返回 false
util.types.isInt8Array(new Int8Array()); // 返回 true
util.types.isInt8Array(new Float64Array()); // 返回 false
```

### `util.types.isInt16Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Int16Array} 实例，则返回 `true`。

```js
util.types.isInt16Array(new ArrayBuffer()); // 返回 false
util.types.isInt16Array(new Int16Array()); // 返回 true
util.types.isInt16Array(new Float64Array()); // 返回 false
```

### `util.types.isInt32Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Int32Array} 实例，则返回 `true`。

```js
util.types.isInt32Array(new ArrayBuffer()); // 返回 false
util.types.isInt32Array(new Int32Array()); // 返回 true
util.types.isInt32Array(new Float64Array()); // 返回 false
```

### `util.types.isKeyObject(value)`

<!-- YAML
added: v16.2.0
-->

- `value` {Object}
- 返回: {boolean}

如果 `value` 是 {KeyObject}，则返回 `true`，否则返回 `false`。

### `util.types.isMap(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Map} 实例，则返回 `true`。

```js
util.types.isMap(new Map()); // 返回 true
```

### `util.types.isMapIterator(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是为内置 {Map} 实例返回的迭代器，则返回 `true`。

```js
const map = new Map();
util.types.isMapIterator(map.keys()); // 返回 true
util.types.isMapIterator(map.values()); // 返回 true
util.types.isMapIterator(map.entries()); // 返回 true
util.types.isMapIterator(map[Symbol.iterator]()); // 返回 true
```

### `util.types.isModuleNamespaceObject(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是 [模块命名空间对象][] 的实例，则返回 `true`。

```mjs
import * as ns from './a.js';

util.types.isModuleNamespaceObject(ns); // 返回 true
```

### `util.types.isNativeError(value)`

<!-- YAML
added: v10.0.0
deprecated: v24.2.0
-->

> Stability: 0 - Deprecated: 改用 [`Error.isError`][]。

**注意：** 从 Node.js v24 开始，`Error.isError()` 目前比 `util.types.isNativeError()` 慢。
如果性能至关重要，请考虑在你的环境中对两者进行基准测试。

- `value` {any}
- 返回: {boolean}

如果该值是由 [内置 `Error` 类型][] 的构造函数返回的，则返回 `true`。

```js
console.log(util.types.isNativeError(new Error())); // true
console.log(util.types.isNativeError(new TypeError())); // true
console.log(util.types.isNativeError(new RangeError())); // true
```

原生错误类型的子类也是原生错误：

```js
class MyError extends Error {}
console.log(util.types.isNativeError(new MyError())); // true
```

一个值是原生错误类的 `instanceof` 并不等同于 `isNativeError()` 对该值返回 `true`。`isNativeError()` 对于来自不同 [领域][] 的错误返回 `true`，而 `instanceof Error` 对于这些错误返回 `false`：

```mjs
import { createContext, runInContext } from 'node:vm';
import { types } from 'node:util';

const context = createContext({});
const myError = runInContext('new Error()', context);
console.log(types.isNativeError(myError)); // true
console.log(myError instanceof Error); // false
```

```cjs
const { createContext, runInContext } = require('node:vm');
const { types } = require('node:util');

const context = createContext({});
const myError = runInContext('new Error()', context);
console.log(types.isNativeError(myError)); // true
console.log(myError instanceof Error); // false
```

相反，`isNativeError()` 对所有不是由原生错误构造函数返回的对象返回 `false`。这包括那些是原生错误 `instanceof` 的值：

```js
const myError = { __proto__: Error.prototype };
console.log(util.types.isNativeError(myError)); // false
console.log(myError instanceof Error); // true
```

### `util.types.isNumberObject(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是数字对象，例如由 `new Number()` 创建，则返回 `true`。

```js
util.types.isNumberObject(0); // 返回 false
util.types.isNumberObject(new Number(0)); // 返回 true
```

### `util.types.isPromise(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Promise}，则返回 `true`。

```js
util.types.isPromise(Promise.resolve(42)); // 返回 true
```

### `util.types.isProxy(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是 {Proxy} 实例，则返回 `true`。

```js
const target = {};
const proxy = new Proxy(target, {});
util.types.isProxy(target); // 返回 false
util.types.isProxy(proxy); // 返回 true
```

### `util.types.isRegExp(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是正则表达式对象，则返回 `true`。

```js
util.types.isRegExp(/abc/); // 返回 true
util.types.isRegExp(new RegExp('abc')); // 返回 true
```

### `util.types.isSet(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Set} 实例，则返回 `true`。

```js
util.types.isSet(new Set()); // 返回 true
```

### `util.types.isSetIterator(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是为内置 {Set} 实例返回的迭代器，则返回 `true`。

```js
const set = new Set();
util.types.isSetIterator(set.keys()); // 返回 true
util.types.isSetIterator(set.values()); // 返回 true
util.types.isSetIterator(set.entries()); // 返回 true
util.types.isSetIterator(set[Symbol.iterator]()); // 返回 true
```

### `util.types.isSharedArrayBuffer(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {SharedArrayBuffer} 实例，则返回 `true`。
这不包括 {ArrayBuffer} 实例。通常，需要同时测试两者；有关此情况，请参阅 [`util.types.isAnyArrayBuffer()`][]。

```js
util.types.isSharedArrayBuffer(new ArrayBuffer()); // 返回 false
util.types.isSharedArrayBuffer(new SharedArrayBuffer()); // 返回 true
```

### `util.types.isStringObject(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是字符串对象，例如由 `new String()` 创建，则返回 `true`。

```js
util.types.isStringObject('foo'); // 返回 false
util.types.isStringObject(new String('foo')); // 返回 true
```

### `util.types.isSymbolObject(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是符号对象，通过调用 `Symbol` 原始值上的 `Object()` 创建，则返回 `true`。

```js
const symbol = Symbol('foo');
util.types.isSymbolObject(symbol); // 返回 false
util.types.isSymbolObject(Object(symbol)); // 返回 true
```

### `util.types.isTypedArray(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {TypedArray} 实例，则返回 `true`。

```js
util.types.isTypedArray(new ArrayBuffer()); // 返回 false
util.types.isTypedArray(new Uint8Array()); // 返回 true
util.types.isTypedArray(new Float64Array()); // 返回 true
```

另请参阅 [`ArrayBuffer.isView()`][]。

### `util.types.isUint8Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Uint8Array} 实例，则返回 `true`。

```js
util.types.isUint8Array(new ArrayBuffer()); // 返回 false
util.types.isUint8Array(new Uint8Array()); // 返回 true
util.types.isUint8Array(new Float64Array()); // 返回 false
```

### `util.types.isUint8ClampedArray(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Uint8ClampedArray} 实例，则返回 `true`。

```js
util.types.isUint8ClampedArray(new ArrayBuffer()); // 返回 false
util.types.isUint8ClampedArray(new Uint8ClampedArray()); // 返回 true
util.types.isUint8ClampedArray(new Float64Array()); // 返回 false
```

### `util.types.isUint16Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Uint16Array} 实例，则返回 `true`。

```js
util.types.isUint16Array(new ArrayBuffer()); // 返回 false
util.types.isUint16Array(new Uint16Array()); // 返回 true
util.types.isUint16Array(new Float64Array()); // 返回 false
```

### `util.types.isUint32Array(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {Uint32Array} 实例，则返回 `true`。

```js
util.types.isUint32Array(new ArrayBuffer()); // 返回 false
util.types.isUint32Array(new Uint32Array()); // 返回 true
util.types.isUint32Array(new Float64Array()); // 返回 false
```

### `util.types.isWeakMap(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {WeakMap} 实例，则返回 `true`。

```js
util.types.isWeakMap(new WeakMap()); // 返回 true
```

### `util.types.isWeakSet(value)`

<!-- YAML
added: v10.0.0
-->

- `value` {any}
- 返回: {boolean}

如果该值是内置的 {WeakSet} 实例，则返回 `true`。

```js
util.types.isWeakSet(new WeakSet()); // 返回 true
```

## 已弃用的 API

以下 API 已被弃用，不应再使用。现有应用程序和模块应更新以找到替代方法。

### `util._extend(target, source)`

<!-- YAML
added: v0.7.5
deprecated: v6.0.0
-->

> Stability: 0 - Deprecated: 改用 [`Object.assign()`][]。

- `target` {Object}
- `source` {Object}

`util._extend()` 方法从未打算在 Node.js 内部模块之外使用。社区发现并使用了它。

它已被弃用，不应在新代码中使用。JavaScript 通过 [`Object.assign()`][] 提供了非常相似的内置功能。

### `util.isArray(object)`

<!-- YAML
added: v0.6.0
deprecated: v4.0.0
-->

> Stability: 0 - Deprecated: 改用 [`Array.isArray()`][]。

- `object` {any}
- 返回: {boolean}

[`Array.isArray()`][] 的别名。

如果给定的 `object` 是 `Array`，则返回 `true`。否则，返回 `false`。

```js
const util = require('node:util');

util.isArray([]);
// 返回: true
util.isArray(new Array());
// 返回: true
util.isArray({});
// 返回: false
```

[常见系统错误]: errors.md#common-system-errors
[对象上的自定义检查函数]: #custom-inspection-functions-on-objects
[自定义 promise 化函数]: #custom-promisified-functions
[自定义 `util.inspect` 颜色]: #customizing-utilinspect-colors
[国际化]: intl.md
[模块命名空间对象]: https://tc39.github.io/ecma262/#sec-module-namespace-exotic-objects
[WHATWG 编码标准]: https://encoding.spec.whatwg.org/
[`'uncaughtException'`]: process.md#event-uncaughtexception
[`'warning'`]: process.md#event-warning
[`Array.isArray()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/isArray
[`ArrayBuffer.isView()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer/isView
[`Error.isError`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Error/isError
[`JSON.stringify()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/JSON/stringify
[`MIMEparams`]: #class-utilmimeparams
[`Object.assign()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/assign
[`Object.freeze()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze
[`Runtime.ScriptId`]: https://chromedevtools.github.io/devtools-protocol/1-3/Runtime/#type-ScriptId
[`assert.deepStrictEqual()`]: assert.md#assertdeepstrictequalactual-expected-message
[`console.error()`]: console.md#consoleerrordata-args
[`mime.toString()`]: #mimetostring
[`mimeParams.entries()`]: #mimeparamsentries
[`napi_create_external()`]: n-api.md#napi_create_external
[`target` 和 `handler`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Proxy#Terminology
[`tty.hasColors()`]: tty.md#writestreamhascolorscount-env
[`util.diff()`]: #utildiffactual-expected
[`util.format()`]: #utilformatformat-args
[`util.inspect()`]: #utilinspectobject-options
[`util.promisify()`]: #utilpromisifyoriginal
[`util.types.isAnyArrayBuffer()`]: #utiltypesisanyarraybuffervalue
[`util.types.isArrayBuffer()`]: #utiltypesisarraybuffervalue
[`util.types.isSharedArrayBuffer()`]: #utiltypesissharedarraybuffervalue
[异步函数]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/async_function
[内置 `Error` 类型]: https://tc39.es/ecma262/#sec-error-objects
[比较函数]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort#Parameters
[构造函数]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/constructor
[默认排序]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Array/sort
[全局符号注册表]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Symbol/for
[已弃用 API 列表]: deprecations.md#list-of-deprecated-apis
[修饰符]: #modifiers
[领域]: https://tc39.es/ecma262/#realm
[语义不兼容的]: https://github.com/nodejs/node/issues/4179
[util.inspect.custom]: #utilinspectcustom
