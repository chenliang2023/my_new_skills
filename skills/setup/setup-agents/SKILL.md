---
name: setup-agents
description: 管理 .workflow/agents.json 里的 agent 注册表。新增 agent、删除 agent、改 agent 名字、角色、标签或成本档。运行时和 adapter 由 /setup-runtimes 管理。
disable-model-invocation: true
---

# Setup Agents

管理项目的 agent 注册表。**用户增删 agent 只动 `.workflow/agents.json`**，其它 skill 会自动读取新表。

agent 注册表是 agent 配置的唯一真相源。它只描述 agent 的稳定身份、角色、能力标签和成本，不描述 agent 在本机、CodeG 或其它运行时中的安装位置。运行时和 adapter 由 `/setup-runtimes` 管理。

## 何时调用

- 首次配置 agent
- 想加一个 agent
- 想删除一个 agent
- 想改 agent 的显示名、角色、成本档或标签
- 需要把 ticket 从一个已停用的 agent 迁移到另一个 agent

不在主流程上每次跑，只在 agent 配置变更时跑。

## 设计原则

- **稳定 ID**：每个 agent 有唯一的 kebab-case `id`。ticket、路由和 adapter 引用都使用它。
- **显示名分离**：`display_name` 只给人和 UI 看，可以修改，不影响 ticket。
- **角色可配置**：`role` 是给人看的角色分类，可选，不参与环境判断；具体路由仍用 `tags`。
- **标签驱动**：`tags` 表示工作类型，例如 `frontend`、`crud`、`architecture`、`security`、`research`、`verify`。
- **成本可比较**：`cost_tier` 为 `cheap`、`medium` 或 `expensive`，只参与动态路由的成本排序。
- **环境无关**：不要在 agent 记录中写 `local`、`codeg`、服务器路径或机器名。
- **没有单一入口**：不再使用 `manual_entry`。阶段默认入口属于 `.workflow/runtimes.json` 的 routing 配置，ticket 也可以覆盖它。

## agents.json 模板

```json
{
  "schema_version": 2,
  "agents": [
    {
      "id": "reasoning",
      "display_name": "Deep Reasoning",
      "role": "architecture and security",
      "cost_tier": "expensive",
      "tags": ["architecture", "security", "design-tradeoff"],
      "description": "处理架构决策和安全敏感任务"
    },
    {
      "id": "general",
      "display_name": "General Builder",
      "role": "general implementation",
      "cost_tier": "medium",
      "tags": ["crud", "service-layer", "api-endpoint", "scaffold", "config", "migration", "verify", "research"],
      "description": "覆盖常规实现、验证和调研"
    },
    {
      "id": "frontend",
      "display_name": "Frontend Builder",
      "role": "frontend",
      "cost_tier": "medium",
      "tags": ["frontend-design", "frontend-impl", "ui", "interaction", "styling"],
      "description": "处理前端、交互和样式"
    }
  ]
}
```

模板中的 ID 和名称都只是示例。项目可以有零个、一个或任意数量的 agent，只要 route 明确使用注册 agent，或目标 adapter 支持 `agent_id: null` 的 agentless 执行。

## 字段说明

| 字段 | 必填 | 含义 |
|------|------|------|
| `schema_version` | 是 | 当前 schema 版本号 |
| `agents` | 是 | agent 数组，顺序无意义 |
| `agents[].id` | 是 | 稳定的 kebab-case ID |
| `agents[].display_name` | 是 | 可变显示名，可含空格或中文 |
| `agents[].role` | 否 | 给人看的角色分类，不替代 tags |
| `agents[].cost_tier` | 是 | `cheap`、`medium` 或 `expensive` |
| `agents[].tags` | 是 | 动态路由使用的标签数组 |
| `agents[].description` | 否 | 一句话角色说明 |

约束：

- `id` 全表唯一，改名要走迁移。
- `tags` 可以重叠。重叠表示多个 agent 都能做，由 route-agent 再按成本和其它条件选择。
- 不要求任何 agent 作为所有阶段的默认入口。
- 运行时 adapter 的 `agent_ids` 只能引用这里存在的 ID、`"*"` 或空数组。
- 空 agent 注册表仍然有效，但只能路由到 `agent_id: null` 的 agentless adapter。

## 流程：新增、删除和修改 agent

### 加 agent

1. 追加唯一的 `id`。
2. 填写 `display_name`、可选 `role`、`cost_tier`、`tags` 和可选描述。
3. 在 `/setup-runtimes` 中把它安装到需要它的 adapter 的 `agent_ids`，或使用 `["*"]` 表示接受任意已注册 agent。
4. 用 `/route-agent` 预检与它相关的 ticket。

### 删 agent

1. 扫描 `.workflow/` 中的 `agent_id`、`agent:` 和旧版 `建议 Agent` 引用。
2. 先把未完成 ticket 改派，或明确标记为 `blocked`。
3. 从 `agents[]` 移除 agent。
4. 从 runtime adapter 的 `agent_ids` 移除死引用，除非 adapter 使用 `["*"]`。

### 改 agent

- 改 `display_name`：安全，不影响 ticket。
- 改 `role`：只影响人读的说明，不影响自动路由。
- 改 `tags`：影响后续自动路由，已显式写 agent 的 ticket 不变。
- 改 `cost_tier`：影响成本兜底顺序，需要重新预检。
- 改 `id`：破坏性操作，必须迁移所有 ticket、handoff、runtime adapter 和脚本引用。

## 迁移旧版 manual_entry

旧版 schema 可能有 `manual_entry` 字段。迁移时不要保留“只能有一个 true”的约束：

1. 删除 `manual_entry` 字段。
2. 读取它原来承担的阶段，把目标写到 `.workflow/runtimes.json` 的 `routing.defaults`。
3. 如果不同阶段需要不同 agent，分别配置 `planning`、`research`、`execution`、`verification` 或 `review`。
4. 如果没有默认值，让路由按能力匹配；仍然有多个候选时询问用户。
5. 运行 `/route-agent`、`/dispatch` 和 `/verify` 预检。

## 与其它 skill 的衔接

- `/setup-runtimes` 管理 runtime、adapter、capabilities、agent policy 和阶段默认目标。
- `/route-agent` 读取 agents 和 runtimes，输出完整的 runtime + adapter + agent。
- `/dispatch` 以 `agent_id` 分派；`display_name` 只能用于展示。
- `/to-tickets` 在 ticket 中写 `agent_id: auto`、稳定 ID 或显式 `null`，不写显示名。
- `/verify` 和 `/code-review` 不再查找 `manual_entry`。
- `/setup-workflow` 完全不管 agent 配置，只管路径类字段。

## 反模式

- 🚫 把某个 agent 设成所有手动 skill 的唯一入口
- 🚫 把 agent ID 写成运行时名称或模型厂商名称
- 🚫 ticket 写 `display_name`，导致改名后引用断裂
- 🚫 新建 agent 不给标签，然后期待它被领域路由选中
- 🚫 修改 agent 后不检查现有 ticket 和 adapter 引用
- 🚫 把 agents.json 放进 `.gitignore`，它必须提交到 git

## 输出格式

```markdown
## 🤖 agents.json 已更新

### ➕ 新增
- `<agent-id>`：role `<role>`，cost `<tier>`，tags `<tag-1>`, `<tag-2>`

### 🔀 路由影响
- `<ticket-id>` 仍显式指定 `<agent-id>`，不受自动路由变化影响
- `<ticket-id>` 需要重新运行 `/route-agent`

### ⚠️ 运行时检查
- <哪些 adapter 需要安装或移除该 agent>
```
