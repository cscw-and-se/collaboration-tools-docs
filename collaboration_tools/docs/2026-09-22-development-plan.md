# 2026-09-22：Collaboration Tools 后续开发计划

本文记录 2026 年 9 月 22 日形成的开发计划，包含 Collaboration Tools 下一阶段的开发范围、来源代码、验证方式和需要确认的设计问题。后续开发进展继续写入 `collaboration_tools/docs/`，文件名使用日期前缀，方便按时间追溯。

## 目标

本轮工作包含三个相互衔接的方向：

1. 将自动启动 Host 与 Guest 的命令整理到当前开发主线上，让本地协作测试可以通过一条命令启动。
2. 以当前上游开源项目为基准，检查代码包版本、协议、服务端、VS Code 扩展和测试的变化，分批同步经过确认的内容。
3. 重新评估 `open-collaboration-agent`。重点处理工具权限、ACP 连接范围和本地工作区访问能力，再确定后续 Agent 方案。

## 当前代码基线

### 目标项目

代码目录：`/Users/baokker/Work/Master/CSCW/智能语义冲突预防-update/collaboration-tools`

调查时的代码状态如下：

- 当前分支为 `dal`，提交为 `d446d3b`。
- `main` 的提交为 `6189536`。
- `dal` 已经包含自动启动器及大量 GreyLock 研究代码，工作区还有未提交的实验记录。后续创建功能分支前，需要先记录这些文件的归属，避免把实验产物混入迁移提交。
- `main` 尚未包含 `oct:collab:open` 及其配套端到端启动流程。

自动启动器的主要代码入口：

- `scripts/open-manual-collaboration.ts`
- `scripts/manual-collaboration-options.ts`
- `packages/open-collaboration-vscode/e2e/manual-collaboration-role.cjs`
- `scripts/extension-host-e2e/macos-visual-capture.ts`
- 根目录 `package.json` 中的 `oct:collab:open`

相关历史提交：

- `e278b60`：加入交互式协作启动器。
- `24bcda0`：等待 Guest 协作工作区准备完成。
- `7de12af`：修正 Guest 项目在资源管理器中的挂载。

当前启动器会执行以下工作：检查项目目录和 VS Code 可执行文件，启动本地服务端，创建 Host 与 Guest 两个独立的 Extension Development Host，自动创建房间并加入，等待双方连接后尝试在 macOS 上排列窗口，终止时关闭客户端和服务端。

### 上游开源项目

开源项目目录：`/Users/baokker/Work/open-collaboration-tools`

调查时上游 `main` 为 `bf8b71f`，版本标签为 `v0.3.1`。与 Agent 直接相关的上游分支包括：

- `origin/jbicker/OCT-ACP-Agent`：ACP Agent 的完整探索分支。
- `origin/jbicker/oct-agent`：较早的 Agent 实现演进分支。
- `origin/jbicker/oct-agent-acp-only`：只保留 ACP 路线的分支。

上游当前 `open-collaboration-agent` 使用 `@agentclientprotocol/sdk`，并配有 `acp-bridge.ts`、`document-operations.ts`、Agent 测试和配置说明。目标项目当前 Agent 仍使用 AI SDK 直接请求模型，版本为 `0.3.0`，两边的依赖、连接方式、文件同步方式和测试结构都需要逐项核对。

目标项目已有一个相关移植分支：`origin/feat/acp-agent-exploration-wty`。该分支还包含 Drift 检测和协作协议调整，不能直接视为上游代码的简单副本。后续需要按文件类别审查，再选择迁移内容。

目标项目还保留了上一轮上游同步分支 `sync-upstream-changes`，当前提交为 `4c09df3`。该分支记录的同步基线是上游 `e60af44`，已经带入聊天、认证、协议和服务端的一批变化。本轮需要以当前上游 `bf8b71f` 重新检查差异，不能只重复上一轮同步提交。

## 阶段一：迁移并验证自动协作启动器

### 分支与范围

计划在评审后创建功能分支：`codex/auto-collaboration-launcher`。分支基线建议从当前正在使用的 `dal` 创建，随后只提交启动器及其必要的构建、测试和文档改动。

本阶段不处理 Agent 和大范围上游同步，避免两个变化来源同时进入一次评审。

### 使用方式

在目标项目根目录执行：

```bash
nvm use 22

set -a
source .env
set +a

npm run build --workspace=packages/open-collaboration-vscode

npm run oct:collab:open -- --workspace "/Users/baokker/Work/Master/CSCW/智能语义冲突预防-update/collaboration-tools/demos/greylock-edit-impact"
```

命令启动后应当出现两个 VS Code 窗口：Host 打开本地项目，Guest 通过协作工作区加入同一个房间。终端会输出服务端地址和房间标识，按 `Ctrl+C` 后应当结束两个客户端与临时服务端进程。

### 验证项目

- 输入一个有效项目目录时，服务端、Host、Guest 都能启动。
- Host 自动创建房间，Guest 自动加入，双方状态都显示已连接。
- Guest 能够看到 Host 的协作工作区，并能观察文件内容同步。
- Host 与 Guest 的编辑可以互相传播，Guest 的 `oct://` 工作区保持可用。
- 关闭任意一个客户端时，启动器能够结束等待并清理其余进程。
- 输入缺少值或不存在的 `--workspace` 时，命令在参数检查处直接报告错误。
- macOS 关闭台前调度时，窗口可以排列；无法排列时，协作连接仍然可以继续。
- 构建、相关单元测试和手动双窗口验收都通过后，才更新 README 与教程。

### 代码与文档更新

验证通过后更新两处教程：

- 目标项目根目录 `README.md`：把 `oct:collab:open` 放在快速开始区域，保留原来的服务端、Host、Guest 手动启动方式。
- 文档项目 `collaboration_tools/how_it_works.md`：新增“通过命令行启动双窗口协作”小节，说明前置条件、命令、窗口角色、停止方式和常见故障；原有手动流程继续保留。

## 阶段二：同步上游代码

### 同步原则

上游同步从 `open-collaboration-tools` 的 `main` `bf8b71f` 开始，目标项目以评审后的功能分支为基线。同步前保存以下信息：两边提交标识、依赖版本、构建结果、测试结果和人工验收结果。

同步按照下面的顺序检查：

1. 根目录 `package.json`、`package-lock.json`、各 workspace 的版本和构建脚本。
2. `open-collaboration-protocol` 的类型、消息和传输层变化。
3. `open-collaboration-server` 的认证、房间、连接和服务进程变化。
4. `open-collaboration-vscode` 的命令、工作区、聊天、扩展生命周期和本地化文件变化。
5. `open-collaboration-agent` 的依赖、连接方式、文件同步、测试和说明文档。

每一组改动都需要单独检查编译错误、公共 API 变化和现有 GreyLock 行为。版本更新要和源码迁移放在同一组验证中，避免只更新锁文件造成依赖状态无法解释。

### 同步验证

同步完成后至少执行：

```bash
nvm use 22
npm ci
npm run build
npm test
npm run oct:collab:open -- --workspace "/Users/baokker/Work/Master/CSCW/智能语义冲突预防-update/collaboration-tools/demos/greylock-edit-impact"
```

还要检查已有 GreyLock 验收命令、聊天功能、简单登录和 Guest 工作区挂载。每个失败结果都记录到对应提交或报告中，再决定是否继续同步下一组内容。

## 阶段三：重新设计 Agent

### 已确认的问题

当前 Agent 的权限判断以 ACP 工具类型和工具名称为依据，默认允许 `read` 与 `edit`。`search`、`execute` 以及适配器未正确填写类型的工具可能被拒绝，因此 `Grep`、`Bash` 这类基础能力无法稳定使用。ACP 适配器还会带来会话创建、权限回调、文件读取、文件写入和响应流处理等多层连接问题。

这些问题需要先用测试和运行日志确认边界，再决定是否继续使用 ACP。

### 可行方向

#### 方向 A：继续使用 ACP，完善能力注册与权限策略

保留 ACP 作为连接协议，在 `open-collaboration-agent` 中建立统一的能力注册表：为读取、编辑、搜索、命令执行分别声明输入、输出、风险级别和审批要求。`Grep` 映射到搜索能力，`Bash` 映射到命令执行能力，适配器未提供标准类型时使用明确的名称映射。

每次工具调用都先经过权限判断，允许用户按单次调用、当前会话或项目配置授予权限。该方向可以保留上游已有的 ACP 兼容能力，工作量集中在权限模型、适配器兼容和测试覆盖。

#### 方向 B：使用本地简单 Agent，直接管理有限工具

移除对 ACP 会话细节的依赖，由本地 Agent 管理模型请求和工具调用。初始只提供读取文件、列出目录、搜索文本、执行受限命令、提交协作修改五类能力。每项能力使用独立的 TypeScript 接口，工具输出采用结构化结果，修改仍然通过协作会话产生可审阅的 diff。

该方向更容易控制 `Grep` 和 `Bash` 的行为，也更容易建立跨平台测试。需要自行处理模型供应商、流式响应、取消请求和权限提示，因此要先确定模型接口与命令执行边界。

#### 方向 C：保留 ACP 连接层，同时在本地增加工具代理

ACP 负责模型会话和消息传递，本地工具代理负责工作区读取、搜索和命令执行。Agent 只向 ACP 适配器暴露统一的工具描述，所有真实操作回到本地代理执行，再把结果返回 ACP。

该方向兼顾 ACP 适配器兼容性与本地权限控制，结构也更清晰。需要维护两套协议之间的请求标识、取消信号、错误信息和权限状态，初期实现范围要严格限定。

### 建议的评审顺序

先对方向 A、B、C 做一个小型能力实验，使用同一组任务比较：读取多文件、搜索符号、执行只读命令、提交单文件修改、提交多文件修改、拒绝危险命令。记录工具调用成功率、用户审批次数、错误信息和协作 diff 是否完整，再选择后续实现方向。

实验必须使用真实本地工作区和真实模型连接。测试代码不使用模拟对象，也不绕过权限判断。

### Agent 必须明确的规则

- Agent 工作目录必须来自当前协作项目，路径解析需要拒绝工作区之外的文件访问。
- `Grep`、目录搜索和文件读取属于读取能力，输出需要限制大小并保留文件路径。
- `Bash` 属于命令执行能力，需要显示完整命令、工作目录、环境变量范围、超时和输出限制。
- 修改文件时只生成协作会话中的可审阅提案，不能绕过用户确认直接改变参与者的本地文件。
- 命令失败、参数错误、权限拒绝和连接中断都要在原位置报告，不能用静默替代行为掩盖错误。
- Host、Guest、Agent 三类身份的权限范围需要分别测试。

## 交付顺序与评审点

1. 评审本文档，确认自动启动器的分支基线和验收标准。
2. 创建 `codex/auto-collaboration-launcher`，迁移启动器并完成双窗口验证。
3. 用户 review 启动器改动，确认后再合入主分支。
4. 创建独立的上游同步分支，按模块同步并完成构建、测试和手动协作验收。
5. 用户 review 上游同步结果，确认后再合入主分支。
6. 选择 Agent 方向，先完成能力实验，再进入 Agent 代码改造。
7. Agent 改造完成后，补充项目文档、Agent 使用说明和故障排查记录。

## 待确认事项

- 自动启动器的首个合并基线使用 `dal`，还是先把 `dal` 中与启动器无关的实验文件整理后再建立分支。
- 双窗口验收是否只针对 macOS，是否需要同时为 Linux 和 Windows 设计启动入口。
- 上游同步是否要包含 `origin/feat/acp-agent-exploration-wty` 中的 Drift 检测与 GreyLock 变化。
- Agent 是否优先选择本地简单 Agent，还是先完成 ACP 能力注册表实验。
- `Bash` 是否允许执行写入命令；允许时，审批范围按单次调用、当前会话还是项目配置保存。
- Agent 产生的多文件修改是否统一生成一个协作提案，还是按文件分别审批。
