# UI 集成与命令系统（你点的每一下都去哪了）

这套协作工具在 VS Code 里的交互非常克制：状态栏一个入口、命令面板一些命令、侧边栏一个 TreeView。

但对开发者来说，UI 不是装饰，它是调用链的起点。你想改 Share/Join 的行为，第一件事不是去看同步层，而是先回答：

- 用户点了哪个命令？
- 这个命令最终调用了哪个 service？
- service 创建/销毁了哪个 `CollaborationInstance`？

这一页按“用户点一下 -> 命令 -> service”的路径把关键点讲清楚。

## 命令是怎么注册的：`Commands.initialize()`

`Commands.initialize()` 把所有命令一次性挂到 VS Code。下面这段代码你可以当作“命令总入口”：

```ts
initialize(): void {
    this.context.subscriptions.push(
        vscode.commands.registerCommand(OctCommands.FollowPeer, (peer?: PeerWithColor) => this.followService.followPeer(peer?.id)),
        vscode.commands.registerCommand(OctCommands.StopFollowPeer, () => this.followService.unfollowPeer()),
        vscode.commands.registerCommand(OctCommands.Enter, async () => {
            await this.openMainQuickpick();
        }),
        vscode.commands.registerCommand(OctCommands.JoinRoom, async () => {
            await this.roomService.joinRoom();
        }),
        vscode.commands.registerCommand(OctCommands.CreateRoom, async () => {
            await this.roomService.createRoom();
        }),
        vscode.commands.registerCommand(OctCommands.CloseConnection, async () => {
            const instance = CollaborationInstance.Current;
            if (instance) {
                await instance.leave();
                instance.dispose();
                this.contextKeyService.setConnection(undefined);
                if (!instance.host) {
                    // Close the workspace if the user is not the host
                    await vscode.commands.executeCommand(CodeCommands.CloseFolder);
                }
            }
        })
    );
    this.statusService.initialize(OctCommands.Enter);
}
```

这段代码解决的问题是：“把 UI 能触发的动作，稳定地落到 service 上”。

- 上游：用户通过状态栏/命令面板/快捷键触发命令 ID。
- 下游：`roomService.createRoom/joinRoom` 进入会话；`CloseConnection` 负责 leave + dispose +（Guest 侧）关闭工作区。

二次开发时，你更常改的是 `openMainQuickpick*` 的内容（也就是用户看到的那一屏），而不是 `registerCommand` 本身。

## 为什么状态栏点一下就弹面板：`openMainQuickpickOutsideSession()`

扩展把“进入协作”的入口统一收敛到了一个 QuickPick：

- 不在会话时给 Create/Join。
- 在会话里给 Invite/Stop/Configure。

不在会话时的选项生成逻辑很直白：

```ts
private async openMainQuickpickOutsideSession(): Promise<void> {
    const items: Array<QuickPickItem<'join' | 'create'>> = [
        {
            key: 'join',
            label: '$(vm-connect) ' + vscode.l10n.t('Join Collaboration Session'),
            detail: vscode.l10n.t('Join an open collaboration session using an invitation code')
        }
    ];
    if (vscode.workspace.workspaceFolders?.length) {
        items.unshift({
            key: 'create',
            label: '$(add) ' + vscode.l10n.t('Create New Collaboration Session'),
            detail: vscode.l10n.t('Become the host of a new collaboration session in your current workspace')
        });
    }
    const index = await showQuickPick(items, {
        placeholder: vscode.l10n.t('Select Collaboration Option')
    });
    if (index === 'create') {
        await this.roomService.createRoom();
    } else if (index === 'join') {
        await this.roomService.joinRoom();
    }
}
```

这里有一个很“人类”的产品决定：

- 没有工作区就不允许 Create（因为 Host 必须共享一个真实 workspace），但仍然允许 Join。

## 在会话里有哪些操作：Invite/Stop/Configure

在会话里，QuickPick 会根据你是不是 Host 显示不同选项。Host 会多一个“Configure permissions”。

下面这段代码就是“会话内 QuickPick 的行为开关”：

```ts
private async openMainQuickpickInSession(instance: CollaborationInstance): Promise<void> {
    const items: Array<QuickPickItem<'invite' | 'stop' | 'update'>> = [
        {
            key: 'invite',
            label: '$(clippy) ' + vscode.l10n.t('Invite Others (Copy Code)'),
            detail: vscode.l10n.t('Copy the invitation code to the clipboard to share with others')
        }
    ];
    if (instance.host) {
        items.push({
            key: 'update',
            label: '$(gear) ' + vscode.l10n.t('Configure Collaboration Session'),
            detail: vscode.l10n.t('Configure the options and permissions of the current session')
        });
        items.push({
            key: 'stop',
            label: '$(circle-slash) ' + vscode.l10n.t('Stop Collaboration Session'),
            detail: vscode.l10n.t('Stop the collaboration session, stop sharing all content and remove all participants')
        });
    } else {
        items.push({
            key: 'stop',
            label: '$(circle-slash) ' + vscode.l10n.t('Leave Collaboration Session'),
            detail: vscode.l10n.t('Leave the collaboration session, closing the current workspace')
        });
    }
    const result = await showQuickPick(items, {
        placeholder: vscode.l10n.t('Select Collaboration Option')
    });
    if (result === 'invite') {
        await this.inviteCallback(instance);
    } else if (result === 'update') {
        await this.updatePermissions(instance);
    } else if (result === 'stop') {
        await vscode.commands.executeCommand(OctCommands.CloseConnection);
    }
}
```

这段代码的价值在于：把“Host/Guest 角色差异”直接体现在 UI 上。

## TreeView 为什么会自动更新：UI 不靠手动刷新

命令系统之外，还有一个容易忽略的点：UI 更新不是靠“每次操作后手动刷新”，而是事件驱动。

`CollaborationStatusService` 会监听 `roomService.onDidJoinRoom`，把状态栏、TreeView 数据源、context keys 一起切换到“已连接态”。这些细节在 [核心服务与状态管理](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/核心服务与状态管理.md) 里会用代码讲清楚。

## Summary

- UI 的核心入口在 `Commands.initialize()`：命令注册决定了调用链从哪开始。
- QuickPick 是整个扩展的主入口：会话外 Create/Join，会话内 Invite/Stop/Configure。
- 二次开发最常见的改动点：QuickPick 选项、邀请链接生成、权限配置入口、断开连接时的清理行为。
- 下一步阅读建议：
  - Create/Join 的连接与工作区切换： [核心服务与状态管理](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/核心服务与状态管理.md)
  - `oct://` 文件树如何工作： [文件系统代理与远程访问](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/文件系统代理与远程访问.md)

