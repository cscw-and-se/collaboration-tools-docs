# Y.js：实时协同的 CRDT 实现

在讨论协同冲突时，我们提到过 OT 和 CRDT 两种算法。Y.js 是目前应用最广、性能较好的 CRDT 开源实现。它把数据同步和冲突合并的底层逻辑都封装好了，你只需要使用它提供的接口，就能为应用（文本编辑器、白板、表单等）加上实时协同能力。

- 官方网址：https://yjs.dev/
- 官方文档：https://docs.yjs.dev/

## 1. 核心概念

**Y.Doc（文档）**

Y.Doc 是 Y.js 的最顶层容器，一个实例代表一个需要同步的协同会话或文档，所有共享数据都放在里面。

**Shared Types（共享类型）**

你不能直接把普通的 JavaScript 对象或字符串放进 Y.Doc，必须使用 Y.js 提供的共享类型。这些类型内置了 CRDT 的合并逻辑，能自动处理并发修改。常用的有：

- `Y.Text`：协同编辑文本，还能记录加粗、斜体等格式信息。
- `Y.Array`：协同编辑数组，多人同时增删改也能保证结果一致。
- `Y.Map`：协同编辑键值对。

**Providers（通信模块）**

Y.Doc 和共享类型只在本地处理数据，并不知道如何把数据发给其他协作者。Provider 负责连接 Y.Doc 和通信后端：监听本地更新并广播出去，同时接收远端更新并应用到本地。

最常用的是 `y-websocket`，它通过 WebSocket 把所有客户端连接到一个中心服务器来交换数据。也可以使用 `y-webrtc` 做 P2P 连接，或用 `y-indexeddb` 做本地存储。

## 2. 代码示例

Y.js 官方提供了不少示例。例如基于 CodeMirror 的协同编辑示例：https://github.com/yjs/yjs-demos/tree/main/codemirror.next

## 3. 为什么选择 Y.js

- **性能较好**：CRDT 算法做过优化，在大型文档和高并发场景下表现稳定。
- **生态丰富**：提供与多种编辑器和框架的绑定（Bindings），例如 Monaco Editor、CodeMirror、ProseMirror、Tiptap，用少量代码就能集成协同能力。
- **离线优先**：基于 CRDT 的架构天然支持离线编辑，断网期间的修改会在恢复连接后自动同步。
- **不绑定后端**：除了 `y-websocket`，也可以自行实现 Provider 对接任意后端。

对我们的协同编程项目来说，Y.js 是成熟且灵活的选择。
