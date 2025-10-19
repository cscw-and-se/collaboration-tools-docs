# 第三章：深入工作区 - 编辑器与文件系统

到目前为止，我们的插件还只能“飘”在 VS Code 的 UI 层。现在，我们要让它“沉”下去，深入到最核心的区域：代码编辑器本身。本章将教你如何读取和修改用户正在编辑的文件内容，以及如何精确定位到文件中的任意位置。

## 3.1 编辑器 API: 与代码对话的桥梁

VS Code 提供了一套丰富的 API 来与编辑器交互。我们的主要入口点是 `vscode.window.activeTextEditor`。

`activeTextEditor` 指向用户**当前**正在操作的那个编辑器窗口。需要特别注意的是，**它的值可能是 `undefined`**。例如，当用户没有打开任何文件，或者焦点在侧边栏上时，就没有“活动的”编辑器。因此，在使用它之前，我们必须先进行判断。

让我们创建一个新命令，用来统计用户当前选中的代码有多少个字符。

**第一步：在 `package.json` 中声明新命令**

```json:package.json
// ... (保留 "commands" 数组中的其他命令) ...
    {
      "command": "helloworld.countSelection",
      "title": "Count Selected Characters"
    }
// ...
```

**第二步：在 `extension.ts` 中实现命令**

```typescript:src/extension.ts
// ... (在 activate 函数中) ...

let countSelectionDisposable = vscode.commands.registerCommand('helloworld.countSelection', () => {
  // 1. 获取当前活动的编辑器
  const editor = vscode.window.activeTextEditor;

  // 2. 确保有活动的编辑器
  if (!editor) {
    vscode.window.showWarningMessage('请先打开一个文件并选择一些文本。');
    return;
  }

  // 3. 获取用户选择的文本
  const selection = editor.selection;
  const selectedText = editor.document.getText(selection);

  // 4. 显示统计结果
  const charCount = selectedText.length;
  vscode.window.showInformationMessage(`选中的文本共有 ${charCount} 个字符。`);
});

context.subscriptions.push(countSelectionDisposable);
```

现在按下 `F5` 运行插件。打开任意一个文件，选中一段文本，然后通过命令面板 (`Ctrl/Cmd+Shift+P`) 运行 "Count Selected Characters" 命令，看看会发生什么。

## 3.2 位置与范围 (Position & Range): 代码世界的 GPS

刚刚的代码中出现了 `editor.selection`，它的类型是 `vscode.Selection`（`Selection` 是 `Range` 的一种特殊形式）。要理解它，我们必须先了解 VS Code 是如何给代码定位的。

  * **`vscode.Position`**: 代表一个**精确的光标位置**，即一个“点”。它由两个数字组成：**行号 (line)** 和 **列号 (character)**。

    > **重要提示**：这两个数字都是**从 0 开始**计数的。所以，文件第一行第一列的位置是 `new vscode.Position(0, 0)`。

  * **`vscode.Range`**: 代表一个**连续的文本范围**，即一个“区域”。它由两个 `Position` 对象构成：**起始点 (start)** 和 **结束点 (end)**。

所以，`editor.selection` 就包含了用户当前光标选择区域的起始和结束坐标。`editor.document.getText(selection)` 正是利用这个坐标范围，从文档中“抠”出了对应的文本。

## 3.3 修改代码：插入、替换与删除

只读取不修改，显然是不够的。修改文档内容的核心 API 是 `editor.edit()`。这个方法能确保我们的修改是**安全**的（支持撤销/重做），并且性能更高。

让我们再创建一个新命令，它会在当前光标位置插入一个当前的时间戳。

**第一步：在 `package.json` 中声明新命令**

```json:package.json
// ...
    {
      "command": "helloworld.insertTimestamp",
      "title": "Insert Current Timestamp"
    }
// ...
```

**第二步：在 `extension.ts` 中实现命令**

```typescript:src/extension.ts
// ... (在 activate 函数中) ...

let insertTimestampDisposable = vscode.commands.registerCommand('helloworld.insertTimestamp', () => {
  const editor = vscode.window.activeTextEditor;
  if (!editor) {
    return; // 没有活动的编辑器，直接返回
  }

  // 获取当前时间戳
  const timestamp = new Date().toLocaleString();

  // 使用 editor.edit() 方法来执行修改
  editor.edit(editBuilder => {
    // editBuilder 提供了 insert, replace, delete 等方法
    // 我们可以直接在当前光标/选择处进行操作
    // editor.selection 可以是单个光标位置，也可以是一个范围
    editBuilder.replace(editor.selection, timestamp);
  });
});

context.subscriptions.push(insertTimestampDisposable);
```

`editor.edit()` 接受一个回调函数，并传入一个 `editBuilder` 对象。你可以在这个回调函数里，调用 `editBuilder` 的方法来“预定”一系列的修改操作。当回调函数执行完毕后，VS Code 会将所有预定的修改一次性应用到文档上。这样做的好处是，**所有这些修改会被当作一个单一的操作**，用户只需按一次 `Ctrl/Cmd+Z` 就可以全部撤销。

现在，按下 `F5` 运行你的插件。打开一个文件，尝试选中文字后运行“Count”命令，或者在任意位置运行“Insert”命令。你已经掌握了与代码交互的核心能力！