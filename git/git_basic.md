# Git 基础

在你的编程旅程中，你可能经历过这样的噩幕：为了保留不同版本的功能，你的文件夹里堆满了 `project_v1.zip`, `project_final.zip`, `project_final_but_for_real_this_time.zip`...

这种方式不仅混乱，而且一旦需要找回某个特定版本中的一小段代码，或者想知道某个文件具体在何时被谁修改了，简直无从下手。如果是团队合作，情况会变得更加灾难——你永远不知道别人手上的版本是不是最新的。

为了解决这个问题，天才程序员 Linus Torvalds（也是 Linux 操作系统的创造者）发明了 **Git**。

## Git 的核心理念与模型

在学习具体命令前，我们先理解 Git 是如何思考的。

### 三个区域：工作区、暂存区、仓库

Git 的核心工作流围绕着三个“区域”展开：

1.  **工作区 (Working Directory)**：就是你在电脑上能看到的、包含项目文件的那个文件夹。你所有的日常编辑工作都在这里发生。
2.  **暂存区 (Staging Area / Index)**：这是一个虚拟的“待提交”区域。当你完成了一个小阶段的工作，你可以把希望被记录下来的那些修改添加(add)到暂存区。它就像一个购物篮，让你精确地选择本次要“结账”（提交）的内容。
3.  **本地仓库 (Local Repository)**：这是 Git 存放所有项目历史记录的地方（你看到的 `.git` 文件夹就是它）。当你提交(commit)时，Git 会把暂存区里的所有内容打包，生成一个永久性的“存档点”（快照），并保存在本地仓库里。

它们之间的关系可以用下图表示：

```
+------------------+      git add     +----------------+      git commit      +----------------------+
|                  |----------------->|                |--------------------->|                      |
|   工作区          |                  |     暂存区      |                     |      本地仓库         |
| (Working Directory)|<---------------|   (Staging Area) |<-----------   ----| (Local Repository) |
|                  |    git checkout  |                |     git reset       |                      |
+------------------+                  +----------------+                     +----------------------+
```

### 历史模型：分支与合并

Git 最强大的地方在于它的分支模型。它不像一条直线，而更像一棵不断生长的树。

假设 `o` 代表一次提交，你的项目历史看起来可能像这样：

```
                                      (合并 feature-B)
                                           |
o---o---o-------------------------o--------o  (main 主分支)
         \                       /
          o---o---o-------------o            (feature-A 分支)
                   \
                    o---o---o                (feature-B 分支)
```

  - **main**: 这是你的主干道，通常存放着稳定、可随时发布的代码。
  - **feature-A / feature-B**: 当你想开发一个新功能（比如 feature-A），你可以从主干道上开辟一条新的岔路（分支）。在这条岔路上，你的所有修改都和 `main` 分支以及其他分支互不影响。
  - 合并 (Merge)：当你的功能开发测试完毕，就可以把它合并回 `main` 分支，让主干道也拥有这个新功能。

这种模型让多人并行开发变得安全、简单、高效。

## 准备工作：安装与配置

### 安装 Git

  - **Windows**: 前往 [git-scm.com](https://git-scm.com/download/win) 下载官方安装包 `Git for Windows`。一路点击“下一步”即可，它会附带一个非常好用的命令行工具 `Git Bash`。
  - **macOS**: 最简单的方式是安装 Homebrew ([https://brew.sh/](https://brew.sh/))，然后打开终端运行：
    ```bash
    brew install git
    ```
    或者，你也可以直接在终端运行 `git`，系统会提示你安装苹果的命令行开发者工具，其中就包含了 Git。
  - **Linux**:
    ```bash
    # Debian/Ubuntu
    sudo apt update && sudo apt install git

    # Fedora/CentOS
    sudo yum install git
    ```

### 首次配置

安装后，打开终端（或 Git Bash），设置你的身份信息。这会记录在你的每一次提交中。

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱@example.com"
```

### 配置 SSH 密钥 (强烈推荐)

每次 `push` 代码到 GitHub 时，如果使用 HTTPS 链接，都需要输入用户名和密码，非常繁琐。使用 SSH 密钥，可以让你的电脑和 GitHub 建立起信任关系，免去密码验证。

1.  **检查现有密钥**：

    ```bash
    ls -al ~/.ssh
    ```

    如果看到 `id_rsa.pub` 或 `id_ed25519.pub` 等文件，说明你可能已经有密钥了，可以跳到第 3 步。

2.  **生成新密钥**：

    ```bash
    # 推荐使用更现代、更安全的 Ed25519 算法
    ssh-keygen -t ed25519 -C "你的GitHub邮箱@example.com"
    ```

    按三次回车键，接受默认设置即可。

3.  **将公钥添加到 GitHub**：

      - 首先，复制你的公钥内容。
        ```bash
        # macOS
        pbcopy < ~/.ssh/id_ed25519.pub

        # Windows (Git Bash)
        cat ~/.ssh/id_ed25519.pub | clip

        # Linux
        cat ~/.ssh/id_ed25519.pub 
        # (然后手动复制终端输出的内容)
        ```
      - 然后，登录 GitHub，点击右上角头像 -\> **Settings** -\> 左侧菜单 **SSH and GPG keys** -\> 点击 **New SSH key**。
      - 将你复制的公钥内容粘贴到 “Key” 输入框中，"Title" 可以随便起一个名字（如 "My MacBook"），最后点击 **Add SSH key**。

4.  **测试连接**：

    ```bash
    ssh -T git@github.com
    ```

    如果看到 `Hi [你的用户名]! You've successfully authenticated...` 的提示，恭喜你，配置成功！

## 核心 Git 命令与工具

### 日常开发循环 (每天使用)

```bash
# 1. 查看当前项目状态（强烈建议每次操作前都敲一下）
git status

# 2. 将你希望“存档”的文件添加到暂存区
git add [某个文件的路径]
git add .  # 将所有修改过的文件都添加到暂存区

# 3. 创建一个“存档点”（Commit），并写下清晰的说明
git commit -m "feat: 完成用户登录页面开发"
```

#### 3\. 分支与合并 (并行开发)

```bash
# 1. 创建一个名为'feature-login'的新分支，并立即切换过去
git checkout -b feature-login

# ... 在新分支上进行开发和多次commit ...

# 2. 开发完成后，切换回主分支
git checkout main

# 3. 将'feature-login'分支的修改合并到主分支
git merge feature-login
```

#### 4\. 团队协作 (与远程同步)

```bash
# 1. 在你准备提交自己的代码前，先从远程仓库拉取最新的版本
git pull

# 2. 将你本地的所有“存档”上传到远程仓库
git push
```

## 不止命令行：善用图形化工具 (GUI)

虽然命令行强大且高效，但在初期，图形化界面能帮你更直观地理解 Git 的工作流程。

  * **JetBrains IDEs (IntelliJ, PyCharm, WebStorm等)**
      * JetBrains 全家桶内置了极其强大的 Git 管理面板。你可以在一个视图中完成提交、推送、拉取、切换分支等所有操作。它的**分支可视化**和**冲突解决**工具做得非常出色，推荐使用。
  * **VS Code**
      * VS Code 自带的“源代码管理”面板足以应付基本的 `add` 和 `commit`。但要发挥它的全部潜力，推荐安装以下几个插件：
        1.  **GitLens**: （强烈推荐）当之无愧的“神器”。它能将 `git blame`（查看每一行代码是谁在何时修改的）信息直接内联显示在你的代码旁，还提供了强大的文件历史和分支对比功能。
        2.  **Git Graph**: 将你的提交历史以美观的图形化网络展示出来，分支的创建、合并和流向一目了然。
        3.  **Git History**: 方便地查看一个文件或整个项目的修改日志，并且可以轻松对比不同版本之间的差异。
   * [**GitHub Desktop**](https://desktop.github.com/)
      * GitHub Desktop 是一个非常方便的 Git 图形化工具，它集成了 GitHub 的所有功能，包括提交、拉取、推送、分支管理等。

**总结**：Git 是每个开发者的必备技能。你可以从 GUI 工具入手，直观地感受它的运作方式，但同时也应该逐步学习和熟悉核心的命令行操作，因为这在未来处理复杂问题和自动化任务时会更加灵活和强大。多用，多练，自然熟能生巧。