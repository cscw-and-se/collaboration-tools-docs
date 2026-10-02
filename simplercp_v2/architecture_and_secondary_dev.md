# SimpleRCPv2 架构与二次开发

这页讲 SimpleRCPv2 内部怎么把浏览器、共享终端、Agent 和磁盘串起来，以及想加功能时该改哪里。

还没跑起来的话，先看 [启动与运行](/simplercp_v2/getting_started.md)。

## 核心约定：工作区是唯一的代码来源

每个项目在服务端有一个工作区目录 `workspaces/<projectId>/`。浏览器里的编辑、共享终端里的命令、Agent 的读写，最后都落到这个目录；有人在终端或 Agent 里直接改了文件，变化也会同步回浏览器。

这样做有三个好处：

- 不需要像 [Collaboration Tools](/collaboration_tools/project_overview.md) 那样实现远程文件系统代理，文件读写就是服务端的本地读写。
- 终端和 Agent 不用改造，在工作区目录里启动进程就能用上它们原本的能力。
- 人和 Agent 改的是同一份代码，不存在各有一个副本、事后再合并的问题。

## 整体结构

```text
浏览器 (React + Monaco + xterm.js)
   │  HTTP /api        项目、文件、成员、聊天、Agent 接口
   │  WS   /ws         在线状态、Activity、Agent 事件广播
   │  WS   /yjs/...    Yjs 文本文档同步
   │  WS   /terminal   共享终端输入输出
   ▼
服务端 (Express + ws)
   ├─ routes/*.ts            HTTP 接口，按领域分文件
   ├─ ProjectRegistry        项目登记与导入
   ├─ auth/                  成员记录、身份解析、权限入口
   ├─ ProjectRuntime         每个项目的运行时资源
   │    ├─ CollaborativeDocuments   Yjs 与磁盘双向同步
   │    ├─ SharedTerminal           node-pty，cwd 为项目工作区
   │    ├─ RoomStore / EventLog / ChatStore
   │    └─ workspaceWatcher         监听外部文件变化
   └─ AgentRunManager        每个项目一个任务队列
        └─ opencode serve (127.0.0.1)
             └─ DeepSeek Provider
   ▼
.simplercp-data/workspaces/<projectId>/
```

## 数据怎么流动

### 浏览器编辑写回磁盘

Monaco 绑定 Yjs 文档，编辑通过 `/yjs` 通道同步给其他人。服务端在 `collaborativeDocuments.ts` 里监听 Yjs 更新，防抖 300ms 后写回磁盘：

```ts
document.on("update", (_update, origin) => {
  if (origin !== FILESYSTEM_ORIGIN) {
    revisions.set(filePath, (revisions.get(filePath) ?? 0) + 1);
    schedulePersist(name, document);
  }
});
```

来源标记为 `FILESYSTEM_ORIGIN` 的更新不会再写盘，避免磁盘和 Yjs 之间来回触发。

### 磁盘变化同步到浏览器

终端命令或 Agent 直接改文件时，`workspaceWatcher` 发现变化，`reloadPath()` 把新内容按最小差异合并进 Yjs，并打上文件系统来源的标记：

```ts
document.transact(() => {
  applyTextDelta(text, previousContent, result.content);
}, FILESYSTEM_ORIGIN);
```

外部写入和浏览器里的编辑之间不做合并。Agent 任务结束时，`agentRunManager.ts` 会检查任务期间有没有别人改过同一批文件，有的话记一条 `concurrent_change` 提示。

### 共享终端

每个项目一个 `node-pty` 进程，工作目录是项目工作区。所有成员连到同一个 `/terminal` 通道，看到同一段输出和 scrollback，相当于同一个 shell 开了几个窗口。连接时服务端从 query 里取 `memberId`，校验成员记录后绑定到连接上，终端输入记到这个成员名下。

终端进程的环境变量按白名单传入，名字里带 `KEY`、`TOKEN`、`SECRET`、`PASSWORD`、`COOKIE` 的变量会被过滤掉，相关逻辑在 `processEnv.ts`。

### Agent

Agent 是服务端启动的 OpenCode 子进程：

- `openCodeProcess.ts` 用 `opencode serve --hostname=127.0.0.1 --port=<port>` 启动，通过 `OPENCODE_CONFIG_CONTENT` 注入 DeepSeek 配置。权限上允许 `edit`、`bash`、`webfetch`，禁止 `external_directory`，让 Agent 只在工作区里活动。
- `agentRunManager.ts` 管理每个项目的任务队列和 session 复用，任务前后各拍一次工作区快照用来对比改了哪些文件，并把 trace 存成文件。
- `openCodeRuntime.ts` 负责和 OpenCode 通信，实现 `agentRuntime.ts` 里定义的接口。

OpenCode 进程也走环境变量白名单，但它必须拿到模型配置，所以 Agent 的 bash 工具有可能读到 DeepSeek Key。终端则读不到。这个限制需要在可信内部环境中使用，并在后续运行审批中处理。

## 数据目录

默认在仓库根目录的 `.simplercp-data/`：

```text
.simplercp-data/
├── registry.json                项目列表
├── projects/<projectId>/        项目元数据，不放代码
│   ├── project.json
│   ├── members.json             成员记录
│   ├── chat.json
│   ├── activity.json
│   ├── agent-sessions/<sessionId>/session.json
│   └── agent-runs/<runId>/
│       ├── run.json
│       └── trace.jsonl
├── workspaces/<projectId>/      项目代码，浏览器、终端、Agent 共用
├── instance/                    旧版本数据迁移的标记文件
└── agent/settings.json          全局 Agent 设置
```

元数据和代码分开放，是为了让终端和 Agent 在工作区里操作时碰不到聊天记录、成员记录和 trace。

`instance/` 只在从旧版本升级时用到。旧版本把元数据放在工作区里，首次启动新版本会把数据迁移出来，迁移过程中写 `migration-<projectId>.started`、`.copied.json`、`.complete` 三个标记，中途断掉下次启动可以接着迁移。

## 成员身份

加入项目是一次 HTTP 调用。服务端生成 `memberId`，和显示名、Role 一起存进 `members.json`；客户端按项目把 `memberId` 存在当前标签页的 `sessionStorage` 里，刷新页面后带着它恢复成同一个成员；新开的标签页没有这份记录，会重新显示加入页。`localStorage` 只保存最近一次的选择，用来在加入页里预选。之后 HTTP 请求通过 `X-SimpleRCP-Member` 请求头带上 `memberId`，WebSocket 通过 query 带上。

聊天、Agent 任务和 session 的归属由服务端根据这个身份填写，客户端在消息里自己写的 `memberId` 会被忽略。`permissions.ts` 里保留了 `can()` 作为统一的权限入口，目前对所有成员都返回允许，以后要加权限控制时从这里改。

这套身份只用来记录谁做了什么，不做鉴权，知道别人的 `memberId` 就能冒充他。

## 代码入口

| 目录 | 职责 | 先看哪些文件 |
| --- | --- | --- |
| `apps/server/src` | 服务端入口、实时通信 | [`index.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/index.ts)、[`createApp.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/createApp.ts)、[`realtime.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/realtime.ts)、[`collaborativeDocuments.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/collaborativeDocuments.ts) |
| `apps/server/src/routes` | HTTP 接口 | `projectRoutes.ts`、`workspaceRoutes.ts`、`collaborationRoutes.ts`、`agentRoutes.ts` |
| `apps/server/src/auth` | 成员记录、身份解析、权限入口 | `identity.ts`、`permissions.ts` |
| `apps/server/src/agent` | Agent 设置、OpenCode 运行时、任务队列、trace | [`agentRunManager.ts`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/server/src/agent/agentRunManager.ts)、`openCodeRuntime.ts`、`openCodeProcess.ts` |
| `apps/client/src` | React 客户端 | [`App.tsx`](https://github.com/Baokker/SimpleRCPv2/blob/main/apps/client/src/App.tsx)、`api.ts`、`socket.ts` |
| `apps/client/src/components` | 首页、工作区和各个面板 | `WorkspaceExplorer.tsx`、`EditorArea.tsx`、`AgentPanel.tsx`、`SharedTerminal.tsx` |
| `packages/shared/src` | 前后端共用的 TypeScript 类型 | `index.ts` |
| `demo/workspace` | 首次启动导入的 Demo 项目 | `src/projectStatus.js` |
| `tests` | 服务端测试和 Playwright E2E | `e2e/`、`fixtures/` |

表中链接指向 `main` 分支。

## 想加功能时改哪里

| 想做的事 | 从哪里改 |
| --- | --- |
| 加一个 HTTP 接口 | 在 `apps/server/src/routes/` 里对应领域的文件中添加，新领域就新建一个文件，再在 `createApp.ts` 里注册。项目相关的资源通过 `runtimeManager.get(projectId)` 获取 |
| 加一种实时消息 | 先在 `packages/shared` 补类型，再改 `apps/server/src/types.ts` 的 `ClientMessage`、`ServerMessage`，最后在 `realtime.ts` 的 `handleRealtimeMessage()` 里处理 |
| 改文件写盘策略 | `collaborativeDocuments.ts` 里的 `schedulePersist`、`persistDocument`、`reloadPath` |
| 换模型供应商或换 Agent | 按 `agentRuntime.ts` 的接口写一个新实现，参考 `openCodeRuntime.ts`；任务编排在 `agentRunManager.ts` |
| 改界面 | 从 `apps/client/src/App.tsx` 的工作区布局进入，各面板在 `components/` 下 |
| 加权限控制 | `auth/permissions.ts` 的 `can()`，调用点已经埋在各个接口里 |

新功能请沿用工作区即代码来源的约定，代码放 `workspaces/<projectId>/`，其他记录放 `projects/<projectId>/`。

## 已知限制

做二次开发前建议先了解这些：

- **不做鉴权，也没有命令和文件隔离。** 知道 `memberId` 就能冒充成员；终端和 Agent 能访问服务端用户能读到的所有文件。只适合可信的内部环境。
- **并发修改会互相覆盖。** 人和 Agent 同时改一个文件时，最终内容取决于谁后写完，系统只给出 `concurrent_change` 提示，被覆盖的内容找不回来。
- **同一项目的 Agent 任务排队执行。** 不同项目之间可以同时跑。
- **服务重启后任务不会续跑。** 处于 `running` 或 `queued` 的任务会被标为失败。
- **复制出来的标签页会沿用原来的身份。** 浏览器的复制标签页功能，以及从页面里点开的新标签页，会带上原标签页的 `sessionStorage`，因此还是同一个成员。要以另一个人加入，请新开标签页后手动输入地址。

每条限制的现象和处理方向，见 SimpleRCPv2 仓库里的 `docs/product/known-issues.md` 和 `docs/product/improvement-roadmap.md`。

## 可以扩展的方向

以下只是基于现有架构的思路，源码里还没有实现：

- 给 Agent 一个独立工作目录，记录开始时的 base 版本，结束后做三方合并，解决并发覆盖。
- 同一项目里多个 Agent 并行执行。
- 服务重启后恢复未完成的任务。
- 更完整的 trace 查看和分析界面。

## 延伸阅读

- [SimpleRCPv2 项目概览](/simplercp_v2/overview.md)
- [SimpleRCPv2 启动与运行](/simplercp_v2/getting_started.md)
- [Collaboration Tools 技术与架构](/collaboration_tools/技术栈与架构.md)
- [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)
