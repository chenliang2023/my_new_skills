---
name: integrate
description: CodeG 的 Merge 流程已经处理了单任务级别的冲突解决和 git 验证。本 skill 只在需要跨多个 ticket 协调时补充使用，或在 CodeG Merge 失败后手动介入。
disable-model-invocation: true
---

# Integrate

重要：CodeG 的 To-dos Merge 流程已经内置了：
- agent 在自己的 session 中执行合并
- 自动解决冲突
- CodeG 用 git 验证合并是否真的成功
- 失败会退回 review

所以大多数情况下你不需要手动 integrate。本 skill 只在以下场景补充使用。

## 何时使用本 skill

- 多个 ticket 合并后出现跨模块冲突（CodeG 逐个 Merge 时没发现，但组合起来有问题）
- CodeG Merge 失败后需要手动介入
- 你想在一次 review 中看所有变更的整体 diff，而不是逐个

## 正常流程（首选）

正常情况下，让 CodeG 自己处理：
1. 每个 ticket 在 To-dos 面板完成后进入 Review 列
2. 逐个点 Merge，CodeG 的 agent 会解决冲突
3. CodeG 验证 git 确实合并了
4. 所有 ticket 合并后运行 `/verify`

## 补充流程：跨 ticket 协调

如果需要手动跨 ticket 协调：

1. **扫描完成状态**：读取 `.workflow/tickets/` 下所有 ticket，找出 CodeG 已标记为 review 但你想批量处理的

2. **整体 diff review**：在 CodeG 中打开项目文件夹，用 Git 面板查看所有已合并 ticket 的累计 diff

3. **解决跨 ticket 冲突**：如果发现跨模块问题，在 CodeG 中开一个新 Pi 会话（手动会话默认入口；如涉及架构决策再升 Harness）整体修复：
   ```
   以下 ticket 已合并但存在跨模块冲突：
   - [002] user-model
   - [004] user-api
   冲突：User 接口在 [002] 加了 email，在 [004] 加了 phoneNumber，但 AuthService 期望的字段不匹配
   请修复让它们一致
   ```

4. **标记为 done**：在 ticket 文件中把状态改为 `done`，附上 CodeG 的 merge commit hash

## 冲突解决原则

如果需要手动解决冲突：

- 找到冲突两边的 primary source（ticket 的 `## 任务` 描述、相关 ADR、spec 决策）
- 理解两边各自想实现什么意图
- 选择能同时满足两个意图的合并结果
- 如果两个 intent 互相矛盾，标记为 `blocked`，退回给用户决策，不要擅自选边

## 不要做的事

- 不要重复 CodeG 已经做的事（Merge、冲突解决、git 验证）
- 不要 `git merge --abort`：CodeG 没有给你这个选项，也不需要
- 不要自动选 "ours" 或 "theirs"：那会丢失一边的 intent
- 不要在集成阶段写新功能：只做合并和冲突解决