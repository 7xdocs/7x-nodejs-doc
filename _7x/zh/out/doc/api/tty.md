# TTY

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/tty.js -->

`node:tty` 模块提供了 `tty.ReadStream` 和 `tty.WriteStream` 类。在大多数情况下，没有必要或可能直接使用此模块。但是，可以通过以下方式访问它：

```js
const tty = require('node:tty');
```

当 Node.js 检测到它正在运行并附加了一个文本终端（"TTY"）时，默认情况下，[`process.stdin`][] 将被初始化为 `tty.ReadStream` 的实例，而 [`process.stdout`][] 和 [`process.stderr`][] 默认情况下将是 `tty.WriteStream` 的实例。判断 Node.js 是否在 TTY 上下文中运行的推荐方法是检查 `process.stdout.isTTY` 属性的值是否为 `true`：

```console
$ node -p -e "Boolean(process.stdout.isTTY)"
true
$ node -p -e "Boolean(process.stdout.isTTY)" | cat
false
```

在大多数情况下，应用程序应该几乎没有理由手动创建 `tty.ReadStream` 和 `tty.WriteStream` 类的实例。

## 类：`tty.ReadStream`

<!-- YAML
added: v0.5.8
-->

* 继承自：{net.Socket}

表示 TTY 的可读端。在正常情况下，[`process.stdin`][] 将是 Node.js 进程中唯一的 `tty.ReadStream` 实例，并且没有理由创建额外的实例。

### `readStream.isRaw`

<!-- YAML
added: v0.7.7
-->

一个 `boolean`，如果 TTY 当前配置为作为原始设备运行，则为 `true`。

当进程启动时，此标志始终为 `false`，即使终端以原始模式运行。其值将随着后续调用 `setRawMode` 而改变。

### `readStream.isTTY`

<!-- YAML
added: v0.5.8
-->

一个 `boolean`，对于 `tty.ReadStream` 实例始终为 `true`。

### `readStream.setRawMode(mode)`

<!-- YAML
added: v0.7.7
-->

* `mode` {boolean} 如果为 `true`，则将 `tty.ReadStream` 配置为作为原始设备运行。如果为 `false`，则将 `tty.ReadStream` 配置为其默认模式运行。`readStream.isRaw` 属性将被设置为结果模式。
* 返回：{this} 读取流实例。

允许配置 `tty.ReadStream`，使其作为原始设备运行。

在原始模式下，输入总是按字符可用，不包括修饰符。此外，终端对所有字符的特殊处理都被禁用，包括回显输入字符。在此模式下，<kbd>Ctrl</kbd>+<kbd>C</kbd> 将不再导致 `SIGINT`。

## 类：`tty.WriteStream`

<!-- YAML
added: v0.5.8
-->

* 继承自：{net.Socket}

表示 TTY 的可写端。在正常情况下，[`process.stdout`][] 和 [`process.stderr`][] 将是 Node.js 进程创建的唯二 `tty.WriteStream` 实例，并且没有理由创建额外的实例。

### `new tty.ReadStream(fd[, options])`

<!-- YAML
added: v0.5.8
changes:
  - version: v0.9.4
    description: The `options` argument is supported.
-->

* `fd` {number} 与 TTY 关联的文件描述符。
* `options` {Object} 传递给父类 `net.Socket` 的选项，参见 [`net.Socket` 构造函数][] 的 `options`。
* 返回：{tty.ReadStream}

为与 TTY 关联的 `fd` 创建一个 `ReadStream`。

### `new tty.WriteStream(fd)`

<!-- YAML
added: v0.5.8
-->

* `fd` {number} 与 TTY 关联的文件描述符。
* 返回：{tty.WriteStream}

为与 TTY 关联的 `fd` 创建一个 `WriteStream`。

### 事件：`'resize'`

<!-- YAML
added: v0.7.7
-->

每当 `writeStream.columns` 或 `writeStream.rows` 属性发生变化时，就会触发 `'resize'` 事件。调用监听器回调时不传递任何参数。

```js
process.stdout.on('resize', () => {
  console.log('screen size has changed!');
  console.log(`${process.stdout.columns}x${process.stdout.rows}`);
});
```

### `writeStream.clearLine(dir[, callback])`

<!-- YAML
added: v0.7.7
changes:
  - version: v12.7.0
    pr-url: https://github.com/nodejs/node/pull/28721
    description: The stream's write() callback and return value are exposed.
-->

* `dir` {number}
  * `-1`: 从光标向左
  * `1`: 从光标向右
  * `0`: 整行
* `callback` {Function} 操作完成后调用。
* 返回：{boolean} 如果流希望调用代码在继续写入更多数据之前等待 `'drain'` 事件被触发，则为 `false`；否则为 `true`。

`writeStream.clearLine()` 按 `dir` 标识的方向清除此 `WriteStream` 的当前行。

### `writeStream.clearScreenDown([callback])`

<!-- YAML
added: v0.7.7
changes:
  - version: v12.7.0
    pr-url: https://github.com/nodejs/node/pull/28721
    description: The stream's write() callback and return value are exposed.
-->

* `callback` {Function} 操作完成后调用。
* 返回：{boolean} 如果流希望调用代码在继续写入更多数据之前等待 `'drain'` 事件被触发，则为 `false`；否则为 `true`。

`writeStream.clearScreenDown()` 从此 `WriteStream` 的当前光标向下清除。

### `writeStream.columns`

<!-- YAML
added: v0.7.7
-->

一个 `number`，指定 TTY 当前具有的列数。每当触发 `'resize'` 事件时，此属性都会更新。

### `writeStream.cursorTo(x[, y][, callback])`

<!-- YAML
added: v0.7.7
changes:
  - version: v12.7.0
    pr-url: https://github.com/nodejs/node/pull/28721
    description: The stream's write() callback and return value are exposed.
-->

* `x` {number}
* `y` {number}
* `callback` {Function} 操作完成后调用。
* 返回：{boolean} 如果流希望调用代码在继续写入更多数据之前等待 `'drain'` 事件被触发，则为 `false`；否则为 `true`。

`writeStream.cursorTo()` 将此 `WriteStream` 的光标移动到指定位置。

### `writeStream.getColorDepth([env])`

<!-- YAML
added: v9.9.0
-->

* `env` {Object} 包含要检查的环境变量的对象。这允许模拟特定终端的使用。**默认值：** `process.env`。
* 返回：{number}

返回：

* `1` 表示 2 色，
* `4` 表示 16 色，
* `8` 表示 256 色，
* `24` 表示 16,777,216 色支持。

使用此方法来确定终端支持的颜色。由于终端中颜色的性质，可能会出现误报或漏报。这取决于进程信息和可能谎报所用终端的环境变量。
可以传入一个 `env` 对象来模拟特定终端的使用。这对于检查特定环境设置的行为很有用。

要强制特定的颜色支持，请使用以下环境设置之一。

* 2 色：`FORCE_COLOR = 0`（禁用颜色）
* 16 色：`FORCE_COLOR = 1`
* 256 色：`FORCE_COLOR = 2`
* 16,777,216 色：`FORCE_COLOR = 3`

使用 `NO_COLOR` 和 `NODE_DISABLE_COLORS` 环境变量也可以禁用颜色支持。

### `writeStream.getWindowSize()`

<!-- YAML
added: v0.7.7
-->

* 返回：{number\[]}

`writeStream.getWindowSize()` 返回与此 `WriteStream` 对应的 TTY 的大小。数组类型为 `[numColumns, numRows]`，其中 `numColumns` 和 `numRows` 表示相应 TTY 中的列数和行数。

### `writeStream.hasColors([count][, env])`

<!-- YAML
added:
 - v11.13.0
 - v10.16.0
-->

* `count` {integer} 请求的颜色数量（至少为 2）。**默认值：** 16。
* `env` {Object} 包含要检查的环境变量的对象。这允许模拟特定终端的使用。**默认值：** `process.env`。
* 返回：{boolean}

如果 `writeStream` 支持至少与 `count` 中提供的颜色数量一样多，则返回 `true`。最低支持为 2（黑色和白色）。

这与 [`writeStream.getColorDepth()`][] 中描述的误报和漏报相同。

```js
process.stdout.hasColors();
// 根据 `stdout` 是否支持至少 16 色返回 true 或 false。
process.stdout.hasColors(256);
// 根据 `stdout` 是否支持至少 256 色返回 true 或 false。
process.stdout.hasColors({ TMUX: '1' });
// 返回 true。
process.stdout.hasColors(2 ** 24, { TMUX: '1' });
// 返回 false（环境设置假装支持 2 ** 8 色）。
```

### `writeStream.isTTY`

<!-- YAML
added: v0.5.8
-->

一个 `boolean`，始终为 `true`。

### `writeStream.moveCursor(dx, dy[, callback])`

<!-- YAML
added: v0.7.7
changes:
  - version: v12.7.0
    pr-url: https://github.com/nodejs/node/pull/28721
    description: The stream's write() callback and return value are exposed.
-->

* `dx` {number}
* `dy` {number}
* `callback` {Function} 操作完成后调用。
* 返回：{boolean} 如果流希望调用代码在继续写入更多数据之前等待 `'drain'` 事件被触发，则为 `false`；否则为 `true`。

`writeStream.moveCursor()` 相对于当前位置移动此 `WriteStream` 的光标。

### `writeStream.rows`

<!-- YAML
added: v0.7.7
-->

一个 `number`，指定 TTY 当前具有的行数。每当触发 `'resize'` 事件时，此属性都会更新。

## `tty.isatty(fd)`

<!-- YAML
added: v0.5.8
-->

* `fd` {number} 一个数字文件描述符
* 返回：{boolean}

如果给定的 `fd` 与 TTY 关联，则 `tty.isatty()` 方法返回 `true`，否则返回 `false`，包括当 `fd` 不是非负整数时。

[`net.Socket` 构造函数]: net.md#new-netsocketoptions
[`process.stderr`]: process.md#processstderr
[`process.stdin`]: process.md#processstdin
[`process.stdout`]: process.md#processstdout
[`writeStream.getColorDepth()`]: #writestreamgetcolordepthenv