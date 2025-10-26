# Timers

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/timers.js -->

`timer` 模块暴露了一个全局 API，用于调度在未来的某个时间点调用的函数。因为定时器函数是全局的，所以不需要调用 `require('node:timers')` 来使用该 API。

Node.js 中的定时器函数实现了一个与 Web 浏览器提供的定时器 API 类似的 API，但使用了一个围绕 Node.js [事件循环][]构建的不同内部实现。

## 类: `Immediate`

此对象在内部创建，并从 [`setImmediate()`][] 返回。它可以传递给 [`clearImmediate()`][] 以取消已调度的操作。

默认情况下，当调度了一个 Immediate 时，只要该 Immediate 是活跃的，Node.js 事件循环将继续运行。由 [`setImmediate()`][] 返回的 `Immediate` 对象导出了 `immediate.ref()` 和 `immediate.unref()` 函数，可用于控制此默认行为。

### `immediate.hasRef()`

<!-- YAML
added: v11.0.0
-->

* 返回: {boolean}

如果为 true，则 `Immediate` 对象将保持 Node.js 事件循环活跃。

### `immediate.ref()`

<!-- YAML
added: v9.7.0
-->

* 返回: {Immediate} 对 `immediate` 的引用

调用时，只要 `Immediate` 是活跃的，就请求 Node.js 事件循环*不要*退出。多次调用 `immediate.ref()` 将没有效果。

默认情况下，所有 `Immediate` 对象都是 "ref'ed" 的，这使得通常不需要调用 `immediate.ref()`，除非之前调用过 `immediate.unref()`。

### `immediate.unref()`

<!-- YAML
added: v9.7.0
-->

* 返回: {Immediate} 对 `immediate` 的引用

调用时，活跃的 `Immediate` 对象将不要求 Node.js 事件循环保持活跃。如果没有其他活动保持事件循环运行，进程可能会在 `Immediate` 对象的回调被调用之前退出。多次调用 `immediate.unref()` 将没有效果。

### `immediate[Symbol.dispose]()`

<!-- YAML
added:
 - v20.5.0
 - v18.18.0
changes:
 - version: v24.2.0
   pr-url: https://github.com/nodejs/node/pull/58467
   description: No longer experimental.
-->

取消 immediate。这类似于调用 `clearImmediate()`。

## 类: `Timeout`

此对象在内部创建，并从 [`setTimeout()`][] 和 [`setInterval()`][] 返回。它可以传递给 [`clearTimeout()`][] 或 [`clearInterval()`][] 以取消已调度的操作。

默认情况下，当使用 [`setTimeout()`][] 或 [`setInterval()`][] 调度定时器时，只要定时器是活跃的，Node.js 事件循环将继续运行。这些函数返回的每个 `Timeout` 对象都导出了 `timeout.ref()` 和 `timeout.unref()` 函数，可用于控制此默认行为。

### `timeout.close()`

<!-- YAML
added: v0.9.1
-->

> Stability: 3 - Legacy: 使用 [`clearTimeout()`][] 代替。

* 返回: {Timeout} 对 `timeout` 的引用

取消超时。

### `timeout.hasRef()`

<!-- YAML
added: v11.0.0
-->

* 返回: {boolean}

如果为 true，则 `Timeout` 对象将保持 Node.js 事件循环活跃。

### `timeout.ref()`

<!-- YAML
added: v0.9.1
-->

* 返回: {Timeout} 对 `timeout` 的引用

调用时，只要 `Timeout` 是活跃的，就请求 Node.js 事件循环*不要*退出。多次调用 `timeout.ref()` 将没有效果。

默认情况下，所有 `Timeout` 对象都是 "ref'ed" 的，这使得通常不需要调用 `timeout.ref()`，除非之前调用过 `timeout.unref()`。

### `timeout.refresh()`

<!-- YAML
added: v10.2.0
-->

* 返回: {Timeout} 对 `timeout` 的引用

将定时器的开始时间设置为当前时间，并重新调度定时器，以在当前时间调整后的先前指定持续时间调用其回调。这对于在不分配新的 JavaScript 对象的情况下刷新定时器非常有用。

在已经调用过其回调的定时器上使用此方法将重新激活定时器。

### `timeout.unref()`

<!-- YAML
added: v0.9.1
-->

* 返回: {Timeout} 对 `timeout` 的引用

调用时，活跃的 `Timeout` 对象将不要求 Node.js 事件循环保持活跃。如果没有其他活动保持事件循环运行，进程可能会在 `Timeout` 对象的回调被调用之前退出。多次调用 `timeout.unref()` 将没有效果。

### `timeout[Symbol.toPrimitive]()`

<!-- YAML
added:
  - v14.9.0
  - v12.19.0
-->

* 返回: {integer} 一个可用于引用此 `timeout` 的数字

将 `Timeout` 强制转换为原始值。该原始值可用于清除 `Timeout`。该原始值只能在创建超时的同一线程中使用。因此，要在 [`worker_threads`][] 中使用它，必须首先将其传递到正确的线程。这增强了与浏览器 `setTimeout()` 和 `setInterval()` 实现的兼容性。

### `timeout[Symbol.dispose]()`

<!-- YAML
added:
 - v20.5.0
 - v18.18.0
changes:
 - version: v24.2.0
   pr-url: https://github.com/nodejs/node/pull/58467
   description: No longer experimental.
-->

取消超时。

## 调度定时器

Node.js 中的定时器是一种内部构造，它在一定时间后调用给定的函数。定时器函数的调用时间取决于用于创建定时器的方法以及 Node.js 事件循环正在执行的其他工作。

### `setImmediate(callback[, ...args])`

<!-- YAML
added: v0.9.1
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `callback` {Function} 在 Node.js [事件循环][]的当前回合结束时调用的函数
* `...args` {any} 调用 `callback` 时传递的可选参数
* 返回: {Immediate} 用于 [`clearImmediate()`][]

调度 `callback` 在 I/O 事件回调之后"立即"执行。

当多次调用 `setImmediate()` 时，`callback` 函数按照它们被创建的顺序排队执行。整个回调队列在每个事件循环迭代中处理。如果从正在执行的回调内部排队了一个 immediate 定时器，则该定时器直到下一个事件循环迭代才会触发。

如果 `callback` 不是函数，将抛出 [`TypeError`][]。

此方法有一个用于 promise 的自定义变体，可通过 [`timersPromises.setImmediate()`][] 使用。

### `setInterval(callback[, delay[, ...args]])`

<!-- YAML
added: v0.0.1
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `callback` {Function} 定时器到期时调用的函数
* `delay` {number} 在调用 `callback` 之前要等待的毫秒数。**默认值:** `1`
* `...args` {any} 调用 `callback` 时传递的可选参数
* 返回: {Timeout} 用于 [`clearInterval()`][]

每隔 `delay` 毫秒调度重复执行 `callback`。

当 `delay` 大于 `2147483647` 或小于 `1` 或 `NaN` 时，`delay` 将被设置为 `1`。非整数延迟会被截断为整数。

如果 `callback` 不是函数，将抛出 [`TypeError`][]。

此方法有一个用于 promise 的自定义变体，可通过 [`timersPromises.setInterval()`][] 使用。

### `setTimeout(callback[, delay[, ...args]])`

<!-- YAML
added: v0.0.1
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `callback` {Function} 定时器到期时调用的函数
* `delay` {number} 在调用 `callback` 之前要等待的毫秒数。**默认值:** `1`
* `...args` {any} 调用 `callback` 时传递的可选参数
* 返回: {Timeout} 用于 [`clearTimeout()`][]

在 `delay` 毫秒后调度执行一次性的 `callback`。

`callback` 可能不会在精确的 `delay` 毫秒后被调用。Node.js 不保证回调触发的确切时间，也不保证它们的顺序。回调将尽可能接近指定的时间调用。

当 `delay` 大于 `2147483647` 或小于 `1` 或 `NaN` 时，`delay` 将被设置为 `1`。非整数延迟会被截断为整数。

如果 `callback` 不是函数，将抛出 [`TypeError`][]。

此方法有一个用于 promise 的自定义变体，可通过 [`timersPromises.setTimeout()`][] 使用。

## 取消定时器

[`setImmediate()`][]、[`setInterval()`][] 和 [`setTimeout()`][] 方法各自返回代表已调度定时器的对象。这些对象可用于取消定时器并防止其触发。

对于 [`setImmediate()`][] 和 [`setTimeout()`][] 的 promise 化变体，可以使用 [`AbortController`][] 来取消定时器。当取消时，返回的 Promise 将被拒绝，并带有一个 `'AbortError'`。

对于 `setImmediate()`：

```mjs
import { setImmediate as setImmediatePromise } from 'node:timers/promises';

const ac = new AbortController();
const signal = ac.signal;

// We do not `await` the promise so `ac.abort()` is called concurrently.
setImmediatePromise('foobar', { signal })
  .then(console.log)
  .catch((err) => {
    if (err.name === 'AbortError')
      console.error('The immediate was aborted');
  });

ac.abort();
```

```cjs
const { setImmediate: setImmediatePromise } = require('node:timers/promises');

const ac = new AbortController();
const signal = ac.signal;

setImmediatePromise('foobar', { signal })
  .then(console.log)
  .catch((err) => {
    if (err.name === 'AbortError')
      console.error('The immediate was aborted');
  });

ac.abort();
```

对于 `setTimeout()`：

```mjs
import { setTimeout as setTimeoutPromise } from 'node:timers/promises';

const ac = new AbortController();
const signal = ac.signal;

// We do not `await` the promise so `ac.abort()` is called concurrently.
setTimeoutPromise(1000, 'foobar', { signal })
  .then(console.log)
  .catch((err) => {
    if (err.name === 'AbortError')
      console.error('The timeout was aborted');
  });

ac.abort();
```

```cjs
const { setTimeout: setTimeoutPromise } = require('node:timers/promises');

const ac = new AbortController();
const signal = ac.signal;

setTimeoutPromise(1000, 'foobar', { signal })
  .then(console.log)
  .catch((err) => {
    if (err.name === 'AbortError')
      console.error('The timeout was aborted');
  });

ac.abort();
```

### `clearImmediate(immediate)`

<!-- YAML
added: v0.9.1
-->

* `immediate` {Immediate} 一个由 [`setImmediate()`][] 返回的 `Immediate` 对象

取消由 [`setImmediate()`][] 创建的 `Immediate` 对象。

### `clearInterval(timeout)`

<!-- YAML
added: v0.0.1
-->

* `timeout` {Timeout|string|number} 一个由 [`setInterval()`][] 返回的 `Timeout` 对象，或者是 `Timeout` 对象的[原始值][]（作为字符串或数字）

取消由 [`setInterval()`][] 创建的 `Timeout` 对象。

### `clearTimeout(timeout)`

<!-- YAML
added: v0.0.1
-->

* `timeout` {Timeout|string|number} 一个由 [`setTimeout()`][] 返回的 `Timeout` 对象，或者是 `Timeout` 对象的[原始值][]（作为字符串或数字）

取消由 [`setTimeout()`][] 创建的 `Timeout` 对象。

## Timers Promises API

<!-- YAML
added: v15.0.0
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/38112
    description: Graduated from experimental.
-->

`timers/promises` API 提供了一组返回 `Promise` 对象的定时器函数。该 API 可通过 `require('node:timers/promises')` 访问。

```mjs
import {
  setTimeout,
  setImmediate,
  setInterval,
} from 'node:timers/promises';
```

```cjs
const {
  setTimeout,
  setImmediate,
  setInterval,
} = require('node:timers/promises');
```

### `timersPromises.setTimeout([delay[, value[, options]]])`

<!-- YAML
added: v15.0.0
-->

* `delay` {number} 在履行 promise 之前要等待的毫秒数。**默认值:** `1`
* `value` {any} 用于履行 promise 的值
* `options` {Object}
  * `ref` {boolean} 设置为 `false` 表示已调度的 `Timeout` 不应要求 Node.js 事件循环保持活跃。**默认值:** `true`
  * `signal` {AbortSignal} 一个可选的 `AbortSignal`，可用于取消已调度的 `Timeout`

```mjs
import {
  setTimeout,
} from 'node:timers/promises';

const res = await setTimeout(100, 'result');

console.log(res);  // Prints 'result'
```

```cjs
const {
  setTimeout,
} = require('node:timers/promises');

setTimeout(100, 'result').then((res) => {
  console.log(res);  // Prints 'result'
});
```

### `timersPromises.setImmediate([value[, options]])`

<!-- YAML
added: v15.0.0
-->

* `value` {any} 用于履行 promise 的值
* `options` {Object}
  * `ref` {boolean} 设置为 `false` 表示已调度的 `Immediate` 不应要求 Node.js 事件循环保持活跃。**默认值:** `true`
  * `signal` {AbortSignal} 一个可选的 `AbortSignal`，可用于取消已调度的 `Immediate`

```mjs
import {
  setImmediate,
} from 'node:timers/promises';

const res = await setImmediate('result');

console.log(res);  // Prints 'result'
```

```cjs
const {
  setImmediate,
} = require('node:timers/promises');

setImmediate('result').then((res) => {
  console.log(res);  // Prints 'result'
});
```

### `timersPromises.setInterval([delay[, value[, options]]])`

<!-- YAML
added: v15.9.0
-->

返回一个异步迭代器，该迭代器每隔 `delay` 毫秒生成值。如果 `ref` 为 `true`，则需要显式或隐式调用异步迭代器的 `next()` 来保持事件循环活跃。

* `delay` {number} 迭代之间等待的毫秒数。**默认值:** `1`
* `value` {any} 迭代器返回的值
* `options` {Object}
  * `ref` {boolean} 设置为 `false` 表示迭代之间的已调度 `Timeout` 不应要求 Node.js 事件循环保持活跃。**默认值:** `true`
  * `signal` {AbortSignal} 一个可选的 `AbortSignal`，可用于取消操作之间的已调度 `Timeout`

```mjs
import {
  setInterval,
} from 'node:timers/promises';

const interval = 100;
for await (const startTime of setInterval(interval, Date.now())) {
  const now = Date.now();
  console.log(now);
  if ((now - startTime) > 1000)
    break;
}
console.log(Date.now());
```

```cjs
const {
  setInterval,
} = require('node:timers/promises');
const interval = 100;

(async function() {
  for await (const startTime of setInterval(interval, Date.now())) {
    const now = Date.now();
    console.log(now);
    if ((now - startTime) > 1000)
      break;
  }
  console.log(Date.now());
})();
```

### `timersPromises.scheduler.wait(delay[, options])`

<!-- YAML
added:
  - v17.3.0
  - v16.14.0
-->

> Stability: 1 - Experimental

* `delay` {number} 在解决 promise 之前要等待的毫秒数
* `options` {Object}
  * `ref` {boolean} 设置为 `false` 表示已调度的 `Timeout` 不应要求 Node.js 事件循环保持活跃。**默认值:** `true`
  * `signal` {AbortSignal} 一个可选的 `AbortSignal`，可用于取消等待
* 返回: {Promise}

由[调度 APIs][]草案规范定义的一个实验性 API，该规范正在作为标准 Web 平台 API 开发。

调用 `timersPromises.scheduler.wait(delay, options)` 等同于调用 `timersPromises.setTimeout(delay, undefined, options)`。

```mjs
import { scheduler } from 'node:timers/promises';

await scheduler.wait(1000); // Wait one second before continuing
```

### `timersPromises.scheduler.yield()`

<!-- YAML
added:
  - v17.3.0
  - v16.14.0
-->

> Stability: 1 - Experimental

* 返回: {Promise}

由[调度 APIs][]草案规范定义的一个实验性 API，该规范正在作为标准 Web 平台 API 开发。

调用 `timersPromises.scheduler.yield()` 等同于调用不带参数的 `timersPromises.setImmediate()`。

[Event Loop]: https://nodejs.org/en/docs/guides/event-loop-timers-and-nexttick/#setimmediate-vs-settimeout
[Scheduling APIs]: https://github.com/WICG/scheduling-apis
[`AbortController`]: globals.md#class-abortcontroller
[`TypeError`]: errors.md#class-typeerror
[`clearImmediate()`]: #clearimmediateimmediate
[`clearInterval()`]: #clearintervaltimeout
[`clearTimeout()`]: #cleartimeouttimeout
[`setImmediate()`]: #setimmediatecallback-args
[`setInterval()`]: #setintervalcallback-delay-args
[`setTimeout()`]: #settimeoutcallback-delay-args
[`timersPromises.setImmediate()`]: #timerspromisessetimmediatevalue-options
[`timersPromises.setInterval()`]: #timerspromisessetintervaldelay-value-options
[`timersPromises.setTimeout()`]: #timerspromisessettimeoutdelay-value-options
[`worker_threads`]: worker_threads.md
[primitive]: #timeoutsymboltoprimitive
