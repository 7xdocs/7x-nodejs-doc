# 环境变量

环境变量是与 Node.js 进程运行环境相关联的变量。

## CLI 环境变量

有一组环境变量可以定义，用于自定义 Node.js 的行为，更多详细信息请参阅 [CLI 环境变量文档][]。

## `process.env`

与环境变量交互的基本 API 是 `process.env`，它包含一个预填充了用户环境变量的对象，这些变量可以被修改和扩展。

更多详细信息请参阅 [`process.env` 文档][]。

## DotEnv

一组用于处理在 `.env` 文件中定义的额外环境变量的实用工具。

> Stability: 2 - Stable

<!--introduced_in=v20.12.0-->

### .env 文件

`.env` 文件（也称为 dotenv 文件）是定义环境变量的文件，Node.js 应用程序随后可以与之交互（由 [dotenv][] 包推广）。

以下是一个基础 `.env` 文件内容的示例：

```text
MY_VAR_A = "my variable A"
MY_VAR_B = "my variable B"
```

这种类型的文件在各种不同的编程语言和平台中使用，但没有正式的规范，因此 Node.js 定义了自己的规范，如下所述。

`.env` 文件是一个包含键值对的文件，每个对由一个变量名后跟等号（`=`），再后跟变量值表示。

此类文件的名称通常是 `.env` 或以 `.env` 开头（例如 `.env.dev`，其中 `dev` 表示特定的目标环境）。这是推荐的命名方案，但不是强制性的，dotenv 文件可以有任何任意的文件名。

#### 变量名

有效的变量名只能包含字母（大写或小写）、数字和下划线（`_`），并且不能以数字开头。

更具体地说，有效的变量名必须匹配以下正则表达式：

```text
^[a-zA-Z_]+[a-zA-Z0-9_]*$
```

推荐的约定是使用大写字母和下划线，必要时使用数字，但任何符合上述定义的变量名都可以正常工作。

例如，以下是一些有效的变量名：`MY_VAR`, `MY_VAR_1`, `my_var`, `my_var_1`, `myVar`, `My_Var123`，而这些是无效的：`1_VAR`, `'my-var'`, `"my var"`, `VAR_#1`。

#### 变量值

变量值由任意文本组成，可以选择用单引号（`'`）或双引号（`"`）括起来。

带引号的变量可以跨越多行，而不带引号的变量则仅限于单行。

请注意，当被 Node.js 解析时，所有值都被解释为文本，这意味着任何值都会在 Node.js 内部生成一个 JavaScript 字符串。例如，以下值：`0`, `true` 和 `{ "hello": "world" }` 将分别生成字面字符串 `'0'`, `'true'` 和 `'{ "hello": "world" }'`，而不是数字零、布尔值 `true` 和具有 `hello` 属性的对象。

有效变量的示例：

```text
MY_SIMPLE_VAR = a simple single line variable
MY_EQUALS_VAR = "this variable contains an = sign!"
MY_HASH_VAR = 'this variable contains a # symbol!'
MY_MULTILINE_VAR = '
this is a multiline variable containing
two separate lines\nSorry, I meant three lines'
```

#### 空格

变量键和值周围的前导和尾随空白字符将被忽略，除非它们被引号括起来。

例如：

```text
   MY_VAR_A   =    my variable a
    MY_VAR_B   =    '   my variable b   '
```

将被视为等同于：

```text
MY_VAR_A = my variable a
MY_VAR_B = '   my variable b   '
```

#### 注释

井号（`#`）字符表示注释的开始，意味着该行的其余部分将被完全忽略。

然而，在引号内找到的井号被视为任何其他标准字符。

例如：

```text
# This is a comment
MY_VAR = my variable # This is also a comment
MY_VAR_A = "# this is NOT a comment"
```

#### `export` 前缀

`export` 关键字可以选择性地添加到变量声明的前面，该关键字将被文件的所有处理完全忽略。

这很有用，使得该文件可以在 shell 终端中直接 sourcing，而无需修改。

示例：

```text
export MY_VAR = my variable
```

### CLI 选项

可以通过以下 CLI 选项之一使用 `.env` 文件来填充 `process.env` 对象：

* [`--env-file=file`][]

* [`--env-file-if-exists=file`][]

### 编程 API

以下两个函数允许您直接与 `.env` 文件交互：

* [`process.loadEnvFile`][] 加载一个 `.env` 文件并将其变量填充到 `process.env` 中

* [`util.parseEnv`][] 解析 `.env` 文件的原始内容，并将其值返回到一个对象中

[CLI 环境变量文档]: cli.md#environment-variables_1
[`--env-file-if-exists=file`]: cli.md#--env-file-if-existsfile
[`--env-file=file`]: cli.md#--env-filefile
[`process.env` 文档]: process.md#processenv
[`process.loadEnvFile`]: process.md#processloadenvfilepath
[`util.parseEnv`]: util.md#utilparseenvcontent
[dotenv]: https://github.com/motdotla/dotenv