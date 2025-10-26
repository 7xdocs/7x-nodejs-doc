# Domain

<!-- YAML
deprecated: v1.4.2
changes:
  - version: v8.8.0
    pr-url: https://github.com/nodejs/node/pull/15695
    description: Any `Promise`s created in VM contexts no longer have a
                 `.domain` property. Their handlers are still executed in the
                 proper domain, however, and `Promise`s created in the main
                 context still possess a `.domain` property.
  - version: v8.0.0
    pr-url: https://github.com/nodejs/node/pull/12489
    description: Handlers for `Promise`s are now invoked in the domain in which
                 the first promise of a chain was created.
-->

<!--introduced_in=v0.10.0-->

> Stability: 0 - Deprecated

<!-- source_link=lib/domain.js -->

**此模块即将被弃用。** 一旦替代 API 确定，此模块将被完全弃用。大多数开发者**没有**理由使用此模块。绝对必须拥有域提供功能的用户可以暂时依赖它，但应预期将来需要迁移到不同的解决方案。

域提供了一种将多个不同 IO 操作作为单个组处理的方法。如果注册到域的任何一个事件触发器或回调发出 `'error'` 事件，或抛出错误，那么域对象将被通知，而不是在 `process.on('uncaughtException')` 处理程序中丢失错误的上下文，或导致程序立即以错误代码退出。

## 警告：不要忽略错误！

<!-- type=misc -->

域错误处理程序不能替代在发生错误时关闭进程。

根据 JavaScript 中 [`throw`][] 的工作方式，几乎没有任何方法可以安全地“从中断处继续”，而不会泄漏引用或造成其他某种未定义的脆弱状态。

响应抛出错误的最安全方式是关闭进程。当然，在正常的 Web 服务器中，可能有许多打开的连接，因为其他人触发了错误就突然关闭这些连接是不合理的。

更好的方法是将错误响应发送给触发错误的请求，同时让其他请求在正常时间内完成，并停止在该工作进程中监听新请求。

这样，`domain` 的使用与集群模块密切相关，因为当工作进程遇到错误时，主进程可以分叉一个新的工作进程。对于扩展到多台机器的 Node.js 程序，终止代理或服务注册表可以记录故障并相应作出反应。

例如，这不是一个好主意：

```js
// XXX WARNING! BAD IDEA!

const d = require('node:domain').create();
d.on('error', (er) => {
  // The error won't crash the process, but what it does is worse!
  // Though we've prevented abrupt process restarting, we are leaking
  // a lot of resources if this ever happens.
  // This is no better than process.on('uncaughtException')!
  console.log(`error, but oh well ${er.message}`);
});
d.run(() => {
  require('node:http').createServer((req, res) => {
    handleRequest(req, res);
  }).listen(PORT);
});
```

通过使用域的上下文以及将程序分离为多个工作进程的弹性，我们可以更恰当地作出反应，并更安全地处理错误。

```js
// Much better!

const cluster = require('node:cluster');
const PORT = +process.env.PORT || 1337;

if (cluster.isPrimary) {
  // A more realistic scenario would have more than 2 workers,
  // and perhaps not put the primary and worker in the same file.
  //
  // It is also possible to get a bit fancier about logging, and
  // implement whatever custom logic is needed to prevent DoS
  // attacks and other bad behavior.
  //
  // See the options in the cluster documentation.
  //
  // The important thing is that the primary does very little,
  // increasing our resilience to unexpected errors.

  cluster.fork();
  cluster.fork();

  cluster.on('disconnect', (worker) => {
    console.error('disconnect!');
    cluster.fork();
  });

} else {
  // the worker
  //
  // This is where we put our bugs!

  const domain = require('node:domain');

  // See the cluster documentation for more details about using
  // worker processes to serve requests. How it works, caveats, etc.

  const server = require('node:http').createServer((req, res) => {
    const d = domain.create();
    d.on('error', (er) => {
      console.error(`error ${er.stack}`);

      // We're in dangerous territory!
      // By definition, something unexpected occurred,
      // which we probably didn't want.
      // Anything can happen now! Be very careful!

      try {
        // Make sure we close down within 30 seconds
        const killtimer = setTimeout(() => {
          process.exit(1);
        }, 30000);
        // But don't keep the process open just for that!
        killtimer.unref();

        // Stop taking new requests.
        server.close();

        // Let the primary know we're dead. This will trigger a
        // 'disconnect' in the cluster primary, and then it will fork
        // a new worker.
        cluster.worker.disconnect();

        // Try to send an error to the request that triggered the problem
        res.statusCode = 500;
        res.setHeader('content-type', 'text/plain');
        res.end('Oops, there was a problem!\n');
      } catch (er2) {
        // Oh well, not much we can do at this point.
        console.error(`Error sending 500! ${er2.stack}`);
      }
    });

    // Because req and res were created before this domain existed,
    // we need to explicitly add them.
    // See the explanation of implicit vs explicit binding below.
    d.add(req);
    d.add(res);

    // Now run the handler function in the domain.
    d.run(() => {
      handleRequest(req, res);
    });
  });
  server.listen(PORT);
}

// This part is not important. Just an example routing thing.
// Put fancy application logic here.
function handleRequest(req, res) {
  switch (req.url) {
    case '/error':
      // We do some async stuff, and then...
      setTimeout(() => {
        // Whoops!
        flerb.bark();
      }, timeout);
      break;
    default:
      res.end('ok');
  }
}
```

## 对 `Error` 对象的添加

<!-- type=misc -->

任何时候将 `Error` 对象路由通过域时，都会向其添加一些额外字段。

* `error.domain` 首先处理错误的域。
* `error.domainEmitter` 发出带有错误对象的 `'error'` 事件的事件触发器。
* `error.domainBound` 绑定到域的回调函数，并将错误作为其第一个参数传递。
* `error.domainThrown` 一个布尔值，指示错误是被抛出、发出还是传递给绑定的回调函数。

## 隐式绑定

<!--type=misc-->

如果正在使用域，那么所有**新的** `EventEmitter` 对象（包括流对象、请求、响应等）将在创建时隐式绑定到活动域。

此外，传递给低级事件循环请求（例如 `fs.open()` 或其他接受回调的方法）的回调将自动绑定到活动域。如果它们抛出错误，则域将捕获该错误。

为了防止过多的内存使用，`Domain` 对象本身不会隐式添加到活动域的子级。如果那样做，将太容易阻止请求和响应对象被正确垃圾回收。

要将 `Domain` 对象嵌套为父 `Domain` 的子级，必须显式添加它们。

隐式绑定将抛出的错误和 `'error'` 事件路由到 `Domain` 的 `'error'` 事件，但不会在 `Domain` 上注册 `EventEmitter`。
隐式绑定仅处理抛出的错误和 `'error'` 事件。

## 显式绑定

<!--type=misc-->

有时，正在使用的域不应用于特定的事件触发器。或者，事件触发器可能是在一个域的上下文中创建的，但应该绑定到其他域。

例如，可能有一个用于 HTTP 服务器的域，但也许我们希望为每个请求使用一个单独的域。

这可以通过显式绑定实现。

```js
// Create a top-level domain for the server
const domain = require('node:domain');
const http = require('node:http');
const serverDomain = domain.create();

serverDomain.run(() => {
  // Server is created in the scope of serverDomain
  http.createServer((req, res) => {
    // Req and res are also created in the scope of serverDomain
    // however, we'd prefer to have a separate domain for each request.
    // create it first thing, and add req and res to it.
    const reqd = domain.create();
    reqd.add(req);
    reqd.add(res);
    reqd.on('error', (er) => {
      console.error('Error', er, req.url);
      try {
        res.writeHead(500);
        res.end('Error occurred, sorry.');
      } catch (er2) {
        console.error('Error sending 500', er2, req.url);
      }
    });
  }).listen(1337);
});
```

## `domain.create()`

* 返回：{Domain}

## 类：`Domain`

* 扩展：{EventEmitter}

`Domain` 类封装了将错误和未捕获异常路由到活动 `Domain` 对象的功能。

要处理它捕获的错误，请监听其 `'error'` 事件。

### `domain.members`

* 类型：{Array}

已显式添加到域的计时器和事件触发器的数组。

### `domain.add(emitter)`

* `emitter` {EventEmitter|Timer} 要添加到域的发射器或计时器

显式将发射器添加到域。如果发射器调用的任何事件处理程序抛出错误，或者发射器发出 `'error'` 事件，它将被路由到域的 `'error'` 事件，就像隐式绑定一样。

这也适用于从 [`setInterval()`][] 和 [`setTimeout()`][] 返回的计时器。如果它们的回调函数抛出错误，它将被域 `'error'` 处理程序捕获。

如果计时器或 `EventEmitter` 已绑定到某个域，则将其从该域中移除，并绑定到此域。

### `domain.bind(callback)`

* `callback` {Function} 回调函数
* 返回：{Function} 绑定的函数

返回的函数将是提供的回调函数的包装器。当调用返回的函数时，抛出的任何错误都将被路由到域的 `'error'` 事件。

```js
const d = domain.create();

function readSomeFile(filename, cb) {
  fs.readFile(filename, 'utf8', d.bind((er, data) => {
    // If this throws, it will also be passed to the domain.
    return cb(er, data ? JSON.parse(data) : null);
  }));
}

d.on('error', (er) => {
  // An error occurred somewhere. If we throw it now, it will crash the program
  // with the normal line number and stack message.
});
```

### `domain.enter()`

`enter()` 方法是 `run()`、`bind()` 和 `intercept()` 方法用于设置活动域的内部方法。它将 `domain.active` 和 `process.domain` 设置为域，并隐式将域推送到域模块管理的域栈上（有关域栈的详细信息，请参见 [`domain.exit()`][]）。调用 `enter()` 划定了绑定到域的异步调用和 IO 操作链的开始。

调用 `enter()` 仅更改活动域，而不更改域本身。`enter()` 和 `exit()` 可以在单个域上任意调用多次。

### `domain.exit()`

`exit()` 方法退出当前域，将其从域栈中弹出。任何时候执行将切换到不同异步调用链的上下文时，确保退出当前域非常重要。调用 `exit()` 划定了绑定到域的异步调用和 IO 操作链的结束或中断。

如果当前执行上下文绑定了多个嵌套域，`exit()` 将退出此域内嵌套的任何域。

调用 `exit()` 仅更改活动域，而不更改域本身。`enter()` 和 `exit()` 可以在单个域上任意调用多次。

### `domain.intercept(callback)`

* `callback` {Function} 回调函数
* 返回：{Function} 拦截的函数

此方法几乎与 [`domain.bind(callback)`][] 相同。但是，除了捕获抛出的错误外，它还将拦截作为函数的第一个参数发送的 [`Error`][] 对象。

这样，常见的 `if (err) return callback(err);` 模式可以用单个错误处理程序在单个位置替换。

```js
const d = domain.create();

function readSomeFile(filename, cb) {
  fs.readFile(filename, 'utf8', d.intercept((data) => {
    // Note, the first argument is never passed to the
    // callback since it is assumed to be the 'Error' argument
    // and thus intercepted by the domain.

    // If this throws, it will also be passed to the domain
    // so the error-handling logic can be moved to the 'error'
    // event on the domain instead of being repeated throughout
    // the program.
    return cb(null, JSON.parse(data));
  }));
}

d.on('error', (er) => {
  // An error occurred somewhere. If we throw it now, it will crash the program
  // with the normal line number and stack message.
});
```

### `domain.remove(emitter)`

* `emitter` {EventEmitter|Timer} 要从域中移除的发射器或计时器

与 [`domain.add(emitter)`][] 相反。从指定的发射器中移除域处理。

### `domain.run(fn[, ...args])`

* `fn` {Function}
* `...args` {any}

在域的上下文中运行提供的函数，隐式绑定在该上下文中创建的所有事件触发器、计时器和低级请求。可以选择将参数传递给函数。

这是使用域的最基本方式。

```js
const domain = require('node:domain');
const fs = require('node:fs');
const d = domain.create();
d.on('error', (er) => {
  console.error('Caught error!', er);
});
d.run(() => {
  process.nextTick(() => {
    setTimeout(() => { // Simulating some various async stuff
      fs.open('non-existent file', 'r', (er, fd) => {
        if (er) throw er;
        // proceed...
      });
    }, 100);
  });
});
```

在此示例中，将触发 `d.on('error')` 处理程序，而不是使程序崩溃。

## 域和 Promise

从 Node.js 8.0.0 开始，Promise 的处理程序在调用 `.then()` 或 `.catch()` 的域中运行：

```js
const d1 = domain.create();
const d2 = domain.create();

let p;
d1.run(() => {
  p = Promise.resolve(42);
});

d2.run(() => {
  p.then((v) => {
    // running in d2
  });
});
```

可以使用 [`domain.bind(callback)`][] 将回调绑定到特定域：

```js
const d1 = domain.create();
const d2 = domain.create();

let p;
d1.run(() => {
  p = Promise.resolve(42);
});

d2.run(() => {
  p.then(p.domain.bind((v) => {
    // running in d1
  }));
});
```

域不会干扰 Promise 的错误处理机制。换句话说，不会为未处理的 `Promise` 拒绝发出 `'error'` 事件。

[`Error`]: errors.md#class-error
[`domain.add(emitter)`]: #domainaddemitter
[`domain.bind(callback)`]: #domainbindcallback
[`domain.exit()`]: #domainexit
[`setInterval()`]: timers.md#setintervalcallback-delay-args
[`setTimeout()`]: timers.md#settimeoutcallback-delay-args
[`throw`]: https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/throw