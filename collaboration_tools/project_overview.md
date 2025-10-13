# 项目概览

## 系统架构与核心目标

collaboration-tools 是一个全栈协作系统的 Monorepo 项目，旨在为开发者提供一个开放、可扩展的实时协同编辑平台。其核心目标是实现多用户实时协同编辑，支持远程配对编程、团队协作开发和教育指导等场景。

该系统采用模块化设计，基于 Yjs 的 CRDT（无冲突复制数据类型）算法实现分布式状态同步，确保在高并发环境下编辑操作的一致性与最终一致性。通过定义开放协作协议（Open Collaboration Protocol），系统实现了客户端与服务端之间的标准化通信，支持多种前端编辑器（如 VS Code）的无缝集成。

系统架构愿景包括：

- **实时协同编辑**：基于 Yjs 实现文档内容的实时同步，支持光标位置、文本选择和编辑操作的即时传播。
- **模块化协议设计**：通过 `open-collaboration-protocol` 包定义通用通信协议，便于扩展至其他编辑器或应用。
- **多平台支持**：同时支持 VS Code 插件和基于 Monaco 编辑器的 Web 应用，满足不同开发环境需求。
- **AI Agent 集成（后续）**：引入 AI 辅助功能，通过 `open-collaboration-agent` 提供智能代码建议、自动补全和协作分析能力。

## 核心组件与职责划分

本系统由多个核心包组成，各组件职责明确，协同工作以实现完整的协作功能。

### 协议层：open-collaboration-protocol

该模块是整个系统的核心通信协议实现，定义了客户端与服务端之间消息传递的格式、编码、加密及传输机制。它封装了 WebSocket 和 Socket.IO 等底层传输方式，并提供抽象连接接口，确保协议的可移植性和扩展性。

主要功能包括：

- 消息编码与压缩（`messaging/encoding.ts`, `messaging/compression.ts`）
- 安全通信支持（`messaging/encryption.ts`）
- 连接管理与错误处理（`connection.ts`, `connection-provider.ts`）

此包作为 TypeScript 库，可供开发者用于构建自定义客户端，适配不同编辑器或 Web 应用。

### 服务端：open-collaboration-server

服务端负责管理协作会话、用户认证、房间管理和消息中继。所有客户端必须连接到同一服务器实例才能参与协作。

关键特性：

- 支持多种认证方式：简单登录（`simple-login-endpoint.ts`）、OAuth（如 GitHub、Google）和 Keycloak 集成（目前由于网络原因等因素，仅支持简单登录）
- 房间管理（`room-manager.ts`）与用户状态跟踪（`user-manager.ts`）
- 消息转发（`message-relay.ts`）与频道通信（`channel.ts`）
- 可配置环境变量（如 `OCT_SERVER_OWNER`, `OCT_JWT_PRIVATE_KEY`）以适应企业部署需求

服务端可通过 Docker 容器化部署，也支持作为 TypeScript 库嵌入自定义应用。

### 客户端集成：open-collaboration-vscode 与 open-collaboration-monaco

#### open-collaboration-vscode

VS Code 扩展提供完整的协作界面，集成于状态栏，支持创建/加入会话、邀请码分享、用户跟随等功能。通过 `extension.ts` 入口启动，利用 `collaboration-connection-provider.ts` 建立与服务器的连接。

主要功能：

- 状态栏“Share”按钮控制协作会话
- 快速选择（QuickPick）菜单管理会话生命周期
- 用户光标颜色标识与跟随功能
- 支持多语言本地化（`.nls.*.json` 文件）

#### open-collaboration-monaco

为基于 Monaco 编辑器的 Web 应用提供协作能力。通过 `MonacoCollabApi` 类暴露简洁 API，支持：
- `createRoom()` / `joinRoom(roomToken)`：房间创建与加入
- `setEditor(editor)`：绑定 Monaco 实例
- `onUsersChanged()`：监听用户状态变化
- `followUser(id)`：跟随指定用户编辑行为

适用于在线 IDE、代码评审工具等 Web 场景。

### 状态同步：open-collaboration-yjs

该模块集成 Yjs 库，实现基于 CRDT 的文档状态同步。Yjs 提供高效的并发编辑算法，确保多个用户同时修改同一文档时不会产生冲突。

核心组件：

- `yjs-provider.ts`：Yjs 与协作协议的桥梁，负责将本地编辑操作广播至其他客户端
- `yjs-normalized-text.ts`：文本内容的规范化处理，保证跨平台一致性

通过 Yjs，系统实现了 OT（操作变换）之外的另一种高效协同方案，特别适合大规模并发场景。

### AI 辅助：open-collaboration-agent

AI Agent 模块为协作系统引入智能化能力，支持 CLI 工具调用 AI 模型进行代码分析、建议生成和文档同步优化。

依赖项：

- `@ai-sdk/openai` 和 `@ai-sdk/anthropic`：接入主流 AI 模型（如 GPT、Claude）
- `ai`：AI 流式处理框架
- 依赖 `open-collaboration-protocol` 和 `open-collaboration-yjs` 获取协作上下文

典型用途：

- 自动生成代码注释
- 协作会话中的智能提示
- 编辑冲突的 AI 辅助解决
