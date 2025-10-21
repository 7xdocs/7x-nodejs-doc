# 域名系统 DNS

<!--introduced_in=v0.10.0-->

> Stability: 2 - Stable

<!-- source_link=lib/dns.js -->

`node:dns` 模块实现了域名解析功能。例如，可以使用它来查询主机名的 IP 地址。

虽然该模块以[域名系统 (DNS)][Domain Name System (DNS)]命名，但它并不总是使用 DNS 协议进行查找。[`dns.lookup()`][] 使用操作系统提供的设施来进行名称解析。它可能不需要执行任何网络通信。要以与系统上其他应用程序相同的方式执行名称解析，请使用 [`dns.lookup()`][]。

```mjs
import dns from 'node:dns';

dns.lookup('example.org', (err, address, family) => {
  console.log('address: %j family: IPv%s', address, family);
});
// address: "2606:2800:21f:cb07:6820:80da:af6b:8b2c" family: IPv6
```

```cjs
const dns = require('node:dns');

dns.lookup('example.org', (err, address, family) => {
  console.log('address: %j family: IPv%s', address, family);
});
// address: "2606:2800:21f:cb07:6820:80da:af6b:8b2c" family: IPv6
```

`node:dns` 模块中的所有其他函数都连接到实际的 DNS 服务器以执行名称解析。它们将始终使用网络来执行 DNS 查询。这些函数不使用 [`dns.lookup()`][] 所使用的同一组配置文件（例如 `/etc/hosts`）。使用这些函数可以始终执行 DNS 查询，绕过其他名称解析设施。

```mjs
import dns from 'node:dns';

dns.resolve4('archive.org', (err, addresses) => {
  if (err) throw err;

  console.log(`addresses: ${JSON.stringify(addresses)}`);

  addresses.forEach((a) => {
    dns.reverse(a, (err, hostnames) => {
      if (err) {
        throw err;
      }
      console.log(`reverse for ${a}: ${JSON.stringify(hostnames)}`);
    });
  });
});
```

```cjs
const dns = require('node:dns');

dns.resolve4('archive.org', (err, addresses) => {
  if (err) throw err;

  console.log(`addresses: ${JSON.stringify(addresses)}`);

  addresses.forEach((a) => {
    dns.reverse(a, (err, hostnames) => {
      if (err) {
        throw err;
      }
      console.log(`reverse for ${a}: ${JSON.stringify(hostnames)}`);
    });
  });
});
```

更多信息请参阅[实现考虑部分][Implementation considerations section]。

## 类：`dns.Resolver`

<!-- YAML
added: v8.3.0
-->

一个独立的 DNS 请求解析器。

创建新的解析器会使用默认的服务器设置。使用 [`resolver.setServers()`][`dns.setServers()`] 设置解析器使用的服务器不会影响其他解析器：

```mjs
import { Resolver } from 'node:dns';
const resolver = new Resolver();
resolver.setServers(['4.4.4.4']);

// 此请求将使用 4.4.4.4 的服务器，独立于全局设置。
resolver.resolve4('example.org', (err, addresses) => {
  // ...
});
```

```cjs
const { Resolver } = require('node:dns');
const resolver = new Resolver();
resolver.setServers(['4.4.4.4']);

// 此请求将使用 4.4.4.4 的服务器，独立于全局设置。
resolver.resolve4('example.org', (err, addresses) => {
  // ...
});
```

`node:dns` 模块中的以下方法可用：

* [`resolver.getServers()`][`dns.getServers()`]
* [`resolver.resolve()`][`dns.resolve()`]
* [`resolver.resolve4()`][`dns.resolve4()`]
* [`resolver.resolve6()`][`dns.resolve6()`]
* [`resolver.resolveAny()`][`dns.resolveAny()`]
* [`resolver.resolveCaa()`][`dns.resolveCaa()`]
* [`resolver.resolveCname()`][`dns.resolveCname()`]
* [`resolver.resolveMx()`][`dns.resolveMx()`]
* [`resolver.resolveNaptr()`][`dns.resolveNaptr()`]
* [`resolver.resolveNs()`][`dns.resolveNs()`]
* [`resolver.resolvePtr()`][`dns.resolvePtr()`]
* [`resolver.resolveSoa()`][`dns.resolveSoa()`]
* [`resolver.resolveSrv()`][`dns.resolveSrv()`]
* [`resolver.resolveTlsa()`][`dns.resolveTlsa()`]
* [`resolver.resolveTxt()`][`dns.resolveTxt()`]
* [`resolver.reverse()`][`dns.reverse()`]
* [`resolver.setServers()`][`dns.setServers()`]

### `Resolver([options])`

<!-- YAML
added: v8.3.0
changes:
  - version:
      - v16.7.0
      - v14.18.0
    pr-url: https://github.com/nodejs/node/pull/39610
    description: The `options` object now accepts a `tries` option.
  - version: v12.18.3
    pr-url: https://github.com/nodejs/node/pull/33472
    description: The constructor now accepts an `options` object.
                 The single supported option is `timeout`.
-->

创建一个新的解析器。

* `options` {Object}
  * `timeout` {integer} 查询超时时间，单位为毫秒，或 `-1` 使用默认超时时间。
  * `tries` {integer} 解析器在放弃之前尝试联系每个名称服务器的次数。**默认值：** `4`
  * `maxTimeout` {integer} 最大重试超时时间，单位为毫秒。**默认值：** `0`，禁用。

### `resolver.cancel()`

<!-- YAML
added: v8.3.0
-->

取消此解析器进行的所有未完成的 DNS 查询。相应的回调将以错误码 `ECANCELLED` 调用。

### `resolver.setLocalAddress([ipv4][, ipv6])`

<!-- YAML
added:
  - v15.1.0
  - v14.17.0
-->

* `ipv4` {string} IPv4 地址的字符串表示。**默认值：** `'0.0.0.0'`
* `ipv6` {string} IPv6 地址的字符串表示。**默认值：** `'::0'`

解析器实例将从指定的 IP 地址发送其请求。这允许程序在多宿主系统上使用时指定出站接口。

如果未指定 v4 或 v6 地址，则设置为默认值，操作系统将自动选择本地地址。

解析器在向 IPv4 DNS 服务器发出请求时将使用 v4 本地地址，在向 IPv6 DNS 服务器发出请求时将使用 v6 本地地址。解析请求的 `rrtype` 对使用的本地地址没有影响。

## `dns.getServers()`

<!-- YAML
added: v0.11.3
-->

* 返回：{string\[]}

返回一个 IP 地址字符串数组，根据 [RFC 5952][] 格式化，这些地址当前配置用于 DNS 解析。如果使用了自定义端口，字符串将包含端口部分。

<!-- eslint-disable @stylistic/js/semi-->

```js
[
  '8.8.8.8',
  '2001:4860:4860::8888',
  '8.8.8.8:1053',
  '[2001:4860:4860::8888]:1053',
]
```

## `dns.lookup(hostname[, options], callback)`

<!-- YAML
added: v0.1.90
changes:
  - version:
    - v22.1.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52492
    description: The `verbatim` option is now deprecated in favor of the new `order` option.
  - version: v18.4.0
    pr-url: https://github.com/nodejs/node/pull/43054
    description: For compatibility with `node:net`, when passing an option
                 object the `family` option can be the string `'IPv4'` or the
                 string `'IPv6'`.
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v17.0.0
    pr-url: https://github.com/nodejs/node/pull/39987
    description: The `verbatim` options defaults to `true` now.
  - version: v8.5.0
    pr-url: https://github.com/nodejs/node/pull/14731
    description: The `verbatim` option is supported now.
  - version: v1.2.0
    pr-url: https://github.com/nodejs/node/pull/744
    description: The `all` option is supported now.
-->

* `hostname` {string}
* `options` {integer | Object}
  * `family` {integer|string} 记录族。必须为 `4`、`6` 或 `0`。出于向后兼容的原因，`'IPv4'` 和 `'IPv6'` 分别被解释为 `4` 和 `6`。值 `0` 表示返回 IPv4 或 IPv6 地址。如果值 `0` 与 `{ all: true }` 一起使用（见下文），则返回 IPv4 和 IPv6 地址中的一个或两个，具体取决于系统的 DNS 解析器。**默认值：** `0`。
  * `hints` {number} 一个或多个[支持的 `getaddrinfo` 标志][supported `getaddrinfo` flags]。可以通过按位 `OR` 运算它们的值来传递多个标志。
  * `all` {boolean} 当为 `true` 时，回调返回所有已解析地址的数组。否则，返回单个地址。**默认值：** `false`。
  * `order` {string} 当为 `verbatim` 时，返回的地址未排序。当为 `ipv4first` 时，返回的地址通过将 IPv4 地址放在 IPv6 地址之前来排序。当为 `ipv6first` 时，返回的地址通过将 IPv6 地址放在 IPv4 地址之前来排序。**默认值：** `verbatim`（地址不重新排序）。默认值可使用 [`dns.setDefaultResultOrder()`][] 或 [`--dns-result-order`][] 配置。
  * `verbatim` {boolean} 当为 `true` 时，回调按 DNS 解析器返回的顺序接收 IPv4 和 IPv6 地址。当为 `false` 时，IPv4 地址放在 IPv6 地址之前。此选项将被弃用，转而使用 `order`。当两者都指定时，`order` 具有更高的优先级。新代码应仅使用 `order`。**默认值：** `true`（地址不重新排序）。默认值可使用 [`dns.setDefaultResultOrder()`][] 或 [`--dns-result-order`][] 配置。
* `callback` {Function}
  * `err` {Error}
  * `address` {string} IPv4 或 IPv6 地址的字符串表示。
  * `family` {integer} `4` 或 `6`，表示 `address` 的族，如果地址不是 IPv4 或 IPv6 地址，则为 `0`。`0` 可能是操作系统使用的名称解析服务存在错误的指示。

将主机名（例如 `'nodejs.org'`）解析为找到的第一个 A (IPv4) 或 AAAA (IPv6) 记录。所有 `option` 属性都是可选的。如果 `options` 是整数，则必须是 `4` 或 `6` – 如果未提供 `options`，则如果找到，将返回 IPv4 或 IPv6 地址，或两者都返回。

当 `all` 选项设置为 `true` 时，`callback` 的参数变为 `(err, addresses)`，其中 `addresses` 是一个具有 `address` 和 `family` 属性的对象数组。

出错时，`err` 是一个 [`Error`][] 对象，其中 `err.code` 是错误代码。请记住，不仅当主机名不存在时 `err.code` 会被设置为 `'ENOTFOUND'`，而且当查找以其他方式失败时（例如没有可用的文件描述符）也会如此。

`dns.lookup()` 不一定与 DNS 协议有关。该实现使用操作系统提供的设施，可以将名称与地址关联，反之亦然。这种实现可能对任何 Node.js 程序的行为产生微妙但重要的影响。在使用 `dns.lookup()` 之前，请花些时间查阅[实现考虑部分][Implementation considerations section]。

用法示例：

```mjs
import dns from 'node:dns';
const options = {
  family: 6,
  hints: dns.ADDRCONFIG | dns.V4MAPPED,
};
dns.lookup('example.org', options, (err, address, family) =>
  console.log('address: %j family: IPv%s', address, family));
// address: "2606:2800:21f:cb07:6820:80da:af6b:8b2c" family: IPv6

// 当 options.all 为 true 时，结果将是一个数组。
options.all = true;
dns.lookup('example.org', options, (err, addresses) =>
  console.log('addresses: %j', addresses));
// addresses: [{"address":"2606:2800:21f:cb07:6820:80da:af6b:8b2c","family":6}]
```

```cjs
const dns = require('node:dns');
const options = {
  family: 6,
  hints: dns.ADDRCONFIG | dns.V4MAPPED,
};
dns.lookup('example.org', options, (err, address, family) =>
  console.log('address: %j family: IPv%s', address, family));
// address: "2606:2800:21f:cb07:6820:80da:af6b:8b2c" family: IPv6

// 当 options.all 为 true 时，结果将是一个数组。
options.all = true;
dns.lookup('example.org', options, (err, addresses) =>
  console.log('addresses: %j', addresses));
// addresses: [{"address":"2606:2800:21f:cb07:6820:80da:af6b:8b2c","family":6}]
```

如果此方法以其 [`util.promisify()`][] 版本调用，且 `all` 未设置为 `true`，则返回一个包含 `address` 和 `family` 属性的 `Object` 的 `Promise`。

### 支持的 getaddrinfo 标志

<!-- YAML
changes:
  - version:
     - v13.13.0
     - v12.17.0
    pr-url: https://github.com/nodejs/node/pull/32183
    description: Added support for the `dns.ALL` flag.
-->

以下标志可以作为提示传递给 [`dns.lookup()`][]。

* `dns.ADDRCONFIG`：将返回的地址类型限制为系统上配置的非回环地址类型。例如，仅当当前系统至少配置了一个 IPv4 地址时才返回 IPv4 地址。
* `dns.V4MAPPED`：如果指定了 IPv6 族，但未找到 IPv6 地址，则返回 IPv4 映射的 IPv6 地址。在某些操作系统（例如 FreeBSD 10.1）上不支持。
* `dns.ALL`：如果指定了 `dns.V4MAPPED`，则返回已解析的 IPv6 地址以及 IPv4 映射的 IPv6 地址。

## `dns.lookupService(address, port, callback)`

<!-- YAML
added: v0.11.14
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `address` {string}
* `port` {number}
* `callback` {Function}
  * `err` {Error}
  * `hostname` {string} 例如 `example.com`
  * `service` {string} 例如 `http`

使用操作系统底层的 `getnameinfo` 实现将给定的 `address` 和 `port` 解析为主机名和服务。

如果 `address` 不是有效的 IP 地址，将抛出 `TypeError`。`port` 将被强制转换为数字。如果它不是合法端口，将抛出 `TypeError`。

出错时，`err` 是一个 [`Error`][] 对象，其中 `err.code` 是错误代码。

```mjs
import dns from 'node:dns';
dns.lookupService('127.0.0.1', 22, (err, hostname, service) => {
  console.log(hostname, service);
  // 打印: localhost ssh
});
```

```cjs
const dns = require('node:dns');
dns.lookupService('127.0.0.1', 22, (err, hostname, service) => {
  console.log(hostname, service);
  // 打印: localhost ssh
});
```

如果此方法以其 [`util.promisify()`][] 版本调用，则返回一个包含 `hostname` 和 `service` 属性的 `Object` 的 `Promise`。

## `dns.resolve(hostname[, rrtype], callback)`

<!-- YAML
added: v0.1.27
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string} 要解析的主机名。
* `rrtype` {string} 资源记录类型。**默认值：** `'A'`。
* `callback` {Function}
  * `err` {Error}
  * `records` {string\[] | Object\[] | Object}

使用 DNS 协议将主机名（例如 `'nodejs.org'`）解析为资源记录数组。`callback` 函数具有参数 `(err, records)`。成功时，`records` 将是资源记录数组。单个结果的类型和结构因 `rrtype` 而异：

| `rrtype`  | `records` 包含          | 结果类型 | 简写方法                 |
| --------- | ----------------------- | -------- | ------------------------ |
| `'A'`     | IPv4 地址（默认）       | {string} | [`dns.resolve4()`][]     |
| `'AAAA'`  | IPv6 地址               | {string} | [`dns.resolve6()`][]     |
| `'ANY'`   | 任何记录                | {Object} | [`dns.resolveAny()`][]   |
| `'CAA'`   | CA 授权记录             | {Object} | [`dns.resolveCaa()`][]   |
| `'CNAME'` | 规范名称记录            | {string} | [`dns.resolveCname()`][] |
| `'MX'`    | 邮件交换记录            | {Object} | [`dns.resolveMx()`][]    |
| `'NAPTR'` | 名称权威指针记录        | {Object} | [`dns.resolveNaptr()`][] |
| `'NS'`    | 名称服务器记录          | {string} | [`dns.resolveNs()`][]    |
| `'PTR'`   | 指针记录                | {string} | [`dns.resolvePtr()`][]   |
| `'SOA'`   | 权威起始记录            | {Object} | [`dns.resolveSoa()`][]   |
| `'SRV'`   | 服务记录                | {Object} | [`dns.resolveSrv()`][]   |
| `'TLSA'`  | 证书关联记录            | {Object} | [`dns.resolveTlsa()`][]  |
| `'TXT'`   | 文本记录                | {string\[]} | [`dns.resolveTxt()`][]   |

出错时，`err` 是一个 [`Error`][] 对象，其中 `err.code` 是[DNS 错误代码][DNS error codes]之一。

## `dns.resolve4(hostname[, options], callback)`

<!-- YAML
added: v0.1.16
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v7.2.0
    pr-url: https://github.com/nodejs/node/pull/9296
    description: This method now supports passing `options`,
                 specifically `options.ttl`.
-->

* `hostname` {string} 要解析的主机名。
* `options` {Object}
  * `ttl` {boolean} 检索每条记录的生存时间值 (TTL)。当为 `true` 时，回调接收一个 `{ address: '1.2.3.4', ttl: 60 }` 对象数组，而不是字符串数组，TTL 以秒表示。
* `callback` {Function}
  * `err` {Error}
  * `addresses` {string\[] | Object\[]}

使用 DNS 协议解析 `hostname` 的 IPv4 地址（`A` 记录）。传递给 `callback` 函数的 `addresses` 参数将包含一个 IPv4 地址数组（例如 `['74.125.79.104', '74.125.79.105', '74.125.79.106']`）。

## `dns.resolve6(hostname[, options], callback)`

<!-- YAML
added: v0.1.16
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
  - version: v7.2.0
    pr-url: https://github.com/nodejs/node/pull/9296
    description: This method now supports passing `options`,
                 specifically `options.ttl`.
-->

* `hostname` {string} 要解析的主机名。
* `options` {Object}
  * `ttl` {boolean} 检索每条记录的生存时间值 (TTL)。当为 `true` 时，回调接收一个 `{ address: '0:1:2:3:4:5:6:7', ttl: 60 }` 对象数组，而不是字符串数组，TTL 以秒表示。
* `callback` {Function}
  * `err` {Error}
  * `addresses` {string\[] | Object\[]}

使用 DNS 协议解析 `hostname` 的 IPv6 地址（`AAAA` 记录）。传递给 `callback` 函数的 `addresses` 参数将包含一个 IPv6 地址数组。

## `dns.resolveAny(hostname, callback)`

<!-- YAML
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `ret` {Object\[]}

使用 DNS 协议解析所有记录（也称为 `ANY` 或 `*` 查询）。传递给 `callback` 函数的 `ret` 参数将是一个包含各种类型记录的数组。每个对象都有一个属性 `type`，指示当前记录的类型。根据 `type`，对象上将存在其他属性：

| 类型      | 属性                                                                                                                                    |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `'A'`     | `address`/`ttl`                                                                                                                         |
| `'AAAA'`  | `address`/`ttl`                                                                                                                         |
| `'CNAME'` | `value`                                                                                                                                 |
| `'MX'`    | 参考 [`dns.resolveMx()`][]                                                                                                              |
| `'NAPTR'` | 参考 [`dns.resolveNaptr()`][]                                                                                                           |
| `'NS'`    | `value`                                                                                                                                 |
| `'PTR'`   | `value`                                                                                                                                 |
| `'SOA'`   | 参考 [`dns.resolveSoa()`][]                                                                                                             |
| `'SRV'`   | 参考 [`dns.resolveSrv()`][]                                                                                                             |
| `'TLSA'`  | 参考 [`dns.resolveTlsa()`][]                                                                                                            |
| `'TXT'`   | 此类型的记录包含一个名为 `entries` 的数组属性，参考 [`dns.resolveTxt()`][]，例如 `{ entries: ['...'], type: 'TXT' }` |

以下是传递给回调的 `ret` 对象示例：

<!-- eslint-disable @stylistic/js/semi -->

```js
[ { type: 'A', address: '127.0.0.1', ttl: 299 },
  { type: 'CNAME', value: 'example.com' },
  { type: 'MX', exchange: 'alt4.aspmx.l.example.com', priority: 50 },
  { type: 'NS', value: 'ns1.example.com' },
  { type: 'TXT', entries: [ 'v=spf1 include:_spf.example.com ~all' ] },
  { type: 'SOA',
    nsname: 'ns1.example.com',
    hostmaster: 'admin.example.com',
    serial: 156696742,
    refresh: 900,
    retry: 900,
    expire: 1800,
    minttl: 60 } ]
```

DNS 服务器运营商可能选择不响应 `ANY` 查询。最好调用单独的方法，如 [`dns.resolve4()`][]、[`dns.resolveMx()`][] 等。更多细节，请参阅 [RFC 8482][]。

## `dns.resolveCname(hostname, callback)`

<!-- YAML
added: v0.3.2
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `addresses` {string\[]}

使用 DNS 协议解析 `hostname` 的 `CNAME` 记录。传递给 `callback` 函数的 `addresses` 参数将包含 `hostname` 可用的规范名称记录数组（例如 `['bar.example.com']`）。

## `dns.resolveCaa(hostname, callback)`

<!-- YAML
added:
  - v15.0.0
  - v14.17.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `records` {Object\[]}

使用 DNS 协议解析 `hostname` 的 `CAA` 记录。传递给 `callback` 函数的 `addresses` 参数将包含 `hostname` 可用的证书颁发机构授权记录数组（例如 `[{critical: 0, iodef: 'mailto:pki@example.com'}, {critical: 128, issue: 'pki.example.com'}]`）。

## `dns.resolveMx(hostname, callback)`

<!-- YAML
added: v0.1.27
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `addresses` {Object\[]}

使用 DNS 协议解析 `hostname` 的邮件交换记录（`MX` 记录）。传递给 `callback` 函数的 `addresses` 参数将包含一个包含 `priority` 和 `exchange` 属性的对象数组（例如 `[{priority: 10, exchange: 'mx.example.com'}, ...]`）。

## `dns.resolveNaptr(hostname, callback)`

<!-- YAML
added: v0.9.12
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `addresses` {Object\[]}

使用 DNS 协议解析 `hostname` 的基于正则表达式的记录（`NAPTR` 记录）。传递给 `callback` 函数的 `addresses` 参数将包含一个具有以下属性的对象数组：

* `flags`
* `service`
* `regexp`
* `replacement`
* `order`
* `preference`

<!-- eslint-skip -->

```js
{
  flags: 's',
  service: 'SIP+D2U',
  regexp: '',
  replacement: '_sip._udp.example.com',
  order: 30,
  preference: 100
}
```

## `dns.resolveNs(hostname, callback)`

<!-- YAML
added: v0.1.90
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `addresses` {string\[]}

使用 DNS 协议解析 `hostname` 的名称服务器记录（`NS` 记录）。传递给 `callback` 函数的 `addresses` 参数将包含 `hostname` 可用的名称服务器记录数组（例如 `['ns1.example.com', 'ns2.example.com']`）。

## `dns.resolvePtr(hostname, callback)`

<!-- YAML
added: v6.0.0
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `addresses` {string\[]}

使用 DNS 协议解析 `hostname` 的指针记录（`PTR` 记录）。传递给 `callback` 函数的 `addresses` 参数将是一个包含回复记录的字符串数组。

## `dns.resolveSoa(hostname, callback)`

<!-- YAML
added: v0.11.10
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `address` {Object}

使用 DNS 协议解析 `hostname` 的权威起始记录（`SOA` 记录）。传递给 `callback` 函数的 `address` 参数将是一个具有以下属性的对象：

* `nsname`
* `hostmaster`
* `serial`
* `refresh`
* `retry`
* `expire`
* `minttl`

<!-- eslint-skip -->

```js
{
  nsname: 'ns.example.com',
  hostmaster: 'root.example.com',
  serial: 2013101809,
  refresh: 10000,
  retry: 2400,
  expire: 604800,
  minttl: 3600
}
```

## `dns.resolveSrv(hostname, callback)`

<!-- YAML
added: v0.1.27
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `addresses` {Object\[]}

使用 DNS 协议解析 `hostname` 的服务记录（`SRV` 记录）。传递给 `callback` 函数的 `addresses` 参数将是一个具有以下属性的对象数组：

* `priority`
* `weight`
* `port`
* `name`

<!-- eslint-skip -->

```js
{
  priority: 10,
  weight: 5,
  port: 21223,
  name: 'service.example.com'
}
```

## `dns.resolveTlsa(hostname, callback)`

<!-- YAML
added:
  - v23.9.0
  - v22.15.0
-->

<!--lint disable no-undefined-references list-item-bullet-indent-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `records` {Object\[]}

<!--lint enable no-undefined-references list-item-bullet-indent-->

使用 DNS 协议解析 `hostname` 的证书关联记录（`TLSA` 记录）。传递给 `callback` 函数的 `records` 参数是一个具有以下属性的对象数组：

* `certUsage`
* `selector`
* `match`
* `data`

<!-- eslint-skip -->

```js
{
  certUsage: 3,
  selector: 1,
  match: 1,
  data: [ArrayBuffer]
}
```

## `dns.resolveTxt(hostname, callback)`

<!-- YAML
added: v0.1.27
changes:
  - version: v18.0.0
    pr-url: https://github.com/nodejs/node/pull/41678
    description: Passing an invalid callback to the `callback` argument
                 now throws `ERR_INVALID_ARG_TYPE` instead of
                 `ERR_INVALID_CALLBACK`.
-->

* `hostname` {string}
* `callback` {Function}
  * `err` {Error}
  * `records` {string\[]}

使用 DNS 协议解析 `hostname` 的文本查询（`TXT` 记录）。传递给 `callback` 函数的 `records` 参数是一个二维数组，包含 `hostname` 可用的文本记录（例如 `[ ['v=spf1 ip4:0.0.0.0 ', '~all' ] ]`）。每个子数组包含一条记录的 TXT 块。根据使用情况，这些可以连接在一起或单独处理。

## `dns.reverse(ip, callback)`

<!-- YAML
added: v0.1.16
-->

* `ip` {string}
* `callback` {Function}
  * `err` {Error}
  * `hostnames` {string\[]}

执行反向 DNS 查询，将 IPv4 或 IPv6 地址解析为主机名数组。

出错时，`err` 是一个 [`Error`][] 对象，其中 `err.code` 是[DNS 错误代码][DNS error codes]之一。

## `dns.setDefaultResultOrder(order)`

<!-- YAML
added:
  - v16.4.0
  - v14.18.0
changes:
  - version:
    - v22.1.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52492
    description: The `ipv6first` value is supported now.
  - version: v17.0.0
    pr-url: https://github.com/nodejs/node/pull/39987
    description: Changed default value to `verbatim`.
-->

* `order` {string} 必须是 `'ipv4first'`、`'ipv6first'` 或 `'verbatim'`。

设置 [`dns.lookup()`][] 和 [`dnsPromises.lookup()`][] 中 `order` 的默认值。值可以是：

* `ipv4first`：将默认 `order` 设置为 `ipv4first`。
* `ipv6first`：将默认 `order` 设置为 `ipv6first`。
* `verbatim`：将默认 `order` 设置为 `verbatim`。

默认值为 `verbatim`，且 [`dns.setDefaultResultOrder()`][] 的优先级高于 [`--dns-result-order`][]。当使用[工作线程][worker threads]时，主线程中的 [`dns.setDefaultResultOrder()`][] 不会影响工作线程中的默认 DNS 顺序。

## `dns.getDefaultResultOrder()`

<!-- YAML
added:
  - v20.1.0
  - v18.17.0
changes:
  - version:
    - v22.1.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52492
    description: The `ipv6first` value is supported now.
-->

获取 [`dns.lookup()`][] 和 [`dnsPromises.lookup()`][] 中 `order` 的默认值。值可以是：

* `ipv4first`：表示 `order` 默认为 `ipv4first`。
* `ipv6first`：表示 `order` 默认为 `ipv6first`。
* `verbatim`：表示 `order` 默认为 `verbatim`。

## `dns.setServers(servers)`

<!-- YAML
added: v0.11.3
-->

* `servers` {string\[]} [RFC 5952][] 格式化地址的数组

设置执行 DNS 解析时要使用的服务器的 IP 地址和端口。`servers` 参数是 [RFC 5952][] 格式化地址的数组。如果端口是 IANA 默认 DNS 端口 (53)，则可以省略。

```js
dns.setServers([
  '8.8.8.8',
  '[2001:4860:4860::8888]',
  '8.8.8.8:1053',
  '[2001:4860:4860::8888]:1053',
]);
```

如果提供了无效地址，将抛出错误。

`dns.setServers()` 方法不得在 DNS 查询进行时调用。

[`dns.setServers()`][] 方法仅影响 [`dns.resolve()`][]、`dns.resolve*()` 和 [`dns.reverse()`][]（特别是不影响 [`dns.lookup()`][]）。

此方法的工作方式非常类似于 [resolve.conf](https://man7.org/linux/man-pages/man5/resolv.conf.5.html)。也就是说，如果尝试使用提供的第一个服务器解析导致 `NOTFOUND` 错误，则 `resolve()` 方法将不会尝试使用后续提供的服务器进行解析。仅当较早的服务器超时或导致其他错误时，才会使用备用 DNS 服务器。

## DNS promises API

<!-- YAML
added: v10.6.0
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/32953
    description: Exposed as `require('dns/promises')`.
  - version:
    - v11.14.0
    - v10.17.0
    pr-url: https://github.com/nodejs/node/pull/26592
    description: This API is no longer experimental.
-->

`dns.promises` API 提供了一组替代的异步 DNS 方法，这些方法返回 `Promise` 对象而不是使用回调。该 API 可通过 `require('node:dns').promises` 或 `require('node:dns/promises')` 访问。

### 类：`dnsPromises.Resolver`

<!-- YAML
added: v10.6.0
-->

一个独立的 DNS 请求解析器。

创建新的解析器会使用默认的服务器设置。使用 [`resolver.setServers()`][`dnsPromises.setServers()`] 设置解析器使用的服务器不会影响其他解析器：

```mjs
import { Resolver } from 'node:dns/promises';
const resolver = new Resolver();
resolver.setServers(['4.4.4.4']);

// 此请求将使用 4.4.4.4 的服务器，独立于全局设置。
const addresses = await resolver.resolve4('example.org');
```

```cjs
const { Resolver } = require('node:dns').promises;
const resolver = new Resolver();
resolver.setServers(['4.4.4.4']);

// 此请求将使用 4.4.4.4 的服务器，独立于全局设置。
resolver.resolve4('example.org').then((addresses) => {
  // ...
});

// 或者，可以使用 async-await 风格编写相同的代码。
(async function() {
  const addresses = await resolver.resolve4('example.org');
})();
```

`dnsPromises` API 中的以下方法可用：

* [`resolver.getServers()`][`dnsPromises.getServers()`]
* [`resolver.resolve()`][`dnsPromises.resolve()`]
* [`resolver.resolve4()`][`dnsPromises.resolve4()`]
* [`resolver.resolve6()`][`dnsPromises.resolve6()`]
* [`resolver.resolveAny()`][`dnsPromises.resolveAny()`]
* [`resolver.resolveCaa()`][`dnsPromises.resolveCaa()`]
* [`resolver.resolveCname()`][`dnsPromises.resolveCname()`]
* [`resolver.resolveMx()`][`dnsPromises.resolveMx()`]
* [`resolver.resolveNaptr()`][`dnsPromises.resolveNaptr()`]
* [`resolver.resolveNs()`][`dnsPromises.resolveNs()`]
* [`resolver.resolvePtr()`][`dnsPromises.resolvePtr()`]
* [`resolver.resolveSoa()`][`dnsPromises.resolveSoa()`]
* [`resolver.resolveSrv()`][`dnsPromises.resolveSrv()`]
* [`resolver.resolveTlsa()`][`dnsPromises.resolveTlsa()`]
* [`resolver.resolveTxt()`][`dnsPromises.resolveTxt()`]
* [`resolver.reverse()`][`dnsPromises.reverse()`]
* [`resolver.setServers()`][`dnsPromises.setServers()`]

### `resolver.cancel()`

<!-- YAML
added:
  - v15.3.0
  - v14.17.0
-->

取消此解析器进行的所有未完成的 DNS 查询。相应的 promise 将被拒绝，错误码为 `ECANCELLED`。

### `dnsPromises.getServers()`

<!-- YAML
added: v10.6.0
-->

* 返回：{string\[]}

返回一个 IP 地址字符串数组，根据 [RFC 5952][] 格式化，这些地址当前配置用于 DNS 解析。如果使用了自定义端口，字符串将包含端口部分。

<!-- eslint-disable @stylistic/js/semi-->

```js
[
  '8.8.8.8',
  '2001:4860:4860::8888',
  '8.8.8.8:1053',
  '[2001:4860:4860::8888]:1053',
]
```

### `dnsPromises.lookup(hostname[, options])`

<!-- YAML
added: v10.6.0
changes:
  - version:
    - v22.1.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52492
    description: The `verbatim` option is now deprecated in favor of the new `order` option.
-->

* `hostname` {string}
* `options` {integer | Object}
  * `family` {integer} 记录族。必须为 `4`、`6` 或 `0`。值 `0` 表示返回 IPv4 或 IPv6 地址。如果值 `0` 与 `{ all: true }` 一起使用（见下文），则返回 IPv4 和 IPv6 地址中的一个或两个，具体取决于系统的 DNS 解析器。**默认值：** `0`。
  * `hints` {number} 一个或多个[支持的 `getaddrinfo` 标志][supported `getaddrinfo` flags]。可以通过按位 `OR` 运算它们的值来传递多个标志。
  * `all` {boolean} 当为 `true` 时，`Promise` 以所有地址的数组解析。否则，返回单个地址。**默认值：** `false`。
  * `order` {string} 当为 `verbatim` 时，`Promise` 以 DNS 解析器返回的 IPv4 和 IPv6 地址顺序解析。当为 `ipv4first` 时，IPv4 地址放在 IPv6 地址之前。当为 `ipv6first` 时，IPv6 地址放在 IPv4 地址之前。**默认值：** `verbatim`（地址不重新排序）。默认值可使用 [`dns.setDefaultResultOrder()`][] 或 [`--dns-result-order`][] 配置。新代码应使用 `{ order: 'verbatim' }`。
  * `verbatim` {boolean} 当为 `true` 时，`Promise` 以 DNS 解析器返回的 IPv4 和 IPv6 地址顺序解析。当为 `false` 时，IPv4 地址放在 IPv6 地址之前。此选项将被弃用，转而使用 `order`。当两者都指定时，`order` 具有更高的优先级。新代码应仅使用 `order`。**默认值：** 当前为 `false`（地址重新排序），但预计在不久的将来会更改。默认值可使用 [`dns.setDefaultResultOrder()`][] 或 [`--dns-result-order`][] 配置。

将主机名（例如 `'nodejs.org'`）解析为找到的第一个 A (IPv4) 或 AAAA (IPv6) 记录。所有 `option` 属性都是可选的。如果 `options` 是整数，则必须是 `4` 或 `6` – 如果未提供 `options`，则如果找到，将返回 IPv4 或 IPv6 地址，或两者都返回。

当 `all` 选项设置为 `true` 时，`Promise` 以 `addresses` 解析，`addresses` 是一个具有 `address` 和 `family` 属性的对象数组。

出错时，`Promise` 被拒绝，并带有一个 [`Error`][] 对象，其中 `err.code` 是错误代码。请记住，不仅当主机名不存在时 `err.code` 会被设置为 `'ENOTFOUND'`，而且当查找以其他方式失败时（例如没有可用的文件描述符）也会如此。

[`dnsPromises.lookup()`][] 不一定与 DNS 协议有关。该实现使用操作系统提供的设施，可以将名称与地址关联，反之亦然。这种实现可能对任何 Node.js 程序的行为产生微妙但重要的影响。在使用 `dnsPromises.lookup()` 之前，请花些时间查阅[实现考虑部分][Implementation considerations section]。

用法示例：

```mjs
import dns from 'node:dns';
const dnsPromises = dns.promises;
const options = {
  family: 6,
  hints: dns.ADDRCONFIG | dns.V4MAPPED,
};

await dnsPromises.lookup('example.org', options).then((result) => {
  console.log('address: %j family: IPv%s', result.address, result.family);
  // address: "2606:2800:21f:cb07:6820:80da:af6b:8b2c" family: IPv6
});

// 当 options.all 为 true 时，结果将是一个数组。
options.all = true;
await dnsPromises.lookup('example.org', options).then((result) => {
  console.log('addresses: %j', result);
  // addresses: [{"address":"2606:2800:21f:cb07:6820:80da:af6b:8b2c","family":6}]
});
```

```cjs
const dns = require('node:dns');
const dnsPromises = dns.promises;
const options = {
  family: 6,
  hints: dns.ADDRCONFIG | dns.V4MAPPED,
};

dnsPromises.lookup('example.org', options).then((result) => {
  console.log('address: %j family: IPv%s', result.address, result.family);
  // address: "2606:2800:21f:cb07:6820:80da:af6b:8b2c" family: IPv6
});

// 当 options.all 为 true 时，结果将是一个数组。
options.all = true;
dnsPromises.lookup('example.org', options).then((result) => {
  console.log('addresses: %j', result);
  // addresses: [{"address":"2606:2800:21f:cb07:6820:80da:af6b:8b2c","family":6}]
});
```

### `dnsPromises.lookupService(address, port)`

<!-- YAML
added: v10.6.0
-->

* `address` {string}
* `port` {number}

使用操作系统底层的 `getnameinfo` 实现将给定的 `address` 和 `port` 解析为主机名和服务。

如果 `address` 不是有效的 IP 地址，将抛出 `TypeError`。`port` 将被强制转换为数字。如果它不是合法端口，将抛出 `TypeError`。

出错时，`Promise` 被拒绝，并带有一个 [`Error`][] 对象，其中 `err.code` 是错误代码。

```mjs
import dnsPromises from 'node:dns/promises';
const result = await dnsPromises.lookupService('127.0.0.1', 22);

console.log(result.hostname, result.service); // 打印: localhost ssh
```

```cjs
const dnsPromises = require('node:dns').promises;
dnsPromises.lookupService('127.0.0.1', 22).then((result) => {
  console.log(result.hostname, result.service);
  // 打印: localhost ssh
});
```

### `dnsPromises.resolve(hostname[, rrtype])`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string} 要解析的主机名。
* `rrtype` {string} 资源记录类型。**默认值：** `'A'`。

使用 DNS 协议将主机名（例如 `'nodejs.org'`）解析为资源记录数组。成功时，`Promise` 以资源记录数组解析。单个结果的类型和结构因 `rrtype` 而异：

| `rrtype`  | `records` 包含          | 结果类型 | 简写方法                     |
| --------- | ----------------------- | -------- | ---------------------------- |
| `'A'`     | IPv4 地址（默认）       | {string} | [`dnsPromises.resolve4()`][] |
| `'AAAA'`  | IPv6 地址               | {string} | [`dnsPromises.resolve6()`][] |
| `'ANY'`   | 任何记录                | {Object} | [`dnsPromises.resolveAny()`][] |
| `'CAA'`   | CA 授权记录             | {Object} | [`dnsPromises.resolveCaa()`][] |
| `'CNAME'` | 规范名称记录            | {string} | [`dnsPromises.resolveCname()`][] |
| `'MX'`    | 邮件交换记录            | {Object} | [`dnsPromises.resolveMx()`][] |
| `'NAPTR'` | 名称权威指针记录        | {Object} | [`dnsPromises.resolveNaptr()`][] |
| `'NS'`    | 名称服务器记录          | {string} | [`dnsPromises.resolveNs()`][] |
| `'PTR'`   | 指针记录                | {string} | [`dnsPromises.resolvePtr()`][] |
| `'SOA'`   | 权威起始记录            | {Object} | [`dnsPromises.resolveSoa()`][] |
| `'SRV'`   | 服务记录                | {Object} | [`dnsPromises.resolveSrv()`][] |
| `'TLSA'`  | 证书关联记录            | {Object} | [`dnsPromises.resolveTlsa()`][] |
| `'TXT'`   | 文本记录                | {string\[]} | [`dnsPromises.resolveTxt()`][] |

出错时，`Promise` 被拒绝，并带有一个 [`Error`][] 对象，其中 `err.code` 是[DNS 错误代码][DNS error codes]之一。

### `dnsPromises.resolve4(hostname[, options])`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string} 要解析的主机名。
* `options` {Object}
  * `ttl` {boolean} 检索每条记录的生存时间值 (TTL)。当为 `true` 时，`Promise` 以 `{ address: '1.2.3.4', ttl: 60 }` 对象数组解析，而不是字符串数组，TTL 以秒表示。

使用 DNS 协议解析 `hostname` 的 IPv4 地址（`A` 记录）。成功时，`Promise` 以 IPv4 地址数组解析（例如 `['74.125.79.104', '74.125.79.105', '74.125.79.106']`）。

### `dnsPromises.resolve6(hostname[, options])`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string} 要解析的主机名。
* `options` {Object}
  * `ttl` {boolean} 检索每条记录的生存时间值 (TTL)。当为 `true` 时，`Promise` 以 `{ address: '0:1:2:3:4:5:6:7', ttl: 60 }` 对象数组解析，而不是字符串数组，TTL 以秒表示。

使用 DNS 协议解析 `hostname` 的 IPv6 地址（`AAAA` 记录）。成功时，`Promise` 以 IPv6 地址数组解析。

### `dnsPromises.resolveAny(hostname)`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string}

使用 DNS 协议解析所有记录（也称为 `ANY` 或 `*` 查询）。成功时，`Promise` 以包含各种类型记录的数组解析。每个对象有一个属性 `type`，指示当前记录的类型。根据 `type`，对象上将存在其他属性：

| 类型      | 属性                                                                                                                                    |
| --------- | --------------------------------------------------------------------------------------------------------------------------------------- |
| `'A'`     | `address`/`ttl`                                                                                                                         |
| `'AAAA'`  | `address`/`ttl`                                                                                                                         |
| `'CNAME'` | `value`                                                                                                                                 |
| `'MX'`    | 参考 [`dnsPromises.resolveMx()`][]                                                                                                      |
| `'NAPTR'` | 参考 [`dnsPromises.resolveNaptr()`][]                                                                                                   |
| `'NS'`    | `value`                                                                                                                                 |
| `'PTR'`   | `value`                                                                                                                                 |
| `'SOA'`   | 参考 [`dnsPromises.resolveSoa()`][]                                                                                                     |
| `'SRV'`   | 参考 [`dnsPromises.resolveSrv()`][]                                                                                                     |
| `'TLSA'`  | 参考 [`dnsPromises.resolveTlsa()`][]                                                                                                    |
| `'TXT'`   | 此类型的记录包含一个名为 `entries` 的数组属性，参考 [`dnsPromises.resolveTxt()`][]，例如 `{ entries: ['...'], type: 'TXT' }` |

以下是结果对象的示例：

<!-- eslint-disable @stylistic/js/semi -->

```js
[ { type: 'A', address: '127.0.0.1', ttl: 299 },
  { type: 'CNAME', value: 'example.com' },
  { type: 'MX', exchange: 'alt4.aspmx.l.example.com', priority: 50 },
  { type: 'NS', value: 'ns1.example.com' },
  { type: 'TXT', entries: [ 'v=spf1 include:_spf.example.com ~all' ] },
  { type: 'SOA',
    nsname: 'ns1.example.com',
    hostmaster: 'admin.example.com',
    serial: 156696742,
    refresh: 900,
    retry: 900,
    expire: 1800,
    minttl: 60 } ]
```

### `dnsPromises.resolveCaa(hostname)`

<!-- YAML
added:
  - v15.0.0
  - v14.17.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的 `CAA` 记录。成功时，`Promise` 以包含 `hostname` 可用的证书颁发机构授权记录的对象数组解析（例如 `[{critical: 0, iodef: 'mailto:pki@example.com'},{critical: 128, issue: 'pki.example.com'}]`）。

### `dnsPromises.resolveCname(hostname)`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的 `CNAME` 记录。成功时，`Promise` 以 `hostname` 可用的规范名称记录数组解析（例如 `['bar.example.com']`）。

### `dnsPromises.resolveMx(hostname)`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的邮件交换记录（`MX` 记录）。成功时，`Promise` 以包含 `priority` 和 `exchange` 属性的对象数组解析（例如 `[{priority: 10, exchange: 'mx.example.com'}, ...]`）。

### `dnsPromises.resolveNaptr(hostname)`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的基于正则表达式的记录（`NAPTR` 记录）。成功时，`Promise` 以具有以下属性的对象数组解析：

* `flags`
* `service`
* `regexp`
* `replacement`
* `order`
* `preference`

<!-- eslint-skip -->

```js
{
  flags: 's',
  service: 'SIP+D2U',
  regexp: '',
  replacement: '_sip._udp.example.com',
  order: 30,
  preference: 100
}
```

### `dnsPromises.resolveNs(hostname)`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的名称服务器记录（`NS` 记录）。成功时，`Promise` 以 `hostname` 可用的名称服务器记录数组解析（例如 `['ns1.example.com', 'ns2.example.com']`）。

### `dnsPromises.resolvePtr(hostname)`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的指针记录（`PTR` 记录）。成功时，`Promise` 以包含回复记录的字符串数组解析。

### `dnsPromises.resolveSoa(hostname)`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的权威起始记录（`SOA` 记录）。成功时，`Promise` 以具有以下属性的对象解析：

* `nsname`
* `hostmaster`
* `serial`
* `refresh`
* `retry`
* `expire`
* `minttl`

<!-- eslint-skip -->

```js
{
  nsname: 'ns.example.com',
  hostmaster: 'root.example.com',
  serial: 2013101809,
  refresh: 10000,
  retry: 2400,
  expire: 604800,
  minttl: 3600
}
```

### `dnsPromises.resolveSrv(hostname)`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的服务记录（`SRV` 记录）。成功时，`Promise` 以具有以下属性的对象数组解析：

* `priority`
* `weight`
* `port`
* `name`

<!-- eslint-skip -->

```js
{
  priority: 10,
  weight: 5,
  port: 21223,
  name: 'service.example.com'
}
```

### `dnsPromises.resolveTlsa(hostname)`

<!-- YAML
added:
  - v23.9.0
  - v22.15.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的证书关联记录（`TLSA` 记录）。成功时，`Promise` 以具有以下属性的对象数组解析：

* `certUsage`
* `selector`
* `match`
* `data`

<!-- eslint-skip -->

```js
{
  certUsage: 3,
  selector: 1,
  match: 1,
  data: [ArrayBuffer]
}
```

### `dnsPromises.resolveTxt(hostname)`

<!-- YAML
added: v10.6.0
-->

* `hostname` {string}

使用 DNS 协议解析 `hostname` 的文本查询（`TXT` 记录）。成功时，`Promise` 以 `hostname` 可用的文本记录的二维数组解析（例如 `[ ['v=spf1 ip4:0.0.0.0 ', '~all' ] ]`）。每个子数组包含一条记录的 TXT 块。根据使用情况，这些可以连接在一起或单独处理。

### `dnsPromises.reverse(ip)`

<!-- YAML
added: v10.6.0
-->

* `ip` {string}

执行反向 DNS 查询，将 IPv4 或 IPv6 地址解析为主机名数组。

出错时，`Promise` 被拒绝，并带有一个 [`Error`][] 对象，其中 `err.code` 是[DNS 错误代码][DNS error codes]之一。

### `dnsPromises.setDefaultResultOrder(order)`

<!-- YAML
added:
  - v16.4.0
  - v14.18.0
changes:
  - version:
    - v22.1.0
    - v20.13.0
    pr-url: https://github.com/nodejs/node/pull/52492
    description: The `ipv6first` value is supported now.
  - version: v17.0.0
    pr-url: https://github.com/nodejs/node/pull/39987
    description: Changed default value to `verbatim`.
-->

* `order` {string} 必须是 `'ipv4first'`、`'ipv6first'` 或 `'verbatim'`。

设置 [`dns.lookup()`][] 和 [`dnsPromises.lookup()`][] 中 `order` 的默认值。值可以是：

* `ipv4first`：将默认 `order` 设置为 `ipv4first`。
* `ipv6first`：将默认 `order` 设置为 `ipv6first`。
* `verbatim`：将默认 `order` 设置为 `verbatim`。

默认值为 `verbatim`，且 [`dnsPromises.setDefaultResultOrder()`][] 的优先级高于 [`--dns-result-order`][]。当使用[工作线程][worker threads]时，主线程中的 [`dnsPromises.setDefaultResultOrder()`][] 不会影响工作线程中的默认 DNS 顺序。

### `dnsPromises.getDefaultResultOrder()`

<!-- YAML
added:
  - v20.1.0
  - v18.17.0
-->

获取 `dnsOrder` 的值。

### `dnsPromises.setServers(servers)`

<!-- YAML
added: v10.6.0
-->

* `servers` {string\[]} [RFC 5952][] 格式化地址的数组

设置执行 DNS 解析时要使用的服务器的 IP 地址和端口。`servers` 参数是 [RFC 5952][] 格式化地址的数组。如果端口是 IANA 默认 DNS 端口 (53)，则可以省略。

```js
dnsPromises.setServers([
  '8.8.8.8',
  '[2001:4860:4860::8888]',
  '8.8.8.8:1053',
  '[2001:4860:4860::8888]:1053',
]);
```

如果提供了无效地址，将抛出错误。

`dnsPromises.setServers()` 方法不得在 DNS 查询进行时调用。

此方法的工作方式非常类似于 [resolve.conf](https://man7.org/linux/man-pages/man5/resolv.conf.5.html)。也就是说，如果尝试使用提供的第一个服务器解析导致 `NOTFOUND` 错误，则 `resolve()` 方法将不会尝试使用后续提供的服务器进行解析。仅当较早的服务器超时或导致其他错误时，才会使用备用 DNS 服务器。

## 错误代码

每个 DNS 查询可以返回以下错误代码之一：

* `dns.NODATA`: DNS 服务器返回无数据的答案。
* `dns.FORMERR`: DNS 服务器声称查询格式错误。
* `dns.SERVFAIL`: DNS 服务器返回一般故障。
* `dns.NOTFOUND`: 未找到域名。
* `dns.NOTIMP`: DNS 服务器未实现请求的操作。
* `dns.REFUSED`: DNS 服务器拒绝查询。
* `dns.BADQUERY`: 格式错误的 DNS 查询。
* `dns.BADNAME`: 格式错误的主机名。
* `dns.BADFAMILY`: 不支持的地址族。
* `dns.BADRESP`: 格式错误的 DNS 回复。
* `dns.CONNREFUSED`: 无法联系 DNS 服务器。
* `dns.TIMEOUT`: 联系 DNS 服务器时超时。
* `dns.EOF`: 文件结束。
* `dns.FILE`: 读取文件时出错。
* `dns.NOMEM`: 内存不足。
* `dns.DESTRUCTION`: 通道正在被销毁。
* `dns.BADSTR`: 格式错误的字符串。
* `dns.BADFLAGS`: 指定了非法标志。
* `dns.NONAME`: 给定的主机名不是数字。
* `dns.BADHINTS`: 指定了非法提示标志。
* `dns.NOTINITIALIZED`: c-ares 库尚未初始化。
* `dns.LOADIPHLPAPI`: 加载 `iphlpapi.dll` 时出错。
* `dns.ADDRGETNETWORKPARAMS`: 找不到 `GetNetworkParams` 函数。
* `dns.CANCELLED`: DNS 查询已取消。

`dnsPromises` API 也导出上述错误代码，例如 `dnsPromises.NODATA`。

## 实现考虑

尽管 [`dns.lookup()`][] 和各种 `dns.resolve*()/dns.reverse()` 函数具有将网络名称与网络地址（或反之）关联的相同目标，但它们的行为截然不同。这些差异可能对 Node.js 程序的行为产生微妙但重要的影响。

### `dns.lookup()`

在底层，[`dns.lookup()`][] 使用与大多数其他程序相同的操作系统设施。例如，[`dns.lookup()`][] 几乎总是以与 `ping` 命令相同的方式解析给定名称。在大多数类 POSIX 操作系统上，[`dns.lookup()`][] 函数的行为可以通过更改 nsswitch.conf(5) 和/或 resolv.conf(5) 中的设置来修改，但更改这些文件将更改在同一操作系统上运行的所有其他程序的行为。

尽管从 JavaScript 的角度来看，对 `dns.lookup()` 的调用是异步的，但它是作为对 getaddrinfo(3) 的同步调用在 libuv 的线程池中实现的。这可能对某些应用程序产生令人惊讶的负面性能影响，更多信息请参阅 [`UV_THREADPOOL_SIZE`][] 文档。

各种网络 API 将在内部调用 `dns.lookup()` 来解析主机名。如果这是一个问题，考虑使用 `dns.resolve()` 将主机名解析为地址，并使用地址而不是主机名。此外，一些网络 API（如 [`socket.connect()`][] 和 [`dgram.createSocket()`][]）允许替换默认解析器 `dns.lookup()`。

### `dns.resolve()`、`dns.resolve*()` 和 `dns.reverse()`

这些函数的实现与 [`dns.lookup()`][] 完全不同。它们不使用 getaddrinfo(3)，并且它们总是在网络上执行 DNS 查询。此网络通信始终是异步完成的，并且不使用 libuv 的线程池。

因此，这些函数不会对 libuv 线程池上发生的其他处理产生与 [`dns.lookup()`][] 相同的负面影响。

它们不使用 [`dns.lookup()`][] 使用的同一组配置文件。例如，它们不使用 `/etc/hosts` 中的配置。

[DNS error codes]: #error-codes
[Domain Name System (DNS)]: https://en.wikipedia.org/wiki/Domain_Name_System
[Implementation considerations section]: #implementation-considerations
[RFC 5952]: https://tools.ietf.org/html/rfc5952#section-6
[RFC 8482]: https://tools.ietf.org/html/rfc8482
[`--dns-result-order`]: cli.md#--dns-result-orderorder
[`Error`]: errors.md#class-error
[`UV_THREADPOOL_SIZE`]: cli.md#uv_threadpool_sizesize
[`dgram.createSocket()`]: dgram.md#dgramcreatesocketoptions-callback
[`dns.getServers()`]: #dnsgetservers
[`dns.lookup()`]: #dnslookuphostname-options-callback
[`dns.resolve()`]: #dnsresolvehostname-rrtype-callback
[`dns.resolve4()`]: #dnsresolve4hostname-options-callback
[`dns.resolve6()`]: #dnsresolve6hostname-options-callback
[`dns.resolveAny()`]: #dnsresolveanyhostname-callback
[`dns.resolveCaa()`]: #dnsresolvecaahostname-callback
[`dns.resolveCname()`]: #dnsresolvecnamehostname-callback
[`dns.resolveMx()`]: #dnsresolvemxhostname-callback
[`dns.resolveNaptr()`]: #dnsresolvenaptrhostname-callback
[`dns.resolveNs()`]: #dnsresolvenshostname-callback
[`dns.resolvePtr()`]: #dnsresolveptrhostname-callback
[`dns.resolveSoa()`]: #dnsresolvesoahostname-callback
[`dns.resolveSrv()`]: #dnsresolvesrvhostname-callback
[`dns.resolveTlsa()`]: #dnsresolvetlsahostname-callback
[`dns.resolveTxt()`]: #dnsresolvetxthostname-callback
[`dns.reverse()`]: #dnsreverseip-callback
[`dns.setDefaultResultOrder()`]: #dnssetdefaultresultorderorder
[`dns.setServers()`]: #dnssetserversservers
[`dnsPromises.getServers()`]: #dnspromisesgetservers
[`dnsPromises.lookup()`]: #dnspromiseslookuphostname-options
[`dnsPromises.resolve()`]: #dnspromisesresolvehostname-rrtype
[`dnsPromises.resolve4()`]: #dnspromisesresolve4hostname-options
[`dnsPromises.resolve6()`]: #dnspromisesresolve6hostname-options
[`dnsPromises.resolveAny()`]: #dnspromisesresolveanyhostname
[`dnsPromises.resolveCaa()`]: #dnspromisesresolvecaahostname
[`dnsPromises.resolveCname()`]: #dnspromisesresolvecnamehostname
[`dnsPromises.resolveMx()`]: #dnspromisesresolvemxhostname
[`dnsPromises.resolveNaptr()`]: #dnspromisesresolvenaptrhostname
[`dnsPromises.resolveNs()`]: #dnspromisesresolvenshostname
[`dnsPromises.resolvePtr()`]: #dnspromisesresolveptrhostname
[`dnsPromises.resolveSoa()`]: #dnspromisesresolvesoahostname
[`dnsPromises.resolveSrv()`]: #dnspromisesresolvesrvhostname
[`dnsPromises.resolveTlsa()`]: #dnspromisesresolvetlsahostname
[`dnsPromises.resolveTxt()`]: #dnspromisesresolvetxthostname
[`dnsPromises.reverse()`]: #dnspromisesreverseip
[`dnsPromises.setDefaultResultOrder()`]: #dnspromisessetdefaultresultorderorder
[`dnsPromises.setServers()`]: #dnspromisessetserversservers
[`socket.connect()`]: net.md#socketconnectoptions-connectlistener
[`util.promisify()`]: util.md#utilpromisifyoriginal
[supported `getaddrinfo` flags]: #supported-getaddrinfo-flags
[worker threads]: worker_threads.md