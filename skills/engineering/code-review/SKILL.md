---
name: code-review
description: 对当前 diff、PR 或分支做双轴代码复审。合并后或发布前运行，本机和服务器都可以。
disable-model-invocation: true
---

# Code Review

对当前 diff、PR 或分支做双轴审查：

1. **规范符合**：代码是否实现了 spec 和 ticket 描述的行为
2. **工程标准**：代码质量、测试质量和架构合理性

## 何时运行

- ticket 合并后，看整体 diff
- 发布前做一次独立复审
- 怀疑实现偏离 spec 时

审查可以在改代码的那台机器上做，也可以在另一台上做。跨机器时先通过 git 和 `.workflow/handoffs/` 同步上下文。

## route

复审是独立阶段，用自己的 phase 重新取推荐：

```yaml
phase: review
local: claude-code-opus
server: codex-high
```

按当前机器取一条。优先选与 execution 阶段推荐不同的 agent，换一双眼睛看。同一台机器上只有那一个 agent 时照用，并注明是同 agent 自审。

缺推荐时运行 `/route-agent`。

## 流程

### 1. 确定范围

读取当前 diff。分支、PR 或工作区的范围由用户明确指定，不根据机器名称或目录名猜测 base。

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
- 机器：<local 或 server>
- Agent：<agent-id 或 manual>
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
