# Socket.IO 封装传输实现（SocketIoTransport）

如果你把协作系统真正跑在复杂网络环境里（校园网、VPN、移动热点），你会发现“重连语义”比“纯粹性能”更重要。`SocketIoTransport` 的价值就在这里：它不试图替你做协议层的复杂事情，只提供一个对协作更友好的连接生命周期。

源码位置：`packages/open-collaboration-protocol/src/transport/socket-io-transport.ts`

## 1. Provider：用 extraHeaders 带握手字段

Socket.IO client 支持 `extraHeaders`，因此我们可以直接把 `x-oct-jwt/public-key/...` 放在 headers 里发给 server：

```ts
export const SocketIoTransportProvider: MessageTransportProvider = {
    id: 'socket.io',
    createTransport: (url, headers) => {
        const socket = io(url, {
            extraHeaders: headers
        });
        const transport = new SocketIoTransport(socket);
        return transport;
    }
};
```

这段代码解决的问题是：“握手字段走标准 headers，不需要像 WebSocket provider 一样做 query 编码”。

## 2. 30 秒窗口：disconnect 不立刻算离线

核心语义就在这段逻辑里：收到 `disconnect` 时不立刻 fire `onDisconnect`，而是给 30 秒重连机会；如果 `reconnect` 在窗口内发生，就取消定时器并 fire `onReconnect`。

```ts
this.socket.on('disconnect', () => {
    this.ready = new Deferred();
    // Give it 30 seconds to reconnect before firing the disconnect event
    this.disconnectTimeout = setTimeout(() => {
        this.onDisconnectEmitter.fire();
        this.disconnectTimeout = undefined;
    }, 30_000);
});
this.socket.io.on('reconnect', () => {
    if (this.disconnectTimeout) {
        clearTimeout(this.disconnectTimeout);
        this.disconnectTimeout = undefined;
        this.ready.resolve();
    }
    this.onReconnectEmitter.fire();
});
```

这段代码解决的问题是：“短暂网络抖动时，不让协作会话立刻进入清理/退出流程”。这也解释了为什么 VS Code 侧会在 `onReconnect` 时触发 resync（例如 Yjs provider 会重新 `connect()`）。

## 3. write/read：仍然是最薄的一层

`SocketIoTransport` 并没有做“可靠投递”之类的额外语义，它仍然只是：

- `write()`：等 ready，再 `socket.send(data)`
- `read()`：监听 `'message'` 事件把二进制交给上层

```ts
constructor(protected socket: Socket) {
    this.socket.on('disconnect', () => {
        this.ready = new Deferred();
        this.disconnectTimeout = setTimeout(() => {
            this.onDisconnectEmitter.fire();
            this.disconnectTimeout = undefined;
        }, 30_000);
    });
    this.socket.io.on('reconnect', () => {
        if (this.disconnectTimeout) {
            clearTimeout(this.disconnectTimeout);
            this.disconnectTimeout = undefined;
            this.ready.resolve();
        }
        this.onReconnectEmitter.fire();
    });
    this.socket.on('connect', () => this.ready.resolve());
}

async write(data: Uint8Array): Promise<void> {
    await this.ready.promise.then(() => this.socket.send(data));
}

read(cb: (data: Uint8Array) => void): void {
    this.socket.on('message', data => cb(data));
}

dispose(): void {
    this.onDisconnectEmitter.dispose();
    this.socket.close();
}
```

也就是说：协议层的 request/response、加密、压缩、路由这些东西不会因为你换了 transport 就改变。

## Summary

- Socket.IO provider 用 `extraHeaders` 传握手字段，路径更直接。
- `SocketIoTransport` 的关键价值是 30 秒重连窗口：disconnect 不立刻算离线。
- 传输层依然是薄封装：write/read，不引入额外的业务语义；复杂逻辑仍在 protocol messaging 层。
