# 源码参考手册

这是一份按模块组织的源码讲解索引，需要改某一块代码时来查，不需要通读。日常编码基本交给 AI，你只要大概知道各个包负责什么，需要深入时，可以把对应的页面交给 AI，让它带着你对照源码看。

先读 [项目概览](/collaboration_tools/project_overview.md) 和 [技术栈与架构](/collaboration_tools/技术栈与架构.md)，再按下面的分类找到你需要的页面。

## 贯穿多个模块的主题

- [一次协作会话的源码走读](/collaboration_tools/session_code_walkthrough.md)：按创建、加入、同步的顺序串起关键源码
- [手动启动与加入机制](/collaboration_tools/manual_start_and_join.md)：分步启动，以及加入超时与审批的原理
- [数据模型与状态同步](/collaboration_tools/数据模型与状态同步.md)
- [二次开发指南](/collaboration_tools/secondary_dev_guide.md)
- [安全与认证](/collaboration_tools/安全与认证/安全与认证.md)：包含 [认证机制](/collaboration_tools/安全与认证/认证机制.md)、[简易登录](/collaboration_tools/安全与认证/简易登录.md) 和 [通信安全](/collaboration_tools/安全与认证/通信安全.md)

## 按模块查

### open-collaboration-agent

- [open-collaboration-agent模块](/collaboration_tools/核心模块详解/open-collaboration-agent模块.md)

### open-collaboration-yjs

- [open-collaboration-yjs模块](/collaboration_tools/核心模块详解/open-collaboration-yjs模块.md)

### open-collaboration-monaco

- [open-collaboration-monaco模块](/collaboration_tools/核心模块详解/open-collaboration-monaco模块.md)

### open-collaboration-vscode

- [open-collaboration-vscode模块](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/open-collaboration-vscode模块.md)
- [UI集成与命令系统](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/UI集成与命令系统.md)
- [文件系统代理与远程访问](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/文件系统代理与远程访问.md)
- [核心服务与状态管理](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/核心服务与状态管理.md)
- [扩展架构与生命周期](/collaboration_tools/核心模块详解/open-collaboration-vscode模块/扩展架构与生命周期.md)

### open-collaboration-server

- [open-collaboration-server模块](/collaboration_tools/核心模块详解/open-collaboration-server模块/open-collaboration-server模块.md)
- [服务架构与启动流程](/collaboration_tools/核心模块详解/open-collaboration-server模块/服务架构与启动流程.md)
- [消息中继与广播系统](/collaboration_tools/核心模块详解/open-collaboration-server模块/消息中继与广播系统.md)
- 房间与用户管理：[房间与用户管理机制](/collaboration_tools/核心模块详解/open-collaboration-server模块/房间与用户管理机制/房间与用户管理机制.md)、[房间生命周期管理](/collaboration_tools/核心模块详解/open-collaboration-server模块/房间与用户管理机制/房间生命周期管理.md)、[用户与对等节点管理](/collaboration_tools/核心模块详解/open-collaboration-server模块/房间与用户管理机制/用户与对等节点管理.md)
- 认证与安全接入：[认证与安全接入](/collaboration_tools/核心模块详解/open-collaboration-server模块/认证与安全接入/认证与安全接入.md)、[简易登录机制](/collaboration_tools/核心模块详解/open-collaboration-server模块/认证与安全接入/简易登录机制.md)、[OAuth集成认证](/collaboration_tools/核心模块详解/open-collaboration-server模块/认证与安全接入/OAuth集成认证.md)、[Keycloak单点登录](/collaboration_tools/核心模块详解/open-collaboration-server模块/认证与安全接入/Keycloak单点登录.md)

### open-collaboration-protocol

- [open-collaboration-protocol模块](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/open-collaboration-protocol模块.md)
- [协议方法对照表](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/协议方法对照表.md)
- [核心连接机制](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/核心连接机制.md)
- 消息编码与处理：[消息编码与处理机制](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/消息编码与处理机制/消息编码与处理机制.md)、[消息编码机制](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/消息编码与处理机制/消息编码机制.md)、[消息压缩策略](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/消息编码与处理机制/消息压缩策略.md)、[消息加密与安全传输](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/消息编码与处理机制/消息加密与安全传输.md)
- [传输层适配实现](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/传输层适配实现/传输层适配实现.md)
- [工具组件与辅助功能](/collaboration_tools/核心模块详解/open-collaboration-protocol模块/工具组件与辅助功能.md)

## 注意

这些页面写于项目早期，部分内容可能与现在的源码不一致，尤其是 Agent 相关的功能是后来才加入的。引用具体实现时，请以 `collaboration-tools/` 下的源码为准。
