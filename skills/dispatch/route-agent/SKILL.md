---
name: route-agent
description: 根据 ticket 内容自动判断应分派给哪个 CodeG agent。读 .workflow/agents.json（agent 注册表，由 /setup-agents 管理）做基于 tags + cost_tier 的动态路由，不硬编码 agent 名。供 /dispatch 调用，也可独立运行做路由预检。
---

# Route Agent

当一个 ticket 没有明确的 `## 建议 Agent` 字段，或用户想覆盖默认路由时，用本 skill 自动判断。

## 设计原则

- **数据驱动**：本 skill **不硬编码任何 agent 名**。agent 表在 `.workflow/agents.json` 里（由 `/setup-agents` 管理），改 agent 配置只动那一个文件，本 skill 自动按新表路由
- **按成本兜底**：当 ticket 模糊到无法匹配明确标签时，从能匹配的 agent 里选 `cost_tier` 最便宜的——保护资源，与"刀刃上用 expensive agent"哲学一致
- **单入口**：手动 skill（grill-me / to-spec / to-tickets / dispatch / verify 等）只走 `agents.json` 里 `manual_entry: true` 的那一个 agent

## 规则来源（优先级从高到低）

1. **ticket 文件里的 `## 建议 Agent` 字段**（`/to-tickets` 已经写过；这一级最高）
2. **基于 ticket 关键词 + agents.json tags 匹配**（见下方路由算法）
3. **cost_tier 兜底**（从匹配到的 agent 里选最便宜的）

> 本 skill **不**用 `config.json` 里的 `agent_routing` 字段——该字段已废弃，被 `agents.json` 取代。

## 路由算法

按顺序检查每条规则，第一条命中就用：

### 规则 1：明确关键词 → 强制昂贵档

ticket 任务描述包含以下关键词时，**强制路由到 `cost_tier: expensive` 且对应 tag 的那个**：

| 关键词 | tag | 说明 |
|--------|-----|------|
| `认证` / `鉴权` / `auth` / `authorization` / `login` | `security` | 安全敏感 |
| `密钥` / `加密` / `token` / `secret` / `credentials` | `security` | 安全敏感 |
| `权限` / `permission` / `role-based` | `security` | 安全敏感 |
| `架构` / `接口设计` / `权衡` / `trade-off` / `选型` | `architecture` | 架构决策 |
| `module boundary` / `重构策略` / `breaking change design` | `architecture` | 架构决策 |

如果多个 agent 都标了对应 tag，**选最便宜的**（在 expensive 范围内挑便宜的）。

### 规则 2：领域关键词 → 匹配 tags

| 关键词 | 匹配 tag |
|--------|---------|
| `前端` / `组件` / `UI` / `CSS` / `样式` / `动画` / `交互` | `frontend-design` / `frontend-impl` / `ui` / `interaction` / `styling` |
| `CRUD` / `服务层` / `API 端点` / `DTO` / `迁移` / `脚手架` | `crud` / `service-layer` / `api-endpoint` / `scaffold` / `migration` |
| `配置` / `环境变量` / `CI/CD` | `config` |
| `调研` / `research` / `查文档` | `research` |
| `verify` / `跑测试` / `构建` | `verify` |

命中后从匹配 agent 里按 `cost_tier` 选最便宜的。

### 规则 3：占位词 → 强制昂贵档

ticket 任务描述里出现以下任一占位词（说明 ticket 没写清楚）：

`待定` / `待评估` / `视情况` / `后续决定` / `TBD` / `TODO: 设计` / `placeholder`

→ 强制路由到 `cost_tier: expensive` 的 agent（刀刃用，避免低档 agent 自己拍板错误决策）。

### 规则 4：兜底

如果规则 1、3 都没匹配上、规则 2 也没明确标签：

- 从 `agents.json` 里**所有** agent 中按 `cost_tier` 选最便宜的（cheap > medium > expensive）
- 如果最便宜的 agent 都没有任何 tag（纯空标签），报错让用户跑 `/setup-agents` 补 tag

## tag 匹配与 cost_tier 的组合

```text
匹配流程：
1. 扫 ticket 关键词
2. 收集命中的 tag 集合
3. 在 agents.json 里找 tag 集合 ⊆ agent.tags 的所有 agent
4. 如果命中多个 agent：
   a. 如果规则要求"强制昂贵档" → 在 cost_tier=expensive 的命中 agent 里选最便宜的
   b. 否则 → 在所有命中 agent 里按 cost_tier 选最便宜的（cheap > medium > expensive）
5. 如果命中 0 个 agent → 走规则 4 兜底
```

## 输出格式

每条 ticket 给一个判断：

```
Ticket: [003] implement-search-ranking
建议 Agent: pi
理由: spec 已明确排序字段和返回结构，标准服务层实现，匹配 tag: service-layer；cost_tier=medium
```

```
Ticket: [007] design-auth-interface
建议 Agent: harness
理由: 命中关键词"鉴权"+"选型"；强制昂贵档 → harness（cost_tier=expensive）；tags: security, architecture
```

## 升级路径（兜底档 → 昂贵档）

如果某个 ticket 在实现过程中发现：

- 需要改 ticket 没提到的接口或模块边界
- 遇到认证/权限相关的隐藏约束
- 决策空间超出 spec

立即在 ticket 上把 `## 建议 Agent` 改成对应昂贵档 agent 的 `id`，重新走 `/dispatch`。**不要硬撑**，但**也不要一开始就升**——先让便宜档 agent 尝试，暴露问题再升。

## 作为独立 skill 运行

用户可对所有 ticket 预跑一遍路由，查看分派全景：

```
[001] init-db-schema        → pi     (tags: scaffold; medium)
[002] user-auth-logic       → harness (tags: security; expensive)
[003] implement-search      → pi     (tags: service-layer; medium)
[004] search-ui             → antigravity (tags: frontend-design; medium)
[005] refactor-data-layer   → pi     (tags: crud; medium)
[006] auth-token-middleware → harness (tags: security; expensive)
```

> 注：以上 agent id 取决于当前 `.workflow/agents.json` 的内容。改 agent 表后这条样本就过时。

## 反模式

- 🚫 **本 skill 写死 agent 名**：agent 表改了，本 skill 不该跟着改
- 🚫 **强制某条 ticket 走昂贵档**：昂贵档是规则驱动（规则 1 / 3），不是手动覆盖
- 🚫 **多个 agent 同时 `manual_entry: true`**：会让本 skill 的"单入口"逻辑失效
- 🚫 **ticket 写"## 建议 Agent: <display_name>"**：用 `id` 写，否则 rename 后引用会断

## 与其它 skill 的衔接

- **`/setup-agents`** 改 agent 表；本 skill 自动按新表路由
- **`/dispatch`** 读本 skill 的输出，生成 CodeG To-dos
- **`/to-tickets`** ticket 模板里 `## 建议 Agent` 字段写 agent `id`