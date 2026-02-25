# open-collaboration-protocol 模块（从“点一下 Share/Join”讲起）

很多同学第一次读协作系统，会下意识从“server 有哪些类、vscode 有哪些 service”开始。然后越读越乱：房间、peer、token、加密、压缩、广播……每个词都认识，但连不成一条可调试的链路。

`open-collaboration-protocol` 这个包解决的事情其实很单纯：**把“HTTP 登录/创建/加入房间” 和 “实时消息收发” 串成一个统一的客户端协议层**。你可以把它理解为：

- 上半段：`ConnectionProvider` 用 HTTP API 跟 server 说话（login / createRoom / joinRoom / poll）
- 下半段：`ProtocolBroadcastConnection` 用传输层（Socket.IO / WebSocket）跟 server 建立实时通道，并提供更“语义化”的 API（room/peer/fs/editor/sync）

这一页先不讲“协议细节百科”，只把你在代码里最常追的 3 个入口钉牢：初始化（crypto）、连接建立（headers + keypair）、连接对象的语义 API（room/peer/fs/...）。

## 1. 初始化：为什么必须先 `initializeProtocol()`

协议层的加密实现不是“自己凭空出现”的，它依赖运行时提供的 crypto（Node 侧是 `crypto.webcrypto`，浏览器侧是 `self.crypto`）。因此客户端启动阶段必须先做一次初始化。

以 VS Code 扩展为例（`packages/open-collaboration-vscode/src/extension.ts`），它在模块加载时就会把 Node 的 `webcrypto` 注入协议层：

```ts
import 'reflect-metadata';
import * as crypto from 'node:crypto';
import { initializeProtocol } from 'open-collaboration-protocol';

initializeProtocol({
    cryptoModule: crypto.webcrypto
});

export async function activate(context: vscode.ExtensionContext) {
    // ...
}
```

这段代码解决的问题是：“让协议层有能力生成 key pair、加密/解密消息”。你不做这步，后面任何加密相关逻辑都会炸。

协议层在真正用到 crypto 时会做一次硬校验（`packages/open-collaboration-protocol/src/utils/crypto.ts`）：

```ts
export const setCryptoModule = (cm: CryptoModule): void => {
    cryptoModule = cm;
};

export const getCryptoLib = (): CryptoLib => {
    if (cryptoModule === undefined) {
        throw new Error('Crypto module is not available. Please call the initializeProtocol() function first.');
    }
    const subtle = cryptoModule.subtle;
    return {
        async generateKeyPair() {
            // ...
        }
    };
};
```

上游怎么到这：客户端启动阶段（扩展激活/网页初始化）。  
下游会发生什么：后续所有 `Encryption.encrypt/decrypt`、`Encryption.generateKeyPair` 都会依赖这个 cryptoModule。

## 2. 建立实时连接：`connect()` 怎么把 roomToken 变成一个连接对象

当你完成 login + create/join 拿到 `roomToken` 之后，客户端会走到 `ConnectionProvider.connect()`（源码：`packages/open-collaboration-protocol/src/connection-provider.ts`）：

```ts
async connect(roomAuthToken: string, host?: types.Peer): Promise<ProtocolBroadcastConnection> {
    const metadata = await this.getMetaData();
    const transportIndex = this.findFitting(metadata.transports, this.options.transports.map(t => t.id));
    const transportProvider = this.options.transports[transportIndex];
    const keyPair = await Encryption.generateKeyPair();
    const transport = transportProvider.createTransport(this.options.url, {
        'x-oct-jwt': roomAuthToken,
        'x-oct-public-key': keyPair.publicKey,
        'x-oct-client': this.options.client ?? 'Unknown OCT JS Client',
        'x-oct-compression': 'gzip'
    });
    const connection = createConnection({ privateKey: keyPair.privateKey, transport, host });
    return connection;
}
```

这段代码解决的问题是：“把 `roomToken` + 本地生成的 keyPair（公钥/私钥）打包成握手 headers，然后选一个 transport 建立通道，最后构造 `ProtocolBroadcastConnection`”。

- 上游怎么到这：`createRoom()` 或 `joinRoom()` 返回 `roomToken`。
- 下游会发生什么：server 端会在 `connectChannel()` 校验 `x-oct-jwt` / `x-oct-public-key`（见 [项目概览](/collaboration_tools/project_overview.md) 的 server 小节），并把连接挂载为 peer 加入房间。

## 3. 连接对象的“语义 API”：`room/peer/fs/editor/sync` 从哪里来的

你在 VS Code 扩展里用到的 `connection.peer.onJoinRequest`、`connection.fs.onReadFile`、`connection.sync.dataUpdate` 这些 API，不是“魔法”，就是 `ProtocolBroadcastConnectionImpl` 把 method name 绑定成了更好用的 handler/emit 接口（源码：`packages/open-collaboration-protocol/src/connection.ts`）：

```ts
export class ProtocolBroadcastConnectionImpl extends AbstractBroadcastConnection {
    room: RoomHandler = {
        onJoin: handler => this.onBroadcast(Messages.Room.Joined, (origin, peer) => {
            this.onDidJoinRoom(peer);
            handler(origin, peer);
        }),
        leave: () => this.sendNotification(Messages.Room.Leave, ''),
        onPermissions: handler => this.onBroadcast(Messages.Room.PermissionsUpdated, handler),
        updatePermissions: permissions => this.sendBroadcast(Messages.Room.PermissionsUpdated, permissions)
    };

    peer: PeerHandler = {
        onJoinRequest: handler => this.onRequest(Messages.Peer.Join, handler),
        onInfo: handler => this.onNotification(Messages.Peer.Info, handler),
        onInit: handler => this.onNotification(Messages.Peer.Init, async (origin, response) => {
            this.onDidInit(response);
            handler(origin, response);
        }),
        init: (target, request) => this.sendNotification(Messages.Peer.Init, target, request)
    };
}
```

这段代码解决的问题是：“上层不需要到处写字符串 method，也不需要自己处理 request/response 的匹配；协议层提供一套统一的 handler 模型（onRequest/onNotification/onBroadcast + sendXXX）。”

- 上游怎么到这：`ConnectionProvider.connect()` 调 `createConnection()` 构造出来。
- 下游会发生什么：VS Code 扩展/文件系统代理/Yjs provider 会在这里注册 handler，消息进来后由协议层路由到正确的回调（详见 [消息编码与处理机制](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/消息编码与处理机制/消息编码与处理机制.md)）。

## 该怎么读这一组文档（按你会真正遇到的问题）

如果你是“要跑通/要排障/要二开”，建议你按下面顺序读：

1. 先搞清楚 token 与 connect： [核心连接机制](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/核心连接机制.md)
2. 再搞清楚消息是怎么被路由的： [消息编码与处理机制](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/消息编码与处理机制/消息编码与处理机制.md)
3. 最后再看加密、压缩、传输的工程细节：
   - [消息压缩策略](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/消息编码与处理机制/消息压缩策略.md)
   - [消息加密与安全传输](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/消息编码与处理机制/消息加密与安全传输.md)
   - [传输层适配实现](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/传输层适配实现/传输层适配实现.md)

如果你是“想快速定位某个 method 是谁发的、谁处理的”，直接跳到：

- [协议方法对照表](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/协议方法对照表.md)

## Summary

- `open-collaboration-protocol` 的核心价值：把 HTTP 的 login/create/join 和实时消息收发统一成一个协议层。
- `initializeProtocol()` 必须先调用，否则加密相关逻辑会因为缺 cryptoModule 直接报错。
- `ConnectionProvider.connect()` 把 `roomToken` + 本地 keyPair 变成握手 headers（`x-oct-jwt/public-key/...`），并构造 `ProtocolBroadcastConnection`。
- `ProtocolBroadcastConnectionImpl` 把 method name 绑定成 `room/peer/fs/editor/sync` 的语义 API，上层写业务时不需要关心底层 message envelope。
