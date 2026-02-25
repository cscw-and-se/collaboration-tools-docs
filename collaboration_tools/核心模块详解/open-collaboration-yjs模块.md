# open-collaboration-yjs 模块（同步的“发动机”和“减震器”）

在这套协作系统里，Yjs 是“发动机”，`open-collaboration-yjs` 是“发动机与传动轴之间的连接件”：它把 Yjs 的 update/awareness 变成协议消息发出去，也把协议消息应用回 Yjs。

而 `yjs-normalized-text.ts` 更像“减震器”：它解决了跨平台换行（CRLF/LF）导致的 offset 漂移问题，否则你会看到非常真实的症状：光标/选区位置不准、选区跳来跳去、甚至回写抖动。

这一页按两个真实痛点讲清楚：

1. update/awareness 的双向桥接。
2. 为什么必须做 normalized text（尤其是 VS Code 协作里）。

## 1) update 双向桥接：防回环是第一原则

`OpenCollaborationYjsProvider` 监听两边：

- Yjs 本地 doc 的 `'update'` 事件 -> 编码 -> `connection.sync.dataUpdate(...)`。
- 协议层 `onDataUpdate` -> 解码 -> `syncProtocol.readSyncMessage(...)` -> 应用到本地 doc。

最关键的防回环点是 `origin !== this`：它避免“我自己应用的 update 又被当成本地更新再广播一遍”。

```ts
private ocpDataUpdateHandler(origin: string, update: types.Binary): void {
    const decoder = this.decode(update);
    const encoder = encoding.createEncoder();
    const syncMessageType = syncProtocol.readSyncMessage(decoder, encoder, this.doc, origin);
    if (syncMessageType === syncProtocol.messageYjsSyncStep1) {
        this.connection.sync.dataUpdate(origin, this.encode(encoder));
    }
}

private yjsUpdateHandler(update: Uint8Array, origin: unknown): void {
    if (origin !== this) {
        const encoder = encoding.createEncoder();
        syncProtocol.writeUpdate(encoder, update);
        this.connection.sync.dataUpdate(this.encode(encoder));
    }
}
```

这段代码解决的问题是：“让 update 能双向流动，同时不产生广播回环”。如果你遇到“文本同步偶尔抖一下再追上”，通常不是网络，而是：

- 重连后在做 sync step。
- 或者某处 origin 没被正确传递，导致本地 update 被误判成需要广播。

## 2) awareness：光标/选区为什么要单独一条链路

光标、选区、可见范围这些状态是“协作态”，它不是文件内容的一部分，所以项目里用 awareness 单独同步。

provider 的 awareness 逻辑也很清晰：

- 收到远端 update：`applyAwarenessUpdate(...)`。
- 本地 awareness 变化：编码后广播 `awarenessUpdate(...)`。

```ts
private ocpAwarenessUpdateHandler(origin: string, update: types.Binary): void {
    const decoder = this.decode(update);
    awarenessProtocol.applyAwarenessUpdate(this.awareness, decoding.readVarUint8Array(decoder), origin);
}

private yjsAwarenessUpdateHandler(changed: AwarenessChange): void {
    const changedClients = changed.added.concat(changed.updated).concat(changed.removed);
    const encoder = encoding.createEncoder();
    encoding.writeVarUint8Array(encoder, awarenessProtocol.encodeAwarenessUpdate(this.awareness, changedClients));
    this.connection.sync.awarenessUpdate(this.encode(encoder));
}
```

这段代码解决的问题是：“让光标/选区这种临时状态能低成本同步”。因此你在排查“文本能同步但看不到光标”时，不要在 update 里死磕，应该回到 awareness 这一条链路看：

- 有没有发 awarenessQuery？
- 有没有收/发 awarenessUpdate？

## 3) NormalizedText：为什么跨平台换行会把协作搞崩

Windows 常见 CRLF（`\\r\\n`），macOS/Linux 常见 LF（`\\n`）。如果你用“绝对字符 offset”去同步选区，换行差一个 `\\r`，选区就会越走越偏。

`YjsNormalizedTextDocument` 的核心做法是：

- 内部统一使用 normalized 视图（把换行规范化）。
- 对外（VS Code）仍然用原始文本视图。
- 两者之间靠 offset 映射互相转换。

下面这段代码是“把 normalized offset 映射回 original offset”的关键：

```ts
originalOffset(normalizedOffset: number): number {
    const lineOffset = this.findLineOffset(normalizedOffset, 'normalized').offsets;
    const delta = normalizedOffset - lineOffset.normalized;
    const originalOffset = lineOffset.offset + delta;
    return originalOffset;
}

private async observe(event: Y.YTextEvent, callback: YjsTextDocumentChangedCallback): Promise<void> {
    if (event.transaction.local) {
        return;
    }
    const hasCR = this._text.includes('\\r');
    const changes = YTextChangeDelta.toChanges(event.delta);
    const changeSet: YTextChange[] = [];
    for (const change of changes) {
        changeSet.push({
            start: this.originalOffset(change.start),
            end: this.originalOffset(change.end),
            text: this.normalize(change.text, hasCR),
        });
    }
    // ...
}
```

这段代码解决的问题是：“在不同换行风格下，仍然能保持编辑位置与选区一致”。如果你看到“选区不准”“光标跳”，第一怀疑点就应该是这里（以及 VS Code 端对 relative position 的使用，见 [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)）。

## Summary

- `open-collaboration-yjs` 负责把 update/awareness 双向桥接到协议层，是协作同步的核心发动机。
- `origin !== this` 是防回环的关键条件。
- awareness 是独立链路：光标/选区问题优先查 awareness，不要只盯文本 update。
- `YjsNormalizedTextDocument` 不是“优化”，是跨平台协作的必要条件；没有它，offset 漂移会非常真实地出现。
- 下一步阅读建议：
  - VS Code 端如何生成 relative position/选区： [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)
  - server 的广播中继： [消息中继与广播系统](/collaboration_tools/核心模块详解/open-collaboration-server模块/消息中继与广播系统.md)

