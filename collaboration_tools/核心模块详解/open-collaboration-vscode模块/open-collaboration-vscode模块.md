# open-collaboration-vscode 模块（VS Code 端主线）

如果你们团队真正用这个项目来做“VS Code 上的实时协同编程”，那么你每天碰到的绝大多数问题都会落在 `open-collaboration-vscode`：

- 用户点了 Share/Join，为什么 UI 是这个反应？
- Guest 看到 `oct://` 工作区时，文件树/打开文件/保存文件分别走哪条链路？
- 同步“看起来怪怪的”（比如回写、抖动、延迟），到底是 VS Code 事件处理的问题，还是同步层的问题？

这一页把 VS Code 端的主线串起来：从“用户动作”一路走到“网络连接 + 协作会话实例”。如果你想先跑通，建议配合 [项目是怎么运行的](/collaboration_tools/how_it_works.md) 一起读。

## 模块里最重要的几个角色

在源码里，你可以先只记这三个类，其他都可以当成“围绕它们转的配角”：

- `Commands`：把 UI 入口（状态栏/命令面板/QuickPick）变成具体动作。
- `CollaborationRoomService`：负责 create/join/tryConnect，决定“什么时候建立连接、什么时候切换工作区”。
- `CollaborationInstance`：一场真实会话的运行时对象，里面挂着 Yjs provider、awareness、文件系统代理、编辑器事件监听。

接下来我们用真实代码把这三个角色串起来。

## 1) Create：从 QuickPick 到创建 `CollaborationInstance`

Host 点 Create 之后，最终会落到 `CollaborationRoomService.createRoom()`。这段代码的价值是：它把“创建房间 + 拿 token + 建连 + 创建实例 + 告诉用户邀请码”这些步骤一口气做完。

```ts
const roomClaim = await connectionProvider.createRoom({
    abortSignal: this.toAbortSignal(this.tokenSource.token, cancelToken),
    reporter: info => progress.report({ message: localizeInfo(info) })
});
if (roomClaim.loginToken) {
    const userToken = roomClaim.loginToken;
    await this.secretStore.storeUserToken(url, userToken);
}
const connection = await connectionProvider.connect(roomClaim.roomToken);
const instance = this.instanceFactory({
    serverUrl: url,
    connection,
    host: true,
    roomId: roomClaim.roomId
});
await vscode.env.clipboard.writeText(roomClaim.roomId);
// ... showInformationMessage + onDidJoinRoomEmitter.fire(instance)
```

怎么理解它的上下游：

- 上游：`Commands` 负责触发 `roomService.createRoom()`（见 [UI 集成与命令系统](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/UI集成与命令系统.md)）。
- 下游：`connectionProvider.connect(roomClaim.roomToken)` 之后，连接就进入 server 的 `connectChannel()`；`instanceFactory(...)` 创建出一场“活着的会话”。

你在二开时改 Create 流程，最常见的改动点是：

- 进度提示/取消逻辑（`withProgress` + abortSignal）。
- 创建完后复制到剪贴板/生成邀请链接（`RoomUri.create(...)`）。

## 2) Join：为什么 Guest 会突然切到 `oct://` 工作区

Join 的核心不只是“连上房间”，还包括“把 VS Code 的 workspace folders 替换成 `oct://` 形式”。这是 Guest 能看到 Host 文件树的前提。

下面这段代码就是 join 成功后做的“工作区切换”动作：

```ts
const roomData: RoomData = {
    serverUrl: url,
    roomToken: roomClaim.roomToken,
    roomId: roomClaim.roomId,
    host: roomClaim.host
};
await this.secretStore.storeRoomData(roomData);
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
}
```

这段代码解决的问题是：“让 Guest 的 VS Code 认为自己打开了一个 workspace，但这个 workspace 的文件 URI 全是 `oct://...`”。

- 上游：Guest 通过 `connectionProvider.joinRoom(...)` 拿到 roomClaim（里面包含 workspace folders）。
- 下游：VS Code 会开始对 `oct://` 发起 `stat/readdir/readFile` 等调用，最终触发文件系统代理（见 [文件系统代理与远程访问](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/文件系统代理与远程访问.md)）。

如果你遇到“Join 成功但工作区没切换/树是空的”，优先看这一段：`storeWorkspace(...)` 返回了什么、`OpenFolder` 是否被执行。

## 3) 会话实例：`CollaborationInstance.init()` 把所有链路“点火”

一旦 create/join 成功，`CollaborationInstance` 会被创建，并在 `init()` 里把协作真正跑起来。

这段代码我建议你反复读三遍，因为它基本解释了“为什么断线后还能追上”“为什么 Guest 才注册文件系统”“为什么会一直 resync”。

```ts
@postConstruct()
protected init(): void {
    if (this.options.host) {
        // The host is always ready
        this._ready.resolve();
    }
    CollaborationInstance.Current = this;
    const connection = this.options.connection;
    this.yjsProvider = new OpenCollaborationYjsProvider(connection, this.yjs, this.yjsAwareness, {
        resyncTimer: 10_000 // resync every 10 seconds
    });
    if (this.options.hostId) {
        this.fileSystemManager = new FileSystemManager(connection, this.yjs, this.options.hostId);
        this.toDispose.push(this.fileSystemManager);
    }
    this.yjsProvider.connect();
    this.toDispose.push(connection);
    this.toDispose.push(connection.onDisconnect(() => {
        this.dispose();
    }));
    this.toDispose.push(connection.onReconnect(() => {
        // Reconnect the Yjs provider
        // This will resync all missed messages
        this.yjsProvider.connect();
    }));
    this.toDispose.push(this.yjsProvider);
    // ... register peer/room handlers + file/editor events
}
```

这段代码解决的问题：

- “会话启动后，如何让同步和事件监听开始工作”：`yjsProvider.connect()` + `registerFileEvents/registerEditorEvents()`。
- “Guest 如何拿到 `oct://` 文件系统能力”：只有在 `options.hostId` 存在时才创建 `FileSystemManager` 并注册 provider（Host 不需要）。
- “断线重连如何补齐状态”：重连时再跑一次 `yjsProvider.connect()`，会触发同步 step 流程（这在排查“重连后不同步”时很关键）。

如果你遇到“编辑器里内容偶尔被回写”，通常不是网络丢包，而是 VS Code 事件与 Yjs 事件在互相触发，建议结合 [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md) 看 `origin` 与节流逻辑。

## Summary

- VS Code 端主线可以按三个类读：`Commands` -> `CollaborationRoomService` -> `CollaborationInstance`。
- Join 的关键不只是 connect，而是把 workspace folders 替换成 `oct://`，从而触发文件系统代理链路。
- `CollaborationInstance.init()` 是“点火开关”：Yjs provider、文件系统、重连 resync 都在这里。
- 下一步阅读建议：
  - UI/命令入口： [UI 集成与命令系统](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/UI集成与命令系统.md)
  - 文件系统代理： [文件系统代理与远程访问](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/文件系统代理与远程访问.md)
  - 会话与状态： [核心服务与状态管理](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/核心服务与状态管理.md)
