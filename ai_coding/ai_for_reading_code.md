# 用 AI 辅助阅读源码

> 这篇文章针对的场景：你拿到一个陌生的代码库，不知道从哪里开始读。

对于研究组的新成员来说，第一次打开 collaboration-tools 的代码可能会有点懵——6 个包、几十个文件、互相依赖。这种时候，AI 是你最好的向导。

## 从入口文件开始，问调用链

不要试图一次性理解整个项目。找到入口文件，让 AI 解释它做了什么，然后顺着调用链往下走。

**示例（以 Claude Code 为例，其他工具同理）：**

```
@packages/open-collaboration-vscode/src/extension.ts
这个文件是 VS Code 扩展的入口，帮我解释它的 activate() 函数做了什么，
以及它初始化了哪些关键服务？
```

AI 会告诉你：`activate()` 里用 InversifyJS 创建了 DI 容器，绑定了哪些服务，然后调用了哪个函数开始监听命令。你再顺着这条线往下问。

**关键原则：一次只问一个函数或一个概念，不要一次性把整个文件扔给 AI 问"这是什么"。**

## 用 @ 把相关文件一起喂给 AI

当你想理解几个文件之间的关系时，把它们一起提供给 AI：

```
@packages/open-collaboration-vscode/src/collaboration-instance.ts
@packages/open-collaboration-yjs/src/yjs-provider.ts
这两个文件是怎么配合工作的？当用户在编辑器里打字时，数据是怎么从
collaboration-instance 流向 yjs-provider 的？
```

单独问其中一个文件，AI 只能给你局部视角。把相关文件一起提供，它能给你完整的数据流。

## 让 AI 画出数据流

文字描述有时候不如图直观。让 AI 用 mermaid 画出执行流程：

```
@packages/open-collaboration-vscode/src/collaboration-instance.ts
用 mermaid sequence diagram 画出：当 host 在编辑器里输入一个字符时，
这个变更是怎么同步到 guest 那边的？
```

得到的 mermaid 图可以直接粘贴到 Docsify 文档里渲染。

## 苏格拉底式：先问"是什么"，再问"为什么"

很多人拿到代码直接问"这段代码怎么改"，但其实应该先问清楚它在做什么。

**两步走：**

第一步，问"是什么"：
```
@packages/open-collaboration-server/src/room/room-manager.ts
RoomManager 的 joinRoom() 方法做了什么？用一句话概括。
```

第二步，问"为什么"：
```
为什么 joinRoom() 里要设置一个 300 秒的超时？如果不设置会怎样？
```

第二个问题往往能让你理解设计决策，而不只是代码行为。这对后续修改代码非常有帮助。

## 让 AI 帮你找"这个功能在哪里实现"

当你知道有某个功能，但不知道代码在哪里时：

```
@packages/open-collaboration-vscode/src/
光标同步功能（就是你能看到其他人的光标位置）是在哪个文件里实现的？
大概是怎么工作的？
```

AI 会根据目录结构和文件名推断，给你一个起点。然后你再用 @ 把它指出的文件读进来确认。

## 一个实际的阅读路径（以 collaboration-tools 为例）

如果你是第一次读这个项目，建议按这个顺序：

1. 先读 [项目是怎么运行的](../collaboration_tools/how_it_works.md)，跑通一次端到端
2. 然后问 AI：`@packages/open-collaboration-vscode/src/extension.ts` 的 activate() 做了什么
3. 顺着 `CollaborationInstance` 往下，问它是怎么初始化 Yjs 的
4. 再问 `YjsProvider` 是怎么把 Yjs 变更发给服务端的
5. 最后看服务端的 `message-relay.ts`，理解服务器只是透明中继

每一步都让 AI 解释，遇到不懂的概念立刻追问。一两个小时就能建立起对整个系统的基本认知。

## 关于工具

本文以 Claude Code 的 `@` 语法为例，但核心方法适用于任何 AI 工具：

- **Cursor**：用 `@file` 引用文件，在 Chat 面板提问
- **网页版 Claude/GPT**：手动粘贴代码片段，效果稍差但同样有效
- **GitHub Copilot Chat**：在 VS Code 里直接问，它能感知当前打开的文件

工具会变，但"从入口开始、顺着调用链、一次一个问题"的方法不会变。
