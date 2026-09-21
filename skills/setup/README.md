# Setup Skills

初始化**被管理项目**的路径配置和 agent 清单。

本仓库是 skill 集合，自身没有 `.workflow/`。下面两个 skill 都用在别的项目里。

## Skills

- **[setup-workflow](./setup-workflow/SKILL.md)**：在被管理的项目里首次启用本工作流。只创建该项目自己的 `.workflow/` 目录与最小 `config.json`，只管理路径类字段。
- **[setup-agents](./setup-agents/SKILL.md)**：管理该项目的 `.workflow/agents.json`。登记本机和服务器上有哪些 agent，每个 agent 只记录 harness、模型强弱、响应速度快慢、所在机器和擅长领域。

## 基线名单

**[agents.baseline.json](./setup-agents/agents.baseline.json)**：固定下来的基础 agent 名单，同时包含本机和服务器的记录。在某个项目里首次运行 `/setup-agents` 时，把这份内容复制成该项目的 `.workflow/agents.json`，再按实际情况增改。

这份基线保存在 skills 仓库里，因为它要跨项目复用；`.workflow/agents.json` 则属于每个项目自己，随项目提交。

## 两份配置各管一类事实

| 文件 | 真相源 | 管理内容 |
|------|--------|----------|
| `<项目>/.workflow/config.json` | `/setup-workflow` | 路径模板 |
| `<项目>/.workflow/agents.json` | `/setup-agents` | agent 的 harness、模型强弱、速度、所在机器和擅长领域 |

没有 runtime 注册表。本机和服务器具备完全相同的能力，差别只在各自装了哪些 agent，因此没有任何东西需要按环境分开注册。
