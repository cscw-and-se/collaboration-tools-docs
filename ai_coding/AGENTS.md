# 2025 — The Year of Agents

文件名玩了一个小巧思：`AGENTS.md` 除了表明这篇内容是关于 Agent 的，同时也是各大 **Coding Agent**（Claude Code 用 `claude.md`）用来理解代码库的初始化文档。

2025 年被广泛认为是 [**Agent 元年**](https://simonwillison.net/2025/Dec/31/the-year-in-llms/)。但在豆包、Gemini 这些应用已经做得很好用的今天（Gemini 网页版虽然看起来像 Chatbot，底层早已是 Agent 内核），我们为什么还需要关心 Agent 到底是什么？

一个直觉上的回答：**这些产品好用，恰恰是因为 Agent 架构在幕后工作。** 理解它，你才能从"用户"变成"建造者"。

## Why Agent

可能比不上大伙一开始就有一份大厂实习。

我大三下（2025.04）只找到一份海外 AI 初创公司的 remote intern。团队不大，但是跟网上的热梗-“人越多神人越多，人越少神人越少”一样，初创里要么是不喜欢大厂氛围，要么是本身就想做点事的技术人。我称我的mentor为 **牛逼的工程师** ，我跟我的 mentor 聊过一次：

> **我**："我怎么才能成为和你一样强的工程师？网上好像都还在讲 Java 八股文？"
>
> **他**："那些东西都是十几年前的了，xxxxxxxxx。"

Java 的那套工程方法论当然没有过时——扎实的基础本身就是工程师的标配。但回头看，当年 PHP 还是 trending 的时候，一批人选择拥抱 Java，才有了后来互联网十几年的基础设施。Agent 现在所处的位置，和那时的 Java 有一些相似之处。如果你想亲自验证这个判断，去 [GitHub Trending](https://github.com/trending) 看看现在大家在建什么。

站在 2026 年 3 月，趋势已经比一年前清晰多了。

## 称不上roadmap的roadmap

> 这不是"必须按顺序走"的路线图，更像是一组台阶 —— 你可以跳着踩，也可以找到适合自己节奏的入口。

### Step 1：扩大信息源

在动手之前，先确保你的信息来源足够多元。以下是一些经过验证的渠道：

| 渠道 | 特点 | 注意事项 |
|---|---|---|
| [GitHub Trending](https://github.com/trending) | 稳定高质量，按语言/时间筛选 | agent 时代水项目变多了，注意区分 vibe coding 产物和真正有价值的 repo |
| [X.com](https://x.com) / [Reddit](https://reddit.com) | 信息快、覆盖广，很多从业者分享最佳实践 | 鱼龙混杂，需要自己筛选信号 |
| 官方 Blog | OpenAI、Anthropic、Manus 等会发 research 和最佳实践 | 高质量但有宣传倾向，带着批判眼光看 |
| **亲手试用 AI 产品** | 作为计算机学生，从 Coding Agent 切入最直观 | 推荐从 Claude Code、CodeX、Gemini CLI、Cursor、Antigravity 等入手 |

信息源多了，每天要处理的量也会变大。一个有意思的练手思路：用你正在试用的 Coding Agent，以产品经理的口吻描述需求 —— **整合信息源，每天生成一份热点日报发送到邮箱**。这个小项目本身就是一次 Agent 实践。

### Step 2：拆解 Agent 的代码架构

到这一步，建议挑几个有代表性的开源 Agent 项目来读源码。目标不是"看完"，而是理解几个核心问题：

- **Tool/Skill 系统是如何设计的？** 截至 2026.03，Tool/Skill 仍然是拓展 Agent 能力边界的主要方式。
- **Memory 怎么管理？** 短期上下文 vs 长期记忆，不同项目的取舍差异很大。
- **Prompt 如何组织和加载？** 系统提示的结构化设计直接决定了 Agent 的行为质量。
- **多步任务的编排？** Plan → Execute → Verify 的循环是怎么实现的。

以下是一些值得拆解的项目（各有侧重，按需选读）：

| 项目 | 特点 | 适合谁 |
|---|---|---|
| [OpenManus](https://github.com/mannaandpoem/OpenManus) | 极致轻量，代码量少，入门友好 | 刚开始看 Agent 源码的人 |
| [smolagents](https://github.com/huggingface/smolagents) (HuggingFace) | 轻量级 Python Agent 库，code agent 在沙箱里写并执行自己的代码 | 想快速理解 code agent 原理 |
| [LangChain](https://github.com/langchain-ai/langchain) / [LangGraph](https://github.com/langchain-ai/langgraph) | 生态最大，chain/tool/memory 的标准化抽象 | 想了解主流框架的设计范式 |
| [CrewAI](https://github.com/crewAIInc/crewAI) | Multi-Agent 协作、角色分工 | 对多智能体系统感兴趣 |
| [AutoGen](https://github.com/microsoft/autogen) (Microsoft) | 事件驱动的多 Agent 对话框架 | 想深入 multi-agent 架构 |
| [OpenHands](https://github.com/All-Hands-AI/OpenHands) | 计算机操作 Agent（浏览器/OS 交互） | 对 computer-use agent 好奇 |
| [e2b-dev/awesome-ai-agents](https://github.com/e2b-dev/awesome-ai-agents) | Agent 项目/论文/框架的策展列表 | 想广泛了解生态全貌 |
| 各类 Claude 应用 | 当下最活跃的生态，但部分 `openclawd` 是 vibe 出来的代码 | 有一定代码阅读能力后再看，注意辨别代码质量 |

我个人当时（2025.03）只读了openmanus，现在回看有些`过时`。读源码的时候可以试着把学到的东西用在你前面做的"日报 Agent"上 —— 比如把日报生成流程拆成 tool，或者加一个 memory 模块来记住你的信息偏好。

### Step 3：把 Agent 用到日常工作流里

当你对架构有了体感之后，真正高频使用各种 Agent 工具会带来第二层理解。你会开始在使用中"看穿"产品背后的设计：

- 看到 Coding Agent 的 "ask user question" 功能 → 你知道这大概率是一个带 hook 的 tool
- 看到 subagent 在后台运行 → 你推测这可能也是通过 hook 机制创建的
- 看到 Agent 能精准理解你的模糊描述 → 你会想这背后的意图识别是怎么做的

以下是我日常的工具组合（基于 macOS 环境，如果你做 agent 相关的开发，Mac 用起来确实会顺手一些，但 Windows/Linux 同样可以）：

#### 终端：Warp + OpenCode + oh-my-opencode

- [**Warp**](https://www.warp.dev/)：AI-native 的终端模拟器，也可以选择 tmux 等传统方案，个人偏好而已。
- [**OpenCode**](https://github.com/anomalyco/opencode)：开源的终端 AI 编程助手，支持接入多种模型（Claude、GPT、Gemini、本地模型等）。核心卖点是代码和上下文留在本地。
- [**oh-my-opencode**](https://github.com/code-yeongyu/oh-my-opencode)：OpenCode 的社区增强配置包。

**OpenCode 日常使用的一些经验：**

| 操作 | 说明 |
|---|---|
| `opencode init` | 初始化项目，会生成 `AGENTS.md` 让 AI 理解你的代码库，建议提交到 Git |
| Plan mode → Build mode | **先用 Plan mode 分析和规划，再切到 Build mode 执行**。这个习惯能避免很多"AI 改了一堆但方向不对"的情况 |
| `/undo` / `/redo` | 随时回滚 AI 的修改，是安全网 |
| `/compact` | 长对话时用来压缩上下文，避免 token 溢出 |
| `/editor` | 打开外部编辑器写长 prompt，比在终端里一行一行敲舒服 |
| `/export` | 把对话导出为 Markdown，方便存档或复盘 |
| 多 session | 可以并行开多个 session 分别处理前端、后端、测试 **这里就是warp或者tmux的好处** |

#### 编辑器：Zed + 你喜欢的 LLM API

- [**Zed**](https://zed.dev)：AI-native 的编辑器，启动快、资源占用小。在一个 repo 下有多个 monorepo 的情况下尤其好用（某些 IDE （特指基于vscode的cursor，antigravity等）在这种场景下确实会卡）。最近 Zed 推出的 ACP 也被 JetBrains 采纳了。

**Zed 的一些使用经验：**

| 场景 | 建议 |
|---|---|
| 大型 monorepo | Zed 的索引速度明显优于同类 IDE，适合需要频繁跳转的项目 |
| AI 辅助 | 内置 AI 面板支持接入你自己的 API Key，可以在编辑器内直接对话 |
| 日常代码浏览 | 轻量、快速打开、不需要等待项目索引完成 |

#### 更多值得关注的工具

- [**Claude Code**](https://claude.ai/code)：Anthropic 官方 CLI 编程助手，直接在终端里运行。支持 Plan 模式和 `@` 上下文引用。回复简洁直接有技术深度，本文档站本身就有 Claude Code 参与写作。
- [**CodeX**](https://openai.com/codex/)：OpenAI 的 AI Coding Agent，拥有完整的命令执行环境（CLI Harness）。
- [**Antigravity**](https://blog.google/technology/google-deepmind/antigravity/)：Google DeepMind 的 Coding Agent，在 Gemini 生态内集成。
- [**Cursor**](https://www.cursor.com/)：AI-first 编辑器，代码库索引和问答能力强。提供教育优惠。
- [**Gemini CLI**](https://github.com/google/gemini-cli)：Google 的命令行 AI 工具，国内获取门槛相对较低，适合入门。

这些工具都在快速迭代，半年后格局可能又不一样。**核心不是绑定某个工具，而是培养 "Plan → Execute → Verify" 的工作习惯。**

### Step 4：从"用得好"到"能造出来"

高频使用之后，你可能已经对这些 Agent 的内部设计有了不少直觉。这时可以尝试把前面做的"日报 Agent"升级，或者从头启动一个新的 Agent 项目。一些可以进一步探索的方向：

- **评测与 Benchmark**：挑出 badcase 建立评测集，写一个 benchmark 框架来量化 Agent 的效果。
- **Multi-Agent 编排**：前置意图识别 → 任务分配 → 执行 → 后置 Validator，这套流程在生产环境中很常见。
- **Memory 系统深入**：短期上下文窗口管理、长期记忆检索（近期 memory 系统是社区热点）。
- **Prompt 工程化**：系统提示的版本管理、A/B 测试、自动优化。

一个实际可用于生产的 Agent 远比 demo 项目复杂，但走到这一步，你的 Agent 方向简历已经有东西可写了，你日常写的一个生产力工具竟然可以写进简历。

## 一些个人的理解

### 关于"被 AI 取代"

Transformer 架构的讲解现在遍地都是，推荐感兴趣的人去看看原理。但一个值得冷静思考的点：

**Transformer 本质上是一个基于概率分布的序列建模器。** 它能做到令人惊叹的事情，但距离真正的 AGI（通用人工智能）还有可观的距离。当前 AI 擅长的是"在训练分布内做高质量的模式匹配和生成"，但面对需要深度推理、物理直觉、持续学习的场景，它仍然力不从心。

所以，与其担心"被取代"，不如思考：**在 AI 能力爆发的时代，和 AI 协作的人相比不和 AI 协作的人，生产力差距正在指数级拉大。** 这才是真正的竞争维度。

### 关于注意力

有趣的是，在 LLM 内部，我们花大量精力优化 **注意力机制（Attention）**—— 让模型的注意力不偏离关键信息，减少幻觉。

在 LLM 外部 —— 也就是我们自己的生活里 —— 同样面临注意力的挑战。Agent 时代的信息密度比以往任何时候都高：GitHub 上可能一半的新 repo 是 agent 随手 vibe 出来的；X 上的推文可能也是 bot 批量生成的；各种"AI 革命"的标题党层出不穷。

**在信息过载的环境中保持专注和判断力，可能是这个时代最被低估的技术技能。** 具体到日常：

- 建立自己的信息筛选机制（比如前面提到的日报 Agent）
- 对所有"看起来很厉害"的东西保持适度怀疑，亲手验证
- 给自己留出不被打断的深度工作时间

### 关于 2026 年的趋势

2026 年的主题似乎正在围绕 **AI Harness** —— 也就是"如何真正驾驭 AI，让它在实际工作中产生稳定价值"。几个代表性的方向：

- **Memory 系统成为基础设施**：从简单的对话历史到结构化的知识图谱，Agent 的记忆能力正在从"有没有"变成"好不好"。各种 memory 架构（向量数据库、图数据库、混合检索）百花齐放。
- **Claude 应用生态爆发**：Anthropic 的 MCP（Model Context Protocol）成为开放标准后，围绕 Claude 的第三方应用迅速增长。各类 `clawd` 应用层出不穷，形成了类似 App Store 的生态雏形。
- **Tool Use 走向标准化**：从各家自定义 function calling 到 MCP、ACP 等开放协议，Agent 调用外部工具的方式正在标准化。这意味着工具生态的碎片化问题正在逐步解决。
- **从 Vibe Coding 到可靠交付**：2025 年的 "vibe coding"（让 AI 随意生成代码，差不多能跑就行）正在向更成熟的方向演进 —— 结构化 prompt、自动化测试、CI/CD 集成、代码审查流程。行业开始认真对待 AI 生成代码的质量问题。

---

