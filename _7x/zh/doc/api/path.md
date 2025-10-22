# Path

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/path.js -->

`node:path` 模块提供了用于处理文件和目录路径的实用工具。可以通过以下方式访问：

```cjs
const path = require('node:path');
```

```mjs
import path from 'node:path';
```

## Windows 与 POSIX

`node:path` 模块的默认操作根据 Node.js 应用程序运行的操作系统而有所不同。具体来说，在 Windows 操作系统上运行时，`node:path` 模块将假定使用的是 Windows 风格的路径。

因此，在 POSIX 和 Windows 上使用 `path.basename()` 可能会产生不同的结果：

在 POSIX 上：

```js
path.basename('C:\\temp\\myfile.html');
// Returns: 'C:\\temp\\myfile.html'
```

在 Windows 上：

```js
path.basename('C:\\temp\\myfile.html');
// Returns: 'myfile.html'
```

为了在任何操作系统上处理 Windows 文件路径时获得一致的结果，请使用 [`path.win32`][]：

在 POSIX 和 Windows 上：

```js
path.win32.basename('C:\\temp\\myfile.html');
// Returns: 'myfile.html'
```

为了在任何操作系统上处理 POSIX 文件路径时获得一致的结果，请使用 [`path.posix`][]：

在 POSIX 和 Windows 上：

```js
path.posix.basename('/tmp/myfile.html');
// Returns: 'myfile.html'
```

在 Windows 上，Node.js 遵循每个驱动器工作目录的概念。当使用不带反斜杠的驱动器路径时，可以观察到这种行为。例如，`path.resolve('C:\\')` 可能返回与 `path.resolve('C:')` 不同的结果。更多信息，请参阅 [此 MSDN 页面][MSDN-Rel-Path]。

## `path.basename(path[, suffix])`

<!-- YAML
added: v0.1.25
changes:
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/5348
    description: Passing a non-string as the `path` argument will throw now.
-->

* `path` {string}
* `suffix` {string} 要移除的可选后缀
* Returns: {string}

`path.basename()` 方法返回 `path` 的最后一部分，类似于 Unix 的 `basename` 命令。尾随的[目录分隔符][`path.sep`]会被忽略。

```js
path.basename('/foo/bar/baz/asdf/quux.html');
// Returns: 'quux.html'

path.basename('/foo/bar/baz/asdf/quux.html', '.html');
// Returns: 'quux'
```

尽管 Windows 通常以不区分大小写的方式处理文件名（包括文件扩展名），但此函数不会。例如，`C:\\foo.html` 和 `C:\\foo.HTML` 指向同一个文件，但 `basename` 将扩展名视为区分大小写的字符串：

```js
path.win32.basename('C:\\foo.html', '.html');
// Returns: 'foo'

path.win32.basename('C:\\foo.HTML', '.html');
// Returns: 'foo.HTML'
```

如果 `path` 不是字符串，或者提供了 `suffix` 但它不是字符串，则会抛出 [`TypeError`][]。

## `path.delimiter`

<!-- YAML
added: v0.9.3
-->

* 类型: {string}

提供特定于平台的路径定界符：

* Windows 上是 `;`
* POSIX 上是 `:`

例如，在 POSIX 上：

```js
console.log(process.env.PATH);
// Prints: '/usr/bin:/bin:/usr/sbin:/sbin:/usr/local/bin'

process.env.PATH.split(path.delimiter);
// Returns: ['/usr/bin', '/bin', '/usr/sbin', '/sbin', '/usr/local/bin']
```

在 Windows 上：

```js
console.log(process.env.PATH);
// Prints: 'C:\Windows\system32;C:\Windows;C:\Program Files\node\'

process.env.PATH.split(path.delimiter);
// Returns ['C:\\Windows\\system32', 'C:\\Windows', 'C:\\Program Files\\node\\']
```

## `path.dirname(path)`

<!-- YAML
added: v0.1.16
changes:
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/5348
    description: Passing a non-string as the `path` argument will throw now.
-->

* `path` {string}
* Returns: {string}

`path.dirname()` 方法返回 `path` 的目录名，类似于 Unix 的 `dirname` 命令。尾随的目录分隔符会被忽略，参见 [`path.sep`][]。

```js
path.dirname('/foo/bar/baz/asdf/quux');
// Returns: '/foo/bar/baz/asdf'
```

如果 `path` 不是字符串，则会抛出 [`TypeError`][]。

## `path.extname(path)`

<!-- YAML
added: v0.1.25
changes:
  - version: v6.0.0
    pr-url: https://github.com/nodejs/node/pull/5348
    description: Passing a non-string as the `path` argument will throw now.
-->

* `path` {string}
* Returns: {string}

`path.extname()` 方法返回 `path` 的扩展名，即 `path` 的最后一部分中从最后一次出现 `.`（点）字符到字符串结尾的部分。如果 `path` 的最后一部分中没有 `.`，或者 `path` 的基本名称（参见 `path.basename()`）除了第一个字符之外没有 `.` 字符，则返回空字符串。

```js
path.extname('index.html');
// Returns: '.html'

path.extname('index.coffee.md');
// Returns: '.md'

path.extname('index.');
// Returns: '.'

path.extname('index');
// Returns: ''

path.extname('.index');
// Returns: ''

path.extname('.index.md');
// Returns: '.md'
```

如果 `path` 不是字符串，则会抛出 [`TypeError`][]。

## `path.format(pathObject)`

<!-- YAML
added: v0.11.15
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44349
    description: The dot will be added if it is not specified in `ext`.
-->

* `pathObject` {Object} 任何具有以下属性的 JavaScript 对象：
  * `dir` {string}
  * `root` {string}
  * `base` {string}
  * `name` {string}
  * `ext` {string}
* Returns: {string}

`path.format()` 方法从对象返回路径字符串。这与 [`path.parse()`][] 相反。

当向 `pathObject` 提供属性时，请注意存在一些组合，其中一个属性优先于另一个：

* 如果提供了 `pathObject.dir`，则 `pathObject.root` 被忽略
* 如果存在 `pathObject.base`，则 `pathObject.ext` 和 `pathObject.name` 被忽略

例如，在 POSIX 上：

```js
// 如果提供了 `dir`、`root` 和 `base`，
// 将返回 `${dir}${path.sep}${base}`。`root` 被忽略。
path.format({
  root: '/ignored',
  dir: '/home/user/dir',
  base: 'file.txt',
});
// Returns: '/home/user/dir/file.txt'

// 如果未指定 `dir`，则将使用 `root`。
// 如果仅提供了 `root` 或 `dir` 等于 `root`，则
// 不会包含平台分隔符。`ext` 将被忽略。
path.format({
  root: '/',
  base: 'file.txt',
  ext: 'ignored',
});
// Returns: '/file.txt'

// 如果未指定 `base`，则将使用 `name` + `ext`。
path.format({
  root: '/',
  name: 'file',
  ext: '.txt',
});
// Returns: '/file.txt'

// 如果 `ext` 中未指定点，则会自动添加。
path.format({
  root: '/',
  name: 'file',
  ext: 'txt',
});
// Returns: '/file.txt'
```

在 Windows 上：

```js
path.format({
  dir: 'C:\\path\\dir',
  base: 'file.txt',
});
// Returns: 'C:\\path\\dir\\file.txt'
```

## `path.matchesGlob(path, pattern)`

<!-- YAML
added:
  - v22.5.0
  - v20.17.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59572
    description: Marking the API stable.
-->

* `path` {string} 要进行全局匹配的路径。
* `pattern` {string} 要检查路径的全局模式。
* Returns: {boolean} `path` 是否匹配 `pattern`。

`path.matchesGlob()` 方法判断 `path` 是否匹配 `pattern`。

例如：

```js
path.matchesGlob('/foo/bar', '/foo/*'); // true
path.matchesGlob('/foo/bar*', 'foo/bird'); // false
```

如果 `path` 或 `pattern` 不是字符串，则会抛出 [`TypeError`][]。

## `path.isAbsolute(path)`

<!-- YAML
added: v0.11.2
-->

* `path` {string}
* Returns: {boolean}

`path.isAbsolute()` 方法判断给定的 `path` 是否是绝对路径。因此，它对于缓解路径遍历并不安全。

如果给定的 `path` 是零长度字符串，则返回 `false`。

例如，在 POSIX 上：

```js
path.isAbsolute('/foo/bar');   // true
path.isAbsolute('/baz/..');    // true
path.isAbsolute('/baz/../..'); // true
path.isAbsolute('qux/');       // false
path.isAbsolute('.');          // false
```

在 Windows 上：

```js
path.isAbsolute('//server');    // true
path.isAbsolute('\\\\server');  // true
path.isAbsolute('C:/foo/..');   // true
path.isAbsolute('C:\\foo\\..'); // true
path.isAbsolute('bar\\baz');    // false
path.isAbsolute('bar/baz');     // false
path.isAbsolute('.');           // false
```

如果 `path` 不是字符串，则会抛出 [`TypeError`][]。

## `path.join([...paths])`

<!-- YAML
added: v0.1.16
-->

* `...paths` {string} 路径段的序列
* Returns: {string}

`path.join()` 方法使用平台特定的分隔符作为定界符将所有给定的 `path` 段连接在一起，然后对生成的路径进行规范化。

零长度的 `path` 段会被忽略。如果连接后的路径字符串是零长度字符串，则返回 `'.'`，表示当前工作目录。

```js
path.join('/foo', 'bar', 'baz/asdf', 'quux', '..');
// Returns: '/foo/bar/baz/asdf'

path.join('foo', {}, 'bar');
// Throws 'TypeError: Path must be a string. Received {}'
```

如果任何路径段不是字符串，则会抛出 [`TypeError`][]。

## `path.normalize(path)`

<!-- YAML
added: v0.1.23
-->

* `path` {string}
* Returns: {string}

`path.normalize()` 方法对给定的 `path` 进行规范化，解析 `'..'` 和 `'.'` 段。

当找到多个连续的路径段分隔符（例如，POSIX 上的 `/` 以及 Windows 上的 `\` 或 `/`）时，它们将被替换为平台特定的路径段分隔符的单个实例（POSIX 上是 `/`，Windows 上是 `\`）。尾随的分隔符会保留。

如果 `path` 是零长度字符串，则返回 `'.'`，表示当前工作目录。

在 POSIX 上，此函数应用的规范化类型并不严格遵循 POSIX 规范。例如，此函数会将两个前导斜杠替换为单个斜杠，就好像它是一个常规的绝对路径一样，而一些 POSIX 系统对以恰好两个斜杠开头的路径赋予了特殊含义。类似地，此函数执行的其他替换，例如移除 `..` 段，可能会改变底层系统解析路径的方式。

例如，在 POSIX 上：

```js
path.normalize('/foo/bar//baz/asdf/quux/..');
// Returns: '/foo/bar/baz/asdf'
```

在 Windows 上：

```js
path.normalize('C:\\temp\\\\foo\\bar\\..\\');
// Returns: 'C:\\temp\\foo\\'
```

由于 Windows 识别多个路径分隔符，两个分隔符都将被 Windows 首选分隔符 (`\`) 的实例替换：

```js
path.win32.normalize('C:////temp\\\\/\\/\\/foo/bar');
// Returns: 'C:\\temp\\foo\\bar'
```

如果 `path` 不是字符串，则会抛出 [`TypeError`][]。

## `path.parse(path)`

<!-- YAML
added: v0.11.15
-->

* `path` {string}
* Returns: {Object}

`path.parse()` 方法返回一个对象，其属性表示 `path` 的重要元素。尾随的目录分隔符会被忽略，参见 [`path.sep`][]。

返回的对象将具有以下属性：

* `dir` {string}
* `root` {string}
* `base` {string}
* `name` {string}
* `ext` {string}

例如，在 POSIX 上：

```js
path.parse('/home/user/dir/file.txt');
// Returns:
// { root: '/',
//   dir: '/home/user/dir',
//   base: 'file.txt',
//   ext: '.txt',
//   name: 'file' }
```

```text
┌─────────────────────┬────────────┐
│          dir        │    base    │
├──────┬              ├──────┬─────┤
│ root │              │ name │ ext │
"  /    home/user/dir / file  .txt "
└──────┴──────────────┴──────┴─────┘
(所有 "" 行中的空格都应被忽略。它们仅用于格式化。)
```

在 Windows 上：

```js
path.parse('C:\\path\\dir\\file.txt');
// Returns:
// { root: 'C:\\',
//   dir: 'C:\\path\\dir',
//   base: 'file.txt',
//   ext: '.txt',
//   name: 'file' }
```

```text
┌─────────────────────┬────────────┐
│          dir        │    base    │
├──────┬              ├──────┬─────┤
│ root │              │ name │ ext │
" C:\      path\dir   \ file  .txt "
└──────┴──────────────┴──────┴─────┘
(所有 "" 行中的空格都应被忽略。它们仅用于格式化。)
```

如果 `path` 不是字符串，则会抛出 [`TypeError`][]。

## `path.posix`

<!-- YAML
added: v0.11.15
changes:
  - version: v15.3.0
    pr-url: https://github.com/nodejs/node/pull/34962
    description: Exposed as `require('path/posix')`.
-->

* 类型: {Object}

`path.posix` 属性提供对 `path` 方法的 POSIX 特定实现的访问。

该 API 可通过 `require('node:path').posix` 或 `require('node:path/posix')` 访问。

## `path.relative(from, to)`

<!-- YAML
added: v0.5.0
changes:
  - version: v6.8.0
    pr-url: https://github.com/nodejs/node/pull/8523
    description: On Windows, the leading slashes for UNC paths are now included
                 in the return value.
-->

* `from` {string}
* `to` {string}
* Returns: {string}

`path.relative()` 方法根据当前工作目录返回从 `from` 到 `to` 的相对路径。如果 `from` 和 `to` 各自解析为相同的路径（在对每个路径调用 `path.resolve()` 之后），则返回零长度字符串。

如果零长度字符串作为 `from` 或 `to` 传递，则将使用当前工作目录代替零长度字符串。

例如，在 POSIX 上：

```js
path.relative('/data/orandea/test/aaa', '/data/orandea/impl/bbb');
// Returns: '../../impl/bbb'
```

在 Windows 上：

```js
path.relative('C:\\orandea\\test\\aaa', 'C:\\orandea\\impl\\bbb');
// Returns: '..\\..\\impl\\bbb'
```

如果 `from` 或 `to` 不是字符串，则会抛出 [`TypeError`][]。

## `path.resolve([...paths])`

<!-- YAML
added: v0.3.4
-->

* `...paths` {string} 路径或路径段的序列
* Returns: {string}

`path.resolve()` 方法将路径或路径段的序列解析为绝对路径。

给定的路径序列从右到左处理，每个后续的 `path` 会被前置，直到构造出绝对路径。例如，给定路径段序列：`/foo`, `/bar`, `baz`，调用 `path.resolve('/foo', '/bar', 'baz')` 将返回 `/bar/baz`，因为 `'baz'` 不是绝对路径，但 `'/bar' + '/' + 'baz'` 是。

如果在处理所有给定的 `path` 段之后，尚未生成绝对路径，则使用当前工作目录。

生成的路径会被规范化，并且除非路径解析为根目录，否则尾随斜杠会被移除。

零长度的 `path` 段会被忽略。

如果没有传递 `path` 段，`path.resolve()` 将返回当前工作目录的绝对路径。

```js
path.resolve('/foo/bar', './baz');
// Returns: '/foo/bar/baz'

path.resolve('/foo/bar', '/tmp/file/');
// Returns: '/tmp/file'

path.resolve('wwwroot', 'static_files/png/', '../gif/image.gif');
// 如果当前工作目录是 /home/myself/node，
// 则返回 '/home/myself/node/wwwroot/static_files/gif/image.gif'
```

如果任何参数不是字符串，则会抛出 [`TypeError`][]。

## `path.sep`

<!-- YAML
added: v0.7.9
-->

* 类型: {string}

提供特定于平台的路径段分隔符：

* Windows 上是 `\`
* POSIX 上是 `/`

例如，在 POSIX 上：

```js
'foo/bar/baz'.split(path.sep);
// Returns: ['foo', 'bar', 'baz']
```

在 Windows 上：

```js
'foo\\bar\\baz'.split(path.sep);
// Returns: ['foo', 'bar', 'baz']
```

在 Windows 上，正斜杠 (`/`) 和反斜杠 (`\`) 都被接受为路径段分隔符；但是，`path` 方法只添加反斜杠 (`\`)。

## `path.toNamespacedPath(path)`

<!-- YAML
added: v9.0.0
-->

* `path` {string}
* Returns: {string}

仅在 Windows 系统上，返回给定 `path` 的等效[命名空间前缀路径][namespace-prefixed path]。如果 `path` 不是字符串，则 `path` 将不加修改地返回。

此方法仅在 Windows 系统上有意义。在 POSIX 系统上，该方法是无效操作，并且总是返回未修改的 `path`。

## `path.win32`

<!-- YAML
added: v0.11.15
changes:
  - version: v15.3.0
    pr-url: https://github.com/nodejs/node/pull/34962
    description: Exposed as `require('path/win32')`.
-->

* 类型: {Object}

`path.win32` 属性提供对 `path` 方法的 Windows 特定实现的访问。

该 API 可通过 `require('node:path').win32` 或 `require('node:path/win32')` 访问。

[MSDN-Rel-Path]: https://docs.microsoft.com/en-us/windows/desktop/FileIO/naming-a-file#fully-qualified-vs-relative-paths
[`TypeError`]: errors.md#class-typeerror
[`path.parse()`]: #pathparsepath
[`path.posix`]: #pathposix
[`path.sep`]: #pathsep
[`path.win32`]: #pathwin32
[namespace-prefixed path]: https://docs.microsoft.com/en-us/windows/desktop/FileIO/naming-a-file#namespaces