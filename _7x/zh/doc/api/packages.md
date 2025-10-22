# Modules: Packages

<!--introduced_in=v12.20.0-->

<!-- type=misc -->

<!-- YAML
changes:
  - version:
    - v14.13.0
    - v12.20.0
    pr-url: https://github.com/nodejs/node/pull/34718
    description: Add support for `"exports"` patterns.
  - version:
    - v14.6.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34117
    description: Add package `"imports"` field.
  - version:
    - v13.7.0
    - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/29866
    description: Unflag conditional exports.
  - version:
    - v13.7.0
    - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/31001
    description: Remove the `--experimental-conditional-exports` option. In 12.16.0, conditional exports are still behind `--experimental-modules`.
  - version:
    - v13.6.0
    - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/31002
    description: Unflag self-referencing a package using its name.
  - version: v12.7.0
    pr-url: https://github.com/nodejs/node/pull/28568
    description:
      Introduce `"exports"` `package.json` field as a more powerful alternative
      to the classic `"main"` field.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/26745
    description:
      Add support for ES modules using `.js` file extension via `package.json`
      `"type"` field.
-->

## 简介

一个包是由 `package.json` 文件描述的文件夹树。包由包含 `package.json` 文件的文件夹及其所有子文件夹组成，直到下一个包含另一个 `package.json` 文件的文件夹，或者一个名为 `node_modules` 的文件夹。

本页为编写 `package.json` 文件的包作者提供指导，并作为 Node.js 定义的 [`package.json`][] 字段的参考。

## 确定模块系统

### 简介

当以下情况被传递给 `node` 作为初始输入，或被 `import` 语句或 `import()` 表达式引用时，Node.js 会将它们视为 [ES 模块][]：

* 具有 `.mjs` 扩展名的文件。

* 当最近的父级 `package.json` 文件包含顶级 [`"type"`][] 字段且值为 `"module"` 时，具有 `.js` 扩展名的文件。

* 通过 `--eval` 参数传递的字符串，或通过 `STDIN` 管道传输给 `node` 的字符串，且带有 `--input-type=module` 标志。

* 包含只能作为 [ES 模块][] 成功解析的语法的代码，例如 `import` 或 `export` 语句或 `import.meta`，但没有明确的标记说明应如何解释。明确的标记包括 `.mjs` 或 `.cjs` 扩展名、`package.json` `"type"` 字段（值为 `"module"` 或 `"commonjs"`）或 `--input-type` 标志。动态 `import()` 表达式在 CommonJS 或 ES 模块中都受支持，不会强制文件被视为 ES 模块。参见 [语法检测][]。

当以下情况被传递给 `node` 作为初始输入，或被 `import` 语句或 `import()` 表达式引用时，Node.js 会将它们视为 [CommonJS][]：

* 具有 `.cjs` 扩展名的文件。

* 当最近的父级 `package.json` 文件包含顶级字段 [`"type"`][] 且值为 `"commonjs"` 时，具有 `.js` 扩展名的文件。

* 通过 `--eval` 或 `--print` 参数传递的字符串，或通过 `STDIN` 管道传输给 `node` 的字符串，且带有 `--input-type=commonjs` 标志。

* 没有父级 `package.json` 文件，或者最近的父级 `package.json` 文件缺少 `type` 字段，并且代码可以作为 CommonJS 成功求值的 `.js` 扩展名文件。换句话说，Node.js 会首先尝试将这些“模糊”文件作为 CommonJS 运行，如果作为 CommonJS 求值失败（因为解析器找到了 ES 模块语法），则会重新尝试将它们作为 ES 模块求值。

在“模糊”文件中编写 ES 模块语法会产生性能成本，因此鼓励作者尽可能明确。特别是，包作者应始终在其 `package.json` 文件中包含 [`"type"`][] 字段，即使包中的所有源代码都是 CommonJS。明确指定包的 `type` 将使包在未来 Node.js 的默认类型发生变化时具有前瞻性，并且也会使构建工具和加载器更容易确定应如何解释包中的文件。

### 语法检测

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

> Stability: 1.2 - Release candidate

Node.js 将检查模糊输入的源代码，以确定其是否包含 ES 模块语法；如果检测到此类语法，输入将被视为 ES 模块。

模糊输入定义为：

* 具有 `.js` 扩展名或无扩展名的文件；并且没有控制性 `package.json` 文件或缺少 `type` 字段的控制性 `package.json` 文件。
* 未指定 `--input-type` 时的字符串输入（`--eval` 或 `STDIN`）。

ES 模块语法定义为在作为 CommonJS 求值时会抛出错误的语法。这包括以下内容：

* `import` 语句（但 _不是_ `import()` 表达式，它们在 CommonJS 中有效）。
* `export` 语句。
* `import.meta` 引用。
* 模块顶层的 `await`。
* CommonJS 包装器变量（`require`、`module`、`exports`、`__dirname`、`__filename`）的词法重新声明。

### 模块加载器

Node.js 有两个系统用于解析说明符和加载模块。

CommonJS 模块加载器：

* 它是完全同步的。
* 负责处理 `require()` 调用。
* 它是可猴子补丁的。
* 支持 [文件夹作为模块][]。
* 解析说明符时，如果没有找到精确匹配，它会尝试添加扩展名（`.js`、`.json`，最后是 `.node`），然后尝试解析 [文件夹作为模块][]。
* 它将 `.json` 视为 JSON 文本文件。
* `.node` 文件被解释为使用 `process.dlopen()` 加载的编译插件模块。
* 它将所有缺少 `.json` 或 `.node` 扩展名的文件视为 JavaScript 文本文件。
* 如果模块图是同步的（即不包含顶级 `await`），它只能用于 [从 CommonJS 模块加载 ECMAScript 模块][]。当用于加载不是 ECMAScript 模块的 JavaScript 文本文件时，该文件将作为 CommonJS 模块加载。

ECMAScript 模块加载器：

* 它是异步的，除非用于为 `require()` 加载模块。
* 负责处理 `import` 语句和 `import()` 表达式。
* 不可猴子补丁，可以使用 [加载器钩子][] 进行自定义。
* 不支持文件夹作为模块，必须完全指定目录索引（例如 `'./startup/index.js'`）。
* 不进行扩展名搜索。当说明符是相对或绝对文件 URL 时，必须提供文件扩展名。
* 可以加载 JSON 模块，但需要导入类型属性。
* 对于 JavaScript 文本文件，只接受 `.js`、`.mjs` 和 `.cjs` 扩展名。
* 可用于加载 JavaScript CommonJS 模块。此类模块通过 `cjs-module-lexer` 传递，以尝试识别命名导出（如果可以通过静态分析确定）。导入的 CommonJS 模块的 URL 会转换为绝对路径，然后通过 CommonJS 模块加载器加载。

### `package.json` 和文件扩展名

在包内，[`package.json`][] 的 [`"type"`][] 字段定义了 Node.js 应如何解释 `.js` 文件。如果 `package.json` 文件没有 `"type"` 字段，则 `.js` 文件被视为 [CommonJS][]。

`package.json` 的 `"type"` 值为 `"module"` 告诉 Node.js 将该包内的 `.js` 文件解释为使用 [ES 模块][] 语法。

`"type"` 字段不仅适用于初始入口点（`node my-app.js`），也适用于 `import` 语句和 `import()` 表达式引用的文件。

```js
// my-app.js，被视为 ES 模块，因为同一文件夹中有一个 package.json
// 文件，其中包含 "type": "module"。

import './startup/init.js';
// 作为 ES 模块加载，因为 ./startup 不包含 package.json 文件，
// 因此继承上一级的 "type" 值。

import 'commonjs-package';
// 作为 CommonJS 加载，因为 ./node_modules/commonjs-package/package.json
// 缺少 "type" 字段或包含 "type": "commonjs"。

import './node_modules/commonjs-package/index.js';
// 作为 CommonJS 加载，因为 ./node_modules/commonjs-package/package.json
// 缺少 "type" 字段或包含 "type": "commonjs"。
```

以 `.mjs` 结尾的文件总是作为 [ES 模块][] 加载，无论最近的父级 `package.json` 如何。

以 `.cjs` 结尾的文件总是作为 [CommonJS][] 加载，无论最近的父级 `package.json` 如何。

```js
import './legacy-file.cjs';
// 作为 CommonJS 加载，因为 .cjs 总是作为 CommonJS 加载。

import 'commonjs-package/src/index.mjs';
// 作为 ES 模块加载，因为 .mjs 总是作为 ES 模块加载。
```

`.mjs` 和 `.cjs` 扩展名可用于在同一包中混合类型：

* 在 `"type": "module"` 包中，可以通过使用 `.cjs` 扩展名命名特定文件来指示 Node.js 将其解释为 [CommonJS][]（因为在此类包中，`.js` 和 `.mjs` 文件都被视为 ES 模块）。

* 在 `"type": "commonjs"` 包中，可以通过使用 `.mjs` 扩展名命名特定文件来指示 Node.js 将其解释为 [ES 模块][]（因为在此类包中，`.js` 和 `.cjs` 文件都被视为 CommonJS）。

### `--input-type` 标志

<!-- YAML
added: v12.0.0
-->

当设置 `--input-type=module` 标志时，通过 `--eval`（或 `-e`）参数传递的字符串，或通过 `STDIN` 管道传输给 `node` 的字符串，将被视为 [ES 模块][]。

```bash
node --input-type=module --eval "import { sep } from 'node:path'; console.log(sep);"

echo "import { sep } from 'node:path'; console.log(sep);" | node --input-type=module
```

为了完整性，还有 `--input-type=commonjs`，用于明确将字符串输入作为 CommonJS 运行。如果未指定 `--input-type`，这是默认行为。

## 包入口点

在包的 `package.json` 文件中，有两个字段可以定义包的入口点：[`"main"`][] 和 [`"exports"`][]。这两个字段都适用于 ES 模块和 CommonJS 模块入口点。

[`"main"`][] 字段在 Node.js 的所有版本中都受支持，但其功能有限：它只定义包的主入口点。

[`"exports"`][] 提供了 [`"main"`][] 的现代替代方案，允许定义多个入口点，支持环境之间的条件入口解析，并 **防止使用 [`"exports"`][] 中定义的入口点之外的任何其他入口点**。这种封装使模块作者能够明确定义其包的公共接口。

对于针对当前支持的 Node.js 版本的新包，推荐使用 [`"exports"`][] 字段。对于支持 Node.js 10 及以下版本的包，[`"main"`][] 字段是必需的。如果同时定义了 [`"exports"`][] 和 [`"main"`][]，在受支持的 Node.js 版本中，[`"exports"`][] 字段优先于 [`"main"`][]。

[条件导出][] 可以在 [`"exports"`][] 中使用，以根据环境定义不同的包入口点，包括包是通过 `require` 还是通过 `import` 引用。有关在单个包中同时支持 CommonJS 和 ES 模块的更多信息，请参阅 [双 CommonJS/ES 模块包部分][]。

现有包引入 [`"exports"`][] 字段将阻止包的消费者使用任何未定义的入口点，包括 [`package.json`][]（例如 `require('your-package/package.json')`）。**这很可能是一个破坏性变更。**

为了使 [`"exports"`][] 的引入非破坏性，请确保导出每个先前受支持的入口点。最好明确指定入口点，以便明确定义包的公共 API。例如，一个先前导出 `main`、`lib`、`feature` 和 `package.json` 的项目可以使用以下 `package.exports`：

```json
{
  "name": "my-package",
  "exports": {
    ".": "./lib/index.js",
    "./lib": "./lib/index.js",
    "./lib/index": "./lib/index.js",
    "./lib/index.js": "./lib/index.js",
    "./feature": "./feature/index.js",
    "./feature/index": "./feature/index.js",
    "./feature/index.js": "./feature/index.js",
    "./package.json": "./package.json"
  }
}
```

或者，项目可以选择使用导出模式导出整个文件夹，包括带扩展名和不带扩展名的子路径：

```json
{
  "name": "my-package",
  "exports": {
    ".": "./lib/index.js",
    "./lib": "./lib/index.js",
    "./lib/*": "./lib/*.js",
    "./lib/*.js": "./lib/*.js",
    "./feature": "./feature/index.js",
    "./feature/*": "./feature/*.js",
    "./feature/*.js": "./feature/*.js",
    "./package.json": "./package.json"
  }
}
```

通过以上方式为任何次要包版本提供向后兼容性，包的下一个主要变更可以适当地将导出限制为仅暴露的特定功能导出：

```json
{
  "name": "my-package",
  "exports": {
    ".": "./lib/index.js",
    "./feature/*.js": "./feature/*.js",
    "./feature/internal/*": null
  }
}
```

### 主入口点导出

编写新包时，建议使用 [`"exports"`][] 字段：

```json
{
  "exports": "./index.js"
}
```

当定义了 [`"exports"`][] 字段时，包的所有子路径都被封装，导入器不再可用。例如，`require('pkg/subpath.js')` 会抛出 [`ERR_PACKAGE_PATH_NOT_EXPORTED`][] 错误。

这种导出的封装为工具和处理包的 semver 升级提供了更可靠的包接口保证。它不是强封装，因为直接 require 包的任何绝对子路径，例如 `require('/path/to/node_modules/pkg/subpath.js')`，仍然会加载 `subpath.js`。

所有当前支持的 Node.js 版本和现代构建工具都支持 `"exports"` 字段。对于使用旧版本 Node.js 或相关构建工具的项目，可以通过同时包含指向同一模块的 `"main"` 字段和 `"exports"` 来实现兼容性：

```json
{
  "main": "./index.js",
  "exports": "./index.js"
}
```

### 子路径导出

<!-- YAML
added: v12.7.0
-->

使用 [`"exports"`][] 字段时，可以通过将主入口点视为 `"."` 子路径来定义自定义子路径以及主入口点：

```json
{
  "exports": {
    ".": "./index.js",
    "./submodule.js": "./src/submodule.js"
  }
}
```

现在，只有 [`"exports"`][] 中定义的子路径可以被消费者导入：

```js
import submodule from 'es-module-package/submodule.js';
// 加载 ./node_modules/es-module-package/src/submodule.js
```

而其他子路径将报错：

```js
import submodule from 'es-module-package/private-module.js';
// 抛出 ERR_PACKAGE_PATH_NOT_EXPORTED
```

#### 子路径中的扩展名

包作者应在其导出中提供带扩展名（`import 'pkg/subpath.js'`）或不带扩展名（`import 'pkg/subpath'`）的子路径。这确保了每个导出模块只有一个子路径，以便所有依赖项导入相同一致的说明符，保持包契约对消费者清晰，并简化包子路径补全。

传统上，包倾向于使用无扩展名风格，其优点是可读性和隐藏包内文件的真实路径。

随着 [导入映射][] 现在为浏览器和其他 JavaScript 运行时中的包解析提供了标准，使用无扩展名风格可能导致导入映射定义臃肿。显式文件扩展名可以通过使导入映射能够利用 [包文件夹映射][] 来映射多个子路径（如果可能），而不是每个包子路径导出都有一个单独的映射条目，从而避免此问题。这也反映了在相对和绝对导入说明符中使用 [完整说明符路径][] 的要求。

#### 导出目标的路径规则和验证

在 [`"exports"`][] 字段中定义路径作为目标时，Node.js 强制执行若干规则以确保安全性、可预测性和适当的封装。理解这些规则对于发布包的作者至关重要。

##### 目标必须是相对 URL

[`"exports"`][] 映射中的所有目标路径（与导出键关联的值）必须是以 `./` 开头的相对 URL 字符串。

```json
// package.json
{
  "name": "my-package",
  "exports": {
    ".": "./dist/main.js",          // 正确
    "./feature": "./lib/feature.js", // 正确
    // "./origin-relative": "/dist/main.js", // 错误：必须以 ./ 开头
    // "./absolute": "file:///dev/null", // 错误：必须以 ./ 开头
    // "./outside": "../common/util.js" // 错误：必须以 ./ 开头
  }
}
```

此行为的原因包括：

* **安全性：** 防止从包自身目录外部导出任意文件。
* **封装：** 确保所有导出路径都相对于包根目录解析，使包自包含。

##### 不允许路径遍历或无效段

导出目标不得解析到包根目录之外的位置。此外，在初始 `./` 之后的 `target` 字符串中以及替换到目标模式中的任何 `subpath` 部分中，通常不允许使用路径段，如 `.`（单点）、`..`（双点）或 `node_modules`（及其 URL 编码等效项）。

```json
// package.json
{
  "name": "my-package",
  "exports": {
    // ".": "./dist/../../elsewhere/file.js", // 无效：路径遍历
    // ".": "././dist/main.js",             // 无效：包含 "." 段
    // ".": "./dist/../dist/main.js",       // 无效：包含 ".." 段
    // "./utils/./helper.js": "./utils/helper.js" // 键包含无效段
  }
}
```

### 导出语法糖

<!-- YAML
added: v12.11.0
-->

如果 `"."` 导出是唯一的导出，[`"exports"`][] 字段为此情况提供了语法糖，即直接的 [`"exports"`][] 字段值。

```json
{
  "exports": {
    ".": "./index.js"
  }
}
```

可以写成：

```json
{
  "exports": "./index.js"
}
```

### 子路径导入

<!-- YAML
added:
  - v14.6.0
  - v12.19.0
-->

除了 [`"exports"`][] 字段，还有一个包 `"imports"` 字段用于创建私有映射，这些映射仅适用于从包本身内部的导入说明符。

`"imports"` 字段中的条目必须始终以 `#` 开头，以确保它们与外部包说明符区分开。

例如，导入字段可用于为内部模块获得条件导出的好处：

```json
// package.json
{
  "imports": {
    "#dep": {
      "node": "dep-node-native",
      "default": "./dep-polyfill.js"
    }
  },
  "dependencies": {
    "dep-node-native": "^1.0.0"
  }
}
```

其中 `import '#dep'` 不会获取外部包 `dep-node-native` 的解析（包括其导出），而是在其他环境中获取相对于包的本地文件 `./dep-polyfill.js`。

与 `"exports"` 字段不同，`"imports"` 字段允许映射到外部包。

导入字段的解析规则在其他方面与导出字段类似。

### 子路径模式

<!-- YAML
added:
  - v14.13.0
  - v12.20.0
changes:
  - version:
    - v16.10.0
    - v14.19.0
    pr-url: https://github.com/nodejs/node/pull/40041
    description: Support pattern trailers in "imports" field.
  - version:
    - v16.9.0
    - v14.19.0
    pr-url: https://github.com/nodejs/node/pull/39635
    description: Support pattern trailers.
-->

对于具有少量导出或导入的包，我们建议显式列出每个导出的子路径条目。但对于具有大量子路径的包，这可能导致 `package.json` 臃肿和维护问题。

对于这些用例，可以使用子路径导出模式：

```json
// ./node_modules/es-module-package/package.json
{
  "exports": {
    "./features/*.js": "./src/features/*.js"
  },
  "imports": {
    "#internal/*.js": "./src/internal/*.js"
  }
}
```

**`*` 映射公开嵌套子路径，因为它只是一种字符串替换语法。**

右侧的所有 `*` 实例将被替换为此值，包括它包含任何 `/` 分隔符。

```js
import featureX from 'es-module-package/features/x.js';
// 加载 ./node_modules/es-module-package/src/features/x.js

import featureY from 'es-module-package/features/y/y.js';
// 加载 ./node_modules/es-module-package/src/features/y/y.js

import internalZ from '#internal/z.js';
// 加载 ./node_modules/es-module-package/src/internal/z.js
```

这是一种直接的静态匹配和替换，没有对文件扩展名进行特殊处理。在映射两侧包含 `"*.js"` 将包的公开导出限制为仅 JS 文件。

导出的静态可枚举属性通过导出模式得以维护，因为可以通过将右侧目标模式视为针对包内文件列表的 `**`  glob 来确定包的单个导出。由于 `node_modules` 路径在导出目标中被禁止，此扩展仅依赖于包自身的文件。

要从模式中排除私有子文件夹，可以使用 `null` 目标：

```json
// ./node_modules/es-module-package/package.json
{
  "exports": {
    "./features/*.js": "./src/features/*.js",
    "./features/private-internal/*": null
  }
}
```

```js
import featureInternal from 'es-module-package/features/private-internal/m.js';
// 抛出：ERR_PACKAGE_PATH_NOT_EXPORTED

import featureX from 'es-module-package/features/x.js';
// 加载 ./node_modules/es-module-package/src/features/x.js
```

### 条件导出

<!-- YAML
added:
  - v13.2.0
  - v12.16.0
changes:
  - version:
    - v13.7.0
    - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/31001
    description: Unflag conditional exports.
-->

条件导出提供了一种根据某些条件映射到不同路径的方法。它们同时支持 CommonJS 和 ES 模块导入。

例如，一个想要为 `require()` 和 `import` 提供不同 ES 模块导出的包可以这样写：

```json
// package.json
{
  "exports": {
    "import": "./index-module.js",
    "require": "./index-require.cjs"
  },
  "type": "module"
}
```

Node.js 实现了以下条件，按从最具体到最不具体的顺序列出，因为条件应按此顺序定义：

* `"node-addons"` - 类似于 `"node"`，匹配任何 Node.js 环境。此条件可用于提供使用本地 C++ 插件的入口点，而不是更通用且不依赖本地插件的入口点。此条件可以通过 [`--no-addons` 标志][] 禁用。
* `"node"` - 匹配任何 Node.js 环境。可以是 CommonJS 或 ES 模块文件。_在大多数情况下，明确指明 Node.js 平台是不必要的。_
* `"import"` - 当包通过 `import` 或 `import()` 加载，或通过 ECMAScript 模块加载器的任何顶级导入或解析操作时匹配。无论目标文件的模块格式如何，都适用。_始终与 `"require"` 互斥。_
* `"require"` - 当包通过 `require()` 加载时匹配。引用的文件应该可以使用 `require()` 加载，尽管无论目标文件的模块格式如何，条件都会匹配。预期格式包括 CommonJS、JSON、本地插件和 ES 模块。_始终与 `"import"` 互斥。_
* `"module-sync"` - 无论包是通过 `import`、`import()` 还是 `require()` 加载，都匹配。格式预期是不在其模块图中包含顶级 await 的 ES 模块 - 如果包含，当模块被 `require()` 时，将抛出 `ERR_REQUIRE_ASYNC_MODULE`。
* `"default"` - 始终匹配的通用回退。可以是 CommonJS 或 ES 模块文件。_此条件应始终放在最后。_

在 [`"exports"`][] 对象中，键的顺序很重要。在条件匹配期间，较早的条目具有更高的优先级，并优先于较晚的条目。_一般规则是条件在对象顺序中应从最具体到最不具体_。

使用 `"import"` 和 `"require"` 条件可能导致一些隐患，这些隐患在 [双 CommonJS/ES 模块包部分][] 中有进一步解释。

`"node-addons"` 条件可用于提供使用本地 C++ 插件的入口点。但是，此条件可以通过 [`--no-addons` 标志][] 禁用。使用 `"node-addons"` 时，建议将 `"default"` 视为提供更通用入口点的增强功能，例如使用 WebAssembly 而不是本地插件。

条件导出也可以扩展到导出子路径，例如：

```json
{
  "exports": {
    ".": "./index.js",
    "./feature.js": {
      "node": "./feature-node.js",
      "default": "./feature.js"
    }
  }
}
```

定义了一个包，其中 `require('pkg/feature.js')` 和 `import 'pkg/feature.js'` 可以在 Node.js 和其他 JS 环境之间提供不同的实现。

使用环境分支时，始终尽可能包含 `"default"` 条件。提供 `"default"` 条件确保任何未知的 JS 环境都能够使用此通用实现，这有助于避免这些 JS 环境为了支持具有条件导出的包而必须伪装成现有环境。因此，使用 `"node"` 和 `"default"` 条件分支通常优于使用 `"node"` 和 `"browser"` 条件分支。

### 嵌套条件

除了直接映射，Node.js 还支持嵌套条件对象。

例如，定义一个仅在 Node.js 中具有双模式入口点但不适用于浏览器的包：

```json
{
  "exports": {
    "node": {
      "import": "./feature-node.mjs",
      "require": "./feature-node.cjs"
    },
    "default": "./feature.mjs"
  }
}
```

条件的匹配顺序与平面条件相同。如果嵌套条件没有任何映射，它将继续检查父条件的剩余条件。通过这种方式，嵌套条件的行为类似于嵌套的 JavaScript `if` 语句。

### 解析用户条件

<!-- YAML
added:
  - v14.9.0
  - v12.19.0
-->

运行 Node.js 时，可以使用 `--conditions` 标志添加自定义用户条件：

```bash
node --conditions=development index.js
```

然后将在包导入和导出中解析 `"development"` 条件，同时根据需要解析现有的 `"node"`、`"node-addons"`、`"default"`、`"import"` 和 `"require"` 条件。

可以使用重复标志设置任意数量的自定义条件。

典型条件应仅包含字母数字字符，必要时使用 ":"、"-" 或 "=" 作为分隔符。其他任何内容可能在 node 之外遇到兼容性问题。

在 node 中，条件限制很少，但具体包括：

1. 它们必须包含至少一个字符。
2. 它们不能以 "." 开头，因为它们可能出现在也允许相对路径的位置。
3. 它们不能包含 ","，因为它们可能被某些 CLI 工具解析为逗号分隔列表。
4. 它们不能是整数属性键，如 "10"，因为这可能对 JS 对象的属性键排序产生意外影响。

### 社区条件定义

除了 Node.js 核心 [实现的](#conditional-exports) `"import"`、`"require"`、`"node"`、`"module-sync"`、`"node-addons"` 和 `"default"` 条件之外的条件字符串默认被忽略。

其他平台可能实现其他条件，用户条件可以在 Node.js 中通过 [`--conditions` / `-C` 标志][] 启用。

由于自定义包条件需要清晰的定义以确保正确使用，下面提供了一个常见已知包条件及其严格定义的列表，以协助生态系统协调。

* `"types"` - 可以被类型系统用于解析给定导出的类型文件。_此条件应始终首先包含。_
* `"browser"` - 任何 Web 浏览器环境。
* `"development"` - 可用于定义仅开发环境的入口点，例如在开发模式下运行时提供额外的调试上下文，如更好的错误消息。_必须始终与 `"production"` 互斥。_
* `"production"` - 可用于定义生产环境入口点。_必须始终与 `"development"` 互斥。_

对于其他运行时，平台特定的条件键定义由 [WinterCG][] 在 [运行时键][] 提案规范中维护。

新的条件定义可以通过为此 [Node.js 文档部分][] 创建拉取请求添加到本列表。在此列出新条件定义的要求是：

* 定义应对所有实现者清晰明确。
* 应明确说明需要此条件的原因。
* 应有足够的现有实现使用。
* 条件名称不应与另一个条件定义或广泛使用的条件冲突。
* 条件定义的列表应提供生态系统其他方式无法实现的协调好处。例如，对于公司特定或应用程序特定的条件，情况可能不一定如此。
* 条件应使 Node.js 用户期望它在 Node.js 核心文档中。`"types"` 条件是一个很好的例子：它并不真正属于 [运行时键][] 提案，但很适合放在 Node.js 文档中。

上述定义可能会在适当的时候移至专用的条件注册表。

### 使用名称自引用包

<!-- YAML
added:
  - v13.1.0
  - v12.16.0
changes:
  - version:
    - v13.6.0
    - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/31002
    description: Unflag self-referencing a package using its name.
-->

在包内，包 `package.json` 中定义的 [`"exports"`][] 字段的值可以通过包的名称引用。例如，假设 `package.json` 是：

```json
// package.json
{
  "name": "a-package",
  "exports": {
    ".": "./index.mjs",
    "./foo.js": "./foo.js"
  }
}
```

然后 _该包中_ 的任何模块都可以引用包本身的导出：

```js
// ./a-module.mjs
import { something } from 'a-package'; // 从 ./index.mjs 导入 "something"。
```

仅当 `package.json` 具有 [`"exports"`][] 时，自引用才可用，并且将仅允许导入该 [`"exports"`][]（在 `package.json` 中）允许的内容。因此，给定先前的包，下面的代码将生成运行时错误：

```js
// ./another-module.mjs

// 从 ./m.mjs 导入 "another"。失败，因为
// "package.json" "exports" 字段
// 未提供名为 "./m.mjs" 的导出。
import { another } from 'a-package/m.mjs';
```

自引用在 ES 模块和 CommonJS 模块中使用 `require` 时也可用。例如，此代码也将工作：

```cjs
// ./a-module.js
const { something } = require('a-package/foo.js'); // 从 ./foo.js 加载。
```

最后，自引用也适用于作用域包。例如，此代码也将工作：

```json
// package.json
{
  "name": "@my/package",
  "exports": "./index.js"
}
```

```cjs
// ./index.js
module.exports = 42;
```

```cjs
// ./other.js
console.log(require('@my/package'));
```

```console
$ node other.js
42
```

## 双 CommonJS/ES 模块包

有关详细信息，请参阅 [包示例仓库][]。

## Node.js `package.json` 字段定义

本节描述了 Node.js 运行时使用的字段。其他工具（如 [npm](https://docs.npmjs.com/cli/v8/configuring-npm/package-json)）使用其他字段，这些字段被 Node.js 忽略且未在此记录。

`package.json` 文件中以下字段在 Node.js 中使用：

* [`"name"`][] - 在包内使用命名导入时相关。也被包管理器用作包的名称。
* [`"main"`][] - 如果未指定 exports，则在加载包时的默认模块，以及在引入 exports 之前的 Node.js 版本中。
* [`"type"`][] - 包类型，确定是将 `.js` 文件加载为 CommonJS 还是 ES 模块。
* [`"exports"`][] - 包导出和条件导出。存在时，限制可以从包内加载哪些子模块。
* [`"imports"`][] - 包导入，供包本身内的模块使用。

### `"name"`

<!-- YAML
added:
  - v13.1.0
  - v12.16.0
changes:
  - version:
    - v13.6.0
    - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/31002
    description: Remove the `--experimental-resolve-self` option.
-->

* 类型：{string}

```json
{
  "name": "package-name"
}
```

`"name"` 字段定义了包的名称。发布到 _npm_ 注册表需要一个满足 [特定要求](https://docs.npmjs.com/files/package.json#name) 的名称。

`"name"` 字段可以与 [`"exports"`][] 字段一起使用，以 [自引用][] 包使用其名称。

### `"main"`

<!-- YAML
added: v0.4.0
-->

* 类型：{string}

```json
{
  "main": "./index.js"
}
```

`"main"` 字段定义了通过 `node_modules` 查找按名称导入包时的入口点。其值是一个路径。

当包具有 [`"exports"`][] 字段时，在按名称导入包时，此字段将优先于 `"main"` 字段。

它还定义了当 [通过 `require()` 加载包目录](modules.md#folders-as-modules) 时使用的脚本。

```cjs
// 这将解析为 ./path/to/directory/index.js。
require('./path/to/directory');
```

### `"type"`

<!-- YAML
added: v12.0.0
changes:
  - version:
    - v13.2.0
    - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/29866
    description: Unflag `--experimental-modules`.
-->

* 类型：{string}

`"type"` 字段定义了 Node.js 用于所有 `.js` 文件的模块格式，这些文件以该 `package.json` 文件作为其最近的父级。

当最近的父级 `package.json` 文件包含顶级字段 `"type"` 且值为 `"module"` 时，以 `.js` 结尾的文件将作为 ES 模块加载。

最近的父级 `package.json` 定义为在当前文件夹、该文件夹的父级等中搜索时找到的第一个 `package.json`，直到到达 node\_modules 文件夹或卷根目录。

```json
// package.json
{
  "type": "module"
}
```

```bash
# 在与前述 package.json 相同的文件夹中
node my-app.js # 作为 ES 模块运行
```

如果最近的父级 `package.json` 缺少 `"type"` 字段，或包含 `"type": "commonjs"`，则 `.js` 文件被视为 [CommonJS][]。如果到达卷根目录且未找到 `package.json`，则 `.js` 文件被视为 [CommonJS][]。

如果最近的父级 `package.json` 包含 `"type": "module"`，则 `.js` 文件的 `import` 语句被视为 ES 模块。

```js
// my-app.js，与上述示例相同的一部分
import './startup.js'; // 由于 package.json 作为 ES 模块加载
```

无论 `"type"` 字段的值如何，`.mjs` 文件始终被视为 ES 模块，`.cjs` 文件始终被视为 CommonJS。

### `"exports"`

<!-- YAML
added: v12.7.0
changes:
  - version:
    - v14.13.0
    - v12.20.0
    pr-url: https://github.com/nodejs/node/pull/34718
    description: Add support for `"exports"` patterns.
  - version:
    - v13.7.0
    - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/29866
    description: Unflag conditional exports.
  - version:
    - v13.7.0
    - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/31008
    description: Implement logical conditional exports ordering.
  - version:
    - v13.7.0
    - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/31001
    description: Remove the `--experimental-conditional-exports` option. In 12.16.0, conditional exports are still behind `--experimental-modules`.
  - version:
    - v13.2.0
    - v12.16.0
    pr-url: https://github.com/nodejs/node/pull/29978
    description: Implement conditional exports.
-->

* 类型：{Object|string|string\[]}

```json
{
  "exports": "./index.js"
}
```

`"exports"` 字段允许定义包的 [入口点][]，当通过 `node_modules` 查找或通过 [自引用][] 其自身名称加载时。它在 Node.js 12+ 中受支持，作为 [`"main"`][] 的替代方案，可以支持定义 [子路径导出][] 和 [条件导出][]，同时封装内部未导出模块。

[条件导出][] 也可以在 `"exports"` 中使用，以根据环境定义不同的包入口点，包括包是通过 `require` 还是通过 `import` 引用。

`"exports"` 中定义的所有路径必须是以 `./` 开头的相对文件 URL。

### `"imports"`

<!-- YAML
added:
 - v14.6.0
 - v12.19.0
-->

* 类型：{Object}

```json
// package.json
{
  "imports": {
    "#dep": {
      "node": "dep-node-native",
      "default": "./dep-polyfill.js"
    }
  },
  "dependencies": {
    "dep-node-native": "^1.0.0"
  }
}
```

导入字段中的条目必须是以 `#` 开头的字符串。

包导入允许映射到外部包。

此字段定义了当前包的 [子路径导入][]。

[CommonJS]: modules.md
[条件导出]: #conditional-exports
[ES module]: esm.md
[ES 模块]: esm.md
[Node.js 文档部分]: https://github.com/nodejs/node/blob/HEAD/doc/api/packages.md#conditions-definitions
[运行时键]: https://runtime-keys.proposal.wintercg.org/
[语法检测]: #syntax-detection
[WinterCG]: https://wintercg.org/
[`"exports"`]: #exports
[`"imports"`]: #imports
[`"main"`]: #main
[`"name"`]: #name
[`"type"`]: #type
[`--conditions` / `-C` 标志]: #resolving-user-conditions
[`--no-addons` 标志]: cli.md#--no-addons
[`ERR_PACKAGE_PATH_NOT_EXPORTED`]: errors.md#err_package_path_not_exported
[`package.json`]: #nodejs-packagejson-field-definitions
[入口点]: #package-entry-points
[文件夹作为模块]: modules.md#folders-as-modules
[导入映射]: https://github.com/WICG/import-maps
[从 CommonJS 模块加载 ECMAScript 模块]: modules.md#loading-ecmascript-modules-using-require
[加载器钩子]: esm.md#loaders
[包文件夹映射]: https://github.com/WICG/import-maps#packages-via-trailing-slashes
[自引用]: #self-referencing-a-package-using-its-name
[子路径导出]: #subpath-exports
[子路径导入]: #subpath-imports
[双 CommonJS/ES 模块包部分]: #dual-commonjses-module-packages
[完整说明符路径]: esm.md#mandatory-file-extensions
[包示例仓库]: https://github.com/nodejs/package-examples