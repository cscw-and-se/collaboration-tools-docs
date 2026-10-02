# SimpleRCPv2 启动与运行

这页带你在本地把 SimpleRCPv2 跑起来，体验一次多人协作，再把 Agent 配好。

还不了解它是干什么的，先看 [项目概览](/simplercp_v2/overview.md)。

## 适合谁读

- 第一次在本地跑 SimpleRCPv2 的人。
- 想知道端口、数据目录、Agent 怎么配置的人。

## 1. 环境要求

- Node.js 20 或更高版本
- pnpm 9

```bash
node --version
pnpm --version
```

## 2. 安装

```bash
git clone https://github.com/Baokker/SimpleRCPv2.git
cd SimpleRCPv2
pnpm install
```

`pnpm install` 会顺带装好仓库指定版本的 OpenCode，不需要另外安装。

## 3. 启动

在仓库根目录运行：

```bash
pnpm dev
```

这条命令同时启动服务端和浏览器客户端：

- 浏览器：`http://127.0.0.1:5173`
- 服务端：`http://127.0.0.1:4000`

不需要指定项目目录。首次启动时，服务端会在仓库根目录创建 `.simplercp-data/`，把 `demo/workspace/` 复制进去并登记为 Demo 项目，打开浏览器就能直接进 Demo。`pnpm dev:demo` 也能用，启动的是同一个应用。

所有数据都在 `.simplercp-data/` 里。想从头再来，停掉服务后删掉这个目录即可。目录里各个文件是做什么的，见 [架构与二次开发](/simplercp_v2/architecture_and_secondary_dev.md#数据目录)。

## 4. 体验一次多人协作

1. 打开 `http://127.0.0.1:5173`，从项目列表进入 Demo。
2. 填写显示名，Role 可以不填，然后进入工作区。Role 只作为标签显示在成员列表里，不影响能做什么。
3. 换一个浏览器，或者开一个无痕窗口，用另一个名字进入同一个 Demo。
4. 两边同时改 `src/projectStatus.js`，观察对方的光标、聊天和终端输出。
5. 在共享终端里运行 Demo 自带的命令：

```bash
npm start
npm test
```

Demo 只用 Node.js 内置功能，不需要再装依赖。

第 3 步要换浏览器或用无痕窗口，是因为成员身份保存在浏览器存储里。同一个浏览器的普通窗口共用存储，会被当成同一个人。

## 5. 配置 Agent

Agent 用的是 OpenCode `1.18.31`，默认模型供应商是 DeepSeek。

1. 复制配置模板：

```bash
cp .env.example .env
```

2. 在 `.env` 里填写 DeepSeek 的配置：

```dotenv
DEEPSEEK_API_KEY=your_deepseek_api_key
DEEPSEEK_BASE_URL=https://api.deepseek.com/v1
DEEPSEEK_MODEL=deepseek-chat
```

3. 重新运行 `pnpm dev`，在全局 Agent 设置页确认 OpenCode 已安装，选择模型并启用 Agent。
4. 进入项目，在 Agent 页签创建任务。新任务会创建一个 OpenCode session，之后在同一个 session 里继续输入会接着上次的上下文。

使用时要知道的几点：

- OpenCode 由服务端启动，只监听 `127.0.0.1`，浏览器访问不到它的端口，也拿不到 API Key。
- 有任务在运行或排队时，不能修改模型和启用状态，免得打断其他人的任务。
- Agent 只能在项目工作区里操作，访问工作区外的目录会被 OpenCode 拒绝。

## 6. 常用配置项

服务端启动时读取仓库根目录的 `.env`，命令行里设置的环境变量优先。

| 变量 | 作用 | 默认值 |
| --- | --- | --- |
| `SIMPLERCP_DATA_DIR` | 数据目录，必须是绝对路径 | 仓库根目录的 `.simplercp-data/` |
| `SIMPLERCP_WORKSPACES_DIR` | 工作区根目录，必须是绝对路径 | 数据目录下的 `workspaces/` |
| `SIMPLERCP_HOST` | 服务端监听地址 | `127.0.0.1` |
| `SIMPLERCP_PUBLIC_URL` | 用户在浏览器里访问的地址 | `http://127.0.0.1:5173` |
| `PORT` | 服务端端口 | `4000` |
| `VITE_SIMPLERCP_CLIENT_PORT` | 开发客户端端口 | `5173` |
| `DEEPSEEK_API_KEY` | DeepSeek API Key，用 Agent 时必填 | 空 |
| `DEEPSEEK_MODEL` | 默认模型 | `deepseek-chat` |
| `SIMPLERCP_AGENT_RUN_TIMEOUT_MS` | 单个 Agent 任务的最长运行时间 | `600000`（10 分钟） |
| `SIMPLERCP_TERMINAL_ENABLED` | 是否启用共享终端 | `true` |
| `SIMPLERCP_IMPORT_ROOTS` | 只允许从这些目录导入项目，逗号分隔 | 不限制 |
| `SIMPLERCP_TERMINAL_HOME` | 终端使用的 HOME | 继承服务端 |
| `SIMPLERCP_TERMINAL_ENV_ALLOW` | 额外允许传给终端的环境变量名，逗号分隔 | 只传基础变量 |
| `SIMPLERCP_AGENT_ENV_ALLOW` | 额外允许传给 OpenCode 的环境变量名，逗号分隔 | 基础变量和 DeepSeek 配置 |

例如把数据放到其他目录：

```bash
SIMPLERCP_DATA_DIR="/srv/simplercp-data" pnpm dev
```

终端和 OpenCode 的环境变量是按白名单传的。名字里带 `KEY`、`TOKEN`、`SECRET`、`PASSWORD`、`COOKIE` 的变量不会进入终端，所以在终端里 `env` 看不到 API Key。

## 7. 在内网里给别人用

把监听地址改成 `0.0.0.0`，就能让同一网络里的其他人访问：

```bash
SIMPLERCP_HOST="0.0.0.0" \
SIMPLERCP_PUBLIC_URL="https://code.example.com" \
VITE_SIMPLERCP_CLIENT_HOST="0.0.0.0" \
VITE_SIMPLERCP_API_ORIGIN="http://127.0.0.1:4000" \
pnpm dev
```

如果前面有反向代理，需要转发 `/api`，并为 `/ws`、`/yjs`、`/terminal` 开启 WebSocket 转发。页面走 HTTPS 时，浏览器会自动改用 WSS。

开放给别人之前请注意，SimpleRCPv2 不做鉴权。项目成员可以在终端里执行任意命令，读到服务端用户能读的文件，项目管理和全局设置对所有人开放。只在可信的内部网络或 VPN 后面使用，不要直接放到公网上。

## 8. 测试与构建

```bash
pnpm test       # 服务端测试、Demo 测试和启动测试
pnpm test:e2e   # 浏览器自动化测试（Playwright）
pnpm build      # TypeScript 检查并构建客户端与服务端
```

## 常见问题

- **浏览器打不开页面**：确认 `pnpm dev` 的两个进程都起来了，再看 `5173` 和 `4000` 端口有没有被占用。
- **两个窗口里的人变成了同一个**：同一浏览器的普通窗口共用成员身份，换一个浏览器或用无痕窗口。
- **Agent 任务报错**：先看 Agent 设置页里 OpenCode 的状态，再确认 `.env` 里填了 `DEEPSEEK_API_KEY`。报错信息里的 Key 会显示成 `[REDACTED]`。
- **任务一直在排队**：同一项目同一时间只执行一个 Agent 任务，前一个结束后才会轮到下一个。

## 延伸阅读

- [SimpleRCPv2 项目概览](/simplercp_v2/overview.md)
- [SimpleRCPv2 架构与二次开发](/simplercp_v2/architecture_and_secondary_dev.md)
- [Collaboration Tools 如何运行](/collaboration_tools/how_it_works.md)
