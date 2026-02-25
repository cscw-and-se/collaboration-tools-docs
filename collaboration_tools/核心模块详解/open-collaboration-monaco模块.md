# open-collaboration-monaco 模块

## 这是什么

Monaco Editor（VS Code 的编辑器内核）的协作插件，让网页版编辑器支持实时多人协作。

如果你要做一个基于 Monaco 的在线 IDE，用这个模块就能加上协作功能：多人同时编辑、光标同步、用户跟随。

## 快速上手

```typescript
import { monacoCollab } from 'open-collaboration-monaco';

const api = monacoCollab({
    serverUrl: 'https://api.open-collab.tools/',
    callbacks: {
        onUserRequestsAccess: async (user) => {
            return confirm(`${user.name} 想加入协作，是否允许？`);
        }
    }
});

// 登录
await api.login();

// 创建房间，返回 room ID
const roomId = await api.createRoom();

// 绑定 Monaco 编辑器实例
api.setEditor(monacoEditorInstance);
```

加入已有房间：

```typescript
await api.joinRoom(roomId);
api.setEditor(monacoEditorInstance);
```

## 关键类

### `monacoCollab()` — 工厂函数

入口，返回 `MonacoCollabApi` 对象。接受 `serverUrl`、`callbacks`、`userToken`、`useCookieAuth` 等配置。

`MonacoCollabApi` 的完整接口（`monaco-api.ts`）：

```typescript
type MonacoCollabApi = {
    createRoom: () => Promise<string | undefined>
    joinRoom: (roomToken: string) => Promise<string | undefined>
    leaveRoom: () => void
    login: () => Promise<string | undefined>
    setEditor: (editor: monaco.editor.IStandaloneCodeEditor) => void
    getUserData: () => Promise<UserData | undefined>
    onUsersChanged: (evt: UsersChangeEvent) => void
    followUser: (id?: string) => void
    getFollowedUser: () => string | undefined
    setFileName: (fileName: string) => void
    // ...
}
```

### `CollaborationInstance` — 核心类

管理 Yjs 文档、awareness、编辑器绑定。每次创建或加入房间都会生成一个实例。

主要职责：

- 把 Monaco 编辑器的变更同步到 Yjs 文档
- 把 Yjs 文档的变更应用到 Monaco 编辑器
- 管理远端用户的光标和选区显示

### `DisposablePeer` — 远端用户

每个远端用户对应一个 `DisposablePeer` 实例，负责注入 CSS 实现光标颜色。颜色是动态生成的，通过 `<style>` 标签注入到页面：

```typescript
// collaboration-peer.ts
const cursorCss = `.${cursorClassName} {
    background-color: ${colorCss} !important;
    border-right: solid 2px;
    // ...
}`;
generateCSS(cursorCss);  // 动态注入 <style> 标签
```

## 与 open-collaboration-vscode 的关系

两者都是编辑器集成，但面向不同场景：

| | open-collaboration-monaco | open-collaboration-vscode |
| --- | --- | --- |
| 目标环境 | 网页（浏览器） | VS Code 扩展 |
| 编辑器 API | Monaco Editor API | VS Code API |
| 典型用途 | 在线 IDE、代码演示平台 | VS Code 协作扩展 |

核心逻辑（Yjs 同步、awareness）两者基本相同，只是编辑器绑定层不同。

## 与其他模块的关系

- 依赖 `open-collaboration-protocol` 连接服务器
- 依赖 `open-collaboration-yjs` 做文档同步
- 直接依赖 `monaco-editor`（peer dependency，由使用方提供）
