---
name: to-spec
description: 将当前对话或 grill 摘要综合成一份正式 spec 文件，存入 .workflow/specs/。不重新 interview，只做综合。
disable-model-invocation: true
---

# To Spec

把 grill-me 产出的摘要（或当前对话中已经讨论清楚的内容）综合成一份正式 spec。**不要重新 interview**，只做综合。

## 前提

- `.workflow/config.json` 应已存在。如果没有，告知用户先运行 `/setup-workflow`。
- 已有 grill 摘要或足够的对话上下文。

## 流程

1. **读取上下文**：读取当前对话历史、grill 摘要文件（如有）、`.workflow/research/` 下的调研文档（如相关）。使用项目已有的术语，尊重相关 ADR。

2. **确定测试 seam**：梳理出实现这个功能要测试的边界（seam）。优先复用已有 seam，其次在最合理的高度新建。seam 越少越好，理想数量是 1。

   向用户确认这些 seam 是否符合预期。

3. **写 spec**：用下面的模板写，然后存入 `.workflow/specs/<feature-name>.md`。

## spec 模板

```markdown
# <功能名> Spec

## 问题陈述
<用户面临的问题，从用户视角表述>

## 解决方案
<解决方案方向，从用户视角表述>

## 用户故事
1. 作为 <角色>，我希望 <功能>，以便 <收益>
2. ...

## 实现决策
- 涉及的模块
- 模块接口
- 技术澄清
- 架构决策
- Schema 变更
- API 契约

不要包含具体文件路径或代码片段，它们会很快过时。

例外：如果 prototype 产出了能精确编码某个决策的片段（状态机、reducer、schema、type shape），内联到相关决策中并注明来自 prototype。

## 测试决策
- 什么是好的测试（只测外部行为，不测实现细节）
- 哪些模块会被测试
- 测试的先例（codebase 中类似的测试）

## 范围外
- 明确列出不做的事情

## 附注
<其它需要记录的信息>
```

## 路径

spec 文件存入 `.workflow/specs/<feature-name>.md`，feature-name 用 kebab-case。

## 下一步

spec 写完后，告知用户：
- 运行 `/to-tickets` 将 spec 拆成可并发的 ticket
- 或如果 spec 足够小，可直接在服务器端运行 `/dispatch`