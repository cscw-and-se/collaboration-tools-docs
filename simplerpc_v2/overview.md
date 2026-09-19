# SimpleRCPv2 项目概览

SimpleRCPv2 是一个以**服务端项目目录为唯一代码来源**的实时协同编程系统。你和同学在浏览器里打开同一个项目，就能一起看文件、改代码、聊天、用同一个终端，还能把 Agent 任务交给它去跑。

它的目标不是把 [Collaboration Tools](/collaboration_tools/project_overview.md) 做全，而是把实时协作的链路压到最短：一条命令启动，浏览器即用，代码只存服务端一份，人和 Agent 操作的是同一份代码。

这篇文档先讲清楚它是干什么的、为什么做、和 Collaboration Tools 有什么区别；具体怎么跑见 [启动与运行](/simplerpc_v2/getting_started.md)，代码结构见 [架构与二次开发](/simplerpc_v2/architecture_and_secondary_dev.md)。

源代码仓库：[github.com/Baokker/SimpleRCPv2](https://github.com/Baokker/SimpleRCPv2)

## 适合谁读

- 想快速体验一次实时协同编程、不想装编辑器插件的同学。
- 已经读过 [Collaboration Tools 项目概览](/collaboration_tools/project_overview.md)，想看新方案差异的人。
- 准备在 SimpleRCPv2 上做二次开发、加功能的人。

## 它长什么样

进入项目后是一个浏览器工作区：左侧是文件树，中间是 Monaco 编辑器，右侧是协作/Agent 面板，底部是共享终端和状态栏。多人编辑时能看到彼此的光标和选区，Agent 的文件变化、输出和 trace 也显示在同一个页面里。

![SimpleRCPv2 协作工作区](assets/workspace-overview.png)

> 截图来自项目 README 中的工作区示例。

## 背景：为什么又做了一个

Collaboration Tools 的路线是“VS Code 插件 + Host/Guest + 审批 + 完整认证”：能力齐全，但要把 server、Host、Guest 三端都拉起来才能看到效果，链路比较长，理解成本也高。

在 Vibe Coding 的场景里，实际需求往往更直接：

- 坐下来就能开始，不想为一次协作装插件、开好几个窗口。
- 人写代码和 Agent 写代码发生在同一个地方，能互相看到。
- 项目代码只存一份，人和 Agent 不要各自维护一个副本。

所以 SimpleRCPv2 做了三个关键取舍：**浏览器客户端**、**服务端工作目录即唯一数据源**、**内置 Agent**。功能刻意保持最小，不追求完整鉴权、审批和冲突兜底，方便快速迭代和二次开发。

## 核心特点

1. **更简洁，更适合 Vibe Coding**：浏览器打开就能用，编辑、终端、聊天、Agent 在同一个页面，不需要在编辑器和浏览器之间来回切。
2. **支持多人、多 Agent**：每位成员有自己的显示名和 Agent session，可以创建任务、继续自己的 session、查看队列和 trace。同一项目内的任务依次执行，不同项目可以同时执行。
3. **启动更方便**：`pnpm dev` 一条命令同时拉起服务端和客户端，不需要传 Workspace 参数，首次启动会自动准备 Demo 项目。
4. **更适合二次开发**：服务端项目目录是唯一数据源，浏览器、共享终端和 Agent 都直接读写它，不用再实现一套远程文件系统代理；启动终端、启动 Agent 都只是在这个目录里起进程，可以复用它们原生的能力。
5. **功能刻意简约**：没有做很多健全性（防护性）操作，冲突、鉴权、隔离都保持在基线最小范围。这既是当前的能力边界，也是二次开发上手快的原因之一。

## 当前功能

- **项目首页**：打开/删除项目、创建空白项目、导入服务端已有目录、导入 ZIP。
- **默认 Demo**：首次启动自动复制 `demo/workspace` 并登记为 Demo 项目。
- **代码保存**：所有项目位于 `.simplercp-data/projects/`，也可用环境变量指向其他绝对路径。
- **文件管理**：按需读取文件树，支持创建、重命名、删除文件和目录。
- **代码编辑**：Monaco Editor + Yjs，同步多人文本、光标和选区，并显示保存与同步状态。
- **实时协作**：成员列表、当前文件、持久化聊天和 Activity；编辑记录带变化行号与增删行数。
- **共享终端**：`node-pty` 在项目目录中运行 shell，所有成员看到同一个终端。
- **外部变化同步**：监听终端或 Agent 产生的文件变化，并更新文件树和已打开的协作文档。
- **错误恢复**：协作连接与终端连接自动重连，离线后可手动重试；项目被删除后返回首页。
- **Agent 设置**：查看 OpenCode 状态和版本、设置 DeepSeek Model、启用或停用 Agent。
- **Agent 任务**：创建任务、继续 session、取消自己的任务，查看队列位置、模型输出和文件变化。
- **Agent trace**：OpenCode SSE 事件、状态、文件变化和并发修改提示按 JSONL 保存，可在线查看和下载。

## 与 Collaboration Tools 的对比

下面这张表按“新人实际会接触到的差异”来整理，更细的实现差异见各自模块文档。

| 维度 | Collaboration Tools | SimpleRCPv2 |
| --- | --- | --- |
| 客户端 | 以 VS Code 插件为主（Extension Development Host），另有 Monaco 等模块 | 浏览器（React + Monaco + xterm.js） |
| 启动方式 | 安装、构建 server，再用 F5 或脚本启动 Host/Guest 两个 VS Code | `pnpm install` + `pnpm dev`，打开 `http://127.0.0.1:5173` |
| 代码来源 | Host 的本地工作区，Guest 通过远程文件系统代理访问 | 服务端 `.simplercp-data/projects/<id>/workspace`，浏览器/终端/Agent 共用这一份 |
| 加入方式 | 邀请码，Host 弹窗 Allow / Deny 后才进入 | 打开项目、填显示名即可进入；角色只影响界面显示 |
| 认证 | JWT、简易登录、OAuth、Keycloak | 无内置账号体系，面向可信成员 |
| 共享终端 | 无内置共享终端 | 内置共享终端（`node-pty`），多人同一个 shell |
| Agent | 无内置 Agent，依赖使用者自己的工具 | 内置 OpenCode + DeepSeek，含任务、session、trace |
| 数据同步 | Yjs 文本 + awareness（光标/在线状态） | Yjs 文本 + 光标选区 + 事件 Activity，服务端负责持久化到磁盘 |
| 部署形态 | server + 分发的 VS Code 扩展 | 单个服务进程（HTTP + WebSocket） |

一句话概括：Collaboration Tools 更像“把 VS Code 协作起来”的通用方案，SimpleRCPv2 更像“为人和 Agent 一起写代码准备的一个轻量工作台”。

## 代码索引

想直接读源码，可以从这些入口开始：

- [`README.md`](https://github.com/Baokker/SimpleRCPv2/blob/main/README.md)：功能、启动、配置和数据目录说明。
- [`apps/server/src/index.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/index.ts)：服务端启动入口。
- [`apps/server/src/createApp.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/createApp.ts)：HTTP API 注册，包含项目、文件、聊天和 Agent 接口。
- [`apps/server/src/realtime.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/realtime.ts)：`/ws`、`/yjs`、`/terminal` 三个实时通道。
- [`apps/server/src/collaborativeDocuments.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/collaborativeDocuments.ts)：Yjs 文档与磁盘文件的双向同步。
- [`apps/server/src/agent/agentRunManager.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/agent/agentRunManager.ts)：Agent 任务队列与执行流程。
- [`apps/client/src/App.tsx`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/client/src/App.tsx)：浏览器工作区主界面。

## 延伸阅读

- [SimpleRCPv2 启动与运行](/simplerpc_v2/getting_started.md)
- [SimpleRCPv2 架构与二次开发](/simplerpc_v2/architecture_and_secondary_dev.md)
- [Collaboration Tools 项目概览](/collaboration_tools/project_overview.md)
