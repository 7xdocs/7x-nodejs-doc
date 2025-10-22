# Permissions

<!--introduced_in=v20.0.0-->

权限可用于控制 Node.js 进程可以访问哪些系统资源，或者进程可以使用这些资源执行哪些操作。

* [基于进程的权限](#process-based-permissions) 控制 Node.js 进程对资源的访问。
  资源可以被完全允许或拒绝，或者可以控制与其相关的操作。例如，可以允许文件系统读取同时拒绝写入。
  此功能不能防范恶意代码。根据 Node.js [安全政策][Security Policy]，Node.js 信任任何要求其运行的代码。

权限模型实现了"安全带"方法，防止受信任的代码无意中更改文件或使用未明确授予访问权限的资源。它在存在恶意代码的情况下不提供安全保证。恶意代码可以绕过权限模型，并在不受权限模型限制的情况下执行任意代码。

如果您发现潜在的安全漏洞，请参考我们的[安全政策][Security Policy]。

## 基于进程的权限

### 权限模型

<!-- YAML
added: v20.0.0
changes:
  - version:
    - v23.5.0
    - v22.13.0
    pr-url: https://github.com/nodejs/node/pull/56201
    description: This feature is no longer experimental.
-->

> Stability: 2 - Stable

Node.js 权限模型是一种在执行期间限制对特定资源访问的机制。
该 API 存在于 [`--permission`][] 标志之后，当启用该标志时，将限制对所有可用权限的访问。

可用权限由 [`--permission`][] 标志记录。

当使用 `--permission` 启动 Node.js 时，通过 `fs` 模块访问文件系统、生成进程、使用 `node:worker_threads`、使用原生插件、使用 WASI 以及启用运行时调试器的能力将受到限制。

```console
$ node --permission index.js

Error: Access to this API has been restricted
    at node:internal/main/run_main_module:23:47 {
  code: 'ERR_ACCESS_DENIED',
  permission: 'FileSystemRead',
  resource: '/home/user/index.js'
}
```

允许生成进程和创建工作线程可以分别使用 [`--allow-child-process`][] 和 [`--allow-worker`][]。

在使用权限模型时允许原生插件，请使用 [`--allow-addons`][] 标志。对于 WASI，使用 [`--allow-wasi`][] 标志。

#### 运行时 API

当通过 [`--permission`][] 标志启用权限模型时，会在 `process` 对象上添加一个新的属性 `permission`。
此属性包含一个函数：

##### `permission.has(scope[, reference])`

用于在运行时检查权限的 API 调用 ([`permission.has()`][])

```js
process.permission.has('fs.write'); // true
process.permission.has('fs.write', '/home/rafaelgss/protected-folder'); // true

process.permission.has('fs.read'); // true
process.permission.has('fs.read', '/home/rafaelgss/protected-folder'); // false
```

#### 文件系统权限

默认情况下，权限模型通过 `node:fs` 模块限制对文件系统的访问。
它不保证用户无法通过其他方式访问文件系统，例如通过 `node:sqlite` 模块。

要允许访问文件系统，请使用 [`--allow-fs-read`][] 和 [`--allow-fs-write`][] 标志：

```console
$ node --permission --allow-fs-read=* --allow-fs-write=* index.js
Hello world!
```

默认情况下，您的应用程序的入口点包含在允许的文件系统读取列表中。例如：

```console
$ node --permission index.js
```

* `index.js` 将被包含在允许的文件系统读取列表中

```console
$ node -r /path/to/custom-require.js --permission index.js.
```

* `/path/to/custom-require.js` 将被包含在允许的文件系统读取列表中。
* `index.js` 将被包含在允许的文件系统读取列表中。

这两个标志的有效参数是：

* `*` - 分别允许所有 `FileSystemRead` 或 `FileSystemWrite` 操作。
* 相对于当前工作目录的路径。
* 绝对路径。

示例：

* `--allow-fs-read=*` - 它将允许所有 `FileSystemRead` 操作。
* `--allow-fs-write=*` - 它将允许所有 `FileSystemWrite` 操作。
* `--allow-fs-write=/tmp/` - 它将允许对 `/tmp/` 文件夹的 `FileSystemWrite` 访问。
* `--allow-fs-read=/tmp/ --allow-fs-read=/home/.gitignore` - 它允许对 `/tmp/` 文件夹 **和** `/home/.gitignore` 路径的 `FileSystemRead` 访问。

也支持通配符：

* `--allow-fs-read=/home/test*` 将允许读取访问匹配通配符的所有内容。例如：`/home/test/file1` 或 `/home/test2`

传递通配符 (`*`) 后，所有后续字符将被忽略。例如：`/home/*.js` 将类似于 `/home/*` 工作。

当权限模型初始化时，如果指定的目录存在，它将自动添加一个通配符 (\*)。例如，如果 `/home/test/files` 存在，它将被视为 `/home/test/files/*`。但是，如果目录不存在，则不会添加通配符，访问将仅限于 `/home/test/files`。如果您想允许访问尚不存在的文件夹，请确保明确包含通配符：`/my-path/folder-do-not-exist/*`。

#### 将权限模型与 `npx` 一起使用

如果您使用 [`npx`][] 执行 Node.js 脚本，您可以通过传递 `--node-options` 标志来启用权限模型。例如：

```bash
npx --node-options="--permission" package-name
```

这将为 [`npx`][] 生成的所有 Node.js 进程设置 `NODE_OPTIONS` 环境变量，而不影响 `npx` 进程本身。

**使用 `npx` 时的 FileSystemRead 错误**

上述命令可能会抛出 `FileSystemRead` 无效访问错误，因为 Node.js 需要文件系统读取访问权限来定位和执行包。为避免这种情况：

1. **使用全局安装的包**
   通过运行以下命令授予对全局 `node_modules` 目录的读取访问权限：

   ```bash
   npx --node-options="--permission --allow-fs-read=$(npm prefix -g)" package-name
   ```

2. **使用 `npx` 缓存**
   如果您是临时安装包或依赖 `npx` 缓存，请授予对 npm 缓存目录的读取访问权限：

   ```bash
   npx --node-options="--permission --allow-fs-read=$(npm config get cache)" package-name
   ```

您通常传递给 `node` 的任何参数（例如，`--allow-*` 标志）也可以通过 `--node-options` 标志传递。这种灵活性使得在使用 `npx` 时可以根据需要轻松配置权限。

#### 权限模型约束

在使用此系统之前，您需要了解一些约束：

* 该模型不会继承到工作线程。
* 使用权限模型时，以下功能将受到限制：
  * 原生模块
  * 子进程
  * 工作线程
  * 调试器协议
  * 文件系统访问
  * WASI
* 权限模型在 Node.js 环境设置完成后初始化。
  但是，某些标志（如 `--env-file` 或 `--openssl-config`）设计为在环境初始化之前读取文件。因此，这些标志不受权限模型规则的约束。这同样适用于可以通过运行时通过 `v8.setFlagsFromString` 设置的 V8 标志。
* 启用权限模型时，无法在运行时请求 OpenSSL 引擎，这会影响内置的 crypto、https 和 tls 模块。
* 启用权限模型时无法加载运行时可加载扩展，这会影响 sqlite 模块。
* 通过 `node:fs` 模块使用现有文件描述符会绕过权限模型。

#### 限制和已知问题

* 符号链接将被跟踪，即使指向访问权限已被授予的路径集之外的位置。相对符号链接可能允许访问任意文件和目录。在启用权限模型启动应用程序时，您必须确保没有已被授予访问权限的路径包含相对符号链接。

[Security Policy]: https://github.com/nodejs/node/blob/main/SECURITY.md
[`--allow-addons`]: cli.md#--allow-addons
[`--allow-child-process`]: cli.md#--allow-child-process
[`--allow-fs-read`]: cli.md#--allow-fs-read
[`--allow-fs-write`]: cli.md#--allow-fs-write
[`--allow-wasi`]: cli.md#--allow-wasi
[`--allow-worker`]: cli.md#--allow-worker
[`--permission`]: cli.md#--permission
[`npx`]: https://docs.npmjs.com/cli/commands/npx
[`permission.has()`]: process.md#processpermissionhasscope-reference