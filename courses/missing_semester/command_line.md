# 第五讲：命令行环境 (Command-line Environment)

> MIT《Missing Semester》学习笔记，整理于 2026-09。以下为学习整理，非官方讲义。

## 课程链接

- 英文讲义：https://missing.csail.mit.edu/2020/command-line/
- 中文译版：https://missing-semester-cn.github.io/2020/command-line/

## 1. 终端复用：tmux

远程任务最怕网络断开导致进程中断。`tmux` 让进程在后台持续运行，断线后可以重新连回来。

```bash
tmux            # 新建会话
tmux ls         # 列出所有会话
tmux attach     # 重新连接
```

`tmux` 还支持在一个窗口里分多个面板，方便同时看多个任务。`mosh` 在 tmux 之上进一步改善了断线和网络切换时的体验。

## 2. SSH

用密钥对实现免密登录：

```bash
ssh-keygen -t ed25519
ssh-copy-id myserver      # 把公钥装到服务器
```

多台服务器可以写进 `~/.ssh/config`，之后直接用别名连接：

```text
Host myserver
    HostName 192.168.1.100
    User me
    Port 2222
```

```bash
ssh myserver
```

端口转发可以把远程端口映射到本地，方便调试远程服务：

```bash
ssh -L 8080:localhost:80 myserver   # 本地 8080 转发到服务器的 80
```

## 3. 用 Git 管理 dotfiles

和上一讲相同：用 Git 裸仓库管理 `.bashrc`、`.zshrc` 等配置文件，换机器时 clone 即可恢复。

## 4. 别名与现代工具

```bash
alias gs='git status'
```

一些比传统工具更快的替代：

- `fd`：`find` 的现代替代，默认忽略 `.git`。
- `rg`（ripgrep）：很快的文本搜索工具。
- `fasd` / `z`：按使用频率跳转到常用目录。

## 5. Bash 还是 Zsh

- **Bash**：通用、兼容 POSIX 标准，适合写需要在各种服务器上运行的脚本。
- **Zsh**：交互体验更好，配合 `Oh My Zsh` 有补全、语法高亮和拼写纠正。

日常交互可以用 Zsh，写通用脚本用 Bash，兼顾体验和可移植性。

## 小结

这一讲围绕“把命令行环境布置得更顺手”：让远程会话不掉线、连接更安全、常用操作更短、配置可以随机器迁移。
