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

上一节的命令已经完成依赖安装和环境文件创建。下面的章节补充运行环境要求和手动启动方式，便于排查启动问题。

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

## 4. 启动后端服务（open-collaboration-server）

在 `collaboration-tools` 项目根目录执行：

```bash
OCT_ACTIVATE_SIMPLE_LOGIN=true npm run start
```

默认监听 `http://localhost:8100`。

如果你后面遇到“怎么都连不上”，先别怀疑人生，第一步就是确认这个端口确实在监听。

## 5. 启动 VS Code 扩展（open-collaboration-vscode）

协作至少需要两个客户端：一个 Host（共享者），一个 Guest（加入者）。

### 5.1 先确认 serverUrl 配置

扩展会读 VS Code settings 里的 `oct.serverUrl`（默认值在扩展的 `package.json` 里）。

你可以先确认默认值是否是你要连接的 server：

- 位置：`collaboration-tools/packages/open-collaboration-vscode/package.json`
- 字段：`oct.serverUrl`

### 5.2 启动 Host（调试模式）

1. 用 VS Code 打开 `collaboration-tools/packages/open-collaboration-vscode`
2. 打开 `src/extension.ts`
3. 按 `F5` 启动调试

这会打开一个新的 Extension Development Host 窗口。这个窗口就是 Host 客户端。

### 5.3 启动 Guest（脚本方式）

项目提供了一个脚本帮你启动第二个 VS Code 实例：

```bash
chmod +x collaboration-tools/packages/open-collaboration-vscode/launch-guest.sh
./collaboration-tools/packages/open-collaboration-vscode/launch-guest.sh
```

它会创建一套独立的临时用户数据目录，确保两个 VS Code 实例互不干扰。

## 6. 创建房间与加入房间

### 6.1 Host 创建房间

在 Host 窗口底部状态栏找到 Share（或 Open Collaboration）入口，创建新会话。

你会看到它最终弹出邀请码，这就是 `roomId`。把它发给 Guest。

### 6.2 Guest 加入房间

Guest 使用 Join Room 输入邀请码加入。

这里有个重要细节：**Guest 加入时通常会“切换工作区”，你会看到 `oct://` scheme**。

- Guest 使用扩展提供的远程文件系统代理访问共享内容。

后面读 [文件系统代理与远程访问](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/文件系统代理与远程访问.md) 会讲清楚细节。

## 7. 加入超时与 Host 审批

很多人第一次跑会卡在 join timeout：Guest 一直转圈，最后超时。

你可以先把它当成一个三方协作：

- Guest 向 server 提交 join 请求
- server 把 join 请求转发给 Host
- Host 弹窗询问是否允许加入

如果 Host 没点 Allow，Guest 就只能一直等。

### 7.1 Host 侧为什么会弹出“Allow / Deny”

Host 侧处理 join request 的逻辑在 `CollaborationInstance` 里。你可以看到它直接用 VS Code 弹窗让你选：

```ts
connection.peer.onJoinRequest(async (_, user) => {
    const message = vscode.l10n.t(
        'User {0} via {1} login wants to join the collaboration session',
        user.email ? `${user.name} (${user.email})` : user.name,
        user.authProvider ?? 'unknown'
    );
    const allow = vscode.l10n.t('Allow');
    const deny = vscode.l10n.t('Deny');
    const result = await vscode.window.showInformationMessage(message, allow, deny);
    const roots = vscode.workspace.workspaceFolders ?? [];
    return result === allow ? {
        workspace: {
            name: vscode.workspace.name ?? 'Collaboration',
            folders: roots.map(e => e.name)
        }
    } : undefined;
});
```

这段代码同时解释了另一件事：

- Host 允许加入时，会把当前 workspace 的文件夹列表返回给 Guest。

### 7.2 Guest 加入后为什么会“替换工作区文件夹”

Guest 拿到 server 返回的 workspace 信息后，会把它映射成 `oct://` 形式的 workspace folders。

```ts
const workspaceFolders = (vscode.workspace.workspaceFolders ?? []);
const workspace = roomClaim.workspace;
const newFolders = workspace.folders.map(folder => ({
    name: folder,
    uri: CollaborationUri.create(workspace.name, folder)
}));
const uri = await storeWorkspace(newFolders, this.context.globalStorageUri);
if (uri) {
    await vscode.commands.executeCommand(CodeCommands.OpenFolder, uri, {
        forceNewWindow: false,
        forceReuseWindow: true,
        noRecentEntry: true
    });
    return true;
} else {
    return vscode.workspace.updateWorkspaceFolders(0, workspaceFolders.length, ...newFolders);
}
```

Guest 打开由扩展提供的远程工作区。

### 7.3 server 端 join 请求的等待与超时

server 端 `RoomManager.requestJoin()` 做了两件事：

- 给这次 join 请求分配一个 `responseId`（用于轮询 `/api/session/poll/:token`）
- 把请求作为协议消息发给 Host，等待 Host 回包（最长 5 分钟）

```ts
const responseId = this.credentials.secureId();
const timeout = setTimeout(() => {
    pollResult.update({
        code: 'JoinTimeout',
        message: 'Join request has timed out',
        params: [],
        failure: true
    });
    pollResult.dispose();
}, 300_000);

const requestMessage = RequestMessage.create(Messages.Peer.Join, this.credentials.secureId(), '', room.host.id, [user]);
const responsePromise = this.messageRelay.sendRequest(room.host, requestMessage, 300_000);
```

出现 join timeout 时，检查 Host 的加入通知：

- Host 端有没有弹窗？有没有点 Allow？

## 8. 常见问题：遇到 X 先看 Y

- **Guest 加入后工作区是空的**：先确认 Host 端当前 workspace folder 是否为空；再看 Guest 的 folder 映射逻辑（上面的 `newFolders` 段）。
- **能看到文件但编辑不同步**：先看 [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)，确认文本同步链路；再回到 VS Code 端的 `collaboration-instance.ts` 看文档更新节流逻辑。
- **Guest 编辑时被拒绝**：先看 Host 是否设置成 readonly，会触发 guest 文件系统 provider 以只读模式注册。

## 9. 延伸阅读

- [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)
- [open-collaboration-vscode 模块](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/open-collaboration-vscode模块.md)
- [房间生命周期管理](/collaboration_tools/核心模块详解/open-collaboration-server模块/房间与用户管理机制/房间生命周期管理.md)
