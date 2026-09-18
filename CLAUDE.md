# Skills 仓库说明

本仓库是一套适配双环境（本机 VS Code + 服务器 CodeG）开发研究工作流的 skill 集合，参考 [mattpocock/skills](https://github.com/mattpocock/skills) 的结构。服务器端使用 [CodeG](https://github.com/xintaofei/codeg) 作为多 agent 编码工作台。

## 目录组织

Skills 按 bucket 文件夹组织：

- `ask/`：根技能，所有 skill 的路由器
- `planning/`：本机规划阶段（用户调用），产出文档供服务器消费
- `dispatch/`：服务器执行阶段（用户调用），驱动三 agent 并发
- `engineering/`：工程实践规范（模型调用），agent 实现时自动遵守
- `setup/`：初始化

每个 skill 文件夹内含一个 `SKILL.md`，使用 YAML frontmatter 声明 `name`、`description`、是否 `disable-model-invocation`。

- `disable-model-invocation: true` = 用户调用（只能人输入触发），职责是编排
- 无此字段 = 模型调用（agent 任务匹配时自动触发），承载可复用纪律

## 双环境架构

- **本机 (Windows + VS Code)**：规划端。负责研究、探索、规划、文档产出
- **服务器 (CodeG)**：执行端。使用 [CodeG](https://github.com/xintaofei/codeg) 做多 agent 并发开发
- CodeG 自带 To-dos 任务管理（git worktree 隔离、并发控制、Review/Merge 流程），skills 不重复造轮子
- 两端通过 `.workflow/` 目录同步上下文，通过 CodeG 的 Skill 管理同步 skills（`~/.codeg/skills/`）

## Agent 阵列

服务器端 agent 列表由 `.workflow/agents.json` 管理（用户可增删改，不绑定到固定三个 agent）。改 agent 表只动这一个文件，跑 `/setup-agents`。

仓库默认表：

- **DeepSeek Harness**（重推理，刀刃用，最贵，只接架构决策 / 安全敏感 ticket）
- **Pi**（覆盖最广，手动会话默认入口）
- **Google Antigravity**（前端/交互）

路由基于 `agents.json` 的 `tags` + `cost_tier` 动态决定，规则详见 [`skills/dispatch/route-agent/SKILL.md`](skills/dispatch/route-agent/SKILL.md "skills/dispatch/route-agent/SKILL.md")。默认按最便宜的 cost_tier 兜底；命中"架构决策 / 安全敏感 / 占位词"等关键词时升昂贵档。

## 主工作流

1. `/grill-me` 打磨想法 → 2. `/research` 如需调研 → 3. `/to-spec` 产出 spec → 4. `/to-tickets` 拆成可并发 ticket → 5. 交接到服务器 `/dispatch` 在 CodeG To-dos 创建任务 → 6. CodeG Review/Merge（agent 执行合并+冲突解决） → 7. `/verify` 全量验证 → 8. 本机 `/code-review` 复审

## 约定

- `.workflow/` 目录是两端共享的上下文载体，必须提交到 git
- ticket 文件是 agent 的工作单元，必须自包含
- `.workflow/` 下的文档遵守 `readable-docs`：先对人可读，再对 agent 可解析
- `examples/` 存放规范产出样例，用于校准合格线，不属于任何项目的 `.workflow/`
- 术语表 `CONTEXT.md` 让两端命名一致
- CodeG 的 Merge 流程处理单任务级别的冲突解决和 git 验证，`/integrate` 只在跨 ticket 协调时补充使用
- 不使用 em-dash

## 首次使用

运行 `/setup-workflow` 进行项目初始化配置。