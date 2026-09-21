---
name: dispatch
description: 读取 .workflow/tickets/ 中未完成的 ticket，按阻塞边和 canonical route 在具备 dispatch 或 execute 能力的 runtime 中创建任务并分派给对应 agent。
disable-model-invocation: true
---

# Dispatch

读取 ticket，计算可执行集合，按 `.workflow/runtimes.json` 和 `.workflow/agents.json` 选择目标，并使用目标 adapter 创建任务。当前目标可以是本机、CodeG 或其它已注册 runtime。

## 前提

- `.workflow/tickets/` 中有 ticket 文件
- `.workflow/config.json` 存在
- `.workflow/agents.json` 和 `.workflow/runtimes.json` 存在，除非本轮明确使用 agentless route
- 目标 runtime 当前可用

## Adapter 能力

先检查目标 adapter 的 `capabilities`：

- `execute`：可以运行实现任务
- `dispatch`：可以创建或安排子任务
- `parallel`：可以并行运行互不依赖的任务
- `isolated-worktree`：每个任务可使用独立 worktree
- `delegation`：支持在会话中委托给其它 agent
- `review`、`merge`、`preflight`、`verify`：对应的结果回收和验证能力

CodeG 的 `codeg-todos` adapter 通常可以提供 To-dos、worktree、并发、Review/Merge、`@` 委托和 preflight。只有目标注册表实际声明这些能力时，才按 CodeG UI 流程操作。没有这些能力时，使用 adapter 提供的等价机制，或串行执行并把限制写在输出中。

## 流程

### 1. 扫描 ticket 状态

读取 `.workflow/tickets/` 下所有 `.md` 文件，收集状态为 `todo` 的 ticket：

```text
<!-- status: todo -->
```

兼容旧状态：`dispatched`、`done`、`blocked`。不要把 `.workflow/bugs/tickets/` 混入 feature 批次，除非用户明确指定。

### 2. 计算可分派集合

对每个 `todo` ticket，检查 `## 🚧 阻塞` 中的依赖是否全部为 `done`。依赖未完成的 ticket 留在等待列表。

### 3. 解析 canonical route

对每个可分派 ticket：

1. 读取 `phase`、`runtime_id`、`adapter_id`、`agent_id`、`capabilities_all` 和 `capabilities_any`
2. 执行 `/route-agent`，传入 `phase: execution`，或复用本轮已经通过预检的完整结果
3. 校验目标 runtime、adapter、agent policy 都存在且可达
4. `agent_id: null` 时确认 adapter 的 `agent_ids` 是空数组
5. 如果显式 route 冲突或能力不足，标记 `blocked` 并报告原因

不要把缺少的 route 默认为本机、CodeG 或某个 agent。

### 4. 创建任务

按目标 adapter 的能力执行：

#### 具备 `dispatch` 的 adapter

使用 adapter 自己的任务队列。对于 `codeg-todos`：

1. 打开 To-dos 面板
2. 创建任务
3. Title：`[<ticket-id>] <ticket 标题>`
4. Description：粘贴 ticket 完整内容
5. Target：选择项目根目录，不要误选某个已经存在的 worktree
6. 如果 `agent_id` 是具体 ID，选择该 ID 对应的已安装 agent。UI 显示名只用于确认，状态记录仍写 ID
7. `agent_id: null` 时，只有 adapter 明确支持 agentless 任务才可以创建
8. 按 adapter 规则 Start 或开启自动处理

#### 没有 `dispatch` 的 adapter

在当前 runtime 创建一个等价的执行单元：可以是当前会话、脚本任务、CI job 或其它任务队列。必须保留 ticket 完整内容和 canonical route。如果没有 `isolated-worktree`，先确认是否允许在共享工作区串行执行。

### 5. 标记为 dispatched

创建任务成功后，更新 ticket 顶部：

```text
<!-- status: dispatched runtime:<runtime-id> adapter:<adapter-id> agent:<agent-id> at:2026-09-21T10:30:00 -->
```

`<agent-id>` 可以是稳定 ID 或 `null`，不能写 `display_name`。时间使用当前 runtime 的时区和 ISO 8601 格式。

### 6. 并发和顺序

- 只有 adapter 声明 `parallel` 时才并行创建或启动任务
- 只有 adapter 声明 `isolated-worktree` 时，才声称每个任务互不干扰
- 没有并发能力时按拓扑顺序串行
- 同一 agent 的任务是否可以并行由 adapter 和 agent 安装能力决定，不要写死为“每个 agent 一个任务”

## `@` 委托

只有 adapter 声明 `delegation` 时使用 `@` 委托。委托目标必须使用 `.workflow/agents.json` 中的 agent ID，并且在同一 adapter 的 `agent_ids` 约束内可达。

没有 delegation 能力时，把子任务拆成普通 ticket 或任务队列项，不要假设当前会话支持 `@`。

## 输出

```markdown
## 🚀 本轮分派 <N> 个

### ✅ 已创建任务

- [002] user-model → local / local-session / general
- [003] config-dto → codeg / codeg-todos / general
- [004] update-docs → local / local-session / null

### ⏳ 等待中

- [005] login-ui，阻塞于 [002] [004]

### ⚠️ 未能分派

- [007] auth-interface：codeg-todos 缺少 `review` 能力，且没有其它可用候选

### ⚙️ Adapter 能力

- 当前目标：`<runtime-id> / <adapter-id>`
- 使用能力：`parallel`、`isolated-worktree`
- 未提供能力：`<none 或列表>`
```

显示名只用于展示，状态和 ticket 引用使用稳定 ID。

## CodeG adapter 的结果回收

如果目标是 `codeg-todos`，任务完成后通常进入 Review 列：

- **Merge**：接受并合并
- **Follow up**：退回修改
- **Complete**：没有代码变更的任务直接完成
- **Abandon**：放弃

只有目标 adapter 声明 `merge` 时，才把这套 Merge 作为内置流程。否则按 `/integrate` 的通用规则回收结果。

## 下一步

- 查看目标 adapter 的任务状态和 review 结果
- 所有依赖满足并完成后运行 `/verify`
- 需要整体 diff 时，在任意具备 `review` 的 runtime 运行 `/code-review`
