---
name: code-review
description: 对当前 diff、PR 或分支做双轴代码复审。可在任意具备 review 能力的 runtime 中运行。
disable-model-invocation: true
---

# Code Review

对当前 diff、PR 或分支做双轴审查：

1. **规范符合**：代码是否实现了 spec 和 ticket 描述的行为
2. **工程标准**：代码质量、测试质量和架构合理性

## 何时运行

- ticket 执行完成后，目标 adapter 进入 review 阶段
- 多个 ticket 集成后，需要看整体 diff
- 发布前，需要在另一个 runtime 做独立复审

审查可以在执行 runtime 完成，也可以在另一个具备 `review` 能力的 runtime 完成。跨 runtime 时先通过 git 和 `.workflow/handoffs/` 同步上下文。

## Canonical route

复审使用独立的 `review` phase，不继承执行 ticket 的 route：

```yaml
phase: review
runtime_id: auto
adapter_id: auto
agent_id: auto
capabilities_all: [review]
capabilities_any: []
```

`agent_id: null` 允许当前用户会话、脚本或人工执行，但目标 adapter 的 `agent_ids` 必须是空数组。选择顺序与 `/verify` 相同：用户显式指定、参数或 handoff 中的 route、`routing.defaults.review`、最后按 `review` 能力动态匹配。多个等价候选时询问用户。

## 流程

### 1. 确定范围

读取当前 diff。分支、PR 或工作区的范围由用户明确指定，不根据 runtime 名称猜测 base。

### 2. 规范符合审查

逐个检查 ticket 的验收标准，并指出代码位置。找不到实现就标记 missing。

### 3. 工程标准审查

检查：

- 测试是否在 ticket 指定的 seam 上，是否只测外部行为
- 是否引入不必要耦合或错误边界
- 命名是否与 CONTEXT.md 和项目术语一致
- 错误、空值和边界条件是否覆盖
- 认证、加密和权限变更是否有明显漏洞

### 4. 输出审查报告

```markdown
## 🔍 Code Review 报告

### 🧭 审查目标
- Phase：`review`
- Runtime：<id>
- Adapter：<id>
- Agent：<id、null 或未指定>
- Diff：<范围>

### ✅ 规范符合
- ✅ 验收标准 1：[文件:行] 实现正确
- ❌ 验收标准 2：未找到实现

### 🏗️ 工程标准
#### 🧪 测试
- seam：<评价>
- 反模式：<无或具体问题>

#### 🧩 架构
- <评价>

#### 🚧 边界与安全
- <评价>

### 🐛 发现
1. 🔴 [严重] <描述> → <建议>
2. 🟡 [建议] <描述> → <建议>

### 🏁 结论
- [ ] ✅ 可合并
- [ ] 🟡 需修复后合并
- [ ] 🔴 需重新实现
```

## 原则

- 假设 agent 善意，先理解 intent
- 追溯 ticket 和 spec，而不是凭个人偏好否定代码
- 不追求完美，追求实现行为、测试和架构都足够可靠
