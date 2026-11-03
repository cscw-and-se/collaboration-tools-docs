# 如何运行

这份文档提供一条可以直接运行的 DeepSeek 双 Agent 协作流程。用户只需要填写 API Key，并把启动命令中的项目路径改成自己的路径；服务端、Host、Guest、两个 Agent 和协作房间都会由启动器自动准备。

## 1. 最短启动流程

### 1.1 安装依赖并创建环境文件

在 `collaboration-tools` 项目根目录执行：

```bash
npm install
cp .env.example .env
chmod 600 .env
```

打开 `.env`，只填写下面这一项：

```dotenv
DEEPSEEK_API_KEY=填入你的 DeepSeek API Key
```

`.env.example` 中已经固定了默认的 `DEEPSEEK_BASE_URL=https://api.deepseek.com` 和 `DEEPSEEK_MODEL=deepseek-flash`，通常不需要修改。启动器会固定创建 `DeepSeek` Profile、`Host-agent` 和 `Guest-agent`，用户无需另外填写 Profile、Agent 名称或服务端地址。真实 API Key 只保存在 `.env`，不要写入 `.env.example`。

### 1.2 构建并启动协作会话

先构建项目：

```bash
npm run build
```

然后把命令中的路径改成需要共享的项目目录并执行：

```bash
npm run open -- --workspace "/你的项目绝对路径"
```

例如：

```bash
npm run open -- --workspace "/Users/baokker/Work/my-project"
```

启动器会自动完成以下工作：

1. 启动临时协作服务端。
2. 打开 Host 和 Guest 两个 VS Code 窗口。
3. 让 Host 创建房间，Guest 加入同一个房间。
4. 启动 `Host-agent` 和 `Guest-agent`，分别归属于 Host 和 Guest。
5. 使用 `.env` 中的 DeepSeek 配置初始化两个 Agent。

终端显示 `OCT manual collaboration is ready.` 后即可使用聊天面板。输入 `@Host-agent` 或 `@Guest-agent`，再填写任务并发送：

```text
@Host-agent 写一个两数之和函数，并保存到 src/two-sum.ts
@Guest-agent 检查这个函数的边界条件 #src/two-sum.ts
```

Agent 生成修改后，Host 会在待审阅列表中查看并应用提案。按启动命令所在终端的 `Ctrl+C` 可以结束整个会话。

### 1.3 选择其他路径或 Agent 数量

`--workspace` 可以使用绝对路径，也可以使用相对于当前终端目录的路径；路径包含空格或中文时需要使用引号。默认会启动两个 Agent。需要只启动一个 Agent 时，可以使用：

```bash
npm run open -- --workspace "/你的项目绝对路径" --agents host
```

## 2. 环境准备

### Node.js 版本

本项目要求 Node.js 版本不低于 `20.10.0`，推荐使用 `22.14.0`。

```bash
node --version
```

上一节的命令已经完成依赖安装和环境文件创建。如果需要分别启动服务端和两个客户端，用于调试或排查问题，见 [手动启动与加入机制](/collaboration_tools/manual_start_and_join.md)。

## 3. Agent 协作方式

### 3.1 Agent 的创建与任务流程

启动器默认在 Host 建立房间并等待 Guest 加入，随后创建两个 Agent：

- `Host-agent` 的属主是 Host。
- `Guest-agent` 的属主是 Guest。
- 两个 Agent 都作为独立 peer 加入同一个协作房间。
- 每个 Agent 使用独立工作副本和 OpenCode server。`packages/open-collaboration-agent/src/agent-session.ts` 负责初始化，`packages/open-collaboration-agent/src/agent-tasks.ts` 按发送者管理任务队列和 OpenCode session。首次收到任务时创建 session，后续同一发送者的任务复用它。

聊天消息通过 `packages/open-collaboration-agent/src/backend/run-backend-agent.ts` 分发。只有消息中包含对应 Agent 的 `@名称` 时才会进入它的任务队列，例如：

```text
@Host-agent 创建 src/two-sum.ts，编写两数之和函数
@Guest-agent 检查这个函数的边界条件 #src/two-sum.ts
```

Agent 收到任务后会执行以下流程：

1. 在 `globalStorageUri/agents/workspaces` 下刷新工作副本，并取得 Host 未保存的文档内容。
2. 解析文件引用；有有效引用时使用指定文件，没有引用时使用发送者当前文件。
3. 通过 OpenCode session 读取文件、搜索内容并生成修改。
4. 把修改提交为提案，并在聊天中通知 Host。
5. Host 在 `OCT: Review Pending Agent Proposals` 中查看差异，选择应用或拒绝。
6. 应用后的文件通过协作协议同步到 Guest。

Agent 使用的 Profile 由启动器根据根目录 `.env` 自动生成，配置入口位于 `scripts/manual-collaboration-agent-config.ts`。Profile 只写入 Provider 信息，API Key 通过 Host 的进程环境传给 Agent。用户需要自定义 Provider 时，可以设置 `OCT_AGENT_CONFIG_DIR`，指定共享项目之外的配置目录；目录包含 `profiles.json` 和 `.env`，配置读取代码位于 `packages/open-collaboration-agent/src/user-config.ts`。

手动控制时，可以在 Host 的协作菜单选择 `Start Agent`，指定属主和 Agent 名称。输入框支持 `@` 成员补全和 `#` 文件引用；聊天视图由 `packages/open-collaboration-vscode/src/chat-webview/chat-webview.ts` 提供。

创建或加入会话后，右侧 `Session Chat` 自动显示。聊天隐藏时，在命令面板执行 `Open Collaboration Tools: Open Chat`，或点击状态栏的对话图标。输入普通文本并发送即可，`@` 后可以从成员列表选择 Agent。

两个 Agent 的进程统一由 Host 启动，Guest 拥有自己的 Agent 身份和任务会话。成员可以提及任意 Agent；当前属主信息用于标识所属协作者。每个 Agent 的任务按 FIFO 执行，默认等待队列上限为 5，Host 统一审阅两个 Agent 的提案。文件在审阅前发生变化时，基准 hash 检查会返回 conflict，可以发送 `@AgentName /retry` 重新生成提案。聊天记录在退出会话后清除。

## 4. 常见问题：遇到 X 先看 Y

- **启动命令报错，或者窗口没有打开**：自动启动器目前面向 macOS，并要求 VS Code 安装在 `/Applications/Visual Studio Code.app`，Node.js 版本不低于 `20.10.0`。先逐项确认这三点。
- **Agent 没有响应**：先确认 `.env` 里的 `DEEPSEEK_API_KEY` 已经填写，并且消息里包含对应 Agent 的 `@名称`。
- **提案提示 conflict**：说明文件在审阅前被改动过，发送 `@AgentName /retry` 让 Agent 重新生成。
- **Guest 一直转圈，最后加入超时**：多半是 Host 没有点 Allow，详细原因见 [手动启动与加入机制](/collaboration_tools/manual_start_and_join.md)。
- **能看到文件但编辑不同步**：先看 [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)，确认文本同步链路。

## 5. 延伸阅读

- [项目概览](/collaboration_tools/project_overview.md)
- [手动启动与加入机制](/collaboration_tools/manual_start_and_join.md)
- [技术栈与架构](/collaboration_tools/技术栈与架构.md)
- [open-collaboration-vscode 模块](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/open-collaboration-vscode模块.md)
