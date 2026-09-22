# 2026-09-22：自动协作启动器进展

## 本次完成内容

目标代码仓库在分支 `codex/auto-collaboration-launcher` 上完成了自动协作启动器的隔离改进，提交为 `1c86526`。

- 协调文件写入项目 `.scratch/` 下的独立运行目录，退出后删除。
- Host 与 Guest 使用带有本次运行标识的独立 VS Code profile，避免多次运行共享状态。
- VS Code profile 使用用户目录下的短路径，规避 macOS IPC socket 路径长度限制。
- `Ctrl+C` 和 `SIGTERM` 监听器会保留到两个 Extension Host 退出后，再移除监听器并清理运行目录。
- 就绪等待阶段把子进程退出转换为显式状态，避免协作已经建立后出现未处理 Promise 拒绝。
- 启动器教程已同步说明运行目录、profile 生命周期和 `OCT_VSCODE_EXECUTABLE_PATH` 配置。
- 目标项目 `README.md` 已在 Quick Start 中加入自动启动命令，并保留服务端、Host、Guest 的手动启动流程。
- `collaboration_tools/how_it_works.md` 已加入自动启动章节，记录前置条件、窗口角色、停止方式和常见检查方式；原有手动流程继续保留。

## 验证结果

已通过：

```bash
npm test -- --run test/manual-collaboration-launcher.test.ts
npm run build
```

使用 `demos/greylock-edit-impact` 执行真实双窗口验收后，Host 自动创建房间，Guest 自动加入，并挂载以下协作工作区：

```text
oct:/greylock-edit-impact/greylock-edit-impact
```

直接执行 TypeScript 启动入口后按 `Ctrl+C`，进程返回码为 `0`，客户端、服务端和本次运行目录均完成清理。`npm run` 包装进程在终端中断时会返回信号状态 `1`，这是 npm 对前台信号的处理结果。

完整 `npm test -- --run` 共通过 124 个测试文件，32 个 GreyLock 固定哈希测试失败。失败内容集中在已有研究文件与冻结哈希不一致，本次启动器改动未触及这些文件。

## 后续工作

等待本阶段 review 后，再决定是否将功能分支合入主分支。完成合入后，再进入上游代码同步阶段。
