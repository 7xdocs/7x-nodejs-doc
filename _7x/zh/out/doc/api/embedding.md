# C++ 嵌入器 API

<!--introduced_in=v12.19.0-->

Node.js 提供了一系列 C++ API，其他 C++ 软件可以使用这些 API 在 Node.js 环境中执行 JavaScript。

这些 API 的文档可以在 Node.js 源码树的 [src/node.h][] 中找到。除了 Node.js 暴露的 API 之外，一些必需的概念由 V8 嵌入器 API 提供。

由于将 Node.js 用作嵌入式库与编写由 Node.js 执行的代码不同，破坏性变更不遵循典型的 Node.js [弃用策略][deprecation policy]，并且可能在每个 semver-major 版本中发生，而不会事先警告。

## 嵌入式应用示例

以下部分将概述如何使用这些 API 从头创建一个应用程序，该应用程序将执行等同于 `node -e <code>` 的操作，即接收一段 JavaScript 代码并在 Node.js 特定环境中运行它。

完整代码可以在 [Node.js 源码树][embedtest.cc] 中找到。

### 设置每个进程的状态

Node.js 需要一些每个进程的状态管理才能运行：

* 解析 Node.js [CLI 选项][CLI options] 的参数，
* V8 每个进程的要求，例如一个 `v8::Platform` 实例。

以下示例展示了如何设置这些内容。一些类名分别来自 `node` 和 `v8` C++ 命名空间。

```cpp
int main(int argc, char** argv) {
  argv = uv_setup_args(argc, argv);
  std::vector<std::string> args(argv, argv + argc);
  // Parse Node.js CLI options, and print any errors that have occurred while
  // trying to parse them.
  std::unique_ptr<node::InitializationResult> result =
      node::InitializeOncePerProcess(args, {
        node::ProcessInitializationFlags::kNoInitializeV8,
        node::ProcessInitializationFlags::kNoInitializeNodeV8Platform
      });

  for (const std::string& error : result->errors())
    fprintf(stderr, "%s: %s\n", args[0].c_str(), error.c_str());
  if (result->early_return() != 0) {
    return result->exit_code();
  }

  // Create a v8::Platform instance. `MultiIsolatePlatform::Create()` is a way
  // to create a v8::Platform instance that Node.js can use when creating
  // Worker threads. When no `MultiIsolatePlatform` instance is present,
  // Worker threads are disabled.
  std::unique_ptr<MultiIsolatePlatform> platform =
      MultiIsolatePlatform::Create(4);
  V8::InitializePlatform(platform.get());
  V8::Initialize();

  // See below for the contents of this function.
  int ret = RunNodeInstance(
      platform.get(), result->args(), result->exec_args());

  V8::Dispose();
  V8::DisposePlatform();

  node::TearDownOncePerProcess();
  return ret;
}
```

### 设置每个实例的状态

<!-- YAML
changes:
  - version: v15.0.0
    pr-url: https://github.com/nodejs/node/pull/35597
    description:
      The `CommonEnvironmentSetup` and `SpinEventLoop` utilities were added.
-->

Node.js 有一个 "Node.js 实例" 的概念，通常被称为 `node::Environment`。每个 `node::Environment` 都与以下内容关联：

* 恰好一个 `v8::Isolate`，即一个 JS 引擎实例，
* 恰好一个 `uv_loop_t`，即一个事件循环，
* 多个 `v8::Context`，但恰好一个主 `v8::Context`，以及
* 一个 `node::IsolateData` 实例，其中包含可以被多个 `node::Environment` 共享的信息。嵌入器应确保 `node::IsolateData` 仅在共享相同 `v8::Isolate` 的 `node::Environment` 之间共享，Node.js 不会执行此检查。

为了设置一个 `v8::Isolate`，需要提供一个 `v8::ArrayBuffer::Allocator`。一个可能的选择是默认的 Node.js 分配器，可以通过 `node::ArrayBufferAllocator::Create()` 创建。当插件使用 Node.js C++ `Buffer` API 时，使用 Node.js 分配器允许微小的性能优化，并且是为了在 [`process.memoryUsage()`][] 中跟踪 `ArrayBuffer` 内存所必需的。

此外，每个用于 Node.js 实例的 `v8::Isolate` 都需要在使用 `MultiIsolatePlatform` 实例（如果正在使用的话）时进行注册和注销，以便平台知道该使用哪个事件循环来执行由该 `v8::Isolate` 调度的任务。

`node::NewIsolate()` 辅助函数会创建一个 `v8::Isolate`，使用一些 Node.js 特定的钩子（例如 Node.js 错误处理程序）对其进行设置，并自动将其注册到平台。

```cpp
int RunNodeInstance(MultiIsolatePlatform* platform,
                    const std::vector<std::string>& args,
                    const std::vector<std::string>& exec_args) {
  int exit_code = 0;

  // Setup up a libuv event loop, v8::Isolate, and Node.js Environment.
  std::vector<std::string> errors;
  std::unique_ptr<CommonEnvironmentSetup> setup =
      CommonEnvironmentSetup::Create(platform, &errors, args, exec_args);
  if (!setup) {
    for (const std::string& err : errors)
      fprintf(stderr, "%s: %s\n", args[0].c_str(), err.c_str());
    return 1;
  }

  Isolate* isolate = setup->isolate();
  Environment* env = setup->env();

  {
    Locker locker(isolate);
    Isolate::Scope isolate_scope(isolate);
    HandleScope handle_scope(isolate);
    // The v8::Context needs to be entered when node::CreateEnvironment() and
    // node::LoadEnvironment() are being called.
    Context::Scope context_scope(setup->context());

    // Set up the Node.js instance for execution, and run code inside of it.
    // There is also a variant that takes a callback and provides it with
    // the `require` and `process` objects, so that it can manually compile
    // and run scripts as needed.
    // The `require` function inside this script does *not* access the file
    // system, and can only load built-in Node.js modules.
    // `module.createRequire()` is being used to create one that is able to
    // load files from the disk, and uses the standard CommonJS file loader
    // instead of the internal-only `require` function.
    MaybeLocal<Value> loadenv_ret = node::LoadEnvironment(
        env,
        "const publicRequire ="
        "  require('node:module').createRequire(process.cwd() + '/');"
        "globalThis.require = publicRequire;"
        "require('node:vm').runInThisContext(process.argv[1]);");

    if (loadenv_ret.IsEmpty())  // There has been a JS exception.
      return 1;

    exit_code = node::SpinEventLoop(env).FromMaybe(1);

    // node::Stop() can be used to explicitly stop the event loop and keep
    // further JavaScript from running. It can be called from any thread,
    // and will act like worker.terminate() if called from another thread.
    node::Stop(env);
  }

  return exit_code;
}
```

[CLI options]: cli.md
[`process.memoryUsage()`]: process.md#processmemoryusage
[deprecation policy]: deprecations.md
[embedtest.cc]: https://github.com/nodejs/node/blob/HEAD/test/embedding/embedtest.cc
[src/node.h]: https://github.com/nodejs/node/blob/HEAD/src/node.h