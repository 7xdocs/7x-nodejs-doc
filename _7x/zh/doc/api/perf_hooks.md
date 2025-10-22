# Performance measurement APIs

<!--introduced_in=v8.5.0-->

> Stability: 2 - Stable

<!-- source_link=lib/perf_hooks.js -->

此模块实现了 W3C [Web Performance APIs][] 的子集以及用于 Node.js 特定性能测量的额外 API。

Node.js 支持以下 [Web Performance APIs][]：

* [High Resolution Time][]
* [Performance Timeline][]
* [User Timing][]
* [Resource Timing][]

```mjs
import { performance, PerformanceObserver } from 'node:perf_hooks';

const obs = new PerformanceObserver((items) => {
  console.log(items.getEntries()[0].duration);
  performance.clearMarks();
});
obs.observe({ type: 'measure' });
performance.measure('Start to Now');

performance.mark('A');
doSomeLongRunningProcess(() => {
  performance.measure('A to Now', 'A');

  performance.mark('B');
  performance.measure('A to B', 'A', 'B');
});
```

```cjs
const { PerformanceObserver, performance } = require('node:perf_hooks');

const obs = new PerformanceObserver((items) => {
  console.log(items.getEntries()[0].duration);
});
obs.observe({ type: 'measure' });
performance.measure('Start to Now');

performance.mark('A');
(async function doSomeLongRunningProcess() {
  await new Promise((r) => setTimeout(r, 5000));
  performance.measure('A to Now', 'A');

  performance.mark('B');
  performance.measure('A to B', 'A', 'B');
})();
```

## `perf_hooks.performance`

<!-- YAML
added: v8.5.0
-->

一个可用于从当前 Node.js 实例收集性能指标的对象。它类似于浏览器中的 [`window.performance`][]。

### `performance.clearMarks([name])`

<!-- YAML
added: v8.5.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
-->

* `name` {string}

如果未提供 `name`，则从性能时间线中移除所有 `PerformanceMark` 对象。如果提供了 `name`，则仅移除指定名称的标记。

### `performance.clearMeasures([name])`

<!-- YAML
added: v16.7.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
-->

* `name` {string}

如果未提供 `name`，则从性能时间线中移除所有 `PerformanceMeasure` 对象。如果提供了 `name`，则仅移除指定名称的测量。

### `performance.clearResourceTimings([name])`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
-->

* `name` {string}

如果未提供 `name`，则从资源时间线中移除所有 `PerformanceResourceTiming` 对象。如果提供了 `name`，则仅移除指定名称的资源。

### `performance.eventLoopUtilization([utilization1[, utilization2]])`

<!-- YAML
added:
 - v14.10.0
 - v12.19.0
-->

* `utilization1` {Object} 之前调用 `eventLoopUtilization()` 的结果。
* `utilization2` {Object} 在 `utilization1` 之前调用 `eventLoopUtilization()` 的结果。
* 返回: {Object}
  * `idle` {number}
  * `active` {number}
  * `utilization` {number}

`eventLoopUtilization()` 方法返回一个对象，其中包含事件循环在空闲和活动状态下的累计持续时间，以高精度毫秒计时器表示。`utilization` 值是计算得到的事件循环利用率（ELU）。

如果主线程上的引导尚未完成，则属性的值为 `0`。ELU 在 [Worker 线程][]上立即可用，因为引导发生在事件循环内。

`utilization1` 和 `utilization2` 都是可选参数。

如果传入了 `utilization1`，则计算并返回当前调用的 `active` 和 `idle` 时间之间的差值，以及相应的 `utilization` 值（类似于 [`process.hrtime()`][]）。

如果同时传入了 `utilization1` 和 `utilization2`，则计算两个参数之间的差值。这是一个便捷选项，因为与 [`process.hrtime()`][] 不同，计算 ELU 比单个减法更复杂。

ELU 类似于 CPU 利用率，不同之处在于它只测量事件循环统计信息而不测量 CPU 使用情况。它表示事件循环在事件循环的事件提供者（例如 `epoll_wait`）之外花费的时间百分比。不考虑其他 CPU 空闲时间。以下是一个示例，说明一个大部分空闲的进程将具有较高的 ELU。

```mjs
import { eventLoopUtilization } from 'node:perf_hooks';
import { spawnSync } from 'node:child_process';

setImmediate(() => {
  const elu = eventLoopUtilization();
  spawnSync('sleep', ['5']);
  console.log(eventLoopUtilization(elu).utilization);
});
```

```cjs
'use strict';
const { eventLoopUtilization } = require('node:perf_hooks').performance;
const { spawnSync } = require('node:child_process');

setImmediate(() => {
  const elu = eventLoopUtilization();
  spawnSync('sleep', ['5']);
  console.log(eventLoopUtilization(elu).utilization);
});
```

尽管运行此脚本时 CPU 大部分空闲，但 `utilization` 的值为 `1`。这是因为对 [`child_process.spawnSync()`][] 的调用阻止了事件循环继续。

传入用户定义的对象而不是之前调用 `eventLoopUtilization()` 的结果将导致未定义的行为。返回值不能保证反映事件循环的任何正确状态。

### `performance.getEntries()`

<!-- YAML
added: v16.7.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
-->

* 返回: {PerformanceEntry\[]}

返回一个按 `performanceEntry.startTime` 时间顺序排列的 `PerformanceEntry` 对象列表。如果只对特定类型或具有特定名称的性能条目感兴趣，请参阅 `performance.getEntriesByType()` 和 `performance.getEntriesByName()`。

### `performance.getEntriesByName(name[, type])`

<!-- YAML
added: v16.7.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
-->

* `name` {string}
* `type` {string}
* 返回: {PerformanceEntry\[]}

返回一个按 `performanceEntry.startTime` 时间顺序排列的 `PerformanceEntry` 对象列表，这些对象的 `performanceEntry.name` 等于 `name`，并且可选地，其 `performanceEntry.entryType` 等于 `type`。

### `performance.getEntriesByType(type)`

<!-- YAML
added: v16.7.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
-->

* `type` {string}
* 返回: {PerformanceEntry\[]}

返回一个按 `performanceEntry.startTime` 时间顺序排列的 `PerformanceEntry` 对象列表，这些对象的 `performanceEntry.entryType` 等于 `type`。

### `performance.mark(name[, options])`

<!-- YAML
added: v8.5.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver. The name argument is no longer optional.
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37136
    description: Updated to conform to the User Timing Level 3 specification.
-->

* `name` {string}
* `options` {Object}
  * `detail` {any} 包含在标记中的额外可选详细信息。
  * `startTime` {number} 用作标记时间的可选时间戳。**默认值**: `performance.now()`。

在性能时间线中创建一个新的 `PerformanceMark` 条目。`PerformanceMark` 是 `PerformanceEntry` 的子类，其 `performanceEntry.entryType` 始终为 `'mark'`，且 `performanceEntry.duration` 始终为 `0`。性能标记用于标记性能时间线中的特定重要时刻。

创建的 `PerformanceMark` 条目被放入全局性能时间线中，可以通过 `performance.getEntries`、`performance.getEntriesByName` 和 `performance.getEntriesByType` 进行查询。当观察完成时，应使用 `performance.clearMarks` 手动从全局性能时间线中清除条目。

### `performance.markResourceTiming(timingInfo, requestedUrl, initiatorType, global, cacheMode, bodyInfo, responseStatus[, deliveryType])`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v22.2.0
    pr-url: https://github.com/nodejs/node/pull/51589
    description: Added bodyInfo, responseStatus, and deliveryType arguments.
-->

* `timingInfo` {Object} [Fetch Timing Info][]
* `requestedUrl` {string} 资源 URL
* `initiatorType` {string} 发起者名称，例如：'fetch'
* `global` {Object}
* `cacheMode` {string} 缓存模式必须为空字符串 ('') 或 'local'
* `bodyInfo` {Object} [Fetch Response Body Info][]
* `responseStatus` {number} 响应的状态码
* `deliveryType` {string} 交付类型。**默认值:** `''`。

_此属性是 Node.js 的扩展。它在 Web 浏览器中不可用。_

在资源时间线中创建一个新的 `PerformanceResourceTiming` 条目。`PerformanceResourceTiming` 是 `PerformanceEntry` 的子类，其 `performanceEntry.entryType` 始终为 `'resource'`。性能资源用于标记资源时间线中的时刻。

创建的 `PerformanceMark` 条目被放入全局资源时间线中，可以通过 `performance.getEntries`、`performance.getEntriesByName` 和 `performance.getEntriesByType` 进行查询。当观察完成时，应使用 `performance.clearResourceTimings` 手动从全局性能时间线中清除条目。

### `performance.measure(name[, startMarkOrOptions[, endMark]])`

<!-- YAML
added: v8.5.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37136
    description: Updated to conform to the User Timing Level 3 specification.
  - version:
      - v13.13.0
      - v12.16.3
    pr-url: https://github.com/nodejs/node/pull/32651
    description: Make `startMark` and `endMark` parameters optional.
-->

* `name` {string}
* `startMarkOrOptions` {string|Object} 可选。
  * `detail` {any} 包含在测量中的额外可选详细信息。
  * `duration` {number} 开始和结束时间之间的持续时间。
  * `end` {number|string} 用作结束时间的时间戳，或标识先前记录的标记的字符串。
  * `start` {number|string} 用作开始时间的时间戳，或标识先前记录的标记的字符串。
* `endMark` {string} 可选。如果 `startMarkOrOptions` 是 {Object}，则必须省略。

在性能时间线中创建一个新的 `PerformanceMeasure` 条目。`PerformanceMeasure` 是 `PerformanceEntry` 的子类，其 `performanceEntry.entryType` 始终为 `'measure'`，且 `performanceEntry.duration` 测量自 `startMark` 和 `endMark` 以来经过的毫秒数。

`startMark` 参数可以标识性能时间线中的任何 _现有_ `PerformanceMark`，或者 _可以_ 标识由 `PerformanceNodeTiming` 类提供的任何时间戳属性。如果指定的 `startMark` 不存在，则抛出错误。

可选的 `endMark` 参数必须标识性能时间线中的任何 _现有_ `PerformanceMark` 或由 `PerformanceNodeTiming` 类提供的任何时间戳属性。如果未传递参数，`endMark` 将为 `performance.now()`，否则如果指定的 `endMark` 不存在，将抛出错误。

创建的 `PerformanceMeasure` 条目被放入全局性能时间线中，可以通过 `performance.getEntries`、`performance.getEntriesByName` 和 `performance.getEntriesByType` 进行查询。当观察完成时，应使用 `performance.clearMeasures` 手动从全局性能时间线中清除条目。

### `performance.nodeTiming`

<!-- YAML
added: v8.5.0
-->

* 类型: {PerformanceNodeTiming}

_此属性是 Node.js 的扩展。它在 Web 浏览器中不可用。_

`PerformanceNodeTiming` 类的一个实例，提供特定 Node.js 操作里程碑的性能指标。

### `performance.now()`

<!-- YAML
added: v8.5.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
-->

* 返回: {number}

返回当前高精度毫秒时间戳，其中 0 表示当前 `node` 进程的开始。

### `performance.setResourceTimingBufferSize(maxSize)`

<!-- YAML
added: v18.8.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
-->

将全局性能资源计时缓冲区大小设置为指定数量的 "resource" 类型性能条目对象。

默认情况下，最大缓冲区大小设置为 250。

### `performance.timeOrigin`

<!-- YAML
added: v8.5.0
-->

* 类型: {number}

[`timeOrigin`][] 指定了当前 `node` 进程开始的高精度毫秒时间戳，以 Unix 时间测量。

### `performance.timerify(fn[, options])`

<!-- YAML
added: v8.5.0
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37475
    description: Added the histogram option.
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37136
    description: Re-implemented to use pure-JavaScript and the ability
                 to time async functions.
-->

* `fn` {Function}
* `options` {Object}
  * `histogram` {RecordableHistogram} 使用 `perf_hooks.createHistogram()` 创建的直方图对象，将记录运行时间（以纳秒为单位）。

_此属性是 Node.js 的扩展。它在 Web 浏览器中不可用。_

将一个函数包装在一个新函数中，该新函数测量被包装函数的运行时间。必须将 `PerformanceObserver` 订阅到 `'function'` 事件类型才能访问计时详细信息。

```mjs
import { performance, PerformanceObserver } from 'node:perf_hooks';

function someFunction() {
  console.log('hello world');
}

const wrapped = performance.timerify(someFunction);

const obs = new PerformanceObserver((list) => {
  console.log(list.getEntries()[0].duration);

  performance.clearMarks();
  performance.clearMeasures();
  obs.disconnect();
});
obs.observe({ entryTypes: ['function'] });

// 将创建一个性能时间线条目
wrapped();
```

```cjs
const {
  performance,
  PerformanceObserver,
} = require('node:perf_hooks');

function someFunction() {
  console.log('hello world');
}

const wrapped = performance.timerify(someFunction);

const obs = new PerformanceObserver((list) => {
  console.log(list.getEntries()[0].duration);

  performance.clearMarks();
  performance.clearMeasures();
  obs.disconnect();
});
obs.observe({ entryTypes: ['function'] });

// 将创建一个性能时间线条目
wrapped();
```

如果包装的函数返回一个 promise，则会在 promise 上附加一个 finally 处理程序，并在调用 finally 处理程序时报告持续时间。

### `performance.toJSON()`

<!-- YAML
added: v16.1.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the `performance` object as
                 the receiver.
-->

一个对象，它是 `performance` 对象的 JSON 表示。它类似于浏览器中的 [`window.performance.toJSON`][]。

#### 事件: `'resourcetimingbufferfull'`

<!-- YAML
added: v18.8.0
-->

当全局性能资源计时缓冲区已满时，会触发 `'resourcetimingbufferfull'` 事件。在事件监听器中使用 `performance.setResourceTimingBufferSize()` 调整资源计时缓冲区大小，或使用 `performance.clearResourceTimings()` 清除缓冲区，以允许将更多条目添加到性能时间线缓冲区。

## 类: `PerformanceEntry`

<!-- YAML
added: v8.5.0
-->

此类的构造函数不直接向用户暴露。

### `performanceEntry.duration`

<!-- YAML
added: v8.5.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceEntry` object as the receiver.
-->

* 类型: {number}

此条目经过的总毫秒数。此值并非对所有性能条目类型都有意义。

### `performanceEntry.entryType`

<!-- YAML
added: v8.5.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceEntry` object as the receiver.
-->

* 类型: {string}

性能条目的类型。它可能是以下之一：

* `'dns'`（仅 Node.js）
* `'function'`（仅 Node.js）
* `'gc'`（仅 Node.js）
* `'http2'`（仅 Node.js）
* `'http'`（仅 Node.js）
* `'mark'`（在 Web 上可用）
* `'measure'`（在 Web 上可用）
* `'net'`（仅 Node.js）
* `'node'`（仅 Node.js）
* `'resource'`（在 Web 上可用）

### `performanceEntry.name`

<!-- YAML
added: v8.5.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceEntry` object as the receiver.
-->

* 类型: {string}

性能条目的名称。

### `performanceEntry.startTime`

<!-- YAML
added: v8.5.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceEntry` object as the receiver.
-->

* 类型: {number}

标记性能条目开始时间的高精度毫秒时间戳。

## 类: `PerformanceMark`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
-->

* 扩展: {PerformanceEntry}

暴露通过 `Performance.mark()` 方法创建的标记。

### `performanceMark.detail`

<!-- YAML
added: v16.0.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceMark` object as the receiver.
-->

* 类型: {any}

使用 `Performance.mark()` 方法创建时指定的额外详细信息。

## 类: `PerformanceMeasure`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
-->

* 扩展: {PerformanceEntry}

暴露通过 `Performance.measure()` 方法创建的测量。

此类的构造函数不直接向用户暴露。

### `performanceMeasure.detail`

<!-- YAML
added: v16.0.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceMeasure` object as the receiver.
-->

* 类型: {any}

使用 `Performance.measure()` 方法创建时指定的额外详细信息。

## 类: `PerformanceNodeEntry`

<!-- YAML
added: v19.0.0
-->

* 扩展: {PerformanceEntry}

_此类是 Node.js 的扩展。它在 Web 浏览器中不可用。_

提供详细的 Node.js 计时数据。

此类的构造函数不直接向用户暴露。

### `performanceNodeEntry.detail`

<!-- YAML
added: v16.0.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceNodeEntry` object as the receiver.
-->

* 类型: {any}

特定于 `entryType` 的额外详细信息。

### `performanceNodeEntry.flags`

<!-- YAML
added:
 - v13.9.0
 - v12.17.0
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37136
    description: Runtime deprecated. Now moved to the detail property
                 when entryType is 'gc'.
-->

> 稳定性: 0 - 已弃用：改用 `performanceNodeEntry.detail`。

* 类型: {number}

当 `performanceEntry.entryType` 等于 `'gc'` 时，`performance.flags` 属性包含有关垃圾回收操作的额外信息。该值可能是以下之一：

* `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_NO`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_CONSTRUCT_RETAINED`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_FORCED`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_SYNCHRONOUS_PHANTOM_PROCESSING`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_ALL_AVAILABLE_GARBAGE`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_ALL_EXTERNAL_MEMORY`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_SCHEDULE_IDLE`

### `performanceNodeEntry.kind`

<!-- YAML
added: v8.5.0
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37136
    description: Runtime deprecated. Now moved to the detail property
                 when entryType is 'gc'.
-->

> 稳定性: 0 - 已弃用：改用 `performanceNodeEntry.detail`。

* 类型: {number}

当 `performanceEntry.entryType` 等于 `'gc'` 时，`performance.kind` 属性标识发生的垃圾回收操作的类型。该值可能是以下之一：

* `perf_hooks.constants.NODE_PERFORMANCE_GC_MAJOR`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_MINOR`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_INCREMENTAL`
* `perf_hooks.constants.NODE_PERFORMANCE_GC_WEAKCB`

### 垃圾回收 ('gc') 详细信息

当 `performanceEntry.type` 等于 `'gc'` 时，`performanceNodeEntry.detail` 属性将是一个包含两个属性的 {Object}：

* `kind` {number} 以下之一：
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_MAJOR`
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_MINOR`
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_INCREMENTAL`
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_WEAKCB`
* `flags` {number} 以下之一：
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_NO`
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_CONSTRUCT_RETAINED`
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_FORCED`
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_SYNCHRONOUS_PHANTOM_PROCESSING`
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_ALL_AVAILABLE_GARBAGE`
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_ALL_EXTERNAL_MEMORY`
  * `perf_hooks.constants.NODE_PERFORMANCE_GC_FLAGS_SCHEDULE_IDLE`

### HTTP ('http') 详细信息

当 `performanceEntry.type` 等于 `'http'` 时，`performanceNodeEntry.detail` 属性将是一个包含额外信息的 {Object}。

如果 `performanceEntry.name` 等于 `HttpClient`，则 `detail` 将包含以下属性：`req`、`res`。并且 `req` 属性将是一个包含 `method`、`url`、`headers` 的 {Object}，`res` 属性将是一个包含 `statusCode`、`statusMessage`、`headers` 的 {Object}。

如果 `performanceEntry.name` 等于 `HttpRequest`，则 `detail` 将包含以下属性：`req`、`res`。并且 `req` 属性将是一个包含 `method`、`url`、`headers` 的 {Object}，`res` 属性将是一个包含 `statusCode`、`statusMessage`、`headers` 的 {Object}。

这可能会增加额外的内存开销，应仅用于诊断目的，默认情况下不应在生产中开启。

### HTTP/2 ('http2') 详细信息

当 `performanceEntry.type` 等于 `'http2'` 时，`performanceNodeEntry.detail` 属性将是一个包含额外性能信息的 {Object}。

如果 `performanceEntry.name` 等于 `Http2Stream`，则 `detail` 将包含以下属性：

* `bytesRead` {number} 为此 `Http2Stream` 接收的 `DATA` 帧字节数。
* `bytesWritten` {number} 为此 `Http2Stream` 发送的 `DATA` 帧字节数。
* `id` {number} 关联的 `Http2Stream` 的标识符
* `timeToFirstByte` {number} `PerformanceEntry` `startTime` 与接收第一个 `DATA` 帧之间经过的毫秒数。
* `timeToFirstByteSent` {number} `PerformanceEntry` `startTime` 与发送第一个 `DATA` 帧之间经过的毫秒数。
* `timeToFirstHeader` {number} `PerformanceEntry` `startTime` 与接收第一个头部之间经过的毫秒数。

如果 `performanceEntry.name` 等于 `Http2Session`，则 `detail` 将包含以下属性：

* `bytesRead` {number} 为此 `Http2Session` 接收的字节数。
* `bytesWritten` {number} 为此 `Http2Session` 发送的字节数。
* `framesReceived` {number} `Http2Session` 接收的 HTTP/2 帧数。
* `framesSent` {number} `Http2Session` 发送的 HTTP/2 帧数。
* `maxConcurrentStreams` {number} 在 `Http2Session` 生命周期内同时打开的最大流数。
* `pingRTT` {number} 自传输 `PING` 帧到接收到其确认之间经过的毫秒数。仅当在 `Http2Session` 上发送了 `PING` 帧时才存在。
* `streamAverageDuration` {number} 所有 `Http2Stream` 实例的平均持续时间（以毫秒为单位）。
* `streamCount` {number} `Http2Session` 处理的 `Http2Stream` 实例数。
* `type` {string} `'server'` 或 `'client'` 以标识 `Http2Session` 的类型。

### Timerify ('function') 详细信息

当 `performanceEntry.type` 等于 `'function'` 时，`performanceNodeEntry.detail` 属性将是一个列出定时函数输入参数的 {Array}。

### Net ('net') 详细信息

当 `performanceEntry.type` 等于 `'net'` 时，`performanceNodeEntry.detail` 属性将是一个包含额外信息的 {Object}。

如果 `performanceEntry.name` 等于 `connect`，则 `detail` 将包含以下属性：`host`、`port`。

### DNS ('dns') 详细信息

当 `performanceEntry.type` 等于 `'dns'` 时，`performanceNodeEntry.detail` 属性将是一个包含额外信息的 {Object}。

如果 `performanceEntry.name` 等于 `lookup`，则 `detail` 将包含以下属性：`hostname`、`family`、`hints`、`verbatim`、`addresses`。

如果 `performanceEntry.name` 等于 `lookupService`，则 `detail` 将包含以下属性：`host`、`port`、`hostname`、`service`。

如果 `performanceEntry.name` 等于 `queryxxx` 或 `getHostByAddr`，则 `detail` 将包含以下属性：`host`、`ttl`、`result`。`result` 的值与 `queryxxx` 或 `getHostByAddr` 的结果相同。

## 类: `PerformanceNodeTiming`

<!-- YAML
added: v8.5.0
-->

* 扩展: {PerformanceEntry}

_此属性是 Node.js 的扩展。它在 Web 浏览器中不可用。_

提供 Node.js 本身的计时详细信息。此类的构造函数不向用户暴露。

### `performanceNodeTiming.bootstrapComplete`

<!-- YAML
added: v8.5.0
-->

* 类型: {number}

Node.js 进程完成引导的高精度毫秒时间戳。如果引导尚未完成，则属性的值为 -1。

### `performanceNodeTiming.environment`

<!-- YAML
added: v8.5.0
-->

* 类型: {number}

Node.js 环境初始化的高精度毫秒时间戳。

### `performanceNodeTiming.idleTime`

<!-- YAML
added:
  - v14.10.0
  - v12.19.0
-->

* 类型: {number}

事件循环在事件循环的事件提供者（例如 `epoll_wait`）内空闲的时间的高精度毫秒时间戳。这不考虑 CPU 使用情况。如果事件循环尚未启动（例如，在主脚本的第一个 tick 中），则属性的值为 0。

### `performanceNodeTiming.loopExit`

<!-- YAML
added: v8.5.0
-->

* 类型: {number}

Node.js 事件循环退出的高精度毫秒时间戳。如果事件循环尚未退出，则属性的值为 -1。它只能在 [`'exit'`][] 事件的处理程序中具有非 -1 的值。

### `performanceNodeTiming.loopStart`

<!-- YAML
added: v8.5.0
-->

* 类型: {number}

Node.js 事件循环开始的高精度毫秒时间戳。如果事件循环尚未启动（例如，在主脚本的第一个 tick 中），则属性的值为 -1。

### `performanceNodeTiming.nodeStart`

<!-- YAML
added: v8.5.0
-->

* 类型: {number}

Node.js 进程初始化的高精度毫秒时间戳。

### `performanceNodeTiming.uvMetricsInfo`

<!-- YAML
added:
  - v22.8.0
  - v20.18.0
-->

* 返回: {Object}
  * `loopCount` {number} 事件循环迭代次数。
  * `events` {number} 事件处理程序已处理的事件数。
  * `eventsWaiting` {number} 调用事件提供者时等待处理的事件数。

这是 `uv_metrics_info` 函数的包装器。它返回当前的事件循环指标集。

建议在通过 `setImmediate` 调度的函数内部使用此属性，以避免在当前循环迭代期间完成所有调度的操作之前收集指标。

```cjs
const { performance } = require('node:perf_hooks');

setImmediate(() => {
  console.log(performance.nodeTiming.uvMetricsInfo);
});
```

```mjs
import { performance } from 'node:perf_hooks';

setImmediate(() => {
  console.log(performance.nodeTiming.uvMetricsInfo);
});
```

### `performanceNodeTiming.v8Start`

<!-- YAML
added: v8.5.0
-->

* 类型: {number}

V8 平台初始化的高精度毫秒时间戳。

## 类: `PerformanceResourceTiming`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
-->

* 扩展: {PerformanceEntry}

提供有关应用程序资源加载的详细网络计时数据。

此类的构造函数不直接向用户暴露。

### `performanceResourceTiming.workerStart`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

在立即分发 `fetch` 请求之前的高精度毫秒时间戳。如果资源未被 worker 拦截，则属性将始终返回 0。

### `performanceResourceTiming.redirectStart`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示启动重定向的获取开始时间的高精度毫秒时间戳。

### `performanceResourceTiming.redirectEnd`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

在接收到最后一个重定向响应的最后一个字节后立即创建的高精度毫秒时间戳。

### `performanceResourceTiming.fetchStart`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

在 Node.js 开始获取资源之前的高精度毫秒时间戳。

### `performanceResourceTiming.domainLookupStart`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

在 Node.js 开始对资源进行域名查找之前的高精度毫秒时间戳。

### `performanceResourceTiming.domainLookupEnd`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示 Node.js 完成资源域名查找之后的时间的高精度毫秒时间戳。

### `performanceResourceTiming.connectStart`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示在 Node.js 开始与服务器建立连接以检索资源之前的时间的高精度毫秒时间戳。

### `performanceResourceTiming.connectEnd`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示在 Node.js 完成与服务器建立连接以检索资源之后的时间的高精度毫秒时间戳。

### `performanceResourceTiming.secureConnectionStart`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示在 Node.js 开始握手过程以保护当前连接之前的时间的高精度毫秒时间戳。

### `performanceResourceTiming.requestStart`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示在 Node.js 从服务器接收响应的第一个字节之前的时间的高精度毫秒时间戳。

### `performanceResourceTiming.responseEnd`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示在 Node.js 接收到资源的最后一个字节之后或传输连接关闭之前的时间的高精度毫秒时间戳，以先发生者为准。

### `performanceResourceTiming.transferSize`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示获取资源的大小（以八位字节为单位）的数字。大小包括响应头字段和响应负载体。

### `performanceResourceTiming.encodedBodySize`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示从获取（HTTP 或缓存）接收到的负载体的大小（以八位字节为单位）的数字，在移除任何应用的内容编码之前。

### `performanceResourceTiming.decodedBodySize`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This property getter must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

* 类型: {number}

表示从获取（HTTP 或缓存）接收到的消息体的大小（以八位字节为单位）的数字，在移除任何应用的内容编码之后。

### `performanceResourceTiming.toJSON()`

<!-- YAML
added:
  - v18.2.0
  - v16.17.0
changes:
  - version: v19.0.0
    pr-url: https://github.com/nodejs/node/pull/44483
    description: This method must be called with the
                 `PerformanceResourceTiming` object as the receiver.
-->

返回一个 `object`，它是 `PerformanceResourceTiming` 对象的 JSON 表示。

## 类: `PerformanceObserver`

<!-- YAML
added: v8.5.0
-->

### `PerformanceObserver.supportedEntryTypes`

<!-- YAML
added: v16.0.0
-->

* 类型: {string\[]}

获取支持的类型。

### `new PerformanceObserver(callback)`

<!-- YAML
added: v8.5.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `callback` {Function}
  * `list` {PerformanceObserverEntryList}
  * `observer` {PerformanceObserver}

`PerformanceObserver` 对象在新的 `PerformanceEntry` 实例被添加到性能时间线时提供通知。

```mjs
import { performance, PerformanceObserver } from 'node:perf_hooks';

const obs = new PerformanceObserver((list, observer) => {
  console.log(list.getEntries());

  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ entryTypes: ['mark'], buffered: true });

performance.mark('test');
```

```cjs
const {
  performance,
  PerformanceObserver,
} = require('node:perf_hooks');

const obs = new PerformanceObserver((list, observer) => {
  console.log(list.getEntries());

  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ entryTypes: ['mark'], buffered: true });

performance.mark('test');
```

由于 `PerformanceObserver` 实例会引入它们自己的额外性能开销，因此实例不应无限期地保持订阅通知。用户应在不再需要观察者时立即断开连接。

当 `PerformanceObserver` 被通知有关新的 `PerformanceEntry` 实例时，会调用 `callback`。回调接收一个 `PerformanceObserverEntryList` 实例和一个对 `PerformanceObserver` 的引用。

### `performanceObserver.disconnect()`

<!-- YAML
added: v8.5.0
-->

断开 `PerformanceObserver` 实例与所有通知的连接。

### `performanceObserver.observe(options)`

<!-- YAML
added: v8.5.0
changes:
  - version: v16.7.0
    pr-url: https://github.com/nodejs/node/pull/39297
    description: Updated to conform to Performance Timeline Level 2. The
                 buffered option has been added back.
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/37136
    description: Updated to conform to User Timing Level 3. The
                 buffered option has been removed.
-->

* `options` {Object}
  * `type` {string} 单个 {PerformanceEntry} 类型。如果已指定 `entryTypes`，则不得提供。
  * `entryTypes` {string\[]} 标识观察者感兴趣的 {PerformanceEntry} 实例类型的字符串数组。如果未提供，将抛出错误。
  * `buffered` {boolean} 如果为 true，则使用全局 `PerformanceEntry` 缓冲条目列表调用观察者回调。如果为 false，则仅将时间点之后创建的 `PerformanceEntry` 发送到观察者回调。**默认值:** `false`。

将 {PerformanceObserver} 实例订阅到由 `options.entryTypes` 或 `options.type` 标识的新 {PerformanceEntry} 实例的通知：

```mjs
import { performance, PerformanceObserver } from 'node:perf_hooks';

const obs = new PerformanceObserver((list, observer) => {
  // 异步调用一次。`list` 包含三个项目。
});
obs.observe({ type: 'mark' });

for (let n = 0; n < 3; n++)
  performance.mark(`test${n}`);
```

```cjs
const {
  performance,
  PerformanceObserver,
} = require('node:perf_hooks');

const obs = new PerformanceObserver((list, observer) => {
  // 异步调用一次。`list` 包含三个项目。
});
obs.observe({ type: 'mark' });

for (let n = 0; n < 3; n++)
  performance.mark(`test${n}`);
```

### `performanceObserver.takeRecords()`

<!-- YAML
added: v16.0.0
-->

* 返回: {PerformanceEntry\[]} 存储在性能观察器中的当前条目列表，并将其清空。

## 类: `PerformanceObserverEntryList`

<!-- YAML
added: v8.5.0
-->

`PerformanceObserverEntryList` 类用于提供对传递给 `PerformanceObserver` 的 `PerformanceEntry` 实例的访问。此类的构造函数不向用户暴露。

### `performanceObserverEntryList.getEntries()`

<!-- YAML
added: v8.5.0
-->

* 返回: {PerformanceEntry\[]}

返回一个按 `performanceEntry.startTime` 时间顺序排列的 `PerformanceEntry` 对象列表。

```mjs
import { performance, PerformanceObserver } from 'node:perf_hooks';

const obs = new PerformanceObserver((perfObserverList, observer) => {
  console.log(perfObserverList.getEntries());
  /**
   * [
   *   PerformanceEntry {
   *     name: 'test',
   *     entryType: 'mark',
   *     startTime: 81.465639,
   *     duration: 0,
   *     detail: null
   *   },
   *   PerformanceEntry {
   *     name: 'meow',
   *     entryType: 'mark',
   *     startTime: 81.860064,
   *     duration: 0,
   *     detail: null
   *   }
   * ]
   */

  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ type: 'mark' });

performance.mark('test');
performance.mark('meow');
```

```cjs
const {
  performance,
  PerformanceObserver,
} = require('node:perf_hooks');

const obs = new PerformanceObserver((perfObserverList, observer) => {
  console.log(perfObserverList.getEntries());
  /**
   * [
   *   PerformanceEntry {
   *     name: 'test',
   *     entryType: 'mark',
   *     startTime: 81.465639,
   *     duration: 0,
   *     detail: null
   *   },
   *   PerformanceEntry {
   *     name: 'meow',
   *     entryType: 'mark',
   *     startTime: 81.860064,
   *     duration: 0,
   *     detail: null
   *   }
   * ]
   */

  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ type: 'mark' });

performance.mark('test');
performance.mark('meow');
```

### `performanceObserverEntryList.getEntriesByName(name[, type])`

<!-- YAML
added: v8.5.0
-->

* `name` {string}
* `type` {string}
* 返回: {PerformanceEntry\[]}

返回一个按 `performanceEntry.startTime` 时间顺序排列的 `PerformanceEntry` 对象列表，这些对象的 `performanceEntry.name` 等于 `name`，并且可选地，其 `performanceEntry.entryType` 等于 `type`。

```mjs
import { performance, PerformanceObserver } from 'node:perf_hooks';

const obs = new PerformanceObserver((perfObserverList, observer) => {
  console.log(perfObserverList.getEntriesByName('meow'));
  /**
   * [
   *   PerformanceEntry {
   *     name: 'meow',
   *     entryType: 'mark',
   *     startTime: 98.545991,
   *     duration: 0,
   *     detail: null
   *   }
   * ]
   */
  console.log(perfObserverList.getEntriesByName('nope')); // []

  console.log(perfObserverList.getEntriesByName('test', 'mark'));
  /**
   * [
   *   PerformanceEntry {
   *     name: 'test',
   *     entryType: 'mark',
   *     startTime: 63.518931,
   *     duration: 0,
   *     detail: null
   *   }
   * ]
   */
  console.log(perfObserverList.getEntriesByName('test', 'measure')); // []

  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ entryTypes: ['mark', 'measure'] });

performance.mark('test');
performance.mark('meow');
```

```cjs
const {
  performance,
  PerformanceObserver,
} = require('node:perf_hooks');

const obs = new PerformanceObserver((perfObserverList, observer) => {
  console.log(perfObserverList.getEntriesByName('meow'));
  /**
   * [
   *   PerformanceEntry {
   *     name: 'meow',
   *     entryType: 'mark',
   *     startTime: 98.545991,
   *     duration: 0,
   *     detail: null
   *   }
   * ]
   */
  console.log(perfObserverList.getEntriesByName('nope')); // []

  console.log(perfObserverList.getEntriesByName('test', 'mark'));
  /**
   * [
   *   PerformanceEntry {
   *     name: 'test',
   *     entryType: 'mark',
   *     startTime: 63.518931,
   *     duration: 0,
   *     detail: null
   *   }
   * ]
   */
  console.log(perfObserverList.getEntriesByName('test', 'measure')); // []

  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ entryTypes: ['mark', 'measure'] });

performance.mark('test');
performance.mark('meow');
```

### `performanceObserverEntryList.getEntriesByType(type)`

<!-- YAML
added: v8.5.0
-->

* `type` {string}
* 返回: {PerformanceEntry\[]}

返回一个按 `performanceEntry.startTime` 时间顺序排列的 `PerformanceEntry` 对象列表，这些对象的 `performanceEntry.entryType` 等于 `type`。

```mjs
import { performance, PerformanceObserver } from 'node:perf_hooks';

const obs = new PerformanceObserver((perfObserverList, observer) => {
  console.log(perfObserverList.getEntriesByType('mark'));
  /**
   * [
   *   PerformanceEntry {
   *     name: 'test',
   *     entryType: 'mark',
   *     startTime: 55.897834,
   *     duration: 0,
   *     detail: null
   *   },
   *   PerformanceEntry {
   *     name: 'meow',
   *     entryType: 'mark',
   *     startTime: 56.350146,
   *     duration: 0,
   *     detail: null
   *   }
   * ]
   */
  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ type: 'mark' });

performance.mark('test');
performance.mark('meow');
```

```cjs
const {
  performance,
  PerformanceObserver,
} = require('node:perf_hooks');

const obs = new PerformanceObserver((perfObserverList, observer) => {
  console.log(perfObserverList.getEntriesByType('mark'));
  /**
   * [
   *   PerformanceEntry {
   *     name: 'test',
   *     entryType: 'mark',
   *     startTime: 55.897834,
   *     duration: 0,
   *     detail: null
   *   },
   *   PerformanceEntry {
   *     name: 'meow',
   *     entryType: 'mark',
   *     startTime: 56.350146,
   *     duration: 0,
   *     detail: null
   *   }
   * ]
   */
  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ type: 'mark' });

performance.mark('test');
performance.mark('meow');
```

## `perf_hooks.createHistogram([options])`

<!-- YAML
added:
  - v15.9.0
  - v14.18.0
-->

* `options` {Object}
  * `lowest` {number|bigint} 最低可分辨值。必须是大于 0 的整数值。**默认值:** `1`。
  * `highest` {number|bigint} 最高可记录值。必须是等于或大于 `lowest` 两倍的整数值。**默认值:** `Number.MAX_SAFE_INTEGER`。
  * `figures` {number} 精度位数。必须是介于 `1` 和 `5` 之间的数字。**默认值:** `3`。
* 返回: {RecordableHistogram}

返回一个 {RecordableHistogram}。

## `perf_hooks.monitorEventLoopDelay([options])`

<!-- YAML
added: v11.10.0
-->

* `options` {Object}
  * `resolution` {number} 采样率，以毫秒为单位。必须大于零。**默认值:** `10`。
* 返回: {IntervalHistogram}

_此属性是 Node.js 的扩展。它在 Web 浏览器中不可用。_

创建一个 `IntervalHistogram` 对象，该对象随时间采样和报告事件循环延迟。延迟将以纳秒报告。

使用计时器检测近似事件循环延迟是有效的，因为计时器的执行与 libuv 事件循环的生命周期 specifically 相关。也就是说，循环中的延迟将导致计时器执行的延迟，而此 API 旨在 specifically 检测这些延迟。

```mjs
import { monitorEventLoopDelay } from 'node:perf_hooks';

const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
// 执行某些操作。
h.disable();
console.log(h.min);
console.log(h.max);
console.log(h.mean);
console.log(h.stddev);
console.log(h.percentiles);
console.log(h.percentile(50));
console.log(h.percentile(99));
```

```cjs
const { monitorEventLoopDelay } = require('node:perf_hooks');
const h = monitorEventLoopDelay({ resolution: 20 });
h.enable();
// 执行某些操作。
h.disable();
console.log(h.min);
console.log(h.max);
console.log(h.mean);
console.log(h.stddev);
console.log(h.percentiles);
console.log(h.percentile(50));
console.log(h.percentile(99));
```

## 类: `Histogram`

<!-- YAML
added: v11.10.0
-->

### `histogram.count`

<!-- YAML
added:
  - v17.4.0
  - v16.14.0
-->

* 类型: {number}

直方图记录的样本数。

### `histogram.countBigInt`

<!-- YAML
added:
  - v17.4.0
  - v16.14.0
-->

* 类型: {bigint}

直方图记录的样本数。

### `histogram.exceeds`

<!-- YAML
added: v11.10.0
-->

* 类型: {number}

事件循环延迟超过最大 1 小时事件循环延迟阈值的次数。

### `histogram.exceedsBigInt`

<!-- YAML
added:
  - v17.4.0
  - v16.14.0
-->

* 类型: {bigint}

事件循环延迟超过最大 1 小时事件循环延迟阈值的次数。

### `histogram.max`

<!-- YAML
added: v11.10.0
-->

* 类型: {number}

记录的最大事件循环延迟。

### `histogram.maxBigInt`

<!-- YAML
added:
  - v17.4.0
  - v16.14.0
-->

* 类型: {bigint}

记录的最大事件循环延迟。

### `histogram.mean`

<!-- YAML
added: v11.10.0
-->

* 类型: {number}

记录的事件循环延迟的平均值。

### `histogram.min`

<!-- YAML
added: v11.10.0
-->

* 类型: {number}

记录的最小事件循环延迟。

### `histogram.minBigInt`

<!-- YAML
added:
  - v17.4.0
  - v16.14.0
-->

* 类型: {bigint}

记录的最小事件循环延迟。

### `histogram.percentile(percentile)`

<!-- YAML
added: v11.10.0
-->

* `percentile` {number} 百分位数值，范围在 (0, 100]。
* 返回: {number}

返回给定百分位数的值。

### `histogram.percentileBigInt(percentile)`

<!-- YAML
added:
  - v17.4.0
  - v16.14.0
-->

* `percentile` {number} 百分位数值，范围在 (0, 100]。
* 返回: {bigint}

返回给定百分位数的值。

### `histogram.percentiles`

<!-- YAML
added: v11.10.0
-->

* 类型: {Map}

返回一个 `Map` 对象，详细说明累积的百分位分布。

### `histogram.percentilesBigInt`

<!-- YAML
added:
  - v17.4.0
  - v16.14.0
-->

* 类型: {Map}

返回一个 `Map` 对象，详细说明累积的百分位分布。

### `histogram.reset()`

<!-- YAML
added: v11.10.0
-->

重置收集的直方图数据。

### `histogram.stddev`

<!-- YAML
added: v11.10.0
-->

* 类型: {number}

记录的事件循环延迟的标准差。

## 类: `IntervalHistogram extends Histogram`

一个在给定间隔上定期更新的 `Histogram`。

### `histogram.disable()`

<!-- YAML
added: v11.10.0
-->

* 返回: {boolean}

禁用更新间隔计时器。如果计时器已停止，返回 `true`，如果已经停止，返回 `false`。

### `histogram.enable()`

<!-- YAML
added: v11.10.0
-->

* 返回: {boolean}

启用更新间隔计时器。如果计时器已启动，返回 `true`，如果已经启动，返回 `false`。

### `histogram[Symbol.dispose]()`

<!-- YAML
added: v24.2.0
-->

当直方图被处置时，禁用更新间隔计时器。

```js
const { monitorEventLoopDelay } = require('node:perf_hooks');
{
  using hist = monitorEventLoopDelay({ resolution: 20 });
  hist.enable();
  // 当退出块时，直方图将被禁用。
}
```

### 克隆 `IntervalHistogram`

{IntervalHistogram} 实例可以通过 {MessagePort} 克隆。在接收端，直方图被克隆为一个普通的 {Histogram} 对象，该对象不实现 `enable()` 和 `disable()` 方法。

## 类: `RecordableHistogram extends Histogram`

<!-- YAML
added:
  - v15.9.0
  - v14.18.0
-->

### `histogram.add(other)`

<!-- YAML
added:
  - v17.4.0
  - v16.14.0
-->

* `other` {RecordableHistogram}

将 `other` 中的值添加到此直方图。

### `histogram.record(val)`

<!-- YAML
added:
  - v15.9.0
  - v14.18.0
-->

* `val` {number|bigint} 要记录在直方图中的量。

### `histogram.recordDelta()`

<!-- YAML
added:
  - v15.9.0
  - v14.18.0
-->

计算自上次调用 `recordDelta()` 以来经过的时间（以纳秒为单位），并将该量记录在直方图中。

## 示例

### 测量异步操作的持续时间

以下示例使用 [Async Hooks][] 和 Performance APIs 来测量 Timeout 操作的实际持续时间（包括执行回调所需的时间）。

```mjs
import { createHook } from 'node:async_hooks';
import { performance, PerformanceObserver } from 'node:perf_hooks';

const set = new Set();
const hook = createHook({
  init(id, type) {
    if (type === 'Timeout') {
      performance.mark(`Timeout-${id}-Init`);
      set.add(id);
    }
  },
  destroy(id) {
    if (set.has(id)) {
      set.delete(id);
      performance.mark(`Timeout-${id}-Destroy`);
      performance.measure(`Timeout-${id}`,
                          `Timeout-${id}-Init`,
                          `Timeout-${id}-Destroy`);
    }
  },
});
hook.enable();

const obs = new PerformanceObserver((list, observer) => {
  console.log(list.getEntries()[0]);
  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ entryTypes: ['measure'], buffered: true });

setTimeout(() => {}, 1000);
```

```cjs
'use strict';
const async_hooks = require('node:async_hooks');
const {
  performance,
  PerformanceObserver,
} = require('node:perf_hooks');

const set = new Set();
const hook = async_hooks.createHook({
  init(id, type) {
    if (type === 'Timeout') {
      performance.mark(`Timeout-${id}-Init`);
      set.add(id);
    }
  },
  destroy(id) {
    if (set.has(id)) {
      set.delete(id);
      performance.mark(`Timeout-${id}-Destroy`);
      performance.measure(`Timeout-${id}`,
                          `Timeout-${id}-Init`,
                          `Timeout-${id}-Destroy`);
    }
  },
});
hook.enable();

const obs = new PerformanceObserver((list, observer) => {
  console.log(list.getEntries()[0]);
  performance.clearMarks();
  performance.clearMeasures();
  observer.disconnect();
});
obs.observe({ entryTypes: ['measure'] });

setTimeout(() => {}, 1000);
```

### 测量加载依赖项所需的时间

以下示例测量 `require()` 操作加载依赖项的持续时间：

<!-- eslint-disable no-global-assign -->

```mjs
import { performance, PerformanceObserver } from 'node:perf_hooks';

// 激活观察者
const obs = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach((entry) => {
    console.log(`import('${entry[0]}')`, entry.duration);
  });
  performance.clearMarks();
  performance.clearMeasures();
  obs.disconnect();
});
obs.observe({ entryTypes: ['function'], buffered: true });

const timedImport = performance.timerify(async (module) => {
  return await import(module);
});

await timedImport('some-module');
```

```cjs
'use strict';
const {
  performance,
  PerformanceObserver,
} = require('node:perf_hooks');
const mod = require('node:module');

// 猴子补丁 require 函数
mod.Module.prototype.require =
  performance.timerify(mod.Module.prototype.require);
require = performance.timerify(require);

// 激活观察者
const obs = new PerformanceObserver((list) => {
  const entries = list.getEntries();
  entries.forEach((entry) => {
    console.log(`require('${entry[0]}')`, entry.duration);
  });
  performance.clearMarks();
  performance.clearMeasures();
  obs.disconnect();
});
obs.observe({ entryTypes: ['function'] });

require('some-module');
```

### 测量一次 HTTP 往返所需的时间

以下示例用于跟踪 HTTP 客户端（`OutgoingMessage`）和 HTTP 请求（`IncomingMessage`）所花费的时间。对于 HTTP 客户端，它表示从开始请求到接收响应之间的时间间隔，对于 HTTP 请求，它表示从接收请求到发送响应之间的时间间隔：

```mjs
import { PerformanceObserver } from 'node:perf_hooks';
import { createServer, get } from 'node:http';

const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((item) => {
    console.log(item);
  });
});

obs.observe({ entryTypes: ['http'] });

const PORT = 8080;

createServer((req, res) => {
  res.end('ok');
}).listen(PORT, () => {
  get(`http://127.0.0.1:${PORT}`);
});
```

```cjs
'use strict';
const { PerformanceObserver } = require('node:perf_hooks');
const http = require('node:http');

const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((item) => {
    console.log(item);
  });
});

obs.observe({ entryTypes: ['http'] });

const PORT = 8080;

http.createServer((req, res) => {
  res.end('ok');
}).listen(PORT, () => {
  http.get(`http://127.0.0.1:${PORT}`);
});
```

### 测量 `net.connect`（仅限 TCP）在连接成功时所需的时间

```mjs
import { PerformanceObserver } from 'node:perf_hooks';
import { connect, createServer } from 'node:net';

const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((item) => {
    console.log(item);
  });
});
obs.observe({ entryTypes: ['net'] });
const PORT = 8080;
createServer((socket) => {
  socket.destroy();
}).listen(PORT, () => {
  connect(PORT);
});
```

```cjs
'use strict';
const { PerformanceObserver } = require('node:perf_hooks');
const net = require('node:net');
const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((item) => {
    console.log(item);
  });
});
obs.observe({ entryTypes: ['net'] });
const PORT = 8080;
net.createServer((socket) => {
  socket.destroy();
}).listen(PORT, () => {
  net.connect(PORT);
});
```

### 测量 DNS 在请求成功时所需的时间

```mjs
import { PerformanceObserver } from 'node:perf_hooks';
import { lookup, promises } from 'node:dns';

const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((item) => {
    console.log(item);
  });
});
obs.observe({ entryTypes: ['dns'] });
lookup('localhost', () => {});
promises.resolve('localhost');
```

```cjs
'use strict';
const { PerformanceObserver } = require('node:perf_hooks');
const dns = require('node:dns');
const obs = new PerformanceObserver((items) => {
  items.getEntries().forEach((item) => {
    console.log(item);
  });
});
obs.observe({ entryTypes: ['dns'] });
dns.lookup('localhost', () => {});
dns.promises.resolve('localhost');
```

[Async Hooks]: async_hooks.md
[Fetch Response Body Info]: https://fetch.spec.whatwg.org/#response-body-info
[Fetch Timing Info]: https://fetch.spec.whatwg.org/#fetch-timing-info
[High Resolution Time]: https://www.w3.org/TR/hr-time-2
[Performance Timeline]: https://w3c.github.io/performance-timeline/
[Resource Timing]: https://www.w3.org/TR/resource-timing-2/
[User Timing]: https://www.w3.org/TR/user-timing/
[Web Performance APIs]: https://w3c.github.io/perf-timing-primer/
[Worker threads]: worker_threads.md#worker-threads
[`'exit'`]: process.md#event-exit
[`child_process.spawnSync()`]: child_process.md#child_processspawnsynccommand-args-options
[`process.hrtime()`]: process.md#processhrtimetime
[`timeOrigin`]: https://w3c.github.io/hr-time/#dom-performance-timeorigin
[`window.performance.toJSON`]: https://developer.mozilla.org/en-US/docs/Web/API/Performance/toJSON
[`window.performance`]: https://developer.mozilla.org/en-US/docs/Web/API/Window/performance