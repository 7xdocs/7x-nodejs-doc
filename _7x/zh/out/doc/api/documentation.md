# 关于本文档

<!--introduced_in=v0.10.0-->

<!-- type=misc -->

欢迎查阅Node.js官方API参考文档！

Node.js是一个基于[V8 JavaScript引擎][]构建的JavaScript运行时。

## 贡献

请在[问题跟踪器][]中报告本文档中的错误。有关提交拉取请求的指导，请参阅[贡献指南][]。

## 稳定性索引

<!--type=misc-->

本文档中随处可见各部分的稳定性指示。有些API已被充分验证且被广泛依赖，因此几乎不可能发生任何变化。其他一些则是全新的、实验性的，或者已知存在风险。

稳定性索引如下：

> Stability: 0 - 已废弃。该特性可能会发出警告。不保证向后兼容性。

<!-- separator -->

> Stability: 1 - 实验性。该特性不受[语义化版本控制][]规则约束。未来任何版本都可能发生不兼容的变更或被移除。不建议在生产环境中使用该特性。
>
> 实验性特性细分为以下阶段：
>
> * 1.0 - 早期开发阶段。此阶段的实验性特性尚未完成，可能会有重大变更。
> * 1.1 - 积极开发阶段。此阶段的实验性特性接近最低可用状态。
> * 1.2 - 候选发布阶段。此阶段的实验性特性有望成为稳定版本。预计不会再有破坏性变更，但可能会根据用户反馈或特性的基础规范发展而发生变更。我们鼓励用户进行测试并提供反馈，以便我们确认该特性已准备好标记为稳定版本。
>
> 实验性特性通常会通过升级为稳定版本或直接移除（无废弃周期）来脱离实验状态。

<!-- separator -->

> Stability: 2 - 稳定。与npm生态系统的兼容性是首要任务。

<!-- separator -->

> Stability: 3 - 遗留。尽管此特性不太可能被移除，并且仍受语义化版本控制保障，但它不再被积极维护，且存在其他替代方案。

如果特性的使用无害且在npm生态系统中被广泛依赖，则会被标记为遗留特性而非废弃特性。在遗留特性中发现的漏洞不太可能被修复。

使用实验性特性时请谨慎，尤其是在编写库时。用户可能不知道正在使用实验性特性。当实验性API发生修改时，漏洞或行为变化可能会让用户感到意外。为避免意外，使用实验性特性可能需要命令行标志。实验性特性也可能会发出[警告][]。

## 稳定性概述

<!-- STABILITY_OVERVIEW_SLOT_BEGIN -->
| API | Stability |
| --- | --------- |
| [Assert](assert.html) | (2) Stable |
| [Assert](addons.html) | (2) Stable |
| [Buffer](buffer.html) | (2) Stable |
| [Cluster 集群](cluster.html) | (2) Stable |
| [Crypto](crypto.html) | (2) Stable |
| [Diagnostics Channel](diagnostics_channel.html) | (2) Stable |
| [Domain](domain.html) | (0) Deprecated |
| [HTTP](http.html) | (2) Stable |
| [HTTP/2](http2.html) | (2) Stable |
| [HTTPS](https.html) | (2) Stable |
| [Inspector](inspector.html) | (2) Stable |
| [Modules: `node:module` API](module.html) | (1) .2 - Release candidate (asynchronous version) Stability: 1.1 - Active development (synchronous version) |
| [Modules: CommonJS modules](modules.html) | (2) Stable |
| [Modules: TypeScript](typescript.html) | (1) .2 - Release candidate |
| [OS](os.html) | (2) Stable |
| [Path](path.html) | (2) Stable |
| [Performance measurement APIs](perf_hooks.html) | (2) Stable |
| [Punycode](punycode.html) | (0) Deprecated |
| [Query string](querystring.html) | (2) Stable |
| [Readline](readline.html) | (2) Stable |
| [REPL](repl.html) | (2) Stable |
| [SQLite](sqlite.html) | (1) .1 - Active development. |
| [Stream](stream.html) | (2) Stable |
| [String decoder](string_decoder.html) | (2) Stable |
| [Timers](timers.html) | (2) Stable |
| [TLS (SSL)](tls.html) | (2) Stable |
| [Trace events](tracing.html) | (1) Experimental |
| [TTY](tty.html) | (2) Stable |
| [UDP/datagram sockets](dgram.html) | (2) Stable |
| [URL](url.html) | (2) Stable |
| [Util](util.html) | (2) Stable |
| [VM (执行 JavaScript)](vm.html) | (2) Stable |
| [Web Crypto API](webcrypto.html) | (2) Stable |
| [Web Streams API](webstreams.html) | (2) Stable |
| [WebAssembly System Interface (WASI)](wasi.html) | (1) Experimental |
| [Worker threads](worker_threads.html) | (2) Stable |
| [Zlib](zlib.html) | (2) Stable |
| [单可执行应用](single-executable-applications.html) | (1) .1 - Active development |
| [域名系统 DNS](dns.html) | (2) Stable |
| [子进程](child_process.html) | (2) Stable |
| [异步上下文追踪](async_context.html) | (2) Stable |
| [异步钩子](async_hooks.html) | (1) 实验性。如果可能，请迁移远离此 API。 我们不建议使用 [`createHook`][]、[`AsyncHook`][] 和 [`executionAsyncResource`][] API，因为它们存在可用性问题、安全风险 和性能影响。异步上下文跟踪用例更适合使用 稳定的 [`AsyncLocalStorage`][] API。如果你有超出 [`AsyncLocalStorage`][] 解决的上下文跟踪需求或 [Diagnostics Channel][] 当前提供的诊断数据 之外的 `createHook`、`AsyncHook` 或 `executionAsyncResource` 用例， 请在 https://github.com/nodejs/node/issues 描述你的用例， 以便我们可以创建更专注的 API。 |
| [控制台](console.html) | (2) Stable |
| [文件系统](fs.html) | (2) Stable |
| [测试运行器](test.html) | (2) Stable |
<!-- STABILITY_OVERVIEW_SLOT_END -->

## JSON输出

<!-- YAML
added: v0.6.12
-->

每个`.html`文档都有一个对应的`.json`文档。这是为了供IDE和其他使用文档的工具使用。

## 系统调用和手册页

封装了系统调用的Node.js函数会对此进行说明。文档会链接到描述系统调用工作原理的相应手册页。

大多数Unix系统调用在Windows上有类似的实现。不过，行为差异可能难以避免。

[V8 JavaScript引擎]: https://v8.dev/
[语义化版本控制]: https://semver.org/
[贡献指南]: https://github.com/nodejs/node/blob/HEAD/CONTRIBUTING.md
[问题跟踪器]: https://github.com/nodejs/node/issues/new
[警告]: process.md#event-warning