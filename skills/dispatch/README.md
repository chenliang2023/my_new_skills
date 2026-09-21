# Dispatch Skills

任务路由、分派、验证和集成阶段使用的 skills。它们可以在任何注册了相应能力的 runtime 中运行。

`.workflow/runtimes.json` 决定可用的 runtime 和 adapter，`.workflow/agents.json` 决定可用的 agent。CodeG 是一个可选 runtime，`codeg-todos` 只是它的一个 adapter；如果当前目标没有它，skill 按 adapter 能力使用本机会话、脚本任务队列或其它执行方式。

## Skills

- **[dispatch](./dispatch/SKILL.md)**：读取 ticket，检查阻塞边，根据 canonical route 和目标 adapter 的能力创建任务并分派给 agent。
- **[route-agent](./route-agent/SKILL.md)**：根据 ticket 的 phase、能力、标签、runtime 和 adapter 约束，输出完整路由，不硬编码 agent 名称或环境。
- **[integrate](./integrate/SKILL.md)**：在 adapter 没有足够的 merge 能力，或需要跨 ticket 协调时使用。
- **[verify](./verify/SKILL.md)**：在独立的 `verification` phase 做全量验证并产出 `.workflow/handoffs/verify-report-<date>.md`，可利用 adapter 提供的 preflight。
