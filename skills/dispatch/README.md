# Dispatch Skills

分派和验证阶段使用的 skills。执行方式统一为 git worktree，本机和服务器都能跑，区别只在用哪台的 agent。

每个 ticket 有两条 agent 推荐（`local`、`server`），`/dispatch` 按当前所在机器取一条。

## Skills

- **[dispatch](./dispatch/SKILL.md)**：扫描 ticket、排执行顺序、建 worktree、选定 agent、给工具建议，并在完成后合并。
- **[route-agent](./route-agent/SKILL.md)**：读取 `.workflow/agents.json`，为每个 ticket 算出本机和服务器各自的最佳 agent。
- **[verify](./verify/SKILL.md)**：合并后做全量验证，产出 `.workflow/handoffs/verify-report-<date>.md`。

没有单独的集成 skill。合并是 dispatch 流程的自然结尾。
