# SimpleRCPv2 启动与运行

这页只做一件事：让你用最短路径把 SimpleRCPv2 跑起来，并知道每一步背后发生了什么。

如果你还没了解它是干什么的，先看 [项目概览](/simplerpc_v2/overview.md)。

## 适合谁读

- 第一次在本地跑 SimpleRCPv2 的人。
- 想确认端口、数据目录、Agent 配置怎么设置的人。

## 1. 环境要求

- Node.js 20 或更高版本
- pnpm 9

```bash
node --version
pnpm --version
```

## 2. 首次安装

```bash
git clone https://github.com/Baokker/SimpleRCPv2.git
cd SimpleRCPv2
pnpm install
```

## 3. 启动

在仓库根目录运行一条命令，会同时启动服务端和浏览器客户端：

```bash
pnpm dev
```

默认地址：

- 浏览器：`http://127.0.0.1:5173`
- 服务端：`http://127.0.0.1:4000`

启动命令**不需要填写 Workspace 参数**。首次启动会创建 `.simplercp-data/`，并把仓库里的 `demo/workspace/` 复制过去、登记为 Demo 项目，所以打开浏览器就能直接进 Demo。

已经装好依赖时，以后也只需要 `pnpm dev`。`pnpm dev:demo` 仍然可用，它和 `pnpm dev` 启动的是同一个应用和默认 Demo。

## 4. 首次启动发生了什么

理解这三步，后面排查问题会轻松很多：

1. 服务端读取仓库根目录的 `.env`（如果存在），加载配置。
2. 创建数据目录，默认是仓库根目录下的 `.simplercp-data/`，首次启动会导入 Demo。
3. 客户端启动后访问服务端 `/api` 接口；进入项目后再建立 `/ws`、`/yjs`、`/terminal` 三类 WebSocket 连接。

数据目录结构如下：

```text
.simplercp-data/
├── registry.json
├── projects
│   └── <projectId>
│       ├── project.json
│       ├── members.json
│       ├── chat.json
│       ├── activity.json
│       ├── agent-sessions
│       │   └── <sessionId>
│       │       └── session.json
│       └── agent-runs
│           └── <runId>
│               ├── run.json
│               └── trace.jsonl
├── workspaces
│   └── <projectId>
│       └── <project files>
├── instance
│   └── migration-<projectId>.*
└── agent
    └── settings.json
```

`projects/<projectId>/` 保存项目元数据、成员记录、聊天、Activity 和 Agent 运行记录。`workspaces/<projectId>/` 是浏览器、共享终端和 Agent 共同访问的代码目录。旧版本迁移过程中，`instance/` 下会保存 `migration-<projectId>.started`、`migration-<projectId>.copied.json` 和 `migration-<projectId>.complete` 等标记，用于支持中断后继续迁移。

`members.json` 保存服务端生成的 `memberId`、显示名、Role 和时间信息。客户端会按项目把 `memberId` 保存在 `localStorage`；刷新页面或重新打开浏览器时会尝试恢复原成员。HTTP 请求使用 `X-SimpleRCP-Member`，WebSocket 连接使用 query 中的 `memberId`。当前不做鉴权，知道 `memberId` 就可以冒充成员，因此只适合可信内部环境。

## 5. 体验一次多人协作

1. 打开 `http://127.0.0.1:5173`，从项目列表进入 Demo。
2. 填写显示名称和可选 Role，进入工作区。Role 会保存到成员记录并显示在成员列表中，当前不改变成员权限。
3. 再开一个浏览器窗口，用另一个显示名称进入同一个 Demo。
4. 两边同时修改 `src/projectStatus.js`，观察光标、聊天和终端输出。
5. 在共享终端里运行 Demo 自带命令：

```bash
npm start
npm test
```

Demo 只使用 Node.js 内置功能，不需要额外安装依赖。

## 6. 配置 Agent（OpenCode + DeepSeek）

Agent 使用 OpenCode `1.18.31` 和 `@opencode-ai/sdk` `1.18.31`，默认 Provider 为 DeepSeek。按下面步骤配置：

1. 运行 `pnpm install`，安装仓库指定版本的 OpenCode 和 TypeScript SDK。
2. 复制配置模板并在 `.env` 中填写 DeepSeek：

```bash
cp .env.example .env
```

```dotenv
DEEPSEEK_API_KEY=your_deepseek_api_key
DEEPSEEK_BASE_URL=https://api.deepseek.com/v1
DEEPSEEK_MODEL=deepseek-chat
```

3. 运行 `pnpm dev`，在全局 Agent 设置页检查 OpenCode 安装状态、选择 Model 并启用 Agent。
4. 进入项目，在 Agent 页签创建任务。新任务创建 OpenCode session，继续输入则复用同一个 session。

几个关键约束：

- OpenCode 由 SimpleRCPv2 服务端启动，只监听 `127.0.0.1`。
- 浏览器读不到 DeepSeek API Key，也不能直接访问 OpenCode 端口。
- 有运行中或排队任务时，服务端会拒绝修改 Model 或 Enabled，避免中断其他成员的任务。
- OpenCode 的 `external_directory` 权限设置为 `deny`，Agent 默认只能在项目工作区内操作。

## 7. 常见配置项

服务端启动时读取仓库根目录 `.env`，命令行环境变量优先级高于 `.env`。常用项：

| 变量 | 作用 | 默认值 |
| --- | --- | --- |
| `SIMPLERCP_DATA_DIR` | 项目数据目录，必须是绝对路径 | 仓库根目录 `.simplercp-data/` |
| `SIMPLERCP_WORKSPACES_DIR` | 工作区根目录，必须是绝对路径 | 数据目录下的 `workspaces/` |
| `SIMPLERCP_HOST` | 服务端监听地址 | `127.0.0.1` |
| `SIMPLERCP_PUBLIC_URL` | 用户访问的浏览器地址 | `http://127.0.0.1:5173` |
| `PORT` | 服务端端口 | `4000` |
| `VITE_SIMPLERCP_CLIENT_PORT` | 开发客户端端口 | `5173` |
| `DEEPSEEK_API_KEY` | DeepSeek API Key，Agent 任务需要 | 空 |
| `DEEPSEEK_MODEL` | 默认 Model | `deepseek-chat` |
| `SIMPLERCP_AGENT_RUN_TIMEOUT_MS` | 单个任务最长运行时间 | `600000` |
| `SIMPLERCP_IMPORT_ROOTS` | 允许导入目录的根路径，使用逗号分隔 | 未设置时允许导入服务端可读取的目录 |
| `SIMPLERCP_TERMINAL_HOME` | 终端进程使用的 HOME | 继承服务端环境 |
| `SIMPLERCP_TERMINAL_ENV_ALLOW` | 允许传给终端的额外环境变量名 | 仅保留基础环境变量 |
| `SIMPLERCP_AGENT_ENV_ALLOW` | 允许传给 OpenCode 的额外环境变量名 | 基础环境变量和 Agent 配置变量 |
| `SIMPLERCP_TERMINAL_ENABLED` | 是否启用共享终端 | `true` |

指定其他数据目录：

```bash
SIMPLERCP_DATA_DIR="/srv/simplercp-data" pnpm dev
```

设置 `SIMPLERCP_IMPORT_ROOTS` 后，导入目录会先解析 realpath，再检查是否位于允许的根目录中；未设置时，导入接口保持默认行为，允许导入服务端可以读取的目录。终端环境会过滤名称包含 `KEY`、`TOKEN`、`SECRET`、`PASSWORD` 或 `COOKIE` 的变量；`SIMPLERCP_TERMINAL_HOME` 可以为终端设置单独的 HOME。OpenCode 仍需要服务端注入模型配置，OpenCode 的 bash 工具可能读取到模型 Key，这是当前已知限制。

## 8. 内网访问

服务端和客户端监听地址都可以配置，内网反向代理可以由同一个域名提供页面和接口：

```bash
SIMPLERCP_HOST="0.0.0.0" \
SIMPLERCP_PUBLIC_URL="https://code.example.com" \
VITE_SIMPLERCP_CLIENT_HOST="0.0.0.0" \
VITE_SIMPLERCP_API_ORIGIN="http://127.0.0.1:4000" \
pnpm dev
```

反向代理需要转发普通 HTTP 路径 `/api`，并为 `/ws`、`/yjs`、`/terminal` 开启 WebSocket 转发。页面使用 HTTPS 时浏览器会自动使用 WSS。

当前版本允许项目成员运行 shell 命令并访问服务端用户可以读取的文件，成员身份只用于记录归属和恢复会话，不提供身份保护。知道 `memberId` 的人可以冒充成员，项目管理和全局设置也开放。请只在可信内部环境或 VPN 后使用，不要直接部署到公网；需要对外提供服务时，应在外部身份认证和网络限制之后部署。

## 9. 测试与构建

```bash
pnpm test       # 服务端测试、Demo 测试和启动测试
pnpm test:e2e   # 浏览器自动化测试（Playwright）
pnpm build      # TypeScript 检查并构建客户端与服务端
```

## 常见问题

- **启动后浏览器打不开**：先确认 `pnpm dev` 是否两个进程都起来了，再确认 `5173`（页面）和 `4000`（服务端）端口没有被占用。
- **需要手动指定项目目录吗**：不需要。项目目录由服务端在数据目录里管理，导入已有目录或 ZIP 时才会用到外部路径。
- **Agent 任务报错**：先看 Agent 设置页的 OpenCode 状态，再确认 `.env` 里的 `DEEPSEEK_API_KEY` 是否填写。Provider 报错信息中的 Key 会被替换成 `[REDACTED]`。
- **任务一直排队**：同一项目同一时间只执行一个 Agent run，这是当前有意的限制。

## 延伸阅读

- [SimpleRCPv2 项目概览](/simplerpc_v2/overview.md)
- [SimpleRCPv2 架构与二次开发](/simplerpc_v2/architecture_and_secondary_dev.md)
- [Collaboration Tools 如何运行](/collaboration_tools/how_it_works.md)
