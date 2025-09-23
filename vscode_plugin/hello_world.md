# 第一章：基础入门 - Hello, World\!

## 1.1 环境搭建与项目创建

在开始施展魔法之前，我们需要先准备好“魔杖”和“工作台”。

#### **必备条件**

1.  **Visual Studio Code**: 你需要安装 VS Code 本体。
2.  **Node.js**: 插件是基于 Node.js 运行的，因此你必须安装它。

#### **第一步：安装脚手架**

VS Code 官方提供了一个非常好用的项目生成器（脚手架），它可以一键帮我们生成标准的插件项目模板。打开你的终端（Terminal 或 Windows 的 PowerShell/CMD），运行以下命令来全局安装它：

```bash
npm install -g yo generator-code
```

  * `yo` (Yeoman) 是一个通用的脚手架工具。
  * `generator-code` 则是专门用于生成 VS Code 插件模板的。

#### **第二步：生成你的项目**

安装完成后，在终端里，`cd` 到一个你希望存放项目的空文件夹中，然后运行：

```bash
yo code
```

接下来，脚手架会像一个向导一样，问你一系列问题来配置你的初始项目。对于初学者，建议按以下方式回答：

```
? What type of extension do you want to create? New Extension (TypeScript)
? What's the name of your extension? HelloWorld
? What's the identifier of your extension? (helloworld) # 直接回车即可
? What's the description of your extension? My first VS Code extension.
? Initialize a git repository? Yes
? Bundle the source code with webpack? No # 初学建议选 No，可以简化项目结构
? Which package manager to use? npm
```

> **提示**：我们强烈推荐选择 **TypeScript**。它是 VS Code 插件开发的官方推荐语言，能带来更好的代码提示和类型安全，让开发过程更轻松。

完成后，你会看到一个名为 `HelloWorld` 的新文件夹。用 VS Code 打开它：

```bash
cd HelloWorld
code .
```

恭喜你，你的第一个插件项目已经创建成功！

## 1.2 项目结构与生命周期解析

当你打开项目，会看到脚手架为我们生成了一系列文件。我们先来认识几个最核心的：

```
.
├── .vscode
│   └── launch.json     # 运行和调试的配置文件
├── src
│   └── extension.ts    # 插件的“心脏”：源代码入口
├── package.json        # 插件的“身份证”：清单文件
└── ...
```

#### **`package.json`：插件的“身份证” (Manifest)**

这个文件向 VS Code 描述了你的插件的一切。它不是普通的 `package.json`，里面包含了很多 VS Code 专用的字段。

  * `name`, `publisher`, `version`: 插件的基础信息。
  * `engines.vscode`: 声明你的插件兼容哪个版本的 VS Code。
  * `main`: 指向插件编译后的入口 JS 文件，例如 `./out/extension.js`。
  * `activationEvents`: **(核心概念)** 定义了你的插件**在什么时候被激活**。为了性能，VS Code 不会一启动就加载所有插件。这里的设置 `onCommand:helloworld.helloWorld` 意味着：只有当用户第一次尝试运行名为 `helloworld.helloWorld` 的命令时，我们的插件才会被激活加载。
  * `contributes`: **(核心概念)** 这是你为 VS Code “贡献”新功能的地方。比如，`"commands"` 数组就定义了一个新的命令，它的 `title` 是显示给用户看的，`command` 则是这个命令的唯一 ID。

#### **`src/extension.ts`：插件的“心脏” (Entry Point)**

这个文件是你的插件代码的起点，它导出了两个生命周期函数：

  * `activate(context)`: 当 `activationEvents` 被触发时，`activate` 函数会被**执行一次**。这是你的插件的**主入口**，所有的初始化、命令注册、事件监听都应该在这里完成。
  * `deactivate()`: 当插件被禁用或 VS Code 关闭时，`deactivate` 函数会被调用，你可以在这里进行一些清理工作。

让我们看一下 `activate` 函数里的核心代码：

```typescript
// 导入 vscode 的 API 模块
import * as vscode from 'vscode';

export function activate(context: vscode.ExtensionContext) {

    // 1. 注册一个命令，这个命令的 ID 必须和 package.json 中的 command ID 一致
    let disposable = vscode.commands.registerCommand('helloworld.helloWorld', () => {
        
        // 2. 当命令被执行时，这里的代码就会运行
        vscode.window.showInformationMessage('Hello World from HelloWorld!');
    });

    // 3. 将注册的命令添加到“订阅”中，以便在插件禁用时自动销毁
    context.subscriptions.push(disposable);
}
```

**逻辑链路**：用户触发 `onCommand` 事件 -\> VS Code 激活插件 -\> 调用 `activate` 函数 -\> `registerCommand` 将一个命令 ID 和一个回调函数关联起来。当用户执行该命令时，回调函数被触发，弹出一个信息框。

## 1.3 运行、调试与日志输出

VS Code 提供了世界一流的插件调试体验。

#### **运行你的插件**

1.  直接在你的项目窗口中按下 `F5` 键。
2.  VS Code 会开始编译你的 TS 代码，然后启动一个新的 VS Code 窗口。这个新窗口被称为\*\*“扩展开发宿主” (Extension Development Host)\*\*，它是一个“干净”的 VS Code 环境，并且已经自动加载了你正在开发的插件。

#### **触发命令**

1.  在**新打开的“扩展开发宿主”窗口**中，按下 `Ctrl+Shift+P` (Windows/Linux) 或 `Cmd+Shift+P` (macOS) 打开**命令面板**。
2.  输入你定义的命令标题 "Hello World"，然后回车。
3.  你会看到窗口右下角弹出了 "Hello World from HelloWorld\!" 的信息框。

#### **设置断点进行调试**

1.  回到你原来的**项目代码窗口**。
2.  打开 `src/extension.ts` 文件。
3.  在 `vscode.window.showInformationMessage(...)` 这一行的行号左边，用鼠标点击一下，你会看到一个**红点**，这就是断点。
4.  再次在“扩展开发宿主”窗口中运行 "Hello World" 命令。
5.  你会发现，代码执行到断点处就暂停了！你的主窗口会自动切换到“运行和调试”视图，在这里你可以查看和监控变量的值、调用堆栈等，就像调试普通程序一样。

#### **使用 `console.log`**

`console.log()` 是最简单直接的调试方式。

1.  在你的命令回调函数中加入一行 `console.log('我的命令被执行了！');`。
2.  按下 `F5` 重新启动调试。
3.  当你在“扩展开发宿主”窗口中运行命令后，回到**主项目窗口**，在下方的面板中找到\*\*“调试控制台” (Debug Console)\*\*，你就能看到这行日志输出。

恭喜你！你已经完成了 VS Code 插件开发的第一步，也是最重要的一步。接下来，我们将学习如何为插件添加更丰富的交互功能。