# 第四讲：数据整理 (Data Wrangling)

> MIT《Missing Semester》学习笔记，整理于 2026-09。以下为学习整理，非官方讲义。

## 课程链接

- 英文讲义：https://missing.csail.mit.edu/2020/data-wrangling/
- 中文译版：https://missing-semester-cn.github.io/2020/data-wrangling/

## 1. 什么是数据整理

数据整理（Data Wrangling）是把数据从一种格式转换、提取成你需要的样子。在命令行里，几乎每次使用管道都涉及数据整理：从一个很大的输入里，提取出少量你真正关心的结果。

## 2. 三个核心工具

- **`grep`**：按模式筛选行。
  ```bash
  grep -i error app.log     # -i 忽略大小写
  ```
- **`sed`**：流式编辑器，最常用的是替换，语法为 `s/REGEX/SUBSTITUTION/g`。
  ```bash
  sed -E 's/foo/bar/g' input.txt
  ```
- **`awk`**：按列处理文本，本身是一门小语言。`$0` 是整行，`$1` 到 `$n` 是各列。

## 3. 正则表达式基础

- `.`：匹配除换行外的任意单个字符。
- `*`：前一个字符出现 0 次或多次；`+`：1 次或多次。
- `^` / `$`：行首 / 行尾。
- `()`：捕获组，配合 `\1`、`\2` 在 `sed` 中引用。

## 4. 几个例子

**统计 SSH 暴力破解的用户名**：

```bash
ssh myserver 'journalctl | grep sshd | grep "Disconnected from"' \
  | sed -E 's/.*user (.*) [^ ]+ port.*$/\1/' \
  | sort | uniq -c | sort -nk1,1 | tail -n10
```

逐段看：先 `grep` 过滤日志，再用 `sed` 和捕获组提取用户名，接着 `sort | uniq -c` 计数，最后按次数排序取前 10。

**批量卸载 Rust nightly 工具链**：

```bash
rustup toolchain list | grep nightly | grep -vE "nightly-x86" \
  | sed 's/-x86.*//' | xargs rustup toolchain uninstall
```

**对 CSV 某一列求和**：

```bash
awk -F, '{sum += $3} END {print sum}' data.csv
```

其中 `-F,` 指定逗号为分隔符；需要更复杂的计算时，可以把结果传给 `bc`。

## 小结

`grep`、`sed`、`awk` 加上管道，可以用一行命令完成过去要手写脚本的数据处理。掌握这三个工具，日常处理日志和表格会轻松很多。
