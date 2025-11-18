# 写在前面

> 本文已借助Github Actions及Pages自动化部署了可阅读的网页：https://cscw-and-se.github.io/collaboration-tools-docs/#/

## 初衷

构建这个文档项目，主要源于两个初衷：

第一，在每次带领本科生进行项目实践的过程中，我发现有些知识点，例如“什么是实时协同编程（RCP）”、“Git 的团队协作流程是怎样的”，总是在被重复地讲授。我希望能够将这些知识系统性地沉淀下来，形成一份统一的参考资料。这不仅能减少我个人的重复性劳动，更重要的是，可以让新加入的同学快速上手，将精力更多地投入到有创造性的工作里。

第二，我们正在研究的实时协同项目本身技术栈较为复杂，涉及到 VS Code 插件开发、TypeScript、Y.js 等多种技术。对于新人来说，这无疑带来了较高的学习门槛。我希望通过这份文档，能够尽可能地降低上手难度，分享探索过程中积累的经验与“坑”。更长远地看，我不希望这些集体的知识随着个体的毕业而流失，导致项目无人能够顺畅地维护和迭代。

因此，我萌生了撰写这份文档的想法，希望能为团队知识的传承贡献一份力量。

## 如何运行本文档

为了提供简单、轻量的阅读和编辑体验，本文档基于 **[docsify](https://docsify.js.org/#/)** 框架构建。你只需要使用 Markdown 语法便可以轻松编辑，并通过以下步骤在本地运行。

### 第一步：安装 Node.js (推荐使用 nvm)

`nvm` (Node Version Manager) 是一个强大的 Node.js 版本管理工具，可以让你在同一台机器上轻松切换不同版本的 Node.js。

- **nvm GitHub 仓库**: [https://github.com/nvm-sh/nvm](https://github.com/nvm-sh/nvm)

- **安装 nvm**:

打开你的终端（Terminal），运行以下命令之一来安装 nvm：

```bash
# 使用 curl
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
# 或者使用 wget
wget -qO- https://raw.githubusercontent.com/nvm-sh/nvm/v0.39.7/install.sh | bash
```

安装完成后，请**重启你的终端**，然后运行 `nvm --version` 来验证是否安装成功。

- 如果你使用的是Windows系统，请参考使用[nvm-windows](https://github.com/coreybutler/nvm-windows)

- **使用 nvm 安装 Node.js**:

```bash
# 安装22版本
nvm install 22
# 使用22版本
nvm use 22
# 查看已安装的 Node.js 版本
nvm ls
# 查看当前使用的 Node.js 版本
nvm current
# 查看 Node.js 的版本信息
node -v
# 查看 npm 的版本信息
npm -v
```

### 第二步：安装 docsify-cli

`docsify-cli` 是 docsify 的命令行工具，可以帮你轻松地初始化和预览网站。

```bash
npm install -g docsify-cli
```

### 第三步：克隆并运行文档

现在，你可以克隆文档的仓库并在本地启动一个实时预览服务。

```bash
# 克隆文档仓库
git clone https://github.com/cscw-and-se/collaboration-tools-docs.git

# 进入文档目录
cd collaboration-tools-docs

# 启动本地服务
docsify serve
```

服务启动后，你会在终端看到如下提示：

```
Serving C:\...\collaboration-tools-docs now.
Listening at http://localhost:3000
```

此时，在浏览器中打开 `http://localhost:3000` 即可访问文档。

## 关于AI

在文档写作的过程中我会使用AI来生成文本，同时作为这份文档的作者，我也是鼓励大家多使用AI来辅助写作和写代码，这点可以在后面的「使用AI编程」章节中体现。反正这里没有维普来查我^^。

-----

本文档尚在不断完善中，如有不足之处，还请多多包涵与指正。
