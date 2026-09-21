---
name: setup-runtimes
description: 管理 .workflow/runtimes.json 里的运行环境和 adapter 注册表。让本机、CodeG 或其它运行面都能作为可选的规划、执行、验证和 review 环境。
disable-model-invocation: true
---

# Setup Runtimes

管理项目的运行时注册表。运行时注册表描述任务在哪里运行、通过什么 adapter 运行，以及该 adapter 提供哪些能力。

`.workflow/runtimes.json` 是运行时和 adapter 的唯一真相源。`.workflow/agents.json` 仍然只描述 agent，不在 agent 记录里写本机、CodeG 或其它环境信息。

## 何时调用

- 第一次在项目上启用这套工作流
- 新增或移除一个运行环境
- 给运行环境增加、删除或替换 adapter
- 改变某个阶段的默认路由
- CodeG To-dos、另一个任务系统或本机执行方式发生变化

不要把运行时配置塞进 `config.json`。`setup-workflow` 只管理路径类字段，`setup-agents` 只管理 agent 注册表。

## 数据模型

```json
{
  "schema_version": 1,
  "runtimes": [
    {
      "id": "local",
      "display_name": "本机工作区",
      "kind": "local",
      "status": "available",
      "adapters": [
        {
          "id": "local-session",
          "display_name": "本机会话",
          "capabilities": ["plan", "research", "execute", "verify", "review", "integrate", "interactive"],
          "agent_ids": ["*"]
        }
      ]
    },
    {
      "id": "codeg",
      "display_name": "CodeG 工作区",
      "kind": "remote",
      "status": "available",
      "adapters": [
        {
          "id": "codeg-todos",
          "display_name": "CodeG To-dos",
          "capabilities": ["execute", "dispatch", "parallel", "isolated-worktree", "review", "merge", "verify", "preflight", "delegation"],
          "agent_ids": ["*"]
        }
      ]
    }
  ],
  "routing": {
    "defaults": {
      "planning": null,
      "research": null,
      "execution": null,
      "verification": null,
      "review": null,
      "integration": null,
      "release": null
    },
    "fallback": "capability_then_ask"
  }
}
```

示例里的 `local`、`codeg` 和 adapter ID 都是可删改的示例，不是内置名称。项目可以只注册一个 runtime，也可以注册多个 runtime 和多个 adapter。

## 字段说明

| 字段 | 必填 | 含义 |
|------|------|------|
| `schema_version` | 是 | 当前运行时注册表 schema 版本 |
| `runtimes` | 是 | 运行环境数组，顺序无意义 |
| `runtimes[].id` | 是 | 稳定的 kebab-case 运行时 ID |
| `runtimes[].display_name` | 是 | 给人看的显示名，可以随时修改 |
| `runtimes[].kind` | 是 | 运行时类型，例如 `local`、`remote`、`hosted` |
| `runtimes[].status` | 是 | `available`、`disabled` 或 `unknown` |
| `adapters[].id` | 是 | 全注册表唯一的 adapter ID，ticket 可引用 |
| `adapters[].display_name` | 是 | adapter 的显示名 |
| `adapters[].capabilities` | 是 | 该 adapter 实际支持的能力集合 |
| `adapters[].agent_ids` | 是 | 可用 agent 的 ID 数组；`[]` 表示只支持 agentless，`["*"]` 表示接受任意已注册 agent |
| `routing.defaults` | 否 | 各工作阶段的默认目标，值为完整 route 对象或 `null` |
| `routing.fallback` | 否 | 无显式路由时的策略，默认 `capability_then_ask` |

`agent_ids: ["*"]` 只表示 adapter 接受任意已注册 agent，不代表每个 agent 已安装或已认证。创建任务前，adapter 必须按自己的发现或健康检查机制确认 agent 可用。

默认目标的格式：

```json
{
  "phase": "execution",
  "runtime_id": "local",
  "adapter_id": "local-session",
  "agent_id": "some-agent",
  "capabilities_all": ["execute"],
  "capabilities_any": []
}
```

`agent_id: null` 表示由当前用户会话、脚本或人工执行，不需要注册 agent。`agent_id: "auto"` 才表示让路由器从可用 agent 中动态选择。

## Canonical route 格式

ticket 和调用参数使用同一组机器字段：

```yaml
phase: execution
runtime_id: auto
adapter_id: auto
agent_id: auto
capabilities_all: [execute]
capabilities_any: []
```

- `capabilities_all`：必须全部具备
- `capabilities_any`：至少具备其中一个；空数组表示没有额外的 OR 条件
- `phase`：`planning`、`research`、`execution`、`dispatch`、`verification`、`review`、`integration` 或 `release`
- `runtime_id`、`adapter_id`、`agent_id`：稳定 ID、`auto` 或显式 `null`，其中 `agent_id: null` 代表 agentless

人读的 Markdown 字段 `Runtime`、`Adapter`、`Agent` 只是这三个 canonical key 的展示别名。

## 能力名称

能力是路由条件，不是环境名称。常用能力包括：

- `plan`、`research`：规划和调研
- `execute`：修改工作区或运行实现任务
- `dispatch`、`parallel`、`delegation`：任务分派和并发协作
- `isolated-worktree`：每个任务使用隔离 worktree
- `verify`、`preflight`：验证命令和自动前置检查
- `review`、`merge`、`integrate`：审查、合并和跨 ticket 协调
- `interactive`：需要用户持续参与的会话

CodeG 的 To-dos、worktree、并发、Review/Merge 和 preflight 都通过能力声明接入。它们是 `codeg-todos` adapter 的可选能力，不是整个工作流的前提。本机 adapter 也可以声明相同能力，或声明自己只有其中一部分。

`preflight` 只表示任务级自动检查，不等于可以执行完整 `/verify`。需要全量验证时，目标必须声明 `verify`。

## 路由规则

路由同时选择三个维度：`runtime_id`、`adapter_id`、`agent_id`。三者不要互相替代。

优先级从高到低：

1. 用户本次调用明确指定的 runtime、adapter 或 agent
2. ticket 的 canonical route 字段
3. `runtimes.json` 中当前 phase 的默认目标
4. 按所需能力、运行时状态和 agent 标签动态匹配
5. 没有唯一结果时询问用户，不猜一个隐藏默认值

如果显式指定的三者互相冲突，报告冲突并停止分派。只有用户明确允许回退时，才可以使用候选目标替换缺失的 runtime、adapter 或 agent。

如果只指定一个维度：

- 只指定 `runtime_id`：在该运行时的可用 adapter 中匹配能力，再匹配 agent
- 只指定 `adapter_id`：由注册表找到所属运行时，再匹配 agent
- 只指定 `agent_id`：在可用运行时和 adapter 中寻找支持该 agent 的目标
- `agent_id: null`：只选择支持 agentless 执行的 adapter

## 流程

### 新增运行时

1. 分配稳定的 `runtime.id`，不要把机器名、IP 或临时路径写进 ID。
2. 添加一个或多个 adapter，声明真实能力和 `agent_ids` 语义。
3. 如果 adapter 只安装了部分 agent，在 `agent_ids` 中列出它们；如果支持任意已注册 agent，显式写 `["*"]`。
4. 只有用户明确需要默认值时，才在 `routing.defaults` 写目标。
5. 用 `/route-agent` 预检若干 ticket，确认没有不可达路由。

### 修改或删除运行时

1. 先扫描 ticket 和 handoff 中的 `runtime_id`、`adapter_id` 引用。
2. 停用前先把未完成 ticket 改派到有效目标，或明确标记为 blocked。
3. 只改显示名不会破坏引用。改 ID 要做迁移，并保留迁移记录。
4. 删除 CodeG adapter 不等于删除工作流。只要另一个 adapter 声明了所需能力，ticket 仍可运行。

## 与其它 skill 的衔接

- `/setup-workflow` 创建目录和路径类 `config.json`，不创建或修改运行时注册表。
- `/setup-agents` 管理 `.workflow/agents.json`，不在 agent 记录里写运行时。
- `/route-agent` 读取 agents 和 runtimes，输出完整的 runtime + adapter + agent 目标。
- `/dispatch` 按 adapter 能力选择任务队列、隔离、并发和委托方式。
- `/verify`、`/code-review` 和 `/integrate` 可以在任何声明对应能力的运行时执行。
- `/context-sync` 通过 git 同步 `.workflow/`，并把同一套 skills 安装到需要它们的 runtime adapter。

## 迁移旧配置

旧版 `agents.json` 可能有 `manual_entry` 字段。迁移时：

1. 保留 agent 的 `id`、`display_name`、`role`、`cost_tier`、`tags` 和 `description`。
2. 删除或忽略 `manual_entry`，不要再要求全表只有一个入口 agent。
3. 把原来想表达的阶段默认入口，拆成 `routing.defaults.<phase>` 的完整目标。
4. 为每个 adapter 明确填写 `agent_ids`：空数组表示 agentless，`["*"]` 表示任意已注册 agent，具体数组表示白名单。
5. 运行 `/route-agent` 和 `/dispatch` 做预检，确认 ticket 没有死引用。

## 反模式

- 🚫 把本机写成规划端、CodeG 写成执行端
- 🚫 用 agent 名称推断运行环境
- 🚫 恢复单一 `manual_entry` agent 作为所有手动 skill 的入口
- 🚫 把 CodeG To-dos、worktree 或 Merge 写成所有 adapter 都必须提供
- 🚫 运行时 ID 使用会变化的机器路径或临时会话名
- 🚫 没有能力声明就声称支持并发、隔离或自动验证
- 🚫 用空的 `agent_ids` 表示“所有 agent”，导致 agentless 和动态 agent 混淆

## 输出格式

```markdown
## 🧭 运行时注册表已更新

### ➕ 变化
- 新增 `<runtime-id>`，提供 `<adapter-id>`
- `<adapter-id>` 能力：`execute`、`verify`
- agent policy：`["*"]` / 指定白名单 / agentless

### 🔀 默认路由
- execution：未设置，按 ticket 能力动态路由
- review：`<runtime-id> / <adapter-id> / <agent-id>`

### ⚠️ 影响
- <受影响的 ticket 或需要重新预检的路由>
```
