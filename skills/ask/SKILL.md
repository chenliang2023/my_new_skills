---
name: ask
description: 问当前场景该用哪个 skill 或哪条流程。本仓库所有 skill 的路由器。不知道从哪开始时，先问它。
disable-model-invocation: true
---

# Ask

你不一定记得每个 skill，所以先问。

一个 **flow** 是一条贯穿多个 skill 的路径。大多数工作沿一条 **主流程** 走，两条 **入口支线** 汇入它。其余是独立的，或跑在底下的词汇层。

## 主流程：idea → ship

大多数工作走这条路。你有一个想法，想让它变成代码。

1. **`/grill-me`** 通过 relentless interview 打磨想法。**从这里开始**：你在本机 VS Code 里，有一个模糊需求或想法，需要把它问清楚。多轮提问，每轮推进 frontier，直到想法足够清晰可以写 spec。

2. **分叉：需要技术调研吗？**
   - **需要** → **`/research`** 派后台 agent 查文档、读源码、验证 API。拿到结果后回到 grill 继续。
   - **不需要** → 直接进下一步。

3. **`/to-spec`** 把 grill 摘要综合成正式 spec。不重新 interview，只做综合。产出 `.workflow/specs/<feature>.md`。

4. **`/to-tickets`** 把 spec 拆成 tracer-bullet ticket。每个 ticket 自包含，声明阻塞边，标注建议 agent。产出 `.workflow/tickets/` 下的一组文件。

5. **交接到服务器**：git push，然后在服务器 CodeG 上 git pull。

6. **`/dispatch`** 在 CodeG 的 To-dos 面板中创建任务，选对应 agent。两种模式：
   - **独立 To-do**：每个 ticket 一个任务，各自在独立 worktree 中运行（适合互不依赖的 ticket）
   - **`@` 委托**：在一个会话中用 `@agent` 让主 agent 委托子任务给其它 agent（适合主从型多 agent 协作）
   不确定哪个 agent？先跑 **`/route-agent`** 预检。

7. **CodeG Review / Merge**：任务完成后在 To-dos 的 Review 列看 diff，点 Merge。CodeG 的 agent 在自己的 session 中执行合并，自动解决冲突，CodeG 用 git 验证。**大多数情况你不需要 `/integrate`**。

8. **`/verify`** 全量验证（类型检查、lint、测试、构建）。也可以在 CodeG 的 Task settings 里设 preflight command，让每个任务进 review 时自动验证。

9. **本机复审**：git pull，然后 **`/code-review`** 做双轴审查（规范符合 + 工程标准），作为交付前最终检查。

### 上下文卫生

步骤 1-4 尽量在**一个不间断的 context window** 里完成（不要 compact 或 clear，直到 `/to-tickets` 之后），让 grill、spec、ticket 建立在同一个思考上。每个 ticket 在服务器端各自独立执行，互不干扰。

## 入口支线

一个起始场景，产生工作后汇入主流程。

- **首次在项目上启用本工作流** → **`/setup-workflow`**。配置 `.workflow/config.json`、在 CodeG 中设置 Task settings、启用 skills 给对应 agent。每个项目运行一次。

- **收到外部 bug 报告或需求** → 直接在 CodeG 的 To-dos 面板创建任务，选 agent 执行。不需要走 grill/spec/ticket 流程。或者从 GitHub/GitLab issue 通过 CodeG 的 Repository 面板创建任务。

- **服务器上 agent 跑出的代码有 bug** → **`/diagnosing-bugs`**。先建紧凑反馈循环（一个命令现在就红），再分阶段定位根因，最后写 regression 测试修复。

- **需要技术调研才能决定方向** → **`/research`**。派后台 agent 查 primary source，产出 Markdown 文件，结果回来后决定是继续 grill 还是直接 spec。

## 工程纪律

不是功能开发，是代码质量保障。model-invoked，agent 实现时自动遵守。

- **`/readable-docs`**：`.workflow/` 下文档的写法规范，以及写完文档后如何在对话里给用户一个工整回复。写 research、spec、ticket、grill 摘要、验证报告时遵守。让文档先对人可读，再对 agent 可解析。
- **`/tdd`**：红绿循环、好测试标准、seam 选择、反模式。agent 实现 ticket 时遵守。
- **`/diagnosing-bugs`**：系统性调试。先有反馈循环再修复，分阶段定位根因。
- **`/code-review`**：双轴复审（规范符合 + 工程标准）。本机拉取服务器结果后运行。
- **`/context-sync`**：本机与服务器 CodeG 之间的上下文同步规范。

## 服务器端执行

- **`/dispatch`** 是入口：在 CodeG To-dos 面板创建任务。不确定 agent 时先跑 `/route-agent`。
- **`/integrate`** 是补充：CodeG 的 Merge 流程已处理单任务级别冲突解决，本 skill 只在跨 ticket 问题时用。
- **`/verify`** 是收尾：全量验证。也可用 CodeG 的 preflight command 自动化。

## 独立

不在主流程上。

- **`/setup-workflow`**：首次配置。每个项目运行一次。
- **`/route-agent`**：独立运行可预检所有 ticket 的 agent 分派。也可被 `/dispatch` 调用。

## 快速决策树

```
你在哪？

├─ 本机 VS Code，有个想法/需求
│   └─ 想法够清晰吗？
│       ├─ 不够 → /grill-me
│       ├─ 需要调研 → /research，然后回 grill
│       └─ 够了 → /to-spec → /to-tickets → git push
│
├─ 服务器 CodeG，ticket 已就绪
│   └─ /dispatch（在 To-dos 创建任务）
│       ├─ 不知选哪个 agent → /route-agent
│       └─ 任务完成 → CodeG Review/Merge
│           └─ 有跨 ticket 问题 → /integrate
│           └─ 全部合并 → /verify
│
├─ 本机，服务器结果已拉取
│   └─ /code-review
│
├─ 首次在项目上用
│   └─ /setup-workflow
│
├─ 遇到 bug
│   └─ /diagnosing-bugs
│
└─ 不知道从哪开始
    └─ 你在这里。看完上面的决策树，选一条路。
```