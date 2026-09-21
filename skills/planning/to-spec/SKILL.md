---
name: to-spec
description: 将当前对话或 grill 摘要综合成一份正式 spec，存入 .workflow/specs/。不重新 interview，只做综合。
disable-model-invocation: true
---

# To Spec

把 grill-me 产出的摘要或当前对话中已经讨论清楚的内容综合成一份正式 spec。不要重新 interview，只做综合。

## 前提

- `.workflow/config.json` 已存在
- 已有 grill 摘要或足够的对话上下文
- 如果引用调研，调研文档已在 `.workflow/research/`

## 流程

1. 读取当前上下文、grill 摘要、调研文档、CONTEXT.md 和相关 ADR。
2. 确定实现行为和测试 seam，必要时向用户确认 seam。
3. 写 spec 到 `.workflow/specs/<feature-name>.md`。
4. 不在 spec 中把 runtime、adapter 或 agent 写成固定前提。只有当行为确实依赖某个能力时，描述能力或约束。

## spec 模板

```markdown
# <功能名> Spec

## 🎯 问题陈述
<用户视角的问题和当前不可接受的行为>

## 💡 解决方案
<做完之后用户的处境有什么不同>

## 👤 用户故事
1. 作为 <角色>，我希望 <功能>，以便 <收益>

## 🏗️ 实现决策
<每条决策说清选了什么、为什么、什么情况下重新讨论>

## 🧪 测试决策
- 测试外部行为，不测试实现细节
- 复用的测试 seam 和相关先例

## 🚫 范围外
- <明确不做的事情>

## 📎 附注
<其它需要记录的信息>
```

涉及多模块、状态迁移或失败路径时画 Mermaid 图。节点使用真实模块名，图不超过九个节点。

## 实现决策规则

- 每条决策说清选择、理由和重新讨论条件
- 不用“涉及模块”“接口列表”填空
- 不要包含容易过时的具体文件路径和代码片段
- 如果 prototype 产出精确 schema、状态机或 reducer，可以内联并标注来源

## 交稿前

按 `/readable-docs` 自检。返工时直接改正文，不保留旧版本和“补充说明”痕迹。

## 下一步

- 运行 `/to-tickets` 拆 ticket，并为每个 ticket 写 route 和所需能力
- spec 足够小时，也可以直接在具备 `dispatch` 的 runtime 上运行 `/dispatch`
