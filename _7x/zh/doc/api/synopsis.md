# 使用方法及示例

## 使用方法

<!--introduced_in=v0.10.0-->

<!--type=misc-->

`node [options] [V8 options] [script.js | -e "script" | - ] [arguments]`

更多信息请参见[命令行选项][]文档。

## 示例

一个用Node.js编写的[Web服务器][]示例，它会响应`'Hello, World!'`：

本文档中的命令以`$`或`>`开头，以模拟它们在用户终端中的显示方式。不要包含`$`和`>`字符。它们仅用于标识每个命令的开始。

不以`$`或`>`开头的行表示前一个命令的输出。

首先，请确保已下载并安装Node.js。更多安装信息请参见[通过包管理器安装Node.js][]。

现在，创建一个名为`projects`的空项目文件夹，然后进入该文件夹。

Linux和Mac：

```bash
mkdir ~/projects
cd ~/projects
```

Windows命令提示符：

```powershell
mkdir %USERPROFILE%\projects
cd %USERPROFILE%\projects
```

Windows PowerShell：

```powershell
mkdir $env:USERPROFILE\projects
cd $env:USERPROFILE\projects
```

接下来，在`projects`文件夹中创建一个新的源文件，命名为`hello-world.js`。

用你喜欢的文本编辑器打开`hello-world.js`，并粘贴以下内容：

```js
const http = require('node:http');

const hostname = '127.0.0.1';
const port = 3000;

const server = http.createServer((req, res) => {
  res.statusCode = 200;
  res.setHeader('Content-Type', 'text/plain');
  res.end('Hello, World!\n');
});

server.listen(port, hostname, () => {
  console.log(`Server running at http://${hostname}:${port}/`);
});
```

保存文件。然后，在终端窗口中，输入以下命令运行`hello-world.js`文件：

```bash
node hello-world.js
```

终端中应该会显示如下输出：

```console
Server running at http://127.0.0.1:3000/
```

现在，打开你喜欢的Web浏览器，访问`http://127.0.0.1:3000`。

如果浏览器显示字符串`Hello, World!`，则表示服务器正常运行。

[命令行选项]: cli.md#options
[通过包管理器安装Node.js]: https://nodejs.org/en/download/package-manager/
[Web服务器]: http.md
