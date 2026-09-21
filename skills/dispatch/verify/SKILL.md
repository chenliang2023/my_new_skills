---
name: verify
description: 在 ticket 的工作树合并后做全量验证，产出共享验证报告。本机和服务器都能跑，用哪台就从 agents.json 里取那台的 agent。
disable-model-invocation: true
---

# Verify

相关工作树合并到主分支后，运行一次全量验证。这是工作流的质量门。

验证不受机器限制。本机和服务器都能跑全部检查，区别只在用哪台的 agent。

## 前提

- 相关 ticket 已完成并合并，或用户明确指定要验证的范围
- 目标 commit 已经在当前机器上
- 工作区干净

## route

验证是独立阶段，用自己的 phase 重新取推荐：

```yaml
phase: verification
local: claude-domestic
server: oh-my-pi-gpt
```

按当前机器取一条：

- 本机就取 `local:`
- 服务器就取 `server:`
- `manual` 表示当前会话或人工执行

验证属于常规档，`medium` 及以上都能做。缺推荐时先运行 `/route-agent`，不要沿用 execution 阶段的 agent，除非判断出来确实是同一个。

## 验证清单

按顺序执行。前一步失败时先报告，不要把后续结果伪装成通过。

1. 类型检查
2. Lint
3. 单元测试
4. 构建
5. 集成测试（如有）
6. 与本批 ticket 相关的 smoke test

命令从项目现有配置读，不要用本文件里的示例替代实际命令。

## 结果报告

写入 `.workflow/handoffs/verify-report-<date>.md`：

```markdown
# 验证报告 <date>

## 🧭 运行目标
- Phase：`verification`
- 机器：<local 或 server>
- Agent：<agent-id 或 manual>
- Commit：<hash>

## 📊 结果

### 🔍 类型检查
- [ ] 通过 / [ ] 失败
- 命令：`<实际命令>`
- 原始输出：<必要部分>

### 🧹 Lint
- [ ] 通过 / [ ] 失败
- 命令：`<实际命令>`

### 🧪 单元测试
- [ ] 全部通过 / [ ] 有失败
- 通过：M / 总计：T
- 失败列表：<测试名和原始输出>

### 📦 构建
- [ ] 通过 / [ ] 失败 / [ ] 不适用

### 🔗 集成测试
- [ ] 通过 / [ ] 失败 / [ ] 不适用

## 🏁 结论
- [ ] 可交付
- [ ] 不可交付
- 待修复项：<命令、原始报错、影响的 ticket>
```

按 `/readable-docs` 写。失败项保留最小原始输出，让另一台机器能据此继续。

## 跨机器验证

在一台机器上改、在另一台机器上验，是这套流程的正常用法：

1. 改的那台 commit 并推送
2. 验的那台拉取同一个 commit
3. 在验的那台按 `phase: verification` 取推荐并执行
4. 报告提交到 git，两台都能看到

同一个 commit 在两台机器上验出不同结果时，先查环境差异（依赖版本、环境变量、外部服务），不要先怀疑代码。

## 回报与下一步

- 全部通过：提交报告，运行 `/code-review`，再决定是否 `/version`
- 有失败：列出失败项和受影响 ticket，写 bug fix ticket，重新走 `/dispatch`，修完再跑 `/verify`

## 反模式

- 🚫 跳过类型检查或 lint 直接跑测试
- 🚫 把「本地能过」当成验证通过
- 🚫 失败项只写「测试失败」，不给命令和原始输出
- 🚫 沿用 execution 阶段的 agent 而不重新走 `/route-agent`
