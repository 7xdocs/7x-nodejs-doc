# Modules: ECMAScript modules

<!--introduced_in=v8.5.0-->

<!-- type=misc -->

<!-- YAML
added: v8.5.0
changes:
  - version:
    - v23.1.0
    - v22.12.0
    - v20.18.3
    - v18.20.5
    pr-url: https://github.com/nodejs/node/pull/55333
    description: Import attributes are no longer experimental.
  - version: v22.0.0
    pr-url: https://github.com/nodejs/node/pull/52104
    description: Drop support for import assertions.
  - version:
    - v21.0.0
    - v20.10.0
    - v18.20.0
    pr-url: https://github.com/nodejs/node/pull/50140
    description: Add experimental support for import attributes.
  - version:
    - v20.0.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/44710
    description: Module customization hooks are executed off the main thread.
  - version:
    - v18.6.0
    - v16.17.0
    pr-url: https://github.com/nodejs/node/pull/42623
    description: Add support for chaining module customization hooks.
  - version:
    - v17.1.0
    - v16.14.0
    pr-url: https://github.com/nodejs/node/pull/40250
    description: Add experimental support for import assertions.
  - version:
    - v17.0.0
    - v16.12.0
    pr-url: https://github.com/nodejs/node/pull/37468
    description:
      Consolidate customization hooks, removed `getFormat`, `getSource`,
      `transformSource`, and `getGlobalPreloadCode` hooks
      added `load` and `globalPreload` hooks
      allowed returning `format` from either `resolve` or `load` hooks.
  - version:
    - v15.3.0
    - v14.17.0
    - v12.22.0
    pr-url: https://github.com/nodejs/node/pull/35781
    description: Stabilize modules implementation.
  - version:
    - v14.13.0
    - v12.20.0
    pr-url: https://github.com/nodejs/node/pull/35249
    description: Support for detection of CommonJS named exports.
  - version: v14.8.0
    pr-url: https://github.com/nodejs/node/pull/34558
    description: Unflag Top-Level Await.
  - version:
    - v14.0.0
    - v13.14.0
    - v12.20.0
    pr-url: https://github.com/nodejs/node/pull/31974
    description: Remove experimental modules warning.
  - version:
    - v13.2.0
    - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/29866
    description: Loading ECMAScript modules no longer requires a command-line flag.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26745
    description:
      Add support for ES modules using `.js` file extension via `package.json`
      `"type"` field.
-->

> Stability: 2 - Stable

## 引言

<!--name=esm-->

ECMAScript 模块是 [官方标准格式][the official standard format]，用于打包 JavaScript 代码以供重用。模块使用各种 [`import`][] 和 [`export`][] 语句来定义。

以下 ES 模块示例导出了一个函数：

```js
// addTwo.mjs
function addTwo(num) {
  return num + 2;
}

export { addTwo };
```

以下 ES 模块示例从 `addTwo.mjs` 导入了该函数：

```js
// app.mjs
import { addTwo } from './addTwo.mjs';

// Prints: 6
console.log(addTwo(4));
```

Node.js 完全支持当前规范的 ECMAScript 模块，并提供了它们与其原始模块格式 [CommonJS][] 之间的互操作性。

<!-- Anchors to make sure old links find a target -->

<i id="esm_package_json_type_field"></i><i id="esm_package_scope_and_file_extensions"></i><i id="esm_input_type_flag"></i>

## 启用

<!-- type=misc -->

Node.js 有两个模块系统：[CommonJS][] 模块和 ECMAScript 模块。

作者可以通过 `.mjs` 文件扩展名、`package.json` 中值为 `"module"` 的 [`"type"`][] 字段，或值为 `"module"` 的 [`--input-type`][] 标志，来告诉 Node.js 将 JavaScript 解释为 ES 模块。这些是代码旨在作为 ES 模块运行的显式标记。

相反，作者可以通过 `.cjs` 文件扩展名、`package.json` 中值为 `"commonjs"` 的 [`"type"`][] 字段，或值为 `"commonjs"` 的 [`--input-type`][] 标志，来明确告诉 Node.js 将 JavaScript 解释为 CommonJS。

当代码缺乏两种模块系统的显式标记时，Node.js 会检查模块的源代码以查找 ES 模块语法。如果找到此类语法，Node.js 会将代码作为 ES 模块运行；否则，它将作为 CommonJS 运行模块。有关更多详细信息，请参阅 [确定模块系统][Determining module system]。

<!-- Anchors to make sure old links find a target -->

<i id="esm_package_entry_points"></i><i id="esm_main_entry_point_export"></i><i id="esm_subpath_exports"></i><i id="esm_package_exports_fallbacks"></i><i id="esm_exports_sugar"></i><i id="esm_conditional_exports"></i><i id="esm_nested_conditions"></i><i id="esm_self_referencing_a_package_using_its_name"></i><i id="esm_internal_package_imports"></i><i id="esm_dual_commonjs_es_module_packages"></i><i id="esm_dual_package_hazard"></i><i id="esm_writing_dual_packages_while_avoiding_or_minimizing_hazards"></i><i id="esm_approach_1_use_an_es_module_wrapper"></i><i id="esm_approach_2_isolate_state"></i>

## 包

此部分已移至 [Modules: Packages](packages.md)。

## `import` 说明符

### 术语

`import` 语句的 _说明符_ 是 `from` 关键字之后的字符串，例如 `import { sep } from 'node:path'` 中的 `'node:path'`。说明符也用于 `export from` 语句，以及作为 `import()` 表达式的参数。

说明符有三种类型：

* _相对说明符_，如 `'./startup.js'` 或 `'../config.mjs'`。它们引用相对于导入文件位置的路径。_对于这些，文件扩展名总是必需的。_

* _裸说明符_，如 `'some-package'` 或 `'some-package/shuffle'`。它们可以分别通过包名引用包的主入口点，或者通过包名前缀引用包内的特定功能模块。_仅对于没有 [`"exports"`][] 字段的包，才需要包含文件扩展名。_

* _绝对说明符_，如 `'file:///opt/nodejs/config.js'`。它们直接且明确地引用完整路径。

裸说明符的解析由 [Node.js 模块解析和加载算法][Node.js module resolution and loading algorithm] 处理。
所有其他说明符的解析始终仅使用标准的相对 [URL][] 解析语义。

与 CommonJS 中一样，除非包的 [`package.json`][] 包含 [`"exports"`][] 字段，否则可以通过将路径附加到包名来访问包内的模块文件。如果包含 [`"exports"`][] 字段，则包内的文件只能通过 [`"exports"`][] 中定义的路径访问。

有关适用于 Node.js 模块解析中裸说明符的这些包解析规则的详细信息，请参阅 [包文档](packages.md)。

### 强制文件扩展名

使用 `import` 关键字解析相对或绝对说明符时，必须提供文件扩展名。目录索引（例如 `'./startup/index.js'`）也必须完全指定。

此行为与 `import` 在浏览器环境中的行为相匹配，假设服务器配置通常如此。

### URLs

ES 模块作为 URL 进行解析和缓存。这意味着特殊字符必须进行 [百分比编码][percent-encoded]，例如 `#` 编码为 `%23`，`?` 编码为 `%3F`。

支持 `file:`、`node:` 和 `data:` URL 方案。像 `'https://example.com/app.js'` 这样的说明符在 Node.js 中本身不支持，除非使用 [自定义 HTTPS 加载器][custom HTTPS loader]。

#### `file:` URLs

如果用于解析模块的 `import` 说明符具有不同的查询或片段，则模块会被多次加载。

```js
import './foo.mjs?query=1'; // 加载带有 "?query=1" 查询的 ./foo.mjs
import './foo.mjs?query=2'; // 加载带有 "?query=2" 查询的 ./foo.mjs
```

可以通过 `/`、`//` 或 `file:///` 引用卷根目录。鉴于 [URL][] 和路径解析之间的差异（例如百分比编码细节），建议在导入路径时使用 [url.pathToFileURL][]。

#### `data:` 导入

<!-- YAML
added: v12.10.0
-->

支持使用以下 MIME 类型通过 [`data:` URLs][] 进行导入：

* 用于 ES 模块的 `text/javascript`
* 用于 JSON 的 `application/json`
* 用于 Wasm 的 `application/wasm`

```js
import 'data:text/javascript,console.log("hello!");';
import _ from 'data:application/json,"world!"' with { type: 'json' };
```

`data:` URLs 仅解析内置模块的 [裸说明符][Terminology] 和 [绝对说明符][Terminology]。解析 [相对说明符][Terminology] 不起作用，因为 `data:` 不是 [特殊方案][special scheme]。例如，尝试从 `data:text/javascript,import "./foo";` 加载 `./foo` 会解析失败，因为 `data:` URLs 没有相对解析的概念。

#### `node:` 导入

<!-- YAML
added:
  - v14.13.1
  - v12.20.0
changes:
  - version:
      - v16.0.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37246
    description: Added `node:` import support to `require(...)`.
-->

`node:` URLs 作为加载 Node.js 内置模块的替代方式被支持。此 URL 方案允许通过有效的绝对 URL 字符串引用内置模块。

```js
import fs from 'node:fs/promises';
```

<a id="import-assertions"></a>

## 导入属性

<!-- YAML
added:
  - v17.1.0
  - v16.14.0
changes:
  - version:
    - v21.0.0
    - v20.10.0
    - v18.20.0
    pr-url: https://github.com/nodejs/node/pull/50140
    description: Switch from Import Assertions to Import Attributes.
-->

[导入属性][Import Attributes MDN] 是模块导入语句的内联语法，用于沿着模块说明符传递更多信息。

```js
import fooData from './foo.json' with { type: 'json' };

const { default: barData } =
  await import('./bar.json', { with: { type: 'json' } });
```

Node.js 仅支持 `type` 属性，它支持以下值：

| 属性 `type` | 用于             |
| ---------------- | ---------------- |
| `'json'`         | [JSON 模块][] |

导入 JSON 模块时，`type: 'json'` 属性是强制性的。

## 内置模块

[内置模块][Built-in modules] 提供其公共 API 的命名导出。还提供了一个默认导出，它是 CommonJS 导出的值。默认导出可用于，除其他外，修改命名导出。内置模块的命名导出仅通过调用 [`module.syncBuiltinESMExports()`][] 来更新。

```js
import EventEmitter from 'node:events';
const e = new EventEmitter();
```

```js
import { readFile } from 'node:fs';
readFile('./foo.txt', (err, source) => {
  if (err) {
    console.error(err);
  } else {
    console.log(source);
  }
});
```

```js
import fs, { readFileSync } from 'node:fs';
import { syncBuiltinESMExports } from 'node:module';
import { Buffer } from 'node:buffer';

fs.readFileSync = () => Buffer.from('Hello, ESM');
syncBuiltinESMExports();

fs.readFileSync === readFileSync;
```

## `import()` 表达式

[动态 `import()`][Dynamic `import()`] 提供了一种异步导入模块的方式。它在 CommonJS 和 ES 模块中都受支持，并且可用于加载 CommonJS 和 ES 模块。

## `import.meta`

* 类型: {Object}

`import.meta` 元属性是一个 `Object`，包含以下属性。它仅在 ES 模块中受支持。

### `import.meta.dirname`

<!-- YAML
added:
  - v21.2.0
  - v20.11.0
changes:
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/58011
    description: This property is no longer experimental.
-->

* 类型: {string} 当前模块的目录名。

这与 [`import.meta.filename`][] 的 [`path.dirname()`][] 相同。

> **注意**：仅存在于 `file:` 模块上。

### `import.meta.filename`

<!-- YAML
added:
  - v21.2.0
  - v20.11.0
changes:
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/58011
    description: This property is no longer experimental.
-->

* 类型: {string} 当前模块的完整绝对路径和文件名，已解析符号链接。

这与 [`import.meta.url`][] 的 [`url.fileURLToPath()`][] 相同。

> **注意** 仅本地模块支持此属性。不使用 `file:` 协议的模块不提供此属性。

### `import.meta.url`

* 类型: {string} 模块的绝对 `file:` URL。

这与在浏览器中定义的方式完全相同，提供当前模块文件的 URL。

这启用了有用的模式，例如相对文件加载：

```js
import { readFileSync } from 'node:fs';
const buffer = readFileSync(new URL('./data.proto', import.meta.url));
```

### `import.meta.main`

<!-- YAML
added:
  - v24.2.0
-->

> Stability: 1.0 - Early development

* 类型: {boolean} 当当前模块是当前进程的入口点时，为 `true`；否则为 `false`。

等效于 CommonJS 中的 `require.main === module`。

类似于 Python 的 `__name__ == "__main__"`。

```js
export function foo() {
  return 'Hello, world';
}

function main() {
  const message = foo();
  console.log(message);
}

if (import.meta.main) main();
// `foo` 可以从另一个模块导入，而不会受到 `main` 可能的副作用影响
```

### `import.meta.resolve(specifier)`

<!-- YAML
added:
  - v13.9.0
  - v12.16.2
changes:
  - version:
    - v20.6.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/49028
    description: No longer behind `--experimental-import-meta-resolve` CLI flag,
                 except for the non-standard `parentURL` parameter.
  - version:
    - v20.6.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/49038
    description: This API no longer throws when targeting `file:` URLs that do
                 not map to an existing file on the local FS.
  - version:
    - v20.0.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/44710
    description: This API now returns a string synchronously instead of a Promise.
  - version:
      - v16.2.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38587
    description: Add support for WHATWG `URL` object to `parentURL` parameter.
-->

> Stability: 1.2 - Release candidate

* `specifier` {string} 相对于当前模块要解析的模块说明符。
* 返回: {string} 说明符将解析到的绝对 URL 字符串。

[`import.meta.resolve`][] 是一个相对于每个模块的作用域解析函数，返回 URL 字符串。

```js
const dependencyAsset = import.meta.resolve('component-lib/asset.css');
// file:///app/node_modules/component-lib/asset.css
import.meta.resolve('./dep.js');
// file:///app/dep.js
```

支持 Node.js 模块解析的所有功能。依赖项解析受包内允许的导出解析的限制。

**注意事项**：

* 这可能导致同步文件系统操作，这可能会类似地影响性能，如同 `require.resolve`。
* 此功能在自定义加载器中不可用（它会导致死锁）。

**非标准 API**：

当使用 `--experimental-import-meta-resolve` 标志时，该函数接受第二个参数：

* `parent` {string|URL} 一个可选的绝对父模块 URL，用于从中解析。**默认值:** `import.meta.url`

## 与 CommonJS 的互操作性

### `import` 语句

`import` 语句可以引用 ES 模块或 CommonJS 模块。`import` 语句仅在 ES 模块中允许，但动态 [`import()`][] 表达式在 CommonJS 中受支持，用于加载 ES 模块。

导入 [CommonJS 模块](#commonjs-namespaces) 时，`module.exports` 对象作为默认导出提供。命名导出可能可用，通过静态分析提供，以便更好地实现生态系统兼容性。

### `require`

CommonJS 模块 `require` 目前仅支持加载同步 ES 模块（即不使用顶层 `await` 的 ES 模块）。

有关详细信息，请参阅 [使用 `require()` 加载 ECMAScript 模块][Loading ECMAScript modules using `require()`]。

### CommonJS 命名空间

<!-- YAML
added: v14.13.0
changes:
  - version: v23.0.0
    pr-url: https://github.com/nodejs/node/pull/53848
    description: Added `'module.exports'` export marker to CJS namespaces.
-->

CommonJS 模块由一个 `module.exports` 对象组成，该对象可以是任何类型。

为了支持这一点，当从 ECMAScript 模块导入 CommonJS 时，会为 CommonJS 模块构造一个命名空间包装器，该包装器始终提供一个指向 CommonJS `module.exports` 值的 `default` 导出键。

此外，对 CommonJS 模块的源文本执行启发式静态分析，以获取 `module.exports` 上值的尽力静态导出列表，以在命名空间上提供。这是必要的，因为这些命名空间必须在 CJS 模块评估之前构造。

这些 CommonJS 命名空间对象还提供了一个 `default` 导出作为 `'module.exports'` 命名导出，以便明确指示它们在 CommonJS 中的表示使用此值，而不是命名空间值。这反映了 [`require(esm)`][] 互操作性支持中处理 `'module.exports'` 导出名称的语义。

导入 CommonJS 模块时，可以使用 ES 模块默认导入或其相应的语法糖可靠地导入：

<!-- eslint-disable no-duplicate-imports -->

```js
import { default as cjs } from 'cjs';
// 与上述相同
import cjsSugar from 'cjs';

console.log(cjs);
console.log(cjs === cjsSugar);
// 打印:
//   <module.exports>
//   true
```

当使用 `import * as m from 'cjs'` 或动态导入时，可以直接观察到这个模块命名空间异质对象：

<!-- eslint-skip -->

```js
import * as m from 'cjs';
console.log(m);
console.log(m === await import('cjs'));
// 打印:
//   [Module] { default: <module.exports>, 'module.exports': <module.exports> }
//   true
```

为了更好地与 JS 生态系统中的现有用法兼容，Node.js 还尝试确定每个导入的 CommonJS 模块的 CommonJS 命名导出，以通过静态分析过程将它们作为单独的 ES 模块导出提供。

例如，考虑一个用以下方式编写的 CommonJS 模块：

```cjs
// cjs.cjs
exports.name = 'exported';
```

前面的模块在 ES 模块中支持命名导入：

<!-- eslint-disable no-duplicate-imports -->

```js
import { name } from './cjs.cjs';
console.log(name);
// 打印: 'exported'

import cjs from './cjs.cjs';
console.log(cjs);
// 打印: { name: 'exported' }

import * as m from './cjs.cjs';
console.log(m);
// 打印:
//   [Module] {
//     default: { name: 'exported' },
//     'module.exports': { name: 'exported' },
//     name: 'exported'
//   }
```

从记录的模块命名空间异质对象的最后一个示例中可以看出，`name` 导出从 `module.exports` 对象复制并在导入模块时直接设置在 ES 模块命名空间上。

对这些命名导出不会检测到对 `module.exports` 的实时绑定更新或新增的导出。

命名导出的检测基于常见的语法模式，但并非总能正确检测到命名导出。在这些情况下，使用上述默认导入形式可能是更好的选择。

命名导出检测涵盖了许多常见的导出模式、重新导出模式以及构建工具和转译器的输出。有关实现的精确语义，请参阅 [cjs-module-lexer][]。

### ES 模块和 CommonJS 之间的差异

#### 没有 `require`、`exports` 或 `module.exports`

在大多数情况下，可以使用 ES 模块 `import` 来加载 CommonJS 模块。

如果需要，可以使用 [`module.createRequire()`][] 在 ES 模块内构造 `require` 函数。

#### 没有 `__filename` 或 `__dirname`

这些 CommonJS 变量在 ES 模块中不可用。

`__filename` 和 `__dirname` 的用例可以通过 [`import.meta.filename`][] 和 [`import.meta.dirname`][] 来复制。

#### 没有插件加载

[插件][Addons] 目前不支持通过 ES 模块导入。

它们可以使用 [`module.createRequire()`][] 或 [`process.dlopen`][] 加载。

#### 没有 `require.main`

要替换 `require.main === module`，有 [`import.meta.main`][] API。

#### 没有 `require.resolve`

相对解析可以通过 `new URL('./local', import.meta.url)` 处理。

对于完整的 `require.resolve` 替换，有 [import.meta.resolve][] API。

或者可以使用 `module.createRequire()`。

#### 没有 `NODE_PATH`

`NODE_PATH` 不是解析 `import` 说明符的一部分。如果需要此行为，请使用符号链接。

#### 没有 `require.extensions`

`import` 不使用 `require.extensions`。模块自定义钩子可以提供替代方案。

#### 没有 `require.cache`

`import` 不使用 `require.cache`，因为 ES 模块加载器有自己的独立缓存。

<i id="esm_experimental_json_modules"></i>

## JSON 模块

<!-- YAML
changes:
  - version:
    - v23.1.0
    - v22.12.0
    - v20.18.3
    - v18.20.5
    pr-url: https://github.com/nodejs/node/pull/55333
    description: JSON modules are no longer experimental.
-->

JSON 文件可以通过 `import` 引用：

```js
import packageConfig from './package.json' with { type: 'json' };
```

`with { type: 'json' }` 语法是强制性的；请参阅 [导入属性][Import Attributes]。

导入的 JSON 仅公开一个 `default` 导出。不支持命名导出。会在 CommonJS 缓存中创建一个缓存条目以避免重复。如果已经从同一路径导入了 JSON 模块，则在 CommonJS 中返回相同的对象。

<i id="esm_experimental_wasm_modules"></i>

## Wasm 模块

<!-- YAML
changes:
  - version: v24.5.0
    pr-url: https://github.com/nodejs/node/pull/57038
    description: Wasm modules no longer require the `--experimental-wasm-modules` flag.
-->

支持导入 WebAssembly 模块实例和 WebAssembly 源阶段导入。

这两种集成都符合 [WebAssembly 的 ES 模块集成提案][ES Module Integration Proposal for WebAssembly]。

### Wasm 源阶段导入

> Stability: 1.2 - Release candidate

<!-- YAML
added: v24.0.0
-->

[源阶段导入][Source Phase Imports] 提案允许使用 `import source` 关键字组合直接导入 `WebAssembly.Module` 对象，而不是获取已经使用其依赖项实例化的模块实例。

当需要为 Wasm 进行自定义实例化，同时仍然通过 ES 模块集成解析和加载它时，这很有用。

例如，要创建模块的多个实例，或将自定义导入传递到 `library.wasm` 的新实例中：

```js
import source libraryModule from './library.wasm';

const instance1 = await WebAssembly.instantiate(libraryModule, importObject1);

const instance2 = await WebAssembly.instantiate(libraryModule, importObject2);
```

除了静态源阶段之外，还有通过 `import.source` 动态阶段导入语法的动态变体：

```js
const dynamicLibrary = await import.source('./library.wasm');

const instance = await WebAssembly.instantiate(dynamicLibrary, importObject);
```

### JavaScript 字符串内置函数

> Stability: 1.2 - Release candidate

<!-- YAML
added: v24.5.0
-->

导入 WebAssembly 模块时，[WebAssembly JS 字符串内置函数提案][WebAssembly JS String Builtins Proposal] 通过 ESM 集成自动启用。这允许 WebAssembly 模块直接使用来自 `wasm:js-string` 命名空间的高效编译时字符串内置函数。

例如，以下 Wasm 模块使用 `wasm:js-string` 的 `length` 内置函数导出一个字符串 `getLength` 函数：

```text
(module
  ;; 编译时导入字符串长度内置函数。
  (import "wasm:js-string" "length" (func $string_length (param externref) (result i32)))

  ;; 定义 getLength，接受一个假定为字符串的 JS 值参数，
  ;; 对其调用字符串长度并返回结果。
  (func $getLength (param $str externref) (result i32)
    local.get $str
    call $string_length
  )

  ;; 导出 getLength 函数。
  (export "getLength" (func $get_length))
)
```

```js
import { getLength } from './string-len.wasm';
getLength('foo'); // 返回 3。
```

Wasm 内置函数是编译时导入，在模块编译期间链接，而不是在实例化期间。它们的行为不像普通的模块图导入，并且无法通过 `WebAssembly.Module.imports(mod)` 进行检查或虚拟化，除非使用直接的 `WebAssembly.compile` API 并禁用字符串内置函数重新编译模块。

在模块实例化之前以源阶段导入模块也会自动使用编译时内置函数：

```js
import source mod from './string-len.wasm';
const { exports: { getLength } } = await WebAssembly.instantiate(mod, {});
getLength('foo'); // 也返回 3。
```

### Wasm 实例阶段导入

> Stability: 1.1 - Active development

实例导入允许任何 `.wasm` 文件作为普通模块导入，进而支持它们的模块导入。

例如，一个包含以下内容的 `index.js`：

```js
import * as M from './library.wasm';
console.log(M);
```

在以下情况下执行：

```bash
node index.mjs
```

将提供 `library.wasm` 实例化的导出接口。

### 保留的 Wasm 命名空间

<!-- YAML
added: v24.5.0
-->

导入 WebAssembly 模块实例时，它们不能使用以保留前缀开头的导入模块名称或导入/导出名称：

* `wasm-js:` - 在所有模块导入名称、模块名称和导出名称中保留。
* `wasm:` - 在模块导入名称和导出名称中保留（允许导入的模块名称以支持未来的内置函数填充）。

使用上述保留名称导入模块将抛出 `WebAssembly.LinkError`。

<i id="esm_experimental_top_level_await"></i>

## 顶层 `await`

<!-- YAML
added: v14.8.0
-->

`await` 关键字可以在 ECMAScript 模块的顶层主体中使用。

假设有一个 `a.mjs`，内容为：

```js
export const five = await Promise.resolve(5);
```

以及一个 `b.mjs`，内容为：

```js
import { five } from './a.mjs';

console.log(five); // 打印 `5`
```

```bash
node b.mjs # 正常工作
```

如果顶层 `await` 表达式从未解析，`node` 进程将以 `13` [状态码][status code] 退出。

```js
import { spawn } from 'node:child_process';
import { execPath } from 'node:process';

spawn(execPath, [
  '--input-type=module',
  '--eval',
  // 永不解析的 Promise：
  'await new Promise(() => {})',
]).once('exit', (code) => {
  console.log(code); // 打印 `13`
});
```

<i id="esm_experimental_loaders"></i>

## 加载器

以前的加载器文档现在位于 [Modules: Customization hooks][Module customization hooks]。

## 解析和加载算法

### 特性

默认解析器具有以下属性：

* 基于 FileURL 的解析，如同 ES 模块所使用的
* 相对和绝对 URL 解析
* 无默认扩展名
* 无文件夹主文件
* 通过 node\_modules 进行裸说明符包解析查找
* 不会因未知扩展名或协议而失败
* 可以选择向加载阶段提供格式提示

默认加载器具有以下属性：

* 通过 `node:` URLs 支持内置模块加载
* 通过 `data:` URLs 支持“内联”模块加载
* 支持 `file:` 模块加载
* 对任何其他 URL 协议失败
* 对 `file:` 加载的未知扩展名失败（仅支持 `.cjs`、`.js` 和 `.mjs`）

### 解析算法

加载 ES 模块说明符的算法通过下面的 **ESM\_RESOLVE** 方法给出。它返回相对于 parentURL 的模块说明符的解析后 URL。

解析算法确定模块加载的完整解析 URL，以及其建议的模块格式。解析算法不确定解析后的 URL 协议是否可以加载，或者文件扩展名是否被允许，相反，这些验证由 Node.js 在加载阶段应用（例如，如果要求加载一个具有非 `file:`、`data:` 或 `node:` 协议的 URL）。

该算法还尝试根据扩展名确定文件的格式（参见下面的 `ESM_FILE_FORMAT` 算法）。如果它不认识文件扩展名（例如，如果不是 `.mjs`、`.cjs` 或 `.json`），则返回 `undefined` 格式，这将在加载阶段抛出。

确定解析 URL 的模块格式的算法由 **ESM\_FILE\_FORMAT** 提供，它返回任何文件的唯一模块格式。对于 ECMAScript 模块返回 _"module"_ 格式，而 _"commonjs"_ 格式用于指示通过传统 CommonJS 加载器加载。未来更新可以扩展其他格式，例如 _"addon"_。

在以下算法中，除非另有说明，所有子程序错误都会作为这些顶级例程的错误传播。

_defaultConditions_ 是条件环境名称数组，`["node", "import"]`。

解析器可以抛出以下错误：

* _无效模块说明符_：模块说明符是无效的 URL、包名或包子路径说明符。
* _无效包配置_：package.json 配置无效或包含无效配置。
* _无效包目标_：包导出或导入为包定义了一个无效类型或字符串目标模块。
* _包路径未导出_：包导出未定义或不允许包中给定模块的目标子路径。
* _包导入未定义_：包导入未定义说明符。
* _模块未找到_：请求的包或模块不存在。
* _不支持的目录导入_：解析后的路径对应于一个目录，这不是模块导入支持的目标。

### 解析算法规范

**ESM\_RESOLVE**(_specifier_, _parentURL_)

> 1. 令 _resolved_ 为 **undefined**。
> 2. 如果 _specifier_ 是有效的 URL，则
>    1. 将 _resolved_ 设置为将 _specifier_ 解析并重新序列化为 URL 的结果。
> 3. 否则，如果 _specifier_ 以 _"/"_、_"./"_ 或 _"../"_ 开头，则
>    1. 将 _resolved_ 设置为 _specifier_ 相对于 _parentURL_ 的 URL 解析结果。
> 4. 否则，如果 _specifier_ 以 _"#"_ 开头，则
>    1. 将 _resolved_ 设置为 **PACKAGE\_IMPORTS\_RESOLVE**(_specifier_, _parentURL_, _defaultConditions_) 的结果。
> 5. 否则，
>    1. 注意：_specifier_ 现在是裸说明符。
>    2. 将 _resolved_ 设置为 **PACKAGE\_RESOLVE**(_specifier_, _parentURL_) 的结果。
> 6. 令 _format_ 为 **undefined**。
> 7. 如果 _resolved_ 是 _"file:"_ URL，则
>    1. 如果 _resolved_ 包含 _"/"_ 或 _"\\"_ 的任何百分比编码（分别为 _"%2F"_ 和 _"%5C"_），则
>       1. 抛出 _无效模块说明符_ 错误。
>    2. 如果 _resolved_ 处的文件是目录，则
>       1. 抛出 _不支持的目录导入_ 错误。
>    3. 如果 _resolved_ 处的文件不存在，则
>       1. 抛出 _模块未找到_ 错误。
>    4. 将 _resolved_ 设置为 _resolved_ 的真实路径，保持相同的 URL 查询字符串和片段组件。
>    5. 将 _format_ 设置为 **ESM\_FILE\_FORMAT**(_resolved_) 的结果。
> 8. 否则，
>    1. 将 _format_ 设置为与 URL _resolved_ 关联的内容类型的模块格式。
> 9. 将 _format_ 和 _resolved_ 返回给加载阶段。

**PACKAGE\_RESOLVE**(_packageSpecifier_, _parentURL_)

> 1. 令 _packageName_ 为 **undefined**。
> 2. 如果 _packageSpecifier_ 是空字符串，则
>    1. 抛出 _无效模块说明符_ 错误。
> 3. 如果 _packageSpecifier_ 是 Node.js 内置模块名称，则
>    1. 返回字符串 _"node:"_ 连接 _packageSpecifier_。
> 4. 如果 _packageSpecifier_ 不以 _"@"_ 开头，则
>    1. 将 _packageName_ 设置为 _packageSpecifier_ 的子字符串，直到第一个 _"/"_ 分隔符或字符串结尾。
> 5. 否则，
>    1. 如果 _packageSpecifier_ 不包含 _"/"_ 分隔符，则
>       1. 抛出 _无效模块说明符_ 错误。
>    2. 将 _packageName_ 设置为 _packageSpecifier_ 的子字符串，直到第二个 _"/"_ 分隔符或字符串结尾。
> 6. 如果 _packageName_ 以 _"."_ 开头或包含 _"\\"_ 或 _"%"_，则
>    1. 抛出 _无效模块说明符_ 错误。
> 7. 令 _packageSubpath_ 为 _"."_ 连接 _packageSpecifier_ 从 _packageName_ 长度位置开始的子字符串。
> 8. 令 _selfUrl_ 为 **PACKAGE\_SELF\_RESOLVE**(_packageName_, _packageSubpath_, _parentURL_) 的结果。
> 9. 如果 _selfUrl_ 不是 **undefined**，返回 _selfUrl_。
> 10. 当 _parentURL_ 不是文件系统根目录时，
>     1. 令 _packageURL_ 为 _"node\_modules/"_ 连接 _packageName_ 相对于 _parentURL_ 的 URL 解析结果。
>     2. 将 _parentURL_ 设置为 _parentURL_ 的父文件夹 URL。
>     3. 如果 _packageURL_ 处的文件夹不存在，则
>        1. 继续下一个循环迭代。
>     4. 令 _pjson_ 为 **READ\_PACKAGE\_JSON**(_packageURL_) 的结果。
>     5. 如果 _pjson_ 不是 **null** 且 _pjson_._exports_ 不是 **null** 或 **undefined**，则
>        1. 返回 **PACKAGE\_EXPORTS\_RESOLVE**(_packageURL_, _packageSubpath_, _pjson.exports_, _defaultConditions_) 的结果。
>     6. 否则，如果 _packageSubpath_ 等于 _"."_，则
>        1. 如果 _pjson.main_ 是字符串，则
>           1. 返回 _packageURL_ 中 _main_ 的 URL 解析结果。
>     7. 否则，
>        1. 返回 _packageURL_ 中 _packageSubpath_ 的 URL 解析结果。
> 11. 抛出 _模块未找到_ 错误。

**PACKAGE\_SELF\_RESOLVE**(_packageName_, _packageSubpath_, _parentURL_)

> 1. 令 _packageURL_ 为 **LOOKUP\_PACKAGE\_SCOPE**(_parentURL_) 的结果。
> 2. 如果 _packageURL_ 是 **null**，则
>    1. 返回 **undefined**。
> 3. 令 _pjson_ 为 **READ\_PACKAGE\_JSON**(_packageURL_) 的结果。
> 4. 如果 _pjson_ 是 **null** 或如果 _pjson_._exports_ 是 **null** 或 **undefined**，则
>    1. 返回 **undefined**。
> 5. 如果 _pjson.name_ 等于 _packageName_，则
>    1. 返回 **PACKAGE\_EXPORTS\_RESOLVE**(_packageURL_, _packageSubpath_, _pjson.exports_, _defaultConditions_) 的结果。
> 6. 否则，返回 **undefined**。

**PACKAGE\_EXPORTS\_RESOLVE**(_packageURL_, _subpath_, _exports_, _conditions_)

注意：此函数由 CommonJS 解析算法直接调用。

> 1. 如果 _exports_ 是一个对象，同时具有以 _"."_ 开头的键和不以 _"."_ 开头的键，则抛出 _无效包配置_ 错误。
> 2. 如果 _subpath_ 等于 _"."_，则
>    1. 令 _mainExport_ 为 **undefined**。
>    2. 如果 _exports_ 是字符串或数组，或者是不包含以 _"."_ 开头的键的对象，则
>       1. 将 _mainExport_ 设置为 _exports_。
>    3. 否则，如果 _exports_ 是包含 _"."_ 属性的对象，则
>       1. 将 _mainExport_ 设置为 _exports_\[_"."_]。
>    4. 如果 _mainExport_ 不是 **undefined**，则
>       1. 令 _resolved_ 为 **PACKAGE\_TARGET\_RESOLVE**( _packageURL_, _mainExport_, **null**, **false**, _conditions_) 的结果。
>       2. 如果 _resolved_ 不是 **null** 或 **undefined**，返回 _resolved_。
> 3. 否则，如果 _exports_ 是一个对象且 _exports_ 的所有键都以 _"."_ 开头，则
>    1. 断言：_subpath_ 以 _"./"_ 开头。
>    2. 令 _resolved_ 为 **PACKAGE\_IMPORTS\_EXPORTS\_RESOLVE**( _subpath_, _exports_, _packageURL_, **false**, _conditions_) 的结果。
>    3. 如果 _resolved_ 不是 **null** 或 **undefined**，返回 _resolved_。
> 4. 抛出 _包路径未导出_ 错误。

**PACKAGE\_IMPORTS\_RESOLVE**(_specifier_, _parentURL_, _conditions_)

注意：此函数由 CommonJS 解析算法直接调用。

> 1. 断言：_specifier_ 以 _"#"_ 开头。
> 2. 如果 _specifier_ 完全等于 _"#"_ 或以 _"#/"_ 开头，则
>    1. 抛出 _无效模块说明符_ 错误。
> 3. 令 _packageURL_ 为 **LOOKUP\_PACKAGE\_SCOPE**(_parentURL_) 的结果。
> 4. 如果 _packageURL_ 不是 **null**，则
>    1. 令 _pjson_ 为 **READ\_PACKAGE\_JSON**(_packageURL_) 的结果。
>    2. 如果 _pjson.imports_ 是非空对象，则
>       1. 令 _resolved_ 为 **PACKAGE\_IMPORTS\_EXPORTS\_RESOLVE**( _specifier_, _pjson.imports_, _packageURL_, **true**, _conditions_) 的结果。
>       2. 如果 _resolved_ 不是 **null** 或 **undefined**，返回 _resolved_。
> 5. 抛出 _包导入未定义_ 错误。

**PACKAGE\_IMPORTS\_EXPORTS\_RESOLVE**(_matchKey_, _matchObj_, _packageURL_, _isImports_, _conditions_)

> 1. 如果 _matchKey_ 以 _"/"_ 结尾，则
>    1. 抛出 _无效模块说明符_ 错误。
> 2. 如果 _matchKey_ 是 _matchObj_ 的键且不包含 _"\*"_，则
>    1. 令 _target_ 为 _matchObj_\[_matchKey_] 的值。
>    2. 返回 **PACKAGE\_TARGET\_RESOLVE**(_packageURL_, _target_, **null**, _isImports_, _conditions_) 的结果。
> 3. 令 _expansionKeys_ 为 _matchObj_ 的键列表，这些键仅包含单个 _"\*"_，通过排序函数 **PATTERN\_KEY\_COMPARE** 排序，该函数按特异性降序排序。
> 4. 对于 _expansionKeys_ 中的每个键 _expansionKey_，执行
>    1. 令 _patternBase_ 为 _expansionKey_ 的子字符串，直到但不包括第一个 _"\*"_ 字符。
>    2. 如果 _matchKey_ 以 _patternBase_ 开头但不等于 _patternBase_，则
>       1. 令 _patternTrailer_ 为 _expansionKey_ 从第一个 _"\*"_ 字符之后的索引开始的子字符串。
>       2. 如果 _patternTrailer_ 长度为零，或者如果 _matchKey_ 以 _patternTrailer_ 结尾且 _matchKey_ 的长度大于或等于 _expansionKey_ 的长度，则
>          1. 令 _target_ 为 _matchObj_\[_expansionKey_] 的值。
>          2. 令 _patternMatch_ 为 _matchKey_ 从 _patternBase_ 长度索引开始到 _matchKey_ 长度减去 _patternTrailer_ 长度的子字符串。
>          3. 返回 **PACKAGE\_TARGET\_RESOLVE**(_packageURL_, _target_, _patternMatch_, _isImports_, _conditions_) 的结果。
> 5. 返回 **null**。

**PATTERN\_KEY\_COMPARE**(_keyA_, _keyB_)

> 1. 断言：_keyA_ 仅包含单个 _"\*"_。
> 2. 断言：_keyB_ 仅包含单个 _"\*"_。
> 3. 令 _baseLengthA_ 为 _keyA_ 中 _"\*"_ 的索引。
> 4. 令 _baseLengthB_ 为 _keyB_ 中 _"\*"_ 的索引。
> 5. 如果 _baseLengthA_ 大于 _baseLengthB_，返回 -1。
> 6. 如果 _baseLengthB_ 大于 _baseLengthA_，返回 1。
> 7. 如果 _keyA_ 的长度大于 _keyB_ 的长度，返回 -1。
> 8. 如果 _keyB_ 的长度大于 _keyA_ 的长度，返回 1。
> 9. 返回 0。

**PACKAGE\_TARGET\_RESOLVE**(_packageURL_, _target_, _patternMatch_, _isImports_, _conditions_)

> 1. 如果 _target_ 是字符串，则
>    1. 如果 _target_ 不以 _"./"_ 开头，则
>       1. 如果 _isImports_ 为 **false**，或者如果 _target_ 以 _"../"_ 或 _"/"_ 开头，或者如果 _target_ 是有效的 URL，则
>          1. 抛出 _无效包目标_ 错误。
>       2. 如果 _patternMatch_ 是字符串，则
>          1. 返回 **PACKAGE\_RESOLVE**(_target_ 中每个 _"\*"_ 实例替换为 _patternMatch_ 的结果, _packageURL_ + _"/"_)。
>       3. 返回 **PACKAGE\_RESOLVE**(_target_, _packageURL_ + _"/"_)。
>    2. 如果 _target_ 在 _"/"_ 或 _"\\"_ 上拆分后，在第一个 _"."_ 段之后包含任何 _""_、_"."_、_".."_ 或 _"node\_modules"_ 段，不区分大小写且包括百分比编码变体，则抛出 _无效包目标_ 错误。
>    3. 令 _resolvedTarget_ 为 _packageURL_ 和 _target_ 连接后的 URL 解析结果。
>    4. 断言：_packageURL_ 包含在 _resolvedTarget_ 中。
>    5. 如果 _patternMatch_ 为 **null**，则
>       1. 返回 _resolvedTarget_。
>    6. 如果 _patternMatch_ 在 _"/"_ 或 _"\\"_ 上拆分后包含任何 _""_、_"."_、_".."_ 或 _"node\_modules"_ 段，不区分大小写且包括百分比编码变体，则抛出 _无效模块说明符_ 错误。
>    7. 返回 _resolvedTarget_ 中每个 _"\*"_ 实例替换为 _patternMatch_ 后的 URL 解析结果。
> 2. 否则，如果 _target_ 是非空对象，则
>    1. 如果 _target_ 包含任何索引属性键，如 ECMA-262 [6.1.7 Array Index][] 中所定义，则抛出 _无效包配置_ 错误。
>    2. 对于 _target_ 的每个属性 _p_，按对象插入顺序，
>       1. 如果 _p_ 等于 _"default"_ 或 _conditions_ 包含 _p_ 的条目，则
>          1. 令 _targetValue_ 为 _target_ 中 _p_ 属性的值。
>          2. 令 _resolved_ 为 **PACKAGE\_TARGET\_RESOLVE**( _packageURL_, _targetValue_, _patternMatch_, _isImports_, _conditions_) 的结果。
>          3. 如果 _resolved_ 等于 **undefined**，继续循环。
>          4. 返回 _resolved_。
>    3. 返回 **undefined**。
> 3. 否则，如果 _target_ 是数组，则
>    1. 如果 \_target.length 为零，返回 **null**。
>    2. 对于 _target_ 中的每个项目 _targetValue_，执行
>       1. 令 _resolved_ 为 **PACKAGE\_TARGET\_RESOLVE**( _packageURL_, _targetValue_, _patternMatch_, _isImports_, _conditions_) 的结果，在任何 _无效包目标_ 错误时继续循环。
>       2. 如果 _resolved_ 是 **undefined**，继续循环。
>       3. 返回 _resolved_。
>    3. 返回或抛出最后一个回退解析 **null** 返回或错误。
> 4. 否则，如果 _target_ 是 _null_，返回 **null**。
> 5. 否则抛出 _无效包目标_ 错误。

**ESM\_FILE\_FORMAT**(_url_)

> 1. 断言：_url_ 对应于现有文件。
> 2. 如果 _url_ 以 _".mjs"_ 结尾，则
>    1. 返回 _"module"_。
> 3. 如果 _url_ 以 _".cjs"_ 结尾，则
>    1. 返回 _"commonjs"_。
> 4. 如果 _url_ 以 _".json"_ 结尾，则
>    1. 返回 _"json"_。
> 5. 如果 _url_ 以 _".wasm"_ 结尾，则
>    1. 返回 _"wasm"_。
> 6. 如果启用了 `--experimental-addon-modules` 且 _url_ 以 _".node"_ 结尾，则
>    1. 返回 _"addon"_。
> 7. 令 _packageURL_ 为 **LOOKUP\_PACKAGE\_SCOPE**(_url_) 的结果。
> 8. 令 _pjson_ 为 **READ\_PACKAGE\_JSON**(_packageURL_) 的结果。
> 9. 令 _packageType_ 为 **null**。
> 10. 如果 _pjson?.type_ 是 _"module"_ 或 _"commonjs"_，则
>     1. 将 _packageType_ 设置为 _pjson.type_。
> 11. 如果 _url_ 以 _".js"_ 结尾，则
>     1. 如果 _packageType_ 不是 **null**，则
>        1. 返回 _packageType_。
>     2. 如果 **DETECT\_MODULE\_SYNTAX**(_source_) 的结果为 true，则
>        1. 返回 _"module"_。
>     3. 返回 _"commonjs"_。
> 12. 如果 _url_ 没有任何扩展名，则
>     1. 如果 _packageType_ 是 _"module"_ 且 _url_ 处的文件包含 WebAssembly 模块的 "application/wasm" 内容类型头，则
>        1. 返回 _"wasm"_。
>     2. 如果 _packageType_ 不是 **null**，则
>        1. 返回 _packageType_。
>     3. 如果 **DETECT\_MODULE\_SYNTAX**(_source_) 的结果为 true，则
>        1. 返回 _"module"_。
>     4. 返回 _"commonjs"_。
> 13. 返回 **undefined**（将在加载阶段抛出）。

**LOOKUP\_PACKAGE\_SCOPE**(_url_)

> 1. 令 _scopeURL_ 为 _url_。
> 2. 当 _scopeURL_ 不是文件系统根目录时，
>    1. 将 _scopeURL_ 设置为 _scopeURL_ 的父 URL。
>    2. 如果 _scopeURL_ 以 _"node\_modules"_ 路径段结尾，返回 **null**。
>    3. 令 _pjsonURL_ 为 _scopeURL_ 内 _"package.json"_ 的解析结果。
>    4. 如果 _pjsonURL_ 处的文件存在，则
>       1. 返回 _scopeURL_。
> 3. 返回 **null**。

**READ\_PACKAGE\_JSON**(_packageURL_)

> 1. 令 _pjsonURL_ 为 _packageURL_ 内 _"package.json"_ 的解析结果。
> 2. 如果 _pjsonURL_ 处的文件不存在，则
>    1. 返回 **null**。
> 3. 如果 _packageURL_ 处的文件无法解析为有效的 JSON，则
>    1. 抛出 _无效包配置_ 错误。
> 4. 返回 _pjsonURL_ 处文件的已解析 JSON 源。

**DETECT\_MODULE\_SYNTAX**(_source_)

> 1. 将 _source_ 解析为 ECMAScript 模块。
> 2. 如果解析成功，则
>    1. 如果 _source_ 包含顶层 `await`、静态 `import` 或 `export` 语句，或 `import.meta`，返回 **true**。
>    2. 如果 _source_ 包含任何 CommonJS 包装器变量（`require`、`exports`、`module`、`__filename` 或 `__dirname`）的顶层词法声明（`const`、`let` 或 `class`），则返回 **true**。
> 3. 否则返回 **false**。

### 自定义 ESM 说明符解析算法

[模块自定义钩子][Module customization hooks] 提供了一种自定义 ESM 说明符解析算法的机制。一个为 ESM 说明符提供 CommonJS 样式解析的示例是 [commonjs-extension-resolution-loader][]。

<!-- Note: The cjs-module-lexer link should be kept in-sync with the deps version -->

[6.1.7 Array Index]: https://tc39.es/ecma262/#integer-index
[Addons]: addons.md
[Built-in modules]: modules.md#built-in-modules
[CommonJS]: modules.md
[Determining module system]: packages.md#determining-module-system
[Dynamic `import()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import
[ES Module Integration Proposal for WebAssembly]: https://github.com/webassembly/esm-integration
[Import Attributes]: #import-attributes
[Import Attributes MDN]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import/with
[JSON modules]: #json-modules
[Loading ECMAScript modules using `require()`]: modules.md#loading-ecmascript-modules-using-require
[Module customization hooks]: module.md#customization-hooks
[Node.js Module Resolution And Loading Algorithm]: #resolution-algorithm-specification
[Source Phase Imports]: https://github.com/tc39/proposal-source-phase-imports
[Terminology]: #terminology
[URL]: https://url.spec.whatwg.org/
[WebAssembly JS String Builtins Proposal]: https://github.com/WebAssembly/js-string-builtins
[`"exports"`]: packages.md#exports
[`"type"`]: packages.md#type
[`--input-type`]: cli.md#--input-typetype
[`data:` URLs]: https://developer.mozilla.org/en-US/docs/Web/HTTP/Basics_of_HTTP/Data_URIs
[`export`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/export
[`import()`]: #import-expressions
[`import.meta.dirname`]: #importmetadirname
[`import.meta.filename`]: #importmetafilename
[`import.meta.main`]: #importmetamain
[`import.meta.resolve`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import.meta/resolve
[`import.meta.url`]: #importmetaurl
[`import`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/import
[`module.createRequire()`]: module.md#modulecreaterequirefilename
[`module.syncBuiltinESMExports()`]: module.md#modulesyncbuiltinesmexports
[`package.json`]: packages.md#nodejs-packagejson-field-definitions
[`path.dirname()`]: path.md#pathdirnamepath
[`process.dlopen`]: process.md#processdlopenmodule-filename-flags
[`require(esm)`]: modules.md#loading-ecmascript-modules-using-require
[`url.fileURLToPath()`]: url.md#urlfileurltopathurl-options
[cjs-module-lexer]: https://github.com/nodejs/cjs-module-lexer/tree/2.0.0
[commonjs-extension-resolution-loader]: https://github.com/nodejs/loaders-test/tree/main/commonjs-extension-resolution-loader
[custom https loader]: module.md#import-from-https
[import.meta.resolve]: #importmetaresolvespecifier
[percent-encoded]: url.md#percent-encoding-in-urls
[special scheme]: https://url.spec.whatwg.org/#special-scheme
[status code]: process.md#exit-codes
[the official standard format]: https://tc39.github.io/ecma262/#sec-modules
[url.pathToFileURL]: url.md#urlpathtofileurlpath-options