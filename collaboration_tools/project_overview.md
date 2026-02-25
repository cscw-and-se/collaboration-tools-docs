# 项目概览（从一次协作会话开始）

第一次打开 `collaboration-tools`，最容易让人迷糊的往往不是某个类怎么写，而是脑子里缺一条“端到端主线”：你在 VS Code 里点一下 Share/Join，背后同时牵动了房间创建、鉴权、文件系统代理、Yjs 文本同步、awareness（光标/选区/在线状态）同步。

建议你先按 [项目是怎么运行的](/collaboration_tools/how_it_works.md) 把 server + 两个 VS Code 跑起来，再回来看这页对照代码，会更顺。

## 从一次真实协作会话开始（你是 Host）

假设你是 Host（发起者）：

1. 你在 VS Code 打开一个本地工作区。
2. 你点状态栏的 Open Collaboration，选择 Create。
3. 扩展向 server 创建房间，拿到 `roomId`（邀请码）和 `roomToken`（JWT，房间会话令牌）。
4. 扩展用 `roomToken` 建立实时连接，进入 Sharing。
5. 你把 `roomId` 发给同学。
6. 同学在 Guest 的 VS Code 里 Join，输入 `roomId`。
7. 你会收到 “Allow / Deny” 弹窗；点 Allow 之后，Guest 才会真正进入协作工作区。
8. 开始协作：文本走 Yjs update；光标/选区/在线状态走 awareness。

把它压缩成一句话：**协作不是“大家一起改文件”，而是“大家共享同一个房间里的状态机”。**

下面按 3 个视角把这条链路落到源码：VS Code 扩展入口、server 侧握手挂载 peer、Yjs/provider 的同步启动。

## VS Code 扩展：从 `activate()` 开始，决定“要不要进入协作态”

扩展的入口就是 `activate()`。它做两件你一定要知道的事：

1. 把 UI 命令挂上（否则你点不到 Share/Join）。
2. 启动时尝试恢复上一次会话（常发生在 Guest 侧：Guest 会保存 roomToken，重启后自动 tryConnect）。

下面这段源码来自 `packages/open-collaboration-vscode/src/extension.ts`：

```ts
export async function activate(context: vscode.ExtensionContext) {
    const container = createContainer(context);
    container.bind(Fetch).toConstantValue(fetch);
    const commands = container.get(Commands);
    commands.initialize();
    const roomService = container.get(CollaborationRoomService);

    const connection = await roomService.tryConnect();
    if (connection) {
        // Wait for the connection to be ready before returning.
        // This allows other extensions that need some workspace information to wait for the data.
        await connection.ready;
    } else {
        await closeSharedEditors();
        removeWorkspaceFolders();
    }
}
```

这段代码解决的问题是：“扩展启动后，要不要进入协作态”。

- 上游怎么到这：VS Code activation events 触发扩展激活。
- 下游会发生什么：`tryConnect()` 成功就会创建 `CollaborationInstance` 并驱动 UI/文件系统代理/Yjs 同步；失败则清掉残留的 `oct://` 工作区状态（典型是上次 Guest 退出不干净）。

如果你想追 Create/Join 的用户路径，从这里往下读最顺：

- 命令与 UI：[`UI集成与命令系统`](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/UI集成与命令系统.md)
- 创建/加入房间：[`核心服务与状态管理`](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/核心服务与状态管理.md)
- 扩展生命周期与清理：[`扩展架构与生命周期`](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/扩展架构与生命周期.md)

## Host 审批加入：弹窗为什么会“卡住加入”

很多“加入卡住”的问题，根本原因不是网络，而是 Host 还没点 Allow。这个弹窗就是 Host 在 `CollaborationInstance` 里注册的 join request handler（源码：`packages/open-collaboration-vscode/src/collaboration-instance.ts`）：

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

这段代码解决的问题是：“Guest 加入是否需要 Host 人工确认，如果允许，需要把当前工作区的名字/根目录列表发给 Guest。”

- 上游怎么到这：Guest 侧发起 join，server 把 join request 转发到 Host peer。
- 下游会发生什么：Host 返回的 `workspace` 信息会变成 Guest 侧的“协作工作区壳”，后续 Guest 的 `oct://` 文件系统代理会根据这个列表展示目录结构。

更细的 join 全链路（含超时到底在等谁）在 [项目是怎么运行的](/collaboration_tools/how_it_works.md) 里会展开。

## Server：`connectChannel()` 在握手阶段做了什么

从 server 视角看，一切都是“某个客户端连上来以后，如何把它挂载成一个 peer，并加入对应房间”。核心入口在 `packages/open-collaboration-server/src/collaboration-server.ts` 的 `connectChannel()`。这里我更建议你拆成两段读：先看握手入参校验与 claim 解析，再看“重连复用 peer vs 新建 peer”。

第一段（握手入参校验 + JWT 解析成 roomClaim）：

```ts
protected async connectChannel(headers: Record<string, string | undefined>, channel: TransportChannel): Promise<void> {
    const jwt = headers['x-oct-jwt'];
    if (!jwt) {
        throw this.logger.createErrorAndLog('No JWT auth token set');
    }
    const publicKey = headers['x-oct-public-key'];
    if (!publicKey) {
        throw this.logger.createErrorAndLog('No encryption key set');
    }
    let compression = headers['x-oct-compression']?.split(',');
    if (compression === undefined || compression.length === 0) {
        compression = ['none'];
    }
    const client = headers['x-oct-client'] ?? 'unknown';
    const roomClaim = await this.credentials.verifyJwt(jwt, isRoomClaim);
    const existingPeer = this.peerManager.getPeer(jwt);
```

第二段（重连复用 peer，或创建 peer 并加入房间）：

```ts
    if (existingPeer) {
        // If a peer with the same JWT already exists, we just update the channel
        // This indicates that a client has reconnected
        existingPeer.channel.transport = channel;
    } else {
        const peer = this.peerFactory({
            jwt,
            user: roomClaim.user,
            host: roomClaim.host ?? false,
            channel,
            client,
            publicKey,
            supportedCompression: compression
        });
        this.peerManager.register(peer);
        await this.roomManager.join(peer, roomClaim.room);
    }
}
```

这段代码解决的问题是：“把一个 socket.io 连接变成可管理的 peer，并把 peer 放进房间。”

- 上游怎么到这：客户端（VS Code 扩展）连接 socket.io 时把 `x-oct-jwt`、`x-oct-public-key` 等 header 带上来。
- 下游会发生什么：`verifyJwt()` 会把 `roomToken` 解析成 `roomClaim`，得到 user/room/host 信息；随后 peer 注册到 `PeerManager`，并由 `RoomManager.join()` 进入房间生命周期（广播、权限、close 等）。

如果你在调试“连接立即断开”，第一时间就该回到这里：缺 jwt / 缺 public key / jwt 不合法，都会直接抛错然后断线。

更细的 server 内部结构在：

- [`open-collaboration-server模块`](/collaboration_tools/核心模块详解/open-collaboration-server模块/open-collaboration-server模块.md)
- [`服务架构与启动流程`](/collaboration_tools/核心模块详解/open-collaboration-server模块/服务架构与启动流程.md)
- [`消息中继与广播系统`](/collaboration_tools/核心模块详解/open-collaboration-server模块/消息中继与广播系统.md)
- [`房间与用户管理机制`](/collaboration_tools/核心模块详解/open-collaboration-server模块/房间与用户管理机制/房间与用户管理机制.md)

## Yjs：`connect()` 为什么一上来要做 sync step + awareness

从协作体验上看，“连上房间”只是开始。真正决定你看到的文本是否一致、光标是否同步的是 Yjs provider。`open-collaboration-yjs` 的 `connect()`（`packages/open-collaboration-yjs/src/yjs-provider.ts`）很典型：一上来先把文档状态对齐，再把 awareness 拉齐。

```ts
connect(): void {
    // write sync step 1
    const encoderSync = encoding.createEncoder();
    syncProtocol.writeSyncStep1(encoderSync, this.doc);
    this.connection.sync.dataUpdate(this.encode(encoderSync));
    // broadcast local state
    const encoderState = encoding.createEncoder();
    syncProtocol.writeSyncStep2(encoderState, this.doc);
    this.connection.sync.dataUpdate(this.encode(encoderState));
    // query awareness info
    this.connection.sync.awarenessQuery();
    // broadcast local awareness info
    const encoderAwareness = encoding.createEncoder();
    encoding.writeVarUint8Array(
        encoderAwareness,
        awarenessProtocol.encodeAwarenessUpdate(this.awareness, [
            this.doc.clientID
        ])
    );
    this.connection.sync.awarenessUpdate(this.encode(encoderAwareness));
}
```

这段代码解决的问题是：“刚连上来时，不依赖‘等别人推你’，而是主动把文档同步和 awareness 同步都启动起来”。

- 上游怎么到这：`CollaborationInstance` 创建 `OpenCollaborationYjsProvider` 后会立刻 `connect()`（并且在 `onReconnect` 时再次 `connect()` 做 resync）。
- 下游会发生什么：对端收到 sync step 后会回对应的 step2/update；awarenessQuery 会触发对端回传所有在线用户的状态，从而你能立刻看到光标/选区。

如果你想把“文本同步”和“awareness 同步”拆开理解，直接读这两页会更快：

- [`open-collaboration-yjs模块`](/collaboration_tools/核心模块详解/open-collaboration-yjs模块.md)
- [`数据模型与状态同步`](/collaboration_tools/数据模型与状态同步.md)

## Summary

- 读代码的主线可以按一次会话走：`activate()` 进入协作态 -> Create/Join -> Host 审批 -> server `connectChannel()` 挂载 peer -> Yjs `connect()` 拉齐文本与 awareness。
- “加入卡住”优先怀疑 Host 是否点了 Allow；这个动作在 `CollaborationInstance` 的 `onJoinRequest` 回调里。
- “一连就断”优先回 `connectChannel()` 看 header/jwt/public key 是否齐全、jwt 是否能 `verifyJwt()`。
- 文本不同步/光标不显示时，优先回 Yjs provider 的 `connect()` 和 `update/awareness handler`，再去看 VS Code 端的编辑器绑定逻辑。
- 下一步阅读建议：
  - 先跑通：[`项目是怎么运行的`](/collaboration_tools/how_it_works.md)
  - 再看 VS Code 端：[`open-collaboration-vscode模块`](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/open-collaboration-vscode模块.md)
  - 再看 server 端：[`open-collaboration-server模块`](/collaboration_tools/核心模块详解/open-collaboration-server模块/open-collaboration-server模块.md)
  - 同步模型：[`open-collaboration-yjs模块`](/collaboration_tools/核心模块详解/open-collaboration-yjs模块.md) + [`数据模型与状态同步`](/collaboration_tools/数据模型与状态同步.md)
