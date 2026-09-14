# 第二讲：Shell 工具与脚本 (Shell Tools and Scripting)

> MIT《Missing Semester》学习笔记，整理于 2026-09。以下为学习整理，非官方讲义。

## 课程链接

- 英文讲义：https://missing.csail.mit.edu/2020/shell-tools/
- 中文译版：https://missing-semester-cn.github.io/2020/shell-tools/

## 1. Shell 是一门编程语言

Shell 不只用来敲单条命令，它支持变量、条件、循环和函数。把这些用起来，就能把重复操作写成脚本。

## 2. 脚本基础

脚本一般以 shebang 开头，告诉系统用哪个解释器：

```bash
#!/bin/bash
echo "hello"
```

保存为 `cleanup.sh` 后，`chmod +x cleanup.sh` 即可直接执行。

变量与参数：

```bash
name="world"
echo "hello $name"

echo "$1"    # 第一个参数
echo "$@"    # 所有参数
```

退出码用来判断上一条命令是否成功，`0` 表示成功，非 `0` 表示失败：

```bash
grep foo file.txt
echo $?      # 查看上一条命令的退出码
```

条件与循环：

```bash
if [ -z "$1" ]; then
    echo "usage: $0 <name>"
fi

for f in *.txt; do
    echo "$f"
done
```

## 3. 查找与批量处理

```bash
find . -name "*.py" -mtime -1                 # 24 小时内修改过的 py 文件
rustup toolchain list | grep nightly | xargs rustup toolchain uninstall
```

`find` 支持按名称、时间、权限、大小等条件查找；`xargs` 把上一条命令的输出当作下一条命令的参数，从而批量执行。

## 4. 文本处理三件套

- `grep`：按模式筛选行。
  ```bash
  journalctl | grep sshd | grep "Disconnected from"
  ```
- `sed`：流式替换，不用打开文件就能批量修改。
  ```bash
  sed 's/kubernetes/k3s/g' input.txt
  ```
- `awk`：按列处理结构化文本（如 CSV）。
  ```bash
  awk -F, '{sum += $3} END {print sum}' data.csv
  ```

## 5. 用 Git 管理 dotfiles

在 `$HOME` 下建一个裸仓库，就能像管理代码一样管理 `.bashrc`、`.vimrc` 等配置文件：

```bash
git init --bare $HOME/.cfg
alias config='/usr/bin/git --git-dir=$HOME/.cfg/ --work-tree=$HOME'
config add .vimrc
config commit -m "update vimrc"
```

换机器时，clone 这个仓库即可恢复配置。

## 小结

这一讲最实用的部分，是把重复操作固化成脚本：改一个好写，改一千个就要靠工具。
