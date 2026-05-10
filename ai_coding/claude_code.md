# Claude Code

> 本文写于 2026 年 2 月，并于 2026 年 5 月做过小幅更新。AI 工具迭代很快，具体操作可能继续变化，但核心用法不会差太多。

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

## 为什么现在特别推荐

- **复杂代码库体验好**：它可以直接读仓库、理解上下文、修改文件、运行命令，比网页版 Claude 适合得多。
- **文风简洁直接**：不废话，不过度解释，这是我个人很喜欢的一点。
- **文书和解释质量高**：除了写代码，Claude Code 在写说明文档、整理技术思路、解释复杂调用链时也很顺。
- **Plan 模式让你始终掌握主导权**：复杂任务先规划，再执行，不容易一上来就乱改文件。
- **适合长期协作**：在一个项目里连续用几次之后，你会逐渐形成“描述目标 -> 它读代码 -> 它改 -> 它验证 -> 你 review”的节奏。
- **这篇文档站本身就大量使用 Claude Code / CodeX 辅助完成**：所以你现在读到的文字，某种程度上也是这种工作流的产物。

## 与 CodeX 的对比

| | Claude Code | CodeX |
|---|---|---|
| 背后模型 | Claude (Anthropic) | GPT (OpenAI) |
| 运行方式 | 本地 CLI，贴近当前工作目录 | 本地/云端 Agent 工作流，强调工程执行 |
| 优势 | 语言流畅、解释清楚、复杂上下文体验好 | 性价比高、执行力强、适合大量编码任务 |
| 付费方式 | API Key、Pro 订阅或中转站 | API Key 或中转站 |
| 适合场景 | 本地项目、复杂代码理解、文档和设计说明 | 修 bug、批量改代码、跑测试、写工程文档 |

两者都值得用。我现在通常会把 CodeX 作为高频编码主力，把 Claude Code 用在复杂理解、关键文书、需要更自然表达或更强技术解释的场景里。

## 注意事项

- 需要稳定的网络环境
- Agentic 模式下会自动修改文件，建议在 git 仓库里使用，方便回滚
- 详细文档：[Claude Code 官方文档](https://docs.anthropic.com/en/docs/claude-code)
- 国内网络环境或预算敏感时，可以参考 [AI API 中转站怎么选](/ai_coding/api_relay_station_selection.md)
