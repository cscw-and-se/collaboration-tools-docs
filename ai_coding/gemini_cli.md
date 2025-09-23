# Gemini CLI: 在终端里与 AI 无缝对话

在我们之前推荐的 AI 工具中，大多是图形化界面的 IDE 或网页应用。但对于许多热爱命令行的开发者来说，如果能在终端窗口里直接与 AI 对话，那将是一种极致的效率体验。**Gemini CLI** 正是为此而生的工具。

## 1\. 什么是 Gemini CLI？

Gemini CLI 是 Google 官方推出的一个命令行接口（Command-Line Interface）工具。它允许你直接在终端（Terminal）里，通过输入命令来与强大的 Gemini 模型进行交互。

这意味着，你可以：

  * 不离开终端，快速向 AI 提问、生成代码、解释概念。
  * 利用 Shell 的管道 (`|`) 和重定向 (`>`) 功能，将 AI 的能力与其它命令行工具（如 `cat`, `grep`, `ls`）组合起来，构建强大的自动化脚本。
  * 在 SSH 远程连接到服务器时，也能方便地使用 AI 能力。

**官方仓库**: [https://github.com/google-gemini/gemini-cli](https://github.com/google-gemini/gemini-cli)

PS: 如果下面的Gemini Cli无法顺利使用，可以体验一下[Qwen Code](https://github.com/QwenLM/qwen-code)，使用方式极度类似，但是在大陆使用更方便。

## 2\. 安装与配置

使用 Gemini CLI 非常简单，只需两步即可完成。

#### **第一步：安装 (需要 Node.js 环境)**

首先，请确保你的电脑已经安装了 [Node.js](https://nodejs.org/) (推荐使用 `nvm` 进行安装)。然后，在终端里运行以下命令来全局安装 Gemini CLI：

```bash
npm install -g @google/gemini-cli
```

也可以使用npx来安装

```bash
# Run instantly with npx
# Using npx (no installation required)
npx https://github.com/google-gemini/gemini-cli
Install globally with npm
npm install -g @google/gemini-cli
```

在Mac上可以使用Brew

```bash
# Install globally with Homebrew (macOS/Linux)
brew install gemini-cli
```

安装完成后，你可以通过运行 `gemini-cli --version` 来验证是否安装成功。

#### **第二步：配置 API 密钥**

要使用 Gemini，你需要一个 API 密钥。你可以通过以下步骤免费获取：

1.  访问 **[Google AI Studio](https://aistudio.google.com/)**。
2.  使用你的 Google 账户登录。
3.  在页面左侧菜单中，点击 **"Get API key"** -\> **"Create API key"**。
4.  复制生成的密钥字符串。

获取密钥后，回到你的终端，运行以下命令来将它配置到 Gemini CLI 中（请将 `YOUR_API_KEY` 替换为你刚刚复制的密钥）：

```bash
gemini config set GOOGLE_API_KEY=YOUR_API_KEY
```

至此，所有准备工作完成！

## 3\. 快速上手：Demo 示例

下面通过几个实用的例子，展示 Gemini CLI 的强大之处。

#### **示例一：生成代码**

直接向 AI 提问，让它为你生成一段 Python 代码。

```bash
gemini "用 Python 写一个函数，计算斐波那契数列的第 n 项，需要包含缓存优化"
```

AI 会直接在终端里返回格式化好的代码和相关解释。

#### **示例二：解释代码（管道的妙用）**

这是 Gemini CLI 最强大的用法之一。假设你有一个名为 `complex_script.js` 的文件，内容看不懂，你可以使用 `cat` 命令和管道 `|` 将文件内容传给 AI。

```bash
# 假设 complex_script.js 文件已存在
cat complex_script.js | gemini "用中文逐行解释一下这段 JavaScript 代码是做什么的"
```

这个工作流让你可以在不复制粘贴的情况下，快速理解任何代码文件的内容。

#### **示例三：生成 Shell 命令**

忘记了某个复杂的 Shell 命令怎么写？让 AI 来帮你。

```bash
gemini "写一个 shell 命令，找到当前目录下所有大于 1MB 的 .log 文件，并把它们移动到 /tmp/logs 目录下"
```

**⚠️ 安全提示**：在执行 AI 生成的、特别是涉及文件删除或移动的命令前，请务必仔细阅读和理解该命令的含义，避免造成意外损失。

#### **示例四：理解图片（多模态能力）**

Gemini 是一个多模态模型，它也能“看懂”图片。你可以使用 `--file` 参数来向它传递图片。

```bash
# 假设你有一张名为 architecture.png 的系统架构图
gemini "这张图展示了一个什么样的系统架构？描述一下它的数据流" --file architecture.png
```

这个功能对于理解图表、UI 设计稿等非常有用。

Gemini CLI 为命令行爱好者打开了一扇新的大门。它简洁、强大且高度可扩展。当你熟悉了它的基本用法后，便可以将其融入到你的日常脚本和自动化任务中，让 AI 成为你指尖的常驻“高参”。