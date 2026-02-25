# 软件开发前置知识速通（本科生/研究生）

很多同学第一次进组做“真实项目”，卡住的往往不是某个语法点，而是：

- 代码能跑起来，但你不知道它为什么这样跑。
- 你能改出一个功能，但不知道怎么把它改得“可维护、可评审、可排障”。
- 你在 VS Code 里点了一个按钮，背后发生了很多事，但你只能凭感觉猜。

这份速通不是为了让你学得“很全”，而是为了让你尽快形成一种工程直觉：遇到问题时知道先看哪里、改动时知道哪些地方影响面最大。

下面我用“你刚开始参与 `collaboration-tools` 项目”这个场景，把最需要补齐的前置知识按实际开发顺序串起来。

## 1) 先把环境跑稳：命令行、Node、包管理、日志

你至少要做到两件事：

- 看到一条命令知道它在做什么（install/build/dev/test）。
- 知道日志从哪里来，怎么加日志，怎么把日志对齐到某一段源码。

建议优先补：

- Shell 与常用命令：`cd`/`ls`/`cat`/`rg`/`find`/重定向/环境变量（见 [Shell](/tech_stack/shell.md)）。
- Node 版本管理（nvm）与 workspace 依赖安装。

在本项目里，对应的第一条实践路径是：先按 [项目是怎么运行的](/collaboration_tools/how_it_works.md) 跑通一次“server + 两个 VS Code”。你能跑通，后面的学习都会变快。

## 2) Git 团队协作：分支、冲突、评审不是可选项

真实开发里，你一定会遇到：

- 同一个文件多人改，冲突怎么解。
- PR review 里别人说“这里不该这样改”，你怎么解释、怎么改回去。
- 你做了一个临时实验，怎么不污染主线。

你不需要一开始就精通，但至少要能独立完成：

- 拉分支、提交、rebase、解决冲突。
- 写出能被别人快速理解的 commit message。

对应资料： [Git 基础](/git/git_basic.md)、[Git 团队协作流程](/git/git_team_workflow.md)。

## 3) TypeScript 与 Node：看得懂异步与类型，才能看得懂协作链路

协作系统的代码天然会大量出现：

- `async/await` + Promise（网络请求、事件回调、超时处理）。
- 类型定义（协议消息、room claim、peer/user 等结构）。

你至少要能做到：

- 看到一个 `async` 函数知道它可能在哪些地方被 await。
- 看到一个类型定义能追到它在 runtime 里被怎么使用。

对应资料： [TypeScript](/tech_stack/typescript.md)。

## 4) VS Code 扩展基础：activation、commands、FileSystemProvider 是三大件

你们团队最关心的是 VS Code 端实时协作，所以你需要把 VS Code 扩展的三个关键机制补起来：

- 扩展什么时候启动（activation）？启动后入口在哪（`activate()`）？
- 用户点 UI 入口触发了什么（commands + quickpick）？
- `oct://` 为什么能当成文件系统（FileSystemProvider）？

对应资料：`vscode_plugin/` 目录下的系列文章（从 [大纲](/vscode_plugin/outline.md) 进入）。

在本项目里，更建议你直接顺着项目文档走：

- [项目概览](/collaboration_tools/project_overview.md)
- [open-collaboration-vscode 模块](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/open-collaboration-vscode模块.md)

## 5) 实时协作基础：分清“文本同步”和“协作态同步”

协作里最常见的误解是把所有同步问题都当成“文本同步坏了”。实际上它至少有两条线：

- 文本内容同步：Yjs update。
- 协作态同步：awareness（光标/选区/可见范围）。

对应资料：

- [Y.js](/tech_stack/yjs.md)
- [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)
- [open-collaboration-yjs 模块](/collaboration_tools/核心模块详解/open-collaboration-yjs模块.md)

## 6) 网络与服务端基础：WebSocket/Socket.IO、JWT、超时轮询

你不需要一开始就能写 server，但你要能看懂：

- “连接失败”是发生在 HTTP 还是 socket。
- “等待 Host 审批”本质上是 request + poll + timeout 的状态机。
- JWT 在这个系统里有不同层级（loginToken vs roomToken）。

对应资料：

- [open-collaboration-server 模块](/collaboration_tools/核心模块详解/open-collaboration-server模块/open-collaboration-server模块.md)
- [房间生命周期管理](/collaboration_tools/核心模块详解/open-collaboration-server模块/房间与用户管理机制/房间生命周期管理.md)
- [认证机制](/collaboration_tools/安全与认证/认证机制.md)

## 7) 调试与验证：最小复现、日志对齐、改动后回归

最后这一条是所有“能独立做事”的分水岭：你遇到 bug 时不要靠玄学重装重启，而是能做到：

- 最小复现：把问题缩到最短链路。
- 日志对齐：知道要在哪几个入口打日志（client/server/protocol/yjs）。
- 改动回归：改完能跑一遍 `host + guest` 的完整链路，确认没有引入新问题。

在本项目里，一个很有效的排障顺序是：

- 先跑通 [项目是怎么运行的](/collaboration_tools/how_it_works.md) 的 checklist。
- 再用 [项目概览](/collaboration_tools/project_overview.md) 里的入口定位到具体文件。
- 同步问题就看 [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md) 把现象归类。

## Summary

- 最重要的是先跑通一次端到端： [项目是怎么运行的](/collaboration_tools/how_it_works.md)。
- 学习顺序建议按“能做事 -> 能排障 -> 能二开”：环境与工具 -> Git/TS -> VS Code 扩展 -> 协作同步 -> server。
- 你如果只读三篇项目文档就上手：
  - [项目概览](/collaboration_tools/project_overview.md)
  - [open-collaboration-vscode 模块](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/open-collaboration-vscode模块.md)
  - [open-collaboration-server 模块](/collaboration_tools/核心模块详解/open-collaboration-server模块/open-collaboration-server模块.md)

