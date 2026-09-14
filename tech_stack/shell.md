# Shell：命令行入门

大学课程里通常会教你 C++、Java、Python 这些用来构建应用的语言，但很少有一门课系统地讲你每天都在用的命令行工具——**Shell**。

> 注意：下面这些命令在 Windows 上可能无法直接使用，请在 Linux、macOS 或 WSL（Windows Subsystem for Linux）环境下操作。

## 为什么要学 Shell

图形界面（GUI）直观、上手快，适合处理日常的简单操作；命令行的优势则在自动化、远程操作和可编程性上。很多开发工具链（Git、Docker、npm、pip）也都以命令行的方式使用，熟练使用它能明显提高日常效率。

- **自动化**：把 100 个文件从 `.jpeg` 改成 `.jpg`，用鼠标可能要点很久，一条命令就能完成。
- **远程服务器**：登录云服务器或实验服务器时，通常只有命令行界面。
- **理解系统**：通过命令行能更直接地接触进程、文件权限、环境变量这些概念。
- **开发工具链**：几乎所有现代开发工具都以命令行为主要接口。

## 交互式 Shell 常用技巧

- **`↑` 上箭头**：调出上一条命令，连续按可以回溯历史。
- **`Tab` 补全**：输入前几个字母后按 `Tab` 自动补全命令或路径；连按两次会列出所有候选。
- **`Ctrl + C`**：中断当前正在运行的程序。
- **`Ctrl + R`**：反向搜索历史命令，输入关键词即可找到匹配的最近命令。
- **`|`（管道）**：把上一条命令的输出作为下一条命令的输入。
  ```bash
  ls -l | grep ".md" | wc -l
  # ls -l 列出文件 -> grep 筛选 .md -> wc -l 统计行数
  ```
- **`>` 和 `>>`（重定向）**：把输出写入文件，`>` 覆盖，`>>` 追加。
  ```bash
  ls > file_list.txt
  ```

## 基础命令：文件与目录

- **`ls`**：列出目录内容。
    - `ls -l`：显示权限、大小、修改时间等详情。
    - `ls -a`：包含以 `.` 开头的隐藏文件。
- **`cd`**：切换目录。
    - `cd my_folder`：进入子目录。
    - `cd ..`：返回上一级。
    - `cd` 或 `cd ~`：回到主目录。
- **`pwd`**：显示当前完整路径。
- **`cat`**：查看文件内容。
    ```bash
    cat README.md
    ```
- **`cp`**：复制文件或目录。
    ```bash
    cp source.txt destination.txt
    cp -r source_dir/ destination_dir/   # -r 递归复制目录
    ```
- **`mv`**：移动或重命名。
    ```bash
    mv old_name.txt new_name.txt   # 重命名
    mv my_file.txt my_folder/      # 移动到目录
    ```
- **`rm`**：删除文件或目录。**没有回收站，请谨慎使用。**
    ```bash
    rm my_file.txt
    rm -r my_folder/   # 删除目录及其内容
    ```

## Shell 脚本

把一串命令写进文本文件，就可以让计算机按顺序自动执行。

```bash
#!/bin/bash

DIR_NAME="my_project"
FILE_NAME="README.md"

if [ ! -d "$DIR_NAME" ]; then
  echo "目录 $DIR_NAME 不存在，正在创建..."
  mkdir $DIR_NAME
fi

cd $DIR_NAME
echo "# $DIR_NAME" > $FILE_NAME

echo "项目初始化完成！"
```

- `#!/bin/bash` 称为 shebang，用来指定脚本使用的解释器。
- 变量通过 `变量名=值` 定义，用 `$变量名` 引用。
- Shell 支持 `if`、`for`、`while` 等控制结构。

## 延伸阅读：MIT The Missing Semester

要系统学习 Shell 和命令行工具，推荐 MIT 的 The Missing Semester。以下四讲最值得先看：

1. **[The Shell](https://missing.csail.mit.edu/2020/course-shell/)**：文件导航、I/O 重定向、基本操作。
2. **[Shell Tools and Scripting](https://missing.csail.mit.edu/2020/shell-tools/)**：`grep`、`sed`、`awk` 等工具与脚本自动化。
3. **[Editors (Vim)](https://missing.csail.mit.edu/2020/editors/)**：服务器上常用的终端编辑器。
    - 打开 Vim 时处于**普通模式**，按 `i` 进入**插入模式**开始输入，按 `Esc` 回到普通模式。
    - `:wq` 保存并退出，`:q!` 不保存强制退出。
    - 终端里运行 `vimtutor` 有官方自带的交互式教程。
4. **[The Command-Line Environment](https://missing.csail.mit.edu/2020/command-line/)**：配置 shell 环境，理解进程和环境变量。

命令行初期确实不如图形界面直观，但它带来的自动化能力回报很高。
