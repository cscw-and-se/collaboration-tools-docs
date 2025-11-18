# 第四章：构建复杂应用 - 视图、配置与状态

到目前为止，我们与用户的交互还仅限于命令和短暂的弹窗。但一个功能丰富的插件，需要一个持久化的 UI 界面来展示信息和提供操作入口。本章将带你学习构建侧边栏和完全自定义的 Webview 界面，这是插件 UI 开发的“两把利剑”。

## 4.1 侧边栏视图 (Sidebar View): 信息的“陈列架”

你每天都在使用侧边栏视图——文件浏览器、搜索面板、Git 面板，这些都是。VS Code 允许我们创建自己的侧边栏视图，最常见的就是**树状视图 (Tree View)**。

要创建一个树状视图，我们需要做两件事：

1.  在 `package.json` 中告诉 VS Code：“嘿，我要在侧边栏占个位置！”
2.  在 `extension.ts` 中提供一个“数据供应器” (`TreeDataProvider`)，告诉 VS Code 这个位置里要显示什么内容。

#### **第一步：在 `package.json` 中声明视图**

打开 `package.json`，在 `contributes` 对象中，新增一个 `"views"` 属性：

```json:package.json
"contributes": {
  "commands": [ ... ],
  "views": {
    "explorer": [
      {
        "id": "helloWorld.dependenciesView",
        "name": "项目依赖"
      }
    ]
  }
}
```

  * `"explorer"`: 表示我们想把视图添加到 VS Code 的“文件浏览器”这个容器里。你也可以选择 `"debug"` (调试侧边栏) 等其他位置。
  * `"id"`: 这个视图的唯一标识符，稍后在代码中会用到。
  * `"name"`: 显示在侧边栏顶部的标题。

#### **第二步：实现 `TreeDataProvider`**

数据供应器是一个遵循 `vscode.TreeDataProvider` 接口的类。它必须实现两个核心方法：

  * `getTreeItem(element)`: 接收一个数据项，返回它的可视化表现（`TreeItem`），决定了“它长什么样”。
  * `getChildren(element)`: 接收一个父数据项，返回它的所有子数据项组成的数组，决定了“它下面有什么”。如果 `element` 是 `undefined`，则表示获取顶层项目。

让我们创建一个简单的 `TreeDataProvider`，用来显示项目中硬编码的几个“依赖项”。

在 `src` 文件夹下新建一个文件 `dependencies.ts`:

```typescript:src/dependencies.ts
import * as vscode from 'vscode';

// TreeDataProvider 需要一个泛型类型，代表我们树中每个节点的数据结构。
// 这里我们简单地使用 vscode.TreeItem 作为节点的数据和UI表现。
export class DepNodeProvider implements vscode.TreeDataProvider<vscode.TreeItem> {

  getTreeItem(element: vscode.TreeItem): vscode.TreeItem {
    return element;
  }

  getChildren(element?: vscode.TreeItem): Thenable<vscode.TreeItem[]> {
    // 如果 element 存在，说明是在请求子节点，我们这里没有子节点，返回空数组
    if (element) {
      return Promise.resolve([]);
    }
    
    // 如果 element 不存在，说明是在请求根节点
    // 我们返回一个硬编码的依赖项列表
    const item1 = new vscode.TreeItem('lodash', vscode.TreeItemCollapsibleState.None);
    item1.description = '4.17.21';
    item1.tooltip = 'A modern JavaScript utility library.';

    const item2 = new vscode.TreeItem('moment', vscode.TreeItemCollapsibleState.None);
    item2.description = '2.29.1';
    item2.tooltip = 'Parse, validate, manipulate, and display dates and times.';

    return Promise.resolve([item1, item2]);
  }
}
```

  * `vscode.TreeItemCollapsibleState.None`: 表示这个节点是“叶子节点”，不能再展开。如果想让它像文件夹一样可以展开，可以使用 `CollapsibleState.Collapsed` 或 `Expanded`。

#### **第三步：在 `extension.ts` 中注册**

最后，在 `activate` 函数中，将我们的数据供应器注册到 VS Code。

```typescript:src/extension.ts
import { DepNodeProvider } from './dependencies'; // 导入我们刚刚创建的类

export function activate(context: vscode.ExtensionContext) {
  // ... (保留之前的代码) ...
  
  // 注册侧边栏视图
  vscode.window.registerTreeDataProvider(
    'helloWorld.dependenciesView', // 这个 ID 必须和 package.json 中的一致
    new DepNodeProvider()
  );
}
```

现在按下 `F5` 运行插件，你会发现在文件浏览器的下方，出现了一个名为“项目依赖”的新面板，里面展示了我们定义的两个库。

## 4.2 Webview: 你的“内置浏览器”

侧边栏功能强大，但UI样式是固定的。如果你想构建一个完全自定义、不受限制的复杂界面（比如一个图表、一个登录表单、一个欢迎页面），就需要使用 **Webview**。

**Webview 本质上是在 VS Code 内部嵌入的一个 `iframe`**。它是一个“沙箱化”的浏览器环境，你可以在里面使用任何你熟悉的 Web 技术：`HTML`, `CSS`, 和 `JavaScript`。

#### **实现一个简单的 Webview**

让我们创建一个命令，当执行时，弹出一个 Webview 面板。

**第一步：在 `package.json` 注册命令**

```json
{
  "command": "helloworld.showWebview",
  "title": "Show Webview Panel"
}
```

**第二步：在 `extension.ts` 实现命令**

```typescript:src/extension.ts
// ... (在 activate 函数中) ...

let showWebviewDisposable = vscode.commands.registerCommand('helloworld.showWebview', () => {
  // 创建并显示一个新的 Webview 面板
  const panel = vscode.window.createWebviewPanel(
    'helloWebview', // 视图类型的标识符
    'Hello Webview', // 面板的标题
    vscode.ViewColumn.One, // 在哪个编辑列中显示
    {
      enableScripts: true // 必须开启，允许 Webview 运行 JavaScript
    }
  );

  // 设置 Webview 的 HTML 内容
  panel.webview.html = getWebviewContent();
});
context.subscriptions.push(showWebviewDisposable);

// ... (在 activate 函数外部) ...

function getWebviewContent() {
  return `<!DOCTYPE html>
    <html lang="en">
    <head>
        <meta charset="UTF-8">
        <meta name="viewport" content="width=device-width, initial-scale=1.0">
        <title>Hello Webview</title>
    </head>
    <body>
        <h1>Hello from our Webview!</h1>
        <p>This is a custom panel built with HTML.</p>
    </body>
    </html>`;
}
```

按下 `F5` 并运行 "Show Webview Panel" 命令，你会看到一个全新的编辑标签页，里面渲染了我们编写的 HTML。

## 4.3 插件与 Webview 通信

Webview 是一个独立的沙箱，它的 JavaScript 环境和你的插件 `extension.ts` 的 Node.js 环境是**完全隔离的**。它们之间不能直接调用函数或共享变量。

唯一的沟通方式是**消息传递 (Message Passing)**，就像两个隔着墙的人写信一样。

#### **插件 (Extension) -\> Webview**

插件可以通过 `panel.webview.postMessage()` 方法向 Webview 发送消息。

#### **Webview -\> 插件 (Extension)**

Webview 中的 JavaScript 需要先通过一个特殊的 `acquireVsCodeApi()` 函数获取一个 API 对象，然后使用这个对象的 `postMessage()` 方法向插件发送消息。插件则通过 `panel.webview.onDidReceiveMessage()` 来监听这些消息。

#### **通信示例**

让我们来改造一下 Webview，让它们可以互相“打招呼”。

**第一步：修改 `getWebviewContent`**

```typescript
function getWebviewContent() {
  return `<!DOCTYPE html>
    <html lang="en">
    <head> ... </head>
    <body>
        <h1>Hello from our Webview!</h1>
        <button id="greetButton">Say Hello to Extension</button>

        <script>
            // 1. 获取 VS Code API 对象
            const vscode = acquireVsCodeApi();

            document.getElementById('greetButton').addEventListener('click', () => {
                // 2. 向插件发送消息
                vscode.postMessage({
                    command: 'hello',
                    text: '你好，我是 Webview！'
                });
            });

            // 4. 监听来自插件的消息
            window.addEventListener('message', event => {
                const message = event.data; // 插件传递过来的消息
                if (message.command === 'refactor') {
                    const counter = document.getElementById('counter');
                    counter.textContent = parseInt(counter.textContent) + 1;
                    vscode.window.showInformationMessage(message.text);
                }
            });
        </script>
    </body>
    </html>`;
}
```

**第二步：修改 Webview 创建逻辑，添加消息监听**

```typescript
let showWebviewDisposable = vscode.commands.registerCommand('helloworld.showWebview', () => {
  const panel = vscode.window.createWebviewPanel(...);
  panel.webview.html = getWebviewContent();

  // 3. 监听来自 Webview 的消息
  panel.webview.onDidReceiveMessage(
    message => {
      switch (message.command) {
        case 'hello':
          vscode.window.showInformationMessage(message.text);
          return;
      }
    },
    undefined,
    context.subscriptions
  );
});
```

现在，当你点击 Webview 中的按钮时，插件会收到消息并弹出一个提示框，实现了从 Webview 到插件的通信。反向通信的逻辑也类似，你可以通过 `panel.webview.postMessage` 从插件向 Webview 发送指令。