# SimpleRCPv2 架构与二次开发

这页回答两个问题：SimpleRCPv2 内部是怎么把“浏览器、终端、Agent、磁盘”这四方串起来的，以及想加东西时应该改哪里。

如果你还没跑起来，先看 [启动与运行](/simplerpc_v2/getting_started.md)。

## 一句话架构

**服务端工作区是唯一代码来源。** 浏览器里的 Yjs 文档、共享终端的 shell、Agent 的读写，最终都写入这个工作区；文件变化也会同步回 Yjs 文档。

这带来三个直接结果：

- 不需要像 [Collaboration Tools](/collaboration_tools/project_overview.md) 那样实现一套远程文件系统代理（Guest 侧没有 `oct://` 这类映射）。
- 终端和 Agent 不用改造，只要在项目目录里启动即可，可以复用它们原生的能力。
- 所有功能共享同一份代码，不存在“人一个副本、Agent 一个副本”的合并问题。

## 整体结构

```text
浏览器 (React + Monaco + xterm.js)
   │  HTTP /api        ── 项目、文件、聊天、Agent 接口
   │  WS   /ws         ── 在线状态、Activity、Agent 事件广播
   │  WS   /yjs/...    ── Yjs 文本文档同步
   │  WS   /terminal   ── 共享终端输入输出
   ▼
服务端 (Express + ws)
   ├─ ProjectRegistry      项目登记与导入
   ├─ MemberStore / permissions 轻量成员身份与统一权限入口
   ├─ ProjectRuntime       每个项目的运行时资源
   │    ├─ CollaborativeDocuments  Yjs ↔ 磁盘双向同步
   │    ├─ SharedTerminal          node-pty，cwd 为项目工作区
   │    ├─ RoomStore / EventLog / ChatStore
   │    └─ workspaceWatcher       监听外部文件变化
   └─ AgentRunManager     任务队列 + OpenCode Runtime
        └─ opencode serve (127.0.0.1)
             └─ DeepSeek Provider
   ▼
.simplercp-data/workspaces/<id>   ← 唯一代码来源
```

## 数据是怎么流的

### 1. 浏览器编辑 → 磁盘

Monaco 绑定 Yjs 文档，编辑通过 `/yjs` 通道同步。服务端在 `collaborativeDocuments.ts` 里监听 Yjs 更新，做 300ms 防抖后写回磁盘：

```ts
document.on("update", (_update, origin) => {
  if (origin !== FILESYSTEM_ORIGIN) {
    revisions.set(filePath, (revisions.get(filePath) ?? 0) + 1);
    schedulePersist(name, document);
  }
});
```

来源是 `FILESYSTEM_ORIGIN` 的更新不会再次写盘，避免“磁盘 → Yjs → 磁盘”的循环。

### 2. 磁盘变化 → 浏览器

终端脚本或 Agent 直接写文件时，`workspaceWatcher` 捕获变化，`reloadPath()` 把新内容以最小 delta 的形式合并进 Yjs，并标记为文件系统来源：

```ts
document.transact(() => {
  applyTextDelta(text, previousContent, result.content);
}, FILESYSTEM_ORIGIN);
```

这样可以把外部变化同步给所有成员，并在检测到并发修改时记录提示。

### 3. 共享终端

每个项目一个 `node-pty` 实例，工作目录就是 `workspaces/<id>`，所有成员的输入输出走同一个 `/terminal` 通道，并共享一段 scrollback。终端连接从 query 中读取 `memberId`，服务端校验成员记录后绑定连接，并给终端输入 Activity 记录填写成员归属。成员端只是“连到同一个 shell 的不同窗口”。

### 4. Agent

Agent 由服务端启动 OpenCode 子进程，只监听 `127.0.0.1`：

- `openCodeProcess.ts` 用 `opencode serve --hostname=127.0.0.1 --port=<port>` 启动，通过 `OPENCODE_CONFIG_CONTENT` 注入 DeepSeek Provider 配置。
- 权限上默认放开 `edit`、`bash`、`webfetch`，禁止 `external_directory`，把 Agent 限制在项目目录内。
- `agentRunManager.ts` 负责每个项目的任务队列、session 复用、run 前后的工作区快照对比，以及 trace 文件保存。
- OpenCode 进程使用经过过滤的环境变量；终端不会继承模型 Key。OpenCode 的 bash 工具可能读取到模型 Key，这是当前已知限制。
- 浏览器读不到 API Key，也不能直接访问 OpenCode 端口。

## 代码目录与入口

| 目录 | 职责 | 关键文件 |
| --- | --- | --- |
| `apps/server/src` | Express + WebSocket 服务 | `index.ts`、`createApp.ts`、`realtime.ts` |
| `apps/server/src/agent` | Agent 设置、OpenCode runtime、队列、trace | `agentRunManager.ts`、`openCodeRuntime.ts`、`openCodeProcess.ts` |
| `apps/client/src` | React 浏览器客户端 | `App.tsx`、`api.ts`、`socket.ts` |
| `apps/client/src/components` | 首页、工作区、面板等界面 | `WorkspaceExplorer.tsx`、`EditorArea.tsx`、`AgentPanel.tsx`、`SharedTerminal.tsx` |
| `apps/server/src/auth` | 成员记录、请求身份解析和统一权限入口 | `identity.ts`、`permissions.ts` |
| `packages/shared/src` | 前后端共享 TypeScript 类型 | `index.ts` |
| `demo/workspace` | 首次启动导入的 Demo 项目 | `src/projectStatus.js` |
| `tests` | 服务端测试与 Playwright E2E | `e2e/`、`fixtures/` |

## 二次开发：常见改动落在哪

- **加一个 HTTP 接口**：在 `createApp.ts` 里注册，项目相关逻辑通过 `runtimeManager.get(projectId)` 拿到运行时资源。
- **加一条实时消息**：先在 `packages/shared` 里补类型，再到 `apps/server/src/types.ts` 的 `ClientMessage` / `ServerMessage` 和 `realtime.ts` 的 `handleRealtimeMessage()` 里处理。
- **改文件持久化策略**：看 `collaborativeDocuments.ts` 的 `schedulePersist` / `persistDocument` / `reloadPath`。
- **加 Agent 能力或换模型供应商**：`agentRuntime.ts` 定义了运行时应实现的接口，`openCodeRuntime.ts` 是当前实现；任务编排在 `agentRunManager.ts`。
- **改界面**：从 `apps/client/src/App.tsx` 的工作区布局进入，具体面板都在 `components/` 下。
- **改成员身份**：`apps/server/src/auth/identity.ts` 负责 `members.json`、HTTP 请求头 `X-SimpleRCP-Member` 和成员恢复；`realtime.ts` 从 WebSocket query 的 `memberId` 解析身份。`permissions.ts` 保留 `can()` 入口，当前所有成员都允许执行已注册操作。

新增功能应复用“工作区即代码来源”这个约定，客户端和 Agent 共享 `workspaces/<id>`，数据记录写入项目元数据目录。

## 为什么说它更容易二次开发

对比 Collaboration Tools，SimpleRCPv2 少了几层通用抽象：

- 没有远程文件系统代理，文件读写就是服务端本地文件读写。
- 成员加入就是一次 HTTP 调用，服务端生成并保存 `memberId`，客户端按项目恢复这个身份。
- 前后端类型集中在一个 `packages/shared` 包，改协议时改动点集中。
- 终端和 Agent 都是在项目工作区里启动进程，接入新工具不需要改协作层。

对应地，它也保留了明确的能力边界：没有账号体系、没有命令隔离、没有三方合并、同一项目 Agent 串行执行。成员身份用于归属记录，不提供身份保护。

## 当前已知的限制

以下都是当前基线的已知问题，做二次开发前建议先读一遍：

- Agent 与成员可能互相覆盖同一文件的修改；系统只能提示 `concurrent_change`，不能找回被覆盖内容。
- 外部进程和成员同时写文件时，最终内容由实际完成顺序决定。
- 同一项目同一时间只执行一个 Agent run，其他任务排队。
- 服务中断会把 `running` / `queued` 的 run 标记为失败，不会自动重试。
- 当前不做鉴权，知道 `memberId` 就可以冒充成员；终端和 Agent 可以运行 shell 命令并访问服务端用户可以读取的文件，只适合可信内部环境。

完整的现象、影响和完整处理方向见项目内 `docs/product/known-issues.md`，改进方向见 `docs/product/improvement-roadmap.md`。

## 可以扩展的方向（建议，非已实现）

这部分是可以在现有架构上延伸的思路，目前源码还没有实现：

- 用独立工作目录 + base 版本做 Agent 修改的三方合并。
- 同一项目内多个 Agent 并行执行。
- 运行恢复：服务重启后续跑未完成的 run。
- 更完整的 trace 分析界面。

## 延伸阅读

- [SimpleRCPv2 项目概览](/simplerpc_v2/overview.md)
- [SimpleRCPv2 启动与运行](/simplerpc_v2/getting_started.md)
- [Collaboration Tools 技术与架构](/collaboration_tools/技术栈与架构.md)
- [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)
