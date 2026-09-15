---
name: route-agent
description: 根据 ticket 内容自动判断应分派给哪个 CodeG agent（Claude/PI/AntiGravity）。供 /dispatch 调用，也可独立运行做路由预检。
---

# Route Agent

当一个 ticket 没有明确的 `## 建议 Agent` 字段，或用户想覆盖默认路由时，用本 skill 自动判断。

## 三个 Agent 的路由规则

### Claude (主力实现 - 重推理)
**强项**：复杂推理、长上下文理解、架构敏感代码。
路由到 Claude 的信号：
- ticket 涉及核心业务逻辑（非标准 CRUD）
- 需要理解多个模块间的交互关系
- 架构变更、接口设计
- 需要强推理的 bug 修复
- 安全敏感代码（认证、加密、权限）
- 不确定性高、需要权衡取舍的实现

### PI (主力实现 - 重产出)
**强项**：快速产出标准业务代码、服务层、API 端点。
路由到 PI 的信号：
- 标准业务逻辑（CRUD、列表/详情/编辑）
- 服务层、API 端点实现
- DTO / schema 定义
- 数据库迁移文件
- 配置文件、环境变量设置
- 脚手架、项目初始化
- 已有明确 spec、不需要太多推理的实现

### AntiGravity (前端/交互)
**强项**：UI 组件、交互逻辑、样式。
路由到 AntiGravity 的信号：
- React/Vue/前端组件
- CSS / 样式实现
- 交互逻辑（拖拽、动画、表单）
- 响应式布局
- 前端状态管理

## Claude vs PI 的分界

两个都是主力实现 agent，分界在"推理密度"：

| 维度 | Claude | PI |
|------|--------|-----|
| 推理密度 | 高（需要权衡、推理） | 低（spec 已明确） |
| 不确定性 | 高（路径不清晰） | 低（照 spec 做） |
| 代码量 | 中（核心逻辑） | 大（标准模式） |
| 架构影响 | 高（动接口） | 低（动实现） |
| 安全敏感 | 是 | 否 |

经验法则：如果 ticket 的 `## 任务` 读完后还需要 agent 自己做架构决策，给 Claude；如果读完就能直接写代码，给 PI。

## 路由算法

1. 读取 ticket 的 `## 任务` 和 `## 验收标准`
2. 先判断是否前端/UI → 是则 AntiGravity
3. 否则判断推理密度：
   - 需要架构决策/安全敏感/不确定性高 → Claude
   - spec 明确/标准模式/CRUD → PI
4. 如果无法判断，默认路由到 Claude（保守选择）
5. 输出建议及理由

## 输出格式

```
Ticket: [003] implement-search-ranking
建议 Agent: PI
理由: spec 已明确排序字段和返回结构，标准服务层实现，照做即可
```

## 作为独立 skill 运行

用户可对所有 ticket 预跑一遍路由，查看分派全景：

```
[001] init-db-schema        → PI
[002] user-auth-logic       → Claude
[003] implement-search      → PI
[004] search-ui             → AntiGravity
[005] refactor-data-layer   → Claude
```