# 第十讲：杂项 (Potpourri)

> MIT《Missing Semester》学习笔记，整理于 2026-09。以下为学习整理，非官方讲义。

## 课程链接

- 英文讲义：https://missing.csail.mit.edu/2020/potpourri/
- 中文译版：https://missing-semester-cn.github.io/2020/potpourri/

这一讲是系列课程的收尾，把前面没覆盖到的零散技巧凑在一起。它们没有统一主题，但都是实际开发中能省时间的小工具。

## 1. 输入效率

- 把 **Caps Lock 映射为 Esc 或 Ctrl**，减少手指移动，尤其在 Vim 和 shell 快捷键里很实用。
- 用 `Ctrl+R` 或 `fzf` 搜索历史命令，不用狂按方向键。

## 2. 更快的搜索与跳转

- `fd`：`find` 的现代替代，默认忽略 `.git`，支持彩色输出。
  ```bash
  fd config -e json
  ```
- `rg`（ripgrep）：很快的文本搜索。
  ```bash
  rg "connection error"
  ```
- `fasd` / `z`：按访问频率跳转目录。
  ```bash
  z src
  ```

## 3. 批量处理与计算

```bash
rustup toolchain list | grep nightly | xargs rustup toolchain uninstall
cat latency.log | paste -sd+ | bc      # 对一列数字求和
```

`xargs` 把上一条命令的输出转成参数，`bc` 用来做命令行计算。

## 4. 用 Git 管理 dotfiles

和前面几讲相同：用 Git 裸仓库管理配置文件，换机器时 clone 一次就能恢复。

## 5. 远程会话与调试

- `tmux` / `mosh`：网络断开或切换时保持远程会话不中断。
- `git bisect`：在多个提交中二分定位引入问题的那个提交。

## 小结

这些技巧零散，但覆盖了输入、搜索、批处理和远程工作几个高频场景，遇到对应问题时能想起来用即可。
