# 第八讲：元编程 (Metaprogramming)

> MIT《Missing Semester》学习笔记，整理于 2026-09。以下为学习整理，非官方讲义。

> 这一讲里的“元编程”不是指反射或宏，而是指**围绕编程过程本身的自动化**：构建、依赖管理、测试和持续集成。

## 课程链接

- 英文讲义：https://missing.csail.mit.edu/2020/metaprogramming/
- 中文译版：https://missing-semester-cn.github.io/2020/metaprogramming/

## 1. 构建系统

手动敲一长串编译命令既容易出错又浪费时间。构建系统（如 `make`）会根据文件的修改时间，只重新编译变动的部分。

```makefile
main: main.c utils.c
	gcc -o main main.c utils.c -Iinclude
```

```bash
make          # 按规则构建
make -j4      # 并行构建
```

好处是不用记住参数和顺序，也不会漏掉某一步。

## 2. 依赖管理

项目通常依赖第三方库（Python 的 `pip`、Node.js 的 `npm`、Rust 的 `cargo` 等）。

- **语义化版本**：`Major.Minor.Patch`，用来表达版本变更的兼容性。
- **锁定文件（lock file）**：保证你、队友和 CI 装到完全一致的版本，避免“我本地能跑，你的不行”这类问题。

## 3. 测试与 Lint

- **Linter**：不运行代码，就能发现未定义的变量、可疑写法等问题。
- **自动化测试**：每次修改后自动回归，防止改坏已有功能。

这两者把“人肉检查”变成“每次都自动执行”。

## 4. 持续集成 (CI)

CI 把构建、Lint、测试串成流水线。每次提交代码，服务器自动执行：拉取代码 → 构建 → Lint → 运行测试。问题在合并前就会暴露，而不是等到发布时才出现。

## 小结

这一讲的核心，是把重复、易错的手工步骤固定成流程，交给机器执行和监督。
