# 第七讲：调试与性能分析 (Debugging and Profiling)

> MIT《Missing Semester》学习笔记，整理于 2026-09。以下为学习整理，非官方讲义。

## 课程链接

- 英文讲义：https://missing.csail.mit.edu/2020/debugging-profiling/
- 中文译版：https://missing-semester-cn.github.io/2020/debugging-profiling/

## 1. 用调试器观察程序内部

调试不只是加 `print`，而是能在程序运行到某一步时查看它的状态。以 GDB 为例：

```bash
gdb ./program
```

常用操作：

- `apropos <regex>`：搜索命令。GDB 命令很多，这个能帮你找到想要的那条。
- `list`：查看源码；`tui enable`：打开内置的源码界面。
- `info locals`：查看当前函数的所有局部变量。
- `x/16xb <addr>`：以十六进制查看内存内容。
- `break file:line if cond`：条件断点，只在满足条件时中断。
- `watch var`：变量被修改时自动中断。
- `p $.next`：`$` 代表上一次打印的值，适合连续查看链表。

在 `~/.gdbinit` 里加上 `set history save on` 可以保存命令历史，之后用 `Ctrl+R` 搜索以前用过的命令。

## 2. 性能分析

调试保证程序正确，性能分析保证程序高效。原则是先用工具找到瓶颈，再针对性优化，不要凭感觉过早优化。

```bash
perf stat ./program
perf record ./program && perf report
```

重点是找到消耗 CPU 最多的热点函数，或占用内存最多的部分，再决定改哪里。

## 3. 静态检查与日志

- **Linter / 静态分析**：在运行前发现潜在错误和不规范写法，是成本最低的一层防线。
- **日志分析**：在生产环境往往无法使用交互式调试器，这时配合 `grep`、`awk` 分析日志：
  ```bash
  journalctl | grep sshd | grep "Disconnected from" | awk '{print $NF}' | sort | uniq -c | sort -nr
  ```

## 4. 一些习惯

- 把好用的 GDB 配置写进 `.gdbinit` 并提交到仓库，团队可以共享。
- 不要只依赖 IDE，命令行 GDB 在处理远程或嵌入式环境时同样重要。

## 小结

调试是从“猜哪里错了”变成“观测程序实际在做什么”；性能分析是把同样的思路用在效率上。
