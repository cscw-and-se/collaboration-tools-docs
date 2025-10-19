# 第二章：核心交互 - 命令、UI 与上下文

在第一章，我们创建了一个只会“自言自语”的插件。现在，我们要让它学会与用户进行真正的“对话”。本章将教你如何注册更复杂的命令，并通过 VS Code 提供的基础 UI 元素（提示框、输入框等）来接收用户的指令和反馈。

## 2.1 命令 (Command): 插件功能的入口

命令是用户与插件功能交互的起点。我们在 `package.json` 的 `contributes.commands` 数组中声明一个命令，然后在 `extension.ts` 中用 `vscode.commands.registerCommand` 来为这个声明“注入灵魂”（即实现它的具体功能）。

让我们来定义一个比 "Hello World" 更酷的新命令。

**第一步：在 `package.json` 中声明新命令**

打开 `package.json` 文件，在 `commands` 数组里，追加一个新的命令声明：

```json:package.json
"contributes": {
  "commands": [
    {
      "command": "helloworld.helloWorld",
      "title": "Hello World"
    },
    {
      "command": "helloworld.askName",
      "title": "Ask Name"
    }
  ]
}
```

现在，我们的插件就有了两个命令：“Hello World” 和 “Ask Name”。接下来我们去实现第二个。

## 2.2 基础 UI 元素: 与用户对话的工具箱

VS Code 在 `vscode.window` 这个 API 命名空间下，为我们提供了丰富的 UI 组件。

#### **信息提示 (Notification)**

这是最简单的反馈方式，我们在第一章已经用过了 `showInformationMessage`。它还有两个兄弟：

  * `vscode.window.showWarningMessage`: 显示一个黄色的警告信息。
  * `vscode.window.showErrorMessage`: 显示一个红色的错误信息。

#### **输入框 (InputBox)**

当你需要用户输入一段自由文本（比如名字、搜索词）时，使用 `showInputBox`。

#### **快速选择 (QuickPick)**

当你需要用户从一个固定的列表中选择一项时，使用 `showQuickPick`。

现在，让我们把这些 UI 元素组合起来，实现 `helloworld.askName` 命令的逻辑。

**第二步：在 `extension.ts` 中实现新命令**

打开 `src/extension.ts`，在 `activate` 函数中，追加以下代码来注册我们的新命令：

```typescript:src/extension.ts
// ... (保留之前的 helloWorld 命令注册代码) ...

// 注册一个新的命令 'helloworld.askName'
let askNameDisposable = vscode.commands.registerCommand('helloworld.askName', async () => {
  // 1. 使用输入框 (InputBox) 来询问用户的名字
  const name = await vscode.window.showInputBox({
    prompt: '你好，请问你的名字是？',
    placeHolder: '例如：小明'
  });

  // 如果用户按下了 Esc 键，name 将会是 undefined
  if (!name) {
    vscode.window.showWarningMessage('操作已取消。');
    return; // 退出命令
  }

  // 2. 使用快速选择 (QuickPick) 让用户选择语言
  const language = await vscode.window.showQuickPick(['中文', 'English'], {
    placeHolder: '请选择你希望的问候语语言'
  });

  if (!language) {
    vscode.window.showWarningMessage('操作已取消。');
    return;
  }

  // 3. 根据用户的选择，使用信息提示 (Notification) 给出反馈
  if (language === '中文') {
    vscode.window.showInformationMessage(`你好, ${name}! 欢迎使用 VS Code 插件开发。`);
  } else {
    vscode.window.showInformationMessage(`Hello, ${name}! Welcome to VS Code extension development.`);
  }
});

// 同样，将新命令也添加到 context.subscriptions 中
context.subscriptions.push(askNameDisposable);
```

> **注意**：我们使用了 `async` 和 `await`。因为 `showInputBox` 和 `showQuickPick` 都是**异步**操作，它们会等待用户的输入，而不会阻塞整个 VS Code 编辑器。`async/await` 能让我们用像写同步代码一样的方式来处理异步流程。

## 2.3 状态栏 (Status Bar): 持久化的信息窗口

状态栏是位于 VS Code 窗口最底部的一条区域，非常适合用来显示一些持久化的、不打扰用户的状态信息。

**第三步：在 `extension.ts` 中创建状态栏项**

一个状态栏项应该在插件被激活时就创建，并一直存在。所以我们直接在 `activate` 函数的顶层来创建它。

```typescript:src/extension.ts
// 在 activate 函数的开始部分添加
let myStatusBarItem: vscode.StatusBarItem;

// ... (这是 activate 函数的主体) ...
export function activate(context: vscode.ExtensionContext) {

  // ... (保留之前的命令注册代码) ...

  // 创建一个状态栏项
  myStatusBarItem = vscode.window.createStatusBarItem(vscode.StatusBarAlignment.Right, 100);

  // 为状态栏项设置属性
  myStatusBarItem.text = `$(smiley) Greet`; // `$(icon-name)` 可以使用 VS Code 内置的 Octicons 图标
  myStatusBarItem.tooltip = '点击这里开始问候';
  myStatusBarItem.command = 'helloworld.askName'; // 点击状态栏时，触发 'askName' 命令

  // 将状态栏项显示出来
  myStatusBarItem.show();

  // 将状态栏项也加入到可销毁对象中
  context.subscriptions.push(myStatusBarItem);
}
```

  * `createStatusBarItem` 的第一个参数决定了它在状态栏的左边还是右边，第二个参数是优先级（数字越大越靠左/右）。
  * `myStatusBarItem.command` 是一个非常实用的属性，它可以让我们的状态栏按钮变成一个快捷方式，直接触发指定的命令。

现在，按下 `F5` 运行你的插件。你会看到右下角出现了一个带笑脸的 "Greet" 按钮。点击它，或者使用 `Ctrl/Cmd+Shift+P` 运行 "Ask Name" 命令，体验一下你刚刚打造的完整交互流程吧！