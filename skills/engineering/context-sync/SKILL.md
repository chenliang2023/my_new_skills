---
name: context-sync
description: 在多个 runtime 之间同步 .workflow 上下文和 skills，确保 spec、ticket、research、术语表和验证报告使用同一份内容。
---

# Context Sync

`.workflow/` 是跨 runtime 的共享上下文载体。通过 git 同步文档，通过各 runtime 的 skill 安装机制同步 skills。不要把某一环境写成唯一生产者或消费者。

## 同步内容

### `.workflow/` 目录

```text
.workflow/
  config.json          # 路径配置
  agents.json          # Agent 注册表
  runtimes.json        # Runtime、adapter、能力和默认路由
  tickets/             # ticket
  specs/               # spec
  research/            # 调研文档
  handoffs/            # runtime 间交接和 verify 报告
  CONTEXT.md           # 术语表
  ADR/                 # 架构决策记录
```

这些文件必须提交到 git。ticket、spec、research 和 handoff 不能依赖某个机器上的未提交状态。

### Skills

把本仓库的 `skills/` 安装到需要它们的 runtime adapter。安装位置由 adapter 决定：例如本地 skill 目录、CodeG 的共享 skill 存储，或其它任务系统的配置目录。

同步原则：

- planning、dispatch、engineering 和 setup skill 是否可用，由 runtime 的任务能力和用户需要决定
- 不因为某个 agent 是“默认入口”就只给它安装 planning 或 dispatch
- engineering skill 应在所有会实现、验证或 review 的 agent 上可用
- agent 列表从 `.workflow/agents.json` 读取，adapter 可用性从 `.workflow/runtimes.json` 读取

## 交接流程

### Runtime A → Runtime B

1. 确保 `.workflow/` 文档已保存
2. 确认 ticket 的 route 字段是显式可达或 `auto`
3. git add、commit、push
4. Runtime B 拉取同一 commit
5. Runtime B 运行 `/context-sync` 或至少检查 registry、spec、ticket 和 CONTEXT.md
6. 按 `/dispatch`、`/verify` 或 `/code-review` 继续，不需要重新解释上下文

### 执行结果 → Review runtime

1. 代码和 `.workflow/handoffs/` 报告提交到 git
2. 目标 review runtime 拉取同一 commit
3. `/code-review` 读取 ticket、spec、verify report 和 diff
4. review 结论写回 handoff 或 pull request

CodeG 的 Merge、worktree、To-dos 和 preflight 只有在 `codeg-todos` adapter 声明对应能力时才使用。本机或其它 adapter 可以提供等价能力，也可以明确不提供并按串行流程执行。

## 检查清单

- [ ] `.workflow/` 已提交到 git
- [ ] `agents.json` 与 ticket 中的 agent ID 一致
- [ ] `runtimes.json` 中引用的 adapter 和 agent 存在
- [ ] route 所需能力由目标 adapter 声明
- [ ] 本轮新增或改动的文档按 `/readable-docs` 自检
- [ ] 目标 runtime 已拉取同一 commit
