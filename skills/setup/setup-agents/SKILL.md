---
name: setup-agents
description: 管理 .workflow/agents.json 里的 agent 注册表。新增 agent、删除 agent、改 agent 名字/标签/成本档。每次改动后 /route-agent /dispatch /verify 会自动按新表路由。任何时候想加/减 agent 时调用。
disable-model-invocation: true
---

# Setup Agents

管理项目的 agent 注册表。**用户增删 agent 只动这一个文件**（`.workflow/agents.json`），其它 skill（`/route-agent`、`/dispatch`、`/verify`）会自动读新表。

agent 注册表是 agent 配置的唯一真相源；`/setup-workflow` 不管 agent 配置（它只管路径类配置）；本文档是 agent 配置的入口。

## 何时调用

- 首次配 agent（在 `/setup-workflow` 之后或之前）
- 想加一个 agent（如 Claude Code、Aider、Jules）
- 想删除一个 agent
- 想改 agent 的名字、角色、成本档、标签
- 切换项目时复用 agent 表

不在主流程上每次跑——只在 agent 配置变更时跑。

## 设计原则

- **每个 agent 用一个稳定的 `id`**（短名，kebab-case）：ticket 文件的 `## 建议 Agent` 字段、dispatch 的 `to:` 标记、shell 脚本里的引用都用 `id`
- **`display_name`**（显示名）独立：可以是 "DeepSeek Harness"、"Pi"、"Google Antigravity"，可以随时改，不影响 ticket
- **每个 agent 必须有 `cost_tier`**：`cheap` / `medium` / `expensive` 三档。`/route-agent` 在 ticket 无法匹配明确标签时，**按 cost_tier 选最便宜的**兜底
- **`tags`**：路由匹配标签集，对应到 ticket 的工作类型（frontend / crud / architecture / security / research / verify 等）
- **`manual_entry`**：是否作为"手动会话默认入口"。手动调用型 skill（grill-me / to-spec / to-tickets / dispatch / verify / setup-agents 等）只启给标了这个的 agent
- **id 一旦定了别改**：改了会破坏现有 ticket 的引用。要改 = 走迁移

## agents.json 模板

```json
{
  "schema_version": 1,
  "agents": [
    {
      "id": "harness",
      "display_name": "DeepSeek Harness",
      "cost_tier": "expensive",
      "manual_entry": false,
      "tags": ["architecture", "security", "core-domain-decision", "design-tradeoff"],
      "description": "重推理，刀刃用，只接架构决策/安全敏感 ticket"
    },
    {
      "id": "pi",
      "display_name": "Pi",
      "cost_tier": "medium",
      "manual_entry": true,
      "tags": ["crud", "service-layer", "api-endpoint", "scaffold", "config", "migration", "verify", "research", "frontend-impl"],
      "description": "覆盖最广，手动会话默认入口"
    },
    {
      "id": "antigravity",
      "display_name": "Google Antigravity",
      "cost_tier": "medium",
      "manual_entry": false,
      "tags": ["frontend-design", "frontend-impl", "ui", "interaction", "styling"],
      "description": "前端/交互，UI 组件、动画、样式"
    }
  ]
}
```

## 字段说明

| 字段 | 必填 | 含义 |
|------|------|------|
| `schema_version` | 是 | 当前 schema 版本号（v1） |
| `agents` | 是 | agent 数组，顺序无意义 |
| `agents[].id` | 是 | 短名，kebab-case，ticket 里用 |
| `agents[].display_name` | 是 | 显示名，文档/UI 里用，可含空格/中文 |
| `agents[].cost_tier` | 是 | `cheap` / `medium` / `expensive`，影响兜底路由 |
| `agents[].manual_entry` | 是 | 是否作为手动会话默认入口；只能有一个 agent 标 `true` |
| `agents[].tags` | 是 | 路由标签数组，每个标签在 `/route-agent` 里有对应规则 |
| `agents[].description` | 否 | 一句话简介 |

**约束**：

- `manual_entry: true` 的 agent 全表只能有一个（多个会让 `/ask` 等 skill 不知道默认走哪个）
- `id` 全表唯一，改名要走迁移流程
- `tags` 数组可以重复（多个 agent 共享 tag），路由时按 `cost_tier` 选最便宜的

## 流程：用户增删 agent

### 加 agent

1. 在 `agents[]` 数组末尾追加一项
2. 给一个唯一的 `id`（kebab-case）
3. 选 `cost_tier`——参考既有 agent：贵档（expensive）= 重推理模型（如 o1 / R1 / Claude Opus）、中档 = Sonnet/Haiku 等主力模型、便宜档 = Haiku/MiniMax/Hermes 等轻量模型
4. 给一组 `tags`——可以跟既有 agent 重叠（重叠意味着"都能干这事"，由路由按 cost 选便宜的）
5. 如果要让它作为手动会话入口（这是新加 agent 唯一要做主入口的场景），把 `manual_entry` 设为 `true`，**同时把别的 agent 的 `manual_entry` 改回 `false`**

### 删 agent

1. 从 `agents[]` 里移除整项
2. **检查现有 ticket**：

   ```bash
   grep -rln "建议 Agent: <id>" .workflow/tickets/ .workflow/bugs/tickets/
   ```

   有结果就先改 ticket（指向别的 agent）再删，避免 ticket 留下死引用

3. `/route-agent` 会在读 `agents.json` 时自动忽略已删 agent；兜底路由会按 cost_tier 选新表里最便宜的

### 改 agent

- **改 `display_name`**：安全，不影响 ticket（ticket 用 `id`）
- **改 `tags`**：影响后续路由。已写的 ticket 不受影响（route-agent 先看 ticket 字段，没匹配上才用 tags 兜底）
- **改 `cost_tier`**：影响兜底路由优先级。要慎重
- **改 `id`**：破坏性操作。要改 = 走迁移（见下文）
- **改 `manual_entry`**：影响所有手动 skill 的默认入口；要同步更新 `setup-workflow` 文档里所有"手动会话默认入口"的引用

### 改 agent id（迁移）

1. 先选新的 `id_new`（kebab-case）
2. 扫所有引用：

   ```bash
   grep -rln "<id_old>" .workflow/ docs/ README.md skills/
   ```

3. 替换为 `<id_new>`（git grep `--files-with-matches` 配合 `sed -i` 批量替换）
4. `agents.json` 里把 id 改名
5. 在 `agents.json` 顶部加 `migrations` 数组（如需多步迁移）：

   ```json
   "migrations": [
     { "from": "claude", "to": "harness", "date": "2026-09-18", "reason": "换主力推理模型" }
   ]
   ```

   留作历史追溯用。

## 与其它 skill 的衔接

- **`/route-agent`** 读 `agents.json` 的 `tags` + `cost_tier` 做兜底路由
- **`/dispatch`** 读 `agents.json` 取 `display_name` 输出到分派清单；状态标记 `to:<id>` 用 `id`
- **`/verify`** 默认入口是 `manual_entry: true` 的 agent
- **`/to-tickets`** ticket 模板 `## 建议 Agent` 字段填 `id`（kebab-case）；输出示例里如果提到 agent 名，用 `display_name`
- **`/setup-workflow`** 完全不管 agent 配置；只管路径类
- **`engineering/context-sync`** 矩阵表里 agent 列按 `agents.json` 当前列表渲染；手动 skill 启给 `manual_entry: true` 的 agent

## 反模式

- 🚫 **agent id 改来改去**：每次改都要扫所有 ticket 改引用；用 `display_name` 改名更安全
- 🚫 **多个 agent 同时标 `manual_entry: true`**：会让 `/ask` 等 skill 不知道默认走哪个
- 🚫 **新建 agent 不给 `tags`**：路由会认为它不能干任何事；兜底时也只在 cost_tier 最便宜时才会选中
- 🚫 **ticket 写 `## 建议 Agent: <display_name>`**：用 `id` 写，否则 rename 后引用会断
- 🚫 **`agents.json` 不提交 git**：其它 skill 读不到 agent 表；必须进 git
- 🚫 **`agents.json` 进 `.gitignore`**：同上

## 输出格式

成功时：

```markdown
## 🤖 agents.json 已更新

### ➕ 新增
- claude（重计）：cost: medium / tags: crud, service-layer / manual_entry: false

### ➖ 删除
- antigravity

### ⚠️ 影响
- [005] search-ui 的 "建议 Agent: antigravity" 已是死引用，请改派到 harness 或 pi
- /verify 默认入口仍是 pi（manual_entry 未变动）
- /route-agent 兜底选择从 antigravity → harness（cost: medium → expensive）跳到 harness，cost 变动
- 请跑一次 /dispatch 预检确认 ticket 不存在孤立引用
```

无影响时（无变化）：

```markdown
## ✅ agents.json 内容与磁盘上一致，未变动
```

## 与 setup-workflow 的边界

| 维度 | setup-workflow | setup-agents |
|------|---------------|--------------|
| 配置文件 | `.workflow/config.json` | `.workflow/agents.json` |
| 管路径类 | ✓（tickets_dir / specs_dir 等） | ✗ |
| 管 agent 类 | ✗ | ✓（id / tags / cost_tier 等） |
| 调用时机 | 首次配置项目 | 任何想改 agent 时 |
| 改它影响 | 路径类 skill | 路由/分派/verify |