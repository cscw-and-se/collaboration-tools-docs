# VS Code 插件开发指南大纲

### **前言：为二次开发而生**

本教程的核心目标，是为本项目中涉及VS Code插件部分的二次开发铺平道路，让团队新成员能快速理解 VS Code 插件的运行机制，并有能力在现有项目上添加新功能或修复问题。

我们将从零开始，系统性地学习一个现代 VS Code 插件所必需的各项核心技术。

#### **核心必读资源**

在开始之前，请务必将以下官方页面加入你的书签，它们是你未来开发中最权威、最可靠的参考：

* **[官方入门教程 (Your First Extension)](https://code.visualstudio.com/api/get-started/your-first-extension)**: 官方手把手教你创建第一个插件。
* **[官方 API 参考文档 (API Reference)](https://code.visualstudio.com/api/references/vscode-api)**: VS Code 所有可用 API 的“新华字典”，最权威的查询手册。
* **[官方插件示例仓库 (Extension Samples)](https://github.com/microsoft/vscode-extension-samples)**: 包含数十个功能各异的官方示例项目，是学习特定功能的最佳“源码教科书”。

### **第一章：基础入门 - Hello, World!**
* `1.1 环境搭建与项目创建`
* `1.2 项目结构与生命周期解析`
* `1.3 运行、调试与日志输出`

### **第二章：核心交互 - 命令、UI 与上下文**
* `2.1 命令 (Command)`: 插件功能的入口。
* `2.2 基础 UI 元素`: `信息提示 (Notification)`, `输入框 (InputBox)`, `快速选择 (QuickPick)`。
* `2.3 状态栏 (Status Bar)`: 在底部状态栏显示信息。

### **第三章：深入工作区 - 编辑器与文件系统**
* `3.1 编辑器 API`: 读取和修改当前打开的文件内容。
* `3.2 位置与范围 (Position & Range)`: 精确定位代码。

### **第四章：构建复杂应用 - 视图、配置与状态**
* `4.1 侧边栏视图 (Sidebar View)`: 使用 `TreeDataProvider` 创建可交互的树状列表。
* `4.2 Webview`: 构建完全自定义的复杂 UI 界面。
* `4.3 插件与 Webview 通信`: 学习如何让你的插件主程序（TypeScript）与 Webview 中的网页（HTML/JS）互相发送和接收消息。