# 第一讲：课程概览与 Shell 简介 (Course Overview + The Shell)

> MIT《Missing Semester》学习笔记，整理于 2026-09。以下为学习整理，非官方讲义。

## 课程链接

- 英文讲义：https://missing.csail.mit.edu/2020/course-shell/
- 中文译版：https://missing-semester-cn.github.io/2020/course-shell/

## 1. 这门课想解决什么问题

大学课程通常讲操作系统、编译原理、机器学习这些理论，但很少系统地讲日常真正高频使用的一环：命令行、编辑器、版本控制、调试和自动化。这些工具在学习和工作中要用上成百上千小时，多数人却靠零散搜索自学，效率不高。

这门课的目标就是补齐这一环，让日常开发更顺手。它不教新理论，而是把最常用的工具讲清楚：你知道它们能做什么、该在什么时候用。

## 2. Shell 基础

Shell 是与操作系统交互的文本界面，常见实现有 bash 和 zsh。一些基本命令：

```bash
date            # 当前时间
echo hello      # 打印
pwd             # 当前目录
cd /tmp         # 切换目录，cd - 返回上一个目录
ls -l           # 列出文件及详情
man ls          # 查看命令手册
```

`$PATH` 决定执行命令时去哪些目录查找：

```bash
echo $PATH
which ls        # 查看 ls 实际来自哪个路径
```

## 3. 重定向与管道

Shell 的强项是把小工具组合起来：

```bash
ls | wc -l                    # 统计当前目录文件数
curl -L <URL> > readme.md     # 把网页内容保存到文件
echo hello >> notes.txt       # 追加而不是覆盖
sort < file.txt | uniq -c     # 读取文件、排序、去重计数
```

- `>` 覆盖写入，`>>` 追加，`<` 从文件读取输入。
- `|` 把上一条命令的输出作为下一条命令的输入。

## 4. 在远程服务器上过滤数据

把过滤放在数据所在的一端，避免传输大量无用数据：

```bash
ssh myserver 'journalctl | grep sshd | grep "Disconnected from"' > ssh.log
```

这条命令在服务器上先筛出 SSH 断连日志，只把结果传回本地。

## 5. 别名与历史记录

```bash
alias ll='ls -lh'     # 给常用命令起短名
history | grep ssh    # 搜索历史命令
```

多数 shell 支持用 `Ctrl+R` 交互式搜索历史。

## 小结

从“手动一条条敲”转为“写命令让机器执行”，是这一讲的重点。后面几讲会在这个基础上，分别展开脚本、编辑器、版本控制和调试。
