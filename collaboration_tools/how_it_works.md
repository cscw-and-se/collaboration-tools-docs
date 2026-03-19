# 如何运行（带你跑通一场真实协作会话）

这页的目标很简单：

- 让你用最短路径跑通 **server + 两个 VS Code 实例**。
- 跑通之后，你能解释“为什么 Guest 会出现 `oct://` 工作区”，以及“加入超时到底在等谁”。

如果你现在还没跑过一次完整流程，我建议你先不要急着读协议细节：先把这页做完，后面的文档会轻松很多。

## 1. 环境准备

### Node.js 版本

本项目要求 Node.js 版本不低于 `20.10.0`，推荐使用 `22.14.0`。

```bash
node --version
```

### 安装依赖与构建

在 `collaboration-tools` 项目根目录执行：

```bash
npm install
npm run build
```

如果编译失败用 `npm ci` 试试。

## 2. 启动后端服务（open-collaboration-server）

在 `collaboration-tools` 项目根目录执行：

```bash
OCT_ACTIVATE_SIMPLE_LOGIN=true npm run start:dev
# 服务器环境可使用
# OCT_ACTIVATE_SIMPLE_LOGIN=true npm run start
```

默认监听 `http://localhost:8100`。

如果你后面遇到“怎么都连不上”，先别怀疑人生，第一步就是确认这个端口确实在监听。

## 3. 启动 VS Code 扩展（open-collaboration-vscode）

协作至少需要两个客户端：一个 Host（共享者），一个 Guest（加入者）。

### 3.1 先确认 serverUrl 配置

扩展会读 VS Code settings 里的 `oct.serverUrl`（默认值在扩展的 `package.json` 里）。

你可以先确认默认值是否是你要连接的 server：

- 位置：`collaboration-tools/packages/open-collaboration-vscode/package.json`
- 字段：`oct.serverUrl`

### 3.2 启动 Host（调试模式）

1. 用 VS Code 打开 `collaboration-tools/packages/open-collaboration-vscode`
2. 打开 `src/extension.ts`
3. 按 `F5` 启动调试

这会打开一个新的 Extension Development Host 窗口。这个窗口就是 Host 客户端。

### 3.3 启动 Guest（脚本方式）

项目提供了一个脚本帮你启动第二个 VS Code 实例：

```bash
chmod +x collaboration-tools/packages/open-collaboration-vscode/launch-guest.sh
./collaboration-tools/packages/open-collaboration-vscode/launch-guest.sh
```

它会创建一套独立的临时用户数据目录，确保两个 VS Code 实例互不干扰。

## 4. 创建房间与加入房间（你会看到的现象，以及它为什么这么设计）

### 4.1 Host 创建房间

在 Host 窗口底部状态栏找到 Share（或 Open Collaboration）入口，创建新会话。

你会看到它最终弹出邀请码，这就是 `roomId`。把它发给 Guest。

### 4.2 Guest 加入房间

Guest 使用 Join Room 输入邀请码加入。

这里有个重要细节：**Guest 加入时通常会“切换工作区”，你会看到 `oct://` scheme**。

这不是装饰效果，而是设计选择：

- Guest 不是直接打开你的本地文件系统。
- Guest 看到的是一个“远程文件系统代理”。

后面读 [文件系统代理与远程访问](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/文件系统代理与远程访问.md) 会讲清楚细节。

## 5. “加入超时”到底在等谁？（把 UI 现象和代码对齐）

很多人第一次跑会卡在 join timeout：Guest 一直转圈，最后超时。

你可以先把它当成一个三方协作：

- Guest 向 server 提交 join 请求
- server 把 join 请求转发给 Host
- Host 弹窗询问是否允许加入

如果 Host 没点 Allow，Guest 就只能一直等。

### 5.1 Host 侧为什么会弹出“Allow / Deny”

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

### 5.2 Guest 加入后为什么会“替换工作区文件夹”

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

你可以把它理解为：Guest 不是“打开你电脑上的目录”，而是“打开一个由扩展提供的远程工作区”。

### 5.3 server 端 join 请求的等待与超时

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

所以当你看到 join timeout，最优先的排查不是“是不是网络不好”，而是：

- Host 端有没有弹窗？有没有点 Allow？

## 6. 常见问题：遇到 X 先看 Y

- **Guest 加入后工作区是空的**：先确认 Host 端当前 workspace folder 是否为空；再看 Guest 的 folder 映射逻辑（上面的 `newFolders` 段）。
- **能看到文件但编辑不同步**：先看 [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)，确认文本同步链路；再回到 VS Code 端的 `collaboration-instance.ts` 看文档更新节流逻辑。
- **Guest 编辑时被拒绝**：先看 Host 是否设置成 readonly，会触发 guest 文件系统 provider 以只读模式注册。

## Summary
- 先跑通一场会话：server + Host + Guest，看到 `oct://` 工作区是正常现象。
- join timeout 绝大多数时候在等 Host 的 Allow，而不是网络。
- Guest “替换工作区文件夹”是核心设计：它需要把 host workspace 映射成远程 URI 才能做文件系统代理。
- 下一步建议阅读：
  - [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)
  - [open-collaboration-vscode 模块](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/open-collaboration-vscode模块.md)
  - [房间生命周期管理](/collaboration_tools/核心模块详解/open-collaboration-server模块/房间与用户管理机制/房间生命周期管理.md)
