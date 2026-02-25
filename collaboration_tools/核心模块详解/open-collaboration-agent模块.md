# open-collaboration-agent 模块

## 这是什么

一个 CLI 工具，加入协作房间后作为 AI 编程助手参与协作。

你在代码里写一行注释 `// @agent 帮我重构这个函数`，它就会检测到这个注释，把当前文件发给 LLM，然后把 LLM 返回的修改直接写回文件。整个过程对其他协作者可见——他们能实时看到 AI 在修改代码。

## 快速上手

```bash
# 加入一个已有的协作房间，使用 Claude 模型
npx open-collaboration-agent -r <room-id> -m claude-3-5-sonnet-latest

# 使用 GPT 模型
npx open-collaboration-agent -r <room-id> -m gpt-4o

# 连接自托管服务器
npx open-collaboration-agent -r <room-id> -s https://your-server.com
```

需要在 `.env` 文件里配置 API Key：

```env
ANTHROPIC_API_KEY=sk-ant-...
# 或
OPENAI_API_KEY=sk-...
```

## 工作流程

```text
启动 → 登录（浏览器打开登录页）→ 加入房间
  → 通过 Yjs 同步文档
  → 监听 host 的 awareness，跟随 host 当前打开的文件
  → 检测文件中的 // @agent-name 注释
  → 把文件内容 + 注释内容发给 LLM
  → 把 LLM 返回的修改应用回文档
```

连接部分的代码（`agent.ts`）：

```typescript
// 登录
const connectionProvider = new ConnectionProvider(cpOptions);
await connectionProvider.login({ reporter: ... });

// 加入房间
const joinResponse = await connectionProvider.joinRoom({ roomId: options.room });

// 建立连接
const connection = await connectionProvider.connect(joinResponse.roomToken);
```

## 关键文件

| 文件 | 职责 |
| --- | --- |
| `main.ts` | CLI 入口，用 `commander` 解析 `-r`、`-m`、`-s` 参数 |
| `agent.ts` | 主逻辑：连接服务器、加入房间、监听文档变化、触发 LLM |
| `document-sync.ts` | Yjs 文档同步，跟随 host 的 awareness 切换当前文件 |
| `prompt.ts` | LLM 调用，根据模型名前缀自动选择 Anthropic 或 OpenAI |
| `agent-util.ts` | 把 LLM 返回的 diff 定位并应用到文档 |

## LLM 交互细节

`prompt.ts` 里的模型选择逻辑很简单：

```typescript
function getProviderForModel(modelId: string) {
    if (modelId.startsWith('claude-')) return anthropic;
    if (modelId.startsWith('gpt-') || modelId.startsWith('o')) return openai;
    throw new Error(`Unknown model: ${modelId}`);
}
```

LLM 的输出格式有两种（由 system prompt 定义）：

1. **全文替换**：返回整个修改后的文件
2. **局部修改**：只返回修改的代码段，用 `==========` 分隔多个修改区域，每个区域前后各保留至少一行上下文用于定位

`agent-util.ts` 的 `applyChanges()` 负责把这些修改定位到文档中并应用。

## 与其他模块的关系

- 依赖 `open-collaboration-protocol` 连接服务器和加入房间
- 依赖 `open-collaboration-yjs` 同步文档内容和 awareness
- 使用 Vercel AI SDK（`ai` 包）统一调用不同 LLM 提供商
