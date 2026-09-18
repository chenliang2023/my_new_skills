# Setup Skills

初始化工作流的 skill。

## Skills

- **[setup-workflow](./setup-workflow/SKILL.md)**：首次启用本工作流。**只**创建 `.workflow/` 目录与最小 `config.json`（路径类字段）。服务端相关配置（CodeG Task settings、Skill 矩阵、agent 安装）由用户在 CodeG 端自行配置。每个项目运行一次。
- **[setup-agents](./setup-agents/SKILL.md)**：管理 `.workflow/agents.json` 里的 agent 注册表。新增 / 删除 / 改 agent 的 `id` / `display_name` / `cost_tier` / `tags`。**用户增删 agent 只动这一个文件**；`/route-agent` / `/dispatch` / `/verify` 会自动按新表路由。