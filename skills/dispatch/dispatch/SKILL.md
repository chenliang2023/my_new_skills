---
name: dispatch
description: 读取 .workflow/tickets/ 中未完成的 ticket，按阻塞边拓扑顺序，在 CodeG 的 To-dos 面板中创建任务并分派给对应 agent（agent 列表来自 .workflow/agents.json）。在服务器端 CodeG 中运行。
disable-model-invocation: true
---

# Dispatch

在服务器 CodeG 上运行。读取 ticket 文件夹，找出所有阻塞已满足的未完成 ticket，在 CodeG 的 To-dos 面板中创建任务并分派给对应 agent。

## CodeG 的 To-dos 机制（你需要知道的）

CodeG 自带完整的任务管理系统，你不不需要重复造轮子：

- **To-dos 面板**：CodeG 左侧侧边栏的 To-dos 面板是任务队列的入口
- **每个任务一个 worktree**：CodeG 自动为每个任务创建 git worktree（`<project>-task-<id>`，分支 `task/<id>`），多个任务互不干扰
- **并发控制**：每个文件夹有并发限制（默认 2），超出排 Queued
- **自动处理**：开启 "Process automatically" 后任务自动领取
- **Review 流程**：任务完成后进入 review 列，你看 diff 后决定 Merge / Follow up / Complete / Abandon
- **Merge 由 agent 执行**：合并由 agent 在自己的 session 中执行，自动解决冲突，CodeG 用 git 验证是否真的合并了
- **`@` 委托**：在一个会话中用 `@agent` 可以让当前 agent 委托子任务给另一个 agent，子 agent 并行运行

## 前提

- `.workflow/tickets/` 中有 ticket 文件
- `.workflow/config.json` 存在
- `.workflow/agents.json` 存在（agent 注册表，由 `/setup-agents` 管理）
- CodeG 中已安装并配置好 `agents.json` 里的全部 agent
- 项目文件夹已在 CodeG 工作区中打开

## 流程

### 1. 扫描 ticket 状态

读取 `.workflow/tickets/` 下所有 `.md` 文件。每个 ticket 文件顶部有状态行：

```
<!-- status: todo | dispatched | done | blocked -->
```

收集所有 `todo` 状态的 ticket。

### 2. 计算可分派的集合

对每个 todo ticket，检查其 `## 阻塞` 列表中的依赖 ticket 是否全部为 `done`。如果全部 done，该 ticket 可分派。

### 3. 在 CodeG To-dos 中创建任务

对每个可分派的 ticket，在 CodeG 的 To-dos 面板中创建一个新任务：

1. **打开 To-dos 面板**（左侧侧边栏，Automations 下方）
2. **点 New task**
3. **填写任务内容**：
   - **Title**：`[<ticket-id>] <ticket 标题>`
   - **Task description**：把 ticket 文件的完整内容粘贴进去（它是自包含的，包含任务、验收标准、测试 seam、上下文）
   - **Target**：选择项目文件夹（project root，不是 worktree）
   - **Agent**：选择 ticket 的 `## 建议 Agent` 字段指定的 agent
4. **Start** 或开启 **Process automatically** 让 CodeG 调度

### 4. 标记为 dispatched

在 ticket 文件顶部把状态从 `todo` 改为 `dispatched`，附上 agent 名和分派时间：

```
<!-- status: dispatched to:<agent-id> via:codeg-todos at:2026-09-15T10:30:00 -->

> `<agent-id>` 从 `agents.json` 的 `id` 字段取值；保持 kebab-case，不要写 `display_name`。
```

### 5. 利用 CodeG 的并发

- CodeG 的并发限制是 per folder 的（默认 2）
- 如果需要更多并发，在 Task settings → General → Max concurrent tasks 中调高
- 多个不互相依赖的 ticket 同时创建为 To-do，CodeG 会自动并行执行
- 有依赖关系的 ticket 等依赖项 done 后再创建

## 两种分派模式

### 模式 A：To-dos 面板（推荐，适合独立任务）

每个 ticket 作为一个独立的 To-do 任务，各自在独立 worktree 中运行。适合：
- 互不依赖的 ticket
- 需要 agent 全程独立完成的任务
- 你想在完成后逐个 review

操作：在 To-dos 面板逐个创建任务，选择对应 agent。

### 模式 B：`@` 委托（适合主从型任务）

一个主 agent（在当前 `agents.json` 里 `manual_entry: true` 的那个，通常负责业务实现）在单个会话中用 `@` 委托子任务给其它 agent。适合：
- 一个 ticket 需要多 agent 协作完成
- 主 agent 负责业务串联，子 agent 负责具体实现（昂贵档 agent 处理架构决策）
- 你希望在一个会话中看到全部进度

操作：在 manual_entry agent 会话中，输入类似：
```
请实现 [003] implement-search 功能。
@<architecture-agent-id> 请你同时设计 [004] search-dto 的接口（这是架构决策）。
@<frontend-agent-id> 请你同时实现 [005] search-ui 组件。
```

`<architecture-agent-id>` / `<frontend-agent-id>` 用对应 agent 的 `id`（从 `agents.json` 里查，不要写 `display_name`）。

子 agent 各自并行运行，结果汇回主会话。

## 并发分派策略

- 同一批次中，不同 agent 可以同时各拿一个 ticket（CodeG 的并发限制自动控制）
- 如果两个 ticket 都建议同一个 agent，按 ticket id 顺序排队
- CodeG 的 `Process automatically` + 拖拽排序可以自动按你想要的顺序消费

## 输出

分派完成后输出。这是给用户看的回复，按 `/readable-docs` 的「写完之后的回复」保持工整：段落短，每条配 emoji，先给本轮派了几个、还剩几个被阻塞。

```markdown
## 🚀 本轮分派 <N> 个

### ✅ 已创建 To-do

- 🤖 [002] user-model → harness（DeepSeek Harness）
- 🤖 [003] config-dto → pi（Pi）
- 🎨 [006] search-ui → antigravity（Google Antigravity）

> 显示名从 `agents.json` 查；分派 ID 用 agent `id`，与 ticket 的 `## 建议 Agent` 字段对齐。

### ⏳ 等待中（阻塞未满足）

- 🚧 [005] login-ui，阻塞于 [002] [004]

### ⚙️ CodeG 设置

- 🔀 并发限制：3（可在 Task settings 中调整）
- ⚡ Process automatically：已开启
```

## 下一步

- 👀 在 CodeG 的 To-dos 面板中观察任务进度
- 🔍 任务完成后在 Review 列查看 diff
- ✅ Merge 后运行 `/verify` 做全量验证
- 🧐 或在本机拉取后运行 `/code-review`

## CodeG 的 Review 与 Merge（你需要知道的结果回收方式）

CodeG 任务完成后不会自动合并，而是进入 Review 列：
- **Merge**：接受，agent 在自己的 session 中执行合并（解决冲突），CodeG 用 git 验证
- **Follow up**：退回修改，可选 Rework / Keep going / Ask / Double-check
- **Complete**：任务没改文件（只是回答问题或验证），直接标记完成
- **Abandon**：放弃

你不需要写自己的 integrate 逻辑：CodeG 的 Merge 流程已经处理了冲突解决和 git 验证。`/integrate` skill 只在需要跨 ticket 协调时补充使用。