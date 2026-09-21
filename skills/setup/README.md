# Setup Skills

初始化项目级工作流路径、Agent 注册表和 Runtime 注册表。

## Skills

- **[setup-workflow](./setup-workflow/SKILL.md)**：首次启用本工作流。只创建 `.workflow/` 目录与最小 `config.json`，只管理路径类字段。
- **[setup-agents](./setup-agents/SKILL.md)**：管理 `.workflow/agents.json` 里的 agent 注册表。新增、删除或修改 agent 的 `id`、`display_name`、`role`、`cost_tier` 和 `tags`。
- **[setup-runtimes](./setup-runtimes/SKILL.md)**：管理 `.workflow/runtimes.json` 里的 runtime、adapter、能力、agent policy 和阶段默认路由。CodeG 是可选 runtime，`codeg-todos` 是它的可选 adapter。

三份注册表各管一类事实：

| 文件 | 真相源 | 管理内容 |
|------|--------|----------|
| `.workflow/config.json` | `/setup-workflow` | 路径模板 |
| `.workflow/agents.json` | `/setup-agents` | agent 身份、角色、标签和成本 |
| `.workflow/runtimes.json` | `/setup-runtimes` | runtime、adapter、能力、agent policy 和路由默认值 |
