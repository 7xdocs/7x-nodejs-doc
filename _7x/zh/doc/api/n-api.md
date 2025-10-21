# Node-API

<!--introduced_in=v8.0.0-->

<!-- type=misc -->

> Stability: 2 - Stable

Node-API（原名 N-API）是一个用于构建原生插件的 API。它独立于底层的 JavaScript 运行时（例如 V8），并作为 Node.js 自身的一部分进行维护。该 API 将在 Node.js 各个版本中保持应用程序二进制接口（ABI）稳定。其目的是将插件与底层 JavaScript 引擎的变化隔离开来，并允许为一个主要版本编译的模块在后续主要版本的 Node.js 上无需重新编译即可运行。[ABI 稳定性][]指南提供了更深入的解释。

插件的构建/打包方法与[ C++ 插件][]章节中概述的方法相同。唯一的区别是原生代码所使用的 API 集合。这里使用的是 Node-API 中可用的函数，而不是 V8 或 [Native Abstractions for Node.js][] API。

Node-API 暴露的 API 通常用于创建和操作 JavaScript 值。概念和操作通常映射到 ECMA-262 语言规范中指定的思想。这些 API 具有以下特性：

* 所有 Node-API 调用都会返回一个类型为 `napi_status` 的状态码。该状态表示 API 调用成功还是失败。
* API 的返回值通过输出参数传递。
* 所有 JavaScript 值都抽象在一个名为 `napi_value` 的不透明类型后面。
* 如果出现错误状态码，可以使用 `napi_get_last_error_info` 获取额外信息。更多信息请参阅错误处理章节[错误处理][]。

## 使用不同编程语言编写插件

Node-API 是一个 C API，确保跨 Node.js 版本和不同编译器级别的 ABI 稳定性。有了这种稳定性保证，可以在 Node-API 之上使用其他编程语言编写插件。有关更多编程语言和引擎绑定的支持详情，请参考[语言和引擎绑定][]。

[`node-addon-api`][] 是官方的 C++ 绑定，提供了一种更高效的方式来编写调用 Node-API 的 C++ 代码。这个包装器是一个仅头文件的库，提供了可内联的 C++ API。使用 `node-addon-api` 构建的二进制文件将依赖于 Node.js 导出的基于 C 的 Node-API 函数的符号。以下代码片段是 `node-addon-api` 的一个示例：

```cpp
Object obj = Object::New(env);
obj["foo"] = String::New(env, "bar");
```

上面的 `node-addon-api` C++ 代码等价于以下基于 C 的 Node-API 代码：

```cpp
napi_status status;
napi_value object, string;
status = napi_create_object(env, &object);
if (status != napi_ok) {
  napi_throw_error(env, ...);
  return;
}

status = napi_create_string_utf8(env, "bar", NAPI_AUTO_LENGTH, &string);
if (status != napi_ok) {
  napi_throw_error(env, ...);
  return;
}

status = napi_set_named_property(env, object, "foo", string);
if (status != napi_ok) {
  napi_throw_error(env, ...);
  return;
}
```

最终结果是插件仅使用导出的 C API。即使插件是用 C++ 编写的，它仍然能获得 C Node-API 提供的 ABI 稳定性的好处。

当使用 `node-addon-api` 而不是 C API 时，请从 `node-addon-api` 的 API [文档][]开始。

[Node-API Resource](https://nodejs.github.io/node-addon-examples/) 为刚接触 Node-API 和 `node-addon-api` 的开发人员提供了极好的指导和提示。更多媒体资源可以在 [Node-API Media][] 页面上找到。

## ABI 稳定性的影响

尽管 Node-API 提供了 ABI 稳定性保证，但 Node.js 的其他部分并没有，并且插件使用的任何外部库也可能没有。特别是，以下 API 都不保证跨主要版本的 ABI 稳定性：

* 通过以下任何方式可用的 Node.js C++ API

  ```cpp
  #include <node.h>
  #include <node_buffer.h>
  #include <node_version.h>
  #include <node_object_wrap.h>
  ```

* 同样包含在 Node.js 中并通过以下方式可用的 libuv API

  ```cpp
  #include <uv.h>
  ```

* 通过以下方式可用的 V8 API

  ```cpp
  #include <v8.h>
  ```

因此，为了让插件在 Node.js 主要版本之间保持 ABI 兼容，它必须仅使用 Node-API，通过限制自己使用

```c
#include <node_api.h>
```

并检查其使用的所有外部库，确保外部库提供与 Node-API 类似的 ABI 稳定性保证。

### ABI 稳定性中的枚举值

Node-API 中定义的所有枚举数据类型应视为固定大小的 `int32_t` 值。位标志枚举类型应明确记录，并且它们作为位值使用位运算符（如位或 `|`）工作。除非另有说明，枚举类型应视为可扩展的。

新的枚举值将添加到枚举定义的末尾。枚举值不会被移除或重命名。

对于从 Node-API 函数返回的枚举类型，或作为 Node-API 函数的输出参数提供的枚举类型，该值是一个整数值，插件应处理未知值。允许引入新值而无需版本保护。例如，在检查 `napi_status` 的 switch 语句中，插件应包含一个默认分支，因为更新的 Node.js 版本可能会引入新的状态码。

对于用作输入参数的枚举类型，除非另有说明，将未知整数值传递给 Node-API 函数的结果是未定义的。新值会通过版本保护添加，以指示引入该值的 Node-API 版本。例如，`napi_get_all_property_names` 可以使用新的枚举值 `napi_key_filter` 进行扩展。

对于同时用作输入参数和输出参数的枚举类型，允许在没有版本保护的情况下引入新值。

## 构建

与用 JavaScript 编写的模块不同，使用 Node-API 开发和部署 Node.js 原生插件需要一组额外的工具。除了开发 Node.js 所需的基本工具外，原生插件开发者还需要一个能够将 C 和 C++ 代码编译成二进制文件的工具链。此外，根据原生插件的部署方式，原生插件的_用户_也需要安装 C/C++ 工具链。

对于 Linux 开发者，必要的 C/C++ 工具链包很容易获得。[GCC][] 在 Node.js 社区中被广泛用于跨平台构建和测试。对于许多开发者来说，[LLVM][] 编译器基础设施也是一个不错的选择。

对于 Mac 开发者，[Xcode][] 提供了所有必需的编译器工具。然而，没有必要安装整个 Xcode IDE。以下命令安装了必要的工具链：

```bash
xcode-select --install
```

对于 Windows 开发者，[Visual Studio][] 提供了所有必需的编译器工具。然而，没有必要安装整个 Visual Studio IDE。以下命令安装了必要的工具链：

```bash
npm install --global windows-build-tools
```

以下章节描述了可用于开发和部署 Node.js 原生插件的额外工具。

### 构建工具

这里列出的两种工具都要求原生插件的_用户_安装了 C/C++ 工具链才能成功安装原生插件。

#### node-gyp

[node-gyp][] 是一个基于 Google [GYP][] 工具的 [gyp-next][] 分支的构建系统，并与 npm 捆绑发布。GYP 以及因此的 node-gyp 要求安装 Python。

历史上，node-gyp 一直是构建原生插件的首选工具。它有着广泛的采用和文档。然而，一些开发者遇到了 node-gyp 的限制。

#### CMake.js

[CMake.js][] 是一个基于 [CMake][] 的替代构建系统。

CMake.js 适用于已经使用 CMake 的项目或受 node-gyp 限制影响的开发者。[`build_with_cmake`][] 是一个基于 CMake 的原生插件项目示例。

### 上传预编译二进制文件

这里列出的三种工具允许原生插件开发者和维护者创建二进制文件并上传到公共或私有服务器。这些工具通常与 CI/CD 构建系统（如 [Travis CI][] 和 [AppVeyor][]）集成，以构建并上传适用于各种平台和架构的二进制文件。然后，这些二进制文件可供不需要安装 C/C++ 工具链的用户下载。

#### node-pre-gyp

[node-pre-gyp][] 是一个基于 node-gyp 的工具，增加了将二进制文件上传到开发者选择的服务器的能力。node-pre-gyp 特别支持将二进制文件上传到 Amazon S3。

#### prebuild

[prebuild][] 是一个支持使用 node-gyp 或 CMake.js 进行构建的工具。与支持多种服务器的 node-pre-gyp 不同，prebuild 只将二进制文件上传到 [GitHub releases][]。prebuild 是使用 CMake.js 的 GitHub 项目的良好选择。

#### prebuildify

[prebuildify][] 是一个基于 node-gyp 的工具。prebuildify 的优点是构建的二进制文件在上传到 npm 时与原生插件捆绑在一起。当安装原生插件时，二进制文件从 npm 下载并立即可供模块用户使用。

## 用法

为了使用 Node-API 函数，请包含位于 node 开发树 src 目录中的文件 [`node_api.h`][]：

```c
#include <node_api.h>
```

这将选择给定 Node.js 版本的默认 `NAPI_VERSION`。为了确保与特定版本的 Node-API 兼容，可以在包含头文件时显式指定版本：

```c
#define NAPI_VERSION 3
#include <node_api.h>
```

这将 Node-API 表面限制为仅指定（及更早）版本中可用的功能。

部分 Node-API 表面是实验性的，需要显式选择加入：

```c
#define NAPI_EXPERIMENTAL
#include <node_api.h>
```

在这种情况下，整个 API 表面，包括任何实验性 API，将对模块代码可用。

偶尔，会引入影响已发布和稳定 API 的实验性功能。这些功能可以通过选择退出禁用：

```c
#define NAPI_EXPERIMENTAL
#define NODE_API_EXPERIMENTAL_<FEATURE_NAME>_OPT_OUT
#include <node_api.h>
```

其中 `<FEATURE_NAME>` 是影响实验性和稳定 API 的实验性功能的名称。

## Node-API 版本矩阵

直到版本 9，Node-API 版本是累加的，并且与 Node.js 独立版本化。这意味着任何版本都是对先前版本的扩展，因为它具有先前版本的所有 API 并增加了一些。每个 Node.js 版本只支持一个 Node-API 版本。例如 v18.15.0 仅支持 Node-API 版本 8。ABI 稳定性得以实现是因为版本 8 是所有先前版本的严格超集。

从版本 9 开始，虽然 Node-API 版本继续独立版本化，但运行在 Node-API 版本 9 的插件可能需要代码更新才能在 Node-API 版本 10 上运行。然而，ABI 稳定性得以维持，因为支持高于版本 8 的 Node-API 版本的 Node.js 版本将支持版本 8 到它们支持的最高版本之间的所有版本，并且默认提供版本 8 的 API，除非插件选择加入更高的 Node-API 版本。这种方法提供了更好优化现有 Node-API 函数的灵活性，同时保持 ABI 稳定性。现有的插件可以继续使用较早版本的 Node-API 运行而无需重新编译。如果插件需要来自较新 Node-API 版本的功能，则需要对现有代码进行更改并重新编译才能使用这些新函数。

在支持 Node-API 版本 9 及更高版本的 Node.js 版本中，定义 `NAPI_VERSION=X` 并使用现有的插件初始化宏将在插件中烘焙请求的 Node-API 版本，该版本将在运行时使用。如果未设置 `NAPI_VERSION`，则默认为 8。

此表在旧版本流中可能不是最新的，最新信息在最新的 API 文档中：
[Node-API 版本矩阵](https://nodejs.org/docs/latest/api/n-api.html#node-api-version-matrix)

<!-- 出于可访问性目的，此表需要行标题。这意味着我们不能用 markdown 来做。因此，使用原始 HTML。 -->

<table>
  <tr>
    <th>Node-API 版本</th>
    <th scope="col">支持起始版本</th>
  </tr>
  <tr>
    <th scope="row">10</th>
    <td>v22.14.0+, 23.6.0+ 及所有更高版本</td>
  </tr>
  <tr>
    <th scope="row">9</th>
    <td>v18.17.0+, 20.3.0+, 21.0.0 及所有更高版本</td>
  </tr>
  <tr>
    <th scope="row">8</th>
    <td>v12.22.0+, v14.17.0+, v15.12.0+, 16.0.0 及所有更高版本</td>
  </tr>
  <tr>
    <th scope="row">7</th>
    <td>v10.23.0+, v12.19.0+, v14.12.0+, 15.0.0 及所有更高版本</td>
  </tr>
  <tr>
    <th scope="row">6</th>
    <td>v10.20.0+, v12.17.0+, 14.0.0 及所有更高版本</td>
  </tr>
  <tr>
    <th scope="row">5</th>
    <td>v10.17.0+, v12.11.0+, 13.0.0 及所有更高版本</td>
  </tr>
  <tr>
    <th scope="row">4</th>
    <td>v10.16.0+, v11.8.0+, 12.0.0 及所有更高版本</td>
  </tr>
  </tr>
    <tr>
    <th scope="row">3</th>
    <td>v6.14.2*, 8.11.2+, v9.11.0+*, 10.0.0 及所有更高版本</td>
  </tr>
  <tr>
    <th scope="row">2</th>
    <td>v8.10.0+*, v9.3.0+*, 10.0.0 及所有更高版本</td>
  </tr>
  <tr>
    <th scope="row">1</th>
    <td>v8.6.0+**, v9.0.0+*, 10.0.0 及所有更高版本</td>
  </tr>
</table>

\* Node-API 是实验性的。

\*\* Node.js 8.0.0 包含了作为实验性的 Node-API。它作为 Node-API 版本 1 发布，但持续演进直到 Node.js 8.6.0。在 Node.js 8.6.0 之前的版本中，API 是不同的。我们推荐 Node-API 版本 3 或更高版本。

每个为 Node-API 记录的 API 都会有一个名为 `added in:` 的标题，稳定的 API 还会有额外的标题 `Node-API version:`。当使用支持 `Node-API version:` 中显示的 Node-API 版本或更高版本的 Node.js 版本时，API 可以直接使用。当使用不支持列出的 `Node-API version:` 的 Node.js 版本，或者没有列出 `Node-API version:` 时，只有 `#define NAPI_EXPERIMENTAL` 在包含 `node_api.h` 或 `js_native_api.h` 之前出现时，该 API 才可用。如果 API 在比 `added in:` 中显示的版本更新的 Node.js 版本上似乎不可用，那么这很可能是其明显缺失的原因。

严格与从原生代码访问 ECMAScript 功能相关的 Node-API 可以在 `js_native_api.h` 和 `js_native_api_types.h` 中找到。这些头文件中定义的 API 包含在 `node_api.h` 和 `node_api_types.h` 中。头文件以这种方式结构化是为了允许在 Node.js 之外实现 Node-API。对于那些实现，Node.js 特定的 API 可能不适用。

可以将插件的 Node.js 特定部分与向 JavaScript 环境暴露实际功能的代码分开，以便后者可以与多个 Node-API 实现一起使用。在下面的示例中，`addon.c` 和 `addon.h` 仅引用 `js_native_api.h`。这确保了 `addon.c` 可以重复用于针对 Node.js 的 Node-API 实现或 Node.js 之外的任何 Node-API 实现进行编译。

`addon_node.c` 是一个单独的文件，包含插件的 Node.js 特定入口点，并在插件加载到 Node.js 环境时通过调用 `addon.c` 来实例化插件。

```c
// addon.h
#ifndef _ADDON_H_
#define _ADDON_H_
#include <js_native_api.h>
napi_value create_addon(napi_env env);
#endif  // _ADDON_H_
```

```c
// addon.c
#include "addon.h"

#define NODE_API_CALL(env, call)                                  \
  do {                                                            \
    napi_status status = (call);                                  \
    if (status != napi_ok) {                                      \
      const napi_extended_error_info* error_info = NULL;          \
      napi_get_last_error_info((env), &error_info);               \
      const char* err_message = error_info->error_message;        \
      bool is_pending;                                            \
      napi_is_exception_pending((env), &is_pending);              \
      /* If an exception is already pending, don't rethrow it */  \
      if (!is_pending) {                                          \
        const char* message = (err_message == NULL)               \
            ? "empty error message"                               \
            : err_message;                                        \
        napi_throw_error((env), NULL, message);                   \
      }                                                           \
      return NULL;                                                \
    }                                                             \
  } while(0)

static napi_value
DoSomethingUseful(napi_env env, napi_callback_info info) {
  // Do something useful.
  return NULL;
}

napi_value create_addon(napi_env env) {
  napi_value result;
  NODE_API_CALL(env, napi_create_object(env, &result));

  napi_value exported_function;
  NODE_API_CALL(env, napi_create_function(env,
                                          "doSomethingUseful",
                                          NAPI_AUTO_LENGTH,
                                          DoSomethingUseful,
                                          NULL,
                                          &exported_function));

  NODE_API_CALL(env, napi_set_named_property(env,
                                             result,
                                             "doSomethingUseful",
                                             exported_function));

  return result;
}
```

```c
// addon_node.c
#include <node_api.h>
#include "addon.h"

NAPI_MODULE_INIT(/* napi_env env, napi_value exports */) {
  // This function body is expected to return a `napi_value`.
  // The variables `napi_env env` and `napi_value exports` may be used within
  // the body, as they are provided by the definition of `NAPI_MODULE_INIT()`.
  return create_addon(env);
}
```

## 环境生命周期 API

[ECMAScript 语言规范][]的[代理部分][]定义了"代理"的概念，作为 JavaScript 代码运行的自包含环境。进程可以并发或顺序地启动和终止多个这样的代理。

Node.js 环境对应于一个 ECMAScript 代理。在主进程中，环境在启动时创建，并且可以在单独的线程上创建额外的环境作为[工作线程][]。当 Node.js 嵌入到另一个应用程序中时，应用程序的主线程也可能在应用程序进程的生命周期内多次构造和销毁 Node.js 环境，这样每个由应用程序创建的 Node.js 环境又可以在其生命周期内创建和销毁作为工作线程的额外环境。

从原生插件的角度来看，这意味着它提供的绑定可能会被多次调用，来自多个上下文，甚至并发地从多个线程调用。

原生插件可能需要分配在 Node.js 环境生命周期内使用的全局状态，使得状态对于插件的每个实例都是唯一的。

为此，Node-API 提供了一种关联数据的方式，使得其生命周期与 Node.js 环境的生命周期绑定。

### `napi_set_instance_data`

<!-- YAML
added:
 - v12.8.0
 - v10.20.0
napiVersion: 6
-->

```c
napi_status napi_set_instance_data(node_api_basic_env env,
                                   void* data,
                                   napi_finalize finalize_cb,
                                   void* finalize_hint);
```

* `[in] env`: 调用 Node-API 的环境。
* `[in] data`: 要使此实例的绑定可用的数据项。
* `[in] finalize_cb`: 当环境被拆除时要调用的函数。该函数接收 `data` 以便可以释放它。[`napi_finalize`][] 提供了更多细节。
* `[in] finalize_hint`: 在收集期间传递给最终化回调的可选提示。

如果 API 成功则返回 `napi_ok`。

此 API 将 `data` 与当前运行的 Node.js 环境关联。`data` 稍后可以使用 `napi_get_instance_data()` 检索。任何先前通过调用 `napi_set_instance_data()` 与当前运行的 Node.js 环境关联的现有数据将被覆盖。如果先前的调用提供了 `finalize_cb`，则不会调用它。

### `napi_get_instance_data`

<!-- YAML
added:
 - v12.8.0
 - v10.20.0
napiVersion: 6
-->

```c
napi_status napi_get_instance_data(node_api_basic_env env,
                                   void** data);
```

* `[in] env`: 调用 Node-API 的环境。
* `[out] data`: 先前通过调用 `napi_set_instance_data()` 与当前运行的 Node.js 环境关联的数据项。

如果 API 成功则返回 `napi_ok`。

此 API 检索先前通过 `napi_set_instance_data()` 与当前运行的 Node.js 环境关联的数据。如果没有设置数据，调用将成功，并且 `data` 将被设置为 `NULL`。

## 基本 Node-API 数据类型

Node-API 暴露以下基本数据类型作为各种 API 使用的抽象。这些 API 应被视为不透明的，只能通过其他 Node-API 调用来内省。

### `napi_status`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

指示 Node-API 调用成功或失败的整数状态码。目前，支持以下状态码。

```c
typedef enum {
  napi_ok,
  napi_invalid_arg,
  napi_object_expected,
  napi_string_expected,
  napi_name_expected,
  napi_function_expected,
  napi_number_expected,
  napi_boolean_expected,
  napi_array_expected,
  napi_generic_failure,
  napi_pending_exception,
  napi_cancelled,
  napi_escape_called_twice,
  napi_handle_scope_mismatch,
  napi_callback_scope_mismatch,
  napi_queue_full,
  napi_closing,
  napi_bigint_expected,
  napi_date_expected,
  napi_arraybuffer_expected,
  napi_detachable_arraybuffer_expected,
  napi_would_deadlock,  /* unused */
  napi_no_external_buffers_allowed,
  napi_cannot_run_js
} napi_status;
```

如果在 API 返回失败状态时需要额外信息，可以通过调用 `napi_get_last_error_info` 获取。

### `napi_extended_error_info`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
typedef struct {
  const char* error_message;
  void* engine_reserved;
  uint32_t engine_error_code;
  napi_status error_code;
} napi_extended_error_info;
```

* `error_message`: UTF8 编码的字符串，包含错误的 VM 中立描述。
* `engine_reserved`: 保留供 VM 特定的错误详情。目前没有为任何 VM 实现。
* `engine_error_code`: VM 特定的错误码。目前没有为任何 VM 实现。
* `error_code`: 源自最后一个错误的 Node-API 状态码。

有关更多信息，请参阅[错误处理][]部分。

### `napi_env`

`napi_env` 用于表示底层 Node-API 实现可以用来持久化 VM 特定状态的上下文。此结构在调用原生函数时传递给它们，并且在发出 Node-API 调用时必须传递回来。具体来说，传递给初始原生函数的相同 `napi_env` 必须传递给任何后续的嵌套 Node-API 调用。为了通用重用而缓存 `napi_env`，并在运行在不同 [`Worker`][] 线程上的同一插件的实例之间传递 `napi_env` 是不允许的。当原生插件的实例被卸载时，`napi_env` 变为无效。此事件的通知通过给 [`napi_add_env_cleanup_hook`][] 和 [`napi_set_instance_data`][] 的回调传递。

### `node_api_basic_env`

> Stability: 1 - Experimental

此 `napi_env` 变体传递给同步终结器（[`node_api_basic_finalize`][]）。有一个接受 `node_api_basic_env` 类型参数作为其第一个参数的 Node-API 子集。这些 API 不访问 JavaScript 引擎的状态，因此从同步终结器调用是安全的。允许将 `napi_env` 类型的参数传递给这些 API，但是，不允许将 `node_api_basic_env` 类型的参数传递给访问 JavaScript 引擎状态的 API。尝试在没有强制转换的情况下这样做将在插件编译时产生编译器警告或错误，这些标志会在将不正确的指针类型传递给函数时发出警告和/或错误。从同步终结器调用此类 API 最终将导致应用程序终止。

### `napi_value`

这是一个不透明指针，用于表示 JavaScript 值。

### `napi_threadsafe_function`

<!-- YAML
added: v10.6.0
napiVersion: 4
-->

这是一个不透明指针，表示一个 JavaScript 函数，可以通过 `napi_call_threadsafe_function()` 从多个线程异步调用。

### `napi_threadsafe_function_release_mode`

<!-- YAML
added: v10.6.0
napiVersion: 4
-->

要给予 `napi_release_threadsafe_function()` 的值，指示线程安全函数是立即关闭（`napi_tsfn_abort`）还是仅释放（`napi_tsfn_release`），从而可通过 `napi_acquire_threadsafe_function()` 和 `napi_call_threadsafe_function()` 后续使用。

```c
typedef enum {
  napi_tsfn_release,
  napi_tsfn_abort
} napi_threadsafe_function_release_mode;
```

### `napi_threadsafe_function_call_mode`

<!-- YAML
added: v10.6.0
napiVersion: 4
-->

要给予 `napi_call_threadsafe_function()` 的值，指示当与线程安全函数关联的队列已满时调用是否应该阻塞。

```c
typedef enum {
  napi_tsfn_nonblocking,
  napi_tsfn_blocking
} napi_threadsafe_function_call_mode;
```

### Node-API 内存管理类型

#### `napi_handle_scope`

这是一个用于控制和修改特定作用域内创建的对象的生命周期的抽象。通常，Node-API 值在句柄作用域的上下文中创建。当从 JavaScript 调用原生方法时，将存在一个默认的句柄作用域。如果用户没有显式创建新的句柄作用域，Node-API 值将在默认句柄作用域中创建。对于在原生方法执行之外的任何代码调用（例如，在 libuv 回调调用期间），模块需要在调用任何可能导致 JavaScript 值创建的函数之前创建一个作用域。

使用 [`napi_open_handle_scope`][] 创建句柄作用域，并使用 [`napi_close_handle_scope`][] 销毁。关闭作用域可以向 GC 指示在句柄作用域生命周期内创建的所有 `napi_value` 不再从当前堆栈帧引用。

有关更多详细信息，请查看[对象生命周期管理][]。

#### `napi_escapable_handle_scope`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

可逃脱句柄作用域是一种特殊类型的句柄作用域，用于将特定句柄作用域内创建的值返回到父作用域。

#### `napi_ref`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

这是用于引用 `napi_value` 的抽象。这允许用户管理 JavaScript 值的生命周期，包括显式定义它们的最小生命周期。

有关更多详细信息，请查看[对象生命周期管理][]。

#### `napi_type_tag`

<!-- YAML
added:
  - v14.8.0
  - v12.19.0
napiVersion: 8
-->

存储为两个无符号 64 位整数的 128 位值。它用作 UUID，可以"标记" JavaScript 对象或 [外部对象][]，以确保它们是某种类型。这比 [`napi_instanceof`][] 更强的检查，因为如果对象的原型被操纵，后者可能报告假阳性。类型标记与 [`napi_wrap`][] 结合使用时最有用，因为它确保从包装对象检索的指针可以安全地转换为先前应用于 JavaScript 对象的类型标签对应的原生类型。

```c
typedef struct {
  uint64_t lower;
  uint64_t upper;
} napi_type_tag;
```

#### `napi_async_cleanup_hook_handle`

<!-- YAML
added:
  - v14.10.0
  - v12.19.0
-->

由 [`napi_add_async_cleanup_hook`][] 返回的不透明值。当异步清理事件链完成时，必须将其传递给 [`napi_remove_async_cleanup_hook`][]。

### Node-API 回调类型

#### `napi_callback_info`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

传递给回调函数的不透明数据类型。它可以用于获取有关调用回调的上下文的额外信息。

#### `napi_callback`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

用户提供的原生函数的函数指针类型，这些函数将通过 Node-API 暴露给 JavaScript。回调函数应满足以下签名：

```c
typedef napi_value (*napi_callback)(napi_env, napi_callback_info);
```

除非[对象生命周期管理][]中讨论的原因，否则在 `napi_callback` 内部创建句柄和/或回调作用域不是必需的。

#### `node_api_basic_finalize`

<!-- YAML
added:
  - v21.6.0
  - v20.12.0
  - v18.20.0
-->

> Stability: 1 - Experimental

插件提供的函数的函数指针类型，允许用户在外部分配的数据准备好清理时收到通知，因为与其关联的对象已被垃圾回收。用户必须提供一个满足以下签名的函数，该函数将在对象被收集时调用。目前，`node_api_basic_finalize` 可用于找出具有外部数据的对象何时被收集。

```c
typedef void (*node_api_basic_finalize)(node_api_basic_env env,
                                      void* finalize_data,
                                      void* finalize_hint);
```

除非[对象生命周期管理][]中讨论的原因，否则在函数体内创建句柄和/或回调作用域不是必需的。

由于这些函数可能在 JavaScript 引擎处于无法执行 JavaScript 代码的状态时调用，因此只能调用接受 `node_api_basic_env` 作为其第一个参数的 Node-API。[`node_api_post_finalizer`][] 可用于安排在当前垃圾回收周期完成后运行需要访问 JavaScript 引擎状态的 Node-API 调用。

对于 [`node_api_create_external_string_latin1`][] 和 [`node_api_create_external_string_utf16`][]，`env` 参数可能为 null，因为外部字符串可以在环境关闭的后期阶段被收集。

变更历史：

* 实验性（`NAPI_EXPERIMENTAL`）：

  只能调用接受 `node_api_basic_env` 作为其第一个参数的 Node-API 调用，否则应用程序将终止并显示适当的错误消息。可以通过定义 `NODE_API_EXPERIMENTAL_BASIC_ENV_OPT_OUT` 来关闭此功能。

#### `napi_finalize`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

插件提供的函数的函数指针类型，允许用户安排一组 Node-API 调用以响应垃圾回收事件，在垃圾回收周期完成后。这些函数指针可以与 [`node_api_post_finalizer`][] 一起使用。

```c
typedef void (*napi_finalize)(napi_env env,
                              void* finalize_data,
                              void* finalize_hint);
```

变更历史：

* 实验性（定义了 `NAPI_EXPERIMENTAL`）：

  此类型的函数可能不再用作终结器，除非与 [`node_api_post_finalizer`][] 一起使用。必须改用 [`node_api_basic_finalize`][]。可以通过定义 `NODE_API_EXPERIMENTAL_BASIC_ENV_OPT_OUT` 来关闭此功能。

#### `napi_async_execute_callback`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

用于支持异步操作的函数的函数指针。回调函数必须满足以下签名：

```c
typedef void (*napi_async_execute_callback)(napi_env env, void* data);
```

此函数的实现必须避免执行 JavaScript 或与 JavaScript 对象交互的 Node-API 调用。Node-API 调用应在 `napi_async_complete_callback` 中进行。不要使用 `napi_env` 参数，因为它很可能导致 JavaScript 执行。

#### `napi_async_complete_callback`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

用于支持异步操作的函数的函数指针。回调函数必须满足以下签名：

```c
typedef void (*napi_async_complete_callback)(napi_env env,
                                             napi_status status,
                                             void* data);
```

除非[对象生命周期管理][]中讨论的原因，否则在函数体内创建句柄和/或回调作用域不是必需的。

#### `napi_threadsafe_function_call_js`

<!-- YAML
added: v10.6.0
napiVersion: 4
-->

用于异步线程安全函数调用的函数指针。回调将在主线程上调用。其目的是使用从其中一个辅助线程通过队列到达的数据项来构造调用 JavaScript 所需的参数，通常通过 `napi_call_function`，然后调用 JavaScript。

从辅助线程通过队列到达的数据在 `data` 参数中给出，要调用的 JavaScript 函数在 `js_callback` 参数中给出。

Node-API 在调用此回调之前设置了环境，因此通过 `napi_call_function` 调用 JavaScript 函数就足够了，而不是通过 `napi_make_callback`。

回调函数必须满足以下签名：

```c
typedef void (*napi_threadsafe_function_call_js)(napi_env env,
                                                 napi_value js_callback,
                                                 void* context,
                                                 void* data);
```

* `[in] env`: 用于 API 调用的环境，或者如果线程安全函数正在被拆除并且 `data` 可能需要释放，则为 `NULL`。
* `[in] js_callback`: 要调用的 JavaScript 函数，或者如果线程安全函数正在被拆除并且 `data` 可能需要释放，则为 `NULL`。如果线程安全函数是在没有 `js_callback` 的情况下创建的，它也可能是 `NULL`。
* `[in] context`: 线程安全函数创建时的可选数据。
* `[in] data`: 由辅助线程创建的数据。回调负责将此原生数据转换为 JavaScript 值（使用 Node-API 函数），这些值可以在调用 `js_callback` 时作为参数传递。此指针完全由线程和此回调管理。因此，此回调应释放数据。

除非[对象生命周期管理][]中讨论的原因，否则在函数体内创建句柄和/或回调作用域不是必需的。

#### `napi_cleanup_hook`

<!-- YAML
added:
  - v19.2.0
  - v18.13.0
napiVersion: 3
-->

与 [`napi_add_env_cleanup_hook`][] 一起使用的函数指针。它将在环境被拆除时调用。

回调函数必须满足以下签名：

```c
typedef void (*napi_cleanup_hook)(void* data);
```

* `[in] data`: 传递给 [`napi_add_env_cleanup_hook`][] 的数据。

#### `napi_async_cleanup_hook`

<!-- YAML
added:
  - v14.10.0
  - v12.19.0
-->

与 [`napi_add_async_cleanup_hook`][] 一起使用的函数指针。它将在环境被拆除时调用。

回调函数必须满足以下签名：

```c
typedef void (*napi_async_cleanup_hook)(napi_async_cleanup_hook_handle handle,
                                        void* data);
```

* `[in] handle`: 在异步清理完成后必须传递给 [`napi_remove_async_cleanup_hook`][] 的句柄。
* `[in] data`: 传递给 [`napi_add_async_cleanup_hook`][] 的数据。

函数体应在异步清理操作结束时启动，在结束时必须将 `handle` 传递给 [`napi_remove_async_cleanup_hook`][] 的调用。

## 错误处理

Node-API 同时使用返回值和 JavaScript 异常进行错误处理。以下部分解释了每种情况的方法。

### 返回值

所有 Node-API 函数共享相同的错误处理模式。所有 API 函数的返回类型都是 `napi_status`。

如果请求成功且没有未捕获的 JavaScript 异常抛出，则返回值为 `napi_ok`。如果发生错误并且抛出了异常，将返回错误的 `napi_status` 值。如果抛出了异常但没有发生错误，将返回 `napi_pending_exception`。

在返回值不是 `napi_ok` 或 `napi_pending_exception` 的情况下，必须调用 [`napi_is_exception_pending`][] 来检查是否有异常挂起。有关更多详细信息，请参阅异常部分。

完整的 `napi_status` 值集合在 `napi_api_types.h` 中定义。

`napi_status` 返回值提供了发生的错误的 VM 独立表示。在某些情况下，能够获取更详细的信息是有用的，包括表示错误的字符串以及 VM（引擎）特定信息。

为了检索此信息，提供了 [`napi_get_last_error_info`][]，它返回一个 `napi_extended_error_info` 结构。`napi_extended_error_info` 结构的格式如下：

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
typedef struct napi_extended_error_info {
  const char* error_message;
  void* engine_reserved;
  uint32_t engine_error_code;
  napi_status error_code;
};
```

* `error_message`: 发生的错误的文本表示。
* `engine_reserved`: 仅保留供引擎使用的不透明句柄。
* `engine_error_code`: VM 特定的错误码。
* `error_code`: 最后一个错误的 Node-API 状态码。

[`napi_get_last_error_info`][] 返回上次进行的 Node-API 调用的信息。

不要依赖任何扩展信息的内容或格式，因为它不受 SemVer 约束，并且可能随时更改。它仅用于日志记录目的。

#### `napi_get_last_error_info`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status
napi_get_last_error_info(node_api_basic_env env,
                         const napi_extended_error_info** result);
```

* `[in] env`: 调用 API 的环境。
* `[out] result`: 包含有关错误的更多信息的 `napi_extended_error_info` 结构。

如果 API 成功则返回 `napi_ok`。

此 API 检索一个 `napi_extended_error_info` 结构，其中包含有关上次发生的错误的信息。

返回的 `napi_extended_error_info` 的内容仅在同一个 `env` 上调用 Node-API 函数之前有效。这包括调用 `napi_is_exception_pending`，因此通常需要复制信息以便以后使用。返回的 `error_message` 指针指向一个静态定义的字符串，因此如果在另一个 Node-API 函数被调用之前将其从 `error_message` 字段（将被覆盖）中复制出来，使用该指针是安全的。

不要依赖任何扩展信息的内容或格式，因为它不受 SemVer 约束，并且可能随时更改。它仅用于日志记录目的。

即使有挂起的 JavaScript 异常，也可以调用此 API。

### 异常

任何 Node-API 函数调用都可能导致挂起的 JavaScript 异常。对于任何 API 函数都是如此，即使是那些可能不导致 JavaScript 执行的函数。

如果函数返回的 `napi_status` 是 `napi_ok`，则没有异常挂起，不需要额外的操作。如果返回的 `napi_status` 是 `napi_ok` 或 `napi_pending_exception` 以外的任何值，为了尝试恢复并继续而不是立即返回，必须调用 [`napi_is_exception_pending`][] 来确定是否有异常挂起。

在许多情况下，当调用 Node-API 函数并且已经有异常挂起时，函数将立即返回，`napi_status` 为 `napi_pending_exception`。然而，并非所有函数都是如此。Node-API 允许调用一部分函数，以便在返回 JavaScript 之前进行一些最小的清理。在这种情况下，`napi_status` 将反映函数的状态。它不会反映先前挂起的异常。为了避免混淆，请在每次函数调用后检查错误状态。

当有异常挂起时，可以采用两种方法之一。

第一种方法是进行任何适当的清理，然后返回，以便执行将返回到 JavaScript。在转换回 JavaScript 的过程中，异常将在调用原生方法的 JavaScript 代码点抛出。在异常挂起时，大多数 Node-API 调用的行为是未指定的，许多将简单地返回 `napi_pending_exception`，因此尽可能少做，然后返回到 JavaScript，在那里可以处理异常。

第二种方法是尝试处理异常。在某些情况下，原生代码可以捕获异常，采取适当的操作，然后继续。这仅在已知可以安全处理异常的具体情况下推荐。在这些情况下，可以使用 [`napi_get_and_clear_last_exception`][] 来获取和清除异常。成功后，结果将包含最后一个抛出的 JavaScript `Object` 的句柄。如果在检索异常后确定无法处理，可以使用 [`napi_throw`][] 重新抛出它，其中 error 是要抛出的 JavaScript 值。

如果原生代码需要抛出异常或确定 `napi_value` 是否是 JavaScript `Error` 对象的实例，还可以使用以下实用函数：[`napi_throw_error`][]、[`napi_throw_type_error`][]、[`napi_throw_range_error`][]、[`node_api_throw_syntax_error`][] 和 [`napi_is_error`][]。

如果原生代码需要创建 `Error` 对象，还可以使用以下实用函数：[`napi_create_error`][]、[`napi_create_type_error`][]、[`napi_create_range_error`][] 和 [`node_api_create_syntax_error`][]，其中 result 是引用新创建的 JavaScript `Error` 对象的 `napi_value`。

Node.js 项目正在向所有内部生成的错误添加错误码。目标是应用程序对所有错误检查使用这些错误码。相关的错误消息将保留，但仅用于日志记录和显示，期望消息可以在不应用 SemVer 的情况下更改。为了在 Node-API 中支持此模型，包括内部功能和模块特定功能（因为这是好的实践），`throw_` 和 `create_` 函数接受一个可选的 code 参数，这是要添加到错误对象的字符串码。如果可选参数是 `NULL`，则没有码与错误关联。如果提供了码，与错误关联的名称也会更新为：

```text
originalName [code]
```

其中 `originalName` 是与错误关联的原始名称，`code` 是提供的码。例如，如果码是 `'ERR_ERROR_1'` 并且正在创建 `TypeError`，名称将是：

```text
TypeError [ERR_ERROR_1]
```

#### `napi_throw`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_throw(napi_env env, napi_value error);
```

* `[in] env`: 调用 API 的环境。
* `[in] error`: 要抛出的 JavaScript 值。

如果 API 成功则返回 `napi_ok`。

此 API 抛出提供的 JavaScript 值。

#### `napi_throw_error`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_throw_error(napi_env env,
                                         const char* code,
                                         const char* msg);
```

* `[in] env`: 调用 API 的环境。
* `[in] code`: 要在错误上设置的可选错误码。
* `[in] msg`: 与错误关联的文本的 C 字符串。

如果 API 成功则返回 `napi_ok`。

此 API 抛出一个带有提供文本的 JavaScript `Error`。

#### `napi_throw_type_error`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_throw_type_error(napi_env env,
                                              const char* code,
                                              const char* msg);
```

* `[in] env`: 调用 API 的环境。
* `[in] code`: 要在错误上设置的可选错误码。
* `[in] msg`: 与错误关联的文本的 C 字符串。

如果 API 成功则返回 `napi_ok`。

此 API 抛出一个带有提供文本的 JavaScript `TypeError`。

#### `napi_throw_range_error`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_throw_range_error(napi_env env,
                                               const char* code,
                                               const char* msg);
```

* `[in] env`: 调用 API 的环境。
* `[in] code`: 要在错误上设置的可选错误码。
* `[in] msg`: 与错误关联的文本的 C 字符串。

如果 API 成功则返回 `napi_ok`。

此 API 抛出一个带有提供文本的 JavaScript `RangeError`。

#### `node_api_throw_syntax_error`

<!-- YAML
added:
  - v17.2.0
  - v16.14.0
napiVersion: 9
-->

```c
NAPI_EXTERN napi_status node_api_throw_syntax_error(napi_env env,
                                                    const char* code,
                                                    const char* msg);
```

* `[in] env`: 调用 API 的环境。
* `[in] code`: 要在错误上设置的可选错误码。
* `[in] msg`: 与错误关联的文本的 C 字符串。

如果 API 成功则返回 `napi_ok`。

此 API 抛出一个带有提供文本的 JavaScript `SyntaxError`。

#### `napi_is_error`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_is_error(napi_env env,
                                      napi_value value,
                                      bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要检查的 `napi_value`。
* `[out] result`: 布尔值，如果 `napi_value` 表示错误对象则设置为 true，否则为 false。

如果 API 成功则返回 `napi_ok`。

此 API 查询 `napi_value` 以检查它是否表示错误对象。

#### `napi_create_error`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_create_error(napi_env env,
                                          napi_value code,
                                          napi_value msg,
                                          napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] code`: 可选的 `napi_value`，包含要与错误关联的错误码字符串。
* `[in] msg`: 引用要用作 `Error` 消息的 JavaScript `string` 的 `napi_value`。
* `[out] result`: 表示创建的错误的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回一个带有提供文本的 JavaScript `Error`。

#### `napi_create_type_error`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_create_type_error(napi_env env,
                                               napi_value code,
                                               napi_value msg,
                                               napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] code`: 可选的 `napi_value`，包含要与错误关联的错误码字符串。
* `[in] msg`: 引用要用作 `Error` 消息的 JavaScript `string` 的 `napi_value`。
* `[out] result`: 表示创建的错误的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回一个带有提供文本的 JavaScript `TypeError`。

#### `napi_create_range_error`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_create_range_error(napi_env env,
                                                napi_value code,
                                                napi_value msg,
                                                napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] code`: 可选的 `napi_value`，包含要与错误关联的错误码字符串。
* `[in] msg`: 引用要用作 `Error` 消息的 JavaScript `string` 的 `napi_value`。
* `[out] result`: 表示创建的错误的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回一个带有提供文本的 JavaScript `RangeError`。

#### `node_api_create_syntax_error`

<!-- YAML
added:
  - v17.2.0
  - v16.14.0
napiVersion: 9
-->

```c
NAPI_EXTERN napi_status node_api_create_syntax_error(napi_env env,
                                                     napi_value code,
                                                     napi_value msg,
                                                     napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] code`: 可选的 `napi_value`，包含要与错误关联的错误码字符串。
* `[in] msg`: 引用要用作 `Error` 消息的 JavaScript `string` 的 `napi_value`。
* `[out] result`: 表示创建的错误的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回一个带有提供文本的 JavaScript `SyntaxError`。

#### `napi_get_and_clear_last_exception`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_and_clear_last_exception(napi_env env,
                                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[out] result`: 如果有挂起的异常则为异常，否则为 `NULL`。

如果 API 成功则返回 `napi_ok`。

即使有挂起的 JavaScript 异常，也可以调用此 API。

#### `napi_is_exception_pending`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_is_exception_pending(napi_env env, bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[out] result`: 布尔值，如果有挂起的异常则设置为 true。

如果 API 成功则返回 `napi_ok`。

即使有挂起的 JavaScript 异常，也可以调用此 API。

#### `napi_fatal_exception`

<!-- YAML
added: v9.10.0
napiVersion: 3
-->

```c
napi_status napi_fatal_exception(napi_env env, napi_value err);
```

* `[in] env`: 调用 API 的环境。
* `[in] err`: 传递给 `'uncaughtException'` 的错误。

在 JavaScript 中触发 `'uncaughtException'`。如果异步回调抛出异常且无法恢复，这很有用。

### 致命错误

如果原生插件中发生不可恢复的错误，可以抛出致命错误以立即终止进程。

#### `napi_fatal_error`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
NAPI_NO_RETURN void napi_fatal_error(const char* location,
                                     size_t location_len,
                                     const char* message,
                                     size_t message_len);
```

* `[in] location`: 发生错误的可选位置。
* `[in] location_len`: 位置的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[in] message`: 与错误关联的消息。
* `[in] message_len`: 消息的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。

函数调用不返回，进程将被终止。

即使有挂起的 JavaScript 异常，也可以调用此 API。

## 对象生命周期管理

在进行 Node-API 调用时，可能会返回底层 VM 堆中对象的句柄作为 `napi_values`。这些句柄必须保持对象"存活"，直到原生代码不再需要它们，否则对象可能在原生代码使用完之前被回收。

当返回对象句柄时，它们与一个"作用域"关联。默认作用域的寿命与原生方法调用的寿命绑定。结果是，默认情况下，句柄保持有效，并且与这些句柄关联的对象将在原生方法调用的寿命期间保持存活。

然而，在许多情况下，有必要使句柄的寿命比原生方法更短或更长。以下章节描述了可用于更改句柄寿命的 Node-API 函数。

### 使句柄寿命短于原生方法

通常需要使句柄的寿命比原生方法更短。例如，考虑一个原生方法，它有一个循环，遍历大数组中的元素：

```c
for (int i = 0; i < 1000000; i++) {
  napi_value result;
  napi_status status = napi_get_element(env, object, i, &result);
  if (status != napi_ok) {
    break;
  }
  // do something with element
}
```

这将导致创建大量句柄，消耗大量资源。此外，即使原生代码只能使用最近的句柄，所有关联的对象也将保持存活，因为它们都共享相同的作用域。

为了处理这种情况，Node-API 提供了建立新"作用域"的能力，新创建的句柄将与该作用域关联。一旦这些句柄不再需要，可以"关闭"作用域，并且任何与该作用域关联的句柄都将无效。可用于打开/关闭作用域的方法是 [`napi_open_handle_scope`][] 和 [`napi_close_handle_scope`][]。

Node-API 仅支持单个嵌套的作用域层次结构。在任何时候只有一个活动作用域，所有新句柄将在该作用域活动时与其关联。作用域必须按照与打开相反的顺序关闭。此外，在原生方法内创建的所有作用域必须在从该方法返回之前关闭。

以前面的例子为例，添加对 [`napi_open_handle_scope`][] 和 [`napi_close_handle_scope`][] 的调用将确保在循环执行期间最多只有一个句柄有效：

```c
for (int i = 0; i < 1000000; i++) {
  napi_handle_scope scope;
  napi_status status = napi_open_handle_scope(env, &scope);
  if (status != napi_ok) {
    break;
  }
  napi_value result;
  status = napi_get_element(env, object, i, &result);
  if (status != napi_ok) {
    break;
  }
  // do something with element
  status = napi_close_handle_scope(env, scope);
  if (status != napi_ok) {
    break;
  }
}
```

在嵌套作用域时，有时需要使内部作用域的句柄寿命超过该作用域的寿命。Node-API 支持"可逃脱作用域"以支持这种情况。可逃脱作用域允许一个句柄被"提升"，以便它"逃脱"当前作用域，并且句柄的寿命从当前作用域变为外部作用域。

可用于打开/关闭可逃脱作用域的方法是 [`napi_open_escapable_handle_scope`][] 和 [`napi_close_escapable_handle_scope`][]。

提升句柄的请求通过 [`napi_escape_handle`][] 发出，该函数只能调用一次。

#### `napi_open_handle_scope`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_open_handle_scope(napi_env env,
                                               napi_handle_scope* result);
```

* `[in] env`: 调用 Node-API 的环境。
* `[out] result`: 表示新作用域的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 打开一个新作用域。

#### `napi_close_handle_scope`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_close_handle_scope(napi_env env,
                                                napi_handle_scope scope);
```

* `[in] env`: 调用 Node-API 的环境。
* `[in] scope`: 要关闭的作用域的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 关闭传入的作用域。作用域必须按照与创建相反的顺序关闭。

即使有挂起的 JavaScript 异常，也可以调用此 API。

#### `napi_open_escapable_handle_scope`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status
    napi_open_escapable_handle_scope(napi_env env,
                                     napi_handle_scope* result);
```

* `[in] env`: 调用 Node-API 的环境。
* `[out] result`: 表示新作用域的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 打开一个新作用域，可以从该作用域提升一个对象到外部作用域。

#### `napi_close_escapable_handle_scope`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status
    napi_close_escapable_handle_scope(napi_env env,
                                      napi_handle_scope scope);
```

* `[in] env`: 调用 Node-API 的环境。
* `[in] scope`: 要关闭的作用域的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 关闭传入的作用域。作用域必须按照与创建相反的顺序关闭。

即使有挂起的 JavaScript 异常，也可以调用此 API。

#### `napi_escape_handle`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_escape_handle(napi_env env,
                               napi_escapable_handle_scope scope,
                               napi_value escapee,
                               napi_value* result);
```

* `[in] env`: 调用 Node-API 的环境。
* `[in] scope`: 表示当前作用域的 `napi_value`。
* `[in] escapee`: 表示要逃脱的 JavaScript `Object` 的 `napi_value`。
* `[out] result`: 表示外部作用域中逃脱的 `Object` 的句柄的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 将 JavaScript 对象的句柄提升，使其在外部作用域的寿命期间有效。每个作用域只能调用一次。如果调用超过一次，将返回错误。

即使有挂起的 JavaScript 异常，也可以调用此 API。

### 对寿命比原生方法更长的值的引用

在某些情况下，插件需要能够创建和引用寿命比单个原生方法调用更长的值。例如，要创建一个构造函数，然后在创建实例的请求中 later 使用该构造函数，必须能够跨多个不同的实例创建请求引用构造函数对象。这对于作为 `napi_value` 返回的普通句柄来说是不可能的，如前一节所述。普通句柄的寿命由作用域管理，并且所有作用域必须在原生方法结束之前关闭。

Node-API 提供了创建对值的持久引用的方法。目前 Node-API 只允许为有限的值类型创建引用，包括对象、外部、函数和符号。

每个引用都有一个关联的计数值，为 0 或更高，该值决定引用是否将保持相应的值存活。计数值为 0 的引用不会阻止值被回收。对象（对象、函数、外部）和符号类型的值成为"弱"引用，并且可以在它们未被回收时仍然被访问。任何大于 0 的计数值将阻止值被回收。

符号值有不同的类型。真正的弱引用行为仅支持通过 `napi_create_symbol` 函数或 JavaScript `Symbol()` 构造函数调用创建的本地符号。通过 `node_api_symbol_for` 函数或 JavaScript `Symbol.for()` 函数调用创建的全局注册符号始终保持强引用，因为垃圾回收器不会回收它们。对于众所周知的符号（如 `Symbol.iterator`）也是如此。它们也永远不会被垃圾回收器回收。

引用可以以初始引用计数创建。然后可以通过 [`napi_reference_ref`][] 和 [`napi_reference_unref`][] 修改计数。如果对象在引用的计数为 0 时被回收，则所有后续获取与引用关联的对象的调用（[`napi_get_reference_value`][]）将为返回的 `napi_value` 返回 `NULL`。尝试为已回收对象的引用调用 [`napi_reference_ref`][] 将导致错误。

引用在不再需要时必须被删除。当引用被删除时，它将不再阻止相应的对象被回收。未能删除持久引用会导致"内存泄漏"，包括持久引用的原生内存和堆上相应的对象被永久保留。

可以有多个持久引用引用同一个对象，每个引用将根据其各自的计数决定是否保持对象存活。对同一对象的多个持久引用可能导致意外地保持原生内存存活。持久引用的原生结构必须保持存活，直到被引用对象的终结器执行。如果为同一对象创建了新的持久引用，该对象的终结器将不会运行，并且较早持久引用指向的原生内存将不会被释放。可以通过在可能的情况下调用 `napi_delete_reference` 和 `napi_reference_unref` 来避免这种情况。

**变更历史：**

* 版本 10（`NAPI_VERSION` 定义为 `10` 或更高）：

  可以为所有值类型创建引用。新的支持的值类型不支持弱引用语义，并且当引用计数变为 0 时，这些类型的值被释放，无法再从引用访问。

#### `napi_create_reference`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_create_reference(napi_env env,
                                              napi_value value,
                                              uint32_t initial_refcount,
                                              napi_ref* result);
```

* `[in] env`: 调用 Node-API 的环境。
* `[in] value`: 要为其创建引用的 `napi_value`。
* `[in] initial_refcount`: 新引用的初始引用计数。
* `[out] result`: 指向新引用的 `napi_ref`。

如果 API 成功则返回 `napi_ok`。

此 API 为传入的值创建一个具有指定引用计数的新引用。

#### `napi_delete_reference`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_delete_reference(napi_env env, napi_ref ref);
```

* `[in] env`: 调用 Node-API 的环境。
* `[in] ref`: 要删除的 `napi_ref`。

如果 API 成功则返回 `napi_ok`。

此 API 删除传入的引用。

即使有挂起的 JavaScript 异常，也可以调用此 API。

#### `napi_reference_ref`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_reference_ref(napi_env env,
                                           napi_ref ref,
                                           uint32_t* result);
```

* `[in] env`: 调用 Node-API 的环境。
* `[in] ref`: 要增加引用计数的 `napi_ref`。
* `[out] result`: 新的引用计数。

如果 API 成功则返回 `napi_ok`。

此 API 增加传入引用的引用计数并返回结果引用计数。

#### `napi_reference_unref`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_reference_unref(napi_env env,
                                             napi_ref ref,
                                             uint32_t* result);
```

* `[in] env`: 调用 Node-API 的环境。
* `[in] ref`: 要减少引用计数的 `napi_ref`。
* `[out] result`: 新的引用计数。

如果 API 成功则返回 `napi_ok`。

此 API 减少传入引用的引用计数并返回结果引用计数。

#### `napi_get_reference_value`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_get_reference_value(napi_env env,
                                                 napi_ref ref,
                                                 napi_value* result);
```

* `[in] env`: 调用 Node-API 的环境。
* `[in] ref`: 请求其对应值的 `napi_ref`。
* `[out] result`: 由 `napi_ref` 引用的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

如果仍然有效，此 API 返回表示与 `napi_ref` 关联的 JavaScript 值的 `napi_value`。否则，结果将为 `NULL`。

### 当前 Node.js 环境退出时的清理

虽然 Node.js 进程通常在退出时释放所有资源，但 Node.js 的嵌入器或未来的 Worker 支持可能要求插件注册清理钩子，这些钩子将在当前 Node.js 环境退出时运行。

Node-API 提供了注册和注销此类回调的函数。当这些回调运行时，插件持有的所有资源都应该被释放。

#### `napi_add_env_cleanup_hook`

<!-- YAML
added: v10.2.0
napiVersion: 3
-->

```c
NODE_EXTERN napi_status napi_add_env_cleanup_hook(node_api_basic_env env,
                                                  napi_cleanup_hook fun,
                                                  void* arg);
```

注册 `fun` 为一个函数，该函数将在当前 Node.js 环境退出时使用 `arg` 参数运行。

一个函数可以安全地使用不同的 `arg` 值多次指定。在这种情况下，它也将被调用多次。多次提供相同的 `fun` 和 `arg` 值是不允许的，并将导致进程中止。

钩子将以相反的顺序调用，即最后添加的钩子将首先被调用。

可以通过使用 [`napi_remove_env_cleanup_hook`][] 来移除此钩子。通常，这发生在为此钩子添加的资源正在被拆除时。

对于异步清理，可以使用 [`napi_add_async_cleanup_hook`][]。

#### `napi_remove_env_cleanup_hook`

<!-- YAML
added: v10.2.0
napiVersion: 3
-->

```c
NAPI_EXTERN napi_status napi_remove_env_cleanup_hook(node_api_basic_env env,
                                                     void (*fun)(void* arg),
                                                     void* arg);
```

注销 `fun` 作为一个函数，该函数将在当前 Node.js 环境退出时使用 `arg` 参数运行。参数和函数值都需要精确匹配。

该函数必须最初是通过 `napi_add_env_cleanup_hook` 注册的，否则进程将中止。

#### `napi_add_async_cleanup_hook`

<!-- YAML
added:
  - v14.8.0
  - v12.19.0
napiVersion: 8
changes:
  - version:
    - v14.10.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34819
    description: Changed signature of the `hook` callback.
-->

```c
NAPI_EXTERN napi_status napi_add_async_cleanup_hook(
    node_api_basic_env env,
    napi_async_cleanup_hook hook,
    void* arg,
    napi_async_cleanup_hook_handle* remove_handle);
```

* `[in] env`: 调用 API 的环境。
* `[in] hook`: 在环境拆除时要调用的函数指针。
* `[in] arg`: 在调用 `hook` 时要传递的指针。
* `[out] remove_handle`: 引用异步清理钩子的可选句柄。

注册 `hook`，它是一个类型为 [`napi_async_cleanup_hook`][] 的函数，将在当前 Node.js 环境退出时使用 `remove_handle` 和 `arg` 参数运行。

与 [`napi_add_env_cleanup_hook`][] 不同，钩子允许是异步的。

否则，行为通常与 [`napi_add_env_cleanup_hook`][] 匹配。

如果 `remove_handle` 不是 `NULL`，将在其中存储一个不透明值，无论钩子是否已被调用，稍后都必须传递给 [`napi_remove_async_cleanup_hook`][]。通常，这发生在为此钩子添加的资源正在被拆除时。

#### `napi_remove_async_cleanup_hook`

<!-- YAML
added:
  - v14.8.0
  - v12.19.0
changes:
  - version:
    - v14.10.0
    - v12.19.0
    pr-url: https://github.com/nodejs/node/pull/34819
    description: Removed `env` parameter.
-->

```c
NAPI_EXTERN napi_status napi_remove_async_cleanup_hook(
    napi_async_cleanup_hook_handle remove_handle);
```

* `[in] remove_handle`: 由 [`napi_add_async_cleanup_hook`][] 创建的异步清理钩子的句柄。

注销与 `remove_handle` 对应的清理钩子。这将阻止钩子被执行，除非它已经开始执行。必须对从 [`napi_add_async_cleanup_hook`][] 获得的任何 `napi_async_cleanup_hook_handle` 值调用此函数。

### Node.js 环境退出时的终结化

Node.js 环境可能在允许 JavaScript 执行的情况下尽快在任意时间被拆除，例如在 [`worker.terminate()`][] 的请求下。当环境被拆除时，JavaScript 对象、线程安全函数和环境实例数据的已注册 `napi_finalize` 回调会立即且独立地调用。

`napi_finalize` 回调的调用安排在手动注册的清理钩子之后。为了确保在环境关闭期间插件终结化的正确顺序，以避免在 `napi_finalize` 回调中使用已释放的内存，插件应使用 `napi_add_env_cleanup_hook` 和 `napi_add_async_cleanup_hook` 注册一个清理钩子，以手动以正确的顺序释放分配的资源。

## 模块注册

Node-API 模块的注册方式与其他模块类似，只是不使用 `NODE_MODULE` 宏，而是使用以下方式：

```c
NAPI_MODULE(NODE_GYP_MODULE_NAME, Init)
```

下一个区别是 `Init` 方法的签名。对于 Node-API 模块，它如下：

```c
napi_value Init(napi_env env, napi_value exports);
```

从 `Init` 返回的值被视为模块的 `exports` 对象。`Init` 方法通过 `exports` 参数传递一个空对象作为便利。如果 `Init` 返回 `NULL`，则作为 `exports` 传递的参数由模块导出。Node-API 模块不能修改 `module` 对象，但可以指定任何内容作为模块的 `exports` 属性。

要将方法 `hello` 添加为一个函数，以便它可以作为插件提供的方法调用：

```c
napi_value Init(napi_env env, napi_value exports) {
  napi_status status;
  napi_property_descriptor desc = {
    "hello",
    NULL,
    Method,
    NULL,
    NULL,
    NULL,
    napi_writable | napi_enumerable | napi_configurable,
    NULL
  };
  status = napi_define_properties(env, exports, 1, &desc);
  if (status != napi_ok) return NULL;
  return exports;
}
```

要设置一个函数作为插件的 `require()` 的返回值：

```c
napi_value Init(napi_env env, napi_value exports) {
  napi_value method;
  napi_status status;
  status = napi_create_function(env, "exports", NAPI_AUTO_LENGTH, Method, NULL, &method);
  if (status != napi_ok) return NULL;
  return method;
}
```

要定义一个类以便可以创建新实例（通常与[对象包装][]一起使用）：

```c
// 注意：部分示例，未包含所有引用代码
napi_value Init(napi_env env, napi_value exports) {
  napi_status status;
  napi_property_descriptor properties[] = {
    { "value", NULL, NULL, GetValue, SetValue, NULL, napi_writable | napi_configurable, NULL },
    DECLARE_NAPI_METHOD("plusOne", PlusOne),
    DECLARE_NAPI_METHOD("multiply", Multiply),
  };

  napi_value cons;
  status =
      napi_define_class(env, "MyObject", New, NULL, 3, properties, &cons);
  if (status != napi_ok) return NULL;

  status = napi_create_reference(env, cons, 1, &constructor);
  if (status != napi_ok) return NULL;

  status = napi_set_named_property(env, exports, "MyObject", cons);
  if (status != napi_ok) return NULL;

  return exports;
}
```

你也可以使用 `NAPI_MODULE_INIT` 宏，它作为 `NAPI_MODULE` 和定义 `Init` 函数的简写：

```c
NAPI_MODULE_INIT(/* napi_env env, napi_value exports */) {
  napi_value answer;
  napi_status result;

  status = napi_create_int64(env, 42, &answer);
  if (status != napi_ok) return NULL;

  status = napi_set_named_property(env, exports, "answer", answer);
  if (status != napi_ok) return NULL;

  return exports;
}
```

参数 `env` 和 `exports` 在宏调用后的函数体中可用。

所有 Node-API 插件都是上下文感知的，意味着它们可能被加载多次。声明这样的模块时有一些设计考虑。[上下文感知插件][]文档提供了更多细节。

变量 `env` 和 `exports` 将在宏调用后的函数体内可用。

有关在对象上设置属性的更多详细信息，请参阅[处理 JavaScript 属性][]部分。

有关构建插件模块的更多详细信息，请参阅现有 API。

## 处理 JavaScript 值

Node-API 暴露了一组 API 来创建所有类型的 JavaScript 值。其中一些类型在 [ECMAScript 语言规范][]的[语言类型部分][]中有文档记录。

基本上，这些 API 用于执行以下操作之一：

1. 创建一个新的 JavaScript 对象
2. 从基本 C 类型转换为 Node-API 值
3. 从 Node-API 值转换为基本 C 类型
4. 获取全局实例，包括 `undefined` 和 `null`

Node-API 值由类型 `napi_value` 表示。任何需要 JavaScript 值的 Node-API 调用都接受一个 `napi_value`。在某些情况下，API 会提前检查 `napi_value` 的类型。然而，为了更好的性能，调用者最好确保相关的 `napi_value` 是 API 期望的 JavaScript 类型。

### 枚举类型

#### `napi_key_collection_mode`

<!-- YAML
added:
 - v13.7.0
 - v12.17.0
 - v10.20.0
napiVersion: 6
-->

```c
typedef enum {
  napi_key_include_prototypes,
  napi_key_own_only
} napi_key_collection_mode;
```

描述 `Keys/Properties` 过滤器枚举：

`napi_key_collection_mode` 限制了收集属性的范围。

`napi_key_own_only` 将收集的属性限制为仅给定对象。`napi_key_include_prototypes` 将包括对象原型链上的所有键。

#### `napi_key_filter`

<!-- YAML
added:
 - v13.7.0
 - v12.17.0
 - v10.20.0
napiVersion: 6
-->

```c
typedef enum {
  napi_key_all_properties = 0,
  napi_key_writable = 1,
  napi_key_enumerable = 1 << 1,
  napi_key_configurable = 1 << 2,
  napi_key_skip_strings = 1 << 3,
  napi_key_skip_symbols = 1 << 4
} napi_key_filter;
```

属性过滤器位标志。这与位运算符一起使用以构建复合过滤器。

#### `napi_key_conversion`

<!-- YAML
added:
 - v13.7.0
 - v12.17.0
 - v10.20.0
napiVersion: 6
-->

```c
typedef enum {
  napi_key_keep_numbers,
  napi_key_numbers_to_strings
} napi_key_conversion;
```

`napi_key_numbers_to_strings` 将整数索引转换为字符串。`napi_key_keep_numbers` 将为整数索引返回数字。

#### `napi_valuetype`

```c
typedef enum {
  // ES6 类型（对应于 typeof）
  napi_undefined,
  napi_null,
  napi_boolean,
  napi_number,
  napi_string,
  napi_symbol,
  napi_object,
  napi_function,
  napi_external,
  napi_bigint,
} napi_valuetype;
```

描述 `napi_value` 的类型。这通常对应于 ECMAScript 语言规范的[语言类型部分][]中描述的类型。除了该部分中的类型，`napi_valuetype` 还可以表示具有外部数据的 `Function` 和 `Object`。

类型为 `napi_external` 的 JavaScript 值在 JavaScript 中显示为普通对象，因此无法在其上设置任何属性，并且没有原型。

#### `napi_typedarray_type`

```c
typedef enum {
  napi_int8_array,
  napi_uint8_array,
  napi_uint8_clamped_array,
  napi_int16_array,
  napi_uint16_array,
  napi_int32_array,
  napi_uint32_array,
  napi_float32_array,
  napi_float64_array,
  napi_bigint64_array,
  napi_biguint64_array,
} napi_typedarray_type;
```

这表示 `TypedArray` 的底层二进制标量数据类型。此枚举的元素对应于 [ECMAScript 语言规范][]的[TypedArray 对象部分][]。

### 对象创建函数

#### `napi_create_array`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_array(napi_env env, napi_value* result)
```

* `[in] env`: 调用 Node-API 的环境。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回一个对应于 JavaScript `Array` 类型的 Node-API 值。JavaScript 数组在 ECMAScript 语言规范的[数组对象部分][]中描述。

#### `napi_create_array_with_length`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_array_with_length(napi_env env,
                                          size_t length,
                                          napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] length`: `Array` 的初始长度。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回一个对应于 JavaScript `Array` 类型的 Node-API 值。`Array` 的 length 属性设置为传入的 length 参数。然而，在创建数组时，VM 不保证底层缓冲区是预分配的。该行为留给底层 VM 实现。如果缓冲区必须是可以通过 C 直接读取和/或写入的连续内存块，请考虑使用 [`napi_create_external_arraybuffer`][]。

JavaScript 数组在 ECMAScript 语言规范的[数组对象部分][]中描述。

#### `napi_create_arraybuffer`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_arraybuffer(napi_env env,
                                    size_t byte_length,
                                    void** data,
                                    napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] length`: 要创建的数组缓冲区的字节长度。
* `[out] data`: 指向 `ArrayBuffer` 的底层字节缓冲区的指针。`data` 可以通过传递 `NULL` 选择性地忽略。
* `[out] result`: 表示 JavaScript `ArrayBuffer` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回一个对应于 JavaScript `ArrayBuffer` 的 Node-API 值。`ArrayBuffer` 用于表示固定长度的二进制数据缓冲区。它们通常用作 `TypedArray` 对象的后备缓冲区。分配的 `ArrayBuffer` 将有一个底层字节缓冲区，其大小由传入的 `length` 参数确定。底层缓冲区可以选择性地返回给调用者，以防调用者想要直接操作缓冲区。此缓冲区只能从原生代码直接写入。要从 JavaScript 写入此缓冲区，需要创建类型化数组或 `DataView` 对象。

JavaScript `ArrayBuffer` 对象在 ECMAScript 语言规范的[ArrayBuffer 对象部分][]中描述。

#### `napi_create_buffer`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_buffer(napi_env env,
                               size_t size,
                               void** data,
                               napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] size`: 底层缓冲区的大小（字节）。
* `[out] data`: 指向底层缓冲区的原始指针。`data` 可以通过传递 `NULL` 选择性地忽略。
* `[out] result`: 表示 `node::Buffer` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 分配一个 `node::Buffer` 对象。虽然这仍然是一个完全支持的数据结构，但在大多数情况下使用 `TypedArray` 就足够了。

#### `napi_create_buffer_copy`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_buffer_copy(napi_env env,
                                    size_t length,
                                    const void* data,
                                    void** result_data,
                                    napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] size`: 输入缓冲区的大小（字节）（应与新缓冲区的大小相同）。
* `[in] data`: 指向要复制的底层缓冲区的原始指针。
* `[out] result_data`: 指向新 `Buffer` 的底层数据缓冲区的指针。`result_data` 可以通过传递 `NULL` 选择性地忽略。
* `[out] result`: 表示 `node::Buffer` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 分配一个 `node::Buffer` 对象，并使用从传入缓冲区复制的数据初始化它。虽然这仍然是一个完全支持的数据结构，但在大多数情况下使用 `TypedArray` 就足够了。

#### `napi_create_date`

<!-- YAML
added:
 - v11.11.0
 - v10.17.0
napiVersion: 5
-->

```c
napi_status napi_create_date(napi_env env,
                             double time,
                             napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] time`: 自 1970 年 1 月 1 日 UTC 以来的 ECMAScript 时间值（毫秒）。
* `[out] result`: 表示 JavaScript `Date` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 不观察闰秒；它们被忽略，因为 ECMAScript 与 POSIX 时间规范对齐。

此 API 分配一个 JavaScript `Date` 对象。

JavaScript `Date` 对象在 ECMAScript 语言规范的[日期对象部分][]中描述。

#### `napi_create_external`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_external(napi_env env,
                                 void* data,
                                 napi_finalize finalize_cb,
                                 void* finalize_hint,
                                 napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] data`: 指向外部数据的原始指针。
* `[in] finalize_cb`: 在外部值被回收时调用的可选回调。[`napi_finalize`][] 提供了更多细节。
* `[in] finalize_hint`: 在回收期间传递给最终化回调的可选提示。
* `[out] result`: 表示外部值的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 分配一个具有附加外部数据的 JavaScript 值。这用于通过 JavaScript 代码传递外部数据，以便稍后可以通过原生代码使用 [`napi_get_value_external`][] 检索。

该 API 添加了一个 `napi_finalize` 回调，该回调将在刚刚创建的 JavaScript 对象被垃圾回收时调用。

创建的值不是对象，因此不支持额外的属性。它被视为一个不同的值类型：使用外部值调用 `napi_typeof()` 会产生 `napi_external`。

#### `napi_create_external_arraybuffer`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status
napi_create_external_arraybuffer(napi_env env,
                                 void* external_data,
                                 size_t byte_length,
                                 napi_finalize finalize_cb,
                                 void* finalize_hint,
                                 napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] external_data`: 指向 `ArrayBuffer` 的底层字节缓冲区的指针。
* `[in] byte_length`: 底层缓冲区的长度（字节）。
* `[in] finalize_cb`: 在 `ArrayBuffer` 被回收时调用的可选回调。[`napi_finalize`][] 提供了更多细节。
* `[in] finalize_hint`: 在回收期间传递给最终化回调的可选提示。
* `[out] result`: 表示 JavaScript `ArrayBuffer` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

**除了 Node.js 之外的一些运行时已经放弃了对外部缓冲区的支持**。在 Node.js 以外的运行时上，此方法可能返回 `napi_no_external_buffers_allowed` 以指示不支持外部缓冲区。其中一个运行时是 Electron，如 [electron/issues/35801](https://github.com/electron/electron/issues/35801) 中所述。

为了保持与所有运行时的最广泛兼容性，你可以在包含 node-api 头文件之前在插件中定义 `NODE_API_NO_EXTERNAL_BUFFERS_ALLOWED`。这样做将隐藏创建外部缓冲区的 2 个函数。这将确保如果你意外使用这些方法之一，会发生编译错误。

此 API 返回一个对应于 JavaScript `ArrayBuffer` 的 Node-API 值。`ArrayBuffer` 的底层字节缓冲区是外部分配和管理的。调用者必须确保字节缓冲区保持有效，直到最终化回调被调用。

该 API 添加了一个 `napi_finalize` 回调，该回调将在刚刚创建的 JavaScript 对象被垃圾回收时调用。

JavaScript `ArrayBuffer` 在 ECMAScript 语言规范的[ArrayBuffer 对象部分][]中描述。

#### `napi_create_external_buffer`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_external_buffer(napi_env env,
                                        size_t length,
                                        void* data,
                                        napi_finalize finalize_cb,
                                        void* finalize_hint,
                                        napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] length`: 输入缓冲区的大小（字节）（应与新缓冲区的大小相同）。
* `[in] data`: 指向要暴露给 JavaScript 的底层缓冲区的原始指针。
* `[in] finalize_cb`: 在 `ArrayBuffer` 被回收时调用的可选回调。[`napi_finalize`][] 提供了更多细节。
* `[in] finalize_hint`: 在回收期间传递给最终化回调的可选提示。
* `[out] result`: 表示 `node::Buffer` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

**除了 Node.js 之外的一些运行时已经放弃了对外部缓冲区的支持**。在 Node.js 以外的运行时上，此方法可能返回 `napi_no_external_buffers_allowed` 以指示不支持外部缓冲区。其中一个运行时是 Electron，如 [electron/issues/35801](https://github.com/electron/electron/issues/35801) 中所述。

为了保持与所有运行时的最广泛兼容性，你可以在包含 node-api 头文件之前在插件中定义 `NODE_API_NO_EXTERNAL_BUFFERS_ALLOWED`。这样做将隐藏创建外部缓冲区的 2 个函数。这将确保如果你意外使用这些方法之一，会发生编译错误。

此 API 分配一个 `node::Buffer` 对象，并使用由传入缓冲区支持的数据初始化它。虽然这仍然是一个完全支持的数据结构，但在大多数情况下使用 `TypedArray` 就足够了。

该 API 添加了一个 `napi_finalize` 回调，该回调将在刚刚创建的 JavaScript 对象被垃圾回收时调用。

对于 Node.js >=4，`Buffer` 是 `Uint8Array`。

#### `napi_create_object`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_object(napi_env env, napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[out] result`: 表示 JavaScript `Object` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 分配一个默认的 JavaScript `Object`。它相当于在 JavaScript 中执行 `new Object()`。

JavaScript `Object` 类型在 ECMAScript 语言规范的[对象类型部分][]中描述。

#### `napi_create_symbol`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_symbol(napi_env env,
                               napi_value description,
                               napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] description`: 可选的 `napi_value`，引用一个 JavaScript `string`，用作符号的描述。
* `[out] result`: 表示 JavaScript `symbol` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 从 UTF8 编码的 C 字符串创建一个 JavaScript `symbol` 值。

JavaScript `symbol` 类型在 ECMAScript 语言规范的[符号类型部分][]中描述。

#### `node_api_symbol_for`

<!-- YAML
added:
  - v17.5.0
  - v16.15.0
napiVersion: 9
-->

```c
napi_status node_api_symbol_for(napi_env env,
                                const char* utf8description,
                                size_t length,
                                napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] utf8description`: UTF-8 C 字符串，表示要用作符号描述的文本。
* `[in] length`: 描述字符串的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[out] result`: 表示 JavaScript `symbol` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 在全局注册表中搜索具有给定描述的现有符号。如果符号已存在，则返回，否则将在注册表中创建一个新符号。

JavaScript `symbol` 类型在 ECMAScript 语言规范的[符号类型部分][]中描述。

#### `napi_create_typedarray`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_typedarray(napi_env env,
                                   napi_typedarray_type type,
                                   size_t length,
                                   napi_value arraybuffer,
                                   size_t byte_offset,
                                   napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] type`: `TypedArray` 内元素的标量数据类型。
* `[in] length`: `TypedArray` 中的元素数量。
* `[in] arraybuffer`: 类型化数组底层的 `ArrayBuffer`。
* `[in] byte_offset`: 在 `ArrayBuffer` 中开始投影 `TypedArray` 的字节偏移量。
* `[out] result`: 表示 JavaScript `TypedArray` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 在现有 `ArrayBuffer` 上创建一个 JavaScript `TypedArray` 对象。`TypedArray` 对象提供对底层数据缓冲区的类似数组的视图，其中每个元素具有相同的底层二进制标量数据类型。

要求 `(length * size_of_element) + byte_offset` 应该 <= 传入数组的字节大小。如果不是，将引发 `RangeError` 异常。

JavaScript `TypedArray` 对象在 ECMAScript 语言规范的[TypedArray 对象部分][]中描述。

#### `node_api_create_buffer_from_arraybuffer`

<!-- YAML
added:
  - v23.0.0
  - v22.12.0
napiVersion: 10
-->

```c
napi_status NAPI_CDECL node_api_create_buffer_from_arraybuffer(napi_env env,
                                                              napi_value arraybuffer,
                                                              size_t byte_offset,
                                                              size_t byte_length,
                                                              napi_value* result)
```

* **`[in] env`**: 调用 API 的环境。
* **`[in] arraybuffer`**: 从中创建缓冲区的 `ArrayBuffer`。
* **`[in] byte_offset`**: 在 `ArrayBuffer` 中开始创建缓冲区的字节偏移量。
* **`[in] byte_length`**: 要从 `ArrayBuffer` 创建的缓冲区的长度（字节）。
* **`[out] result`**: 表示创建的 JavaScript `Buffer` 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 从现有 `ArrayBuffer` 创建一个 JavaScript `Buffer` 对象。`Buffer` 对象是一个特定于 Node.js 的类，提供了一种直接在 JavaScript 中处理二进制数据的方式。

字节范围 `[byte_offset, byte_offset + byte_length)` 必须在 `ArrayBuffer` 的边界内。如果 `byte_offset + byte_length` 超过 `ArrayBuffer` 的大小，将引发 `RangeError` 异常。

#### `napi_create_dataview`

<!-- YAML
added: v8.3.0
napiVersion: 1
-->

```c
napi_status napi_create_dataview(napi_env env,
                                 size_t byte_length,
                                 napi_value arraybuffer,
                                 size_t byte_offset,
                                 napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] length`: `DataView` 中的元素数量。
* `[in] arraybuffer`: `DataView` 底层的 `ArrayBuffer`。
* `[in] byte_offset`: 在 `ArrayBuffer` 中开始投影 `DataView` 的字节偏移量。
* `[out] result`: 表示 JavaScript `DataView` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 在现有 `ArrayBuffer` 上创建一个 JavaScript `DataView` 对象。`DataView` 对象提供对底层数据缓冲区的类似数组的视图，但允许 `ArrayBuffer` 中不同大小和类型的项。

要求 `byte_length + byte_offset` 小于或等于传入数组的字节大小。如果不是，将引发 `RangeError` 异常。

JavaScript `DataView` 对象在 ECMAScript 语言规范的[DataView 对象部分][]中描述。

### 从 C 类型转换为 Node-API 的函数

#### `napi_create_int32`

<!-- YAML
added: v8.4.0
napiVersion: 1
-->

```c
napi_status napi_create_int32(napi_env env, int32_t value, napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要在 JavaScript 中表示的整数值。
* `[out] result`: 表示 JavaScript `number` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 用于从 C `int32_t` 类型转换为 JavaScript `number` 类型。

JavaScript `number` 类型在 ECMAScript 语言规范的[数字类型部分][]中描述。

#### `napi_create_uint32`

<!-- YAML
added: v8.4.0
napiVersion: 1
-->

```c
napi_status napi_create_uint32(napi_env env, uint32_t value, napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要在 JavaScript 中表示的无符号整数值。
* `[out] result`: 表示 JavaScript `number` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 用于从 C `uint32_t` 类型转换为 JavaScript `number` 类型。

JavaScript `number` 类型在 ECMAScript 语言规范的[数字类型部分][]中描述。

#### `napi_create_int64`

<!-- YAML
added: v8.4.0
napiVersion: 1
-->

```c
napi_status napi_create_int64(napi_env env, int64_t value, napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要在 JavaScript 中表示的整数值。
* `[out] result`: 表示 JavaScript `number` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 用于从 C `int64_t` 类型转换为 JavaScript `number` 类型。

JavaScript `number` 类型在 ECMAScript 语言规范的[数字类型部分][]中描述。注意 `int64_t` 的完整范围无法在 JavaScript 中以完全精度表示。超出 [`Number.MIN_SAFE_INTEGER`][] `-(2**53 - 1)` - [`Number.MAX_SAFE_INTEGER`][] `(2**53 - 1)` 范围的整数值将丢失精度。

#### `napi_create_double`

<!-- YAML
added: v8.4.0
napiVersion: 1
-->

```c
napi_status napi_create_double(napi_env env, double value, napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要在 JavaScript 中表示的双精度值。
* `[out] result`: 表示 JavaScript `number` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 用于从 C `double` 类型转换为 JavaScript `number` 类型。

JavaScript `number` 类型在 ECMAScript 语言规范的[数字类型部分][]中描述。

#### `napi_create_bigint_int64`

<!-- YAML
added: v10.7.0
napiVersion: 6
-->

```c
napi_status napi_create_bigint_int64(napi_env env,
                                     int64_t value,
                                     napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要在 JavaScript 中表示的整数值。
* `[out] result`: 表示 JavaScript `BigInt` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 将 C `int64_t` 类型转换为 JavaScript `BigInt` 类型。

#### `napi_create_bigint_uint64`

<!-- YAML
added: v10.7.0
napiVersion: 6
-->

```c
napi_status napi_create_bigint_uint64(napi_env env,
                                      uint64_t value,
                                      napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要在 JavaScript 中表示的无符号整数值。
* `[out] result`: 表示 JavaScript `BigInt` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 将 C `uint64_t` 类型转换为 JavaScript `BigInt` 类型。

#### `napi_create_bigint_words`

<!-- YAML
added: v10.7.0
napiVersion: 6
-->

```c
napi_status napi_create_bigint_words(napi_env env,
                                     int sign_bit,
                                     size_t word_count,
                                     const uint64_t* words,
                                     napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] sign_bit`: 确定生成的 `BigInt` 是正还是负。
* `[in] word_count`: `words` 数组的长度。
* `[in] words`: `uint64_t` 小端 64 位字的数组。
* `[out] result`: 表示 JavaScript `BigInt` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 将无符号 64 位字的数组转换为单个 `BigInt` 值。

生成的 `BigInt` 计算为：(–1)<sup>`sign_bit`</sup> (`words[0]` × (2<sup>64</sup>)<sup>0</sup> + `words[1]` × (2<sup>64</sup>)<sup>1</sup> + …)

#### `napi_create_string_latin1`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_string_latin1(napi_env env,
                                      const char* str,
                                      size_t length,
                                      napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] str`: 表示 ISO-8859-1 编码字符串的字符缓冲区。
* `[in] length`: 字符串的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[out] result`: 表示 JavaScript `string` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 从 ISO-8859-1 编码的 C 字符串创建一个 JavaScript `string` 值。原生字符串被复制。

JavaScript `string` 类型在 ECMAScript 语言规范的[字符串类型部分][]中描述。

#### `node_api_create_external_string_latin1`

<!-- YAML
added:
 - v20.4.0
 - v18.18.0
napiVersion: 10
-->

```c
napi_status
node_api_create_external_string_latin1(napi_env env,
                                       char* str,
                                       size_t length,
                                       napi_finalize finalize_callback,
                                       void* finalize_hint,
                                       napi_value* result,
                                       bool* copied);
```

* `[in] env`: 调用 API 的环境。
* `[in] str`: 表示 ISO-8859-1 编码字符串的字符缓冲区。
* `[in] length`: 字符串的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[in] finalize_callback`: 在字符串被回收时调用的函数。该函数将使用以下参数调用：
  * `[in] env`: 插件运行的环境。如果字符串作为工作线程或主 Node.js 实例终止的一部分被回收，此值可能为 null。
  * `[in] data`: 这是作为 `void*` 指针的 `str` 值。
  * `[in] finalize_hint`: 这是给予 API 的 `finalize_hint` 值。[`napi_finalize`][] 提供了更多细节。此参数是可选的。传递 null 值意味着插件在相应的 JavaScript 字符串被回收时不需要收到通知。
* `[in] finalize_hint`: 在回收期间传递给最终化回调的可选提示。
* `[out] result`: 表示 JavaScript `string` 的 `napi_value`。
* `[out] copied`: 字符串是否被复制。如果是，终结器将已经被调用来销毁 `str`。

如果 API 成功则返回 `napi_ok`。

此 API 从 ISO-8859-1 编码的 C 字符串创建一个 JavaScript `string` 值。原生字符串可能不会被复制，因此必须在整个 JavaScript 值的生命周期中存在。

JavaScript `string` 类型在 ECMAScript 语言规范的[字符串类型部分][]中描述。

#### `napi_create_string_utf16`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_string_utf16(napi_env env,
                                     const char16_t* str,
                                     size_t length,
                                     napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] str`: 表示 UTF16-LE 编码字符串的字符缓冲区。
* `[in] length`: 字符串的长度（双字节代码单元），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[out] result`: 表示 JavaScript `string` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 从 UTF16-LE 编码的 C 字符串创建一个 JavaScript `string` 值。原生字符串被复制。

JavaScript `string` 类型在 ECMAScript 语言规范的[字符串类型部分][]中描述。

#### `node_api_create_external_string_utf16`

<!-- YAML
added:
 - v20.4.0
 - v18.18.0
napiVersion: 10
-->

```c
napi_status
node_api_create_external_string_utf16(napi_env env,
                                      char16_t* str,
                                      size_t length,
                                      napi_finalize finalize_callback,
                                      void* finalize_hint,
                                      napi_value* result,
                                      bool* copied);
```

* `[in] env`: 调用 API 的环境。
* `[in] str`: 表示 UTF16-LE 编码字符串的字符缓冲区。
* `[in] length`: 字符串的长度（双字节代码单元），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[in] finalize_callback`: 在字符串被回收时调用的函数。该函数将使用以下参数调用：
  * `[in] env`: 插件运行的环境。如果字符串作为工作线程或主 Node.js 实例终止的一部分被回收，此值可能为 null。
  * `[in] data`: 这是作为 `void*` 指针的 `str` 值。
  * `[in] finalize_hint`: 这是给予 API 的 `finalize_hint` 值。[`napi_finalize`][] 提供了更多细节。此参数是可选的。传递 null 值意味着插件在相应的 JavaScript 字符串被回收时不需要收到通知。
* `[in] finalize_hint`: 在回收期间传递给最终化回调的可选提示。
* `[out] result`: 表示 JavaScript `string` 的 `napi_value`。
* `[out] copied`: 字符串是否被复制。如果是，终结器将已经被调用来销毁 `str`。

如果 API 成功则返回 `napi_ok`。

此 API 从 UTF16-LE 编码的 C 字符串创建一个 JavaScript `string` 值。原生字符串可能不会被复制，因此必须在整个 JavaScript 值的生命周期中存在。

JavaScript `string` 类型在 ECMAScript 语言规范的[字符串类型部分][]中描述。

#### `napi_create_string_utf8`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_create_string_utf8(napi_env env,
                                    const char* str,
                                    size_t length,
                                    napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] str`: 表示 UTF8 编码字符串的字符缓冲区。
* `[in] length`: 字符串的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[out] result`: 表示 JavaScript `string` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 从 UTF8 编码的 C 字符串创建一个 JavaScript `string` 值。原生字符串被复制。

JavaScript `string` 类型在 ECMAScript 语言规范的[字符串类型部分][]中描述。

### 创建优化属性键的函数

许多 JavaScript 引擎（包括 V8）使用内部化字符串作为键来设置和获取属性值。它们通常使用哈希表来创建和查找这些字符串。虽然每个键的创建会增加一些成本，但它通过启用字符串指针的比较而不是整个字符串的比较来提高之后的性能。

如果新的 JavaScript 字符串打算用作属性键，那么对于某些 JavaScript 引擎，使用本节中的函数会更高效。否则，请使用 `napi_create_string_utf8` 或 `node_api_create_external_string_utf8` 系列函数，因为使用属性键创建方法创建/存储字符串可能会有额外的开销。

#### `node_api_create_property_key_latin1`

<!-- YAML
added:
  - v22.9.0
  - v20.18.0
napiVersion: 10
-->

```c
napi_status NAPI_CDECL node_api_create_property_key_latin1(napi_env env,
                                                           const char* str,
                                                           size_t length,
                                                           napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] str`: 表示 ISO-8859-1 编码字符串的字符缓冲区。
* `[in] length`: 字符串的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[out] result`: 表示要用作对象属性键的优化 JavaScript `string` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 从 ISO-8859-1 编码的 C 字符串创建一个优化的 JavaScript `string` 值，用作对象的属性键。原生字符串被复制。与 `napi_create_string_latin1` 相比，后续使用相同 `str` 指针调用此函数可能会从请求的 `napi_value` 创建中受益于加速。

JavaScript `string` 类型在 ECMAScript 语言规范的[字符串类型部分][]中描述。

#### `node_api_create_property_key_utf16`

<!-- YAML
added:
  - v21.7.0
  - v20.12.0
napiVersion: 10
-->

```c
napi_status NAPI_CDECL node_api_create_property_key_utf16(napi_env env,
                                                          const char16_t* str,
                                                          size_t length,
                                                          napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] str`: 表示 UTF16-LE 编码字符串的字符缓冲区。
* `[in] length`: 字符串的长度（双字节代码单元），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[out] result`: 表示要用作对象属性键的优化 JavaScript `string` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 从 UTF16-LE 编码的 C 字符串创建一个优化的 JavaScript `string` 值，用作对象的属性键。原生字符串被复制。

JavaScript `string` 类型在 ECMAScript 语言规范的[字符串类型部分][]中描述。

#### `node_api_create_property_key_utf8`

<!-- YAML
added:
  - v22.9.0
  - v20.18.0
napiVersion: 10
-->

```c
napi_status NAPI_CDECL node_api_create_property_key_utf8(napi_env env,
                                                         const char* str,
                                                         size_t length,
                                                         napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] str`: 表示 UTF8 编码字符串的字符缓冲区。
* `[in] length`: 字符串的长度（双字节代码单元），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[out] result`: 表示要用作对象属性键的优化 JavaScript `string` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 从 UTF8 编码的 C 字符串创建一个优化的 JavaScript `string` 值，用作对象的属性键。原生字符串被复制。

JavaScript `string` 类型在 ECMAScript 语言规范的[字符串类型部分][]中描述。

### 从 Node-API 转换为 C 类型的函数

#### `napi_get_array_length`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_array_length(napi_env env,
                                  napi_value value,
                                  uint32_t* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示正在查询长度的 JavaScript `Array` 的 `napi_value`。
* `[out] result`: 表示数组长度的 `uint32`。

如果 API 成功则返回 `napi_ok`。

此 API 返回数组的长度。

`Array` 长度在 ECMAScript 语言规范的[数组实例长度部分][]中描述。

#### `napi_get_arraybuffer_info`

<!-- YAML
added: v8.0.0
napiVersion: 1
changes:
  - version: v24.9.0
    pr-url: https://github.com/nodejs/node/pull/59071
    description: Added support for `SharedArrayBuffer`.
-->

```c
napi_status napi_get_arraybuffer_info(napi_env env,
                                      napi_value arraybuffer,
                                      void** data,
                                      size_t* byte_length)
```

* `[in] env`: 调用 API 的环境。
* `[in] arraybuffer`: 表示正在查询的 `ArrayBuffer` 或 `SharedArrayBuffer` 的 `napi_value`。
* `[out] data`: `ArrayBuffer` 或 `SharedArrayBuffer` 的底层数据缓冲区。如果长度为 `0`，这可能是 `NULL` 或任何其他指针值。
* `[out] byte_length`: 底层数据缓冲区的长度（字节）。

如果 API 成功则返回 `napi_ok`。

此 API 用于检索 `ArrayBuffer` 或 `SharedArrayBuffer` 的底层数据缓冲区及其长度。

_警告_：使用此 API 时要小心。底层数据缓冲区的生命周期由 `ArrayBuffer` 或 `SharedArrayBuffer` 管理，即使在返回后也是如此。使用此 API 的一个可能安全的方式是与 [`napi_create_reference`][] 结合使用，这可以用于保证对 `ArrayBuffer` 或 `SharedArrayBuffer` 生命周期的控制。在同一个回调中使用返回的数据缓冲区也是安全的，只要没有调用其他可能触发 GC 的 API。

#### `napi_get_buffer_info`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_buffer_info(napi_env env,
                                 napi_value value,
                                 void** data,
                                 size_t* length)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示正在查询的 `node::Buffer` 或 `Uint8Array` 的 `napi_value`。
* `[out] data`: `node::Buffer` 或 `Uint8Array` 的底层数据缓冲区。如果长度为 `0`，这可能是 `NULL` 或任何其他指针值。
* `[out] length`: 底层数据缓冲区的长度（字节）。

如果 API 成功则返回 `napi_ok`。

此方法返回与 [`napi_get_typedarray_info`][] 相同的 `data` 和 `byte_length`。并且 `napi_get_typedarray_info` 也接受 `node::Buffer`（一个 Uint8Array）作为值。

此 API 用于检索 `node::Buffer` 的底层数据缓冲区及其长度。

_警告_：使用此 API 时要小心，因为如果底层数据缓冲区由 VM 管理，则其生命周期无法保证。

#### `napi_get_prototype`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_prototype(napi_env env,
                               napi_value object,
                               napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 表示要返回其原型的 JavaScript `Object` 的 `napi_value`。这返回相当于 `Object.getPrototypeOf` 的结果（与函数的 `prototype` 属性不同）。
* `[out] result`: 表示给定对象原型的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

#### `napi_get_typedarray_info`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_typedarray_info(napi_env env,
                                     napi_value typedarray,
                                     napi_typedarray_type* type,
                                     size_t* length,
                                     void** data,
                                     napi_value* arraybuffer,
                                     size_t* byte_offset)
```

* `[in] env`: 调用 API 的环境。
* `[in] typedarray`: 表示要查询其属性的 `TypedArray` 的 `napi_value`。
* `[out] type`: `TypedArray` 内元素的标量数据类型。
* `[out] length`: `TypedArray` 中的元素数量。
* `[out] data`: 底层 `TypedArray` 的数据缓冲区，已通过 `byte_offset` 值调整，使其指向 `TypedArray` 中的第一个元素。如果数组的长度为 `0`，这可能是 `NULL` 或任何其他指针值。
* `[out] arraybuffer`: `TypedArray` 底层的 `ArrayBuffer`。
* `[out] byte_offset`: 底层原生数组中数组第一个元素所在的字节偏移量。数据参数的值已经过调整，因此数据指向数组中的第一个元素。因此，原生数组的第一个字节将在 `data - byte_offset`。

如果 API 成功则返回 `napi_ok`。

此 API 返回类型化数组的各种属性。

任何输出参数都可以是 `NULL`，如果不需要该属性。

_警告_：使用此 API 时要小心，因为底层数据缓冲区由 VM 管理。

#### `napi_get_dataview_info`

<!-- YAML
added: v8.3.0
napiVersion: 1
-->

```c
napi_status napi_get_dataview_info(napi_env env,
                                   napi_value dataview,
                                   size_t* byte_length,
                                   void** data,
                                   napi_value* arraybuffer,
                                   size_t* byte_offset)
```

* `[in] env`: 调用 API 的环境。
* `[in] dataview`: 表示要查询其属性的 `DataView` 的 `napi_value`。
* `[out] byte_length`: `DataView` 中的字节数。
* `[out] data`: `DataView` 的底层数据缓冲区。如果 byte\_length 为 `0`，这可能是 `NULL` 或任何其他指针值。
* `[out] arraybuffer`: `DataView` 底层的 `ArrayBuffer`。
* `[out] byte_offset`: 数据缓冲区中开始投影 `DataView` 的字节偏移量。

如果 API 成功则返回 `napi_ok`。

任何输出参数都可以是 `NULL`，如果不需要该属性。

此 API 返回 `DataView` 的各种属性。

#### `napi_get_date_value`

<!-- YAML
added:
 - v11.11.0
 - v10.17.0
napiVersion: 5
-->

```c
napi_status napi_get_date_value(napi_env env,
                                napi_value value,
                                double* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript `Date` 的 `napi_value`。
* `[out] result`: 时间值作为 `double`，表示为自 1970 年 1 月 1 日 UTC 午夜以来的毫秒数。

此 API 不观察闰秒；它们被忽略，因为 ECMAScript 与 POSIX 时间规范对齐。

如果 API 成功则返回 `napi_ok`。如果传入非日期 `napi_value`，则返回 `napi_date_expected`。

此 API 返回给定 JavaScript `Date` 的时间值的 C double 基元。

#### `napi_get_value_bool`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_value_bool(napi_env env, napi_value value, bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript `Boolean` 的 `napi_value`。
* `[out] result`: 给定 JavaScript `Boolean` 的等效 C 布尔基元。

如果 API 成功则返回 `napi_ok`。如果传入非布尔 `napi_value`，则返回 `napi_boolean_expected`。

此 API 返回给定 JavaScript `Boolean` 的等效 C 布尔基元。

#### `napi_get_value_double`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_value_double(napi_env env,
                                  napi_value value,
                                  double* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript `number` 的 `napi_value`。
* `[out] result`: 给定 JavaScript `number` 的等效 C double 基元。

如果 API 成功则返回 `napi_ok`。如果传入非数字 `napi_value`，则返回 `napi_number_expected`。

此 API 返回给定 JavaScript `number` 的等效 C double 基元。

#### `napi_get_value_bigint_int64`

<!-- YAML
added: v10.7.0
napiVersion: 6
-->

```c
napi_status napi_get_value_bigint_int64(napi_env env,
                                        napi_value value,
                                        int64_t* result,
                                        bool* lossless);
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript `BigInt` 的 `napi_value`。
* `[out] result`: 给定 JavaScript `BigInt` 的等效 C `int64_t` 基元。
* `[out] lossless`: 指示 `BigInt` 值是否无损转换。

如果 API 成功则返回 `napi_ok`。如果传入非 `BigInt`，则返回 `napi_bigint_expected`。

此 API 返回给定 JavaScript `BigInt` 的等效 C `int64_t` 基元。如果需要，它将截断值，将 `lossless` 设置为 `false`。

#### `napi_get_value_bigint_uint64`

<!-- YAML
added: v10.7.0
napiVersion: 6
-->

```c
napi_status napi_get_value_bigint_uint64(napi_env env,
                                        napi_value value,
                                        uint64_t* result,
                                        bool* lossless);
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript `BigInt` 的 `napi_value`。
* `[out] result`: 给定 JavaScript `BigInt` 的等效 C `uint64_t` 基元。
* `[out] lossless`: 指示 `BigInt` 值是否无损转换。

如果 API 成功则返回 `napi_ok`。如果传入非 `BigInt`，则返回 `napi_bigint_expected`。

此 API 返回给定 JavaScript `BigInt` 的等效 C `uint64_t` 基元。如果需要，它将截断值，将 `lossless` 设置为 `false`。

#### `napi_get_value_bigint_words`

<!-- YAML
added: v10.7.0
napiVersion: 6
-->

```c
napi_status napi_get_value_bigint_words(napi_env env,
                                        napi_value value,
                                        int* sign_bit,
                                        size_t* word_count,
                                        uint64_t* words);
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript `BigInt` 的 `napi_value`。
* `[out] sign_bit`: 表示 JavaScript `BigInt` 是正还是负的整数。
* `[in/out] word_count`: 必须初始化为 `words` 数组的长度。返回时，它将设置为存储此 `BigInt` 所需的实际字数。
* `[out] words`: 指向预分配的 64 位字数组的指针。

如果 API 成功则返回 `napi_ok`。

此 API 将单个 `BigInt` 值转换为符号位、64 位小端数组和数组中的元素数量。`sign_bit` 和 `words` 都可以设置为 `NULL`，以便仅获取 `word_count`。

#### `napi_get_value_external`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_value_external(napi_env env,
                                    napi_value value,
                                    void** result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript 外部值的 `napi_value`。
* `[out] result`: 由 JavaScript 外部值包装的数据的指针。

如果 API 成功则返回 `napi_ok`。如果传入非外部 `napi_value`，则返回 `napi_invalid_arg`。

此 API 检索先前传递给 `napi_create_external()` 的外部数据指针。

#### `napi_get_value_int32`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_value_int32(napi_env env,
                                 napi_value value,
                                 int32_t* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript `number` 的 `napi_value`。
* `[out] result`: 给定 JavaScript `number` 的等效 C `int32` 基元。

如果 API 成功则返回 `napi_ok`。如果传入非数字 `napi_value`，则返回 `napi_number_expected`。

此 API 返回给定 JavaScript `number` 的等效 C `int32` 基元。

如果数字超过 32 位整数的范围，则结果将被截断为底部 32 位的等效值。如果值 > 2<sup>31</sup> - 1，这可能导致大的正数变为负数。

非有限数字值（`NaN`、`+Infinity` 或 `-Infinity`）将结果设置为零。

#### `napi_get_value_int64`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_value_int64(napi_env env,
                                 napi_value value,
                                 int64_t* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript `number` 的 `napi_value`。
* `[out] result`: 给定 JavaScript `number` 的等效 C `int64` 基元。

如果 API 成功则返回 `napi_ok`。如果传入非数字 `napi_value`，则返回 `napi_number_expected`。

此 API 返回给定 JavaScript `number` 的等效 C `int64` 基元。

超出 [`Number.MIN_SAFE_INTEGER`][] `-(2**53 - 1)` - [`Number.MAX_SAFE_INTEGER`][] `(2**53 - 1)` 范围的 `number` 值将丢失精度。

非有限数字值（`NaN`、`+Infinity` 或 `-Infinity`）将结果设置为零。

#### `napi_get_value_string_latin1`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_value_string_latin1(napi_env env,
                                         napi_value value,
                                         char* buf,
                                         size_t bufsize,
                                         size_t* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript 字符串的 `napi_value`。
* `[in] buf`: 写入 ISO-8859-1 编码字符串的缓冲区。如果传入 `NULL`，则字符串的长度（字节，不包括空终止符）将在 `result` 中返回。
* `[in] bufsize`: 目标缓冲区的大小。当此值不足时，返回的字符串将被截断并空终止。如果此值为零，则不返回字符串，并且不对缓冲区进行任何更改。
* `[out] result`: 复制到缓冲区中的字节数，不包括空终止符。

如果 API 成功则返回 `napi_ok`。如果传入非 `string` `napi_value`，则返回 `napi_string_expected`。

此 API 返回与传入值对应的 ISO-8859-1 编码字符串。

#### `napi_get_value_string_utf8`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_value_string_utf8(napi_env env,
                                       napi_value value,
                                       char* buf,
                                       size_t bufsize,
                                       size_t* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript 字符串的 `napi_value`。
* `[in] buf`: 写入 UTF8 编码字符串的缓冲区。如果传入 `NULL`，则字符串的长度（字节，不包括空终止符）将在 `result` 中返回。
* `[in] bufsize`: 目标缓冲区的大小。当此值不足时，返回的字符串将被截断并空终止。如果此值为零，则不返回字符串，并且不对缓冲区进行任何更改。
* `[out] result`: 复制到缓冲区中的字节数，不包括空终止符。

如果 API 成功则返回 `napi_ok`。如果传入非 `string` `napi_value`，则返回 `napi_string_expected`。

此 API 返回与传入值对应的 UTF8 编码字符串。

#### `napi_get_value_string_utf16`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_value_string_utf16(napi_env env,
                                        napi_value value,
                                        char16_t* buf,
                                        size_t bufsize,
                                        size_t* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript 字符串的 `napi_value`。
* `[in] buf`: 写入 UTF16-LE 编码字符串的缓冲区。如果传入 `NULL`，则字符串的长度（双字节代码单元，不包括空终止符）将返回。
* `[in] bufsize`: 目标缓冲区的大小。当此值不足时，返回的字符串将被截断并空终止。如果此值为零，则不返回字符串，并且不对缓冲区进行任何更改。
* `[out] result`: 复制到缓冲区中的双字节代码单元数，不包括空终止符。

如果 API 成功则返回 `napi_ok`。如果传入非 `string` `napi_value`，则返回 `napi_string_expected`。

此 API 返回与传入值对应的 UTF16 编码字符串。

#### `napi_get_value_uint32`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_value_uint32(napi_env env,
                                  napi_value value,
                                  uint32_t* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 表示 JavaScript `number` 的 `napi_value`。
* `[out] result`: 给定 `napi_value` 作为 `uint32_t` 的等效 C 基元。

如果 API 成功则返回 `napi_ok`。如果传入非数字 `napi_value`，则返回 `napi_number_expected`。

此 API 返回给定 `napi_value` 作为 `uint32_t` 的等效 C 基元。

### 获取全局实例的函数

#### `napi_get_boolean`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_boolean(napi_env env, bool value, napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要检索的布尔值。
* `[out] result`: 表示要检索的 JavaScript `Boolean` 单例的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 用于返回用于表示给定布尔值的 JavaScript 单例对象。

#### `napi_get_global`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_global(napi_env env, napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[out] result`: 表示 JavaScript `global` 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回 `global` 对象。

#### `napi_get_null`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_null(napi_env env, napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[out] result`: 表示 JavaScript `null` 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回 `null` 对象。

#### `napi_get_undefined`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_undefined(napi_env env, napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[out] result`: 表示 JavaScript Undefined 值的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 返回 Undefined 对象。

## 处理 JavaScript 值和抽象操作

Node-API 暴露了一组 API 来对 JavaScript 值执行一些抽象操作。

这些 API 支持执行以下操作之一：

1. 将 JavaScript 值强制转换为特定的 JavaScript 类型（如 `number` 或 `string`）。
2. 检查 JavaScript 值的类型。
3. 检查两个 JavaScript 值是否相等。

### `napi_coerce_to_bool`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_coerce_to_bool(napi_env env,
                                napi_value value,
                                napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要强制转换的 JavaScript 值。
* `[out] result`: 表示强制转换后的 JavaScript `Boolean` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 实现了 ECMAScript 语言规范的[ToBoolean 部分][]中定义的抽象操作 `ToBoolean()`。

### `napi_coerce_to_number`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_coerce_to_number(napi_env env,
                                  napi_value value,
                                  napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要强制转换的 JavaScript 值。
* `[out] result`: 表示强制转换后的 JavaScript `number` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 实现了 ECMAScript 语言规范的[ToNumber 部分][]中定义的抽象操作 `ToNumber()`。如果传入的值是对象，此函数可能运行 JS 代码。

### `napi_coerce_to_object`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_coerce_to_object(napi_env env,
                                  napi_value value,
                                  napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要强制转换的 JavaScript 值。
* `[out] result`: 表示强制转换后的 JavaScript `Object` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 实现了 ECMAScript 语言规范的[ToObject 部分][]中定义的抽象操作 `ToObject()`。

### `napi_coerce_to_string`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_coerce_to_string(napi_env env,
                                  napi_value value,
                                  napi_value* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要强制转换的 JavaScript 值。
* `[out] result`: 表示强制转换后的 JavaScript `string` 的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此 API 实现了 ECMAScript 语言规范的[ToString 部分][]中定义的抽象操作 `ToString()`。如果传入的值是对象，此函数可能运行 JS 代码。

### `napi_typeof`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_typeof(napi_env env, napi_value value, napi_valuetype* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要查询其类型的 JavaScript 值。
* `[out] result`: JavaScript 值的类型。

如果 API 成功则返回 `napi_ok`。

* 如果 `value` 的类型不是已知的 ECMAScript 类型并且 `value` 不是外部值，则返回 `napi_invalid_arg`。

此 API 表示的行为类似于在对象上调用 `typeof` 运算符，如 ECMAScript 语言规范的[typeof 运算符部分][]中所定义。但是，有一些区别：

1. 它支持检测外部值。
2. 它将 `null` 检测为单独的类型，而 ECMAScript `typeof` 会检测为 `object`。

如果 `value` 的类型无效，则返回错误。

### `napi_instanceof`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_instanceof(napi_env env,
                            napi_value object,
                            napi_value constructor,
                            bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要检查的 JavaScript 值。
* `[in] constructor`: 要检查的构造函数函数的 JavaScript 函数对象。
* `[out] result`: 布尔值，如果 `object instanceof constructor` 为 true 则设置为 true。

如果 API 成功则返回 `napi_ok`。

此 API 表示在对象上调用 `instanceof` 运算符，如 ECMAScript 语言规范的[instanceof 运算符部分][]中所定义。

### `napi_is_array`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_is_array(napi_env env, napi_value value, bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要检查的 JavaScript 值。
* `[out] result`: 给定对象是否为数组。

如果 API 成功则返回 `napi_ok`。

此 API 表示在对象上执行 `IsArray` 操作，如 ECMAScript 语言规范的[IsArray 部分][]中所定义。

### `napi_is_arraybuffer`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_is_arraybuffer(napi_env env, napi_value value, bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要检查的 JavaScript 值。
* `[out] result`: 给定对象是否为 `ArrayBuffer`。

如果 API 成功则返回 `napi_ok`。

此 API 检查传入的 `Object` 是否为数组缓冲区。

### `napi_is_buffer`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_is_buffer(napi_env env, napi_value value, bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要检查的 JavaScript 值。
* `[out] result`: 给定的 `napi_value` 是否表示 `node::Buffer` 或 `Uint8Array` 对象。

如果 API 成功则返回 `napi_ok`。

此 API 检查传入的 `Object` 是否为缓冲区或 Uint8Array。如果调用者需要检查值是否为 Uint8Array，应优先使用 [`napi_is_typedarray`][]。

### `napi_is_date`

<!-- YAML
added:
 - v11.11.0
 - v10.17.0
napiVersion: 5
-->

```c
napi_status napi_is_date(napi_env env, napi_value value, bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要检查的 JavaScript 值。
* `[out] result`: 给定的 `napi_value` 是否表示 JavaScript `Date` 对象。

如果 API 成功则返回 `napi_ok`。

此 API 检查传入的 `Object` 是否为日期。

### `napi_is_error`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_is_error(napi_env env, napi_value value, bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要检查的 JavaScript 值。
* `[out] result`: 给定的 `napi_value` 是否表示 `Error` 对象。

如果 API 成功则返回 `napi_ok`。

此 API 检查传入的 `Object` 是否为 `Error`。

### `napi_is_typedarray`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_is_typedarray(napi_env env, napi_value value, bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要检查的 JavaScript 值。
* `[out] result`: 给定的 `napi_value` 是否表示 `TypedArray`。

如果 API 成功则返回 `napi_ok`。

此 API 检查传入的 `Object` 是否为类型化数组。

### `napi_is_dataview`

<!-- YAML
added: v8.3.0
napiVersion: 1
-->

```c
napi_status napi_is_dataview(napi_env env, napi_value value, bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] value`: 要检查的 JavaScript 值。
* `[out] result`: 给定的 `napi_value` 是否表示 `DataView`。

如果 API 成功则返回 `napi_ok`。

此 API 检查传入的 `Object` 是否为 `DataView`。

### `napi_strict_equals`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_strict_equals(napi_env env,
                               napi_value lhs,
                               napi_value rhs,
                               bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] lhs`: 要检查的 JavaScript 值。
* `[in] rhs`: 要检查的 JavaScript 值。
* `[out] result`: 两个 `napi_value` 对象是否相等。

如果 API 成功则返回 `napi_ok`。

此 API 表示在 ECMAScript 语言规范的[严格相等比较算法][]中定义的抽象严格相等比较操作。

### `napi_detach_arraybuffer`

<!-- YAML
added:
 - v14.8.0
 - v12.19.0
napiVersion: 8
-->

```c
napi_status napi_detach_arraybuffer(napi_env env,
                                    napi_value arraybuffer)
```

* `[in] env`: 调用 API 的环境。
* `[in] arraybuffer`: 要分离的 JavaScript `ArrayBuffer`。

如果 API 成功则返回 `napi_ok`。

此 API 表示 ECMAScript 语言规范的[ArrayBuffer 分离操作][]中定义的 `ArrayBuffer` 分离操作。这用于从 `ArrayBuffer` 的后备内存中分离。这通常用于在将 `ArrayBuffer` 传输到另一个线程后释放内存。

### `napi_is_detached_arraybuffer`

<!-- YAML
added:
 - v14.8.0
 - v12.19.0
napiVersion: 8
-->

```c
napi_status napi_is_detached_arraybuffer(napi_env env,
                                         napi_value arraybuffer,
                                         bool* result)
```

* `[in] env`: 调用 API 的环境。
* `[in] arraybuffer`: 要检查的 JavaScript `ArrayBuffer`。
* `[out] result`: 给定的 `ArrayBuffer` 是否已分离。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `ArrayBuffer` 是否已分离。

## 使用 JavaScript 属性

Node-API 暴露了一组 API 来获取和设置 JavaScript 对象的属性。描述这些属性的描述符可以在[属性描述符部分][]中找到。

### `napi_get_property_names`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property_names(napi_env env,
                                    napi_value object,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`，其中包含给定对象的属性名称字符串。API 可以添加额外的 `undefined` 值到数组中，以防止其成为密集数组，并且可以添加任意索引。有关详细信息，请参阅问题 [#3996](https://github.com/nodejs/node/issues/3996)。

如果 API 成功则返回 `napi_ok`。

此 API 返回给定对象的属性名称数组。`result` 中的属性名称不包含继承的属性。

### `napi_get_all_property_names`

<!-- YAML
added:
 - v13.7.0
 - v12.17.0
 - v10.20.0
napiVersion: 6
-->

```c
napi_get_all_property_names(napi_env env,
                            napi_value object,
                            napi_key_collection_mode key_mode,
                            napi_key_filter key_filter,
                            napi_key_conversion key_conversion,
                            napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] key_mode`: 是否检索原型属性以及是否检索不可枚举属性。
* `[in] key_filter`: 哪些值要过滤掉（使用按位或）。
* `[in] key_conversion`: 是否将数字属性索引转换为字符串。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`，其中包含给定对象的属性名称。

如果 API 成功则返回 `napi_ok`。

此 API 返回一个数组，其中包含给定对象的属性名称。`object` 中不存在的属性可能包含在结果中。

### `napi_set_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] key`: 要设置的属性的名称。这应该是一个字符串或 `symbol`。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性。

### `napi_get_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] key`: 要检索的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性。

### `napi_has_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性。

### `napi_delete_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_property(napi_env env,
                                 napi_value object,
                                 napi_value key,
                                 bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] key`: 要删除的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该属性，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `key` 属性。

### `napi_has_own_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_has_own_property(napi_env env,
                                  napi_value object,
                                  napi_value key,
                                  bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有自己的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有自己的命名属性。`key` 必须是字符串或 `symbol`。

### `napi_set_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] utf8name`: 要设置的属性的名称。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性，其中属性名称是 UTF8 编码的字符串。

### `napi_get_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] utf8name`: 要检索的属性的名称。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性，其中属性名称是 UTF8 编码的字符串。

### `napi_has_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] utf8name`: 要检查的属性的名称。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性，其中属性名称是 UTF8 编码的字符串。

### `napi_set_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] index`: 要设置的属性的索引。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置元素。

### `napi_get_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] index`: 要检索的属性的索引。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取元素。

### `napi_has_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] index`: 要检查的属性的索引。
* `[out] result`: 对象是否具有要检查的元素。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有索引属性。

### `napi_delete_element`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_element(napi_env env,
                                napi_value object,
                                uint32_t index,
                                bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] index`: 要删除的属性的索引。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该元素，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `index` 属性。

### `napi_define_properties`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_properties(napi_env env,
                                   napi_value object,
                                   size_t property_count,
                                   const napi_property_descriptor* properties);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要设置属性的对象。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组。

如果 API 成功则返回 `napi_ok`。

此方法允许高效定义对象的多个属性。给定的属性描述符数组用于设置对象的属性。此 API 的默认行为类似于 `Object.defineProperties()`。

### `napi_object_freeze`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_freeze(napi_env env,
                               napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要冻结的对象。

如果 API 成功则返回 `napi_ok`。

此方法冻结给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.freeze()`。

### `napi_object_seal`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_seal(napi_env env,
                             napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要密封的对象。

如果 API 成功则返回 `napi_ok`。

此方法密封给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.seal()`。

## 使用 JavaScript 函数

Node-API 提供了一组 API 来从原生代码调用 JavaScript 函数。这些 API 支持两种调用 JavaScript 函数的方式：正常方式，函数类似于 JavaScript 代码调用，以及作为构造函数的方式。

此外，Node-API 提供了一种创建新函数实例的 API。在这种情况下，原生代码提供了作为新函数实现的原生函数。结果是可以通过 JavaScript 代码调用的函数对象。

### `napi_call_function`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_call_function(napi_env env,
                                           napi_value recv,
                                           napi_value func,
                                           size_t argc,
                                           const napi_value* argv,
                                           napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] recv`: 作为函数调用的 `this` 值的 `object`。
* `[in] func`: 要调用的 JavaScript `function`。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法允许从原生代码调用 JavaScript 函数对象。这是用于调用函数的原生 API。一个典型的用法是在操作完成时调用用 JavaScript 编写的回调。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_new_instance`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_new_instance(napi_env env,
                                          napi_value constructor,
                                          size_t argc,
                                          const napi_value* argv,
                                          napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] constructor`: 表示要调用的 JavaScript 函数的 `napi_value`。此 API 必须将构造函数传递给 JavaScript 函数。JavaScript 函数创建新对象并初始化它，然后用作构造函数。有关更多详细信息，请参阅[构造函数定义][]。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示构造函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法用于从原生代码实例化 JavaScript 对象。这相当于在 JavaScript 中执行 `new Constructor()`，其中 `Constructor` 是作为构造函数传递的函数对象。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_define_class`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_class(napi_env env,
                              const char* utf8name,
                              size_t length,
                              napi_callback constructor,
                              void* data,
                              size_t property_count,
                              const napi_property_descriptor* properties,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] utf8name`: 类的名称；这通常是传递给构造函数的名称。
* `[in] length`: 类名称的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[in] constructor`: 处理类实例化和构造的回调函数。这应该是一个静态成员函数，其签名如下所述。
* `[in] data`: 作为 `this` 参数传递给构造函数的任意数据。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组，用于静态方法和类实例化后添加到类原型的属性。有关更多详细信息，请参阅 `napi_property_descriptor`。
* `[out] result`: 表示构造函数的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

定义与 C++ 类相对应的 JavaScript 类，包括：

* 一个 JavaScript 函数，作为类的构造函数。此函数必须与传递给 `napi_define_class` 的 `constructor` 回调相对应。
* 与 C++ 类实例关联的所有静态数据/方法。这些属性被添加到构造函数中。
* 与 C++ 类实例关联的所有非静态数据/方法。这些属性被添加到构造函数的原型中。

C++ `constructor` 回调应该是一个静态函数，在创建 JavaScript 对象时调用，它本身就是一个类的实例。回调的签名如下：

```c
napi_value callback(napi_env env, napi_callback_info info);
```

接收的参数：

* `[in] env`: 调用回调的环境。
* `[in] info`: 回调信息。

回调可以返回以下内容：

* 另一个 `napi_value`，它应该是一个表示新创建的实例的 JavaScript 对象。
* `NULL`，如果构造函数抛出异常。

回调内部，`napi_callback_info` 参数可用于检索传入的参数（相当于每个构造函数接收的 `arguments` 对象）和新的实例的 `this`。

提供给 `napi_define_class` 的 `data` 指针可以在回调内部作为 `info` 参数传递给 [`napi_get_cb_info`][] 时作为新的 `this` 指针访问。

构造函数的原型被自动设置为具有属性 `constructor` 的对象，该属性对应于构造函数本身。没有必要通过 `properties` 参数传递属性描述符来设置它。

使用此 API 定义的函数可以通过 JavaScript 中的 `new` 运算符调用。

通常，在调用 `new` 时，构造函数会创建一个类型为构造函数的新普通对象（例如，构造函数通常是一个全局属性，其原型是一个普通对象）。这个新创建的对象的原型被设置为构造函数的 `prototype` 属性。构造函数运行，如果对象被返回，则 `new` 的结果就是该对象。如果构造函数返回非对象，则返回新创建的对象。

使用 `napi_define_class` 时，允许原生代码返回一个对象，该对象与通过运行构造函数创建的对象不同。为了支持此功能，使用 `napi_define_class` 定义的构造函数的 `prototype` 属性必须是一个包装了原生构造函数的普通对象。然后，当调用构造函数时，Node-API 将把新创建对象的原型设置为这个普通对象。然后，原生构造函数可以返回一个对象，其内部原型与普通对象不同。返回的对象将被 `new` 运算符使用。

例如，使用以下 JavaScript 和原生代码：

```js
'use strict';

const Example = require('bindings')('example').Example;
const example = new Example();
console.log(example instanceof Example);
```

```c
// ...
napi_value ExampleConstructor(napi_env env, napi_callback_info info) {
  napi_value new_target;
  napi_status status = napi_get_new_target(env, info, &new_target);
  assert(status == napi_ok);
  bool is_new_target;
  status = napi_strict_equals(env, new_target, new_target, &is_new_target);
  assert(status == napi_ok);
  if (!is_new_target) {
    // 这发生在 `Example()` 被调用时。
    // 返回一个假实例以允许 `instanceof` 工作。
    napi_value instance;
    status = napi_new_instance(env, new_target, 0, NULL, &instance);
    assert(status == napi_ok);
    return instance;
  }

  // 这是实际的构造函数。
  napi_value this;
  status = napi_get_cb_info(env, info, 0, NULL, &this, NULL);
  assert(status == napi_ok);
  // 返回 `this` 或我们选择的任何其他对象。
  return this;
}
napi_property_descriptor desc = { "Example", NULL, ExampleConstructor, NULL, NULL, NULL, napi_default, NULL };
napi_value result;
napi_define_class(env, "Example", NAPI_AUTO_LENGTH, ExampleConstructor, data, 1, &desc, &result);
```

当原生构造函数运行时，`new_target` 指向由 `napi_define_class` 创建的构造函数。当 `Example` 被调用为函数而不是使用 `new` 时，`new_target` 是 `undefined`。然后，原生构造函数可以检查 `new_target` 是否为构造函数本身，以确定它是作为函数调用还是作为构造函数调用。

当构造函数作为函数调用时，行为完全取决于原生代码。在某些情况下，例如 [`URL`][] 构造函数，当作为函数调用时，它返回一个新对象，就好像它是作为构造函数调用一样。在其他情况下，当作为函数调用时，它可能抛出或返回 `undefined`。

在上面的示例中，当构造函数作为函数调用时，原生代码通过调用构造函数创建一个新实例，然后返回该实例。这允许 `instanceof` 对返回的对象工作，因为它具有正确的原型。当作为构造函数调用时，它返回 `this`，这可能是另一个对象。这允许原生代码返回一个单例或一个先前创建的对象。如果原生代码没有返回任何对象，可以返回 `this` 或任何其他创建的对象。

### `napi_get_prototype`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_prototype(napi_env env,
                               napi_value object,
                               napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要返回其原型的 `napi_value`。这返回相当于 `Object.getPrototypeOf` 的结果（与函数的 `prototype` 属性不同）。
* `[out] result`: 表示给定对象原型的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

### `napi_get_property_names`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property_names(napi_env env,
                                    napi_value object,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`，其中包含给定对象的属性名称字符串。API 可以添加额外的 `undefined` 值到数组中，以防止其成为密集数组，并且可以添加任意索引。有关详细信息，请参阅问题 [#3996](https://github.com/nodejs/node/issues/3996)。

如果 API 成功则返回 `napi_ok`。

此 API 返回给定对象的属性名称数组。`result` 中的属性名称不包含继承的属性。

### `napi_set_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] key`: 要设置的属性的名称。这应该是一个字符串或 `symbol`。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性。

### `napi_get_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] key`: 要检索的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性。

### `napi_has_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性。

### `napi_delete_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_property(napi_env env,
                                 napi_value object,
                                 napi_value key,
                                 bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] key`: 要删除的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该属性，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `key` 属性。

### `napi_has_own_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_has_own_property(napi_env env,
                                  napi_value object,
                                  napi_value key,
                                  bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有自己的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有自己的命名属性。`key` 必须是字符串或 `symbol`。

### `napi_set_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] utf8name`: 要设置的属性的名称。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性，其中属性名称是 UTF8 编码的字符串。

### `napi_get_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] utf8name`: 要检索的属性的名称。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性，其中属性名称是 UTF8 编码的字符串。

### `napi_has_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] utf8name`: 要检查的属性的名称。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性，其中属性名称是 UTF8 编码的字符串。

### `napi_set_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] index`: 要设置的属性的索引。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置元素。

### `napi_get_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] index`: 要检索的属性的索引。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取元素。

### `napi_has_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] index`: 要检查的属性的索引。
* `[out] result`: 对象是否具有要检查的元素。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有索引属性。

### `napi_delete_element`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_element(napi_env env,
                                napi_value object,
                                uint32_t index,
                                bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] index`: 要删除的属性的索引。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该元素，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `index` 属性。

### `napi_define_properties`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_properties(napi_env env,
                                   napi_value object,
                                   size_t property_count,
                                   const napi_property_descriptor* properties);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要设置属性的对象。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组。

如果 API 成功则返回 `napi_ok`。

此方法允许高效定义对象的多个属性。给定的属性描述符数组用于设置对象的属性。此 API 的默认行为类似于 `Object.defineProperties()`。

### `napi_object_freeze`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_freeze(napi_env env,
                               napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要冻结的对象。

如果 API 成功则返回 `napi_ok`。

此方法冻结给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.freeze()`。

### `napi_object_seal`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_seal(napi_env env,
                             napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要密封的对象。

如果 API 成功则返回 `napi_ok`。

此方法密封给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.seal()`。

## 使用 JavaScript 函数

Node-API 提供了一组 API 来从原生代码调用 JavaScript 函数。这些 API 支持两种调用 JavaScript 函数的方式：正常方式，函数类似于 JavaScript 代码调用，以及作为构造函数的方式。

此外，Node-API 提供了一种创建新函数实例的 API。在这种情况下，原生代码提供了作为新函数实现的原生函数。结果是可以通过 JavaScript 代码调用的函数对象。

### `napi_call_function`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_call_function(napi_env env,
                                           napi_value recv,
                                           napi_value func,
                                           size_t argc,
                                           const napi_value* argv,
                                           napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] recv`: 作为函数调用的 `this` 值的 `object`。
* `[in] func`: 要调用的 JavaScript `function`。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法允许从原生代码调用 JavaScript 函数对象。这是用于调用函数的原生 API。一个典型的用法是在操作完成时调用用 JavaScript 编写的回调。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_new_instance`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_new_instance(napi_env env,
                                          napi_value constructor,
                                          size_t argc,
                                          const napi_value* argv,
                                          napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] constructor`: 表示要调用的 JavaScript 函数的 `napi_value`。此 API 必须将构造函数传递给 JavaScript 函数。JavaScript 函数创建新对象并初始化它，然后用作构造函数。有关更多详细信息，请参阅[构造函数定义][]。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示构造函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法用于从原生代码实例化 JavaScript 对象。这相当于在 JavaScript 中执行 `new Constructor()`，其中 `Constructor` 是作为构造函数传递的函数对象。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_define_class`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_class(napi_env env,
                              const char* utf8name,
                              size_t length,
                              napi_callback constructor,
                              void* data,
                              size_t property_count,
                              const napi_property_descriptor* properties,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] utf8name`: 类的名称；这通常是传递给构造函数的名称。
* `[in] length`: 类名称的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[in] constructor`: 处理类实例化和构造的回调函数。这应该是一个静态成员函数，其签名如下所述。
* `[in] data`: 作为 `this` 参数传递给构造函数的任意数据。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组，用于静态方法和类实例化后添加到类原型的属性。有关更多详细信息，请参阅 `napi_property_descriptor`。
* `[out] result`: 表示构造函数的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

定义与 C++ 类相对应的 JavaScript 类，包括：

* 一个 JavaScript 函数，作为类的构造函数。此函数必须与传递给 `napi_define_class` 的 `constructor` 回调相对应。
* 与 C++ 类实例关联的所有静态数据/方法。这些属性被添加到构造函数中。
* 与 C++ 类实例关联的所有非静态数据/方法。这些属性被添加到构造函数的原型中。

C++ `constructor` 回调应该是一个静态函数，在创建 JavaScript 对象时调用，它本身就是一个类的实例。回调的签名如下：

```c
napi_value callback(napi_env env, napi_callback_info info);
```

接收的参数：

* `[in] env`: 调用回调的环境。
* `[in] info`: 回调信息。

回调可以返回以下内容：

* 另一个 `napi_value`，它应该是一个表示新创建的实例的 JavaScript 对象。
* `NULL`，如果构造函数抛出异常。

回调内部，`napi_callback_info` 参数可用于检索传入的参数（相当于每个构造函数接收的 `arguments` 对象）和新的实例的 `this`。

提供给 `napi_define_class` 的 `data` 指针可以在回调内部作为 `info` 参数传递给 [`napi_get_cb_info`][] 时作为新的 `this` 指针访问。

构造函数的原型被自动设置为具有属性 `constructor` 的对象，该属性对应于构造函数本身。没有必要通过 `properties` 参数传递属性描述符来设置它。

使用此 API 定义的函数可以通过 JavaScript 中的 `new` 运算符调用。

通常，在调用 `new` 时，构造函数会创建一个类型为构造函数的新普通对象（例如，构造函数通常是一个全局属性，其原型是一个普通对象）。这个新创建的对象的原型被设置为构造函数的 `prototype` 属性。构造函数运行，如果对象被返回，则 `new` 的结果就是该对象。如果构造函数返回非对象，则返回新创建的对象。

使用 `napi_define_class` 时，允许原生代码返回一个对象，该对象与通过运行构造函数创建的对象不同。为了支持此功能，使用 `napi_define_class` 定义的构造函数的 `prototype` 属性必须是一个包装了原生构造函数的普通对象。然后，当调用构造函数时，Node-API 将把新创建对象的原型设置为这个普通对象。然后，原生构造函数可以返回一个对象，其内部原型与普通对象不同。返回的对象将被 `new` 运算符使用。

例如，使用以下 JavaScript 和原生代码：

```js
'use strict';

const Example = require('bindings')('example').Example;
const example = new Example();
console.log(example instanceof Example);
```

```c
// ...
napi_value ExampleConstructor(napi_env env, napi_callback_info info) {
  napi_value new_target;
  napi_status status = napi_get_new_target(env, info, &new_target);
  assert(status == napi_ok);
  bool is_new_target;
  status = napi_strict_equals(env, new_target, new_target, &is_new_target);
  assert(status == napi_ok);
  if (!is_new_target) {
    // 这发生在 `Example()` 被调用时。
    // 返回一个假实例以允许 `instanceof` 工作。
    napi_value instance;
    status = napi_new_instance(env, new_target, 0, NULL, &instance);
    assert(status == napi_ok);
    return instance;
  }

  // 这是实际的构造函数。
  napi_value this;
  status = napi_get_cb_info(env, info, 0, NULL, &this, NULL);
  assert(status == napi_ok);
  // 返回 `this` 或我们选择的任何其他对象。
  return this;
}
napi_property_descriptor desc = { "Example", NULL, ExampleConstructor, NULL, NULL, NULL, napi_default, NULL };
napi_value result;
napi_define_class(env, "Example", NAPI_AUTO_LENGTH, ExampleConstructor, data, 1, &desc, &result);
```

当原生构造函数运行时，`new_target` 指向由 `napi_define_class` 创建的构造函数。当 `Example` 被调用为函数而不是使用 `new` 时，`new_target` 是 `undefined`。然后，原生构造函数可以检查 `new_target` 是否为构造函数本身，以确定它是作为函数调用还是作为构造函数调用。

当构造函数作为函数调用时，行为完全取决于原生代码。在某些情况下，例如 [`URL`][] 构造函数，当作为函数调用时，它返回一个新对象，就好像它是作为构造函数调用一样。在其他情况下，当作为函数调用时，它可能抛出或返回 `undefined`。

在上面的示例中，当构造函数作为函数调用时，原生代码通过调用构造函数创建一个新实例，然后返回该实例。这允许 `instanceof` 对返回的对象工作，因为它具有正确的原型。当作为构造函数调用时，它返回 `this`，这可能是另一个对象。这允许原生代码返回一个单例或一个先前创建的对象。如果原生代码没有返回任何对象，可以返回 `this` 或任何其他创建的对象。

### `napi_get_prototype`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_prototype(napi_env env,
                               napi_value object,
                               napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要返回其原型的 `napi_value`。这返回相当于 `Object.getPrototypeOf` 的结果（与函数的 `prototype` 属性不同）。
* `[out] result`: 表示给定对象原型的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

### `napi_get_property_names`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property_names(napi_env env,
                                    napi_value object,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`，其中包含给定对象的属性名称字符串。API 可以添加额外的 `undefined` 值到数组中，以防止其成为密集数组，并且可以添加任意索引。有关详细信息，请参阅问题 [#3996](https://github.com/nodejs/node/issues/3996)。

如果 API 成功则返回 `napi_ok`。

此 API 返回给定对象的属性名称数组。`result` 中的属性名称不包含继承的属性。

### `napi_set_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] key`: 要设置的属性的名称。这应该是一个字符串或 `symbol`。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性。

### `napi_get_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] key`: 要检索的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性。

### `napi_has_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性。

### `napi_delete_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_property(napi_env env,
                                 napi_value object,
                                 napi_value key,
                                 bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] key`: 要删除的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该属性，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `key` 属性。

### `napi_has_own_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_has_own_property(napi_env env,
                                  napi_value object,
                                  napi_value key,
                                  bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有自己的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有自己的命名属性。`key` 必须是字符串或 `symbol`。

### `napi_set_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] utf8name`: 要设置的属性的名称。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性，其中属性名称是 UTF8 编码的字符串。

### `napi_get_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] utf8name`: 要检索的属性的名称。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性，其中属性名称是 UTF8 编码的字符串。

### `napi_has_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] utf8name`: 要检查的属性的名称。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性，其中属性名称是 UTF8 编码的字符串。

### `napi_set_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] index`: 要设置的属性的索引。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置元素。

### `napi_get_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] index`: 要检索的属性的索引。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取元素。

### `napi_has_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] index`: 要检查的属性的索引。
* `[out] result`: 对象是否具有要检查的元素。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有索引属性。

### `napi_delete_element`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_element(napi_env env,
                                napi_value object,
                                uint32_t index,
                                bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] index`: 要删除的属性的索引。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该元素，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `index` 属性。

### `napi_define_properties`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_properties(napi_env env,
                                   napi_value object,
                                   size_t property_count,
                                   const napi_property_descriptor* properties);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要设置属性的对象。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组。

如果 API 成功则返回 `napi_ok`。

此方法允许高效定义对象的多个属性。给定的属性描述符数组用于设置对象的属性。此 API 的默认行为类似于 `Object.defineProperties()`。

### `napi_object_freeze`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_freeze(napi_env env,
                               napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要冻结的对象。

如果 API 成功则返回 `napi_ok`。

此方法冻结给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.freeze()`。

### `napi_object_seal`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_seal(napi_env env,
                             napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要密封的对象。

如果 API 成功则返回 `napi_ok`。

此方法密封给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.seal()`。

## 使用 JavaScript 函数

Node-API 提供了一组 API 来从原生代码调用 JavaScript 函数。这些 API 支持两种调用 JavaScript 函数的方式：正常方式，函数类似于 JavaScript 代码调用，以及作为构造函数的方式。

此外，Node-API 提供了一种创建新函数实例的 API。在这种情况下，原生代码提供了作为新函数实现的原生函数。结果是可以通过 JavaScript 代码调用的函数对象。

### `napi_call_function`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_call_function(napi_env env,
                                           napi_value recv,
                                           napi_value func,
                                           size_t argc,
                                           const napi_value* argv,
                                           napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] recv`: 作为函数调用的 `this` 值的 `object`。
* `[in] func`: 要调用的 JavaScript `function`。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法允许从原生代码调用 JavaScript 函数对象。这是用于调用函数的原生 API。一个典型的用法是在操作完成时调用用 JavaScript 编写的回调。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_new_instance`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_new_instance(napi_env env,
                                          napi_value constructor,
                                          size_t argc,
                                          const napi_value* argv,
                                          napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] constructor`: 表示要调用的 JavaScript 函数的 `napi_value`。此 API 必须将构造函数传递给 JavaScript 函数。JavaScript 函数创建新对象并初始化它，然后用作构造函数。有关更多详细信息，请参阅[构造函数定义][]。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示构造函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法用于从原生代码实例化 JavaScript 对象。这相当于在 JavaScript 中执行 `new Constructor()`，其中 `Constructor` 是作为构造函数传递的函数对象。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_define_class`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_class(napi_env env,
                              const char* utf8name,
                              size_t length,
                              napi_callback constructor,
                              void* data,
                              size_t property_count,
                              const napi_property_descriptor* properties,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] utf8name`: 类的名称；这通常是传递给构造函数的名称。
* `[in] length`: 类名称的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[in] constructor`: 处理类实例化和构造的回调函数。这应该是一个静态成员函数，其签名如下所述。
* `[in] data`: 作为 `this` 参数传递给构造函数的任意数据。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组，用于静态方法和类实例化后添加到类原型的属性。有关更多详细信息，请参阅 `napi_property_descriptor`。
* `[out] result`: 表示构造函数的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

定义与 C++ 类相对应的 JavaScript 类，包括：

* 一个 JavaScript 函数，作为类的构造函数。此函数必须与传递给 `napi_define_class` 的 `constructor` 回调相对应。
* 与 C++ 类实例关联的所有静态数据/方法。这些属性被添加到构造函数中。
* 与 C++ 类实例关联的所有非静态数据/方法。这些属性被添加到构造函数的原型中。

C++ `constructor` 回调应该是一个静态函数，在创建 JavaScript 对象时调用，它本身就是一个类的实例。回调的签名如下：

```c
napi_value callback(napi_env env, napi_callback_info info);
```

接收的参数：

* `[in] env`: 调用回调的环境。
* `[in] info`: 回调信息。

回调可以返回以下内容：

* 另一个 `napi_value`，它应该是一个表示新创建的实例的 JavaScript 对象。
* `NULL`，如果构造函数抛出异常。

回调内部，`napi_callback_info` 参数可用于检索传入的参数（相当于每个构造函数接收的 `arguments` 对象）和新的实例的 `this`。

提供给 `napi_define_class` 的 `data` 指针可以在回调内部作为 `info` 参数传递给 [`napi_get_cb_info`][] 时作为新的 `this` 指针访问。

构造函数的原型被自动设置为具有属性 `constructor` 的对象，该属性对应于构造函数本身。没有必要通过 `properties` 参数传递属性描述符来设置它。

使用此 API 定义的函数可以通过 JavaScript 中的 `new` 运算符调用。

通常，在调用 `new` 时，构造函数会创建一个类型为构造函数的新普通对象（例如，构造函数通常是一个全局属性，其原型是一个普通对象）。这个新创建的对象的原型被设置为构造函数的 `prototype` 属性。构造函数运行，如果对象被返回，则 `new` 的结果就是该对象。如果构造函数返回非对象，则返回新创建的对象。

使用 `napi_define_class` 时，允许原生代码返回一个对象，该对象与通过运行构造函数创建的对象不同。为了支持此功能，使用 `napi_define_class` 定义的构造函数的 `prototype` 属性必须是一个包装了原生构造函数的普通对象。然后，当调用构造函数时，Node-API 将把新创建对象的原型设置为这个普通对象。然后，原生构造函数可以返回一个对象，其内部原型与普通对象不同。返回的对象将被 `new` 运算符使用。

例如，使用以下 JavaScript 和原生代码：

```js
'use strict';

const Example = require('bindings')('example').Example;
const example = new Example();
console.log(example instanceof Example);
```

```c
// ...
napi_value ExampleConstructor(napi_env env, napi_callback_info info) {
  napi_value new_target;
  napi_status status = napi_get_new_target(env, info, &new_target);
  assert(status == napi_ok);
  bool is_new_target;
  status = napi_strict_equals(env, new_target, new_target, &is_new_target);
  assert(status == napi_ok);
  if (!is_new_target) {
    // 这发生在 `Example()` 被调用时。
    // 返回一个假实例以允许 `instanceof` 工作。
    napi_value instance;
    status = napi_new_instance(env, new_target, 0, NULL, &instance);
    assert(status == napi_ok);
    return instance;
  }

  // 这是实际的构造函数。
  napi_value this;
  status = napi_get_cb_info(env, info, 0, NULL, &this, NULL);
  assert(status == napi_ok);
  // 返回 `this` 或我们选择的任何其他对象。
  return this;
}
napi_property_descriptor desc = { "Example", NULL, ExampleConstructor, NULL, NULL, NULL, napi_default, NULL };
napi_value result;
napi_define_class(env, "Example", NAPI_AUTO_LENGTH, ExampleConstructor, data, 1, &desc, &result);
```

当原生构造函数运行时，`new_target` 指向由 `napi_define_class` 创建的构造函数。当 `Example` 被调用为函数而不是使用 `new` 时，`new_target` 是 `undefined`。然后，原生构造函数可以检查 `new_target` 是否为构造函数本身，以确定它是作为函数调用还是作为构造函数调用。

当构造函数作为函数调用时，行为完全取决于原生代码。在某些情况下，例如 [`URL`][] 构造函数，当作为函数调用时，它返回一个新对象，就好像它是作为构造函数调用一样。在其他情况下，当作为函数调用时，它可能抛出或返回 `undefined`。

在上面的示例中，当构造函数作为函数调用时，原生代码通过调用构造函数创建一个新实例，然后返回该实例。这允许 `instanceof` 对返回的对象工作，因为它具有正确的原型。当作为构造函数调用时，它返回 `this`，这可能是另一个对象。这允许原生代码返回一个单例或一个先前创建的对象。如果原生代码没有返回任何对象，可以返回 `this` 或任何其他创建的对象。

### `napi_get_prototype`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_prototype(napi_env env,
                               napi_value object,
                               napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要返回其原型的 `napi_value`。这返回相当于 `Object.getPrototypeOf` 的结果（与函数的 `prototype` 属性不同）。
* `[out] result`: 表示给定对象原型的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

### `napi_get_property_names`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property_names(napi_env env,
                                    napi_value object,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`，其中包含给定对象的属性名称字符串。API 可以添加额外的 `undefined` 值到数组中，以防止其成为密集数组，并且可以添加任意索引。有关详细信息，请参阅问题 [#3996](https://github.com/nodejs/node/issues/3996)。

如果 API 成功则返回 `napi_ok`。

此 API 返回给定对象的属性名称数组。`result` 中的属性名称不包含继承的属性。

### `napi_set_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] key`: 要设置的属性的名称。这应该是一个字符串或 `symbol`。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性。

### `napi_get_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] key`: 要检索的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性。

### `napi_has_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性。

### `napi_delete_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_property(napi_env env,
                                 napi_value object,
                                 napi_value key,
                                 bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] key`: 要删除的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该属性，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `key` 属性。

### `napi_has_own_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_has_own_property(napi_env env,
                                  napi_value object,
                                  napi_value key,
                                  bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有自己的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有自己的命名属性。`key` 必须是字符串或 `symbol`。

### `napi_set_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] utf8name`: 要设置的属性的名称。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性，其中属性名称是 UTF8 编码的字符串。

### `napi_get_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] utf8name`: 要检索的属性的名称。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性，其中属性名称是 UTF8 编码的字符串。

### `napi_has_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] utf8name`: 要检查的属性的名称。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性，其中属性名称是 UTF8 编码的字符串。

### `napi_set_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] index`: 要设置的属性的索引。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置元素。

### `napi_get_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] index`: 要检索的属性的索引。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取元素。

### `napi_has_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] index`: 要检查的属性的索引。
* `[out] result`: 对象是否具有要检查的元素。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有索引属性。

### `napi_delete_element`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_element(napi_env env,
                                napi_value object,
                                uint32_t index,
                                bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] index`: 要删除的属性的索引。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该元素，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `index` 属性。

### `napi_define_properties`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_properties(napi_env env,
                                   napi_value object,
                                   size_t property_count,
                                   const napi_property_descriptor* properties);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要设置属性的对象。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组。

如果 API 成功则返回 `napi_ok`。

此方法允许高效定义对象的多个属性。给定的属性描述符数组用于设置对象的属性。此 API 的默认行为类似于 `Object.defineProperties()`。

### `napi_object_freeze`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_freeze(napi_env env,
                               napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要冻结的对象。

如果 API 成功则返回 `napi_ok`。

此方法冻结给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.freeze()`。

### `napi_object_seal`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_seal(napi_env env,
                             napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要密封的对象。

如果 API 成功则返回 `napi_ok`。

此方法密封给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.seal()`。

## 使用 JavaScript 函数

Node-API 提供了一组 API 来从原生代码调用 JavaScript 函数。这些 API 支持两种调用 JavaScript 函数的方式：正常方式，函数类似于 JavaScript 代码调用，以及作为构造函数的方式。

此外，Node-API 提供了一种创建新函数实例的 API。在这种情况下，原生代码提供了作为新函数实现的原生函数。结果是可以通过 JavaScript 代码调用的函数对象。

### `napi_call_function`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_call_function(napi_env env,
                                           napi_value recv,
                                           napi_value func,
                                           size_t argc,
                                           const napi_value* argv,
                                           napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] recv`: 作为函数调用的 `this` 值的 `object`。
* `[in] func`: 要调用的 JavaScript `function`。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法允许从原生代码调用 JavaScript 函数对象。这是用于调用函数的原生 API。一个典型的用法是在操作完成时调用用 JavaScript 编写的回调。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_new_instance`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_new_instance(napi_env env,
                                          napi_value constructor,
                                          size_t argc,
                                          const napi_value* argv,
                                          napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] constructor`: 表示要调用的 JavaScript 函数的 `napi_value`。此 API 必须将构造函数传递给 JavaScript 函数。JavaScript 函数创建新对象并初始化它，然后用作构造函数。有关更多详细信息，请参阅[构造函数定义][]。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示构造函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法用于从原生代码实例化 JavaScript 对象。这相当于在 JavaScript 中执行 `new Constructor()`，其中 `Constructor` 是作为构造函数传递的函数对象。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_define_class`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_class(napi_env env,
                              const char* utf8name,
                              size_t length,
                              napi_callback constructor,
                              void* data,
                              size_t property_count,
                              const napi_property_descriptor* properties,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] utf8name`: 类的名称；这通常是传递给构造函数的名称。
* `[in] length`: 类名称的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[in] constructor`: 处理类实例化和构造的回调函数。这应该是一个静态成员函数，其签名如下所述。
* `[in] data`: 作为 `this` 参数传递给构造函数的任意数据。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组，用于静态方法和类实例化后添加到类原型的属性。有关更多详细信息，请参阅 `napi_property_descriptor`。
* `[out] result`: 表示构造函数的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

定义与 C++ 类相对应的 JavaScript 类，包括：

* 一个 JavaScript 函数，作为类的构造函数。此函数必须与传递给 `napi_define_class` 的 `constructor` 回调相对应。
* 与 C++ 类实例关联的所有静态数据/方法。这些属性被添加到构造函数中。
* 与 C++ 类实例关联的所有非静态数据/方法。这些属性被添加到构造函数的原型中。

C++ `constructor` 回调应该是一个静态函数，在创建 JavaScript 对象时调用，它本身就是一个类的实例。回调的签名如下：

```c
napi_value callback(napi_env env, napi_callback_info info);
```

接收的参数：

* `[in] env`: 调用回调的环境。
* `[in] info`: 回调信息。

回调可以返回以下内容：

* 另一个 `napi_value`，它应该是一个表示新创建的实例的 JavaScript 对象。
* `NULL`，如果构造函数抛出异常。

回调内部，`napi_callback_info` 参数可用于检索传入的参数（相当于每个构造函数接收的 `arguments` 对象）和新的实例的 `this`。

提供给 `napi_define_class` 的 `data` 指针可以在回调内部作为 `info` 参数传递给 [`napi_get_cb_info`][] 时作为新的 `this` 指针访问。

构造函数的原型被自动设置为具有属性 `constructor` 的对象，该属性对应于构造函数本身。没有必要通过 `properties` 参数传递属性描述符来设置它。

使用此 API 定义的函数可以通过 JavaScript 中的 `new` 运算符调用。

通常，在调用 `new` 时，构造函数会创建一个类型为构造函数的新普通对象（例如，构造函数通常是一个全局属性，其原型是一个普通对象）。这个新创建的对象的原型被设置为构造函数的 `prototype` 属性。构造函数运行，如果对象被返回，则 `new` 的结果就是该对象。如果构造函数返回非对象，则返回新创建的对象。

使用 `napi_define_class` 时，允许原生代码返回一个对象，该对象与通过运行构造函数创建的对象不同。为了支持此功能，使用 `napi_define_class` 定义的构造函数的 `prototype` 属性必须是一个包装了原生构造函数的普通对象。然后，当调用构造函数时，Node-API 将把新创建对象的原型设置为这个普通对象。然后，原生构造函数可以返回一个对象，其内部原型与普通对象不同。返回的对象将被 `new` 运算符使用。

例如，使用以下 JavaScript 和原生代码：

```js
'use strict';

const Example = require('bindings')('example').Example;
const example = new Example();
console.log(example instanceof Example);
```

```c
// ...
napi_value ExampleConstructor(napi_env env, napi_callback_info info) {
  napi_value new_target;
  napi_status status = napi_get_new_target(env, info, &new_target);
  assert(status == napi_ok);
  bool is_new_target;
  status = napi_strict_equals(env, new_target, new_target, &is_new_target);
  assert(status == napi_ok);
  if (!is_new_target) {
    // 这发生在 `Example()` 被调用时。
    // 返回一个假实例以允许 `instanceof` 工作。
    napi_value instance;
    status = napi_new_instance(env, new_target, 0, NULL, &instance);
    assert(status == napi_ok);
    return instance;
  }

  // 这是实际的构造函数。
  napi_value this;
  status = napi_get_cb_info(env, info, 0, NULL, &this, NULL);
  assert(status == napi_ok);
  // 返回 `this` 或我们选择的任何其他对象。
  return this;
}
napi_property_descriptor desc = { "Example", NULL, ExampleConstructor, NULL, NULL, NULL, napi_default, NULL };
napi_value result;
napi_define_class(env, "Example", NAPI_AUTO_LENGTH, ExampleConstructor, data, 1, &desc, &result);
```

当原生构造函数运行时，`new_target` 指向由 `napi_define_class` 创建的构造函数。当 `Example` 被调用为函数而不是使用 `new` 时，`new_target` 是 `undefined`。然后，原生构造函数可以检查 `new_target` 是否为构造函数本身，以确定它是作为函数调用还是作为构造函数调用。

当构造函数作为函数调用时，行为完全取决于原生代码。在某些情况下，例如 [`URL`][] 构造函数，当作为函数调用时，它返回一个新对象，就好像它是作为构造函数调用一样。在其他情况下，当作为函数调用时，它可能抛出或返回 `undefined`。

在上面的示例中，当构造函数作为函数调用时，原生代码通过调用构造函数创建一个新实例，然后返回该实例。这允许 `instanceof` 对返回的对象工作，因为它具有正确的原型。当作为构造函数调用时，它返回 `this`，这可能是另一个对象。这允许原生代码返回一个单例或一个先前创建的对象。如果原生代码没有返回任何对象，可以返回 `this` 或任何其他创建的对象。

### `napi_get_prototype`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_prototype(napi_env env,
                               napi_value object,
                               napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要返回其原型的 `napi_value`。这返回相当于 `Object.getPrototypeOf` 的结果（与函数的 `prototype` 属性不同）。
* `[out] result`: 表示给定对象原型的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

### `napi_get_property_names`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property_names(napi_env env,
                                    napi_value object,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`，其中包含给定对象的属性名称字符串。API 可以添加额外的 `undefined` 值到数组中，以防止其成为密集数组，并且可以添加任意索引。有关详细信息，请参阅问题 [#3996](https://github.com/nodejs/node/issues/3996)。

如果 API 成功则返回 `napi_ok`。

此 API 返回给定对象的属性名称数组。`result` 中的属性名称不包含继承的属性。

### `napi_set_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] key`: 要设置的属性的名称。这应该是一个字符串或 `symbol`。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性。

### `napi_get_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] key`: 要检索的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性。

### `napi_has_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性。

### `napi_delete_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_property(napi_env env,
                                 napi_value object,
                                 napi_value key,
                                 bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] key`: 要删除的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该属性，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `key` 属性。

### `napi_has_own_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_has_own_property(napi_env env,
                                  napi_value object,
                                  napi_value key,
                                  bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有自己的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有自己的命名属性。`key` 必须是字符串或 `symbol`。

### `napi_set_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] utf8name`: 要设置的属性的名称。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性，其中属性名称是 UTF8 编码的字符串。

### `napi_get_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] utf8name`: 要检索的属性的名称。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性，其中属性名称是 UTF8 编码的字符串。

### `napi_has_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] utf8name`: 要检查的属性的名称。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性，其中属性名称是 UTF8 编码的字符串。

### `napi_set_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] index`: 要设置的属性的索引。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置元素。

### `napi_get_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] index`: 要检索的属性的索引。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取元素。

### `napi_has_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] index`: 要检查的属性的索引。
* `[out] result`: 对象是否具有要检查的元素。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有索引属性。

### `napi_delete_element`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_element(napi_env env,
                                napi_value object,
                                uint32_t index,
                                bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] index`: 要删除的属性的索引。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该元素，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `index` 属性。

### `napi_define_properties`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_properties(napi_env env,
                                   napi_value object,
                                   size_t property_count,
                                   const napi_property_descriptor* properties);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要设置属性的对象。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组。

如果 API 成功则返回 `napi_ok`。

此方法允许高效定义对象的多个属性。给定的属性描述符数组用于设置对象的属性。此 API 的默认行为类似于 `Object.defineProperties()`。

### `napi_object_freeze`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_freeze(napi_env env,
                               napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要冻结的对象。

如果 API 成功则返回 `napi_ok`。

此方法冻结给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.freeze()`。

### `napi_object_seal`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_seal(napi_env env,
                             napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要密封的对象。

如果 API 成功则返回 `napi_ok`。

此方法密封给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.seal()`。

## 使用 JavaScript 函数

Node-API 提供了一组 API 来从原生代码调用 JavaScript 函数。这些 API 支持两种调用 JavaScript 函数的方式：正常方式，函数类似于 JavaScript 代码调用，以及作为构造函数的方式。

此外，Node-API 提供了一种创建新函数实例的 API。在这种情况下，原生代码提供了作为新函数实现的原生函数。结果是可以通过 JavaScript 代码调用的函数对象。

### `napi_call_function`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_call_function(napi_env env,
                                           napi_value recv,
                                           napi_value func,
                                           size_t argc,
                                           const napi_value* argv,
                                           napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] recv`: 作为函数调用的 `this` 值的 `object`。
* `[in] func`: 要调用的 JavaScript `function`。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法允许从原生代码调用 JavaScript 函数对象。这是用于调用函数的原生 API。一个典型的用法是在操作完成时调用用 JavaScript 编写的回调。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_new_instance`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_new_instance(napi_env env,
                                          napi_value constructor,
                                          size_t argc,
                                          const napi_value* argv,
                                          napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] constructor`: 表示要调用的 JavaScript 函数的 `napi_value`。此 API 必须将构造函数传递给 JavaScript 函数。JavaScript 函数创建新对象并初始化它，然后用作构造函数。有关更多详细信息，请参阅[构造函数定义][]。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示构造函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法用于从原生代码实例化 JavaScript 对象。这相当于在 JavaScript 中执行 `new Constructor()`，其中 `Constructor` 是作为构造函数传递的函数对象。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_define_class`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_class(napi_env env,
                              const char* utf8name,
                              size_t length,
                              napi_callback constructor,
                              void* data,
                              size_t property_count,
                              const napi_property_descriptor* properties,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] utf8name`: 类的名称；这通常是传递给构造函数的名称。
* `[in] length`: 类名称的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[in] constructor`: 处理类实例化和构造的回调函数。这应该是一个静态成员函数，其签名如下所述。
* `[in] data`: 作为 `this` 参数传递给构造函数的任意数据。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组，用于静态方法和类实例化后添加到类原型的属性。有关更多详细信息，请参阅 `napi_property_descriptor`。
* `[out] result`: 表示构造函数的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

定义与 C++ 类相对应的 JavaScript 类，包括：

* 一个 JavaScript 函数，作为类的构造函数。此函数必须与传递给 `napi_define_class` 的 `constructor` 回调相对应。
* 与 C++ 类实例关联的所有静态数据/方法。这些属性被添加到构造函数中。
* 与 C++ 类实例关联的所有非静态数据/方法。这些属性被添加到构造函数的原型中。

C++ `constructor` 回调应该是一个静态函数，在创建 JavaScript 对象时调用，它本身就是一个类的实例。回调的签名如下：

```c
napi_value callback(napi_env env, napi_callback_info info);
```

接收的参数：

* `[in] env`: 调用回调的环境。
* `[in] info`: 回调信息。

回调可以返回以下内容：

* 另一个 `napi_value`，它应该是一个表示新创建的实例的 JavaScript 对象。
* `NULL`，如果构造函数抛出异常。

回调内部，`napi_callback_info` 参数可用于检索传入的参数（相当于每个构造函数接收的 `arguments` 对象）和新的实例的 `this`。

提供给 `napi_define_class` 的 `data` 指针可以在回调内部作为 `info` 参数传递给 [`napi_get_cb_info`][] 时作为新的 `this` 指针访问。

构造函数的原型被自动设置为具有属性 `constructor` 的对象，该属性对应于构造函数本身。没有必要通过 `properties` 参数传递属性描述符来设置它。

使用此 API 定义的函数可以通过 JavaScript 中的 `new` 运算符调用。

通常，在调用 `new` 时，构造函数会创建一个类型为构造函数的新普通对象（例如，构造函数通常是一个全局属性，其原型是一个普通对象）。这个新创建的对象的原型被设置为构造函数的 `prototype` 属性。构造函数运行，如果对象被返回，则 `new` 的结果就是该对象。如果构造函数返回非对象，则返回新创建的对象。

使用 `napi_define_class` 时，允许原生代码返回一个对象，该对象与通过运行构造函数创建的对象不同。为了支持此功能，使用 `napi_define_class` 定义的构造函数的 `prototype` 属性必须是一个包装了原生构造函数的普通对象。然后，当调用构造函数时，Node-API 将把新创建对象的原型设置为这个普通对象。然后，原生构造函数可以返回一个对象，其内部原型与普通对象不同。返回的对象将被 `new` 运算符使用。

例如，使用以下 JavaScript 和原生代码：

```js
'use strict';

const Example = require('bindings')('example').Example;
const example = new Example();
console.log(example instanceof Example);
```

```c
// ...
napi_value ExampleConstructor(napi_env env, napi_callback_info info) {
  napi_value new_target;
  napi_status status = napi_get_new_target(env, info, &new_target);
  assert(status == napi_ok);
  bool is_new_target;
  status = napi_strict_equals(env, new_target, new_target, &is_new_target);
  assert(status == napi_ok);
  if (!is_new_target) {
    // 这发生在 `Example()` 被调用时。
    // 返回一个假实例以允许 `instanceof` 工作。
    napi_value instance;
    status = napi_new_instance(env, new_target, 0, NULL, &instance);
    assert(status == napi_ok);
    return instance;
  }

  // 这是实际的构造函数。
  napi_value this;
  status = napi_get_cb_info(env, info, 0, NULL, &this, NULL);
  assert(status == napi_ok);
  // 返回 `this` 或我们选择的任何其他对象。
  return this;
}
napi_property_descriptor desc = { "Example", NULL, ExampleConstructor, NULL, NULL, NULL, napi_default, NULL };
napi_value result;
napi_define_class(env, "Example", NAPI_AUTO_LENGTH, ExampleConstructor, data, 1, &desc, &result);
```

当原生构造函数运行时，`new_target` 指向由 `napi_define_class` 创建的构造函数。当 `Example` 被调用为函数而不是使用 `new` 时，`new_target` 是 `undefined`。然后，原生构造函数可以检查 `new_target` 是否为构造函数本身，以确定它是作为函数调用还是作为构造函数调用。

当构造函数作为函数调用时，行为完全取决于原生代码。在某些情况下，例如 [`URL`][] 构造函数，当作为函数调用时，它返回一个新对象，就好像它是作为构造函数调用一样。在其他情况下，当作为函数调用时，它可能抛出或返回 `undefined`。

在上面的示例中，当构造函数作为函数调用时，原生代码通过调用构造函数创建一个新实例，然后返回该实例。这允许 `instanceof` 对返回的对象工作，因为它具有正确的原型。当作为构造函数调用时，它返回 `this`，这可能是另一个对象。这允许原生代码返回一个单例或一个先前创建的对象。如果原生代码没有返回任何对象，可以返回 `this` 或任何其他创建的对象。

### `napi_get_prototype`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_prototype(napi_env env,
                               napi_value object,
                               napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要返回其原型的 `napi_value`。这返回相当于 `Object.getPrototypeOf` 的结果（与函数的 `prototype` 属性不同）。
* `[out] result`: 表示给定对象原型的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

### `napi_get_property_names`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property_names(napi_env env,
                                    napi_value object,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`，其中包含给定对象的属性名称字符串。API 可以添加额外的 `undefined` 值到数组中，以防止其成为密集数组，并且可以添加任意索引。有关详细信息，请参阅问题 [#3996](https://github.com/nodejs/node/issues/3996)。

如果 API 成功则返回 `napi_ok`。

此 API 返回给定对象的属性名称数组。`result` 中的属性名称不包含继承的属性。

### `napi_set_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] key`: 要设置的属性的名称。这应该是一个字符串或 `symbol`。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性。

### `napi_get_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] key`: 要检索的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性。

### `napi_has_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性。

### `napi_delete_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_property(napi_env env,
                                 napi_value object,
                                 napi_value key,
                                 bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] key`: 要删除的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该属性，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `key` 属性。

### `napi_has_own_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_has_own_property(napi_env env,
                                  napi_value object,
                                  napi_value key,
                                  bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有自己的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有自己的命名属性。`key` 必须是字符串或 `symbol`。

### `napi_set_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] utf8name`: 要设置的属性的名称。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性，其中属性名称是 UTF8 编码的字符串。

### `napi_get_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] utf8name`: 要检索的属性的名称。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性，其中属性名称是 UTF8 编码的字符串。

### `napi_has_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] utf8name`: 要检查的属性的名称。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性，其中属性名称是 UTF8 编码的字符串。

### `napi_set_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] index`: 要设置的属性的索引。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置元素。

### `napi_get_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] index`: 要检索的属性的索引。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取元素。

### `napi_has_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] index`: 要检查的属性的索引。
* `[out] result`: 对象是否具有要检查的元素。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有索引属性。

### `napi_delete_element`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_element(napi_env env,
                                napi_value object,
                                uint32_t index,
                                bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] index`: 要删除的属性的索引。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该元素，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `index` 属性。

### `napi_define_properties`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_properties(napi_env env,
                                   napi_value object,
                                   size_t property_count,
                                   const napi_property_descriptor* properties);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要设置属性的对象。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组。

如果 API 成功则返回 `napi_ok`。

此方法允许高效定义对象的多个属性。给定的属性描述符数组用于设置对象的属性。此 API 的默认行为类似于 `Object.defineProperties()`。

### `napi_object_freeze`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_freeze(napi_env env,
                               napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要冻结的对象。

如果 API 成功则返回 `napi_ok`。

此方法冻结给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.freeze()`。

### `napi_object_seal`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_seal(napi_env env,
                             napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要密封的对象。

如果 API 成功则返回 `napi_ok`。

此方法密封给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.seal()`。

## 使用 JavaScript 函数

Node-API 提供了一组 API 来从原生代码调用 JavaScript 函数。这些 API 支持两种调用 JavaScript 函数的方式：正常方式，函数类似于 JavaScript 代码调用，以及作为构造函数的方式。

此外，Node-API 提供了一种创建新函数实例的 API。在这种情况下，原生代码提供了作为新函数实现的原生函数。结果是可以通过 JavaScript 代码调用的函数对象。

### `napi_call_function`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_call_function(napi_env env,
                                           napi_value recv,
                                           napi_value func,
                                           size_t argc,
                                           const napi_value* argv,
                                           napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] recv`: 作为函数调用的 `this` 值的 `object`。
* `[in] func`: 要调用的 JavaScript `function`。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法允许从原生代码调用 JavaScript 函数对象。这是用于调用函数的原生 API。一个典型的用法是在操作完成时调用用 JavaScript 编写的回调。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_new_instance`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_new_instance(napi_env env,
                                          napi_value constructor,
                                          size_t argc,
                                          const napi_value* argv,
                                          napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] constructor`: 表示要调用的 JavaScript 函数的 `napi_value`。此 API 必须将构造函数传递给 JavaScript 函数。JavaScript 函数创建新对象并初始化它，然后用作构造函数。有关更多详细信息，请参阅[构造函数定义][]。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示构造函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法用于从原生代码实例化 JavaScript 对象。这相当于在 JavaScript 中执行 `new Constructor()`，其中 `Constructor` 是作为构造函数传递的函数对象。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_define_class`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_class(napi_env env,
                              const char* utf8name,
                              size_t length,
                              napi_callback constructor,
                              void* data,
                              size_t property_count,
                              const napi_property_descriptor* properties,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] utf8name`: 类的名称；这通常是传递给构造函数的名称。
* `[in] length`: 类名称的长度（字节），如果是空终止的则为 `NAPI_AUTO_LENGTH`。
* `[in] constructor`: 处理类实例化和构造的回调函数。这应该是一个静态成员函数，其签名如下所述。
* `[in] data`: 作为 `this` 参数传递给构造函数的任意数据。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组，用于静态方法和类实例化后添加到类原型的属性。有关更多详细信息，请参阅 `napi_property_descriptor`。
* `[out] result`: 表示构造函数的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

定义与 C++ 类相对应的 JavaScript 类，包括：

* 一个 JavaScript 函数，作为类的构造函数。此函数必须与传递给 `napi_define_class` 的 `constructor` 回调相对应。
* 与 C++ 类实例关联的所有静态数据/方法。这些属性被添加到构造函数中。
* 与 C++ 类实例关联的所有非静态数据/方法。这些属性被添加到构造函数的原型中。

C++ `constructor` 回调应该是一个静态函数，在创建 JavaScript 对象时调用，它本身就是一个类的实例。回调的签名如下：

```c
napi_value callback(napi_env env, napi_callback_info info);
```

接收的参数：

* `[in] env`: 调用回调的环境。
* `[in] info`: 回调信息。

回调可以返回以下内容：

* 另一个 `napi_value`，它应该是一个表示新创建的实例的 JavaScript 对象。
* `NULL`，如果构造函数抛出异常。

回调内部，`napi_callback_info` 参数可用于检索传入的参数（相当于每个构造函数接收的 `arguments` 对象）和新的实例的 `this`。

提供给 `napi_define_class` 的 `data` 指针可以在回调内部作为 `info` 参数传递给 [`napi_get_cb_info`][] 时作为新的 `this` 指针访问。

构造函数的原型被自动设置为具有属性 `constructor` 的对象，该属性对应于构造函数本身。没有必要通过 `properties` 参数传递属性描述符来设置它。

使用此 API 定义的函数可以通过 JavaScript 中的 `new` 运算符调用。

通常，在调用 `new` 时，构造函数会创建一个类型为构造函数的新普通对象（例如，构造函数通常是一个全局属性，其原型是一个普通对象）。这个新创建的对象的原型被设置为构造函数的 `prototype` 属性。构造函数运行，如果对象被返回，则 `new` 的结果就是该对象。如果构造函数返回非对象，则返回新创建的对象。

使用 `napi_define_class` 时，允许原生代码返回一个对象，该对象与通过运行构造函数创建的对象不同。为了支持此功能，使用 `napi_define_class` 定义的构造函数的 `prototype` 属性必须是一个包装了原生构造函数的普通对象。然后，当调用构造函数时，Node-API 将把新创建对象的原型设置为这个普通对象。然后，原生构造函数可以返回一个对象，其内部原型与普通对象不同。返回的对象将被 `new` 运算符使用。

例如，使用以下 JavaScript 和原生代码：

```js
'use strict';

const Example = require('bindings')('example').Example;
const example = new Example();
console.log(example instanceof Example);
```

```c
// ...
napi_value ExampleConstructor(napi_env env, napi_callback_info info) {
  napi_value new_target;
  napi_status status = napi_get_new_target(env, info, &new_target);
  assert(status == napi_ok);
  bool is_new_target;
  status = napi_strict_equals(env, new_target, new_target, &is_new_target);
  assert(status == napi_ok);
  if (!is_new_target) {
    // 这发生在 `Example()` 被调用时。
    // 返回一个假实例以允许 `instanceof` 工作。
    napi_value instance;
    status = napi_new_instance(env, new_target, 0, NULL, &instance);
    assert(status == napi_ok);
    return instance;
  }

  // 这是实际的构造函数。
  napi_value this;
  status = napi_get_cb_info(env, info, 0, NULL, &this, NULL);
  assert(status == napi_ok);
  // 返回 `this` 或我们选择的任何其他对象。
  return this;
}
napi_property_descriptor desc = { "Example", NULL, ExampleConstructor, NULL, NULL, NULL, napi_default, NULL };
napi_value result;
napi_define_class(env, "Example", NAPI_AUTO_LENGTH, ExampleConstructor, data, 1, &desc, &result);
```

当原生构造函数运行时，`new_target` 指向由 `napi_define_class` 创建的构造函数。当 `Example` 被调用为函数而不是使用 `new` 时，`new_target` 是 `undefined`。然后，原生构造函数可以检查 `new_target` 是否为构造函数本身，以确定它是作为函数调用还是作为构造函数调用。

当构造函数作为函数调用时，行为完全取决于原生代码。在某些情况下，例如 [`URL`][] 构造函数，当作为函数调用时，它返回一个新对象，就好像它是作为构造函数调用一样。在其他情况下，当作为函数调用时，它可能抛出或返回 `undefined`。

在上面的示例中，当构造函数作为函数调用时，原生代码通过调用构造函数创建一个新实例，然后返回该实例。这允许 `instanceof` 对返回的对象工作，因为它具有正确的原型。当作为构造函数调用时，它返回 `this`，这可能是另一个对象。这允许原生代码返回一个单例或一个先前创建的对象。如果原生代码没有返回任何对象，可以返回 `this` 或任何其他创建的对象。

### `napi_get_prototype`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_prototype(napi_env env,
                               napi_value object,
                               napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要返回其原型的 `napi_value`。这返回相当于 `Object.getPrototypeOf` 的结果（与函数的 `prototype` 属性不同）。
* `[out] result`: 表示给定对象原型的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

### `napi_get_property_names`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property_names(napi_env env,
                                    napi_value object,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[out] result`: 表示 JavaScript `Array` 的 `napi_value`，其中包含给定对象的属性名称字符串。API 可以添加额外的 `undefined` 值到数组中，以防止其成为密集数组，并且可以添加任意索引。有关详细信息，请参阅问题 [#3996](https://github.com/nodejs/node/issues/3996)。

如果 API 成功则返回 `napi_ok`。

此 API 返回给定对象的属性名称数组。`result` 中的属性名称不包含继承的属性。

### `napi_set_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] key`: 要设置的属性的名称。这应该是一个字符串或 `symbol`。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性。

### `napi_get_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] key`: 要检索的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性。

### `napi_has_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_property(napi_env env,
                              napi_value object,
                              napi_value key,
                              bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性。

### `napi_delete_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_property(napi_env env,
                                 napi_value object,
                                 napi_value key,
                                 bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] key`: 要删除的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该属性，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `key` 属性。

### `napi_has_own_property`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_has_own_property(napi_env env,
                                  napi_value object,
                                  napi_value key,
                                  bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] key`: 要检查的属性的名称。这应该是一个字符串或 `symbol`。
* `[out] result`: 对象是否具有自己的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有自己的命名属性。`key` 必须是字符串或 `symbol`。

### `napi_set_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] utf8name`: 要设置的属性的名称。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置属性，其中属性名称是 UTF8 编码的字符串。

### `napi_get_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] utf8name`: 要检索的属性的名称。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取属性，其中属性名称是 UTF8 编码的字符串。

### `napi_has_named_property`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_named_property(napi_env env,
                                    napi_value object,
                                    const char* utf8name,
                                    bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] utf8name`: 要检查的属性的名称。
* `[out] result`: 对象是否具有要检查的属性。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有命名属性，其中属性名称是 UTF8 编码的字符串。

### `napi_set_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_set_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value value);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要在其上设置属性的对象。
* `[in] index`: 要设置的属性的索引。
* `[in] value`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 在 `Object` 上设置元素。

### `napi_get_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中检索属性的对象。
* `[in] index`: 要检索的属性的索引。
* `[out] result`: 属性值。

如果 API 成功则返回 `napi_ok`。

此 API 从 `Object` 获取元素。

### `napi_has_element`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_has_element(napi_env env,
                             napi_value object,
                             uint32_t index,
                             bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要查询的对象。
* `[in] index`: 要检查的属性的索引。
* `[out] result`: 对象是否具有要检查的元素。

如果 API 成功则返回 `napi_ok`。

此 API 检查 `Object` 是否具有索引属性。

### `napi_delete_element`

<!-- YAML
added: v8.2.0
napiVersion: 1
-->

```c
napi_status napi_delete_element(napi_env env,
                                napi_value object,
                                uint32_t index,
                                bool* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要从中删除属性的对象。
* `[in] index`: 要删除的属性的索引。
* `[out] result`: 删除是否成功。`result` 可以为 `true`，如果对象没有该元素，则为 `false`。

如果 API 成功则返回 `napi_ok`。

此 API 尝试从 `object` 中删除自己的 `index` 属性。

### `napi_define_properties`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_properties(napi_env env,
                                   napi_value object,
                                   size_t property_count,
                                   const napi_property_descriptor* properties);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要设置属性的对象。
* `[in] property_count`: `properties` 数组中的元素数量。
* `[in] properties`: 属性描述符数组。

如果 API 成功则返回 `napi_ok`。

此方法允许高效定义对象的多个属性。给定的属性描述符数组用于设置对象的属性。此 API 的默认行为类似于 `Object.defineProperties()`。

### `napi_object_freeze`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_freeze(napi_env env,
                               napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要冻结的对象。

如果 API 成功则返回 `napi_ok`。

此方法冻结给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.freeze()`。

### `napi_object_seal`

<!-- YAML
added:
  - v18.7.0
  - v16.17.0
napiVersion: 9
-->

```c
napi_status napi_object_seal(napi_env env,
                             napi_value object);
```

* `[in] env`: 调用 API 的环境。
* `[in] object`: 要密封的对象。

如果 API 成功则返回 `napi_ok`。

此方法密封给定的对象。这阻止了向对象添加新属性，并标记所有现有属性为不可配置。有关更多详细信息，请参阅 `Object.seal()`。

## 使用 JavaScript 函数

Node-API 提供了一组 API 来从原生代码调用 JavaScript 函数。这些 API 支持两种调用 JavaScript 函数的方式：正常方式，函数类似于 JavaScript 代码调用，以及作为构造函数的方式。

此外，Node-API 提供了一种创建新函数实例的 API。在这种情况下，原生代码提供了作为新函数实现的原生函数。结果是可以通过 JavaScript 代码调用的函数对象。

### `napi_call_function`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_call_function(napi_env env,
                                           napi_value recv,
                                           napi_value func,
                                           size_t argc,
                                           const napi_value* argv,
                                           napi_value* result);
```

* `[in] env`: 调用 API 的环境。
* `[in] recv`: 作为函数调用的 `this` 值的 `object`。
* `[in] func`: 要调用的 JavaScript `function`。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: JavaScript 值数组，表示函数的参数。如果 `argc` 为零，此参数可以为 `NULL`。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法允许从原生代码调用 JavaScript 函数对象。这是用于调用函数的原生 API。一个典型的用法是在操作完成时调用用 JavaScript 编写的回调。

JavaScript 函数在 ECMAScript 语言规范的[函数对象部分][]中描述。

### `napi_define_class`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_define_class(napi_env env,
                              const char* utf8name,
                              size_t length,
                              napi_callback constructor,
                              void* data,
                              size_t property_count,
                              const napi_property_descriptor* properties,
                              napi_value* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] utf8name`: JavaScript 构造函数名称。为清晰起见，在包装 C++ 类时建议使用 C++ 类名。
* `[in] length`: `utf8name` 的字节长度，如果以 null 结尾则为 `NAPI_AUTO_LENGTH`。
* `[in] constructor`: 处理类实例构造的回调函数。当包装 C++ 类时，此方法必须是一个具有 [`napi_callback`][] 签名的静态成员。不能使用 C++ 类构造函数。[`napi_callback`][] 提供了更多细节。
* `[in] data`: 可选数据，将作为回调信息的 `data` 属性传递给构造函数回调。
* `[in] property_count`: `properties` 数组参数中的项目数。
* `[in] properties`: 属性描述符数组，描述类上的静态和实例数据属性、访问器和方法。参见 `napi_property_descriptor`。
* `[out] result`: 一个 `napi_value`，表示该类的构造函数。

如果 API 成功则返回 `napi_ok`。

定义一个 JavaScript 类，包括：

* 一个具有类名的 JavaScript 构造函数。当包装相应的 C++ 类时，通过 `constructor` 传递的回调可用于实例化一个新的 C++ 类实例，然后可以使用 [`napi_wrap`][] 将其放入正在构造的 JavaScript 对象实例中。
* 构造函数函数上的属性，其实现可以调用 C++ 类的相应*静态*数据属性、访问器和方法（由具有 `napi_static` 属性的属性描述符定义）。
* 构造函数函数的 `prototype` 对象上的属性。当包装 C++ 类时，可以在检索到通过 [`napi_unwrap`][] 放入 JavaScript 对象实例中的 C++ 类实例后，从属性描述符中给出的静态函数调用 C++ 类的*非静态*数据属性、访问器和方法（不带 `napi_static` 属性）。

当包装 C++ 类时，通过 `constructor` 传递的 C++ 构造函数回调应该是类上的一个静态方法，该方法调用实际的类构造函数，然后将新的 C++ 实例包装在 JavaScript 对象中，并返回包装对象。有关详细信息，请参阅 [`napi_wrap`][]。

从 [`napi_define_class`][] 返回的 JavaScript 构造函数通常会被保存并在以后用于从本地代码构造类的新实例，和/或检查提供的值是否为该类的实例。在这种情况下，为了防止函数值被垃圾回收，可以使用 [`napi_create_reference`][] 创建一个强持久引用，确保引用计数保持 >= 1。

任何通过 `data` 参数或通过 `napi_property_descriptor` 数组项的 `data` 字段传递给此 API 的非 `NULL` 数据都可以与结果 JavaScript 构造函数（在 `result` 参数中返回）关联，并在类被垃圾回收时通过将 JavaScript 函数和数据传递给 [`napi_add_finalizer`][] 来释放。

### `napi_wrap`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_wrap(napi_env env,
                      napi_value js_object,
                      void* native_object,
                      napi_finalize finalize_cb,
                      void* finalize_hint,
                      napi_ref* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] js_object`: 将作为本地对象包装器的 JavaScript 对象。
* `[in] native_object`: 将被包装在 JavaScript 对象中的本地实例。
* `[in] finalize_cb`: 可选的本地回调，当 JavaScript 对象被垃圾回收时可用于释放本地实例。[`napi_finalize`][] 提供了更多细节。
* `[in] finalize_hint`: 传递给最终化回调的可选上下文提示。
* `[out] result`: 指向包装对象的可选引用。

如果 API 成功则返回 `napi_ok`。

将本地实例包装在 JavaScript 对象中。之后可以使用 `napi_unwrap()` 检索本地实例。

当 JavaScript 代码调用使用 `napi_define_class()` 定义的类的构造函数时，会调用该构造函数的 `napi_callback`。在构造了本地类的实例之后，回调必须然后调用 `napi_wrap()` 将新构造的实例包装在已经创建的 JavaScript 对象中，该对象是构造函数回调的 `this` 参数。（该 `this` 对象是从构造函数的 `prototype` 创建的，因此它已经具有所有实例属性和方法的定义。）

通常，在包装类实例时，应该提供一个最终化回调，该回调简单地删除作为最终化回调的 `data` 参数接收的本地实例。

可选返回的引用最初是一个弱引用，意味着它的引用计数为 0。通常，在需要实例保持有效的异步操作期间，此引用计数会临时增加。

*注意*：如果获得了可选的返回引用，则仅应在最终化回调调用时通过 [`napi_delete_reference`][] 删除它。如果在此之前删除，则最终化回调可能永远不会被调用。因此，在获取引用时，还需要一个最终化回调以实现引用的正确处置。

最终化回调可能会被延迟，这会产生一个窗口期，在此期间对象已被垃圾回收（弱引用无效）但最终化器尚未被调用。当在 `napi_wrap()` 返回的弱引用上使用 `napi_get_reference_value()` 时，您仍应处理空结果。

在对象上第二次调用 `napi_wrap()` 将返回错误。要将另一个本地实例与对象关联，请首先使用 `napi_remove_wrap()`。

### `napi_unwrap`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_unwrap(napi_env env,
                        napi_value js_object,
                        void** result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] js_object`: 与本地实例关联的对象。
* `[out] result`: 指向包装的本地实例的指针。

如果 API 成功则返回 `napi_ok`。

检索之前使用 `napi_wrap()` 包装在 JavaScript 对象中的本地实例。

当 JavaScript 代码在类上调用方法或属性访问器时，会调用相应的 `napi_callback`。如果回调是针对实例方法或访问器的，则回调的 `this` 参数是包装器对象；然后可以通过在包装器对象上调用 `napi_unwrap()` 来获得调用目标的包装 C++ 实例。

### `napi_remove_wrap`

<!-- YAML
added: v8.5.0
napiVersion: 1
-->

```c
napi_status napi_remove_wrap(napi_env env,
                             napi_value js_object,
                             void** result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] js_object`: 与本地实例关联的对象。
* `[out] result`: 指向包装的本地实例的指针。

如果 API 成功则返回 `napi_ok`。

检索之前使用 `napi_wrap()` 包装在 JavaScript 对象 `js_object` 中的本地实例，并移除包装。如果与包装关联了最终化回调，则当 JavaScript 对象被垃圾回收时，它将不再被调用。

### `napi_type_tag_object`

<!-- YAML
added:
  - v14.8.0
  - v12.19.0
napiVersion: 8
-->

```c
napi_status napi_type_tag_object(napi_env env,
                                 napi_value js_object,
                                 const napi_type_tag* type_tag);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] js_object`: 要标记的 JavaScript 对象或 [external][]。
* `[in] type_tag`: 用于标记对象的标签。

如果 API 成功则返回 `napi_ok`。

将 `type_tag` 指针的值与 JavaScript 对象或 [external][] 关联。然后可以使用 `napi_check_object_type_tag()` 来比较附加到对象的标签与插件拥有的标签，以确保对象具有正确的类型。

如果对象已经有关联的类型标签，此 API 将返回 `napi_invalid_arg`。

### `napi_check_object_type_tag`

<!-- YAML
added:
  - v14.8.0
  - v12.19.0
napiVersion: 8
-->

```c
napi_status napi_check_object_type_tag(napi_env env,
                                       napi_value js_object,
                                       const napi_type_tag* type_tag,
                                       bool* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] js_object`: 要检查类型标签的 JavaScript 对象或 [external][]。
* `[in] type_tag`: 用于与对象上找到的任何标签进行比较的标签。
* `[out] result`: 给定的类型标签是否与对象上的类型标签匹配。如果在对象上未找到类型标签，也返回 `false`。

如果 API 成功则返回 `napi_ok`。

将作为 `type_tag` 给出的指针与在 `js_object` 上找到的任何指针进行比较。如果在 `js_object` 上未找到标签，或者找到了标签但与 `type_tag` 不匹配，则 `result` 设置为 `false`。如果找到标签且与 `type_tag` 匹配，则 `result` 设置为 `true`。

### `napi_add_finalizer`

<!-- YAML
added: v8.0.0
napiVersion: 5
-->

```c
napi_status napi_add_finalizer(napi_env env,
                               napi_value js_object,
                               void* finalize_data,
                               node_api_basic_finalize finalize_cb,
                               void* finalize_hint,
                               napi_ref* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] js_object`: 将附加本地数据的 JavaScript 对象。
* `[in] finalize_data`: 要传递给 `finalize_cb` 的可选数据。
* `[in] finalize_cb`: 当 JavaScript 对象被垃圾回收时用于释放本地数据的本地回调。[`napi_finalize`][] 提供了更多细节。
* `[in] finalize_hint`: 传递给最终化回调的可选上下文提示。
* `[out] result`: 对 JavaScript 对象的可选引用。

如果 API 成功则返回 `napi_ok`。

添加一个 `napi_finalize` 回调，当 `js_object` 中的 JavaScript 对象被垃圾回收时将调用该回调。

此 API 可以在单个 JavaScript 对象上多次调用。

*注意*：如果获得了可选的返回引用，则仅应在最终化回调调用时通过 [`napi_delete_reference`][] 删除它。如果在此之前删除，则最终化回调可能永远不会被调用。因此，在获取引用时，还需要一个最终化回调以实现引用的正确处置。

#### `node_api_post_finalizer`

<!-- YAML
added:
  - v21.0.0
  - v20.10.0
  - v18.19.0
-->

> Stability: 1 - Experimental

```c
napi_status node_api_post_finalizer(node_api_basic_env env,
                                    napi_finalize finalize_cb,
                                    void* finalize_data,
                                    void* finalize_hint);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] finalize_cb`: 当 JavaScript 对象被垃圾回收时用于释放本地数据的本地回调。[`napi_finalize`][] 提供了更多细节。
* `[in] finalize_data`: 要传递给 `finalize_cb` 的可选数据。
* `[in] finalize_hint`: 传递给最终化回调的可选上下文提示。

如果 API 成功则返回 `napi_ok`。

在事件循环中异步调度一个 `napi_finalize` 回调。

通常，最终化器在 GC（垃圾收集器）收集对象时调用。此时，调用任何可能导致 GC 状态变化的 Node-API 将被禁用，并会使 Node.js 崩溃。

`node_api_post_finalizer` 通过允许插件将此类 Node-API 的调用推迟到 GC 最终化之外的时间点，有助于解决此限制。

## 简单的异步操作

插件模块通常需要利用 libuv 中的异步助手作为其实现的一部分。这允许它们安排工作异步执行，以便它们的方法可以在工作完成之前返回。这允许它们避免阻塞 Node.js 应用程序的整体执行。

Node-API 为这些支持函数提供了一个 ABI 稳定的接口，涵盖了最常见的异步用例。

Node-API 定义了用于管理异步工作线程的 `napi_async_work` 结构。实例使用 [`napi_create_async_work`][] 和 [`napi_delete_async_work`][] 创建/删除。

`execute` 和 `complete` 回调是在执行器准备好执行和完成任务时分别调用的函数。

`execute` 函数应避免进行任何可能导致 JavaScript 执行或与 JavaScript 对象交互的 Node-API 调用。大多数情况下，任何需要调用 Node-API 的代码应在 `complete` 回调中进行。避免在 execute 回调中使用 `napi_env` 参数，因为它可能会执行 JavaScript。

这些函数实现以下接口：

```c
typedef void (*napi_async_execute_callback)(napi_env env,
                                            void* data);
typedef void (*napi_async_complete_callback)(napi_env env,
                                             napi_status status,
                                             void* data);
```

当这些方法被调用时，传递的 `data` 参数将是传入 `napi_create_async_work` 调用的插件提供的 `void*` 数据。

一旦创建，异步工作线程可以使用 [`napi_queue_async_work`][] 函数排队执行：

```c
napi_status napi_queue_async_work(node_api_basic_env env,
                                  napi_async_work work);
```

如果工作在线程开始执行之前需要取消，可以使用 [`napi_cancel_async_work`][]。

调用 [`napi_cancel_async_work`][] 后，`complete` 回调将以状态值 `napi_cancelled` 被调用。即使工作被取消，也不应在 `complete` 回调调用之前删除工作。

### `napi_create_async_work`

<!-- YAML
added: v8.0.0
napiVersion: 1
changes:
  - version: v8.6.0
    pr-url: https://github.com/nodejs/node/pull/14697
    description: 添加了 `async_resource` 和 `async_resource_name` 参数。
-->

```c
napi_status napi_create_async_work(napi_env env,
                                   napi_value async_resource,
                                   napi_value async_resource_name,
                                   napi_async_execute_callback execute,
                                   napi_async_complete_callback complete,
                                   void* data,
                                   napi_async_work* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] async_resource`: 与异步工作关联的可选对象，将传递给可能的 `async_hooks` [`init` hooks][]。
* `[in] async_resource_name`: 用于标识资源类型的标识符，用于通过 `async_hooks` API 暴露的诊断信息。
* `[in] execute`: 应调用以异步执行逻辑的本地函数。给定的函数从工作池线程调用，并且可以与主事件循环线程并行执行。
* `[in] complete`: 当异步逻辑完成或被取消时将调用的本地函数。给定的函数从主事件循环线程调用。[`napi_async_complete_callback`][] 提供了更多细节。
* `[in] data`: 用户提供的数据上下文。这将传回 execute 和 complete 函数。
* `[out] result`: `napi_async_work*`，这是新创建的异步工作的句柄。

如果 API 成功则返回 `napi_ok`。

此 API 分配一个工作对象，用于异步执行逻辑。当不再需要工作时，应使用 [`napi_delete_async_work`][] 释放它。

`async_resource_name` 应该是一个以 null 结尾的 UTF-8 编码字符串。

`async_resource_name` 标识符由用户提供，应代表正在执行的异步工作的类型。还建议对标识符应用命名空间，例如包含模块名称。有关更多信息，请参阅 [`async_hooks` 文档][async_hooks `type`]。

### `napi_delete_async_work`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_delete_async_work(napi_env env,
                                   napi_async_work work);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] work`: 调用 `napi_create_async_work` 返回的句柄。

如果 API 成功则返回 `napi_ok`。

此 API 释放先前分配的工作对象。

即使存在待处理的 JavaScript 异常，也可以调用此 API。

### `napi_queue_async_work`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_queue_async_work(node_api_basic_env env,
                                  napi_async_work work);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] work`: 调用 `napi_create_async_work` 返回的句柄。

如果 API 成功则返回 `napi_ok`。

此 API 请求调度先前分配的工作以执行。一旦成功返回，不得再次使用相同的 `napi_async_work` 项调用此 API，否则结果将是未定义的。

### `napi_cancel_async_work`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_cancel_async_work(node_api_basic_env env,
                                   napi_async_work work);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] work`: 调用 `napi_create_async_work` 返回的句柄。

如果 API 成功则返回 `napi_ok`。

此 API 取消排队的工作（如果尚未开始执行）。如果已经开始执行，则无法取消，并将返回 `napi_generic_failure`。如果成功，`complete` 回调将以状态值 `napi_cancelled` 被调用。即使工作已成功取消，也不应在 `complete` 回调调用之前删除工作。

即使存在待处理的 JavaScript 异常，也可以调用此 API。

## 自定义异步操作

上述简单的异步工作 API 可能不适用于所有场景。当使用任何其他异步机制时，以下 API 是确保异步操作被运行时正确跟踪所必需的。

### `napi_async_init`

<!-- YAML
added: v8.6.0
napiVersion: 1
-->

```c
napi_status napi_async_init(napi_env env,
                            napi_value async_resource,
                            napi_value async_resource_name,
                            napi_async_context* result)
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] async_resource`: 与异步工作关联的对象，将传递给可能的 `async_hooks` [`init` hooks][]，并且可以通过 [`async_hooks.executionAsyncResource()`][] 访问。
* `[in] async_resource_name`: 用于标识资源类型的标识符，用于通过 `async_hooks` API 暴露的诊断信息。
* `[out] result`: 初始化的异步上下文。

如果 API 成功则返回 `napi_ok`。

`async_resource` 对象需要保持活动状态直到 [`napi_async_destroy`][]，以保持 `async_hooks` 相关 API 的正确操作。为了与先前版本保持 ABI 兼容性，`napi_async_context` 不维护对 `async_resource` 对象的强引用，以避免引入内存泄漏。但是，如果 `async_resource` 在 `napi_async_context` 被 `napi_async_destroy` 销毁之前被 JavaScript 引擎垃圾回收，则调用与 `napi_async_context` 相关的 API，如 [`napi_open_callback_scope`][] 和 [`napi_make_callback`][]，可能会导致问题，例如在使用 `AsyncLocalStorage` API 时丢失异步上下文。

为了与先前版本保持 ABI 兼容性，为 `async_resource` 传递 `NULL` 不会导致错误。但是，这不推荐，因为这将导致 `async_hooks` [`init` hooks][] 和 `async_hooks.executionAsyncResource()` 的不良行为，因为资源现在需要由底层的 `async_hooks` 实现以提供异步回调之间的链接。

### `napi_async_destroy`

<!-- YAML
added: v8.6.0
napiVersion: 1
-->

```c
napi_status napi_async_destroy(napi_env env,
                               napi_async_context async_context);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] async_context`: 要销毁的异步上下文。

如果 API 成功则返回 `napi_ok`。

即使存在待处理的 JavaScript 异常，也可以调用此 API。

### `napi_make_callback`

<!-- YAML
added: v8.0.0
napiVersion: 1
changes:
  - version: v8.6.0
    pr-url: https://github.com/nodejs/node/pull/15189
    description: 添加了 `async_context` 参数。
-->

```c
NAPI_EXTERN napi_status napi_make_callback(napi_env env,
                                           napi_async_context async_context,
                                           napi_value recv,
                                           napi_value func,
                                           size_t argc,
                                           const napi_value* argv,
                                           napi_value* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] async_context`: 调用回调的异步操作的上下文。这通常应该是先前从 [`napi_async_init`][] 获得的值。为了与先前版本保持 ABI 兼容性，为 `async_context` 传递 `NULL` 不会导致错误。然而，这会导致异步钩子的不正确操作。潜在问题包括在使用 `AsyncLocalStorage` API 时丢失异步上下文。
* `[in] recv`: 传递给被调用函数的 `this` 值。
* `[in] func`: 表示要调用的 JavaScript 函数的 `napi_value`。
* `[in] argc`: `argv` 数组中的元素数量。
* `[in] argv`: 表示函数参数的 JavaScript 值的 `napi_value` 数组。如果 `argc` 为零，则可以通过传入 `NULL` 来省略此参数。
* `[out] result`: 表示返回的 JavaScript 对象的 `napi_value`。

如果 API 成功则返回 `napi_ok`。

此方法允许从本地插件调用 JavaScript 函数对象。此 API 类似于 `napi_call_function`。但是，它用于在从异步操作返回后（当堆栈上没有其他脚本时）从本地代码回调到 JavaScript。它是 `node::MakeCallback` 的一个相当简单的包装器。

注意，在 `napi_async_complete_callback` 中不需要使用 `napi_make_callback`；在这种情况下，回调的异步上下文已经设置好，因此直接调用 `napi_call_function` 是足够且合适的。在实现不使用 `napi_create_async_work` 的自定义异步行为时，可能需要使用 `napi_make_callback` 函数。

在回调期间由 JavaScript 在微任务队列上调度的任何 `process.nextTick` 或 Promise 在返回到 C/C++ 之前运行。

### `napi_open_callback_scope`

<!-- YAML
added: v9.6.0
napiVersion: 3
-->

```c
NAPI_EXTERN napi_status napi_open_callback_scope(napi_env env,
                                                 napi_value resource_object,
                                                 napi_async_context context,
                                                 napi_callback_scope* result)
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] resource_object`: 与异步工作关联的对象，将传递给可能的 `async_hooks` [`init` hooks][]。此参数已被弃用，在运行时被忽略。请改用 [`napi_async_init`][] 中的 `async_resource` 参数。
* `[in] context`: 调用回调的异步操作的上下文。这应该是先前从 [`napi_async_init`][] 获得的值。
* `[out] result`: 新创建的作用域。

在某些情况下（例如，解析 Promise），在进行某些 Node-API 调用时，需要具有与回调关联的作用域。如果堆栈上没有其他脚本，可以使用 [`napi_open_callback_scope`][] 和 [`napi_close_callback_scope`][] 函数来打开/关闭所需的作用域。

### `napi_close_callback_scope`

<!-- YAML
added: v9.6.0
napiVersion: 3
-->

```c
NAPI_EXTERN napi_status napi_close_callback_scope(napi_env env,
                                                  napi_callback_scope scope)
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] scope`: 要关闭的作用域。

即使存在待处理的 JavaScript 异常，也可以调用此 API。

## 版本管理

### `napi_get_node_version`

<!-- YAML
added: v8.4.0
napiVersion: 1
-->

```c
typedef struct {
  uint32_t major;
  uint32_t minor;
  uint32_t patch;
  const char* release;
} napi_node_version;

napi_status napi_get_node_version(node_api_basic_env env,
                                  const napi_node_version** version);
```

* `[in] env`: 调用该 API 所在的环境。
* `[out] version`: 指向 Node.js 本身版本信息的指针。

如果 API 成功则返回 `napi_ok`。

此函数使用当前运行的 Node.js 的主版本、次版本和补丁版本填充 `version` 结构，并使用 [`process.release.name`][`process.release`] 的值填充 `release` 字段。

返回的缓冲区是静态分配的，不需要释放。

### `napi_get_version`

<!-- YAML
added: v8.0.0
napiVersion: 1
-->

```c
napi_status napi_get_version(node_api_basic_env env,
                             uint32_t* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[out] result`: 支持的 Node-API 最高版本。

如果 API 成功则返回 `napi_ok`。

此 API 返回 Node.js 运行时支持的 Node-API 最高版本。Node-API 计划是增量的，因此新版本的 Node.js 可能支持额外的 API 函数。为了允许插件在运行支持它的 Node.js 版本时使用新函数，同时在运行不支持它的 Node.js 版本时提供回退行为：

* 调用 `napi_get_version()` 以确定 API 是否可用。
* 如果可用，使用 `uv_dlsym()` 动态加载指向函数的指针。
* 使用动态加载的指针调用函数。
* 如果函数不可用，提供不使用该函数的备用实现。

## 内存管理

### `napi_adjust_external_memory`

<!-- YAML
added: v8.5.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_adjust_external_memory(node_api_basic_env env,
                                                    int64_t change_in_bytes,
                                                    int64_t* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] change_in_bytes`: 由 JavaScript 对象保持活动状态的外部分配内存的变化。
* `[out] result`: 调整后的值。此值应反映包含给定 `change_in_bytes` 后的外部内存总量。不应依赖返回值的绝对值。例如，实现可能对所有插件使用单个计数器，或对每个插件使用一个计数器。

如果 API 成功则返回 `napi_ok`。

此函数向运行时指示由 JavaScript 对象（即指向由本地插件分配的自身内存的 JavaScript 对象）保持活动状态的外部分配内存量。注册外部分配内存可能会（但不保证）比原本更频繁地触发全局垃圾回收。

期望以插件不减少外部内存超过其增加的外部内存的方式调用此函数。

## Promise

Node-API 提供了用于创建 `Promise` 对象的设施，如 ECMA 规范中 [Section Promise objects][] 所述。它将 Promise 实现为一对对象。当通过 `napi_create_promise()` 创建 Promise 时，会创建一个 "deferred" 对象并与 `Promise` 一起返回。延迟对象绑定到创建的 `Promise`，并且是使用 `napi_resolve_deferred()` 或 `napi_reject_deferred()` 解决或拒绝 `Promise` 的唯一方法。由 `napi_create_promise()` 创建的延迟对象通过 `napi_resolve_deferred()` 或 `napi_reject_deferred()` 释放。`Promise` 对象可以返回到 JavaScript，在那里可以以通常的方式使用。

例如，创建一个 Promise 并将其传递给异步工作线程：

```c
napi_deferred deferred;
napi_value promise;
napi_status status;

// 创建 Promise。
status = napi_create_promise(env, &deferred, &promise);
if (status != napi_ok) return NULL;

// 将 deferred 传递给执行异步操作的函数。
do_something_asynchronous(deferred);

// 将 promise 返回给 JS
return promise;
```

上面的函数 `do_something_asynchronous()` 将执行其异步操作，然后它将解决或拒绝 deferred，从而完成 Promise 并释放 deferred：

```c
napi_deferred deferred;
napi_value undefined;
napi_status status;

// 创建一个值用于完成 deferred。
status = napi_get_undefined(env, &undefined);
if (status != napi_ok) return NULL;

// 根据异步操作是否成功，解决或拒绝与 deferred 关联的 Promise。
if (asynchronous_action_succeeded) {
  status = napi_resolve_deferred(env, deferred, undefined);
} else {
  status = napi_reject_deferred(env, deferred, undefined);
}
if (status != napi_ok) return NULL;

// 此时 deferred 已被释放，因此我们应将其赋值为 NULL。
deferred = NULL;
```

### `napi_create_promise`

<!-- YAML
added: v8.5.0
napiVersion: 1
-->

```c
napi_status napi_create_promise(napi_env env,
                                napi_deferred* deferred,
                                napi_value* promise);
```

* `[in] env`: 调用该 API 所在的环境。
* `[out] deferred`: 新创建的延迟对象，之后可以传递给 `napi_resolve_deferred()` 或 `napi_reject_deferred()` 以分别解决或拒绝关联的 Promise。
* `[out] promise`: 与延迟对象关联的 JavaScript Promise。

如果 API 成功则返回 `napi_ok`。

此 API 创建一个延迟对象和一个 JavaScript Promise。

### `napi_resolve_deferred`

<!-- YAML
added: v8.5.0
napiVersion: 1
-->

```c
napi_status napi_resolve_deferred(napi_env env,
                                  napi_deferred deferred,
                                  napi_value resolution);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] deferred`: 要解决其关联 Promise 的延迟对象。
* `[in] resolution`: 用于解决 Promise 的值。

此 API 通过与其关联的延迟对象解决 JavaScript Promise。因此，它只能用于解决对应的延迟对象可用的 JavaScript Promise。这实际上意味着 Promise 必须使用 `napi_create_promise()` 创建，并且从该调用返回的延迟对象必须被保留以便传递给此 API。

延迟对象在成功完成时被释放。

### `napi_reject_deferred`

<!-- YAML
added: v8.5.0
napiVersion: 1
-->

```c
napi_status napi_reject_deferred(napi_env env,
                                 napi_deferred deferred,
                                 napi_value rejection);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] deferred`: 要解决其关联 Promise 的延迟对象。
* `[in] rejection`: 用于拒绝 Promise 的值。

此 API 通过与其关联的延迟对象拒绝 JavaScript Promise。因此，它只能用于拒绝对应的延迟对象可用的 JavaScript Promise。这实际上意味着 Promise 必须使用 `napi_create_promise()` 创建，并且从该调用返回的延迟对象必须被保留以便传递给此 API。

延迟对象在成功完成时被释放。

### `napi_is_promise`

<!-- YAML
added: v8.5.0
napiVersion: 1
-->

```c
napi_status napi_is_promise(napi_env env,
                            napi_value value,
                            bool* is_promise);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] value`: 要检查的值
* `[out] is_promise`: 指示 `promise` 是否是本地 Promise 对象（即由底层引擎创建的 Promise 对象）的标志。

## 脚本执行

Node-API 提供了一个 API，用于使用底层 JavaScript 引擎执行包含 JavaScript 的字符串。

### `napi_run_script`

<!-- YAML
added: v8.5.0
napiVersion: 1
-->

```c
NAPI_EXTERN napi_status napi_run_script(napi_env env,
                                        napi_value script,
                                        napi_value* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] script`: 包含要执行的脚本的 JavaScript 字符串。
* `[out] result`: 执行脚本产生的值。

此函数执行一段 JavaScript 代码并返回其结果，但有以下注意事项：

* 与 `eval` 不同，此函数不允许脚本访问当前词法作用域，因此也不允许访问 [模块作用域][]，这意味着伪全局变量如 `require` 将不可用。
* 脚本可以访问 [全局作用域][]。脚本中的函数和 `var` 声明将被添加到 [`global`][] 对象。使用 `let` 和 `const` 进行的变量声明将在全局可见，但不会添加到 [`global`][] 对象。
* 脚本中的 `this` 是 [`global`][]。

## libuv 事件循环

Node-API 提供了一个函数，用于获取与特定 `napi_env` 关联的当前事件循环。

### `napi_get_uv_event_loop`

<!-- YAML
added:
  - v9.3.0
  - v8.10.0
napiVersion: 2
-->

```c
NAPI_EXTERN napi_status napi_get_uv_event_loop(node_api_basic_env env,
                                               struct uv_loop_s** loop);
```

* `[in] env`: 调用该 API 所在的环境。
* `[out] loop`: 当前的 libuv 循环实例。

注意：虽然 libuv 随着时间的推移相对稳定，但它不提供 ABI 稳定性保证。应避免使用此函数。它的使用可能导致插件无法跨 Node.js 版本工作。[asynchronous-thread-safe-function-calls](https://nodejs.org/docs/latest/api/n-api.html#asynchronous-thread-safe-function-calls) 是许多用例的替代方案。

## 异步线程安全函数调用

JavaScript 函数通常只能从本地插件的主线程调用。如果插件创建了额外的线程，则不得从这些线程调用需要 `napi_env`、`napi_value` 或 `napi_ref` 的 Node-API 函数。

当插件有额外的线程并且需要基于这些线程完成的处理调用 JavaScript 函数时，这些线程必须与插件的主线程通信，以便主线程可以代表它们调用 JavaScript 函数。线程安全函数 API 提供了一种简单的方法来实现这一点。

这些 API 提供了 `napi_threadsafe_function` 类型以及创建、销毁和调用此类型对象的 API。
`napi_create_threadsafe_function()` 创建一个对持有 JavaScript 函数的 `napi_value` 的持久引用，该函数可以从多个线程调用。调用是异步进行的。这意味着将用于调用 JavaScript 回调的值将被放入队列中，并且对于队列中的每个值，最终都会调用 JavaScript 函数。

在创建 `napi_threadsafe_function` 时，可以提供一个 `napi_finalize` 回调。当线程安全函数即将被销毁时，将在主线程上调用此回调。它接收在构造期间给出的上下文和最终化数据，并提供了在线程之后进行清理的机会，例如通过调用 `uv_thread_join()`。**除了主循环线程之外，在最终化回调完成后，不应有任何线程使用线程安全函数。**

在调用 `napi_create_threadsafe_function()` 期间给出的 `context` 可以通过调用 `napi_get_threadsafe_function_context()` 从任何线程检索。

### 调用线程安全函数

`napi_call_threadsafe_function()` 可用于发起对 JavaScript 的调用。`napi_call_threadsafe_function()` 接受一个参数，该参数控制 API 是否阻塞行为。如果设置为 `napi_tsfn_nonblocking`，则 API 表现为非阻塞，如果队列已满则返回 `napi_queue_full`，防止数据成功添加到队列。如果设置为 `napi_tsfn_blocking`，则 API 会阻塞直到队列中有空间可用。如果线程安全函数创建时最大队列大小为 0，则 `napi_call_threadsafe_function()` 永远不会阻塞。

不应从 JavaScript 线程使用 `napi_tsfn_blocking` 调用 `napi_call_threadsafe_function()`，因为如果队列已满，它可能导致 JavaScript 线程死锁。

实际调用 JavaScript 由通过 `call_js_cb` 参数给出的回调控制。每次通过成功调用 `napi_call_threadsafe_function()` 将值放入队列时，`call_js_cb` 会在主线程上调用一次。如果未给出此类回调，则将使用默认回调，并且生成的 JavaScript 调用将没有参数。`call_js_cb` 回调在其参数中接收要调用的 JavaScript 函数作为 `napi_value`，以及在创建 `napi_threadsafe_function` 时使用的 `void*` 上下文指针，以及由辅助线程之一创建的下一个数据指针。然后回调可以使用诸如 `napi_call_function()` 之类的 API 来调用 JavaScript。

回调也可能在 `env` 和 `call_js_cb` 都设置为 `NULL` 的情况下被调用，以指示不再可能调用 JavaScript，而队列中可能仍有需要释放的项。这通常发生在 Node.js 进程退出时仍有线程安全函数处于活动状态时。

不需要通过 `napi_make_callback()` 调用 JavaScript，因为 Node-API 在适合回调的上下文中运行 `call_js_cb`。

在事件循环的每个 tick 中可能会调用零个或多个排队项。应用程序不应依赖特定行为，除了在调用回调方面会取得进展，并且随着时间推移事件将被调用。

### 线程安全函数的引用计数

在线程安全函数的存在期间，可以向其添加和移除线程。因此，除了在创建时指定初始线程数之外，可以调用 `napi_acquire_threadsafe_function` 来指示新线程将开始使用线程安全函数。类似地，可以调用 `napi_release_threadsafe_function` 来指示现有线程将停止使用线程安全函数。

当每个使用该对象的线程都调用了 `napi_release_threadsafe_function()` 或在响应 `napi_call_threadsafe_function` 的调用时收到了返回状态 `napi_closing` 时，`napi_threadsafe_function` 对象将被销毁。在 `napi_threadsafe_function` 被销毁之前，队列会被清空。`napi_release_threadsafe_function()` 应该是与给定 `napi_threadsafe_function` 关联的最后一个 API 调用，因为在调用完成后，无法保证 `napi_threadsafe_function` 仍然分配。出于同样的原因，在收到 `napi_call_threadsafe_function` 的调用返回值为 `napi_closing` 后，不要使用线程安全函数。与 `napi_threadsafe_function` 关联的数据可以在其 `napi_finalize` 回调中释放，该回调被传递给 `napi_create_threadsafe_function()`。`napi_create_threadsafe_function` 的参数 `initial_thread_count` 标记线程安全函数的初始获取次数，而不是在创建时多次调用 `napi_acquire_threadsafe_function`。

一旦使用 `napi_threadsafe_function` 的线程数达到零，后续线程无法通过调用 `napi_acquire_threadsafe_function()` 开始使用它。事实上，所有后续与其关联的 API 调用，除了 `napi_release_threadsafe_function()`，都将返回错误值 `napi_closing`。

可以通过向 `napi_release_threadsafe_function()` 提供 `napi_tsfn_abort` 值来"中止"线程安全函数。这将导致所有与线程安全函数关联的后续 API（除了 `napi_release_threadsafe_function()`）返回 `napi_closing`，即使其引用计数尚未达到零。特别是，`napi_call_threadsafe_function()` 将返回 `napi_closing`，从而通知线程不再可能对线程安全函数进行异步调用。这可以用作终止线程的标准。**一旦从 `napi_call_threadsafe_function()` 收到返回值 `napi_closing`，线程不得再使用线程安全函数，因为它不再保证被分配。**

### 决定是否保持进程运行

与 libuv 句柄类似，线程安全函数可以被"引用"和"取消引用"。一个"被引用"的线程安全函数将导致创建它的事件循环线程保持活动状态，直到线程安全函数被销毁。相反，一个"未被引用"的线程安全函数不会阻止事件循环退出。为此存在 API `napi_ref_threadsafe_function` 和 `napi_unref_threadsafe_function`。

`napi_unref_threadsafe_function` 不会将线程安全函数标记为能够被销毁，`napi_ref_threadsafe_function` 也不会阻止它被销毁。

### `napi_create_threadsafe_function`

<!-- YAML
added: v10.6.0
napiVersion: 4
changes:
  - version:
     - v12.6.0
     - v10.17.0
    pr-url: https://github.com/nodejs/node/pull/27791
    description: 使 `func` 参数对于自定义 `call_js_cb` 是可选的。
-->

```c
NAPI_EXTERN napi_status
napi_create_threadsafe_function(napi_env env,
                                napi_value func,
                                napi_value async_resource,
                                napi_value async_resource_name,
                                size_t max_queue_size,
                                size_t initial_thread_count,
                                void* thread_finalize_data,
                                napi_finalize thread_finalize_cb,
                                void* context,
                                napi_threadsafe_function_call_js call_js_cb,
                                napi_threadsafe_function* result);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] func`: 从另一个线程调用的可选 JavaScript 函数。如果向 `call_js_cb` 传递了 `NULL`，则必须提供它。
* `[in] async_resource`: 与异步工作关联的可选对象，将传递给可能的 `async_hooks` [`init` hooks][]。
* `[in] async_resource_name`: 一个 JavaScript 字符串，用于为通过 `async_hooks` API 暴露的诊断信息提供资源类型的标识符。
* `[in] max_queue_size`: 队列的最大大小。`0` 表示无限制。
* `[in] initial_thread_count`: 初始获取次数，即初始线程数，包括主线程，这些线程将使用此函数。
* `[in] thread_finalize_data`: 要传递给 `thread_finalize_cb` 的可选数据。
* `[in] thread_finalize_cb`: 当 `napi_threadsafe_function` 被销毁时要调用的可选函数。
* `[in] context`: 要附加到结果 `napi_threadsafe_function` 的可选数据。
* `[in] call_js_cb`: 可选的回调，用于响应不同线程上的调用而调用 JavaScript 函数。此回调将在主线程上调用。如果未给出，JavaScript 函数将被调用，没有参数，且 `this` 值为 `undefined`。[`napi_threadsafe_function_call_js`][] 提供了更多细节。
* `[out] result`: 异步线程安全的 JavaScript 函数。

**变更历史：**

* 版本 10（`NAPI_VERSION` 定义为 `10` 或更高）：

  在 `call_js_cb` 中抛出的未捕获异常使用 [`'uncaughtException'`][] 事件处理，而不是被忽略。

### `napi_get_threadsafe_function_context`

<!-- YAML
added: v10.6.0
napiVersion: 4
-->

```c
NAPI_EXTERN napi_status
napi_get_threadsafe_function_context(napi_threadsafe_function func,
                                     void** result);
```

* `[in] func`: 要检索其上下文的线程安全函数。
* `[out] result`: 存储上下文的位置。

此 API 可以从任何使用 `func` 的线程调用。

### `napi_call_threadsafe_function`

<!-- YAML
added: v10.6.0
napiVersion: 4
changes:
  - version: v14.5.0
    pr-url: https://github.com/nodejs/node/pull/33453
    description: 对 `napi_would_deadlock` 的支持已被撤销。
  - version: v14.1.0
    pr-url: https://github.com/nodejs/node/pull/32689
    description: 当从主线程或工作线程使用 `napi_tsfn_blocking` 调用且队列已满时返回 `napi_would_deadlock`。
-->

```c
NAPI_EXTERN napi_status
napi_call_threadsafe_function(napi_threadsafe_function func,
                              void* data,
                              napi_threadsafe_function_call_mode is_blocking);
```

* `[in] func`: 要调用的异步线程安全 JavaScript 函数。
* `[in] data`: 通过在线程安全 JavaScript 函数创建期间提供的回调 `call_js_cb` 发送到 JavaScript 的数据。
* `[in] is_blocking`: 标志，其值可以是 `napi_tsfn_blocking` 以指示如果队列已满则调用应阻塞，或 `napi_tsfn_nonblocking` 以指示每当队列已满时调用应立即返回状态 `napi_queue_full`。

不应从 JavaScript 线程使用 `napi_tsfn_blocking` 调用此 API，因为如果队列已满，它可能导致 JavaScript 线程死锁。

如果从任何线程使用 `abort` 设置为 `napi_tsfn_abort` 调用了 `napi_release_threadsafe_function()`，则此 API 将返回 `napi_closing`。仅当 API 返回 `napi_ok` 时，值才会被添加到队列中。

此 API 可以从任何使用 `func` 的线程调用。

### `napi_acquire_threadsafe_function`

<!-- YAML
added: v10.6.0
napiVersion: 4
-->

```c
NAPI_EXTERN napi_status
napi_acquire_threadsafe_function(napi_threadsafe_function func);
```

* `[in] func`: 要开始使用的异步线程安全 JavaScript 函数。

线程在将 `func` 传递给任何其他线程安全函数 API 之前应调用此 API，以指示它将开始使用 `func`。这可以防止 `func` 在所有其他线程停止使用它时被销毁。

此 API 可以从任何将开始使用 `func` 的线程调用。

### `napi_release_threadsafe_function`

<!-- YAML
added: v10.6.0
napiVersion: 4
-->

```c
NAPI_EXTERN napi_status
napi_release_threadsafe_function(napi_threadsafe_function func,
                                 napi_threadsafe_function_release_mode mode);
```

* `[in] func`: 要递减其引用计数的异步线程安全 JavaScript 函数。
* `[in] mode`: 标志，其值可以是 `napi_tsfn_release` 以指示当前线程将不再对线程安全函数进行进一步调用，或 `napi_tsfn_abort` 以指示除了当前线程之外，没有其他线程应对线程安全函数进行任何进一步调用。如果设置为 `napi_tsfn_abort`，则对 `napi_call_threadsafe_function()` 的进一步调用将返回 `napi_closing`，并且不会有进一步的值被放入队列。

当线程停止使用 `func` 时应调用此 API。在调用此 API 后将 `func` 传递给任何线程安全 API 具有未定义的结果，因为 `func` 可能已被销毁。

此 API 可以从任何将停止使用 `func` 的线程调用。

### `napi_ref_threadsafe_function`

<!-- YAML
added: v10.6.0
napiVersion: 4
-->

```c
NAPI_EXTERN napi_status
napi_ref_threadsafe_function(node_api_basic_env env, napi_threadsafe_function func);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] func`: 要引用的线程安全函数。

此 API 用于指示主线程上运行的事件循环在 `func` 被销毁之前不应退出。类似于 [`uv_ref`][]，它也是幂等的。

`napi_unref_threadsafe_function` 不会将线程安全函数标记为能够被销毁，`napi_ref_threadsafe_function` 也不会阻止它被销毁。`napi_acquire_threadsafe_function` 和 `napi_release_threadsafe_function` 可用于此目的。

此 API 只能从主线程调用。

### `napi_unref_threadsafe_function`

<!-- YAML
added: v10.6.0
napiVersion: 4
-->

```c
NAPI_EXTERN napi_status
napi_unref_threadsafe_function(node_api_basic_env env, napi_threadsafe_function func);
```

* `[in] env`: 调用该 API 所在的环境。
* `[in] func`: 要取消引用的线程安全函数。

此 API 用于指示主线程上运行的事件循环可以在 `func` 被销毁之前退出。类似于 [`uv_unref`][]，它也是幂等的。

此 API 只能从主线程调用。

## 杂项实用程序

### `node_api_get_module_file_name`

<!-- YAML
added:
  - v15.9.0
  - v14.18.0
  - v12.22.0
napiVersion: 9
-->

```c
NAPI_EXTERN napi_status
node_api_get_module_file_name(node_api_basic_env env, const char** result);

```

* `[in] env`: 调用该 API 所在的环境。
* `[out] result`: 包含加载插件的绝对位置的 URL。对于本地文件系统上的文件，它将以 `file://` 开头。字符串以 null 结尾，由 `env` 拥有，因此不得修改或释放。

如果插件加载过程在加载期间无法建立插件的文件名，则 `result` 可能是一个空字符串。

[ABI Stability]: https://nodejs.org/en/docs/guides/abi-stability/
[AppVeyor]: https://www.appveyor.com
[C++ Addons]: addons.md
[CMake]: https://cmake.org
[CMake.js]: https://github.com/cmake-js/cmake-js
[ECMAScript Language Specification]: https://tc39.es/ecma262/
[Error handling]: #error-handling
[GCC]: https://gcc.gnu.org
[GYP]: https://gyp.gsrc.io
[GitHub releases]: https://help.github.com/en/github/administering-a-repository/about-releases
[LLVM]: https://llvm.org
[Native Abstractions for Node.js]: https://github.com/nodejs/nan
[Node-API Media]: https://github.com/nodejs/abi-stable-node/blob/HEAD/node-api-media.md
[Object lifetime management]: #object-lifetime-management
[Object wrap]: #object-wrap
[Section Agents]: https://tc39.es/ecma262/#sec-agents
[Section Array instance length]: https://tc39.es/ecma262/#sec-properties-of-array-instances-length
[Section Array objects]: https://tc39.es/ecma262/#sec-array-objects
[Section ArrayBuffer objects]: https://tc39.es/ecma262/#sec-arraybuffer-objects
[Section DataView objects]: https://tc39.es/ecma262/#sec-dataview-objects
[Section Date objects]: https://tc39.es/ecma262/#sec-date-objects
[Section DefineOwnProperty]: https://tc39.es/ecma262/#sec-ordinary-object-internal-methods-and-internal-slots-defineownproperty-p-desc
[Section Function objects]: https://tc39.es/ecma262/#sec-function-objects
[Section IsArray]: https://tc39.es/ecma262/#sec-isarray
[Section IsStrctEqual]: https://tc39.es/ecma262/#sec-strict-equality-comparison
[Section Promise objects]: https://tc39.es/ecma262/#sec-promise-objects
[Section SharedArrayBuffer objects]: https://tc39.es/ecma262/#sec-sharedarraybuffer-objects
[Section ToBoolean]: https://tc39.es/ecma262/#sec-toboolean
[Section ToNumber]: https://tc39.es/ecma262/#sec-tonumber
[Section ToObject]: https://tc39.es/ecma262/#sec-toobject
[Section ToString]: https://tc39.es/ecma262/#sec-tostring
[Section TypedArray objects]: https://tc39.es/ecma262/#sec-typedarray-objects
[Section detachArrayBuffer]: https://tc39.es/ecma262/#sec-detacharraybuffer
[Section instanceof operator]: https://tc39.es/ecma262/#sec-instanceofoperator
[Section isDetachedBuffer]: https://tc39.es/ecma262/#sec-isdetachedbuffer
[Section language types]: https://tc39.es/ecma262/#sec-ecmascript-data-types-and-values
[Section number type]: https://tc39.es/ecma262/#sec-ecmascript-language-types-number-type
[Section object type]: https://tc39.es/ecma262/#sec-object-type
[Section property attributes]: https://tc39.es/ecma262/#sec-property-attributes
[Section string type]: https://tc39.es/ecma262/#sec-ecmascript-language-types-string-type
[Section symbol type]: https://tc39.es/ecma262/#sec-ecmascript-language-types-symbol-type
[Section typeof operator]: https://tc39.es/ecma262/#sec-typeof-operator
[Travis CI]: https://travis-ci.org
[Visual Studio]: https://visualstudio.microsoft.com
[Working with JavaScript properties]: #working-with-javascript-properties
[Xcode]: https://developer.apple.com/xcode/
[`'uncaughtException'`]: process.md#event-uncaughtexception
[`Number.MAX_SAFE_INTEGER`]: https://tc39.es/ecma262/#sec-number.max_safe_integer
[`Number.MIN_SAFE_INTEGER`]: https://tc39.es/ecma262/#sec-number.min_safe_integer
[`Worker`]: worker_threads.md#class-worker
[`async_hooks.executionAsyncResource()`]: async_hooks.md#async_hooksexecutionasyncresource
[`build_with_cmake`]: https://github.com/nodejs/node-addon-examples/tree/main/src/8-tooling/build_with_cmake
[`global`]: globals.md#global
[`init` hooks]: async_hooks.md#initasyncid-type-triggerasyncid-resource
[`napi_add_async_cleanup_hook`]: #napi_add_async_cleanup_hook
[`napi_add_env_cleanup_hook`]: #napi_add_env_cleanup_hook
[`napi_add_finalizer`]: #napi_add_finalizer
[`napi_async_cleanup_hook`]: #napi_async_cleanup_hook
[`napi_async_complete_callback`]: #napi_async_complete_callback
[`napi_async_destroy`]: #napi_async_destroy
[`napi_async_init`]: #napi_async_init
[`napi_callback`]: #napi_callback
[`napi_cancel_async_work`]: #napi_cancel_async_work
[`napi_close_callback_scope`]: #napi_close_callback_scope
[`napi_close_escapable_handle_scope`]: #napi_close_escapable_handle_scope
[`napi_close_handle_scope`]: #napi_close_handle_scope
[`napi_create_async_work`]: #napi_create_async_work
[`napi_create_error`]: #napi_create_error
[`napi_create_external_arraybuffer`]: #napi_create_external_arraybuffer
[`napi_create_range_error`]: #napi_create_range_error
[`napi_create_reference`]: #napi_create_reference
[`napi_create_type_error`]: #napi_create_type_error
[`napi_define_class`]: #napi_define_class
[`napi_delete_async_work`]: #napi_delete_async_work
[`napi_delete_reference`]: #napi_delete_reference
[`napi_escape_handle`]: #napi_escape_handle
[`napi_finalize`]: #napi_finalize
[`napi_get_and_clear_last_exception`]: #napi_get_and_clear_last_exception
[`napi_get_array_length`]: #napi_get_array_length
[`napi_get_element`]: #napi_get_element
[`napi_get_last_error_info`]: #napi_get_last_error_info
[`napi_get_property`]: #napi_get_property
[`napi_get_reference_value`]: #napi_get_reference_value
[`napi_get_typedarray_info`]: #napi_get_typedarray_info
[`napi_get_value_external`]: #napi_get_value_external
[`napi_has_property`]: #napi_has_property
[`napi_instanceof`]: #napi_instanceof
[`napi_is_error`]: #napi_is_error
[`napi_is_exception_pending`]: #napi_is_exception_pending
[`napi_is_typedarray`]: #napi_is_typedarray
[`napi_make_callback`]: #napi_make_callback
[`napi_open_callback_scope`]: #napi_open_callback_scope
[`napi_open_escapable_handle_scope`]: #napi_open_escapable_handle_scope
[`napi_open_handle_scope`]: #napi_open_handle_scope
[`napi_property_attributes`]: #napi_property_attributes
[`napi_property_descriptor`]: #napi_property_descriptor
[`napi_queue_async_work`]: #napi_queue_async_work
[`napi_reference_ref`]: #napi_reference_ref
[`napi_reference_unref`]: #napi_reference_unref
[`napi_remove_async_cleanup_hook`]: #napi_remove_async_cleanup_hook
[`napi_remove_env_cleanup_hook`]: #napi_remove_env_cleanup_hook
[`napi_set_instance_data`]: #napi_set_instance_data
[`napi_set_property`]: #napi_set_property
[`napi_threadsafe_function_call_js`]: #napi_threadsafe_function_call_js
[`napi_throw_error`]: #napi_throw_error
[`napi_throw_range_error`]: #napi_throw_range_error
[`napi_throw_type_error`]: #napi_throw_type_error
[`napi_throw`]: #napi_throw
[`napi_unwrap`]: #napi_unwrap
[`napi_wrap`]: #napi_wrap
[`node-addon-api`]: https://github.com/nodejs/node-addon-api
[`node_api.h`]: https://github.com/nodejs/node/blob/HEAD/src/node_api.h
[`node_api_basic_finalize`]: #node_api_basic_finalize
[`node_api_create_external_string_latin1`]: #node_api_create_external_string_latin1
[`node_api_create_external_string_utf16`]: #node_api_create_external_string_utf16
[`node_api_create_syntax_error`]: #node_api_create_syntax_error
[`node_api_post_finalizer`]: #node_api_post_finalizer
[`node_api_throw_syntax_error`]: #node_api_throw_syntax_error
[`process.release`]: process.md#processrelease
[`uv_ref`]: https://docs.libuv.org/en/v1.x/handle.html#c.uv_ref
[`uv_unref`]: https://docs.libuv.org/en/v1.x/handle.html#c.uv_unref
[`worker.terminate()`]: worker_threads.md#workerterminate
[async_hooks `type`]: async_hooks.md#type
[context-aware addons]: addons.md#context-aware-addons
[docs]: https://github.com/nodejs/node-addon-api#api-documentation
[external]: #napi_create_external
[externals]: #napi_create_external
[global scope]: globals.md
[gyp-next]: https://github.com/nodejs/gyp-next
[language and engine bindings]: https://github.com/nodejs/abi-stable-node/blob/doc/node-api-engine-bindings.md
[module scope]: modules.md#the-module-scope
[node-gyp]: https://github.com/nodejs/node-gyp
[node-pre-gyp]: https://github.com/mapbox/node-pre-gyp
[prebuild]: https://github.com/prebuild/prebuild
[prebuildify]: https://github.com/prebuild/prebuildify
[worker threads]: https://nodejs.org/api/worker_threads.html
