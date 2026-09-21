---
name: route-agent
description: 根据 ticket 内容和注册表自动选择 runtime、adapter 与 agent。读取 .workflow/runtimes.json 和 .workflow/agents.json，不硬编码环境或 agent 名称。
---

# Route Agent

当 ticket 没有完整路由，或用户想在分派前预检目标时，使用本 skill。输出必须同时包含 `runtime_id`、`adapter_id` 和 `agent_id`，并说明所需能力。

## 设计原则

- **数据驱动**：runtime、adapter 和 agent 都从注册表读取。
- **能力优先**：先确认 adapter 能完成任务所需的能力，再比较 agent 标签和成本。
- **三个维度分离**：agent 代表谁做，runtime 代表在哪里做，adapter 代表通过什么机制做。
- **不隐藏回退**：没有唯一候选时列出候选并询问，不把本机、CodeG 或某个 agent 当成隐含默认。
- **显式优先**：用户本次指定高于 ticket，ticket 高于阶段默认，阶段默认高于动态路由。

## Canonical route

所有新 ticket 和调用参数使用下面的机器格式：

```yaml
phase: execution
runtime_id: auto
adapter_id: auto
agent_id: auto
capabilities_all: [execute]
capabilities_any: []
```

- `phase`：`planning`、`research`、`execution`、`dispatch`、`verification`、`review`、`integration` 或 `release`
- `runtime_id`、`adapter_id`：稳定 ID、`auto` 或显式 `null`
- `agent_id`：稳定 agent ID、`auto` 或 `null`。`null` 表示当前用户会话、脚本或人工执行，不需要注册 agent
- `capabilities_all`：必须全部具备
- `capabilities_any`：非空时至少具备一个

人读的 `Runtime`、`Adapter`、`Agent` 字段只是 canonical key 的展示别名。旧 ticket 的 `runtime`、`adapter`、`agent` 和 `## 🤖 建议 Agent` 可以兼容读取，但新文件不要继续生成旧格式。

## 规则来源（优先级从高到低）

1. 本次调用的显式 `runtime_id`、`adapter_id`、`agent_id`
2. ticket 的 canonical route 字段
3. `.workflow/runtimes.json` 当前 phase 的 `routing.defaults`
4. ticket 所需能力与 runtime adapter 的 `capabilities` 匹配
5. ticket 关键词与 `.workflow/agents.json` 的 `tags` 匹配
6. `cost_tier` 兜底
7. 仍有多个同分候选时询问用户

任何显式值冲突都报告为路由错误。不要悄悄换成另一个 runtime 或 agent。

## Phase 与能力默认值

调用者必须传入 phase。ticket 没写时按调用场景补齐：

| phase | 默认要求 |
|-------|----------|
| `planning` | `capabilities_any: [plan, interactive]` |
| `research` | `capabilities_all: [research]` |
| `execution` | `capabilities_all: [execute]` |
| `dispatch` | `capabilities_any: [dispatch, execute]` |
| `verification` | `capabilities_all: [verify]` |
| `review` | `capabilities_all: [review]` |
| `integration` | `capabilities_any: [integrate, merge]` |
| `release` | `capabilities_all: [execute, verify]` |

后台调研、并发、隔离或委托是额外约束：只有 ticket 或用户明确要求时，才把 `delegation`、`parallel`、`isolated-worktree` 加进 `capabilities_all`。

## 路由算法

### 1. 解析约束

读取 ticket 的 runtime、adapter、agent、phase 和 capabilities：

- `runtime_id` 只限制运行环境
- `adapter_id` 只限制任务机制
- `agent_id` 只限制执行者
- `capabilities_all` 和 `capabilities_any` 描述任务必须具备的执行能力

如果只有旧版 `capabilities: execute, verify`，按逗号拆成 `capabilities_all`。如果要表达二选一，必须改成 `capabilities_any`，不能把“或”写进普通字符串。

### 2. 过滤 runtime 和 adapter

只保留 `status: available` 且满足全部 `capabilities_all`、至少一个 `capabilities_any` 的 adapter。

- 指定 runtime 时，只在该 runtime 内匹配
- 指定 adapter 时，校验它属于指定 runtime，或从注册表反查所属 runtime
- `agent_ids: []` 的 adapter 只允许 `agent_id: null`
- `agent_ids: ["*"]` 的 adapter 接受任意已注册 agent，但 dispatch 前必须做安装或健康检查
- 指定 agent ID 的 adapter 只允许该白名单中的 agent

如果没有候选，输出缺失的能力、agent policy 或健康检查，不要自动降级到另一个环境。

### 3. 匹配 agent

在可达 adapter 上读取 `.workflow/agents.json`：

1. `agent_id: null` 时跳过 agent 匹配，确认 adapter 支持 agentless
2. 明确指定 agent ID 时，校验该 agent 存在且 adapter 可使用
3. `agent_id: auto` 时，关键词命中安全、架构或其它高风险标签时，优先对应 tag 的 `expensive` agent
4. 领域关键词命中 agent tags 时，保留匹配候选
5. 占位词 `待定`、`待评估`、`视情况`、`TBD`、`TODO: 设计`、`placeholder` 出现时，要求 architecture 或相近的高推理标签
6. 仍有多个候选时按 `cost_tier` 选择最便宜者，顺序为 `cheap`、`medium`、`expensive`
7. 成本相同且结果仍不唯一时询问用户

### 4. 选择默认和回退

如果只有一个满足能力的 route，直接使用它。如果多个 route 都满足：

- 当前 phase 有 `routing.defaults` 且目标仍可用，优先使用该目标
- 否则按 runtime 状态、adapter 能力、agent policy 和 agent 成本排序
- 排序后仍相同，列出候选并请用户选择

默认值必须来自注册表，不能从“本机”“服务器”“CodeG”或某个 agent 的名字推断。

## 输出格式

每条 ticket 输出完整目标：

```text
Ticket: [003] implement-search-ranking
Phase: execution
Runtime ID: local
Adapter ID: local-session
Agent ID: general
Capabilities all: execute, verify
Capabilities any: none
理由: spec 已明确排序字段和返回结构；local-session 提供 execute 和 verify；general 命中 service-layer，cost_tier=medium
```

Agentless route：

```text
Ticket: [010] update-docs
Phase: execution
Runtime ID: local
Adapter ID: local-session
Agent ID: null
Capabilities all: execute
Capabilities any: none
理由: 这是当前用户会话中的文档修改，目标 adapter 声明 agentless 支持
```

没有唯一结果：

```text
Ticket: [007] design-auth-interface
状态: blocked
候选:
- codeg / codeg-todos / reasoning
- local / local-session / reasoning
原因: 两个 adapter 都满足 execute，但没有阶段默认，也没有 ticket 级 runtime 选择
需要用户决定: 选择 runtime，或把 routing.defaults.execution 配置为其中一个
```

## 升级路径

实现中发现 ticket 需要的能力、runtime 或 agent 与原路由不符时：

1. 在 ticket 的 canonical route 中写明新的约束
2. 重新运行本 skill，并传入正确 phase
3. 如果没有满足全部约束的 adapter，标记 `blocked`
4. 只有用户允许时，才回退到能力较少的 adapter，并在 dispatch 输出中说明缺少什么

## 独立预检

可以对所有未完成 ticket 做路由预检：

```text
[001] init-db-schema      → local / local-session / general
[002] user-auth-logic     → codeg / codeg-todos / reasoning
[003] update-docs         → local / local-session / null
[004] implement-search    → auto / auto / auto，候选 2 个，等待 execution 默认
```

ID 和显示名都必须从当前注册表读取。文档中的任何示例都不是项目默认值。

## 反模式

- 🚫 把本机或 CodeG 写成固定规划端、执行端
- 🚫 只输出 agent，不输出 runtime 和 adapter
- 🚫 用 agent 名称推断它安装在哪个环境
- 🚫 在本 skill 中写死任何具体 agent 名称
- 🚫 把 `capabilities_all` 和 `capabilities_any` 混成一个逗号字符串
- 🚫 多个候选时静默选择一个没有记录的默认值
- 🚫 adapter 缺少必需能力时假装支持隔离、并发或 Merge
- 🚫 用空 `agent_ids` 表示任意 agent

## 与其它 skill 的衔接

- **`/setup-agents`** 管 agent ID、role、tags 和 cost_tier
- **`/setup-runtimes`** 管 runtime、adapter、capabilities、agent policy 和阶段默认
- **`/to-tickets`** 写 ticket 的 canonical route
- **`/dispatch`** 以 `phase: dispatch` 或 `phase: execution` 使用本 skill 的完整输出
- **`/verify`、`/code-review`、`/integrate`** 分别以 `verification`、`review`、`integration` phase 重新检查目标
