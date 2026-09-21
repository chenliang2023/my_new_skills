---
name: grill-me
description: 通过 relentless interview 打磨一个想法、计划或设计，直到它足够清晰可以写 spec。在当前可交互的 runtime 中运行。
disable-model-invocation: true
---

# Grill Me

你有一个模糊的想法或需求，需要通过 rigorous interview 把它打磨清楚，然后才能写 spec 或 ticket。

## 定位

这是交互式规划入口。它不属于某个固定地点或固定 agent。只要当前 runtime 具备 `interactive` 能力，就可以运行本 skill。若需要在另一个 runtime 继续，使用 `.workflow/handoffs/` 保存摘要。

如果项目有 `.workflow/config.json`，可以把摘要写入 `.workflow/handoffs/grill-summary-<date>.md`。是否写入本机、CodeG 或其它位置由当前 runtime 的路径和同步策略决定。

## interview 规则

### 轮次制

每一轮收集用户回答后，综合出当前理解，再提出下一批问题。每轮不超过 5 个问题，优先解决阻塞后续理解的问题。

### frontier 推进

每轮结束时明确告诉用户：

- **已确认的事实**：本轮从用户回答、代码或调研中确定的事实
- **下一轮问题**：当前 frontier 上最靠前的问题
- **估计剩余轮次**：还需几轮才能形成可交付 spec

### 事实与决策的分工

- **事实**是 agent 的职责：查文档、读代码、验证 API 行为。不要把本该查的东西拿去问用户。
- **决策**是用户的职责：范围、优先级和业务规则。不要替用户拍板，把决策点呈现出来。

## 何时结束

满足以下全部条件时结束 interview：

1. 问题陈述清楚，从用户视角说明现在发生什么
2. 解决方案轮廓清晰，不在根本不同的方向间摇摆
3. 验收标准可描述，能表达“做到 X 就算完成”
4. 边界明确，范围外已经讨论过

结束时输出 interview 摘要：

```markdown
## Interview 摘要

### 🎯 问题
<用户视角的问题陈述>

### 💡 解决方案轮廓
<做完之后用户的处境有什么不同>

### 📌 已确认的事实
<每条注明来源：用户原话、调研文档或代码现状>

### ✅ 验收标准
1. ...
2. ...

### 🚫 范围外
- ...

### ❓ 未决
<需要用户拍板、需要调研或需要先看其它结论的问题>
```

摘要按 `/readable-docs` 写，段落短，emoji 只用于导航。若后续由另一个 runtime 执行，摘要必须足够自包含，不要假设当前 session 仍然存在。

## 与其它 skill 的关系

- 摘要喂给 `/to-spec`
- 需要技术事实时暂停并运行 `/research`
- 需要明确执行位置时，在 `/to-tickets` 为 ticket 写 route，而不是在 grill 阶段假设某个环境
