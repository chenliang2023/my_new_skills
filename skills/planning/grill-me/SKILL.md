---
name: grill-me
description: 通过 relentless interview 打磨一个想法、计划或设计，直到它足够清晰可以交由服务器端 agent 实现。在本机 VS Code 端运行。
disable-model-invocation: true
---

# Grill Me

你有一个模糊的想法或需求，需要通过 rigorous interview 把它打磨清楚，然后才能交给 CodeG 的 agent 去做。

## 定位

这是本机规划端的入口 skill。你在本机 VS Code 里运行它，它通过多轮提问把你的想法从"我觉得要做个 X"打磨成"这是一个有明确边界、明确验收标准的 spec 输入"。

如果你在一个有 `.workflow/config.json` 的项目目录中，用 `/grill-with-docs` 代替：它会额外把术语写入 `CONTEXT.md`，把决策记录为 ADR。本机没有项目仓库时（纯探索性思考），用本 skill。

## interview 规则

### 轮次制

每一轮你（agent）收集用户回答后，综合出当前理解，然后提出下一批问题。每轮问题不超过 5 个，优先解决阻塞后续理解的问题。

### frontier 推进

每一轮结束后，明确告诉用户：
- **已确认的事实**（本轮从用户回答中确定的事实）
- **下一轮要解决的问题**（当前 frontier 上最靠前的问题）
- **估计剩余轮次**（粗略：还需几轮才能到达可交付 spec）

### 事实与决策的分工

- **事实**是 agent 的职责：查文档、读代码、搜索 API 行为。不要把本该你查的东西拿去问用户。
- **决策**是用户的职责：范围取舍、优先级、业务规则。不要替用户做决策，把决策点呈现出来让用户选。

## 何时结束

当满足以下全部条件时结束 interview：

1. 问题陈述清楚：用户面临的问题是什么，从用户视角表述
2. 解决方案轮廓清晰：大致方向已定，不是在几个根本不同的方向间摇摆
3. 验收标准可描述：能用"做到 X 就算完成"来表达
4. 边界明确：哪些东西不在范围内已经讨论过

结束时输出一份 **interview 摘要**，格式：

```markdown
## Interview 摘要

### 问题
<用户视角的问题陈述>

### 解决方案轮廓
<方向性描述>

### 验收标准
1. ...
2. ...

### 范围外
- ...

### 下一步
建议运行 `/to-spec` 将本摘要转为正式 spec，或运行 `/research` 先做技术调研。
```

如果项目有 `.workflow/config.json`，将摘要写入 `.workflow/handoffs/grill-summary-<date>.md`。

## 与其它 skill 的关系

- 产出的摘要直接喂给 `/to-spec`
- 如果 interview 中发现需要技术调研才能回答的问题，暂停 interview，先跑 `/research`，拿到结果后继续
- 如果想法太大、一个 session 装不下，改用 `/wayfinder`（待实现）