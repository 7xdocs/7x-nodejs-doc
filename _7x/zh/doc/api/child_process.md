# 子进程

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/child_process.js -->

`node:child_process` 模块提供了以类似于但不完全等同于 popen(3) 的方式生成子进程的能力。这个功能主要由 [`child_process.spawn()`][] 函数提供：

```cjs
const { spawn } = require('node:child_process');
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.error(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});
```

```mjs
import { spawn } from 'node:child_process';
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.error(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});
```

默认情况下，会在父 Node.js 进程和生成的子进程之间建立 `stdin`、`stdout` 和 `stderr` 的管道。这些管道的容量有限（且依赖于平台）。如果子进程向 stdout 写入的数据超过该限制且未被捕获，则子进程会阻塞，等待管道缓冲区接受更多数据。这与 shell 中管道的行为相同。如果输出不会被消耗，请使用 `{ stdio: 'ignore' }` 选项。

如果 `env` 在 `options` 对象中，则使用 `options.env.PATH` 环境变量执行命令查找。否则，使用 `process.env.PATH`。如果设置了 `options.env` 但没有 `PATH`，在 Unix 上将在默认搜索路径 `/usr/bin:/bin` 上执行查找（有关 execvpe/execvp 的信息，请参阅操作系统的手册），在 Windows 上使用当前进程的环境变量 `PATH`。

在 Windows 上，环境变量不区分大小写。Node.js 按字典顺序对 `env` 键进行排序，并使用第一个不区分大小写匹配的键。只有第一个（按字典顺序）条目会被传递给子进程。在 Windows 上，当向 `env` 选项传递具有相同键的多个变体（例如 `PATH` 和 `Path`）的对象时，这可能会导致问题。

[`child_process.spawn()`][] 方法异步地生成子进程，不会阻塞 Node.js 事件循环。[`child_process.spawnSync()`][] 函数以同步方式提供等效功能，它会阻塞事件循环，直到生成的进程退出或被终止。

为了方便起见，`node:child_process` 模块提供了几个同步和异步的替代方法，用于 [`child_process.spawn()`][] 和 [`child_process.spawnSync()`][]。这些替代方法中的每一个都是在 [`child_process.spawn()`][] 或 [`child_process.spawnSync()`][] 之上实现的。

* [`child_process.exec()`][]：生成一个 shell 并在该 shell 中运行命令，完成后将 `stdout` 和 `stderr` 传递给回调函数。
* [`child_process.execFile()`][]：类似于 [`child_process.exec()`][]，不同之处在于它默认直接生成命令而不先生成 shell。
* [`child_process.fork()`][]：生成一个新的 Node.js 进程并调用指定模块，同时建立了一个 IPC 通信通道，允许在父进程和子进程之间发送消息。
* [`child_process.execSync()`][]：[`child_process.exec()`][] 的同步版本，会阻塞 Node.js 事件循环。
* [`child_process.execFileSync()`][]：[`child_process.execFile()`][] 的同步版本，会阻塞 Node.js 事件循环。

对于某些用例，例如自动化 shell 脚本，[同步对应方法][]可能更方便。然而，在许多情况下，同步方法可能会对性能产生重大影响，因为在生成的进程完成时会停滞事件循环。

## 异步进程创建

[`child_process.spawn()`][]、[`child_process.fork()`][]、[`child_process.exec()`][] 和 [`child_process.execFile()`][] 方法都遵循其他 Node.js API 典型的惯用异步编程模式。

每个方法都返回一个 [`ChildProcess`][] 实例。这些对象实现了 Node.js [`EventEmitter`][] API，允许父进程注册监听器函数，这些函数在子进程生命周期中发生某些事件时被调用。

[`child_process.exec()`][] 和 [`child_process.execFile()`][] 方法还允许指定一个可选的 `callback` 函数，该函数在子进程终止时被调用。

### 在 Windows 上生成 `.bat` 和 `.cmd` 文件

[`child_process.exec()`][] 和 [`child_process.execFile()`][] 之间区别的重要性可能因平台而异。在类 Unix 操作系统（Unix、Linux、macOS）上，[`child_process.execFile()`][] 可能更高效，因为它默认不生成 shell。然而，在 Windows 上，`.bat` 和 `.cmd` 文件在没有终端的情况下自身是不可执行的，因此不能使用 [`child_process.execFile()`][] 启动。在 Windows 上运行时，可以使用设置了 `shell` 选项的 [`child_process.spawn()`][]、[`child_process.exec()`][] 或通过生成 `cmd.exe` 并将 `.bat` 或 `.cmd` 文件作为参数传递（这是 `shell` 选项和 [`child_process.exec()`][] 所做的）来调用 `.bat` 和 `.cmd` 文件。在任何情况下，如果脚本文件名包含空格，则需要用引号括起来。

```cjs
// 或者...
const { exec, spawn } = require('node:child_process');

exec('my.bat', (err, stdout, stderr) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(stdout);
});

// 文件名中包含空格的脚本：
const bat = spawn('"my script.cmd" a b', { shell: true });
// 或者：
exec('"my script.cmd" a b', (err, stdout, stderr) => {
  // ...
});
```

```mjs
// 或者...
import { exec, spawn } from 'node:child_process';

exec('my.bat', (err, stdout, stderr) => {
  if (err) {
    console.error(err);
    return;
  }
  console.log(stdout);
});

// 文件名中包含空格的脚本：
const bat = spawn('"my script.cmd" a b', { shell: true });
// 或者：
exec('"my script.cmd" a b', (err, stdout, stderr) => {
  // ...
});
```

### `child_process.exec(command[, options][, callback])`

<!-- YAML
added: v0.1.90
changes:
  - version:
      - v16.4.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38862
    description: The `cwd` option can be a WHATWG `URL` object using
                 `file:` protocol.
  - version: v15.4.0
    pr-url: https://github.com/nodejs/node/pull/36308
    description: AbortSignal support was added.
  - version: v8.8.0
    pr-url: https://github.com/nodejs/node/pull/15380
    description: The `windowsHide` option is supported now.
-->

* `command` {string} 要运行的命令，参数以空格分隔。
* `options` {Object}
  * `cwd` {string|URL} 子进程的当前工作目录。**默认值：** `process.cwd()`。
  * `env` {Object} 环境键值对。**默认值：** `process.env`。
  * `encoding` {string} **默认值：** `'utf8'`
  * `shell` {string} 用于执行命令的 shell。请参阅 [Shell 要求][] 和 [默认 Windows shell][]。**默认值：** 在 Unix 上是 `'/bin/sh'`，在 Windows 上是 `process.env.ComSpec`。
  * `signal` {AbortSignal} 允许使用 AbortSignal 中止子进程。
  * `timeout` {number} **默认值：** `0`
  * `maxBuffer` {number} 允许在 stdout 或 stderr 上的最大数据量（以字节为单位）。如果超出，子进程将被终止，并且任何输出都会被截断。请参阅 [`maxBuffer` 和 Unicode][] 的注意事项。**默认值：** `1024 * 1024`。
  * `killSignal` {string|integer} **默认值：** `'SIGTERM'`
  * `uid` {number} 设置进程的用户标识（参见 setuid(2)）。
  * `gid` {number} 设置进程的组标识（参见 setgid(2)）。
  * `windowsHide` {boolean} 隐藏通常在 Windows 系统上创建的子进程控制台窗口。**默认值：** `false`。
* `callback` {Function} 进程终止时调用，并传入输出。
  * `error` {Error}
  * `stdout` {string|Buffer}
  * `stderr` {string|Buffer}
* 返回：{ChildProcess}

生成一个 shell，然后在该 shell 中执行 `command`，缓冲任何生成的输出。传递给 exec 函数的 `command` 字符串由 shell 直接处理，特殊字符（根据 [shell](https://en.wikipedia.org/wiki/List_of_command-line_interpreters) 不同而变化）需要相应处理：

```cjs
const { exec } = require('node:child_process');

exec('"/path/to/test file/test.sh" arg1 arg2');
// 使用双引号，这样路径中的空格不会被解释为多个参数的分隔符。

exec('echo "The \\$HOME variable is $HOME"');
// $HOME 变量在第一个实例中被转义，但在第二个中没有。
```

```mjs
import { exec } from 'node:child_process';

exec('"/path/to/test file/test.sh" arg1 arg2');
// 使用双引号，这样路径中的空格不会被解释为多个参数的分隔符。

exec('echo "The \\$HOME variable is $HOME"');
// $HOME 变量在第一个实例中被转义，但在第二个中没有。
```

**切勿将未经过滤的用户输入传递给此函数。任何包含 shell 元字符的输入都可能被用来触发任意命令执行。**

如果提供了 `callback` 函数，则使用参数 `(error, stdout, stderr)` 调用它。成功时，`error` 将为 `null`。出错时，`error` 将是 [`Error`][] 的一个实例。`error.code` 属性将是进程的退出代码。按照惯例，任何非 `0` 的退出代码都表示错误。`error.signal` 将是终止进程的信号。

传递给回调的 `stdout` 和 `stderr` 参数将包含子进程的 stdout 和 stderr 输出。默认情况下，Node.js 会将输出解码为 UTF-8 并将字符串传递给回调。`encoding` 选项可用于指定用于解码 stdout 和 stderr 输出的字符编码。如果 `encoding` 是 `'buffer'` 或无法识别的字符编码，则将把 `Buffer` 对象传递给回调。

```cjs
const { exec } = require('node:child_process');
exec('cat *.js missing_file | wc -l', (error, stdout, stderr) => {
  if (error) {
    console.error(`exec error: ${error}`);
    return;
  }
  console.log(`stdout: ${stdout}`);
  console.error(`stderr: ${stderr}`);
});
```

```mjs
import { exec } from 'node:child_process';
exec('cat *.js missing_file | wc -l', (error, stdout, stderr) => {
  if (error) {
    console.error(`exec error: ${error}`);
    return;
  }
  console.log(`stdout: ${stdout}`);
  console.error(`stderr: ${stderr}`);
});
```

如果 `timeout` 大于 `0`，则如果子进程运行时间超过 `timeout` 毫秒，父进程将发送由 `killSignal` 属性标识的信号（默认为 `'SIGTERM'`）。

与 exec(3) POSIX 系统调用不同，`child_process.exec()` 不会替换现有进程，而是使用 shell 来执行命令。

如果此方法以其 [`util.promisify()`][] 化版本调用，则返回一个 `Promise`，用于一个具有 `stdout` 和 `stderr` 属性的 `Object`。返回的 `ChildProcess` 实例作为 `child` 属性附加到 `Promise` 上。如果发生错误（包括任何导致退出码非 0 的错误），则返回一个被拒绝的 promise，其中包含回调中给出的相同 `error` 对象，但还有两个额外的属性 `stdout` 和 `stderr`。

```cjs
const util = require('node:util');
const exec = util.promisify(require('node:child_process').exec);

async function lsExample() {
  const { stdout, stderr } = await exec('ls');
  console.log('stdout:', stdout);
  console.error('stderr:', stderr);
}
lsExample();
```

```mjs
import { promisify } from 'node:util';
import child_process from 'node:child_process';
const exec = promisify(child_process.exec);

async function lsExample() {
  const { stdout, stderr } = await exec('ls');
  console.log('stdout:', stdout);
  console.error('stderr:', stderr);
}
lsExample();
```

如果启用了 `signal` 选项，在相应的 `AbortController` 上调用 `.abort()` 类似于在子进程上调用 `.kill()`，不同之处在于传递给回调的错误将是一个 `AbortError`：

```cjs
const { exec } = require('node:child_process');
const controller = new AbortController();
const { signal } = controller;
const child = exec('grep ssh', { signal }, (error) => {
  console.error(error); // 一个 AbortError
});
controller.abort();
```

```mjs
import { exec } from 'node:child_process';
const controller = new AbortController();
const { signal } = controller;
const child = exec('grep ssh', { signal }, (error) => {
  console.error(error); // 一个 AbortError
});
controller.abort();
```

### `child_process.execFile(file[, args][, options][, callback])`

<!-- YAML
added: v0.1.91
changes:
  - version:
      - v23.11.0
      - v22.15.0
    pr-url: https://github.com/nodejs/node/pull/57389
    description: Passing `args` when `shell` is set to `true` is deprecated.
  - version:
      - v16.4.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38862
    description: The `cwd` option can be a WHATWG `URL` object using
                 `file:` protocol.
  - version:
      - v15.4.0
      - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/36308
    description: AbortSignal support was added.
  - version: v8.8.0
    pr-url: https://github.com/nodejs/node/pull/15380
    description: The `windowsHide` option is supported now.
-->

* `file` {string} 要运行的可执行文件的名称或路径。
* `args` {string\[]} 字符串参数列表。
* `options` {Object}
  * `cwd` {string|URL} 子进程的当前工作目录。
  * `env` {Object} 环境键值对。**默认值：** `process.env`。
  * `encoding` {string} **默认值：** `'utf8'`
  * `timeout` {number} **默认值：** `0`
  * `maxBuffer` {number} 允许在 stdout 或 stderr 上的最大数据量（以字节为单位）。如果超出，子进程将被终止，并且任何输出都会被截断。请参阅 [`maxBuffer` 和 Unicode][] 的注意事项。**默认值：** `1024 * 1024`。
  * `killSignal` {string|integer} **默认值：** `'SIGTERM'`
  * `uid` {number} 设置进程的用户标识（参见 setuid(2)）。
  * `gid` {number} 设置进程的组标识（参见 setgid(2)）。
  * `windowsHide` {boolean} 隐藏通常在 Windows 系统上创建的子进程控制台窗口。**默认值：** `false`。
  * `windowsVerbatimArguments` {boolean} 在 Windows 上不对参数进行引号或转义处理。在 Unix 上被忽略。**默认值：** `false`。
  * `shell` {boolean|string} 如果为 `true`，则在 shell 内运行 `command`。在 Unix 上使用 `'/bin/sh'`，在 Windows 上使用 `process.env.ComSpec`。可以指定一个不同的 shell 作为字符串。请参阅 [Shell 要求][] 和 [默认 Windows shell][]。**默认值：** `false`（无 shell）。
  * `signal` {AbortSignal} 允许使用 AbortSignal 中止子进程。
* `callback` {Function} 进程终止时调用，并传入输出。
  * `error` {Error}
  * `stdout` {string|Buffer}
  * `stderr` {string|Buffer}
* 返回：{ChildProcess}

`child_process.execFile()` 函数类似于 [`child_process.exec()`][]，不同之处在于它默认不生成 shell。相反，指定的可执行文件 `file` 直接作为新进程生成，使其比 [`child_process.exec()`][] 稍微更高效。

支持与 [`child_process.exec()`][] 相同的选项。由于没有生成 shell，因此不支持诸如 I/O 重定向和文件通配等行为。

```cjs
const { execFile } = require('node:child_process');
const child = execFile('node', ['--version'], (error, stdout, stderr) => {
  if (error) {
    throw error;
  }
  console.log(stdout);
});
```

```mjs
import { execFile } from 'node:child_process';
const child = execFile('node', ['--version'], (error, stdout, stderr) => {
  if (error) {
    throw error;
  }
  console.log(stdout);
});
```

传递给回调的 `stdout` 和 `stderr` 参数将包含子进程的 stdout 和 stderr 输出。默认情况下，Node.js 会将输出解码为 UTF-8 并将字符串传递给回调。`encoding` 选项可用于指定用于解码 stdout 和 stderr 输出的字符编码。如果 `encoding` 是 `'buffer'` 或无法识别的字符编码，则将把 `Buffer` 对象传递给回调。

如果此方法以其 [`util.promisify()`][] 化版本调用，则返回一个 `Promise`，用于一个具有 `stdout` 和 `stderr` 属性的 `Object`。返回的 `ChildProcess` 实例作为 `child` 属性附加到 `Promise` 上。如果发生错误（包括任何导致退出码非 0 的错误），则返回一个被拒绝的 promise，其中包含回调中给出的相同 `error` 对象，但还有两个额外的属性 `stdout` 和 `stderr`。

```cjs
const util = require('node:util');
const execFile = util.promisify(require('node:child_process').execFile);
async function getVersion() {
  const { stdout } = await execFile('node', ['--version']);
  console.log(stdout);
}
getVersion();
```

```mjs
import { promisify } from 'node:util';
import child_process from 'node:child_process';
const execFile = promisify(child_process.execFile);
async function getVersion() {
  const { stdout } = await execFile('node', ['--version']);
  console.log(stdout);
}
getVersion();
```

**如果启用了 `shell` 选项，请勿将未经过滤的用户输入传递给此函数。任何包含 shell 元字符的输入都可能被用来触发任意命令执行。**

如果启用了 `signal` 选项，在相应的 `AbortController` 上调用 `.abort()` 类似于在子进程上调用 `.kill()`，不同之处在于传递给回调的错误将是一个 `AbortError`：

```cjs
const { execFile } = require('node:child_process');
const controller = new AbortController();
const { signal } = controller;
const child = execFile('node', ['--version'], { signal }, (error) => {
  console.error(error); // 一个 AbortError
});
controller.abort();
```

```mjs
import { execFile } from 'node:child_process';
const controller = new AbortController();
const { signal } = controller;
const child = execFile('node', ['--version'], { signal }, (error) => {
  console.error(error); // 一个 AbortError
});
controller.abort();
```

### `child_process.fork(modulePath[, args][, options])`

<!-- YAML
added: v0.5.0
changes:
  - version:
      - v17.4.0
      - v16.14.0
    pr-url: https://github.com/nodejs/node/pull/41225
    description: The `modulePath` parameter can be a WHATWG `URL` object using
                 `file:` protocol.
  - version:
      - v16.4.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38862
    description: The `cwd` option can be a WHATWG `URL` object using
                 `file:` protocol.
  - version:
      - v15.13.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37256
    description: timeout was added.
  - version:
      - v15.11.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37325
    description: killSignal for AbortSignal was added.
  - version:
      - v15.6.0
      - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/36603
    description: AbortSignal support was added.
  - version:
      - v13.2.0
      - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30162
    description: The `serialization` option is supported now.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10866
    description: The `stdio` option can now be a string.
  - version: v6.4.0
    pr-url: https://github.com/nodejs/node/pull/7811
    description: The `stdio` option is supported now.
-->

* `modulePath` {string|URL} 要在子进程中运行的模块。
* `args` {string\[]} 字符串参数列表。
* `options` {Object}
  * `cwd` {string|URL} 子进程的当前工作目录。
  * `detached` {boolean} 准备子进程独立于其父进程运行。具体行为取决于平台（参见 [`options.detached`][]）。
  * `env` {Object} 环境键值对。**默认值：** `process.env`。
  * `execPath` {string} 用于创建子进程的可执行文件。
  * `execArgv` {string\[]} 传递给可执行文件的字符串参数列表。**默认值：** `process.execArgv`。
  * `gid` {number} 设置进程的组标识（参见 setgid(2)）。
  * `serialization` {string} 指定用于在进程之间发送消息的序列化类型。可能的值是 `'json'` 和 `'advanced'`。有关更多详细信息，请参阅 [高级序列化][]。**默认值：** `'json'`。
  * `signal` {AbortSignal} 允许使用 AbortSignal 关闭子进程。
  * `killSignal` {string|integer} 当生成的进程因超时或中止信号而被杀死时使用的信号值。**默认值：** `'SIGTERM'`。
  * `silent` {boolean} 如果为 `true`，则子进程的 stdin、stdout 和 stderr 将被管道传输到父进程，否则它们将从父进程继承，有关更多详细信息，请参阅 [`child_process.spawn()`][] 的 [`stdio`][] 的 `'pipe'` 和 `'inherit'` 选项。**默认值：** `false`。
  * `stdio` {Array|string} 参见 [`child_process.spawn()`][] 的 [`stdio`][]。当提供此选项时，它会覆盖 `silent`。如果使用数组变体，它必须包含一个值为 `'ipc'` 的项，否则将抛出错误。例如 `[0, 1, 2, 'ipc']`。
  * `uid` {number} 设置进程的用户标识（参见 setuid(2)）。
  * `windowsVerbatimArguments` {boolean} 在 Windows 上不对参数进行引号或转义处理。在 Unix 上被忽略。**默认值：** `false`。
  * `timeout` {number} 进程允许运行的最长时间（以毫秒为单位）。**默认值：** `undefined`。
* 返回：{ChildProcess}

`child_process.fork()` 方法是 [`child_process.spawn()`][] 的一个特例，专门用于生成新的 Node.js 进程。与 [`child_process.spawn()`][] 一样，返回一个 [`ChildProcess`][] 对象。返回的 [`ChildProcess`][] 将有一个额外的内置通信通道，允许在父进程和子进程之间来回传递消息。有关详细信息，请参阅 [`subprocess.send()`][]。

请记住，生成的 Node.js 子进程独立于父进程，除了在两者之间建立的 IPC 通信通道。每个进程都有自己的内存，带有自己的 V8 实例。由于需要额外的资源分配，不建议生成大量子 Node.js 进程。

默认情况下，`child_process.fork()` 将使用父进程的 [`process.execPath`][] 生成新的 Node.js 实例。`options` 对象中的 `execPath` 属性允许使用替代的执行路径。

使用自定义 `execPath` 启动的 Node.js 进程将使用在子进程上使用环境变量 `NODE_CHANNEL_FD` 标识的文件描述符（fd）与父进程通信。

与 fork(2) POSIX 系统调用不同，`child_process.fork()` 不会克隆当前进程。

[`child_process.spawn()`][] 中可用的 `shell` 选项不被 `child_process.fork()` 支持，如果设置将被忽略。

如果启用了 `signal` 选项，在相应的 `AbortController` 上调用 `.abort()` 类似于在子进程上调用 `.kill()`，不同之处在于传递给回调的错误将是一个 `AbortError`：

```cjs
const { fork } = require('node:child_process');
const process = require('node:process');

if (process.argv[2] === 'child') {
  setTimeout(() => {
    console.log(`Hello from ${process.argv[2]}!`);
  }, 1_000);
} else {
  const controller = new AbortController();
  const { signal } = controller;
  const child = fork(__filename, ['child'], { signal });
  child.on('error', (err) => {
    // 如果控制器中止，这将与 err 一起被调用，err 是一个 AbortError
  });
  controller.abort(); // 停止子进程
}
```

```mjs
import { fork } from 'node:child_process';
import process from 'node:process';

if (process.argv[2] === 'child') {
  setTimeout(() => {
    console.log(`Hello from ${process.argv[2]}!`);
  }, 1_000);
} else {
  const controller = new AbortController();
  const { signal } = controller;
  const child = fork(import.meta.url, ['child'], { signal });
  child.on('error', (err) => {
    // 如果控制器中止，这将与 err 一起被调用，err 是一个 AbortError
  });
  controller.abort(); // 停止子进程
}
```

### `child_process.spawn(command[, args][, options])`

<!-- YAML
added: v0.1.90
changes:
  - version:
      - v23.11.0
      - v22.15.0
    pr-url: https://github.com/nodejs/node/pull/57389
    description: Passing `args` when `shell` is set to `true` is deprecated.
  - version:
      - v16.4.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38862
    description: The `cwd` option can be a WHATWG `URL` object using
                 `file:` protocol.
  - version:
      - v15.13.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37256
    description: timeout was added.
  - version:
      - v15.11.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37325
    description: killSignal for AbortSignal was added.
  - version:
      - v15.5.0
      - v14.17.0
    pr-url: https://github.com/nodejs/node/pull/36432
    description: AbortSignal support was added.
  - version:
      - v13.2.0
      - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/30162
    description: The `serialization` option is supported now.
  - version: v8.8.0
    pr-url: https://github.com/nodejs/node/pull/15380
    description: The `windowsHide` option is supported now.
  - version: v6.4.0
    pr-url: https://github.com/nodejs/node/pull/7696
    description: The `argv0` option is supported now.
  - version: v5.7.0
    pr-url: https://github.com/nodejs/node/pull/4598
    description: The `shell` option is supported now.
-->

* `command` {string} 要运行的命令。
* `args` {string\[]} 字符串参数列表。
* `options` {Object}
  * `cwd` {string|URL} 子进程的当前工作目录。
  * `env` {Object} 环境键值对。**默认值：** `process.env`。
  * `argv0` {string} 显式设置发送给子进程的 `argv[0]` 的值。如果未指定，这将设置为 `command`。
  * `stdio` {Array|string} 子进程的 stdio 配置（参见 [`options.stdio`][`stdio`]）。
  * `detached` {boolean} 准备子进程独立于其父进程运行。具体行为取决于平台（参见 [`options.detached`][]）。
  * `uid` {number} 设置进程的用户标识（参见 setuid(2)）。
  * `gid` {number} 设置进程的组标识（参见 setgid(2)）。
  * `serialization` {string} 指定用于在进程之间发送消息的序列化类型。可能的值是 `'json'` 和 `'advanced'`。有关更多详细信息，请参阅 [高级序列化][]。**默认值：** `'json'`。
  * `shell` {boolean|string} 如果为 `true`，则在 shell 内运行 `command`。在 Unix 上使用 `'/bin/sh'`，在 Windows 上使用 `process.env.ComSpec`。可以指定一个不同的 shell 作为字符串。请参阅 [Shell 要求][] 和 [默认 Windows shell][]。**默认值：** `false`（无 shell）。
  * `windowsVerbatimArguments` {boolean} 在 Windows 上不对参数进行引号或转义处理。在 Unix 上被忽略。当指定 `shell` 且为 CMD 时，此选项自动设置为 `true`。**默认值：** `false`。
  * `windowsHide` {boolean} 隐藏通常在 Windows 系统上创建的子进程控制台窗口。**默认值：** `false`。
  * `signal` {AbortSignal} 允许使用 AbortSignal 中止子进程。
  * `timeout` {number} 进程允许运行的最长时间（以毫秒为单位）。**默认值：** `undefined`。
  * `killSignal` {string|integer} 当生成的进程因超时或中止信号而被杀死时使用的信号值。**默认值：** `'SIGTERM'`。
* 返回：{ChildProcess}

`child_process.spawn()` 方法使用给定的 `command` 和命令行参数 `args` 生成一个新进程。如果省略，`args` 默认为空数组。

**如果启用了 `shell` 选项，请勿将未经过滤的用户输入传递给此函数。任何包含 shell 元字符的输入都可能被用来触发任意命令执行。**

第三个参数可用于指定其他选项，默认为：

```js
const defaults = {
  cwd: undefined,
  env: process.env,
};
```

使用 `cwd` 指定从中生成进程的工作目录。如果未给出，则默认继承当前工作目录。如果给定，但路径不存在，则子进程会发出 `ENOENT` 错误并立即退出。当命令不存在时也会发出 `ENOENT`。

使用 `env` 指定对新进程可见的环境变量，默认为 [`process.env`][]。

`env` 中的 `undefined` 值将被忽略。

运行 `ls -lh /usr` 的示例，捕获 `stdout`、`stderr` 和退出代码：

```cjs
const { spawn } = require('node:child_process');
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.error(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});
```

```mjs
import { spawn } from 'node:child_process';
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.stderr.on('data', (data) => {
  console.error(`stderr: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process exited with code ${code}`);
});
```

示例：一种非常复杂的方式来运行 `ps ax | grep ssh`

```cjs
const { spawn } = require('node:child_process');
const ps = spawn('ps', ['ax']);
const grep = spawn('grep', ['ssh']);

ps.stdout.on('data', (data) => {
  grep.stdin.write(data);
});

ps.stderr.on('data', (data) => {
  console.error(`ps stderr: ${data}`);
});

ps.on('close', (code) => {
  if (code !== 0) {
    console.log(`ps process exited with code ${code}`);
  }
  grep.stdin.end();
});

grep.stdout.on('data', (data) => {
  console.log(data.toString());
});

grep.stderr.on('data', (data) => {
  console.error(`grep stderr: ${data}`);
});

grep.on('close', (code) => {
  if (code !== 0) {
    console.log(`grep process exited with code ${code}`);
  }
});
```

```mjs
import { spawn } from 'node:child_process';
const ps = spawn('ps', ['ax']);
const grep = spawn('grep', ['ssh']);

ps.stdout.on('data', (data) => {
  grep.stdin.write(data);
});

ps.stderr.on('data', (data) => {
  console.error(`ps stderr: ${data}`);
});

ps.on('close', (code) => {
  if (code !== 0) {
    console.log(`ps process exited with code ${code}`);
  }
  grep.stdin.end();
});

grep.stdout.on('data', (data) => {
  console.log(data.toString());
});

grep.stderr.on('data', (data) => {
  console.error(`grep stderr: ${data}`);
});

grep.on('close', (code) => {
  if (code !== 0) {
    console.log(`grep process exited with code ${code}`);
  }
});
```

检查 `spawn` 失败的示例：

```cjs
const { spawn } = require('node:child_process');
const subprocess = spawn('bad_command');

subprocess.on('error', (err) => {
  console.error('Failed to start subprocess.');
});
```

```mjs
import { spawn } from 'node:child_process';
const subprocess = spawn('bad_command');

subprocess.on('error', (err) => {
  console.error('Failed to start subprocess.');
});
```

某些平台（macOS、Linux）将使用 `argv[0]` 的值作为进程标题，而其他平台（Windows、SunOS）将使用 `command`。

Node.js 在启动时使用 `process.execPath` 覆盖 `argv[0]`，因此 Node.js 子进程中的 `process.argv[0]` 将与从父进程传递给 `spawn` 的 `argv0` 参数不匹配。请改用 `process.argv0` 属性检索它。

如果启用了 `signal` 选项，在相应的 `AbortController` 上调用 `.abort()` 类似于在子进程上调用 `.kill()`，不同之处在于传递给回调的错误将是一个 `AbortError`：

```cjs
const { spawn } = require('node:child_process');
const controller = new AbortController();
const { signal } = controller;
const grep = spawn('grep', ['ssh'], { signal });
grep.on('error', (err) => {
  // 如果控制器中止，这将与 err 一起被调用，err 是一个 AbortError
});
controller.abort(); // 停止子进程
```

```mjs
import { spawn } from 'node:child_process';
const controller = new AbortController();
const { signal } = controller;
const grep = spawn('grep', ['ssh'], { signal });
grep.on('error', (err) => {
  // 如果控制器中止，这将与 err 一起被调用，err 是一个 AbortError
});
controller.abort(); // 停止子进程
```

#### `options.detached`

<!-- YAML
added: v0.7.10
-->

在 Windows 上，将 `options.detached` 设置为 `true` 使得子进程在父进程退出后可以继续运行。子进程将拥有自己的控制台窗口。一旦为子进程启用，就无法禁用。

在非 Windows 平台上，如果 `options.detached` 设置为 `true`，则子进程将成为新进程组和会话的领导者。无论子进程是否分离，它们都可能在父进程退出后继续运行。有关更多信息，请参见 setsid(2)。

默认情况下，父进程将等待分离的子进程退出。为了防止父进程等待给定的 `subprocess` 退出，请使用 `subprocess.unref()` 方法。这样做将导致父进程的事件循环不将子进程包括在其引用计数中，允许父进程独立于子进程退出，除非在子进程和父进程之间建立了 IPC 通道。

当使用 `detached` 选项启动长时间运行的进程时，该进程在父进程退出后不会在后台继续运行，除非它提供了一个未连接到父进程的 `stdio` 配置。如果父进程的 `stdio` 被继承，则子进程将保持附加到控制终端。

长时间运行进程的示例，通过分离并忽略其父进程的 `stdio` 文件描述符，以忽略父进程的终止：

```cjs
const { spawn } = require('node:child_process');
const process = require('node:process');

const subprocess = spawn(process.argv[0], ['child_program.js'], {
  detached: true,
  stdio: 'ignore',
});

subprocess.unref();
```

```mjs
import { spawn } from 'node:child_process';
import process from 'node:process';

const subprocess = spawn(process.argv[0], ['child_program.js'], {
  detached: true,
  stdio: 'ignore',
});

subprocess.unref();
```

或者，可以将子进程的输出重定向到文件中：

```cjs
const { openSync } = require('node:fs');
const { spawn } = require('node:child_process');
const out = openSync('./out.log', 'a');
const err = openSync('./out.log', 'a');

const subprocess = spawn('prg', [], {
  detached: true,
  stdio: [ 'ignore', out, err ],
});

subprocess.unref();
```

```mjs
import { openSync } from 'node:fs';
import { spawn } from 'node:child_process';
const out = openSync('./out.log', 'a');
const err = openSync('./out.log', 'a');

const subprocess = spawn('prg', [], {
  detached: true,
  stdio: [ 'ignore', out, err ],
});

subprocess.unref();
```

#### `options.stdio`

<!-- YAML
added: v0.7.10
changes:
  - version:
      - v15.6.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/29412
    description: Added the `overlapped` stdio flag.
  - version: v3.3.1
    pr-url: https://github.com/nodejs/node/pull/2727
    description: The value `0` is now accepted as a file descriptor.
-->

`options.stdio` 选项用于配置在父进程和子进程之间建立的管道。默认情况下，子进程的 stdin、stdout 和 stderr 被重定向到 [`ChildProcess`][] 对象上相应的 [`subprocess.stdin`][]、[`subprocess.stdout`][] 和 [`subprocess.stderr`][] 流。这相当于将 `options.stdio` 设置为 `['pipe', 'pipe', 'pipe']`。

为方便起见，`options.stdio` 可以是以下字符串之一：

* `'pipe'`：相当于 `['pipe', 'pipe', 'pipe']`（默认值）
* `'overlapped'`：相当于 `['overlapped', 'overlapped', 'overlapped']`
* `'ignore'`：相当于 `['ignore', 'ignore', 'ignore']`
* `'inherit'`：相当于 `['inherit', 'inherit', 'inherit']` 或 `[0, 1, 2]`

否则，`options.stdio` 的值是一个数组，其中每个索引对应于子进程中的一个 fd。fd 0、1 和 2 分别对应于 stdin、stdout 和 stderr。可以指定额外的 fd 以在父进程和子进程之间创建额外的管道。该值是以下之一：

1. `'pipe'`：在子进程和父进程之间创建一个管道。管道的父端作为 `child_process` 对象上的属性 [`subprocess.stdio[fd]`][`subprocess.stdio`] 暴露给父进程。为 fd 0、1 和 2 创建的管道也可分别作为 [`subprocess.stdin`][]、[`subprocess.stdout`][] 和 [`subprocess.stderr`][] 使用。这些不是实际的 Unix 管道，因此子进程不能通过它们的描述符文件使用它们，例如 `/dev/fd/2` 或 `/dev/stdout`。
2. `'overlapped'`：与 `'pipe'` 相同，只是在句柄上设置了 `FILE_FLAG_OVERLAPPED` 标志。这对于子进程 stdio 句柄上的重叠 I/O 是必需的。有关更多详细信息，请参阅[文档](https://docs.microsoft.com/en-us/windows/win32/fileio/synchronous-and-asynchronous-i-o)。在非 Windows 系统上，这与 `'pipe'` 完全相同。
3. `'ipc'`：创建一个 IPC 通道，用于在父进程和子进程之间传递消息/文件描述符。一个 [`ChildProcess`][] 最多可以有一个 IPC stdio 文件描述符。设置此选项会启用 [`subprocess.send()`][] 方法。如果子进程是 Node.js 实例，则 IPC 通道的存在将启用 [`process.send()`][] 和 [`process.disconnect()`][] 方法，以及子进程内的 [`'disconnect'`][] 和 [`'message'`][] 事件。

   除了 [`process.send()`][] 之外，以任何其他方式访问 IPC 通道 fd 或与不是 Node.js 实例的子进程一起使用 IPC 通道是不支持的。
4. `'ignore'`：指示 Node.js 忽略子进程中的 fd。虽然 Node.js 将始终为它生成的进程打开 fd 0、1 和 2，但将 fd 设置为 `'ignore'` 将导致 Node.js 打开 `/dev/null` 并将其附加到子进程的 fd。
5. `'inherit'`：将相应的 stdio 流传递给父进程/从父进程传递。在前三个位置，这分别相当于 `process.stdin`、`process.stdout` 和 `process.stderr`。在任何其他位置，相当于 `'ignore'`。
6. {Stream} 对象：与子进程共享一个引用 tty、文件、套接字或管道的可读或可写流。流的底层文件描述符在子进程中复制到与 `stdio` 数组中的索引对应的 fd。流必须具有底层描述符（文件流在 `'open'` 事件发生之前不会开始）。
   **注意：** 虽然从技术上讲可以将 `stdin` 作为可写流传递，或将 `stdout`/`stderr` 作为可读流传递，但不建议这样做。可读和可写流设计具有不同的行为，如果错误地使用它们（例如，在需要可写流的地方传递可读流）可能导致意外结果或错误。不鼓励这种做法，因为如果流遇到错误，可能会导致未定义的行为或丢弃回调。始终确保 `stdin` 用作可读流，`stdout`/`stderr` 用作可写流，以维持父进程和子进程之间的预期数据流。
7. 正整数：整数值被解释为在父进程中打开的文件描述符。它与子进程共享，类似于 {Stream} 对象可以共享的方式。在 Windows 上不支持传递套接字。
8. `null`、`undefined`：使用默认值。对于 stdio fd 0、1 和 2（换句话说，stdin、stdout 和 stderr）会创建一个管道。对于 fd 3 及更高，默认是 `'ignore'`。

```cjs
const { spawn } = require('node:child_process');
const process = require('node:process');

// 子进程将使用父进程的 stdio。
spawn('prg', [], { stdio: 'inherit' });

// 仅共享 stderr 生成子进程。
spawn('prg', [], { stdio: ['pipe', 'pipe', process.stderr] });

// 打开一个额外的 fd=4，与呈现 startd 样式接口的程序交互。
spawn('prg', [], { stdio: ['pipe', null, null, null, 'pipe'] });
```

```mjs
import { spawn } from 'node:child_process';
import process from 'node:process';

// 子进程将使用父进程的 stdio。
spawn('prg', [], { stdio: 'inherit' });

// 仅共享 stderr 生成子进程。
spawn('prg', [], { stdio: ['pipe', 'pipe', process.stderr] });

// 打开一个额外的 fd=4，与呈现 startd 样式接口的程序交互。
spawn('prg', [], { stdio: ['pipe', null, null, null, 'pipe'] });
```

_值得注意的是，当在父进程和子进程之间建立 IPC 通道，并且子进程是 Node.js 实例时，子进程启动时 IPC 通道是未引用的（使用 `unref()`），直到子进程为 [`'disconnect'`][] 事件或 [`'message'`][] 事件注册事件处理程序。这允许子进程正常退出，而不会被打开的 IPC 通道保持进程。_
另请参阅：[`child_process.exec()`][] 和 [`child_process.fork()`][]。

## 同步进程创建

[`child_process.spawnSync()`][]、[`child_process.execSync()`][] 和 [`child_process.execFileSync()`][] 方法是同步的，将阻塞 Node.js 事件循环，暂停任何其他代码的执行，直到生成的进程退出。

像这样的阻塞调用主要用于简化通用脚本任务以及简化应用程序配置在启动时的加载/处理。

### `child_process.execFileSync(file[, args][, options])`

<!-- YAML
added: v0.11.12
changes:
  - version:
      - v16.4.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38862
    description: The `cwd` option can be a WHATWG `URL` object using
                 `file:` protocol.
  - version: v10.10.0
    pr-url: https://github.com/nodejs/node/pull/22409
    description: The `input` option can now be any `TypedArray` or a
                 `DataView`.
  - version: v8.8.0
    pr-url: https://github.com/nodejs/node/pull/15380
    description: The `windowsHide` option is supported now.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10653
    description: The `input` option can now be a `Uint8Array`.
  - version:
    - v6.2.1
    - v4.5.0
    pr-url: https://github.com/nodejs/node/pull/6939
    description: The `encoding` option can now explicitly be set to `buffer`.
-->

* `file` {string} 要运行的可执行文件的名称或路径。
* `args` {string\[]} 字符串参数列表。
* `options` {Object}
  * `cwd` {string|URL} 子进程的当前工作目录。
  * `input` {string|Buffer|TypedArray|DataView} 将作为 stdin 传递给生成的进程的值。如果 `stdio[0]` 设置为 `'pipe'`，提供此值将覆盖 `stdio[0]`。
  * `stdio` {string|Array} 子进程的 stdio 配置。参见 [`child_process.spawn()`][] 的 [`stdio`][]。默认情况下，stderr 将输出到父进程的 stderr，除非指定了 `stdio`。**默认值：** `'pipe'`。
  * `env` {Object} 环境键值对。**默认值：** `process.env`。
  * `uid` {number} 设置进程的用户标识（参见 setuid(2)）。
  * `gid` {number} 设置进程的组标识（参见 setgid(2)）。
  * `timeout` {number} 进程允许运行的最长时间（以毫秒为单位）。**默认值：** `undefined`。
  * `killSignal` {string|integer} 当生成的进程被杀死时使用的信号值。**默认值：** `'SIGTERM'`。
  * `maxBuffer` {number} 允许在 stdout 或 stderr 上的最大数据量（以字节为单位）。如果超出，子进程将被终止。请参阅 [`maxBuffer` 和 Unicode][] 的注意事项。**默认值：** `1024 * 1024`。
  * `encoding` {string} 用于所有 stdio 输入和输出的编码。**默认值：** `'buffer'`。
  * `windowsHide` {boolean} 隐藏通常在 Windows 系统上创建的子进程控制台窗口。**默认值：** `false`。
  * `shell` {boolean|string} 如果为 `true`，则在 shell 内运行 `command`。在 Unix 上使用 `'/bin/sh'`，在 Windows 上使用 `process.env.ComSpec`。可以指定一个不同的 shell 作为字符串。请参阅 [Shell 要求][] 和 [默认 Windows shell][]。**默认值：** `false`（无 shell）。
* 返回：{Buffer|string} 命令的 stdout。

`child_process.execFileSync()` 方法通常与 [`child_process.execFile()`][] 相同，不同之处在于该方法在子进程完全关闭之前不会返回。当遇到超时并发送 `killSignal` 时，该方法在进程完全退出之前不会返回。

如果子进程拦截并处理 `SIGTERM` 信号且未退出，则父进程将仍然等待直到子进程退出。

如果进程超时或具有非零退出代码，则此方法将抛出一个 [`Error`][]，其中将包括底层 [`child_process.spawnSync()`][] 的完整结果。

**如果启用了 `shell` 选项，请勿将未经过滤的用户输入传递给此函数。任何包含 shell 元字符的输入都可能被用来触发任意命令执行。**

```cjs
const { execFileSync } = require('node:child_process');

try {
  const stdout = execFileSync('my-script.sh', ['my-arg'], {
    // 从子进程捕获 stdout 和 stderr。覆盖了将子进程 stderr 流式传输到父进程 stderr 的默认行为
    stdio: 'pipe',

    // 对 stdio 管道使用 utf8 编码
    encoding: 'utf8',
  });

  console.log(stdout);
} catch (err) {
  if (err.code) {
    // 生成子进程失败
    console.error(err.code);
  } else {
    // 子进程已生成但以非零退出代码退出
    // 错误包含来自子进程的任何 stdout 和 stderr
    const { stdout, stderr } = err;

    console.error({ stdout, stderr });
  }
}
```

```mjs
import { execFileSync } from 'node:child_process';

try {
  const stdout = execFileSync('my-script.sh', ['my-arg'], {
    // 从子进程捕获 stdout 和 stderr。覆盖了将子进程 stderr 流式传输到父进程 stderr 的默认行为
    stdio: 'pipe',

    // 对 stdio 管道使用 utf8 编码
    encoding: 'utf8',
  });

  console.log(stdout);
} catch (err) {
  if (err.code) {
    // 生成子进程失败
    console.error(err.code);
  } else {
    // 子进程已生成但以非零退出代码退出
    // 错误包含来自子进程的任何 stdout 和 stderr
    const { stdout, stderr } = err;

    console.error({ stdout, stderr });
  }
}
```

### `child_process.execSync(command[, options])`

<!-- YAML
added: v0.11.12
changes:
  - version:
      - v16.4.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38862
    description: The `cwd` option can be a WHATWG `URL` object using
                 `file:` protocol.
  - version: v10.10.0
    pr-url: https://github.com/nodejs/node/pull/22409
    description: The `input` option can now be any `TypedArray` or a
                 `DataView`.
  - version: v8.8.0
    pr-url: https://github.com/nodejs/node/pull/15380
    description: The `windowsHide` option is supported now.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10653
    description: The `input` option can now be a `Uint8Array`.
-->

* `command` {string} 要运行的命令。
* `options` {Object}
  * `cwd` {string|URL} 子进程的当前工作目录。
  * `input` {string|Buffer|TypedArray|DataView} 将作为 stdin 传递给生成的进程的值。如果 `stdio[0]` 设置为 `'pipe'`，提供此值将覆盖 `stdio[0]`。
  * `stdio` {string|Array} 子进程的 stdio 配置。参见 [`child_process.spawn()`][] 的 [`stdio`][]。默认情况下，stderr 将输出到父进程的 stderr，除非指定了 `stdio`。**默认值：** `'pipe'`。
  * `env` {Object} 环境键值对。**默认值：** `process.env`。
  * `shell` {string} 用于执行命令的 shell。请参阅 [Shell 要求][] 和 [默认 Windows shell][]。**默认值：** 在 Unix 上是 `'/bin/sh'`，在 Windows 上是 `process.env.ComSpec`。
  * `uid` {number} 设置进程的用户标识。（参见 setuid(2)）。
  * `gid` {number} 设置进程的组标识。（参见 setgid(2)）。
  * `timeout` {number} 进程允许运行的最长时间（以毫秒为单位）。**默认值：** `undefined`。
  * `killSignal` {string|integer} 当生成的进程被杀死时使用的信号值。**默认值：** `'SIGTERM'`。
  * `maxBuffer` {number} 允许在 stdout 或 stderr 上的最大数据量（以字节为单位）。如果超出，子进程将被终止，并且任何输出都会被截断。请参阅 [`maxBuffer` 和 Unicode][] 的注意事项。**默认值：** `1024 * 1024`。
  * `encoding` {string} 用于所有 stdio 输入和输出的编码。**默认值：** `'buffer'`。
  * `windowsHide` {boolean} 隐藏通常在 Windows 系统上创建的子进程控制台窗口。**默认值：** `false`。
* 返回：{Buffer|string} 命令的 stdout。

`child_process.execSync()` 方法通常与 [`child_process.exec()`][] 相同，不同之处在于该方法在子进程完全关闭之前不会返回。当遇到超时并发送 `killSignal` 时，该方法在进程完全退出之前不会返回。如果子进程拦截并处理 `SIGTERM` 信号且未退出，则父进程将等待直到子进程退出。

如果进程超时或具有非零退出代码，则此方法将抛出错误。[`Error`][] 对象将包含来自 [`child_process.spawnSync()`][] 的整个结果。

**切勿将未经过滤的用户输入传递给此函数。任何包含 shell 元字符的输入都可能被用来触发任意命令执行。**

### `child_process.spawnSync(command[, args][, options])`

<!-- YAML
added: v0.11.12
changes:
  - version:
      - v16.4.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38862
    description: The `cwd` option can be a WHATWG `URL` object using
                 `file:` protocol.
  - version: v10.10.0
    pr-url: https://github.com/nodejs/node/pull/22409
    description: The `input` option can now be any `TypedArray` or a
                 `DataView`.
  - version: v8.8.0
    pr-url: https://github.com/nodejs/node/pull/15380
    description: The `windowsHide` option is supported now.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/10653
    description: The `input` option can now be a `Uint8Array`.
  - version:
    - v6.2.1
    - v4.5.0
    pr-url: https://github.com/nodejs/node/pull/6939
    description: The `encoding` option can now explicitly be set to `buffer`.
  - version: v5.7.0
    pr-url: https://github.com/nodejs/node/pull/4598
    description: The `shell` option is supported now.
-->

* `command` {string} 要运行的命令。
* `args` {string\[]} 字符串参数列表。
* `options` {Object}
  * `cwd` {string|URL} 子进程的当前工作目录。
  * `input` {string|Buffer|TypedArray|DataView} 将作为 stdin 传递给生成的进程的值。如果 `stdio[0]` 设置为 `'pipe'`，提供此值将覆盖 `stdio[0]`。
  * `argv0` {string} 显式设置发送给子进程的 `argv[0]` 的值。如果未指定，这将设置为 `command`。
  * `stdio` {string|Array} 子进程的 stdio 配置。参见 [`child_process.spawn()`][] 的 [`stdio`][]。**默认值：** `'pipe'`。
  * `env` {Object} 环境键值对。**默认值：** `process.env`。
  * `uid` {number} 设置进程的用户标识（参见 setuid(2)）。
  * `gid` {number} 设置进程的组标识（参见 setgid(2)）。
  * `timeout` {number} 进程允许运行的最长时间（以毫秒为单位）。**默认值：** `undefined`。
  * `killSignal` {string|integer} 当生成的进程被杀死时使用的信号值。**默认值：** `'SIGTERM'`。
  * `maxBuffer` {number} 允许在 stdout 或 stderr 上的最大数据量（以字节为单位）。如果超出，子进程将被终止，并且任何输出都会被截断。请参阅 [`maxBuffer` 和 Unicode][] 的注意事项。**默认值：** `1024 * 1024`。
  * `encoding` {string} 用于所有 stdio 输入和输出的编码。**默认值：** `'buffer'`。
  * `shell` {boolean|string} 如果为 `true`，则在 shell 内运行 `command`。在 Unix 上使用 `'/bin/sh'`，在 Windows 上使用 `process.env.ComSpec`。可以指定一个不同的 shell 作为字符串。请参阅 [Shell 要求][] 和 [默认 Windows shell][]。**默认值：** `false`（无 shell）。
  * `windowsVerbatimArguments` {boolean} 在 Windows 上不对参数进行引号或转义处理。在 Unix 上被忽略。当指定 `shell` 且为 CMD 时，此选项自动设置为 `true`。**默认值：** `false`。
  * `windowsHide` {boolean} 隐藏通常在 Windows 系统上创建的子进程控制台窗口。**默认值：** `false`。
* 返回：{Object}
  * `pid` {number} 子进程的 pid。
  * `output` {Array} stdio 输出结果数组。
  * `stdout` {Buffer|string} `output[1]` 的内容。
  * `stderr` {Buffer|string} `output[2]` 的内容。
  * `status` {number|null} 子进程的退出代码，如果子进程因信号终止则为 `null`。
  * `signal` {string|null} 用于杀死子进程的信号，如果子进程不是因信号终止则为 `null`。
  * `error` {Error} 如果子进程失败或超时，则为错误对象。

`child_process.spawnSync()` 方法通常与 [`child_process.spawn()`][] 相同，不同之处在于该函数在子进程完全关闭之前不会返回。当遇到超时并发送 `killSignal` 时，该方法在进程完全退出之前不会返回。如果进程拦截并处理 `SIGTERM` 信号且未退出，则父进程将等待直到子进程退出。

**如果启用了 `shell` 选项，请勿将未经过滤的用户输入传递给此函数。任何包含 shell 元字符的输入都可能被用来触发任意命令执行。**

## 类：`ChildProcess`

<!-- YAML
added: v2.2.0
-->

* 扩展：{EventEmitter}

`ChildProcess` 的实例表示生成的子进程。

`ChildProcess` 的实例不打算直接创建。相反，使用 [`child_process.spawn()`][]、[`child_process.exec()`][]、[`child_process.execFile()`][] 或 [`child_process.fork()`][] 方法来创建 `ChildProcess` 的实例。

### 事件：`'close'`

<!-- YAML
added: v0.7.7
-->

* `code` {number} 如果子进程自行退出，则为退出代码，如果子进程因信号终止则为 `null`。
* `signal` {string} 用于终止子进程的信号，如果子进程不是因信号终止则为 `null`。

当进程结束_且_子进程的 stdio 流已关闭时，会发出 `'close'` 事件。这与 [`'exit'`][] 事件不同，因为多个进程可能共享相同的 stdio 流。`'close'` 事件将在 [`'exit'`][] 已经发出之后总是发出，或者如果子进程未能生成则 [`'error'`][] 之后发出。

如果进程退出，`code` 是进程的最终退出代码，否则为 `null`。如果进程因接收信号而终止，`signal` 是信号的字符串名称，否则为 `null`。两者之一将总是非 `null`。

```cjs
const { spawn } = require('node:child_process');
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process close all stdio with code ${code}`);
});

ls.on('exit', (code) => {
  console.log(`child process exited with code ${code}`);
});
```

```mjs
import { spawn } from 'node:child_process';
const ls = spawn('ls', ['-lh', '/usr']);

ls.stdout.on('data', (data) => {
  console.log(`stdout: ${data}`);
});

ls.on('close', (code) => {
  console.log(`child process close all stdio with code ${code}`);
});

ls.on('exit', (code) => {
  console.log(`child process exited with code ${code}`);
});
```

### 事件：`'disconnect'`

<!-- YAML
added: v0.7.2
-->

在父进程中调用 [`subprocess.disconnect()`][] 方法或在子进程中调用 [`process.disconnect()`][] 后会发出 `'disconnect'` 事件。断开连接后，不再可能发送或接收消息，并且 [`subprocess.connected`][] 属性为 `false`。

### 事件：`'error'`

* `err` {Error} 错误。

在以下情况下会发出 `'error'` 事件：

* 进程无法生成。
* 进程无法被杀死。
* 向子进程发送消息失败。
* 子进程通过 `signal` 选项被中止。

在错误发生后，`'exit'` 事件可能会也可能不会触发。当同时监听 `'exit'` 和 `'error'` 事件时，请防止意外多次调用处理函数。

另请参阅 [`subprocess.kill()`][] 和 [`subprocess.send()`][]。

### 事件：`'exit'`

<!-- YAML
added: v0.1.90
-->

* `code` {number} 如果子进程自行退出，则为退出代码，如果子进程因信号终止则为 `null`。
* `signal` {string} 用于终止子进程的信号，如果子进程不是因信号终止则为 `null`。

在子进程结束后发出 `'exit'` 事件。如果进程退出，`code` 是进程的最终退出代码，否则为 `null`。如果进程因接收信号而终止，`signal` 是信号的字符串名称，否则为 `null`。两者之一将总是非 `null`。

当触发 `'exit'` 事件时，子进程 stdio 流可能仍然打开。

Node.js 为 `SIGINT` 和 `SIGTERM` 建立信号处理程序，Node.js 进程不会因接收到这些信号而立即终止。相反，Node.js 将执行一系列清理操作，然后重新引发被处理的信号。

参见 waitpid(2)。

### 事件：`'message'`

<!-- YAML
added: v0.5.9
-->

* `message` {Object} 解析后的 JSON 对象或原始值。
* `sendHandle` {Handle|undefined} `undefined` 或一个 [`net.Socket`][]、[`net.Server`][] 或 [`dgram.Socket`][] 对象。

当子进程使用 [`process.send()`][] 发送消息时，会触发 `'message'` 事件。

消息经过序列化和解析。生成的消息可能与最初发送的消息不同。

如果在生成子进程时使用的 `serialization` 选项设置为 `'advanced'`，则 `message` 参数可以包含 JSON 无法表示的数据。有关更多详细信息，请参阅 [高级序列化][]。

### 事件：`'spawn'`

<!-- YAML
added:
  - v15.1.0
  - v14.17.0
-->

当子进程成功生成时，会发出 `'spawn'` 事件。如果子进程未能成功生成，则不会发出 `'spawn'` 事件，而是发出 `'error'` 事件。

如果发出，`'spawn'` 事件会在所有其他事件之前发出，并且在通过 `stdout` 或 `stderr` 接收任何数据之前发出。

无论生成进程**内部**是否发生错误，`'spawn'` 事件都会触发。例如，如果 `bash some-command` 成功生成，则 `'spawn'` 事件将触发，尽管 `bash` 可能无法生成 `some-command`。当使用 `{ shell: true }` 时，此注意事项也适用。

### `subprocess.channel`

<!-- YAML
added: v7.1.0
changes:
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/30165
    description: The object no longer accidentally exposes native C++ bindings.
-->

* 类型：{Object} 表示与子进程的 IPC 通道的管道。

`subprocess.channel` 属性是对子进程的 IPC 通道的引用。如果不存在 IPC 通道，则此属性为 `undefined`。

#### `subprocess.channel.ref()`

<!-- YAML
added: v7.1.0
-->

如果之前调用过 `.unref()`，此方法会使 IPC 通道保持父进程的事件循环运行。

#### `subprocess.channel.unref()`

<!-- YAML
added: v7.1.0
-->

此方法使 IPC 通道不保持父进程的事件循环运行，并允许其在通道打开时完成。

### `subprocess.connected`

<!-- YAML
added: v0.7.2
-->

* 类型：{boolean} 在调用 `subprocess.disconnect()` 后设置为 `false`。

`subprocess.connected` 属性指示是否仍然可以从子进程发送和接收消息。当 `subprocess.connected` 为 `false` 时，不再可能发送或接收消息。

### `subprocess.disconnect()`

<!-- YAML
added: v0.7.2
-->

关闭父进程和子进程之间的 IPC 通道，允许子进程在没有其他连接保持其活动状态时正常退出。调用此方法后，父进程和子进程中的 `subprocess.connected` 和 `process.connected` 属性（分别）将被设置为 `false`，并且不再可能在进程之间传递消息。

当没有正在接收的消息时，将发出 `'disconnect'` 事件。这通常会在调用 `subprocess.disconnect()` 后立即触发。

当子进程是 Node.js 实例（例如，使用 [`child_process.fork()`][] 生成）时，可以在子进程内调用 `process.disconnect()` 方法来关闭 IPC 通道。

### `subprocess.exitCode`

* 类型：{integer}

`subprocess.exitCode` 属性指示子进程的退出代码。如果子进程仍在运行，则该字段将为 `null`。

### `subprocess.kill([signal])`

<!-- YAML
added: v0.1.90
-->

* `signal` {number|string}
* 返回：{boolean}

`subprocess.kill()` 方法向子进程发送信号。如果未提供参数，则进程将发送 `'SIGTERM'` 信号。有关可用信号的列表，请参见 signal(7)。如果 kill(2) 成功，则此函数返回 `true`，否则返回 `false`。

```cjs
const { spawn } = require('node:child_process');
const grep = spawn('grep', ['ssh']);

grep.on('close', (code, signal) => {
  console.log(
    `child process terminated due to receipt of signal ${signal}`);
});

// 向进程发送 SIGHUP。
grep.kill('SIGHUP');
```

```mjs
import { spawn } from 'node:child_process';
const grep = spawn('grep', ['ssh']);

grep.on('close', (code, signal) => {
  console.log(
    `child process terminated due to receipt of signal ${signal}`);
});

// 向进程发送 SIGHUP。
grep.kill('SIGHUP');
```

如果无法传递信号，[`ChildProcess`][] 对象可能会发出 [`'error'`][] 事件。向已经退出的子进程发送信号不是错误，但可能会产生无法预见的后果。特别是，如果进程标识符（PID）已被重新分配给另一个进程，则信号将传递给该进程，这可能导致意外结果。

虽然该函数被称为 `kill`，但传递给子进程的信号可能实际上并不会终止该进程。

有关参考，请参见 kill(2)。

在不存在 POSIX 信号的 Windows 上，`signal` 参数将被忽略，除了 `'SIGKILL'`、`'SIGTERM'`、`'SIGINT'` 和 `'SIGQUIT'`，进程将总是被强制且突然地杀死（类似于 `'SIGKILL'`）。有关更多详细信息，请参阅[信号事件][]。

在 Linux 上，当尝试杀死其父进程时，子进程的子进程不会被终止。这可能在 shell 中运行新进程或使用 `ChildProcess` 的 `shell` 选项时发生：

```cjs
const { spawn } = require('node:child_process');

const subprocess = spawn(
  'sh',
  [
    '-c',
    `node -e "setInterval(() => {
      console.log(process.pid, 'is alive')
    }, 500);"`,
  ], {
    stdio: ['inherit', 'inherit', 'inherit'],
  },
);

setTimeout(() => {
  subprocess.kill(); // 不会终止 shell 中的 Node.js 进程。
}, 2000);
```

```mjs
import { spawn } from 'node:child_process';

const subprocess = spawn(
  'sh',
  [
    '-c',
    `node -e "setInterval(() => {
      console.log(process.pid, 'is alive')
    }, 500);"`,
  ], {
    stdio: ['inherit', 'inherit', 'inherit'],
  },
);

setTimeout(() => {
  subprocess.kill(); // 不会终止 shell 中的 Node.js 进程。
}, 2000);
```

### `subprocess[Symbol.dispose]()`

<!-- YAML
added:
 - v20.5.0
 - v18.18.0
changes:
 - version: v24.2.0
   pr-url: https://github.com/nodejs/node/pull/58467
   description: No longer experimental.
-->

使用 `'SIGTERM'` 调用 [`subprocess.kill()`][]。

### `subprocess.killed`

<!-- YAML
added: v0.5.10
-->

* 类型：{boolean} 在 `subprocess.kill()` 用于成功向子进程发送信号后设置为 `true`。

`subprocess.killed` 属性指示是否已成功从 `subprocess.kill()` 向子进程发送信号。`killed` 属性不表示子进程已被终止。

### `subprocess.pid`

<!-- YAML
added: v0.1.90
-->

* 类型：{integer|undefined}

返回子进程的进程标识符（PID）。如果子进程因错误而未能生成，则该值为 `undefined` 并发出 `error`。

```cjs
const { spawn } = require('node:child_process');
const grep = spawn('grep', ['ssh']);

console.log(`Spawned child pid: ${grep.pid}`);
grep.stdin.end();
```

```mjs
import { spawn } from 'node:child_process';
const grep = spawn('grep', ['ssh']);

console.log(`Spawned child pid: ${grep.pid}`);
grep.stdin.end();
```

### `subprocess.ref()`

<!-- YAML
added: v0.7.10
-->

在调用 `subprocess.unref()` 后调用 `subprocess.ref()` 将恢复已移除的子进程的引用计数，强制父进程在退出自身之前等待子进程退出。

```cjs
const { spawn } = require('node:child_process');
const process = require('node:process');

const subprocess = spawn(process.argv[0], ['child_program.js'], {
  detached: true,
  stdio: 'ignore',
});

subprocess.unref();
subprocess.ref();
```

```mjs
import { spawn } from 'node:child_process';
import process from 'node:process';

const subprocess = spawn(process.argv[0], ['child_program.js'], {
  detached: true,
  stdio: 'ignore',
});

subprocess.unref();
subprocess.ref();
```

### `subprocess.send(message[, sendHandle[, options]][, callback])`

<!-- YAML
added: v0.5.9
changes:
  - version: v5.8.0
    pr-url: https://github.com/nodejs/node/pull/5283
    description: The `options` parameter, and the `keepOpen` option
                 in particular, is supported now.
  - version: v5.0.0
    pr-url: https://github.com/nodejs/node/pull/3516
    description: This method returns a boolean for flow control now.
  - version: v4.0.0
    pr-url: https://github.com/nodejs/node/pull/2620
    description: The `callback` parameter is supported now.
-->

* `message` {Object}
* `sendHandle` {Handle|undefined} `undefined`，或一个 [`net.Socket`][]、[`net.Server`][] 或 [`dgram.Socket`][] 对象。
* `options` {Object} 如果存在，`options` 参数是一个用于参数化某些类型句柄的发送的对象。`options` 支持以下属性：
  * `keepOpen` {boolean} 在传递 `net.Socket` 实例时可以使用的值。当为 `true` 时，套接字在发送过程中保持打开状态。**默认值：** `false`。
* `callback` {Function}
* 返回：{boolean}

当在父进程和子进程之间建立了 IPC 通道时（即，当使用 [`child_process.fork()`][] 时），可以使用 `subprocess.send()` 方法向子进程发送消息。当子进程是 Node.js 实例时，可以通过 [`'message'`][] 事件接收这些消息。

消息经过序列化和解析。生成的消息可能与最初发送的消息不同。

例如，在父脚本中：

```cjs
const { fork } = require('node:child_process');
const forkedProcess = fork(`${__dirname}/sub.js`);

forkedProcess.on('message', (message) => {
  console.log('PARENT got message:', message);
});

// 导致子进程打印：CHILD got message: { hello: 'world' }
forkedProcess.send({ hello: 'world' });
```

```mjs
import { fork } from 'node:child_process';
const forkedProcess = fork(`${import.meta.dirname}/sub.js`);

forkedProcess.on('message', (message) => {
  console.log('PARENT got message:', message);
});

// 导致子进程打印：CHILD got message: { hello: 'world' }
forkedProcess.send({ hello: 'world' });
```

然后子脚本 `'sub.js'` 可能如下所示：

```js
process.on('message', (message) => {
  console.log('CHILD got message:', message);
});

// 导致父进程打印：PARENT got message: { foo: 'bar', baz: null }
process.send({ foo: 'bar', baz: NaN });
```

子 Node.js 进程将有自己的 [`process.send()`][] 方法，允许子进程向父进程发送消息。

当发送 `{cmd: 'NODE_foo'}` 消息时有一个特殊情况。包含 `cmd` 属性中带有 `NODE_` 前缀的消息保留供 Node.js 核心内部使用，并且不会在子进程的 [`'message'`][] 事件中发出。相反，此类消息使用 `'internalMessage'` 事件发出，并由 Node.js 内部使用。应用程序应避免使用此类消息或监听 `'internalMessage'` 事件，因为它可能在不通知的情况下更改。

可以传递给 `subprocess.send()` 的可选 `sendHandle` 参数用于将 TCP 服务器或套接字对象传递给子进程。子进程将接收该对象作为传递给在 [`'message'`][] 事件上注册的回调函数的第二个参数。在套接字中接收和缓冲的任何数据都不会发送给子进程。不支持在 Windows 上发送 IPC 套接字。

可选的 `callback` 是一个函数，在消息发送之后但在子进程可能收到它之前调用。该函数使用单个参数调用：成功时为 `null`，失败时为 [`Error`][] 对象。

如果未提供 `callback` 函数且无法发送消息，则 [`ChildProcess`][] 对象将发出 `'error'` 事件。例如，当子进程已经退出时，可能会发生这种情况。

`subprocess.send()` 将在通道已关闭或未发送消息的积压超过使得发送更多消息不明智的阈值时返回 `false`。否则，该方法返回 `true`。`callback` 函数可用于实现流量控制。

#### 示例：发送服务器对象

`sendHandle` 参数可以用于，例如，将 TCP 服务器对象的句柄传递给子进程，如下例所示：

```cjs
const { fork } = require('node:child_process');
const { createServer } = require('node:net');

const subprocess = fork('subprocess.js');

// 打开服务器对象并发送句柄。
const server = createServer();
server.on('connection', (socket) => {
  socket.end('handled by parent');
});
server.listen(1337, () => {
  subprocess.send('server', server);
});
```

```mjs
import { fork } from 'node:child_process';
import { createServer } from 'node:net';

const subprocess = fork('subprocess.js');

// 打开服务器对象并发送句柄。
const server = createServer();
server.on('connection', (socket) => {
  socket.end('handled by parent');
});
server.listen(1337, () => {
  subprocess.send('server', server);
});
```

然后子进程将接收服务器对象作为：

```js
process.on('message', (m, server) => {
  if (m === 'server') {
    server.on('connection', (socket) => {
      socket.end('handled by child');
    });
  }
});
```

一旦服务器现在在父进程和子进程之间共享，一些连接可以由父进程处理，一些由子进程处理。

虽然上面的示例使用 `node:net` 模块创建的服务器，但 `node:dgram` 模块服务器使用完全相同的工作流程，除了监听 `'message'` 事件而不是 `'connection'` 事件并使用 `server.bind()` 而不是 `server.listen()`。但是，这仅在 Unix 平台上受支持。

#### 示例：发送套接字对象

类似地，`sendHandler` 参数可用于传递套接字的句柄给子进程。下面的示例生成两个子进程，每个处理具有“正常”或“特殊”优先级的连接：

```cjs
const { fork } = require('node:child_process');
const { createServer } = require('node:net');

const normal = fork('subprocess.js', ['normal']);
const special = fork('subprocess.js', ['special']);

// 打开服务器并将套接字发送给子进程。使用 pauseOnConnect 防止套接字在发送给子进程之前被读取。
const server = createServer({ pauseOnConnect: true });
server.on('connection', (socket) => {

  // 如果这是特殊优先级...
  if (socket.remoteAddress === '74.125.127.100') {
    special.send('socket', socket);
    return;
  }
  // 这是正常优先级。
  normal.send('socket', socket);
});
server.listen(1337);
```

```mjs
import { fork } from 'node:child_process';
import { createServer } from 'node:net';

const normal = fork('subprocess.js', ['normal']);
const special = fork('subprocess.js', ['special']);

// 打开服务器并将套接字发送给子进程。使用 pauseOnConnect 防止套接字在发送给子进程之前被读取。
const server = createServer({ pauseOnConnect: true });
server.on('connection', (socket) => {

  // 如果这是特殊优先级...
  if (socket.remoteAddress === '74.125.127.100') {
    special.send('socket', socket);
    return;
  }
  // 这是正常优先级。
  normal.send('socket', socket);
});
server.listen(1337);
```

`subprocess.js` 将接收套接字句柄作为传递给事件回调函数的第二个参数：

```js
process.on('message', (m, socket) => {
  if (m === 'socket') {
    if (socket) {
      // 检查客户端套接字是否存在。
      // 套接字在发送和子进程接收之间可能被关闭。
      socket.end(`Request handled with ${process.argv[2]} priority`);
    }
  }
});
```

不要在对已传递给子进程的套接字上使用 `.maxConnections`。父进程无法跟踪套接字何时被销毁。

子进程中的任何 `'message'` 处理程序都应验证 `socket` 是否存在，因为连接在发送到子进程的过程中可能已被关闭。

### `subprocess.signalCode`

* 类型：{string|null}

`subprocess.signalCode` 属性指示子进程接收到的信号（如果有），否则为 `null`。

### `subprocess.spawnargs`

* 类型：{Array}

`subprocess.spawnargs` 属性表示启动子进程时使用的完整命令行参数列表。

### `subprocess.spawnfile`

* 类型：{string}

`subprocess.spawnfile` 属性指示启动的子进程的可执行文件名。

对于 [`child_process.fork()`][]，其值将等于 [`process.execPath`][]。
对于 [`child_process.spawn()`][]，其值将是可执行文件的名称。
对于 [`child_process.exec()`][]，其值将是启动子进程的 shell 的名称。

### `subprocess.stderr`

<!-- YAML
added: v0.1.90
-->

* 类型：{stream.Readable|null|undefined}

表示子进程的 `stderr` 的 `Readable Stream`。

如果子进程的 `stdio[2]` 设置为除 `'pipe'` 之外的任何值，则此值为 `null`。

`subprocess.stderr` 是 `subprocess.stdio[2]` 的别名。两个属性将引用相同的值。

如果子进程无法成功生成，则 `subprocess.stderr` 属性可以为 `null` 或 `undefined`。

### `subprocess.stdin`

<!-- YAML
added: v0.1.90
-->

* 类型：{stream.Writable|null|undefined}

表示子进程的 `stdin` 的 `Writable Stream`。

如果子进程等待读取所有输入，则子进程在此流通过 `end()` 关闭之前不会继续。

如果子进程的 `stdio[0]` 设置为除 `'pipe'` 之外的任何值，则此值为 `null`。

`subprocess.stdin` 是 `subprocess.stdio[0]` 的别名。两个属性将引用相同的值。

如果子进程无法成功生成，则 `subprocess.stdin` 属性可以为 `null` 或 `undefined`。

### `subprocess.stdio`

<!-- YAML
added: v0.7.10
-->

* 类型：{Array}

一个稀疏数组，包含到子进程的管道，对应于传递给 [`child_process.spawn()`][] 的 [`stdio`][] 选项中已设置为值 `'pipe'` 的位置。`subprocess.stdio[0]`、`subprocess.stdio[1]` 和 `subprocess.stdio[2]` 也分别可用作 `subprocess.stdin`、`subprocess.stdout` 和 `subprocess.stderr`。

在以下示例中，只有子进程的 fd `1`（stdout）被配置为管道，因此只有父进程的 `subprocess.stdio[1]` 是流，数组中的所有其他值都是 `null`。

```cjs
const assert = require('node:assert');
const fs = require('node:fs');
const child_process = require('node:child_process');

const subprocess = child_process.spawn('ls', {
  stdio: [
    0, // 使用父进程的 stdin 作为子进程。
    'pipe', // 将子进程的 stdout 管道到父进程。
    fs.openSync('err.out', 'w'), // 将子进程的 stderr 定向到文件。
  ],
});

assert.strictEqual(subprocess.stdio[0], null);
assert.strictEqual(subprocess.stdio[0], subprocess.stdin);

assert(subprocess.stdout);
assert.strictEqual(subprocess.stdio[1], subprocess.stdout);

assert.strictEqual(subprocess.stdio[2], null);
assert.strictEqual(subprocess.stdio[2], subprocess.stderr);
```

```mjs
import assert from 'node:assert';
import fs from 'node:fs';
import child_process from 'node:child_process';

const subprocess = child_process.spawn('ls', {
  stdio: [
    0, // 使用父进程的 stdin 作为子进程。
    'pipe', // 将子进程的 stdout 管道到父进程。
    fs.openSync('err.out', 'w'), // 将子进程的 stderr 定向到文件。
  ],
});

assert.strictEqual(subprocess.stdio[0], null);
assert.strictEqual(subprocess.stdio[0], subprocess.stdin);

assert(subprocess.stdout);
assert.strictEqual(subprocess.stdio[1], subprocess.stdout);

assert.strictEqual(subprocess.stdio[2], null);
assert.strictEqual(subprocess.stdio[2], subprocess.stderr);
```

如果子进程无法成功生成，则 `subprocess.stdio` 属性可以为 `undefined`。

### `subprocess.stdout`

<!-- YAML
added: v0.1.90
-->

* 类型：{stream.Readable|null|undefined}

表示子进程的 `stdout` 的 `Readable Stream`。

如果子进程的 `stdio[1]` 设置为除 `'pipe'` 之外的任何值，则此值为 `null`。

`subprocess.stdout` 是 `subprocess.stdio[1]` 的别名。两个属性将引用相同的值。

```cjs
const { spawn } = require('node:child_process');

const subprocess = spawn('ls');

subprocess.stdout.on('data', (data) => {
  console.log(`Received chunk ${data}`);
});
```

```mjs
import { spawn } from 'node:child_process';

const subprocess = spawn('ls');

subprocess.stdout.on('data', (data) => {
  console.log(`Received chunk ${data}`);
});
```

如果子进程无法成功生成，则 `subprocess.stdout` 属性可以为 `null` 或 `undefined`。

### `subprocess.unref()`

<!-- YAML
added: v0.7.10
-->

默认情况下，父进程将等待分离的子进程退出。为了防止父进程等待给定的 `subprocess` 退出，请使用 `subprocess.unref()` 方法。这样做将导致父进程的事件循环不将子进程包括在其引用计数中，允许父进程独立于子进程退出，除非在子进程和父进程之间建立了 IPC 通道。

```cjs
const { spawn } = require('node:child_process');
const process = require('node:process');

const subprocess = spawn(process.argv[0], ['child_program.js'], {
  detached: true,
  stdio: 'ignore',
});

subprocess.unref();
```

```mjs
import { spawn } from 'node:child_process';
import process from 'node:process';

const subprocess = spawn(process.argv[0], ['child_program.js'], {
  detached: true,
  stdio: 'ignore',
});

subprocess.unref();
```

## `maxBuffer` 和 Unicode

`maxBuffer` 选项指定在 `stdout` 或 `stderr` 上允许的最大字节数。如果超出此值，则子进程将被终止。这会影响包括多字节字符编码（如 UTF-8 或 UTF-16）的输出。例如，`console.log('中文测试')` 将向 `stdout` 发送 13 个 UTF-8 编码的字节，尽管只有 4 个字符。

## Shell 要求

Shell 应理解 `-c` 开关。如果 shell 是 `'cmd.exe'`，它应理解 `/d /s /c` 开关，并且命令行解析应兼容。

## 默认 Windows shell

尽管 Microsoft 指定 `%COMSPEC%` 必须包含根环境中 `'cmd.exe'` 的路径，但子进程并不总是受相同要求的约束。因此，在可以生成 shell 的 `child_process` 函数中，如果 `process.env.ComSpec` 不可用，则使用 `'cmd.exe'` 作为回退。

## 高级序列化

<!-- YAML
added:
 - v13.2.0
 - v12.16.0
-->

子进程支持一种基于 [`node:v8` 模块的序列化 API][v8.serdes] 的 IPC 序列化机制，该机制基于 [HTML 结构化克隆算法][]。这通常更强大，支持更多内置 JavaScript 对象类型，例如 `BigInt`、`Map` 和 `Set`、`ArrayBuffer` 和 `TypedArray`、`Buffer`、`Error`、`RegExp` 等。

但是，这种格式不是 JSON 的完整超集，例如，在此类内置类型对象上设置的属性不会通过序列化步骤传递。此外，性能可能不等同于 JSON，取决于传递数据的结构。因此，此功能需要通过将 `serialization` 选项设置为 `'advanced'` 来调用 [`child_process.spawn()`][] 或 [`child_process.fork()`][] 时选择启用。

[高级序列化]: #advanced-serialization
[默认 Windows shell]: #default-windows-shell
[HTML 结构化克隆算法]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Workers_API/Structured_clone_algorithm
[Shell 要求]: #shell-requirements
[信号事件]: process.md#signal-events
[`'disconnect'`]: process.md#event-disconnect
[`'error'`]: #event-error
[`'exit'`]: #event-exit
[`'message'`]: process.md#event-message
[`ChildProcess`]: #class-childprocess
[`Error`]: errors.md#class-error
[`EventEmitter`]: events.md#class-eventemitter
[`child_process.exec()`]: #child_processexeccommand-options-callback
[`child_process.execFile()`]: #child_processexecfilefile-args-options-callback
[`child_process.execFileSync()`]: #child_processexecfilesyncfile-args-options
[`child_process.execSync()`]: #child_processexecsynccommand-options
[`child_process.fork()`]: #child_processforkmodulepath-args-options
[`child_process.spawn()`]: #child_processspawncommand-args-options
[`child_process.spawnSync()`]: #child_processspawnsynccommand-args-options
[`dgram.Socket`]: dgram.md#class-dgramsocket
[`maxBuffer` 和 Unicode]: #maxbuffer-and-unicode
[`net.Server`]: net.md#class-netserver
[`net.Socket`]: net.md#class-netsocket
[`options.detached`]: #optionsdetached
[`process.disconnect()`]: process.md#processdisconnect
[`process.env`]: process.md#processenv
[`process.execPath`]: process.md#processexecpath
[`process.send()`]: process.md#processsendmessage-sendhandle-options-callback
[`stdio`]: #optionsstdio
[`subprocess.connected`]: #subprocessconnected
[`subprocess.disconnect()`]: #subprocessdisconnect
[`subprocess.kill()`]: #subprocesskillsignal
[`subprocess.send()`]: #subprocesssendmessage-sendhandle-options-callback
[`subprocess.stderr`]: #subprocessstderr
[`subprocess.stdin`]: #subprocessstdin
[`subprocess.stdio`]: #subprocessstdio
[`subprocess.stdout`]: #subprocessstdout
[`util.promisify()`]: util.md#utilpromisifyoriginal
[同步对应方法]: #synchronous-process-creation
[v8.serdes]: v8.md#serialization-api