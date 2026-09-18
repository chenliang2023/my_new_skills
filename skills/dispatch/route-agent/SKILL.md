---
name: route-agent
description: 根据 ticket 内容自动判断应分派给哪个 CodeG agent（DeepSeek Harness / Pi / Google AntiGravity）。供 /dispatch 调用，也可独立运行做路由预检。
---

# Route Agent

当一个 ticket 没有明确的 `## 建议 Agent` 字段，或用户想覆盖默认路由时，用本 skill 自动判断。

## 规则来源（优先级从高到低）

1. **ticket 文件里的 `## 建议 Agent` 字段**（`/to-tickets` 已经写过；这一级最高）
2. **用户写在 `.workflow/config.json` 里的 `agent_routing` 覆盖项**（项目级偏好；可选）
3. **本 skill 内嵌规则**（默认 Pi；架构决策 / 安全敏感升 Harness；前端 → AntiGravity）

`config.json` 的覆盖项结构：

```json
{
  "agent_routing": {
    "harness": ["architecture", "security", "core-domain-decision"],
    "pi": ["crud", "service-layer", "api-endpoint", "scaffold", "config", "migration", "verify", "research"],
    "antigravity": ["ui", "interaction", "styling"]
  }
}
```

如果 `agent_routing` 写得不完整（缺某 agent 的标签数组），缺失的部分回到本 skill 内嵌规则。

## 设计原则

**DeepSeek Harness 比较贵，只用在刀刃上**。它只接两类 ticket：

1. 需要 agent 自己做出**架构决策**的（接口设计、模块边界、权衡取舍、需要拍板的多方案对比）
2. **安全敏感**的（认证、加密、权限、密钥、token 处理、合规审计）

**判断边界**：

- 如果 ticket 已经把"做什么 / 不做什么"写清楚了，**不要**走 Harness，让 Pi 干
- 如果 ticket 出现"待定 / 待评估 / 视情况 / 后续决定"之类的占位词，**必须**走 Harness
- 如果是中等复杂度的实现但**不涉及架构或安全**，下放 Pi

其它一律下放给 Pi 或 AntiGravity。**默认路由是 Pi**，不是 Harness（与原 Claude 默认策略不同）。

## 三个 Agent 的路由规则

### 🐳 DeepSeek Harness（重推理，刀刃用，最贵）

**强项**：复杂推理、长上下文理解、架构敏感代码。

**只接两类 ticket**：

- 涉及**架构决策**：接口设计、模块边界、需要权衡取舍的实现路径
- **安全敏感**：认证、加密、权限、密钥、token 处理

**不接**：

- 任何 spec 已经写清楚的实现（无论看起来多大规模）
- 标准 CRUD / 服务层 / DTO / 迁移文件
- 任何可以读完任务直接动手的 ticket
- 多模块中等复杂度的实现（前提是不需要架构决策）→ 下放 Pi

### ⚡ Pi（覆盖最广，默认路由）

**强项**：快速产出标准业务代码、服务层、API 端点、调研、配置、跑命令。

**路由到 Pi 的信号**：

- 标准业务逻辑（CRUD、列表/详情/编辑）
- 服务层、API 端点实现
- DTO / schema 定义
- 数据库迁移文件
- 配置文件、环境变量设置
- 脚手架、项目初始化
- 多模块中等复杂度的实现（前提是不需要架构决策）
- 技术调研、文档查证（不需要强推理的）
- **verify / 集成测试 / 跑命令报告结果** 等手动会话（默认入口；遇到架构级疑问再升 Harness）

### 🎨 AntiGravity（前端/交互）

**强项**：UI 组件、交互逻辑、样式。

**路由到 AntiGravity 的信号**：

- React/Vue/前端组件
- CSS / 样式实现
- 交互逻辑（拖拽、动画、表单）
- 响应式布局
- 前端状态管理

## 路由算法

1. 读取 ticket 的 `## 任务` 和 `## 验收标准`
2. 判断是否安全敏感（认证/加密/权限/密钥）→ **Harness**
3. 判断是否需要架构决策（读完任务后 agent 还需要做选择/权衡/接口设计）→ **Harness**
4. 判断是否前端/UI → **AntiGravity**
5. 其它全部 → **Pi**（默认）
6. 如果 ticket 模糊到连前端/安全/架构都无法判断，**仍然路由到 Pi**，让 Pi 在实现过程中暴露问题，再升 Harness

> 与原策略的反转：原 Claude 路线下"无法判断时默认路由到 Claude（保守）"，现在默认路由到 Pi（成本优先），等 Pi 暴露问题再升 Harness。

## Pi vs Harness 的分界

| 维度 | Pi | DeepSeek Harness |
|------|----|-----------------|
| 推理密度 | 中低（spec 已明确或读完能做） | 高（需要权衡、推理、做架构选择） |
| 不确定性 | 低（路径清晰） | 高（路径不清晰，需要 agent 决策） |
| 代码量 | 大小都行（标准模式大量产出） | 中（核心逻辑、决策点） |
| 架构影响 | 低（动实现，不动接口） | 高（动接口、模块边界） |
| 安全敏感 | 否 | 是 |
| 成本 | 低 | 高 |

**经验法则**：

- 如果 ticket 的 `## 任务` 读完后 agent 还需要自己**设计接口或选型** → Harness
- 如果读完就能直接写代码（无论代码量大小）→ Pi
- 如果不确定是否安全敏感或架构敏感 → 倾向 Pi，让实现暴露问题再升级

## 升级路径（Pi → Harness）

如果在 Pi 实现过程中发现：

- 需要改 ticket 没提到的接口或模块边界
- 遇到认证/权限相关的隐藏约束
- 决策空间超出 spec

立即在 ticket 上把 `## 建议 Agent` 改成 Harness，重新走 `/dispatch`。**不要硬撑**，但**也不要一开始就升**，先让 Pi 尝试。

## 输出格式

```
Ticket: [003] implement-search-ranking
建议 Agent: Pi
理由: spec 已明确排序字段和返回结构，标准服务层实现，照做即可
```

```
Ticket: [007] design-auth-interface
建议 Agent: DeepSeek Harness
理由: 涉及认证接口设计，属于安全敏感 + 需要架构决策
```

## 作为独立 skill 运行

用户可对所有 ticket 预跑一遍路由，查看分派全景：

```
[001] init-db-schema        → Pi
[002] user-auth-logic       → DeepSeek Harness
[003] implement-search      → Pi
[004] search-ui             → AntiGravity
[005] refactor-data-layer   → Pi（中等复杂度，无架构决策）
[006] auth-token-middleware → DeepSeek Harness（安全敏感）
```