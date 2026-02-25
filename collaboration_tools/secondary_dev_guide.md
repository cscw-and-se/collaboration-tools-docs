# 二次开发指南

> 前置条件：先按 [项目是怎么运行的](how_it_works.md) 跑通一次完整链路（server + 两个 VS Code 实例），再来看这篇。

这篇文档面向想在 `open-collaboration-tools` 基础上做二次开发的同学，通过几个典型场景说明改动入口在哪、改什么、不需要改什么。

---

## 快速定位：改哪个包？

| 你想改的东西 | 对应包 | 关键文件 |
|---|---|---|
| VS Code 界面、命令、光标显示 | `open-collaboration-vscode` | `commands.ts`, `collaboration-instance.ts` |
| 协作会话的创建/加入流程 | `open-collaboration-vscode` | `collaboration-room-service.ts` |
| 文件系统代理（`oct://` 路径） | `open-collaboration-vscode` | `collaboration-file-system.ts` |
| 消息类型定义 | `open-collaboration-protocol` | `messages.ts` |
| 服务端房间/用户管理 | `open-collaboration-server` | `room-manager.ts`, `peer-manager.ts` |
| 认证方式 | `open-collaboration-server` | `auth-endpoints/` |
| Yjs 同步逻辑 | `open-collaboration-yjs` | `yjs-provider.ts` |

---

## 场景一：给 VS Code 扩展加一个新命令

以"显示当前房间所有成员列表"为例。

**第一步：在 `commands-list.ts` 里声明命令 ID**

```typescript
// packages/open-collaboration-vscode/src/commands-list.ts
export namespace OctCommands {
    // ... 已有命令
    export const ShowMembers = 'oct.showMembers';  // 新增
}
```

**第二步：在 `package.json` 的 `contributes.commands` 里声明**

```json
// packages/open-collaboration-vscode/package.json
{
  "contributes": {
    "commands": [
      {
        "command": "oct.showMembers",
        "title": "%oct.showMembers.title%",
        "category": "Open Collaboration"
      }
    ]
  }
}
```

**第三步：在 `commands.ts` 的 `initialize()` 里注册处理函数**

```typescript
// packages/open-collaboration-vscode/src/commands.ts
initialize(): void {
    this.context.subscriptions.push(
        // ... 已有命令
        vscode.commands.registerCommand(OctCommands.ShowMembers, async () => {
            const instance = CollaborationInstance.Current;
            if (!instance) return;
            // instance.peers 里有所有成员信息
            const members = instance.peers.map(p => p.peer.name).join(', ');
            vscode.window.showInformationMessage(`当前成员：${members}`);
        })
    );
}
```

`Commands` 类通过 InversifyJS 的 `autoBindInjectable: true` 自动注册，不需要手动在 `inversify.ts` 里添加绑定。如果你的新命令需要一个全新的服务类，才需要在 `inversify.ts` 里绑定。

---

## 场景二：在协议层加一个新消息类型

假设你想让 host 能广播一条"会话公告"给所有成员。

**第一步：在 `messages.ts` 里定义新消息类型**

```typescript
// packages/open-collaboration-protocol/src/messages.ts
export namespace Messages {
    // ... 已有命名空间
    export namespace Session {
        // BroadcastType：服务端会自动转发给房间里所有人
        export const Announcement = new BroadcastType<[string]>('session/announcement');
    }
}
```

**第二步：在 `collaboration-instance.ts` 里发送和接收**

发送端（host）：
```typescript
// packages/open-collaboration-vscode/src/collaboration-instance.ts
this.connection.room.onRequest(Messages.Session.Announcement, (_, text) => {
    // 收到公告时显示通知
    vscode.window.showInformationMessage(`公告：${text}`);
});

// 发送公告
this.connection.room.sendBroadcast(Messages.Session.Announcement, 'Hello everyone!');
```

**关键点：服务端不需要改。** `message-relay.ts` 会自动把 `BroadcastType` 的消息转发给房间里所有 peer，它不关心消息内容是什么。只有当你需要服务端参与处理逻辑时（比如持久化、权限校验），才需要改服务端。

---

## 场景三：修改服务端认证方式

项目支持通过实现 `AuthEndpoint` 接口来添加新的认证方式。以 `SimpleLoginEndpoint` 为参考：

**第一步：实现 `AuthEndpoint` 接口**

```typescript
// packages/open-collaboration-server/src/auth-endpoints/my-auth-endpoint.ts
import { injectable, inject } from 'inversify';
import { AuthEndpoint, AuthSuccessEvent } from './auth-endpoint.js';
import { Emitter } from 'open-collaboration-protocol';

@injectable()
export class MyAuthEndpoint implements AuthEndpoint {
    private authSuccessEmitter = new Emitter<AuthSuccessEvent>();
    onDidAuthenticate = this.authSuccessEmitter.event;

    shouldActivate(): boolean {
        return true; // 或者读配置决定是否启用
    }

    onStart(app: Express, hostname: string, port: number): void {
        app.post('/api/login/my-auth', async (req, res) => {
            // 验证逻辑...
            await this.authSuccessEmitter.fire({
                token: req.body.token,
                userInfo: { name: 'user', authProvider: 'MyAuth' }
            });
            res.send('Ok');
        });
    }
}
```

**第二步：在 `inversify-module.ts` 里注册**

```typescript
// packages/open-collaboration-server/src/inversify-module.ts
import { MyAuthEndpoint } from './auth-endpoints/my-auth-endpoint.js';

// 找到 AuthEndpoint 的 multiInject 绑定，添加一行：
container.bind(AuthEndpoint).to(MyAuthEndpoint);
```

服务端在 `collaboration-server.ts` 里用 `@multiInject(AuthEndpoint)` 收集所有认证端点，所以只需要绑定，不需要改其他地方。

---

## 场景四：理解并修改光标/选区同步

光标同步走的是 Yjs 的 **awareness** 机制，不是文档同步（`DataUpdate`）。

**数据流：**
```
本地光标移动
  → VS Code selection change 事件
  → collaboration-instance.ts 更新 awareness 状态
  → yjs-provider.ts yjsAwarenessUpdateHandler 编码并广播
  → 服务端中继（sync/awarenessUpdate）
  → 远端 yjs-provider.ts ocpAwarenessUpdateHandler 解码
  → 远端 collaboration-instance.ts rerenderPresence() 渲染装饰
```

**关键文件：**
- `collaboration-instance.ts`：`rerenderPresence()` 方法负责把 awareness 数据渲染成 VS Code 的 `TextEditorDecorationType`
- `yjs-provider.ts`：`yjsAwarenessUpdateHandler` 和 `ocpAwarenessUpdateHandler` 处理编解码

**awareness 数据结构（`ClientAwareness` 类型）：**
```typescript
// packages/open-collaboration-protocol/src/types.ts
interface ClientAwareness {
    peer: string;           // peer ID
    currentFile?: string;   // 当前打开的文件路径
    selection?: {           // 选区
        start: { line: number; character: number };
        end: { line: number; character: number };
    };
    viewColumn?: number;
}
```

**修改光标颜色：** 颜色来自 `package.json` 的 `contributes.colors`，在 `collaboration-instance.ts` 的 `createDecorationType()` 里通过 `userColors` 数组循环分配。

---

## 调试技巧

**服务端日志：**
```bash
OCT_ACTIVATE_SIMPLE_LOGIN=true npm run start:dev
```
服务端会打印每个 peer 的连接、房间创建/加入、消息中继等日志。

**VS Code 扩展调试：**
1. 在 `packages/open-collaboration-vscode` 目录下按 `F5`
2. 在 `collaboration-instance.ts` 的关键方法上打断点
3. 用 VS Code 的 Debug Console 查看变量

**网络层调试：**
在 `yjs-provider.ts` 的 `yjsUpdateHandler` 和 `ocpDataUpdateHandler` 里加 `console.log`，可以看到每次文档更新的触发和接收。

**最小复现路径：**
遇到同步问题时，先用 `host + 一个 guest` 的最小配置复现，再加人。大多数同步 bug 在两人场景下就能稳定复现。
