# VM (执行 JavaScript)

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!--name=vm-->

<!-- source_link=lib/vm.js -->

`node:vm` 模块支持在 V8 虚拟机上下文中编译和运行代码。

<strong class="critical">`node:vm` 模块不是安全机制。不要用它来运行不受信任的代码。</strong>

JavaScript 代码可以被编译并立即运行，或者编译、保存，然后稍后运行。

一个常见的用例是在不同的 V8 上下文中运行代码。这意味着被调用的代码具有与调用代码不同的全局对象。

可以通过将对象 [_contextifying_][contextified] 来提供上下文。被调用的代码将上下文中的任何属性视为全局变量。被调用代码导致的全局变量的任何更改都会反映在上下文对象中。

```js
const vm = require('node:vm');

const x = 1;

const context = { x: 2 };
vm.createContext(context); // 上下文化对象。

const code = 'x += 40; var y = 17;';
// `x` 和 `y` 是上下文中的全局变量。
// 最初，x 的值为 2，因为这是 context.x 的值。
vm.runInContext(code, context);

console.log(context.x); // 42
console.log(context.y); // 17

console.log(x); // 1; y 未定义。
```

## 类：`vm.Script`

<!-- YAML
added: v0.3.1
-->

`vm.Script` 类的实例包含预编译的脚本，这些脚本可以在特定的上下文中执行。

### `new vm.Script(code[, options])`

<!-- YAML
added: v0.3.1
changes:
  - version:
    - v21.7.0
    - v20.12.0
    pr-url: https://github.com/nodejs/node/pull/51244
    description: Added support for
                `vm.constants.USE_MAIN_CONTEXT_DEFAULT_LOADER`.
  - version:
    - v17.0.0
    - v16.12.0
    pr-url: https://github.com/nodejs/node/pull/40249
    description: Added support for import attributes to the
                 `importModuleDynamically` parameter.
  - version: v10.6.0
    pr-url: https://github.com/nodejs/node/pull/20300
    description: The `produceCachedData` is deprecated in favour of
                 `script.createCachedData()`.
  - version: v5.7.0
    pr-url: https://github.com/nodejs/node/pull/4777
    description: The `cachedData` and `produceCachedData` options are
                 supported now.
-->

* `code` {string} 要编译的 JavaScript 代码。
* `options` {Object|string}
  * `filename` {string} 指定此脚本产生的堆栈跟踪中使用的文件名。**默认值:** `'evalmachine.<anonymous>'`。
  * `lineOffset` {number} 指定此脚本产生的堆栈跟踪中显示的行号偏移量。**默认值:** `0`。
  * `columnOffset` {number} 指定此脚本产生的堆栈跟踪中显示的首行列号偏移量。**默认值:** `0`。
  * `cachedData` {Buffer|TypedArray|DataView} 提供可选的 `Buffer`、`TypedArray` 或 `DataView`，其中包含 V8 为所提供的源代码生成的代码缓存数据。当提供时，`cachedDataRejected` 值将被设置为 `true` 或 `false`，取决于 V8 对数据的接受情况。
  * `produceCachedData` {boolean} 当为 `true` 且不存在 `cachedData` 时，V8 将尝试为 `code` 生成代码缓存数据。成功后，将生成一个包含 V8 代码缓存数据的 `Buffer` 并存储在返回的 `vm.Script` 实例的 `cachedData` 属性中。`cachedDataProduced` 值将根据代码缓存数据是否成功生成而设置为 `true` 或 `false`。此选项**已弃用**，推荐使用 `script.createCachedData()`。**默认值:** `false`。
  * `importModuleDynamically`
    {Function|vm.constants.USE\_MAIN\_CONTEXT\_DEFAULT\_LOADER}
    用于指定在此脚本评估期间调用 `import()` 时应如何加载模块。此选项是实验性模块 API 的一部分。我们不建议在生产环境中使用它。详细信息请参阅 [编译 API 中对动态 `import()` 的支持][]。

如果 `options` 是字符串，则它指定文件名。

创建一个新的 `vm.Script` 对象会编译 `code` 但不会运行它。编译后的 `vm.Script` 可以在以后多次运行。`code` 不绑定到任何全局对象；相反，它在每次运行前绑定，仅针对该次运行。

### `script.cachedDataRejected`

<!-- YAML
added: v5.7.0
-->

* 类型: {boolean|undefined}

当向 `vm.Script` 提供 `cachedData` 时，此值将根据 V8 对数据的接受情况设置为 `true` 或 `false`。否则该值为 `undefined`。

### `script.createCachedData()`

<!-- YAML
added: v10.6.0
-->

* 返回: {Buffer}

创建一个代码缓存，可用于 `Script` 构造函数的 `cachedData` 选项。返回一个 `Buffer`。此方法可以在任何时候调用任意次数。

`Script` 的代码缓存不包含任何 JavaScript 可观察状态。代码缓存可以安全地与脚本源一起保存，并多次用于构建新的 `Script` 实例。

`Script` 源中的函数可以被标记为延迟编译，它们在 `Script` 构造时不会被编译。这些函数将在首次调用时被编译。代码缓存序列化了 V8 当前已知的关于 `Script` 的元数据，它可以用来加速未来的编译。

```js
const script = new vm.Script(`
function add(a, b) {
  return a + b;
}

const x = add(1, 2);
`);

const cacheWithoutAdd = script.createCachedData();
// 在 `cacheWithoutAdd` 中，函数 `add()` 被标记为在调用时完全编译。

script.runInThisContext();

const cacheWithAdd = script.createCachedData();
// `cacheWithAdd` 包含完全编译的函数 `add()`。
```

### `script.runInContext(contextifiedObject[, options])`

<!-- YAML
added: v0.3.1
changes:
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/6635
    description: The `breakOnSigint` option is supported now.
-->

* `contextifiedObject` {Object} 一个由 `vm.createContext()` 方法返回的 [contextified][] 对象。
* `options` {Object}
  * `displayErrors` {boolean} 当为 `true` 时，如果在编译 `code` 时发生 [`Error`][]，导致错误的代码行会附加到堆栈跟踪中。**默认值:** `true`。
  * `timeout` {integer} 指定在执行终止前执行 `code` 的毫秒数。如果执行被终止，将会抛出 [`Error`][]。此值必须是一个严格的正整数。
  * `breakOnSigint` {boolean} 如果为 `true`，接收 `SIGINT` (<kbd>Ctrl</kbd>+<kbd>C</kbd>) 将终止执行并抛出 [`Error`][]。在脚本执行期间，通过 `process.on('SIGINT')` 附加的事件现有处理程序将被禁用，但在之后会继续工作。**默认值:** `false`。
* 返回: {any} 脚本中执行的最后一个语句的结果。

在给定的 `contextifiedObject` 中运行 `vm.Script` 对象包含的已编译代码，并返回结果。运行的代码无法访问局部作用域。

以下示例编译了增加一个全局变量、设置另一个全局变量的值的代码，然后多次执行该代码。全局变量包含在 `context` 对象中。

```js
const vm = require('node:vm');

const context = {
  animal: 'cat',
  count: 2,
};

const script = new vm.Script('count += 1; name = "kitty";');

vm.createContext(context);
for (let i = 0; i < 10; ++i) {
  script.runInContext(context);
}

console.log(context);
// 打印: { animal: 'cat', count: 12, name: 'kitty' }
```

使用 `timeout` 或 `breakOnSigint` 选项将导致启动新的事件循环和相应的线程，这会带来非零的性能开销。

### `script.runInNewContext([contextObject[, options]])`

<!-- YAML
added: v0.3.1
changes:
  - version:
    - v22.8.0
    - v20.18.0
    pr-url: https://github.com/nodejs/node/pull/54394
    description: The `contextObject` argument now accepts `vm.constants.DONT_CONTEXTIFY`.
  - version: v14.6.0
    pr-url: https://github.com/nodejs/node/pull/34023
    description: The `microtaskMode` option is supported now.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19016
    description: The `contextCodeGeneration` option is supported now.
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/6635
    description: The `breakOnSigint` option is supported now.
-->

* `contextObject` {Object|vm.constants.DONT\_CONTEXTIFY|undefined}
  可以是 [`vm.constants.DONT_CONTEXTIFY`][] 或将被 [contextified][] 的对象。如果为 `undefined`，为了向后兼容性，将创建一个空的上下文化对象。
* `options` {Object}
  * `displayErrors` {boolean} 当为 `true` 时，如果在编译 `code` 时发生 [`Error`][]，导致错误的代码行会附加到堆栈跟踪中。**默认值:** `true`。
  * `timeout` {integer} 指定在执行终止前执行 `code` 的毫秒数。如果执行被终止，将会抛出 [`Error`][]。此值必须是一个严格的正整数。
  * `breakOnSigint` {boolean} 如果为 `true`，接收 `SIGINT` (<kbd>Ctrl</kbd>+<kbd>C</kbd>) 将终止执行并抛出 [`Error`][]。在脚本执行期间，通过 `process.on('SIGINT')` 附加的事件现有处理程序将被禁用，但在之后会继续工作。**默认值:** `false`。
  * `contextName` {string} 新创建上下文的人类可读名称。**默认值:** `'VM Context i'`，其中 `i` 是所创建上下文的递增数字索引。
  * `contextOrigin` {string} 对应于新创建上下文的[来源][origin]，用于显示目的。来源应格式化为 URL，但仅包含方案、主机和端口（如果需要），类似于 [`URL`][] 对象的 [`url.origin`][] 属性的值。最值得注意的是，此字符串应省略尾部斜杠，因为那表示路径。**默认值:** `''`。
  * `contextCodeGeneration` {Object}
    * `strings` {boolean} 如果设置为 false，任何对 `eval` 或函数构造函数（`Function`、`GeneratorFunction` 等）的调用都将抛出 `EvalError`。**默认值:** `true`。
    * `wasm` {boolean} 如果设置为 false，任何编译 WebAssembly 模块的尝试都将抛出 `WebAssembly.CompileError`。**默认值:** `true`。
  * `microtaskMode` {string} 如果设置为 `afterEvaluate`，微任务（通过 `Promise` 和 `async function` 调度的任务）将在脚本运行后立即运行。在这种情况下，它们包含在 `timeout` 和 `breakOnSigint` 范围内。
* 返回: {any} 脚本中执行的最后一个语句的结果。

此方法是 `script.runInContext(vm.createContext(options), options)` 的快捷方式。它同时做了几件事：

1. 创建一个新的上下文。
2. 如果 `contextObject` 是一个对象，则使用新上下文对其进行 [contextified][]。如果 `contextObject` 是 undefined，则创建一个新对象并对其进行 [contextified][]。如果 `contextObject` 是 [`vm.constants.DONT_CONTEXTIFY`][]，则不 [contextify][] 任何东西。
3. 在创建的上下文中运行 `vm.Script` 对象包含的已编译代码。代码无法访问此方法被调用的作用域。
4. 返回结果。

以下示例编译了设置全局变量的代码，然后在不同的上下文中多次执行该代码。全局变量设置并包含在每个单独的 `context` 中。

```js
const vm = require('node:vm');

const script = new vm.Script('globalVar = "set"');

const contexts = [{}, {}, {}];
contexts.forEach((context) => {
  script.runInNewContext(context);
});

console.log(contexts);
// 打印: [{ globalVar: 'set' }, { globalVar: 'set' }, { globalVar: 'set' }]

// 如果上下文是从上下文化对象创建的，这将抛出。
// vm.constants.DONT_CONTEXTIFY 允许创建具有普通全局对象的上下文，这些对象可以被冻结。
const freezeScript = new vm.Script('Object.freeze(globalThis); globalThis;');
const frozenContext = freezeScript.runInNewContext(vm.constants.DONT_CONTEXTIFY);
```

### `script.runInThisContext([options])`

<!-- YAML
added: v0.3.1
changes:
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/6635
    description: The `breakOnSigint` option is supported now.
-->

* `options` {Object}
  * `displayErrors` {boolean} 当为 `true` 时，如果在编译 `code` 时发生 [`Error`][]，导致错误的代码行会附加到堆栈跟踪中。**默认值:** `true`。
  * `timeout` {integer} 指定在执行终止前执行 `code` 的毫秒数。如果执行被终止，将会抛出 [`Error`][]。此值必须是一个严格的正整数。
  * `breakOnSigint` {boolean} 如果为 `true`，接收 `SIGINT` (<kbd>Ctrl</kbd>+<kbd>C</kbd>) 将终止执行并抛出 [`Error`][]。在脚本执行期间，通过 `process.on('SIGINT')` 附加的事件现有处理程序将被禁用，但在之后会继续工作。**默认值:** `false`。
* 返回: {any} 脚本中执行的最后一个语句的结果。

在当前 `global` 对象的上下文中运行 `vm.Script` 包含的已编译代码。运行的代码无法访问局部作用域，但*可以*访问当前的 `global` 对象。

以下示例编译了增加一个 `global` 变量的代码，然后多次执行该代码：

```js
const vm = require('node:vm');

global.globalVar = 0;

const script = new vm.Script('globalVar += 1', { filename: 'myfile.vm' });

for (let i = 0; i < 1000; ++i) {
  script.runInThisContext();
}

console.log(globalVar);

// 1000
```

### `script.sourceMapURL`

<!-- YAML
added:
  - v19.1.0
  - v18.13.0
-->

* 类型: {string|undefined}

当脚本从包含 source map 魔术注释的源编译时，此属性将设置为 source map 的 URL。

```mjs
import vm from 'node:vm';

const script = new vm.Script(`
function myFunc() {}
//# sourceMappingURL=sourcemap.json
`);

console.log(script.sourceMapURL);
// 打印: sourcemap.json
```

```cjs
const vm = require('node:vm');

const script = new vm.Script(`
function myFunc() {}
//# sourceMappingURL=sourcemap.json
`);

console.log(script.sourceMapURL);
// 打印: sourcemap.json
```

## 类：`vm.Module`

<!-- YAML
added:
 - v13.0.0
 - v12.16.0
-->

> Stability: 1 - Experimental

此功能仅在启用 `--experimental-vm-modules` 命令标志时可用。

`vm.Module` 类提供了一个低级接口，用于在 VM 上下文中使用 ECMAScript 模块。它是 `vm.Script` 类的对应物，紧密反映了 ECMAScript 规范中定义的 [模块记录][]。

然而，与 `vm.Script` 不同，每个 `vm.Module` 对象从创建起就绑定到一个上下文。

使用 `vm.Module` 对象需要三个不同的步骤：创建/解析、链接和评估。以下示例说明了这三个步骤。

此实现处于比 [ECMAScript 模块加载器][] 更低的级别。目前还没有办法与加载器交互，不过支持已在计划中。

```mjs
import vm from 'node:vm';

const contextifiedObject = vm.createContext({
  secret: 42,
  print: console.log,
});

// 步骤 1
//
// 通过构造一个新的 `vm.SourceTextModule` 对象来创建一个模块。这
// 会解析提供的源文本，如果出现任何问题则会抛出 `SyntaxError`。
// 默认情况下，模块在顶级上下文中创建。但在这里，我们
// 指定 `contextifiedObject` 作为此模块所属的上下文。
//
// 这里，我们尝试从模块 "foo" 获取默认导出，并
// 将其放入本地绑定 "secret"。

const rootModule = new vm.SourceTextModule(`
  import s from 'foo';
  s;
  print(s);
`, { context: contextifiedObject });

// 步骤 2
//
// 将此模块的导入依赖"链接"到它。
//
// 通过 `sourceTextModule.moduleRequests` 获取 SourceTextModule 的请求依赖项
// 并解析它们。
//
// 即使没有依赖项的顶级模块也必须显式链接。传递
// 给 `sourceTextModule.linkRequests(modules)` 的数组可以是
// 空的。
//
// 注意：这是一个人为的例子，因为 resolveAndLinkDependencies
// 每次被调用时都会创建一个新的 "foo" 模块。在完整的
// 模块系统中，可能会使用缓存来避免重复模块。

const moduleMap = new Map([
  ['root', rootModule],
]);

function resolveAndLinkDependencies(module) {
  const requestedModules = module.moduleRequests.map((request) => {
    // 在完整的模块系统中，resolveAndLinkDependencies 会
    // 使用模块缓存键 `[specifier, attributes]` 解析模块。
    // 在这个例子中，我们只使用说明符作为键。
    const specifier = request.specifier;

    let requestedModule = moduleMap.get(specifier);
    if (requestedModule === undefined) {
      requestedModule = new vm.SourceTextModule(`
        // "secret" 变量指的是我们在创建上下文时添加到
        // "contextifiedObject" 的全局变量。
        export default secret;
      `, { context: referencingModule.context });
      moduleMap.set(specifier, linkedModule);
      // 同时解析新模块的依赖项。
      resolveAndLinkDependencies(requestedModule);
    }

    return requestedModule;
  });

  module.linkRequests(requestedModules);
}

resolveAndLinkDependencies(rootModule);
rootModule.instantiate();

// 步骤 3
//
// 评估模块。evaluate() 方法返回一个 promise，该 promise 将在
// 模块评估完成后解析。

// 打印 42。
await rootModule.evaluate();
```

```cjs
const vm = require('node:vm');

const contextifiedObject = vm.createContext({
  secret: 42,
  print: console.log,
});

(async () => {
  // 步骤 1
  //
  // 通过构造一个新的 `vm.SourceTextModule` 对象来创建一个模块。这
  // 会解析提供的源文本，如果出现任何问题则会抛出 `SyntaxError`。
  // 默认情况下，模块在顶级上下文中创建。但在这里，我们
  // 指定 `contextifiedObject` 作为此模块所属的上下文。
  //
  // 这里，我们尝试从模块 "foo" 获取默认导出，并
  // 将其放入本地绑定 "secret"。

  const rootModule = new vm.SourceTextModule(`
    import s from 'foo';
    s;
    print(s);
  `, { context: contextifiedObject });

  // 步骤 2
  //
  // 将此模块的导入依赖"链接"到它。
  //
  // 通过 `sourceTextModule.moduleRequests` 获取 SourceTextModule 的请求依赖项
  // 并解析它们。
  //
  // 即使没有依赖项的顶级模块也必须显式链接。传递
  // 给 `sourceTextModule.linkRequests(modules)` 的数组可以是
  // 空的。
  //
  // 注意：这是一个人为的例子，因为 resolveAndLinkDependencies
  // 每次被调用时都会创建一个新的 "foo" 模块。在完整的
  // 模块系统中，可能会使用缓存来避免重复模块。

  const moduleMap = new Map([
    ['root', rootModule],
  ]);

  function resolveAndLinkDependencies(module) {
    const requestedModules = module.moduleRequests.map((request) => {
      // 在完整的模块系统中，resolveAndLinkDependencies 会
      // 使用模块缓存键 `[specifier, attributes]` 解析模块。
      // 在这个例子中，我们只使用说明符作为键。
      const specifier = request.specifier;

      let requestedModule = moduleMap.get(specifier);
      if (requestedModule === undefined) {
        requestedModule = new vm.SourceTextModule(`
          // "secret" 变量指的是我们在创建上下文时添加到
          // "contextifiedObject" 的全局变量。
          export default secret;
        `, { context: referencingModule.context });
        moduleMap.set(specifier, linkedModule);
        // 同时解析新模块的依赖项。
        resolveAndLinkDependencies(requestedModule);
      }

      return requestedModule;
    });

    module.linkRequests(requestedModules);
  }

  resolveAndLinkDependencies(rootModule);
  rootModule.instantiate();

  // 步骤 3
  //
  // 评估模块。evaluate() 方法返回一个 promise，该 promise 将在
  // 模块评估完成后解析。

  // 打印 42。
  await rootModule.evaluate();
})();
```

### `module.error`

* 类型: {any}

如果 `module.status` 是 `'errored'`，此属性包含模块在评估期间抛出的异常。如果状态是其他任何值，访问此属性将导致抛出异常。

值 `undefined` 不能用于没有抛出异常的情况，因为可能与 `throw undefined;` 产生歧义。

对应于 ECMAScript 规范中 [循环模块记录][] 的 `[[EvaluationError]]` 字段。

### `module.evaluate([options])`

* `options` {Object}
  * `timeout` {integer} 指定在终止执行前评估的毫秒数。如果执行被中断，将抛出 [`Error`][]。此值必须是一个严格的正整数。
  * `breakOnSigint` {boolean} 如果为 `true`，接收 `SIGINT` (<kbd>Ctrl</kbd>+<kbd>C</kbd>) 将终止执行并抛出 [`Error`][]。在脚本执行期间，通过 `process.on('SIGINT')` 附加的事件现有处理程序将被禁用，但在之后会继续工作。**默认值:** `false`。
* 返回: {Promise} 成功时使用 `undefined` 完成。

评估模块。

这必须在模块链接后调用；否则它将拒绝。它也可以在模块已经被评估时调用，在这种情况下，如果初始评估成功结束（`module.status` 是 `'evaluated'`），它将不执行任何操作，或者它将重新抛出初始评估导致的异常（`module.status` 是 `'errored'`）。

当模块正在评估时（`module.status` 是 `'evaluating'`）不能调用此方法。

对应于 ECMAScript 规范中 [循环模块记录][] 的 [Evaluate() 具体方法][] 字段。

### `module.identifier`

* 类型: {string}

当前模块的标识符，在构造函数中设置。

### `module.link(linker)`

<!-- YAML
changes:
  - version:
    - v21.1.0
    - v20.10.0
    - v18.19.0
    pr-url: https://github.com/nodejs/node/pull/50141
    description: The option `extra.assert` is renamed to `extra.attributes`. The
                 former name is still provided for backward compatibility.
-->

* `linker` {Function}
  * `specifier` {string} 请求模块的说明符：
    ```mjs
    import foo from 'foo';
    //              ^^^^^ 模块说明符
    ```

  * `referencingModule` {vm.Module} 调用 `link()` 的 `Module` 对象。

  * `extra` {Object}
    * `attributes` {Object} 来自属性的数据：
      ```mjs
      import foo from 'foo' with { name: 'value' };
      //                         ^^^^^^^^^^^^^^^^^ 属性
      ```
      根据 ECMA-262，如果存在不受支持的属性，宿主应触发错误。
    * `assert` {Object} `extra.attributes` 的别名。

  * 返回: {vm.Module|Promise}
* 返回: {Promise}

链接模块依赖项。此方法必须在评估之前调用，并且每个模块只能调用一次。

使用 [`sourceTextModule.linkRequests(modules)`][] 和 [`sourceTextModule.instantiate()`][] 来同步或异步链接模块。

该函数预期返回一个 `Module` 对象或最终解析为 `Module` 对象的 `Promise`。返回的 `Module` 必须满足以下两个不变条件：

* 它必须与父 `Module` 属于同一上下文。
* 它的 `status` 不能是 `'errored'`。

如果返回的 `Module` 的 `status` 是 `'unlinked'`，则将在返回的 `Module` 上递归调用此方法，并使用相同的提供的 `linker` 函数。

`link()` 返回一个 `Promise`，该 Promise 将在所有链接实例解析为有效的 `Module` 时解析，或者在链接器函数抛出异常或返回无效的 `Module` 时拒绝。

链接器函数大致对应于 ECMAScript 规范中实现定义的 [HostResolveImportedModule][] 抽象操作，但有一些关键区别：

* 链接器函数允许是异步的，而 [HostResolveImportedModule][] 是同步的。

模块链接期间使用的实际 [HostResolveImportedModule][] 实现是返回在链接期间链接的模块的实现。因为在那个时候所有模块都已经完全链接，所以 [HostResolveImportedModule][] 实现根据规范是完全同步的。

对应于 ECMAScript 规范中 [循环模块记录][] 的 [Link() 具体方法][] 字段。

### `module.namespace`

* 类型: {Object}

模块的命名空间对象。这仅在链接（`module.link()`）完成后可用。

对应于 ECMAScript 规范中的 [GetModuleNamespace][] 抽象操作。

### `module.status`

* 类型: {string}

模块的当前状态。将是以下之一：

* `'unlinked'`：尚未调用 `module.link()`。

* `'linking'`：已调用 `module.link()`，但链接器函数返回的 Promise 尚未全部解析。

* `'linked'`：模块已成功链接，其所有依赖项都已链接，但尚未调用 `module.evaluate()`。

* `'evaluating'`：模块正在通过自身或父模块的 `module.evaluate()` 进行评估。

* `'evaluated'`：模块已成功评估。

* `'errored'`：模块已评估，但抛出了异常。

除了 `'errored'`，此状态字符串对应于规范中的 [循环模块记录][] 的 `[[Status]]` 字段。`'errored'` 对应于规范中的 `'evaluated'`，但 `[[EvaluationError]]` 设置为不是 `undefined` 的值。

## 类：`vm.SourceTextModule`

<!-- YAML
added: v9.6.0
-->

> Stability: 1 - Experimental

此功能仅在启用 `--experimental-vm-modules` 命令标志时可用。

* 扩展: {vm.Module}

`vm.SourceTextModule` 类提供了 ECMAScript 规范中定义的 [源文本模块记录][]。

### `new vm.SourceTextModule(code[, options])`

<!-- YAML
changes:
  - version:
    - v17.0.0
    - v16.12.0
    pr-url: https://github.com/nodejs/node/pull/40249
    description: Added support for import attributes to the
                 `importModuleDynamically` parameter.
-->

* `code` {string} 要解析的 JavaScript 模块代码
* `options`
  * `identifier` {string} 在堆栈跟踪中使用的字符串。**默认值:** `'vm:module(i)'`，其中 `i` 是特定于上下文的递增索引。
  * `cachedData` {Buffer|TypedArray|DataView} 提供可选的 `Buffer`、`TypedArray` 或 `DataView`，其中包含 V8 为所提供的源代码生成的代码缓存数据。`code` 必须与生成此 `cachedData` 的模块相同。
  * `context` {Object} 由 `vm.createContext()` 方法返回的 [contextified][] 对象，用于编译和评估此 `Module`。如果未指定上下文，模块将在当前执行上下文中进行评估。
  * `lineOffset` {integer} 指定此 `Module` 产生的堆栈跟踪中显示的行号偏移量。**默认值:** `0`。
  * `columnOffset` {integer} 指定此 `Module` 产生的堆栈跟踪中显示的首行列号偏移量。**默认值:** `0`。
  * `initializeImportMeta` {Function} 在此 `Module` 评估期间调用以初始化 `import.meta`。
    * `meta` {import.meta}
    * `module` {vm.SourceTextModule}
  * `importModuleDynamically` {Function} 用于指定在此模块评估期间调用 `import()` 时应如何加载模块。此选项是实验性模块 API 的一部分。我们不建议在生产环境中使用它。详细信息请参阅 [编译 API 中对动态 `import()` 的支持][]。

创建一个新的 `SourceTextModule` 实例。

分配给 `import.meta` 对象的属性如果是对象，可能允许模块访问指定 `context` 之外的信息。使用 `vm.runInContext()` 在特定上下文中创建对象。

```mjs
import vm from 'node:vm';

const contextifiedObject = vm.createContext({ secret: 42 });

const module = new vm.SourceTextModule(
  'Object.getPrototypeOf(import.meta.prop).secret = secret;',
  {
    initializeImportMeta(meta) {
      // 注意：此对象是在顶级上下文中创建的。因此，
      // Object.getPrototypeOf(import.meta.prop) 指向的是
      // 顶级上下文中的 Object.prototype，而不是
      // contextifiedObject 中的那个。
      meta.prop = {};
    },
  });
// 该模块有一个空的 `moduleRequests` 数组。
module.linkRequests([]);
module.instantiate();
await module.evaluate();

// 现在，Object.prototype.secret 将等于 42。
//
// 要解决此问题，请将上面的
//     meta.prop = {};
// 替换为
//     meta.prop = vm.runInContext('{}', contextifiedObject);
```

```cjs
const vm = require('node:vm');
const contextifiedObject = vm.createContext({ secret: 42 });
(async () => {
  const module = new vm.SourceTextModule(
    'Object.getPrototypeOf(import.meta.prop).secret = secret;',
    {
      initializeImportMeta(meta) {
        // 注意：此对象是在顶级上下文中创建的。因此，
        // Object.getPrototypeOf(import.meta.prop) 指向的是
        // 顶级上下文中的 Object.prototype，而不是
        // contextifiedObject 中的那个。
        meta.prop = {};
      },
    });
  // 该模块有一个空的 `moduleRequests` 数组。
  module.linkRequests([]);
  module.instantiate();
  await module.evaluate();
  // 现在，Object.prototype.secret 将等于 42。
  //
  // 要解决此问题，请将上面的
  //     meta.prop = {};
  // 替换为
  //     meta.prop = vm.runInContext('{}', contextifiedObject);
})();
```

### `sourceTextModule.createCachedData()`

<!-- YAML
added:
 - v13.7.0
 - v12.17.0
-->

* 返回: {Buffer}

创建一个代码缓存，可用于 `SourceTextModule` 构造函数的 `cachedData` 选项。返回一个 `Buffer`。此方法可以在模块评估之前调用任意次数。

`SourceTextModule` 的代码缓存不包含任何 JavaScript 可观察状态。代码缓存可以安全地与脚本源一起保存，并多次用于构建新的 `SourceTextModule` 实例。

`SourceTextModule` 源中的函数可以被标记为延迟编译，它们在 `SourceTextModule` 构造时不会被编译。这些函数将在首次调用时被编译。代码缓存序列化了 V8 当前已知的关于 `SourceTextModule` 的元数据，它可以用来加速未来的编译。

```js
// 创建一个初始模块
const module = new vm.SourceTextModule('const a = 1;');

// 从此模块创建缓存数据
const cachedData = module.createCachedData();

// 使用缓存数据创建一个新模块。代码必须相同。
const module2 = new vm.SourceTextModule('const a = 1;', { cachedData });
```

### `sourceTextModule.dependencySpecifiers`

<!-- YAML
changes:
  - version: v24.4.0
    pr-url: https://github.com/nodejs/node/pull/20300
    description: This is deprecated in favour of `sourceTextModule.moduleRequests`.
-->

> Stability: 0 - Deprecated: 使用 [`sourceTextModule.moduleRequests`][] 代替。

* 类型: {string\[]}

此模块所有依赖项的说明符。返回的数组被冻结以防止任何更改。

对应于 ECMAScript 规范中 [循环模块记录][] 的 `[[RequestedModules]]` 字段。

### `sourceTextModule.hasAsyncGraph()`

<!-- YAML
added: v24.9.0
-->

* 返回: {boolean}

遍历依赖图，如果其依赖项中的任何模块或此模块本身包含顶级 `await` 表达式，则返回 `true`，否则返回 `false`。

如果图足够大，搜索可能会很慢。

这要求模块首先被实例化。如果模块尚未实例化，将抛出错误。

### `sourceTextModule.hasTopLevelAwait()`

<!-- YAML
added: v24.9.0
-->

* 返回: {boolean}

返回模块本身是否包含任何顶级 `await` 表达式。

这对应于 ECMAScript 规范中 [循环模块记录][] 的 `[[HasTLA]]` 字段。

### `sourceTextModule.instantiate()`

<!-- YAML
added: v24.8.0
-->

* 返回: {undefined}

使用链接的请求模块实例化模块。

这解析了模块的导入绑定，包括重新导出的绑定名称。如果有任何无法解析的绑定，将同步抛出错误。

如果请求的模块包括循环依赖，则必须在调用此方法之前对循环中的所有模块调用 [`sourceTextModule.linkRequests(modules)`][] 方法。

### `sourceTextModule.linkRequests(modules)`

<!-- YAML
added: v24.8.0
-->

* `modules` {vm.Module\[]} 此模块依赖的 `vm.Module` 对象数组。数组中模块的顺序是 [`sourceTextModule.moduleRequests`][] 的顺序。
* 返回: {undefined}

链接模块依赖项。此方法必须在评估之前调用，并且每个模块只能调用一次。

`modules` 数组中模块实例的顺序应对应于解析的 [`sourceTextModule.moduleRequests`][] 的顺序。如果两个模块请求具有相同的说明符和导入属性，它们必须使用相同的模块实例解析，否则将抛出 `ERR_MODULE_LINK_MISMATCH`。例如，当链接此模块的请求时：

<!-- eslint-disable no-duplicate-imports -->

```mjs
import foo from 'foo';
import source Foo from 'foo';
```

<!-- eslint-enable no-duplicate-imports -->

`modules` 数组必须包含对同一实例的两个引用，因为两个模块请求是相同的，但处于两个阶段。

如果模块没有依赖项，`modules` 数组可以为空。

用户可以使用 `sourceTextModule.moduleRequests` 来实现 ECMAScript 规范中宿主定义的 [HostLoadImportedModule][] 抽象操作，并使用 `sourceTextModule.linkRequests()` 在具有所有依赖项的模块上调用规范定义的 [FinishLoadingImportedModule][]。

由 `SourceTextModule` 的创建者决定依赖项的解析是同步还是异步。

在 `modules` 数组中的每个模块链接后，调用 [`sourceTextModule.instantiate()`][]。

### `sourceTextModule.moduleRequests`

<!-- YAML
added: v24.4.0
-->

* 类型: {ModuleRequest\[]} 此模块的依赖项。

此模块请求的导入依赖项。返回的数组被冻结以防止任何更改。

例如，给定一个源文本：

<!-- eslint-disable no-duplicate-imports -->

```mjs
import foo from 'foo';
import fooAlias from 'foo';
import bar from './bar.js';
import withAttrs from '../with-attrs.ts' with { arbitraryAttr: 'attr-val' };
import source Module from 'wasm-mod.wasm';
```

<!-- eslint-enable no-duplicate-imports -->

`sourceTextModule.moduleRequests` 的值将是：

```js
[
  {
    specifier: 'foo',
    attributes: {},
    phase: 'evaluation',
  },
  {
    specifier: 'foo',
    attributes: {},
    phase: 'evaluation',
  },
  {
    specifier: './bar.js',
    attributes: {},
    phase: 'evaluation',
  },
  {
    specifier: '../with-attrs.ts',
    attributes: { arbitraryAttr: 'attr-val' },
    phase: 'evaluation',
  },
  {
    specifier: 'wasm-mod.wasm',
    attributes: {},
    phase: 'source',
  },
];
```

## 类：`vm.SyntheticModule`

<!-- YAML
added:
 - v13.0.0
 - v12.16.0
-->

> Stability: 1 - Experimental

此功能仅在启用 `--experimental-vm-modules` 命令标志时可用。

* 扩展: {vm.Module}

`vm.SyntheticModule` 类提供了 WebIDL 规范中定义的 [合成模块记录][]。合成模块的目的是为向 ECMAScript 模块图暴露非 JavaScript 源提供通用接口。

```js
const vm = require('node:vm');

const source = '{ "a": 1 }';
const module = new vm.SyntheticModule(['default'], function() {
  const obj = JSON.parse(source);
  this.setExport('default', obj);
});

// 在链接中使用 `module`...
```

### `new vm.SyntheticModule(exportNames, evaluateCallback[, options])`

<!-- YAML
added:
 - v13.0.0
 - v12.16.0
-->

* `exportNames` {string\[]} 将从模块导出的名称数组。
* `evaluateCallback` {Function} 在模块评估时调用。
* `options`
  * `identifier` {string} 在堆栈跟踪中使用的字符串。**默认值:** `'vm:module(i)'`，其中 `i` 是特定于上下文的递增索引。
  * `context` {Object} 由 `vm.createContext()` 方法返回的 [contextified][] 对象，用于编译和评估此 `Module`。

创建一个新的 `SyntheticModule` 实例。

分配给此实例导出的对象可能允许模块的导入者访问指定 `context` 之外的信息。使用 `vm.runInContext()` 在特定上下文中创建对象。

### `syntheticModule.setExport(name, value)`

<!-- YAML
added:
 - v13.0.0
 - v12.16.0
changes:
  - version: v24.8.0
    pr-url: https://github.com/nodejs/node/pull/59000
    description: No longer need to call `syntheticModule.link()` before
                 calling this method.
-->

* `name` {string} 要设置的导出名称。
* `value` {any} 设置导出的值。

此方法使用给定值设置模块导出绑定槽。

```mjs
import vm from 'node:vm';

const m = new vm.SyntheticModule(['x'], () => {
  m.setExport('x', 1);
});

await m.evaluate();

assert.strictEqual(m.namespace.x, 1);
```

```cjs
const vm = require('node:vm');
(async () => {
  const m = new vm.SyntheticModule(['x'], () => {
    m.setExport('x', 1);
  });
  await m.evaluate();
  assert.strictEqual(m.namespace.x, 1);
})();
```

## 类型：`ModuleRequest`

<!-- YAML
added: v24.4.0
-->

* 类型: {Object}
  * `specifier` {string} 请求模块的说明符。
  * `attributes` {Object} 传递给 [ImportDeclaration][] 中 [WithClause][] 的 `"with"` 值，如果未提供值则为空对象。
  * `phase` {string} 请求模块的阶段（`"source"` 或 `"evaluation"`）。

`ModuleRequest` 表示请求导入具有给定导入属性和阶段的模块。

## `vm.compileFunction(code[, params[, options]])`

<!-- YAML
added: v10.10.0
changes:
  - version:
    - v21.7.0
    - v20.12.0
    pr-url: https://github.com/nodejs/node/pull/51244
    description: Added support for
                `vm.constants.USE_MAIN_CONTEXT_DEFAULT_LOADER`.
  - version:
    - v19.6.0
    - v18.15.0
    pr-url: https://github.com/nodejs/node/pull/46320
    description: The return value now includes `cachedDataRejected`
                 with the same semantics as the `vm.Script` version
                 if the `cachedData` option was passed.
  - version:
    - v17.0.0
    - v16.12.0
    pr-url: https://github.com/nodejs/node/pull/40249
    description: Added support for import attributes to the
                 `importModuleDynamically` parameter.
  - version: v15.9.0
    pr-url: https://github.com/nodejs/node/pull/35431
    description: Added `importModuleDynamically` option again.
  - version: v14.3.0
    pr-url: https://github.com/nodejs/node/pull/33364
    description: Removal of `importModuleDynamically` due to compatibility
                 issues.
  - version:
    - v14.1.0
    - v13.14.0
    pr-url: https://github.com/nodejs/node/pull/32985
    description: The `importModuleDynamically` option is now supported.
-->

* `code` {string} 要编译的函数体。
* `params` {string\[]} 包含函数所有参数的字符串数组。
* `options` {Object}
  * `filename` {string} 指定此脚本产生的堆栈跟踪中使用的文件名。**默认值:** `''`。
  * `lineOffset` {number} 指定此脚本产生的堆栈跟踪中显示的行号偏移量。**默认值:** `0`。
  * `columnOffset` {number} 指定此脚本产生的堆栈跟踪中显示的首行列号偏移量。**默认值:** `0`。
  * `cachedData` {Buffer|TypedArray|DataView} 提供可选的 `Buffer`、`TypedArray` 或 `DataView`，其中包含 V8 为所提供的源代码生成的代码缓存数据。这必须由之前对 [`vm.compileFunction()`][] 的调用产生，使用相同的 `code` 和 `params`。
  * `produceCachedData` {boolean} 指定是否生成新的缓存数据。**默认值:** `false`。
  * `parsingContext` {Object} 应在其中编译所述函数的 [contextified][] 对象。
  * `contextExtensions` {Object\[]} 包含一组上下文扩展（包装当前作用域的对象）的数组，在编译时应用。**默认值:** `[]`。
  * `importModuleDynamically`
    {Function|vm.constants.USE\_MAIN\_CONTEXT\_DEFAULT\_LOADER}
    用于指定在此函数评估期间调用 `import()` 时应如何加载模块。此选项是实验性模块 API 的一部分。我们不建议在生产环境中使用它。详细信息请参阅 [编译 API 中对动态 `import()` 的支持][]。
* 返回: {Function}

将给定的代码编译到提供的上下文中（如果未提供上下文，则使用当前上下文），并将其包装在具有给定 `params` 的函数中返回。

## `vm.constants`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

* 类型: {Object}

返回一个包含 VM 操作常用常量的对象。

### `vm.constants.USE_MAIN_CONTEXT_DEFAULT_LOADER`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
-->

> Stability: 1.1 - Active development

一个常量，可用作 `vm.Script` 和 `vm.compileFunction()` 的 `importModuleDynamically` 选项，以便 Node.js 使用主上下文的默认 ESM 加载器来加载请求的模块。

详细信息请参阅 [编译 API 中对动态 `import()` 的支持][]。

## `vm.createContext([contextObject[, options]])`

<!-- YAML
added: v0.3.1
changes:
  - version:
    - v22.8.0
    - v20.18.0
    pr-url: https://github.com/nodejs/node/pull/54394
    description: The `contextObject` argument now accepts `vm.constants.DONT_CONTEXTIFY`.
  - version:
    - v21.7.0
    - v20.12.0
    pr-url: https://github.com/nodejs/node/pull/51244
    description: Added support for
                 `vm.constants.USE_MAIN_CONTEXT_DEFAULT_LOADER`.
  - version:
    - v21.2.0
    - v20.11.0
    pr-url: https://github.com/nodejs/node/pull/50360
    description: The `importModuleDynamically` option is supported now.
  - version: v14.6.0
    pr-url: https://github.com/nodejs/node/pull/34023
    description: The `microtaskMode` option is supported now.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19398
    description: The first argument can no longer be a function.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19016
    description: The `codeGeneration` option is supported now.
-->

* `contextObject` {Object|vm.constants.DONT\_CONTEXTIFY|undefined}
  可以是 [`vm.constants.DONT_CONTEXTIFY`][] 或将被 [contextified][] 的对象。如果为 `undefined`，为了向后兼容性，将创建一个空的上下文化对象。
* `options` {Object}
  * `name` {string} 新创建上下文的人类可读名称。**默认值:** `'VM Context i'`，其中 `i` 是所创建上下文的递增数字索引。
  * `origin` {string} 对应于新创建上下文的[来源][origin]，用于显示目的。来源应格式化为 URL，但仅包含方案、主机和端口（如果需要），类似于 [`URL`][] 对象的 [`url.origin`][] 属性的值。最值得注意的是，此字符串应省略尾部斜杠，因为那表示路径。**默认值:** `''`。
  * `codeGeneration` {Object}
    * `strings` {boolean} 如果设置为 false，任何对 `eval` 或函数构造函数（`Function`、`GeneratorFunction` 等）的调用都将抛出 `EvalError`。**默认值:** `true`。
    * `wasm` {boolean} 如果设置为 false，任何编译 WebAssembly 模块的尝试都将抛出 `WebAssembly.CompileError`。**默认值:** `true`。
  * `microtaskMode` {string} 如果设置为 `afterEvaluate`，微任务（通过 `Promise` 和 `async function` 调度的任务）将在脚本通过 [`script.runInContext()`][] 运行后立即运行。在这种情况下，它们包含在 `timeout` 和 `breakOnSigint` 范围内。
  * `importModuleDynamically`
    {Function|vm.constants.USE\_MAIN\_CONTEXT\_DEFAULT\_LOADER}
    用于指定在此上下文中调用 `import()` 而没有引用脚本或模块时应如何加载模块。此选项是实验性模块 API 的一部分。我们不建议在生产环境中使用它。详细信息请参阅 [编译 API 中对动态 `import()` 的支持][]。
* 返回: {Object} 上下文化对象。

如果给定的 `contextObject` 是一个对象，`vm.createContext()` 方法将[准备该对象][contextified]并返回对其的引用，以便可以在调用 [`vm.runInContext()`][] 或 [`script.runInContext()`][] 时使用。在此类脚本内部，全局对象将被 `contextObject` 包装，保留其所有现有属性，同时也拥有任何标准[全局对象][]所具有的内置对象和函数。在 vm 模块运行的脚本之外，全局变量将保持不变。

```js
const vm = require('node:vm');

global.globalVar = 3;

const context = { globalVar: 1 };
vm.createContext(context);

vm.runInContext('globalVar *= 2;', context);

console.log(context);
// 打印: { globalVar: 2 }

console.log(global.globalVar);
// 打印: 3
```

如果省略 `contextObject`（或显式传递为 `undefined`），将返回一个新的空的 [contextified][] 对象。

当新创建上下文中的全局对象被 [contextified][] 时，与普通全局对象相比，它有一些怪癖。例如，它不能被冻结。要创建一个没有上下文化怪癖的上下文，请将 [`vm.constants.DONT_CONTEXTIFY`][] 作为 `contextObject` 参数传递。有关详细信息，请参阅 [`vm.constants.DONT_CONTEXTIFY`][] 的文档。

`vm.createContext()` 方法主要用于创建单个上下文，该上下文可用于运行多个脚本。例如，如果模拟 Web 浏览器，该方法可用于创建表示窗口全局对象的单个上下文，然后在该上下文中一起运行所有 `<script>` 标签。

提供的上下文的 `name` 和 `origin` 通过 Inspector API 可见。

## `vm.isContext(object)`

<!-- YAML
added: v0.11.7
-->

* `object` {Object}
* 返回: {boolean}

如果给定的 `object` 对象已使用 [`vm.createContext()`][] 进行 [contextified][]，或者它是使用 [`vm.constants.DONT_CONTEXTIFY`][] 创建的上下文的全局对象，则返回 `true`。

## `vm.measureMemory([options])`

<!-- YAML
added: v13.10.0
-->

> Stability: 1 - Experimental

测量 V8 已知的并由当前 V8 隔离器已知的所有上下文或主上下文使用的内存。

* `options` {Object} 可选。
  * `mode` {string} `'summary'` 或 `'detailed'`。在摘要模式下，仅返回为主上下文测量的内存。在详细模式下，将返回为当前 V8 隔离器已知的所有上下文测量的内存。**默认值:** `'summary'`
  * `execution` {string} `'default'` 或 `'eager'`。使用默认执行，promise 直到下一个计划的垃圾回收开始后才会解析，这可能需要一段时间（或者如果程序在下一个 GC 之前退出，则永远不会解析）。使用急切执行，将立即启动 GC 来测量内存。**默认值:** `'default'`
* 返回: {Promise} 如果内存测量成功，promise 将使用包含内存使用信息的对象解析。否则它将因 `ERR_CONTEXT_NOT_INITIALIZED` 错误而被拒绝。

Promise 可能解析的对象的格式特定于 V8 引擎，并且可能从一个版本的 V8 更改为下一个版本。

返回的结果与 `v8.getHeapSpaceStatistics()` 返回的统计信息不同，因为 `vm.measureMemory()` 测量当前 V8 实例中每个 V8 特定上下文可访问的内存，而 `v8.getHeapSpaceStatistics()` 的结果测量当前 V8 实例中每个堆空间占用的内存。

```js
const vm = require('node:vm');
// 测量主上下文使用的内存。
vm.measureMemory({ mode: 'summary' })
  // 这与 vm.measureMemory() 相同
  .then((result) => {
    // 当前格式是：
    // {
    //   total: {
    //      jsMemoryEstimate: 2418479, jsMemoryRange: [ 2418479, 2745799 ]
    //    }
    // }
    console.log(result);
  });

const context = vm.createContext({ a: 1 });
vm.measureMemory({ mode: 'detailed', execution: 'eager' })
  .then((result) => {
    // 在此处引用上下文，以便在测量完成之前不会被 GC 回收。
    console.log(context.a);
    // {
    //   total: {
    //     jsMemoryEstimate: 2574732,
    //     jsMemoryRange: [ 2574732, 2904372 ]
    //   },
    //   current: {
    //     jsMemoryEstimate: 2438996,
    //     jsMemoryRange: [ 2438996, 2768636 ]
    //   },
    //   other: [
    //     {
    //       jsMemoryEstimate: 135736,
    //       jsMemoryRange: [ 135736, 465376 ]
    //     }
    //   ]
    // }
    console.log(result);
  });
```

## `vm.runInContext(code, contextifiedObject[, options])`

<!-- YAML
added: v0.3.1
changes:
  - version:
    - v21.7.0
    - v20.12.0
    pr-url: https://github.com/nodejs/node/pull/51244
    description: Added support for
                `vm.constants.USE_MAIN_CONTEXT_DEFAULT_LOADER`.
  - version:
    - v17.0.0
    - v16.12.0
    pr-url: https://github.com/nodejs/node/pull/40249
    description: Added support for import attributes to the
                 `importModuleDynamically` parameter.
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/6635
    description: The `breakOnSigint` option is supported now.
-->

* `code` {string} 要编译和运行的 JavaScript 代码。
* `contextifiedObject` {Object} 在编译和运行 `code` 时将用作 `global` 的 [contextified][] 对象。
* `options` {Object|string}
  * `filename` {string} 指定此脚本产生的堆栈跟踪中使用的文件名。**默认值:** `'evalmachine.<anonymous>'`。
  * `lineOffset` {number} 指定此脚本产生的堆栈跟踪中显示的行号偏移量。**默认值:** `0`。
  * `columnOffset` {number} 指定此脚本产生的堆栈跟踪中显示的首行列号偏移量。**默认值:** `0`。
  * `displayErrors` {boolean} 当为 `true` 时，如果在编译 `code` 时发生 [`Error`][]，导致错误的代码行会附加到堆栈跟踪中。**默认值:** `true`。
  * `timeout` {integer} 指定在执行终止前执行 `code` 的毫秒数。如果执行被终止，将会抛出 [`Error`][]。此值必须是一个严格的正整数。
  * `breakOnSigint` {boolean} 如果为 `true`，接收 `SIGINT` (<kbd>Ctrl</kbd>+<kbd>C</kbd>) 将终止执行并抛出 [`Error`][]。在脚本执行期间，通过 `process.on('SIGINT')` 附加的事件现有处理程序将被禁用，但在之后会继续工作。**默认值:** `false`。
  * `cachedData` {Buffer|TypedArray|DataView} 提供可选的 `Buffer`、`TypedArray` 或 `DataView`，其中包含 V8 为所提供的源代码生成的代码缓存数据。
  * `importModuleDynamically`
    {Function|vm.constants.USE\_MAIN\_CONTEXT\_DEFAULT\_LOADER}
    用于指定在此脚本评估期间调用 `import()` 时应如何加载模块。此选项是实验性模块 API 的一部分。我们不建议在生产环境中使用它。详细信息请参阅 [编译 API 中对动态 `import()` 的支持][]。

`vm.runInContext()` 方法编译 `code`，在 `contextifiedObject` 的上下文中运行它，然后返回结果。运行的代码无法访问局部作用域。`contextifiedObject` 对象*必须*先前已使用 [`vm.createContext()`][] 方法进行 [contextified][]。

如果 `options` 是字符串，则它指定文件名。

以下示例使用单个 [contextified][] 对象编译和执行不同的脚本：

```js
const vm = require('node:vm');

const contextObject = { globalVar: 1 };
vm.createContext(contextObject);

for (let i = 0; i < 10; ++i) {
  vm.runInContext('globalVar *= 2;', contextObject);
}
console.log(contextObject);
// 打印: { globalVar: 1024 }
```

## `vm.runInNewContext(code[, contextObject[, options]])`

<!-- YAML
added: v0.3.1
changes:
  - version:
    - v22.8.0
    - v20.18.0
    pr-url: https://github.com/nodejs/node/pull/54394
    description: The `contextObject` argument now accepts `vm.constants.DONT_CONTEXTIFY`.
  - version:
    - v21.7.0
    - v20.12.0
    pr-url: https://github.com/nodejs/node/pull/51244
    description: Added support for
                `vm.constants.USE_MAIN_CONTEXT_DEFAULT_LOADER`.
  - version:
    - v17.0.0
    - v16.12.0
    pr-url: https://github.com/nodejs/node/pull/40249
    description: Added support for import attributes to the
                 `importModuleDynamically` parameter.
  - version: v14.6.0
    pr-url: https://github.com/nodejs/node/pull/34023
    description: The `microtaskMode` option is supported now.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/19016
    description: The `contextCodeGeneration` option is supported now.
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/6635
    description: The `breakOnSigint` option is supported now.
-->

* `code` {string} 要编译和运行的 JavaScript 代码。
* `contextObject` {Object|vm.constants.DONT\_CONTEXTIFY|undefined}
  可以是 [`vm.constants.DONT_CONTEXTIFY`][] 或将被 [contextified][] 的对象。如果为 `undefined`，为了向后兼容性，将创建一个空的上下文化对象。
* `options` {Object|string}
  * `filename` {string} 指定此脚本产生的堆栈跟踪中使用的文件名。**默认值:** `'evalmachine.<anonymous>'`。
  * `lineOffset` {number} 指定此脚本产生的堆栈跟踪中显示的行号偏移量。**默认值:** `0`。
  * `columnOffset` {number} 指定此脚本产生的堆栈跟踪中显示的首行列号偏移量。**默认值:** `0`。
  * `displayErrors` {boolean} 当为 `true` 时，如果在编译 `code` 时发生 [`Error`][]，导致错误的代码行会附加到堆栈跟踪中。**默认值:** `true`。
  * `timeout` {integer} 指定在执行终止前执行 `code` 的毫秒数。如果执行被终止，将会抛出 [`Error`][]。此值必须是一个严格的正整数。
  * `breakOnSigint` {boolean} 如果为 `true`，接收 `SIGINT` (<kbd>Ctrl</kbd>+<kbd>C</kbd>) 将终止执行并抛出 [`Error`][]。在脚本执行期间，通过 `process.on('SIGINT')` 附加的事件现有处理程序将被禁用，但在之后会继续工作。**默认值:** `false`。
  * `contextName` {string} 新创建上下文的人类可读名称。**默认值:** `'VM Context i'`，其中 `i` 是所创建上下文的递增数字索引。
  * `contextOrigin` {string} 对应于新创建上下文的[来源][origin]，用于显示目的。来源应格式化为 URL，但仅包含方案、主机和端口（如果需要），类似于 [`URL`][] 对象的 [`url.origin`][] 属性的值。最值得注意的是，此字符串应省略尾部斜杠，因为那表示路径。**默认值:** `''`。
  * `contextCodeGeneration` {Object}
    * `strings` {boolean} 如果设置为 false，任何对 `eval` 或函数构造函数（`Function`、`GeneratorFunction` 等）的调用都将抛出 `EvalError`。**默认值:** `true`。
    * `wasm` {boolean} 如果设置为 false，任何编译 WebAssembly 模块的尝试都将抛出 `WebAssembly.CompileError`。**默认值:** `true`。
  * `cachedData` {Buffer|TypedArray|DataView} 提供可选的 `Buffer`、`TypedArray` 或 `DataView`，其中包含 V8 为所提供的源代码生成的代码缓存数据。
  * `importModuleDynamically`
    {Function|vm.constants.USE\_MAIN\_CONTEXT\_DEFAULT\_LOADER}
    用于指定在此脚本评估期间调用 `import()` 时应如何加载模块。此选项是实验性模块 API 的一部分。我们不建议在生产环境中使用它。详细信息请参阅 [编译 API 中对动态 `import()` 的支持][]。
  * `microtaskMode` {string} 如果设置为 `afterEvaluate`，微任务（通过 `Promise` 和 `async function` 调度的任务）将在脚本运行后立即运行。在这种情况下，它们包含在 `timeout` 和 `breakOnSigint` 范围内。
* 返回: {any} 脚本中执行的最后一个语句的结果。

此方法是 `(new vm.Script(code, options)).runInContext(vm.createContext(options), options)` 的快捷方式。如果 `options` 是字符串，则它指定文件名。

它同时做了几件事：

1. 创建一个新的上下文。
2. 如果 `contextObject` 是一个对象，则使用新上下文对其进行 [contextified][]。如果 `contextObject` 是 undefined，则创建一个新对象并对其进行 [contextified][]。如果 `contextObject` 是 [`vm.constants.DONT_CONTEXTIFY`][]，则不 [contextify][] 任何东西。
3. 将代码编译为 `vm.Script`
4. 在创建的上下文中运行已编译的代码。代码无法访问此方法被调用的作用域。
5. 返回结果。

以下示例编译并执行增加全局变量并设置新变量的代码。这些全局变量包含在 `contextObject` 中。

```js
const vm = require('node:vm');

const contextObject = {
  animal: 'cat',
  count: 2,
};

vm.runInNewContext('count += 1; name = "kitty"', contextObject);
console.log(contextObject);
// 打印: { animal: 'cat', count: 3, name: 'kitty' }

// 如果上下文是从上下文化对象创建的，这将抛出。
// vm.constants.DONT_CONTEXTIFY 允许创建具有普通全局对象的上下文，这些对象可以被冻结。
const frozenContext = vm.runInNewContext('Object.freeze(globalThis); globalThis;', vm.constants.DONT_CONTEXTIFY);
```

## `vm.runInThisContext(code[, options])`

<!-- YAML
added: v0.3.1
changes:
  - version:
    - v21.7.0
    - v20.12.0
    pr-url: https://github.com/nodejs/node/pull/51244
    description: Added support for
                `vm.constants.USE_MAIN_CONTEXT_DEFAULT_LOADER`.
  - version:
    - v17.0.0
    - v16.12.0
    pr-url: https://github.com/nodejs/node/pull/40249
    description: Added support for import attributes to the
                 `importModuleDynamically` parameter.
  - version: v6.3.0
    pr-url: https://github.com/nodejs/node/pull/6635
    description: The `breakOnSigint` option is supported now.
-->

* `code` {string} 要编译和运行的 JavaScript 代码。
* `options` {Object|string}
  * `filename` {string} 指定此脚本产生的堆栈跟踪中使用的文件名。**默认值:** `'evalmachine.<anonymous>'`。
  * `lineOffset` {number} 指定此脚本产生的堆栈跟踪中显示的行号偏移量。**默认值:** `0`。
  * `columnOffset` {number} 指定此脚本产生的堆栈跟踪中显示的首行列号偏移量。**默认值:** `0`。
  * `displayErrors` {boolean} 当为 `true` 时，如果在编译 `code` 时发生 [`Error`][]，导致错误的代码行会附加到堆栈跟踪中。**默认值:** `true`。
  * `timeout` {integer} 指定在执行终止前执行 `code` 的毫秒数。如果执行被终止，将会抛出 [`Error`][]。此值必须是一个严格的正整数。
  * `breakOnSigint` {boolean} 如果为 `true`，接收 `SIGINT` (<kbd>Ctrl</kbd>+<kbd>C</kbd>) 将终止执行并抛出 [`Error`][]。在脚本执行期间，通过 `process.on('SIGINT')` 附加的事件现有处理程序将被禁用，但在之后会继续工作。**默认值:** `false`。
  * `cachedData` {Buffer|TypedArray|DataView} 提供可选的 `Buffer`、`TypedArray` 或 `DataView`，其中包含 V8 为所提供的源代码生成的代码缓存数据。
  * `importModuleDynamically`
    {Function|vm.constants.USE\_MAIN\_CONTEXT\_DEFAULT\_LOADER}
    用于指定在此脚本评估期间调用 `import()` 时应如何加载模块。此选项是实验性模块 API 的一部分。我们不建议在生产环境中使用它。详细信息请参阅 [编译 API 中对动态 `import()` 的支持][]。
* 返回: {any} 脚本中执行的最后一个语句的结果。

`vm.runInThisContext()` 编译 `code`，在当前 `global` 的上下文中运行它，并返回结果。运行的代码无法访问局部作用域，但可以访问当前的 `global` 对象。

如果 `options` 是字符串，则它指定文件名。

以下示例说明了使用 `vm.runInThisContext()` 和 JavaScript [`eval()`][] 函数运行相同的代码：

<!-- eslint-disable prefer-const -->

```js
const vm = require('node:vm');
let localVar = 'initial value';

const vmResult = vm.runInThisContext('localVar = "vm";');
console.log(`vmResult: '${vmResult}', localVar: '${localVar}'`);
// 打印: vmResult: 'vm', localVar: 'initial value'

const evalResult = eval('localVar = "eval";');
console.log(`evalResult: '${evalResult}', localVar: '${localVar}'`);
// 打印: evalResult: 'eval', localVar: 'eval'
```

因为 `vm.runInThisContext()` 无法访问局部作用域，`localVar`  unchanged。相反，直接的 `eval()` 调用*可以*访问局部作用域，因此值 `localVar` 被更改。这样，`vm.runInThisContext()` 非常像[间接的 `eval()` 调用][]，例如 `(0,eval)('code')`。

## 示例：在 VM 中运行 HTTP 服务器

当使用 [`script.runInThisContext()`][] 或 [`vm.runInThisContext()`][] 时，代码在当前 V8 全局上下文中执行。传递给此 VM 上下文的代码将具有其自己独立的作用域。

为了使用 `node:http` 模块运行一个简单的 Web 服务器，传递给上下文的代码必须自己调用 `require('node:http')`，或者具有传递给它的 `node:http` 模块的引用。例如：

```js
'use strict';
const vm = require('node:vm');

const code = `
((require) => {
  const http = require('node:http');

  http.createServer((request, response) => {
    response.writeHead(200, { 'Content-Type': 'text/plain' });
    response.end('Hello World\\n');
  }).listen(8124);

  console.log('Server running at http://127.0.0.1:8124/');
})`;

vm.runInThisContext(code)(require);
```

上述情况中的 `require()` 与它传递自的上下文共享状态。当执行不受信任的代码时，这可能会引入风险，例如以不希望的方式更改上下文中的对象。

## "上下文化"一个对象是什么意思？

在 Node.js 中执行的所有 JavaScript 都在一个"上下文"的作用域内运行。根据 [V8 嵌入者指南][]：

> 在 V8 中，上下文是一个执行环境，允许独立的、不相关的 JavaScript 应用程序在 V8 的单个实例中运行。您必须明确指定希望任何 JavaScript 代码运行的上下文。

当使用对象调用 `vm.createContext()` 方法时，`contextObject` 参数将用于包装 V8 上下文新实例的全局对象（如果 `contextObject` 是 `undefined`，将在上下文化之前从当前上下文创建一个新对象）。此 V8 上下文为使用 `node:vm` 模块方法运行的 `code` 提供了一个隔离的全局环境，使其可以在其中操作。创建 V8 上下文并将其与外部上下文中的 `contextObject` 关联的过程就是本文档所指的"上下文化"对象。

上下文化会给上下文中的 `globalThis` 值带来一些怪癖。例如，它不能被冻结，并且它与外部上下文中的 `contextObject` 不是引用相等的。

```js
const vm = require('node:vm');

// 未定义的 `contextObject` 选项使全局对象被上下文化。
const context = vm.createContext();
console.log(vm.runInContext('globalThis', context) === context);  // false
// 被上下文化的全局对象不能被冻结。
try {
  vm.runInContext('Object.freeze(globalThis);', context);
} catch (e) {
  console.log(e); // TypeError: Cannot freeze
}
console.log(vm.runInContext('globalThis.foo = 1; foo;', context));  // 1
```

要创建一个具有普通全局对象并在外部上下文中访问全局代理且怪癖较少的上下文，请将 `vm.constants.DONT_CONTEXTIFY` 指定为 `contextObject` 参数。

### `vm.constants.DONT_CONTEXTIFY`

当在 vm API 中用作 `contextObject` 参数时，此常量指示 Node.js 创建一个上下文，而不以 Node.js 特定的方式用另一个对象包装其全局对象。因此，新上下文内部的 `globalThis` 值的行为更接近于普通对象。

```js
const vm = require('node:vm');

// 使用 vm.constants.DONT_CONTEXTIFY 冻结全局对象。
const context = vm.createContext(vm.constants.DONT_CONTEXTIFY);
vm.runInContext('Object.freeze(globalThis);', context);
try {
  vm.runInContext('bar = 1; bar;', context);
} catch (e) {
  console.log(e); // Uncaught ReferenceError: bar is not defined
}
```

当 `vm.constants.DONT_CONTEXTIFY` 用作 [`vm.createContext()`][] 的 `contextObject` 参数时，返回的对象是一个代理类对象，指向新创建上下文中的全局对象，具有较少的 Node.js 特定怪癖。它与新上下文中的 `globalThis` 值是引用相等的，可以从外部修改，并可用于直接访问新上下文中的内置对象。

```js
const vm = require('node:vm');

const context = vm.createContext(vm.constants.DONT_CONTEXTIFY);

// 返回的对象与新上下文中的 globalThis 引用相等。
console.log(vm.runInContext('globalThis', context) === context);  // true

// 可用于直接访问新上下文中的全局变量。
console.log(context.Array);  // [Function: Array]
vm.runInContext('foo = 1;', context);
console.log(context.foo);  // 1
context.bar = 1;
console.log(vm.runInContext('bar;', context));  // 1

// 可以被冻结，并且它会影响内部上下文。
Object.freeze(context);
try {
  vm.runInContext('baz = 1; baz;', context);
} catch (e) {
  console.log(e); // Uncaught ReferenceError: baz is not defined
}
```

## 超时与异步任务和 Promise 的交互

`Promise` 和 `async function` 可以安排由 JavaScript 引擎异步运行的任务。默认情况下，这些任务在当前堆栈上的所有 JavaScript 函数执行完毕后运行。这允许逃脱 `timeout` 和 `breakOnSigint` 选项的功能。

例如，以下由 `vm.runInNewContext()` 执行的代码，超时为 5 毫秒，在 promise 解析后安排一个无限循环运行。计划的循环永远不会被超时中断：

```js
const vm = require('node:vm');

function loop() {
  console.log('entering loop');
  while (1) console.log(Date.now());
}

vm.runInNewContext(
  'Promise.resolve().then(() => loop());',
  { loop, console },
  { timeout: 5 },
);
// 这在 'entering loop' 之前打印 (!)
console.log('done executing');
```

可以通过将 `microtaskMode: 'afterEvaluate'` 传递给创建 `Context` 的代码来解决这个问题：

```js
const vm = require('node:vm');

function loop() {
  while (1) console.log(Date.now());
}

vm.runInNewContext(
  'Promise.resolve().then(() => loop());',
  { loop, console },
  { timeout: 5, microtaskMode: 'afterEvaluate' },
);
```

在这种情况下，通过 `promise.then()` 安排的微任务将在从 `vm.runInNewContext()` 返回之前运行，并将被 `timeout` 功能中断。这仅适用于在 `vm.Context` 中运行的代码，因此例如 [`vm.runInThisContext()`][] 不接受此选项。

Promise 回调被放入它们创建时所在上下文的微任务队列中。例如，如果在上面的示例中将 `() => loop()` 替换为 `loop`，那么 `loop` 将被推入全局微任务队列，因为它是来自外部（主）上下文的函数，因此也将能够逃脱超时。

如果异步调度函数如 `process.nextTick()`、`queueMicrotask()`、`setTimeout()`、`setImmediate()` 等在 `vm.Context` 内部可用，传递给它们的函数将被添加到全局队列中，这些队列由所有上下文共享。因此，传递给这些函数的回调也不受超时控制。

### 当 `microtaskMode` 是 `'afterEvaluate'` 时，注意在上下文之间共享 Promise

在 `'afterEvaluate'` 模式下，`Context` 有自己的微任务队列，与外部（主）上下文使用的全局微任务队列分开。虽然此模式对于强制执行 `timeout` 和启用 `breakOnSigint` 与异步任务是必要的，但它也使得在上下文之间共享 Promise 具有挑战性。

在下面的示例中，一个 promise 在内部上下文中创建并与外部上下文共享。当外部上下文 `await` 该 promise 时，外部上下文的执行流程以一种令人惊讶的方式被破坏：日志语句永远不会执行。

```mjs
import * as vm from 'node:vm';

const inner_context = vm.createContext({}, { microtaskMode: 'afterEvaluate' });

// runInContext() 返回在内部上下文中创建的 Promise。
const inner_promise = vm.runInContext(
  'Promise.resolve()',
  context,
);

// 作为执行 `await` 的一部分，JavaScript 运行时必须将一个任务
// 排入 `inner_promise` 创建时所在上下文的微任务队列。
// 一个任务被添加到内部微任务队列，但**它不会自动运行**：此任务将无限期保持挂起。
//
// 由于外部微任务队列为空，外部模块中的执行
// 会落空，下面的日志语句永远不会执行。
await inner_promise;

console.log('this will NOT be printed');
```

为了在不同微任务队列的上下文之间成功共享 promise，有必要确保每当外部上下文将任务排入内部微任务队列时，内部微任务队列上的任务将被运行。

每当在此上下文上调用脚本或模块的 `runInContext()` 或 `SourceTextModule.evaluate()` 时，给定上下文的微任务队列上的任务就会运行。在我们的示例中，可以通过在 `await inner_promise` 之前安排第二次调用 `runInContext()` 来恢复正常的执行流程。

```mjs
// 安排 `runInContext()` 以手动清空内部上下文微任务队列；它将在下面的 `await` 语句之后运行。
setImmediate(() => {
  vm.runInContext('', context);
});

await inner_promise;

console.log('OK');
```

**注意：** 严格来说，在这种模式下，`node:vm` 偏离了 ECMAScript 规范中[排队作业][]的字面意义，允许来自不同上下文的异步任务以与它们排队顺序不同的顺序运行。

## 编译 API 中对动态 `import()` 的支持

以下 API 支持 `importModuleDynamically` 选项，以在 vm 模块编译的代码中启用动态 `import()`。

* `new vm.Script`
* `vm.compileFunction()`
* `new vm.SourceTextModule`
* `vm.runInThisContext()`
* `vm.runInContext()`
* `vm.runInNewContext()`
* `vm.createContext()`

此选项仍然是实验性模块 API 的一部分。我们不建议在生产环境中使用它。

### 当未指定 `importModuleDynamically` 选项或为 undefined 时

如果未指定此选项，或者它是 `undefined`，包含 `import()` 的代码仍然可以由 vm API 编译，但当编译的代码执行并实际调用 `import()` 时，结果将因 [`ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING`][] 而拒绝。

### 当 `importModuleDynamically` 是 `vm.constants.USE_MAIN_CONTEXT_DEFAULT_LOADER` 时

此选项目前不支持 `vm.SourceTextModule`。

使用此选项，当在编译的代码中发起 `import()` 时，Node.js 将使用主上下文的默认 ESM 加载器来加载请求的模块，并将其返回给正在执行的代码。

这使得正在编译的代码可以访问 Node.js 内置模块，如 `fs` 或 `http`。如果代码在不同的上下文中执行，请注意从主上下文加载的模块创建的对象仍然来自主上下文，而不是新上下文中内置类的 `instanceof`。

```cjs
const { Script, constants } = require('node:vm');
const script = new Script(
  'import("node:fs").then(({readFile}) => readFile instanceof Function)',
  { importModuleDynamically: constants.USE_MAIN_CONTEXT_DEFAULT_LOADER });

// false：从主上下文加载的 URL 不是新上下文中 Function 类的实例。
script.runInNewContext().then(console.log);
```

```mjs
import { Script, constants } from 'node:vm';

const script = new Script(
  'import("node:fs").then(({readFile}) => readFile instanceof Function)',
  { importModuleDynamically: constants.USE_MAIN_CONTEXT_DEFAULT_LOADER });

// false：从主上下文加载的 URL 不是新上下文中 Function 类的实例。
script.runInNewContext().then(console.log);
```

此选项还允许脚本或函数加载用户模块：

```mjs
import { Script, constants } from 'node:vm';
import { resolve } from 'node:path';
import { writeFileSync } from 'node:fs';

// 将 test.js 和 test.txt 写入当前正在运行的脚本所在的目录。
writeFileSync(resolve(import.meta.dirname, 'test.mjs'),
              'export const filename = "./test.json";');
writeFileSync(resolve(import.meta.dirname, 'test.json'),
              '{"hello": "world"}');

// 编译一个脚本，该脚本加载 test.mjs，然后 test.json，就像脚本放在同一目录中一样。
const script = new Script(
  `(async function() {
    const { filename } = await import('./test.mjs');
    return import(filename, { with: { type: 'json' } })
  })();`,
  {
    filename: resolve(import.meta.dirname, 'test-with-default.js'),
    importModuleDynamically: constants.USE_MAIN_CONTEXT_DEFAULT_LOADER,
  });

// { default: { hello: 'world' } }
script.runInThisContext().then(console.log);
```

```cjs
const { Script, constants } = require('node:vm');
const { resolve } = require('node:path');
const { writeFileSync } = require('node:fs');

// 将 test.js 和 test.txt 写入当前正在运行的脚本所在的目录。
writeFileSync(resolve(__dirname, 'test.mjs'),
              'export const filename = "./test.json";');
writeFileSync(resolve(__dirname, 'test.json'),
              '{"hello": "world"}');

// 编译一个脚本，该脚本加载 test.mjs，然后 test.json，就像脚本放在同一目录中一样。
const script = new Script(
  `(async function() {
    const { filename } = await import('./test.mjs');
    return import(filename, { with: { type: 'json' } })
  })();`,
  {
    filename: resolve(__dirname, 'test-with-default.js'),
    importModuleDynamically: constants.USE_MAIN_CONTEXT_DEFAULT_LOADER,
  });

// { default: { hello: 'world' } }
script.runInThisContext().then(console.log);
```

使用主上下文的默认加载器加载用户模块有一些注意事项：

1. 被解析的模块将相对于传递给 `vm.Script` 或 `vm.compileFunction()` 的 `filename` 选项。解析可以使用绝对路径或 URL 字符串的 `filename`。如果 `filename` 既不是绝对路径也不是 URL 的字符串，或者它是 undefined，解析将相对于进程的当前工作目录。对于 `vm.createContext()`，解析始终相对于当前工作目录，因为此选项仅在不存在引用脚本或模块时使用。
2. 对于任何解析为特定路径的 `filename`，一旦进程成功从该路径加载特定模块，结果可能会被缓存，随后从同一路径加载相同模块将返回相同的内容。如果 `filename` 是 URL 字符串，如果它具有不同的搜索参数，则缓存将不会被命中。对于不是 URL 字符串的 `filename`，目前无法绕过缓存行为。

### 当 `importModuleDynamically` 是一个函数时

当 `importModuleDynamically` 是一个函数时，它将在编译的代码中调用 `import()` 时被调用，以便用户自定义应如何编译和评估请求的模块。目前，必须使用 `--experimental-vm-modules` 标志启动 Node.js 实例才能使此选项工作。如果未设置该标志，此回调将被忽略。如果评估的代码实际调用 `import()`，结果将因 [`ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING_FLAG`][] 而拒绝。

回调 `importModuleDynamically(specifier, referrer, importAttributes)` 具有以下签名：

* `specifier` {string} 传递给 `import()` 的说明符
* `referrer` {vm.Script|Function|vm.SourceTextModule|Object}
  引用者是 `new vm.Script`、`vm.runInThisContext`、`vm.runInContext` 和 `vm.runInNewContext` 的已编译 `vm.Script`。它是 `vm.compileFunction` 的已编译 `Function`，`new vm.SourceTextModule` 的已编译 `vm.SourceTextModule`，以及 `vm.createContext()` 的上下文 `Object`。
* `importAttributes` {Object} 传递给 [`optionsExpression`][] 可选参数的 `"with"` 值，如果未提供值则为空对象。
* `phase` {string} 动态导入的阶段（`"source"` 或 `"evaluation"`）。
* 返回: {Module Namespace Object|vm.Module} 返回 `vm.Module` 是推荐的，以便利用错误跟踪，并避免包含 `then` 函数导出的命名空间出现问题。

```mjs
// 此脚本必须使用 --experimental-vm-modules 运行。
import { Script, SyntheticModule } from 'node:vm';

const script = new Script('import("foo.json", { with: { type: "json" } })', {
  async importModuleDynamically(specifier, referrer, importAttributes) {
    console.log(specifier);  // 'foo.json'
    console.log(referrer);   // 已编译的脚本
    console.log(importAttributes);  // { type: 'json' }
    const m = new SyntheticModule(['bar'], () => { });
    await m.link(() => { });
    m.setExport('bar', { hello: 'world' });
    return m;
  },
});
const result = await script.runInThisContext();
console.log(result);  //  { bar: { hello: 'world' } }
```

```cjs
// 此脚本必须使用 --experimental-vm-modules 运行。
const { Script, SyntheticModule } = require('node:vm');

(async function main() {
  const script = new Script('import("foo.json", { with: { type: "json" } })', {
    async importModuleDynamically(specifier, referrer, importAttributes) {
      console.log(specifier);  // 'foo.json'
      console.log(referrer);   // 已编译的脚本
      console.log(importAttributes);  // { type: 'json' }
      const m = new SyntheticModule(['bar'], () => { });
      await m.link(() => { });
      m.setExport('bar', { hello: 'world' });
      return m;
    },
  });
  const result = await script.runInThisContext();
  console.log(result);  //  { bar: { hello: 'world' } }
})();
```

[循环模块记录]: https://tc39.es/ecma262/#sec-cyclic-module-records
[ECMAScript 模块加载器]: esm.md#modules-ecmascript-modules
[Evaluate() 具体方法]: https://tc39.es/ecma262/#sec-moduleevaluation
[FinishLoadingImportedModule]: https://tc39.es/ecma262/#sec-FinishLoadingImportedModule
[GetModuleNamespace]: https://tc39.es/ecma262/#sec-getmodulenamespace
[HostLoadImportedModule]: https://tc39.es/ecma262/#sec-HostLoadImportedModule
[HostResolveImportedModule]: https://tc39.es/ecma262/#sec-hostresolveimportedmodule
[ImportDeclaration]: https://tc39.es/ecma262/#prod-ImportDeclaration
[Link() 具体方法]: https://tc39.es/ecma262/#sec-moduledeclarationlinking
[模块记录]: https://tc39.es/ecma262/#sec-abstract-module-records
[源文本模块记录]: https://tc39.es/ecma262/#sec-source-text-module-records
[编译 API 中对动态 `import()` 的支持]: #support-of-dynamic-import-in-compilation-apis
[合成模块记录]: https://tc39.es/ecma262/#sec-synthetic-module-records
[V8 嵌入者指南]: https://v8.dev/docs/embed#contexts
[WithClause]: https://tc39.es/ecma262/#prod-WithClause
[`ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING_FLAG`]: errors.md#err_vm_dynamic_import_callback_missing_flag
[`ERR_VM_DYNAMIC_IMPORT_CALLBACK_MISSING`]: errors.md#err_vm_dynamic_import_callback_missing
[`Error`]: errors.md#class-error
[`URL`]: url.md#class-url
[`eval()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval
[`optionsExpression`]: https://tc39.es/proposal-import-attributes/#sec-evaluate-import-call
[`script.runInContext()`]: #scriptrunincontextcontextifiedobject-options
[`script.runInThisContext()`]: #scriptruninthiscontextoptions
[`sourceTextModule.instantiate()`]: #sourcetextmoduleinstantiate
[`sourceTextModule.linkRequests(modules)`]: #sourcetextmodulelinkrequestsmodules
[`sourceTextModule.moduleRequests`]: #sourcetextmodulemodulerequests
[`url.origin`]: url.md#urlorigin
[`vm.compileFunction()`]: #vmcompilefunctioncode-params-options
[`vm.constants.DONT_CONTEXTIFY`]: #vmconstantsdont_contextify
[`vm.createContext()`]: #vmcreatecontextcontextobject-options
[`vm.runInContext()`]: #vmrunincontextcode-contextifiedobject-options
[`vm.runInThisContext()`]: #vmruninthiscontextcode-options
[contextified]: #what-does-it-mean-to-contextify-an-object
[排队作业]: https://tc39.es/ecma262/#sec-hostenqueuepromisejob
[全局对象]: https://tc39.es/ecma262/#sec-global-object
[间接的 `eval()` 调用]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/eval#direct_and_indirect_eval
[origin]: https://developer.mozilla.org/en-US/docs/Glossary/Origin