# Modules: CommonJS modules

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!--name=module-->

CommonJS 模块是 Node.js 打包 JavaScript 代码的原始方式。Node.js 也支持浏览器和其他 JavaScript 运行时使用的 [ECMAScript modules][] 标准。

在 Node.js 中，每个文件被视为一个独立的模块。例如，考虑一个名为 `foo.js` 的文件：

```js
const circle = require('./circle.js');
console.log(`The area of a circle of radius 4 is ${circle.area(4)}`);
```

在第一行，`foo.js` 加载了与 `foo.js` 在同一目录下的模块 `circle.js`。

以下是 `circle.js` 的内容：

```js
const { PI } = Math;

exports.area = (r) => PI * r ** 2;

exports.circumference = (r) => 2 * PI * r;
```

模块 `circle.js` 导出了函数 `area()` 和 `circumference()`。通过向特殊的 `exports` 对象指定额外的属性，函数和对象被添加到模块的根部。

模块本地的变量将是私有的，因为模块被 Node.js 包装在一个函数中（参见 [模块包装器](#the-module-wrapper)）。在这个例子中，变量 `PI` 对 `circle.js` 是私有的。

`module.exports` 属性可以被赋予一个新值（例如一个函数或对象）。

在以下代码中，`bar.js` 使用了 `square` 模块，该模块导出了一个 Square 类：

```js
const Square = require('./square.js');
const mySquare = new Square(2);
console.log(`The area of mySquare is ${mySquare.area()}`);
```

`square` 模块在 `square.js` 中定义：

```js
// Assigning to exports will not modify module, must use module.exports
module.exports = class Square {
  constructor(width) {
    this.width = width;
  }

  area() {
    return this.width ** 2;
  }
};
```

CommonJS 模块系统在 [`module` 核心模块][] 中实现。

## 启用

<!-- type=misc -->

Node.js 有两个模块系统：CommonJS 模块和 [ECMAScript modules][]。

默认情况下，Node.js 会将以下文件视为 CommonJS 模块：

* 扩展名为 `.cjs` 的文件；

* 扩展名为 `.js` 的文件，且最近的父 `package.json` 文件包含顶级字段 [`"type"`][]，其值为 `"commonjs"`。

* 扩展名为 `.js` 或无扩展名的文件，且最近的父 `package.json` 文件不包含顶级字段 [`"type"`][]，或者在任何父文件夹中没有 `package.json` 文件；除非该文件包含除非被当作 ES 模块评估否则会出错的语法。包作者应包含 [`"type"`][] 字段，即使在所有源文件都是 CommonJS 的包中也是如此。明确说明包的 `type` 将使构建工具和加载器更容易确定包中的文件应如何被解释。

* 扩展名不是 `.mjs`、`.cjs`、`.json`、`.node` 或 `.js` 的文件（当最近的父 `package.json` 文件包含顶级字段 [`"type"`][] 且值为 `"module"` 时，这些文件只有在通过 `require()` 引入时才会被识别为 CommonJS 模块，而不是当它们被用作程序的命令行入口点时）。

更多详情请参见 [确定模块系统][]。

调用 `require()` 总是使用 CommonJS 模块加载器。调用 `import()` 总是使用 ECMAScript 模块加载器。

## 访问主模块

<!-- type=misc -->

当一个文件直接通过 Node.js 运行时，`require.main` 被设置为其 `module`。这意味着可以通过测试 `require.main === module` 来判断一个文件是否被直接运行。

对于文件 `foo.js`，如果通过 `node foo.js` 运行，这将为 `true`，但如果通过 `require('./foo')` 运行，则为 `false`。

当入口点不是 CommonJS 模块时，`require.main` 是 `undefined`，并且主模块无法访问。

## 包管理器技巧

<!-- type=misc -->

Node.js `require()` 函数的语义被设计得足够通用，以支持合理的目录结构。包管理器程序如 `dpkg`、`rpm` 和 `npm` 有望能够在不修改的情况下从 Node.js 模块构建原生包。

下面我们给出一个可能可行的建议目录结构：

假设我们希望在 `/usr/lib/node/<some-package>/<some-version>` 文件夹中存放特定版本包的内容。

包可以相互依赖。为了安装包 `foo`，可能需要安装特定版本的包 `bar`。`bar` 包本身可能有依赖关系，在某些情况下，这些依赖甚至可能冲突或形成循环依赖。

因为 Node.js 会查找它加载的任何模块的 `realpath`（即解析符号链接），然后 [在 `node_modules` 文件夹中查找它们的依赖项](#loading-from-node_modules-folders)，这种情况可以通过以下架构解决：

* `/usr/lib/node/foo/1.2.3/`：`foo` 包的内容，版本 1.2.3。
* `/usr/lib/node/bar/4.3.2/`：`foo` 所依赖的 `bar` 包的内容。
* `/usr/lib/node/foo/1.2.3/node_modules/bar`：指向 `/usr/lib/node/bar/4.3.2/` 的符号链接。
* `/usr/lib/node/bar/4.3.2/node_modules/*`：指向 `bar` 所依赖的包的符号链接。

因此，即使遇到循环，或者存在依赖冲突，每个模块都能够获得一个可以使用的依赖版本。

当 `foo` 包中的代码执行 `require('bar')` 时，它将获得符号链接到 `/usr/lib/node/foo/1.2.3/node_modules/bar` 的版本。然后，当 `bar` 包中的代码调用 `require('quux')` 时，它将获得符号链接到 `/usr/lib/node/bar/4.3.2/node_modules/quux` 的版本。

此外，为了使模块查找过程更加优化，我们可以将包放在 `/usr/lib/node_modules/<name>/<version>` 而不是直接放在 `/usr/lib/node` 中。然后 Node.js 将不会费心在 `/usr/node_modules` 或 `/node_modules` 中查找缺失的依赖项。

为了使模块对 Node.js REPL 可用，可能还需要将 `/usr/lib/node_modules` 文件夹添加到 `$NODE_PATH` 环境变量中。由于使用 `node_modules` 文件夹的模块查找都是相对的，并且基于调用 `require()` 的文件的真实路径，包本身可以位于任何地方。

## 使用 `require()` 加载 ECMAScript 模块

<!-- YAML
added:
  - v22.0.0
  - v20.17.0
changes:
  - version:
    - v23.5.0
    - v22.13.0
    - v20.19.0
    pr-url: https://github.com/nodejs/node/pull/56194
    description: This feature no longer emits an experimental warning by default,
                 though the warning can still be emitted by --trace-require-module.
  - version:
    - v23.0.0
    - v22.12.0
    - v20.19.0
    pr-url: https://github.com/nodejs/node/pull/55085
    description: This feature is no longer behind the `--experimental-require-module` CLI flag.
  - version:
    - v23.0.0
    - v22.12.0
    pr-url: https://github.com/nodejs/node/pull/54563
    description: Support `'module.exports'` interop export in `require(esm)`.
-->

> Stability: 1.2 - Release candidate

`.mjs` 扩展名保留给 [ECMAScript Modules][]。关于哪些文件被解析为 ECMAScript 模块的更多信息，请参见 [确定模块系统][] 部分。

`require()` 仅支持加载满足以下要求的 ECMAScript 模块：

* 模块是完全同步的（不包含顶级 `await`）；并且
* 满足以下条件之一：
  1. 文件具有 `.mjs` 扩展名。
  2. 文件具有 `.js` 扩展名，且最近的 `package.json` 包含 `"type": "module"`。
  3. 文件具有 `.js` 扩展名，最近的 `package.json` 不包含 `"type": "commonjs"`，并且模块包含 ES 模块语法。

如果被加载的 ES 模块满足要求，`require()` 可以加载它并返回 [模块命名空间对象][module namespace object]。在这种情况下，它类似于动态 `import()`，但是同步运行并直接返回命名空间对象。

使用以下 ES 模块：

```mjs
// distance.mjs
export function distance(a, b) { return Math.sqrt((b.x - a.x) ** 2 + (b.y - a.y) ** 2); }
```

```mjs
// point.mjs
export default class Point {
  constructor(x, y) { this.x = x; this.y = y; }
}
```

CommonJS 模块可以使用 `require()` 加载它们：

```cjs
const distance = require('./distance.mjs');
console.log(distance);
// [Module: null prototype] {
//   distance: [Function: distance]
// }

const point = require('./point.mjs');
console.log(point);
// [Module: null prototype] {
//   default: [class Point],
//   __esModule: true,
// }
```

为了与将 ES 模块转换为 CommonJS 的现有工具互操作，这些工具随后可以通过 `require()` 加载真正的 ES 模块，返回的命名空间将包含一个 `__esModule: true` 属性（如果它有 `default` 导出），以便由工具生成的消费代码能够识别真正 ES 模块中的默认导出。如果命名空间已经定义了 `__esModule`，则不会添加此属性。此属性是实验性的，未来可能会更改。它只应由遵循现有生态系统约定将 ES 模块转换为 CommonJS 模块的工具使用。直接以 CommonJS 编写的代码应避免依赖它。

当 ES 模块同时包含命名导出和默认导出时，`require()` 返回的结果是 [模块命名空间对象][module namespace object]，它将默认导出放在 `.default` 属性中，类似于 `import()` 返回的结果。
要自定义 `require(esm)` 直接返回的内容，ES 模块可以使用字符串名称 `"module.exports"` 导出所需的值。

<!-- eslint-disable @stylistic/js/semi -->

```mjs
// point.mjs
export default class Point {
  constructor(x, y) { this.x = x; this.y = y; }
}

// `distance` is lost to CommonJS consumers of this module, unless it's
// added to `Point` as a static property.
export function distance(a, b) { return Math.sqrt((b.x - a.x) ** 2 + (b.y - a.y) ** 2); }
export { Point as 'module.exports' }
```

<!-- eslint-disable node-core/no-duplicate-requires -->

```cjs
const Point = require('./point.mjs');
console.log(Point); // [class Point]

// Named exports are lost when 'module.exports' is used
const { distance } = require('./point.mjs');
console.log(distance); // undefined
```

注意在上面的例子中，当使用 `module.exports` 导出名称时，命名导出将对 CommonJS 消费者丢失。为了允许 CommonJS 消费者继续访问命名导出，模块可以确保默认导出是一个对象，并将命名导出作为属性附加到该对象上。例如，对于上面的例子，`distance` 可以作为静态方法附加到默认导出 `Point` 类上。

<!-- eslint-disable @stylistic/js/semi -->

```mjs
export function distance(a, b) { return Math.sqrt((b.x - a.x) ** 2 + (b.y - a.y) ** 2); }

export default class Point {
  constructor(x, y) { this.x = x; this.y = y; }
  static distance = distance;
}

export { Point as 'module.exports' }
```

<!-- eslint-disable node-core/no-duplicate-requires -->

```cjs
const Point = require('./point.mjs');
console.log(Point); // [class Point]

const { distance } = require('./point.mjs');
console.log(distance); // [Function: distance]
```

如果被 `require()` 的模块包含顶级 `await`，或者它 `import` 的模块图包含顶级 `await`，将会抛出 [`ERR_REQUIRE_ASYNC_MODULE`][]。在这种情况下，用户应使用 [`import()`][] 加载异步模块。

如果启用了 `--experimental-print-required-tla`，Node.js 将在评估之前评估模块，尝试定位顶级 await，并打印它们的位置以帮助用户修复它们，而不是抛出 `ERR_REQUIRE_ASYNC_MODULE`。

使用 `require()` 加载 ES 模块的支持目前是实验性的，可以使用 `--no-experimental-require-module` 禁用。要打印使用此功能的位置，请使用 [`--trace-require-module`][]。

可以通过检查 [`process.features.require_module`][] 是否为 `true` 来检测此功能。

## 整体流程

<!-- type=misc -->

要获取调用 `require()` 时将加载的确切文件名，请使用 `require.resolve()` 函数。

综合以上所有内容，以下是 `require()` 执行的高级算法伪代码：

```text
require(X) from module at path Y
1. If X is a core module,
   a. return the core module
   b. STOP
2. If X begins with '/'
   a. set Y to the file system root
3. If X is equal to '.', or X begins with './', '/' or '../'
   a. LOAD_AS_FILE(Y + X)
   b. LOAD_AS_DIRECTORY(Y + X)
   c. THROW "not found"
4. If X begins with '#'
   a. LOAD_PACKAGE_IMPORTS(X, dirname(Y))
5. LOAD_PACKAGE_SELF(X, dirname(Y))
6. LOAD_NODE_MODULES(X, dirname(Y))
7. THROW "not found"

MAYBE_DETECT_AND_LOAD(X)
1. If X parses as a CommonJS module, load X as a CommonJS module. STOP.
2. Else, if the source code of X can be parsed as ECMAScript module using
  <a href="esm.md#resolver-algorithm-specification">DETECT_MODULE_SYNTAX defined in
  the ESM resolver</a>,
  a. Load X as an ECMAScript module. STOP.
3. THROW the SyntaxError from attempting to parse X as CommonJS in 1. STOP.

LOAD_AS_FILE(X)
1. If X is a file, load X as its file extension format. STOP
2. If X.js is a file,
    a. Find the closest package scope SCOPE to X.
    b. If no scope was found
      1. MAYBE_DETECT_AND_LOAD(X.js)
    c. If the SCOPE/package.json contains "type" field,
      1. If the "type" field is "module", load X.js as an ECMAScript module. STOP.
      2. If the "type" field is "commonjs", load X.js as a CommonJS module. STOP.
    d. MAYBE_DETECT_AND_LOAD(X.js)
3. If X.json is a file, load X.json to a JavaScript Object. STOP
4. If X.node is a file, load X.node as binary addon. STOP

LOAD_INDEX(X)
1. If X/index.js is a file
    a. Find the closest package scope SCOPE to X.
    b. If no scope was found, load X/index.js as a CommonJS module. STOP.
    c. If the SCOPE/package.json contains "type" field,
      1. If the "type" field is "module", load X/index.js as an ECMAScript module. STOP.
      2. Else, load X/index.js as a CommonJS module. STOP.
2. If X/index.json is a file, parse X/index.json to a JavaScript object. STOP
3. If X/index.node is a file, load X/index.node as binary addon. STOP

LOAD_AS_DIRECTORY(X)
1. If X/package.json is a file,
   a. Parse X/package.json, and look for "main" field.
   b. If "main" is a falsy value, GOTO 2.
   c. let M = X + (json main field)
   d. LOAD_AS_FILE(M)
   e. LOAD_INDEX(M)
   f. LOAD_INDEX(X) DEPRECATED
   g. THROW "not found"
2. LOAD_INDEX(X)

LOAD_NODE_MODULES(X, START)
1. let DIRS = NODE_MODULES_PATHS(START)
2. for each DIR in DIRS:
   a. LOAD_PACKAGE_EXPORTS(X, DIR)
   b. LOAD_AS_FILE(DIR/X)
   c. LOAD_AS_DIRECTORY(DIR/X)

NODE_MODULES_PATHS(START)
1. let PARTS = path split(START)
2. let I = count of PARTS - 1
3. let DIRS = []
4. while I >= 0,
   a. if PARTS[I] = "node_modules", GOTO d.
   b. DIR = path join(PARTS[0 .. I] + "node_modules")
   c. DIRS = DIR + DIRS
   d. let I = I - 1
5. return DIRS + GLOBAL_FOLDERS

LOAD_PACKAGE_IMPORTS(X, DIR)
1. Find the closest package scope SCOPE to DIR.
2. If no scope was found, return.
3. If the SCOPE/package.json "imports" is null or undefined, return.
4. If `--experimental-require-module` is enabled
  a. let CONDITIONS = ["node", "require", "module-sync"]
  b. Else, let CONDITIONS = ["node", "require"]
5. let MATCH = PACKAGE_IMPORTS_RESOLVE(X, pathToFileURL(SCOPE),
  CONDITIONS) <a href="esm.md#resolver-algorithm-specification">defined in the ESM resolver</a>.
6. RESOLVE_ESM_MATCH(MATCH).

LOAD_PACKAGE_EXPORTS(X, DIR)
1. Try to interpret X as a combination of NAME and SUBPATH where the name
   may have a @scope/ prefix and the subpath begins with a slash (`/`).
2. If X does not match this pattern or DIR/NAME/package.json is not a file,
   return.
3. Parse DIR/NAME/package.json, and look for "exports" field.
4. If "exports" is null or undefined, return.
5. If `--experimental-require-module` is enabled
  a. let CONDITIONS = ["node", "require", "module-sync"]
  b. Else, let CONDITIONS = ["node", "require"]
6. let MATCH = PACKAGE_EXPORTS_RESOLVE(pathToFileURL(DIR/NAME), "." + SUBPATH,
   `package.json` "exports", CONDITIONS) <a href="esm.md#resolver-algorithm-specification">defined in the ESM resolver</a>.
7. RESOLVE_ESM_MATCH(MATCH)

LOAD_PACKAGE_SELF(X, DIR)
1. Find the closest package scope SCOPE to DIR.
2. If no scope was found, return.
3. If the SCOPE/package.json "exports" is null or undefined, return.
4. If the SCOPE/package.json "name" is not the first segment of X, return.
5. let MATCH = PACKAGE_EXPORTS_RESOLVE(pathToFileURL(SCOPE),
   "." + X.slice("name".length), `package.json` "exports", ["node", "require"])
   <a href="esm.md#resolver-algorithm-specification">defined in the ESM resolver</a>.
6. RESOLVE_ESM_MATCH(MATCH)

RESOLVE_ESM_MATCH(MATCH)
1. let RESOLVED_PATH = fileURLToPath(MATCH)
2. If the file at RESOLVED_PATH exists, load RESOLVED_PATH as its extension
   format. STOP
3. THROW "not found"
```

## 缓存

<!--type=misc-->

模块在第一次加载后被缓存。这意味着（除其他外）每次调用 `require('foo')` 如果解析到同一个文件，将返回完全相同的对象。

如果 `require.cache` 没有被修改，多次调用 `require('foo')` 不会导致模块代码被执行多次。这是一个重要的特性。有了它，可以返回"部分完成"的对象，从而允许加载传递依赖项，即使它们会导致循环。

要让模块多次执行代码，请导出一个函数，然后调用该函数。

### 模块缓存的注意事项

<!--type=misc-->

模块基于其解析的文件名进行缓存。由于模块可能根据调用模块的位置（从 `node_modules` 文件夹加载）解析为不同的文件名，因此不能 _保证_ `require('foo')` 如果解析到不同的文件将始终返回完全相同的对象。

此外，在不区分大小写的文件系统或操作系统上，不同的解析文件名可能指向同一个文件，但缓存仍会将它们视为不同的模块并多次重新加载文件。例如，`require('./foo')` 和 `require('./FOO')` 返回两个不同的对象，无论 `./foo` 和 `./FOO` 是否是同一个文件。

## 内置模块

<!--type=misc-->

<!-- YAML
changes:
  - version:
      - v16.0.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/37246
    description: Added `node:` import support to `require(...)`.
-->

Node.js 有几个编译到二进制文件中的模块。这些模块在本文档的其他地方有更详细的描述。

内置模块在 Node.js 源代码中定义，并位于 `lib/` 文件夹中。

可以使用 `node:` 前缀来识别内置模块，在这种情况下，它会绕过 `require` 缓存。例如，`require('node:http')` 将始终返回内置的 HTTP 模块，即使有同名的 `require.cache` 条目。

某些内置模块在其标识符传递给 `require()` 时总是优先加载。例如，`require('http')` 将始终返回内置的 HTTP 模块，即使存在同名的文件。

所有内置模块的列表可以从 [`module.builtinModules`][] 中检索。列出的模块都不带 `node:` 前缀，除了那些强制要求此类前缀的模块（如下一节所述）。

### 强制使用 `node:` 前缀的内置模块

当通过 `require()` 加载时，某些内置模块必须使用 `node:` 前缀请求。此要求的存在是为了防止新引入的内置模块与已经使用该名称的用户空间包发生冲突。目前需要 `node:` 前缀的内置模块有：

* [`node:sea`][]
* [`node:sqlite`][]
* [`node:test`][]
* [`node:test/reporters`][]

这些模块的列表在 [`module.builtinModules`][] 中公开，包括前缀。

## 循环

<!--type=misc-->

当存在循环 `require()` 调用时，模块在返回时可能尚未完成执行。

考虑这种情况：

`a.js`：

```js
console.log('a starting');
exports.done = false;
const b = require('./b.js');
console.log('in a, b.done = %j', b.done);
exports.done = true;
console.log('a done');
```

`b.js`：

```js
console.log('b starting');
exports.done = false;
const a = require('./a.js');
console.log('in b, a.done = %j', a.done);
exports.done = true;
console.log('b done');
```

`main.js`：

```js
console.log('main starting');
const a = require('./a.js');
const b = require('./b.js');
console.log('in main, a.done = %j, b.done = %j', a.done, b.done);
```

当 `main.js` 加载 `a.js` 时，`a.js` 依次加载 `b.js`。此时，`b.js` 尝试加载 `a.js`。为了防止无限循环，将 `a.js` 导出对象的 **未完成副本** 返回给 `b.js` 模块。然后 `b.js` 完成加载，并将其 `exports` 对象提供给 `a.js` 模块。

到 `main.js` 加载两个模块时，它们都已完成。因此，这个程序的输出将是：

```console
$ node main.js
main starting
a starting
b starting
in b, a.done = false
b done
in a, b.done = true
a done
in main, a.done = true, b.done = true
```

需要仔细规划以允许循环模块依赖在应用程序中正确工作。

## 文件模块

<!--type=misc-->

如果未找到确切的文件名，Node.js 将尝试加载所需文件名并添加扩展名：`.js`、`.json`，最后是 `.node`。当加载具有不同扩展名的文件（例如 `.cjs`）时，必须将其全名传递给 `require()`，包括其文件扩展名（例如 `require('./file.cjs')`）。

`.json` 文件被解析为 JSON 文本文件，`.node` 文件被解释为使用 `process.dlopen()` 加载的编译插件模块。使用任何其他扩展名（或根本没有扩展名）的文件被解析为 JavaScript 文本文件。请参考 [确定模块系统][] 部分以了解将使用哪种解析目标。

以 `'/'` 为前缀的必需模块是文件的绝对路径。例如，`require('/home/marco/foo.js')` 将加载 `/home/marco/foo.js` 文件。

以 `'./'` 为前缀的必需模块相对于调用 `require()` 的文件。也就是说，`circle.js` 必须与 `foo.js` 在同一目录中，`require('./circle')` 才能找到它。

没有以 `'/'`、`'./'` 或 `'../'` 开头来表示文件，模块必须是核心模块或从 `node_modules` 文件夹加载。

如果给定路径不存在，`require()` 将抛出 [`MODULE_NOT_FOUND`][] 错误。

## 文件夹作为模块

<!--type=misc-->

> Stability: 3 - Legacy: Use [subpath exports][] or [subpath imports][] instead.

有三种方式可以将文件夹作为参数传递给 `require()`。

第一种是在文件夹的根目录中创建一个 [`package.json`][] 文件，该文件指定一个 `main` 模块。示例 [`package.json`][] 文件可能如下所示：

```json
{ "name" : "some-library",
  "main" : "./lib/some-library.js" }
```

如果这是在 `./some-library` 文件夹中，那么 `require('./some-library')` 将尝试加载 `./some-library/lib/some-library.js`。

如果目录中不存在 [`package.json`][] 文件，或者 [`"main"`][] 条目缺失或无法解析，则 Node.js 将尝试从该目录加载 `index.js` 或 `index.node` 文件。例如，如果在前面的示例中没有 [`package.json`][] 文件，那么 `require('./some-library')` 将尝试加载：

* `./some-library/index.js`
* `./some-library/index.node`

如果这些尝试失败，Node.js 将使用默认错误报告整个模块缺失：

```console
Error: Cannot find module 'some-library'
```

在上述所有三种情况下，`import('./some-library')` 调用将导致 [`ERR_UNSUPPORTED_DIR_IMPORT`][] 错误。使用包 [子路径导出][subpath exports] 或 [子路径导入][subpath imports] 可以提供与文件夹作为模块相同的包含组织好处，并且适用于 `require` 和 `import`。

## 从 `node_modules` 文件夹加载

<!--type=misc-->

如果传递给 `require()` 的模块标识符不是 [内置](#built-in-modules) 模块，并且不以 `'/'`、`'../'` 或 `'./'` 开头，那么 Node.js 从当前模块的目录开始，添加 `/node_modules`，并尝试从该位置加载模块。Node.js 不会将 `node_modules` 附加到已经以 `node_modules` 结尾的路径。

如果在那里找不到，那么它移动到父目录，依此类推，直到到达文件系统的根目录。

例如，如果 `'/home/ry/projects/foo.js'` 处的文件调用 `require('bar.js')`，那么 Node.js 将按以下顺序查找以下位置：

* `/home/ry/projects/node_modules/bar.js`
* `/home/ry/node_modules/bar.js`
* `/home/node_modules/bar.js`
* `/node_modules/bar.js`

这允许程序本地化它们的依赖项，这样它们就不会冲突。

可以通过在模块名称后包含路径后缀来要求模块分发的特定文件或子模块。例如，`require('example-module/path/to/file')` 将解析 `path/to/file`，相对于 `example-module` 的位置。带后缀的路径遵循相同的模块解析语义。

## 从全局文件夹加载

<!-- type=misc -->

如果 `NODE_PATH` 环境变量设置为以冒号分隔的绝对路径列表，那么 Node.js 将在其他地方找不到模块时搜索这些路径。

在 Windows 上，`NODE_PATH` 以分号 (`;`) 而不是冒号分隔。

`NODE_PATH` 最初是为了在当前 [模块解析][] 算法定义之前从不同路径加载模块而创建的。

`NODE_PATH` 仍然受支持，但现在不太必要，因为 Node.js 生态系统已经就定位依赖模块的约定达成一致。有时，依赖 `NODE_PATH` 的部署在人们不知道必须设置 `NODE_PATH` 时表现出令人惊讶的行为。有时模块的依赖关系发生变化，导致在搜索 `NODE_PATH` 时加载不同版本（甚至不同模块）的模块。

此外，Node.js 将在以下 GLOBAL_FOLDERS 列表中搜索：

* 1: `$HOME/.node_modules`
* 2: `$HOME/.node_libraries`
* 3: `$PREFIX/lib/node`

其中 `$HOME` 是用户的主目录，`$PREFIX` 是 Node.js 配置的 `node_prefix`。

这些主要是由于历史原因。

强烈建议将依赖项放在本地 `node_modules` 文件夹中。这些将加载得更快，更可靠。

## 模块包装器

<!-- type=misc -->

在模块代码执行之前，Node.js 会用一个函数包装器包装它，如下所示：

```js
(function(exports, require, module, __filename, __dirname) {
// Module code actually lives in here
});
```

通过这样做，Node.js 实现了以下几点：

* 它将顶级变量（用 `var`、`const` 或 `let` 定义）的作用域限定在模块而不是全局对象。
* 它有助于提供一些看起来是全局但实际上特定于模块的变量，例如：
  * `module` 和 `exports` 对象，实现者可以使用它们从模块导出值。
  * 便利变量 `__filename` 和 `__dirname`，包含模块的绝对文件名和目录路径。

## 模块作用域

### `__dirname`

<!-- YAML
added: v0.1.27
-->

* 类型: {string}

当前模块的目录名。这与 [`__filename`][] 的 [`path.dirname()`][] 相同。

示例：从 `/Users/mjr` 运行 `node example.js`

```js
console.log(__dirname);
// Prints: /Users/mjr
console.log(path.dirname(__filename));
// Prints: /Users/mjr
```

### `__filename`

<!-- YAML
added: v0.0.1
-->

* 类型: {string}

当前模块的文件名。这是当前模块文件的绝对路径，符号链接已解析。

对于主程序，这不一定与命令行中使用的文件名相同。

参见 [`__dirname`][] 获取当前模块的目录名。

示例：

从 `/Users/mjr` 运行 `node example.js`

```js
console.log(__filename);
// Prints: /Users/mjr/example.js
console.log(__dirname);
// Prints: /Users/mjr
```

给定两个模块：`a` 和 `b`，其中 `b` 是 `a` 的依赖项，并且存在以下目录结构：

* `/Users/mjr/app/a.js`
* `/Users/mjr/app/node_modules/b/b.js`

在 `b.js` 内对 `__filename` 的引用将返回 `/Users/mjr/app/node_modules/b/b.js`，而在 `a.js` 内对 `__filename` 的引用将返回 `/Users/mjr/app/a.js`。

### `exports`

<!-- YAML
added: v0.1.12
-->

* 类型: {Object}

对 `module.exports` 的引用，输入更短。有关何时使用 `exports` 和何时使用 `module.exports` 的详细信息，请参见 [exports 快捷方式][] 部分。

### `module`

<!-- YAML
added: v0.1.16
-->

* 类型: {module}

对当前模块的引用，请参见关于 [`module` 对象][] 的部分。特别是，`module.exports` 用于定义模块导出什么并通过 `require()` 使其可用。

### `require(id)`

<!-- YAML
added: v0.1.13
-->

* `id` {string} 模块名或路径
* 返回: {any} 导出的模块内容

用于导入模块、`JSON` 和本地文件。可以从 `node_modules` 导入模块。可以使用相对路径（例如 `./`、`./foo`、`./bar/baz`、`../foo`）导入本地模块和 JSON 文件，该路径将根据 [`__dirname`][] 命名的目录（如果已定义）或当前工作目录进行解析。POSIX 风格的相对路径以与操作系统无关的方式解析，这意味着上述示例在 Windows 上的工作方式与在 Unix 系统上的工作方式相同。

```js
// Importing a local module with a path relative to the `__dirname` or current
// working directory. (On Windows, this would resolve to .\path\myLocalModule.)
const myLocalModule = require('./path/myLocalModule');

// Importing a JSON file:
const jsonData = require('./path/filename.json');

// Importing a module from node_modules or Node.js built-in module:
const crypto = require('node:crypto');
```

#### `require.cache`

<!-- YAML
added: v0.3.0
-->

* 类型: {Object}

模块在需要时缓存在此对象中。通过从此对象中删除键值，下一次 `require` 将重新加载模块。这不适用于 [原生插件][native addons]，重新加载会导致错误。

添加或替换条目也是可能的。在检查内置模块之前会检查此缓存，如果将与内置模块名称匹配的名称添加到缓存中，则只有带 `node:` 前缀的 require 调用才会接收内置模块。请小心使用！

<!-- eslint-disable node-core/no-duplicate-requires, no-restricted-syntax -->

```js
const assert = require('node:assert');
const realFs = require('node:fs');

const fakeFs = {};
require.cache.fs = { exports: fakeFs };

assert.strictEqual(require('fs'), fakeFs);
assert.strictEqual(require('node:fs'), realFs);
```

#### `require.extensions`

<!-- YAML
added: v0.3.0
deprecated: v0.10.6
-->

> Stability: 0 - Deprecated

* 类型: {Object}

指示 `require` 如何处理某些文件扩展名。

将扩展名为 `.sjs` 的文件作为 `.js` 处理：

```js
require.extensions['.sjs'] = require.extensions['.js'];
```

**已弃用。** 过去，此列表用于通过按需编译将非 JavaScript 模块加载到 Node.js 中。然而，在实践中，有更好的方法可以做到这一点，例如通过其他 Node.js 程序加载模块，或提前将它们编译为 JavaScript。

避免使用 `require.extensions`。使用可能会导致细微的错误，并且解析扩展名会随着每个注册的扩展名而变慢。

#### `require.main`

<!-- YAML
added: v0.1.17
-->

* 类型: {module | undefined}

表示 Node.js 进程启动时加载的入口脚本的 `Module` 对象，如果程序的入口点不是 CommonJS 模块，则为 `undefined`。参见 ["访问主模块"](#accessing-the-main-module)。

在 `entry.js` 脚本中：

```js
console.log(require.main);
```

```bash
node entry.js
```

<!-- eslint-skip -->

```js
Module {
  id: '.',
  path: '/absolute/path/to',
  exports: {},
  filename: '/absolute/path/to/entry.js',
  loaded: false,
  children: [],
  paths:
   [ '/absolute/path/to/node_modules',
     '/absolute/path/node_modules',
     '/absolute/node_modules',
     '/node_modules' ] }
```

#### `require.resolve(request[, options])`

<!-- YAML
added: v0.3.0
changes:
  - version: v8.9.0
    pr-url: https://github.com/nodejs/node/pull/16397
    description: The `paths` option is now supported.
-->

* `request` {string} 要解析的模块路径。
* `options` {Object}
  * `paths` {string\[]} 从中解析模块位置的路径。如果存在，这些路径将代替默认解析路径使用，但始终包含的 [GLOBAL_FOLDERS][] 如 `$HOME/.node_modules` 除外。这些路径中的每一个都用作模块解析算法的起点，这意味着从此位置检查 `node_modules` 层次结构。
* 返回: {string}

使用内部的 `require()` 机制来查找模块的位置，但不是加载模块，只是返回解析后的文件名。

如果找不到模块，则抛出 `MODULE_NOT_FOUND` 错误。

##### `require.resolve.paths(request)`

<!-- YAML
added: v8.9.0
-->

* `request` {string} 正在检索其查找路径的模块路径。
* 返回: {string\[]|null}

返回一个包含在解析 `request` 期间搜索的路径的数组，如果 `request` 字符串引用核心模块（例如 `http` 或 `fs`），则返回 `null`。

## `module` 对象

<!-- YAML
added: v0.1.16
-->

<!-- name=module -->

* 类型: {Object}

在每个模块中，`module` 自由变量是对表示当前模块的对象的引用。为方便起见，`module.exports` 也可以通过 `exports` 模块全局访问。`module` 实际上不是全局的，而是每个模块本地的。

### `module.children`

<!-- YAML
added: v0.1.16
-->

* 类型: {module\[]}

此模块首次需要的模块对象。

### `module.exports`

<!-- YAML
added: v0.1.16
-->

* 类型: {Object}

`module.exports` 对象由 `Module` 系统创建。有时这是不可接受的；许多人希望他们的模块是某个类的实例。为此，将所需的导出对象分配给 `module.exports`。将所需对象分配给 `exports` 只会重新绑定本地 `exports` 变量，这可能不是想要的。

例如，假设我们正在制作一个名为 `a.js` 的模块：

```js
const EventEmitter = require('node:events');

module.exports = new EventEmitter();

// Do some work, and after some time emit
// the 'ready' event from the module itself.
setTimeout(() => {
  module.exports.emit('ready');
}, 1000);
```

然后在另一个文件中我们可以这样做：

```js
const a = require('./a');
a.on('ready', () => {
  console.log('module "a" is ready');
});
```

对 `module.exports` 的赋值必须立即完成。不能在任何回调中完成。这不起作用：

`x.js`：

```js
setTimeout(() => {
  module.exports = { a: 'hello' };
}, 0);
```

`y.js`：

```js
const x = require('./x');
console.log(x.a);
```

#### `exports` 快捷方式

<!-- YAML
added: v0.1.16
-->

`exports` 变量在模块的文件级作用域内可用，并在模块评估之前被赋予 `module.exports` 的值。

它允许一个快捷方式，因此 `module.exports.f = ...` 可以更简洁地写为 `exports.f = ...`。但是，请注意，像任何变量一样，如果为 `exports` 分配了新值，它不再绑定到 `module.exports`：

```js
module.exports.hello = true; // Exported from require of module
exports = { hello: false };  // Not exported, only available in the module
```

当 `module.exports` 属性被新对象完全替换时，通常也会重新分配 `exports`：

<!-- eslint-disable func-name-matching -->

```js
module.exports = exports = function Constructor() {
  // ... etc.
};
```

为了说明行为，想象一下这个假设的 `require()` 实现，它与 `require()` 实际所做的非常相似：

```js
function require(/* ... */) {
  const module = { exports: {} };
  ((module, exports) => {
    // Module code here. In this example, define a function.
    function someFunc() {}
    exports = someFunc;
    // At this point, exports is no longer a shortcut to module.exports, and
    // this module will still export an empty default object.
    module.exports = someFunc;
    // At this point, the module will now export someFunc, instead of the
    // default object.
  })(module, module.exports);
  return module.exports;
}
```

### `module.filename`

<!-- YAML
added: v0.1.16
-->

* 类型: {string}

模块的完全解析文件名。

### `module.id`

<!-- YAML
added: v0.1.16
-->

* 类型: {string}

模块的标识符。通常这是完全解析的文件名。

### `module.isPreloading`

<!-- YAML
added:
  - v15.4.0
  - v14.17.0
-->

* 类型: {boolean} 如果模块在 Node.js 预加载阶段运行，则为 `true`。

### `module.loaded`

<!-- YAML
added: v0.1.16
-->

* 类型: {boolean}

模块是否已完成加载，或正在加载过程中。

### `module.parent`

<!-- YAML
added: v0.1.16
deprecated:
  - v14.6.0
  - v12.19.0
-->

> Stability: 0 - Deprecated: Please use [`require.main`][] and
> [`module.children`][] instead.

* 类型: {module | null | undefined}

第一个需要此模块的模块，如果当前模块是当前进程的入口点，则为 `null`，或者如果模块是由不是 CommonJS 模块的东西加载的（例如：REPL 或 `import`），则为 `undefined`。

### `module.path`

<!-- YAML
added: v11.14.0
-->

* 类型: {string}

模块的目录名。这通常与 [`module.id`][] 的 [`path.dirname()`][] 相同。

### `module.paths`

<!-- YAML
added: v0.4.0
-->

* 类型: {string\[]}

模块的搜索路径。

### `module.require(id)`

<!-- YAML
added: v0.5.1
-->

* `id` {string}
* 返回: {any} 导出的模块内容

`module.require()` 方法提供了一种加载模块的方式，就像从原始模块调用 `require()` 一样。

为了做到这一点，有必要获取对 `module` 对象的引用。由于 `require()` 返回 `module.exports`，并且 `module` 通常 _仅_ 在特定模块的代码中可用，因此必须显式导出才能使用。

## `Module` 对象

此部分已移至 [Modules: `module` core module](module.md#the-module-object)。

<!-- Anchors to make sure old links find a target -->

* <a id="modules_module_builtinmodules" href="module.html#modulebuiltinmodules">`module.builtinModules`</a>
* <a id="modules_module_createrequire_filename" href="module.html#modulecreaterequirefilename">`module.createRequire(filename)`</a>
* <a id="modules_module_syncbuiltinesmexports" href="module.html#modulesyncbuiltinesmexports">`module.syncBuiltinESMExports()`</a>

## Source map v3 支持

此部分已移至 [Modules: `module` core module](module.md#source-map-support)。

<!-- Anchors to make sure old links find a target -->

* <a id="modules_module_findsourcemap_path_error" href="module.html#modulefindsourcemappath">`module.findSourceMap(path)`</a>
* <a id="modules_class_module_sourcemap" href="module.html#class-modulesourcemap">Class: `module.SourceMap`</a>
  * <a id="modules_new_sourcemap_payload" href="module.html#new-sourcemappayload--linelengths-">`new SourceMap(payload)`</a>
  * <a id="modules_sourcemap_payload" href="module.html#sourcemappayload">`sourceMap.payload`</a>
  * <a id="modules_sourcemap_findentry_linenumber_columnnumber" href="module.html#sourcemapfindentrylineoffset-columnoffset">`sourceMap.findEntry(lineNumber, columnNumber)`</a>

[确定模块系统]: packages.md#determining-module-system
[ECMAScript Modules]: esm.md
[GLOBAL_FOLDERS]: #loading-from-the-global-folders
[`"main"`]: packages.md#main
[`"type"`]: packages.md#type
[`--trace-require-module`]: cli.md#--trace-require-modulemode
[`ERR_REQUIRE_ASYNC_MODULE`]: errors.md#err_require_async_module
[`ERR_UNSUPPORTED_DIR_IMPORT`]: errors.md#err_unsupported_dir_import
[`MODULE_NOT_FOUND`]: errors.md#module_not_found
[`__dirname`]: #__dirname
[`__filename`]: #__filename
[`import()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import
[`module.builtinModules`]: module.md#modulebuiltinmodules
[`module.children`]: #modulechildren
[`module.id`]: #moduleid
[`module` core module]: module.md
[`module` object]: #the-module-object
[`node:sea`]: single-executable-applications.md#single-executable-application-api
[`node:sqlite`]: sqlite.md
[`node:test/reporters`]: test.md#test-reporters
[`node:test`]: test.md
[`package.json`]: packages.md#nodejs-packagejson-field-definitions
[`path.dirname()`]: path.md#pathdirnamepath
[`process.features.require_module`]: process.md#processfeaturesrequire_module
[`require.main`]: #requiremain
[exports shortcut]: #exports-shortcut
[module namespace object]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/import#module_namespace_object
[module resolution]: #all-together
[native addons]: addons.md
[subpath exports]: packages.md#subpath-exports
[subpath imports]: packages.md#subpath-imports