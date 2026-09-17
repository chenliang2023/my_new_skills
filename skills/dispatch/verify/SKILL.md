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

## 🌿 分支
main (commit: abc123)

## 📊 验证结果

### 🔍 类型检查
- [ ] 通过 / [ ] 失败
- 命令：`tsc --noEmit`
- 错误数：N
- 错误摘要：<前 10 条>

### 🧹 Lint
- [ ] 通过 / [ ] 失败
- 命令：`eslint .`
- 错误数：N
- 警告数：N

### 🧪 单元测试
- [ ] 全部通过 / [ ] N 条失败
- 命令：`npm test`
- 通过：M / 总计：T
- 失败列表：<失败的测试名>

### 📦 构建
- [ ] 通过 / [ ] 失败
- 命令：`npm run build`
- 产物：<路径>

### 🔗 集成测试
- [ ] 通过 / [ ] 失败 / [ ] 不适用
- 失败列表：

## 🏁 结论
- [ ] 可交付：所有验证通过
- [ ] 不可交付：存在失败，需修复
- 待修复项：<每条写清失败的命令、报错原文、影响的 ticket 和范围>
```

报告按 `/readable-docs` 写。这份报告是状态密集型，emoji 用得密一点才有价值：每节标题、每条验证结论、每个失败项前面都配上，让人一眼扫出哪些过了、哪些没过。

但不要替代文字，通过的是什么要写清楚。段落也要短，一项验证的结论、命令、输出各自成段。

待修复项不能只写命令名，要让本机的你能据此判断该退回哪个 ticket，不需要重新跑一遍验证。失败列表和错误摘要一律贴原始输出，不要转述。

失败项之间有依赖关系（比如类型检查不过导致构建必然失败）时，用一张 Mermaid 图说明因果，比列表更清楚，读者能直接看出先修哪个。

## 回报给用户

按 `/readable-docs` 的「写完之后的回复」发一条四块回复。

验证类的第二块要直接给**哪些过了、哪些没过**，失败的最小单元贴原始报错；第三块写清退回哪个 ticket 修。不要只说「验证完成」，也不要复述报告全文。

## 下一步

- ✅ 如果全部通过：告知用户在服务器上 push 主分支，然后在本机拉取并运行 `/code-review` 做最终复审
- ❌ 如果有失败：列出失败项，可在 CodeG To-dos 中创建修复任务，退回 `/dispatch` 流程