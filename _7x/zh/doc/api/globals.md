# 全局对象

<!--introduced_in=v0.10.0-->

<!-- type=misc -->

> Stability: 2 - Stable

这些对象在所有模块中都可用。

以下变量看起来是全局的，但实际上不是。它们只存在于 [CommonJS 模块][] 的作用域中：

* [`__dirname`][]
* [`__filename`][]
* [`exports`][]
* [`module`][]
* [`require()`][]

此处列出的对象是 Node.js 特有的。JavaScript 语言本身也有一些[内置对象][]，它们也是全局可访问的。

## 类：`AbortController`

<!-- YAML
added:
  - v15.0.0
  - v14.17.0
changes:
  - version: v15.4.0
    pr-url: https://github.com/nodejs/node/pull/35949
    description: No longer experimental.
-->

一个实用类，用于在选定的基于 `Promise` 的 API 中发出取消信号。该 API 基于 Web API {AbortController}。

```js
const ac = new AbortController();

ac.signal.addEventListener('abort', () => console.log('Aborted!'),
                           { once: true });

ac.abort();

console.log(ac.signal.aborted);  // Prints true
```

### `abortController.abort([reason])`

<!-- YAML
added:
  - v15.0.0
  - v14.17.0
changes:
  - version:
      - v17.2.0
      - v16.14.0
    pr-url: https://github.com/nodejs/node/pull/40807
    description: Added the new optional reason argument.
-->

* `reason` {any} 一个可选的原因，可在 `AbortSignal` 的 `reason` 属性中获取。

触发中止信号，导致 `abortController.signal` 发出 `'abort'` 事件。

### `abortController.signal`

<!-- YAML
added:
  - v15.0.0
  - v14.17.0
-->

* 类型：{AbortSignal}

### 类：`AbortSignal`

<!-- YAML
added:
  - v15.0.0
  - v14.17.0
-->

* 继承自：{EventTarget}

`AbortSignal` 用于在调用 `abortController.abort()` 方法时通知观察者。

#### 静态方法：`AbortSignal.abort([reason])`

<!-- YAML
added:
  - v15.12.0
  - v14.17.0
changes:
  - version:
      - v17.2.0
      - v16.14.0
    pr-url: https://github.com/nodejs/node/pull/40807
    description: Added the new optional reason argument.
-->

* `reason` {any}
* 返回：{AbortSignal}

返回一个新的已经中止的 `AbortSignal`。

#### 静态方法：`AbortSignal.timeout(delay)`

<!-- YAML
added:
  - v17.3.0
  - v16.14.0
-->

* `delay` {number} 在触发 AbortSignal 之前等待的毫秒数。

返回一个新的 `AbortSignal`，它将在 `delay` 毫秒后中止。

#### 静态方法：`AbortSignal.any(signals)`

<!-- YAML
added:
  - v20.3.0
  - v18.17.0
-->

* `signals` {AbortSignal\[]} 用于组合成新 `AbortSignal` 的 `AbortSignal` 数组。

返回一个新的 `AbortSignal`，如果任何提供的信号中止，它也会中止。它的 [`abortSignal.reason`][] 将被设置为导致它中止的那个 `signals` 的原因。

#### 事件：`'abort'`

<!-- YAML
added:
  - v15.0.0
  - v14.17.0
-->

当调用 `abortController.abort()` 方法时，会发出 `'abort'` 事件。回调函数被调用时带有一个参数对象，该对象的 `type` 属性设置为 `'abort'`：

```js
const ac = new AbortController();

// Use either the onabort property...
ac.signal.onabort = () => console.log('aborted!');

// Or the EventTarget API...
ac.signal.addEventListener('abort', (event) => {
  console.log(event.type);  // Prints 'abort'
}, { once: true });

ac.abort();
```

与 `AbortSignal` 关联的 `AbortController` 只会触发一次 `'abort'` 事件。我们建议代码在添加 `'abort'` 事件监听器之前检查 `abortSignal.aborted` 属性是否为 `false`。

任何附加到 `AbortSignal` 的事件监听器应使用 `{ once: true }` 选项（或者，如果使用 `EventEmitter` API 附加监听器，使用 `once()` 方法），以确保事件监听器在 `'abort'` 事件被处理后立即移除。否则可能导致内存泄漏。

#### `abortSignal.aborted`

<!-- YAML
added:
  - v15.0.0
  - v14.17.0
-->

* 类型：{boolean} 在 `AbortController` 中止后为 true。

#### `abortSignal.onabort`

<!-- YAML
added:
  - v15.0.0
  - v14.17.0
-->

* 类型：{Function}

用户代码可以设置的可选回调函数，用于在 `abortController.abort()` 函数被调用时收到通知。

#### `abortSignal.reason`

<!-- YAML
added:
  - v17.2.0
  - v16.14.0
-->

* 类型：{any}

触发 `AbortSignal` 时指定的可选原因。

```js
const ac = new AbortController();
ac.abort(new Error('boom!'));
console.log(ac.signal.reason);  // Error: boom!
```

#### `abortSignal.throwIfAborted()`

<!-- YAML
added:
  - v17.3.0
  - v16.17.0
-->

如果 `abortSignal.aborted` 为 `true`，则抛出 `abortSignal.reason`。

## 类：`Blob`

<!-- YAML
added: v18.0.0
-->

参见 {Blob}。

## 类：`Buffer`

<!-- YAML
added: v0.1.103
-->

* 类型：{Function}

用于处理二进制数据。参见 [buffer 章节][]。

## 类：`ByteLengthQueuingStrategy`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`ByteLengthQueuingStrategy`][] 的浏览器兼容实现。

## `__dirname`

这个变量看起来是全局的，但实际上不是。参见 [`__dirname`][]。

## `__filename`

这个变量看起来是全局的，但实际上不是。参见 [`__filename`][]。

## `atob(data)`

<!-- YAML
added: v16.0.0
-->

> Stability: 3 - Legacy. Use `Buffer.from(data, 'base64')` instead.

[`buffer.atob()`][] 的全局别名。

## 类：`BroadcastChannel`

<!-- YAML
added: v18.0.0
-->

参见 {BroadcastChannel}。

## `btoa(data)`

<!-- YAML
added: v16.0.0
-->

> Stability: 3 - Legacy. Use `buf.toString('base64')` instead.

[`buffer.btoa()`][] 的全局别名。

## `clearImmediate(immediateObject)`

<!-- YAML
added: v0.9.1
-->

[`clearImmediate`][] 在 [timers][] 章节中描述。

## `clearInterval(intervalObject)`

<!-- YAML
added: v0.0.1
-->

[`clearInterval`][] 在 [timers][] 章节中描述。

## `clearTimeout(timeoutObject)`

<!-- YAML
added: v0.0.1
-->

[`clearTimeout`][] 在 [timers][] 章节中描述。

## 类：`CloseEvent`

<!-- YAML
added: v23.0.0
-->

{CloseEvent} 的浏览器兼容实现。使用 [`--no-experimental-websocket`][] CLI 标志可以禁用此 API。

## 类：`CompressionStream`

<!-- YAML
added: v18.0.0
changes:
 - version: v24.7.0
   pr-url: https://github.com/nodejs/node/pull/59464
   description: format now accepts `brotli` value.
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`CompressionStream`][] 的浏览器兼容实现。

## `console`

<!-- YAML
added: v0.1.100
-->

* 类型：{Object}

用于打印到 stdout 和 stderr。参见 [`console`][] 章节。

## 类：`CountQueuingStrategy`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`CountQueuingStrategy`][] 的浏览器兼容实现。

## 类：`Crypto`

<!-- YAML
added:
  - v17.6.0
  - v16.15.0
changes:
  - version: v23.0.0
    pr-url: https://github.com/nodejs/node/pull/52564
    description: No longer experimental.
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/42083
    description: No longer behind `--experimental-global-webcrypto` CLI flag.
-->

{Crypto} 的浏览器兼容实现。仅当 Node.js 二进制文件编译时包含了 `node:crypto` 模块的支持时，此全局对象才可用。

## `crypto`

<!-- YAML
added:
  - v17.6.0
  - v16.15.0
changes:
  - version: v23.0.0
    pr-url: https://github.com/nodejs/node/pull/52564
    description: No longer experimental.
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/42083
    description: No longer behind `--experimental-global-webcrypto` CLI flag.
-->

[Web Crypto API][] 的浏览器兼容实现。

## 类：`CryptoKey`

<!-- YAML
added:
  - v17.6.0
  - v16.15.0
changes:
  - version: v23.0.0
    pr-url: https://github.com/nodejs/node/pull/52564
    description: No longer experimental.
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/42083
    description: No longer behind `--experimental-global-webcrypto` CLI flag.
-->

{CryptoKey} 的浏览器兼容实现。仅当 Node.js 二进制文件编译时包含了 `node:crypto` 模块的支持时，此全局对象才可用。

## 类：`CustomEvent`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
changes:
  - version: v23.0.0
    pr-url: https://github.com/nodejs/node/pull/52723
    description: No longer experimental.
  - version:
    - v22.1.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52618
    description: CustomEvent is now stable.
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44860
    description: No longer behind `--experimental-global-customevent` CLI flag.
-->

{CustomEvent} 的浏览器兼容实现。

## 类：`DecompressionStream`

<!-- YAML
added: v18.0.0
changes:
 - version: v24.7.0
   pr-url: https://github.com/nodejs/node/pull/59464
   description: format now accepts `brotli` value.
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`DecompressionStream`][] 的浏览器兼容实现。

## 类：`Event`

<!-- YAML
added: v15.0.0
changes:
  - version: v15.4.0
    pr-url: https://github.com/nodejs/node/pull/35949
    description: No longer experimental.
-->

`Event` 类的浏览器兼容实现。更多细节参见 [`EventTarget` and `Event` API][]。

## 类：`EventSource`

<!-- YAML
added:
  - v22.3.0
  - v20.18.0
-->

> Stability: 1 - Experimental. Enable this API with the [`--experimental-eventsource`][]
> CLI flag.

{EventSource} 的浏览器兼容实现。

## 类：`EventTarget`

<!-- YAML
added: v15.0.0
changes:
  - version: v15.4.0
    pr-url: https://github.com/nodejs/node/pull/35949
    description: No longer experimental.
-->

`EventTarget` 类的浏览器兼容实现。更多细节参见 [`EventTarget` and `Event` API][]。

## `exports`

这个变量看起来是全局的，但实际上不是。参见 [`exports`][]。

## `fetch`

<!-- YAML
added:
  - v17.5.0
  - v16.15.0
changes:
  - version:
    - v21.0.0
    pr-url: https://github.com/nodejs/node/pull/45684
    description: No longer experimental.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41811
    description: No longer behind `--experimental-fetch` CLI flag.
-->

[`fetch()`][] 函数的浏览器兼容实现。

```mjs
const res = await fetch('https://nodejs.org/api/documentation.json');
if (res.ok) {
  const data = await res.json();
  console.log(data);
}
```

该实现基于 [undici](https://undici.nodejs.org)，一个为 Node.js 从头编写的 HTTP/1.1 客户端。你可以通过读取 `process.versions.undici` 属性来了解你的 Node.js 进程中捆绑的 `undici` 版本。

### 自定义分发器

你可以使用自定义分发器来分发请求，将其传递给 fetch 的选项对象。分发器必须与 `undici` 的 [`Dispatcher` 类](https://undici.nodejs.org/#/docs/api/Dispatcher.md) 兼容。

```js
fetch(url, { dispatcher: new MyAgent() });
```

通过安装 `undici` 并使用 `setGlobalDispatcher()` 方法，可以更改 Node.js 中的全局分发器。调用此方法将同时影响 `undici` 和 Node.js。

```mjs
import { setGlobalDispatcher } from 'undici';
setGlobalDispatcher(new MyAgent());
```

### 相关类

以下全局对象可与 `fetch` 一起使用：

* [`FormData`](https://nodejs.org/api/globals.html#class-formdata)
* [`Headers`](https://nodejs.org/api/globals.html#class-headers)
* [`Request`](https://nodejs.org/api/globals.html#request)
* [`Response`](https://nodejs.org/api/globals.html#response)。

## 类：`File`

<!-- YAML
added: v20.0.0
-->

参见 {File}。

## 类：`FormData`

<!-- YAML
added:
  - v17.6.0
  - v16.15.0
changes:
  - version:
    - v21.0.0
    pr-url: https://github.com/nodejs/node/pull/45684
    description: No longer experimental.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41811
    description: No longer behind `--experimental-fetch` CLI flag.
-->

{FormData} 的浏览器兼容实现。

## `global`

<!-- YAML
added: v0.1.27
-->

> Stability: 3 - Legacy. Use [`globalThis`][] instead.

* 类型：{Object} 全局命名空间对象。

在浏览器中，顶层作用域传统上是全局作用域。这意味着 `var something` 将定义一个新的全局变量，除非在 ECMAScript 模块中。在 Node.js 中，这是不同的。顶层作用域不是全局作用域；Node.js 模块中的 `var something` 将局部于该模块，无论它是 [CommonJS 模块][] 还是 [ECMAScript 模块][]。

## 类：`Headers`

<!-- YAML
added:
  - v17.5.0
  - v16.15.0
changes:
  - version:
    - v21.0.0
    pr-url: https://github.com/nodejs/node/pull/45684
    description: No longer experimental.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41811
    description: No longer behind `--experimental-fetch` CLI flag.
-->

{Headers} 的浏览器兼容实现。

## `localStorage`

<!-- YAML
added: v22.4.0
-->

> Stability: 1.0 - Early development.

[`localStorage`][] 的浏览器兼容实现。数据以未加密的形式存储在由 [`--localstorage-file`][] CLI 标志指定的文件中。可以存储的最大数据量为 10 MB。不支持在 Web Storage API 之外修改此数据。使用 [`--experimental-webstorage`][] CLI 标志启用此 API。在服务器上下文中使用时，`localStorage` 数据不是按用户或按请求存储的，而是在所有用户和请求之间共享。

## 类：`MessageChannel`

<!-- YAML
added: v15.0.0
-->

`MessageChannel` 类。更多细节参见 [`MessageChannel`][]。

## 类：`MessageEvent`

<!-- YAML
added: v15.0.0
-->

{MessageEvent} 的浏览器兼容实现。

## 类：`MessagePort`

<!-- YAML
added: v15.0.0
-->

`MessagePort` 类。更多细节参见 [`MessagePort`][]。

## `module`

这个变量看起来是全局的，但实际上不是。参见 [`module`][]。

## 类：`Navigator`

<!-- YAML
added: v21.0.0
-->

> Stability: 1.1 - Active development. Disable this API with the
> [`--no-experimental-global-navigator`][] CLI flag.

[Navigator API][] 的部分实现。

## `navigator`

<!-- YAML
added: v21.0.0
-->

> Stability: 1.1 - Active development. Disable this API with the
> [`--no-experimental-global-navigator`][] CLI flag.

[`window.navigator`][] 的部分实现。

### `navigator.hardwareConcurrency`

<!-- YAML
added: v21.0.0
-->

* 类型：{number}

`navigator.hardwareConcurrency` 只读属性返回当前 Node.js 实例可用的逻辑处理器数量。

```js
console.log(`This process is running on ${navigator.hardwareConcurrency} logical processors`);
```

### `navigator.language`

<!-- YAML
added: v21.2.0
-->

* 类型：{string}

`navigator.language` 只读属性返回一个字符串，表示 Node.js 实例的首选语言。语言将由 Node.js 在运行时使用的 ICU 库根据操作系统的默认语言确定。

该值表示 [RFC 5646][] 中定义的语言版本。

在没有 ICU 的构建中，回退值为 `'en-US'`。

```js
console.log(`The preferred language of the Node.js instance has the tag '${navigator.language}'`);
```

### `navigator.languages`

<!-- YAML
added: v21.2.0
-->

* 类型：{Array<string>}

`navigator.languages` 只读属性返回一个字符串数组，表示 Node.js 实例的首选语言。默认情况下，`navigator.languages` 仅包含 `navigator.language` 的值，该值将由 Node.js 在运行时使用的 ICU 库根据操作系统的默认语言确定。

在没有 ICU 的构建中，回退值为 `['en-US']`。

```js
console.log(`The preferred languages are '${navigator.languages}'`);
```

### `navigator.platform`

<!-- YAML
added: v21.2.0
-->

* 类型：{string}

`navigator.platform` 只读属性返回一个字符串，标识运行 Node.js 实例的平台。

```js
console.log(`This process is running on ${navigator.platform}`);
```

### `navigator.userAgent`

<!-- YAML
added: v21.1.0
-->

* 类型：{string}

`navigator.userAgent` 只读属性返回由运行时名称和主版本号组成的用户代理字符串。

```js
console.log(`The user-agent is ${navigator.userAgent}`); // Prints "Node.js/21"
```

### `navigator.locks`

<!-- YAML
added: v24.5.0
-->

> Stability: 1 - Experimental

`navigator.locks` 只读属性返回一个 [`LockManager`][] 实例，可用于协调对同一进程内多个线程可能共享的资源的访问。此全局实现与 [浏览器 `LockManager`][] API 的语义匹配。

```mjs
// Request an exclusive lock
await navigator.locks.request('my_resource', async (lock) => {
  // The lock has been acquired.
  console.log(`Lock acquired: ${lock.name}`);
  // Lock is automatically released when the function returns
});

// Request a shared lock
await navigator.locks.request('shared_resource', { mode: 'shared' }, async (lock) => {
  // Multiple shared locks can be held simultaneously
  console.log(`Shared lock acquired: ${lock.name}`);
});
```

```cjs
// Request an exclusive lock
navigator.locks.request('my_resource', async (lock) => {
  // The lock has been acquired.
  console.log(`Lock acquired: ${lock.name}`);
  // Lock is automatically released when the function returns
}).then(() => {
  console.log('Lock released');
});

// Request a shared lock
navigator.locks.request('shared_resource', { mode: 'shared' }, async (lock) => {
  // Multiple shared locks can be held simultaneously
  console.log(`Shared lock acquired: ${lock.name}`);
}).then(() => {
  console.log('Shared lock released');
});
```

详细 API 文档参见 [`worker.locks`][]。

## 类：`PerformanceEntry`

<!-- YAML
added: v19.0.0
-->

`PerformanceEntry` 类。更多细节参见 [`PerformanceEntry`][]。

## 类：`PerformanceMark`

<!-- YAML
added: v19.0.0
-->

`PerformanceMark` 类。更多细节参见 [`PerformanceMark`][]。

## 类：`PerformanceMeasure`

<!-- YAML
added: v19.0.0
-->

`PerformanceMeasure` 类。更多细节参见 [`PerformanceMeasure`][]。

## 类：`PerformanceObserver`

<!-- YAML
added: v19.0.0
-->

`PerformanceObserver` 类。更多细节参见 [`PerformanceObserver`][]。

## 类：`PerformanceObserverEntryList`

<!-- YAML
added: v19.0.0
-->

`PerformanceObserverEntryList` 类。更多细节参见 [`PerformanceObserverEntryList`][]。

## 类：`PerformanceResourceTiming`

<!-- YAML
added: v19.0.0
-->

`PerformanceResourceTiming` 类。更多细节参见 [`PerformanceResourceTiming`][]。

## `performance`

<!-- YAML
added: v16.0.0
-->

[`perf_hooks.performance`][] 对象。

## `process`

<!-- YAML
added: v0.1.7
-->

* 类型：{Object}

进程对象。参见 [`process` object][] 章节。

## `queueMicrotask(callback)`

<!-- YAML
added: v11.0.0
-->

* `callback` {Function} 要排队的函数。

`queueMicrotask()` 方法将一个微任务排队以调用 `callback`。如果 `callback` 抛出异常，[`process` object][] 的 `'uncaughtException'` 事件将被触发。

微任务队列由 V8 管理，可以以类似于由 Node.js 管理的 [`process.nextTick()`][] 队列的方式使用。在 Node.js 事件循环的每一轮中，`process.nextTick()` 队列总是在微任务队列之前处理。

```js
// Here, `queueMicrotask()` is used to ensure the 'load' event is always
// emitted asynchronously, and therefore consistently. Using
// `process.nextTick()` here would result in the 'load' event always emitting
// before any other promise jobs.

DataHandler.prototype.load = async function load(key) {
  const hit = this._cache.get(key);
  if (hit !== undefined) {
    queueMicrotask(() => {
      this.emit('load', hit);
    });
    return;
  }

  const data = await fetchData(key);
  this._cache.set(key, data);
  this.emit('load', data);
};
```

## 类：`ReadableByteStreamController`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`ReadableByteStreamController`][] 的浏览器兼容实现。

## 类：`ReadableStream`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`ReadableStream`][] 的浏览器兼容实现。

## 类：`ReadableStreamBYOBReader`

<!-- YAML
added: v18.0.0
changes:
- version:
  - v23.11.0
  - v22.15.0
  pr-url: https://github.com/nodejs/node/pull/57510
  description: Marking the API stable.
-->

[`ReadableStreamBYOBReader`][] 的浏览器兼容实现。

## 类：`ReadableStreamBYOBRequest`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`ReadableStreamBYOBRequest`][] 的浏览器兼容实现。

## 类：`ReadableStreamDefaultController`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`ReadableStreamDefaultController`][] 的浏览器兼容实现。

## 类：`ReadableStreamDefaultReader`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`ReadableStreamDefaultReader`][] 的浏览器兼容实现。

## `require()`

这个变量看起来是全局的，但实际上不是。参见 [`require()`][]。

## 类：`Response`

<!-- YAML
added:
  - v17.5.0
  - v16.15.0
changes:
  - version:
    - v21.0.0
    pr-url: https://github.com/nodejs/node/pull/45684
    description: No longer experimental.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41811
    description: No longer behind `--experimental-fetch` CLI flag.
-->

{Response} 的浏览器兼容实现。

## 类：`Request`

<!-- YAML
added:
  - v17.5.0
  - v16.15.0
changes:
  - version:
    - v21.0.0
    pr-url: https://github.com/nodejs/node/pull/45684
    description: No longer experimental.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41811
    description: No longer behind `--experimental-fetch` CLI flag.
-->

{Request} 的浏览器兼容实现。

## `sessionStorage`

<!-- YAML
added: v22.4.0
-->

> Stability: 1.0 - Early development.

[`sessionStorage`][] 的浏览器兼容实现。数据存储在内存中，存储配额为 10 MB。`sessionStorage` 数据仅在当前运行的进程中持久存在，并且不在工作线程之间共享。

## `setImmediate(callback[, ...args])`

<!-- YAML
added: v0.9.1
-->

[`setImmediate`][] 在 [timers][] 章节中描述。

## `setInterval(callback, delay[, ...args])`

<!-- YAML
added: v0.0.1
-->

[`setInterval`][] 在 [timers][] 章节中描述。

## `setTimeout(callback, delay[, ...args])`

<!-- YAML
added: v0.0.1
-->

[`setTimeout`][] 在 [timers][] 章节中描述。

## 类：`Storage`

<!-- YAML
added: v22.4.0
-->

> Stability: 1.0 - Early development. Enable this API with the
> [`--experimental-webstorage`][] CLI flag.

{Storage} 的浏览器兼容实现。

## `structuredClone(value[, options])`

<!-- YAML
added: v17.0.0
-->

WHATWG [`structuredClone`][] 方法。

## 类：`SubtleCrypto`

<!-- YAML
added:
  - v17.6.0
  - v16.15.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/42083
    description: No longer behind `--experimental-global-webcrypto` CLI flag.
-->

{SubtleCrypto} 的浏览器兼容实现。仅当 Node.js 二进制文件编译时包含了 `node:crypto` 模块的支持时，此全局对象才可用。

## 类：`DOMException`

<!-- YAML
added: v17.0.0
-->

WHATWG {DOMException} 类。

## 类：`TextDecoder`

<!-- YAML
added: v11.0.0
-->

WHATWG `TextDecoder` 类。参见 [`TextDecoder`][] 章节。

## 类：`TextDecoderStream`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`TextDecoderStream`][] 的浏览器兼容实现。

## 类：`TextEncoder`

<!-- YAML
added: v11.0.0
-->

WHATWG `TextEncoder` 类。参见 [`TextEncoder`][] 章节。

## 类：`TextEncoderStream`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`TextEncoderStream`][] 的浏览器兼容实现。

## 类：`TransformStream`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`TransformStream`][] 的浏览器兼容实现。

## 类：`TransformStreamDefaultController`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`TransformStreamDefaultController`][] 的浏览器兼容实现。

## 类：`URL`

<!-- YAML
added: v10.0.0
-->

WHATWG `URL` 类。参见 [`URL`][] 章节。

## 类：`URLPattern`

<!-- YAML
added: v24.0.0
-->

> Stability: 1 - Experimental

WHATWG `URLPattern` 类。参见 [`URLPattern`][] 章节。

## 类：`URLSearchParams`

<!-- YAML
added: v10.0.0
-->

WHATWG `URLSearchParams` 类。参见 [`URLSearchParams`][] 章节。

## 类：`WebAssembly`

<!-- YAML
added: v8.0.0
-->

* 类型：{Object}

作为所有 W3C [WebAssembly][webassembly-org] 相关功能命名空间的对象。有关用法和兼容性，请参见 [Mozilla Developer Network][webassembly-mdn]。

## 类：`WebSocket`

<!-- YAML
added:
  - v21.0.0
  - v20.10.0
changes:
  - version: v22.4.0
    pr-url: https://github.com/nodejs/node/pull/53352
    description: No longer experimental.
  - version: v22.0.0
    pr-url: https://github.com/nodejs/node/pull/51594
    description: No longer behind `--experimental-websocket` CLI flag.
-->

{WebSocket} 的浏览器兼容实现。使用 [`--no-experimental-websocket`][] CLI 标志可以禁用此 API。

## 类：`WritableStream`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`WritableStream`][] 的浏览器兼容实现。

## 类：`WritableStreamDefaultController`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`WritableStreamDefaultController`][] 的浏览器兼容实现。

## 类：`WritableStreamDefaultWriter`

<!-- YAML
added: v18.0.0
changes:
 - version:
    - v23.11.0
    - v22.15.0
   pr-url: https://github.com/nodejs/node/pull/57510
   description: Marking the API stable.
-->

[`WritableStreamDefaultWriter`][] 的浏览器兼容实现。

[CommonJS module]: modules.md
[CommonJS modules]: modules.md
[ECMAScript module]: esm.md
[Navigator API]: https://html.spec.whatwg.org/multipage/system-state.html#the-navigator-object
[RFC 5646]: https://www.rfc-editor.org/rfc/rfc5646.txt
[Web Crypto API]: webcrypto.md
[`--experimental-eventsource`]: cli.md#--experimental-eventsource
[`--experimental-webstorage`]: cli.md#--experimental-webstorage
[`--localstorage-file`]: cli.md#--localstorage-filefile
[`--no-experimental-global-navigator`]: cli.md#--no-experimental-global-navigator
[`--no-experimental-websocket`]: cli.md#--no-experimental-websocket
[`ByteLengthQueuingStrategy`]: webstreams.md#class-bytelengthqueuingstrategy
[`CompressionStream`]: webstreams.md#class-compressionstream
[`CountQueuingStrategy`]: webstreams.md#class-countqueuingstrategy
[`DecompressionStream`]: webstreams.md#class-decompressionstream
[`EventTarget` and `Event` API]: events.md#eventtarget-and-event-api
[`LockManager`]: worker_threads.md#class-lockmanager
[`MessageChannel`]: worker_threads.md#class-messagechannel
[`MessagePort`]: worker_threads.md#class-messageport
[`PerformanceEntry`]: perf_hooks.md#class-performanceentry
[`PerformanceMark`]: perf_hooks.md#class-performancemark
[`PerformanceMeasure`]: perf_hooks.md#class-performancemeasure
[`PerformanceObserverEntryList`]: perf_hooks.md#class-performanceobserverentrylist
[`PerformanceObserver`]: perf_hooks.md#class-performanceobserver
[`PerformanceResourceTiming`]: perf_hooks.md#class-performanceresourcetiming
[`ReadableByteStreamController`]: webstreams.md#class-readablebytestreamcontroller
[`ReadableStreamBYOBReader`]: webstreams.md#class-readablestreambyobreader
[`ReadableStreamBYOBRequest`]: webstreams.md#class-readablestreambyobrequest
[`ReadableStreamDefaultController`]: webstreams.md#class-readablestreamdefaultcontroller
[`ReadableStreamDefaultReader`]: webstreams.md#class-readablestreamdefaultreader
[`ReadableStream`]: webstreams.md#class-readablestream
[`TextDecoderStream`]: webstreams.md#class-textdecoderstream
[`TextDecoder`]: util.md#class-utiltextdecoder
[`TextEncoderStream`]: webstreams.md#class-textencoderstream
[`TextEncoder`]: util.md#class-utiltextencoder
[`TransformStreamDefaultController`]: webstreams.md#class-transformstreamdefaultcontroller
[`TransformStream`]: webstreams.md#class-transformstream
[`URLPattern`]: url.md#class-urlpattern
[`URLSearchParams`]: url.md#class-urlsearchparams
[`URL`]: url.md#class-url
[`WritableStreamDefaultController`]: webstreams.md#class-writablestreamdefaultcontroller
[`WritableStreamDefaultWriter`]: webstreams.md#class-writablestreamdefaultwriter
[`WritableStream`]: webstreams.md#class-writablestream
[`__dirname`]: modules.md#__dirname
[`__filename`]: modules.md#__filename
[`abortSignal.reason`]: #abortsignalreason
[`buffer.atob()`]: buffer.md#bufferatobdata
[`buffer.btoa()`]: buffer.md#bufferbtoadata
[`clearImmediate`]: timers.md#clearimmediateimmediate
[`clearInterval`]: timers.md#clearintervaltimeout
[`clearTimeout`]: timers.md#cleartimeouttimeout
[`console`]: console.md
[`exports`]: modules.md#exports
[`fetch()`]: https://developer.mozilla.org/en-US/docs/Web/API/fetch
[`globalThis`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/globalThis
[`localStorage`]: https://developer.mozilla.org/en-US/docs/Web/API/Window/localStorage
[`module`]: modules.md#module
[`perf_hooks.performance`]: perf_hooks.md#perf_hooksperformance
[`process.nextTick()`]: process.md#processnexttickcallback-args
[`process` object]: process.md#process
[`require()`]: modules.md#requireid
[`sessionStorage`]: https://developer.mozilla.org/en-US/docs/Web/API/Window/sessionStorage
[`setImmediate`]: timers.md#setimmediatecallback-args
[`setInterval`]: timers.md#setintervalcallback-delay-args
[`setTimeout`]: timers.md#settimeoutcallback-delay-args
[`structuredClone`]: https://developer.mozilla.org/en-US/docs/Web/API/structuredClone
[`window.navigator`]: https://developer.mozilla.org/en-US/docs/Web/API/Window/navigator
[`worker.locks`]: worker_threads.md#workerlocks
[browser `LockManager`]: https://developer.mozilla.org/en-US/docs/Web/API/LockManager
[buffer section]: buffer.md
[built-in objects]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects
[timers]: timers.md
[webassembly-mdn]: https://developer.mozilla.org/en-US/docs/WebAssembly
[webassembly-org]: https://webassembly.org