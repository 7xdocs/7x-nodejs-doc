# Modules: `node:module` API

<!--introduced_in=v12.20.0-->

<!-- YAML
added: v0.3.7
-->

## `Module` 对象

* 类型: {Object}

提供与 `Module` 实例（在 [CommonJS][] 模块中常见的 [`module`][] 变量）交互时的通用工具方法。通过 `import 'node:module'` 或 `require('node:module')` 访问。

### `module.builtinModules`

<!-- YAML
added:
  - v9.3.0
  - v8.10.0
  - v6.13.0
changes:
  - version: v23.5.0
    pr-url: https://github.com/nodejs/node/pull/56185
    description: The list now also contains prefix-only modules.
-->

* 类型: {string\[]}

Node.js 提供的所有模块名称的列表。可用于验证一个模块是否由第三方维护。

此上下文中的 `module` 与 [模块包装器][] 提供的对象不同。要访问它，需要引入 `Module` 模块：

```mjs
// module.mjs
// 在 ECMAScript 模块中
import { builtinModules as builtin } from 'node:module';
```

```cjs
// module.cjs
// 在 CommonJS 模块中
const builtin = require('node:module').builtinModules;
```

### `module.createRequire(filename)`

<!-- YAML
added: v12.2.0
-->

* `filename` {string|URL} 用于构造 require 函数的文件名。必须是 file URL 对象、file URL 字符串或绝对路径字符串。
* 返回: {require} Require 函数

```mjs
import { createRequire } from 'node:module';
const require = createRequire(import.meta.url);

// sibling-module.js 是一个 CommonJS 模块。
const siblingModule = require('./sibling-module');
```

### `module.findPackageJSON(specifier[, base])`

<!-- YAML
added:
  - v23.2.0
  - v22.14.0
-->

> Stability: 1.1 - Active Development

* `specifier` {string|URL} 要检索其 `package.json` 的模块的说明符。传递 _裸说明符_ 时，返回包根目录的 `package.json`。传递 _相对说明符_ 或 _绝对说明符_ 时，返回最近的父级 `package.json`。
* `base` {string|URL} 包含模块的绝对位置（`file:` URL 字符串或文件系统路径）。对于 CJS，使用 `__filename`（而不是 `__dirname`！）；对于 ESM，使用 `import.meta.url`。如果 `specifier` 是 _绝对说明符_，则无需传递此参数。
* 返回: {string|undefined} 如果找到 `package.json` 则返回路径。当 `specifier` 是一个包时，返回包的根目录 `package.json`；当是相对或未解析的说明符时，返回最接近 `specifier` 的 `package.json`。

> **注意**：不要使用此功能来尝试确定模块格式。影响该确定的因素很多；package.json 的 `type` 字段是 _最不_ 具有决定性的（例如，文件扩展名会覆盖它，而加载器钩子又会覆盖文件扩展名）。

> **注意**：目前这仅利用了内置的默认解析器；如果注册了 [`resolve` 自定义钩子][resolve hook]，它们将不会影响解析。这将来可能会改变。

```text
/path/to/project
  ├ packages/
    ├ bar/
      ├ bar.js
      └ package.json // name = '@foo/bar'
    └ qux/
      ├ node_modules/
        └ some-package/
          └ package.json // name = 'some-package'
      ├ qux.js
      └ package.json // name = '@foo/qux'
  ├ main.js
  └ package.json // name = '@foo'
```

```mjs
// /path/to/project/packages/bar/bar.js
import { findPackageJSON } from 'node:module';

findPackageJSON('..', import.meta.url);
// '/path/to/project/package.json'
// 传递绝对说明符时结果相同：
findPackageJSON(new URL('../', import.meta.url));
findPackageJSON(import.meta.resolve('../'));

findPackageJSON('some-package', import.meta.url);
// '/path/to/project/packages/bar/node_modules/some-package/package.json'
// 当传递绝对说明符时，如果解析的模块位于具有嵌套 `package.json` 的子文件夹内，可能会得到不同的结果。
findPackageJSON(import.meta.resolve('some-package'));
// '/path/to/project/packages/bar/node_modules/some-package/some-subfolder/package.json'

findPackageJSON('@foo/qux', import.meta.url);
// '/path/to/project/packages/qux/package.json'
```

```cjs
// /path/to/project/packages/bar/bar.js
const { findPackageJSON } = require('node:module');
const { pathToFileURL } = require('node:url');
const path = require('node:path');

findPackageJSON('..', __filename);
// '/path/to/project/package.json'
// 传递绝对说明符时结果相同：
findPackageJSON(pathToFileURL(path.join(__dirname, '..')));

findPackageJSON('some-package', __filename);
// '/path/to/project/packages/bar/node_modules/some-package/package.json'
// 当传递绝对说明符时，如果解析的模块位于具有嵌套 `package.json` 的子文件夹内，可能会得到不同的结果。
findPackageJSON(pathToFileURL(require.resolve('some-package')));
// '/path/to/project/packages/bar/node_modules/some-package/some-subfolder/package.json'

findPackageJSON('@foo/qux', __filename);
// '/path/to/project/packages/qux/package.json'
```

### `module.isBuiltin(moduleName)`

<!-- YAML
added:
  - v18.6.0
  - v16.17.0
-->

* `moduleName` {string} 模块名称
* 返回: {boolean} 如果模块是内置模块则返回 true，否则返回 false

```mjs
import { isBuiltin } from 'node:module';
isBuiltin('node:fs'); // true
isBuiltin('fs'); // true
isBuiltin('wss'); // false
```

### `module.register(specifier[, parentURL][, options])`

<!-- YAML
added:
  - v20.6.0
  - v18.19.0
changes:
  - version:
    - v23.6.1
    - v22.13.1
    - v20.18.2
    pr-url: https://github.com/nodejs-private/node-private/pull/629
    description: Using this feature with the permission model enabled requires
                 passing `--allow-worker`.
  - version:
    - v20.8.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/49655
    description: Add support for WHATWG URL instances.
-->

> Stability: 1.2 - Release candidate

* `specifier` {string|URL} 要注册的自定义钩子；这应该是与传递给 `import()` 相同的字符串，但如果它是相对的，则相对于 `parentURL` 进行解析。
* `parentURL` {string|URL} 如果你想相对于一个基础 URL（例如 `import.meta.url`）解析 `specifier`，可以在此处传递该 URL。**默认值:** `'data:'`
* `options` {Object}
  * `parentURL` {string|URL} 如果你想相对于一个基础 URL（例如 `import.meta.url`）解析 `specifier`，可以在此处传递该 URL。如果 `parentURL` 作为第二个参数提供，则忽略此属性。**默认值:** `'data:'`
  * `data` {any} 传递给 [`initialize`][] 钩子的任意可克隆的 JavaScript 值。
  * `transferList` {Object\[]} 传递给 `initialize` 钩子的 [可传输对象][]。

注册一个导出 [钩子][] 的模块，这些钩子用于自定义 Node.js 模块的解析和加载行为。参见 [自定义钩子][]。

如果与 [权限模型][] 一起使用，此功能需要 `--allow-worker`。

### `module.registerHooks(options)`

<!-- YAML
added:
  - v23.5.0
  - v22.15.0
-->

> Stability: 1.1 - Active development

* `options` {Object}
  * `load` {Function|undefined} 参见 [load 钩子][]。**默认值:** `undefined`。
  * `resolve` {Function|undefined} 参见 [resolve 钩子][]。**默认值:** `undefined`。

注册 [钩子][]，这些钩子用于自定义 Node.js 模块的解析和加载行为。参见 [自定义钩子][]。

### `module.stripTypeScriptTypes(code[, options])`

<!-- YAML
added:
  - v23.2.0
  - v22.13.0
-->

> Stability: 1.2 - Release candidate

* `code` {string} 要从中剥离类型注解的代码。
* `options` {Object}
  * `mode` {string} **默认值:** `'strip'`。可能的值有：
    * `'strip'` 仅剥离类型注解，不执行 TypeScript 特性的转换。
    * `'transform'` 剥离类型注解并将 TypeScript 特性转换为 JavaScript。
  * `sourceMap` {boolean} **默认值:** `false`。仅当 `mode` 为 `'transform'` 时，如果为 `true`，将为转换后的代码生成 source map。
  * `sourceUrl` {string} 指定在 source map 中使用的源 URL。
* 返回: {string} 剥离了类型注解的代码。
  `module.stripTypeScriptTypes()` 从 TypeScript 代码中移除类型注解。它可以在使用 `vm.runInContext()` 或 `vm.compileFunction()` 运行 TypeScript 代码之前，用于剥离类型注解。
  默认情况下，如果代码包含需要转换的 TypeScript 特性（如 `Enums`），它将抛出一个错误，更多信息请参见 [类型剥离][]。
  当模式为 `'transform'` 时，它还会将 TypeScript 特性转换为 JavaScript，更多信息请参见 [转换 TypeScript 特性][]。
  当模式为 `'strip'` 时，不会生成 source map，因为位置信息被保留了。
  如果提供了 `sourceMap` 而模式为 `'strip'`，将会抛出错误。

_警告_：由于 TypeScript 解析器的变化，此函数的输出在不同 Node.js 版本间不应被视为稳定。

```mjs
import { stripTypeScriptTypes } from 'node:module';
const code = 'const a: number = 1;';
const strippedCode = stripTypeScriptTypes(code);
console.log(strippedCode);
// 打印: const a         = 1;
```

```cjs
const { stripTypeScriptTypes } = require('node:module');
const code = 'const a: number = 1;';
const strippedCode = stripTypeScriptTypes(code);
console.log(strippedCode);
// 打印: const a         = 1;
```

如果提供了 `sourceUrl`，它将被附加在输出末尾作为注释：

```mjs
import { stripTypeScriptTypes } from 'node:module';
const code = 'const a: number = 1;';
const strippedCode = stripTypeScriptTypes(code, { mode: 'strip', sourceUrl: 'source.ts' });
console.log(strippedCode);
// 打印: const a         = 1\n\n//# sourceURL=source.ts;
```

```cjs
const { stripTypeScriptTypes } = require('node:module');
const code = 'const a: number = 1;';
const strippedCode = stripTypeScriptTypes(code, { mode: 'strip', sourceUrl: 'source.ts' });
console.log(strippedCode);
// 打印: const a         = 1\n\n//# sourceURL=source.ts;
```

当 `mode` 为 `'transform'` 时，代码被转换为 JavaScript：

```mjs
import { stripTypeScriptTypes } from 'node:module';
const code = `
  namespace MathUtil {
    export const add = (a: number, b: number) => a + b;
  }`;
const strippedCode = stripTypeScriptTypes(code, { mode: 'transform', sourceMap: true });
console.log(strippedCode);
// 打印:
// var MathUtil;
// (function(MathUtil) {
//     MathUtil.add = (a, b)=>a + b;
// })(MathUtil || (MathUtil = {}));
// # sourceMappingURL=data:application/json;base64, ...
```

```cjs
const { stripTypeScriptTypes } = require('node:module');
const code = `
  namespace MathUtil {
    export const add = (a: number, b: number) => a + b;
  }`;
const strippedCode = stripTypeScriptTypes(code, { mode: 'transform', sourceMap: true });
console.log(strippedCode);
// 打印:
// var MathUtil;
// (function(MathUtil) {
//     MathUtil.add = (a, b)=>a + b;
// })(MathUtil || (MathUtil = {}));
// # sourceMappingURL=data:application/json;base64, ...
```

### `module.syncBuiltinESMExports()`

<!-- YAML
added: v12.12.0
-->

`module.syncBuiltinESMExports()` 方法更新所有内置 [ES 模块][] 的实时绑定，以匹配 [CommonJS][] 导出的属性。它不会从 [ES 模块][] 中添加或移除导出的名称。

```js
const fs = require('node:fs');
const assert = require('node:assert');
const { syncBuiltinESMExports } = require('node:module');

fs.readFile = newAPI;

delete fs.readFileSync;

function newAPI() {
  // ...
}

fs.newAPI = newAPI;

syncBuiltinESMExports();

import('node:fs').then((esmFS) => {
  // 它将现有的 readFile 属性与新值同步
  assert.strictEqual(esmFS.readFile, newAPI);
  // readFileSync 已从所需的 fs 中删除
  assert.strictEqual('readFileSync' in fs, false);
  // syncBuiltinESMExports() 不会从 esmFS 中移除 readFileSync
  assert.strictEqual('readFileSync' in esmFS, true);
  // syncBuiltinESMExports() 不会添加名称
  assert.strictEqual(esmFS.newAPI, undefined);
});
```

## 模块编译缓存

<!-- YAML
added: v22.1.0
changes:
  - version: v22.8.0
    pr-url: https://github.com/nodejs/node/pull/54501
    description: add initial JavaScript APIs for runtime access.
-->

模块编译缓存可以通过使用 [`module.enableCompileCache()`][] 方法或 [`NODE_COMPILE_CACHE=dir`][] 环境变量来启用。启用后，每当 Node.js 编译一个 CommonJS 或 ECMAScript 模块时，它将使用指定目录中持久化的磁盘 [V8 代码缓存][] 来加速编译。
这可能会减慢模块图的首次加载速度，但如果模块的内容没有更改，后续加载同一模块图可能会获得显著的速度提升。

要清理磁盘上生成的编译缓存，只需删除缓存目录。下次使用同一目录存储编译缓存时，将重新创建缓存目录。为了避免磁盘被陈旧的缓存填满，建议使用 [`os.tmpdir()`][] 下的目录。如果通过调用 [`module.enableCompileCache()`][] 启用编译缓存而未指定目录，Node.js 将使用 [`NODE_COMPILE_CACHE=dir`][] 环境变量（如果已设置），否则默认使用 `path.join(os.tmpdir(), 'node-compile-cache')`。要定位正在运行的 Node.js 实例使用的编译缓存目录，请使用 [`module.getCompileCacheDir()`][]。

目前，当使用编译缓存与 [V8 JavaScript 代码覆盖率][] 时，V8 收集的覆盖率对于从代码缓存反序列化的函数可能不太精确。建议在运行测试以生成精确覆盖率时关闭此功能。

可以通过 [`NODE_DISABLE_COMPILE_CACHE=1`][] 环境变量禁用已启用的模块编译缓存。当编译缓存导致意外或不希望的行为（例如，测试覆盖率不精确）时，这可能很有用。

由一个版本的 Node.js 生成的编译缓存不能被不同版本的 Node.js 重用。如果使用相同的基础目录来持久化缓存，不同版本的 Node.js 生成的缓存将分开存储，因此它们可以共存。

目前，当启用编译缓存并且重新加载一个模块时，代码缓存会立即从编译后的代码生成，但仅在 Node.js 实例即将退出时才会写入磁盘。这一点可能会改变。[`module.flushCompileCache()`][] 方法可用于确保累积的代码缓存被刷新到磁盘，以防应用程序想要生成其他 Node.js 实例并让它们早在父进程退出之前就共享缓存。

### `module.constants.compileCacheStatus`

<!-- YAML
added: v22.8.0
-->

> Stability: 1.1 - Active Development

以下常量作为 [`module.enableCompileCache()`][] 返回的对象中的 `status` 字段返回，以指示启用 [模块编译缓存][] 尝试的结果。

<table>
  <tr>
    <th>常量</th>
    <th>描述</th>
  </tr>
  <tr>
    <td><code>ENABLED</code></td>
    <td>
      Node.js 已成功启用编译缓存。用于存储编译缓存的目录将在返回对象的 <code>directory</code> 字段中返回。
    </td>
  </tr>
  <tr>
    <td><code>ALREADY_ENABLED</code></td>
    <td>
      编译缓存之前已经启用，无论是通过先前调用 <code>module.enableCompileCache()</code>，还是通过 <code>NODE_COMPILE_CACHE=dir</code> 环境变量。用于存储编译缓存的目录将在返回对象的 <code>directory</code> 字段中返回。
    </td>
  </tr>
  <tr>
    <td><code>FAILED</code></td>
    <td>
      Node.js 未能启用编译缓存。这可能是由于缺少使用指定目录的权限，或各种文件系统错误。失败的详细信息将在返回对象的 <code>message</code> 字段中返回。
    </td>
  </tr>
  <tr>
    <td><code>DISABLED</code></td>
    <td>
      Node.js 无法启用编译缓存，因为环境变量 <code>NODE_DISABLE_COMPILE_CACHE=1</code> 已被设置。
    </td>
  </tr>
</table>

### `module.enableCompileCache([cacheDir])`

<!-- YAML
added: v22.8.0
-->

> Stability: 1.1 - Active Development

* `cacheDir` {string|undefined} 可选路径，指定存储/检索编译缓存的目录。
* 返回: {Object}
  * `status` {integer} [`module.constants.compileCacheStatus`][] 之一
  * `message` {string|undefined} 如果 Node.js 无法启用编译缓存，此字段包含错误消息。仅当 `status` 为 `module.constants.compileCacheStatus.FAILED` 时设置。
  * `directory` {string|undefined} 如果编译缓存已启用，此字段包含存储编译缓存的目录。仅当 `status` 为 `module.constants.compileCacheStatus.ENABLED` 或 `module.constants.compileCacheStatus.ALREADY_ENABLED` 时设置。

在当前 Node.js 实例中启用 [模块编译缓存][]。

如果未指定 `cacheDir`，Node.js 将使用 [`NODE_COMPILE_CACHE=dir`][] 环境变量指定的目录（如果已设置），否则使用 `path.join(os.tmpdir(), 'node-compile-cache')`。对于一般用例，建议调用 `module.enableCompileCache()` 而不指定 `cacheDir`，以便在必要时可以通过 `NODE_COMPILE_CACHE` 环境变量覆盖目录。

由于编译缓存应该是一种静默优化，不是应用程序功能所必需的，因此此方法设计为在无法启用编译缓存时不抛出任何异常。相反，它将返回一个包含 `message` 字段中错误信息的对象以辅助调试。
如果编译缓存成功启用，返回对象中的 `directory` 字段包含存储编译缓存的目录路径。返回对象中的 `status` 字段将是 `module.constants.compileCacheStatus` 值之一，以指示启用 [模块编译缓存][] 尝试的结果。

此方法仅影响当前 Node.js 实例。要在子工作线程中启用它，要么在子工作线程中也调用此方法，要么将 `process.env.NODE_COMPILE_CACHE` 值设置为编译缓存目录，以便行为可以继承到子工作线程中。目录可以从该方法返回的 `directory` 字段获取，或者使用 [`module.getCompileCacheDir()`][]。

### `module.flushCompileCache()`

<!-- YAML
added:
 - v23.0.0
 - v22.10.0
-->

> Stability: 1.1 - Active Development

将当前 Node.js 实例中已加载模块累积的 [模块编译缓存][] 刷新到磁盘。此方法在所有刷新文件系统操作结束（无论成功与否）后返回。如果有任何错误，此操作将静默失败，因为编译缓存未命中不应干扰应用程序的实际操作。

### `module.getCompileCacheDir()`

<!-- YAML
added: v22.8.0
-->

> Stability: 1.1 - Active Development

* 返回: {string|undefined} 如果启用了 [模块编译缓存][]，则返回其目录路径，否则返回 `undefined`。

<i id="module_customization_hooks"></i>

## 自定义钩子

<!-- YAML
added: v8.8.0
changes:
  - version:
    - v23.5.0
    - v22.15.0
    pr-url: https://github.com/nodejs/node/pull/55698
    description: Add support for synchronous and in-thread hooks.
  - version:
    - v20.6.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/48842
    description: Added `initialize` hook to replace `globalPreload`.
  - version:
    - v18.6.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42623
    description: Add support for chaining loaders.
  - version: v16.12.0
    pr-url: https://github.com/nodejs/node/pull/37468
    description: Removed `getFormat`, `getSource`, `transformSource`, and
                 `globalPreload`; added `load` hook and `getGlobalPreload` hook.
-->

<!-- type=misc -->

> Stability: 1.2 - Release candidate (asynchronous version)
> Stability: 1.1 - Active development (synchronous version)

目前支持两种类型的模块自定义钩子：

1. `module.register(specifier[, parentURL][, options])`，它接受一个导出异步钩子函数的模块。这些函数在单独的加载器线程上运行。
2. `module.registerHooks(options)`，它接受同步钩子函数，这些函数直接在加载模块的线程上运行。

<i id="enabling_module_customization_hooks"></i>

### 启用

模块解析和加载可以通过以下方式自定义：

1. 使用 `node:module` 的 [`register`][] 方法注册一个导出一组异步钩子函数的文件，
2. 使用 `node:module` 的 [`registerHooks`][] 方法注册一组同步钩子函数。

可以使用 [`--import`][] 或 [`--require`][] 标志在应用程序代码运行之前注册钩子：

```bash
node --import ./register-hooks.js ./my-app.js
node --require ./register-hooks.js ./my-app.js
```

```mjs
// register-hooks.js
// 如果此文件不包含顶级 await，则只能被 require()。
// 使用 module.register() 在专用线程中注册异步钩子。
import { register } from 'node:module';
register('./hooks.mjs', import.meta.url);
```

```cjs
// register-hooks.js
const { register } = require('node:module');
const { pathToFileURL } = require('node:url');
// 使用 module.register() 在专用线程中注册异步钩子。
register('./hooks.mjs', pathToFileURL(__filename));
```

```mjs
// 使用 module.registerHooks() 在主线程中注册同步钩子。
import { registerHooks } from 'node:module';
registerHooks({
  resolve(specifier, context, nextResolve) { /* implementation */ },
  load(url, context, nextLoad) { /* implementation */ },
});
```

```cjs
// 使用 module.registerHooks() 在主线程中注册同步钩子。
const { registerHooks } = require('node:module');
registerHooks({
  resolve(specifier, context, nextResolve) { /* implementation */ },
  load(url, context, nextLoad) { /* implementation */ },
});
```

传递给 `--import` 或 `--require` 的文件也可以是依赖项的导出：

```bash
node --import some-package/register ./my-app.js
node --require some-package/register ./my-app.js
```

其中 `some-package` 有一个 [`"exports"`][] 字段定义了 `/register` 导出，映射到一个调用 `register()` 的文件，如下面的 `register-hooks.js` 示例。

使用 `--import` 或 `--require` 可确保在任何应用程序文件（包括应用程序的入口点以及默认情况下的任何工作线程）被导入之前注册钩子。

或者，可以从入口点调用 `register()` 和 `registerHooks()`，但对于任何应在钩子注册后运行的 ESM 代码，必须使用动态 `import()`。

```mjs
import { register } from 'node:module';

register('http-to-https', import.meta.url);

// 因为这是一个动态 `import()`，`http-to-https` 钩子将运行来处理 `./my-app.js` 及其导入或需要的任何其他文件。
await import('./my-app.js');
```

```cjs
const { register } = require('node:module');
const { pathToFileURL } = require('node:url');

register('http-to-https', pathToFileURL(__filename));

// 因为这是一个动态 `import()`，`http-to-https` 钩子将运行来处理 `./my-app.js` 及其导入或需要的任何其他文件。
import('./my-app.js');
```

自定义钩子将对在注册之后加载的任何模块以及它们通过 `import` 和内置 `require` 引用的模块运行。
用户使用 `module.createRequire()` 创建的 `require` 函数只能由同步钩子自定义。

在此示例中，我们注册了 `http-to-https` 钩子，但它们仅对随后导入的模块可用——在本例中为 `my-app.js` 及其通过 `import` 或在 CommonJS 依赖项中通过内置 `require` 引用的任何内容。

如果 `import('./my-app.js')` 是静态的 `import './my-app.js'`，那么应用程序将在 `http-to-https` 钩子注册之前 _已经_ 被加载。这是由于 ES 模块规范，其中静态导入首先从树的叶子开始评估，然后回到主干。`my-app.js` 内部 _可能有_ 静态导入，这些导入在 `my-app.js` 被动态导入之前不会被评估。

如果使用同步钩子，则 `import`、`require` 和使用 `createRequire()` 创建的用户 `require` 都受支持。

```mjs
import { registerHooks, createRequire } from 'node:module';

registerHooks({ /* 同步钩子的实现 */ });

const require = createRequire(import.meta.url);

// 同步钩子影响 import、require() 以及通过 createRequire() 创建的用户 require() 函数。
await import('./my-app.js');
require('./my-app-2.js');
```

```cjs
const { register, registerHooks } = require('node:module');
const { pathToFileURL } = require('node:url');

registerHooks({ /* 同步钩子的实现 */ });

const userRequire = createRequire(__filename);

// 同步钩子影响 import、require() 以及通过 createRequire() 创建的用户 require() 函数。
import('./my-app.js');
require('./my-app-2.js');
userRequire('./my-app-3.js');
```

最后，如果您只想在应用程序运行之前注册钩子，并且不想为此目的创建单独的文件，可以将 `data:` URL 传递给 `--import`：

```bash
node --import 'data:text/javascript,import { register } from "node:module"; import { pathToFileURL } from "node:url"; register("http-to-https", pathToFileURL("./"));' ./my-app.js
```

### 链式调用

可以多次调用 `register`：

```mjs
// entrypoint.mjs
import { register } from 'node:module';

register('./foo.mjs', import.meta.url);
register('./bar.mjs', import.meta.url);
await import('./my-app.mjs');
```

```cjs
// entrypoint.cjs
const { register } = require('node:module');
const { pathToFileURL } = require('node:url');

const parentURL = pathToFileURL(__filename);
register('./foo.mjs', parentURL);
register('./bar.mjs', parentURL);
import('./my-app.mjs');
```

在此示例中，注册的钩子将形成链。这些链以后进先出（LIFO）的方式运行。如果 `foo.mjs` 和 `bar.mjs` 都定义了一个 `resolve` 钩子，它们将被这样调用（注意从右到左）：
Node.js 默认 ← `./foo.mjs` ← `./bar.mjs`
（从 `./bar.mjs` 开始，然后是 `./foo.mjs`，然后是 Node.js 默认值）。
所有其他钩子也是如此。

注册的钩子也会影响 `register` 本身。在此示例中，`bar.mjs` 将通过 `foo.mjs` 注册的钩子进行解析和加载（因为 `foo` 的钩子将已经被添加到链中）。这允许实现诸如用非 JavaScript 语言编写钩子等功能，只要先前注册的钩子能够将其转译为 JavaScript。

不能在定义钩子的模块内部调用 `register` 方法。

`registerHooks` 的链式调用工作方式类似。如果混合使用同步和异步钩子，同步钩子总是在异步钩子开始运行之前首先运行，也就是说，在最后一个运行的同步钩子中，其下一个钩子包括异步钩子的调用。

```mjs
// entrypoint.mjs
import { registerHooks } from 'node:module';

const hook1 = { /* 钩子的实现 */ };
const hook2 = { /* 钩子的实现 */ };
// hook2 在 hook1 之前运行。
registerHooks(hook1);
registerHooks(hook2);
```

```cjs
// entrypoint.cjs
const { registerHooks } = require('node:module');

const hook1 = { /* 钩子的实现 */ };
const hook2 = { /* 钩子的实现 */ };
// hook2 在 hook1 之前运行。
registerHooks(hook1);
registerHooks(hook2);
```

### 与模块自定义钩子的通信

异步钩子在专用线程上运行，与运行应用程序代码的主线程分离。这意味着改变全局变量不会影响其他线程，并且必须使用消息通道在线程之间进行通信。

`register` 方法可用于将数据传递给 [`initialize`][] 钩子。传递给钩子的数据可能包括可传输对象，如端口。

```mjs
import { register } from 'node:module';
import { MessageChannel } from 'node:worker_threads';

// 此示例演示了如何使用消息通道与钩子通信，通过将 `port2` 发送给钩子。
const { port1, port2 } = new MessageChannel();

port1.on('message', (msg) => {
  console.log(msg);
});
port1.unref();

register('./my-hooks.mjs', {
  parentURL: import.meta.url,
  data: { number: 1, port: port2 },
  transferList: [port2],
});
```

```cjs
const { register } = require('node:module');
const { pathToFileURL } = require('node:url');
const { MessageChannel } = require('node:worker_threads');

// 此示例展示了如何使用消息通道与钩子通信，通过将 `port2` 发送给钩子。
const { port1, port2 } = new MessageChannel();

port1.on('message', (msg) => {
  console.log(msg);
});
port1.unref();

register('./my-hooks.mjs', {
  parentURL: pathToFileURL(__filename),
  data: { number: 1, port: port2 },
  transferList: [port2],
});
```

同步模块钩子在运行应用程序代码的同一线程上运行。它们可以直接改变主线程访问的上下文的全局变量。

### 钩子

#### `module.register()` 接受的异步钩子

[`register`][] 方法可用于注册一个导出一组钩子的模块。这些钩子是 Node.js 为自定义模块解析和加载过程而调用的函数。导出的函数必须具有特定的名称和签名，并且必须作为命名导出。

```mjs
export async function initialize({ number, port }) {
  // 从 `register` 接收数据。
}

export async function resolve(specifier, context, nextResolve) {
  // 获取 `import` 或 `require` 说明符并将其解析为 URL。
}

export async function load(url, context, nextLoad) {
  // 获取已解析的 URL 并返回要评估的源代码。
}
```

异步钩子在单独的线程中运行，与运行应用程序代码的主线程隔离。这意味着它是一个不同的 [领域][]。主线程可能随时终止钩子线程，因此不要依赖异步操作（如 `console.log`）完成。默认情况下，它们会被继承到子工作线程中。

#### `module.registerHooks()` 接受的同步钩子

<!-- YAML
added:
  - v23.5.0
  - v22.15.0
-->

> Stability: 1.1 - Active development

`module.registerHooks()` 方法接受同步钩子函数。
不支持也不需要 `initialize()`，因为钩子实现者可以在调用 `module.registerHooks()` 之前直接运行初始化代码。

```mjs
function resolve(specifier, context, nextResolve) {
  // 获取 `import` 或 `require` 说明符并将其解析为 URL。
}

function load(url, context, nextLoad) {
  // 获取已解析的 URL 并返回要评估的源代码。
}
```

同步钩子在加载模块的同一线程和同一 [领域][] 中运行。与异步钩子不同，默认情况下它们不会被继承到子工作线程中，但如果钩子是使用 [`--import`][] 或 [`--require`][] 预加载的文件注册的，子工作线程可以通过 `process.execArgv` 继承预加载的脚本。有关详细信息，请参见 [`Worker` 的文档][]。

在同步钩子中，用户可以期望 `console.log()` 以与模块代码中的 `console.log()` 相同的方式完成。

#### 钩子的约定

钩子是 [链][] 的一部分，即使该链仅包含一个自定义（用户提供）钩子和始终存在的默认钩子。钩子函数是嵌套的：每个函数必须始终返回一个普通对象，而链式调用是每个函数调用 `next<hookName>()` 的结果，这是对后续加载器钩子（按 LIFO 顺序）的引用。

返回缺少必需属性的值的钩子会触发异常。返回而不调用 `next<hookName>()` _且_ 不返回 `shortCircuit: true` 的钩子也会触发异常。这些错误是为了防止链无意中断开。从钩子返回 `shortCircuit: true` 以表示链有意在您的钩子处结束。

#### `initialize()`

<!-- YAML
added:
  - v20.6.0
  - v18.19.0
-->

> Stability: 1.2 - Release candidate

* `data` {any} 来自 `register(loader, import.meta.url, { data })` 的数据。

`initialize` 钩子仅被 [`register`][] 接受。`registerHooks()` 不支持也不需要它，因为同步钩子的初始化可以在调用 `registerHooks()` 之前直接运行。

`initialize` 钩子提供了一种定义自定义函数的方法，该函数在钩子模块初始化时在钩子线程中运行。初始化发生在通过 [`register`][] 注册钩子模块时。

此钩子可以从 [`register`][] 调用接收数据，包括端口和其他可传输对象。`initialize` 的返回值可以是 {Promise}，在这种情况下，主应用程序线程执行恢复之前将等待它。

模块自定义代码：

```mjs
// path-to-my-hooks.js

export async function initialize({ number, port }) {
  port.postMessage(`increment: ${number + 1}`);
}
```

调用者代码：

```mjs
import assert from 'node:assert';
import { register } from 'node:module';
import { MessageChannel } from 'node:worker_threads';

// 此示例展示了如何使用消息通道在主（应用程序）线程和运行在钩子线程上的钩子之间进行通信，通过将 `port2` 发送给 `initialize` 钩子。
const { port1, port2 } = new MessageChannel();

port1.on('message', (msg) => {
  assert.strictEqual(msg, 'increment: 2');
});
port1.unref();

register('./path-to-my-hooks.js', {
  parentURL: import.meta.url,
  data: { number: 1, port: port2 },
  transferList: [port2],
});
```

```cjs
const assert = require('node:assert');
const { register } = require('node:module');
const { pathToFileURL } = require('node:url');
const { MessageChannel } = require('node:worker_threads');

// 此示例展示了如何使用消息通道在主（应用程序）线程和运行在钩子线程上的钩子之间进行通信，通过将 `port2` 发送给 `initialize` 钩子。
const { port1, port2 } = new MessageChannel();

port1.on('message', (msg) => {
  assert.strictEqual(msg, 'increment: 2');
});
port1.unref();

register('./path-to-my-hooks.js', {
  parentURL: pathToFileURL(__filename),
  data: { number: 1, port: port2 },
  transferList: [port2],
});
```

#### `resolve(specifier, context, nextResolve)`

<!-- YAML
changes:
  - version:
    - v23.5.0
    - v22.15.0
    pr-url: https://github.com/nodejs/node/pull/55698
    description: Add support for synchronous and in-thread hooks.
  - version:
    - v21.0.0
    - v20.10.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/50140
    description: The property `context.importAssertions` is replaced with
                 `context.importAttributes`. Using the old name is still
                 supported and will emit an experimental warning.
  - version:
    - v18.6.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42623
    description: Add support for chaining resolve hooks. Each hook must either
      call `nextResolve()` or include a `shortCircuit` property set to `true`
      in its return.
  - version:
    - v17.1.0
    - v16.14.0
    pr-url: https://github.com/nodejs/node/pull/40250
    description: Add support for import assertions.
-->

* `specifier` {string}
* `context` {Object}
  * `conditions` {string\[]} 相关 `package.json` 的导出条件
  * `importAttributes` {Object} 一个对象，其键值对表示要导入模块的属性
  * `parentURL` {string|undefined} 导入此模块的模块，如果这是 Node.js 入口点，则为 undefined
* `nextResolve` {Function} 链中的下一个 `resolve` 钩子，或者是最后一个用户提供的 `resolve` 钩子之后的 Node.js 默认 `resolve` 钩子
  * `specifier` {string}
  * `context` {Object|undefined} 如果省略，则提供默认值。如果提供，默认值将与提供的属性合并，优先使用提供的属性。
* 返回: {Object|Promise} 异步版本接受包含以下属性的对象，或者将解析为此类对象的 `Promise`。同步版本仅接受同步返回的对象。
  * `format` {string|null|undefined} 给 `load` 钩子的提示（可能会被忽略）。它可以是模块格式（如 `'commonjs'` 或 `'module'`）或任意值，如 `'css'` 或 `'yaml'`。
  * `importAttributes` {Object|undefined} 缓存模块时要使用的导入属性（可选；如果省略，将使用输入值）
  * `shortCircuit` {undefined|boolean} 表示此钩子意图终止 `resolve` 钩子链的信号。**默认值:** `false`
  * `url` {string} 此输入解析到的绝对 URL

> **警告** 对于异步版本，尽管支持返回 promise 和 async 函数，但调用 `resolve` 仍可能阻塞主线程，从而影响性能。

`resolve` 钩子链负责告诉 Node.js 在哪里找到以及如何缓存给定的 `import` 语句或表达式，或 `require` 调用。它可以可选地返回一个格式（如 `'module'`）作为给 `load` 钩子的提示。如果指定了格式，`load` 钩子最终负责提供最终的 `format` 值（并且可以自由忽略 `resolve` 提供的提示）；如果 `resolve` 提供了 `format`，则即使仅为了将值传递给 Node.js 默认 `load` 钩子，也需要自定义 `load` 钩子。

导入类型属性是用于将已加载模块保存到内部模块缓存的缓存键的一部分。如果模块应使用与源代码中存在的属性不同的属性进行缓存，则 `resolve` 钩子负责返回一个 `importAttributes` 对象。

`context` 中的 `conditions` 属性是一个条件数组，将用于匹配此解析请求的 [包导出条件][条件导出]。它们可用于在其他地方查找条件映射，或在调用默认解析逻辑时修改列表。

当前的 [包导出条件][条件导出] 始终在传递给钩子的 `context.conditions` 数组中。为了在调用 `defaultResolve` 时保证 _默认的 Node.js 模块说明符解析行为_，传递给它的 `context.conditions` 数组 _必须_ 包含最初传递给 `resolve` 钩子的 `context.conditions` 数组的 _所有_ 元素。

<!-- TODO(joyeecheung): Math.random() is a bit too contrived. At least do a
find-and-replace mangling on the URLs. -->

```mjs
// module.register() 接受的异步版本。
export async function resolve(specifier, context, nextResolve) {
  const { parentURL = null } = context;

  if (Math.random() > 0.5) { // 某个条件。
    // 对于部分或所有说明符，执行一些自定义的解析逻辑。
    // 始终返回 {url: <string>} 形式的对象。
    return {
      shortCircuit: true,
      url: parentURL ?
        new URL(specifier, parentURL).href :
        new URL(specifier).href,
    };
  }

  if (Math.random() < 0.5) { // 另一个条件。
    // 当调用 `defaultResolve` 时，可以修改参数。在这种情况下，它添加了另一个用于匹配条件导出的值。
    return nextResolve(specifier, {
      ...context,
      conditions: [...context.conditions, 'another-condition'],
    });
  }

  // 推迟到链中的下一个钩子，如果这是最后一个用户指定的加载器，则将是 Node.js 默认解析钩子。
  return nextResolve(specifier);
}
```

```mjs
// module.registerHooks() 接受的同步版本。
function resolve(specifier, context, nextResolve) {
  // 类似于上面的异步 resolve()，因为该版本没有任何异步逻辑。
}
```

#### `load(url, context, nextLoad)`

<!-- YAML
changes:
  - version:
    - v23.5.0
    - v22.15.0
    pr-url: https://github.com/nodejs/node/pull/55698
    description: Add support for synchronous and in-thread version.
  - version: v22.6.0
    pr-url: https://github.com/nodejs/node/pull/56350
    description: Add support for `source` with format `commonjs-typescript` and `module-typescript`.
  - version: v20.6.0
    pr-url: https://github.com/nodejs/node/pull/47999
    description: Add support for `source` with format `commonjs`.
  - version:
    - v18.6.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42623
    description: Add support for chaining load hooks. Each hook must either
      call `nextLoad()` or include a `shortCircuit` property set to `true` in
      its return.
-->

* `url` {string} `resolve` 链返回的 URL
* `context` {Object}
  * `conditions` {string\[]} 相关 `package.json` 的导出条件
  * `format` {string|null|undefined} `resolve` 钩子链可选提供的格式。这可以是任何字符串值作为输入；输入值不需要符合下面描述的可接受返回值列表。
  * `importAttributes` {Object}
* `nextLoad` {Function} 链中的下一个 `load` 钩子，或者是最后一个用户提供的 `load` 钩子之后的 Node.js 默认 `load` 钩子
  * `url` {string}
  * `context` {Object|undefined} 如果省略，则提供默认值。如果提供，默认值将与提供的属性合并，优先使用提供的属性。在默认的 `nextLoad` 中，如果 `url` 指向的模块没有显式的模块类型信息，则 `context.format` 是强制性的。
    <!-- TODO(joyeecheung): make it at least optionally non-mandatory by allowing
         JS-style/TS-style module detection when the format is simply unknown -->
* 返回: {Object|Promise} 异步版本接受包含以下属性的对象，或者将解析为此类对象的 `Promise`。同步版本仅接受同步返回的对象。
  * `format` {string}
  * `shortCircuit` {undefined|boolean} 表示此钩子意图终止 `load` 钩子链的信号。**默认值:** `false`
  * `source` {string|ArrayBuffer|TypedArray} 供 Node.js 评估的源代码

`load` 钩子提供了一种自定义方法来解释、检索和解析 URL。它还负责验证导入属性。

`format` 的最终值必须是以下之一：

| `format`                | 描述                                           | `load` 返回的 `source` 可接受类型           |
| ----------------------- | ----------------------------------------------------- | -------------------------------------------------- |
| `'addon'`               | 加载 Node.js 插件                                  | {null}                                             |
| `'builtin'`             | 加载 Node.js 内置模块                         | {null}                                             |
| `'commonjs-typescript'` | 加载具有 TypeScript 语法的 Node.js CommonJS 模块 | {string\|ArrayBuffer\|TypedArray\|null\|undefined} |
| `'commonjs'`            | 加载 Node.js CommonJS 模块                        | {string\|ArrayBuffer\|TypedArray\|null\|undefined} |
| `'json'`                | 加载 JSON 文件                                      | {string\|ArrayBuffer\|TypedArray}                  |
| `'module-typescript'`   | 加载具有 TypeScript 语法的 ES 模块              | {string\|ArrayBuffer\|TypedArray}                  |
| `'module'`              | 加载 ES 模块                                     | {string\|ArrayBuffer\|TypedArray}                  |
| `'wasm'`                | 加载 WebAssembly 模块                             | {ArrayBuffer\|TypedArray}                          |

对于类型 `'builtin'`，`source` 的值被忽略，因为目前无法替换 Node.js 内置（核心）模块的值。

##### 异步 `load` 钩子的注意事项

当使用异步 `load` 钩子时，对于 `'commonjs'` 省略 `source` 与提供 `source` 有非常不同的效果：

* 当提供了 `source` 时，此模块的所有 `require` 调用将由具有注册的 `resolve` 和 `load` 钩子的 ESM 加载器处理；此模块的所有 `require.resolve` 调用将由具有注册的 `resolve` 钩子的 ESM 加载器处理；只有一部分 CommonJS API 可用（例如，没有 `require.extensions`，没有 `require.cache`，没有 `require.resolve.paths`），并且对 CommonJS 模块加载器的猴子补丁将不适用。
* 如果 `source` 是 undefined 或 `null`，它将由 CommonJS 模块加载器处理，并且 `require`/`require.resolve` 调用不会经过注册的钩子。对于空值 `source` 的此行为是临时的——将来，将不支持空值 `source`。

这些注意事项不适用于同步 `load` 钩子，在这种情况下，自定义 CommonJS 模块可用的完整 CommonJS API 集，并且 `require`/`require.resolve` 总是经过注册的钩子。

Node.js 内部的异步 `load` 实现（作为链中最后一个钩子的 `next` 的值）在 `format` 为 `'commonjs'` 时为了向后兼容性返回 `null` 作为 `source`。以下是一个钩子示例，它将选择使用非默认行为：

```mjs
import { readFile } from 'node:fs/promises';

// module.register() 接受的异步版本。对于 module.registerHooks() 接受的同步版本不需要此修复。
export async function load(url, context, nextLoad) {
  const result = await nextLoad(url, context);
  if (result.format === 'commonjs') {
    result.source ??= await readFile(new URL(result.responseURL ?? url));
  }
  return result;
}
```

这也不适用于同步 `load` 钩子，在这种情况下，返回的 `source` 包含下一个钩子加载的源代码，无论模块格式如何。

> **警告**：异步 `load` 钩子和来自 CommonJS 模块的命名空间导出不兼容。尝试同时使用它们将导致导入返回空对象。这可能在将来得到解决。这不适用于同步 `load` 钩子，在这种情况下，可以正常使用导出。

> 这些类型都对应于 ECMAScript 中定义的类。

* 特定的 {ArrayBuffer} 对象是 {SharedArrayBuffer}。
* 特定的 {TypedArray} 对象是 {Uint8Array}。

如果基于文本的格式（即 `'json'`、`'module'`）的源值不是字符串，则使用 [`util.TextDecoder`][] 将其转换为字符串。

`load` 钩子提供了一种自定义检索已解析 URL 的源代码的方法。这将允许加载器潜在地避免从磁盘读取文件。它还可以用于将无法识别的格式映射到支持的格式，例如将 `yaml` 映射到 `module`。

```mjs
// module.register() 接受的异步版本。
export async function load(url, context, nextLoad) {
  const { format } = context;

  if (Math.random() > 0.5) { // 某个条件
    /*
      对于部分或所有 URL，执行一些自定义的检索源代码的逻辑。
      始终返回 {
        format: <string>,
        source: <string|buffer>,
      } 形式的对象。
    */
    return {
      format,
      shortCircuit: true,
      source: '...',
    };
  }

  // 推迟到链中的下一个钩子。
  return nextLoad(url);
}
```

```mjs
// module.registerHooks() 接受的同步版本。
function load(url, context, nextLoad) {
  // 类似于上面的异步 load()，因为该版本没有任何异步逻辑。
}
```

在更高级的场景中，这也可以用于将不支持的源转换为支持的源（参见下面的 [示例](#示例)）。

### 示例

各种模块自定义钩子可以一起使用，以实现对 Node.js 代码加载和评估行为的广泛自定义。

#### 从 HTTPS 导入

下面的钩子注册了钩子以启用对此类说明符的基本支持。虽然这似乎是对 Node.js 核心功能的重大改进，但实际上使用这些钩子存在明显的缺点：性能比从磁盘加载文件慢得多，没有缓存，也没有安全性。

```mjs
// https-hooks.mjs
import { get } from 'node:https';

export function load(url, context, nextLoad) {
  // 对于要通过网络加载的 JavaScript，我们需要获取并返回它。
  if (url.startsWith('https://')) {
    return new Promise((resolve, reject) => {
      get(url, (res) => {
        let data = '';
        res.setEncoding('utf8');
        res.on('data', (chunk) => data += chunk);
        res.on('end', () => resolve({
          // 此示例假设所有网络提供的 JavaScript 都是 ES 模块代码。
          format: 'module',
          shortCircuit: true,
          source: data,
        }));
      }).on('error', (err) => reject(err));
    });
  }

  // 让 Node.js 处理所有其他 URL。
  return nextLoad(url);
}
```

```mjs
// main.mjs
import { VERSION } from 'https://coffeescript.org/browser-compiler-modern/coffeescript.js';

console.log(VERSION);
```

使用前面的钩子模块，运行
`node --import 'data:text/javascript,import { register } from "node:module"; import { pathToFileURL } from "node:url"; register(pathToFileURL("./https-hooks.mjs"));' ./main.mjs`
根据 `main.mjs` 中 URL 处的模块打印当前版本的 CoffeeScript。

<!-- TODO(joyeecheung): add an example on how to implement it with a fetchSync based on
workers and Atomics.wait() - or all these examples are too much to be put in the API
documentation already and should be put into a repository instead? -->

#### 转译

Node.js 无法理解的格式的源代码可以使用 [`load` 钩子][load hook] 转换为 JavaScript。

这比在运行 Node.js 之前转译源文件性能更低；转译器钩子应仅用于开发和测试目的。

##### 异步版本

```mjs
// coffeescript-hooks.mjs
import { readFile } from 'node:fs/promises';
import { findPackageJSON } from 'node:module';
import coffeescript from 'coffeescript';

const extensionsRegex = /\.(coffee|litcoffee|coffee\.md)$/;

export async function load(url, context, nextLoad) {
  if (extensionsRegex.test(url)) {
    // CoffeeScript 文件可以是 CommonJS 或 ES 模块。使用自定义格式告诉 Node.js 不要检测其模块类型。
    const { source: rawSource } = await nextLoad(url, { ...context, format: 'coffee' });
    // 此钩子将所有导入的 CoffeeScript 文件的 CoffeeScript 源代码转换为 JavaScript 源代码。
    const transformedSource = coffeescript.compile(rawSource.toString(), url);

    // 为了确定 Node.js 将如何解释转译结果，请在文件系统中向上搜索最近的父 package.json 文件并读取其 "type" 字段。
    return {
      format: await getPackageType(url),
      shortCircuit: true,
      source: transformedSource,
    };
  }

  // 让 Node.js 处理所有其他 URL。
  return nextLoad(url, context);
}

async function getPackageType(url) {
  // `url` 仅在第一次迭代时是文件路径，当传递来自 load() 钩子的已解析 url 时
  // 来自 load() 的实际文件路径将包含文件扩展名，因为规范要求如此
  // 这个简单的检查 `url` 是否包含文件扩展名的真值检查对于大多数项目都有效，但不能涵盖一些边缘情况（例如无扩展名文件或以尾随空格结尾的 url）
  const pJson = findPackageJSON(url);

  return readFile(pJson, 'utf8')
    .then(JSON.parse)
    .then((json) => json?.type)
    .catch(() => undefined);
}
```

##### 同步版本

```mjs
// coffeescript-sync-hooks.mjs
import { readFileSync } from 'node:fs';
import { registerHooks, findPackageJSON } from 'node:module';
import coffeescript from 'coffeescript';

const extensionsRegex = /\.(coffee|litcoffee|coffee\.md)$/;

function load(url, context, nextLoad) {
  if (extensionsRegex.test(url)) {
    const { source: rawSource } = nextLoad(url, { ...context, format: 'coffee' });
    const transformedSource = coffeescript.compile(rawSource.toString(), url);

    return {
      format: getPackageType(url),
      shortCircuit: true,
      source: transformedSource,
    };
  }

  return nextLoad(url, context);
}

function getPackageType(url) {
  const pJson = findPackageJSON(url);
  if (!pJson) {
    return undefined;
  }
  try {
    const file = readFileSync(pJson, 'utf-8');
    return JSON.parse(file)?.type;
  } catch {
    return undefined;
  }
}

registerHooks({ load });
```

#### 运行钩子

```coffee
# main.coffee
import { scream } from './scream.coffee'
console.log scream 'hello, world'

import { version } from 'node:process'
console.log "Brought to you by Node.js version #{version}"
```

```coffee
# scream.coffee
export scream = (str) -> str.toUpperCase()
```

为了运行示例，添加一个包含 CoffeeScript 文件模块类型的 `package.json` 文件。

```json
{
  "type": "module"
}
```

这仅用于运行示例。在真实的加载器中，即使在没有 `package.json` 中显式类型的情况下，`getPackageType()` 也必须能够返回 Node.js 已知的 `format`，否则 `nextLoad` 调用将抛出 `ERR_UNKNOWN_FILE_EXTENSION`（如果为 undefined）或 `ERR_UNKNOWN_MODULE_FORMAT`（如果它不是 [load 钩子][] 文档中列出的已知格式）。

使用前面的钩子模块，运行
`node --import 'data:text/javascript,import { register } from "node:module"; import { pathToFileURL } from "node:url"; register(pathToFileURL("./coffeescript-hooks.mjs"));' ./main.coffee`
或 `node --import ./coffeescript-sync-hooks.mjs ./main.coffee`
会导致 `main.coffee` 在其源代码从磁盘加载后但在 Node.js 执行之前被转换为 JavaScript；对于任何通过任何加载文件的 `import` 语句引用的 `.coffee`、`.litcoffee` 或 `.coffee.md` 文件也是如此。

#### 导入映射

前两个示例定义了 `load` 钩子。这是一个 `resolve` 钩子的示例。此钩子模块读取一个 `import-map.json` 文件，该文件定义了哪些说明符应覆盖到其他 URL（这是 "导入映射" 规范的一小部分子集的非常简单的实现）。

##### 异步版本

```mjs
// import-map-hooks.js
import fs from 'node:fs/promises';

const { imports } = JSON.parse(await fs.readFile('import-map.json'));

export async function resolve(specifier, context, nextResolve) {
  if (Object.hasOwn(imports, specifier)) {
    return nextResolve(imports[specifier], context);
  }

  return nextResolve(specifier, context);
}
```

##### 同步版本

```mjs
// import-map-sync-hooks.js
import fs from 'node:fs/promises';
import module from 'node:module';

const { imports } = JSON.parse(fs.readFileSync('import-map.json', 'utf-8'));

function resolve(specifier, context, nextResolve) {
  if (Object.hasOwn(imports, specifier)) {
    return nextResolve(imports[specifier], context);
  }

  return nextResolve(specifier, context);
}

module.registerHooks({ resolve });
```

##### 使用钩子

使用这些文件：

```mjs
// main.js
import 'a-module';
```

```json
// import-map.json
{
  "imports": {
    "a-module": "./some-module.js"
  }
}
```

```mjs
// some-module.js
console.log('some module!');
```

运行 `node --import 'data:text/javascript,import { register } from "node:module"; import { pathToFileURL } from "node:url"; register(pathToFileURL("./import-map-hooks.js"));' main.js`
或 `node --import ./import-map-sync-hooks.js main.js`
应该打印 `some module!`。

## Source Map 支持

<!-- YAML
added:
 - v13.7.0
 - v12.17.0
-->

> Stability: 1 - Experimental

Node.js 支持 TC39 ECMA-426 [Source Map][] 格式（以前称为 Source map revision 3 格式）。

本节中的 API 是与 source map 缓存交互的辅助工具。当启用 source map 解析并且在模块的页脚中找到 [source map include directives][] 时，将填充此缓存。

要启用 source map 解析，必须使用标志 [`--enable-source-maps`][] 运行 Node.js，或者通过设置 [`NODE_V8_COVERAGE=dir`][] 启用代码覆盖，或者通过 [`module.setSourceMapsSupport()`][] 以编程方式启用。

```mjs
// module.mjs
// 在 ECMAScript 模块中
import { findSourceMap, SourceMap } from 'node:module';
```

```cjs
// module.cjs
// 在 CommonJS 模块中
const { findSourceMap, SourceMap } = require('node:module');
```

### `module.getSourceMapsSupport()`

<!-- YAML
added:
  - v23.7.0
  - v22.14.0
-->

* 返回: {Object}
  * `enabled` {boolean} 是否启用了 source maps 支持
  * `nodeModules` {boolean} 是否为 `node_modules` 中的文件启用了支持。
  * `generatedCode` {boolean} 是否为来自 `eval` 或 `new Function` 的生成代码启用了支持。

此方法返回是否启用于堆栈跟踪的 [Source Map v3][Source Map] 支持。

<!-- Anchors to make sure old links find a target -->

<a id="module_module_findsourcemap_path_error"></a>

### `module.findSourceMap(path)`

<!-- YAML
added:
 - v13.7.0
 - v12.17.0
-->

* `path` {string}
* 返回: {module.SourceMap|undefined} 如果找到 source map 则返回 `module.SourceMap`，否则返回 `undefined`。

`path` 是应获取对应 source map 的文件的已解析路径。

### `module.setSourceMapsSupport(enabled[, options])`

<!-- YAML
added:
  - v23.7.0
  - v22.14.0
-->

* `enabled` {boolean} 启用 source map 支持。
* `options` {Object} 可选
  * `nodeModules` {boolean} 是否为 `node_modules` 中的文件启用支持。**默认值:** `false`。
  * `generatedCode` {boolean} 是否为来自 `eval` 或 `new Function` 的生成代码启用支持。**默认值:** `false`。

此函数启用或禁用用于堆栈跟踪的 [Source Map v3][Source Map] 支持。

它提供了与使用命令行选项 `--enable-source-maps` 启动 Node.js 进程相同的功能，并附加了用于改变对 `node_modules` 中文件或生成代码的支持的选项。

只有在启用 source maps 后加载的 JavaScript 文件中的 source maps 才会被解析和加载。最好使用命令行选项 `--enable-source-maps` 以避免丢失在此 API 调用之前加载的模块的 source maps。

### 类: `module.SourceMap`

<!-- YAML
added:
 - v13.7.0
 - v12.17.0
-->

#### `new SourceMap(payload[, { lineLengths }])`

<!-- YAML
changes:
  - version: v20.5.0
    pr-url: https://github.com/nodejs/node/pull/48461
    description: Add support for `lineLengths`.
-->

* `payload` {Object}
* `lineLengths` {number\[]}

创建一个新的 `sourceMap` 实例。

`payload` 是一个对象，其键与 [Source map 格式][] 匹配：

* `file` {string}
* `version` {number}
* `sources` {string\[]}
* `sourcesContent` {string\[]}
* `names` {string\[]}
* `mappings` {string}
* `sourceRoot` {string}

`lineLengths` 是一个可选数组，表示生成代码中每行的长度。

#### `sourceMap.payload`

* 返回: {Object}

获取用于构造 [`SourceMap`][] 实例的有效负载。

#### `sourceMap.findEntry(lineOffset, columnOffset)`

* `lineOffset` {number} 生成源文件中的从零开始的行号偏移量
* `columnOffset` {number} 生成源文件中的从零开始的列号偏移量
* 返回: {Object}

给定生成源文件中的行偏移量和列偏移量，如果找到，则返回一个表示原始文件中 SourceMap 范围的对象，否则返回一个空对象。

返回的对象包含以下键：

* `generatedLine` {number} 生成源中范围开始的行偏移量
* `generatedColumn` {number} 生成源中范围开始的列偏移量
* `originalSource` {string} 原始源的文件名，如 SourceMap 中报告的那样
* `originalLine` {number} 原始源中范围开始的行偏移量
* `originalColumn` {number} 原始源中范围开始的列偏移量
* `name` {string}

返回值表示它在 SourceMap 中出现的原始范围，基于从零开始的偏移量，_而不是_ 错误消息和 CallSite 对象中出现的从 1 开始的行号和列号。

要从 Error 堆栈和 CallSite 对象报告的行号和列号获取对应的从 1 开始的行号和列号，请使用 `sourceMap.findOrigin(lineNumber, columnNumber)`

#### `sourceMap.findOrigin(lineNumber, columnNumber)`

<!-- YAML
added:
  - v20.4.0
  - v18.18.0
-->

* `lineNumber` {number} 生成源中调用点的从 1 开始的行号
* `columnNumber` {number} 生成源中调用点的从 1 开始的列号
* 返回: {Object}

给定生成源中调用点的从 1 开始的 `lineNumber` 和 `columnNumber`，查找原始源中对应的调用点位置。

如果提供的 `lineNumber` 和 `columnNumber` 在任何 source map 中都找不到，则返回一个空对象。否则，返回的对象包含以下键：

* `name` {string|undefined} source map 中范围的名称（如果提供了）
* `fileName` {string} 原始源的文件名，如 SourceMap 中报告的那样
* `lineNumber` {number} 原始源中对应调用点的从 1 开始的行号
* `columnNumber` {number} 原始源中对应调用点的从 1 开始的列号

[CommonJS]: modules.md
[条件导出]: packages.md#conditional-exports
[自定义钩子]: #customization-hooks
[ES 模块]: esm.md
[权限模型]: permissions.md#permission-model
[Source Map]: https://tc39.es/ecma426/
[Source map 格式]: https://tc39.es/ecma426/#sec-source-map-format
[V8 JavaScript 代码覆盖率]: https://v8project.blogspot.com/2017/12/javascript-code-coverage.html
[V8 代码缓存]: https://v8.dev/blog/code-caching-for-devs
[`"exports"`]: packages.md#exports
[`--enable-source-maps`]: cli.md#--enable-source-maps
[`--import`]: cli.md#--importmodule
[`--require`]: cli.md#-r---require-module
[`NODE_COMPILE_CACHE=dir`]: cli.md#node_compile_cachedir
[`NODE_DISABLE_COMPILE_CACHE=1`]: cli.md#node_disable_compile_cache1
[`NODE_V8_COVERAGE=dir`]: cli.md#node_v8_coveragedir
[`SourceMap`]: #class-modulesourcemap
[`initialize`]: #initialize
[`module.constants.compileCacheStatus`]: #moduleconstantscompilecachestatus
[`module.enableCompileCache()`]: #moduleenablecompilecachecachedir
[`module.flushCompileCache()`]: #moduleflushcompilecache
[`module.getCompileCacheDir()`]: #modulegetcompilecachedir
[`module.setSourceMapsSupport()`]: #modulesetsourcemapssupportenabled-options
[`module`]: #the-module-object
[`os.tmpdir()`]: os.md#ostmpdir
[`registerHooks`]: #moduleregisterhooksoptions
[`register`]: #moduleregisterspecifier-parenturl-options
[`util.TextDecoder`]: util.md#class-utiltextdecoder
[链]: #chaining
[钩子]: #customization-hooks
[load hook]: #loadurl-context-nextload
[模块编译缓存]: #module-compile-cache
[模块包装器]: modules.md#the-module-wrapper
[领域]: https://tc39.es/ecma262/#realm
[resolve hook]: #resolvespecifier-context-nextresolve
[source map include directives]: https://tc39.es/ecma426/#sec-linking-generated-code
[`Worker` 的文档]: worker_threads.md#new-workerfilename-options
[可传输对象]: worker_threads.md#portpostmessagevalue-transferlist
[转换 TypeScript 特性]: typescript.md#typescript-features
[类型剥离]: typescript.md#type-stripping