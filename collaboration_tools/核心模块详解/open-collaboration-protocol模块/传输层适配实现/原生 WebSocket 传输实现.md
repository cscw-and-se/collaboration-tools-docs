# 原生 WebSocket 传输实现（WebSocketTransport）

这页只对照 `open-collaboration-protocol` 当前代码里真实实现的 WebSocket transport：它很轻、很直接，也很“没有多余语义”。

如果你期待它像 Socket.IO 一样自动重连、断线缓冲，那你会失望：**这里没有**。这不是缺陷，而是设计取舍。

源码位置：`packages/open-collaboration-protocol/src/transport/websocket-transport.ts`

## 1. Provider：把 base URL 转成 ws/wss，并把 headers 拼进 query

WebSocket 在浏览器环境里不能像 Socket.IO 一样随意塞自定义 headers，因此 provider 选择把 headers 编成 query string：

```ts
if (url.startsWith('https')) {
    url = url.replace('https', 'wss');
} else if (url.startsWith('http')) {
    url = url.replace('http', 'ws');
}
if (url.endsWith('/')) {
    url = url.slice(0, -1);
}
const query = Object.entries(headers).map(([key, value]) => `${encodeURIComponent(key)}=${encodeURIComponent(value)}`).join('&');
const socket = new WebSocketTransportProvider.Constructor(url + '/websocket' + (query ? '?' + query : ''));
socket.binaryType = 'arraybuffer';
```

这段代码解决的问题是：“在 WebSocket 受限的环境里，仍然把 `x-oct-jwt/public-key/...` 这些握手信息带给 server”。

上游怎么到这：`ConnectionProvider.connect()` 选择了 `WebSocketTransportProvider`。  
下游会发生什么：server 端需要支持 `/websocket` 路径，并能从 query 里解析出握手字段（如果 server 侧只启用了 socket.io，则这条路径可能不可用）。

## 2. Transport：没有重连事件，断开就是断开

WebSocketTransport 本体基本就是一层事件转发 + ready gate：

```ts
get onReconnect(): Event<void> {
    return Event.None;
}
constructor(protected socket: WebSocket) {
    this.socket.onclose = () => this.onDisconnectEmitter.fire();
    this.socket.onerror = () => this.onErrorEmitter.fire('Websocket connection closed unexpectedly.');
    this.socket.onopen = () => this.ready.resolve();
}
async write(data: Uint8Array): Promise<void> {
    await this.ready.promise.then(() => this.socket.send(data));
}
read(cb: (data: Uint8Array) => void): void {
    this.socket.onmessage = event => cb(event.data);
}
```

这段代码解决的问题是：“提供最小的可用 transport 契约”。

你在排障时可以直接用这几条结论：

- `onReconnect` 永远不会触发（Event.None）
- `onclose` 触发后，协议层会 dispose connection（见 [传输层适配实现](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/传输层适配实现/传输层适配实现.md)）
- `write()` 会等待 `onopen`，因此“写不出去”很可能意味着连接还没 ready

## Summary

- WebSocketTransportProvider 用 query 携带握手字段，是为了解决浏览器无法自定义 headers 的限制。
- WebSocketTransport 没有自动重连，`onReconnect = Event.None`，断开后协议层通常会直接 dispose。
- 如果你需要重连语义，请优先考虑 Socket.IO transport，或在更上层实现重连策略。

