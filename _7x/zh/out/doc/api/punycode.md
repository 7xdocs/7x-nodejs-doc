# Punycode

<!-- YAML
deprecated: v7.0.0
-->

<!--introduced_in=v0.10.0-->

> Stability: 0 - Deprecated

<!-- source_link=lib/punycode.js -->

**Node.js 内置的 punycode 模块已被弃用。**
在未来的 Node.js 主要版本中，此模块将被移除。当前依赖 `punycode` 模块的用户应切换到使用用户提供的 [Punycode.js][] 模块。对于基于 punycode 的 URL 编码，请参阅 [`url.domainToASCII`][] 或更通用的 [WHATWG URL API][]。

`punycode` 模块是 [Punycode.js][] 模块的捆绑版本。可以通过以下方式访问：

```js
const punycode = require('node:punycode');
```

[Punycode][] 是由 RFC 3492 定义的一种字符编码方案，主要用于国际化域名。由于 URL 中的主机名仅限于 ASCII 字符，因此包含非 ASCII 字符的域名必须使用 Punycode 方案转换为 ASCII。例如，翻译成英文单词 `'example'` 的日文字符是 `'例'`。国际化域名 `'例.com'`（相当于 `'example.com'`）由 Punycode 表示为 ASCII 字符串 `'xn--fsq.com'`。

`punycode` 模块提供了 Punycode 标准的一个简单实现。

`punycode` 模块是 Node.js 使用的第三方依赖，为了方便开发者而提供。对该模块的修复或其他修改必须指向 [Punycode.js][] 项目。

## `punycode.decode(string)`

<!-- YAML
added: v0.5.1
-->

* `string` {string}

`punycode.decode()` 方法将仅包含 ASCII 字符的 [Punycode][] 字符串转换为等效的 Unicode 码点字符串。

```js
punycode.decode('maana-pta'); // 'mañana'
punycode.decode('--dqo34k'); // '☃-⌘'
```

## `punycode.encode(string)`

<!-- YAML
added: v0.5.1
-->

* `string` {string}

`punycode.encode()` 方法将 Unicode 码点字符串转换为仅包含 ASCII 字符的 [Punycode][] 字符串。

```js
punycode.encode('mañana'); // 'maana-pta'
punycode.encode('☃-⌘'); // '--dqo34k'
```

## `punycode.toASCII(domain)`

<!-- YAML
added: v0.6.1
-->

* `domain` {string}

`punycode.toASCII()` 方法将表示国际化域名的 Unicode 字符串转换为 [Punycode][]。只有域名中的非 ASCII 部分会被转换。在已经只包含 ASCII 字符的字符串上调用 `punycode.toASCII()` 将不会有任何效果。

```js
// encode domain names
punycode.toASCII('mañana.com');  // 'xn--maana-pta.com'
punycode.toASCII('☃-⌘.com');   // 'xn----dqo34k.com'
punycode.toASCII('example.com'); // 'example.com'
```

## `punycode.toUnicode(domain)`

<!-- YAML
added: v0.6.1
-->

* `domain` {string}

`punycode.toUnicode()` 方法将包含 [Punycode][] 编码字符的域名字符串转换为 Unicode。只有域名中 [Punycode][] 编码的部分会被转换。

```js
// decode domain names
punycode.toUnicode('xn--maana-pta.com'); // 'mañana.com'
punycode.toUnicode('xn----dqo34k.com');  // '☃-⌘.com'
punycode.toUnicode('example.com');       // 'example.com'
```

## `punycode.ucs2`

<!-- YAML
added: v0.7.0
-->

### `punycode.ucs2.decode(string)`

<!-- YAML
added: v0.7.0
-->

* `string` {string}

`punycode.ucs2.decode()` 方法返回一个数组，包含字符串中每个 Unicode 符号的数值码点。

```js
punycode.ucs2.decode('abc'); // [0x61, 0x62, 0x63]
// surrogate pair for U+1D306 tetragram for centre:
punycode.ucs2.decode('\uD834\uDF06'); // [0x1D306]
```

### `punycode.ucs2.encode(codePoints)`

<!-- YAML
added: v0.7.0
-->

* `codePoints` {integer\[]}

`punycode.ucs2.encode()` 方法基于数值码点值数组返回一个字符串。

```js
punycode.ucs2.encode([0x61, 0x62, 0x63]); // 'abc'
punycode.ucs2.encode([0x1D306]); // '\uD834\uDF06'
```

## `punycode.version`

<!-- YAML
added: v0.6.1
-->

* 类型: {string}

返回一个字符串，标识当前的 [Punycode.js][] 版本号。

[Punycode]: https://tools.ietf.org/html/rfc3492
[Punycode.js]: https://github.com/bestiejs/punycode.js
[WHATWG URL API]: url.md#the-whatwg-url-api
[`url.domainToASCII`]: url.md#urldomaintoasciidomain