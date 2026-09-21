---
name: dispatch
description: 读取 .workflow/tickets/ 中未完成的 ticket，按阻塞边排出执行顺序，为每个 ticket 建一个 git worktree，按当前所在机器从两条推荐里选 agent，并给出工具建议。
disable-model-invocation: true
---

# Dispatch

把 ticket 变成实际执行。整条工作流的执行方式只有一种：**每个 ticket 一个 git worktree**。

不区分本机和服务器。当前在哪台机器上跑，就用那台机器的推荐 agent。两台机器看到的 spec、ticket 和报告通过 git 同步。

## 前提

- `.workflow/tickets/` 中有 ticket 文件
- `.workflow/config.json` 存在，且含 `worktree_dir` 和 `branch_template`
- `.workflow/agents.json` 存在（推荐值为 `manual` 时可缺）
- 工作区干净，或用户确认可以把当前改动先提交

## 路由的两条推荐

每个 ticket 的 route 块长这样：

```yaml
phase: execution
local: claude-code-opus
server: codex-high
```

- 当前机器是 **本机**：取 `local:` 的值
- 当前机器是 **服务器**：取 `server:` 的值
- 值是 `manual`：由当前会话或人工执行，不启动外部 agent
- 值指向不存在的 agent：停止这个 ticket 的分派，报告失效的 id，让用户运行 `/route-agent` 重算

本 skill 不自己算推荐。缺推荐时先调用 `/route-agent`。

## 流程

### 1. 扫描 ticket 状态

读取 `.workflow/tickets/` 下所有 `.md` 文件，收集状态为 `todo` 的 ticket：

```text
<!-- status: todo -->
```

其它状态：`dispatched`、`done`、`blocked`。不要混入 `.workflow/bugs/tickets/`，除非用户明确指定。

### 2. 计算执行顺序

对每个 `todo` ticket，检查 `## 🚧 阻塞` 里的依赖是否全部为 `done`。

排出批次：

- **第 1 批**：所有依赖已完成的 ticket
- **第 2 批**：仅依赖第 1 批的 ticket
- 依次类推

同一批内的 ticket 互不依赖，可以并行开 worktree。批次之间必须串行。

如果存在循环依赖，停止并报告环上的 ticket，不要自己打破环。

### 3. 建 worktree

对每个即将执行的 ticket：

```bash
git worktree add <worktree_dir>/<ticket-id> -b <branch_template 展开>
```

例如 `ticket/003-add-search` 对应 `.worktrees/003/`。

规则：

- 每个 ticket 一个 worktree，一个分支，不共用
- worktree 目录必须已在 `.gitignore` 中
- 已存在同 id 的 worktree 时，复用并检查它是否落后于主分支
- 建 worktree 失败（例如分支已存在）时停止，报告原因，不要强删已有分支

### 4. 选定 agent

按当前机器取 `local:` 或 `server:`：

- 具体 id：在这台机器上用该 agent 执行这个 worktree
- `manual`：当前会话直接做，或提示用户手动执行

一次分派里，同一台机器上的多个 ticket 可以交给不同 agent，也可以交给同一个 agent，只要在各自的 worktree 里完成。

### 5. 给出工具建议

每个 ticket 附一份工具建议。工具建议描述这个 ticket 需要什么能力，不写死具体命令。

| 场景 | 建议 |
|------|------|
| 需要跑测试 | 该项目已有的测试命令，不要引入新的测试框架 |
| 需要类型检查或 lint | 项目已有的配置，不新建配置文件 |
| 需要读大范围源码 | 用检索工具定位，不要全量读入 |
| 需要调外部 API 或库行为 | 先查官方文档或源码，把结论记进 ticket 上下文 |
| 涉及数据库 schema | 先做可回滚的迁移脚本，再改代码 |
| 涉及前端交互 | 改完在浏览器里实际点一遍，不要只看代码 |
| 涉及并发或缓存 | 明确写清一致性和失效策略再动手 |

工具建议不是命令清单。只写这个 ticket 真正需要的能力，别的不要凑。

### 6. 标记为 dispatched

worktree 建好后更新 ticket 顶部：

```text
<!-- status: dispatched host:<local|server> agent:<agent-id> worktree:<path> branch:<branch> at:<ISO8601> -->
```

`<agent-id>` 可以写 `manual`，不能写 `display_name`。时间用当前机器时区的 ISO 8601 格式。

## 结果回收和合并

每个 ticket 完成后，在它自己的 worktree 里 commit，然后合回主分支。这是 dispatch 流程的自然结尾，不需要单独的集成阶段。

顺序：

1. 在 worktree 里跑一遍该 ticket 的验收命令
2. commit，message 用 Conventional Commits，带上 ticket id，例如 `feat(search): add ranking [003]`
3. 按批次顺序合回主分支，一次一个，避免交叉冲突
4. 合并冲突时按下面原则处理
5. 合并完成后删除 worktree 和分支，更新 ticket 状态为 `done`

### 冲突处理

- 先找双方的来源：ticket 的 `## 🎯 任务` 和 spec 里的决策
- 选择能同时满足两边意图的结果
- 两边意图真的矛盾时，把 ticket 标记 `blocked` 并退回用户，不要自动选一边
- 不在没有跑验收命令的情况下声称冲突已解决

跨 ticket 的冲突只有在同一批合并时才出现，因为不同批次本来就是串行的。

## 并发

同一批次的 ticket 可以并行，因为它们的 worktree 互相独立。

跨机器并行时注意：两台机器各自建 worktree、各自 commit，推送后由一方合并。不要让两台机器同时往主分支合并。

## 输出

```markdown
## 🚀 本批分派 <N> 个

### 🧭 当前机器
<local 或 server>

### 🟢 第 1 批
- [002] user-model → claude-code-opus
  - worktree: `.worktrees/002`，分支 `ticket/002-user-model`
  - 工具建议：跑现成的单元测试；不引入新框架
- [004] update-docs → manual
  - worktree: `.worktrees/004`，分支 `ticket/004-update-docs`
  - 工具建议：改完检查链接是否可达

### 🟡 第 2 批（等待第 1 批）
- [005] login-ui，阻塞于 [002]
  - 预排 agent: codex-high

### ⚠️ 未能分派
- [007] auth-interface：`local:` 指向的 `old-agent` 已不在 agents.json，需运行 `/route-agent`

### 🔄 合并计划
- 第 1 批合并顺序：[002] → [004]
- 合并后删除 worktree 和分支
```

## 反模式

- 🚫 多个 ticket 共用一个 worktree
- 🚫 不查阻塞边就一次性全开
- 🚫 把 `local:` 的值用在服务器上，或反过来
- 🚫 工具建议写成与项目无关的通用命令清单
- 🚫 冲突时自动选一边
- 🚫 把 `.worktrees/` 提交进 git
- 🚫 跳过验收命令直接合并

## 下一步

- 每个 ticket 完成后按上面流程合并，然后运行 `/verify`
- 需要整体 diff 审查时运行 `/code-review`
- 需要发版时运行 `/version`
