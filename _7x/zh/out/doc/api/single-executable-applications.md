# 单可执行应用

<!--introduced_in=v19.7.0-->

<!-- YAML
added:
  - v19.7.0
  - v18.16.0
changes:
  - version: v20.6.0
    pr-url: https://github.com/nodejs/node/pull/46824
    description: Added support for "useSnapshot".
  - version: v20.6.0
    pr-url: https://github.com/nodejs/node/pull/48191
    description: Added support for "useCodeCache".
-->

> Stability: 1.1 - Active development

<!-- source_link=src/node_sea.cc -->

此功能允许将 Node.js 应用方便地分发到未安装 Node.js 的系统上。

Node.js 支持通过注入由 Node.js 准备好的数据块（可以包含捆绑的脚本）到 `node` 二进制文件中来创建[单可执行应用][]。在启动时，程序会检查是否注入了任何内容。如果找到了数据块，则执行数据块中的脚本。否则 Node.js 将正常操作。

单可执行应用功能目前仅支持使用 [CommonJS][] 模块系统运行单个嵌入脚本。

用户可以使用 `node` 二进制文件本身以及任何能够向二进制文件中注入资源的工具，从他们捆绑的脚本创建单可执行应用。

以下是使用其中一种工具 [postject][] 创建单可执行应用的步骤：

1. 创建一个 JavaScript 文件：
   ```bash
   echo 'console.log(`Hello, ${process.argv[2]}!`);' > hello.js
   ```

2. 创建一个配置文件，构建可以注入到单可执行应用的数据块（详情请参阅[生成单可执行应用准备数据块][]）：
   ```bash
   echo '{ "main": "hello.js", "output": "sea-prep.blob" }' > sea-config.json
   ```

3. 生成要注入的数据块：
   ```bash
   node --experimental-sea-config sea-config.json
   ```

4. 创建 `node` 可执行文件的副本，并根据需要命名：

   * 在非 Windows 系统上：

   ```bash
   cp $(command -v node) hello
   ```

   * 在 Windows 上：

   ```text
   node -e "require('fs').copyFileSync(process.execPath, 'hello.exe')"
   ```

   `.exe` 扩展名是必需的。

5. 移除二进制文件的签名（仅限 macOS 和 Windows）：

   * 在 macOS 上：

   ```bash
   codesign --remove-signature hello
   ```

   * 在 Windows 上（可选）：

   可以使用已安装的 [Windows SDK][] 中的 [signtool][]。如果跳过此步骤，请忽略 postject 发出的任何与签名相关的警告。

   ```powershell
   signtool remove /s hello.exe
   ```

6. 通过运行 `postject` 并指定以下选项，将数据块注入到复制的二进制文件中：

   * `hello` / `hello.exe` - 第 4 步中创建的 `node` 可执行文件副本的名称。
   * `NODE_SEA_BLOB` - 二进制文件中资源 / 注释 / 段的名称，数据块的内容将存储在此。
   * `sea-prep.blob` - 第 1 步中创建的数据块的名称。
   * `--sentinel-fuse NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2` - Node.js 项目用于检测文件是否已被注入的[熔断器][]。
   * `--macho-segment-name NODE_SEA`（仅在 macOS 上需要）- 二进制文件中段的名称，数据块的内容将存储在此。

   总结一下，以下是各平台所需的命令：

   * 在 Linux 上：
     ```bash
     npx postject hello NODE_SEA_BLOB sea-prep.blob \
         --sentinel-fuse NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2
     ```

   * 在 Windows - PowerShell 上：
     ```powershell
     npx postject hello.exe NODE_SEA_BLOB sea-prep.blob `
         --sentinel-fuse NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2
     ```

   * 在 Windows - 命令提示符上：
     ```text
     npx postject hello.exe NODE_SEA_BLOB sea-prep.blob ^
         --sentinel-fuse NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2
     ```

   * 在 macOS 上：
     ```bash
     npx postject hello NODE_SEA_BLOB sea-prep.blob \
         --sentinel-fuse NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2 \
         --macho-segment-name NODE_SEA
     ```

7. 为二进制文件签名（仅限 macOS 和 Windows）：

   * 在 macOS 上：

   ```bash
   codesign --sign - hello
   ```

   * 在 Windows 上（可选）：

   需要有证书才能工作。但是，未签名的二进制文件仍然可以运行。

   ```powershell
   signtool sign /fd SHA256 hello.exe
   ```

8. 运行二进制文件：

   * 在非 Windows 系统上

   ```console
   $ ./hello world
   Hello, world!
   ```

   * 在 Windows 上

   ```console
   $ .\hello.exe world
   Hello, world!
   ```

## 生成单可执行应用准备数据块

可以使用用于构建单可执行应用的 Node.js 二进制文件的 `--experimental-sea-config` 标志来生成要注入到应用程序中的单可执行应用准备数据块。它接受一个 JSON 格式的配置文件路径。如果传递给它的路径不是绝对路径，Node.js 将使用相对于当前工作目录的路径。

配置目前读取以下顶级字段：

```json
{
  "main": "/path/to/bundled/script.js",
  "output": "/path/to/write/the/generated/blob.blob",
  "disableExperimentalSEAWarning": true, // Default: false
  "useSnapshot": false,  // Default: false
  "useCodeCache": true, // Default: false
  "execArgv": ["--no-warnings", "--max-old-space-size=4096"], // Optional
  "execArgvExtension": "env", // Default: "env", options: "none", "env", "cli"
  "assets": {  // Optional
    "a.dat": "/path/to/a.dat",
    "b.txt": "/path/to/b.txt"
  }
}
```

如果路径不是绝对路径，Node.js 将使用相对于当前工作目录的路径。用于生成数据块的 Node.js 二进制文件的版本必须与将要注入数据块的二进制文件版本相同。

注意：当生成跨平台的 SEA（例如，在 `darwin-arm64` 上生成用于 `linux-x64` 的 SEA）时，必须将 `useCodeCache` 和 `useSnapshot` 设置为 false，以避免生成不兼容的可执行文件。因为代码缓存和快照只能在编译它们的同一平台上加载，生成的可执行文件在启动时尝试加载在不同平台上构建的代码缓存或快照时可能会崩溃。

### 资源

用户可以通过在配置中添加一个键值路径字典作为 `assets` 字段来包含资源。在构建时，Node.js 会从指定路径读取资源并将它们捆绑到准备数据块中。在生成的可执行文件中，用户可以使用 [`sea.getAsset()`][] 和 [`sea.getAssetAsBlob()`][] API 来检索资源。

```json
{
  "main": "/path/to/bundled/script.js",
  "output": "/path/to/write/the/generated/blob.blob",
  "assets": {
    "a.jpg": "/path/to/a.jpg",
    "b.txt": "/path/to/b.txt"
  }
}
```

单可执行应用可以按如下方式访问资源：

```cjs
const { getAsset, getAssetAsBlob, getRawAsset, getAssetKeys } = require('node:sea');
// 获取所有资源键。
const keys = getAssetKeys();
console.log(keys); // ['a.jpg', 'b.txt']
// 返回 ArrayBuffer 中数据的副本。
const image = getAsset('a.jpg');
// 将资源解码为 UTF8 字符串返回。
const text = getAsset('b.txt', 'utf8');
// 返回包含资源的 Blob。
const blob = getAssetAsBlob('a.jpg');
// 返回包含原始资源的 ArrayBuffer，不进行复制。
const raw = getRawAsset('a.jpg');
```

有关更多信息，请参阅 [`sea.getAsset()`][]、[`sea.getAssetAsBlob()`][]、[`sea.getRawAsset()`][] 和 [`sea.getAssetKeys()`][] API 的文档。

### 启动快照支持

`useSnapshot` 字段可用于启用启动快照支持。在这种情况下，当最终的可执行文件启动时，`main` 脚本不会运行。相反，它会在构建机器上生成单可执行应用准备数据块时运行。生成的准备数据块将包含一个捕获了 `main` 脚本初始化状态的快照。带有注入的准备数据块的最终可执行文件将在运行时反序列化该快照。

当 `useSnapshot` 为 true 时，主脚本必须调用 [`v8.startupSnapshot.setDeserializeMainFunction()`][] API 来配置在最终可执行文件被用户启动时需要运行的代码。

在单可执行应用中使用快照的典型模式是：

1. 在构建时，在构建机器上，运行主脚本以将堆初始化到准备好接收用户输入的状态。该脚本还应使用 [`v8.startupSnapshot.setDeserializeMainFunction()`][] 配置一个主函数。此函数将被编译并序列化到快照中，但在构建时不会调用。
2. 在运行时，主函数将在用户机器的反序列化堆上运行，以处理用户输入并生成输出。

启动快照脚本的一般约束也适用于用于为单可执行应用构建快照的主脚本，并且主脚本可以使用 [`v8.startupSnapshot` API][] 来适应这些约束。请参阅[Node.js 中的启动快照支持文档][]。

### V8 代码缓存支持

当在配置中将 `useCodeCache` 设置为 `true` 时，在生成单可执行应用准备数据块期间，Node.js 将编译 `main` 脚本以生成 V8 代码缓存。生成的代码缓存将成为准备数据块的一部分，并被注入到最终的可执行文件中。当单可执行应用启动时，Node.js 将使用代码缓存来加速编译，而不是从头开始编译 `main` 脚本，然后执行该脚本，这将提高启动性能。

**注意：** 当 `useCodeCache` 为 `true` 时，`import()` 不起作用。

### 执行参数

`execArgv` 字段可用于指定 Node.js 特定的参数，这些参数将在单可执行应用启动时自动应用。这允许应用程序开发人员配置 Node.js 运行时选项，而无需最终用户了解这些标志。

例如，以下配置：

```json
{
  "main": "/path/to/bundled/script.js",
  "output": "/path/to/write/the/generated/blob.blob",
  "execArgv": ["--no-warnings", "--max-old-space-size=2048"]
}
```

将指示 SEA 在启动时使用 `--no-warnings` 和 `--max-old-space-size=2048` 标志。在可执行文件中嵌入的脚本中，可以使用 `process.execArgv` 属性访问这些标志：

```js
// 如果使用 `sea user-arg1 user-arg2` 启动可执行文件
console.log(process.execArgv);
// 打印：['--no-warnings', '--max-old-space-size=2048']
console.log(process.argv);
// 打印 ['/path/to/sea', 'path/to/sea', 'user-arg1', 'user-arg2']
```

用户提供的参数位于 `process.argv` 数组中，从索引 2 开始，类似于使用以下命令启动应用程序时的情况：

```console
node --no-warnings --max-old-space-size=2048 /path/to/bundled/script.js user-arg1 user-arg2
```

### 执行参数扩展

`execArgvExtension` 字段控制如何在 `execArgv` 字段指定的参数之外提供额外的执行参数。它接受以下三个字符串值之一：

* `"none"`：不允许扩展。将仅使用 `execArgv` 中指定的参数，并且将忽略 `NODE_OPTIONS` 环境变量。
* `"env"`：_（默认）_ `NODE_OPTIONS` 环境变量可以扩展执行参数。这是为了保持向后兼容性的默认行为。
* `"cli"`：可以使用 `--node-options="--flag1 --flag2"` 启动可执行文件，这些标志将被解析为 Node.js 的执行参数，而不是传递给用户脚本。这允许使用 `NODE_OPTIONS` 环境变量不支持的参数。

例如，使用 `"execArgvExtension": "cli"`：

```json
{
  "main": "/path/to/bundled/script.js",
  "output": "/path/to/write/the/generated/blob.blob",
  "execArgv": ["--no-warnings"],
  "execArgvExtension": "cli"
}
```

可执行文件可以这样启动：

```console
./my-sea --node-options="--trace-exit" user-arg1 user-arg2
```

这将等同于运行：

```console
node --no-warnings --trace-exit /path/to/bundled/script.js user-arg1 user-arg2
```

## 在注入的主脚本中

### 单可执行应用 API

`node:sea` 内置模块允许从嵌入到可执行文件中的 JavaScript 主脚本与单可执行应用进行交互。

#### `sea.isSea()`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

* 返回：{boolean} 此脚本是否在单可执行应用内运行。

### `sea.getAsset(key[, encoding])`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

此方法可用于检索在构建时配置为捆绑到单可执行应用中的资源。
当找不到匹配的资源时会抛出错误。

* `key`  {string} 在单可执行应用配置的 `assets` 字段指定的字典中的资源键。
* `encoding` {string} 如果指定，资源将被解码为字符串。支持 `TextDecoder` 接受的任何编码。如果未指定，则返回包含资源副本的 `ArrayBuffer`。
* 返回：{string|ArrayBuffer}

### `sea.getAssetAsBlob(key[, options])`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

与 [`sea.getAsset()`][] 类似，但返回一个 {Blob}。
当找不到匹配的资源时会抛出错误。

* `key`  {string} 在单可执行应用配置的 `assets` 字段指定的字典中的资源键。
* `options` {Object}
  * `type` {string} Blob 的可选 MIME 类型。
* 返回：{Blob}

### `sea.getRawAsset(key)`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

此方法可用于检索在构建时配置为捆绑到单可执行应用中的资源。
当找不到匹配的资源时会抛出错误。

与 `sea.getAsset()` 或 `sea.getAssetAsBlob()` 不同，此方法不返回副本。相反，它返回可执行文件内捆绑的原始资源。

目前，用户应避免写入返回的 ArrayBuffer。如果注入的段未标记为可写或未正确对齐，写入返回的 ArrayBuffer 很可能导致崩溃。

* `key`  {string} 在单可执行应用配置的 `assets` 字段指定的字典中的资源键。
* 返回：{ArrayBuffer}

### `sea.getAssetKeys()`

<!-- YAML
added: v24.8.0
-->

* 返回 {string\[]} 包含嵌入到可执行文件中的所有资源键的数组。如果未嵌入任何资源，则返回空数组。

此方法可用于检索嵌入到单可执行应用中的所有资源键的数组。
当不在单可执行应用内运行时，会抛出错误。

### 注入的主脚本中的 `require(id)` 不是基于文件的

注入的主脚本中的 `require()` 与可用于非注入模块的 [`require()`][] 不同。它也没有非注入的 [`require()`][] 所具有的任何属性，除了 [`require.main`][]。它只能用于加载内置模块。尝试加载只能在文件系统中找到的模块将抛出错误。

用户可以将他们的应用程序捆绑到一个独立的 JavaScript 文件中以注入到可执行文件中，而不是依赖基于文件的 `require()`。这也确保了更确定的依赖关系图。

但是，如果仍然需要基于文件的 `require()`，也可以实现：

```js
const { createRequire } = require('node:module');
require = createRequire(__filename);
```

### 注入的主脚本中的 `__filename` 和 `module.filename`

注入的主脚本中的 `__filename` 和 `module.filename` 的值等于 [`process.execPath`][]。

### 注入的主脚本中的 `__dirname`

注入的主脚本中的 `__dirname` 的值等于 [`process.execPath`][] 的目录名。

## 注意

### 单可执行应用创建过程

旨在创建单可执行 Node.js 应用的工具必须将使用 `--experimental-sea-config` 准备的 blob 内容注入到：

* 如果 `node` 二进制文件是 [PE][] 文件，则注入到名为 `NODE_SEA_BLOB` 的资源中
* 如果 `node` 二进制文件是 [Mach-O][] 文件，则注入到 `NODE_SEA` 段中名为 `NODE_SEA_BLOB` 的节中
* 如果 `node` 二进制文件是 [ELF][] 文件，则注入到名为 `NODE_SEA_BLOB` 的注释中

在二进制文件中搜索 `NODE_SEA_FUSE_fce680ab2cc467b6e072b8b5df1996b2:0` [熔断器][]字符串，并将最后一个字符翻转为 `1` 以指示已注入资源。

### 平台支持

单可执行支持仅在以下平台上定期在 CI 上进行测试：

* Windows
* macOS
* Linux（Node.js [支持的所有发行版][]，除了 Alpine；以及 Node.js [支持的所有架构][]，除了 s390x）

这是由于缺乏更好的工具来生成可用于在其他平台上测试此功能的单可执行文件。

欢迎提出关于其他资源注入工具/工作流程的建议。请访问 <https://github.com/nodejs/single-executable/discussions> 开始讨论以帮助我们记录它们。

[CommonJS]: modules.md#modules-commonjs-modules
[ELF]: https://en.wikipedia.org/wiki/Executable_and_Linkable_Format
[Generating single executable preparation blobs]: #generating-single-executable-preparation-blobs
[Mach-O]: https://en.wikipedia.org/wiki/Mach-O
[PE]: https://en.wikipedia.org/wiki/Portable_Executable
[Windows SDK]: https://developer.microsoft.com/en-us/windows/downloads/windows-sdk/
[`process.execPath`]: process.md#processexecpath
[`require()`]: modules.md#requireid
[`require.main`]: modules.md#accessing-the-main-module
[`sea.getAsset()`]: #seagetassetkey-encoding
[`sea.getAssetAsBlob()`]: #seagetassetasblobkey-options
[`sea.getAssetKeys()`]: #seagetassetkeys
[`sea.getRawAsset()`]: #seagetrawassetkey
[`v8.startupSnapshot.setDeserializeMainFunction()`]: v8.md#v8startupsnapshotsetdeserializemainfunctioncallback-data
[`v8.startupSnapshot` API]: v8.md#startup-snapshot-api
[documentation about startup snapshot support in Node.js]: cli.md#--build-snapshot
[fuse]: https://www.electronjs.org/docs/latest/tutorial/fuses
[postject]: https://github.com/nodejs/postject
[signtool]: https://learn.microsoft.com/en-us/windows/win32/seccrypto/signtool
[single executable applications]: https://github.com/nodejs/single-executable
[supported by Node.js]: https://github.com/nodejs/node/blob/main/BUILDING.md#platform-list
