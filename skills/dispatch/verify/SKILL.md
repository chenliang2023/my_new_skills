---
name: verify
description: 在具备 verify 能力的 runtime 中做全量验证，利用可选的 preflight，并产出共享验证报告。
disable-model-invocation: true
---

# Verify

所有目标 ticket 完成并达到可验证状态后，运行一次全量验证。这是工作流的质量门，不属于某个固定环境或 agent。

## Adapter 能力

- `verify`：可以运行验证命令
- `preflight`：任务进入 review 前自动运行命令
- `execute`：需要修改测试或修复失败时
- `review`：需要同时做结果审查时

CodeG 的 Task settings 和 preflight 只是 `codeg-todos` adapter 的一种实现。其它 adapter 可以使用脚本、CI 或当前会话完成同样的验证。

## 前提

- `.workflow/tickets/` 中相关 ticket 已完成，或用户明确指定要验证的范围
- 代码和上下文已在目标 runtime 同步到同一 commit
- 目标 route 至少提供 `verify`

## Canonical route

验证使用独立的 phase，不继承执行 ticket 的 route：

```yaml
phase: verification
runtime_id: auto
adapter_id: auto
agent_id: auto
capabilities_all: [verify]
capabilities_any: []
```

`agent_id: null` 允许当前用户会话、脚本或人工执行，但目标 adapter 的 `agent_ids` 必须是空数组。`preflight` 不能替代 `verify`。

## 选择 route

优先级：

1. 用户本次指定的 runtime、adapter、agent
2. `/verify` 调用参数或 handoff 中 phase 为 `verification` 的 route
3. `runtimes.json` 的 `routing.defaults.verification`
4. `/route-agent` 按 `verify` 能力动态匹配

不查找 `manual_entry`，也不默认为某个 agent、CodeG、本机或服务器。如果有多个等价目标，报告候选并让用户选择，除非注册表已有阶段默认。

## 验证清单

按顺序执行，前一步失败时先报告，不把后续结果伪装成通过：

1. 类型检查
2. Lint
3. 单元测试
4. 构建
5. 集成测试（如有）
6. 与本批 ticket 相关的 smoke test 或迁移检查

命令从项目配置、spec、ticket 或用户输入读取，不要使用仓库外的固定命令示例替代实际命令。

## 结果报告

写入 `.workflow/handoffs/verify-report-<date>.md`：

```markdown
# 验证报告 <date>

## 🧭 运行目标
- Phase：`verification`
- Runtime：<runtime-id>
- Adapter：<adapter-id>
- Agent：<agent-id 或 null>
- Commit：<hash>

## 📊 验证结果

### 🔍 类型检查
- [ ] 通过 / [ ] 失败
- 命令：`<实际命令>`
- 原始输出：<必要部分>

### 🧹 Lint
- [ ] 通过 / [ ] 失败
- 命令：`<实际命令>`
- 原始输出：<必要部分>

### 🧪 单元测试
- [ ] 全部通过 / [ ] 有失败
- 命令：`<实际命令>`
- 通过：M / 总计：T
- 失败列表：<测试名和原始输出>

### 📦 构建
- [ ] 通过 / [ ] 失败 / [ ] 不适用
- 命令：`<实际命令>`

### 🔗 集成测试
- [ ] 通过 / [ ] 失败 / [ ] 不适用

## 🏁 结论
- [ ] 可交付：所有必要验证通过
- [ ] 不可交付：存在失败，需修复
- 待修复项：<命令、原始报错、影响 ticket 和范围>
```

报告按 `/readable-docs` 写。失败项保留最小原始输出，让另一个 runtime 能据此继续，不要只写“测试失败”。

## 回报与下一步

- 全部通过：提交验证报告，按需要在任意 `review` runtime 运行 `/code-review`，再决定是否 `/version`
- 有失败：列出失败项和受影响 ticket，在具备 `execute` 的 route 上创建修复任务，修复后重新 `/verify`
