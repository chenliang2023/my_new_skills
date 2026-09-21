---
name: integrate
description: 在 adapter 没有足够的 merge 能力，或多个 ticket 之间需要跨模块协调时使用。CodeG Merge 只是可选 adapter 能力。
disable-model-invocation: true
---

# Integrate

集成阶段负责把多个 ticket 的结果放到一个一致的工作树中，并处理跨 ticket 冲突。它不假设所有 adapter 都有同一种 Merge UI。

## Canonical route

集成使用独立的 phase：

```yaml
phase: integration
runtime_id: auto
adapter_id: auto
agent_id: auto
capabilities_all: []
capabilities_any: [integrate, merge]
```

如果目标 adapter 只能提供 `merge`，就使用其内置合并能力。如果需要手动跨模块协调，目标必须提供 `integrate` 或 `execute`，并在调用中明确这一点。

## 何时使用

- 多个 ticket 组合后出现跨模块冲突
- 目标 adapter 没有 `merge` 能力
- adapter 的内置 Merge 失败，需要人工介入
- 需要整体 diff review，而不是逐个 ticket review

如果目标 adapter 声明了 `merge`，先使用它的内置流程。CodeG 的 To-dos Merge、冲突处理和 git 验证属于 `codeg-todos` 的能力，不要在外层重复实现。

## 通用流程

1. 扫描 ticket 状态和各自的 route，确认哪些结果已完成
2. 在一个具备 `integrate`、`merge` 或 `execute` 能力的 runtime 中建立集成工作区
3. 读取 spec、ticket、ADR 和相关 verify report，理解每一边的 intent
4. 查看累计 diff，定位跨模块冲突
5. 只做合并、冲突解决和必要的回归验证，不在集成阶段扩展新功能
6. 更新 ticket 状态和 handoff，记录 commit、route 和剩余风险
7. 运行 `/verify`

## 冲突解决原则

- 找到双方的 primary source：ticket 任务、spec 决策和 ADR
- 选择能同时满足 intent 的结果
- 如果两个 intent 互相矛盾，标记 `blocked`，退回用户决策
- 不自动选择“ours”或“theirs”
- 不在没有测试或验证的情况下声称冲突已经解决

## 结果记录

```markdown
## 🔀 集成报告

### 🧭 目标
- Phase：`integration`
- Runtime：<runtime-id>
- Adapter：<adapter-id>
- Agent：<agent-id 或 null>

### ✅ 已集成
- [002] <ticket>：<commit>
- [004] <ticket>：<commit>

### ⚠️ 冲突与决定
- <冲突>：<依据和决定>

### 🧪 验证
- <命令和结果>

### 🏁 状态
- [ ] 可继续 `/verify`
- [ ] blocked，等待用户决定
```

## CodeG 特殊说明

当且仅当目标是 `codeg-todos` 且它声明 `merge` 时：

- 逐个查看 Review 列
- 使用 adapter 的 Merge、Follow up、Complete 或 Abandon
- 让 CodeG agent 处理它自己的 worktree 合并
- 依赖 CodeG 的 git 验证结果

不要把这些 UI 名称写成其它 adapter 的要求。
