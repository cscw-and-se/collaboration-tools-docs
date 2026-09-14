# 第六讲：版本控制 Git (Version Control)

> MIT《Missing Semester》学习笔记，整理于 2026-09。以下为学习整理，非官方讲义。

## 课程链接

- 英文讲义：https://missing.csail.mit.edu/2020/version-control/
- 中文译版：https://missing-semester-cn.github.io/2020/version-control/

## 1. Git 的数据模型

Git 保存的是一系列**快照**，而不是文件之间的差异：

- **blob**：文件内容。
- **tree**：目录，把文件名映射到 blob 或子 tree。
- **commit**：一次快照，指向它的父 commit。
- 历史因此是一张**有向无环图（DAG）**。
- 所有对象（blob、tree、commit）都用 **SHA-1 哈希**唯一标识。
- `HEAD`、`master` 等是**引用**，也就是指向某个 commit 哈希的别名。

理解了这套模型，大部分 Git 操作都能自己推理出来，出问题时也知道该从哪一步回退。

## 2. 暂存区

提交之前先经过暂存区，由你决定这次提交包含哪些改动：

```bash
git add file.txt
git status
git commit -m "fix parser bug"
```

好处是可以把不相关的改动拆成多次提交，保持每个提交都干净。

## 3. 常用操作

```bash
git checkout -b feature_x   # 新建并切换到新分支
git stash                   # 暂时保存当前改动
git stash pop               # 恢复暂存的改动
git commit --amend          # 修改最近一次提交
git bisect                  # 二分查找引入 bug 的提交
```

`git bisect` 的思路是：标出一个已知的好版本和一个坏版本，Git 自动切到中间版本让你验证，几次就能定位到具体是哪次提交引入的问题。

## 4. 团队协作

Gitflow 是一种常见约定：

- **`main`**：存放正式发布的历史。
- **`develop`**：功能集成的分支。
- **`hotfix`**：从 `main` 拉出，修复紧急问题后同时合回 `main` 和 `develop`。

## 5. 一些习惯

- 冲突是并行开发的常态，用 `git mergetool` 等工具正常处理即可。
- 提交信息写清“为什么”，回溯时比“改了什么”更有用。
- 用 `.gitignore` 排除 `.DS_Store`、编译产物等，避免仓库变臃肿。

## 小结

Git 的核心是快照和 DAG。把这层理解清楚，就不会再一遇到问题就删掉重 clone。
