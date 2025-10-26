# Readline

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/readline.js -->

`node:readline` 模块提供了一个接口，用于从 [Readable][] 流（例如 [`process.stdin`][]）一次一行地读取数据。

要使用基于 Promise 的 API：

```mjs
import * as readline from 'node:readline/promises';
```

```cjs
const readline = require('node:readline/promises');
```

要使用回调和同步 API：

```mjs
import * as readline from 'node:readline';
```

```cjs
const readline = require('node:readline');
```

以下简单示例说明了 `node:readline` 模块的基本用法。

```mjs
import * as readline from 'node:readline/promises';
import { stdin as input, stdout as output } from 'node:process';

const rl = readline.createInterface({ input, output });

const answer = await rl.question('What do you think of Node.js? ');

console.log(`Thank you for your valuable feedback: ${answer}`);

rl.close();
```

```cjs
const readline = require('node:readline');
const { stdin: input, stdout: output } = require('node:process');

const rl = readline.createInterface({ input, output });

rl.question('What do you think of Node.js? ', (answer) => {
  // TODO: Log the answer in a database
  console.log(`Thank you for your valuable feedback: ${answer}`);

  rl.close();
});
```

一旦调用此代码，Node.js 应用程序在 `readline.Interface` 关闭之前不会终止，因为接口会等待在 `input` 流上接收数据。

<a id='readline_class_interface'></a>

## 类：`InterfaceConstructor`

<!-- YAML
added: v0.1.104
-->

* 继承自：{EventEmitter}

`InterfaceConstructor` 类的实例是使用 `readlinePromises.createInterface()` 或 `readline.createInterface()` 方法构造的。每个实例都与一个 `input` [Readable][] 流和一个 `output` [Writable][] 流相关联。`output` 流用于打印用户输入的提示，该输入到达并从 `input` 流中读取。

### 事件：`'close'`

<!-- YAML
added: v0.1.98
-->

在以下情况之一发生时，会触发 `'close'` 事件：

* 调用 `rl.close()` 方法，并且 `InterfaceConstructor` 实例已放弃对 `input` 和 `output` 流的控制；
* `input` 流接收到其 `'end'` 事件；
* `input` 流接收 <kbd>Ctrl</kbd>+<kbd>D</kbd> 以发出传输结束（EOT）信号；
* `input` 流接收 <kbd>Ctrl</kbd>+<kbd>C</kbd> 以发出 `SIGINT` 信号，并且在 `InterfaceConstructor` 实例上未注册 `'SIGINT'` 事件监听器。

监听器函数被调用时不传递任何参数。

一旦发出 `'close'` 事件，`InterfaceConstructor` 实例就完成了。

### 事件：`'line'`

<!-- YAML
added: v0.1.98
-->

每当 `input` 流接收到行结束输入（`\n`、`\r` 或 `\r\n`）时，就会触发 `'line'` 事件。这通常发生在用户按下 <kbd>Enter</kbd> 或 <kbd>Return</kbd> 时。

如果从流中读取了新数据，并且该流在没有最终行结束标记的情况下结束，也会触发 `'line'` 事件。

监听器函数被调用时，会传入一个包含接收到的单行输入的字符串。

```js
rl.on('line', (input) => {
  console.log(`Received: ${input}`);
});
```

### 事件：`'history'`

<!-- YAML
added:
  - v15.8.0
  - v14.18.0
-->

每当历史数组发生变化时，就会触发 `'history'` 事件。

监听器函数被调用时，会传入一个包含历史数组的数组。它将反映所有更改，由于 `historySize` 和 `removeHistoryDuplicates` 而添加和删除的行。

主要目的是允许监听器持久化历史记录。监听器也可以更改历史对象。这对于防止某些行（如密码）被添加到历史记录中可能很有用。

```js
rl.on('history', (history) => {
  console.log(`Received: ${history}`);
});
```

### 事件：`'pause'`

<!-- YAML
added: v0.7.5
-->

在以下情况之一发生时，会触发 `'pause'` 事件：

* `input` 流被暂停。
* `input` 流未暂停并接收到 `'SIGCONT'` 事件。（参见事件 [`'SIGTSTP'`][] 和 [`'SIGCONT'`][]。）

监听器函数被调用时不传递任何参数。

```js
rl.on('pause', () => {
  console.log('Readline paused.');
});
```

### 事件：`'resume'`

<!-- YAML
added: v0.7.5
-->

每当 `input` 流恢复时，就会触发 `'resume'` 事件。

监听器函数被调用时不传递任何参数。

```js
rl.on('resume', () => {
  console.log('Readline resumed.');
});
```

### 事件：`'SIGCONT'`

<!-- YAML
added: v0.7.5
-->

当先前使用 <kbd>Ctrl</kbd>+<kbd>Z</kbd>（即 `SIGTSTP`）移动到后台的 Node.js 进程随后使用 fg(1p) 被带回前台时，会触发 `'SIGCONT'` 事件。

如果在 `SIGTSTP` 请求*之前* `input` 流已被暂停，则不会发出此事件。

监听器函数被调用时不传递任何参数。

```js
rl.on('SIGCONT', () => {
  // `prompt` 会自动恢复流
  rl.prompt();
});
```

Windows 上*不*支持 `'SIGCONT'` 事件。

### 事件：`'SIGINT'`

<!-- YAML
added: v0.3.0
-->

每当 `input` 流接收到 <kbd>Ctrl+C</kbd> 输入（通常称为 `SIGINT`）时，就会触发 `'SIGINT'` 事件。如果在 `input` 流接收到 `SIGINT` 时没有注册 `'SIGINT'` 事件监听器，则会发出 `'pause'` 事件。

监听器函数被调用时不传递任何参数。

```js
rl.on('SIGINT', () => {
  rl.question('Are you sure you want to exit? ', (answer) => {
    if (answer.match(/^y(es)?$/i)) rl.pause();
  });
});
```

### 事件：`'SIGTSTP'`

<!-- YAML
added: v0.7.5
-->

当 `input` 流接收到 <kbd>Ctrl</kbd>+<kbd>Z</kbd> 输入（通常称为 `SIGTSTP`）时，会触发 `'SIGTSTP'` 事件。如果在 `input` 流接收到 `SIGTSTP` 时没有注册 `'SIGTSTP'` 事件监听器，Node.js 进程将被发送到后台。

当使用 fg(1p) 恢复程序时，将发出 `'pause'` 和 `'SIGCONT'` 事件。这些可用于恢复 `input` 流。

如果在进程被发送到后台之前 `input` 已被暂停，则不会发出 `'pause'` 和 `'SIGCONT'` 事件。

监听器函数被调用时不传递任何参数。

```js
rl.on('SIGTSTP', () => {
  // 这将覆盖 SIGTSTP 并防止程序进入后台。
  console.log('Caught SIGTSTP.');
});
```

Windows 上*不*支持 `'SIGTSTP'` 事件。

### `rl.close()`

<!-- YAML
added: v0.1.98
-->

`rl.close()` 方法关闭 `InterfaceConstructor` 实例并放弃对 `input` 和 `output` 流的控制。调用时，将发出 `'close'` 事件。

调用 `rl.close()` 不会立即停止 `InterfaceConstructor` 实例发出其他事件（包括 `'line'`）。

### `rl[Symbol.dispose]()`

<!-- YAML
added:
  - v23.10.0
  - v22.15.0
-->

`rl.close()` 的别名。

### `rl.pause()`

<!-- YAML
added: v0.3.4
-->

`rl.pause()` 方法暂停 `input` 流，允许在必要时稍后恢复。

调用 `rl.pause()` 不会立即停止 `InterfaceConstructor` 实例发出其他事件（包括 `'line'`）。

### `rl.prompt([preserveCursor])`

<!-- YAML
added: v0.1.98
-->

* `preserveCursor` {boolean} 如果为 `true`，则防止光标位置重置为 `0`。

`rl.prompt()` 方法将 `InterfaceConstructor` 实例配置的 `prompt` 写入 `output` 中的新行，以便为用户提供新的输入位置。

调用时，如果 `input` 流已暂停，`rl.prompt()` 将恢复它。

如果创建 `InterfaceConstructor` 时将 `output` 设置为 `null` 或 `undefined`，则不会写入提示。

### `rl.resume()`

<!-- YAML
added: v0.3.4
-->

如果 `input` 流已暂停，`rl.resume()` 方法会恢复它。

### `rl.setPrompt(prompt)`

<!-- YAML
added: v0.1.98
-->

* `prompt` {string}

`rl.setPrompt()` 方法设置每当调用 `rl.prompt()` 时将写入 `output` 的提示。

### `rl.getPrompt()`

<!-- YAML
added:
  - v15.3.0
  - v14.17.0
-->

* 返回：{string} 当前提示字符串

`rl.getPrompt()` 方法返回 `rl.prompt()` 使用的当前提示。

### `rl.write(data[, key])`

<!-- YAML
added: v0.1.98
-->

* `data` {string}
* `key` {Object}
  * `ctrl` {boolean} `true` 表示 <kbd>Ctrl</kbd> 键。
  * `meta` {boolean} `true` 表示 <kbd>Meta</kbd> 键。
  * `shift` {boolean} `true` 表示 <kbd>Shift</kbd> 键。
  * `name` {string} 键的名称。

`rl.write()` 方法将 `data` 或由 `key` 标识的键序列写入 `output`。仅当 `output` 是 [TTY][] 文本终端时才支持 `key` 参数。有关键组合的列表，请参阅 [TTY 键绑定][]。

如果指定了 `key`，则忽略 `data`。

调用时，如果 `input` 流已暂停，`rl.write()` 将恢复它。

如果创建 `InterfaceConstructor` 时将 `output` 设置为 `null` 或 `undefined`，则不会写入 `data` 和 `key`。

```js
rl.write('Delete this!');
// 模拟 Ctrl+U 删除之前写入的行
rl.write(null, { ctrl: true, name: 'u' });
```

`rl.write()` 方法将数据写入 `readline` `Interface` 的 `input`，*就像由用户提供一样*。

### `rl[Symbol.asyncIterator]()`

<!-- YAML
added:
 - v11.4.0
 - v10.16.0
changes:
  - version:
     - v11.14.0
     - v10.17.0
    pr-url: https://github.com/nodejs/node/pull/26989
    description: Symbol.asyncIterator support is no longer experimental.
-->

* 返回：{AsyncIterator}

创建一个 `AsyncIterator` 对象，该对象将输入流中的每一行作为字符串进行迭代。此方法允许通过 `for await...of` 循环异步迭代 `InterfaceConstructor` 对象。

输入流中的错误不会被转发。

如果循环通过 `break`、`throw` 或 `return` 终止，将调用 [`rl.close()`][]。换句话说，迭代 `InterfaceConstructor` 将始终完全消耗输入流。

性能与传统的 `'line'` 事件 API 不相当。对于性能敏感的应用，请使用 `'line'`。

```js
async function processLineByLine() {
  const rl = readline.createInterface({
    // ...
  });

  for await (const line of rl) {
    // 读取输入中的每一行将在此处作为 `line` 依次可用。
  }
}
```

一旦调用，`readline.createInterface()` 将开始消耗输入流。在接口创建和异步迭代之间进行异步操作可能会导致遗漏行。

### `rl.line`

<!-- YAML
added: v0.1.98
changes:
  - version:
      - v15.8.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/33676
    description: Value will always be a string, never undefined.
-->

* 类型：{string}

当前由 node 处理的输入数据。

这可以在从 TTY 流收集输入时使用，以检索在 `line` 事件发出之前已处理到当前的值。一旦发出 `line` 事件，此属性将为空字符串。

请注意，如果在实例运行时修改该值，并且没有同时控制 `rl.cursor`，可能会产生意外后果。

**如果不使用 TTY 流进行输入，请使用 [`'line'`][] 事件。**

一个可能的使用案例如下：

```js
const values = ['lorem ipsum', 'dolor sit amet'];
const rl = readline.createInterface(process.stdin);
const showResults = debounce(() => {
  console.log(
    '\n',
    values.filter((val) => val.startsWith(rl.line)).join(' '),
  );
}, 300);
process.stdin.on('keypress', (c, k) => {
  showResults();
});
```

### `rl.cursor`

<!-- YAML
added: v0.1.98
-->

* 类型：{number|undefined}

光标相对于 `rl.line` 的位置。

当从 TTY 流读取输入时，这将跟踪当前光标在输入字符串中的位置。光标的位置决定了在处理输入时将修改输入字符串的哪部分，以及终端插入符号将呈现的列。

### `rl.getCursorPos()`

<!-- YAML
added:
 - v13.5.0
 - v12.16.0
-->

* 返回：{Object}
  * `rows` {number} 光标当前所在提示符的行
  * `cols` {number} 光标当前所在的屏幕列

返回光标相对于输入提示符 + 字符串的实际位置。长输入（换行）字符串以及多行提示符都包含在计算中。

## Promises API

<!-- YAML
added: v17.0.0
changes:
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57513
    description: Marking the API stable.
-->

### 类：`readlinePromises.Interface`

<!-- YAML
added: v17.0.0
-->

* 继承自：{readline.InterfaceConstructor}

`readlinePromises.Interface` 类的实例是使用 `readlinePromises.createInterface()` 方法构造的。每个实例都与一个 `input` [Readable][] 流和一个 `output` [Writable][] 流相关联。`output` 流用于打印用户输入的提示，该输入到达并从 `input` 流中读取。

#### `rl.question(query[, options])`

<!-- YAML
added: v17.0.0
-->

* `query` {string} 要写入 `output` 的语句或查询，前置到提示符。
* `options` {Object}
  * `signal` {AbortSignal} 可选地允许使用 `AbortSignal` 取消 `question()`。
* 返回：{Promise} 一个 Promise，使用用户对 `query` 的输入来兑现。

`rl.question()` 方法通过将 `query` 写入 `output` 来显示它，等待用户在 `input` 上提供输入，然后调用 `callback` 函数，将提供的输入作为第一个参数传递。

调用时，如果 `input` 流已暂停，`rl.question()` 将恢复它。

如果创建 `readlinePromises.Interface` 时将 `output` 设置为 `null` 或 `undefined`，则不会写入 `query`。

如果在 `rl.close()` 之后调用问题，它将返回一个被拒绝的 Promise。

用法示例：

```mjs
const answer = await rl.question('What is your favorite food? ');
console.log(`Oh, so your favorite food is ${answer}`);
```

使用 `AbortSignal` 取消问题。

```mjs
const signal = AbortSignal.timeout(10_000);

signal.addEventListener('abort', () => {
  console.log('The food question timed out');
}, { once: true });

const answer = await rl.question('What is your favorite food? ', { signal });
console.log(`Oh, so your favorite food is ${answer}`);
```

### 类：`readlinePromises.Readline`

<!-- YAML
added: v17.0.0
-->

#### `new readlinePromises.Readline(stream[, options])`

<!-- YAML
added: v17.0.0
-->

* `stream` {stream.Writable} 一个 [TTY][] 流。
* `options` {Object}
  * `autoCommit` {boolean} 如果为 `true`，则无需调用 `rl.commit()`。

#### `rl.clearLine(dir)`

<!-- YAML
added: v17.0.0
-->

* `dir` {integer}
  * `-1`：从光标向左
  * `1`：从光标向右
  * `0`：整行
* 返回：this

`rl.clearLine()` 方法向内部待处理操作列表添加一个操作，该操作清除关联 `stream` 的当前行，方向由 `dir` 指定。调用 `rl.commit()` 以查看此方法的效果，除非构造时传入了 `autoCommit: true`。

#### `rl.clearScreenDown()`

<!-- YAML
added: v17.0.0
-->

* 返回：this

`rl.clearScreenDown()` 方法向内部待处理操作列表添加一个操作，该操作从光标当前位置向下清除关联的流。调用 `rl.commit()` 以查看此方法的效果，除非构造时传入了 `autoCommit: true`。

#### `rl.commit()`

<!-- YAML
added: v17.0.0
-->

* 返回：{Promise}

`rl.commit()` 方法将所有待处理操作发送到关联的 `stream` 并清除内部待处理操作列表。

#### `rl.cursorTo(x[, y])`

<!-- YAML
added: v17.0.0
-->

* `x` {integer}
* `y` {integer}
* 返回：this

`rl.cursorTo()` 方法向内部待处理操作列表添加一个操作，该操作将光标移动到关联 `stream` 中的指定位置。调用 `rl.commit()` 以查看此方法的效果，除非构造时传入了 `autoCommit: true`。

#### `rl.moveCursor(dx, dy)`

<!-- YAML
added: v17.0.0
-->

* `dx` {integer}
* `dy` {integer}
* 返回：this

`rl.moveCursor()` 方法向内部待处理操作列表添加一个操作，该操作将光标*相对*于其当前位置在关联的 `stream` 中移动。调用 `rl.commit()` 以查看此方法的效果，除非构造时传入了 `autoCommit: true`。

#### `rl.rollback()`

<!-- YAML
added: v17.0.0
-->

* 返回：this

`rl.rollback` 方法清除内部待处理操作列表，而不将其发送到关联的 `stream`。

### `readlinePromises.createInterface(options)`

<!-- YAML
added: v17.0.0
-->

* `options` {Object}
  * `input` {stream.Readable} 要监听的 [Readable][] 流。此选项是*必需的*。
  * `output` {stream.Writable} 要写入 readline 数据的 [Writable][] 流。
  * `completer` {Function} 用于 Tab 自动补全的可选函数。
  * `terminal` {boolean} 如果 `input` 和 `output` 流应被视为 TTY，并向其写入 ANSI/VT100 转义码，则为 `true`。**默认值：** 在实例化时检查 `output` 流上的 `isTTY`。
  * `history` {string\[]} 历史行的初始列表。此选项仅在 `terminal` 由用户或通过内部 `output` 检查设置为 `true` 时才有意义，否则根本不会初始化历史缓存机制。**默认值：** `[]`。
  * `historySize` {number} 保留的历史行最大数量。要禁用历史记录，请将此值设置为 `0`。此选项仅在 `terminal` 由用户或通过内部 `output` 检查设置为 `true` 时才有意义，否则根本不会初始化历史缓存机制。**默认值：** `30`。
  * `removeHistoryDuplicates` {boolean} 如果为 `true`，则当添加到历史列表的新输入行与旧行重复时，从列表中删除旧行。**默认值：** `false`。
  * `prompt` {string} 要使用的提示字符串。**默认值：** `'> '`。
  * `crlfDelay` {number} 如果 `\r` 和 `\n` 之间的延迟超过 `crlfDelay` 毫秒，则 `\r` 和 `\n` 都将被视为单独的行结束输入。`crlfDelay` 将被强制转换为不小于 `100` 的数字。可以设置为 `Infinity`，在这种情况下，`\r` 后跟 `\n` 将始终被视为单个换行符（这对于读取具有 `\r\n` 行分隔符的[文件][]可能是合理的）。**默认值：** `100`。
  * `escapeCodeTimeout` {number} `readlinePromises` 等待字符的持续时间（当读取模糊键序列时，以毫秒为单位，该序列既可以形成使用到目前为止读取的输入的完整键序列，也可以接受额外输入以完成更长的键序列）。**默认值：** `500`。
  * `tabSize` {integer} 制表符等于的空格数（最小为 1）。**默认值：** `8`。
  * `signal` {AbortSignal} 允许使用 AbortSignal 关闭接口。
* 返回：{readlinePromises.Interface}

`readlinePromises.createInterface()` 方法创建一个新的 `readlinePromises.Interface` 实例。

```mjs
import { createInterface } from 'node:readline/promises';
import { stdin, stdout } from 'node:process';
const rl = createInterface({
  input: stdin,
  output: stdout,
});
```

```cjs
const { createInterface } = require('node:readline/promises');
const rl = createInterface({
  input: process.stdin,
  output: process.stdout,
});
```

一旦创建了 `readlinePromises.Interface` 实例，最常见的情况是监听 `'line'` 事件：

```js
rl.on('line', (line) => {
  console.log(`Received: ${line}`);
});
```

如果此实例的 `terminal` 为 `true`，那么如果 `output` 流定义了 `output.columns` 属性并在列更改时（或在列更改时）在 `output` 上发出 `'resize'` 事件，则 `output` 流将获得最佳兼容性（[`process.stdout`][] 在它是 TTY 时会自动执行此操作）。

#### `completer` 函数的使用

`completer` 函数将用户输入的当前行作为参数，并返回一个包含 2 个条目的 `Array`：

* 一个包含补全匹配项的 `Array`。
* 用于匹配的子字符串。

例如：`[[substr1, substr2, ...], originalsubstring]`。

```js
function completer(line) {
  const completions = '.help .error .exit .quit .q'.split(' ');
  const hits = completions.filter((c) => c.startsWith(line));
  // 如果未找到匹配项，则显示所有补全
  return [hits.length ? hits : completions, line];
}
```

`completer` 函数也可以返回一个 {Promise}，或者是异步的：

```js
async function completer(linePartial) {
  await someAsyncWork();
  return [['123'], linePartial];
}
```

## 回调 API

<!-- YAML
added: v0.1.104
-->

### 类：`readline.Interface`

<!-- YAML
added: v0.1.104
changes:
  - version: v17.0.0
    pr-url: https://github.com/nodejs/node/pull/37947
    description: The class `readline.Interface` now inherits from `Interface`.
-->

* 继承自：{readline.InterfaceConstructor}

`readline.Interface` 类的实例是使用 `readline.createInterface()` 方法构造的。每个实例都与一个 `input` [Readable][] 流和一个 `output` [Writable][] 流相关联。`output` 流用于打印用户输入的提示，该输入到达并从 `input` 流中读取。

#### `rl.question(query[, options], callback)`

<!-- YAML
added: v0.3.3
-->

* `query` {string} 要写入 `output` 的语句或查询，前置到提示符。
* `options` {Object}
  * `signal` {AbortSignal} 可选地允许使用 `AbortController` 取消 `question()`。
* `callback` {Function} 一个回调函数，使用用户对 `query` 的输入调用。

`rl.question()` 方法通过将 `query` 写入 `output` 来显示它，等待用户在 `input` 上提供输入，然后调用 `callback` 函数，将提供的输入作为第一个参数传递。

调用时，如果 `input` 流已暂停，`rl.question()` 将恢复它。

如果创建 `readline.Interface` 时将 `output` 设置为 `null` 或 `undefined`，则不会写入 `query`。

传递给 `rl.question()` 的 `callback` 函数不接受典型的 `Error` 对象或 `null` 作为第一个参数。`callback` 使用提供的答案作为唯一参数调用。

如果在 `rl.close()` 之后调用 `rl.question()`，将抛出错误。

用法示例：

```js
rl.question('What is your favorite food? ', (answer) => {
  console.log(`Oh, so your favorite food is ${answer}`);
});
```

使用 `AbortController` 取消问题。

```js
const ac = new AbortController();
const signal = ac.signal;

rl.question('What is your favorite food? ', { signal }, (answer) => {
  console.log(`Oh, so your favorite food is ${answer}`);
});

signal.addEventListener('abort', () => {
  console.log('The food question timed out');
}, { once: true });

setTimeout(() => ac.abort(), 10000);
```

### `readline.clearLine(stream, dir[, callback])`

<!-- YAML
added: v0.7.7
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v12.7.0
    pr-url: https://github.com/nodejs/node/pull/28674
    description: The stream's write() callback and return value are exposed.
-->

* `stream` {stream.Writable}
* `dir` {number}
  * `-1`：从光标向左
  * `1`：从光标向右
  * `0`：整行
* `callback` {Function} 操作完成后调用。
* 返回：{boolean} 如果 `stream` 希望调用代码在继续写入更多数据之前等待 `'drain'` 事件发出，则为 `false`；否则为 `true`。

`readline.clearLine()` 方法清除给定 [TTY][] 流的当前行，方向由 `dir` 指定。

### `readline.clearScreenDown(stream[, callback])`

<!-- YAML
added: v0.7.7
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v12.7.0
    pr-url: https://github.com/nodejs/node/pull/28641
    description: The stream's write() callback and return value are exposed.
-->

* `stream` {stream.Writable}
* `callback` {Function} 操作完成后调用。
* 返回：{boolean} 如果 `stream` 希望调用代码在继续写入更多数据之前等待 `'drain'` 事件发出，则为 `false`；否则为 `true`。

`readline.clearScreenDown()` 方法从光标当前位置向下清除给定的 [TTY][] 流。

### `readline.createInterface(options)`

<!-- YAML
added: v0.1.98
changes:
  - version:
      - v15.14.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37932
    description: The `signal` option is supported now.
  - version:
      - v15.8.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/33662
    description: The `history` option is supported now.
  - version: v13.9.0
    pr-url: https://github.com/nodejs/node/pull/31318
    description: The `tabSize` option is supported now.
  - version:
    - v8.3.0
    - v6.11.4
    pr-url: https://github.com/nodejs/node/pull/13497
    description: Remove max limit of `crlfDelay` option.
  - version: v6.6.0
    pr-url: https://github.com/nodejs/node/pull/8109
    description: The `crlfDelay` option is supported now.
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/7125
    description: The `prompt` option is supported now.
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/6352
    description: The `historySize` option can be `0` now.
-->

* `options` {Object}
  * `input` {stream.Readable} 要监听的 [Readable][] 流。此选项是*必需的*。
  * `output` {stream.Writable} 要写入 readline 数据的 [Writable][] 流。
  * `completer` {Function} 用于 Tab 自动补全的可选函数。
  * `terminal` {boolean} 如果 `input` 和 `output` 流应被视为 TTY，并向其写入 ANSI/VT100 转义码，则为 `true`。**默认值：** 在实例化时检查 `output` 流上的 `isTTY`。
  * `history` {string\[]} 历史行的初始列表。此选项仅在 `terminal` 由用户或通过内部 `output` 检查设置为 `true` 时才有意义，否则根本不会初始化历史缓存机制。**默认值：** `[]`。
  * `historySize` {number} 保留的历史行最大数量。要禁用历史记录，请将此值设置为 `0`。此选项仅在 `terminal` 由用户或通过内部 `output` 检查设置为 `true` 时才有意义，否则根本不会初始化历史缓存机制。**默认值：** `30`。
  * `removeHistoryDuplicates` {boolean} 如果为 `true`，则当添加到历史列表的新输入行与旧行重复时，从列表中删除旧行。**默认值：** `false`。
  * `prompt` {string} 要使用的提示字符串。**默认值：** `'> '`。
  * `crlfDelay` {number} 如果 `\r` 和 `\n` 之间的延迟超过 `crlfDelay` 毫秒，则 `\r` 和 `\n` 都将被视为单独的行结束输入。`crlfDelay` 将被强制转换为不小于 `100` 的数字。可以设置为 `Infinity`，在这种情况下，`\r` 后跟 `\n` 将始终被视为单个换行符（这对于读取具有 `\r\n` 行分隔符的[文件][]可能是合理的）。**默认值：** `100`。
  * `escapeCodeTimeout` {number} `readline` 等待字符的持续时间（当读取模糊键序列时，以毫秒为单位，该序列既可以形成使用到目前为止读取的输入的完整键序列，也可以接受额外输入以完成更长的键序列）。**默认值：** `500`。
  * `tabSize` {integer} 制表符等于的空格数（最小为 1）。**默认值：** `8`。
  * `signal` {AbortSignal} 允许使用 AbortSignal 关闭接口。中止信号将在内部调用接口上的 `close`。
* 返回：{readline.Interface}

`readline.createInterface()` 方法创建一个新的 `readline.Interface` 实例。

```mjs
import { createInterface } from 'node:readline';
import { stdin, stdout } from 'node:process';
const rl = createInterface({
  input: stdin,
  output: stdout,
});
```

```cjs
const { createInterface } = require('node:readline');
const rl = createInterface({
  input: process.stdin,
  output: process.stdout,
});
```

一旦创建了 `readline.Interface` 实例，最常见的情况是监听 `'line'` 事件：

```js
rl.on('line', (line) => {
  console.log(`Received: ${line}`);
});
```

如果此实例的 `terminal` 为 `true`，那么如果 `output` 流定义了 `output.columns` 属性并在列更改时（或在列更改时）在 `output` 上发出 `'resize'` 事件，则 `output` 流将获得最佳兼容性（[`process.stdout`][] 在它是 TTY 时会自动执行此操作）。

当使用 `stdin` 作为输入创建 `readline.Interface` 时，程序在接收到 [EOF 字符][]之前不会终止。要退出而不等待用户输入，请调用 `process.stdin.unref()`。

#### `completer` 函数的使用

`completer` 函数将用户输入的当前行作为参数，并返回一个包含 2 个条目的 `Array`：

* 一个包含补全匹配项的 `Array`。
* 用于匹配的子字符串。

例如：`[[substr1, substr2, ...], originalsubstring]`。

```js
function completer(line) {
  const completions = '.help .error .exit .quit .q'.split(' ');
  const hits = completions.filter((c) => c.startsWith(line));
  // 如果未找到匹配项，则显示所有补全
  return [hits.length ? hits : completions, line];
}
```

如果 `completer` 函数接受两个参数，则可以异步调用它：

```js
function completer(linePartial, callback) {
  callback(null, [['123'], linePartial]);
}
```

### `readline.cursorTo(stream, x[, y][, callback])`

<!-- YAML
added: v0.7.7
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v12.7.0
    pr-url: https://github.com/nodejs/node/pull/28674
    description: The stream's write() callback and return value are exposed.
-->

* `stream` {stream.Writable}
* `x` {number}
* `y` {number}
* `callback` {Function} 操作完成后调用。
* 返回：{boolean} 如果 `stream` 希望调用代码在继续写入更多数据之前等待 `'drain'` 事件发出，则为 `false`；否则为 `true`。

`readline.cursorTo()` 方法将光标移动到给定 [TTY][] `stream` 中的指定位置。

### `readline.moveCursor(stream, dx, dy[, callback])`

<!-- YAML
added: v0.7.7
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v12.7.0
    pr-url: https://github.com/nodejs/node/pull/28674
    description: The stream's write() callback and return value are exposed.
-->

* `stream` {stream.Writable}
* `dx` {number}
* `dy` {number}
* `callback` {Function} 操作完成后调用。
* 返回：{boolean} 如果 `stream` 希望调用代码在继续写入更多数据之前等待 `'drain'` 事件发出，则为 `false`；否则为 `true`。

`readline.moveCursor()` 方法将光标*相对*于其当前位置在给定的 [TTY][] `stream` 中移动。

## `readline.emitKeypressEvents(stream[, interface])`

<!-- YAML
added: v0.7.7
-->

* `stream` {stream.Readable}
* `interface` {readline.InterfaceConstructor}

`readline.emitKeypressEvents()` 方法使给定的 [Readable][] 流开始发出与接收到的输入对应的 `'keypress'` 事件。

可选地，`interface` 指定一个 `readline.Interface` 实例，当检测到复制粘贴的输入时，该实例的自动补全将被禁用。

如果 `stream` 是 [TTY][]，则它必须处于原始模式。

如果 `input` 是终端，任何 readline 实例都会自动在其 `input` 上调用此方法。关闭 `readline` 实例不会停止 `input` 发出 `'keypress'` 事件。

```js
readline.emitKeypressEvents(process.stdin);
if (process.stdin.isTTY)
  process.stdin.setRawMode(true);
```

## 示例：小型 CLI

以下示例说明了使用 `readline.Interface` 类实现一个小型命令行界面：

```mjs
import { createInterface } from 'node:readline';
import { exit, stdin, stdout } from 'node:process';
const rl = createInterface({
  input: stdin,
  output: stdout,
  prompt: 'OHAI> ',
});

rl.prompt();

rl.on('line', (line) => {
  switch (line.trim()) {
    case 'hello':
      console.log('world!');
      break;
    default:
      console.log(`Say what? I might have heard '${line.trim()}'`);
      break;
  }
  rl.prompt();
}).on('close', () => {
  console.log('Have a great day!');
  exit(0);
});
```

```cjs
const { createInterface } = require('node:readline');
const rl = createInterface({
  input: process.stdin,
  output: process.stdout,
  prompt: 'OHAI> ',
});

rl.prompt();

rl.on('line', (line) => {
  switch (line.trim()) {
    case 'hello':
      console.log('world!');
      break;
    default:
      console.log(`Say what? I might have heard '${line.trim()}'`);
      break;
  }
  rl.prompt();
}).on('close', () => {
  console.log('Have a great day!');
  process.exit(0);
});
```

## 示例：逐行读取文件流

`readline` 的一个常见用例是一次一行地使用输入文件。最简单的方法是利用 [`fs.ReadStream`][] API 以及 `for await...of` 循环：

```mjs
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

async function processLineByLine() {
  const fileStream = createReadStream('input.txt');

  const rl = createInterface({
    input: fileStream,
    crlfDelay: Infinity,
  });
  // 注意：我们使用 crlfDelay 选项将 input.txt 中的所有 CR LF 实例（'\r\n'）识别为单个换行符。

  for await (const line of rl) {
    // input.txt 中的每一行将在此处作为 `line` 依次可用。
    console.log(`Line from file: ${line}`);
  }
}

processLineByLine();
```

```cjs
const { createReadStream } = require('node:fs');
const { createInterface } = require('node:readline');

async function processLineByLine() {
  const fileStream = createReadStream('input.txt');

  const rl = createInterface({
    input: fileStream,
    crlfDelay: Infinity,
  });
  // 注意：我们使用 crlfDelay 选项将 input.txt 中的所有 CR LF 实例（'\r\n'）识别为单个换行符。

  for await (const line of rl) {
    // input.txt 中的每一行将在此处作为 `line` 依次可用。
    console.log(`Line from file: ${line}`);
  }
}

processLineByLine();
```

或者，可以使用 [`'line'`][] 事件：

```mjs
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

const rl = createInterface({
  input: createReadStream('sample.txt'),
  crlfDelay: Infinity,
});

rl.on('line', (line) => {
  console.log(`Line from file: ${line}`);
});
```

```cjs
const { createReadStream } = require('node:fs');
const { createInterface } = require('node:readline');

const rl = createInterface({
  input: createReadStream('sample.txt'),
  crlfDelay: Infinity,
});

rl.on('line', (line) => {
  console.log(`Line from file: ${line}`);
});
```

目前，`for await...of` 循环可能稍慢一些。如果 `async` / `await` 流程和速度都至关重要，可以采用混合方法：

```mjs
import { once } from 'node:events';
import { createReadStream } from 'node:fs';
import { createInterface } from 'node:readline';

(async function processLineByLine() {
  try {
    const rl = createInterface({
      input: createReadStream('big-file.txt'),
      crlfDelay: Infinity,
    });

    rl.on('line', (line) => {
      // 处理该行。
    });

    await once(rl, 'close');

    console.log('File processed.');
  } catch (err) {
    console.error(err);
  }
})();
```

```cjs
const { once } = require('node:events');
const { createReadStream } = require('node:fs');
const { createInterface } = require('node:readline');

(async function processLineByLine() {
  try {
    const rl = createInterface({
      input: createReadStream('big-file.txt'),
      crlfDelay: Infinity,
    });

    rl.on('line', (line) => {
      // 处理该行。
    });

    await once(rl, 'close');

    console.log('File processed.');
  } catch (err) {
    console.error(err);
  }
})();
```

## TTY 键绑定

<table>
  <tr>
    <th>键绑定</th>
    <th>描述</th>
    <th>备注</th>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>Backspace</kbd></td>
    <td>向左删除行</td>
    <td>在 Linux、Mac 和 Windows 上不起作用</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>Delete</kbd></td>
    <td>向右删除行</td>
    <td>在 Mac 上不起作用</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>C</kbd></td>
    <td>发出 <code>SIGINT</code> 或关闭 readline 实例</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>H</kbd></td>
    <td>向左删除</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>D</kbd></td>
    <td>向右删除或在当前行为空 / EOF 时关闭 readline 实例</td>
    <td>在 Windows 上不起作用</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>U</kbd></td>
    <td>从当前位置删除到行首</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>K</kbd></td>
    <td>从当前位置删除到行尾</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>Y</kbd></td>
    <td>召回先前删除的文本</td>
    <td>仅适用于通过 <kbd>Ctrl</kbd>+<kbd>U</kbd> 或 <kbd>Ctrl</kbd>+<kbd>K</kbd> 删除的文本</td>
  </tr>
  <tr>
    <td><kbd>Meta</kbd>+<kbd>Y</kbd></td>
    <td>在先前删除的文本之间循环</td>
    <td>仅当最后一次击键是 <kbd>Ctrl</kbd>+<kbd>Y</kbd> 或 <kbd>Meta</kbd>+<kbd>Y</kbd> 时可用</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>A</kbd></td>
    <td>转到行首</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>E</kbd></td>
    <td>转到行尾</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>B</kbd></td>
    <td>向后一个字符</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>F</kbd></td>
    <td>向前一个字符</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>L</kbd></td>
    <td>清屏</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>N</kbd></td>
    <td>下一个历史项</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>P</kbd></td>
    <td>上一个历史项</td>
    <td></td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>-</kbd></td>
    <td>撤销上一个更改</td>
    <td>任何发出键码 <code>0x1F</code> 的击键都将执行此操作。
    在许多终端中，例如 <code>xterm</code>，
    这绑定到 <kbd>Ctrl</kbd>+<kbd>-</kbd>。</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>6</kbd></td>
    <td>重做上一个更改</td>
    <td>许多终端没有默认的重做击键。
    我们选择键码 <code>0x1E</code> 来执行重做。
    在 <code>xterm</code> 中，默认绑定到 <kbd>Ctrl</kbd>+<kbd>6</kbd>。</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>Z</kbd></td>
    <td>将正在运行的进程移动到后台。键入
    <code>fg</code> 并按 <kbd>Enter</kbd>
    返回。</td>
    <td>在 Windows 上不起作用</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>W</kbd> 或 <kbd>Ctrl</kbd>
   +<kbd>Backspace</kbd></td>
    <td>向后删除到单词边界</td>
    <td><kbd>Ctrl</kbd>+<kbd>Backspace</kbd> 在
    Linux、Mac 和 Windows 上不起作用</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>Delete</kbd></td>
    <td>向前删除到单词边界</td>
    <td>在 Mac 上不起作用</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>左箭头</kbd> 或
    <kbd>Meta</kbd>+<kbd>B</kbd></td>
    <td>向左一个单词</td>
    <td><kbd>Ctrl</kbd>+<kbd>左箭头</kbd> 在
    Mac 上不起作用</td>
  </tr>
  <tr>
    <td><kbd>Ctrl</kbd>+<kbd>右箭头</kbd> 或
    <kbd>Meta</kbd>+<kbd>F</kbd></td>
    <td>向右一个单词</td>
    <td><kbd>Ctrl</kbd>+<kbd>右箭头</kbd> 在
    Mac 上不起作用</td>
  </tr>
  <tr>
    <td><kbd>Meta</kbd>+<kbd>D</kbd> 或 <kbd>Meta</kbd>
   +<kbd>Delete</kbd></td>
    <td>向右删除单词</td>
    <td><kbd>Meta</kbd>+<kbd>Delete</kbd> 在
    Windows 上不起作用</td>
  </tr>
  <tr>
    <td><kbd>Meta</kbd>+<kbd>Backspace</kbd></td>
    <td>向左删除单词</td>
    <td>在 Mac 上不起作用</td>
  </tr>
</table>


[EOF character]: https://en.wikipedia.org/wiki/End-of-file#EOF_character
[Readable]: stream.md#readable-streams
[TTY]: tty.md
[TTY keybindings]: #tty-keybindings
[Writable]: stream.md#writable-streams
[`'SIGCONT'`]: #event-sigcont
[`'SIGTSTP'`]: #event-sigtstp
[`'line'`]: #event-line
[`fs.ReadStream`]: fs.md#class-fsreadstream
[`process.stdin`]: process.md#processstdin
[`process.stdout`]: process.md#processstdout
[`rl.close()`]: #rlclose
[reading files]: #example-read-file-stream-line-by-line
