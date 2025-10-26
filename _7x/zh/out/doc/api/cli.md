# 命令行 API

<!--introduced_in=v5.9.1-->

<!--type=misc-->

Node.js 提供了多种 CLI 选项。这些选项暴露了内置的调试功能、多种执行脚本的方式以及其他有用的运行时选项。

要在终端中作为手册页查看此文档，请运行 `man node`。

## 概要

`node [options] [V8 options] [<program-entry-point> | -e "script" | -] [--] [arguments]`

`node inspect [<program-entry-point> | -e "script" | <host>:<port>] …`

`node --v8-options`

不带参数执行以启动 [REPL][]。

有关 `node inspect` 的更多信息，请参阅 [调试器][] 文档。

## 程序入口点

程序入口点是一个类似于说明符的字符串。如果该字符串不是绝对路径，则会从当前工作目录解析为相对路径。然后该路径由 [CommonJS][] 模块加载器解析。如果未找到相应的文件，则会抛出错误。

如果找到文件，则在以下任何条件下，其路径将传递给 [ES 模块加载器][Modules loaders]：

* 程序启动时使用了强制入口点使用 ECMAScript 模块加载器加载的命令行标志，例如 `--import`。
* 文件具有 `.mjs` 或 `.wasm` 扩展名。
* 文件没有 `.cjs` 扩展名，并且最近的父级 `package.json` 文件包含一个值为 `"module"` 的顶层 [`"type"`][] 字段。

否则，文件将使用 CommonJS 模块加载器加载。有关更多详细信息，请参阅 [模块加载器][Modules loaders]。

### ECMAScript 模块加载器入口点注意事项

加载时，[ES 模块加载器][Modules loaders] 会加载程序入口点，`node` 命令仅接受具有 `.js`、`.mjs` 或 `.cjs` 扩展名的文件作为输入。使用以下标志时，将启用其他文件扩展名：

* [`--experimental-addon-modules`][] 用于具有 `.node` 扩展名的文件。

## 选项

<!-- YAML
changes:
  - version: v10.12.0
    pr-url: https://github.com/nodejs/node/pull/23020
    description: Underscores instead of dashes are now allowed for
                 Node.js options as well, in addition to V8 options.
-->

> Stability: 2 - Stable

所有选项，包括 V8 选项，都允许单词之间用破折号（`-`）或下划线（`_`）分隔。例如，`--pending-deprecation` 等同于 `--pending_deprecation`。

如果传递了接受单个值的选项（例如 `--max-http-header-size`）多次，则使用最后传递的值。来自命令行的选项优先于通过 [`NODE_OPTIONS`][] 环境变量传递的选项。

### `-`

<!-- YAML
added: v8.0.0
-->

标准输入的别名。类似于其他命令行实用程序中 `-` 的用法，表示从标准输入读取脚本，其余选项传递给该脚本。

### `--`

<!-- YAML
added: v6.11.0
-->

表示节点选项的结束。将剩余的参数传递给脚本。如果在此之前没有提供脚本文件名或 eval/print 脚本，则下一个参数将用作脚本文件名。

### `--abort-on-uncaught-exception`

<!-- YAML
added: v0.10.8
-->

中止而不是退出会生成一个核心文件，用于使用调试器（如 `lldb`、`gdb` 和 `mdb`）进行事后分析。

如果传递了此标志，仍然可以通过 [`process.setUncaughtExceptionCaptureCallback()`][]（以及通过使用它的 `node:domain` 模块）将行为设置为不中止。

### `--allow-addons`

<!-- YAML
added:
  - v21.6.0
  - v20.12.0
-->

> Stability: 1.1 - Active development

使用 [权限模型][Permission Model] 时，进程默认将无法使用原生插件。
除非用户在启动 Node.js 时显式传递 `--allow-addons` 标志，否则尝试这样做将抛出 `ERR_DLOPEN_DISABLED`。

示例：

```cjs
// 尝试 require 一个原生插件
require('nodejs-addon-example');
```

```console
$ node --permission --allow-fs-read=* index.js
node:internal/modules/cjs/loader:1319
  return process.dlopen(module, path.toNamespacedPath(filename));
                 ^

Error: Cannot load native addon because loading addons is disabled.
    at Module._extensions..node (node:internal/modules/cjs/loader:1319:18)
    at Module.load (node:internal/modules/cjs/loader:1091:32)
    at Module._load (node:internal/modules/cjs/loader:938:12)
    at Module.require (node:internal/modules/cjs/loader:1115:19)
    at require (node:internal/modules/helpers:130:18)
    at Object.<anonymous> (/home/index.js:1:15)
    at Module._compile (node:internal/modules/cjs/loader:1233:14)
    at Module._extensions..js (node:internal/modules/cjs/loader:1287:10)
    at Module.load (node:internal/modules/cjs/loader:1091:32)
    at Module._load (node:internal/modules/cjs/loader:938:12) {
  code: 'ERR_DLOPEN_DISABLED'
}
```

### `--allow-child-process`

<!-- YAML
added: v20.0.0
changes:
  - version: v24.4.0
    pr-url: https://github.com/nodejs/node/pull/58853
    description: When spawning process with the permission model enabled.
                 The flags are inherit to the child Node.js process through
                 NODE_OPTIONS environment variable.
-->

> Stability: 1.1 - Active development

使用 [权限模型][Permission Model] 时，进程默认将无法生成任何子进程。
除非用户在启动 Node.js 时显式传递 `--allow-child-process` 标志，否则尝试这样做将抛出 `ERR_ACCESS_DENIED`。

示例：

```js
const childProcess = require('node:child_process');
// 尝试绕过权限
childProcess.spawn('node', ['-e', 'require("fs").writeFileSync("/new-file", "example")']);
```

```console
$ node --permission --allow-fs-read=* index.js
node:internal/child_process:388
  const err = this._handle.spawn(options);
                           ^

Error: Access to this API has been restricted
    at ChildProcess.spawn (node:internal/child_process:388:28)
    at node:internal/main/run_main_module:17:47 {
  code: 'ERR_ACCESS_DENIED',
  permission: 'ChildProcess'
}
```

`child_process.fork()` API 从父进程继承执行参数。这意味着如果 Node.js 在启用权限模型的情况下启动并且设置了 `--allow-child-process` 标志，则使用 `child_process.fork()` 创建的任何子进程将自动接收所有相关的权限模型标志。

此行为也适用于 `child_process.spawn()`，但在那种情况下，标志是通过 `NODE_OPTIONS` 环境变量传播的，而不是直接通过进程参数。

### `--allow-fs-read`

<!-- YAML
added: v20.0.0
changes:
  - version: v24.2.0
    pr-url: https://github.com/nodejs/node/pull/58579
    description: Entrypoints of your application are allowed to be read implicitly.
  - version:
    - v23.5.0
    - v22.13.0
    pr-url: https://github.com/nodejs/node/pull/56201
    description: Permission Model and --allow-fs flags are stable.
  - version: v20.7.0
    pr-url: https://github.com/nodejs/node/pull/49047
    description: Paths delimited by comma (`,`) are no longer allowed.
-->

此标志使用 [权限模型][Permission Model] 配置文件系统读取权限。

`--allow-fs-read` 标志的有效参数是：

* `*` - 允许所有 `FileSystemRead` 操作。
* 可以使用多个 `--allow-fs-read` 标志允许多个路径。
  示例 `--allow-fs-read=/folder1/ --allow-fs-read=/folder1/`

示例可以在 [文件系统权限][File System Permissions] 文档中找到。

初始化器模块和自定义的 `--require` 模块具有隐式的读取权限。

```console
$ node --permission -r custom-require.js -r custom-require-2.js index.js
```

* `custom-require.js`、`custom-require-2.js` 和 `index.js` 将默认在允许读取的列表中。

```js
process.permission.has('fs.read', 'index.js'); // true
process.permission.has('fs.read', 'custom-require.js'); // true
process.permission.has('fs.read', 'custom-require-2.js'); // true
```

### `--allow-fs-write`

<!-- YAML
added: v20.0.0
changes:
  - version:
    - v23.5.0
    - v22.13.0
    pr-url: https://github.com/nodejs/node/pull/56201
    description: Permission Model and --allow-fs flags are stable.
  - version: v20.7.0
    pr-url: https://github.com/nodejs/node/pull/49047
    description: Paths delimited by comma (`,`) are no longer allowed.
-->

此标志使用 [权限模型][Permission Model] 配置文件系统写入权限。

`--allow-fs-write` 标志的有效参数是：

* `*` - 允许所有 `FileSystemWrite` 操作。
* 可以使用多个 `--allow-fs-write` 标志允许多个路径。
  示例 `--allow-fs-write=/folder1/ --allow-fs-write=/folder1/`

不再允许使用逗号（`,`）分隔的路径。
当传递带有逗号的单个标志时，将显示警告。

示例可以在 [文件系统权限][File System Permissions] 文档中找到。

### `--allow-wasi`

<!-- YAML
added:
- v22.3.0
- v20.16.0
-->

> Stability: 1.1 - Active development

使用 [权限模型][Permission Model] 时，进程默认将无法创建任何 WASI 实例。
出于安全原因，除非用户在主 Node.js 进程中显式传递 `--allow-wasi` 标志，否则调用将抛出 `ERR_ACCESS_DENIED`。

示例：

```js
const { WASI } = require('node:wasi');
// 尝试绕过权限
new WASI({
  version: 'preview1',
  // 尝试挂载整个文件系统
  preopens: {
    '/': '/',
  },
});
```

```console
$ node --permission --allow-fs-read=* index.js

Error: Access to this API has been restricted
    at node:internal/main/run_main_module:30:49 {
  code: 'ERR_ACCESS_DENIED',
  permission: 'WASI',
}
```

### `--allow-worker`

<!-- YAML
added: v20.0.0
-->

> Stability: 1.1 - Active development

使用 [权限模型][Permission Model] 时，进程默认将无法创建任何工作线程。
出于安全原因，除非用户在主 Node.js 进程中显式传递 `--allow-worker` 标志，否则调用将抛出 `ERR_ACCESS_DENIED`。

示例：

```js
const { Worker } = require('node:worker_threads');
// 尝试绕过权限
new Worker(__filename);
```

```console
$ node --permission --allow-fs-read=* index.js

Error: Access to this API has been restricted
    at node:internal/main/run_main_module:17:47 {
  code: 'ERR_ACCESS_DENIED',
  permission: 'WorkerThreads'
}
```

### `--build-snapshot`

<!-- YAML
added: v18.8.0
-->

> Stability: 1 - Experimental

在进程退出时生成快照 blob 并将其写入磁盘，稍后可以使用 `--snapshot-blob` 加载。

构建快照时，如果未指定 `--snapshot-blob`，则默认情况下生成的 blob 将写入当前工作目录中的 `snapshot.blob`。否则，它将写入 `--snapshot-blob` 指定的路径。

```console
$ echo "globalThis.foo = 'I am from the snapshot'" > snapshot.js

# 运行 snapshot.js 来初始化应用程序并将其状态快照到 snapshot.blob。
$ node --snapshot-blob snapshot.blob --build-snapshot snapshot.js

$ echo "console.log(globalThis.foo)" > index.js

# 加载生成的快照并从 index.js 启动应用程序。
$ node --snapshot-blob snapshot.blob index.js
I am from the snapshot
```

[`v8.startupSnapshot` API][] 可用于在快照构建时指定入口点，从而避免在反序列化时需要额外的入口脚本：

```console
$ echo "require('v8').startupSnapshot.setDeserializeMainFunction(() => console.log('I am from the snapshot'))" > snapshot.js
$ node --snapshot-blob snapshot.blob --build-snapshot snapshot.js
$ node --snapshot-blob snapshot.blob
I am from the snapshot
```

有关更多信息，请查看 [`v8.startupSnapshot` API][] 文档。

目前对运行时快照的支持是实验性的，因为：

1. 快照中尚不支持用户态模块，因此只能快照一个单个文件。然而，用户可以在构建快照之前使用他们选择的打包工具将他们的应用程序打包成单个脚本。
2. 只有一部分内置模块在快照中工作，尽管 Node.js 核心测试套件检查了一些相当复杂的应用程序可以被快照。正在添加对更多模块的支持。如果在构建快照时发生任何崩溃或错误行为，请在 [Node.js 问题跟踪器][Node.js issue tracker] 中提交报告，并在 [用户态快照的跟踪问题][tracking issue for user-land snapshots] 中链接到它。

### `--build-snapshot-config`

<!-- YAML
added:
  - v21.6.0
  - v20.12.0
-->

> Stability: 1 - Experimental

指定一个 JSON 配置文件的路径，该文件配置快照创建行为。

目前支持以下选项：

* `builder` {string} 必需。提供在构建快照之前执行的脚本的名称，就像已使用 `builder` 作为主脚本名称传递了 [`--build-snapshot`][] 一样。
* `withoutCodeCache` {boolean} 可选。包含代码缓存会减少在快照中包含的函数在编译上花费的时间，但代价是更大的快照大小并可能破坏快照的可移植性。

使用此标志时，在命令行上提供的其他脚本文件将不会被执行，而是被解释为常规命令行参数。

### `-c`, `--check`

<!-- YAML
added:
  - v5.0.0
  - v4.2.0
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19600
    description: The `--require` option is now supported when checking a file.
-->

语法检查脚本而不执行。

### `--completion-bash`

<!-- YAML
added: v10.12.0
-->

打印可源的 bash 补全脚本。

```bash
node --completion-bash > node_bash_completion
source node_bash_completion
```

### `-C condition`, `--conditions=condition`

<!-- YAML
added:
  - v14.9.0
  - v12.19.0
changes:
  - version:
    - v22.9.0
    - v20.18.0
    pr-url: https://github.com/nodejs/node/pull/54209
    description: The flag is no longer experimental.
-->

提供自定义的 [条件导出][conditional exports] 解析条件。

允许任意数量的自定义字符串条件名称。

Node.js 的默认条件 `"node"`、`"default"`、`"import"` 和 `"require"` 将始终按定义应用。

例如，使用 "development" 解析运行模块：

```bash
node -C development app.js
```

### `--cpu-prof`

<!-- YAML
added: v12.0.0
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53343
    description: The `--cpu-prof` flags are now stable.
-->

在启动时启动 V8 CPU 分析器，并在退出前将 CPU 分析文件写入磁盘。

如果未指定 `--cpu-prof-dir`，则生成的分析文件将放在当前工作目录中。

如果未指定 `--cpu-prof-name`，则生成的分析文件命名为 `CPU.${yyyymmdd}.${hhmmss}.${pid}.${tid}.${seq}.cpuprofile`。

```console
$ node --cpu-prof index.js
$ ls *.cpuprofile
CPU.20190409.202950.15293.0.0.cpuprofile
```

如果指定了 `--cpu-prof-name`，则提供的值将用作文件名的模板。支持以下占位符，并在运行时被替换：

* `${pid}` — 当前进程 ID

```console
$ node --cpu-prof --cpu-prof-name 'CPU.${pid}.cpuprofile' index.js
$ ls *.cpuprofile
CPU.15293.cpuprofile
```

### `--cpu-prof-dir`

<!-- YAML
added: v12.0.0
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53343
    description: The `--cpu-prof` flags are now stable.
-->

指定由 `--cpu-prof` 生成的 CPU 分析文件将被放置的目录。

默认值由 [`--diagnostic-dir`][] 命令行选项控制。

### `--cpu-prof-interval`

<!-- YAML
added: v12.2.0
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53343
    description: The `--cpu-prof` flags are now stable.
-->

指定由 `--cpu-prof` 生成的 CPU 分析文件的采样间隔（以微秒为单位）。默认为 1000 微秒。

### `--cpu-prof-name`

<!-- YAML
added: v12.0.0
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53343
    description: The `--cpu-prof` flags are now stable.
-->

指定由 `--cpu-prof` 生成的 CPU 分析文件的文件名。

### `--diagnostic-dir=directory`

设置所有诊断输出文件写入的目录。默认为当前工作目录。

影响以下内容的默认输出目录：

* [`--cpu-prof-dir`][]
* [`--heap-prof-dir`][]
* [`--redirect-warnings`][]

### `--disable-proto=mode`

<!-- YAML
added:
 - v13.12.0
 - v12.17.0
-->

禁用 `Object.prototype.__proto__` 属性。如果 `mode` 是 `delete`，则该属性将被完全移除。如果 `mode` 是 `throw`，则访问该属性将抛出带有代码 `ERR_PROTO_ACCESS` 的异常。

### `--disable-sigusr1`

<!-- YAML
added:
  - v23.7.0
  - v22.14.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59707
    description: The option is no longer experimental.
-->

禁用通过向进程发送 `SIGUSR1` 信号来启动调试会话的能力。

### `--disable-warning=code-or-type`

<!-- YAML
added:
  - v21.3.0
  - v20.11.0
-->

> Stability: 1.1 - Active development

通过 `code` 或 `type` 禁用特定的进程警告。

从 [`process.emitWarning()`][emit_warning] 发出的警告可能包含 `code` 和 `type`。此选项将不发出具有匹配 `code` 或 `type` 的警告。

[弃用警告列表][deprecation warnings]。

Node.js 核心警告类型有：`DeprecationWarning` 和 `ExperimentalWarning`

例如，当使用 `node --disable-warning=DEP0025` 执行时，以下脚本将不会发出 [DEP0025 `require('node:sys')`][DEP0025 warning]：

```mjs
import sys from 'node:sys';
```

```cjs
const sys = require('node:sys');
```

例如，当使用 `node --disable-warning=ExperimentalWarning` 执行时，以下脚本将发出 [DEP0025 `require('node:sys')`][DEP0025 warning]，但不会发出任何实验性警告（例如在 <=v21 中的 [ExperimentalWarning: `vm.measureMemory` is an experimental feature][]）：

```mjs
import sys from 'node:sys';
import vm from 'node:vm';

vm.measureMemory();
```

```cjs
const sys = require('node:sys');
const vm = require('node:vm');

vm.measureMemory();
```

### `--disable-wasm-trap-handler`

<!-- YAML
added:
- v22.2.0
- v20.15.0
-->

默认情况下，Node.js 启用基于陷阱处理程序的 WebAssembly 边界检查。因此，V8 不需要在从 WebAssembly 编译的代码中插入内联边界检查，这可能会显著加速 WebAssembly 执行，但这种优化需要分配一个大的虚拟内存笼（目前是 10GB）。如果由于系统配置或硬件限制，Node.js 进程无法访问足够大的虚拟内存地址空间，用户将无法运行任何涉及在此虚拟内存笼中分配的 WebAssembly，并将看到内存不足错误。

```console
$ ulimit -v 5000000
$ node -p "new WebAssembly.Memory({ initial: 10, maximum: 100 });"
[eval]:1
new WebAssembly.Memory({ initial: 10, maximum: 100 });
^

RangeError: WebAssembly.Memory(): could not allocate memory
    at [eval]:1:1
    at runScriptInThisContext (node:internal/vm:209:10)
    at node:internal/process/execution:118:14
    at [eval]-wrapper:6:24
    at runScript (node:internal/process/execution:101:62)
    at evalScript (node:internal/process/execution:136:3)
    at node:internal/main/eval_string:49:3

```

`--disable-wasm-trap-handler` 禁用此优化，以便当 Node.js 进程可用的虚拟内存地址空间低于 V8 WebAssembly 内存笼所需时，用户至少可以运行 WebAssembly（性能较差）。

### `--disallow-code-generation-from-strings`

<!-- YAML
added: v9.8.0
-->

使从字符串生成代码的内置语言特性（如 `eval` 和 `new Function`）抛出异常。这不影响 Node.js `node:vm` 模块。

### `--dns-result-order=order`

<!-- YAML
added:
  - v16.4.0
  - v14.18.0
changes:
  - version:
    - v22.1.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52492
    description: The `ipv6first` is supported now.
  - version: v17.0.0
    pr-url: https://github.com/nodejs/node/pull/39987
    description: Changed default value to `verbatim`.
-->

设置 [`dns.lookup()`][] 和 [`dnsPromises.lookup()`][] 中 `order` 的默认值。值可以是：

* `ipv4first`：将默认 `order` 设置为 `ipv4first`。
* `ipv6first`：将默认 `order` 设置为 `ipv6first`。
* `verbatim`：将默认 `order` 设置为 `verbatim`。

默认为 `verbatim`，并且 [`dns.setDefaultResultOrder()`][] 的优先级高于 `--dns-result-order`。

### `--enable-fips`

<!-- YAML
added: v6.0.0
-->

在启动时启用符合 FIPS 的加密。（要求 Node.js 针对 FIPS 兼容的 OpenSSL 构建。）

### `--enable-network-family-autoselection`

<!-- YAML
added: v18.18.0
-->

启用族自动选择算法，除非连接选项显式禁用它。

### `--enable-source-maps`

<!-- YAML
added: v12.12.0
changes:
  - version:
      - v15.11.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37362
    description: This API is no longer experimental.
-->

为堆栈跟踪启用 [Source Map][] 支持。

当使用转译器（如 TypeScript）时，应用程序抛出的堆栈跟踪引用的是转译后的代码，而不是原始源代码位置。`--enable-source-maps` 启用 Source Maps 的缓存，并尽力报告相对于原始源文件的堆栈跟踪。

覆盖 `Error.prepareStackTrace` 可能会阻止 `--enable-source-maps` 修改堆栈跟踪。在覆盖函数中调用并返回原始 `Error.prepareStackTrace` 的结果，以使用 source maps 修改堆栈跟踪。

```js
const originalPrepareStackTrace = Error.prepareStackTrace;
Error.prepareStackTrace = (error, trace) => {
  // 修改 error 和 trace 并使用原始的 Error.prepareStackTrace 格式化堆栈跟踪。
  return originalPrepareStackTrace(error, trace);
};
```

注意，启用 source maps 可能会在访问 `Error.stack` 时给应用程序带来延迟。如果在应用程序中频繁访问 `Error.stack`，请考虑 `--enable-source-maps` 的性能影响。

### `--entry-url`

<!-- YAML
added:
  - v23.0.0
  - v22.10.0
-->

> Stability: 1 - Experimental

当存在时，Node.js 会将入口点解释为 URL，而不是路径。

遵循 [ECMAScript 模块][ECMAScript module] 解析规则。

URL 中的任何查询参数或哈希将通过 [`import.meta.url`][] 访问。

```bash
node --entry-url 'file:///path/to/file.js?queryparams=work#and-hashes-too'
node --entry-url 'file.ts?query#hash'
node --entry-url 'data:text/javascript,console.log("Hello")'
```

### `--env-file-if-exists=file`

<!-- YAML
added: v22.9.0
changes:
  - version: v24.10.0
    pr-url: https://github.com/nodejs/node/pull/59925
    description: The `--env-file-if-exists` flag is no longer experimental.
-->

行为与 [`--env-file`][] 相同，但如果文件不存在则不会抛出错误。

### `--env-file=file`

<!-- YAML
added: v20.6.0
changes:
  - version: v24.10.0
    pr-url: https://github.com/nodejs/node/pull/59925
    description: The `--env-file` flag is no longer experimental.
  - version:
    - v21.7.0
    - v20.12.0
    pr-url: https://github.com/nodejs/node/pull/51289
    description: Add support to multi-line values.
-->

从相对于当前目录的文件加载环境变量，使其在 `process.env` 上对应用程序可用。配置 Node.js 的 [环境变量][environment_variables]，例如 `NODE_OPTIONS`，会被解析并应用。如果同一个变量在环境和文件中都有定义，则环境中的值优先。

您可以传递多个 `--env-file` 参数。后续文件会覆盖先前文件中定义的变量。

如果文件不存在，则会抛出错误。

```bash
node --env-file=.env --env-file=.development.env index.js
```

文件的格式应为每行一个环境变量名称和值的键值对，用 `=` 分隔：

```text
PORT=3000
```

`#` 之后的任何文本都被视为注释：

```text
# 这是一个注释
PORT=3000 # 这也是一个注释
```

值可以用以下引号开头和结尾：`` ` ``、`"` 或 `'`。它们会从值中省略。

```text
USERNAME="nodejs" # 将得到 `nodejs` 作为值。
```

支持多行值：

```text
MULTI_LINE="THIS IS
A MULTILINE"
# 将得到 `THIS IS\nA MULTILINE` 作为值。
```

键前的 export 关键字被忽略：

```text
export USERNAME="nodejs" # 将得到 `nodejs` 作为值。
```

如果您想从可能不存在的文件加载环境变量，可以使用 [`--env-file-if-exists`][] 标志代替。

### `-e`, `--eval "script"`

<!-- YAML
added: v0.5.2
changes:
  - version: v22.6.0
    pr-url: https://github.com/nodejs/node/pull/53725
    description: Eval now supports experimental type-stripping.
  - version: v5.11.0
    pr-url: https://github.com/nodejs/node/pull/5348
    description: Built-in libraries are now available as predefined variables.
-->

将后续参数作为 JavaScript 求值。REPL 中预定义的模块也可以在 `script` 中使用。

在 Windows 上，使用 `cmd.exe` 时单引号无法正常工作，因为它只识别双引号 `"` 用于引用。在 Powershell 或 Git bash 中，`'` 和 `"` 都可使用。

可以运行包含内联类型的代码，除非提供了 [`--no-experimental-strip-types`][] 标志。

### `--experimental-addon-modules`

<!-- YAML
added: v23.6.0
-->

> Stability: 1.0 - Early development

启用对 `.node` 插件的实验性导入支持。

### `--experimental-config-file=config`

<!-- YAML
added: v23.10.0
-->

> Stability: 1.0 - Early development

如果存在，Node.js 将在指定路径查找配置文件。Node.js 将读取配置文件并应用设置。配置文件应是一个具有以下结构的 JSON 文件。`$schema` 中的 `vX.Y.Z` 必须替换为您使用的 Node.js 版本。

```json
{
  "$schema": "https://nodejs.org/dist/vX.Y.Z/docs/node-config-schema.json",
  "nodeOptions": {
    "import": [
      "amaro/strip"
    ],
    "watch-path": "src",
    "watch-preserve-output": true
  },
  "testRunner": {
    "test-isolation": "process"
  }
}
```

配置文件支持特定命名空间的选项：

* `nodeOptions` 字段包含在 [`NODE_OPTIONS`][] 中允许的 CLI 标志。

* 像 `testRunner` 这样的命名空间字段包含特定于该子系统的配置。

不支持无操作标志。
目前并非所有 V8 标志都受支持。

可以使用 [官方 JSON 模式](../node-config-schema.json) 来验证配置文件，这可能因 Node.js 版本而异。配置文件中的每个键对应于可以作为命令行参数传递的标志。键的值是传递给标志的值。

例如，上面的配置文件等效于以下命令行参数：

```bash
node --import amaro/strip --watch-path=src --watch-preserve-output --test-isolation=process
```

配置的优先级如下：

1. NODE_OPTIONS 和命令行选项
2. 配置文件
3. Dotenv NODE_OPTIONS

配置文件中的值不会覆盖环境变量和命令行选项中的值，但会覆盖由 `--env-file` 标志解析的 `NODE_OPTIONS` 环境文件中的值。

在同一或不同命名空间中不能重复键。

如果配置文件包含未知键或不能在命名空间中使用的键，配置解析器将抛出错误。

Node.js 不会对用户提供的配置进行清理或验证，因此 **绝不** 使用不受信任的配置文件。

### `--experimental-default-config-file`

<!-- YAML
added: v23.10.0
-->

> Stability: 1.0 - Early development

如果存在 `--experimental-default-config-file` 标志，Node.js 将在当前工作目录中查找 `node.config.json` 文件并将其作为配置文件加载。

### `--experimental-eventsource`

<!-- YAML
added:
  - v22.3.0
  - v20.18.0
-->

在全局作用域上启用 [EventSource Web API][] 的暴露。

### `--experimental-import-meta-resolve`

<!-- YAML
added:
  - v13.9.0
  - v12.16.2
changes:
  - version:
    - v20.6.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/49028
    description: synchronous import.meta.resolve made available by default, with
                 the flag retained for enabling the experimental second argument
                 as previously supported.
-->

启用实验性的 `import.meta.resolve()` 父 URL 支持，允许传递第二个 `parentURL` 参数用于上下文解析。

以前控制整个 `import.meta.resolve` 功能。

### `--experimental-inspector-network-resource`

<!-- YAML
added:
  - v24.5.0
-->

> Stability: 1.1 - Active Development

启用对检查器网络资源的实验性支持。

### `--experimental-loader=module`

<!-- YAML
added: v8.8.0
changes:
  - version:
    - v23.6.1
    - v22.13.1
    - v20.18.2
    pr-url: https://github.com/nodejs-private/node-private/pull/629
    description: Using this feature with the permission model enabled requires
                 passing `--allow-worker`.
  - version: v12.11.1
    pr-url: https://github.com/nodejs/node/pull/29752
    description: This flag was renamed from `--loader` to
                 `--experimental-loader`.
-->

> 不鼓励使用此标志，并可能在未来的 Node.js 版本中移除。请改用
> [带有 `register()` 的 `--import`][module customization hooks: enabling]。

指定包含导出的 [模块自定义钩子][module customization hooks] 的 `module`。`module` 可以是任何被接受为 [`import` 说明符][`import` specifier] 的字符串。

如果与 [权限模型][Permission Model] 一起使用，此功能需要传递 `--allow-worker`。

### `--experimental-network-inspection`

<!-- YAML
added:
  - v22.6.0
  - v20.18.0
-->

> Stability: 1 - Experimental

启用与 Chrome DevTools 进行网络检查的实验性支持。

### `--experimental-print-required-tla`

<!-- YAML
added:
  - v22.0.0
  - v20.17.0
-->

如果被 `require()` 的 ES 模块包含顶层 `await`，此标志允许 Node.js 评估该模块，尝试定位顶层 await，并打印它们的位置以帮助用户找到它们。

### `--experimental-require-module`

<!-- YAML
added:
  - v22.0.0
  - v20.17.0
changes:
  - version:
    - v23.0.0
    - v22.12.0
    - v20.19.0
    pr-url: https://github.com/nodejs/node/pull/55085
    description: This is now true by default.
-->

> Stability: 1.1 - Active Development

支持在 `require()` 中加载同步的 ES 模块图。

参见 [使用 `require()` 加载 ECMAScript 模块][Loading ECMAScript modules using `require()`]。

### `--experimental-sea-config`

<!-- YAML
added: v20.0.0
-->

> Stability: 1 - Experimental

使用此标志生成可以注入到 Node.js 二进制文件中的 blob，以生成 [单一可执行应用程序][]。有关详细信息，请参阅关于 [此配置][`--experimental-sea-config`] 的文档。

### `--experimental-shadow-realm`

<!-- YAML
added:
  - v19.0.0
  - v18.13.0
-->

使用此标志启用 [ShadowRealm][] 支持。

### `--experimental-test-coverage`

<!-- YAML
added:
  - v19.7.0
  - v18.15.0
changes:
  - version:
    - v20.1.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/47686
    description: This option can be used with `--test`.
-->

与 `node:test` 模块一起使用时，会生成代码覆盖率报告作为测试运行器输出的一部分。如果没有运行测试，则不会生成覆盖率报告。有关更多详细信息，请参阅关于 [从测试收集代码覆盖率][collecting code coverage from tests] 的文档。

### `--experimental-test-module-mocks`

<!-- YAML
added:
  - v22.3.0
  - v20.18.0
changes:
  - version:
    - v23.6.1
    - v22.13.1
    - v20.18.2
    pr-url: https://github.com/nodejs-private/node-private/pull/629
    description: Using this feature with the permission model enabled requires
                 passing `--allow-worker`.
-->

> Stability: 1.0 - Early development

在测试运行器中启用模块模拟。

如果与 [权限模型][Permission Model] 一起使用，此功能需要传递 `--allow-worker`。

### `--experimental-transform-types`

<!-- YAML
added: v22.7.0
-->

> Stability: 1.2 - Release candidate

启用将仅限 TypeScript 的语法转换为 JavaScript 代码。隐含 `--enable-source-maps`。

### `--experimental-vm-modules`

<!-- YAML
added: v9.6.0
-->

在 `node:vm` 模块中启用实验性的 ES 模块支持。

### `--experimental-wasi-unstable-preview1`

<!-- YAML
added:
  - v13.3.0
  - v12.16.0
changes:
  - version:
    - v20.0.0
    - v18.17.0
    pr-url: https://github.com/nodejs/node/pull/47286
    description: This option is no longer required as WASI is
                 enabled by default, but can still be passed.
  - version: v13.6.0
    pr-url: https://github.com/nodejs/node/pull/30980
    description: changed from `--experimental-wasi-unstable-preview0` to
                 `--experimental-wasi-unstable-preview1`.
-->

启用实验性的 WebAssembly 系统接口 (WASI) 支持。

### `--experimental-webstorage`

<!-- YAML
added: v22.4.0
-->

启用实验性的 [`Web Storage`][] 支持。

### `--experimental-worker-inspection`

<!-- YAML
added:
  - v24.1.0
-->

> Stability: 1.1 - Active Development

启用与 Chrome DevTools 进行工作线程检查的实验性支持。

### `--expose-gc`

<!-- YAML
added:
  - v22.3.0
  - v20.18.0
-->

> Stability: 1 - Experimental. This flag is inherited from V8 and is subject to
> change upstream.

此标志将暴露 V8 的 gc 扩展。

```js
if (globalThis.gc) {
  globalThis.gc();
}
```

### `--force-context-aware`

<!-- YAML
added: v12.12.0
-->

禁止加载非 [上下文感知][context-aware] 的原生插件。

### `--force-fips`

<!-- YAML
added: v6.0.0
-->

在启动时强制启用符合 FIPS 的加密。（无法从脚本代码中禁用。）（与 `--enable-fips` 的要求相同。）

### `--force-node-api-uncaught-exceptions-policy`

<!-- YAML
added:
  - v18.3.0
  - v16.17.0
-->

在 Node-API 异步回调上强制执行 `uncaughtException` 事件。

为了防止现有的插件崩溃进程，此标志默认未启用。将来，此标志将默认启用以强制执行正确的行为。

### `--frozen-intrinsics`

<!-- YAML
added: v11.12.0
-->

> Stability: 1 - Experimental

启用实验性的冻结内置对象，如 `Array` 和 `Object`。

仅支持根上下文。不保证 `globalThis.Array` 确实是默认的内置对象引用。在此标志下代码可能会中断。

为了允许添加 polyfill，[`--require`][] 和 [`--import`][] 都在冻结内置对象之前运行。

### `--heap-prof`

<!-- YAML
added: v12.4.0
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53343
    description: The `--heap-prof` flags are now stable.
-->

在启动时启动 V8 堆分析器，并在退出前将堆分析文件写入磁盘。

如果未指定 `--heap-prof-dir`，则生成的分析文件将放在当前工作目录中。

如果未指定 `--heap-prof-name`，则生成的分析文件命名为 `Heap.${yyyymmdd}.${hhmmss}.${pid}.${tid}.${seq}.heapprofile`。

```console
$ node --heap-prof index.js
$ ls *.heapprofile
Heap.20190409.202950.15293.0.001.heapprofile
```

### `--heap-prof-dir`

<!-- YAML
added: v12.4.0
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53343
    description: The `--heap-prof` flags are now stable.
-->

指定由 `--heap-prof` 生成的堆分析文件将被放置的目录。

默认值由 [`--diagnostic-dir`][] 命令行选项控制。

### `--heap-prof-interval`

<!-- YAML
added: v12.4.0
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53343
    description: The `--heap-prof` flags are now stable.
-->

指定由 `--heap-prof` 生成的堆分析文件的平均采样间隔（以字节为单位）。默认为 512 \* 1024 字节。

### `--heap-prof-name`

<!-- YAML
added: v12.4.0
changes:
  - version:
    - v22.4.0
    - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/53343
    description: The `--heap-prof` flags are now stable.
-->

指定由 `--heap-prof` 生成的堆分析文件的文件名。

### `--heapsnapshot-near-heap-limit=max_count`

<!-- YAML
added:
  - v15.1.0
  - v14.18.0
-->

> Stability: 1 - Experimental

当 V8 堆使用量接近堆限制时，将 V8 堆快照写入磁盘。`count` 应该是一个非负整数（在这种情况下，Node.js 将向磁盘写入不超过 `max_count` 个快照）。

生成快照时，可能会触发垃圾回收并降低堆使用量。因此，在 Node.js 实例最终耗尽内存之前，可能会将多个快照写入磁盘。这些堆快照可以进行比较，以确定在连续拍摄快照的时间段内分配了哪些对象。不保证 Node.js 会向磁盘写入恰好 `max_count` 个快照，但当 `max_count` 大于 `0` 时，它会尽力在 Node.js 实例耗尽内存之前生成至少一个且最多 `max_count` 个快照。

生成 V8 快照需要时间和内存（V8 堆管理的内存和 V8 堆外部的原生内存）。堆越大，需要的资源就越多。Node.js 将调整 V8 堆以适应额外的 V8 堆内存开销，并尽力避免使用进程可用的所有内存。当进程使用的内存超过系统认为合适的值时，根据系统配置，进程可能会被系统突然终止。

```console
$ node --max-old-space-size=100 --heapsnapshot-near-heap-limit=3 index.js
Wrote snapshot to Heap.20200430.100036.49580.0.001.heapsnapshot
Wrote snapshot to Heap.20200430.100037.49580.0.002.heapsnapshot
Wrote snapshot to Heap.20200430.100038.49580.0.003.heapsnapshot

<--- Last few GCs --->

[49580:0x110000000]     4826 ms: Mark-sweep 130.6 (147.8) -> 130.5 (147.8) MB, 27.4 / 0.0 ms  (average mu = 0.126, current mu = 0.034) allocation failure scavenge might not succeed
[49580:0x110000000]     4845 ms: Mark-sweep 130.6 (147.8) -> 130.6 (147.8) MB, 18.8 / 0.0 ms  (average mu = 0.088, current mu = 0.031) allocation failure scavenge might not succeed


<--- JS stacktrace --->

FATAL ERROR: Ineffective mark-compacts near heap limit Allocation failed - JavaScript heap out of memory
....
```

### `--heapsnapshot-signal=signal`

<!-- YAML
added: v12.0.0
-->

启用一个信号处理程序，当收到指定信号时，导致 Node.js 进程写入堆转储。`signal` 必须是一个有效的信号名称。默认禁用。

```console
$ node --heapsnapshot-signal=SIGUSR2 index.js &
$ ps aux
USER       PID %CPU %MEM    VSZ   RSS TTY      STAT START   TIME COMMAND
node         1  5.5  6.1 787252 247004 ?       Ssl  16:43   0:02 node --heapsnapshot-signal=SIGUSR2 index.js
$ kill -USR2 1
$ ls
Heap.20190718.133405.15554.0.001.heapsnapshot
```

### `-h`, `--help`

<!-- YAML
added: v0.1.3
-->

打印节点命令行选项。此选项的输出比本文档更简略。

### `--icu-data-dir=file`

<!-- YAML
added: v0.11.15
-->

指定 ICU 数据加载路径。（覆盖 `NODE_ICU_DATA`。）

### `--import=module`

<!-- YAML
added:
 - v19.0.0
 - v18.18.0
-->

> Stability: 1 - Experimental

在启动时预加载指定的模块。如果多次提供该标志，每个模块将按它们出现的顺序依次执行，从 [`NODE_OPTIONS`][] 中提供的模块开始。

遵循 [ECMAScript 模块][ECMAScript module] 解析规则。
使用 [`--require`][] 加载 [CommonJS 模块][CommonJS module]。
使用 `--require` 预加载的模块将在使用 `--import` 预加载的模块之前运行。

模块被预加载到主线程以及任何工作线程、分叉进程或集群进程中。

### `--input-type=type`

<!-- YAML
added: v12.0.0
changes:
  - version: v23.6.0
    pr-url: https://github.com/nodejs/node/pull/56350
    description: Add support for `-typescript` values.
  - version:
    - v22.7.0
    - v20.19.0
    pr-url: https://github.com/nodejs/node/pull/53619
    description: ESM syntax detection is enabled by default.
-->

这将配置 Node.js 将 `--eval` 或 `STDIN` 输入解释为 CommonJS 或作为 ES 模块。有效值为 `"commonjs"`、`"module"`、`"module-typescript"` 和 `"commonjs-typescript"`。`"-typescript"` 值在使用 `--no-experimental-strip-types` 标志时不可用。默认无值，或者如果传递了 `--no-experimental-detect-module` 则为 `"commonjs"`。

如果未提供 `--input-type`，Node.js 将尝试通过以下步骤检测语法：

1. 将输入作为 CommonJS 运行。
2. 如果步骤 1 失败，将输入作为 ES 模块运行。
3. 如果步骤 2 因 SyntaxError 失败，则剥离类型。
4. 如果步骤 3 因错误代码 [`ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`][] 或 [`ERR_INVALID_TYPESCRIPT_SYNTAX`][] 失败，则抛出步骤 2 的错误，包括 TypeScript 错误在消息中，否则作为 CommonJS 运行。
5. 如果步骤 4 失败，将输入作为 ES 模块运行。

为了避免多次语法检测的延迟，可以使用 `--input-type=type` 标志来指定应如何解释 `--eval` 输入。

REPL 不支持此选项。将 `--input-type=module` 与 [`--print`][] 一起使用将抛出错误，因为 `--print` 不支持 ES 模块语法。

### `--insecure-http-parser`

<!-- YAML
added:
 - v13.4.0
 - v12.15.0
 - v10.19.0
-->

在 HTTP 解析器上启用宽松标志。这可能允许与不符合规范的 HTTP 实现进行互操作。

启用后，解析器将接受以下内容：

* 无效的 HTTP 标头值。
* 无效的 HTTP 版本。
* 允许消息同时包含 `Transfer-Encoding` 和 `Content-Length` 标头。
* 当存在 `Connection: close` 时，允许消息后有额外数据。
* 在提供 `chunked` 后允许额外的传输编码。
* 允许使用 `\n` 作为令牌分隔符而不是 `\r\n`。
* 允许在块后不提供 `\r\n`。
* 允许在块大小和 `\r\n` 之间存在空格。

以上所有内容都会使您的应用程序面临请求走私或投毒攻击。避免使用此选项。

<!-- Anchor to make sure old links find a target -->

<a id="inspector_security"></a>

#### 警告：将检查器绑定到公共 IP:端口组合是不安全的

将检查器绑定到具有开放端口的公共 IP（包括 `0.0.0.0`）是不安全的，因为它允许外部主机连接到检查器并执行 [远程代码执行][remote code execution] 攻击。

如果指定了主机，请确保：

* 主机无法从公共网络访问。
* 防火墙不允许在端口上进行不需要的连接。

**更具体地说，如果端口（默认为 `9229`）没有防火墙保护，`--inspect=0.0.0.0` 是不安全的。**

有关更多信息，请参阅 [调试安全影响][debugging security implications] 部分。

### `--inspect-brk[=[host:]port]`

<!-- YAML
added: v7.6.0
-->

在 `host:port` 上激活检查器并在用户脚本开始时中断。默认 `host:port` 是 `127.0.0.1:9229`。如果指定了端口 `0`，将使用随机可用端口。

有关 Node.js 调试器的进一步说明，请参阅 [Node.js 的 V8 检查器集成][V8 Inspector integration for Node.js]。

### `--inspect-port=[host:]port`

<!-- YAML
added: v7.6.0
-->

设置检查器激活时使用的 `host:port`。在通过发送 `SIGUSR1` 信号激活检查器时很有用。除非传递了 [`--disable-sigusr1`][]。

默认主机是 `127.0.0.1`。如果指定了端口 `0`，将使用随机可用端口。

请参阅下面关于 `host` 参数使用的 [安全警告][security warning]。

### `--inspect-publish-uid=stderr,http`

指定检查器 Web Socket URL 的暴露方式。

默认情况下，检查器 websocket URL 在 stderr 中可用，并在 `http://host:port/json/list` 端点下可用。

### `--inspect-wait[=[host:]port]`

<!-- YAML
added:
  - v22.2.0
  - v20.15.0
-->

在 `host:port` 上激活检查器并等待调试器附加。默认 `host:port` 是 `127.0.0.1:9229`。如果指定了端口 `0`，将使用随机可用端口。

有关 Node.js 调试器的进一步说明，请参阅 [Node.js 的 V8 检查器集成][V8 Inspector integration for Node.js]。

### `--inspect[=[host:]port]`

<!-- YAML
added: v6.3.0
-->

在 `host:port` 上激活检查器。默认为 `127.0.0.1:9229`。如果指定了端口 `0`，将使用随机可用端口。

V8 检查器集成允许工具（如 Chrome DevTools 和 IDE）调试和分析 Node.js 实例。这些工具通过 tcp 端口附加到 Node.js 实例，并使用 [Chrome DevTools 协议][Chrome DevTools Protocol] 进行通信。有关 Node.js 调试器的进一步说明，请参阅 [Node.js 的 V8 检查器集成][V8 Inspector integration for Node.js]。

### `-i`, `--interactive`

<!-- YAML
added: v0.7.7
-->

即使标准输入似乎不是终端，也会打开 REPL。

### `--jitless`

<!-- YAML
added: v12.0.0
-->

> Stability: 1 - Experimental. This flag is inherited from V8 and is subject to
> change upstream.

禁用 [运行时分配可执行内存][jitless]。出于安全原因，在某些平台上可能需要这样做。它也可以减少其他平台上的攻击面，但性能影响可能很严重。

### `--localstorage-file=file`

<!-- YAML
added: v22.4.0
-->

用于存储 `localStorage` 数据的文件。如果文件不存在，则在首次访问 `localStorage` 时创建。同一文件可能由多个 Node.js 进程并发共享。除非 Node.js 使用 `--experimental-webstorage` 标志启动，否则此标志无效。

### `--max-http-header-size=size`

<!-- YAML
added:
 - v11.6.0
 - v10.15.0
changes:
  - version: v13.13.0
    pr-url: https://github.com/nodejs/node/pull/32520
    description: Change maximum default size of HTTP headers from 8 KiB to 16 KiB.
-->

指定 HTTP 标头的最大大小（以字节为单位）。默认为 16 KiB。

### `--max-old-space-size-percentage=PERCENTAGE`

将 V8 旧内存部分的最大内存大小设置为可用系统内存的百分比。当同时指定此标志和 `--max-old-space-size` 时，此标志优先。

`PERCENTAGE` 参数必须是一个大于 0 且最多为 100 的数字，表示分配给 V8 堆的可用系统内存的百分比。

```bash
# 使用 50% 的可用系统内存
node --max-old-space-size-percentage=50 index.js

# 使用 75% 的可用系统内存
node --max-old-space-size-percentage=75 index.js
```

### `--napi-modules`

<!-- YAML
added: v7.10.0
-->

此选项无效。为了兼容性而保留。

### `--network-family-autoselection-attempt-timeout`

<!-- YAML
added:
  - v22.1.0
  - v20.13.0
-->

设置网络族自动选择尝试超时的默认值。有关更多信息，请参阅 [`net.getDefaultAutoSelectFamilyAttemptTimeout()`][]。

### `--no-addons`

<!-- YAML
added:
  - v16.10.0
  - v14.19.0
-->

禁用 `node-addons` 导出条件以及禁用加载原生插件。当指定 `--no-addons` 时，调用 `process.dlopen` 或 require 原生 C++ 插件将失败并抛出异常。

### `--no-async-context-frame`

<!-- YAML
added: v24.0.0
-->

禁用由 `AsyncContextFrame` 支持的 [`AsyncLocalStorage`][] 的使用，并使用依赖 async_hooks 的先前实现。先前的模型被保留以与 Electron 兼容，并用于上下文流可能不同的情况。但是，如果发现流有差异，请报告。

### `--no-deprecation`

<!-- YAML
added: v0.8.0
-->

静默弃用警告。

### `--no-experimental-detect-module`

<!-- YAML
added:
  - v21.1.0
  - v20.10.0
changes:
  - version:
    - v22.7.0
    - v20.19.0
    pr-url: https://github.com/nodejs/node/pull/53619
    description: Syntax detection is enabled by default.
-->

禁用使用 [语法检测][syntax detection] 来确定模块类型。

### `--no-experimental-global-navigator`

<!-- YAML
added: v21.2.0
-->

> Stability: 1 - Experimental

禁止在全局作用域上暴露 [Navigator API][]。

### `--no-experimental-repl-await`

<!-- YAML
added: v16.6.0
-->

使用此标志禁用 REPL 中的顶层 await。

### `--no-experimental-require-module`

<!-- YAML
added:
  - v22.0.0
  - v20.17.0
changes:
  - version:
    - v23.0.0
    - v22.12.0
    - v20.19.0
    pr-url: https://github.com/nodejs/node/pull/55085
    description: This is now false by default.
-->

> Stability: 1.1 - Active Development

禁用支持在 `require()` 中加载同步 ES 模块图。

参见 [使用 `require()` 加载 ECMAScript 模块][Loading ECMAScript modules using `require()`]。

### `--no-experimental-sqlite`

<!-- YAML
added: v22.5.0
changes:
  - version:
    - v23.4.0
    - v22.13.0
    pr-url: https://github.com/nodejs/node/pull/55890
    description: SQLite is unflagged but still experimental.
-->

禁用实验性的 [`node:sqlite`][] 模块。

### `--no-experimental-strip-types`

<!-- YAML
added: v22.6.0
changes:
  - version: v23.6.0
    pr-url: https://github.com/nodejs/node/pull/56350
    description: Type stripping is enabled by default.
-->

> Stability: 1.2 - Release candidate

禁用对 TypeScript 文件的实验性类型剥离。有关更多信息，请参阅 [TypeScript 类型剥离][TypeScript type-stripping] 文档。

### `--no-experimental-websocket`

<!-- YAML
added: v22.0.0
-->

禁止在全局作用域上暴露 {WebSocket}。

### `--no-extra-info-on-fatal-exception`

<!-- YAML
added: v17.0.0
-->

隐藏导致退出的致命异常的额外信息。

### `--no-force-async-hooks-checks`

<!-- YAML
added: v9.0.0
-->

禁用对 `async_hooks` 的运行时检查。当 `async_hooks` 启用时，这些仍将动态启用。

### `--no-global-search-paths`

<!-- YAML
added: v16.10.0
-->

不从全局路径（如 `$HOME/.node_modules` 和 `$NODE_PATH`）搜索模块。

### `--no-network-family-autoselection`

<!-- YAML
added: v19.4.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/46790
    description: The flag was renamed from `--no-enable-network-family-autoselection`
                 to `--no-network-family-autoselection`. The old name can still work as
                 an alias.
-->

禁用族自动选择算法，除非连接选项显式启用它。

### `--no-warnings`

<!-- YAML
added: v6.0.0
-->

静默所有进程警告（包括弃用警告）。

### `--node-memory-debug`

<!-- YAML
added:
  - v15.0.0
  - v14.18.0
-->

启用对 Node.js 内部内存泄漏的额外调试检查。这通常仅对调试 Node.js 本身的开发人员有用。

### `--openssl-config=file`

<!-- YAML
added: v6.9.0
-->

在启动时加载 OpenSSL 配置文件。除其他用途外，如果 Node.js 针对启用 FIPS 的 OpenSSL 构建，这可用于启用符合 FIPS 的加密。

### `--openssl-legacy-provider`

<!-- YAML
added:
  - v17.0.0
  - v16.17.0
-->

启用 OpenSSL 3.0 传统提供程序。有关更多信息，请参阅 [OSSL_PROVIDER-legacy][OSSL_PROVIDER-legacy]。

### `--openssl-shared-config`

<!-- YAML
added:
  - v18.5.0
  - v16.17.0
  - v14.21.0
-->

启用从 OpenSSL 配置文件中读取默认配置部分 `openssl_conf`。默认配置文件名为 `openssl.cnf`，但可以使用环境变量 `OPENSSL_CONF` 或使用命令行选项 `--openssl-config` 来更改。默认 OpenSSL 配置文件的位置取决于 OpenSSL 如何链接到 Node.js。共享 OpenSSL 配置可能会有不必要的后果，建议使用特定于 Node.js 的配置部分，即 `nodejs_conf`，当不使用此选项时，这是默认的。

### `--pending-deprecation`

<!-- YAML
added: v8.0.0
-->

发出待弃用警告。

待弃用通常与运行时弃用相同，但显著的区别是它们默认是关闭的，除非设置了 `--pending-deprecation` 命令行标志或 `NODE_PENDING_DEPRECATION=1` 环境变量，否则不会发出。待弃用用于提供一种选择性的“早期警告”机制，开发人员可以利用它来检测已弃用的 API 使用情况。

### `--permission`

<!-- YAML
added: v20.0.0
changes:
  - version:
    - v23.5.0
    - v22.13.0
    pr-url: https://github.com/nodejs/node/pull/56201
    description: Permission Model is now stable.
-->

为当前进程启用权限模型。启用后，以下权限受到限制：

* 文件系统 - 可通过 [`--allow-fs-read`][]、[`--allow-fs-write`][] 标志管理
* 子进程 - 可通过 [`--allow-child-process`][] 标志管理
* 工作线程 - 可通过 [`--allow-worker`][] 标志管理
* WASI - 可通过 [`--allow-wasi`][] 标志管理
* 插件 - 可通过 [`--allow-addons`][] 标志管理

### `--preserve-symlinks`

<!-- YAML
added: v6.3.0
-->

指示模块加载器在解析和缓存模块时保留符号链接。

默认情况下，当 Node.js 从符号链接到不同磁盘位置的路径加载模块时，Node.js 将解引用该链接，并使用模块的实际磁盘“真实路径”作为标识符和作为定位其他依赖模块的根路径。在大多数情况下，此默认行为是可接受的。但是，当使用符号链接的对等依赖项时，如下例所示，如果 `moduleA` 尝试将 `moduleB` 作为对等依赖项 require，则默认行为会导致抛出异常：

```text
{appDir}
 ├── app
 │   ├── index.js
 │   └── node_modules
 │       ├── moduleA -> {appDir}/moduleA
 │       └── moduleB
 │           ├── index.js
 │           └── package.json
 └── moduleA
     ├── index.js
     └── package.json
```

`--preserve-symlinks` 命令行标志指示 Node.js 对模块使用符号链接路径而不是真实路径，从而可以找到符号链接的对等依赖项。

但是，请注意，使用 `--preserve-symlinks` 可能会有其他副作用。具体来说，如果从依赖树中的多个位置链接符号链接的 _原生_ 模块，则这些模块可能无法加载（Node.js 会将它们视为两个单独的模块，并尝试多次加载该模块，导致抛出异常）。

`--preserve-symlinks` 标志不适用于主模块，这允许 `node --preserve-symlinks node_module/.bin/<foo>` 工作。要对主模块应用相同的行为，请同时使用 `--preserve-symlinks-main`。

### `--preserve-symlinks-main`

<!-- YAML
added: v10.2.0
-->

指示模块加载器在解析和缓存主模块（`require.main`）时保留符号链接。

此标志的存在是为了使主模块可以选择加入与 `--preserve-symlinks` 给予所有其他导入相同的行为；然而，它们是单独的标志，以便与旧版本的 Node.js 向后兼容。

`--preserve-symlinks-main` 不意味着 `--preserve-symlinks`；当不希望解析相对路径之前遵循符号链接时，除了 `--preserve-symlinks` 之外，请使用 `--preserve-symlinks-main`。

有关更多信息，请参阅 [`--preserve-symlinks`][]。

### `-p`, `--print "script"`

<!-- YAML
added: v0.6.4
changes:
  - version: v5.11.0
    pr-url: https://github.com/nodejs/node/pull/5348
    description: Built-in libraries are now available as predefined variables.
-->

与 `-e` 相同，但打印结果。

### `--prof`

<!-- YAML
added: v2.0.0
-->

生成 V8 分析器输出。

### `--prof-process`

<!-- YAML
added: v5.2.0
-->

处理使用 V8 选项 `--prof` 生成的 V8 分析器输出。

### `--redirect-warnings=file`

<!-- YAML
added: v8.0.0
-->

将进程警告写入给定文件而不是打印到 stderr。如果文件不存在，将被创建；如果存在，将被追加。如果尝试将警告写入文件时发生错误，则警告将改为写入 stderr。

`file` 名称可以是绝对路径。如果不是，它将写入的默认目录由 [`--diagnostic-dir`][] 命令行选项控制。

### `--report-compact`

<!-- YAML
added:
 - v13.12.0
 - v12.17.0
-->

以紧凑格式、单行 JSON 写入报告，比旨在供人类阅读的默认多行格式更容易被日志处理系统使用。

### `--report-dir=directory`, `report-directory=directory`

<!-- YAML
added: v11.8.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This option is no longer experimental.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/27312
    description: Changed from `--diagnostic-report-directory` to
                 `--report-directory`.
-->

生成报告的位置。

### `--report-exclude-env`

<!-- YAML
added:
  - v23.3.0
  - v22.13.0
-->

当传递 `--report-exclude-env` 时，生成的诊断报告将不包含 `environmentVariables` 数据。

### `--report-exclude-network`

<!-- YAML
added:
  - v22.0.0
  - v20.13.0
-->

从诊断报告中排除 `header.networkInterfaces`。默认情况下未设置此选项，并且网络接口被包括在内。

### `--report-filename=filename`

<!-- YAML
added: v11.8.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This option is no longer experimental.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/27312
    description: changed from `--diagnostic-report-filename` to
                 `--report-filename`.
-->

报告将写入的文件名。

如果文件名设置为 `'stdout'` 或 `'stderr'`，则报告将分别写入进程的 stdout 或 stderr。

### `--report-on-fatalerror`

<!-- YAML
added: v11.8.0
changes:
  - version:
    - v14.0.0
    - v13.14.0
    - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32496
    description: This option is no longer experimental.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/27312
    description: changed from `--diagnostic-report-on-fatalerror` to
                 `--report-on-fatalerror`.
-->

在致命错误（Node.js 运行时内部错误，如内存不足）导致应用程序终止时触发报告。用于检查各种诊断数据元素（如堆、栈、事件循环状态、资源消耗等）以推理致命错误。

### `--report-on-signal`

<!-- YAML
added: v11.8.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This option is no longer experimental.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/27312
    description: changed from `--diagnostic-report-on-signal` to
                 `--report-on-signal`.
-->

在接收到指定（或预定义）信号给运行的 Node.js 进程时生成报告。触发报告的信号通过 `--report-signal` 指定。

### `--report-signal=signal`

<!-- YAML
added: v11.8.0
changes:
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This option is no longer experimental.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/27312
    description: changed from `--diagnostic-report-signal` to
                 `--report-signal`.
-->

设置或重置用于报告生成的信号（Windows 上不支持）。默认信号是 `SIGUSR2`。

### `--report-uncaught-exception`

<!-- YAML
added: v11.8.0
changes:
  - version:
      - v18.8.0
      - v16.18.0
    pr-url: https://github.com/nodejs/node/pull/44208
    description: Report is not generated if the uncaught exception is handled.
  - version:
     - v13.12.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32242
    description: This option is no longer experimental.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/27312
    description: changed from `--diagnostic-report-uncaught-exception` to
                 `--report-uncaught-exception`.
-->

当进程由于未捕获的异常退出时启用报告生成。在与原生栈和其他运行时环境数据结合检查 JavaScript 栈时非常有用。

### `-r`, `--require module`

<!-- YAML
added: v1.6.0
changes:
  - version:
      - v23.0.0
      - v22.12.0
      - v20.19.0
    pr-url: https://github.com/nodejs/node/pull/51977
    description: This option also supports ECMAScript module.
-->

在启动时预加载指定的模块。

遵循 `require()` 的模块解析规则。`module` 可以是文件路径，也可以是节点模块名称。

使用 `--require` 预加载的模块将在使用 `--import` 预加载的模块之前运行。

模块被预加载到主线程以及任何工作线程、分叉进程或集群进程中。

### `--run`

<!-- YAML
added: v22.0.0
changes:
  - version: v22.3.0
    pr-url: https://github.com/nodejs/node/pull/53032
    description: NODE_RUN_SCRIPT_NAME environment variable is added.
  - version: v22.3.0
    pr-url: https://github.com/nodejs/node/pull/53058
    description: NODE_RUN_PACKAGE_JSON_PATH environment variable is added.
  - version: v22.3.0
    pr-url: https://github.com/nodejs/node/pull/53154
    description: Traverses up to the root directory and finds
                 a `package.json` file to run the command from, and updates
                 `PATH` environment variable accordingly.
-->

这从 package.json 的 `"scripts"` 对象运行指定的命令。如果提供了不存在的 `"command"`，它将列出可用的脚本。

`--run` 将遍历到根目录并找到一个 `package.json` 文件来运行命令。

`--run` 为当前目录的每个祖先将 `./node_modules/.bin` 添加到 `PATH`，以便在存在多个 `node_modules` 目录的不同文件夹中执行二进制文件，如果 `ancestor-folder/node_modules/.bin` 是一个目录。

`--run` 在包含相关 `package.json` 的目录中执行命令。

例如，以下命令将运行当前文件夹中 `package.json` 的 `test` 脚本：

```console
$ node --run test
```

您还可以向命令传递参数。`--` 之后的任何参数将附加到脚本：

```console
$ node --run test -- --verbose
```

#### 有意限制

`node --run` 并不意味着匹配 `npm run` 或其他包管理器的 `run` 命令的行为。Node.js 的实现有意更有限，以便专注于最常见用例的顶级性能。
其他 `run` 实现中故意排除的一些功能是：

* 除了指定的脚本外，还运行 `pre` 或 `post` 脚本。
* 定义包管理器特定的环境变量。

#### 环境变量

使用 `--run` 运行脚本时会设置以下环境变量：

* `NODE_RUN_SCRIPT_NAME`：正在运行的脚本的名称。例如，如果使用 `--run` 运行 `test`，则此变量的值将为 `test`。
* `NODE_RUN_PACKAGE_JSON_PATH`：正在处理的 `package.json` 的路径。

### `--secure-heap-min=n`

<!-- YAML
added: v15.6.0
-->

当使用 `--secure-heap` 时，`--secure-heap-min` 标志指定从安全堆分配的最小值。最小值为 `2`。最大值是 `--secure-heap` 或 `2147483647` 中的较小者。给定的值必须是 2 的幂。

### `--secure-heap=n`

<!-- YAML
added: v15.6.0
-->

初始化一个 `n` 字节的 OpenSSL 安全堆。初始化后，在密钥生成和其他操作期间，OpenSSL 会将安全堆用于选定类型的分配。例如，这有助于防止敏感信息因指针溢出或不足而泄漏。

安全堆是固定大小的，无法在运行时调整大小，因此，如果使用，选择足够大的堆以覆盖所有应用程序用途非常重要。

给定的堆大小必须是 2 的幂。任何小于 2 的值将禁用安全堆。

安全堆默认禁用。

安全堆在 Windows 上不可用。

有关更多详细信息，请参阅 [`CRYPTO_secure_malloc_init`][]。

### `--snapshot-blob=path`

<!-- YAML
added: v18.8.0
-->

> Stability: 1 - Experimental

与 `--build-snapshot` 一起使用时，`--snapshot-blob` 指定生成的快照 blob 写入的路径。如果未指定，生成的 blob 将写入当前工作目录中的 `snapshot.blob`。

不与 `--build-snapshot` 一起使用时，`--snapshot-blob` 指定用于恢复应用程序状态的 blob 的路径。

加载快照时，Node.js 会检查：

1. 运行的 Node.js 二进制文件的版本、架构和平台与生成快照的二进制文件完全相同。
2. V8 标志和 CPU 功能与生成快照的二进制文件兼容。

如果它们不匹配，Node.js 拒绝加载快照并以状态码 1 退出。

### `--test`

<!-- YAML
added:
  - v18.1.0
  - v16.17.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/46983
    description: The test runner is now stable.
  - version:
      - v19.2.0
      - v18.13.0
    pr-url: https://github.com/nodejs/node/pull/45214
    description: Test runner now supports running in watch mode.
-->

启动 Node.js 命令行测试运行器。此标志不能与 `--watch-path`、`--check`、`--eval`、`--interactive` 或检查器结合使用。有关更多详细信息，请参阅关于 [从命令行运行测试][running tests from the command line] 的文档。

### `--test-concurrency`

<!-- YAML
added:
  - v21.0.0
  - v20.10.0
  - v18.19.0
-->

测试运行器 CLI 将并发执行的最大测试文件数。如果 `--test-isolation` 设置为 `'none'`，则忽略此标志，并发性为一。否则，并发性默认为 `os.availableParallelism() - 1`。

### `--test-coverage-branches=threshold`

<!-- YAML
added: v22.8.0
-->

> Stability: 1 - Experimental

要求覆盖分支的最小百分比。如果代码覆盖率未达到指定的阈值，进程将以代码 `1` 退出。

### `--test-coverage-exclude`

<!-- YAML
added:
  - v22.5.0
-->

> Stability: 1 - Experimental

使用 glob 模式从代码覆盖率中排除特定文件，该模式可以匹配绝对和相对文件路径。

此选项可以指定多次以排除多个 glob 模式。

如果同时提供了 `--test-coverage-exclude` 和 `--test-coverage-include`，文件必须满足 **两者** 标准才能包含在覆盖率报告中。

默认情况下，所有匹配的测试文件都从覆盖率报告中排除。指定此选项将覆盖默认行为。

### `--test-coverage-functions=threshold`

<!-- YAML
added: v22.8.0
-->

> Stability: 1 - Experimental

要求覆盖函数的最小百分比。如果代码覆盖率未达到指定的阈值，进程将以代码 `1` 退出。

### `--test-coverage-include`

<!-- YAML
added:
  - v22.5.0
-->

> Stability: 1 - Experimental

使用 glob 模式在代码覆盖率中包含特定文件，该模式可以匹配绝对和相对文件路径。

此选项可以指定多次以包含多个 glob 模式。

如果同时提供了 `--test-coverage-exclude` 和 `--test-coverage-include`，文件必须满足 **两者** 标准才能包含在覆盖率报告中。

### `--test-coverage-lines=threshold`

<!-- YAML
added: v22.8.0
-->

> Stability: 1 - Experimental

要求覆盖行的最小百分比。如果代码覆盖率未达到指定的阈值，进程将以代码 `1` 退出。

### `--test-force-exit`

<!-- YAML
added:
  - v22.0.0
  - v20.14.0
-->

配置测试运行器在所有已知测试执行完成后退出进程，即使事件循环本应保持活动状态。

### `--test-global-setup=module`

<!-- YAML
added: v24.0.0
-->

> Stability: 1.0 - Early development

指定一个在所有测试执行之前将被评估的模块，并可用于为测试设置全局状态或固定装置。

有关更多详细信息，请参阅关于 [全局设置和拆卸][global setup and teardown] 的文档。

### `--test-isolation=mode`

<!-- YAML
added: v22.8.0
changes:
  - version: v23.6.0
    pr-url: https://github.com/nodejs/node/pull/56298
    description: This flag was renamed from `--experimental-test-isolation` to
                 `--test-isolation`.
-->

配置测试运行器中使用的测试隔离类型。当 `mode` 为 `'process'` 时，每个测试文件在单独的子进程中运行。当 `mode` 为 `'none'` 时，所有测试文件在与测试运行器相同的进程中运行。默认的隔离模式是 `'process'`。如果不存在 `--test` 标志，则忽略此标志。有关更多信息，请参阅 [测试运行器执行模型][test runner execution model] 部分。

### `--test-name-pattern`

<!-- YAML
added: v18.11.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/46983
    description: The test runner is now stable.
-->

一个正则表达式，配置测试运行器仅执行名称与提供模式匹配的测试。有关更多详细信息，请参阅关于 [按名称过滤测试][filtering tests by name] 的文档。

如果同时提供了 `--test-name-pattern` 和 `--test-skip-pattern`，测试必须满足 **两者** 要求才能被执行。

### `--test-only`

<!-- YAML
added:
  - v18.0.0
  - v16.17.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/46983
    description: The test runner is now stable.
-->

配置测试运行器仅执行设置了 `only` 选项的顶层测试。当禁用测试隔离时，不需要此标志。

### `--test-reporter`

<!-- YAML
added:
  - v19.6.0
  - v18.15.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/46983
    description: The test runner is now stable.
-->

运行测试时使用的测试报告器。有关更多详细信息，请参阅关于 [测试报告器][test reporters] 的文档。

### `--test-reporter-destination`

<!-- YAML
added:
  - v19.6.0
  - v18.15.0
changes:
  - version: v20.0.0
    pr-url: https://github.com/nodejs/node/pull/46983
    description: The test runner is now stable.
-->

相应测试报告器的目标。有关更多详细信息，请参阅关于 [测试报告器][test reporters] 的文档。

### `--test-rerun-failures`

<!-- YAML
added:
  - v24.7.0
-->

一个文件的路径，允许测试运行器在运行之间持久化测试套件的状态。测试运行器将使用此文件来确定哪些测试已经成功或失败，允许重新运行失败的测试而无需重新运行整个测试套件。如果该文件不存在，测试运行器将创建它。有关更多详细信息，请参阅关于 [测试重新运行][test reruns] 的文档。

### `--test-shard`

<!-- YAML
added:
  - v20.5.0
  - v18.19.0
-->

要执行的测试套件分片，格式为 `<index>/<total>`，其中

* `index` 是一个正整数，表示划分部分的索引。
* `total` 是一个正整数，表示划分部分的总数。

此命令将把所有测试文件分成 `total` 个相等的部分，并且只运行恰好位于 `index` 部分中的那些。

例如，要将测试套件分成三个部分，请使用：

```bash
node --test --test-shard=1/3
node --test --test-shard=2/3
node --test --test-shard=3/3
```

### `--test-skip-pattern`

<!-- YAML
added:
  - v22.1.0
-->

一个正则表达式，配置测试运行器跳过名称与提供模式匹配的测试。有关更多详细信息，请参阅关于 [按名称过滤测试][filtering tests by name] 的文档。

如果同时提供了 `--test-name-pattern` 和 `--test-skip-pattern`，测试必须满足 **两者** 要求才能被执行。

### `--test-timeout`

<!-- YAML
added:
  - v21.2.0
  - v20.11.0
-->

测试执行将在多少毫秒后失败。如果未指定，子测试从其父级继承此值。默认值为 `Infinity`。

### `--test-update-snapshots`

<!-- YAML
added: v22.3.0
changes:
  - version:
    - v23.4.0
    - v22.13.0
    pr-url: https://github.com/nodejs/node/pull/55897
    description: Snapshot testing is no longer experimental.
-->

重新生成测试运行器用于 [快照测试][snapshot testing] 的快照文件。

### `--throw-deprecation`

<!-- YAML
added: v0.11.14
-->

对弃用抛出错误。

### `--title=title`

<!-- YAML
added: v10.7.0
-->

在启动时设置 `process.title`。

### `--tls-cipher-list=list`

<!-- YAML
added: v4.0.0
-->

指定替代的默认 TLS 密码套件列表。要求 Node.js 构建时支持加密（默认）。

### `--tls-keylog=file`

<!-- YAML
added:
 - v13.2.0
 - v12.16.0
-->

将 TLS 密钥材料记录到文件。密钥材料采用 NSS `SSLKEYLOGFILE` 格式，可供软件（如 Wireshark）用于解密 TLS 流量。

### `--tls-max-v1.2`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
-->

将 [`tls.DEFAULT_MAX_VERSION`][] 设置为 'TLSv1.2'。用于禁用对 TLSv1.3 的支持。

### `--tls-max-v1.3`

<!-- YAML
added: v12.0.0
-->

将默认 [`tls.DEFAULT_MAX_VERSION`][] 设置为 'TLSv1.3'。用于启用对 TLSv1.3 的支持。

### `--tls-min-v1.0`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
-->

将默认 [`tls.DEFAULT_MIN_VERSION`][] 设置为 'TLSv1'。用于与旧的 TLS 客户端或服务器兼容。

### `--tls-min-v1.1`

<!-- YAML
added:
 - v12.0.0
 - v10.20.0
-->

将默认 [`tls.DEFAULT_MIN_VERSION`][] 设置为 'TLSv1.1'。用于与旧的 TLS 客户端或服务器兼容。

### `--tls-min-v1.2`

<!-- YAML
added:
 - v12.2.0
 - v10.20.0
-->

将默认 [`tls.DEFAULT_MIN_VERSION`][] 设置为 'TLSv1.2'。这是 12.x 及更高版本的默认值，但为了与旧版本的 Node.js 兼容，支持此选项。

### `--tls-min-v1.3`

<!-- YAML
added: v12.0.0
-->

将默认 [`tls.DEFAULT_MIN_VERSION`][] 设置为 'TLSv1.3'。用于禁用对 TLSv1.2 的支持，因为 TLSv1.3 不如 TLSv1.3 安全。

### `--trace-deprecation`

<!-- YAML
added: v0.8.0
-->

打印弃用的堆栈跟踪。

### `--trace-env`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
-->

打印当前 Node.js 实例中完成的任何环境变量访问的信息到 stderr，包括：

* Node.js 内部完成的环境变量读取。
* 形式为 `process.env.KEY = "SOME VALUE"` 的写入。
* 形式为 `process.env.KEY` 的读取。
* 形式为 `Object.defineProperty(process.env, 'KEY', {...})` 的定义。
* 形式为 `Object.hasOwn(process.env, 'KEY')`、`process.env.hasOwnProperty('KEY')` 或 `'KEY' in process.env` 的查询。
* 形式为 `delete process.env.KEY` 的删除。
* 形式为 `...process.env` 或 `Object.keys(process.env)` 的枚举。

仅打印被访问的环境变量的名称。不打印值。

要打印访问的堆栈跟踪，请使用 `--trace-env-js-stack` 和/或 `--trace-env-native-stack`。

### `--trace-env-js-stack`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
-->

除了 `--trace-env` 的功能外，还打印访问的 JavaScript 堆栈跟踪。

### `--trace-env-native-stack`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
-->

除了 `--trace-env` 的功能外，还打印访问的原生堆栈跟踪。

### `--trace-event-categories`

<!-- YAML
added: v7.7.0
-->

当使用 `--trace-events-enabled` 启用跟踪事件跟踪时，应跟踪的逗号分隔的类别列表。

### `--trace-event-file-pattern`

<!-- YAML
added: v9.8.0
-->

指定跟踪事件数据的文件路径的模板字符串，它支持 `${rotation}` 和 `${pid}`。

### `--trace-events-enabled`

<!-- YAML
added: v7.7.0
-->

启用跟踪事件跟踪信息的收集。

### `--trace-exit`

<!-- YAML
added:
 - v13.5.0
 - v12.16.0
-->

每当环境主动退出时打印堆栈跟踪，即调用 `process.exit()`。

### `--trace-require-module=mode`

<!-- YAML
added:
 - v23.5.0
 - v22.13.0
 - v20.19.0
-->

打印有关 [使用 `require()` 加载 ECMAScript 模块][Loading ECMAScript modules using `require()`] 的使用信息。

当 `mode` 为 `all` 时，打印所有使用情况。当 `mode` 为 `no-node-modules` 时，排除来自 `node_modules` 文件夹的使用情况。

### `--trace-sigint`

<!-- YAML
added:
 - v13.9.0
 - v12.17.0
-->

在 SIGINT 上打印堆栈跟踪。

### `--trace-sync-io`

<!-- YAML
added: v2.1.0
-->

在事件循环的第一轮之后检测到同步 I/O 时打印堆栈跟踪。

### `--trace-tls`

<!-- YAML
added: v12.2.0
-->

将 TLS 数据包跟踪信息打印到 `stderr`。这可用于调试 TLS 连接问题。

### `--trace-uncaught`

<!-- YAML
added: v13.1.0
-->

打印未捕获异常的堆栈跟踪；通常，打印与创建 `Error` 相关的堆栈跟踪，而此选项使 Node.js 也打印与抛出值相关的堆栈跟踪（该值不需要是 `Error` 实例）。

启用此选项可能会对垃圾回收行为产生负面影响。

### `--trace-warnings`

<!-- YAML
added: v6.0.0
-->

打印进程警告（包括弃用）的堆栈跟踪。

### `--track-heap-objects`

<!-- YAML
added: v2.4.0
-->

跟踪堆对象分配以获取堆快照。

### `--unhandled-rejections=mode`

<!-- YAML
added:
  - v12.0.0
  - v10.17.0
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/33021
    description: Changed default mode to `throw`. Previously, a warning was
                 emitted.
-->

使用此标志可以更改发生未处理的拒绝时应发生的情况。可以选择以下模式之一：

* `throw`：发出 [`unhandledRejection`][]。如果未设置此钩子，则将未处理的拒绝作为未捕获的异常抛出。这是默认值。
* `strict`：将未处理的拒绝作为未捕获的异常抛出。如果异常被处理，则发出 [`unhandledRejection`][]。
* `warn`：无论是否设置了 [`unhandledRejection`][] 钩子，始终触发警告，但不打印弃用警告。
* `warn-with-error-code`：发出 [`unhandledRejection`][]。如果未设置此钩子，则触发警告，并将进程退出代码设置为 1。
* `none`：静默所有警告。

如果拒绝发生在命令行入口点的 ES 模块静态加载阶段，它将始终将其作为未捕获的异常抛出。

### `--use-bundled-ca`, `--use-openssl-ca`

<!-- YAML
added: v6.11.0
-->

使用当前 Node.js 版本提供的捆绑的 Mozilla CA 存储或使用 OpenSSL 的默认 CA 存储。默认存储可在构建时选择。

由 Node.js 提供的捆绑 CA 存储是 Mozilla CA 存储的快照，在发布时固定。它在所有支持的平台上都是相同的。

使用 OpenSSL 存储允许对存储进行外部修改。对于大多数 Linux 和 BSD 发行版，此存储由发行版维护者和系统管理员维护。OpenSSL CA 存储位置取决于 OpenSSL 库的配置，但可以在运行时使用环境变量更改。

参见 `SSL_CERT_DIR` 和 `SSL_CERT_FILE`。

### `--use-env-proxy`

<!-- YAML
added: v24.5.0
-->

> Stability: 1.1 - Active Development

启用后，Node.js 将在启动期间解析 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY` 环境变量，并通过指定的代理隧道请求。

这等同于设置 [`NODE_USE_ENV_PROXY=1`][] 环境变量。当两者都设置时，`--use-env-proxy` 优先。

### `--use-largepages=mode`

<!-- YAML
added:
 - v13.6.0
 - v12.17.0
-->

在启动时将 Node.js 静态代码重新映射到大内存页。如果目标系统支持，这将导致 Node.js 静态代码被移动到 2 MiB 页面而不是 4 KiB 页面。

`mode` 的有效值如下：

* `off`：不会尝试映射。这是默认值。
* `on`：如果操作系统支持，将尝试映射。映射失败将被忽略，并且消息将打印到标准错误。
* `silent`：如果操作系统支持，将尝试映射。映射失败将被忽略，并且不会报告。

### `--use-system-ca`

<!-- YAML
added: v23.8.0
changes:
  - version: v23.9.0
    pr-url: https://github.com/nodejs/node/pull/57009
    description: Added support on non-Windows and non-macOS.
-->

Node.js 使用系统存储中存在的受信任 CA 证书以及 `--use-bundled-ca` 选项和 `NODE_EXTRA_CA_CERTS` 环境变量。在 Windows 和 macOS 以外的平台上，这会从 OpenSSL 信任的目录和文件加载证书，类似于 `--use-openssl-ca`，不同之处在于它在首次加载后缓存证书。

在 Windows 和 macOS 上，证书信任策略计划遵循 [Chromium 的本地受信任证书策略][Chromium's policy for locally trusted certificates]：

在 macOS 上，尊重以下设置：

* 默认和系统钥匙串
  * 信任：
    * 任何“使用此证书时”标志设置为“始终信任”的证书，或
    * 任何“安全套接字层 (SSL)”标志设置为“始终信任”的证书。
  * 不信任：
    * 任何“使用此证书时”标志设置为“从不信任”的证书，或
    * 任何“安全套接字层 (SSL)”标志设置为“从不信任”的证书。

在 Windows 上，尊重以下设置（与 Chromium 的策略不同，目前不支持不信任和中间 CA）：

* 本地计算机（通过 `certlm.msc` 访问）
  * 信任：
    * 受信任的根证书颁发机构
    * 受信任的人员
    * 企业信任 -> 企业 -> 受信任的根证书颁发机构
    * 企业信任 -> 企业 -> 受信任的人员
    * 企业信任 -> 组策略 -> 受信任的根证书颁发机构
    * 企业信任 -> 组策略 -> 受信任的人员
* 当前用户（通过 `certmgr.msc` 访问）
  * 信任：
    * 受信任的根证书颁发机构
    * 企业信任 -> 组策略 -> 受信任的根证书颁发机构

在 Windows 和 macOS 上，Node.js 将检查用户对证书的设置是否不禁止它们用于 TLS 服务器身份验证，然后再使用它们。

在其他系统上，Node.js 从 Node.js 链接到的 OpenSSL 版本尊重的默认证书文件（通常是 `/etc/ssl/cert.pem`）和默认证书目录（通常是 `/etc/ssl/certs`）加载证书。这通常适用于主要 Linux 发行版和其他类 Unix 系统的约定。如果覆盖的 OpenSSL 环境变量（通常取决于 Node.js 链接到的 OpenSSL 的配置，是 `SSL_CERT_FILE` 和 `SSL_CERT_DIR`）已设置，则将使用指定的路径来加载证书而不是。这些环境变量可以用作变通方法，如果 Node.js 链接到的 OpenSSL 版本使用的常规路径由于某种原因与用户拥有的系统配置不一致。

### `--v8-options`

<!-- YAML
added: v0.1.3
-->

打印 V8 命令行选项。

### `--v8-pool-size=num`

<!-- YAML
added: v5.10.0
-->

设置 V8 的线程池大小，该线程池将用于分配后台作业。

如果设置为 `0`，则 Node.js 将基于并行量的估计选择线程池的适当大小。

并行量是指给定机器中可以同时执行的计算数量。通常，它与 CPU 数量相同，但在虚拟机或容器等环境中可能会有所不同。

### `-v`, `--version`

<!-- YAML
added: v0.1.3
-->

打印节点的版本。

### `--watch`

<!-- YAML
added:
  - v18.11.0
  - v16.19.0
changes:
  - version:
    - v22.0.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52074
    description: Watch mode is now stable.
  - version:
      - v19.2.0
      - v18.13.0
    pr-url: https://github.com/nodejs/node/pull/45214
    description: Test runner now supports running in watch mode.
-->

以监视模式启动 Node.js。在监视模式下，监视文件中的更改会导致 Node.js 进程重启。默认情况下，监视模式将监视入口点以及任何 required 或 imported 的模块。使用 `--watch-path` 指定要监视的路径。

此标志不能与 `--check`、`--eval`、`--interactive` 或 REPL 结合使用。

注意：`--watch` 标志需要一个文件路径作为参数，并且与 `--run` 或内联脚本输入不兼容，因为 `--run` 优先并忽略监视模式。如果未提供文件，Node.js 将以状态码 `9` 退出。

```bash
node --watch index.js
```

### `--watch-kill-signal`

<!-- YAML
added:
  - v24.4.0
-->

> Stability: 1.1 - Active Development

自定义在监视模式重启时发送给进程的信号。

```bash
node --watch --watch-kill-signal SIGINT test.js
```

### `--watch-path`

<!-- YAML
added:
  - v18.11.0
  - v16.19.0
changes:
  - version:
    - v22.0.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52074
    description: Watch mode is now stable.
-->

以监视模式启动 Node.js 并指定要监视的路径。在监视模式下，监视路径中的更改会导致 Node.js 进程重启。这将关闭对 required 或 imported 模块的监视，即使与 `--watch` 结合使用也是如此。

此标志不能与 `--check`、`--eval`、`--interactive`、`--test` 或 REPL 结合使用。

注意：使用 `--watch-path` 隐式启用 `--watch`，这需要一个文件路径并且与 `--run` 不兼容，因为 `--run` 优先并忽略监视模式。

```bash
node --watch-path=./src --watch-path=./tests index.js
```

此选项仅在 macOS 和 Windows 上受支持。在不支持该选项的平台上使用时会抛出 `ERR_FEATURE_UNAVAILABLE_ON_PLATFORM` 异常。

### `--watch-preserve-output`

<!-- YAML
added:
  - v19.3.0
  - v18.13.0
-->

在监视模式重启进程时禁用清除控制台。

```bash
node --watch --watch-preserve-output test.js
```

### `--zero-fill-buffers`

<!-- YAML
added: v6.0.0
-->

自动零填充所有新分配的 [`Buffer`][] 和 [`SlowBuffer`][] 实例。

## 环境变量

> Stability: 2 - Stable

### `FORCE_COLOR=[1, 2, 3]`

`FORCE_COLOR` 环境变量用于启用 ANSI 彩色输出。值可以是：

* `1`、`true` 或空字符串 `''` 表示 16 色支持，
* `2` 表示 256 色支持，或
* `3` 表示 1600 万色支持。

当使用 `FORCE_COLOR` 并设置为支持的值时，`NO_COLOR` 和 `NODE_DISABLE_COLORS` 环境变量都会被忽略。

任何其他值将导致彩色输出被禁用。

### `NODE_COMPILE_CACHE=dir`

<!-- YAML
added: v22.1.0
-->

> Stability: 1.1 - Active Development

为 Node.js 实例启用 [模块编译缓存][module compile cache]。有关详细信息，请参阅 [模块编译缓存][module compile cache] 的文档。

### `NODE_DEBUG=module[,…]`

<!-- YAML
added: v0.1.32
-->

`','` 分隔的核心模块列表，应打印调试信息。

### `NODE_DEBUG_NATIVE=module[,…]`

`','` 分隔的核心 C++ 模块列表，应打印调试信息。

### `NODE_DISABLE_COLORS=1`

<!-- YAML
added: v0.3.0
-->

设置后，REPL 中将不使用颜色。

### `NODE_DISABLE_COMPILE_CACHE=1`

<!-- YAML
added: v22.8.0
-->

> Stability: 1.1 - Active Development

为 Node.js 实例禁用 [模块编译缓存][module compile cache]。有关详细信息，请参阅 [模块编译缓存][module compile cache] 的文档。

### `NODE_EXTRA_CA_CERTS=file`

<!-- YAML
added: v7.3.0
-->

设置后，众所周知的“根”CA（如 VeriSign）将使用 `file` 中的额外证书进行扩展。该文件应包含一个或多个受信任的 PEM 格式证书。如果文件丢失或格式错误，将（一次）通过 [`process.emitWarning()`][emit_warning] 发出消息，但任何错误 otherwise 将被忽略。

当为 TLS 或 HTTPS 客户端或服务器显式指定 `ca` 选项属性时，既不使用众所周知的证书也不使用额外证书。

当 `node` 作为 setuid root 运行或设置了 Linux 文件功能时，此环境变量将被忽略。

`NODE_EXTRA_CA_CERTS` 环境变量仅在 Node.js 进程首次启动时读取。在运行时使用 `process.env.NODE_EXTRA_CA_CERTS` 更改值对当前进程没有影响。

### `NODE_ICU_DATA=file`

<!-- YAML
added: v0.11.15
-->

ICU（`Intl` 对象）数据的数据路径。在使用 small-icu 支持编译时将扩展链接的数据。

### `NODE_NO_WARNINGS=1`

<!-- YAML
added: v6.11.0
-->

设置为 `1` 时，进程警告被静默。

### `NODE_OPTIONS=options...`

<!-- YAML
added: v8.0.0
-->

一个空格分隔的命令行选项列表。`options...` 在命令行选项之前解释，因此命令行选项将覆盖或复合 `options...` 中的任何内容。如果使用了环境中不允许的选项（如 `-p` 或脚本文件），Node.js 将退出并报错。

如果选项值包含空格，可以使用双引号进行转义：

```bash
NODE_OPTIONS='--require "./my path/file.js"'
```

作为命令行选项传递的单例标志将覆盖传递给 `NODE_OPTIONS` 的相同标志：

```bash
# 检查器将在端口 5555 上可用
NODE_OPTIONS='--inspect=localhost:4444' node --inspect=localhost:5555
```

可以传递多次的标志将被视为其 `NODE_OPTIONS` 实例首先传递，然后是其命令行实例：

```bash
NODE_OPTIONS='--require "./a.js"' node --require "./b.js"
# 等同于：
node --require "./a.js" --require "./b.js"
```

允许的 Node.js 选项在以下列表中。如果一个选项同时支持 --XX 和 --no-XX 变体，则两者都支持，但下面列表中仅包含一个。

<!-- node-options-node start -->

* `--allow-addons`
* `--allow-child-process`
* `--allow-fs-read`
* `--allow-fs-write`
* `--allow-wasi`
* `--allow-worker`
* `--conditions`, `-C`
* `--cpu-prof-dir`
* `--cpu-prof-interval`
* `--cpu-prof-name`
* `--cpu-prof`
* `--diagnostic-dir`
* `--disable-proto`
* `--disable-sigusr1`
* `--disable-warning`
* `--disable-wasm-trap-handler`
* `--dns-result-order`
* `--enable-fips`
* `--enable-network-family-autoselection`
* `--enable-source-maps`
* `--entry-url`
* `--experimental-abortcontroller`
* `--experimental-addon-modules`
* `--experimental-detect-module`
* `--experimental-eventsource`
* `--experimental-import-meta-resolve`
* `--experimental-json-modules`
* `--experimental-loader`
* `--experimental-modules`
* `--experimental-print-required-tla`
* `--experimental-require-module`
* `--experimental-shadow-realm`
* `--experimental-specifier-resolution`
* `--experimental-test-isolation`
* `--experimental-top-level-await`
* `--experimental-transform-types`
* `--experimental-vm-modules`
* `--experimental-wasi-unstable-preview1`
* `--experimental-webstorage`
* `--force-context-aware`
* `--force-fips`
* `--force-node-api-uncaught-exceptions-policy`
* `--frozen-intrinsics`
* `--heap-prof-dir`
* `--heap-prof-interval`
* `--heap-prof-name`
* `--heap-prof`
* `--heapsnapshot-near-heap-limit`
* `--heapsnapshot-signal`
* `--http-parser`
* `--icu-data-dir`
* `--import`
* `--input-type`
* `--insecure-http-parser`
* `--inspect-brk`
* `--inspect-port`, `--debug-port`
* `--inspect-publish-uid`
* `--inspect-wait`
* `--inspect`
* `--localstorage-file`
* `--max-http-header-size`
* `--max-old-space-size-percentage`
* `--napi-modules`
* `--network-family-autoselection-attempt-timeout`
* `--no-addons`
* `--no-async-context-frame`
* `--no-deprecation`
* `--no-experimental-global-navigator`
* `--no-experimental-repl-await`
* `--no-experimental-sqlite`
* `--no-experimental-strip-types`
* `--no-experimental-websocket`
* `--no-extra-info-on-fatal-exception`
* `--no-force-async-hooks-checks`
* `--no-global-search-paths`
* `--no-network-family-autoselection`
* `--no-warnings`
* `--node-memory-debug`
* `--openssl-config`
* `--openssl-legacy-provider`
* `--openssl-shared-config`
* `--pending-deprecation`
* `--permission`
* `--preserve-symlinks-main`
* `--preserve-symlinks`
* `--prof-process`
* `--redirect-warnings`
* `--report-compact`
* `--report-dir`, `--report-directory`
* `--report-exclude-env`
* `--report-exclude-network`
* `--report-filename`
* `--report-on-fatalerror`
* `--report-on-signal`
* `--report-signal`
* `--report-uncaught-exception`
* `--require`, `-r`
* `--secure-heap-min`
* `--secure-heap`
* `--snapshot-blob`
* `--test-coverage-branches`
* `--test-coverage-exclude`
* `--test-coverage-functions`
* `--test-coverage-include`
* `--test-coverage-lines`
* `--test-global-setup`
* `--test-isolation`
* `--test-name-pattern`
* `--test-only`
* `--test-reporter-destination`
* `--test-reporter`
* `--test-rerun-failures`
* `--test-shard`
* `--test-skip-pattern`
* `--throw-deprecation`
* `--title`
* `--tls-cipher-list`
* `--tls-keylog`
* `--tls-max-v1.2`
* `--tls-max-v1.3`
* `--tls-min-v1.0`
* `--tls-min-v1.1`
* `--tls-min-v1.2`
* `--tls-min-v1.3`
* `--trace-deprecation`
* `--trace-env-js-stack`
* `--trace-env-native-stack`
* `--trace-env`
* `--trace-event-categories`
* `--trace-event-file-pattern`
* `--trace-events-enabled`
* `--trace-exit`
* `--trace-require-module`
* `--trace-sigint`
* `--trace-sync-io`
* `--trace-tls`
* `--trace-uncaught`
* `--trace-warnings`
* `--track-heap-objects`
* `--unhandled-rejections`
* `--use-bundled-ca`
* `--use-env-proxy`
* `--use-largepages`
* `--use-openssl-ca`
* `--use-system-ca`
* `--v8-pool-size`
* `--watch-kill-signal`
* `--watch-path`
* `--watch-preserve-output`
* `--watch`
* `--zero-fill-buffers`

<!-- node-options-node end -->

允许的 V8 选项是：

<!-- node-options-v8 start -->

* `--abort-on-uncaught-exception`
* `--disallow-code-generation-from-strings`
* `--enable-etw-stack-walking`
* `--expose-gc`
* `--interpreted-frames-native-stack`
* `--jitless`
* `--max-old-space-size`
* `--max-semi-space-size`
* `--perf-basic-prof-only-functions`
* `--perf-basic-prof`
* `--perf-prof-unwinding-info`
* `--perf-prof`
* `--stack-trace-limit`

<!-- node-options-v8 end -->

<!-- node-options-others start -->

`--perf-basic-prof-only-functions`、`--perf-basic-prof`、`--perf-prof-unwinding-info` 和 `--perf-prof` 仅在 Linux 上可用。

`--enable-etw-stack-walking` 仅在 Windows 上可用。

<!-- node-options-others end -->

### `NODE_PATH=path[:…]`

<!-- YAML
added: v0.1.32
-->

`':'` 分隔的目录列表，前缀到模块搜索路径。

在 Windows 上，这是一个 `';'` 分隔的列表。

### `NODE_PENDING_DEPRECATION=1`

<!-- YAML
added: v8.0.0
-->

设置为 `1` 时，发出待弃用警告。

待弃用通常与运行时弃用相同，但显著的区别是它们默认是关闭的，除非设置了 `--pending-deprecation` 命令行标志或 `NODE_PENDING_DEPRECATION=1` 环境变量，否则不会发出。待弃用用于提供一种选择性的“早期警告”机制，开发人员可以利用它来检测已弃用的 API 使用情况。

### `NODE_PENDING_PIPE_INSTANCES=instances`

设置管道服务器等待连接时待处理管道实例句柄的数量。此设置仅适用于 Windows。

### `NODE_PRESERVE_SYMLINKS=1`

<!-- YAML
added: v7.1.0
-->

设置为 `1` 时，指示模块加载器在解析和缓存模块时保留符号链接。

### `NODE_REDIRECT_WARNINGS=file`

<!-- YAML
added: v8.0.0
-->

设置后，进程警告将发送到给定文件而不是打印到 stderr。如果文件不存在，将被创建；如果存在，将被追加。如果尝试将警告写入文件时发生错误，则警告将改为写入 stderr。这等同于使用 `--redirect-warnings=file` 命令行标志。

### `NODE_REPL_EXTERNAL_MODULE=file`

<!-- YAML
added:
 - v13.0.0
 - v12.16.0
changes:
  - version:
     - v22.3.0
     - v20.16.0
    pr-url: https://github.com/nodejs/node/pull/52905
    description:
      Remove the possibility to use this env var with
      kDisableNodeOptionsEnv for embedders.
-->

指向 Node.js 模块的路径，该模块将代替内置 REPL 加载。将此值覆盖为空字符串（`''`）将使用内置 REPL。

### `NODE_REPL_HISTORY=file`

<!-- YAML
added: v3.0.0
-->

用于存储持久 REPL 历史记录的文件路径。默认路径是 `~/.node_repl_history`，由此变量覆盖。将值设置为空字符串（`''` 或 `' '`）将禁用持久 REPL 历史记录。

### `NODE_SKIP_PLATFORM_CHECK=value`

<!-- YAML
added: v14.5.0
-->

如果 `value` 等于 `'1'`，则在 Node.js 启动期间跳过对受支持平台的检查。Node.js 可能无法正确执行。在不支持的平台上遇到的任何问题将不会修复。

### `NODE_TEST_CONTEXT=value`

如果 `value` 等于 `'child'`，测试报告器选项将被覆盖，测试输出将以 TAP 格式发送到 stdout。如果提供任何其他值，Node.js 不保证使用的报告器格式或其稳定性。

### `NODE_TLS_REJECT_UNAUTHORIZED=value`

如果 `value` 等于 `'0'`，则对 TLS 连接禁用证书验证。这使得 TLS 以及扩展的 HTTPS 不安全。强烈建议不要使用此环境变量。

### `NODE_USE_ENV_PROXY=1`

<!-- YAML
added: v24.0.0
-->

> Stability: 1.1 - Active Development

启用后，Node.js 将在启动期间解析 `HTTP_PROXY`、`HTTPS_PROXY` 和 `NO_PROXY` 环境变量，并通过指定的代理隧道请求。

这也可以使用 [`--use-env-proxy`][] 命令行标志启用。当两者都设置时，`--use-env-proxy` 优先。

### `NODE_USE_SYSTEM_CA=1`

<!-- YAML
added: v24.6.0
-->

Node.js 使用系统存储中存在的受信任 CA 证书以及 `--use-bundled-ca` 选项和 `NODE_EXTRA_CA_CERTS` 环境变量。

这也可以使用 [`--use-system-ca`][] 命令行标志启用。当两者都设置时，`--use-system-ca` 优先。

### `NODE_V8_COVERAGE=dir`

设置后，Node.js 将开始将 [V8 JavaScript 代码覆盖率][V8 JavaScript code coverage] 和 [Source Map][] 数据输出到作为参数提供的目录（覆盖率信息以 JSON 格式写入带有 `coverage` 前缀的文件）。

`NODE_V8_COVERAGE` 会自动传播到子进程，使得检测调用 `child_process.spawn()` 系列函数的应用程序更加容易。`NODE_V8_COVERAGE` 可以设置为空字符串，以防止传播。

#### 覆盖率输出

覆盖率作为 [ScriptCoverage][] 对象的数组输出在顶层键 `result` 上：

```json
{
  "result": [
    {
      "scriptId": "67",
      "url": "internal/tty.js",
      "functions": []
    }
  ]
}
```

#### Source map 缓存

> Stability: 1 - Experimental

如果找到，source map 数据将附加到 JSON 覆盖率对象的顶层键 `source-map-cache` 上。

`source-map-cache` 是一个对象，其键表示从中提取 source map 的文件，值包括原始 source-map URL（在键 `url` 中）、解析的 Source Map v3 信息（在键 `data` 中）以及源文件的行长度（在键 `lineLengths` 中）。

```json
{
  "result": [
    {
      "scriptId": "68",
      "url": "file:///absolute/path/to/source.js",
      "functions": []
    }
  ],
  "source-map-cache": {
    "file:///absolute/path/to/source.js": {
      "url": "./path-to-map.json",
      "data": {
        "version": 3,
        "sources": [
          "file:///absolute/path/to/original.js"
        ],
        "names": [
          "Foo",
          "console",
          "info"
        ],
        "mappings": "MAAMA,IACJC,YAAaC",
        "sourceRoot": "./"
      },
      "lineLengths": [
        13,
        62,
        38,
        27
      ]
    }
  }
}
```

### `NO_COLOR=<any>`

[`NO_COLOR`][] 是 `NODE_DISABLE_COLORS` 的别名。环境变量的值是任意的。

### `OPENSSL_CONF=file`

<!-- YAML
added: v6.11.0
-->

在启动时加载 OpenSSL 配置文件。除其他用途外，如果 Node.js 使用 `./configure --openssl-fips` 构建，这可用于启用符合 FIPS 的加密。

如果使用了 [`--openssl-config`][] 命令行选项，则忽略环境变量。

### `SSL_CERT_DIR=dir`

<!-- YAML
added: v7.7.0
-->

如果启用了 `--use-openssl-ca`，或者在 macOS 和 Windows 以外的平台上启用了 `--use-system-ca`，这将覆盖并设置 OpenSSL 包含受信任证书的目录。

请注意，除非显式设置了子环境，否则此环境变量将由任何子进程继承，如果它们使用 OpenSSL，可能会导致它们信任与 node 相同的 CA。

### `SSL_CERT_FILE=file`

<!-- YAML
added: v7.7.0
-->

如果启用了 `--use-openssl-ca`，或者在 macOS 和 Windows 以外的平台上启用了 `--use-system-ca`，这将覆盖并设置 OpenSSL 包含受信任证书的文件。

请注意，除非显式设置了子环境，否则此环境变量将由任何子进程继承，如果它们使用 OpenSSL，可能会导致它们信任与 node 相同的 CA。

### `TZ`

<!-- YAML
added: v0.0.1
changes:
  - version:
     - v16.2.0
    pr-url: https://github.com/nodejs/node/pull/38642
    description:
      Changing the TZ variable using process.env.TZ = changes the timezone
      on Windows as well.
  - version:
     - v13.0.0
    pr-url: https://github.com/nodejs/node/pull/20026
    description:
      Changing the TZ variable using process.env.TZ = changes the timezone
      on POSIX systems.
-->

`TZ` 环境变量用于指定时区配置。

虽然 Node.js 不支持 [其他环境中处理 `TZ` 的所有各种方式][ways that `TZ` is handled in other environments]，但它确实支持基本的 [时区 ID][timezone IDs]（如 `'Etc/UTC'`、`'Europe/Paris'` 或 `'America/New_York'`）。它可能支持一些其他缩写或别名，但强烈不建议使用这些，并且不能保证。

```console
$ TZ=Europe/Dublin node -pe "new Date().toString()"
Wed May 12 2021 20:30:48 GMT+0100 (Irish Standard Time)
```

### `UV_THREADPOOL_SIZE=size`

将 libuv 线程池中使用的线程数设置为 `size` 个线程。

Node.js 尽可能使用异步系统 API，但在它们不存在的地方，使用 libuv 的线程池基于同步系统 API 创建异步节点 API。使用线程池的 Node.js API 有：

* 所有 `fs` API，除了文件监视器 API 和那些显式同步的 API
* 异步加密 API，如 `crypto.pbkdf2()`、`crypto.scrypt()`、`crypto.randomBytes()`、`crypto.randomFill()`、`crypto.generateKeyPair()`
* `dns.lookup()`
* 所有 `zlib` API，除了那些显式同步的 API

因为 libuv 的线程池具有固定大小，这意味着如果出于任何原因这些 API 中的任何一个花费很长时间，其他（看似无关的）在 libuv 线程池中运行的 API 将经历性能下降。为了缓解此问题，一个潜在的解决方案是通过将 `'UV_THREADPOOL_SIZE'` 环境变量设置为大于 `4`（其当前默认值）的值来增加 libuv 线程池的大小。然而，从进程内部使用 `process.env.UV_THREADPOOL_SIZE=size` 设置此值不能保证工作，因为线程池将在运行时初始化期间创建，远在用户代码运行之前。有关更多信息，请参阅 [libuv 线程池文档][libuv threadpool documentation]。

## 有用的 V8 选项

V8 有自己的一组 CLI 选项。任何提供给 `node` 的 V8 CLI 选项都将传递给 V8 处理。V8 的选项 _没有稳定性保证_。V8 团队自己并不认为它们是正式 API 的一部分，并保留随时更改它们的权利。同样，它们也不在 Node.js 稳定性保证的范围内。许多 V8 选项仅对 V8 开发人员有意义。尽管如此，有一小部分 V8 选项广泛适用于 Node.js，它们在此记录：

<!-- v8-options start -->

### `--abort-on-uncaught-exception`

### `--disallow-code-generation-from-strings`

### `--enable-etw-stack-walking`

### `--expose-gc`

### `--harmony-shadow-realm`

### `--interpreted-frames-native-stack`

### `--jitless`

<!-- Anchor to make sure old links find a target -->

<a id="--max-old-space-sizesize-in-megabytes"></a>

### `--max-old-space-size=SIZE` (in MiB)

设置 V8 旧内存部分的最大内存大小。当内存消耗接近限制时，V8 将花费更多时间在垃圾回收上，以释放未使用的内存。

在具有 2 GiB 内存的机器上，考虑将其设置为 1536（1.5 GiB）以为其他用途留出一些内存并避免交换。

```bash
node --max-old-space-size=1536 index.js
```

<!-- Anchor to make sure old links find a target -->

<a id="--max-semi-space-sizesize-in-megabytes"></a>

### `--max-semi-space-size=SIZE` (in MiB)

以 MiB（mebibytes）设置 V8 的 [清除垃圾回收器][scavenge garbage collector] 的 [半空间][semi-space] 最大大小。增加半空间的最大大小可能会提高 Node.js 的吞吐量，但代价是更多内存消耗。

由于 V8 堆的年轻代大小是半空间大小的三倍（参见 V8 中的 [`YoungGenerationSizeFromSemiSpaceSize`][]），半空间增加 1 MiB 适用于三个独立的半空间中的每一个，并导致堆大小增加 3 MiB。吞吐量的改进取决于您的工作负载（参见 [#42511][]）。

默认值取决于内存限制。例如，在具有 512 MiB 内存限制的 64 位系统上，半空间的最大大小默认为 1 MiB。对于最多并包括 2GiB 的内存限制，在 64 位系统上半空间的最大默认大小将小于 16 MiB。

要为您的应用程序获得最佳配置，您应该在为应用程序运行基准测试时尝试不同的 max-semi-space-size 值。

例如，在 64 位系统上进行基准测试：

```bash
for MiB in 16 32 64 128; do
    node --max-semi-space-size=$MiB index.js
done
```

### `--perf-basic-prof`

### `--perf-basic-prof-only-functions`

### `--perf-prof`

### `--perf-prof-unwinding-info`

### `--prof`

### `--security-revert`

### `--stack-trace-limit=limit`

错误堆栈跟踪中收集的最大堆栈帧数。将其设置为 0 将禁用堆栈跟踪收集。默认值为 10。

```bash
node --stack-trace-limit=12 -p -e "Error.stackTraceLimit" # 打印 12
```

<!-- v8-options end -->

[#42511]: https://github.com/nodejs/node/issues/42511
[Chrome DevTools Protocol]: https://chromedevtools.github.io/devtools-protocol/
[Chromium's policy for locally trusted certificates]: https://chromium.googlesource.com/chromium/src/+/main/net/data/ssl/chrome_root_store/faq.md#does-the-chrome-certificate-verifier-consider-local-trust-decisions
[CommonJS]: modules.md
[CommonJS module]: modules.md
[DEP0025 warning]: deprecations.md#dep0025-requirenodesys
[ECMAScript module]: esm.md#modules-ecmascript-modules
[EventSource Web API]: https://html.spec.whatwg.org/multipage/server-sent-events.html#server-sent-events
[ExperimentalWarning: `vm.measureMemory` is an experimental feature]: vm.md#vmmeasurememoryoptions
[File System Permissions]: permissions.md#file-system-permissions
[Loading ECMAScript modules using `require()`]: modules.md#loading-ecmascript-modules-using-require
[Module customization hooks]: module.md#customization-hooks
[Module customization hooks: enabling]: module.md#enabling
[Modules loaders]: packages.md#modules-loaders
[Navigator API]: globals.md#navigator
[Node.js issue tracker]: https://github.com/nodejs/node/issues
[OSSL_PROVIDER-legacy]: https://www.openssl.org/docs/man3.0/man7/OSSL_PROVIDER-legacy.html
[Permission Model]: permissions.md#permission-model
[REPL]: repl.md
[ScriptCoverage]: https://chromedevtools.github.io/devtools-protocol/tot/Profiler#type-ScriptCoverage
[ShadowRealm]: https://github.com/tc39/proposal-shadowrealm
[Source Map]: https://tc39.es/ecma426/
[TypeScript type-stripping]: typescript.md#type-stripping
[V8 Inspector integration for Node.js]: debugger.md#v8-inspector-integration-for-nodejs
[V8 JavaScript code coverage]: https://v8project.blogspot.com/2017/12/javascript-code-coverage.html
[`"type"`]: packages.md#type
[`--allow-addons`]: #--allow-addons
[`--allow-child-process`]: #--allow-child-process
[`--allow-fs-read`]: #--allow-fs-read
[`--allow-fs-write`]: #--allow-fs-write
[`--allow-wasi`]: #--allow-wasi
[`--allow-worker`]: #--allow-worker
[`--build-snapshot`]: #--build-snapshot
[`--cpu-prof-dir`]: #--cpu-prof-dir
[`--diagnostic-dir`]: #--diagnostic-dirdirectory
[`--disable-sigusr1`]: #--disable-sigusr1
[`--env-file-if-exists`]: #--env-file-if-existsfile
[`--env-file`]: #--env-filefile
[`--experimental-addon-modules`]: #--experimental-addon-modules
[`--experimental-sea-config`]: single-executable-applications.md#generating-single-executable-preparation-blobs
[`--heap-prof-dir`]: #--heap-prof-dir
[`--import`]: #--importmodule
[`--no-experimental-strip-types`]: #--no-experimental-strip-types
[`--openssl-config`]: #--openssl-configfile
[`--preserve-symlinks`]: #--preserve-symlinks
[`--print`]: #-p---print-script
[`--redirect-warnings`]: #--redirect-warningsfile
[`--require`]: #-r---require-module
[`--use-env-proxy`]: #--use-env-proxy
[`--use-system-ca`]: #--use-system-ca
[`AsyncLocalStorage`]: async_context.md#class-asynclocalstorage
[`Buffer`]: buffer.md#class-buffer
[`CRYPTO_secure_malloc_init`]: https://www.openssl.org/docs/man3.0/man3/CRYPTO_secure_malloc_init.html
[`ERR_INVALID_TYPESCRIPT_SYNTAX`]: errors.md#err_invalid_typescript_syntax
[`ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`]: errors.md#err_unsupported_typescript_syntax
[`NODE_OPTIONS`]: #node_optionsoptions
[`NODE_USE_ENV_PROXY=1`]: #node_use_env_proxy1
[`NO_COLOR`]: https://no-color.org
[`SlowBuffer`]: buffer.md#class-slowbuffer
[`Web Storage`]: https://developer.mozilla.org/en-US/docs/Web/API/Web_Storage_API
[`YoungGenerationSizeFromSemiSpaceSize`]: https://chromium.googlesource.com/v8/v8.git/+/refs/tags/10.3.129/src/heap/heap.cc#328
[`dns.lookup()`]: dns.md#dnslookuphostname-options-callback
[`dns.setDefaultResultOrder()`]: dns.md#dnssetdefaultresultorderorder
[`dnsPromises.lookup()`]: dns.md#dnspromiseslookuphostname-options
[`import.meta.url`]: esm.md#importmetaurl
[`import` specifier]: esm.md#import-specifiers
[`net.getDefaultAutoSelectFamilyAttemptTimeout()`]: net.md#netgetdefaultautoselectfamilyattempttimeout
[`node:sqlite`]: sqlite.md
[`process.setUncaughtExceptionCaptureCallback()`]: process.md#processsetuncaughtexceptioncapturecallbackfn
[`tls.DEFAULT_MAX_VERSION`]: tls.md#tlsdefault_max_version
[`tls.DEFAULT_MIN_VERSION`]: tls.md#tlsdefault_min_version
[`unhandledRejection`]: process.md#event-unhandledrejection
[`v8.startupSnapshot` API]: v8.md#startup-snapshot-api
[collecting code coverage from tests]: test.md#collecting-code-coverage
[conditional exports]: packages.md#conditional-exports
[context-aware]: addons.md#context-aware-addons
[debugger]: debugger.md
[debugging security implications]: https://nodejs.org/en/docs/guides/debugging-getting-started/#security-implications
[deprecation warnings]: deprecations.md#list-of-deprecated-apis
[emit_warning]: process.md#processemitwarningwarning-options
[environment_variables]: #environment-variables
[filtering tests by name]: test.md#filtering-tests-by-name
[global setup and teardown]: test.md#global-setup-and-teardown
[jitless]: https://v8.dev/blog/jitless
[libuv threadpool documentation]: https://docs.libuv.org/en/latest/threadpool.html
[module compile cache]: module.md#module-compile-cache
[remote code execution]: https://www.owasp.org/index.php/Code_Injection
[running tests from the command line]: test.md#running-tests-from-the-command-line
[scavenge garbage collector]: https://v8.dev/blog/orinoco-parallel-scavenger
[security warning]: #warning-binding-inspector-to-a-public-ipport-combination-is-insecure
[semi-space]: https://www.memorymanagement.org/glossary/s.html#semi.space
[single executable application]: single-executable-applications.md
[snapshot testing]: test.md#snapshot-testing
[syntax detection]: packages.md#syntax-detection
[test reporters]: test.md#test-reporters
[test reruns]: test.md#rerunning-failed-tests
[test runner execution model]: test.md#test-runner-execution-model
[timezone IDs]: https://en.wikipedia.org/wiki/List_of_tz_database_time_zones
[tracking issue for user-land snapshots]: https://github.com/nodejs/node/issues/44014
[ways that `TZ` is handled in other environments]: https://www.gnu.org/software/libc/manual/html_node/TZ-Variable.html