# Dispatch Skills

服务器端（CodeG）执行阶段使用的 skills。用户调用，驱动 `.workflow/agents.json` 里声明的 agent 并发执行 ticket。

这些 skill 配合 CodeG 的 To-dos 面板使用，不重复 CodeG 已有的任务管理、worktree 隔离、Review/Merge 流程。

## Skills

- **[dispatch](./dispatch/SKILL.md)**：读取 ticket，在 CodeG 的 To-dos 面板中创建任务，分派给对应 agent。支持两种模式：独立 To-do（适合互不依赖的 ticket）和 `@` 委托（适合主从型多 agent 协作）。
- **[route-agent](./route-agent/SKILL.md)**：根据 ticket 内容自动判断应分派给哪个 agent（基于 `.workflow/agents.json` 的 tags + cost_tier 路由，不硬编码 agent 名）。供 `/dispatch` 调用，也可独立运行做路由预检。
- **[integrate](./integrate/SKILL.md)**：跨 ticket 协调（补充用）。CodeG 的 Merge 流程已处理单任务级别的冲突解决和 git 验证，本 skill 只在跨 ticket 问题时使用。
- **[verify](./verify/SKILL.md)**：全量验证（类型检查、lint、测试、构建），可利用 CodeG 的 preflight command 或手动执行。产出验证报告供本机复审。