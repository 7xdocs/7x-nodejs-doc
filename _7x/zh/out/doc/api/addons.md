# Assert

<!--introduced_in=v0.1.21-->

> Stability: 2 - Stable

<!-- source_link=lib/assert.js -->

`node:assert` 模块提供了一组用于验证不变量的断言函数。

## 严格断言模式

<!-- YAML
added: v9.9.0
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/34001
    description: Exposed as `require('node:assert/strict')`.
  - version:
      - v13.9.0
      - v12.16.2
    pr-url: https://github.com/nodejs/node/pull/31635
    description: Changed "strict mode" to "strict assertion mode" and "legacy
                 mode" to "legacy assertion mode" to avoid confusion with the
                 more usual meaning of "strict mode".
  - version: v9.9.0
    pr-url: https://github.com/nodejs/node/pull/17615
    description: Added error diffs to the strict assertion mode.
  - version: v9.9.0
    pr-url: https://github.com/nodejs/node/pull/17002
    description: Added strict assertion mode to the assert module.
-->

在严格断言模式下，非严格方法的行为类似于对应的严格方法。例如，[`assert.deepEqual()`][] 的行为将类似于 [`assert.deepStrictEqual()`][]。

在严格断言模式下，对象的错误消息会显示差异。在传统断言模式下，对象的错误消息会显示对象，通常会被截断。

要使用严格断言模式：

```mjs
import { strict as assert } from 'node:assert';
```

```cjs
const assert = require('node:assert').strict;
```

```mjs
import assert from 'node:assert/strict';
```

```cjs
const assert = require('node:assert/strict');
```

错误差异示例：

```mjs
import { strict as assert } from 'node:assert';

assert.deepEqual([[[1, 2, 3]], 4, 5], [[[1, 2, '3']], 4, 5]);
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected ... Lines skipped
//
//   [
//     [
// ...
//       2,
// +     3
// -     '3'
//     ],
// ...
//     5
//   ]
```

```cjs
const assert = require('node:assert/strict');

assert.deepEqual([[[1, 2, 3]], 4, 5], [[[1, 2, '3']], 4, 5]);
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected ... Lines skipped
//
//   [
//     [
// ...
//       2,
// +     3
// -     '3'
//     ],
// ...
//     5
//   ]
```

要停用颜色，请使用 `NO_COLOR` 或 `NODE_DISABLE_COLORS` 环境变量。这也会停用 REPL 中的颜色。有关终端环境中颜色支持的更多信息，请阅读 tty 的 [`getColorDepth()`][] 文档。

## 传统断言模式

传统断言模式在以下方法中使用 [`==` 运算符][]：

* [`assert.deepEqual()`][]
* [`assert.equal()`][]
* [`assert.notDeepEqual()`][]
* [`assert.notEqual()`][]

要使用传统断言模式：

```mjs
import assert from 'node:assert';
```

```cjs
const assert = require('node:assert');
```

传统断言模式可能会产生令人惊讶的结果，特别是在使用 [`assert.deepEqual()`][] 时：

```cjs
// 警告：在传统断言模式下，这不会抛出 AssertionError！
assert.deepEqual(/a/gi, new Date());
```

## 类：`assert.AssertionError`

* 继承自：{errors.Error}

表示断言失败。由 `node:assert` 模块抛出的所有错误都将是 `AssertionError` 类的实例。

### `new assert.AssertionError(options)`

<!-- YAML
added: v0.1.21
-->

* `options` {Object}
  * `message` {string} 如果提供，错误消息将设置为此值。
  * `actual` {any} 错误实例上的 `actual` 属性。
  * `expected` {any} 错误实例上的 `expected` 属性。
  * `operator` {string} 错误实例上的 `operator` 属性。
  * `stackStartFn` {Function} 如果提供，生成的堆栈跟踪将省略此函数之前的帧。
  * `diff` {string} 如果设置为 `'full'`，则在断言错误中显示完整差异。默认为 `'simple'`。接受的值：`'simple'`、`'full'`。

{Error} 的一个子类，表示断言失败。

所有实例都包含内置的 `Error` 属性（`message` 和 `name`）以及：

* `actual` {any} 设置为诸如 [`assert.strictEqual()`][] 等方法的 `actual` 参数。
* `expected` {any} 设置为诸如 [`assert.strictEqual()`][] 等方法的 `expected` 值。
* `generatedMessage` {boolean} 指示消息是否是自动生成的（`true`）。
* `code` {string} 值始终为 `ERR_ASSERTION`，以表明错误是断言错误。
* `operator` {string} 设置为传入的运算符值。

```mjs
import assert from 'node:assert';

// 生成一个 AssertionError 以稍后比较错误消息：
const { message } = new assert.AssertionError({
  actual: 1,
  expected: 2,
  operator: 'strictEqual',
});

// 验证错误输出：
try {
  assert.strictEqual(1, 2);
} catch (err) {
  assert(err instanceof assert.AssertionError);
  assert.strictEqual(err.message, message);
  assert.strictEqual(err.name, 'AssertionError');
  assert.strictEqual(err.actual, 1);
  assert.strictEqual(err.expected, 2);
  assert.strictEqual(err.code, 'ERR_ASSERTION');
  assert.strictEqual(err.operator, 'strictEqual');
  assert.strictEqual(err.generatedMessage, true);
}
```

```cjs
const assert = require('node:assert');

// 生成一个 AssertionError 以稍后比较错误消息：
const { message } = new assert.AssertionError({
  actual: 1,
  expected: 2,
  operator: 'strictEqual',
});

// 验证错误输出：
try {
  assert.strictEqual(1, 2);
} catch (err) {
  assert(err instanceof assert.AssertionError);
  assert.strictEqual(err.message, message);
  assert.strictEqual(err.name, 'AssertionError');
  assert.strictEqual(err.actual, 1);
  assert.strictEqual(err.expected, 2);
  assert.strictEqual(err.code, 'ERR_ASSERTION');
  assert.strictEqual(err.operator, 'strictEqual');
  assert.strictEqual(err.generatedMessage, true);
}
```

## 类：`assert.Assert`

<!-- YAML
added:
 - v24.6.0
 - v22.19.0
-->

`Assert` 类允许创建具有自定义选项的独立断言实例。

### `new assert.Assert([options])`

<!-- YAML
changes:
  - version: v24.9.0
    pr-url: https://github.com/nodejs/node/pull/59762
    description: Added `skipPrototype` option.
-->

* `options` {Object}
  * `diff` {string} 如果设置为 `'full'`，则在断言错误中显示完整差异。默认为 `'simple'`。接受的值：`'simple'`、`'full'`。
  * `strict` {boolean} 如果设置为 `true`，非严格方法的行为类似于对应的严格方法。默认为 `true`。
  * `skipPrototype` {boolean} 如果设置为 `true`，则在深度相等检查中跳过原型和构造函数的比较。默认为 `false`。

创建一个新的断言实例。`diff` 选项控制断言错误消息中差异的详细程度。

```js
const { Assert } = require('node:assert');
const assertInstance = new Assert({ diff: 'full' });
assertInstance.deepStrictEqual({ a: 1 }, { a: 2 });
// 在错误消息中显示完整差异。
```

**重要**：当从 `Assert` 实例解构断言方法时，这些方法会失去与实例配置选项（如 `diff`、`strict` 和 `skipPrototype` 设置）的连接。解构的方法将回退到默认行为。

```js
const myAssert = new Assert({ diff: 'full' });

// 这按预期工作 - 使用 'full' 差异
myAssert.strictEqual({ a: 1 }, { b: { c: 1 } });

// 这失去了 'full' 差异设置 - 回退到默认的 'simple' 差异
const { strictEqual } = myAssert;
strictEqual({ a: 1 }, { b: { c: 1 } });
```

`skipPrototype` 选项影响所有深度相等方法：

```js
class Foo {
  constructor(a) {
    this.a = a;
  }
}

class Bar {
  constructor(a) {
    this.a = a;
  }
}

const foo = new Foo(1);
const bar = new Bar(1);

// 默认行为 - 由于不同的构造函数而失败
const assert1 = new Assert();
assert1.deepStrictEqual(foo, bar); // AssertionError

// 跳过原型比较 - 如果属性相等则通过
const assert2 = new Assert({ skipPrototype: true });
assert2.deepStrictEqual(foo, bar); // OK
```

当解构时，方法会失去对实例的 `this` 上下文的访问，并恢复为默认的断言行为（diff: 'simple'，非严格模式）。要在使用解构方法时保持自定义选项，请避免解构，并直接在实例上调用方法。

## `assert(value[, message])`

<!-- YAML
added: v0.5.9
-->

* `value` {any} 被检查为真值的输入。
* `message` {string|Error}

[`assert.ok()`][] 的别名。

## `assert.deepEqual(actual, expected[, message])`

<!-- YAML
added: v0.1.21
changes:
  - version: v25.0.0
    pr-url: https://github.com/nodejs/node/pull/59448
    description: Promises are not considered equal anymore if they are not of
                 the same instance.
  - version: v25.0.0
    pr-url: https://github.com/nodejs/node/pull/57627
    description: Invalid dates are now considered equal.
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57622
    description: Recursion now stops when either side encounters a circular
                 reference.
  - version:
      - v22.2.0
      - v20.15.0
    pr-url: https://github.com/nodejs/node/pull/51805
    description: Error cause and errors properties are now compared as well.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41020
    description: Regular expressions lastIndex property is now compared as well.
  - version:
      - v16.0.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38113
    description: In Legacy assertion mode, changed status from Deprecated to
                 Legacy.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/30766
    description: NaN is now treated as being identical if both sides are
                 NaN.
  - version: v12.0.0
    pr-url: https://github.com/nodejs/node/pull/25008
    description: The type tags are now properly compared and there are a couple
                 minor comparison adjustments to make the check less surprising.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15001
    description: The `Error` names and messages are now properly compared.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12142
    description: The `Set` and `Map` content is also compared.
  - version:
      - v6.4.0
      - v4.7.1
    pr-url: https://github.com/nodejs/node/pull/8002
    description: Typed array slices are handled correctly now.
  - version:
      - v6.1.0
      - v4.5.0
    pr-url: https://github.com/nodejs/node/pull/6432
    description: Objects with circular references can be used as inputs now.
  - version:
      - v5.10.1
      - v4.4.3
    pr-url: https://github.com/nodejs/node/pull/5910
    description: Handle non-`Uint8Array` typed arrays correctly.
-->

* `actual` {any}
* `expected` {any}
* `message` {string|Error}

**严格断言模式**

[`assert.deepStrictEqual()`][] 的别名。

**传统断言模式**

> Stability: 3 - Legacy: 请改用 [`assert.deepStrictEqual()`][]。

测试 `actual` 和 `expected` 参数之间的深度相等性。请考虑使用 [`assert.deepStrictEqual()`][] 代替。[`assert.deepEqual()`][] 可能会产生令人惊讶的结果。

_深度相等_ 意味着子对象的可枚举"自有"属性也通过以下规则递归评估。

### 比较细节

* 原始值使用 [`==` 运算符][] 进行比较，但 {NaN} 除外。如果两边都是 {NaN}，则被视为相同。
* 对象的[类型标签][Object.prototype.toString()] 应该相同。
* 只考虑[可枚举的"自有"属性][]。
* {Error} 的名称、消息、原因和错误总是被比较，即使这些不是可枚举属性。
* [对象包装器][] 既作为对象也作为解包后的值进行比较。
* `Object` 属性是无序比较的。
* {Map} 键和 {Set} 项是无序比较的。
* 当双方不同或任一方遇到循环引用时，递归停止。
* 实现不会测试对象的 [`[[Prototype]]`][prototype-spec]。
* {Symbol} 属性不会被比较。
* {WeakMap}、{WeakSet} 和 {Promise} 实例不会进行结构比较。只有当它们引用同一个对象时才相等。任何不同 `WeakMap`、`WeakSet` 或 `Promise` 实例之间的比较都将导致不相等，即使它们包含相同的内容。
* {RegExp} 的 lastIndex、flags 和 source 总是被比较，即使这些不是可枚举属性。

以下示例不会抛出 [`AssertionError`][]，因为原始值是使用 [`==` 运算符][] 进行比较的。

```mjs
import assert from 'node:assert';
// 警告：这不会抛出 AssertionError！

assert.deepEqual('+00000000', false);
```

```cjs
const assert = require('node:assert');
// 警告：这不会抛出 AssertionError！

assert.deepEqual('+00000000', false);
```

"深度"相等意味着子对象的可枚举"自有"属性也会被评估：

```mjs
import assert from 'node:assert';

const obj1 = {
  a: {
    b: 1,
  },
};
const obj2 = {
  a: {
    b: 2,
  },
};
const obj3 = {
  a: {
    b: 1,
  },
};
const obj4 = { __proto__: obj1 };

assert.deepEqual(obj1, obj1);
// OK

// b 的值不同：
assert.deepEqual(obj1, obj2);
// AssertionError: { a: { b: 1 } } deepEqual { a: { b: 2 } }

assert.deepEqual(obj1, obj3);
// OK

// 原型被忽略：
assert.deepEqual(obj1, obj4);
// AssertionError: { a: { b: 1 } } deepEqual {}
```

```cjs
const assert = require('node:assert');

const obj1 = {
  a: {
    b: 1,
  },
};
const obj2 = {
  a: {
    b: 2,
  },
};
const obj3 = {
  a: {
    b: 1,
  },
};
const obj4 = { __proto__: obj1 };

assert.deepEqual(obj1, obj1);
// OK

// b 的值不同：
assert.deepEqual(obj1, obj2);
// AssertionError: { a: { b: 1 } } deepEqual { a: { b: 2 } }

assert.deepEqual(obj1, obj3);
// OK

// 原型被忽略：
assert.deepEqual(obj1, obj4);
// AssertionError: { a: { b: 1 } } deepEqual {}
```

如果值不相等，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 [`AssertionError`][]。

## `assert.deepStrictEqual(actual, expected[, message])`

<!-- YAML
added: v1.2.0
changes:
  - version: v25.0.0
    pr-url: https://github.com/nodejs/node/pull/59448
    description: Promises are not considered equal anymore if they are not of
                 the same instance.
  - version: v25.0.0
    pr-url: https://github.com/nodejs/node/pull/57627
    description: Invalid dates are now considered equal.
  - version: v24.0.0
    pr-url: https://github.com/nodejs/node/pull/57622
    description: Recursion now stops when either side encounters a circular
                 reference.
  - version:
    - v22.2.0
    - v20.15.0
    pr-url: https://github.com/nodejs/node/pull/51805
    description: Error cause and errors properties are now compared as well.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41020
    description: Regular expressions lastIndex property is now compared as well.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15169
    description: Enumerable symbol properties are now compared.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15036
    description: The `NaN` is now compared using the
              [SameValueZero](https://tc39.github.io/ecma262/#sec-samevaluezero)
              comparison.
  - version: v8.5.0
    pr-url: https://github.com/nodejs/node/pull/15001
    description: The `Error` names and messages are now properly compared.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12142
    description: The `Set` and `Map` content is also compared.
  - version:
    - v6.4.0
    - v4.7.1
    pr-url: https://github.com/nodejs/node/pull/8002
    description: Typed array slices are handled correctly now.
  - version: v6.1.0
    pr-url: https://github.com/nodejs/node/pull/6432
    description: Objects with circular references can be used as inputs now.
  - version:
    - v5.10.1
    - v4.4.3
    pr-url: https://github.com/nodejs/node/pull/5910
    description: Handle non-`Uint8Array` typed arrays correctly.
-->

* `actual` {any}
* `expected` {any}
* `message` {string|Error}

测试 `actual` 和 `expected` 参数之间的深度严格相等性。"深度"相等意味着子对象的可枚举"自有"属性也通过以下规则递归评估。

### 比较细节

* 原始值使用 [`Object.is()`][] 进行比较。
* 对象的[类型标签][Object.prototype.toString()] 应该相同。
* 对象的 [`[[Prototype]]`][prototype-spec] 使用 [`===` 运算符][] 进行比较。
* 只考虑[可枚举的"自有"属性][]。
* {Error} 的名称、消息、原因和错误总是被比较，即使这些不是可枚举属性。`errors` 也会被比较。
* 可枚举的自有 {Symbol} 属性也会被比较。
* [对象包装器][] 既作为对象也作为解包后的值进行比较。
* `Object` 属性是无序比较的。
* {Map} 键和 {Set} 项是无序比较的。
* 当双方不同或任一方遇到循环引用时，递归停止。
* {WeakMap}、{WeakSet} 和 {Promise} 实例不会进行结构比较。只有当它们引用同一个对象时才相等。任何不同 `WeakMap`、`WeakSet` 或 `Promise` 实例之间的比较都将导致不相等，即使它们包含相同的内容。
* {RegExp} 的 lastIndex、flags 和 source 总是被比较，即使这些不是可枚举属性。

```mjs
import assert from 'node:assert/strict';

// 这失败是因为 1 !== '1'。
assert.deepStrictEqual({ a: 1 }, { a: '1' });
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
//   {
// +   a: 1
// -   a: '1'
//   }

// 以下对象没有自有属性
const date = new Date();
const object = {};
const fakeDate = {};
Object.setPrototypeOf(fakeDate, Date.prototype);

// 不同的 [[Prototype]]：
assert.deepStrictEqual(object, fakeDate);
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
// + {}
// - Date {}

// 不同的类型标签：
assert.deepStrictEqual(date, fakeDate);
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
// + 2018-04-26T00:49:08.604Z
// - Date {}

assert.deepStrictEqual(NaN, NaN);
// OK，因为 Object.is(NaN, NaN) 是 true。

// 不同的解包数字：
assert.deepStrictEqual(new Number(1), new Number(2));
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
// + [Number: 1]
// - [Number: 2]

assert.deepStrictEqual(new String('foo'), Object('foo'));
// OK，因为对象和字符串在解包后是相同的。

assert.deepStrictEqual(-0, -0);
// OK

// 不同的零：
assert.deepStrictEqual(0, -0);
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
// + 0
// - -0

const symbol1 = Symbol();
const symbol2 = Symbol();
assert.deepStrictEqual({ [symbol1]: 1 }, { [symbol1]: 1 });
// OK，因为两个对象上的是同一个符号。

assert.deepStrictEqual({ [symbol1]: 1 }, { [symbol2]: 1 });
// AssertionError [ERR_ASSERTION]: Inputs identical but not reference equal:
//
// {
//   Symbol(): 1
// }

const weakMap1 = new WeakMap();
const weakMap2 = new WeakMap();
const obj = {};

weakMap1.set(obj, 'value');
weakMap2.set(obj, 'value');

// 比较不同的实例失败，即使内容相同
assert.deepStrictEqual(weakMap1, weakMap2);
// AssertionError: Values have same structure but are not reference-equal:
//
// WeakMap {
//   <items unknown>
// }

// 比较同一个实例自身成功
assert.deepStrictEqual(weakMap1, weakMap1);
// OK

const weakSet1 = new WeakSet();
const weakSet2 = new WeakSet();
weakSet1.add(obj);
weakSet2.add(obj);

// 比较不同的实例失败，即使内容相同
assert.deepStrictEqual(weakSet1, weakSet2);
// AssertionError: Values have same structure but are not reference-equal:
// + actual - expected
//
// WeakSet {
//   <items unknown>
// }

// 比较同一个实例自身成功
assert.deepStrictEqual(weakSet1, weakSet1);
// OK
```

```cjs
const assert = require('node:assert/strict');

// 这失败是因为 1 !== '1'。
assert.deepStrictEqual({ a: 1 }, { a: '1' });
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
//   {
// +   a: 1
// -   a: '1'
//   }

// 以下对象没有自有属性
const date = new Date();
const object = {};
const fakeDate = {};
Object.setPrototypeOf(fakeDate, Date.prototype);

// 不同的 [[Prototype]]：
assert.deepStrictEqual(object, fakeDate);
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
// + {}
// - Date {}

// 不同的类型标签：
assert.deepStrictEqual(date, fakeDate);
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
// + 2018-04-26T00:49:08.604Z
// - Date {}

assert.deepStrictEqual(NaN, NaN);
// OK，因为 Object.is(NaN, NaN) 是 true。

// 不同的解包数字：
assert.deepStrictEqual(new Number(1), new Number(2));
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
// + [Number: 1]
// - [Number: 2]

assert.deepStrictEqual(new String('foo'), Object('foo'));
// OK，因为对象和字符串在解包后是相同的。

assert.deepStrictEqual(-0, -0);
// OK

// 不同的零：
assert.deepStrictEqual(0, -0);
// AssertionError: Expected inputs to be strictly deep-equal:
// + actual - expected
//
// + 0
// - -0

const symbol1 = Symbol();
const symbol2 = Symbol();
assert.deepStrictEqual({ [symbol1]: 1 }, { [symbol1]: 1 });
// OK，因为两个对象上的是同一个符号。

assert.deepStrictEqual({ [symbol1]: 1 }, { [symbol2]: 1 });
// AssertionError [ERR_ASSERTION]: Inputs identical but not reference equal:
//
// {
//   Symbol(): 1
// }

const weakMap1 = new WeakMap();
const weakMap2 = new WeakMap();
const obj = {};

weakMap1.set(obj, 'value');
weakMap2.set(obj, 'value');

// 比较不同的实例失败，即使内容相同
assert.deepStrictEqual(weakMap1, weakMap2);
// AssertionError: Values have same structure but are not reference-equal:
//
// WeakMap {
//   <items unknown>
// }

// 比较同一个实例自身成功
assert.deepStrictEqual(weakMap1, weakMap1);
// OK

const weakSet1 = new WeakSet();
const weakSet2 = new WeakSet();
weakSet1.add(obj);
weakSet2.add(obj);

// 比较不同的实例失败，即使内容相同
assert.deepStrictEqual(weakSet1, weakSet2);
// AssertionError: Values have same structure but are not reference-equal:
// + actual - expected
//
// WeakSet {
//   <items unknown>
// }

// 比较同一个实例自身成功
assert.deepStrictEqual(weakSet1, weakSet1);
// OK
```

如果值不相等，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 `AssertionError`。

## `assert.doesNotMatch(string, regexp[, message])`

<!-- YAML
added:
  - v13.6.0
  - v12.16.0
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/38111
    description: This API is no longer experimental.
-->

* `string` {string}
* `regexp` {RegExp}
* `message` {string|Error}

期望 `string` 输入不匹配正则表达式。

```mjs
import assert from 'node:assert/strict';

assert.doesNotMatch('I will fail', /fail/);
// AssertionError [ERR_ASSERTION]: The input was expected to not match the ...

assert.doesNotMatch(123, /pass/);
// AssertionError [ERR_ASSERTION]: The "string" argument must be of type string.

assert.doesNotMatch('I will pass', /different/);
// OK
```

```cjs
const assert = require('node:assert/strict');

assert.doesNotMatch('I will fail', /fail/);
// AssertionError [ERR_ASSERTION]: The input was expected to not match the ...

assert.doesNotMatch(123, /pass/);
// AssertionError [ERR_ASSERTION]: The "string" argument must be of type string.

assert.doesNotMatch('I will pass', /different/);
// OK
```

如果值匹配，或者 `string` 参数是除 `string` 以外的其他类型，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 [`AssertionError`][]。

## `assert.doesNotReject(asyncFn[, error][, message])`

<!-- YAML
added: v10.0.0
-->

* `asyncFn` {Function|Promise}
* `error` {RegExp|Function}
* `message` {string}
* 返回：{Promise}

等待 `asyncFn` 承诺完成，或者如果 `asyncFn` 是一个函数，则立即调用该函数并等待返回的承诺完成。然后检查承诺是否未被拒绝。

如果 `asyncFn` 是一个函数并且它同步抛出错误，`assert.doesNotReject()` 将返回一个被拒绝的 `Promise`，并带有该错误。如果该函数不返回承诺，`assert.doesNotReject()` 将返回一个被拒绝的 `Promise`，并带有 [`ERR_INVALID_RETURN_VALUE`][] 错误。在这两种情况下，错误处理程序都会被跳过。

使用 `assert.doesNotReject()` 实际上并不有用，因为捕获拒绝然后再次拒绝它没有什么好处。相反，考虑在不应拒绝的特定代码路径旁边添加注释，并保持错误消息尽可能具有表现力。

如果指定，`error` 可以是 [`Class`][]、{RegExp} 或验证函数。有关更多详细信息，请参阅 [`assert.throws()`][]。

除了等待完成的异步性质外，其行为与 [`assert.doesNotThrow()`][] 相同。

```mjs
import assert from 'node:assert/strict';

await assert.doesNotReject(
  async () => {
    throw new TypeError('Wrong value');
  },
  SyntaxError,
);
```

```cjs
const assert = require('node:assert/strict');

(async () => {
  await assert.doesNotReject(
    async () => {
      throw new TypeError('Wrong value');
    },
    SyntaxError,
  );
})();
```

```mjs
import assert from 'node:assert/strict';

assert.doesNotReject(Promise.reject(new TypeError('Wrong value')))
  .then(() => {
    // ...
  });
```

```cjs
const assert = require('node:assert/strict');

assert.doesNotReject(Promise.reject(new TypeError('Wrong value')))
  .then(() => {
    // ...
  });
```

## `assert.doesNotThrow(fn[, error][, message])`

<!-- YAML
added: v0.1.21
changes:
  - version:
    - v5.11.0
    - v4.4.5
    pr-url: https://github.com/nodejs/node/pull/2407
    description: The `message` parameter is respected now.
  - version: v4.2.0
    pr-url: https://github.com/nodejs/node/pull/3276
    description: The `error` parameter can now be an arrow function.
-->

* `fn` {Function}
* `error` {RegExp|Function}
* `message` {string}

断言函数 `fn` 不会抛出错误。

使用 `assert.doesNotThrow()` 实际上并不有用，因为捕获错误然后重新抛出它没有什么好处。相反，考虑在不应抛出的特定代码路径旁边添加注释，并保持错误消息尽可能具有表现力。

当调用 `assert.doesNotThrow()` 时，它会立即调用 `fn` 函数。

如果抛出错误并且它与 `error` 参数指定的类型相同，则抛出 [`AssertionError`][]。如果错误是不同类型，或者 `error` 参数未定义，则错误将传播回调用者。

如果指定，`error` 可以是 [`Class`][]、{RegExp} 或验证函数。有关更多详细信息，请参阅 [`assert.throws()`][]。

例如，以下将抛出 {TypeError}，因为断言中没有匹配的错误类型：

```mjs
import assert from 'node:assert/strict';

assert.doesNotThrow(
  () => {
    throw new TypeError('Wrong value');
  },
  SyntaxError,
);
```

```cjs
const assert = require('node:assert/strict');

assert.doesNotThrow(
  () => {
    throw new TypeError('Wrong value');
  },
  SyntaxError,
);
```

但是，以下将导致带有消息“Got unwanted exception...”的 [`AssertionError`][]：

```mjs
import assert from 'node:assert/strict';

assert.doesNotThrow(
  () => {
    throw new TypeError('Wrong value');
  },
  TypeError,
);
```

```cjs
const assert = require('node:assert/strict');

assert.doesNotThrow(
  () => {
    throw new TypeError('Wrong value');
  },
  TypeError,
);
```

如果抛出 [`AssertionError`][] 并且为 `message` 参数提供了值，则 `message` 的值将附加到 [`AssertionError`][] 消息：

```mjs
import assert from 'node:assert/strict';

assert.doesNotThrow(
  () => {
    throw new TypeError('Wrong value');
  },
  /Wrong value/,
  'Whoops',
);
// Throws: AssertionError: Got unwanted exception: Whoops
```

```cjs
const assert = require('node:assert/strict');

assert.doesNotThrow(
  () => {
    throw new TypeError('Wrong value');
  },
  /Wrong value/,
  'Whoops',
);
// Throws: AssertionError: Got unwanted exception: Whoops
```

## `assert.equal(actual, expected[, message])`

<!-- YAML
added: v0.1.21
changes:
  - version:
      - v16.0.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38113
    description: In Legacy assertion mode, changed status from Deprecated to
                 Legacy.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/30766
    description: NaN is now treated as being identical if both sides are
                 NaN.
-->

* `actual` {any}
* `expected` {any}
* `message` {string|Error}

**严格断言模式**

[`assert.strictEqual()`][] 的别名。

**传统断言模式**

> Stability: 3 - Legacy: 请改用 [`assert.strictEqual()`][]。

使用 [`==` 运算符][] 测试 `actual` 和 `expected` 参数之间的浅层强制相等性。`NaN` 被特殊处理，如果两边都是 `NaN`，则被视为相同。

```mjs
import assert from 'node:assert';

assert.equal(1, 1);
// OK, 1 == 1
assert.equal(1, '1');
// OK, 1 == '1'
assert.equal(NaN, NaN);
// OK

assert.equal(1, 2);
// AssertionError: 1 == 2
assert.equal({ a: { b: 1 } }, { a: { b: 1 } });
// AssertionError: { a: { b: 1 } } == { a: { b: 1 } }
```

```cjs
const assert = require('node:assert');

assert.equal(1, 1);
// OK, 1 == 1
assert.equal(1, '1');
// OK, 1 == '1'
assert.equal(NaN, NaN);
// OK

assert.equal(1, 2);
// AssertionError: 1 == 2
assert.equal({ a: { b: 1 } }, { a: { b: 1 } });
// AssertionError: { a: { b: 1 } } == { a: { b: 1 } }
```

如果值不相等，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 `AssertionError`。

## `assert.fail([message])`

<!-- YAML
added: v0.1.21
-->

* `message` {string|Error} **默认值：** `'Failed'`

抛出带有提供的错误消息或默认错误消息的 [`AssertionError`][]。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 [`AssertionError`][]。

```mjs
import assert from 'node:assert/strict';

assert.fail();
// AssertionError [ERR_ASSERTION]: Failed

assert.fail('boom');
// AssertionError [ERR_ASSERTION]: boom

assert.fail(new TypeError('need array'));
// TypeError: need array
```

```cjs
const assert = require('node:assert/strict');

assert.fail();
// AssertionError [ERR_ASSERTION]: Failed

assert.fail('boom');
// AssertionError [ERR_ASSERTION]: boom

assert.fail(new TypeError('need array'));
// TypeError: need array
```

## `assert.ifError(value)`

<!-- YAML
added: v0.1.97
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18247
    description: Instead of throwing the original error it is now wrapped into
                 an [`AssertionError`][] that contains the full stack trace.
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18247
    description: Value may now only be `undefined` or `null`. Before all falsy
                 values were handled the same as `null` and did not throw.
-->

* `value` {any}

如果 `value` 不是 `undefined` 或 `null`，则抛出 `value`。这在测试回调中的 `error` 参数时很有用。堆栈跟踪包含从传递给 `ifError()` 的错误的所有帧，包括 `ifError()` 本身的潜在新帧。

```mjs
import assert from 'node:assert/strict';

assert.ifError(null);
// OK
assert.ifError(0);
// AssertionError [ERR_ASSERTION]: ifError got unwanted exception: 0
assert.ifError('error');
// AssertionError [ERR_ASSERTION]: ifError got unwanted exception: 'error'
assert.ifError(new Error());
// AssertionError [ERR_ASSERTION]: ifError got unwanted exception: Error

// 创建一些随机错误帧。
let err;
(function errorFrame() {
  err = new Error('test error');
})();

(function ifErrorFrame() {
  assert.ifError(err);
})();
// AssertionError [ERR_ASSERTION]: ifError got unwanted exception: test error
//     at ifErrorFrame
//     at errorFrame
```

```cjs
const assert = require('node:assert/strict');

assert.ifError(null);
// OK
assert.ifError(0);
// AssertionError [ERR_ASSERTION]: ifError got unwanted exception: 0
assert.ifError('error');
// AssertionError [ERR_ASSERTION]: ifError got unwanted exception: 'error'
assert.ifError(new Error());
// AssertionError [ERR_ASSERTION]: ifError got unwanted exception: Error

// 创建一些随机错误帧。
let err;
(function errorFrame() {
  err = new Error('test error');
})();

(function ifErrorFrame() {
  assert.ifError(err);
})();
// AssertionError [ERR_ASSERTION]: ifError got unwanted exception: test error
//     at ifErrorFrame
//     at errorFrame
```

## `assert.match(string, regexp[, message])`

<!-- YAML
added:
  - v13.6.0
  - v12.16.0
changes:
  - version: v16.0.0
    pr-url: https://github.com/nodejs/node/pull/38111
    description: This API is no longer experimental.
-->

* `string` {string}
* `regexp` {RegExp}
* `message` {string|Error}

期望 `string` 输入匹配正则表达式。

```mjs
import assert from 'node:assert/strict';

assert.match('I will fail', /pass/);
// AssertionError [ERR_ASSERTION]: The input did not match the regular ...

assert.match(123, /pass/);
// AssertionError [ERR_ASSERTION]: The "string" argument must be of type string.

assert.match('I will pass', /pass/);
// OK
```

```cjs
const assert = require('node:assert/strict');

assert.match('I will fail', /pass/);
// AssertionError [ERR_ASSERTION]: The input did not match the regular ...

assert.match(123, /pass/);
// AssertionError [ERR_ASSERTION]: The "string" argument must be of type string.

assert.match('I will pass', /pass/);
// OK
```

如果值不匹配，或者 `string` 参数是除 `string` 以外的其他类型，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 [`AssertionError`][]。

## `assert.notDeepEqual(actual, expected[, message])`

<!-- YAML
added: v0.1.21
changes:
  - version:
      - v16.0.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38113
    description: In Legacy assertion mode, changed status from Deprecated to
                 Legacy.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/30766
    description: NaN is now treated as being identical if both sides are
                 NaN.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15001
    description: The `Error` names and messages are now properly compared.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12142
    description: The `Set` and `Map` content is also compared.
  - version:
      - v6.4.0
      - v4.7.1
    pr-url: https://github.com/nodejs/node/pull/8002
    description: Typed array slices are handled correctly now.
  - version:
      - v6.1.0
      - v4.5.0
    pr-url: https://github.com/nodejs/node/pull/6432
    description: Objects with circular references can be used as inputs now.
  - version:
      - v5.10.1
      - v4.4.3
    pr-url: https://github.com/nodejs/node/pull/5910
    description: Handle non-`Uint8Array` typed arrays correctly.
-->

* `actual` {any}
* `expected` {any}
* `message` {string|Error}

**严格断言模式**

[`assert.notDeepStrictEqual()`][] 的别名。

**传统断言模式**

> Stability: 3 - Legacy: 请改用 [`assert.notDeepStrictEqual()`][]。

测试任何深度不相等。与 [`assert.deepEqual()`][] 相反。

```mjs
import assert from 'node:assert';

const obj1 = {
  a: {
    b: 1,
  },
};
const obj2 = {
  a: {
    b: 2,
  },
};
const obj3 = {
  a: {
    b: 1,
  },
};
const obj4 = { __proto__: obj1 };

assert.notDeepEqual(obj1, obj1);
// AssertionError: { a: { b: 1 } } notDeepEqual { a: { b: 1 } }

assert.notDeepEqual(obj1, obj2);
// OK

assert.notDeepEqual(obj1, obj3);
// AssertionError: { a: { b: 1 } } notDeepEqual { a: { b: 1 } }

assert.notDeepEqual(obj1, obj4);
// OK
```

```cjs
const assert = require('node:assert');

const obj1 = {
  a: {
    b: 1,
  },
};
const obj2 = {
  a: {
    b: 2,
  },
};
const obj3 = {
  a: {
    b: 1,
  },
};
const obj4 = { __proto__: obj1 };

assert.notDeepEqual(obj1, obj1);
// AssertionError: { a: { b: 1 } } notDeepEqual { a: { b: 1 } }

assert.notDeepEqual(obj1, obj2);
// OK

assert.notDeepEqual(obj1, obj3);
// AssertionError: { a: { b: 1 } } notDeepEqual { a: { b: 1 } }

assert.notDeepEqual(obj1, obj4);
// OK
```

如果值深度相等，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 `AssertionError`。

## `assert.notDeepStrictEqual(actual, expected[, message])`

<!-- YAML
added: v1.2.0
changes:
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15398
    description: The `-0` and `+0` are not considered equal anymore.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15036
    description: The `NaN` is now compared using the
              [SameValueZero](https://tc39.github.io/ecma262/#sec-samevaluezero)
              comparison.
  - version: v9.0.0
    pr-url: https://github.com/nodejs/node/pull/15001
    description: The `Error` names and messages are now properly compared.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12142
    description: The `Set` and `Map` content is also compared.
  - version:
    - v6.4.0
    - v4.7.1
    pr-url: https://github.com/nodejs/node/pull/8002
    description: Typed array slices are handled correctly now.
  - version: v6.1.0
    pr-url: https://github.com/nodejs/node/pull/6432
    description: Objects with circular references can be used as inputs now.
  - version:
    - v5.10.1
    - v4.4.3
    pr-url: https://github.com/nodejs/node/pull/5910
    description: Handle non-`Uint8Array` typed arrays correctly.
-->

* `actual` {any}
* `expected` {any}
* `message` {string|Error}

测试深度严格不相等。与 [`assert.deepStrictEqual()`][] 相反。

```mjs
import assert from 'node:assert/strict';

assert.notDeepStrictEqual({ a: 1 }, { a: '1' });
// OK
```

```cjs
const assert = require('node:assert/strict');

assert.notDeepStrictEqual({ a: 1 }, { a: '1' });
// OK
```

如果值深度严格相等，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 [`AssertionError`][]。

## `assert.notEqual(actual, expected[, message])`

<!-- YAML
added: v0.1.21
changes:
  - version:
      - v16.0.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/38113
    description: In Legacy assertion mode, changed status from Deprecated to
                 Legacy.
  - version: v14.0.0
    pr-url: https://github.com/nodejs/node/pull/30766
    description: NaN is now treated as being identical if both sides are
                 NaN.
-->

* `actual` {any}
* `expected` {any}
* `message` {string|Error}

**严格断言模式**

[`assert.notStrictEqual()`][] 的别名。

**传统断言模式**

> Stability: 3 - Legacy: 请改用 [`assert.notStrictEqual()`][]。

使用 [`!=` 运算符][] 测试浅层强制不相等。`NaN` 被特殊处理，如果两边都是 `NaN`，则被视为相同。

```mjs
import assert from 'node:assert';

assert.notEqual(1, 2);
// OK

assert.notEqual(1, 1);
// AssertionError: 1 != 1

assert.notEqual(1, '1');
// AssertionError: 1 != '1'
```

```cjs
const assert = require('node:assert');

assert.notEqual(1, 2);
// OK

assert.notEqual(1, 1);
// AssertionError: 1 != 1

assert.notEqual(1, '1');
// AssertionError: 1 != '1'
```

如果值相等，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 `AssertionError`。

## `assert.notStrictEqual(actual, expected[, message])`

<!-- YAML
added: v0.1.21
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/17003
    description: Used comparison changed from Strict Equality to `Object.is()`.
-->

* `actual` {any}
* `expected` {any}
* `message` {string|Error}

测试 `actual` 和 `expected` 参数之间的严格不相等性，由 [`Object.is()`][] 确定。

```mjs
import assert from 'node:assert/strict';

assert.notStrictEqual(1, 2);
// OK

assert.notStrictEqual(1, 1);
// AssertionError [ERR_ASSERTION]: Expected "actual" to be strictly unequal to:
//
// 1

assert.notStrictEqual(1, '1');
// OK
```

```cjs
const assert = require('node:assert/strict');

assert.notStrictEqual(1, 2);
// OK

assert.notStrictEqual(1, 1);
// AssertionError [ERR_ASSERTION]: Expected "actual" to be strictly unequal to:
//
// 1

assert.notStrictEqual(1, '1');
// OK
```

如果值严格相等，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 `AssertionError`。

## `assert.ok(value[, message])`

<!-- YAML
added: v0.1.21
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/18319
    description: The `assert.ok()` (no arguments) will now use a predefined
                 error message.
-->

* `value` {any}
* `message` {string|Error}

测试 `value` 是否为真值。它等效于 `assert.equal(!!value, true, message)`。

如果 `value` 不是真值，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数是 `undefined`，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 `AssertionError`。
如果根本没有传递任何参数，`message` 将被设置为字符串：``'No value argument passed to `assert.ok()`'``。

请注意，在 `repl` 中，错误消息将与文件中抛出的错误消息不同！有关更多详细信息，请参见下文。

```mjs
import assert from 'node:assert/strict';

assert.ok(true);
// OK
assert.ok(1);
// OK

assert.ok();
// AssertionError: No value argument passed to `assert.ok()`

assert.ok(false, 'it\'s false');
// AssertionError: it's false

// 在 repl 中：
assert.ok(typeof 123 === 'string');
// AssertionError: false == true

// 在文件中（例如 test.js）：
assert.ok(typeof 123 === 'string');
// AssertionError: The expression evaluated to a falsy value:
//
//   assert.ok(typeof 123 === 'string')

assert.ok(false);
// AssertionError: The expression evaluated to a falsy value:
//
//   assert.ok(false)

assert.ok(0);
// AssertionError: The expression evaluated to a falsy value:
//
//   assert.ok(0)
```

```cjs
const assert = require('node:assert/strict');

assert.ok(true);
// OK
assert.ok(1);
// OK

assert.ok();
// AssertionError: No value argument passed to `assert.ok()`

assert.ok(false, 'it\'s false');
// AssertionError: it's false

// 在 repl 中：
assert.ok(typeof 123 === 'string');
// AssertionError: false == true

// 在文件中（例如 test.js）：
assert.ok(typeof 123 === 'string');
// AssertionError: The expression evaluated to a falsy value:
//
//   assert.ok(typeof 123 === 'string')

assert.ok(false);
// AssertionError: The expression evaluated to a falsy value:
//
//   assert.ok(false)

assert.ok(0);
// AssertionError: The expression evaluated to a falsy value:
//
//   assert.ok(0)
```

```mjs
import assert from 'node:assert/strict';

// 使用 `assert()` 效果相同：
assert(0);
// AssertionError: The expression evaluated to a falsy value:
//
//   assert(0)
```

```cjs
const assert = require('node:assert');

// 使用 `assert()` 效果相同：
assert(0);
// AssertionError: The expression evaluated to a falsy value:
//
//   assert(0)
```

## `assert.rejects(asyncFn[, error][, message])`

<!-- YAML
added: v10.0.0
-->

* `asyncFn` {Function|Promise}
* `error` {RegExp|Function|Object|Error}
* `message` {string}
* 返回：{Promise}

等待 `asyncFn` 承诺完成，或者如果 `asyncFn` 是一个函数，则立即调用该函数并等待返回的承诺完成。然后检查承诺是否被拒绝。

如果 `asyncFn` 是一个函数并且它同步抛出错误，`assert.rejects()` 将返回一个被拒绝的 `Promise`，并带有该错误。如果该函数不返回承诺，`assert.rejects()` 将返回一个被拒绝的 `Promise`，并带有 [`ERR_INVALID_RETURN_VALUE`][] 错误。在这两种情况下，错误处理程序都会被跳过。

除了等待完成的异步性质外，其行为与 [`assert.throws()`][] 相同。

如果指定，`error` 可以是 [`Class`][]、{RegExp}、验证函数、将测试每个属性的对象，或错误实例，其中将测试每个属性，包括不可枚举的 `message` 和 `name` 属性。

如果指定，`message` 将是 [`AssertionError`][] 提供的消息，如果 `asyncFn` 未能拒绝。

```mjs
import assert from 'node:assert/strict';

await assert.rejects(
  async () => {
    throw new TypeError('Wrong value');
  },
  {
    name: 'TypeError',
    message: 'Wrong value',
  },
);
```

```cjs
const assert = require('node:assert/strict');

(async () => {
  await assert.rejects(
    async () => {
      throw new TypeError('Wrong value');
    },
    {
      name: 'TypeError',
      message: 'Wrong value',
    },
  );
})();
```

```mjs
import assert from 'node:assert/strict';

await assert.rejects(
  async () => {
    throw new TypeError('Wrong value');
  },
  (err) => {
    assert.strictEqual(err.name, 'TypeError');
    assert.strictEqual(err.message, 'Wrong value');
    return true;
  },
);
```

```cjs
const assert = require('node:assert/strict');

(async () => {
  await assert.rejects(
    async () => {
      throw new TypeError('Wrong value');
    },
    (err) => {
      assert.strictEqual(err.name, 'TypeError');
      assert.strictEqual(err.message, 'Wrong value');
      return true;
    },
  );
})();
```

```mjs
import assert from 'node:assert/strict';

assert.rejects(
  Promise.reject(new Error('Wrong value')),
  Error,
).then(() => {
  // ...
});
```

```cjs
const assert = require('node:assert/strict');

assert.rejects(
  Promise.reject(new Error('Wrong value')),
  Error,
).then(() => {
  // ...
});
```

`error` 不能是字符串。如果提供字符串作为第二个参数，则假定 `error` 被省略，该字符串将用于 `message`。这可能导致容易遗漏的错误。如果考虑使用字符串作为第二个参数，请仔细阅读 [`assert.throws()`][] 中的示例。

## `assert.strictEqual(actual, expected[, message])`

<!-- YAML
added: v0.1.21
changes:
  - version: v10.0.0
    pr-url: https://github.com/nodejs/node/pull/17003
    description: Used comparison changed from Strict Equality to `Object.is()`.
-->

* `actual` {any}
* `expected` {any}
* `message` {string|Error}

测试 `actual` 和 `expected` 参数之间的严格相等性，由 [`Object.is()`][] 确定。

```mjs
import assert from 'node:assert/strict';

assert.strictEqual(1, 2);
// AssertionError [ERR_ASSERTION]: Expected inputs to be strictly equal:
//
// 1 !== 2

assert.strictEqual(1, 1);
// OK

assert.strictEqual('Hello foobar', 'Hello World!');
// AssertionError [ERR_ASSERTION]: Expected inputs to be strictly equal:
// + actual - expected
//
// + 'Hello foobar'
// - 'Hello World!'
//          ^

const apples = 1;
const oranges = 2;
assert.strictEqual(apples, oranges, `apples ${apples} !== oranges ${oranges}`);
// AssertionError [ERR_ASSERTION]: apples 1 !== oranges 2

assert.strictEqual(1, '1', new TypeError('Inputs are not identical'));
// TypeError: Inputs are not identical
```

```cjs
const assert = require('node:assert/strict');

assert.strictEqual(1, 2);
// AssertionError [ERR_ASSERTION]: Expected inputs to be strictly equal:
//
// 1 !== 2

assert.strictEqual(1, 1);
// OK

assert.strictEqual('Hello foobar', 'Hello World!');
// AssertionError [ERR_ASSERTION]: Expected inputs to be strictly equal:
// + actual - expected
//
// + 'Hello foobar'
// - 'Hello World!'
//          ^

const apples = 1;
const oranges = 2;
assert.strictEqual(apples, oranges, `apples ${apples} !== oranges ${oranges}`);
// AssertionError [ERR_ASSERTION]: apples 1 !== oranges 2

assert.strictEqual(1, '1', new TypeError('Inputs are not identical'));
// TypeError: Inputs are not identical
```

如果值不严格相等，将抛出 [`AssertionError`][]，其 `message` 属性设置为 `message` 参数的值。如果 `message` 参数未定义，则分配默认错误消息。如果 `message` 参数是 {Error} 的实例，则将抛出该错误而不是 [`AssertionError`][]。

## `assert.throws(fn[, error][, message])`

<!-- YAML
added: v0.1.21
changes:
  - version: v10.2.0
    pr-url: https://github.com/nodejs/node/pull/20485
    description: The `error` parameter can be an object containing regular
                 expressions now.
  - version: v9.9.0
    pr-url: https://github.com/nodejs/node/pull/17584
    description: The `error` parameter can now be an object as well.
  - version: v4.2.0
    pr-url: https://github.com/nodejs/node/pull/3276
    description: The `error` parameter can now be an arrow function.
-->

* `fn` {Function}
* `error` {RegExp|Function|Object|Error}
* `message` {string}

期望函数 `fn` 抛出错误。

如果指定，`error` 可以是 [`Class`][]、{RegExp}、验证函数、将测试每个属性以进行严格深度相等的验证对象，或错误实例，其中将测试每个属性以进行严格深度相等，包括不可枚举的 `message` 和 `name` 属性。使用对象时，也可以使用正则表达式，当针对字符串属性进行验证时。参见以下示例。

如果指定，`message` 将附加到 `AssertionError` 提供的消息中，如果 `fn` 调用未能抛出或错误验证失败。

自定义验证对象/错误实例：

```mjs
import assert from 'node:assert/strict';

const err = new TypeError('Wrong value');
err.code = 404;
err.foo = 'bar';
err.info = {
  nested: true,
  baz: 'text',
};
err.reg = /abc/i;

assert.throws(
  () => {
    throw err;
  },
  {
    name: 'TypeError',
    message: 'Wrong value',
    info: {
      nested: true,
      baz: 'text',
    },
    // 只有验证对象上的属性将被测试。
    // 使用嵌套对象要求所有属性都存在。否则验证将失败。
  },
);

// 使用正则表达式验证错误属性：
assert.throws(
  () => {
    throw err;
  },
  {
    // `name` 和 `message` 属性是字符串，对它们使用正则表达式将匹配字符串。如果失败，将抛出错误。
    name: /^TypeError$/,
    message: /Wrong/,
    foo: 'bar',
    info: {
      nested: true,
      // 不能对嵌套属性使用正则表达式！
      baz: 'text',
    },
    // `reg` 属性包含一个正则表达式，只有当验证对象包含相同的正则表达式时，才会通过。
    reg: /abc/i,
  },
);

// 由于不同的 `message` 和 `name` 属性而失败：
assert.throws(
  () => {
    const otherErr = new Error('Not found');
    // 从 `err` 复制所有可枚举属性到 `otherErr`。
    for (const [key, value] of Object.entries(err)) {
      otherErr[key] = value;
    }
    throw otherErr;
  },
  // 当使用错误作为验证对象时，错误的 `message` 和 `name` 属性也将被检查。
  err,
);
```

```cjs
const assert = require('node:assert/strict');

const err = new TypeError('Wrong value');
err.code = 404;
err.foo = 'bar';
err.info = {
  nested: true,
  baz: 'text',
};
err.reg = /abc/i;

assert.throws(
  () => {
    throw err;
  },
  {
    name: 'TypeError',
    message: 'Wrong value',
    info: {
      nested: true,
      baz: 'text',
    },
    // 只有验证对象上的属性将被测试。
    // 使用嵌套对象要求所有属性都存在。否则验证将失败。
  },
);

// 使用正则表达式验证错误属性：
assert.throws(
  () => {
    throw err;
  },
  {
    // `name` 和 `message` 属性是字符串，对它们使用正则表达式将匹配字符串。如果失败，将抛出错误。
    name: /^TypeError$/,
    message: /Wrong/,
    foo: 'bar',
    info: {
      nested: true,
      // 不能对嵌套属性使用正则表达式！
      baz: 'text',
    },
    // `reg` 属性包含一个正则表达式，只有当验证对象包含相同的正则表达式时，才会通过。
    reg: /abc/i,
  },
);

// 由于不同的 `message` 和 `name` 属性而失败：
assert.throws(
  () => {
    const otherErr = new Error('Not found');
    // 从 `err` 复制所有可枚举属性到 `otherErr`。
    for (const [key, value] of Object.entries(err)) {
      otherErr[key] = value;
    }
    throw otherErr;
  },
  // 当使用错误作为验证对象时，错误的 `message` 和 `name` 属性也将被检查。
  err,
);
```

使用构造函数验证 instanceof：

```mjs
import assert from 'node:assert/strict';

assert.throws(
  () => {
    throw new Error('Wrong value');
  },
  Error,
);
```

```cjs
const assert = require('node:assert/strict');

assert.throws(
  () => {
    throw new Error('Wrong value');
  },
  Error,
);
```

使用 {RegExp} 验证错误消息：

使用正则表达式会对错误对象运行 `.toString`，因此也会包括错误名称。

```mjs
import assert from 'node:assert/strict';

assert.throws(
  () => {
    throw new Error('Wrong value');
  },
  /^Error: Wrong value$/,
);
```

```cjs
const assert = require('node:assert/strict');

assert.throws(
  () => {
    throw new Error('Wrong value');
  },
  /^Error: Wrong value$/,
);
```

自定义错误验证：

该函数必须返回 `true` 以指示所有内部验证通过。否则它将失败并抛出 [`AssertionError`][]。

```mjs
import assert from 'node:assert/strict';

assert.throws(
  () => {
    throw new Error('Wrong value');
  },
  (err) => {
    assert(err instanceof Error);
    assert(/value/.test(err));
    // 避免从验证函数返回除 `true` 以外的任何内容。
    // 否则，不清楚验证的哪部分失败。相反，抛出关于特定验证失败的错误（如本示例所示），并向该错误添加尽可能多的有帮助的调试信息。
    return true;
  },
  'unexpected error',
);
```

```cjs
const assert = require('node:assert/strict');

assert.throws(
  () => {
    throw new Error('Wrong value');
  },
  (err) => {
    assert(err instanceof Error);
    assert(/value/.test(err));
    // 避免从验证函数返回除 `true` 以外的任何内容。
    // 否则，不清楚验证的哪部分失败。相反，抛出关于特定验证失败的错误（如本示例所示），并向该错误添加尽可能多的有帮助的调试信息。
    return true;
  },
  'unexpected error',
);
```

`error` 不能是字符串。如果提供字符串作为第二个参数，则假定 `error` 被省略，该字符串将用于 `message`。这可能导致容易遗漏的错误。使用相同的消息作为抛出的错误消息将导致 `ERR_AMBIGUOUS_ARGUMENT` 错误。如果考虑使用字符串作为第二个参数，请仔细阅读以下示例：

```mjs
import assert from 'node:assert/strict';

function throwingFirst() {
  throw new Error('First');
}

function throwingSecond() {
  throw new Error('Second');
}

function notThrowing() {}

// 第二个参数是字符串，输入函数抛出了 Error。
// 第一种情况不会抛出，因为它与输入函数抛出的错误消息不匹配！
assert.throws(throwingFirst, 'Second');
// 在下一个示例中，消息与错误的消息相比没有好处，并且由于不清楚用户是否确实打算匹配错误消息，Node.js 抛出 `ERR_AMBIGUOUS_ARGUMENT` 错误。
assert.throws(throwingSecond, 'Second');
// TypeError [ERR_AMBIGUOUS_ARGUMENT]

// 字符串仅在函数未抛出时使用（作为消息）：
assert.throws(notThrowing, 'Second');
// AssertionError [ERR_ASSERTION]: Missing expected exception: Second

// 如果打算匹配错误消息，请改为执行此操作：
// 因为错误消息匹配，所以不会抛出。
assert.throws(throwingSecond, /Second$/);

// 如果错误消息不匹配，则抛出 AssertionError。
assert.throws(throwingFirst, /Second$/);
// AssertionError [ERR_ASSERTION]
```

```cjs
const assert = require('node:assert/strict');

function throwingFirst() {
  throw new Error('First');
}

function throwingSecond() {
  throw new Error('Second');
}

function notThrowing() {}

// 第二个参数是字符串，输入函数抛出了 Error。
// 第一种情况不会抛出，因为它与输入函数抛出的错误消息不匹配！
assert.throws(throwingFirst, 'Second');
// 在下一个示例中，消息与错误的消息相比没有好处，并且由于不清楚用户是否确实打算匹配错误消息，Node.js 抛出 `ERR_AMBIGUOUS_ARGUMENT` 错误。
assert.throws(throwingSecond, 'Second');
// TypeError [ERR_AMBIGUOUS_ARGUMENT]

// 字符串仅在函数未抛出时使用（作为消息）：
assert.throws(notThrowing, 'Second');
// AssertionError [ERR_ASSERTION]: Missing expected exception: Second

// 如果打算匹配错误消息，请改为执行此操作：
// 因为错误消息匹配，所以不会抛出。
assert.throws(throwingSecond, /Second$/);

// 如果错误消息不匹配，则抛出 AssertionError。
assert.throws(throwingFirst, /Second$/);
// AssertionError [ERR_ASSERTION]
```

由于容易混淆且容易出错的表示法，避免使用字符串作为第二个参数。

## `assert.partialDeepStrictEqual(actual, expected[, message])`

<!-- YAML
added:
  - v23.4.0
  - v22.13.0
changes:
  - version: v25.0.0
    pr-url: https://github.com/nodejs/node/pull/59448
    description: Promises are not considered equal anymore if they are not of
                 the same instance.
  - version: v25.0.0
    pr-url: https://github.com/nodejs/node/pull/57627
    description: Invalid dates are now considered equal.
  - version:
      - v24.0.0
      - v22.17.0
    pr-url: https://github.com/nodejs/node/pull/57370
    description: partialDeepStrictEqual is now Stable. Previously, it had been Experimental.
-->

* `actual` {any}
* `expected` {any}
* `message` {string|Error}

测试 `actual` 和 `expected` 参数之间的部分深度相等性。"深度"相等意味着子对象的可枚举"自有"属性也通过以下规则递归评估。"部分"相等意味着只比较存在于 `expected` 参数上的属性。

此方法始终通过与 [`assert.deepStrictEqual()`][] 相同的测试用例，表现为其超集。

### 比较细节

* 原始值使用 [`Object.is()`][] 进行比较。
* 对象的[类型标签][Object.prototype.toString()] 应该相同。
* 对象的 [`[[Prototype]]`][prototype-spec] 不会被比较。
* 只考虑[可枚举的"自有"属性][]。
* {Error} 的名称、消息、原因和错误总是被比较，即使这些不是可枚举属性。`errors` 也会被比较。
* 可枚举的自有 {Symbol} 属性也会被比较。
* [对象包装器][] 既作为对象也作为解包后的值进行比较。
* `Object` 属性是无序比较的。
* {Map} 键和 {Set} 项是无序比较的。
* 当双方不同或双方都遇到循环引用时，递归停止。
* {WeakMap}、{WeakSet} 和 {Promise} 实例不会进行结构比较。只有当它们引用同一个对象时才相等。任何不同 `WeakMap`、`WeakSet` 或 `Promise` 实例之间的比较都将导致不相等，即使它们包含相同的内容。
* {RegExp} 的 lastIndex、flags 和 source 总是被比较，即使这些不是可枚举属性。
* 稀疏数组中的空洞被忽略。

```mjs
import assert from 'node:assert';

assert.partialDeepStrictEqual(
  { a: { b: { c: 1 } } },
  { a: { b: { c: 1 } } },
);
// OK

assert.partialDeepStrictEqual(
  { a: 1, b: 2, c: 3 },
  { b: 2 },
);
// OK

assert.partialDeepStrictEqual(
  [1, 2, 3, 4, 5, 6, 7, 8, 9],
  [4, 5, 8],
);
// OK

assert.partialDeepStrictEqual(
  new Set([{ a: 1 }, { b: 1 }]),
  new Set([{ a: 1 }]),
);
// OK

assert.partialDeepStrictEqual(
  new Map([['key1', 'value1'], ['key2', 'value2']]),
  new Map([['key2', 'value2']]),
);
// OK

assert.partialDeepStrictEqual(123n, 123n);
// OK

assert.partialDeepStrictEqual(
  [1, 2, 3, 4, 5, 6, 7, 8, 9],
  [5, 4, 8],
);
// AssertionError

assert.partialDeepStrictEqual(
  { a: 1 },
  { a: 1, b: 2 },
);
// AssertionError

assert.partialDeepStrictEqual(
  { a: { b: 2 } },
  { a: { b: '2' } },
);
// AssertionError
```

```cjs
const assert = require('node:assert');

assert.partialDeepStrictEqual(
  { a: { b: { c: 1 } } },
  { a: { b: { c: 1 } } },
);
// OK

assert.partialDeepStrictEqual(
  { a: 1, b: 2, c: 3 },
  { b: 2 },
);
// OK

assert.partialDeepStrictEqual(
  [1, 2, 3, 4, 5, 6, 7, 8, 9],
  [4, 5, 8],
);
// OK

assert.partialDeepStrictEqual(
  new Set([{ a: 1 }, { b: 1 }]),
  new Set([{ a: 1 }]),
);
// OK

assert.partialDeepStrictEqual(
  new Map([['key1', 'value1'], ['key2', 'value2']]),
  new Map([['key2', 'value2']]),
);
// OK

assert.partialDeepStrictEqual(123n, 123n);
// OK

assert.partialDeepStrictEqual(
  [1, 2, 3, 4, 5, 6, 7, 8, 9],
  [5, 4, 8],
);
// AssertionError

assert.partialDeepStrictEqual(
  { a: 1 },
  { a: 1, b: 2 },
);
// AssertionError

assert.partialDeepStrictEqual(
  { a: { b: 2 } },
  { a: { b: '2' } },
);
// AssertionError
```

[Object wrappers]: https://developer.mozilla.org/en-US/docs/Glossary/Primitive#Primitive_wrapper_objects_in_JavaScript
[Object.prototype.toString()]: https://tc39.github.io/ecma262/#sec-object.prototype.tostring
[`!=` operator]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Inequality
[`===` operator]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Strict_equality
[`==` operator]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Equality
[`AssertionError`]: #class-assertassertionerror
[`Class`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Classes
[`ERR_INVALID_RETURN_VALUE`]: errors.md#err_invalid_return_value
[`Object.is()`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/Object/is
[`assert.deepEqual()`]: #assertdeepequalactual-expected-message
[`assert.deepStrictEqual()`]: #assertdeepstrictequalactual-expected-message
[`assert.doesNotThrow()`]: #assertdoesnotthrowfn-error-message
[`assert.equal()`]: #assertequalactual-expected-message
[`assert.notDeepEqual()`]: #assertnotdeepequalactual-expected-message
[`assert.notDeepStrictEqual()`]: #assertnotdeepstrictequalactual-expected-message
[`assert.notEqual()`]: #assertnotequalactual-expected-message
[`assert.notStrictEqual()`]: #assertnotstrictequalactual-expected-message
[`assert.ok()`]: #assertokvalue-message
[`assert.strictEqual()`]: #assertstrictequalactual-expected-message
[`assert.throws()`]: #assertthrowsfn-error-message
[`getColorDepth()`]: tty.md#writestreamgetcolordepthenv
[enumerable "own" properties]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Enumerability_and_ownership_of_properties
[prototype-spec]: https://tc39.github.io/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots