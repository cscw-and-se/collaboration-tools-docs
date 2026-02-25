# open-collaboration-server 模块（房间、Peer、消息中继、认证）

如果你把整个协作系统想成一个“多人同时在线的状态机”，那么 `open-collaboration-server` 就是状态机的裁判：

- 谁能进房间（鉴权 + join 审批）？
- 谁在房间里（Peer 管理）？
- 谁发给谁（消息中继：request/notification/broadcast）？
- 断线重连算不算同一个人（JWT 复用策略）？

这一页用“从连接进来到同步跑起来”的顺序，把 server 端主链路讲清楚，并在关键位置贴 10-30 行真实代码方便你对照源码。

## server 端的三条主线

你读 server 源码时，建议把注意力拆成三条主线（不然很容易迷路）：

1. 连接入口：`CollaborationServer.connectChannel()`，决定这条连接是否可信、是否要复用既有 Peer。
2. 房间与 join：`RoomManager.prepareRoom/join/requestJoin/leaveRoom`，决定 room 的生命周期和 join 审批行为。
3. 消息分发：`PeerImpl.receiveMessage()` + `MessageRelay.sendRequest/sendBroadcast`，决定消息怎么路由与广播。

下面我们按这三条主线走一遍。

## 1) 连接入口：`connectChannel()` 为什么要 JWT + public key

客户端连到 server（Socket.IO）时，会在 headers 里带上：

- `x-oct-jwt`：会话 JWT（可能是 user token，也可能是 room token，取决于阶段）。
- `x-oct-public-key`：用于消息级加密的公钥。
- `x-oct-compression`：声明支持的压缩算法集合。

server 端入口代码非常“硬核”地把这些条件写死了：

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
    if (existingPeer) {
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

这段代码解决的核心问题是：“server 如何把一条 socket 连接变成一个可管理的 Peer，并把它挂到某个 room”。你读这段时建议重点看：

- `verifyJwt(jwt, isRoomClaim)`：这意味着连接阶段用的 JWT 必须能解成 roomClaim。
- `existingPeer` 分支：断线重连复用 Peer，只替换 transport，这会影响你们对“重连是否算同一会话”的理解。

## 2) 房间与 join 审批：`requestJoin()` 到底在等谁

Guest join 一个房间时，你在客户端会看到 “WaitingForHost” 或 “JoinTimeout”。这不是 UI 编的状态，而是 server 在 `RoomManager.requestJoin()` 里明确写出来的状态机。

这段代码的关键点有三个：

1. 给 join 请求生成 `responseId`，用于客户端轮询。
2. 向 Host 发送 `Messages.Peer.Join` request，并等待 response（5 分钟超时）。
3. Host 允许加入时，server 给 Guest 签发新的 roomToken（JWT），用 `roomClock` 确保会话唯一。

```ts
async requestJoin(room: Room, user: User): Promise<string> {
    const responseId = this.credentials.secureId();
    const timeout = setTimeout(() => {
        pollResult.update({
            code: 'JoinTimeout',
            message: 'Join request has timed out',
            params: [],
            failure: true
        });
        pollResult.dispose();
    }, 300_000); // 5 minutes of timeout
    // ...
    const requestMessage = RequestMessage.create(Messages.Peer.Join, this.credentials.secureId(), '', room.host.id, [user]);
    const responsePromise = this.messageRelay.sendRequest(room.host, requestMessage, 300_000);
    responsePromise.then(async response => {
        if (ResponseMessage.is(response)) {
            const joinResponse = response.content.response as JoinResponse | undefined;
            if (!joinResponse) {
                pollResult.update({ failure: true, code: 'JoinRejected', params: [], message: 'Join request has been rejected' });
                return;
            }
            const claim: RoomClaim = { room: room.id, user: { ...user }, roomClock: ++room.clock };
            const jwt = await this.credentials.generateJwt(claim);
            const joinRoomResponse: JoinRoomResponse = { roomId: room.id, roomToken: jwt, workspace: joinResponse.workspace, host: room.host.toProtocol() };
            pollResult.update(joinRoomResponse);
        }
    });
    pollResult.update({ failure: false, code: 'WaitingForHost', params: [], message: 'Waiting for host to accept join request' });
    return responseId;
}
```

这段代码最“真实”的地方在于：它说明 join 不是瞬时的，而是一个“Host 参与的审批流程”。因此当你排查 join 超时时，建议按这条链路看：

- Host 端是否收到了 join request（见 VS Code 端的 `onJoinRequest` 弹窗逻辑）。
- request 是否在 `sendRequest` 层超时（消息是否路由到了 host peer）。
- poll 是否还在（`pollResults` 是否被提前 dispose）。

更细的房间生命周期拆解在 [房间与用户管理机制](/collaboration_tools/核心模块详解/open-collaboration-server模块/房间与用户管理机制/房间与用户管理机制.md)。

## 3) 广播与加密：为什么 `sendBroadcast()` 要“裁剪 key”

协作里最频繁的消息是 broadcast（比如 Yjs update、awareness update、成员加入/离开通知）。而消息级加密意味着：同一条广播消息里，会带很多个“给不同 peer 的对称密钥加密结果”。但某个 peer 收到广播时，其实只需要属于自己的那把 key。

`MessageRelay.sendBroadcast()` 里就有一个很关键的工程化处理：给每个 peer 发消息时，只保留它那一条 key，其余全裁剪掉。

```ts
for (const peer of room.peers) {
    if (peer !== origin) {
        // Find the key for the target peer
        const peerKey = message.metadata.encryption.keys.find(e => e.target === peer.id);
        if (peerKey && BroadcastMessage.isEncrypted(message)) {
            // Adjust the message to only contain the key for the target peer
            // All other keys are not of use for the target peer
            const messageWithSingleKey: EncryptedBroadcastMessage = {
                ...message,
                metadata: {
                    ...message.metadata,
                    encryption: { keys: [peerKey] }
                }
            };
            peer.channel.sendMessage(messageWithSingleKey);
        } else if (!Message.isEncrypted(message)) {
            peer.channel.sendMessage(message);
        }
    }
}
```

这段代码解决的问题是：“让加密广播在工程上可用（避免把所有人的 key 都塞给每个人）”。如果你们之后要扩展新的广播消息类型，建议保持同样的策略：广播时按目标裁剪 key，避免把无关密钥分发出去。

## Summary

- server 端主线建议抓三条：`connectChannel()`（入口与重连策略）、`RoomManager.requestJoin()`（join 审批与轮询）、`MessageRelay.sendBroadcast()`（广播与加密裁剪）。
- join 的超时/等待不是 UI 凭空出现的，而是 server 明确维护的状态机；排障要按“host 是否审批 -> request 是否超时 -> poll 是否存活”走。
- 下一步阅读建议：
  - 启动与 API： [服务架构与启动流程](/collaboration_tools/核心模块详解/open-collaboration-server模块/服务架构与启动流程.md)
  - 房间与 Peer： [房间与用户管理机制](/collaboration_tools/核心模块详解/open-collaboration-server模块/房间与用户管理机制/房间与用户管理机制.md)
  - 广播与消息路由： [消息中继与广播系统](/collaboration_tools/核心模块详解/open-collaboration-server模块/消息中继与广播系统.md)
  - 认证与接入： [认证与安全接入](/collaboration_tools/核心模块详解/open-collaboration-server模块/认证与安全接入/认证与安全接入.md)

