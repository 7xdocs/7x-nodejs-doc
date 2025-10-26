# WebAssembly System Interface (WASI)

<!--introduced_in=v12.16.0-->

> Stability: 1 - Experimental

<strong class="critical">`node:wasi` 模块目前未提供某些 WASI 运行时提供的全面文件系统安全特性。对安全文件系统沙箱的完整支持可能在未来实现，也可能不会实现。在此期间，请不要依赖它来运行不受信任的代码。</strong>

<!-- source_link=lib/wasi.js -->

WASI API 提供了 [WebAssembly System Interface][] 规范的实现。WASI 通过一系列类 POSIX 的函数，使 WebAssembly 应用程序能够访问底层操作系统。

```mjs
import { readFile } from 'node:fs/promises';
import { WASI } from 'node:wasi';
import { argv, env } from 'node:process';

const wasi = new WASI({
  version: 'preview1',
  args: argv,
  env,
  preopens: {
    '/local': '/some/real/path/that/wasm/can/access',
  },
});

const wasm = await WebAssembly.compile(
  await readFile(new URL('./demo.wasm', import.meta.url)),
);
const instance = await WebAssembly.instantiate(wasm, wasi.getImportObject());

wasi.start(instance);
```

```cjs
'use strict';
const { readFile } = require('node:fs/promises');
const { WASI } = require('node:wasi');
const { argv, env } = require('node:process');
const { join } = require('node:path');

const wasi = new WASI({
  version: 'preview1',
  args: argv,
  env,
  preopens: {
    '/local': '/some/real/path/that/wasm/can/access',
  },
});

(async () => {
  const wasm = await WebAssembly.compile(
    await readFile(join(__dirname, 'demo.wasm')),
  );
  const instance = await WebAssembly.instantiate(wasm, wasi.getImportObject());

  wasi.start(instance);
})();
```

要运行上面的示例，创建一个新的名为 `demo.wat` 的 WebAssembly 文本格式文件：

```text
(module
    ;; Import the required fd_write WASI function which will write the given io vectors to stdout
    ;; The function signature for fd_write is:
    ;; (File Descriptor, *iovs, iovs_len, nwritten) -> Returns number of bytes written
    (import "wasi_snapshot_preview1" "fd_write" (func $fd_write (param i32 i32 i32 i32) (result i32)))

    (memory 1)
    (export "memory" (memory 0))

    ;; Write 'hello world\n' to memory at an offset of 8 bytes
    ;; Note the trailing newline which is required for the text to appear
    (data (i32.const 8) "hello world\n")

    (func $main (export "_start")
        ;; Creating a new io vector within linear memory
        (i32.store (i32.const 0) (i32.const 8))  ;; iov.iov_base - This is a pointer to the start of the 'hello world\n' string
        (i32.store (i32.const 4) (i32.const 12))  ;; iov.iov_len - The length of the 'hello world\n' string

        (call $fd_write
            (i32.const 1) ;; file_descriptor - 1 for stdout
            (i32.const 0) ;; *iovs - The pointer to the iov array, which is stored at memory location 0
            (i32.const 1) ;; iovs_len - We're printing 1 string stored in an iov - so one.
            (i32.const 20) ;; nwritten - A place in memory to store the number of bytes written
        )
        drop ;; Discard the number of bytes written from the top of the stack
    )
)
```

使用 [wabt](https://github.com/WebAssembly/wabt) 将 `.wat` 编译为 `.wasm`

```bash
wat2wasm demo.wat
```

## 安全性

<!-- YAML
added:
  - v21.2.0
  - v20.11.0
changes:
  - version:
    - v21.2.0
    - v20.11.0
    pr-url: https://github.com/nodejs/node/pull/50396
    description: Clarify WASI security properties.
-->

WASI 提供了一个基于能力的模型，通过该模型，应用程序被提供其自定义的 `env`、`preopens`、`stdin`、`stdout`、`stderr` 和 `exit` 能力。

**当前的 Node.js 威胁模型并未提供某些 WASI 运行时中存在的安全沙箱功能。**

虽然支持能力特性，但它们并未在 Node.js 中构成安全模型。例如，可以使用各种技术绕过文件系统沙箱。该项目正在探索是否可以在未来添加这些安全保证。

## 类：`WASI`

<!-- YAML
added:
 - v13.3.0
 - v12.16.0
-->

`WASI` 类提供了 WASI 系统调用 API 以及用于处理基于 WASI 的应用程序的额外便捷方法。每个 `WASI` 实例代表一个独立的环境。

### `new WASI([options])`

<!-- YAML
added:
 - v13.3.0
 - v12.16.0
changes:
 - version: v20.1.0
   pr-url: https://github.com/nodejs/node/pull/47390
   description: default value of returnOnExit changed to true.
 - version: v20.0.0
   pr-url: https://github.com/nodejs/node/pull/47391
   description: The version option is now required and has no default value.
 - version: v19.8.0
   pr-url: https://github.com/nodejs/node/pull/46469
   description: version field added to options.
-->

* `options` {Object}
  * `args` {Array} 一个字符串数组，WebAssembly 应用程序会将其视为命令行参数。第一个参数是 WASI 命令本身的虚拟路径。**默认值:** `[]`。
  * `env` {Object} 一个类似于 `process.env` 的对象，WebAssembly 应用程序会将其视为其环境。**默认值:** `{}`。
  * `preopens` {Object} 此对象表示 WebAssembly 应用程序的本地目录结构。`preopens` 的字符串键被视为文件系统中的目录。`preopens` 中对应的值是主机机器上这些目录的真实路径。
  * `returnOnExit` {boolean} 默认情况下，当 WASI 应用程序调用 `__wasi_proc_exit()` 时，`wasi.start()` 将返回指定的退出码，而不是终止进程。将此选项设置为 `false` 将导致 Node.js 进程以指定的退出码退出。**默认值:** `true`。
  * `stdin` {integer} 在 WebAssembly 应用程序中用作标准输入的文件描述符。**默认值:** `0`。
  * `stdout` {integer} 在 WebAssembly 应用程序中用作标准输出的文件描述符。**默认值:** `1`。
  * `stderr` {integer} 在 WebAssembly 应用程序中用作标准错误的文件描述符。**默认值:** `2`。
  * `version` {string} 请求的 WASI 版本。当前唯一支持的版本是 `unstable` 和 `preview1`。此选项是强制性的。

### `wasi.getImportObject()`

<!-- YAML
added: v19.8.0
-->

返回一个导入对象，如果除了 WASI 提供的导入之外不需要其他 WASM 导入，可以将其传递给 `WebAssembly.instantiate()`。

如果传入构造函数的版本是 `unstable`，它将返回：

```json
{ wasi_unstable: wasi.wasiImport }
```

如果传入构造函数的版本是 `preview1` 或者未指定版本，它将返回：

```json
{ wasi_snapshot_preview1: wasi.wasiImport }
```

### `wasi.start(instance)`

<!-- YAML
added:
 - v13.3.0
 - v12.16.0
-->

* `instance` {WebAssembly.Instance}

尝试通过调用其 `_start()` 导出，开始将 `instance` 作为 WASI 命令执行。如果 `instance` 不包含 `_start()` 导出，或者 `instance` 包含 `_initialize()` 导出，则将抛出异常。

`start()` 要求 `instance` 导出一个名为 `memory` 的 [`WebAssembly.Memory`][]。如果 `instance` 没有 `memory` 导出，将抛出异常。

如果 `start()` 被多次调用，将抛出异常。

### `wasi.initialize(instance)`

<!-- YAML
added:
 - v14.6.0
 - v12.19.0
-->

* `instance` {WebAssembly.Instance}

如果存在，尝试通过调用其 `_initialize()` 导出，将 `instance` 初始化为 WASI reactor。如果 `instance` 包含 `_start()` 导出，则将抛出异常。

`initialize()` 要求 `instance` 导出一个名为 `memory` 的 [`WebAssembly.Memory`][]。如果 `instance` 没有 `memory` 导出，将抛出异常。

如果 `initialize()` 被多次调用，将抛出异常。

### `wasi.finalizeBindings(instance[, options])`

<!-- YAML
added: v24.4.0
-->

* `instance` {WebAssembly.Instance}
* `options` {Object}
  * `memory` {WebAssembly.Memory} **默认值:** `instance.exports.memory`。

设置 WASI 主机绑定到 `instance`，而不调用 `initialize()` 或 `start()`。当 WASI 模块在子线程中实例化以跨线程共享内存时，此方法很有用。

`finalizeBindings()` 要求 `instance` 导出一个名为 `memory` 的 [`WebAssembly.Memory`][]，或者用户在 `options.memory` 中指定一个 [`WebAssembly.Memory`][] 对象。如果 `memory` 无效，将抛出异常。

`start()` 和 `initialize()` 将在内部调用 `finalizeBindings()`。如果 `finalizeBindings()` 被多次调用，将抛出异常。

### `wasi.wasiImport`

<!-- YAML
added:
 - v13.3.0
 - v12.16.0
-->

* 类型: {Object}

`wasiImport` 是一个实现了 WASI 系统调用 API 的对象。在实例化 [`WebAssembly.Instance`][] 时，此对象应作为 `wasi_snapshot_preview1` 导入传入。

[WebAssembly System Interface]: https://wasi.dev/
[`WebAssembly.Instance`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Instance
[`WebAssembly.Memory`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/WebAssembly/Memory