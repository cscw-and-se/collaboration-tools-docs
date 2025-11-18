# Git 团队协作流程

当你从独自开发过渡到团队协作，仅仅会用 `git commit` 和 `git push` 是远远不够的。一套清晰、一致的协作流程，是保证项目高效推进、避免代码混乱和冲突的“交通规则”。

**核心理念：** 代码仓库是团队共享的、最重要的资产。我们的一切规范都旨在让它保持**清晰、可追溯、易于协作**。

## 1\. 分支管理策略 (Branching Strategy)

我们采用业界流行且简单高效的 **GitHub Flow** 模型作为基础。

  * **`main` 分支 (主分支):**

      * 这是**最稳定**的分支，代表了可以随时演示或部署的“正式版本”。
      * **规范：**
          * 在前期快速开发时，可以允许直接向 `main` 分支推送（push）代码，或者直接向 `main` 分支合并（merge）代码。
          * 在项目已经进入稳定阶段后，**严禁**直接向 `main` 分支推送（push）代码，或者直接向 `main` 分支合并（merge）代码。
          * 所有对 `main` 分支的变更，都必须通过“拉取请求”（Pull Request）并经过至少一人审查（Code Review）后方可合入。
          * `main` 分支的每一次提交都应该是可随时运行的。

  * **`feature` 分支 (功能分支):**

      * 这是我们日常开发的主要阵地。每当要开发一个新功能或修复一个Bug时，都应该创建一个新的 `feature` 分支。
      * **规范：**
          * **分支命名：** 采用 `类型/简短描述` 的格式，例如：
              * `feature/user-login` (开发用户登录功能)
              * `bugfix/login-button-style` (修复登录按钮样式问题)
              * `refactor/api-service` (重构API服务)
          * **分支起点：** 始终从最新的 `main` 分支创建。
          * **生命周期：** 功能开发完成、合并回 `main` 分支后，应及时删除。

## 2\. 日常开发工作流 (Daily Development Workflow)

每个团队成员每天都应遵循以下标准流程：

**第 1 步：开始新任务**

```bash
# 1.1. 切换到主分支
git checkout main

# 1.2. 拉取远程最新代码，确保你的 main 分支是全球最新的
git pull origin main
```

**第 2 步：创建你的功能分支**

```bash
# 从最新的 main 分支创建并切换到你的新功能分支
git checkout -b feature/your-feature-name
```

**第 3 步：编码与本地提交**
在新分支上进行代码编写。遵循“小步提交”原则，完成一个小的、独立的功能点就进行一次 `git commit`。并编写规范的 Commit 信息（见下一节）。

**第 4 步：保持分支与主干同步（关键步骤！）**
在你的功能开发期间，`main` 分支可能已经被其他同事合入了新的代码。为了尽早发现并解决潜在的冲突，你需要定期将 `main` 的更新同步到你自己的 `feature` 分支。

这里有两种主流方法：`merge` 和 `rebase`。

#### **方法 A: 使用 `merge` (推荐新手使用，更安全)**

这种方法会把 `main` 分支的最新历史作为一个“合并点”加入到你的 `feature` 分支中。

```bash
# 假设你当前在 feature/your-feature-name 分支
git fetch origin # 先拉取远程所有分支的最新信息
git merge origin/main # 将远程的 main 分支合并到你当前的分支
```

  * **优点：** 操作简单、安全直观，完全保留了代码仓库的真实历史记录。
  * **缺点：** 会产生一个额外的“Merge branch 'main' into ...”的合并提交，如果频繁同步，会让分支的历史记录显得驳杂。

#### **方法 B: 使用 `rebase` (推荐熟练者使用，历史更清晰)**

Rebase（变基）会“拔起”你的 `feature` 分支，然后把它“嫁接”到 `main` 分支的最新位置上，让你的分支历史看起来像是从最新的 `main` 开始开发的一样。

```bash
# 假设你当前在 feature/your-feature-name 分支
git fetch origin
git rebase origin/main
# 如果出现冲突，需要手动解决冲突，然后继续执行
# 这里给的是命令行的示例，但是VSCode等IDE有解决冲突的图形化界面
git add .
git rebase --continue
# rebase 完成后，需要强制推送到远程仓库
git push -f origin feature/your-feature-name
```
  * **优点：** 会形成一条非常干净的、线性的提交历史，没有多余的合并记录。
  * **缺点：** 它会**重写**你的提交历史！这可能导致冲突处理更复杂。

> **Rebase 的黄金法则：** **永远不要对一个已经推送到远程、并且有多人正在使用的公共分支进行 `rebase`！** 因为这会给所有协作者带来巨大的麻烦。对于只属于你自己的 `feature` 分支，使用 `rebase` 是安全的。

**团队建议：** 如果团队成员对 Git 还不太熟悉，强烈建议统一使用 `merge` 方法。如果团队追求干净的历史记录，并对 `rebase` 的风险有充分认识，可以约定使用 `rebase`。

**第 5 步：推送分支与创建拉取请求 (PR)**
当功能开发完成、并自测通过后，在 GitHub 或 GitLab 上创建一个从你的 `feature` 分支到 `main` 分支的 Pull Request (PR)。

```bash
# 将本地分支推送到远程仓库，如果远程不存在该分支，-u 会建立关联
git push -u origin feature/your-feature-name
```

**第 6 步：代码审查 (Code Review) 与合并**
在 PR 中指定至少一位审查者，根据审查意见修改代码，直到 PR 被批准（Approve）。最后，由负责人将其合并（Merge）到 `main` 分支，并删除已被合并的远程 `feature` 分支。

## 3\. 关键实践与规范

#### **Commit 信息规范**

我们采用 **Conventional Commits** 规范，格式为：`类型(范围): 简短描述`。

  * **常用类型:** `feat`, `fix`, `docs`, `style`, `refactor`, `test`, `chore`
  * **示例:**
    > `feat(patient): add model highlight button`
    > `fix(backend): resolve user login authentication error`
    > `docs(readme): update project setup instructions`

#### **Pull Request (PR) 规范**

  * **清晰的标题:** 简明扼要地描述这个PR做了什么，例如 `feat(doctor): Implement diagnostic report display page`。
  * **详细的描述:** 这个PR解决了什么问题？具体做了哪些改动？
  * **小而专注:** 尽量保持每个PR的功能单一、代码量不宜过大，方便审查。

#### **代码审查 (Code Review) 文化**

  * **审查者：** 态度友好，对事不对人。多提问（“这里为什么这么做？”）而不是直接下命令。
  * **被审查者：** 对反馈保持开放心态。Code Review 是提升代码质量和个人能力的最佳途径。

## 4\. 推荐的参考资料

1.  **Pro Git (中文版) - 官方Git教程:**

      * [https://git-scm.com/book/zh/v2](https://git-scm.com/book/zh/v2)

2.  **交互式Git学习网站 - Learn Git Branching:**

      * [https://learngitbranching.js.org/](https://learngitbranching.js.org/)

3.  **Conventional Commits 规范官网:**

      * [https://www.conventionalcommits.org/zh-hans/v1.0.0/](https://www.conventionalcommits.org/zh-hans/v1.0.0/)

4.  **GitHub Flow 工作流介绍:**

      * [https://docs.github.com/en/get-started/quickstart/github-flow](https://docs.github.com/en/get-started/quickstart/github-flow)