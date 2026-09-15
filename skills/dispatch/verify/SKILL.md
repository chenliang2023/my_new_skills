---
name: verify
description: 所有 ticket 合并后，在 CodeG 中做全量验证。可利用 CodeG 的 preflight command 或手动执行。产出验证报告供本机复审。
disable-model-invocation: true
---

# Verify

所有 ticket 合并到主分支后，做一次全量验证。这是服务器端交付前的最后一关。

## CodeG 的 preflight 机制

CodeG 的 Task settings → Worktree tab 有一个 **Preflight command** 字段：
- 任务进入 review 状态时自动在 worktree 中运行
- 红灯 / 绿灯直接显示在卡片上
- 失败时显示输出尾部，不需要打开终端

建议在项目文件夹的 Task settings 中设置 preflight command（如 `npm test` 或 `cargo test`），这样每个任务完成时就自动验证了。

## 前提

- `.workflow/tickets/` 中所有 ticket 状态为 `done`
- 代码已通过 CodeG Merge 合并到主分支

## 验证清单

按顺序执行，前一步失败则停下报告。

### 1. 类型检查
- 运行项目的类型检查命令（如 `tsc --noEmit`、`mypy`、`cargo check`）
- 记录所有错误

### 2. Lint
- 运行项目的 linter（如 `eslint`、`ruff`、`clippy`）
- 记录所有 warning 和 error

### 3. 单元测试
- 运行全量测试套件
- 记录失败测试

### 4. 构建
- 运行项目构建命令
- 确认构建产物正常生成

### 5. 集成测试（如有）
- 运行跨模块集成测试
- 记录失败

## 在 CodeG 中执行

两种方式：

### 方式 A：开一个新会话手动跑（推荐）

在 CodeG 中打开项目文件夹，开一个新的 Claude Code 会话：
```
请运行以下验证命令并报告结果：
1. tsc --noEmit
2. eslint .
3. npm test
4. npm run build
```

### 方式 B：创建一个 To-do 让 agent 跑

在 To-dos 面板创建一个任务：
- Title：`[verify] 全量验证`
- Description：上述验证清单
- Agent：Claude
- Start

完成后在 Review 列查看结果。

## 验证报告

产出报告写入 `.workflow/handoffs/verify-report-<date>.md`：

```markdown
# 验证报告 <date>

## 分支
main (commit: abc123)

## 验证结果

### 类型检查
- [ ] 通过 / [ ] 失败
- 命令：`tsc --noEmit`
- 错误数：N
- 错误摘要：<前 10 条>

### Lint
- [ ] 通过 / [ ] 失败
- 命令：`eslint .`
- 错误数：N
- 警告数：N

### 单元测试
- [ ] 全部通过 / [ ] N 条失败
- 命令：`npm test`
- 通过：M / 总计：T
- 失败列表：<失败的测试名>

### 构建
- [ ] 通过 / [ ] 失败
- 命令：`npm run build`
- 产物：<路径>

### 集成测试
- [ ] 通过 / [ ] 失败 / [ ] 不适用
- 失败列表：

## 结论
- [ ] 可交付：所有验证通过
- [ ] 不可交付：存在失败，需修复
- 待修复项：<列表>
```

## 下一步

- 如果全部通过：告知用户在服务器上 push 主分支，然后在本机拉取并运行 `/code-review` 做最终复审
- 如果有失败：列出失败项，可在 CodeG To-dos 中创建修复任务，退回 `/dispatch` 流程