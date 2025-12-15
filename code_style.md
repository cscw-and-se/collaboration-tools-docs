# 代码规范

这份规范帮助团队在不同项目、不同语言之间保持一致的开发风格。遵循以下原则，就能让代码更易读、易测、易维护。

## 通用原则

- **可读性优先**：变量、函数、文件夹都要取有语义的名字，避免 `tmp`、`data2` 这类模糊命名。
- **单一职责**：一个函数/模块只做一件事，超过 80~100 行的函数通常需要拆分。
- **显式优于隐式**：重要的假设（单位、边界条件、格式）要写在代码或注释里，不要依赖集体记忆。
- **Fail Fast**：在入口处校验参数和配置，早暴露问题，减少排查成本。
- **持续自动化**：Lint、格式化、测试、构建要配置成一键执行，避免手工操作导致的差异。

## TypeScript 编码规范

1. **严格模式**：`tsconfig` 必须启用 `strict`, `noImplicitAny`, `noUnusedLocals` 等选项，禁止在项目里关闭。
2. **类型优先**：
   - API 请求/响应都需要有独立的 `type` 或 `interface`，不要直接使用 `any` 或 `unknown`。
   - 如果某个对象在多个文件中使用，抽离到 `types/` 或 `models/` 目录。
   - 能用联合类型就不要滥用枚举，能使用 `readonly` 就不要写死可变数组。
3. **函数签名明确**：参数超过 3 个时优先使用对象参数，并提供默认值：

```ts
interface FetchOptions {
  retry?: number;
  timeoutMs?: number;
}

async function fetchUserProfile(id: string, { retry = 0, timeoutMs = 5000 }: FetchOptions = {}) {
  // ...
}
```

4. **异常与错误处理**：
   - 异步函数统一使用 `try/catch` 包裹，catch 中抛出的错误需要带上上下文（用户 ID、请求 URL）。
   - 自定义错误类型时继承 `Error` 并补全 `name`, `message`。
5. **模块组织**：
   - 前后端共享类型放在 `shared/` 或单独包里，通过路径别名统一引用。
   - 默认导出仅在真的只有一个主入口时使用，其他情况使用具名导出，方便 IDE 自动补全。

## Git 与 Commit 规范

- **分支策略**：`main` 保持可部署状态；`feature/<topic>` 开发，`fix/<issue>` 修 bug。合并前同步最新 main 并解决冲突。
- **Commit 粒度**：每个 commit 只做一类修改（功能/修复/文档/格式化），避免把不相关的变更堆在一起。
- **Commit Message**：采用简化版 Conventional Commits：
  - `feat: 添加实时协同入口`
  - `fix: 修复 yjs 初始化顺序导致的报错`
  - `docs: 更新 CodeX 使用指南`
  - `chore: bump dependencies`
- **PR/合并**：提交 PR 前至少完成 lint、测试；描述中说明背景、修改点、测试方式。禁止无审查直接推 main。

## 工具与质量保障

- **必备工具**：`ESLint + Prettier`（统一格式和静态检查）、`Husky`（pre-commit/pre-push hook）、`Jest/Vitest`（单元测试）。
- **脚本约定**：Node/TS 项目必须提供 `npm run lint`, `npm run test`, `npm run typecheck` 并在 CI 中执行。
- **文档同步**：发布前更新 README、API 说明或 Changelog；遇到手动步骤（数据迁移、配置修改）在 PR 描述或 `ops.md` 写清楚。
- **代码评审**：CR 重点关注边界条件、容错、并发与安全；发现规范遗漏欢迎直接在本文件补充并提 PR。

遵循以上约定能让我们在不同项目中保持一致的协作体验。如遇到新的技术栈，请参照相同结构扩展本规范。

