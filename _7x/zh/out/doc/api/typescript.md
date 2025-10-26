# Modules: TypeScript

<!-- YAML
changes:
  - version: v24.3.0
    pr-url: https://github.com/nodejs/node/pull/58643
    description: Type stripping no longer emits an experimental warning.
  - version: v23.6.0
    pr-url: https://github.com/nodejs/node/pull/56350
    description: Type stripping is enabled by default.
  - version: v22.7.0
    pr-url: https://github.com/nodejs/node/pull/54283
    description: Added `--experimental-transform-types` flag.
-->

<!--introduced_in=v22.6.0-->

> Stability: 1.2 - Release candidate

## 启用

在 Node.js 中有两种方式启用运行时 TypeScript 支持：

1. 为了获得 TypeScript 所有语法和特性的[完整支持][]，包括使用任何版本的 TypeScript，请使用第三方包。

2. 为了轻量级支持，你可以使用内置的[类型清除][]支持。

## 完整的 TypeScript 支持

要使用 TypeScript 并获得对所有 TypeScript 特性的完整支持，包括 `tsconfig.json`，你可以使用第三方包。这些说明使用 [`tsx`][] 作为示例，但还有许多其他类似的库可用。

1. 使用你的项目正在使用的任何包管理器将该包安装为开发依赖。例如，使用 `npm`：

   ```bash
   npm install --save-dev tsx
   ```

2. 然后你可以通过以下方式运行你的 TypeScript 代码：

   ```bash
   npx tsx your-file.ts
   ```

   或者，你也可以通过以下方式使用 `node` 运行：

   ```bash
   node --import=tsx your-file.ts
   ```

## 类型清除

<!-- YAML
added: v22.6.0
-->

默认情况下，Node.js 将执行仅包含可擦除 TypeScript 语法的 TypeScript 文件。
Node.js 将用空白替换 TypeScript 语法，
并且不执行类型检查。
要启用不可擦除的 TypeScript 语法的转换（这需要 JavaScript 代码生成），
例如 `enum` 声明、参数属性，请使用 [`--experimental-transform-types`][] 标志。
要禁用此功能，请使用 [`--no-experimental-strip-types`][] 标志。

Node.js 会忽略 `tsconfig.json` 文件，因此
依赖于 `tsconfig.json` 中设置的功能，
例如 paths 或将新的 JavaScript 语法转换为旧标准，
是故意不支持的。要获得完整的 TypeScript 支持，请参阅[完整的 TypeScript 支持][]。

类型清除功能设计为轻量级。
通过故意不支持需要 JavaScript 代码生成的语法，
并用空白替换内联类型，Node.js 可以在不需要源映射的情况下运行 TypeScript 代码。

类型清除与大多数版本的 TypeScript 兼容，
但我们推荐使用 5.8 或更高版本，并配合以下 `tsconfig.json` 设置：

```json
{
  "compilerOptions": {
     "noEmit": true, // 可选 - 参见下面的说明
     "target": "esnext",
     "module": "nodenext",
     "rewriteRelativeImportExtensions": true,
     "erasableSyntaxOnly": true,
     "verbatimModuleSyntax": true
  }
}
```

如果你打算只执行 `*.ts` 文件（例如构建脚本），请使用 `noEmit` 选项。如果你打算分发 `*.js` 文件，则不需要此标志。

### 确定模块系统

Node.js 在 TypeScript 文件中支持 [CommonJS][] 和 [ES Modules][] 两种语法。Node.js 不会将一种模块系统转换为另一种；如果你希望你的代码作为 ES 模块运行，你必须使用 `import` 和 `export` 语法，如果你希望你的代码作为 CommonJS 运行，你必须使用 `require` 和 `module.exports`。

* `.ts` 文件的模块系统将[以与 `.js` 文件相同的方式确定][]。要使用 `import` 和 `export` 语法，请在最近的父级 `package.json` 中添加 `"type": "module"`。
* `.mts` 文件将始终作为 ES 模块运行，类似于 `.mjs` 文件。
* `.cts` 文件将始终作为 CommonJS 模块运行，类似于 `.cjs` 文件。
* `.tsx` 文件不受支持。

与 JavaScript 文件一样，在 `import` 语句和 `import()` 表达式中[文件扩展名是强制性的][]：`import './file.ts'`，而不是 `import './file'`。由于向后兼容性，在 `require()` 调用中文件扩展名也是强制性的：`require('./file.ts')`，而不是 `require('./file')`，类似于在 CommonJS 文件的 `require` 调用中 `.cjs` 扩展名是强制性的一样。

`tsconfig.json` 选项 `allowImportingTsExtensions` 将允许 TypeScript 编译器 `tsc` 对包含 `.ts` 扩展名的 `import` 说明符的文件进行类型检查。

### TypeScript 特性

由于 Node.js 仅删除内联类型，任何涉及用新的 JavaScript 语法_替换_ TypeScript 语法的 TypeScript 特性都会报错，除非传递了 [`--experimental-transform-types`][] 标志。

最需要转换的特性包括：

* `Enum` 声明
* 包含运行时代码的 `namespace`
* 包含运行时代码的旧版 `module`
* 参数属性
* 导入别名

不包含运行时代码的 `namespaces` 和 `module` 是受支持的。
以下示例将正确工作：

```ts
// 这个命名空间正在导出一个类型
namespace TypeOnly {
   export type A = string;
}
```

这将导致 [`ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`][] 错误：

```ts
// 这个命名空间正在导出一个值
namespace A {
   export let x = 1
}
```

由于装饰器目前是 [TC39 Stage 3 提案](https://github.com/tc39/proposal-decorators)并且很快将被 JavaScript 引擎支持，
它们不会被转换并将导致解析器错误。
这是一个临时限制，将在未来解决。

此外，Node.js 不读取 `tsconfig.json` 文件，并且不支持依赖于 `tsconfig.json` 中设置的功能，例如 paths 或将新的 JavaScript 语法转换为旧标准。

### 不使用 `type` 关键字导入类型

由于类型清除的性质，需要 `type` 关键字来正确清除类型导入。没有 `type` 关键字，Node.js 会将导入视为值导入，这将导致运行时错误。tsconfig 选项 [`verbatimModuleSyntax`][] 可用于匹配此行为。

以下示例将正确工作：

```ts
import type { Type1, Type2 } from './module.ts';
import { fn, type FnParams } from './fn.ts';
```

这将导致运行时错误：

```ts
import { Type1, Type2 } from './module.ts';
import { fn, FnParams } from './fn.ts';
```

### 非文件形式的输入

可以为 `--eval` 和 STDIN 启用类型清除。模块系统将由 `--input-type` 确定，就像 JavaScript 一样。

REPL、`--check` 和 `inspect` 不支持 TypeScript 语法。

### 源映射

由于内联类型被空白替换，对于堆栈跟踪中正确的行号来说源映射是不必要的；Node.js 不会生成它们。
当启用 [`--experimental-transform-types`][] 时，默认启用源映射。

### 依赖项中的类型清除

为了阻止包作者发布用 TypeScript 编写的包，Node.js 拒绝处理 `node_modules` 路径下文件夹内的 TypeScript 文件。

### 路径别名

[`tsconfig` "paths"][] 不会被转换，因此会产生错误。最接近的可用功能是 [子路径导入][]，但有限制，即它们需要以 `#` 开头。

[CommonJS]: modules.md
[ES Modules]: esm.md
[完整的 TypeScript 支持]: #full-typescript-support
[`--experimental-transform-types`]: cli.md#--experimental-transform-types
[`--no-experimental-strip-types`]: cli.md#--no-experimental-strip-types
[`ERR_UNSUPPORTED_TYPESCRIPT_SYNTAX`]: errors.md#err_unsupported_typescript_syntax
[`tsconfig` "paths"]: https://www.typescriptlang.org/tsconfig/#paths
[`tsx`]: https://tsx.is/
[`verbatimModuleSyntax`]: https://www.typescriptlang.org/tsconfig/#verbatimModuleSyntax
[文件扩展名是强制性的]: esm.md#mandatory-file-extensions
[完整支持]: #full-typescript-support
[子路径导入]: packages.md#subpath-imports
[以与 `.js` 文件相同的方式确定]: packages.md#determining-module-system
[类型清除]: #type-stripping
