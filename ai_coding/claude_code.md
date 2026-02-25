# Claude Code

> 本文写于 2026 年 2 月。AI 工具迭代很快，具体操作可能已经变化，但核心用法不会差太多。

## 这是什么

Anthropic 官方的 CLI 编程助手，直接在终端里运行，能读取整个代码库、执行命令、修改文件。

和 Cursor 这类 AI IDE 不同，Claude Code 是纯命令行的，适合已经习惯终端工作流的开发者。你不需要切换编辑器，在任何项目目录下运行 `claude` 就能开始。

## 安装

```bash
npm install -g @anthropic-ai/claude-code
```

需要 Anthropic API Key，或者通过 Claude.ai Pro 订阅使用（Pro 订阅包含 Claude Code 的使用额度）。

## 核心功能

### Plan 模式：先规划，再执行

这是 Claude Code 最重要的功能之一。在执行复杂任务前，先让 Claude Code 输出一个计划，你确认后再动手。

```
> /plan 帮我重构 collaboration-instance.ts，把 awareness 相关逻辑抽取成独立的类
```

Claude Code 会先分析代码，列出它打算做的每一步，等你确认后才开始修改文件。

适合用 Plan 模式的场景：任务涉及多个文件、不确定影响范围、第一次接触某个模块。

### @ 上下文引用

用 `@` 把相关文件或目录喂给 Claude，回答会更准确：

```
> @packages/open-collaboration-vscode/src/collaboration-instance.ts 这个文件里的 rerenderPresence 方法是做什么的？
```

原则：上下文越精准，回答越准确。不要把整个项目都塞进去，挑相关的文件就好。

### Agentic 模式

Claude Code 可以自主执行多步任务：读文件 → 修改 → 运行测试 → 再修改。你只需要描述目标，它来执行。

```
> 给 open-collaboration-agent 模块加一个 --timeout 参数，默认 30 秒，超时后自动退出
```

它会自己找到 `main.ts`，修改 commander 的参数定义，然后在 `agent.ts` 里加上超时逻辑。

## 为什么推荐

- **文风简洁直接**：不废话，不过度解释，这是我个人很喜欢的一点
- **对代码库的理解比网页版 Claude 强得多**：因为它能直接读文件，不需要你手动粘贴代码
- **Plan 模式让你始终掌握主导权**：不会在你不知情的情况下乱改文件
- **这篇文档就是用 Claude Code 写的**：所以你现在读到的文字，某种程度上也是它的作品

## 与 CodeX 的对比

| | Claude Code | CodeX |
|---|---|---|
| 背后模型 | Claude (Anthropic) | GPT (OpenAI) |
| 运行方式 | 本地 CLI | 云端沙箱 |
| 付费方式 | API Key 或 Pro 订阅 | API Key 或中转站 |
| 适合场景 | 本地项目，需要读写本地文件 | 云端任务，隔离环境执行 |

两者都值得用，我通常根据任务类型选择。

## 注意事项

- 需要稳定的网络环境
- Agentic 模式下会自动修改文件，建议在 git 仓库里使用，方便回滚
- 详细文档：[Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)

---

中转站推荐：https://www.packyapi.com/